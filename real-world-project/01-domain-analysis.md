# 도메인 분석 — 전자상거래 Bounded Context 식별

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 전자상거래 도메인에서 어떤 기준으로 Bounded Context를 식별하는가?
- 주문 / 상품 / 회원 / 배송 / 결제 각 Context의 핵심 모델과 Ubiquitous Language는?
- Context Map을 작성해 각 Context 간 통합 패턴을 어떻게 결정하는가?
- "상품"이라는 단어가 각 Context마다 다른 의미를 가지는 것은 어떻게 처리하는가?
- Core / Supporting / Generic Subdomain을 전자상거래에서 어떻게 분류하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

이론으로 배운 DDD 개념이 실제 전자상거래 시스템에 어떻게 적용되는지 살펴보자. "주문 시스템을 만들어라"라는 요구사항이 들어왔을 때, 첫 번째 질문은 "어떤 데이터를 어느 테이블에 저장할까?"가 아니라 "이 도메인에는 어떤 개념들이 있고, 그것들이 어떻게 협력하는가?"다.

도메인 분석은 소프트웨어 아키텍처의 출발점이다. Context 경계를 잘못 설정하면 나중에 수정 비용이 기하급수적으로 커진다.

---

## 😱 흔한 실수 (Before — Context 경계 없이 단일 모델로 시작)

```
단일 모델 안티패턴:

Product 테이블:
  id, name, description, price, stock_quantity, category_id,
  seller_id, thumbnail_url, detail_images, weight, shipping_method,
  review_count, rating_average, is_on_sale, sale_price, ...

문제: "상품"이 너무 많은 것을 담당
  상품 정보 관리 (셀러가 관리)
  가격/할인 정책 (운영팀이 관리)
  재고 수량 (창고 시스템이 관리)
  리뷰/평점 (고객이 작성)
  배송 방법 (물류팀이 관리)

한 테이블에 모든 담당자가 접근 → 동시 수정 충돌 → 락 경합
"가격 변경"이 "재고 조회"에 영향 → 불필요한 의존
셀러 팀이 재고 시스템 코드를 이해해야 함 → 높은 인지 부하
```

---

## ✨ 올바른 접근 (After — Context 식별과 경계 설정)

```
전자상거래 Bounded Context 맵:

┌─────────────────────────────────────────────────────────────────┐
│                    전자상거래 시스템                              │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│  │  상품    │    │  주문    │    │  결제    │                  │
│  │ Context  │    │ Context  │    │ Context  │                  │
│  │          │    │          │    │          │                  │
│  │ - 상품목록│    │ - 주문   │    │ - 결제   │                  │
│  │ - 카탈로그│    │ - 주문항목│    │ - 환불   │                  │
│  │ - 검색   │    │ - 장바구니│    │ - 결제수단│                  │
│  └──────────┘    └──────────┘    └──────────┘                  │
│                                                                 │
│  ┌──────────┐    ┌──────────┐    ┌──────────┐                  │
│  │  재고    │    │  배송    │    │  회원    │                  │
│  │ Context  │    │ Context  │    │ Context  │                  │
│  │          │    │          │    │          │                  │
│  │ - 재고   │    │ - 배송   │    │ - 회원   │                  │
│  │ - 입고   │    │ - 배송현황│    │ - 포인트 │                  │
│  │ - 예약   │    │ - 반품   │    │ - 등급   │                  │
│  └──────────┘    └──────────┘    └──────────┘                  │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔬 내부 동작 원리

### 1. Context 식별 기준

```
기준 1: Ubiquitous Language의 경계

