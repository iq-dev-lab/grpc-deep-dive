# 연결 관리 튜닝 — keepAlive와 Channel Pool

---

## 🎯 핵심 질문

- keepAliveTime과 keepAliveTimeout은 각각 무엇을 제어하는가?
- AWS ALB, Nginx, 방화벽이 유휴 gRPC 연결을 끊는 이유와 해결책은?
- maxConnectionAge로 연결을 주기적으로 리셋하면 무슨 이점이 있는가?
- Channel Pool이 필요한 경우는 언제이고 어떻게 구성하는가?
- keepAliveWithoutCalls 설정이 필요한 상황은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

AWS ALB, Nginx 같은 프록시는 기본적으로 60초 동안 트래픽이 없으면 연결을 종료합니다. gRPC를 이 환경에서 사용하면 keepAlive를 설정하지 않으면 계속 "Connection refused" 에러를 마주합니다.

---

## 😱 흔한 실수 (Before)

```yaml
# 실수 1: keepAlive 미설정
grpc:
  server:
    enable-keep-alive: false  # ❌ ALB 60초 후 연결 종료

# 실수 2: maxConnectionAge 미설정
# → 연결이 영원히 유지됨
# → 부하 분산 불균형

# 실수 3: Channel Pool 없이 단일 연결
stub = UserServiceGrpc.newBlockingStub(channel)
# → 1개 연결로 모든 요청 처리 (병목)
```

---

## ✨ 올바른 접근 (After)

```yaml
grpc:
  server:
    port: 9090
    enable-keep-alive: true
    keep-alive-time: 30s        # ALB 60초보다 짧게
    keep-alive-timeout: 10s     # PONG 대기시간
    keep-alive-without-calls: true  # 유휴 시에도 PING
    max-connection-age: 5m      # 5분마다 연결 리셋
    max-connection-idle: 2m     # 2분 유휴 후 종료

  client:
    user-service:
      address: 'dns:///user-service:9090'
      enable-keep-alive: true
      keep-alive-time: 30s
      keep-alive-timeout: 10s
      keep-alive-without-calls: true
      load-balancing-policy: round_robin
```

```java
// Channel Pool 구성
@Configuration
public class GrpcChannelPoolConfiguration {
    
    @Bean
    public ManagedChannelBuilder<?> userServiceBuilder() {
        return ManagedChannelBuilder
            .forTarget("dns:///user-service:9090")
            .defaultLoadBalancingPolicy("round_robin");
    }
    
    @Bean
    public List<ManagedChannel> userServiceChannels(
            ManagedChannelBuilder<?> builder) {
        
        List<ManagedChannel> channels = new ArrayList<>();
        
        for (int i = 0; i < 5; i++) {  // 5개 연결
            channels.add(
                builder.build()
            );
        }
        
        return channels;
    }
    
    @Bean
    public UserServiceGrpc.UserServiceBlockingStub 
            userServiceStub(
            List<ManagedChannel> channels) {
        
        // 라운드로빈으로 채널 선택
        return new PooledBlockingStub(channels);
    }
}

// Channel Pool Wrapper
public class PooledBlockingStub {
    
    private final List<ManagedChannel> channels;
    private final AtomicInteger counter = 
        new AtomicInteger(0);
    
    public PooledBlockingStub(List<ManagedChannel> channels) {
        this.channels = channels;
    }
    
    public UserServiceGrpc.UserServiceBlockingStub 
            getNextStub() {
        
        ManagedChannel channel = 
            channels.get(
                counter.getAndIncrement() % channels.size()
            );
        
        return UserServiceGrpc.newBlockingStub(channel);
    }
    
    public GetUserResponse getUser(GetUserRequest request) {
        return getNextStub().getUser(request);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. keepAlive PING/PONG 메커니즘

```
Client                      Server
  │                           │
  │───────── PING ───────────▶│ (30초 주기)
  │                     (HTTP/2 특수 프레임)
  │◀────────── PONG ──────────│ (즉시)
  │                           │
  │         (PING/PONG 반복)  │
  │         ↓                  │
  │ 30초 주기로 PING 전송      │
  │ 10초 내에 PONG 받음        │
  │ → 연결 유지됨              │
  │                           │
  │ (요청 없어도 진행)         │
  │ keep-alive-without-calls  │
  │ true일 때만               │
  │                           │
  │                           │

