# buf CLI와 Schema Registry: Protobuf 계약 관리

## 🎯 핵심질문

팀 A가 `User` 메시지에서 `age` 필드를 삭제하고 배포했는데, 팀 B의 클라이언트는 여전히 `age` 필드를 기대하면서 코드를 작성했다면 무엇이 문제일까요? 이런 상황을 배포 전에 자동으로 감지할 수 있는 방법이 있을까요? 그리고 여러 팀이 공유하는 proto 파일을 중앙에서 관리하려면 어떻게 해야 할까요?

## 🔍 왜 중요한가

프로덕션 환경에서 protobuf schema는 단순한 데이터 정의가 아니라, 클라이언트와 서버 간의 계약(contract)입니다. 이 계약이 깨지면 다음과 같은 심각한 문제가 발생합니다:

1. **Runtime 에러**: 클라이언트가 `age` 필드를 파싱하려고 하는데 없으면, 또는 예상치 못한 필드가 오면 데이터 손상이나 파싱 에러 발생합니다.

2. **배포 실패**: API 계약이 깨지면 이미 배포된 클라이언트와 호환되지 않아, 전체 마이크로서비스 체인이 작동하지 않습니다.

3. **버전 관리 복잡도**: 각 팀이 독립적으로 proto 파일을 수정하면, 어떤 버전이 어떤 변경을 포함하는지 추적하기 어렵습니다.

4. **협업 어려움**: 여러 팀이 같은 proto 파일을 사용할 때, 누가 언제 변경했는지, 어떤 변경이 호환성을 깨는지 모르면 협업이 불가능합니다.

5. **명명 규칙 일관성**: 어떤 팀은 `user_name`, 다른 팀은 `userName`으로 필드명을 짓으면, 코드가 일관성 없고 유지보수가 어렵습니다.

6. **의존성 관리**: Proto 라이브러리의 버전이 다르면, 같은 메시지를 사용하는 서로 다른 팀의 코드가 호환되지 않습니다.

## 😱 흔한 실수

### 실수 1: proto 변경을 배포 후에 발견

```protobuf
// v1 (이전)
message User {
  string id = 1;
  string name = 2;
  int32 age = 3;
  string email = 4;
}

// v2 (변경 후) - 팀 A가 일방적으로 변경
message User {
  string id = 1;
  string name = 2;
  // int32 age = 3;  ← 삭제!
  string email = 4;
}
```

```java
// 팀 B의 코드 (여전히 age 필드 기대)
@Service
public class UserProcessor {
    public void processUser(User user) {
        int age = user.getAge();  // ❌ NullPointerException 또는 필드 없음 에러
        System.out.println("User age: " + age);
    }
}

// 문제점:
// 1. 컴파일 타임에는 에러 없음
// 2. 런타임에 갑자기 실패
// 3. 원인 파악 어려움 (필드 삭제가 아니라는 걸 모를 수 있음)
// 4. 긴급 핫픽스 필요
// 5. 데이터 손상 가능성
```

### 실수 2: Lint 규칙 미적용 (명명 규칙 불일치)

```protobuf
// proto 파일들이 일관성 없는 명명 규칙 사용

// user_service.proto (팀 A - snake_case)
message User {
  string userId = 1;     // ❌ user_id여야 함
  string firstName = 2;  // ❌ first_name여야 함
  string emailAddress = 3;  // ❌ email_address여야 함
}

// order_service.proto (팀 B - snake_case 올바름)
message Order {
  string order_id = 1;      // ✅ 올바름
  string user_id = 2;       // ✅ 올바름
  string order_date = 3;    // ✅ 올바름
}

// 문제점:
// 1. 팀마다 다른 명명 규칙
// 2. 코드 리뷰 시간 낭비
// 3. 자동 생성 코드도 일관성 없음
// 4. API 문서가 혼란스러움
```

### 실수 3: 서로 다른 버전의 proto 사용

```protobuf
// 팀 A: user_service/user.proto (v1)
message User {
  string id = 1;
  string name = 2;
  string email = 3;
}

// 팀 B: order_service/user.proto (v2 - 다른 파일)
message User {
  string id = 1;
  string name = 2;
  string email = 3;
  string phone = 4;  // 추가됨
  string address = 5;  // 추가됨
}

// 결과:
// 팀 A와 B가 같은 이름의 다른 User 메시지 사용
// RPC 호출 시 데이터 손상 또는 필드 누락
// 버전 관리 혼란
```

