# 주문 Aggregate 설계 — 불변식과 상태 전이 캡슐화

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `Order` / `OrderLine` / `OrderStatus`의 핵심 불변식은 무엇인가?
- 주문 상태 전이 로직을 Aggregate 내부에 캡슐화하는 구체적인 구현은?
- 상태 전이 불변식을 단위 테스트로 어떻게 검증하는가?
- "결제 완료 전 배송 불가" 같은 불변식을 코드로 어떻게 강제하는가?
- 주문 취소와 부분 취소의 불변식 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

전자상거래 주문 시스템에서 가장 흔한 버그는 "배송 중인 주문이 취소됐다", "이미 취소된 주문에 결제가 됐다" 같은 상태 불일치다. 이것은 상태 전이 로직이 여러 Service에 흩어져 있을 때 발생한다. Aggregate가 자신의 상태 전이를 캡슐화하면 이런 버그를 컴파일 타임에 가깝게 방지할 수 있다.

---

## 😱 흔한 실수 (Before — 상태 전이 로직이 Service에 분산)

```java
// 상태 관리가 Service에 흩어진 안티패턴
@Service
public class OrderService {
    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        // 상태 확인이 Service에 있음 — 다른 Service에서 우회 가능
        if (order.getStatus().equals("SHIPPED")) {
            throw new CannotCancelException();
        }
        order.setStatus("CANCELLED");  // setter로 직접 변경
        orderRepository.save(order);
    }
}

@Service
public class AdminOrderService {
    public void forceCancel(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        // 이쪽에는 SHIPPED 확인이 없음 → 배송 중 취소 가능!
        order.setStatus("CANCELLED");
        orderRepository.save(order);
    }
}
```

---

## ✨ 올바른 접근 (After — Aggregate가 상태 전이 캡슐화)

```java
// OrderStatus: 유효한 전이 규칙 포함
public enum OrderStatus {
    PENDING {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return next == PAID || next == CANCELLED;
        }
    },
    PAID {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return next == PREPARING || next == CANCELLED || next == REFUNDING;
        }
    },
    PREPARING {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return next == SHIPPED;
        }
    },
    SHIPPED {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return next == DELIVERED || next == RETURN_REQUESTED;
        }
    },
    DELIVERED {
        @Override public boolean canTransitionTo(OrderStatus next) {
            return next == RETURN_REQUESTED;
        }
    },
    CANCELLED, REFUNDING, REFUNDED, RETURN_REQUESTED, RETURNED;

    public boolean canTransitionTo(OrderStatus next) {
        return false;  // 기본값: 전이 불가
    }
}

// Order Aggregate: 상태 전이 로직 캡슐화
public class Order extends AggregateRoot {

    private OrderId id;
    private OrderStatus status;
    private List<OrderLine> lines;
    private Money totalAmount;

    // 결제 완료: PENDING → PAID
    public void confirmPayment(PaymentId paymentId) {
        transitionTo(OrderStatus.PAID);  // 전이 규칙 검증 포함
        this.paymentId = paymentId;
        registerEvent(new PaymentConfirmed(this.id, paymentId, this.totalAmount));
    }

    // 배송 시작: PREPARING → SHIPPED
    public void ship(TrackingNumber trackingNumber) {
        transitionTo(OrderStatus.SHIPPED);
        this.trackingNumber = trackingNumber;
        registerEvent(new OrderShipped(this.id, trackingNumber, LocalDateTime.now()));
    }

    // 취소: PENDING 또는 PAID에서만 가능
    public void cancel(String reason) {
        transitionTo(OrderStatus.CANCELLED);
        registerEvent(new OrderCancelled(this.id, reason, LocalDateTime.now()));
    }

    // 공통 전이 로직
    private void transitionTo(OrderStatus next) {
        if (!this.status.canTransitionTo(next)) {
            throw new InvalidOrderStatusTransitionException(
                this.id, this.status, next
            );
        }
        OrderStatus previous = this.status;
        this.status = next;
        registerEvent(new OrderStatusChanged(this.id, previous, next));
    }

    // 불변식: 항목 최소 1개
    public static Order place(CustomerId customerId, List<OrderLine> lines,
                               Discount discount, Money shippingFee) {
        if (lines == null || lines.isEmpty()) {
            throw new EmptyOrderException("주문 항목은 최소 1개 이상이어야 합니다");
        }
        if (lines.size() > 10) {
            throw new OrderLimitExceededException("주문 항목은 최대 10개입니다");
        }
        // ...
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 주문 상태 전이 다이어그램

```
주문 상태 전이:

