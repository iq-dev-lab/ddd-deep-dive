# Outbox Pattern 완전 분해 — DB 트랜잭션과 이벤트 원자성

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Dual Write 문제란 무엇이고, 왜 이벤트 유실이 발생하는가?
- Outbox 테이블이 어떻게 DB 저장과 이벤트 발행의 원자성을 보장하는가?
- Polling Publisher와 CDC(Debezium) 방식의 차이와 선택 기준은?
- Outbox Pattern 구현 시 멱등성과 중복 처리를 어떻게 설계하는가?
- Spring + JPA 환경에서 Outbox Pattern을 어떻게 실용적으로 구현하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Kafka를 사용해 이벤트를 발행하면 두 가지 독립된 쓰기가 발생한다. DB 저장과 Kafka 발행. 둘은 각기 다른 시스템이어서 하나가 성공해도 다른 하나가 실패할 수 있다. 이것이 Dual Write 문제다.

주문이 DB에 저장됐는데 Kafka 발행에 실패하면 → 포인트 미적립, 재고 미차감, 이메일 미발송. 반대로 Kafka에 발행됐는데 DB 롤백되면 → 없는 주문에 대한 이벤트가 떠돌아다닌다.

Outbox Pattern은 이 두 쓰기를 **하나의 DB 트랜잭션으로 묶어** 원자성을 보장하는 검증된 해법이다.

---

## 😱 흔한 실수 (Before — Dual Write 문제)

```java
// Dual Write 안티패턴 — 두 개의 독립된 쓰기
@Service
@Transactional
public class OrderApplicationService {

    private final KafkaTemplate<String, Object> kafkaTemplate;

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = Order.place(...);
        orderRepository.save(order);  // Write 1: DB 저장

        // Write 2: Kafka 발행 — DB 저장과 원자적으로 보장 불가!
        kafkaTemplate.send("order.events", new OrderPlaced(order.id(), ...));

        return order.id();
    }
}
```

```
실패 시나리오 A: DB 성공 → Kafka 실패
  orderRepository.save(order) → 커밋 성공
  kafkaTemplate.send() → 네트워크 오류!
  
  결과:
    주문은 DB에 있음
    OrderPlaced 이벤트는 Kafka에 없음
    → 포인트 미적립, 재고 미차감, 이메일 미발송
    → "주문했는데 포인트가 안 쌓아요!" 고객 CS

실패 시나리오 B: Kafka 성공 → DB 실패
  kafkaTemplate.send() → 성공 (비동기로 보내짐)
  orderRepository.save(order) → DB 제약 위반으로 예외!
  → 트랜잭션 롤백

  결과:
    주문은 DB에 없음
    OrderPlaced 이벤트는 Kafka에 있음
    → 핸들러가 Order를 조회하지만 없음 → 오류
    → "없는 주문에 대한 이벤트 처리 실패" 로그 폭발

실패 시나리오 C: 서버 장애 (커밋 후 Kafka 발행 전)
  @Transactional
  public OrderId placeOrder(...) {
      orderRepository.save(order);  // 커밋됨
      // 서버 다운!
      kafkaTemplate.send(...);      // 실행 안 됨
  }
  → 가장 발견하기 어려운 유실 케이스
```

---

## ✨ 올바른 접근 (After — Outbox Pattern)

```
Outbox Pattern 핵심 아이디어:
  "DB 저장"과 "이벤트 저장"을 같은 트랜잭션으로 묶는다
  Kafka 발행은 별도 프로세스(Relay)가 담당

트랜잭션 범위:
  ┌────────────────────────────────────────┐
  │  DB Transaction                        │
  │  INSERT INTO orders ...                │
  │  INSERT INTO outbox_messages ...       │  ← 이벤트를 DB에 저장
  └────────────────────────────────────────┘
        ↓ 커밋 완료
  
  Relay Process (별도 스케줄러 또는 CDC):
  SELECT * FROM outbox_messages WHERE status = 'PENDING'
        ↓
  kafkaTemplate.send(...)
        ↓
  UPDATE outbox_messages SET status = 'PUBLISHED'
```

