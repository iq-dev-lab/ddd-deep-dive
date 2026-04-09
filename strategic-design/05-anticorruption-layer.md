# Anticorruption Layer — 외부 모델의 오염을 막는 번역기

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 외부 시스템의 모델이 도메인 모델을 "오염"시킨다는 것이 구체적으로 무슨 의미인가?
- ACL이 Translator로 동작하는 구조는 어떻게 생겼는가?
- Spring에서 ACL을 Mapper/Adapter 패턴으로 어떻게 구현하는가?
- ACL 없이 외부 API를 직접 사용했을 때 어떤 시나리오에서 장애가 발생하는가?
- ACL이 너무 복잡해지는 것을 어떻게 방지하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

현실의 소프트웨어는 단독으로 존재하지 않는다. PG사 결제 API, SMS 발송 서비스, 물류사 연동, 레거시 ERP, 외부 파트너 API... 이 모든 외부 시스템은 자신만의 모델과 언어를 갖고 있다. 이 외부 모델이 우리 도메인 코드 안으로 스며들면 어떻게 되는가?

외부 API가 업데이트될 때마다 도메인 코드를 수정해야 한다. 도메인 테스트에 외부 SDK가 필요해진다. 외부 시스템의 이상한 코드(`status = 'B2C_RETAIL'`)가 도메인 클래스 안에 들어온다. ACL은 이 오염을 막는 방역선이다.

---

## 😱 흔한 실수 (Before — 외부 모델이 도메인을 오염시키는 패턴)

```
상황: 물류사 API 직접 연동

외부 물류사 API 응답 구조:
  {
    "SHIP_NO": "KR20240315001",
    "CUST_TYPE": "B2C",
    "ITEM_LIST": [
      {"SKU": "PRD-001", "QTY": 2, "UNIT_PRC": 15000}
    ],
    "SHIP_ADDR": {
      "CITY_CD": "11", "DIST_CD": "680", "ADDR1": "..."
    },
    "STA_CD": "DLV",   ← 배송 중 (DLV = Delivery)
    "ETA_DT": "20240317"
  }
```

```java
// ACL 없이 외부 모델을 도메인에 직접 사용

// 외부 API 응답이 도메인 클래스에 침투
public class Shipment {  // 우리 도메인 클래스인데...
    private String shipNo;     // 외부 명명 규칙 (언더스코어 대신 카멜케이스지만 의미는 외부 것)
    private String custType;   // "B2C", "B2B" — 외부 코드
    private String staCd;      // "DLV", "PKG", "DEL" — 외부 상태 코드
    private String etaDt;      // "20240317" — 외부 날짜 형식

    // 외부 상태 코드를 직접 if/else로 처리
    public boolean isDelivering() {
        return "DLV".equals(this.staCd);  // 외부 코드가 도메인 로직에 침투
    }

    public boolean isDelivered() {
        return "DEL".equals(this.staCd);  // "DEL"이 뭔지 3개월 후엔 모름
    }
}

@Service
public class ShipmentService {
    public ShipmentStatus getStatus(String shipNo) {
        ExternalLogisticsResponse response = logisticsApi.getShipment(shipNo);
        // 외부 API 응답 필드명을 Service에서 직접 파싱
        String staCd = response.getStaCd();
        if ("DLV".equals(staCd)) return ShipmentStatus.IN_TRANSIT;
        if ("PKG".equals(staCd)) return ShipmentStatus.PREPARING;
        if ("DEL".equals(staCd)) return ShipmentStatus.DELIVERED;
        // 외부 코드가 추가되면 여기를 수정
        return ShipmentStatus.UNKNOWN;
    }
}
```

```
6개월 후 장애:

물류사 API 변경:
  "DLV"가 "DELIVERING"으로 변경됨 (v2 API)
  
영향범위:
  isDelivering() 메서드 오작동 → false 반환
  getStatus() 에서 UNKNOWN 반환
  배송 중인 주문이 모두 "알 수 없음" 상태로 표시
  고객 CS 폭발

더 심각한 문제:
  "DLV".equals() 가 코드 전체에 15곳 분산됨
  → 전부 찾아서 수정
  → 테스트: 외부 API 없이는 staCd 로직 테스트 불가능
  → QA에서 실제 물류사 API 호출 필요 → 느린 테스트

레거시 코드가 된 후:
  신입 개발자: "staCd가 뭔가요? DLV는 무슨 뜻이에요?"
  → 문서 없음. 물류사 담당자에게 물어봐야 함.
```

