# Value Object 설계 원칙 — 불변성과 자기 검증

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Value Object의 불변성이 왜 "공유 인스턴스 버그"를 막는가?
- Value Object가 생성 시점에 스스로를 검증해야 하는 이유(Self-Validation)는?
- Value Object가 단순 데이터 홀더를 넘어 도메인 로직을 가질 수 있는 이유는?
- Primitive Obsession 안티패턴을 구체적으로 어떻게 해소하는가?
- Value Object를 설계할 때 흔히 빠지는 함정은 무엇인가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Value Object는 그냥 getter만 있는 DTO 아닌가?" 라는 오해가 많다. 그렇지 않다. Value Object의 **불변성**은 공유 상태로 인한 버그를 원천 차단하고, **자기 검증**은 항상 유효한 객체를 보장한다. 그리고 Value Object는 단순한 데이터 묶음이 아니라 **도메인 로직의 적절한 위치**다.

`Money.add()`, `DateRange.contains()`, `Email.domain()`처럼 Value Object에 도메인 로직이 있으면 Service에 흩어지던 계산 로직이 응집된다. 이것이 "풍부한 도메인 모델(Rich Domain Model)"의 출발점이다.

---

## 😱 흔한 실수 (Before — 가변 Value Object와 검증 분산)

```java
// 안티패턴 1: 가변 Value Object — 공유 시 버그 발생
public class Money {
    private BigDecimal amount;
    private Currency currency;

    public void setAmount(BigDecimal amount) {  // ← setter 있음! 가변!
        this.amount = amount;
    }
}

// 이 코드에서 발생하는 버그:
@Service
public class DiscountService {
    public void apply10PercentDiscount(Order order) {
        Money price = order.getTotalAmount();  // Money 참조를 가져옴
        price.setAmount(price.getAmount().multiply(new BigDecimal("0.9")));
        // ↑ order.totalAmount도 변경됨! 같은 객체를 참조하기 때문
    }
}

Order order1 = Order.place(...);
Order order2 = new Order(order1.getTotalAmount(), ...);  // 같은 Money 인스턴스 공유
discountService.apply10PercentDiscount(order1);
// → order2.totalAmount도 할인됨! 공유 인스턴스 문제
```

```java
// 안티패턴 2: 검증이 생성 시점이 아닌 사용 시점에
public class Money {
    public final BigDecimal amount;
    public final Currency currency;

    public Money(BigDecimal amount, Currency currency) {
        this.amount = amount;   // ← 검증 없음
        this.currency = currency;
    }
}

// 검증 로직이 여기저기 흩어짐
@Service
public class OrderService {
    public void placeOrder(Money price) {
        if (price.amount == null || price.amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new InvalidPriceException(); // Service에서 검증
        }
        // ...
    }
}

@Service
public class PaymentService {
    public void processPayment(Money amount) {
        if (amount.amount.compareTo(BigDecimal.ZERO) <= 0) {
            throw new InvalidAmountException(); // 또 다른 Service에서 중복 검증
        }
    }
}

// 결과: 검증이 빠진 Service에서 음수 금액 처리 버그 발생 가능
```

---

## ✨ 올바른 접근 (After — 불변 + 자기 검증 + 도메인 로직 포함)

