# 서브도메인 분류 — Core / Supporting / Generic

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Core / Supporting / Generic Subdomain의 차이는 무엇인가?
- 왜 Core Domain에 가장 뛰어난 개발자와 자원을 집중해야 하는가?
- Supporting과 Generic Subdomain에 과잉 투자하면 어떤 기회비용이 발생하는가?
- 서브도메인 분류가 팀 구성, 기술 선택, 아웃소싱 결정에 어떻게 영향을 미치는가?
- 우리 회사의 서브도메인을 어떻게 분류하고, 우선순위를 어떻게 결정하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

모든 기능이 똑같이 중요하지 않다. 그런데 많은 팀이 SMS 발송 시스템을 직접 구현하느라 3개월을 쓰고, 핵심 추천 알고리즘에는 1개월밖에 투자하지 못한다. SMS 발송은 어디서나 비슷하게 구현되는 Generic 기능이고, 추천 알고리즘이야말로 경쟁사와의 차이를 만드는 Core이다.

서브도메인 분류는 **"무엇에 집중할 것인가"** 를 결정하는 전략적 도구다. Core에는 최고의 개발자와 DDD를 적용하고, Generic에는 외부 솔루션을 쓰는 것이 올바른 자원 배분이다.

---

## 😱 흔한 실수 (Before — 모든 기능을 동등하게 취급)

```
상황: 음식 배달 플랫폼 개발

잘못된 자원 배분:
  ① SMS/알림 시스템: 팀 내 시니어 개발자 2명, 2개월 투자
     자체 구현: 발송 큐, 재시도, 통신사별 연동, 수신 확인 추적...

  ② 이메일 발송 시스템: 주니어 1명, 1개월 투자
     자체 구현: 템플릿 엔진, 수신 거부 관리, 바운스 처리...

  ③ 회원 가입/로그인: 팀 전체 3명, 3개월 투자
     자체 구현: 비밀번호 암호화, OAuth, 2FA, 세션 관리...

  ④ 배달 경로 최적화 알고리즘: 시니어 1명, 2개월 투자
     (이것이 경쟁 우위의 핵심인데 투자가 적음)

  ⑤ 실시간 배달 현황 추적: 팀 전체 2명, 3개월 투자
     (이것도 핵심 차별화 요소인데 뒤늦게 시작)

결과:
  SMS, 이메일, 회원 관리에 총 8개월 투자
  → AWS SNS, SendGrid, Firebase Auth를 쓰면 2주면 됐을 기능
  → 이 시간에 배달 최적화와 실시간 추적을 더 잘 만들 수 있었음

  경쟁사가 먼저 "AI 기반 배달 경로 최적화" 출시
  → 우리는 SMS 시스템 유지보수에 바쁨
```

---

## ✨ 올바른 접근 (After — 서브도메인 분류 기반 자원 배분)

```
음식 배달 플랫폼의 서브도메인 분류:

Core Domain (핵심):
  ① 배달 경로 최적화
     → 라이더의 현재 위치, 교통상황, 음식 준비 시간을 고려한 최적 배정
     → 경쟁사 대비 배달 속도 20% 향상이 핵심 지표
     → 외부 솔루션 없음, 반드시 자체 구현
     → 최고 시니어 2명 + DDD + 정교한 알고리즘 투자

  ② 실시간 배달 현황 추적
     → 라이더 위치를 초 단위로 고객에게 전달
     → 고객 만족도와 직결 (재주문율에 영향)
     → 독자적 WebSocket + 위치 보정 알고리즘 필요
     → Core로 분류, 시니어 개발자 집중

Supporting Domain (지원):
  ③ 음식점 관리
     → 메뉴 등록, 영업 시간, 주문 접수 관리
     → 경쟁 우위보다 운영 효율이 목적
     → 기능 자체는 중요하나 차별화 요소 아님
     → 중간 수준 투자, 표준 패턴으로 구현

  ④ 정산 시스템
     → 라이더/음식점에 대금 지급
     → 정확성이 중요하지만 차별화 요소 아님
     → Supporting Domain으로 분류

Generic Domain (일반):
  ⑤ 회원 가입/인증 → Firebase Auth 또는 Keycloak
  ⑥ SMS/알림 발송  → AWS SNS 또는 Twilio
  ⑦ 이메일 발송    → SendGrid 또는 AWS SES
  ⑧ 결제 처리      → Toss Payments, KG이니시스 등 PG사 연동
  ⑨ 지도/위치 서비스 → Google Maps API, Kakao Maps API

자원 배분 결과:
  Core (배달 최적화 + 추적): 시니어 4명, 전체 개발 시간의 60%
  Supporting (음식점 + 정산): 중간 2명, 30%
  Generic: 외부 서비스 사용, 연동만, 10%

  → 6개월 후: 배달 속도 업계 최고, 추적 정확도 99.9%
  → SMS/이메일에 시간 낭비 없음
```

