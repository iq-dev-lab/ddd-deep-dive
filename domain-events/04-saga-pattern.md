# Saga Pattern — 분산 트랜잭션 없는 비즈니스 프로세스 조율

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 2PC 없이 여러 Bounded Context에 걸친 비즈니스 프로세스를 어떻게 조율하는가?
- Choreography Saga와 Orchestration Saga의 선택 기준은 무엇인가?
- 보상 트랜잭션(Compensating Transaction)은 어떻게 설계하는가?
- Saga 중간에 실패하면 어떻게 일관성을 복구하는가?
- Saga 상태 관리와 모니터링은 어떻게 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"주문 → 결제 → 재고 차감 → 배송 예약"이 여러 서비스에 걸쳐 있을 때, 2PC(Two-Phase Commit)는 이론적으로 원자성을 보장하지만 실제로는 분산 환경에서 성능과 가용성을 심각하게 저하시킨다. 하나의 서비스가 응답 없으면 전체 시스템이 블로킹된다.

Saga는 긴 트랜잭션을 **일련의 짧은 로컬 트랜잭션**으로 분해하고, 실패 시 이미 완료된 단계를 **보상 트랜잭션**으로 되돌리는 패턴이다. 2PC 없이 결과적 일관성을 달성하는 실용적 해법이다.

---

## 😱 흔한 실수 (Before — 2PC 또는 거대 트랜잭션)

```java
// 안티패턴 1: 분산 2PC (실제로 거의 작동 안 함)
@Transactional  // JTA/XA 분산 트랜잭션 시도
public void placeOrder(PlaceOrderCommand command) {
    // 여러 DB에 걸친 하나의 트랜잭션
    orderRepository.save(order);          // DB A
    paymentService.charge(amount);        // DB B (또는 외부 PG사)
    inventoryService.decrease(items);     // DB C
    shippingService.reserve(address);     // DB D
    // → XA 트랜잭션: 모든 참여자가 Prepare → 모두 Commit
    // → 하나라도 장애 → 전체 블로킹
    // → PG사는 XA를 지원하지 않음 (불가능한 요구사항)
}

// 안티패턴 2: 거대 동기 호출 체인
@Transactional
public void placeOrder(PlaceOrderCommand command) {
    Order order = Order.place(...);
    orderRepository.save(order);

    // 순차 동기 호출 — 각각 수초 걸릴 수 있음
    PaymentResult payment = paymentService.charge(order.totalAmount());  // 외부 PG사 호출
    if (!payment.isSuccess()) {
        order.cancel("결제 실패");
        return;
    }
    inventoryService.decrease(order.lines());  // 재고 서비스 호출
    shippingService.reserve(order.address());   // 배송 서비스 호출

    // 결제 성공 → 재고 차감 실패 → 어떻게 결제를 돌려주는가?
    // 보상 로직이 없으면 결제됐는데 주문이 불완전한 상태
}
```

---

## ✨ 올바른 접근 (After — Saga Pattern)

### Choreography Saga (이벤트 반응형)

```
이벤트 흐름:

OrderPlaced → 결제 서비스 수신
    → PaymentApproved → 재고 서비스 수신
        → InventoryDecreased → 배송 서비스 수신
            → ShippingReserved → 완료

실패 흐름:
OrderPlaced → PaymentApproved → InventoryDecreaseFailed
    → 재고 실패 이벤트 발행
        → 결제 서비스 수신 → PaymentRefunded (보상)
            → 주문 서비스 수신 → OrderCancelled (보상)
```

```java
// 결제 서비스: OrderPlaced 이벤트를 받아 결제 처리
@KafkaListener(topics = "order.events")
@Transactional
public void on(OrderPlaced event) {
    try {
        Payment payment = paymentDomainService.charge(event.customerId(), event.totalAmount());
        paymentRepository.save(payment);
        // 성공 → 다음 단계로 이벤트
        publishEvent(new PaymentApproved(payment.id(), event.orderId(), event.totalAmount()));
    } catch (PaymentFailedException e) {
        // 실패 → 보상 이벤트 발행
        publishEvent(new PaymentFailed(event.orderId(), e.reason()));
    }
}

// 주문 서비스: 결제 실패 시 주문 취소 (보상)
@KafkaListener(topics = "payment.events")
@Transactional
public void on(PaymentFailed event) {
    Order order = orderRepository.findById(event.orderId()).orElseThrow();
    order.cancel("결제 실패: " + event.reason());  // 보상 트랜잭션
    orderRepository.save(order);
    publishEvent(new OrderCancelled(event.orderId(), "결제 실패"));
}
```

### Orchestration Saga (중앙 조율자)

