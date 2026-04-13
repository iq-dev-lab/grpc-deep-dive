# Protobuf 진화 규칙 — Backward/Forward Compatibility

---

## 🎯 핵심 질문

- Backward compatible은 뭐고 Forward compatible은 뭐가 다른가?
- 신 서버가 구 클라이언트 데이터를 어떻게 처리할까?
- 필드 타입을 변경해도 호환되나?
- 필드를 삭제하면 뭐가 문제일까?
- reserved 키워드를 언제 써야 하나?

---

## 🔍 왜 이 개념이 실무에서 중요한가

마이크로서비스 환경에서 서버와 클라이언트가 다른 속도로 배포됩니다. Protobuf은 스키마 호환성을 통해 이를 지원하지만, 호환성 규칙을 어기면 무한 순환 요청, 데이터 손실, 자동 실패 같은 버그가 발생합니다. 금융 시스템에서 금액을 잘못 저장하는 일도 스키마 진화 실수에서 비롯됩니다.

---

## 😱 흔한 실수 (Before — 호환성 규칙을 무시하는 접근)

```
// 실수 1: 필드 타입 변경 (int32 → int64)
// v1:
message Transaction {
  string tx_id = 1;
  int32 amount = 2;      // 32비트 정수
}

// v2 (잘못 변경):
message Transaction {
  string tx_id = 1;
  int64 amount = 2;      // 64비트로 변경? 호환 깨짐!
}

// 문제: wire type이 같아도 범위가 다름
// 구 클라이언트: amount=2000000000 (2B)
// 신 서버: 이를 int32로 읽음 → overflow 발생

// 실수 2: 필드 삭제 후 다시 사용
// v1:
message Event {
  string event_id = 1;
  int32 event_type = 2;
  string description = 3;  // 필드 3
}

// v1.5 (잘못된 삭제):
message Event {
  string event_id = 1;
  int32 event_type = 2;
  // string description = 3;  // 삭제, 번호 3 해방
}

// v2 (필드 3 재사용):
message Event {
  string event_id = 1;
  int32 event_type = 2;
  bool is_critical = 3;    // 번호 3 재사용? 재앙!
}

// 문제: 구 클라이언트가 description="로그인 실패"를 전송
// 신 서버: 필드 3을 is_critical=true로 해석 → 데이터 손상

// 실수 3: 필드 이름만 변경, 번호는 유지
message User {
  string user_id = 1;
  string name = 2;         // 이전 이름
}

// 다음 버전:
message User {
  string user_id = 1;
  string user_name = 2;    // 이름 변경 (번호는 2로 유지)
}

// 원래 호환되지만, 필드명 변경은 클라이언트 코드에 영향
// 클라이언트: getName() → 없음 (getUserName() 필요)

// 실수 4: required 필드 추가 (proto2)
message User {
  required string user_id = 1;
  // required string email = 2;  추가 시 기존 메시지 파싱 실패!
}
```

---

## ✨ 올바른 접근 (After — 호환성 규칙을 지키는 접근)

