# Aggregate 설계 — 일관성 경계를 결정하는 원칙

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Aggregate Root를 통해서만 내부 객체를 변경해야 하는 이유는?
- Aggregate 경계는 무엇을 기준으로 결정하는가?
- 경계 설정 오류의 실제 증상은 어떻게 나타나는가?
- Aggregate 간 참조는 왜 ID로만 해야 하는가?
- 불변식(Invariant)이 Aggregate 경계를 결정하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

데이터베이스 외래키를 그대로 객체 참조로 옮기면 어떻게 되는가? `Order`가 `Customer`, `Product`, `Coupon`을 직접 참조하고, 이 모든 객체가 하나의 거대한 그래프를 형성한다. 어느 한 곳을 바꾸면 다른 곳이 영향받는다. 트랜잭션이 여러 테이블에 걸쳐 락을 잡는다. "주문을 저장한다"는 단순한 행위가 전체 객체 그래프를 로드해야 가능해진다.

Aggregate는 이 문제를 해결하는 **일관성 경계**다. Aggregate Root만이 외부에서 접근 가능한 진입점이고, Aggregate 내부의 모든 변경은 Root를 통해서만 이루어진다. 이 경계 안에서는 항상 비즈니스 불변식이 보장된다.

---

## 😱 흔한 실수 (Before — 경계 없는 객체 그래프)

```java
// 경계 없는 객체 그래프 — 외부에서 내부를 직접 조작
@Entity
public class Order {
    @OneToMany(cascade = CascadeType.ALL)
    private List<OrderLine> lines;

    public List<OrderLine> getLines() {
        return lines;  // 외부에서 직접 수정 가능!
    }
}

// 문제 코드: Aggregate 경계를 우회해 직접 조작
@Service
public class OrderService {

    public void addItem(Long orderId, Long productId, int quantity) {
        Order order = orderRepository.findById(orderId).orElseThrow();
        Product product = productRepository.findById(productId).orElseThrow();

        // 불변식 우회! OrderLine을 직접 추가
        OrderLine line = new OrderLine(product, quantity);
        order.getLines().add(line);  // ← Order의 불변식 검증 없이 직접 추가

        // 불변식이 깨진 상태:
        // - 주문 항목 수 최대 제한 검사 없음
        // - 동일 상품 중복 항목 허용
        // - 총 금액 재계산 안 됨
    }
}
```

```
구체적 결과:
  주문 최대 10개 항목 제한 → 직접 add()로 11개 추가 가능
  총 금액이 실제 항목 합계와 다름 (재계산 안 됨)
  동일 상품이 두 줄로 나뉘어 저장

  버그 보고: "주문 금액이 이상합니다"
  원인 추적: OrderService의 add() → ShoppingCartService의 merge() → ...
  → 불변식을 깬 지점이 여러 곳에 분산됨
```

---

## ✨ 올바른 접근 (After — Aggregate Root를 통한 접근)

