# gRPC 로드밸런싱: L4의 함정과 L7 해결책

## 🎯 핵심질문

TCP 로드밸런서가 일반적인 웹 애플리케이션에서는 잘 작동하지만, gRPC 환경에서는 왜 모든 요청이 한 서버로만 가게 될까요? HTTP/2의 어떤 특성 때문에 이런 일이 발생하고, L7 로드밸런서와 클라이언트 사이드 로드밸런싱은 어떻게 이 문제를 해결할까요?

## 🔍 왜 중요한가

프로덕션 환경에서 gRPC 서버 3개를 배포했는데, 모니터링 대시보드에서 1개 서버만 높은 CPU 사용률을 보이고 나머지는 유휴 상태라면 심각한 문제입니다. 이는 단순한 설정 오류가 아니라 gRPC의 기본 아키텍처와 HTTP/2의 특성으로 인한 필연적인 결과입니다.

1. **단일 TCP 연결의 재사용**: HTTP/2는 한 번 연결된 TCP 소켓에서 여러 스트림(요청)을 멀티플렉싱합니다. 클라이언트가 LB와 한 번 연결되면, LB가 이를 한 서버에 매핑하고, 이후의 모든 요청은 그 서버로 갑니다.

2. **L4 로드밸런서의 한계**: AWS NLB, F5, HAProxy (TCP 모드)는 연결 수준에서만 분산을 결정합니다. 연결 내의 개별 스트림은 볼 수 없으므로, HTTP/2의 장점을 활용할 수 없습니다.

3. **수평 확장의 무의미화**: 3개 서버를 운영해도 1개만 부하를 받으므로, 비용 대비 성능 향상이 거의 없습니다.

4. **병목 현상**: 모든 요청이 한 서버로 가면서 그 서버가 CPU 병목이 되고, 나머지 서버가 유휴 상태여서 전체 클러스터의 처리량이 제한됩니다.

5. **클라이언트 사이드 로드밸런싱의 필요성**: 클라이언트가 여러 서버 주소를 알고 직접 분산하면, 프록시 없이도 부하를 균등하게 분산할 수 있습니다.

## 😱 흔한 실수

### 실수 1: TCP 로드밸런서만 사용

```java
// ❌ 잘못된 접근: 아키텍처
// Client → NLB (TCP) → 서버 A, 서버 B, 서버 C
//
// 실제 동작:
// 1. Client가 NLB:443에 연결
// 2. NLB가 RoundRobin으로 서버 A 선택
// 3. Client ↔ 서버 A의 HTTP/2 연결 유지
// 4. 모든 요청이 서버 A로만 → 서버 B, C 유휴

@RestController
public class ClientController {
    private final OrderServiceGrpc.OrderServiceBlockingStub stub;
    
    @GetMapping("/orders/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable String id) {
        try {
            // 이 요청은 항상 같은 서버로 감
            // 왜냐하면 TCP 연결이 이미 정해졌기 때문
            OrderResponse response = stub.getOrder(
                GetOrderRequest.newBuilder()
                    .setOrderId(id)
                    .build()
            );
            return ResponseEntity.ok(response);
        } catch (StatusRuntimeException e) {
            return ResponseEntity.status(500).body(null);
        }
    }
}

// 문제점:
// 1. 서버 A의 CPU가 100% (병목)
// 2. 서버 B, C의 CPU가 0% (유휴)
// 3. 전체 처리량 = 1개 서버 용량
// 4. 수평 확장 효과 없음
```

### 실수 2: gRPC 기본 채널로 단일 서버 연결

```java
// ❌ 잘못된 접근: 로드밸런싱 정책 없음
@Configuration
public class GrpcClientConfig {
    
    @Bean
    public OrderServiceGrpc.OrderServiceBlockingStub orderServiceStub() {
        // 단순 주소 지정 → 단일 서버 연결
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress("10.0.1.10", 50051)  // 단일 IP
            .usePlaintext()
            .build();
        
        return OrderServiceGrpc.newBlockingStub(channel);
    }
}

// 문제점:
// 1. 항상 10.0.1.10으로만 연결
// 2. 서버 추가/제거 시 코드 수정 필요
// 3. 동적 스케일링 불가능
// 4. 서버 장애 시 자동 페일오버 없음
```

