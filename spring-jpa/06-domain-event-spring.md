# Domain Event Spring 통합 — `AbstractAggregateRoot`와 자동 발행

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `AbstractAggregateRoot`가 `@DomainEvents`와 `@AfterDomainEventsPublication`으로 동작하는 내부 구조는?
- `save()` 호출 이후 이벤트가 자동 발행되는 흐름은 어떻게 되는가?
- `SimpleJpaRepository`가 이벤트를 발행하는 시점과 트랜잭션 경계 관계는?
- `AbstractAggregateRoot` 사용 시 주의해야 할 함정은?
- 커스텀 `AggregateRoot`와 `AbstractAggregateRoot` 중 어느 것을 선택해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Ch3-08과 Ch4-02에서 이벤트를 수집하고 Application Service에서 발행하는 방법을 다뤘다. Spring Data는 이것을 자동화하는 `AbstractAggregateRoot`를 제공한다. `save()` 한 번으로 Aggregate 저장과 이벤트 발행이 동시에 이루어진다. 편리하지만 내부 동작을 모르면 "이벤트가 왜 발행되지 않지?", "트랜잭션이 커밋 전인데 핸들러가 실행된다고?" 같은 혼란이 생긴다.

이 문서는 `AbstractAggregateRoot`의 내부 메커니즘을 분해하고, 언제 사용하고 언제 커스텀 방식을 선택해야 하는지 기준을 제시한다.

---

## 😱 흔한 실수 (Before — AbstractAggregateRoot 없는 수동 발행의 반복)

```java
// AbstractAggregateRoot 없이 매 Application Service 메서드마다 반복
@Service
@Transactional
public class OrderApplicationService {

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = orderFactory.create(command);
        orderRepository.save(order);

        // 매번 반복되는 boilerplate
        List<DomainEvent> events = order.pullDomainEvents();
        events.forEach(eventPublisher::publishEvent);

        return order.id();
    }

    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId()).orElseThrow();
        order.cancel(command.reason());
        orderRepository.save(order);

        // 또 반복
        List<DomainEvent> events = order.pullDomainEvents();
        events.forEach(eventPublisher::publishEvent);
    }

    public void addItem(AddItemCommand command) {
        Order order = orderRepository.findById(command.orderId()).orElseThrow();
        order.addLine(...);
        orderRepository.save(order);

        // 계속 반복 — 실수로 빠뜨리면 이벤트 유실
        List<DomainEvent> events = order.pullDomainEvents();
        events.forEach(eventPublisher::publishEvent);
    }
}
```

---

## ✨ 올바른 접근 (After — AbstractAggregateRoot로 자동화)

```java
// AbstractAggregateRoot 상속
@Entity
@Table(name = "orders")
public class Order extends AbstractAggregateRoot<Order> {

    @EmbeddedId
    private OrderId id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    // ...

    public static Order place(CustomerId customerId, List<OrderLine> lines,
                               Discount discount, Money shippingFee) {
        Order order = new Order(new OrderId(), customerId, PENDING, lines);
        // AbstractAggregateRoot의 registerEvent() 사용
        order.registerEvent(new OrderPlaced(
            order.id, customerId, OrderLineSnapshot.of(lines), order.totalAmount, LocalDateTime.now()
        ));
        return order;
    }

    public void cancel(String reason) {
        validateCancellable();
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelled(this.id, reason, LocalDateTime.now()));
    }
}

// Application Service — 이벤트 발행 코드 없음
@Service
@Transactional
public class OrderApplicationService {

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = orderFactory.create(command);
        orderRepository.save(order);
        // 끝! save() 시 Spring Data가 자동으로 이벤트 발행
        return order.id();
    }

    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId()).orElseThrow();
        order.cancel(command.reason());
        orderRepository.save(order);
        // 끝! 이벤트 발행 자동
    }
}
```

---

## 🔬 내부 동작 원리

### 1. AbstractAggregateRoot 구조 분해

```java
// AbstractAggregateRoot 실제 구현 (Spring Data Commons)
public class AbstractAggregateRoot<A extends AbstractAggregateRoot<A>> {

    @Transient  // JPA에서 영속화 제외
    private transient final List<Object> domainEvents = new ArrayList<>();

    // Aggregate가 호출: 이벤트를 내부 리스트에 추가
    protected <T> T registerEvent(T event) {
        Objects.requireNonNull(event, "이벤트는 null일 수 없습니다");
        domainEvents.add(event);
        return event;
    }

    // Spring Data가 호출: 이벤트 목록 반환 + 목록 비움
    @DomainEvents
    Collection<Object> domainEvents() {
        return Collections.unmodifiableList(domainEvents);
    }

    // Spring Data가 호출: 이벤트 발행 완료 후 목록 비움
    @AfterDomainEventsPublication
    protected void clearDomainEvents() {
        domainEvents.clear();
    }
}
```