```java
public class Order {  // Aggregate Root

    private OrderId id;
    private OrderStatus status;
    private List<OrderLine> lines = new ArrayList<>();  // 내부 컬렉션
    private Money totalAmount;
    private static final int MAX_LINES = 10;

    // 외부는 Root의 메서드를 통해서만 변경
    public void addLine(ProductId productId, int quantity, Money unitPrice) {
        // 불변식 검증: Root가 책임짐
        if (lines.size() >= MAX_LINES) {
            throw new OrderLimitExceededException("주문 항목은 최대 " + MAX_LINES + "개");
        }
        if (status != OrderStatus.DRAFT) {
            throw new OrderNotModifiableException("접수된 주문은 항목을 변경할 수 없습니다");
        }

        // 동일 상품 중복 처리
        Optional<OrderLine> existing = findLineByProduct(productId);
        if (existing.isPresent()) {
            existing.get().increaseQuantity(quantity);
        } else {
            lines.add(new OrderLine(productId, quantity, unitPrice));
        }

        // 총 금액 재계산 (불변식 유지)
        recalculateTotalAmount();
    }

    public void removeLine(ProductId productId) {
        boolean removed = lines.removeIf(l -> l.productId().equals(productId));
        if (!removed) throw new OrderLineNotFoundException(productId);
        recalculateTotalAmount();
    }

    // 외부에는 불변 뷰만 제공
    public List<OrderLine> lines() {
        return Collections.unmodifiableList(lines);
    }

    private void recalculateTotalAmount() {
        this.totalAmount = lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO_KRW, Money::add);
    }

    private Optional<OrderLine> findLineByProduct(ProductId productId) {
        return lines.stream().filter(l -> l.productId().equals(productId)).findFirst();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Aggregate의 핵심 원칙 4가지

```
원칙 1: 루트를 통한 접근 (Aggregate Root만 외부 참조 가능)
  외부 → Order (Root) → OrderLine (내부)
  외부가 OrderLine을 직접 참조하거나 수정할 수 없음
  → 모든 변경은 Order의 메서드를 통해서만

원칙 2: 일관성 경계 (Aggregate 내 불변식은 항상 보장)
  Aggregate Root의 모든 메서드는 실행 후
  Aggregate 내부 불변식이 참임을 보장
  → addLine() 후 totalAmount = sum(lines)는 항상 참

원칙 3: ID 참조 (Aggregate 간 참조는 ID만)
  Order는 Customer를 직접 참조하지 않음
  Order.customerId (CustomerId VO) → Customer Aggregate를 ID로만 참조
  → Aggregate 경계 보호, 로딩 범위 제한

원칙 4: 트랜잭션 단위 (하나의 트랜잭션 = 하나의 Aggregate)
  하나의 요청에서 두 Aggregate를 동시에 수정하면
  → 설계 재검토 신호
  → 이벤트 기반 Eventually Consistent로 분리 고려
```

### 2. 불변식이 Aggregate 경계를 결정한다

```
불변식(Invariant): "항상 참이어야 하는 비즈니스 규칙"

불변식을 찾아라 → Aggregate 경계가 보인다:

주문 도메인의 불변식:
  ① "총 금액 = 모든 항목의 소계 합" → Order + OrderLine이 같은 Aggregate
  ② "주문 항목은 최대 10개" → OrderLine이 Order를 통해서만 추가됨
  ③ "PAID 상태에서는 항목 변경 불가" → Order가 상태와 항목을 함께 관리

이 불변식들이 Order와 OrderLine이 같은 Aggregate임을 결정:
  불변식이 두 객체에 걸쳐 있고, 함께 보장해야 함
  → 같은 Aggregate Root(Order) 아래에 배치

반면 Customer와 Order의 관계:
  "고객의 주문 수 = 주문 목록의 개수" → 이런 불변식이 있는가?
  → 이런 강한 불변식이 없음
  → Customer와 Order는 다른 Aggregate
  → Order.customerId로 ID만 참조

불변식 발견 방법:
  Q: "이 두 객체 중 하나가 변하면, 다른 하나도 반드시 일관되게 변해야 하는가?"
  → YES → 같은 Aggregate
  → NO  → 다른 Aggregate (Eventually Consistent)
```

### 3. 경계 오류의 증상

```
증상 1: 여러 Aggregate를 하나의 트랜잭션에서 수정
  @Transactional
  public void placeOrder(Long customerId, List<Item> items) {
      Customer customer = customerRepository.findById(customerId).orElseThrow();
      customer.decreasePoint(items);    // Customer Aggregate 수정
      Order order = Order.place(...);
      orderRepository.save(order);       // Order Aggregate 수정
      inventoryRepository.decreaseStock(items); // Inventory Aggregate 수정
  }
  → 세 Aggregate가 하나의 트랜잭션에 묶임
  → 하나라도 실패하면 전체 롤백 (분산 시 문제)
  → 락 경합 증가

