# Domain Event 설계 — 과거형 이름과 이벤트 데이터 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 이벤트 이름이 왜 과거형(`OrderPlaced`, `PaymentCompleted`)이어야 하는가?
- 이벤트에 포함할 데이터를 "최소"로 할 때와 "풍부하게" 할 때의 트레이드오프는?
- 이벤트 버저닝은 왜 필요하고, 소비자 호환성을 어떻게 유지하는가?
- Domain Event를 Aggregate에서 어떻게 수집하고 발행하는가?
- 이벤트 발행과 DB 저장의 원자성 문제(메시지 유실)를 어떻게 해결하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"주문이 완료됐을 때 이메일을 발송해야 한다"는 요구사항을 어떻게 구현하는가? 가장 쉬운 방법은 `OrderService.placeOrder()` 끝에 `emailService.send()`를 직접 호출하는 것이다. 하지만 이렇게 하면 주문 서비스가 이메일 서비스를 알게 되고, "적립금 적립"이 추가될 때 또 코드를 고쳐야 하며, "SMS 발송"이 추가되면 또 고쳐야 한다.

Domain Event는 이 문제를 해결한다. `OrderPlaced` 이벤트를 발행하면 이메일 서비스, 적립금 서비스, SMS 서비스가 각자 이벤트를 구독해서 처리한다. **발행자(주문)는 구독자를 모른다.** 이것이 느슨한 결합의 핵심이고, 마이크로서비스 경계를 가능하게 하는 메커니즘이다.

---

## 😱 흔한 실수 (Before — 직접 호출 방식의 강한 결합)

```java
@Service
@Transactional
public class OrderService {

    private final EmailService emailService;
    private final PointService pointService;
    private final SmsService smsService;
    private final InventoryService inventoryService;
    private final PushNotificationService pushService;

    public void placeOrder(PlaceOrderRequest request) {
        Order order = Order.place(...);
        orderRepository.save(order);

        // 주문 완료 후 해야 할 일들이 OrderService에 모두 집중
        emailService.sendOrderConfirmation(order);           // 확인 이메일
        pointService.earnPoints(order.customerId(), order.totalAmount()); // 적립금
        smsService.sendOrderSms(order);                      // SMS
        inventoryService.decreaseStock(order.lines());       // 재고 차감
        pushService.sendOrderCompletedPush(order.customerId()); // 푸시 알림

        // 새 요구사항: "쿠폰도 사용 처리해야 한다" → 또 여기 추가
        // 새 요구사항: "마케팅 데이터 전송" → 또 여기 추가
    }
}
```

```
문제:
  OrderService가 5개 서비스에 의존
  새 요구사항마다 OrderService 수정 필요 (OCP 위반)
  이메일 서비스 장애 → 주문 전체 실패
  SMS 서비스가 느리면 주문 응답이 느려짐
  단위 테스트에 5개 서비스 Mock 필요

  "알림 방식 변경: SMS 대신 카카오 알림톡"
  → OrderService 코드 직접 수정
  → 테스트 다시 작성
  → 배포 조율
```

---

## ✨ 올바른 접근 (After — Domain Event 기반 느슨한 결합)

