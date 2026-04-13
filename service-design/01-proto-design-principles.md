# .proto 파일 설계 원칙

---

## 🎯 핵심 질문

- 서비스 이름, 메서드 이름, 필드 이름의 명명 규칙은?
- 왜 모든 RPC에 전용 Request/Response 메시지를 만들까?
- 패키지 구조는 어떻게 해야 할까?
- buf lint 규칙은 무엇인가?
- proto 파일 버전 관리는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

proto 파일은 API 계약입니다. 명명 규칙을 어기면 팀원이 코드를 읽기 어렵고, Request/Response를 생략하면 향후 확장이 불가능하며, 버전 관리 없이는 API 변경이 데이터 손상을 초래합니다. Google이 제시한 설계 원칙을 따르면 수천 개의 마이크로서비스도 일관되게 관리할 수 있습니다.

---

## 😱 흔한 실수 (Before — 일관성 없는 설계)

```
// 실수 1: 메서드 이름이 snake_case
service UserService {
  rpc get_user (GetUserRequest) returns (User);         // ❌ snake_case
  rpc create_user (CreateUserRequest) returns (User);   // ❌
  rpc list_all_users (ListRequest) returns (UserList);  // ❌
}

// 생성된 Java 코드:
// get_user() → java의 getUser()로 자동 변환
// → 복잡한 이름 변환, 대소문자 불일치

// 실수 2: Request/Response 생략
service UserService {
  rpc GetUser (int32) returns (User);  // int32을 직접? 위험!
  
  // 문제: 향후 user_id 외에 필드 추가 불가능
  // rpc GetUser (GetUserRequest) returns (User)  이렇게 해야 확장 가능
}

// 실수 3: 패키지 구조 무시
// user_service.proto → package: myapp.services.user_service
// order_service.proto → package: services.order
// notification_service.proto → package: notif

// 패키지 일관성 없음 → 클라이언트 import 혼란

// 실수 4: 필드명이 서로 다름
message CreateUserRequest {
  string email = 1;
  string full_name = 2;      // full_name?
}

message UpdateUserRequest {
  string email_address = 1;   // email_address?
  string name = 2;            // name? (full_name과 다름)
}

// 같은 개념을 다르게 표현 → 클라이언트 혼동

// 실수 5: buf lint 무시
// proto 파일을 자유롭게 작성
// → 팀마다 다른 스타일
// → API 일관성 깨짐
```

---

## ✨ 올바른 접근 (After — 일관된 설계 원칙)

```
// 올바른 접근 1: Google 명명 규칙
service UserService {
  // 메서드: CamelCase (시작 대문자)
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
  rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
  rpc DeleteUser (DeleteUserRequest) returns (google.protobuf.Empty);
  
  // 메서드명은 동사 + 명사
  // Get, Create, List, Delete, Update, Search, ...
}

message GetUserRequest {
  string user_id = 1;         // Request는 구체적
}

message GetUserResponse {
  User user = 1;              // Response도 명시적
}

message User {
  string user_id = 1;
  string email = 2;
  string full_name = 3;       // 필드명: snake_case (소문자)
  int32 age = 4;
  // 필드명은 단수형, snake_case 고정
}

// 올바른 접근 2: 일관된 필드명
message CreateUserRequest {
  string email = 1;           // email (email_address 아님)
  string full_name = 2;       // full_name (name 아님)
  optional int32 age = 3;     // 선택사항
}

message UpdateUserRequest {
  string user_id = 1;         // 항상 필요
  string email = 2;           // 같은 이름 (email_address 아님)
  string full_name = 3;       // 같은 이름 (name 아님)
  optional int32 age = 4;
  google.protobuf.FieldMask update_mask = 5;  // PATCH 표현
}

// 올바른 접근 3: 패키지 버전 관리
// proto 파일 구조:
// protos/
// ├── google/protobuf/  (외부)
// ├── myapp/
// │   ├── v1/
// │   │   ├── user_service.proto
// │   │   ├── order_service.proto
// │   │   └── common.proto
// │   └── v2/
// │       ├── user_service.proto
// │       └── common.proto

package myapp.v1;

service UserService {
  rpc GetUser (GetUserRequest) returns (User);
}

// 올바른 접근 4: buf.yaml 설정
version: v1
build:
  roots:
    - protos
lint:
  rules:
    enabled:
      - FIELD_NAMES_LOWER_SNAKE_CASE
      - SERVICE_NAMES_END_WITH_SERVICE
      - RPC_NAMES_CASE
  ignore_patterns:
    - google/protobuf  // 외부 proto는 제외
```

