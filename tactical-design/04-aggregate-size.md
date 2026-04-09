# Aggregate 크기 결정 — 작은 Aggregate를 선호해야 하는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 큰 Aggregate가 트랜잭션 충돌과 락 경합을 일으키는 구체적 시나리오는?
- Aggregate 간 참조를 ID로만 해야 하는 이유는 무엇인가?
- 작은 Aggregate와 Eventually Consistent 사이의 트레이드오프는?
- "이 두 객체가 같은 Aggregate여야 하는가?"를 판단하는 실용적 기준은?
- 큰 Aggregate를 작게 분리하는 리팩터링 과정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Aggregate 크기는 시스템의 처리량(throughput)과 직결된다. 큰 Aggregate는 더 많은 데이터를 잠그고(lock), 더 긴 트랜잭션을 만들며, 동시 요청이 증가할수록 충돌이 기하급수적으로 증가한다. 반면 작은 Aggregate는 잠금 범위가 작아 동시성이 높아지지만, 경계를 넘는 일관성은 Eventually Consistent로 처리해야 한다.

DDD 전문가들의 권고는 명확하다: **"가능한 한 작게, 불변식이 요구하는 만큼만."** 크게 만들어 놓고 분리하는 것보다 작게 시작하는 것이 훨씬 쉽다.

---

## 😱 흔한 실수 (Before — 너무 큰 Aggregate)

```java
// "큰 Aggregate" 안티패턴 — 모든 관련 객체를 포함
@Entity
public class Customer {  // Aggregate Root

    @Id
    private Long id;

    private String name;
    private String email;

    // 주소 목록 (주소가 수십 개가 될 수 있음)
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    private List<Address> addresses;

    // 모든 주문 (주문이 수천 개가 될 수 있음!)
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    private List<Order> orders;

    // 모든 결제 수단
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    private List<PaymentMethod> paymentMethods;

    // 포인트 이력
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    private List<PointHistory> pointHistories;

    // 리뷰
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.EAGER)
    private List<Review> reviews;
}
```

```
이 설계의 현실:

1. "고객 정보 조회" 하나에:
   고객 기본 정보 + 주문 3,000개 + 각 주문의 항목 + 결제 이력 + 포인트 이력 + 리뷰 모두 로드
   → 쿼리 수십 개, 응답 수초

2. "고객 이름 변경" 작업:
   Customer 전체를 락 → 이 고객의 주문 추가/조회도 모두 대기
   다른 사용자가 이 고객의 주문을 보려고 해도 이름 변경이 완료될 때까지 대기

3. 동시 주문 추가 시나리오:
   사용자가 두 브라우저 탭에서 동시에 주문 시도
   → 두 트랜잭션이 Customer Aggregate 전체를 잠그려 함
   → 하나만 성공, 나머지는 Optimistic Lock Exception 또는 대기
   → "동시 주문이 많을수록 실패율 증가"
```

---

## ✨ 올바른 접근 (After — 작은 Aggregate + ID 참조)

```java
// 작은 Aggregate 1: Customer — 핵심 식별 정보만
public class Customer {

    private CustomerId id;
    private String name;
    private Email email;
    private MembershipLevel level;
    private Point currentPoints;  // 현재 포인트 잔액 (이력은 별도)

    // Customer만의 불변식: 이메일은 유일, 포인트는 음수 불가
    public void changeEmail(Email newEmail) {
        this.email = newEmail;
        this.events.add(new CustomerEmailChanged(this.id, newEmail));
    }

    public void earnPoints(Point amount) {
        this.currentPoints = this.currentPoints.add(amount);
        this.events.add(new PointEarned(this.id, amount, currentPoints));
    }
}

// 작은 Aggregate 2: Order — 고객 ID로만 참조
public class Order {
    private OrderId id;
    private CustomerId customerId;  // Customer 객체 참조 ❌, ID만 ✅
    private List<OrderLine> lines;
    private OrderStatus status;
    private Money totalAmount;
    // Customer의 다른 주문을 알지 못함
}

// 작은 Aggregate 3: PointWallet — 포인트 로직만
public class PointWallet {
    private PointWalletId id;
    private CustomerId customerId;  // Customer ID로 참조
    private List<PointEntry> entries;  // 포인트 이력

    public void earn(Point amount, EarnReason reason) {
        entries.add(PointEntry.earned(amount, reason));
        this.events.add(new PointEarned(customerId, amount));
    }
}
```

