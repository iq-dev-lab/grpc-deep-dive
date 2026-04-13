# 예외 처리 통합 — Spring 예외를 gRPC Status로

---

## 🎯 핵심 질문

- @GrpcExceptionHandler와 GrpcExceptionAdvice는 어떻게 동작하는가?
- 비즈니스 예외를 gRPC Status Code로 매핑하는 전략은?
- google.rpc.ErrorDetails를 응답에 포함하는 방법은?
- Bean Validation의 ConstraintViolationException을 INVALID_ARGUMENT로 변환하는 방법은?
- 예외가 처리되지 않으면 gRPC는 어떤 Status를 반환하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC는 HTTP/2 기반이므로 일반적인 HTTP 상태 코드(400, 500)를 사용할 수 없습니다. 대신 gRPC Status Code를 사용해야 하는데, 예외를 올바른 상태 코드로 변환하지 않으면 클라이언트가 에러의 원인을 알 수 없습니다. 또한 google.rpc.ErrorDetails를 활용하면 구조화된 에러 정보를 클라이언트에 전달할 수 있습니다.

---

## 😱 흔한 실수 (Before)

```java
// 실수 1: 예외를 그냥 throw
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    @Override
    public void getUser(GetUserRequest request,
            StreamObserver<GetUserResponse> responseObserver) {
        User user = userRepository.findById(request.getId())
                .orElseThrow(() -> new RuntimeException("Not found"));
        // ❌ RuntimeException → gRPC INTERNAL(500)로 처리
    }
}

// 실수 2: 에러 메시지에 스택 트레이스 포함
@GrpcExceptionHandler
public StatusRuntimeException handleException(Exception e) {
    return Status.INTERNAL
        .withDescription(e.getStackTrace().toString())  
        // ❌ 내부 정보 노출, 메시지 너무 김
        .asException();
}

// 실수 3: 검증 예외 처리 안 함
@GrpcService
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {
    @Override
    public void createOrder(CreateOrderRequest request,
            StreamObserver<CreateOrderResponse> responseObserver) {
        if (request.getAmount() <= 0) {
            throw new IllegalArgumentException("Invalid amount");
            // ❌ UNKNOWN(2) 상태로 처리
        }
    }
}

// 실수 4: ErrorDetails 사용하지 않고 문자열만 사용
throw Status.INVALID_ARGUMENT
    .withDescription("Validation failed: amount must be > 0")  
    // ❌ 클라이언트가 파싱 어려움
    .asException();
```

---

## ✨ 올바른 접근 (After)