---

## ✨ 올바른 접근 (After — ACL이 번역기 역할)

```
ACL 구조:

  우리 도메인                ACL                  외부 물류사
  (깨끗한 언어)           (번역 레이어)              (이상한 언어)

  Shipment ←── ShipmentStatus ←── LogisticsTranslator ←── "DLV", "PKG", "DEL"
  (IN_TRANSIT)                    (번역 담당)

물류사 API가 "DLV" → "DELIVERING"으로 바꿔도:
  LogisticsTranslator.translateStatus() 만 수정
  도메인 코드(Shipment, ShipmentService) 무관
```

```java
// ACL 레이어 구조

// 1. 도메인 Port — 도메인이 필요한 것을 인터페이스로 정의
public interface ShipmentTrackingPort {
    Optional<ShipmentTracking> findTracking(TrackingNumber trackingNumber);
}

// 2. 외부 API 클라이언트 — HTTP 통신만 담당
@Component
public class LogisticsApiClient {

    private final RestTemplate restTemplate;

    public Optional<LogisticsResponse> getShipment(String shipNo) {
        try {
            LogisticsResponse response = restTemplate.getForObject(
                logisticsApiUrl + "/shipments/{shipNo}",
                LogisticsResponse.class,
                shipNo
            );
            return Optional.ofNullable(response);
        } catch (HttpClientErrorException.NotFound e) {
            return Optional.empty();
        }
    }
}

// 3. Translator — 외부 모델 → 도메인 모델 번역 (핵심!)
@Component
public class LogisticsTranslator {

    public ShipmentTracking toDomain(LogisticsResponse response) {
        return new ShipmentTracking(
            new TrackingNumber(response.getShipNo()),
            translateStatus(response.getStaCd()),      // 번역 핵심
            translateDate(response.getEtaDt()),        // 날짜 형식 번역
            translateAddress(response.getShipAddr())  // 주소 구조 번역
        );
    }

    // 번역 로직이 한 곳에 집중 — 외부 코드 변경 시 여기만 수정
    private ShipmentStatus translateStatus(String erpCode) {
        return switch (erpCode) {
            case "DLV"  -> ShipmentStatus.IN_TRANSIT;    // 배송 중
            case "PKG"  -> ShipmentStatus.PREPARING;      // 포장 중
            case "DEL"  -> ShipmentStatus.DELIVERED;      // 배송 완료
            case "RET"  -> ShipmentStatus.RETURNED;       // 반송
            case "FAIL" -> ShipmentStatus.DELIVERY_FAILED; // 배달 실패
            // v2 API: "DELIVERING" 추가 시 여기만 수정
            default -> throw new UnknownLogisticsStatusException(
                "알 수 없는 물류사 상태 코드: " + erpCode + ". 물류사에 확인 필요."
            );
        };
    }

    private LocalDate translateDate(String erpDate) {
        // "20240317" → LocalDate
        return LocalDate.parse(erpDate, DateTimeFormatter.ofPattern("yyyyMMdd"));
    }
}

// 4. Adapter — Port 구현, API 클라이언트 + Translator 조합
@Component
public class LogisticsShipmentAdapter implements ShipmentTrackingPort {

    private final LogisticsApiClient apiClient;
    private final LogisticsTranslator translator;

    @Override
    public Optional<ShipmentTracking> findTracking(TrackingNumber trackingNumber) {
        return apiClient.getShipment(trackingNumber.value())
            .map(translator::toDomain);
    }
}

// 5. 도메인 — 외부 물류사를 전혀 모름
@Service
public class ShipmentApplicationService {

    private final ShipmentTrackingPort trackingPort;  // ACL을 통해 접근

    public ShipmentStatus getStatus(TrackingNumber trackingNumber) {
        return trackingPort.findTracking(trackingNumber)
            .map(ShipmentTracking::status)  // IN_TRANSIT, DELIVERED 등 깨끗한 도메인 언어
            .orElseThrow(() -> new TrackingNotFoundException(trackingNumber));
    }
}
```

