# Anemic Domain Model 안티패턴 — Entity가 데이터 컨테이너가 될 때

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Anemic Domain Model이 무엇이고, 왜 DDD의 모든 장점을 무효화하는가?
- Service에 비즈니스 로직이 쌓이는 패턴이 왜 자연스럽게 발생하는가?
- Service 비대화 코드를 도메인 모델로 이전하는 구체적인 리팩터링 전략은?
- "도메인 로직이 어디에 있어야 하는가?"를 판단하는 기준은?
- Anemic Model이 테스트 어려움과 직접 연결되는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Martin Fowler는 Anemic Domain Model을 "2003년 안티패턴"으로 명명했지만 2024년 현재도 여전히 가장 흔한 패턴이다. "DDD를 한다"고 하면서 실제로는 Anemic Model을 쓰는 경우가 많다. Entity에 `@Entity` 어노테이션과 getter/setter만 있고, 모든 로직이 `xxxService`에 있다면 Anemic Model이다.

---

## 😱 흔한 실수 (Before — Anemic Entity + 비대한 Service)

```java
// Anemic Entity: 데이터 컨테이너
@Entity
public class Order {
    @Id private Long id;
    private Long customerId;
    private String status;           // "PENDING", "PAID", etc.
    private BigDecimal totalAmount;
    @OneToMany private List<OrderItem> items;

    // getter/setter만 있음 — 도메인 로직 없음
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal total) { this.totalAmount = total; }
    // ... 더 많은 getter/setter
}

// 비대한 Service: 모든 도메인 로직
@Service
@Transactional
public class OrderService {

    public void cancelOrder(Long orderId, String reason) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        // 상태 확인 로직 — Service에 있음
        if ("SHIPPED".equals(order.getStatus()) || "DELIVERED".equals(order.getStatus())) {
            throw new CannotCancelException("배송 중이거나 배송 완료된 주문은 취소 불가");
        }
        if ("CANCELLED".equals(order.getStatus())) {
            throw new AlreadyCancelledException("이미 취소된 주문입니다");
        }

        // 상태 변경 — setter로 직접
        order.setStatus("CANCELLED");
        order.setCancelledAt(LocalDateTime.now());
        order.setCancelReason(reason);

        orderRepository.save(order);

        // 이메일 발송 (Service 책임 과적)
        emailService.sendCancellationEmail(order.getCustomerId(), orderId);

        // 포인트 회수 (Service 책임 과적)
        pointService.revokePoints(order.getCustomerId(), orderId);
    }

    public BigDecimal calculateOrderTotal(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        // 계산 로직이 Service에 — Entity가 아닌 곳에
        return order.getItems().stream()
            .map(item -> item.getUnitPrice().multiply(new BigDecimal(item.getQuantity())))
            .reduce(BigDecimal.ZERO, BigDecimal::add);
    }
}
```

```
Anemic Model의 증상:
  Entity: getter/setter 외에 public 메서드가 없음
  Service 메서드: 100줄 이상이 다수
  같은 계산 로직이 여러 Service에 복사됨
  비즈니스 규칙을 알려면 Service를 뒤져야 함
  단위 테스트 없이 통합 테스트만 존재
```

---

## ✨ 올바른 접근 (After — Rich Domain Model)