```java
// 비즈니스 예외 정의
public class UserNotFoundException extends RuntimeException {
    private final String userId;
    
    public UserNotFoundException(String userId) {
        super("User not found: " + userId);
        this.userId = userId;
    }
}

public class InvalidOrderException extends RuntimeException {
    private final List<String> errors;
    
    public InvalidOrderException(List<String> errors) {
        super("Order validation failed");
        this.errors = errors;
    }
    
    public List<String> getErrors() {
        return errors;
    }
}

// 글로벌 예외 핸들러
@GrpcExceptionHandler
public class GlobalGrpcExceptionHandler {
    
    // 예외 1: 리소스를 찾을 수 없음
    @GrpcExceptionHandler
    public StatusRuntimeException handleUserNotFound(
            UserNotFoundException ex) {
        
        ErrorInfo errorInfo = ErrorInfo.newBuilder()
                .setReason("USER_NOT_FOUND")
                .setDomain("user-service")
                .putMetadata("user_id", 
                           ex.getUserId())
                .build();
        
        Status status = Status.NOT_FOUND
                .withDescription(ex.getMessage());
        
        return status
            .withCause(ex)
            .asException();
    }
    
    // 예외 2: 검증 실패
    @GrpcExceptionHandler
    public StatusRuntimeException handleInvalidOrder(
            InvalidOrderException ex) {
        
        BadRequest badRequest = BadRequest.newBuilder()
                .addAllFieldViolations(
                    ex.getErrors().stream()
                        .map(error -> 
                            BadRequest.FieldViolation.newBuilder()
                                .setField(error.split(":")[0])
                                .setDescription(
                                    error.split(":")[1]
                                )
                                .build()
                        )
                        .collect(Collectors.toList())
                )
                .build();
        
        return Status.INVALID_ARGUMENT
                .withDescription(ex.getMessage())
                .asException();
    }
    
    // 예외 3: 접근 거부
    @GrpcExceptionHandler
    public StatusRuntimeException handleAccessDenied(
            AccessDeniedException ex) {
        return Status.PERMISSION_DENIED
                .withDescription(ex.getMessage())
                .asException();
    }
    
    // 예외 4: 처리되지 않은 예외 (기본값)
    @GrpcExceptionHandler
    public StatusRuntimeException handleGenericException(
            Exception ex) {
        log.error("Unhandled exception", ex);
        
        return Status.INTERNAL
                .withDescription(
                    "An internal error occurred")
                // 스택 트레이스 미포함 (보안)
                .asException();
    }
}

// gRPC Service: 올바른 예외 처리
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    
    @Override
    public void getUser(GetUserRequest request,
            StreamObserver<GetUserResponse> responseObserver) {
        try {
            if (request.getId().isEmpty()) {
                throw new InvalidOrderException(
                    List.of("id:User ID is required")
                );
            }
            
            User user = userRepository.findById(request.getId())
                    .orElseThrow(() -> 
                        new UserNotFoundException(
                            request.getId()
                        )
                    );
            
            GetUserResponse response = GetUserResponse.newBuilder()
                    .setId(user.getId())
                    .setName(user.getName())
                    .build();
            
            responseObserver.onNext(response);
            responseObserver.onCompleted();
            
        } catch (Exception e) {
            // @GrpcExceptionHandler에 위임
            throw e;
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 예외 처리 체인 (발생부터 Status 반환까지)

```
┌──────────────────────────┐
│  gRPC 메서드 실행        │
│  try-catch              │
└────────────┬─────────────┘
             │
      ┌──────┴───────┐
      ▼              ▼
   ✅ 성공    ❌ 예외 발생
      │              │
      │              ▼
      │      ┌──────────────────────┐
      │      │ GrpcExceptionHandler │
      │      │ 스캔 및 매칭         │
      │      └──────┬───────────────┘
      │             │
      │      ┌──────┴──────┐
      │      ▼             ▼
      │   매칭  비매칭
      │   예외  예외
      │   처리  │
      │   │     ▼
      │   │  ┌────────────────┐
      │   │  │ 일반 예외로    │
      │   │  │ 처리           │
      │   │  │ INTERNAL(13)   │
      │   │  └────────┬───────┘
      │   │           │
      │   └───────┬───┘
      │           ▼
      └────────▶ ┌──────────────────┐
                 │ Status 생성      │
                 │ (Code + Message) │
                 └────────┬─────────┘
                          │
                          ▼
                 ┌──────────────────┐
                 │ gRPC 프레임      │
                 │ 클라이언트 반환  │
                 └──────────────────┘
```

### 2. GrpcExceptionAdvice 구현 (여러 예외 매핑)

```java
@GrpcExceptionHandler
@Slf4j
public class UserServiceExceptionHandler {
    
    // 매핑 테이블
    private static final Map<Class<?>, Status> 
        EXCEPTION_STATUS_MAP = Map.ofEntries(
        Map.entry(UserNotFoundException.class, 
                 Status.NOT_FOUND),
        Map.entry(InvalidOrderException.class, 
                 Status.INVALID_ARGUMENT),
        Map.entry(AccessDeniedException.class, 
                 Status.PERMISSION_DENIED),
        Map.entry(IllegalArgumentException.class, 
                 Status.INVALID_ARGUMENT),
        Map.entry(RuntimeException.class, 
                 Status.INTERNAL)
    );
    
    // 단일 매서드: 모든 예외 처리
    @GrpcExceptionHandler
    public StatusRuntimeException handle(Exception ex) {
        
        Status status = EXCEPTION_STATUS_MAP.getOrDefault(
            ex.getClass(),
            Status.INTERNAL
        );
        
        log.error("gRPC exception: {} - {}", 
                 status.getCode(), 
                 ex.getMessage());
        
        return status
            .withDescription(ex.getMessage())
            .asException();
    }
    
    // 또는 개별 처리
    @GrpcExceptionHandler(UserNotFoundException.class)
    public StatusRuntimeException handleUserNotFound(
            UserNotFoundException ex) {
        log.warn("User not found: {}", ex.getMessage());
        return Status.NOT_FOUND
            .withDescription(ex.getMessage())
            .asException();
    }
    
    @GrpcExceptionHandler(InvalidOrderException.class)
    public StatusRuntimeException handleInvalidOrder(
            InvalidOrderException ex) {
        log.warn("Order validation failed");
        return Status.INVALID_ARGUMENT
            .withDescription(ex.getMessage())
            .asException();
    }
    
