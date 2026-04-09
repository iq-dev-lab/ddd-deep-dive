# Aggregate 설계 실수 — 너무 큰 Aggregate와 직접 객체 참조

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 너무 큰 Aggregate가 락 경합과 로딩 비용을 어떻게 유발하는가?
- Aggregate 간 직접 객체 참조가 경계를 어떻게 허무는가?
- 트랜잭션 범위가 Aggregate를 넘어가는 설계의 증상과 수정 방법은?
- "이 두 객체가 같은 Aggregate여야 하는가?"를 판단하는 실용적 체크리스트는?
- 너무 큰 Aggregate를 분리하는 리팩터링 순서는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Aggregate 경계 실수는 두 가지 극단으로 나타난다: 너무 큰 Aggregate(성능 문제)와 너무 작은 Aggregate(불변식 위반). 실제로 더 흔한 것은 "너무 큰" 쪽이다. 처음에는 관련 객체들을 하나의 Aggregate에 넣으면 관리가 편해 보이지만, 트래픽이 늘면 락 경합이 폭발하고 로딩 시간이 급격히 증가한다.

---

## 😱 흔한 실수 (Before — 너무 큰 Aggregate)

```java
// 하나의 Customer Aggregate에 모든 것을 포함
@Entity
public class Customer {

    @Id private Long id;
    private String name;
    private String email;

    // 모든 주문 — 주문이 수천 개가 될 수 있음
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    private List<Order> orders = new ArrayList<>();  // ← 수천 개 로딩!

    // 모든 배송지
    @OneToMany(cascade = CascadeType.ALL)
    private List<ShippingAddress> addresses = new ArrayList<>();

    // 포인트 이력
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    private List<PointTransaction> pointHistory = new ArrayList<>();  // ← 수천 개!

    // 모든 리뷰
    @OneToMany(cascade = CascadeType.ALL)
    private List<Review> reviews = new ArrayList<>();

    // Aggregate 간 직접 객체 참조
    @ManyToOne
    private Category preferredCategory;  // ← 다른 Aggregate!

    @ManyToOne
    private Seller favoriteSeller;       // ← 다른 Aggregate!
}
```

```
결과:

1. "고객 이름 변경" 작업:
   Customer 로드 → orders 3,000개 + pointHistory 5,000개 모두 로딩
   → 수 초 소요, 수백 MB 메모리

2. 동시 주문 시 락 경합:
   고객 A가 두 탭에서 동시 주문 시도
   → 둘 다 Customer(A) 전체를 잠그려 함
   → OptimisticLockException 또는 대기 발생

3. CascadeType.ALL + Seller 참조:
   Customer 저장 → Seller도 업데이트 (의도치 않게!)
   Customer 삭제 → Seller도 삭제! (재앙)
```

---

## ✨ 올바른 접근 (After — 작은 Aggregate + ID 참조)

```java
// Customer Aggregate: 핵심 식별 정보만
public class Customer {
    private CustomerId id;
    private String name;
    private Email email;
    private MembershipLevel membershipLevel;
    private Point currentPoints;     // 현재 잔액 (이력 아님)
    private List<Address> addresses; // 2~3개, 불변식 있음 → 같은 Aggregate

    // Category, Seller: ID로만 참조
    // Order: 별도 Aggregate

    // Customer만의 불변식
    public void changeEmail(Email newEmail) {
        this.email = newEmail;
        registerEvent(new CustomerEmailChanged(this.id, newEmail));
    }

    public void earnPoints(Point amount) {
        this.currentPoints = currentPoints.add(amount);
        registerEvent(new PointEarned(this.id, amount, currentPoints));
    }
}

// Order: 별도 Aggregate (CustomerId로만 참조)
public class Order {
    private OrderId id;
    private CustomerId customerId;   // ID만, 객체 참조 ❌
    private CategoryId categoryId;   // ID만
    private SellerId sellerId;       // ID만
    private List<OrderLine> lines;
    private OrderStatus status;
}

// PointHistory: 별도 Aggregate (필요 시)
public class PointWallet {
    private PointWalletId id;
    private CustomerId customerId;   // ID만
    private Point balance;
    private List<PointEntry> entries; // 최근 N건만 유지
}
```

---

## 🔬 내부 동작 원리

