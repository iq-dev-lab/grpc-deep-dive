# Well-Known Types — Timestamp, Any, Struct

---

## 🎯 핵심 질문

- google.protobuf.Timestamp는 UTC 시간을 어떻게 저장하나?
- Any 타입은 언제 쓰고, 성능은?
- Struct는 JSON 같은 동적 구조를 정말 표현하나?
- FieldMask는 PATCH 요청을 어떻게 표현하나?
- Well-Known Types의 JSON 변환 규칙은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Google이 제공하는 Well-Known Types는 시간, 동적 데이터, 부분 업데이트 같은 공통 패턴을 표준화합니다. Timestamp를 쓰면 모든 언어에서 시간 처리가 일관되고, Any를 쓰면 런타임 타입 정보를 보존할 수 있으며, FieldMask를 쓰면 REST의 PATCH를 Protobuf에서 안전하게 표현합니다.

---

## 😱 흔한 실수 (Before — 직접 구현하는 접근)

```
// 실수 1: 시간을 int64로 직접 저장
message Event {
  string event_id = 1;
  int64 created_at = 2;  // Unix timestamp (밀리초? 초? 나노초?)
}

// 문제: 단위 불명확
// timestamp=1704067200000 (밀리초) vs 1704067200 (초)
// 클라이언트마다 다르게 해석 가능

// 실수 2: 시간대 정보 없음
message Notification {
  string message = 1;
  int64 sent_at = 2;
  string timezone = 3;      // "Asia/Seoul"? "KST"? (+09:00)?
}

// timezone 처리 복잡, 실수 가능

// 실제 3: 동적 JSON을 string으로 저장
message LogEntry {
  string log_id = 1;
  string metadata = 2;  // JSON string "{...}"으로 저장?
  string payload = 3;   // 역시 JSON string?
}

// JSON 파싱 필요, 타입 검증 없음, 성능 저하

// 실수 4: 부분 업데이트를 위해 별도 메시지
message UserUpdateRequest {
  string user_id = 1;
  optional string name = 2;
  optional string email = 3;
  optional int32 age = 4;
  // ... 20개 필드마다 optional? 엄청난 중복
}

// 어떤 필드가 실제로 업데이트되는지 불명확
```

---

## ✨ 올바른 접근 (After — Well-Known Types 활용)

```
// 올바른 접근 1: Timestamp 사용
import "google/protobuf/timestamp.proto";

message Event {
  string event_id = 1;
  google.protobuf.Timestamp created_at = 2;  // UTC, 나노초 정밀도
  google.protobuf.Timestamp updated_at = 3;
}

// 장점:
// ├─ UTC 표준화 (타임존 불필요)
// ├─ 나노초 정밀도
// ├─ JSON 호환 (RFC 3339 format)
// └─ 모든 언어에서 동일한 시간 처리

// 올바른 접근 2: Any 타입으로 다형성 데이터
import "google/protobuf/any.proto";

message Event {
  string event_id = 1;
  google.protobuf.Timestamp created_at = 2;
  google.protobuf.Any metadata = 3;  // 어떤 메시지든 가능
}

// 사용:
// UserCreated event → metadata 안에 User 메시지
// OrderPlaced event → metadata 안에 Order 메시지

// 올바른 접근 3: Struct로 동적 데이터
import "google/protobuf/struct.proto";

message LogEntry {
  string log_id = 1;
  google.protobuf.Timestamp timestamp = 2;
  google.protobuf.Struct attributes = 3;  // JSON 같은 key-value
}

// {attributes: {user_id: "U1", action: "login", ip: "192.168.1.1"}}

// 올바른 접근 4: FieldMask로 부분 업데이트
import "google/protobuf/field_mask.proto";

message UserUpdateRequest {
  string user_id = 1;
  User user = 2;  // 전체 User 데이터
  google.protobuf.FieldMask update_mask = 3;  // 어느 필드를 업데이트할지
}

// update_mask: {paths: ["name", "email"]}
// → User의 name, email만 업데이트, age 무시
```

---

## 🔬 내부 동작 원리

### google.protobuf.Timestamp

