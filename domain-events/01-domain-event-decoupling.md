# Domain Event가 결합도를 낮추는 원리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 이벤트 발행자가 구독자를 모르는 설계가 어떻게 결합도를 낮추는가?
- 동기 직접 호출과 이벤트 기반 통신의 일관성 트레이드오프는 무엇인가?
- Bounded Context 간 통신에서 이벤트가 적합한 경우와 그렇지 않은 경우는?
- 이벤트 기반으로 전환 시 "결과적 일관성"을 비즈니스가 수용해야 하는 이유는?
- 이벤트 기반 설계가 Open/Closed Principle(OCP)과 어떻게 연결되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Ch3-08에서 Domain Event의 기본 개념을 다뤘다. 이 문서는 **왜 이벤트가 결합도를 낮추는가**를 더 깊이 분석한다. 결합도(Coupling)는 소프트웨어 품질의 핵심 지표다. 발행자가 구독자를 알면, 구독자가 늘어날수록 발행자 코드도 함께 커진다. 구독자가 바뀌면 발행자도 재배포해야 한다.

이벤트 기반 설계는 이 연결을 끊는다. 발행자는 "무슨 일이 일어났다"를 이벤트로 알리고, 누가 반응할지는 모른다. 이것이 마이크로서비스 경계를 가능하게 하고, 팀 간 독립 개발을 가능하게 하는 메커니즘이다.

---

## 😱 흔한 실수 (Before — 직접 호출로 강한 결합)

```java
// 주문 완료 시 연쇄 직접 호출 — 결합도 누적
@Service
@Transactional
public class OrderService {

    // 주문 서비스가 알아야 하는 서비스 목록이 계속 늘어남
    private final EmailService emailService;
    private final PointService pointService;
    private final InventoryService inventoryService;
    private final MarketingService marketingService;
    private final SlackNotificationService slackService;

    public void placeOrder(PlaceOrderRequest request) {
        Order order = Order.place(...);
        orderRepository.save(order);

        // 기능이 추가될 때마다 여기 줄이 늘어남
        emailService.sendConfirmation(order);            // v1에서 추가
        pointService.earn(order.customerId(), order.totalAmount()); // v2에서 추가
        inventoryService.decrease(order.lines());        // v3에서 추가
        marketingService.recordPurchase(order);          // v4에서 추가
        slackService.notifyLargeOrder(order);            // v5에서 추가

        // "리뷰 유도 메시지 발송" 추가 → 또 한 줄
        // "파트너 수수료 계산" 추가 → 또 한 줄
        // "물류 예약" 추가 → 또 한 줄
    }
}
```

```
결합도 누적의 현실:

구조적 결합:
  OrderService → EmailService
  OrderService → PointService
  OrderService → InventoryService
  OrderService → MarketingService
  OrderService → SlackNotificationService
  
  → OrderService가 5개 서비스에 의존
  → EmailService 패키지 변경 → OrderService 재컴파일

배포 결합:
  "슬랙 알림 기능 제거" → OrderService 코드 수정 후 재배포
  → 주문 기능과 무관한 변경이 OrderService 배포를 요구

장애 결합:
  SlackNotificationService가 느림 (1초 지연)
  → 모든 주문 응답이 1초 느려짐
  
  InventoryService 장애
  → 주문 서비스 전체 실패 (트랜잭션 롤백)

테스트 결합:
  OrderService 단위 테스트에 5개 Mock 필요
  EmailService Mock이 바뀌면 OrderService 테스트 재작성
```

---

## ✨ 올바른 접근 (After — 이벤트로 결합 제거)

```java
// 주문 서비스 — 이벤트만 발행, 구독자를 모름
@Service
@Transactional
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.place(...);
        orderRepository.save(order);

        // 이벤트 발행: 누가 받는지 모름 (OCP 적용)
        order.pullEvents().forEach(eventPublisher::publishEvent);

        return order.id();
        // 끝. 새 기능이 추가돼도 이 코드는 변하지 않음
    }
}

// 각 관심사가 독립적으로 구독
@Component
public class OrderConfirmationEmailHandler {
    @EventListener
    @Async
    public void on(OrderPlaced event) {
        emailService.sendConfirmation(event.customerId(), event.totalAmount());
    }
}

@Component
public class InventoryDecreaseHandler {
    @EventListener
    @Async
    public void on(OrderPlaced event) {
        inventoryService.decrease(event.lines());
    }
}

// 새 기능 추가 → OrderApplicationService 코드 수정 없이 핸들러만 추가
@Component
public class PartnerCommissionHandler {       // 신규 추가
    @EventListener
    @Async
    public void on(OrderPlaced event) {
        commissionService.calculate(event);
    }
}
```

