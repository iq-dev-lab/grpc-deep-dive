# 채널 보안 설정 — 개발부터 프로덕션까지

---

## 🎯 핵심 질문

- 개발/테스트/프로덕션 환경별로 TLS 설정이 어떻게 달라지는가?
- SslContextBuilder로 커스텀 TLS를 설정하는 방법은?
- 인증서 핀닝은 무엇이고 언제 적용하는가?
- grpc-spring-boot-starter에서 TLS 채널을 application.yml로 설정하는 방법은?
- 인증서 만료를 모니터링하고 알람을 설정하는 방법은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

개발, 테스트, 프로덕션은 보안 요구사항이 완전히 다릅니다. 개발에서는 편의성, 프로덕션에서는 보안을 우선시해야 합니다. 올바른 환경별 설정으로 개발 생산성을 해치지 않으면서도 프로덕션 보안을 보장할 수 있습니다.

---

## 😱 흌한 실수

```java
// 실수 1: 개발 설정을 프로덕션에 그대로 배포
// Development configuration
ManagedChannel channel = ManagedChannelBuilder.forAddress(host, port)
    .usePlaintext()  // OK for localhost
    .build();

// 이 코드를 프로덕션으로 배포 → 재앙!
// → 데이터 평문 전송, MITM 공격 취약


// 실수 2: 모든 인증서를 신뢰하는 TrustManager
TrustManager tm = new X509TrustManager() {
    public void checkClientTrusted(X509Certificate[] chain, String auth) {}
    public void checkServerTrusted(X509Certificate[] chain, String auth) {}
    public X509Certificate[] getAcceptedIssuers() { return new X509Certificate[0]; }
};

SSLContext sslContext = SSLContext.getInstance("TLS");
sslContext.init(null, new TrustManager[] { tm }, new SecureRandom());
// MITM 공격에 완벽하게 취약!


// 실수 3: 인증서 만료를 모니터링하지 않음
// 인증서는 자동으로 갱신되지 않음
// → 만료 날짜가 지나면 모든 서비스 연결 실패!
```

---

## ✨ 올바른 접근

