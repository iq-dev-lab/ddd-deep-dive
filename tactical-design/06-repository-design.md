# Repository 설계 원칙 — 컬렉션처럼 동작해야 한다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- "Repository는 컬렉션처럼 동작해야 한다"는 것이 무슨 의미인가?
- 테이블당 Repository를 만들면 Aggregate 불변식이 어떻게 깨지는가?
- 도메인 레이어가 Repository 인터페이스를 소유해야 하는 이유는?
- `Add` / `Remove` / `Find` 시맨틱으로 Repository 인터페이스를 설계하는 방법은?
- JPA를 사용할 때 Aggregate Repository 설계의 주의사항은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대부분의 개발자가 Repository를 "DB 접근 객체"로 이해한다. 그래서 테이블마다 Repository를 만들고, SQL 쿼리를 감싸는 메서드들을 추가한다. 하지만 DDD에서 Repository의 역할은 다르다. Repository는 **도메인 객체의 컬렉션**처럼 행동해야 한다. `List<Order>`에서 Order를 꺼내 수정하고 다시 넣는 것처럼, Repository에서 Order를 꺼내 수정하면 자동으로 영속화되어야 한다.

이 철학이 "테이블당 Repository"가 아닌 **"Aggregate Root당 Repository"** 원칙의 근거다.

---

## 😱 흔한 실수 (Before — 테이블당 Repository)

```java
// 테이블마다 Repository를 만드는 패턴
public interface OrderRepository extends JpaRepository<Order, Long> { }
public interface OrderLineRepository extends JpaRepository<OrderLine, Long> { }
public interface OrderDiscountRepository extends JpaRepository<OrderDiscount, Long> { }

// 이 Repository들을 사용하는 Service
@Service
@Transactional
public class OrderService {

    public void addItemToOrder(Long orderId, Long productId, int quantity, BigDecimal price) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        // OrderLine을 직접 생성하고 저장 — Order의 불변식 우회
        OrderLine line = new OrderLine(orderId, productId, quantity, price);
        orderLineRepository.save(line);  // ← Aggregate Root를 통하지 않음!

        // 총 금액 재계산을 Order가 모름
        // → order.totalAmount가 실제 항목 합계와 불일치!
    }

    public void applyDiscount(Long orderId, BigDecimal discountRate) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        // 할인 정보를 직접 저장 — Order가 모르는 상태에서
        OrderDiscount discount = new OrderDiscount(orderId, discountRate);
        orderDiscountRepository.save(discount);  // ← Order 불변식 검증 없음

        // Order의 상태 변경 없이 DB에 직접 할인 정보 삽입
        // → Order.status = DRAFT인데 할인이 적용됨 (비즈니스 규칙 위반)
    }
}
```

```
결과:
  Order.totalAmount = 50,000원
  실제 OrderLine 합계 = 75,000원 (Order를 통하지 않고 직접 추가)
  → DB 불일치 상태

  "배송 중인 주문에 할인이 적용됐습니다" 버그 보고
  → SHIPPED 상태 Order에 OrderDiscount 직접 삽입이 가능했기 때문
  → 불변식: "DRAFT 상태에서만 할인 적용 가능" 이 지켜지지 않음
```

---

## ✨ 올바른 접근 (After — Aggregate Root당 Repository)

```java
// Domain Layer: Repository 인터페이스 (컬렉션 시맨틱)
public interface OrderRepository {
    void save(Order order);           // add: 저장 또는 업데이트
    void remove(OrderId id);          // remove: 삭제
    Optional<Order> findById(OrderId id);          // find by ID
    List<Order> findByCustomerId(CustomerId id);   // find by condition
}

// Domain Service / Application Service — Root를 통해서만 접근
@Service
@Transactional
public class OrderApplicationService {

    public void addItemToOrder(AddItemCommand command) {
        Order order = orderRepository.findById(command.orderId()).orElseThrow();

        // Aggregate Root의 메서드를 통해 항목 추가
        // → 불변식 검증 포함 (최대 10개, 총 금액 재계산, 상태 확인)
        order.addLine(command.productId(), command.quantity(), command.unitPrice());

        // Root를 통해 저장 → OrderLine도 함께 영속화 (Cascade)
        orderRepository.save(order);
    }

    public void applyDiscount(ApplyDiscountCommand command) {
        Order order = orderRepository.findById(command.orderId()).orElseThrow();

        // Order가 스스로 할인 적용 가능 여부 검증
        order.applyDiscount(command.discount());

        orderRepository.save(order);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 컬렉션 시맨틱이란

```
Repository = 메모리 내 컬렉션인 척하는 인터페이스