### 1. 직접 객체 참조가 경계를 허무는 방식

```
직접 참조의 문제:

@Entity
public class Order {
    @ManyToOne
    private Customer customer;  // Customer 객체 직접 참조

    // 이제 가능해진 것들:
    public String getCustomerName() {
        return customer.getName();  // Order → Customer 로딩 강제
    }

    public void updateCustomerEmail(Email newEmail) {
        customer.setEmail(newEmail);  // Order를 통해 Customer 수정!
        // Customer Aggregate의 불변식 검증 없이 직접 수정
    }
}

// Service에서 발생하는 문제
@Transactional
public void placeOrder(Long orderId, String newEmail) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.getCustomer().setEmail(newEmail);  // Order를 통해 Customer 수정
    orderRepository.save(order);
    // → CascadeType.MERGE가 있으면 Customer도 DB에 저장됨
    // → Order 저장이 Customer의 Aggregate 불변식을 우회
}

ID 참조로 해결:

@Entity
public class Order {
    @Embedded
    private CustomerId customerId;  // ID만

    // Customer 정보 필요 시 Application Service에서 별도 조회
}

// Application Service
Order order = orderRepository.findById(orderId).orElseThrow();
Customer customer = customerRepository.findById(order.customerId()).orElseThrow();
// 각 Aggregate를 각자의 Repository를 통해 접근
// Customer Aggregate의 불변식은 Customer.changeEmail()이 보장
```

### 2. 트랜잭션 범위 초과의 증상

```java
// 증상: 하나의 @Transactional에 여러 Repository 저장
@Transactional
public void completeOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.complete();
    orderRepository.save(order);                    // Order Aggregate 저장

    Customer customer = customerRepository.findById(order.getCustomerId()).orElseThrow();
    customer.earnPoints(order.getTotalAmount());
    customerRepository.save(customer);              // Customer Aggregate 저장 ←

    Inventory inventory = inventoryRepository.findByOrder(orderId).orElseThrow();
    inventory.confirm();
    inventoryRepository.save(inventory);            // Inventory Aggregate 저장 ←

    // 세 Aggregate를 하나의 트랜잭션 → 설계 재검토 신호
}

// 올바른 접근: 이벤트로 분리
@Transactional
public void completeOrder(Long orderId) {
    Order order = orderRepository.findById(orderId).orElseThrow();
    order.complete();  // OrderCompleted 이벤트 등록
    orderRepository.save(order);
    order.pullDomainEvents().forEach(eventPublisher::publishEvent);
    // 포인트 적립, 재고 확정은 이벤트 핸들러에서 별도 트랜잭션
}
```

### 3. Aggregate 크기 체크리스트

```
같은 Aggregate여야 하는 기준:
  ✅ "이 두 객체의 불변식이 항상 함께 보장돼야 하는가?"
     Order + OrderLine: "총 금액 = 항목 합계" → YES
  
  ✅ "이 두 객체가 항상 함께 로드되는가?"
     Order + OrderLine: 주문 조회 시 항목도 필요 → 가능

다른 Aggregate여야 하는 기준:
  ❌ "이 두 객체가 서로 다른 사람에 의해 동시 수정될 수 있는가?"
     Order와 Customer: 주문 추가와 고객 정보 수정 → 다른 Aggregate
  
  ❌ "한쪽이 수천/수만 개로 늘어날 수 있는가?"
     Customer와 Orders: 주문은 수천 개 가능 → 다른 Aggregate
  
  ❌ "이 두 객체 사이에 강한 불변식이 없는가?"
     Customer와 Category: 고객이 좋아하는 카테고리가 바뀐다고 고객 상태가 바뀌지 않음

실용 체크:
  □ 이 Aggregate를 로드하면 몇 개의 객체가 함께 로드되는가?
    → 10개 이상이면 재검토
  □ 이 Aggregate를 저장하면 몇 개의 테이블에 쿼리가 발생하는가?
    → 3개 이상이면 재검토
  □ 이 Aggregate에 락이 걸리면 다른 어떤 작업이 영향받는가?
    → 무관한 작업이 영향받으면 재검토
```

---

## 💻 실전 코드

### 큰 Aggregate 분리 예시

