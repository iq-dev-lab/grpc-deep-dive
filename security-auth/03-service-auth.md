# API 키와 서비스 간 인증

---

## 🎯 핵심 질문

- 서비스 간 인증에서 JWT, mTLS, API Key, SPIFFE 중 어떤 기준으로 선택하는가?
- 서비스 계정(Service Account)이란 무엇이고 어떻게 활용하는가?
- Kubernetes에서 Service Account Token을 gRPC 인증에 연결하는 방법은?
- SPIFFE/SPIRE가 서비스 메시도에서 신원 관리를 어떻게 자동화하는가?
- API Key를 안전하게 전달하고 관리하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이크로서비스 아키텍처에서는 서비스 간 신뢰할 수 있는 통신이 필수입니다. Kubernetes 환경에서는 Service Account로 각 파드의 신원을 자동으로 관리할 수 있고, 더 나아가 SPIFFE/SPIRE로 인증서를 자동 생성/갱신하여 운영 부담을 최소화할 수 있습니다. 올바른 선택은 비용 절감, 보안 강화, 운영 효율성을 동시에 달성합니다.

---

## 😱 흔한 실수

```java
// 실수 1: API Key를 코드에 하드코딩
public class ApiClient {
    private static final String API_KEY = "sk_live_123456789";  // 위험!
    
    // 문제: Git 저장소에 노출되면 즉시 탈취됨
}


// 실수 2: 서비스마다 다른 인증 방식 혼용
// Service A → JWT (토큰 기반)
// Service B → mTLS (인증서 기반)
// Service C → API Key (키 기반)
// 결과: 관리 복잡도 증가, 보안 일관성 없음


// 실수 3: Kubernetes Service Account를 사용하지 않음
// 각 서비스에 hardcode된 API Key
// 파드 재시작, 마이그레이션 → 수동으로 키 교체 필요
```

---

## ✨ 올바른 접근

```java
/**
 * API Key 메타데이터 전달 (안전한 방식)
 */
public class ApiKeyClient {
    
    public ManagedChannel createChannelWithApiKey(
            String host, int port, String apiKey) {
        
        // CallCredentials로 API Key 자동 주입
        return ClientInterceptors.intercept(
            ManagedChannelBuilder.forAddress(host, port)
                .usePlaintext()
                .build(),
            new ClientInterceptor() {
                @Override
                public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
                        MethodDescriptor<ReqT, RespT> method,
                        CallOptions callOptions,
                        Channel next) {
                    
                    return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(
                        next.newCall(method, callOptions)) {
                        
                        @Override
                        public void start(Listener<RespT> responseListener,
                                Metadata headers) {
                            
                            // API Key를 메타데이터에 추가
                            Metadata.Key<String> apiKeyKey = 
                                Metadata.Key.of("x-api-key",
                                    Metadata.ASCII_STRING_MARSHALLER);
                            headers.put(apiKeyKey, apiKey);
                            
                            super.start(responseListener, headers);
                        }
                    };
                }
            });
    }
}

/**
 * Kubernetes Service Account 활용
 */
public class K8sServiceAccountClient {
    
    public ManagedChannel createK8sAuthenticatedChannel(
            String host, int port) throws IOException {
        
        // K8s Service Account Token 읽기
        String tokenPath = "/var/run/secrets/kubernetes.io/serviceaccount/token";
        String token = new String(Files.readAllBytes(Paths.get(tokenPath)));
        
        // Token을 JWT처럼 사용
        return ClientInterceptors.intercept(
            ManagedChannelBuilder.forAddress(host, port)
                .usePlaintext()
                .build(),
            new ClientInterceptor() {
                @Override
                public <ReqT, RespT> ClientCall<ReqT, RespT> interceptCall(
                        MethodDescriptor<ReqT, RespT> method,
                        CallOptions callOptions,
                        Channel next) {
                    
                    return new ForwardingClientCall.SimpleForwardingClientCall<ReqT, RespT>(
                        next.newCall(method, callOptions)) {
                        
                        @Override
                        public void start(Listener<RespT> responseListener,
                                Metadata headers) {
                            
                            Metadata.Key<String> authKey = 
                                Metadata.Key.of("authorization",
                                    Metadata.ASCII_STRING_MARSHALLER);
                            headers.put(authKey, "Bearer " + token);
                            
                            super.start(responseListener, headers);
                        }
                    };
                }
            });
    }
}

/**
 * SPIFFE/SPIRE를 통한 자동 인증서 관리
 */
public class SpiffeClient {
    
    /**
     * SPIFFE URI로 신뢰 가능한 서비스 식별
     * spiffe://example.com/service/my-service
     */
    public ManagedChannel createSpiffeSecureChannel(
            String host, int port) throws Exception {
        
        // SPIRE Agent와 통신하여 인증서 획득
        // (자동 갱신, mTLS 기반)
        
        String spiffeTrustDomain = "example.com";
        String serviceName = "my-service";
        
        SslContextBuilder sslBuilder = GrpcSslContexts.forClient()
            .trustManager(loadSpiffeTrustBundle())
            .keyManager(loadSpiffer Certificate());
        
        return NettyChannelBuilder.forAddress(host, port)
            .sslContext(sslBuilder.build())
            .overrideAuthority("spiffe://" + spiffeTrustDomain + 
                "/service/" + serviceName)
            .build();
    }
    
    private File loadSpiffeTrustBundle() throws Exception {
        // SPIRE Agent socket을 통해 SVID (SPIFFE Verifiable Identity) 획득
        // /tmp/spire-agent/public/bundle.crt
        return new File("/tmp/spire-agent/public/bundle.crt");
    }
    
    private File loadSpirreCertificate() throws Exception {
        // SPIRE Agent socket을 통해 인증서 자동 갱신
        // /tmp/spire-agent/public/svid.crt
        return new File("/tmp/spire-agent/public/svid.crt");
    }
}

/**
 * 서비스 간 인증 선택 기준
 */
public class AuthenticationStrategy {
    
    /**
     * 선택 로직:
     * 1. Kubernetes 환경 → Service Account Token + SPIRE
     * 2. 내부 마이크로서비스 → mTLS (인증서 자동 관리)
     * 3. 외부 API / 사용자 → JWT (토큰)
     * 4. 레거시 시스템 → API Key (필요시)
     */
    public enum AuthMethod {
        // Kubernetes 환경 최적화
        K8S_SERVICE_ACCOUNT,  // K8s Token 자동 주입
        SPIFFE_SPIRE,        // 인증서 자동 생성/갱신
        
        // 일반 환경
        MTLS,                // 인증서 기반 (수동 관리)
        JWT,                 // 토큰 기반
        API_KEY,             // 키 기반 (레거시)
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 서비스 간 인증 방식 비교표

```
┌─────────────────┬───────────┬──────────┬──────────┬──────────┐
│ 방식             │ 배포 난이│ 갱신     │ 추적성   │ 비용     │
├─────────────────┼───────────┼──────────┼──────────┼──────────┤
│ JWT             │ 중간      │ 수동 (1h)│ 중간     │ 낮음     │
│ mTLS            │ 높음      │ 수동 (1y)│ 높음     │ 중간     │
│ API Key         │ 낮음      │ 수동     │ 낮음     │ 낮음     │
│ SPIFFE/SPIRE    │ 높음 초기 │ 자동     │ 높음     │ 중간     │
│ K8s Svc Account │ 낮음      │ 자동     │ 중간     │ 낮음     │
└─────────────────┴───────────┴──────────┴──────────┴──────────┘