---

## 🔬 내부 동작 원리

### Google 명명 규칙

```
Proto 요소별 명명 규칙:
┌───────────────────┬──────────────┬─────────────────────┐
│요소               │형식          │예시                 │
├───────────────────┼──────────────┼─────────────────────┤
│Package            │snake_case    │myapp.v1.user_service│
│Service            │CamelCase     │UserService          │
│메서드             │CamelCase     │GetUser, ListUsers   │
│메시지             │CamelCase     │CreateUserRequest    │
│필드               │snake_case    │user_id, full_name  │
│Enum               │UPPER_SNAKE   │ACTIVE, INACTIVE    │
│Enum값             │UPPER_SNAKE   │ROLE_ADMIN          │
└───────────────────┴──────────────┴─────────────────────┘

생성된 코드에 미치는 영향:

Proto:
  message User {
    string user_id = 1;        // snake_case
    string full_name = 2;
  }

Java:
  public String getUserId()     // CamelCase로 변환
  public String getFullName()
  public Builder setUserId()

Python:
  user.user_id                  // snake_case 유지
  user.full_name

Go:
  user.UserId                   // PascalCase로 변환
  user.FullName

JavaScript:
  user.user_id                  // snake_case (기본)
  user.fullName                 // camelCase (option)

규칙을 따르면:
  ✓ 각 언어의 관례 자동 적용
  ✓ 생성 코드 일관성
  ✓ IDE 자동완성 정확
```

### Request/Response 패턴

```
왜 모든 RPC에 전용 Request/Response?

Anti-pattern (위험):
  rpc GetUser(int32) returns (User)
  
  문제 1: 향후 필드 추가 불가능
    → int32에 다른 필드 추가? (불가능, 타입 고정)
  
  문제 2: API 진화 제약
    구 클라이언트: GetUser(123)
    신 서버: GetUserRequest{user_id:123, include_deleted:true} 필요
    → 호환성 깨짐!
  
  문제 3: 선택적 필드 표현 어려움
    GetUser(int32) → int32만 가능
    int32가 0일 때: "ID 0"인가? "미설정"인가?

Pattern (권장):
  message GetUserRequest {
    string user_id = 1;           // 필수
    bool include_inactive = 2;    // 옵션
    repeated string fields = 3;   // projection
  }
  
  rpc GetUser(GetUserRequest) returns (GetUserResponse);
  
  message GetUserResponse {
    User user = 1;
    map<string, string> metadata = 2;
  }

이점:
  ✓ 필드 추가 자유로움 (GetUserRequest에 새 필드)
  ✓ 선택적 필드 표현 (optional int32 limit = 4)
  ✓ 응답 메타데이터 추가 가능
  ✓ 각 RPC마다 고유 요청 (혼동 방지)

패턴화:
  1. 단일 리소스 조회:
     GetUserRequest {user_id}
     GetUserResponse {user}
  
  2. 목록 조회:
     ListUsersRequest {page_token, page_size}
     ListUsersResponse {users[], next_page_token}
  
  3. 생성:
     CreateUserRequest {email, full_name}
     CreateUserResponse {user}
  
  4. 업데이트:
     UpdateUserRequest {user_id, user, update_mask}
     UpdateUserResponse {user}
  
  5. 삭제:
     DeleteUserRequest {user_id}
     DeleteUserResponse {}  (또는 google.protobuf.Empty)
```

### 패키지 구조와 버전 관리

