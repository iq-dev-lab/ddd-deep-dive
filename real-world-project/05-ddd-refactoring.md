# DDD 리팩터링 — Transaction Script에서 점진적으로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Transaction Script 코드를 DDD로 점진적으로 이전하는 전략은?
- Strangler Fig 패턴으로 레거시를 안전하게 개선하는 방법은?
- 리팩터링 중 테스트가 안전망이 되는 이유와 테스트 작성 순서는?
- "빅뱅 리팩터링"이 왜 위험하고 어떻게 피하는가?
- 작은 리팩터링 단계를 어떻게 설계해야 팀이 무리 없이 따라올 수 있는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

새 프로젝트에서 DDD를 처음부터 적용하는 경우보다, 이미 돌아가고 있는 레거시 코드를 DDD로 개선해야 하는 경우가 훨씬 많다. "지금 당장 전부 뜯어고치자"는 빅뱅 리팩터링은 팀을 지치게 하고 버그를 양산한다. 점진적이고 안전한 리팩터링 전략이 현실에서 필요하다.

---

## 😱 흔한 실수 (Before — 전형적인 Transaction Script)

```java
// 전형적인 Transaction Script — 모든 로직이 Service에 집중
@Service
@Transactional
public class OrderService {

    @Autowired private OrderRepository orderRepository;
    @Autowired private ProductRepository productRepository;
    @Autowired private CustomerRepository customerRepository;
    @Autowired private InventoryRepository inventoryRepository;
    @Autowired private EmailService emailService;
    @Autowired private PointRepository pointRepository;

    public Long placeOrder(Long customerId, List<Long> productIds, List<Integer> quantities,
                           String couponCode, String shippingCity, String shippingStreet,
                           String shippingZip, boolean isExpress) {
        // 고객 조회
        Customer customer = customerRepository.findById(customerId)
            .orElseThrow(() -> new RuntimeException("고객 없음"));

        // 할인 계산 (Service에서 직접)
        BigDecimal discountRate = BigDecimal.ZERO;
        if ("GOLD".equals(customer.getMembershipLevel())) {
            discountRate = new BigDecimal("0.10");
        } else if ("SILVER".equals(customer.getMembershipLevel())) {
            discountRate = new BigDecimal("0.05");
        }

        // 상품 조회 및 재고 확인
        List<OrderItem> items = new ArrayList<>();
        BigDecimal subtotal = BigDecimal.ZERO;
        for (int i = 0; i < productIds.size(); i++) {
            Long productId = productIds.get(i);
            int quantity = quantities.get(i);

            Product product = productRepository.findById(productId)
                .orElseThrow(() -> new RuntimeException("상품 없음: " + productId));

            Inventory inventory = inventoryRepository.findByProductId(productId)
                .orElseThrow(() -> new RuntimeException("재고 없음"));

            if (inventory.getQuantity() < quantity) {
                throw new RuntimeException("재고 부족: " + productId);
            }

            inventory.setQuantity(inventory.getQuantity() - quantity);
            inventoryRepository.save(inventory);

            BigDecimal lineTotal = product.getPrice().multiply(new BigDecimal(quantity));
            subtotal = subtotal.add(lineTotal);
            items.add(new OrderItem(productId, product.getName(), product.getPrice(), quantity));
        }

        // 배송비 계산 (Service에서 직접)
        BigDecimal shippingFee = new BigDecimal("2500");
        if ("제주".equals(shippingCity)) {
            shippingFee = new BigDecimal("5000");
        }
        if (isExpress) {
            shippingFee = shippingFee.add(new BigDecimal("3000"));
        }

        // 쿠폰 처리 (Service에서 직접)
        // ... 쿠폰 조회 및 적용 ...

        // 총 금액
        BigDecimal discount = subtotal.multiply(discountRate);
        BigDecimal totalAmount = subtotal.subtract(discount).add(shippingFee);

        // 주문 저장 (anemic entity에 setter로)
        Order order = new Order();
        order.setCustomerId(customerId);
        order.setStatus("PENDING");
        order.setTotalAmount(totalAmount);
        order.setItems(items);
        order.setShippingCity(shippingCity);
        order.setShippingStreet(shippingStreet);
        order.setShippingZip(shippingZip);
        order.setCreatedAt(LocalDateTime.now());
        Long orderId = orderRepository.save(order).getId();

        // 이메일 발송
        emailService.sendOrderConfirmation(customer.getEmail(), orderId, totalAmount);

        // 포인트 적립
        PointRecord point = new PointRecord();
        point.setCustomerId(customerId);
        point.setAmount(totalAmount.multiply(new BigDecimal("0.01")));
        point.setType("EARN");
        point.setOrderId(orderId);
        pointRepository.save(point);

        return orderId;
    }
}
```