"상품(Product)"이 각 Context에서 다른 의미:

  상품 Context (카탈로그):
    상품 = 판매 가능한 항목의 명세
    상품명, 설명, 이미지, 카테고리
    → Catalog Item

  주문 Context:
    상품 = 이 주문에서 구매한 항목의 스냅샷
    주문 시점의 상품명, 가격 (이후 변경 무관)
    → Order Line Item (스냅샷)

  재고 Context:
    상품 = 창고에서 추적하는 물리적 단위
    SKU, 위치, 수량, 배치 번호
    → Stock Keeping Unit (SKU)

  배송 Context:
    상품 = 배송할 물리적 패키지 구성
    무게, 크기, 취급 주의 여부
    → Shipment Item

→ 같은 "상품"이 Context마다 다른 이름과 의미를 가짐
→ Context 경계를 넘으면 개념이 달라짐

기준 2: 변경 이유와 속도

  카탈로그: 마케팅팀이 변경, 주 단위
  재고: 창고 시스템이 변경, 실시간
  가격: 운영팀이 변경, 이벤트성
  배송: 물류팀이 변경, 계약 변경 시

→ 변경 주체와 빈도가 다르면 다른 Context

기준 3: 팀 구조 (Conway의 법칙)
  상품팀 → 상품 Context
  물류팀 → 배송 + 재고 Context
  결제팀 → 결제 Context
  고객팀 → 회원 Context
```

### 2. 각 Context의 Ubiquitous Language

```
주문 Context (Core Domain):
  - 주문(Order): 고객이 상품을 구매하기로 한 약속
  - 주문항목(OrderLine): 특정 상품을 특정 수량으로 구매하는 행
  - 주문상태: 접수 → 결제완료 → 배송중 → 배송완료 → 취소/반품
  - 장바구니(Cart): 주문 전 임시 보관소
  - 총 결제금액: 상품금액 + 배송비 - 할인금액

상품 Context (Core Domain):
  - 상품(Product): 판매 가능한 항목의 공개 명세
  - SKU(Stock Keeping Unit): 색상/사이즈 조합의 최소 단위
  - 카테고리: 상품 분류 체계
  - 상품가격: 정가, 할인가, 기간별 가격
  - 상품이미지: 대표이미지, 상세이미지

재고 Context (Supporting Domain):
  - 재고(Inventory): 창고에 보유한 상품의 물리적 수량
  - 예약(Reservation): 주문 확정을 위해 임시로 확보한 재고
  - 입고(Inbound): 외부에서 재고를 받아 등록하는 행위
  - 재고부족(Stockout): 재고가 0 이하로 떨어진 상태

결제 Context (Supporting Domain):
  - 결제(Payment): 구매 대금 지불 행위
  - 결제수단: 신용카드, 계좌이체, 포인트, 쿠폰
  - 승인: PG사로부터 결제가 확정된 상태
  - 환불(Refund): 결제를 취소하고 금액을 되돌리는 행위

배송 Context (Supporting Domain):
  - 배송(Shipment): 물리적 상품을 고객에게 전달하는 과정
  - 운송장번호: 배송사에서 부여한 추적 번호
  - 배송현황: 집하 → 이동중 → 배달중 → 배달완료
  - 반품(Return): 배송된 상품을 되돌려 받는 과정

회원 Context (Generic Domain):
  - 회원(Member): 서비스를 이용하는 개인
  - 포인트(Point): 구매 이력에 따라 적립되는 가상 화폐
  - 등급(MembershipLevel): Bronze, Silver, Gold, VIP
  - 주소록: 회원이 등록한 배송지 목록
```

### 3. Context Map — 통합 패턴 결정

```
Context 간 관계와 통합 패턴:

주문 Context ↔ 상품 Context:
  관계: Customer/Supplier (주문이 소비자, 상품이 공급자)
  통합: ACL + Published Language
  흐름: 주문 시 상품 Context에서 현재 가격/정보 조회
        주문 생성 시점의 정보를 OrderLine에 스냅샷으로 저장
  이벤트: ProductPriceChanged → 장바구니의 가격 갱신

