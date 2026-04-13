# Spring Security 통합 — gRPC 메서드 레벨 권한

---

## 🎯 핵심 질문

- gRPC 서버에 Spring Security를 적용할 때 SecurityContext는 어떻게 설정되는가?
- @PreAuthorize를 gRPC 메서드에 적용하려면 무엇이 필요한가?
- JWT Interceptor와 SecurityContextHolder를 연동하는 방법은?
- 인증 실패 시 gRPC Status.UNAUTHENTICATED로 변환하는 방법은?
- gRPC의 비동기 처리에서 ThreadLocal 기반 SecurityContext의 문제점은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC는 기본적으로 Spring Security와 통합되지 않습니다. Interceptor에서 JWT를 파싱하고 ThreadLocal의 SecurityContext를 설정해야 @PreAuthorize가 작동합니다. 비동기 스트리밍 처리에서는 더 복잡해지는데, 이를 올바르게 처리하지 않으면 보안 우회 또는 NPE를 마주합니다.

---

## 😱 흔한 실수 (Before)

```java
// 실수 1: SecurityContext 설정 없이 @PreAuthorize 사용
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    @PreAuthorize("hasRole('ADMIN')")  // ❌ 작동 안 함
    @Override
    public void getUser(GetUserRequest request,
            StreamObserver<GetUserResponse> responseObserver) {
        // SecurityContextHolder.getContext()는 null
    }
}

// 실수 2: JWT를 파싱하지만 SecurityContext에 설정 안 함
@GrpcService
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {
    @Override
    public void createOrder(CreateOrderRequest request,
            StreamObserver<CreateOrderResponse> responseObserver) {
        String jwt = extractJwt();  // JWT 파싱함
        // 하지만 SecurityContext에 설정 안 함 → 권한 검사 안 됨
    }
}

// 실수 3: 인증 실패를 RuntimeException으로 throw
@GrpcService
public class AuthServiceImpl extends AuthServiceGrpc.AuthServiceImplBase {
    @Override
    public void authenticate(AuthRequest request,
            StreamObserver<AuthResponse> responseObserver) {
        if (!isValid(request)) {
            throw new RuntimeException("Invalid token");  
            // ❌ gRPC가 INTERNAL(500)으로 처리
        }
    }
}

// 실수 4: 비동기 처리에서 SecurityContext 전파 안 함
@GrpcService
public class StreamServiceImpl 
        extends StreamServiceGrpc.StreamServiceImplBase {
    @Override
    public void streamOrders(OrderFilter filter,
            StreamObserver<Order> responseObserver) {
        Flux.fromIterable(getOrders())
            .subscribeOn(Schedulers.boundedElastic())
            .subscribe(order -> {
                // 다른 스레드에서는 SecurityContext가 null
                responseObserver.onNext(order);
            });
    }
}
```

---

## ✨ 올바른 접근 (After)

```yaml
# application.yml: Spring Security 설정
spring:
  security:
    oauth2:
      resource-server:
        jwt:
          issuer-uri: https://auth-server.com
          jwk-set-uri: https://auth-server.com/.well-known/jwks.json
```

