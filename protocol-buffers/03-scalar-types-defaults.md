# 스칼라 타입과 기본값 — 없는 필드는 전송되지 않는다

---

## 🎯 핵심 질문

- proto3에서 int32 age = 0;은 직렬화되나?
- "가격 0원"과 "가격 설정 안 함"을 구분할 수 있나?
- optional 키워드는 언제 써야 하나?
- hasField()는 어떻게 작동하는가?
- 기본값이 생략되면 데이터 손실이 될까?

---

## 🔍 왜 이 개념이 실무에서 중요한가

proto3에서는 기본값인 필드를 직렬화하지 않아 용량을 더 줄입니다. 그러나 "가격 0원"과 "가격 미설정"을 구분해야 하는 경우, optional 키워드가 없으면 API 계약을 실패합니다. 결제 시스템에서 금액 0을 누락 필드로 착각하는 버그는 매달 수십만 원의 손실을 초래할 수 있습니다.

---

## 😱 흔한 실수 (Before — 기본값을 무시하는 접근)

```
// 실수 1: 금액이 0인 경우 구분 불가
message Invoice {
  string invoice_id = 1;
  int32 total_amount = 2;  // 0원 할인은? 필드 안 보냄?
}

// 클라이언트: Invoice{id:"INV001", total_amount:0}
// 직렬화: 0a 06 49 4e 56 30 30 31 (total_amount 생략!)
// 서버 역직렬화: Invoice{id:"INV001"} (amount=0, 기본값)
// 서버 해석: "금액 설정 안 함 vs 0원 할인" 구분 불가!

// 실수 2: optional 없이 부재 표현 시도
message UserProfile {
  string user_id = 1;
  int32 age = 2;        // optional이 없으면
  bool is_verified = 3; // 필드 부재를 표현 불가
}

// age가 없는 사람: age=0 (어린이?) vs age 미설정?

// 실수 3: 선택적 필드를 string으로 감싸기
message Product {
  string sku = 1;
  string discount_amount = 2;  // int32가 아니라 string으로?
                               // JSON 호환성 때문에?
                               // → 타입 안정성 상실
}
```

---

## ✨ 올바른 접근 (After — optional과 명시적 기본값 처리)

```
// 올바른 접근 1: optional로 필드 부재 명시
message Invoice {
  string invoice_id = 1;
  int32 total_amount = 2;
  optional int32 discount_amount = 3;  // 없을 수도, 있을 수도
  
  // hasDiscountAmount() 사용 가능
  // getDiscountAmount() = 0 (기본값, 설정 안 됨)
  // vs getDiscountAmount() = 5 (명시적 0) 구분 가능
}

// 올바른 접근 2: oneof로 상호 배타적 필드 표현
message Transaction {
  string tx_id = 1;
  oneof payment {
    int32 amount_cents = 2;  // 금액
    bool is_free = 3;        // 무료 거래
  }
  // 하나만 설정 가능! 둘 다 0/false인 경우 없음
}

// 올바른 접근 3: 명확한 기본값 명시
message Configuration {
  string config_id = 1;
  int32 retry_count = 2;  // 기본값: 3 (명시적으로)
  // 기본값이 0이 아니면?
  // → Oneof 패턴으로 우회하거나
  // → Wrapper 타입 사용 (google.protobuf.Int32Value)
}

// 올바른 접근 4: Wrapper 타입으로 명시적 null 표현
import "google/protobuf/wrappers.proto";

message UserProfile {
  string user_id = 1;
  google.protobuf.Int32Value age = 2;  // 0, null, 값 구분 가능
  google.protobuf.StringValue nickname = 3;  // "", null, 값
}
```

---

## 🔬 내부 동작 원리

### proto3 기본값과 직렬화

