# TLS와 mTLS — 서비스 간 Zero Trust

---

## 🎯 핵심 질문

- TLS 1.3 핸드쉐이크는 어떤 과정을 거치고, gRPC에서 왜 필수에 가까운가?
- mTLS에서 클라이언트 인증서는 어떻게 서버 신원 검증에 추가되는가?
- Zero Trust 아키텍처에서 서비스 간 mTLS의 역할은?
- ManagedChannelBuilder에서 TLS를 설정하는 방법은?
- 인증서가 만료되면 어떻게 되고, 핫 리로드는 어떻게 구현하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

TLS/mTLS는 마이크로서비스 간 신뢰할 수 있는 통신을 보장합니다. 네트워크가 신뢰되지 않는 환경(공용 클라우드, Kubernetes)에서 mTLS를 통해 "누가 누구인지 검증"하고, "통신 내용이 암호화"되며, "메시지 무결성이 보장"됩니다. Google, Amazon, Netflix와 같은 대규모 서비스는 모두 서비스 메시도 mTLS를 사용합니다.

---

## 😱 흔한 실수

```java
// 실수 1: 개발 환경 usePlaintext()를 프로덕션에 그대로 사용
// 개발 용 설정
ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
    .usePlaintext()  // OK for local development
    .build();

// 이 코드를 프로덕션으로 배포하면 → 데이터 평문 전송!
// → MITM 공격, 금융 정보 도용 가능


// 실수 2: 자체 서명 인증서를 검증 없이 수락
System.setProperty("javax.net.ssl.trustStore", "/path/to/truststore");
System.setProperty("javax.net.ssl.trustStorePassword", "password");
// 문제: 모든 인증서를 신뢰 → MITM 공격 취약


// 실수 3: 인증서 만료 모니터링 없이 운영
// 인증서 만료 5일 전
// → 아무도 알림을 받지 못함
// → 5일 후 모든 서비스 연결 실패 (장애 발생!)
```

---

## ✨ 올바른 접근

```java
// TLS 설정 (최소한)
ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
    // .usePlaintext() 제거!
    // TLS 자동 설정 (기본값)
    .build();

// mTLS 설정 (권장)
File certChainFile = new File("/etc/grpc/certs/cert.pem");
File privateKeyFile = new File("/etc/grpc/certs/key.pem");
File rootCertFile = new File("/etc/grpc/certs/ca.pem");

SslContextBuilder sslContextBuilder = GrpcSslContexts.forClient()
    .trustManager(rootCertFile)
    .keyManager(certChainFile, privateKeyFile)
    .build();

ManagedChannel channel = NettyChannelBuilder.forAddress(host, port)
    .sslContext(sslContextBuilder)
    .build();

// 서버 설정
File serverCertChain = new File("/etc/grpc/certs/server-cert.pem");
File serverPrivateKey = new File("/etc/grpc/certs/server-key.pem");
File clientCA = new File("/etc/grpc/certs/client-ca.pem");

SslContextBuilder serverSslBuilder = GrpcSslContexts.forServer()
    .keyManager(serverCertChain, serverPrivateKey)
    .trustManager(clientCA)  // 클라이언트 인증서 검증
    .clientAuth(ClientAuth.REQUIRE)  // mTLS 필수
    .build();

Server server = NettyServerBuilder.forPort(50051)
    .sslContext(serverSslBuilder)
    .addService(new MyServiceImpl())
    .build()
    .start();
```

---

## 🔬 내부 동작 원리

### 1. TLS 1.3 핸드쉐이크 흐름

