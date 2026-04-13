# gRPC Deadline과 Timeout: 분산 시스템의 시간 제어

## 🎯 핵심질문

마이크로서비스 아키텍처에서 A → B → C로 이어지는 RPC 체인이 있을 때, 원래 클라이언트가 설정한 2초 deadline이 각 서비스에서 어떻게 자동으로 전파되고, 남은 시간이 정확하게 계산되며, 타임아웃 발생 시 어떤 메커니즘으로 요청이 중단될까?

## 🔍 왜 중요한가

분산 시스템에서 deadline 관리는 단순한 편의 기능이 아닙니다. 실제 프로덕션 환경에서 다음과 같은 시나리오를 마주합니다:

1. **Cascading Timeout 방지**: 한 서비스의 느린 응답이 전체 시스템을 마비시키지 않도록 해야 합니다. deadline이 없으면 B 서비스가 1초 걸릴 때, C 서비스는 여전히 1초를 기다리게 되어 총 2초 이상 소요됩니다.

2. **자동 Context 전파**: Java의 ThreadLocal처럼 gRPC는 deadline을 자동으로 모든 downstream 호출에 전파합니다. 개발자가 각 서비스마다 수동으로 설정할 필요가 없습니다.

3. **Tail Latency 제어**: 금융거래, 결제 시스템에서 1%의 요청이 30초 이상 걸리면 사용자 경험이 저하됩니다. deadline을 통해 이런 tail latency를 제어합니다.

4. **Resource Cleanup**: deadline이 만료되면 gRPC 프레임워크가 자동으로 RST_STREAM 프레임을 전송하여 즉시 리소스를 반환합니다. 없으면 완료될 때까지 스레드와 메모리를 점유합니다.

5. **Observability**: deadline 초과로 인한 실패는 `DEADLINE_EXCEEDED` 상태 코드로 명확하게 추적할 수 있어, 모니터링과 알림이 용이합니다.

## 😱 흔한 실수

### 실수 1: Deadline 없이 무한 대기

```java
// ❌ 잘못된 접근: deadline 설정이 없음
@RestController
public class OrderController {
    private final OrderServiceGrpc.OrderServiceBlockingStub stub;
    
    @PostMapping("/orders")
    public ResponseEntity<String> createOrder(@RequestBody OrderRequest req) {
        try {
            // 타임아웃이 없으므로 downstream 서비스가 응답하지 않으면 무한 대기
            OrderResponse response = stub.createOrder(
                Order.newBuilder()
                    .setCustomerId(req.getCustomerId())
                    .setAmount(req.getAmount())
                    .build()
            );
            return ResponseEntity.ok(response.getStatus());
        } catch (StatusRuntimeException e) {
            return ResponseEntity.status(500).body("Error: " + e.getMessage());
        }
    }
}

// 문제점:
// 1. 60초 이상 대기 가능 (default deadline 또는 무제한)
// 2. 클라이언트 타임아웃(30초)과 서버 deadline이 일치하지 않음
// 3. Cascading timeout: B가 50초 걸리면 C는 0초만 남음 (실패율 증가)
// 4. 스레드 풀이 가득 차서 새 요청 불가능
```

### 실수 2: 고정된 Deadline 사용

```java
// ❌ 잘못된 접근: 모든 요청에 동일한 deadline
public class PaymentServiceClient {
    private final ManagedChannel channel;
    
    public void processPayment() {
        PaymentServiceGrpc.PaymentServiceBlockingStub stub = 
            PaymentServiceGrpc.newBlockingStub(channel);
        
        // 5초는 네트워크 상태에 따라 너무 짧거나 길 수 있음
        stub = stub.withDeadlineAfter(5, TimeUnit.SECONDS);
        
        try {
            PaymentResponse response = stub.pay(PaymentRequest.getDefaultInstance());
        } catch (StatusRuntimeException e) {
            // DEADLINE_EXCEEDED인지 다른 이유인지 확인 필요
            if (e.getStatus() == Status.DEADLINE_EXCEEDED) {
                System.out.println("Timeout 발생");
            }
        }
    }
}

// 문제점:
// 1. 네트워크 지연, 부하 증가 시 쉽게 타임아웃
// 2. API별 특성(DB 쿼리 복잡도)을 고려하지 않음
// 3. 의도적 deadline 초과 테스트 불가능
// 4. 느린 네트워크 환경에서 사용 불가능
```

