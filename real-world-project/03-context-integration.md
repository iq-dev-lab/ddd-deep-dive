# Bounded Context 간 통합 — 주문 이벤트가 여러 Context로 전파

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `OrderPlaced` 이벤트가 배송 / 재고 / 결제 Context로 전파되는 전체 흐름은?
- Outbox Pattern으로 이벤트 유실 없이 발행하는 완전한 구현은?
- Saga로 주문 취소 시 재고 복원과 결제 환불을 조율하는 방법은?
- 이벤트 전파 중 특정 Context가 장애일 때 시스템은 어떻게 동작하는가?
- 전자상거래에서 Choreography Saga와 Orchestration Saga 중 어느 것이 적합한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

전자상거래의 핵심 시나리오: "고객이 주문 버튼을 눌렀다." 이 하나의 행위가 연쇄적으로 재고 차감, 배송 예약, 결제 요청, 이메일 발송, 포인트 적립을 유발한다. 이것들이 하나의 트랜잭션으로 묶이면 하나라도 실패 시 전체가 실패한다. 실제로 결제 PG사는 분산 트랜잭션을 지원하지 않는다.

이벤트 기반 통합과 Saga 패턴이 이 문제를 실전에서 어떻게 해결하는지 구체적으로 살펴본다.

---

## 😱 흔한 실수 (Before — 단일 거대 트랜잭션)

```java
@Service
@Transactional
public class OrderPlacementService {
    public void placeOrder(PlaceOrderCommand command) {
        Order order = Order.place(...);
        orderRepository.save(order);

        // 모든 것이 하나의 트랜잭션
        paymentService.charge(order.customerId(), order.totalAmount());   // 외부 PG사
        inventoryService.decrease(order.lines());                         // 재고 DB
        shipmentService.reserve(order.shippingAddress());                 // 배송 시스템
        emailService.sendConfirmation(order.customerId());                // 이메일 서버
        pointService.earn(order.customerId(), order.totalAmount());       // 포인트 DB

        // PG사가 타임아웃 → 5초 대기 → 전체 롤백
        // 이메일 서버 장애 → 주문 전체 실패
        // 재고가 충분한데 PG사 장애로 주문 불가
    }
}
```

---

## ✨ 올바른 접근 (After — 이벤트 기반 분리 + Outbox)

```
이벤트 흐름 전체:

주문 Context:
  1. Order.place() → [PENDING 상태, OrderPlaced 이벤트 생성]
  2. orderRepository.save() + outboxRepository.save(OrderPlaced) [같은 트랜잭션]
  3. Outbox Relay → Kafka "order.events" 토픽

재고 Context:
  4. OrderPlaced 이벤트 수신
  5. 재고 예약 또는 차감
  6. InventoryDecreased 이벤트 발행

결제 Context:
  7. Order PENDING → 결제 요청 (Saga 조율)
  8. PG사 결제 승인
  9. PaymentApproved 이벤트 발행

배송 Context:
  10. PaymentApproved 이벤트 수신
  11. 배송 생성 (ShipmentCreated)

주문 Context:
  12. PaymentApproved → Order.confirmPayment() → [PAID 상태]
  13. ShipmentCreated → 배송 정보 연결

이메일/포인트:
  14. OrderPlaced, PaymentApproved 이벤트에 비동기 반응
```

---

## 🔬 내부 동작 원리

### 1. OrderPlaced 이벤트 설계 — Fat Event

```java
// OrderPlaced: 각 Context가 필요한 정보를 자급자족
public record OrderPlaced(
    UUID eventId,                    // 멱등성 키
    OrderId orderId,
    CustomerId customerId,
    List<OrderLineSnapshot> lines,   // 재고 차감에 필요
    Money totalAmount,               // 결제에 필요
    ShippingAddress shippingAddress, // 배송 예약에 필요
    Money shippingFee,
    CouponId appliedCouponId,        // nullable
    LocalDateTime placedAt
) implements DomainEvent {

    // 이벤트 타입과 버전
    public String eventType() { return "OrderPlaced"; }
    public String version() { return "2"; }

    @Override
    public LocalDateTime occurredAt() { return placedAt; }
}

// Kafka 메시지 봉투
public record EventEnvelope(
    String eventId,
    String eventType,
    String version,
    String aggregateType,
    String aggregateId,
    LocalDateTime occurredAt,
    String payload  // JSON
) {}
```

