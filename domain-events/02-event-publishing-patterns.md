# 이벤트 발행 패턴 — Aggregate 내부 수집, Application Service 발행

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Aggregate가 이벤트를 직접 발행하지 않고 내부에서 수집해야 하는 이유는?
- `ApplicationEventPublisher` vs Kafka 직접 발행 중 언제 무엇을 선택해야 하는가?
- 발행 실패 시 이벤트가 유실되는 시나리오는 어떻게 발생하는가?
- `@TransactionalEventListener`와 `@EventListener`의 차이와 선택 기준은?
- Spring Data의 `AbstractAggregateRoot`는 어떻게 이벤트를 자동 발행하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

이벤트를 "언제, 어디서, 어떻게" 발행하느냐가 신뢰성을 결정한다. Aggregate 내부에서 직접 Kafka를 호출하면 도메인 레이어가 인프라를 알게 된다. Application Service가 저장 전에 이벤트를 발행하면 저장이 실패해도 이벤트는 이미 나갔다. 저장 후 발행하다 서버가 죽으면 이벤트는 유실된다.

이 문서는 이벤트 발행의 책임을 올바르게 분배하는 방법과, 유실 없이 발행하는 패턴을 다룬다.

---

## 😱 흔한 실수 (Before — 발행 시점과 위치의 오류)

```java
// 안티패턴 1: Aggregate가 인프라(Kafka)를 직접 호출
public class Order {

    @Autowired  // ← Aggregate에 DI?! 도메인 오염
    private KafkaTemplate<String, Object> kafkaTemplate;

    public void cancel(String reason) {
        this.status = OrderStatus.CANCELLED;
        // Aggregate가 Kafka를 직접 호출 — 도메인이 인프라를 앎
        kafkaTemplate.send("order.events", new OrderCancelled(this.id, reason));
        // 테스트 시 Kafka Mock 필요, 도메인 순수성 파괴
    }
}

// 안티패턴 2: 저장 전 이벤트 발행 (데이터 불일치)
@Service
@Transactional
public class OrderService {
    public void placeOrder(PlaceOrderCommand command) {
        Order order = Order.place(...);

        // 저장 전에 이벤트 발행!
        eventPublisher.publishEvent(new OrderPlaced(order.id()));

        orderRepository.save(order);  // 만약 여기서 실패하면?
        // → Order는 DB에 없는데 OrderPlaced 이벤트는 나감
        // → 수신자가 이벤트를 처리하려 하지만 DB에 Order 없음 → 오류
    }
}

// 안티패턴 3: 트랜잭션 완료 전 이벤트 처리 (유령 데이터)
@Service
@Transactional
public class OrderService {
    public void placeOrder(PlaceOrderCommand command) {
        Order order = Order.place(...);
        orderRepository.save(order);

        // @EventListener는 같은 트랜잭션에서 즉시 실행
        eventPublisher.publishEvent(new OrderPlaced(order.id()));
        // → 핸들러가 실행되지만 트랜잭션이 아직 커밋 안 됨
        // → 핸들러에서 새 트랜잭션으로 Order를 조회하면? → 없음!
        // → LazyInitializationException 또는 데이터 없음
    }
}
```

---

## ✨ 올바른 접근 (After — 수집 → 저장 → 발행)

```java
// Step 1: Aggregate는 이벤트를 내부 컬렉션에 수집만
public class Order extends AggregateRoot {

    public static Order place(CustomerId customerId, List<OrderLine> lines,
                               Discount discount, Money shippingFee) {
        Order order = new Order(new OrderId(), customerId, PENDING, lines, ...);
        // 인프라 호출 없이 내부 리스트에 추가
        order.registerEvent(new OrderPlaced(
            order.id, customerId, OrderLineSnapshot.of(lines), order.totalAmount, LocalDateTime.now()
        ));
        return order;
    }

    public void cancel(String reason) {
        validateCancellable();
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelled(this.id, reason, LocalDateTime.now()));
        // Kafka, EventPublisher 모름 — 순수 도메인 코드
    }
}

// Step 2: Application Service — 저장 후 발행
@Service
@Transactional
public class OrderApplicationService {

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = orderFactory.create(command);
        orderRepository.save(order);  // DB 저장 먼저

        // 저장 성공 후 이벤트 꺼내서 발행
        List<DomainEvent> events = order.pullDomainEvents();
        events.forEach(eventPublisher::publishEvent);
        // @TransactionalEventListener(AFTER_COMMIT) 핸들러라면
        // 트랜잭션 커밋 후에 핸들러 실행됨

        return order.id();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Aggregate가 이벤트를 직접 발행하지 않는 이유

```
이유 1: 도메인 레이어 순수성
  Aggregate가 Kafka, Spring EventPublisher를 알면:
  → 도메인 클래스에 인프라 import
  → 도메인 단위 테스트에 Kafka Mock 필요
  → 도메인이 특정 기술(Kafka)에 종속됨