주문 Context → 재고 Context:
  관계: Customer/Supplier (주문이 소비자, 재고가 공급자)
  통합: Domain Event (비동기)
  흐름: OrderPlaced → InventoryDecrease
        OrderCancelled → InventoryRestore

주문 Context → 결제 Context:
  관계: Customer/Supplier
  통합: Saga (주문-결제는 강한 조율 필요)
  흐름: 주문 → 결제 요청 → 결제 승인 → 주문 확정
        주문 취소 → 환불 요청 → 환불 완료

주문 Context → 배송 Context:
  관계: Customer/Supplier
  통합: Domain Event (비동기)
  흐름: OrderPaid → ShipmentCreate
        ShipmentDelivered → OrderCompleted

회원 Context → 주문 Context:
  관계: Upstream (회원이 주문의 Upstream)
  통합: Domain Event (포인트 적립)
  흐름: OrderCompleted → PointEarned

관계 요약:
  주문 Context = 중심 (Core Domain)
  다른 Context는 주문의 생명주기 이벤트에 반응
```

### 4. Subdomain 분류

```
Core Subdomain (경쟁 우위를 만드는 핵심):
  주문 Context: 주문 생성, 상태 전이, 불변식
  상품 Context: 상품 구성, 가격 정책, 검색
  
  → 직접 개발, 최고의 설계 적용
  → DDD + 풍부한 도메인 모델

Supporting Subdomain (핵심을 지원):
  재고 Context: 재고 관리, 예약
  배송 Context: 배송 추적, 배송사 연동
  결제 Context: PG사 연동, 정산
  
  → 직접 개발 또는 맞춤 SaaS
  → 상대적으로 단순한 설계 허용

Generic Subdomain (범용, 사외 구매 가능):
  회원 인증: Keycloak, OAuth2 Provider 활용
  이메일/SMS: 외부 서비스 활용
  파일 스토리지: S3, CDN 활용
  
  → 외부 서비스 사용 (직접 구현 낭비)
```

---

## 💻 실전 코드

### Context 경계 코드로 표현하기

```java
// 주문 Context: 상품 정보를 스냅샷으로 저장
public class OrderLine {

    private final ProductId productId;        // 상품 Context의 ID만 참조
    private final String productNameSnapshot; // 주문 시점 스냅샷 (이후 변경 무관)
    private final Money unitPriceSnapshot;    // 주문 시점 가격 스냅샷
    private int quantity;

    // 주문 Context에서의 "상품"은 스냅샷 값
    public OrderLine(ProductId productId, String productName,
                     Money unitPrice, int quantity) {
        this.productId = productId;
        this.productNameSnapshot = productName;   // 스냅샷 저장
        this.unitPriceSnapshot = unitPrice;       // 스냅샷 저장
        this.quantity = quantity;
    }

    public Money subtotal() {
        return unitPriceSnapshot.multiply(quantity);  // 스냅샷 가격 사용
    }
}

// 상품 Context: 가격이 변해도 기존 주문에 영향 없음
public class Product {  // 상품 Context의 Product
    private final ProductId id;
    private String name;
    private Money currentPrice;   // 현재 가격 (변경 가능)

    public void updatePrice(Money newPrice) {
        Money oldPrice = this.currentPrice;
        this.currentPrice = newPrice;
        registerEvent(new ProductPriceChanged(this.id, oldPrice, newPrice));
        // 장바구니(Cart)는 이 이벤트를 수신해 가격 업데이트
        // 완료된 OrderLine은 스냅샷이므로 영향 없음
    }
}

// 재고 Context: "상품"을 SKU 단위로 관리
public class Inventory {  // 재고 Context의 재고
    private final SkuId skuId;            // 재고 Context의 식별자
    private final ProductId productId;    // 상품 Context ID 참조
    private int quantityOnHand;           // 실제 보유 수량
    private int reservedQuantity;         // 예약된 수량