```java
// ServerInterceptor: JWT 파싱 및 SecurityContext 설정
@Component
public class JwtServerInterceptor implements ServerInterceptor {
    
    private final JwtTokenProvider jwtProvider;
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        String authToken = headers.get(
            Metadata.Key.of("authorization", ASCII_STRING_MARSHALLER)
        );
        
        if (authToken != null && authToken.startsWith("Bearer ")) {
            String token = authToken.substring(7);
            
            try {
                Authentication auth = jwtProvider.getAuthentication(token);
                
                // SecurityContextHolder에 설정
                SecurityContextHolder.getContext()
                    .setAuthentication(auth);
                
                // 컨텍스트를 메타데이터에 저장 (비동기 처리용)
                call.attributes().put(
                    Context.KEY_SECURITY_CONTEXT,
                    SecurityContextHolder.getContext()
                );
                
            } catch (JwtException e) {
                call.close(Status.UNAUTHENTICATED
                    .withDescription("Invalid JWT token"), 
                    new Metadata());
                return new ServerCall.Listener<ReqT>() {};
            }
        }
        
        return next.startCall(call, headers);
    }
}

// gRPC Service: @PreAuthorize 적용
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    
    @PreAuthorize("hasRole('ADMIN')")  // ✅ 작동함
    @Override
    public void getUser(GetUserRequest request,
            StreamObserver<GetUserResponse> responseObserver) {
        
        Authentication auth = 
            SecurityContextHolder.getContext().getAuthentication();
        
        if (auth == null || !auth.isAuthenticated()) {
            responseObserver.onError(
                Status.UNAUTHENTICATED.asException()
            );
            return;
        }
        
        GetUserResponse response = GetUserResponse.newBuilder()
                .setId(request.getId())
                .setName("Admin User")
                .build();
        
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}

// Exception Handler: 예외를 gRPC Status로 변환
@GrpcExceptionHandler
public StatusRuntimeException handleAuthException(
        AccessDeniedException ex) {
    return Status.PERMISSION_DENIED
        .withDescription("Access denied: " + ex.getMessage())
        .asException();
}

// 비동기 처리: SecurityContext 전파
@GrpcService
public class StreamServiceImpl 
        extends StreamServiceGrpc.StreamServiceImplBase {
    
    @Override
    public void streamOrders(OrderFilter filter,
            StreamObserver<Order> responseObserver) {
        
        // 현재 SecurityContext 캡처
        SecurityContext securityContext = 
            SecurityContextHolder.getContext();
        
        Flux.fromIterable(getOrders(filter))
            .subscribeOn(Schedulers.boundedElastic())
            .subscribe(
                order -> {
                    // 비동기 스레드에서 SecurityContext 복원
                    SecurityContextHolder.setContext(
                        securityContext
                    );
                    
                    try {
                        responseObserver.onNext(order);
                    } finally {
                        SecurityContextHolder.clearContext();
                    }
                },
                error -> responseObserver.onError(error),
                responseObserver::onCompleted
            );
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Spring Security + gRPC 통합 흐름

```
┌─────────────────────────────────────────┐
│  gRPC Client Request 도착                │
├─────────────────────────────────────────┤
│ Metadata:                               │
│  Authorization: Bearer eyJhbGc...      │
└────────────┬────────────────────────────┘
             │
             ▼
    ┌────────────────────┐
    │ JwtServerInterceptor│
    │ - 헤더에서 토큰    │
    │   추출 및 파싱     │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ JwtTokenProvider   │
    │ - JWT 검증         │
    │ - Authentication   │
    │   객체 생성        │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ SecurityContext    │
    │ 설정 (ThreadLocal) │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ @GrpcService 메서드│
    │ 실행               │
    └────────┬───────────┘
             │
      ┌──────┴──────┐
      ▼             ▼
 ✅ 성공    ❌ AccessDeniedException
      │             │
      └──────┬──────┘
             ▼
    ┌────────────────────┐
    │ @GrpcExceptionHandler
    │ 예외 → Status     │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ gRPC Client 응답   │
    │ Status: OK 또는    │
    │         PERMISSION_│
    │         DENIED     │
    └────────────────────┘
```

### 2. ServerInterceptor에서 JWT 파싱 → SecurityContext 설정

```java
@Component
public class JwtServerInterceptor implements ServerInterceptor {
    
    private static final Metadata.Key<String> AUTHORIZATION_KEY =
            Metadata.Key.of("authorization", 
                           ASCII_STRING_MARSHALLER);
    
    private final JwtTokenProvider jwtProvider;
    private final AuthenticationManager authenticationManager;
    