---

## 🔬 내부 동작 원리

### 1. ACL의 세 가지 구성 요소

```
ACL 내부 구성:

┌──────────────────────────────────────────────────────────┐
│                   ACL 레이어                              │
│                                                          │
│  ┌─────────────┐    ┌─────────────┐    ┌─────────────┐  │
│  │   Facade    │    │  Translator │    │   Adapter   │  │
│  │ (단순화된   │    │ (언어/모델  │    │ (Port 구현, │  │
│  │  인터페이스) │    │  번역)      │    │  통신 처리) │  │
│  └──────┬──────┘    └──────┬──────┘    └──────┬──────┘  │
│         │                  │                  │          │
└─────────┼──────────────────┼──────────────────┼──────────┘
          ▼                  ▼                  ▼
                   외부 시스템 / API

Facade:
  외부 시스템의 복잡한 API를 단순한 인터페이스로 감춤
  여러 외부 API 호출을 하나의 도메인 연산으로 묶음
  
  예: 물류사의 "배송 조회" + "예상 도착일 조회" 두 API를
      → findShipmentDetail() 하나로 묶기

Translator:
  외부 모델의 개념을 도메인 개념으로 번역
  "DLV" → ShipmentStatus.IN_TRANSIT
  "20240317" → LocalDate.of(2024, 3, 17)
  
  가장 중요한 구성 요소 — 번역 로직이 여기 집중

Adapter:
  도메인의 Port 인터페이스를 구현
  외부 API 클라이언트와 Translator를 연결
  의존성 역전 원칙 구현
```

### 2. 도메인 오염의 구체적 증상

```
오염 증상 1: 외부 코드가 도메인 조건문에 등장

  // 오염된 코드
  if ("B2C".equals(order.getCustType())) { ... }
  if (shipment.getStaCd().startsWith("DLV")) { ... }
  
  → "B2C", "DLV"는 외부 코드이지 도메인 언어가 아님
  → ACL이 없다는 증거

오염 증상 2: 도메인 테스트에 외부 라이브러리 필요

  @Test
  void shipment_isDelivering() {
      LogisticsResponse externalResponse = new LogisticsResponse();
      externalResponse.setStaCd("DLV");  // ← 외부 라이브러리를 테스트에서 직접 사용
      // ...
  }
  
  → 도메인 테스트가 외부 물류사를 알아야 함
  → ACL이 있다면: Shipment shipment = new Shipment(IN_TRANSIT); 로 끝

오염 증상 3: 외부 API 필드명이 도메인 클래스에 사용

  public class Order {
      private String ord_no;     // 외부 ERP 필드명 (언더스코어)
      private String cust_cd;    // 외부 ERP 필드명
      private int sta_cd;        // 외부 ERP 숫자 코드
  }
  
  → 도메인 클래스가 ERP의 DB 구조와 동일해짐

오염 증상 4: 외부 패키지가 도메인 패키지에 import됨

  package com.example.shipping.domain;
  import com.logistics.api.sdk.ShipmentResponse;  // ← 외부 SDK import!
  
  → 도메인 패키지에 외부 의존성 침투
```

### 3. ACL이 방어하는 세 가지 시나리오