```java
// Before: Customer에 모든 것 포함
@Entity
public class Customer {
    @OneToMany(cascade = ALL)
    private List<Order> orders;         // ← 분리 대상
    @OneToMany(cascade = ALL)
    private List<PointTransaction> pts; // ← 분리 대상
    @ManyToOne
    private Seller favoriteSeller;      // ← ID로 교체
}

// After: Customer 핵심만
@Entity
public class Customer {
    @EmbeddedId private CustomerId id;
    private String name;
    private Email email;
    private MembershipLevel level;
    private Point currentPoints;  // 잔액만 (이력은 PointWallet에)
    @OneToMany(cascade = ALL, orphanRemoval = true)
    private List<Address> addresses;  // 2~3개로 제한, 불변식 있음

    @Embedded
    private SellerId favoriteSellerId;  // ID만으로 교체

    public static final int MAX_ADDRESSES = 5;

    public void addAddress(Address address) {
        if (addresses.size() >= MAX_ADDRESSES) {
            throw new AddressLimitException("배송지는 최대 " + MAX_ADDRESSES + "개");
        }
        addresses.add(address);
    }
}

// Order: 독립 Aggregate
@Entity
public class Order {
    @EmbeddedId private OrderId id;
    @Embedded private CustomerId customerId;  // Customer ID만
    // ...
}

// PointWallet: 독립 Aggregate (포인트 이력 관리)
@Entity
public class PointWallet {
    @EmbeddedId private PointWalletId id;
    @Embedded private CustomerId customerId;  // Customer ID만
    private Point balance;
    @OneToMany(cascade = ALL)
    private List<PointEntry> entries;  // 최근 100건 유지

    public void earn(Point amount, OrderId orderId) {
        if (entries.size() >= 100) {
            entries.remove(0);  // 오래된 이력 제거 (아카이브로 이동)
        }
        entries.add(PointEntry.earned(amount, orderId));
        this.balance = balance.add(amount);
    }
}
```

---

## 📊 설계 비교

```
큰 Aggregate vs 작은 Aggregate:

                큰 Aggregate           작은 Aggregate
────────────┼──────────────────────┼──────────────────────────
로딩 성능    │ 느림 (전부 로드)       │ 빠름 (필요한 것만)
────────────┼──────────────────────┼──────────────────────────
락 경합     │ 높음 (전체 락)         │ 낮음 (최소 락)
────────────┼──────────────────────┼──────────────────────────
불변식 표현  │ 복잡한 불변식 가능     │ 단순한 불변식만
────────────┼──────────────────────┼──────────────────────────
Eventually   │ 불필요               │ 필요 (이벤트)
Consistent   │                      │
────────────┼──────────────────────┼──────────────────────────
테스트       │ 전체 그래프 구성 필요  │ 개별 단위 테스트
```

---

## ⚖️ 트레이드오프

```
너무 작은 Aggregate의 위험:
  불변식이 여러 Aggregate에 걸치면 보장 불가
  "주문 항목과 총액의 일치" → 두 Aggregate로 나누면 Eventually Consistent 필요
  → 실제로 불변식이 있으면 같은 Aggregate 유지

현실적 조언:
  불변식이 요구하는 만큼만 크게 (더 이상 아니게)
  락 경합이 실제로 발생했을 때 분리 (조기 최적화 금지)
  작게 시작 → 성능 문제 발생 시 재구성이 더 쉬움
```

---

## 📌 핵심 정리

```
Aggregate 설계 실수 핵심:

너무 큰 Aggregate 증상:
  로드 시 수천 개 객체 포함
  단순 필드 변경에 전체 락
  불필요한 CascadeType.ALL 범위

직접 객체 참조 문제:
  다른 Aggregate를 @ManyToOne으로 참조
  → Order.getCustomer().setEmail() 가능
  → Aggregate 불변식 우회
  → CascadeType.ALL로 의도치 않은 저장/삭제

해결:
  다른 Aggregate는 ID로만 참조
  트랜잭션 하나 = Aggregate 하나
  경계 결정: "불변식이 함께 보장돼야 하는가?"
```

---

## 🤔 생각해볼 문제

**Q1.** 주문 취소 시 재고를 즉시 복원해야 한다는 요구사항이 있다. Order와 Inventory를 같은 Aggregate로 묶어야 하는가?

<details>
<summary>해설 보기</summary>

