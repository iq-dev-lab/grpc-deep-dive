# 스트리밍 흐름 제어 — HTTP/2 Flow Control

---

## 🎯 핵심 질문

- HTTP/2 Flow Control의 Window Size는 무엇이고 어떻게 동작하는가?
- WINDOW_UPDATE Frame은 언제 전송되고 무슨 역할을 하는가?
- Flow Control이 없으면 어떤 문제가 발생하는가?
- 연결 레벨 Window와 스트림 레벨 Window는 어떻게 다른가?
- gRPC에서 Window Size를 조정하면 처리량이 어떻게 변하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

HTTP/2 Flow Control은 빠른 서버가 느린 클라이언트를 압도하지 않도록 자동으로 속도를 조절하는 메커니즘입니다. Window Size를 이해하지 못하면 대용량 스트리밍에서 OOM 에러가 발생하거나, 반대로 너무 작게 설정하면 처리량이 급격히 떨어집니다. 올바른 Window Size 튜닝으로 초당 1GB 이상의 처리량을 유지하면서도 메모리 안정성을 확보할 수 있습니다.

---

## 😱 흔한 실수

```java
// 실수 1: Window Size를 기본값으로 두고 대용량 스트리밍
public void streamLargeData() {
    // ManagedChannelBuilder가 기본값 사용
    ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
        .usePlaintext()
        .build();  // Window Size = 64KB (기본값)
    
    // 100MB 파일 스트리밍 시도
    // 서버가 초당 100MB 전송, 클라이언트가 초당 10MB 처리
    // → 90MB가 Window에 쌓임 → 메모리 고갈
}


// 실수 2: Window Size를 너무 크게 설정
ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
    .withOption(ChannelOption.GRPC_MAX_RECEIVE_MESSAGE_LENGTH, 1024*1024*1024)  // 1GB (!!)
    .build();

// 문제:
// - 느린 클라이언트(초당 10MB) vs 빠른 서버(초당 1GB)
// - 서버가 1GB를 모두 클라이언트 메모리에 쏟아붓기
// - 클라이언트: OOM, 강제 종료


// 실수 3: Reactive Streams Backpressure와 HTTP/2 Flow Control 혼동
public void streamWithReactiveBackpressure() {
    // 클라이언트에서 Backpressure 요청
    subscription.request(1000);  // "1000개만 보내달라"
    
    // 하지만 서버가 HTTP/2 Window에 관계없이 계속 전송
    // → Reactive Streams와 HTTP/2는 독립적!
    // → Backpressure를 두 번 구현해야 함 (비효율)
}
```

---

## ✨ 올바른 접근

```java
// 최적 Window Size 설정
ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
    .usePlaintext()
    .withOption(ChannelOption.GRPC_MAX_INBOUND_MESSAGE_SIZE, 256*1024*1024)  // 256MB
    // 기본값은 64KB인데, 대용량 스트리밍은 이를 초과할 수 있음
    .build();

// 또는 gRPC Spring Boot에서:
grpc:
  server:
    max-inbound-message-size: 268435456  # 256MB
  client:
    CHANNEL_NAME:
      max-inbound-message-size: 268435456
```

---

## 🔬 내부 동작 원리

### 1. Window Size 동작 메커니즘 (ASCII art)

