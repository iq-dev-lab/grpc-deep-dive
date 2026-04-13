# 스트리밍 에러 처리 — 재연결과 부분 실패 복구

---

## 🎯 핵심 질문

- 스트리밍 중 서버 에러가 발생하면 HTTP/2 레벨에서 무슨 일이 일어나는가?
- Exponential Backoff 재연결 전략을 어떻게 구현하는가?
- "이미 전송된 메시지"와 "전송 예정 메시지"의 처리 경계를 어떻게 나누는가?
- 부분 성공(일부 성공, 일부 실패)을 어떻게 표현하는가?
- 서버 재시작 시 클라이언트가 자동으로 재연결하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

스트리밍은 장시간 유지되는 연결이므로, 네트워크 끊김, 서버 재시작, 타임아웃이 발생할 가능성이 높습니다. Unary RPC는 즉시 실패를 반환하지만, 스트리밍은 onError()가 비동기로 호출되고, 이미 전송된 메시지와 손실된 메시지를 구분해야 합니다. 올바른 에러 처리와 재연결 전략으로 금융 시스템, IoT 센서, 게임 등 고가용성 시스템을 구축할 수 있습니다.

---

## 😱 흔한 실수

```java
// 실수 1: onError() 무시하고 계속
stub.subscribe(request, new StreamObserver<Data>() {
    @Override
    public void onError(Throwable t) {
        log.error("Error occurred", t);
        // 아무것도 안 함 - 재연결 시도 없음
        // 결과: 사용자는 영원히 데이터를 받지 못함
    }
});


// 실수 2: 무한 루프 재연결 (지수 백오프 없음)
private void reconnect() {
    try {
        subscribeToStream();
    } catch (Exception e) {
        Thread.sleep(1000);  // 항상 1초 대기
        reconnect();  // 즉시 재시도
    }
    // 서버가 다운되면 초당 1000번 재연결 시도 → 서버 부하
}


// 실수 3: 이미 처리된 메시지를 다시 처리
List<Message> receivedMessages = new ArrayList<>();
stub.subscribe(request, new StreamObserver<Message>() {
    @Override
    public void onNext(Message msg) {
        receivedMessages.add(msg);
    }
    @Override
    public void onError(Throwable t) {
        // 재연결 시도
        reconnect();  // 같은 메시지를 다시 받을 수 있음!
        // 결과: 메시지 중복 처리
    }
});
```

---

## ✨ 올바른 접근

