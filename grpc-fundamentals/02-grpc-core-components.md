# gRPC 핵심 구성 요소 — .proto에서 Stub까지

---

## 🎯 핵심 질문

이 문서를 읽고 나면 다음 질문에 답할 수 있습니다.

- `.proto` 파일을 작성하면 어떤 과정을 거쳐 실행 가능한 코드가 되는가?
- `protoc` 컴파일러와 플러그인은 어떻게 동작하고, 무엇을 생성하는가?
- Channel, Stub, Server의 역할은 각각 무엇이고 어떻게 연결되는가?
- Blocking Stub, Async Stub, Future Stub의 차이는 무엇이고 언제 쓰는가?
- gRPC 서버가 요청을 받아 처리하고 응답을 돌려주는 전체 흐름은 어떻게 되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

gRPC를 처음 쓸 때 가장 많이 틀리는 부분은 Channel과 Stub을 매 요청마다 새로 만드는 것이다. Channel은 비싸고, Stub은 Channel을 감싼 경량 객체다. 이 관계를 모르면 커넥션 풀 고갈, 성능 저하, 리소스 누수를 경험한다. 전체 구성 요소의 역할과 생명주기를 이해하는 것이 gRPC를 안전하게 쓰는 출발점이다.

---

## 😱 흔한 실수 (Before — 원리를 모를 때의 접근)

```
실수 1: 매 요청마다 Channel과 Stub을 새로 생성

// ❌ 잘못된 패턴 — 매 요청마다 Channel 생성
public ProductResponse getProduct(String productId) {
    ManagedChannel channel = ManagedChannelBuilder
        .forAddress("product-service", 9090)
        .usePlaintext()
        .build();  // TCP + TLS 연결 수립 — 매우 비쌈!
    
    ProductServiceGrpc.ProductServiceBlockingStub stub =
        ProductServiceGrpc.newBlockingStub(channel);
    
    ProductResponse response = stub.getProduct(
        GetProductRequest.newBuilder()
            .setProductId(productId)
            .build()
    );
    
    channel.shutdown();  // 연결 즉시 종료
    return response;
}

문제:
  매 호출마다 TCP 연결 수립 + TLS 핸드쉐이크
  → HTTP/2의 연결 재사용 이점이 완전히 사라짐
  → gRPC를 쓰지만 HTTP/1.1보다 느린 아이러니한 상황

실수 2: Blocking Stub을 WebFlux/비동기 환경에서 사용

// ❌ WebFlux에서 Blocking Stub 사용
@GetMapping("/products/{id}")
public Mono<ProductResponse> getProduct(@PathVariable String id) {
    // Blocking Stub → 스레드 블록 → Reactor 스케줄러 고갈
    ProductResponse response = blockingStub.getProduct(
        GetProductRequest.newBuilder().setProductId(id).build()
    );
    return Mono.just(response);
}

문제:
  Blocking Stub은 응답이 올 때까지 현재 스레드를 블록
  Reactor의 Non-blocking 스레드 풀에서 블로킹 → 스케줄러 고갈
  → 전체 서비스 응답 불능

실수 3: proto 파일 위치와 Java 패키지 설정 혼동
  java_package 없이 proto package만 설정
  → 생성된 클래스가 예상치 못한 위치에 생성됨
  → 프로젝트 구조와 import 경로 불일치로 빌드 실패
```

---

## ✨ 올바른 접근 (After — gRPC 원리를 알고 선택하는 접근)

```
올바른 Channel 생명주기 관리:

// ✅ 올바른 패턴 — Channel을 Bean으로 싱글톤 관리
@Configuration
public class GrpcClientConfig {
    
    @Bean
    public ManagedChannel productServiceChannel() {
        return ManagedChannelBuilder
            .forAddress("product-service", 9090)
            .usePlaintext()
            // .useTransportSecurity()  // 프로덕션에서 TLS
            .keepAliveTime(30, TimeUnit.SECONDS)    // 유휴 연결 유지
            .keepAliveTimeout(5, TimeUnit.SECONDS)
            .build();
    }
    
    @Bean
    public ProductServiceGrpc.ProductServiceBlockingStub productServiceStub(
            ManagedChannel channel) {
        return ProductServiceGrpc.newBlockingStub(channel);
        // Stub은 Channel 위의 경량 래퍼 — Thread-safe, 재사용 가능
    }
}

// 사용: Bean 주입으로 재사용
@Service
public class ProductClientService {
    private final ProductServiceGrpc.ProductServiceBlockingStub stub;
    // Channel은 애플리케이션 종료 시까지 유지
}

환경별 Stub 선택:
  Blocking Stub:  동기 처리, 간단한 비즈니스 로직
  Future Stub:    Java ListenableFuture 기반 비동기
  Async Stub:     StreamObserver 콜백 기반 (스트리밍 필수)
  Reactor Stub:   Spring WebFlux + Mono/Flux (reactive-grpc)
```

