# DDD와 레이어드 아키텍처 비교 — Controller → Service → Repository의 문제점

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 전통적 레이어드 아키텍처에서 도메인 로직이 왜 Service로 흘러가는가?
- DDD의 레이어 구조(Domain Layer 분리)는 기존 3-tier와 어떻게 다른가?
- 도메인 레이어가 분리됐을 때 테스트가 쉬워지는 이유는?
- Hexagonal Architecture(Ports and Adapters)가 DDD와 어떻게 연결되는가?
- 기존 레이어드 구조에서 DDD 레이어 구조로 마이그레이션하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Spring Boot로 프로젝트를 시작하면 자연스럽게 Controller → Service → Repository 구조가 만들어진다. 이 구조가 익숙하고, 많은 튜토리얼이 이 방식으로 가르친다. 그런데 시스템이 커질수록 `OrderService`가 `PaymentService`, `InventoryService`, `NotificationService`를 주입받고, Service가 Service를 호출하는 복잡한 의존 그래프가 생긴다. 도메인 로직은 Service에, 데이터는 Entity에, 그 사이 어딘가에 비즈니스의 진실이 흩어진다.

DDD의 레이어 분리는 이 문제를 해결한다. **Domain Layer를 명확히 분리**하고, 이 레이어가 인프라(JPA, 외부 API, 메시지 큐)에 전혀 의존하지 않도록 만드는 것이 핵심이다. 이것이 되면 비즈니스 로직을 DB 없이, Spring Context 없이 테스트할 수 있다.

---

## 😱 흔한 실수 (Before — 전통 레이어드 아키텍처의 문제)

```
전통 3-tier 레이어드 아키텍처:

  ┌──────────────────────────────────────┐
  │         Presentation Layer           │
  │   (Controller, DTO, Request/Response) │
  └──────────────────┬───────────────────┘
                     │ 호출
  ┌──────────────────▼───────────────────┐
  │           Business Layer             │
  │   (Service — 비즈니스 로직이 여기)      │
  └──────────────────┬───────────────────┘
                     │ 호출
  ┌──────────────────▼───────────────────┐
  │            Data Layer                │
  │   (Repository, Entity — 데이터만)     │
  └──────────────────────────────────────┘

문제: "Business Layer" = Service 가 비대해짐
```

```java
// 전형적인 레이어드 아키텍처의 Service
@Service
@Transactional
public class OrderService {

    // 의존성이 계속 늘어남
    private final OrderRepository orderRepository;
    private final ProductRepository productRepository;
    private final MemberRepository memberRepository;
    private final CouponRepository couponRepository;
    private final PaymentService paymentService;          // Service → Service 의존
    private final InventoryService inventoryService;
    private final NotificationService notificationService;
    private final ShippingService shippingService;

    public OrderResponse placeOrder(PlaceOrderRequest request) {
        // 1. 회원 조회 및 검증
        Member member = memberRepository.findById(request.getMemberId())
            .orElseThrow(() -> new MemberNotFoundException(request.getMemberId()));

        // 2. 상품 조회 및 재고 확인 (inventoryService 사용)
        List<Product> products = request.getItems().stream()
            .map(item -> productRepository.findById(item.getProductId())
                .orElseThrow(() -> new ProductNotFoundException(item.getProductId())))
            .collect(toList());

        // 3. 재고 검증 (inventoryService 호출)
        request.getItems().forEach(item ->
            inventoryService.checkStock(item.getProductId(), item.getQuantity())
        );

        // 4. 쿠폰 적용 (쿠폰 로직이 Service에)
        BigDecimal totalAmount = calculateTotal(request.getItems(), products);
        if (request.getCouponId() != null) {
            Coupon coupon = couponRepository.findById(request.getCouponId()).orElseThrow();
            if (coupon.isExpired()) throw new CouponExpiredException();
            if (totalAmount.compareTo(coupon.getMinimumAmount()) < 0)
                throw new CouponNotApplicableException();
            totalAmount = totalAmount.subtract(coupon.getDiscountAmount());
        }

        // 5. 주문 생성
        Order order = new Order();
        order.setMemberId(member.getId());
        order.setStatus("PENDING");
        order.setTotalAmount(totalAmount);
        // ... 수십 줄의 setter 호출

        Order savedOrder = orderRepository.save(order);

        // 6. 결제 처리 (paymentService 호출)
        paymentService.processPayment(savedOrder.getId(), request.getPaymentInfo());

        // 7. 재고 차감 (inventoryService 호출)
        inventoryService.decreaseStock(request.getItems());

        // 8. 알림 발송
        notificationService.sendOrderConfirmation(member.getEmail(), savedOrder);

        // 9. 배송 요청
        shippingService.requestShipment(savedOrder.getId(), request.getShippingAddress());

        return orderMapper.toResponse(savedOrder);
    }

    // placeOrder() 만 300줄 — 이 안에서 뭘 수정하면 뭐가 영향받는지 모름
}
```

