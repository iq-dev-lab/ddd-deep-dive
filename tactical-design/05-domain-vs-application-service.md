# Domain Service vs Application Service — 책임의 분리

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Domain Service와 Application Service의 핵심 차이는 무엇인가?
- 단일 Entity에 표현할 수 없는 도메인 로직은 왜 Domain Service로 가야 하는가?
- Application Service의 역할(오케스트레이션, 트랜잭션 경계, 포트 연결)은 무엇인가?
- 두 서비스를 혼동하면 어떻게 도메인 로직이 흩어지는가?
- `XXXService`라는 이름을 보면 Domain Service인지 Application Service인지 어떻게 구분하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"Service에 비즈니스 로직을 넣으면 안 된다"는 원칙을 알지만, 도메인 로직이 어느 Service에도 맞지 않을 때 어떻게 해야 하는가? 예를 들어 "두 고객 간 포인트 이전"은 `CustomerA.transferTo(CustomerB, points)`처럼 하나의 Entity에 표현하기 어렵다. 이런 경우가 Domain Service가 필요한 정확한 상황이다.

Application Service는 이와 전혀 다른 역할이다. 트랜잭션을 시작하고, Repository에서 Aggregate를 꺼내고, Domain 메서드를 호출하고, 이벤트를 발행하고, 외부 포트를 호출하는 **오케스트레이션**이 역할이다. 비즈니스 규칙을 직접 표현하지 않는다.

---

## 😱 흔한 실수 (Before — 두 역할이 섞인 Service)

```java
// 모든 것이 섞인 OrderService — 흔한 안티패턴
@Service
@Transactional
public class OrderService {

    // Application Service의 역할: 오케스트레이션
    // Domain Service의 역할: 도메인 로직 계산
    // 둘이 섞여 있음

    public OrderResponse placeOrder(PlaceOrderRequest request) {

        // 1. Aggregate 조회 (Application Service 역할)
        Customer customer = customerRepository.findById(request.getCustomerId()).orElseThrow();

        // 2. 도메인 로직: 할인율 계산 — 여러 Aggregate에 걸친 로직 (Domain Service가 해야 함)
        BigDecimal discountRate = BigDecimal.ZERO;
        if (customer.getMembershipLevel() == MembershipLevel.GOLD) {
            discountRate = new BigDecimal("0.10");
        } else if (customer.getMembershipLevel() == MembershipLevel.SILVER) {
            discountRate = new BigDecimal("0.05");
        }
        // 쿠폰 할인 추가
        if (request.getCouponCode() != null) {
            Coupon coupon = couponRepository.findByCode(request.getCouponCode()).orElseThrow();
            if (coupon.isApplicable(customer, request.getItems())) {
                discountRate = discountRate.add(coupon.getDiscountRate());
            }
        }

        // 3. 더 많은 도메인 로직: 배송비 계산
        BigDecimal shippingFee = BigDecimal.ZERO;
        if (request.getAddress().getRegion().equals("제주")) {
            shippingFee = new BigDecimal("3000");
        } else if (request.getTotalAmount().compareTo(new BigDecimal("50000")) < 0) {
            shippingFee = new BigDecimal("2500");
        }

        // 4. Order 생성 + 저장 (Application Service 역할)
        Order order = Order.place(customer, request.getItems(), discountRate, shippingFee);
        orderRepository.save(order);

        return OrderResponse.from(order);
    }
}
```

```
문제:
  할인율 계산 로직이 OrderService에 있음
    → PromotionService에도 동일한 할인 계산이 복사됨
    → CartService에도 미리보기용으로 복사됨
    → 3곳에서 관리 → 하나 변경 시 3곳 모두 수정해야

  배송비 계산 로직이 OrderService에 있음
    → ShippingEstimateService에도 복사됨
    → 복잡해질수록 중복 증가

  "제주 지역 배송비 변경" 요청:
    OrderService, ShippingEstimateService, ... 모두 수정
    → 하나 빠뜨리면 버그
```

---

## ✨ 올바른 접근 (After — 역할 분리)