    public JwtServerInterceptor(
            JwtTokenProvider jwtProvider,
            AuthenticationManager authenticationManager) {
        this.jwtProvider = jwtProvider;
        this.authenticationManager = authenticationManager;
    }
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        try {
            String authHeader = headers.get(AUTHORIZATION_KEY);
            
            if (authHeader == null || 
                !authHeader.startsWith("Bearer ")) {
                // 토큰 없으면 익명 사용자
                return next.startCall(call, headers);
            }
            
            String token = authHeader.substring(7);
            
            // JWT 검증 및 클레임 추출
            Claims claims = jwtProvider.getClaimsFromToken(token);
            String username = claims.getSubject();
            String roles = (String) claims.get("roles");
            
            // Authentication 객체 생성
            List<GrantedAuthority> authorities = 
                Arrays.stream(roles.split(","))
                    .map(SimpleGrantedAuthority::new)
                    .collect(Collectors.toList());
            
            Authentication authentication =
                new UsernamePasswordAuthenticationToken(
                    username,
                    null,
                    authorities
                );
            
            // SecurityContext에 설정
            SecurityContext context = 
                SecurityContextHolder.createEmptyContext();
            context.setAuthentication(authentication);
            SecurityContextHolder.setContext(context);
            
            // 비동기 처리를 위해 Context에도 저장
            Context grpcContext = Context.current()
                .withValue(
                    SECURITY_CONTEXT_KEY,
                    context
                );
            
            return grpcContext.wrap(
                next.startCall(call, headers)
            );
            
        } catch (JwtException | ExpiredJwtException e) {
            String description = "Authentication failed: " 
                + e.getMessage();
            call.close(
                Status.UNAUTHENTICATED
                    .withDescription(description),
                new Metadata()
            );
            return new ServerCall.Listener<ReqT>() {};
        } finally {
            // 요청 처리 후 Context 정리
            // (gRPC는 자동으로 처리)
        }
    }
}
```

### 3. @GrpcService에 @PreAuthorize 적용 설정

```java
// 1. @EnableGlobalMethodSecurity 활성화
@Configuration
@EnableGlobalMethodSecurity(
    prePostEnabled = true,  // @PreAuthorize 활성화
    securedEnabled = true,  // @Secured 활성화
    jsr250Enabled = true    // @RolesAllowed 활성화
)
public class SecurityConfig {
    
    @Bean
    public AuthenticationManager authenticationManager(
            AuthenticationProvider provider) {
        return new ProviderManager(provider);
    }
}

// 2. @GrpcService에 @PreAuthorize 적용
@GrpcService
@Slf4j
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    
    private final UserRepository userRepository;
    
    // 누구나 접근 가능
    @Override
    public void listUsers(Empty request,
            StreamObserver<User> responseObserver) {
        userRepository.findAll()
            .forEach(responseObserver::onNext);
        responseObserver.onCompleted();
    }
    
    // ADMIN 역할만 접근 가능
    @PreAuthorize("hasRole('ADMIN')")
    @Override
    public void deleteUser(DeleteUserRequest request,
            StreamObserver<Empty> responseObserver) {
        
        Authentication auth = 
            SecurityContextHolder.getContext()
                .getAuthentication();
        
        log.info("Delete user requested by: {}", 
                 auth.getName());
        
        userRepository.deleteById(request.getId());
        responseObserver.onNext(Empty.getDefaultInstance());
        responseObserver.onCompleted();
    }
    
    // USER 이상 역할 필요
    @PreAuthorize("hasAnyRole('USER', 'ADMIN')")
    @Override
    public void updateUser(UpdateUserRequest request,
            StreamObserver<User> responseObserver) {
        User updated = userRepository.update(
            request.getId(),
            request.getName()
        );
        responseObserver.onNext(updated);
        responseObserver.onCompleted();
    }
    
    // 현재 사용자만 자신의 데이터 접근
    @PreAuthorize("@userService.isOwner(#request.id)")
    @Override
    public void getOwnProfile(ProfileRequest request,
            StreamObserver<UserProfile> responseObserver) {
        User user = userRepository.findById(request.getId())
                .orElseThrow(() -> 
                    Status.NOT_FOUND.asException()
                );
        
        UserProfile profile = UserProfile.newBuilder()
                .setUserId(user.getId())
                .setEmail(user.getEmail())
                .setRole(user.getRole())
                .build();
        
        responseObserver.onNext(profile);
        responseObserver.onCompleted();
    }
}

// 3. Custom SecurityExpression (위의 @userService.isOwner())
@Component("userService")
public class UserSecurityService {
    
    private final UserRepository userRepository;
    
    public boolean isOwner(String userId) {
        Authentication auth = 
            SecurityContextHolder.getContext()
                .getAuthentication();
        
        if (auth == null || !auth.isAuthenticated()) {
            return false;
        }
        
        String currentUser = auth.getName();
        User user = userRepository.findById(userId)
                .orElse(null);
        
        return user != null && 
               user.getUsername().equals(currentUser);
    }
}
```

### 4. 비동기 처리에서 SecurityContext 전파 방법

```java
// Context 키 정의
public class ContextKeys {
    static final Context.Key<SecurityContext> 
        SECURITY_CONTEXT_KEY = 
        Context.key("security-context", 
                   SecurityContext.class);
}