### 실수 3: 모든 요청마다 새로운 채널 생성

```java
// ❌ 잘못된 접근: 비효율적인 채널 재생성
@RestController
public class ClientController {
    
    @GetMapping("/orders/{id}")
    public ResponseEntity<OrderResponse> getOrder(@PathVariable String id) {
        try {
            // 매 요청마다 새로운 채널 생성 (매우 비효율적)
            ManagedChannel channel = ManagedChannelBuilder
                .forAddress("order-service", 50051)
                .usePlaintext()
                .build();
            
            OrderServiceGrpc.OrderServiceBlockingStub stub = 
                OrderServiceGrpc.newBlockingStub(channel);
            
            OrderResponse response = stub.getOrder(
                GetOrderRequest.newBuilder().setOrderId(id).build()
            );
            
            channel.shutdown();  // 연결 종료
            
            return ResponseEntity.ok(response);
        } catch (StatusRuntimeException e) {
            return ResponseEntity.status(500).body(null);
        }
    }
}

// 문제점:
// 1. 매 요청마다 TCP 연결 생성/소멸 (SSL/TLS handshake 오버헤드)
// 2. HTTP/2의 연결 재사용 장점 상실
// 3. 메모리 누수 (shutdown 실패 가능성)
// 4. 수백 개의 TIME_WAIT 소켓 발생
```

## ✨ 올바른 접근

### 올바른 구현 1: L7 로드밸런서 (Envoy)

```java
// ✅ 올바른 접근: Envoy 프록시로 gRPC 트래픽 분산
// 아키텍처:
// Client → Envoy:443 (gRPC 트래픽) → [Server A, B, C]
// 
// Envoy 설정:
/*
admin:
  access_log_path: /tmp/admin_access.log
  address:
    socket_address:
      address: 0.0.0.0
      port_number: 9901

static_resources:
  listeners:
  - name: grpc_listener
    address:
      socket_address:
        address: 0.0.0.0
        port_number: 443
    filter_chains:
    - filters:
      - name: envoy.filters.network.http_connection_manager
        typed_config:
          "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
          stat_prefix: grpc_stats
          codec_type: AUTO
          route_config:
            name: grpc_routes
            virtual_hosts:
            - name: grpc_service
              domains: ["*"]
              routes:
              - match:
                  prefix: "/"
                route:
                  cluster: grpc_backend
                  timeout: 10s
          http_filters:
          - name: envoy.filters.http.router
            typed_config:
              "@type": type.googleapis.com/envoy.extensions.filters.http.router.v3.Router

  clusters:
  - name: grpc_backend
    type: STRICT_DNS
    connect_timeout: 1s
    lb_policy: ROUND_ROBIN
    
    # gRPC 헬스체크
    health_checks:
    - timeout: 1s
      interval: 5s
      unhealthy_threshold: 3
      healthy_threshold: 2
      grpc_health_check:
        service_name: ""
    
    endpoints:
    - lb_endpoints:
      - endpoint:
          address:
            socket_address:
              address: server-a
              port_number: 50051
      - endpoint:
          address:
            socket_address:
              address: server-b
              port_number: 50051
      - endpoint:
          address:
            socket_address:
              address: server-c
              port_number: 50051
*/

@Configuration
public class GrpcClientConfigWithEnvoy {
    
    @Bean
    public OrderServiceGrpc.OrderServiceBlockingStub orderServiceStub() {
        // Envoy 프록시를 통한 연결
        ManagedChannel channel = ManagedChannelBuilder
            .forAddress("envoy-proxy.example.com", 443)
            .useTransportSecurity()
            .build();
        
        return OrderServiceGrpc.newBlockingStub(channel);
    }
}

// Envoy의 동작:
// 1. Client → Envoy로 단일 HTTP/2 연결
// 2. Envoy가 각 RPC 스트림을 다른 백엔드로 분산
// 3. Stream 1 (GetOrder) → Server A
// 4. Stream 2 (GetOrder) → Server B
// 5. Stream 3 (ListOrders) → Server C
// 
// 결과: 부하가 3개 서버에 균등 분산
```

