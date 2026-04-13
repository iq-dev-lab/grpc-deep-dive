# gRPC 생태계 — Web, Gateway, Envoy, Reflection

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- 브라우저에서 gRPC를 직접 호출할 수 없는 이유는 무엇이고, gRPC-Web은 어떻게 해결하는가?
- gRPC-Gateway로 REST와 gRPC를 동시에 지원할 때의 구조는 어떻게 되는가?
- Envoy Proxy가 gRPC 트래픽을 어떻게 처리하고, 왜 L7 프록시가 필요한가?
- gRPC Reflection은 스키마 파일 없이 서비스를 탐색할 수 있게 어떻게 돕는가?
- 내부 gRPC + 외부 REST 아키텍처를 구성할 때 어떤 컴포넌트가 필요한가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC의 장점은 서비스 간 내부 통신에서 극대화된다. 그런데 외부 클라이언트(브라우저, 모바일)가 직접 gRPC를 호출해야 하는 상황이 생기면 생태계를 알아야 한다. "gRPC는 브라우저 지원 안 된다"는 반쪽 사실이다. gRPC-Web, gRPC-Gateway, Envoy를 조합하면 내부는 gRPC, 외부는 REST/gRPC-Web으로 동시에 노출하는 아키텍처가 가능하다.

---

## 😱 흔한 실수 (Before — 생태계를 모를 때의 접근)

```
실수 1: "브라우저에서 gRPC 못 쓴다 → 프론트엔드용 REST API도 만들어야 한다"

  실제 상황:
    내부: gRPC 서비스 (주문, 상품, 결제)
    외부: 프론트엔드 (React/Vue)
  
  잘못된 해결책:
    gRPC 서비스와 별도로 REST Controller도 만들기
    → 동일 비즈니스 로직을 두 가지 방식으로 노출
    → 코드 중복, 유지보수 2배

  올바른 해결책:
    gRPC-Gateway 또는 gRPC-Web + Envoy
    → 단일 서버에서 REST와 gRPC 동시 노출

실수 2: Envoy 없이 gRPC 클라이언트 사이드 로드밸런싱만으로 해결하려 함

  마이크로서비스 10개가 상품 서비스 3개 인스턴스를 호출
  클라이언트 사이드 로드밸런싱:
    → 각 서비스가 DNS로 3개 주소를 알아야 함
    → 각 서비스에서 로드밸런서 설정 관리
    → 인스턴스 추가/제거 시 모든 클라이언트 재설정

  Envoy 중앙화:
    → 모든 gRPC 트래픽이 Envoy를 통과
    → 서버 인스턴스 변경을 Envoy에서만 관리
    → 클라이언트는 Envoy 주소 하나만 알면 됨

실수 3: Reflection을 프로덕션에 활성화해 서비스 스키마 노출

  gRPC Reflection: 스키마 없이도 서비스 구조 탐색 가능
  → 개발 환경에서 grpcurl 사용 시 편리
  → 프로덕션에서 활성화 → 서비스 메서드, 메시지 구조가 외부에 노출됨
  → 보안 취약점: 서비스 구조를 파악해 공격 벡터 탐색 가능
```

---

## ✨ 올바른 접근 (After — 생태계를 알고 조합하는 접근)

```
내부 gRPC + 외부 REST 아키텍처:

브라우저/모바일
    │
    ▼
[API Gateway / Nginx]
    │ REST (HTTP/1.1 + JSON)
    ▼
[gRPC-Gateway (Go)] 또는 [Spring Cloud Gateway + gRPC-Web-Proxy]
    │ gRPC (HTTP/2 + Protobuf)
    ▼
[Envoy Proxy] ← L7 로드밸런싱, mTLS, 관찰성
    │ gRPC
    ▼
[gRPC 서비스들: 주문, 상품, 결제, 배송]

각 컴포넌트 역할:
  gRPC-Gateway:  REST ↔ gRPC 변환 레이어
  Envoy:         서비스 간 L7 로드밸런싱, 트래픽 관리, mTLS
  Reflection:    개발 환경에서만 활성화, grpcurl로 탐색

환경별 Reflection 설정:
  # application-dev.yml
  grpc.server.reflection-service-enabled: true

  # application-prod.yml
  grpc.server.reflection-service-enabled: false
```