증상 2: Aggregate 내부 컬렉션을 외부에서 직접 수정
  order.getLines().add(new OrderLine(...));  // 불변식 우회
  customer.getAddresses().remove(address);  // Customer가 모름

증상 3: Aggregate 경계를 넘는 연쇄 로딩
  Order를 로드하면 Customer, Product, Coupon, Shipping이 모두 로드됨
  → N+1 문제 또는 거대한 JOIN 쿼리

증상 4: Repository가 하나의 Aggregate에 속하지 않는 객체를 저장
  orderLineRepository.save(line);  // OrderLine의 별도 Repository
  → OrderLine은 Order 없이 독립 저장 불가
  → Order.save()가 내부 OrderLine을 함께 저장해야 함
```

### 4. 올바른 Aggregate 경계 결정 과정

```
Step 1: 도메인 이벤트와 불변식 목록 작성
  주문 취소 시:
    - 모든 항목 취소됨
    - 총 금액 0원으로 변경 (또는 환불 금액 계산)
    - 취소 사유 기록
  → OrderLine들이 Order와 함께 변해야 함 → 같은 Aggregate

Step 2: "같이 변해야 하는가?" 질문
  Order ↔ OrderLine: "항목이 추가되면 총 금액도 바뀐다" → 같이 변함 → 같은 Aggregate
  Order ↔ Customer: "주문이 추가됐다고 고객 정보가 바뀌진 않는다" → 따로 변함 → 다른 Aggregate
  Order ↔ Inventory: "주문 취소 시 재고는 Eventually Consistent로 복원" → 다른 Aggregate

Step 3: 경계 검증 — "하나의 트랜잭션으로 충분한가?"
  Order 저장 한 번으로 불변식이 모두 보장되는가?
  → YES → 경계가 적절
  → NO (다른 Aggregate도 함께 저장해야 함) → 설계 재검토

Step 4: 크기 검증 — 작을수록 좋다
  "이 Aggregate를 저장할 때 무엇이 함께 로드되어야 하는가?"
  → 너무 많은 것이 로드된다면 경계가 너무 큰 것
```

---

## 💻 실전 코드

### Aggregate 경계 표현 — JPA 매핑

```java
@Entity
@Table(name = "orders")
public class Order {

    @EmbeddedId
    private OrderId id;

    @Enumerated(EnumType.STRING)
    private OrderStatus status;

    // OrderLine: Order 내부 Entity (Aggregate 내부)
    @OneToMany(cascade = CascadeType.ALL, orphanRemoval = true, fetch = FetchType.LAZY)
    @JoinColumn(name = "order_id")
    private List<OrderLine> lines = new ArrayList<>();

    // Customer: 다른 Aggregate → ID로만 참조
    @Embedded
    private CustomerId customerId;  // 객체 참조 아님

    // Coupon: 다른 Aggregate → ID로만 참조
    @Embedded
    private CouponId appliedCouponId;  // 객체 참조 아님

    // 외부 접근 차단
    public List<OrderLine> lines() {
        return Collections.unmodifiableList(lines);
    }

    // 내부 변경은 Root 메서드를 통해서만
    public void addLine(ProductId productId, int quantity, Money unitPrice) {
        validateCanModify();
        validateMaxLines();
        mergeOrAddLine(productId, quantity, unitPrice);
        recalculateTotalAmount();
    }
}

@Entity
@Table(name = "order_lines")
public class OrderLine {

    @EmbeddedId
    private OrderLineId id;

    @Embedded
    private ProductId productId;  // Product Aggregate → ID로만 참조

    private int quantity;

    @Embedded
    private Money unitPrice;

    // 외부에서 직접 생성 불가 — Order.addLine()을 통해서만
    // (패키지 가시성 또는 내부 팩토리 메서드 활용)
    OrderLine(ProductId productId, int quantity, Money unitPrice) {
        this.id = new OrderLineId();
        this.productId = productId;
        this.quantity = quantity;
        this.unitPrice = unitPrice;
    }

