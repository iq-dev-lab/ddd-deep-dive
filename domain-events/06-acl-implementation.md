# Anti-Corruption Layer 실전 구현 — 외부 API를 도메인 모델로

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 외부 API 응답 구조 변경이 도메인 모델에 전파되지 않도록 하는 Translator 구현 방법은?
- Mapper 패턴으로 외부 DTO와 도메인 객체를 분리하는 Spring 구현 예시는?
- ACL 레이어의 테스트 전략은 어떻게 설계하는가?
- ACL이 너무 복잡해질 때 책임을 어떻게 분리하는가?
- 복수의 외부 시스템을 단일 포트 인터페이스로 추상화하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Ch2-05에서 ACL의 개념을 다뤘다. 이 문서는 **Spring 환경에서의 실전 구현**에 집중한다. 이론과 실제 사이의 간격을 메우는 것이 목표다: Port 인터페이스 설계, Translator 구현, Adapter 조합, 그리고 외부 API 변경에 대한 테스트 전략까지.

외부 API는 우리 통제 밖이다. 공급자가 응답 필드명을 바꾸거나, 상태 코드를 추가하거나, API 버전을 올린다. ACL이 없으면 이 변경들이 도메인 코드 전체에 파급된다. ACL이 있으면 Translator 한 클래스만 수정하면 된다.

---

## 😱 흔한 실수 (Before — 외부 DTO가 도메인에 직접 침투)

```java
// 물류사 SDK 응답 DTO (외부 라이브러리)
public class CjLogisticsTrackingResponse {
    public String trackNo;        // 운송장 번호
    public String statusCd;       // "01"=접수, "02"=집하, "80"=배달완료, "91"=미배달
    public String delivDt;        // "20240315" 형식
    public List<TrackingDetail> details;
}

// 안티패턴: 도메인 서비스가 외부 DTO를 직접 사용
@Service
public class ShipmentService {

    private final CjLogisticsApiClient cjClient;

    public ShipmentStatus getStatus(String trackingNumber) {
        CjLogisticsTrackingResponse response = cjClient.track(trackingNumber);

        // 외부 코드가 도메인 로직에 침투
        if ("80".equals(response.statusCd)) {         // "80"이 뭔지 알아야 함
            return ShipmentStatus.DELIVERED;
        } else if ("91".equals(response.statusCd)) {  // "91"이 뭔지 알아야 함
            return ShipmentStatus.DELIVERY_FAILED;
        } else if (response.statusCd.startsWith("0")) {
            return ShipmentStatus.IN_TRANSIT;
        }
        return ShipmentStatus.UNKNOWN;
    }
}
```

```
6개월 후 문제:
  CJ대한통운 API v2 배포: statusCd "02" → "02A" (집하 완료) 변경
  ShipmentService의 startsWith("0") 로직이 "02A"도 매칭
  → 버그 없이 넘어가지만 명확성 상실

  배송사를 한진택배로 교체:
  한진택배 API 응답: { "trackingStatus": "PICKED_UP", "estimatedDate": "2024-03-15" }
  → ShipmentService 전체 수정 필요 (CJ 코드와 한진 코드가 섞임)
```

---

## ✨ 올바른 접근 (After — 완전한 ACL 레이어)