```java
// 1. Domain Event 정의 — 과거에 일어난 사실
public record OrderPlaced(
    OrderId orderId,
    CustomerId customerId,
    List<OrderLineSnapshot> lines,
    Money totalAmount,
    ShippingAddress shippingAddress,
    LocalDateTime placedAt
) implements DomainEvent {
    // 불변 레코드 — 과거 사실은 변경 불가
}

// 2. Aggregate가 이벤트 발행
public class Order {
    private final List<DomainEvent> events = new ArrayList<>();

    public static Order place(CustomerId customerId, List<OrderLine> lines,
                               Discount discount, Money shippingFee) {
        Order order = new Order(new OrderId(), customerId, PENDING, lines, ...);
        // 이벤트 기록 (아직 발행 안 함)
        order.events.add(new OrderPlaced(
            order.id, customerId,
            OrderLineSnapshot.of(lines),
            order.totalAmount,
            LocalDateTime.now()
        ));
        return order;
    }

    public List<DomainEvent> pullEvents() {
        List<DomainEvent> pulled = new ArrayList<>(this.events);
        this.events.clear();  // 꺼내면 비움
        return pulled;
    }
}

// 3. Application Service가 이벤트 발행
@Service
@Transactional
public class OrderApplicationService {

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = /* 생성 */;
        orderRepository.save(order);

        // 이벤트 발행 — 구독자가 누구인지 모름
        order.pullEvents().forEach(eventPublisher::publishEvent);

        return order.id();
    }
}

// 4. 각 서비스가 독립적으로 구독
@Component
public class OrderConfirmationEmailHandler {
    @EventListener
    @Async  // 비동기 처리 — 주문 응답에 영향 없음
    public void on(OrderPlaced event) {
        emailService.sendOrderConfirmation(event.customerId(), event.totalAmount());
    }
}

@Component
public class PointEarnHandler {
    @EventListener
    @Async
    public void on(OrderPlaced event) {
        pointService.earn(event.customerId(), event.totalAmount());
    }
}

// 새 요구사항 추가 → OrderService 수정 없이 새 핸들러만 추가
@Component
public class KakaoAlimtalkHandler {
    @EventListener
    @Async
    public void on(OrderPlaced event) {
        kakaoService.sendOrderAlimtalk(event.customerId(), event.orderId());
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 왜 과거형 이름인가

```
이벤트 이름 = 비즈니스에서 일어난 사실(Fact)

현재형 명령(Command)과 과거형 이벤트(Event)의 차이:

Command (명령 — 미래에 할 것):
  PlaceOrder       → "주문하라" (실패할 수 있음)
  CancelPayment    → "결제를 취소하라" (실패할 수 있음)
  ShipOrder        → "배송하라" (거부될 수 있음)

Event (사실 — 이미 일어난 것):
  OrderPlaced      → "주문이 됐다" (이미 일어남, 취소 불가)
  PaymentCancelled → "결제가 취소됐다" (이미 일어남)
  OrderShipped     → "배송이 시작됐다" (이미 일어남)

과거형이 중요한 이유:
  1. "이미 일어난 사실"이라는 의미 명확
     → 핸들러는 이미 일어난 일에 반응
     → Command처럼 "거부"할 수 없음

  2. 이벤트 소싱(Event Sourcing) 기반
     → 과거 사실의 로그가 시스템 상태를 재구성
     → "OrderPlaced → PaymentApproved → OrderShipped" = 배달 중인 주문

  3. 명확한 시제로 코드 가독성 향상
     if (event instanceof OrderPlaced) → "아, 주문 접수됐을 때"
     if (event instanceof PlaceOrder) → "주문을 해라? 이벤트인데?"

명명 패턴:
  [도메인 개념] + [과거형 동사]
  OrderPlaced, PaymentCompleted, ShipmentDispatched, ReviewSubmitted
  MemberRegistered, CouponExpired, InventoryReplenished
```

### 2. 이벤트 데이터: 최소 vs 풍부한 데이터

```
이벤트에 무엇을 담을 것인가? — 두 가지 전략

전략 1: 최소 데이터 (Thin Event)
  public record OrderPlaced(OrderId orderId, LocalDateTime placedAt) {}

  장점:
    이벤트 크기 작음 (네트워크, 저장 효율)
    스키마가 단순 → 버전 관리 쉬움
    개인정보 등 민감 데이터 노출 최소화

  단점:
    핸들러가 추가 조회 필요 (N+1 문제)
    핸들러가 조회 시점에 데이터가 변경됐을 수 있음
    DB가 분리된 마이크로서비스에서 조회 불가

  적합한 상황:
    핸들러가 같은 서비스 내에 있음 (같은 DB 접근 가능)
    이벤트가 자주 발행되고 크기가 중요할 때