    // Root 내부에서만 수정 허용 (package-private)
    void increaseQuantity(int additional) {
        if (additional <= 0) throw new IllegalArgumentException();
        this.quantity += additional;
    }

    public Money subtotal() {
        return unitPrice.multiply(quantity);
    }
}
```

---

## 📊 설계 비교

```
경계 없는 객체 그래프 vs 올바른 Aggregate:

                경계 없는 그래프         올바른 Aggregate
────────────┼──────────────────────┼──────────────────────────
불변식 보장  │ ❌ 외부에서 우회 가능   │ ✅ Root 메서드만 허용
────────────┼──────────────────────┼──────────────────────────
로딩 범위   │ 객체 그래프 전체       │ Aggregate만 로드
────────────┼──────────────────────┼──────────────────────────
트랜잭션    │ 여러 테이블 락 동시    │ Aggregate 테이블만 락
────────────┼──────────────────────┼──────────────────────────
테스트      │ 전체 그래프 구성 필요  │ Aggregate 단독 테스트
────────────┼──────────────────────┼──────────────────────────
버그 추적   │ 불변식 훼손 지점 분산  │ Root 메서드에서 집중 발견
────────────┼──────────────────────┼──────────────────────────
Repository  │ 객체마다 별도          │ Root 하나만 (Rule)
```

---

## ⚖️ 트레이드오프

```
Aggregate 경계의 어려움:
  ① 경계가 너무 작으면 Eventually Consistent 복잡도 증가
     Order와 OrderLine을 분리하면 → 항목 추가 시 두 트랜잭션 → Saga 필요
  
  ② 경계가 너무 크면 락 경합, 느린 로딩
     Order가 Customer, Product, Inventory를 포함하면
     → 주문 목록 조회 시 모든 고객, 상품, 재고 로드

  ③ 도메인 전문가와 합의 필요
     "주문과 결제 정보는 같은 Aggregate인가?"
     → 기술 결정이 아닌 도메인 불변식 분석이 기준

현실적 접근:
  작게 시작 → 불변식 위반이 발생하면 크게 조정
  큰 경계로 시작 → 락 경합/성능 문제 발생 시 분리
  대부분의 경우 "작은 Aggregate + Eventually Consistent"가 권장됨
```

---

## 📌 핵심 정리

```
Aggregate 설계 핵심:

Aggregate = 일관성 경계
  내부: 항상 불변식이 보장됨
  경계: Root를 통해서만 접근
  단위: 하나의 트랜잭션으로 저장

경계 결정 기준:
  "이 두 객체가 항상 함께 일관성이 보장되어야 하는가?"
  → YES: 같은 Aggregate
  → NO:  다른 Aggregate (ID로 참조, Eventually Consistent)

Root의 책임:
  불변식 검증 및 보장
  내부 객체 접근 통제
  도메인 이벤트 발행

경계 오류 증상:
  여러 Aggregate를 하나의 트랜잭션에서 수정
  내부 컬렉션을 외부에서 직접 수정
  거대한 객체 그래프 로딩
  Aggregate 내부 객체의 별도 Repository 존재
```

---

## 🤔 생각해볼 문제

**Q1.** 게시글(Post)과 댓글(Comment)의 관계를 Aggregate로 설계할 때, 어떤 경우에 같은 Aggregate이고 어떤 경우에 다른 Aggregate인가?

<details>
<summary>해설 보기</summary>

**불변식의 존재 여부로 결정됩니다.**

**같은 Aggregate가 적합한 경우:**
- "게시글당 댓글은 최대 100개" — Post와 Comment 사이에 불변식 존재
- "게시글이 삭제되면 댓글도 삭제" — 강한 생명주기 의존

```java
public class Post {  // Aggregate Root
    private List<Comment> comments;
    private static final int MAX_COMMENTS = 100;