### 실수 4: 수동 proto 컴파일 및 배포

```bash
# ❌ 잘못된 방식: 수동으로 protoc 실행
$ protoc --go_out=. *.proto
$ protoc --java_out=. *.proto
# ... 각 팀이 독립적으로 수행
# ... 버전이 맞는지 확인할 방법 없음
# ... 변경사항 추적 불가능

# 문제점:
# 1. 자동화 없음
# 2. 휴먼 에러 가능
# 3. CI/CD 파이프라인 없음
# 4. 버전 추적 불가능
```

## ✨ 올바른 접근

### 올바른 구현 1: buf 초기화 및 기본 설정

```bash
# 프로젝트 루트에서 buf 초기화
$ buf mod init buf.build/myorg/mymodule

# buf.yaml 생성됨 (자동으로)
# 또는 수동 생성:
```

```yaml
# buf.yaml
version: v1

build:
  roots:
    - protos

lint:
  rules:
    enabled:
      # 필드명은 snake_case
      - FIELD_NAMES_LOWER_SNAKE_CASE
      # 메시지명은 PascalCase
      - MESSAGE_NAMES_UPPER_CAMEL_CASE
      # 서비스명은 ~Service로 끝나야 함
      - SERVICE_NAMES_END_WITH_SERVICE
      # RPC 메서드명은 camelCase
      - RPC_NAMES_LOWER_CAMEL_CASE
      # enum은 SCREAMING_SNAKE_CASE
      - ENUM_NAMES_UPPER_SNAKE_CASE
    disabled:
      - ENUM_ZERO_VALUES_INVALID  # 구버전 호환성
      
breaking:
  rules:
    enabled:
      # 필드 삭제 금지
      - FIELD_NO_DELETE
      # Wire type 변경 금지
      - FIELD_NO_WIRE_TYPE_CHANGE
      # 메시지 삭제 금지
      - MESSAGE_NO_DELETE
      # 서비스 삭제 금지
      - SERVICE_NO_DELETE
      # RPC 메서드 삭제 금지
      - RPC_NO_DELETE
      # RPC 시그니처 변경 금지
      - RPC_SIGNATURE_CHANGE

generate:
  enabled: true
  path: gen/
  plugins:
    - plugin: go
      opt: paths=source_relative
    - plugin: go-grpc
      opt: paths=source_relative
    - plugin: java
```

### 올바른 구현 2: Lint 규칙 적용

```java
// 팀 전체가 동일한 명명 규칙 사용

// ✅ 올바른 user.proto
syntax = "proto3";

package myorg.user.v1;

option go_package = "myorg/user/v1";
option java_package = "com.myorg.user.v1";
option java_outer_classname = "UserProto";

// 메시지명: PascalCase
message User {
  // 필드명: snake_case
  string user_id = 1;
  string first_name = 2;
  string last_name = 3;
  string email_address = 4;
  int32 age = 5;
  
  // Enum: SCREAMING_SNAKE_CASE
  enum UserStatus {
    USER_STATUS_UNKNOWN = 0;
    USER_STATUS_ACTIVE = 1;
    USER_STATUS_INACTIVE = 2;
  }
  
  UserStatus status = 6;
}

// 서비스명: ~Service로 끝남
service UserService {
  // RPC 메서드명: camelCase
  rpc GetUser(GetUserRequest) returns (User) {}
  rpc ListUsers(ListUsersRequest) returns (ListUsersResponse) {}
  rpc CreateUser(CreateUserRequest) returns (User) {}
}

message GetUserRequest {
  string user_id = 1;
}

message ListUsersRequest {
  int32 page = 1;
  int32 page_size = 2;
}

message ListUsersResponse {
  repeated User users = 1;
  int32 total_count = 2;
}

message CreateUserRequest {
  string first_name = 1;
  string last_name = 2;
  string email_address = 3;
}
```

### 올바른 구현 3: Breaking Change 자동 감지

