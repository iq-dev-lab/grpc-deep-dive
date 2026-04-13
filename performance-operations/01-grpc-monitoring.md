# gRPC 모니터링 — Micrometer + Prometheus

---

## 🎯 핵심 질문

- grpc-spring-boot-starter가 자동으로 수집하는 Micrometer 메트릭은 무엇인가?
- gRPC Status Code별 에러율을 어떻게 추적하는가?
- Prometheus + Grafana 대시보드를 구성하는 방법은?
- p99 응답시간이 급증할 때 어떤 메트릭으로 병목을 진단하는가?
- 알람 규칙은 어떻게 설정하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC 서비스의 건강상태를 모르면 사용자가 에러를 보고하기 전까지 문제를 감지할 수 없습니다. Micrometer는 자동으로 메트릭을 수집하고 Prometheus에 저장하면 Grafana에서 시각화하고 알람을 설정할 수 있습니다.

---

## 😱 흔한 실수 (Before)

```java
// 실수 1: 메트릭 수집 비활성화
management:
  endpoints:
    web:
      exposure:
        include: health  # ❌ metrics 제외

// 실수 2: grpc_server_calls_total만 보고 에러율 확인 안 함
grpc_server_calls_total = 10000
grpc_server_calls_total{status=OK} = 9900
grpc_server_calls_total{status=INTERNAL} = 100
// → 에러율 1% 무시

// 실시 3: p50(중앙값)만 보고 p99 급증 놓침
p50 = 10ms (정상)
p99 = 1000ms (문제!)  // ❌ 확인 안 함
```

---

## ✨ 올바른 접근 (After)

```yaml
# application.yml: 메트릭 수집 활성화
management:
  endpoints:
    web:
      exposure:
        include: health,metrics,prometheus
  metrics:
    enable:
      jvm: true
      logback: true
      http: true
      process: true
      system: true
  metrics:
    tags:
      application: my-grpc-service
      environment: production
```

```java
// Java: 커스텀 메트릭 추가
@Service
public class OrderServiceImpl 
        extends OrderServiceGrpc.OrderServiceImplBase {
    
    private final MeterRegistry meterRegistry;
    
    public OrderServiceImpl(MeterRegistry meterRegistry) {
        this.meterRegistry = meterRegistry;
    }
    
    @Override
    public void createOrder(CreateOrderRequest request,
            StreamObserver<CreateOrderResponse> responseObserver) {
        
        long startTime = System.nanoTime();
        
        try {
            Order order = processOrder(request);
            
            // 성공 메트릭
            meterRegistry.counter(
                "orders.created",
                "status", "success")
                .increment();
            
            CreateOrderResponse response = 
                CreateOrderResponse.newBuilder()
                    .setOrderId(order.getId())
                    .build();
            
            responseObserver.onNext(response);
            responseObserver.onCompleted();
            
        } catch (Exception e) {
            // 실패 메트릭
            meterRegistry.counter(
                "orders.created",
                "status", "failure",
                "error", e.getClass().getSimpleName())
                .increment();
            
            responseObserver.onError(e);
        } finally {
            // 처리 시간 메트릭
            long duration = 
                (System.nanoTime() - startTime) / 1_000_000;
            meterRegistry.timer("orders.creation.time")
                .record(duration, TimeUnit.MILLISECONDS);
        }
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Micrometer gRPC 계측 자동화 구조

```
┌─────────────────────────────┐
│ gRPC 서버 시작              │
└────────────┬────────────────┘
             │
             ▼
┌──────────────────────────┐
│ GrpcServerMetricsObserver│
│ 자동 등록                │
└────────────┬─────────────┘
             │
   ┌─────────┴──────────┬───────────┬───────┐
   ▼                    ▼           ▼       ▼
메서드별           요청 수      응답 시간  상태
메트릭 수집        (Counter)   (Timer)    코드
                                        (Tag)

grpc_server_calls_total (Counter)
├─ service: UserService
├─ method: GetUser
└─ status: OK|INVALID_ARGUMENT|...

grpc_server_processing_duration_seconds (Timer)
├─ service: UserService
├─ method: GetUser
└─ histograms: 0.005, 0.01, 0.025, 0.05, ...

grpc_server_messages_received_total (Counter)
├─ service: UserService
├─ method: GetUser
└─ (스트리밍 메시지 수)

grpc_server_messages_sent_total (Counter)
├─ service: UserService
├─ method: GetUser
└─ (응답 메시지 수)
```

### 2. 주요 메트릭 종류와 의미

```
1. Counter (누적값)
   ────────────────
   grpc_server_calls_total
   └─ 총 요청 수 (Status별 분류)
      label: method, status
      
   예시:
   grpc_server_calls_total{
       method="GetUser",
       status="OK"
   } = 9900
   
   grpc_server_calls_total{
       method="GetUser",
       status="INTERNAL"
   } = 100

2. Timer (분포)
   ───────────
   grpc_server_processing_duration_seconds
   └─ 요청 처리 시간 분포
      histogram: p50, p90, p99
      
   예시:
   grpc_server_processing_duration_seconds
       {method="GetUser", le="+Inf"} = 1000
   grpc_server_processing_duration_seconds
       {method="GetUser", le="0.1"} = 950  (100ms 이상)
   grpc_server_processing_duration_seconds
       {method="GetUser", le="0.05"} = 900 (50ms 이상)

3. Gauge (현재값)
   ──────────────
   process_uptime_seconds
   └─ 프로세스 시작 후 경과시간
   
   jvm_memory_used_bytes
   └─ JVM 메모리 사용량
```

### 3. Prometheus 스크레이프 설정 + Grafana 쿼리

```yaml
# prometheus.yml: gRPC 메트릭 수집
global:
  scrape_interval: 15s
  evaluation_interval: 15s