### 실수 3: Deadline Propagation 무시

```java
// ❌ 잘못된 접근: 새로운 stub 생성으로 deadline이 끊김
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    
    @Override
    public void getUser(GetUserRequest req, StreamObserver<User> responseObserver) {
        try {
            // 새로운 Channel에서 stub을 생성하면 deadline이 전파되지 않음
            ManagedChannel newChannel = ManagedChannelBuilder
                .forAddress("order-service", 50051)
                .usePlaintext()
                .build();
            
            OrderServiceGrpc.OrderServiceBlockingStub newStub = 
                OrderServiceGrpc.newBlockingStub(newChannel);
            
            // 여기서는 deadline이 0이므로 즉시 타임아웃 가능
            GetOrdersResponse orders = newStub.getOrders(
                GetOrdersRequest.newBuilder().setUserId(req.getId()).build()
            );
        } catch (StatusRuntimeException e) {
            // deadline이 전파되지 않아 예측 불가능한 타임아웃 발생
        }
    }
}

// 문제점:
// 1. deadline 체인이 끊김 (A→B: 5초, B→C: 무제한)
// 2. C 서비스가 오래 걸려도 B의 deadline 초과까지 기다림
// 3. 불필요한 리소스 낭비 및 메모리 누수
```

## ✨ 올바른 접근

### 올바른 구현 1: Client-side Deadline 설정

```java
// ✅ 올바른 접근: 명시적 deadline과 error handling
@RestController
public class OrderController {
    private final OrderServiceGrpc.OrderServiceBlockingStub orderStub;
    
    @PostMapping("/orders")
    public ResponseEntity<OrderResponse> createOrder(@RequestBody OrderRequest req) {
        try {
            // deadline을 명시적으로 설정 (상대 시간)
            OrderServiceGrpc.OrderServiceBlockingStub stubWithDeadline = 
                orderStub.withDeadlineAfter(3, TimeUnit.SECONDS);
            
            OrderResponse response = stubWithDeadline.createOrder(
                Order.newBuilder()
                    .setCustomerId(req.getCustomerId())
                    .setAmount(req.getAmount())
                    .build()
            );
            
            return ResponseEntity.ok(response);
            
        } catch (StatusRuntimeException e) {
            // deadline 초과 명확하게 처리
            if (e.getStatus() == Status.DEADLINE_EXCEEDED) {
                return ResponseEntity
                    .status(HttpStatus.GATEWAY_TIMEOUT)
                    .body(null);
            }
            return ResponseEntity
                .status(HttpStatus.INTERNAL_SERVER_ERROR)
                .body(null);
        }
    }
}
```

### 올바른 구현 2: Deadline Propagation이 적용된 체인

```java
// ✅ 올바른 접근: Deadline이 자동으로 B→C로 전파됨
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    
    private final OrderServiceGrpc.OrderServiceBlockingStub orderStub;
    private static final Logger logger = LoggerFactory.getLogger(UserServiceImpl.class);
    
    public UserServiceImpl(OrderServiceGrpc.OrderServiceBlockingStub orderStub) {
        this.orderStub = orderStub;
    }
    
    @Override
    public void getUser(GetUserRequest req, StreamObserver<User> responseObserver) {
        try {
            // Context.current()에 deadline이 자동으로 포함됨
            // deadline이 없으면 새로 생성, 있으면 그것을 사용
            
            Deadline currentDeadline = Context.current().getDeadline();
            logger.info("Current deadline: {}", currentDeadline);
            
            // 이 stub 호출은 자동으로 upstream deadline을 상속받음
            GetOrdersResponse orders = orderStub.getOrders(
                GetOrdersRequest.newBuilder()
                    .setUserId(req.getId())
                    .build()
            );
            
            User user = User.newBuilder()
                .setId(req.getId())
                .setName("John Doe")
                .setOrderCount(orders.getOrdersList().size())
                .build();
                
            responseObserver.onNext(user);
            responseObserver.onCompleted();
            
        } catch (StatusRuntimeException e) {
            if (e.getStatus() == Status.DEADLINE_EXCEEDED) {
                logger.warn("Deadline exceeded for user {}", req.getId());
            }
            responseObserver.onError(e);
        }
    }
}
```