```
단일 버전 구조:
  protos/
  └── myapp/
      ├── common.proto          // 공유 타입
      ├── user_service.proto    // 서비스
      ├── order_service.proto
      └── payment_service.proto
  
  package myapp.v1;
  
  단점: v2 필요시 migration 어려움

멀티 버전 구조 (권장):
  protos/
  └── myapp/
      ├── common.proto          // 버전 무관
      ├── v1/
      │   ├── user_service.proto
      │   ├── order_service.proto
      │   └── common_v1.proto   // 버전별 custom
      └── v2/
          ├── user_service.proto (신 API)
          ├── order_service.proto
          └── common_v2.proto

  package myapp.v1;  (v1/user_service.proto)
  package myapp.v2;  (v2/user_service.proto)

서버 구현:
  // 양쪽 버전 지원
  @GrpcService
  public class UserServiceV1Impl 
    extends UserServiceGrpc.UserServiceImplBase {
    // v1 구현
  }
  
  @GrpcService
  public class UserServiceV2Impl 
    extends UserServiceV2Grpc.UserServiceImplBase {
    // v2 구현 (확장된 필드)
  }

클라이언트:
  // v1 클라이언트
  UserServiceGrpc.UserServiceStub stub = ...
  stub.getUser(GetUserRequest.newBuilder()
    .setUserId("U1")
    .build());
  
  // v2 클라이언트
  UserServiceV2Grpc.UserServiceStub stubV2 = ...
  stubV2.getUser(v2.GetUserRequest.newBuilder()
    .setUserId("U1")
    .setIncludeMetadata(true)
    .build());
```

### buf 도구와 lint 규칙

```
buf.yaml 설정:
  version: v1
  build:
    roots:
      - protos
    exclude_paths:
      - google/protobuf
  
  lint:
    rules:
      enabled:
        - FIELD_NAMES_LOWER_SNAKE_CASE
        - MESSAGE_NAMES_UPPER_CAMEL_CASE
        - SERVICE_NAMES_END_WITH_SERVICE
        - RPC_NAMES_CASE
        - FIELD_NAMES_LOWER_SNAKE_CASE
        - ENUM_NAMES_UPPER_CAMEL_CASE
        - ENUM_VALUE_NAMES_UPPER_SNAKE_CASE
      severity: error

명명 규칙 검사:
  ✓ message User {} (CamelCase)
  ✓ string user_id = 1 (snake_case)
  ✓ service UserService {} (CamelCase + Service)
  ✓ enum Status {} (CamelCase)
  ✓ ACTIVE = 0; (UPPER_SNAKE)
  
  실행:
    buf lint protos/
    
  위반 시:
    protos/user_service.proto:10:1: service name "userService" 
    should be PascalCase and end with "Service"

추가 규칙:
  - PACKAGE_LOWER_SNAKE_CASE
  - ENUM_ZERO_VALUE_SUFFIX  (enum의 0번은 UNSPECIFIED)
  - FILE_LOWER_SNAKE_CASE
  - RPC_RETURN_TYPE_NOT_STREAMING
```

---

## 💻 실전 실험

```bash
# 프로젝트 설정

mkdir -p protos/myapp/{v1,v2,common}

cat > buf.yaml << 'EOF'
version: v1
build:
  roots:
    - protos
lint:
  rules:
    enabled:
      - FIELD_NAMES_LOWER_SNAKE_CASE
      - MESSAGE_NAMES_UPPER_CAMEL_CASE
      - SERVICE_NAMES_END_WITH_SERVICE
      - RPC_NAMES_CASE
EOF

cat > protos/myapp/v1/user_service.proto << 'EOF'
syntax = "proto3";

package myapp.v1;

import "google/protobuf/empty.proto";

service UserService {
  rpc GetUser (GetUserRequest) returns (GetUserResponse);
  rpc CreateUser (CreateUserRequest) returns (CreateUserResponse);
  rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
  rpc DeleteUser (DeleteUserRequest) returns (google.protobuf.Empty);
}

message GetUserRequest {
  string user_id = 1;
}

message GetUserResponse {
  User user = 1;
}

message User {
  string user_id = 1;
  string email = 2;
  string full_name = 3;
  int32 age = 4;
}

message CreateUserRequest {
  string email = 1;
  string full_name = 2;
  optional int32 age = 3;
}

message CreateUserResponse {
  User user = 1;
}

message ListUsersRequest {
  int32 page_size = 1;
  string page_token = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  string next_page_token = 2;
}

message DeleteUserRequest {
  string user_id = 1;
}
EOF

# Lint 검사
buf lint protos/
# → 통과!

# 코드 생성
buf generate protos/
```