컬렉션 방식:
  List<Order> orders = new ArrayList<>();
  
  Order order = new Order(...);
  orders.add(order);               // 추가
  
  Order found = orders.stream()
      .filter(o -> o.id().equals(id))
      .findFirst().orElseThrow();  // 조회
  
  found.cancel();                  // 도메인 메서드 호출 (컬렉션이 재저장 신경 안 씀)
  
  orders.remove(order);            // 삭제

Repository 방식 (동일한 시맨틱):
  Order order = Order.place(...);
  orderRepository.save(order);     // add → 저장
  
  Order found = orderRepository.findById(id).orElseThrow(); // 조회
  
  found.cancel();                  // 도메인 메서드 호출
  orderRepository.save(found);     // 변경 반영 (또는 JPA 더티 체킹으로 자동)
  
  orderRepository.remove(id);      // 삭제

Repository가 컬렉션처럼 행동한다는 것:
  1. "어떻게 저장하는가" (SQL)를 숨김
  2. Aggregate 단위로 완전히 저장/로드
  3. 메서드 이름이 비즈니스 언어 (findByCustomerId, findPendingOrders)
  4. 기술 용어 없음 (executeQuery, updateRow 등 없음)
```

### 2. 도메인 레이어가 인터페이스를 소유하는 이유

```
의존성 방향 (DDD에서 핵심):

  잘못된 방향:
  Domain Layer → JPA Repository (Infrastructure)
  → Domain이 JPA를 알게 됨 (인프라 오염)
  → Domain 테스트 시 JPA/DB 필요

  올바른 방향 (의존성 역전):
  Infrastructure Layer → Domain Repository Interface
  → Infrastructure가 Domain을 의존 (반대)
  → Domain은 인터페이스만 알고, 구현은 Infrastructure가 제공

  패키지 구조:
  com.example.order.domain/
    OrderRepository.java    ← 인터페이스 (Domain 소유)
  
  com.example.order.infrastructure/
    JpaOrderRepository.java ← 구현체 (Infrastructure 소유)
    implements OrderRepository

  효과:
  ① Domain 테스트에 JPA 없음
     FakeOrderRepository implements OrderRepository 로 대체
  ② JPA → MongoDB 교체 시 Domain 코드 변경 없음
     JpaOrderRepository → MongoOrderRepository 교체만
  ③ Domain 레이어가 순수 Java (JPA import 없음)
```

### 3. Repository 인터페이스 설계 가이드

```
좋은 Repository 인터페이스:

public interface OrderRepository {
    // 컬렉션 시맨틱: save (add or update)
    void save(Order order);
    void saveAll(List<Order> orders);  // 배치 처리

    // 컬렉션 시맨틱: remove
    void remove(OrderId id);

    // 조회: ID 기반 (항상 필요)
    Optional<Order> findById(OrderId id);

    // 조회: 도메인 조건 기반 (비즈니스 언어)
    List<Order> findByCustomerId(CustomerId customerId);
    List<Order> findPendingOrders();
    List<Order> findOrdersToShipBefore(LocalDate date);
    boolean existsById(OrderId id);
}

나쁜 Repository 인터페이스:

public interface OrderRepository {
    // 기술 용어 노출
    void executeInsert(Order order);
    void updateOrderStatus(Long orderId, String status);  // 상태를 외부에서 직접

    // JPA 기술 노출
    @Query("SELECT o FROM Order o WHERE o.status = 'PENDING'")
    List<Order> findPendingOrdersJpql();  // 도메인 인터페이스에 JPA 어노테이션

    // 불필요한 Aggregate 부분 업데이트
    void updateTotalAmount(Long orderId, BigDecimal amount);  // 이렇게 하면 안 됨
    // → 항상 전체 Aggregate를 저장해야 함
}
```

### 4. 복잡한 쿼리는 어떻게 처리하는가 (CQRS)

```
문제: "최근 30일 내 주문 중 배송 완료된 주문의 고객별 총 금액"
      이런 복잡한 리포트 쿼리를 Repository에 넣어야 하는가?