```
┌──────────────────────────────────────────────────────────────┐
│                    TLS 1.3 Handshake                         │
└──────────────────────────────────────────────────────────────┘

CLIENT                                          SERVER
  │                                               │
  ├─── ClientHello ───────────────────────────►│
  │     - Supported Cipher Suites               │
  │     - Supported Versions (TLS 1.3)          │
  │     - Key Share (ephemeral public key)      │
  │                                              │
  │     [암호화 시작]                            │
  │                                              │
  │◄─── ServerHello ───────────────────────────┤
  │     - Selected Cipher Suite                 │
  │     - Key Share (server ephemeral key)      │
  │     [대칭키 협상 완료]                      │
  │                                              │
  │◄─── EncryptedExtensions ────────────────────┤
  │     - Supported Algorithms                  │
  │                                              │
  │◄─── Certificate ────────────────────────────┤
  │     - Server Certificate Chain              │
  │     [모든 데이터가 지금부터 암호화됨]        │
  │                                              │
  │◄─── CertificateVerify ──────────────────────┤
  │     - Handshake Hash Signature              │
  │                                              │
  │◄─── Finished ───────────────────────────────┤
  │     - MAC of all handshake messages         │
  │                                              │
  ├─── Certificate ────────────────────────────►│
  │     - Client Certificate Chain (mTLS)      │
  │                                              │
  ├─── CertificateVerify ──────────────────────►│
  │     - Handshake Hash Signature              │
  │                                              │
  ├─── Finished ───────────────────────────────►│
  │     - MAC of all handshake messages         │
  │                                              │
  ├─ [TLS Connection Established] ──────────────┤
  │                                              │
  ├─── Application Data (encrypted) ──────────►│
  │     [이제 모든 데이터가 암호화됨]            │


TLS 1.3의 특징:
- 1-RTT (이전 TLS 1.2는 2-RTT였음)
- 처음부터 암호화 (ClientHello 직후부터 보호)
- PFS (Perfect Forward Secrecy): 키 손실 시 과거 데이터 안전
```

### 2. mTLS 추가 단계 — 클라이언트 인증서 교환

```
┌─────────────────────────────────────────────────┐
│ 일반 TLS (Server Cert 검증)                    │
└─────────────────────────────────────────────────┘
  CLIENT 검증: "이 인증서가 정말 example.com인가?"
  → 인증서 서명 확인 (신뢰된 CA 체인)
  → 도메인 이름 확인 (CN = example.com)


┌─────────────────────────────────────────────────┐
│ mTLS (Server + Client Cert 검증)               │
└─────────────────────────────────────────────────┘
  SERVER 검증:
  - Client Certificate 수신
  - Client 인증서가 신뢰된 CA로 서명되었는가?
  - Client 인증서가 만료되지 않았는가?
  - Client 인증서가 폐지 목록에 있는가?

  CLIENT 검증: (위와 동일)
  - Server Certificate
  - Server 인증서가 신뢰된 CA로 서명되었는가?
  - Server 인증서가 만료되지 않았는가?

  결과: 양쪽 모두 신원이 확인된 "신뢰할 수 있는 연결"


시나리오 비교:
┌──────────┬─────────────┬──────────────────┐
│          │ TLS (웹)    │ mTLS (서비스 간) │
├──────────┼─────────────┼──────────────────┤
│ 검증 대상│ 서버만      │ 서버 + 클라이언트│
│ 사용 례  │ HTTPS       │ Microservices    │
│ 신뢰도   │ 중간 (공개) │ 높음 (폐쇄)      │
└──────────┴─────────────┴──────────────────┘
```

### 3. ManagedChannelBuilder TLS 설정 (Java 코드)

