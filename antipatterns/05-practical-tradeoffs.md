# 실무에서의 타협 — 완벽한 DDD는 없다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- JPA 어노테이션을 도메인 클래스에 허용하는 범위와 그 근거는?
- Aggregate 경계를 이상적으로 유지하기 어려운 실무 제약은 무엇인가?
- 팀 역량과 도메인 복잡도에 따른 DDD 점진적 적용 전략은?
- "이론적으로 옳은 것"과 "팀이 실제로 할 수 있는 것" 사이의 균형은?
- DDD를 도입할 때 가장 먼저 포기해야 할 것과 끝까지 지켜야 할 것은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

이 레포지토리의 마지막 문서다. 지금까지 "올바른 DDD"를 다뤘다면, 이 문서는 "현실의 DDD"를 다룬다. 모든 팀이 처음부터 완벽한 Aggregate 설계, 완전한 Persistence Ignorance, Outbox Pattern을 적용할 수 없다. 중요한 것은 "어디서 타협할 것인가"를 의식적으로 결정하는 것이다.

의식적인 타협은 기술 부채가 아니다. 현재 상황에서 최선을 선택하는 것이다. 다만 "나중에 개선할 수 있도록" 설계해야 한다.

---

## 😱 흔한 실수 (Before — 두 가지 극단)

```
극단 1: "완벽한 DDD 아니면 안 한다"

  결과:
    DDD 도입 프로젝트가 6개월 넘도록 계속
    팀원들이 이론만 배우고 적용 못함
    비즈니스 요구사항은 계속 쌓임
    결국 빅뱅으로 전환 시도 → 실패 → 포기

극단 2: "DDD는 너무 복잡하다, 우리는 안 한다"

  결과:
    Transaction Script로 모든 것을 구현
    6개월 후 Service가 수천 줄
    비즈니스 규칙이 어디 있는지 아무도 모름
    버그 수정에 수일 소요
    신규 기능 추가가 점점 느려짐

올바른 접근:
  현재 팀이 소화할 수 있는 수준에서 시작
  핵심 개념부터 하나씩 적용
  의식적으로 타협 지점을 결정하고 문서화
```

---

## ✨ 올바른 접근 (After — 의식적 타협 전략)

```
타협의 3원칙:

1. "지금 적용할 수 없어도 나중에 개선할 수 있는 구조로"
   → 타협 내용을 문서화 (TODO, ADR)
   → 경계를 명확히 (나중에 리팩터링 가능한 지점)

2. "팀이 이해하는 것만 적용한다"
   → 이해 못 한 패턴을 억지로 적용 → 더 나쁜 결과
   → 작은 것부터 팀이 이해하면서 적용

3. "핵심 불변식만큼은 포기하지 않는다"
   → JPA @Entity 허용해도 setter 막기
   → Persistence Ignorance 포기해도 불변식 검증 유지
```

---

## 🔬 내부 동작 원리

### 1. JPA 어노테이션 허용 범위와 근거

```
실무 타협 가이드라인:

✅ 허용 (PI를 크게 해치지 않음):
  @Entity, @Table           → 매핑 선언, 불가피
  @Id, @EmbeddedId          → ID 개념은 도메인과 공유
  @Embedded, @Embeddable    → VO 개념과 자연스럽게 매핑
  @Enumerated               → Enum은 도메인 개념
  @Transient                → 비영속 필드 표시
  @Version                  → 낙관적 잠금 (기술적 필요)
  @Column(nullable=false)   → DB 제약 명시 (문서 역할도)
  protected 기본 생성자      → JPA 필요, 불변식 우회 없음
  @CreatedDate, @LastModified → 감사 정보 (기술적 필요)

❌ 제한 (도메인 설계 훼손):
  public setter             → 불변식 보호 불가, 대안 없음
  public 기본 생성자         → 팩토리 패턴 무력화
  지연 로딩 의존 도메인 로직  → 트랜잭션 결합
  @Query JPQL in Domain     → 인프라 언어 침투
  @Transactional in Domain  → 트랜잭션 관리는 Application 레이어

회색 지대 (상황에 따라):
  @Column(name="legacy_col") → DB 스키마 침투. 레거시 연동 시 허용
  @JsonIgnore               → 직렬화 관심사. 별도 DTO 권장하지만 허용 가능
  @OneToMany fetch=LAZY      → 성능 설정. 도메인 설계보단 인프라 관심사

타협 문서화:
  // TODO(DDD): 완전 분리 시 JpaEntity 분리 예정
  // TRADE-OFF: 현재 @Column 직접 사용 (레거시 스키마 호환)
  // ADR-007: Domain 클래스에 @Embedded 허용 결정
```