---

## 🔬 내부 동작 원리

### 1. 세 가지 서브도메인의 정의

```
Core Domain (핵심 도메인):
  정의: 비즈니스가 다른 경쟁자보다 잘해야 하는 영역
        소프트웨어가 존재하는 이유 그 자체
  
  특징:
    - 경쟁 우위의 원천
    - 외부 솔루션으로 대체 불가 (차별화가 사라짐)
    - 가장 복잡한 도메인 로직이 집중됨
    - 지속적으로 진화하고 개선되어야 함
  
  투자 방침:
    → 가장 뛰어난 개발자
    → DDD 적용 (Aggregate, Domain Event 등)
    → 충분한 시간과 리소스
    → 도메인 전문가와의 긴밀한 협업
  
  예시:
    전자상거래: 상품 추천 알고리즘, 개인화 검색
    금융: 리스크 평가 모델, 사기 탐지 알고리즘
    물류: 배차 최적화, 경로 계산
    게임: 매칭 알고리즘, 밸런싱 시스템

─────────────────────────────────────────────

Supporting Domain (지원 도메인):
  정의: Core를 지원하기 위해 필요하지만,
        Core만큼 차별화가 중요하지 않은 영역
  
  특징:
    - Core와 달리 "잘 하면" 되는 수준 (최고가 아니어도)
    - 외부 솔루션이 있지만 비즈니스 특성에 맞게 커스터마이징 필요
    - 비교적 안정적인 도메인 로직
  
  투자 방침:
    → 중간 수준의 개발자
    → 적절한 수준의 설계 (Transaction Script 또는 부분 DDD)
    → 외부 솔루션 부분 활용 고려
  
  예시:
    전자상거래: 상품 카탈로그 관리, 재고 관리
    배달: 음식점 관리, 정산 처리
    HR 시스템: 근태 관리, 휴가 관리

─────────────────────────────────────────────

Generic Domain (일반 도메인):
  정의: 비즈니스에 특화되지 않은 일반적 문제를 해결하는 영역
        모든 회사에서 비슷하게 필요한 기능
  
  특징:
    - 외부 SaaS/오픈소스로 완전히 대체 가능
    - 자체 구현 시 외부 솔루션보다 나을 것이 없음
    - 차별화와 무관
  
  투자 방침:
    → 외부 솔루션(SaaS, 오픈소스) 사용
    → 연동(Adapter) 개발만 투자
    → 직접 구현은 피함
  
  예시:
    인증/인가: Firebase Auth, Keycloak, Auth0
    이메일: SendGrid, AWS SES, Mailgun
    SMS: AWS SNS, Twilio, NHN Cloud
    결제: Toss Payments, PG사 SDK
    파일 저장: AWS S3, GCS
    모니터링: Datadog, New Relic
```

### 2. 분류 기준 — "이 기능을 잘 해야 경쟁에서 이기는가?"

