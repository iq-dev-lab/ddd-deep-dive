# Strategic Design vs Tactical Design — 큰 그림부터 시작하는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Strategic Design과 Tactical Design은 각각 무엇을 다루는가?
- 왜 Tactical Design(Aggregate, Entity, VO)보다 Strategic Design(Bounded Context, Context Map)을 먼저 해야 하는가?
- Context Map 없이 Aggregate 설계를 시작하면 어떤 문제가 발생하는가?
- DDD의 전략적 도구(Subdomain, Bounded Context)와 전술적 도구(Aggregate, Repository)의 관계는?
- 언제 Strategic Design이 필요하고, 언제 Tactical Design으로 바로 시작해도 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

DDD를 공부하면 보통 Aggregate, Entity, Value Object부터 배운다. 이것들이 코드로 바로 이어지기 때문에 더 흥미롭고 구체적으로 느껴진다. 그런데 실제 프로젝트에서 Aggregate를 먼저 설계하기 시작하면, 3개월 후에 "이 Aggregate 경계가 잘못됐다"를 깨닫고 전면 재설계하는 상황을 맞닥뜨리기 쉽다.

Strategic Design이 없으면 **어느 팀이 어느 코드를 소유하는가**, **서비스 간 경계가 어디인가**, **이 Aggregate가 어느 컨텍스트에 속하는가**를 알 수 없다. 건축으로 비유하면, 각 방의 인테리어(Tactical)를 먼저 결정하기 전에 건물의 구조와 층수(Strategic)를 먼저 결정해야 하는 것과 같다.

---

## 😱 흔한 실수 (Before — Strategic Design 없이 Tactical부터 시작)

```
상황: 물류 SaaS 서비스 개발

전술부터 시작한 팀의 접근:
  "주문이 있으니 Order Aggregate"
  "배송이 있으니 Delivery Aggregate"
  "결제가 있으니 Payment Aggregate"
  "창고가 있으니 Warehouse Aggregate"
  
  → 4개 Aggregate를 가진 하나의 모듈로 시작
```

```java
// Strategic Design 없이 만들어진 Aggregate
public class Order {
    private Long id;
    private List<OrderItem> items;
    private Delivery delivery;     // ← 문제: Order가 Delivery를 직접 포함
    private Payment payment;       // ← 문제: Order가 Payment를 직접 포함
    private Warehouse warehouse;   // ← 문제: Order가 Warehouse를 직접 포함

    // Order가 배송, 결제, 창고 모두를 알고 있음
    public void confirmDelivery() {
        this.delivery.setStatus("CONFIRMED");
        this.warehouse.decreaseStock(this.items);
        this.payment.capture();
    }
}

// 3개월 후 요구사항: "배송 파트너사를 외부 API로 교체"
// → Order가 Delivery를 직접 포함하므로 Order까지 수정
// → Payment도 얽혀있으므로 결제 로직도 재검토
// → 테스트 범위: Order 전체 (배송 + 결제 + 창고까지)

// 6개월 후 요구사항: "결제를 별도 팀이 관리하는 서비스로 분리"
// → Order.payment 필드 제거 + Order.confirmDelivery() 재설계
// → 이미 Order를 참조하는 Delivery, Payment가 얽혀 분리 불가
```

```
Strategic Design 없이 시작한 결과:
  Order: 500줄 (배송, 결제, 창고 로직 혼재)
  팀 A: "Order 건드리려면 팀 B, C, D 모두 알려야 함"
  팀 B: "Payment 수정하면 Order가 영향받아 팀 A 확인 필요"
  
  → 3개 팀이 하나의 클래스를 공유하는 "분산 모놀리스"
  → Aggregate 경계가 없는 것과 다름없음
```

---

## ✨ 올바른 접근 (After — Strategic Design 먼저)