    @GrpcExceptionHandler(
        {AccessDeniedException.class,
         AuthenticationException.class})
    public StatusRuntimeException handleAuthException(
            Exception ex) {
        log.warn("Auth error: {}", ex.getMessage());
        return Status.PERMISSION_DENIED
            .withDescription("Access denied")
            .asException();
    }
}
```

### 3. ErrorDetails 포함 응답 (구조화된 에러)

```java
// 1. BadRequest (검증 실패)
@GrpcExceptionHandler
public StatusRuntimeException handleValidationError(
        ConstraintViolationException ex) {
    
    BadRequest.Builder badRequest = 
        BadRequest.newBuilder();
    
    ex.getConstraintViolations().forEach(violation -> {
        badRequest.addFieldViolations(
            BadRequest.FieldViolation.newBuilder()
                .setField(violation.getPropertyPath()
                    .toString())
                .setDescription(violation.getMessage())
                .build()
        );
    });
    
    Status status = Status.INVALID_ARGUMENT
        .withDescription(
            "Validation failed: " + 
            ex.getConstraintViolations().size() + 
            " error(s)");
    
    return status.asException();
}

// 2. ResourceInfo (리소스 정보)
@GrpcExceptionHandler
public StatusRuntimeException handleResourceNotFound(
        UserNotFoundException ex) {
    
    ResourceInfo resourceInfo = ResourceInfo.newBuilder()
        .setResourceType("User")
        .setResourceName("users/" + ex.getUserId())
        .setDescription("The requested user was not found")
        .build();
    
    Status status = Status.NOT_FOUND
        .withDescription(ex.getMessage());
    
    return status.asException();
}

// 3. ErrorInfo (에러 분류)
@GrpcExceptionHandler
public StatusRuntimeException handleBusinessError(
        InvalidOrderException ex) {
    
    ErrorInfo errorInfo = ErrorInfo.newBuilder()
        .setReason("ORDER_VALIDATION_FAILED")
        .setDomain("order-service")
        .putMetadata("error_count", 
                    String.valueOf(ex.getErrors().size()))
        .putMetadata("timestamp", 
                    Instant.now().toString())
        .build();
    
    return Status.INVALID_ARGUMENT
        .withDescription(ex.getMessage())
        .asException();
}

// 4. RetryInfo (재시도 정보)
@GrpcExceptionHandler
public StatusRuntimeException handleTemporaryFailure(
        TemporaryServiceUnavailableException ex) {
    
    RetryInfo retryInfo = RetryInfo.newBuilder()
        .setRetryDelay(
            Duration.newBuilder()
                .setSeconds(5)
                .build()
        )
        .build();
    
    return Status.UNAVAILABLE
        .withDescription(
            "Service temporarily unavailable, " +
            "please retry in 5 seconds")
        .asException();
}
```

### 4. Bean Validation 통합

```java
@GrpcService
public class OrderServiceImpl 
        extends OrderServiceGrpc.OrderServiceImplBase {
    
    private final Validator validator;
    
    @Override
    public void createOrder(CreateOrderRequest request,
            StreamObserver<CreateOrderResponse> responseObserver) {
        
        try {
            // 1. 요청 검증
            Order order = Order.builder()
                .customerId(request.getCustomerId())
                .amount(request.getAmount())
                .items(request.getItemsList())
                .build();
            
            // 2. Bean Validation 실행
            Set<ConstraintViolation<Order>> violations = 
                validator.validate(order);
            
            if (!violations.isEmpty()) {
                throw new ConstraintViolationException(
                    "Validation failed", violations);
            }
            
            // 3. 비즈니스 로직
            Order saved = orderRepository.save(order);
            
            CreateOrderResponse response = 
                CreateOrderResponse.newBuilder()
                    .setOrderId(saved.getId())
                    .build();
            
            responseObserver.onNext(response);
            responseObserver.onCompleted();
            
        } catch (ConstraintViolationException e) {
            // 3. @GrpcExceptionHandler에 위임
            throw e;
        }
    }
}

// Order 모델: 검증 규칙 정의
@Data
@Builder
public class Order {
    
    @NotBlank(message = "Customer ID is required")
    private String customerId;
    
    @Positive(message = "Amount must be greater than 0")
    private BigDecimal amount;
    
    @NotEmpty(message = "Order must contain at least one item")
    private List<OrderItem> items;
}

// 예외 핸들러: ConstraintViolationException 처리
@GrpcExceptionHandler
public StatusRuntimeException handleConstraintViolation(
        ConstraintViolationException ex) {
    
    BadRequest.Builder builder = BadRequest.newBuilder();
    
    ex.getConstraintViolations()
        .forEach(violation -> {
            builder.addFieldViolations(
                BadRequest.FieldViolation.newBuilder()
                    .setField(
                        violation.getPropertyPath()
                            .toString()
                    )
                    .setDescription(
                        violation.getMessage()
                    )
                    .build()
            );
        });
    
    return Status.INVALID_ARGUMENT
        .withDescription(
            "Validation failed: " + 
            ex.getConstraintViolations().size() + 
            " error(s)")
        .asException();
}
```

---

## 💻 실전 실험

```bash
# Test: 예외 처리 확인
grpcurl -plaintext \
  -d '{"id": ""}' \
  localhost:9090 \
  io.grpc.examples.UserService/GetUser