### 올바른 구현 2: 클라이언트 사이드 로드밸런싱 (Kubernetes)

```java
// ✅ 올바른 접근: Kubernetes headless service + DNS
// 
// k8s 설정:
/*
apiVersion: v1
kind: Service
metadata:
  name: order-service
spec:
  clusterIP: None  # Headless 서비스!
  selector:
    app: order-service
  ports:
  - port: 50051
    targetPort: 50051
    protocol: TCP
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: order-service
spec:
  replicas: 3
  selector:
    matchLabels:
      app: order-service
  template:
    metadata:
      labels:
        app: order-service
    spec:
      containers:
      - name: order-service
        image: order-service:latest
        ports:
        - containerPort: 50051
*/

@Configuration
public class GrpcClientConfigWithK8s {
    
    @Bean
    public OrderServiceGrpc.OrderServiceBlockingStub orderServiceStub() {
        // Kubernetes headless service 사용
        ManagedChannel channel = ManagedChannelBuilder
            // "dns:///" 스키마로 DNS 기반 발견 활성화
            .forTarget("dns:///order-service.default.svc.cluster.local")
            .usePlaintext()
            // round_robin 정책 활성화
            .defaultLoadBalancingPolicy("round_robin")
            .build();
        
        return OrderServiceGrpc.newBlockingStub(channel);
    }
}

// DNS 기반 발견의 동작:
// 1. DNS 쿼리: order-service.default.svc.cluster.local
// 2. DNS 응답: [10.244.0.2, 10.244.0.3, 10.244.0.4] (3개 Pod)
// 3. gRPC 클라이언트가 3개 주소 모두 알게 됨
// 4. round_robin 정책으로 요청 분산
// 5. 자동 장애 감지 및 페일오버
```

### 올바른 구현 3: 커스텀 로드밸런싱 정책

```java
// ✅ 올바른 접근: 프로덕션급 클라이언트 설정
@Configuration
public class GrpcClientConfiguration {
    
    @Bean
    public ManagedChannel orderServiceChannel(
            @Value("${grpc.order-service.targets:server-a:50051,server-b:50051,server-c:50051}") 
            String targets) {
        
        // DNS 기반 또는 명시적 주소 지정
        return ManagedChannelBuilder
            .forTarget("dns:///order-service.example.com")
            .usePlaintext()
            // 로드밸런싱 정책 설정
            .defaultLoadBalancingPolicy("round_robin")  // or "least_request"
            // 헬스체크 설정
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(10, TimeUnit.SECONDS)
            .keepAliveWithoutCalls(true)
            // 재시도 정책
            .retryBufferSize(16 * 1024 * 1024)  // 16MB
            .perRpcBufferLimit(1024 * 1024)      // 1MB
            .build();
    }
    
    @Bean
    public OrderServiceGrpc.OrderServiceBlockingStub orderServiceStub(
            ManagedChannel channel) {
        return OrderServiceGrpc.newBlockingStub(channel);
    }
}

// application.yaml
/*
grpc:
  order-service:
    targets: "dns:///order-service.default.svc.cluster.local"
    load-balancing-policy: "round_robin"
    keep-alive-time: 30s
    keep-alive-timeout: 10s
*/
```

### 올바른 구현 4: 가중 로드밸런싱