```
문제:
  하나의 메서드에 600줄
  모든 비즈니스 로직이 Service에
  Entity는 getter/setter 덩어리
  단위 테스트 불가 (모든 것이 통합 테스트)
  할인 계산 로직이 다른 Service에도 복사됨
```

---

## ✨ 올바른 접근 (After — 점진적 DDD 이전)

```
리팩터링 단계 (Strangler Fig):

Phase 1: 테스트 안전망 구축 (2주)
  기존 코드 그대로 유지
  통합 테스트로 현재 동작 기록
  → "이것이 올바른 동작이다" 기준점 수립

Phase 2: Value Object 추출 (2주)
  Money, Address, MembershipLevel, Discount 추출
  Service 코드 최소 변경
  기존 테스트 모두 통과 확인

Phase 3: 도메인 로직 Entity로 이전 (3주)
  할인 계산: PricingDomainService로 추출
  배송비 계산: ShippingDomainService로 추출
  Order.place() 팩토리 메서드 추가

Phase 4: 이벤트 기반으로 전환 (3주)
  이메일, 포인트 → 이벤트 핸들러로
  Outbox Pattern 도입

Phase 5: 정리 (2주)
  기존 setter 제거
  Transaction Script 코드 제거
  테스트를 단위 테스트로 교체
```

---

## 🔬 내부 동작 원리

### 1. Strangler Fig 패턴

```
Strangler Fig (교살 무화과나무) 패턴:
  열대우림의 무화과나무가 숙주 나무를 천천히 감싸며 성장
  숙주가 죽으면 무화과나무가 그 자리를 차지

소프트웨어 적용:
  레거시 코드를 즉시 삭제하지 않음
  새 코드를 레거시 옆에 점진적으로 추가
  새 코드가 레거시를 점점 대체
  레거시가 더 이상 호출되지 않으면 삭제

구현 방법:

단계 1: 새 코드를 레거시 옆에 추가
  // 레거시 (그대로 유지)
  public Long placeOrder(Long customerId, ...) { /* 기존 코드 */ }

  // 새 코드 (옆에 추가)
  public OrderId placeOrderV2(PlaceOrderCommand command) { /* 새 DDD 코드 */ }

단계 2: 새 API 엔드포인트에서 새 코드 호출
  @PostMapping("/v2/orders")
  public ResponseEntity<OrderId> placeOrderV2(@RequestBody PlaceOrderCommand command) {
      return ResponseEntity.ok(orderService.placeOrderV2(command));
  }

단계 3: 클라이언트를 점진적으로 새 API로 이전
  모바일 앱 새 버전: /v2/orders 사용
  백오피스: /v2/orders 사용
  레거시 /v1/orders 사용량 → 0%

단계 4: 레거시 삭제
  placeOrder(Long, ...) 메서드 제거
  /v1/orders 엔드포인트 제거
```

### 2. 테스트 안전망 구축 전략