```java
// SagaOrchestrator: 전체 프로세스를 중앙에서 관리
@Service
@Transactional
public class OrderPlacementSaga {

    public void start(OrderId orderId) {
        SagaState saga = SagaState.start(orderId);
        sagaRepository.save(saga);

        // Step 1: 결제 요청 (Command 발행)
        publishCommand(new ChargePaymentCommand(orderId, saga.amount()));
    }

    @KafkaListener(topics = "saga.payment.response")
    @Transactional
    public void onPaymentResponse(PaymentResponse response) {
        SagaState saga = sagaRepository.findByOrderId(response.orderId()).orElseThrow();

        if (response.isSuccess()) {
            saga.markPaymentDone(response.paymentId());
            sagaRepository.save(saga);
            // Step 2: 재고 차감 요청
            publishCommand(new DecreaseInventoryCommand(response.orderId(), saga.lines()));
        } else {
            saga.markFailed("결제 실패: " + response.reason());
            sagaRepository.save(saga);
            // 보상: 주문 취소
            publishCommand(new CancelOrderCommand(response.orderId(), "결제 실패"));
        }
    }

    @KafkaListener(topics = "saga.inventory.response")
    @Transactional
    public void onInventoryResponse(InventoryResponse response) {
        SagaState saga = sagaRepository.findByOrderId(response.orderId()).orElseThrow();

        if (response.isSuccess()) {
            saga.markInventoryDone();
            sagaRepository.save(saga);
            // Step 3: 배송 예약
            publishCommand(new ReserveShippingCommand(response.orderId(), saga.address()));
        } else {
            saga.markFailed("재고 부족");
            sagaRepository.save(saga);
            // 보상: 결제 환불 + 주문 취소
            publishCommand(new RefundPaymentCommand(response.orderId(), saga.paymentId()));
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Choreography vs Orchestration 선택 기준

```
Choreography Saga (이벤트 반응형):

구조: 각 서비스가 이벤트에 반응해서 다음 단계 처리
장점:
  - 중앙 조율자 없음 → 낮은 결합도
  - 각 서비스가 독립적으로 배포/확장
  - 단순한 경우 구현이 직관적

단점:
  - 전체 흐름이 코드에 명시적으로 보이지 않음
  - 디버깅 어려움 (이벤트 흐름 추적)
  - 복잡한 실패 시나리오에서 보상 로직 분산

적합한 경우:
  3단계 이하의 단순한 흐름
  서비스 간 결합을 최소화해야 할 때
  이벤트 기반 팀 문화

─────────────────────────────────────────────

Orchestration Saga (중앙 조율자):

구조: Saga Orchestrator가 각 서비스에 Command를 보내고 응답을 받음
장점:
  - 전체 흐름이 Orchestrator 코드에 명시적
  - 복잡한 조건 분기, 보상 로직 관리 용이
  - 실패 지점 파악과 재시도 용이

단점:
  - Orchestrator가 모든 서비스를 알게 됨 (결합도 약간 증가)
  - Orchestrator 자체가 단일 장애점 가능성
  - 추가 코드 (Orchestrator + 상태 관리)

적합한 경우:
  복잡한 비즈니스 프로세스 (5단계 이상)
  복잡한 보상 로직이 있을 때
  전체 흐름 가시성이 중요할 때
```

### 2. 보상 트랜잭션 설계

```
보상 트랜잭션 원칙:
  "이미 완료된 단계를 비즈니스적으로 되돌리는 작업"
  → 기술적 롤백(ROLLBACK)이 아님
  → 별도의 비즈니스 로직으로 표현됨

보상 트랜잭션 예시:

Step         정방향                    보상 (역방향)
─────────────────────────────────────────────────────────
1. 주문      OrderPlaced              → OrderCancelled
2. 결제      PaymentApproved          → PaymentRefunded
3. 재고      InventoryDecreased       → InventoryRestored
4. 배송 예약  ShippingReserved         → ShippingCancelled

보상 불가능한 경우:
  이미 배송 출발한 패키지 → 회수 요청 (비용 발생)
  이미 발송된 이메일 → 취소 불가 (사과 이메일로 보상)
  결제 환불 → 3~5 영업일 소요 (즉각 취소 불가)

→ 보상이 완벽하지 않은 경우 → 운영팀에 알림 + 수동 처리 프로세스
```

### 3. Saga 상태 관리

```java
// Saga 상태 추적 테이블
@Entity
@Table(name = "order_placement_saga")
public class OrderPlacementSagaState {

    @Id
    private UUID sagaId;
    private OrderId orderId;

    @Enumerated(EnumType.STRING)
    private SagaStep currentStep;  // PAYMENT, INVENTORY, SHIPPING, COMPLETED, FAILED

    @Enumerated(EnumType.STRING)
    private SagaStatus status;     // IN_PROGRESS, COMPLETED, COMPENSATING, FAILED

    private PaymentId paymentId;   // 보상 시 사용 (환불용)
    private String failureReason;
    private LocalDateTime startedAt;
    private LocalDateTime completedAt;

