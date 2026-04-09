# JPA Entity와 DDD Entity 분리 패턴 — Mapper로 변환하는 비용

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Domain Object와 JPA Entity를 완전히 분리했을 때 얻는 것과 잃는 것은?
- Mapper 클래스가 두 레이어 사이를 변환하는 구조는 어떻게 생겼는가?
- 두 모델이 섞였을 때 `@Transient`가 증식하는 문제란 무엇인가?
- 어느 팀/프로젝트에 완전 분리가 적합하고 어느 경우에는 과잉인가?
- MapStruct를 활용한 Mapper 자동화는 어떻게 하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Ch5-01에서 실용적 PI를 다뤘다. 이 문서는 PI의 극단적 형태인 **완전 분리**를 깊이 다룬다. 도메인 클래스와 JPA Entity가 완전히 다른 클래스가 되면, 도메인은 순수하지만 Mapper라는 변환 비용이 생긴다. 이 비용이 언제 정당화되는지, 어떻게 최소화하는지가 핵심이다.

---

## 😱 흔한 실수 (Before — 두 모델이 섞여 @Transient 증식)

```java
// JPA Entity와 도메인 모델이 섞인 클래스
@Entity
public class Order {

    @Id @GeneratedValue
    private Long id;

    private String status;  // DB 컬럼

    // 도메인 로직에 필요하지만 JPA 영속 대상이 아님 → @Transient 폭발
    @Transient
    private List<DomainEvent> domainEvents = new ArrayList<>();

    @Transient
    private OrderPricingPolicy pricingPolicy;  // 도메인 정책 객체 (DB에 저장 안 함)

    @Transient
    private boolean isNewOrder;  // 도메인 플래그

    @Transient
    private List<String> validationMessages;

    // 영속 필드와 비영속 필드가 뒤섞임
    // → 어떤 필드가 DB에 저장되는지 파악 어려움
    // → JPA 직렬화 규칙이 도메인 설계에 영향

    @PostLoad  // JPA 생명주기 콜백이 도메인에 침투
    private void initTransientFields() {
        this.domainEvents = new ArrayList<>();
        this.pricingPolicy = new StandardPricingPolicy();
    }
}
```

---

## ✨ 올바른 접근 (After — 완전 분리 + Mapper)

```java
// 1. 순수 Domain 클래스 (JPA import 없음)
public class Order {  // no @Entity, no @Id

    private final OrderId id;
    private OrderStatus status;
    private final CustomerId customerId;
    private List<OrderLine> lines;
    private Money totalAmount;
    private final List<DomainEvent> domainEvents = new ArrayList<>();

    // 불변 생성자, 팩토리 메서드, 도메인 메서드...
    public static Order place(...) { ... }
    public void cancel(String reason) { ... }
    public List<DomainEvent> pullDomainEvents() { ... }
    // @Transient, @PostLoad 없음
}

// 2. JPA Entity (도메인 로직 없음)
@Entity
@Table(name = "orders")
public class OrderJpaEntity {

    @Id
    private String id;  // UUID 문자열

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    private String customerId;

    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true)
    @JoinColumn(name = "order_id")
    private List<OrderLineJpaEntity> lines;

    private BigDecimal totalAmount;
    private String totalCurrency;

    // getter/setter만 (도메인 로직 없음)
    // JPA가 하는 일만 함
}

// 3. Mapper — 양방향 변환
@Component
public class OrderMapper {

    public OrderJpaEntity toJpa(Order domain) {
        OrderJpaEntity entity = new OrderJpaEntity();
        entity.setId(domain.id().value().toString());
        entity.setStatus(domain.status());
        entity.setCustomerId(domain.customerId().value().toString());
        entity.setLines(domain.lines().stream()
            .map(this::toJpa)
            .collect(toList()));
        entity.setTotalAmount(domain.totalAmount().amount());
        entity.setTotalCurrency(domain.totalAmount().currency().name());
        return entity;
    }

    public Order toDomain(OrderJpaEntity entity) {
        return Order.reconstitute(  // 저장소에서 복원하는 별도 팩토리 메서드
            new OrderId(UUID.fromString(entity.getId())),
            entity.getStatus(),
            new CustomerId(Long.parseLong(entity.getCustomerId())),
            entity.getLines().stream().map(this::toDomain).collect(toList()),
            new Money(entity.getTotalAmount(), Currency.valueOf(entity.getTotalCurrency()))
        );
    }
}

// 4. Repository 구현 — Mapper 사용
@Component
public class JpaOrderRepository implements OrderRepository {

    private final OrderJpaEntityRepository jpaRepo;
    private final OrderMapper mapper;

    @Override
    public void save(Order order) {
        OrderJpaEntity entity = mapper.toJpa(order);
        jpaRepo.save(entity);
    }

    @Override
    public Optional<Order> findById(OrderId id) {
        return jpaRepo.findById(id.value().toString())
            .map(mapper::toDomain);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 완전 분리로 얻는 것과 잃는 것

```
얻는 것:
  ① 완전한 도메인 순수성
     Domain 클래스에 JPA import 없음
     @Entity, @Transient, @PostLoad 없음
     DB 변경이 Domain 코드에 영향 없음
  
  ② 테스트 완전 독립
     Domain 테스트: 순수 Java, Spring 없음
     JPA 테스트: Entity만, Domain 로직 없음
  
  ③ 기술 교체 용이
     JPA → MongoDB: Mapper와 Entity만 교체
     Domain 코드 변경 없음

