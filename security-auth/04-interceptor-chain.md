# Interceptor 체인 — gRPC 미들웨어 패턴

---

## 🎯 핵심 질문

- ClientInterceptor와 ServerInterceptor의 인터페이스 구조와 차이는?
- Interceptor 체인에서 실행 순서는 어떻게 결정되는가?
- 로깅, 인증, 분산 추적을 Interceptor로 구현하면 어떤 이점이 있는가?
- @GrpcGlobalInterceptor로 모든 서비스에 인터셉터를 적용하는 방법은?
- Interceptor가 Context를 통해 데이터를 전달하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Interceptor는 gRPC의 미들웨어 패턴입니다. 인증, 로깅, 메트릭, 분산 추적을 모든 RPC에서 일관되게 처리할 수 있고, 비즈니스 로직과 완전히 분리합니다. 올바른 Interceptor 체인으로 비기능 요구사항(NFR)을 깔끔하게 해결할 수 있습니다.

---

## 😱 흔한 실수

```java
// 실수 1: Interceptor 순서를 고려하지 않음
ServerBuilder.forPort(50051)
    .addService(new MyServiceImpl())
    .intercept(new LoggingInterceptor())  // 인증 전에 로깅!
    .intercept(new AuthenticationInterceptor())
    .build()
    .start();

// 문제: 미인증 요청도 로그에 기록 → 보안 이슈


// 실수 2: ClientInterceptor와 ServerInterceptor 개념 혼동
// ClientInterceptor를 Server에 추가하려고 시도
server.intercept(new ClientInterceptor() { ... });  // 컴파일 에러!


// 실수 3: Interceptor에서 예외를 삼키기
try {
    next.handle(request);
} catch (Exception e) {
    log.error("Error", e);
    // onError() 호출 안 함 → 클라이언트 영원히 대기
}
```

---

## ✨ 올바른 접근

