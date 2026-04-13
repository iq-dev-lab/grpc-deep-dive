# JWT 기반 인증 — Interceptor로 토큰 주입/검증

---

## 🎯 핵심 질문

- CallCredentials를 사용해 모든 RPC에 자동으로 JWT를 주입하는 방법은?
- ServerInterceptor에서 Authorization 메타데이터를 추출하고 검증하는 흐름은?
- gRPC Context에 인증 정보를 저장하고 비즈니스 로직에서 꺼내는 방법은?
- 스트리밍 중 JWT가 만료되면 어떻게 처리하는가?
- Status.UNAUTHENTICATED와 Status.PERMISSION_DENIED의 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC는 HTTP/2 위에서 동작하므로, HTTP의 Authorization 헤더를 메타데이터로 전달할 수 있습니다. JWT를 사용하면 토큰 기반 인증으로 상태 없는(stateless) 인증 시스템을 구축할 수 있어, 마이크로서비스 환경에서 확장성이 우수합니다. CallCredentials로 클라이언트 측에서 자동 토큰 주입, ServerInterceptor로 서버 측에서 자동 검증하면, 비즈니스 로직과 인증 로직을 완전히 분리할 수 있습니다.

---

## 😱 흔한 실수

```java
// 실수 1: 토큰을 메타데이터가 아닌 요청 메시지에 담기
message LoginRequest {
    string username = 1;
    string password = 2;
    string token = 3;  // 잘못된 설계!
}

// 문제:
// - Proto 계약이 오염됨
// - 모든 RPC에서 토큰 필드 필요
// - 토큰이 Proto로 직렬화됨 (느림)


// 실수 2: 매 요청마다 JWT 서명 검증 (공개키 페칭 포함)
@Override
public void intercept(ServerCall<ReqT, RespT> call,
        Metadata headers, ServerCallHandler<ReqT, RespT> next) {
    
    String token = headers.get("authorization");
    
    // 매번 공개키를 원격 서버에서 가져옴 (네트워크 I/O!)
    PublicKey key = fetchPublicKeyFromAuthServer();
    
    // 검증 수행
    verify(token, key);
    
    // 결과: 매 요청마다 100ms 추가 지연
}


// 실수 3: Context 대신 ThreadLocal 사용
private static ThreadLocal<User> currentUser = new ThreadLocal<>();

@Override
public void intercept(...) {
    currentUser.set(user);  // Thread Local에 저장
    // 비동기 처리에서 컨텍스트 손실!
}

// 비즈니스 로직 (다른 스레드에서 실행)
executorService.submit(() -> {
    User user = currentUser.get();  // null! (컨텍스트 손실)
});
```

---

## ✨ 올바른 접근