### 2. save() 이후 이벤트 발행 흐름

```
호출 흐름:

1. orderRepository.save(order)
   ↓
2. JpaOrderRepository.save(order)  → springDataRepo.save(order)
   ↓
3. SimpleJpaRepository.save(entity)
   ↓
4. JpaEntityInformation 확인: entity가 @DomainEvents를 가지는가?
   ↓ (AbstractAggregateRoot는 @DomainEvents 있음)
5. entity.domainEvents() 호출 → 이벤트 목록 수집
   ↓
6. EntityManager.persist() / merge() 실행 (DB 저장)
   ↓
7. ApplicationEventPublisher.publishEvent(event) × N번 호출
   ↓
8. entity.clearDomainEvents() 호출 (@AfterDomainEventsPublication)

타이밍:
  이벤트 발행(7단계)은 DB 저장(6단계) 후, 트랜잭션 커밋 전

  @TransactionalEventListener(AFTER_COMMIT) 핸들러:
  → 트랜잭션 커밋 후 실행 (DB에 데이터 확정됨)
  
  @EventListener 핸들러:
  → 7단계에서 즉시 실행 (트랜잭션 아직 열려 있음)
```

### 3. AbstractAggregateRoot의 함정

```
함정 1: save() 없이 이벤트 발행 안 됨
  order.cancel("이유");  // 이벤트 등록됨
  // orderRepository.save(order) 없으면 이벤트 발행 안 됨!
  
  JPA 더티 체킹으로 Order는 업데이트되지만
  @DomainEvents는 save()가 호출될 때만 실행됨
  → 항상 명시적 save() 호출 필요

함정 2: 동일 이벤트 두 번 발행 가능
  orderRepository.save(order);  // 이벤트 발행 + clearDomainEvents()
  order.registerEvent(new OtherEvent());  // 새 이벤트 등록
  orderRepository.save(order);  // 새 이벤트만 발행 (OK)
  
  하지만:
  orderRepository.save(order);  // 이벤트 발행
  // clearDomainEvents()가 호출돼 이벤트 비워짐
  // order.registerEvent()를 다시 호출하지 않으면 두 번째 save()에서 이벤트 없음

함정 3: @Transient 필드의 직렬화 주의
  AbstractAggregateRoot의 domainEvents는 @Transient
  JPA: 영속화 안 됨 (정상)
  Jackson: @Transient 무시할 수 있음 (설정 필요)
  Lazy Loading 후 detach: domainEvents 초기화될 수 있음

함정 4: Outbox Pattern과 함께 사용
  AbstractAggregateRoot는 ApplicationEventPublisher로만 발행
  Outbox 테이블에 저장하려면 별도 처리 필요
  → Outbox가 필요하다면 커스텀 AggregateRoot 권장
```

### 4. AbstractAggregateRoot vs 커스텀 AggregateRoot

```
AbstractAggregateRoot 선택:
  ✅ 같은 서비스 내부 이벤트만 사용
  ✅ ApplicationEventPublisher로 충분
  ✅ Outbox Pattern 불필요
  ✅ Spring Data와 강하게 통합된 환경

커스텀 AggregateRoot 선택:
  ✅ Outbox Pattern 사용 (Application Service에서 직접 제어 필요)
  ✅ 도메인이 Spring Data에 의존하지 않길 원함
  ✅ 이벤트 발행 시점을 세밀하게 제어해야 함
  ✅ 완전한 PI(Persistence Ignorance) 추구

커스텀 AggregateRoot 구현:
  public abstract class AggregateRoot {
      @Transient
      private final List<DomainEvent> events = new ArrayList<>();
      
      protected void registerEvent(DomainEvent event) {
          events.add(event);
      }
      
      public List<DomainEvent> pullDomainEvents() {
          List<DomainEvent> pulled = List.copyOf(events);
          events.clear();
          return pulled;
      }
  }
  
  // Application Service에서 명시적 발행
  order.pullDomainEvents().forEach(publisher::publishEvent);
```

---

## 💻 실전 코드

