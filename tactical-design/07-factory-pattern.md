# Factory 패턴 — 복잡한 Aggregate 생성의 책임 분리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 복잡한 생성 로직을 생성자에 두었을 때 어떤 책임 과적 문제가 생기는가?
- 정적 팩토리 메서드, Factory 메서드 패턴, Factory 클래스 중 언제 어느 것을 선택하는가?
- Aggregate 생성 시점에 불변식 검증을 강제하는 방법은?
- Domain Service와 Factory의 차이는 무엇인가?
- 테스트에서 Aggregate 생성을 단순화하는 Test Builder 패턴은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`new Order(customerId, items, coupon, shippingAddress, promotionCode, referralCode, ...)`처럼 생성자에 파라미터가 10개 이상이 되면 어떤 문제가 생기는가? 파라미터 순서를 헷갈리고, 검증 로직이 생성자에 뒤섞이며, 생성 방식이 여러 곳에서 다르게 표현된다.

Factory 패턴은 "복잡한 객체 생성"을 책임지는 전용 메서드나 클래스를 만드는 것이다. DDD에서 Factory는 단순히 "new 키워드를 감싸는 것"이 아니라 **생성 시점의 불변식을 검증하고, 의미 있는 이름으로 생성 의도를 표현하는 도메인 개념**이다.

---

## 😱 흔한 실수 (Before — 생성자에 모든 것을 넣기)

```java
// 파라미터 폭발 생성자
public class Order {

    public Order(Long customerId,
                 String customerName,  // ← 왜 생성자에?
                 List<OrderItemRequest> items,
                 String couponCode,
                 BigDecimal couponDiscountAmount,
                 String shippingCity,
                 String shippingStreet,
                 String shippingZipCode,
                 boolean isExpressDelivery,
                 String promotionCode,
                 boolean isFirstOrder,  // ← 외부에서 계산해서 넘기는 상태
                 BigDecimal membershipDiscountRate) {

        // 생성자에 뒤섞인 검증과 생성 로직
        if (items == null || items.isEmpty()) {
            throw new IllegalArgumentException("항목이 없습니다");
        }

        BigDecimal total = items.stream()
            .mapToDouble(i -> i.price().doubleValue() * i.quantity())
            .sum(); // 계산 로직이 생성자에

        if (couponCode != null && couponDiscountAmount != null) {
            total -= couponDiscountAmount.doubleValue();
        }

        if (isFirstOrder && promotionCode != null) {
            total *= 0.9; // 첫 주문 10% 할인
        }

        // 복잡한 배송비 계산도 생성자에
        BigDecimal shippingFee = "제주".equals(shippingCity) ? 3000 : 2500;
        if (isExpressDelivery) shippingFee += 5000;

        this.id = generateId();
        this.customerId = customerId;
        // ...
    }
}

// 사용하는 곳 — 파라미터 순서 실수 위험
Order order = new Order(
    customerId,
    customer.getName(),
    cartItems,
    coupon.getCode(),
    coupon.getDiscountAmount(),
    address.getCity(),
    address.getStreet(),
    address.getZipCode(),
    false,           // ← isExpressDelivery? isFirstOrder? 헷갈림
    null,
    customerService.isFirstOrder(customerId),  // ← Service를 호출해서 넘겨야 함
    membershipRate
);
```

---

## ✨ 올바른 접근 (After — Factory 패턴)

