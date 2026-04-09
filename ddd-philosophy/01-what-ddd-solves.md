# DDD가 해결하는 문제 — 소프트웨어 복잡도의 본질

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 소프트웨어 복잡도는 왜 발생하며, 기술 중심 설계는 어떻게 복잡도를 키우는가?
- Anemic Domain Model이란 무엇이고, 왜 Service 비대화로 이어지는가?
- Transaction Script 방식은 언제까지 버티고, 어떤 시점에 무너지기 시작하는가?
- DDD가 복잡도를 다루는 방식은 기존 접근과 어떻게 다른가?
- "도메인 중심 설계"가 실제 코드에서 어떤 모습으로 나타나는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대부분의 시스템은 처음에는 단순하다. CRUD 화면 몇 개, Service 클래스 몇 개, 테이블 몇 개. 그런데 1~2년이 지나면 `OrderService`가 2,000줄이 되고, 메서드 이름과 실제 동작이 일치하지 않으며, 새 요구사항이 들어올 때마다 "어디를 수정해야 하는지" 찾는 데만 반나절이 걸린다.

이것이 **우발적 복잡도(Accidental Complexity)**다. 비즈니스 자체의 복잡도가 아니라, 잘못된 설계 방식이 만들어낸 복잡도다. DDD는 이 우발적 복잡도를 줄이고, 피할 수 없는 **본질적 복잡도(Essential Complexity)**를 도메인 모델 안에 명확하게 표현하는 방법론이다.

---

## 😱 흔한 실수 (Before — 기술 중심 설계의 전형)

```
상황: 전자상거래 "주문" 기능 구현

기술 중심 설계의 판단:
  "주문이 있으니까 orders 테이블"
  "주문 항목이 있으니까 order_items 테이블"
  "비즈니스 로직은 OrderService에"
  "데이터 접근은 OrderRepository에"

결과물:
```

```java
// 전형적인 Anemic Domain Model
@Entity
public class Order {
    @Id
    @GeneratedValue
    private Long id;

    private Long customerId;
    private String status;      // "PENDING", "PAID", "SHIPPED", "CANCELLED"
    private BigDecimal totalAmount;
    private LocalDateTime createdAt;

    @OneToMany(mappedBy = "order", cascade = CascadeType.ALL)
    private List<OrderItem> items = new ArrayList<>();

    // 비즈니스 로직 없음 — getter/setter만 존재
    public Long getId() { return id; }
    public void setId(Long id) { this.id = id; }
    public String getStatus() { return status; }
    public void setStatus(String status) { this.status = status; }
    public List<OrderItem> getItems() { return items; }
    public void setItems(List<OrderItem> items) { this.items = items; }
    public BigDecimal getTotalAmount() { return totalAmount; }
    public void setTotalAmount(BigDecimal totalAmount) { this.totalAmount = totalAmount; }
    // ... 나머지 getter/setter
}

// 모든 비즈니스 로직이 Service로 쏟아짐
@Service
@Transactional
public class OrderService {

    public OrderDto placeOrder(Long customerId, List<OrderItemRequest> items) {
        // 검증 로직이 Service에
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("주문 항목이 없습니다");
        }

        // 가격 계산 로직이 Service에
        BigDecimal total = BigDecimal.ZERO;
        for (OrderItemRequest item : items) {
            Product product = productRepository.findById(item.getProductId())
                .orElseThrow(() -> new RuntimeException("상품 없음"));
            total = total.add(product.getPrice().multiply(BigDecimal.valueOf(item.getQuantity())));
        }

        // 상태 설정 로직이 Service에
        Order order = new Order();
        order.setCustomerId(customerId);
        order.setStatus("PENDING");
        order.setTotalAmount(total);
        order.setCreatedAt(LocalDateTime.now());

        Order saved = orderRepository.save(order);

        // 항목 저장 로직도 Service에
        for (OrderItemRequest itemReq : items) {
            OrderItem orderItem = new OrderItem();
            orderItem.setOrder(saved);
            orderItem.setProductId(itemReq.getProductId());
            orderItem.setQuantity(itemReq.getQuantity());
            // ...
            orderItemRepository.save(orderItem);
        }

        return orderMapper.toDto(saved);
    }

    public void cancelOrder(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new RuntimeException("주문 없음"));

        // 비즈니스 규칙이 Service에 흩어짐
        if ("SHIPPED".equals(order.getStatus())) {
            throw new IllegalStateException("배송 중인 주문은 취소할 수 없습니다");
        }
        if ("CANCELLED".equals(order.getStatus())) {
            throw new IllegalStateException("이미 취소된 주문입니다");
        }

        order.setStatus("CANCELLED");
        orderRepository.save(order);

        // 환불 로직도 Service에 (PaymentService 직접 호출)
        paymentService.refund(orderId);
        // 재고 복원 로직도 Service에
        inventoryService.restore(orderId);
    }

    public void payOrder(Long orderId, PaymentInfo paymentInfo) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new RuntimeException("주문 없음"));

        if (!"PENDING".equals(order.getStatus())) {
            throw new IllegalStateException("결제 가능한 상태가 아닙니다");
        }

        // 결제 로직...
        paymentService.pay(paymentInfo);
        order.setStatus("PAID");
        orderRepository.save(order);
    }
}
```