```
결과:
  "고객 이름 변경": Customer만 잠금 → 주문 처리와 무관
  "주문 추가": Order Aggregate만 잠금 → 고객 정보 수정과 무관
  "동시 주문": 각 Order가 독립 → 충돌 없음 (Orders끼리 경쟁 없음)
  "고객 조회": Customer만 로드 → 빠름 (3,000개 주문 로드 없음)
```

---

## 🔬 내부 동작 원리

### 1. 큰 Aggregate의 락 경합 메커니즘

```
시나리오: 쇼핑몰에서 동시 요청 100건/초

큰 Aggregate (Customer에 Order 포함):

  요청 1: 고객 A 이름 변경 → Customer(A) 락 획득 (0.5초)
  요청 2: 고객 A 주문 추가 → Customer(A) 락 대기!
  요청 3: 고객 A 포인트 조회 → Customer(A) 락 대기!
  요청 4~100: 모두 대기 또는 타임아웃

  → 이름 변경 1건이 해당 고객의 모든 작업을 블로킹
  → 인기 고객(VIP)일수록 충돌 심화

작은 Aggregate:

  요청 1: 고객 A 이름 변경 → Customer(A) 락 (0.01초)
  요청 2: 고객 A 주문 추가 → Order(새ID) 락 (별도 자원)
  요청 3: 고객 A 포인트 조회 → PointWallet(A) 읽기 (공유 락)
  요청 4~100: 모두 독립 처리

  → 락 범위가 최소화 → 처리량 최대화

Optimistic Locking (낙관적 잠금):
  @Version 컬럼으로 충돌 감지
  큰 Aggregate에서:
    요청 1, 2가 Customer(버전=5) 동시 읽기
    요청 1이 먼저 커밋 → Customer(버전=6)
    요청 2 커밋 시도 → "버전 불일치" → OptimisticLockException
    → 재시도 필요 → 큰 Aggregate일수록 재시도 빈번
```

### 2. Aggregate 간 ID 참조의 이점

```
객체 직접 참조 (안티패턴):

@Entity
public class Order {
    @ManyToOne(fetch = FetchType.LAZY)
    private Customer customer;  // Customer 객체 직접 참조

    @ManyToOne(fetch = FetchType.LAZY)
    private Coupon coupon;      // Coupon 객체 직접 참조
}

문제:
  order.getCustomer().getName() — 지연 로딩이지만 Customer 전체 로드
  order.getCustomer().setEmail(new Email("new@example.com"))
  → Order를 통해 Customer를 수정! Aggregate 경계 침범

ID 참조 (권장):

public class Order {
    private CustomerId customerId;  // Customer Aggregate → ID만
    private CouponId appliedCouponId;  // Coupon Aggregate → ID만
    // Customer 또는 Coupon 객체 없음
}

이점:
  ① 경계 명확: Order를 통해 Customer를 수정 불가
  ② 로딩 범위: Order 로드 시 Customer 로드 없음
  ③ 서비스 분리 용이: Order Service가 Customer DB를 모름
  ④ 테스트 단순: Order 생성 시 Customer 필요 없음

ID가 아닌 객체가 필요한 경우:
  Application Service에서 필요한 Aggregate를 각각 로드:

  @Service
  public class OrderApplicationService {
      public void displayOrderDetail(OrderId orderId) {
          Order order = orderRepository.findById(orderId).orElseThrow();
          Customer customer = customerRepository.findById(order.customerId()).orElseThrow();
          // 두 Aggregate를 별도로 로드해서 DTO 조합
          return OrderDetailDto.of(order, customer);
      }
  }
```

### 3. 크기 결정 실용 기준

