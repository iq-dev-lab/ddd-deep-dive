# Aggregate JPA 매핑 — `CascadeType.ALL`의 함정

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `CascadeType.ALL`이 Aggregate 경계를 넘어갈 때 어떤 의도치 않은 영속성 전파가 발생하는가?
- `FetchType.LAZY`가 Aggregate 외부에서 초기화될 때 `LazyInitializationException`이 발생하는 이유와 올바른 처리 방법은?
- Aggregate 경계가 JPA Fetch 전략을 어떻게 결정해야 하는가?
- `orphanRemoval = true`의 의미와 Aggregate 내부 객체 삭제 시 올바른 사용법은?
- 연관 관계에서 `mappedBy`와 `@JoinColumn`의 차이와 올바른 선택 기준은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

DDD의 Aggregate 경계와 JPA의 연관 관계 매핑은 긴장 관계에 있다. JPA는 `@OneToMany(cascade = CascadeType.ALL)`로 부모가 자식을 완전히 관리하게 한다. 하지만 이것이 다른 Aggregate를 향하면 의도치 않게 다른 Aggregate의 생명주기까지 관리하게 된다. "주문을 저장하면 고객도 업데이트된다"는 버그가 여기서 나온다.

---

## 😱 흔한 실수 (Before — CascadeType.ALL 남용)

```java
// 잘못된 Cascade 설정 — Aggregate 경계를 넘는 CASCADE
@Entity
public class Order {

    // OrderLine: Order 내부 → CascadeType.ALL 정상
    @OneToMany(cascade = CascadeType.ALL)
    private List<OrderLine> lines;

    // Customer: 다른 Aggregate → CascadeType.ALL 위험!
    @ManyToOne(cascade = CascadeType.ALL)  // ← 의도치 않은 CASCADE
    private Customer customer;

    // Coupon: 다른 Aggregate → CascadeType.ALL 위험!
    @ManyToOne(cascade = {CascadeType.PERSIST, CascadeType.MERGE})  // ← 여전히 위험
    private Coupon coupon;
}
```

```java
// 실제 발생하는 버그
@Service
@Transactional
public class OrderService {
    public void placeOrder(PlaceOrderCommand command) {
        Customer customer = customerRepository.findById(command.customerId()).orElseThrow();

        // customer.name을 실수로 변경
        customer.updateName("잘못된 이름");  // 테스트 중 실수

        Order order = Order.place(customer, command.items());
        orderRepository.save(order);
        // CascadeType.MERGE가 있으면 customer.name = "잘못된 이름"이 DB에 저장됨!
        // → Customer가 Order를 저장하는 과정에서 의도치 않게 변경됨
    }
}
```

```java
// LazyInitializationException 발생 패턴
@Service
public class OrderNotificationService {

    @Transactional(readOnly = true)
    public String getOrderSummary(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        return order.getSummary();  // 트랜잭션 종료
    }
}

// Controller
public ResponseEntity<String> getOrderDetail(Long orderId) {
    String summary = orderNotificationService.getOrderSummary(orderId);
    // 트랜잭션이 닫힌 후
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.lines().get(0).productName();  // ← LazyInitializationException!
    // lines가 LAZY이고 트랜잭션이 없음
}
```

---

## ✨ 올바른 접근 (After — Aggregate 경계에 맞는 JPA 설정)

