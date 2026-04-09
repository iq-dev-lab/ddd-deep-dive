# Bounded Context와 마이크로서비스 — Context ≠ 서비스

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- "1 Bounded Context = 1 마이크로서비스"가 왜 항상 옳지 않은가?
- Context 경계가 정리되기 전에 서비스를 분리하면 왜 분산 모놀리스가 되는가?
- 하나의 Context가 여러 서비스로 분리되는 경우와 하나의 서비스에 여러 Context가 공존하는 경우의 트레이드오프는?
- 모놀리스에서 마이크로서비스로 이전하는 올바른 순서는?
- 지금 당장 마이크로서비스가 아니더라도 DDD를 적용하는 의미는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"마이크로서비스를 도입하면 DDD도 자동으로 되는 것 아닌가?" 라고 생각하는 팀이 많다. 결론부터 말하면 정반대다. **DDD 없이 마이크로서비스를 분리하면 "분산 모놀리스"가 된다.** 서비스가 물리적으로 분리됐지만, 서비스 간 강한 결합이 유지되어 배포가 더 어렵고 장애가 더 복잡하며 디버깅이 훨씬 어려운 상태다.

올바른 순서는 반대다. **Context 경계를 먼저 정리하고, 그 경계를 따라 서비스를 분리한다.** Context Map 없이 그은 서비스 경계는 비즈니스 현실과 맞지 않아 시간이 지나면서 계속 재조정이 필요하다.

---

## 😱 흔한 실수 (Before — Context 없이 서비스 분리)

```
상황: "우리도 마이크로서비스 해야 해!" — 6개월 프로젝트

팀의 결정:
  "주문, 배송, 결제, 회원을 각각 서비스로 분리하자"
  → 기술 기준으로 분리 (각 기능 = 하나의 서비스)
  → Context Map 분석 없음, Bounded Context 설계 없음

분리 결과:
  order-service    (주문 처리)
  shipping-service (배송 관리)
  payment-service  (결제 처리)
  member-service   (회원 관리)
```

```java
// 분산 모놀리스의 전형적 코드

// order-service의 OrderController
@PostMapping("/orders")
public OrderResponse placeOrder(@RequestBody PlaceOrderRequest request) {
    // 주문 처리를 위해 3개 서비스 동기 호출
    MemberResponse member = memberClient.getMember(request.getMemberId());    // 1. 회원 조회
    ProductResponse product = productClient.getProduct(request.getProductId()); // 2. 상품 조회
    InventoryResponse inventory = inventoryClient.checkStock(request.getProductId()); // 3. 재고 확인

    if (!inventory.isAvailable()) throw new OutOfStockException();

    Order order = Order.builder()
        .memberId(member.getId())
        .memberGrade(member.getGrade())       // ← 회원 서비스 모델이 주문 서비스에 침투
        .productId(product.getId())
        .productName(product.getName())       // ← 상품 서비스 모델이 주문 서비스에 침투
        .productPrice(product.getPrice())
        .build();

    orderRepository.save(order);

    // 주문 완료 후 또 다른 서비스 동기 호출
    paymentClient.initiatePayment(order.getId(), order.getTotalAmount()); // 4. 결제 시작
    inventoryClient.decreaseStock(request.getProductId(), request.getQuantity()); // 5. 재고 차감

    return OrderResponse.from(order);
}
```

```
분산 모놀리스의 현실:

하나의 주문 처리에 5개 서비스 동기 호출
→ 총 응답 시간 = 각 서비스 응답 시간의 합
→ member-service 200ms + product-service 150ms + inventory 100ms + payment 300ms = 750ms+

장애 전파:
  inventory-service 다운 → order-service 전체 주문 불가
  payment-service 느려짐 → order-service 전체 응답 지연
  → 하나의 서비스 장애가 전체 시스템으로 전파
  → 물리적으로 분리됐지만 장애는 모놀리스처럼 전파됨

배포의 악몽:
  "회원 등급 조회 API 응답 형식 변경" → member-service 배포
  → order-service, shipping-service, payment-service 모두 영향
  → 4개 서비스 동시 배포 조율 필요
  → 마이크로서비스의 "독립 배포" 이점이 사라짐

디버깅:
  "주문 실패 원인은?" → 5개 서비스의 로그를 Kibana에서 TraceId로 추적
  → 분산 트레이싱 없으면 디버깅 불가능
  → 모놀리스보다 디버깅이 훨씬 어려움

결론: 물리적으로 분리된 모놀리스 (최악의 조합)
  모놀리스의 단점 (강한 결합) +
  마이크로서비스의 단점 (네트워크 복잡도, 분산 트랜잭션) 동시에 가짐
```

