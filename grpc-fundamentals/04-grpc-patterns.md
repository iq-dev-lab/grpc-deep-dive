# gRPC 통신 4가지 패턴 — 언제 무엇을 쓰는가

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- Unary, Server Streaming, Client Streaming, Bidirectional Streaming 각각의 내부 동작은 어떻게 다른가?
- "실시간 데이터가 필요하다"는 요구사항에서 Polling과 Server Streaming 중 어떤 기준으로 선택하는가?
- 파일 업로드처럼 큰 데이터를 보낼 때 Unary와 Client Streaming 중 무엇이 맞는가?
- 잘못된 패턴 선택이 실무에서 어떤 구체적인 문제를 만드는가?
- `grpcurl`로 각 패턴을 직접 호출하고 차이를 확인하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC를 도입하는 팀의 70%는 Unary만 쓴다. 나머지 30%는 스트리밍 패턴을 잘못 선택해서 Polling을 없애려다 더 복잡한 연결 관리 코드를 만들거나, Client Streaming을 Unary 반복 호출로 구현해 네트워크 오버헤드를 그대로 유지한다. 4가지 패턴이 HTTP/2 레벨에서 어떻게 다른지 알면 "어떤 상황에서 어떤 패턴이 적합한가"의 판단이 명확해진다.

---

## 😱 흔한 실수 (Before — REST 사고방식으로 gRPC를 보는 접근)

```
실수 1: 실시간 데이터를 Polling으로 구현 후 gRPC로도 동일하게 Polling

// ❌ REST Polling 사고방식을 gRPC에 그대로 적용
@Scheduled(fixedDelay = 1000)
public void pollOrderStatus() {
    GetOrderStatusResponse response = stub.getOrderStatus(
        GetOrderStatusRequest.newBuilder()
            .setOrderId("order-123")
            .build()
    );
    if (response.getStatus() == OrderStatus.SHIPPED) {
        notifyUser(response);
    }
}
// 1초마다 RPC 호출 → Server Streaming으로 해결 가능한 문제

실수 2: 대용량 파일 업로드를 Unary 1회 호출로

// ❌ 100MB 파일을 Unary로 한 번에 전송
byte[] fileBytes = Files.readAllBytes(Paths.get("large-file.pdf")); // 100MB 메모리!
UploadFileRequest request = UploadFileRequest.newBuilder()
    .setFileContent(ByteString.copyFrom(fileBytes))  // 100MB 직렬화
    .build();
stub.uploadFile(request);  // 100MB 단일 메시지 → 서버 maxInboundMessageSize 초과!

// gRPC 기본 최대 메시지 크기: 4MB
// 100MB 파일 → protobuf.InvalidProtocolBufferException: Protocol message too large

실수 3: Bidirectional Streaming을 Server Streaming으로 잘못 구현

// ❌ 채팅 서비스를 Server Streaming으로 구현
// 클라이언트마다 서버로 메시지 보내기: REST POST 호출
// 메시지 받기: gRPC Server Streaming 구독
// → 두 개의 서로 다른 연결이 필요 (REST + gRPC)
// → Bidirectional Streaming 하나로 해결 가능했던 것을 복잡하게 구현
```

---

## ✨ 올바른 접근 (After — 패턴별 적합한 시나리오)

```
패턴 선택 기준:

요청 1개 → 응답 1개:          Unary
요청 1개 → 응답 N개 (실시간):  Server Streaming
요청 N개 → 응답 1개 (집계):    Client Streaming
요청 N개 → 응답 N개 (양방향):  Bidirectional Streaming

시나리오별 올바른 패턴:
  상품 조회, 주문 생성         → Unary
  실시간 주식 시세 구독         → Server Streaming
  실시간 로그 스트림 조회       → Server Streaming
  파일 청크 업로드             → Client Streaming
  배치 데이터 전송 후 결과 수신 → Client Streaming
  채팅 서비스                 → Bidirectional Streaming
  게임 상태 동기화             → Bidirectional Streaming
  실시간 협업 편집기           → Bidirectional Streaming
```