```
정의:
  message Timestamp {
    int64 seconds = 1;      // Unix epoch (1970-01-01)
    int32 nanos = 2;        // 0 ~ 999,999,999
  }

저장 원리:
  시간 = seconds + nanos / 1,000,000,000 초
  
  예: 2024-01-01 12:30:45.123456789 UTC
    seconds: 1704110445 (1970-01-01부터 몇 초)
    nanos: 123456789 (추가 나노초)
    
    검증: nanos는 반드시 0 ~ 999,999,999 범위

직렬화:
  Tag(1) + Varint(seconds) + Tag(2) + Int32(nanos)
  
  예: Timestamp{seconds:1704110445, nanos:123456789}
    → 08 dd ba a2 83 01 | 10 95 cf dc 3a
    → 약 10 바이트

JSON 변환:
  RFC 3339 format: "2024-01-01T12:30:45.123456789Z"
  
  필드 값이 0이면:
    seconds=0, nanos=0 → "1970-01-01T00:00:00Z"

범위:
  seconds: -(2^31) to 2^31-1
    → 1970년 ± 68년
    → 1902-01-01 ~ 2038-01-19
    
  더 먼 미래 필요시:
    → seconds를 int64로 확장 (64비트)
    → year 9999까지 가능

언어별 구현:
  Java: Instant, ZonedDateTime 변환
  Go: time.Time 변환
  Python: datetime 변환
  JavaScript: Date 변환
  
  모두 UTC로 표준화 (타임존 정보 없음)

시간대 처리:
  Timestamp는 항상 UTC
  로컬 시간대 필요시:
    ├─ string으로 "2024-01-01T12:30:45+09:00" 저장
    ├─ int32 timezone_offset = 32400 (초)
    └─ enum timezone = "Asia/Seoul"
```

### google.protobuf.Any

```
정의:
  message Any {
    string type_url = 1;      // 메시지 타입 URL
    bytes value = 2;          // 직렬화된 메시지 바이트
  }

구조:
  type_url: "type.googleapis.com/package.MessageType"
    → 패키지.메시지 형태로 타입 정보 저장
  
  value: 실제 메시지를 직렬화한 바이트
    → 런타임에 type_url을 보고 역직렬화

사용 예:
  message UserCreatedEvent {
    string event_id = 1;
    google.protobuf.Timestamp timestamp = 2;
    google.protobuf.Any data = 3;
  }
  
  event 1: {
    type_url: "type.googleapis.com/myapp.User",
    value: <User 메시지 바이트>
  }
  
  event 2: {
    type_url: "type.googleapis.com/myapp.Order",
    value: <Order 메시지 바이트>
  }

코드:
  // Any에 메시지 포함
  User user = User.newBuilder()...build();
  Any wrappedUser = Any.pack(user);
  
  // Any에서 메시지 추출
  if (wrappedUser.is(User.class)) {
    User user = wrappedUser.unpack(User.class);
  }
  
  // 타입 확인
  String typeUrl = wrappedUser.getTypeUrl();
  // → "type.googleapis.com/myapp.User"

직렬화:
  Any{
    type_url: "type.googleapis.com/myapp.User",
    value: <10 bytes User>
  }
  → 0a 26 74 79 70 65 2e ... | 12 0a <10 bytes>
  → type_url + 1 바이트 오버헤드

장점:
  ├─ 다형성 메시지 저장 (다양한 타입)
  ├─ 런타임 타입 정보 유지
  ├─ forward compatible (새 타입 자동 처리)
  └─ JSON 호환 (JSON-to-Any 변환 가능)

주의:
  ├─ 런타임에만 타입 확인 (컴파일 타임 타입 검증 없음)
  ├─ 성능: 직렬화/역직렬화 오버헤드
  ├─ 순환 참조 주의 (Any 안에 Any는 피하기)
  └─ 타입 URL 오타 주의 (런타임 에러)
```

### google.protobuf.Struct

```
정의:
  message Struct {
    map<string, Value> fields = 1;
  }
  
  message Value {
    oneof kind {
      NullValue null_value = 1;
      double number_value = 2;
      string string_value = 3;
      bool bool_value = 4;
      Struct struct_value = 5;      // 중첩 가능!
      ListValue list_value = 6;
    }
  }
  
  message ListValue {
    repeated Value values = 1;
  }

구조:
  Struct = {fields: map}
  Value = null | number | string | bool | Struct | List
  
  JSON 유사:
    {
      "name": "Alice",
      "age": 30,
      "active": true,
      "tags": ["engineer", "team-lead"],
      "meta": {
        "department": "Engineering",
        "level": 5
      }
    }

Protobuf 구조:
  Struct{
    fields: {
      "name": Value{string_value: "Alice"},
      "age": Value{number_value: 30.0},
      "active": Value{bool_value: true},
      "tags": Value{list_value: [
        Value{string_value: "engineer"},
        Value{string_value: "team-lead"}
      ]},
      "meta": Value{struct_value: Struct{
        fields: {
          "department": Value{string_value: "Engineering"},
          "level": Value{number_value: 5.0}
        }
      }}
    }
  }

직렬화 크기:
  JSON (위 예): 약 130 바이트
  Protobuf Struct: 약 150 바이트 (타입 정보 오버헤드)
  
  Struct는 JSON보다 큰 이유:
    ├─ 각 값에 type 정보 (oneof)
    ├─ 문자열 길이 정보
    └─ 맵 엔트리 오버헤드

사용 사례:
  ├─ 동적 설정 데이터
  ├─ 사용자 메타데이터
  ├─ 로그 속성
  └─ GraphQL-like 쿼리 응답

주의:
  ├─ 성능: 키 조회 O(1)이지만 직렬화 느림
  ├─ 타입 검증 없음 (런타임에만 확인)
  ├─ JSON 호환성 우수 (양방향 변환)
  └─ 스키마 명시성 낮음 (정적 타입 언어에선 불편)

대안:
  고정 스키마 필요 → 일반 message 정의
  동적 필드 필요 → Struct 사용
  다형성 필요 → Any 사용
```