─────────────────────────────────────────────

전략 2: 풍부한 데이터 (Fat Event / Enriched Event)
  public record OrderPlaced(
      OrderId orderId,
      CustomerId customerId,
      List<OrderLineSnapshot> lines,     // 스냅샷
      Money totalAmount,
      ShippingAddress shippingAddress,
      MembershipLevel customerLevel,     // 핸들러가 필요한 정보 포함
      LocalDateTime placedAt
  ) {}

  장점:
    핸들러가 추가 조회 없이 처리 가능 (자급자족)
    다른 서비스(마이크로서비스)에서도 처리 가능
    이벤트 발행 시점의 스냅샷 → 이후 데이터 변경에 무관

  단점:
    이벤트 크기 증가
    개인정보 등 민감 데이터 포함 위험
    스키마 변경 영향 범위 큼

  적합한 상황:
    이벤트가 다른 서비스(Bounded Context)로 전송됨
    핸들러가 DB를 직접 접근 불가
    이벤트 소싱(Event Sourcing)에서 상태 재구성에 사용

─────────────────────────────────────────────

현실적 권고:
  기본: 핸들러가 필요한 정보 + 관련 ID
  핸들러가 모두 같은 서비스: Thin Event + 필요 시 조회
  핸들러 중 일부가 다른 서비스: Fat Event (그 서비스가 필요한 것 포함)

  단, 개인정보(이름, 전화번호, 이메일)는:
  → 이벤트에 직접 포함 X (로그에 남을 수 있음)
  → ID만 포함하고 핸들러에서 조회
```

### 3. 이벤트 버저닝 전략

```
왜 버저닝이 필요한가:
  OrderPlaced 이벤트에 새 필드를 추가하면 어떻게 되는가?
  
  v1: { orderId, customerId, totalAmount }
  v2: { orderId, customerId, totalAmount, couponId }  ← 새 필드 추가
  
  기존 소비자가 v2 이벤트를 받으면:
    couponId를 모름 → 무시하면 됨 (하위 호환)
  
  v2 소비자가 v1 이벤트를 받으면:
    couponId가 없음 → null로 처리하면 됨 (하위 호환)
  
  그러나 필드 삭제나 타입 변경은:
    → 기존 소비자가 깨짐 (하위 비호환)
    → 버전 관리 전략 필요

전략 1: 스키마 하위 호환 유지 (Backward Compatible)
  규칙:
    새 필드 추가: 항상 Optional 또는 nullable
    기존 필드 삭제: 금지 (deprecated 표시 후 충분한 기간 유지)
    필드 이름 변경: 금지 (새 필드 추가 → 기존 필드 deprecated)
    타입 변경: 금지

  public record OrderPlaced(
      OrderId orderId,              // v1
      CustomerId customerId,        // v1
      Money totalAmount,            // v1
      @Nullable CouponId couponId,  // v2 추가 — nullable
      @Deprecated String customerName  // v2에서 deprecated, v3에서 삭제
  ) {}

전략 2: 이벤트 타입에 버전 포함
  public record OrderPlacedV1(OrderId orderId, CustomerId customerId) {}
  public record OrderPlacedV2(OrderId orderId, CustomerId customerId, CouponId couponId) {}

  // 발행자가 두 버전 동시 발행
  eventPublisher.publish(new OrderPlacedV1(order.id, order.customerId));
  eventPublisher.publish(new OrderPlacedV2(order.id, order.customerId, order.couponId));

  // 또는 V1 → V2 업그레이더 Adapter 사용