### 2. Aggregate 경계 실무 제약

```
이상적 Aggregate 경계가 어려운 경우:

제약 1: 성능 vs 불변식
  "Order와 OrderLine이 같은 Aggregate이면 항상 함께 로드"
  "하지만 주문 목록에서는 항목이 필요 없음"
  
  타협: LAZY 로딩 + 필요 시 JOIN FETCH
  @OneToMany(fetch = LAZY)
  // 목록 조회: OrderLine 로딩 안 함 (lazy)
  // 상세 조회: JOIN FETCH로 함께 로딩
  
  포기하지 말 것: OrderLine은 여전히 Order 내부 Entity
                  직접 OrderLine 수정 불가, Order를 통해서만

제약 2: 레거시 스키마
  "이상적으론 Money VO로 분리하고 싶지만
   기존 테이블이 amount, currency_code로 별도 컬럼"
  
  타협: @AttributeOverride로 기존 컬럼명 매핑
  @Embedded
  @AttributeOverrides({
    @AttributeOverride(name="amount", column=@Column(name="total_amount")),
    @AttributeOverride(name="currency", column=@Column(name="currency_code"))
  })
  private Money totalAmount;
  
  포기하지 말 것: Money VO는 유지 (검증, 불변, 연산)
                  DB 컬럼명만 레거시에 맞춤

제약 3: 외부 시스템 동기 강요
  "이상적으론 이벤트로 비동기 처리하고 싶지만
   고객이 즉시 재고 차감 확인을 원함"
  
  타협: 재고 확인은 동기, 차감은 이벤트
  // 주문 전: 재고 있는지 동기 확인 (고객 즉각 피드백)
  inventoryPort.isAvailable(productId, quantity)
  // 주문 후: 차감은 이벤트 (Order 응답에 영향 없음)
  publishEvent(new OrderPlaced(...))

제약 4: 팀 역량
  "Outbox Pattern이 이상적이지만 팀이 Kafka 운영 경험 없음"
  
  타협 1: @TransactionalEventListener(AFTER_COMMIT)으로 시작
           → 서버 재시작 시 이벤트 유실 가능 (모니터링 보완)
  타협 2: 데이터 중요도별 분리
           포인트, 이메일: @TransactionalEventListener (유실 허용)
           재고, 결제: Outbox Pattern (유실 불허)
```

### 3. 팀 역량별 DDD 적용 단계

```
팀 역량 Level 1 (DDD 입문):

  적용할 것:
    Entity에 기본 불변식 (유효성 검증, setter 제한)
    Service 분리 (Application Service + Domain Service)
    기본적인 Repository 패턴

  타협:
    JPA Entity = Domain Entity (완전 분리 X)
    Value Object 일부만 (Money 정도)
    이벤트 없이 직접 호출

  목표: "Entity가 자신의 불변식을 보호한다"

─────────────────────────────────────────────

팀 역량 Level 2 (DDD 기본):

  추가 적용:
    Value Object 체계적 추출 (Address, Email, OrderId)
    Aggregate 경계 명시 (OrderLine은 Order 내부)
    Domain Event 도입 (내부 이벤트부터)
    Repository 인터페이스와 구현 분리

  타협:
    Outbox Pattern 없이 @TransactionalEventListener
    완전한 PI 포기 (JPA 어노테이션 Domain에 허용)

  목표: "Aggregate가 일관성 경계를 보장한다"

─────────────────────────────────────────────

팀 역량 Level 3 (DDD 숙련):

  추가 적용:
    Outbox Pattern + Kafka (신뢰성 있는 이벤트)
    Saga Pattern (분산 트랜잭션 조율)
    CQRS (읽기/쓰기 모델 분리)
    완전한 PI (Domain Entity와 JPA Entity 분리)

  타협:
    모든 Context에 DDD 적용 X (CRUD는 간단하게)
    완벽한 Ubiquitous Language 대신 "충분히 명확한" 언어

  목표: "시스템 전체가 비즈니스 언어로 표현된다"
```

