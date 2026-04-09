# Ubiquitous Language 실전 — 컨텍스트마다 언어가 다를 수 있다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 같은 단어가 컨텍스트마다 다른 의미를 가질 때, 이것이 왜 설계 문제의 신호인가?
- 언어 충돌을 발견했을 때 Context 경계를 어떻게 수정하는가?
- Glossary가 단순한 용어집이 아니라 Context 경계의 문서화가 되는 이유는?
- 실제 프로젝트에서 Ubiquitous Language를 어떻게 구축하고 유지하는가?
- Context 내부의 언어 일관성과 Context 간 언어 충돌을 어떻게 구분하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Ch1에서 Ubiquitous Language의 기본 개념을 다뤘다면, 이 문서는 **Bounded Context와 Ubiquitous Language의 교차점**을 다룬다. 핵심 통찰은 이것이다: **언어 충돌은 Context 경계가 잘못됐다는 신호다.**

"고객(Customer)"이라는 단어를 영업팀과 CS팀이 다른 의미로 쓴다면, 이것은 단순한 용어 혼선이 아니라 "영업 컨텍스트"와 "CS 컨텍스트"가 아직 분리되지 않았다는 설계 신호다. 언어 충돌을 발견하고 Context 경계를 정제하는 것이 이 문서의 핵심이다.

---

## 😱 흔한 실수 (Before — 언어 충돌을 무시하고 강제로 통일)

```
상황: 병원 예약 시스템

팀 회의에서 발견된 언어 충돌:
  간호사: "환자가 예약했다" — Appointment 생성을 의미
  의사: "환자가 예약했다" — 진료 동의서에 서명까지 완료를 의미
  원무팀: "환자가 예약했다" — 보험 적용 및 수납 코드 등록까지 완료를 의미

개발팀의 잘못된 대응:
  "용어를 통일합시다! '예약 완료'는 원무팀 기준으로 합시다"
  → Appointment.status = "CONFIRMED" (원무팀 기준으로 통일)

결과:
  의사 화면: "예약 완료"된 환자 목록 조회
    → 실제로는 동의서 서명 안 한 환자도 포함
    → 의사: "동의서 서명 안 한 환자는 진료하면 안 되는데?"
  
  간호사 화면: "예약 완료"된 환자 접수
    → 보험 코드 등록 안 된 환자를 접수
    → 수납 오류 발생

  근본 원인:
    세 팀의 "예약 완료"는 서로 다른 비즈니스 단계를 의미했음
    강제 통일이 각 팀의 도메인 규칙을 훼손함
```

---

## ✨ 올바른 접근 (After — 언어 충돌을 Context 경계 재설계로 해결)

```
언어 충돌 분석:

"예약"이라는 단어의 의미 충돌 지점:
  ┌──────────────────────────────────────────────────┐
  │ 환자가 날짜/시간 선택 → 예약 요청               │
  │ ↓                                              │
  │ 간호사가 확인 → "예약 접수" (간호사의 완료 기준)  │
  │ ↓                                              │
  │ 환자가 동의서 서명 → "예약 확정" (의사의 기준)    │
  │ ↓                                              │
  │ 원무팀 보험 코드 등록 → "예약 완료" (원무팀 기준) │
  └──────────────────────────────────────────────────┘

언어 충돌 → 서로 다른 Context를 나타냄:
  예약 Context (Appointment Context):
    "예약"의 의미: 환자가 날짜/시간을 선택하고 간호사가 확인한 상태
    핵심 언어: 예약(Appointment), 접수(Accept), 내원(Arrive), 노쇼(NoShow)
  
  진료 동의 Context (Consent Context):
    "확정"의 의미: 환자가 진료 동의서에 서명 완료
    핵심 언어: 동의서(ConsentForm), 서명(Sign), 철회(Revoke)
  
  수납/보험 Context (Billing Context):
    "완료"의 의미: 보험 코드 등록, 수납 준비 완료
    핵심 언어: 청구(Bill), 보험 청구(InsuranceClaim), 수납(Payment)
```