```java
// ✅ 올바른 접근: 서버 용량에 따른 가중 분산
@Configuration
public class WeightedLoadBalancingConfig {
    
    @Bean
    public io.grpc.LoadBalancer.Helper createWeightedLB() {
        // 커스텀 가중 로드밸런싱 정책
        // 예: Server A (가중치 2), Server B (가중치 1), Server C (가중치 1)
        
        return new io.grpc.LoadBalancer.Helper() {
            @Override
            public SynchronizationContext getSynchronizationContext() {
                return null;
            }
            
            @Override
            public void updateBalancingState(ConnectivityState newState, 
                    SubchannelPicker newPicker) {
                // 가중치 기반 픽킹 로직
            }
        };
    }
}

// Service Config로 가중치 설정:
/*
{
  "loadBalancingConfig": [
    {"round_robin": {}},
    {
      "weighted_target": {
        "targets": {
          "server-a": {
            "weight": 2,
            "childPolicy": [{"round_robin": {}}]
          },
          "server-b": {
            "weight": 1,
            "childPolicy": [{"round_robin": {}}]
          },
          "server-c": {
            "weight": 1,
            "childPolicy": [{"round_robin": {}}]
          }
        }
      }
    }
  ]
}
*/
```

## 🔬 내부 동작 원리

### L4 vs L7 로드밸런싱 메커니즘

```
┌─────────────────────────────────────────────────────────────┐
│ L4 (TCP) 로드밸런싱의 문제점                                 │
└─────────────────────────────────────────────────────────────┘

Client → NLB:443 (TCP)
         │
         ├─ Connection 1 → Select Server A (RoundRobin)
         │    └─ TCP 연결 유지
         │
         └─ Connection 2 → Select Server B
              └─ TCP 연결 유지

HTTP/2 멀티플렉싱 (같은 TCP 연결):
┌──────────────────────────────────────────────────────────┐
│ Client ↔ Server A (Connection 1)                         │
│                                                          │
│ Stream 1: GetOrder(1) ──→ Server A                       │
│ Stream 2: GetOrder(2) ──→ Server A                       │
│ Stream 3: ListOrders() ──→ Server A                      │
│ Stream 4: GetOrder(3) ──→ Server A                       │
│ ...                                                       │
│                                                          │
│ 결과: 모든 요청이 Server A로만!                          │
└──────────────────────────────────────────────────────────┘

Server A 부하: 100%
Server B 부하: 0%
Server C 부하: 0%

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────┐
│ L7 (HTTP/2) 로드밸런싱의 해결책 (Envoy)                      │
└─────────────────────────────────────────────────────────────┘

Client → Envoy:443 (HTTP/2)
         │
         └─ HTTP/2 연결 (단일)
            │
            ├─ Stream 1: GetOrder(1) ──→ Server A
            ├─ Stream 2: GetOrder(2) ──→ Server B
            ├─ Stream 3: ListOrders() ──→ Server C
            ├─ Stream 4: GetOrder(3) ──→ Server A
            ├─ Stream 5: GetUser(1) ──→ Server B
            └─ ...
               
Envoy가 HTTP/2 스트림 단위로 분산!

결과:

Client 관점:
  Client ↔ Envoy (단일 TCP 연결, HTTP/2)
  
Envoy 관점:
  Envoy ↔ Server A (여러 TCP 연결)
  Envoy ↔ Server B (여러 TCP 연결)
  Envoy ↔ Server C (여러 TCP 연결)

Server A 부하: ~33%
Server B 부하: ~33%
Server C 부하: ~34%

전체 처리량: 3배 향상!

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

┌─────────────────────────────────────────────────────────────┐
│ 클라이언트 사이드 로드밸런싱                                 │
└─────────────────────────────────────────────────────────────┘

Client (gRPC)
  │
  ├─ Channel 1 → Server A:50051
  │  └─ Stream 1, 4: GetOrder
  │
  ├─ Channel 2 → Server B:50051
  │  └─ Stream 2, 5: GetOrder
  │
  └─ Channel 3 → Server C:50051
     └─ Stream 3: ListOrders

클라이언트가 직접 여러 서버에 연결
각 연결은 독립적인 HTTP/2 스트림 사용
로드밸런싱 정책 (round_robin, least_request)에 따라 분산

결과:
  프록시 불필요 (지연 감소)
  클라이언트가 직접 부하 분산
  장애 서버 자동 감지 및 우회
  DNS 기반 자동 스케일링 지원
```

