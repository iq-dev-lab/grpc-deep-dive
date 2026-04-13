# 분산 추적 — OpenTelemetry gRPC Instrumentation

---

## 🎯 핵심 질문

- OpenTelemetry gRPC 자동 계측은 어떻게 TraceID를 메타데이터로 전파하는가?
- Baggage를 사용해 사용자 ID나 요청 컨텍스트를 체인 전체에 전달하는 방법은?
- Jaeger와 Zipkin 연동 설정은 어떻게 다른가?
- Sampling 전략별(전체/비율/레이트 리미팅) 장단점은?
- grpcurl로 수동으로 TraceID를 주입하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이크로서비스 아키텍처에서 요청이 A → B → C 서비스를 거치면, 전체 호출 체인을 추적해야 합니다. OpenTelemetry는 자동으로 TraceID를 전파하고 Jaeger에 저장하면 각 서비스의 지연시간 병목을 시각화할 수 있습니다.

---

## 😱 흔한 실수 (Before)

```java
// 실수 1: TraceID 없이 로깅
log.info("Processing order: " + orderId);
// 로그에서 이 요청이 어느 서비스에서 온 건지 알 수 없음

// 실수 2: Baggage에 민감한 데이터 포함
baggage.put("user_password", password);
// → 모든 추적 시스템에 노출

// 실시 3: 100% Sampling 설정 (고트래픽)
sampler: always_on
// ❌ 모든 요청 추적 → 추적 시스템 과부하

// 실수 4: TraceID 수동 전파 (자동화 불가)
String traceId = extractTraceId();
// 모든 다운스트림 호출에 수동으로 추가
```

---

## ✨ 올바른 접근 (After)

```xml
<!-- pom.xml: OpenTelemetry 의존성 -->
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-api</artifactId>
    <version>1.28.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry</groupId>
    <artifactId>opentelemetry-exporter-jaeger-thrift</artifactId>
    <version>1.28.0</version>
</dependency>
<dependency>
    <groupId>io.opentelemetry.instrumentation</groupId>
    <artifactId>opentelemetry-grpc-1.53</artifactId>
    <version>1.28.0-alpha</version>
</dependency>
```

```yaml
# application.yml: OpenTelemetry 설정
spring:
  application:
    name: order-service

otel:
  exporter:
    jaeger:
      endpoint: http://jaeger:14250
  
  traces:
    sampler:
      type: parentbased_traceidratio
      traceidratio: 0.1  # 10% 샘플링
  
  attributes:
    service:
      name: order-service
      version: 1.0.0
    deployment:
      environment: production
```

```java
// Java: TraceID 자동 전파 + Baggage 사용
@Service
public class OrderService {
    
    private final Tracer tracer;
    private final BaggageManager baggageManager;
    
    public OrderService(Tracer tracer, 
                       BaggageManager baggageManager) {
        this.tracer = tracer;
        this.baggageManager = baggageManager;
    }
    
    public Order createOrder(CreateOrderRequest request) {
        // 현재 Span 가져오기
        Span span = tracer.currentSpan();
        
        // TraceID는 자동으로 메타데이터에 포함됨
        log.info("TraceID: {}", span.getSpanContext()
            .getTraceId());
        
        // Baggage에 사용자 ID 저장 (민감하지 않은 정보)
        baggageManager.setBaggage("user_id", 
            getCurrentUserId());
        baggageManager.setBaggage("request_source", 
            "rest_api");
        
        try (Scope scope = tracer.withSpan(
                tracer.spanBuilder("validate_order")
                    .startSpan())) {
            
            validateOrder(request);
        }
        
        try (Scope scope = tracer.spanBuilder(
                "save_order")
            .startSpan()) {
            
            Order order = orderRepository.save(
                request.toEntity()
            );
            return order;
        }
    }
}

// 다운스트림 호출: gRPC 자동으로 TraceID 전파
@Service
public class PaymentClient {
    
    @GrpcClient("payment-service")
    private PaymentServiceGrpc.PaymentServiceBlockingStub 
        paymentStub;
    
    public PaymentResponse processPayment(String orderId) {
        // TraceID는 자동으로 메타데이터에 추가됨
        // (OpenTelemetry 자동 계측)
        
        return paymentStub.processPayment(
            ProcessPaymentRequest.newBuilder()
                .setOrderId(orderId)
                .build()
        );
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 분산 추적 전파 흐름 (A → B → C → D)

```
┌──────────────────┐
│ Frontend         │
│ (TraceID 생성)   │
│ TraceID: abc123  │
└────────┬─────────┘
         │
         │ HTTP 요청
         │ Header: traceparent: abc123
         ▼