타임아웃 시나리오:
  │───────── PING ───────────▶│
  │                           │ (응답 안 함)
  │                   [10s 대기]
  │◀── RST_STREAM ─────────────│
  │    (GOAWAY)              │
  │                           │
  └─ 연결 강제 종료 ──────────┘
```

### 2. 방화벽/프록시의 유휴 연결 종료와 keepAlive 대응

```
AWS ALB 타임아웃 정책 (기본 60초):

방화벽/ALB 없음:
┌──────────┐        ┌──────────┐
│  Client  │───────│ Server   │
│ (유휴)   │       │ (유휴)   │
└──────────┘        └──────────┘
연결 유지됨, 데이터 전송 없음

ALB 있음 + keepAlive 없음:
┌──────────┐    ┌─────┐        ┌──────────┐
│  Client  │───│ ALB │────────│ Server   │
│ (유휴)   │    └─────┘        │          │
└──────────┘        ↓           └──────────┘
         60초 후 ALB가
         연결 강제 종료
         ↓
    ❌ ECONNRESET

ALB 있음 + keepAlive 있음:
┌──────────┐    ┌─────┐        ┌──────────┐
│  Client  │───│ ALB │────────│ Server   │
│ (30초마다 PING) │        │
└──────────┘    └─────┘        └──────────┘
        │          통과
        └───PING──▶ (HTTP/2 프레임)
             ✅ 연결 유지됨
```

### 3. maxConnectionAge (주기적 연결 리셋)

```
연결 타임라인:

생성         5분 (maxConnectionAge)      10분
 │──────────────────────│──────────────────
 │                      ▼
 ├─ 새 연결          연결 1 종료
 │                  (우아하게)
 │                  
 │                   10초 유예
 │                   (기존 요청 완료)
 │
 │                   새 연결 생성 (연결 2)
 │
 └──────────────────────────────────

장점:
├─ 로드밸런서 리셋 (새 연결은 다른 인스턴스로)
├─ 메모리 누수 방지
└─ 연결 정보 갱신

타임라인 예시:
시간 0초: 클라이언트가 서버 A에 연결 (conn1)
시간 10초: 요청 1 처리
시간 30초: 요청 2 처리
시간 5분: conn1 종료 신호 (GOAWAY)
시간 5분 10초: 기존 요청 완료 대기
시간 5분 10초: 새 연결 (conn2) → 서버 B (LB 재분배)
```

### 4. Channel Pool 구성 패턴

```
단일 채널:
┌─────────────┐
│ Client Pool │
│  (1개 conn) │
└──────┬──────┘
       │
       ▼
    Server
   처리량: 낮음 (직렬 처리)

Channel Pool:
┌─────────────┐
│ Client Pool │
│  (5개 conn) │
└──┬──┬──┬──┬─┘
   │  │  │  │
   ▼  ▼  ▼  ▼
  Server  (병렬 처리)
  처리량: 높음 (동시 요청)
```

```java
// Channel Pool 구현 패턴
public class ManagedChannelPool {
    
    private final List<ManagedChannel> channels;
    private final AtomicInteger roundRobinCounter;
    private final int poolSize;
    
    public ManagedChannelPool(String target, int poolSize) {
        this.poolSize = poolSize;
        this.channels = new ArrayList<>();
        this.roundRobinCounter = new AtomicInteger(0);
        
        // poolSize개 채널 생성
        for (int i = 0; i < poolSize; i++) {
            ManagedChannel channel = 
                ManagedChannelBuilder
                    .forTarget(target)
                    .defaultLoadBalancingPolicy(
                        "round_robin"
                    )
                    .build();
            channels.add(channel);
        }
    }
    
    public ManagedChannel getChannel() {
        int index = 
            roundRobinCounter.getAndIncrement() % poolSize;
        return channels.get(index);
    }
    
    public void shutdown() {
        channels.forEach(channel -> {
            try {
                channel.shutdownNow()
                    .awaitTermination(10, 
                        TimeUnit.SECONDS);
            } catch (InterruptedException e) {
                Thread.currentThread().interrupt();
            }
        });
    }
}

// 사용 예시
ManagedChannelPool pool = new ManagedChannelPool(
    "dns:///user-service:9090", 5);