```
Strategic Design 단계:

Step 1: 서브도메인 식별
  Core Domain:    주문 처리 (경쟁 우위의 핵심)
  Supporting:     창고 관리, 배송 관리
  Generic:        결제 처리 (외부 PG사 사용)

Step 2: Bounded Context 경계 설정
  주문 컨텍스트 (Order Context)    → 팀 A 소유
  배송 컨텍스트 (Shipping Context)  → 팀 B 소유
  결제 컨텍스트 (Payment Context)   → 외부 PG사 (ACL로 분리)
  창고 컨텍스트 (Inventory Context) → 팀 C 소유

Step 3: Context Map 작성
  Order → Shipping: Customer-Supplier
    (Order가 Customer, Shipping이 Supplier)
    Order가 배송 요청, Shipping이 배송 처리
  
  Order → Payment: Anticorruption Layer
    (외부 PG사 모델이 Order 도메인을 오염시키지 않도록)
  
  Order → Inventory: Published Language
    (재고 차감 이벤트로 통신, 계약된 이벤트 포맷)
```

```java
// Strategic Design 이후의 Aggregate — 경계가 명확함

// 주문 컨텍스트: Order는 배송/결제/창고를 직접 포함하지 않음
public class Order {  // 주문 컨텍스트의 Aggregate Root
    private OrderId id;
    private CustomerId customerId;
    private List<OrderLine> lines;
    private OrderStatus status;
    // Delivery, Payment, Warehouse를 직접 포함하지 않음
    // 참조는 ID로만

    public void place() {
        this.status = OrderStatus.PLACED;
        this.events.add(new OrderPlaced(this.id, this.lines));
        // OrderPlaced 이벤트 → Shipping Context가 수신
        // OrderPlaced 이벤트 → Inventory Context가 수신
        // OrderPlaced 이벤트 → Payment Context가 수신
    }
}

// 배송 컨텍스트: Shipment는 Order를 직접 참조하지 않음
public class Shipment {  // 배송 컨텍스트의 Aggregate Root
    private ShipmentId id;
    private OrderId orderId;      // Order ID로만 참조 (객체 직접 참조 ❌)
    private ShippingAddress address;
    private CarrierId carrierId;
    private ShipmentStatus status;

    // OrderPlaced 이벤트를 수신해서 생성됨
    public static Shipment createFrom(OrderPlaced event, ShippingAddress address) {
        return new Shipment(
            new ShipmentId(),
            event.orderId(),
            address,
            ShipmentStatus.PREPARING
        );
    }
}
```

```
결과:
  "배송 파트너사 외부 API 교체" → Shipping Context만 수정
  "결제 서비스 분리" → Payment Context만 분리, Order 무관
  팀 A: Order Context만 관리 → 다른 팀 승인 없이 배포 가능
  팀 B: Shipping Context만 관리 → 독립적 배포 가능
```

---

## 🔬 내부 동작 원리

### 1. Strategic Design과 Tactical Design의 범위

```
DDD 도구 지도:

Strategic Design (전략 수준)
  목적: "어떤 소프트웨어를 만들 것인가" 를 결정
  대상: 팀, 조직, 시스템 경계
  ┌────────────────────────────────────────────┐
  │ • Subdomain (서브도메인)                    │
  │   Core / Supporting / Generic 분류          │
  │                                            │
  │ • Bounded Context (경계된 컨텍스트)          │
  │   도메인 모델의 경계, 팀의 소유권 경계         │
  │                                            │
  │ • Context Map (컨텍스트 맵)                 │
  │   컨텍스트 간 관계: ACL, Shared Kernel 등   │
  │                                            │
  │ • Ubiquitous Language (유비쿼터스 언어)      │
  │   각 컨텍스트 내의 공유 언어                 │
  └────────────────────────────────────────────┘
                      ↓
Tactical Design (전술 수준)
  목적: "하나의 Bounded Context를 어떻게 구현할 것인가" 를 결정
  대상: 클래스, 메서드, 패키지 구조
  ┌────────────────────────────────────────────┐
  │ • Aggregate (애그리거트)                    │
  │   일관성 경계, 트랜잭션 단위                  │
  │                                            │
  │ • Entity / Value Object                   │
  │   식별자 vs 값으로 구분되는 도메인 객체        │
  │                                            │
  │ • Repository (리포지토리)                   │
  │   Aggregate 단위 데이터 접근                │
  │                                            │
  │ • Domain Service (도메인 서비스)            │
  │   단일 Entity로 표현 불가한 도메인 로직       │
  │                                            │
  │ • Domain Event (도메인 이벤트)              │
  │   컨텍스트 간 결합도를 낮추는 통신 수단        │
  └────────────────────────────────────────────┘

관계: Strategic ⊃ Tactical
  하나의 Bounded Context 안에 여러 Aggregate가 존재
  Bounded Context 경계가 정해지지 않으면
  Aggregate 경계도 정할 수 없음
```

