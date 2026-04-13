# 마이그레이션 전략 — REST에서 gRPC로 점진적 전환

---

## 🎯 핵심 질문

- Strangler Fig 패턴을 gRPC 마이그레이션에 어떻게 적용하는가?
- Proto-First 설계는 코드 작성 전에 .proto를 먼저 확정하는 것이 왜 중요한가?
- gRPC-Gateway로 REST와 gRPC를 동시에 지원하는 과도기 운영 방법은?
- Feature Flag를 사용해 문제 발생 시 즉시 REST로 롤백하는 방법은?
- 클라이언트 팀별로 마이그레이션 순서를 어떻게 계획하는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

REST에서 gRPC로 한 번에 전환하면 문제 발생 시 대규모 장애가 발생합니다. Strangler Fig 패턴으로 점진적으로 전환하면 리스크를 최소화하고 문제를 조기에 감지할 수 있습니다.

---

## 😱 흔한 실수 (Before)

```yaml
# 실수 1: 빅뱅 방식 (한 번에 전환)
# 월요일: REST만 사용
# 화요일: 모두 gRPC로 전환
# → 문제 발생 시 전체 서비스 다운

# 실수 2: .proto 설계 없이 구현 시작
# → 중간에 메시지 구조 변경
# → Breaking Change 발생

# 실수 3: 클라이언트 준비 없이 서버 변경
# → 클라이언트가 새로운 엔드포인트 모름
```

---

## ✨ 올바른 접근 (After)

```yaml
# Phase 1: Proto-First 설계 (1주)
# - 팀 간 .proto 리뷰 및 합의
# - Breaking Change 없는 구조 확정

# Phase 2: Strangler Fig (4주)
# - 신규 기능: gRPC로 개발
# - 기존 기능: 병렬 지원 (REST + gRPC)

# Phase 3: Feature Flag 롤아웃 (2주)
# - 일부 클라이언트부터 gRPC 사용
# - 모니터링 및 피드백 수집

# Phase 4: 전체 전환 (진행 중)
# - REST 코드 제거
```

```java
// Phase 1: Proto-First 설계
// user-service.proto
syntax = "proto3";

package io.grpc.examples;

message GetUserRequest {
    string id = 1;
}

message User {
    string id = 1;
    string name = 2;
    string email = 3;
    // 필드 4~10 예약 (향후 확장용)
}

message GetUserResponse {
    User user = 1;
    // 필드 2~10 예약
}

service UserService {
    rpc GetUser (GetUserRequest) 
        returns (GetUserResponse);
}
```

```java
// Phase 2: Strangler Fig 구현
// 기존 REST API (Spring MVC)
@RestController
@RequestMapping("/api/users")
public class UserRestController {
    
    @Autowired
    private UserServiceImpl userService;
    
    @GetMapping("/{id}")
    public UserDTO getUser(@PathVariable String id) {
        User user = userService.getUser(id);
        return UserDTO.fromEntity(user);
    }
}

// 신규 gRPC API (병렬 실행)
@GrpcService
public class UserServiceImpl 
        extends UserServiceGrpc.UserServiceImplBase {
    
    @Autowired
    private UserRepository userRepository;
    
    @Override
    public void getUser(GetUserRequest request,
            StreamObserver<GetUserResponse> observer) {
        
        User user = userRepository.findById(request.getId())
                .orElseThrow(() -> 
                    Status.NOT_FOUND.asException());
        
        GetUserResponse response = 
            GetUserResponse.newBuilder()
                .setUser(toProto(user))
                .build();
        
        observer.onNext(response);
        observer.onCompleted();
    }
    
    private io.grpc.examples.User toProto(User entity) {
        return io.grpc.examples.User.newBuilder()
                .setId(entity.getId())
                .setName(entity.getName())
                .setEmail(entity.getEmail())
                .build();
    }
}
```

```yaml
# Phase 3: Feature Flag 롤아웃
featureFlags:
  useGrpc:
    enabled: true
    clients:
      - mobile-app      # gRPC 사용
      # web-app: REST 사용 (아직)
```

```java
// Feature Flag 기반 라우팅
@Configuration
public class ClientConfiguration {
    
    @Bean
    public UserServiceClient userServiceClient(
            @Value("${featureFlags.useGrpc.enabled}") 
            boolean useGrpc) {
        
        if (useGrpc) {
            return new GrpcUserServiceClient(
                createGrpcStub()
            );
        } else {
            return new RestUserServiceClient(
                new RestTemplate()
            );
        }
    }
}

// 추상화 인터페이스
public interface UserServiceClient {
    User getUser(String id);
}

// gRPC 구현
public class GrpcUserServiceClient 
        implements UserServiceClient {
    
    private final UserServiceGrpc
        .UserServiceBlockingStub stub;
    
    @Override
    public User getUser(String id) {
        GetUserResponse response = stub.getUser(
            GetUserRequest.newBuilder()
                .setId(id)
                .build()
        );
        return User.fromProto(response.getUser());
    }
}

// REST 구현 (기존)
public class RestUserServiceClient 
        implements UserServiceClient {
    
    private final RestTemplate restTemplate;
    
    @Override
    public User getUser(String id) {
        UserDTO dto = restTemplate.getForObject(
            "http://localhost:8080/api/users/" + id,
            UserDTO.class
        );
        return User.fromDTO(dto);
    }
}
```