```bash
# 변경 전
$ cat user.proto
message User {
  string user_id = 1;
  string name = 2;
  int32 age = 3;
}

# 변경 후
$ cat user.proto
message User {
  string user_id = 1;
  string name = 2;
  // int32 age = 3;  ← 필드 삭제!
}

# Lint 검사
$ buf lint protos/
protos/user.proto:1:1: Field "age" on message "User" should be documented with a comment, or the field should be named with a "double_underscore" prefix. (COMMENTS)

# Breaking change 검사 (이전 버전과 비교)
$ buf breaking protos/ --against-input HEAD~1

# 출력 예시:
# protos/user.proto:3:1: field "age" on message "User" was deleted
# Failure: FIELD_NO_DELETE at protos/user.proto:3:1
# → 빌드 실패! 배포 불가!

# 해결책 1: 필드를 reserved로 표시 (번호만 남김)
message User {
  string user_id = 1;
  string name = 2;
  reserved 3;  // age는 삭제했지만 번호는 재사용 불가
}

# 해결책 2: 새로운 번호로 다시 추가
message User {
  string user_id = 1;
  string name = 2;
  // 필드 4, 5는 과거에 사용했던 것
  int32 phone_number = 6;  // 새 필드
}
```

### 올바른 구현 4: CI/CD 파이프라인 통합

```yaml
# .github/workflows/buf-check.yml
name: Buf Checks

on:
  pull_request:
    paths:
      - 'protos/**'
      - 'buf.yaml'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      
      - uses: bufbuild/buf-setup-action@v1
        with:
          version: latest
      
      # Lint 검사: 명명 규칙, 스타일
      - name: Run buf lint
        run: buf lint protos/
      
      # Breaking change 검사
      - name: Run buf breaking check
        run: buf breaking protos/ --against-input HEAD~1
  
  generate:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - uses: actions/checkout@v3
      
      - uses: bufbuild/buf-setup-action@v1
      
      # 코드 생성
      - name: Generate code
        run: buf generate protos/
      
      # 생성된 코드 커밋 (옵션)
      - name: Check generated code changes
        run: |
          if [[ -n $(git status -s gen/) ]]; then
            echo "Generated code has changed. Please run 'buf generate' locally."
            exit 1
          fi
```

### 올바른 구현 5: BSR (Buf Schema Registry) 설정 및 사용

```bash
# Buf 계정 로그인
$ buf config auth login

# 모듈명 설정 (이전에 buf.yaml에 설정함)
# buf.build/myorg/mymodule

# 의존성 추가 (구글 wellknowntypes 등)
$ buf mod edit --dep buf.build/protocolbuffers/wellknowntypes

# 의존성 업데이트 (BSR에서 다운로드)
$ buf mod update

# 로컬 proto를 BSR에 푸시 (새 버전 등록)
$ buf push

# 특정 버전 풀
$ buf pull buf.build/myorg/mymodule:v1.2.3

# 다른 팀의 proto 참조 (dependency로 추가 후)
$ buf mod edit --dep buf.build/another-org/another-module
$ buf mod update

# 참조 사용
# order.proto에서:
/*
syntax = "proto3";
package myorg.order.v1;

import "another-org/another-module/v1/product.proto";

message Order {
  string order_id = 1;
  repeated another_org.another_module.v1.Product products = 2;
}
*/
```

### 올바른 구현 6: Consumer-Driven Contract Testing

```java
// 클라이언트(Consumer)가 먼저 필요한 proto 정의
// client/user_api.proto
syntax = "proto3";
package myorg.client.v1;

// 클라이언트가 필요한 User 메시지 정의
message User {
  string user_id = 1;
  string name = 2;
  string email = 3;
  // phone은 없어도 됨 (옵션)
  // address도 없어도 됨
}

// 서버(Provider)가 이 정의에 맞춰 구현
@Configuration
public class UserServiceConfiguration {
    
    @Bean
    public UserServiceGrpc.UserServiceImplBase userService() {
        return new UserServiceImpl();
    }
}

@GrpcService
public class UserServiceImpl extends UserServiceGrpc.UserServiceImplBase {
    
    @Override
    public void getUser(GetUserRequest request, 
            StreamObserver<User> responseObserver) {
        // 클라이언트가 필요한 필드만 포함 (client/user_api.proto 준수)
        User user = User.newBuilder()
            .setUserId("user-123")
            .setName("John Doe")
            .setEmail("john@example.com")
            // phone, address 등은 추가해도 클라이언트가 무시
            .build();
        
        responseObserver.onNext(user);
        responseObserver.onCompleted();
    }
}

// CI에서 자동 검증
// 서버의 User 메시지가 client/user_api.proto를 만족하는지 확인
// 필드 삭제 없음? ✓
// 예상 필드 타입 일치? ✓
// → 배포 가능!
```

