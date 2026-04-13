# 필드 번호가 API의 본체인 이유

---

## 🎯 핵심 질문

- 왜 "필드 이름"이 아니라 "필드 번호"로 통신할까?
- 컴파일 후 필드 이름은 정말 사라질까?
- 필드 번호를 재사용하면 왜 기존 데이터가 손상될까?
- 필드 번호를 안전하게 관리하려면?
- 1~15번과 16~2047번의 성능 차이는?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Protobuf은 필드 이름을 직렬화 데이터에 포함하지 않습니다. 대신 필드 번호만 저장해 크기를 50% 이상 줄입니다. 하지만 이는 필드 번호가 API 계약이 되어, 잘못된 번호 재할당은 런타임에 데이터를 조용히 손상시킵니다. 필드 번호 관리 실수는 카톡 메시지가 다른 사용자에게 전달되는 수준의 버그를 만듭니다.

---

## 😱 흔한 실수 (Before — 필드 번호를 경시하는 접근)

```
// 실수 1: 새로운 필드를 생각 없이 추가
message User {
  string name = 1;
  int32 age = 2;
  // 나중에 필드 추가
  string phone = 2;      // ❌ 기존 age 필드와 같은 번호!
}

// 문제: 기존 데이터에서 age=30을 읽으려 했는데,
// 새 코드에서는 이를 phone="..."로 해석함
// → 30이 전화번호로, 전화번호가 나이로 읽혀 DATA CORRUPTION

// 실수 2: 필드를 삭제 후 번호 재사용
message Event {
  string user_id = 1;
  int32 event_type = 2;
  string description = 3;  // 나중에 삭제
  // 버전 업데이트 후
  bool is_premium = 3;     // ❌ 3번 재사용!
}

// 문제: 구 버전 클라이언트에서 description="로그인"을 전송
// 신 버전 서버에서 이를 is_premium=true로 해석 (버그!)

// 실수 3: 필드 번호 계획 없이 막 할당
message Product {
  string sku = 100;        // 필드 번호 100? 2바이트 Tag로 낭비
  string name = 50;        // 필드 번호 50? 2바이트 Tag
  string description = 101;
  string category = 3;     // 자주 쓰는데 번호 3?
}
```

---

## ✨ 올바른 접근 (After — 필드 번호를 계약으로 관리하는 접근)

```
// 올바른 접근 1: 필드 번호 명확한 계획
message User {
  // 자주 쓰는 필드: 1~15 (1바이트 Tag)
  string id = 1;
  string name = 2;
  string email = 3;
  int32 age = 4;
  
  // 가끔 쓰는 필드: 16~30 (2바이트 Tag)
  string phone = 16;
  string address = 17;
  
  // 드물게 쓰는 필드: 31+ (2바이트 이상 Tag)
  string notes = 31;
  int64 created_at = 32;
}

// 올바른 접근 2: 필드 삭제 시 번호 보호
message Event {
  string user_id = 1;
  int32 event_type = 2;
  // string description = 3;  // 삭제됨
  
  // 번호 3은 다시 절대 쓰지 말 것! reserved로 보호
  reserved 3;
  
  // 새로운 필드는 4번부터
  string event_name = 4;
  bool is_premium = 5;
}

// 올바른 접근 3: 스키마 진화 전략
message Event {
  string user_id = 1;
  int32 event_type = 2;
  string event_name = 4;
  bool is_premium = 5;
  
  // 향후 추가: 필드 번호 미리 보유하지 말 것
  // 다음 사람이 6, 7, 8 자유롭게 쓸 수 있도록
  
  // 필드 번호 계획 문서 (예시)
  // 1~15: 핵심 필드 (항상 필요)
  // 16~30: 선택적 필드 (사용자/세션)
  // 31~100: 확장 필드 (분석/메타데이터)
  // 100+: 향후 예약
}
```

---

## 🔬 내부 동작 원리

### 필드 이름 vs 필드 번호