    public boolean isCompensating() {
        return status == SagaStatus.COMPENSATING;
    }

    public void markPaymentDone(PaymentId paymentId) {
        this.paymentId = paymentId;
        this.currentStep = SagaStep.INVENTORY;
    }

    public void startCompensation(String reason) {
        this.status = SagaStatus.COMPENSATING;
        this.failureReason = reason;
    }
}
```

---

## 💻 실전 코드

### Choreography Saga — 보상 흐름 완전 구현

```java
// 재고 서비스: PaymentApproved 이벤트 수신 후 재고 차감 시도
@Component
public class InventoryChoreographySagaHandler {

    @KafkaListener(topics = "payment.events", groupId = "inventory-service")
    @Transactional
    public void onPaymentApproved(PaymentApproved event) {
        try {
            List<InventoryDecreaseResult> results = event.lines().stream()
                .map(line -> decreaseInventory(line.productId(), line.quantity()))
                .collect(toList());

            boolean anyFailed = results.stream().anyMatch(r -> !r.isSuccess());
            if (anyFailed) {
                // 일부 성공한 재고 차감 원복 (부분 보상)
                results.stream()
                    .filter(InventoryDecreaseResult::isSuccess)
                    .forEach(r -> inventoryService.restore(r.productId(), r.quantity()));

                publishEvent(new InventoryDecreaseFailed(
                    event.orderId(), event.paymentId(), "재고 부족"
                ));
            } else {
                publishEvent(new InventoryDecreased(event.orderId(), event.paymentId(), results));
            }
        } catch (Exception e) {
            publishEvent(new InventoryDecreaseFailed(event.orderId(), event.paymentId(), e.getMessage()));
        }
    }
}

// 결제 서비스: 재고 실패 시 결제 환불 (보상)
@Component
public class PaymentCompensationHandler {

    @KafkaListener(topics = "inventory.events", groupId = "payment-service")
    @Transactional
    public void onInventoryDecreaseFailed(InventoryDecreaseFailed event) {
        Payment payment = paymentRepository.findById(event.paymentId()).orElseThrow();
        payment.refund("재고 부족으로 인한 자동 환불");
        paymentRepository.save(payment);
        publishEvent(new PaymentRefunded(event.orderId(), event.paymentId(), payment.amount()));
    }
}

// 주문 서비스: 결제 환불 완료 → 주문 취소 (보상)
@Component
public class OrderCompensationHandler {

    @KafkaListener(topics = "payment.events", groupId = "order-service")
    @Transactional
    public void onPaymentRefunded(PaymentRefunded event) {
        Order order = orderRepository.findById(event.orderId()).orElseThrow();
        order.cancel("결제 환불 완료");
        orderRepository.save(order);
    }
}
```

---

## 📊 설계 비교

```
2PC vs Saga:

                2PC (분산 트랜잭션)      Saga
────────────┼──────────────────────┼──────────────────────────
일관성       │ 강한 일관성             │ 결과적 일관성
────────────┼──────────────────────┼──────────────────────────
가용성       │ 낮음 (하나 장애 → 전체) │ 높음 (각 서비스 독립)
────────────┼──────────────────────┼──────────────────────────
성능         │ 낮음 (Prepare 단계)    │ 높음 (로컬 트랜잭션)
────────────┼──────────────────────┼──────────────────────────
복잡도       │ 인프라 복잡 (XA)        │ 보상 로직 복잡
────────────┼──────────────────────┼──────────────────────────
외부 시스템  │ ❌ XA 미지원 시 불가    │ ✅ 이벤트로 연동 가능
────────────┼──────────────────────┼──────────────────────────
디버깅       │ 상대적으로 단순          │ 이벤트 추적 필요
```

---

## ⚖️ 트레이드오프

```
Saga의 어려움:
  ① 보상 트랜잭션의 복잡도
     "이미 배송된 물건은 어떻게 보상?" — 비즈니스 정책 필요
     보상이 불가능한 단계 → 운영 개입 프로세스

  ② 격리 수준 부재 (ACD but not I)
     Saga는 격리(Isolation) 없음
     중간 상태가 다른 Saga에 노출될 수 있음
     → 대응: Semantic Lock (진행 중 표시), Countermeasures

  ③ 디버깅 어려움
     실패 시나리오가 이벤트로 분산
     → 분산 트레이싱 필수 (Zipkin, Jaeger)
     → Saga 상태 추적 테이블

현실적 조언:
  처음에는 Choreography로 단순하게
  복잡해지면 Orchestration으로
  보상 로직은 도메인 전문가와 함께 설계 (비즈니스 결정)
```

---

## 📌 핵심 정리

```
Saga Pattern 핵심:

