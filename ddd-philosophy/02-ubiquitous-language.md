# Ubiquitous Language — 언어가 코드 구조를 결정한다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Ubiquitous Language란 무엇이고, 왜 개발자와 도메인 전문가가 같은 언어를 써야 하는가?
- 언어의 불일치가 어떻게 실제 버그와 잘못된 설계로 이어지는가?
- 코드에서 Ubiquitous Language가 무너진 징후는 어떻게 찾아내는가?
- Glossary를 어떻게 관리하고, 코드 명명과 어떻게 일치시키는가?
- Bounded Context마다 같은 단어가 다른 의미를 가질 수 있는 이유는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

개발자가 "예약"이라고 부르고 운영팀이 "접수"라고 부르며 CS팀이 "신청"이라고 부르는 상황에서, 코드의 클래스 이름이 `Reservation`일 때 어떤 일이 벌어지는가? 세 팀이 같은 기능을 논의하는데 서로 다른 단어를 쓴다. 회의록이 달라지고, 코드 리뷰가 달라지고, QA 시나리오가 달라진다. 그리고 어느 날 "예약 취소 시 접수 상태가 변경되지 않는 버그"가 보고된다 — 사실 이것은 버그가 아니라, 언어가 달랐기 때문에 같은 개념을 두 번 구현한 것이었다.

Ubiquitous Language는 개발자, 도메인 전문가, 기획자, QA가 **하나의 공유된 언어**로 소통하게 만드는 DDD의 핵심 도구다. 이 언어가 코드에 그대로 반영될 때, 코드는 비즈니스의 명세가 된다.

---

## 😱 흔한 실수 (Before — 언어가 흩어진 코드베이스)

```
실제 상황: 헬스케어 예약 시스템

도메인 전문가(의사, 간호사)의 언어:
  "진료 예약"을 "예약" (appointment)으로 부름
  환자가 오면 "접수"
  진료가 완료되면 "완료" 또는 "차팅 완료"
  환자가 안 오면 "노쇼"

개발자가 만든 코드:
```

```java
// 개발자는 "일반적인" 용어를 씀
public class Reservation {       // 도메인: "예약"? "접수"? 뭐가 다른지 모름
    private String state;        // "PENDING", "CONFIRMED", "DONE", "ABSENT"
                                 // 도메인 전문가와 한 번도 검증 안 됨
    private Date reservedAt;     // 예약한 시각? 접수한 시각?
    private boolean isCheckedIn; // "체크인"? 도메인에서 쓰는 말인가?
}

@Service
public class ReservationService {
    public void confirm(Long id) { ... }   // 도메인에서 "확정"이란 개념이 있나?
    public void checkIn(Long id) { ... }   // "체크인"? 도메인에서는 "접수"
    public void complete(Long id) { ... }  // 도메인: "차팅 완료"와 동일?
    public void markAbsent(Long id) { ... } // 도메인: "노쇼" 처리
}
```

```
6개월 후 버그 보고:
  "예약 확정했는데 접수 처리가 안 됐다고 합니다"
  
  개발자: "'확정'은 CONFIRMED 상태로 바꾸는 거고,
           '접수'는 checkIn()을 호출하는 거 아닌가요?"
  
  간호사: "아뇨, 저희는 예약 확정 자체를 '접수했다'고 해요.
           환자가 실제로 병원에 오는 건 '내원'이라고 하는데,
           그게 화면에서 '체크인'이라고 돼 있어서..."
  
  결과:
    개발자의 "confirm" = 예약을 확정하는 행위
    간호사의 "접수"    = 예약이 완료된 상태 (= 개발자의 "confirm"과 동일)
    환자의 "내원"      = 실제로 병원에 온 것 (= 개발자의 "checkIn"과 동일)
  
  즉, 같은 개념을 다른 이름으로 구현해서 혼란 발생
  이 언어 불일치가 3번의 잘못된 기능 구현으로 이어짐

추가 문제:
  state = "DONE"
    Q: 진료가 완료된 것인가? 차팅(진료 기록 작성)이 완료된 것인가?
    → 개발자는 모르고, 코드만 있음
    → 새 개발자가 "DONE" 상태를 잘못 처리
```

---

## ✨ 올바른 접근 (After — Ubiquitous Language가 코드에 반영된 모습)