```java
@Entity
@Table(name = "orders")
public class Order {

    @EmbeddedId
    private OrderId id;

    // ✅ Aggregate 내부 Entity: CascadeType.ALL + orphanRemoval
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private List<OrderLine> lines = new ArrayList<>();

    // ✅ 다른 Aggregate: CASCADE 없이 ID로만 참조
    @Embedded
    private CustomerId customerId;  // Customer 객체 참조 X, ID만

    // ✅ 다른 Aggregate: CASCADE 없이 ID로만 참조
    @Embedded
    private CouponId appliedCouponId;  // Coupon 객체 참조 X, ID만

    // Cascade 없음 → 다른 Aggregate는 자신의 Repository를 통해서만 저장
}

// OrderLine: Aggregate 내부 Entity
@Entity
@Table(name = "order_lines")
public class OrderLine {

    @EmbeddedId
    private OrderLineId id;

    @Embedded
    private ProductId productId;  // Product Aggregate → ID만

    private int quantity;

    @Embedded
    private Money unitPrice;

    protected OrderLine() {}

    // Order 내부에서만 생성 (package-private)
    OrderLine(ProductId productId, int quantity, Money unitPrice) {
        this.id = new OrderLineId();
        this.productId = productId;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    public Money subtotal() {
        return unitPrice.multiply(quantity);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. CascadeType별 의미와 Aggregate 경계

```
CascadeType 종류:

PERSIST: em.persist(parent) → child도 persist
MERGE:   em.merge(parent)   → child도 merge
REMOVE:  em.remove(parent)  → child도 remove
REFRESH: em.refresh(parent) → child도 refresh
DETACH:  em.detach(parent)  → child도 detach
ALL:     위 모두

Aggregate 내부 Entity에 적합한 설정:
  CascadeType.ALL + orphanRemoval = true
  이유:
    - Order 저장 → OrderLine도 저장 (PERSIST)
    - Order 업데이트 → OrderLine도 업데이트 (MERGE)
    - Order 삭제 → OrderLine도 삭제 (REMOVE)
    - Order에서 line 제거 → line 자동 삭제 (orphanRemoval)

다른 Aggregate에 CascadeType이 있으면 안 되는 이유:
  Order → Customer 관계에 CascadeType.MERGE가 있으면:
    Order를 저장할 때 customer 객체에 변경이 있으면 DB에 반영
    → Order Service가 Customer를 의도치 않게 수정
    → Aggregate 경계 위반

orphanRemoval의 의미:
  "부모 컬렉션에서 제거된 자식은 DB에서도 삭제"
  
  order.getLines().remove(line);  // 컬렉션에서 제거
  orderRepository.save(order);    // → DELETE FROM order_lines WHERE id = ?
  
  CascadeType.REMOVE와의 차이:
    CascadeType.REMOVE: 부모 삭제 시 자식 삭제
    orphanRemoval: 부모 컬렉션에서 분리 시 자식 삭제 (부모 존재해도)
  
  Aggregate 내부 Entity에는 둘 다 필요:
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
```

### 2. LazyInitializationException 발생 원리

```
JPA 세션(영속성 컨텍스트) 생명주기:

@Transactional 시작 → 영속성 컨텍스트 열림
    Order 조회 → lines는 프록시 객체 (실제 로딩 안 됨)
@Transactional 종료 → 영속성 컨텍스트 닫힘

트랜잭션 종료 후:
    order.lines() 접근 → 프록시 초기화 시도
    → 영속성 컨텍스트 없음 → LazyInitializationException!

발생하는 일반적 패턴:

1. Controller에서 @Transactional이 없는 메서드
   Order order = orderService.findOrder(id);  // 트랜잭션 종료됨
   order.lines();  // ← 예외!

2. JSON 직렬화 중 지연 로딩
   return order;  // Jackson이 order.lines()를 직렬화하려 함 → 예외

3. DTO 변환을 트랜잭션 외부에서
   Order order = orderRepo.findById(id);  // 트랜잭션 종료
   OrderDto dto = mapper.toDto(order);    // dto 변환 중 lines 접근 → 예외
```

### 3. Aggregate 경계가 Fetch 전략을 결정한다

```
원칙: Aggregate 경계 = 항상 함께 로드되어야 하는 범위

Aggregate 내부 → EAGER 또는 JOIN FETCH:
  Order를 로드하면 OrderLine도 항상 필요
  → EAGER 또는 @Query + JOIN FETCH

  @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true,
             fetch = FetchType.LAZY)  // LAZY이지만 항상 JOIN FETCH로 로드
  ...
  
  // Repository에서 JOIN FETCH
  @Query("SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.lines WHERE o.id = :id")
  Optional<Order> findByIdWithLines(@Param("id") OrderId id);