```java
// Domain Service: 여러 Aggregate에 걸친 도메인 로직
// 인프라 의존 없음 (DB, 외부 API 없음)
public class PricingDomainService {

    // 도메인 로직: 고객 등급 + 쿠폰 → 최종 할인 계산
    // 여러 Aggregate(Customer, Coupon)에 걸친 도메인 규칙
    public Discount calculateDiscount(Customer customer, Optional<Coupon> coupon, Money baseAmount) {
        Discount memberDiscount = customer.membershipLevel().discount();  // Customer가 직접 답

        Discount couponDiscount = coupon
            .filter(c -> c.isApplicable(customer, baseAmount))
            .map(Coupon::discount)
            .orElse(Discount.NONE);

        return memberDiscount.combine(couponDiscount);  // 할인 합산 로직
    }
}

public class ShippingDomainService {

    // 도메인 로직: 지역 + 금액 → 배송비 계산
    public Money calculateShippingFee(ShippingAddress address, Money orderAmount) {
        if (address.isIsland()) return ShippingPolicy.ISLAND_FEE;
        if (orderAmount.isGreaterThan(ShippingPolicy.FREE_SHIPPING_THRESHOLD)) return Money.ZERO_KRW;
        return ShippingPolicy.DEFAULT_FEE;
    }
}

// Application Service: 오케스트레이션만
// 트랜잭션 경계, Repository 조회, Domain 메서드 호출, 이벤트 발행
@Service
@Transactional
public class OrderApplicationService {

    private final CustomerRepository customerRepository;
    private final OrderRepository orderRepository;
    private final CouponRepository couponRepository;
    private final PricingDomainService pricingService;
    private final ShippingDomainService shippingService;
    private final ApplicationEventPublisher eventPublisher;

    public OrderId placeOrder(PlaceOrderCommand command) {
        // 1. Aggregate 조회
        Customer customer = customerRepository.findById(command.customerId()).orElseThrow();
        Optional<Coupon> coupon = command.couponCode()
            .flatMap(couponRepository::findByCode);

        // 2. 도메인 로직 위임 (Domain Service)
        Money baseAmount = command.calculateBaseAmount();
        Discount discount = pricingService.calculateDiscount(customer, coupon, baseAmount);
        Money shippingFee = shippingService.calculateShippingFee(command.address(), baseAmount);

        // 3. Aggregate 생성 (도메인 로직은 Order.place()에)
        Order order = Order.place(command.customerId(), command.lines(), discount, shippingFee);

        // 4. 저장 및 이벤트 발행
        orderRepository.save(order);
        order.pullEvents().forEach(eventPublisher::publishEvent);

        return order.id();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 세 가지 위치 비교

```
비즈니스 로직의 위치별 분류:

A. Entity / Aggregate 메서드:
   단일 Aggregate의 데이터만으로 결정되는 로직
   
   order.cancel()            → Order의 상태만 필요
   customer.earnPoints()     → Customer의 데이터만 필요
   coupon.isExpired()        → Coupon의 유효기간만 필요

B. Domain Service:
   여러 Aggregate에 걸친 도메인 로직
   또는 하나의 Entity에 자연스럽게 속하기 어려운 로직
   
   pricingService.calculateDiscount(customer, coupon)
     → Customer와 Coupon 두 Aggregate가 필요
   transferService.transfer(accountA, accountB, amount)
     → 두 Account Aggregate에 걸친 로직

C. Application Service:
   도메인 로직이 아닌 유스케이스 흐름 제어
   
   Repository 조회, 트랜잭션 경계, 이벤트 발행,
   DTO 변환, 외부 포트 호출
   "무엇을 어떤 순서로 실행하는가" 를 결정

판단 질문:
  "이 코드가 비즈니스 규칙을 표현하는가?"
  → YES + 하나의 Aggregate → Entity 메서드
  → YES + 여러 Aggregate → Domain Service
  → NO (오케스트레이션) → Application Service
```

### 2. Domain Service의 특징

```
Domain Service 특징:

① 인프라 의존 없음
   Domain Service는 DB, 외부 API, 메시지 큐를 모름
   → 순수 Java 코드로 구현 가능
   → 단위 테스트 시 Mock 없이 테스트 가능

② Stateless (상태 없음)
   Domain Service는 상태를 갖지 않음
   → Aggregate에서 상태를 받아 계산만 수행

③ Ubiquitous Language로 명명
   이름이 도메인 언어에서 파생됨
   PricingService ✅ (도메인 개념 "가격 책정")
   DataProcessor ❌ (기술 용어)

④ 드물게 사용해야 함
   대부분의 로직은 Entity/VO에 위치
   Domain Service가 너무 많으면 → Anemic Domain Model 신호
   "이 로직이 정말 하나의 Entity에 속하지 않는가?" 재검토