잃는 것:
  ① 클래스 수 증가
     Order → Order + OrderJpaEntity + OrderMapper
     파일 3배, 관리 포인트 증가
  
  ② Mapper 유지 비용
     Domain 필드 추가 → Mapper도 수정
     Mapper 실수 (필드 누락)로 인한 데이터 유실 버그
  
  ③ JPA 더티 체킹 불가
     Domain 객체를 수정해도 JPA가 모름
     항상 명시적 save() 필요
  
  ④ 성능 오버헤드
     Domain → JPA Entity 변환 시 객체 생성

언제 완전 분리가 가치 있는가:
  ✅ 도메인 복잡도가 높아 Domain 순수성이 중요
  ✅ 팀이 커서 Domain과 Infrastructure를 분리된 팀이 관리
  ✅ 기술 변경 가능성이 있음 (JPA → NoSQL 등)
  ✅ 도메인 테스트 속도가 매우 중요한 시스템

과잉인 경우:
  ❌ 소규모 팀 + 단순 CRUD 도메인
  ❌ 비즈니스 로직보다 데이터 관리가 주목적
  ❌ 팀에 DDD 경험이 없어 Mapper 유지 비용 인지 어려움
```

### 2. `reconstitute` 팩토리 메서드 패턴

```java
// 저장소에서 복원하는 별도 팩토리 메서드
// → place()와 다른 경로: 불변식 검증 없이 그냥 복원
public class Order {

    // 새 주문 생성: 불변식 검증 포함
    public static Order place(CustomerId customerId, List<OrderLine> lines, ...) {
        validateLines(lines);  // 검증
        Order order = new Order(new OrderId(), ...);
        order.registerEvent(new OrderPlaced(...));
        return order;
    }

    // DB에서 복원: 이미 저장된 유효한 데이터 → 검증 불필요
    public static Order reconstitute(OrderId id, OrderStatus status,
                                      CustomerId customerId, List<OrderLine> lines,
                                      Money totalAmount) {
        // 검증 없음 (DB에서 나온 데이터는 이미 검증됨)
        return new Order(id, status, customerId, lines, totalAmount);
    }

    private Order(OrderId id, OrderStatus status, CustomerId customerId,
                  List<OrderLine> lines, Money totalAmount) {
        this.id = id;
        this.status = status;
        this.customerId = customerId;
        this.lines = new ArrayList<>(lines);
        this.totalAmount = totalAmount;
    }
}
```

### 3. MapStruct로 Mapper 자동화

```java
// MapStruct: 컴파일 타임에 Mapper 구현체 자동 생성
@Mapper(componentModel = "spring", uses = {MoneyMapper.class})
public interface OrderMapper {

    @Mapping(source = "id.value", target = "id")
    @Mapping(source = "customerId.value", target = "customerId")
    @Mapping(source = "totalAmount.amount", target = "totalAmount")
    @Mapping(source = "totalAmount.currency", target = "totalCurrency")
    OrderJpaEntity toJpa(Order domain);

    @Mapping(target = "id", expression = "java(new OrderId(UUID.fromString(entity.getId())))")
    @Mapping(target = "customerId", expression = "java(new CustomerId(Long.parseLong(entity.getCustomerId())))")
    @Mapping(target = "totalAmount", expression = "java(moneyMapper.toDomain(entity))")
    Order toDomain(OrderJpaEntity entity);
}