### 올바른 구현 3: Server-side Deadline 검사

```java
// ✅ 올바른 접근: 서버가 deadline을 주기적으로 확인
@GrpcService
public class OrderServiceImpl extends OrderServiceGrpc.OrderServiceImplBase {
    
    private static final Logger logger = LoggerFactory.getLogger(OrderServiceImpl.class);
    
    @Override
    public void createOrder(Order req, StreamObserver<OrderResponse> responseObserver) {
        try {
            // 1. 현재 deadline 확인
            Deadline deadline = Context.current().getDeadline();
            logger.info("Create order deadline: {}", deadline);
            
            // 2. 작업 시작 전 deadline 확인
            if (deadline != null && deadline.isExpired()) {
                responseObserver.onError(
                    Status.DEADLINE_EXCEEDED
                        .withDescription("Deadline already exceeded at service entry")
                        .asException()
                );
                return;
            }
            
            // 3. 데이터베이스 조회 (100ms)
            OrderData orderData = queryDatabase(req.getCustomerId());
            
            // 4. 오래 걸리는 작업 도중 주기적으로 확인
            if (deadline != null && deadline.isExpired()) {
                responseObserver.onError(
                    Status.DEADLINE_EXCEEDED
                        .withDescription("Deadline exceeded during processing")
                        .asException()
                );
                return;
            }
            
            // 5. 결제 처리 (2초)
            PaymentResult paymentResult = processPayment(req.getAmount());
            
            // 6. 최종 확인 (마지막 체크)
            if (deadline != null && deadline.isExpired()) {
                responseObserver.onError(
                    Status.DEADLINE_EXCEEDED
                        .withDescription("Deadline exceeded before sending response")
                        .asException()
                );
                return;
            }
            
            OrderResponse response = OrderResponse.newBuilder()
                .setOrderId(UUID.randomUUID().toString())
                .setStatus("SUCCESS")
                .build();
                
            responseObserver.onNext(response);
            responseObserver.onCompleted();
            
        } catch (Exception e) {
            responseObserver.onError(
                Status.INTERNAL.withCause(e).asException()
            );
        }
    }
    
    private OrderData queryDatabase(String customerId) {
        // DB 조회 로직
        return new OrderData(customerId, 1000.0);
    }
    
    private PaymentResult processPayment(double amount) {
        // 결제 처리 로직
        return new PaymentResult(true, "Payment processed");
    }
}

class OrderData {
    String customerId;
    double maxCredit;
    
    OrderData(String customerId, double maxCredit) {
        this.customerId = customerId;
        this.maxCredit = maxCredit;
    }
}

class PaymentResult {
    boolean success;
    String message;
    
    PaymentResult(boolean success, String message) {
        this.success = success;
        this.message = message;
    }
}
```

### 올바른 구현 4: Spring Boot Configuration

```java
// ✅ 올바른 접근: application.yaml에서 deadline 설정
// application.yaml 또는 application.properties
/*
grpc:
  client:
    payment-service:
      address: localhost:50051
      deadline: 5s
    order-service:
      address: localhost:50052
      deadline: 3s
    user-service:
      address: localhost:50053
      deadline: 2s
*/

@Configuration
public class GrpcClientConfiguration {
    
    @Bean
    public OrderServiceGrpc.OrderServiceBlockingStub orderServiceStub(
            @Value("${grpc.client.order-service.deadline:5s}") String deadline,
            @Value("${grpc.client.order-service.address:localhost:50051}") String address) {
        
        ManagedChannel channel = ManagedChannelBuilder
            .forTarget(address)
            .usePlaintext()
            .build();
        
        OrderServiceGrpc.OrderServiceBlockingStub stub = 
            OrderServiceGrpc.newBlockingStub(channel);
        
        // 설정된 deadline을 적용
        Duration duration = Duration.parse("PT" + deadline.replace("s", "S"));
        stub = stub.withDeadlineAfter(duration.getSeconds(), TimeUnit.SECONDS);
        
        return stub;
    }
}
```