이유 2: 트랜잭션 경계 제어 불가
  Aggregate는 트랜잭션을 모름
  이벤트가 트랜잭션 중간에 발행되면:
  → 저장 전 발행 → 데이터 불일치
  → 커밋 전 발행 → 다른 트랜잭션이 없는 데이터 조회

이유 3: 여러 Aggregate 변경 시 이벤트 조율
  한 트랜잭션에서 Order + PointWallet 변경 시:
  모든 변경 후 모든 이벤트를 한 번에 발행해야
  → Application Service만 이 전체 맥락을 앎

올바른 분리:
  Aggregate: "무슨 이벤트를 발행할지" 결정 + 내부 수집
  Application Service: "언제 발행할지" 결정 + 실제 발행
```

### 2. `@EventListener` vs `@TransactionalEventListener`

```
@EventListener (즉시 실행):
  @EventListener
  public void on(OrderPlaced event) { ... }
  
  특징:
    publishEvent() 호출 즉시 실행
    발행자와 같은 트랜잭션
    핸들러 예외 → 발행자 트랜잭션 롤백
  
  문제:
    핸들러에서 새 트랜잭션으로 Order 조회 시
    아직 커밋 안 됐으므로 데이터 없음!
  
  사용 사례:
    핸들러가 같은 트랜잭션을 공유해야 할 때
    핸들러 실패 시 전체 롤백이 의도된 때

─────────────────────────────────────────────

@TransactionalEventListener (커밋 후 실행):
  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  public void on(OrderPlaced event) { ... }
  
  특징:
    트랜잭션 커밋 완료 후 실행
    별도 트랜잭션 (기본: 새 트랜잭션 없음)
    핸들러 예외 → 발행자 트랜잭션에 영향 없음 (이미 커밋됨)
  
  이점:
    핸들러 실행 시 DB에 Order가 이미 존재
    발행자 트랜잭션과 완전 분리
  
  주의:
    기본적으로 트랜잭션 없음 → @Transactional 또는 새 트랜잭션 명시 필요
    롤백 이벤트 (AFTER_ROLLBACK)도 처리 가능

  @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
  @Transactional(propagation = Propagation.REQUIRES_NEW)
  public void on(OrderPlaced event) {
      // 이 시점에는 Order가 DB에 있음 (커밋됨)
      // 새 트랜잭션으로 핸들러 로직 실행
  }

─────────────────────────────────────────────

선택 기준:
  핸들러가 발행자와 같은 DB, 즉시 실행, 실패 시 롤백 원함
  → @EventListener
  
  핸들러가 별도 트랜잭션, 커밋 후 실행, 발행자와 독립
  → @TransactionalEventListener(AFTER_COMMIT)
  
  외부 시스템(Kafka) 발행
  → @TransactionalEventListener(AFTER_COMMIT) + Outbox Pattern (신뢰성)
```

### 3. `ApplicationEventPublisher` vs Kafka 직접 발행

```
ApplicationEventPublisher (Spring 내부 이벤트):
  
  장점:
    단순 (인프라 없음)
    동기/비동기 선택 가능 (@Async)
    트랜잭션 연동 (@TransactionalEventListener)
    로컬 개발 환경 복잡도 없음
  
  한계:
    같은 JVM 내에서만 작동
    서버 재시작 시 미처리 이벤트 유실
    다른 서비스(마이크로서비스)에는 전달 불가
  
  적합한 경우:
    같은 서비스 내 Context 간 통신
    모놀리스 내 이벤트
    Kafka 도입 전 시작 단계

─────────────────────────────────────────────

Kafka (외부 메시지 브로커):

  장점:
    다른 서비스(JVM)에 전달 가능
    내구성 있는 이벤트 저장 (재시작 후에도 처리 가능)
    높은 처리량, 확장성
    이벤트 이력 유지 (이벤트 소싱 가능)
  
  한계:
    로컬 개발 환경에 Kafka 필요
    네트워크 지연
    at-least-once → 멱등성 필요
    Dual Write 문제 → Outbox Pattern 필요
  
  적합한 경우:
    Bounded Context가 별도 서비스로 분리됨
    이벤트 내구성이 중요
    높은 처리량이 필요

─────────────────────────────────────────────