---

## 🔬 내부 동작 원리

### 1. Unary RPC — 가장 단순한 패턴

```
Unary: 요청 1개 → 응답 1개 (일반적인 함수 호출과 동일)

.proto 정의:
  rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse);

HTTP/2 Frame 흐름:
  클라이언트                          서버
      │── HEADERS Frame ───────────────▶│
      │   :path = /order.v1/CreateOrder │
      │── DATA Frame (END_STREAM) ──────▶│  ← END_STREAM: 클라이언트 전송 완료
      │   [Protobuf 요청 데이터]          │
      │                                 │── 비즈니스 로직 처리
      │◀─ HEADERS Frame ────────────────│
      │   :status = 200                │
      │◀─ DATA Frame ───────────────────│
      │   [Protobuf 응답 데이터]          │
      │◀─ HEADERS Frame (Trailers) ─────│  ← grpc-status = 0
      │   (END_STREAM)                  │     스트림 완전히 닫힘

서버 구현:
  @Override
  public void createOrder(
      CreateOrderRequest request,
      StreamObserver<CreateOrderResponse> responseObserver) {
    
    Order order = orderService.create(request);
    
    responseObserver.onNext(CreateOrderResponse.newBuilder()
        .setOrderId(order.getId())
        .build());
    responseObserver.onCompleted();  // 스트림 종료
  }

특징:
  - 요청과 응답이 완전히 메모리에 올라온 후 처리
  - 간단하고 직관적 — HTTP REST와 동일한 사고방식
  - 응답 시간이 빠를 때(수 ms) 최적
  - 응답이 크거나 처리가 오래 걸리면 클라이언트가 마냥 대기
```

### 2. Server Streaming — 서버가 여러 응답을 푸시

```
Server Streaming: 요청 1개 → 응답 N개

.proto 정의:
  rpc WatchOrderStatus(WatchOrderRequest) returns (stream OrderStatusEvent);
  //                                                ^^^^^^
  //                                        stream 키워드 = 여러 응답

HTTP/2 Frame 흐름:
  클라이언트                          서버
      │── HEADERS Frame ───────────────▶│
      │── DATA Frame (END_STREAM) ──────▶│  ← 클라이언트 전송 완료
      │                                 │
      │◀─ HEADERS Frame ────────────────│
      │◀─ DATA Frame (OrderEvent #1) ───│  ← 첫 번째 이벤트
      │◀─ DATA Frame (OrderEvent #2) ───│  ← 두 번째 이벤트 (시간이 지나면)
      │◀─ DATA Frame (OrderEvent #3) ───│  ← 세 번째 이벤트
      │   ...                           │
      │◀─ HEADERS Frame (Trailers) ─────│  ← 스트림 종료 (주문 최종 완료 등)

서버 구현:
  @Override
  public void watchOrderStatus(
      WatchOrderRequest request,
      StreamObserver<OrderStatusEvent> responseObserver) {
    
    String orderId = request.getOrderId();
    
    // 이벤트 발생할 때마다 onNext() 호출
    orderEventService.subscribe(orderId, event -> {
        responseObserver.onNext(OrderStatusEvent.newBuilder()
            .setStatus(event.getStatus())
            .setTimestamp(event.getTimestamp())
            .build());
        
        // 최종 상태 도달 시 스트림 종료
        if (event.isFinal()) {
            responseObserver.onCompleted();
        }
    });
  }

클라이언트 수신:
  stub.watchOrderStatus(
      WatchOrderRequest.newBuilder().setOrderId("order-123").build(),
      new StreamObserver<OrderStatusEvent>() {
          @Override
          public void onNext(OrderStatusEvent event) {
              System.out.println("상태 변경: " + event.getStatus());
              // Polling 없이 서버가 변경 시 즉시 알려줌
          }
          @Override
          public void onError(Throwable t) { /* 에러 처리 */ }
          @Override
          public void onCompleted() { /* 스트림 완료 */ }
      }
  );

Polling vs Server Streaming 비교:
  Polling (1초마다):
    클라이언트 → 서버: 매 1초 RPC 요청
    → 변경 없어도 요청 발생 → 불필요한 네트워크/서버 부하
    → 최대 1초 지연 발생 가능

  Server Streaming:
    연결 1회 수립 → 이벤트 발생 시 즉시 전달
    → 변경이 없으면 아무 것도 전송하지 않음
    → 이벤트 발생 즉시 클라이언트에 도달 (수 ms 이내)
```

