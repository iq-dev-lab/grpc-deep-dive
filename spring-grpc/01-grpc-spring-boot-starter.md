# grpc-spring-boot-starter 설정

---

## 🎯 핵심 질문

- @GrpcService와 @GrpcClient 어노테이션은 내부적으로 어떻게 동작하는가?
- application.yml에서 gRPC 서버/클라이언트를 어떻게 설정하는가?
- grpc.server.port와 server.port를 동시에 열면 어떻게 되는가?
- keepAlive 설정은 어떤 경우에 필요하고 어떻게 설정하는가?
- Spring Boot Actuator와 gRPC Health Indicator를 연동하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

grpc-spring-boot-starter는 gRPC와 Spring Boot의 진입점입니다. 올바른 설정이 없으면 채널 연결 실패, keepAlive 타임아웃, 또는 대량의 "Connection refused" 에러를 마주합니다. 실무 환경의 쿠버네티스, AWS ALB, 프록시 환경에서 안정적으로 동작하려면 이 설정들의 의미를 명확히 알아야 합니다.

---

## 😱 흔한 실수 (Before)

```java
// 실수 1: @GrpcClient의 주소를 http 형식처럼 지정
@GrpcClient("http://localhost:9090")  // ❌ 틀림
private UserServiceGrpc.UserServiceBlockingStub userService;

// 실수 2: ManagedChannel을 Spring Bean으로 직접 정의
@Bean
public ManagedChannel managedChannel() {
    return ManagedChannelBuilder.forAddress("localhost", 9090)
            .usePlaintext()
            .build();  // ❌ grpc.client.* 설정 무시됨, 정리 안 됨
}

// 실수 3: 테스트 환경에서 고정 포트 사용
grpc:
  server:
    port: 9090  # ❌ 병렬 테스트 포트 충돌

// 실수 4: keepAlive 미설정
grpc:
  server:
    enable-keep-alive: false  # ❌ ALB 60초 후 종료
```

---

## ✨ 올바른 접근 (After)

```yaml
grpc:
  server:
    port: 9090
    enable-keep-alive: true
    keep-alive-time: 30s
    keep-alive-timeout: 10s
    keep-alive-without-calls: true
    max-inbound-message-size: 4194304
    max-metadata-size: 8192

  client:
    user-service:
      address: 'dns:///user-service:9090'
      enable-keep-alive: true
      keep-alive-time: 30s
      keep-alive-without-calls: true
      negotiation-type: plaintext

management:
  endpoints:
    web:
      exposure:
        include: health,metrics
  health:
    grpc:
      enabled: true
```

```java
@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    @Override
    public void getUser(GetUserRequest request,
            StreamObserver<GetUserResponse> responseObserver) {
        try {
            GetUserResponse response = GetUserResponse.newBuilder()
                    .setId(request.getId())
                    .setName("John Doe")
                    .build();
            responseObserver.onNext(response);
            responseObserver.onCompleted();
        } catch (Exception e) {
            responseObserver.onError(e);
        }
    }
}

@Service
public class UserClient {
    @GrpcClient("user-service")
    private UserServiceGrpc.UserServiceBlockingStub stub;
    
    public User getUser(String id) {
        return stub.getUser(GetUserRequest.newBuilder()
                .setId(id)
                .build());
    }
}
```

---

## 🔬 내부 동작 원리

### 1. @GrpcService 자동 등록 메커니즘

```
┌──────────────────────────────────┐
│  Spring Boot Application Start   │
└────────────┬─────────────────────┘
             │
             ▼
      ┌─────────────┐
      │ ClassPath   │
      │ Scan        │
      │ @GrpcService│
      └──────┬──────┘
             │
         ┌───┴───┐
         ▼       ▼
    ┌────────┐ ┌────────┐
    │ User   │ │ Order  │
    │Service │ │Service │
    │Impl    │ │Impl    │
    └────┬───┘ └───┬────┘
         │         │
         └────┬────┘
              ▼
    ┌──────────────────┐
    │ GrpcServiceRegis │
    │ try (Map)        │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ bindService()    │
    │ 호출             │
    └────────┬─────────┘
             │
             ▼
    ┌──────────────────┐
    │ Server listening │
    │ port 9090        │
    └──────────────────┘
```