---

## ✨ 올바른 접근 (After — Context 경계 먼저, 서비스 분리는 나중에)

```
Step 1: Context 경계 정의 (서비스 분리 전에 반드시)

  이벤트 스토밍 → Context Map:
    주문 Context: 주문 생성, 쿠폰/포인트 적용, 주문 상태 관리
    배송 Context: 배송 준비, 택배사 연동, 배송 추적
    결제 Context: PG사 연동, 결제 승인/취소/환불
    회원 Context: 가입, 로그인, 등급 관리

  Context 간 통합 패턴 결정:
    주문 ← 회원: Customer-Supplier (주문이 회원 정보 조회)
    주문 → 배송: Published Language (OrderPlaced 이벤트 발행)
    주문 → 결제: Customer-Supplier (주문이 결제 API 호출)

Step 2: 먼저 모놀리스 내 논리적 분리
  물리적 서비스 분리 전, 하나의 코드베이스 안에서 패키지로 경계 표현
  com.example.order / com.example.shipping / com.example.payment / com.example.member
  ArchUnit으로 패키지 간 의존 금지 강제
  → Context 경계가 안정적인지 검증

Step 3: 이벤트 기반 통신으로 교체
  서비스 간 동기 호출 → 이벤트로 교체
  주문 완료 → OrderPlaced 이벤트 발행
  배송 Context가 구독 → Shipment 생성
  → 이 단계에서 물리적 분리가 쉬워짐

Step 4: 필요한 Context만 분리 (전략적 결정)
  모든 Context를 동시에 분리할 필요 없음
  가장 독립적이고 변경이 빈번한 Context부터
  결제 Context: 외부 PG사 연동 변경이 잦음 → 먼저 분리
  배송 Context: 배송사 연동 독립적 → 분리 용이
  회원 Context: 다른 Context와 강한 결합 → 나중에
```

```java
// 올바른 서비스 분리 후 — 이벤트 기반 비동기 통신

// 주문 서비스 — 자신의 Context만 처리
@Service
@Transactional
public class OrderApplicationService {

    // 회원 서비스를 동기 호출하지 않음
    // 회원 정보는 필요한 것만 주문 시 Command에 포함
    public OrderId placeOrder(PlaceOrderCommand command) {
        // command에 memberId + 주문 시점 등급 포함 (이미 알고 있는 정보)
        Order order = Order.place(
            command.customerId(),
            command.membershipLevel(),  // 주문 시점의 등급 (스냅샷)
            command.lines()
        );
        orderRepository.save(order);

        // 비동기 이벤트 발행 — 다른 서비스를 기다리지 않음
        order.pullEvents().forEach(eventPublisher::publishEvent);
        // OrderPlaced 이벤트 → Kafka → 배송 서비스, 재고 서비스가 각자 처리

        return order.getId();  // 즉시 응답 (배송/결제 완료 기다리지 않음)
    }
}

// 배송 서비스 — OrderPlaced 이벤트 수신, 독립 처리
@Component
public class OrderPlacedHandler {
    @KafkaListener(topics = "order.placed")
    @Transactional
    public void handle(OrderPlacedEvent event) {
        Shipment shipment = Shipment.prepareFor(
            event.orderId(),
            event.shippingAddress()
        );
        shipmentRepository.save(shipment);
        // 주문 서비스에 동기 응답 없음 — 독립적으로 처리
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Context와 서비스의 가능한 조합

```
조합 1: 1 Context = 1 서비스 (이상적이지만 항상 최선은 아님)

  주문 Context → order-service
  배송 Context → shipping-service
  결제 Context → payment-service

  적합한 상황:
    팀이 Context별로 분리되어 있음
    각 Context의 배포 주기가 다름
    트래픽 규모가 Context마다 크게 다름

