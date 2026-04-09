# Context 간 데이터 동기화 — Eventual Consistency 허용 설계

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Event Sourcing 없이 Context 간 읽기 모델을 어떻게 동기화하는가?
- Eventual Consistency를 허용하는 설계 결정의 근거는 무엇인가?
- 정합성 검증 전략과 데이터 불일치를 어떻게 감지하는가?
- 읽기 모델이 지연됐을 때 UX를 어떻게 설계해야 하는가?
- Context 간 데이터 복제와 API 호출 중 어느 것을 선택해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"주문 목록 화면에 배송 현황도 함께 보여주세요"라는 요구사항이 오면 어떻게 하는가? 주문 Context와 배송 Context가 분리됐다면, 두 Context의 데이터를 합쳐야 한다. 매번 두 서비스를 동기 호출하면 응답이 느리고 하나가 다운되면 화면 전체가 실패한다.

이벤트로 읽기 모델을 동기화하면 단일 쿼리로 빠른 응답이 가능하지만, 최신 데이터와 수 초의 차이가 날 수 있다. 이 트레이드오프를 비즈니스와 합의하고 올바르게 설계하는 것이 이 문서의 핵심이다.

---

## 😱 흔한 실수 (Before — 화면마다 다른 서비스 동기 호출)

```java
// 안티패턴: 화면 조회마다 여러 서비스 동기 호출
@GetMapping("/my-orders")
public List<OrderListItemResponse> getMyOrders(@AuthenticationPrincipal Long customerId) {

    // 1. 주문 서비스에서 주문 목록
    List<OrderDto> orders = orderClient.getOrdersByCustomer(customerId);  // ~100ms

    // 2. 각 주문의 배송 정보를 개별 조회 — N+1!
    List<OrderListItemResponse> result = new ArrayList<>();
    for (OrderDto order : orders) {
        ShipmentDto shipment = shippingClient.getShipmentByOrder(order.id());  // ~80ms × N건
        result.add(OrderListItemResponse.merge(order, shipment));
    }

    // 주문 20건 → 20번의 배송 조회 → 100ms + 80ms × 20 = 1700ms
    return result;
}
```

```
문제:
  N+1 쿼리: 주문 N건 → 배송 N번 호출
  배송 서비스 장애 → 주문 목록 전체 실패
  배송 서비스 응답 지연 → 주문 목록 응답 지연
  
  "배송 서비스 장애인데 주문 목록도 안 보여요" — 고객 불만
```

---

## ✨ 올바른 접근 (After — 이벤트 기반 읽기 모델 동기화)

```java
// 읽기 모델: 주문 + 배송 통합 뷰
@Entity
@Table(name = "order_summary_view")  // 읽기 전용 통합 테이블
public class OrderSummaryView {
    @Id private Long orderId;
    private Long customerId;
    private BigDecimal totalAmount;
    private String orderStatus;
    
    // 배송 Context에서 동기화된 데이터
    private String trackingNumber;       // nullable (배송 전은 null)
    private String shipmentStatus;       // null, PREPARING, SHIPPED, DELIVERED
    private LocalDate estimatedDelivery;
    
    private LocalDateTime lastUpdatedAt;  // 동기화 시각
}

// 이벤트 핸들러: 주문 이벤트 수신 → 읽기 모델 갱신
@Component
public class OrderSummaryViewUpdater {

    @EventListener
    @Transactional
    public void on(OrderPlaced event) {
        OrderSummaryView view = new OrderSummaryView();
        view.setOrderId(event.orderId().value());
        view.setCustomerId(event.customerId().value());
        view.setTotalAmount(event.totalAmount().amount());
        view.setOrderStatus("PENDING");
        view.setLastUpdatedAt(LocalDateTime.now());
        orderSummaryViewRepository.save(view);
    }

    @KafkaListener(topics = "shipping.events")  // 배송 서비스에서 발행
    @Transactional
    public void on(ShipmentDispatched event) {
        orderSummaryViewRepository.findByOrderId(event.orderId().value())
            .ifPresent(view -> {
                view.setTrackingNumber(event.trackingNumber().value());
                view.setShipmentStatus("SHIPPED");
                view.setEstimatedDelivery(event.estimatedDelivery());
                view.setLastUpdatedAt(LocalDateTime.now());
                orderSummaryViewRepository.save(view);
            });
    }
}

// 조회 API: 단일 읽기 모델에서 빠른 응답
@GetMapping("/my-orders")
public List<OrderListItemResponse> getMyOrders(@AuthenticationPrincipal Long customerId) {
    // 단일 쿼리! 배송 서비스 호출 없음
    List<OrderSummaryView> views = orderSummaryViewRepository.findByCustomerId(customerId);
    return views.stream().map(OrderListItemResponse::from).collect(toList());
    // ~10ms, 배송 서비스 장애와 무관
}
```