# 응답: Status NOT_FOUND
# Error details: User ID is required

# Test: 검증 실패
grpcurl -plaintext \
  -d '{"amount": -100}' \
  localhost:9090 \
  io.grpc.examples.OrderService/CreateOrder

# 응답: Status INVALID_ARGUMENT
# Field violations: amount must be greater than 0
```

---

## 📊 성능/비용 비교

```
┌──────────────────────────────────┐
│ 예외 처리 오버헤드 비교         │
├──────────────────────────────────┤
│                                   │
│ 방식          오버헤드  명확성   │
│ ──────────────────────────────── │
│ No handling   0ms       낮음     │
│ Basic Status  <1ms      중간     │
│ ErrorDetails  1~2ms     높음     │
│ Log + Detail  2~5ms     매우높음 │
│                                   │
│ 권장: ErrorDetails 추가는 최소  │
└──────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
예외 메시지 상세도:
├─ 간단: "Error occurred"
│  ✅ 빠름, 보안
│  ❌ 디버깅 어려움
├─ 상세: 모든 정보 포함
│  ✅ 디버깅 쉬움
│  ❌ 느림, 보안 위험
└─ 권선: 프로덕션은 간단, 
         개발은 상세
```

---

## 📌 핵심 정리

```
1. 예외 → Status 매핑
   Status.NOT_FOUND (404)
   Status.INVALID_ARGUMENT (400)
   Status.PERMISSION_DENIED (403)
   Status.INTERNAL (500)

2. @GrpcExceptionHandler 사용
   클래스 또는 메서드 수준 적용

3. ErrorDetails 활용
   BadRequest, ResourceInfo, ErrorInfo 등

4. 보안: 스택 트레이스 미포함
   내부 정보 노출 방지
```

---

## 🤔 생각해볼 문제

### Q1: 예외를 처리하지 않으면 gRPC는 어떤 Status를 반환하는가?

<details>
<summary>해설 보기</summary>

**정답: Status.INTERNAL (코드 13)**

처리되지 않은 예외 → gRPC 런타임 → INTERNAL

```
예외 발생
  ↓
@GrpcExceptionHandler 매칭 없음
  ↓
gRPC 프레임워크가 처리
  ↓
Status.INTERNAL로 변환
  ↓
클라이언트가 서버 에러로 인식
```

**올바른 방법:** 모든 예외를 @GrpcExceptionHandler로 처리

</details>

---

### Q2: ConstraintViolationException을 INVALID_ARGUMENT로 변환하려면?

<details>
<summary>해설 보기</summary>

**정답: @GrpcExceptionHandler로 매핑**

```java
@GrpcExceptionHandler
public StatusRuntimeException handle(
        ConstraintViolationException ex) {
    return Status.INVALID_ARGUMENT
        .withDescription("Validation failed")
        .asException();
}
```

BadRequest 상세 정보 포함:

```java
BadRequest badRequest = BadRequest.newBuilder()
    .addAllFieldViolations(
        ex.getConstraintViolations().stream()
            .map(v -> BadRequest.FieldViolation
                .newBuilder()
                .setField(v.getPropertyPath().toString())
                .setDescription(v.getMessage())
                .build())
            .collect(Collectors.toList())
    )
    .build();

return Status.INVALID_ARGUMENT
    .withDescription("Validation failed")
    .asException();
```

</details>

---

### Q3: 에러 메시지에 스택 트레이스를 포함하면 안 되는 이유는?

<details>
<summary>해설 보기</summary>

**정답: 보안 위험 + 성능 저하**

**1. 보안 위험:**
```
스택 트레이스 예시:
at com.example.UserService.getUser(UserService.java:123)
at com.example.OrderService.process(OrderService.java:456)
→ 클래스명, 파일명, 라인 번호 노출
→ 공격자가 시스템 구조 파악 가능
```

**2. 성능 저하:**
```
StackTraceElement[] elements = 
    new Exception().getStackTrace();  // 비용 높음
String trace = Arrays.toString(elements);  // 메모리 증가
// 메시지 크기 증가 → 네트워크 전송 시간 증가
```

**올바른 방법:**

```java
// ❌ 나쁜 예
return Status.INTERNAL
    .withDescription(ex.getStackTrace().toString())
    .asException();

// ✅ 좋은 예
return Status.INTERNAL
    .withDescription("An internal error occurred")
    .asException();

// 로그에만 스택 트레이스 기록
log.error("User processing failed", ex);
```

</details>

---

**[⬅️ 이전: Spring Security 통합](./02-spring-security.md)** | **[홈으로 🏠](../README.md)** | **[다음: gRPC + Spring WebFlux ➡️](./04-grpc-webflux.md)**
