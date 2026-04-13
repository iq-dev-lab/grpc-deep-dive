# HTTP/2 기반 통신 — Frame, Stream, Multiplexing

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- HTTP/2의 Frame, Stream, Message 세 계층은 각각 무엇이고 어떻게 연결되는가?
- HTTP/1.1 Head-of-Line Blocking은 정확히 어떤 상황에서 발생하고, HTTP/2는 어떻게 없애는가?
- 하나의 TCP 연결에서 여러 gRPC 호출이 어떻게 동시에 처리되는가?
- gRPC 요청이 HTTP/2 레벨에서 어떤 헤더와 데이터로 전송되는가?
- Wireshark로 HTTP/2 Frame을 직접 관찰하면 무엇이 보이는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC 성능 이점의 절반은 HTTP/2 멀티플렉싱에서 온다. 이 원리를 모르면 "gRPC Channel을 여러 개 만들어야 성능이 나오지 않나?"라는 오해를 하거나, 서버 측 `maxConcurrentCallsPerConnection` 설정이 무슨 의미인지 파악하지 못한다. HTTP/2의 Frame 구조를 알면 gRPC 성능 문제의 절반은 진단할 수 있다.

---

## 😱 흔한 실수 (Before — HTTP/1.1 사고방식으로 gRPC를 보는 접근)

```
실수 1: HTTP/1.1처럼 연결 풀을 만들려고 Channel 여러 개 생성

// ❌ HTTP/1.1 Connection Pool 사고방식을 gRPC에 적용
List<ManagedChannel> channelPool = new ArrayList<>();
for (int i = 0; i < 10; i++) {
    channelPool.add(ManagedChannelBuilder
        .forAddress("product-service", 9090)
        .usePlaintext()
        .build());
}
// "동시 요청을 위해 연결 10개 미리 만들자"

문제:
  HTTP/2 단일 Channel은 이미 내부적으로 멀티플렉싱 지원
  10개의 동시 RPC = 1개의 Channel로 10개의 스트림
  → Channel 10개를 만들면 TCP 연결 10개 = 불필요한 리소스 낭비
  → 서버에 불필요한 연결 부하

실수 2: gRPC 응답이 느린 이유를 "패킷 손실" 하나로 진단

  증상: 가끔 gRPC 응답이 수백 ms 지연됨
  잘못된 진단: "네트워크 패킷 손실 → 재전송 → 지연"
  
  실제 원인 후보:
    HTTP/2 FLOW CONTROL — 수신 측 윈도우가 꽉 찬 상태
    → 보내는 쪽이 WINDOW_UPDATE 받을 때까지 대기
    이를 패킷 손실로 오인
  
  또는: maxConcurrentStreams 초과
    서버가 동시 스트림 최대 100개 설정
    101번째 RPC → 기존 스트림이 완료될 때까지 대기
    → 특정 시간대에 집중되는 지연

실수 3: HTTP/2의 헤더 압축을 모르고 과도한 메타데이터 추가

  // 매 RPC마다 수백 바이트짜리 JWT 전체를 메타데이터로 전송
  Metadata metadata = new Metadata();
  metadata.put(AUTHORIZATION_HEADER, "Bearer " + veryLongJwtToken);
  stub.withInterceptors(MetadataUtils.attachHeaders(metadata));
  
  실제: HTTP/2 HPACK 헤더 압축 → 반복 헤더는 인덱스로 대체
  → 같은 JWT라면 두 번째 요청부터는 헤더 크기 대폭 감소
  → 메타데이터 크기를 과도하게 걱정할 필요 없음 (단, 첫 요청은 전체 전송)
```

---

## ✨ 올바른 접근 (After — HTTP/2 원리를 알고 난 설계)

