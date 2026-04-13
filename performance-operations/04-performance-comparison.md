# gRPC vs REST 성능 비교 — 수치로 보는 차이

---

## 🎯 핵심 질문

- 동일 비즈니스 로직에서 JSON/REST와 Protobuf/gRPC의 직렬화 크기 차이는?
- 처리량(TPS)과 지연시간(p99)에서 gRPC가 유리한 시나리오는?
- JMH로 올바른 벤치마크를 수행하는 방법은?
- gRPC가 REST보다 느릴 수 있는 시나리오는?
- 벤치마크 결과를 신뢰하기 위해 주의해야 할 함정은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

기술 도입 결정은 정량적 데이터가 필요합니다. 잘못된 벤치마크는 잘못된 결정을 초래합니다. JMH 벤치마크를 올바르게 수행해야 신뢰할 수 있는 비교가 가능합니다.

---

## 😱 흔한 실수 (Before)

```java
// 실수 1: JVM 워밍업 없이 벤치마크
@Test
public void benchmarkGrpc() {
    for (int i = 0; i < 10; i++) {
        stub.getUser(request);  // ❌ i=0,1 느림 (JIT 미완성)
    }
}

// 실수 2: 단일 요청 크기로 벤치마크
GetUserResponse getUser() {  // 크기: 100 바이트
    return response;
}
// 실제 운영: 평균 5KB 메시지

// 실수 3: localhost에서만 벤치마크
// → 네트워크 RTT 0ms, 현실성 낮음
```

---

## ✨ 올바른 접근 (After)

```xml
<!-- pom.xml: JMH 의존성 -->
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-core</artifactId>
    <version>1.35</version>
</dependency>
<dependency>
    <groupId>org.openjdk.jmh</groupId>
    <artifactId>jmh-generator-annprocess</artifactId>
    <version>1.35</version>
</dependency>
```

```java
// JMH 벤치마크: 올바른 설정
@Fork(value = 3,
    warmupIterations = 5)
@Measurement(iterations = 10,
    time = 1, timeUnit = TimeUnit.SECONDS)
@Threads(10)  // 10개 스레드
@State(Scope.Thread)
@BenchmarkMode(Mode.Throughput)
@OutputTimeUnit(TimeUnit.SECONDS)
public class GrpcRestBenchmark {
    
    private static final int MESSAGE_SIZE = 5000;  // 5KB
    
    private UserServiceGrpc.UserServiceBlockingStub 
        grpcStub;
    private RestTemplate restTemplate;
    
    @Setup
    public void setUp() {
        // 서버 시작
        grpcStub = UserServiceGrpc
            .newBlockingStub(channel);
        restTemplate = new RestTemplate();
    }
    
    // ✅ gRPC 벤치마크
    @Benchmark
    public GetUserResponse benchmarkGrpc() {
        GetUserRequest request = 
            GetUserRequest.newBuilder()
                .setId("user123")
                .setData(generateData(MESSAGE_SIZE))
                .build();
        
        return grpcStub.getUser(request);
    }
    
    // ✅ REST 벤치마크
    @Benchmark
    public UserDTO benchmarkRest() {
        UserRequest request = new UserRequest();
        request.setId("user123");
        request.setData(generateData(MESSAGE_SIZE));
        
        return restTemplate.postForObject(
            "http://localhost:8080/users",
            request,
            UserDTO.class
        );
    }
    
    private String generateData(int sizeBytes) {
        return "x".repeat(sizeBytes);
    }
    
    public static void main(String[] args) throws Exception {
        new Runner(
            new OptionsBuilder()
                .include(GrpcRestBenchmark.class
                    .getSimpleName())
                .build()
        ).run();
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 직렬화 크기 비교 (실제 메시지)

```
메시지: User 정보 (전형적인 크기)
{
  "id": "user123456",
  "name": "John Doe",
  "email": "john@example.com",
  "age": 28,
  "active": true,
  "metadata": {
    "created_at": "2024-01-15T10:30:00Z",
    "last_login": "2024-04-13T15:45:30Z",
    "login_count": 157
  }
}