```java
// Outbox 테이블 엔티티
@Entity
@Table(name = "outbox_messages")
public class OutboxMessage {

    @Id
    private UUID id;

    @Column(name = "aggregate_type")
    private String aggregateType;  // "Order"

    @Column(name = "aggregate_id")
    private String aggregateId;    // Order ID

    @Column(name = "event_type")
    private String eventType;      // "OrderPlaced"

    @Column(columnDefinition = "jsonb")
    private String payload;        // JSON 직렬화된 이벤트

    @Enumerated(EnumType.STRING)
    private OutboxStatus status;   // PENDING, PUBLISHED, FAILED

    @Column(name = "created_at")
    private LocalDateTime createdAt;

    public static OutboxMessage from(DomainEvent event, ObjectMapper objectMapper) {
        return new OutboxMessage(
            UUID.randomUUID(),
            event.getClass().getSimpleName().replace("...", ""),
            extractAggregateId(event),
            event.getClass().getSimpleName(),
            serialize(event, objectMapper),
            OutboxStatus.PENDING,
            LocalDateTime.now()
        );
    }
}

// Application Service — Outbox와 함께 저장
@Service
@Transactional
public class OrderApplicationService {

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = orderFactory.create(command);
        orderRepository.save(order);

        // 이벤트를 Kafka 대신 Outbox 테이블에 저장 (같은 트랜잭션!)
        order.pullDomainEvents().stream()
            .map(event -> OutboxMessage.from(event, objectMapper))
            .forEach(outboxRepository::save);

        // → DB 트랜잭션이 커밋되면 Order + OutboxMessage 동시에 저장됨
        // → 트랜잭션 롤백되면 둘 다 없어짐
        // → 원자성 보장!

        return order.id();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Outbox 테이블 구조

```sql
CREATE TABLE outbox_messages (
    id              UUID         PRIMARY KEY DEFAULT gen_random_uuid(),
    aggregate_type  VARCHAR(100) NOT NULL,   -- "Order", "Payment"
    aggregate_id    VARCHAR(100) NOT NULL,   -- Aggregate의 ID 값
    event_type      VARCHAR(200) NOT NULL,   -- "OrderPlaced", "PaymentCompleted"
    payload         JSONB        NOT NULL,   -- 이벤트 JSON
    status          VARCHAR(20)  NOT NULL DEFAULT 'PENDING',  -- PENDING, PUBLISHED, FAILED
    retry_count     INT          NOT NULL DEFAULT 0,
    created_at      TIMESTAMP    NOT NULL DEFAULT NOW(),
    published_at    TIMESTAMP,               -- 발행 완료 시각
    error_message   TEXT                     -- 실패 시 오류 메시지
);

CREATE INDEX idx_outbox_status_created ON outbox_messages(status, created_at)
WHERE status = 'PENDING';  -- PENDING 조회 최적화
```

### 2. Polling Publisher 방식

```
구조:
  DB → (스케줄러 주기적 조회) → Kafka

장점:
  구현 단순 (스케줄러 + JDBC)
  추가 인프라 없음 (Debezium 불필요)
  DB 종류 무관

단점:
  폴링 주기만큼 지연 (1초 주기면 최대 1초 지연)
  고빈도 폴링 시 DB 부하
  분산 환경에서 여러 인스턴스 중복 처리 방지 필요 (락 필요)
```

```java
// Polling Publisher 구현
@Component
@RequiredArgsConstructor
public class OutboxPollingPublisher {

    private final OutboxMessageRepository outboxRepository;
    private final KafkaTemplate<String, String> kafkaTemplate;
    private final ObjectMapper objectMapper;