```java
// 계층 구조:
// Domain Port (인터페이스) ← 도메인이 의존
//   ↑ implements
// ACL Adapter (구현체) ← 인프라가 Domain을 의존 (DI)
//   ├── External API Client (HTTP 통신)
//   └── Translator (번역)

// 1. Domain Port — 도메인이 필요로 하는 것 (순수 도메인 언어)
public interface ShipmentTrackingPort {
    Optional<ShipmentTracking> findTracking(TrackingNumber trackingNumber);
    boolean isDelivered(TrackingNumber trackingNumber);
}

// 2. Domain 값 객체들 (외부와 무관)
public record ShipmentTracking(
    TrackingNumber trackingNumber,
    ShipmentTrackingStatus status,    // 도메인 언어 (PREPARING, IN_TRANSIT, DELIVERED...)
    LocalDate estimatedDelivery,
    List<TrackingEvent> events
) {}

// 3. Translator — 번역 전담
@Component
public class CjLogisticsTranslator {

    public ShipmentTracking toDomain(CjLogisticsTrackingResponse response) {
        return new ShipmentTracking(
            new TrackingNumber(response.trackNo),
            translateStatus(response.statusCd),
            parseDate(response.delivDt),
            translateEvents(response.details)
        );
    }

    // 번역 로직이 한 곳에 집중
    private ShipmentTrackingStatus translateStatus(String cjCode) {
        return switch (cjCode) {
            case "01"       -> ShipmentTrackingStatus.ACCEPTED;
            case "02", "02A"-> ShipmentTrackingStatus.PICKED_UP;  // v2 추가도 여기만
            case "11", "12" -> ShipmentTrackingStatus.IN_TRANSIT;
            case "80"       -> ShipmentTrackingStatus.DELIVERED;
            case "91"       -> ShipmentTrackingStatus.DELIVERY_FAILED;
            default         -> throw new UnknownCjStatusException("알 수 없는 CJ 상태코드: " + cjCode);
        };
    }

    private LocalDate parseDate(String cjDate) {
        if (cjDate == null || cjDate.isBlank()) return null;
        return LocalDate.parse(cjDate, DateTimeFormatter.BASIC_ISO_DATE);  // "20240315"
    }
}

// 4. Adapter — Port 구현, Client + Translator 조합
@Component
public class CjLogisticsShipmentAdapter implements ShipmentTrackingPort {

    private final CjLogisticsApiClient cjClient;
    private final CjLogisticsTranslator translator;

    @Override
    public Optional<ShipmentTracking> findTracking(TrackingNumber trackingNumber) {
        return cjClient.track(trackingNumber.value())
            .map(translator::toDomain);
    }

    @Override
    public boolean isDelivered(TrackingNumber trackingNumber) {
        return findTracking(trackingNumber)
            .map(t -> t.status() == ShipmentTrackingStatus.DELIVERED)
            .orElse(false);
    }
}

// 5. 도메인 서비스 — 외부 물류사를 전혀 모름
@Service
public class ShipmentApplicationService {

    private final ShipmentTrackingPort trackingPort;  // ACL을 통해 접근

    public ShipmentStatusResponse getShipmentStatus(TrackingNumber trackingNumber) {
        return trackingPort.findTracking(trackingNumber)
            .map(tracking -> new ShipmentStatusResponse(
                tracking.status().koreanName(),     // 도메인 언어
                tracking.estimatedDelivery()
            ))
            .orElseThrow(() -> new TrackingNotFoundException(trackingNumber));
    }
}
```

---

## 🔬 내부 동작 원리

### 1. ACL의 계층 구조와 책임

```
계층 분리:

Domain Layer:
  ShipmentTrackingPort (인터페이스) — 도메인이 필요한 것 정의
  ShipmentTracking (VO) — 도메인 모델
  TrackingNumber (VO) — 도메인 식별자

Infrastructure Layer:
  CjLogisticsApiClient — HTTP 클라이언트 (기술 세부사항)
  CjLogisticsTranslator — 번역 (도메인 ↔ 외부 모델)
  CjLogisticsShipmentAdapter — Port 구현체

각 클래스의 단일 책임:

CjLogisticsApiClient:
  - HTTP 요청 실행
  - 재시도, 타임아웃
  - 외부 DTO 반환
  - 인증 처리
  ❌ 번역 없음, 도메인 로직 없음

CjLogisticsTranslator:
  - 외부 DTO → 도메인 VO 변환
  - 코드 번역 ("80" → DELIVERED)
  - 형식 변환 ("20240315" → LocalDate)
  ❌ HTTP 호출 없음, 도메인 비즈니스 로직 없음

CjLogisticsShipmentAdapter:
  - ShipmentTrackingPort 구현
  - Client 호출 → Translator에 위임
  - 예외 처리 (외부 예외 → 도메인 예외)
  ❌ 번역 로직 없음, HTTP 직접 호출 없음
```