// 복잡한 변환은 @Named 메서드로 처리
@Mapper(componentModel = "spring")
public abstract class MoneyMapper {

    @Named("toMoneyDomain")
    public Money toDomain(OrderJpaEntity entity) {
        return new Money(entity.getTotalAmount(), Currency.valueOf(entity.getTotalCurrency()));
    }
}
```

---

## 💻 실전 코드

### 분리 수준별 코드 비교

```java
// 수준 1: 완전 통합 (최소 PI)
@Entity
public class Order {
    @Id @GeneratedValue private Long id;
    @Setter private String status;  // setter 있음
    @Transient private List<DomainEvent> events = new ArrayList<>();
    // JPA + 도메인 혼재
}

// 수준 2: 실용적 타협 (권장 기본값)
@Entity
public class Order {
    @EmbeddedId private OrderId id;
    @Enumerated(EnumType.STRING) private OrderStatus status;
    protected Order() {}  // JPA 기본 생성자만 허용
    public static Order place(...) { ... }  // 팩토리 메서드
    @Transient private final List<DomainEvent> events = new ArrayList<>();
    // JPA 어노테이션 있지만 불변식 보호됨
}

// 수준 3: 완전 분리 (최대 PI)
public class Order {  // JPA 없음
    private final OrderId id;
    private OrderStatus status;
    public static Order place(...) { ... }
    public static Order reconstitute(...) { ... }
}

@Entity class OrderJpaEntity { /* JPA만 */ }
@Component class OrderMapper { /* 변환 */ }
```

---

## 📊 설계 비교

```
완전 통합 vs 실용적 타협 vs 완전 분리:

                완전 통합          실용적 타협        완전 분리
────────────┼─────────────────┼──────────────────┼──────────────────
클래스 수    │ N개               │ N개               │ 3N개 (+ Mapper)
────────────┼─────────────────┼──────────────────┼──────────────────
도메인 순수성 │ 낮음              │ 중간              │ 높음
────────────┼─────────────────┼──────────────────┼──────────────────
테스트 독립성 │ 낮음              │ 중간~높음         │ 높음
────────────┼─────────────────┼──────────────────┼──────────────────
Mapper 비용  │ 없음              │ 없음              │ 높음
────────────┼─────────────────┼──────────────────┼──────────────────
기술 교체   │ 어려움            │ 중간              │ 쉬움
────────────┼─────────────────┼──────────────────┼──────────────────
팀 규모 적합 │ 소규모 빠른 개발  │ 대부분의 팀        │ 대규모, 복잡 도메인
```

---

## ⚖️ 트레이드오프

```
Mapper 유지 비용의 현실:
  "Order에 새 필드 couponId 추가"
  → Order 클래스 수정
  → OrderJpaEntity 수정
  → OrderMapper 수정
  → 테스트 수정
  → 4곳 수정 (실수 가능성 증가)

MapStruct로 비용 절감:
  컴파일 타임 자동 생성 → 필드 누락 시 컴파일 오류
  반복 코드 감소
  단, 복잡한 변환은 여전히 수동

대안: 절충안 — 핵심 Aggregate만 완전 분리
  복잡한 Core Domain Aggregate → 완전 분리
  단순 Supporting Domain → 실용적 타협
  Generic Domain → 통합 (단순 CRUD)
```

---

## 📌 핵심 정리

```
JPA Entity 분리 핵심:

완전 분리의 가치:
  도메인 완전 순수 + 기술 독립 + 테스트 독립
  
완전 분리의 비용:
  Mapper 클래스 + 양방향 변환 유지 + JPA 더티 체킹 불가

@Transient 증식 문제:
  신호: 도메인 클래스에 @Transient가 많아짐
  → Domain과 JPA Entity가 섞인 것
  → 완전 분리 고려

reconstitute 패턴:
  저장소 복원 = 검증 없는 별도 팩토리
  place() ≠ reconstitute()

MapStruct:
  컴파일 타임 Mapper 자동 생성
  필드 누락 → 컴파일 오류 (안전)

선택 기준:
  소규모 + 단순: 실용적 타협
  대규모 + 복잡 Core: 완전 분리 (Core만)
