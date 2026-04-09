# Value Object JPA 매핑 — `@Embedded`와 `AttributeConverter`

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `@Embeddable`로 Value Object를 같은 테이블에 매핑하는 방법과 한계는?
- 불변 Value Object를 JPA와 함께 사용할 때 기본 생성자 문제를 어떻게 해결하는가?
- `AttributeConverter`로 단순 Value Object를 단일 컬럼에 매핑하는 방법은?
- 두 방법(@Embeddable vs AttributeConverter)의 선택 기준은 무엇인가?
- `@AttributeOverride`로 컬럼명 충돌을 어떻게 해결하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`Money`, `Email`, `Address`, `OrderId` 같은 Value Object를 JPA로 어떻게 저장하는가? 가장 단순한 방법은 원시 타입(String, BigDecimal)으로 분해해서 각 컬럼에 저장하는 것이지만, 그러면 도메인 의미를 잃는다. JPA는 `@Embeddable`과 `AttributeConverter` 두 가지 메커니즘을 제공한다. 각각 언제, 어떻게 써야 하는지 알아야 도메인 모델의 풍부함을 DB 매핑에서도 유지할 수 있다.

---

## 😱 흔한 실수 (Before — VO를 원시 타입으로 분해)

```java
// 안티패턴: Value Object 없이 원시 타입 나열
@Entity
@Table(name = "orders")
public class Order {

    @Id
    private Long id;

    // Money를 분해해서 저장 — 도메인 의미 상실
    private BigDecimal amount;    // 어느 통화? 음수 가능?
    private String currency;      // "KRW"? "USD"? 검증 없음

    // Address를 분해해서 저장
    private String shippingCity;
    private String shippingStreet;
    private String shippingZipCode;  // 5자리 검증 없음

    // Email을 그냥 String으로
    private String customerEmail;  // 형식 검증 없음, 어디서든 임의로 변경 가능

    // 검증 로직이 Service에 흩어짐
    public void setAmount(BigDecimal amount) {
        if (amount == null || amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("금액은 양수여야 합니다");
        }
        this.amount = amount;  // 하지만 setter가 public이면 우회 가능
    }
}
```

```
결과:
  Order 테이블 컬럼: id, amount, currency, shipping_city, shipping_street,
                    shipping_zip_code, customer_email, ...
  컬럼이 많아질수록 어떤 컬럼이 어떤 개념인지 파악 어려움
  
  order.setAmount(new BigDecimal("-100"));  // setter 우회 → 음수 저장됨
  order.setCurrency("INVALID");            // 검증 없음
```

---

## ✨ 올바른 접근 (After — @Embeddable과 AttributeConverter 활용)

```java
// 방법 1: @Embeddable — 복합 VO (여러 컬럼)
@Embeddable
public final class Money {

    @Column(name = "amount", precision = 19, scale = 2)
    private BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(name = "currency", length = 3)
    private Currency currency;

    protected Money() {}  // JPA 전용

    public Money(BigDecimal amount, Currency currency) {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0) throw new NegativeAmountException(amount);
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    // 불변 연산
    public Money add(Money other) { ... }
    public Money multiply(double rate) { ... }

    @Override public boolean equals(Object o) { ... }
    @Override public int hashCode() { ... }
}

// 방법 2: AttributeConverter — 단순 VO (단일 컬럼)
@Converter(autoApply = true)  // autoApply: 모든 Email 타입에 자동 적용
public class EmailConverter implements AttributeConverter<Email, String> {

    @Override
    public String convertToDatabaseColumn(Email email) {
        return email == null ? null : email.value();
    }

    @Override
    public Email convertToEntityAttribute(String dbValue) {
        return dbValue == null ? null : new Email(dbValue);
    }
}

// 사용
@Entity
@Table(name = "orders")
public class Order {

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "total_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "total_currency"))
    })
    private Money totalAmount;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "shipping_fee_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "shipping_fee_currency"))
    })
    private Money shippingFee;  // Money를 두 번 사용 → @AttributeOverride 필수

    @Column(name = "customer_email")  // EmailConverter가 autoApply라면 @Convert 불필요
    private Email customerEmail;     // DB: VARCHAR, 도메인: Email VO
}
```

