# 테스트 전략 — 인프라 없이 도메인 불변식을 검증한다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Aggregate 불변식을 JPA 없이 순수 단위 테스트로 검증하는 패턴은?
- Application Service 통합 테스트에서 Repository를 인메모리 구현으로 대체하는 방법은?
- 도메인 이벤트 발행 여부를 검증하는 테스트를 어떻게 작성하는가?
- 테스트 피라미드에서 DDD 환경의 각 계층을 어떻게 배분해야 하는가?
- `@DataJpaTest`와 순수 단위 테스트를 어떻게 병행하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

"모든 테스트가 `@SpringBootTest`로 돌아가고 있어 테스트가 3분이나 걸립니다." 흔한 상황이다. DDD를 올바르게 구현하면 도메인의 핵심 로직 대부분을 Spring, JPA, DB 없이 순수 Java로 테스트할 수 있다. 이것이 가능한 이유는 도메인이 인프라에 의존하지 않기 때문이다.

이 문서는 DDD 환경에서의 테스트 피라미드를 구체적으로 설계하는 방법을 다룬다. 도메인 단위 테스트, Application Service 테스트(InMemory), JPA 통합 테스트, E2E 테스트를 각각 어떻게 작성하고 배분할지가 핵심이다.

---

## 😱 흔한 실수 (Before — 모든 테스트를 @SpringBootTest로)

```java
// 모든 테스트가 Spring Context 로드
@SpringBootTest
class OrderServiceTest {

    @Autowired
    private OrderService orderService;

    @Autowired
    private OrderRepository orderRepository;  // JPA

    @Autowired
    private CustomerRepository customerRepository;  // JPA

    @Test
    void placeOrder_withValidItems_createsOrder() {
        // DB setup
        Customer customer = customerRepository.save(new Customer("테스트 고객"));
        List<OrderItemRequest> items = List.of(new OrderItemRequest(1L, 2, 10000));

        // 실제 테스트
        OrderId orderId = orderService.placeOrder(new PlaceOrderRequest(customer.getId(), items));

        // DB에서 조회해서 검증
        Order order = orderRepository.findById(orderId).orElseThrow();
        assertThat(order.status()).isEqualTo(OrderStatus.PENDING);
        assertThat(order.lines()).hasSize(1);
    }
}
```

```
문제:
  Spring Context 로딩: ~5~30초
  DB 연결 필요 (H2 또는 실제 DB)
  데이터 격리 문제 (@Transactional 롤백 또는 cleanup 필요)
  
  테스트가 50개면 → 각 테스트 수 초 → 전체 수 분
  CI 파이프라인에서 테스트 병목이 됨
  
  도메인 불변식("주문 항목 최대 10개")을 테스트하는데
  왜 Spring Context가 필요한가?
```

---

## ✨ 올바른 접근 (After — 계층별 테스트 분리)