```
6개월 후의 현실:
  OrderService: 1,500줄
  cancelOrder() 메서드: 80줄 (쿠폰 환불, 포인트 복원, 알림, 재고 복원...)
  새 요구사항: "배송 준비 중 상태에서만 배송지 변경 허용"
    → 어디에? OrderService? ShippingService? Order 엔티티?
    → 상태 문자열 "PREPARING"을 몇 군데에 추가해야 하는가?
    → "PREPARING" 상태를 빠뜨리면 버그. 검증할 방법이 없음.

문제의 본질:
  Order 엔티티는 데이터 컨테이너 (DB 테이블의 Java 복사본)
  비즈니스 규칙이 Service에 흩어짐
  상태 전이 규칙이 if/else 문자열 비교로 구현됨
  "취소 가능한가?" 라는 도메인 질문에 Order가 답할 수 없음
```

---

## ✨ 올바른 접근 (After — 도메인 중심 설계)

```java
// Rich Domain Model — Order가 자신의 불변식을 보호한다
public class Order {  // JPA 어노테이션 없음 — 순수 도메인 객체

    private OrderId id;
    private CustomerId customerId;
    private OrderStatus status;          // String이 아닌 열거형
    private List<OrderLine> lines;       // 불변 컬렉션으로 노출
    private List<DomainEvent> events = new ArrayList<>();

    // 불변식 1: 주문은 최소 1개 이상의 상품을 포함해야 한다
    // 불변식 2: 주문 생성 시 항상 PENDING 상태로 시작한다
    public static Order place(CustomerId customerId, List<OrderLineRequest> requests) {
        if (requests == null || requests.isEmpty()) {
            throw new EmptyOrderException("주문에 최소 1개 이상의 상품이 필요합니다");
        }

        List<OrderLine> lines = requests.stream()
            .map(r -> new OrderLine(r.getProductId(), r.getQuantity(), r.getPrice()))
            .collect(toList());

        Order order = new Order(new OrderId(), customerId, OrderStatus.PENDING, lines);
        order.events.add(new OrderPlaced(order.id, customerId, lines));
        return order;
    }

    // 불변식 3: PENDING 상태에서만 취소할 수 있다
    // 불변식 4: 이미 취소된 주문은 다시 취소할 수 없다
    public void cancel() {
        if (!this.status.isCancellable()) {
            throw new OrderNotCancellableException(
                String.format("'%s' 상태의 주문은 취소할 수 없습니다", this.status.displayName())
            );
        }
        this.status = OrderStatus.CANCELLED;
        this.events.add(new OrderCancelled(this.id));
    }

    // 불변식 5: PENDING 상태에서만 결제할 수 있다
    public void pay(PaymentId paymentId) {
        if (this.status != OrderStatus.PENDING) {
            throw new OrderNotPayableException("결제 가능한 상태가 아닙니다: " + this.status);
        }
        this.status = OrderStatus.PAID;
        this.events.add(new OrderPaid(this.id, paymentId, this.totalAmount()));
    }

    // 도메인 질문에 Order가 직접 답한다
    public boolean isCancellable() {
        return this.status.isCancellable();
    }

    public Money totalAmount() {
        return lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);
    }

    // 외부에는 불변 뷰만 제공 — 직접 조작 불가
    public List<OrderLine> getLines() {
        return Collections.unmodifiableList(lines);
    }

    public List<DomainEvent> pullEvents() {
        List<DomainEvent> occurred = new ArrayList<>(events);
        events.clear();
        return occurred;
    }
}

// 상태 전이 규칙이 코드로 명확하게 표현됨
public enum OrderStatus {
    PENDING("주문 접수"),
    PAID("결제 완료"),
    PREPARING("배송 준비 중"),
    SHIPPED("배송 중"),
    DELIVERED("배송 완료"),
    CANCELLED("취소됨");

    private final String displayName;

    // 상태 전이 규칙이 한 곳에 집중됨
    public boolean isCancellable() {
        return this == PENDING || this == PAID;
    }

    public boolean canChangeShippingAddress() {
        return this == PREPARING;
    }

    public String displayName() { return displayName; }
}

// Application Service는 오케스트레이션만 담당
@Service
@Transactional
public class OrderApplicationService {

    public void cancelOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));

        order.cancel();  // 비즈니스 규칙은 Order가 안다

        orderRepository.save(order);
        // 이벤트 발행은 save() 이후 자동 처리 (AbstractAggregateRoot 또는 직접 발행)
    }
}
```