## 🔬 내부 동작 원리

### Deadline Propagation 메커니즘

```
┌──────────────────────────────────────────────────────────────────┐
│ Client (API Gateway)                                              │
│ withDeadlineAfter(5, SECONDS) → Deadline = now + 5초              │
│                                                                   │
│  ┌─ RPC Call ────────────────────────────────────────────────┐   │
│  │ gRPC Header: grpc-timeout=5000m                           │   │
│  │ (5000 milliseconds)                                        │   │
│  │                                                            │   │
│  │  [Request 시작 시간: T0]                                   │   │
│  └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
                            ↓
                    [네트워크: 100ms]
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│ Service B (User Service)                    [경과: 100ms]         │
│ Context.current().getDeadline() → 4900ms 남음                    │
│                                                                   │
│  ┌─ Downstream RPC ──────────────────────────────────────────┐   │
│  │ grpc-timeout=4900m (자동으로 감소됨)                        │   │
│  │                                                            │   │
│  │  [요청 시작 시간: T0 + 100ms]                              │   │
│  └────────────────────────────────────────────────────────────┘   │
└──────────────────────────────────────────────────────────────────┘
                            ↓
                    [네트워크: 50ms]
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│ Service C (Order Service)                  [경과: 150ms]         │
│ Context.current().getDeadline() → 4850ms 남음                    │
│                                                                   │
│  ┌─ DB Query (200ms) ───────────────────────────────────────┐    │
│  │ deadline 확인: 4850ms > 200ms → OK                        │    │
│  │                                                           │    │
│  │  SELECT * FROM orders ... (2ms)                          │    │
│  │  isExpired() → false                                     │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                   │
│  ┌─ Slow Operation (3000ms) ─────────────────────────────────┐    │
│  │ deadline 확인: 4650ms > 3000ms → OK                        │   │
│  │                                                           │    │
│  │  for i in 1..1000:                                       │    │
│  │    if (deadline.isExpired()) {                           │    │
│  │      onError(DEADLINE_EXCEEDED)                          │    │
│  │      return                                              │    │
│  │    }                                                     │    │
│  │    // 작업 계속                                           │    │
│  └───────────────────────────────────────────────────────────┘    │
│                                                                   │
│  Response: 200 바이트                                             │
└──────────────────────────────────────────────────────────────────┘
                            ↓
                    [네트워크: 50ms]
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│ Service B (계속)                          [경과: 400ms]          │
│ Order Service에서 받은 Response를 처리                            │
│ Context.current().getDeadline() → 4600ms 남음                    │
│                                                                   │
│  Response를 조합하여 위로 전달                                    │
└──────────────────────────────────────────────────────────────────┘
                            ↓
                    [네트워크: 50ms]
                            ↓
┌──────────────────────────────────────────────────────────────────┐
│ Client (완료)                              [경과: 450ms]         │
│ Total elapsed: 450ms (deadline 5000ms 내에 완료)                  │
│                                                                   │
│ ✅ 성공                                                          │
└──────────────────────────────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Deadline Timeout 시나리오 (Service C가 느린 경우):

┌──────────────────────────────────────────────────────────────────┐
│ Client                                                            │
│ withDeadlineAfter(2, SECONDS) → Deadline = now + 2초              │
└──────────────────────────────────────────────────────────────────┘
                            ↓ (100ms)
┌──────────────────────────────────────────────────────────────────┐
│ Service B                           [경과: 100ms]                │
│ Deadline 남음: 1900ms                                            │
│                                                                   │
│  grpc-timeout=1900m 으로 Service C에 요청                        │
└──────────────────────────────────────────────────────────────────┘
                            ↓ (50ms)
┌──────────────────────────────────────────────────────────────────┐
│ Service C                           [경과: 150ms]                │
│ Deadline 남음: 1850ms                                            │
│ Slow operation 시작 (3000ms 예상)                                │
│                                                                   │
│  @1400ms: isExpired() → true!                                    │
│  ❌ DEADLINE_EXCEEDED 에러 발생                                  │
│                                                                   │
│  RST_STREAM 프레임 전송 (리소스 즉시 해제)                        │
└──────────────────────────────────────────────────────────────────┘
                            ↓ (에러)
┌──────────────────────────────────────────────────────────────────┐
│ Service B                           [경과: 1850ms]               │
│ StatusRuntimeException 캐치                                       │
│ Status.DEADLINE_EXCEEDED 처리                                    │
│                                                                   │
│ 남은 시간이 거의 없으므로 다른 재시도 불가능                      │
└──────────────────────────────────────────────────────────────────┘
                            ↓ (에러)
┌──────────────────────────────────────────────────────────────────┐
│ Client                              [경과: 2000ms]               │
│ ❌ StatusRuntimeException 수신                                   │
│ Status: DEADLINE_EXCEEDED                                        │
│                                                                   │
│ 클라이언트는 즉시 요청 중단 및 에러 처리                          │
└──────────────────────────────────────────────────────────────────┘
```