```
구체적 문제:

① 단위 테스트가 불가능에 가까움
   OrderService 테스트 작성 시:
   → orderRepository.mock(), productRepository.mock(),
     memberRepository.mock(), couponRepository.mock(),
     paymentService.mock(), inventoryService.mock(),
     notificationService.mock(), shippingService.mock()
   → 8개의 Mock이 필요
   → Mock 설정 코드가 테스트 코드보다 길어짐
   → "이 테스트가 실제로 비즈니스 규칙을 검증하는가?" 불명확

② 비즈니스 규칙이 어디에 있는지 모름
   쿠폰 적용 규칙 → OrderService (여기서)
   쿠폰 유효성 → CouponService (다른 곳에도)
   재고 검증 → InventoryService (또 다른 곳)

③ 변경 영향 범위 예측 불가
   "쿠폰 최소 주문 금액 계산 방식 변경"
   → OrderService 수정 + CouponService 수정
   → 두 곳 다 찾아야 함
   → 하나 빠뜨리면 버그
```

---

## ✨ 올바른 접근 (After — DDD 레이어 구조)

```
DDD 레이어 아키텍처:

  ┌──────────────────────────────────────────────┐
  │            Presentation Layer                │
  │   Controller, DTO, Request/Response 변환      │
  │   → Application Service 호출                 │
  └──────────────────┬───────────────────────────┘
                     │
  ┌──────────────────▼───────────────────────────┐
  │           Application Layer                  │
  │   Application Service                        │
  │   → 오케스트레이션 (순서 조율)                  │
  │   → 트랜잭션 경계                             │
  │   → 인프라 서비스 호출 (이벤트 발행, 알림)      │
  │   → 도메인 로직은 여기에 없음                  │
  └──────────────────┬───────────────────────────┘
                     │ 사용
  ┌──────────────────▼───────────────────────────┐
  │             Domain Layer  ★ 핵심              │
  │   Aggregate (Order, Payment, Shipment)        │
  │   Entity, Value Object                       │
  │   Domain Service (도메인 로직)                │
  │   Repository Interface (인터페이스만)          │
  │   Domain Event                               │
  │                                              │
  │   → 인프라에 의존하지 않음                     │
  │   → JPA, Kafka, Redis를 모름                  │
  │   → 순수 Java (또는 언어 자체)                 │
  └──────────────────┬───────────────────────────┘
                     │ 구현
  ┌──────────────────▼───────────────────────────┐
  │           Infrastructure Layer               │
  │   JpaOrderRepository (Repository 구현체)      │
  │   KafkaEventPublisher (이벤트 발행)           │
  │   ExternalPaymentAdapter (외부 API 연동)      │
  │   → Domain Layer의 인터페이스를 구현            │
  └──────────────────────────────────────────────┘

핵심: Domain Layer → 다른 Layer를 의존하지 않음
      다른 Layer들 → Domain Layer를 의존함
```