---

## 🔬 내부 동작 원리

### 1. @Embeddable 동작 방식

```
@Embeddable의 핵심:
  "이 클래스의 필드를 소유자(Owner) 테이블의 컬럼으로 직접 포함"

DB 스키마 결과:
  orders 테이블:
    id             BIGINT
    total_amount   DECIMAL(19,2)    ← Money.amount
    total_currency VARCHAR(3)       ← Money.currency
    shipping_fee_amount   DECIMAL(19,2)   ← ShippingFee.amount
    shipping_fee_currency VARCHAR(3)      ← ShippingFee.currency

  Money 테이블은 존재하지 않음
  orders 테이블에 컬럼으로 직접 포함됨

@AttributeOverride 필요한 이유:
  Order가 Money를 두 곳에서 사용:
    totalAmount → columns: amount, currency
    shippingFee → columns: amount, currency  ← 충돌!
  
  기본 컬럼명이 겹치므로 @AttributeOverride로 각각 구분:
    totalAmount.amount   → total_amount
    shippingFee.amount   → shipping_fee_amount

JPA가 @Embeddable을 로드하는 방식:
  1. Order 조회 → orders 테이블에서 단일 행 로드
  2. amount, currency 컬럼 → Money 객체 생성 (기본 생성자 사용)
  3. Reflection으로 필드 직접 설정
  4. Order.totalAmount에 Money 객체 주입

  → Money의 기본 생성자가 protected여도 Reflection이 접근 가능
```

### 2. AttributeConverter 동작 방식

```
AttributeConverter의 핵심:
  "VO ↔ DB 단일 컬럼 값 변환"

적합한 경우:
  하나의 컬럼으로 표현되는 VO
  Email → VARCHAR (email 주소 문자열)
  OrderId → VARCHAR (UUID 문자열)
  Percentage → DECIMAL (숫자)
  Color → VARCHAR (#FF0000)

동작 흐름:
  저장 시: Email.value() → "user@example.com" → VARCHAR 컬럼
  조회 시: "user@example.com" → new Email("user@example.com") → Email VO

autoApply = true:
  @Converter(autoApply = true)으로 선언하면
  엔티티에 @Convert 어노테이션 불필요
  해당 타입을 가진 모든 @Entity 필드에 자동 적용

autoApply = false (기본):
  @Column(name = "customer_email")
  @Convert(converter = EmailConverter.class)
  private Email customerEmail;  // 명시적 선언 필요
```

### 3. @Embeddable vs AttributeConverter 선택 기준

```
@Embeddable이 적합한 경우:

  ✅ VO가 여러 필드를 가짐 (복합 값)
     Money: amount + currency → 두 컬럼
     Address: city + street + zipCode → 세 컬럼
     DateRange: startDate + endDate → 두 컬럼

  ✅ 필드 각각을 DB 쿼리 조건으로 사용해야 함
     WHERE amount > 10000 → amount가 별도 컬럼이어야 쿼리 가능
     WHERE city = '서울' → city가 별도 컬럼이어야 쿼리 가능

  ✅ 컬럼 인덱스가 필요한 경우
     @Column(name = "currency") → 인덱스 생성 가능

AttributeConverter가 적합한 경우:

  ✅ VO가 단일 값 (단일 컬럼으로 표현)
     Email → VARCHAR
     OrderId → VARCHAR (UUID)
     ZipCode → VARCHAR(5)
     Percentage → DECIMAL

  ✅ 기존 DB 스키마에 단일 컬럼으로 이미 저장됨
     email VARCHAR → Email VO로 변환

  ✅ 범용 타입 (여러 Entity에서 동일하게 사용)
     Email은 Order, Customer, Review 등 여러 곳에서 사용
     → autoApply = true로 한 번만 등록

결정 트리:
  "이 VO를 몇 개의 컬럼으로 저장하는가?"
  → 1개: AttributeConverter
  → 2개 이상: @Embeddable
  
  "이 VO의 내부 필드로 DB 쿼리가 필요한가?"
  → YES: @Embeddable (필드가 컬럼이 됨)
  → NO:  AttributeConverter도 충분
```