### 4. 끝까지 지켜야 할 것 vs 포기할 수 있는 것

```
절대 포기하지 말 것 (Core of DDD):

  1. Entity의 불변식 보호
     public setter 금지
     도메인 메서드를 통한 상태 변경만 허용
     → 이것이 없으면 DDD가 아님

  2. Ubiquitous Language
     코드가 비즈니스 언어를 사용
     order.cancel() vs order.setStatus("CANCELLED")
     → 이것이 없으면 DDD가 아님

  3. Aggregate 경계의 존재
     어디서 어디까지가 하나의 일관성 단위인지 명확히
     경계가 아예 없으면 의미 없음

  4. 비즈니스 규칙의 도메인 레이어 집중
     Service에 비즈니스 로직이 흩어지지 않도록
     → 이것이 없으면 Anemic Model

─────────────────────────────────────────────

타협 가능한 것 (Tactical Details):

  1. Persistence Ignorance 수준
     완전 분리 → JPA 어노테이션 허용으로 타협 가능
     (불변식 보호를 유지하면서)

  2. 이벤트 신뢰성 수준
     Outbox → @TransactionalEventListener로 타협 가능
     (중요도 낮은 이벤트에서)

  3. CQRS 적용 범위
     전체 적용 → 핵심 조회 화면만 읽기 모델 적용

  4. Context 서비스 분리 수준
     마이크로서비스 → 모놀리스 내 패키지 분리로 타협

  5. Value Object 완전성
     모든 원시 타입 래핑 → 핵심 VO만 (Money, OrderId 등)
```

---

## 💻 실전 코드

### 타협 포인트 문서화 패턴

```java
/**
 * 주문 Aggregate.
 *
 * <h3>설계 결정 (ADR-012):</h3>
 * <p>JPA {@code @Entity}를 Domain 클래스에 직접 적용.</p>
 * <p>이상적으로는 JPA Entity와 Domain Object를 완전 분리해야 하지만,
 * 현재 팀 규모(4명)와 초기 단계를 고려해 단일 클래스를 사용합니다.</p>
 *
 * <h3>허용된 타협:</h3>
 * <ul>
 *   <li>{@code @Entity}, {@code @EmbeddedId} 등 JPA 어노테이션 허용</li>
 *   <li>{@code protected} 기본 생성자 (JPA 필요)</li>
 * </ul>
 *
 * <h3>포기하지 않은 것:</h3>
 * <ul>
 *   <li>public setter 없음 → 모든 변경은 도메인 메서드로</li>
 *   <li>불변식 검증 → 각 메서드 내부에서 보장</li>
 *   <li>Ubiquitous Language → cancel(), confirmPayment() 등 비즈니스 언어</li>
 * </ul>
 *
 * @see <a href="docs/adr/ADR-012-domain-jpa-colocation.md">ADR-012</a>
 */
@Entity
@Table(name = "orders")
public class Order extends AggregateRoot {
    // ...
}
```

### ADR (Architecture Decision Record) 템플릿

```
ADR-012: Domain 클래스에 JPA 어노테이션 허용

상태: 승인됨 (2024-03-15)

맥락:
  완전한 Persistence Ignorance를 위해 Domain Entity와 JPA Entity를 분리하는 것이
  이상적이지만, 팀 규모(4명)와 개발 초기 단계를 고려합니다.

결정:
  Domain 클래스에 JPA 어노테이션을 허용하되, 다음 제약을 지킵니다.
  - public setter 금지
  - 불변식 검증은 도메인 메서드에서 보장
  - protected 기본 생성자만 허용

허용 어노테이션:
  @Entity, @Table, @Id, @EmbeddedId, @Embedded, @Enumerated, @Transient, @Version,
  @OneToMany(cascade=ALL, orphanRemoval=true, 내부 Entity만), @Column(nullable=false)

금지 사항:
  public setter, public 기본 생성자, @ManyToOne to other Aggregates (ID 참조로)

결과:
  긍정: 개발 속도 향상, 팀 학습 부담 감소
  부정: Domain이 JPA에 의존, 기술 교체 시 Domain 수정 필요
  완화: 단위 테스트로 Domain 로직은 여전히 독립적으로 검증 가능

재검토 시점:
  팀이 10명 이상이 되거나 JPA 교체 요구가 발생할 때
```