---

## 🔬 내부 동작 원리

### 1. 읽기 모델 동기화 패턴

```
패턴 1: 이벤트 기반 동기화 (권장)

  주문 서비스 → OrderPlaced 이벤트 → 읽기 모델 업데이트
  배송 서비스 → ShipmentDispatched 이벤트 → 읽기 모델 업데이트
  
  장점:
    조회 시 다른 서비스 호출 없음 → 빠른 응답
    배송 서비스 장애와 무관 (읽기 모델은 독립)
  
  단점:
    이벤트 처리 지연 → 수 초간 오래된 데이터 표시
    읽기 모델과 원본 데이터의 일시적 불일치

패턴 2: API Composition (실시간 필요 시)

  조회 시 여러 서비스 병렬 호출 → 조합
  
  CompletableFuture<List<OrderDto>> ordersFuture =
      CompletableFuture.supplyAsync(() -> orderClient.getOrders(customerId));
  CompletableFuture<Map<Long, ShipmentDto>> shipmentsFuture =
      CompletableFuture.supplyAsync(() -> shippingClient.getShipments(orderIds));
  
  // 병렬 실행 → 총 응답 시간 = max(주문, 배송) (직렬 아님)
  
  장점: 항상 최신 데이터
  단점: 다른 서비스 의존, 하나 장애 시 처리 필요

패턴 3: 하이브리드 (캐시 + 실시간)

  기본: 읽기 모델에서 빠른 응답 (캐시된 데이터)
  상세: 실시간 조회 (개별 상품 클릭 시)
  
  목록 화면: 읽기 모델 (약간 오래될 수 있음)
  상세 화면: 실시간 API 호출 (항상 최신)
```

### 2. Eventual Consistency 허용 결정 근거

```
비즈니스 질문: "이 데이터가 몇 초 지연돼도 괜찮은가?"

허용 가능한 경우 (Eventual Consistency OK):
  주문 목록의 배송 현황: "배송 출발했어요"가 3초 뒤에 보여도 됨
  포인트 잔액: "적립 후 5초 뒤 반영"이 허용됨
  상품 리뷰 수: "리뷰 등록 직후 1~2초 지연" 허용
  재고 수량 표시: "10개 있는데 9개로 보여도 됨" (실제 주문 시 검증)

허용 불가한 경우 (강한 일관성 필요):
  결제 금액: 결제 후 즉시 올바른 금액 표시 필요
  재고 차감: 결제 완료 후 재고 즉시 차감 (중복 판매 방지)
  로그인 상태: 로그아웃 즉시 반영 필요

결정 기준:
  "이 데이터가 틀렸을 때 고객이 잘못된 결정을 내리는가?"
  → YES: 강한 일관성 필요
  → NO:  Eventual Consistency 허용

  "이 데이터의 불일치가 비즈니스 손실(환불, CS 비용)을 유발하는가?"
  → YES: 강한 일관성 투자 가치
  → NO:  UX로 해결 가능
```

### 3. 정합성 검증과 불일치 감지