---

## 🔬 내부 동작 원리

### 1. 브라우저에서 gRPC를 직접 못 쓰는 이유

```
gRPC의 HTTP/2 요구사항 vs 브라우저 제약:

gRPC가 HTTP/2에서 사용하는 기능:
  ① Trailers (응답 후 헤더 전송)
     응답 DATA Frame 이후에 HEADERS Frame (grpc-status 포함)
     → 브라우저의 Fetch API / XMLHttpRequest: Trailer 헤더 접근 불가
     → 브라우저는 응답 헤더를 응답 완료 전에 접근할 수 없음

  ② 클라이언트 스트리밍 / Bidirectional 스트리밍
     브라우저의 Fetch API: 요청 본문을 스트리밍 불가 (ReadableStream 제한)
     → 클라이언트 → 서버 방향 스트리밍 불가

  ③ HTTP/2 직접 제어
     브라우저는 HTTP/2 프레임 레벨을 노출하지 않음
     → gRPC에서 필요한 :path, content-type=application/grpc 등의 제어 불가

gRPC-Web의 해결 방식:
  Trailer를 응답 Body 마지막에 포함:
    ↓ 일반 gRPC Trailer
    HEADERS Frame: grpc-status: 0

    ↓ gRPC-Web에서 변환
    DATA Frame에 Trailer 데이터를 인코딩해서 포함:
      [0x80] (Trailer 플래그) + 길이 + "grpc-status: 0\r\n"
    → 브라우저가 응답 Body에서 Trailer를 읽을 수 있음

  클라이언트 스트리밍 미지원:
    gRPC-Web 명세: Unary + Server Streaming만 지원
    Client Streaming / Bidirectional은 실험적 또는 미지원

gRPC-Web 프록시 필요성:
  브라우저 → gRPC-Web 요청 (HTTP/1.1 또는 HTTP/2)
  Envoy/grpc-web-proxy → gRPC로 변환 (HTTP/2 + Trailers 복원)
  서버 → 일반 gRPC 응답
```

### 2. gRPC-Gateway — REST와 gRPC 동시 지원

```
gRPC-Gateway 동작 방식 (Go 기반):

.proto 파일에 HTTP 규칙 추가:
  import "google/api/annotations.proto";
  
  service OrderService {
    rpc CreateOrder(CreateOrderRequest) returns (CreateOrderResponse) {
      option (google.api.http) = {
        post: "/v1/orders"    // REST: POST /v1/orders
        body: "*"             // 요청 본문 전체를 Protobuf 메시지로 매핑
      };
    }
    
    rpc GetOrder(GetOrderRequest) returns (GetOrderResponse) {
      option (google.api.http) = {
        get: "/v1/orders/{order_id}"  // REST: GET /v1/orders/{order_id}
        // {order_id} → GetOrderRequest.order_id 필드로 매핑
      };
    }
  }

protoc-gen-grpc-gateway 플러그인으로 생성:
  → OrderService.gw.go (HTTP 핸들러 + gRPC 클라이언트 코드)

실제 동작:
  REST 클라이언트 → gRPC-Gateway
    POST /v1/orders
    {"userId": "user-1", "items": [...]}
  
  gRPC-Gateway:
    1. JSON → Protobuf 역직렬화 (jsonpb 라이브러리)
    2. CreateOrderRequest 메시지 구성
    3. gRPC 서버로 Unary 호출
    4. CreateOrderResponse 수신
    5. Protobuf → JSON 직렬화
    6. HTTP 200 + JSON 응답 반환

Spring Boot 환경에서 gRPC-Gateway 없이 동일 효과:
  gRPC-Gateway는 Go 기반 → Java 환경에서 별도 Go 서버 필요
  
  대안 (Spring Boot):
  ① @RestController + gRPC Stub 호출:
    @RestController로 REST 엔드포인트 노출
    내부에서 Stub으로 gRPC 서비스 호출
    → 동일 JVM 또는 인프라 서비스로 분리 가능
  
  ② grpc-spring-boot-starter의 gRPC + HTTP 동시 포트:
    grpc.server.port=9090  (gRPC)
    server.port=8080       (HTTP/REST - Spring MVC)
    → 단일 애플리케이션에서 두 포트 동시 운영
```