```java
// 언어 충돌을 해소한 각 Context의 모델

// 예약 Context — 간호사의 언어
package com.hospital.appointment.domain;

public class Appointment {
    private AppointmentStatus status;

    public void accept() {  // 간호사가 "접수했다"
        this.status = AppointmentStatus.ACCEPTED;
        // 이 시점이 간호사의 "예약 완료"
    }
    
    public void arrive() {  // 환자가 "내원했다"
        this.status = AppointmentStatus.ARRIVED;
    }
}

// 진료 동의 Context — 의사의 언어
package com.hospital.consent.domain;

public class ConsentForm {
    private ConsentStatus status;
    private AppointmentId appointmentId;  // Appointment를 ID로만 참조

    public void sign(PatientSignature signature) {  // 환자가 "서명했다"
        this.status = ConsentStatus.SIGNED;
        this.events.add(new ConsentSigned(this.appointmentId, signature));
        // 이 시점이 의사의 "예약 완료"
    }
}

// 수납 Context — 원무팀의 언어
package com.hospital.billing.domain;

public class VisitBilling {
    private BillingStatus status;
    private AppointmentId appointmentId;
    private InsuranceCode insuranceCode;

    public void registerInsurance(InsuranceCode code) {  // "보험 등록했다"
        this.insuranceCode = code;
        this.status = BillingStatus.INSURANCE_REGISTERED;
        // 이 시점이 원무팀의 "예약 완료"
    }
    
    public void complete(Payment payment) {  // "수납 완료"
        this.status = BillingStatus.PAID;
    }
}
```

```
결과:
  각 팀이 자신의 언어로 자신의 Context를 정의
  언어 충돌이 해소됨 (각 Context 내에서는 용어가 명확)
  
  의사 화면: "ConsentSigned 이벤트가 있는 Appointment 목록"
  간호사 화면: "ACCEPTED 상태의 Appointment 목록"
  원무팀 화면: "INSURANCE_REGISTERED 상태의 VisitBilling 목록"
  
  → 각 화면이 자신의 Context에서 명확한 상태를 조회
  → "예약 완료"라는 모호한 통합 상태 없음
```

---

## 🔬 내부 동작 원리

### 1. 언어 충돌이 Context 경계 오류의 신호인 이유

```
언어 충돌 발견 → Context 경계 문제 진단 흐름:

Step 1: 언어 충돌 발견
  "A팀과 B팀이 같은 단어를 다른 의미로 쓴다"

Step 2: 충돌 분석
  같은 개념인가, 다른 개념인가?
  
  Case A: 사실 같은 개념 (용어만 다름)
    → Ubiquitous Language 통일 (하나의 용어로 합의)
    → Context 경계 변경 없음
    예: A팀은 "회원", B팀은 "사용자" → 합의해서 "회원"으로 통일
  
  Case B: 실제로 다른 개념 (이름만 같음)
    → Context 경계 재설정 필요
    → 각 Context에서 다른 이름을 쓰거나, 같은 이름이지만 다른 클래스
    예: "고객"이 영업팀에서는 "잠재 고객", 서비스팀에서는 "가입 고객"
        → SalesProspect (영업 Context) / Customer (서비스 Context)

Step 3: Context 경계 재설정
  "예약"의 충돌 → 예약/동의/수납 세 Context로 분리
  각 Context에서 독립적 언어 정의

언어 충돌을 통해 발견한 Context 경계가
언어 기반으로 발견한 Context 경계이므로 더 안정적:
  언어 경계 = 비즈니스 경계 = 팀 경계
  → Conway's Law와 자연스럽게 일치
```

### 2. Context별 Glossary 작성 방법