```java
/**
 * 에러 처리 및 재연결 패턴
 */
public class ResilientStreamingClient {
    
    private final ManagedChannel channel;
    private final ScheduledExecutorService scheduler;
    private volatile boolean shouldReconnect = true;
    private int retryAttempt = 0;
    private long lastSuccessfulMessageTime = System.currentTimeMillis();
    
    public ResilientStreamingClient(String host, int port) {
        this.channel = ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext().build();
        this.scheduler = Executors.newScheduledThreadPool(1);
    }
    
    /**
     * 스트리밍 시작 (자동 재연결 포함)
     */
    public void subscribe(String id) {
        subscribeInternal(id);
    }
    
    private void subscribeInternal(String id) {
        try {
            stub.subscribe(
                SubscribeRequest.newBuilder().setId(id).build(),
                new ResilientStreamObserver(id));
        } catch (Exception e) {
            log.error("Failed to start subscription for {}", id, e);
            scheduleRetry(id);
        }
    }
    
    /**
     * 재연결 스케줄링 (Exponential Backoff)
     */
    private void scheduleRetry(String id) {
        if (!shouldReconnect) return;
        
        // Exponential Backoff: 1s, 2s, 4s, 8s, ... (최대 5분)
        long delayMs = Math.min(300000, 
            (long)Math.pow(2, retryAttempt) * 1000);
        retryAttempt++;
        
        log.info("Scheduling retry for {} in {}ms (attempt {})", 
            id, delayMs, retryAttempt);
        
        scheduler.schedule(() -> {
            if (shouldReconnect) {
                subscribeInternal(id);
            }
        }, delayMs, TimeUnit.MILLISECONDS);
    }
    
    /**
     * 에러 처리 및 상태별 재연결 결정
     */
    private class ResilientStreamObserver 
            implements StreamObserver<Data> {
        
        private final String id;
        private long lastReceivedSequenceNumber = -1;
        
        public ResilientStreamObserver(String id) {
            this.id = id;
            retryAttempt = 0;  // 성공하면 리셋
        }
        
        @Override
        public void onNext(Data data) {
            try {
                // 순서 추적 (중복 감지)
                long currentSeq = data.getSequenceNumber();
                if (currentSeq <= lastReceivedSequenceNumber) {
                    log.warn("Duplicate or out-of-order message: {} <= {}",
                        currentSeq, lastReceivedSequenceNumber);
                    return;  // 무시
                }
                
                lastReceivedSequenceNumber = currentSeq;
                lastSuccessfulMessageTime = System.currentTimeMillis();
                
                // 비즈니스 로직 처리
                processMessage(data);
                
            } catch (Exception e) {
                log.error("Error processing message", e);
                // 처리 오류: 메시지는 저장되었으므로 재연결만
            }
        }
        
        @Override
        public void onError(Throwable t) {
            Status status = extractStatus(t);
            
            log.error("Stream error: {} ({})", 
                status.getCode(), status.getDescription(), t);
            
            // 상태 코드별 처리
            if (isRetryable(status)) {
                log.info("Error is retryable, scheduling reconnection");
                scheduleRetry(id);
            } else {
                log.error("Non-retryable error, giving up: {}", 
                    status.getCode());
                // 치명적 에러: 재연결 포기
                shouldReconnect = false;
            }
        }
        
        @Override
        public void onCompleted() {
            log.info("Stream completed gracefully");
            retryAttempt = 0;
        }
        
        /**
         * Status Code별 재시도 판단
         */
        private boolean isRetryable(Status status) {
            switch (status.getCode()) {
                // 재시도 가능
                case UNAVAILABLE:           // 서버 다운
                case DEADLINE_EXCEEDED:     // 타임아웃
                case RESOURCE_EXHAUSTED:    // 메모리 부족
                case INTERNAL:              // 서버 오류
                case UNKNOWN:               // 알 수 없는 오류
                    return true;
                    
                // 재시도 불가능
                case INVALID_ARGUMENT:      // 요청 형식 오류
                case NOT_FOUND:            // 리소스 없음
                case PERMISSION_DENIED:     // 권한 없음
                case UNAUTHENTICATED:       // 인증 필요
                case FAILED_PRECONDITION:   // 사전 조건 위반
                case OUT_OF_RANGE:         // 범위 벗어남
                case UNIMPLEMENTED:        // RPC 미구현
                case CANCELLED:            // 사용자 취소
                    return false;
                    
                default:
                    return false;
            }
        }
        
        private Status extractStatus(Throwable t) {
            if (t instanceof StatusRuntimeException) {
                return ((StatusRuntimeException)t).getStatus();
            }
            return Status.UNKNOWN.withCause(t);
        }
    }
    
    private void processMessage(Data data) {
        // 비즈니스 로직
    }
    
    public void shutdown() {
        shouldReconnect = false;
        scheduler.shutdown();
        channel.shutdown();
    }
}

/**
 * 부분 실패 응답 패턴
 */
message BatchProcessResponse {
    int32 total = 1;
    int32 successful = 2;
    int32 failed = 3;
    
    message FailedItem {
        string id = 1;
        string error = 2;
        int32 error_code = 3;  // Retryable?
    }
    
    repeated FailedItem failures = 4;
}

/**
 * 서버에서 부분 실패 응답
 */
public UploadResponse processFileChunks(
        Iterator<FileChunk> chunks) {
    
    int total = 0, successful = 0, failed = 0;
    List<String> failedChunks = new ArrayList<>();
    
    for (FileChunk chunk : chunks) {
        total++;
        try {
            saveChunkToStorage(chunk);
            successful++;
        } catch (IOException e) {
            failed++;
            failedChunks.add(String.format(
                "Chunk %d: %s", chunk.getIndex(), e.getMessage()));
        }
    }
    
    return UploadResponse.newBuilder()
        .setTotalChunks(total)
        .setSuccessful(successful)
        .setFailed(failed)
        .addAllFailedChunkIds(failedChunks)
        .build();
}
```

---

## 🔬 내부 동작 원리

### 1. 스트리밍 에러 전파 경로 — RST_STREAM → onError()

