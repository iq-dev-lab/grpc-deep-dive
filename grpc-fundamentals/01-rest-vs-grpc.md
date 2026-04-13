# REST vs gRPC — 왜 REST가 마이크로서비스에서 한계를 보이는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- REST가 단일 서버 애플리케이션에서는 충분한데 마이크로서비스에서 왜 문제가 되는가?
- JSON 직렬화 비용과 HTTP/1.1 연결 오버헤드는 실제로 얼마나 영향을 주는가?
- 스키마 계약이 없는 REST API가 Breaking Change를 일으키는 구체적인 시나리오는 무엇인가?
- gRPC가 해결하는 문제는 무엇이고, 어떤 비용을 치르는가?
- "REST 대신 굳이 gRPC를?"에 대한 근거 있는 판단은 어떻게 내리는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

서비스가 하나일 때 REST는 단순하고 직관적이다. 하지만 서비스가 10개, 20개로 늘어나면 서비스 간 통신이 시스템의 복잡도를 결정한다. REST를 마이크로서비스에서 쓸 때 발생하는 문제는 갑자기 터지지 않는다. 서비스가 늘어날수록 서서히 쌓이다가, Breaking Change가 프로덕션에서 터지거나 응답 시간이 누적 지연으로 올라가면서 드러난다. 이 문서는 그 임계점이 어디인지, 그리고 gRPC가 어떤 트레이드오프로 그 문제를 해결하는지를 다룬다.

---

## 😱 흔한 실수 (Before — REST로만 생각하는 접근)

```
상황: 주문 서비스가 상품 서비스의 REST API를 호출함

// 상품 서비스 응답 (v1)
{
  "productId": "prod-1",
  "name": "노트북",
  "price": 1500000
}

6개월 후, 상품 서비스 팀이 필드명을 수정함:
{
  "product_id": "prod-1",    // camelCase → snake_case 변경
  "productName": "노트북",    // name → productName 변경
  "priceKrw": 1500000        // price → priceKrw 변경
}

결과:
  주문 서비스: order.getProduct().getPrice()  → NullPointerException
  결제 서비스: payment.getProductId()         → null
  배송 서비스: shipping.getName()             → null

  세 팀이 모두 상품 서비스 변경을 몰랐음
  계약서가 없었기 때문 — 그냥 문서(Swagger/Notion)만 있었음
  문서는 오래됐고 코드와 달랐음

또 다른 흔한 실수:
  호출마다 HTTP Connection을 새로 맺음 (HTTP/1.1 기본 동작)
  → 주문 처리 1건 = 상품 조회 + 재고 확인 + 결제 검증 + 배송 주소 검증
    각각 별도 TCP 연결 → TLS 핸드쉐이크 × 4
    10ms × 4 = 40ms 추가 지연 (비즈니스 로직 외 순수 연결 비용)

REST가 나쁜 게 아님:
  "REST를 쓰지 마라"가 아니라
  "REST의 어떤 특성이 MSA에서 어떤 문제를 만드는지 알고 선택해야 한다"
```

---

## ✨ 올바른 접근 (After — gRPC 원리를 알고 선택하는 접근)

```
gRPC + Protocol Buffers로 계약을 코드로 고정:

// order_service.proto — 계약서
service ProductService {
  rpc GetProduct(GetProductRequest) returns (GetProductResponse);
}

message GetProductRequest {
  string product_id = 1;  // 필드 번호 1 — 이것이 실제 계약
}

message GetProductResponse {
  string product_id = 1;
  string name = 2;
  int64 price_krw = 3;
}

상품 서비스 팀이 필드명을 바꾸려면:
  .proto 파일을 수정 → protoc 컴파일 시 오류 발생 (기존 필드 번호 변경 시)
  또는 buf breaking 실행 → CI에서 Breaking Change 감지 후 배포 차단

주문 서비스는 컴파일 타임에 타입 불일치를 발견:
  productService.getProduct(request).getName()  // 컴파일 오류로 즉시 발견
  런타임이 아닌 빌드 타임에 계약 불일치 탐지

연결 오버헤드 감소:
  gRPC는 HTTP/2 기반 — 하나의 TCP 연결 위에 여러 RPC 병렬 처리
  주문 처리 1건 = 상품 조회 + 재고 + 결제 + 배송 주소
    → 단일 Channel 재사용 → TCP/TLS 핸드쉐이크 1회
    → 나머지 3개 RPC는 동일 연결 재사용
```