---

## 🔬 내부 동작 원리

### 1. .proto 파일 → 코드 생성 전체 흐름

```
.proto 파일 작성:

syntax = "proto3";
package product.v1;

option java_package = "com.example.product.v1";
option java_multiple_files = true;
option java_outer_classname = "ProductProto";

service ProductService {
  rpc GetProduct(GetProductRequest) returns (GetProductResponse);
  rpc ListProducts(ListProductsRequest) returns (stream ProductResponse);
}

message GetProductRequest {
  string product_id = 1;
}

message GetProductResponse {
  string product_id = 1;
  string name = 2;
  int64 price_krw = 3;
  bool in_stock = 4;
}

↓ protoc 실행:

protoc \
  --java_out=./src/main/java \
  --grpc-java_out=./src/main/java \
  --proto_path=./src/main/proto \
  ./src/main/proto/product/v1/product.proto

생성되는 파일:

1. GetProductRequest.java         ← 요청 메시지 클래스 (Builder 포함)
2. GetProductResponse.java        ← 응답 메시지 클래스 (Builder 포함)
3. ListProductsRequest.java       ← 스트리밍 요청 클래스
4. ProductResponse.java           ← 스트리밍 응답 클래스
5. ProductServiceGrpc.java        ← 핵심 파일: Stub + ImplBase 모두 포함

ProductServiceGrpc.java 내부 구조:
  ┌─ ProductServiceGrpc
  │   ├── ProductServiceBlockingStub   ← 동기 클라이언트
  │   ├── ProductServiceFutureStub     ← Future 비동기 클라이언트
  │   ├── ProductServiceStub           ← StreamObserver 비동기 클라이언트
  │   └── ProductServiceImplBase       ← 서버 구현 추상 클래스
```

### 2. protoc 플러그인 동작 방식

```
protoc 컴파일러 동작:

protoc 자체:
  → .proto 파싱
  → AST(Abstract Syntax Tree) 생성
  → FileDescriptorProto(메타데이터) 생성
  → 각 플러그인에 stdin으로 전달 (CodeGeneratorRequest)

플러그인 실행:

protoc-gen-java (메시지 생성):
  CodeGeneratorRequest 수신
  → 메시지 클래스 생성 (equals, hashCode, toString, Builder)
  → Serialization/Deserialization 코드 생성
  → CodeGeneratorResponse 반환 (파일 목록 + 내용)

protoc-gen-grpc-java (서비스 생성):
  CodeGeneratorRequest 수신 (서비스 정의 포함)
  → Stub 클래스 생성 (Blocking/Future/Async)
  → 서버 ImplBase 추상 클래스 생성
  → 메서드 디스크립터 생성 (HTTP/2 경로 정의)
  → CodeGeneratorResponse 반환

메서드 디스크립터 예시:
  MethodDescriptor.forMethod(
      "product.v1.ProductService/GetProduct"  // HTTP/2 :path 헤더 값
      MethodDescriptor.MethodType.UNARY,      // 스트리밍 타입
      ProtoUtils.marshaller(GetProductRequest.getDefaultInstance()),
      ProtoUtils.marshaller(GetProductResponse.getDefaultInstance())
  )
  
  → 이 경로가 HTTP/2 :path 헤더로 서버에 전달됨
  → 서버는 이 경로로 어느 메서드를 호출할지 라우팅
```

### 3. Channel — 연결의 실체