### DNS 기반 클라이언트 사이드 로드밸런싱

```
┌──────────────────────────────────────────────────────────┐
│ 1. Kubernetes Service 설정                               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ kind: Service                                            │
│ metadata:                                                │
│   name: order-service                                    │
│ spec:                                                    │
│   clusterIP: None  # ← Headless!                         │
│   selector:                                              │
│     app: order-service                                   │
│   ports:                                                 │
│   - port: 50051                                          │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────────┐
│ 2. DNS 쿼리 & 응답                                        │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ $ nslookup order-service.default.svc.cluster.local       │
│                                                          │
│ Name:   order-service.default.svc.cluster.local          │
│ Address: 10.244.0.2 (Pod A)                              │
│ Address: 10.244.0.3 (Pod B)                              │
│ Address: 10.244.0.4 (Pod C)                              │
│                                                          │
│ (일반 Service라면 10.96.0.100 단일 VIP만 반환)            │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────────┐
│ 3. gRPC 클라이언트 채널 생성                               │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ ManagedChannel channel = ManagedChannelBuilder            │
│   .forTarget("dns:///order-service.default...")          │
│   .defaultLoadBalancingPolicy("round_robin")             │
│   .build();                                              │
│                                                          │
│ 내부 동작:                                               │
│ 1. DNS 쿼리 실행                                          │
│ 2. 반환된 3개 주소 발견                                     │
│ 3. 각 주소로 SubChannel 생성                              │
│ 4. round_robin 정책으로 스트림 분산                        │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────────┐
│ 4. 실제 요청 분산                                         │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ Request 1 → round_robin(0) → 10.244.0.2 (Pod A)          │
│ Request 2 → round_robin(1) → 10.244.0.3 (Pod B)          │
│ Request 3 → round_robin(2) → 10.244.0.4 (Pod C)          │
│ Request 4 → round_robin(0) → 10.244.0.2 (Pod A)          │
│ Request 5 → round_robin(1) → 10.244.0.3 (Pod B)          │
│ ...                                                      │
│                                                          │
└──────────────────────────────────────────────────────────┘

부하 분산 결과:
Pod A: 33% (33 requests)
Pod B: 33% (33 requests)
Pod C: 34% (34 requests)
```

## 💻 실전 실험

### 실험 1: L4 vs L7 로드밸런싱 성능 비교