```
1단계: 도메인 전문가와 언어 정의 (Glossary 작성)

| 도메인 용어    | 정의                              | 코드 명칭       |
|------------|----------------------------------|----------------|
| 예약        | 환자가 특정 날짜/시간에 진료를 신청한 것 | Appointment    |
| 접수        | 예약이 병원 측에서 확인/승인된 상태     | ACCEPTED       |
| 내원        | 환자가 실제로 병원에 온 것             | ARRIVED        |
| 진료중       | 의사가 환자를 진료하는 상태             | IN_CONSULTATION|
| 차팅 완료    | 진료 기록 작성까지 완료된 상태          | CHARTING_DONE  |
| 노쇼        | 예약 시간에 내원하지 않은 상태          | NO_SHOW        |
| 예약 취소    | 환자 또는 병원이 예약을 철회한 것        | CANCELLED      |
```

```java
// Ubiquitous Language가 코드에 그대로 반영됨
public class Appointment {  // "예약" → Appointment

    private AppointmentId id;
    private PatientId patientId;
    private DoctorId doctorId;
    private AppointmentStatus status;
    private LocalDateTime scheduledAt;  // "예약된 일시"

    // 도메인 전문가가 "접수한다"고 말하는 행위
    public void accept() {
        if (this.status != AppointmentStatus.REQUESTED) {
            throw new AppointmentNotAcceptableException(
                "요청 상태의 예약만 접수할 수 있습니다. 현재 상태: " + this.status
            );
        }
        this.status = AppointmentStatus.ACCEPTED;
        this.events.add(new AppointmentAccepted(this.id));
    }

    // 도메인 전문가가 "내원했다"고 말하는 행위
    public void arrive() {
        if (this.status != AppointmentStatus.ACCEPTED) {
            throw new AppointmentException("접수된 예약만 내원 처리할 수 있습니다");
        }
        this.status = AppointmentStatus.ARRIVED;
        this.events.add(new PatientArrived(this.id, this.patientId));
    }

    // 도메인 전문가가 "차팅 완료"라고 말하는 행위
    public void completeCharting(ChartNote chartNote) {
        if (this.status != AppointmentStatus.IN_CONSULTATION) {
            throw new AppointmentException("진료중 상태에서만 차팅 완료할 수 있습니다");
        }
        this.status = AppointmentStatus.CHARTING_DONE;
        this.events.add(new ChartingCompleted(this.id, chartNote));
    }

    // 도메인 전문가가 "노쇼 처리"라고 말하는 행위
    public void markAsNoShow() {
        if (this.status != AppointmentStatus.ACCEPTED) {
            throw new AppointmentException("접수 상태에서만 노쇼 처리할 수 있습니다");
        }
        this.status = AppointmentStatus.NO_SHOW;
        this.events.add(new AppointmentNoShow(this.id, this.patientId));
    }
}

// 상태 이름이 도메인 언어와 일치
public enum AppointmentStatus {
    REQUESTED,       // 요청됨 (환자가 신청)
    ACCEPTED,        // 접수됨 (병원이 확인)
    ARRIVED,         // 내원함
    IN_CONSULTATION, // 진료중
    CHARTING_DONE,   // 차팅 완료
    NO_SHOW,         // 노쇼
    CANCELLED        // 취소됨
}
```

```
결과:
  간호사: "예약을 접수했는데 왜..."
  개발자: "appointment.accept() 호출하면 ACCEPTED 상태가 됩니다"
  간호사: "아, 코드 이름이 저희 용어랑 같네요"

  새 요구사항: "노쇼 환자가 다시 예약하면 별도 확인 절차"
  개발자: AppointmentStatus.NO_SHOW 에서 출발
  → 어디를 수정해야 할지 즉시 파악 가능
```

---

## 🔬 내부 동작 원리

### 1. 언어 불일치가 버그를 만드는 메커니즘