```

---

## 🤔 생각해볼 문제

**Q1.** Mapper에서 `toDomain()` 변환 시 필드를 누락하면 어떤 버그가 발생하는가?

<details>
<summary>해설 보기</summary>

**조용한 데이터 유실(Silent Data Loss) 버그가 발생합니다.**

```java
// 버그가 있는 Mapper
public Order toDomain(OrderJpaEntity entity) {
    return Order.reconstitute(
        new OrderId(entity.getId()),
        entity.getStatus(),
        new CustomerId(entity.getCustomerId()),
        entity.getLines().stream().map(this::toDomain).collect(toList())
        // ← totalAmount를 누락!
    );
}

// 결과:
Order order = orderRepository.findById(orderId).orElseThrow();
order.totalAmount()  // → null!
order.cancel()       // → NullPointerException in cancel's amount calculation
```

**방지 방법:**
1. **MapStruct 사용**: 컴파일 타임에 매핑되지 않은 필드 경고
2. **단위 테스트**: `toDomain(toJpa(domain)).equals(domain)` 왕복 테스트
3. **빌더 패턴**: 모든 필드를 명시적으로 설정 강제

```java
// 왕복 테스트 (Round-Trip Test)
@Test
void mapperRoundTrip() {
    Order original = OrderBuilder.anOrder().paid().build();
    OrderJpaEntity entity = mapper.toJpa(original);
    Order restored = mapper.toDomain(entity);
    
    assertThat(restored.id()).isEqualTo(original.id());
    assertThat(restored.status()).isEqualTo(original.status());
    assertThat(restored.totalAmount()).isEqualTo(original.totalAmount());
    // 모든 필드 검증 → 누락 즉시 발견
}
```

</details>

---

**Q2.** 완전 분리에서 JPA 더티 체킹이 작동하지 않으면 어떻게 변경을 감지하고 저장하는가?

<details>
<summary>해설 보기</summary>

**항상 명시적 `save()`를 호출해야 합니다.**

```java
// 완전 분리에서의 저장 흐름
@Service
@Transactional
public class OrderApplicationService {

    public void cancelOrder(CancelOrderCommand command) {
        // 1. 도메인 객체 조회 (JPA Entity → Domain 변환)
        Order order = orderRepository.findById(command.orderId()).orElseThrow();

        // 2. 도메인 메서드 호출 (Domain 객체 상태 변경)
        order.cancel(command.reason());

        // 3. 명시적 저장 필수 (JPA 더티 체킹 없음)
        orderRepository.save(order);  // ← 반드시 필요!
        // save() 내부: Domain → JPA Entity 변환 → jpaRepo.save()
    }
}
```

**장점**: 변경 의도가 코드에 명시적으로 드러납니다. "저장이 필요하다"는 것이 자명합니다.

**단점**: 실수로 `save()`를 빠뜨리면 변경이 반영되지 않는 버그. 이것이 InMemoryRepository로 테스트하면 발견됩니다 (In-Memory에서도 `save()` 없으면 변경 없음).

</details>

---

**Q3.** 완전 분리된 구조에서 JPQL/Querydsl로 복잡한 조건 쿼리를 사용해야 할 때 어떻게 처리하는가?

<details>
<summary>해설 보기</summary>

**쓰기 측(Domain Repository)과 읽기 측(Query Repository)을 분리합니다.**

```java
// 쓰기 측: Domain Repository (완전 분리)
public interface OrderRepository {
    void save(Order order);
    Optional<Order> findById(OrderId id);
}

// 읽기 측: Query Repository (JPA 직접 사용 허용)
@Repository
public interface OrderQueryRepository {
    // JPA Entity 또는 DTO 직접 반환 (Domain 변환 없음)
    @Query("""
        SELECT new com.example.OrderSummaryDto(
            o.id, o.status, o.totalAmount, c.name
        )
        FROM OrderJpaEntity o
        JOIN CustomerJpaEntity c ON c.id = o.customerId
        WHERE o.customerId = :customerId
          AND o.status IN :statuses
        ORDER BY o.createdAt DESC
        """)
    List<OrderSummaryDto> findSummaryByCustomer(
        @Param("customerId") String customerId,
        @Param("statuses") List<String> statuses
    );
}
```

CQRS 패턴: 쓰기(Command)는 완전 분리된 Domain Repository, 읽기(Query)는 JPA 직접 사용 읽기 전용 Repository. 읽기 측은 PI를 엄격히 지킬 필요가 없습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Persistence Ignorance](./01-persistence-ignorance.md)** | **[홈으로 🏠](../README.md)** | **[다음: Value Object JPA 매핑 ➡️](./03-value-object-jpa.md)**

</div>