```java
// Domain Layer — 순수 도메인 로직, 인프라 의존 없음
public class Order {  // JPA 어노테이션 없음

    public static Order place(CustomerId customerId,
                              List<OrderLineRequest> lines,
                              Optional<Coupon> coupon) {
        if (lines.isEmpty()) throw new EmptyOrderException();

        List<OrderLine> orderLines = lines.stream()
            .map(r -> new OrderLine(r.productId(), r.quantity(), r.price()))
            .collect(toList());

        Money originalTotal = orderLines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);

        Money finalTotal = coupon
            .map(c -> c.apply(originalTotal))  // Coupon이 스스로 할인 계산
            .orElse(originalTotal);

        Order order = new Order(new OrderId(), customerId, orderLines, finalTotal);
        order.events.add(new OrderPlaced(order.id, customerId, orderLines, finalTotal));
        return order;
    }
}

// Application Layer — 오케스트레이션만, 도메인 로직 없음
@Service
@Transactional
public class OrderApplicationService {

    // 의존성이 2개로 줄어듦 (Repository + EventPublisher)
    private final OrderRepository orderRepository;
    private final ApplicationEventPublisher eventPublisher;

    public OrderId placeOrder(PlaceOrderCommand command) {
        Optional<Coupon> coupon = command.couponId()
            .flatMap(couponRepository::findById);

        // 도메인 로직은 Order가 담당
        Order order = Order.place(
            command.customerId(),
            command.lines(),
            coupon
        );

        orderRepository.save(order);

        // 이벤트 발행 — 인프라 역할
        order.pullEvents().forEach(eventPublisher::publishEvent);

        return order.getId();
    }
}

// Infrastructure Layer — Domain 인터페이스 구현
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final JpaOrderJpaRepository jpaRepo;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        jpaRepo.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepo.findById(id.value())
            .map(mapper::toDomain);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 의존성 방향이 역전된다

```
전통 레이어드 (위→아래 의존):
  Controller → Service → Repository → DB
  
  문제: Repository(JPA)를 바꾸면 Service가 영향받음
       Service를 테스트하려면 Repository(JPA)가 필요

DDD 레이어 (안→밖 의존, 도메인이 중심):
  
               ┌─────────────────────────────────┐
               │          Domain Layer           │ ← 아무것도 의존 안 함
               │  Order, OrderRepository(인터페이스)│
               └──────────────┬──────────────────┘
                              │ 사용 (의존)
     ┌────────────────────────▼────────────────────────┐
     │                                                 │
  Application Layer              Infrastructure Layer
  (OrderApplicationService)      (JpaOrderRepository)
     │                                   │
     └──────────── Domain 사용 ────────────┘
     
  핵심: 인프라가 도메인을 의존
        도메인은 인프라를 모름
  
  Repository 구현체를 JPA → MongoDB로 교체?
  → Infrastructure Layer만 수정
  → Domain Layer, Application Layer 무관

  Domain Layer를 테스트?
  → JPA, Spring 없이 순수 Java로 테스트
  → Mock 거의 필요 없음
```

### 2. 각 레이어의 명확한 책임

```
Presentation Layer 책임:
  ① HTTP 요청을 Command/Query 객체로 변환
  ② Application Service 호출
  ③ 결과를 HTTP 응답으로 변환
  ④ 입력값 형식 검증 (길이, 형식, null 체크)
  
  포함하면 안 되는 것:
  ❌ 비즈니스 로직 (할인 계산, 재고 확인)
  ❌ DB 조회

Application Layer 책임:
  ① 유스케이스(Use Case) 흐름 조율
  ② 트랜잭션 경계 정의 (@Transactional)
  ③ Domain 객체 조회 → Domain 메서드 호출 → 저장
  ④ 이벤트 발행, 외부 서비스 호출 (인프라 통해)
  
  포함하면 안 되는 것:
  ❌ 비즈니스 규칙 ("취소 가능한가?" 판단은 Domain이)
  ❌ SQL 쿼리, JPA 직접 사용