```java
// Layer 1: 도메인 단위 테스트 (JPA, Spring 없음)
class OrderTest {

    @Test
    void place_withEmptyItems_throwsException() {
        assertThatThrownBy(() -> Order.place(CUSTOMER_ID, List.of(), Discount.NONE, SHIPPING_FEE))
            .isInstanceOf(EmptyOrderException.class);
    }

    @Test
    void addLine_exceedsMaxLines_throwsException() {
        Order order = OrderBuilder.anOrder().build();
        // 최대 10개 항목까지 추가
        for (int i = 0; i < 10; i++) {
            order.addLine(new ProductId((long) i), 1, Money.ofKrw(1000));
        }
        // 11번째 추가 시 예외
        assertThatThrownBy(() -> order.addLine(new ProductId(99L), 1, Money.ofKrw(1000)))
            .isInstanceOf(OrderLimitExceededException.class);
    }

    @Test
    void cancel_afterShipping_throwsException() {
        Order order = OrderBuilder.anOrder().shipped();
        assertThatThrownBy(() -> order.cancel("취소"))
            .isInstanceOf(OrderNotCancellableException.class);
    }
}

// Layer 2: Application Service 테스트 (InMemory, 빠름)
class OrderApplicationServiceTest {

    private InMemoryOrderRepository orderRepo;
    private FakeEventPublisher eventPublisher;
    private OrderApplicationService sut;

    @BeforeEach
    void setUp() {
        orderRepo = new InMemoryOrderRepository();
        eventPublisher = new FakeEventPublisher();
        sut = new OrderApplicationService(orderRepo, eventPublisher, new OrderFactory(...));
    }

    @Test
    void placeOrder_publishesOrderPlacedEvent() {
        sut.placeOrder(validPlaceOrderCommand());

        assertThat(eventPublisher.eventsOf(OrderPlaced.class)).hasSize(1);
    }

    @Test
    void cancelOrder_afterPlacing_orderIsCancelledInRepository() {
        OrderId orderId = sut.placeOrder(validPlaceOrderCommand());
        sut.cancelOrder(new CancelOrderCommand(orderId, "고객 요청"));

        Order order = orderRepo.findById(orderId).orElseThrow();
        assertThat(order.status()).isEqualTo(OrderStatus.CANCELLED);
    }
}

// Layer 3: JPA 통합 테스트 (실제 DB 매핑 검증)
@DataJpaTest
class OrderJpaRepositoryTest {

    @Autowired
    private OrderJpaRepository orderRepository;

    @Test
    void save_andFindById_returnsPersistedOrder() {
        Order order = OrderBuilder.anOrder().build();
        orderRepository.save(order);

        Optional<Order> found = orderRepository.findById(order.id());
        assertThat(found).isPresent();
        assertThat(found.get().totalAmount()).isEqualTo(order.totalAmount());
    }
}
```

---

## 🔬 내부 동작 원리

### 1. DDD 환경의 테스트 피라미드

```
테스트 피라미드 (DDD 환경):

         /────────────────────────────────────\
        /          E2E / API 테스트             \    ← 10%
       /  (실제 서버, 실제 DB, 실제 외부 API)      \
      /─────────────────────────────────────────\
     /       통합 테스트 (@DataJpaTest)            \   ← 20%
    /  (JPA 매핑, 실제 DB, 쿼리 검증)               \
   /─────────────────────────────────────────────\
  /   Application Service 테스트 (InMemory)        \  ← 30%
 /  (유스케이스 흐름, 이벤트 발행, 포트 연결)          \
/─────────────────────────────────────────────────\
                도메인 단위 테스트                      ← 40%
      (Aggregate 불변식, VO 검증, 도메인 로직)
      (순수 Java, Spring 없음, 매우 빠름)

비율 목표:
  도메인 단위 테스트: 40% — 가장 많이, 가장 빠름 (< 1초)
  Application Service: 30% — 유스케이스 검증 (< 1초)
  JPA 통합: 20% — 영속성 매핑 검증 (수 초)
  E2E: 10% — 핵심 경로만 (수십 초)
```

### 2. 도메인 단위 테스트 패턴

```java
// 테스트 명명 규칙: [메서드]_[시나리오]_[기대결과]
class OrderTest {

    // 1. 정상 케이스
    @Test
    void place_withValidItems_createsOrderWithPendingStatus() { ... }

    // 2. 불변식 위반
    @Test
    void place_withEmptyItems_throwsEmptyOrderException() { ... }

    @Test
    void cancel_afterDelivery_throwsOrderNotCancellableException() { ... }

    // 3. 상태 전이
    @Test
    void pay_fromPending_changesStatusToPaid() { ... }

    // 4. 도메인 이벤트
    @Test
    void place_publishesOrderPlacedEvent() {
        Order order = Order.place(CUSTOMER_ID, LINES, Discount.NONE, SHIPPING_FEE);
        
        List<DomainEvent> events = order.pullDomainEvents();
        
        assertThat(events).hasSize(1);
        assertThat(events.get(0)).isInstanceOf(OrderPlaced.class);
        
        OrderPlaced event = (OrderPlaced) events.get(0);
        assertThat(event.orderId()).isEqualTo(order.id());
        assertThat(event.totalAmount()).isEqualTo(order.totalAmount());
    }

    // 5. Value Object 검증
    @Test
    void totalAmount_equalsToSumOfAllLineSubtotals() {
        List<OrderLine> lines = List.of(
            new OrderLine(PRODUCT_A, 2, Money.ofKrw(10_000)),  // 20,000
            new OrderLine(PRODUCT_B, 1, Money.ofKrw(5_000))    // 5,000
        );
        Order order = Order.place(CUSTOMER_ID, lines, Discount.NONE, Money.ofKrw(2_500));
        
        assertThat(order.totalAmount()).isEqualTo(Money.ofKrw(27_500));  // 25,000 + 2,500
    }
}
```