```
언어 불일치 → 개념 불일치 → 코드 불일치 → 버그

Step 1: 기획자와 개발자가 다른 언어를 씀
  기획서: "회원이 '구독 해지'를 신청하면 즉시 서비스가 종료됩니다"
  개발자: "회원 삭제 기능을 구현하면 되겠군"

Step 2: 개발자가 자신의 해석으로 구현
  void deleteAccount(Long memberId) {
      memberRepository.deleteById(memberId);
      // 회원 데이터 완전 삭제
  }

Step 3: QA에서 발견 (또는 운영 중 발견)
  "구독 해지 신청을 했는데 로그인이 안 돼요"
  → 기획 의도: 현재 구독 기간 종료까지 서비스 이용 가능
  → 구현 결과: 즉시 계정 삭제

실제 도메인 언어를 반영했다면:
  "구독 해지" ≠ "계정 삭제"
  → Subscription.cancelAtPeriodEnd()  // 기간 종료 시 해지 예약
  → Member 자체는 삭제되지 않음
  → 코드 이름에서 의도를 읽을 수 있음
```

### 2. 코드에서 Ubiquitous Language 무너진 징후

```
징후 1: 코드 이름이 도메인 언어와 다름

  도메인: "환불"  → 코드: refund()     ✅
  도메인: "반품"  → 코드: returnItem() ✅
  도메인: "교환"  → 코드: processExchange() ✅
  
  vs
  
  도메인: "정산"  → 코드: calculate()   ❌ (너무 일반적)
  도메인: "환불"  → 코드: reverse()     ❌ (기술적 용어)
  도메인: "구독"  → 코드: subscription / plan / recurring 혼용 ❌

징후 2: 개발자가 도메인 용어를 설명해야 함

  도메인 전문가: "환불 처리 로직 어디에 있어요?"
  개발자: "음... PaymentService의 processReversal() 메서드인데,
           그 전에 OrderService의 rollbackTransaction()을 먼저..."
  → 도메인 언어와 코드 언어가 다르다는 증거

징후 3: 같은 개념이 여러 이름으로 불림

  "사용자"가 코드에서:
    User, Member, Customer, Account, Principal 로 혼용됨
  → 같은 개념인가? 다른 개념인가? 아무도 모름

징후 4: 주석이 코드를 번역함

  // 여기서 "confirm"은 실제로는 "접수" 처리임
  void confirm(Long reservationId) { ... }
  → 주석이 필요하다면 이름이 잘못된 것
```

### 3. Glossary 관리 방법

```
Ubiquitous Language Glossary 구조:

용어: Appointment (예약)
정의: 환자가 특정 날짜/시간에 특정 의사에게 진료를 신청한 것
범위: 진료 컨텍스트 (Consultation Context)
관련 개념: AppointmentStatus, Patient, Doctor
주의: 배송 컨텍스트의 Delivery Slot(배송 슬롯)과 다름

용어: NoShow (노쇼)
정의: 예약 시간에 환자가 내원하지 않은 상태
발생 조건: ACCEPTED 상태에서 내원 시간 초과 시
불변식: 노쇼 처리 후 같은 날 재예약 시 별도 승인 필요
코드: AppointmentStatus.NO_SHOW, Appointment.markAsNoShow()

---

관리 원칙:
  1. 코드가 아닌 도메인 전문가 언어가 기준
     → 기술 용어(캐시, 트랜잭션)를 Glossary에 넣지 않음
  
  2. 문서와 코드를 동기화
     → Glossary가 바뀌면 코드도 함께 바뀜
     → 코드 리뷰 시 "이 용어가 Glossary에 있는가?" 확인
  
  3. Context별로 분리
     → 주문 컨텍스트의 "주문" vs 배송 컨텍스트의 "주문"
     → 같은 단어라도 컨텍스트마다 다른 정의를 가질 수 있음
```

### 4. Bounded Context마다 언어가 다를 수 있는 이유