```
Server Error Scenario:
┌─────────────────────────────────────────────────────────┐
│ Server는 500 INTERNAL_ERROR 발생                       │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ Server가 RST_STREAM frame 전송                         │
│ (error_code: INTERNAL_ERROR = 0x2)                     │
└─────────────────────────────────────────────────────────┘
         ↓ (네트워크 50ms)
┌─────────────────────────────────────────────────────────┐
│ Client Netty가 RST_STREAM 수신                         │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ gRPC가 StreamObserver.onError() 호출                   │
│ StatusRuntimeException(Status.INTERNAL)                │
└─────────────────────────────────────────────────────────┘
         ↓
┌─────────────────────────────────────────────────────────┐
│ User Code 실행                                         │
│ log.error("Stream error")                             │
│ scheduleRetry()                                        │
└─────────────────────────────────────────────────────────┘


Timeline:
T=0ms:   Server처리 중
T=10ms:  Server error 발생
T=11ms:  RST_STREAM 생성
T=20ms:  Client 수신
T=21ms:  onError() 호출
T=22ms:  User code 실행
T=1022ms: Retry (1초 백오프)
```

### 2. Status Code별 재시도 가능 여부 판단

```
┌──────────────────┬─────────────┬────────────────────────┐
│ Status Code      │ Retryable   │ 설명                   │
├──────────────────┼─────────────┼────────────────────────┤
│ UNAVAILABLE      │ ✓           │ 서버 다운, 재시작 중  │
│ DEADLINE_EXCEEDED│ ✓           │ 타임아웃, 과부하      │
│ RESOURCE_EXHAUSTED│✓           │ 서버 리소스 부족      │
│ INTERNAL         │ ✓           │ 서버 에러              │
│ UNKNOWN          │ ✓           │ 미정의 에러            │
├──────────────────┼─────────────┼────────────────────────┤
│ INVALID_ARGUMENT │ ✗           │ 요청 형식 오류         │
│ NOT_FOUND        │ ✗           │ 리소스 없음            │
│ PERMISSION_DENIED│ ✗           │ 권한 없음              │
│ UNAUTHENTICATED  │ ✗           │ 인증 실패              │
│ CANCELLED        │ ✗           │ 사용자 취소            │
└──────────────────┴─────────────┴────────────────────────┘
```

### 3. Exponential Backoff 재연결 구현

```
Exponential Backoff 시뮬레이션:

T=0ms:   Attempt 1, Error → Wait 1s
T=1000ms: Attempt 2, Error → Wait 2s
T=3000ms: Attempt 3, Error → Wait 4s
T=7000ms: Attempt 4, Error → Wait 8s
T=15000ms: Attempt 5, Error → Wait 16s
T=31000ms: Attempt 6, Error → Wait 32s
...

최대 대기: 5분 (300초)


계산 공식:
delay = min(maxDelay, baseDelay * 2^attempt)
      = min(300000, 1000 * 2^attempt) milliseconds


장점:
1. 서버 과부하 방지
2. 네트워크 안정화 시간 제공
3. 자동 무한 재연결 가능
```

### 4. 부분 실패 응답 패턴

```
Batch 처리에서 부분 실패:

Request: 100개 항목 처리
┌──────────────────────────────────┐
│ Item 1-95: 성공 (95개)          │
│ Item 96: DB 연결 오류 (1개)      │
│ Item 97-100: 성공 (4개)         │
└──────────────────────────────────┘

Response:
{
  "total": 100,
  "successful": 99,
  "failed": 1,
  "failures": [
    {
      "id": "item_96",
      "error": "Database connection timeout",
      "error_code": 6  // DEADLINE_EXCEEDED
    }
  ]
}

클라이언트:
- Item 96만 재시도
- 나머지 99개는 이미 저장됨
- 결과: 중복 처리 방지!
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# 에러 처리 및 재연결 테스트

# 1. 정상 스트리밍
java -cp grpc-all.jar:. StreamingClient &

# 2. 서버를 의도적으로 중단
sleep 5
kill <server_pid>

# 3. 클라이언트 로그 확인
# [ERROR] Stream error: UNAVAILABLE
# [INFO] Scheduling retry in 1000ms
# [INFO] Reconnecting...
# [INFO] Stream reestablished

# 4. 서버 재시작
java -cp grpc-all.jar:. StreamingServer &

# 5. 클라이언트가 자동 재연결 확인
# [INFO] Stream reestablished

# 6. 부분 실패 테스트
echo "Test batch with failures"
java -cp grpc-all.jar:. BatchProcessClient
```