─────────────────────────────────────────────

조합 2: 1 Context = 여러 서비스 (Context 내 규모 분리)

  주문 Context →  order-command-service (쓰기)
                  order-query-service   (읽기, CQRS)

  배송 Context →  shipping-service      (핵심 배송 처리)
                  tracking-service      (위치 추적, 트래픽이 매우 높음)

  적합한 상황:
    하나의 Context 안에서 일부 기능만 트래픽이 극단적으로 높음
    읽기/쓰기 트래픽 패턴이 완전히 다름 (CQRS)
    Context 내 특정 기능이 다른 팀이 담당

  주의:
    Context 내부를 물리적으로 분리해도 도메인 모델은 동일해야 함
    → order-command-service와 order-query-service는 같은 Order 도메인 모델 공유

─────────────────────────────────────────────

조합 3: 여러 Context = 1 서비스 (모놀리스 or 점진적 분리 중간 단계)

  monolith-service →  주문 Context
                      배송 Context
                      결제 Context

  적합한 상황:
    팀이 작아서 서비스 분리 운영 비용이 부담
    아직 Context 경계가 안정화되지 않음 (실험 중)
    마이크로서비스 이전 중간 단계

  핵심: 하나의 서비스 안이라도 패키지로 Context 경계를 표현
    → 나중에 분리할 때 경계가 이미 존재함

─────────────────────────────────────────────

조합 4: 잘못된 분리 — Context 경계를 무시한 서비스 분리 (분산 모놀리스)

  user-service      → 회원의 모든 것 (주문 이력, 배송지, 등급, 포인트...)
  product-service   → 상품의 모든 것 (재고, 가격, 전시, 리뷰...)
  transaction-service → 주문 + 결제 + 배송을 하나로

  → Context 경계 없이 기술적/기능적 기준으로 분리
  → 서비스 간 강한 결합 → 분산 모놀리스
```

### 2. 분산 모놀리스를 판별하는 증상

```
분산 모놀리스 판별 체크리스트:

⚠️ 증상 1: 한 기능을 위해 여러 서비스를 동시에 배포해야 한다
  "주문 상태 표시 변경" → order-service + member-service + shipping-service 배포
  → 서비스 간 계약이 강하게 결합된 것

⚠️ 증상 2: 서비스 A가 다운되면 서비스 B도 사용 불가
  payment-service 장애 → order-service 전체 주문 불가
  → 동기 HTTP 호출 체인으로 장애가 전파됨

⚠️ 증상 3: 각 서비스의 코드가 다른 서비스의 도메인 모델을 알고 있다
  order-service의 코드에 MemberGrade, ProductCategory가 import됨
  → Context 경계가 실제로는 없는 것

⚠️ 증상 4: 분산 트랜잭션이 필요하다
  "주문 생성 + 재고 차감 + 포인트 적립" 을 하나의 트랜잭션으로 처리하고 싶음
  → 트랜잭션 경계가 여러 서비스에 걸쳐 있음 = Context 경계 오류

⚠️ 증상 5: 서비스를 추가/제거할 때 다른 서비스가 영향받는다
  "tracking-service 추가" → shipping-service, order-service 코드 수정 필요
  → 인터페이스(이벤트/API)가 아닌 구현에 결합된 것

건강한 마이크로서비스 체크리스트:
  ✅ 하나의 서비스 배포 → 다른 서비스 배포 불필요
  ✅ 하나의 서비스 장애 → 다른 서비스는 저하(degraded)되지만 동작
  ✅ 서비스 간 통신은 이벤트 또는 명확한 API 계약
  ✅ 각 서비스가 자신의 데이터베이스 소유
  ✅ 새 서비스 추가 시 기존 서비스 코드 수정 불필요
```

### 3. 모놀리스에서 마이크로서비스로 — 올바른 순서

```
Strangler Fig 패턴으로 점진적 분리:

                    시간 →

Phase 1: 모놀리스 (빠른 개발)
  ┌─────────────────────────────┐
  │        Monolith             │
  │  order / shipping / payment │  ← 패키지로 논리적 경계만
  └─────────────────────────────┘