```
변화:
  "취소 가능한가?" → order.isCancellable()   // Order가 답한다
  "결제 가능한가?" → status.canPay()          // OrderStatus가 답한다
  "배송지 변경 가능한가?" → status.canChangeShippingAddress()

새 상태 "PREPARING" 추가 시:
  OrderStatus 열거형에 추가
  isCancellable(), canChangeShippingAddress() 등 수정
  → 누락 불가 (열거형이 강제함)
  → Service에 if문 추가 없음
```

---

## 🔬 내부 동작 원리

### 1. 소프트웨어 복잡도의 두 가지 종류

```
본질적 복잡도 (Essential Complexity):
  비즈니스 자체가 복잡하기 때문에 발생하는 복잡도
  피할 수 없음 — 소프트웨어가 해결해야 하는 문제 자체의 복잡도

  예시: 전자상거래 주문 처리
    - 쿠폰 + 포인트 + 현금 혼합 결제 계산 규칙
    - 주문 상태에 따른 취소/환불/재주문 가능 여부
    - 상품 유형별 배송비 계산 (냉장품, 대형 가전, 일반 택배)
    - 타임세일 + 멤버십 할인 중복 적용 우선순위

우발적 복잡도 (Accidental Complexity):
  잘못된 설계/기술 선택이 만들어낸 복잡도
  피할 수 있음 — 올바른 설계로 제거 가능

  예시:
    - 비즈니스 규칙이 Service / Controller / Batch에 흩어짐
    - 상태가 문자열 "PENDING", "PAID"로 비교됨 (오타 버그)
    - 주문 변경 방법이 setter를 통해 어디서든 가능
    - 트랜잭션 경계가 불명확해 데이터 정합성 보장 어려움

DDD의 목표:
  본질적 복잡도 → 도메인 모델 안에 명확히 표현
  우발적 복잡도 → 올바른 경계와 추상화로 제거
```

### 2. Anemic Domain Model이 만드는 악순환

```
Anemic Domain Model 성장 패턴:

초기 (3개월):
  Order {id, status, items, totalAmount}  ← 데이터만
  OrderService {placeOrder(), cancelOrder()}  ← 로직만

  문제 없어 보임. 단순하고 명확해 보임.

6개월:
  요구사항 추가: "쿠폰 적용", "포인트 적립", "배송비 계산"
  → OrderService에 메서드 추가: applyCoupon(), earnPoints(), calculateShipping()
  → 서로 관련된 로직이 분리된 메서드에 흩어짐

12개월:
  OrderService: 1,200줄
  placeOrder(): 150줄 (상품 검증, 재고 확인, 쿠폰, 포인트, 배송비, 결제...)
  cancelOrder(): 120줄 (환불, 쿠폰 복원, 포인트 차감, 재고 복원, 알림...)

  새 개발자 진입 시:
    Q: "주문 취소 로직이 어디 있나요?"
    A: "OrderService, PaymentService, InventoryService, NotificationService에 나뉘어 있어요"
    → 실수로 일부를 빠뜨리면 데이터 불일치

18개월:
  "배송비 계산 로직이 OrderService와 CartService 두 곳에 있는데 다르네요?"
  → 복사-붙여넣기로 퍼진 비즈니스 로직
  → 어느 것이 맞는지 아무도 모름

핵심 문제:
  Order 엔티티는 비즈니스 규칙을 모른다
    → "이 주문이 취소 가능한가?" 를 Order에게 물을 수 없음
    → Service가 규칙을 알아야 함
    → Service가 비대해짐
    → 규칙이 Service에 중복됨
    → 규칙 변경 시 모든 Service를 찾아야 함
```

### 3. Transaction Script vs Domain Model

