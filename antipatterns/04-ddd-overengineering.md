# DDD 과잉 적용 — CRUD에 DDD를 적용했을 때

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 단순 CRUD에 DDD를 적용하면 어떤 과잉 복잡도가 생기는가?
- 도메인 복잡도에 따른 설계 수준(Transaction Script → Table Module → Domain Model)은?
- "언제 DDD를 내려놓아야 하는가?"를 판단하는 기준은?
- DDD를 적용하지 않고도 유지보수 가능한 코드를 작성하는 방법은?
- 시스템 일부에만 DDD를 적용하는 선택적 적용 전략은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

DDD는 강력한 도구지만 모든 곳에 적용해야 하는 도구가 아니다. 간단한 게시판, 사용자 설정 페이지, 관리자 메뉴 편집 같은 단순 CRUD에 Aggregate, Repository, Domain Event, Outbox Pattern을 모두 적용하면 오히려 복잡도만 늘어난다.

"올바른 도구를 올바른 곳에 사용하는 것"이 진짜 설계 역량이다.

---

## 😱 흔한 실수 (Before — CRUD에 DDD 과잉 적용)

```java
// 단순한 "공지사항 등록" 기능에 DDD 전체 적용
// → 공지사항은 제목, 내용, 작성자, 날짜의 단순 CRUD

// Aggregate: 필요 없음 → 복잡성만 추가
public class Notice extends AggregateRoot {
    private NoticeId id;
    private Title title;          // Value Object for simple String
    private Content content;      // Value Object for simple String
    private AuthorId authorId;
    private NoticeStatus status;  // DRAFT, PUBLISHED → 불변식 없음

    public static Notice create(Title title, Content content, AuthorId authorId) {
        Notice notice = new Notice();
        notice.registerEvent(new NoticeCreated(notice.id, title, content));
        return notice;
    }

    public void publish() {
        if (status != NoticeStatus.DRAFT) throw new AlreadyPublishedException();
        this.status = NoticeStatus.PUBLISHED;
        registerEvent(new NoticePublished(this.id));
    }
}

// Repository 인터페이스: 필요 없음
public interface NoticeRepository {
    void save(Notice notice);
    Optional<Notice> findById(NoticeId id);
    List<Notice> findAllPublished();
}

// JPA 구현체: 필요 없음
@Repository
public class JpaNoticeRepository implements NoticeRepository { ... }

// Application Service: 불필요한 레이어
@Service
public class NoticeApplicationService {
    public NoticeId createNotice(CreateNoticeCommand command) {
        Notice notice = Notice.create(new Title(command.title()), ...);
        noticeRepository.save(notice);
        notice.pullDomainEvents().forEach(publisher::publishEvent);
        return notice.id();
    }
}

// 이벤트 핸들러: 필요 없음 (공지사항 생성에 이벤트 반응 없음)
@EventListener
public void on(NoticeCreated event) {
    // 아무것도 하지 않음 — 이벤트가 불필요
}
```

```
현실:
  공지사항 저장: @Service + @Repository (JpaRepository) 로 충분
  5줄이면 될 것을 100줄로 만든 과잉 설계
  "Title Value Object"가 왜 있는지 신규 개발자가 모름
```

---

## ✨ 올바른 접근 (After — 도메인 복잡도에 맞는 설계)

```java
// 단순 CRUD에는 Transaction Script 또는 Table Module
// 공지사항 — 비즈니스 규칙이 거의 없음

// Spring Data JPA 직접 사용 (충분함)
public interface NoticeJpaRepository extends JpaRepository<NoticeEntity, Long> {
    List<NoticeEntity> findByStatusOrderByCreatedAtDesc(NoticeStatus status);
    Page<NoticeEntity> findAll(Pageable pageable);
}

// 단순 Service (Transaction Script)
@Service
@Transactional
public class NoticeService {

    public Long createNotice(String title, String content, Long authorId) {
        // 간단한 검증
        if (title == null || title.isBlank()) throw new IllegalArgumentException("제목 필요");
        if (title.length() > 200) throw new IllegalArgumentException("제목 200자 이하");

        NoticeEntity entity = new NoticeEntity(title, content, authorId);
        return noticeRepository.save(entity).getId();
    }

    public void publishNotice(Long noticeId) {
        NoticeEntity notice = noticeRepository.findById(noticeId).orElseThrow();
        notice.setStatus(NoticeStatus.PUBLISHED);
        notice.setPublishedAt(LocalDateTime.now());
        // JPA 더티 체킹 → 자동 저장
    }

    @Transactional(readOnly = true)
    public List<NoticeResponse> getPublishedNotices() {
        return noticeRepository.findByStatusOrderByCreatedAtDesc(NoticeStatus.PUBLISHED)
            .stream().map(NoticeResponse::from).collect(toList());
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 도메인 복잡도에 따른 설계 스펙트럼

```
Martin Fowler의 세 가지 아키텍처 패턴:

1. Transaction Script (절차적)
   
   적합한 도메인:
     비즈니스 규칙이 거의 없음
     단순 CRUD (공지사항, 설정, 로그)
     비즈니스 로직의 재사용이 거의 없음
   
   특징:
     Service 메서드가 직접 DB를 조작
     객체는 단순 데이터 홀더
     빠른 개발, 낮은 복잡도
   
   예: 공지사항, FAQ, 배너 관리, 관리자 설정

2. Table Module (테이블 중심)
   
   적합한 도메인:
     중간 수준의 비즈니스 로직
     보고서, 집계, 조회 중심
   
   특징:
     테이블 단위로 클래스 구성
     각 클래스가 해당 테이블의 CRUD + 규칙
   
   예: 매출 보고서, 통계 집계, 이력 조회

3. Domain Model (도메인 중심 — DDD)
   
   적합한 도메인:
     복잡한 비즈니스 규칙
     여러 객체에 걸친 불변식
     풍부한 도메인 로직이 비즈니스 차별화
   
   특징:
     Entity에 로직 집중
     Repository, Domain Event, Saga
   
   예: 주문, 결제, 재고, 보험 계약, 금융 상품

선택 기준:
  "이 도메인의 비즈니스 로직이 경쟁 우위를 만드는가?"
  → YES (Core Domain): Domain Model
  → NO (Supporting): 단순하게 시작, 복잡해지면 이동
  → NO (Generic): Transaction Script 또는 외부 솔루션
```

### 2. DDD가 과잉인 신호

```
과잉 적용 체크리스트:

□ Entity에 불변식이 없다
  (setter로 모든 필드를 자유롭게 변경해도 됨)
  → DDD 불필요

□ Domain Event를 발행하지만 핸들러가 없다
  (이벤트를 발행하는데 아무도 구독 안 함)
  → 이벤트 불필요

□ Repository 인터페이스가 JpaRepository와 동일하다
  (findById, save, findAll 외에 도메인 메서드 없음)
  → 도메인 Repository 불필요

□ Domain Service가 단순 CRUD 작업만 한다
  (데이터 읽고, 다른 Service에 넘기고, 저장하는 것만)
  → Domain Service 불필요

□ Value Object가 단순 String 래핑이다
  (NoticeTitle("제목") - 검증도 없고 로직도 없는)
  → Value Object 불필요 (일반 String으로)

□ Aggregate에 항상 단 하나의 Entity만 있다
  (Order + OrderLine 없이 Order만)
  → Aggregate 불필요 (단순 Entity)

□ Application Service가 Repository 호출만 한다
  (find → method call → save → event publish)
  이외의 도메인 로직 없음
  → Application Service 불필요
```

### 3. 선택적 DDD 적용 전략

```
시스템을 세 영역으로 구분:

Core Domain (핵심 — DDD 완전 적용):
  비즈니스 차별화 요소
  복잡한 불변식과 상태 전이
  
  전자상거래: 주문 처리, 가격 정책, 재고 관리
  금융: 대출 심사, 포트폴리오 관리
  물류: 배송 경로 최적화, 화물 추적

Supporting Domain (지원 — 적당히):
  핵심을 지원하지만 차별화 요소 아님
  중간 복잡도
  
  전자상거래: 배송 조회, 쿠폰 발급, 포인트 관리
  → 일부 DDD (Entity에 불변식) + 일부 Transaction Script

Generic Domain (범용 — 최소 또는 외부):
  어느 시스템에나 있는 기능
  비즈니스 차별화 없음
  
  전자상거래: 공지사항, 이메일 발송, 파일 업로드, 로그
  → Transaction Script 또는 외부 SaaS