```java
/**
 * ServerInterceptor 체인
 * 순서: 인증 → 로깅 → 분산추적
 */
public class ServerInterceptorChain {
    
    public Server createServerWithInterceptors(int port) {
        return ServerBuilder.forPort(port)
            .addService(new MyServiceImpl())
            
            // 1. 인증 (가장 먼저)
            .intercept(new AuthenticationInterceptor())
            
            // 2. 로깅 (인증 후)
            .intercept(new LoggingInterceptor())
            
            // 3. 분산추적 (마지막)
            .intercept(new TracingInterceptor())
            
            .build();
    }
}

/**
 * 인증 Interceptor
 */
public class AuthenticationInterceptor implements ServerInterceptor {
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        // 1. 토큰 검증
        String authHeader = headers.get(
            Metadata.Key.of("authorization", 
                Metadata.ASCII_STRING_MARSHALLER));
        
        if (authHeader == null) {
            call.close(Status.UNAUTHENTICATED.withDescription(
                "Missing authorization header"), new Metadata());
            return new ServerCall.Listener<ReqT>() {};
        }
        
        try {
            // 2. 토큰 해석
            String token = authHeader.substring(7);
            AuthContext authContext = verifyToken(token);
            
            // 3. Context에 저장
            Context newContext = Context.current()
                .withValue(AUTH_CONTEXT_KEY, authContext);
            
            // 4. 다음 Interceptor로 진행
            return Contexts.interceptCall(
                newContext, call, headers, next);
            
        } catch (Exception e) {
            call.close(Status.UNAUTHENTICATED, new Metadata());
            return new ServerCall.Listener<ReqT>() {};
        }
    }
    
    private AuthContext verifyToken(String token) {
        // 토큰 검증 로직
        return new AuthContext();
    }
}

/**
 * 로깅 Interceptor
 */
public class LoggingInterceptor implements ServerInterceptor {
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        String methodName = call.getMethodDescriptor().getFullMethodName();
        long startTime = System.currentTimeMillis();
        
        // 요청 로깅
        log.info("RPC started: {}", methodName);
        
        // 응답 결과를 감싸기
        return new ForwardingServerCallListener.SimpleForwardingServerCallListener<ReqT>(
            next.startCall(
                new ForwardingServerCall.SimpleForwardingServerCall<ReqT, RespT>(call) {
                    @Override
                    public void close(Status status, Metadata trailers) {
                        long duration = System.currentTimeMillis() - startTime;
                        log.info("RPC completed: {} [{}ms] Status: {}",
                            methodName, duration, status.getCode());
                        super.close(status, trailers);
                    }
                },
                headers)) {};
    }
}

/**
 * 분산추적 Interceptor
 */
public class TracingInterceptor implements ServerInterceptor {
    
    private final Tracer tracer;
    
    public TracingInterceptor(Tracer tracer) {
        this.tracer = tracer;
    }
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        String methodName = call.getMethodDescriptor().getFullMethodName();
        String traceId = headers.get(
            Metadata.Key.of("x-trace-id", 
                Metadata.ASCII_STRING_MARSHALLER));
        
        // Span 생성
        Span span = tracer.buildSpan(methodName)
            .withTag("trace.id", traceId)
            .start();
        
        return Contexts.interceptCall(
            Context.current().withValue(SPAN_KEY, span),
            call, headers, new ServerCallHandler<ReqT, RespT>() {
                @Override
                public ServerCall.Listener<ReqT> startCall(
                        ServerCall<ReqT, RespT> call,
                        Metadata headers) {
                    try (Scope scope = tracer.scopeManager()
                        .activate(span)) {
                        return next.startCall(call, headers);
                    }
                }
            });
    }
}

/**
 * ClientInterceptor 체인
 */
public class ClientInterceptorChain {
    
    public ManagedChannel createAuthenticatedChannel(
            String host, int port) {
        
        return ClientInterceptors.intercept(
            ManagedChannelBuilder.forAddress(host, port)
                .usePlaintext()
                .build(),
            
            // 1. 토큰 주입
            new ClientInterceptor() {
                @Override
                public <ReqT, RespT> ClientCall<ReqT, RespT> 
                        interceptCall(MethodDescriptor<ReqT, RespT> method,
                        CallOptions callOptions, Channel next) {
                    
                    return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(
                        next.newCall(method, callOptions)) {
                        
                        @Override
                        public void start(Listener<RespT> responseListener,
                                Metadata headers) {
                            // 토큰 추가
                            headers.put(Metadata.Key.of("authorization",
                                Metadata.ASCII_STRING_MARSHALLER),
                                "Bearer " + getToken());
                            super.start(responseListener, headers);
                        }
                    };
                }
            },
            
            // 2. 로깅
            new ClientInterceptor() {
                @Override
                public <ReqT, RespT> ClientCall<ReqT, RespT> 
                        interceptCall(MethodDescriptor<ReqT, RespT> method,
                        CallOptions callOptions, Channel next) {
                    
                    long startTime = System.currentTimeMillis();
                    log.info("Client RPC: {}", method.getFullMethodName());
                    
                    return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(
                        next.newCall(method, callOptions)) {
                        
                        @Override
                        public void start(Listener<RespT> responseListener,
                                Metadata headers) {
                            super.start(
                                new ForwardingClientCallListener.SimpleForwardingClientCallListener<RespT>(
                                    responseListener) {
                                    
                                    @Override
                                    public void onClose(Status status,
                                            Metadata trailers) {
                                        long duration = System.currentTimeMillis() - startTime;
                                        log.info("Client RPC completed: {}ms",
                                            duration);
                                        super.onClose(status, trailers);
                                    }
                                },
                                headers);
                        }
                    };
                }
            });
    }
    
    private String getToken() {
        return "valid-token";
    }
}

/**
 * Spring Boot: @GrpcGlobalInterceptor 사용
 */
@Configuration
public class GrpcInterceptorConfig {
    
    @Bean
    @GrpcGlobalServerInterceptor
    public AuthenticationInterceptor authenticationInterceptor() {
        return new AuthenticationInterceptor();
    }
    
    @Bean
    @GrpcGlobalServerInterceptor
    public LoggingInterceptor loggingInterceptor() {
        return new LoggingInterceptor();
    }
    
    @Bean
    @GrpcGlobalServerInterceptor
    public TracingInterceptor tracingInterceptor(Tracer tracer) {
        return new TracingInterceptor(tracer);
    }
    
    // 클라이언트 Interceptor
    @Bean
    @GrpcGlobalClientInterceptor
    public ClientInterceptor clientAuthInterceptor() {
        return new ClientAuthInterceptor(getTokenProvider());
    }
    
    private TokenProvider getTokenProvider() {
        return () -> "token";
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Interceptor 체인 실행 순서

```
ServerInterceptor 호출 순서:
┌─────────────────────────────────────────────────────────┐
│ ClientRequest                                           │
│ [HEADERS with authorization token]                     │
└────────────────────┬────────────────────────────────────┘
                     │
        ┌────────────▼───────────────┐
        │ AuthenticationInterceptor  │
        │ - 토큰 검증               │
        │ - Context 저장            │
        │ - 인증 실패 → close()     │
        └────────────┬───────────────┘
                     │
        ┌────────────▼───────────────┐
        │ LoggingInterceptor        │
        │ - startTime 기록          │
        │ - 메서드 이름 로깅        │
        │ - 응답 결과 로깅          │
        └────────────┬───────────────┘
                     │
        ┌────────────▼───────────────┐
        │ TracingInterceptor        │
        │ - Span 생성               │
        │ - trace-id 추적           │
        │ - Span 클로즈             │
        └────────────┬───────────────┘
                     │
        ┌────────────▼───────────────┐
        │ ServiceImpl.method()       │
        │ - 비즈니스 로직            │
        │ - Context 조회            │
        └────────────┬───────────────┘
                     │
                [응답]


