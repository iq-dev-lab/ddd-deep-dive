# Context Map 패턴 완전 분해 — 팀 관계가 통합 패턴을 결정한다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Context Map의 7가지 패턴은 각각 어떤 상황에서 사용하는가?
- 왜 "기술적으로 가장 좋은 통합 방법"이 아닌 "팀 관계"가 패턴을 결정하는가?
- Shared Kernel과 Customer-Supplier의 차이는 무엇이고, 언제 각각을 선택하는가?
- Conformist와 Anticorruption Layer 중 어느 것을 선택해야 하는가?
- Context Map을 코드와 아키텍처에 어떻게 반영하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

두 Context를 연결하는 방법은 기술적으로 여러 가지가 있다. REST API, 이벤트, 공유 라이브러리, DB 공유... 그런데 DDD는 "어떤 기술을 쓸 것인가"보다 **"두 팀(또는 시스템)의 관계가 어떠한가"** 가 통합 패턴을 결정한다고 말한다.

권력이 대등한 두 팀이 공유하는 코드는 Shared Kernel이 적합하고, 외부 레거시 시스템의 모델에 우리 도메인이 오염되지 않으려면 Anticorruption Layer가 필요하다. 팀 관계를 먼저 파악하지 않고 기술부터 결정하면, 나중에 팀 간 갈등이 통합 구조 자체를 무너뜨린다.

---

## 😱 흔한 실수 (Before — 팀 관계 무시한 통합)

```
상황: 스타트업이 대형 레거시 ERP와 통합

레거시 ERP의 특성:
  - 20년 된 시스템, 변경 불가
  - 이상한 데이터 구조: customer_type = 'B' (뭔지 설명 없음)
  - 비즈니스 용어가 다름: "Order"를 "SalesDocument"라고 부름
  - API가 없고 DB 직접 접근만 가능

개발팀의 잘못된 접근:
  "일단 ERP DB를 직접 읽어서 우리 코드에서 쓰자"
```

```java
// ERP DB를 직접 사용하는 코드 — 레거시 모델이 도메인을 오염
@Service
public class OrderService {

    @Autowired
    private LegacyErpDataSource legacyDataSource;

    public void processOrder(Long orderId) {
        // ERP의 이상한 구조가 도메인 코드에 직접 침투
        String sql = "SELECT * FROM SALES_DOCUMENT WHERE DOC_TYPE = 'OR' AND DOC_NO = ?";
        Map<String, Object> salesDoc = jdbcTemplate.queryForMap(sql, orderId);

        // ERP 용어가 도메인 코드에 섞임
        String customerType = (String) salesDoc.get("CUST_TYPE");
        // "B"가 뭔지 모르는 상태에서 조건 분기
        if ("B".equals(customerType)) {
            // ... B2B 처리?
        }

        // ERP의 금액 단위가 원화가 아닌 경우 처리 없음
        BigDecimal amount = (BigDecimal) salesDoc.get("NET_AMOUNT");
    }
}
```

```
6개월 후:
  "ERP에서 customer_type 'B'의 의미가 바뀌었습니다"
  → 코드 전체에 "B".equals() 가 30곳에 퍼져 있음
  → 전부 찾아서 수정

  "ERP가 버전업 되면서 SALES_DOCUMENT 테이블 구조가 변경됐습니다"
  → 우리 도메인 코드 전체가 영향받음
  → 레거시 ERP 변경 → 우리 도메인 코드 수정 → 테스트 → 배포
    (이것이 "레거시가 우리를 지배하는" 상황)
```

---

## ✨ 올바른 접근 (After — Context Map 패턴 기반 통합)

```
팀/시스템 관계 분석:
  우리 스타트업: 레거시 ERP를 바꿀 수 없음 (권력 열세)
  레거시 ERP: 변경 불가, 우리에게 API 제공 불가
  → Anticorruption Layer (ACL) 패턴 선택

ACL 구조:
  우리 도메인 → ACL (번역기) → 레거시 ERP
  ERP 변경 → ACL만 수정 → 도메인 무관
```

