# 에러 처리 — gRPC Status Code와 google.rpc.Status

---

## 🎯 핵심 질문

- gRPC Status Code 13가지는 각각 언제 쓸까?
- HTTP 상태 코드와의 매핑은?
- google.rpc.Status의 details 필드는?
- 에러 정보를 어디까지 클라이언트에 노출할까?
- 재시도 가능 에러는 어떻게 표현할까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC는 13가지 표준 Status Code로 모든 언어에서 동일한 에러 처리를 가능하게 합니다. HTTP처럼 자의적인 상태 코드를 남발하면 클라이언트는 재시도 정책을 결정할 수 없고, 에러 메시지를 공개하면 보안 취약점이 됩니다.

---

## 😱 흔한 실수 (Before — 표준 무시하는 에러 처리)

```
// 실수 1: HTTP 상태 코드 혼용
@GrpcService
public class UserServiceImpl {
  @Override
  public void getUser(GetUserRequest req, StreamObserver<User> res) {
    if (req.getUserId().isEmpty()) {
      res.onError(new StatusRuntimeException(
        Status.INVALID_ARGUMENT
        .withDescription("HTTP 400 Bad Request")  // HTTP처럼?
      ));
    }
  }
}

// 문제: gRPC는 HTTP 상태 코드를 모름
// Status.INVALID_ARGUMENT는 gRPC 표준

// 실수 2: 에러 메시지에 보안정보 노출
res.onError(new StatusRuntimeException(
  Status.INTERNAL
  .withDescription("Database connection failed: " +
    "user=db_admin, host=prod-db-01.internal, " +
    "port=5432, password=xxxxx")  // 노출 금지!
));

// 문제: 에러 메시지가 클라이언트에 전달
// 민감 정보 유출, 보안 침해

// 실수 3: 재시도 정보 없음
res.onError(new StatusRuntimeException(
  Status.UNAVAILABLE
  .withDescription("Service temporarily down")
  // 재시도 후 몇 ms에 가능할까? (클라이언트 모름)
));

// 실수 4: 세부 에러 정보 복잡하게 표현
message ErrorResponse {
  string error_code = 1;        // "ERR_001"?
  string message = 2;
  string debug_info = 3;
  repeated string suggestions = 4;
  // 형식 제각각
}
```

---

## ✨ 올바른 접근 (After — 표준 Status Code와 ErrorDetails)

```
// 올바른 접근 1: gRPC Status Code 사용
@GrpcService
public class UserServiceImpl {
  @Override
  public void getUser(GetUserRequest req, StreamObserver<User> res) {
    // 유효성 검사
    if (req.getUserId().isEmpty()) {
      res.onError(new StatusRuntimeException(
        Status.INVALID_ARGUMENT
        .withDescription("user_id is required")
      ));
      return;
    }
    
    try {
      User user = userRepository.findById(req.getUserId());
      res.onNext(user);
      res.onCompleted();
    } catch (UserNotFoundException e) {
      res.onError(new StatusRuntimeException(
        Status.NOT_FOUND
        .withDescription("User not found")
      ));
    } catch (DatabaseException e) {
      res.onError(new StatusRuntimeException(
        Status.UNAVAILABLE
        .withDescription("Database temporarily unavailable")
        // 재시도 정보 추가
        .withCause(e)
      ));
    }
  }
}

// 올바른 접근 2: ErrorDetails로 구조화된 정보
import com.google.rpc.Status;
import com.google.rpc.BadRequest;

public void getUser(GetUserRequest req, StreamObserver<User> res) {
  if (req.getUserId().isEmpty()) {
    // 구조화된 에러 정보
    BadRequest.FieldViolation violation = 
      BadRequest.FieldViolation.newBuilder()
        .setField("user_id")
        .setDescription("must not be empty")
        .build();
    
    BadRequest badRequest = BadRequest.newBuilder()
      .addFieldViolations(violation)
      .build();
    
    Status status = Status.newBuilder()
      .setCode(Code.INVALID_ARGUMENT_VALUE)
      .setMessage("Request validation failed")
      .addDetails(Any.pack(badRequest))
      .build();
    
    // 클라이언트가 구조화된 정보 파싱 가능
  }
}

// 올바른 접근 3: 재시도 정보 포함
import com.google.rpc.RetryInfo;

public void createOrder(CreateOrderRequest req, StreamObserver<Order> res) {
  try {
    Order order = orderService.create(req);
    res.onNext(order);
    res.onCompleted();
  } catch (RateLimitExceededException e) {
    RetryInfo retryInfo = RetryInfo.newBuilder()
      .setRetryDelay(Duration.newBuilder()
        .setSeconds(5)  // 5초 후 재시도
        .build())
      .build();
    
    Status status = Status.newBuilder()
      .setCode(Code.RESOURCE_EXHAUSTED_VALUE)
      .setMessage("Rate limit exceeded")
      .addDetails(Any.pack(retryInfo))
      .build();
  }
}

// 올바른 접근 4: 보안정보 숨기기
public void internalError(Exception e) {
  // 클라이언트에게
  Status clientStatus = Status.newBuilder()
    .setCode(Code.INTERNAL_VALUE)
    .setMessage("Internal server error")  // 구체적 정보 없음
    .build();
  
  // 로그에만
  logger.error("Database connection failed: " + e.getMessage(), e);
}
```