### AbstractAggregateRoot 활용 완전 예시

```java
@Entity
@Table(name = "orders")
public class Order extends AbstractAggregateRoot<Order> {

    @EmbeddedId
    private OrderId id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @Embedded
    private CustomerId customerId;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private List<OrderLine> lines = new ArrayList<>();

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "total_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "total_currency"))
    })
    private Money totalAmount;

    @Column(name = "created_at", nullable = false, updatable = false)
    private LocalDateTime createdAt;

    @Version
    private Long version;

    protected Order() {}

    public static Order place(CustomerId customerId, List<OrderLine> lines,
                               Discount discount, Money shippingFee) {
        Order order = new Order();
        order.id = new OrderId();
        order.customerId = customerId;
        order.status = OrderStatus.PENDING;
        order.lines = new ArrayList<>(lines);
        order.createdAt = LocalDateTime.now();
        order.recalculateTotalAmount(discount, shippingFee);

        // registerEvent: AbstractAggregateRoot 메서드
        order.registerEvent(new OrderPlaced(
            order.id, customerId,
            OrderLineSnapshot.of(lines),
            order.totalAmount,
            order.createdAt
        ));
        return order;
    }

    public void cancel(String reason) {
        if (status != OrderStatus.PENDING && status != OrderStatus.PAID) {
            throw new OrderNotCancellableException(status);
        }
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelled(this.id, reason, LocalDateTime.now()));
    }
}

// @TransactionalEventListener: save() 후 트랜잭션 커밋 시 실행
@Component
public class OrderEventHandler {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)
    public void onOrderPlaced(OrderPlaced event) {
        // 트랜잭션 커밋 후 실행 → Order가 DB에 반드시 존재
        pointWalletService.earn(event.customerId(), event.totalAmount());
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async
    public void sendConfirmationEmail(OrderPlaced event) {
        emailService.sendOrderConfirmation(event.customerId(), event.totalAmount());
    }
}
```

---

## 📊 설계 비교

```
AbstractAggregateRoot vs 커스텀 AggregateRoot:

                AbstractAggregateRoot        커스텀 AggregateRoot
────────────┼──────────────────────────┼──────────────────────────
편의성       │ 높음 (save()로 자동 발행) │ 중간 (pullEvents() 수동 호출)
────────────┼──────────────────────────┼──────────────────────────
Spring 의존 │ 있음 (Spring Data 클래스) │ 없음 (순수 Java)
────────────┼──────────────────────────┼──────────────────────────
Outbox 지원 │ 어려움 (별도 처리 필요)  │ 쉬움 (pullEvents() 후 저장)
────────────┼──────────────────────────┼──────────────────────────
발행 시점   │ save() 내부 (DB 저장 후) │ 명시적 (개발자가 결정)
────────────┼──────────────────────────┼──────────────────────────
이벤트 제어 │ 제한적 (Spring Data 위임)│ 완전 제어
────────────┼──────────────────────────┼──────────────────────────
도메인 PI   │ 낮음 (Spring 클래스 상속) │ 높음 (독립)
```

---

## ⚖️ 트레이드오프

```
AbstractAggregateRoot의 편의와 비용:
  편의: save() 한 번으로 저장 + 이벤트 발행 자동화
  비용:
    - 도메인 클래스가 Spring Data를 상속 (PI 훼손)
    - Outbox Pattern과 통합 어려움
    - save() 없으면 이벤트 발행 안 됨 (함정)
    - 이벤트 발행 시점 제어 어려움

현실적 권고:
  Outbox 없이 @TransactionalEventListener로 충분하면 → AbstractAggregateRoot
  Kafka + Outbox Pattern 필요하면 → 커스텀 AggregateRoot
  마이크로서비스 이전 단계 모놀리스 → AbstractAggregateRoot로 시작
  완전한 MSA + 높은 신뢰성 → 커스텀 AggregateRoot + Outbox
```

---

## 📌 핵심 정리

```
AbstractAggregateRoot 핵심:

동작 메커니즘:
  @DomainEvents: 이벤트 목록 반환 (Spring Data가 호출)
  @AfterDomainEventsPublication: 발행 후 목록 비움
  SimpleJpaRepository.save(): @DomainEvents → 이벤트 발행 → @AfterDomainEvents

발행 타이밍:
  DB 저장 후, 트랜잭션 커밋 전
  → @TransactionalEventListener(AFTER_COMMIT)으로 커밋 후 처리 권장

주의 사항:
  save() 없으면 이벤트 발행 안 됨 (더티 체킹만으로 불충분)
  clearDomainEvents() 후 새 이벤트를 다시 registerEvent()해야 발행됨

선택 기준:
  Spring Data 통합 + 단순 이벤트 → AbstractAggregateRoot
  Outbox Pattern + 완전 PI → 커스텀 AggregateRoot
```