상황별 선택:

K8s + 마이크로서비스:
  Service Account Token (자동) + SPIRE (갱신 자동)

온프레미스 + 여러 팀:
  mTLS + 중앙 인증서 관리 시스템

레거시 시스템 통합:
  API Key (점진적 마이그레이션)

모바일 앱 + API:
  JWT (stateless, 만료 가능)
```

### 2. API Key 메타데이터 전달 구현 (Java 코드)

```java
// 이미 위의 ApiKeyClient 클래스에 포함됨
// 핵심:
// - CallCredentials 또는 ClientInterceptor로 자동 주입
// - 환경 변수에서 읽기 (Secret 관리 도구와 연동)
// - gRPC 메타데이터 레이어에서만 전달 (Proto 오염 X)
```

### 3. Kubernetes Service Account JWT 활용 흐름

```
K8s Pod 생성:
┌────────────────────────────────────────┐
│ apiVersion: v1                         │
│ kind: ServiceAccount                   │
│ metadata:                              │
│   name: my-service-account             │
│   namespace: default                   │
└────────────────────────────────────────┘
         ↓
K8s Admission Controller가 Token 자동 주입:
┌────────────────────────────────────────┐
│ Pod에 Secret Volume 마운트:            │
│ /var/run/secrets/kubernetes.io/        │
│   serviceaccount/                      │
│   ├── token                           │
│   ├── ca.crt                          │
│   └── namespace                       │
└────────────────────────────────────────┘
         ↓
애플리케이션이 Token 사용:
┌────────────────────────────────────────┐
│ String token = readFile(               │
│   "/var/run/secrets/kubernetes.io/"    │
│   "serviceaccount/token")              │
│                                        │
│ headers.put("authorization",           │
│   "Bearer " + token)                   │
└────────────────────────────────────────┘
         ↓