```
단일 Channel, 충분한 동시성:

// ✅ 단일 Channel로 수백 개의 동시 RPC 처리
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("product-service", 9090)
    .usePlaintext()
    .build();

// 10개의 동시 RPC — 모두 동일 Channel 사용
// 내부적으로 Stream ID 1, 3, 5, 7, 9, 11, 13, 15, 17, 19 각각 할당
// 하나의 TCP 연결 위에서 병렬 처리

// 다만 서버가 여러 인스턴스인 경우:
// → NameResolver로 여러 서버 주소 해석
// → LoadBalancer가 요청을 각 서버의 Channel로 분산
// → 인스턴스당 Channel 1개 (총 N개)
```

---

## 🔬 내부 동작 원리

### 1. HTTP/1.1 Head-of-Line Blocking의 실체

```
HTTP/1.1 파이프라이닝 (이론):

클라이언트 → 서버:
  [요청A] [요청B] [요청C]   ← 한 번에 여러 요청 전송
  
서버 → 클라이언트:
  [응답A] [응답B] [응답C]   ← 요청 순서대로 응답해야 함

Head-of-Line Blocking:
  요청A 처리: 100ms
  요청B 처리: 10ms
  요청C 처리: 5ms
  
  응답 순서:
    응답A: t=100ms (정상)
    응답B: t=100ms (B는 10ms에 끝났지만 A 완료까지 대기)
    응답C: t=100ms (C는 5ms에 끝났지만 A 완료까지 대기)
  
  → B, C는 이미 완료됐는데 A를 기다려야 함
  → "앞 차가 막히면 뒤 차도 못 감"

HTTP/1.1의 현실적 해결책:
  브라우저: 도메인당 최대 6개 TCP 연결 동시 유지
  서버: Keep-Alive로 연결 재사용하지만 직렬 처리
  → 동시성을 연결 수로 해결 → 연결 관리 비용 증가

REST 클라이언트 (HttpClient/RestTemplate):
  Connection Pool로 10~20개 연결 유지
  → 동시 요청 10~20개까지는 괜찮지만 그 이상은 대기
```

### 2. HTTP/2 Stream — 하나의 연결에서 병렬 처리

```
HTTP/2 핵심 개념 3계층:

Frame  → HTTP/2의 최소 단위 (바이너리 포맷)
Stream → Frame들의 논리적 묶음 (하나의 요청-응답 = 하나의 Stream)
Message → 하나 이상의 Frame으로 구성된 완전한 요청 또는 응답

Stream ID 규칙:
  클라이언트가 시작하는 Stream: 홀수 (1, 3, 5, 7, ...)
  서버가 시작하는 Stream: 짝수 (2, 4, 6, ...)
  Stream 0: 연결 수준 설정 (SETTINGS Frame)

HTTP/2 멀티플렉싱:

단일 TCP 연결에서 여러 Stream 병렬 처리:

TCP 연결:
  ├── Stream 1 (RPC A: GetProduct)    → [HEADERS][DATA]  ← [HEADERS][DATA][HEADERS(Trailers)]
  ├── Stream 3 (RPC B: ListProducts)  → [HEADERS][DATA]  ← [HEADERS][DATA][DATA][DATA][HEADERS]
  ├── Stream 5 (RPC C: CreateOrder)   → [HEADERS][DATA]  ← [HEADERS][DATA][HEADERS]
  └── Stream 7 (RPC D: GetInventory)  → [HEADERS][DATA]  ← [HEADERS][DATA][HEADERS]

시간 흐름:
t=0ms:  Stream 1 HEADERS 전송 (GetProduct 시작)
t=0ms:  Stream 3 HEADERS 전송 (ListProducts 시작) ← 동시!
t=0ms:  Stream 5 HEADERS 전송 (CreateOrder 시작)  ← 동시!
t=5ms:  Stream 5 DATA 수신 (CreateOrder 응답, 빠름)
t=10ms: Stream 1 DATA 수신 (GetProduct 응답)
t=100ms: Stream 3 DATA 수신 × 여러 번 (ListProducts 스트리밍)

결론:
  C가 A보다 먼저 끝나도 C 응답을 즉시 받을 수 있음
  Stream이 독립적으로 처리되기 때문
  → Head-of-Line Blocking 없음
```

