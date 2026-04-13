# gRPC + Spring WebFlux — Reactive Stub 통합

---

## 🎯 핵심 질문

- reactor-grpc-stub이 생성하는 ReactorServiceStub은 무엇이 다른가?
- gRPC Unary RPC를 Mono로, Server Streaming을 Flux로 변환하는 방법은?
- Reactive Streams의 Backpressure가 HTTP/2 Flow Control과 어떻게 연동되는가?
- Blocking Stub을 WebFlux 환경에서 쓰면 왜 위험한가?
- subscribeOn(Schedulers.boundedElastic())은 언제 필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Blocking Stub으로 gRPC를 호출하면 WebFlux의 이벤트 루프 스레드를 블로킹하므로 전체 시스템이 응답 불능이 됩니다. reactor-grpc-stub을 사용해 Reactive 패턴으로 변환해야 높은 동시성을 유지할 수 있습니다.

---

## 😱 흔한 실수 (Before)

```java
// 실수 1: WebFlux 환경에서 BlockingStub 사용
@RestController
public class UserController {
    @Autowired
    private UserServiceGrpc.UserServiceBlockingStub stub;
    
    @GetMapping("/users/{id}")
    public Mono<User> getUser(@PathVariable String id) {
        return Mono.fromCallable(() -> 
            stub.getUser(GetUserRequest.newBuilder()
                .setId(id)
                .build())
        );  // ❌ 스레드 블로킹 → 이벤트 루프 정체
    }
}

// 실수 2: Flux를 구독하지 않고 리턴
@RestController
public class OrderController {
    @Autowired
    private OrderServiceGrpc.OrderServiceStub stub;
    
    @GetMapping("/orders/{id}")
    public Flux<Order> streamOrders(@PathVariable String id) {
        Flux<Order> flux = Flux.create(sink -> {
            stub.streamOrders(
                StreamOrderRequest.newBuilder()
                    .setId(id)
                    .build(),
                new StreamObserver<Order>() {
                    public void onNext(Order order) {
                        sink.next(order);
                    }
                    public void onError(Throwable t) {
                        sink.error(t);
                    }
                    public void onCompleted() {
                        sink.complete();
                    }
                }
            );
        });
        
        return flux;  // ❌ 구독이 없으면 호출 안 됨
    }
}

// 실수 3: 에러 처리 없이 Mono/Flux 리턴
Mono.just(someValue)
    .flatMap(v -> Mono.fromCallable(() -> 
        stub.process(v)
    ));  // ❌ 에러 발생 시 구독자에게 전달 안 됨
```

---

## ✨ 올바른 접근 (After)

```xml
<!-- pom.xml: reactor-grpc-stub 의존성 -->
<dependency>
    <groupId>io.grpc</groupId>
    <artifactId>grpc-stub</artifactId>
    <version>1.53.0</version>
</dependency>
<dependency>
    <groupId>com.salesforce.servicelibs</groupId>
    <artifactId>reactor-grpc-stub</artifactId>
    <version>1.2.4</version>
</dependency>
```

```java
// Reactive Stub 주입
@Service
public class UserService {
    
    @GrpcClient("user-service")
    private ReactorUserServiceGrpc.ReactorUserServiceStub reactiveStub;
    
    // Unary RPC → Mono
    public Mono<User> getUserReactive(String id) {
        return ReactorStubs.oneToOne(
            observer -> reactiveStub.getUser(
                GetUserRequest.newBuilder()
                    .setId(id)
                    .build(),
                observer
            )
        ).subscribeOn(Schedulers.boundedElastic());
    }
    
    // Server Streaming RPC → Flux
    public Flux<Order> streamOrders(String customerId) {
        return ReactorStubs.oneToMany(
            observer -> reactiveStub.streamOrders(
                StreamOrderRequest.newBuilder()
                    .setCustomerId(customerId)
                    .build(),
                observer
            )
        ).subscribeOn(Schedulers.boundedElastic());
    }
}

// REST Controller: Reactive 사용
@RestController
@RequiredArgsConstructor
public class UserController {
    
    private final UserService userService;
    
    @GetMapping("/users/{id}")
    public Mono<UserDTO> getUser(@PathVariable String id) {
        return userService.getUserReactive(id)
            .map(this::toDTO)
            .onErrorResume(error -> 
                Mono.error(new ServiceException(
                    "User service failed", error))
            );
    }
    
    @GetMapping("/orders/{customerId}/stream")
    public Flux<OrderDTO> streamOrders(
            @PathVariable String customerId) {
        return userService.streamOrders(customerId)
            .map(this::toDTO)
            .onErrorResume(error -> 
                Flux.error(new ServiceException(
                    "Order service failed", error))
            );
    }
    
    private UserDTO toDTO(User user) {
        return UserDTO.builder()
            .id(user.getId())
            .name(user.getName())
            .build();
    }
    
    private OrderDTO toDTO(Order order) {
        return OrderDTO.builder()
            .orderId(order.getId())
            .amount(order.getAmount())
            .build();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Reactive ↔ StreamObserver 브리지

```
ReactorServiceStub (Reactive)
    │
    ├─ Unary RPC: Mono<T>
    │  ┌───────────────────────────────┐
    │  │ StreamObserver 감싸기         │
    │  │ Subscriber 구독하면          │
    │  │ onNext() 1회 호출             │
    │  └───────────────────────────────┘
    │
    ├─ Server Streaming: Flux<T>
    │  ┌───────────────────────────────┐
    │  │ StreamObserver 감싸기         │
    │  │ Subscriber 구독하면          │
    │  │ onNext() 여러 번 호출        │
    │  │ onCompleted() 완료 신호      │
    │  └───────────────────────────────┘
    │
    └─ Bidirectional Streaming: Flux<ReqT> → Flux<RespT>
       ┌───────────────────────────────┐
       │ 양방향 StreamObserver 브리지  │
       │ Flux 발행 = gRPC 전송        │
       │ Flux 수신 = gRPC 수신        │
       └───────────────────────────────┘