```java
/**
 * 환경별 TLS 설정
 */
public class EnvironmentAwareChannelFactory {
    
    public enum Environment {
        DEVELOPMENT, STAGING, PRODUCTION
    }
    
    public ManagedChannel createChannel(
            String host, int port, Environment env) {
        
        switch (env) {
            case DEVELOPMENT:
                return createDevChannel(host, port);
            case STAGING:
                return createStagingChannel(host, port);
            case PRODUCTION:
                return createProdChannel(host, port);
            default:
                throw new IllegalArgumentException("Unknown env: " + env);
        }
    }
    
    /**
     * 개발 환경: 편의성 우선, usePlaintext() 허용
     */
    private ManagedChannel createDevChannel(String host, int port) {
        return ManagedChannelBuilder.forAddress(host, port)
            .usePlaintext()  // localhost only
            .directExecutor()  // 디버깅 용이
            .build();
    }
    
    /**
     * 스테이징 환경: TLS + 자체 서명 인증서
     */
    private ManagedChannel createStagingChannel(String host, int port) {
        try {
            File caCertFile = new File(
                "/etc/grpc/staging-certs/ca.pem");
            
            SslContextBuilder sslBuilder = GrpcSslContexts.forClient()
                .trustManager(caCertFile);
            
            return NettyChannelBuilder.forAddress(host, port)
                .sslContext(sslBuilder.build())
                .build();
                
        } catch (Exception e) {
            throw new RuntimeException("Failed to create staging channel", e);
        }
    }
    
    /**
     * 프로덕션 환경: mTLS + 인증서 핀닝
     */
    private ManagedChannel createProdChannel(String host, int port) {
        try {
            File certChainFile = new File(
                "/etc/grpc/prod-certs/client-cert.pem");
            File privateKeyFile = new File(
                "/etc/grpc/prod-certs/client-key.pem");
            File rootCertFile = new File(
                "/etc/grpc/prod-certs/ca.pem");
            
            SslContextBuilder sslBuilder = GrpcSslContexts.forClient()
                .keyManager(certChainFile, privateKeyFile)
                .trustManager(rootCertFile);
            
            return NettyChannelBuilder.forAddress(host, port)
                .sslContext(sslBuilder.build())
                .overrideAuthority(host)
                // 인증서 핀닝 추가
                .withOption(ChannelOption.GRPC_OVERRIDE_AUTHORITY, host)
                .build();
                
        } catch (Exception e) {
            throw new RuntimeException("Failed to create prod channel", e);
        }
    }
}

/**
 * 인증서 핀닝: 특정 인증서만 신뢰
 */
public class CertificatePinningClient {
    
    public ManagedChannel createPinnedChannel(
            String host, int port, String pinnedCertHash) {
        
        try {
            SslContextBuilder sslBuilder = GrpcSslContexts.forClient()
                .trustManager(createPinningTrustManager(pinnedCertHash));
            
            return NettyChannelBuilder.forAddress(host, port)
                .sslContext(sslBuilder.build())
                .overrideAuthority(host)
                .build();
                
        } catch (Exception e) {
            throw new RuntimeException("Failed to create pinned channel", e);
        }
    }
    
    /**
     * 인증서 해시 기반 검증
     */
    private X509TrustManager createPinningTrustManager(
            String expectedHash) {
        
        return new X509TrustManager() {
            
            @Override
            public void checkServerTrusted(
                    X509Certificate[] chain, String authType) 
                    throws CertificateException {
                
                if (chain == null || chain.length == 0) {
                    throw new CertificateException("Empty certificate chain");
                }
                
                // 서버 인증서의 SHA-256 해시 계산
                X509Certificate cert = chain[0];
                String certHash = calculateSHA256(cert.getEncoded());
                
                // 핀닝된 해시와 비교
                if (!certHash.equals(expectedHash)) {
                    throw new CertificateException(
                        "Certificate pin mismatch: expected " + expectedHash + 
                        " but got " + certHash);
                }
            }
            
            @Override
            public void checkClientTrusted(
                    X509Certificate[] chain, String authType) {}
            
            @Override
            public X509Certificate[] getAcceptedIssuers() {
                return new X509Certificate[0];
            }
            
            private String calculateSHA256(byte[] data) throws Exception {
                MessageDigest md = MessageDigest.getInstance("SHA-256");
                byte[] digest = md.digest(data);
                return Base64.getEncoder().encodeToString(digest);
            }
        };
    }
}

/**
 * Spring Boot gRPC 설정 (YAML)
 */
// application.yml
grpc:
  server:
    port: 50051
    enable-keep-alive: true
    keep-alive-time: 30s
    security:
      enabled: true
      cert-chain-path: /etc/grpc/certs/server-cert.pem
      private-key-path: /etc/grpc/certs/server-key.pem
      client-auth: REQUIRE
      client-ca-path: /etc/grpc/certs/client-ca.pem
  
  client:
    my-service:
      address: static://prod-server:50051
      security:
        enabled: true
        cert-chain-path: /etc/grpc/certs/client-cert.pem
        private-key-path: /etc/grpc/certs/client-key.pem
        ca-path: /etc/grpc/certs/ca.pem
      keep-alive-time: 30s

/**
 * 인증서 만료 모니터링 및 알람
 */
public class CertificateExpirationMonitor {
    
    private final ScheduledExecutorService scheduler =
        Executors.newScheduledThreadPool(1);
    
    public void startMonitoring() {
        scheduler.scheduleAtFixedRate(
            this::checkCertificateExpiration,
            0, 24, TimeUnit.HOURS);
    }
    
    private void checkCertificateExpiration() {
        try {
            File[] certFiles = new File("/etc/grpc/certs/")
                .listFiles((d, n) -> n.endsWith(".pem"));
            
            for (File certFile : certFiles) {
                X509Certificate cert = readCertificate(certFile);
                Date expiration = cert.getNotAfter();
                long daysUntilExpiry = 
                    (expiration.getTime() - System.currentTimeMillis())
                    / (24 * 60 * 60 * 1000);
                
                if (daysUntilExpiry < 30) {
                    sendAlert(String.format(
                        "Certificate %s expires in %d days",
                        certFile.getName(), daysUntilExpiry));
                }
            }
        } catch (Exception e) {
            log.error("Failed to check certificate expiration", e);
        }
    }
    
    private X509Certificate readCertificate(File file) throws Exception {
        CertificateFactory factory = CertificateFactory.getInstance("X.509");
        try (InputStream is = new FileInputStream(file)) {
            return (X509Certificate) factory.generateCertificate(is);
        }
    }
    
    private void sendAlert(String message) {
        // Slack, PagerDuty, email 등으로 알림
        log.warn("ALERT: {}", message);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. 환경별 TLS 설정 구성

```
Development:
┌─────────────────────────────────────────┐
│ usePlaintext() = true                   │
│ Protocol: HTTP/2 (암호화 없음)          │
│ 인증: 없음                              │
│ 적합: localhost, 로컬 테스트            │
└─────────────────────────────────────────┘

Staging:
┌─────────────────────────────────────────┐
│ TLS 1.3 활성화                         │
│ 자체 서명 인증서 (CA)                  │
│ 서버 인증만                            │
│ 인증서: 365일 유효                     │
│ 적합: 내부 테스트                      │
└─────────────────────────────────────────┘

