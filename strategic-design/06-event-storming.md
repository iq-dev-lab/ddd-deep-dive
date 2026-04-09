# 이벤트 스토밍 — 도메인 전문가와 함께 컨텍스트 발견하기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 이벤트 스토밍이란 무엇이고, 왜 개발자 혼자 도메인을 설계하면 안 되는가?
- Command / Event / Aggregate / Policy / Read Model 스티커는 각각 무엇을 나타내는가?
- 이벤트 스토밍 결과물이 어떻게 Bounded Context 경계의 초안이 되는가?
- 원격으로 이벤트 스토밍을 진행할 때 사용하는 도구와 실전 팁은?
- 이벤트 스토밍 후 DDD 코드 설계로 어떻게 이어가는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

개발자가 혼자 도메인을 분석해 설계하면 어떻게 되는가? 요구사항 문서를 읽고, 이해한 대로 Aggregate와 Entity를 설계한다. 3개월 후 도메인 전문가가 시스템을 보고 "이게 아닌데요"라고 한다. 개발자가 이해한 "예약"과 의사의 "예약"이 달랐기 때문이다.

이벤트 스토밍은 Alberto Brandolini가 고안한 **도메인 전문가와 개발자가 함께 도메인을 발견하는 워크숍**이다. 포스트잇과 화이트보드만 있으면 된다. 이 워크숍의 결과물이 Bounded Context 경계의 초안이 되고, Aggregate의 후보가 되며, Domain Event의 목록이 된다. 코드를 한 줄도 작성하기 전에 도메인에 대한 공유된 이해를 만드는 것이 목적이다.

---

## 😱 흔한 실수 (Before — 개발자 주도 도메인 분석)

```
상황: 병원 예약 시스템 개발

개발자 주도 분석 과정:
  1. 기획서 읽기 (30페이지 PDF)
  2. DB 테이블 설계: appointments, doctors, patients, schedules
  3. Entity 설계: Appointment, Doctor, Patient, Schedule
  4. Service 설계: AppointmentService, ScheduleService

구현 후 도메인 전문가(의사, 간호사)에게 데모:

의사: "이게... 진료과별 예약 시스템인가요, 의사별 예약 시스템인가요?"
개발자: "의사별로 설계했습니다"
의사: "저는 내과이고 특정 진료실을 쓰는데, 진료실 예약이랑 의사 예약이 다른 거 알고 계세요?"

간호사: "수술 예약은 일반 진료 예약과 프로세스가 완전히 달라요"
개발자: "...Appointment 하나로 처리했는데요"

원무팀: "보험 수가 코드 입력이 없어요. 이게 없으면 청구를 못 해요"
개발자: "기획서에는 없었는데요..."

결과:
  3개월 개발 후 50% 재설계
  핵심 도메인 개념을 잘못 이해한 채로 구현
  "수술 예약"과 "진료 예약"이 다르다는 것을 3개월 후 발견

근본 원인:
  도메인 전문가와 함께 탐색하지 않음
  기획서는 비즈니스 규칙을 다 담지 못함
  개발자가 추측으로 채운 공백이 잘못된 설계로 이어짐
```

---

## ✨ 올바른 접근 (After — 이벤트 스토밍 워크숍)

```
이벤트 스토밍 준비:
  참여자: 의사 2명, 간호사 2명, 원무팀 1명, 개발자 3명, 퍼실리테이터 1명
  장소: 화이트보드가 있는 큰 방 (또는 Miro 온라인 보드)
  준비물: 색깔별 포스트잇, 마커
  시간: 4~6시간 (처음이라면 하루)

이벤트 스토밍 진행 과정:

Step 1: Domain Event 도출 (주황색 포스트잇)
  퍼실리테이터: "이 시스템에서 일어나는 중요한 일들을 과거형으로 적어보세요"
  
  도메인 전문가들이 포스트잇 붙이기:
  [예약이 요청됨] [예약이 접수됨] [환자가 내원함] [진료가 시작됨]
  [동의서가 서명됨] [차팅이 완료됨] [수납이 완료됨] [예약이 취소됨]
  [처방이 발행됨] [검사가 의뢰됨] [수술이 예약됨] [수술실이 배정됨]
  
  → 이 과정에서 이미 "수술 예약"과 "진료 예약"이 다른 이벤트임을 발견!

Step 2: Command 추가 (파란색 포스트잇)
  각 Event 앞에 "무엇이 이 Event를 발생시켰는가?"
  
  [예약 요청] → [예약이 요청됨]
  [예약 접수] → [예약이 접수됨]
  [수납 처리] → [수납이 완료됨]

Step 3: Aggregate 발견 (노란색 포스트잇)
  "Command가 어느 개념에 작용하는가?"
  
  [예약 요청] → (Appointment) → [예약이 요청됨]
  [수납 처리] → (Bill) → [수납이 완료됨]
  → Appointment와 Bill이 다른 개념임을 발견 (수납은 예약이 아님)

Step 4: Policy 추가 (보라색 포스트잇)
  "이 Event가 발생하면 자동으로 무엇이 일어나는가?"
  
  [예약이 접수됨] → "WHENEVER: 보험 청구 코드 자동 생성"
  [수납이 완료됨] → "WHENEVER: 영수증 자동 발행, 알림 발송"

Step 5: Bounded Context 발견
  이벤트들이 자연스럽게 묶이는 그룹 발견:
  
  [예약 관련 이벤트] → 예약 Context
  [진료/차팅 관련] → 진료 Context
  [수납/청구 관련] → 수납 Context
  [수술 관련]     → 수술 Context (진료와 완전히 다른 프로세스!)
```