```
기준 1: 진짜 불변식이 있는가?
  "이 두 객체 변경을 하나의 트랜잭션으로 보장해야 하는가?"
  → YES: 같은 Aggregate (예: Order + OrderLine)
  → NO:  다른 Aggregate (예: Order + Customer)

기준 2: 동시 접근 패턴
  "이 두 개념이 서로 다른 사람/프로세스에 의해 동시에 수정될 수 있는가?"
  → YES: 다른 Aggregate 권장 (락 충돌 방지)
  → NO:  같은 Aggregate 가능

기준 3: 로딩 단위
  "이 개념을 조회할 때 항상 저 개념도 함께 필요한가?"
  → YES: 같은 Aggregate 고려
  → NO (선택적 필요): 다른 Aggregate (필요 시 별도 로드)

기준 4: 크기
  "이 Aggregate가 몇 개의 Entity와 VO를 포함하는가?"
  → 3개 이하: 작은 편 (좋음)
  → 5~10개: 재검토 필요
  → 10개 이상: 거의 확실히 너무 큰 것

기준 5: 트랜잭션 경계
  "Application Service의 @Transactional 하나에서 몇 개의 Repository를 쓰는가?"
  → 1개: 이상적
  → 2개: 재검토 (하나를 이벤트로 처리할 수 없는가?)
  → 3개+: 설계 문제 신호
```

### 4. 큰 Aggregate를 분리하는 리팩터링 과정

```
Before: Customer Aggregate (주문 목록 포함)
  Customer {
    id, name, email, level, points
    List<Order> orders    ← 분리 대상
    List<Address> addresses ← 유지 (주소는 2~3개, 불변식 있음)
  }

Step 1: 불변식 분석
  Customer ↔ Order 불변식이 있는가?
  "고객의 VIP 등급 = 주문 수가 100개 이상" → customerOrderCount로 대체 가능
  → 강한 불변식 없음 → 분리 가능

Step 2: Order를 별도 Aggregate로 분리
  Order {
    id, customerId (ID만), status, lines, totalAmount
    // Customer 직접 참조 제거
  }

Step 3: 느슨한 일관성 처리
  "고객 삭제 시 모든 주문도 삭제" → OrderDeletedByCustomerPolicy
  CustomerDeleted 이벤트 → Order 삭제 (Eventually Consistent)

Step 4: 읽기 모델 분리 (CQRS)
  "고객의 주문 목록 조회" → CustomerOrderListView (읽기 전용)
  OrderRepository.findByCustomerId() — 읽기 쿼리는 별도

After: 두 개의 작은 Aggregate
  Customer { id, name, email, level, points, List<Address>(2~3개) }
  Order { id, customerId, status, List<OrderLine> }
```

---

## 💻 실전 코드

### Optimistic Lock으로 충돌 감지

```java
@Entity
public class Order {

    @Version
    private Long version;  // 낙관적 잠금 버전

    // ... 나머지 필드

    public void addLine(ProductId productId, int quantity, Money unitPrice) {
        // 두 요청이 동시에 같은 Order를 수정하려 하면
        // 늦은 쪽이 OptimisticLockException 발생
        validateCanModify();
        lines.add(new OrderLine(productId, quantity, unitPrice));
        recalculateTotalAmount();
    }
}

// Application Service: 재시도 처리
@Service
public class OrderApplicationService {

    @Retryable(
        value = OptimisticLockingFailureException.class,
        maxAttempts = 3,
        backoff = @Backoff(delay = 100, multiplier = 2)
    )
    @Transactional
    public void addItemToOrder(OrderId orderId, AddItemCommand command) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.addLine(command.productId(), command.quantity(), command.unitPrice());
        orderRepository.save(order);
    }
}
```

### 큰 Aggregate 분리 전후 비교

```java
// Before: 하나의 큰 Aggregate
public class PurchaseOrder {
    private Long id;
    private Supplier supplier;      // Supplier Aggregate 직접 포함
    private List<PurchaseItem> items;
    private List<Receipt> receipts; // 입고 이력 직접 포함
    private List<Payment> payments; // 결제 이력 직접 포함
    // → 입고 처리 시 결제 이력까지 락, 결제 시 입고 이력까지 락
}

// After: 세 개의 작은 Aggregate
public class PurchaseOrder {  // 발주서
    private PurchaseOrderId id;
    private SupplierId supplierId;  // ID만 참조
    private List<PurchaseItem> items;
    private OrderStatus status;
}

public class GoodsReceipt {  // 입고
    private ReceiptId id;
    private PurchaseOrderId purchaseOrderId;  // ID만 참조
    private LocalDate receivedDate;
    private List<ReceivedItem> items;
}

public class PaymentVoucher {  // 지급
    private VoucherId id;
    private PurchaseOrderId purchaseOrderId;  // ID만 참조
    private Money amount;
    private LocalDate dueDate;
}

// 각 Aggregate가 독립적 → 동시 처리 가능
// 입고 담당자와 결제 담당자가 동시에 처리해도 충돌 없음
```