```java
// ACL — ERP 모델을 도메인 모델로 번역하는 번역기
@Component
public class ErpOrderTranslator {

    private final LegacyErpRepository erpRepository;

    // ERP의 이상한 구조를 우리 도메인 모델로 변환
    public Optional<Order> findOrder(OrderId orderId) {
        Optional<SalesDocument> salesDoc = erpRepository.findSalesDocument(orderId.value());

        return salesDoc.map(doc -> {
            // ERP의 customer_type 번역
            CustomerType customerType = translateCustomerType(doc.getCustomerType());
            // ERP의 금액 단위 번역
            Money amount = Money.ofErpUnit(doc.getNetAmount(), doc.getCurrency());

            return new Order(
                orderId,
                customerType,
                amount
                // 우리 도메인 모델의 필드로 매핑
            );
        });
    }

    private CustomerType translateCustomerType(String erpCode) {
        return switch (erpCode) {
            case "B" -> CustomerType.BUSINESS;  // B의 의미를 ACL이 알고 있음
            case "C" -> CustomerType.CONSUMER;
            default -> throw new UnknownErpCustomerTypeException(erpCode);
        };
    }
}

// 도메인 코드 — ERP를 전혀 모름
@Service
public class OrderApplicationService {

    private final OrderRepository orderRepository;  // ACL을 통해 ERP 접근

    public void processOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        // order.customerType은 BUSINESS 또는 CONSUMER — 깨끗한 도메인 언어
        // ERP의 "B", "C" 코드를 모름
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 7가지 Context Map 패턴

```
패턴 1: Shared Kernel (공유 커널)
────────────────────────────────
정의: 두 Context가 특정 코드/데이터를 공동 소유

    Context A ←──공유 코드──→ Context B
    (팀 A 소유)                (팀 B 소유)

적합한 상황:
  - 두 팀이 긴밀히 협력하고 신뢰 관계 있음
  - 공유할 코드가 작고 안정적 (자주 변경되지 않음)
  - 예: Money, UserId, CommonEventBase 같은 순수 Value Object

주의사항:
  - 공유 코드 변경 시 두 팀 모두 합의 필요
  - 공유 범위를 최소화 (가능한 작게)
  - 팀 간 신뢰가 없으면 변경 갈등 발생

────────────────────────────────
패턴 2: Customer-Supplier (고객-공급자)
────────────────────────────────
정의: Supplier가 API를 제공하고, Customer가 그것을 소비

    Customer Context ──의존→ Supplier Context
    (API를 사용)              (API를 제공)

특징:
  - Supplier가 API 변경 시 Customer에게 영향
  - Supplier가 Customer의 요구사항을 반영해줄 의지 있음
  - Supplier에 계획/요구사항을 조기에 전달

예시:
  주문 팀(Customer) ← 상품 팀(Supplier)이 제공하는 상품 조회 API
  → 상품팀이 API 변경 시 주문팀과 사전 협의

────────────────────────────────
패턴 3: Conformist (순응자)
────────────────────────────────
정의: Supplier의 모델을 그대로 따름 (번역 없이 수용)

    Context A ──완전 순응→ Context B (변경 불가 또는 무관심)

적합한 상황:
  - Supplier가 우리 요구를 반영할 의지 없음
  - Supplier 모델이 충분히 우리 도메인에 맞음
  - 번역 비용이 이익보다 큼

단점:
  - Supplier 모델 변경 시 즉각 영향
  - 우리 도메인 언어가 Supplier 언어에 오염

예시:
  Google Maps API를 Conformist로 사용 (모델을 그대로 씀)
  → Maps 모델이 우리 도메인에 잘 맞는 경우

────────────────────────────────
패턴 4: Anticorruption Layer (부패 방지 레이어)
────────────────────────────────
정의: 외부 모델로부터 우리 도메인을 보호하는 번역 레이어

    우리 도메인 → ACL (번역기) → 외부 Context

적합한 상황:
  - 외부 시스템 모델이 우리 도메인에 맞지 않음
  - 레거시 시스템, 외부 API와 통합
  - 외부 모델 변경이 우리 도메인에 전파되지 않아야 함

구현:
  Facade, Translator, Adapter 패턴 조합

────────────────────────────────
패턴 5: Open Host Service (개방형 호스트 서비스)
────────────────────────────────
정의: 우리 Context를 여러 Consumer가 사용할 수 있도록 공개 API 제공

    Consumer 1 ┐
    Consumer 2 ├→ Open Host Service → 우리 Context
    Consumer 3 ┘