```

### 2. Unary → Mono 변환

```java
// BlockingStub (문제)
GetUserResponse response = blockingStub.getUser(request);
// → 동기 블로킹, WebFlux에서 이벤트 루프 차단

// ReactiveStub (해결)
Mono<GetUserResponse> response = 
    ReactorStubs.oneToOne(observer -> 
        reactiveStub.getUser(request, observer)
    );
// → 비동기 Mono, 이벤트 루프 유지

// 내부 동작
/*
1. oneToOne() 호출
2. Mono 객체 생성 (구독 대기)
3. Subscriber 구독 시
4. StreamObserver 래퍼 생성
5. gRPC 호출 (비동기)
6. onNext(response) 호출
7. Mono.onNext(response)
8. 완료
*/
```

### 3. Server Streaming → Flux 변환

```java
// BlockingStub (문제)
Iterator<Order> response = blockingStub.streamOrders(request);
for (Order order : response) {
    // 처리
}  // → 동기 이터레이션, 블로킹

// ReactiveStub (해결)
Flux<Order> response = 
    ReactorStubs.oneToMany(observer -> 
        reactiveStub.streamOrders(request, observer)
    );

response.subscribe(order -> {
    // 처리
});  // → 비동기 스트림, 논-블로킹

// 내부 동작
/*
1. oneToMany() 호출
2. Flux 객체 생성 (구독 대기)
3. Subscriber 구독 시
4. StreamObserver 래퍼 생성
5. gRPC 호출 (서버 스트리밍 시작)
6. onNext(order1) → Flux.onNext(order1)
7. onNext(order2) → Flux.onNext(order2)
8. onNext(orderN) → Flux.onNext(orderN)
9. onCompleted() → Flux.onComplete()
10. 완료
*/
```

### 4. Backpressure 연동 (Reactive Streams ↔ HTTP/2 Flow Control)

```
Subscriber가 느린 처리 중:
request(1)  ← Subscriber가 1개만 요청
    │
    ▼
┌──────────────────────┐
│ Flux 배압 신호       │
│ (Reactive Streams)   │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ gRPC Flow Control    │
│ WINDOW_UPDATE 전송   │
│ (HTTP/2)             │
└──────────┬───────────┘
           │
           ▼
┌──────────────────────┐
│ 서버가 1개만 전송    │
│ 나머지 대기          │
└──────────────────────┘

장점: 메모리 소비 제어
      수신자가 처리 가능한 속도로 수신
```

---

## 💻 실전 실험

```java
// Configuration: ReactorServiceStub 자동 생성
@Configuration
public class GrpcClientConfiguration {
    
    @Bean
    public ReactorUserServiceGrpc.ReactorUserServiceStub 
            userServiceStub(ManagedChannel channel) {
        return ReactorUserServiceGrpc
            .newReactorStub(channel);
    }
    
    @Bean
    public ReactorOrderServiceGrpc.ReactorOrderServiceStub 
            orderServiceStub(ManagedChannel channel) {
        return ReactorOrderServiceGrpc
            .newReactorStub(channel);
    }
}

// Service: Reactive 처리
@Service
@Slf4j
public class OrderService {
    
    @Autowired
    private ReactorOrderServiceGrpc.ReactorOrderServiceStub 
        orderStub;
    
    // 단일 결과
    public Mono<Order> getOrder(String orderId) {
        return ReactorStubs.oneToOne(observer -> 
            orderStub.getOrder(
                GetOrderRequest.newBuilder()
                    .setId(orderId)
                    .build(),
                observer
            )
        ).subscribeOn(Schedulers.boundedElastic())
         .doOnError(error -> 
            log.error("Failed to get order", error));
    }
    
    // 스트림 결과
    public Flux<Order> listOrders(String customerId) {
        return ReactorStubs.oneToMany(observer -> 
            orderStub.listOrders(
                ListOrdersRequest.newBuilder()
                    .setCustomerId(customerId)
                    .build(),
                observer
            )
        ).subscribeOn(Schedulers.boundedElastic())
         .doOnError(error -> 
            log.error("Failed to list orders", error));
    }
    