```
Proto 정의:
  syntax = "proto3";  // proto2가 아니라 proto3!
  
  message User {
    string name = 1;
    int32 age = 2;
    bool is_active = 3;
    string email = 4;
  }

proto3의 기본값:
  ┌────────────┬────────────────┐
  │타입        │기본값           │
  ├────────────┼────────────────┤
  │int32/64    │0               │
  │uint32/64   │0               │
  │sint32/64   │0               │
  │float/double│0.0             │
  │bool        │false           │
  │string      │"" (빈 문자열)  │
  │bytes       │b"" (빈 바이트) │
  │enum        │첫 enum value   │
  │repeated    │[] (빈 리스트)  │
  │message     │null (설정안됨) │
  └────────────┴────────────────┘

직렬화 규칙 (proto3):
  기본값인 필드 → 직렬화하지 않음!
  
예시:
  User{name:"", age:0, is_active:false, email:""}
  → 빈 메시지 (0 바이트!)
  
  User{name:"Alice", age:0, is_active:false, email:""}
  → 필드 1만 직렬화
    0a 05 41 6c 69 63 65 (7 바이트)

문제점:
  역직렬화 시 기본값 필드는 복원 불가능!
  
  직렬화: User{name:"Alice", age:0} → [0x0a, 0x05, ...]
  역직렬화: ← User{name:"Alice", age:0} (age가 기본값)
  
  하지만 이게 다를 수 있음:
  원본: age 명시적으로 0으로 설정
  역직렬화: age 설정 안 됨 (기본값 0)
  → 구분 불가!
```

### optional 키워드 (proto3.12+)

```
Proto 정의:
  syntax = "proto3";
  
  message User {
    string user_id = 1;
    optional int32 age = 2;      // ← optional 키워드
    string email = 3;
    optional string phone = 4;
  }

동작:
  age가 설정되지 않으면:
    직렬화: 필드 2 생략
    역직렬화: hasAge() = false, getAge() = 0
    
  age=0으로 명시 설정되면:
    직렬화: Tag(2) + Value(0) 포함!
    역직렬화: hasAge() = true, getAge() = 0
    
  age=30으로 설정되면:
    직렬화: Tag(2) + Varint(30)
    역직렬화: hasAge() = true, getAge() = 30

바이트 레벨:
  User{user_id:"U001", age:0}
  → 0a 04 55 30 30 31 | 10 00  (필드 2 포함!)
      ↑ field1        ↑ field2, value=0
  
  User{user_id:"U001"}  (age 생략)
  → 0a 04 55 30 30 31  (필드 2 없음!)

생성된 코드:
  boolean hasAge();                    // 명시적 설정 여부
  int getAge();                        // 값 (미설정시 0)
  Builder setAge(int value);
  Builder clearAge();
```

### Wrapper 타입 (구글 제공)

```
파일: google/protobuf/wrappers.proto

정의:
  message Int32Value {
    int32 value = 1;
  }

사용:
  message UserProfile {
    string user_id = 1;
    google.protobuf.Int32Value age = 2;  // int32 대신
  }

동작:
  age가 null (설정 안 됨):
    직렬화: 필드 2 생략 (null)
    
  age=0으로 설정:
    직렬화: 
      Tag(2) = 0x12
      Length = 0x01 (1 바이트)
      Value = 0x08 0x00 (nested Int32Value{value:0})
    
  age=30:
    직렬화: 0x12 0x01 0x08 0x1e

장점:
  null (미설정) vs 0 vs 30을 모두 구분 가능!

단점:
  크기 증가: null일 때도 header 포함
  복잡도: 매번 null check 필요

언제 쓸까:
  ├─ 선택적 숫자 필드 (선택사항)
  ├─ JSON API 호환성 (JSON null 표현)
  ├─ 레거시 proto2 호환
  └─ oneof로 불충분할 때
```

### oneof로 상호 배타적 필드 표현