### 2. 복수의 외부 시스템을 단일 포트로

```java
// 여러 배송사를 단일 포트로 추상화

// 배송사 선택 전략
@Component
public class ShipmentTrackingPortRouter implements ShipmentTrackingPort {

    private final Map<CarrierType, ShipmentTrackingPort> adapters;

    public ShipmentTrackingPortRouter(
            CjLogisticsShipmentAdapter cjAdapter,
            HanjinShipmentAdapter hanjinAdapter,
            LotteShipmentAdapter lotteAdapter) {
        this.adapters = Map.of(
            CarrierType.CJ, cjAdapter,
            CarrierType.HANJIN, hanjinAdapter,
            CarrierType.LOTTE, lotteAdapter
        );
    }

    @Override
    public Optional<ShipmentTracking> findTracking(TrackingNumber trackingNumber) {
        CarrierType carrier = trackingNumber.inferCarrier();  // 운송장 번호 형식으로 배송사 추론
        ShipmentTrackingPort adapter = adapters.get(carrier);
        if (adapter == null) throw new UnsupportedCarrierException(carrier);
        return adapter.findTracking(trackingNumber);
    }
}

// 도메인은 배송사 구분 없이 동일한 포트 사용
// → 새 배송사 추가 = 새 Adapter 추가 (도메인 변경 없음)
```

### 3. Facade — 복잡한 외부 API 단순화

```java
// 외부 API가 여러 엔드포인트로 나뉜 경우 → Facade로 합치기
@Component
public class CjLogisticsApiClient {

    // CJ API는 기본 추적 + 상세 이력이 별도 엔드포인트
    public Optional<CjLogisticsTrackingResponse> track(String trackNo) {
        try {
            // 기본 상태 조회
            CjBasicStatusResponse basicStatus = cjRestClient.getBasicStatus(trackNo);
            // 이력 조회 (필요 시)
            CjDetailResponse detail = cjRestClient.getDetail(trackNo);

            // 두 응답을 하나로 합쳐서 반환 (Facade)
            return Optional.of(merge(basicStatus, detail));
        } catch (CjApiNotFoundException e) {
            return Optional.empty();
        } catch (CjApiException e) {
            throw new ExternalCarrierApiException("CJ API 오류: " + e.getMessage(), e);
        }
    }

    private CjLogisticsTrackingResponse merge(CjBasicStatusResponse basic, CjDetailResponse detail) {
        return new CjLogisticsTrackingResponse(
            basic.trackNo,
            basic.statusCd,
            basic.delivDt,
            detail.trackingHistory
        );
    }
}
```

### 4. ACL 테스트 전략

```
테스트 계층:

1. Translator 단위 테스트 (최우선)
   - 외부 코드 → 도메인 모델 번역 검증
   - Mock 없음, 순수 Java
   - 모든 상태 코드 케이스 커버

2. ApiClient 단위 테스트
   - HTTP 요청/응답 형식 검증
   - MockWebServer 또는 WireMock 사용
   - 타임아웃, 재시도 동작 검증

3. Adapter 통합 테스트 (선택적)
   - Translator + Client 조합 검증
   - WireMock으로 외부 API 시뮬레이션

4. 외부 API 계약 테스트 (Consumer-Driven)
   - Pact 등으로 API 계약 검증
   - 외부 API 변경 시 자동 감지
```

---

## 💻 실전 코드

### Translator 완전 테스트