```
분류 질문 흐름:

Q1: 이 기능을 경쟁사보다 잘 하면 비즈니스 성과가 달라지는가?
  YES → Core Domain 후보
  NO  → Q2로

Q2: 이 기능이 없으면 Core가 동작하지 않는가?
  YES → Supporting Domain
  NO  → Q3로

Q3: 외부 솔루션으로 완전히 대체 가능한가?
  YES → Generic Domain → 외부 솔루션 사용
  NO  → Supporting Domain 재검토

판단 예시 — 온라인 교육 플랫폼:

  영상 강의 스트리밍:
    Q1: 경쟁사보다 잘 하면? → 버퍼링 없는 스트리밍 = 만족도
    → 하지만 이미 AWS CloudFront, Cloudflare Stream이 있음
    → Generic Domain (외부 CDN 사용)

  강의 진도 추적 및 학습 분석:
    Q1: 경쟁사보다 잘 하면? → 더 나은 학습 경험 → YES
    → Core Domain
    → 어떤 영상 구간에서 이탈하는지, 어떤 순서로 학습할 때 완강률이 높은지
    → 자체 분석 엔진 필요

  수강생 커뮤니티:
    Q1: 경쟁사보다 잘 하면? → 약간의 차이는 있지만 핵심 차별화 아님
    Q2: 없으면 Core 동작 안 하는가? → 아님
    Q3: 외부 솔루션? → Discourse, Slack 연동으로 충분
    → Generic Domain

  강사 수익 정산:
    Q1: 경쟁사보다 잘 하면? → 강사 유치에 도움은 되지만 핵심 아님
    Q2: 없으면? → 강사가 수익을 못 받으므로 서비스 운영 불가
    → Supporting Domain
```

### 3. Core Domain이 잘못 분류됐을 때의 결과

```
실패 패턴 1: Core를 Generic으로 잘못 분류

  회사: 신용 대출 스타트업
  
  잘못된 판단:
    "신용 평가 로직? 외부 신용평가 기관 API 쓰면 되잖아"
    → 나이스평가, KCB API 연동으로 처리
    → 개발 3주 만에 완성
  
  3년 후:
    경쟁사: 자체 머신러닝 기반 신용 평가 모델 개발
            거래 패턴, 소비 습관, 비재무 데이터까지 반영
            기존 신용평가 기관이 놓치는 "씬파일러" 대출 가능
    우리: 여전히 외부 API 사용
          = 경쟁사와 동일한 심사 기준, 차별화 없음
  
  결론: 신용 평가가 Core Domain이었는데 Generic으로 처리
        → 핵심 경쟁력 상실

실패 패턴 2: Generic을 Core처럼 취급

  회사: 이커머스 플랫폼
  
  잘못된 판단:
    "결제 시스템을 자체 구현하면 수수료를 아낄 수 있다"
    → PCI DSS 인증, 카드사 연동, 가상계좌, 에스크로...
    → 시니어 3명, 1년 투자
  
  결과:
    Toss Payments 수수료: 0.3~1.5%
    자체 구현 비용: 인건비 3명 × 1년 = 약 3억 원
    3억을 아끼려다 3억을 씀
    그 동안 상품 추천 알고리즘과 검색 개선은 후순위로 밀림
    경쟁사가 더 나은 개인화 추천으로 시장 점유율 상승

올바른 분류가 가져오는 효과:
  자원 집중: 가장 중요한 곳에 최고의 자원
  시간 절약: Generic은 연동만, 직접 구현 없음
  경쟁력: Core에서 경쟁사를 앞지름
  유지보수: Generic은 외부 솔루션이 알아서 개선
```

### 4. 서브도메인과 팀 구조의 관계

```
Conway's Law: "시스템 구조는 그것을 만든 조직의 커뮤니케이션 구조를 따른다"

서브도메인 분류 → 팀 구조 결정:

Core Domain팀:
  구성: 시니어 중심, 도메인 전문가 포함
  목표: 경쟁 우위 창출, 지속적 혁신
  업무: 새 알고리즘 개발, 도메인 모델 정교화
  KPI: 핵심 비즈니스 지표 (배달 시간, 추천 CTR, 전환율)

Supporting Domain팀:
  구성: 중간 수준 개발자, 안정적 운영 능력
  목표: Core를 안정적으로 지원
  업무: 기능 개발 및 운영, 성능 최적화
  KPI: 안정성, 처리량, 오류율

Generic Domain:
  팀 없음 또는 인프라팀이 연동 담당
  목표: 빠른 연동, 안정적 운영
  업무: SaaS 계약, SDK 연동, 장애 대응
  KPI: 가용성, 비용 효율

---

현실에서의 주의사항:
  ① Core Domain은 시간이 지나면 바뀔 수 있음
     스타트업 초기에는 "사용자 획득"이 Core
     성장 후에는 "이탈 방지"나 "수익화"가 Core가 될 수 있음
     → 주기적으로 서브도메인 분류를 재검토
  
  ② 팀 규모에 따라 유연하게 적용
     5인 스타트업: Core 하나에 집중, Generic은 모두 SaaS
     100인 기업: Core팀 분리, Supporting팀 분리
```