    public int availableQuantity() {
        return quantityOnHand - reservedQuantity;  // 재고 Context의 "가용 수량" 개념
    }
}
```

---

## 📊 설계 비교

```
단일 모델 vs Context 분리:

                단일 모델              Context 분리
────────────┼──────────────────────┼──────────────────────────
"상품" 개념  │ 하나의 Product 클래스  │ Context별 다른 모델
            │ (모든 의미 혼재)       │ CatalogItem, OrderLineItem,
            │                      │ Inventory, ShipmentItem
────────────┼──────────────────────┼──────────────────────────
팀 협업     │ 단일 코드베이스 충돌   │ Context별 독립 개발
────────────┼──────────────────────┼──────────────────────────
변경 영향   │ 가격 변경 → 전체 영향 │ 가격 변경 → 상품 Context만
────────────┼──────────────────────┼──────────────────────────
언어 명확성 │ "상품"의 의미 불명확   │ Context별 명확한 언어
────────────┼──────────────────────┼──────────────────────────
테스트      │ 전체 결합 테스트       │ Context별 독립 테스트
```

---

## ⚖️ 트레이드오프

```
Context 분리의 비용:
  ① Context 간 통신 복잡도
     직접 DB 조인 불가 → 이벤트 또는 API 호출
     단순한 "상품 목록 + 재고 수량" 조회도 두 Context 필요

  ② 데이터 중복
     OrderLine의 상품명 스냅샷 → 상품 Context와 중복
     → 의도적 중복 (불변성 보장을 위해)

  ③ 초기 설계 비용
     Context 경계 결정에 도메인 전문가와 긴 논의 필요
     잘못 설정하면 나중에 수정 비용 매우 큼

언제 분리를 연기할 것인가:
  도메인 지식이 부족할 때: 모놀리스로 시작 후 분리
  팀 규모가 작을 때: Context가 많으면 관리 부담
  트래픽이 작을 때: 분리의 기술적 이점이 미미
```

---

## 📌 핵심 정리

```
전자상거래 Context 식별 핵심:

6개 Context:
  주문(Core), 상품(Core), 재고(Supporting),
  결제(Supporting), 배송(Supporting), 회원(Generic)

Context 식별 기준:
  Ubiquitous Language 경계 (같은 단어, 다른 의미)
  변경 이유와 속도
  팀 구조 (Conway의 법칙)

"상품"의 다중 의미:
  카탈로그: 공개 명세 (CatalogItem)
  주문: 구매 스냅샷 (OrderLineItem)
  재고: 물리적 단위 (SKU)
  배송: 패키지 구성 (ShipmentItem)

Context Map:
  주문이 중심, 다른 Context가 이벤트로 반응
  주문↔결제: Saga (강한 조율)
  주문→재고,배송: Domain Event (비동기)
```

---

## 🤔 생각해볼 문제

**Q1.** "쿠폰 Context"를 별도로 만들어야 하는가? 아니면 주문 Context나 회원 Context에 통합해야 하는가?

<details>
<summary>해설 보기</summary>

**쿠폰의 복잡도와 팀 구조에 따라 결정합니다.**

```
쿠폰을 별도 Context로 분리하는 경우:
  쿠폰 발급, 유효성, 사용 이력이 복잡한 비즈니스 로직을 가질 때
  마케팅팀이 독립적으로 쿠폰을 관리해야 할 때
  다른 서비스(배달, 쇼핑 등)에서도 쿠폰을 공유해야 할 때

  Coupon Context:
    - 쿠폰 발급 정책 (누구에게, 언제, 얼마)
    - 쿠폰 유효성 검증
    - 사용 이력 관리
    - 동시 사용 제한 (한 주문에 하나)

쿠폰을 주문 Context에 통합하는 경우:
  쿠폰이 단순히 "주문 할인 코드"로만 쓰일 때
  쿠폰 로직이 복잡하지 않을 때
  
  // 주문 Context 내부
  public class CouponCode {
      private final String code;
      public Money discountAmount() { ... }
  }