다른 Pod이 Token 검증:
┌────────────────────────────────────────┐
│ K8s API Server에서 공개키 로드         │
│ JWT 검증 (HMAC SHA-256)                │
│ "이 token은 정말 my-service-account인가?"│
└────────────────────────────────────────┘
```

### 4. SPIFFE/SPIRE 개념 — 서비스 SVID와 자동 인증서 발급

```
SPIFFE (Secure Production Identity Framework For Everyone)
  표준화된 서비스 ID 형식: spiffe://trust-domain/service/name

SPIRE (SPIFFE Runtime Environment)
  구현체: 자동 인증서 생성, 갱신, 배포


흐름:
┌────────────────────────────────────────┐
│ 1. Pod 생성                            │
│    spiffe://example.com/service/web-api│
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ 2. SPIRE Agent (Pod 내)                │
│    SPIRE Server에 "이건 web-api Pod"  │
│    신청                                │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ 3. SPIRE Server 검증                  │
│    - Attestation (증명)                │
│      "K8s API로 Pod 신분 확인"        │
│    - SVID 발급 (CA로 서명된 인증서)   │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ 4. Pod에 인증서 배포                   │
│    /tmp/spire-agent/public/            │
│    ├── svid.crt (인증서, 24시간 유효) │
│    ├── svid.key (비공개키)             │
│    └── bundle.crt (신뢰 체인)         │
└────────────────────────────────────────┘
         ↓
┌────────────────────────────────────────┐
│ 5. Pod가 인증서로 mTLS 통신           │
│    (인증서 갱신은 SPIRE 자동)         │
└────────────────────────────────────────┘


이점:
- 자동 갱신 (인증서 만료 걱정 없음)
- 중앙 관리 (SPIRE Server)
- K8s와 통합 (Attestation)
- Zero Trust (모든 서비스 ID 검증)
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# 서비스 간 인증 설정 테스트

# 1. Kubernetes Service Account 확인
kubectl create serviceaccount my-service
kubectl describe sa my-service

# 2. Token 추출
TOKEN=$(kubectl get secret $(kubectl get secret \
  -o jsonpath='{.items[0].metadata.name}') \
  -o jsonpath='{.data.token}' | base64 -d)

# 3. 다른 서비스에 gRPC 요청 (Token 포함)
grpcurl -H "authorization: Bearer $TOKEN" \
  -plaintext \
  localhost:50051 MyService/MyMethod

# 4. SPIRE 설치 및 테스트 (선택)
# kubectl apply -f https://raw.githubusercontent.com/spiffe/spire/main/...
```

---

## 📊 성능/비용 비교

```
5년간 운영 비용 기준 (100개 마이크로서비스)

┌────────────────────┬──────────┬────────────┬──────────────┐
│ 방식               │ 초기 설정│ 연간 관리  │ 사건 대응    │
├────────────────────┼──────────┼────────────┼──────────────┤
│ API Key (수동)     │ 낮음     │ 높음 (관리)│ 높음 (교체)  │
│ JWT (수동)         │ 중간     │ 높음 (갱신)│ 중간 (폐지)  │
│ mTLS (수동)        │ 높음     │ 높음 (갱신)│ 높음 (재발급)│
│ SPIFFE/SPIRE       │ 매우높음 │ 낮음 (자동)│ 낮음 (자동)  │
│ K8s Svc Account    │ 낮음     │ 낮음 (자동)│ 낮음 (자동)  │
└────────────────────┴──────────┴────────────┴──────────────┘
```

---

## 📌 핵심 정리

```
서비스 간 인증 선택:

1. K8s 환경 → Service Account Token
2. mTLS 필수 → SPIFFE/SPIRE (자동화)
3. 간단함 → API Key (환경 변수)
4. 사용자 → JWT (만료 가능)
5. 혼합 사용 시 → 중앙 정책으로 관리
```

---

## 🤔 생각해볼 문제

### Q1: API Key가 유출되면 즉시 탈취되지만, JWT는 만료되므로 안전한가?

<details>
<summary>해설 보기</summary>

부분적으로 맞습니다. JWT는:
- 장점: 만료 시간이 있어 자동 무효화
- 단점: 만료까지 유효 (예: 1시간)

따라서 **Token Revocation List (TRL)**를 추가하여, 만료 전에도 폐지 가능하게 해야 합니다.

```java
// 고급: JWT 폐지 목록 관리
public class JwtRevocationManager {
    private final Set<String> revokedTokens = 
        Collections.synchronizedSet(new HashSet<>());
    
    public void revoke(String token) {
        revokedTokens.add(jti(token));  // JWT ID로 관리
    }
    
    public boolean isRevoked(String token) {
        return revokedTokens.contains(jti(token));
    }
}
```
</details>

---

<div align="center">

**[⬅️ 이전: JWT 기반 인증](./02-jwt-auth-interceptor.md)** | **[홈으로 🏠](../README.md)** | **[다음: Interceptor 체인 ➡️](./04-interceptor-chain.md)**

</div>