```
Glossary 구조 예시 — 예약 Context:

───────────────────────────────────────────────────
컨텍스트: 예약 Context (Appointment Context)
담당팀: 예약관리팀
─────────────────────────────────────────────────
용어 | 정의 | 코드 명칭 | 다른 Context와의 관계
─────────────────────────────────────────────────
예약 | 환자가 특정 날짜/시간에 특정 의사에게 | Appointment | 
     | 진료를 신청한 것                    |             |
─────────────────────────────────────────────────
접수 | 병원(간호사)이 예약을 확인하고 승인한 | accept()    | 
     | 상태. 이 시점부터 예약은 유효함       |             |
─────────────────────────────────────────────────
내원 | 환자가 실제로 병원에 도착한 것        | arrive()    |
─────────────────────────────────────────────────
노쇼 | 예약 시간에 환자가 내원하지 않은 것   | NoShow      |
     | 접수 상태에서 일정 시간 경과 시 처리  |             |
─────────────────────────────────────────────────

주의사항:
  "예약 완료" — 이 Context에서 사용하지 않는 용어.
  간호사가 접수한 것은 "접수", 환자가 온 것은 "내원"으로 구분.
  원무팀의 "예약 완료"는 수납 Context의 개념임.
───────────────────────────────────────────────────

---

Glossary가 Context 경계의 문서화가 되는 이유:
  1. "이 용어는 이 Context에서만 유효합니다" 명시
  2. "다른 Context와의 관계"에 통합 패턴 기록
  3. 새 개발자가 Context 경계를 Glossary로 파악
  4. "이 단어가 모호한가?" = "Context 경계가 모호한가?"

실용적 관리:
  /docs/glossary/appointment-context.md
  /docs/glossary/billing-context.md
  코드 저장소에 함께 관리 → 코드 변경 시 Glossary도 같이 리뷰
```

### 3. 컨텍스트 언어 지도 (Context Language Map)

```
전체 시스템에서 같은 단어가 각 Context에서 어떤 의미를 갖는지 시각화:

"고객 (Customer / Member / User / Patient ...)"

──────────────────────────────────────────────────────
Context           | 이 Context에서의 이름 | 의미
──────────────────────────────────────────────────────
회원 Context       | Member              | 가입한 사람, 포인트/등급 관리
마케팅 Context     | Subscriber          | 마케팅 수신 동의한 사람
주문 Context       | Customer            | 주문을 하는 사람
배송 Context       | Recipient           | 배송 받는 사람 (배송지 소유자)
CS Context         | Requester           | 문의/불만을 제기하는 사람
──────────────────────────────────────────────────────

"같은 사람"이지만 Context마다 다른 관점의 다른 이름:
  → 각 Context의 Ubiquitous Language가 다름
  → 이름이 다르면 역할과 책임이 명확해짐

이름을 억지로 통일하면:
  Member에 배송지 필드 추가 → 회원 Context가 배송 관심사를 흡수
  Member에 CS 노트 필드 추가 → 회원 Context가 CS 관심사를 흡수
  → 만능 클래스 재현
```

### 4. Ubiquitous Language 진화 관리

```
언어는 비즈니스와 함께 진화한다:

변화 예시:
  초기: "구매" → Purchase
  6개월 후: 구독 모델 도입 → "구매"가 "정기결제"와 "일시결제"로 분화
    → Purchase → OneTimePurchase / SubscriptionPurchase?
    → 또는 새 Context? SubscriptionContext 분리?

관리 원칙:
  ① 언어 변화를 코드 변경보다 먼저 팀에 공지
     PR 설명에 "이 변경은 XX라는 개념을 YY로 재명명합니다" 명시
  
  ② 이름 변경(rename refactoring)은 용기 있게 즉시
     "나중에 바꾸자" → 기술 부채로 쌓임
     IDE rename refactoring → 컴파일 타임에 누락 체크
  
  ③ Glossary를 코드 저장소에서 관리
     언어 변경 시 Glossary PR → 코드 PR
     코드 리뷰 시 "Glossary와 일치하는가?" 체크

  ④ Context 간 번역 테이블 유지
     OrderId가 Shipping Context에서는 ShipmentOrderRef로 불린다면
     이 매핑을 문서화 (번역 오류 방지)
```