### 3. Envoy Proxy — gRPC를 위한 L7 프록시

```
Envoy가 gRPC에서 필요한 이유:

L4 로드밸런서 문제 복습:
  nginx (L4): TCP 연결을 서버로 분산
    gRPC HTTP/2 단일 TCP 연결 → 이 연결을 서버 A로 고정
    → 모든 RPC가 서버 A로만 → 서버 B, C는 유휴
    → L4는 gRPC의 스트림 레벨을 볼 수 없음

Envoy (L7 gRPC 지원):
  HTTP/2 스트림 레벨에서 로드밸런싱:
    클라이언트 → Envoy: 단일 HTTP/2 연결
    Envoy → 서버 A: 일부 스트림 (RPC 1, 3, 5)
    Envoy → 서버 B: 일부 스트림 (RPC 7, 9)
    Envoy → 서버 C: 일부 스트림 (RPC 11, 13)
    → 스트림 레벨 로드밸런싱

Envoy gRPC 설정 예시 (envoy.yaml):
  static_resources:
    listeners:
    - name: grpc_listener
      address:
        socket_address: { address: 0.0.0.0, port_value: 10000 }
      filter_chains:
      - filters:
        - name: envoy.filters.network.http_connection_manager
          typed_config:
            "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
            codec_type: AUTO
            route_config:
              virtual_hosts:
              - name: grpc_service
                domains: ["*"]
                routes:
                - match:
                    prefix: "/order.v1.OrderService/"
                  route:
                    cluster: order_service_cluster
            http_filters:
            - name: envoy.filters.http.grpc_stats   # gRPC 메트릭 수집
            - name: envoy.filters.http.router

    clusters:
    - name: order_service_cluster
      type: STRICT_DNS
      lb_policy: ROUND_ROBIN
      http2_protocol_options: {}  # HTTP/2 (gRPC) 업스트림 연결
      load_assignment:
        cluster_name: order_service_cluster
        endpoints:
        - lb_endpoints:
          - endpoint:
              address:
                socket_address: { address: order-service, port_value: 9090 }

Envoy gRPC 추가 기능:
  ① gRPC → HTTP/1.1 트랜스코딩 (gRPC-JSON Transcoding)
     내부 gRPC 서비스를 REST로 외부 노출
     `.proto`의 http 옵션 사용
  
  ② gRPC Retry
     grpc-status 기반 자동 재시도 설정
  
  ③ gRPC Timeout 전파
     grpc-timeout 헤더 처리 및 전파
  
  ④ 관찰성 (Observability)
     gRPC 메서드별 요청 수, 에러율, 지연시간 자동 수집
     Prometheus 메트릭 노출
```

### 4. gRPC Reflection — 스키마 없이 서비스 탐색