### Context 내부 구조

```java
// gRPC Context는 ThreadLocal처럼 동작
// 각 RPC 핸들러마다 새로운 Context가 생성됨

// Context.current()는 현재 스레드의 Context를 반환
Context context = Context.current();

// deadline은 Context에 저장된 속성
Deadline deadline = context.getDeadline();

// deadline이 있으면, 이 정보는 모든 downstream gRPC 호출에 자동 포함됨
// grpc-timeout HTTP/2 헤더로 변환되어 전송됨

// deadline.isExpired()는 내부적으로 System.nanoTime()과 비교
// deadline time <= current time → true (만료됨)
```

## 💻 실전 실험

### 실험 1: Deadline 전파 확인

```java
// 다음 코드로 deadline이 체인을 통해 어떻게 전파되는지 추적할 수 있습니다

@GrpcService
public class ServiceAImpl extends ServiceAGrpc.ServiceAImplBase {
    private final ServiceBGrpc.ServiceBBlockingStub bStub;
    
    @Override
    public void callChain(Request req, StreamObserver<Response> responseObserver) {
        Deadline deadline = Context.current().getDeadline();
        System.out.println("[Service A] Deadline: " + deadline);
        System.out.println("[Service A] Time until deadline: " + 
            (deadline.timeRemaining(TimeUnit.MILLISECONDS)) + "ms");
        
        try {
            Response response = bStub.callChain(req);
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (StatusRuntimeException e) {
            responseObserver.onError(e);
        }
    }
}

@GrpcService
public class ServiceBImpl extends ServiceBGrpc.ServiceBImplBase {
    private final ServiceCGrpc.ServiceCBlockingStub cStub;
    
    @Override
    public void callChain(Request req, StreamObserver<Response> responseObserver) {
        Deadline deadline = Context.current().getDeadline();
        System.out.println("[Service B] Deadline: " + deadline);
        System.out.println("[Service B] Time until deadline: " + 
            (deadline.timeRemaining(TimeUnit.MILLISECONDS)) + "ms");
        
        try {
            // 100ms 대기
            Thread.sleep(100);
            
            Response response = cStub.callChain(req);
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (StatusRuntimeException e) {
            responseObserver.onError(e);
        } catch (InterruptedException e) {
            responseObserver.onError(Status.INTERNAL.asException());
        }
    }
}

@GrpcService
public class ServiceCImpl extends ServiceCGrpc.ServiceCImplBase {
    
    @Override
    public void callChain(Request req, StreamObserver<Response> responseObserver) {
        Deadline deadline = Context.current().getDeadline();
        System.out.println("[Service C] Deadline: " + deadline);
        System.out.println("[Service C] Time until deadline: " + 
            (deadline.timeRemaining(TimeUnit.MILLISECONDS)) + "ms");
        
        try {
            // 작업 수행 중 주기적으로 deadline 확인
            for (int i = 0; i < 10; i++) {
                if (deadline.isExpired()) {
                    System.out.println("[Service C] Deadline exceeded at step " + i);
                    responseObserver.onError(
                        Status.DEADLINE_EXCEEDED.asException()
                    );
                    return;
                }
                Thread.sleep(50);
                System.out.println("[Service C] Step " + i + ", time left: " +
                    deadline.timeRemaining(TimeUnit.MILLISECONDS) + "ms");
            }
            
            Response response = Response.newBuilder()
                .setMessage("Success from C")
                .build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
            
        } catch (InterruptedException e) {
            responseObserver.onError(Status.INTERNAL.asException());
        }
    }
}

// 테스트 코드
@Test
public void testDeadlinePropagation() {
    ServiceAGrpc.ServiceABlockingStub stub = 
        ServiceAGrpc.newBlockingStub(channel)
            .withDeadlineAfter(2, TimeUnit.SECONDS);
    
    Request request = Request.newBuilder()
        .setMessage("Test deadline")
        .build();
    
    Response response = stub.callChain(request);
    assertNotNull(response);
}
```