다른 Aggregate 참조 → 객체 참조 없음 (ID만):
  Order에서 Customer 정보가 필요하면 Application Service에서 별도 로드
  
  // Application Service
  Order order = orderRepository.findById(orderId).orElseThrow();
  Customer customer = customerRepository.findById(order.customerId()).orElseThrow();
  return OrderDetailDto.of(order, customer);
  
  → Customer 지연 로딩 문제 없음 (애초에 Order에 Customer 없음)

EAGER vs LAZY 결정:

  Aggregate Root 로드 시 항상 필요한 내부 Entity:
  → @OneToMany(fetch = FetchType.LAZY) + Repository에서 JOIN FETCH
  → EAGER는 모든 조회에 JOIN → 불필요한 데이터 로드 주의

  권장 패턴: 항상 LAZY로 선언 + 필요한 경우만 JOIN FETCH 명시
  → 성능 예측 가능, N+1 방지
```

### 4. N+1 문제와 JOIN FETCH

```
N+1 문제 발생:

List<Order> orders = orderRepository.findByCustomerId(customerId);
// → SELECT * FROM orders WHERE customer_id = ?  (1번)

orders.forEach(order -> {
    System.out.println(order.lines().size());
    // → SELECT * FROM order_lines WHERE order_id = ?  (N번)
});
// → 주문 10개 = SQL 11번 (1 + 10)

JOIN FETCH로 해결:

@Query("""
    SELECT DISTINCT o FROM Order o
    LEFT JOIN FETCH o.lines
    WHERE o.customerId = :customerId
    """)
List<Order> findByCustomerIdWithLines(@Param("customerId") CustomerId customerId);
// → SELECT o.*, ol.* FROM orders o LEFT JOIN order_lines ol ON ...
// → SQL 1번으로 Order + OrderLine 모두 로드

DISTINCT 이유:
  JOIN 결과: Order 1개 × OrderLine 5개 = 5행
  DISTINCT 없으면 Order가 5개로 중복됨
  → DISTINCT로 Order 중복 제거 (JPA 레벨)
  또는 Set<Order> 사용 (equals/hashCode 필요)
```

---

## 💻 실전 코드

### 올바른 Aggregate JPA 매핑 전체 예시

```java
@Entity
@Table(name = "orders")
public class Order {

    @EmbeddedId
    private OrderId id;

    @Enumerated(EnumType.STRING)
    @Column(name = "status", nullable = false)
    private OrderStatus status;

    // 다른 Aggregate: ID로만 참조 (CASCADE 없음)
    @AttributeOverride(name = "value", column = @Column(name = "customer_id", nullable = false))
    @Embedded
    private CustomerId customerId;

    // Aggregate 내부: CascadeType.ALL + orphanRemoval
    @OneToMany(
        cascade = CascadeType.ALL,
        orphanRemoval = true,
        fetch = FetchType.LAZY
    )
    @JoinColumn(name = "order_id", nullable = false)
    private List<OrderLine> lines = new ArrayList<>();

    // 복합 VO
    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount",   column = @Column(name = "total_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "total_currency"))
    })
    private Money totalAmount;

    @Version  // 낙관적 잠금
    private Long version;

    protected Order() {}

    public void addLine(ProductId productId, int quantity, Money unitPrice) {
        validateCanModify();
        if (lines.size() >= 10) throw new OrderLimitExceededException();

        findLineByProduct(productId).ifPresentOrElse(
            line -> line.increaseQuantity(quantity),
            () -> lines.add(new OrderLine(productId, quantity, unitPrice))
        );
        recalculateTotalAmount();
    }

    public void removeLine(ProductId productId) {
        boolean removed = lines.removeIf(l -> l.productId().equals(productId));
        if (!removed) throw new OrderLineNotFoundException(productId);
        recalculateTotalAmount();
        // orphanRemoval = true → save() 시 DB에서도 삭제됨
    }

    public List<OrderLine> lines() {
        return Collections.unmodifiableList(lines);
    }
}