```
시나리오 1: 외부 API 버전 업그레이드
  
  물류사 API v1: staCd = "DLV"
  물류사 API v2: staCd = "DELIVERING"
  
  ACL 없음: 코드 전체에서 "DLV" 검색 → 수정 → 테스트 → 배포
  ACL 있음: LogisticsTranslator.translateStatus() 한 줄 수정 → 배포

시나리오 2: 외부 API 공급자 교체

  기존: CJ대한통운 API 사용
  변경: 한진택배 API로 교체 (완전히 다른 모델)
  
  ACL 없음: 물류사 코드가 도메인 전체에 퍼져 있어 대규모 수정
  ACL 있음: 새 Adapter + Translator 구현 → 기존 도메인 코드 무관
            (Port 인터페이스를 구현하는 새 Adapter만 작성)

시나리오 3: 외부 API 장애 격리

  외부 물류사 API 다운 → 우리 서비스는?
  
  ACL 없음: 외부 API 호출이 Service에 직접 → Service가 예외 처리를 알아야 함
  ACL 있음: Adapter에서 FallbackShipmentTracking 반환 (캐시 또는 기본값)
  
  @Override
  public Optional<ShipmentTracking> findTracking(TrackingNumber trackingNumber) {
      try {
          return apiClient.getShipment(trackingNumber.value())
              .map(translator::toDomain);
      } catch (LogisticsApiUnavailableException e) {
          log.warn("물류사 API 장애, 캐시된 추적 정보 반환: {}", trackingNumber);
          return trackingCache.get(trackingNumber);  // Adapter에서 폴백 처리
      }
  }
```

---

## 💻 실전 코드

### ACL 전체 구조 — PG사 결제 연동

```java
// 외부 PG사 SDK 응답 (Toss Payments)
public class TossPaymentResponse {
    private String paymentKey;
    private String orderId;
    private String status;      // "READY", "IN_PROGRESS", "DONE", "CANCELED", "ABORTED"
    private long totalAmount;
    private String method;      // "카드", "가상계좌", "계좌이체"
    private Card card;          // 카드 상세 (null if not card)
}

// 도메인 Port 인터페이스 (도메인 레이어에 위치)
public interface PaymentGatewayPort {
    PaymentResult requestPayment(PaymentRequest request);
    void cancelPayment(PaymentKey paymentKey, Money cancelAmount);
}

// ACL: Translator
@Component
public class TossPaymentTranslator {

    public PaymentResult toDomain(TossPaymentResponse response) {
        return new PaymentResult(
            new PaymentKey(response.getPaymentKey()),
            new OrderId(response.getOrderId()),
            translateStatus(response.getStatus()),
            Money.ofKrw(response.getTotalAmount()),
            translateMethod(response.getMethod())
        );
    }

    private PaymentStatus translateStatus(String tossStatus) {
        return switch (tossStatus) {
            case "DONE"     -> PaymentStatus.APPROVED;
            case "CANCELED" -> PaymentStatus.CANCELLED;
            case "ABORTED"  -> PaymentStatus.FAILED;
            case "READY", "IN_PROGRESS" -> PaymentStatus.PENDING;
            default -> throw new UnknownPaymentStatusException(tossStatus);
        };
    }

    private PaymentMethod translateMethod(String tossMethod) {
        return switch (tossMethod) {
            case "카드"    -> PaymentMethod.CREDIT_CARD;
            case "가상계좌" -> PaymentMethod.VIRTUAL_ACCOUNT;
            case "계좌이체" -> PaymentMethod.BANK_TRANSFER;
            default -> PaymentMethod.OTHER;
        };
    }
}

// ACL: Adapter (Port 구현)
@Component
public class TossPaymentAdapter implements PaymentGatewayPort {

    private final TossPaymentsClient tossClient;
    private final TossPaymentTranslator translator;

    @Override
    public PaymentResult requestPayment(PaymentRequest request) {
        TossPaymentRequest tossRequest = new TossPaymentRequest(
            request.paymentKey().value(),
            request.orderId().value(),
            request.amount().toLong()
        );

        TossPaymentResponse response = tossClient.confirm(tossRequest);
        return translator.toDomain(response);
    }

    @Override
    public void cancelPayment(PaymentKey paymentKey, Money cancelAmount) {
        tossClient.cancel(paymentKey.value(), cancelAmount.toLong(), "고객 요청");
    }
}

// 도메인 Application Service — Toss를 전혀 모름
@Service
@Transactional
public class PaymentApplicationService {

    private final PaymentGatewayPort paymentGatewayPort;  // ACL을 통해 접근

    public void approvePayment(ApprovePaymentCommand command) {
        PaymentRequest request = new PaymentRequest(
            command.paymentKey(),
            command.orderId(),
            command.amount()
        );

        PaymentResult result = paymentGatewayPort.requestPayment(request);
        // result.status()는 APPROVED, CANCELLED, FAILED — 깨끗한 도메인 언어
        // "DONE", "ABORTED" 같은 PG사 언어 없음

        if (result.status() == PaymentStatus.APPROVED) {
            // 결제 완료 처리...
        }
    }
}
```