[PENDING] ──결제확인──▶ [PAID] ──출고준비──▶ [PREPARING] ──발송──▶ [SHIPPED]
    │                    │                                              │
    │취소                │취소/환불요청                                 │
    ▼                    ▼                                              ▼
[CANCELLED]         [REFUNDING]                                   [DELIVERED]
                        │                                              │
                        ▼                                              │반품요청
                    [REFUNDED]                                         ▼
                                                              [RETURN_REQUESTED]
                                                                       │
                                                                       ▼
                                                                  [RETURNED]

핵심 불변식:
  SHIPPED, DELIVERED에서는 취소 불가 → 반품 프로세스로
  PENDING에서는 결제 또는 취소만 가능
  PAID에서는 환불 또는 출고 준비만 가능
  이미 CANCELLED/RETURNED에서는 모든 전이 불가
```

### 2. 불변식 설계 패턴

```java
// 불변식 1: 주문 금액 = 항목 소계 합 + 배송비 - 할인
private void recalculateTotal(Discount discount, Money shippingFee) {
    Money itemsTotal = lines.stream()
        .map(OrderLine::subtotal)
        .reduce(Money.ZERO_KRW, Money::add);
    this.totalAmount = itemsTotal.subtract(discount.applyTo(itemsTotal)).add(shippingFee);
    // 이 불변식은 항목 추가/제거, 할인 적용마다 자동 유지
}

// 불변식 2: 동일 상품은 하나의 항목으로 병합
public void addItem(ProductId productId, int quantity, Money unitPrice) {
    validateModifiable();
    Optional<OrderLine> existing = findLineByProduct(productId);
    if (existing.isPresent()) {
        existing.get().increaseQuantity(quantity);  // 수량 증가
    } else {
        if (lines.size() >= MAX_LINES) throw new OrderLimitExceededException();
        lines.add(new OrderLine(productId, quantity, unitPrice));
    }
    recalculateTotal(this.discount, this.shippingFee);
}

// 불변식 3: 수정 가능 상태 검증
private void validateModifiable() {
    if (status != OrderStatus.PENDING) {
        throw new OrderNotModifiableException(
            "PENDING 상태에서만 주문 항목을 수정할 수 있습니다. 현재 상태: " + status
        );
    }
}
```

### 3. OrderLine 불변식

```java
public class OrderLine {

    private final ProductId productId;
    private final String productNameSnapshot;
    private final Money unitPriceSnapshot;
    private int quantity;

    public OrderLine(ProductId productId, String productName,
                     Money unitPrice, int quantity) {
        Objects.requireNonNull(productId);
        Objects.requireNonNull(productName);
        Objects.requireNonNull(unitPrice);
        if (quantity <= 0) {
            throw new IllegalArgumentException("수량은 양수여야 합니다: " + quantity);
        }
        if (unitPrice.isZero()) {
            throw new IllegalArgumentException("단가는 0이 될 수 없습니다");
        }
        this.productId = productId;
        this.productNameSnapshot = productName;
        this.unitPriceSnapshot = unitPrice;
        this.quantity = quantity;
    }

    // 수량 변경은 Order를 통해서만 (package-private)
    void increaseQuantity(int additional) {
        if (additional <= 0) throw new IllegalArgumentException();
        this.quantity += additional;
    }

    void decreaseQuantity(int amount) {
        if (amount >= quantity) throw new InsufficientQuantityException();
        this.quantity -= amount;
    }

    public Money subtotal() {
        return unitPriceSnapshot.multiply(quantity);
    }
}
```

---

## 💻 실전 코드

### 상태 전이 불변식 단위 테스트 전체

```java
class OrderStatusTransitionTest {

    @Test
    void pending_canTransitionTo_paid() {
        Order order = OrderBuilder.anOrder().build();  // PENDING
        assertDoesNotThrow(() -> order.confirmPayment(new PaymentId()));
        assertThat(order.status()).isEqualTo(OrderStatus.PAID);
    }

    @Test
    void pending_canTransitionTo_cancelled() {
        Order order = OrderBuilder.anOrder().build();
        assertDoesNotThrow(() -> order.cancel("고객 요청"));
        assertThat(order.status()).isEqualTo(OrderStatus.CANCELLED);
    }