---

## 🔬 내부 동작 원리

### gRPC Status Code (13가지)

```
┌──────┬──────────────────────┬──────────────┬─────────────────────┐
│Code  │이름                  │HTTP 매핑     │언제 쓸까             │
├──────┼──────────────────────┼──────────────┼─────────────────────┤
│0     │OK                    │200          │성공 (보통 안 씀)    │
│1     │CANCELLED             │408/499      │클라이언트가 취소   │
│2     │UNKNOWN               │500          │알 수 없는 에러      │
│3     │INVALID_ARGUMENT      │400          │입력값 검증 실패     │
│4     │DEADLINE_EXCEEDED     │504          │타임아웃             │
│5     │NOT_FOUND             │404          │리소스 없음          │
│6     │ALREADY_EXISTS        │409          │중복된 리소스        │
│7     │PERMISSION_DENIED     │403          │권한 없음            │
│8     │RESOURCE_EXHAUSTED    │429          │할당량 초과          │
│9     │FAILED_PRECONDITION   │400          │조건 미충족          │
│10    │ABORTED               │409          │트랜잭션 중단        │
│11    │OUT_OF_RANGE          │400          │범위 초과            │
│12    │UNIMPLEMENTED         │501          │메서드 미구현        │
│13    │INTERNAL              │500          │서버 내부 에러       │
│14    │UNAVAILABLE           │503          │서비스 이용 불가     │
│15    │DATA_LOSS             │500          │데이터 손실          │
│16    │UNAUTHENTICATED       │401          │인증 없음            │
└──────┴──────────────────────┴──────────────┴─────────────────────┘

자주 쓰는 코드:
  INVALID_ARGUMENT: 입력값 문제 (필드 검증)
  NOT_FOUND: 리소스 없음
  PERMISSION_DENIED: 권한 부족
  UNAVAILABLE: 서비스 일시 불가 (재시도 가능)
  INTERNAL: 서버 에러 (재시도 불가)
  UNAUTHENTICATED: 토큰 없음/만료

클라이언트 재시도 정책:
  CANCELLED: 재시도 X
  INVALID_ARGUMENT: 재시도 X (입력값 고정)
  DEADLINE_EXCEEDED: 재시도 가능 (exponential backoff)
  RESOURCE_EXHAUSTED: 재시도 가능 (더 오래 대기)
  INTERNAL: 재시도 X (서버 버그)
  UNAVAILABLE: 재시도 가능 (서비스 회복 대기)
```

### google.rpc.Status와 ErrorDetails

