# 메타데이터 — gRPC의 HTTP 헤더

---

## 🎯 핵심 질문

- 메타데이터는 뭐고 어디에 저장되나?
- 일반 메타데이터와 Trailer는 뭐가 다른가?
- 인증 토큰은 어떻게 메타데이터에 담길까?
- Interceptor에서 메타데이터를 추출하려면?
- 요청 ID와 추적은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

메타데이터는 HTTP/2 헤더와 같습니다. 인증 토큰, 요청 ID, 추적 정보, 타임존 등을 전달하며, Interceptor에서 자동으로 처리하면 매 RPC마다 반복 코드를 줄입니다. 분산 추적(tracing)은 메타데이터로 전파되어야 마이크로서비스 간 요청을 추적할 수 있습니다.

---

## 😱 흔한 실수

```
// 실수 1: 메타데이터와 메시지 필드 혼용
message GetUserRequest {
  string user_id = 1;
  string auth_token = 2;         // ❌ 메타데이터로 가야 할 것
  string trace_id = 3;           // ❌ 메타데이터로 가야 할 것
  string request_id = 4;         // ❌ 메타데이터로 가야 할 것
}

// 문제: 매 요청마다 토큰 전송, 메시지 크기 증가

// 실수 2: 메타데이터 처리 반복
public void getUser(GetUserRequest req, ...) {
  // 매 메서드마다
  String token = extractTokenFromHeader(req);  // 반복!
  if (!validateToken(token)) {
    throw new UnauthorizedException();
  }
}

// 실수 3: 분산 추적 미구현
// Trace ID 관리 X → 요청 추적 불가능
// A→B→C 체인에서 실패하면 어디서인지 알 수 없음
```

---

## ✨ 올바른 접근

```
// 올바른 접근 1: 메타데이터 전송
// 클라이언트
Metadata headers = new Metadata();
Metadata.Key<String> authKey = 
  Metadata.Key.of("authorization", Metadata.ASCII_STRING_MARSHALLER);
headers.put(authKey, "Bearer eyJhbG...");

GetUserRequest request = GetUserRequest.newBuilder()
  .setUserId("U1")
  .build();

stub.withCompression("gzip")
  .withInterceptors(new ClientInterceptor() {...})
  .getUser(request);

// 올바른 접근 2: Interceptor로 자동 처리
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
  
  @Override
  public void getUser(GetUserRequest req, StreamObserver<User> res) {
    // Interceptor가 이미 Context에 토큰 저장함
    String token = SecurityContextHolder.getContext().getToken();
    // 토큰 검증 불필요 (Interceptor가 이미 했음)
  }
}

// 올바른 접근 3: 분산 추적
Metadata headers = new Metadata();
String traceId = generateTraceId();  // "trace-abc-123"
headers.put(
  Metadata.Key.of("x-trace-id", Metadata.ASCII_STRING_MARSHALLER),
  traceId
);

// 서버에서 자동 전파
A→B→C 체인 시
B가 받은 trace-id를 C에 자동 전달
최종적으로 로그에 모두 trace-id 기록
→ 전체 요청 경로 추적 가능
```

---

## 🔬 내부 동작 원리

### 메타데이터와 HTTP/2 헤더

```
gRPC 메타데이터 ↔ HTTP/2 헤더

HTTP/2 Frame:
┌──────────────────────────────────────┐
│HEADERS Frame                         │
├──────────────────────────────────────┤
│ :authority: example.com              │
│ :method: POST                        │
│ :path: /user.UserService/GetUser    │
│ :scheme: https                       │
│ content-type: application/grpc       │
│ te: trailers                         │
│ authorization: Bearer token...      │ ← 메타데이터
│ x-trace-id: trace-abc-123           │ ← 메타데이터
│ x-request-id: req-456-789           │ ← 메타데이터
│ user-agent: grpc-java/1.45           │
└──────────────────────────────────────┘

gRPC Metadata:
  - HTTP/2 헤더프레임에 저장
  - Key-Value 쌍
  - 바이너리 값도 가능
  
메타데이터 종류:
  1. 일반 메타데이터 (Request Headers)
     HEADERS 프레임에 포함
     
  2. Trailer (Response Trailers)
     DATA 전송 후 마지막에 전송
     Status Code, Status Message 포함
```

### Metadata API