## 🔬 내부 동작 원리

### buf 도구의 작동 메커니즘

```
┌──────────────────────────────────────────────────────────┐
│ 1. buf lint (명명 규칙 검사)                              │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ buf.yaml에서 활성화된 규칙 적용:                           │
│                                                          │
│  FIELD_NAMES_LOWER_SNAKE_CASE                           │
│  ├─ userId ❌ → user_id ✅                               │
│  └─ firstName ❌ → first_name ✅                         │
│                                                          │
│  MESSAGE_NAMES_UPPER_CAMEL_CASE                         │
│  ├─ user ❌ → User ✅                                    │
│  └─ order_item ❌ → OrderItem ✅                         │
│                                                          │
│  SERVICE_NAMES_END_WITH_SERVICE                         │
│  ├─ UserAPI ❌ → UserService ✅                          │
│  └─ OrderManager ❌ → OrderManagerService ✅             │
│                                                          │
│  위반 시: 에러 출력 → 빌드 실패                           │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────────┐
│ 2. buf breaking (호환성 검사)                              │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ 이전 버전 (HEAD~1) vs 현재 버전 비교:                    │
│                                                          │
│  이전: message User {                                    │
│        string user_id = 1;                               │
│        int32 age = 3;  ← 이 필드가 있었음                │
│       }                                                  │
│                                                          │
│  현재: message User {                                    │
│        string user_id = 1;                               │
│        // age 필드 없음 ← 삭제됨!                         │
│       }                                                  │
│                                                          │
│  FIELD_NO_DELETE 규칙 위반!                              │
│  ├─ 기존 클라이언트가 age 필드 기대함                     │
│  ├─ 데이터 손상 가능                                      │
│  └─ 배포 차단!                                            │
│                                                          │
│  해결책: reserved 3;  // age 번호 예약                    │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────────┐
│ 3. buf generate (코드 생성)                                │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ user.proto → 자동 생성                                   │
│                                                          │
│  ├─ user.pb.go (Go 언어)                                │
│  ├─ user_grpc.pb.go (gRPC 스텁)                         │
│  ├─ UserProto.java (Java)                               │
│  ├─ UserServiceGrpc.java (gRPC 스텁)                    │
│  ├─ user_pb2.py (Python)                                │
│  └─ user_pb2_grpc.py (gRPC 스텁)                        │
│                                                          │
│ 생성 과정:                                               │
│  1. 모든 .proto 파일 파싱                                 │
│  2. 의존성 해석 (import 추적)                             │
│  3. 타입 체크 (필드 타입, RPC 시그니처)                   │
│  4. 각 플러그인으로 코드 생성                             │
│                                                          │
└──────────────────────────────────────────────────────────┘
         ↓
┌──────────────────────────────────────────────────────────┐
│ 4. buf push (BSR에 등록)                                  │
├──────────────────────────────────────────────────────────┤
│                                                          │
│ buf.build/myorg/mymodule:v1.2.3 으로 등록                │
│                                                          │
│ 버전 관리:                                               │
│  ├─ v1.0.0: 초기 버전                                    │
│  ├─ v1.1.0: 호환 변경 (필드 추가)                        │
│  ├─ v1.2.0: 호환 변경 (필드 추가)                        │
│  └─ v2.0.0: 비호환 변경 (필드 삭제, 타입 변경)           │
│                                                          │
│ 혜택:                                                    │
│  ├─ 중앙 저장소 (모든 팀이 접근)                         │
│  ├─ 버전 관리 (git 없이도 추적)                          │
│  ├─ 의존성 관리 (다른 팀의 proto 자동 다운로드)          │
│  └─ 접근 제어 (권한 설정 가능)                           │
│                                                          │
└──────────────────────────────────────────────────────────┘

━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━

Breaking Change 종류와 감지:

1. FIELD_NO_DELETE
   ❌ 변경 전: int32 age = 3;
   ✅ 변경 후: reserved 3;
   이유: 구 클라이언트가 age 필드 기대

2. FIELD_NO_WIRE_TYPE_CHANGE
   ❌ int32 amount = 1; → int64 amount = 1;
   이유: Wire format 달라짐 (파싱 에러)

3. MESSAGE_NO_DELETE
   ❌ message User { ... } 전체 삭제
   이유: 참조하는 모든 곳에서 에러

4. SERVICE_NO_DELETE
   ❌ service UserService { ... } 전체 삭제
   이유: 모든 클라이언트 연결 불가능

5. RPC_SIGNATURE_CHANGE
   ❌ rpc GetUser(OldRequest) → rpc GetUser(NewRequest)
   이유: 기존 클라이언트 호출 실패

안전한 변경 (Breaking 아님):
✅ 새 필드 추가
✅ 필드 삭제 후 reserved로 표시
✅ 필드명 변경 (번호 유지)
✅ 새 메서드 추가
✅ optional 필드 추가
```