### 2. Strategic Design을 먼저 해야 하는 이유

```
Aggregate 경계는 Bounded Context 경계에 의존한다:

잘못된 순서:
  Order Aggregate 설계 → Payment 포함? → 포함
  → 나중에 Payment Context 분리 결정
  → Order Aggregate 전면 재설계 필요
  → 이미 Order.payment에 의존하는 코드가 100곳

올바른 순서:
  Context Map 작성 → Order Context ≠ Payment Context
  → Order는 PaymentId만 참조
  → Payment Context 분리 시 Order는 무관

Context Map이 결정하는 것들:
  ① 어느 팀이 어느 코드를 소유하는가
     → Order Context: 팀 A / Payment Context: 외부
  
  ② 컨텍스트 간 통신 방식
     → Shared Kernel: 공유 코드 직접 참조
     → ACL: 번역 레이어 통해 접근
     → Event: 이벤트 기반 비동기 통신
  
  ③ 변경 전파 범위
     → Context 경계 = 변경 전파 경계
     → Shipping Context 변경 → Order Context 무관

Tactical Design 없이 Strategic만으로는 부족:
  Context Map은 "무엇"을 분리할지 알려줌
  Aggregate는 "어떻게" 일관성을 보장할지 알려줌
  두 레벨이 함께 동작해야 DDD가 완성
```

### 3. 각 전략적 도구의 역할

```
Subdomain vs Bounded Context:
  헷갈리기 쉬운 개념 — 다른 것임

  Subdomain (서브도메인):
    비즈니스 도메인의 분류 (발견하는 것)
    비즈니스의 문제 공간
    예: "주문 관리", "배송 관리", "결제 처리"
    Core / Supporting / Generic으로 분류

  Bounded Context (경계된 컨텍스트):
    소프트웨어 모델의 경계 (설계하는 것)
    소프트웨어의 해결 공간
    예: OrderContext, ShippingContext, PaymentContext
    팀이 소유하고 구현함

  관계:
    이상적으로는 1 Subdomain : 1 Bounded Context
    현실에서는 다를 수 있음:
      1 Subdomain에 2 Bounded Context (복잡도 분리)
      2 Subdomain이 1 Bounded Context 공유 (레거시 시스템)

Context Map 패턴:
  Shared Kernel: 두 Context가 공유 코드를 함께 소유
    → 변경 시 양쪽 팀이 합의 필요
    → 밀접한 팀에만 적합

  Customer-Supplier: 한 쪽(Supplier)이 API를 제공
    → Customer는 Supplier API에 맞춰 개발
    → Supplier가 변경 시 Customer에 영향

  Anticorruption Layer: 외부 모델의 침투 차단
    → 레거시/외부 API와 연동 시
    → 번역 레이어가 도메인 모델을 보호

  Published Language: 공개된 표준 형식으로 통신
    → 이벤트 스키마, OpenAPI 명세
    → 불특정 다수가 통합 가능
```

### 4. 언제 Strategic, 언제 Tactical인가