    @Test
    void shipped_cannotBeCancelled() {
        Order order = OrderBuilder.anOrder().shipped();
        assertThatThrownBy(() -> order.cancel("취소 시도"))
            .isInstanceOf(InvalidOrderStatusTransitionException.class)
            .hasMessageContaining("SHIPPED")
            .hasMessageContaining("CANCELLED");
    }

    @Test
    void delivered_cannotBeCancelled() {
        Order order = OrderBuilder.anOrder().delivered();
        assertThatThrownBy(() -> order.cancel("취소"))
            .isInstanceOf(InvalidOrderStatusTransitionException.class);
    }

    @Test
    void cancelled_cannotTransitionToAnyStatus() {
        Order order = OrderBuilder.anOrder().cancelled();
        assertThatThrownBy(() -> order.confirmPayment(new PaymentId()))
            .isInstanceOf(InvalidOrderStatusTransitionException.class);
        assertThatThrownBy(() -> order.cancel("다시 취소"))
            .isInstanceOf(InvalidOrderStatusTransitionException.class);
    }

    @Nested
    class OrderLineInvariantTest {

        @Test
        void addItem_sameProduct_mergesIntoSingleLine() {
            Order order = OrderBuilder.anOrder().build();
            ProductId product = new ProductId(1L);
            order.addItem(product, 2, Money.ofKrw(10_000));
            order.addItem(product, 3, Money.ofKrw(10_000));  // 동일 상품

            assertThat(order.lines()).hasSize(1);
            assertThat(order.lines().get(0).quantity()).isEqualTo(5);  // 합산
        }

        @Test
        void addItem_exceedsMax_throwsException() {
            Order order = OrderBuilder.anOrder().build();
            for (int i = 0; i < 10; i++) {
                order.addItem(new ProductId((long)i), 1, Money.ofKrw(1_000));
            }
            assertThatThrownBy(() -> order.addItem(new ProductId(99L), 1, Money.ofKrw(1_000)))
                .isInstanceOf(OrderLimitExceededException.class);
        }

        @Test
        void totalAmount_updatesAfterItemChange() {
            Order order = OrderBuilder.anOrder().build();
            Money initialTotal = order.totalAmount();
            order.addItem(new ProductId(99L), 1, Money.ofKrw(5_000));
            assertThat(order.totalAmount()).isGreaterThan(initialTotal);
        }
    }
}
```

---

## 📊 설계 비교

```
상태 전이 로직 위치 비교:

                Service에 분산          Aggregate에 캡슐화
────────────┼──────────────────────┼──────────────────────────
불변식 보장  │ Service마다 중복 검사 │ Aggregate 메서드에 집중
────────────┼──────────────────────┼──────────────────────────
우회 가능성  │ ❌ setter로 우회 가능  │ ✅ 메서드만 경로
────────────┼──────────────────────┼──────────────────────────
테스트      │ Service Mock 필요     │ 순수 Java 단위 테스트
────────────┼──────────────────────┼──────────────────────────
코드 중복   │ 각 Service에서 중복   │ 한 곳에서 정의
────────────┼──────────────────────┼──────────────────────────
가독성      │ 비즈니스 규칙 분산     │ 도메인 모델에 집중
```

---

## ⚖️ 트레이드오프

```
상태 전이 enum의 복잡도:
  상태가 많아질수록 enum이 복잡해짐
  → 상태 머신 라이브러리 도입 고려 (Spring Statemachine)
  → 단, 순수 도메인 코드가 외부 의존 증가

부분 취소의 어려움:
  "3개 중 2개만 취소" → Order 수준의 상태 전이가 아닌
  OrderLine 수준의 상태 관리 필요
  → OrderLine에 별도 상태 추가 (ACTIVE, CANCELLED)
  → Order 불변식: "모든 항목이 CANCELLED면 Order도 CANCELLED"
```

---

## 📌 핵심 정리

```
주문 Aggregate 설계 핵심:

불변식 목록:
  항목 수: 1 ≤ lines ≤ 10
  총 금액 = sum(항목 소계) + 배송비 - 할인
  상태 전이: 허용된 경로만 (enum 또는 상태 머신)
  수정 가능: PENDING 상태에서만

상태 전이 캡슐화:
  OrderStatus enum에 canTransitionTo() 정의
  Order.transitionTo()가 검증 + 이벤트 발행
  외부에서 setStatus() 없음 — 도메인 메서드만

테스트:
  각 상태에서 허용/불허 전이 망라
  경계값 테스트 (최소 1개, 최대 10개)
  복합 불변식 (항목 변경 → 총 금액 재계산)