```java
public final class Money {  // final: 상속 불가

    private final BigDecimal amount;   // final: 한 번 설정 후 변경 불가
    private final Currency currency;

    // Self-Validation: 생성 시점에 모든 검증
    public Money(BigDecimal amount, Currency currency) {
        Objects.requireNonNull(amount, "금액은 null일 수 없습니다");
        Objects.requireNonNull(currency, "통화는 null일 수 없습니다");
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new NegativeAmountException("금액은 음수일 수 없습니다: " + amount);
        }
        // 정규화: 소수점 이하 자릿수 통일
        this.amount = amount.setScale(2, RoundingMode.HALF_UP);
        this.currency = currency;
    }

    // 명시적 팩토리 메서드 — 의도 명확
    public static Money ofKrw(long amount) {
        return new Money(BigDecimal.valueOf(amount), Currency.KRW);
    }

    public static Money ofUsd(BigDecimal amount) {
        return new Money(amount, Currency.USD);
    }

    public static final Money ZERO_KRW = Money.ofKrw(0);  // Null Object

    // 도메인 로직: 연산은 새 인스턴스 반환 (불변성 유지)
    public Money add(Money other) {
        requireSameCurrency(other);
        return new Money(this.amount.add(other.amount), this.currency);
    }

    public Money subtract(Money other) {
        requireSameCurrency(other);
        BigDecimal result = this.amount.subtract(other.amount);
        if (result.compareTo(BigDecimal.ZERO) < 0) {
            throw new InsufficientAmountException(
                "결과가 음수입니다: " + this + " - " + other
            );
        }
        return new Money(result, this.currency);
    }

    public Money multiply(double rate) {
        if (rate < 0) throw new IllegalArgumentException("비율은 음수일 수 없습니다: " + rate);
        return new Money(this.amount.multiply(BigDecimal.valueOf(rate)), this.currency);
    }

    public Money discountBy(Percentage discount) {
        return multiply(1.0 - discount.value());
    }

    public boolean isGreaterThan(Money other) {
        requireSameCurrency(other);
        return this.amount.compareTo(other.amount) > 0;
    }

    public boolean isZero() {
        return this.amount.compareTo(BigDecimal.ZERO) == 0;
    }

    private void requireSameCurrency(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new CurrencyMismatchException(this.currency + " ≠ " + other.currency);
        }
    }

    // 값 동등성
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

    @Override
    public String toString() {
        return amount + " " + currency;
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 불변성이 공유 인스턴스 버그를 막는 원리

```
가변 객체의 공유 문제:

┌──────────┐   참조    ┌──────────┐
│  Order1  │ ────────> │  Money   │ amount = 10000
└──────────┘           └──────────┘
┌──────────┐   참조          ↑
│  Order2  │ ────────────────┘  ← 같은 인스턴스 공유!
└──────────┘

order1.getTotalAmount().setAmount(9000);
// → order2.getTotalAmount().amount도 9000으로 변경됨!
// → Order2가 모르는 사이에 금액이 바뀜

불변 객체의 공유는 안전:

┌──────────┐   참조    ┌──────────┐
│  Order1  │ ────────> │  Money   │ amount = 10000 (불변)
└──────────┘           └──────────┘
┌──────────┐   참조          ↑
│  Order2  │ ────────────────┘  ← 같은 인스턴스 공유해도 안전!
└──────────┘

Money discounted = order1.getTotalAmount().multiply(0.9);
// → 새 Money(9000) 인스턴스 생성
// → order1.totalAmount는 여전히 10000
// → order2.totalAmount도 여전히 10000
// → 공유해도 안전!

불변성의 추가 이점:
  스레드 안전: 여러 스레드가 같은 Money 인스턴스 읽어도 안전
  캐싱 가능: 같은 값이면 같은 인스턴스 재사용 (Flyweight)
  HashSet/HashMap 키 사용 안전: hashCode가 변하지 않음
```

### 2. Self-Validation의 보장 범위

```
Self-Validation의 핵심 원칙:
  "Money 인스턴스가 존재한다면 항상 유효하다"

일반적 검증 방식:
  Money 생성 → 어딘가에서 검증 → 사용
  문제: "어딘가"가 빠질 수 있음

Self-Validation:
  Money 생성 자체가 검증 → 생성됐으면 항상 유효
  문제: 검증 실패 시 생성 자체가 불가 (예외 발생)

검증 항목 결정:
  ✅ 항상 참이어야 하는 불변식 → 생성자에서 검증
      음수 금액 불가
      null 불가
      이메일 형식 필수

  ❌ 비즈니스 규칙 → Application Service에서 검증
      "이 고객은 이 금액을 지불할 능력이 있는가?" — VO가 알 수 없음
      "오늘은 할인 이벤트 기간인가?" — 외부 상태 의존

  ⚠️ 경계 케이스:
      "주문 금액은 최소 1,000원 이상" — 비즈니스 규칙
      Money 자체의 규칙인가, 주문 도메인의 규칙인가?
      → Order.place()에서 검증하는 것이 적합 (Money는 1원도 표현 가능)
```

### 3. Value Object의 도메인 로직

```
Value Object에 도메인 로직을 두는 이유:

Service에 로직이 있을 때:
  @Service
  public class PricingService {
      public Money applyMemberDiscount(Money price, MembershipLevel level) {
          if (level == MembershipLevel.GOLD) {
              return new Money(price.amount.multiply(new BigDecimal("0.9")), price.currency);
          }
          return price;
      }
  }
  
  → 금액 계산 로직이 Service에 흩어짐
  → 여러 Service에 중복될 수 있음
  → "10% 할인"의 도메인 지식이 Service에 숨음