```
Proto 정의:
  message Status {
    int32 code = 1;                  // gRPC Status Code
    string message = 2;              // 에러 메시지
    repeated google.protobuf.Any details = 3;  // 상세 정보
  }

ErrorDetails 종류:
  ├─ google.rpc.BadRequest
  │   └─ FieldViolation (필드별 검증 에러)
  │
  ├─ google.rpc.PreconditionFailure
  │   └─ Violation (조건 미충족)
  │
  ├─ google.rpc.RetryInfo
  │   └─ retry_delay (재시도 대기 시간)
  │
  ├─ google.rpc.ResourceInfo
  │   └─ resource_type, resource_name (리소스 정보)
  │
  ├─ google.rpc.ErrorInfo
  │   └─ reason, domain (에러 종류)
  │
  └─ google.rpc.Help
      └─ links (도움말 링크)

사용 예:

1. 필드 검증 실패
  Status {
    code: INVALID_ARGUMENT
    message: "Request validation failed"
    details: [
      Any {
        type_url: "type.googleapis.com/google.rpc.BadRequest"
        value: <BadRequest bytes>
      }
    ]
  }
  
  BadRequest {
    field_violations: [
      { field: "email", description: "invalid format" },
      { field: "age", description: "must be >= 0" }
    ]
  }

2. 재시도 정보
  Status {
    code: UNAVAILABLE
    message: "Rate limit exceeded"
    details: [
      Any {
        type_url: "type.googleapis.com/google.rpc.RetryInfo"
        value: <RetryInfo bytes>
      }
    ]
  }
  
  RetryInfo {
    retry_delay: { seconds: 30 }
  }

3. 리소스 정보
  Status {
    code: NOT_FOUND
    message: "User not found"
    details: [
      Any {
        type_url: "type.googleapis.com/google.rpc.ResourceInfo"
        value: <ResourceInfo bytes>
      }
    ]
  }
  
  ResourceInfo {
    resource_type: "users"
    resource_name: "users/U123456"
  }

클라이언트 파싱:
  try {
    stub.getUser(request);
  } catch (StatusRuntimeException e) {
    Status status = e.getStatus();
    
    for (Any detail : status.getDetailsList()) {
      if (detail.is(BadRequest.class)) {
        BadRequest badRequest = detail.unpack(BadRequest.class);
        for (BadRequest.FieldViolation v : badRequest.getFieldViolationsList()) {
          System.out.println(v.getField() + ": " + v.getDescription());
        }
      } else if (detail.is(RetryInfo.class)) {
        RetryInfo retryInfo = detail.unpack(RetryInfo.class);
        long delayMs = retryInfo.getRetryDelay().getSeconds() * 1000;
        Thread.sleep(delayMs);  // 대기 후 재시도
      }
    }
  }
```

### HTTP 매핑 (gRPC-HTTP Bridge)

```
gRPC ← → HTTP/REST (gRPC-Gateway)

Status 코드 매핑:
  gRPC                → HTTP
  OK (0)              → 200 OK
  CANCELLED (1)       → 408 Request Timeout / 499 Client Closed
  UNKNOWN (2)         → 500 Internal Server Error
  INVALID_ARGUMENT (3)→ 400 Bad Request
  DEADLINE_EXCEEDED   → 504 Gateway Timeout
  NOT_FOUND (5)       → 404 Not Found
  ALREADY_EXISTS (6)  → 409 Conflict
  PERMISSION_DENIED   → 403 Forbidden
  RESOURCE_EXHAUSTED  → 429 Too Many Requests
  FAILED_PRECONDITION → 400 Bad Request
  ABORTED (10)        → 409 Conflict
  OUT_OF_RANGE (11)   → 400 Bad Request
  UNIMPLEMENTED (12)  → 501 Not Implemented
  INTERNAL (13)       → 500 Internal Server Error
  UNAVAILABLE (14)    → 503 Service Unavailable
  DATA_LOSS (15)      → 500 Internal Server Error
  UNAUTHENTICATED (16)→ 401 Unauthorized

예시:
  gRPC 클라이언트:
    Status.NOT_FOUND (5)
    
  REST 클라이언트 (gRPC-Gateway 거쳐):
    HTTP 404 Not Found
    {
      "code": 5,
      "message": "User not found",
      "details": [...]
    }
```

---

## 💻 실전 실험

```java
// 에러 처리 구현

@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
  
  @Override
  public void getUser(GetUserRequest request, 
                      StreamObserver<User> responseObserver) {
    try {
      // 입력 검증
      if (request.getUserId().isEmpty()) {
        throw new IllegalArgumentException("user_id is required");
      }
      
      // 비즈니스 로직
      User user = userRepository.findById(request.getUserId())
        .orElseThrow(() -> new EntityNotFoundException("User not found"));
      
      responseObserver.onNext(user);
      responseObserver.onCompleted();
      
    } catch (IllegalArgumentException e) {
      // 클라이언트 에러: 입력값 문제
      responseObserver.onError(
        Status.INVALID_ARGUMENT
          .withDescription(e.getMessage())
          .asException()
      );
    } catch (EntityNotFoundException e) {
      // 404
      responseObserver.onError(
        Status.NOT_FOUND
          .withDescription(e.getMessage())
          .asException()
      );
    } catch (DatabaseException e) {
      // 500: 서버 에러 (보안: 상세정보 숨기기)
      logger.error("Database error", e);
      responseObserver.onError(
        Status.INTERNAL
          .withDescription("Internal server error")
          .asException()
      );
    }
  }
}
```