```java
/**
 * 개발 환경: TLS (자체 서명 인증서)
 */
public class DevTlsClient {
    
    public ManagedChannel createChannel(String host, int port) {
        File certFile = new File("/etc/grpc/certs/dev-ca.pem");
        
        try {
            SslContextBuilder sslContextBuilder = GrpcSslContexts.forClient()
                .trustManager(certFile)  // 자체 서명 CA 신뢰
                .build();
            
            return NettyChannelBuilder.forAddress(host, port)
                .sslContext(sslContextBuilder)
                .build();
        } catch (Exception e) {
            throw new RuntimeException("Failed to create TLS channel", e);
        }
    }
}

/**
 * 프로덕션 환경: mTLS (클라이언트/서버 인증서)
 */
public class ProdMtlsClient {
    
    public ManagedChannel createChannel(String host, int port) {
        File certChainFile = new File("/etc/grpc/certs/client-cert.pem");
        File privateKeyFile = new File("/etc/grpc/certs/client-key.pem");
        File rootCertFile = new File("/etc/grpc/certs/ca-cert.pem");
        
        try {
            SslContextBuilder sslContextBuilder = GrpcSslContexts.forClient()
                .keyManager(certChainFile, privateKeyFile)  // 클라이언트 인증
                .trustManager(rootCertFile)  // 서버 검증
                .build();
            
            return NettyChannelBuilder.forAddress(host, port)
                .sslContext(sslContextBuilder)
                .overrideAuthority(host)  // 도메인 검증
                .build();
        } catch (Exception e) {
            throw new RuntimeException("Failed to create mTLS channel", e);
        }
    }
}

/**
 * 서버: mTLS 설정
 */
public class ProdMtlsServer {
    
    public Server createMtlsServer(int port) {
        File serverCertChain = new File("/etc/grpc/certs/server-cert.pem");
        File serverPrivateKey = new File("/etc/grpc/certs/server-key.pem");
        File clientRootCA = new File("/etc/grpc/certs/client-ca.pem");
        
        try {
            SslContextBuilder sslContextBuilder = GrpcSslContexts.forServer()
                .keyManager(serverCertChain, serverPrivateKey)
                .trustManager(clientRootCA)
                .clientAuth(ClientAuth.REQUIRE)  // 클라이언트 인증 필수
                .build();
            
            return NettyServerBuilder.forPort(port)
                .sslContext(sslContextBuilder)
                .addService(new MyServiceImpl())
                .build();
        } catch (Exception e) {
            throw new RuntimeException("Failed to create mTLS server", e);
        }
    }
}

/**
 * 인증서 만료 모니터링 및 핫 리로드
 */
public class CertificateReloader {
    
    private File certChainFile;
    private File keyFile;
    private File rootCertFile;
    private volatile SslContext cachedSslContext;
    private ScheduledExecutorService executor;
    
    public CertificateReloader(File certChain, File key, File rootCert) {
        this.certChainFile = certChain;
        this.keyFile = key;
        this.rootCertFile = rootCert;
        this.executor = Executors.newScheduledThreadPool(1);
        
        // 초기 로드
        this.cachedSslContext = buildSslContext();
        
        // 24시간마다 인증서 확인 및 갱신
        executor.scheduleAtFixedRate(
            this::reloadCertificateIfNeeded,
            1, 24, TimeUnit.HOURS);
    }
    
    /**
     * 인증서가 변경되면 자동으로 재로드
     */
    private void reloadCertificateIfNeeded() {
        try {
            long certModified = certChainFile.lastModified();
            long keyModified = keyFile.lastModified();
            
            // 파일이 최근에 수정되었으면 재로드
            if (isCertificateNeedsReload(certModified, keyModified)) {
                SslContext newContext = buildSslContext();
                this.cachedSslContext = newContext;
                log.info("Certificate reloaded successfully");
                
                // 알림 (Slack, PagerDuty 등)
                notifyAdmins("Certificate reloaded");
            }
            
            // 만료 날짜 확인
            checkCertificateExpiration(certChainFile);
            
        } catch (Exception e) {
            log.error("Failed to reload certificate", e);
            // 알림: 인증서 재로드 실패
            notifyAdmins("Certificate reload failed: " + e.getMessage());
        }
    }
    
    private boolean isCertificateNeedsReload(long cert, long key) {
        long lastReloadTime = System.currentTimeMillis() - (24 * 60 * 60 * 1000);
        return cert > lastReloadTime || key > lastReloadTime;
    }
    
    private void checkCertificateExpiration(File certFile) {
        try {
            X509Certificate cert = readCertificate(certFile);
            Date expiration = cert.getNotAfter();
            long daysUntilExpiry = (expiration.getTime() - System.currentTimeMillis())
                / (24 * 60 * 60 * 1000);
            
            if (daysUntilExpiry < 30) {
                log.warn("Certificate will expire in {} days", daysUntilExpiry);
                notifyAdmins("Certificate expiring in " + daysUntilExpiry + " days");
            }
        } catch (Exception e) {
            log.error("Failed to check certificate expiration", e);
        }
    }
    
    private SslContext buildSslContext() {
        try {
            return GrpcSslContexts.forServer()
                .keyManager(certChainFile, keyFile)
                .trustManager(rootCertFile)
                .clientAuth(ClientAuth.REQUIRE)
                .build();
        } catch (Exception e) {
            throw new RuntimeException("Failed to build SSL context", e);
        }
    }
    
    public SslContext getSslContext() {
        return cachedSslContext;
    }
    
    private X509Certificate readCertificate(File file) throws Exception {
        CertificateFactory factory = CertificateFactory.getInstance("X.509");
        try (InputStream is = new FileInputStream(file)) {
            return (X509Certificate) factory.generateCertificate(is);
        }
    }
    
    private void notifyAdmins(String message) {
        // Send Slack message, PagerDuty alert, etc.
        log.info("ALERT: {}", message);
    }
    
    public void shutdown() {
        executor.shutdown();
    }
}
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# TLS/mTLS 설정 및 테스트

# 1. 자체 서명 인증서 생성 (개발용)
openssl req -x509 -newkey rsa:2048 -keyout dev-key.pem \
  -out dev-cert.pem -days 365 -nodes

# 2. 서버 인증서 (프로덕션)
openssl req -newkey rsa:2048 -keyout server-key.pem \
  -out server.csr -nodes
openssl x509 -req -days 365 -in server.csr \
  -signkey server-key.pem -out server-cert.pem

# 3. 클라이언트 인증서 (mTLS)
openssl req -newkey rsa:2048 -keyout client-key.pem \
  -out client.csr -nodes
openssl x509 -req -days 365 -in client.csr \
  -signkey client-key.pem -out client-cert.pem

# 4. TLS 서버 시작
java -cp grpc-all.jar:. \
  -Dgrpc.cert=server-cert.pem \
  -Dgrpc.key=server-key.pem \
  TlsServer &

# 5. TLS 클라이언트 테스트
java -cp grpc-all.jar:. \
  -Dgrpc.ca=ca-cert.pem \
  TlsClient
```

