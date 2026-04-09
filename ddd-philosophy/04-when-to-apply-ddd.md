# DDD 적용 판단 기준 — 도메인 복잡도에 따른 아키텍처 선택

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 모든 프로젝트에 DDD를 적용하는 것이 왜 위험한가?
- 도메인 복잡도를 어떻게 측정하고, 어떤 기준으로 아키텍처를 선택하는가?
- Transaction Script, Table Module, Domain Model 각각이 적합한 상황은?
- "지금은 단순하지만 나중에 복잡해지면 어떡하나?" 라는 불안에 어떻게 대응하는가?
- DDD 도입의 실질적 비용과 편익을 어떻게 계산하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

DDD를 처음 공부하고 나면 "앞으로 모든 프로젝트를 DDD로 짜야겠다"는 생각이 들기 쉽다. 그런데 게시판에 Aggregate Root를 설계하고, 회원 닉네임 변경에 Domain Event를 발행하면 어떻게 되는가? 3개의 파일로 끝날 기능을 30개의 파일로 만드는 결과를 낳는다.

**도구는 문제에 맞아야 한다.** DDD는 복잡한 도메인을 다루기 위한 도구다. 단순한 문제에 복잡한 도구를 쓰면 도구 자체가 문제가 된다. 언제 DDD를 쓸 것인지 판단하는 능력이, DDD를 아는 것만큼 중요하다.

---

## 😱 흔한 실수 (Before — 복잡도와 무관하게 DDD 적용)

```
상황: 사내 공지사항 게시판 개발

DDD를 무조건 적용한 결과:
```

```java
// 공지사항 게시판에 적용한 DDD 코드
public class Notice {  // Aggregate Root
    private NoticeId id;                    // Value Object
    private AuthorId authorId;             // Value Object
    private Title title;                   // Value Object (불변성 보장)
    private Content content;               // Value Object (불변성 보장)
    private NoticeStatus status;           // Enum (게시됨/임시저장/삭제됨)
    private List<DomainEvent> events;

    // 공지사항 등록 팩토리 메서드
    public static Notice publish(AuthorId authorId, Title title, Content content) {
        Notice notice = new Notice(new NoticeId(), authorId, title, content, NoticeStatus.PUBLISHED);
        notice.events.add(new NoticePublished(notice.id, authorId));
        return notice;
    }

    // 불변식: 삭제된 공지사항은 수정 불가
    public void update(Title newTitle, Content newContent) {
        if (this.status == NoticeStatus.DELETED) {
            throw new NoticeNotModifiableException("삭제된 공지사항은 수정할 수 없습니다");
        }
        this.title = newTitle;
        this.content = newContent;
        this.events.add(new NoticeUpdated(this.id));
    }
}

// 이를 위해 필요한 파일들:
// NoticeId.java, AuthorId.java, Title.java, Content.java
// NoticeStatus.java, NoticeRepository.java (인터페이스)
// JpaNoticeRepository.java, NoticeMapper.java
// NoticeApplicationService.java, NoticePublished.java
// NoticeUpdated.java, NoticeDeleted.java
// NoticeNotModifiableException.java
// ...총 15+ 파일
```

```
실제로 필요한 것:
  공지사항은 단순 CRUD
  비즈니스 불변식: "삭제된 공지는 수정 불가" — 이것뿐
  조회가 95%, 쓰기가 5%
  
  과잉 설계의 결과:
    개발 기간: 단순 구현 대비 3배
    신규 개발자 온보딩: "이 단순한 게 왜 이렇게 복잡하지?"
    "공지 내용 typo 수정" 요청 → 15개 파일 이해 후 수정

  반면 단순 구현이었다면:
    NoticeController, NoticeService, Notice (Entity), NoticeRepository
    → 4개 파일, 2시간 구현
```

---

## ✨ 올바른 접근 (After — 복잡도에 맞는 아키텍처 선택)

```
복잡도 진단 → 아키텍처 선택 → 구현

진단 질문:
  1. 도메인 규칙이 복잡한가? (단순 if문 이상의 비즈니스 로직)
  2. 상태 전이가 복잡한가? (상태가 여러 개이고 전이 규칙이 복잡)
  3. 도메인 전문가와 지속적 협업이 필요한가?
  4. 장기적으로 유지보수할 시스템인가?
  5. 여러 팀이 협업하는가?

모두 "아니오"  → Transaction Script
일부 "예"     → Table Module 또는 부분적 Domain Model
대부분 "예"   → Domain Model (DDD)
```