    @Scheduled(fixedDelay = 1000)  // 1초마다 실행
    @Transactional
    public void publishPendingMessages() {
        // 분산 환경: SELECT FOR UPDATE SKIP LOCKED로 동시 처리 방지
        List<OutboxMessage> pending = outboxRepository.findPendingWithLock(10);  // 한 번에 10개

        for (OutboxMessage message : pending) {
            try {
                String topic = topicFor(message.eventType());
                kafkaTemplate.send(topic, message.aggregateId(), message.payload()).get();
                // .get()으로 동기 확인 — 성공하면 상태 업데이트
                outboxRepository.markPublished(message.id());
            } catch (Exception e) {
                log.error("Outbox 메시지 발행 실패: id={}", message.id(), e);
                outboxRepository.markFailed(message.id(), e.getMessage());
            }
        }
    }
}

// Repository — SELECT FOR UPDATE SKIP LOCKED
public interface OutboxMessageJpaRepository extends JpaRepository<OutboxMessage, UUID> {

    @Lock(LockModeType.PESSIMISTIC_WRITE)
    @QueryHints(@QueryHint(name = "javax.persistence.lock.timeout", value = "0"))
    @Query("""
        SELECT m FROM OutboxMessage m
        WHERE m.status = 'PENDING'
        ORDER BY m.createdAt ASC
        LIMIT :limit
        """)
    List<OutboxMessage> findPendingWithLock(@Param("limit") int limit);
}
```

### 3. CDC (Debezium) 방식

```
구조:
  DB (Write-Ahead Log) → Debezium (CDC) → Kafka

장점:
  DB 커밋과 거의 동시에 이벤트 발행 (지연 최소)
  애플리케이션 코드 변경 불필요 (별도 인프라)
  DB 부하 최소 (WAL 읽기)

단점:
  추가 인프라 (Debezium, Kafka Connect)
  DB가 CDC를 지원해야 함 (PostgreSQL WAL, MySQL binlog)
  운영 복잡도 증가

Debezium 설정 (PostgreSQL):
  {
    "name": "order-outbox-connector",
    "config": {
      "connector.class": "io.debezium.connector.postgresql.PostgresConnector",
      "database.hostname": "localhost",
      "database.port": "5432",
      "database.user": "dbuser",
      "database.dbname": "orderdb",
      "table.include.list": "public.outbox_messages",
      "transforms": "outbox",
      "transforms.outbox.type": "io.debezium.transforms.outbox.EventRouter",
      "transforms.outbox.route.by.field": "aggregate_type",
      "transforms.outbox.table.field.event.type": "event_type",
      "transforms.outbox.table.field.event.payload": "payload"
    }
  }

→ outbox_messages 테이블에 INSERT → Debezium이 WAL에서 감지 → Kafka 토픽으로 라우팅
```

### 4. 선택 기준: Polling vs CDC

```
Polling Publisher가 적합한 경우:
  ✅ 소규모 팀, 단순함 선호
  ✅ 이벤트 발행 지연 1~5초 허용
  ✅ 이벤트 빈도가 낮음 (초당 수백 건 이하)
  ✅ 추가 인프라 운영 부담 없고 싶을 때

CDC(Debezium)가 적합한 경우:
  ✅ 이벤트 발행 지연이 밀리초 수준이어야 할 때
  ✅ 높은 이벤트 빈도 (초당 수천 건 이상)
  ✅ DB 폴링 부하를 피하고 싶을 때
  ✅ 이미 Kafka Connect 인프라가 있을 때

현실적 선택:
  Phase 1: Polling Publisher (단순하게 시작)
  Phase 2: CDC (트래픽 증가 후 전환)
```

### 5. 멱등성 처리

```
Outbox + Kafka = at-least-once 전송
→ 동일 이벤트가 두 번 발행될 수 있음

소비자 멱등성 설계:

방법 1: 이벤트 ID로 중복 체크
  @Transactional
  public void on(OrderPlaced event) {
      if (processedEvents.contains(event.eventId())) {
          log.info("중복 이벤트, 무시: {}", event.eventId());
          return;
      }
      // 처리
      pointWallet.earn(event.totalAmount());
      processedEvents.mark(event.eventId());
  }

방법 2: DB 유니크 제약
  CREATE UNIQUE INDEX ON point_entries(order_id);
  // 동일 order_id로 중복 INSERT 시 DB가 거부