```
// 올바른 접근 1: 필드 타입 변경 금지
// v1:
message Transaction {
  string tx_id = 1;
  int32 amount = 2;  // int32로 고정
}

// v2 (타입 변경 필요시):
// 방법 1: 새 필드 추가
message Transaction {
  string tx_id = 1;
  int32 amount = 2;        // 호환성을 위해 유지
  int64 amount_cents = 3;  // 새 필드 (더 큰 범위)
  // 클라이언트가 amount_cents 사용으로 마이그레이션
}

// 방법 2: 버전 분리 (더 나음)
// v1 메시지는 그대로 두고,
// v2 패키지에 새 메시지 정의
package v2;
message TransactionV2 {
  string tx_id = 1;
  int64 amount = 2;  // v2에서는 int64 가능
}

// 올바른 접근 2: 필드 삭제 시 reserved
// v1:
message Event {
  string event_id = 1;
  int32 event_type = 2;
  string description = 3;
}

// v2 (설명 필드 제거, 호환성 유지):
message Event {
  string event_id = 1;
  int32 event_type = 2;
  // string description = 3;  // 삭제
  
  // 번호 3 영구 예약 (재사용 금지!)
  reserved 3;
  
  // 새 필드는 4번부터
  bool is_critical = 4;
}

// 올바른 접근 3: 필드명 변경은 안전 (번호 유지)
message User {
  string user_id = 1;
  string name = 2;         // 원래 이름
}

// 다음 버전:
message User {
  string user_id = 1;
  string display_name = 2;  // 이름만 변경 (번호 2 유지)
  // 직렬화/역직렬화는 번호로 동작하므로 호환
  // 하지만 클라이언트 코드 업데이트 필요
}

// 올바른 접근 4: optional으로 선택적 필드 추가
message User {
  string user_id = 1;
  string name = 2;
  optional string email = 3;    // 새 필드, optional
  optional string phone = 4;
}

// 호환성:
// 구 클라이언트: email, phone 모르고 생략해서 전송
// 신 서버: 부재 필드는 기본값으로 처리
// → 완벽한 호환성!
```

---

## 🔬 내부 동작 원리

### Backward Compatibility (구 클라이언트 ← 신 서버)

```
시나리오: 구 버전 클라이언트, 신 버전 서버

v1 스키마:
  message User {
    string user_id = 1;
    string name = 2;
    int32 age = 3;
  }

v2 스키마 (신 서버):
  message User {
    string user_id = 1;
    string name = 2;
    int32 age = 3;
    optional string email = 4;   // 새 필드 추가
    optional string phone = 5;   // 새 필드 추가
  }

구 클라이언트 요청:
  User{user_id:"U1", name:"Alice", age:30}
  
  직렬화:
    0a 02 55 31 | 12 05 41 6c 69 63 65 | 18 1e
    └─ field1  └─ field2              └─ field3

신 서버 역직렬화:
  Tag 0x0a → field=1, wire=2 → user_id="U1" ✓
  Tag 0x12 → field=2, wire=2 → name="Alice" ✓
  Tag 0x18 → field=3, wire=0 → age=30 ✓
  (필드 4, 5는 수신 안 함)
  
  파싱 결과: User{user_id:"U1", name:"Alice", age:30, email:null, phone:null}
  
  처리: email, phone 기본값으로 처리 → OK!

호환성 보장:
  ✓ 새 필드는 무시됨
  ✓ 기존 필드 변경 없음
  ✓ 기본값으로 안전하게 처리

조건:
  ├─ 새 필드만 추가 (기존 필드 수정 X)
  ├─ 필드 번호 변경 X
  ├─ 필드 타입 변경 X
  └─ 필드 삭제 X (reserved만 가능)
```

### Forward Compatibility (신 클라이언트 ← 구 서버)

```
시나리오: 신 버전 클라이언트, 구 버전 서버

v1 스키마 (구 서버):
  message User {
    string user_id = 1;
    string name = 2;
    int32 age = 3;
  }

신 클라이언트 (v2 스키마):
  message User {
    string user_id = 1;
    string name = 2;
    int32 age = 3;
    optional string email = 4;
    optional string phone = 5;
  }

신 클라이언트 요청:
  User{user_id:"U1", name:"Alice", age:30, email:"alice@example.com"}
  
  직렬화:
    0a 02 55 31 | 12 05 41 6c 69 63 65 | 18 1e | 22 12 61 6c...
    └─ field1  └─ field2              └─ f3  └─ field4

구 서버 역직렬화:
  Tag 0x0a → field=1 → user_id="U1" ✓
  Tag 0x12 → field=2 → name="Alice" ✓
  Tag 0x18 → field=3 → age=30 ✓
  Tag 0x22 → field=4, wire=2 → 모르는 필드!
  
  처리: 미지의 필드는 "unknown field"로 저장 (무시)
  파싱 결과: User{user_id:"U1", name:"Alice", age:30}
  
  응답: User (email 필드 없음)

신 클라이언트 역직렬화:
  Tag 0x0a → user_id="U1" ✓
  Tag 0x12 → name="Alice" ✓
  Tag 0x18 → age=30 ✓
  (email 필드는 response에 없음)
  
  처리: email 기본값(null) → OK!

호환성 보장:
  ✓ 미지의 필드 무시 (unknown field)
  ✓ 기본값으로 안전하게 처리
  ✓ 메시지 구조 변경 없음

조건:
  ├─ 필드 번호 변경 X
  ├─ 필드 타입 변경 X (wire type 동일 필요)
  └─ 기존 필드 의미 변경 X
```