---

## 🔬 내부 동작 원리

### 1. 이벤트 스토밍 스티커 체계

```
색깔별 스티커 의미:

🟠 주황색 — Domain Event (도메인 이벤트)
  "시스템에서 일어난 중요한 일" — 항상 과거형
  
  좋은 예: [예약이 접수됨], [결제가 완료됨], [배송이 시작됨]
  나쁜 예: [예약 접수], [결제 완료] (과거형이 아님)
           [데이터가 저장됨] (기술적 이벤트, 비즈니스 이벤트 아님)
  
  핵심: 도메인 전문가가 "중요하다"고 생각하는 것

🔵 파란색 — Command (명령)
  "이 이벤트를 발생시키는 행위/의도"
  
  예: [예약 접수] → [예약이 접수됨]
      [결제 승인] → [결제가 완료됨]
  
  Command = 사용자 또는 시스템의 의도

🟡 노란색 — Aggregate
  "Command가 작용하는 비즈니스 개념"
  
  예: [예약 접수] → (Appointment) → [예약이 접수됨]
  
  → 이것이 Aggregate Root의 후보가 됨

🟣 보라색 — Policy (정책)
  "Event 발생 시 자동으로 실행되는 규칙"
  
  예: [예약이 접수됨] → "WHENEVER: 확인 SMS 발송"
      [수납이 완료됨] → "WHENEVER: 영수증 발행"
  
  → Domain Event Handler의 후보가 됨

🟢 초록색 — Read Model (읽기 모델)
  "사용자 또는 시스템이 의사결정을 위해 보는 데이터"
  
  예: (예약 현황 화면) — 의사가 오늘 예약 목록 볼 때
      (재고 현황) — 주문 처리 전 재고 확인 시

👤 분홍색 — Actor (행위자)
  "Command를 실행하는 사람 또는 시스템"
  
  예: [환자] → [예약 요청]
      [간호사] → [예약 접수]
      [시스템] → [자동 노쇼 처리]
```

### 2. Bounded Context 발견 과정

```
이벤트 스토밍 보드에서 Context를 발견하는 방법:

Before Context 발견 (이벤트만 나열):
  [예약이 요청됨][예약이 접수됨][환자가 내원함][진료가 시작됨]
  [동의서가 서명됨][처방이 발행됨][검사가 의뢰됨]
  [수납이 완료됨][보험 청구가 완료됨]
  [수술이 예약됨][수술실이 배정됨][마취팀이 배정됨]

Context 발견 힌트:
  ① 언어가 바뀌는 지점: "예약"과 "수납"은 완전히 다른 언어
  ② 부서/역할이 바뀌는 지점: 간호사 담당과 원무팀 담당이 다름
  ③ Policy가 Context 경계를 연결: "예약이 접수되면(A) → 보험 코드 등록(B)"
     → A와 B가 다른 Context의 신호

After Context 발견:
  ┌──────────────────┐  ┌──────────────────┐
  │   예약 Context   │  │   진료 Context   │
  │[예약이 요청됨]   │  │[진료가 시작됨]   │
  │[예약이 접수됨]   │  │[동의서가 서명됨] │
  │[환자가 내원함]   │  │[처방이 발행됨]   │
  │[예약이 취소됨]   │  │[검사가 의뢰됨]   │
  └──────────────────┘  └──────────────────┘

  ┌──────────────────┐  ┌──────────────────┐
  │   수납 Context   │  │   수술 Context   │
  │[수납이 완료됨]   │  │[수술이 예약됨]   │
  │[보험 청구됨]     │  │[수술실이 배정됨] │
  │[영수증이 발행됨] │  │[마취팀이 배정됨] │
  └──────────────────┘  └──────────────────┘
```