```java
// Phase 1: 기존 동작을 통합 테스트로 기록 (리팩터링 전)
@SpringBootTest
@Transactional
class OrderServiceLegacyBehaviorTest {

    @Autowired
    private OrderService orderService;  // 레거시 Service

    @Test
    void placeOrder_goldMember_gets10PercentDiscount() {
        Long customerId = setupGoldMember();
        Long productId = setupProduct(new BigDecimal("10000"), 100);

        Long orderId = orderService.placeOrder(
            customerId, List.of(productId), List.of(2), null, "서울", "강남구", "12345", false
        );

        Order order = orderRepository.findById(orderId).orElseThrow();
        // 10% 할인 + 배송비 2,500원
        assertThat(order.getTotalAmount()).isEqualByComparingTo(new BigDecimal("20500"));
        // 20,000 × 0.9 + 2,500 = 20,500
    }

    @Test
    void placeOrder_jejuAddress_chargesHigherShipping() {
        // 제주 배송비 5,000원
    }

    @Test
    void placeOrder_insufficientInventory_throws() {
        // 재고 부족 시 예외
    }
}

// 이 테스트들이 리팩터링 후에도 모두 통과해야 함 → 안전망
```

### 3. Value Object 추출 단계 (Phase 2)

```java
// Before: Service에서 원시 타입 사용
if ("GOLD".equals(customer.getMembershipLevel())) {
    discountRate = new BigDecimal("0.10");
}

// After: Value Object 추출 (Service 로직은 최소 변경)
public enum MembershipLevel {
    BRONZE, SILVER, GOLD, VIP;

    public Discount defaultDiscount() {
        return switch (this) {
            case GOLD -> new Discount(new Percentage(10));
            case SILVER -> new Discount(new Percentage(5));
            case VIP -> new Discount(new Percentage(15));
            default -> Discount.NONE;
        };
    }
}

// Service는 이제 VO를 사용하지만 구조는 그대로
MembershipLevel level = MembershipLevel.valueOf(customer.getMembershipLevel());
Discount discount = level.defaultDiscount();
// 기존 테스트 모두 통과

// Value Object 단위 테스트 추가
class MembershipLevelTest {
    @Test
    void goldMember_gets10PercentDiscount() {
        assertThat(MembershipLevel.GOLD.defaultDiscount().rate())
            .isEqualTo(new Percentage(10));
    }
}
```

### 4. 도메인 로직 이전 단계 (Phase 3)

```java
// Before: Service에 흩어진 배송비 계산
BigDecimal shippingFee = new BigDecimal("2500");
if ("제주".equals(shippingCity)) shippingFee = new BigDecimal("5000");
if (isExpress) shippingFee = shippingFee.add(new BigDecimal("3000"));

// Step 1: Domain Service 추출 (Service는 위임만)
public class ShippingDomainService {
    public Money calculateShippingFee(Address address, boolean isExpress) {
        Money baseFee = address.isIsland() ? ISLAND_FEE : STANDARD_FEE;
        return isExpress ? baseFee.add(EXPRESS_SURCHARGE) : baseFee;
    }
}

// Service에서 위임 (기존 구조 최소 변경)
BigDecimal shippingFee = shippingService
    .calculateShippingFee(new Address(shippingCity, shippingStreet, shippingZip), isExpress)
    .amount();

// Step 2: ShippingDomainService 단위 테스트 추가 (빠른 피드백)
class ShippingDomainServiceTest {
    @Test
    void jejuAddress_chargesIslandFee() {
        Address jeju = new Address("제주시", "연동", "63100");
        Money fee = shippingService.calculateShippingFee(jeju, false);
        assertThat(fee).isEqualTo(Money.ofKrw(5000));
    }
}
```

---

## 💻 실전 코드

### 리팩터링 체크리스트