목적: 2PC 없이 여러 서비스에 걸친 비즈니스 프로세스 조율
방법: 로컬 트랜잭션 + 이벤트/Command + 보상 트랜잭션

Choreography:
  이벤트 반응형, 서비스 독립
  단순 흐름에 적합, 복잡하면 디버깅 어려움

Orchestration:
  중앙 조율자, 흐름 명시적
  복잡한 보상 로직, 가시성 필요 시 적합

보상 트랜잭션:
  기술적 롤백이 아닌 비즈니스 되돌리기
  각 단계별 보상 정의 (OrderCancelled, PaymentRefunded)
  보상 불가 단계 → 운영 프로세스

일관성:
  결과적 일관성 (Eventually Consistent)
  중간 상태 노출 가능 → Semantic Lock으로 방어
```

---

## 🤔 생각해볼 문제

**Q1.** Saga 중간에 Orchestrator 서버가 다운되면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Saga 상태를 DB에 영속화하므로, 재시작 후 복구 가능합니다.**

```java
// 서버 재시작 후 미완료 Saga 복구
@Scheduled(fixedDelay = 30000)  // 30초마다
public void recoverIncompleteSagas() {
    List<OrderPlacementSagaState> inProgress = sagaRepository
        .findByStatusAndStartedAtBefore(SagaStatus.IN_PROGRESS, LocalDateTime.now().minusMinutes(5));
    // 5분 이상 진행 중인 Saga → 장애로 판단 → 재시도 또는 보상

    inProgress.forEach(saga -> {
        if (saga.retryCount() < 3) {
            retrySagaFromCurrentStep(saga);  // 현재 단계부터 재시도
        } else {
            startCompensation(saga);          // 보상 시작
        }
    });
}
```

Saga 상태 테이블이 있으면: 재시작 후 currentStep에서 재개 가능. 상태가 없으면: 재시작 후 어느 단계까지 됐는지 알 수 없어 보상만 가능합니다.

</details>

---

**Q2.** "결제는 성공했는데 재고 차감이 실패했다"는 상황에서 보상 이벤트 `PaymentRefunded`가 발행됐지만, 결제 서비스가 현재 다운 상태다. 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Kafka + Outbox Pattern이 있다면 결제 서비스 복구 후 자동 처리됩니다.**

```
흐름:
  재고 서비스: InventoryDecreaseFailed 이벤트 발행 → Kafka에 영속됨
  결제 서비스: 현재 다운
  
  결제 서비스 복구:
  → Kafka Consumer가 미처리 메시지 재처리
  → InventoryDecreaseFailed 이벤트 수신
  → PaymentRefunded 처리

Kafka의 Consumer Group Offset:
  다운된 동안 커밋 안 된 오프셋
  → 재시작 후 해당 오프셋부터 재처리
  → 멱등성이 있다면 중복 처리해도 안전
```

이것이 Kafka 기반 Saga의 장점입니다. 이벤트가 Kafka에 남아 있으면 처리 순서대로 자동 복구됩니다. 멱등성 설계가 선행돼야 합니다.

</details>

---

**Q3.** Saga에서 "격리 없음(No Isolation)" 문제로 인해 중간 상태를 다른 요청이 읽으면 어떤 문제가 생기고, 어떻게 방어하는가?

<details>
<summary>해설 보기</summary>

**"Dirty Read" 문제가 발생할 수 있습니다.**

```
시나리오:
  Saga 1: Order A 처리 중 (재고 차감 완료, 결제 진행 중)
  Query: "Order A의 상태는?" → "재고는 차감됐지만 결제 미완료"

  다른 사용자가 동일 상품 주문 시:
  재고 조회: 차감됐으므로 "재고 없음" → 주문 실패
  하지만 Saga 1이 실패하면 재고가 복원될 것
  → 불필요한 주문 거절

대응 방법:

방법 1: Semantic Lock
  Saga 진행 중인 Aggregate에 "처리 중" 플래그
  다른 Saga가 같은 Aggregate를 처리하려 하면 대기 또는 거절

  Order.sagaStatus = PROCESSING  // Saga 시작 시 설정
  // 다른 주문 취소 요청 → "처리 중인 주문은 취소 불가"

방법 2: Countermeasures (대응 조치)
  재시도 가능한 트랜잭션 (Retriable Transaction)
  → 실패 시 자동 재시도

  보상 가능한 트랜잭션 (Compensatable Transaction)
  → 실패 시 보상

방법 3: 피벗 트랜잭션 설계
  되돌릴 수 없는 단계(결제 승인) 전까지는 보상 가능하도록 설계
  피벗 이후는 보상 불가 → 운영 프로세스로 처리
```

</details>

---

<div align="center">

**[⬅️ 이전: Outbox Pattern](./03-outbox-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: Context 간 데이터 동기화 ➡️](./05-context-data-sync.md)**

</div>