**작동 원리:**
1. Spring Boot 시작 시 classpath 스캔
2. @GrpcService 어노테이션 찾아 자동 등록
3. GrpcServiceRegistry에 저장
4. io.grpc.Server의 bindService() 호출
5. grpc.server.port에서 리스닝

```java
@Configuration
public class GrpcServerConfiguration {
    @Bean
    public GrpcServerConfigurer grpcServerConfigurer(
            List<GrpcServiceDefinition> grpcServiceDefinitions) {
        return serverBuilder -> {
            grpcServiceDefinitions.forEach(definition -> {
                serverBuilder.addService(
                    definition.getInterceptors(), 
                    definition.getServiceDefinition()
                );
            });
        };
    }

    @Bean
    public Server grpcServer(GrpcServerConfigurer configurer) {
        ServerBuilder<?> serverBuilder = 
            NettyServerBuilder.forPort(9090);
        configurer.configure(serverBuilder);
        return serverBuilder.build().start();
    }
}
```

### 2. @GrpcClient 채널 생성 과정

```
application.yml:
├─ grpc.client.user-service.address
├─ grpc.client.user-service.keep-alive-time
└─ grpc.client.user-service.negotiation-type
         │
         ▼
┌──────────────────────────────────┐
│ GrpcClientBeanPostProcessor      │
│ @GrpcClient 어노테이션 감지      │
└───────────┬──────────────────────┘
            │
            ▼
    ┌────────────────────┐
    │ grpc.client.* 설정 │
    │ 로드               │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ ManagedChannelBuilder
    │ 생성               │
    ├─ address 설정      │
    ├─ negotiation 설정  │
    ├─ interceptor 추가  │
    ├─ keepAlive 설정    │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ ManagedChannel     │
    │ 빌드               │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ Stub 생성          │
    │ UserServiceStub    │
    └────────┬───────────┘
             │
             ▼
    ┌────────────────────┐
    │ Spring Bean으로    │
    │ @GrpcClient 필드에 │
    │ 주입               │
    └────────────────────┘
```

**작동 원리:**
1. BeanPostProcessor가 @GrpcClient 감지
2. grpc.client.* 설정에서 구성 로드
3. ManagedChannelBuilder 생성 및 설정 적용
4. Interceptor 추가
5. ManagedChannel 빌드
6. Stub 생성 및 주입

```java
@Configuration
public class GrpcClientConfiguration {
    @Bean
    public ManagedChannel userServiceChannel(
            GrpcClientConfigProperties properties) {
        
        GrpcClientProperties clientProps = 
            properties.getClient().get("user-service");
        
        ManagedChannelBuilder<?> channelBuilder =
            ManagedChannelBuilder
                .forTarget(clientProps.getAddress())
                .defaultLoadBalancingPolicy("round_robin");
        
        List<ClientInterceptor> interceptors = new ArrayList<>();
        interceptors.add(new LoggingInterceptor());
        interceptors.add(new MetricsInterceptor());
        
        channelBuilder.intercept(interceptors);
        
        if (clientProps.isEnableKeepAlive()) {
            NettyChannelBuilder nettyBuilder = 
                (NettyChannelBuilder) channelBuilder;
            nettyBuilder.keepAliveTime(30, TimeUnit.SECONDS)
                       .keepAliveTimeout(10, TimeUnit.SECONDS)
                       .keepAliveWithoutCalls(true);
        }
        
        return channelBuilder.build();
    }

    @Bean
    public UserServiceGrpc.UserServiceBlockingStub 
            userServiceStub(ManagedChannel channel) {
        return UserServiceGrpc.newBlockingStub(channel);
    }
}
```

### 3. 주요 설정 항목 상세 설명