구현:
  com.example.order/ ← DDD (Aggregate, Repository, Event)
  com.example.shipping/ ← 경량 DDD (일부 Entity에 불변식)
  com.example.notice/ ← Transaction Script (단순 CRUD)
  com.example.email/ ← 외부 서비스 직접 호출
```

---

## 💻 실전 코드

### 같은 기능, 다른 복잡도

```java
// Transaction Script — 공지사항 (단순 CRUD)
@Service
@Transactional
public class NoticeService {
    public Long create(String title, String content, Long authorId) {
        // 간단 검증
        Assert.hasText(title, "제목 필요");
        Assert.isTrue(title.length() <= 200, "제목 200자 이하");
        // 저장
        return noticeRepository.save(new Notice(title, content, authorId)).getId();
    }
}

// Domain Model — 주문 (복잡한 비즈니스)
@Service
@Transactional
public class OrderApplicationService {
    public OrderId placeOrder(PlaceOrderCommand command) {
        // 여러 비즈니스 규칙 조율
        validateInventory(command.lines());
        Customer customer = customerRepository.findById(command.customerId()).orElseThrow();
        Discount discount = pricingService.calculateDiscount(customer, command.coupon(), command.baseAmount());
        Money shippingFee = shippingService.calculateFee(command.address(), discount.base());
        Order order = Order.place(command.customerId(), command.lines(), discount, shippingFee);
        orderRepository.save(order);
        order.pullDomainEvents().forEach(publisher::publishEvent);
        return order.id();
    }
}
```

---

## 📊 설계 비교

```
도메인 복잡도별 설계:

                CRUD                 지원 도메인          핵심 도메인
────────────┼──────────────────┼──────────────────┼──────────────────
설계 패턴    │ Transaction Script│ 경량 DDD          │ 완전한 DDD
────────────┼──────────────────┼──────────────────┼──────────────────
불변식      │ 없음 (단순 검증)  │ 일부 있음         │ 복잡한 불변식
────────────┼──────────────────┼──────────────────┼──────────────────
Entity 로직 │ getter/setter    │ 일부 메서드        │ 풍부한 메서드
────────────┼──────────────────┼──────────────────┼──────────────────
Domain Event│ 불필요           │ 일부 (통합 시)    │ 핵심 메커니즘
────────────┼──────────────────┼──────────────────┼──────────────────
테스트 전략  │ 통합 테스트 충분  │ 혼합             │ 단위 테스트 중심
────────────┼──────────────────┼──────────────────┼──────────────────
개발 속도   │ 빠름             │ 중간             │ 초기 느림, 장기 빠름
```

---

## ⚖️ 트레이드오프

```
DDD 과잉 적용의 실제 비용:
  신규 팀원 온보딩 어려움
    "이 Value Object가 왜 있어요?" → 의도 불명확
  
  코드량 증가 → 유지보수 부담
    간단한 기능이 10개 파일로 구성
  
  팀 생산성 저하
    간단한 수정도 Aggregate, Event, Handler 모두 수정

DDD 과소 적용의 비용:
  Core Domain에 Transaction Script 사용 시
  → 비즈니스 로직이 Service에 분산
  → 불변식 보호 없음
  → 점점 복잡해지면서 수정 불가능한 스파게티

균형 찾기:
  각 도메인의 복잡도를 정기적으로 재평가
  Simple → Complex로 전환은 Strangler Fig로
  Complex → Simple 단순화는 드물지만 불필요한 추상화 제거로
```

---

## 📌 핵심 정리

```
DDD 과잉 적용 핵심:

DDD가 맞는 곳:
  복잡한 비즈니스 규칙 (Core Domain)
  여러 객체에 걸친 불변식
  도메인 로직이 경쟁 우위

DDD가 과잉인 곳:
  단순 CRUD (공지사항, 설정, 로그)
  불변식이 없는 Entity
  비즈니스 로직이 "저장 → 조회"만인 경우

설계 선택지:
  Transaction Script: 단순 CRUD
  경량 DDD: 일부 불변식 있는 지원 도메인
  완전한 DDD: Core Domain

과잉 적용 신호:
  Domain Event 핸들러가 없음
  Value Object가 검증만 있음
  Aggregate에 하나의 Entity만
  Application Service가 Repository 호출만