### ACL 테스트 — 외부 의존 없이

```java
// Translator 단위 테스트 — 외부 API 없이 번역 로직만 검증
class TossPaymentTranslatorTest {

    private final TossPaymentTranslator translator = new TossPaymentTranslator();

    @Test
    void translateStatus_DONE_to_APPROVED() {
        TossPaymentResponse response = createResponse("DONE", 10000L, "카드");
        PaymentResult result = translator.toDomain(response);
        assertThat(result.status()).isEqualTo(PaymentStatus.APPROVED);
    }

    @Test
    void translateStatus_ABORTED_to_FAILED() {
        TossPaymentResponse response = createResponse("ABORTED", 10000L, "카드");
        PaymentResult result = translator.toDomain(response);
        assertThat(result.status()).isEqualTo(PaymentStatus.FAILED);
    }

    @Test
    void translateMethod_virtualAccount() {
        TossPaymentResponse response = createResponse("DONE", 10000L, "가상계좌");
        PaymentResult result = translator.toDomain(response);
        assertThat(result.method()).isEqualTo(PaymentMethod.VIRTUAL_ACCOUNT);
    }
}

// Application Service 테스트 — FakeAdapter로 외부 없이 테스트
class PaymentApplicationServiceTest {

    private final FakePaymentGatewayPort fakeGateway = new FakePaymentGatewayPort();
    private final PaymentApplicationService sut = new PaymentApplicationService(fakeGateway);

    @Test
    void approvePayment_success() {
        fakeGateway.willSucceed();  // 가짜 성공 응답 설정
        sut.approvePayment(approveCommand());
        assertThat(paymentRepository.findLatest().status()).isEqualTo(PaymentStatus.APPROVED);
    }
}

// 가짜 Adapter (테스트용)
class FakePaymentGatewayPort implements PaymentGatewayPort {
    private boolean willSucceed = true;

    public void willSucceed() { this.willSucceed = true; }
    public void willFail() { this.willSucceed = false; }

    @Override
    public PaymentResult requestPayment(PaymentRequest request) {
        if (willSucceed) {
            return new PaymentResult(request.paymentKey(), request.orderId(),
                PaymentStatus.APPROVED, request.amount(), PaymentMethod.CREDIT_CARD);
        }
        return new PaymentResult(..., PaymentStatus.FAILED, ...);
    }
}
```

---

## 📊 설계 비교

```
ACL 없음 vs ACL 있음:

                ACL 없음                   ACL 있음
────────────┼────────────────────────┼────────────────────────────
외부 변경 시  │ 도메인 코드 전체 수정    │ ACL(Translator)만 수정
────────────┼────────────────────────┼────────────────────────────
외부 교체 시  │ 대규모 리팩터링          │ 새 Adapter 구현
            │                        │ 기존 도메인 코드 무관
────────────┼────────────────────────┼────────────────────────────
도메인 테스트 │ 외부 SDK/API 필요        │ FakeAdapter로 독립 테스트
────────────┼────────────────────────┼────────────────────────────
도메인 언어  │ 외부 코드 침투           │ 깨끗한 도메인 언어 유지
            │ ("DLV", "B2C" 등)      │ (IN_TRANSIT, CONSUMER 등)
────────────┼────────────────────────┼────────────────────────────
초기 구현   │ 빠름                    │ 느림 (ACL 레이어 추가)
────────────┼────────────────────────┼────────────────────────────
장기 유지보수 │ 외부 의존도 높아 어려움  │ 도메인 독립적으로 유지
```

---

## ⚖️ 트레이드오프