현실적 권고:
  처음에는 주문 Context 내부의 단순 VO로 시작
  쿠폰 발급 로직이 복잡해지면 별도 Context로 분리
  마케팅팀이 독립적 배포를 원하면 분리 신호
```

</details>

---

**Q2.** 주문 Context와 상품 Context가 분리됐는데, "주문 시 상품 재고를 확인해야 한다"는 요구사항을 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**재고 확인은 재고 Context의 포트를 통해 동기 조회, 차감은 이벤트로 비동기 처리합니다.**

```java
// 주문 Context: 재고 확인을 위한 포트 인터페이스
public interface InventoryCheckPort {
    boolean isAvailable(ProductId productId, int quantity);
    // → 재고 Context의 HTTP API 또는 gRPC 동기 호출
}

// Application Service: 주문 생성 시 재고 확인
@Service
@Transactional
public class OrderApplicationService {

    private final InventoryCheckPort inventoryCheck;

    public OrderId placeOrder(PlaceOrderCommand command) {
        // 1. 재고 동기 확인 (즉각 피드백 필요)
        command.lines().forEach(line -> {
            if (!inventoryCheck.isAvailable(line.productId(), line.quantity())) {
                throw new OutOfStockException(line.productId());
            }
        });

        // 2. 주문 생성 + 저장
        Order order = orderFactory.create(command);
        orderRepository.save(order);

        // 3. 재고 차감은 이벤트로 비동기 (주문 응답에 영향 없음)
        order.pullDomainEvents().forEach(eventPublisher::publishEvent);
        return order.id();
    }
}

// 재고 Context: OrderPlaced 이벤트 수신 → 재고 차감
@EventListener
@Transactional
public void on(OrderPlaced event) {
    event.lines().forEach(line ->
        inventoryRepository.findBySku(line.productId())
            .ifPresent(inv -> {
                inv.decrease(line.quantity());
                inventoryRepository.save(inv);
            })
    );
}
```

</details>

---

**Q3.** 전자상거래 도메인에서 "리뷰(Review)"는 어느 Context에 속하는가?

<details>
<summary>해설 보기</summary>

**리뷰는 독립된 별도 Context 또는 상품 Context의 하위 영역이 적합합니다.**

```
리뷰의 특성 분석:

리뷰 작성 조건: 구매 이력이 있어야 함 → 주문 Context와 연관
리뷰 표시 위치: 상품 상세 페이지 → 상품 Context와 연관
리뷰 관리: 운영팀이 부적절 리뷰 삭제 → 별도 관리 필요

설계 결정:

방법 1: 상품 Context 내부 (단순한 경우)
  Product.reviews: List<Review>  // 상품 Context 내부
  ReviewAggregate: 리뷰 내용, 평점, 작성자 ID, 주문 ID

방법 2: 별도 리뷰 Context (복잡한 경우)
  Review Context:
    - 리뷰 작성 (구매 검증: 주문 Context와 이벤트로 연결)
    - 리뷰 승인/거부 (운영 흐름)
    - 평점 집계 → 상품 Context 읽기 모델 업데이트

  ProductId를 리뷰 Context에서 ID로만 참조
  ReviewPosted → 상품 Context가 평점 집계 업데이트

리뷰 Context의 Ubiquitous Language:
  - 리뷰(Review): 구매한 상품에 대한 평가 기록
  - 구매 인증: 이 리뷰가 실제 구매자로부터 작성됐음을 보증
  - 평점(Rating): 1~5점 척도의 수치 평가
  - 도움돼요: 다른 고객이 이 리뷰가 도움이 됐다고 표시
```

</details>

---

<div align="center">

**[⬅️ 이전: 테스트 전략](../spring-jpa/07-testing-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 주문 Aggregate 설계 ➡️](./02-order-aggregate.md)**

</div>