```java
// 1. Transaction Script — 단순 CRUD (공지사항)
@Service
@Transactional
public class NoticeService {

    public NoticeDto createNotice(CreateNoticeRequest request) {
        Notice notice = Notice.builder()
            .title(request.getTitle())
            .content(request.getContent())
            .authorId(request.getAuthorId())
            .status(NoticeStatus.PUBLISHED)
            .build();
        return noticeRepository.save(notice).toDto();
    }

    public NoticeDto updateNotice(Long id, UpdateNoticeRequest request) {
        Notice notice = noticeRepository.findById(id).orElseThrow();
        if (notice.isDeleted()) throw new IllegalStateException("삭제된 공지는 수정 불가");
        notice.update(request.getTitle(), request.getContent());
        return notice.toDto();
    }
}

// → 단순. 빠르게 개발. 충분함.

// 2. Domain Model (DDD) — 복잡한 주문 도메인
public class Order {  // 복잡한 불변식과 상태 전이가 존재
    
    // 불변식 1: 주문은 최소 1개 이상의 상품
    // 불변식 2: PENDING 상태에서만 취소 가능
    // 불변식 3: 취소 불가 상태(SHIPPED)에서 취소 시도 → 예외
    // 불변식 4: 쿠폰은 주문당 1개만 적용 가능
    // 불변식 5: 포인트 사용 시 보유 포인트 초과 불가
    // ... 이런 규칙이 10개 이상
    
    public void applyCoupon(Coupon coupon) {
        if (this.appliedCoupon != null) {
            throw new CouponAlreadyAppliedException("쿠폰은 주문당 1개만 적용 가능합니다");
        }
        if (!coupon.isApplicable(this.totalAmount())) {
            throw new CouponNotApplicableException("적용 조건 미충족: 최소 " + coupon.minimumAmount());
        }
        this.appliedCoupon = coupon.id();
        this.events.add(new CouponApplied(this.id, coupon.id()));
    }
    
    // 복잡한 상태 전이: PENDING → PAID → PREPARING → SHIPPED → DELIVERED
    public void ship(TrackingNumber trackingNumber) {
        if (this.status != OrderStatus.PAID) {
            throw new OrderNotShippableException("결제 완료된 주문만 배송 처리할 수 있습니다");
        }
        this.status = OrderStatus.SHIPPED;
        this.trackingNumber = trackingNumber;
        this.shippedAt = LocalDateTime.now();
        this.events.add(new OrderShipped(this.id, trackingNumber));
    }
}

// → DDD가 적합. 불변식이 많고, 상태 전이가 복잡하며, 규칙이 자주 변경됨.
```

---

## 🔬 내부 동작 원리

### 1. 아키텍처 패턴 3단계

```
Martin Fowler의 "Patterns of Enterprise Application Architecture" 기반:

─────────────────────────────────────────────────────
Level 1: Transaction Script (절차적 스크립트)
─────────────────────────────────────────────────────
구조:
  Service 메서드 = 하나의 비즈니스 트랜잭션
  Entity는 데이터 보관소 (getter/setter)

적합한 상황:
  - 단순 CRUD (게시판, 설정, 메뉴 관리)
  - 규칙이 단순하거나 거의 없음
  - 단기 프로젝트, 프로토타입

장점: 빠른 개발, 이해가 쉬움
한계: 비즈니스 로직이 복잡해지면 Service가 비대해짐

─────────────────────────────────────────────────────
Level 2: Table Module (테이블 모듈)
─────────────────────────────────────────────────────
구조:
  테이블(또는 뷰) 하나당 클래스 하나
  클래스가 해당 테이블 관련 모든 로직 담당

적합한 상황:
  - 중간 복잡도의 비즈니스 로직
  - 데이터베이스 테이블 구조가 비즈니스와 잘 대응
  - 보고서, 통계, 배치 처리

장점: Transaction Script보다 응집도 높음
한계: 여러 테이블에 걸친 복잡한 비즈니스 규칙 처리 어려움

─────────────────────────────────────────────────────
Level 3: Domain Model (DDD)
─────────────────────────────────────────────────────
구조:
  비즈니스 개념을 객체로 표현
  객체가 자신의 데이터와 행동을 함께 가짐
  Aggregate, Entity, Value Object, Domain Event

적합한 상황:
  - 복잡한 비즈니스 규칙과 불변식
  - 상태 전이가 복잡
  - 도메인 전문가와 지속적 협업
  - 장기 유지보수 필요

장점: 복잡도 증가에 선형적으로 대응, 테스트 용이
한계: 초기 설계 비용, 팀 역량 필요

─────────────────────────────────────────────────────
선택 기준: 도메인 복잡도

  낮음 ──────────────────────────────── 높음
   │                                    │
Transaction Script → Table Module → Domain Model
```