---

## 📊 설계 비교

```
완벽한 DDD vs 실용적 DDD:

                완벽한 DDD              실용적 DDD (타협)
────────────┼──────────────────────┼──────────────────────────
PI 수준      │ 완전 분리              │ JPA 어노테이션 허용
────────────┼──────────────────────┼──────────────────────────
이벤트 신뢰성│ Outbox Pattern        │ @TransactionalEventListener
────────────┼──────────────────────┼──────────────────────────
불변식 보호  │ 완전 보장              │ 완전 보장 (타협 불가)
────────────┼──────────────────────┼──────────────────────────
유비쿼터스   │ 완전 일치              │ 충분히 명확
Language     │                      │
────────────┼──────────────────────┼──────────────────────────
팀 학습 비용 │ 매우 높음             │ 중간
────────────┼──────────────────────┼──────────────────────────
개발 속도    │ 초기 느림             │ 중간
────────────┼──────────────────────┼──────────────────────────
장기 유지보수│ 매우 용이             │ 용이
```

---

## ⚖️ 트레이드오프

```
타협의 위험:
  타협이 습관이 되면 "어차피 나중에" 하다가 영영 못 함
  → 타협 내용을 반드시 문서화 (ADR, TODO)
  → 분기별 기술 부채 검토 세션 운영

타협의 올바른 사용:
  현재 상황에서 최선을 선택
  나중에 개선 가능한 구조로 설계
  타협 이유를 팀이 공유

DDD 여정의 현실:
  모든 팀이 Level 1에서 시작
  핵심 개념 하나씩 적용하며 성장
  6개월~1년이면 Level 2 도달 가능
  완벽한 DDD를 추구하다 아무것도 못 하는 것보다
  불완전하지만 실용적인 DDD가 낫다
```

---

## 📌 핵심 정리

```
실무 DDD 타협 핵심:

절대 타협 불가:
  Entity 불변식 보호 (setter 금지)
  Ubiquitous Language (비즈니스 언어 코드화)
  Aggregate 경계 (어느 수준에서든 명시)
  비즈니스 로직의 도메인 레이어 집중

타협 가능:
  Persistence Ignorance 수준
  이벤트 신뢰성 수준 (Outbox 없이 시작)
  CQRS 적용 범위
  Context 분리 수준 (모놀리스 패키지부터)
  Value Object 완전성

팀 역량별 단계:
  Level 1: Entity 불변식 + 기본 Repository
  Level 2: VO 추출 + Aggregate 경계 + 기본 Event
  Level 3: Outbox + Saga + CQRS + 완전 PI

의식적 타협:
  타협 이유를 ADR, TODO, 주석으로 문서화
  재검토 시점을 명시
  팀 전체가 타협 내용을 공유

결론:
  완벽한 DDD는 없다
  오늘 할 수 있는 최선을 다하고
  내일 더 나아지는 것이 실용적 DDD
```

---

## 🤔 생각해볼 문제

**Q1.** "팀이 DDD 책을 다 읽었는데 실제 코드에 적용하면 항상 어색하다"는 문제의 원인과 해결책은?

<details>
<summary>해설 보기</summary>

**이론과 실전 사이의 간격은 정상입니다. 작은 것 하나부터 실제 코드에 적용해야 합니다.**

```
원인 분석:

1. "전부 적용하거나 아예 안 한다" 사고
   → 실제 코드베이스는 이미 레거시가 있음
   → 전부 적용이 불가능해 보여서 시작 못함

2. 도메인 지식 부족
   → "Ubiquitous Language"는 도메인 전문가와 만들어야
   → 책만 읽어서는 도메인 언어를 모름

3. 실습 없이 이론만
   → DDD는 손으로 코딩해봐야 체득

해결책:

Step 1: 기존 코드에서 가장 큰 불만 찾기
  "이 Service가 너무 비대하다"
  → 하나의 메서드에서 Value Object 하나만 추출

Step 2: 실제 코드에 하나씩 적용
  Money VO 추출 → 검증, 연산 테스트 → "아 이게 VO구나"
  Cancel 메서드 → 불변식 검증 포함 → "이게 Rich Domain Model이구나"

Step 3: 도메인 전문가와 이야기
  "취소 가능한 조건이 뭐예요?" → 직접 확인
  코드에 그 언어를 반영 → Ubiquitous Language 체감

Step 4: 팀원과 코드 리뷰
  "이 로직이 Entity에 있어야 하는가 Service에 있어야 하는가?"
  → 논의 자체가 DDD 학습

현실: DDD는 6개월~1년의 실전 적용이 있어야 자연스러워짐
```