적합한 상황:
  - 우리 Context가 여러 다른 Context에서 사용됨
  - 각 Consumer마다 다른 통합 방식을 원함
  - 안정적인 공개 API가 필요

예시:
  상품 검색 Context → 웹, 앱, 제휴사 등 다양한 Consumer에 API 제공
  REST API 또는 이벤트 스키마로 정의된 인터페이스

────────────────────────────────
패턴 6: Published Language (공표된 언어)
────────────────────────────────
정의: 표준화된 스키마/프로토콜로 통신

    Context A ──표준 스키마──→ Context B

적합한 상황:
  - 특정 팀에 종속되지 않는 표준이 필요
  - 불특정 다수의 Consumer가 통합
  - Open Host Service와 함께 사용

예시:
  이벤트 스키마를 CloudEvents 표준으로 정의
  OpenAPI(Swagger) 명세로 API 정의
  → Consumer가 이 표준에 맞춰 통합

────────────────────────────────
패턴 7: Separate Ways (별도 방식)
────────────────────────────────
정의: 두 Context가 아예 통합하지 않음

    Context A        Context B
    (독립)            (독립)

적합한 상황:
  - 통합 비용이 이익보다 큼
  - 실제로 두 Context가 서로 필요하지 않음
  - 중복을 허용하는 것이 결합보다 낫다고 판단

예시:
  결제 팀과 HR 팀은 통합할 이유가 없음
  → Separate Ways로 각자 독립 발전
```

### 2. 팀 관계가 패턴을 결정하는 이유

```
기술 선택 전에 팀 관계를 먼저 파악:

권력 관계 분석:
  우리가 상위 (Supplier): 우리가 API를 설계하고 제공
    → Open Host Service + Published Language
  
  대등한 관계 (협력): 두 팀이 함께 결정
    → Shared Kernel (최소화) 또는 Customer-Supplier
  
  우리가 하위 (Customer): 외부/레거시에 의존
    외부 모델이 우리와 잘 맞음 → Conformist
    외부 모델이 우리를 오염시킴 → Anticorruption Layer

협력 의지 분석:
  외부가 우리 요구를 들어줄 의지 있음:
    → Customer-Supplier (Supplier에게 요구사항 반영 요청 가능)
  
  외부가 우리 요구를 무시함 (또는 변경 불가):
    → ACL 또는 Conformist

현실적 예시:
  내부 팀 A → 내부 팀 B: Customer-Supplier (협력 가능)
  우리 → AWS S3: Conformist (S3 API를 그대로 사용, 오염 적음)
  우리 → 레거시 ERP: ACL (ERP 모델이 이상하고 변경 불가)
  우리 → 외부 파트너 API: ACL (파트너 모델이 우리와 다름)
  공유 Value Object: Shared Kernel (Money, Address 등)
  공개 서비스: Open Host Service + Published Language
```

### 3. Context Map 시각화

```
실전 Context Map 예시 — 이커머스 플랫폼:

                     ┌──────────────────────────────────────────────────┐
                     │                 Context Map                       │
                     └──────────────────────────────────────────────────┘

┌─────────────────┐      Customer-Supplier      ┌─────────────────────┐
│   주문 Context  │◄──────────────────────────  │   상품 Context      │
│  (팀 A 소유)    │  (주문이 상품 정보 조회)       │  (팀 B 소유)        │
└────────┬────────┘                            └─────────────────────┘
         │
         │ Published Language (OrderPlaced 이벤트)
         ▼
┌─────────────────┐      ACL                    ┌─────────────────────┐
│   배송 Context  │◄──────────────────────────  │   외부 택배사 API   │
│  (팀 C 소유)    │  (택배사 모델 번역)            │  (변경 불가)        │
└─────────────────┘                            └─────────────────────┘
         ▲
         │ Published Language (ShipmentDispatched 이벤트)
         │
┌─────────────────┐      Conformist             ┌─────────────────────┐
│   결제 Context  │──────────────────────────►  │   PG사 SDK          │
│  (팀 D 소유)    │  (PG사 모델이 우리에게 맞음)   │  (Toss Payments)    │
└─────────────────┘                            └─────────────────────┘

         ┌──────────────────────────────────────┐
         │         Shared Kernel                 │
         │  Money.java, UserId.java              │
         │  (팀 A, B, C, D 공동 소유, 변경 희귀)  │
         └──────────────────────────────────────┘