원칙: "설계 복잡도는 도메인 복잡도를 따라야 한다"
      단순한 도메인을 복잡하게 만드는 것도 나쁜 설계
```

---

## 🤔 생각해볼 문제

**Q1.** 처음에 단순 CRUD로 시작한 "쿠폰" 기능이 점점 복잡해졌다. Transaction Script에서 DDD로 전환하는 시점과 방법은?

<details>
<summary>해설 보기</summary>

**복잡도 임계점을 감지하고 점진적으로 전환합니다.**

```
전환 신호:
  □ 쿠폰 검증 로직이 여러 Service에 중복됨
  □ "이미 사용된 쿠폰인가?" 체크가 4곳 이상에 있음
  □ 쿠폰 상태 전이 규칙(발급→활성→사용→만료)이 Service에 산재
  □ 쿠폰 버그가 잦고 원인 파악이 어려움

전환 방법 (점진적):
  Step 1: 중복 검증 로직 → Coupon Entity 메서드로
  boolean isUsable() { return status == ACTIVE && !isExpired(); }

  Step 2: 상태 변경 → 도메인 메서드로
  coupon.use(orderId)  // 불변식 검증 포함

  Step 3: 상태 전이 enum 추가
  CouponStatus.ISSUED.canTransitionTo(CouponStatus.USED)

  Step 4: 필요 시 Domain Event 추가
  CouponUsed 이벤트 → 사용 이력 기록, 분석 데이터 전송

전환 완료 기준:
  "쿠폰 검증 로직이 Coupon 클래스 하나에 집중됐는가?"
  → YES: 전환 완료
```

</details>

---

**Q2.** 팀장이 "우리 모든 코드에 DDD를 적용하자"고 한다. 어떻게 설득하는가?

<details>
<summary>해설 보기</summary>

**비즈니스 가치와 비용으로 설득합니다.**

```
설득 전략:

1. 구체적인 비용 계산:
   "공지사항 CRUD 기능에 DDD 적용 시:
    파일 수: 2개 → 10개
    개발 시간: 2시간 → 1일
    신규 입사자 이해 시간: 10분 → 2시간"

2. 선택적 적용 제안:
   "핵심 비즈니스(주문, 결제)에는 DDD 완전 적용 동의
    공지사항, 설정 등 단순 CRUD는 Transaction Script 제안"

3. 기회비용 설명:
   "CRUD에 DDD 적용하는 시간 = 핵심 도메인 품질 향상 시간"

4. 단계적 접근 제안:
   "6개월 후 복잡도를 재평가해서 DDD가 필요하면 전환"

핵심 메시지:
  "DDD는 복잡한 도메인을 다루는 도구입니다.
   복잡하지 않은 도메인에 쓰는 것은 망치로 나사를 박는 것입니다."
```

</details>

---

**Q3.** "Transaction Script는 나쁜 설계"라는 편견을 어떻게 바로잡는가?

<details>
<summary>해설 보기</summary>

**설계 패턴에는 본질적으로 나쁜 것이 없고, 맥락에 맞지 않는 적용이 나쁜 것입니다.**

```
바로잡기:

Transaction Script의 올바른 위치:
  복잡도가 낮은 도메인에서는 최선의 선택
  빠른 개발, 낮은 인지 부하, 명확한 흐름

Transaction Script가 나쁜 경우:
  복잡한 비즈니스 로직에 적용 시
  → 비즈니스 규칙이 여러 메서드에 중복
  → 불변식 보호 없음
  → 변경 시 모든 메서드 찾아야 함

예시로 설득:
  "WordPress의 공지사항 플러그인에 DDD를 쓰겠습니까?"
  → 물론 아님. Transaction Script로 충분
  
  "은행의 대출 심사 시스템에 Transaction Script를 쓰겠습니까?"
  → 물론 아님. Domain Model이 필수

결론:
  "적합한 도구를 적합한 곳에"
  Transaction Script = 단순 도메인의 올바른 도구
  Domain Model = 복잡한 도메인의 올바른 도구
```

</details>

---

<div align="center">

**[⬅️ 이전: Context 경계 실수](./03-context-boundary-mistakes.md)** | **[홈으로 🏠](../README.md)** | **[다음: 실무에서의 타협 ➡️](./05-practical-tradeoffs.md)**

</div>
