# 읽기 모델 분리 — 조회 성능을 위한 별도 모델

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 주문 목록 조회를 위해 왜 쓰기 모델과 분리된 읽기 모델이 필요한가?
- Domain Event로 읽기 모델을 동기화하는 방법은?
- CQRS의 Command와 Query를 어떻게 코드 수준에서 분리하는가?
- 읽기 모델과 원본 데이터의 불일치를 어떻게 감지하고 복구하는가?
- 페이지네이션, 검색, 정렬을 읽기 모델에서 어떻게 지원하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

전자상거래에서 "내 주문 목록" 화면은 주문 정보뿐 아니라 배송 현황, 상품 이미지, 리뷰 여부 등 여러 Context의 데이터를 조합해야 한다. 매번 여러 Context에 동기 호출하면 응답이 느리고 하나가 다운되면 전체가 실패한다. 읽기 전용 통합 모델을 이벤트로 동기화하면 단일 쿼리로 빠른 응답이 가능하다.

---

## 😱 흔한 실수 (Before — 화면마다 다중 Context 동기 조합)

```java
// 주문 목록 화면: 여러 Context 동기 호출
@GetMapping("/my-orders")
public List<OrderListResponse> getMyOrders(@AuthenticationPrincipal Long customerId) {
    List<Order> orders = orderRepository.findByCustomerId(customerId);  // 주문 Context

    return orders.stream().map(order -> {
        // 각 주문마다 다른 Context 호출 — N+1 패턴!
        ShipmentStatus shipStatus = shipmentClient.getStatus(order.id());   // 배송 Context
        ProductInfo product = productClient.getInfo(order.mainProductId()); // 상품 Context
        boolean hasReview = reviewClient.hasReview(customerId, order.id()); // 리뷰 Context

        return OrderListResponse.of(order, shipStatus, product, hasReview);
    }).collect(toList());

    // 주문 20건: 20×3 = 60번의 외부 호출
    // 배송 서비스 장애 → 주문 목록 전체 실패
    // 응답 시간: 각 Context 합산 → 수 초
}
```

---

## ✨ 올바른 접근 (After — 읽기 모델 동기화)

```java
// 읽기 전용 통합 뷰 테이블
@Entity
@Table(name = "order_list_view")
public class OrderListView {
    @Id
    private String orderId;

    private String customerId;
    private String status;

    // 주문 Context 데이터
    private BigDecimal totalAmount;
    private LocalDateTime orderedAt;

    // 상품 Context 데이터 (스냅샷)
    private String mainProductName;
    private String mainProductThumbnailUrl;
    private int totalItemCount;

    // 배송 Context 데이터 (동기화됨)
    private String shipmentStatus;          // null: 배송 전
    private String trackingNumber;          // null: 배송 전
    private LocalDate estimatedDelivery;

    // 리뷰 Context 데이터
    private boolean hasReview;

    private LocalDateTime lastSyncedAt;
}

// 조회 API: 단일 쿼리
@GetMapping("/my-orders")
public Page<OrderListResponse> getMyOrders(
    @AuthenticationPrincipal String customerId,
    @RequestParam(defaultValue = "0") int page) {

    return orderListViewRepository
        .findByCustomerIdOrderByOrderedAtDesc(customerId, PageRequest.of(page, 20))
        .map(OrderListResponse::from);
    // 단일 쿼리, 빠름, 외부 서비스 호출 없음
}
```

---

## 🔬 내부 동작 원리

### 1. 이벤트 기반 읽기 모델 동기화

```java
// 주문 이벤트 → 읽기 모델 생성
@Component
public class OrderListViewUpdater {

    @EventListener
    @Transactional
    public void on(OrderPlaced event) {
        OrderListView view = new OrderListView();
        view.setOrderId(event.orderId().value().toString());
        view.setCustomerId(event.customerId().value().toString());
        view.setStatus("PENDING");
        view.setTotalAmount(event.totalAmount().amount());
        view.setOrderedAt(event.placedAt());
        view.setTotalItemCount(event.lines().size());

        // 주문의 첫 번째 상품명/이미지 (대표 상품)
        event.lines().stream().findFirst().ifPresent(line -> {
            view.setMainProductName(line.productName());
            view.setMainProductThumbnailUrl(line.thumbnailUrl());
        });
        view.setLastSyncedAt(LocalDateTime.now());
        orderListViewRepository.save(view);
    }

    // 주문 상태 변경 시 읽기 모델 업데이트
    @EventListener
    @Transactional
    public void on(OrderStatusChanged event) {
        orderListViewRepository.findById(event.orderId().value().toString())
            .ifPresent(view -> {
                view.setStatus(event.newStatus().name());
                view.setLastSyncedAt(LocalDateTime.now());
                orderListViewRepository.save(view);
            });
    }

    // 배송 Context 이벤트 수신 → 읽기 모델 업데이트
    @KafkaListener(topics = "shipping.events")
    @Transactional
    public void on(ShipmentDispatched event) {
        orderListViewRepository.findByOrderId(event.orderId().value().toString())
            .ifPresent(view -> {
                view.setShipmentStatus("SHIPPED");
                view.setTrackingNumber(event.trackingNumber().value());
                view.setEstimatedDelivery(event.estimatedDelivery());
                view.setLastSyncedAt(LocalDateTime.now());
                orderListViewRepository.save(view);
            });
    }

    // 리뷰 Context 이벤트
    @KafkaListener(topics = "review.events")
    @Transactional
    public void on(ReviewPosted event) {
        orderListViewRepository.findByOrderId(event.orderId().value().toString())
            .ifPresent(view -> {
                view.setHasReview(true);
                orderListViewRepository.save(view);
            });
    }
}
```