┌──────────────────────────────────────┐
│  직렬화 크기 비교                    │
├──────────────────────────────────────┤
│                                       │
│ 형식      크기    압축      % 절감   │
│ ────────────────────────────────────  │
│ JSON      395B   152B      61%       │
│ Protobuf  156B   98B       75%       │
│ XML       612B   187B      69%       │
│                                       │
│ 대역폭 절감:                         │
│ REST (압축): 152B                   │
│ gRPC:       98B                     │
│ 차이:       35% 절감                 │
└──────────────────────────────────────┘

실제 네트워크 트래픽:
1000명 동시 사용자 × 10 요청/분

REST (압축):
152B × 1000 × 10 = 1.52MB/분

gRPC:
98B × 1000 × 10 = 0.98MB/분

절감: 35% 대역폭 감소
```

### 2. 처리량 비교 (TPS)

```
┌──────────────────────────────────────┐
│ 처리량 비교 (동일 하드웨어)        │
├──────────────────────────────────────┤
│                                       │
│ 시나리오     REST(HTTP/1.1)  gRPC   │
│ ────────────────────────────────────  │
│ Unary       2,000 req/s    10,000   │
│ Streaming   N/A            50,000   │
│ Small msg   5,000 req/s    8,000    │
│ Large msg   1,000 req/s    6,000    │
│                                       │
│ 이유:                                │
│ • HTTP/2 다중화 (gRPC)              │
│ • 연결 재사용 (gRPC)                │
│ • Protobuf 직렬화 오버헤드 적음    │
└──────────────────────────────────────┘

구체적 측정:
10000 요청, 100개 동시 연결

REST (HTTP/1.1):
시간: 5.0초
TPS: 2,000

gRPC:
시간: 1.0초
TPS: 10,000

5배 빠름!
```

### 3. JMH 벤치마크 설정 (워밍업, 측정)

```java
// JMH 워밍업 및 측정 상세 설정
@Fork(value = 3,  // 3번 실행 (별도 JVM)
    warmupIterations = 5,  // 워밍업 5회
    jvmArgs = {
        "-Xms2G",  // 힙 메모리 2GB
        "-Xmx2G",
        "-XX:+UseG1GC"  // GC 알고리즘
    })
@Measurement(
    iterations = 10,       // 10번 측정
    time = 1,              // 각 1초간
    timeUnit = TimeUnit.SECONDS)
@Warmup(
    iterations = 5,        // 5회 워밍업
    time = 1,
    timeUnit = TimeUnit.SECONDS)
@State(Scope.Benchmark)    // 전역 상태
@Threads(10)               // 10개 스레드
@BenchmarkMode({
    Mode.Throughput,       // req/sec
    Mode.AverageTime,      // 평균 시간
    Mode.SampleTime        // 샘플 분포 (p99)
})
@OutputTimeUnit(TimeUnit.MILLISECONDS)
public class OptimizedBenchmark {
    
    private volatile byte[] data;  // JIT 최적화 방지
    
    @Setup(Level.Trial)  // 모든 실행 전 한 번
    public void setUpTrial() {
        // 서버 시작
    }
    
    @Setup(Level.Iteration)  // 각 이터레이션 전
    public void setUpIteration() {
        // 워밍 생성
        data = new byte[5000];
    }
    
    @Benchmark
    public void benchmark() {
        // 벤치마크 코드
        doWork();
    }
    
    private void doWork() {
        // 구현
    }
}

// 워밍업 과정:
시간 0~5초: 워밍업 (JIT 컴파일)
시간 5~15초: 측정 (안정화된 성능)
결과: 일관된 성능 수치
```

### 4. gRPC가 유리한 vs REST가 유리한 경우

```
┌──────────────────────────────────────┐
│  시나리오별 추천                     │
├──────────────────────────────────────┤
│                                       │
│  gRPC 권장                           │
│  ─────────────────────────────────── │
│  • 높은 처리량 (> 1000 req/s)       │
│  • 낮은 지연 시간 (<100ms)          │
│  • 서버 간 통신 (마이크로서비스)    │
│  • 실시간 스트리밍                  │
│  • 대역폭 제약 (모바일)             │
│                                       │
│  REST 권장                           │
│  ─────────────────────────────────── │
│  • 낮은 처리량 (< 100 req/s)        │
│  • 캐싱 중요 (HTTP 캐싱)            │
│  • 클라이언트 다양성 (웹 브라우저) │
│  • 단순한 CRUD                      │
│  • 디버깅 우선 (텍스트 기반)       │
└──────────────────────────────────────┘
```

---

## 💻 실전 실험

```bash
# JMH 벤치마크 실행
mvn clean jmh:run -DincludeDefault=true