```java
// 정적 팩토리 메서드: 생성 의도를 이름으로 표현
public class Order {

    // "일반 주문 생성"
    public static Order place(CustomerId customerId, List<OrderLine> lines,
                               Discount discount, Money shippingFee) {
        Objects.requireNonNull(customerId);
        if (lines == null || lines.isEmpty()) {
            throw new EmptyOrderException("주문 항목이 최소 1개 이상이어야 합니다");
        }

        Money totalAmount = lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO_KRW, Money::add);

        Money finalAmount = discount.applyTo(totalAmount).add(shippingFee);

        Order order = new Order(
            new OrderId(),
            customerId,
            OrderStatus.PENDING,
            new ArrayList<>(lines),
            finalAmount
        );
        order.events.add(new OrderPlaced(order.id, customerId, lines, finalAmount));
        return order;
    }

    // "재주문 생성" — 다른 생성 시나리오
    public static Order reorder(Order previousOrder) {
        Objects.requireNonNull(previousOrder);
        if (!previousOrder.isDelivered()) {
            throw new ReorderNotAllowedException("배송 완료된 주문만 재주문 가능");
        }
        return Order.place(
            previousOrder.customerId(),
            previousOrder.lines(),  // 동일 항목
            Discount.NONE,
            ShippingFee.DEFAULT
        );
    }

    // private 생성자 — 팩토리 메서드를 통해서만 생성 강제
    private Order(OrderId id, CustomerId customerId, OrderStatus status,
                  List<OrderLine> lines, Money totalAmount) {
        this.id = id;
        this.customerId = customerId;
        this.status = status;
        this.lines = lines;
        this.totalAmount = totalAmount;
    }
}

// Factory 클래스: 외부 정보가 필요한 복잡한 생성
@Component
public class OrderFactory {

    private final PricingDomainService pricingService;
    private final ShippingDomainService shippingService;

    // Application Service에서 사용
    public Order create(PlaceOrderCommand command, Customer customer, Optional<Coupon> coupon) {
        List<OrderLine> lines = command.lines().stream()
            .map(r -> new OrderLine(r.productId(), r.quantity(), r.unitPrice()))
            .collect(toList());

        Discount discount = pricingService.calculateDiscount(customer, coupon, lines);
        Money shippingFee = shippingService.calculateShippingFee(command.address(), discount.base());

        return Order.place(customer.id(), lines, discount, shippingFee);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Factory 세 가지 유형 비교

```
유형 1: 정적 팩토리 메서드 (Static Factory Method)
  위치: Aggregate 클래스 내부 (static 메서드)
  
  public static Order place(...) { ... }
  public static Order reorder(...) { ... }
  public static Payment approve(...) { ... }

  장점:
    이름으로 생성 의도 표현 (place, reorder, approve)
    private 생성자 강제 → 외부 new 불가
    생성 규칙이 도메인 클래스와 함께 존재
  
  적합한 상황:
    생성 시 외부 서비스/Repository 호출 불필요
    생성 방식이 2~3가지 이하
    Aggregate 자체의 정보만으로 생성 가능

────────────────────────────────────────────────

유형 2: Factory 메서드 패턴 (다른 Aggregate가 생성)
  위치: 관련 Aggregate 또는 도메인 서비스

  // Shipment는 OrderPlaced 이벤트로 생성됨
  public class Shipment {
      public static Shipment createFrom(OrderPlaced event, ShippingAddress address) {
          return new Shipment(
              new ShipmentId(),
              event.orderId(),
              address,
              ShipmentStatus.PREPARING
          );
      }
  }

  // 또는 다른 Aggregate가 관련 Aggregate를 생성
  public class Order {
      public Shipment createShipment(ShippingAddress address) {
          return Shipment.createFrom(this, address);
      }
  }

  적합한 상황:
    생성에 다른 Aggregate의 정보가 필요
    이벤트로부터 Aggregate 생성

────────────────────────────────────────────────

유형 3: Factory 클래스 (별도 클래스)
  위치: 도메인 레이어 또는 응용 레이어

  @Component
  public class OrderFactory {
      private final PricingDomainService pricingService;
      private final ShippingDomainService shippingService;

      public Order create(PlaceOrderCommand command, Customer customer) { ... }
  }

  적합한 상황:
    생성에 Domain Service(계산 로직)가 필요
    생성 로직이 복잡해서 Aggregate 클래스에 넣기 어색
    여러 의존성이 필요한 생성

────────────────────────────────────────────────

유형 선택 기준:

  외부 의존 없음 + 단순 생성 → 정적 팩토리 메서드
  다른 Aggregate 정보 필요 → 관련 Aggregate의 팩토리 메서드
  Domain Service / 복잡한 계산 필요 → Factory 클래스
```

### 2. 생성 시점 불변식 검증의 중요성

```
불변식 검증 타이밍:

나쁜 패턴 — 생성 후 검증:
  Order order = new Order();
  order.setCustomerId(customerId);
  order.setItems(items);
  order.setStatus(OrderStatus.PENDING);
  // 이 시점에 불변식 검증이 없음
  // → 항목 없는 Order가 생성 가능
  // → status가 생략 가능

  orderValidator.validate(order);  // 나중에 검증
  orderRepository.save(order);

문제:
  validate()를 호출하지 않으면 유효하지 않은 Order가 저장됨
  여러 곳에서 Order를 생성하면 validate()를 잊을 수 있음