### 2. Outbox Pattern 전체 구현

```java
// Application Service: 주문 저장 + Outbox 저장 (원자적)
@Service
@Transactional
public class OrderApplicationService {

    public OrderId placeOrder(PlaceOrderCommand command) {
        // 재고 확인 (동기)
        validateInventory(command.lines());

        Order order = orderFactory.create(command);
        orderRepository.save(order);

        // Outbox에 이벤트 저장 (같은 트랜잭션!)
        order.pullDomainEvents().stream()
            .map(event -> OutboxMessage.pending(event, "order.events", objectMapper))
            .forEach(outboxRepository::save);

        // 트랜잭션 커밋 → Order + Outbox 원자적으로 저장됨
        return order.id();
    }
}

// Outbox Relay (별도 스케줄러)
@Scheduled(fixedDelay = 500)
@Transactional
public void relay() {
    outboxRepository.findTop20PendingWithLock().forEach(msg -> {
        try {
            kafkaTemplate.send(msg.topic(), msg.aggregateId(), msg.payload())
                         .get(5, TimeUnit.SECONDS);
            msg.markPublished();
        } catch (Exception e) {
            msg.markFailed(e.getMessage());
        }
        outboxRepository.save(msg);
    });
}
```

### 3. 주문 취소 Saga — Choreography 방식

```
취소 Saga 흐름:

고객: "주문 취소" 요청
  ↓
주문 Context:
  Order.cancel() → [CANCELLED 상태]
  OrderCancelled 이벤트 발행 (Outbox)

재고 Context:
  OrderCancelled 수신 → 재고 복원
  InventoryRestored 이벤트 발행

결제 Context:
  OrderCancelled 수신 (결제 완료 상태라면)
  → 환불 요청 → PG사 환불 API 호출
  → RefundCompleted 이벤트 발행

주문 Context:
  RefundCompleted 수신 → Order 환불 완료 기록
  → 고객에게 취소/환불 완료 알림 이메일
```

```java
// 재고 Context: OrderCancelled 이벤트 처리
@KafkaListener(topics = "order.events", groupId = "inventory-service")
@Transactional
public void onOrderCancelled(EventEnvelope envelope) {
    if (!"OrderCancelled".equals(envelope.eventType())) return;

    OrderCancelled event = deserialize(envelope.payload(), OrderCancelled.class);

    // 멱등성: 이미 처리한 이벤트인지 확인
    if (processedEvents.contains(event.eventId())) {
        log.info("중복 이벤트 무시: {}", event.eventId());
        return;
    }

    event.lines().forEach(line -> {
        inventoryRepository.findByProductId(line.productId())
            .ifPresent(inv -> {
                inv.restore(line.quantity());  // 보상: 재고 복원
                inventoryRepository.save(inv);
            });
    });

    processedEvents.mark(event.eventId());
    publishEvent(new InventoryRestored(event.orderId(), event.lines()));
}

// 결제 Context: OrderCancelled 이벤트 처리
@KafkaListener(topics = "order.events", groupId = "payment-service")
@Transactional
public void onOrderCancelled(EventEnvelope envelope) {
    if (!"OrderCancelled".equals(envelope.eventType())) return;

    OrderCancelled event = deserialize(envelope.payload(), OrderCancelled.class);

    // 결제가 완료된 상태인지 확인
    paymentRepository.findByOrderId(event.orderId()).ifPresent(payment -> {
        if (payment.isApproved()) {
            payment.refund();  // PG사 환불 API 호출
            paymentRepository.save(payment);
            publishEvent(new RefundCompleted(event.orderId(), payment.amount()));
        }
    });
}
```

### 4. 장애 시나리오별 동작