```
같은 "주문(Order)"이지만 컨텍스트마다 다른 의미:

주문 컨텍스트 (Order Context):
  Order = 고객이 상품을 구매하기로 한 계약
  중요한 속성: 주문 항목, 총 금액, 주문 상태, 쿠폰 적용 여부
  중요한 행동: 주문하기, 취소하기, 결제하기

배송 컨텍스트 (Shipping Context):
  Order = 배송해야 할 패키지 묶음
  중요한 속성: 배송지 주소, 택배사, 운송장 번호, 무게/크기
  중요한 행동: 배송 시작, 배송 완료, 반품 접수

결제 컨텍스트 (Payment Context):
  Order = 결제가 필요한 청구 건
  중요한 속성: 결제 금액, 결제 수단, 결제 상태
  중요한 행동: 결제 승인, 결제 취소, 환불

┌──────────────────┐  OrderPlaced 이벤트  ┌──────────────────┐
│   주문 컨텍스트    │ ──────────────────> │   배송 컨텍스트    │
│  Order: 구매 계약 │                     │  Order: 배송 패키지│
└──────────────────┘                     └──────────────────┘
         │
         │ OrderPlaced 이벤트
         ▼
┌──────────────────┐
│   결제 컨텍스트    │
│  Order: 청구 건  │
└──────────────────┘

핵심:
  각 컨텍스트의 "Order"는 다른 클래스
  컨텍스트 간 통신은 이벤트 (OrderId만 전달)
  배송 컨텍스트가 주문 컨텍스트의 Order를 직접 참조 ❌
  → 이것이 Bounded Context의 핵심
```

---

## 💻 실전 코드

### Glossary 기반 코드 작성 예시

```java
// Glossary: "정책(Policy)" = 특정 조건에서 적용되는 비즈니스 규칙
// Glossary: "할인(Discount)" = 원가에서 차감되는 금액 또는 비율
// Glossary: "적립(Accumulate)" = 구매 금액의 일부를 포인트로 쌓는 것

// 도메인 언어가 코드에 직접 반영됨
public class MembershipDiscountPolicy {

    // "회원 등급별 할인율을 계산한다" → calculateDiscountRate()
    public DiscountRate calculateDiscountRate(MembershipLevel level, Money orderAmount) {
        return switch (level) {
            case GOLD -> DiscountRate.of(10);    // GOLD 회원: 10% 할인
            case SILVER -> DiscountRate.of(5);   // SILVER 회원: 5% 할인
            default -> DiscountRate.NONE;
        };
    }
}

public class PointAccumulationPolicy {

    // "구매 금액의 1%를 적립한다" → accumulate()
    public Point accumulate(Money purchaseAmount) {
        return Point.of(purchaseAmount.toLong() / 100);
    }
}

// 도메인 이벤트도 언어 일치
public record OrderPlaced(         // "주문이 접수됐다"
    OrderId orderId,
    CustomerId customerId,
    Money totalAmount,
    LocalDateTime placedAt
) implements DomainEvent {}

public record OrderCancelled(      // "주문이 취소됐다"
    OrderId orderId,
    String cancellationReason
) implements DomainEvent {}

public record PointAccumulated(    // "포인트가 적립됐다"
    CustomerId customerId,
    Point accumulatedPoint,
    OrderId sourceOrderId
) implements DomainEvent {}
```

### Glossary와 코드 불일치를 감지하는 방법

```java
// 코드 리뷰 체크리스트:
// 1. 새 클래스/메서드 이름이 Glossary에 있는가?
// 2. Glossary에 없다면 새 용어인가, 기존 용어의 다른 표현인가?
// 3. 도메인 전문가가 이 이름을 읽었을 때 의미를 이해할 수 있는가?

// 나쁜 예: 기술적 관점의 이름
public class DataProcessor {          // 무엇을 처리하는가?
    void processData(Object data) {}  // 어떤 비즈니스 행위인가?
    void updateRecord(Long id) {}     // 무엇을 업데이트하는가?
}

// 좋은 예: 도메인 언어 기반 이름
public class OrderFulfillmentService {          // "주문 이행"
    void confirmShipment(OrderId orderId) {}    // "배송 확정"
    void processReturn(ReturnRequest request) {} // "반품 처리"
}
```

---

## 📊 설계 비교