```
Proto 정의:
  message Payment {
    string payment_id = 1;
    oneof payment_method {
      int32 amount_cents = 2;    // 유료 결제
      bool is_free_trial = 3;    // 무료
      string gift_card_id = 4;   // 기프트카드
    }
  }

규칙:
  하나의 oneof 그룹에서 최대 하나의 필드만 설정 가능!

예시:
  Payment{payment_id:"P1", amount_cents:50000}  ✓
  Payment{payment_id:"P1", is_free_trial:true}  ✓
  Payment{payment_id:"P1", 
          amount_cents:50000, 
          is_free_trial:true}                     ✗ (에러!)

직렬화:
  Payment{payment_id:"P1", amount_cents:5000}
  → Tag(1) + "P1" + Tag(2) + Value(5000)
  
  역직렬화 시:
    which_payment_method() = AMOUNT_CENTS
    getAmountCents() = 5000
    getIsFreeeTrial() = false (다른 oneof 필드)

생성 코드:
  enum PaymentMethodCase {
    PAYMENT_METHOD_NOT_SET(0),
    AMOUNT_CENTS(2),
    IS_FREE_TRIAL(3),
    GIFT_CARD_ID(4);
  }
  
  PaymentMethodCase getPaymentMethodCase();
  clearPaymentMethod();
```

### 기본값 생략으로 인한 메시지 진화

```
메시지 크기 최적화:

버전 1:
  message Event {
    string event_id = 1;
    int64 timestamp = 2;      // 항상 필요
    int32 user_id = 3;        // 항상 필요
    string event_type = 4;    // 항상 필요
    int32 event_count = 5;    // 대부분 1이 기본값
  }

최적화된 버전 (기본값 이용):
  message Event {
    string event_id = 1;
    int64 timestamp = 2;
    int32 user_id = 3;
    string event_type = 4;
    // int32 event_count = 5;  // 생략 가능 (기본값 1)
    //
    // 또는 기본값이 0이 아닌 경우:
    //   oneof event_count_or_default {
    //     int32 custom_count = 5;
    //   }
  }

네트워크 최적화 결과:
  event_count 필드 (기본값 1): 평균 100만 메시지 중 99만 개가 1
  → 직렬화 생략으로 메시지당 2~3 바이트 절감
  → 월 수십 GB 대역폭 절감!
```

---

## 💻 실전 실험

```java
// proto3 기본값 생략 검증

import java.util.*;

public class DefaultValueTest {
  public static void main(String[] args) throws Exception {
    
    // 케이스 1: 기본값 필드는 직렬화 생략
    System.out.println("=== 케이스 1: 기본값 필드 생략 ===");
    
    // message User {
    //   string name = 1;
    //   int32 age = 2;
    //   bool is_active = 3;
    // }
    
    // User{name:"", age:0, is_active:false}
    byte[] emptyDefaults = new byte[0];  // 아무것도 전송 안 함!
    System.out.println("기본값만: " + bytesToHex(emptyDefaults) + 
                      " (0 바이트)");
    
    // User{name:"Alice", age:0, is_active:false}
    byte[] nameOnly = {
      0x0a, 0x05,                           // Tag(1), Length=5
      0x41, 0x6c, 0x69, 0x63, 0x65         // "Alice"
    };
    System.out.println("name만: " + bytesToHex(nameOnly) + 
                      " (7 바이트, age/is_active 생략)");
    
    // 케이스 2: optional 필드
    System.out.println("\n=== 케이스 2: optional로 명시적 0 표현 ===");
    
    // message Invoice {
    //   string invoice_id = 1;
    //   optional int32 discount = 2;
    // }
    
    // Invoice{invoice_id:"INV1", discount:0}
    byte[] withOptionalZero = {
      0x0a, 0x04, 0x49, 0x4e, 0x56, 0x31, // Tag(1), "INV1"
      0x10, 0x00                            // Tag(2), Value=0 (포함!)
    };
    System.out.println("optional discount=0: " + bytesToHex(withOptionalZero));
    
    // Invoice{invoice_id:"INV1"}  (discount 미설정)
    byte[] withoutOptional = {
      0x0a, 0x04, 0x49, 0x4e, 0x56, 0x31  // Tag(1), "INV1" (필드2 없음)
    };
    System.out.println("discount 미설정: " + bytesToHex(withoutOptional));
    
    System.out.println("\n→ 같은 값 0도 optional이면 구분 가능!");
    
    // 케이스 3: Wrapper 타입 (null 가능)
    System.out.println("\n=== 케이스 3: Wrapper 타입으로 null 구분 ===");
    
    // message Profile {
    //   string user_id = 1;
    //   google.protobuf.Int32Value age = 2;  // null 가능
    // }
    
    // Profile{user_id:"U1", age:null}
    byte[] ageNull = {
      0x0a, 0x02, 0x55, 0x31                // Tag(1), "U1" (필드2 없음)
    };
    System.out.println("age=null: " + bytesToHex(ageNull));
    
    // Profile{user_id:"U1", age:0}  (Wrapper Int32Value{value:0})
    byte[] ageZero = {
      0x0a, 0x02, 0x55, 0x31,             // Tag(1), "U1"
      0x12, 0x02,                          // Tag(2), Length=2
      0x08, 0x00                           // Nested: Tag(1), Value=0
    };
    System.out.println("age=0: " + bytesToHex(ageZero));
    
    System.out.println("\n→ Wrapper 타입은 null과 0을 구분하지만 용량 증가!");
  }
  
  private static String bytesToHex(byte[] bytes) {
    if (bytes.length == 0) return "(empty)";
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
=== 케이스 1: 기본값 필드 생략 ===
기본값만: (empty) (0 바이트)
name만: 0a 05 41 6c 69 63 65 (7 바이트, age/is_active 생략)

=== 케이스 2: optional로 명시적 0 표현 ===
optional discount=0: 0a 04 49 4e 56 31 10 00
discount 미설정: 0a 04 49 4e 56 31

=== 케이스 3: Wrapper 타입으로 null 구분 ===
age=null: 0a 02 55 31
age=0: 0a 02 55 31 12 02 08 00

→ Wrapper 타입은 null과 0을 구분하지만 용량 증가!
```