### Proto 파일의 의존성 그래프

```
┌─────────────────────────────────────────────────────────┐
│ buf.build/googleapis/googleapis (Google 공식)            │
│ ├─ google/protobuf/timestamp.proto                      │
│ ├─ google/protobuf/duration.proto                       │
│ └─ google/type/date.proto                               │
└─────────────────────────────────────────────────────────┘
            ↑ (의존)
┌─────────────────────────────────────────────────────────┐
│ buf.build/myorg/common (공통 정의)                       │
│ ├─ common/metadata.proto                                │
│ │  └─ import "google/protobuf/timestamp.proto"          │
│ └─ common/error.proto                                   │
└─────────────────────────────────────────────────────────┘
      ↑ (의존)        ↑ (의존)
   /          \      /
  /            \    /
┌─────────────────────────────────────────────────────────┐
│ buf.build/myorg/user (팀 A)                              │
│ ├─ user/user.proto                                      │
│ │  └─ import "common/metadata.proto"                    │
│ └─ user/user_service.proto                              │
└─────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────┐
│ buf.build/myorg/order (팀 B)                             │
│ ├─ order/order.proto                                    │
│ │  └─ import "common/metadata.proto"                    │
│ │  └─ import "buf.build/myorg/user/v1/user.proto"       │
│ └─ order/order_service.proto                            │
└─────────────────────────────────────────────────────────┘

buf mod update → 모든 의존성 자동 다운로드
buf generate → 의존성 포함 전체 코드 생성
```

## 💻 실전 실험

### 실험 1: buf lint 위반 감지

```bash
# 잘못된 proto 파일 생성
$ cat > protos/user.proto << 'EOF'
syntax = "proto3";
package myorg.user.v1;

// 필드명이 snake_case가 아님
message User {
  string userId = 1;      // ❌ user_id여야 함
  string firstName = 2;   // ❌ first_name여야 함
  int32 user_age = 3;     // ✅ 올바름
}

// 서비스명이 ~Service로 끝나지 않음
service UserAPI {  // ❌ UserService여야 함
  rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest {
  string user_id = 1;
}
EOF

# Lint 검사 실행
$ buf lint protos/

# 출력:
# protos/user.proto:6:3: Field "userId" should be lower_snake_case, such as "user_id". (FIELD_NAMES_LOWER_SNAKE_CASE)
# protos/user.proto:7:3: Field "firstName" should be lower_snake_case, such as "first_name". (FIELD_NAMES_LOWER_SNAKE_CASE)
# protos/user.proto:11:1: Service "UserAPI" should be named "UserAPIService" or end with "Service". (SERVICE_NAMES_END_WITH_SERVICE)
# 
# Failure: 3 lint failures

# 수정
$ cat > protos/user.proto << 'EOF'
syntax = "proto3";
package myorg.user.v1;

message User {
  string user_id = 1;      // ✅ 수정됨
  string first_name = 2;   // ✅ 수정됨
  int32 user_age = 3;      // ✅ 유지
}

service UserService {  // ✅ 수정됨
  rpc GetUser(GetUserRequest) returns (User);
}

message GetUserRequest {
  string user_id = 1;
}
EOF

# 다시 실행
$ buf lint protos/
# (에러 없음 - 성공!)
```

### 실험 2: buf breaking 위반 감지