### 3. Hot Spot과 의문 스티커

```
이벤트 스토밍 중 의문 처리:

빨간색 스티커 — Hot Spot (핫스팟):
  "여기서 논쟁이 있다" 또는 "불명확하다"
  
  예: "예약 취소 시 수납 취소는 자동인가, 수동인가?"
      "수술 예약은 일반 예약 시스템에 포함인가, 별도인가?"
  
  → 나중에 별도로 심층 논의
  → 이 지점이 Context 경계일 가능성 높음

처리 방법:
  핫스팟을 즉시 해결하려 하지 말 것
  → 워크숍 흐름을 끊음
  → 핫스팟에 빨간 스티커 붙이고 계속 진행
  → 워크숍 후 관련 전문가와 별도 논의

핫스팟 통계:
  이벤트 스토밍 4시간 → 평균 20~40개 핫스팟 발생
  → 이 핫스팟 목록 자체가 "아직 모르는 것" 목록
  → 개발 전에 해소해야 할 것들
```

### 4. 이벤트 스토밍 결과물 → DDD 설계 연결

```
이벤트 스토밍 결과물을 DDD 코드 설계로 변환:

[이벤트 스토밍]            [DDD 코드 설계]

Domain Event ────────→ Domain Event 클래스
  [예약이 접수됨]          AppointmentAccepted.java

Command ──────────────→ Application Service 메서드
  [예약 접수]              appointmentService.accept(id)
  또는 Aggregate 메서드
                           appointment.accept()

Aggregate ────────────→ Aggregate Root 클래스
  (Appointment)            Appointment.java

Policy ───────────────→ Domain Event Handler
  "WHENEVER: SMS 발송"     @EventListener AppointmentAcceptedHandler

Read Model ───────────→ Query Model / CQRS 읽기 모델
  (예약 현황 화면)          AppointmentListView.java

Bounded Context ──────→ 패키지 구조
  예약 Context             com.hospital.appointment
  수납 Context             com.hospital.billing
```

---

## 💻 실전 코드

### 이벤트 스토밍 결과물 → 코드

```java
// 이벤트 스토밍에서 발견된 Domain Event → 코드
public record AppointmentAccepted(
    AppointmentId appointmentId,
    PatientId patientId,
    DoctorId doctorId,
    LocalDateTime acceptedAt
) implements DomainEvent {}

// 이벤트 스토밍에서 발견된 Aggregate → 코드
public class Appointment {

    // 이벤트 스토밍에서 발견된 Command들
    public void accept() {                          // [예약 접수] Command
        if (this.status != AppointmentStatus.REQUESTED) {
            throw new AppointmentNotAcceptableException();
        }
        this.status = AppointmentStatus.ACCEPTED;
        this.events.add(new AppointmentAccepted(id, patientId, doctorId, LocalDateTime.now()));
    }

    public void arrive() {                          // [내원 처리] Command
        if (this.status != AppointmentStatus.ACCEPTED) {
            throw new AppointmentException("접수 상태에서만 내원 처리 가능");
        }
        this.status = AppointmentStatus.ARRIVED;
        this.events.add(new PatientArrived(id, patientId));
    }

    public void markAsNoShow() {                    // [노쇼 처리] Command
        this.status = AppointmentStatus.NO_SHOW;
        this.events.add(new AppointmentNoShow(id, patientId));
    }
}

// 이벤트 스토밍에서 발견된 Policy → 코드
@Component
public class AppointmentAcceptedHandler {

    private final SmsNotificationPort smsPort;

    // Policy: "WHENEVER 예약이 접수되면 → 확인 SMS 발송"
    @EventListener
    @Async
    public void handle(AppointmentAccepted event) {
        smsPort.send(
            event.patientId(),
            "예약이 확인됐습니다. 방문 일정: " + event.acceptedAt()
        );
    }
}
```

### Miro 기반 원격 이벤트 스토밍 세팅