---

## 💻 실전 코드

### 서브도메인 분류가 코드 구조에 반영되는 방식

```java
// Core Domain — 가장 정교한 설계 적용

// 배달 배정 최적화 (Core)
public class DeliveryAssignment {  // Aggregate Root

    private AssignmentId id;
    private OrderId orderId;
    private RiderId riderId;
    private OptimizationScore score;  // 배정 최적화 점수 (Value Object)

    // 핵심 비즈니스 규칙: 라이더 상태, 거리, 교통, 음식 준비 시간을 모두 고려
    public static DeliveryAssignment optimize(
            Order order,
            List<AvailableRider> availableRiders,
            TrafficCondition traffic,
            FoodPreparationTime prepTime) {

        return availableRiders.stream()
            .map(rider -> AssignmentCandidate.calculate(order, rider, traffic, prepTime))
            .max(Comparator.comparing(AssignmentCandidate::score))
            .map(candidate -> new DeliveryAssignment(order.id(), candidate.riderId(), candidate.score()))
            .orElseThrow(() -> new NoAvailableRiderException(order.id()));
    }
}

// Supporting Domain — 적절한 수준의 설계
// 음식점 메뉴 관리 (Supporting) — 간단한 CRUD + 약간의 규칙
@Service
@Transactional
public class RestaurantMenuService {

    public void addMenu(RestaurantId restaurantId, AddMenuRequest request) {
        Restaurant restaurant = restaurantRepository.findById(restaurantId).orElseThrow();
        // 규칙: 음식점당 최대 200개 메뉴
        if (restaurant.getMenuCount() >= 200) {
            throw new MenuLimitExceededException("최대 200개 메뉴까지 등록 가능합니다");
        }
        restaurant.addMenu(request.toMenu());
    }
}

// Generic Domain — 외부 연동 Adapter만 작성
// SMS 발송 (Generic) — AWS SNS 연동
@Component
public class AwsSnsNotificationAdapter implements NotificationPort {

    private final AmazonSNS amazonSNS;

    @Override
    public void sendSms(PhoneNumber to, String message) {
        PublishRequest request = new PublishRequest()
            .withPhoneNumber(to.value())
            .withMessage(message);
        amazonSNS.publish(request);
        // 3rd party 서비스가 재시도, 전송 확인, 통신사별 처리를 담당
    }
}
```

### 서브도메인 분류 의사결정 문서 예시

```markdown
# Subdomain Classification — 배달 플랫폼

## Core Domain
| 서브도메인 | 분류 이유 | 담당 팀 | 기술 접근 |
|----------|---------|--------|---------|
| 배달 경로 최적화 | 배달 속도가 핵심 KPI. 외부 솔루션은 우리 데이터 특성 반영 불가 | 플랫폼 코어팀 | DDD, 자체 알고리즘 |
| 실시간 위치 추적 | 고객 만족도 직결. 정확도와 지연 시간이 차별화 | 플랫폼 코어팀 | DDD, 자체 구현 |

## Supporting Domain
| 서브도메인 | 분류 이유 | 담당 팀 | 기술 접근 |
|----------|---------|--------|---------|
| 음식점 관리 | Core 지원에 필수이나 차별화 아님 | 파트너팀 | 표준 CRUD + 일부 도메인 로직 |
| 라이더 정산 | 운영 필수이나 차별화 아님 | 정산팀 | Table Module |

## Generic Domain
| 서브도메인 | 사용 솔루션 | 이유 |
|----------|----------|-----|
| SMS/알림 | AWS SNS | 업계 표준, 수동 구현 대비 비용/품질 우위 |
| 이메일 | SendGrid | 발송 최적화, 수신 거부 관리 내장 |
| 인증 | Firebase Auth | OAuth, 2FA, 세션 관리 내장 |
| 결제 | Toss Payments | PCI DSS 인증, 카드사 연동 내장 |
```

---

## 📊 설계 비교