```
grpc.server.port (기본값: 9090)
├─ 클라이언트가 연결할 gRPC 서버의 포트
├─ 값: 1~65535 정수
├─ 특수값 0: 사용 가능한 포트 자동 할당 (테스트용)
└─ server.port와 별개 (REST API는 server.port)

grpc.server.enable-keep-alive (기본값: false)
├─ HTTP/2 PING 프레임으로 연결 유지 여부
├─ true: ALB/Nginx/방화벽 60초 타임아웃 방지
└─ false: 유휴 연결이 종료될 수 있음 (위험)

grpc.server.keep-alive-time (기본값: 2h)
├─ PING 프레임 전송 주기 (30s, 5m, 2h 등)
├─ ALB: 60초 → 30~50초 권장
├─ 너무 짧으면: CPU/네트워크 트래픽 증가
└─ 너무 길면: 유휴 연결 종료 위험

grpc.server.keep-alive-timeout (기본값: 20s)
├─ PONG 응답 대기 시간
├─ PING → [대기 keep-alive-timeout] → PONG
├─ 시간 초과: 연결 강제 종료
└─ 권장값: keep-alive-time의 1/3

grpc.server.keep-alive-without-calls (기본값: false)
├─ 활성 요청이 없어도 PING 전송 여부
├─ true: 항상 주기적으로 PING 전송 (권장)
└─ false: 요청 처리 중일 때만 연결 유지

grpc.server.max-inbound-message-size (기본값: 4MB)
├─ 수신 메시지 최대 크기 (바이트)
├─ 초과 시: Status.RESOURCE_EXHAUSTED 반환
├─ 대용량 파일 업로드: 크기 증가 필요
└─ 예: 100MB → 104857600 (100*1024*1024)
```

### 4. Health Indicator 연동 (Spring Boot Actuator + gRPC)

```
HTTP GET /actuator/health
         │
         ▼
┌──────────────────────────────┐
│ CompositeHealthContributor   │
│ (모든 HealthIndicator 합산)  │
└────┬────────────┬─────────────┘
     │            │
     ▼            ▼
┌─────────┐  ┌──────────────┐
│ Database │  │ gRPC Server  │
│ Health   │  │ Health Check │
└─────────┘  └──────┬───────┘
                    │
         ┌──────────┴──────────┐
         ▼                     ▼
   ┌──────────────┐   ┌──────────────┐
   │ Check if     │   │ Report final │
   │ Running      │   │ status       │
   └──────────────┘   └──────────────┘
                           │
                           ▼
          ┌────────────────────────────┐
          │ HTTP 200 Response:         │
          │ {                          │
          │   "status": "UP",          │
          │   "components": {          │
          │     "grpcServer": {        │
          │       "status": "UP",      │
          │       "details": {         │
          │         "port": 9090       │
          │       }                    │
          │     },                     │
          │     "db": {                │
          │       "status": "UP"       │
          │     }                      │
          │   }                        │
          │ }                          │
          └────────────────────────────┘
```

```java
@Component
public class GrpcHealthIndicator 
        extends AbstractHealthIndicator {
    
    @Value("${grpc.server.port:9090}")
    private int grpcPort;
    
    private final Server grpcServer;
    
    public GrpcHealthIndicator(Server grpcServer) {
        this.grpcServer = grpcServer;
    }
    
    @Override
    protected void doHealthCheck(Health.Builder builder) {
        try {
            if (grpcServer == null || grpcServer.isShutdown()) {
                builder.down()
                       .withDetail("port", grpcPort)
                       .withDetail("message", 
                           "gRPC server is not running");
                return;
            }
            
            if (grpcServer.isTerminated()) {
                builder.down()
                       .withDetail("port", grpcPort)
                       .withDetail("message", 
                           "gRPC server is terminated");
                return;
            }
            
            ManagedChannel channel = 
                ManagedChannelBuilder
                    .forAddress("localhost", grpcPort)
                    .usePlaintext()
                    .build();
            
            HealthGrpc.HealthBlockingStub healthStub = 
                HealthGrpc.newBlockingStub(channel);
            
            HealthCheckResponse response = 
                healthStub.check(
                    HealthCheckRequest.newBuilder()
                        .setService("io.grpc.examples.MyService")
                        .build()
                );
            
            if (response.getStatus() == 
                HealthCheckResponse.ServingStatus.SERVING) {
                builder.up()
                       .withDetail("port", grpcPort)
                       .withDetail("services", 3);
            } else {
                builder.outOfService()
                       .withDetail("port", grpcPort);
            }
            
            channel.shutdown();
        } catch (Exception e) {
            builder.down()
                   .withDetail("error", e.getMessage())
                   .withDetail("port", grpcPort);
        }
    }
}
```