### 3. Client Streaming — 클라이언트가 여러 요청을 전송

```
Client Streaming: 요청 N개 → 응답 1개

.proto 정의:
  rpc BulkUploadProducts(stream ProductUploadRequest) returns (BulkUploadResponse);
  //                      ^^^^^^
  //              stream 키워드 = 여러 요청

HTTP/2 Frame 흐름:
  클라이언트                          서버
      │── HEADERS Frame ───────────────▶│
      │── DATA Frame (Product #1) ──────▶│
      │── DATA Frame (Product #2) ──────▶│
      │── DATA Frame (Product #3) ──────▶│
      │   ...                           │
      │── DATA Frame (END_STREAM) ──────▶│  ← 클라이언트 전송 완료 신호
      │                                 │── 전체 데이터 집계 처리
      │◀─ HEADERS Frame ────────────────│
      │◀─ DATA Frame (집계 결과) ─────────│  ← 단일 응답
      │◀─ HEADERS Frame (Trailers) ─────│

클라이언트 구현:
  ProductServiceGrpc.ProductServiceStub asyncStub =
      ProductServiceGrpc.newStub(channel);
  
  StreamObserver<BulkUploadResponse> responseObserver =
      new StreamObserver<>() {
          @Override
          public void onNext(BulkUploadResponse response) {
              System.out.println("업로드 완료: " + response.getSuccessCount() + "개");
          }
          // ...
      };
  
  StreamObserver<ProductUploadRequest> requestObserver =
      asyncStub.bulkUploadProducts(responseObserver);
  
  // 청크 단위로 전송 (메모리 효율적)
  for (ProductChunk chunk : productChunks) {
      requestObserver.onNext(ProductUploadRequest.newBuilder()
          .addAllProducts(chunk.getProducts())
          .build());
      // 각 청크는 독립 DATA Frame → 전체를 메모리에 올릴 필요 없음
  }
  requestObserver.onCompleted();  // 전송 완료 → 서버에서 응답 보냄

서버 구현:
  @Override
  public StreamObserver<ProductUploadRequest> bulkUploadProducts(
      StreamObserver<BulkUploadResponse> responseObserver) {
    
    List<Product> accumulated = new ArrayList<>();
    
    return new StreamObserver<ProductUploadRequest>() {
        @Override
        public void onNext(ProductUploadRequest request) {
            // 각 청크를 받을 때마다 처리 (스트리밍 처리)
            accumulated.addAll(request.getProductsList());
        }
        @Override
        public void onCompleted() {
            // 모든 청크 수신 완료 → 집계 처리 후 단일 응답
            int savedCount = productService.saveAll(accumulated);
            responseObserver.onNext(BulkUploadResponse.newBuilder()
                .setSuccessCount(savedCount)
                .build());
            responseObserver.onCompleted();
        }
        @Override
        public void onError(Throwable t) {
            responseObserver.onError(t);
        }
    };
  }

Unary vs Client Streaming (파일 업로드 100MB):
  Unary:
    클라이언트: 100MB 전체를 메모리에 로드 → ByteString 변환
    단일 메시지: gRPC 기본 4MB 제한 초과 → 에러
    서버: 100MB를 한 번에 수신 → 메모리 스파이크

  Client Streaming:
    클라이언트: 1MB 청크 × 100회 전송
    각 청크: 별도 DATA Frame → 4MB 제한 무관
    서버: 청크마다 처리 → 메모리 일정하게 유지
```

### 4. Bidirectional Streaming — 완전한 양방향 통신