```

---

## 🤔 생각해볼 문제

**Q1.** "배송 중인 주문을 취소하려는 고객"을 어떻게 처리해야 하는가? 비즈니스 프로세스와 Aggregate 불변식의 균형은?

<details>
<summary>해설 보기</summary>

**Aggregate 불변식은 유지하되, 별도 반품 프로세스로 처리합니다.**

```java
// 배송 중 취소 → 반품 요청으로 전환
public class Order {
    public void requestReturn(String reason) {
        // SHIPPED, DELIVERED에서만 반품 가능
        if (status != OrderStatus.SHIPPED && status != OrderStatus.DELIVERED) {
            throw new ReturnNotAllowedException("배송 완료 또는 배송 중인 주문만 반품 요청 가능");
        }
        transitionTo(OrderStatus.RETURN_REQUESTED);
        registerEvent(new ReturnRequested(this.id, reason, LocalDateTime.now()));
    }
}

// 비즈니스 프로세스:
// 고객: "취소"를 원하지만 → 실제로는 반품 프로세스
// 배송사에 회수 요청 → 입고 확인 → 환불
// UX: "반품/취소 신청" 버튼이 내부적으로 requestReturn() 호출
```

불변식이 비즈니스 현실을 올바르게 표현합니다: 이미 출발한 택배를 "취소"할 수 없고, 반품 프로세스를 거쳐야 합니다.

</details>

---

**Q2.** `OrderStatus` enum의 `canTransitionTo()` 메서드가 복잡해지면 어떻게 리팩터링하는가?

<details>
<summary>해설 보기</summary>

**상태 전이 규칙을 별도 클래스(Transition Rule)로 추출합니다.**

```java
// 상태 전이 규칙을 별도 관리
public class OrderStatusTransitionRule {

    private static final Map<OrderStatus, Set<OrderStatus>> ALLOWED_TRANSITIONS = Map.of(
        OrderStatus.PENDING,  Set.of(PAID, CANCELLED),
        OrderStatus.PAID,     Set.of(PREPARING, CANCELLED, REFUNDING),
        OrderStatus.PREPARING, Set.of(SHIPPED),
        OrderStatus.SHIPPED,  Set.of(DELIVERED, RETURN_REQUESTED),
        OrderStatus.DELIVERED, Set.of(RETURN_REQUESTED)
    );

    public static void validate(OrderStatus current, OrderStatus next) {
        Set<OrderStatus> allowed = ALLOWED_TRANSITIONS.getOrDefault(current, Set.of());
        if (!allowed.contains(next)) {
            throw new InvalidOrderStatusTransitionException(current, next,
                "허용된 전이: " + allowed);
        }
    }
}

// Order에서 사용
private void transitionTo(OrderStatus next) {
    OrderStatusTransitionRule.validate(this.status, next);
    this.status = next;
}
```

규칙이 더 복잡해지면 Spring StateMachine이나 별도 DSL 고려.

</details>

---

**Q3.** 쿠폰 할인이 적용된 주문을 취소할 때, 쿠폰은 어떻게 처리해야 하는가? 이것이 Order Aggregate의 책임인가?

<details>
<summary>해설 보기</summary>

**쿠폰 복원은 Order Aggregate의 책임이 아닙니다 — 이벤트로 처리합니다.**

```java
// Order가 취소될 때: OrderCancelled 이벤트 발행
public void cancel(String reason) {
    transitionTo(OrderStatus.CANCELLED);
    registerEvent(new OrderCancelled(
        this.id, reason, this.appliedCouponId,  // 쿠폰 ID 포함
        LocalDateTime.now()
    ));
    // Order는 쿠폰 복원 방법을 모름 (쿠폰 Context 모름)
}

// 쿠폰 Context: OrderCancelled 이벤트 수신 → 쿠폰 복원
@EventListener
public void on(OrderCancelled event) {
    if (event.appliedCouponId() != null) {
        Coupon coupon = couponRepository.findById(event.appliedCouponId()).orElseThrow();
        coupon.restore();  // 쿠폰 재사용 가능 상태로
        couponRepository.save(coupon);
    }
}
```

Order Aggregate는 쿠폰 복원을 "알려야 할 사실"(OrderCancelled)만 발행하고, 쿠폰 Context가 자체 규칙으로 처리합니다. 이것이 느슨한 결합의 핵심입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 도메인 분석](./01-domain-analysis.md)** | **[홈으로 🏠](../README.md)** | **[다음: Context 간 통합 ➡️](./03-context-integration.md)**

</div>