┌──────────────────┐
│ Service A        │
│ (TraceID 수신)   │
│ TraceID: abc123  │
│ SpanID: span1    │
└────────┬─────────┘
         │
         │ gRPC 호출 (자동 전파)
         │ Metadata: traceparent: abc123
         ▼
┌──────────────────┐
│ Service B        │
│ TraceID: abc123  │
│ SpanID: span2    │
│ (Parent: span1)  │
└────────┬─────────┘
         │
         │ gRPC 호출
         │ Metadata: traceparent: abc123
         ▼
┌──────────────────┐
│ Service C        │
│ TraceID: abc123  │
│ SpanID: span3    │
│ (Parent: span2)  │
└────────┬─────────┘
         │
         │ DB 쿼리
         │ SpanID: span4
         ▼
┌──────────────────┐
│ Jaeger           │
│                  │
│ Trace: abc123    │
│ Spans:           │
│ - span1 (A)      │
│ - span2 (B)      │
│ - span3 (C)      │
│ - span4 (DB)     │
│                  │
│ Duration:        │
│ Total: 250ms     │
│ A→B: 50ms        │
│ B→C: 100ms       │
│ C→DB: 80ms       │
└──────────────────┘
```

### 2. OpenTelemetry gRPC Instrumentation 설정

```java
@Configuration
public class OtelGrpcConfiguration {
    
    @Bean
    public io.opentelemetry.sdk.trace.SdkTracerProvider 
            tracerProvider() {
        return SdkTracerProvider.builder()
            .addSpanProcessor(
                BatchSpanProcessor.builder(
                    JaegerThriftSpanExporter.builder()
                        .setEndpoint(
                            "http://jaeger:14250")
                        .build()
                )
                .build()
            )
            .setSampler(
                ParentBasedSampler.builder(
                    TraceIdRatioBased.create(0.1)
                )
                .build()
            )
            .build();
    }
    
    @Bean
    public OpenTelemetry openTelemetry(
            SdkTracerProvider tracerProvider) {
        return OpenTelemetrySdk.builder()
            .setTracerProvider(tracerProvider)
            .build();
    }
    
    @Bean
    public Tracer tracer(OpenTelemetry otel) {
        return otel.getTracer(
            "order-service", "1.0.0"
        );
    }
}

// gRPC Interceptor: 자동 계측
@Component
public class OpenTelemetryServerInterceptor 
        implements ServerInterceptor {
    
    private final Tracer tracer;
    
    @Override
    public <ReqT, RespT> ServerCall.Listener<ReqT> 
            interceptCall(
            ServerCall<ReqT, RespT> call,
            Metadata headers,
            ServerCallHandler<ReqT, RespT> next) {
        
        // TraceID 추출 (W3C Trace Context)
        String traceparent = headers.get(
            Metadata.Key.of("traceparent", 
                           ASCII_STRING_MARSHALLER)
        );
        
        Span span = tracer.spanBuilder(
                call.getMethodDescriptor()
                    .getFullMethodName()
            )
            .setParent(extractContext(traceparent))
            .startSpan();
        
        try (Scope scope = span.makeCurrent()) {
            return next.startCall(call, headers);
        } finally {
            span.end();
        }
    }
    
    private Context extractContext(String traceparent) {
        // W3C Trace Context 파싱
        if (traceparent == null) {
            return Context.current();
        }
        return W3CTraceContextPropagator
            .getInstance()
            .extract(Context.current(), 
                    new MapTextMapGetter(
                        parseTraceparent(traceparent)
                    ));
    }
}
```

### 3. Jaeger 연동 (OTLP exporter)

```java
@Configuration
public class JaegerConfiguration {
    
    @Bean
    public SdkTracerProvider jaegerTracerProvider(
            @Value("${otel.exporter.jaeger.endpoint}") 
            String jaegerEndpoint) {
        
        JaegerThriftSpanExporter exporter =
            JaegerThriftSpanExporter.builder()
                .setEndpoint(jaegerEndpoint)
                .build();
        
        return SdkTracerProvider.builder()
            .addSpanProcessor(
                BatchSpanProcessor.builder(exporter)
                    .setMaxQueueSize(2048)
                    .setMaxExportBatchSize(512)
                    .setScheduleDelayMillis(5000)
                    .build()
            )
            .build();
    }
}

// Jaeger UI: http://localhost:16686
// - 서비스 선택: order-service
// - TraceID 검색
// - 각 Span의 지연시간 시각화
```

### 4. Baggage 활용 (사용자 ID 체인 전파)

```java
// Baggage 저장 (Service A)
@Service
public class OrderService {
    
    private final BaggageManager baggageManager;
    