// JOIN FETCH를 활용한 Repository
interface OrderJpaRepository extends JpaRepository<Order, OrderId> {

    @Query("""
        SELECT DISTINCT o FROM Order o
        LEFT JOIN FETCH o.lines
        WHERE o.id = :id
        """)
    Optional<Order> findByIdWithLines(@Param("id") OrderId id);

    @Query("""
        SELECT DISTINCT o FROM Order o
        LEFT JOIN FETCH o.lines
        WHERE o.customerId = :customerId
        ORDER BY o.createdAt DESC
        """)
    List<Order> findByCustomerIdWithLines(@Param("customerId") CustomerId customerId);
}
```

---

## 📊 설계 비교

```
Aggregate 내부 vs Aggregate 간 JPA 설정:

                Aggregate 내부           다른 Aggregate 참조
────────────┼──────────────────────┼──────────────────────────
참조 방식    │ 객체 참조 (@OneToMany) │ ID만 (@Embedded)
────────────┼──────────────────────┼──────────────────────────
Cascade     │ CascadeType.ALL       │ 없음
────────────┼──────────────────────┼──────────────────────────
orphanRemoval│ true                 │ 해당 없음
────────────┼──────────────────────┼──────────────────────────
Fetch       │ LAZY + JOIN FETCH    │ 해당 없음 (별도 Repository)
────────────┼──────────────────────┼──────────────────────────
저장        │ 부모 save()로 함께    │ 각 Aggregate 별도 save()
────────────┼──────────────────────┼──────────────────────────
삭제        │ 부모 삭제 시 함께     │ 독립적
```

---

## ⚖️ 트레이드오프

```
JOIN FETCH의 한계:
  컬렉션 2개를 동시에 JOIN FETCH하면 Hibernate 오류:
  "cannot simultaneously fetch multiple bags"
  
  해결:
  1. List 중 하나를 Set으로 변경
  2. BatchSize 설정으로 N+1을 N/배치크기+1으로 감소
     @BatchSize(size = 20)
     @OneToMany(...)
     private List<OrderLine> lines;
  3. EntityGraph 활용

FetchType.EAGER의 숨겨진 위험:
  항상 JOIN → 목록 조회 시 모든 Order의 OrderLine 전부 로드
  Order 1,000개 → OrderLine 10,000개 전부 로드
  → 페이지네이션과 EAGER는 호환 불가 (메모리에서 잘라냄)
  → 항상 LAZY + 명시적 JOIN FETCH 권장
```

---

## 📌 핵심 정리

```
Aggregate JPA 매핑 핵심:

CascadeType.ALL 규칙:
  Aggregate 내부 Entity → CascadeType.ALL + orphanRemoval = true
  다른 Aggregate → Cascade 없음 (ID 참조로 대체)

LazyInitializationException 방지:
  항상 FetchType.LAZY 선언
  Repository에서 JOIN FETCH로 명시적 로드
  트랜잭션 범위 안에서 모든 Aggregate 접근 완료

orphanRemoval:
  컬렉션에서 제거된 내부 Entity를 DB에서도 삭제
  Aggregate 내부 Entity에만 사용 (다른 Aggregate에 사용 시 의도치 않은 삭제!)

N+1 해결:
  Repository에서 @Query + JOIN FETCH
  컬렉션 여러 개 → BatchSize 또는 EntityGraph
  FetchType.EAGER는 페이지네이션과 함께 사용 금지

다른 Aggregate:
  객체 참조 없음 → LazyInitializationException 없음
  ID로만 참조 → Application Service에서 필요 시 별도 로드
```

---

## 🤔 생각해볼 문제

**Q1.** `Order`에 `CascadeType.ALL`로 `Customer`를 연관지었다가 나중에 제거하려 한다. 기존 데이터와 코드가 영향받는 부분은?

<details>
<summary>해설 보기</summary>

**코드, DB 스키마, 기존 동작 방식 모두 영향을 받습니다.**

```java
// Before: Order → Customer 객체 참조 (CascadeType.ALL)
@Entity
public class Order {
    @ManyToOne(cascade = CascadeType.ALL)
    @JoinColumn(name = "customer_id")
    private Customer customer;
}