```
ACL의 비용:
  ① 추가 레이어 = 추가 코드
     Port 인터페이스 + Adapter + Translator 세 클래스
     단순한 외부 API 연동에 과잉처럼 느껴질 수 있음
  
  ② 번역 로직 유지 비용
     외부 코드가 추가될 때마다 Translator 수정
     번역 실수 발생 가능 (테스트로 방어)
  
  ③ 성능 오버헤드
     외부 응답 → 도메인 객체 변환 비용
     대부분 무시 가능한 수준 (ns 단위)

ACL이 과잉인 경우:
  외부 API가 우리 도메인에 완벽히 맞는 경우
  → Conformist로 직접 사용 (번역 불필요)
  
  한 번만 호출하고 사용이 끝나는 경우
  → 단순 DTO 매핑으로 충분

ACL이 필수인 경우:
  외부 모델이 이상하거나 도메인과 다름
  외부 시스템이 자주 바뀌거나 교체 가능성 있음
  도메인 테스트에 외부 의존이 생기는 경우
  레거시 시스템 통합
```

---

## 📌 핵심 정리

```
ACL 핵심:

역할: 외부 모델이 도메인 모델을 오염시키지 않도록 하는 번역 레이어

구성 요소:
  Port (인터페이스): 도메인이 필요로 하는 것을 추상화
  Translator: 외부 모델 ↔ 도메인 모델 번역 (핵심)
  Adapter: Port 구현 + 외부 API 클라이언트 + Translator 연결
  Facade: 복잡한 외부 API를 단순한 인터페이스로 감춤

ACL이 방어하는 것:
  외부 코드("DLV", "B2C")가 도메인 if/else에 침투
  외부 API 변경 시 도메인 코드 수정 연쇄
  도메인 테스트에 외부 SDK 필요
  외부 시스템 교체 시 대규모 리팩터링

테스트 이점:
  도메인 테스트: FakeAdapter로 외부 없이 실행
  Translator 테스트: 번역 로직만 단독 검증
  Adapter 테스트: 실제 외부 API와의 통합 테스트 (소수)

선택 기준:
  외부 모델이 도메인과 다름 → ACL 필수
  외부 API가 잘 설계되고 안정적 → Conformist 고려
```

---

## 🤔 생각해볼 문제

**Q1.** 다음 코드에서 도메인 오염의 증상을 찾고, ACL로 어떻게 개선할 수 있는가?

```java
@Service
public class MemberService {

    private final NaverApiClient naverApiClient;

    public Member loginWithNaver(String authCode) {
        NaverUserResponse naverUser = naverApiClient.getUser(authCode);

        // Naver API 응답 구조가 도메인에 침투
        Member member = memberRepository.findByExternalId("NAVER_" + naverUser.getId())
            .orElseGet(() -> {
                Member newMember = new Member();
                newMember.setExternalId("NAVER_" + naverUser.getId());
                newMember.setEmail(naverUser.getEmail());
                // naverUser.getAge()는 "20-29" 형식의 문자열
                newMember.setAgeGroup(naverUser.getAge());  // "20-29"가 도메인에 들어옴
                newMember.setGenderCode(naverUser.getGender()); // "M", "F"
                return memberRepository.save(newMember);
            });
        return member;
    }
}
```

<details>
<summary>해설 보기</summary>

**오염 증상:**
1. `"NAVER_" + naverUser.getId()` — Naver ID 형식이 도메인 코드에 침투
2. `"20-29"` 형식의 나이 문자열이 도메인 필드에 저장됨
3. `"M"`, `"F"` 같은 외부 성별 코드가 도메인에 침투
4. MemberService가 NaverApiClient를 직접 의존 → 카카오 로그인 추가 시 카카오도 Service에 추가됨

**ACL로 개선:**