Phase 2: 논리적 분리 강화 (선행 필수)
  ┌─────────────────────────────┐
  │        Monolith             │
  │  ┌──────┐ ┌──────────┐     │
  │  │order │ │shipping  │     │  ← 패키지 간 직접 의존 금지 (ArchUnit)
  │  └──────┘ └──────────┘     │  ← 이벤트로 통신 교체
  │  ┌─────────────────────┐   │
  │  │      payment        │   │
  │  └─────────────────────┘   │
  └─────────────────────────────┘

Phase 3: 가장 독립적인 Context 먼저 분리
  ┌────────────┐    Kafka     ┌───────────────────┐
  │ Monolith   │ ──이벤트──→ │ shipping-service  │  ← 먼저 분리
  │ (order +   │             │ (독립 배포)        │
  │  payment)  │             └───────────────────┘
  └────────────┘

Phase 4: 점진적 분리 완성
  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐
  │order-service │  │shipping-svc  │  │payment-svc   │
  │              │  │              │  │              │
  └──────────────┘  └──────────────┘  └──────────────┘

분리 순서 결정 기준:
  1. 가장 독립적인 Context (다른 Context 의존이 적은 것)
  2. 배포 주기가 가장 다른 Context (자주 바뀌는 것)
  3. 트래픽 패턴이 가장 다른 Context (스케일링 필요한 것)
  4. 팀이 독립적으로 소유하는 Context
```

### 4. 지금 당장 마이크로서비스가 아니어도 DDD가 가치 있는 이유

```
"DDD는 마이크로서비스를 위한 것이다" → 오해

DDD의 가치는 모놀리스에서도 동일:
  ① Context 경계 = 모듈 경계
     모놀리스 안의 패키지 구조가 Context 경계를 따름
     → 나중에 마이크로서비스 분리 시 경계가 이미 존재
     → "어디서 자르면 되는가?" 명확

  ② Aggregate = 트랜잭션 단위 명확화
     DB가 하나여도 Aggregate 경계를 지키면
     → 나중에 DB를 Context별로 분리하기 쉬움

  ③ Domain Event = 결합도 낮춤
     모놀리스 안에서도 이벤트로 통신하면
     → 나중에 Kafka를 끼우면 자동으로 비동기 통신

  ④ Ubiquitous Language = 팀 내 소통 비용 절감
     서비스 분리와 무관하게 가치 있음

현실적 충고:
  마이크로서비스 = DDD의 결과물 (충분한 도메인 이해 후)
  마이크로서비스 = DDD의 시작점이 아님

  "마이크로서비스 도입 = DDD 적용" 이라는 착각이
  분산 모놀리스의 가장 큰 원인
```

---

## 💻 실전 코드

### Context 경계를 유지하는 서비스 분리 — Kafka 이벤트 기반

```java
// 주문 서비스 — Context 경계를 유지하며 이벤트 발행
@Service
@Transactional
public class OrderApplicationService {

    private final OrderRepository orderRepository;
    private final KafkaOrderEventPublisher eventPublisher;

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.place(command.customerId(), command.lines());
        orderRepository.save(order);

        // Outbox Pattern: DB 트랜잭션 내에서 이벤트 저장 (메시지 유실 방지)
        List<DomainEvent> events = order.pullEvents();
        events.forEach(outboxRepository::save);  // 같은 트랜잭션

        return order.getId();
    }
}

// Outbox → Kafka 릴레이 (별도 프로세스)
@Scheduled(fixedDelay = 1000)
public void relayOutboxEvents() {
    List<OutboxEvent> pending = outboxRepository.findPending();
    pending.forEach(event -> {
        kafkaTemplate.send("order.events", event.payload());
        outboxRepository.markPublished(event.id());
    });
}

