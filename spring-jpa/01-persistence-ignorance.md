# Persistence Ignorance 원칙 — 도메인이 JPA를 몰라야 하는 이유

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Entity` / `@Column` / `@OneToMany`가 도메인 클래스에 침투할 때 어떤 문제가 생기는가?
- 도메인 모델이 JPA를 알면 왜 테스트가 복잡해지는가?
- 실용적 타협(JPA 어노테이션 허용 범위)과 완전 분리의 트레이드오프는?
- JPA 없이 도메인 테스트를 구성하는 방법은?
- Persistence Ignorance를 어느 수준까지 추구해야 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대부분의 Spring JPA 프로젝트는 도메인 클래스와 JPA Entity를 동일시한다. `@Entity`, `@Id`, `@OneToMany`가 붙은 클래스가 곧 도메인 클래스다. 이것이 시작은 단순하지만, 도메인이 JPA의 제약을 따르게 된다. 불변이어야 할 Value Object에 기본 생성자가 필요하고, 도메인 불변식을 깨는 `setter`가 생기고, 지연 로딩이 도메인 메서드를 호출하다 터진다.

Persistence Ignorance(PI)는 "도메인 모델은 저장 방식을 몰라야 한다"는 원칙이다. DB가 MySQL에서 MongoDB로 바뀌어도 도메인 코드는 변하지 않아야 한다. 현실에서 완전한 PI는 비용이 크므로, 팀의 상황에 맞는 실용적 타협점을 찾는 것이 이 문서의 목표다.

---

## 😱 흔한 실수 (Before — JPA가 도메인 설계를 지배)

```java
// JPA가 도메인 설계를 강제하는 패턴
@Entity
@Table(name = "orders")
public class Order {

    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;

    // JPA는 기본 생성자를 요구 → 불변식 보장 불가
    protected Order() {}

    // setter가 필요해짐 (기본 생성자 후 JPA가 필드 설정)
    public void setStatus(String status) { this.status = status; }
    public void setCustomerId(Long customerId) { this.customerId = customerId; }

    // @OneToMany는 컬렉션을 지연 로딩
    @OneToMany(cascade = CascadeType.ALL, fetch = FetchType.LAZY)
    private List<OrderLine> lines;

    // 도메인 메서드에서 지연 로딩 초기화 — 트랜잭션 없으면 LazyInitializationException
    public Money calculateTotal() {
        return lines.stream()  // 여기서 lines를 초기화 시도 → 예외 가능
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO_KRW, Money::add);
    }

    // @Transient: JPA 필드가 아닌 것을 명시 — 도메인 오브젝트에 JPA 언어
    @Transient
    private List<DomainEvent> domainEvents = new ArrayList<>();
}
```

```
문제 목록:

1. 불변식 보호 불가
   protected Order() {} → 어디서나 빈 Order 생성 가능
   order.setStatus("ANYTHING") → 상태 전이 규칙 없이 직접 설정

2. 도메인 메서드가 트랜잭션에 의존
   order.calculateTotal() → 트랜잭션 없으면 LazyInitializationException
   도메인 로직이 영속성 컨텍스트에 의존하게 됨

3. JPA 언어가 도메인에 침투
   @Transient, @Column, @Entity → 도메인 클래스에 JPA import
   도메인 레이어가 JPA에 결합

4. 테스트 어려움
   new Order() → 기본 생성자만으로 유효하지 않은 Order 생성
   Order.place()라는 팩토리 패턴을 쓸 수 없음 (기본 생성자가 필요하므로)
```

---

## ✨ 올바른 접근 (After — 실용적 PI)

```java
// 실용적 타협: JPA 어노테이션은 허용하되 도메인 설계를 훼손하지 않음
@Entity
@Table(name = "orders")
public class Order {

    @EmbeddedId
    private OrderId id;  // Value Object를 ID로

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    @Embedded
    private CustomerId customerId;  // VO 유지

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private List<OrderLine> lines = new ArrayList<>();

    @Embedded
    private Money totalAmount;

    // JPA 기본 생성자: protected로 숨김 (도메인 코드에서 직접 사용 불가)
    protected Order() {}

    // 팩토리 메서드: 올바른 생성 강제 (불변식 보장)
    public static Order place(CustomerId customerId, List<OrderLine> lines,
                               Discount discount, Money shippingFee) {
        validateLines(lines);
        Order order = new Order();  // JPA 기본 생성자 내부적으로 사용
        order.id = new OrderId();
        order.customerId = customerId;
        order.status = OrderStatus.PENDING;
        order.lines = new ArrayList<>(lines);
        order.recalculateTotalAmount(discount, shippingFee);
        order.registerEvent(new OrderPlaced(order.id, customerId, ...));
        return order;
    }