</details>

---

**Q2.** 새로 합류한 팀원이 기존 DDD 코드를 이해 못해 Anemic 방식으로 코드를 추가하고 있다. 어떻게 대응하는가?

<details>
<summary>해설 보기</summary>

**개인 탓이 아닌 팀 프로세스 문제로 보고 구조적으로 해결합니다.**

```
즉각 대응:
  코드 리뷰에서 "이 로직은 Order.cancel()에 있어야 합니다"
  → 구체적인 예시와 이유 제시
  → 추상적인 "DDD 방식으로 해주세요" X

구조적 해결:

1. 온보딩 문서 작성:
   "우리 팀의 DDD 적용 가이드"
   - 허용된 타협 목록 (ADR)
   - Entity에 로직 넣는 예시 코드
   - "이렇게 하지 마세요" 예시

2. ArchUnit 규칙 강제:
   @ArchTest
   static ArchRule noPublicSetters = noMethods()
       .that().haveNameStartingWith("set")
       .should().bePublic()
       .andShould().beDeclaredInClassesThat().resideInPackage("..domain..");
   → 실수를 자동으로 잡아줌

3. 페어 프로그래밍:
   새 팀원이 기능 구현 시 경험자와 함께
   DDD 방식이 자연스럽게 전파됨

4. 팀 DDD 세션 (월 1회):
   실제 코드 리뷰 중 발견된 패턴 공유
   좋은 예시와 나쁜 예시 함께 검토
```

</details>

---

**Q3.** 이 레포지토리(ddd-deep-dive)를 다 읽은 후 실무에서 DDD를 적용하려 한다. 첫 번째로 해야 할 일은 무엇인가?

<details>
<summary>해설 보기</summary>

**지금 가장 고통스러운 코드 하나를 찾아 Value Object를 추출하는 것부터 시작합니다.**

```
행동 계획 (첫 주):

Day 1: 가장 고통스러운 코드 찾기
  "이 코드를 수정할 때마다 버그가 생긴다"
  "이 로직이 여기 있는 게 맞나?" 하는 의구심이 드는 곳
  → 그곳이 DDD 적용의 시작점

Day 2: 도메인 전문가와 이야기
  "이 비즈니스 규칙을 정확히 설명해주세요"
  → 그 언어를 코드에 반영할 준비

Day 3: 가장 작은 Value Object 추출
  BigDecimal amount → Money(amount, currency) VO
  단위 테스트 추가: Money.add(), Money.subtract()
  팀원에게 PR 리뷰 요청: "이 방향 어떻게 생각해요?"

Day 4-5: 팀과 공유
  "이렇게 바꿨더니 이런 버그가 방지됐어요"
  구체적인 이점을 코드로 보여줌

첫 달 목표:
  Money, Address, OrderId 등 기본 VO 추출 완료
  하나의 Entity에 불변식 메서드 추가
  팀이 "이 방향이 좋다"고 공감

6개월 목표:
  핵심 Aggregate 설계 완료
  기본 Repository 패턴 적용
  팀 전체가 DDD 방식으로 코딩

1년 목표:
  Domain Event 도입
  읽기 모델 일부 분리
  팀이 DDD를 "자연스럽게" 사용

기억할 것:
  DDD는 목적이 아닌 수단
  목적은 "비즈니스 문제를 잘 해결하는 소프트웨어"
  DDD가 그 목적을 달성하는 데 도움이 될 때 사용
```

</details>

---

<div align="center">

**[⬅️ 이전: DDD 과잉 적용](./04-ddd-overengineering.md)** | **[홈으로 🏠](../README.md)**

</div>