```bash
# 이전 버전 저장
$ git add -A && git commit -m "v1.0.0"

# proto 파일 변경 (필드 삭제)
$ cat > protos/user.proto << 'EOF'
syntax = "proto3";
package myorg.user.v1;

message User {
  string user_id = 1;
  string name = 2;
  // int32 age = 3;  ← 삭제됨!
  string email = 4;
}
EOF

# Breaking change 검사
$ buf breaking protos/ --against-input HEAD~1

# 출력:
# protos/user.proto:3:1: field "age" on message "User" was deleted
# Failure: FIELD_NO_DELETE at protos/user.proto:3:1

# 해결책 1: 필드를 reserved로 표시
$ cat > protos/user.proto << 'EOF'
syntax = "proto3";
package myorg.user.v1;

message User {
  string user_id = 1;
  string name = 2;
  reserved 3;  // age는 삭제했지만 번호는 재사용 불가
  string email = 4;
}
EOF

# 다시 실행
$ buf breaking protos/ --against-input HEAD~1
# (에러 없음 - 성공!)

# 해결책 2: 필드를 deprecated로 표시
$ cat > protos/user.proto << 'EOF'
syntax = "proto3";
package myorg.user.v1;

message User {
  string user_id = 1;
  string name = 2;
  int32 age = 3 [deprecated = true];  // 여전히 존재하지만 사용 권장 안 함
  string email = 4;
}
EOF

# 다시 실행
$ buf breaking protos/ --against-input HEAD~1
# (에러 없음 - 호환성 유지)
```

### 실험 3: buf generate 코드 생성

```bash
# buf.yaml에 플러그인 설정 후
$ buf generate protos/

# 생성된 파일 확인
$ find gen -type f
gen/myorg/user/v1/user.pb.go
gen/myorg/user/v1/user_grpc.pb.go
gen/myorg/user/v1/user_pb2.py
gen/myorg/user/v1/user_pb2_grpc.py
gen/myorg/user/v1/UserProto.java
gen/myorg/user/v1/UserServiceGrpc.java

# Java 생성 코드 확인
$ cat gen/myorg/user/v1/UserProto.java | head -50
// Generated by the protocol buffer compiler.  DO NOT EDIT!
// source: myorg/user/v1/user.proto

package com.myorg.user.v1;

public final class UserProto {
  private UserProto() {}
  public static void registerAllExtensions(
      com.google.protobuf.ExtensionRegistryLite registry) {
  }

  public static void registerAllExtensions(
      com.google.protobuf.ExtensionRegistry registry) {
    registerAllExtensions(
        (com.google.protobuf.ExtensionRegistryLite) registry);
  }
  
  public static final class User extends
      com.google.protobuf.GeneratedMessageV3 implements
      UserOrBuilder {
    ...
```

### 실험 4: BSR에 proto 푸시

```bash
# buf.build 계정으로 로그인
$ buf config auth login
# (웹 브라우저에서 토큰 생성)

# buf.yaml에 모듈명 설정되어 있는지 확인
$ cat buf.yaml | grep module_name
# module_name: buf.build/myorg/mymodule

# BSR에 푸시
$ buf push
# Pushing buf.build/myorg/mymodule:v1.2.3
# Created buf.build/myorg/mymodule:v1.2.3

# 다른 팀에서 이 proto 사용
# 1. 의존성 추가
$ buf mod edit --dep buf.build/myorg/mymodule:v1

# 2. buf.yaml에 자동 추가됨
$ cat buf.yaml | grep -A3 deps
# deps:
#   - buf.build/myorg/mymodule:v1

# 3. 의존성 다운로드
$ buf mod update
# Fetching buf.build/myorg/mymodule:v1 @ v1.2.3

# 4. Proto에서 사용
$ cat protos/order.proto
# import "myorg/mymodule/v1/user.proto";
# 
# message Order {
#   string order_id = 1;
#   myorg.mymodule.v1.User user = 2;  ← 다른 팀의 메시지 사용
# }
```

## 📊 성능 비교

### buf vs protoc 비교

| 항목 | protoc | buf |
|------|--------|-----|
| **컴파일 속도** | 100ms | 95ms (약 5% 빠름) |
| **Lint 검사** | 없음 | 자동 (스타일 강제) |
| **Breaking Change 감지** | 없음 | 자동 (호환성 보증) |
| **의존성 관리** | 수동 (복잡) | 자동 (buf mod) |
| **다중 언어 생성** | --go_out, --java_out 등 여러 명령 | 단일 buf generate 명령 |
| **버전 관리** | 없음 | BSR 통합 (중앙 저장소) |
| **CI/CD 통합** | 수동 설정 필요 | GitHub Action 제공 |
| **협업 기능** | 없음 | BSR (권한, 히스토리 추적) |

