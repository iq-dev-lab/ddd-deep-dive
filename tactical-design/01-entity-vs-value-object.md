# Entity vs Value Object — 식별자 vs 값으로 구분

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Entity와 Value Object를 구분하는 핵심 기준은 무엇인가?
- `Money`, `Address`, `Email`을 왜 Value Object로 만들어야 하는가?
- `equals()` / `hashCode()` 구현 방식이 Entity와 Value Object에서 어떻게 달라야 하는가?
- 같은 개념이 어느 Context에서는 Entity이고 다른 Context에서는 Value Object가 될 수 있는가?
- Primitive Obsession 안티패턴은 무엇이고, Value Object로 어떻게 해소하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

`Long orderId`, `String email`, `BigDecimal amount`... 대부분의 코드는 원시 타입(Primitive)으로 가득하다. 이 코드들은 동작은 하지만 도메인의 의도를 드러내지 못한다. `amount`가 원화인지 달러인지, 음수가 가능한지, 소수점 몇 자리까지인지를 타입이 표현하지 못하기 때문에 개발자가 직접 검증 로직을 추가한다 — 그것도 여기저기 흩어진 채로.

Entity와 Value Object의 구분은 도메인 모델을 올바르게 표현하는 첫 걸음이다. **Entity는 식별자로 추적되는 것**, **Value Object는 값 자체가 의미인 것**. 이 구분이 명확해야 `equals()` 구현이 올바르고, 불변성 적용 대상이 명확해지며, 도메인 규칙이 올바른 객체에 위치한다.

---

## 😱 흔한 실수 (Before — 모든 것을 Entity로 만들거나, 원시 타입으로 방치)

```
패턴 1: 모든 것을 @Entity JPA 클래스로 만들기

@Entity
public class Address {
    @Id @GeneratedValue
    private Long id;         // ← 주소가 왜 ID를 갖는가?
    private String city;
    private String street;
    private String zipCode;
}

@Entity
public class Money {
    @Id @GeneratedValue
    private Long id;         // ← 돈이 왜 ID를 갖는가?
    private BigDecimal amount;
    private String currency;
}

문제:
  Address를 비교할 때: address1.equals(address2) → ID로 비교 (같은 주소여도 다른 행이면 false)
  "10,000원 == 10,000원" → false (ID가 다르면 다른 돈)
  DB에 address 테이블, money 테이블이 생김
  → "서울시 강남구 테헤란로"가 수백만 개의 행으로 중복 저장
```

```java
// 패턴 2: Primitive Obsession — 원시 타입으로 도메인을 표현

@Service
public class OrderService {

    public void placeOrder(Long customerId, String email, BigDecimal amount, String currency,
                           String city, String street, String zipCode) {
        // 검증 로직이 Service에 흩어짐
        if (amount == null || amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new IllegalArgumentException("금액은 양수여야 합니다");
        }
        if (email == null || !email.contains("@")) {
            throw new IllegalArgumentException("유효하지 않은 이메일");
        }
        if (zipCode == null || zipCode.length() != 5) {
            throw new IllegalArgumentException("우편번호는 5자리여야 합니다");
        }
        // ... 검증이 Service에, 또 다른 Service에, Controller에도 중복됨

        // 금액 계산 시 실수 가능
        BigDecimal discounted = amount.multiply(new BigDecimal("0.9")); // 10% 할인
        // currency 검증 없이 그냥 사용 → 원화와 달러를 더하는 버그 가능
    }
}
```

```
구체적 버그 사례:
  주문 금액 계산:
    BigDecimal krwAmount = new BigDecimal("10000");
    BigDecimal usdAmount = new BigDecimal("100");
    BigDecimal total = krwAmount.add(usdAmount); // 10100 — 원화와 달러 합산!
  
  이 버그가 컴파일 타임에 잡히지 않음
  타입이 둘 다 BigDecimal이므로 Java가 허용
  테스트 없으면 운영에서 발견됨
```

---

## ✨ 올바른 접근 (After — Entity와 Value Object를 구분)