```java
@SpringBootTest
public class LoadBalancingComparisonTest {
    
    private final OrderServiceGrpc.OrderServiceBlockingStub l4LbStub;
    private final OrderServiceGrpc.OrderServiceBlockingStub l7LbStub;
    private final OrderServiceGrpc.OrderServiceBlockingStub clientSideLbStub;
    
    @Test
    public void compareLoadBalancing() throws Exception {
        int requestCount = 1000;
        Map<String, Integer> l4Distribution = new ConcurrentHashMap<>();
        Map<String, Integer> l7Distribution = new ConcurrentHashMap<>();
        Map<String, Integer> clientLbDistribution = new ConcurrentHashMap<>();
        
        // L4 로드밸런서 (TCP): 모든 요청이 하나의 서버로
        for (int i = 0; i < requestCount; i++) {
            try {
                GetOrderRequest req = GetOrderRequest.newBuilder()
                    .setOrderId("order-" + i)
                    .build();
                
                OrderResponse resp = l4LbStub.getOrder(req);
                String serverId = resp.getServerId();
                l4Distribution.merge(serverId, 1, Integer::sum);
            } catch (StatusRuntimeException e) {
                // ignore
            }
        }
        
        // L7 로드밸런서 (Envoy): 요청이 여러 서버에 분산
        for (int i = 0; i < requestCount; i++) {
            try {
                GetOrderRequest req = GetOrderRequest.newBuilder()
                    .setOrderId("order-" + i)
                    .build();
                
                OrderResponse resp = l7LbStub.getOrder(req);
                String serverId = resp.getServerId();
                l7Distribution.merge(serverId, 1, Integer::sum);
            } catch (StatusRuntimeException e) {
                // ignore
            }
        }
        
        // 클라이언트 사이드 로드밸런싱: 요청이 여러 서버에 분산
        for (int i = 0; i < requestCount; i++) {
            try {
                GetOrderRequest req = GetOrderRequest.newBuilder()
                    .setOrderId("order-" + i)
                    .build();
                
                OrderResponse resp = clientSideLbStub.getOrder(req);
                String serverId = resp.getServerId();
                clientLbDistribution.merge(serverId, 1, Integer::sum);
            } catch (StatusRuntimeException e) {
                // ignore
            }
        }
        
        System.out.println("L4 Distribution: " + l4Distribution);
        // 출력 예: {server-a: 1000, server-b: 0, server-c: 0}
        
        System.out.println("L7 Distribution: " + l7Distribution);
        // 출력 예: {server-a: 333, server-b: 333, server-c: 334}
        
        System.out.println("Client-side Distribution: " + clientLbDistribution);
        // 출력 예: {server-a: 333, server-b: 334, server-c: 333}
        
        // L4는 한 서버에만, L7과 클라이언트 사이드는 균등 분산
        assertEquals(1000, l4Distribution.values().stream().mapToInt(Integer::intValue).sum());
        assertEquals(1, l4Distribution.size());  // 1개 서버만
        
        assertEquals(1000, l7Distribution.values().stream().mapToInt(Integer::intValue).sum());
        assertTrue(l7Distribution.size() >= 3);  // 3개 서버 모두 사용
        
        assertEquals(1000, clientLbDistribution.values().stream().mapToInt(Integer::intValue).sum());
        assertTrue(clientLbDistribution.size() >= 3);  // 3개 서버 모두 사용
    }
}
```

### 실험 2: 클라이언트 사이드 로드밸런싱 검증

```java
@SpringBootTest
public class ClientSideLbTest {
    
    @Test
    public void testDnsBasedLoadBalancing() throws Exception {
        // Kubernetes headless service 사용
        ManagedChannel channel = ManagedChannelBuilder
            .forTarget("dns:///order-service.default.svc.cluster.local")
            .usePlaintext()
            .defaultLoadBalancingPolicy("round_robin")
            .build();
        
        OrderServiceGrpc.OrderServiceBlockingStub stub = 
            OrderServiceGrpc.newBlockingStub(channel);
        
        Map<String, Integer> distribution = new ConcurrentHashMap<>();
        
        // 300개 요청 분산 (3개 서버 × 100)
        for (int i = 0; i < 300; i++) {
            GetOrderRequest req = GetOrderRequest.newBuilder()
                .setOrderId("order-" + i)
                .build();
            
            OrderResponse resp = stub.getOrder(req);
            String serverId = resp.getServerId();
            distribution.merge(serverId, 1, Integer::sum);
            
            // 진행 상황 출력
            if ((i + 1) % 100 == 0) {
                System.out.println("Progress: " + (i + 1) + "/300, Distribution so far: " + distribution);
            }
        }
        
        System.out.println("Final Distribution: " + distribution);
        
        // 검증: 각 서버가 대략 100개씩 받아야 함 (±20의 오차 범위)
        for (String serverId : distribution.keySet()) {
            int count = distribution.get(serverId);
            assertTrue(count > 80 && count < 120, 
                "Server " + serverId + " received " + count + " requests, expected ~100");
        }
        
        channel.shutdown();
    }
}
```

