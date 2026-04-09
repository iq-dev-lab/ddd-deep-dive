# Repository 구현 — 도메인 인터페이스를 인프라가 구현한다

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 도메인 레이어에 Repository 인터페이스만 두고 인프라 레이어에서 구현체를 두는 구조는 어떻게 생겼는가?
- Spring Data JPA가 도메인 Repository 인터페이스를 구현하도록 어떻게 연결하는가?
- 이 구조에서 도메인 테스트가 인프라 없이 가능한 이유는?
- 커스텀 쿼리가 필요한 경우 Spring Data JPA 구현체에 어떻게 추가하는가?
- 도메인 Repository 인터페이스를 `JpaRepository`를 직접 상속하는 것과 비교했을 때 장단점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

대부분의 Spring 프로젝트는 `interface OrderRepository extends JpaRepository<Order, Long>`으로 시작한다. 빠르고 편리하지만, 도메인 Repository가 JPA에 직접 의존한다. 도메인 레이어에서 `JpaRepository`, `Pageable`, `Sort` 같은 Spring Data 타입이 보이기 시작한다.

도메인 인터페이스와 JPA 구현체를 분리하면, 도메인 테스트에서 `JpaRepository`를 mock할 필요가 없어지고, `InMemoryRepository`로 빠른 순수 단위 테스트가 가능해진다. 또한 JPA를 다른 기술로 교체해도 도메인 코드는 변하지 않는다.

---

## 😱 흔한 실수 (Before — 도메인 Repository가 JPA에 직접 의존)

```java
// 도메인 레이어에 JPA 의존 침투
package com.example.order.domain;

import org.springframework.data.jpa.repository.JpaRepository;  // ← 인프라 import!
import org.springframework.data.domain.Pageable;               // ← Spring Data 타입!

public interface OrderRepository extends JpaRepository<Order, Long> {

    // JPA 쿼리 어노테이션이 도메인 인터페이스에
    @Query("SELECT o FROM Order o WHERE o.status = :status")
    List<Order> findByStatus(@Param("status") String status);

    // Spring Data 타입(Pageable)이 도메인 인터페이스에
    Page<Order> findByCustomerId(Long customerId, Pageable pageable);
}
```

```
문제:
  도메인 레이어가 spring-data-jpa에 의존
  → build.gradle에서 도메인 모듈이 JPA를 알아야 함
  → 도메인만 테스트할 때 JPA 설정 필요
  
  @Query, Pageable 등 JPA/Spring Data 타입이 도메인에 노출
  → MongoDB로 전환 시 도메인 인터페이스도 수정 필요
  
  InMemoryOrderRepository로 교체하려면:
  JpaRepository 메서드(save, findAll, deleteAll 등)도 모두 구현해야 함
  → 테스트용 구현이 매우 복잡해짐
```

---

## ✨ 올바른 접근 (After — 도메인 인터페이스 + 인프라 구현체)

```java
// 도메인 레이어: 순수 도메인 언어 (JPA import 없음)
package com.example.order.domain.repository;

public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    List<Order> findPendingOrders();
    boolean existsById(OrderId id);
    void remove(OrderId id);
}

// 인프라 레이어: Spring Data JPA 구현
package com.example.order.infrastructure.persistence;

// Spring Data 내부용 JPA Repository (인프라에만 존재)
interface SpringDataOrderRepository
    extends JpaRepository<Order, String> {  // 인프라 레이어에만

    @Query("""
        SELECT DISTINCT o FROM Order o
        LEFT JOIN FETCH o.lines
        WHERE o.customerId = :customerId
        ORDER BY o.createdAt DESC
        """)
    List<Order> findByCustomerIdWithLines(@Param("customerId") String customerId);

    @Query("""
        SELECT DISTINCT o FROM Order o
        LEFT JOIN FETCH o.lines
        WHERE o.status = 'PENDING'
        """)
    List<Order> findPendingOrdersWithLines();
}

// 도메인 인터페이스 구현체 (인프라 레이어)
@Repository
public class JpaOrderRepository implements OrderRepository {

    private final SpringDataOrderRepository springDataRepo;

    @Override
    public void save(Order order) {
        springDataRepo.save(order);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        // JOIN FETCH로 lines 함께 로드
        return springDataRepo.findByIdWithLines(id.value().toString());
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return springDataRepo.findByCustomerIdWithLines(customerId.value().toString());
    }

    @Override
    public List<Order> findPendingOrders() {
        return springDataRepo.findPendingOrdersWithLines();
    }

    @Override
    public boolean existsById(OrderId id) {
        return springDataRepo.existsById(id.value().toString());
    }

    @Override
    public void remove(OrderId id) {
        springDataRepo.deleteById(id.value().toString());
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 의존성 역전 구조

```
의존성 방향:

Before (잘못된 방향):
  Domain → Spring Data JPA (인프라)
  
  OrderRepository (Domain) extends JpaRepository (Infrastructure)
  → Domain이 Infrastructure에 의존

After (올바른 방향):
  Infrastructure → Domain
  
  JpaOrderRepository (Infrastructure) implements OrderRepository (Domain)
  → Infrastructure가 Domain에 의존

패키지 구조:

  com.example.order/
  ├── domain/
  │   ├── Order.java
  │   ├── OrderId.java
  │   └── repository/
  │       └── OrderRepository.java       ← 인터페이스 (Domain 소유)
  └── infrastructure/
      └── persistence/
          ├── SpringDataOrderRepository.java  ← JpaRepository 상속 (인프라)
          └── JpaOrderRepository.java         ← OrderRepository 구현 (인프라)

결과:
  도메인 모듈은 spring-data-jpa 의존성 없음
  인프라 모듈만 spring-data-jpa에 의존
  도메인 테스트에 JPA 설정 불필요
```

### 2. InMemory Repository로 도메인 테스트

```java
// 테스트용 InMemory 구현
public class InMemoryOrderRepository implements OrderRepository {

    private final Map<OrderId, Order> store = new ConcurrentHashMap<>();

    @Override
    public void save(Order order) {
        store.put(order.id(), order);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id));
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return store.values().stream()
            .filter(o -> o.customerId().equals(customerId))
            .collect(Collectors.toList());
    }

    @Override
    public List<Order> findPendingOrders() {
        return store.values().stream()
            .filter(o -> o.status() == OrderStatus.PENDING)
            .collect(Collectors.toList());
    }

    @Override
    public boolean existsById(OrderId id) {
        return store.containsKey(id);
    }

    @Override
    public void remove(OrderId id) {
        store.remove(id);
    }

    // 테스트 유틸
    public int size() { return store.size(); }
    public void clear() { store.clear(); }
    public Collection<Order> findAll() { return store.values(); }
}
```

### 3. 도메인 테스트 — Spring 없음

```java
// 완전히 순수한 도메인 테스트
class OrderApplicationServiceTest {

    // Spring Context 없음, DB 없음, @SpringBootTest 없음
    private final InMemoryOrderRepository orderRepository = new InMemoryOrderRepository();
    private final FakeEventPublisher eventPublisher = new FakeEventPublisher();
    private final OrderFactory orderFactory = new OrderFactory(
        new PricingDomainService(),
        new ShippingDomainService()
    );

    private final OrderApplicationService sut = new OrderApplicationService(
        orderRepository,
        orderFactory,
        eventPublisher
    );

    @BeforeEach
    void setUp() {
        orderRepository.clear();
        eventPublisher.clear();
    }

    @Test
    void placeOrder_savesOrderAndPublishesEvent() {
        PlaceOrderCommand command = PlaceOrderCommand.of(CUSTOMER_ID, ITEMS);

        OrderId orderId = sut.placeOrder(command);

        assertThat(orderRepository.findById(orderId)).isPresent();
        assertThat(eventPublisher.published())
            .hasSize(1)
            .first()
            .isInstanceOf(OrderPlaced.class);
    }

