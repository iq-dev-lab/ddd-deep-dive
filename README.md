<div align="center">

# 🏛️ DDD Deep Dive

**"엔티티와 서비스를 만드는 것과, 도메인의 불변식(Invariant)을 보호하는 객체 설계를 아는 것은 다르다"**

<br/>

> *"`@Entity` 붙이고 `JpaRepository` 상속하면 DDD겠지 — 와 — 왜 Aggregate Root를 통해서만 내부 객체를 변경해야 하는지, 왜 Value Object는 불변이어야 하는지, 왜 Repository는 Aggregate 단위인지 아는 것의 차이를 만드는 레포"*

Bounded Context가 왜 팀과 코드를 동시에 분리하는지, Aggregate가 왜 일관성 경계인지, Domain Event가 어떻게 결합도를 낮추는지, Outbox Pattern이 왜 원자성 문제의 유일한 해법인지까지  
**왜 이렇게 설계해야 하는가** 라는 질문으로 DDD의 내부 원리를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![DDD](https://img.shields.io/badge/DDD-Eric_Evans-4A90D9?style=flat-square&logo=bookstack&logoColor=white)](https://dddcommunity.org/)
[![Spring](https://img.shields.io/badge/Spring_Data_JPA-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://docs.spring.io/spring-data/jpa/docs/current/reference/html/)
[![Docs](https://img.shields.io/badge/Docs-43개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

DDD에 관한 자료는 넘쳐납니다. 하지만 대부분은 **용어 소개**에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "Aggregate는 일관성 경계입니다" | Aggregate Root를 통하지 않고 내부에 직접 접근했을 때 어떤 불변식이 우회되고, 어떤 시나리오에서 데이터 정합성이 깨지는가 |
| "Value Object는 불변이어야 합니다" | `Money`를 Entity로 만들었을 때 공유 인스턴스가 어떻게 버그를 만들고, `equals()`가 왜 ID 기반이 아닌 값 기반이어야 하는가 |
| "Bounded Context를 나누세요" | Order가 배송 컨텍스트와 결제 컨텍스트에서 왜 다른 모델을 가져야 하는가, 컨텍스트 경계가 너무 작거나 너무 클 때 발생하는 실제 문제 |
| "Domain Event를 쓰세요" | `OrderPlaced` 이벤트 발행과 DB 저장을 어떻게 원자적으로 처리하는가, Outbox Pattern 없이 Kafka만 쓰면 어떤 장애 시나리오가 생기는가 |
| "@Entity 붙이면 됩니다" | JPA Entity와 DDD Entity가 같은 클래스일 때 영속성 관심사가 도메인 모델을 어떻게 오염시키는가, Persistence Ignorance가 실제로 의미하는 것 |
| "Repository는 데이터 접근 레이어입니다" | 테이블당 Repository를 만들었을 때 Aggregate 불변식이 어떻게 깨지는가, 왜 Repository는 컬렉션처럼 동작해야 하는가 |
| 이론 나열 | Anemic vs Rich Domain Model Before/After 코드 비교 + Spring + JPA + Kafka 실험 환경 + 불변식 단위 테스트 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Ch1](https://img.shields.io/badge/🔹_Ch1-DDD가_해결하는_문제-4A90D9?style=for-the-badge)](./ddd-philosophy/01-what-ddd-solves.md)
[![Ch2](https://img.shields.io/badge/🔹_Ch2-서브도메인_분류-4A90D9?style=for-the-badge)](./strategic-design/01-subdomain-classification.md)
[![Ch3](https://img.shields.io/badge/🔹_Ch3-Entity_vs_Value_Object-4A90D9?style=for-the-badge)](./tactical-design/01-entity-vs-value-object.md)
[![Ch4](https://img.shields.io/badge/🔹_Ch4-Domain_Event_결합도-4A90D9?style=for-the-badge)](./domain-events/01-domain-event-decoupling.md)
[![Ch5](https://img.shields.io/badge/🔹_Ch5-Persistence_Ignorance-4A90D9?style=for-the-badge)](./spring-jpa/01-persistence-ignorance.md)
[![Ch6](https://img.shields.io/badge/🔹_Ch6-전자상거래_도메인_분석-4A90D9?style=for-the-badge)](./real-world-project/01-domain-analysis.md)
[![Ch7](https://img.shields.io/badge/🔹_Ch7-Anemic_Domain_Model-4A90D9?style=for-the-badge)](./antipatterns/01-anemic-domain-model.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: DDD란 무엇인가 — 설계의 철학

> **핵심 질문:** DDD는 어떤 문제를 해결하는가? Ubiquitous Language가 코드 구조를 결정한다는 것이 실제로 어떤 의미인가? 모든 프로젝트에 DDD가 답인가?

<details>
<summary><b>소프트웨어 복잡도의 본질부터 레이어드 아키텍처 비교까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. DDD가 해결하는 문제 — 소프트웨어 복잡도의 본질](./ddd-philosophy/01-what-ddd-solves.md) | 기술 중심 설계와 도메인 중심 설계의 차이, Anemic Domain Model이 만드는 Service 비대화 문제, 복잡도가 올라갈수록 트랜잭션 스크립트 방식이 무너지는 이유 |
| [02. Ubiquitous Language — 언어가 코드 구조를 결정한다](./ddd-philosophy/02-ubiquitous-language.md) | 개발자와 도메인 전문가가 같은 언어를 쓰지 않을 때 생기는 번역 비용, 잘못된 언어 선택이 실제 버그로 이어지는 시나리오, Glossary 작성과 코드 명명 일치 방법 |
| [03. Strategic Design vs Tactical Design — 큰 그림부터 시작하는 이유](./ddd-philosophy/03-strategic-vs-tactical.md) | Big Picture(Context Map)를 먼저 그리지 않고 전술 설계부터 시작했을 때 생기는 Aggregate 경계 오류, 전략과 전술 도구를 언제 꺼내는지 판단 기준 |
| [04. DDD 적용 판단 기준 — 도메인 복잡도에 따른 아키텍처 선택](./ddd-philosophy/04-when-to-apply-ddd.md) | 단순 CRUD에 DDD를 적용했을 때의 과잉 복잡도 비용, 도메인 복잡도 측정 방법(불변식 수, 상태 전이, 이해관계자 수), DDD 적용 비용과 편익 계산 |
| [05. DDD와 레이어드 아키텍처 비교 — Controller → Service → Repository의 문제점](./ddd-philosophy/05-ddd-vs-layered.md) | 전통 레이어드 구조에서 도메인 로직이 Service로 흘러가는 이유, 도메인 레이어가 분리됐을 때 테스트가 쉬워지는 이유, Hexagonal Architecture와의 관계 |

</details>

<br/>

### 🔹 Chapter 2: Strategic Design — 큰 그림 그리기

> **핵심 질문:** Bounded Context 경계를 어떻게 찾는가? 왜 같은 "Order"가 컨텍스트마다 다른 모델을 가져야 하는가? Anticorruption Layer는 언제 필요한가?

<details>
<summary><b>서브도메인 분류부터 마이크로서비스 분리 기준까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 서브도메인 분류 — Core / Supporting / Generic](./strategic-design/01-subdomain-classification.md) | Core Domain에 집중해야 하는 전략적 이유, Supporting과 Generic Subdomain에 내부 자원을 낭비하면 생기는 기회비용, 서브도메인 분류로 팀 우선순위를 결정하는 방법 |
| [02. Bounded Context 완전 분해 — 경계를 결정하는 기준](./strategic-design/02-bounded-context.md) | 언어의 경계와 팀의 경계가 Context를 결정하는 이유, Order가 배송/결제 Context에서 다른 속성과 행동을 갖는 이유, 하나의 개념이 Context마다 다른 모델을 갖는 것이 일관성을 높이는 방식 |
| [03. Ubiquitous Language 실전 — 컨텍스트마다 언어가 다를 수 있다](./strategic-design/03-ubiquitous-language-practice.md) | 같은 단어가 컨텍스트마다 다른 의미를 가질 때 충돌이 설계 문제의 신호인 이유, 언어 충돌을 발견하고 Context 경계를 수정하는 과정, Glossary가 Context 경계의 문서화가 되는 방법 |
| [04. Context Map 패턴 완전 분해 — 팀 관계가 통합 패턴을 결정한다](./strategic-design/04-context-map-patterns.md) | Shared Kernel / Customer-Supplier / Conformist / Anticorruption Layer / Open Host Service / Published Language / Separate Ways 7가지 패턴의 의미, 팀 간 권력 관계가 어떤 패턴을 선택하는지 결정하는 이유 |
| [05. Anticorruption Layer — 외부 모델의 오염을 막는 번역기](./strategic-design/05-anticorruption-layer.md) | 외부 시스템/레거시 API 응답이 도메인 모델을 오염시키는 구체적 시나리오, ACL이 Translator로 동작하는 구조, Spring에서 ACL을 구현하는 Mapper/Adapter 패턴 |
| [06. 이벤트 스토밍 — 도메인 전문가와 함께 컨텍스트 발견하기](./strategic-design/06-event-storming.md) | Command / Event / Aggregate / Policy 스티커로 도메인 흐름을 시각화하는 워크숍 방법, 이벤트 스토밍 결과물이 Bounded Context 경계 초안이 되는 과정, 원격 이벤트 스토밍 도구와 실전 팁 |
| [07. Bounded Context와 마이크로서비스 — Context ≠ 서비스](./strategic-design/07-context-vs-microservice.md) | Context 경계가 정리되기 전에 서비스를 분리하면 분산 모놀리스가 되는 이유, 하나의 Context가 여러 서비스가 되는 경우와 하나의 서비스에 여러 Context가 공존하는 경우의 트레이드오프 |

</details>

<br/>

### 🔹 Chapter 3: Tactical Design — 도메인 모델 설계

> **핵심 질문:** Aggregate가 왜 일관성 경계인가? Value Object를 불변으로 만드는 것이 왜 버그를 막는가? Repository가 Aggregate 단위여야 하는 이유는?

<details>
<summary><b>Entity / Value Object 구분부터 Domain Event 설계까지 (8개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Entity vs Value Object — 식별자 vs 값으로 구분](./tactical-design/01-entity-vs-value-object.md) | ID로 구분하는 Entity와 값이 같으면 동일한 Value Object의 핵심 차이, `Money` / `Address` / `Email`을 Value Object로 만들어야 하는 이유, `equals()` / `hashCode()` 구현 방식의 차이와 영향 |
| [02. Value Object 설계 원칙 — 불변성과 자기 검증](./tactical-design/02-value-object-design.md) | 불변성이 공유 인스턴스 버그를 막는 원리, Value Object가 생성 시점에 스스로를 검증해야 하는 이유(Self-Validation), Value Object가 행동(도메인 로직)을 가질 수 있는 이유, Primitive Obsession 안티패턴 |
| [03. Aggregate 설계 — 일관성 경계를 결정하는 원칙](./tactical-design/03-aggregate-design.md) | Aggregate Root를 통해서만 내부 객체를 변경해야 하는 이유(불변식 우회 방지), Aggregate 경계 설정의 실전 기준(같은 트랜잭션에서 반드시 함께 변해야 하는 객체들), 경계 오류의 실제 증상 |
| [04. Aggregate 크기 결정 — 작은 Aggregate를 선호해야 하는 이유](./tactical-design/04-aggregate-size.md) | 큰 Aggregate가 트랜잭션 충돌과 락 경합을 일으키는 구체적 시나리오, Aggregate 간 참조를 ID로만 해야 하는 이유(지연 로딩과 Aggregate 경계 보호), 작은 Aggregate와 Eventually Consistent 설계의 트레이드오프 |
| [05. Domain Service vs Application Service — 책임의 분리](./tactical-design/05-domain-vs-application-service.md) | 단일 Entity에 표현할 수 없는 도메인 로직이 Domain Service로 가야 하는 이유, Application Service의 역할(오케스트레이션, 트랜잭션 경계, 포트 연결), 두 서비스를 혼동했을 때 도메인 로직이 흩어지는 문제 |
| [06. Repository 설계 원칙 — 컬렉션처럼 동작해야 한다](./tactical-design/06-repository-design.md) | 테이블당 Repository를 만들었을 때 Aggregate 불변식이 깨지는 시나리오, Add / Remove / Find 시맨틱이 도메인 레이어 Repository 인터페이스를 설계하는 방법, 도메인 레이어가 Repository 인터페이스를 소유하는 이유 |
| [07. Factory 패턴 — 복잡한 Aggregate 생성의 책임 분리](./tactical-design/07-factory-pattern.md) | 복잡한 생성 로직을 생성자에 두었을 때 생기는 책임 과적 문제, Factory Method vs Factory Class vs 정적 팩토리 선택 기준, Aggregate 생성 시점에 불변식 검증을 강제하는 방법 |
| [08. Domain Event 설계 — 과거형 이름과 이벤트 데이터 설계](./tactical-design/08-domain-event-design.md) | 이벤트가 과거형 이름(`OrderPlaced`, `PaymentCompleted`)을 가져야 하는 이유(이미 일어난 사실), 이벤트가 포함해야 할 최소 데이터 vs 풍부한 데이터의 트레이드오프, 이벤트 버저닝 전략과 소비자 호환성 유지 |

</details>

<br/>

### 🔹 Chapter 4: Domain Event와 Bounded Context 통합

> **핵심 질문:** DB 저장과 이벤트 발행을 어떻게 원자적으로 처리하는가? Saga는 어떻게 분산 트랜잭션 없이 여러 컨텍스트를 조율하는가?

<details>
<summary><b>이벤트 발행 패턴부터 Saga 보상 트랜잭션까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Domain Event가 결합도를 낮추는 원리](./domain-events/01-domain-event-decoupling.md) | 이벤트 발행자가 구독자를 모르는 설계가 결합도를 낮추는 원리, 동기 직접 호출과 이벤트 기반 통신의 일관성 트레이드오프, Bounded Context 간 통신에서 이벤트가 적합한 경우와 그렇지 않은 경우 |
| [02. 이벤트 발행 패턴 — Aggregate 내부 수집, Application Service 발행](./domain-events/02-event-publishing-patterns.md) | Aggregate가 내부에서 이벤트를 수집하고 Application Service에서 발행하는 이유(Aggregate가 인프라를 몰라야 함), `ApplicationEventPublisher` vs Kafka 직접 발행 선택 기준, 발행 실패 시 이벤트 유실 시나리오 |
| [03. Outbox Pattern 완전 분해 — DB 트랜잭션과 이벤트 원자성](./domain-events/03-outbox-pattern.md) | DB 저장 성공 후 Kafka 발행 실패 시 이벤트가 유실되는 Dual Write 문제, Outbox 테이블로 같은 트랜잭션에 이벤트를 저장하는 구조, Polling Publisher vs CDC(Debezium) 방식의 구현 비교와 선택 기준 |
| [04. Saga Pattern — 분산 트랜잭션 없는 비즈니스 프로세스 조율](./domain-events/04-saga-pattern.md) | 2PC 없이 여러 Bounded Context에 걸친 비즈니스 프로세스를 조율하는 Saga 구조, Choreography Saga(이벤트 반응형)와 Orchestration Saga(중앙 조율자)의 선택 기준, 보상 트랜잭션(Compensating Transaction) 설계 방법 |
| [05. Context 간 데이터 동기화 — Eventual Consistency 허용 설계](./domain-events/05-context-data-sync.md) | Event Sourcing 없이 Context 간 읽기 모델을 동기화하는 방법, Eventual Consistency를 허용하는 설계 결정의 근거, 정합성 검증 전략과 데이터 불일치를 감지하는 방법 |
| [06. Anti-Corruption Layer 실전 구현 — 외부 API를 도메인 모델로](./domain-events/06-acl-implementation.md) | 외부 API 응답 구조 변경이 도메인 모델에 전파되지 않도록 하는 Translator 구현, Mapper 패턴으로 외부 DTO와 도메인 객체를 분리하는 Spring 구현 예시, ACL 레이어의 테스트 전략 |

</details>

<br/>

### 🔹 Chapter 5: Spring + JPA로 DDD 구현하기

> **핵심 질문:** `@Entity`를 도메인 클래스에 붙이면 어떤 문제가 생기는가? Value Object를 JPA와 함께 불변으로 유지하는 방법은? `AbstractAggregateRoot`는 어떻게 동작하는가?

<details>
<summary><b>Persistence Ignorance부터 도메인 테스트 전략까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Persistence Ignorance 원칙 — 도메인이 JPA를 몰라야 하는 이유](./spring-jpa/01-persistence-ignorance.md) | `@Entity` / `@Column` / `@OneToMany`가 도메인 클래스에 침투할 때 생기는 영속성 관심사 오염, 도메인 모델이 JPA를 알면 테스트가 복잡해지는 이유, 실용적 타협(JPA 어노테이션 허용 범위)과 완전 분리의 트레이드오프 |
| [02. JPA Entity와 DDD Entity 분리 패턴 — Mapper로 변환하는 비용](./spring-jpa/02-jpa-vs-ddd-entity.md) | Domain Object와 JPA Entity를 완전히 분리했을 때 얻는 것과 잃는 것, Mapper 클래스가 두 레이어 사이를 변환하는 구조, 두 모델이 섞였을 때 `@Transient`가 증식하는 문제 |
| [03. Value Object JPA 매핑 — `@Embedded`와 `AttributeConverter`](./spring-jpa/03-value-object-jpa.md) | `@Embeddable`로 Value Object를 같은 테이블에 매핑하는 방법, 불변 Value Object를 JPA와 함께 사용하기 위한 기본 생성자 문제 해결, `AttributeConverter`로 단순 Value Object를 단일 컬럼에 매핑하는 방법 |
| [04. Aggregate JPA 매핑 — `CascadeType.ALL`의 함정](./spring-jpa/04-aggregate-jpa-mapping.md) | `CascadeType.ALL`이 Aggregate 경계를 넘어갈 때 발생하는 의도치 않은 영속성 전파, `FetchType.LAZY`가 Aggregate 외부에서 초기화될 때 `LazyInitializationException`이 발생하는 이유, Aggregate 경계가 JPA Fetch 전략을 결정하는 방식 |
| [05. Repository 구현 — 도메인 인터페이스를 인프라가 구현한다](./spring-jpa/05-repository-implementation.md) | 도메인 레이어에 Repository 인터페이스만 두고 인프라 레이어에서 `JpaRepository`를 상속한 구현체를 두는 구조, Spring Data JPA가 도메인 Repository 인터페이스를 구현하도록 연결하는 방법, 이 구조에서 도메인 테스트가 인프라 없이 가능한 이유 |
| [06. Domain Event Spring 통합 — `AbstractAggregateRoot`와 자동 발행](./spring-jpa/06-domain-event-spring.md) | `AbstractAggregateRoot`가 `@DomainEvents`와 `@AfterDomainEventsPublication`으로 동작하는 내부 구조, `save()` 호출 이후 이벤트가 자동 발행되는 흐름, `SimpleJpaRepository`가 이벤트를 발행하는 시점과 트랜잭션 경계 |
| [07. 테스트 전략 — 인프라 없이 도메인 불변식을 검증한다](./spring-jpa/07-testing-strategy.md) | Aggregate 불변식을 JPA 없이 순수 단위 테스트로 검증하는 패턴, Application Service 통합 테스트에서 Repository를 인메모리 구현으로 대체하는 방법, 도메인 이벤트 발행 여부를 검증하는 테스트 작성법 |

</details>

<br/>

### 🔹 Chapter 6: DDD 실전 프로젝트 — 전자상거래 도메인

> **핵심 질문:** 실제 전자상거래 도메인에서 어떻게 Context를 식별하고 Aggregate를 설계하는가? 주문 취소 시 재고 복원을 Saga로 어떻게 구현하는가?

<details>
<summary><b>도메인 분석부터 레거시 점진적 리팩터링까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. 도메인 분석 — 전자상거래 Bounded Context 식별](./real-world-project/01-domain-analysis.md) | 주문 / 상품 / 회원 / 배송 / 결제 Context 식별 과정, 각 Context의 핵심 도메인 모델과 Ubiquitous Language, Context Map 작성으로 통합 패턴 결정 |
| [02. 주문 Aggregate 설계 — 불변식과 상태 전이 캡슐화](./real-world-project/02-order-aggregate.md) | `Order` / `OrderLine` / `OrderStatus`의 불변식(최소 1개 상품, 결제 완료 전 배송 불가), 주문 상태 전이 로직을 Aggregate 내부에 캡슐화하는 구현, 상태 전이 불변식을 단위 테스트로 검증하는 방법 |
| [03. Bounded Context 간 통합 — 주문 이벤트가 여러 Context로 전파](./real-world-project/03-context-integration.md) | `OrderPlaced` 이벤트가 배송 / 재고 / 결제 Context로 전파되는 전체 흐름, Outbox Pattern으로 이벤트 유실 없이 발행하는 구현, Saga로 주문 취소 시 재고 복원과 결제 환불을 조율하는 방법 |
| [04. 읽기 모델 분리 — 조회 성능을 위한 별도 모델](./real-world-project/04-read-model-separation.md) | 주문 목록 조회를 위해 쓰기 모델과 분리된 읽기 모델을 구성하는 이유, Domain Event로 읽기 모델을 동기화하는 방법, CQRS 패턴과의 연결 지점(다음 레포 연결) |
| [05. DDD 리팩터링 — Transaction Script에서 점진적으로](./real-world-project/05-ddd-refactoring.md) | 기존 Service에 모든 로직이 몰린 Transaction Script 코드를 DDD로 점진적으로 이전하는 전략, Strangler Fig 패턴으로 레거시를 안전하게 개선하는 방법, 리팩터링 중 테스트가 안전망이 되는 이유 |

</details>

<br/>

### 🔹 Chapter 7: DDD 안티패턴과 트레이드오프

> **핵심 질문:** 어떤 실수가 DDD의 이점을 모두 날리는가? 완벽한 DDD를 추구하는 것이 왜 위험한가? 팀 역량과 도메인 복잡도에 따라 어디서 타협할 것인가?

<details>
<summary><b>Anemic Domain Model부터 실무 타협 전략까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Anemic Domain Model 안티패턴 — Entity가 데이터 컨테이너가 될 때](./antipatterns/01-anemic-domain-model.md) | Service에 모든 비즈니스 로직이 쌓이고 Entity에는 getter/setter만 남는 패턴이 생기는 이유, Anemic Model이 DDD의 모든 장점을 무효화하는 방식, Service 비대화 코드를 도메인 모델로 이전하는 리팩터링 전략 |
| [02. Aggregate 설계 실수 — 너무 큰 Aggregate와 직접 객체 참조](./antipatterns/02-aggregate-design-mistakes.md) | 하나의 Aggregate에 너무 많은 객체를 포함했을 때 발생하는 락 경합과 로딩 비용, Aggregate 간 직접 객체 참조가 경계를 허무는 방식, 트랜잭션 범위가 Aggregate를 넘어가는 설계의 증상과 수정 |
| [03. Bounded Context 경계 실수 — 너무 세분화 vs 너무 크게](./antipatterns/03-context-boundary-mistakes.md) | Context를 너무 세분화했을 때 발생하는 분산 모놀리스(모든 요청이 여러 서비스를 횡단) 문제, Context가 너무 클 때 Ubiquitous Language가 무너지는 방식, 경계를 재조정하는 전략과 판단 기준 |
| [04. DDD 과잉 적용 — CRUD에 DDD를 적용했을 때](./antipatterns/04-ddd-overengineering.md) | 단순 CRUD 도메인에 Aggregate / Repository / Domain Event를 모두 적용했을 때의 과잉 복잡도 비용, 도메인 복잡도에 따른 설계 수준 선택(Transaction Script → Table Module → Domain Model), 언제 DDD를 내려놓아야 하는가 |
| [05. 실무에서의 타협 — 완벽한 DDD는 없다](./antipatterns/05-practical-tradeoffs.md) | JPA 어노테이션을 도메인 클래스에 허용하는 범위와 그 근거, Aggregate 경계를 이상적으로 유지하기 어려운 실무 제약, 팀 역량과 도메인 복잡도에 따른 DDD 점진적 적용 전략 |

</details>

---

## 🔬 핵심 분석 대상

```
DDD 설계 전체 흐름:

1. Strategic Design (큰 그림)
   비즈니스 도메인 분석
     ↓
   Subdomain 식별 (Core / Supporting / Generic)
     ↓
   Bounded Context 경계 설정 (언어의 경계, 팀의 경계)
     ↓
   Context Map 작성 (컨텍스트 간 관계 정의)

2. Tactical Design (구현)
   각 Bounded Context 내부:
     ├── Aggregate (일관성 경계)
     │     ├── Aggregate Root (진입점, 불변식 수호자)
     │     ├── Entity (식별자로 구분)
     │     └── Value Object (값으로 구분, 불변)
     ├── Domain Service (Aggregate로 표현 불가한 도메인 로직)
     ├── Repository (Aggregate 단위, 컬렉션처럼)
     ├── Factory (복잡한 Aggregate 생성)
     └── Domain Event (Context 간 통신)

3. Bounded Context 통합
   주문 Context  →  [OrderPlaced 이벤트]  →  배송 Context
   주문 Context  →  [OrderPlaced 이벤트]  →  재고 Context
   주문 Context  →  [OrderPlaced 이벤트]  →  결제 Context

Aggregate 불변식 보호:
   외부 → Order.addLine() (Aggregate Root의 메서드만 허용)
   ❌ 금지: order.getLines().add(new OrderLine())  // 불변식 우회
   ✅ 허용: order.addLine(productId, qty, price)   // 불변식 검증 포함

   Order.addLine() 내부가 보호하는 불변식:
     → status 검사 (취소된 주문에는 추가 불가)
     → 최대 수량 검사
     → 직접 접근하면 이 검증들이 모두 우회됨

Value Object vs Entity:
   Money — 100원짜리 A와 100원짜리 B는 같다 (값이 같으면 동일)
   Order  — 주문 #1과 주문 #2는 내용이 같아도 다르다 (ID로 구분)

   Money를 Entity로 만들면?
     → money.setAmount(200)       // 불변성 파괴
     → 다른 Order가 같은 인스턴스 공유 → 사이드 이펙트 버그

Outbox Pattern — DB 저장과 이벤트 발행의 원자성:
   ❌ 잘못된 방법:
     transaction.begin()
     repository.save(order)       // DB 성공
     kafka.send(OrderPlaced)      // Kafka 실패 → 이벤트 유실!
     transaction.commit()

   ✅ Outbox Pattern:
     transaction.begin()
     repository.save(order)       // Order 저장
     outbox.save(OrderPlaced)     // 같은 트랜잭션에 이벤트 저장
     transaction.commit()
     // 별도 Polling Publisher 또는 Debezium CDC → Kafka 발행
```

---

## 🚀 실험 환경

```yaml
# docker-compose.yml
services:
  mysql:
    image: mysql:8.0
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: ddd_shop
    ports:
      - "3306:3306"

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    depends_on:
      - zookeeper
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
    ports:
      - "9092:9092"

  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
```

```java
// Before — Anemic Domain Model (이 레포에서 제거할 코드)
@Entity
public class Order {
    @Id private Long id;
    private String status;
    @OneToMany private List<OrderLine> lines;
    // getter/setter만 존재 — 비즈니스 로직 없음
}

@Service
public class OrderService {
    public void placeOrder(OrderRequest request) {
        Order order = new Order();
        order.setStatus("PENDING");
        if (request.getItems().isEmpty()) throw new Exception("...");
        // 모든 비즈니스 로직이 Service에 쌓임
    }
}

// After — Rich Domain Model with DDD (이 레포에서 만들 코드)
public class Order {  // Aggregate Root
    private OrderId id;
    private CustomerId customerId;
    private OrderStatus status;
    private List<OrderLine> lines;
    private List<DomainEvent> events = new ArrayList<>();

    // 불변식: 주문은 최소 1개 이상의 상품을 가져야 한다
    // 불변식: 취소된 주문에는 상품을 추가할 수 없다
    public static Order place(CustomerId customerId, List<OrderLineRequest> lines) {
        if (lines.isEmpty()) throw new EmptyOrderException();
        Order order = new Order(customerId, lines);
        order.events.add(new OrderPlaced(order.id, customerId));
        return order;
    }

    public void addLine(ProductId productId, int quantity, Money price) {
        if (this.status != OrderStatus.PENDING)
            throw new OrderNotModifiableException();
        this.lines.add(new OrderLine(productId, quantity, price));
    }

    public Money totalAmount() {
        return lines.stream()
            .map(OrderLine::subtotal)
            .reduce(Money.ZERO, Money::add);
    }
}
```

---

## 📖 각 문서 구성 방식

모든 문서는 동일한 구조로 작성됩니다.

| 섹션 | 설명 |
|------|------|
| 🎯 **핵심 질문** | 이 문서를 읽고 나면 답할 수 있는 질문 |
| 🔍 **왜 이 개념이 실무에서 중요한가** | 실무에서 마주치는 문제 상황과 이 개념의 연결 |
| 😱 **흔한 실수** | Before — DDD 없이 짜는 코드와 그 결과 |
| ✨ **올바른 접근** | After — DDD로 짠 코드 |
| 🔬 **내부 동작 원리** | 왜 이 설계가 더 나은가 — 불변식, 캡슐화, 결합도 분석 |
| 💻 **실전 코드** | Spring + JPA 기반 구현, 단위 테스트 포함 |
| 📊 **설계 비교** | Anemic vs Rich Domain Model, 직접 호출 vs 이벤트 등 |
| ⚖️ **트레이드오프** | 이 설계의 장단점, 언제 다른 접근을 택할 것인가 |
| 📌 **핵심 정리** | 한 화면 요약 |
| 🤔 **생각해볼 문제** | 개념을 더 깊이 이해하기 위한 질문 + 해설 |

---

## 🗺️ 추천 학습 경로

<details>
<summary><b>🟢 "Service에 로직 다 넣는 게 왜 문제인지 모른다" — DDD 첫 발 (3일)</b></summary>

<br/>

```
Day 1  Ch1-01  DDD가 해결하는 문제 → Anemic Model의 함정 이해
       Ch7-01  Anemic Domain Model 안티패턴 → 내가 짜고 있는 코드 진단
Day 2  Ch3-01  Entity vs Value Object → 가장 기본적인 모델 구분
       Ch3-02  Value Object 설계 원칙 → 불변성이 왜 중요한가
Day 3  Ch3-03  Aggregate 설계 → 일관성 경계를 이해
       Ch3-06  Repository 설계 원칙 → 테이블당 Repository의 문제 이해
```

</details>

<details>
<summary><b>🟡 "DDD 개념은 아는데 실제 Spring 코드로 어떻게 짜는지 모른다" — 구현 집중 (1주)</b></summary>

<br/>

```
Day 1  Ch3-03  Aggregate 설계 → 일관성 경계 원칙
       Ch3-04  Aggregate 크기 결정 → 작은 Aggregate 선호 이유
Day 2  Ch5-01  Persistence Ignorance → JPA 오염 문제 이해
       Ch5-03  Value Object JPA 매핑 → @Embedded, AttributeConverter
Day 3  Ch5-04  Aggregate JPA 매핑 → CascadeType.ALL 함정
       Ch5-05  Repository 구현 → 도메인 인터페이스 + 인프라 구현체
Day 4  Ch3-08  Domain Event 설계 → 이벤트 데이터 설계 원칙
       Ch4-02  이벤트 발행 패턴 → Aggregate 내부 수집, Service 발행
Day 5  Ch4-03  Outbox Pattern → DB 저장과 이벤트 원자성
       Ch5-06  AbstractAggregateRoot → Spring 자동 이벤트 발행
Day 6  Ch5-07  테스트 전략 → 도메인 단위 테스트
       Ch6-02  주문 Aggregate 설계 → 실전 예시
Day 7  Ch6-03  Context 간 통합 → Saga와 이벤트 전파 전체 흐름
```

</details>

<details>
<summary><b>🔴 "Strategic Design부터 Saga까지 DDD 전체를 완전히 정복한다" — 전체 정복 (7주)</b></summary>

<br/>

```
1주차  Chapter 1 전체 — DDD의 철학과 판단 기준
        → 내 프로젝트에 DDD가 필요한지 판단, 레이어드 아키텍처와 비교

2주차  Chapter 2 전체 — Strategic Design
        → 이벤트 스토밍 워크숍 시뮬레이션, Context Map 직접 작성

3주차  Chapter 3 전체 — Tactical Design
        → Entity / VO / Aggregate / Repository 직접 구현 + 불변식 테스트

4주차  Chapter 4 전체 — Domain Event와 통합 패턴
        → Outbox Pattern 구현, Choreography Saga 실험

5주차  Chapter 5 전체 — Spring + JPA 구현
        → Persistence Ignorance 적용, AbstractAggregateRoot 이벤트 발행 디버깅

6주차  Chapter 6 전체 — 전자상거래 실전 프로젝트
        → 주문 Aggregate 전체 구현, Saga로 주문 취소 흐름 구현

7주차  Chapter 7 전체 — 안티패턴과 트레이드오프
        → 내 코드의 DDD 안티패턴 진단, 점진적 리팩터링 계획 수립
```

</details>

---

## 🔗 연관 레포지토리

| 레포 | 주요 내용 | 연관 챕터 |
|------|----------|-----------|
| [spring-data-transaction](https://github.com/dev-book-lab/spring-data-transaction) | JPA/Hibernate 기본 동작, `@Transactional` 내부 구조 | Ch5 전체 (JPA Entity 매핑, Cascade, Fetch 전략 이해 선행 필수) |
| [spring-core-deep-dive](https://github.com/dev-book-lab/spring-core-deep-dive) | Bean 생명주기, DI, AOP Proxy | Ch5-06 (`AbstractAggregateRoot` 이벤트 발행 흐름), Ch5-07 (Application Service 테스트) |
| [kafka-deep-dive](https://github.com/dev-book-lab/kafka-deep-dive) | Kafka Producer/Consumer, 메시지 보장 | Ch4-03 (Outbox Pattern → Kafka 발행), Ch4-04 (Saga → Kafka 이벤트 기반 조율) |
| [database-internals](https://github.com/dev-book-lab/database-internals) | MySQL InnoDB MVCC, 트랜잭션 격리 | Ch5-04 (Aggregate 경계와 트랜잭션 범위의 관계), Ch4-03 (Outbox 테이블의 트랜잭션 보장) |

> 💡 이 레포는 **도메인 설계 원칙**에 집중합니다. Spring을 몰라도 Chapter 1~3의 Strategic/Tactical Design은 순수 도메인 설계 관점으로 학습할 수 있습니다. Chapter 5는 `spring-data-transaction` 선행 학습 시 더 깊이 연결됩니다.

---

## 📚 Reference

- [Domain-Driven Design — Eric Evans (The Blue Book)](https://www.oreilly.com/library/view/domain-driven-design-tackling/0321125215/)
- [Implementing Domain-Driven Design — Vaughn Vernon (The Red Book)](https://www.oreilly.com/library/view/implementing-domain-driven-design/9780133039900/)
- [Domain-Driven Design Distilled — Vaughn Vernon](https://www.oreilly.com/library/view/domain-driven-design-distilled/9780134434964/)
- [DDD Community — dddcommunity.org](https://dddcommunity.org/)
- [Martin Fowler — DDD 관련 글 모음](https://martinfowler.com/tags/domain%20driven%20design.html)
- [Udi Dahan Blog — Aggregate, Domain Event 설계](https://udidahan.com/)
- [Spring Data — AbstractAggregateRoot 공식 문서](https://docs.spring.io/spring-data/commons/docs/current/api/org/springframework/data/domain/AbstractAggregateRoot.html)

---

<div align="center">

**⭐️ 도움이 되셨다면 Star를 눌러주세요!**

Made with ❤️ by [Dev Book Lab](https://github.com/dev-book-lab)

<br/>

*"엔티티와 서비스를 만드는 것과, 도메인의 불변식(Invariant)을 보호하는 객체 설계를 아는 것은 다르다"*

</div>