### 실험 3: 로드밸런싱 정책 비교 (round_robin vs least_request)

```java
@SpringBootTest
public class LbPolicyComparisonTest {
    
    @Test
    public void compareLbPolicies() throws Exception {
        // round_robin 정책
        ManagedChannel rrChannel = ManagedChannelBuilder
            .forTarget("dns:///order-service.default.svc.cluster.local")
            .usePlaintext()
            .defaultLoadBalancingPolicy("round_robin")
            .build();
        
        // least_request 정책
        ManagedChannel lrChannel = ManagedChannelBuilder
            .forTarget("dns:///order-service.default.svc.cluster.local")
            .usePlaintext()
            .defaultLoadBalancingPolicy("least_request")
            .build();
        
        OrderServiceGrpc.OrderServiceBlockingStub rrStub = 
            OrderServiceGrpc.newBlockingStub(rrChannel);
        
        OrderServiceGrpc.OrderServiceBlockingStub lrStub = 
            OrderServiceGrpc.newBlockingStub(lrChannel);
        
        Map<String, Integer> rrDistribution = new ConcurrentHashMap<>();
        Map<String, Integer> lrDistribution = new ConcurrentHashMap<>();
        
        // round_robin: 일정한 순서로 분산
        for (int i = 0; i < 300; i++) {
            GetOrderRequest req = GetOrderRequest.newBuilder()
                .setOrderId("order-" + i)
                .build();
            
            OrderResponse resp = rrStub.getOrder(req);
            rrDistribution.merge(resp.getServerId(), 1, Integer::sum);
        }
        
        // least_request: 가장 적은 요청을 받은 서버로 분산
        for (int i = 0; i < 300; i++) {
            GetOrderRequest req = GetOrderRequest.newBuilder()
                .setOrderId("order-" + i)
                .build();
            
            OrderResponse resp = lrStub.getOrder(req);
            lrDistribution.merge(resp.getServerId(), 1, Integer::sum);
        }
        
        System.out.println("Round-robin Distribution: " + rrDistribution);
        System.out.println("Least-request Distribution: " + lrDistribution);
        
        // round_robin은 순차적으로 균등 분산
        // least_request는 진행 중인 요청 수가 적은 서버로 분산
    }
}
```

## 📊 성능 비교

### L4 vs L7 vs 클라이언트 사이드 로드밸런싱

| 지표 | L4 로드밸런서 | L7 로드밸런서 (Envoy) | 클라이언트 사이드 |
|------|-------------|------------------|------------|
| **분산 단위** | TCP 연결 | HTTP/2 스트림 | HTTP/2 스트림 |
| **Server A 부하** | 100% | 33% | 33% |
| **Server B 부하** | 0% | 33% | 33% |
| **Server C 부하** | 0% | 34% | 34% |
| **전체 처리량** | 1000 req/s | 3000 req/s | 3000 req/s |
| **프록시 필요** | 불필요 | 필수 | 불필요 |
| **지연 (Latency)** | 낮음 | 중간 (프록시 오버헤드) | 매우 낮음 |
| **수평 확장성** | 없음 | 있음 | 있음 |
| **장애 감지** | LB 의존 | LB 의존 | 클라이언트 자동 감지 |
| **복잡도** | 낮음 | 높음 | 중간 |

### 실제 부하 분산 결과 (3개 서버)

```
1000 요청 기준:

L4 로드밸런서:
Server A: ████████████████████ 1000개 (100%)
Server B: (0%)
Server C: (0%)

L7 로드밸런서:
Server A: ██████ 333개 (33%)
Server B: ██████ 333개 (33%)
Server C: ██████ 334개 (34%)

클라이언트 사이드:
Server A: ██████ 333개 (33%)
Server B: ██████ 334개 (34%)
Server C: ██████ 333개 (33%)
```

## ⚖️ 트레이드오프