# 결과 분석
# Throughput: X ops/sec (높을수록 좋음)
# AverageTime: X ms (낮을수록 좋음)
# SampleTime p99: X ms (낮을수록 좋음)

# 리모트 벤치마크 (현실성 높음)
java -jar benchmark.jar \
  -rf json \
  -rff results.json

# 결과 파일 분석
# results.json을 파싱하여 그래프 작성
```

---

## 📊 성능/비용 비교

```
┌──────────────────────────────────────┐
│ 참고 성능 수치 (단일 머신)         │
├──────────────────────────────────────┤
│                                       │
│ 측정         REST        gRPC        │
│ ────────────────────────────────────  │
│ Unary RPC    2K req/s   10K req/s    │
│ p50 latency  10ms       2ms          │
│ p99 latency  50ms       5ms          │
│ 메시지 크기   150B       98B          │
│ 연결 효율     1 conn/   100 conn/   │
│             100 req     1 conn      │
│                                       │
│ 주의: 환경에 따라 크게 달라질 수 있음 │
└──────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
gRPC 도입 결정 기준:

보험: gRPC 권장하는 경우
├─ 팀에 Protobuf 전문가 있음
├─ 처리량 > 1000 req/s 필요
├─ 마이크로서비스 간 통신
└─ 성능이 중요 (금융, 거래)

위험: REST 유지하는 경우
├─ 팀이 Protobuf 모름
├─ 처리량 < 100 req/s
├─ 다양한 클라이언트 (웹, 모바일, 외부)
└─ 단순한 API
```

---

## 📌 핵심 정리

```
1. 직렬화 크기: Protobuf 30~50% 작음
2. 처리량: gRPC 5~10배 높음
3. 지연시간: gRPC 5배 낮음
4. JMH 필수: 워밍업, 충분한 측정
5. 단순한 API는 REST, 높은 처리량은 gRPC
```

---

## 🤔 생각해볼 문제

### Q1: JVM 워밍업 없이 벤치마크하면?

<details>
<summary>해설 보기</summary>

**정답: 잘못된 결과 (초기 10~20배 느림)**

```
첫 1000 요청:
├─ Bytecode → Machine code JIT 컴파일 중
├─ 성능: 2ms/req (느림)
└─ 결과 영향: 크음

이후 9000 요청:
├─ JIT 컴파일 완료
├─ 성능: 0.1ms/req (빠름)
└─ 결과 대표성: 높음

평균 = (1000 * 2 + 9000 * 0.1) / 10000 
     = 0.22ms/req (왜곡됨)

올바른 방법:
├─ 처음 5000개 버림 (워밍업)
├─ 이후 5000개만 측정
└─ 평균 = 0.1ms/req (정확)
```

</details>

---

### Q2: localhost에서만 벤치마크하면?

<details>
<summary>해설 보기</summary>

**정답: 네트워크 오버헤드 0 → 실제와 다름**

```
localhost 벤치마크:
RTT = 0ms
결과: 10000 req/s ← 현실성 없음

실제 네트워크:
RTT = 1ms (같은 데이터센터)
RTT = 50ms (다른 데이터센터)
결과: 최대 1000 req/s (RTT 영향)

→ 네트워크 지연을 반영하지 않음

올바른 방법:
├─ 다른 머신에서 벤치마크
├─ 실제 네트워크 조건 반영
└─ 더 정확한 결과
```

</details>

---

### Q3: 단일 요청 크기로 벤치마크하면?

<details>
<summary>해설 보기</summary>

**정답: 실제 운영 패턴과 다름**

```
벤치마크: 100B 메시지
결과: gRPC 10000 req/s

실제 운영: 5KB 메시지
직렬화 오버헤드 증가
GC 압력 증가
결과: gRPC 3000 req/s

→ 결과가 3배 왜곡됨

올바른 방법:
├─ 실제 메시지 크기 사용
├─ 메시지 다양성 포함
└─ 현실적인 결과
```

</details>

---

**[⬅️ 이전: 연결 관리 튜닝](./03-connection-tuning.md)** | **[홈으로 🏠](../README.md)** | **[다음: 마이그레이션 전략 ➡️](./05-migration-strategy.md)**