```
Proto 정의:
  message User {
    string name = 1;      ← 필드 번호 (필수!)
    int32 age = 2;        ← 필드 번호 (필수!)
    string email = 3;     ← 필드 번호 (필수!)
  }

코드 생성 (protoc):
  User.java
    - private String name;
    - private int age;
    - private String email;
    - public String getName() { ... }
    - public int getAge() { ... }
    - etc...

직렬화 (toByteArray()):
  User{name:"Alice", age:30, email:"alice@example.com"}
  
  ┌────────────────────────────────────────────────┐
  │ 0a 05 41 6c 69 63 65 | 10 1e | 1a 13 61 6c... │
  ├────────────────────────────────────────────────┤
  │ Tag(f=1,w=2)│5 bytes│Tag(f=2,w=0)│data...    │
  │ field=1     │"Alice" field=2    │"alice@..." │
  │ (필드명 없음!) │       │(필드명 없음!) │           │
  └────────────────────────────────────────────────┘

역직렬화 (parseFrom):
  bytes[] data = { 0x0a, 0x05, ... };
  
  1. Tag 0x0a 파싱 → field=1, wire=2 (length-delimited)
  2. Length 0x05 → 5바이트 읽음
  3. 데이터 처리: User::name = "Alice"
  
  4. Tag 0x10 파싱 → field=2, wire=0 (varint)
  5. Varint 디코딩: 0x1e = 30
  6. 데이터 처리: User::age = 30

=> 필드 번호만으로 어느 필드인지 완벽하게 판별!
```

### 필드 번호 재사용 시 데이터 손상 시나리오

```
시간 흐름:

[초기 상태] 2024년 1월
  message User {
    string name = 1;
    int32 age = 2;
    string phone = 3;
  }
  
  저장된 데이터: {name:"Alice", age:30, phone:"010-1234-5678"}
  바이트: 0a 05 41 6c 69 63 65 | 10 1e | 1a 0b 30 31 30 2d 31 32 33 34 2d 35 36 37 38

[문제 발생] 2024년 3월 (개발자 실수)
  message User {
    string name = 1;
    int32 age = 2;
    // string phone = 3;  ← 삭제
    string country = 3;  ← 3번 재사용! 😱
  }

[클라이언트 읽기 시도] 2024년 3월 신서버, 구버전 클라이언트
  
  구버전이 전송한 데이터:
    0a 05 41 6c 69 63 65 | 10 1e | 1a 0b 30 31 30 2d 31 32 33 34 2d 35 36 37 38
    └─ name="Alice"      └─age=30  └─ phone="010-1234-5678" (필드 3)
  
  신버전 서버 파싱:
    0a 05 41 6c 69 63 65  ← Field 1 (name="Alice") ✓
    10 1e                ← Field 2 (age=30) ✓
    1a 0b 30 31 30 2d... ← Field 3 (country=???)
    
    구서버는 phone이 문자열이지만,
    신서버는 country로 해석 → 데이터는 맞지만 필드가 다름!
    
    문제: User{name:"Alice", age:30, country:"010-1234-5678"}
    
이게 wire type이 같으면 더 심합니다:

[심한 예시] bool vs int32

  v1:
    message Transaction {
      string tx_id = 1;
      bool success = 2;      // wire=0 (varint)
      int32 amount = 3;      // wire=0 (varint)
    }
  
  v2: (개발자가 bool/int32 헷갈림)
    message Transaction {
      string tx_id = 1;
      int32 amount = 2;      // bool 대신 int32 (같은 필드 2!)
      bool success = 3;
    }
  
  구버전 클라이언트 → 신버전 서버:
    0a 02 tx1 | 08 01 | 10 64  (success=true, amount=100)
    신버전이 파싱:
    0a 02 tx1  ← tx_id="tx1"
    08 01      ← amount=1 (success의 1을 amount으로!) 😱😱😱
    10 64      ← success=true (amount의 100을 success로!)
    
    100원 거래가 1원으로 기록됨!
```

### reserved로 필드 번호 보호