### 3. 이벤트 발행 검증

```java
// FakeEventPublisher: 발행된 이벤트 추적
public class FakeEventPublisher implements ApplicationEventPublisher {

    private final List<Object> publishedEvents = new ArrayList<>();

    @Override
    public void publishEvent(Object event) {
        publishedEvents.add(event);
    }

    // 특정 타입의 이벤트 필터링
    @SuppressWarnings("unchecked")
    public <T> List<T> eventsOf(Class<T> type) {
        return publishedEvents.stream()
            .filter(type::isInstance)
            .map(e -> (T) e)
            .collect(Collectors.toList());
    }

    public List<Object> allEvents() { return List.copyOf(publishedEvents); }
    public boolean hasEvents() { return !publishedEvents.isEmpty(); }
    public void clear() { publishedEvents.clear(); }
}

// 이벤트 발행 검증 테스트
@Test
void placeOrder_publishesOrderPlacedEventWithCorrectData() {
    PlaceOrderCommand command = PlaceOrderCommand.of(CUSTOMER_ID, LINES);

    sut.placeOrder(command);

    List<OrderPlaced> events = eventPublisher.eventsOf(OrderPlaced.class);
    assertThat(events).hasSize(1);

    OrderPlaced event = events.get(0);
    assertThat(event.customerId()).isEqualTo(CUSTOMER_ID);
    assertThat(event.totalAmount()).isGreaterThan(Money.ZERO_KRW);
    assertThat(event.placedAt()).isNotNull();
}

@Test
void cancelOrder_afterPlacing_publishesOrderCancelledEvent() {
    OrderId orderId = sut.placeOrder(PlaceOrderCommand.of(CUSTOMER_ID, LINES));
    eventPublisher.clear();  // 이전 이벤트 초기화

    sut.cancelOrder(new CancelOrderCommand(orderId, "고객 요청"));

    List<OrderCancelled> events = eventPublisher.eventsOf(OrderCancelled.class);
    assertThat(events).hasSize(1);
    assertThat(events.get(0).orderId()).isEqualTo(orderId);
    assertThat(events.get(0).reason()).isEqualTo("고객 요청");
}
```

### 4. @DataJpaTest — JPA 매핑 검증

```java
// JPA 매핑만 검증 — 도메인 로직은 이미 단위 테스트로 검증됨
@DataJpaTest
@Import(OrderMapper.class)  // 필요한 Bean만 추가
class JpaOrderRepositoryTest {

    @Autowired
    private TestEntityManager entityManager;

    @Autowired
    private SpringDataOrderRepository springDataRepo;

    private JpaOrderRepository sut;

    @BeforeEach
    void setUp() {
        sut = new JpaOrderRepository(springDataRepo);
    }

    @Test
    void save_persistsAllFields() {
        Order order = OrderBuilder.anOrder()
            .forCustomer(new CustomerId(100L))
            .withGoldMemberDiscount()
            .build();

        sut.save(order);
        entityManager.flush();
        entityManager.clear();  // 1차 캐시 비움

        Order loaded = sut.findById(order.id()).orElseThrow();
        assertThat(loaded.customerId()).isEqualTo(new CustomerId(100L));
        assertThat(loaded.status()).isEqualTo(OrderStatus.PENDING);
        assertThat(loaded.totalAmount()).isEqualTo(order.totalAmount());
        assertThat(loaded.lines()).hasSize(order.lines().size());
    }

    @Test
    void findById_withLines_avoidsNPlusOne() {
        // 주문 3개 저장
        for (int i = 0; i < 3; i++) {
            sut.save(OrderBuilder.anOrder().build());
        }
        entityManager.flush();
        entityManager.clear();

        // SQL 쿼리 수 검증 (JOIN FETCH로 1회만 실행되어야 함)
        // QueryCounterAssert 또는 Hibernate Statistics 활용
        List<Order> orders = sut.findByCustomerId(CUSTOMER_ID);
        orders.forEach(o -> o.lines().size());  // 각 주문의 lines 접근
        // JOIN FETCH가 없으면 N+1 발생
    }

    @Test
    void save_withValueObjects_persistsCorrectly() {
        Money totalAmount = Money.ofKrw(50_000);
        Order order = OrderBuilder.anOrder().withTotalAmount(totalAmount).build();

        sut.save(order);
        entityManager.flush();
        entityManager.clear();

        Order loaded = sut.findById(order.id()).orElseThrow();
        // Value Object가 올바르게 @Embedded 매핑됐는지 검증
        assertThat(loaded.totalAmount()).isEqualTo(totalAmount);
    }
}
```