    @Test
    void cancelOrder_afterPlacing_changesStatusAndPublishesEvent() {
        OrderId orderId = sut.placeOrder(PlaceOrderCommand.of(CUSTOMER_ID, ITEMS));

        sut.cancelOrder(new CancelOrderCommand(orderId, "고객 요청"));

        Order order = orderRepository.findById(orderId).orElseThrow();
        assertThat(order.status()).isEqualTo(OrderStatus.CANCELLED);
        assertThat(eventPublisher.published())
            .anyMatch(e -> e instanceof OrderCancelled);
    }
}

// FakeEventPublisher
public class FakeEventPublisher implements ApplicationEventPublisher {

    private final List<Object> events = new ArrayList<>();

    @Override
    public void publishEvent(Object event) {
        events.add(event);
    }

    public List<Object> published() { return List.copyOf(events); }
    public void clear() { events.clear(); }
}
```

### 4. Spring Data JPA와 연결하는 방법

```java
// 방법 1: @Repository 어노테이터를 붙인 구현체 (기본 방법)
@Repository
public class JpaOrderRepository implements OrderRepository {
    // Spring이 @Repository 빈으로 등록 → OrderRepository 타입으로 주입 가능
}

// Application Service에서 사용
@Service
public class OrderApplicationService {
    private final OrderRepository orderRepository;  // 도메인 인터페이스 타입

    public OrderApplicationService(OrderRepository orderRepository) {
        this.orderRepository = orderRepository;
        // 런타임: JpaOrderRepository 주입 (Spring이 구현체 선택)
        // 테스트:  InMemoryOrderRepository 주입 (수동 주입)
    }
}

// 방법 2: 별도 @Configuration으로 명시적 바인딩
@Configuration
public class RepositoryConfig {

    @Bean
    public OrderRepository orderRepository(SpringDataOrderRepository springDataRepo) {
        return new JpaOrderRepository(springDataRepo);
    }
}
```

---

## 💻 실전 코드

### 커스텀 쿼리가 있는 완전한 구현

```java
// 도메인 인터페이스 — 비즈니스 언어
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
    List<Order> findByCustomerId(CustomerId customerId);
    List<Order> findOrdersToShipBefore(LocalDate deadline);
    long countByCustomerIdAndStatus(CustomerId customerId, OrderStatus status);
}

// Spring Data 내부용 (인프라만)
interface SpringDataOrderRepository extends JpaRepository<Order, String>,
                                             JpaSpecificationExecutor<Order> {

    @Query("""
        SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.lines
        WHERE o.id = :id
        """)
    Optional<Order> findByIdWithLines(@Param("id") String id);

    @Query("""
        SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.lines
        WHERE o.customerId = :customerId
        ORDER BY o.createdAt DESC
        """)
    List<Order> findByCustomerIdWithLines(@Param("customerId") String customerId);

    @Query("""
        SELECT DISTINCT o FROM Order o LEFT JOIN FETCH o.lines
        WHERE o.status = 'PAID'
          AND o.estimatedShipDate <= :deadline
        """)
    List<Order> findOrdersToShipBefore(@Param("deadline") LocalDate deadline);

    long countByCustomerIdAndStatus(String customerId, OrderStatus status);
}

// 구현체
@Repository
@RequiredArgsConstructor
public class JpaOrderRepository implements OrderRepository {

    private final SpringDataOrderRepository springDataRepo;

    @Override
    public void save(Order order) {
        springDataRepo.save(order);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return springDataRepo.findByIdWithLines(id.value().toString());
    }

    @Override
    public List<Order> findByCustomerId(CustomerId customerId) {
        return springDataRepo.findByCustomerIdWithLines(customerId.value().toString());
    }