### 2. 도메인 복잡도 측정 기준

```
복잡도 지표:

① 불변식(Invariant) 수
  불변식 = 항상 참이어야 하는 비즈니스 규칙
  
  낮음 (0~2개): "게시글은 제목이 있어야 한다"
  중간 (3~7개): "주문은 최소 1개 상품, 쿠폰 1개, 결제 완료 후 배송"
  높음 (8개+):  "보험 약관 계산: 연령, 직업, 지역, 질병 이력에 따른 복잡한 계산"

② 상태 전이 복잡도
  낮음: 상태 없음 또는 2개 (활성/비활성)
  중간: 3~5개 상태, 전이 규칙 명확
  높음: 6개+ 상태, 조건부 전이, 되돌릴 수 없는 전이

③ 도메인 개념의 풍부함
  낮음: CRUD 위주, 계산 로직 없음
  중간: 할인 계산, 배송비 계산 등 일부 도메인 로직
  높음: 금융 상품 계산, 의료 진단 로직, 물류 최적화 등

④ 변경 빈도와 방향
  낮음: 요구사항이 단순하고 변경 드묾
  높음: 비즈니스 규칙이 자주 변경되고 확장됨

⑤ 이해관계자 수
  낮음: 개발팀만 참여
  높음: 도메인 전문가(의사, 세무사, 물류 전문가 등)와 협업

판단 공식 (단순화):
  복잡도 점수 = 불변식 수 + 상태 수 + 이해관계자 수
  
  10점 미만: Transaction Script
  10~20점:  Table Module 또는 부분적 DDD
  20점 이상: Domain Model (DDD)
```

### 3. "나중에 복잡해지면" 에 대한 현실적 대응

```
흔한 딜레마:
  "지금은 단순하지만, 나중에 요구사항이 복잡해지면
   Transaction Script를 Domain Model로 바꾸기 어렵지 않나?"

현실:
  ① 복잡해지지 않는 시스템이 더 많다
     프로토타입 → 폐기
     사내 도구 → 단순함 유지
     → 과잉 설계를 피했을 때의 절약이 훨씬 큼

  ② 복잡해진다면 — 점진적 리팩터링 가능
     Transaction Script → 비즈니스 로직을 Entity로 이동
     Entity → Aggregate로 격상
     Service → Application Service로 분리
     (완전한 전환보다 핵심 부분만 먼저 적용)

  ③ 완벽한 예측은 불가능
     처음부터 DDD로 짰는데 결국 CRUD로 끝나는 프로젝트도 많음
     → 지금의 복잡도에 맞는 도구를 쓰고, 필요할 때 진화

현실적 전략:
  처음: Transaction Script로 시작
  Service가 200줄 넘어가는 메서드 등장 시 → Entity에 로직 이동 신호
  Entity 이동 후에도 Service에 로직이 쌓이면 → Domain Model 전환 고려
  
  "코드가 아프다는 신호에 반응하는 것이, 처음부터 완벽히 설계하는 것보다 낫다"
```

### 4. 도메인별 복잡도 실례

```
Transaction Script가 적합한 도메인:
  ✅ 사내 공지사항 게시판
  ✅ 설정/메뉴 관리 어드민
  ✅ 단순 파일 업로드/다운로드
  ✅ 로그 조회 시스템
  ✅ 사용자 프로필 (CRUD 위주)

Table Module이 적합한 도메인:
  ✅ 재고 관리 (입출고 기록, 재고 현황)
  ✅ 회계 전표 처리
  ✅ 일정 관리 시스템
  ✅ 간단한 쇼핑몰 (할인, 쿠폰이 단순한)

Domain Model (DDD)이 적합한 도메인:
  ✅ 복잡한 전자상거래 (쿠폰, 포인트, 멤버십, 배송비 정책이 복잡)
  ✅ 금융 상품 (이자 계산, 한도, 상환 스케줄)
  ✅ 의료 예약/진료 시스템
  ✅ 물류/배송 최적화
  ✅ 보험 계약/청구 처리
  ✅ ERP (제조, 구매, 영업 프로세스)

같은 "쇼핑몰"이라도:
  스타트업 초기 MVP   → Transaction Script
  할인 정책 3가지     → 부분 Domain Model
  멤버십 + 쿠폰 + 포인트 + 복잡한 할인 → Domain Model
```