### 실험 2: Deadline Timeout 동작 확인

```java
@Test
public void testDeadlineTimeout() throws InterruptedException {
    ServiceAGrpc.ServiceABlockingStub stub = 
        ServiceAGrpc.newBlockingStub(channel)
            .withDeadlineAfter(500, TimeUnit.MILLISECONDS); // 매우 짧은 deadline
    
    Request request = Request.newBuilder()
        .setMessage("Test timeout")
        .build();
    
    long startTime = System.currentTimeMillis();
    
    try {
        Response response = stub.callChain(request);
        fail("Expected DEADLINE_EXCEEDED");
    } catch (StatusRuntimeException e) {
        long elapsedTime = System.currentTimeMillis() - startTime;
        
        assertEquals(Status.DEADLINE_EXCEEDED, e.getStatus());
        
        // deadline이 정확하게 지켜지는지 확인
        // 500ms ± 50ms 범위 내에서 타임아웃되어야 함
        assertTrue(elapsedTime >= 500 && elapsedTime < 700,
            "Expected timeout around 500ms, got " + elapsedTime + "ms");
        
        System.out.println("Timeout occurred at: " + elapsedTime + "ms");
    }
}
```

### 실험 3: Network tracing으로 grpc-timeout 헤더 확인

```bash
# tcpdump를 사용하여 HTTP/2 헤더 캡처
tcpdump -i lo -s 0 'tcp port 50051' -A | grep -i timeout

# 출력 예시:
# grpc-timeout: 5000m (5000 milliseconds)
# grpc-timeout: 4850m (deadline이 전파되면서 감소)
# grpc-timeout: 4700m
```

## 📊 성능 비교

### Deadline 설정 유무에 따른 성능 비교

| 지표 | Deadline 없음 | Deadline 3초 | Deadline 1초 |
|------|--------------|------------|-----------|
| P50 (중간값) | 150ms | 145ms | 150ms |
| P95 (95 percentile) | 2800ms | 2850ms | 950ms |
| P99 (99 percentile) | 28000ms | 3050ms | 1020ms |
| Max | 120초 | 3100ms | 1150ms |
| 성공률 | 98.5% | 99.2% | 98.8% |
| 리소스 해제 시간 | ~30초 (타임아웃 대기) | ~3초 | ~1초 |

**해석:**
- Deadline이 없으면 느린 요청이 최대 120초 이상 대기하여 tail latency 증가
- Deadline을 설정하면 P95, P99가 크게 개선됨
- 너무 짧은 deadline (1초)은 정상 요청도 타임아웃시킬 수 있음
- 적절한 deadline (3초) 설정으로 tail latency 제어와 높은 성공률 달성

### 체인 단계별 Deadline 감소