좋은 패턴 — 생성 시 검증:
  // 팩토리 메서드 안에서 검증
  public static Order place(CustomerId customerId, List<OrderLine> lines) {
      if (lines.isEmpty()) throw new EmptyOrderException(); // 생성 불가 → 검증됨
      // 유효한 Order만 생성 가능 → validate() 호출 불필요
      return new Order(new OrderId(), customerId, PENDING, lines);
  }

  Order order = Order.place(customerId, lines);  // 예외 없으면 항상 유효
  orderRepository.save(order);  // 검증 불필요

원칙: "생성됐으면 항상 유효하다"
  Aggregate가 존재한다는 것 자체가 불변식이 만족된다는 보장
```

### 3. Domain Service와 Factory의 차이

```
Factory:
  새 Aggregate 인스턴스를 만들어 반환
  생성의 "어떻게"를 캡슐화
  새 ID 발급, 초기 상태 설정, 초기 이벤트 발행

  public class OrderFactory {
      public Order create(PlaceOrderCommand command) {
          // 새 Order 인스턴스 반환
          return Order.place(command.customerId(), lines, discount, shippingFee);
      }
  }

Domain Service:
  여러 Aggregate 간 도메인 로직
  상태 없음, 새 인스턴스를 직접 만들지 않음 (또는 작은 VO 생성)

  public class PricingDomainService {
      public Discount calculateDiscount(Customer customer, Optional<Coupon> coupon) {
          // Discount VO를 계산해서 반환 (Order를 생성하지 않음)
          return new Discount(memberDiscount.rate().add(couponDiscount.rate()));
      }
  }

혼용 가능:
  Factory가 Domain Service를 사용해 생성에 필요한 계산을 수행
  → Factory가 생성 책임, Domain Service가 계산 책임
```

### 4. ID 발급 전략

```
Factory에서 ID를 발급하는 방법들:

방법 1: UUID (애플리케이션 레이어에서 발급)
  public static Order place(...) {
      return new Order(OrderId.generate(), ...);  // UUID.randomUUID()
  }
  → DB 의존 없이 ID 생성 가능
  → 분산 환경에서 충돌 없음
  → 장점: 저장 전에 ID 알 수 있음 (이벤트에 포함 가능)

방법 2: DB 시퀀스 (DB에서 발급)
  → JPA @GeneratedValue(strategy = GenerationType.SEQUENCE)
  → 저장 전에 ID 모름 → save() 후 ID 알 수 있음
  → 단점: 이벤트 발행 전 저장 필요 또는 Outbox Pattern

방법 3: 도메인 의미 있는 ID
  // 주문번호: 날짜 + 순번
  public record OrderNumber(String value) {
      public static OrderNumber generate() {
          String date = LocalDate.now().format(DateTimeFormatter.BASIC_ISO_DATE);
          return new OrderNumber(date + "-" + sequence.getAndIncrement());
      }
  }
  → 비즈니스 친화적 ID
  → 분산 환경에서 충돌 방지 로직 필요

권장: UUID (단순, 분산 친화, 저장 전 ID 알 수 있음)
```

---

## 💻 실전 코드

### 테스트용 Builder 패턴

```java
// Test Builder — 테스트에서 Aggregate 생성 단순화
public class OrderBuilder {
    private CustomerId customerId = new CustomerId(1L);
    private List<OrderLine> lines = List.of(
        new OrderLine(new ProductId(1L), 1, Money.ofKrw(10_000))
    );
    private Discount discount = Discount.NONE;
    private Money shippingFee = Money.ofKrw(2_500);

    public static OrderBuilder anOrder() {
        return new OrderBuilder();
    }

    public OrderBuilder forCustomer(CustomerId customerId) {
        this.customerId = customerId;
        return this;
    }

    public OrderBuilder withLines(List<OrderLine> lines) {
        this.lines = new ArrayList<>(lines);
        return this;
    }

    public OrderBuilder withGoldMemberDiscount() {
        this.discount = new Discount(new Percentage(10));
        return this;
    }

    public OrderBuilder withFreeShipping() {
        this.shippingFee = Money.ZERO_KRW;
        return this;
    }

    public Order build() {
        return Order.place(customerId, lines, discount, shippingFee);
    }

    public Order placed() {
        return build();  // PENDING 상태
    }

    public Order paid() {
        Order order = build();
        order.pay(new PaymentId());
        return order;
    }