// Reactive 처리에서 SecurityContext 유지
@GrpcService
public class ReactiveStreamServiceImpl 
        extends ReactiveStreamServiceGrpc.
        ReactiveStreamServiceImplBase {
    
    private final OrderService orderService;
    
    @Override
    public void streamOrders(OrderFilter filter,
            StreamObserver<Order> responseObserver) {
        
        // 현재 Context 캡처
        Context grpcContext = Context.current();
        SecurityContext securityContext = 
            grpcContext.get(SECURITY_CONTEXT_KEY);
        
        // Reactor 처리
        Flux.fromIterable(
                orderService.findOrders(filter)
            )
            .subscribeOn(Schedulers.boundedElastic())
            .contextWrite(
                reactor.util.context.Context.of(
                    "security", securityContext
                )
            )
            .subscribe(
                order -> {
                    // 컨텍스트 복원
                    if (securityContext != null) {
                        SecurityContextHolder.setContext(
                            securityContext
                        );
                    }
                    responseObserver.onNext(order);
                },
                error -> {
                    log.error("Stream error", error);
                    responseObserver.onError(error);
                },
                responseObserver::onCompleted
            );
    }
}
```

---

## 💻 실전 실험

```java
// JwtTokenProvider 구현
@Component
public class JwtTokenProvider {
    
    @Value("${jwt.secret}")
    private String jwtSecret;
    
    @Value("${jwt.expiration:86400000}")
    private long jwtExpirationMs;
    
    public String generateToken(Authentication authentication) {
        UserPrincipal userPrincipal = 
            (UserPrincipal) authentication.getPrincipal();
        
        return Jwts.builder()
            .setSubject(userPrincipal.getUsername())
            .claim("roles", 
                   getRolesAsString(authentication))
            .setIssuedAt(new Date())
            .setExpiration(
                new Date(System.currentTimeMillis() + 
                        jwtExpirationMs)
            )
            .signWith(SignatureAlgorithm.HS512, jwtSecret)
            .compact();
    }
    
    public Claims getClaimsFromToken(String token) {
        return Jwts.parser()
            .setSigningKey(jwtSecret)
            .parseClaimsJws(token)
            .getBody();
    }
    
    public Authentication getAuthentication(String token) {
        Claims claims = getClaimsFromToken(token);
        String username = claims.getSubject();
        String rolesString = (String) claims.get("roles");
        
        Collection<GrantedAuthority> authorities = 
            Arrays.stream(rolesString.split(","))
                .map(SimpleGrantedAuthority::new)
                .collect(Collectors.toList());
        
        return new UsernamePasswordAuthenticationToken(
            username, null, authorities
        );
    }
    
    private String getRolesAsString(
            Authentication authentication) {
        return authentication.getAuthorities()
            .stream()
            .map(GrantedAuthority::getAuthority)
            .collect(Collectors.joining(","));
    }
}

// Test: 권한 검사
@RunWith(SpringRunner.class)
@SpringBootTest
public class SecurityIntegrationTest {
    
    @Autowired
    private JwtTokenProvider jwtProvider;
    
    @Test
    public void testAdminAccess() {
        Authentication auth = 
            new UsernamePasswordAuthenticationToken(
                "admin", null,
                List.of(new SimpleGrantedAuthority(
                    "ROLE_ADMIN"
                ))
            );
        
        String token = jwtProvider.generateToken(auth);
        Claims claims = jwtProvider.getClaimsFromToken(token);
        
        assertEquals("admin", claims.getSubject());
        assertEquals("ROLE_ADMIN", 
                     claims.get("roles"));
    }
}
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────────┐
│  인증 방식별 오버헤드 비교            │
├────────────────────────────────────────┤
│                                         │
│  방식           오버헤드  보안성       │
│  ─────────────────────────────────── │
│  No Auth        최소      0%           │
│  Basic Auth     낮음      중간         │
│  JWT            낮음~중간 높음         │
│  mTLS           높음      매우 높음    │
│  OAuth2         높음      매우 높음    │
│                                         │
│  JWT 추천: 높은 처리량, 중간 보안     │
│  mTLS 추천: 낮은 처리량, 최고 보안   │
└────────────────────────────────────────┘