---

## 📊 성능/비용 비교

```
필드 타입별 메시지 크기 (1000만 메시지, 기본값 50%)

┌──────────────────┬───────────┬───────────┬────────────┐
│필드 타입         │기본값 50% │선택적(opt)│Wrapper타입 │
├──────────────────┼───────────┼───────────┼────────────┤
│int32             │4B (절반) │4B        │5B         │
│string            │5B (절반) │5B        │6B         │
│bool              │1B (절반) │1B        │2B         │
│repeated int32    │0B (절반) │N/A       │N/A        │
└──────────────────┴───────────┴───────────┴────────────┘

메시지 예시 (20필드, 기본값이 50%인 경우):
  일반 필드: 10필드 × 4B = 40B (절반 생략)
  optional: 모두 포함 = 80B
  Wrapper: 모두 포함 (null 가능) = 100B+

네트워크:
  1000만 메시지/월
  기본값 생략: 40B × 1000만 = 400MB/월
  Optional: 80B × 1000만 = 800MB/월
  → 100% 네트워크 증가!

권장:
  기본값이 1도 충분하면 기본값 생략 활용
  정말 null 필요하면 optional 또는 oneof 사용
  Wrapper는 JSON 호환성 필요시만
```

---

## ⚖️ 트레이드오프

```
✅ 기본값 생략의 장점:
├─ 작은 메시지 크기 (기본값 필드 제로)
├─ 빠른 직렬화/역직렬화
├─ proto3의 표준 동작
└─ 대부분 필드에 적합

❌ 기본값 생략의 단점:
├─ null vs 기본값 구분 불가
├─ "0원" vs "미설정" 구분 불가
├─ optional 필드는 추가 용량
└─ 스키마 문서화 필수

선택 기준:
  기본값 생략 (권장):
    ├─ 선택적이 아닌 필수 필드
    ├─ 기본값이 명확한 경우
    └─ 높은 처리량, 낮은 지연

  optional:
    ├─ 부재 여부 추적 필요
    ├─ 금액, 수량 등 0이 의미있을 때
    └─ 구 proto2 호환성

  Wrapper:
    ├─ JSON null 표현 필요
    ├─ 레거시 시스템 호환
    └─ 크기 < 성능
```