---

## 💻 실전 코드

### 언어 충돌 탐지 체크리스트 코드로 표현

```java
// 안티패턴: 같은 클래스에 여러 Context의 언어가 섞임
// → 이것을 발견하면 Context 분리 신호

public class User {
    // 회원 Context
    private MembershipLevel membershipLevel;  // 회원 등급
    private LocalDate joinedAt;               // 가입일

    // 마케팅 Context — 섞임!
    private Boolean marketingConsent;         // 마케팅 동의
    private String preferredCategory;         // 선호 카테고리

    // CS Context — 섞임!
    private Integer complainCount;            // 불만 제기 횟수
    private Boolean isVip;                    // VIP 고객 (CS 기준)

    // 배송 Context — 섞임!
    private String defaultShippingAddress;    // 기본 배송지
}

// ↑ 이 클래스가 4개 Context의 언어를 가지고 있음 → 분리 필요

// 올바른 접근: 각 Context의 언어로 분리된 클래스
package com.example.member.domain;
public class Member {
    private MemberId id;
    private MembershipLevel level;
    private LocalDate joinedAt;
    // 마케팅, CS, 배송 정보 없음
}

package com.example.marketing.domain;
public class MarketingProfile {
    private MemberId memberId;          // Member를 ID로 참조
    private Boolean marketingConsent;
    private Set<Category> preferences;
}

package com.example.cs.domain;
public class CsProfile {
    private MemberId memberId;
    private List<Complaint> complaints;
    private VipStatus vipStatus;
}
```

### Context별 Glossary를 테스트로 표현

```java
// Glossary의 용어 정의를 Enum/상수로 표현
// → 코드가 문서 역할

public enum AppointmentStatus {
    REQUESTED("요청됨 - 환자가 예약 신청한 상태"),
    ACCEPTED("접수됨 - 간호사가 확인한 상태. 간호사의 '예약 완료'"),
    ARRIVED("내원함 - 환자가 실제로 병원에 온 상태"),
    IN_CONSULTATION("진료중"),
    CHARTING_DONE("차팅 완료"),
    NO_SHOW("노쇼 - 예약 시간에 오지 않은 상태"),
    CANCELLED("취소됨");

    private final String description;  // Glossary 정의가 코드에 포함됨

    AppointmentStatus(String description) {
        this.description = description;
    }

    // "이 Context에서 '예약 완료'라는 용어를 쓰지 않는 이유" 를 코드로 표현
    // CONFIRMED, COMPLETED 같은 이름이 없음 → 모호성 제거
}
```

---

## 📊 설계 비교

```
통일된 언어 강요 vs Context별 독립 언어:

                  언어 강요 통일          Context별 독립 언어
────────────────┼──────────────────┼────────────────────────────
이해 용이성      │ 처음엔 단순해 보임  │ Context 내에서는 매우 명확
────────────────┼──────────────────┼────────────────────────────
언어 충돌        │ 억지 통일          │ Context 경계로 해소
────────────────┼──────────────────┼────────────────────────────
도메인 전문가와  │ "왜 우리 용어를    │ "코드가 우리 용어와 일치해요"
소통            │ 못 써요?" 갈등     │
────────────────┼──────────────────┼────────────────────────────
버그 발생        │ 모호한 상태로 인한  │ 명확한 상태 전이로 최소화
                │ 로직 오류          │
────────────────┼──────────────────┼────────────────────────────
Context 분리     │ 불가 (언어가 얽힘) │ 자연스럽게 가능
────────────────┼──────────────────┼────────────────────────────
신규 팀원 온보딩  │ "이 상태가 팀마다  │ Context 내에서 명확한 언어
                │ 다른 의미?" 혼란  │
```

---

## ⚖️ 트레이드오프