```
ManagedChannel 내부 구조:

ManagedChannel
  │
  ├── TransportFactory
  │     → NettyChannelTransport (기본) 또는 OkHttpChannelTransport (Android)
  │     → 실제 TCP 소켓 + HTTP/2 연결 관리
  │
  ├── LoadBalancer
  │     → 여러 서버 주소 중 어느 서버로 요청을 보낼지 결정
  │     → RoundRobinLoadBalancer (기본), PickFirstLoadBalancer
  │
  ├── NameResolver
  │     → 서버 주소 해석 (DNS, 서비스 디스커버리)
  │     → DNS: product-service → [10.0.1.1:9090, 10.0.1.2:9090]
  │
  └── ClientInterceptors
        → 모든 RPC에 공통으로 적용되는 미들웨어
        → 인증 토큰 추가, 로깅, 추적 ID 주입

Channel 생명주기:
  IDLE       → 연결 없음 (첫 RPC 전)
  CONNECTING → TCP + TLS 핸드쉐이크 진행 중
  READY      → 연결 수립 완료, RPC 처리 가능
  TRANSIENT_FAILURE → 일시적 연결 실패, 재시도 중
  SHUTDOWN   → channel.shutdown() 호출 후

Channel 재사용의 중요성:
  IDLE → READY: TCP 수립(1 RTT) + TLS(1~2 RTT) = 2~3 RTT
  READY → 새 RPC: HTTP/2 스트림 생성 = ~0ms (기존 연결 재사용)
  
  Channel을 매 요청마다 생성: RTT 비용 매 요청 발생
  Channel을 싱글톤으로 유지: RTT 비용 최초 1회만 발생
```

### 4. Stub — 세 가지 종류와 차이

```
세 가지 Stub 비교:

① BlockingStub (동기):

ProductServiceGrpc.ProductServiceBlockingStub stub =
    ProductServiceGrpc.newBlockingStub(channel);

GetProductResponse response = stub
    .withDeadlineAfter(3, TimeUnit.SECONDS)  // 타임아웃 설정
    .getProduct(GetProductRequest.newBuilder()
        .setProductId("prod-1")
        .build());
// 응답이 올 때까지 현재 스레드 블록
// 간단, 직관적 — 동기 코드에서 사용

② FutureStub (Java Future 기반):

ProductServiceGrpc.ProductServiceFutureStub stub =
    ProductServiceGrpc.newFutureStub(channel);

ListenableFuture<GetProductResponse> future = stub
    .getProduct(GetProductRequest.newBuilder()
        .setProductId("prod-1")
        .build());

// 논블로킹 — 콜백 등록 가능
Futures.addCallback(future, new FutureCallback<GetProductResponse>() {
    @Override
    public void onSuccess(GetProductResponse result) {
        // 성공 처리
    }
    @Override
    public void onFailure(Throwable t) {
        // 실패 처리
    }
}, executor);

③ AsyncStub (StreamObserver 콜백):

ProductServiceGrpc.ProductServiceStub stub =
    ProductServiceGrpc.newStub(channel);

stub.getProduct(
    GetProductRequest.newBuilder().setProductId("prod-1").build(),
    new StreamObserver<GetProductResponse>() {
        @Override
        public void onNext(GetProductResponse response) {
            // 응답 수신 (스트리밍에서 여러 번 호출)
        }
        @Override
        public void onError(Throwable t) {
            // 에러 처리
        }
        @Override
        public void onCompleted() {
            // 스트림 완료
        }
    }
);
// 논블로킹, 스트리밍에서 필수

④ ReactorStub (WebFlux/Reactive):

// reactive-grpc 라이브러리 사용
ReactorProductServiceGrpc.ReactorProductServiceStub stub =
    ReactorProductServiceGrpc.newReactorStub(channel);

Mono<GetProductResponse> response = stub.getProduct(
    GetProductRequest.newBuilder().setProductId("prod-1").build()
);
// Reactor의 Mono/Flux로 래핑 — WebFlux와 자연스러운 통합

선택 기준:
  Spring MVC (서블릿)    → BlockingStub
  비동기 필요, Guava 사용 → FutureStub
  스트리밍 구현          → AsyncStub (필수)
  Spring WebFlux        → ReactorStub
```

### 5. 서버 구현 — ImplBase와 요청 처리 흐름