```
gRPC Reflection 동작 방식:

활성화 (grpc-spring-boot-starter):
  grpc:
    server:
      reflection-service-enabled: true

Reflection이 제공하는 것:
  서비스 목록, 메서드 목록, 메시지 구조
  → .proto 파일 없이도 grpcurl로 서비스 탐색 가능

Reflection 프로토콜:
  service ServerReflection {
    rpc ServerReflectionInfo(stream ServerReflectionRequest)
        returns (stream ServerReflectionResponse);
  }
  → Bidirectional Streaming으로 구현

grpcurl과 Reflection:
  # Reflection 없이 (proto 파일 직접 지정)
  grpcurl -proto order.proto -plaintext localhost:9090 \
    order.v1.OrderService/CreateOrder -d '{...}'
  
  # Reflection 있으면 (proto 파일 불필요)
  grpcurl -plaintext localhost:9090 list
  # 출력:
  # grpc.reflection.v1alpha.ServerReflection
  # order.v1.OrderService
  # product.v1.ProductService
  
  grpcurl -plaintext localhost:9090 describe order.v1.OrderService
  # 출력:
  # order.v1.OrderService is a service:
  # service OrderService {
  #   rpc CreateOrder ( .order.v1.CreateOrderRequest )
  #       returns ( .order.v1.CreateOrderResponse );
  #   rpc WatchOrderStatus ( .order.v1.WatchOrderRequest )
  #       returns ( stream .order.v1.OrderStatusEvent );
  # }
  
  grpcurl -plaintext localhost:9090 \
    order.v1.OrderService/CreateOrder \
    -d '{"user_id": "user-1"}'
  # Reflection으로 요청/응답 구조 파악 → 자동으로 JSON 변환

Postman gRPC 지원:
  Reflection이 활성화되면 Postman에서 자동으로 서비스 탐색
  → REST API 테스트하듯이 gRPC 호출 가능
```

---

## 💻 실전 실험

### 실험 1: gRPC Reflection으로 서비스 탐색

```bash
# 서버 시작 (Reflection 활성화)
docker compose up -d grpc-server

# 전체 서비스 목록
grpcurl -plaintext localhost:9090 list

# 특정 서비스 메서드 목록
grpcurl -plaintext localhost:9090 list order.v1.OrderService

# 메시지 구조 확인
grpcurl -plaintext localhost:9090 describe order.v1.CreateOrderRequest
# 출력:
# order.v1.CreateOrderRequest is a message:
# message CreateOrderRequest {
#   string user_id = 1;
#   repeated .order.v1.OrderItem items = 2;
#   string shipping_address = 3;
# }

# 직접 호출 (proto 파일 없이)
grpcurl -plaintext localhost:9090 \
  order.v1.OrderService/CreateOrder \
  -d '{
    "user_id": "user-1",
    "items": [{"product_id": "prod-1", "quantity": 2}]
  }'
```

### 실험 2: Envoy gRPC 프록시 설정 및 확인

```bash
# Docker Compose로 Envoy + gRPC 서버 시작
docker compose up -d envoy grpc-server

# Envoy를 통해 gRPC 호출 (포트 10000)
grpcurl -plaintext localhost:10000 list

# Envoy Admin API로 상태 확인 (포트 9901)
curl http://localhost:9901/stats | grep grpc
# grpc.order_service.upstream_rq_total: 42
# grpc.order_service.upstream_rq_completed: 40

# Envoy가 여러 인스턴스로 분산하는지 확인
for i in $(seq 1 10); do
  grpcurl -plaintext localhost:10000 \
    order.v1.OrderService/GetServerInfo &
done
wait
# 각 요청이 다른 서버 인스턴스로 분산됨
```

### 실험 3: gRPC-Gateway (HTTP → gRPC) 확인

```bash
# gRPC-Gateway 포트 (8080)로 REST 호출
curl -X POST http://localhost:8080/v1/orders \
  -H "Content-Type: application/json" \
  -d '{"userId": "user-1", "items": [{"productId": "prod-1", "quantity": 2}]}'

# 응답: JSON (gRPC-Gateway가 Protobuf → JSON 변환)
# {"orderId": "order-abc123", "status": "ORDER_STATUS_PENDING"}

# gRPC 직접 호출 (동일 서비스)
grpcurl -plaintext localhost:9090 \
  order.v1.OrderService/CreateOrder \
  -d '{"user_id": "user-1", "items": [{"product_id": "prod-1", "quantity": 2}]}'

# 동일한 서버 로직이 두 가지 방식으로 호출됨
```

---

## 📊 성능/비용 비교