    public void processOrder(String orderId, 
                            String userId) {
        // Baggage: 모든 다운스트림 서비스에 자동 전파
        baggageManager.setBaggage(
            "user_id", userId);
        baggageManager.setBaggage(
            "request_correlation_id", 
            UUID.randomUUID().toString());
        
        // gRPC 호출 시 Baggage 자동 전파됨
        paymentClient.processPayment(orderId);
    }
}

// Baggage 읽기 (Service B)
@Service
public class PaymentService {
    
    @Override
    public void processPayment(
            ProcessPaymentRequest request,
            StreamObserver<PaymentResponse> observer) {
        
        // Baggage에서 사용자 ID 읽기 (자동 전파됨)
        String userId = 
            Baggage.current()
                .getEntryValue("user_id");
        
        log.info("Processing payment for user: {}", 
                userId);
        
        // 비즈니스 로직...
    }
}
```

---

## 💻 실전 실험

```bash
# grpcurl로 TraceID 수동 주입
grpcurl -plaintext \
  -H 'traceparent: 00-0af7651916cd43dd8448eb211c80319c-b7ad6b7169203331-01' \
  -d '{"orderId": "order123"}' \
  localhost:9090 \
  io.grpc.examples.OrderService/ProcessPayment

# 응답: PaymentResponse
# Jaeger에서 추적 ID 0af7651916cd43dd8448eb211c80319c로 검색 가능
```

---

## 📊 성능/비용 비교

```
┌──────────────────────────────────────┐
│ Sampling 전략별 비교                │
├──────────────────────────────────────┤
│                                       │
│ 전략          비용  정확성  지연     │
│ ────────────────────────────────────  │
│ Always On     높음  100%   높음      │
│ 10%          중간  90%    중간      │
│ 1%           낮음  낮음   낮음      │
│ AdaptiveSamp 중간  95%    중간      │
│                                       │
│ 권장: ParentBased + 10~50%          │
└──────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
Sampling 비율:
├─ 높음 (50%)
│  ✅ 정확한 추적
│  ❌ 시스템 오버헤드
├─ 낮음 (1%)
│  ✅ 낮은 오버헤드
│  ❌ 드물게 발생하는 에러 미추적
└─ 권장: 10% (균형)

Jaeger vs Zipkin:
├─ Jaeger
│  ✅ gRPC 기본 지원, 성능 좋음
│  ❌ 메모리 사용 많음
├─ Zipkin
│  ✅ 가벼움, 이해하기 쉬움
│  ❌ 성능 트레이드오프
└─ 권장: 프로덕션은 Jaeger
```

---

## 📌 핵심 정리

```
1. TraceID 자동 전파 (메타데이터)
2. OpenTelemetry gRPC 계측
3. Baggage: 민감하지 않은 정보만
4. Sampling: 10~50% 권장
5. Jaeger UI로 시각화
```

---

## 🤔 생각해볼 문제

### Q1: Baggage에 비밀번호를 저장하면?

<details>
<summary>해설 보기</summary>

**정답: 보안 위험 (모든 시스템에 노출)**

Baggage는 모든 추적 시스템, 로그, 메트릭에 포함됩니다.

```
Baggage: {"user_password": "secret123"}
  ↓
Jaeger UI, 로그, 모니터링 시스템 노출
  ↓
보안 위반
```

**안전한 방법:**
- Baggage: user_id, request_id만
- 민감정보: 암호화된 값 또는 토큰

</details>

---

### Q2: 100% Sampling을 고트래픽 환경에서 사용하면?

<details>
<summary>해설 보기</summary>

**정답: Jaeger 시스템 과부하**

```
매초 100,000 요청:
100% 샘플링 → 100,000 스팬/초
→ Jaeger 메모리 고갈
→ 추적 드롭 시작
→ 일부 요청은 추적 안 됨

10% 샘플링:
10% × 100,000 = 10,000 스팬/초 (관리 가능)
```

</details>

---

### Q3: ParentBased 샘플링은 왜 권장되는가?

<details>
<summary>해설 보기</summary>

**정답: 요청 체인의 일관성 유지**

```
ParentBased 샘플러:
├─ 부모 스팬이 샘플됨 → 자식도 샘플
├─ 부모 스팬이 샘플 안 됨 → 자식도 안 함
└─ 부모가 없으면 → 설정된 비율 적용

장점: 전체 요청 체인이 추적됨 또는 안 됨
      일부만 추적되는 일관성 문제 방지
```

</details>

---

**[⬅️ 이전: gRPC 모니터링](./01-grpc-monitoring.md)** | **[홈으로 🏠](../README.md)** | **[다음: 연결 관리 튜닝 ➡️](./03-connection-tuning.md)**