---

## 💻 실전 코드

### 테스트 유틸 클래스 모음

```java
// OrderBuilder: 다양한 상태의 Order를 쉽게 생성
public class OrderBuilder {

    private CustomerId customerId = new CustomerId(1L);
    private List<OrderLine> lines = defaultLines();
    private Discount discount = Discount.NONE;
    private Money shippingFee = Money.ofKrw(2_500);

    public static OrderBuilder anOrder() { return new OrderBuilder(); }

    public OrderBuilder forCustomer(CustomerId customerId) {
        this.customerId = customerId; return this;
    }

    public OrderBuilder withGoldMemberDiscount() {
        this.discount = new Discount(new Percentage(10)); return this;
    }

    public OrderBuilder withFreeShipping() {
        this.shippingFee = Money.ZERO_KRW; return this;
    }

    public Order build() {
        return Order.place(customerId, lines, discount, shippingFee);
    }

    // 다양한 상태의 Order 생성
    public Order placed() { return build(); }

    public Order paid() {
        Order order = build();
        order.pay(new PaymentId(UUID.randomUUID()));
        return order;
    }

    public Order shipped() {
        Order order = paid();
        order.ship(new TrackingNumber("CJ1234567890KR"));
        return order;
    }

    public Order delivered() {
        Order order = shipped();
        order.confirmDelivery();
        return order;
    }

    private static List<OrderLine> defaultLines() {
        return List.of(
            new OrderLine(new ProductId(1L), 1, Money.ofKrw(10_000))
        );
    }
}

// MoneyAssert: Money 커스텀 검증
public class MoneyAssert extends AbstractAssert<MoneyAssert, Money> {

    public MoneyAssert(Money actual) { super(actual, MoneyAssert.class); }

    public static MoneyAssert assertThat(Money actual) {
        return new MoneyAssert(actual);
    }

    public MoneyAssert isEqualToKrw(long amount) {
        isNotNull();
        if (actual.amount().compareTo(BigDecimal.valueOf(amount)) != 0
            || actual.currency() != Currency.KRW) {
            failWithMessage("예상: %s KRW, 실제: %s %s", amount, actual.amount(), actual.currency());
        }
        return this;
    }

    public MoneyAssert isGreaterThan(Money other) {
        isNotNull();
        if (!actual.isGreaterThan(other)) {
            failWithMessage("예상: %s > %s", actual, other);
        }
        return this;
    }
}
```

---

## 📊 설계 비교

```
@SpringBootTest vs 계층별 테스트:

                @SpringBootTest 전체        계층별 분리
────────────┼───────────────────────────┼──────────────────────────
테스트 속도  │ 느림 (수 분)               │ 도메인: 매우 빠름 (< 1ms)
            │                           │ AppService: 빠름 (< 10ms)
            │                           │ JPA: 중간 (수 초)
────────────┼───────────────────────────┼──────────────────────────
피드백 속도  │ 늦음                       │ 빠름 (도메인 변경 즉시 확인)
────────────┼───────────────────────────┼──────────────────────────
실패 진단   │ 어려움 (여러 계층 동시)     │ 쉬움 (계층별 명확한 실패 위치)
────────────┼───────────────────────────┼──────────────────────────
테스트 설정 │ 복잡 (전체 Spring 설정)    │ 단순 (필요한 것만)
────────────┼───────────────────────────┼──────────────────────────
검증 범위   │ 넓음 (통합 검증)           │ 계층별 명확한 책임 검증
────────────┼───────────────────────────┼──────────────────────────
유지보수    │ 어려움 (설정 변경 영향 큼) │ 독립적 (계층별 독립)
```