```
시나리오 A: 재고 Context 장애 중 주문 발생

  주문 저장 + Outbox 저장: 성공 (트랜잭션 완료)
  Kafka 발행: 성공 (Outbox Relay가 처리)
  재고 Context: 장애 중
  → Kafka Consumer가 다운됨
  → 재고 Context 복구 → Consumer 재시작
  → Kafka 오프셋부터 재처리
  → InventoryDecreased 처리됨
  
  결과: 최종적으로 재고 차감됨 (수 분 지연)
        주문은 정상 완료 (재고 Context 장애와 무관)

시나리오 B: PG사 타임아웃 (결제 Saga 실패)

  주문: PENDING 상태 저장
  결제 요청 → PG사 타임아웃 (30초 대기)
  → PaymentFailed 이벤트 발행
  → Order.cancel("결제 실패") 처리
  → 재고 복원 (예약 해제)
  
  결과: 주문 취소 + 재고 원복 (Eventually Consistent)

시나리오 C: 주문 Outbox 처리 중 서버 재시작

  Kafka 발행 중 서버 재시작
  → OutboxMessage.status = PENDING 그대로
  → 서버 재시작 후 Relay가 PENDING 메시지 재처리
  → at-least-once: 동일 이벤트 두 번 발행 가능
  → 소비자 멱등성으로 안전 처리
```

---

## 💻 실전 코드

### 이벤트 소비자 멱등성 처리

```java
// 멱등성 테이블 기반 중복 처리
@Entity
@Table(name = "processed_events",
       uniqueConstraints = @UniqueConstraint(columnNames = {"event_id", "consumer_group"}))
public class ProcessedEvent {
    @Id @GeneratedValue
    private Long id;

    @Column(name = "event_id", nullable = false)
    private String eventId;

    @Column(name = "consumer_group", nullable = false)
    private String consumerGroup;

    @Column(name = "processed_at", nullable = false)
    private LocalDateTime processedAt;
}

// 공통 멱등 처리기
@Component
public class IdempotentEventProcessor {

    private final ProcessedEventRepository processedEventRepo;

    @Transactional
    public boolean processIfNotDuplicate(String eventId, String consumerGroup,
                                          Runnable handler) {
        try {
            processedEventRepo.save(new ProcessedEvent(eventId, consumerGroup, LocalDateTime.now()));
            handler.run();
            return true;
        } catch (DataIntegrityViolationException e) {
            // 유니크 제약 위반 = 이미 처리된 이벤트
            log.info("중복 이벤트 무시: eventId={}, consumer={}", eventId, consumerGroup);
            return false;
        }
    }
}

// 사용
@KafkaListener(topics = "order.events", groupId = "inventory-service")
public void handle(EventEnvelope envelope) {
    idempotentProcessor.processIfNotDuplicate(
        envelope.eventId(), "inventory-service",
        () -> handleOrderPlaced(deserialize(envelope))
    );
}
```

---

## 📊 설계 비교

```
단일 트랜잭션 vs 이벤트 기반 Saga:

                단일 트랜잭션          이벤트 기반 Saga
────────────┼──────────────────────┼──────────────────────────
일관성       │ 강한 일관성            │ 결과적 일관성
────────────┼──────────────────────┼──────────────────────────
가용성       │ 낮음 (하나 장애 전체)  │ 높음 (격리)
────────────┼──────────────────────┼──────────────────────────
응답 시간   │ 모든 처리 합산         │ 핵심만 동기 (빠름)
────────────┼──────────────────────┼──────────────────────────
PG사 연동   │ XA 불가 (현실 불가)   │ 이벤트로 연동 가능
────────────┼──────────────────────┼──────────────────────────
보상 로직   │ 자동 롤백              │ 보상 트랜잭션 설계 필요
────────────┼──────────────────────┼──────────────────────────
디버깅      │ 스택 트레이스 명확     │ 이벤트 추적 필요
```

---

## ⚖️ 트레이드오프

```
Choreography Saga (현재 설계):
  장점: 각 Context 독립, 결합도 낮음
  단점: 전체 흐름이 코드에 명시적이지 않음
  
  전자상거래 취소 흐름이 3~4단계면 Choreography로 충분
  더 복잡해지면 (5단계 이상) Orchestration 고려

이벤트 스키마 진화:
  OrderPlaced v1 → v2 (쿠폰 필드 추가)
  기존 소비자: nullable 필드로 하위 호환
  Schema Registry로 스키마 버전 관리 권장
```

---

## 📌 핵심 정리