Domain Layer 책임:
  ① 비즈니스 규칙과 불변식 보호
  ② 상태 전이 정의
  ③ Domain Event 발행 (수집)
  ④ Repository 인터페이스 정의 (구현은 Infrastructure에)
  
  포함하면 안 되는 것:
  ❌ @Entity, @Column (JPA 어노테이션) — 실용적 타협 시 허용
  ❌ 외부 API 호출
  ❌ ApplicationEventPublisher 직접 사용

Infrastructure Layer 책임:
  ① Domain의 Repository 인터페이스 구현 (JPA, MongoDB 등)
  ② 외부 API 클라이언트 (Payment Gateway, SMS API 등)
  ③ 이벤트 발행 구현 (Kafka, RabbitMQ 등)
  ④ Domain과 JPA Entity 간 매핑 (Mapper)
```

### 3. Hexagonal Architecture (육각형 아키텍처)와의 관계

```
Hexagonal Architecture = Ports and Adapters 패턴
  (Alistair Cockburn 제안, DDD와 잘 어울림)

  Port (포트) = Domain Layer가 정의하는 인터페이스
    Inbound Port: Application Service가 외부에 노출하는 인터페이스
    Outbound Port: Domain이 필요로 하는 인프라 인터페이스 (Repository 등)

  Adapter (어댑터) = 외부 세계와 Port를 연결하는 구현체
    Inbound Adapter: Controller, CLI, Test (Port를 호출)
    Outbound Adapter: JpaRepository, KafkaPublisher (Port를 구현)

                     ┌─────────────────────────────────────┐
  [Controller]       │         Hexagon (Domain + App)       │     [JpaRepository]
  [REST API]  ──────>│  PlaceOrderUseCase (Inbound Port)    │──> [OrderRepository Port]
  [CLI]       ──────>│                                      │     구현체
                     │  OrderApplicationService             │
                     │  Order (Aggregate)                   │
                     │  OrderRepository (Outbound Port)     │──> [KafkaPublisher]
                     └─────────────────────────────────────┘

  장점:
    Controller를 CLI로 교체 → Domain 무관
    JPA를 MongoDB로 교체  → Domain 무관
    Domain 테스트 → Adapter 없이 가능 (가짜 Adapter로 대체)

DDD + Hexagonal 조합:
  DDD: 도메인 모델 설계 방법론
  Hexagonal: 의존성 방향을 잡는 아키텍처 패턴
  → DDD의 Domain Layer를 Hexagon 중심에 배치하는 것이 자연스러운 조합
```

### 4. 테스트 피라미드와 레이어 분리의 관계

```
테스트 피라미드:

        ┌───────────┐
        │   E2E     │ ← 소수, 전체 흐름 검증 (느림, 비쌈)
      ┌─┤   Tests   ├─┐
      │ └───────────┘ │
    ┌─┤  Integration  ├─┐
    │ │     Tests     │ │ ← Application Layer 테스트
    │ └───────────────┘ │
  ┌─┤    Unit Tests     ├─┐
  │ │ (Domain Layer)    │ │ ← 다수, 빠름, Mock 최소 (인프라 없음)
  └─└───────────────────┘─┘

Domain Layer 분리 시의 테스트 이점:
  Order.cancel() 테스트:
    → JPA 불필요
    → Spring 불필요
    → Mock 불필요
    → 순수 Java 객체 생성 후 메서드 호출
    → 속도: ~1ms (JPA 테스트 대비 수백 배 빠름)

  Application Layer 테스트:
    → Repository는 InMemory 구현체로 대체
    → 외부 API는 Mock으로 대체
    → DB 없이 유스케이스 검증

  Infrastructure Layer 테스트:
    → JPA 실제 사용 (@DataJpaTest)
    → SQL 쿼리 정확성 검증
    → 소수만 작성 (느리므로)

레이어가 섞이면:
  Order에 @Entity가 있으면 → Order 테스트에 Spring이 필요
  Service가 JPA에 직접 의존 → Service 테스트에 DB가 필요
  → 테스트가 느려지고, 모든 테스트가 통합 테스트 수준으로 무거워짐