---

## ⚖️ 트레이드오프

```
계층별 테스트의 비용:
  InMemoryRepository 구현 및 유지 필요
  FakeEventPublisher, FakePort 등 테스트 대역 관리
  Builder 패턴 유지 (도메인 변경 시 Builder도 수정)

현실적 균형:
  도메인 로직이 단순하면 → Application Service 테스트를 @DataJpaTest로 대체 가능
  도메인 로직이 복잡하면 → 순수 단위 테스트 가치 높음

테스트 속도 목표:
  도메인 테스트 전체: < 30초
  Application Service 테스트: < 1분
  JPA 통합 테스트: < 3분
  전체 테스트: < 5분 (CI 허용 범위)
```

---

## 📌 핵심 정리

```
DDD 테스트 전략 핵심:

테스트 피라미드:
  도메인 단위 (40%) → 불변식, VO, 도메인 로직
  Application Service (30%) → 유스케이스, 이벤트 발행
  JPA 통합 (20%) → 매핑, 쿼리
  E2E (10%) → 핵심 경로만

도메인 단위 테스트:
  Spring 없음, JPA 없음, 순수 Java
  Aggregate 불변식, 상태 전이, 이벤트 등록 검증
  Test Builder로 다양한 상태 생성

Application Service 테스트:
  InMemoryRepository + FakeEventPublisher
  유스케이스 흐름 검증
  이벤트 발행 여부 및 내용 검증

JPA 통합 테스트 (@DataJpaTest):
  Value Object 매핑 검증
  JOIN FETCH N+1 방지 검증
  낙관적 잠금, 제약 조건 검증

이벤트 검증:
  FakeEventPublisher.eventsOf(Type.class) 패턴
  이벤트 타입, 이벤트 데이터 모두 검증
```

---

## 🤔 생각해볼 문제

**Q1.** 도메인 단위 테스트에서 `Order.place()`가 `OrderPlaced` 이벤트를 발행하는지 검증했다. 그런데 Application Service 테스트에서도 같은 것을 검증해야 하는가?

<details>
<summary>해설 보기</summary>

**두 테스트는 다른 것을 검증합니다.**

```java
// 도메인 단위 테스트: "Order.place()가 이벤트를 등록하는가?"
@Test
void place_registersOrderPlacedEvent() {
    Order order = Order.place(CUSTOMER_ID, LINES, Discount.NONE, SHIPPING_FEE);
    
    assertThat(order.pullDomainEvents())
        .hasSize(1)
        .first().isInstanceOf(OrderPlaced.class);
    // 도메인이 책임지는 부분: 이벤트를 내부에 등록
}

// Application Service 테스트: "ApplicationService가 이벤트를 발행하는가?"
@Test
void placeOrder_publishesEventViaPublisher() {
    sut.placeOrder(command);
    
    assertThat(eventPublisher.eventsOf(OrderPlaced.class)).hasSize(1);
    // Application Service가 책임지는 부분: pullEvents() → publishEvent() 호출
}
```

둘 다 필요합니다:
- 도메인 테스트: Order가 이벤트를 올바르게 등록하는지
- Application Service 테스트: Service가 이벤트를 실제로 발행하는지 (pullEvents → publish 연결)

만약 Application Service에서 `order.pullDomainEvents().forEach(publisher::publishEvent)` 코드를 빠뜨리면 도메인 테스트는 통과하지만 Application Service 테스트에서 실패합니다. 두 테스트가 서로 다른 버그를 잡습니다.

</details>

---

**Q2.** `InMemoryOrderRepository`가 JPA의 `@Version` 낙관적 잠금을 재현하지 않는다. 이 차이를 테스트에서 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**낙관적 잠금은 JPA 통합 테스트에서만 검증하고, InMemory 테스트는 도메인 로직에 집중합니다.**