선택 전략:
  Phase 1 (모놀리스): ApplicationEventPublisher + @TransactionalEventListener
  Phase 2 (서비스 분리): ApplicationEventPublisher → Kafka Relay (Outbox)
  Phase 3 (완전 분리): Kafka 직접 발행 + Outbox Pattern
```

### 4. 유실 시나리오와 방어 방법

```
시나리오 1: 저장 성공 → 서버 다운 → 이벤트 미발행
  
  orderRepository.save(order);    // 커밋
  // 서버 다운!
  eventPublisher.publishEvent();   // 실행 안 됨
  // → Order는 DB에 있지만 OrderPlaced 이벤트 없음
  // → 이메일 미발송, 포인트 미적립

시나리오 2: 저장 성공 → Kafka 발행 실패 (네트워크 오류)
  
  orderRepository.save(order);    // 커밋
  kafkaTemplate.send("order.events", event);  // 네트워크 오류!
  // → Order는 DB에 있지만 Kafka 메시지 없음

시나리오 3: 이벤트 발행 성공 → DB 저장 실패 (롤백)
  
  eventPublisher.publishEvent();  // 핸들러 실행됨
  orderRepository.save(order);    // 예외! 롤백!
  // → Order는 DB에 없지만 이벤트 핸들러는 이미 실행됨
  // → 포인트는 적립됐는데 주문은 없음

방어 방법:
  시나리오 1, 2: Outbox Pattern (다음 문서에서 상세 설명)
  시나리오 3: 저장 후 발행 (save 먼저, 성공하면 발행)
  
  @Transactional
  public OrderId placeOrder(PlaceOrderCommand command) {
      Order order = orderFactory.create(command);
      orderRepository.save(order);             // 저장 먼저
      // 저장 성공 후에만 이벤트 수집
      events.forEach(eventPublisher::publishEvent); // 그 다음 발행
      return order.id();
      // 트랜잭션 커밋 후 @TransactionalEventListener 핸들러 실행됨
  }
```

---

## 💻 실전 코드

### AggregateRoot 기반 이벤트 수집

```java
// 재사용 가능한 AggregateRoot 추상 클래스
public abstract class AggregateRoot {

    @Transient  // JPA에서 영속화 제외
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    protected void registerEvent(DomainEvent event) {
        Objects.requireNonNull(event, "이벤트는 null일 수 없습니다");
        this.domainEvents.add(event);
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = Collections.unmodifiableList(new ArrayList<>(this.domainEvents));
        this.domainEvents.clear();  // 꺼내면 비움 (중복 발행 방지)
        return events;
    }

    public boolean hasEvents() {
        return !domainEvents.isEmpty();
    }
}

// Spring Data의 AbstractAggregateRoot 대안으로 사용
@Entity
public class Order extends AggregateRoot {
    // registerEvent()로 이벤트 등록
}

// Application Service
@Service
@Transactional
public class OrderApplicationService {

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = orderFactory.create(command);
        orderRepository.save(order);

        // 이벤트 발행: 저장 성공 후에만 실행됨
        order.pullDomainEvents().forEach(eventPublisher::publishEvent);

        return order.id();
    }

    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId()).orElseThrow();
        order.cancel(command.reason());
        orderRepository.save(order);

        order.pullDomainEvents().forEach(eventPublisher::publishEvent);
    }
}
```

### `ApplicationEventPublisher` + `@TransactionalEventListener` 조합

```java
// 핸들러: 트랜잭션 커밋 후 실행 (Order가 DB에 있음을 보장)
@Component
public class OrderPlacedEventHandler {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Transactional(propagation = Propagation.REQUIRES_NEW)  // 별도 트랜잭션
    public void handleForPoints(OrderPlaced event) {
        PointWallet wallet = pointWalletRepository
            .findByCustomerId(event.customerId())
            .orElseGet(() -> PointWallet.create(event.customerId()));

        wallet.earn(PointAmount.of(event.totalAmount()));
        pointWalletRepository.save(wallet);
    }

    @TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
    @Async  // 비동기 별도 스레드
    public void handleForEmail(OrderPlaced event) {
        // 이메일 발송: 비동기, 실패해도 주문 트랜잭션 무관
        emailPort.sendOrderConfirmation(event.customerId(), event.orderId(), event.totalAmount());
    }
}

// 발행 실패 이벤트 처리 (선택적)
@Component
public class EventPublicationFailureHandler {