```
Reserved 키워드: 삭제된 필드 번호를 미래에 쓰지 않기 위한 명시

// 방법 1: 번호로 예약
message Event {
  string event_id = 1;
  int32 event_type = 2;
  // string description = 3;  ← 삭제됨
  
  reserved 3;  // 이 번호는 영구히 예약
}

// 방법 2: 여러 번호 예약
message Event {
  reserved 3, 4, 5;  // 3~5번 모두 예약
}

// 방법 3: 범위 예약
message Event {
  reserved 100 to 500;  // 100~500번 모두 예약
}

// 방법 4: 필드명으로도 예약 (protobuf 버전 체킹)
message Event {
  reserved "description", "old_field";
}

컴파일 시:
  protoc를 실행하면 예약된 필드 번호를 쓰려고 하면 에러 발생
  
  message Event {
    string description = 3;  // ❌ 에러! "3 is reserved"
  }
  
  이렇게 실수를 미리 방지합니다.
```

### 필드 번호 할당 전략

```
효율성 고려:
  Tag = (field_number << 3) | wire_type
  
  필드 번호 1~15: 1바이트 Tag
    1 << 3 | 2 = 10 = 0x0a (1바이트)
    15 << 3 | 2 = 122 = 0x7a (1바이트)
  
  필드 번호 16~31: 2바이트 Tag
    16 << 3 | 2 = 130 = 0x82, 0x01 (2바이트)
    31 << 3 | 2 = 250 = 0xfa, 0x01 (2바이트)
  
  필드 번호 2048+: 3바이트 Tag
    2048 << 3 | 2 = 16386 = 0x82, 0x80, 0x01 (3바이트)

추천 할당:
  ┌─────────────────────────┬──────────────────────────┐
  │ 범위        │ 빈도    │ 예시                      │
  ├─────────────────────────┼──────────────────────────┤
  │ 1~15        │ 매우 자주│ id, name, email, status  │
  │ 16~31       │ 자주    │ age, created_at, tags    │
  │ 32~100      │ 가끔    │ metadata, config, extra  │
  │ 101~2047    │ 드물게  │ 향후 확장, 부외 필드     │
  │ 2048+       │ 미사용  │ (피하기)                  │
  └─────────────────────────┴──────────────────────────┘

메시지 설계 도구:
  
  message UserProfile {
    // 1~15: 핵심 필드 (항상 필요)
    string user_id = 1;      // PK
    string email = 2;        // 로그인 키
    string name = 3;         // 표시 이름
    int64 created_at = 4;    // 계정 생성일
    string status = 5;       // active/inactive
    
    // 16~31: 사용자 세부정보 (자주)
    string phone = 16;
    int32 age = 17;
    string country = 18;
    
    // 32~100: 선택적 필드 (가끔)
    string bio = 32;
    string profile_picture_url = 33;
    bool is_verified = 34;
    
    // 101+: 향후 확장 (미리 점유 금지)
    // 필요하면 나중에 101, 102, ... 사용
  }
```

---

## 💻 실전 실험

```bash
# protoc로 스키마 진화 감지

cat > user_v1.proto << 'EOF'
syntax = "proto3";

message User {
  string id = 1;
  string name = 2;
  int32 age = 3;
}
EOF

cat > user_v2.proto << 'EOF'
syntax = "proto3";

message User {
  string id = 1;
  string name = 2;
  // int32 age = 3;  // 삭제됨
  
  reserved 3;  // 보호!
  
  string email = 4;  // 새 필드
}
EOF
```