### 3. HTTP/2 Frame 구조

```
HTTP/2 Frame 바이너리 형식:

┌─────────────────────────────────────────────────────────┐
│ Length (24비트)  │ Type (8비트) │ Flags (8비트)            │
├─────────────────────────────────────────────────────────┤
│ Reserved (1비트) │ Stream ID (31비트)                     │
├─────────────────────────────────────────────────────────┤
│ Payload (가변 길이, Length 바이트)                         │
└─────────────────────────────────────────────────────────┘

Frame 타입:
  0x0  DATA       → 실제 데이터 (Protobuf 메시지)
  0x1  HEADERS    → HTTP 헤더 (요청/응답 메타데이터)
  0x3  RST_STREAM → 스트림 강제 종료
  0x4  SETTINGS   → 연결 파라미터 설정 (최대 스트림 수, 윈도우 크기)
  0x6  PING       → 연결 alive 확인
  0x7  GOAWAY     → 연결 종료 예고
  0x8  WINDOW_UPDATE → Flow Control 윈도우 업데이트
  0x9  CONTINUATION → HEADERS가 너무 클 때 이어지는 프레임

gRPC 요청의 실제 Frame 순서:

클라이언트 → 서버:
  Frame 1: HEADERS (Stream ID = 1)
    :method = POST
    :scheme = https
    :path = /product.v1.ProductService/GetProduct
    :authority = product-service:9090
    content-type = application/grpc
    grpc-encoding = identity (또는 gzip)
    grpc-accept-encoding = identity,deflate,gzip
    grpc-timeout = 3000000u  (3초 = 3,000,000 마이크로초)
    user-agent = grpc-java/1.58.0
    te = trailers  (Trailer 헤더 지원 선언)

  Frame 2: DATA (Stream ID = 1)
    Payload:
      [0x00]              ← 압축 플래그 (0 = 미압축, 1 = 압축)
      [0x00 0x00 0x00 0x08] ← 메시지 길이 (4바이트 big-endian) = 8바이트
      [0x0a 0x06 0x70 0x72 0x6f 0x64 2d 31] ← Protobuf: product_id = "prod-1"
    END_STREAM 미설정 (아직 스트림 열려있음)
    
    ※ Unary RPC는 DATA Frame에 END_STREAM 플래그 설정 → "이게 마지막 데이터"

서버 → 클라이언트:
  Frame 3: HEADERS (Stream ID = 1, 응답 헤더)
    :status = 200
    content-type = application/grpc
    grpc-encoding = identity

  Frame 4: DATA (Stream ID = 1, 응답 데이터)
    Payload:
      [0x00]              ← 압축 플래그
      [0x00 0x00 0x00 0x28] ← 메시지 길이 40바이트
      [Protobuf 응답 데이터]
    END_STREAM 미설정

  Frame 5: HEADERS (Stream ID = 1, Trailers — 스트림 종료)
    grpc-status = 0    (0 = OK)
    grpc-message =     (에러 메시지, 성공이면 빈 문자열)
    END_STREAM 설정    ← "스트림 완전히 닫힘"
```

### 4. HPACK 헤더 압축

