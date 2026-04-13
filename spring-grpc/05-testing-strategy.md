# 테스트 전략 — 단위·통합·Mock Stub

---

## 🎯 핵심 질문

- InProcessChannel을 사용한 단위 테스트는 실제 네트워크 테스트와 무엇이 다른가?
- GrpcServerExtension(JUnit 5)으로 내장 서버를 띄우는 방법은?
- Mock Stub으로 클라이언트 코드를 격리해 테스트하는 방법은?
- Testcontainers로 실제 gRPC 서버 컨테이너를 올려 통합 테스트하는 방법은?
- 비동기 Server Streaming 테스트를 어떻게 동기적으로 검증하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC는 비동기 호출이므로 일반 단위 테스트 패턴이 통용되지 않습니다. InProcessChannel로 네트워크를 피하거나 Testcontainers로 실제 환경을 시뮬레이션해야 정확한 테스트가 가능합니다.

---

## 😱 흔한 실수 (Before)

```java
// 실수 1: Mock 없이 실제 서버 포트 사용
@Test
public void testGetUser() {
    ManagedChannel channel = 
        ManagedChannelBuilder
            .forAddress("localhost", 9090)
            .usePlaintext()
            .build();
    // ❌ 실제 서버 실행 필요, 포트 충돌
}

// 실수 2: CountDownLatch 없이 비동기 테스트
@Test
public void testStreamOrders() {
    List<Order> orders = new ArrayList<>();
    
    stub.streamOrders(request, 
        new StreamObserver<Order>() {
            public void onNext(Order order) {
                orders.add(order);
            }
        }
    );
    
    assertEquals(10, orders.size());  // ❌ 비동기, 실행 안 됨
}

// 실수 3: InProcessChannel을 사용하지 않음
// → 불필요한 포트 할당, 느린 테스트
```

---

## ✨ 올바른 접근 (After)

```java
// 1. InProcessChannel을 사용한 단위 테스트
@ExtendWith(GrpcCleanupExtension.class)
public class UserServiceUnitTest {
    
    private UserServiceImpl service;
    private Server inProcessServer;
    private ManagedChannel channel;
    private UserServiceGrpc.UserServiceBlockingStub stub;
    
    @BeforeEach
    void setUp() throws IOException {
        String serverName = 
            InProcessServerBuilder.generateName();
        
        inProcessServer = InProcessServerBuilder
            .forName(serverName)
            .directExecutor()
            .addService(new UserServiceImpl(
                mock(UserRepository.class)))
            .build()
            .start();
        
        channel = InProcessChannelBuilder
            .forName(serverName)
            .directExecutor()
            .build();
        
        stub = UserServiceGrpc.newBlockingStub(channel);
    }
    
    @Test
    void testGetUser() {
        GetUserRequest request = GetUserRequest.newBuilder()
            .setId("user123")
            .build();
        
        GetUserResponse response = stub.getUser(request);
        
        assertEquals("user123", response.getId());
    }
}

// 2. GrpcServerExtension을 사용한 통합 테스트
@ExtendWith(GrpcServerExtension.class)
public class UserServiceIntegrationTest {
    
    private UserServiceGrpc.UserServiceBlockingStub stub;
    
    @RegisterExtension
    static final GrpcServerExtension grpcServer =
        new GrpcServerExtension() {
            @Override
            protected void configureServerBuilder(
                    ServerBuilder<?> serverBuilder) {
                serverBuilder.addService(
                    new UserServiceImpl(
                        new UserRepositoryImpl()
                    )
                );
            }
        };
    
    @BeforeEach
    void setUp(GrpcServerExtension grpcServer) {
        ManagedChannel channel = 
            ManagedChannelBuilder
                .forAddress("localhost", 
                           grpcServer.getPort())
                .usePlaintext()
                .build();
        stub = UserServiceGrpc.newBlockingStub(channel);
    }
    
    @Test
    void testCreateUser() {
        CreateUserRequest request = 
            CreateUserRequest.newBuilder()
                .setName("John")
                .build();
        
        CreateUserResponse response = 
            stub.createUser(request);
        
        assertTrue(response.getId().length() > 0);
    }
}

// 3. Mock Stub으로 클라이언트 테스트
@ExtendWith(MockitoExtension.class)
public class OrderClientTest {
    
    @Mock
    private OrderServiceGrpc.OrderServiceBlockingStub stub;
    
    private OrderClient orderClient;
    
    @BeforeEach
    void setUp() {
        orderClient = new OrderClient(stub);
    }
    
    @Test
    void testGetOrderWithMockStub() {
        Order mockOrder = Order.newBuilder()
            .setId("order123")
            .setAmount(100)
            .build();
        
        when(stub.getOrder(any()))
            .thenReturn(mockOrder);
        
        Order result = orderClient.getOrder("order123");
        
        assertEquals("order123", result.getId());
        verify(stub, times(1)).getOrder(any());
    }
}

// 4. 비동기 스트리밍 테스트 (CountDownLatch)
@Test
void testStreamOrdersWithCountDownLatch() 
        throws InterruptedException {
    List<Order> orders = Collections
        .synchronizedList(new ArrayList<>());
    
    CountDownLatch latch = new CountDownLatch(1);
    
    stub.streamOrders(
        StreamOrderRequest.newBuilder()
            .setCustomerId("customer123")
            .build(),
        new StreamObserver<Order>() {
            @Override
            public void onNext(Order order) {
                orders.add(order);
            }
            
            @Override
            public void onError(Throwable t) {
                latch.countDown();
            }
            
            @Override
            public void onCompleted() {
                latch.countDown();
            }
        }
    );
    
    // 최대 5초 대기
    assertTrue(latch.await(5, TimeUnit.SECONDS),
              "Stream did not complete");
    
    assertEquals(10, orders.size());
}
```