```
메타데이터 정의:
  // String 메타데이터
  Metadata.Key<String> KEY = 
    Metadata.Key.of("x-trace-id", Metadata.ASCII_STRING_MARSHALLER);
  
  // 바이너리 메타데이터
  Metadata.Key<byte[]> BINARY_KEY =
    Metadata.Key.of("x-binary-data-bin", Metadata.BINARY_BYTE_MARSHALLER);

클라이언트 전송:
  // 방법 1: Metadata 객체
  Metadata metadata = new Metadata();
  metadata.put(KEY, "trace-123");
  
  // 방법 2: withMetadata()
  stub.withMetadata(metadata)
    .getUser(request);
  
  // 방법 3: Interceptor
  stub.withInterceptors(new ClientInterceptor() {
    @Override
    public <ReqT, RespT> ClientCall<ReqT, RespT> 
      interceptCall(MethodDescriptor<ReqT, RespT> method, 
                   CallOptions options, Channel next) {
      
      ClientCall<ReqT, RespT> call = next.newCall(method, options);
      return new ForwardingClientCall.SimpleForwardingClientCall(call) {
        @Override
        public void start(Listener<RespT> responseListener, Metadata headers) {
          headers.put(KEY, "trace-456");
          super.start(responseListener, headers);
        }
      };
    }
  });

서버 수신:
  // Context에서 추출
  String traceId = Context.current()
    .get(ContextKeys.TRACE_ID);
  
  // 또는 Interceptor에서
  @GrpcGlobalInterceptor
  public static class ServerLoggingInterceptor 
    implements ServerInterceptor {
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> 
      interceptCall(ServerCall<ReqT, RespT> call,
                   Metadata headers,
                   ServerCallHandler<ReqT, RespT> next) {
      
      String traceId = headers.get(KEY);
      System.out.println("Trace ID: " + traceId);
      
      return next.startCall(call, headers);
    }
  }
```

### 일반 메타데이터 vs Trailer

```
Request Flow:

┌─────────────────────────────────────────────────┐
│Client sends HEADERS (메타데이터)               │
├─────────────────────────────────────────────────┤
│ Authorization: Bearer token                     │
│ X-Trace-ID: trace-123                          │
│ X-Request-ID: req-456                          │
│ Custom-Header: value                           │
└─────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────┐
│Server receives & processes                     │
└─────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────┐
│Server sends DATA (메시지)                      │
├─────────────────────────────────────────────────┤
│ User{user_id:"U1", name:"Alice"...}           │
└─────────────────────────────────────────────────┘
        ↓
┌─────────────────────────────────────────────────┐
│Server sends TRAILERS (응답 메타데이터)         │
├─────────────────────────────────────────────────┤
│ :status: 200 (또는 에러코드)                   │
│ grpc-message: OK (또는 에러메시지)             │
│ grpc-encoding: gzip                            │
│ X-Response-Time: 145ms                         │
│ X-Server-Version: v1.45                        │
└─────────────────────────────────────────────────┘

메타데이터 vs Trailer:
  메타데이터 (HEADERS):
    - 요청 시작 전 전송
    - 인증, 권한, 추적 정보
    - 항상 있음
    
  Trailer (마지막):
    - 응답 메시지 전송 후
    - Status Code, Status Message
    - 처리 시간, 버전 정보 등
    - 선택적
```

### 실전 예시: 분산 추적 (Distributed Tracing)

```
메타데이터로 Trace ID 전파:

A (시작)
  ├─ Trace-ID: "trace-abc-123"
  ├─ Span-ID: "span-1"
  └─ RPC 호출 B

B (중간)
  ├─ 수신: Trace-ID: "trace-abc-123"
  ├─ 생성: Span-ID: "span-2"
  ├─ 계속 전파: Trace-ID, Span-ID
  └─ RPC 호출 C

C (최종)
  ├─ 수신: Trace-ID: "trace-abc-123", Span-ID: "span-2"
  ├─ 생성: Span-ID: "span-3"
  └─ 처리 후 반환

로그:
  [trace-abc-123, span-1] A: Starting request
  [trace-abc-123, span-2] B: Received from A
  [trace-abc-123, span-3] C: Processing
  [trace-abc-123, span-3] C: Complete
  [trace-abc-123, span-2] B: Received from C
  [trace-abc-123, span-1] A: Complete

모니터링:
  trace-abc-123으로 조회하면:
  A→B→C 전체 요청 경로, 각 단계 처리시간 시각화
```