---

## 📊 성능/비용 비교

```
재연결 전략별 복구 시간 및 리소스

┌─────────────────────┬────────────┬──────────────┬────────────┐
│ 재연결 전략         │ 첫 복구 시간│ 서버 부하    │ 총 시도   │
├─────────────────────┼────────────┼──────────────┼────────────┤
│ 즉시 재연결         │ 100ms      │ 매우 높음    │ 무한       │
│ (sleep 없음)        │            │ (폭주 공격)  │            │
├─────────────────────┼────────────┼──────────────┼────────────┤
│ 고정 1초            │ 1000ms     │ 높음         │ 무한       │
│ (계속 1초마다)      │            │ (1000 req/s) │            │
├─────────────────────┼────────────┼──────────────┼────────────┤
│ Exponential Backoff │ 1000ms     │ 낮음         │ 무한       │
│ (1s→2s→4s→...)     │            │ (자동 조절)  │            │
├─────────────────────┼────────────┼──────────────┼────────────┤
│ Circuit Breaker     │ 300000ms   │ 가장 낮음    │ 유한       │
│ (5분 후 포기)       │            │ (재연결 중지)│ (5분 후)   │
└─────────────────────┴────────────┴──────────────┴────────────┘
```

---

## ⚖️ 트레이드오프

```
✅ 장점: 자동 재연결
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 고가용성 (사용자 개입 없음)
2. 자동 복구 (서버 재시작)
3. 순서 보장 (중복 감지)

❌ 제약사항
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 상태 관리 복잡
2. 타임아웃 대기 시간
3. 메모리 누수 가능성 (리소스 해제)
```

---

## 📌 핵심 정리

```
Streaming 에러 처리 = 자동 재연결 + 상태 추적

1. onError(): 비동기 콜백 처리
2. Status Code: 재시도 판단 기준
3. Exponential Backoff: 서버 과부하 방지
4. Sequence Number: 중복 감지
5. 부분 실패: 재시도 대상 최소화
```

---

## 🤔 생각해볼 문제

### Q1: Exponential Backoff를 사용할 때, 만약 서버가 5분 이상 다운되면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

기본 Exponential Backoff는 최대 5분(300초)의 대기 시간이 설정되어 있습니다. 이 후에는 계속 5분마다 재시도합니다. 사용자가 원한다면 Circuit Breaker 패턴을 추가하여 일정 실패 횟수 후 포기할 수 있습니다.

```
Circuit Breaker 패턴:
- 5번 연속 실패 후 "Circuit Open" (30분 동안 재연결 중단)
- 30분 후 "Half-Open" (다시 시도)
- 성공하면 "Circuit Close" (정상 복구)
```
</details>

### Q2: 클라이언트가 10개 메시지를 수신한 후 에러가 발생했을 때, 재연결 후 11번째 메시지부터 수신하려면 어떻게 해야 하는가?

<details>
<summary>해설 보기</summary>

메시지에 `sequence_number` 필드를 추가하고, 클라이언트에서 마지막으로 받은 sequence number를 저장합니다. 재연결 시 `SubscribeRequest.from_sequence = 11`로 요청하면 서버가 그 다음부터 전송합니다.

```java
message SubscribeRequest {
    string id = 1;
    int64 from_sequence = 2;  // Optional: 복구용
}

클라이언트:
SubscribeRequest.newBuilder()
    .setId("stream_id")
    .setFromSequence(lastReceivedSeq + 1)
    .build()
```
</details>

### Q3: 만약 클라이언트가 중간에 크래시되면 서버는 언제 그 사실을 알 수 있는가?

<details>
<summary>해설 보기</summary>

TCP Keep-Alive와 gRPC Keep-Alive 신호에 따라 다릅니다.

```
시나리오:
- Keep-Alive interval: 30초 (설정값)
- 클라이언트가 T=10s에 크래시

T=10s: 클라이언트 crash
T=40s: 서버가 "30초 동안 신호 없음" 감지
T=40s: TCP connection reset 감지
T=40s: onError() 호출 (서버에서)

서버: 해당 클라이언트의 리소스 해제 가능
```
</details>

---

<div align="center">

**[⬅️ 이전: 스트리밍 흐름 제어](./04-flow-control.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 5 — TLS와 mTLS ➡️](../security-auth/01-tls-mtls.md)**

</div>