    // 도메인 메서드: 불변식 포함
    public void cancel(String reason) {
        if (status != OrderStatus.PENDING && status != OrderStatus.PAID) {
            throw new OrderNotCancellableException(status);
        }
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelled(this.id, reason));
    }

    // 외부에는 불변 뷰만 제공
    public List<OrderLine> lines() {
        return Collections.unmodifiableList(lines);
    }

    // @Transient로 비영속 이벤트 컬렉션
    @Transient
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    protected void registerEvent(DomainEvent event) {
        domainEvents.add(event);
    }

    public List<DomainEvent> pullDomainEvents() {
        List<DomainEvent> events = new ArrayList<>(domainEvents);
        domainEvents.clear();
        return events;
    }
}
```

---

## 🔬 내부 동작 원리

### 1. PI 위반의 단계별 비용

```
위반 수준 1: @Entity, @Id만 있음 (경미)
  → 비용: 가장 작음
  → JPA 기본 제약만 적용
  → 실용적으로 허용 가능

위반 수준 2: @Column 네이밍 + Fetch 전략 (보통)
  → 비용: DB 스키마가 도메인에 노출
  → @Column(name = "cust_id", nullable = false)
  → DB 컬럼명이 도메인 코드에 보임
  → 허용 수준에 따라 결정

위반 수준 3: 기본 생성자 강제 + setter (심각)
  → 비용: 불변식 보호 불가
  → setter를 통한 상태 우회 가능
  → public 기본 생성자 → 팩토리 패턴 무력화

위반 수준 4: 지연 로딩에 의존하는 도메인 로직 (최악)
  → 비용: 도메인 로직이 영속성 컨텍스트에 결합
  → 트랜잭션 없이 도메인 메서드 호출 불가
  → 도메인 단위 테스트에 Spring 컨텍스트 필요

PI 목표: 위반 수준 1~2는 허용, 3~4는 방지
```

### 2. JPA와 공존하는 패턴

```
패턴 1: protected 기본 생성자
  JPA가 사용하는 기본 생성자를 protected로
  → 외부 도메인 코드에서 사용 불가 (같은 패키지 제외)
  → 팩토리 메서드가 유일한 외부 생성 경로

  protected Order() {}  // JPA 전용
  public static Order place(...) { ... }  // 도메인 생성 경로

패턴 2: Value Object를 @Embedded로
  Money, Address, OrderId를 VO로 유지
  → 불변, 자기 검증 유지
  → @Embeddable + protected 기본 생성자

패턴 3: 컬렉션 외부 수정 차단
  @OneToMany 필드는 private
  외부에는 unmodifiableList만 반환
  수정은 도메인 메서드를 통해서만

패턴 4: @Transient로 비영속 필드 표시
  DomainEvent 컬렉션은 @Transient
  영속 대상이 아님을 명시

패턴 5: Fetch 전략을 Aggregate 설계에 맞게
  Aggregate 내부: FetchType.EAGER 또는 JOIN FETCH
  → 항상 함께 로드되어야 하는 객체
  다른 Aggregate 참조는 없음 (ID만 참조)
```

### 3. 완전 분리(Full Separation)의 트레이드오프

```
완전 분리:
  Domain 클래스: 순수 Java (JPA import 없음)
  JPA Entity: 별도 클래스
  Mapper: Domain ↔ JPA Entity 변환

  Domain Order ↔ (Mapper) ↔ OrderJpaEntity
  
  장점:
    완전한 PI — 도메인이 JPA 모름
    도메인 테스트 완전히 독립
    JPA → MongoDB 전환 시 Domain 코드 무관
  
  단점:
    클래스 수 2배
    Mapper 코드 유지 비용
    양방향 변환 버그 가능성
    대규모 팀에서만 가치가 있음

실용적 타협:
  Domain 클래스에 JPA 어노테이션 허용 (@Entity, @Id, @Embedded)
  단, 불변식과 도메인 설계는 훼손하지 않음
  
  허용: @Entity, @Id, @Embedded, @Enumerated, @Transient
  제한: public setter, public 기본 생성자, 지연 로딩 의존 도메인 로직

팀 규모별 선택:
  소규모(5인 이하): 실용적 타협 (JPA 어노테이션 도메인에 허용)
  중간(5~20인): 실용적 타협 + 엄격한 컨벤션
  대규모(20인+): 완전 분리 고려 (비용 감당 가능)
```

### 4. 도메인 테스트에서 JPA 없애기

```java
// JPA 없는 순수 도메인 테스트
class OrderTest {