```
Bidirectional Streaming: 요청 N개 ↔ 응답 N개

.proto 정의:
  rpc Chat(stream ChatMessage) returns (stream ChatMessage);
  //        ^^^^^^                       ^^^^^^
  //  양쪽 모두 stream → 동시에 메시지 교환 가능

HTTP/2 Frame 흐름:
  클라이언트                          서버
      │── HEADERS Frame ───────────────▶│
      │── DATA Frame ("안녕하세요") ──────▶│
      │◀─ DATA Frame ("안녕하세요!") ──────│  ← 동시에 교환 가능
      │── DATA Frame ("잘 지내셨나요?") ──▶│
      │◀─ DATA Frame ("네, 잘 지냈습니다") ─│
      │   ...                           │
      │── DATA Frame (END_STREAM) ──────▶│  ← 클라이언트 전송 완료
      │◀─ HEADERS Frame (Trailers) ─────│  ← 서버 종료

동작 특성:
  - 클라이언트와 서버가 독립적으로 메시지 전송 가능
  - 한쪽이 END_STREAM을 보내도 다른 쪽은 계속 보낼 수 있음
  - Half-close: 한쪽이 완료를 선언해도 반대쪽 스트림은 유지

서버 구현:
  @Override
  public StreamObserver<ChatMessage> chat(
      StreamObserver<ChatMessage> responseObserver) {
    
    return new StreamObserver<ChatMessage>() {
        @Override
        public void onNext(ChatMessage message) {
            // 클라이언트 메시지 수신 → 방 전체에 브로드캐스트
            chatRoomService.broadcast(message.getRoomId(), message);
            // 필요하면 응답도 즉시 전송
            responseObserver.onNext(ChatMessage.newBuilder()
                .setContent("수신됨: " + message.getContent())
                .build());
        }
        @Override
        public void onCompleted() {
            responseObserver.onCompleted();
        }
        @Override
        public void onError(Throwable t) {
            responseObserver.onError(t);
        }
    };
  }

WebSocket vs Bidirectional Streaming 비교:
  WebSocket:
    브라우저 기본 지원, 텍스트/바이너리 모두 지원
    프로토콜 계약 없음 (직접 JSON 구조 정의)
    메시지 타입 강제 없음 → 런타임 에러 가능

  Bidirectional gRPC:
    브라우저 직접 호출 불가 (gRPC-Web + Proxy 필요)
    Protobuf 계약 → 타입 안전 메시지 교환
    서비스 간 통신에서 강점
```

---

## 💻 실전 실험

### 실험: grpcurl로 각 패턴 직접 호출

```bash
# Docker 환경 시작
docker compose up -d grpc-server

# ① Unary 호출
grpcurl -plaintext localhost:9090 \
  order.v1.OrderService/CreateOrder \
  -d '{"user_id": "user-1", "items": [{"product_id": "prod-1", "quantity": 2}]}'

# 출력: 단일 응답 후 종료
# {
#   "orderId": "order-abc123",
#   "status": "ORDER_STATUS_PENDING"
# }

# ② Server Streaming 호출
grpcurl -plaintext localhost:9090 \
  order.v1.OrderService/WatchOrderStatus \
  -d '{"order_id": "order-abc123"}'

# 출력: 상태가 바뀔 때마다 출력 (Ctrl+C로 종료)
# {
#   "status": "ORDER_STATUS_CONFIRMED",
#   "timestamp": "2024-01-15T10:30:00Z"
# }
# {
#   "status": "ORDER_STATUS_SHIPPED",
#   "timestamp": "2024-01-15T11:00:00Z"
# }

# ③ Client Streaming — grpcurl은 stdin으로 여러 메시지 입력
echo '{"products": [{"id": "p1", "name": "노트북"}]}
{"products": [{"id": "p2", "name": "마우스"}]}' | \
grpcurl -plaintext localhost:9090 \
  product.v1.ProductService/BulkUploadProducts \
  -d @  # stdin에서 읽기

# ④ Bidirectional Streaming
# grpcurl은 stdin 입력이 스트림으로 전송됨
grpcurl -plaintext localhost:9090 \
  chat.v1.ChatService/Chat \
  -d '{"room_id": "room-1", "content": "안녕하세요"}
{"room_id": "room-1", "content": "잘 지내셨나요?"}'
```