Value Object에 로직이 있을 때:
  // Money가 스스로 계산
  public Money discountBy(Percentage discount) {
      return multiply(1.0 - discount.value());
  }

  // Percentage도 Value Object
  public record Percentage(double value) {
      public Percentage {
          if (value < 0 || value > 100) {
              throw new IllegalArgumentException("비율은 0~100 사이: " + value);
          }
      }
  }

  // 사용:
  Money original = Money.ofKrw(10000);
  Money discounted = original.discountBy(new Percentage(10));
  // → 명확하고 테스트하기 쉬움

Value Object에 두면 좋은 도메인 로직:
  ✅ 해당 VO의 값만으로 결정되는 계산
      Money.add(), Money.multiply()
      DateRange.contains(LocalDate)
      Email.domain(), PhoneNumber.format()
  
  ❌ 외부 상태(다른 Aggregate, DB)에 의존하는 로직
      "이 이메일이 이미 등록됐는가?" — VO가 알 수 없음
      → Repository가 필요한 로직은 Domain Service로
```

### 4. 다양한 Value Object 설계 패턴

```
패턴 1: 단순 래핑 — Primitive 보호
  public record OrderId(Long value) {
      public OrderId { Objects.requireNonNull(value); }
  }

패턴 2: 복합 값 — 여러 원시 타입의 묶음
  public final class Address {
      private final String city;
      private final String street;
      private final ZipCode zipCode;  // ZipCode도 VO
      // ...
  }

패턴 3: 자체 계산 — 도메인 연산 포함
  public final class DateRange {
      private final LocalDate start;
      private final LocalDate end;

      public DateRange(LocalDate start, LocalDate end) {
          Objects.requireNonNull(start);
          Objects.requireNonNull(end);
          if (end.isBefore(start)) {
              throw new InvalidDateRangeException("종료일은 시작일 이후여야 합니다");
          }
          this.start = start;
          this.end = end;
      }

      public boolean contains(LocalDate date) {
          return !date.isBefore(start) && !date.isAfter(end);
      }

      public long days() { return ChronoUnit.DAYS.between(start, end); }

      public boolean overlaps(DateRange other) {
          return !end.isBefore(other.start) && !start.isAfter(other.end);
      }
  }

패턴 4: Null Object — 기본값 표현
  public class Money {
      public static final Money ZERO = Money.ofKrw(0);
      // null 대신 ZERO 사용 → NullPointerException 방지
  }

  List<Money> discounts = order.discounts();
  Money totalDiscount = discounts.stream()
      .reduce(Money.ZERO, Money::add);  // ZERO로 시작 → NPE 없음

패턴 5: Enum과 조합
  public enum Color {
      RED("#FF0000"), GREEN("#00FF00"), BLUE("#0000FF");
      
      private final String hex;
      
      public Color mix(Color other) {
          // 색상 혼합 도메인 로직
      }
  }
```

---

## 💻 실전 코드

### 실전 Value Object 모음

```java
// Email — 형식 검증 + 도메인 추출
public record Email(String value) {
    public Email {
        Objects.requireNonNull(value);
        if (!value.matches("^[a-zA-Z0-9._%+\\-]+@[a-zA-Z0-9.\\-]+\\.[a-zA-Z]{2,}$")) {
            throw new InvalidEmailException("유효하지 않은 이메일: " + value);
        }
        value = value.toLowerCase().trim();
    }

    public String domain() {
        return value.substring(value.lastIndexOf('@') + 1);
    }

    public boolean isCorporateEmail(String companyDomain) {
        return domain().equals(companyDomain);
    }
}

// PhoneNumber — 국가 코드 포함, 형식 정규화
public final class PhoneNumber {
    private final String normalized;  // +82-10-1234-5678 형식으로 정규화

    public PhoneNumber(String raw) {
        Objects.requireNonNull(raw);
        String digits = raw.replaceAll("[^0-9+]", "");
        this.normalized = normalize(digits);
    }

    private String normalize(String digits) {
        if (digits.startsWith("010") && digits.length() == 11) {
            return "+82-" + digits.substring(1, 3) + "-" + digits.substring(3, 7) + "-" + digits.substring(7);
        }
        throw new InvalidPhoneNumberException("인식할 수 없는 전화번호: " + digits);
    }