scrape_configs:
  - job_name: 'grpc-service'
    static_configs:
      - targets: ['localhost:8080']
    metrics_path: '/actuator/prometheus'
```

```promql
# PromQL 쿼리 예시

1. 에러율 계산
────────────
rate(grpc_server_calls_total{status!="OK"}[5m]) /
  rate(grpc_server_calls_total[5m])
// 지난 5분 동안의 에러율

2. p99 응답 시간
───────────────
histogram_quantile(0.99, 
  rate(grpc_server_processing_duration_seconds_bucket[5m]))
// 지난 5분 동안 99% 요청의 응답 시간

3. 메서드별 처리량
──────────────
rate(grpc_server_calls_total[1m])
// 분당 요청 수

4. 상태별 요청 수
──────────────
grpc_server_calls_total{status="INTERNAL"}
// INTERNAL 에러 총합
```

### 4. 알람 규칙 설정

```yaml
# alerts.yml: 알람 규칙
groups:
  - name: grpc_alerts
    interval: 30s
    rules:
      # 규칙 1: 에러율 > 1%
      - alert: HighErrorRate
        expr: |
          (rate(grpc_server_calls_total{status!="OK"}[5m]) /
           rate(grpc_server_calls_total[5m])) > 0.01
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "High error rate detected"
          description: "Error rate is {{ $value }}"
      
      # 규칙 2: p99 > 500ms
      - alert: HighLatency
        expr: |
          histogram_quantile(0.99,
            rate(grpc_server_processing_duration_seconds_bucket[5m])
          ) > 0.5
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High p99 latency"
          description: "p99 is {{ $value }}s"
      
      # 규칙 3: 서버 다운
      - alert: ServiceDown
        expr: up{job="grpc-service"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "gRPC service is down"
```

---

## 💻 실전 실험

```bash
# 메트릭 엔드포인트 확인
curl http://localhost:8080/actuator/prometheus | 
  grep grpc_server

# 출력 예시:
# grpc_server_calls_total{method="GetUser",status="OK"} 150
# grpc_server_calls_total{method="GetUser",status="NOT_FOUND"} 5
# grpc_server_processing_duration_seconds_bucket{method="GetUser",le="0.05"} 140

# Docker로 Prometheus + Grafana 실행
docker-compose up -d

# Grafana 대시보드 접근
# http://localhost:3000 (admin:admin)
```

---

## 📊 성능/비용 비교

```
┌────────────────────────────────────┐
│ 메트릭 수집 오버헤드             │
├────────────────────────────────────┤
│                                     │
│ 메트릭     오버헤드  메모리      │
│ ──────────────────────────────────  │
│ 기본       <1%     ~5MB           │
│ + Custom   1~2%    +10MB          │
│ + Histogram 2~3%   +50MB          │
│                                     │
│ 권장: 필요한 메트릭만 수집        │
└────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
메트릭 상세도:
├─ 최소 (메서드만)
│  ✅ 빠름, 메모리 적음
│  ❌ 디버깅 어려움
├─ 표준 (메서드+상태)
│  ✅ 균형잡힘
│  ❌ 특정 조건 추적 어려움
└─ 최대 (모든 레이블)
   ✅ 상세 정보
   ❌ Cardinality 폭발, 느림

권장: 표준 + 필요한 커스텀
```

---

## 📌 핵심 정리

```
1. Micrometer 자동 계측
   grpc_server_calls_total, duration, messages

2. 메트릭 활성화
   management.endpoints.web.exposure.include: prometheus

3. Prometheus 스크레이프
   /actuator/prometheus 엔드포인트

4. PromQL 쿼리로 분석
   에러율, p99, 처리량

5. 알람 규칙 설정
   에러율 > 1%, p99 > 500ms
```

---

## 🤔 생각해볼 문제

### Q1: grpc_server_calls_total만으로 충분한가?

<details>
<summary>해설 보기</summary>

**정답: 아니오. Status별 분석 필요**

```
grpc_server_calls_total = 10000 (총합만으로는 위험)

올바른 분석:
grpc_server_calls_total{status="OK"} = 9900
grpc_server_calls_total{status="INVALID_ARGUMENT"} = 50
grpc_server_calls_total{status="INTERNAL"} = 50

에러율 = (50+50) / 10000 = 1%

→ Status별 분류로 문제 원인 파악 가능
```

</details>

---

### Q2: Histogram 메트릭은 왜 필요한가?

<details>
<summary>해설 보기</summary>

**정답: 응답 시간의 분포를 알기 위해**

```
평균만으로는 불충분:
평균 = 50ms (거짓)

실제 분포:
p50 = 10ms  (50% 사용자)
p90 = 50ms  (90% 사용자)
p99 = 1000ms (1% 사용자) ← 느린 사용자 무시됨

→ p99를 모니터링하지 않으면
   일부 사용자의 나쁜 경험 무시 가능
```

</details>

---

### Q3: Cardinality 폭발이란?

<details>
<summary>해설 보기</summary>

**정답: 레이블 조합이 너무 많아서 메모리 고갈**

```
안전한 경우:
labels: method=GetUser, status=OK
// 조합 = 1개

위험한 경우:
labels: method, user_id, timestamp, ...
// user_id = 1000개, 요청마다 다르면
// 조합 = 1000 * 5 (status) * ... = 수백만 개
// → Prometheus 메모리 고갈
```

</details>

---

**[⬅️ 이전: 테스트 전략](../spring-grpc/05-testing-strategy.md)** | **[홈으로 🏠](../README.md)** | **[다음: 분산 추적 ➡️](./02-distributed-tracing.md)**