**묶지 않아도 됩니다. "즉시"의 비즈니스 정의를 확인하세요.**

```java
// 비즈니스: "즉시 = 수 초 이내"라면 이벤트로 충분
@Transactional
public void cancelOrder(CancelOrderCommand command) {
    Order order = orderRepository.findById(command.orderId()).orElseThrow();
    order.cancel(command.reason());  // OrderCancelled 이벤트 등록
    orderRepository.save(order);
    order.pullDomainEvents().forEach(eventPublisher::publishEvent);
    // Outbox → Kafka → 재고 Context → 재고 복원 (수 초)
}

// 진짜 "즉시 = 같은 트랜잭션"이 필요한 경우:
// 재고가 복원되지 않으면 다른 고객의 주문이 불가능해지는 경우
// → 비즈니스 전문가와 "재고 복원 지연이 허용되는가?" 확인 필수
```

대부분의 경우 수 초 내 Eventually Consistent로 충분합니다. Order와 Inventory를 같은 Aggregate로 묶으면 수백만 건의 재고를 Order와 함께 로드해야 하는 성능 재앙이 발생합니다.

</details>

---

**Q2.** `@OneToMany(cascade = CascadeType.ALL)`을 `@OneToOne` 관계에도 사용하면 위험한가?

<details>
<summary>해설 보기</summary>

**1:1 관계에서도 다른 Aggregate라면 CascadeType.ALL은 위험합니다.**

```java
// 문제: Order ↔ Payment 1:1 관계에 CascadeType.ALL
@Entity
public class Order {
    @OneToOne(cascade = CascadeType.ALL)
    private Payment payment;  // Payment는 별도 Aggregate!
}

// 위험 상황:
Order order = orderRepository.findById(id).orElseThrow();
order.setTotalAmount(newAmount);  // Order 수정
orderRepository.save(order);  // → Payment도 MERGE → Payment 업데이트됨!

// Order 삭제 시:
orderRepository.delete(order);  // → Payment도 삭제됨! 결제 이력 손실

// 올바른 설계:
@Entity
public class Order {
    @Embedded
    private PaymentId paymentId;  // ID만

    // Payment는 PaymentRepository를 통해서만 접근
}
```

CascadeType은 "같은 Aggregate 내부 관계"에만 사용하고, 다른 Aggregate 참조에는 절대 사용하지 않습니다.

</details>

---

**Q3.** 너무 큰 Aggregate를 작게 분리했더니 "고객의 총 주문 금액 조회"가 여러 Aggregate를 조회해야 해서 느려졌다. 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**읽기 모델(CQRS Query Side)로 해결합니다.**

```java
// 쓰기 측: Aggregate는 작게 유지
// Customer Aggregate: 이름, 이메일, 등급만
// Order Aggregate: 각 주문의 정보만

// 읽기 측: CustomerSummaryView 생성
@Entity
@Table(name = "customer_summary_view")
public class CustomerSummaryView {
    @Id private String customerId;
    private String name;
    private String email;
    private String membershipLevel;
    private int totalOrderCount;           // 집계
    private BigDecimal totalOrderAmount;   // 집계
    private LocalDateTime lastOrderedAt;   // 집계
}

// 이벤트로 동기화
@EventListener
@Transactional
public void on(OrderPlaced event) {
    customerSummaryViewRepository.findById(event.customerId().value())
        .ifPresent(view -> {
            view.setTotalOrderCount(view.getTotalOrderCount() + 1);
            view.setTotalOrderAmount(view.getTotalOrderAmount().add(event.totalAmount().amount()));
            view.setLastOrderedAt(event.placedAt());
            customerSummaryViewRepository.save(view);
        });
}

// 조회: 단일 쿼리로 빠르게
@GetMapping("/customers/{customerId}/summary")
public CustomerSummaryResponse getSummary(@PathVariable String customerId) {
    return customerSummaryViewRepository.findById(customerId)
        .map(CustomerSummaryResponse::from)
        .orElseThrow();
}
```

Aggregate 경계는 쓰기 성능을 위해 작게, 읽기 성능은 읽기 모델로 최적화합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Anemic Domain Model](./01-anemic-domain-model.md)** | **[홈으로 🏠](../README.md)** | **[다음: Context 경계 실수 ➡️](./03-context-boundary-mistakes.md)**

</div>