### 4. null 처리와 기본값

```java
// @Embeddable null 처리
@Entity
public class Order {

    @Embedded
    private ShippingAddress shippingAddress;  // null 가능

    // JPA는 @Embeddable의 모든 필드가 null이면 객체 자체를 null로 설정
    // → shippingAddress.city = null, shippingAddress.street = null
    // → JPA가 ShippingAddress 객체 자체를 null로 로드할 수 있음
}

// null 방지: @AttributeOverride + nullable = false
@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "city",
        column = @Column(name = "shipping_city", nullable = false)),
    @AttributeOverride(name = "street",
        column = @Column(name = "shipping_street", nullable = false))
})
private ShippingAddress shippingAddress;

// Null Object 패턴으로 null 방지
@Embeddable
public final class Money {
    public static final Money ZERO = new Money(BigDecimal.ZERO, Currency.KRW);
    // order.totalAmount가 null이 아닌 ZERO로 초기화
}
```

---

## 💻 실전 코드

### 복잡한 VO 매핑 — Address

```java
@Embeddable
public final class Address {

    @Column(name = "city", length = 50)
    private String city;

    @Column(name = "street", length = 200)
    private String street;

    @Embedded  // @Embeddable 중첩 가능
    private ZipCode zipCode;  // ZipCode도 @Embeddable

    protected Address() {}

    public Address(String city, String street, ZipCode zipCode) {
        Objects.requireNonNull(city, "도시는 null일 수 없습니다");
        Objects.requireNonNull(street, "도로명은 null일 수 없습니다");
        Objects.requireNonNull(zipCode, "우편번호는 null일 수 없습니다");
        this.city = city;
        this.street = street;
        this.zipCode = zipCode;
    }

    public boolean isIsland() {
        return city.contains("제주");
    }

    @Override
    public boolean equals(Object o) {
        if (!(o instanceof Address)) return false;
        Address a = (Address) o;
        return city.equals(a.city) && street.equals(a.street) && zipCode.equals(a.zipCode);
    }

    @Override
    public int hashCode() {
        return Objects.hash(city, street, zipCode);
    }
}

@Embeddable
public final class ZipCode {

    @Column(name = "zip_code", length = 5)
    private String value;

    protected ZipCode() {}

    public ZipCode(String value) {
        if (value == null || !value.matches("\\d{5}")) {
            throw new InvalidZipCodeException("우편번호는 5자리 숫자여야 합니다: " + value);
        }
        this.value = value;
    }
}
```

### AttributeConverter — OrderId, Email, Percentage

```java
// UUID 기반 ID를 VARCHAR로 변환
@Converter(autoApply = true)
public class OrderIdConverter implements AttributeConverter<OrderId, String> {

    @Override
    public String convertToDatabaseColumn(OrderId orderId) {
        return orderId == null ? null : orderId.value().toString();
    }

    @Override
    public OrderId convertToEntityAttribute(String dbValue) {
        return dbValue == null ? null : new OrderId(UUID.fromString(dbValue));
    }
}

// Percentage를 DECIMAL로 변환
@Converter(autoApply = true)
public class PercentageConverter implements AttributeConverter<Percentage, BigDecimal> {

    @Override
    public BigDecimal convertToDatabaseColumn(Percentage percentage) {
        return percentage == null ? null : BigDecimal.valueOf(percentage.value());
    }

    @Override
    public Percentage convertToEntityAttribute(BigDecimal dbValue) {
        return dbValue == null ? null : new Percentage(dbValue.doubleValue());
    }
}

// 사용: 자동 적용으로 @Convert 불필요
@Entity
public class Order {
    @EmbeddedId
    private OrderId id;  // OrderIdConverter 자동 적용

    @Column(name = "customer_email")
    private Email customerEmail;  // EmailConverter 자동 적용

    @Column(name = "discount_rate")
    private Percentage discountRate;  // PercentageConverter 자동 적용
}
```