### Protobuf 타입 호환성 매트릭스

```
같은 필드 번호에서 타입 변경 가능성:

┌──────────────────┬─────────────────┬────────────────────┐
│원래 타입         │변경 가능한 타입 │호환성              │
├──────────────────┼─────────────────┼────────────────────┤
│int32             │uint32           │부분 (음수 문제)   │
│int32             │sint32           │부분 (인코딩 다름) │
│int32 ← int64     │O (확장)        │O 역호환성         │
│int64 → int32     │X (축소)        │X 데이터 손실      │
│float ← double    │O (확장)        │O 역호환성         │
│double → float    │X (축소)        │X 정밀도 손실      │
│string ← bytes    │O (같은 wire)    │문제 (인코딩)      │
│message ← 다른 msg│X (구조 다름)    │X 파싱 실패        │
└──────────────────┴─────────────────┴────────────────────┘

규칙:
  같은 Wire Type일 때만 변경 고려
  └─ Wire Type 0: int32, uint32, sint32, bool, enum
  └─ Wire Type 1: double, sfixed64, fixed64
  └─ Wire Type 2: string, bytes, message, packed repeated
  └─ Wire Type 5: float, sfixed32, fixed32

안전한 변경:
  ├─ 필드 추가 (새 번호)
  ├─ 필드 삭제 (reserved 필수)
  ├─ 필드명 변경 (번호 유지)
  └─ 타입 "호환" 변경 (같은 wire type, 데이터 범위 고려)

위험한 변경:
  ├─ 필드 타입 변경 (int32 ← int64)
  ├─ 필드 번호 변경
  ├─ 필드 삭제 (reserved 없이)
  └─ 필드 번호 재사용
```

### reserved로 필드 보호

```
Proto 정의:
  message Event {
    string event_id = 1;
    int32 event_type = 2;
    // string description = 3;  ← 삭제됨
    
    // 번호 3 영구 예약
    reserved 3;
  }

또는:
  message Event {
    reserved 3, 4, 5;  // 여러 번호 예약
  }

또는:
  message Event {
    reserved 3 to 10;  // 범위 예약
  }

또는:
  message Event {
    reserved "description";  // 필드명으로 예약
  }

효과:
  1. 컴파일 타임 보호
     → 다음 개발자가 reserved 번호를 사용하려고 하면 에러
  2. 문서화
     → 왜 3번이 없는지 명시적
  3. 상호운용성 보장
     → 구/신 버전이 안전하게 공존

주의:
  reserved 없이 필드를 삭제하면:
    → 미래 개발자가 번호 3을 모르고 재사용 가능
    → 구 버전 클라이언트 데이터가 손상됨!
```

---

## 💻 실전 실험

```bash
# 호환성 검증 (buf 도구)

cat > user_v1.proto << 'EOF'
syntax = "proto3";

package myapp.v1;

message User {
  string user_id = 1;
  string name = 2;
  int32 age = 3;
}
EOF

cat > user_v2.proto << 'EOF'
syntax = "proto3";

package myapp.v2;

message User {
  string user_id = 1;
  string name = 2;
  int32 age = 3;
  optional string email = 4;      // 새 필드 추가 (호환)
  optional string phone = 5;      // 새 필드 추가 (호환)
}
EOF

# buf 설치 및 호환성 검사
# brew install bufbuild/buf/buf

# Backward compatibility 검사 (구 클라이언트 ← 신 서버)
# buf breaking check user_v1.proto --against-input user_v2.proto

# Forward compatibility 검사 (신 클라이언트 ← 구 서버)
# → Protobuf는 자동으로 unknown field 무시하므로 보장됨
```