**Actuator 설정:**

```yaml
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info
      base-path: /actuator
  
  health:
    grpc:
      enabled: true
    livenessState:
      enabled: true
    readinessState:
      enabled: true
  
  endpoint:
    health:
      show-details: always
```

---

## 💻 실전 실험

```yaml
# application.yml 완전 설정
spring:
  application:
    name: user-service
  profiles:
    active: local

grpc:
  server:
    port: ${GRPC_SERVER_PORT:9090}
    enable-keep-alive: true
    keep-alive-time: 30s
    keep-alive-timeout: 10s
    keep-alive-without-calls: true
    max-inbound-message-size: 4194304
    max-metadata-size: 8192
    max-concurrent-streams: 100
    flow-control-window: 1048576
  
  client:
    order-service:
      address: 'dns:///order-service.default.svc.cluster.local:9090'
      enable-keep-alive: true
      keep-alive-time: 30s
      keep-alive-without-calls: true
      keep-alive-timeout: 10s
      negotiation-type: plaintext
      load-balancing-policy: round_robin
    
    payment-service:
      address: 'static://payment-service:9090'
      enable-keep-alive: true
      keep-alive-time: 60s
      keep-alive-without-calls: false
      negotiation-type: plaintext

management:
  endpoints:
    web:
      exposure:
        include: health,metrics,info,prometheus
      base-path: /actuator
  
  health:
    grpc:
      enabled: true
    livenessState:
      enabled: true
    readinessState:
      enabled: true
  
  endpoint:
    health:
      show-details: always

server:
  port: 8080

logging:
  level:
    io.grpc: DEBUG
```

```bash
# Health 엔드포인트 확인
curl http://localhost:8080/actuator/health
# 응답:
# {
#   "status": "UP",
#   "components": {
#     "grpcServer": {
#       "status": "UP",
#       "details": {
#         "port": 9090,
#         "services": 3
#       }
#     }
#   }
# }

# Prometheus 메트릭 확인
curl http://localhost:8080/actuator/metrics

# gRPC 메트릭만 조회
curl http://localhost:8080/actuator/metrics/grpc.server.calls.total

# grpcurl로 서비스 리스트 확인
grpcurl -plaintext localhost:9090 list

# 메서드 설명 조회
grpcurl -plaintext localhost:9090 \
  describe io.grpc.examples.UserService/GetUser

# RPC 호출
grpcurl -plaintext \
  -d '{"id": "user123"}' \
  localhost:9090 \
  io.grpc.examples.UserService/GetUser
```

---

## 📊 성능/비용 비교