// 배송 서비스 — 이벤트 수신, Context 내부만 처리
@KafkaListener(topics = "order.events")
public void handle(String payload) {
    OrderPlacedEvent event = deserialize(payload);

    Shipment shipment = Shipment.prepareFor(
        new OrderId(event.orderId()),
        ShippingAddress.of(event.shippingAddress())
    );
    shipmentRepository.save(shipment);
    // 주문 서비스에 응답하지 않음 — 독립 처리
}
```

### 하나의 서비스에 여러 Context 공존 — 패키지로 경계 표현

```java
// 모놀리스 내 패키지 구조 (여러 Context 공존)
/*
  com.example/
    ├── order/                    ← 주문 Context
    │   ├── domain/
    │   │   ├── Order.java
    │   │   └── OrderRepository.java
    │   └── application/
    │       └── OrderApplicationService.java
    │
    ├── shipping/                 ← 배송 Context
    │   ├── domain/
    │   │   ├── Shipment.java
    │   │   └── ShipmentRepository.java
    │   └── application/
    │       └── ShipmentApplicationService.java
    │
    └── shared/                   ← Shared Kernel (최소화)
        ├── Money.java
        └── OrderId.java
*/

// ArchUnit: 패키지 간 의존 금지로 경계 강제
@AnalyzeClasses(packages = "com.example")
class ContextBoundaryArchTest {

    @ArchTest
    static ArchRule orderShouldNotDependOnShipping =
        noClasses()
            .that().resideInAPackage("..order.domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..shipping.domain..")
            .because("주문 Context의 도메인이 배송 Context 도메인을 직접 참조하면 안 됩니다");

    @ArchTest
    static ArchRule onlySharedKernelCanBeCrossReferenced =
        noClasses()
            .that().resideInAPackage("..order.domain..")
            .and().doNotHaveSimpleName("OrderId")
            .should().dependOnClassesThat()
            .resideInAPackage("..shipping.domain..")
            .because("Context 간 참조는 Shared Kernel의 ID 클래스만 허용");
}
```

---

## 📊 설계 비교

```
모놀리스 vs 분산 모놀리스 vs 올바른 마이크로서비스:

                모놀리스           분산 모놀리스         올바른 마이크로서비스
                (모듈화됨)         (Context 없이)        (Context 기반)
────────────┼────────────────┼──────────────────┼───────────────────────
배포         │ 전체 배포        │ 서비스별 배포      │ Context별 독립 배포
            │ (전략적 방법)    │ (그러나 항상 동시) │ (진정한 독립)
────────────┼────────────────┼──────────────────┼───────────────────────
장애 격리   │ 단일 장애점      │ 장애 전파됨        │ Circuit Breaker로 격리
            │                │ (= 모놀리스와 같음) │
────────────┼────────────────┼──────────────────┼───────────────────────
서비스 간   │ 함수 호출        │ 동기 HTTP 체인     │ 이벤트 기반 비동기
통신         │ (빠름, 안정적)  │ (느리고 취약)      │ (느슨한 결합)
────────────┼────────────────┼──────────────────┼───────────────────────
데이터      │ 공유 DB          │ 서비스별 DB        │ Context별 DB
────────────┼────────────────┼──────────────────┼───────────────────────
복잡도      │ 낮음~중간        │ 매우 높음          │ 높음 (가치 있는 복잡도)
────────────┼────────────────┼──────────────────┼───────────────────────
팀 규모     │ 소~중            │ 중~대 (역설적)     │ 중~대 (Context별 팀)
────────────┼────────────────┼──────────────────┼───────────────────────
DDD 없이   │ 가능             │ → 분산 모놀리스    │ 불가능 (항상 실패)
────────────┼────────────────┼──────────────────┼───────────────────────
권장 상황   │ 5~20명 팀       │ 권장하지 않음      │ 20명+ 팀,
            │ 성장 중인 스타트업│                   │ Context가 명확한 경우
```

---

## ⚖️ 트레이드오프

```
마이크로서비스 전환의 현실적 비용:

① 운영 복잡도
   서비스 수 × 운영 비용
   Kubernetes, 서비스 메시, 분산 트레이싱, Centralized Logging 필요
   DevOps 역량 필수

② 분산 트랜잭션 문제
   모놀리스: 하나의 DB 트랜잭션으로 모든 것 처리
   마이크로서비스: Saga 패턴 + 보상 트랜잭션 필요
   → 복잡도 급증

③ 로컬 개발 환경
   모놀리스: 하나의 JVM 실행
   마이크로서비스: 4개 서비스 + Kafka + 4개 DB를 로컬에서 실행
   → Docker Compose 필요, 개발자 경험 저하