───────────────────────────────────────────────
패턴 선택 이유:

주문 ← 상품 (Customer-Supplier):
  팀 A가 팀 B의 API를 사용. 팀 B는 팀 A의 요구를 들어줌.
  → Customer-Supplier

배송 ← 외부 택배사 (ACL):
  택배사 API가 이상하고 변경 불가, 우리 도메인 오염 방지 필요
  → ACL로 번역

결제 → PG사 (Conformist):
  Toss Payments SDK는 잘 설계되어 있고 우리 도메인에 큰 영향 없음
  → Conformist (번역 비용 < 번역 이익)

Shared Kernel:
  Money, UserId처럼 모든 Context에서 동일하게 쓰는 순수 VO
  → 최소화해서 공유
```

---

## 💻 실전 코드

### Customer-Supplier 패턴 구현

```java
// Supplier Context (상품 Context)가 제공하는 API
// → Open Host Service로 설계
@RestController
@RequestMapping("/api/products")
public class ProductApiController {

    // Customer(주문 Context)가 필요로 하는 정보만 제공
    // Supplier가 자신의 내부 모델을 그대로 노출하지 않음 (Published Language)
    @GetMapping("/{productId}/price-info")
    public ProductPriceInfoResponse getPriceInfo(@PathVariable Long productId) {
        Product product = productService.findById(new ProductId(productId));
        return new ProductPriceInfoResponse(
            product.getId().value(),
            product.currentPrice().amount(),
            product.currentPrice().currency()
        );
        // 상품의 모든 정보가 아닌, 주문에 필요한 가격 정보만 제공
    }
}

// Customer Context (주문 Context)에서 Supplier API 사용
@Component
public class ProductPriceClient {  // Anti-corruption이 필요 없을 때

    private final RestTemplate restTemplate;

    public Money getPriceOf(ProductId productId) {
        ProductPriceInfoResponse response = restTemplate.getForObject(
            "/api/products/{id}/price-info", ProductPriceInfoResponse.class, productId.value()
        );
        return Money.of(response.amount(), response.currency());
    }
}
```

### Anticorruption Layer 완전 구현

```java
// 외부 레거시 ERP와 통합 — ACL 패턴
// 계층 구조: 도메인 → Port(인터페이스) → ACL → ERP

// 1. 도메인 레이어: Port 정의 (도메인이 필요한 것을 인터페이스로)
public interface LegacyOrderPort {
    Optional<Order> findOrderById(OrderId orderId);
    List<Order> findOrdersByCustomer(CustomerId customerId);
}

// 2. ACL: Port 구현 + ERP 번역
@Component
public class ErpOrderAdapter implements LegacyOrderPort {

    private final ErpJdbcClient erpJdbcClient;
    private final ErpOrderTranslator translator;

    @Override
    public Optional<Order> findOrderById(OrderId orderId) {
        // ERP의 이상한 쿼리 → 우리 도메인 모델로 번역
        return erpJdbcClient.querySalesDocument(orderId.value())
            .map(translator::toDomain);
    }
}

// 3. Translator: 번역 로직 집중
@Component
public class ErpOrderTranslator {

    public Order toDomain(ErpSalesDocument salesDoc) {
        return new Order(
            new OrderId(salesDoc.getDocNo()),
            translateCustomer(salesDoc),
            translateAmount(salesDoc),
            translateStatus(salesDoc)
        );
    }

    private OrderStatus translateStatus(ErpSalesDocument doc) {
        // ERP의 status 코드 → 우리 도메인 언어
        return switch (doc.getStatusCode()) {
            case "A0" -> OrderStatus.PENDING;
            case "B1" -> OrderStatus.CONFIRMED;
            case "C2" -> OrderStatus.SHIPPED;
            case "D9" -> OrderStatus.CANCELLED;
            default -> throw new UnknownErpStatusException(
                "알 수 없는 ERP 상태 코드: " + doc.getStatusCode()
            );
        };
    }
}

// 4. 도메인은 ERP를 전혀 모름
@Service
public class OrderApplicationService {

    private final LegacyOrderPort legacyOrderPort;  // ACL을 통해 접근

    public OrderDetail getOrderDetail(OrderId orderId) {
        Order order = legacyOrderPort.findOrderById(orderId)
            .orElseThrow(() -> new OrderNotFoundException(orderId));
        // Order는 깨끗한 도메인 언어만 사용
        return OrderDetail.from(order);
    }
}
```

---

## 📊 설계 비교

```
Context Map 패턴 선택 기준:

패턴               | 팀 관계        | 우리 통제력 | 번역 필요 | 적합한 상황
───────────────────┼──────────────┼──────────┼─────────┼──────────────────
Shared Kernel      │ 대등, 신뢰    │ 공동 소유  │ 불필요   │ 순수 VO 공유
Customer-Supplier  │ 협력적        │ 제한적    │ 일부     │ 내부 팀 간 API
Conformist         │ 상위에 종속   │ 없음      │ 최소     │ 잘 설계된 외부 API
ACL                │ 상위에 종속   │ 없음      │ 필수     │ 레거시, 이상한 외부
Open Host Service  │ 우리가 상위   │ 완전      │ Consumer │ 여러 Consumer에 제공
Published Language │ 표준 기반     │ 표준 따름  │ 표준으로 │ 공개 API, 이벤트 스키마
Separate Ways      │ 무관계        │ 독립      │ 불필요   │ 통합 필요 없는 경우
```

---

## ⚖️ 트레이드오프

```
각 패턴의 주요 트레이드오프:

Shared Kernel:
  장점: 코드 중복 없음, 통합 단순
  단점: 변경 시 양쪽 팀 합의 필요, 결합도 높음
  → 공유 범위를 최소화할수록 좋음

Customer-Supplier:
  장점: Supplier가 Customer 요구 반영 가능
  단점: Supplier 우선순위에 Customer가 종속
  → SLA 합의, API 버전 관리 정책 필요

Conformist:
  장점: 번역 비용 없음, 빠른 통합
  단점: 외부 모델에 종속, 외부 변경 즉각 영향
  → 외부 API가 안정적이고 우리 도메인에 맞을 때만

ACL:
  장점: 도메인 보호, 외부 변경 격리
  단점: 번역 코드 유지 비용, 초기 구현 복잡
  → 외부 시스템이 자주 변경되거나 모델이 이상할 때 가치

Open Host Service + Published Language:
  장점: 여러 Consumer 지원, 표준화
  단점: API 안정성 부담, 버전 관리 필요
  → 우리 서비스가 플랫폼이 될 때 적합
```

---

## 📌 핵심 정리

```
Context Map 패턴 핵심:

7가지 패턴:
  공유 소유: Shared Kernel (대등 협력, 최소 공유)
  단방향 의존: Customer-Supplier (협력), Conformist (순응)
  외부 격리: Anticorruption Layer (번역으로 도메인 보호)
  제공자: Open Host Service + Published Language (표준 API)
  독립: Separate Ways (통합 불필요)

패턴 선택 기준:
  1. 권력 관계: 우리가 상위인가, 대등인가, 하위인가?
  2. 협력 의지: 외부가 우리 요구를 들어주는가?
  3. 외부 모델 품질: 우리 도메인에 맞는가, 오염시키는가?

팀 관계가 기술보다 먼저:
  "REST API로 할까, 이벤트로 할까?" 보다
  "이 팀과의 관계가 어떠한가?" 먼저 파악

ACL의 가치:
  레거시/외부 시스템 변경 → ACL만 수정 → 도메인 무관
  "외부가 변해도 우리 도메인은 안정적" — ACL의 핵심 가치
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 상황에서 어떤 Context Map 패턴을 선택해야 하는가?

```
A. 우리 회사가 AWS S3를 파일 저장에 사용. S3 API가 우리 도메인에 잘 맞음.
B. 외부 물류사 API 연동. 물류사는 20년 된 XML 기반 API만 제공하고 변경 불가.
C. 우리 회사 내 두 팀(주문팀, 재고팀)이 공동으로 사용하는 ProductId 클래스.
D. 우리 회사가 외부 파트너사들에게 상품 검색 API를 제공해야 함.
```

<details>
<summary>해설 보기</summary>

**A. Conformist**
S3 API가 우리 도메인에 잘 맞고, Amazon이 우리 요구사항을 반영해줄 수 없으므로 Conformist. S3의 `Bucket`, `Key`, `PutObject` 같은 개념을 그대로 사용해도 도메인 오염이 크지 않습니다.