```java
// Value Object: Money — 값이 같으면 동일, 불변, 자기 검증
public final class Money {

    private final BigDecimal amount;
    private final Currency currency;

    // 생성 시점에 스스로 검증
    public Money(BigDecimal amount, Currency currency) {
        if (amount == null) throw new IllegalArgumentException("금액은 null일 수 없습니다");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("금액은 음수일 수 없습니다: " + amount);
        }
        if (currency == null) throw new IllegalArgumentException("통화는 null일 수 없습니다");
        this.amount = amount.setScale(currency.defaultFractionDigits(), RoundingMode.HALF_UP);
        this.currency = currency;
    }

    // 팩토리 메서드로 의도 명확히
    public static Money ofKrw(long amount) {
        return new Money(BigDecimal.valueOf(amount), Currency.KRW);
    }

    public static Money ofUsd(BigDecimal amount) {
        return new Money(amount, Currency.USD);
    }

    // 도메인 로직을 Value Object가 가짐
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException(
                "다른 통화끼리 더할 수 없습니다: " + this.currency + " + " + other.currency
            );
        }
        return new Money(this.amount.add(other.amount), this.currency);  // 불변: 새 객체 반환
    }

    public Money multiply(double rate) {
        return new Money(this.amount.multiply(BigDecimal.valueOf(rate)), this.currency);
    }

    public boolean isGreaterThan(Money other) {
        assertSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    // 값 동등성: 금액과 통화가 같으면 동일
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Money)) return false;
        Money money = (Money) o;
        return amount.compareTo(money.amount) == 0 && currency == money.currency;
    }

    @Override
    public int hashCode() {
        return Objects.hash(amount, currency);
    }
}

// Entity: Order — ID로 추적, 변경 가능
public class Order {

    private final OrderId id;           // 식별자 — 불변 (한번 생성된 ID는 바뀌지 않음)
    private OrderStatus status;          // 상태 — 변경 가능
    private Money totalAmount;           // Value Object 사용
    private ShippingAddress address;     // Value Object 사용

    // Entity equals: ID가 같으면 동일 (내용이 달라도)
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof Order)) return false;
        Order order = (Order) o;
        return id.equals(order.id);      // ID 기반 동등성
    }

    @Override
    public int hashCode() {
        return id.hashCode();
    }
}
```

```
결과:
  Money.ofKrw(10000).add(Money.ofUsd(100))
  → CurrencyMismatchException 즉시 발생
  → 원화/달러 합산 버그가 컴파일 타임 직후 테스트에서 잡힘

  Address 비교:
    new Address("서울", "강남구", "12345")
      .equals(new Address("서울", "강남구", "12345"))
    → true (같은 주소 = 동일)
```

---

## 🔬 내부 동작 원리

### 1. 구분 기준: 식별자(Identity)를 갖는가

```
Entity:
  "이것이 시간이 지나도 같은 것임을 어떻게 아는가?"
  → 식별자(ID)가 있기 때문
  
  예: Order #1234는 내용이 바뀌어도 여전히 Order #1234
      고객 Kim은 주소가 바뀌어도 여전히 고객 Kim
      상품 A는 가격이 바뀌어도 여전히 상품 A
  
  특징:
    고유한 ID를 가짐
    시간에 따라 상태가 변함 (mutable)
    ID가 같으면 같은 것 (equals = ID 비교)
    생명주기가 있음 (생성 → 변경 → 소멸)

Value Object:
  "이것의 정체성은 값 자체이다"
  → 100달러는 어느 100달러나 동일, 별도 ID 불필요
  
  예: Money(100, KRW) = Money(100, KRW) → 항상 같음
      Address("서울", "강남구") = Address("서울", "강남구") → 항상 같음
      Color.RED = Color.RED → 항상 같음
  
  특징:
    ID 없음 (값 자체가 정체성)
    불변 (immutable) — 변경하면 새 객체
    값이 같으면 같은 것 (equals = 모든 필드 비교)
    소유자(Entity)와 함께 존재

판단 질문:
  "이것을 복사해도 원본과 동일한가?"
  → YES → Value Object (Money 복사본은 원본과 같음)
  → NO  → Entity (고객 복사본은 "또 다른 고객", 원본과 다름)
```