```java
// 필드 번호 재사용 버그 시뮬레이션

import java.util.*;

public class FieldNumberTest {
  public static void main(String[] args) throws Exception {
    // 시나리오: v1 User 메시지 생성 및 직렬화
    // message User {
    //   string name = 1;
    //   int32 age = 2;
    //   string phone = 3;
    // }
    
    byte[] v1Data = {
      0x0a, 0x05, // Tag(1), Length 5
      0x41, 0x6c, 0x69, 0x63, 0x65, // "Alice"
      0x10, 0x1e, // Tag(2), Value 30
      0x1a, 0x0b, // Tag(3), Length 11
      0x30, 0x31, 0x30, 0x2d, 0x31, 0x32, 0x33, 0x34, // "010-1234"
      0x2d, 0x35, 0x36, 0x37, 0x38 // "-5678"
    };
    
    System.out.println("=== v1 데이터 (name:Alice, age:30, phone:010-1234-5678) ===");
    System.out.println("Bytes: " + bytesToHex(v1Data));
    
    // v2에서 필드 3을 phone → country로 재사용
    // message User {
    //   string name = 1;
    //   int32 age = 2;
    //   string country = 3;  // phone 대신!
    // }
    
    System.out.println("\n=== v2 파싱 결과 (country 필드로 재사용) ===");
    System.out.println("name: Alice ✓");
    System.out.println("age: 30 ✓");
    System.out.println("country: 010-1234-5678 ❌ (phone이 country로!)");
    System.out.println("\n→ 데이터 손상! 이것이 field number contract 위반.");
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
=== v1 데이터 (name:Alice, age:30, phone:010-1234-5678) ===
Bytes: 0a 05 41 6c 69 63 65 10 1e 1a 0b 30 31 30 2d 31 32 33 34 2d 35 36 37 38

=== v2 파싱 결과 (country 필드로 재사용) ===
name: Alice ✓
age: 30 ✓
country: 010-1234-5678 ❌ (phone이 country로!)

→ 데이터 손상! 이것이 field number contract 위반.
```

---

## 📊 성능/비용 비교

```
필드 번호 범위별 Tag 크기 (wire type=2, length-delimited 기준)

┌──────────────┬──────────────┬──────────────┬──────────┐
│필드번호 범위 │Tag 바이트수  │메시지당 증가 │1M메시지 │
├──────────────┼──────────────┼──────────────┼──────────┤
│1~15         │1            │7필드 × 1B   │7MB      │
│16~31        │2            │7필드 × 2B   │14MB     │
│32~100       │2            │7필드 × 2B   │14MB     │
│100~200      │2            │7필드 × 2B   │14MB     │
│200~2047     │3            │7필드 × 3B   │21MB     │
│2048+        │4+           │7필드 × 4B+  │28MB+    │
└──────────────┴──────────────┴──────────────┴──────────┘

실무 메시지 (20필드, 자주쓰는 순):
  최적 설계 (1~15번): 메시지당 25B
  비최적 설계 (모두 100+번): 메시지당 35B
  
  초당 1M 메시지:
  최적: 25MB/s, 비최적: 35MB/s → 네트워크 비용 40% 증가!
```

---

## ⚖️ 트레이드오프

```
✅ 필드 번호 기반의 장점:
├─ 작은 직렬화 크기 (필드명 없음)
├─ 빠른 파싱 (숫자 비교만 하면 됨)
├─ Schema 진화 지원 (이전/신 클라이언트 호환)
└─ 언어 독립적 (모든 언어에서 동일 번호 의미)

❌ 필드 번호 기반의 단점:
├─ 한번 할당하면 번호는 영구 계약 (번호 변경 불가)
├─ 번호 재사용 실수 → 조용한 데이터 손상 (에러 미발생)
├─ 필드 번호 관리 부담 (reserved 필요)
└─ 필드명 변경은 자유롭지만 번호는 신중해야 함

안전 원칙:
├─ reserved로 삭제 필드 보호 (필수)
├─ 1~15번은 핵심 필드에만 할당
├─ 필드 번호 계획 문서 유지
└─ Code Review 시 필드 번호 변경 감지
```

---

## 📌 핵심 정리