    @Test
    void place_withValidItems_createsOrder() {
        // JPA, DB, Spring 없음 — 순수 Java 객체
        CustomerId customerId = new CustomerId(1L);
        List<OrderLine> lines = List.of(
            new OrderLine(new ProductId(1L), 2, Money.ofKrw(10_000))
        );

        Order order = Order.place(customerId, lines, Discount.NONE, Money.ofKrw(2_500));

        assertThat(order.customerId()).isEqualTo(customerId);
        assertThat(order.status()).isEqualTo(OrderStatus.PENDING);
        assertThat(order.totalAmount()).isEqualTo(Money.ofKrw(22_500));
        assertThat(order.pullDomainEvents()).hasSize(1);
        assertThat(order.pullDomainEvents().get(0)).isInstanceOf(OrderPlaced.class);
    }

    @Test
    void cancel_afterPending_changesStatusToCancelled() {
        Order order = OrderBuilder.anOrder().placed();

        order.cancel("고객 요청");

        assertThat(order.status()).isEqualTo(OrderStatus.CANCELLED);
        assertThat(order.pullDomainEvents())
            .anyMatch(e -> e instanceof OrderCancelled);
    }

    @Test
    void cancel_afterShipped_throwsException() {
        Order shipped = OrderBuilder.anOrder().shipped();  // Test Builder 활용
        assertThatThrownBy(() -> shipped.cancel("이미 배송됨"))
            .isInstanceOf(OrderNotCancellableException.class);
    }
}
// → Spring Context 없음, DB 없음, @SpringBootTest 없음 → 매우 빠름
```

---

## 💻 실전 코드

### @Embedded Value Object — 불변 유지

```java
// 불변 Value Object + JPA 요구사항 공존
@Embeddable
public final class Money {

    @Column(name = "amount", precision = 19, scale = 2)
    private final BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(name = "currency", length = 3)
    private final Currency currency;

    // JPA 전용: protected + null 초기화 (Reflection으로 필드 주입됨)
    protected Money() {
        this.amount = null;
        this.currency = null;
    }

    // 도메인 생성자: 검증 포함
    public Money(BigDecimal amount, Currency currency) {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0) throw new NegativeAmountException(amount);
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    // 불변 연산
    public Money add(Money other) {
        if (!currency.equals(other.currency)) throw new CurrencyMismatchException();
        return new Money(amount.add(other.amount), currency);
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Money)) return false;
        Money m = (Money) o;
        return amount.compareTo(m.amount) == 0 && currency == m.currency;
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount.stripTrailingZeros(), currency);
    }
}
```

---

## 📊 설계 비교

```
JPA 직접 사용 vs 실용적 PI vs 완전 분리:

                JPA 직접 사용         실용적 PI           완전 분리
────────────┼───────────────────┼──────────────────┼──────────────────
PI 수준      │ 낮음 (JPA 결합)    │ 중간 (어노테이션 허용)│ 높음 (완전 분리)
────────────┼───────────────────┼──────────────────┼──────────────────
도메인 순수성 │ 낮음              │ 중간 (~허용 범위)  │ 높음
────────────┼───────────────────┼──────────────────┼──────────────────
테스트 독립성 │ 낮음 (Spring 필요) │ 중간~높음         │ 높음
────────────┼───────────────────┼──────────────────┼──────────────────
구현 복잡도  │ 낮음              │ 중간              │ 높음 (Mapper 추가)
────────────┼───────────────────┼──────────────────┼──────────────────
기술 교체   │ 어려움            │ 중간              │ 쉬움 (도메인 무관)
────────────┼───────────────────┼──────────────────┼──────────────────
팀 규모 적합 │ 소규모 빠른 개발   │ 대부분의 팀       │ 대규모, 복잡 도메인
```

---

## ⚖️ 트레이드오프

```
실용적 PI의 허용 범위:

허용 (PI를 크게 해치지 않음):
  @Entity, @Table: 클래스 레벨, 의미 충분히 명확
  @Id, @GeneratedValue: ID 개념은 JPA와 도메인 공유 가능
  @Embedded, @Embeddable: VO 개념과 자연스럽게 매핑
  @Enumerated: Enum은 도메인 개념
  @Transient: 비영속 필드 표시 (인프라 언어지만 부담 적음)
  protected 기본 생성자: 불변식을 크게 해치지 않으면서 JPA 요구 충족

제한 (도메인 설계 훼손):
  public setter: 불변식 보호 불가
  public 기본 생성자: 팩토리 패턴 무력화
  FetchType.EAGER 무분별 사용: 과도한 로딩
  지연 로딩에 의존하는 도메인 메서드
  @Column(nullable=false, unique=true) 남용: DB 제약이 도메인 제약보다 먼저
```

---

## 📌 핵심 정리

```
Persistence Ignorance 핵심:

원칙: 도메인 모델은 저장 방식을 몰라야 한다
현실: 완전한 PI는 비용이 크므로 실용적 타협