```
전자상거래 Context 통합 핵심:

이벤트 흐름:
  OrderPlaced → 재고차감, 배송생성, 이메일, 포인트 (비동기)
  PaymentApproved → 배송시작 (비동기)
  OrderCancelled → 재고복원, 환불 (보상 Saga)

Outbox Pattern:
  주문 저장 + 이벤트 저장 = 같은 트랜잭션
  Relay → Kafka (재시도 가능)
  소비자 멱등성 (DB 유니크 제약)

취소 Saga (Choreography):
  OrderCancelled → [재고복원 | 환불요청] 병렬 처리
  각 Context가 자신의 보상 로직 독립 실행

장애 격리:
  재고 Context 장애 → 주문 성공 (Eventually Consistent)
  PG사 타임아웃 → 주문 취소 + 보상 Saga
```

---

## 🤔 생각해볼 문제

**Q1.** 재고 차감과 결제 중 어느 것을 먼저 처리해야 하는가? 순서에 따른 위험은 무엇인가?

<details>
<summary>해설 보기</summary>

**"결제 먼저" 방식이 더 안전합니다. 순서별 위험을 분석합니다.**

```
재고 차감 먼저:
  재고 차감 → 결제 시도 → 결제 실패
  → 재고 복원 필요 (보상)
  → 복원 실패 시 재고 손실 (재고 있는데 팔 수 없음)
  
  위험: 재고 차감 후 결제 실패가 많으면 재고 부정확

결제 먼저 (권장):
  결제 승인 → 재고 차감 시도 → 재고 없음
  → 결제 환불 + 주문 취소 (보상)
  → 환불은 비즈니스적으로 처리 가능 (3-5 영업일)
  
  위험: 결제됐는데 재고 없음 → 고객 불편
  하지만 이 케이스가 더 드물고 처리 가능

현실적 해결:
  주문 시 재고 "예약" (차감 아님) + 결제 승인
  결제 완료 후 예약 → 실제 차감 확정
  예약 타임아웃 (30분) → 자동 해제
```

</details>

---

**Q2.** 주문 취소 Saga에서 환불이 완료됐는데 재고 복원이 실패하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**데이터 불일치가 발생합니다 — Dead Letter Queue + 수동 처리로 관리합니다.**

```
발생 상황:
  RefundCompleted 이벤트 → 재고 Context 장애 → 복원 실패
  → 환불은 됐지만 재고는 차감된 상태 유지
  → 실제로 제품이 있는데 시스템은 없다고 표시

자동 복구 시도:
  Kafka Consumer 재시작 후 재처리
  (at-least-once + 멱등성 → 중복 없이 재시도)

자동 복구 실패 시:
  Dead Letter Queue로 이동
  운영 알림 (슬랙, PagerDuty)
  운영팀 수동 재고 조정

모니터링:
  DLQ 메시지 수 알림
  "재고 복원 실패" 전용 대시보드
  정기 재고 실사로 불일치 감지

완벽한 자동화는 불가능 — 마지막 방어선은 사람
```

</details>

---

**Q3.** `OrderPlaced` 이벤트가 크면(Fat Event) 개인정보가 포함될 수 있다. 어떻게 설계해야 하는가?

<details>
<summary>해설 보기</summary>

**개인정보는 이벤트에서 제외하고 별도 조회로 처리합니다.**

```java
// 개인정보 제외 버전
public record OrderPlaced(
    UUID eventId,
    OrderId orderId,
    CustomerId customerId,          // ID만 (이름/이메일 없음)
    List<OrderLineSnapshot> lines,  // 상품 정보만 (개인정보 없음)
    Money totalAmount,
    LocalDate estimatedDelivery,
    LocalDateTime placedAt
) {}

// 배송 Context: 배송지 주소 필요 시 별도 API 조회
@EventListener
public void on(OrderPlaced event) {
    // 배송 Context가 주소 필요 → 회원 Context API 조회
    ShippingAddress address = membershipPort.getShippingAddress(event.customerId());
    Shipment shipment = Shipment.createFor(event.orderId(), address, event.lines());
    shipmentRepository.save(shipment);
}

// 또는 암호화 후 포함 (GDPR 고려)
// 삭제 요청 시 이벤트 로그의 암호화 키 삭제 → 복호화 불가
```

이벤트 로그(Kafka, ELK)는 많은 곳에 저장되므로 개인정보 포함은 GDPR/개인정보보호법 위반 위험이 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: 주문 Aggregate 설계](./02-order-aggregate.md)** | **[홈으로 🏠](../README.md)** | **[다음: 읽기 모델 분리 ➡️](./04-read-model-separation.md)**

</div>