---

## 🔬 내부 동작 원리

### 1. Strangler Fig 패턴 적용 단계

```
Phase 1: 현재 상태 (REST만)
┌──────────────────────┐
│ API Gateway          │
└──────┬───────────────┘
       │
       ▼
   REST API
   (기존)

Phase 2: Strangler 구축 (병렬)
┌──────────────────────┐
│ API Gateway          │
└──────┬────────────┬──┘
       │            │
       ▼            ▼
   REST API      gRPC API
   (기존)        (신규)

Phase 3: 라우팅 점진적 변경
┌──────────────────────┐
│ API Gateway          │
│ (라우팅 규칙)        │
│ 10% → gRPC           │
└──────┬────────────┬──┘
       │            │
       ▼            ▼
   REST API      gRPC API
   (90%)         (10%)
      ↓            ↑
    모니터링 OK?
      │
      ▼
   30% → gRPC
   ...

Phase 4: 완전 전환
┌──────────────────────┐
│ API Gateway          │
└──────┬───────────────┘
       │
       ▼
   gRPC API
   (100%)
   
   REST API 제거
```

### 2. Proto-First 설계 프로세스

```
1단계: 요구사항 분석
├─ 메시지 구조 결정
├─ RPC 메서드 정의
└─ 향후 확장성 고려

2단계: 팀 리뷰
├─ 프로토콜 엔지니어 검토
├─ API 디자인 리뷰
└─ Breaking Change 점검

3단계: .proto 확정
├─ 모든 팀 승인
└─ 버전 태깅

4단계: 코드 생성
├─ protoc 컴파일
├─ Stub 생성
└─ 개발 시작

5단계: 지속적 진화
├─ 필드 추가 (기존 필드 유지)
├─ 새 RPC 메서드 추가
└─ Deprecated 처리
```

```protobuf
// 좋은 설계 (향후 확장 고려)
message User {
    string id = 1;
    string name = 2;
    string email = 3;
    
    // 필드 4~20 예약 (향후 확장용)
    reserved 4 to 20;
    
    // 향후 추가될 필드들:
    // phone_number = 21;
    // address = 22;
}

// 나쁜 설계 (향후 확장 고려 없음)
message User {
    string id = 1;
    string name = 2;
    string email = 3;
    // 다음 필드는 4, 5, ... 순차적
}
```

### 3. Feature Flag 기반 롤백 전략

```
롤백 시나리오:

Normal Case (gRPC 정상):
Client
  ↓
Feature Flag: useGrpc=true
  ↓
GrpcUserServiceClient
  ↓
gRPC Server (정상)
  ↓
Response 반환

Error Case (gRPC 장애):
Client
  ↓
Feature Flag: useGrpc=false (자동으로 변경)
  ↓
RestUserServiceClient
  ↓
REST Server (기존)
  ↓
Response 반환

자동 롤백:
1. gRPC 에러율 > 1% 감지 (모니터링)
2. Feature Flag 자동 비활성화
3. 클라이언트가 RestUserServiceClient 사용
4. 서비스 복구 (수동 개입 없음)
```

```java
// 자동 롤백 메커니즘
@Component
public class CircuitBreakerService {
    
    private final FeatureFlagService flagService;
    private final MetricsRegistry metricsRegistry;
    
    @Scheduled(fixedRate = 60000)  // 1분마다
    public void monitorErrorRate() {
        
        double errorRate = metricsRegistry
            .getErrorRate("grpc_server_calls_total");
        
        if (errorRate > 0.01) {  // 1% 이상
            log.error("gRPC error rate exceeded: {}", 
                     errorRate);
            
            // 자동 롤백
            flagService.disable("useGrpc");
            
            // 알람
            sendAlert("gRPC service degraded, " +
                     "rolling back to REST");
        }
    }
}
```

### 4. 클라이언트 팀별 마이그레이션 순서 결정

```
순서 결정 기준:

Tier 1: 내부 팀 + 낮은 트래픽
├─ web-app-internal (내부용)
├─ admin-portal (트래픽 낮음)
└─ 위험도: 낮음

Tier 2: 중요 클라이언트 + 중간 트래픽
├─ mobile-app (중요, 중간 트래픽)
├─ partner-api (중요)
└─ 위험도: 중간

Tier 3: 핵심 클라이언트 + 높은 트래픽
├─ web-app (높은 트래픽)
├─ public-api (외부 사용자)
└─ 위험도: 높음

Timeline:
Week 1-2: Tier 1 (10% 트래픽)
Week 3-4: Tier 2 (30% 트래픽)
Week 5-6: Tier 3 (70% 트래픽)
Week 7-8: 전체 (100% 트래픽)

모니터링:
├─ 에러율
├─ 응답 시간
├─ 메모리/CPU
├─ 사용자 피드백
└─ 롤백 자동화
```