해결: CQRS — 쓰기 모델(Repository)과 읽기 모델 분리

쓰기 측 (Repository):
  OrderRepository: 도메인 일관성이 필요한 조회만
  → findById, findByCustomerId, findPendingOrders

읽기 측 (Query Repository / Read Model):
  → 복잡한 보고서 쿼리는 별도 읽기 전용 DAO

// 읽기 전용 쿼리 (도메인 레이어 외부, 인프라/응용 레이어)
@Repository
public interface OrderReportQueryRepository {

    @Query("""
        SELECT new com.example.OrderSummaryByCustomer(
            o.customerId, SUM(o.totalAmount.amount), COUNT(o.id)
        )
        FROM Order o
        WHERE o.status = 'DELIVERED'
          AND o.deliveredAt >= :from
        GROUP BY o.customerId
        """)
    List<OrderSummaryByCustomer> findDeliveredOrderSummaryByCustomer(
        @Param("from") LocalDateTime from
    );
}

원칙:
  쓰기 Repository: 도메인 레이어에 인터페이스, Aggregate 중심
  읽기 Repository: 응용/인프라 레이어, DTO 직접 반환, 자유로운 JOIN
  
  도메인 규칙이 필요한 조회 → 쓰기 Repository (Aggregate 반환)
  화면 표시용 복잡 조회 → 읽기 Repository (DTO 반환)
```

---

## 💻 실전 코드

### InMemory Repository — 도메인 테스트용

```java
// 도메인/통합 테스트용 Fake 구현체
public class InMemoryOrderRepository implements OrderRepository {

    private final Map<OrderId, Order> store = new ConcurrentHashMap<>();
    private final AtomicLong sequence = new AtomicLong();

    @Override
    public void save(Order order) {
        store.put(order.id(), deepCopy(order));  // 컬렉션과 같이 저장
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id)).map(this::deepCopy);
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return store.values().stream()
            .filter(o -> o.customerId().equals(customerId))
            .map(this::deepCopy)
            .collect(toList());
    }

    @Override
    public void remove(OrderId id) {
        store.remove(id);
    }

    @Override
    public boolean existsById(OrderId id) {
        return store.containsKey(id);
    }

    // 불변 보장: 저장/반환 시 깊은 복사
    private Order deepCopy(Order order) {
        // 실제로는 직렬화/역직렬화 또는 복사 생성자
        return order;  // 불변 Aggregate라면 참조 공유 OK
    }

    public int size() { return store.size(); }
    public void clear() { store.clear(); }
}

// 테스트에서 사용
class OrderApplicationServiceTest {

    private final InMemoryOrderRepository orderRepository = new InMemoryOrderRepository();
    private final FakeEventPublisher eventPublisher = new FakeEventPublisher();
    private final OrderApplicationService sut = new OrderApplicationService(
        orderRepository, eventPublisher, /* ... */
    );

    @Test
    void placeOrder_savesOrderAndPublishesEvent() {
        sut.placeOrder(validCommand());

        assertThat(orderRepository.size()).isEqualTo(1);
        assertThat(eventPublisher.published()).hasSize(1);
        assertThat(eventPublisher.published().get(0)).isInstanceOf(OrderPlaced.class);
    }
}
```

### JPA 구현체 — Aggregate 경계 유지

```java
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final JpaOrderEntityRepository jpaRepo;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        jpaRepo.save(entity);  // Cascade로 OrderLine도 함께 저장
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepo.findByIdWithLines(id.value())  // JOIN FETCH로 함께 로드
            .map(mapper::toDomain);
    }
}

// JPA 엔티티 (Domain 클래스와 분리)
@Entity
@Table(name = "orders")
class OrderJpaEntity {

    @Id
    private Long id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    // OrderLine: Cascade로 함께 저장/삭제 (같은 Aggregate)
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private List<OrderLineJpaEntity> lines;

    // CustomerId: Value Object → 컬럼으로 매핑
    @Column(name = "customer_id")
    private Long customerId;
}