```
결합도 비교:

Before: OrderService → [EmailService, PointService, InventoryService, ...]
After:  OrderApplicationService → [OrderPlaced 이벤트 타입만]
        각 Handler → [자신의 서비스만]

OCP 적용:
  "새 기능 추가" = "새 EventHandler 추가"
  기존 코드(OrderApplicationService) 수정 없음
  → 확장에 열려 있고, 변경에 닫혀 있음
```

---

## 🔬 내부 동작 원리

### 1. 결합도의 세 가지 차원

```
차원 1: 구조적 결합 (Structural Coupling)
  "A가 B의 클래스/인터페이스를 import한다"
  
  직접 호출: OrderService imports EmailService
    → EmailService 패키지/시그니처 변경 → OrderService 재컴파일
  
  이벤트 기반: OrderService는 OrderPlaced 이벤트 타입만 import
    → EmailHandler가 바뀌어도 OrderService 무관
    → 이벤트 타입이 안정적이면 구조적 결합 최소화

차원 2: 시간적 결합 (Temporal Coupling)
  "A와 B가 동시에 실행 가능해야 한다"
  
  동기 직접 호출:
    EmailService가 다운 → OrderService도 실패
    EmailService가 느림 → OrderService 응답 느려짐
  
  비동기 이벤트:
    EmailService가 다운 → 이벤트 큐에 쌓임, 복구 후 처리
    EmailService가 느림 → OrderService 응답에 영향 없음

차원 3: 의미적 결합 (Semantic Coupling)
  "A의 변경이 B의 동작에 영향을 미친다"
  
  "주문 취소" 시나리오:
    직접 호출: Order.cancel() 내부에서 inventory.restore() 직접 호출
      → Inventory의 restore() 시그니처가 바뀌면 Order도 변경
    
    이벤트: Order.cancel() → OrderCancelled 이벤트
      → Inventory가 이벤트 수신해서 restore()
      → Order는 Inventory의 내부 구현을 모름
```

### 2. 이벤트 기반이 적합한 경우 vs 직접 호출이 나은 경우

```
이벤트 기반이 적합한 경우:

  ✅ 다른 Bounded Context와의 통신
     주문 Context → 배송 Context
     발행자와 수신자가 다른 팀, 다른 배포 단위
  
  ✅ "부수 효과"가 비즈니스 핵심이 아닌 경우
     "주문 완료 → 이메일 발송" — 이메일이 안 가도 주문은 완료됨
     "주문 완료 → 마케팅 데이터 전송" — 선택적 부수 효과
  
  ✅ 결과적 일관성을 허용하는 경우
     "포인트 적립은 수 초 내에 반영되면 됨"
     → 즉각적 일관성 불필요
  
  ✅ 수신자가 여러 개이고 독립적으로 처리하는 경우
     OrderPlaced → 이메일, 포인트, 재고, 마케팅이 각자 처리

─────────────────────────────────────────────

직접 호출(동기)이 나은 경우:

  ✅ 즉각적 피드백이 비즈니스 필수인 경우
     "재고가 없으면 주문 불가" — 즉시 응답이 필요
     → 이벤트로 재고 확인은 불가 (응답 전에 재고 상태 알아야 함)
  
  ✅ 같은 Bounded Context 내 Aggregate 간 상호작용
     Order와 OrderLine: 같은 트랜잭션, 같은 Context
  
  ✅ 결과를 즉시 알아야 하는 경우
     "결제가 승인됐는가?" → PG사에 동기 HTTP 요청
     이벤트로 기다릴 수 없음
  
  ✅ 실패 시 즉각 롤백이 필요한 경우
     분산 트랜잭션 없이 즉각 일관성이 비즈니스 필수

판단 원칙:
  "이 작업의 성공/실패가 이 유스케이스의 성공/실패를 결정하는가?"
  → YES: 동기 호출 (재고 확인, 결제 승인)
  → NO:  이벤트 (이메일 발송, 포인트 적립, 마케팅 데이터)
```

### 3. 일관성 트레이드오프