### 2. CQRS 구조

```
Command Side (쓰기):
  OrderApplicationService.placeOrder() → Order Aggregate 수정
  → 도메인 불변식 보장
  → 이벤트 발행 (Outbox)
  → 단순 쿼리만 허용 (복잡 JOIN 없음)

Query Side (읽기):
  OrderListViewRepository → 읽기 전용 테이블
  → 복잡한 JOIN, 집계 허용
  → 도메인 불변식 없음 (단순 데이터 반환)
  → 캐싱 가능 (데이터 변경 없음)

코드 분리:
  com.example.order.application/
    command/
      OrderApplicationService.java     ← 쓰기 전용
    query/
      OrderQueryService.java           ← 읽기 전용

  com.example.order.infrastructure/
    persistence/
      JpaOrderRepository.java          ← 쓰기 Repository
    readmodel/
      OrderListViewRepository.java     ← 읽기 전용 Repository
      OrderListViewUpdater.java        ← 이벤트 핸들러
```

### 3. 읽기 모델 쿼리 — 페이지네이션 + 검색 + 정렬

```java
public interface OrderListViewRepository extends JpaRepository<OrderListView, String> {

    // 기본 목록 (최신순)
    Page<OrderListView> findByCustomerIdOrderByOrderedAtDesc(
        String customerId, Pageable pageable
    );

    // 상태 필터
    Page<OrderListView> findByCustomerIdAndStatusInOrderByOrderedAtDesc(
        String customerId, List<String> statuses, Pageable pageable
    );

    // 기간 필터
    Page<OrderListView> findByCustomerIdAndOrderedAtBetweenOrderByOrderedAtDesc(
        String customerId, LocalDateTime from, LocalDateTime to, Pageable pageable
    );

    // 복합 검색 (Querydsl 또는 Specification)
    @Query("""
        SELECT v FROM OrderListView v
        WHERE v.customerId = :customerId
          AND (:status IS NULL OR v.status = :status)
          AND (:productName IS NULL OR v.mainProductName LIKE %:productName%)
          AND v.orderedAt BETWEEN :from AND :to
        ORDER BY v.orderedAt DESC
        """)
    Page<OrderListView> searchOrders(
        @Param("customerId") String customerId,
        @Param("status") String status,
        @Param("productName") String productName,
        @Param("from") LocalDateTime from,
        @Param("to") LocalDateTime to,
        Pageable pageable
    );

    // 관리자 전체 조회
    @Query("SELECT v FROM OrderListView v ORDER BY v.orderedAt DESC")
    Page<OrderListView> findAllOrders(Pageable pageable);
}
```

### 4. 읽기 모델 정합성 검증 및 복구

```java
// 정기 검증 배치
@Scheduled(cron = "0 0 3 * * *")  // 새벽 3시 매일
@Transactional
public void reconcileOrderListView() {
    LocalDateTime since = LocalDateTime.now().minusDays(1);

    // 최근 24시간 주문 중 읽기 모델 없는 것 찾기
    List<Order> recentOrders = orderRepository.findOrdersSince(since);

    recentOrders.forEach(order -> {
        orderListViewRepository.findById(order.id().value().toString())
            .ifPresentOrElse(
                view -> validateConsistency(order, view),
                () -> {
                    log.warn("읽기 모델 누락: orderId={}", order.id());
                    rebuildView(order);
                }
            );
    });
}

private void validateConsistency(Order order, OrderListView view) {
    if (!order.status().name().equals(view.getStatus())) {
        log.warn("읽기 모델 상태 불일치: orderId={}, domain={}, view={}",
            order.id(), order.status(), view.getStatus());
        view.setStatus(order.status().name());
        view.setLastSyncedAt(LocalDateTime.now());
        orderListViewRepository.save(view);
        metricsService.increment("order.view.inconsistency");
    }
}

// 전체 재구성 (관리자 API)
@PostMapping("/admin/rebuild-order-views")
@PreAuthorize("hasRole('ADMIN')")
public void rebuildAllOrderViews() {
    log.info("주문 읽기 모델 전체 재구성 시작");
    orderListViewRepository.deleteAll();

    orderRepository.findAll().forEach(order -> {
        // 이벤트 재발행 또는 직접 재구성
        rebuildView(order);
    });
    log.info("재구성 완료");
}
```