---

## 💻 실전 실험

```java
// 메타데이터 Interceptor 구현

// 클라이언트 Interceptor
public class ClientMetadataInterceptor implements ClientInterceptor {
  @Override
  public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
    MethodDescriptor<ReqT, RespT> method,
    CallOptions options,
    Channel next) {
    
    return new ForwardingClientCall.SimpleForwardingClientCall<>(
      next.newCall(method, options)) {
      
      @Override
      public void start(Listener<RespT> responseListener, Metadata headers) {
        // Trace ID 생성 또는 기존 것 사용
        String traceId = generateOrGetTraceId();
        headers.put(
          Metadata.Key.of("x-trace-id", Metadata.ASCII_STRING_MARSHALLER),
          traceId);
        
        // 요청 ID
        headers.put(
          Metadata.Key.of("x-request-id", Metadata.ASCII_STRING_MARSHALLER),
          UUID.randomUUID().toString());
        
        // 타임스탠프
        headers.put(
          Metadata.Key.of("x-request-time", Metadata.ASCII_STRING_MARSHALLER),
          String.valueOf(System.currentTimeMillis()));
        
        super.start(responseListener, headers);
      }
    };
  }
  
  private String generateOrGetTraceId() {
    // Context에서 가져오거나 생성
    return UUID.randomUUID().toString();
  }
}

// 서버 Interceptor
@GrpcGlobalInterceptor
public class ServerMetadataInterceptor implements ServerInterceptor {
  private static final Logger logger = LoggerFactory.getLogger("gRPC");
  
  @Override
  public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
    ServerCall<ReqT, RespT> call,
    Metadata headers,
    ServerCallHandler<ReqT, RespT> next) {
    
    // 메타데이터 추출
    String traceId = headers.get(
      Metadata.Key.of("x-trace-id", Metadata.ASCII_STRING_MARSHALLER));
    String requestId = headers.get(
      Metadata.Key.of("x-request-id", Metadata.ASCII_STRING_MARSHALLER));
    
    // MDC에 저장 (로깅)
    MDC.put("traceId", traceId);
    MDC.put("requestId", requestId);
    
    logger.info("Request received: {}", call.getMethodDescriptor().getFullMethodName());
    
    return next.startCall(call, headers);
  }
}
```

---

## 📊 성능/비용 비교

```
메타데이터 크기:
  Authorization: 300B (토큰)
  X-Trace-ID: 36B (UUID)
  X-Request-ID: 36B
  X-Custom-Headers: 100B
  Total: ~500B per request

네트워크:
  100만 요청: 500B × 1M = 500MB
  월: 500MB × 30일 = 15GB
  비용: ~$150 (GB당 $10)

메타데이터 vs 메시지 필드:
  메시지에 포함: 메시지 크기 증가 + 직렬화 오버헤드
  메타데이터로 전송: HTTP/2 헤더 압축 (50~90%)
  → 효율성 우위
```

---

## ⚖️ 트레이드오프

```
✅ 메타데이터:
├─ 메시지와 분리
├─ HTTP/2 헤더 압축
├─ Interceptor로 자동화
└─ 표준화 가능

❌ 메타데이터:
├─ 추가 API 학습
├─ Context 관리 복잡
├─ 유형별 직렬화 필요
└─ 디버깅 어려움
```

---

## 📌 핵심 정리

```
메타데이터 = HTTP/2 헤더
  Key-Value 쌍
  String 또는 바이너리
  HTTP/2 압축
  
일반 메타데이터 (HEADERS):
  요청 시작 시 전송
  인증, 추적, 요청ID
  
Trailer (마지막):
  응답 후 전송
  Status Code, Message
  선택적
  
자동화:
  Interceptor로 모든 RPC에 적용
  @GrpcGlobalInterceptor 어노테이션
  
분산 추적:
  Trace-ID 메타데이터로 전파
  A→B→C 체인 전체 추적 가능
  로그/모니터링 통합
```

---

<div align="center">

**[⬅️ 이전: 에러 처리](./02-error-handling.md)** | **[홈으로 🏠](../README.md)** | **[다음: Deadline과 Timeout ➡️](./04-deadline-timeout.md)**

</div>