```java
// 리팩터링 진행 상황 추적
/*
Phase 1: 안전망 구축 ✅
  [✅] OrderService 통합 테스트 작성 (기존 동작 기록)
  [✅] 현재 커버리지: 75%

Phase 2: Value Object 추출 ✅
  [✅] Money VO 추출 + 단위 테스트
  [✅] Address VO 추출 + 단위 테스트
  [✅] MembershipLevel enum 추출 + 단위 테스트
  [✅] Discount VO 추출 + 단위 테스트
  [✅] 기존 통합 테스트 모두 통과 확인

Phase 3: 도메인 로직 이전 🔄 (진행 중)
  [✅] ShippingDomainService 추출
  [✅] PricingDomainService 추출
  [🔄] Order.place() 팩토리 메서드 추가 (진행 중)
  [⬜] OrderLine 불변식 구현
  [⬜] 상태 전이 캡슐화

Phase 4: 이벤트 기반 전환 ⬜ (예정)
  [⬜] OrderPlaced 이벤트 정의
  [⬜] 이메일 → EventHandler 이전
  [⬜] 포인트 → EventHandler 이전
  [⬜] Outbox Pattern 도입

Phase 5: 정리 ⬜ (예정)
  [⬜] setter 제거
  [⬜] 레거시 메서드 삭제
  [⬜] 통합 테스트 → 단위 테스트로 교체
*/
```

---

## 📊 설계 비교

```
빅뱅 리팩터링 vs 점진적 리팩터링:

                빅뱅 리팩터링           점진적 리팩터링
────────────┼──────────────────────┼──────────────────────────
기간         │ 수 개월 동결           │ 계속 출시하면서 진행
────────────┼──────────────────────┼──────────────────────────
위험도       │ 매우 높음              │ 낮음 (단계별 검증)
────────────┼──────────────────────┼──────────────────────────
팀 부담      │ 전체 집중 필요         │ 분산 (일부만 리팩터링)
────────────┼──────────────────────┼──────────────────────────
비즈니스 가치│ 리팩터링 기간 동안 없음│ 매 단계마다 가치 발생
────────────┼──────────────────────┼──────────────────────────
버그 위험    │ 한 번에 많은 변경      │ 단계별 검증으로 최소화
────────────┼──────────────────────┼──────────────────────────
롤백 가능성  │ 어려움 (전체 롤백)     │ 각 단계별 독립 롤백
```

---

## ⚖️ 트레이드오프

```
점진적 리팩터링의 현실적 어려움:
  ① "두 개의 방식" 공존 기간 복잡도
     레거시 코드 + 새 코드가 동시에 존재
     팀원이 어느 쪽을 써야 하는지 혼란
     → 컨벤션 문서 + 페어 프로그래밍

  ② 레거시와 새 코드 간 데이터 공유
     같은 DB 테이블을 레거시와 새 코드가 함께 사용
     → 스키마 변경 시 양쪽 호환성 유지

  ③ 팀의 리팩터링 피로
     "언제 끝나는가?" 불투명
     → 명확한 Phase와 완료 기준 정의

현실적 조언:
  "완벽한 DDD"를 목표로 하지 말 것
  "현재보다 나은" 상태를 단계별로 달성
  비즈니스 가치와 기술 부채 감소를 함께 설명
```

---

## 📌 핵심 정리

```
DDD 리팩터링 핵심:

Strangler Fig 패턴:
  레거시 옆에 새 코드 추가
  점진적으로 트래픽 이전
  레거시 사용량 0%되면 삭제

테스트 안전망 (먼저!):
  리팩터링 전 기존 동작을 통합 테스트로 기록
  리팩터링 후에도 이 테스트가 모두 통과해야

단계별 순서:
  1. 안전망 구축 (통합 테스트)
  2. Value Object 추출 (리스크 최소)
  3. 도메인 로직 → Domain Service/Entity 이전
  4. 이벤트 기반 전환
  5. 레거시 정리

각 단계 완료 기준:
  기존 통합 테스트 모두 통과
  새 단위 테스트 추가됨
  팀 리뷰 완료

피해야 할 것:
  빅뱅 리팩터링 (전체를 한 번에)
  테스트 없이 리팩터링
  비즈니스 기능과 동시에 대규모 구조 변경
```

---

## 🤔 생각해볼 문제

**Q1.** 리팩터링 중 기존 통합 테스트가 실패했다. 이것이 버그인가, 테스트가 잘못된 것인가? 어떻게 구분하는가?

<details>
<summary>해설 보기</summary>

**실패 원인을 체계적으로 분석합니다.**