    public String formatted() { return normalized; }
}

// Quantity — 음수 방지, 단위 포함
public record Quantity(int value, Unit unit) {
    public Quantity {
        if (value < 0) throw new IllegalArgumentException("수량은 음수일 수 없습니다: " + value);
        Objects.requireNonNull(unit);
    }

    public Quantity add(Quantity other) {
        if (this.unit != other.unit) throw new UnitMismatchException();
        return new Quantity(this.value + other.value, this.unit);
    }

    public boolean isEnoughFor(Quantity required) {
        if (this.unit != required.unit) throw new UnitMismatchException();
        return this.value >= required.value;
    }
}

// Percentage — 0~100 범위 강제
public record Percentage(double value) {
    public Percentage {
        if (value < 0 || value > 100) {
            throw new IllegalArgumentException("비율은 0~100 사이여야 합니다: " + value);
        }
    }

    public static final Percentage ZERO = new Percentage(0);
    public static final Percentage HUNDRED = new Percentage(100);

    public double toDecimal() { return value / 100.0; }

    public Money applyDiscountTo(Money original) {
        return original.multiply(1.0 - toDecimal());
    }
}
```

### Value Object 단위 테스트

```java
class MoneyTest {

    @Test
    void add_sameCurrency_returnsSum() {
        Money a = Money.ofKrw(10000);
        Money b = Money.ofKrw(5000);
        assertThat(a.add(b)).isEqualTo(Money.ofKrw(15000));
    }

    @Test
    void add_differentCurrency_throwsException() {
        Money krw = Money.ofKrw(10000);
        Money usd = Money.ofUsd(new BigDecimal("10"));
        assertThatThrownBy(() -> krw.add(usd))
            .isInstanceOf(CurrencyMismatchException.class);
    }

    @Test
    void constructor_negativeAmount_throwsException() {
        assertThatThrownBy(() -> Money.ofKrw(-1))
            .isInstanceOf(NegativeAmountException.class);
    }

    @Test
    void immutability_addDoesNotModifyOriginal() {
        Money original = Money.ofKrw(10000);
        original.add(Money.ofKrw(5000));          // 연산 결과를 사용하지 않음
        assertThat(original).isEqualTo(Money.ofKrw(10000));  // 원본 불변
    }

    @Test
    void equality_sameAmountAndCurrency_isEqual() {
        Money m1 = Money.ofKrw(10000);
        Money m2 = Money.ofKrw(10000);
        assertThat(m1).isEqualTo(m2);
        assertThat(m1).hasSameHashCodeAs(m2);
    }
}
```

---

## 📊 설계 비교

```
가변 VO vs 불변 VO:

                가변 Value Object          불변 Value Object
────────────┼────────────────────────┼──────────────────────────
공유 안전성  │ ❌ 공유 인스턴스 버그    │ ✅ 공유해도 안전
────────────┼────────────────────────┼──────────────────────────
스레드 안전  │ ❌ 동기화 필요           │ ✅ 기본으로 스레드 안전
────────────┼────────────────────────┼──────────────────────────
예측 가능성  │ ❌ 언제 바뀔지 모름      │ ✅ 항상 생성 시 값 유지
────────────┼────────────────────────┼──────────────────────────
HashSet/Map  │ ❌ 값 변경 시 잃어버림   │ ✅ hashCode 불변
키 사용      │                        │
────────────┼────────────────────────┼──────────────────────────
캐싱        │ ❌ 캐시 무효화 복잡       │ ✅ 자유롭게 캐싱 가능
────────────┼────────────────────────┼──────────────────────────
테스트       │ 상태 변화 추적 필요       │ 입출력만 확인

검증 위치 비교:

                외부 검증                  Self-Validation
────────────┼────────────────────────┼──────────────────────────
검증 위치    │ Service, Controller 등  │ 생성자 (Value Object 내)
            │ 여러 곳에 분산           │