---

## 💻 실전 코드

### 복잡도에 따른 같은 기능의 다른 구현

```java
// 시나리오: "포인트 적립" 기능

// ──────────────────────────────────────────
// Case 1: 단순 → Transaction Script
// "구매 금액의 1% 포인트 적립" 이것뿐
// ──────────────────────────────────────────
@Service
public class PointService {
    public void earnPoints(Long memberId, BigDecimal purchaseAmount) {
        int earnedPoints = purchaseAmount.intValue() / 100;  // 1%
        Member member = memberRepository.findById(memberId).orElseThrow();
        member.setPoints(member.getPoints() + earnedPoints);
        memberRepository.save(member);
    }
}

// ──────────────────────────────────────────
// Case 2: 복잡 → Domain Model
// "회원 등급별 적립률 다름, 이벤트 기간 중 2배 적립,
//  포인트 유효기간 1년, 소멸 예정 포인트 알림,
//  특정 카테고리 구매 시 추가 적립..."
// ──────────────────────────────────────────
public class PointWallet {  // Aggregate Root

    private PointWalletId id;
    private MemberId memberId;
    private List<PointEntry> entries;  // 적립/사용 이력 (Entity)

    // 불변식: 포인트는 음수가 될 수 없다
    // 불변식: 만료된 포인트는 사용 불가
    // 불변식: 한 번의 거래에서 사용 가능한 포인트는 총 잔액의 최대 100%

    public void earn(Point amount, EarnReason reason, LocalDate expiryDate) {
        if (amount.isNegative()) throw new InvalidPointAmountException("적립 금액은 양수여야 합니다");
        PointEntry entry = PointEntry.earned(amount, reason, expiryDate);
        this.entries.add(entry);
        this.events.add(new PointEarned(this.id, this.memberId, amount, reason));
    }

    public void use(Point amount, UseReason reason) {
        Point availableBalance = this.availableBalance();
        if (amount.isGreaterThan(availableBalance)) {
            throw new InsufficientPointException(
                String.format("잔액 부족: 보유 %d, 사용 시도 %d", availableBalance.value(), amount.value())
            );
        }
        // FIFO: 먼저 적립된 포인트부터 사용 (만료일 가까운 것부터)
        deductFromEntries(amount, reason);
        this.events.add(new PointUsed(this.id, this.memberId, amount, reason));
    }

    public Point availableBalance() {
        return entries.stream()
            .filter(PointEntry::isNotExpired)
            .map(PointEntry::remainingAmount)
            .reduce(Point.ZERO, Point::add);
    }
}
```

### 리팩터링 신호 감지

```java
// 이 Service를 보면 Domain Model로의 전환 신호를 읽을 수 있음
@Service
public class OrderService {

    public void processOrder(Long orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();

        // 🚨 신호 1: if/else 체인이 길어짐 → 상태 전이 로직을 Order로 이동
        if ("PENDING".equals(order.getStatus())) {
            if (order.getTotalAmount().compareTo(BigDecimal.ZERO) <= 0) {
                throw new RuntimeException("금액이 0 이하입니다");
            }
            // ...
        } else if ("PAID".equals(order.getStatus())) {
            if (order.getShippingAddress() == null) {
                throw new RuntimeException("배송지가 없습니다");
            }
            // ...
        }

        // 🚨 신호 2: Order 내부 데이터를 Service가 직접 조작
        // order.getItems().add(new OrderItem(...))  ← 불변식 우회
        // order.setStatus("PROCESSING")             ← 전이 규칙 없이 상태 변경

        // 🚨 신호 3: Order에 대한 질문을 Service가 대신 답함
        boolean canCancel = "PENDING".equals(order.getStatus())
            || "PAID".equals(order.getStatus());
        // → order.isCancellable() 로 이동해야 함
    }
}
```

---

## 📊 설계 비교