---

## 🔬 내부 동작 원리

### 1. REST API의 느슨한 계약 문제

```
REST의 API 계약 현실:

클라이언트가 아는 것:
  URL: /api/v1/products/{id}
  Method: GET
  응답: JSON (구조는 문서 참고)

문서(Swagger) 현실:
  ① 문서가 최신이 아닌 경우가 많음 (코드는 바뀌었지만 문서는 6개월 전)
  ② 타입 정보가 느슨함 ("price": 숫자 — 원? 달러? int? float?)
  ③ 필드 null 가능 여부를 정확히 표현하기 어려움
  ④ 컴파일러가 계약 위반을 잡아주지 않음

JSON의 특성:
  {
    "price": 1500000    // int인가? float인가? string인가?
  }
  
  → JavaScript는 암묵 변환 → 문제가 안 보임
  → Java의 @JsonProperty("price") double price → 의도치 않은 타입 확장
  → 서버가 int, 클라이언트가 float로 받아도 오류 없음 → 정밀도 손실 잠복

REST API 버전 관리의 현실:
  /api/v1/products  →  /api/v2/products
  → v1 클라이언트를 언제까지 지원? 누가 v1 쓰는지 알 수 있나?
  → v1을 끊으면 어떤 팀에 영향이 가나?
  → 직접 추적해야 하고, 도구 지원이 없음
```

### 2. HTTP/1.1의 연결 오버헤드

```
HTTP/1.1 기본 동작 — 요청당 연결:

클라이언트 → 서버 TCP 연결 수립 (3-way handshake):
  SYN        →
             ← SYN-ACK
  ACK        →
  (왕복 1회 = 1 RTT)

HTTPS라면 TLS 핸드쉐이크 추가:
  TLS 1.3: 1 RTT 추가 (TLS 1.2: 2 RTT)
  인증서 교환, 키 합의
  → 총 2~3 RTT 소요 (로컬: ~0.1ms × 2 = 0.2ms, 다른 AZ: 5ms × 2 = 10ms)

Connection: keep-alive 로 연결 재사용 가능하지만:
  → 유휴 타임아웃 후 연결 끊김 (Nginx 기본 75초)
  → 프록시/로드밸런서를 거치면 더 짧아짐
  → 재연결 시 다시 핸드쉐이크

HTTP/1.1 Head-of-Line Blocking:
  
  연결 1:  [요청A ──────────] [요청B ──────────]
           (A 완료 전까지 B는 대기)
  
  해결: 연결을 여러 개 맺음 (브라우저: 도메인당 6개)
  → 서버 입장: 클라이언트 × 연결 수 = 수천 개의 TCP 연결
  → 파일 디스크립터, 메모리 소비

MSA 환경에서의 실제 연결 비용 계산:
  서비스 10개, 각 서비스가 3개 서비스 호출
  → 서비스당 최소 3개 외부 연결 유지 필요
  → 시스템 전체: 10 × 3 = 30개 연결 그룹
  → 각 연결 재수립 시 RTT × 2~3회 비용
  
  초당 1,000 요청 처리 시:
    REST(연결 재수립): 1,000 × 2ms(TLS) = 2,000ms 누적 지연
    gRPC(연결 재사용): 0ms (이미 수립된 HTTP/2 스트림)
```

### 3. JSON 직렬화 비용