```java
class CjLogisticsTranslatorTest {

    private final CjLogisticsTranslator translator = new CjLogisticsTranslator();

    @Test
    void translateStatus_delivered_80() {
        CjLogisticsTrackingResponse response = createResponse("80", "20240315");
        ShipmentTracking tracking = translator.toDomain(response);
        assertThat(tracking.status()).isEqualTo(ShipmentTrackingStatus.DELIVERED);
    }

    @Test
    void translateStatus_deliveryFailed_91() {
        CjLogisticsTrackingResponse response = createResponse("91", null);
        ShipmentTracking tracking = translator.toDomain(response);
        assertThat(tracking.status()).isEqualTo(ShipmentTrackingStatus.DELIVERY_FAILED);
    }

    @Test
    void translateDate_standardFormat() {
        CjLogisticsTrackingResponse response = createResponse("80", "20240315");
        ShipmentTracking tracking = translator.toDomain(response);
        assertThat(tracking.estimatedDelivery()).isEqualTo(LocalDate.of(2024, 3, 15));
    }

    @Test
    void translateDate_nullDate() {
        CjLogisticsTrackingResponse response = createResponse("11", null);
        ShipmentTracking tracking = translator.toDomain(response);
        assertThat(tracking.estimatedDelivery()).isNull();
    }

    @Test
    void translateStatus_unknownCode_throwsException() {
        CjLogisticsTrackingResponse response = createResponse("99", null);
        assertThatThrownBy(() -> translator.toDomain(response))
            .isInstanceOf(UnknownCjStatusException.class)
            .hasMessageContaining("99");
    }

    // 모든 상태 코드를 파라미터로 테스트
    @ParameterizedTest
    @CsvSource({
        "01, ACCEPTED",
        "02, PICKED_UP",
        "02A, PICKED_UP",  // v2 신규 코드
        "11, IN_TRANSIT",
        "80, DELIVERED",
        "91, DELIVERY_FAILED"
    })
    void translateAllStatusCodes(String cjCode, ShipmentTrackingStatus expected) {
        CjLogisticsTrackingResponse response = createResponse(cjCode, null);
        ShipmentTracking tracking = translator.toDomain(response);
        assertThat(tracking.status()).isEqualTo(expected);
    }
}
```

### WireMock을 이용한 API 클라이언트 테스트

```java
@SpringBootTest
class CjLogisticsApiClientTest {

    @Autowired
    private CjLogisticsApiClient cjClient;

    private WireMockServer wireMock;

    @BeforeEach
    void setUp() {
        wireMock = new WireMockServer(8089);
        wireMock.start();
    }

    @Test
    void track_success() {
        wireMock.stubFor(get(urlEqualTo("/api/tracking/CJ1234567890KR"))
            .willReturn(aResponse()
                .withStatus(200)
                .withHeader("Content-Type", "application/json")
                .withBody("""
                    {
                        "trackNo": "CJ1234567890KR",
                        "statusCd": "80",
                        "delivDt": "20240315"
                    }
                    """)));

        Optional<CjLogisticsTrackingResponse> result = cjClient.track("CJ1234567890KR");

        assertThat(result).isPresent();
        assertThat(result.get().statusCd).isEqualTo("80");
    }

    @Test
    void track_notFound_returnsEmpty() {
        wireMock.stubFor(get(anyUrl()).willReturn(aResponse().withStatus(404)));

        Optional<CjLogisticsTrackingResponse> result = cjClient.track("UNKNOWN123");
        assertThat(result).isEmpty();
    }
}
```

---

## 📊 설계 비교

```
ACL 없음 vs ACL 있음:

                ACL 없음               ACL 있음
────────────┼──────────────────────┼──────────────────────────
외부 변경 영향│ 도메인 코드 전체        │ Translator만 수정
────────────┼──────────────────────┼──────────────────────────
배송사 교체  │ 대규모 수정             │ 새 Adapter 추가
────────────┼──────────────────────┼──────────────────────────
도메인 테스트 │ 외부 API Mock 필요     │ FakePort로 독립 테스트
────────────┼──────────────────────┼──────────────────────────
번역 테스트  │ 서비스 전체 실행 필요   │ Translator만 단위 테스트
────────────┼──────────────────────┼──────────────────────────
도메인 언어  │ 외부 코드 혼재          │ 도메인 언어로 일관
            │ ("80", "91" 등)       │ (DELIVERED, FAILED)
────────────┼──────────────────────┼──────────────────────────
복잡도       │ 낮음 (초기)            │ 중간 (Port, Adapter, Translator)
```

---

## ⚖️ 트레이드오프