┌────────────────────────────────────────┐
│  메모리 영향                           │
├────────────────────────────────────────┤
│                                         │
│  기능               메모리 증가        │
│  ────────────────────────────────────  │
│  기본 설정          기준               │
│  + JWT 검증         +1~5MB             │
│  + SecurityContext  +1MB당 요청 수      │
│  + Cache (1000)     +10~20MB           │
│                                         │
│  권장: 요청당 1KB 정도 할당           │
└────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
JWT vs mTLS:
├─ JWT
│  ✅ 장점: 상태 비저장, 높은 처리량
│  ❌ 단점: 토큰 해킹 시 취약
├─ mTLS
│  ✅ 장점: 최고 보안, 상호 검증
│  ❌ 단점: 인증서 관리 복잡, CPU 증가
└─ 권장: API 게이트웨이는 JWT, 
         내부 서비스는 mTLS

SecurityContext 전파:
├─ ThreadLocal (기본)
│  ✅ 장점: 구현 간단
│  ❌ 단점: 비동기 처리 복잡
├─ gRPC Context
│  ✅ 장점: 비동기 안전
│  ❌ 단점: 코드 복잡도 증가
└─ 권장: 비동기 처리 있으면 
         gRPC Context 사용
```

---

## 📌 핵심 정리

```
1. JWT → SecurityContext 설정
   ServerInterceptor에서 토큰 파싱
   → Authentication 객체 생성
   → SecurityContextHolder에 설정

2. @PreAuthorize 적용
   @EnableGlobalMethodSecurity 필요
   @GrpcService 메서드에 적용 가능

3. 예외 처리
   예외 → Status로 변환
   UNAUTHENTICATED, PERMISSION_DENIED 등

4. 비동기 처리
   SecurityContext 캡처 후 비동기 스레드에서 복원
   또는 gRPC Context 사용

5. 성능
   JWT: 낮은 오버헤드, 높은 처리량
   mTLS: 높은 보안, 높은 오버헤드
```

---

## 🤔 생각해볼 문제

### Q1: SecurityContext를 설정하지 않으면 @PreAuthorize는 어떻게 동작하는가?

<details>
<summary>해설 보기</summary>

**정답: @PreAuthorize는 작동하지 않음 (또는 PERMISSION_DENIED)**

```java
// SecurityContext 없음
SecurityContextHolder.getContext().getAuthentication() 
// → null

@PreAuthorize("hasRole('ADMIN')")
// → getAuthentication()이 null
// → hasRole('ADMIN')은 false 평가
// → PERMISSION_DENIED 예외 발생
```

결과: `Status.PERMISSION_DENIED` 반환

**해결책:** ServerInterceptor에서 JWT 파싱 후 SecurityContext 설정

</details>

---

### Q2: 인증 실패를 RuntimeException으로 throw하면?

<details>
<summary>해설 보기</summary>

**정답: gRPC가 INTERNAL(500)으로 처리**

```java
throw new RuntimeException("Invalid token");
// ↓
gRPC가 처리 안 된 예외로 봄
// ↓
Status.INTERNAL (상태 코드 13)로 변환
// ↓
클라이언트가 서버 에러로 오인
```

**올바른 방법:**

```java
throw Status.UNAUTHENTICATED
    .withDescription("Invalid token")
    .asException();
// ↓
Status.UNAUTHENTICATED (상태 코드 16)으로 처리
// ↓
클라이언트가 인증 실패로 올바르게 인식
```

</details>

---

### Q3: 비동기 처리에서 SecurityContext 전파 없으면?

<details>
<summary>해설 보기</summary>

**정답: 비동기 스레드에서 NullPointerException 발생**

```
주 스레드:
├─ SecurityContext 설정 ✅
└─ Flux.subscribeOn(boundedElastic()) ↓

boundedElastic() 스레드:
├─ SecurityContextHolder.getContext() 
│  → null (다른 스레드)
├─ auth.getName() 호출
└─ NullPointerException ❌
```

**해결책: SecurityContext 캡처 후 복원**

```java
SecurityContext context = 
    SecurityContextHolder.getContext();

Flux.fromIterable(orders)
    .subscribeOn(Schedulers.boundedElastic())
    .subscribe(order -> {
        SecurityContextHolder.setContext(context);
        // 비즈니스 로직
        SecurityContextHolder.clearContext();
    });
```

</details>

---

**[⬅️ 이전: grpc-spring-boot-starter 설정](./01-grpc-spring-boot-starter.md)** | **[홈으로 🏠](../README.md)** | **[다음: 예외 처리 통합 ➡️](./03-exception-handling.md)**