Domain Service가 적합한 경우:
  - 두 Aggregate 간 상호작용 (이체, 포인트 이전)
  - 특정 Aggregate에 속하기 어색한 도메인 계산 (배송비 정책)
  - 외부 도메인 인터페이스를 통한 도메인 규칙 (환율 기반 금액 변환)
```

### 3. Application Service의 책임 목록

```
Application Service = 유스케이스의 오케스트레이터

책임 1: 트랜잭션 경계
  @Transactional — 유스케이스 하나 = 트랜잭션 하나
  Domain 계층은 트랜잭션을 모름

책임 2: Aggregate 조회
  Repository를 통해 필요한 Aggregate 조회
  Domain 계층은 Repository를 직접 사용하지 않음

책임 3: 도메인 로직 위임
  직접 계산하지 않고 Entity/Domain Service에 위임
  "어떻게"는 Domain이 알고 Application은 "누구에게 물을지"만 결정

책임 4: 이벤트 발행
  Domain Event를 수집하여 발행 (Kafka, Spring Event 등)
  Domain 계층은 이벤트 발행 인프라를 모름

책임 5: 외부 포트 호출
  ACL을 통한 외부 시스템 호출
  이메일 발송, SMS, 알림 — Application Service에서 Port 인터페이스 호출

책임 6: DTO 변환 (입/출력 경계)
  Command → Aggregate 생성 파라미터
  Aggregate → Response DTO

포함하면 안 되는 것:
  ❌ 할인율 계산 (도메인 로직)
  ❌ 유효성 도메인 규칙 ("이 주문은 취소 가능한가?")
  ❌ 비즈니스 분기 ("GOLD 고객이면 10% 할인")
  → 이런 것이 있으면 Domain Service 또는 Entity로 이동
```

### 4. 두 역할 혼동 시 발생하는 문제

```
Application Service에 도메인 로직이 있으면:

문제 1: 중복 발생
  OrderApplicationService.calculateDiscount()
  CartApplicationService.calculateDiscount()   ← 복사
  PromotionApplicationService.calculateDiscount() ← 또 복사

문제 2: 테스트 어려움
  할인 계산을 테스트하려면 Repository Mock이 필요
  (Application Service는 인프라에 의존하므로)
  → Domain Service에 있으면: 순수 Java 단위 테스트

문제 3: 재사용 불가
  OrderApplicationService의 할인 로직을
  다른 Application Service에서 재사용하려면 Service 간 의존
  → Domain Service로 추출하면 자유롭게 재사용

Domain Service에 오케스트레이션이 있으면:

문제: Domain Service가 Repository, 이벤트 발행 등 인프라에 의존
  → Domain 계층이 인프라를 알게 됨 (의존성 역전 위반)
  → Domain Service 테스트에 Mock 필요
  → Domain Service를 Infrastructure가 아닌 Application이 소유하는 이유 없어짐
```

---

## 💻 실전 코드

### Domain Service 단위 테스트 — Mock 없음

```java
class PricingDomainServiceTest {

    // 외부 의존 없이 순수 객체만으로 테스트
    private final PricingDomainService pricingService = new PricingDomainService();

    @Test
    void goldMemberGets10PercentDiscount() {
        Customer goldMember = CustomerBuilder.withLevel(MembershipLevel.GOLD).build();
        Money baseAmount = Money.ofKrw(100_000);

        Discount discount = pricingService.calculateDiscount(goldMember, Optional.empty(), baseAmount);

        assertThat(discount.rate()).isEqualTo(new Percentage(10));
        assertThat(baseAmount.discountBy(discount.rate())).isEqualTo(Money.ofKrw(90_000));
    }

    @Test
    void membershipAndCouponDiscountsCombine() {
        Customer silverMember = CustomerBuilder.withLevel(MembershipLevel.SILVER).build();
        Coupon fivePercentCoupon = CouponBuilder.withRate(5).build();
        Money baseAmount = Money.ofKrw(100_000);

        Discount discount = pricingService.calculateDiscount(
            silverMember, Optional.of(fivePercentCoupon), baseAmount
        );

        // Silver 5% + 쿠폰 5% = 10%
        assertThat(discount.rate()).isEqualTo(new Percentage(10));
    }
}
```

### Command 객체로 Application Service 입력 표현

```java
// Command: Application Service의 입력 (Immutable)
public record PlaceOrderCommand(
    CustomerId customerId,
    List<OrderLineRequest> lines,
    Optional<CouponCode> couponCode,
    ShippingAddress address
) {
    // 기본 검증 (형식 수준)
    public PlaceOrderCommand {
        Objects.requireNonNull(customerId);
        Objects.requireNonNull(lines);
        if (lines.isEmpty()) throw new IllegalArgumentException("주문 항목이 없습니다");
    }

    public Money calculateBaseAmount() {
        return lines.stream()
            .map(l -> l.unitPrice().multiply(l.quantity()))
            .reduce(Money.ZERO_KRW, Money::add);
    }
}
```

---

## 📊 설계 비교

```
Entity / Domain Service / Application Service 책임 비교:

                Entity/VO           Domain Service      Application Service