방법 3: 비즈니스 키 기반 멱등성
  // 이미 적립된 주문인지 확인
  public void earn(OrderId orderId, Money amount) {
      boolean alreadyEarned = pointHistory.existsByOrderId(orderId);
      if (alreadyEarned) return;  // 중복 적립 방지
      pointHistory.add(new PointEntry(orderId, amount));
  }
```

---

## 💻 실전 코드

### 완전한 Outbox Pattern 구현

```java
// Outbox 메시지 JPA 엔티티
@Entity
@Table(name = "outbox_messages")
@Getter
public class OutboxMessage {

    @Id
    private UUID id;
    private String aggregateType;
    private String aggregateId;
    private String eventType;

    @Column(columnDefinition = "text")
    private String payload;

    @Enumerated(EnumType.STRING)
    private OutboxStatus status;

    private int retryCount;
    private LocalDateTime createdAt;
    private LocalDateTime publishedAt;

    protected OutboxMessage() {}

    public static OutboxMessage pending(DomainEvent event, ObjectMapper mapper) {
        try {
            return new OutboxMessage(
                UUID.randomUUID(),
                extractAggregateType(event),
                extractAggregateId(event),
                event.getClass().getSimpleName(),
                mapper.writeValueAsString(event),
                OutboxStatus.PENDING,
                0, LocalDateTime.now(), null
            );
        } catch (JsonProcessingException e) {
            throw new EventSerializationException("이벤트 직렬화 실패", e);
        }
    }

    public void markPublished() {
        this.status = OutboxStatus.PUBLISHED;
        this.publishedAt = LocalDateTime.now();
    }

    public void markFailed(String error) {
        this.retryCount++;
        if (this.retryCount >= 3) {
            this.status = OutboxStatus.DEAD_LETTER;
        }
    }
}

// Polling 기반 Relay
@Component
@Slf4j
public class OutboxRelayScheduler {

    private final OutboxMessageRepository outboxRepo;
    private final KafkaTemplate<String, String> kafka;

    @Scheduled(fixedDelay = 500)  // 500ms마다
    public void relay() {
        outboxRepo.findTop20ByStatusOrderByCreatedAtAsc(OutboxStatus.PENDING)
            .forEach(this::publishToKafka);
    }

    @Transactional
    private void publishToKafka(OutboxMessage message) {
        try {
            String topic = resolveKafkaTopic(message.eventType());
            kafka.send(topic, message.aggregateId(), message.payload())
                 .get(5, TimeUnit.SECONDS);  // 동기 확인
            message.markPublished();
            outboxRepo.save(message);
        } catch (Exception e) {
            log.error("Outbox 발행 실패: id={}, type={}", message.id(), message.eventType(), e);
            message.markFailed(e.getMessage());
            outboxRepo.save(message);
        }
    }

    private String resolveKafkaTopic(String eventType) {
        return switch (eventType) {
            case "OrderPlaced", "OrderCancelled" -> "order.events";
            case "PaymentCompleted" -> "payment.events";
            default -> "domain.events";
        };
    }
}
```

---

## 📊 설계 비교

```
Dual Write vs Outbox Pattern:

                Dual Write              Outbox Pattern
────────────┼──────────────────────┼──────────────────────────
원자성       │ ❌ 보장 불가           │ ✅ DB 트랜잭션으로 보장
────────────┼──────────────────────┼──────────────────────────
유실 위험    │ 높음                  │ 매우 낮음 (Relay 재시도)
────────────┼──────────────────────┼──────────────────────────
구현 복잡도  │ 낮음                  │ 중간 (Outbox 테이블 + Relay)
────────────┼──────────────────────┼──────────────────────────
발행 지연    │ 없음 (직접 발행)       │ Polling: ~1초, CDC: ~ms
────────────┼──────────────────────┼──────────────────────────
중복 가능성  │ 낮음                  │ at-least-once → 멱등성 필요

Polling vs CDC:

                Polling Publisher       CDC (Debezium)