    public void addComment(Comment comment) {
        if (comments.size() >= MAX_COMMENTS) throw new CommentLimitException();
        comments.add(comment);
    }
}
```

**다른 Aggregate가 적합한 경우 (더 일반적):**
- 댓글이 수백만 개 → Post 로드 시 모든 댓글 로드 = 성능 문제
- 댓글이 독립적으로 수정/삭제됨
- 댓글의 좋아요가 게시글 불변식에 영향 없음

```java
public class Post {  // Aggregate Root — 댓글 포함 않음
    private PostId id;
    // Comment를 포함하지 않음
}

public class Comment {  // 별도 Aggregate Root
    private CommentId id;
    private PostId postId;  // Post를 ID로만 참조
}
```

"댓글 수 = Post.commentCount"가 불변식이라면 → Post에 commentCount 필드만 유지하고 Comment는 별도 Aggregate로 분리하는 절충안도 가능합니다.

</details>

---

**Q2.** "주문 취소 시 재고를 즉시 복원해야 한다"는 요구사항이 있다. Order와 Inventory를 같은 Aggregate로 묶어야 하는가?

<details>
<summary>해설 보기</summary>

**같은 Aggregate로 묶으면 안 됩니다. 이벤트 기반 Eventually Consistent가 올바른 접근입니다.**

Order와 Inventory를 같은 Aggregate로 묶으면:
- 주문 조회 시 모든 재고 정보 로드 필요
- 주문 저장 시 재고 테이블도 락
- 재고 서비스가 별도로 운영될 때 분리 불가

**올바른 설계:**
```java
// Order Aggregate
public class Order {
    public void cancel(String reason) {
        this.status = OrderStatus.CANCELLED;
        this.events.add(new OrderCancelled(this.id, this.lines));
        // → 이벤트를 통해 Inventory에게 알림
    }
}

// Inventory Aggregate (별도 트랜잭션)
@EventListener
public void on(OrderCancelled event) {
    event.lines().forEach(line -> {
        Inventory inventory = inventoryRepository.findByProduct(line.productId()).orElseThrow();
        inventory.restore(line.quantity());  // 재고 복원
        inventoryRepository.save(inventory);
    });
}
```

"즉시"의 의미: 비즈니스에서 "즉시"는 보통 "수 초 이내"를 의미합니다. 이벤트 처리가 수 ms~수 초 내에 처리된다면 비즈니스 요건을 충족합니다. 진정한 강한 일관성(동일 트랜잭션)이 필요한지 도메인 전문가와 확인이 필요합니다.

</details>

---

**Q3.** OrderLine에 별도 Repository(`OrderLineRepository`)를 만들어야 할 상황이 있는가?

<details>
<summary>해설 보기</summary>

**일반적으로 없습니다. 있다면 설계 재검토가 필요합니다.**

DDD 원칙: **Repository는 Aggregate Root 단위로만 존재합니다.** `OrderLine`은 `Order` Aggregate의 내부 Entity이므로, `OrderLineRepository`는 Aggregate 경계를 위반합니다.

**예외적으로 정당화될 수 있는 경우:**
- 대용량 주문 시스템에서 특정 항목만 조회하는 읽기 전용 쿼리

```java
// 허용: 읽기 전용 DAO (CQRS 쿼리 측)
@Repository
public interface OrderLineReadRepository {
    // 쓰기 없음, 조회만
    @Query("SELECT ol FROM OrderLine ol WHERE ol.productId = :productId")
    List<OrderLineDto> findByProduct(@Param("productId") Long productId);
}
```

읽기 모델(Query Model)에서의 사용은 CQRS 관점에서 허용됩니다. 하지만 `orderLineRepository.save(line)`처럼 직접 저장하는 것은 허용하지 않아야 합니다.

</details>

---

<div align="center">

**[⬅️ 이전: Value Object 설계 원칙](./02-value-object-design.md)** | **[홈으로 🏠](../README.md)** | **[다음: Aggregate 크기 결정 ➡️](./04-aggregate-size.md)**

</div>