```
ACL의 현실적 비용:
  ① 클래스 수 증가: Port + Adapter + Translator + Client
     소규모 팀에서 처음에는 과잉으로 느껴질 수 있음
  
  ② 유지 비용: Translator의 외부 코드 목록 관리
     외부 API 신규 상태 추가 시 Translator 업데이트 필요
  
  ③ 완전한 분리(Domain ↔ JPA Entity)의 추가 비용:
     JPA Entity와 Domain 클래스 이중 유지 + Mapper
     (Ch5-02에서 상세 논의)

ACL을 단순화할 수 있는 경우:
  외부 API가 잘 설계됐고 도메인과 유사하면 → Conformist (직접 사용)
  외부 API가 매우 안정적이고 자주 바뀌지 않으면 → 간단한 Mapper로 충분

언제 완전한 ACL이 가치 있는가:
  외부 API가 자주 바뀌거나 버전업이 잦을 때
  외부 API 공급자가 우리 요구를 무시할 때
  도메인 테스트에서 외부 의존을 완전히 제거해야 할 때
```

---

## 📌 핵심 정리

```
ACL 실전 구현 핵심:

계층 구조:
  Domain Port (인터페이스) → 도메인 언어로 정의
  External Client → HTTP 통신 전담
  Translator → 번역 전담 (외부 코드 → 도메인 VO)
  Adapter → Port 구현, Client + Translator 조합

단일 책임:
  Client: HTTP만
  Translator: 번역만
  Adapter: 조합만
  → 각자 독립적으로 테스트 가능

복수 외부 시스템:
  Router로 동일 Port 인터페이스 아래 여러 Adapter
  → 새 배송사 = 새 Adapter (도메인 무관)

테스트 전략:
  Translator: 모든 상태 코드 단위 테스트 (Mock 없음)
  Client: WireMock으로 HTTP 테스트
  도메인: FakePort로 ACL 없이 독립 테스트

ACL 가치:
  외부 변경 → Translator만 수정
  공급자 교체 → 새 Adapter 추가
  도메인은 외부를 전혀 모름
```

---

## 🤔 생각해볼 문제

**Q1.** CJ대한통운 외에 한진택배도 지원해야 한다. 두 배송사의 API 응답 형식이 전혀 다르다. 어떻게 설계하는가?

<details>
<summary>해설 보기</summary>

**각 배송사마다 별도 Translator + Adapter를 만들고, Router로 통합합니다.**

```java
// 한진 전용 Translator
@Component
public class HanjinLogisticsTranslator {
    public ShipmentTracking toDomain(HanjinTrackingResponse response) {
        return new ShipmentTracking(
            new TrackingNumber(response.getTrackingId()),
            translateHanjinStatus(response.getTrackingStatus()),  // 한진 상태 코드 번역
            parseHanjinDate(response.getExpectedDate()),
            List.of()
        );
    }

    private ShipmentTrackingStatus translateHanjinStatus(String hanjinStatus) {
        return switch (hanjinStatus) {
            case "PICKED_UP"  -> ShipmentTrackingStatus.PICKED_UP;
            case "MOVING"     -> ShipmentTrackingStatus.IN_TRANSIT;
            case "DELIVERED"  -> ShipmentTrackingStatus.DELIVERED;
            default -> throw new UnknownHanjinStatusException(hanjinStatus);
        };
    }
}

// 한진 전용 Adapter
@Component
public class HanjinShipmentAdapter implements ShipmentTrackingPort {
    private final HanjinApiClient hanjinClient;
    private final HanjinLogisticsTranslator translator;

    @Override
    public Optional<ShipmentTracking> findTracking(TrackingNumber trackingNumber) {
        return hanjinClient.track(trackingNumber.value())
            .map(translator::toDomain);
    }
}

// Router: Port 구현, 배송사 선택
@Primary  // ShipmentTrackingPort로 주입 시 Router가 선택됨
@Component
public class ShipmentTrackingPortRouter implements ShipmentTrackingPort {
    private final Map<CarrierType, ShipmentTrackingPort> adapters = Map.of(
        CarrierType.CJ, cjAdapter,
        CarrierType.HANJIN, hanjinAdapter
    );

    @Override
    public Optional<ShipmentTracking> findTracking(TrackingNumber trackingNumber) {
        CarrierType carrier = trackingNumber.inferCarrier();
        return adapters.getOrDefault(carrier, defaultAdapter).findTracking(trackingNumber);
    }
}
```