```
HTTP/1.1의 헤더 반복 문제:

요청 1:
  content-type: application/grpc
  grpc-encoding: identity
  user-agent: grpc-java/1.58.0
  :method: POST
  :path: /product.v1.ProductService/GetProduct
  authorization: Bearer eyJhbGciOiJIUzI1NiJ9...  (수백 바이트)
  → 총 헤더: ~600 바이트

요청 2 (동일 서비스, 다른 메서드):
  content-type: application/grpc        ← 동일
  grpc-encoding: identity               ← 동일
  user-agent: grpc-java/1.58.0          ← 동일
  :method: POST                         ← 동일
  :path: /product.v1.ProductService/ListProducts  ← 다름
  authorization: Bearer eyJhbGci...     ← 동일
  → 동일 내용이 또 수백 바이트 전송

HTTP/2 HPACK 압축:

정적 테이블 (61개 공통 헤더 미리 정의):
  인덱스 1:  :authority
  인덱스 2:  :method = GET
  인덱스 3:  :method = POST
  인덱스 4:  :path = /
  인덱스 23: content-type = application/json
  ...

동적 테이블 (연결 내 본 헤더를 추가):
  요청 1 이후 동적 테이블:
    [62] content-type = application/grpc
    [63] grpc-encoding = identity
    [64] user-agent = grpc-java/1.58.0
    [65] authorization = Bearer eyJ...

요청 2 전송:
  :method = POST      → 인덱스 3 (1바이트)
  :path = /product.v1.ProductService/ListProducts → 새 값 (리터럴)
  content-type       → 인덱스 62 (1바이트) ← 동일 헤더 재전송 불필요!
  grpc-encoding      → 인덱스 63 (1바이트)
  user-agent         → 인덱스 64 (1바이트)
  authorization      → 인덱스 65 (1바이트) ← JWT 재전송 없음!

결과: 두 번째 요청부터 헤더 크기 대폭 감소
  첫 요청: ~600 바이트
  두 번째 요청: ~30 바이트 (인덱스 참조)
```

### 5. Flow Control — 수신 측 보호

```
HTTP/2 Flow Control 동작:

배경: 서버가 처리하는 것보다 빠르게 클라이언트가 데이터를 보내면?
  → 수신 측 버퍼 오버플로우
  → Out-of-Memory 또는 데이터 손실

Window Size (수신 가능한 바이트 수):
  연결 초기: SETTINGS Frame에서 initial_window_size 교환
  기본값: 65,535 바이트 (64KB - 1)

Flow Control 동작:

  클라이언트 Window: 65535 바이트 사용 가능

  클라이언트 → 서버: DATA Frame (10,000 바이트)
    → 서버 수신 Window: 65535 - 10000 = 55535 바이트 남음

  클라이언트 → 서버: DATA Frame (20,000 바이트)
    → 서버 수신 Window: 55535 - 20000 = 35535 바이트 남음

  서버가 데이터 처리 완료 → WINDOW_UPDATE Frame 전송
    → "처리된 만큼 Window 복원해줘"
    Increment: 30000 → 서버 Window: 35535 + 30000 = 65535 복원

  만약 Window가 0이 되면:
    클라이언트: DATA Frame 전송 중단 (차단)
    서버의 WINDOW_UPDATE를 기다림
    → 이것이 "Backpressure" — 수신 측이 속도를 제어

gRPC에서의 실제 설정:
  .flowControlWindow(1024 * 1024)  // Channel 수준 1MB Window
  서버: maxInboundFlowControlWindowBytes 설정
  
  스트리밍 응답이 많은 경우 Window 크게 설정 → 처리량 증가
  처리 속도보다 전송 속도가 빠른 경우 Window 작게 → 메모리 보호
```

---

## 💻 실전 실험

### 실험 1: Wireshark로 HTTP/2 Frame 관찰