```
언어 불일치 비용 vs Ubiquitous Language 도입 비용:

                   언어 불일치              Ubiquitous Language
─────────────────┼──────────────────────┼──────────────────────────
커뮤니케이션 비용  │ 회의마다 용어 설명 필요 │ 동일 언어로 즉시 소통
                │ "이 기능이 그 기능이냐" │
─────────────────┼──────────────────────┼──────────────────────────
버그 발생 원인    │ 언어 오해로 인한 잘못된 │ 코드 = 비즈니스 명세
                │ 구현                  │ → 오해 가능성 최소화
─────────────────┼──────────────────────┼──────────────────────────
온보딩           │ 새 개발자가 용어 학습에 │ 코드 보면 도메인 이해
                │ 오랜 시간 필요          │ (코드가 문서)
─────────────────┼──────────────────────┼──────────────────────────
요구사항 반영     │ 기획서 → 번역 → 코드   │ 기획서 용어 → 코드 직접
                │ (번역 오류 발생 가능)   │ 매핑
─────────────────┼──────────────────────┼──────────────────────────
초기 비용         │ 낮음                  │ Glossary 작성 시간 필요
─────────────────┼──────────────────────┼──────────────────────────
장기 유지보수 비용 │ 매우 높음              │ 낮음
```

---

## ⚖️ 트레이드오프

```
Ubiquitous Language 도입의 현실적 어려움:

① 초기 합의 비용
   도메인 전문가, 기획자, 개발자가 모여 언어를 정의하는 세션 필요
   특히 "같은 말, 다른 의미" 충돌을 해결하는 데 시간 필요
   → 초기에 투자하지 않으면 나중에 더 큰 비용

② 언어는 계속 진화함
   비즈니스가 변하면 도메인 언어도 변함
   언어가 변하면 코드 이름도 함께 변해야 함
   → 이름 변경(rename) 비용이 지속적으로 발생
   → 하지만 이 비용이 언어 불일치 비용보다 낮음

③ 팀원마다 도메인 이해도가 다름
   도메인을 잘 모르는 개발자는 기술 용어를 씀
   → 코드 리뷰에서 "이 이름이 Glossary와 맞는가?" 확인이 필요

현실적 전략:
  완벽한 Glossary부터 시작하려 하지 말 것
  → 핵심 개념 10~20개부터 시작
  → 코드 리뷰에서 점진적으로 확장
  → 불일치 발견 시 즉시 수정 (기술 부채로 남기지 않음)
```

---

## 📌 핵심 정리

```
Ubiquitous Language 핵심:

정의:
  개발자 + 도메인 전문가 + 기획자가 공유하는 하나의 언어
  이 언어가 코드 이름에 그대로 반영됨

왜 필요한가:
  언어 불일치 → 개념 오해 → 잘못된 구현 → 버그
  번역 비용 없이 코드가 비즈니스 명세 역할

코드에서 확인하는 법:
  도메인 전문가가 코드를 읽었을 때 의미를 이해할 수 있는가?
  주석 없이 메서드 이름만으로 비즈니스 의도를 읽을 수 있는가?

Bounded Context와의 관계:
  같은 단어가 컨텍스트마다 다른 의미를 가질 수 있음
  → 각 컨텍스트 내에서 언어는 일관되어야 함
  → 컨텍스트 경계가 언어 경계

관리 방법:
  Glossary 유지 (용어 정의 + 코드 명칭 매핑)
  코드 리뷰 시 Glossary와의 일치 확인
  언어 변경 시 코드도 함께 변경 (rename refactoring)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 Ubiquitous Language가 무너진 부분을 찾고, 어떻게 개선할 수 있는가?

```java
@Service
public class ProductService {
    public void updateProductStatus(Long productId, int flag) {
        Product product = productRepository.findById(productId).orElseThrow();
        if (flag == 1) {
            product.setActive(true);
        } else if (flag == 2) {
            product.setActive(false);
            product.setHidden(true);
        } else if (flag == 3) {
            product.setDeleted(true);
        }
        productRepository.save(product);
    }
}
```

<details>
<summary>해설 보기</summary>

**무너진 부분들:**

1. `updateProductStatus()`와 `flag` — "flag가 1이면 뭔가, 2이면 뭔가"는 도메인 언어가 전혀 아닙니다. 도메인 전문가에게 "flag 2로 상품 처리"는 아무 의미도 없습니다.
2. `setActive(true/false)`, `setHidden(true)`, `setDeleted(true)` — 불린 플래그 조합이 어떤 비즈니스 상태를 의미하는지 코드로 알 수 없습니다.

**개선:**

```java
// Glossary: 
//   "판매 중" = 고객에게 노출되고 구매 가능한 상태 → SELLING
//   "판매 중지" = 고객에게 노출되지 않는 상태 → HIDDEN
//   "삭제" = 시스템에서 완전히 제거된 상태 → DISCONTINUED