```java
// Port (도메인 레이어)
public interface OAuthPort {
    OAuthUserProfile getUserProfile(OAuthCode code, OAuthProvider provider);
}

// Translator
@Component
public class NaverOAuthTranslator {
    public OAuthUserProfile toDomain(NaverUserResponse response) {
        return new OAuthUserProfile(
            new OAuthId(OAuthProvider.NAVER, response.getId()),
            response.getEmail(),
            translateAgeGroup(response.getAge()),   // "20-29" → AgeGroup.TWENTIES
            translateGender(response.getGender())   // "M" → Gender.MALE
        );
    }

    private AgeGroup translateAgeGroup(String naverAge) {
        return switch (naverAge) {
            case "0-9"   -> AgeGroup.UNDER_TEN;
            case "20-29" -> AgeGroup.TWENTIES;
            default -> AgeGroup.UNKNOWN;
        };
    }
}

// Application Service — Naver를 모름
public Member loginWithOAuth(OAuthCode code, OAuthProvider provider) {
    OAuthUserProfile profile = oAuthPort.getUserProfile(code, provider);
    // profile.ageGroup()은 AgeGroup.TWENTIES — 깨끗한 도메인 언어
}
```

</details>

---

**Q2.** ACL의 Translator가 너무 복잡해지고 있다. 외부 API의 응답이 100개 필드이고, 도메인 모델로 70개 필드를 번역해야 할 때 어떻게 관리하는가?

<details>
<summary>해설 보기</summary>

**전략 1: 번역을 여러 메서드로 분리**
```java
public class LargeErpTranslator {
    public Order toDomain(ErpResponse response) {
        return new Order(
            translateBasicInfo(response),
            translatePricing(response),
            translateShipping(response),
            translateStatus(response)
        );
    }

    private OrderBasicInfo translateBasicInfo(ErpResponse r) { ... }
    private OrderPricing translatePricing(ErpResponse r) { ... }
}
```

**전략 2: 필요한 것만 번역 (YAGNI)**
외부 API의 100개 필드 중 실제로 도메인에서 사용하는 것만 번역. 나머지는 무시. "지금 필요한 10개만" 번역하고, 필요해질 때 추가.

**전략 3: MapStruct 활용**
```java
@Mapper(componentModel = "spring")
public interface LogisticsMapper {
    @Mapping(source = "staCd", target = "status", qualifiedByName = "translateStatus")
    ShipmentTracking toDomain(LogisticsResponse response);

    @Named("translateStatus")
    default ShipmentStatus translateStatus(String staCd) { ... }
}
```

복잡한 번역 로직은 `@Named` 메서드로 명시적으로 처리하고, 단순 필드 매핑은 MapStruct에 위임합니다.

</details>

---

**Q3.** ACL과 일반 Service 레이어의 경계가 모호해질 때 어떻게 구분하는가? ACL에서 비즈니스 로직이 실행되는 것은 허용되는가?

<details>
<summary>해설 보기</summary>

**ACL에는 번역 로직만, 비즈니스 로직은 도메인으로.**

```
ACL (허용):
  ✅ 외부 코드 → 도메인 코드 번역 (staCd "DLV" → IN_TRANSIT)
  ✅ 외부 날짜 형식 → 도메인 날짜 형식 변환
  ✅ 외부 API 장애 시 폴백 (캐시 반환)
  ✅ 외부 API 호출 재시도 (기술적 처리)

ACL (금지):
  ❌ "외부에서 받은 금액이 0이면 무료 배송 처리" — 비즈니스 규칙
  ❌ "외부 상태가 DLV이면 알림 발송" — 비즈니스 흐름
  ❌ 데이터베이스 직접 접근

판단 기준:
  "이 로직이 외부 시스템을 모르더라도 여전히 필요한가?"
  → YES: 비즈니스 로직 → 도메인으로
  → NO (번역에만 필요): ACL에 위치 가능
```

ACL에 비즈니스 로직이 들어가는 순간 도메인 테스트에서 ACL을 같이 테스트해야 하고, ACL이 비대해집니다. "ACL은 번역기, 비즈니스는 도메인"이 원칙입니다.

</details>

---

<div align="center">

**[⬅️ 이전: Context Map 패턴 완전 분해](./04-context-map-patterns.md)** | **[홈으로 🏠](../README.md)** | **[다음: 이벤트 스토밍 ➡️](./06-event-storming.md)**

</div>