```java
/**
 * 클라이언트: CallCredentials로 자동 토큰 주입
 */
public class JwtCallCredentials extends CallCredentials {
    
    private final TokenProvider tokenProvider;
    
    public JwtCallCredentials(TokenProvider tokenProvider) {
        this.tokenProvider = tokenProvider;
    }
    
    @Override
    public void applyRequestMetadata(RequestInfo requestInfo,
            Executor appExecutor, MetadataApplier applier) {
        
        appExecutor.execute(() -> {
            try {
                // 토큰 생성 (캐싱됨)
                String token = tokenProvider.getToken();
                
                Metadata metadata = new Metadata();
                metadata.put(
                    Metadata.Key.of("authorization", 
                        Metadata.ASCII_STRING_MARSHALLER),
                    "Bearer " + token);
                
                applier.apply(metadata);
                
            } catch (Exception e) {
                applier.fail(Status.UNAUTHENTICATED
                    .withDescription("Failed to obtain token")
                    .asException());
            }
        });
    }
    
    @Override
    public void thisUsesUnstableApi() {}
}

/**
 * 클라이언트: 채널에 자격증명 추가
 */
public class AuthenticatedClient {
    
    public ManagedChannel createAuthenticatedChannel(String host, int port) {
        // TokenProvider 설정
        TokenProvider tokenProvider = new JwtTokenProvider(
            "secret-key",
            "service-account@example.com");
        
        // CallCredentials 생성
        CallCredentials credentials = new JwtCallCredentials(tokenProvider);
        
        // 채널 생성
        ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext()
            .build();
        
        // 모든 RPC에 자동으로 자격증명 추가
        return ClientInterceptors.intercept(
            channel, new ClientAuthInterceptor(credentials));
    }
}

/**
 * 서버: ServerInterceptor로 자동 검증
 */
public class JwtAuthServerInterceptor implements ServerInterceptor {
    
    private final JwtVerifier jwtVerifier;
    private static final Context.Key<AuthContext> AUTH_CONTEXT_KEY =
        Context.key("auth-context");
    
    public JwtAuthServerInterceptor(JwtVerifier jwtVerifier) {
        this.jwtVerifier = jwtVerifier;
    }
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        // Authorization 메타데이터 추출
        String authHeader = headers.get(
            Metadata.Key.of("authorization", 
                Metadata.ASCII_STRING_MARSHALLER));
        
        // 토큰 검증
        AuthContext authContext = null;
        if (authHeader != null && authHeader.startsWith("Bearer ")) {
            String token = authHeader.substring(7);
            try {
                authContext = jwtVerifier.verify(token);
                
            } catch (JwtVerificationException e) {
                call.close(Status.UNAUTHENTICATED
                    .withDescription("Invalid token: " + e.getMessage()),
                    new Metadata());
                return new ServerCall.Listener<ReqT>() {};
            }
        } else {
            // 토큰 없음
            call.close(Status.UNAUTHENTICATED
                .withDescription("Missing authorization header"),
                new Metadata());
            return new ServerCall.Listener<ReqT>() {};
        }
        
        // Context에 인증 정보 저장 (Context 사용)
        Context newContext = Context.current()
            .withValue(AUTH_CONTEXT_KEY, authContext);
        
        // Context를 적용하면서 다음 interceptor 호출
        return Contexts.interceptCall(newContext, call, headers, next);
    }
    
    public static AuthContext getCurrentAuthContext() {
        return AUTH_CONTEXT_KEY.get();
    }
}

/**
 * 비즈니스 로직: Context에서 인증 정보 조회
 */
public class MyServiceImpl extends MyServiceGrpc.MyServiceImplBase {
    
    @Override
    public void myMethod(MyRequest request,
            StreamObserver<MyResponse> responseObserver) {
        
        // Context에서 인증 정보 조회 (어디서든 가능)
        AuthContext authContext = 
            JwtAuthServerInterceptor.getCurrentAuthContext();
        
        if (authContext == null) {
            responseObserver.onError(Status.UNAUTHENTICATED.asException());
            return;
        }
        
        // 사용자 정보 사용
        String userId = authContext.getUserId();
        List<String> permissions = authContext.getPermissions();
        
        log.info("Request from user: {} with permissions: {}",
            userId, permissions);
        
        // 비즈니스 로직
        MyResponse response = MyResponse.newBuilder()
            .setResult("OK for " + userId)
            .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}

/**
 * JWT 토큰 생성 및 캐싱
 */
public class JwtTokenProvider implements TokenProvider {
    
    private final String secretKey;
    private final String subject;
    private volatile String cachedToken;
    private volatile long tokenExpiryTime;
    private final ScheduledExecutorService executor;
    
    public JwtTokenProvider(String secretKey, String subject) {
        this.secretKey = secretKey;
        this.subject = subject;
        this.executor = Executors.newScheduledThreadPool(1);
    }
    
    @Override
    public String getToken() throws Exception {
        long now = System.currentTimeMillis();
        
        // 토큰이 유효하면 캐시된 토큰 반환
        if (cachedToken != null && now < tokenExpiryTime - 60000) {
            return cachedToken;
        }
        
        // 토큰 갱신
        this.cachedToken = generateToken();
        this.tokenExpiryTime = now + (3600 * 1000);  // 1시간
        
        return cachedToken;
    }
    
    private String generateToken() {
        // JWT 생성 (HMAC SHA-256)
        Instant now = Instant.now();
        Instant expiresAt = now.plusSeconds(3600);
        
        return Jwts.builder()
            .setSubject(subject)
            .setIssuedAt(Date.from(now))
            .setExpiration(Date.from(expiresAt))
            .signWith(SignatureAlgorithm.HS256, secretKey.getBytes())
            .compact();
    }
}

/**
 * JWT 검증
 */
public class JwtVerifier {
    
    private final String secretKey;
    
    public JwtVerifier(String secretKey) {
        this.secretKey = secretKey;
    }
    
    public AuthContext verify(String token) throws JwtVerificationException {
        try {
            Claims claims = Jwts.parser()
                .setSigningKey(secretKey.getBytes())
                .parseClaimsJws(token)
                .getBody();
            
            return new AuthContext(
                claims.getSubject(),
                extractPermissions(claims));
            
        } catch (JwtException e) {
            throw new JwtVerificationException("Invalid token", e);
        }
    }
    
    private List<String> extractPermissions(Claims claims) {
        Object permsObj = claims.get("permissions");
        if (permsObj instanceof List) {
            return (List<String>) permsObj;
        }
        return Collections.emptyList();
    }
}

/**
 * 스트리밍 중 JWT 만료 처리
 */
public class StreamingAuthInterceptor implements ServerInterceptor {
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        // 초기 검증
        AuthContext authContext = verifyToken(headers);
        if (authContext == null) {
            call.close(Status.UNAUTHENTICATED.asException(), 
                new Metadata());
            return new ServerCall.Listener<ReqT>() {};
        }
        
        // 토큰 만료 시간이 1분 남았으면 경고
        if (authContext.getExpiresIn() < 60) {
            // 클라이언트에 "토큰 갱신하라"는 신호 전송 가능
            log.warn("Token expiring soon for user: {}", 
                authContext.getUserId());
        }
        
        Context newContext = Context.current()
            .withValue(AUTH_CONTEXT_KEY, authContext);
        
        return Contexts.interceptCall(newContext, call, headers, next);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. CallCredentials 동작 — 모든 RPC에 자동 주입

```
Request Flow:
┌──────────────────────────────────────────────────────┐
│ 클라이언트가 RPC 호출                               │
│ stub.myMethod(request)                              │
└──────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────┐
│ gRPC 내부: CallCredentials 실행                     │
│ (ClientAuthInterceptor가 가로챔)                    │
└──────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────┐
│ JwtCallCredentials.applyRequestMetadata()           │
│ - tokenProvider.getToken() 호출                     │
│ - JWT 토큰 획득                                     │
│ - metadata에 "authorization" 헤더 추가             │
└──────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────┐
│ 모든 RPC에 자동으로 메타데이터 추가됨               │
│ [HEADERS]                                            │
│ authorization: Bearer <JWT_TOKEN>                   │
│ content-type: application/grpc                      │
│ [DATA]                                              │
│ <Request Message>                                   │
└──────────────────────────────────────────────────────┘
```

### 2. ServerInterceptor 메타데이터 추출 흐름

```
Server 처리 순서:
┌────────────────────────────────────────────────────────┐
│ 1. HTTP/2 HEADERS Frame 수신                         │
│    authorization: Bearer eyJhbGc...                  │
└────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────┐
│ 2. ServerInterceptor.interceptCall() 호출            │
│    (Metadata 포함)                                   │
└────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────┐
│ 3. Authorization 헤더 추출                           │
│    authHeader = headers.get("authorization")        │
│    token = authHeader.substring(7)  // "Bearer " 제거
└────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────┐
│ 4. JWT 검증                                          │
│    AuthContext = jwtVerifier.verify(token)         │
│    - 서명 확인                                      │
│    - 만료 시간 확인                                │
│    - 권한 추출                                      │
└────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────┐
│ 5. Context에 저장                                    │
│    Context.current().withValue(AUTH_CONTEXT_KEY,    │
│      authContext)                                    │
└────────────────────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────────────────────┐
│ 6. 비즈니스 로직 실행                               │
│    myServiceImpl.myMethod()                         │
│    (Context에서 인증 정보 조회 가능)                │
└────────────────────────────────────────────────────────┘
```

### 3. Context로 인증 정보 전파 (Java 코드)

```java
/**
 * Context: ThreadLocal과 달리 비동기에도 안전
 */