```java
// 호환성 시뮬레이션

import java.util.*;

public class CompatibilityTest {
  public static void main(String[] args) throws Exception {
    
    System.out.println("=== Backward Compatibility 테스트 ===");
    System.out.println("구 클라이언트 → 신 서버");
    
    // 구 버전 메시지 (v1)
    byte[] v1Bytes = {
      0x0a, 0x02, 0x55, 0x31,      // field1: "U1"
      0x12, 0x05, 0x41, 0x6c, 0x69, 0x63, 0x65,  // field2: "Alice"
      0x18, 0x1e                    // field3: 30
    };
    
    System.out.println("v1 메시지 바이트: " + bytesToHex(v1Bytes));
    System.out.println("필드: user_id=U1, name=Alice, age=30");
    
    // 신 버전 서버에서 역직렬화
    System.out.println("\n신 서버 (v2) 역직렬화:");
    System.out.println("field1(user_id): U1 ✓");
    System.out.println("field2(name): Alice ✓");
    System.out.println("field3(age): 30 ✓");
    System.out.println("field4(email): null (미수신) ✓");
    System.out.println("field5(phone): null (미수신) ✓");
    System.out.println("→ Backward compatible! 기본값으로 안전 처리");
    
    System.out.println("\n=== Forward Compatibility 테스트 ===");
    System.out.println("신 클라이언트 → 구 서버");
    
    // 신 버전 메시지 (v2)
    byte[] v2Bytes = {
      0x0a, 0x02, 0x55, 0x31,      // field1: "U1"
      0x12, 0x05, 0x41, 0x6c, 0x69, 0x63, 0x65,  // field2: "Alice"
      0x18, 0x1e,                   // field3: 30
      0x22, 0x12, 0x61, 0x6c, 0x69, 0x63, 0x65,  // field4: "alice@example.com" (새 필드)
      0x40, 0x65, 0x78, 0x61, 0x6d, 0x70, 0x6c, 0x65
    };
    
    System.out.println("v2 메시지 바이트: " + bytesToHex(v2Bytes));
    System.out.println("필드: user_id=U1, name=Alice, age=30, email=alice@example.com");
    
    // 구 버전 서버에서 역직렬화
    System.out.println("\n구 서버 (v1) 역직렬화:");
    System.out.println("field1(user_id): U1 ✓");
    System.out.println("field2(name): Alice ✓");
    System.out.println("field3(age): 30 ✓");
    System.out.println("field4(unknown): 무시됨 (unknown field) ✓");
    System.out.println("→ Forward compatible! unknown field 무시");
    System.out.println("→ 서버 응답: User{user_id:U1, name:Alice, age:30}");
    
    System.out.println("\n=== 호환성 위반 (필드 번호 재사용) ===");
    System.out.println("v1: field3 = int32 age");
    System.out.println("v2: field3 = bool is_critical");
    
    // 구 클라이언트 데이터
    byte[] wrongData = {
      0x0a, 0x02, 0x55, 0x31,      // field1: "U1"
      0x12, 0x05, 0x41, 0x6c, 0x69, 0x63, 0x65,  // field2: "Alice"
      0x18, 0x1e                    // field3: 30 (int32)
    };
    
    System.out.println("구 클라이언트 데이터: age=30");
    System.out.println("신 서버 해석: is_critical=true (30 != 0) ❌");
    System.out.println("→ 데이터 손상! 나이 30이 중요도 플래그로!?");
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
=== Backward Compatibility 테스트 ===
구 클라이언트 → 신 서버
v1 메시지 바이트: 0a 02 55 31 12 05 41 6c 69 63 65 18 1e
필드: user_id=U1, name=Alice, age=30
신 서버 (v2) 역직렬화:
field1(user_id): U1 ✓
field2(name): Alice ✓
field3(age): 30 ✓
field4(email): null (미수신) ✓
field5(phone): null (미수신) ✓
→ Backward compatible! 기본값으로 안전 처리

=== Forward Compatibility 테스트 ===
신 클라이언트 → 구 서버
v2 메시지 바이트: 0a 02 55 31 12 05 41 6c 69 63 65 18 1e 22 12 61 6c ...
필드: user_id=U1, name=Alice, age=30, email=alice@example.com
구 서버 (v1) 역직렬화:
field1(user_id): U1 ✓
field2(name): Alice ✓
field3(age): 30 ✓
field4(unknown): 무시됨 (unknown field) ✓
→ Forward compatible! unknown field 무시
→ 서버 응답: User{user_id:U1, name:Alice, age:30}

=== 호환성 위반 (필드 번호 재사용) ===
v1: field3 = int32 age
v2: field3 = bool is_critical
구 클라이언트 데이터: age=30
신 서버 해석: is_critical=true (30 != 0) ❌
→ 데이터 손상! 나이 30이 중요도 플래그로!?
```