```
서버 구현:

// 1. ImplBase를 상속해 비즈니스 로직 구현
@GrpcService  // grpc-spring-boot-starter 어노테이션
public class ProductServiceImpl
    extends ProductServiceGrpc.ProductServiceImplBase {
    
    private final ProductRepository repository;
    
    @Override
    public void getProduct(
            GetProductRequest request,
            StreamObserver<GetProductResponse> responseObserver) {
        
        // 1. 요청 파라미터 추출
        String productId = request.getProductId();
        
        // 2. 비즈니스 로직
        Product product = repository.findById(productId)
            .orElseThrow(() -> Status.NOT_FOUND
                .withDescription("Product not found: " + productId)
                .asRuntimeException());
        
        // 3. 응답 빌드
        GetProductResponse response = GetProductResponse.newBuilder()
            .setProductId(product.getId())
            .setName(product.getName())
            .setPriceKrw(product.getPrice())
            .setInStock(product.isInStock())
            .build();
        
        // 4. 응답 전송 (Unary: onNext 1번 + onCompleted 1번)
        responseObserver.onNext(response);
        responseObserver.onCompleted();
    }
}

// 2. 서버 시작 (grpc-spring-boot-starter 없이 직접 구성 시)
Server server = ServerBuilder
    .forPort(9090)
    .addService(new ProductServiceImpl())
    .intercept(new AuthInterceptor())     // 서버 인터셉터
    .maxInboundMessageSize(4 * 1024 * 1024)  // 최대 메시지 크기
    .build()
    .start();

요청 처리 흐름 전체:

클라이언트                          서버
    │                                │
    │── HTTP/2 HEADERS Frame ────────▶│
    │   :method = POST               │   → 라우팅: /product.v1.ProductService/GetProduct
    │   :path = /product.v1/GetProduct│
    │   content-type = application/grpc
    │   grpc-timeout = 3S            │
    │                                │
    │── HTTP/2 DATA Frame ───────────▶│
    │   [5바이트 헤더 + Protobuf 바이트] │   → Protobuf 역직렬화
    │   (압축 플래그 1바이트 + 길이 4바이트 + 메시지)
    │                                │
    │                                │── getProduct() 호출
    │                                │── 비즈니스 로직
    │                                │── responseObserver.onNext()
    │                                │
    │◀─ HTTP/2 HEADERS Frame ─────────│
    │   :status = 200                │
    │   content-type = application/grpc
    │                                │
    │◀─ HTTP/2 DATA Frame ────────────│
    │   [5바이트 헤더 + Protobuf 바이트] │   ← 응답 직렬화
    │                                │
    │◀─ HTTP/2 HEADERS Frame ─────────│   ← Trailers (스트림 종료)
    │   grpc-status = 0 (OK)        │
    │   grpc-message =              │
```

---

## 💻 실전 실험

### 실험 1: 생성된 코드 구조 확인

```bash
# Gradle 빌드 후 생성된 파일 확인
./gradlew generateProto

# 생성된 파일 목록
find build/generated/source/proto -name "*.java" | sort
# 출력 예시:
# build/generated/source/proto/main/grpc/com/example/product/v1/ProductServiceGrpc.java
# build/generated/source/proto/main/java/com/example/product/v1/GetProductRequest.java
# build/generated/source/proto/main/java/com/example/product/v1/GetProductResponse.java

# ProductServiceGrpc.java 구조 확인
grep -n "class\|BlockingStub\|FutureStub\|ImplBase" \
  build/generated/source/proto/main/grpc/com/example/product/v1/ProductServiceGrpc.java
```

### 실험 2: Channel 상태 모니터링

```java
// Channel 상태 변화 모니터링
ManagedChannel channel = ManagedChannelBuilder
    .forAddress("product-service", 9090)
    .usePlaintext()
    .build();

// 초기 상태 확인
System.out.println(channel.getState(false));  // IDLE

// 첫 RPC 호출 → CONNECTING → READY
stub.getProduct(request);
System.out.println(channel.getState(false));  // READY

// 상태 변화 콜백 등록
channel.notifyWhenStateChanged(
    ConnectivityState.READY,
    () -> System.out.println("상태 변경됨: " + channel.getState(false))
);
```