```
원격 이벤트 스토밍 도구:
  Miro (https://miro.com): 가장 인기, 포스트잇 기능 강력
  Mural (https://mural.co): Miro와 유사
  FigJam (https://figma.com/figjam): Figma 사용자에게 친숙

Miro 세팅 체크리스트:
  □ 색상별 포스트잇 템플릿 준비 (주황, 파랑, 노랑, 보라, 초록)
  □ 화면 오른쪽에 색상 범례 (각 색깔의 의미 표시)
  □ 타임라인 화살표 (왼쪽 = 과거, 오른쪽 = 미래)
  □ 핫스팟용 빨간 스티커 준비
  □ 참여자에게 사전에 Miro 사용법 공유

원격 이벤트 스토밍 팁:
  ① 침묵 브레인스토밍 먼저 (5분)
     모두가 동시에 포스트잇 작성 → 한 사람이 독점하지 않음
  
  ② 투표 기능 활용
     "이 이벤트가 중요한가?" → Miro 투표로 우선순위 결정
  
  ③ 세션을 짧게 (2~3시간 × 2회)
     온라인 집중력 한계 고려 → 하루 6시간보다 2~3시간 × 2회가 효과적
  
  ④ 결과물 즉시 사진/스크린샷 저장
     세션 종료 전 반드시 저장 → 다음 세션에서 이어가기
```

---

## 📊 설계 비교

```
이벤트 스토밍 vs 전통적 요구사항 분석:

                전통적 분석              이벤트 스토밍
────────────┼──────────────────────┼────────────────────────────
참여자       │ 기획자 → 개발자 → QA │ 도메인 전문가 + 개발자 함께
            │ (순차적)              │ (동시)
────────────┼──────────────────────┼────────────────────────────
결과물       │ 요구사항 문서         │ 이벤트 맵 + Context 경계 초안
────────────┼──────────────────────┼────────────────────────────
Domain      │ 개발자가 추측          │ 전문가와 함께 발견
Event 발견   │                      │
────────────┼──────────────────────┼────────────────────────────
오해 발견    │ 구현 후               │ 워크숍 중 (조기 발견)
────────────┼──────────────────────┼────────────────────────────
Context     │ 아키텍처 회의에서 결정  │ 이벤트 흐름에서 자연스럽게 발견
경계         │ (추측 기반)            │
────────────┼──────────────────────┼────────────────────────────
소요 시간   │ 분석 2주 + 재설계 2주  │ 워크숍 1~2일 + 설계 1주
────────────┼──────────────────────┼────────────────────────────
도메인 지식  │ 개발자에게 단방향 전달  │ 양방향 공유/발견
전파         │                      │
```

---

## ⚖️ 트레이드오프

```
이벤트 스토밍의 어려움:

① 도메인 전문가 참여 확보
   바쁜 의사, 간호사를 하루 종일 워크숍에 참여시키기 어려움
   → 2~3시간 짧은 세션으로 분리
   → 경영진 지지 필요 (이 투자의 가치를 설명)

② 퍼실리테이터 역할의 중요성
   진행자가 없으면 일부 사람이 독점하거나 방향을 잃음
   → 퍼실리테이터: 내용 전문가 아닌 프로세스 전문가
   → DDD 경험자가 퍼실리테이터 또는 옵저버로 참여

③ 결과물이 "초안"에 불과
   이벤트 스토밍 = 탐색 단계, 완성이 아님
   → 이후 추가 분석, 프로토타입, 검증이 필요
   → "이벤트 스토밍 결과를 그대로 코드로" 는 위험

④ 원격 이벤트 스토밍의 한계
   물리적 공간의 에너지와 상호작용을 온라인으로 완전히 재현 불가
   → 가능하면 첫 번째는 대면으로
   → 후속 세션은 온라인으로 진행

현실적 적용:
  "완벽한 이벤트 스토밍"보다 "도메인 전문가와 함께하는 1시간"이 시작
  정해진 형식보다 "함께 비즈니스 흐름을 그려보기" 자체가 핵심
```

---

## 📌 핵심 정리

```
이벤트 스토밍 핵심:

목적:
  도메인 전문가와 개발자가 함께 도메인을 발견
  오해를 조기에 발견 (구현 후가 아닌 구현 전)
  Bounded Context 경계의 초안 생성

스티커 체계:
  🟠 Domain Event: 과거형, 비즈니스에서 중요한 사건
  🔵 Command: 이벤트를 일으키는 의도/행위
  🟡 Aggregate: Command가 작용하는 비즈니스 개념
  🟣 Policy: "WHENEVER~": 이벤트에 반응하는 규칙
  🟢 Read Model: 의사결정을 위한 데이터

Context 발견:
  언어가 바뀌는 지점 → Context 경계 후보
  Policy가 연결하는 지점 → Context 간 통신 후보
  Hot Spot → 불명확한 경계, 추가 탐색 필요

결과물 → 코드:
  Domain Event → 이벤트 클래스
  Command → Application Service 메서드 / Aggregate 메서드
  Aggregate → Aggregate Root 클래스
  Policy → Domain Event Handler
  Context → 패키지 구조

팁:
  도메인 전문가가 말하는 것을 그대로 과거형 이벤트로 적기
  개발 용어 사용 금지 (DB, 테이블, CRUD)
  판단/논쟁은 Hot Spot으로 표시하고 계속 진행
```