```
┌─────────────────────────────────────┐
│ Client → A → B → C 체인             │
│ 초기 Deadline: 5000ms                │
└─────────────────────────────────────┘

단계 | 경과 시간 | 남은 Deadline | 요청 전송 시 gRPC 헤더
-----|---------|--------------|--------------------
A 입장 | 50ms | 4950ms | grpc-timeout=4950m
B 입장 | 100ms | 4900ms | grpc-timeout=4900m
C 입장 | 150ms | 4850ms | grpc-timeout=4850m
C 처리 중 | 1000ms | 4000ms | (주기적 확인)
C 처리 중 | 2000ms | 3000ms | (주기적 확인)
C 완료 | 2500ms | 2500ms | (정상 응답)
B 처리 | 2600ms | 2400ms | (응답 조합)
A 완료 | 2650ms | 2350ms | (최종 응답)
```

## ⚖️ 트레이드오프

| 항목 | 짧은 Deadline (1초) | 중간 Deadline (3초) | 긴 Deadline (10초) |
|------|-----------------|-----------------|-----------------|
| **Tail Latency 제어** | 매우 좋음 | 좋음 | 나쁨 |
| **리소스 낭비** | 최소 | 중간 | 많음 |
| **정상 요청 성공률** | 낮음 (50%↓) | 높음 (99%↑) | 매우 높음 (99.9%↑) |
| **복합 쿼리 처리** | 불가능 | 가능 | 충분함 |
| **네트워크 불안정성 대응** | 약함 | 보통 | 좋음 |
| **모니터링 난이도** | 쉬움 | 중간 | 어려움 |
| **에러율** | 높음 | 낮음 | 매우 낮음 |

**권장:**
- **Fast API** (단순 조회): 1~2초
- **Normal API** (복합 조회): 3~5초
- **Slow API** (DB 쿼리, 계산): 5~10초
- **Batch Job**: 30초 이상

## 📌 핵심 정리

1. **Deadline은 절대 시각, Timeout은 상대 기간**: `withDeadline()`은 특정 시점까지, `withDeadlineAfter()`는 현재로부터 N초 후까지를 의미합니다. 대부분의 경우 `withDeadlineAfter()` 사용이 편리합니다.

2. **자동 Context 전파로 체인 단순화**: A→B→C 체인에서 클라이언트의 deadline이 자동으로 모든 downstream 호출에 포함되어, 각 서비스가 남은 시간을 정확히 알 수 있습니다.

3. **주기적 deadline 확인으로 빠른 응답**: 서버에서 오래 걸리는 작업 중간중간 `deadline.isExpired()`를 확인하면, 불필요한 계산을 즉시 중단하고 RST_STREAM 프레임으로 리소스를 즉시 해제합니다.

4. **Tail Latency 대폭 감소**: Deadline 설정으로 P99 latency가 10배 이상 개선되며, 리소스도 빠르게 정리되어 전체 시스템 안정성이 높아집니다.

5. **API 특성에 맞는 deadline 설정 필수**: 모든 API에 동일한 deadline을 적용하기보다 복잡도에 따라 차등 설정하는 것이 효과적입니다.

## 🤔 생각해볼 문제

1. **Cascading Timeout 방지**: A에서 deadline을 10초로 설정했는데, B가 9초, C가 9초 걸린다면 최종 완료는 18초가 되어 A의 deadline을 초과합니다. 이런 상황에서 어떻게 deadline을 설정해야 할까요? 각 단계별로 얼마나 할당하는 것이 합리적일까요?

2. **Timeout vs Retry 전략**: Deadline 초과로 인한 DEADLINE_EXCEEDED 에러가 발생했을 때, 남은 시간이 1초만 있다면 이를 다시 시도해야 할까요? 아니면 즉시 에러를 반환해야 할까요? 이를 판단하는 기준은 무엇일까요?

3. **부분 실패 처리**: A→B→C 체인에서 C가 deadline 초과로 실패했지만, B가 이미 수행한 작업(예: DB insert)의 롤백은 어떻게 처리하나요? Deadline 초과가 비즈니스 에러와 동일하게 취급되어야 할까요?

---

**[⬅️ 이전: 메타데이터](./03-metadata.md)** | **[홈으로 🏠](../README.md)** | **[다음: 로드밸런싱 ➡️](./05-load-balancing.md)**