```java
// 정기 검증 배치: 읽기 모델과 원본 데이터 비교
@Scheduled(cron = "0 */10 * * * *")  // 10분마다
public void reconcileOrderSummaryView() {
    LocalDateTime since = LocalDateTime.now().minusMinutes(15);  // 최근 15분

    // 최근 업데이트된 주문 조회
    List<Order> recentOrders = orderRepository.findUpdatedSince(since);

    recentOrders.forEach(order -> {
        OrderSummaryView view = orderSummaryViewRepository
            .findByOrderId(order.id().value())
            .orElse(null);

        if (view == null) {
            // 읽기 모델 누락 → 재생성
            log.warn("읽기 모델 누락 감지: orderId={}", order.id());
            orderSummaryViewUpdater.rebuild(order);
        } else if (!view.orderStatus().equals(order.status().name())) {
            // 상태 불일치 → 읽기 모델 갱신
            log.warn("읽기 모델 불일치 감지: orderId={}, view={}, origin={}",
                order.id(), view.orderStatus(), order.status());
            orderSummaryViewUpdater.refresh(order);
            metricsService.increment("order.view.inconsistency");
        }
    });
}

// 이벤트 재발행 (읽기 모델 전체 재구성)
@PostMapping("/admin/rebuild-order-view")
public void rebuildOrderSummaryView() {
    log.info("주문 읽기 모델 전체 재구성 시작");
    orderSummaryViewRepository.deleteAll();

    orderRepository.findAll().forEach(order -> {
        orderSummaryViewUpdater.rebuild(order);
    });
    log.info("주문 읽기 모델 재구성 완료");
}
```

### 4. UX로 Eventual Consistency 처리

```
읽기 모델이 지연됐을 때 UX 설계:

전략 1: 낙관적 UI 업데이트 (Optimistic Update)
  주문 완료 직후 프론트엔드가 즉시 UI 업데이트
  (서버 응답을 기다리지 않고 클라이언트에서 상태 업데이트)
  → 서버 동기화가 완료되면 실제 데이터로 교체
  
  Order placed → 즉시 "주문 완료" 화면 표시
  (백그라운드에서 읽기 모델 업데이트 중)

전략 2: 설명적 메시지
  "포인트는 잠시 후 적립됩니다"
  "배송 현황은 최대 1분 후 업데이트됩니다"
  → 사용자가 지연을 이해하고 수용

전략 3: 폴링 또는 WebSocket
  주문 완료 후 주문 상태를 주기적으로 조회 (1초마다)
  배송 상태 변경 시 WebSocket으로 실시간 푸시
  → "배송이 출발했습니다!" 즉시 알림

전략 4: 마지막 업데이트 시간 표시
  "배송 현황: 출발함 (5분 전 기준)"
  → 사용자가 데이터 신선도를 인지
```

---

## 💻 실전 코드

### 이벤트 기반 읽기 모델 구성

```java
// 읽기 전용 Repository (JPA 없이 단순 쿼리)
public interface OrderSummaryViewRepository extends JpaRepository<OrderSummaryView, Long> {

    List<OrderSummaryView> findByCustomerIdOrderByCreatedAtDesc(Long customerId);

    @Query("""
        SELECT v FROM OrderSummaryView v
        WHERE v.customerId = :customerId
          AND v.orderStatus IN ('PENDING', 'PAID', 'SHIPPED')
        ORDER BY v.createdAt DESC
        """)
    List<OrderSummaryView> findActiveOrders(@Param("customerId") Long customerId);
}

// 읽기 모델 재구성 (이벤트 재발행 또는 원본에서)
@Component
public class OrderSummaryViewRebuilder {

    @Transactional
    public void rebuild(Order order) {
        OrderSummaryView view = orderSummaryViewRepository
            .findByOrderId(order.id().value())
            .orElse(new OrderSummaryView());

        view.setOrderId(order.id().value());
        view.setCustomerId(order.customerId().value());
        view.setTotalAmount(order.totalAmount().amount());
        view.setOrderStatus(order.status().name());
        view.setLastUpdatedAt(LocalDateTime.now());

        // 배송 정보 최신화 (있으면)
        shipmentRepository.findByOrderId(order.id()).ifPresent(shipment -> {
            view.setTrackingNumber(shipment.trackingNumber() != null
                ? shipment.trackingNumber().value() : null);
            view.setShipmentStatus(shipment.status().name());
        });

        orderSummaryViewRepository.save(view);
    }
}
```