---

## 📊 성능/비용 비교

```
호환성 유지를 위한 설계 패턴 비용

┌─────────────────────┬──────────┬──────────┬─────────────────┐
│패턴                 │메시지크기│파싱속도  │유지보수          │
├─────────────────────┼──────────┼──────────┼─────────────────┤
│필드 추가 (optional) │+1 Tag   │동일     │간단 (호환성 우수)│
│필드 타입 변경       │변함      │느려짐   │복잡 (호환성 약함)│
│required → optional  │+Tag     │동일     │간단              │
│버전 분리 (v1,v2)   │동일     │동일     │복잡 (이중 관리) │
└─────────────────────┴──────────┴──────────┴─────────────────┘

마이그레이션 비용 (필드 타입 변경: int32 → int64):

방법 1: 신 필드 추가
  + 호환성 100% 유지
  - 메시지 크기 증가 (2개 필드)
  - 클라이언트 마이그레이션 오래 (dual write)

방법 2: 버전 분리 (package v1, v2)
  + 깔끔한 스키마
  - 이중 코드 생성 (protoc v1.proto, v2.proto)
  - 서버에서 양쪽 처리

방법 3: 강제 마이그레이션
  + 메시지 크기 최소화
  - 호환성 단절 (구 클라이언트 실패)
  - 데이터 손실 가능성

권장: 방법 1 (필드 추가) → 이후 서서히 마이그레이션
```

---

## ⚖️ 트레이드오프

```
✅ 호환성 유지의 장점:
├─ 서버/클라이언트 독립 배포 가능
├─ 무중단 배포 (zero downtime)
├─ 구 버전 클라이언트도 서비스 가능
└─ 데이터 손실 없음

❌ 호환성 유지의 단점:
├─ 메시지 구조 진화 제약 (필드 추가만 권장)
├─ 레거시 필드 유지 (이중 필드)
├─ 복잡한 마이그레이션 (dual write)
└─ 스키마 관리 부담

설계 원칙:
  필드 추가:
    ├─ 항상 새 번호로 (재사용 금지)
    ├─ optional로 선택적 표현
    └─ reserved로 삭제 필드 보호
    
  필드 수정:
    ├─ 타입 변경 피하기 (새 필드 추가)
    ├─ 타입 변경 필수시 wire type 동일 확인
    └─ 명시적 마이그레이션 기간 필요
    
  필드 삭제:
    ├─ 반드시 reserved로 보호
    ├─ 문서에 삭제 이유 기록
    └─ 충분한 사용 중단 기간 필요
```

---

## 📌 핵심 정리