```java
// Application Service 테스트 (InMemory): 낙관적 잠금 테스트하지 않음
@Test
void placeOrder_savesOrder() {
    // 동시성 검증 없음 - InMemory는 단일 스레드
    sut.placeOrder(command);
    assertThat(orderRepo.size()).isEqualTo(1);
}

// JPA 통합 테스트: 낙관적 잠금 검증
@DataJpaTest
class OrderConcurrencyTest {

    @Test
    void concurrentModification_throwsOptimisticLockException() {
        Order order = orderRepository.save(OrderBuilder.anOrder().build());
        
        // 같은 Order를 두 번 로드 (각각 version = 0)
        Order order1 = orderRepository.findById(order.id()).orElseThrow();
        Order order2 = orderRepository.findById(order.id()).orElseThrow();
        
        order1.cancel("첫 번째 취소");
        orderRepository.save(order1);  // version = 1로 업데이트
        
        order2.cancel("두 번째 취소");
        assertThatThrownBy(() -> orderRepository.save(order2))
            .isInstanceOf(ObjectOptimisticLockingFailureException.class);
        // version 불일치 → 예외
    }
}
```

이것이 "테스트 계층별 책임 분리"의 핵심입니다. InMemory가 재현 못하는 것은 JPA 통합 테스트가 담당합니다.

</details>

---

**Q3.** Application Service 테스트에서 `InMemoryOrderRepository`를 사용하다가 JPA 환경과 다른 동작이 발생해 프로덕션에서만 버그가 나왔다. 어떻게 이 간격을 줄이는가?

<details>
<summary>해설 보기</summary>

**계약 테스트(Contract Test)로 InMemory와 JPA 구현체가 동일하게 동작함을 검증합니다.**

```java
// OrderRepository 계약 테스트 — 모든 구현체가 따라야 하는 행동 정의
abstract class OrderRepositoryContractTest {

    // 각 구현체 테스트에서 이 메서드를 오버라이드
    protected abstract OrderRepository createRepository();

    private OrderRepository sut;

    @BeforeEach
    void setUp() { sut = createRepository(); }

    @Test
    void save_andFindById_returnsEquivalentOrder() {
        Order original = OrderBuilder.anOrder().build();
        sut.save(original);
        
        Order found = sut.findById(original.id()).orElseThrow();
        assertThat(found.id()).isEqualTo(original.id());
        assertThat(found.status()).isEqualTo(original.status());
        assertThat(found.totalAmount()).isEqualTo(original.totalAmount());
    }

    @Test
    void findById_nonExistentId_returnsEmpty() {
        assertThat(sut.findById(new OrderId())).isEmpty();
    }

    @Test
    void save_thenModify_withoutSave_notReflected() {
        Order order = OrderBuilder.anOrder().build();
        sut.save(order);
        
        // save() 없이 수정
        order.cancel("테스트");  // 내부 상태만 변경
        
        // 저장소에서 다시 조회
        Order found = sut.findById(order.id()).orElseThrow();
        // InMemory: 참조 공유 여부에 따라 다름 → 깊은 복사 필수
        // JPA: 더티 체킹으로 업데이트될 수 있음 (트랜잭션 내부라면)
        // → 이 차이를 명시적으로 문서화
        assertThat(found.status()).isEqualTo(OrderStatus.PENDING);  // 수정 전 상태
    }
}

// InMemory 구현 계약 테스트
class InMemoryOrderRepositoryContractTest extends OrderRepositoryContractTest {
    @Override
    protected OrderRepository createRepository() {
        return new InMemoryOrderRepository();
    }
}

// JPA 구현 계약 테스트
@DataJpaTest
class JpaOrderRepositoryContractTest extends OrderRepositoryContractTest {
    @Autowired private SpringDataOrderRepository springDataRepo;

    @Override
    protected OrderRepository createRepository() {
        return new JpaOrderRepository(springDataRepo);
    }
}
```

이 패턴으로 두 구현체의 동작 차이를 조기에 발견할 수 있습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Domain Event Spring 통합](./06-domain-event-spring.md)** | **[홈으로 🏠](../README.md)** | **[Chapter 6로 이동: 실전 프로젝트 분석 ➡️](../real-world-project/01-ecommerce-domain-analysis.md)**

</div>