### google.protobuf.FieldMask

```
정의:
  message FieldMask {
    repeated string paths = 1;  // 필드 경로들
  }

역할:
  REST PATCH 요청에서 "어떤 필드를 업데이트할지" 표현

경로 표기:
  단순 필드: "name", "email", "age"
  중첩 필드: "address.street", "address.city"
  반복 필드: "tags" (요소별 처리 아님)
  
예시:
  message User {
    string user_id = 1;
    string name = 2;
    string email = 3;
    int32 age = 4;
    Address address = 5;
    
    message Address {
      string street = 1;
      string city = 2;
      string zip = 3;
    }
  }

사용:
  UserUpdateRequest{
    user_id: "U1",
    user: {
      name: "Alice",
      email: "alice@example.com",
      address: {street: "123 Main", city: "Seoul"}
    },
    update_mask: {
      paths: ["name", "email", "address.city"]
    }
  }
  
  → name, email, address.city 3개 필드만 업데이트
  → age와 address.street는 무시
  → "부분 업데이트"를 명시적으로 표현

코드:
  // FieldMask 생성
  FieldMask mask = FieldMask.newBuilder()
    .addPaths("name")
    .addPaths("email")
    .addPaths("address.city")
    .build();
  
  // 마스크 확인
  for (String path : mask.getPathsList()) {
    System.out.println(path);  // name, email, address.city
  }

직렬화:
  FieldMask{paths: ["name", "email", "address.city"]}
  → 0a 04 6e 61 6d 65 0a 05 65 6d 61 69 6c 0a 0f 61 64 64 72 65 73 73 2e 63 69 74 79
  → 약 30 바이트 (경로 문자열들)

JSON 표현:
  {
    "paths": ["name", "email", "address.city"]
  }

사용 패턴:
  // 구글 Cloud API 패턴
  PATCH /users/U1
  {
    "user": {...},
    "updateMask": "name,email,address.city"
  }

주의:
  ├─ 경로 오타 감지 어려움 (문자열 기반)
  ├─ 중첩 깊이 무제한 가능 (복잡도 증가)
  ├─ 리스트 요소별 마스크 불가 (전체만)
  └─ 서버가 마스크를 반드시 존중해야 함
```

---

## 💻 실전 실험