public class Product {
    private ProductStatus status;

    public void startSelling() {  // "판매 시작"
        this.status = ProductStatus.SELLING;
    }

    public void hide() {          // "판매 중지 (숨김)"
        this.status = ProductStatus.HIDDEN;
    }

    public void discontinue() {   // "단종 처리"
        this.status = ProductStatus.DISCONTINUED;
    }
}

public enum ProductStatus { SELLING, HIDDEN, DISCONTINUED }
```

이제 `product.hide()` 하나로 의도가 명확하고, 도메인 전문가도 이해할 수 있는 코드가 됩니다.

</details>

---

**Q2.** "결제(Payment)"라는 단어가 결제 컨텍스트에서는 "고객이 돈을 지불하는 행위"이고, 정산 컨텍스트에서는 "파트너사에게 수수료를 지급하는 행위"를 의미한다면, 이 두 컨텍스트에서 Payment 클래스를 어떻게 설계해야 하는가?

<details>
<summary>해설 보기</summary>

**두 컨텍스트에서 완전히 다른 클래스로 분리해야 합니다.**

```java
// 결제 컨텍스트 (Payment Context)
package com.example.payment;

public class Payment {  // 고객이 주문 대금을 지불한 것
    private PaymentId id;
    private CustomerId customerId;
    private OrderId orderId;
    private Money amount;
    private PaymentMethod method;   // 신용카드, 계좌이체 등
    private PaymentStatus status;   // PENDING, APPROVED, CANCELLED

    public void approve(TransactionId transactionId) { ... }
    public void cancel() { ... }
}

// 정산 컨텍스트 (Settlement Context)
package com.example.settlement;

public class PartnerSettlement {  // 파트너사에게 수수료를 지급하는 것
    private SettlementId id;
    private PartnerId partnerId;
    private Money settlementAmount;
    private Money commissionFee;
    private SettlementPeriod period;  // 정산 주기 (주간, 월간)
    private SettlementStatus status;  // PENDING, PROCESSED, PAID

    public void process() { ... }
    public void markAsPaid(BankTransferId transferId) { ... }
}
```

두 개념은 이름이 비슷해도 완전히 다른 비즈니스 규칙과 속성을 갖습니다. 억지로 하나의 `Payment` 클래스로 통합하면 두 컨텍스트의 불변식이 서로를 침범합니다.

**컨텍스트 간 연결**: 결제가 완료되면 `PaymentApproved` 이벤트가 발행되고, 정산 컨텍스트가 이 이벤트를 구독해서 `PartnerSettlement` 레코드를 생성합니다.

</details>

---

**Q3.** Ubiquitous Language가 코드베이스 전체에 걸쳐 일관되게 유지되는지 확인하는 실용적인 방법은 무엇인가?

<details>
<summary>해설 보기</summary>

**실용적인 방법들:**

1. **ADR(Architecture Decision Record) 또는 Glossary 문서**를 코드 저장소에 함께 관리합니다. (`/docs/glossary.md`)

2. **코드 리뷰 체크리스트**에 추가:
   - "새 클래스/메서드 이름이 Glossary에 있는 용어인가?"
   - "도메인 전문가가 이 이름을 이해할 수 있는가?"

3. **ArchUnit으로 네이밍 규칙 강제**:
```java
@Test
void domainClassesShouldUseDomainLanguage() {
    // 예: Payment 컨텍스트에서 "Transaction"이라는 이름 사용 금지
    // (우리 도메인에서 "거래"는 "Payment"로 통일하기로 했음)
    noClasses()
        .that().resideInAPackage("..payment..")
        .should().haveSimpleNameContaining("Transaction")
        .check(importedClasses);
}
```

4. **도메인 전문가를 코드 리뷰에 간헐적으로 참여**시킵니다. 코드를 읽히고 "이 이름이 무슨 뜻인지 바로 이해되시나요?"를 물어보는 것만으로도 언어 불일치를 조기에 발견할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: DDD가 해결하는 문제](./01-what-ddd-solves.md)** | **[홈으로 🏠](../README.md)** | **[다음: Strategic Design vs Tactical Design ➡️](./03-strategic-vs-tactical.md)**

</div>