마이크로서비스가 적합하지 않은 경우:
  ① 팀이 작음 (< 10명): 운영 비용이 이익을 초과
  ② 도메인이 불확실: Context 경계가 자주 바뀌면 서비스 재조정 비용
  ③ 트래픽이 낮음: 스케일링이 필요하지 않으면 이점 없음

현실적 조언:
  "마이크로서비스부터" → 분산 모놀리스 위험 높음
  "모놀리스 + DDD(패키지 경계)" → Context가 안정되면 분리
  → 이것이 Martin Fowler가 말하는 "Monolith First" 전략

  필요한 시점:
    독립 배포가 실제로 필요할 때
    특정 Context의 스케일링이 다른 것과 크게 다를 때
    팀이 Context별로 분리될 때
```

---

## 📌 핵심 정리

```
Context ≠ 서비스, 핵심 요약:

Context와 서비스의 관계:
  1 Context = 1 서비스: 이상적, 팀 규모가 뒷받침될 때
  1 Context = N 서비스: CQRS, 트래픽 분산이 필요할 때
  N Context = 1 서비스: 소규모 팀, 분리 이전 단계
  Context 없이 서비스 분리: 분산 모놀리스 → 피할 것

분산 모놀리스의 특징:
  서비스 간 동기 HTTP 체인
  장애 전파 (한 서비스 다운 → 전체 영향)
  서비스 경계를 넘는 도메인 모델 의존
  여러 서비스를 동시에 배포해야 함

올바른 마이크로서비스 분리 순서:
  1. Context 경계 정의 (이벤트 스토밍, Context Map)
  2. 모놀리스 내 패키지로 논리적 경계 표현
  3. 이벤트 기반 통신으로 교체
  4. 가장 독립적인 Context부터 물리적 분리
  5. 점진적으로 확장

DDD가 모놀리스에서도 가치 있는 이유:
  Context 경계 = 미래 서비스 경계 (분리 준비 완료)
  Aggregate = 트랜잭션 단위 명확화
  Domain Event = 결합도 감소 (서비스 분리 준비)
  Ubiquitous Language = 팀 소통 비용 절감
```

---

## 🤔 생각해볼 문제

**Q1.** 현재 모놀리스 서비스가 있고 팀이 20명으로 늘어났다. 마이크로서비스 전환을 고려 중이다. 어떤 순서로 진행하고, 어느 Context를 가장 먼저 분리해야 하는가?

<details>
<summary>해설 보기</summary>

**분리 순서와 기준:**

```
Step 1: 현재 상태 진단 (1~2주)
  현재 모놀리스의 Context 경계가 명확한가?
  패키지 구조가 Context를 반영하는가?
  → 명확하지 않으면 패키지 리팩터링 먼저

Step 2: 분산 모놀리스 위험 제거 (4~8주)
  서비스 간 동기 HTTP 호출을 이벤트로 교체
  ArchUnit으로 Context 간 직접 의존 금지
  → 물리적 분리 없이 논리적 분리 먼저 강화

Step 3: 첫 번째 분리 대상 선정
  우선순위 기준:
  ① 배포 주기가 가장 다른 Context
     (다른 것과 무관하게 자주 배포해야 하는 것)
  ② 다른 Context의 의존이 가장 적은 것 (독립적)
  ③ 트래픽 패턴이 극단적으로 다른 것 (스케일링 필요)
  ④ 팀이 독립적으로 소유하는 것

  일반적으로 적합한 첫 번째 분리 대상:
  ✅ 알림/이메일 서비스 (Generic, 다른 것과 분리됨)
  ✅ 파일/이미지 서비스 (독립적, 트래픽 다름)
  ❌ 주문 서비스 (Core, 다른 것과 결합 많음)
  ❌ 회원 서비스 (모든 서비스가 의존)

Step 4: 학습 후 확장
  첫 분리에서 운영 노하우 축적 (K8s, 모니터링, 배포)
  → 다음 분리 대상 결정
```

</details>

---

**Q2.** "주문 Context를 order-command-service와 order-query-service로 분리하는 CQRS"는 Context 경계 위반인가?

<details>
<summary>해설 보기</summary>

**Context 경계 위반이 아닙니다. 이것은 Context 내 기술적 분리입니다.**

```
올바른 이해:
  Bounded Context = 비즈니스/도메인의 경계
  서비스 분리 = 기술/운영의 경계

  order-command-service와 order-query-service는
  둘 다 "주문 Context"에 속함
  → 같은 Ubiquitous Language를 씀
  → 같은 도메인 모델(Order, OrderLine 등)을 공유