---

## 📊 성능/비용 비교

```
표준 Status Code 사용:
  ✓ 모든 언어에서 동일 처리
  ✓ 클라이언트 재시도 정책 자동화
  ✓ 모니터링 알람 표준화
  
비표준 에러 코드:
  ✗ 언어별 매핑 필요 (유지보수)
  ✗ 클라이언트 재시도 정책 불명확
  ✗ 모니터링 복잡도
```

---

## ⚖️ 트레이드오프

```
✅ gRPC Status Code:
├─ 표준화된 13가지 코드
├─ 모든 언어 지원
└─ 자동 재시도 정책

❌ gRPC Status Code:
├─ HTTP와 다름 (개발자 혼동)
├─ 세부 정보 제한
└─ 커스텀 코드 불가능

ErrorDetails:
✅ 구조화된 에러 정보
✗ 복잡도 증가
```

---

## 📌 핵심 정리

```
13가지 Status Code:
  INVALID_ARGUMENT: 입력값 문제 (재시도 X)
  NOT_FOUND: 리소스 없음 (재시도 X)
  PERMISSION_DENIED: 권한 부족 (재시도 X)
  UNAVAILABLE: 서비스 불가 (재시도 O)
  INTERNAL: 서버 에러 (재시도 X)
  UNAUTHENTICATED: 인증 없음 (재시도 X)
  
ErrorDetails:
  BadRequest: 필드 검증
  RetryInfo: 재시도 정보
  ResourceInfo: 리소스 식별
  ErrorInfo: 에러 종류
  
보안:
  클라이언트: 기본 메시지만 (구체정보 X)
  로그: 전체 정보 (디버깅)
  
HTTP 매핑:
  INVALID_ARGUMENT → 400
  NOT_FOUND → 404
  PERMISSION_DENIED → 403
  UNAVAILABLE → 503
  INTERNAL → 500
```

---

## 🤔 생각해볼 문제

**Q1: 비즈니스 예외 (중복 주문) 는 뭘 쓸까?**
```
주문이 이미 존재: ALREADY_EXISTS?
주문이 데이터베이스에만 있음 (나중 조회 가능):
  FAILED_PRECONDITION?
  ABORTED?
```
<details>
<summary>해설 보기</summary>

ALREADY_EXISTS (409 Conflict)

상황:
```proto
rpc CreateOrder(CreateOrderRequest) returns (Order);
```

중복 주문 detect:
```java
if (orderRepository.existsByOrderId(request.getOrderId())) {
  responseObserver.onError(
    Status.ALREADY_EXISTS
      .withDescription("Order already exists")
      .asException()
  );
}
```

HTTP: 409 Conflict
의미: 동일 리소스 이미 존재
재시도: X (재전송해도 같은 결과)

</details>

**Q2: Rate Limit 응답은?**
<details>
<summary>해설 보기</summary>

RESOURCE_EXHAUSTED (429 Too Many Requests)

```java
if (rateLimiter.isExceeded()) {
  long retryAfterSeconds = 30;
  
  RetryInfo retryInfo = RetryInfo.newBuilder()
    .setRetryDelay(Duration.newBuilder()
      .setSeconds(retryAfterSeconds)
      .build())
    .build();
  
  Status status = Status.newBuilder()
    .setCode(Code.RESOURCE_EXHAUSTED_VALUE)
    .setMessage("Rate limit exceeded")
    .addDetails(Any.pack(retryInfo))
    .build();
}
```

클라이언트:
- 자동으로 30초 후 재시도
- exponential backoff 적용

</details>

---

<div align="center">

**[⬅️ 이전: Proto 설계 원칙](./01-proto-design-principles.md)** | **[홈으로 🏠](../README.md)** | **[다음: 메타데이터 ➡️](./03-metadata.md)**

</div>