```
시뮬레이션: 100MB 파일 스트리밍, Window Size = 64KB

┌─────────────────────────────────────────────────────────────┐
│ T=0ms: Window = 64KB, Server ──► DATA[0-64KB] ──► Client   │
│        Client 수신 버퍼: [████████████████] (64KB)          │
│        Window = 0 (가득 참)                                 │
│                                                             │
│ T=5ms: Server는 더 보낼 것이 있지만                        │
│        Window = 0이므로 STOP (Flow Control 작동)            │
│        Server는 WINDOW_UPDATE를 기다림                      │
│                                                             │
│ T=50ms: Client가 64KB 처리 완료                             │
│         WINDOW_UPDATE(+64KB) 전송                           │
│         Window = 64KB (복구)                               │
│                                                             │
│ T=55ms: Server ──► DATA[64KB-128KB] ──► Client            │
│         Window = 0 (다시 가득 찬다)                         │
│                                                             │
│ T=100ms: WINDOW_UPDATE(+64KB) 수신                         │
│          Window = 64KB                                      │
│                                                             │
│ [반복...]                                                   │
│                                                             │
│ 100MB ÷ 64KB = 1600개 청크                                  │
│ 각 청크: 64KB 수신 → 처리(50ms) → WINDOW_UPDATE            │
│ 총 시간: 1600 × 50ms = 80초                                │
│ 처리량: 100MB / 80s = 1.25MB/s (느림!)                     │
└─────────────────────────────────────────────────────────────┘


최적 Window Size로 개선:

┌─────────────────────────────────────────────────────────────┐
│ Window = 256MB로 증가                                       │
│                                                             │
│ T=0ms:   Server ──► DATA[0-256MB] ──► Client              │
│          Client 버퍼: [████████...] (256MB)                │
│          Window = 0                                        │
│                                                             │
│ T=100ms: Client가 256MB 모두 처리 완료                      │
│          (100MB만 필요)                                    │
│          WINDOW_UPDATE(+256MB) 전송                         │
│                                                             │
│ 총 시간: 100ms (거의 즉시!)                               │
│ 처리량: 100MB / 0.1s = 1GB/s                              │
│ 메모리: Client 256MB (안정적)                              │
└─────────────────────────────────────────────────────────────┘
```

### 2. 연결 레벨 vs 스트림 레벨 Window

```
┌──────────────────────────────────────────────────┐
│ Connection Level Window: 65535 bytes (기본값)   │
│ (모든 스트림의 합계)                            │
│                                                  │
│ Stream 1 Window: 65535 bytes (독립)             │
│ Stream 2 Window: 65535 bytes (독립)             │
│ Stream 3 Window: 65535 bytes (독립)             │
│ ...                                             │
│                                                  │
│ 제약: Sum(All Streams) <= Connection Window   │
└──────────────────────────────────────────────────┘


시나리오 1: 단일 스트림 100MB 파일
┌────────────────────────────────┐
│ Connection Window = 256MB      │
│ Stream 1 Window = 256MB        │
│ Stream 2,3,... Window = 64KB   │
│ (사용 안 함)                    │
│                                │
│ Stream 1만 전송 가능 (256MB)  │
└────────────────────────────────┘


시나리오 2: 100개 스트림 병렬 전송
┌────────────────────────────────┐
│ Connection Window = 256MB      │
│ Stream 1 Window = 2.56MB       │
│ Stream 2 Window = 2.56MB       │
│ ...                            │
│ Stream 100 Window = 2.56MB     │
│ (각 스트림: 256MB ÷ 100)       │
│                                │
│ 병렬로 각 2.56MB씩 전송        │
│ 총 256MB 동시 처리 가능        │
└────────────────────────────────┘
```

### 3. Flow Control이 없으면 발생하는 문제 시나리오

```
시뮬레이션: Flow Control 비활성화 (이론적)

┌────────────────────────────────────────────────────────────┐
│ Server: 100MB/s 전송 능력                                 │
│ Client: 10MB/s 처리 능력                                  │
│ (네트워크 대역폭: 1000MB/s)                               │
│                                                            │
│ T=0s:    Server 전송 시작                                │
│ T=1s:    100MB 송신 (90MB 누적 = 수신 버퍼 대기)          │
│ T=2s:    200MB 송신 (290MB 누적)                         │
│ T=3s:    300MB 송신 (590MB 누적)                         │
│ T=5s:    500MB 송신 (1.1GB 누적!!!)                      │
│          → Client 메모리 (2GB) 80% 도달                   │
│ T=6s:    600MB 송신 (1.5GB 누적)                         │
│          → Client 메모리 고갈                             │
│          → OOM Exception 발생                             │
│          → 프로세스 강제 종료                             │
│          → 모든 데이터 손실                               │
│                                                            │
│ HTTP/2 Flow Control이 있으면:                            │
│ T=0s:    Window = 256MB, Server ──► 256MB 전송           │
│ T=0.1s:  Client ◄── WINDOW_UPDATE 수신                   │
│ T=0.15s: Server ──► 다음 256MB 전송                      │
│ [안정적 흐름 유지]                                        │
└────────────────────────────────────────────────────────────┘
```

### 4. gRPC에서 Window Size 설정 및 튜닝