interface JpaOrderEntityRepository extends JpaRepository<OrderJpaEntity, Long> {

    // N+1 방지: JOIN FETCH로 한 번에 로드
    @Query("SELECT DISTINCT o FROM OrderJpaEntity o LEFT JOIN FETCH o.lines WHERE o.id = :id")
    Optional<OrderJpaEntity> findByIdWithLines(@Param("id") Long id);
}
```

---

## 📊 설계 비교

```
테이블당 Repository vs Aggregate당 Repository:

                테이블당 Repository       Aggregate당 Repository
────────────┼──────────────────────┼──────────────────────────
불변식 보호  │ ❌ 외부에서 우회 가능   │ ✅ Root를 통해서만 저장
────────────┼──────────────────────┼──────────────────────────
저장 단위   │ 테이블별 개별 저장      │ Aggregate 전체 한 번에
────────────┼──────────────────────┼──────────────────────────
테스트      │ 인프라 없이 테스트 어려움│ InMemory로 도메인 테스트
────────────┼──────────────────────┼──────────────────────────
도메인 오염  │ 높음 (JPA 기술 침투)  │ 낮음 (인터페이스 분리)
────────────┼──────────────────────┼──────────────────────────
조회 편의   │ 유연한 쿼리 가능       │ 복잡 쿼리는 읽기 모델로
────────────┼──────────────────────┼──────────────────────────
기술 교체   │ Domain 코드 영향       │ Infrastructure만 교체
```

---

## ⚖️ 트레이드오프

```
Aggregate Repository의 한계:
  ① 복잡한 쿼리 표현 어려움
     "고객별 월별 주문 통계" → 도메인 Repository로 표현 어색
     → CQRS 읽기 모델로 분리

  ② 큰 Aggregate의 로딩 비용
     Aggregate 전체를 항상 로드
     1,000개 OrderLine이 있으면 매번 1,000개 로드
     → Aggregate 크기를 작게 유지해야 하는 또 다른 이유

  ③ JPA와의 임피던스 불일치
     JPA는 Entity와 영속성을 직접 연결
     순수 Domain Repository 구현 시 JPA Entity와 Domain 객체 이중 유지 필요
     → 현실적 타협: Domain 클래스에 @Entity 붙이기 (완전한 분리 포기)

현실적 타협:
  소규모 팀: Domain 클래스에 @Entity 직접 (JpaRepository 인터페이스 상속도 허용)
  대규모/복잡: Domain 클래스 ↔ JPA Entity 분리 + Mapper
  중간: Spring Data JPA를 감싸는 어댑터 패턴
```

---

## 📌 핵심 정리

```
Repository 설계 핵심:

컬렉션 시맨틱:
  Repository = 도메인 객체의 컬렉션
  save() = add or update
  remove() = delete
  find*() = 비즈니스 언어 기반 조회

Aggregate Root당 하나의 Repository:
  OrderRepository → OrderLine, OrderDiscount 없음
  → 저장/조회는 항상 Order Root를 통해
  → 불변식이 Root에서 보호됨

도메인 레이어가 인터페이스 소유:
  OrderRepository (인터페이스) → Domain Layer
  JpaOrderRepository (구현체) → Infrastructure Layer
  → 의존성 역전: 인프라가 도메인을 의존

복잡 쿼리는 읽기 모델로:
  쓰기: Aggregate Repository (도메인 일관성)
  읽기: Query Repository (JOIN, GROUP BY, DTO 반환)
  → CQRS의 핵심 패턴
```

---

## 🤔 생각해볼 문제

**Q1.** `findByStatus("PENDING")`과 `findPendingOrders()`의 차이는 무엇이고, 왜 후자가 더 나은가?

<details>
<summary>해설 보기</summary>

**`findByStatus("PENDING")`의 문제:**

1. **String 상수 노출**: "PENDING"이 비즈니스 의미 없는 기술 문자열
2. **타입 안전 없음**: "PENDIG"처럼 오타 시 런타임 오류
3. **도메인 언어 부재**: Repository를 읽는 사람이 "PENDING"이 무슨 의미인지 모름

**`findPendingOrders()`의 장점:**
```java
// 도메인 언어로 표현된 인터페이스
public interface OrderRepository {
    List<Order> findPendingOrders();          // "접수 대기 중인 주문"
    List<Order> findOrdersToShip();           // "배송 준비가 필요한 주문"
    List<Order> findExpiredOrders(LocalDate today);  // "만료된 주문"
}