---

## 📊 설계 비교

```
큰 Aggregate vs 작은 Aggregate:

                큰 Aggregate           작은 Aggregate
────────────┼──────────────────────┼──────────────────────────
락 범위      │ 연관 객체 전체         │ 해당 Aggregate만
────────────┼──────────────────────┼──────────────────────────
트랜잭션 시간│ 길어짐 (더 많은 락)    │ 짧아짐
────────────┼──────────────────────┼──────────────────────────
처리량       │ 낮음 (충돌 많음)       │ 높음
────────────┼──────────────────────┼──────────────────────────
로딩 성능   │ 느림 (관련 객체 전부)  │ 빠름 (필요한 것만)
────────────┼──────────────────────┼──────────────────────────
일관성      │ 강한 일관성            │ Eventually Consistent
            │ (복잡한 불변식 가능)   │ (단순한 불변식만)
────────────┼──────────────────────┼──────────────────────────
복잡도      │ 불변식 표현 용이       │ 이벤트 처리 복잡도 추가
────────────┼──────────────────────┼──────────────────────────
테스트      │ 전체 그래프 구성 필요  │ 개별 Aggregate만
```

---

## ⚖️ 트레이드오프

```
작은 Aggregate의 비용:
  ① Eventually Consistent 복잡도
     두 Aggregate가 결국 일치해야 할 때 이벤트 처리 필요
     이벤트 처리 실패 시 불일치 → 보상 트랜잭션(Saga) 필요
  
  ② 읽기 쿼리 복잡도
     여러 Aggregate 데이터를 조합하려면 Application Service에서 여러 조회
     → CQRS 읽기 모델이 필요해짐

  ③ 일관성 허용 여부 논쟁
     "고객 포인트가 1초 지연될 수 있다" — 비즈니스가 수용 가능한가?
     → 도메인 전문가와 합의 필수

현실적 조언:
  모든 것을 Eventually Consistent로 만들 필요 없음
  "이 불일치가 비즈니스에 허용 가능한가?" 가 핵심 질문
  금융 도메인: 결제와 잔액은 강한 일관성 필요
  이커머스: 주문과 재고는 Eventually Consistent 허용 가능
```

---

## 📌 핵심 정리

```
Aggregate 크기 핵심:

원칙: 가능한 한 작게, 불변식이 요구하는 만큼만

작은 Aggregate의 이점:
  짧은 트랜잭션 → 락 경합 감소 → 처리량 증가
  독립적 로딩 → 빠른 응답
  명확한 경계 → 테스트 단순

Aggregate 간 참조: ID로만
  객체 참조 → 경계 침범, 거대한 로딩
  ID 참조 → 경계 보호, 필요 시 별도 로드

크기 판단 기준:
  진짜 불변식이 있는가? (없으면 분리)
  동시 수정 가능성이 있는가? (있으면 분리)
  로딩 시 항상 함께 필요한가? (아니면 분리)
  Entity 수가 10개 이상인가? (거의 확실히 분리 필요)

Eventually Consistent:
  Aggregate 경계를 넘는 일관성은 이벤트로
  "수 초 내 일치"가 비즈니스에 허용 가능한지 확인
```

---

## 🤔 생각해볼 문제

**Q1.** 포럼 시스템에서 `Forum`(포럼), `Post`(게시글), `Comment`(댓글), `Like`(좋아요) 중 어떤 것을 같은 Aggregate로, 어떤 것을 별도 Aggregate로 설계해야 하는가?

<details>
<summary>해설 보기</summary>

**권장 설계:**