```java
// 생성된 코드 사용

import myapp.v1.UserServiceGrpc;
import myapp.v1.UserServiceGrpc.UserServiceStub;
import myapp.v1.GetUserRequest;
import myapp.v1.GetUserResponse;

public class UserClient {
  public static void main(String[] args) {
    // gRPC Stub 생성
    UserServiceStub stub = UserServiceGrpc.newStub(channel);
    
    // Request 생성 (명명 규칙 따름)
    GetUserRequest request = GetUserRequest.newBuilder()
      .setUserId("U123")
      .build();
    
    // RPC 호출 (메서드명 CamelCase)
    stub.getUser(request, new StreamObserver<GetUserResponse>() {
      @Override
      public void onNext(GetUserResponse response) {
        System.out.println("User: " + response.getUser().getFullName());
      }
      
      @Override
      public void onError(Throwable t) {
        t.printStackTrace();
      }
      
      @Override
      public void onCompleted() {
      }
    });
  }
}
```

---

## 📊 성능/비용 비교

```
설계 원칙 준수의 효과:

┌──────────────────┬──────────┬──────────┬──────────┐
│항목              │준수      │미준수    │효과      │
├──────────────────┼──────────┼──────────┼──────────┤
│코드 일관성       │우수      │낮음      │유지보수 50% 감소│
│IDE 자동완성      │정확      │오류      │개발 생산성 30% 증가│
│팀 온보딩         │2주       │1개월     │신입 적응 1/2|
│버전 관리 복잡도  │낮음      │높음      │마이그레이션 수월|
│API 확장성        │완벽      │제약      │리뷰 기간 단축|
└──────────────────┴──────────┴──────────┴──────────┘

팀 규모별 효과:

소팀 (<5명):
  규칙 위반해도 괜찮음 (소통 가능)
  
중팀 (5~20명):
  규칙 준수 가치 증대 (일관성 필수)
  
대팀 (20명 이상):
  규칙 미준수시 재앙 (마이크로서비스 간 혼동)

비용:
  규칙 정의: 1시간
  문서화: 2시간
  팀 교육: 1시간
  → 총 4시간
  
  효과 (버그 감소):
    year 1: 버그 20% 감소 = $10,000 절감
    
  ROI: 4시간 투자로 $10,000 + 편의성 gained
```

---

## ⚖️ 트레이드오프

```
✅ 엄격한 설계 규칙의 장점:
├─ 일관된 API (팀 전체)
├─ 자동 코드생성 정확성
├─ 쉬운 유지보수
└─ 빠른 온보딩

❌ 엄격한 설계 규칙의 단점:
├─ 초기 학습곡선 (규칙 이해)
├─ 명명 규칙 논의 시간
├─ 예외 처리 복잡도
└─ 문서화 부담

실제 balance:
  신입팀: 규칙 먼저 정의 (장기 이득)
  기존팀: 점진적 도입 (마이그레이션)
  다양한팀: 유연하되 문서화 (합의)
```

---

## 📌 핵심 정리