---

## 📊 설계 비교

```
@Embeddable vs AttributeConverter:

                @Embeddable                  AttributeConverter
────────────┼───────────────────────────┼──────────────────────────
적합한 VO    │ 복합 값 (여러 필드)          │ 단순 값 (단일 컬럼)
────────────┼───────────────────────────┼──────────────────────────
컬럼 수      │ VO 필드 수만큼              │ 항상 1개
────────────┼───────────────────────────┼──────────────────────────
DB 쿼리      │ 각 필드로 WHERE 가능        │ 변환된 값으로만
────────────┼───────────────────────────┼──────────────────────────
인덱스       │ 각 컬럼에 인덱스 생성 가능  │ 변환 컬럼에 인덱스
────────────┼───────────────────────────┼──────────────────────────
중첩         │ @Embeddable 안에 @Embeddable│ 불가 (단일 컬럼)
────────────┼───────────────────────────┼──────────────────────────
null 처리   │ 모든 필드 null → 객체 null  │ null 명시적 처리
────────────┼───────────────────────────┼──────────────────────────
예시        │ Money, Address, DateRange   │ Email, OrderId, ZipCode
────────────┼───────────────────────────┼──────────────────────────
기본 생성자  │ protected 필요              │ 불필요 (변환만)
```

---

## ⚖️ 트레이드오프

```
@Embeddable의 한계:
  같은 Entity에서 동일 @Embeddable 타입을 두 번 사용하면
  @AttributeOverride로 컬럼명을 모두 명시해야 함
  → 코드가 장황해짐

  @Embeddable 안에 @Embeddable을 중첩할 수 있지만
  깊어질수록 @AttributeOverride 관리가 복잡해짐

AttributeConverter의 한계:
  변환된 단일 컬럼 값으로만 쿼리 가능
  "10,000원 이상인 주문 조회" → amount 컬럼이 별도로 없으면 불가
  → 중요한 조회 조건이 있으면 @Embeddable이 낫다

실용적 결정:
  Money: @Embeddable (amount로 범위 쿼리 필요)
  Email: AttributeConverter (email 문자열 전체로만 조회)
  OrderId: AttributeConverter (ID는 항상 전체 값으로 조회)
  Address: @Embeddable (city로 필터링 필요할 수 있음)
```

---

## 📌 핵심 정리

```
Value Object JPA 매핑 핵심:

@Embeddable:
  복합 VO (여러 필드) → 소유자 테이블 컬럼으로 직접 포함
  같은 타입 두 번 사용 → @AttributeOverride 필수
  기본 생성자 protected 필요 (JPA Reflection 사용)
  각 필드가 독립 컬럼 → DB 쿼리, 인덱스 가능

AttributeConverter:
  단순 VO (단일 컬럼) → DB 타입으로 변환
  autoApply = true → 해당 타입 전체 자동 적용
  기본 생성자 불필요
  변환 로직이 Converter에 집중

선택 기준:
  "이 VO를 몇 개 컬럼으로 저장하는가?" → 1개: Converter, 2개+: Embeddable
  "내부 필드로 DB 쿼리가 필요한가?" → YES: Embeddable

null 방지:
  @Column(nullable = false) + Null Object 패턴 조합
```

---

## 🤔 생각해볼 문제

**Q1.** `Money`를 `@Embeddable`로 매핑할 때 `orders` 테이블에 `totalAmount`와 `shippingFee` 두 곳에서 쓴다. `@AttributeOverride`를 빠뜨리면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Hibernate 시작 시 예외가 발생하거나 컬럼명이 충돌해 의도치 않은 공유가 발생합니다.**

```java
@Entity
public class Order {
    @Embedded
    private Money totalAmount;     // → 컬럼: amount, currency

    @Embedded
    private Money shippingFee;     // → 컬럼: amount, currency  ← 충돌!
}
```