```
Transaction Script 방식:
  하나의 비즈니스 연산 = 하나의 절차적 스크립트

  placeOrder(customerId, items):
    1. 입력 검증
    2. 재고 확인
    3. 가격 계산
    4. Order 레코드 생성
    5. OrderItem 레코드 생성
    6. 결제 처리
    7. 재고 차감
    8. 알림 발송
    → 모든 단계가 하나의 메서드에

  장점: 단순한 연산에서 직관적이고 빠르게 구현 가능
  한계: 연산이 복잡해질수록 스크립트가 길어지고, 규칙이 흩어짐

Domain Model 방식:
  비즈니스 개념을 객체로 표현, 객체가 자신의 행동을 가짐

  Order.place(customerId, items):
    → Order가 검증 (빈 항목 불허)
    → Order가 상태 설정 (항상 PENDING으로 시작)
    → OrderPlaced 이벤트 발행 (다른 연산 트리거)

  Order.cancel():
    → Order가 취소 가능 여부 판단
    → Order가 상태 전이
    → OrderCancelled 이벤트 발행

  장점: 복잡한 도메인에서 규칙이 응집됨, 테스트가 쉬움
  한계: 단순 CRUD에 적용하면 과잉 설계

복잡도에 따른 선택:
  단순 CRUD        → Transaction Script로 충분
  중간 복잡도       → Table Module (테이블 단위 로직 집합)
  높은 복잡도       → Domain Model (DDD)

DDD는 복잡도 높은 비즈니스 문제에 대한 답이지,
모든 문제의 답이 아님.
```

### 4. 도메인 모델이 복잡도를 다루는 방식

```
DDD가 복잡도를 다루는 3가지 메커니즘:

① 캡슐화 — 불변식을 객체 안에 가둠
  
  "주문은 PENDING 상태에서만 취소 가능하다"
  → Order.cancel() 내부에서만 검증
  → 외부에서는 규칙을 알 필요 없음
  → 규칙이 바뀌면 Order.cancel() 하나만 수정

② Ubiquitous Language — 도메인 언어와 코드 언어의 일치
  
  비즈니스: "주문을 접수한다"  →  Order.place()
  비즈니스: "주문을 취소한다"  →  Order.cancel()
  비즈니스: "결제가 완료됐다"  →  OrderStatus.PAID
  
  코드 읽기 = 비즈니스 규칙 읽기
  → 개발자와 도메인 전문가가 같은 언어로 소통

③ 경계 — 복잡도를 격리
  
  Bounded Context: 주문 컨텍스트, 배송 컨텍스트, 결제 컨텍스트
  → 각 컨텍스트가 자신의 복잡도를 관리
  → 컨텍스트 간 의존성을 이벤트로 최소화
  → 한 컨텍스트의 변경이 다른 컨텍스트에 파급되지 않음
```

---

## 💻 실전 코드

### 단위 테스트 — 도메인 로직이 Order 안에 있으면 테스트가 단순해진다

```java
class OrderTest {

    @Test
    @DisplayName("빈 항목으로 주문하면 예외가 발생한다")
    void place_withEmptyItems_throwsException() {
        assertThatThrownBy(() ->
            Order.place(new CustomerId(1L), Collections.emptyList())
        ).isInstanceOf(EmptyOrderException.class);
    }

    @Test
    @DisplayName("PENDING 상태의 주문은 취소할 수 있다")
    void cancel_pendingOrder_succeeds() {
        Order order = Order.place(new CustomerId(1L), sampleItems());

        order.cancel();

        assertThat(order.getStatus()).isEqualTo(OrderStatus.CANCELLED);
    }

    @Test
    @DisplayName("SHIPPED 상태의 주문은 취소할 수 없다")
    void cancel_shippedOrder_throwsException() {
        Order order = orderInStatus(OrderStatus.SHIPPED);

        assertThatThrownBy(order::cancel)
            .isInstanceOf(OrderNotCancellableException.class)
            .hasMessageContaining("배송 중");
    }

    @Test
    @DisplayName("주문 취소 시 OrderCancelled 이벤트가 발행된다")
    void cancel_publishesOrderCancelledEvent() {
        Order order = Order.place(new CustomerId(1L), sampleItems());

        order.cancel();

        List<DomainEvent> events = order.pullEvents();
        assertThat(events).hasSize(2); // OrderPlaced + OrderCancelled
        assertThat(events.get(1)).isInstanceOf(OrderCancelled.class);
    }
}
```