---

## 📊 설계 비교

```
API Composition vs 이벤트 기반 읽기 모델:

                API Composition          이벤트 기반 읽기 모델
────────────┼──────────────────────┼──────────────────────────
데이터 신선도 │ 항상 최신              │ 이벤트 처리 지연 (수 초)
────────────┼──────────────────────┼──────────────────────────
응답 속도    │ 여러 서비스 합산 시간   │ 단일 쿼리, 빠름
────────────┼──────────────────────┼──────────────────────────
가용성       │ 하나 장애 → 영향       │ 읽기는 독립, 장애 격리
────────────┼──────────────────────┼──────────────────────────
복잡도       │ 낮음 (병렬 호출)       │ 이벤트 핸들러 + 재구성 로직
────────────┼──────────────────────┼──────────────────────────
데이터 일관성 │ 강한 일관성            │ 결과적 일관성
────────────┼──────────────────────┼──────────────────────────
적합한 경우  │ 실시간 필요, 트래픽 낮 │ 높은 조회 트래픽, 복잡 집계
```

---

## ⚖️ 트레이드오프

```
이벤트 기반 읽기 모델의 어려움:
  ① 이벤트 누락 시 읽기 모델 불일치
     해결: 정기 검증 배치 + 재구성 기능

  ② 이벤트 순서 보장
     "ShipmentDispatched → ShipmentDelivered" 순서가 뒤바뀌면?
     읽기 모델이 이상한 상태가 됨
     해결: 이벤트에 타임스탬프 포함 → 오래된 이벤트 무시

  ③ 스키마 마이그레이션
     읽기 모델 스키마 변경 시 기존 데이터 전체 재구성 필요
     → 재구성 스크립트 준비

현실적 전략:
  목록 화면 → 이벤트 기반 읽기 모델
  상세 화면 → 실시간 API 호출
  금융 데이터 → 항상 실시간 (강한 일관성)
```

---

## 📌 핵심 정리

```
Context 간 데이터 동기화 핵심:

읽기 모델 동기화:
  이벤트 수신 → 읽기 전용 테이블 업데이트
  조회 시 단일 쿼리 → 빠른 응답, 장애 격리

Eventual Consistency 허용 기준:
  "데이터가 틀렸을 때 잘못된 결정이 발생하는가?"
  → NO: Eventual Consistency 허용
  금융/재고/결제: 강한 일관성 필요

정합성 검증:
  정기 배치로 원본과 읽기 모델 비교
  불일치 감지 시 재구성 + 알림
  이벤트 재발행으로 읽기 모델 전체 재구성 가능

UX 설계:
  낙관적 업데이트 (프론트엔드 즉시 반영)
  설명적 메시지 ("잠시 후 반영됩니다")
  상세 화면은 실시간 API 조회
```

---

## 🤔 생각해볼 문제

**Q1.** 읽기 모델에서 조회한 재고 수량이 실제보다 많게 표시됐다. 고객이 이를 보고 주문했는데, 실제 재고 처리 시 부족으로 주문이 실패했다. 이 상황을 설계로 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

**재고 수량은 읽기 모델에서 표시만 하고, 실제 처리 시 원본에서 확인하는 이중 체크 구조가 필요합니다.**

```java
// 표시용: 읽기 모델 (빠른 응답, 약간 오래될 수 있음)
@GetMapping("/products/{id}")
public ProductDetailResponse getProduct(@PathVariable Long productId) {
    ProductSummaryView view = productSummaryViewRepository.findById(productId).orElseThrow();
    // 읽기 모델의 재고 수량 표시 (실시간 아닐 수 있음)
    // → "재고: 약 5개" 또는 "품절 임박" 표현으로 정확하지 않음을 암시
    return ProductDetailResponse.from(view);
}

// 주문 처리: 원본에서 실시간 재고 확인 (강한 일관성)
@Transactional
public OrderId placeOrder(PlaceOrderCommand command) {
    // 원본 Inventory Aggregate에서 실시간 확인
    command.lines().forEach(line -> {
        Inventory inv = inventoryRepository.findByProductId(line.productId()).orElseThrow();
        if (!inv.isAvailable(line.quantity())) {
            throw new OutOfStockException(line.productId());
        }
    });
    // 재고가 충분하면 주문 진행
}
```