```
아키텍처별 특성 비교:

                Transaction   Table     Domain
                Script        Module    Model
────────────┼─────────────┼─────────┼──────────────
개발 속도    │ 빠름         │ 중간    │ 느림 (초기)
────────────┼─────────────┼─────────┼──────────────
복잡도 대응  │ 낮음         │ 중간    │ 높음
────────────┼─────────────┼─────────┼──────────────
테스트 용이  │ Service 모킹 │ 중간    │ Entity 단독
            │ 많이 필요    │         │ 테스트 가능
────────────┼─────────────┼─────────┼──────────────
팀 역량 필요 │ 낮음         │ 중간    │ 높음
────────────┼─────────────┼─────────┼──────────────
도메인 표현  │ 절차적       │ 테이블  │ 비즈니스
            │ (how)       │ 중심    │ 언어 (what)
────────────┼─────────────┼─────────┼──────────────
단순 CRUD   │ 최적         │ 적합    │ 과잉
────────────┼─────────────┼─────────┼──────────────
복잡 도메인  │ 유지보수 불가 │ 한계    │ 최적
```

---

## ⚖️ 트레이드오프

```
DDD 적용 판단의 핵심 트레이드오프:

지금 단순하게 → 나중에 복잡해지면 리팩터링 비용
지금 DDD로  → 초기 과잉 설계 비용

이 트레이드오프를 결정하는 요소:
  ① 비즈니스 생존 가능성: MVP라면 빠른 출시가 우선
  ② 도메인 안정성: 규칙이 자주 변한다면 DDD가 유리
  ③ 팀 역량: DDD에 익숙하지 않은 팀에 강제하면 역효과
  ④ 유지보수 기간: 1년 후 폐기 예정이라면 Transaction Script로 충분

혼합 전략 (가장 현실적):
  핵심 도메인(Core) → Domain Model
  지원 도메인(Supporting) → Table Module 또는 부분 Domain Model
  일반 기능(Generic) → Transaction Script 또는 외부 솔루션
  
  예: 쇼핑몰
    주문/결제 처리 (Core) → DDD 적용
    정산 (Supporting)    → Table Module
    공지사항 (Generic)    → Transaction Script
    SMS 발송 (Generic)    → 외부 서비스 (SaaS)
```

---

## 📌 핵심 정리

```
DDD 적용 판단 기준:

도메인 복잡도가 기준:
  불변식 수 + 상태 전이 복잡도 + 이해관계자 수 + 변경 빈도
  → 낮으면 Transaction Script, 높으면 Domain Model

DDD가 과잉인 징후:
  - 게시판 CRUD에 Aggregate Root
  - 단순 계산 로직에 Domain Event
  - 도메인 전문가 없이 "DDD스럽게" 짜는 것
  - 개발 속도보다 "올바른 설계"가 더 중요한 초기 MVP

DDD가 필요한 징후:
  - Service 메서드가 200줄 이상
  - "이 규칙이 어디에 있지?" 를 찾는 데 시간 소요
  - 비즈니스 규칙이 Service 여러 곳에 중복
  - 상태 전이를 if/else 문자열 비교로 처리

현실적 접근:
  ① 지금의 복잡도에 맞는 도구 선택
  ② 코드가 아프다는 신호에 반응 (리팩터링)
  ③ Core Domain에 집중 — 모든 곳을 DDD로 할 필요 없음
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 시스템들의 도메인 복잡도를 평가하고, 어떤 아키텍처 패턴이 적합한지 이유와 함께 답하세요.

```
A. 사내 직원 식단 신청 시스템 (매일 아침 점심 메뉴 선택)
B. 은행 대출 심사 시스템 (신용 점수, 소득, 담보, DSR 계산)
C. 쇼핑몰 배너 관리 어드민 (이미지, 링크, 노출 기간 설정)
D. 병원 수술 일정 관리 (의사 가용 시간, 수술실 배정, 마취과 협업)
```

<details>
<summary>해설 보기</summary>

**A. 직원 식단 신청 → Transaction Script**
- 불변식: "마감 전에만 신청 가능" 정도
- 상태 전이: 신청/취소 2가지
- → 단순 CRUD, Transaction Script로 충분

**B. 은행 대출 심사 → Domain Model (DDD)**
- 불변식 다수: DTI, DSR 한도, 담보 인정 비율, 신용 등급별 한도
- 복잡한 계산: 소득 산정 방식, 담보 평가, 금리 산출
- 도메인 전문가 필수: 심사 기준은 금융 규제 전문가가 결정
- → DDD 적합. `LoanApplication`, `CreditAssessment`, `CollateralValuation` Aggregate 설계 필요

**C. 배너 관리 어드민 → Transaction Script**
- 불변식: "노출 기간이 미래여야 한다" 정도
- 도메인 로직 없음
- → Transaction Script로 충분. Aggregate를 만들 이유 없음

**D. 수술 일정 관리 → Domain Model (DDD)**
- 불변식 다수: 의사 1인당 동시 수술 불가, 수술실 중복 배정 불가, 마취과 동반 필수 수술 유형
- 복잡한 상태 전이: 예약→확정→준비→진행중→완료/취소
- 도메인 전문가 필수: 수술 유형별 규칙, 응급 상황 우선순위
- → DDD 적합. `SurgerySchedule`, `OperatingRoom`, `SurgeryTeam` Aggregate 설계 필요

</details>

---

**Q2.** "Transaction Script로 시작했는데 Service가 500줄이 됐다. 지금 DDD로 전환할 수 있는가?" — 리팩터링 시작점과 우선순위를 어떻게 결정하는가?

<details>
<summary>해설 보기</summary>

**가능합니다. 단, 전면 재작성이 아닌 점진적 이동이 핵심입니다.**

**우선순위 결정 기준:**

```
1. 가장 많이 바뀌는 비즈니스 규칙을 찾는다
   → 변경이 잦은 규칙이 Entity로 이동해야 할 1순위