────────────┼──────────────────────┼──────────────────────────
지연         │ 폴링 주기 (1~5초)     │ 밀리초
────────────┼──────────────────────┼──────────────────────────
DB 부하      │ 있음 (SELECT)         │ 최소 (WAL 읽기)
────────────┼──────────────────────┼──────────────────────────
인프라       │ 없음                  │ Debezium + Kafka Connect
────────────┼──────────────────────┼──────────────────────────
구현 복잡도  │ 낮음                  │ 높음
────────────┼──────────────────────┼──────────────────────────
적합한 규모  │ 중소 규모              │ 대규모, 고빈도
```

---

## ⚖️ 트레이드오프

```
Outbox Pattern의 비용:
  ① Outbox 테이블 크기 관리
     발행 완료된 메시지 정리 필요 (배치 삭제)
     보관 정책: "발행 후 7일 보관 → 삭제"
  
  ② Relay 프로세스 관리
     Relay 장애 시 이벤트 지연 (유실은 아님)
     Relay 모니터링 필요 (Prometheus + Grafana)
  
  ③ 분산 환경에서 중복 Relay 방지
     여러 인스턴스가 동시에 같은 메시지를 발행하지 않도록
     → SELECT FOR UPDATE SKIP LOCKED 또는 리더 선출

실용적 단순화:
  Spring Boot + @Scheduled 만으로도 Polling Publisher 구현 가능
  프로덕션 전 Outbox Pattern 없이 @TransactionalEventListener로 시작
  → 트래픽이 높아지고 유실이 발생하면 Outbox 도입
```

---

## 📌 핵심 정리

```
Outbox Pattern 핵심:

Dual Write 문제:
  DB 저장 + Kafka 발행 = 두 개의 독립된 쓰기
  하나 성공, 하나 실패 → 데이터 불일치

Outbox 해법:
  DB 저장 + Outbox 저장 = 하나의 트랜잭션
  별도 Relay가 Outbox → Kafka 발행 (재시도 가능)
  → 원자성 보장, 유실 없음

테이블 구조:
  id, aggregate_type, aggregate_id, event_type, payload, status, created_at

구현 방식:
  Polling Publisher: 스케줄러로 주기적 조회 + 발행 (단순)
  CDC (Debezium): WAL 기반 실시간 캡처 (고성능)

멱등성:
  at-least-once 전송 → 중복 가능
  소비자: 이벤트 ID 기반 중복 체크 또는 DB 유니크 제약

운영:
  Dead Letter Queue: 재시도 한도 초과 메시지 보관
  모니터링: PENDING 메시지 수 알림 (지연 감지)
```

---

## 🤔 생각해볼 문제

**Q1.** Outbox 테이블의 `PENDING` 메시지가 갑자기 수천 개로 늘어났다. 원인과 대응 방법은?

<details>
<summary>해설 보기</summary>

**Relay 프로세스 장애 또는 Kafka 장애가 원인입니다.**

```
원인 1: Relay 스케줄러 장애
  - 스케줄러 스레드가 멈춤
  - 서버 재시작으로 스케줄러 중단
  → 대응: Relay 프로세스 헬스체크 + 자동 재시작

원인 2: Kafka 발행 실패
  - Kafka 클러스터 장애
  - 네트워크 오류
  → Relay가 실패를 기록하고 재시도
  → 대응: Kafka 클러스터 복구 → Relay 재시도로 자동 해소

원인 3: 직렬화 실패
  - 이벤트 직렬화 버그로 FAILED 상태로 쌓임
  → 코드 수정 후 FAILED 메시지를 PENDING으로 재설정

모니터링 설정:
  SELECT COUNT(*) FROM outbox_messages WHERE status = 'PENDING';
  → 임계값(예: 100개) 초과 시 슬랙 알림

재처리:
  UPDATE outbox_messages SET status = 'PENDING', retry_count = 0
  WHERE status = 'FAILED' AND created_at > NOW() - INTERVAL '1 hour';
  → 최근 1시간 실패 메시지 재처리