Hibernate 결과:
- 최신 Hibernate: `org.hibernate.MappingException: Repeated column in mapping for entity: amount`
- 오래된 버전: 두 필드가 같은 컬럼을 공유 → 하나 저장 시 다른 하나 덮어씌움

**해결:**
```java
@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "amount",   column = @Column(name = "total_amount")),
    @AttributeOverride(name = "currency", column = @Column(name = "total_currency"))
})
private Money totalAmount;

@Embedded
@AttributeOverrides({
    @AttributeOverride(name = "amount",   column = @Column(name = "shipping_fee_amount")),
    @AttributeOverride(name = "currency", column = @Column(name = "shipping_fee_currency"))
})
private Money shippingFee;
```

</details>

---

**Q2.** `AttributeConverter`의 `convertToEntityAttribute()`에서 검증 실패(`new Email("invalid")` 예외)가 발생하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**Entity 로드 중 예외가 발생하므로 해당 Entity를 아예 조회할 수 없게 됩니다.**

```java
@Override
public Email convertToEntityAttribute(String dbValue) {
    return new Email(dbValue);  // DB에 저장된 값이 유효하지 않으면 예외!
}
```

발생 시나리오:
- DB에 `"invalid-email"` 이 저장돼 있는데 `Email` 생성자가 형식 검증
- `orderRepository.findById(id)` → `convertToEntityAttribute()` → 예외
- → 해당 주문을 전혀 조회할 수 없음

**방어 전략:**
```java
@Override
public Email convertToEntityAttribute(String dbValue) {
    if (dbValue == null) return null;
    try {
        return new Email(dbValue);
    } catch (InvalidEmailException e) {
        // 로그 기록 후 Null 처리 또는 raw 값 보존
        log.warn("DB에 유효하지 않은 이메일 저장됨: {}", dbValue);
        return Email.ofUnchecked(dbValue);  // 검증 없는 복원용 팩토리
    }
}
```

근본 해결: DB에 저장되는 시점에 검증이 완료돼야 합니다. AttributeConverter의 검증은 "이미 유효한 값을 복원"하는 것이지 "새 입력을 검증"하는 것이 아닙니다.

</details>

---

**Q3.** `@Embeddable` VO의 모든 컬럼이 null인 경우, JPA가 VO 객체를 null로 반환하는데 이것이 문제가 되는 경우와 해결 방법은?

<details>
<summary>해설 보기</summary>

**배송 전인 주문의 `shippingAddress`가 null로 반환될 때 NullPointerException 위험이 있습니다.**

```java
// DB: shipping_city = NULL, shipping_street = NULL
Order order = orderRepository.findById(id).orElseThrow();
order.shippingAddress().city()  // → NullPointerException! (address가 null)
```

**해결 방법 1: `@Column(nullable = false)` 강제**
```java
@AttributeOverride(name = "city", column = @Column(name = "city", nullable = false))
// → DB 제약으로 null 저장 불가 → JPA가 항상 객체 반환
```

**해결 방법 2: Null Object 패턴**
```java
public static final Address UNKNOWN = new Address("미정", "미정", new ZipCode("00000"));

// @PostLoad로 null 방지
@PostLoad
private void initNullAddresses() {
    if (shippingAddress == null) {
        shippingAddress = Address.UNKNOWN;
    }
}
```

**해결 방법 3: `@Embedded` 중첩 VO에 기본값 적용**
```java
@Embedded
private ShippingAddress shippingAddress = ShippingAddress.PENDING;
// 초기값을 Null Object로 설정 → 저장 전에도 null 없음
```

</details>

---

<div align="center">

**[⬅️ 이전: JPA Entity와 DDD Entity 분리](./02-jpa-vs-ddd-entity.md)** | **[홈으로 🏠](../README.md)** | **[다음: Aggregate JPA 매핑 ➡️](./04-aggregate-jpa-mapping.md)**

</div>