```
Context별 독립 언어의 어려움:

① Context 간 번역 비용
   "Order Context의 Order와 Shipping Context의 Shipment가 어떻게 연결?"
   → 번역 테이블/Glossary 유지 비용
   → 신규 개발자가 전체 Context 언어 지도를 이해하는 데 시간 필요

② 언어 충돌이 Context 오류 신호인지 판단의 어려움
   모든 언어 충돌이 Context 문제는 아님
   단순한 용어 혼선 → 합의로 해결
   실제 다른 개념 → Context 재설계
   → 판단이 틀리면 불필요한 Context 분리

③ 완전히 독립적인 언어 관리의 현실적 어려움
   소규모 팀에서 Context별 Glossary를 완벽히 유지하기 어려움
   → 최소한 "이 Context에서 쓰지 않는 용어" 목록만 관리해도 효과

현실적 접근:
  ① 핵심 용어 10~20개부터 Glossary 시작
  ② "이 단어가 헷갈린다"고 누군가 말하면 → 즉시 Glossary 업데이트
  ③ 코드 리뷰에서 "이 이름이 Glossary와 맞는가?" 확인을 습관화
```

---

## 📌 핵심 정리

```
Ubiquitous Language 실전 핵심:

언어 충돌 = Context 경계 문제의 신호:
  같은 단어가 팀마다 다른 의미 → 같은 Context에 섞여 있는 것
  → 언어 충돌 지점을 따라 Context 경계를 분리

Context별 독립 언어:
  Context A: Customer (구매하는 사람)
  Context B: Recipient (배송 받는 사람)
  같은 사람이지만 역할에 따라 다른 이름
  → 이름이 역할을 명확히 함

Glossary = Context 경계의 문서화:
  각 Glossary는 하나의 Context를 정의
  "이 Context에서 쓰지 않는 용어"도 명시 (모호성 제거)
  코드와 함께 버전 관리

언어 진화 관리:
  비즈니스 변화 → 언어 변화 → 코드 이름 변화
  rename refactoring을 두려워하지 말 것
  Glossary 변경 → 코드 변경 순서로 진행

실용적 체크리스트:
  □ 팀이 같은 단어를 다른 의미로 쓰지 않는가?
  □ 코드 이름이 도메인 전문가의 언어와 일치하는가?
  □ 한 클래스에 여러 Context의 언어가 섞이지 않았는가?
  □ 새 기능을 논의할 때 도메인 전문가와 같은 단어를 쓰는가?
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 언어 충돌을 찾고, 어느 Context를 분리해야 하는지 제안하세요.

```java
public class Product {
    private Long id;
    private String name;
    private BigDecimal price;

    // 재고 관련 (창고팀 언어)
    private int stockQuantity;
    private String warehouseLocation;
    private LocalDate lastRestockedAt;

    // 전시 관련 (MD/기획 언어)
    private Boolean isDisplayed;
    private Integer displayOrder;
    private String promotionBadge;    // "신상", "베스트", "할인"

    // 상품 리뷰 관련 (고객 언어)
    private Double averageRating;
    private Integer reviewCount;
}
```

<details>
<summary>해설 보기</summary>

**세 가지 Context의 언어가 섞여 있습니다.**

```
Product 클래스에 섞인 언어:
  상품 Context (기본): id, name, price
  재고 Context (창고팀 언어): stockQuantity, warehouseLocation, lastRestockedAt
  전시 Context (MD 언어): isDisplayed, displayOrder, promotionBadge
  리뷰 Context (고객 언어): averageRating, reviewCount
```

**분리 제안:**

```java
// 상품 Context (카탈로그 중심)
package com.example.catalog.domain;
public class Product {
    private ProductId id;
    private String name;
    private Money price;
    private ProductCategory category;
    // 재고, 전시, 리뷰 정보 없음
}

// 재고 Context
package com.example.inventory.domain;
public class ProductInventory {
    private ProductId productId;    // Product를 ID로만 참조
    private int quantity;
    private WarehouseLocation location;
    private LocalDate lastRestockedAt;