────────────┼───────────────────┼──────────────────┼──────────────────────
역할         │ 도메인 규칙 표현   │ 여러 Aggregate에   │ 유스케이스 오케스트레이션
            │ (단일 객체 범위)   │ 걸친 도메인 로직   │
────────────┼───────────────────┼──────────────────┼──────────────────────
상태         │ 유 (Aggregate)    │ 무 (Stateless)    │ 무 (Stateless)
────────────┼───────────────────┼──────────────────┼──────────────────────
인프라 의존  │ 없음              │ 없음              │ 있음 (Repository, Port)
────────────┼───────────────────┼──────────────────┼──────────────────────
트랜잭션    │ 모름              │ 모름              │ 정의함 (@Transactional)
────────────┼───────────────────┼──────────────────┼──────────────────────
테스트      │ 단위 테스트 (Mock 0)│ 단위 테스트 (Mock 0)│ 통합 테스트 (Mock 필요)
────────────┼───────────────────┼──────────────────┼──────────────────────
예시        │ order.cancel()    │ 배송비 계산 정책   │ placeOrder()
            │ coupon.apply()    │ 포인트 이전 로직   │ cancelOrder()
```

---

## ⚖️ 트레이드오프

```
Domain Service 남용 위험:
  너무 많은 Domain Service → Anemic Domain Model로 회귀
  "이 Entity는 도메인 로직이 없고 Domain Service에 다 있다"
  → Entity가 Anemic해진 것 → 로직을 다시 Entity로

  원칙: Domain Service는 드물게 사용
  대부분의 도메인 로직은 Entity/VO에 위치해야 함

Application Service 크기 조절:
  Application Service가 너무 크면 → 유스케이스가 너무 많은 것
  → 여러 Application Service로 분리 (OrderPlacementService, OrderCancellationService 등)
  
  Application Service가 도메인 로직을 포함하면 → Domain Service 추출

명명 규칙으로 구분:
  Domain Service: 도메인 명사 중심 (PricingPolicy, DiscountCalculator, TransferService)
  Application Service: 유스케이스 중심 (OrderApplicationService, PaymentApplicationService)
```

---

## 📌 핵심 정리

```
Domain Service vs Application Service:

Domain Service:
  여러 Aggregate에 걸친 도메인 로직
  인프라 없음, 상태 없음, Stateless
  단위 테스트 가능 (Mock 0)
  드물게 사용 (대부분 Entity에)

Application Service:
  유스케이스 오케스트레이션 (흐름 제어)
  트랜잭션 경계 (@Transactional)
  Repository 조회, 이벤트 발행, 외부 포트 호출
  직접 비즈니스 로직 없음 (Entity/Domain Service에 위임)

판단 질문:
  "이 코드가 비즈니스 규칙인가, 흐름 제어인가?"
  → 비즈니스 규칙 + 단일 Aggregate → Entity
  → 비즈니스 규칙 + 여러 Aggregate → Domain Service
  → 흐름 제어 → Application Service

경고 신호:
  Application Service에 if/else 할인 계산 → Domain Service로
  Domain Service에 @Autowired Repository → Application Service로 이동
  XXXService가 너무 크고 복잡 → 역할 재검토
```

---

## 🤔 생각해볼 문제

**Q1.** "이체(Transfer)" 기능에서 `accountA.transfer(accountB, amount)`로 하나의 Entity에 구현하는 것이 왜 어색한가? Domain Service로 구현하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**`accountA.transfer(accountB, amount)`의 문제:**

1. Account A가 Account B라는 다른 Aggregate를 직접 참조하게 됨 → Aggregate 경계 침범
2. "차감"은 A의 책임이지만 "적립"은 B의 책임 → 두 Aggregate가 각자 책임져야 함
3. Account A가 Account B의 존재를 알아야 함 → 강한 결합

**Domain Service로 구현:**

```java
// TransferDomainService: 두 Account Aggregate에 걸친 이체 로직
public class TransferDomainService {