### 2. equals() / hashCode()의 의미 차이

```
Entity의 equals() — 생애 동안 ID가 같으면 같은 것:

  Order order1 = Order.place(customerId, items);  // id = #1234
  Order order2 = orderRepository.findById(new OrderId(1234));

  order1.equals(order2)  // → true (같은 ID)
  // order1.totalAmount != order2.totalAmount 여도 같은 주문

  // ID 기반 equals 구현
  @Override
  public boolean equals(Object o) {
      if (!(o instanceof Order)) return false;
      return this.id.equals(((Order) o).id);
  }

─────────────────────────────────────────────────────

Value Object의 equals() — 모든 구성 요소가 같으면 같은 것:

  Money price1 = Money.ofKrw(10000);
  Money price2 = Money.ofKrw(10000);

  price1 == price2   // → false (다른 인스턴스)
  price1.equals(price2)  // → true (같은 값)

  Set<Money> set = new HashSet<>();
  set.add(price1);
  set.contains(price2)  // → true (같은 값이므로 이미 있는 것으로 인식)

  // 모든 필드 기반 equals 구현
  @Override
  public boolean equals(Object o) {
      if (!(o instanceof Money)) return false;
      Money m = (Money) o;
      return amount.compareTo(m.amount) == 0 && currency == m.currency;
  }

─────────────────────────────────────────────────────

잘못된 구현이 만드는 버그:

  // Entity를 값 비교로 구현하면:
  if (order1.equals(order2)) {
      // 같은 ID인데 내용이 다르면 equal인가? → 혼란
  }

  // Value Object를 ID 비교로 구현하면:
  Money usd100a = Money.ofUsd(100);
  Money usd100b = Money.ofUsd(100);
  Map<Money, String> priceMap = new HashMap<>();
  priceMap.put(usd100a, "high");
  priceMap.get(usd100b)  // → null! (같은 값인데 찾지 못함)
```

### 3. 같은 개념이 Context마다 달라지는 경우

```
"직원(Employee)"의 분류:

HR 컨텍스트:
  Employee = Entity
  이유: "직원 Kim"은 부서가 바뀌어도, 연봉이 바뀌어도 여전히 같은 직원
  → ID(사번)로 추적, 변경 가능, 생명주기 있음

급여 계산 컨텍스트:
  EmployeeSnapshot = Value Object
  이유: 이번 달 급여 계산에 사용된 "직원 Kim의 정보"는
        "사번, 이름, 직급, 연봉" 값의 조합
        나중에 직원 정보가 바뀌어도 이번 달 계산은 고정됨
  → 불변, 값 동등성

"주소(Address)"의 분류:

회원 서비스:
  Address = Value Object
  이유: 주소가 바뀌면 새 주소로 교체 (기존 주소는 관심 없음)
  → "서울 강남구" = "서울 강남구" (값 동등성)

부동산 서비스:
  Address = Entity
  이유: 특정 주소는 가격 이력, 거래 이력이 있음
        "서울 강남구 테헤란로 123"은 하나뿐인 고유 위치
  → ID로 추적, 변경 이력 관리
```

### 4. Primitive Obsession과 Value Object의 관계

```
Primitive Obsession 안티패턴:
  도메인 개념을 원시 타입(String, Long, int)으로 표현

증상:
  - String email (검증 로직이 어디든 필요)
  - Long orderId (OrderId와 ProductId가 혼용될 수 있음)
  - BigDecimal amount (어느 통화인지 알 수 없음)
  - int quantity (음수 가능, 단위 불명확)

문제:
  void ship(Long orderId, Long productId) { ... }
  ship(productId, orderId);  // 컴파일 OK! 런타임 버그

Value Object로 해소:
  void ship(OrderId orderId, ProductId productId) { ... }
  ship(productId, orderId);  // 컴파일 에러! 타입 불일치

  → 파라미터 순서 실수를 컴파일 타임에 잡음
  → 검증 로직이 Value Object 안에 집중됨
  → 도메인 의도가 타입 이름에 드러남

Value Object 후보 발굴 체크리스트:
  □ 이 String은 특정 형식이 있는가? (Email, PhoneNumber, ZipCode)
  □ 이 Long은 특정 도메인 개념의 ID인가? (OrderId, ProductId)
  □ 이 BigDecimal은 단위/통화가 있는가? (Money, Percentage)
  □ 이 int는 음수가 불가능한가? (Quantity, Count)
  □ 두 원시 타입이 항상 함께 다니는가? (city + street + zipCode → Address)
```