실용적 PI 허용 범위:
  @Entity, @Id, @Embedded, @Enumerated, @Transient ← 허용
  public setter, public 기본 생성자 ← 방지
  지연 로딩에 의존하는 도메인 메서드 ← 방지

도메인 설계 우선:
  불변식: 팩토리 메서드 + protected 기본 생성자
  Value Object: @Embeddable + protected 기본 생성자
  컬렉션: private + unmodifiableList 외부 제공
  도메인 이벤트: @Transient

테스트 독립성:
  도메인 단위 테스트: JPA 없이 순수 Java
  @SpringBootTest 없음, DB 없음 → 빠른 테스트
```

---

## 🤔 생각해볼 문제

**Q1.** `@Entity`가 붙은 클래스에 `@DomainService`처럼 비즈니스 로직을 함께 넣는 것은 PI를 위반하는가?

<details>
<summary>해설 보기</summary>

**`@Entity` 어노테이션 자체가 PI 위반은 아닙니다. 도메인 로직이 영속성 메커니즘에 의존할 때 PI 위반입니다.**

```java
@Entity
public class Order {
    // ✅ PI 위반 아님: 도메인 로직이 JPA에 의존하지 않음
    public void cancel(String reason) {
        validateCancellable();
        this.status = OrderStatus.CANCELLED;
        registerEvent(new OrderCancelled(this.id, reason));
    }

    // ❌ PI 위반: 도메인 로직이 지연 로딩에 의존
    public Money calculateTotal() {
        return this.lines.stream()  // lines가 LAZY → 트랜잭션 없으면 예외
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO_KRW, Money::add);
    }
}
```

판단 기준: "이 코드가 JPA 없이 (순수 Java로) 실행 가능한가?"
→ 가능하면 PI 위반 없음. 불가능하면 PI 위반입니다.

</details>

---

**Q2.** `@GeneratedValue(strategy = IDENTITY)`를 쓰면 저장 전에 ID를 알 수 없다. 이것이 Aggregate ID 설계에 어떤 영향을 미치는가?

<details>
<summary>해설 보기</summary>

**ID를 저장 전에 알 수 없으면 Domain Event에 ID를 포함하기 어렵습니다.**

```java
// 문제: IDENTITY 전략
@GeneratedValue(strategy = GenerationType.IDENTITY)
private Long id;

public static Order place(...) {
    Order order = new Order();
    // id는 null! → save()하기 전까지 ID 없음
    order.registerEvent(new OrderPlaced(order.id, ...));  // id가 null인 이벤트!
}
```

**해결 방법:**
```java
// 1. UUID 사용 (저장 전 ID 알 수 있음 — 권장)
@EmbeddedId  // 또는 @Id
private OrderId id = new OrderId();  // UUID.randomUUID()

// 2. SEQUENCE 전략 + nextval 미리 할당
@GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "order_seq")
// save() 전에 sequence.nextval을 가져와서 ID 설정 가능

// 3. ID 발급 후 이벤트 등록
Order order = orderRepository.save(new Order(...));  // ID 생성
order.registerEvent(new OrderPlaced(order.id(), ...));  // ID 있음
// → 저장 후 이벤트 등록: Outbox 패턴과 함께 사용
```

UUID를 도메인 ID로 사용하는 것이 가장 깔끔한 해법입니다. DB의 auto-increment는 읽기 성능에 유리하지만 분산 환경과 이벤트 설계에 제약이 있습니다.

</details>

---

**Q3.** Persistence Ignorance를 지키면서 도메인 객체에 JPA의 `@Version`으로 낙관적 잠금을 적용하는 것이 적절한가?

<details>
<summary>해설 보기</summary>

**기술적 타협이지만, 도메인 설계를 훼손하지 않으므로 허용 가능합니다.**

```java
@Entity
public class Order {
    @Version
    private Long version;  // JPA 낙관적 잠금 → 도메인에 JPA 언어

    // 이것이 PI를 얼마나 위반하는가?
    // → version 필드가 JPA 전용 목적
    // → 도메인 로직에서 version을 직접 사용하지 않음
    // → 허용 가능한 타협 수준
}
```

완전 분리 방식에서의 처리:
```java
// Domain 클래스 — version 없음
public class Order { ... }

// JPA Entity 클래스 — version 있음
@Entity
public class OrderJpaEntity {
    @Version
    private Long version;
    // ...
}
```

`@Version`은 도메인 개념이 아닌 순수 기술적 메커니즘입니다. 실용적 PI에서는 허용하되, 도메인 로직에서 `version` 값을 직접 읽거나 쓰지 않도록 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: ACL 실전 구현](../domain-events/06-acl-implementation.md)** | **[홈으로 🏠](../README.md)** | **[다음: JPA Entity와 DDD Entity 분리 ➡️](./02-jpa-vs-ddd-entity.md)**

</div>