```
서브도메인별 투자 수준 비교:

                Core           Supporting     Generic
────────────┼──────────────┼─────────────┼─────────────────
개발자 수준  │ 최고 시니어   │ 중간         │ 없음 (연동만)
────────────┼──────────────┼─────────────┼─────────────────
아키텍처     │ DDD           │ Table Module │ SaaS 연동
            │ (Aggregate,  │ 또는 부분 DDD │ + Adapter
            │ Domain Event) │             │
────────────┼──────────────┼─────────────┼─────────────────
자원 배분    │ 50~60%        │ 30~40%      │ 5~10%
────────────┼──────────────┼─────────────┼─────────────────
목표         │ 경쟁 우위 창출 │ 안정적 지원  │ 빠른 연동
────────────┼──────────────┼─────────────┼─────────────────
자체 구현    │ 필수          │ 선택적       │ 최소화 (피함)
────────────┼──────────────┼─────────────┼─────────────────
KPI         │ 비즈니스 성과 │ 안정성/처리량 │ 비용/가용성
```

---

## ⚖️ 트레이드오프

```
서브도메인 분류의 어려움:

① 분류 자체가 어렵다
   "이게 Core인지 Supporting인지 어떻게 아는가?"
   → 비즈니스 전략과 연결되어야 함
   → 경영진, 도메인 전문가, 개발팀 공동 결정
   → 처음엔 틀릴 수 있음 → 주기적 재검토

② Core는 시간이 지나면 Supporting이 될 수 있음
   스마트폰 초기: 터치 인터페이스가 Core
   지금: 터치는 Generic (당연한 것)
   → Core가 Generic으로 상품화(Commoditization)되는 순간을 포착해야 함

③ Generic을 직접 구현해야 하는 경우
   규제: 금융 데이터를 외부 SaaS에 넣을 수 없는 경우
   보안: 외부 서비스에 민감한 데이터 전송 금지
   비용: SaaS 비용이 자체 구현보다 크게 높은 경우
   → 이 경우 Generic을 자체 구현하되 최소한의 투자로

현실적 조언:
  "이것이 우리의 Core인가?" 라는 질문을 팀이 항상 의식할 것
  새 기능 개발 시 "SaaS로 대체 가능한가?" 를 먼저 검토할 것
  Core가 아닌 기능에 시니어 개발자를 배치하지 말 것
```

---

## 📌 핵심 정리

```
서브도메인 분류 핵심:

Core Domain:
  정의: 경쟁 우위의 원천, 비즈니스 차별화의 핵심
  투자: 최고 개발자 + DDD + 충분한 시간
  특징: 외부 솔루션 불가, 자체 구현 필수

Supporting Domain:
  정의: Core를 지원하는 필수 기능, 차별화 아님
  투자: 적절한 수준의 설계, 중간 개발자
  특징: 외부 솔루션 일부 활용 가능

Generic Domain:
  정의: 모든 회사에서 비슷하게 필요한 일반 기능
  투자: 연동(Adapter) 개발만
  특징: SaaS/오픈소스로 완전 대체, 자체 구현 피함

분류 기준:
  "이 기능을 잘 해야 경쟁에서 이기는가?" → Core
  "없으면 Core가 동작 안 하는가?" → Supporting
  "외부 솔루션으로 충분한가?" → Generic

핵심 원칙:
  Core에 집중 → 자원의 불균등 배분이 전략
  Generic의 자체 구현 = 기회비용 낭비
  Core가 무엇인지 모르면 DDD 시작 불가
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 회사들의 Core Domain은 무엇인가? 같은 "결제 처리" 기능이 어느 회사에서는 Core이고 어느 회사에서는 Generic인가?

```
A. 토스(Toss) — 핀테크 스타트업
B. 쿠팡 — 이커머스
C. 카카오페이 — 간편결제
D. 배달의민족 — 음식 배달
```

<details>
<summary>해설 보기</summary>

**결제 처리의 분류:**

**A. 토스**: 결제 처리가 **Core Domain**입니다. 토스는 "더 쉽고 저렴한 결제/금융"이 비즈니스 핵심입니다. 결제 UX, 정산 속도, 수수료 구조가 경쟁 우위의 원천이므로 자체 개발이 필수입니다.

**B. 쿠팡**: 결제 처리는 **Generic Domain**입니다. 쿠팡의 Core는 "빠른 배송(로켓배송)", 물류 최적화, 가격 경쟁력입니다. 결제는 PG사 연동으로 충분하며, 실제로 쿠팡은 다양한 PG사를 연동하여 사용합니다.

**C. 카카오페이**: 결제 처리가 **Core Domain**입니다. 카카오페이의 존재 이유가 결제 서비스이므로, 결제 안정성, UX, 보안이 핵심 차별화입니다.

**D. 배달의민족**: 결제 처리는 **Generic Domain** 또는 경계에 위치합니다. 배민의 Core는 배달 최적화, 음식점 검색/추천입니다. 결제는 Toss Payments 등 PG사와 제휴합니다. 단, 배민페이라는 자체 결제 수단은 락인(Lock-in) 전략으로 Supporting에 가깝습니다.

**교훈**: 같은 기능이라도 회사의 비즈니스 전략에 따라 Core/Generic이 달라집니다.

</details>

---

**Q2.** 처음에는 Generic으로 분류해 Twilio SMS를 썼는데, 3년 후 월 SMS 발송량이 5,000만 건이 됐다. 이제 비용이 너무 크다. 이 시점에 자체 구현으로 전환해야 하는가?

<details>
<summary>해설 보기</summary>

**비용 임계점에 도달했다면 자체 구현을 고려할 수 있습니다.** 단, 이 경우에도 여전히 "SMS 발송이 Core Domain이 되었는가?"는 아닙니다.

**판단 기준:**

```
비용 계산:
  Twilio 비용: 5,000만 건 × $0.0075 = $375,000/월 = 연 $4.5M
  자체 구현 비용: 통신사 직접 계약 + 개발 유지보수
  
  만약 자체 구현 + 통신사 계약으로 60% 절감 → 연 $2.7M 절감
  개발 비용(1년): $500K
  손익분기점: 1년 이내
  → 자체 구현이 경제적으로 타당