```
JSON 직렬화의 비용 구조:

직렬화 과정:
  Java 객체 → JSON 문자열 → 바이트 배열 → 네트워크 전송
  
  Java 객체 → JSON 변환 비용:
    Reflection 사용 → 필드 이름을 런타임에 읽음
    각 필드: getClass().getDeclaredFields() → 이름 조회 → toString()
    
  네트워크 페이로드:
    {"productId":"prod-1","name":"노트북","price":1500000,"category":"electronics","inStock":true}
    → 90바이트 (UTF-8)
    → 필드 이름이 데이터 크기에 포함됨
    → "productId" = 11바이트, 실제 데이터 "prod-1" = 6바이트
       키가 값보다 큰 경우 다수

Protobuf 직렬화:
  message ProductResponse {
    string product_id = 1;
    string name = 2;
    int64 price = 3;
    string category = 4;
    bool in_stock = 5;
  }
  
  직렬화 결과 (16진수):
    0a 06 70 72 6f 64 2d 31   ← product_id: "prod-1" (8바이트)
    12 06 eb85b8ed8ab8eb82a9  ← name: "노트북" (8바이트, UTF-8 3바이트×3)
    18 c0 d6 5b              ← price: 1500000 (Varint, 4바이트)
    22 0b 65 6c 65 63...     ← category: "electronics" (13바이트)
    28 01                   ← in_stock: true (2바이트)
  
  총 약 35~40바이트 vs JSON 90바이트 → 55% 이상 절감
  
  직렬화 속도:
    Protobuf: 필드 번호 → 비트 연산 → 바이트 직접 기록 (Reflection 없음)
    JSON: Reflection + toString() + 이스케이핑 처리
    → 직렬화 속도 3~5배 차이 (벤치마크 환경에 따라 다름)
```

### 4. gRPC가 해결하는 것과 치르는 비용

```
gRPC가 해결하는 문제:

① 타입 안전한 계약 (.proto → 컴파일 타임 검증)
② 직렬화 효율 (Protobuf → JSON 대비 크기/속도 개선)
③ 연결 재사용 (HTTP/2 → 단일 Channel로 다중 RPC)
④ 스트리밍 지원 (Server/Client/Bidirectional Streaming 내장)
⑤ 코드 자동 생성 (protoc → 클라이언트 Stub, 서버 Skeleton)

gRPC가 치르는 비용:

① 브라우저 직접 호출 불가
   → HTTP/2의 Trailer 헤더를 브라우저가 지원하지 않음
   → gRPC-Web + 프록시 필요 (Envoy, grpc-gateway)
   → REST는 브라우저에서 fetch()로 바로 호출 가능

② 디버깅/테스트 불편
   → JSON은 curl로 바로 확인 가능
   → Protobuf 바이너리는 grpcurl 또는 별도 도구 필요
   → Wireshark 캡처도 Protobuf 디코더 없이는 읽기 어려움

③ 학습 곡선
   → .proto 문법, protoc 설정, Stub 사용법 학습 필요
   → Spring + gRPC 통합 설정이 REST보다 복잡

④ 생태계
   → REST: 모든 언어/프레임워크에서 기본 지원
   → gRPC: 지원하는 언어/런타임 목록 확인 필요 (PHP, Ruby 지원 제한적)

선택 기준:
  gRPC가 유리한 경우:
    서비스 간 내부 통신 (브라우저 직접 호출 불필요)
    대용량 데이터, 고빈도 호출 (직렬화 비용이 민감)
    실시간 스트리밍 필요 (주식 시세, 로그 스트림)
    다언어 팀 (Java, Go, Python이 동일 .proto 공유)

  REST가 유리한 경우:
    외부 공개 API (브라우저, 모바일 앱의 직접 호출)
    디버깅/개발 편의성이 중요한 초기 단계
    이미 REST 인프라가 잘 갖춰진 경우
    클라이언트 언어/환경이 gRPC를 지원하지 않는 경우
```