```

</details>

---

**Q2.** 같은 주문 이벤트가 두 번 발행됐을 때 포인트 적립 핸들러가 두 번 실행되는 것을 DB 레벨에서 방지하는 방법은?

<details>
<summary>해설 보기</summary>

**DB 유니크 제약이 가장 안전하고 단순한 방법입니다.**

```sql
-- 포인트 이력 테이블: order_id는 한 번만 나타날 수 있음
CREATE TABLE point_entries (
    id          BIGSERIAL    PRIMARY KEY,
    customer_id BIGINT       NOT NULL,
    order_id    UUID         NOT NULL,
    amount      DECIMAL      NOT NULL,
    type        VARCHAR(20)  NOT NULL,  -- EARN, SPEND
    created_at  TIMESTAMP    NOT NULL DEFAULT NOW(),
    CONSTRAINT uk_point_entry_order UNIQUE (order_id, type)
    -- 같은 주문에 대한 같은 타입(EARN)은 한 번만 허용
);
```

```java
// 핸들러: DB 유니크 제약으로 자동 멱등성
@EventListener
@Transactional
public void on(OrderPlaced event) {
    try {
        PointEntry entry = new PointEntry(
            event.customerId(),
            event.orderId(),    // ← 유니크 제약 키
            PointAmount.of(event.totalAmount()),
            PointType.EARN
        );
        pointEntryRepository.save(entry);
        // 중복 시 DataIntegrityViolationException 발생
    } catch (DataIntegrityViolationException e) {
        // 이미 처리된 이벤트 → 무시
        log.info("이미 처리된 주문의 포인트 적립 시도 무시: orderId={}", event.orderId());
    }
}
```

이 방법의 장점: 별도 처리 이력 테이블 불필요, DB가 원자적으로 보장.

</details>

---

**Q3.** Outbox Pattern을 `@TransactionalEventListener`와 함께 사용할 때, Spring Data의 `AbstractAggregateRoot`를 쓰면 Outbox에 저장이 되는가?

<details>
<summary>해설 보기</summary>

**`AbstractAggregateRoot`는 Outbox를 자동으로 저장하지 않습니다. 수동 구현이 필요합니다.**

`AbstractAggregateRoot`의 동작:
- `save()` 후 Spring Data가 `@DomainEvents`로 수집된 이벤트를 `ApplicationEventPublisher`로 발행
- 이것은 Spring 내부 이벤트 시스템을 사용
- Outbox 테이블에는 저장하지 않음

**Outbox와 함께 사용하려면:**

```java
// 방법 1: Application Service에서 직접 Outbox 저장
@Service
@Transactional
public class OrderApplicationService {

    public OrderId placeOrder(PlaceOrderCommand command) {
        Order order = orderFactory.create(command);
        orderRepository.save(order);  // AbstractAggregateRoot면 여기서 이벤트 발행됨

        // AbstractAggregateRoot가 이벤트를 발행하기 전에 Outbox에 저장
        // → order.pullDomainEvents() 대신 별도 수집 방법 필요

        // 방법: AbstractAggregateRoot를 쓰지 말고 커스텀 AggregateRoot 사용
        order.pullDomainEvents()
            .stream()
            .map(e -> OutboxMessage.pending(e, objectMapper))
            .forEach(outboxRepository::save);

        return order.id();
    }
}

// 방법 2: @TransactionalEventListener에서 Outbox 저장
// AbstractAggregateRoot의 이벤트 → @TransactionalEventListener → Outbox 저장
@TransactionalEventListener(phase = TransactionPhase.BEFORE_COMMIT)
public void onOrderPlaced(OrderPlaced event) {
    outboxRepository.save(OutboxMessage.pending(event, objectMapper));
    // 같은 트랜잭션에서 Outbox 저장 (BEFORE_COMMIT)
}
```

결론: Outbox Pattern을 완전히 제어하려면 `AbstractAggregateRoot` 대신 커스텀 `AggregateRoot`를 사용하는 것이 명확합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 이벤트 발행 패턴](./02-event-publishing-patterns.md)** | **[홈으로 🏠](../README.md)** | **[다음: Saga Pattern ➡️](./04-saga-pattern.md)**

</div>