```

---

## 💻 실전 코드

### 패키지 구조 — 레이어 분리가 패키지로 표현됨

```
com.example.order/
│
├── presentation/                   ← Presentation Layer
│   ├── OrderController.java
│   ├── request/
│   │   └── PlaceOrderRequest.java
│   └── response/
│       └── OrderResponse.java
│
├── application/                    ← Application Layer
│   ├── OrderApplicationService.java
│   └── command/
│       └── PlaceOrderCommand.java
│
├── domain/                         ← Domain Layer ★ 핵심
│   ├── Order.java                  (Aggregate Root)
│   ├── OrderLine.java              (Entity)
│   ├── OrderId.java                (Value Object)
│   ├── OrderStatus.java            (Enum)
│   ├── OrderRepository.java        (Repository 인터페이스)
│   └── event/
│       ├── OrderPlaced.java        (Domain Event)
│       └── OrderCancelled.java
│
└── infrastructure/                 ← Infrastructure Layer
    ├── JpaOrderRepository.java     (Repository 구현체)
    ├── OrderJpaEntity.java         (JPA Entity — domain과 분리)
    └── OrderMapper.java            (Domain ↔ JPA Entity 변환)
```

### Domain Layer의 순수성 확인 — ArchUnit

```java
@AnalyzeClasses(packages = "com.example.order")
class LayerDependencyTest {

    @ArchTest
    static ArchRule domainLayerShouldNotDependOnInfrastructure =
        noClasses()
            .that().resideInAPackage("..order.domain..")
            .should().dependOnClassesThat()
            .resideInAnyPackage(
                "..order.infrastructure..",
                "org.springframework.data.jpa..",
                "jakarta.persistence.."
            )
            .because("Domain Layer는 JPA, 인프라에 의존하면 안 됩니다");

    @ArchTest
    static ArchRule applicationLayerShouldNotDependOnPresentation =
        noClasses()
            .that().resideInAPackage("..order.application..")
            .should().dependOnClassesThat()
            .resideInAPackage("..order.presentation..");
}
```

### InMemory Repository로 Domain Layer 단위 테스트

```java
// 테스트용 InMemory 구현체 — JPA 없이 Domain 테스트
class InMemoryOrderRepository implements OrderRepository {

    private final Map<OrderId, Order> store = new HashMap<>();

    @Override
    public void save(Order order) {
        store.put(order.getId(), order);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id));
    }
}

// Application Service 테스트 — DB, Spring 없이
class OrderApplicationServiceTest {

    private OrderRepository orderRepository = new InMemoryOrderRepository();
    private FakeEventPublisher eventPublisher = new FakeEventPublisher();
    private OrderApplicationService sut = new OrderApplicationService(
        orderRepository, eventPublisher
    );

    @Test
    void placeOrder_savesOrderAndPublishesEvent() {
        PlaceOrderCommand command = new PlaceOrderCommand(
            new CustomerId(1L),
            List.of(new OrderLineRequest(new ProductId(1L), 2, Money.of(10_000)))
        );

        OrderId orderId = sut.placeOrder(command);

        Order saved = orderRepository.findById(orderId).orElseThrow();
        assertThat(saved.getStatus()).isEqualTo(OrderStatus.PENDING);

        assertThat(eventPublisher.published()).hasSize(1);
        assertThat(eventPublisher.published().get(0)).isInstanceOf(OrderPlaced.class);
    }
}
```

---

## 📊 설계 비교

```
전통 레이어드 vs DDD 레이어드:

                전통 레이어드            DDD 레이어드
────────────┼──────────────────────┼──────────────────────────
레이어 수    │ 3개                   │ 4개
            │ (Presentation,       │ (Presentation,
            │  Business, Data)      │  Application, Domain,
            │                      │  Infrastructure)
────────────┼──────────────────────┼──────────────────────────
도메인 로직  │ Service에 흩어짐       │ Domain Layer에 응집
────────────┼──────────────────────┼──────────────────────────
의존성 방향  │ 위 → 아래 (단방향)     │ 외부 → 도메인 (안쪽으로)
            │ Controller→Service   │ 인프라가 도메인을 의존
            │ →Repository          │ (의존성 역전)