자체 구현 시 주의사항:
  - 여전히 "Generic"처럼 다루어야 함 (최소 투자)
  - 발송 큐, 재시도, 통신사별 처리 등 SaaS가 제공하던 것 직접 구현 필요
  - 팀의 유지보수 부담 증가
  
대안:
  SMS 아닌 카카오 알림톡(더 저렴) 비중 확대
  AWS SNS Direct Connect로 단가 협상
  → 자체 구현 없이도 비용 절감 가능
```

**핵심**: 비용 이유로 직접 구현하더라도 최소한의 투자로 운영하고, Core Domain 개발에 더 집중하는 것이 올바른 방향입니다.

</details>

---

**Q3.** 팀에서 "우리 서비스의 Core Domain이 뭔지 모르겠다"는 의견이 나왔다. 어떻게 Core Domain을 발견하는가?

<details>
<summary>해설 보기</summary>

**Core Domain 발견 워크숍 방법:**

```
Step 1: "우리가 경쟁사보다 잘하는 것" 브레인스토밍 (30분)
  모든 팀원이 포스트잇에 작성
  → "빠른 배달", "정확한 위치 추적", "개인화 추천"...

Step 2: 고객이 우리를 선택하는 이유 분석 (30분)
  NPS 설문, 이탈 고객 인터뷰, 긍정 리뷰 분석
  → "다른 데보다 배달이 빠르다" → 배달 최적화가 Core 신호

Step 3: 경쟁사가 복사할 수 없는 것 (20분)
  "경쟁사가 우리 코드를 그대로 복사해도 따라올 수 없는 것은?"
  → 축적된 데이터, 독자적 알고리즘, 네트워크 효과

Step 4: 만약 이 기능이 없어지면 (10분)
  "이 기능을 Twilio/AWS/외부 SaaS로 바꿔도 경쟁력이 유지되는가?"
  → "유지된다" → Generic
  → "유지 안 된다" → Core 또는 Supporting
```

흔히 Core Domain은 가장 복잡하고, 도메인 전문가와 가장 많은 대화가 필요하며, 요구사항이 가장 자주 바뀌는 영역입니다. "이 코드를 이해하려면 비즈니스를 알아야 한다"면 Core의 신호입니다.

</details>

---

<div align="center">

**[⬅️ 이전: DDD와 레이어드 아키텍처 비교](../ddd-philosophy/05-ddd-vs-layered.md)** | **[홈으로 🏠](../README.md)** | **[다음: Bounded Context 완전 분해 ➡️](./02-bounded-context.md)**

</div>