```
Backward Compatible (구 클라이언트 ← 신 서버):
  구 클라이언트의 데이터를 신 서버가 처리 가능
  조건: 기존 필드 변경/삭제 X, 새 필드만 추가
  
Forward Compatible (신 클라이언트 ← 구 서버):
  신 클라이언트의 데이터를 구 서버가 처리 가능
  조건: unknown field 무시 (Protobuf 기본)
  
호환성 규칙:
  1. 필드 추가: 새 번호 사용 (항상 안전)
  2. 필드 삭제: reserved 필수 (재사용 금지)
  3. 필드 수정: 타입/번호 변경 금지
  4. 필드명 변경: 번호 유지 (직렬화 안전)
  
reserved 사용:
  - 삭제 필드 번호 영구 예약
  - 컴파일 타임 보호
  - 미래 재사용 방지
  
마이그레이션:
  점진적 진화 권장
  (새 필드 → dual write/read → 구 필드 삭제+reserved)
```

---

## 🤔 생각해볼 문제

**Q1: 필드 타입을 int32에서 int64로 변경해야 한다면?**
```
현재:
  message User {
    int32 user_id = 1;
  }

user_id가 30억 이상 필요함. 어떻게?
```
<details>
<summary>해설 보기</summary>

방법 1: 새 필드 추가 (권장)
```proto
message User {
  int32 user_id = 1;              // 호환성 유지
  int64 user_id_v2 = 2;           // 새 필드
}
```

마이그레이션:
1. user_id_v2에 값 쓰기 시작
2. 구 클라이언트가 user_id_v2 해석 시작하기까지 대기
3. 충분한 시간 후 user_id 삭제, reserved 1로 변경

방법 2: 버전 분리
```proto
package myapp.v2;
message User {
  int64 user_id = 1;  // v2에서는 int64
}
```

단점:
- 이중 메시지 관리
- 클라이언트가 v1→v2 마이그레이션 필요

권장: 방법 1 (호환성 최우선)

</details>

**Q2: optional 필드가 정말 backward compatible할까?**
```
v1 (optional 없음):
  message User {
    string email = 3;  // 일반 필드
  }

v2 (optional 추가):
  message User {
    optional string email = 3;  // optional로 변경
  }

호환성 보장?
```
<details>
<summary>해설 보기</summary>

네, 호환성 완벽합니다.

비교:
- 일반 필드: 기본값이면 직렬화 생략
- optional 필드: 모든 값 직렬화 (기본값도)

v1 클라이언트가 email="alice@example.com" 전송
→ v2 서버: email="alice@example.com", hasEmail()=true

v1 클라이언트가 email 생략
→ v2 서버: hasEmail()=false, email=""

v2 클라이언트가 email 설정
→ v1 서버: email 수신 (필드 3), 문자열로 처리

v2 클라이언트가 email 미설정
→ v1 서버: email 생략, 기본값("")로 처리

모두 호환성 유지!

</details>

**Q3: 필드를 삭제했는데 reserved를 깜빡했다면?**
```
v1:
  message Event {
    string event_id = 1;
    string description = 2;  // 필드 2
  }

v2 (실수):
  message Event {
    string event_id = 1;
    // description 삭제, reserved 안 함!
  }

v3 (나중):
  message Event {
    string event_id = 1;
    bool is_critical = 2;  // 번호 2 재사용
  }

문제?
```
<details>
<summary>해설 보기</summary>

데이터 손상!

v1 클라이언트 → v3 서버:
- 클라이언트: description="로그인 성공"를 필드 2에
- 서버: 필드 2를 is_critical=true로 해석
- 결과: 로그인 메시지가 중요도로 변환됨 (버그)

해결:
v2에서라도 reserved 2를 추가:
```proto
message Event {
  string event_id = 1;
  reserved 2;  // 늦어도 지금 추가!
}
```

그 후 v3에서:
```proto
message Event {
  string event_id = 1;
  reserved 2;
  bool is_critical = 3;  // 번호 3 사용
}
```

교훈:
필드 삭제할 때 reserved를 깜빡하면 재앙입니다.
Code Review 시 반드시 체크!

</details>

---

<div align="center">

**[⬅️ 이전: Well-Known Types](./05-well-known-types.md)** | **[홈으로 🏠](../README.md)** | **[다음: 직렬화 성능 측정 ➡️](./07-serialization-benchmark.md)**

</div>