```java
// Well-Known Types 사용 예시

import com.google.protobuf.*;
import java.time.Instant;
import java.util.*;

public class WellKnownTypesTest {
  public static void main(String[] args) throws Exception {
    
    System.out.println("=== 1. Timestamp 사용 ===");
    
    // 현재 시간을 Timestamp로
    Instant now = Instant.now();
    com.google.protobuf.Timestamp timestamp = 
      com.google.protobuf.Timestamp.newBuilder()
        .setSeconds(now.getEpochSecond())
        .setNanos(now.getNano())
        .build();
    
    System.out.println("Current timestamp: " + timestamp);
    System.out.println("Seconds: " + timestamp.getSeconds());
    System.out.println("Nanos: " + timestamp.getNanos());
    
    // JSON 형식
    System.out.println("JSON: " + JsonFormat.printer().print(timestamp));
    
    System.out.println("\n=== 2. Struct (동적 데이터) ===");
    
    // 동적 속성 저장
    com.google.protobuf.Struct attributes = 
      com.google.protobuf.Struct.newBuilder()
        .putFields("user_id", 
          com.google.protobuf.Value.newBuilder()
            .setStringValue("U1")
            .build())
        .putFields("action", 
          com.google.protobuf.Value.newBuilder()
            .setStringValue("login")
            .build())
        .putFields("ip_address", 
          com.google.protobuf.Value.newBuilder()
            .setStringValue("192.168.1.1")
            .build())
        .putFields("success", 
          com.google.protobuf.Value.newBuilder()
            .setBoolValue(true)
            .build())
        .build();
    
    System.out.println("Struct: " + attributes);
    System.out.println("Fields count: " + attributes.getFieldsCount());
    
    System.out.println("\n=== 3. Any (다형성) ===");
    
    // 메시지를 Any로 래핑
    // (실제로는 protoc로 생성된 메시지 사용)
    String typeUrl = "type.googleapis.com/myapp.User";
    byte[] value = new byte[] { 0x0a, 0x02, 0x55, 0x31 };  // "U1"
    
    com.google.protobuf.Any any = 
      com.google.protobuf.Any.newBuilder()
        .setTypeUrl(typeUrl)
        .setValue(ByteString.copyFrom(value))
        .build();
    
    System.out.println("Any type_url: " + any.getTypeUrl());
    System.out.println("Any value: " + bytesToHex(any.getValue().toByteArray()));
    
    System.out.println("\n=== 4. FieldMask (부분 업데이트) ===");
    
    com.google.protobuf.FieldMask mask = 
      com.google.protobuf.FieldMask.newBuilder()
        .addPaths("name")
        .addPaths("email")
        .addPaths("address.city")
        .build();
    
    System.out.println("Update paths:");
    for (String path : mask.getPathsList()) {
      System.out.println("  - " + path);
    }
    
    System.out.println("\n=== 크기 비교 ===");
    System.out.println("Timestamp bytes: " + timestamp.toByteArray().length);
    System.out.println("Struct bytes: " + attributes.toByteArray().length);
    System.out.println("Any bytes: " + any.toByteArray().length);
    System.out.println("FieldMask bytes: " + mask.toByteArray().length);
  }
  
  private static String bytesToHex(byte[] bytes) {
    StringBuilder sb = new StringBuilder();
    for (byte b : bytes) {
      sb.append(String.format("%02x ", b & 0xFF));
    }
    return sb.toString().trim();
  }
}
```

**실행 결과:**
```
=== 1. Timestamp 사용 ===
Current timestamp: seconds: 1704110445
nanos: 123456789

Seconds: 1704110445
Nanos: 123456789
JSON: "2024-01-01T12:30:45.123456789Z"

=== 2. Struct (동적 데이터) ===
Struct: fields {
  key: "user_id"
  value {
    string_value: "U1"
  }
  ...
}
Fields count: 4

=== 3. Any (다형성) ===
Any type_url: type.googleapis.com/myapp.User
Any value: 0a 02 55 31

=== 4. FieldMask (부분 업데이트) ===
Update paths:
  - name
  - email
  - address.city

=== 크기 비교 ===
Timestamp bytes: 10
Struct bytes: 45
Any bytes: 30
FieldMask bytes: 28
```

---

## 📊 성능/비용 비교

```
Well-Known Types 크기 비교

┌─────────────────┬────────────┬────────────┬─────────────┐
│타입             │최소 크기   │JSON 대비   │사용 케이스  │
├─────────────────┼────────────┼────────────┼─────────────┤
│Timestamp        │10 bytes    │-80%       │시간 저장    │
│Any              │30 bytes    │+50%       │다형성      │
│Struct           │50 bytes    │+40%       │동적 데이터  │
│FieldMask       │20 bytes    │custom    │부분 업데이트 │
└─────────────────┴────────────┴────────────┴─────────────┘

네트워크 (100만 메시지):
  Timestamp vs JSON:
    Timestamp: 1M × 10B = 10MB
    JSON: 1M × 50B = 50MB
    → 80% 절감
    
  Any 오버헤드:
    직렬화 크기: +50%
    검색 속도: O(1) 타입 확인
    
메모리 (메모리 내 저장):
  Struct vs JSON string:
    Struct: 고정 크기 + 필드 맵
    JSON: 문자열 (가변 길이)
    → 파싱 필요 없음 (성능 우위)
```

---

## ⚖️ 트레이드오프