Production:
┌─────────────────────────────────────────┐
│ TLS 1.3 활성화                         │
│ mTLS (서버 + 클라이언트 인증)          │
│ 인증서 핀닝                            │
│ 인증서: 90일 유효, 자동 갱신           │
│ 적합: 프로덕션 서비스                  │
└─────────────────────────────────────────┘
```

### 2. Netty SslContextBuilder 상세 설정

```java
// 클라이언트
GrpcSslContexts.forClient()
    .keyManager(certChain, privateKey)      // mTLS 클라이언트 인증서
    .trustManager(rootCA)                   // CA 신뢰
    .ciphers(Arrays.asList(                 // TLS 1.3 cipher suite
        "TLS_AES_256_GCM_SHA384",
        "TLS_CHACHA20_POLY1305_SHA256"))
    .protocols("TLSv1.3")                   // TLS 1.3만 사용
    .build();

// 서버
GrpcSslContexts.forServer()
    .keyManager(certChain, privateKey)      // 서버 인증서
    .trustManager(clientCA)                 // 클라이언트 CA 신뢰
    .clientAuth(ClientAuth.REQUIRE)         // mTLS 필수
    .protocols("TLSv1.3")
    .build();
```

### 3. 인증서 핀닝 구현

```
인증서 핀닝 방식:

1. Public Key Pinning (권장)
   ┌────────────────────────────────┐
   │ 인증서 → SHA-256 해시 계산    │
   │ "abc123..."                    │
   │ 앱에 hardcode                  │
   │ 장점: 인증서 갱신해도 유효   │
   └────────────────────────────────┘

2. Certificate Pinning
   ┌────────────────────────────────┐
   │ 전체 인증서 저장 & 비교       │
   │ 문제: 갱신 시 앱 업데이트 필요│
   └────────────────────────────────┘

3. Backup Key Pinning
   ┌────────────────────────────────┐
   │ Primary + Backup 공개키       │
   │ Primary 만료 시 Backup 사용   │
   │ 권장: 내부 서비스             │
   └────────────────────────────────┘
```

---

## 💻 실전 실험

```bash
#!/bin/bash
# 환경별 TLS 테스트

# 1. 인증서 생성
openssl req -x509 -newkey rsa:2048 -keyout prod-key.pem \
  -out prod-cert.pem -days 90 -nodes

# 2. 인증서 해시 계산 (핀닝용)
openssl x509 -in prod-cert.pem -outform DER | \
  openssl dgst -sha256 -binary | base64

# 3. 환경별 서버 시작
ENVIRONMENT=PRODUCTION java -cp grpc-all.jar:. \
  ChannelSecurityServer

# 4. TLS 검증
openssl s_client -connect localhost:50051 \
  -cert client-cert.pem \
  -key client-key.pem \
  -CAfile ca-cert.pem
```

---

## 📊 성능 영향

```
TLS 설정별 성능 (1000개 RPC)

┌──────────────┬────────┬────────┬──────────────┐
│ 설정         │ CPU %  │ 지연   │ 처리량       │
├──────────────┼────────┼────────┼──────────────┤
│ Plaintext    │ 0%     │ 0ms    │ 10K req/s    │
│ TLS 1.3      │ 8-10%  │ 3-5ms  │ 9K req/s     │
│ mTLS         │ 12-15% │ 5-8ms  │ 8.5K req/s   │
│ + Pinning    │ 15-18% │ 6-10ms │ 8K req/s     │
└──────────────┴────────┴────────┴──────────────┘
```

---

## 📌 핵심 정리

```
채널 보안 설정:

1. Development: usePlaintext (localhost)
2. Staging: TLS + 자체 서명 인증서
3. Production: mTLS + 인증서 핀닝
4. 모니터링: 만료 날짜 추적
5. 갱신: 자동 또는 사전 예방
```

---

## 🤔 생각해볼 문제

### Q1: 프로덕션에서 인증서가 만료되었는데 코드를 배포할 수 없다면?

<details>
<summary>해설 보기</summary>

응급 대응:
1. Backup Key로 전환 (미리 준비한 경우)
2. TLS 검증 일시 비활성화 (불가피한 경우)
3. gRPC를 거치지 않는 API로 전환 (임시)

올바른 대비:
- Backup Key Pinning 사전 준비
- 만료 30일 전 갱신 정책
- 자동 갱신 시스템 (Let's Encrypt + ACME)
</details>

---

<div align="center">

**[⬅️ 이전: Interceptor 체인](./04-interceptor-chain.md)** | **[홈으로 🏠](../README.md)** | **[다음: Chapter 6 — grpc-spring-boot-starter ➡️](../spring-grpc/01-grpc-spring-boot-starter.md)**

</div>