---

## 📊 성능/비용 비교

```
패턴별 연결 비용과 특성:

                    Unary           Server      Client      Bidirectional
                                  Streaming   Streaming    Streaming
─────────────────────────────────────────────────────────────────────
TCP 연결 수립         1회             1회         1회           1회
스트림 수             1개             1개         1개           1개
클라이언트 DATA Frame  1회             1회         N회           N회
서버 DATA Frame      1회             N회         1회           N회
메모리 사용 (클라이언트) 요청 전체       요청 전체   청크 단위      청크 단위
메모리 사용 (서버)    응답 전체        응답마다     집계 방식       메시지마다
적합한 요청 크기      ~수 MB 이하      작음         청크 단위       작음
적합한 응답 지연      낮음             높음 무관    낮음           낮음
─────────────────────────────────────────────────────────────────────

Polling 1초 vs Server Streaming (상태 변경 빈도: 분당 2회):
  Polling:
    RPC 호출 수: 60회/분
    변경 감지 지연: 평균 500ms
    네트워크 트래픽: 60 × (요청+응답 크기)

  Server Streaming:
    RPC 호출 수: 1회 (연결) + 2회 이벤트 = 3회/분
    변경 감지 지연: ~수 ms (이벤트 즉시 전달)
    네트워크 트래픽: 1회 연결 + 2회 이벤트 크기
```

---

## ⚖️ 트레이드오프

```
각 패턴의 트레이드오프:

Unary:
  ✅ 단순, 구현 쉬움, REST와 동일 사고방식
  ❌ 응답이 느린 경우 클라이언트 대기, 대용량 전송 불가

Server Streaming:
  ✅ 실시간 데이터 푸시, Polling 제거, 이벤트 즉시 전달
  ❌ 연결 유지 비용, 스트림 생명주기 관리 필요
  ❌ 클라이언트가 처리 못할 속도로 데이터 오면 Backpressure 처리 필요

Client Streaming:
  ✅ 대용량 데이터 청크 전송, 메모리 효율적
  ❌ 서버가 모든 청크를 받아야 응답 — 중간 실패 복구 복잡
  ❌ 서버 측 집계 로직 필요

Bidirectional Streaming:
  ✅ 완전한 양방향 실시간 통신
  ❌ 가장 복잡한 구현 (스트림 생명주기, 에러, 재연결)
  ❌ 브라우저 미지원 — 서비스 간 통신에 적합
  ❌ 연결 유지 중 서버 재시작 → 클라이언트 재연결 로직 필요

스트리밍 공통 주의사항:
  - 연결이 끊겼을 때 재연결 전략 필요
  - 서버 재배포 시 진행 중인 스트림이 끊김 → 클라이언트 처리 필요
  - 장시간 스트림 → keepAlive 설정 필수
    (방화벽/프록시가 유휴 연결을 끊을 수 있음)
```

---

## 📌 핵심 정리

```
gRPC 4가지 패턴 선택 기준:

Unary (rpc Method(Req) returns (Res)):
  사용: 일반적인 요청-응답, 처리 시간이 짧은 경우
  예:  상품 조회, 주문 생성, 인증

Server Streaming (rpc Method(Req) returns (stream Res)):
  사용: 서버가 지속적으로 데이터를 보내야 하는 경우
  예:  실시간 시세, 로그 스트림, 상태 변경 구독
  핵심: Polling → Server Streaming으로 변경 시 즉각적 이점

Client Streaming (rpc Method(stream Req) returns (Res)):
  사용: 대용량 데이터를 청크로 전송 후 집계 결과
  예:  파일 업로드, 배치 데이터 입력
  핵심: 대용량 = Unary 불가 → Client Streaming

Bidirectional Streaming (rpc Method(stream Req) returns (stream Res)):
  사용: 양방향 실시간 통신, 완전한 양방향 메시지 교환
  예:  채팅, 게임 상태, 실시간 협업
  핵심: WebSocket의 서비스 간 통신 버전

잘못된 패턴 선택의 증상:
  "Polling을 아직 쓰고 있다" → Server Streaming 검토
  "파일 업로드가 4MB 제한에 걸린다" → Client Streaming
  "두 개의 연결로 채팅을 구현했다" → Bidirectional Streaming
```