전략 3: 이벤트 봉투(Envelope)에 버전 정보
  {
    "type": "OrderPlaced",
    "version": "2",
    "timestamp": "2024-03-15T10:00:00Z",
    "payload": { ... }
  }
  
  소비자가 버전을 확인하고 처리:
  if ("2".equals(envelope.version())) {
      // V2 처리 로직
  } else {
      // V1 처리 로직 (하위 호환)
  }
```

### 4. 이벤트 발행과 DB 저장의 원자성 — Outbox Pattern

```
문제: 이벤트 유실(Lost Event)

@Transactional
public void placeOrder(PlaceOrderCommand command) {
    Order order = Order.place(...);
    orderRepository.save(order);       // DB 저장 성공

    // 여기서 장애 발생 (서버 다운, 네트워크 오류 등)

    eventPublisher.publish(event);     // 이벤트 발행 실패!
    // → Order는 저장됐는데 OrderPlaced 이벤트는 발행 안 됨
    // → 이메일 미발송, 재고 미차감, 포인트 미적립
}

해결: Outbox Pattern (트랜잭셔널 아웃박스)

DB 저장과 이벤트 저장을 같은 트랜잭션으로:

@Transactional
public void placeOrder(PlaceOrderCommand command) {
    Order order = Order.place(...);
    orderRepository.save(order);

    // 이벤트를 같은 트랜잭션으로 DB에 저장 (Outbox 테이블)
    List<DomainEvent> events = order.pullEvents();
    events.forEach(event -> outboxRepository.save(
        OutboxMessage.of(event)  // 이벤트를 JSON으로 직렬화하여 저장
    ));
    // 이 시점에 트랜잭션 커밋 → Order + Outbox 메시지 모두 저장됨
    // 서버가 여기서 죽어도 Outbox에 메시지가 있음
}

// 별도 스케줄러: Outbox → 메시지 브로커 릴레이
@Scheduled(fixedDelay = 1000)
@Transactional
public void relayOutboxMessages() {
    List<OutboxMessage> pending = outboxRepository.findPendingMessages();
    for (OutboxMessage message : pending) {
        try {
            kafkaTemplate.send(message.topic(), message.payload());
            outboxRepository.markPublished(message.id());
        } catch (Exception e) {
            // 실패 시 재시도 (idempotent 소비자 필요)
            log.warn("메시지 발행 실패, 재시도 예정: {}", message.id());
        }
    }
}

Outbox 테이블 구조:
  CREATE TABLE outbox_messages (
    id UUID PRIMARY KEY,
    aggregate_type VARCHAR(100),
    aggregate_id VARCHAR(100),
    event_type VARCHAR(200),
    payload JSONB,
    status VARCHAR(20) DEFAULT 'PENDING',
    created_at TIMESTAMP,
    published_at TIMESTAMP
  );
```

---

## 💻 실전 코드

### Domain Event 기반 인터페이스와 Aggregate 통합

```java
// DomainEvent 마커 인터페이스
public interface DomainEvent {
    LocalDateTime occurredAt();  // 이벤트 발생 시각
}

// 이벤트를 가진 Aggregate를 위한 추상 기반
public abstract class AggregateRoot {
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    protected void registerEvent(DomainEvent event) {
        this.domainEvents.add(event);
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(this.domainEvents);
        this.domainEvents.clear();
        return Collections.unmodifiableList(events);
    }
}

// Aggregate에서 사용
public class Order extends AggregateRoot {

    public static Order place(CustomerId customerId, List<OrderLine> lines,
                               Discount discount, Money shippingFee) {
        Order order = new Order(/* ... */);
        order.registerEvent(new OrderPlaced(
            order.id, customerId,
            OrderLineSnapshot.of(lines),
            order.totalAmount,
            LocalDateTime.now()
        ));
        return order;
    }

    public void cancel(String reason) {
        validateCancellable();
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelled(this.id, reason, LocalDateTime.now()));
    }
}

// Spring의 AbstractAggregateRoot 활용
public class Order extends AbstractAggregateRoot<Order> {
    // AbstractAggregateRoot.registerEvent() 그대로 사용 가능
    // Spring Data가 save() 시 자동으로 이벤트 발행