```bash
# Docker Compose 환경 시작 (gRPC 서버 + 클라이언트)
docker compose up -d

# Wireshark 시작 후 캡처 필터 설정
# 캡처 필터: tcp.port == 9090
# 디스플레이 필터: http2

# TLS 없는 환경(usePlaintext)에서 gRPC 호출
grpcurl -plaintext localhost:9090 \
  product.v1.ProductService/GetProduct \
  -d '{"product_id": "prod-1"}'

# Wireshark에서 확인:
# HTTP2 HEADERS Frame (Stream 1) → :path = /product.v1.ProductService/GetProduct
# HTTP2 DATA Frame (Stream 1) → Protobuf 바이트
# HTTP2 HEADERS Frame (Stream 1, Trailers) → grpc-status: 0

# 멀티플렉싱 관찰: 동시에 여러 grpcurl 실행
for i in 1 2 3 4 5; do
  grpcurl -plaintext localhost:9090 \
    product.v1.ProductService/GetProduct \
    -d "{\"product_id\": \"prod-$i\"}" &
done
# Wireshark: Stream 1, 3, 5, 7, 9 가 동시에 보임
```

### 실험 2: HTTP/2 Frame을 직접 디코딩

```bash
# h2c (HTTP/2 Cleartext) 연결 내용 덤프
# grpc-server 포트 9090에서 패킷 캡처
tcpdump -i any -w /tmp/grpc.pcap port 9090

# 다른 터미널에서 gRPC 호출
grpcurl -plaintext localhost:9090 \
  product.v1.ProductService/GetProduct \
  -d '{"product_id": "prod-1"}'

# pcap 파일 분석 (tshark)
tshark -r /tmp/grpc.pcap -T fields \
  -e frame.number \
  -e http2.type \
  -e http2.streamid \
  -e http2.flags \
  -Y http2

# 출력 예시:
# 5   1(HEADERS)  1   0x04  ← 클라이언트 요청 헤더, END_HEADERS
# 6   0(DATA)     1   0x01  ← 클라이언트 요청 데이터, END_STREAM
# 7   1(HEADERS)  1   0x04  ← 서버 응답 헤더
# 8   0(DATA)     1   0x00  ← 서버 응답 데이터
# 9   1(HEADERS)  1   0x05  ← Trailers (END_HEADERS + END_STREAM)
```

### 실험 3: maxConcurrentStreams 제한 관찰

```bash
# 서버에서 동시 스트림 최대 5개 설정
# application.yml:
# grpc:
#   server:
#     max-concurrent-calls-per-connection: 5

# 6개 동시 요청 전송
for i in $(seq 1 6); do
  curl -s http://localhost:8081/concurrent-grpc-test &
done
wait

# 6번째 요청은 앞의 5개 중 하나가 완료될 때까지 대기
# 서버 로그에서 "Stream refused" 또는 대기 시간 확인
```

---

## 📊 성능/비용 비교

```
HTTP/1.1 vs HTTP/2 동시 요청 처리 비교 (참고치):

10개 동시 REST 요청 (HTTP/1.1, Connection Pool 10):
  10개 TCP 연결 (이미 풀에 있다고 가정)
  각 요청: 순서대로 응답 대기
  총 소요: max(각 요청 처리 시간) + 연결 오버헤드

10개 동시 gRPC 요청 (HTTP/2, 단일 Channel):
  1개 TCP 연결, 10개 스트림 (Stream 1,3,5,7,9,11,13,15,17,19)
  각 스트림: 독립적으로 처리, HOL Blocking 없음
  총 소요: max(각 RPC 처리 시간)

헤더 크기 비교 (HPACK 압축):
  HTTP/1.1 첫 요청:         ~600 바이트
  HTTP/1.1 반복 요청:       ~600 바이트 (항상 전체 전송)
  HTTP/2 첫 요청:           ~600 바이트 (동적 테이블 구성)
  HTTP/2 반복 요청:         ~30 바이트 (인덱스 참조)

Frame 오버헤드:
  HTTP/2 Frame 헤더:     9바이트 (고정)
  HTTP/2 Data Frame:     9 + payload 바이트
  → 매우 작은 메시지(1바이트)면 Frame 헤더가 9배 오버헤드
  → 일반적인 gRPC 페이로드(수십 바이트 이상)에서는 무시할 수준
```

---

## ⚖️ 트레이드오프