    public Order shipped() {
        Order order = paid();
        order.ship(new TrackingNumber("CJ1234567890KR"));
        return order;
    }
}

// 테스트에서 사용
class OrderTest {

    @Test
    void goldMemberOrderHas10PercentDiscount() {
        Order order = OrderBuilder.anOrder()
            .withGoldMemberDiscount()
            .build();
        assertThat(order.totalAmount()).isEqualTo(Money.ofKrw(9_000));
    }

    @Test
    void shippedOrderCannotBeCancelled() {
        Order shipped = OrderBuilder.anOrder().shipped();
        assertThatThrownBy(shipped::cancel).isInstanceOf(OrderNotCancellableException.class);
    }
}
```

---

## 📊 설계 비교

```
생성자 직접 호출 vs 팩토리 메서드 vs 팩토리 클래스:

                생성자 직접        정적 팩토리        팩토리 클래스
────────────┼────────────────┼──────────────┼──────────────────
의도 표현    │ ❌ new Order()  │ ✅ Order.place │ ✅ factory.create
            │                │ Order.reorder │
────────────┼────────────────┼──────────────┼──────────────────
생성 제한    │ ❌ 누구나 new   │ ✅ private 생성자│ ✅ private 생성자
────────────┼────────────────┼──────────────┼──────────────────
불변식 보장  │ 생성자에 넣어야  │ 팩토리에 집중  │ 팩토리에 집중
────────────┼────────────────┼──────────────┼──────────────────
외부 의존   │ ❌ 생성자에 불가 │ ❌ static이므로 │ ✅ DI 가능
            │ (DI 불가)      │              │
────────────┼────────────────┼──────────────┼──────────────────
복잡도      │ 낮음            │ 중간          │ 높음
────────────┼────────────────┼──────────────┼──────────────────
적합한 경우  │ 단순 VO        │ 대부분의 Aggregate│ 복잡한 생성
            │                │              │ (여러 Domain Service)
```

---

## ⚖️ 트레이드오프

```
팩토리 클래스의 비용:
  추가 클래스 (OrderFactory) → 파일 수 증가
  의존성 주입 설정 필요
  단순한 경우 과잉 설계

정적 팩토리 메서드의 한계:
  외부 의존 (Domain Service) 주입 불가 (static 메서드)
  → 생성에 계산이 필요하면 Factory 클래스로
  
  "Order.place(customerId, command)"처럼 파라미터가 많아지면
  → Builder 패턴 고려

현실적 조언:
  대부분: 정적 팩토리 메서드로 시작
  생성 로직이 복잡해지면: Factory 클래스로 분리
  생성자 파라미터가 5개 이상: Builder 패턴 검토
```

---

## 📌 핵심 정리

```
Factory 패턴 핵심:

목적:
  복잡한 Aggregate 생성을 전담하는 메서드/클래스
  생성 의도를 이름으로 표현
  생성 시점에 불변식 검증 강제

세 가지 유형:
  정적 팩토리 메서드: Aggregate 내부, Order.place()
  팩토리 메서드: 관련 Aggregate가 생성, Shipment.createFrom(event)
  팩토리 클래스: 별도 @Component, 복잡한 생성에

불변식 강제:
  private 생성자 → 팩토리를 통해서만 생성
  팩토리에서 모든 검증 → 생성됐으면 항상 유효

ID 발급:
  UUID 권장 (저장 전 ID 알 수 있음, 이벤트 포함 가능)

테스트:
  Test Builder로 다양한 상태의 Aggregate 간편 생성
  단일 메서드로 여러 상태(placed, paid, shipped) 표현