```
Strategic Design이 꼭 필요한 상황:
  ① 여러 팀이 하나의 시스템을 나눠 개발
     → 팀 간 경계 = Bounded Context 경계
  
  ② 마이크로서비스 분리를 고려 중
     → Context 경계가 정해지지 않으면 분리 실패
  
  ③ 레거시 시스템에 새 기능 추가
     → 레거시가 어느 Context에 속하는지 파악 필요

  ④ 여러 서브시스템 간 통합이 복잡
     → Context Map으로 통합 패턴 결정

Tactical Design만으로 충분한 상황:
  ① 단일 팀의 단일 Context 구현
     → Aggregate, Repository, Domain Service 설계
  
  ② 이미 Context 경계가 명확한 상태
     → 각 Context의 내부 구현에 집중

현실적 접근:
  작은 팀 + 단일 서비스 → Tactical부터, Strategic은 큰 결정 때만
  여러 팀 + 여러 서비스 → Strategic 먼저 (필수)
  
  최소한: "이 클래스가 어느 Context에 속하는가?"
  를 항상 의식하는 것만으로도 Strategic 사고의 시작
```

---

## 💻 실전 코드

### Context Map이 코드 패키지 구조를 결정한다

```
Context Map 결정:
  주문 컨텍스트 (Order Context)   → com.example.order
  배송 컨텍스트 (Shipping Context) → com.example.shipping
  결제 컨텍스트 (Payment Context)  → com.example.payment (ACL)

패키지 구조:
  com.example.order/
    ├── domain/
    │   ├── Order.java           (Aggregate Root)
    │   ├── OrderLine.java       (Entity)
    │   ├── OrderStatus.java     (Value Object / Enum)
    │   ├── OrderId.java         (Value Object)
    │   └── OrderRepository.java (Repository Interface)
    ├── application/
    │   └── OrderApplicationService.java
    └── infrastructure/
        ├── JpaOrderRepository.java
        └── payment/
            └── PaymentServiceAdapter.java  (ACL — Payment Context 번역)

  com.example.shipping/
    ├── domain/
    │   ├── Shipment.java
    │   └── ShipmentRepository.java
    └── application/
        └── ShipmentApplicationService.java

컨텍스트 간 통신 — 이벤트로만:
  // Order Context에서 발행
  public record OrderPlaced(
      OrderId orderId,
      List<OrderLineDto> lines,    // 배송 Context에 필요한 정보
      ShippingAddress address
  ) implements DomainEvent {}

  // Shipping Context에서 구독
  @Component
  public class OrderPlacedHandler {
      @EventListener
      public void handle(OrderPlaced event) {
          Shipment shipment = Shipment.createFrom(event);
          shipmentRepository.save(shipment);
      }
  }
```

### ArchUnit으로 Context 경계 강제

```java
// 컨텍스트 간 직접 의존 금지를 테스트로 강제
@AnalyzeClasses(packages = "com.example")
class ContextBoundaryTest {

    @ArchTest
    static ArchRule orderContextShouldNotDependOnShippingDomain =
        noClasses()
            .that().resideInAPackage("..order.domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..shipping.domain..")
            .because("Order Context는 Shipping Context의 도메인을 직접 참조할 수 없습니다. " +
                     "이벤트 또는 ACL을 통해 통신하세요.");

    @ArchTest
    static ArchRule shippingContextShouldNotDependOnPaymentDomain =
        noClasses()
            .that().resideInAPackage("..shipping.domain..")
            .should().dependOnClassesThat()
            .resideInAPackage("..payment.domain..");
}
```

---

## 📊 설계 비교