### 실험 3: Stub 타입별 동작 비교

```bash
# 서버 시작
docker compose up -d grpc-server

# BlockingStub: 동기 호출 (grpcurl로 시뮬레이션)
time grpcurl -plaintext localhost:9090 \
  product.v1.ProductService/GetProduct \
  -d '{"product_id": "prod-1"}'

# 스트리밍 (AsyncStub 필요): server streaming 호출
grpcurl -plaintext localhost:9090 \
  product.v1.ProductService/ListProducts \
  -d '{"category": "electronics"}'
# 여러 응답이 순서대로 출력됨
```

---

## 📊 성능/비용 비교

```
Channel 생성 비용 vs 재사용:

                        Channel 매 요청 생성    Channel 싱글톤 재사용
────────────────────────────────────────────────────────────
TCP 연결 수립 비용       1~2ms (매 요청)         최초 1회만
TLS 핸드쉐이크           1~2ms (매 요청)         최초 1회만
HTTP/2 Settings 교환    ~0.1ms (매 요청)        최초 1회만
실제 RPC 처리           0.1~1ms               0.1~1ms
────────────────────────────────────────────────────────────
초당 1,000 요청 시      ~4,000ms 연결 비용      ~0ms 연결 비용
메모리 (채널당)          ~수 MB (OS 버퍼 포함)   단일 채널 유지

Stub 생성 비용 (참고):
  Stub 자체는 Channel의 경량 래퍼 (수백 바이트)
  → Thread-safe, 매 요청에서 같은 Stub 재사용 가능
  → Stub은 내부 상태 없음 (Deadline, Metadata는 메서드 체인으로 설정)
```

---

## ⚖️ 트레이드오프

```
코드 생성 방식의 장단점:

장점:
  ① 타입 안전성 — 잘못된 필드 접근은 컴파일 오류
  ② 반복 코드 제거 — 직렬화/역직렬화, HTTP 처리 자동 생성
  ③ 다언어 지원 — 동일 .proto에서 Java, Go, Python, C++ 코드 생성
  ④ IDE 지원 — 자동완성, 리팩터링, 타입 추론

단점:
  ① 빌드 단계 추가 — protoc 실행이 빌드 파이프라인에 필요
  ② proto 파일 관리 — 여러 서비스가 공유하는 proto의 위치와 버전 관리 필요
  ③ 생성된 코드 이해 필요 — Builder 패턴, StreamObserver 등 생소한 API
  ④ 일부 언어 플러그인 품질 차이 — 공식 지원 언어(Java, Go, Python, C++)와 커뮤니티 지원 차이

Channel vs Connection Pool:
  REST: Connection Pool (HikariCP처럼 연결 N개 미리 생성)
  gRPC: 단일 Channel로 HTTP/2 스트림 N개 = 연결 1개로 충분
  → gRPC는 기본적으로 Channel 1개로 수백 개의 동시 RPC 처리 가능
  → 서버 인스턴스가 여러 개라면 인스턴스당 Channel 1개 권장
```

---

## 📌 핵심 정리

```
gRPC 구성 요소 핵심:

.proto → 코드 생성:
  protoc + 플러그인 → 메시지 클래스 + Stub + ImplBase
  메서드 디스크립터: /package.ServiceName/MethodName → HTTP/2 :path

Channel (비싼 리소스):
  TCP + TLS + HTTP/2 연결 관리
  싱글톤으로 유지, 매 요청마다 생성 금지
  상태: IDLE → CONNECTING → READY → TRANSIENT_FAILURE

Stub (경량 래퍼):
  Channel 위에 Thread-safe 래퍼
  BlockingStub: 동기, 스레드 블록
  FutureStub:   ListenableFuture 반환
  AsyncStub:    StreamObserver 콜백 (스트리밍 필수)
  ReactorStub:  Mono/Flux 반환 (WebFlux용)

서버 ImplBase:
  상속 후 메서드 오버라이드
  responseObserver.onNext() → 응답 전송
  responseObserver.onCompleted() → 스트림 종료
  responseObserver.onError() → 에러 전달

HTTP/2 요청 흐름:
  HEADERS Frame (경로, 메타데이터) + DATA Frame (Protobuf 바이트)
  → 응답: HEADERS + DATA + Trailers(grpc-status)
```