---

## 📊 성능/비용 비교

```
평문 vs TLS vs mTLS (1000개 RPC)

┌──────────────┬───────────┬──────────┬────────────┐
│ 프로토콜     │ CPU 사용률│ 지연 증가│ 보안 수준  │
├──────────────┼───────────┼──────────┼────────────┤
│ Plaintext    │ 0% (기준) │ 0ms (기준)│ 없음 (위험)│
│ TLS          │ 5-10%    │ 2-5ms   │ 중간       │
│ mTLS         │ 8-15%    │ 3-7ms   │ 높음       │
└──────────────┴───────────┴──────────┴────────────┘
```

---

## ⚖️ 트레이드오프

```
✅ 장점
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. 데이터 암호화 (평문 전송 방지)
2. 신원 검증 (mTLS)
3. 무결성 보장 (데이터 변조 방지)

❌ 제약사항
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

1. CPU 오버헤드 (8-15%)
2. 인증서 관리 복잡성
3. 지연 증가 (3-7ms)
```

---

## 📌 핵심 정리

```
TLS/mTLS = 신뢰할 수 있는 서비스 간 통신

1. TLS: 서버 검증 + 암호화
2. mTLS: TLS + 클라이언트 검증
3. 1-RTT handshake (빠른 연결)
4. 인증서 관리 (만료, 갱신)
5. Zero Trust 아키텍처의 기반
```

---

## 🤔 생각해볼 문제

### Q1: 클라이언트가 자체 서명 인증서의 CA를 신뢰하도록 설정했는데, 공격자가 같은 CA로 위조 인증서를 만들면 어떻게 되는가?

<details>
<summary>해설 보기</summary>

공격자가 CA의 비공개키를 가지고 있다면 위조 인증서를 만들 수 있습니다. 이는 보안 실패입니다. 해결책:

1. **Certificate Pinning**: 특정 인증서만 신뢰
2. **OCSP Stapling**: 인증서 폐지 확인
3. **Public CA 사용**: 자체 서명 CA 대신 Let's Encrypt 등 사용
</details>

### Q2: 인증서가 만료되는 순간, 모든 gRPC 연결이 즉시 끊어지는가?

<details>
<summary>해설 보기</summary>

아니오. 기존 연결(TCP)은 유지되지만, **새 연결**은 TLS 핸드셰이크에서 실패합니다. Keep-Alive가 없다면 기존 연결도 계속 사용할 수 있어서 위험합니다. 따라서 인증서 갱신 **전에** 리로드하는 것이 중요합니다.

Timeline:
```
T=-1min: 인증서 갱신 (파일 교체)
T=0s: 기존 연결 → 계속 사용
T=5s: 새 연결 시도 → TLS 핸드셰이크 성공 (새 인증서 사용)
T=기존 30분: Keep-Alive timeout → 기존 연결 종료
```
</details>

### Q3: mTLS에서 클라이언트 인증서가 도메인 검증(CN)을 포함하지 않으면, 어떤 속성으로 클라이언트를 식별하는가?

<details>
<summary>해설 보기</summary>

클라이언트 인증서는 도메인을 검증할 필요가 없습니다. 대신 **주체(Subject)** 필드를 사용합니다:

```
Subject: CN=service-a, O=MyCompany, C=US

서버가 검증:
- 이 인증서의 CN이 "service-a"인가?
- 인증서가 신뢰된 CA로 서명되었는가?

결과: "service-a"라는 서비스임을 확인
```
</details>

---

<div align="center">

**[⬅️ 이전: 스트리밍 에러 처리](../streaming-patterns/05-streaming-error-handling.md)** | **[홈으로 🏠](../README.md)** | **[다음: JWT 기반 인증 ➡️](./02-jwt-auth-interceptor.md)**

</div>