| 항목 | L4 | L7 (Envoy) | 클라이언트 사이드 |
|------|-------|----------|------------|
| **아키텍처 복잡도** | 낮음 | 높음 | 중간 |
| **운영 비용** | 낮음 | 중간 (Envoy 관리) | 낮음 |
| **부하 분산 효율** | 나쁨 | 우수 | 우수 |
| **지연시간** | 낮음 | 높음 (프록시) | 가장 낮음 |
| **장애 복구** | 느림 | 빠름 | 즉시 |
| **모니터링 난이도** | 쉬움 | 중간 | 어려움 |
| **클라이언트 수정** | 없음 | 없음 | 필요 |
| **기존 인프라 호환** | 좋음 | 별도 구축 | Kubernetes 필요 |

**권장:**
- **기존 TCP LB만 있음**: → L7 Envoy 추가 필요
- **Kubernetes 환경**: → 클라이언트 사이드 (가장 효율적)
- **멀티 클라우드**: → L7 로드밸런서 (일관성)
- **극저지연 필요**: → 클라이언트 사이드 (프록시 없음)

## 📌 핵심 정리

1. **HTTP/2의 단일 연결 특성**: gRPC는 HTTP/2를 사용하여 한 번 연결된 TCP 소켓을 재사용합니다. 따라서 L4 로드밸런서가 연결을 특정 서버에 할당하면, 모든 요청이 그 서버로 가게 됩니다.

2. **L7 로드밸런싱의 필요성**: HTTP/2 스트림 단위로 요청을 분산하려면 L7 로드밸런서(Envoy, Nginx)가 필수입니다. 이들은 HTTP/2 프로토콜을 이해하고 개별 스트림을 다른 백엔드로 라우팅할 수 있습니다.

3. **클라이언트 사이드 로드밸런싱의 효율성**: 클라이언트가 여러 서버 주소를 알고 직접 분산하면, 프록시 없이도 완벽한 부하 분산이 가능합니다. Kubernetes headless service와 DNS를 사용하면 자동 스케일링까지 지원합니다.

4. **수평 확장의 실현**: L7 또는 클라이언트 사이드 로드밸런싱을 사용하면, 서버를 추가할 때마다 처리량이 선형으로 증가합니다. 3개 서버는 1개 서버의 3배 처리량을 제공합니다.

5. **지연시간 vs 복잡도**: 프록시 없는 클라이언트 사이드가 지연이 가장 낮지만, Kubernetes 환경을 요구합니다. L7 로드밸런서는 약간의 지연이 있지만 기존 인프라에 쉽게 통합됩니다.

## 🤔 생각해볼 문제

1. **Sticky Connection 필요성**: 일부 애플리케이션은 같은 클라이언트의 연속 요청이 같은 서버로 가야 합니다(상태 유지). 그런데 L7 또는 클라이언트 사이드 로드밸런싱에서 round_robin을 사용하면 요청마다 다른 서버로 갑니다. 이 문제를 어떻게 해결할 수 있을까요?

2. **동적 서버 추가/제거**: Kubernetes 환경에서 Pod이 추가되면 DNS가 자동으로 업데이트되고, 클라이언트가 주기적으로 DNS를 다시 조회합니다. 그런데 이 중간에 일부 요청은 삭제된 Pod으로 갈 가능성이 있습니다. 이를 어떻게 감지하고 처리할까요?

3. **캐시 지역성(Affinity)**: 만약 서버가 각각 다른 데이터를 캐싱하고 있다면, 요청을 항상 같은 서버로 보내는 것이 효율적입니다. 그런데 무조건 round_robin으로 분산하면 캐시 효율이 떨어집니다. 이를 어떻게 해결할까요?

---

**[⬅️ 이전: Deadline과 Timeout](./04-deadline-timeout.md)** | **[홈으로 🏠](../README.md)** | **[다음: Buf Schema Registry ➡️](./06-buf-schema-registry.md)**