---

## 💻 실전 코드

### DTO와 응답 모델

```java
// 읽기 모델 → API 응답 DTO
public record OrderListResponse(
    String orderId,
    String status,
    String statusDisplayName,          // "배송 중", "배달 완료" 등 한글 표시
    BigDecimal totalAmount,
    int totalItemCount,
    String mainProductName,
    String mainProductThumbnailUrl,
    String shipmentStatus,
    String trackingNumber,
    LocalDate estimatedDelivery,
    boolean hasReview,
    LocalDateTime orderedAt
) {
    public static OrderListResponse from(OrderListView view) {
        return new OrderListResponse(
            view.getOrderId(),
            view.getStatus(),
            OrderStatusDisplayName.of(view.getStatus()),  // 표시용 이름 변환
            view.getTotalAmount(),
            view.getTotalItemCount(),
            view.getMainProductName(),
            view.getMainProductThumbnailUrl(),
            view.getShipmentStatus(),
            view.getTrackingNumber(),
            view.getEstimatedDelivery(),
            view.isHasReview(),
            view.getOrderedAt()
        );
    }
}

// Query Service: 읽기만 담당
@Service
@Transactional(readOnly = true)  // 읽기 전용 트랜잭션
public class OrderQueryService {

    private final OrderListViewRepository viewRepository;

    public Page<OrderListResponse> getMyOrders(String customerId, OrderSearchQuery query) {
        Pageable pageable = PageRequest.of(query.page(), query.size(),
            Sort.by(Sort.Direction.DESC, "orderedAt"));

        Page<OrderListView> views = viewRepository.searchOrders(
            customerId,
            query.status(),
            query.productName(),
            query.from() != null ? query.from() : LocalDateTime.now().minusYears(1),
            query.to() != null ? query.to() : LocalDateTime.now(),
            pageable
        );

        return views.map(OrderListResponse::from);
    }
}
```

---

## 📊 설계 비교

```
다중 Context 실시간 조합 vs 읽기 모델 동기화:

                실시간 조합              읽기 모델 동기화
────────────┼──────────────────────┼──────────────────────────
데이터 신선도 │ 항상 최신              │ 이벤트 처리 지연 (수 초)
────────────┼──────────────────────┼──────────────────────────
응답 속도    │ 여러 Context 합산      │ 단일 쿼리 (빠름)
────────────┼──────────────────────┼──────────────────────────
외부 장애 격리│ 하나 장애 → 화면 실패 │ 읽기는 독립적
────────────┼──────────────────────┼──────────────────────────
구현 복잡도  │ 낮음 (직접 호출)       │ 높음 (이벤트 핸들러 + 재구성)
────────────┼──────────────────────┼──────────────────────────
페이지네이션 │ 어려움 (N+1)          │ 쉬움 (단일 테이블)
────────────┼──────────────────────┼──────────────────────────
검색/정렬   │ 제한적                │ SQL로 자유롭게
```

---

## ⚖️ 트레이드오프

```
읽기 모델의 어려움:
  이벤트 누락 시 읽기 모델 불일치 → 정기 검증 배치 필요
  이벤트 순서 역전 → 타임스탬프 기반 "최신 것이 이김" 로직
  스키마 변경 시 전체 재구성 필요

허용 지연:
  주문 상태: 즉시 (같은 트랜잭션 이벤트)
  배송 현황: 수 초 (Kafka 처리 시간)
  리뷰 여부: 수 초~수 분 (비중요)
  
  → 비즈니스와 합의된 SLA 정의 필요
```

---

## 📌 핵심 정리