---

## 📌 핵심 정리

```
proto3 기본값:
  int/bool/string 등: 0, false, ""
  → 직렬화 생략 (메시지 크기 절감!)
  
문제:
  역직렬화 시 "미설정" vs "기본값" 구분 불가
  예: int32 price=0 (0원 할인 vs 미설정)
  
해결 방법:
  1. optional int32 price (명시적 설정 추적)
  2. oneof payment { int32 amount; bool free; }
  3. google.protobuf.Int32Value (null 가능)
  
선택 기준:
  자주 기본값? → 일반 필드 (생략)
  null 필요? → optional 또는 oneof
  JSON 호환? → Wrapper 타입
  
비용:
  기본값 생략: 메시지당 4B 절감
  optional: 4B 증가 (null 구분)
  Wrapper: 6B 증가 (크기, 복잡도)
```

---

## 🤔 생각해볼 문제

**Q1: optional int32 age = 2; 를 사용할 때, age=0으로 설정하면 직렬화되나?**
```
message User {
  optional int32 age = 2;
}

User u1 = User.newBuilder().setAge(0).build();
User u2 = User.newBuilder().build();  // age 미설정

u1.toByteArray() vs u2.toByteArray()는 같을까?
```
<details>
<summary>해설 보기</summary>

다릅니다!

u1 (age=0 명시):
  - hasAge() = true
  - 직렬화: 10 00 (Tag(2) + Value(0) 포함)

u2 (age 미설정):
  - hasAge() = false
  - 직렬화: (필드 2 없음)

optional 키워드의 핵심:
  필드 0도 명시적으로 설정된 것으로 간주
  → "0원" vs "미설정" 구분 가능!

</details>

**Q2: repeated int32 scores = 1; 필드의 기본값은?**
```
message GameResult {
  repeated int32 scores = 1;
}

GameResult g1 = GameResult.newBuilder().build();
GameResult g2 = GameResult.newBuilder()
                 .addScores(0)
                 .addScores(0)
                 .build();

g1.getScoresList().size() vs g2.getScoresList().size() ?
```
<details>
<summary>해설 보기</summary>

g1.getScoresList().size() = 0 (빈 리스트)
g2.getScoresList().size() = 2 (요소 2개)

proto3에서:
- repeated 필드의 기본값은 빈 리스트
- 빈 리스트는 직렬화 생략
- scores=[0, 0]은 명시적으로 직렬화됨

규칙:
  repeated 필드는 항상 명시적 (포함/미포함 구분)
  optional이 없어도 부재 여부는 알 수 있음

</details>

**Q3: google.protobuf.StringValue를 JSON으로 변환하면?**
```
message Profile {
  string user_id = 1;
  google.protobuf.StringValue nickname = 2;
}

Profile p1 = Profile.newBuilder()
              .setUserId("U1")
              .setNickname(StringValue.of("Alice"))
              .build();

Profile p2 = Profile.newBuilder()
              .setUserId("U1")
              .build();  // nickname 미설정

JSON 변환 결과는?
```
<details>
<summary>해설 보기</summary>

표준 protobuf JSON 변환:

p1 (nickname="Alice"):
```json
{
  "userId": "U1",
  "nickname": "Alice"
}
```

p2 (nickname 미설정):
```json
{
  "userId": "U1",
  "nickname": null
}
```

Wrapper 타입은 JSON에서:
- 값이 있으면: "Alice"
- null이면: null
- 기본값: null (생략 아님!)

이것이 Wrapper 타입이 JSON 호환성을 제공하는 이유입니다.

</details>

---

<div align="center">

**[⬅️ 이전: 필드 번호가 API의 본체](./02-field-number-contract.md)** | **[홈으로 🏠](../README.md)** | **[다음: 복합 타입 ➡️](./04-complex-types.md)**

</div>