도메인 코드(ShipmentApplicationService)는 변경 없습니다.

</details>

---

**Q2.** Translator에서 처리되지 않은 외부 코드를 받으면 예외를 던져야 하는가, UNKNOWN 상태로 처리해야 하는가?

<details>
<summary>해설 보기</summary>

**도메인 중요도와 외부 API 안정성에 따라 다릅니다.**

**예외를 던지는 경우 (권장):**
```java
private ShipmentTrackingStatus translateStatus(String code) {
    return switch (code) {
        case "80" -> ShipmentTrackingStatus.DELIVERED;
        default -> throw new UnknownCarrierStatusException("알 수 없는 배송 상태 코드: " + code);
    };
    // 장점: 새 코드 추가 즉시 감지, 빠른 실패
    // 단점: 예상치 못한 코드에 전체 실패
}
```

**UNKNOWN으로 처리하는 경우:**
```java
private ShipmentTrackingStatus translateStatus(String code) {
    return switch (code) {
        case "80" -> ShipmentTrackingStatus.DELIVERED;
        default -> {
            log.warn("알 수 없는 배송 상태 코드: {}, UNKNOWN으로 처리", code);
            metricsService.increment("shipment.unknown_status_code", "code", code);
            yield ShipmentTrackingStatus.UNKNOWN;
        }
    };
    // 장점: 서비스 중단 없음
    // 단점: 문제를 나중에 발견할 수 있음
}
```

**권장:** 예외를 던지되 알림으로 즉시 인지하고, Dead Letter Queue에 저장해서 코드 추가 후 재처리. 서비스 중단보다 빠른 감지가 더 중요합니다.

</details>

---

**Q3.** ACL Translator 테스트에서 외부 API가 반환할 수 있는 모든 상태 코드를 어떻게 파악하고 커버하는가?

<details>
<summary>해설 보기</summary>

**외부 API 문서 + 실제 운영 로그 + Pact 계약 테스트를 조합합니다.**

```java
// 1. API 문서 기반: 파라미터화 테스트
@ParameterizedTest
@CsvSource({
    "01, ACCEPTED",
    "02, PICKED_UP",
    "11, IN_TRANSIT",
    "12, IN_TRANSIT",
    "80, DELIVERED",
    "91, DELIVERY_FAILED"
})
void translateAllDocumentedStatusCodes(String apiCode, String expectedStatus) { ... }

// 2. 실제 운영에서 발견된 코드 추가
// 로그 분석: SELECT DISTINCT statusCd FROM cj_api_response_log
// → "02A" 발견 → 테스트 추가 → Translator 코드 추가

// 3. Pact 계약 테스트 (Consumer-Driven Contract Testing)
// 우리가 기대하는 응답 형식을 정의 → CJ가 이를 확인하고 준수
@Pact(consumer = "shipment-service", provider = "cj-logistics-api")
public RequestResponsePact trackingPact(PactDslWithProvider builder) {
    return builder
        .given("송장번호 CJ1234567890KR이 존재함")
        .uponReceiving("송장 추적 요청")
            .path("/api/tracking/CJ1234567890KR")
            .method("GET")
        .willRespondWith()
            .status(200)
            .body(new PactDslJsonBody()
                .stringType("trackNo", "CJ1234567890KR")
                .stringMatcher("statusCd", "\\d{2}[A-Z]?", "80")
                .stringMatcher("delivDt", "\\d{8}", "20240315"))
        .toPact();
}
```

운영 로그에서 실제로 받은 응답을 기록하고, 주기적으로 새 코드를 발견해 테스트와 Translator에 추가하는 것이 가장 실용적인 방법입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Context 간 데이터 동기화](./05-context-data-sync.md)** | **[홈으로 🏠](../README.md)** | **[Chapter 5로 이동: Spring + JPA — Persistence Ignorance ➡️](../spring-jpa/01-persistence-ignorance.md)**

</div>