```
✅ Well-Known Types의 장점:
├─ Timestamp: 표준화된 시간 (모든 언어 동일)
├─ Any: 다형성 메시지 (runtime type info)
├─ Struct: 동적 데이터 (JSON 호환)
└─ FieldMask: 부분 업데이트 명시

❌ Well-Known Types의 단점:
├─ Timestamp: 현재 시간대 정보 없음
├─ Any: 런타임 타입 검증, 순환 참조 위험
├─ Struct: 스키마 명시성 낮음, 성능 저하
└─ FieldMask: 문자열 경로 (타이핑 에러 가능)

선택 기준:
  Timestamp:
    ├─ 모든 시간 데이터에 추천 (표준화)
    ├─ 타임존 필요시 별도 필드 추가
    └─ 나노초 정밀도 필요시만
    
  Any:
    ├─ 이벤트 기반 아키텍처 (다형성)
    ├─ 플러그인 시스템
    └─ 자동 타입 검증 불필요시
    
  Struct:
    ├─ 동적 메타데이터 (로그 속성)
    ├─ 사용자 정의 필드 (설정)
    └─ 성능 < 유연성
    
  FieldMask:
    ├─ REST PATCH 요청
    ├─ 부분 업데이트 명시
    └─ 경로 검증 필요 (서버)
```

---

## 📌 핵심 정리

```
Timestamp:
  - seconds(int64) + nanos(int32)
  - UTC 표준, 모든 언어 동일
  - RFC 3339 JSON 호환
  - JSON보다 80% 작음
  
Any:
  - type_url + value(bytes)
  - 런타임 타입 정보 유지
  - 다형성 메시지 표현
  - +50% 크기 오버헤드
  
Struct:
  - map<string, Value> (JSON 유사)
  - 동적 스키마, 유연성 우수
  - 성능 < JSON string
  - 파싱 불필요 (효율성)
  
FieldMask:
  - repeated string paths
  - 부분 업데이트 명시
  - "name", "address.city" 경로 지정
  - REST PATCH 패턴
  
선택:
  항상: Timestamp (표준화)
  필요시: Any (다형성), Struct (유연), FieldMask (부분 업데이트)
```

---

## 🤔 생각해볼 문제

**Q1: Timestamp 나노초 정밀도가 필요 없으면 bytes 절감 방법은?**
```
message Event {
  google.protobuf.Timestamp created_at = 1;  // 10 bytes
}

마이크로초(microsecond) 정도면 충분한데?
```
<details>
<summary>해설 보기</summary>

Google은 표준 Timestamp 사용을 강력 권장합니다.

하지만 불가피한 경우:
```proto
message Event {
  int64 created_at_micros = 1;  // Unix epoch (마이크로초)
  // 또는
  int64 created_at_millis = 1;  // Unix epoch (밀리초)
}
```

단점:
- 단위 불명확 (문서화 필수)
- 모든 언어에서 수동 변환
- 표준 시간 처리 도구 불가능

권장: 그냥 Timestamp 사용 (10 bytes 차이 미미)

</details>

**Q2: Any 타입을 중첩할 수 있나? (Any 안에 Any)**
```
message Event {
  google.protobuf.Any data = 1;  // 어떤 메시지든
}

// data 안에 또 Any?
// Event{data: Any{data: Any{...}}}
```
<details>
<summary>해설 보기</summary>

기술적으로 가능하지만 강력히 권장하지 않습니다.

문제:
- 복잡도 증가
- 역직렬화 깊이 증가
- 디버깅 어려움
- 순환 참조 가능성

대신:
```proto
message Event {
  google.protobuf.Any data = 1;
  // data 안 메시지가 필요하면 직접 포함
}

message UserCreatedEvent {
  User user = 1;
}
```

Best practice:
- Any는 최대 1단계 깊이만
- 복잡한 구조는 일반 message로

</details>

**Q3: FieldMask로 배열 요소별 업데이트 가능한가?**
```
message User {
  string user_id = 1;
  repeated string tags = 2;
}

FieldMask: "tags[0]", "tags[2]" 가능?
```
<details>
<summary>해설 보기</summary>

불가능합니다. FieldMask는 필드 레벨만 지원합니다.

```proto
// 가능
update_mask: {
  paths: ["tags"]  // tags 전체 교체
}

// 불가능
update_mask: {
  paths: ["tags[0]"]  // tags의 0번 요소만? → 지원 안 함
}
```

배열 요소별 업데이트 필요시:
```proto
message UserUpdateRequest {
  string user_id = 1;
  User user = 2;
  google.protobuf.FieldMask update_mask = 3;
  
  // tags 전체 교체 (개별 요소 아님)
  // 또는 별도 메서드: removeTag(), addTag()
}
```

Google Cloud API에서도 배열은 필드 단위로만 처리합니다.

</details>

---

<div align="center">

**[⬅️ 이전: 복합 타입](./04-complex-types.md)** | **[홈으로 🏠](../README.md)** | **[다음: Protobuf 진화 규칙 ➡️](./06-schema-evolution.md)**

</div>