public class ContextPropagation {
    
    // Context Key 정의
    static final Context.Key<AuthContext> AUTH_KEY = 
        Context.key("auth");
    
    // 비동기 작업에서 Context 전파
    public void asyncWorkWithContext(String token) {
        // 현재 Context에 값 저장
        Context ctx = Context.current()
            .withValue(AUTH_KEY, new AuthContext(token));
        
        // 비동기 작업에서 Context 사용
        ctx.run(() -> {
            // 다른 스레드에서 실행되어도 Context가 유지됨
            AuthContext auth = AUTH_KEY.get();
            log.info("In async: {}", auth);
        });
    }
}
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# JWT 인증 테스트

# 1. 토큰 생성
TOKEN=$(java -cp grpc-all.jar:. TokenGenerator \
  -subject "test-service" \
  -secret "my-secret-key")

echo "Generated Token: $TOKEN"

# 2. 올바른 토큰으로 요청
grpcurl -H "authorization: Bearer $TOKEN" \
  -plaintext \
  -d '{"message":"hello"}' \
  localhost:50051 MyService/MyMethod

# 3. 잘못된 토큰으로 요청
grpcurl -H "authorization: Bearer INVALID" \
  -plaintext \
  localhost:50051 MyService/MyMethod
# [Error] UNAUTHENTICATED: Invalid token