---

## 🔬 내부 동작 원리

### 1. 테스트 유형별 비교 (단위/통합/E2E)

```
┌─────────────────────────────────────────────┐
│ 테스트 유형별 특성 비교                    │
├─────────────────────────────────────────────┤
│                                              │
│  특성          단위 테스트   통합 테스트   │
│  ────────────────────────────────────────   │
│  속도          빠름         중간            │
│  격리도        높음 (Mock)   중간           │
│  네트워크      없음         실제 사용      │
│  도구          InProcess    Testcontainers│
│                Channel                      │
│  비용          낮음         높음            │
│  신뢰도        중간         높음            │
│                                              │
│  추천: 단위 80% + 통합 20%                 │
└─────────────────────────────────────────────┘

InProcessChannel:
├─ 장점: 네트워크 무시, 빠름 (1~5ms)
├─ 용도: 단위 테스트
└─ 제약: 실제 네트워크 시뮬레이션 안 됨

Testcontainers:
├─ 장점: 실제 컨테이너, 네트워크 포함
├─ 용도: 통합 테스트
└─ 제약: 느림 (100~500ms)
```

### 2. InProcessChannel + GrpcServerExtension 단위 테스트

```java
@ExtendWith(GrpcCleanupExtension.class)
public class UserServiceTest {
    
    private UserServiceImpl service;
    private Server server;
    private ManagedChannel channel;
    private UserServiceGrpc.UserServiceBlockingStub stub;
    
    @BeforeEach
    void setUp() throws IOException {
        // 1. 메모리 내 서버 생성
        String serverName = 
            InProcessServerBuilder.generateName();
        
        service = new UserServiceImpl(
            mock(UserRepository.class));
        
        server = InProcessServerBuilder
            .forName(serverName)
            .directExecutor()  // 같은 스레드 실행
            .addService(service)
            .build()
            .start();
        
        // 2. 메모리 내 채널 생성
        channel = InProcessChannelBuilder
            .forName(serverName)
            .directExecutor()  // 같은 스레드 실행
            .build();
        
        stub = UserServiceGrpc.newBlockingStub(channel);
    }
    
    @AfterEach
    void tearDown() throws InterruptedException {
        channel.shutdownNow().awaitTermination(
            5, TimeUnit.SECONDS);
        server.shutdownNow().awaitTermination(
            5, TimeUnit.SECONDS);
    }
    
    @Test
    void testGetUserSuccess() {
        GetUserRequest request = 
            GetUserRequest.newBuilder()
                .setId("user123")
                .build();
        
        GetUserResponse response = stub.getUser(request);
        
        assertEquals("user123", response.getId());
        assertNotNull(response.getName());
    }
    
    @Test
    void testGetUserNotFound() {
        GetUserRequest request = 
            GetUserRequest.newBuilder()
                .setId("nonexistent")
                .build();
        
        StatusRuntimeException ex = 
            assertThrows(
                StatusRuntimeException.class,
                () -> stub.getUser(request)
            );
        
        assertEquals(Status.NOT_FOUND.getCode(),
                    ex.getStatus().getCode());
    }
}
```

### 3. Mock Stub 패턴 (Mockito + StreamObserver)

```java
@ExtendWith(MockitoExtension.class)
public class OrderClientTest {
    
    @Mock
    private OrderServiceGrpc.OrderServiceBlockingStub stub;
    
    @Mock
    private OrderServiceGrpc.OrderServiceStub asyncStub;
    
    private OrderClient client;
    
    @BeforeEach
    void setUp() {
        client = new OrderClient(stub, asyncStub);
    }
    
    // Unary RPC 테스트
    @Test
    void testGetOrderWithMock() {
        Order mockOrder = Order.newBuilder()
            .setId("order123")
            .setAmount(100)
            .build();
        
        when(stub.getOrder(any()))
            .thenReturn(mockOrder);
        
        Order result = client.getOrder("order123");
        
        assertEquals("order123", result.getId());
        verify(stub).getOrder(
            argThat(req -> 
                req.getId().equals("order123")
            )
        );
    }
    
    // Server Streaming 테스트
    @Test
    void testStreamOrdersWithMock() {
        ArgumentCaptor<StreamObserver> captor = 
            ArgumentCaptor.forClass(
                StreamObserver.class
            );
        
        doAnswer(invocation -> {
            StreamObserver<Order> observer = 
                invocation.getArgument(1);
            
            observer.onNext(Order.newBuilder()
                .setId("order1").build());
            observer.onNext(Order.newBuilder()
                .setId("order2").build());
            observer.onCompleted();
            
            return null;
        }).when(asyncStub)
         .streamOrders(
            any(),
            captor.capture()
        );
        
        List<Order> orders = new ArrayList<>();
        client.streamOrders("customer123")
            .subscribe(orders::add);
        
        assertEquals(2, orders.size());
    }
}
```