    @Override
    public List<Order> findOrdersToShipBefore(LocalDate deadline) {
        return springDataRepo.findOrdersToShipBefore(deadline);
    }

    @Override
    public long countByCustomerIdAndStatus(CustomerId customerId, OrderStatus status) {
        return springDataRepo.countByCustomerIdAndStatus(
            customerId.value().toString(), status
        );
    }
}
```

---

## 📊 설계 비교

```
Domain extends JpaRepository vs 분리된 인터페이스:

                Domain extends JpaRepository   분리된 인터페이스 + 구현체
────────────┼──────────────────────────────┼──────────────────────────
코드량       │ 적음 (인터페이스만)            │ 많음 (인터페이스 + 구현체)
────────────┼──────────────────────────────┼──────────────────────────
Domain 순수성 │ 낮음 (JPA 타입 침투)          │ 높음 (도메인 언어만)
────────────┼──────────────────────────────┼──────────────────────────
테스트 독립  │ Mock Jpa 필요                 │ InMemory 구현만 있으면 됨
────────────┼──────────────────────────────┼──────────────────────────
기술 교체   │ 도메인 인터페이스도 수정       │ 구현체만 교체
────────────┼──────────────────────────────┼──────────────────────────
테스트 속도  │ @DataJpaTest 필요 (느림)       │ 순수 Java 테스트 (빠름)
────────────┼──────────────────────────────┼──────────────────────────
팀 적합성   │ 소규모, 빠른 개발              │ 중~대규모, DDD 중요 팀
```

---

## ⚖️ 트레이드오프

```
분리된 구현체의 비용:
  클래스 수 증가: 인터페이스 + Spring Data 인터페이스 + 구현체
  보일러플레이트: save(), findById() 등 위임 코드 반복

실용적 단순화:
  Spring Data의 Querydsl, Specification을 사용하고 싶다면
  → 구현체에서 JpaSpecificationExecutor를 사용하고
     도메인 인터페이스에는 노출하지 않음

  모든 Repository를 완전 분리할 필요는 없음:
  → Core Domain Aggregate는 분리 (테스트 독립성 중요)
  → Supporting Domain의 단순 CRUD는 JpaRepository 직접 사용도 허용
```

---

## 📌 핵심 정리

```
Repository 구현 핵심:

구조:
  도메인 인터페이스 (Domain 소유) → JPA import 없음
  SpringDataRepository (인프라) → JpaRepository 상속
  JpaXxxRepository (인프라) → 도메인 인터페이스 구현

장점:
  도메인 테스트: InMemoryRepository로 JPA 없이
  기술 독립: JPA → MongoDB 전환 시 도메인 무관
  명확한 API: 도메인 언어로만 표현된 인터페이스

InMemoryRepository:
  Map<Id, Aggregate>로 구현
  도메인 테스트에 주입 → Spring Context 없이 빠른 테스트
  변경 감지 없음 → 명시적 save() 확인 가능

Spring DI 연결:
  @Repository 구현체 → Spring이 자동 등록
  타입: OrderRepository (도메인) → 주입 시 JpaOrderRepository 선택
```

---

## 🤔 생각해볼 문제

**Q1.** `InMemoryOrderRepository`에서 `save(order)` 후 같은 객체를 수정하면 저장된 내용도 바뀌는 문제가 발생할 수 있다. 어떻게 방지하는가?

<details>
<summary>해설 보기</summary>

**깊은 복사(Deep Copy)로 방지합니다.**

```java
public class InMemoryOrderRepository implements OrderRepository {

    private final Map<OrderId, Order> store = new ConcurrentHashMap<>();

    @Override
    public void save(Order order) {
        // 얕은 복사 문제: 같은 참조 저장 시 외부에서 수정하면 저장된 것도 바뀜
        store.put(order.id(), deepCopy(order));  // 깊은 복사 필요
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return Optional.ofNullable(store.get(id))
            .map(this::deepCopy);  // 반환 시도 깊은 복사
    }