for (int i = 0; i < 100; i++) {
    ManagedChannel channel = pool.getChannel();
    UserServiceGrpc.UserServiceBlockingStub stub = 
        UserServiceGrpc.newBlockingStub(channel);
    
    GetUserResponse response = 
        stub.getUser(request);
}

pool.shutdown();
```

---

## 💻 실전 실험

```bash
# 연결 상태 확인
netstat -an | grep 9090
# 또는
ss -an | grep 9090

# tcpdump로 PING/PONG 프레임 확인
tcpdump -i lo -n 'tcp port 9090 and tcp[tcpflags] & tcp-ack != 0'

# gRPC 채널 상태 확인
grpcurl -plaintext localhost:9090 list
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────┐
│ 단일 채널 vs Channel Pool         │
├────────────────────────────────────┤
│                                     │
│ 메트릭         1개 채널  5개 채널  │
│ ────────────────────────────────── │
│ 처리량(req/s)  500      2000      │
│ 지연시간       50ms     12ms      │
│ CPU 사용률     80%      40%       │
│ 메모리         50MB     200MB     │
│                                     │
│ 권장: 동시성 높으면 Pool 사용     │
└────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
keepAlive-time:
├─ 짧게 (10s)
│  ✅ 빠른 감지
│  ❌ CPU/트래픽 증가
├─ 길게 (300s)
│  ✅ 오버헤드 감소
│  ❌ 늦은 감지
└─ 권장: 30~60s

Channel Pool:
├─ 작게 (1개)
│  ✅ 메모리 적음
│  ❌ 처리량 낮음
├─ 크게 (10개)
│  ✅ 처리량 높음
│  ❌ 메모리 증가
└─ 권장: CPU 코어당 1개
```

---

## 📌 핵심 정리

```
1. keepAlive 필수 (ALB/프록시 환경)
   time: 30s, timeout: 10s, without-calls: true

2. maxConnectionAge: 5분 (LB 재분배)

3. Channel Pool: 동시성 높으면 필수
   크기 = CPU 코어 수

4. 모니터링: netstat/ss로 연결 수 확인
```

---

## 🤔 생각해볼 문제

### Q1: 프록시 타임아웃이 60초인데 keepAlive-time을 50초로 설정하면?

<details>
<summary>해설 보기</summary>

**정답: 가능하지만 위험 (여유 10초)**

```
타임라인:
시간 0: 연결 생성
시간 50: PING 1 전송
시간 60: 프록시 가능 종료

프록시가 조금 늦으면:
시간 60.5: 프록시가 RST
시간 50.5: PONG 대기 중

→ 경계에서 불안정

권장: 20초 여유
keep-alive-time: 40s (60 - 20)
```

</details>

---

### Q2: maxConnectionAge를 1분으로 설정하면?

<details>
<summary>해설 보기</summary>

**정답: 높은 오버헤드 + LB 재분배 빈번**

```
매초 100개 요청:
maxConnectionAge: 1분
→ 매분 오래된 연결 교체
→ 새 연결 생성 빈번
→ TCP 핸드셰이크 빈번 (10~20ms)
→ 지연시간 증가

권장:
├─ 트래픽 많으면: 5~10분
├─ 트래픽 적으면: 10~30분
└─ LB 전략에 따라 조정
```

</details>

---

### Q3: Channel Pool 크기를 결정하는 공식은?

<details>
<summary>해설 보기</summary>

**정답: CPU 코어 수 기반**

```
Pool 크기 = CPU 코어 수 * 1~2

이유:
├─ gRPC는 I/O 바운드 (CPU 대기 많음)
├─ 채널당 1~2 동시 요청 처리 가능
└─ 4 코어 → 4~8개 채널

검증 방법:
1. CPU 사용률 확인
2. 메모리 사용 확인 (1 채널 = ~1MB)
3. 응답 시간 확인

감소:
├─ CPU > 80% → Pool 크기 감소
├─ 메모리 > 예산 → Pool 크기 감소
└─ 응답시간 정상 → 감소 고려
```

</details>

---

**[⬅️ 이전: 분산 추적](./02-distributed-tracing.md)** | **[홈으로 🏠](../README.md)** | **[다음: gRPC vs REST 성능 비교 ➡️](./04-performance-comparison.md)**