```
Strategic 없이 시작 vs Strategic 먼저 시작:

                Strategic 없이            Strategic 먼저
─────────────┼──────────────────────┼──────────────────────────
초기 개발속도  │ 빠름                  │ 느림 (설계 투자 필요)
─────────────┼──────────────────────┼──────────────────────────
3개월 후      │ Aggregate 경계 오류   │ Context 경계가 가이드
             │ 발견, 재설계 필요      │ Aggregate 설계가 명확
─────────────┼──────────────────────┼──────────────────────────
팀 분리       │ 코드 소유권 불명확    │ Context = 팀 소유 경계
─────────────┼──────────────────────┼──────────────────────────
서비스 분리   │ Aggregate 얽혀 분리   │ Context 경계 따라 분리
             │ 매우 어려움            │ 비교적 명확
─────────────┼──────────────────────┼──────────────────────────
변경 전파     │ 하나 바꾸면 다른 곳   │ Context 경계 내부로
             │ 여러 곳 영향          │ 변경 격리
─────────────┼──────────────────────┼──────────────────────────
신규 개발자   │ "이 코드가 어느 팀    │ 패키지 구조 보면
온보딩        │ 것인가?" 불명확       │ Context 소유권 명확
```

---

## ⚖️ 트레이드오프

```
Strategic Design의 비용:
  ① 시간: Context Map 작성, 팀 간 합의에 시간 소요
  ② 전문성: 도메인 전문가, 기획자, 개발자 모두의 참여 필요
  ③ 불확실성: 처음에 Context 경계를 잘못 그을 수 있음
     → Context 재조정 비용이 발생

Context 경계 재조정 비용을 줄이는 방법:
  ① 작게 시작: 완벽한 Context Map보다 핵심 3~4개 Context 먼저
  ② 이벤트로 통신: 직접 참조 대신 이벤트 → 경계 조정 시 영향 최소화
  ③ 패키지로 먼저: 물리적 서비스 분리 전, 패키지로 논리적 경계 먼저
     com.example.order / com.example.shipping 분리
     → 실제 마이크로서비스 분리는 나중에

Strategic Design이 불필요한 경우:
  단일 팀, 단일 서비스, 단순한 도메인
  → Context는 하나, Tactical Design에 집중
  → Aggregate, Repository, Domain Service로 충분
```

---

## 📌 핵심 정리

```
Strategic vs Tactical 요약:

Strategic Design (큰 그림):
  무엇을 → Subdomain 식별 (Core/Supporting/Generic)
  어디에 → Bounded Context 경계 결정
  어떻게 연결 → Context Map 작성 (ACL, Shared Kernel, Event)
  
  결과물: Context 경계 + 팀 소유권 + 통합 패턴

Tactical Design (구현):
  각 Context 내부:
    Aggregate (일관성 경계) + Entity/VO
    Repository (Aggregate 단위 접근)
    Domain Service (Aggregate로 표현 불가 로직)
    Domain Event (Context 내부/간 통신)

왜 Strategic 먼저:
  Aggregate 경계 ⊂ Bounded Context 경계
  Context를 모르면 Aggregate 경계를 잘못 설계
  → 나중에 Context를 나누려면 Aggregate 전면 재설계

현실적 순서:
  1. Core Domain 식별 (어디에 집중할 것인가)
  2. 대략적 Context 경계 (3~5개)
  3. Context Map (통합 패턴)
  4. 각 Context의 Tactical Design
  5. 경계 재조정 (점진적)
```

---

## 🤔 생각해볼 문제

**Q1.** "우리 팀은 2명이고 시스템도 단순한데, Context Map이 정말 필요한가?" 라는 질문에 어떻게 답하겠는가?

<details>
<summary>해설 보기</summary>

**2명 팀 + 단순 시스템이라면 Context Map이 필수는 아닙니다.**

하지만 최소한 이것만 의식하세요:

```
"이 클래스가 어느 Context에 속하는가?"

예:
  Order 관련 → order 패키지
  Shipping 관련 → shipping 패키지
  이 두 패키지는 서로 직접 참조하지 않음
```