```

---

## 🤔 생각해볼 문제

**Q1.** `Order.place()`와 `Order.reorder()`는 왜 두 개의 팩토리 메서드로 분리해야 하는가? 하나의 메서드로 `isReorder` 플래그로 처리하면 안 되는가?

<details>
<summary>해설 보기</summary>

**`isReorder` 플래그는 전형적인 "불리언 함정"입니다.**

```java
// 나쁜 예: 플래그로 분기
public static Order create(CustomerId customerId, List<OrderLine> lines, boolean isReorder) {
    if (isReorder) {
        // 재주문 로직
    } else {
        // 일반 주문 로직
    }
}
// create(customerId, lines, false) — false가 무슨 의미인지 불명확
// create(customerId, lines, true) — true가 재주문인지 모름
```

**두 메서드로 분리하면:**
```java
Order.place(customerId, items)     // 명확한 의도
Order.reorder(previousOrder)       // 명확한 의도 + 다른 파라미터
```

**추가 이점:**
- 파라미터가 다름: 일반 주문은 `items`가 필요하지만 재주문은 `previousOrder`가 필요
- 검증이 다름: 재주문은 이전 주문이 배송 완료 상태인지 확인
- 이벤트가 다를 수 있음: `OrderPlaced` vs `OrderReordered`
- 각각 독립적으로 변경 가능 (재주문 로직이 바뀌어도 일반 주문에 영향 없음)

이름이 명확한 메서드는 코드 자체가 문서입니다.

</details>

---

**Q2.** 팩토리 클래스(`OrderFactory`)와 Application Service(`OrderApplicationService`)의 경계를 어떻게 그어야 하는가?

<details>
<summary>해설 보기</summary>

**책임 분리 기준:**

```
OrderFactory (도메인 레이어):
  ✅ Order 생성에 필요한 도메인 계산 (할인, 배송비)
  ✅ 여러 파라미터로부터 Order 조립
  ✅ 불변식 검증
  ❌ Repository 조회 (인프라 의존)
  ❌ 이벤트 발행 (인프라 의존)
  ❌ 외부 API 호출

OrderApplicationService (응용 레이어):
  ✅ Repository에서 필요한 Aggregate 조회
  ✅ Factory 호출로 Aggregate 생성
  ✅ 저장 (Repository.save())
  ✅ 이벤트 발행
  ❌ 도메인 계산 직접 처리 (Factory/Domain Service에 위임)
```

```java
// 명확한 경계:
@Service
public class OrderApplicationService {
    @Transactional
    public OrderId placeOrder(PlaceOrderCommand command) {
        // Application Service: 조회 + 위임 + 저장
        Customer customer = customerRepository.findById(command.customerId()).orElseThrow();
        Optional<Coupon> coupon = command.couponCode().flatMap(couponRepository::findByCode);

        // Factory에 생성 위임
        Order order = orderFactory.create(command, customer, coupon);

        orderRepository.save(order);
        order.pullEvents().forEach(eventPublisher::publishEvent);
        return order.id();
    }
}

@Component
public class OrderFactory {
    // Factory: 생성 책임만
    public Order create(PlaceOrderCommand command, Customer customer, Optional<Coupon> coupon) {
        Discount discount = pricingService.calculateDiscount(customer, coupon, command.baseAmount());
        Money shippingFee = shippingService.calculate(command.address(), discount.base());
        return Order.place(customer.id(), buildLines(command), discount, shippingFee);
    }
}
```

</details>

---

**Q3.** 이벤트 핸들러에서 "이벤트로부터 새 Aggregate를 생성"할 때 Factory를 어떻게 활용하는가?

<details>
<summary>해설 보기</summary>

**이벤트 데이터로부터 Aggregate를 생성하는 정적 팩토리 메서드를 사용합니다.**

```java
// Shipment Aggregate
public class Shipment {

    // 이벤트로부터 생성하는 정적 팩토리 메서드
    public static Shipment createFrom(OrderPlaced event, ShippingAddress address) {
        Objects.requireNonNull(event);
        Objects.requireNonNull(address);

        Shipment shipment = new Shipment(
            new ShipmentId(),
            event.orderId(),    // OrderId만 참조
            address,
            ShipmentStatus.PREPARING,
            event.placedAt()
        );
        shipment.events.add(new ShipmentCreated(shipment.id, event.orderId()));
        return shipment;
    }
}

// 이벤트 핸들러에서 사용
@Component
public class OrderPlacedShippingHandler {

    @EventListener
    @Transactional
    public void on(OrderPlaced event) {
        ShippingAddress address = ShippingAddress.of(event.shippingAddress());
        Shipment shipment = Shipment.createFrom(event, address);  // 팩토리 메서드
        shipmentRepository.save(shipment);
    }
}
```

이벤트 기반 생성의 장점: 이벤트에 포함된 스냅샷 데이터만으로 Aggregate를 생성하므로, 주문 서비스의 DB를 다시 조회할 필요가 없습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Repository 설계 원칙](./06-repository-design.md)** | **[홈으로 🏠](../README.md)** | **[다음: Domain Event 설계 ➡️](./08-domain-event-design.md)**

</div>