```
직접 호출 (강한 일관성):

  @Transactional
  public void placeOrder(PlaceOrderCommand command) {
      Order order = Order.place(...);
      orderRepository.save(order);
      pointService.earn(...);      // 같은 트랜잭션
      inventoryService.decrease(); // 같은 트랜잭션
      // → 세 작업이 모두 성공 또는 모두 실패
      // → 항상 일관된 상태
  }

  비용:
    락 범위 증가 (order + point + inventory 테이블)
    처리 시간 증가 (세 서비스 순차 실행)
    InventoryService 장애 → 주문 실패

─────────────────────────────────────────────

이벤트 기반 (결과적 일관성):

  @Transactional
  public void placeOrder(PlaceOrderCommand command) {
      Order order = Order.place(...);
      orderRepository.save(order);   // 주문만 저장
      publishEvent(OrderPlaced);     // 이벤트만 발행
      // → 주문 저장만 원자적으로 보장
  }

  // 비동기로 각자 처리
  on(OrderPlaced) → pointService.earn()       // 별도 트랜잭션
  on(OrderPlaced) → inventoryService.decrease() // 별도 트랜잭션

  이점:
    주문 응답 즉시 (이메일/포인트 기다리지 않음)
    EmailService 장애가 주문에 영향 없음
    각 서비스가 독립적으로 스케일 가능
  
  비용:
    "주문 직후 포인트가 바로 보여야 한다" → 수 초 지연
    이벤트 처리 실패 시 재시도 로직 필요
    "포인트 적립 전 취소됐으면?" 등 복잡한 에지케이스

결정 요소:
  비즈니스가 수초 지연을 허용하는가? → 이벤트
  실패 시 보상이 가능한가? (포인트 차감으로 보상) → 이벤트
  실패가 즉각 비즈니스 문제인가? → 동기 호출
```

### 4. Bounded Context 간 이벤트의 특징

```
Context 내부 이벤트 vs Context 간 이벤트:

Context 내부 (Spring @EventListener):
  같은 JVM, 같은 트랜잭션 가능
  빠름 (메모리 내 전달)
  발행 실패 = 예외 (바로 인지)
  
  @TransactionalEventListener(phase = AFTER_COMMIT)
  // 트랜잭션 커밋 후 실행 → 원자성 어느 정도 보장

Context 간 (Kafka, RabbitMQ):
  다른 JVM, 다른 트랜잭션
  네트워크 지연 존재
  발행 실패 = 나중에 인지 (Outbox로 보완)
  at-least-once 전송 → 중복 처리 가능성
  
  → 소비자는 멱등하게 설계 필요

경계를 넘는 이벤트의 계약:
  이벤트 스키마가 공개 계약이 됨
  → Published Language (Ch2에서 다룬 패턴)
  → 스키마 변경 시 버전 관리 필요
  → 갑작스러운 필드 삭제 = 계약 위반
```

---

## 💻 실전 코드

### 이벤트 기반 결합도 측정 — ArchUnit

```java
// ArchUnit: Domain Event만을 통해 Context 경계를 넘는지 검증
@ArchTest
static ArchRule orderContextShouldNotDirectlyCallShipping =
    noClasses()
        .that().resideInAPackage("..order.application..")
        .should().dependOnClassesThat()
        .resideInAPackage("..shipping.application..")
        .because("주문 Context는 배송 Context의 Application Service를 직접 호출할 수 없습니다. " +
                 "이벤트를 통해 통신하세요.");

// 이벤트 타입은 공유 모듈에
@ArchTest
static ArchRule onlySharedEventsCanCrossContextBoundary =
    classes()
        .that().resideInAPackage("..order.domain.events..")
        .should().onlyBeAccessed().byClassesThat()
        .resideInAnyPackage("..order..", "..shared.events..");
```

### 결합도 낮은 이벤트 핸들러 패턴

```java
// 이벤트 핸들러 — 단일 책임, 독립 테스트 가능
@Component
public class OrderPlacedHandlers {

    // 각 핸들러는 독립적 — 하나 실패해도 다른 것에 영향 없음 (비동기)
    @EventListener
    @Async
    @Transactional
    public void earnPoints(OrderPlaced event) {
        pointWalletRepository.findByCustomerId(event.customerId())
            .ifPresent(wallet -> {
                wallet.earn(PointAmount.of(event.totalAmount()));
                pointWalletRepository.save(wallet);
            });
    }

    @EventListener
    @Async
    @Transactional
    public void decreaseInventory(OrderPlaced event) {
        event.lines().forEach(line ->
            inventoryRepository.findByProductId(line.productId())
                .ifPresent(inv -> {
                    inv.decrease(line.quantity());
                    inventoryRepository.save(inv);
                })
        );
    }
}

// 이벤트 핸들러 단위 테스트 — OrderService Mock 불필요
class OrderPlacedHandlersTest {

    private final InMemoryPointWalletRepository walletRepo = new InMemoryPointWalletRepository();
    private final OrderPlacedHandlers handlers = new OrderPlacedHandlers(walletRepo);

    @Test
    void earnPoints_onOrderPlaced() {
        PointWallet wallet = PointWallet.create(CUSTOMER_ID);
        walletRepo.save(wallet);

        handlers.earnPoints(new OrderPlaced(ORDER_ID, CUSTOMER_ID, Money.ofKrw(50_000), NOW));

        PointWallet updated = walletRepo.findByCustomerId(CUSTOMER_ID).orElseThrow();
        assertThat(updated.balance()).isEqualTo(PointAmount.of(500)); // 1%
    }
}
```