```
┌───────────────────────────────────────────────┐
│   REST (HTTP/1.1) vs gRPC (HTTP/2) 비교      │
├───────────────────────────────────────────────┤
│                                                │
│  메트릭            REST         gRPC          │
│  ──────────────────────────────────────────   │
│  포트 수           1개          1개           │
│  CPU 사용률        100%         20~30%        │
│  메모리(1k conn)   500MB        100MB         │
│  평균 응답시간     50ms         10ms          │
│  처리량            500 req/s    5000 req/s    │
│  메시지 크기       2KB (JSON)   0.5KB (Proto) │
│                                                │
│ gRPC 이점:                                    │
│ • HTTP/2 다중화 → 연결 수 감소               │
│ • Protobuf → 직렬화 오버헤드 감소            │
│ • 높은 처리량, 낮은 지연시간                 │
└───────────────────────────────────────────────┘

┌───────────────────────────────────────────────┐
│  설정에 따른 메모리 영향                     │
├───────────────────────────────────────────────┤
│                                                │
│  설정                      메모리 증가        │
│  ──────────────────────────────────────────   │
│  기본 설정                 기준               │
│  + max-inbound-message    +1MB당 +1MB        │
│    100MB 설정             +100MB              │
│  + max-concurrent-streams +50~100MB          │
│    1000으로 설정                             │
│  + Channel Pool 10개      +50MB               │
│                                                │
│  → 메모리 예산에 맞춰 조정 필요              │
└───────────────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
┌────────────────────────────────────────────────┐
│  keep-alive-time (PING 주기) 트레이드오프     │
├────────────────────────────────────────────────┤
│                                                 │
│  짧게 설정 (10초)                              │
│  ✅ 장점: 연결 끊김 빠르게 감지               │
│  ❌ 단점: CPU 증가, 네트워크 트래픽 증가     │
│                                                 │
│  길게 설정 (300초)                             │
│  ✅ 장점: 오버헤드 감소                       │
│  ❌ 단점: 끊긴 연결 감지 늦음                │
│                                                 │
│  권장값: 30~60초 (실제 프록시 타임아웃)     │
│                                                 │
├────────────────────────────────────────────────┤
│                                                 │
│  max-inbound-message-size 트레이드오프       │
├────────────────────────────────────────────────┤
│                                                 │
│  작게 설정 (1MB)                               │
│  ✅ 장점: DoS 공격 방어, 메모리 효율         │
│  ❌ 단점: 대용량 파일 업로드 불가            │
│                                                 │
│  크게 설정 (1GB)                               │
│  ✅ 장점: 대용량 메시지 처리 가능            │
│  ❌ 단점: 악의적 메시지로 메모리 소진       │
│                                                 │
│  권장값: 실제 메시지 크기 + 10~20% 여유     │
│                                                 │
├────────────────────────────────────────────────┤
│                                                 │
│  max-concurrent-streams 트레이드오프        │
├────────────────────────────────────────────────┤
│                                                 │
│  낮게 설정 (10)                                │
│  ✅ 장점: 메모리 절감                         │
│  ❌ 단점: 동시 요청 처리 불가 → 큐 대기    │
│                                                 │
│  높게 설정 (1000)                              │
│  ✅ 장점: 동시성 높음                         │
│  ❌ 단점: 메모리 증가, 과부하 위험           │
│                                                 │
│  권장값: 예상 동시 요청수 * 1.5              │
└────────────────────────────────────────────────┘
```

---

## 📌 핵심 정리

```
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 설정 원칙 (Configuration Hierarchy)
   
   application.yml (grpc.server/client.*)
          ↓
   GrpcServerConfigurer/ClientInterceptor
          ↓
   ManagedChannel / Server 생성
   
   → 설정 파일에서 모든 것을 제어
   → Bean 직접 생성은 자동화와 충돌

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

2. keepAlive 필수 설정 (클라우드/프록시 환경)
   
   enable-keep-alive: true
   keep-alive-time: 30s (ALB 60초 < 30초)
   keep-alive-without-calls: true
   keep-alive-timeout: 10s
   
   → ALB/Nginx/방화벽의 유휴 연결 종료 방지

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

3. 포트 격리 (gRPC ≠ REST API)
   
   grpc.server.port: 9090 (gRPC)
   server.port: 8080 (REST API, Actuator)
   
   → 포트 충돌 방지, 방화벽 규칙 명확화

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

4. 클라이언트 주소 형식
   
   DNS 기반: dns:///service-name:9090 (권장)
   정적 주소: static://host:9090
   축약형: service-name:9090
   ❌ HTTP: http://localhost:9090 (불가)
   
   → DNS는 로드밸런싱, 서비스 디스커버리 지원

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

5. 테스트 설정
   
   port: 0 → 랜덤 포트 할당
   enable-keep-alive: false (테스트 불필요)
   
   → 병렬 테스트 실행 가능, 포트 충돌 해결

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

6. Health Check 통합
   
   Actuator /health 엔드포인트로 모니터링
   쿠버네티스 readinessProbe/livenessProbe 연동
   
   → 자동 롤링 업데이트, 자가 치유 가능

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
```