    public Transfer transfer(Account source, Account target, Money amount) {
        // 도메인 규칙: 잔액 검증
        if (!source.hasSufficientBalance(amount)) {
            throw new InsufficientBalanceException(source.id(), amount);
        }

        // 각 Aggregate는 자신의 상태만 변경
        source.withdraw(amount);           // Account A가 차감
        target.deposit(amount);            // Account B가 적립

        // 이체 기록 생성
        return Transfer.record(source.id(), target.id(), amount);
    }
}

// Application Service: 오케스트레이션
@Transactional
public void transfer(TransferCommand command) {
    Account source = accountRepository.findById(command.sourceId()).orElseThrow();
    Account target = accountRepository.findById(command.targetId()).orElseThrow();

    Transfer transfer = transferDomainService.transfer(source, target, command.amount());

    accountRepository.save(source);
    accountRepository.save(target);
    transferRepository.save(transfer);
}
```

단, 두 Account를 같은 트랜잭션에서 수정하는 것이 맞는지는 도메인 전문가와 확인 필요. 실제 금융 시스템에서는 Eventually Consistent + 보상 트랜잭션이 일반적입니다.

</details>

---

**Q2.** Application Service가 다른 Application Service를 호출하는 것이 허용되는가?

<details>
<summary>해설 보기</summary>

**원칙적으로 피해야 합니다. 발생한다면 설계를 재검토해야 합니다.**

**문제점:**
- 트랜잭션 중첩 → 예측하기 어려운 동작
- 유스케이스 간 강한 결합 → 하나 변경 시 다른 것도 영향

**대안 1: 공통 로직을 Domain Service로 추출**
```java
// 두 Application Service가 같은 Domain Service를 사용
OrderApplicationService.placeOrder() → PricingDomainService
CartApplicationService.calculateTotal() → PricingDomainService
```

**대안 2: 이벤트로 연결**
```java
// A 유스케이스 완료 → 이벤트 발행 → B 유스케이스 실행
OrderPlaced 이벤트 → PointApplicationService.earnPoints()
// 트랜잭션 분리, 느슨한 결합
```

**허용되는 경우 (매우 드물게):**
서로 다른 컨텍스트의 Application Service를 직접 호출하는 것보다는 ACL을 통한 인터페이스 호출. 같은 컨텍스트 내에서는 공통 로직을 Domain Service로 추출하는 것이 더 나은 설계입니다.

</details>

---

**Q3.** "배송비 계산 정책"을 Domain Service에 구현했는데, 배송사별 요금표를 DB에서 읽어와야 한다. 이 경우 Domain Service가 Repository를 의존해야 하는가?

<details>
<summary>해설 보기</summary>

**아닙니다. Port 인터페이스를 통해 의존성을 역전합니다.**

```java
// Domain Layer: Port 인터페이스 (DB를 모름)
public interface ShippingRatePort {
    ShippingRate findRateFor(CarrierType carrier, Region region);
}

// Domain Service: Port를 통해 요금 조회
public class ShippingDomainService {

    private final ShippingRatePort shippingRatePort;  // 인터페이스만 의존

    public Money calculateShippingFee(ShippingAddress address, CarrierType carrier) {
        ShippingRate rate = shippingRatePort.findRateFor(carrier, address.region());
        return rate.applyTo(address);
    }
}

// Infrastructure Layer: Port 구현 (DB 접근)
@Component
public class JpaShippingRateAdapter implements ShippingRatePort {
    private final ShippingRateJpaRepository repository;

    @Override
    public ShippingRate findRateFor(CarrierType carrier, Region region) {
        return repository.findByCarrierAndRegion(carrier.name(), region.code())
            .map(ShippingRateMapper::toDomain)
            .orElse(ShippingRate.DEFAULT);
    }
}
```

이렇게 하면 Domain Service는 인프라를 모르고, Port 인터페이스만 알게 됩니다. 테스트 시 `FakeShippingRatePort`로 대체 가능하여 DB 없이 테스트할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Aggregate 크기 결정](./04-aggregate-size.md)** | **[홈으로 🏠](../README.md)** | **[다음: Repository 설계 원칙 ➡️](./06-repository-design.md)**

</div>