---

## 🤔 생각해볼 문제

**Q1.** `ManagedChannel`을 `@Bean`으로 등록하고 `@PreDestroy`로 `shutdown()`을 호출해야 하는 이유는? `shutdown()` 호출 없이 애플리케이션이 종료되면 어떤 일이 발생하는가?

<details>
<summary>해설 보기</summary>

`shutdown()`을 호출하지 않으면:

- **OS 리소스 누수**: TCP 소켓이 즉시 닫히지 않고 `TIME_WAIT` 상태로 남을 수 있습니다
- **진행 중인 RPC 강제 종료**: 처리 중인 요청이 비정상 종료됩니다
- **서버 쪽 불필요한 에러 로그**: 클라이언트 연결이 갑자기 끊기면 서버에서 `RST` 패킷을 받고 에러를 기록합니다

올바른 종료 패턴:
```java
@PreDestroy
public void shutdown() throws InterruptedException {
    channel.shutdown();
    if (!channel.awaitTermination(5, TimeUnit.SECONDS)) {
        channel.shutdownNow();  // 강제 종료
    }
}
```

`shutdown()`은 새 RPC를 거부하고 진행 중인 RPC가 완료될 때까지 기다립니다. `shutdownNow()`는 즉시 강제 종료합니다.

</details>

---

**Q2.** Stub에 `withDeadlineAfter(3, TimeUnit.SECONDS)`를 설정하면 그 Stub 인스턴스에 Deadline이 고정되는가? 그래서 Stub을 매 요청마다 새로 만들어야 하는가?

<details>
<summary>해설 보기</summary>

아닙니다. `withDeadlineAfter()`는 새로운 Stub 인스턴스를 반환합니다. 원본 Stub은 변경되지 않습니다.

```java
// 원본 stub은 Deadline 없음
ProductServiceBlockingStub stub = ProductServiceGrpc.newBlockingStub(channel);

// 새 인스턴스 생성 (원본 불변)
ProductServiceBlockingStub stubWith3s = stub.withDeadlineAfter(3, TimeUnit.SECONDS);

// 각 요청에서 메서드 체인으로 사용
stub.withDeadlineAfter(3, TimeUnit.SECONDS).getProduct(request);
// → 매번 짧은 수명의 Stub 객체가 생성됨 (경량, 문제 없음)
```

따라서 원본 Stub을 싱글톤으로 두고, 각 요청에서 `stub.withDeadlineAfter()`로 임시 Stub을 체인해서 쓰면 됩니다. 새 Channel을 생성하지 않고 Stub만 생성하는 비용은 무시할 수준입니다.

</details>

---

**Q3.** `java_multiple_files = true` 옵션이 없으면 어떻게 되는가? 대규모 프로젝트에서 이 옵션이 중요한 이유는?

<details>
<summary>해설 보기</summary>

`java_multiple_files = false` (기본값)이면:

- 모든 메시지 클래스가 `java_outer_classname`으로 지정한 하나의 파일에 중첩 클래스로 생성됩니다
- 예: `ProductProto.GetProductRequest`, `ProductProto.GetProductResponse`

`java_multiple_files = true`이면:

- 각 메시지가 독립된 파일로 생성됩니다
- 예: `GetProductRequest.java`, `GetProductResponse.java`

대규모 프로젝트에서의 중요성:

- **IDE 성능**: 하나의 거대한 파일보다 여러 파일이 인덱싱/파싱 효율적
- **import 명확성**: `GetProductRequest`를 바로 import할 수 있어 코드 가독성 향상
- **빌드 캐시**: 특정 메시지 변경 시 해당 파일만 재컴파일 (전체 파일 재컴파일 방지)
- **코드 리뷰**: PR에서 변경된 메시지를 파일 단위로 확인 가능

실무에서는 `java_multiple_files = true`를 기본으로 설정하는 것이 권장됩니다.

</details>

---

<div align="center">

**[⬅️ 이전: REST vs gRPC](./01-rest-vs-grpc.md)** | **[홈으로 🏠](../README.md)** | **[다음: HTTP/2 기반 통신 ➡️](./03-http2-multiplexing.md)**

</div>