---

## 🤔 생각해볼 문제

### Q1: grpc.server.port와 server.port를 동일하게 설정하면 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

**정답: 포트 바인딩 실패 (Address already in use)**

gRPC와 REST API는 서로 다른 프로토콜로 동일 포트에 공존할 수 없습니다.

```
server.port: 8080  (Spring Web - HTTP/1.1)
grpc.server.port: 8080  (gRPC - HTTP/2)
```

Spring Boot 시작 시:
1. Tomcat이 8080 포트에 바인딩
2. gRPC 서버가 같은 8080에 바인딩 시도
3. **"Address already in use" 예외 발생**
4. 애플리케이션 시작 실패

**해결책:**
- 다른 포트 사용: server.port: 8080, grpc.server.port: 9090
- 또는 gRPC 서버만 필요 시 Tomcat 비활성화

```yaml
spring:
  main:
    web-application-type: none  # Tomcat 비활성화
```

</details>

---

### Q2: @GrpcClient 주소를 `http://localhost:9090`으로 설정하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

**정답: 채널 생성 실패 또는 연결 불가 (UNAVAILABLE)**

gRPC는 URI 스키마를 다르게 해석합니다.

```java
@GrpcClient("http://localhost:9090")  // ❌ 틀림
// → ManagedChannelBuilder가 
//   "http://localhost:9090" 전체를 호스트명으로 인식
// → 실제로는 "http://localhost:9090" 라는 호스트 찾음
// → DNS 조회 실패
```

**gRPC 주소 형식:**
- `dns:///service-name:9090` (DNS 기반, 권장)
- `static://localhost:9090` (정적 주소)
- `localhost:9090` (축약형, 기본값 static://)

**올바른 설정:**
```yaml
grpc:
  client:
    user-service:
      address: 'dns:///user-service:9090'
```

</details>

---

### Q3: keepAliveTime을 2초로 설정하면 어떤 문제가 발생하는가?

<details>
<summary>해설 보기</summary>

**정답: 서버가 too_many_pings 에러로 연결 강제 종료**

gRPC 서버는 과도한 PING을 방어하기 위해 속도 제한을 설정합니다.

```
┌────────────┐                    ┌────────────┐
│  gRPC      │   PING (2s)        │  gRPC      │
│  Client    │──────────────────▶ │  Server    │
│            │                    │            │
│            │ ◀─────────────────  │  (받음)    │
│            │    PONG (2s)       │            │
│            │                    │            │
│            │   PING (2s)        │            │
│            │──────────────────▶ │            │
│            │                    │  too_many  │
│            │ ◀─────────────────  │  _pings!  │
│            │  GOAWAY frame      │  연결 종료 │
└────────────┘                    └────────────┘
```

gRPC 서버의 기본 PING 정책:
- 최소 5분마다 1개 PING만 허용
- 또는 초당 최대 2개 PING만 허용

**결과:**
```
io.grpc.StatusRuntimeException: UNAVAILABLE: HTTP status code: 503
Reason: GOAWAY received
Description: too_many_pings
```

**해결책:**
```yaml
grpc:
  server:
    keep-alive-time: 30s  # 최소 필요
  client:
    user-service:
      keep-alive-time: 30s
```

**서버 커스텀 PING 정책 설정:**
```java
@Bean
public GrpcServerConfigurer grpcServerConfigurer() {
    return serverBuilder -> {
        if (serverBuilder instanceof NettyServerBuilder) {
            ((NettyServerBuilder) serverBuilder)
                .permitKeepAliveWithoutCalls(true)
                .permitKeepAliveTime(10, TimeUnit.SECONDS);
        }
    };
}
```

</details>

---

**[⬅️ 이전: 채널 보안 설정](../security-auth/05-channel-security.md)** | **[홈으로 🏠](../README.md)** | **[다음: Spring Security 통합 ➡️](./02-spring-security.md)**