```
HTTP/2 멀티플렉싱의 장단점:

장점:
  ① HOL Blocking 제거 — 느린 요청이 빠른 요청을 차단하지 않음
  ② 연결 재사용 — TCP/TLS 핸드쉐이크 비용 1회로 감소
  ③ 헤더 압축 — HPACK으로 반복 헤더 크기 감소
  ④ 스트리밍 기본 지원 — 스트림 단위로 양방향 통신 가능

단점:
  ① TCP 레벨 HOL Blocking 잔존
     HTTP/2는 TCP 위에서 동작
     TCP 패킷 손실 시 재전송 완료까지 모든 스트림 대기
     → HTTP/3 (QUIC)에서 해결 (Stream별 독립 패킷 처리)
     → gRPC over HTTP/3은 현재 실험적 지원
  
  ② maxConcurrentStreams 제한
     서버/클라이언트 모두 동시 스트림 최대값 설정 가능
     기본: 서버 100개, 클라이언트 무제한
     → 초과 시 새 스트림 대기 또는 거절
  
  ③ L4 로드밸런서 무력화
     단일 TCP 연결 → L4 로드밸런서는 이 연결을 한 서버로만 라우팅
     → 클라이언트 사이드 로드밸런싱 또는 L7 Proxy(Envoy) 필요
     (자세한 내용은 service-design/05-load-balancing.md 참고)

HTTP/2와 HTTP/3 비교:
  HTTP/2: TCP 기반, 스트림 멀티플렉싱, TCP HOL Blocking 잔존
  HTTP/3: QUIC(UDP) 기반, 스트림별 독립 패킷, HOL Blocking 완전 제거
  gRPC over HTTP/3: 실험적 단계, 프로덕션 사용 미권장 (2024 기준)
```

---

## 📌 핵심 정리

```
HTTP/2 멀티플렉싱 핵심:

3계층 구조:
  Frame  → 바이너리 최소 단위 (9바이트 헤더 + 페이로드)
  Stream → 논리적 채널 (홀수 ID: 클라이언트, 짝수 ID: 서버)
  Message → 하나 이상의 Frame으로 완전한 요청/응답

HOL Blocking 해결:
  HTTP/1.1: 연결 1개당 요청 1개 직렬 처리
  HTTP/2:   연결 1개에 스트림 N개 병렬 처리
  → 느린 요청(Stream A)이 빠른 요청(Stream B)을 차단 안 함

gRPC 요청 Frame 순서:
  → HEADERS Frame (:path, content-type, grpc-timeout)
  → DATA Frame (5바이트 gRPC 헤더 + Protobuf 바이트)
  ← HEADERS Frame (:status = 200)
  ← DATA Frame (응답 Protobuf)
  ← HEADERS Frame (Trailers: grpc-status = 0)

HPACK 압축:
  반복 헤더 → 인덱스 참조
  첫 요청: 전체 헤더 전송 + 동적 테이블 구성
  이후 요청: 인덱스로 대체 → 헤더 크기 대폭 감소

Flow Control:
  Window Size = 수신 가능 바이트
  WINDOW_UPDATE로 용량 복원
  Window 0 → 전송 차단 (Backpressure)
```

---

## 🤔 생각해볼 문제

**Q1.** gRPC가 HTTP/2 기반이라 TCP Head-of-Line Blocking은 여전히 존재한다. 이것이 실무에서 실제로 문제가 되는 경우는 언제이고, 어떻게 감지하는가?

<details>
<summary>해설 보기</summary>

TCP HOL Blocking이 실제로 문제가 되는 경우:

- **패킷 손실률이 높은 네트워크**: 와이파이, 혼잡한 WAN 구간에서 패킷 손실 시 재전송이 완료될 때까지 동일 연결의 모든 스트림이 대기
- **지리적으로 먼 서버**: RTT가 높고 패킷 손실률이 높은 환경 (한국-미국 간 gRPC 등)