```
테스트의 특징:
  JPA 없음, Spring Context 없음, DB 없음
  Order 객체 하나로 모든 비즈니스 규칙 검증
  테스트 속도: 수 ms (DB 연동 테스트 대비 100~1000배 빠름)
  이것이 가능한 이유: 비즈니스 로직이 Order 안에 있기 때문
```

---

## 📊 설계 비교

```
Anemic Model vs Rich Domain Model:

                    Anemic Model          Rich Domain Model
─────────────────┼────────────────────┼────────────────────────
Entity의 역할     │ 데이터 컨테이너       │ 불변식 수호자 + 행동 주체
                │ (getter/setter만)    │ (cancel(), pay(), place())
─────────────────┼────────────────────┼────────────────────────
비즈니스 규칙 위치 │ Service에 흩어짐     │ 해당 Entity/VO 안
─────────────────┼────────────────────┼────────────────────────
상태 전이 검증    │ if ("PAID".equals()) │ status.isCancellable()
                │ → 오타 버그 가능       │ → 컴파일 타임 검증
─────────────────┼────────────────────┼────────────────────────
단위 테스트       │ Service 테스트 필요  │ Order 단독 테스트 가능
                │ (Mock 다수 필요)      │ (외부 의존성 없음)
─────────────────┼────────────────────┼────────────────────────
새 규칙 추가      │ 모든 Service 탐색    │ 해당 메서드 하나 수정
─────────────────┼────────────────────┼────────────────────────
코드 읽기         │ 규칙 찾기 어려움     │ 코드 = 비즈니스 명세
─────────────────┼────────────────────┼────────────────────────
초기 개발 속도    │ 빠름                 │ 느림 (설계 투자 필요)
─────────────────┼────────────────────┼────────────────────────
복잡도 증가 시    │ 급격히 저하          │ 선형적으로 유지
```

---

## ⚖️ 트레이드오프

```
DDD 도입의 비용:
  ① 초기 학습 비용
     - Aggregate, Bounded Context, Domain Event 개념 학습
     - 팀 전체의 이해와 합의 필요
  
  ② 초기 개발 속도 저하
     - 설계에 더 많은 시간 투자
     - 단순 CRUD도 풍부하게 모델링하면 과잉 설계
  
  ③ 팀 역량 의존성
     - 도메인 전문가와 지속적 소통 필요
     - 설계 판단이 개인 역량에 크게 의존

DDD 도입의 편익:
  ① 비즈니스 규칙의 응집
     - "이 규칙이 어디에?" → 해당 Aggregate 확인
     - 규칙 변경 시 영향 범위 최소화
  
  ② 테스트 용이성
     - 핵심 비즈니스 로직을 인프라 없이 테스트
  
  ③ 변경 내성
     - 복잡도가 증가해도 기존 코드 구조 유지
     - 새 요구사항 추가 시 기존 코드 수정 최소화

DDD가 적합한 상황:
  - 도메인 전문가와 협업이 가능한 환경
  - 비즈니스 규칙이 복잡하고 자주 변경됨
  - 장기적으로 유지보수할 시스템

DDD가 과잉인 상황:
  - 단순 CRUD 시스템 (게시판, 설정 관리 등)
  - 프로토타입 / 단기 프로젝트
  - 도메인 전문가 접근이 어려운 환경
```

---

## 📌 핵심 정리

```
DDD가 해결하는 문제:

소프트웨어 복잡도 = 본질적 복잡도 + 우발적 복잡도
  DDD: 본질적 복잡도를 도메인 모델에 명확히 표현
        우발적 복잡도를 경계와 캡슐화로 제거

Anemic Domain Model의 함정:
  Entity = 데이터 컨테이너 (getter/setter)
  Service = 비즈니스 로직 창고 (점점 비대해짐)
  → 복잡도가 Service에 흩어지고, 규칙이 중복됨

Rich Domain Model의 핵심:
  Entity가 자신의 불변식을 보호한다
  Service는 오케스트레이션만 한다
  코드가 비즈니스 언어와 일치한다

도입 판단:
  도메인 복잡도가 높고, 장기 유지보수가 필요한 시스템 → DDD 적합
  단순 CRUD, 단기 프로젝트 → Transaction Script로 충분
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 Anemic Domain Model의 증상을 찾아보세요. 어떤 비즈니스 규칙이 Entity 밖으로 새나갔는가?

```java
@Service
public class MemberService {
    public void upgradeMembership(Long memberId) {
        Member member = memberRepository.findById(memberId).orElseThrow();
        int purchaseCount = orderRepository.countByMemberId(memberId);
        if (purchaseCount >= 10) {
            member.setMembershipLevel("GOLD");
        } else if (purchaseCount >= 5) {
            member.setMembershipLevel("SILVER");
        }
        memberRepository.save(member);
    }
}
```

<details>
<summary>해설 보기</summary>

**새나간 비즈니스 규칙들:**

1. **멤버십 등급 판단 규칙** — "구매 10회 이상이면 GOLD, 5회 이상이면 SILVER"는 `Member`의 비즈니스 규칙인데 `MemberService`가 알고 있습니다.
2. **등급 비교 로직** — 문자열 `"GOLD"`, `"SILVER"` 비교는 오타에 취약하며, 새 등급 추가 시 `Service`를 찾아 수정해야 합니다.

**Rich Domain Model로 이동하면:**

```java
public class Member {
    private MembershipLevel level;
    private int totalPurchaseCount;