// After: ID만 참조
@Entity
public class Order {
    @Embedded
    private CustomerId customerId;
    // customer 객체 참조 제거
}
```

**영향 범위:**

1. **DB 스키마**: `orders.customer_id`는 그대로지만 FK 제약 처리 방식 변경
2. **코드**: `order.getCustomer().getName()` → `customerRepository.findById(order.customerId())`로 교체
3. **기존 동작**: 이전에 Order 저장 시 Customer도 함께 저장됐다면 → 이제 명시적으로 CustomerRepository.save() 필요
4. **Cascade 삭제**: Order 삭제 시 Customer도 삭제되던 것 → 이제 Customer 독립 유지
5. **Service 변경**: Customer 조회가 필요한 곳에 `customerRepository.findById(order.customerId())` 추가

마이그레이션 순서: 코드 수정 → 데이터 검증 → 고아 Customer 처리 → 배포.

</details>

---

**Q2.** `orphanRemoval = true` 설정에서 `order.lines().clear()`를 호출하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**모든 `OrderLine`이 DB에서 삭제됩니다 — 의도적인 경우에만 사용해야 합니다.**

```java
@Transactional
public void clearOrderLines(OrderId orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.lines().clear();  // 모든 OrderLine 컬렉션에서 제거
    orderRepository.save(order);
    // → DELETE FROM order_lines WHERE order_id = ?
    // → 모든 항목 삭제!
}
```

외부에서 `order.lines()`가 `unmodifiableList`를 반환한다면 `clear()` 자체가 `UnsupportedOperationException`을 발생시킵니다.

```java
// 보호 패턴
public List<OrderLine> lines() {
    return Collections.unmodifiableList(lines);  // ← clear() 불가
}

// 전체 삭제가 필요하면 도메인 메서드로 명시
public void clearAllLines() {
    validateCanModify();
    this.lines.clear();  // 내부 필드에 직접 접근
    recalculateTotalAmount();
}
```

</details>

---

**Q3.** `@OneToMany`에서 `@JoinColumn`을 사용하는 것과 `mappedBy`를 사용하는 것의 차이는? Aggregate 내부 관계에서는 어느 것이 적합한가?

<details>
<summary>해설 보기</summary>

**Aggregate Root가 연관을 소유해야 하므로 `@JoinColumn`이 적합합니다.**

```java
// @JoinColumn: "이 쪽(Order)이 FK를 소유"
@OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
@JoinColumn(name = "order_id")  // order_lines.order_id FK는 Order가 관리
private List<OrderLine> lines;

// mappedBy: "저 쪽(OrderLine.order 필드)이 FK를 소유"
@OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
private List<OrderLine> lines;

// mappedBy를 쓰려면 OrderLine에 역방향 참조 필요
@Entity
public class OrderLine {
    @ManyToOne
    @JoinColumn(name = "order_id")
    private Order order;  // ← 역방향 참조 (Aggregate 경계 위반 가능성)
}
```

**Aggregate 내부에서는 `@JoinColumn` 선호:**
- `OrderLine`이 `Order`를 역참조할 필요 없음 (Aggregate Root를 통해서만 접근)
- `mappedBy`는 양방향 관계 → OrderLine이 Order를 직접 참조 → Aggregate 내부가 Root를 알게 됨
- `@JoinColumn`으로 단방향 유지 → Order만 OrderLine을 알고, OrderLine은 Order를 모름

단, Hibernate에서 `@JoinColumn` 단방향 `@OneToMany`는 불필요한 UPDATE SQL이 발생할 수 있으므로, 성능이 중요한 경우 `mappedBy` + 역참조를 허용하는 절충도 검토합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Value Object JPA 매핑](./03-value-object-jpa.md)** | **[홈으로 🏠](../README.md)** | **[다음: Repository 구현 ➡️](./05-repository-implementation.md)**

</div>