2명이 한 달 만에 만든 것도 2년 후에는 5명이 유지보수합니다. 패키지 수준의 Context 분리(`com.example.order`, `com.example.shipping`)만 해도 나중에 팀이 커졌을 때 경계를 찾는 비용이 크게 줄어듭니다.

**판단 기준**: "이 클래스를 바꾸면 어느 다른 클래스가 영향받는가?" — 영향 범위가 예측 불가능하다면, 지금 당장 Strategic Design이 필요한 신호입니다.

</details>

---

**Q2.** 다음 상황에서 Strategic Design의 문제점을 찾아보세요. Context Map을 어떻게 수정해야 하는가?

```
현재 설계:
  Order Context ──(Shared Kernel)──> Payment Context
  
  공유하는 코드:
    Money.java (금액 Value Object)
    Currency.java (통화 열거형)
    PaymentStatus.java (결제 상태)

6개월 후 문제:
  Payment 팀이 PaymentStatus에 새 상태 "PARTIAL_REFUND" 추가 필요
  → Order 팀이 "그럼 Order 쪽 코드도 영향받는데?" 우려
  → 양쪽 팀이 합의하는 데 3주 소요
```

<details>
<summary>해설 보기</summary>

**문제**: Shared Kernel은 두 팀이 함께 소유하므로, 변경 시 항상 양쪽 합의가 필요합니다. `PaymentStatus`는 Payment Context의 개념인데 Order Context가 함께 소유하는 것이 문제입니다.

**개선 방향:**

```
수정된 Context Map:
  Order Context ──(Customer-Supplier or ACL)──> Payment Context

Money.java, Currency.java:
  → 공통 라이브러리로 분리 (진짜 공통 Value Object)
  → 양쪽 Context가 독립적으로 포함

PaymentStatus.java:
  → Payment Context만 소유
  → Order Context는 PaymentId만 참조
  → 결제 상태 변경 시 PaymentStatusChanged 이벤트 발행
  → Order Context는 이 이벤트를 구독

결과:
  Payment 팀이 PARTIAL_REFUND 추가
  → Order 팀에 합의 요청 없음
  → Order Context는 이벤트만 구독 (필요 시 처리 추가)
```

Shared Kernel은 **정말로 두 팀이 함께 진화시켜야 하는 코드**에만 적용해야 합니다. `Money`, `Currency` 같은 순수 Value Object가 적합하고, 상태 열거형은 소유권을 명확히 해야 합니다.

</details>

---

**Q3.** Strategic Design과 Tactical Design을 동시에 진행하는 것이 가능한가? 순서를 엄격하게 지켜야 하는가?

<details>
<summary>해설 보기</summary>

**엄격한 순서는 필요 없지만, 방향성은 지켜야 합니다.**

현실적인 접근:

```
1단계 (Strategic 스케치):
   핵심 Context 3~5개 식별 (1~2시간)
   대략적 Context Map (통합 패턴만)
   
2단계 (Tactical 시작):
   Core Domain의 핵심 Aggregate 설계
   구현하면서 도메인 지식 획득
   
3단계 (Strategic 조정):
   구현 중 발견한 문제로 Context 경계 재조정
   → Aggregate가 두 Context에 걸쳐 있으면 Context 재분리
   → 이벤트가 너무 복잡하면 Context 합치기

반복:
  Strategic ↔ Tactical 을 계속 오가는 것이 정상
  완벽한 Strategic Design을 먼저 완성하려 하지 말 것
  → 구현하면서 도메인에 대한 이해가 깊어짐
  → 이해가 깊어지면 더 나은 경계를 발견함
```

Eric Evans도 말했습니다: "도메인 모델은 지식 탐구의 결과물이지, 선행 설계의 산물이 아니다."

</details>

---

<div align="center">

**[⬅️ 이전: Ubiquitous Language](./02-ubiquitous-language.md)** | **[홈으로 🏠](../README.md)** | **[다음: DDD 적용 판단 기준 ➡️](./04-when-to-apply-ddd.md)**

</div>