    @TransactionalEventListener(phase = TransactionPhase.AFTER_ROLLBACK)
    public void handleRollback(OrderPlaced event) {
        // 트랜잭션 롤백됐을 때 알림 (선택적)
        log.warn("주문 트랜잭션 롤백으로 OrderPlaced 이벤트 미발행: {}", event.orderId());
    }
}
```

---

## 📊 설계 비교

```
이벤트 발행 방법 비교:

                ApplicationEventPublisher    Kafka 직접
────────────┼────────────────────────────┼──────────────────────
범위         │ 같은 JVM                    │ 다른 서비스 가능
────────────┼────────────────────────────┼──────────────────────
내구성       │ 서버 재시작 시 유실 가능      │ 브로커에 영속
────────────┼────────────────────────────┼──────────────────────
트랜잭션 연동 │ @TransactionalEventListener │ Outbox Pattern 필요
────────────┼────────────────────────────┼──────────────────────
신뢰성       │ 낮음 (유실 가능)             │ at-least-once 보장
────────────┼────────────────────────────┼──────────────────────
복잡도       │ 낮음                        │ 높음 (Outbox, 멱등성)
────────────┼────────────────────────────┼──────────────────────
적합한 시작  │ 모놀리스, 프로토타입         │ 마이크로서비스, 프로덕션

@EventListener vs @TransactionalEventListener:

                @EventListener              @TransactionalEventListener
────────────┼────────────────────────────┼──────────────────────────
실행 시점    │ publishEvent() 즉시          │ 트랜잭션 커밋 후
────────────┼────────────────────────────┼──────────────────────────
트랜잭션     │ 발행자와 공유               │ 별도 (기본: 없음)
────────────┼────────────────────────────┼──────────────────────────
핸들러 예외  │ 발행자 롤백                  │ 발행자 무관 (이미 커밋)
────────────┼────────────────────────────┼──────────────────────────
데이터 가용성│ 커밋 전 (새 TX에서 없을 수) │ 커밋 후 (반드시 존재)
```

---

## ⚖️ 트레이드오프

```
@TransactionalEventListener의 함정:
  트랜잭션 외부에서 호출되면 이벤트 처리 안 됨
  → @Transactional 없는 메서드에서 publishEvent()하면 핸들러 실행 안 됨
  
  해결: 활성 트랜잭션이 없으면 @EventListener처럼 즉시 실행하도록 설정
  @TransactionalEventListener(fallbackExecution = true)

이벤트 수집(pullDomainEvents)과 영속화의 순서:
  save() 후 pullEvents()의 문제:
    JPA의 경우 save()는 즉시 INSERT가 아닐 수 있음
    flush() 없으면 트랜잭션 커밋 직전까지 지연
    pullEvents() 후 flush() 실패하면 이벤트 발행됐지만 저장 안 됨
  
  해결: save() → flush() → pullEvents() 순서 보장
  또는 Outbox Pattern으로 같은 트랜잭션에서 이벤트도 저장
```

---

## 📌 핵심 정리

```
이벤트 발행 패턴 핵심:

책임 분리:
  Aggregate: 이벤트를 내부에 수집 (registerEvent)
  Application Service: 저장 후 이벤트 발행 (pullEvents → publish)
  인프라: 실제 전송 (Kafka, Spring EventPublisher)

발행 순서:
  Save → (Flush) → pullEvents → publishEvent
  NOT: publishEvent → Save (데이터 불일치)

@TransactionalEventListener 권장:
  커밋 후 실행 → 데이터 반드시 존재
  발행자와 독립 트랜잭션
  핸들러 예외가 발행자에 영향 없음

ApplicationEventPublisher vs Kafka:
  같은 서비스 내 → ApplicationEventPublisher
  서비스 간, 내구성 필요 → Kafka + Outbox Pattern
  신뢰성 필요 → Outbox Pattern (다음 문서)