    public static Order place(...) {
        Order order = new Order(...);
        order.registerEvent(new OrderPlaced(...));
        return order;
    }
}
// orderRepository.save(order) 후 자동으로 OrderPlaced 이벤트 발행됨
```

### 이벤트 버저닝 실전 예시

```java
// 버전 1 이벤트
public record OrderPlacedV1(
    String orderId,
    String customerId,
    BigDecimal totalAmount,
    LocalDateTime placedAt
) implements DomainEvent {
    @Override public LocalDateTime occurredAt() { return placedAt; }
}

// 버전 2 이벤트 (쿠폰 정보 추가)
public record OrderPlacedV2(
    String orderId,
    String customerId,
    BigDecimal totalAmount,
    @Nullable String couponId,  // 새 필드, null 허용
    LocalDateTime placedAt
) implements DomainEvent {
    @Override public LocalDateTime occurredAt() { return placedAt; }
}

// Kafka 메시지 봉투 (Envelope)
public record EventEnvelope(
    String eventType,
    String version,
    String aggregateId,
    LocalDateTime occurredAt,
    String payload  // JSON 직렬화된 이벤트
) {}

// 소비자: 버전별 처리
@KafkaListener(topics = "order.events")
public void handle(String message) {
    EventEnvelope envelope = objectMapper.readValue(message, EventEnvelope.class);

    if ("OrderPlaced".equals(envelope.eventType())) {
        switch (envelope.version()) {
            case "1" -> handleV1(objectMapper.readValue(envelope.payload(), OrderPlacedV1.class));
            case "2" -> handleV2(objectMapper.readValue(envelope.payload(), OrderPlacedV2.class));
            default  -> log.warn("알 수 없는 버전: {}", envelope.version());
        }
    }
}

private void handleV1(OrderPlacedV1 event) {
    // V1 처리 (couponId 없음)
    handleOrderPlaced(event.orderId(), event.customerId(), null);
}

private void handleV2(OrderPlacedV2 event) {
    // V2 처리 (couponId 있을 수 있음)
    handleOrderPlaced(event.orderId(), event.customerId(), event.couponId());
}
```

---

## 📊 설계 비교

```
직접 호출 vs Domain Event:

                직접 호출                Domain Event
────────────┼──────────────────────┼──────────────────────────
결합도       │ 높음 (직접 의존)       │ 낮음 (이벤트만 알면 됨)
────────────┼──────────────────────┼──────────────────────────
새 기능 추가 │ 발행자 코드 수정       │ 새 핸들러만 추가
────────────┼──────────────────────┼──────────────────────────
장애 전파   │ 구독자 장애 → 발행자   │ 비동기 → 격리 가능
            │ 전체 실패             │
────────────┼──────────────────────┼──────────────────────────
응답 속도   │ 구독자 처리 시간 포함  │ 비동기 → 즉시 응답
────────────┼──────────────────────┼──────────────────────────
일관성      │ 강한 일관성 (같은 TX)  │ Eventually Consistent
────────────┼──────────────────────┼──────────────────────────
테스트      │ 구독자 Mock 필요       │ 이벤트 발행 여부만 확인
────────────┼──────────────────────┼──────────────────────────
원자성      │ 트랜잭션으로 보장      │ Outbox Pattern 필요
```

---

## ⚖️ 트레이드오프

```
Domain Event의 어려움:

① Eventually Consistent 허용 여부
   "주문 완료 직후 포인트 즉시 보여야 한다" → 이벤트로 처리하면 수 초 지연
   → 비즈니스가 허용하는가? 도메인 전문가와 합의 필요

② 이벤트 유실 방지 비용
   Outbox Pattern 구현 → 추가 테이블, 스케줄러, 중복 처리(idempotency) 구현
   → 작은 팀에서는 Spring @TransactionalEventListener로 시작