# 4. 토큰 없이 요청
grpcurl -plaintext \
  localhost:50051 MyService/MyMethod
# [Error] UNAUTHENTICATED: Missing authorization header
```

---

## 📊 성능/비용 비교

```
인증 방식별 성능 (1000개 RPC)

┌────────────────┬──────────┬─────────────┬──────────────┐
│ 방식           │ CPU 사용 │ 캐시 효과   │ 지연         │
├────────────────┼──────────┼─────────────┼──────────────┤
│ 매번 검증      │ 높음     │ 없음        │ +100ms       │
│ (공개키 페칭)  │          │             │              │
├────────────────┼──────────┼─────────────┼──────────────┤
│ 로컬 검증      │ 중간     │ 있음 (토큰) │ +2ms         │
│ (캐싱)         │          │             │              │
├────────────────┼──────────┼─────────────┼──────────────┤
│ OAuth 위임     │ 낮음     │ 있음 (JWT)  │ +1ms         │
│ (내부 서비스)  │          │             │              │
└────────────────┴──────────┴─────────────┴──────────────┘
```

---

## ⚖️ 트레이드오프

```
✅ 장점
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Stateless 인증 (확장성 우수)
2. CallCredentials로 자동 주입
3. Context로 안전한 정보 전파

❌ 제약사항
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 토큰 폐지 어려움 (만료까지 대기)
2. 토큰 크기 증가 (메타데이터 오버헤드)
```

---

## 📌 핵심 정리

```
JWT 기반 인증:

1. CallCredentials: 클라이언트 자동 토큰 주입
2. ServerInterceptor: 서버 자동 검증
3. Context: 비동기 안전 정보 전파
4. 토큰 캐싱: 매 요청마다 검증 X
5. Stateless: 마이크로서비스에 최적
```

---

## 🤔 생각해볼 문제

### Q1: CallCredentials와 ClientInterceptor의 차이는 무엇인가?

<details>
<summary>해설 보기</summary>

```
CallCredentials:
- 인증 정보만 추가 (자격증명)
- 모든 RPC에 자동 적용
- 사용: 토큰 주입

ClientInterceptor:
- RPC 전체를 가로챔
- 요청/응답 수정, 로깅, 추적
- 사용: 로깅, 메트릭, 타이밍
```
</details>

### Q2: JWT 토큰이 스트리밍 중간에 만료되면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

초기 검증(HEADERS)에서만 확인하므로, 스트리밍이 진행되는 동안 토큰이 만료되어도 영향 없습니다. 하지만 종료 후 재연결 시에는 새 토큰이 필요합니다.

해결책: 스트리밍 중간에 토큰 갱신 메시지 전송 가능 (커스텀 메시지).
</details>

---

<div align="center">

**[⬅️ 이전: TLS와 mTLS](./01-tls-mtls.md)** | **[홈으로 🏠](../README.md)** | **[다음: API 키와 서비스 간 인증 ➡️](./03-service-auth.md)**

</div>