---

## 💻 실전 실험

### 실험 1: JSON vs Protobuf 직렬화 크기 비교

```bash
# Docker 환경 시작
docker compose up -d grpc-server

# JSON REST 응답 크기 측정
curl -s http://localhost:8080/api/v1/products/prod-1 | wc -c
# 예시 출력: 312 (바이트)

# 동일 데이터를 gRPC로 호출, 응답 크기 확인
grpcurl -plaintext localhost:9090 \
  product.v1.ProductService/GetProduct \
  -d '{"product_id": "prod-1"}' \
  -v 2>&1 | grep "Response-Size\|Content-Length"

# grpcurl 없다면 설치
brew install grpcurl        # macOS
# 또는
go install github.com/fullstorydev/grpcurl/cmd/grpcurl@latest

# 서비스 목록 조회 (Reflection 활성화 시)
grpcurl -plaintext localhost:9090 list
# 출력 예시:
# grpc.reflection.v1alpha.ServerReflection
# product.v1.ProductService
# order.v1.OrderService

# 서비스 구조 확인
grpcurl -plaintext localhost:9090 describe product.v1.ProductService
```

### 실험 2: HTTP/1.1 vs HTTP/2 연결 재사용 비교

```bash
# HTTP/1.1 연결 수 확인 (REST)
# 10개 동시 요청 보내기
for i in $(seq 1 10); do
  curl -s http://localhost:8080/api/v1/products/prod-$i &
done
wait

# 서버에서 연결 수 확인 (Linux)
ss -tn | grep :8080 | grep ESTABLISHED | wc -l
# 예시: 10 (요청당 별도 연결)

# HTTP/2 연결 수 확인 (gRPC)
# grpc-client 서비스에서 10개 RPC 동시 호출
curl http://localhost:8081/bench/concurrent?count=10

# 서버에서 연결 수 확인
ss -tn | grep :9090 | grep ESTABLISHED | wc -l
# 예시: 1 (단일 연결에서 10개 스트림)
```

### 실험 3: Breaking Change 감지 (buf CLI)

```bash
# buf CLI 설치
brew install bufbuild/buf/buf

# 기존 .proto 기준으로 스냅샷 저장
buf breaking --against .git#branch=main

# 필드 번호를 바꾸거나 필드를 삭제하면:
# 예: product_id = 1 을 삭제하고 id = 1 로 변경

buf breaking --against .git#branch=main
# 출력 예시:
# proto/product/v1/product.proto:5:3:Field "1" on message "GetProductRequest" changed name from "product_id" to "id".
# → CI에서 Breaking Change 자동 감지 가능
```

---

## 📊 성능/비용 비교

```
REST vs gRPC 직렬화 비교 (같은 데이터):

데이터: 상품 정보 10개 필드, 상품명(한국어), 가격, 카테고리 등

직렬화 크기:
  JSON:        약 280~350 바이트
  Protobuf:    약 120~160 바이트
  절감:        약 50~55%

직렬화 속도 (Java, JMH 벤치마크 참고치):
  JSON (Jackson):   약 400~600 ns/op
  Protobuf:         약 100~200 ns/op
  차이:             3~5배

연결 수립 비용 (AWS 동일 VPC, TLS 1.3):
  HTTP/1.1 신규 연결:  약 1~2 ms (TCP + TLS)
  HTTP/2 스트림 추가:  약 0.01 ms (이미 수립된 연결 재사용)
  
초당 1,000 요청 처리 시 연결 비용:
  REST(매 요청 재연결): 1,000 × 1.5ms = 1,500ms 누적
  gRPC(연결 재사용):   1,000 × 0.01ms = 10ms 누적

주의: 위 수치는 참고치. 실제 환경(네트워크 RTT, 페이로드 크기, JVM 상태)에 따라 다름
```

---

## ⚖️ 트레이드오프