```java
/**
 * gRPC Client - Window Size 설정
 */
public class OptimizedGrpcClient {
    
    public ManagedChannel createOptimizedChannel(String host, int port) {
        return ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext()
            // Connection Level Window Size
            // 기본값: 65535 bytes
            // 권장: 최소 1MB (대용량 스트리밍)
            .withOption(ChannelOption.HTTP2_FLOW_CONTROL_WINDOW, 
                10 * 1024 * 1024)  // 10MB
            
            // Stream Level Inbound Message Size
            // 기본값: 4MB
            // 권장: 256MB (대용량)
            .withOption(ChannelOption.GRPC_MAX_INBOUND_MESSAGE_SIZE, 
                256 * 1024 * 1024)  // 256MB
            
            // 연결 타임아웃
            .keepAliveWithoutCalls(true)
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(5, TimeUnit.SECONDS)
            
            // 헬스체크 (선택사항)
            .enableRetry()
            .retryBufferSize(16 * 1024 * 1024)  // 16MB
            
            .build();
    }
}

/**
 * gRPC Server - Window Size 설정
 */
public class OptimizedGrpcServer {
    
    public Server createOptimizedServer(int port) {
        return ServerBuilder.forPort(port)
            .addService(new MyServiceImpl())
            // Connection Level Window Size
            .withOption(ChannelOption.HTTP2_FLOW_CONTROL_WINDOW, 
                10 * 1024 * 1024)  // 10MB
            
            // Message Size
            .maxInboundMessageSize(256 * 1024 * 1024)  // 256MB
            
            // 스레드 풀
            .executor(Executors.newFixedThreadPool(16))
            
            // Keep Alive
            .keepAliveTime(30, TimeUnit.SECONDS)
            .keepAliveTimeout(5, TimeUnit.SECONDS)
            .permitKeepAliveWithoutCalls(true)
            .permitKeepAliveTime(5, TimeUnit.MINUTES)
            
            .build();
    }
}

/**
 * Spring Boot gRPC 설정
 */
// application.yml
grpc:
  server:
    port: 50051
    enable-keep-alive: true
    keep-alive-time: 30s
    keep-alive-timeout: 5s
    permit-keep-alive-without-calls: true
    max-inbound-message-size: 268435456  # 256MB
  
  client:
    my-service:
      address: static://localhost:50051
      negotiation-type: plaintext
      max-inbound-message-size: 268435456  # 256MB
      enable-keep-alive: true
      keep-alive-time: 30s
      keep-alive-timeout: 5s


/**
 * Window Size 모니터링 및 튜닝
 */
public class FlowControlMonitor {
    
    private final AtomicLong windowUpdateCount = new AtomicLong();
    private final AtomicLong flowControlBlockedTime = new AtomicLong();
    
    /**
     * Metrics 수집 (Prometheus)
     */
    public void registerMetrics(MeterRegistry registry) {
        registry.gauge("grpc.flow_control.window_updates",
            windowUpdateCount::get);
        
        registry.timer("grpc.flow_control.blocked_time")
            .record(flowControlBlockedTime.get(), TimeUnit.MILLISECONDS);
    }
    
    /**
     * Window Size 자동 조정
     */
    public int optimizeWindowSize(long clientProcessingRate, 
                                  long serverTransmissionRate) {
        // 클라이언트가 처리할 수 있는 최대 버퍼 크기
        long optimalWindow = clientProcessingRate * 100;  // 100ms 분량
        
        // 범위 제한: 1MB ~ 256MB
        return (int) Math.max(1024 * 1024, 
                Math.min(256 * 1024 * 1024, optimalWindow));
    }
}
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# Window Size 성능 비교 테스트

# 1. 100MB 파일로 스트리밍 테스트
dd if=/dev/zero of=stream_test.bin bs=1M count=100

# 2. Window Size 64KB (기본값)
echo "Testing with Window Size 64KB..."
time java -cp grpc-all.jar:. StreamClient \
  -window-size 65536 \
  -file stream_test.bin

# 3. Window Size 256MB (최적)
echo "Testing with Window Size 256MB..."
time java -cp grpc-all.jar:. StreamClient \
  -window-size 268435456 \
  -file stream_test.bin

# 4. 메모리 프로파일링
jmap -histo:live <pid> | head -20

# 5. 네트워크 모니터링
iftop -i eth0 -n
```