### 4. Testcontainers gRPC 통합 테스트

```java
@SpringBootTest(
    webEnvironment = SpringBootTest.WebEnvironment
        .RANDOM_PORT)
@Testcontainers
public class GrpcIntegrationTest {
    
    @Container
    static GenericContainer<?> grpcServer =
        new GenericContainer<>(
            DockerImageName.parse(
                "my-grpc-server:latest"
            )
        )
        .withExposedPorts(9090);
    
    @BeforeEach
    void setUp() {
        String host = grpcServer.getHost();
        Integer port = grpcServer
            .getMappedPort(9090);
        
        ManagedChannel channel = 
            ManagedChannelBuilder
                .forAddress(host, port)
                .usePlaintext()
                .build();
        
        stub = UserServiceGrpc
            .newBlockingStub(channel);
    }
    
    @Test
    void testRealServerIntegration() {
        GetUserResponse response = stub.getUser(
            GetUserRequest.newBuilder()
                .setId("user123")
                .build()
        );
        
        assertNotNull(response);
    }
}
```

---

## 💻 실전 실험

```bash
# Test 실행
mvn test

# 특정 테스트만 실행
mvn test -Dtest=UserServiceTest

# 통합 테스트만 실행
mvn test -Dtest=*IntegrationTest
```

---

## 📊 성능/비용 비교

```
┌─────────────────────────────────────┐
│ 테스트 유형별 성능 비교            │
├─────────────────────────────────────┤
│                                      │
│ 유형      시간    비용  신뢰도      │
│ ──────────────────────────────────  │
│ 단위      1ms    낮음  중간        │
│ 통합      100ms  중간  높음        │
│ E2E       1s     높음  최고        │
│                                      │
│ 권장: 단위 70%, 통합 20%, E2E 10% │
└─────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
InProcessChannel:
├─ 장점: 빠름, 간단
└─ 단점: 네트워크 시뮬레이션 없음

Testcontainers:
├─ 장점: 실제 환경
└─ 단점: 느림, 복잡

권장: 단위 테스트는 InProcessChannel,
      통합 테스트는 Testcontainers
```

---

## 📌 핵심 정리

```
1. InProcessChannel: 단위 테스트 (빠름)
2. GrpcServerExtension: 통합 테스트
3. Mock Stub: 클라이언트 격리
4. CountDownLatch: 비동기 스트리밍
5. Testcontainers: 실제 환경 시뮬레이션
```

---

## 🤔 생각해볼 문제

### Q1: CountDownLatch를 사용하지 않으면?

<details>
<summary>해설 보기</summary>

**정답: 테스트가 조기 종료 (결과 미검증)**

```
CountDownLatch 없음:
├─ streamOrders() 호출
├─ (비동기 호출)
├─ assert 실행 (orders는 아직 비어있음)
└─ 테스트 통과 (잘못된 통과!)

CountDownLatch 있음:
├─ streamOrders() 호출 (비동기)
├─ latch.await() (대기)
├─ 스트림 완료 시 latch.countDown()
├─ assert 실행 (orders 확인)
└─ 테스트 검증 완료
```

</details>

---

### Q2: InProcessChannel vs 실제 네트워크 채널의 차이?

<details>
<summary>해설 보기</summary>

**정답: 속도와 정확성의 트레이드오프**

```
InProcessChannel (메모리):
├─ 속도: 1~5ms
├─ 네트워크 오버헤드: 없음
└─ 현실성: 낮음

실제 네트워크 채널:
├─ 속도: 50~500ms
├─ 네트워크 오버헤드: 포함
└─ 현실성: 높음

사용 시기:
├─ 단위 테스트 → InProcessChannel
├─ 통합 테스트 → 실제 채널
└─ 성능 테스트 → 실제 환경
```

</details>

---

### Q3: Mock Stub은 언제 사용?

<details>
<summary>해설 보기</summary>

**정답: 외부 의존성이 없을 때 클라이언트 테스트**

```
Mock Stub 사용:
├─ 상황: gRPC 서버가 별도 팀 개발중
├─ 목표: 클라이언트 로직만 테스트
├─ 이점: 서버 없이 빠른 테스트
└─ 비용: Mock 유지보수 필요

실제 서버 사용:
├─ 상황: 통합 테스트 필요
├─ 목표: 전체 흐름 검증
├─ 이점: 현실적 검증
└─ 비용: 느린 테스트
```

</details>

---

**[⬅️ 이전: gRPC + Spring WebFlux](./04-grpc-webflux.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 7 — gRPC 모니터링 ➡️](../performance-operations/01-grpc-monitoring.md)**