```
REST의 장단점:

장점:
  ① 브라우저 직접 호출 — fetch(), axios 모두 지원
  ② 디버깅 용이 — curl, Postman으로 즉시 테스트
  ③ 범용 생태계 — 모든 언어, 프레임워크, 인프라 지원
  ④ 낮은 학습 곡선 — HTTP 메서드, URL, JSON은 누구나 앎

단점:
  ① 느슨한 계약 — 런타임 전까지 Breaking Change 모름
  ② 직렬화 비용 — 대용량·고빈도 환경에서 누적
  ③ 연결 오버헤드 — HTTP/1.1 재연결, HTTP/2는 스트리밍 제한적
  ④ 스트리밍 불편 — SSE/WebSocket 별도 구현 필요

gRPC의 장단점:

장점:
  ① 타입 안전한 계약 — 컴파일 타임에 Breaking Change 감지
  ② 효율적 직렬화 — Protobuf로 크기/속도 개선
  ③ 연결 재사용 — HTTP/2 단일 Channel, 멀티플렉싱
  ④ 스트리밍 내장 — 4가지 패턴 기본 지원
  ⑤ 코드 자동 생성 — 클라이언트/서버 코드를 proto에서 생성

단점:
  ① 브라우저 미지원 — gRPC-Web + Proxy 필요
  ② 디버깅 불편 — 바이너리 페이로드, 별도 도구 필요
  ③ 학습 곡선 — proto 문법, protoc, Channel/Stub 개념
  ④ 생태계 제한 — 일부 언어/플랫폼 지원 미흡

선택 판단 기준:
  "서비스 간 내부 통신" + "고빈도/대용량" + "다언어 팀"
  → gRPC 적합

  "외부 API" + "브라우저 클라이언트" + "단순 CRUD"
  → REST 적합

  "두 가지 다 필요"
  → gRPC 내부 + gRPC-Gateway로 REST 외부 노출
```

---

## 📌 핵심 정리

```
REST vs gRPC 핵심:

REST의 MSA 한계:
  ① 스키마 계약 없음 → 런타임 Breaking Change
  ② JSON 직렬화 비용 → 대용량·고빈도 환경에서 누적
  ③ HTTP/1.1 연결 오버헤드 → 서비스 간 호출마다 재연결 비용
  ④ 스트리밍 미지원 → 별도 프로토콜(SSE, WebSocket) 필요

gRPC가 해결하는 방식:
  ① .proto 파일 → 타입 안전한 계약 → 컴파일 타임 검증
  ② Protobuf → TLV 인코딩 → JSON 대비 크기/속도 개선
  ③ HTTP/2 Channel → 단일 연결로 다중 RPC 멀티플렉싱
  ④ 4가지 스트리밍 패턴 내장

gRPC를 선택할 때 치르는 비용:
  브라우저 미지원, 디버깅 불편, 학습 곡선
  → 서비스 간 내부 통신에서는 이 비용이 작음
  → 외부 API에서는 gRPC-Gateway로 REST 병행

판단 기준:
  "REST로 하면 어떤 문제가 생기는가"를 먼저 파악하고
  그 문제가 gRPC 도입 비용보다 클 때 전환
```

---

## 🤔 생각해볼 문제

**Q1.** REST API에서 `Content-Type: application/json`과 `Content-Type: application/x-protobuf`를 지원하면 gRPC 없이도 Protobuf 직렬화 이점을 얻을 수 있는가? 그렇다면 gRPC의 추가적인 가치는 무엇인가?

<details>
<summary>해설 보기</summary>

REST API에서 Protobuf를 Content-Type으로 사용하면 직렬화 크기/속도는 개선됩니다. 실제로 일부 팀은 이 방식을 쓰기도 합니다.

그러나 gRPC는 직렬화만이 아닙니다:

- **HTTP/2 멀티플렉싱**: REST over HTTP/2도 가능하지만 대부분 실제 구현은 HTTP/1.1
- **스트리밍**: HTTP/1.1 기반 REST에서 Server Streaming은 SSE, Client Streaming은 WebSocket 등 별도 메커니즘
- **타입 안전한 코드 생성**: `.proto` → Stub 자동 생성 → 컴파일 타임 타입 검증
- **Deadline 전파**: 호출 체인 전체에 타임아웃 자동 전파 (Context Propagation)
- **Interceptor 체인**: 인증/로깅/추적을 미들웨어로 구성하는 표준화된 방법

"Protobuf over REST"는 직렬화 문제만 해결합니다. gRPC는 그 위에 HTTP/2 멀티플렉싱 + 스트리밍 + 코드 계약 + 미들웨어 체계를 통합 제공합니다.

</details>

---

**Q2.** 팀이 REST API를 이미 잘 운영 중이라면, Breaking Change는 API 버저닝(`/v1`, `/v2`)으로 충분히 해결되는 것 아닌가? gRPC의 `.proto` 계약이 더 나은 이유는?

<details>
<summary>해설 보기</summary>

URL 버저닝(`/v1`, `/v2`)의 한계:

- **누가 v1을 쓰는지 추적이 어렵습니다**: 내부 서비스 10개 중 어느 팀이 v1을 아직 쓰는지 파악하려면 별도 추적이 필요합니다.
- **v1을 언제 끊을 수 있는지 불명확합니다**: 클라이언트 모두가 v2로 이전할 때까지 v1을 계속 유지해야 하는데, 이 타이밍을 코드로 강제할 수 없습니다.
- **필드 수준 변경은 버전으로 해결이 안 됩니다**: `/v1/products`에서 응답의 `price` 필드를 `priceKrw`로 바꾸면 v1 클라이언트가 깨집니다.

`.proto`의 장점:

- **필드 번호 불변 계약**: 필드 이름은 바꿀 수 있어도 번호는 바꾸면 컴파일 오류 또는 `buf breaking` 차단
- **패키지 버전 관리**: `package order.v1`을 `package order.v2`로 분리하면 두 버전을 동시에 컴파일 가능
- **Consumer-Driven Testing**: 각 서비스가 자신이 소비하는 필드를 테스트로 명세하면, 공급자 변경 시 소비자 테스트가 자동으로 실패

</details>

---

**Q3.** gRPC가 HTTP/2를 쓰기 때문에 항상 HTTP/1.1보다 빠른가? gRPC가 REST보다 느릴 수 있는 시나리오는 무엇인가?

<details>
<summary>해설 보기</summary>

gRPC가 느릴 수 있는 시나리오:

- **페이로드가 매우 작고 호출 빈도가 낮을 때**: Protobuf 인코딩/디코딩 오버헤드가 직렬화 절감보다 클 수 있습니다 (단, 대부분의 실무에서는 무시할 수준)
- **HTTP/1.1 + Keep-Alive + 연결 풀이 잘 구성된 경우**: 연결이 이미 재사용되고 있다면 HTTP/2 전환 이점이 줄어듭니다
- **JVM 워밍업 전 초기 요청**: gRPC Channel 수립 + Protobuf 클래스 초기화 비용이 처음 몇 개 요청에서 나타납니다
- **gRPC 메타데이터 오버헤드**: 모든 RPC에 헤더(`:method`, `:path`, `content-type`, `grpc-timeout` 등) 포함 — 페이로드가 아주 작으면 헤더 비중이 큼

결론: gRPC가 일반적으로 더 효율적이지만 "항상 빠르다"는 단언은 위험합니다. 벤치마크는 실제 서비스 환경(페이로드 크기, 호출 빈도, 네트워크 토폴로지)에서 직접 측정해야 합니다.

</details>

---

<div align="center">

**[홈으로 🏠](../README.md)** | **[다음: gRPC 핵심 구성 요소 ➡️](./02-grpc-core-components.md)**

</div>