---

## 🤔 생각해볼 문제

**Q1.** Server Streaming에서 서버가 이벤트를 초당 10,000개 생성하는데 클라이언트가 초당 100개밖에 처리 못한다면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

HTTP/2 Flow Control이 자동으로 개입합니다:

1. 클라이언트의 수신 Window가 소진됩니다 (클라이언트가 처리 못하고 쌓임)
2. 클라이언트가 WINDOW_UPDATE를 보내지 않으면 서버의 쓰기가 차단됩니다
3. 서버는 `onNext()` 호출이 블록되거나, 내부 버퍼가 가득 차서 지연이 발생합니다

이것이 Backpressure의 동작 방식입니다. 근본적인 해결:
- 클라이언트 처리 속도를 높이거나
- 서버에서 이벤트 샘플링/집계 후 전송하거나
- Spring WebFlux의 `Flux`로 처리하면 Reactive Streams Backpressure와 연동됩니다

</details>

---

**Q2.** Client Streaming에서 청크 전송 중간에 서버가 재시작되면 어떻게 되는가? 이미 전송한 청크들은 어떻게 되는가?

<details>
<summary>해설 보기</summary>

서버 재시작 시:
- TCP 연결이 끊기고 HTTP/2 스트림이 GOAWAY 또는 RST_STREAM으로 종료됩니다
- 클라이언트의 `onError()`가 호출되고 `StatusCode.UNAVAILABLE`을 받습니다
- 이미 서버로 전송된 청크는 서버가 처리했을 수도, 아닐 수도 있습니다 (원자성 없음)

처리 방법:
- 각 청크에 시퀀스 번호를 포함하여 서버가 중복 처리를 방지합니다
- 서버에 "어디까지 받았나요?" API(Unary)를 만들어 재시작 지점을 파악합니다
- 또는 전체를 멱등 연산으로 설계하여 처음부터 재전송합니다

이 문제가 Client Streaming의 단점 중 하나입니다. 내결함성이 중요하다면 메시지 큐(Kafka 등)가 더 적합할 수 있습니다.

</details>

---

**Q3.** `grpc-timeout` 헤더는 Streaming RPC에도 적용되는가? Server Streaming에서 `withDeadlineAfter(30, SECONDS)`를 설정하면 30초 후에 무슨 일이 발생하는가?

<details>
<summary>해설 보기</summary>

`grpc-timeout`은 스트리밍 RPC에도 적용됩니다. Deadline은 "스트림 전체 지속 시간"의 최대값입니다.

Server Streaming에서 30초 Deadline 설정 시:
- 30초 이내에 `onCompleted()`가 호출되지 않으면 클라이언트가 스트림을 강제 종료합니다
- 클라이언트: `DEADLINE_EXCEEDED` 상태로 `onError()` 호출
- 서버: Context가 취소됨을 감지 (`Context.current().isCancelled()`)

실시간 스트리밍(주식 시세 등)에서는:
- Deadline을 설정하지 않거나 매우 크게 설정합니다
- 대신 클라이언트에서 명시적으로 `cancel()`을 호출하여 스트림을 닫습니다
- 서버에서 `Context.current().addListener()`로 취소를 감지하고 리소스를 정리합니다

</details>

---

<div align="center">

**[⬅️ 이전: HTTP/2 기반 통신](./03-http2-multiplexing.md)** | **[홈으로 🏠](../README.md)** | **[다음: gRPC 생태계 ➡️](./05-grpc-ecosystem.md)**

</div>