2. 버그가 가장 많이 발생하는 코드를 찾는다
   → 비즈니스 규칙이 여러 곳에 중복된 증거

3. 테스트하기 가장 어려운 코드를 찾는다
   → Mock이 많이 필요한 Service 로직 → Entity로 이동하면 Mock 제거
```

**점진적 이동 예시:**

```java
// Before: Service에 있는 취소 가능 판단 로직
public void cancelOrder(Long id) {
    Order order = orderRepository.findById(id).orElseThrow();
    if (!"PENDING".equals(order.getStatus()) && !"PAID".equals(order.getStatus())) {
        throw new IllegalStateException("취소 불가");
    }
    order.setStatus("CANCELLED");
}

// Step 1: Order에 메서드 이동 (가장 먼저, 가장 안전)
public class Order {
    public boolean isCancellable() {
        return "PENDING".equals(status) || "PAID".equals(status);
    }
    public void cancel() {
        if (!isCancellable()) throw new OrderNotCancellableException(...);
        this.status = "CANCELLED";
    }
}

// Step 2: String 상태를 Enum으로 (다음 단계)
// Step 3: Domain Event 추가 (그 다음)
// Step 4: 다른 Aggregate 분리 (마지막)
```

한 번에 전부 바꾸려 하면 실패합니다. 2~3주 단위의 작은 단계로, 각 단계마다 테스트를 추가하며 진행합니다.

</details>

---

**Q3.** 팀의 일부만 DDD에 익숙하고 나머지는 Transaction Script에 익숙할 때, 프로젝트에서 어떻게 아키텍처를 결정하는가?

<details>
<summary>해설 보기</summary>

**팀 역량이 아키텍처 선택에 중요한 변수입니다.**

**현실적 전략:**

```
접근 1: 혼합 적용 (권장)
  Core Domain → DDD 경험자가 주도해서 설계
  나머지 → Transaction Script 또는 Table Module
  
  → 전체 팀에 DDD를 강요하지 않음
  → DDD 경험자가 Core의 모범 예시를 만들고 공유
  → 팀 전체가 점진적으로 학습

접근 2: 학습 기간 설정
  스프린트 1~2개를 "DDD 패턴 학습" 전용으로 투자
  페어 프로그래밍으로 DDD 경험자와 비경험자 짝 맞춤

피해야 할 접근:
  ❌ "DDD를 모르면 코드를 짤 수 없다"는 규칙
     → 팀 생산성 급감, 갈등 발생
  
  ❌ DDD 경험자만 핵심 코드를 짜는 구조
     → 버스 팩터(핵심 인물 의존도) 문제
     → 지식 공유 차단
```

가장 중요한 것: **팀이 이해하고 유지할 수 있는 수준의 설계**가 최고의 설계입니다. 아무리 완벽한 DDD 설계라도 팀이 이해하지 못하면 무너집니다.

</details>

---

<div align="center">

**[⬅️ 이전: Strategic Design vs Tactical Design](./03-strategic-vs-tactical.md)** | **[홈으로 🏠](../README.md)** | **[다음: DDD와 레이어드 아키텍처 비교 ➡️](./05-ddd-vs-layered.md)**

</div>