```
읽기 모델 분리 핵심:

읽기 모델 목적:
  여러 Context 데이터를 단일 쿼리로 조회
  복잡한 JOIN, 페이지네이션, 검색 지원
  외부 Context 장애와 무관한 읽기

동기화 방법:
  Domain Event → 읽기 모델 업데이트 핸들러
  Kafka 이벤트 → 다른 Context 데이터 동기화
  정기 재구성 → 불일치 복구

CQRS 분리:
  Command: OrderApplicationService (불변식 보장)
  Query: OrderQueryService (복잡 쿼리 허용)
  코드 패키지도 command/, query/로 분리

정합성 관리:
  새벽 배치로 원본 vs 읽기 모델 비교
  불일치 감지 시 재구성 + 알림
  관리자 API로 전체 재구성 가능
```

---

## 🤔 생각해볼 문제

**Q1.** "주문 직후 배송 현황이 바로 보여야 한다"는 기획 요구사항이 왔다. 읽기 모델의 지연과 어떻게 타협하는가?

<details>
<summary>해설 보기</summary>

**낙관적 UI 업데이트와 폴링/WebSocket으로 해결합니다.**

```javascript
// 프론트엔드: 주문 완료 직후 낙관적 UI 업데이트
function placeOrder(orderData) {
    return orderApi.placeOrder(orderData).then(orderId => {
        // API 응답 전에 즉시 UI 업데이트
        orderList.unshift({
            orderId,
            status: 'PENDING',
            statusDisplayName: '주문 접수',
            // 주문 데이터로 임시 표시
            mainProductName: orderData.lines[0].productName,
            shipmentStatus: null  // "배송 준비 중"
        });

        // 백그라운드에서 실제 데이터 동기화 확인
        pollUntilSynced(orderId);
    });
}

// 배송 현황 실시간 업데이트
function pollShipmentStatus(orderId) {
    const interval = setInterval(async () => {
        const status = await orderApi.getShipmentStatus(orderId);
        if (status.trackingNumber) {
            updateOrderView(orderId, status);
            clearInterval(interval);
        }
    }, 3000);  // 3초마다 확인
}
```

기획자에게: "배송 현황(송장번호)은 택배사가 데이터를 전송하는 데 최소 수 분이 걸립니다. 주문 접수 확인은 즉시, 배송 현황은 '잠시 후 업데이트됩니다' 메시지로 처리합니다."

</details>

---

**Q2.** 읽기 모델 테이블이 매우 커져서 조회가 느려졌다. 어떻게 최적화하는가?

<details>
<summary>해설 보기</summary>

**파티셔닝, 인덱스, 보관 정책을 적용합니다.**

```sql
-- 인덱스 최적화
CREATE INDEX idx_order_list_view_customer_ordered
    ON order_list_view(customer_id, ordered_at DESC);

CREATE INDEX idx_order_list_view_status
    ON order_list_view(status)
    WHERE status IN ('PENDING', 'PAID', 'PREPARING', 'SHIPPED');

-- 파티셔닝 (월별)
CREATE TABLE order_list_view_2024_01
    PARTITION OF order_list_view
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- 보관 정책: 2년 이상 주문은 아카이브
@Scheduled(cron = "0 0 4 1 * *")  // 매월 1일 새벽 4시
public void archiveOldOrderViews() {
    LocalDateTime cutoff = LocalDateTime.now().minusYears(2);
    orderListViewArchiveRepository.archiveOlderThan(cutoff);
    orderListViewRepository.deleteOlderThan(cutoff);
}
```

"최근 1년" 주문이 자주 조회된다면 해당 기간만 인덱스에 집중하고, 오래된 주문은 별도 아카이브 테이블로 이동합니다.

</details>

---

**Q3.** 읽기 모델이 있으면 원본 Order Aggregate는 어떤 용도로만 사용하는가?

<details>
<summary>해설 보기</summary>

**Order Aggregate는 쓰기(명령) 전용으로, 도메인 불변식 보장과 이벤트 발행만 담당합니다.**

```java
// Order Aggregate 사용 패턴:
// ✅ 사용 (Command)
orderRepository.findById(orderId) → order.cancel() → orderRepository.save(order)
orderRepository.findById(orderId) → order.addItem() → orderRepository.save(order)

// ❌ 사용 금지 (Query — 읽기 모델로)
orderRepository.findByCustomerId(customerId)  // 목록 조회는 읽기 모델로
orderRepository.findByStatus("PENDING")       // 상태 조회는 읽기 모델로

// 예외: 도메인 로직 실행에 필요한 단건 조회
orderRepository.findById(orderId)  // → 취소, 결제 등 명령 실행 전
```

CQRS의 핵심: Command(쓰기)는 도메인 모델(Aggregate), Query(읽기)는 읽기 모델(View). 둘을 섞지 않습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Context 간 통합](./03-context-integration.md)** | **[홈으로 🏠](../README.md)** | **[다음: DDD 리팩터링 ➡️](./05-ddd-refactoring.md)**

</div>