```
gRPC-Web vs 직접 gRPC vs REST 비교:

                    gRPC (내부)    gRPC-Web       REST
─────────────────────────────────────────────────────
브라우저 직접 호출      불가           가능           가능
직렬화                Protobuf       Protobuf       JSON
압축 효율             높음           높음            낮음
스트리밍 지원          4가지          Unary+Server   SSE/WS별도
추가 Proxy 필요       불필요          Envoy 필요     불필요
디버깅 편의성          낮음           낮음           높음
─────────────────────────────────────────────────────

Envoy gRPC 처리 오버헤드:
  Envoy 자체 지연: ~0.1~0.5ms (L7 처리 비용)
  로드밸런싱 이점: 서버 인스턴스 부하 균등 분산
  → 오버헤드 < 이점 (대규모 트래픽에서)

gRPC-Gateway 변환 비용 (Go):
  JSON 파싱 + Protobuf 직렬화: ~수십 μs
  → 전체 네트워크 지연 대비 무시할 수준
  → 처리량이 매우 높은 경우(초당 수만 요청) 고려 필요
```

---

## ⚖️ 트레이드오프

```
gRPC-Web:
  ✅ 브라우저에서 Protobuf + 타입 안전한 gRPC 호출
  ✅ 동일 .proto로 프론트엔드 코드 생성 (protoc-gen-grpc-web)
  ❌ Proxy(Envoy) 필수 → 운영 복잡도 증가
  ❌ Client Streaming / Bidirectional Streaming 미지원

gRPC-Gateway:
  ✅ 단일 .proto에서 REST와 gRPC 동시 지원
  ✅ JSON 클라이언트와의 하위 호환성 유지
  ❌ Go 기반 → Spring Boot 환경에서 별도 서비스 필요
  ❌ JSON 변환 레이어 → 직렬화 비용 추가
  ❌ REST 의미론과 gRPC 의미론의 매핑 복잡성

Envoy Proxy:
  ✅ gRPC L7 로드밸런싱, 재시도, 타임아웃, mTLS
  ✅ 운영 관찰성 (메트릭, 트레이싱 자동화)
  ✅ 서비스 메시(Istio)의 기반 컴포넌트
  ❌ 운영 복잡도 추가 (설정, 모니터링)
  ❌ 추가 홉 → 미세한 지연 증가

Reflection:
  ✅ 개발/테스트 편의성 극대화 (grpcurl, Postman)
  ❌ 프로덕션 활성화 시 서비스 구조 노출 → 보안 취약
  → 개발: 활성화, 프로덕션: 비활성화가 원칙
```

---

## 📌 핵심 정리

```
gRPC 생태계 핵심:

브라우저 지원:
  gRPC 직접 호출 불가 (Trailer, 스트리밍 제약)
  → gRPC-Web + Envoy Proxy
  → 또는 gRPC-Gateway로 REST 변환 노출

gRPC-Gateway:
  .proto의 google.api.http 옵션 → REST 매핑 생성
  JSON REST 요청 → Protobuf gRPC 변환
  외부 REST 클라이언트 지원 + 내부 gRPC 유지

Envoy Proxy:
  L7 gRPC 로드밸런싱 (스트림 단위 분산)
  gRPC-JSON 트랜스코딩
  gRPC 메트릭, 재시도, 타임아웃
  서비스 메시(Istio)의 데이터 플레인

gRPC Reflection:
  서비스 구조를 런타임에 노출
  grpcurl, Postman에서 proto 없이 서비스 탐색
  개발 환경에서만 활성화 (보안상 중요)

추천 아키텍처:
  외부: REST (gRPC-Gateway 또는 REST Controller)
  내부: gRPC (Envoy Proxy로 L7 로드밸런싱)
  브라우저: gRPC-Web + Envoy (또는 REST를 외부 노출)
```

---

## 🤔 생각해볼 문제

**Q1.** gRPC-Gateway를 사용하면 REST로 외부에 노출할 수 있다. 그렇다면 외부 API는 gRPC-Gateway를 통해 REST로 노출하고, 내부 서비스 간에는 gRPC로 직접 통신하는 아키텍처의 장단점은?