```
구분 방법:

1. "동작이 변했는가?" vs "테스트가 오래됐는가?"

   동작이 변한 것(버그 가능성):
     주문 금액 계산 결과가 달라짐
     이메일이 발송되지 않음
     재고가 차감되지 않음
     → 비즈니스 로직 버그 → 즉시 수정 후 진행

   테스트가 오래된 것 (테스트 개선 필요):
     내부 구현 방식이 변해서 테스트가 깨짐
     (setter → 팩토리 메서드 변경으로 직접 접근 불가)
     하지만 실제 동작은 동일
     → 테스트를 새 방식에 맞게 업데이트

2. 프로덕션 로그 비교:
   리팩터링 전후로 같은 요청에 다른 응답이 오는가?
   → YES: 버그 → 수정 필요
   → NO:  테스트 문제 → 테스트 업데이트

3. 원칙: "동작 변경 없이 구조만 변경" 이 리팩터링의 정의
   비즈니스 동작이 변했다면 그것은 리팩터링이 아닌 기능 변경
```

</details>

---

**Q2.** 팀원들이 리팩터링에 참여하지 않고 레거시 방식으로 계속 코드를 추가한다. 어떻게 해결하는가?

<details>
<summary>해설 보기</summary>

**기술적 강제와 팀 문화 두 가지로 접근합니다.**

```
기술적 강제:
  ArchUnit으로 레거시 패턴 사용 금지 규칙 설정
  @ArchTest
  static ArchRule noSetterOnEntities = noMethods()
      .that().haveNameStartingWith("set")
      .should().beDeclaredInClassesThat()
      .areAnnotatedWith(Entity.class)
      .because("Entity는 setter 대신 도메인 메서드를 사용해야 합니다");
  
  PR 리뷰 체크리스트:
    □ 새 Service 메서드에 도메인 로직이 없는가?
    □ Entity에 setter를 추가하지 않았는가?
    □ 단위 테스트가 있는가?

팀 문화:
  리팩터링 목적과 이점 공유 (테스트 속도, 버그 감소 수치)
  페어 프로그래밍으로 새 방식 전파
  "리팩터링 챔피언" 역할 설정 (팀 내 DDD 전도사)
  작은 성공 사례를 자주 공유 ("이 리팩터링 후 버그가 50% 감소")

현실적: 완벽한 적용보다 점진적 개선에 집중
한 번에 전환 불가 → 새 코드부터 DDD 적용 → 기존 코드는 기회가 될 때
```

</details>

---

**Q3.** 레거시 코드에 테스트가 거의 없다. 통합 테스트 안전망을 구축할 때 어떤 시나리오부터 테스트해야 하는가?

<details>
<summary>해설 보기</summary>

**위험도가 높은 경로(Happy Path + 핵심 예외)부터 시작합니다.**

```
우선순위 결정 기준:

1순위: 비즈니스 핵심 경로 (Happy Path)
  "주문 성공" 전체 흐름
  "결제 성공" 전체 흐름
  이것이 실패하면 매출 손실 → 가장 먼저

2순위: 고빈도 오류 시나리오
  "재고 부족 시 주문 실패"
  "잘못된 결제 정보 시 실패"
  운영 로그에서 빈번한 예외 확인 → 우선순위

3순위: 비즈니스 규칙 경계값
  "10개 초과 주문 실패"
  "음수 수량 주문 실패"
  "취소 불가 상태에서 취소 실패"

4순위: 복잡한 계산
  "GOLD 등급 10% 할인 + 쿠폰 5% 추가 할인 조합"
  "제주 지역 + 특급 배송비 조합"
  계산 오류 = 재무 손실

시작: @SpringBootTest 통합 테스트 5~10개로 핵심 경로 커버
     → 이후 단위 테스트로 세밀하게 보완
```

</details>

---

<div align="center">

**[⬅️ 이전: 읽기 모델 분리](./04-read-model-separation.md)** | **[홈으로 🏠](../README.md)** | **[Chapter 7로 이동: Anemic Domain Model ➡️](../antipatterns/01-anemic-domain-model.md)**

</div>