---

## 📊 성능/비용 비교

```
Window Size별 처리량 및 메모리 사용량 (100MB 파일)

┌──────────────┬─────────┬──────────┬────────────┬──────────────┐
│ Window Size  │ 처리량  │ 메모리   │ 예상 시간  │ CPU 점유율   │
├──────────────┼─────────┼──────────┼────────────┼──────────────┤
│ 64KB (기본)  │ 1MB/s   │ 200MB    │ 100초      │ 높음 (context)│
│ 1MB          │ 50MB/s  │ 300MB    │ 2초        │ 보통         │
│ 16MB         │ 200MB/s │ 400MB    │ 0.5초      │ 낮음         │
│ 256MB (최적) │ 500MB/s │ 500MB    │ 0.2초      │ 매우 낮음    │
└──────────────┴─────────┴──────────┴────────────┴──────────────┘
```

---

## ⚖️ 트레이드오프

```
✅ 장점: Flow Control 사용
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 자동 백프레셔 (Automatic Backpressure)
2. 메모리 안정성 (OOM 방지)
3. 공정한 리소스 배분 (여러 스트림)

❌ 제약사항
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. Window Size 튜닝 필요
2. 연결/스트림 레벨 Window 관리 복잡
3. 설정 오류 시 성능 급락
```

---

## 📌 핵심 정리

```
HTTP/2 Flow Control = 자동 속도 제어

1. Window Size (기본 64KB) → 처리량 병목
2. 대용량 스트리밍: 256MB로 설정
3. 연결/스트림 레벨 Window 구분
4. WINDOW_UPDATE로 버퍼 복구
5. 설정 최적화로 10배 성능 향상
```

---

## 🤔 생각해볼 문제

### Q1: Window Size가 64KB로 설정된 상태에서 초당 100MB 파일을 전송하면, 정확히 몇 개의 WINDOW_UPDATE 프레임이 주고받아지는가?

<details>
<summary>해설 보기</summary>

```
100MB = 104,857,600 bytes
Window Size = 65,536 bytes (64KB)

필요한 청크 수: 104,857,600 / 65,536 = 1,600개

Timeline:
T=0ms: 청크 1-1600 전송 시작
T=1ms: 청크 1 도착 (Window = 0)
T=2ms: 청크 1 처리 (50ms 소요), WINDOW_UPDATE 전송
T=52ms: 청크 2 도착
...
T=80000ms: 청크 1600 완료

WINDOW_UPDATE 프레임: 1,600개
각 WINDOW_UPDATE: 9 bytes (frame header)
총 오버헤드: 1,600 × 9 = 14,400 bytes (무시할 수준)

총 처리 시간: 80초
처리량: 100MB / 80s = 1.25MB/s

결론: WINDOW_UPDATE는 많지만, CPU 오버헤드가 주요 문제
```
</details>

### Q2: 연결 레벨 Window와 스트림 레벨 Window의 합이 맞지 않으면 무슨 일이 일어나는가?

<details>
<summary>해설 보기</summary>

예: Connection Window = 256MB, Stream 1 Window = 300MB

```
이론적으로 불가능!

HTTP/2 스펙:
Sum(Stream Windows) <= Connection Window

Stream 1이 300MB를 요청하면:
- 서버: "Connection에는 256MB만 가능"
- Stream 1이 최대 256MB까지 송신
- 나머지 44MB는 다른 스트림에 양보

자동으로 조정되므로 에러는 아니지만,
성능 저하 가능 (스트림 우선순위 분쟁)
```
</details>

### Q3: gRPC 클라이언트가 Window Size를 무시하고 계속 데이터를 버퍼링하면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

Flow Control은 HTTP/2 프로토콜 레벨이므로, gRPC 클라이언트가 "무시"할 수 없습니다. Netty 네트워크 라이브러리가 자동으로 Window를 관리합니다. 만약 애플리케이션 수준에서 Reactive Streams 없이 무한정 버퍼링하면 OOM이 발생합니다.
</details>

---

<div align="center">

**[⬅️ 이전: Bidirectional Streaming](./03-bidirectional-streaming.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스트리밍 에러 처리 ➡️](./05-streaming-error-handling.md)**

</div>