    // 비즈니스 규칙이 Member 안에 있음
    public void recordPurchase() {
        this.totalPurchaseCount++;
        this.level = MembershipLevel.from(this.totalPurchaseCount);
    }
}

public enum MembershipLevel {
    GENERAL, SILVER, GOLD;

    public static MembershipLevel from(int purchaseCount) {
        if (purchaseCount >= 10) return GOLD;
        if (purchaseCount >= 5) return SILVER;
        return GENERAL;
    }
}
```

이제 등급 기준이 바뀌면 `MembershipLevel.from()` 하나만 수정하면 됩니다.

</details>

---

**Q2.** "Transaction Script로 시작했다가 나중에 Domain Model로 리팩터링하면 되지 않나?" 라는 주장의 장단점은 무엇인가?

<details>
<summary>해설 보기</summary>

**이 전략의 현실적 장점:**
- 초기에 빠르게 출시하고 복잡도를 확인한 후 투자를 결정할 수 있습니다.
- 실제로 복잡도가 낮은 시스템은 끝까지 Transaction Script로도 충분합니다.

**현실적 어려움:**

1. **리팩터링 타이밍**: Service가 비대해진 시점에는 이미 비즈니스 로직이 여러 Service에 산재하고, 의존성이 복잡하게 얽혀 있습니다. 이 상태에서 Domain Model로 전환하는 것은 새로 작성하는 것보다 훨씬 어렵습니다.

2. **테스트 부재**: Anemic Model로 작성된 코드는 Service Mock이 많아 테스트 작성이 어렵고, 테스트가 없으면 리팩터링 시 회귀 버그를 잡기 어렵습니다.

3. **팀의 관성**: "이렇게 돌아가고 있는데 굳이?" 라는 심리적 저항이 리팩터링을 막습니다.

**현실적 전략**: 처음부터 완벽한 DDD가 아니더라도, 적어도 **비즈니스 규칙을 Service가 아닌 Entity에 두는 습관**부터 시작하면 나중의 리팩터링 비용이 크게 줄어듭니다.

</details>

---

**Q3.** Order 도메인에 "주문 금액이 10만 원 이상이면 무료 배송" 이라는 규칙을 추가해야 한다. Rich Domain Model에서 이 규칙은 어느 객체에 위치해야 하는가? 그 이유는?

<details>
<summary>해설 보기</summary>

**정답: `Order` 또는 별도의 `ShippingPolicy` Value Object**

"주문 금액이 10만 원 이상이면 무료 배송"은 **주문의 금액과 배송비 사이의 관계**입니다.

```java
// 방법 1: Order에 직접 — 단순한 경우
public class Order {
    public Money shippingFee() {
        return this.totalAmount().isGreaterThanOrEqualTo(Money.of(100_000))
            ? Money.ZERO
            : Money.of(3_000);
    }
}

// 방법 2: ShippingPolicy Value Object — 정책이 복잡하거나 자주 변경될 때
public class ShippingPolicy {
    private final Money freeShippingThreshold;
    private final Money defaultShippingFee;

    public Money calculate(Money orderAmount) {
        return orderAmount.isGreaterThanOrEqualTo(freeShippingThreshold)
            ? Money.ZERO
            : defaultShippingFee;
    }
}
```

**Service에 두면 안 되는 이유**: "이 주문의 배송비는 얼마인가?"는 Order가 답할 수 있는 질문입니다. Service가 알아야 할 이유가 없으며, Service에 두면 동일 로직이 여러 Service에 복사될 위험이 있습니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: Ubiquitous Language — 언어가 코드 구조를 결정한다 ➡️](./02-ubiquitous-language.md)**

</div>