────────────┼────────────────────────┼──────────────────────────
누락 위험    │ ❌ 검증 빠질 수 있음     │ ✅ 생성 자체가 검증
────────────┼────────────────────────┼──────────────────────────
항상 유효    │ ❌ 보장 없음             │ ✅ 존재하면 항상 유효
────────────┼────────────────────────┼──────────────────────────
중복 코드    │ ❌ 검증 중복 많음        │ ✅ 한 곳에 집중
```

---

## ⚖️ 트레이드오프

```
불변성의 비용:
  ① 연산마다 새 객체 생성
     money.add(other) → 새 Money 인스턴스
     대량 계산 시 GC 부담 (일반적으로 무시 가능)
  
  ② 변경이 필요한 경우 복잡
     "금액을 점진적으로 변경" → 매번 새 인스턴스로 교체
     → 실제로는 Entity(Aggregate)가 새 VO로 교체하는 패턴으로 해결

  ③ JPA의 protected 생성자 요구
     완전한 불변이 JPA 요구사항과 충돌
     → protected 생성자는 허용 (Reflection으로만 사용)

Self-Validation의 비용:
  ① 예외가 생성자에서 발생
     try-catch가 생성자 호출을 감싸야 할 수 있음
  
  ② 너무 엄격한 검증의 부작용
     "이메일은 최신 RFC 5321 완전 검증" → 대부분의 유효한 이메일 거부
     → 실용적 수준의 검증이 적절

도메인 로직의 범위:
  Value Object에 너무 많은 로직 → "서비스" 역할을 하는 VO (안티패턴)
  Value Object에 너무 적은 로직 → Anemic VO (데이터 컨테이너)
  
  기준: "이 로직이 이 값의 종류로만 결정되는가?"
  → YES → VO에 위치
  → NO (다른 Aggregate나 외부 상태 필요) → Domain Service
```

---

## 📌 핵심 정리

```
Value Object 설계 원칙:

불변성 (Immutability):
  모든 필드: final
  클래스: final (상속 방지)
  연산: 새 인스턴스 반환
  이유: 공유 안전, 스레드 안전, 예측 가능

Self-Validation:
  생성자에서 모든 검증
  "존재하면 항상 유효" 보장
  비즈니스 정책 검증은 외부에서 (VO는 값의 유효성만)

도메인 로직:
  해당 값만으로 결정되는 계산 → VO 내부
  money.add(), dateRange.contains(), email.domain()
  외부 의존 로직 → Domain Service

Primitive Obsession 해소:
  String → 의미 있는 VO (Email, ZipCode, TrackingNumber)
  Long → 도메인 ID (OrderId, ProductId)
  BigDecimal + String → Money
  두 원시 타입이 항상 함께 → 하나의 VO로

Null Object 패턴:
  null 대신 의미 있는 기본값 VO
  Money.ZERO, Quantity.ZERO
  → NullPointerException 방지
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 Primitive Obsession을 찾고, Value Object로 개선하세요.

```java
@Service
public class LoanService {
    public void applyLoan(Long memberId, BigDecimal amount, String currency,
                          int termMonths, double annualInterestRate,
                          String city, String street, String zipCode) {
        if (annualInterestRate < 0 || annualInterestRate > 50) {
            throw new IllegalArgumentException("금리는 0~50% 사이여야 합니다");
        }
        if (termMonths <= 0 || termMonths > 360) {
            throw new IllegalArgumentException("대출 기간은 1~360개월이어야 합니다");
        }
        // ...
    }
}
```

<details>
<summary>해설 보기</summary>

**Primitive Obsession 목록과 Value Object 설계:**

```java
// 1. BigDecimal + String currency → Money
Money loanAmount = Money.of(amount, currency);

// 2. int termMonths → LoanTerm
public record LoanTerm(int months) {
    public LoanTerm {
        if (months <= 0 || months > 360) {
            throw new InvalidLoanTermException("대출 기간은 1~360개월이어야 합니다: " + months);
        }
    }
    public LoanTerm ofYears(int years) { return new LoanTerm(years * 12); }
    public LocalDate endDate(LocalDate startDate) { return startDate.plusMonths(months); }
}

// 3. double annualInterestRate → InterestRate
public record InterestRate(double annualRate) {
    public InterestRate {
        if (annualRate < 0 || annualRate > 50) {
            throw new InvalidInterestRateException("금리는 0~50% 사이여야 합니다: " + annualRate);
        }
    }
    public double monthlyRate() { return annualRate / 12 / 100; }
    // 월 납입금 계산 도메인 로직
    public Money calculateMonthlyPayment(Money principal, LoanTerm term) {
        double r = monthlyRate();
        int n = term.months();
        double factor = Math.pow(1 + r, n);
        BigDecimal monthly = principal.amount().multiply(
            BigDecimal.valueOf(r * factor / (factor - 1))
        );
        return new Money(monthly, principal.currency());
    }
}

// 4. city + street + zipCode → Address
public record Address(String city, String street, ZipCode zipCode) { ... }

// 개선된 메서드 시그니처
public void applyLoan(MemberId memberId, Money amount, LoanTerm term,
                      InterestRate rate, Address collateralAddress) {
    // 검증 로직이 Value Object 생성 시 자동으로 실행됨
    // 이 메서드는 오케스트레이션만 담당
}
```