---

## 📊 설계 비교

```
직접 호출 vs 이벤트 기반:

                직접 호출               이벤트 기반
────────────┼──────────────────────┼──────────────────────────
결합도       │ 높음 (클래스 의존)     │ 낮음 (이벤트 타입만)
────────────┼──────────────────────┼──────────────────────────
OCP         │ ❌ 기능 추가 시 수정   │ ✅ 핸들러만 추가
────────────┼──────────────────────┼──────────────────────────
일관성      │ 강한 일관성 가능       │ 결과적 일관성
────────────┼──────────────────────┼──────────────────────────
응답 시간   │ 모든 처리 시간 합산    │ 즉시 응답 (비동기)
────────────┼──────────────────────┼──────────────────────────
장애 격리   │ ❌ 하나 장애 → 전체   │ ✅ 독립 처리
────────────┼──────────────────────┼──────────────────────────
테스트      │ 많은 Mock 필요        │ 핸들러 독립 테스트
────────────┼──────────────────────┼──────────────────────────
디버깅      │ 스택 트레이스 명확     │ 분산 추적 필요
────────────┼──────────────────────┼──────────────────────────
즉각 피드백  │ ✅ 가능               │ ❌ 비동기 결과 불명확
```

---

## ⚖️ 트레이드오프

```
이벤트 기반의 숨겨진 비용:
  ① 디버깅 복잡도
     "주문했는데 포인트가 안 쌓였습니다"
     → 이벤트가 발행됐는가? 핸들러가 수신했는가? 어디서 실패?
     → 분산 트레이싱 (Zipkin, Jaeger)이 없으면 추적 어려움

  ② 이벤트 계약 관리
     이벤트 스키마 변경 시 모든 소비자와 조율 필요
     → Schema Registry 도입 고려

  ③ 결과적 일관성의 비즈니스 설득
     "주문 직후 포인트가 안 보여요" — 고객 불만
     → UX로 해결: "포인트는 곧 적립됩니다" 메시지

  ④ 멱등성 구현
     at-least-once 전송 → 중복 이벤트 처리 설계 필요

언제 직접 호출을 유지할 것인가:
  팀이 작아서 이벤트 인프라 운영 부담이 클 때
  즉각 일관성이 비즈니스 필수일 때
  이벤트 기반의 복잡도가 단순함의 가치를 초과할 때
```

---

## 📌 핵심 정리

```
Domain Event 결합도 핵심:

이벤트가 결합도를 낮추는 원리:
  발행자: OrderPlaced 이벤트 타입만 알면 됨
  수신자: 자신의 관심 이벤트만 구독
  → 발행자-수신자 간 직접 의존 없음

OCP 적용:
  새 기능 = 새 EventHandler (기존 코드 수정 없음)
  제거 = Handler 삭제 (기존 코드 수정 없음)

이벤트 적합한 경우:
  다른 Context 간 통신
  부수 효과 (이메일, 포인트, 알림)
  결과적 일관성 허용 가능
  수신자 다수

직접 호출이 나은 경우:
  즉각 피드백 필요 (재고 확인, 결제 승인)
  즉각 일관성이 비즈니스 필수
  같은 트랜잭션 경계 내

일관성 트레이드오프:
  직접 호출: 강한 일관성, 락 범위 크고 응답 느림
  이벤트: 결과적 일관성, 빠른 응답, 장애 격리
```

---

## 🤔 생각해볼 문제

**Q1.** "주문 완료 시 재고를 차감해야 한다"는 요구사항이 있다. 재고 부족 시 주문 자체를 실패시켜야 하는가? 이벤트로 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**재고 확인(주문 가능 여부)은 동기, 재고 차감은 이벤트가 이상적입니다.**