③ 이벤트 순서 보장
   Kafka: 파티션 내 순서만 보장
   "OrderPlaced → PaymentApproved" 순서가 보장되는가?
   → 소비자가 순서 무관하게 처리 가능하도록 설계 필요 (멱등성)

④ 이벤트 스키마 관리
   이벤트가 많아질수록 버전 관리 복잡
   Schema Registry (Confluent, AWS Glue) 도입 고려

가볍게 시작하는 방법:
  @TransactionalEventListener (Spring) → 같은 트랜잭션 내 이벤트 발행
  → Outbox 없이도 어느 정도 원자성 보장
  → 이벤트 처리 실패 시 트랜잭션 롤백 연동 가능
  → 외부 브로커 없이 시작 가능
```

---

## 📌 핵심 정리

```
Domain Event 설계 핵심:

이름: 과거형 (이미 일어난 사실)
  OrderPlaced, PaymentCompleted, ShipmentDispatched
  → 명령(Command)과 구분: PlaceOrder(명령) vs OrderPlaced(이벤트)

이벤트 데이터:
  최소 데이터: 핸들러가 같은 서비스 내, 추가 조회 가능
  풍부한 데이터: 다른 서비스(마이크로서비스)로 전송, 자급자족
  민감 데이터(이메일, 전화번호): 이벤트에 포함하지 말고 ID만

버저닝:
  새 필드 추가: nullable로 추가 (하위 호환)
  기존 필드 삭제/변경: 금지 (새 버전 이벤트 타입 추가)
  이벤트 봉투에 버전 정보 포함

원자성:
  Outbox Pattern: DB 저장과 이벤트 저장을 같은 트랜잭션
  스케줄러가 Outbox → 메시지 브로커 릴레이
  소비자는 멱등하게 처리 (중복 수신 가능)

발행 위치:
  Aggregate가 이벤트 등록 (registerEvent)
  Application Service가 save() 후 발행 (pullEvents)
  Spring Data: AbstractAggregateRoot 활용
```

---

## 🤔 생각해볼 문제

**Q1.** `OrderPlaced` 이벤트에 고객 이메일을 포함시키자는 의견이 있다. 이메일 핸들러가 고객 서비스를 조회하지 않아도 되기 때문이다. 어떻게 판단해야 하는가?

<details>
<summary>해설 보기</summary>

**이메일(개인정보)은 이벤트에 포함하지 않는 것이 원칙입니다.**

**문제점:**
1. **로그 노출**: 이벤트는 Kafka, Kibana, Datadog 등 다양한 곳에 기록됨 → 이메일이 로그에 노출
2. **GDPR/개인정보보호법**: 개인정보 삭제 요청 시 이벤트 로그에서도 삭제해야 → 불가능에 가까움
3. **감사 추적**: 누가 이메일을 봤는지 추적 어려움

**대안:**
```java
// 이벤트에는 ID만
public record OrderPlaced(
    OrderId orderId,
    CustomerId customerId,  // ID만! 이메일 없음
    Money totalAmount,
    LocalDateTime placedAt
) {}