**B. Anticorruption Layer**
20년 된 XML 기반 API는 모델이 이상하고 변경 불가. 우리 도메인 언어와 XML 구조 사이를 번역하는 ACL이 필수. `LogisticsApiAdapter` + `LogisticsOrderTranslator` 로 구현.

**C. Shared Kernel**
`ProductId`는 두 팀이 공동으로 사용하는 순수 Value Object. Shared Kernel로 관리하되, 공유 범위를 `ProductId`만으로 최소화. 변경 시 두 팀 합의 필요하다는 것을 팀이 인지해야 함.

**D. Open Host Service + Published Language**
외부 파트너사들이 통합할 수 있도록 표준화된 REST API(OpenAPI 명세) 또는 이벤트 스키마 제공. 내부 도메인 모델을 직접 노출하지 않고 `ProductSearchResponse` 같은 Published Language 형식으로 제공.

</details>

---

**Q2.** Conformist와 ACL 중 어느 것을 선택해야 하는지 판단하는 기준을 제시하세요.

<details>
<summary>해설 보기</summary>

**핵심 판단 질문:**

```
1. 외부 모델이 우리 도메인 언어와 일치하는가?
   일치하면 → Conformist (번역 불필요)
   다르면 → ACL 고려

2. 외부 모델의 "나쁜 개념"이 우리 도메인에 퍼질 위험이 있는가?
   customer_type = 'B' 같은 불명확한 코드 → ACL 필요
   잘 정의된 개념 → Conformist 가능

3. 외부 모델이 얼마나 자주 변경되는가?
   자주 변경 → ACL (변경 격리 비용 정당화)
   거의 변경 안 함 → Conformist (번역 비용 > 격리 가치)

4. 도메인 테스트 시 외부 모델이 영향을 주는가?
   도메인 테스트에 외부 SDK/구조가 필요 → ACL 필요
   외부 없이 테스트 가능 → Conformist도 괜찮음

실용적 판단:
  Conformist: AWS SDK, 잘 설계된 PG사 SDK, 표준 OpenAPI
  ACL: 레거시 시스템, 이상한 XML API, 불명확한 코드 사용 시스템
```

</details>

---

**Q3.** "내부 팀 간에도 ACL이 필요한가? Shared Kernel이나 Customer-Supplier로 충분하지 않은가?"

<details>
<summary>해설 보기</summary>

**내부 팀 간에도 ACL이 필요한 경우가 있습니다.**

**ACL이 내부에서 필요한 상황:**

1. **레거시 내부 시스템**: 오래된 모듈이 이상한 구조를 갖고 있고, 새 모듈이 이것을 사용해야 할 때.
2. **급격히 다른 도메인 모델**: 두 팀의 도메인이 근본적으로 달라서 직접 연결하면 도메인이 오염될 때.
3. **임시 통합**: 곧 재설계될 예정인 내부 모듈과 통합할 때 (재설계 후 ACL만 수정).

```java
// 내부 레거시 팀과의 ACL 예시
// 레거시 주문 시스템(이상한 구조) ← ACL → 신규 배송 시스템

// 레거시 Order 모델 (이상한 구조)
public class LegacyOrder {
    String ord_no;      // 주문번호 (언더스코어 + 약자)
    String cust_cd;     // 고객코드
    int sta_cd;         // 상태코드 (1=접수, 2=확인, 3=출고, 99=취소)
}

// ACL: 레거시 모델 → 신규 도메인 모델
@Component
public class LegacyOrderAdapter implements OrderQueryPort {
    public Optional<Order> findById(OrderId id) {
        LegacyOrder legacy = legacyRepository.findByOrdNo(id.value());
        return Optional.ofNullable(legacy).map(this::translate);
    }

    private Order translate(LegacyOrder legacy) {
        return new Order(
            new OrderId(legacy.ord_no),
            new CustomerId(legacy.cust_cd),
            translateStatus(legacy.sta_cd)
        );
    }
}
```

Customer-Supplier나 Shared Kernel로 충분한 경우: 두 팀이 협력하고 도메인 모델이 비슷할 때. ACL은 "외부 모델의 오염을 막아야 할 때"라는 필요성이 있을 때 선택합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Ubiquitous Language 실전](./03-ubiquitous-language-practice.md)** | **[홈으로 🏠](../README.md)** | **[다음: Anticorruption Layer ➡️](./05-anticorruption-layer.md)**

</div>