```
Forum (별도 Aggregate): 포럼 설정, 규칙, 운영자 목록
Post (별도 Aggregate): 게시글 내용, 작성자, 상태
Comment (별도 Aggregate): 댓글 내용, 작성자, 대상 게시글 ID
Like (별도 Aggregate 또는 이벤트만): 좋아요 기록
```

**이유:**
- `Post.likeCount`와 `Like` 기록: 강한 일관성 불필요 → Like 추가 시 이벤트로 Post.likeCount 업데이트 (Eventually Consistent)
- `Comment`는 Post와 독립적으로 수정/삭제됨 → 별도 Aggregate
- 수백만 댓글이 있는 게시글에서 Post 로드 시 모든 Comment 로드는 불가

**좋아요 처리:**
```java
// Like를 별도 Aggregate로 (중복 방지 필요 시)
public class Like {
    private LikeId id;
    private PostId postId;      // Post ID만 참조
    private MemberId memberId;  // Member ID만 참조
    
    // 불변식: 한 사람이 같은 게시글에 한 번만 좋아요 가능
    // → Like Aggregate의 유일성 제약
}
```

</details>

---

**Q2.** "동시에 1,000명이 같은 상품에 주문하면 재고 차감이 충돌한다"는 문제를 Aggregate 설계로 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**Inventory Aggregate에 낙관적 잠금 또는 비동기 처리로 해결합니다.**

**방법 1: 낙관적 잠금 + 재시도**
```java
@Entity
public class Inventory {
    @Version private Long version;  // 동시 수정 감지

    public void decrease(int quantity) {
        if (this.stock < quantity) throw new OutOfStockException();
        this.stock -= quantity;
    }
}
// 1,000개 중 하나만 성공 → 나머지 재시도
```

**방법 2: 비동기 재고 차감 (Eventually Consistent)**
```java
// 주문 시 재고 예약(Reserve), 실제 차감은 비동기
order.place();  // OrderPlaced 이벤트 발행
// → Inventory가 이벤트 수신 → 순차 처리 (충돌 없음)
```

**방법 3: 파티셔닝 (고급)**
하나의 Inventory를 여러 Sub-Inventory로 분할:
- 재고 10,000개 → 10개 Sub-Inventory × 1,000개
- 각 요청이 다른 Sub-Inventory에서 처리 → 충돌 분산

대부분의 경우 방법 1(낙관적 잠금 + 재시도)이 가장 실용적입니다.

</details>

---

**Q3.** `Order`가 `customerId`만 참조하는데, "주문 목록 화면에서 고객 이름도 함께 보여줘야 한다"는 요구사항이 생겼다. 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**CQRS 읽기 모델 또는 Application Service에서 조합합니다.**

**방법 1: Application Service에서 조합 (간단한 경우)**
```java
public List<OrderSummaryDto> getOrderList(CustomerId customerId) {
    List<Order> orders = orderRepository.findByCustomerId(customerId);
    Customer customer = customerRepository.findById(customerId).orElseThrow();

    return orders.stream()
        .map(o -> OrderSummaryDto.of(o, customer.getName()))
        .collect(toList());
}
```

**방법 2: 읽기 전용 JOIN 쿼리 (CQRS 쿼리 측)**
```java
// 도메인 레이어 외부의 쿼리 레이어
@Repository
public interface OrderQueryRepository {
    @Query("""
        SELECT new com.example.OrderSummaryDto(
            o.id, o.totalAmount, o.status, c.name
        )
        FROM Order o
        JOIN Customer c ON c.id = o.customerId
        WHERE o.customerId = :customerId
        """)
    List<OrderSummaryDto> findOrderSummaryByCustomer(@Param("customerId") Long customerId);
}
```

방법 2는 도메인 레이어의 "ID 참조" 원칙을 지키면서도 읽기 성능을 최적화합니다. 쓰기(Command)는 Aggregate 경계를 엄격히 지키고, 읽기(Query)는 필요에 따라 자유롭게 JOIN합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Aggregate 설계](./03-aggregate-design.md)** | **[홈으로 🏠](../README.md)** | **[다음: Domain Service vs Application Service ➡️](./05-domain-vs-application-service.md)**

</div>