    private Order deepCopy(Order order) {
        // 직렬화/역직렬화로 깊은 복사
        try {
            byte[] bytes = objectMapper.writeValueAsBytes(order);
            return objectMapper.readValue(bytes, Order.class);
        } catch (Exception e) {
            throw new RuntimeException("깊은 복사 실패", e);
        }
    }
}
```

**간단한 대안**: Aggregate가 불변(Immutable)하게 설계돼 있다면 얕은 복사도 안전합니다. 대부분의 경우 Aggregate는 내부 상태가 변하므로 깊은 복사가 안전합니다. 테스트 목적이라면 이 차이를 인지하고 사용하면 됩니다.

</details>

---

**Q2.** 도메인 `OrderRepository` 인터페이스에 `findAll()`이 없다. 전체 목록을 가져와야 할 때 어떻게 하는가?

<details>
<summary>해설 보기</summary>

**도메인 Repository에 `findAll()`을 노출하는 것은 일반적으로 안티패턴입니다.**

도메인 Repository의 `findAll()`이 필요한 상황 분석:
- 관리자 페이지에서 전체 주문 목록 → 읽기 모델(Query Side) 책임
- 배치 작업에서 전체 조회 → 별도 배치 Repository 또는 직접 쿼리

```java
// 읽기 측: 별도 Query Repository (도메인 규칙 없음)
public interface OrderQueryRepository {
    Page<OrderSummaryDto> findAllOrders(OrderSearchCondition condition, Pageable pageable);
}

// 배치 전용: Chunk 단위 처리
@Configuration
public class OrderBatchConfig {
    @Bean
    public JpaPagingItemReader<Order> orderItemReader(EntityManagerFactory emf) {
        return new JpaPagingItemReaderBuilder<Order>()
            .entityManagerFactory(emf)
            .queryString("SELECT o FROM Order o WHERE o.status = 'PENDING'")
            .pageSize(100)
            .build();
    }
}
```

진짜로 필요하다면 도메인 인터페이스에 의미 있는 이름으로 추가:
```java
public interface OrderRepository {
    List<Order> findAllPendingOrdersForDailyReport();  // 일별 보고서용
    // findAll() 대신 명확한 의도가 담긴 메서드
}
```

</details>

---

**Q3.** `JpaOrderRepository` 구현체에서 `SpringDataOrderRepository.save()`를 호출할 때 JPA 더티 체킹이 작동하는가?

<details>
<summary>해설 보기</summary>

**구현체를 통해도 JPA 더티 체킹은 작동합니다. 단, 트랜잭션 경계 안에서만.**

```java
@Service
@Transactional
public class OrderApplicationService {

    public void cancelOrder(CancelOrderCommand command) {
        Order order = orderRepository.findById(command.orderId()).orElseThrow();
        // order는 영속성 컨텍스트에 의해 관리됨 (Managed 상태)

        order.cancel(command.reason());
        // JPA 더티 체킹: 트랜잭션 커밋 시 자동 UPDATE

        orderRepository.save(order);  // 명시적 save는 선택적 (더티 체킹으로도 저장됨)
        // 하지만 명시적 save()를 권장 — 의도 명확성
    }
}
```

**InMemoryRepository와의 차이:**
- JPA: `save()` 없이도 더티 체킹으로 자동 저장
- InMemory: `save()` 없으면 변경 저장 안 됨

이 차이 때문에 명시적 `save()`를 항상 호출하는 것을 권장합니다. InMemory와 JPA 구현체가 동일하게 동작해야 테스트 신뢰도가 높아집니다.

</details>

---

<div align="center">

**[⬅️ 이전: Aggregate JPA 매핑](./04-aggregate-jpa-mapping.md)** | **[홈으로 🏠](../README.md)** | **[다음: Domain Event Spring 통합 ➡️](./06-domain-event-spring.md)**

</div>