```
필드 번호의 역할:
  1. 직렬화에서 필드명 대신 사용 → 크기 절감
  2. API 계약 → 변경 불가
  3. Wire format의 핵심 → 데이터 해석 기준
  
위험:
  번호 재사용 = 조용한 데이터 손상
  
관리:
  1. reserved로 삭제 필드 보호 (필수!)
  2. 1~15번: 핵심 필드만 (1바이트 Tag)
  3. 16+번: 추가 필드 (2바이트 이상 Tag)
  4. 번호 계획 문서 유지
  
결과:
  필드 번호는 "변경 불가능한 계약"
  proto 설계 시 가장 신중한 결정이 필요한 부분
```

---

## 🤔 생각해볼 문제

**Q1: 기존 프로덕션 User 메시지가 있고, 필드를 추가해야 한다면?**
```
현재:
  message User {
    string id = 1;
    string name = 2;
    int32 age = 3;  // 필드 3까지 사용 중
  }

새 버전에서 이메일을 추가하려면?
A) email = 3;     (age 필드와 같은 번호)
B) email = 4;     (다음 번호)
C) email = 100;   (충분히 떨어진 번호)

어느 것이 맞고 왜?
```
<details>
<summary>해설 보기</summary>

정답: B (email = 4)

이유:
- A는 data corruption (절대 금지)
- C는 비효율적 (Tag가 2바이트 → 네트워크 낭비)
- B가 정답 (age=3 다음이므로 4번이 자연스럽고, Tag는 1바이트)

다음 필드도 5, 6, 7... 순서대로 할당합니다.

</details>

**Q2: 마이크로서비스 A와 B가 같은 User proto를 공유할 때, A가 필드 번호를 변경하면?**
```
공유 User.proto:
  message User {
    string id = 1;
    string name = 2;
  }

A 팀이 필드를 추가하면서:
  message User {
    string id = 1;
    string name = 2;
    string email = 3;      // 신 필드
  }

B 팀이 아직 구 버전 사용 중일 때,
A → B로 User 전송하면?
```
<details>
<summary>해설 보기</summary>

B 팀에서 email 필드를 모르므로:
- A에서 email="alice@example.com"을 전송
- B에서 파싱할 때 필드 3을 무시 → OK
- 기존 데이터 손실 없음 ✓

역방향 (B → A)은 어떨까?
- B가 구 User{id, name}만 전송
- A에서 파싱할 때 email 필드 없음
- 기본값(빈 문자열) 처리 → OK

=> Forward compatible! (신 클라이언트 ← 구 서버)
   Backward compatible도 보장 (구 클라이언트 ← 신 서버)

이것이 Protobuf의 강점입니다.
단, 필드 번호가 바뀌면 이 호환성이 깨집니다!

</details>

**Q3: 필드 번호 1~15를 모두 쓴 메시지가 있는데, 새 필드를 추가해야 한다면?**
```
message LargeMessage {
  string field1 = 1;
  string field2 = 2;
  // ... 
  string field15 = 15;
  // 1~15 모두 사용 완료
  
  // 새 필드 추가?
  string field16 = ?;  // 어떤 번호를 쓸까?
}

A) field16 = 16 (2바이트 Tag)
B) 못 추가함 (Tag 낭비 피하기)
C) 필드 구조 재설계
```
<details>
<summary>해설 보기</summary>

정답: A (field16 = 16)

1~15번이 모두 차면 자연스럽게 16부터 시작합니다.
2바이트 Tag의 낭비는 감수해야 합니다.

하지만 이런 상황이 자주 생기는 건 설계 문제입니다:
- 1~15번을 "자주 쓰는 필드"로 엄격하게 관리
- 초기에 자주 쓸 필드들만 1~15번 할당
- 가끔 쓸 필드는 16번부터

예: 
  1~10번: 핵심 필드 (항상 필요)
  11~15번: 일반 필드 (자주)
  16~100번: 선택적 필드 (가끔)
  
이렇게 미리 계획하면 1~15번이 고갈되지 않습니다.

</details>

---

<div align="center">

**[⬅️ 이전: Protobuf 직렬화 원리](./01-tlv-encoding.md)** | **[홈으로 🏠](../README.md)** | **[다음: 스칼라 타입과 기본값 ➡️](./03-scalar-types-defaults.md)**

</div>