UX 전략: 상품 목록/상세 화면의 재고는 "약 N개 이상" 또는 "재고 있음/없음" 수준으로 표현하고, 정확한 수량에 의존하지 않도록 합니다.

</details>

---

**Q2.** 읽기 모델이 생성되는 이벤트 핸들러가 실패하면 읽기 모델이 누락될 수 있다. 이를 탐지하고 보정하는 방법은?

<details>
<summary>해설 보기</summary>

**정기 검증 배치와 이벤트 재발행으로 자동 보정합니다.**

```java
// 탐지: 원본과 읽기 모델 카운트 비교
@Scheduled(cron = "0 0 * * * *")  // 매 시간
public void detectMissingViews() {
    long orderCount = orderRepository.count();
    long viewCount = orderSummaryViewRepository.count();

    if (orderCount != viewCount) {
        log.warn("읽기 모델 누락 감지: 주문={}, 뷰={}", orderCount, viewCount);
        alertService.notifyViewInconsistency(orderCount, viewCount);

        // 자동 보정: 누락된 뷰 재생성
        Set<Long> orderIds = orderRepository.findAllIds();
        Set<Long> viewIds = orderSummaryViewRepository.findAllOrderIds();
        orderIds.removeAll(viewIds);  // 누락된 ID

        orderIds.forEach(orderId -> {
            Order order = orderRepository.findById(new OrderId(orderId)).orElseThrow();
            orderSummaryViewRebuilder.rebuild(order);
        });
    }
}
```

</details>

---

**Q3.** 이벤트 기반 읽기 모델에서 "Order가 취소됐는데 배송 현황이 여전히 배송 중으로 표시된다"는 버그 리포트가 들어왔다. 원인과 해결 방법은?

<details>
<summary>해설 보기</summary>

**이벤트 처리 순서 문제 또는 이벤트 누락이 원인입니다.**

```java
// 취소 핸들러: OrderCancelled 이벤트 → 읽기 모델 업데이트
@EventListener
@Transactional
public void on(OrderCancelled event) {
    orderSummaryViewRepository.findByOrderId(event.orderId().value())
        .ifPresent(view -> {
            view.setOrderStatus("CANCELLED");
            view.setShipmentStatus(null);         // 배송 현황 초기화
            view.setTrackingNumber(null);         // 송장번호 초기화
            view.setLastUpdatedAt(LocalDateTime.now());
            orderSummaryViewRepository.save(view);
        });
}

// 순서 보장: 타임스탬프 기반
@EventListener
@Transactional
public void on(ShipmentDispatched event) {
    orderSummaryViewRepository.findByOrderId(event.orderId().value())
        .ifPresent(view -> {
            // 주문이 이미 취소됐으면 배송 이벤트 무시
            if ("CANCELLED".equals(view.orderStatus())) {
                log.info("취소된 주문의 배송 이벤트 무시: orderId={}", event.orderId());
                return;
            }
            view.setShipmentStatus("SHIPPED");
            view.setLastUpdatedAt(LocalDateTime.now());
            orderSummaryViewRepository.save(view);
        });
}
```

근본 원인이 이벤트 순서라면: `OrderCancelled`가 `ShipmentDispatched`보다 늦게 처리된 것. 읽기 모델에서 상태 우선순위를 설정하거나 이벤트 타임스탬프로 최신 상태를 결정합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Saga Pattern](./04-saga-pattern.md)** | **[홈으로 🏠](../README.md)** | **[다음: ACL 실전 구현 ➡️](./06-acl-implementation.md)**

</div>