```java
// Rich Domain Model: 로직이 Entity 안에
public class Order extends AggregateRoot {

    private OrderId id;
    private OrderStatus status;
    private List<OrderLine> lines;
    private Money totalAmount;

    // 도메인 메서드: 비즈니스 로직이 Entity 내부에
    public void cancel(String reason) {
        // 불변식 검증: Entity가 스스로 지킴
        if (!status.canTransitionTo(OrderStatus.CANCELLED)) {
            throw new InvalidOrderStatusTransitionException(status, OrderStatus.CANCELLED);
        }
        this.status = OrderStatus.CANCELLED;
        this.cancelledAt = LocalDateTime.now();
        this.cancelReason = reason;
        // 이벤트 발행 (이메일, 포인트는 이벤트 핸들러 담당)
        registerEvent(new OrderCancelled(this.id, reason, LocalDateTime.now()));
    }

    // 계산 로직: Entity가 스스로 계산
    public Money calculateTotal() {
        return lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO_KRW, Money::add)
            .add(shippingFee)
            .subtract(discount.amount());
    }

    // 상태 조회: 도메인 의미 있는 메서드
    public boolean isCancellable() {
        return status.canTransitionTo(OrderStatus.CANCELLED);
    }

    public boolean hasBeenShipped() {
        return status == OrderStatus.SHIPPED || status == OrderStatus.DELIVERED;
    }
}

// Thin Application Service: 오케스트레이션만
@Service
@Transactional
public class OrderApplicationService {

    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId()).orElseThrow();
        order.cancel(command.reason());  // 도메인 메서드에 위임
        orderRepository.save(order);
        order.pullDomainEvents().forEach(eventPublisher::publishEvent);
        // 끝 — Service는 조율만
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Anemic Model이 자연스럽게 발생하는 이유

```
원인 1: "Entity는 DB 테이블" 사고방식
  JPA를 처음 배우면 Entity = DB 매핑 객체
  로직은 당연히 Service에 → Anemic Model 탄생

원인 2: Transaction Script 습관
  절차적 프로그래밍 → 함수(Service)가 데이터(Entity)를 처리
  OOP 전환이 불완전하면 Anemic Model

원인 3: "Entity에 로직 넣으면 테스트 어렵다"는 오해
  사실은 반대: Entity에 로직이 있으면 더 쉬운 단위 테스트
  Service에 로직이 있으면 Mock이 많은 통합 테스트 필요

원인 4: 팀 컨벤션 부재
  "Service에 로직 넣는다" 암묵적 규칙
  새 팀원도 기존 패턴 따름 → 확산

원인 5: 조급한 DDD 적용
  "Repository 만들었으니 DDD" → Entity는 Anemic한 채로
```

### 2. Anemic Model이 DDD 장점을 무효화하는 방식

```
DDD 이점: 도메인 지식의 코드 집중
  Anemic Model: "취소 가능 조건"이 어디 있는가?
    OrderService.cancelOrder() → 300줄 파악 필요
    AdminOrderService → 다른 조건?
    ReturnService → 또 다른 조건?
  
  Rich Model: Order.isCancellable()
    → 한 곳에서 명확히

DDD 이점: 불변식 보호
  Anemic Model: order.setStatus("ANYTHING") 가능
    → 불변식 우회 가능
    → 런타임에서야 발견되는 버그
  
  Rich Model: order.cancel("이유")만 허용
    → 취소 조건 자동 검증

DDD 이점: 도메인 언어 표현
  Anemic Model: 코드 읽어도 비즈니스 이해 불가
    if ("SHIPPED".equals(order.getStatus())) { ... }
  
  Rich Model: 코드 자체가 비즈니스 언어
    if (order.hasBeenShipped()) { ... }
    order.isCancellable()
    order.cancel("reason")

DDD 이점: 빠른 단위 테스트
  Anemic Model: Service 테스트 = DB + 이메일 + 포인트 Mock 필요
  Rich Model: Entity 테스트 = 순수 Java (Mock 0)
```

### 3. 도메인 로직 위치 판단 기준

```
질문 1: "이 로직이 해당 객체의 데이터로만 결정되는가?"
  → YES: Entity/VO 메서드
  order.calculateTotal() → lines 데이터만 필요 → Order 메서드

질문 2: "이 로직이 여러 Aggregate에 걸친 계산인가?"
  → YES: Domain Service
  pricingService.calculateDiscount(customer, coupon) → 두 Aggregate

질문 3: "이 로직이 외부 시스템(DB, 이메일, API)에 의존하는가?"
  → YES: Application Service 또는 Infrastructure
  emailService.send() → 인프라
  orderRepository.save() → 인프라

"이 비즈니스 규칙이 어디 있는가?" 테스트:
  새 팀원에게 "주문 취소 조건이 뭐야?" 물었을 때
  → 코드 하나 가리키면 성공 (Order.cancel() 내부)
  → "Service 보셔야 해요..." 라면 실패 (Anemic)