---

## 💻 실전 실험

```yaml
# 마이그레이션 체크리스트

Proto-First 설계:
  - [ ] .proto 파일 작성
  - [ ] 팀 리뷰 완료
  - [ ] Breaking Change 체크
  - [ ] 버전 태깅

gRPC-Gateway 설정:
  - [ ] gateway.proto 작성
  - [ ] REST 라우팅 규칙 정의
  - [ ] 테스트

Feature Flag:
  - [ ] Feature Flag 서비스 구축
  - [ ] 클라이언트 인터페이스 추상화
  - [ ] 자동 롤백 로직

모니터링:
  - [ ] 에러율 알람
  - [ ] 성능 대시보드
  - [ ] 로그 수집

Tier별 롤아웃:
  - [ ] Tier 1 (10%) ← 1주
  - [ ] Tier 2 (30%) ← 2주
  - [ ] Tier 3 (70%) ← 3주
  - [ ] 전체 (100%) ← 4주
```

---

## 📊 성능/비용 비교

```
┌──────────────────────────────────────┐
│ 마이그레이션 비용 분석             │
├──────────────────────────────────────┤
│                                       │
│ 항목          Time    Cost   Risk   │
│ ────────────────────────────────────  │
│ Big Bang      1주    낮음   높음    │
│ Strangler     4주    중간   낮음    │
│ Blue-Green    1주    높음   중간    │
│                                       │
│ 권장: Strangler (리스크 최소) │
└──────────────────────────────────────┘
```

---

## ⚖️ 트레이드오프

```
빅뱅 방식:
├─ 장점: 빠름 (1주)
└─ 단점: 위험 높음, 대규모 장애 가능

Strangler Fig:
├─ 장점: 리스크 낮음, 안정성 높음
└─ 단점: 시간 오래 걸림 (4주)

Blue-Green:
├─ 장점: 즉각 롤백 가능
└─ 단점: 더블 인프라 필요, 비용 높음

권장: Strangler Fig
(리스크와 시간의 최적 균형)
```

---

## 📌 핵심 정리

```
1. Proto-First: .proto 먼저 확정
2. Strangler Fig: 점진적 전환
3. Feature Flag: 즉시 롤백
4. Tier별 롤아웃: 위험도 낮춘 순서
5. 모니터링: 자동 롤백 설정
```

---

## 🤔 생각해볼 문제

### Q1: Proto-First 설계가 중요한 이유는?

<details>
<summary>해설 보기</summary>

**정답: Breaking Change 방지 + 팀 간 계약**

```
Code-First (나쁜 예):
1. User 메시지 정의 (필드 1~3)
2. 개발 시작
3. 중간에 address 필드 추가 (필드 4)
4. 다른 팀이 이미 구현한 클라이언트 깨짐
5. 호환성 문제 발생

Proto-First (좋은 예):
1. 모든 필드 미리 예약 (필드 1~30)
2. 팀 리뷰 및 합의
3. 필요시 예약된 필드 사용
4. Breaking Change 없음
5. 버전 호환성 유지
```

</details>

---

### Q2: Strangler Fig 패턴이 빅뱅보다 나은 이유는?

<details>
<summary>해설 보기</summary>

**정답: 문제 조기 감지 + 빠른 롤백**

```
빅뱅:
월요일 자정: gRPC로 100% 전환
월요일 새벽: gRPC 성능 문제 발생
월요일 아침: 전체 서비스 다운

Strangler Fig:
주 1: 10% 클라이언트 → gRPC
주 1 모니터링: OK
주 2: 30% 클라이언트 → gRPC
주 2 모니터링: 에러율 0.5% → Feature Flag 비활성화
      (자동 롤백, 영향도 10%)
주 3: 원인 분석 및 수정
주 4: 다시 10% 트래픽 → 성공
주 8: 100% 전환

결과: 최소한의 피해
```

</details>

---

### Q3: 자동 롤백은 어떻게 구현되는가?

<details>
<summary>해설 보기</summary>

**정답: 메트릭 모니터링 + Feature Flag 자동 변경**

```
구현 단계:

1. 메트릭 수집
   grpc_server_calls_total{status=INTERNAL}

2. 임계값 설정
   if (error_rate > 1%) → trigger

3. Feature Flag 변경
   useGrpc = false (자동)

4. 로그 기록
   "Auto-rollback triggered, error_rate=1.2%"

5. 알람 발송
   Slack, PagerDuty

코드 예:
@Scheduled(fixedRate = 60000)
public void checkAndRollback() {
    if (metricsRegistry.getErrorRate() > 0.01) {
        featureFlagService.disable("useGrpc");
    }
}
```

</details>

---

**[⬅️ 이전: gRPC vs REST 성능 비교](./04-performance-comparison.md)** | **[홈으로 🏠](../README.md)**