```java
// 주문 접수 전: 재고 확인 (동기 — 즉각 피드백 필요)
@Transactional
public OrderId placeOrder(PlaceOrderCommand command) {
    // 재고 확인: 부족하면 즉시 실패 (고객에게 즉각 알려야 함)
    command.lines().forEach(line -> {
        Inventory inv = inventoryPort.findByProduct(line.productId()).orElseThrow();
        if (!inv.isAvailable(line.quantity())) {
            throw new OutOfStockException(line.productId());
        }
    });

    Order order = Order.place(...);
    orderRepository.save(order);

    // 재고 차감: 이벤트로 비동기 (주문 응답에 영향 없음)
    order.pullEvents().forEach(eventPublisher::publishEvent);
    return order.id();
}

// 재고 차감 핸들러 (비동기)
@EventListener @Async @Transactional
public void on(OrderPlaced event) {
    event.lines().forEach(line -> {
        Inventory inv = inventoryRepository.findByProduct(line.productId()).orElseThrow();
        inv.decrease(line.quantity());
        inventoryRepository.save(inv);
    });
}
```

**문제: 재고 확인과 차감 사이에 다른 주문이 끼어들면?**
- 낙관적 잠금(Optimistic Lock)으로 차감 시 충돌 감지
- 충돌 시 보상: OrderPlaced 이벤트 처리 실패 → OrderCancelled 이벤트 발행
- 이것이 Ch4-04에서 다루는 Saga 패턴의 시작점

</details>

---

**Q2.** 이벤트 핸들러가 실패했을 때 발행자에게 실패를 알려야 하는가?

<details>
<summary>해설 보기</summary>

**이벤트 기반 설계에서 발행자에게 실패를 동기로 알리는 것은 이벤트의 목적과 맞지 않습니다.**

**대신 이 방법들을 사용합니다:**

```
방법 1: 재시도 (Retry)
  핸들러 실패 → 일정 시간 후 재시도 (지수 백오프)
  @Retryable(maxAttempts = 3, backoff = @Backoff(delay = 1000, multiplier = 2))

방법 2: Dead Letter Queue (DLQ)
  재시도 한도 초과 → DLQ로 이동
  운영자가 DLQ 모니터링 → 수동 재처리 또는 보상

방법 3: 보상 이벤트
  핸들러 실패 → 역방향 이벤트 발행 (보상 트랜잭션)
  포인트 적립 실패 → PointEarnFailed → 주문 상태 업데이트 (Saga)

방법 4: 알림 (비동기)
  핸들러 실패 → 슬랙/PagerDuty 알림
  → 발행자에게 알리는 것이 아니라 운영팀에 알림
```

발행자가 "이메일 핸들러가 실패했는가?"를 알고 처리하려 한다면, 그것은 이벤트가 아닌 동기 호출이 적합한 상황입니다.

</details>

---

**Q3.** 한 서비스 내의 두 Aggregate 간 통신에 이벤트를 사용하는 것이 과잉 설계인가?

<details>
<summary>해설 보기</summary>

**과잉일 수도, 적절할 수도 있습니다. 맥락이 중요합니다.**

**같은 서비스 내에서도 이벤트가 적합한 경우:**
```java
// Order → PointWallet 간 이벤트 (같은 서비스, 다른 Aggregate)
// 장점: 나중에 PointWallet을 별도 서비스로 분리할 때 이벤트만 외부로 보내면 됨
// → 미래 분리를 위한 준비

@EventListener @Transactional
public void on(OrderPlaced event) {
    // PointWallet이 나중에 다른 서비스로 분리돼도 이 코드는 그대로 외부 이벤트 핸들러가 됨
    pointWalletRepository.findByCustomerId(event.customerId())
        .ifPresent(wallet -> wallet.earn(PointAmount.of(event.totalAmount())));
}
```

**직접 호출이 더 나은 경우:**
```java
// Order와 OrderLine: 같은 Aggregate 내부 → 이벤트 불필요
// Order와 Cart: 같은 Context, Cart → Order 전환은 유스케이스 흐름
// → Application Service에서 직접 Cart 조회 후 Order 생성이 명확
```

**결론:** 두 Aggregate가 미래에 다른 Context로 분리될 가능성이 있거나, 부수 효과로 처리하고 싶다면 이벤트 사용. 강한 일관성이 필요하거나 분리 가능성이 없다면 직접 호출이 더 단순합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Domain Event 설계](../tactical-design/08-domain-event-design.md)** | **[홈으로 🏠](../README.md)** | **[다음: 이벤트 발행 패턴 ➡️](./02-event-publishing-patterns.md)**

</div>