CQRS 분리가 Context 경계를 침범하지 않는 방법:
  ① 공유 도메인 모델 라이브러리
     order-domain.jar (공유 라이브러리)
     ├── order-command-service (이 라이브러리 사용)
     └── order-query-service   (이 라이브러리 사용)

  ② 이벤트로 동기화
     order-command-service가 OrderPlaced 이벤트 발행
     order-query-service가 수신 → 읽기 모델 업데이트

Context 경계 위반이 되는 경우:
  order-query-service가 shipping Context의 데이터를 직접 포함
  → "주문 + 배송 통합 조회 모델"이 order-query-service에 있으면
  → 이것은 shipping Context의 언어가 order Context에 침투
  → 별도 "주문-배송 집계 서비스" 또는 BFF에서 처리해야 함
```

</details>

---

**Q3.** 마이크로서비스로 분리한 후 "서비스 A의 데이터와 서비스 B의 데이터를 함께 조회하는 화면"을 어떻게 구현하는가? Context 경계를 위반하지 않으면서.

<details>
<summary>해설 보기</summary>

**세 가지 방법이 있으며, 상황에 따라 선택합니다.**

**방법 1: BFF (Backend for Frontend) — API Composition**
```java
// BFF 서버에서 두 서비스 데이터를 합침
@GetMapping("/order-dashboard/{orderId}")
public OrderDashboardResponse getDashboard(@PathVariable Long orderId) {
    CompletableFuture<OrderDto> orderFuture = 
        CompletableFuture.supplyAsync(() -> orderClient.getOrder(orderId));
    CompletableFuture<ShipmentDto> shipmentFuture = 
        CompletableFuture.supplyAsync(() -> shippingClient.getShipment(orderId));

    return CompletableFuture.allOf(orderFuture, shipmentFuture)
        .thenApply(v -> OrderDashboardResponse.merge(orderFuture.join(), shipmentFuture.join()))
        .join();
}
// 장점: 단순, 실시간 데이터
// 단점: 두 서비스가 동시에 응답해야 함, 지연 합산
```

**방법 2: 이벤트 기반 읽기 모델 (CQRS 집계 뷰)**
```java
// 별도 읽기 전용 서비스 (또는 BFF 내 읽기 DB)
// 이벤트를 구독해서 집계 뷰를 미리 만들어 놓음
@EventListener
public void on(OrderPlaced event) {
    orderShippingView.save(new OrderShippingView(event.orderId(), event.customerId(), null));
}
@EventListener  
public void on(ShipmentDispatched event) {
    orderShippingView.update(event.orderId(), event.trackingNumber());
}

// 조회는 집계 뷰에서 단일 쿼리
@GetMapping("/order-dashboard/{orderId}")
public OrderDashboardResponse getDashboard(@PathVariable Long orderId) {
    return orderShippingView.findByOrderId(orderId);  // 단일 쿼리, 빠름
}
// 장점: 빠른 조회, 서비스 장애 무관
// 단점: 이벤트 처리 지연으로 약간의 데이터 지연 (Eventually Consistent)
```

**방법 3: GraphQL Federation**
각 서비스가 자신의 타입을 정의하고, Federation Gateway가 클라이언트 쿼리에 따라 자동으로 조합. 서비스 간 직접 의존 없이 클라이언트 주도로 데이터 조합.

**선택 기준:**
- 실시간성 중요 + 간단한 조합 → BFF API Composition
- 성능 중요 + 복잡한 집계 → 이벤트 기반 읽기 모델
- 다양한 클라이언트 + 유연한 조합 → GraphQL Federation

</details>

---

<div align="center">

**[⬅️ 이전: 이벤트 스토밍](./06-event-storming.md)** | **[홈으로 🏠](../README.md)** | **[Chapter 3로 이동: Tactical Design — Entity vs Value Object ➡️](../tactical-design/01-entity-vs-value-object.md)**

</div>