```

---

## 💻 실전 코드

### Anemic → Rich Domain Model 리팩터링

```java
// Step 1: 상태 확인 로직 Entity로 이전
// Before (Service)
if ("SHIPPED".equals(order.getStatus()) || "DELIVERED".equals(order.getStatus())) {
    throw new CannotCancelException();
}

// After (Entity 메서드)
public boolean isCancellable() {
    return status != OrderStatus.SHIPPED
        && status != OrderStatus.DELIVERED
        && status != OrderStatus.CANCELLED;
}

// Step 2: 상태 변경 로직 Entity로 이전
// Before (Service)
order.setStatus("CANCELLED");

// After (Entity 도메인 메서드)
public void cancel(String reason) {
    if (!isCancellable()) throw new OrderNotCancellableException(status);
    this.status = OrderStatus.CANCELLED;
    registerEvent(new OrderCancelled(this.id, reason, LocalDateTime.now()));
}

// Step 3: 계산 로직 Entity로 이전
// Before (Service)
public BigDecimal calculateOrderTotal(Long orderId) {
    Order order = ...;
    return order.getItems().stream()
        .map(item -> item.getUnitPrice().multiply(...))
        .reduce(...);
}

// After (Entity 메서드)
public Money calculateTotal() {
    return lines.stream().map(OrderLine::subtotal).reduce(Money.ZERO_KRW, Money::add);
}
```

---

## 📊 설계 비교

```
Anemic Domain Model vs Rich Domain Model:

                Anemic                    Rich
────────────┼──────────────────────┼──────────────────────────
Entity 역할  │ 데이터 컨테이너         │ 불변식 + 로직 + 이벤트
────────────┼──────────────────────┼──────────────────────────
Service 역할 │ 모든 비즈니스 로직     │ 오케스트레이션만
────────────┼──────────────────────┼──────────────────────────
불변식 보호  │ ❌ setter로 우회 가능   │ ✅ 도메인 메서드만
────────────┼──────────────────────┼──────────────────────────
도메인 언어  │ "SHIPPED" 문자열 비교  │ order.hasBeenShipped()
────────────┼──────────────────────┼──────────────────────────
테스트 속도  │ 느림 (Mock 많음)       │ 빠름 (순수 Java)
────────────┼──────────────────────┼──────────────────────────
로직 위치   │ Service에 분산         │ Entity에 집중
```

---

## ⚖️ 트레이드오프

```
Rich Domain Model의 현실적 어려움:
  JPA와의 긴장: public setter 없으면 JPA 구성 복잡
  → protected 생성자 + 팩토리 메서드로 해결
  
  팀 학습 비용: 도메인 로직이 어디 있는지 새 패턴 학습
  → 점진적 이전 + 페어 프로그래밍

Anemic이 허용될 수 있는 경우:
  단순 CRUD 도메인 (비즈니스 규칙이 거의 없음)
  팀이 절차적 프로그래밍에 훨씬 익숙하고 학습 비용이 큰 경우
  프로토타입/MVP 단계
  → 이 경우엔 솔직하게 "Transaction Script를 쓴다"고 인정하는 것이 나음
```

---

## 📌 핵심 정리

```
Anemic Domain Model 핵심:

증상:
  Entity: getter/setter만 (도메인 메서드 없음)
  Service: 100줄 이상, 상태 확인 로직 포함
  비즈니스 규칙이 여러 Service에 복사됨
  단위 테스트가 없고 통합 테스트만 있음

해결:
  상태 확인 로직 → Entity 메서드
  상태 변경 → 도메인 메서드 (setter 제거)
  계산 로직 → Entity/VO 메서드
  부수 효과 → Domain Event + Handler

판단 기준:
  "이 로직이 해당 객체 데이터만으로 결정?"
  → YES: Entity/VO
  → NO (여러 Aggregate): Domain Service
  → NO (인프라 필요): Application Service
