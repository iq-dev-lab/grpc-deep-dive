<div align="center">

# 🔗 gRPC + Protocol Buffers Deep Dive

**"REST로 서비스를 연결하는 것과, 타입 안전한 계약으로 서비스를 연결하고 통신 비용을 줄이는 것은 다르다"**

<br/>

> *"`@FeignClient`에 URL 붙이면 되겠지 — 와 — Protobuf 인코딩이 JSON 대비 왜 68% 작은지, HTTP/2 멀티플렉싱이 HTTP/1.1 Head-of-Line Blocking을 어떻게 없애는지, `.proto` 파일이 어떻게 Breaking Change를 방지하는 API 계약서가 되는지 아는 것의 차이를 만드는 레포"*

Tag-Length-Value 인코딩에서 필드 이름이 사라지고 번호만 남는 원리, 하나의 TCP 연결에서 수십 개의 RPC가 병렬로 흐르는 HTTP/2 스트림 구조, 서버/클라이언트/양방향 스트리밍의 적합한 시나리오,  
Interceptor 체인으로 인증·추적·재시도를 미들웨어처럼 쌓는 방법까지  
**왜 이렇게 설계됐는가** 라는 질문으로 gRPC 내부를 끝까지 파헤칩니다

<br/>

[![GitHub](https://img.shields.io/badge/GitHub-dev--book--lab-181717?style=flat-square&logo=github)](https://github.com/dev-book-lab)
[![gRPC](https://img.shields.io/badge/gRPC-1.x-244c5a?style=flat-square&logo=grpc&logoColor=white)](https://grpc.io/docs/)
[![Protobuf](https://img.shields.io/badge/Protocol_Buffers-proto3-4285F4?style=flat-square&logo=google&logoColor=white)](https://protobuf.dev/)
[![Spring](https://img.shields.io/badge/Spring_Boot-3.x-6DB33F?style=flat-square&logo=spring&logoColor=white)](https://github.com/grpc-ecosystem/grpc-spring)
[![Docs](https://img.shields.io/badge/Docs-38개-blue?style=flat-square&logo=readthedocs&logoColor=white)](./README.md)
[![License](https://img.shields.io/badge/License-MIT-yellow?style=flat-square&logo=opensourceinitiative&logoColor=white)](./LICENSE)

</div>

---

## 🎯 이 레포에 대하여

gRPC에 관한 자료는 넘쳐납니다. 하지만 대부분은 **"어떻게 쓰나"** 에서 멈춥니다.

| 일반 자료 | 이 레포 |
|----------|---------|
| "gRPC는 REST보다 빠릅니다" | Protobuf TLV 인코딩이 JSON 대비 크기를 68% 줄이는 원리, HTTP/2 멀티플렉싱이 동시 RPC 처리량을 높이는 방식, 직렬화·연결 비용의 실제 벤치마크 수치 |
| "`.proto` 파일에 서비스를 정의하세요" | `protoc` 컴파일러가 Stub·Server Skeleton을 생성하는 과정, 필드 번호가 API 계약의 실체인 이유, 필드 번호 재사용이 데이터 손상을 일으키는 시나리오 |
| "Server Streaming을 쓰면 됩니다" | HTTP/2 Frame/Stream/Message 계층에서 스트림 데이터가 어떻게 흐르는지, Backpressure와 Flow Control이 수신 측을 보호하는 원리 |
| "Interceptor로 인증하세요" | ClientInterceptor/ServerInterceptor 체인 구성, 메타데이터로 JWT·TraceID를 전파하는 방식, Deadline이 호출 체인 전체에 전파되는 Context Propagation 원리 |
| "gRPC-Gateway로 REST도 지원됩니다" | gRPC-Web의 브라우저 제약과 해결 방식, Envoy Proxy가 HTTP/1.1 → HTTP/2 변환을 처리하는 구조, L4 로드밸런서가 gRPC에서 효과 없는 이유 |
| "Buf로 proto를 관리하세요" | `buf breaking`으로 Breaking Change를 CI에서 감지하는 방법, Backward/Forward Compatibility 보장 조건, Consumer-Driven Contract Testing 적용 |
| 이론 나열 | 실행 가능한 `grpcurl` 실험 + Wireshark HTTP/2 Frame 캡처 + Protobuf 16진수 바이트 분석 + Docker Compose 환경 + Spring Boot 연결 |

---

## 🚀 빠른 시작

각 챕터의 첫 문서부터 바로 학습을 시작하세요!

[![Fundamentals](https://img.shields.io/badge/🔹_Fundamentals-REST_vs_gRPC-244c5a?style=for-the-badge&logo=grpc&logoColor=white)](./grpc-fundamentals/01-rest-vs-grpc.md)
[![Protobuf](https://img.shields.io/badge/🔹_Protobuf-직렬화_원리-4285F4?style=for-the-badge&logo=google&logoColor=white)](./protocol-buffers/01-tlv-encoding.md)
[![ServiceDesign](https://img.shields.io/badge/🔹_ServiceDesign-.proto_설계_원칙-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./service-design/01-proto-design-principles.md)
[![Streaming](https://img.shields.io/badge/🔹_Streaming-Server_Streaming-FF6B35?style=for-the-badge&logo=grpc&logoColor=white)](./streaming-patterns/01-server-streaming.md)
[![Security](https://img.shields.io/badge/🔹_Security-TLS와_mTLS-DC382D?style=for-the-badge&logo=letsencrypt&logoColor=white)](./security-auth/01-tls-mtls.md)
[![Spring](https://img.shields.io/badge/🔹_Spring-grpc--spring--boot--starter-6DB33F?style=for-the-badge&logo=spring&logoColor=white)](./spring-grpc/01-grpc-spring-boot-starter.md)
[![Operations](https://img.shields.io/badge/🔹_Operations-gRPC_모니터링-FF9500?style=for-the-badge&logo=prometheus&logoColor=white)](./performance-operations/01-grpc-monitoring.md)

---

## 📚 전체 학습 지도

> 💡 각 섹션을 클릭하면 상세 문서 목록이 펼쳐집니다

<br/>

### 🔹 Chapter 1: gRPC 설계 철학과 아키텍처

> **핵심 질문:** REST는 왜 마이크로서비스에서 한계를 드러내는가? gRPC는 어떤 문제를 해결하고, 어떤 비용을 치르는가? HTTP/2 위에서 하나의 연결이 어떻게 수십 개의 RPC를 동시에 처리하는가?

<details>
<summary><b>REST의 한계부터 gRPC 생태계까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. REST vs gRPC — 왜 REST가 마이크로서비스에서 한계를 보이는가](./grpc-fundamentals/01-rest-vs-grpc.md) | 느슨한 스키마 계약으로 Breaking Change가 발생하는 시나리오, JSON 직렬화 비용과 HTTP/1.1 연결 오버헤드, gRPC가 해결하는 문제와 도입 비용(브라우저 미지원, 학습 곡선) |
| [02. gRPC 핵심 구성 요소 — .proto에서 Stub까지](./grpc-fundamentals/02-grpc-core-components.md) | `.proto` 파일 → `protoc` 컴파일 → Stub 생성 → Channel → Server 전체 흐름, `protoc` 플러그인 동작 방식, Blocking/Async/Future Stub의 차이 |
| [03. HTTP/2 기반 통신 — Frame, Stream, Multiplexing](./grpc-fundamentals/03-http2-multiplexing.md) | HTTP/2의 Frame/Stream/Message 3계층 구조, 하나의 TCP 연결에서 여러 RPC가 병렬로 흐르는 멀티플렉싱 원리, HTTP/1.1 Head-of-Line Blocking과의 차이, Wireshark로 HTTP/2 Frame 관찰 |
| [04. gRPC 통신 4가지 패턴 — 언제 무엇을 쓰는가](./grpc-fundamentals/04-grpc-patterns.md) | Unary/Server Streaming/Client Streaming/Bidirectional Streaming 각각의 적합한 시나리오, 잘못된 패턴 선택이 만드는 문제(Polling을 Streaming으로, 스트리밍을 Unary로), `grpcurl`로 각 패턴 직접 실험 |
| [05. gRPC 생태계 — Web, Gateway, Envoy, Reflection](./grpc-fundamentals/05-grpc-ecosystem.md) | gRPC-Web이 브라우저에서 동작하기 위해 Proxy가 필요한 이유, gRPC-Gateway로 REST/gRPC 동시 지원, Envoy Proxy의 gRPC 트랜스코딩, gRPC Reflection으로 스키마 없이 서비스 탐색 |

</details>

<br/>

### 🔹 Chapter 2: Protocol Buffers 완전 분해

> **핵심 질문:** Protobuf는 어떻게 JSON보다 작고 빠른가? 필드 이름이 아닌 번호로 직렬화하는 이유는 무엇인가? 필드를 삭제하거나 추가할 때 왜 일부는 안전하고 일부는 데이터를 손상시키는가?

<details>
<summary><b>TLV 인코딩부터 직렬화 벤치마크까지 (7개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Protobuf 직렬화 원리 — Tag-Length-Value 인코딩](./protocol-buffers/01-tlv-encoding.md) | 필드 번호 + Wire Type으로 구성된 TLV Tag 계산 방식, Varint 압축(작은 숫자는 더 작게 — ZigZag 인코딩), 실제 16진수 바이트로 보는 JSON vs Protobuf 크기 비교 |
| [02. 필드 번호가 API의 본체인 이유](./protocol-buffers/02-field-number-contract.md) | 컴파일 후 필드 이름은 사라지고 번호만 남는 원리, 필드 번호 재사용이 기존 클라이언트 데이터를 손상시키는 시나리오, 번호 관리를 API 설계의 핵심으로 다뤄야 하는 이유 |
| [03. 스칼라 타입과 기본값 — 없는 필드는 전송되지 않는다](./protocol-buffers/03-scalar-types-defaults.md) | int32/int64/string/bool/bytes의 Wire Type과 직렬화 방식, 기본값(0, "", false)인 필드가 페이로드에 포함되지 않는 proto3 원칙, `optional`로 명시적 부재를 표현하는 방법 |
| [04. 복합 타입 — message 중첩, repeated, map, oneof](./protocol-buffers/04-complex-types.md) | 중첩 message의 직렬화 구조(length-delimited), repeated 배열이 packed encoding으로 압축되는 방식, map이 내부적으로 repeated message로 변환되는 원리, oneof로 유니온 타입 표현 |
| [05. Well-Known Types — Timestamp, Any, Struct](./protocol-buffers/05-well-known-types.md) | `google.protobuf.Timestamp`/`Duration`/`Any`/`Struct`/`FieldMask`의 사용 목적과 JSON 변환 규칙, `Any`로 런타임 타입을 담는 방식, `FieldMask`로 부분 업데이트(PATCH) 표현 |
| [06. Protobuf 진화 규칙 — Backward/Forward Compatibility](./protocol-buffers/06-schema-evolution.md) | 신규 필드 추가가 안전한 이유(구 클라이언트는 모르는 필드 무시), 필드 삭제 시 `reserved` 키워드가 필수인 이유, 타입 변경이 위험한 경우와 안전한 경우 구분 |
| [07. 직렬화 성능 측정 — JSON vs Protobuf vs MessagePack](./protocol-buffers/07-serialization-benchmark.md) | 동일 메시지를 JSON/Protobuf/MessagePack으로 직렬화할 때의 크기와 처리 속도 벤치마크, 네트워크 비용 절감 효과 수치화, 언제 Protobuf의 이점이 크고 언제 JSON이 더 실용적인가 |

</details>

<br/>

### 🔹 Chapter 3: gRPC 서비스 설계

> **핵심 질문:** `.proto` 파일을 어떻게 설계해야 Breaking Change 없이 서비스를 진화시킬 수 있는가? gRPC에서 에러, 메타데이터, 타임아웃은 어떻게 다루는가?

<details>
<summary><b>.proto 설계 원칙부터 서비스 계약 관리까지 (6개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. .proto 파일 설계 원칙](./service-design/01-proto-design-principles.md) | 서비스·메시지·필드 명명 규칙(snake_case vs CamelCase), Request/Response Wrapper 패턴으로 하위 호환성을 유지하는 이유, `v1`/`v2` 패키지 버전 관리 전략 |
| [02. 에러 처리 — gRPC Status Code와 google.rpc.Status](./service-design/02-error-handling.md) | gRPC Status Code 13가지 의미와 HTTP 상태 코드 매핑, `google.rpc.Status`로 상세 에러 정보(ErrorDetails)를 전달하는 방법, 클라이언트에서 Status Code를 처리하는 패턴 |
| [03. 메타데이터 — gRPC의 HTTP 헤더](./service-design/03-metadata.md) | 메타데이터가 HTTP/2 헤더 프레임으로 전송되는 방식, 인증 토큰·요청 ID·트레이스 ID를 메타데이터로 전달하는 패턴, Interceptor에서 메타데이터를 추출하고 Context에 전파하는 방법 |
| [04. 데드라인과 타임아웃 — Context Propagation](./service-design/04-deadline-timeout.md) | Deadline이 호출 체인을 따라 전파되는 원리(A→B→C에서 남은 시간이 자동 감소), 타임아웃 없는 gRPC 호출이 스레드 누수를 만드는 시나리오, Deadline Exceeded 에러의 올바른 처리 |
| [05. 로드밸런싱 전략 — 왜 L4가 gRPC에서 효과 없는가](./service-design/05-load-balancing.md) | HTTP/2의 단일 TCP 연결 재사용이 L4 로드밸런서를 우회하는 원리, 클라이언트 사이드 로드밸런싱(Round-Robin/Pick-First), Envoy Proxy로 L7 레벨 로드밸런싱을 해결하는 방법 |
| [06. 서비스 계약 관리 — Buf Schema Registry](./service-design/06-buf-schema-registry.md) | Buf CLI로 중앙화된 `.proto` 관리, `buf breaking`으로 CI에서 Breaking Change 자동 감지, Consumer-Driven Contract Testing으로 서비스 간 계약을 코드로 검증하는 방법 |

</details>

<br/>

### 🔹 Chapter 4: gRPC 스트리밍 패턴

> **핵심 질문:** 스트리밍은 어떤 상황에서 Polling보다 나은가? Backpressure와 Flow Control은 수신 측을 어떻게 보호하는가? 스트림 중간에 에러가 발생하면 어떻게 복구하는가?

<details>
<summary><b>Server Streaming부터 에러 복구까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. Server Streaming — 실시간 데이터 푸시](./streaming-patterns/01-server-streaming.md) | 단일 요청에 여러 응답이 흐르는 HTTP/2 Stream 구조, 실시간 주식 시세·로그 스트리밍 구현, Polling 방식과의 연결 수·지연 비교, Backpressure가 필요한 시나리오 |
| [02. Client Streaming — 배치 데이터 전송](./streaming-patterns/02-client-streaming.md) | 클라이언트가 여러 메시지를 보내고 단일 응답을 받는 패턴, 파일 업로드·배치 데이터 전송 구현, 청크 단위 전송으로 메모리 효율을 높이는 방법, 서버에서 스트림을 닫는 타이밍 |
| [03. Bidirectional Streaming — 채팅과 게임 상태 동기화](./streaming-patterns/03-bidirectional-streaming.md) | 클라이언트·서버가 동시에 메시지를 보내는 양방향 스트림, 채팅 서비스·게임 상태 동기화 구현, 스트림 생명주기(시작·데이터·완료·에러) 관리, WebSocket과의 비교 |
| [04. 스트리밍 흐름 제어 — HTTP/2 Flow Control](./streaming-patterns/04-flow-control.md) | HTTP/2 Window Size가 수신 측 버퍼를 보호하는 원리, WINDOW_UPDATE 프레임으로 수신 가능 용량을 알리는 방식, `initialFlowControlWindow` 설정과 처리량·메모리 트레이드오프 |
| [05. 스트리밍 에러 처리 — 재연결과 부분 실패 복구](./streaming-patterns/05-streaming-error-handling.md) | 스트림 중간 에러 발생 시 Status Code 전달 방식, Exponential Backoff 재연결 전략, 이미 전송된 메시지와 미전송 메시지의 처리 경계, 부분 실패를 응답 메시지로 표현하는 패턴 |

</details>

<br/>

### 🔹 Chapter 5: gRPC 보안과 인증

> **핵심 질문:** 서비스 간 통신에서 신뢰는 어떻게 구축하는가? mTLS는 Zero Trust를 어떻게 구현하는가? Interceptor로 인증·로깅·추적을 어떻게 미들웨어처럼 쌓는가?

<details>
<summary><b>TLS/mTLS부터 Interceptor 체인까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. TLS와 mTLS — 서비스 간 Zero Trust](./security-auth/01-tls-mtls.md) | gRPC에서 TLS가 사실상 필수인 이유(HTTP/2 + Cleartext의 위험성), 서버 인증서 검증 흐름, mTLS로 클라이언트 신원까지 검증하는 Zero Trust 구현, 인증서 갱신 전략 |
| [02. JWT 기반 인증 — Interceptor로 토큰 주입/검증](./security-auth/02-jwt-auth-interceptor.md) | `CallCredentials`로 모든 RPC에 자동으로 JWT를 주입하는 방법, ServerInterceptor에서 Authorization 메타데이터를 추출·검증하는 방식, 만료된 토큰을 스트리밍 중간에 처리하는 패턴 |
| [03. API 키와 서비스 간 인증](./security-auth/03-service-auth.md) | 내부 서비스 간 인증에 적합한 방식(JWT vs mTLS vs API Key), 인증 정보를 gRPC Context에 전파해 비즈니스 로직에서 꺼내 쓰는 패턴, 서비스 계정 기반 인증 구현 |
| [04. Interceptor 체인 — gRPC 미들웨어 패턴](./security-auth/04-interceptor-chain.md) | ClientInterceptor/ServerInterceptor 인터페이스 구조, 로깅·인증·분산 추적·재시도를 체인으로 조합하는 방법, Interceptor 실행 순서와 Context 전달, `grpc-kotlin-stub`의 Coroutine Interceptor |
| [05. 채널 보안 설정 — 개발부터 프로덕션까지](./security-auth/05-channel-security.md) | `ManagedChannelBuilder`의 `usePlaintext()` vs `useTransportSecurity()` 선택 기준, 개발 환경에서 TLS 없이 테스트하는 방법, 프로덕션 인증서 설정과 핀닝 전략 |

</details>

<br/>

### 🔹 Chapter 6: Spring Boot + gRPC 통합

> **핵심 질문:** Spring Boot에서 gRPC 서버와 클라이언트를 어떻게 구성하는가? Spring Security, WebFlux, 예외 처리를 gRPC와 어떻게 통합하는가? 테스트는 어떻게 작성하는가?

<details>
<summary><b>grpc-spring-boot-starter부터 테스트 전략까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. grpc-spring-boot-starter 설정](./spring-grpc/01-grpc-spring-boot-starter.md) | `@GrpcService`로 서비스 구현체 등록, `@GrpcClient`로 Stub 자동 주입, 서버 포트·최대 메시지 크기·keepAlive 설정, application.yml 기반 채널 구성 |
| [02. Spring Security 통합 — gRPC 메서드 레벨 권한](./spring-grpc/02-spring-security.md) | gRPC 서버에 Spring Security를 적용하는 방법, JWT Interceptor와 SecurityContext 연동, `@PreAuthorize`로 메서드 레벨 권한 설정, 인증 실패 시 gRPC Status.UNAUTHENTICATED 반환 |
| [03. 예외 처리 통합 — Spring 예외를 gRPC Status로](./spring-grpc/03-exception-handling.md) | `GrpcExceptionAdvice`로 Spring 예외를 gRPC Status Code로 변환하는 전역 핸들러, `@GrpcExceptionHandler`를 활용한 서비스별 예외 매핑, ErrorDetails를 응답에 포함하는 방법 |
| [04. gRPC + Spring WebFlux — Reactive Stub 통합](./spring-grpc/04-grpc-webflux.md) | Reactive Stub(`ReactorOrderServiceStub`)으로 비동기 스트리밍 처리, gRPC Server Streaming을 `Flux`로, Client Streaming을 `Mono`로 래핑하는 방법, Backpressure와 Reactive Streams 연동 |
| [05. 테스트 전략 — 단위·통합·Mock Stub](./spring-grpc/05-testing-strategy.md) | `GrpcServerExtension`으로 내장 서버를 띄워 단위 테스트, Testcontainers로 Docker 기반 통합 테스트, `InProcessChannel`로 네트워크 없이 gRPC 호출 테스트, Mock Stub으로 클라이언트 코드 격리 테스트 |

</details>

<br/>

### 🔹 Chapter 7: 운영과 성능 튜닝

> **핵심 질문:** gRPC 서비스의 병목은 어디서 발생하는가? 모니터링과 분산 추적은 어떻게 설정하는가? 기존 REST API를 gRPC로 점진적으로 전환하는 전략은 무엇인가?

<details>
<summary><b>모니터링부터 마이그레이션 전략까지 (5개 문서)</b></summary>

<br/>

| 문서 | 다루는 내용 |
|------|------------|
| [01. gRPC 모니터링 — Micrometer + Prometheus](./performance-operations/01-grpc-monitoring.md) | `grpc-spring-boot-starter`의 Micrometer 자동 계측, 요청 수·에러율·p99 응답시간 메트릭 수집, Grafana 대시보드 구성, gRPC Status Code별 에러율 알람 설정 |
| [02. 분산 추적 — OpenTelemetry gRPC Instrumentation](./performance-operations/02-distributed-tracing.md) | OpenTelemetry gRPC Instrumentation으로 서비스 간 TraceID 자동 전파, Baggage로 사용자 ID·요청 컨텍스트를 체인 전체에 전달하는 방법, Jaeger/Zipkin 연동 |
| [03. 연결 관리 튜닝 — keepAlive와 Channel Pool](./performance-operations/03-connection-tuning.md) | `keepAliveTime`/`keepAliveTimeout`으로 유휴 TCP 연결을 유지하는 원리, 프록시·방화벽이 유휴 연결을 끊는 문제와 해결, 최대 동시 연결 수 설정, Channel Pool 구성 필요성 |
| [04. gRPC vs REST 성능 비교 — 수치로 보는 차이](./performance-operations/04-performance-comparison.md) | 동일 비즈니스 로직에서 JSON/REST vs Protobuf/gRPC의 직렬화 크기·시간·TPS·지연시간 벤치마크, gRPC 이점이 두드러지는 워크로드와 그렇지 않은 경우, 측정 방법론과 함정 |
| [05. 마이그레이션 전략 — REST에서 gRPC로 점진적 전환](./performance-operations/05-migration-strategy.md) | Strangler Fig 패턴으로 REST를 gRPC로 점진적 교체, gRPC-Gateway로 REST/gRPC 동시 지원하는 과도기 운영, 클라이언트 하위 호환성 유지, `.proto` 우선 설계로 팀 간 계약을 먼저 확정하는 방법 |

</details>

---

## 🧪 실험 환경

> 모든 실험은 Docker Compose 하나로 재현 가능합니다

```yaml
# docker-compose.yml
services:
  grpc-server:
    build:
      context: ./grpc-server
    ports:
      - "9090:9090"   # gRPC
      - "8080:8080"   # gRPC-Gateway (REST 변환)
    environment:
      SPRING_PROFILES_ACTIVE: docker

  grpc-client:
    build:
      context: ./grpc-client
    depends_on:
      - grpc-server
    ports:
      - "8081:8081"

  envoy:
    image: envoyproxy/envoy:v1.28-latest
    volumes:
      - ./envoy.yaml:/etc/envoy/envoy.yaml
    ports:
      - "9901:9901"   # Admin
      - "10000:10000" # gRPC 프록시

  wireshark:
    image: linuxserver/wireshark:latest
    network_mode: host
    cap_add:
      - NET_ADMIN
```

```bash
# grpcurl로 gRPC 서비스 탐색 및 호출
grpcurl -plaintext localhost:9090 list
grpcurl -plaintext localhost:9090 describe order.v1.OrderService
grpcurl -plaintext localhost:9090 order.v1.OrderService/CreateOrder \
  -d '{"user_id": "user-1", "items": [{"product_id": "prod-1", "quantity": 2}]}'

# 서버 스트리밍 호출
grpcurl -plaintext localhost:9090 order.v1.OrderService/WatchOrderStatus \
  -d '{"order_id": "order-123"}'

# Wireshark HTTP/2 Frame 캡처
# Filter: http2 and tcp.port==9090
```

---

## 🗺️ 선행 학습 연결

이 레포는 다음 Deep Dive 시리즈와 연결됩니다:

| 레포 | 연결 지점 |
|------|----------|
| **network-deep-dive** | HTTP/2 멀티플렉싱 · TLS 핸드쉐이크 · TCP Flow Control — Chapter 1·5의 전제 지식 |
| **msa-deep-dive** | 서비스 간 통신 패턴 · gRPC vs REST 선택 기준 · 서비스 메시 컨텍스트 — Chapter 3·7과 연결 |
| **spring-webflux-deep-dive** | Reactive Streams · Backpressure · Mono/Flux — Chapter 4·6의 WebFlux 통합에서 시너지 |

---

## 📂 디렉토리 구조

```
grpc-deep-dive/
├── grpc-fundamentals/       # Chapter 1: gRPC 설계 철학과 아키텍처 (5개)
├── protocol-buffers/        # Chapter 2: Protocol Buffers 완전 분해 (7개)
├── service-design/          # Chapter 3: gRPC 서비스 설계 (6개)
├── streaming-patterns/      # Chapter 4: gRPC 스트리밍 패턴 (5개)
├── security-auth/           # Chapter 5: gRPC 보안과 인증 (5개)
├── spring-grpc/             # Chapter 6: Spring Boot + gRPC 통합 (5개)
└── performance-operations/  # Chapter 7: 운영과 성능 튜닝 (5개)
```

---

<div align="center">

**총 38개 문서** · REST와의 대비로 시작해 내부 원리로 끝나는 구조

</div>