---

## 🤔 생각해볼 문제

**Q1.** `AbstractAggregateRoot`를 상속한 Order에서 `save()` 없이 JPA 더티 체킹만으로 업데이트하면 이벤트가 발행되지 않는다. 왜인가?

<details>
<summary>해설 보기</summary>

**`@DomainEvents`는 `SimpleJpaRepository.save()`가 호출될 때만 실행되기 때문입니다.**

```java
// SimpleJpaRepository.save() 내부 (간략화)
@Transactional
public <S extends T> S save(S entity) {
    // 1. 신규 Entity인지 확인
    if (entityInformation.isNew(entity)) {
        em.persist(entity);
    } else {
        entity = em.merge(entity);
    }
    // 2. @DomainEvents 메서드 호출 → 이벤트 수집
    // 3. ApplicationEventPublisher로 이벤트 발행
    // 4. @AfterDomainEventsPublication 호출
    return entity;
}
```

JPA 더티 체킹은 `EntityManager`가 내부적으로 처리하며 `SimpleJpaRepository.save()`를 거치지 않습니다. 따라서 `@DomainEvents`가 실행되지 않고 이벤트도 발행되지 않습니다.

이것이 `AbstractAggregateRoot`를 사용할 때 항상 명시적 `save()`를 호출해야 하는 이유입니다.

</details>

---

**Q2.** 같은 트랜잭션에서 Order를 두 번 수정하고 두 번 `save()`를 호출하면 이벤트가 어떻게 발행되는가?

<details>
<summary>해설 보기</summary>

**각 `save()` 호출마다 그 시점까지 등록된 이벤트가 발행됩니다.**

```java
@Transactional
public void processOrder(OrderId orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();

    order.addItem(...);          // ItemAdded 이벤트 등록
    orderRepository.save(order); // ItemAdded 발행 + clearDomainEvents()

    order.applyDiscount(...);    // DiscountApplied 이벤트 등록
    orderRepository.save(order); // DiscountApplied 발행 + clearDomainEvents()
}
```

각 `save()` 후 `clearDomainEvents()`가 호출되므로, 두 번째 `save()`에서 이전 이벤트는 이미 비워진 상태입니다. 중복 발행 없습니다.

**주의**: 두 `save()` 사이에 이벤트 핸들러가 `@EventListener`(즉시)라면 같은 트랜잭션 내에서 실행됩니다. `@TransactionalEventListener(AFTER_COMMIT)`이라면 트랜잭션 커밋 후 모아서 처리됩니다.

</details>

---

**Q3.** `AbstractAggregateRoot`와 `@TransactionalEventListener`를 함께 쓸 때, 이벤트 핸들러에서 예외가 발생하면 Order 저장 트랜잭션은 롤백되는가?

<details>
<summary>해설 보기</summary>

**`AFTER_COMMIT` 단계라면 Order 저장 트랜잭션은 이미 커밋돼서 롤백되지 않습니다.**

```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void onOrderPlaced(OrderPlaced event) {
    pointService.earn(event.customerId(), event.totalAmount());
    throw new RuntimeException("포인트 적립 실패!");
    // → 새 트랜잭션(REQUIRES_NEW)만 롤백됨
    // → Order 저장 트랜잭션은 이미 COMMIT됨 → 롤백 안 됨
}
```

`TransactionPhase.BEFORE_COMMIT` 단계라면 같은 트랜잭션에서 실행되므로 예외 시 Order도 롤백됩니다.

```java
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void onOrderPlaced(OrderPlaced event) {
    throw new RuntimeException("핸들러 실패!");
    // → Order 저장 트랜잭션 롤백됨!
}
```

권장: `AFTER_COMMIT` + `REQUIRES_NEW` 조합으로 Order 저장과 핸들러를 격리. 핸들러 실패는 재시도 또는 DLQ로 처리.

</details>

---

<div align="center">

**[⬅️ 이전: Repository 구현](./05-repository-implementation.md)** | **[홈으로 🏠](../README.md)** | **[다음: 테스트 전략 ➡️](./07-testing-strategy.md)**

</div>