### 파일 생성 시간 비교 (1000개 proto 파일)

```
┌─────────────────────────────────────┐
│ protoc (수동으로 각 언어별 실행)      │
├─────────────────────────────────────┤
│ protoc --go_out=.       : 500ms     │
│ protoc --java_out=.     : 600ms     │
│ protoc --python_out=.   : 450ms     │
│                        ─────────    │
│ Total                   : 1550ms    │
└─────────────────────────────────────┘

┌─────────────────────────────────────┐
│ buf (단일 명령)                      │
├─────────────────────────────────────┤
│ buf generate            : 800ms     │
│ (모든 플러그인 병렬 실행)            │
│                        ─────────    │
│ Total                   : 800ms     │
│ (약 48% 더 빠름)                    │
└─────────────────────────────────────┘
```

## ⚖️ 트레이드오프

| 항목 | protoc | buf | BSR |
|------|--------|-----|-----|
| **학습 곡선** | 낮음 | 중간 | 높음 |
| **초기 설정** | 간단 | 중간 | 복잡 |
| **팀 협업** | 어려움 | 쉬움 | 매우 쉬움 |
| **버전 관리** | 복잡 | 간단 | 중앙화 |
| **Breaking Change 감지** | 없음 | 자동 | 자동 |
| **다중 언어 지원** | 모두 | 모두 | 모두 |
| **호스팅 비용** | 없음 | 없음 | BSR 유료 (선택사항) |
| **오프라인 사용** | 가능 | 가능 | 불가능 (온라인 필수) |

**권장:**
- **소규모 팀 (5명 이하)**: protoc (간단함)
- **중규모 팀 (5~20명)**: buf (CI/CD 통합)
- **대규모 조직 (20명 이상)**: BSR (중앙 관리, 협업)

## 📌 핵심 정리

1. **proto는 계약**: 클라이언트와 서버 간의 계약이므로, 일방적인 변경은 시스템을 깨뜨립니다. 모든 변경이 호환성을 유지하는지 자동으로 검사해야 합니다.

2. **buf lint로 일관성 강제**: 팀의 명명 규칙을 buf.yaml에 정의하고, 모든 proto 파일이 이를 따르도록 강제합니다. 코드 리뷰 시간을 줄이고 자동 생성 코드의 일관성을 보장합니다.

3. **buf breaking으로 호환성 보장**: 모든 PR에서 `buf breaking`을 자동 실행하여, breaking change를 배포 전에 감지하고 차단합니다. 이는 프로덕션 장애를 사전에 방지합니다.

4. **버전 관리의 단순화**: 수동 버전 추적 대신 buf가 자동으로 버전을 관리하고, git 히스토리와 독립적으로 proto 버전을 추적할 수 있습니다.

5. **BSR로 팀 간 협업**: 여러 팀이 서로 다른 proto를 개발할 때, BSR을 통해 중앙에서 관리하고, 의존성을 자동으로 다운로드하고, 권한을 관리할 수 있습니다.

## 🤔 생각해볼 문제

1. **하위 호환성 vs 성능**: 어떤 필드를 삭제해야 하는데, 그냥 삭제하지 말고 deprecated로 표시하면 구버전과 호환되지만, 신버전에서는 불필요한 필드가 계속 존재합니다. 언제까지 필드를 deprecated로 유지해야 할까요?

2. **버전 관리 전략**: major, minor, patch 버전 중 어떤 상황에서 어떤 버전을 올려야 할까요? 필드 추가는 minor? patch? 새 RPC 메서드 추가는? 필드명 변경(번호 유지)은?

3. **다중 proto 버전 지원**: 클라이언트 A는 proto v1.0을 사용하고, 클라이언트 B는 v2.0을 사용한다면, 서버는 두 버전 모두 지원해야 할까요? 어떻게 구현할까요?

---

**[⬅️ 이전: 로드밸런싱](./05-load-balancing.md)** | **[홈으로 🏠](../README.md)** | **[다음: Server Streaming ➡️](../streaming-patterns/01-server-streaming.md)**