</details>

---

**Q2.** Value Object에 `equals()`를 구현할 때 `BigDecimal`의 비교가 주의가 필요한 이유는?

<details>
<summary>해설 보기</summary>

**`BigDecimal.equals()`는 값뿐 아니라 scale(소수점 자릿수)도 비교합니다.**

```java
BigDecimal a = new BigDecimal("10.00");
BigDecimal b = new BigDecimal("10.0");
BigDecimal c = new BigDecimal("10");

a.equals(b)  // → false! (scale이 다름: 2 vs 1)
a.equals(c)  // → false! (scale이 다름: 2 vs 0)

a.compareTo(b)  // → 0 (값이 같음)
a.compareTo(c)  // → 0 (값이 같음)
```

**Money equals() 올바른 구현:**

```java
@Override
public boolean equals(Object o) {
    if (!(o instanceof Money)) return false;
    Money m = (Money) o;
    // equals()가 아닌 compareTo()로 값 비교
    return amount.compareTo(m.amount) == 0 && currency == m.currency;
}

@Override
public int hashCode() {
    // stripTrailingZeros()로 scale 제거 후 hashCode
    // "10.00", "10.0", "10" 모두 같은 hashCode
    return Objects.hash(amount.stripTrailingZeros(), currency);
}
```

**생성자에서 scale 정규화로 근본 해결:**
```java
public Money(BigDecimal amount, Currency currency) {
    // 생성 시 scale 통일 → equals에서 compareTo 불필요
    this.amount = amount.setScale(2, RoundingMode.HALF_UP);
    this.currency = currency;
}
// 이렇게 하면 equals()에서 amount.equals()를 써도 정상 동작
```

</details>

---

**Q3.** 불변 Value Object를 JPA로 저장할 때 `protected` 기본 생성자가 필요하다. 이것이 불변성을 깨는 것 아닌가? 어떻게 안전하게 처리하는가?

<details>
<summary>해설 보기</summary>

**`protected` 기본 생성자는 불변성을 실질적으로 깨지 않습니다.**

JPA는 Reflection을 통해 `protected` 생성자로 인스턴스를 생성한 후, 필드에 직접 값을 주입합니다. 이 과정은 JPA 프레임워크 내부에서만 발생하며, 애플리케이션 코드에서는 접근할 수 없습니다.

```java
@Embeddable
public final class Money {

    @Column(name = "amount")
    private final BigDecimal amount;

    @Enumerated(EnumType.STRING)
    @Column(name = "currency")
    private final Currency currency;

    // JPA 전용 — 애플리케이션 코드에서는 사용 불가
    protected Money() {
        this.amount = null;  // JPA가 Reflection으로 직접 필드에 주입
        this.currency = null;
    }

    // 애플리케이션에서 사용하는 생성자 (검증 포함)
    public Money(BigDecimal amount, Currency currency) {
        Objects.requireNonNull(amount);
        Objects.requireNonNull(currency);
        if (amount.compareTo(BigDecimal.ZERO) < 0) {
            throw new NegativeAmountException(amount.toString());
        }
        this.amount = amount;
        this.currency = currency;
    }
}
```

**실용적 대안 — Kotlin의 `data class`나 Lombok `@Value`:**
```java
// Lombok @Value는 자동으로 불변 클래스 생성
@Value  // = @Data + @FieldDefaults(makeFinal=true) + ...
@Embeddable
public class Money {
    BigDecimal amount;
    Currency currency;
}
```

핵심: `protected` 생성자는 기술적 타협이지만, 애플리케이션 코드에서 실수로 사용될 위험은 `private`보다 약간 높을 뿐입니다. 팀 컨벤션으로 "JPA VO에는 protected 생성자 허용"을 명시하면 됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: Entity vs Value Object](./01-entity-vs-value-object.md)** | **[홈으로 🏠](../README.md)** | **[다음: Aggregate 설계 ➡️](./03-aggregate-design.md)**

</div>