    // 재고 Context의 언어
    public boolean isOutOfStock() { return quantity == 0; }
    public void restock(int amount) { ... }
}

// 전시 Context
package com.example.display.domain;
public class ProductDisplay {
    private ProductId productId;
    private boolean displayed;
    private int displayOrder;
    private PromotionBadge badge;

    // 전시 Context의 언어
    public void feature(PromotionBadge badge) { ... }
    public void hide() { ... }
}
```

이벤트로 연결: `ProductPriceChanged` → 전시 Context가 수신해서 표시 가격 업데이트.

</details>

---

**Q2.** Glossary를 관리하는 시간이 없다는 팀에게 최소한의 실용적인 방법을 제안하세요.

<details>
<summary>해설 보기</summary>

**"완벽한 Glossary"보다 "살아있는 최소 Glossary"가 훨씬 낫습니다.**

**최소 Glossary 운영법:**

```markdown
# /docs/glossary.md — 최소 버전

## ⚠️ 이 Context에서 쓰지 않는 용어 (혼동 방지)
- "예약 완료" → 쓰지 않음. "접수(ACCEPTED)" 또는 "내원(ARRIVED)"를 명확히 사용
- "User" → 쓰지 않음. "Member" (회원), "Customer" (주문 맥락)로 구분

## ✅ 핵심 용어 (5개 이내)
| 도메인 용어 | 코드 이름 | 한 줄 정의 |
|-----------|---------|---------|
| 접수 | ACCEPTED (enum) | 간호사가 예약을 확인한 상태 |
| 내원 | ARRIVED (enum) | 환자가 병원에 실제로 온 상태 |
```

**실용적 운영 루틴:**
1. 코드 리뷰 시 새 클래스/메서드 이름이 Glossary와 다르면 코멘트
2. 팀 회의에서 "이 단어가 헷갈린다"는 말이 나오면 즉시 Glossary 업데이트 (5분)
3. 스프린트마다 Glossary 1개 항목 추가 (부담 없는 주기)

가장 중요한 것: **"이 Context에서 쓰지 않는 용어" 목록** 하나만 유지해도 언어 충돌의 80%를 방지합니다.

</details>

---

**Q3.** 마이크로서비스로 분리된 상황에서, A 서비스의 `OrderId`가 B 서비스에서는 `ReferenceOrderId`라고 불린다. 이 번역을 어떻게 명시적으로 관리해야 하는가?

<details>
<summary>해설 보기</summary>

**번역 매핑을 ACL(Anti-Corruption Layer)에서 명시적으로 처리합니다.**

```java
// B 서비스 (Shipping Context)의 ACL — A 서비스 이벤트를 수신
@Component
public class OrderEventTranslator {

    // A 서비스 이벤트 (주문 Context 언어)
    public Shipment translateFrom(OrderPlaced orderEvent) {
        // 명시적 번역: A의 orderId → B의 referenceOrderId
        return Shipment.builder()
            .referenceOrderId(ShipmentOrderRef.from(orderEvent.orderId()))
            // ^ 번역이 일어나는 지점이 명확
            .build();
    }
}

// 번역 테이블 문서화 (/docs/translation/order-shipping.md)
// | A Context (Order) | B Context (Shipping) | 설명 |
// |------------------|---------------------|------|
// | OrderId           | ShipmentOrderRef    | 배송이 참조하는 주문 ID |
// | OrderLine.productId | PackageItem.productCode | 배송 패키지의 상품 코드 |
```

이 번역을 묵시적(암묵적)으로 처리하면 나중에 두 Context의 개념이 바뀔 때 번역 오류가 숨겨집니다. 명시적 Translator 클래스를 두면 번역 로직이 한 곳에 집중되고, 변경 시 영향 범위를 파악하기 쉽습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Bounded Context 완전 분해](./02-bounded-context.md)** | **[홈으로 🏠](../README.md)** | **[다음: Context Map 패턴 완전 분해 ➡️](./04-context-map-patterns.md)**

</div>