<details>
<summary>해설 보기</summary>

장점:
- 외부 클라이언트(브라우저, 모바일, 파트너사)가 익숙한 REST로 호출 가능
- 내부는 gRPC의 효율적 직렬화와 타입 안전한 계약 유지
- 외부 API와 내부 계약을 단일 .proto로 관리 (단, http 옵션 추가)
- API 게이트웨이(Kong, AWS API Gateway)와 REST 통합 쉬움

단점:
- gRPC-Gateway가 추가 컴포넌트 (운영 부담)
- JSON → Protobuf 변환 오버헤드 (일반적으로 무시할 수준)
- Streaming RPC는 REST로 변환 어려움 (Server Streaming → chunked response)
- REST와 gRPC 의미론 차이 (에러 코드 매핑, null 처리 등)

실무 판단:
- 외부 API가 있고 REST를 유지해야 한다면: gRPC-Gateway
- 완전히 새 프로젝트이고 외부 클라이언트도 gRPC-Web 지원 가능하다면: gRPC 통일

</details>

---

**Q2.** Istio 서비스 메시를 사용하면 Envoy가 Sidecar로 각 Pod에 주입된다. 이 경우 별도로 Envoy를 배포하지 않아도 gRPC L7 로드밸런싱이 자동으로 적용되는가?

<details>
<summary>해설 보기</summary>

네, Istio의 Sidecar Proxy(Envoy)가 자동으로 gRPC L7 처리를 담당합니다:

- 각 Pod에 주입된 Envoy Sidecar가 HTTP/2 트래픽을 인터셉트
- gRPC 스트림 레벨 로드밸런싱 자동 적용
- mTLS 자동 적용 (서비스 간 트래픽 암호화)
- 메트릭, 트레이싱 자동 수집

단, 주의사항:
- Sidecar 주입 오버헤드: 각 요청이 Envoy를 2번 통과 (발신 Pod의 Sidecar + 수신 Pod의 Sidecar) → ~1ms 추가 지연 가능
- Istio의 VirtualService 설정으로 gRPC 라우팅 규칙 정의 필요
- Istio가 없다면 별도 Envoy Gateway 배포가 가장 실용적

</details>

---

**Q3.** gRPC Reflection을 프로덕션에서 비활성화했는데, CI/CD 파이프라인에서 gRPC 서비스의 헬스체크를 `grpcurl`로 수행해야 한다. Reflection 없이 헬스체크를 구현하는 방법은?

<details>
<summary>해설 보기</summary>

두 가지 방법:

**방법 1: gRPC Health Checking Protocol 사용**
```java
// grpc-services 의존성 추가
// io.grpc:grpc-services

// 서버에 HealthStatusManager 등록
HealthStatusManager healthManager = new HealthStatusManager();
Server server = ServerBuilder.forPort(9090)
    .addService(healthManager.getHealthService())
    .build();

// 서비스 상태 설정
healthManager.setStatus("order.v1.OrderService",
    HealthCheckResponse.ServingStatus.SERVING);
```

헬스체크 호출 (proto 파일 불필요 — 표준 gRPC Health 프로토콜):
```bash
grpcurl -plaintext localhost:9090 \
  grpc.health.v1.Health/Check \
  -d '{"service": "order.v1.OrderService"}'
```

**방법 2: proto 파일을 CI 이미지에 포함**
```bash
grpcurl -proto health.proto -plaintext localhost:9090 \
  grpc.health.v1.Health/Check
```

gRPC Health Checking Protocol은 표준화된 방식으로 Kubernetes의 liveness/readiness probe와도 통합됩니다:
```yaml
livenessProbe:
  grpc:
    port: 9090
    service: "order.v1.OrderService"
```

</details>

---

<div align="center">

**[⬅️ 이전: gRPC 통신 4가지 패턴](./04-grpc-patterns.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 2 — Protobuf 직렬화 원리 ➡️](../protocol-buffers/01-tlv-encoding.md)**

</div>