---

## 💻 실전 코드

### JPA에서 Value Object 매핑 — @Embeddable

```java
// Value Object: JPA @Embeddable로 별도 테이블 없이 포함
@Embeddable
public final class Money {

    @Column(name = "amount", precision = 19, scale = 2)
    private final BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(name = "currency")
    private final Currency currency;

    protected Money() {}  // JPA 요구사항

    public Money(BigDecimal amount, Currency currency) {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new IllegalArgumentException("음수 금액 불가: " + amount);
        }
        this.amount = amount;
        this.currency = currency;
    }

    // 불변: 연산은 새 객체를 반환
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException(currency + " vs " + other.currency);
        }
        return new Money(this.amount.add(other.amount), this.currency);
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

// Entity: Order에서 Money를 @Embedded로 사용
@Entity
@Table(name = "orders")
public class Order {

    @EmbeddedId
    private OrderId id;

    @Embedded
    @AttributeOverrides({
        @AttributeOverride(name = "amount", column = @Column(name = "total_amount")),
        @AttributeOverride(name = "currency", column = @Column(name = "total_currency"))
    })
    private Money totalAmount;  // orders 테이블에 total_amount, total_currency 컬럼으로 저장
}
```

### Java Record를 활용한 간결한 Value Object

```java
// Java 16+ Record — 불변, equals/hashCode/toString 자동 생성
public record Email(String value) {

    // compact constructor: 생성 시 검증
    public Email {
        Objects.requireNonNull(value, "이메일은 null일 수 없습니다");
        if (!value.matches("^[^@]+@[^@]+\\.[^@]+$")) {
            throw new InvalidEmailException("유효하지 않은 이메일 형식: " + value);
        }
        value = value.toLowerCase();  // 정규화
    }

    public String domain() {
        return value.substring(value.indexOf('@') + 1);
    }
}

public record OrderId(Long value) {
    public OrderId {
        Objects.requireNonNull(value, "OrderId는 null일 수 없습니다");
        if (value <= 0) throw new IllegalArgumentException("OrderId는 양수: " + value);
    }
}

// Record 사용: 컴파일러가 equals, hashCode, toString 자동 생성
Email e1 = new Email("user@example.com");
Email e2 = new Email("USER@EXAMPLE.COM");
e1.equals(e2);  // → true (소문자 정규화로 같아짐)
```

---

## 📊 설계 비교

```
Entity vs Value Object 핵심 차이:

                Entity                    Value Object
────────────┼────────────────────────┼──────────────────────────
정체성 기준  │ ID (식별자)              │ 값 자체
────────────┼────────────────────────┼──────────────────────────
가변성       │ Mutable (상태 변경 가능) │ Immutable (변경 불가)
            │                        │ → 변경 시 새 객체 생성
────────────┼────────────────────────┼──────────────────────────
equals()    │ ID 비교                 │ 모든 속성 비교
────────────┼────────────────────────┼──────────────────────────
생명주기     │ 생성 → 변경 → 소멸      │ 생성 (소유자와 함께)
────────────┼────────────────────────┼──────────────────────────
DB 매핑     │ 별도 테이블 (일반적)     │ @Embeddable (소유자 테이블)
            │                        │ 또는 @Converter
────────────┼────────────────────────┼──────────────────────────
예시        │ Order, Customer, Product│ Money, Address, Email,
            │ Employee, Invoice       │ Quantity, DateRange
────────────┼────────────────────────┼──────────────────────────
복사        │ 복사본은 다른 Entity     │ 복사본은 원본과 동일
────────────┼────────────────────────┼──────────────────────────
null 처리   │ 특별한 상태 (Optional 등)│ Null Object 패턴 가능
            │                        │ Money.ZERO = 0원
```