────────────┼──────────────────────┼──────────────────────────
Domain 테스트│ Service Mock 다수     │ 순수 Java, Mock 최소
────────────┼──────────────────────┼──────────────────────────
인프라 교체  │ Service까지 영향       │ Infrastructure만 교체
────────────┼──────────────────────┼──────────────────────────
Service 크기 │ 계속 비대해짐           │ 오케스트레이션만
            │ (300~1000줄)         │ (50~100줄)
────────────┼──────────────────────┼──────────────────────────
초기 구조   │ 단순 (3레이어)          │ 복잡 (4레이어 + 패키지)
```

---

## ⚖️ 트레이드오프

```
DDD 레이어 구조의 비용:
  ① 파일 수 증가
     Domain Object + JPA Entity + Mapper 가 분리되면
     하나의 개념에 3개의 파일이 생김
     → 단순한 도메인에서는 과잉

  ② 학습 곡선
     레이어 간 경계, 의존성 방향 원칙을 팀 전체가 이해해야 함

  ③ 초기 보일러플레이트
     Repository 인터페이스 + 구현체 + Mapper 초기 세팅

실용적 타협 방법:
  ① Domain Object와 JPA Entity를 같은 클래스로 사용
     @Entity를 Domain 클래스에 붙임 (순수 DDD에서는 ❌)
     → 단순 도메인에서는 Mapper 비용을 줄임
     → 복잡도가 올라가면 분리로 리팩터링

  ② Application Service와 Domain Service 통합 (소규모 팀)
     → 유스케이스 수가 적을 때 Service 하나로 시작

  ③ Repository 인터페이스 생략 (완전 분리 불필요 시)
     → Spring Data JPA Repository 직접 사용
     → 나중에 인터페이스 추출 가능

선택 기준:
  팀 규모와 시스템 복잡도에 따라 레이어 분리 수준 결정
  "레이어 구조가 팀을 돕는가, 방해하는가?" 가 판단 기준
```

---

## 📌 핵심 정리

```
DDD 레이어 구조 핵심:

4개 레이어:
  Presentation  → 입출력 변환 (Controller, DTO)
  Application   → 유스케이스 오케스트레이션 (트랜잭션, 순서)
  Domain        → 비즈니스 규칙 (Aggregate, VO, Domain Event)
  Infrastructure → 기술 구현 (JPA, Kafka, 외부 API)

의존성 방향:
  도메인이 중심 → 다른 레이어가 도메인을 의존
  도메인은 인프라를 모름 (의존성 역전)

테스트 이점:
  Domain Layer = 순수 Java → Mock 없이 단위 테스트
  Application Layer = InMemory Repository로 테스트
  Infrastructure Layer = 통합 테스트 (소수)

Hexagonal Architecture:
  Port = 도메인이 정의하는 인터페이스 (Repository, Use Case)
  Adapter = 외부와의 연결 (Controller, JpaRepository)
  → DDD Domain Layer를 Hexagon 중심에 배치

실용적 시작:
  Domain Object에 @Entity 허용 (타협)
  → Service가 비대해지면 Application/Domain 분리 신호
  → Repository가 JPA에 직접 의존하면 도메인 오염 신호
```

---

## 🤔 생각해볼 문제

**Q1.** `@Transactional`은 어느 레이어에 있어야 하는가? Domain Layer에 두면 안 되는 이유는?

<details>
<summary>해설 보기</summary>

**`@Transactional`은 Application Layer(Application Service)에 있어야 합니다.**

**Domain Layer에 두면 안 되는 이유:**

`@Transactional`은 Spring의 어노테이션이므로, Domain Layer에 붙이면 Domain이 Spring에 의존하게 됩니다. 이는 Domain Layer의 순수성을 깨트립니다.

```java
// ❌ Domain Layer에 @Transactional
public class Order {
    @Transactional  // ← Spring 의존! Domain이 인프라를 알게 됨
    public void cancel() { ... }
}