---

## 🤔 생각해볼 문제

**Q1.** 이벤트 스토밍에서 "데이터가 저장됨", "API가 호출됨" 같은 기술적 이벤트가 등장하면 어떻게 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**기술적 이벤트는 제거하거나 비즈니스 언어로 재정의합니다.**

이벤트 스토밍에서 Domain Event는 **비즈니스에서 의미 있는 사건**이어야 합니다. 기술적 이벤트는 구현 세부사항이지 도메인 지식이 아닙니다.

```
기술적 이벤트 → 비즈니스 이벤트로 변환:

"데이터가 저장됨" → X (도메인 의미 없음)
  → 무엇이 저장됐는가? "예약이 접수됨" ✅

"API가 호출됨" → X
  → 왜 호출됐는가? "결제가 승인 요청됨" ✅

"테이블에 레코드가 삽입됨" → X
  → "주문이 생성됨" ✅
```

기술적 이벤트가 많이 등장하면, 참여자들이 비즈니스 관점으로 생각하지 않는다는 신호입니다. 퍼실리테이터가 "도메인 전문가 입장에서 중요한 일인가요?"라고 물어보는 것으로 리다이렉션합니다.

</details>

---

**Q2.** 이벤트 스토밍 결과물을 보면 Aggregate 후보가 너무 많다 (20개 이상). 이것을 어떻게 줄이고 정제하는가?

<details>
<summary>해설 보기</summary>

**단계별 정제 방법:**

**Step 1: 이벤트를 묶어 자연스러운 그룹 찾기**
비슷한 언어를 쓰는 이벤트들이 같은 Aggregate에 속할 가능성이 높습니다.

**Step 2: "이 Command가 항상 같은 객체에 작용하는가?" 확인**
`[예약 접수]`와 `[예약 취소]`는 둘 다 Appointment에 작용 → 하나의 Aggregate

**Step 3: 불변식 기준 통합**
"항상 함께 일관성이 보장되어야 하는 것들" → 같은 Aggregate

```
초기: 20개 Aggregate 후보
  Appointment, AppointmentRequest, AppointmentSlot, Doctor, DoctorSchedule,
  Patient, PatientRecord, Consent, ConsentForm, Bill, ...

정제 후: 7개 Aggregate
  Appointment (예약 + 슬롯 + 상태 통합)
  Doctor (의사 + 스케줄 통합)
  Patient (환자 + 기록 기본 정보)
  ConsentForm (동의서)
  Bill (청구)
  Prescription (처방)
  SurgerySchedule (수술 예약)
```

**Step 4: 크기 검증**
"이 Aggregate를 하나의 트랜잭션으로 처리할 때 문제가 없는가?"를 확인합니다. 너무 크면 분리, 너무 작으면 통합을 고려합니다.

</details>

---

**Q3.** 이벤트 스토밍을 도메인 전문가 없이 개발자끼리만 진행하는 것이 의미 있는가?

<details>
<summary>해설 보기</summary>

**의미는 있지만, 본래 목적을 달성하지는 못합니다.**

**개발자끼리 진행 시 얻을 수 있는 것:**
- 팀 내 도메인 이해 수준 파악
- 모르는 것(핫스팟)의 목록 생성
- 이후 도메인 전문가와 대화할 질문 목록 준비
- 초기 설계 아이디어 공유

**놓치는 것:**
- 도메인 전문가만 아는 암묵적 지식
- 실제 비즈니스 규칙의 예외와 엣지케이스
- "우리가 모르는 줄도 몰랐던 것" 발견

**실용적 접근:**
```
1단계: 개발자끼리 이벤트 스토밍 (2시간)
   → 우리가 이해한 것 + 모르는 것 목록 생성

2단계: 도메인 전문가와 검증 세션 (1시간)
   → 우리가 그린 것을 보여주고 "맞나요?" 질문
   → "그게 아니라 이렇습니다"가 나오는 순간이 핵심

이 방법이 "도메인 전문가 없이 요구사항 문서만 읽는 것"보다 훨씬 효과적:
  이벤트 맵이 있으면 전문가가 구체적으로 피드백 가능
  "이 이벤트가 맞나요?"가 "요구사항이 무엇인가요?"보다 대화하기 쉬움
```

</details>

---

<div align="center">

**[⬅️ 이전: Anticorruption Layer](./05-anticorruption-layer.md)** | **[홈으로 🏠](../README.md)** | **[다음: Bounded Context와 마이크로서비스 ➡️](./07-context-vs-microservice.md)**

</div>