감지 방법:
```bash
# TCP 재전송 통계 확인
netstat -s | grep -i retransmit

# 또는 ss로 특정 연결 재전송 확인
ss -tn | grep :9090
# Retrans 컬럼이 증가하면 TCP HOL Blocking 영향 중
```

실무에서의 판단:
- 동일 AZ(가용 영역) 내 서비스 간: 패킷 손실률 < 0.01%, TCP HOL Blocking 거의 무시 가능
- 크로스 리전, 퍼블릭 인터넷: 패킷 손실률이 높으면 HTTP/3(QUIC) 검토

</details>

---

**Q2.** gRPC 서버에서 `maxConcurrentCallsPerConnection`을 설정하면 초과된 요청은 어떻게 처리되는가? 클라이언트에서 이를 어떻게 인지하고 처리해야 하는가?

<details>
<summary>해설 보기</summary>

`maxConcurrentCallsPerConnection` 초과 시:

- 서버: `RST_STREAM` Frame 전송 (Stream ID, Error Code: `REFUSED_STREAM`)
- 또는: `GOAWAY` Frame으로 연결 종료 예고 후 새 연결 유도

클라이언트에서 수신하는 오류:
- `StatusCode.UNAVAILABLE` + `grpc-status-details`에 재시도 가능 여부 포함
- 재시도 가능(`REFUSED_STREAM`)이면 새 연결 또는 다른 서버로 재시도

클라이언트 처리 방법:
```java
// gRPC 재시도 정책 설정 (service config)
Map<String, Object> retryPolicy = new HashMap<>();
retryPolicy.put("maxAttempts", 3.0);
retryPolicy.put("initialBackoff", "0.1s");
retryPolicy.put("maxBackoff", "1s");
retryPolicy.put("backoffMultiplier", 2.0);
retryPolicy.put("retryableStatusCodes", List.of("UNAVAILABLE"));
```

`REFUSED_STREAM`은 서버가 요청을 아직 처리하지 않은 상태이므로 멱등 여부와 관계없이 안전하게 재시도 가능합니다.

</details>

---

**Q3.** HTTP/2 SETTINGS Frame에서 교환하는 `INITIAL_WINDOW_SIZE`와 `MAX_CONCURRENT_STREAMS`는 각각 어느 시점에 어떻게 적용되는가?

<details>
<summary>해설 보기</summary>

연결 수립 직후 양방향으로 SETTINGS Frame을 교환합니다:

```
클라이언트 → 서버: SETTINGS
  SETTINGS_MAX_CONCURRENT_STREAMS = 1000  (클라이언트가 허용하는 서버 주도 스트림 수)
  SETTINGS_INITIAL_WINDOW_SIZE = 65535

서버 → 클라이언트: SETTINGS
  SETTINGS_MAX_CONCURRENT_STREAMS = 100   (서버가 허용하는 동시 스트림 수)
  SETTINGS_INITIAL_WINDOW_SIZE = 65535

양측: SETTINGS_ACK 전송 (수신 확인)
```

적용 시점:
- `MAX_CONCURRENT_STREAMS`: 즉시 적용. 상대방이 이 값을 초과하는 스트림을 시도하면 `RST_STREAM` 또는 연결 오류
- `INITIAL_WINDOW_SIZE`: 이후 생성되는 새 스트림에 적용. 기존 스트림의 Window는 별도 `WINDOW_UPDATE`로 조정

실무 설정:
```yaml
# grpc-spring-boot-starter
grpc:
  server:
    max-concurrent-calls-per-connection: 200
    flow-control-window: 1048576  # 1MB
```

</details>

---

<div align="center">

**[⬅️ 이전: gRPC 핵심 구성 요소](./02-grpc-core-components.md)** | **[홈으로 🏠](../README.md)** | **[다음: gRPC 통신 4가지 패턴 ➡️](./04-grpc-patterns.md)**

</div>