// 구현체에서 실제 조건 표현 (인프라 세부사항)
public Optional<Order> findPendingOrders() {
    return jpaRepo.findByStatusIn(
        List.of(OrderStatus.PENDING, OrderStatus.AWAITING_PAYMENT)
    );
}
```

"접수 대기 중인 주문"이 `PENDING` 상태만인지, `PENDING + AWAITING_PAYMENT`인지는 구현 세부사항입니다. 인터페이스는 비즈니스 의도만 표현하고, 구현체가 실제 쿼리 조건을 결정합니다.

</details>

---

**Q2.** Repository에서 조회한 Aggregate을 수정한 후 `save()`를 호출하지 않으면 어떻게 되는가? JPA와 In-Memory Repository의 동작이 다른가?

<details>
<summary>해설 보기</summary>

**JPA와 In-Memory Repository의 동작이 다릅니다.**

**JPA 환경 (`@Transactional` 있는 경우):**
```java
@Transactional
public void cancelOrder(OrderId orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.cancel();
    // orderRepository.save(order)를 호출하지 않아도...
    // JPA 더티 체킹(Dirty Checking)으로 트랜잭션 종료 시 자동 UPDATE 실행됨!
}
```

이것은 편리하지만, Repository 인터페이스를 "컬렉션"처럼 보이게 만들 수 있습니다. 도메인 레이어의 의도를 명시적으로 표현하기 위해 `save()`를 명시적으로 호출하는 것이 권장됩니다.

**In-Memory Repository:**
```java
public void cancelOrder(OrderId orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.cancel();
    // save()를 호출하지 않으면 변경이 반영되지 않음!
    // In-Memory Repository는 JPA 더티 체킹 없음
}
// → 테스트에서 통과했는데 운영에서 실패하는 버그 가능
```

**권장 방식:**
```java
// 명시적 save() 호출 — JPA와 In-Memory 양쪽에서 동일하게 동작
order.cancel();
orderRepository.save(order);  // 명시적 표현
```

일관성 있는 코드를 위해 JPA 더티 체킹에 의존하지 않고 명시적 `save()`를 호출하는 것이 Domain-driven 관점에서 더 명확합니다.

</details>

---

**Q3.** Repository에 페이지네이션(`findAll(Pageable pageable)`)을 추가해야 할 때 도메인 레이어를 오염시키지 않는 방법은?

<details>
<summary>해설 보기</summary>

**도메인 Repository와 읽기 쿼리 DAO를 분리합니다.**

```java
// Domain Layer Repository (도메인 일관성 쿼리)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId id);
    // Pageable 없음 — 도메인 개념 아님
}

// Infrastructure/Application Layer (읽기 전용 쿼리)
// Spring Data가 제공하는 Pageable을 여기서만 사용
@Repository
public interface OrderListQueryRepository {

    Page<OrderListItemDto> findOrdersByCustomer(
        @Param("customerId") Long customerId,
        Pageable pageable
    );
}
```

**이유:**
- `Pageable`은 Spring Framework의 개념 → 도메인 레이어를 Spring에 오염
- 도메인 Repository의 목적은 불변식 보장이 필요한 Aggregate 조회
- 화면 표시용 목록 조회는 읽기 모델의 책임

**만약 도메인 인터페이스에 페이지네이션을 표현해야 한다면:**
```java
// 도메인 개념으로 페이지 표현
public record Page<T>(List<T> items, int totalCount, int pageNumber, int pageSize) {}

public interface OrderRepository {
    Page<Order> findByCustomerId(CustomerId id, int page, int size);
    // Spring의 Pageable 대신 도메인 정의 Page 사용
}
```

이렇게 하면 도메인 레이어가 Spring에 의존하지 않습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Domain Service vs Application Service](./05-domain-vs-application-service.md)** | **[홈으로 🏠](../README.md)** | **[다음: Factory 패턴 ➡️](./07-factory-pattern.md)**

</div>