// ✅ Application Layer에 @Transactional
@Service
public class OrderApplicationService {
    @Transactional  // ← 트랜잭션 경계는 Application이 정의
    public void cancelOrder(OrderId orderId) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        order.cancel();  // Domain 로직 호출
        orderRepository.save(order);
    }
}
```

**추가 이유**: 트랜잭션 경계는 "어느 범위가 하나의 비즈니스 연산인가"를 결정하는 것으로, 유스케이스 수준의 결정입니다. 이것은 Domain 로직이 아니라 Application 수준의 조율입니다.

</details>

---

**Q2.** `OrderRepository`를 Domain Layer의 인터페이스로 정의하고, `JpaOrderRepository`를 Infrastructure Layer에서 구현할 때, Spring Data JPA의 `JpaRepository`를 어떻게 연결하는가?

<details>
<summary>해설 보기</summary>

**두 가지 방법이 있습니다.**

**방법 1: 위임(Delegation) 패턴 — 가장 명확한 분리**

```java
// Domain Layer — 순수 인터페이스
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// Infrastructure Layer — Spring Data JPA 내부에서 사용
interface OrderJpaRepository extends JpaRepository<OrderJpaEntity, Long> {
    // Spring Data JPA 메서드
}

// Infrastructure Layer — Domain 인터페이스 구현
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final OrderJpaRepository jpaRepository;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toEntity(order);
        jpaRepository.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepository.findById(id.value())
            .map(mapper::toDomain);
    }
}
```

**방법 2: Domain Repository가 JpaRepository를 직접 상속 (실용적 타협)**

```java
// Domain + Infrastructure 경계 타협
public interface OrderRepository extends JpaRepository<Order, Long> {
    // Order에 @Entity가 있을 때 가능
    // 간단하지만 Domain이 JPA를 알게 됨 (순수 DDD 위반)
}
```

**선택 기준**: 도메인 모델의 순수성이 중요하고, JPA를 나중에 교체할 가능성이 있다면 → 방법 1. 실용적 타협이 필요하고 단순성을 원한다면 → 방법 2.

</details>

---

**Q3.** 레이어드 아키텍처에서 Controller가 Repository를 직접 호출하면 안 되는 이유는? 실제로 허용되는 경우는 없는가?

<details>
<summary>해설 보기</summary>

**원칙적으로 Controller → Repository 직접 호출은 금지입니다.**

**이유:**

1. **비즈니스 로직 우회**: Application Service가 담당하는 트랜잭션 경계, 도메인 규칙 적용, 이벤트 발행이 모두 건너뜁니다.
2. **레이어 책임 위반**: Controller는 입출력 변환만 담당해야 하는데, 데이터 접근 로직이 섞입니다.
3. **테스트 어려움**: Controller 테스트 시 Repository를 Mock해야 하는 의존성이 추가됩니다.

**예외적으로 허용되는 경우:**

CQRS(Command Query Responsibility Segregation) 패턴에서 **Query(조회) 쪽**은 예외입니다:

```java
// 조회 전용 — Controller가 읽기 전용 Repository 직접 사용 허용
@GetMapping("/orders/{id}")
public OrderDetailResponse getOrder(@PathVariable Long id) {
    // 단순 조회: 도메인 로직 없음, 이벤트 없음, 상태 변경 없음
    return orderReadRepository.findDetailById(id)  // 읽기 전용 DTO 직접 반환
        .orElseThrow(() -> new OrderNotFoundException(id));
}
```

쓰기(Command)는 반드시 Application Service를 거쳐야 하고, 읽기(Query)는 성능과 단순성을 위해 Repository를 직접 사용하는 것을 허용하는 것이 CQRS의 실용적 적용입니다.

</details>

---

<div align="center">

**[⬅️ 이전: DDD 적용 판단 기준](./04-when-to-apply-ddd.md)** | **[홈으로 🏠](../README.md)** | **[Chapter 2로 이동: Strategic Design — 서브도메인 분류 ➡️](../strategic-design/01-subdomain-classification.md)**

</div>