```
명명 규칙:
  Package: snake_case (myapp.v1)
  Service: CamelCase + Service (UserService)
  RPC: CamelCase (GetUser, ListUsers)
  Message: CamelCase (GetUserRequest)
  Field: snake_case (user_id, full_name)
  Enum: CamelCase (Status)
  Enum값: UPPER_SNAKE (ACTIVE, INACTIVE)
  
Request/Response:
  모든 RPC에 전용 Request/Response
  확장성, 선택적 필드 표현 용이
  패턴화된 구조 (Get, Create, List, Update, Delete)
  
패키지 구조:
  단일 버전: protos/myapp/
  멀티 버전: protos/myapp/{v1,v2,v3}/
  버전별 호환성 독립 관리
  
buf 도구:
  명명 규칙 자동 검사 (lint)
  스키마 변경 감지 (breaking)
  코드 생성 (generate)
  스키마 레지스트리 (BSR)
  
결과:
  일관된 API 설계
  자동화된 품질 검사
  빠른 개발 속도
  장기 유지보수성
```

---

## 🤔 생각해볼 문제

**Q1: 메서드명 GetUser와 FetchUser 중 뭘 쓸까?**
```
Google 규칙: Get (조회)
회사 관례: Fetch (검색)

어느 것을 택할까?
```
<details>
<summary>해설 보기</summary>

Google API 규칙 따르기 (권장):
- Get: 이미 존재하는 리소스 조회
- Create: 새 리소스 생성
- Update: 기존 리소스 수정
- Delete: 리소스 삭제
- List: 여러 리소스 조회 (페이징)
- Search: 쿼리 기반 검색

Fetch는 비표준입니다.

규칙을 정하면:
```proto
rpc GetUser (GetUserRequest) returns (User);
rpc ListUsers (ListUsersRequest) returns (ListUsersResponse);
rpc SearchUsers (SearchUsersRequest) returns (SearchUsersResponse);
```

일관성이 중요합니다.

</details>

**Q2: Request를 꼭 만들어야 할까? (단일 필드)**
```
message GetUserRequest {
  string user_id = 1;  // 이 하나만?
}

그냥 GetUser(string user_id)로 안 될까?
```
<details>
<summary>해설 보기</summary>

짧은 기간만 봐선 불필요해 보입니다.

하지만:
```proto
v1:
rpc GetUser(GetUserRequest) returns (User);
message GetUserRequest {
  string user_id = 1;
}

v2 (1년 후):
message GetUserRequest {
  string user_id = 1;
  bool include_deleted = 2;      // 필드 추가
  repeated string fields = 3;    // projection
  optional bool async = 4;       // 비동기 옵션
}
```

v1 클라이언트: include_deleted 모르고 생략
→ 호환성 유지! (Backward compatible)

Request 없이:
```proto
rpc GetUser(string) returns (User);
rpc GetUserV2(GetUserRequestV2) returns (User);  // 새 메서드?
```
메서드 버전이 증가 (복잡도)

결론:
처음부터 Request/Response 만들기!

</details>

**Q3: buf.yaml을 무시하고 자유롭게 코딩해도 될까?**
```
팀이 자유도를 원함.
규칙 적용이 부담스러움.

점진적으로 도입하는 방법?
```
<details>
<summary>해설 보기</summary>

단계적 도입:

Phase 1: 문서화만 (강제 X)
```
PROTO_STYLE_GUIDE.md 작성
buf.yaml 없음
```

Phase 2: Lint 검사만 (CI에 추가)
```yaml
lint:
  rules:
    enabled:
      - FIELD_NAMES_LOWER_SNAKE_CASE  # 중요한 것부터
  severity: warn  # 경고만 (실패 아님)
```

Phase 3: 강제 (CI fail)
```yaml
severity: error  # 강제
```

권장 타임라인:
Week 1: Phase 1 (교육)
Week 2-3: Phase 2 (기존 proto 정리)
Week 4+: Phase 3 (강제)

효과:
팀의 저항감 최소화
점진적 문화 변화
버그 감소 확인 후 강제

</details>

---

<div align="center">

**[⬅️ 이전: 직렬화 성능 측정](../protocol-buffers/07-serialization-benchmark.md)** | **[홈으로 🏠](../README.md)** | **[다음: 에러 처리 ➡️](./02-error-handling.md)**

</div>