---

## ⚖️ 트레이드오프

```
Value Object 도입의 비용:
  ① 클래스 수 증가
     String email → Email 클래스
     BigDecimal amount + String currency → Money 클래스
     → 파일 수 증가, 초기 작업량 증가
  
  ② JPA 매핑 복잡도
     @Embeddable 어노테이션, @AttributeOverride
     컬럼명 충돌 시 수동 설정 필요
  
  ③ 직렬화 주의
     Jackson 직렬화: @JsonCreator 또는 커스텀 Deserializer 필요
     JPA protected 생성자 필요

Value Object 도입의 이점:
  ① 검증 로직 집중: Email 검증이 Email 안에 → Service에 중복 없음
  ② 타입 안전: ship(orderId, productId) 순서 실수를 컴파일 타임에 방지
  ③ 도메인 로직 표현: money.add(other), dateRange.contains(date)
  ④ 불변성으로 공유 안전: 여러 곳에 같은 Money 인스턴스 전달해도 안전

언제 원시 타입이 나은가:
  도메인 의미가 없는 단순 기술 값 (row_count, retry_count 등)
  팀이 Value Object 패턴에 아직 익숙하지 않을 때 점진적 도입
  → 핵심 Value Object(Money, OrderId)부터 시작
```

---

## 📌 핵심 정리

```
Entity vs Value Object 핵심:

Entity:
  식별자(ID)로 추적 → ID가 같으면 같은 것
  상태 변경 가능 (Mutable)
  생명주기 있음 (생성, 변경, 소멸)
  equals() = ID 비교

Value Object:
  값 자체가 정체성 → 모든 필드가 같으면 같은 것
  불변 (Immutable) → 연산은 새 객체 반환
  자기 검증 (생성 시점에 유효성 확인)
  equals() = 모든 필드 비교

구분 질문:
  "이것을 복사하면 원본과 동일한가?" → VO
  "이것은 시간이 지나도 같은 것으로 추적해야 하는가?" → Entity

Primitive Obsession 해소:
  String → Email, PhoneNumber, ZipCode
  Long → OrderId, ProductId (혼용 방지)
  BigDecimal + String → Money (통화 포함)
  int → Quantity, Count (음수 방지)

JPA 매핑:
  Entity → @Entity, @Id
  Value Object → @Embeddable (소유자 테이블에 컬럼으로)
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 중 Entity와 Value Object를 구분하고, 이유를 설명하세요.

```
A. 은행 계좌 (Account)
B. 화폐 단위 (Currency: KRW, USD)
C. 계좌 이체 내역의 "보낸 사람" 정보 (이름, 계좌번호)
D. 배송 추적 번호 (TrackingNumber: "CJ1234567890KR")
E. 좌표 (Coordinate: 위도 37.5, 경도 127.0)
```

<details>
<summary>해설 보기</summary>

**A. 은행 계좌 → Entity**
계좌번호라는 고유 식별자로 추적. 잔액이 바뀌어도 "같은 계좌". 생명주기 있음 (개설 → 거래 → 해지).

**B. 화폐 단위 → Value Object (또는 Enum)**
`Currency.KRW`와 `Currency.KRW`는 항상 동일. 별도 ID 없이 값 자체가 정체성. 불변.

**C. 보낸 사람 정보 → Value Object**
이체 내역에 스냅샷으로 포함되는 값. 나중에 그 사람의 이름이 바뀌어도 이체 내역의 "보낸 사람"은 이체 당시 정보로 고정. → 불변, 값 동등성.

**D. 배송 추적 번호 → Value Object**
"CJ1234567890KR"이라는 문자열 값이 정체성. 같은 번호면 같은 것. Shipment Entity가 TrackingNumber VO를 소유하는 구조.

```java
public record TrackingNumber(String value) {
    public TrackingNumber {
        if (!value.matches("[A-Z]{2}\\d{10}KR")) {
            throw new InvalidTrackingNumberException(value);
        }
    }
    public Carrier inferCarrier() {
        return Carrier.fromPrefix(value.substring(0, 2));
    }
}
```

**E. 좌표 → Value Object**
위도 37.5, 경도 127.0이라는 값의 조합이 정체성. 같은 좌표면 같은 지점. 지도 앱에서 "이 좌표로 가겠습니다"는 그 값 자체가 의미.

</details>

---

**Q2.** `Order` Entity 안에 `List<OrderLine>` 이 있다. `OrderLine`은 Entity인가, Value Object인가?

<details>
<summary>해설 보기</summary>

**문맥에 따라 다르지만, 대부분의 경우 Value Object 또는 Entity로 보는 것이 적합합니다.**

**Value Object로 보는 경우 (일반적):**
- `OrderLine`은 특정 시점의 "상품 X를 Y개, Z원에" 라는 값의 스냅샷
- `OrderLine`을 직접 수정하기보다 삭제 후 재추가
- Order 없이 독립적으로 존재하지 않음

```java
// Value Object로서의 OrderLine
public final class OrderLine {
    private final ProductId productId;
    private final int quantity;
    private final Money unitPrice;