ClientInterceptor 호출 순서:
┌──────────────────────────────────────────────────────┐
│ stub.myMethod(request)                              │
└──────────────┬───────────────────────────────────────┘
               │
    ┌──────────▼───────────────┐
    │ 토큰 주입 Interceptor    │
    │ - Authorization 헤더     │
    │ - Bearer token 추가      │
    └──────────┬───────────────┘
               │
    ┌──────────▼───────────────┐
    │ 로깅 Interceptor         │
    │ - RPC 호출 기록          │
    │ - 완료 시간 기록         │
    └──────────┬───────────────┘
               │
        [서버로 전송]
```

### 2. Interceptor의 데이터 흐름

```java
/**
 * Interceptor가 데이터를 다음 Interceptor로 전달하는 방법
 */

// 방법 1: Context 사용 (권장)
Context newContext = Context.current()
    .withValue(MY_KEY, myValue);
return Contexts.interceptCall(newContext, call, headers, next);

// 방법 2: Metadata 사용 (클라이언트→서버)
headers.put(MY_METADATA_KEY, myValue);
return next.startCall(new ForwardingServerCall<..>() { ... });

// 방법 3: ServerCall 감싸기 (응답 수정)
return next.startCall(new ForwardingServerCall<..>() {
    @Override
    public void close(Status status, Metadata trailers) {
        // 응답 수정 가능
        super.close(status, trailers);
    }
}, headers);
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# Interceptor 체인 테스트

# 1. 서버 시작 (모든 Interceptor 포함)
java -cp grpc-all.jar:. InterceptorChainServer &

# 2. 클라이언트 요청 (로그 확인)
java -cp grpc-all.jar:. InterceptorChainClient

# 로그 출력 순서:
# [AUTH] Validating token...
# [AUTH] Token valid, user: john
# [LOG] RPC started: MyService/MyMethod
# [TRACE] Span created: trace-123
# [SERVICE] Processing request...
# [TRACE] Span closed
# [LOG] RPC completed: 45ms
```

---

## 📊 성능 비교

```
Interceptor 오버헤드 (1000개 RPC)

┌──────────────┬────────┬─────────┬──────────────┐
│ Interceptor  │ CPU %  │ 지연    │ 메모리       │
├──────────────┼────────┼─────────┼──────────────┤
│ 없음         │ 0%     │ 0ms     │ 0MB          │
│ 로깅 1개     │ 2%     │ 1ms     │ 10MB         │
│ 3개 체인     │ 5%     │ 3ms     │ 30MB         │
│ 5개 체인     │ 10%    │ 5ms     │ 50MB         │
└──────────────┴────────┴─────────┴──────────────┘
```

---

## 📌 핵심 정리

```
Interceptor 체인:

1. 순서 중요 (인증 → 로깅 → 추적)
2. Context로 데이터 전달
3. 예외 처리 필수 (onError())
4. Spring: @GrpcGlobalInterceptor
5. 비기능 요구사항 분리
```

---

<div align="center">

**[⬅️ 이전: API 키와 서비스 간 인증](./03-service-auth.md)** | **[홈으로 🏠](../README.md)** | **[다음: 채널 보안 설정 ➡️](./05-channel-security.md)**

</div>