// 이메일 핸들러가 필요 시 조회
@EventListener
@Async
public void on(OrderPlaced event) {
    // 이메일 핸들러가 Customer 서비스에서 조회
    Customer customer = customerRepository.findById(event.customerId()).orElseThrow();
    emailService.sendOrderConfirmation(customer.email(), event.totalAmount());
}
```

**마이크로서비스에서 조회가 불가능한 경우:**
- 이벤트에 포함하되 암호화 또는 토크나이징 처리
- 별도 개인정보 조회 API (PII API)를 통해 핸들러가 조회
- 이벤트 처리 시 개인정보 서비스를 통한 안전한 조회

</details>

---

**Q2.** 같은 트랜잭션에서 `Order`가 저장되고 `OrderPlaced` 이벤트가 발행됐는데, 이벤트 핸들러(`PointEarnHandler`)에서 예외가 발생하면 `Order` 저장도 롤백되는가?

<details>
<summary>해설 보기</summary>

**처리 방식에 따라 다릅니다.**

**방법 1: `@EventListener` + `@Async` (비동기)**
```java
@EventListener
@Async  // 별도 스레드, 별도 트랜잭션
public void on(OrderPlaced event) {
    pointService.earn(event.customerId(), event.totalAmount());
    // 예외 발생해도 Order 트랜잭션에 영향 없음
    // 단, 포인트 적립 실패 → 별도 처리 필요
}
```
→ Order 저장 롤백 없음. 포인트 적립은 별도 실패 처리.

**방법 2: `@TransactionalEventListener` (동기, 권장)**
```java
@TransactionalEventListener(phase = TransactionPhase.AFTER_COMMIT)
public void on(OrderPlaced event) {
    // 트랜잭션 커밋 후 실행 → 예외 발생해도 Order 롤백 없음
    pointService.earn(event.customerId(), event.totalAmount());
}
```
→ Order 저장 롤백 없음. 안전.

**방법 3: `@EventListener` (동기, 같은 트랜잭션)**
```java
@EventListener  // @Async 없음, 같은 트랜잭션
public void on(OrderPlaced event) {
    throw new RuntimeException("포인트 적립 실패!");
    // → Order 트랜잭션도 롤백!
}
```
→ Order 저장도 롤백됨. 의도한 경우에만 사용.

**권장 패턴:**
- 핵심 비즈니스 (주문 저장): 트랜잭션 내
- 부수 효과 (이메일, 포인트): `@TransactionalEventListener(AFTER_COMMIT)` 또는 Outbox Pattern

</details>

---

**Q3.** 이벤트 소비자가 같은 이벤트를 두 번 받을 수 있다(Kafka at-least-once). 포인트 적립 핸들러가 중복 실행되지 않으려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

**멱등성(Idempotency) 처리가 필요합니다.**

```java
// 방법 1: 이벤트 ID 기반 중복 체크
@EventListener
@Transactional
public void on(OrderPlaced event) {
    // 이미 처리한 이벤트인지 확인
    if (processedEventRepository.existsById(event.eventId())) {
        log.info("중복 이벤트 무시: {}", event.eventId());
        return;
    }

    pointService.earn(event.customerId(), event.totalAmount());
    processedEventRepository.save(new ProcessedEvent(event.eventId(), LocalDateTime.now()));
}

// 방법 2: 비즈니스 키 기반 멱등성
@Transactional
public void earnPoints(CustomerId customerId, OrderId orderId, Money amount) {
    // orderId 기준으로 이미 적립됐는지 확인
    if (pointHistoryRepository.existsByOrderId(orderId)) {
        log.info("이미 포인트 적립됨: orderId={}", orderId);
        return;
    }
    pointHistory.add(new PointEntry(customerId, orderId, amount));
}

// 방법 3: DB 유니크 제약 (가장 단순)
// CREATE UNIQUE INDEX ON point_entries(order_id);
// → 중복 INSERT 시 DB가 거부 → 예외 잡아서 무시
```

**이벤트 설계 시 포함해야 할 이유:**
```java
public record OrderPlaced(
    EventId eventId,          // 이벤트 고유 ID (UUID) ← 멱등성 키로 사용
    OrderId orderId,
    CustomerId customerId,
    Money totalAmount,
    LocalDateTime placedAt
) {}
```

멱등성이 보장되면 Kafka의 at-least-once 전송 정책을 안전하게 사용할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Factory 패턴](./07-factory-pattern.md)** | **[홈으로 🏠](../README.md)** | **[Chapter 4로 이동: Domain Events — 이벤트 결합도 ➡️](../domain-events/01-event-coupling.md)**

</div>