    // 배압 제어
    public Flux<Order> listOrdersWithBackpressure(
            String customerId,
            int batchSize) {
        return listOrders(customerId)
            .buffer(batchSize)  // batchSize만큼 수집
            .flatMapIterable(batch -> batch)  // 분산
            .delayElement(Duration.ofMillis(100));  // 처리 속도 조절
    }
}

// Test
@SpringBootTest
public class GrpcWebFluxIntegrationTest {
    
    @Autowired
    private OrderService orderService;
    
    @Test
    public void testGetOrderMono() {
        orderService.getOrder("order123")
            .subscribe(
                order -> System.out.println(order),
                error -> System.err.println(error),
                () -> System.out.println("Complete")
            );
    }
    
    @Test
    public void testListOrdersFlux() {
        orderService.listOrders("customer456")
            .take(5)  // 처음 5개만
            .doOnNext(order -> 
                System.out.println(order))
            .blockLast();  // 테스트 대기
    }
}
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────┐
│ BlockingStub vs ReactiveStub       │
├────────────────────────────────────┤
│                                     │
│ 메트릭        BlockingStub  Reactive
│ ──────────────────────────────────  │
│ 동시 요청     100개         10000개  │
│ 스레드 수     100개         10개    │
│ 메모리        100MB         10MB    │
│ 응답 지연     정상          최소    │
│                                     │
│ Reactive의 장점:                   │
│ • 적은 스레드 → 메모리 절감        │
│ • 높은 동시성 → 처리량 증가        │
│ • 논-블로킹 → 지연시간 감소        │
└────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
BlockingStub:
├─ 장점: 구현 간단, 동기 흐름 명확
└─ 단점: WebFlux에서 위험 (블로킹)

ReactiveStub:
├─ 장점: 높은 동시성, 논-블로킹
└─ 단점: 코드 복잡도 증가, 학습곡선

권장:
├─ WebFlux 사용 → ReactiveStub 필수
├─ Spring MVC → BlockingStub 괜찮음
└─ 혼합 환경 → ReactiveStub 권장
```

---

## 📌 핵심 정리

```
1. WebFlux 환경: 반드시 ReactiveStub 사용
2. Unary RPC → Mono 변환
3. Server Streaming → Flux 변환
4. subscribeOn(boundedElastic()) 사용
5. 배압 처리 자동으로 됨
```

---

## 🤔 생각해볼 문제

### Q1: Blocking Stub을 WebFlux에서 사용하면?

<details>
<summary>해설 보기</summary>

**정답: 이벤트 루프 스레드 블로킹 → 전체 응답 불능**

```
Netty 이벤트 루프 (단일 스레드):
├─ Request A 도착
├─ blockingStub 호출 (블로킹)
│  └─ 응답 대기 중... (60ms)
├─ Request B 도착 (대기 중)
├─ Request C 도착 (대기 중)
└─ Request A 완료 (60ms 후)

결과: 모든 요청이 대기
```

**올바른 방법:**

```
Netty 이벤트 루프 (단일 스레드):
├─ Request A → reactiveStub (비동기)
├─ Request B → reactiveStub (비동기)
├─ Request C → reactiveStub (비동기)
└─ 응답들이 들어올 때마다 처리
```

</details>

---

### Q2: subscribeOn(Schedulers.boundedElastic())은 언제 필요한가?

<details>
<summary>해설 보기</summary>

**정답: I/O 바운드 작업을 별도 스레드 풀에서 실행할 때**

```
boundedElastic() 스케줄러:
├─ I/O 바운드 작업 전용 (네트워크, DB)
├─ 최대 100개 스레드 풀
├─ 스레드 재사용 (효율적)
└─ 이벤트 루프 블로킹 방지

사용 패턴:

❌ 나쁜 예: Schedulers.immediate()
Mono.fromCallable(() -> gRPC호출())
    .subscribeOn(Schedulers.immediate())
    // → 호출 스레드에서 실행

✅ 좋은 예: boundedElastic()
Mono.fromCallable(() -> gRPC호출())
    .subscribeOn(Schedulers.boundedElastic())
    // → 별도 스레드에서 실행
```

</details>

---

### Q3: Flux를 리턴하면 자동으로 스트리밍되는가?

<details>
<summary>해설 보기</summary>

**정답: 아니오. 구독이 있어야 스트리밍 시작**

```
❌ 문제: 구독 없음
Flux<Order> flux = service.streamOrders();
return flux;  // 스트리밍 안 시작됨

✅ 올바른 방법 1: REST 컨트롤러가 자동 구독
@GetMapping(produces = 
    "application/x-ndjson")
public Flux<Order> stream() {
    return service.streamOrders();
    // → Spring이 자동으로 구독 후 전송
}

✅ 올바른 방법 2: 명시적 구독
Flux<Order> flux = service.streamOrders();
flux.subscribe(
    order -> system.out.println(order),
    error -> log.error(error),
    () -> log.info("Complete")
);
```

</details>

---

**[⬅️ 이전: 예외 처리 통합](./03-exception-handling.md)** | **[홈으로 🏠](../README.md)** | **[다음: 테스트 전략 ➡️](./05-testing-strategy.md)**