```

---

## 🤔 생각해볼 문제

**Q1.** Application Service에서 `orderRepository.save(order)` 이후 `order.pullDomainEvents()`를 호출하면, JPA가 실제로 DB에 저장하기 전에 이벤트를 꺼낼 수 있는가?

<details>
<summary>해설 보기</summary>

**JPA의 쓰기 지연(Write-Behind)으로 인해 실제로 가능한 문제입니다.**

```java
@Transactional
public OrderId placeOrder(PlaceOrderCommand command) {
    Order order = orderFactory.create(command);
    orderRepository.save(order);  // → EntityManager.persist() → 1차 캐시에만
    // 아직 INSERT SQL 실행 안 됨!

    List<DomainEvent> events = order.pullDomainEvents();
    events.forEach(eventPublisher::publishEvent);
    // @TransactionalEventListener라면 여기선 큐에 쌓임

    return order.id();
    // 메서드 종료 → 트랜잭션 커밋 시 flush → INSERT SQL 실행
    // 그 다음 @TransactionalEventListener 핸들러 실행
    // → 이 순서라면 OK
}
```

**잠재적 문제:** 핸들러가 `@EventListener`(즉시 실행)이고 `@Requires_New` 트랜잭션이라면, INSERT SQL 실행 전에 핸들러가 Order를 조회하려 할 때 없을 수 있습니다.

**안전한 패턴:**
```java
orderRepository.save(order);
entityManager.flush();  // 명시적 flush → INSERT SQL 즉시 실행
order.pullDomainEvents().forEach(eventPublisher::publishEvent);
// 이제 다른 트랜잭션에서도 Order 조회 가능
```

또는 `@TransactionalEventListener(AFTER_COMMIT)` 사용 → 커밋 후에는 반드시 존재.

</details>

---

**Q2.** `pullDomainEvents()`를 호출하면 이벤트 리스트가 비워진다. Application Service에서 실수로 두 번 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**두 번째 호출에서는 빈 리스트가 반환됩니다.**

```java
List<DomainEvent> events1 = order.pullDomainEvents();  // [OrderPlaced]
List<DomainEvent> events2 = order.pullDomainEvents();  // [] (빈 리스트)

events1.forEach(eventPublisher::publishEvent);  // 정상 발행
events2.forEach(eventPublisher::publishEvent);  // 아무것도 발행 안 됨
```

**실제 문제 상황:**
```java
order.pullDomainEvents().forEach(eventPublisher::publishEvent);  // 발행됨
// ...
order.pullDomainEvents().forEach(eventPublisher::publishEvent);  // 두 번째: 빈 리스트
// → 이벤트 유실처럼 보이는 버그
```

**방어 전략:**
```java
// 1. 이벤트 꺼내기와 발행을 한 곳에서
private void publishEventsOf(AggregateRoot aggregate) {
    aggregate.pullDomainEvents().forEach(eventPublisher::publishEvent);
}

// 2. AggregateRoot에서 hasEvents()로 확인
if (order.hasEvents()) {
    publishEventsOf(order);
}

// 3. Spring Data AbstractAggregateRoot 사용
// → save() 후 자동으로 한 번만 발행 (Spring Data가 관리)
```

</details>

---

**Q3.** `@Async` 핸들러는 별도 스레드에서 실행되는데, 그 스레드에서 예외가 발생하면 어떻게 감지하고 처리하는가?

<details>
<summary>해설 보기</summary>

**`AsyncUncaughtExceptionHandler`를 구현하거나 Future를 사용합니다.**

```java
// 방법 1: AsyncUncaughtExceptionHandler 구현
@Configuration
@EnableAsync
public class AsyncConfig implements AsyncConfigurer {

    @Override
    public AsyncUncaughtExceptionHandler getAsyncUncaughtExceptionHandler() {
        return (ex, method, params) -> {
            log.error("비동기 핸들러 예외 발생: method={}, params={}", method.getName(), params, ex);
            // 슬랙 알림, PagerDuty 등 운영 알림
            alertService.notifyAsyncHandlerFailure(method.getName(), ex);
        };
    }
}

// 방법 2: 핸들러 내부에서 try-catch + 재시도
@EventListener
@Async
public void handleForEmail(OrderPlaced event) {
    try {
        emailPort.sendOrderConfirmation(event.customerId(), event.totalAmount());
    } catch (Exception e) {
        log.error("이메일 발송 실패: orderId={}", event.orderId(), e);
        // 재시도 큐에 등록 또는 DLQ로
        retryQueue.enqueue(event, "email_confirmation");
    }
}

// 방법 3: @Retryable 활용
@EventListener
@Async
@Retryable(value = EmailSendException.class, maxAttempts = 3,
           backoff = @Backoff(delay = 1000, multiplier = 2))
public void handleForEmail(OrderPlaced event) {
    emailPort.sendOrderConfirmation(event.customerId(), event.totalAmount());
}

@Recover
public void recoverEmailFailure(EmailSendException e, OrderPlaced event) {
    log.error("이메일 발송 최종 실패, DLQ로 이동: orderId={}", event.orderId());
    deadLetterQueue.send(event);
}
```

</details>

---

<div align="center">

**[⬅️ 이전: Domain Event 결합도](./01-domain-event-decoupling.md)** | **[홈으로 🏠](../README.md)** | **[다음: Outbox Pattern ➡️](./03-outbox-pattern.md)**

</div>