```

---

## 🤔 생각해볼 문제

**Q1.** "Order Entity에 너무 많은 로직이 들어가면 God Object가 되지 않는가?" 라는 반론에 어떻게 답하는가?

<details>
<summary>해설 보기</summary>

**로직을 Entity에 넣는 것이 God Object를 만드는 게 아닙니다. 경계 없이 모든 것을 넣는 것이 God Object입니다.**

```java
// God Object (안티패턴): 관련 없는 모든 것을 포함
public class Order {
    public void sendEmail() { ... }          // 인프라 책임 ❌
    public void calculateTax() { ... }       // 세금 Context 책임 ❌
    public void updateInventory() { ... }    // 재고 Context 책임 ❌
    public void notifySlack() { ... }        // 인프라 책임 ❌
}

// Rich Domain Model (올바름): 주문 Aggregate의 책임만
public class Order {
    public void cancel(String reason) { ... }           // ✅ 주문 상태 전이
    public void addLine(ProductId, quantity, price) {} // ✅ 주문 항목 관리
    public Money calculateTotal() { ... }               // ✅ 주문 금액 계산
    public boolean isCancellable() { ... }              // ✅ 주문 가능 여부
}
```

"Order가 얼마나 커야 하는가?"의 기준은 **Aggregate 경계**입니다. 주문 Aggregate의 불변식을 지키는 로직은 Order에 있어야 합니다. 이메일, 재고, 세금은 다른 Context이므로 Order가 알 필요 없습니다.

</details>

---

**Q2.** Anemic Model이 있는 대규모 레거시 시스템에서 Rich Model로 가는 첫 번째 단계는 무엇인가?

<details>
<summary>해설 보기</summary>

**가장 많이 중복되는 비즈니스 규칙을 찾아 Entity 메서드로 추출하는 것입니다.**

```java
// Step 0: 중복 코드 탐색
// OrderService.cancelOrder(): if (SHIPPED...) throw
// AdminOrderService.forceStatus(): if (SHIPPED...) throw
// ReturnService.checkCancellable(): if (SHIPPED...) throw

// Step 1: Entity에 is메서드 추가 (setter는 아직 유지)
public class Order {
    // 기존 getter/setter 유지
    
    // 새 메서드 추가 (부작용 없음)
    public boolean isCancellable() {
        return !"SHIPPED".equals(status)
            && !"DELIVERED".equals(status)
            && !"CANCELLED".equals(status);
    }
}

// Step 2: Service에서 새 메서드 사용
if (!order.isCancellable()) throw new CannotCancelException();

// Step 3: 단위 테스트 추가 (즉시 가능)
class OrderTest {
    @Test
    void shippedOrder_isNotCancellable() {
        Order order = createOrderWithStatus("SHIPPED");
        assertFalse(order.isCancellable());
    }
}
```

이렇게 부작용이 없는 `is`메서드부터 시작하면 위험 없이 점진적으로 Rich Model로 전환할 수 있습니다.

</details>

---

**Q3.** "DDD를 하면 Service가 얇아지는데, Service가 얇으면 도메인 서비스가 비대해지지 않는가?" 라는 질문에 어떻게 답하는가?

<details>
<summary>해설 보기</summary>

**도메인 서비스가 비대해지면 그것도 안티패턴입니다. 로직의 자연스러운 집이 어디인지가 핵심입니다.**

```
올바른 로직 배치:

하나의 Entity 데이터로 결정되는 로직
  → Entity 메서드 (가장 먼저 고려)
  order.calculateTotal(), order.isCancellable()

여러 Entity에 걸친 도메인 로직
  → Domain Service (필요할 때만)
  pricingService.calculateDiscount(customer, coupon)

외부 상태에 의존하는 계산 (DB, API)
  → Application Service 또는 Infrastructure
  
"도메인 서비스에 뭐든 넣기" 패턴:
  → Service 이름만 바꾼 Anemic Model
  → Entity가 데이터 컨테이너인 건 여전함

판단: "Domain Service를 쓰려는 이유가 Entity에 넣기 어색해서인가?"
  → 어색한 이유가 JPA 제약이면 해결하고 Entity에
  → 어색한 이유가 여러 Aggregate 필요하면 Domain Service 정당
```

</details>

---

<div align="center">

**[⬅️ 이전: DDD 리팩터링](../real-world-project/05-ddd-refactoring.md)** | **[홈으로 🏠](../README.md)** | **[다음: Aggregate 설계 실수 ➡️](./02-aggregate-design-mistakes.md)**

</div>