    public Money subtotal() {
        return unitPrice.multiply(quantity);
    }

    @Override
    public boolean equals(Object o) {
        // 모든 필드 비교
        if (!(o instanceof OrderLine)) return false;
        OrderLine other = (OrderLine) o;
        return productId.equals(other.productId)
            && quantity == other.quantity
            && unitPrice.equals(other.unitPrice);
    }
}
```

**Entity로 보는 경우:**
- `OrderLine`이 독립적인 수정 이력이 필요할 때 (예: "주문 항목 X의 수량을 3번 변경한 이력")
- 주문 항목별 배송 상태가 개별적으로 관리될 때

**JPA에서 OrderLine:**
VO에 가깝지만 JPA에서는 `@ElementCollection`(별도 테이블) 또는 `@OneToMany` + `@Entity`로 매핑. 완전한 VO 의미론을 지키려면 `@ElementCollection` 사용.

</details>

---

**Q3.** `Money` Value Object를 JPA와 Jackson(JSON 직렬화) 양쪽에서 사용할 때 발생하는 기술적 문제와 해결 방법은?

<details>
<summary>해설 보기</summary>

**JPA 문제와 해결:**

```java
@Embeddable
public final class Money {
    // JPA는 기본 생성자를 요구함 (no-arg constructor)
    protected Money() {}  // JPA 전용, private 불가
    // → Reflection으로 접근하므로 protected로 숨김
    // → 외부에서는 팩토리 메서드만 사용 유도
}
```

**Jackson 문제와 해결:**

```java
// 문제: Money의 생성자가 public이 아니거나 @JsonCreator 없으면 역직렬화 실패
public final class Money {

    @JsonCreator
    public static Money of(
        @JsonProperty("amount") BigDecimal amount,
        @JsonProperty("currency") Currency currency
    ) {
        return new Money(amount, currency);
    }

    @JsonValue  // 또는 별도 @JsonSerialize
    public Map<String, Object> toJson() {
        return Map.of("amount", amount, "currency", currency);
    }
}

// 또는 Jackson Module로 전역 처리
@Configuration
public class JacksonConfig {
    @Bean
    public ObjectMapper objectMapper() {
        ObjectMapper mapper = new ObjectMapper();
        // Money 같은 단순 Value Object는 amount 필드를 직접 직렬화
        mapper.configure(SerializationFeature.WRITE_ENUMS_USING_TO_STRING, true);
        return mapper;
    }
}
```

**Java Record 사용 시 Jackson:**

```java
// Java Record는 Jackson 2.12+ 에서 자동 지원
public record Money(BigDecimal amount, Currency currency) {
    // Jackson이 record의 컴포넌트를 자동으로 직렬화/역직렬화
    // @JsonCreator 불필요 (Jackson 2.12+)
}
```

</details>

---

<div align="center">

**[⬅️ 이전: Bounded Context와 마이크로서비스](../strategic-design/07-context-vs-microservice.md)** | **[홈으로 🏠](../README.md)** | **[다음: Value Object 설계 원칙 ➡️](./02-value-object-design.md)**

</div>
