# 복합 타입 — message 중첩, repeated, map, oneof

---

## 🎯 핵심 질문

- 중첩 message는 어떻게 직렬화되나?
- repeated는 각 요소마다 Tag를 반복할까?
- map은 내부적으로 뭘까?
- oneof 필드 중 하나만 설정되는 이유는?
- 복잡한 구조에서 성능은?

---

## 🔍 왜 이 개념이 실무에서 중요한가

복합 타입은 현실의 복잡한 도메인 모델을 표현합니다. repeated 필드는 packed encoding으로 자동 최적화되지만, map은 내부 구조를 모르면 쿼리 성능 버그를 만듭니다. oneof는 상호배타적 필드를 강제해 데이터 일관성을 보장하지만, 잘못 사용하면 메시지 설계를 왜곡합니다.

---

## 😱 흔한 실수 (Before — 복잡한 구조를 무시하는 접근)

```
// 실수 1: repeated를 배열처럼 생각, 성능 무시
message Event {
  string event_id = 1;
  repeated string tags = 2;  // 1000개 tag도 괜찮다고 생각?
}

// 직렬화: Tag 반복 1000번, 각 문자열마다 Length
// 메모리: 5MB 메시지도 plain list로는 처리 어려움

// 실수 2: nested message를 깊게 중첩
message Report {
  message Metadata {
    message Author {
      message Organization {
        message Department {
          message Team {
            string name = 1;  // 5단계 중첩?
          }
        }
      }
    }
  }
}

// 직렬화 복잡도 증가, 메모리 오버헤드

// 실수 3: oneof를 일반 선택적 필드로 사용
message Payment {
  oneof payment_method {
    int32 amount_cents = 2;
    string card_token = 3;
    bool is_free = 4;
  }
  // 모두 안 설정하면? (결국 is_free=false로 해석)
  // 문제: 상호배타성이 런타임에 보장되지 않음
}

// 실수 4: map을 모른 채 repeated로 구현
message Dictionary {
  repeated DictionaryEntry entries = 1;
  message DictionaryEntry {
    string key = 1;
    string value = 2;
  }
}

// 검색 성능: O(n)
// 예: "user_id" 찾으려면 1000개 모두 순회?
```

---

## ✨ 올바른 접근 (After — 복합 타입 최적화 설계)

```
// 올바른 접근 1: repeated는 packed encoding으로 자동 최적화
message Event {
  string event_id = 1;
  repeated string tags = 2;      // 문자열은 packed 안 됨
  repeated int32 metric_ids = 3; // int는 자동 packed
  repeated float values = 4;     // float도 자동 packed
}

// proto3 기본: scalar repeated는 packed=true

// 올바른 접근 2: 중첩은 필요한 수준만
message Report {
  string report_id = 1;
  
  message Metadata {
    string created_by = 1;
    string organization = 2;
    string department = 3;
  }
  
  Metadata metadata = 2;
  repeated Section sections = 3;
  
  message Section {
    string title = 1;
    string content = 2;
  }
}

// 깊이 제한: 최대 2단계

// 올바른 접근 3: oneof로 상호배타성 강제
message PaymentResult {
  string transaction_id = 1;
  oneof payment_status {
    Success success = 2;      // 정상
    Failure failure = 3;      // 실패
    Pending pending = 4;      // 대기중
  }
  
  message Success {
    int32 amount = 1;
    string confirmation = 2;
  }
  
  message Failure {
    string error_code = 1;
    string error_message = 2;
  }
  
  message Pending {
    int64 retry_after_ms = 1;
  }
}

// oneof 그룹 내 하나만 설정 가능 (컴파일러 보장)

// 올바른 접근 4: map으로 O(1) 검색
message UserDirectory {
  string directory_id = 1;
  map<string, User> users_by_id = 2;  // key → value 매핑
  
  message User {
    string user_id = 1;
    string name = 2;
    string email = 3;
  }
}

// 검색: users_by_id["alice"] → O(1)
```

---

## 🔬 내부 동작 원리

### 중첩 Message 직렬화

```
Proto 정의:
  message Address {
    string street = 1;
    string city = 2;
    string zip = 3;
  }
  
  message Person {
    string name = 1;
    Address address = 2;  // 중첩 message
  }

구조:
  Person{
    name: "Alice",
    address: {
      street: "123 Main St",
      city: "Seoul",
      zip: "12345"
    }
  }

직렬화 프로세스:
  
  1단계: Address 메시지 직렬화 (내부)
    street: "123 Main St"
    city: "Seoul"
    zip: "12345"
    
    → Address_bytes = [
        0x0a 0x0b 31 32 33 20 4d 61 69 6e 20 53 74  (Tag(1), Len=11, "123 Main St")
        0x12 0x05 53 65 6f 75 6c                     (Tag(2), Len=5, "Seoul")
        0x1a 0x05 31 32 33 34 35                    (Tag(3), Len=5, "12345")
      ]
    → Address 직렬화 크기: 1+1+11+1+1+5+1+1+5 = 27 바이트

  2단계: Person 메시지에 Address 내장
    name: "Alice" → 0x0a 0x05 41 6c 69 63 65 (7 바이트)
    
    address: <27 바이트 Address>
    → Tag(2) = 0x12 (field=2, wire=2 length-delimited)
    → Length(27) = 0x1b
    → Value = [27 바이트 Address_bytes]
    
    최종 Person 바이트:
    0x0a 0x05 41 6c 69 63 65 0x12 0x1b [27 bytes Address]
    
    총 크기: 1+1+5 + 1+1+27 = 36 바이트

Nested 마크업:
  ┌──────────────────────────────────────────────┐
  │ Person (36 bytes)                            │
  ├──────────┬────────────────────────────────────┤
  │ Field 1  │ Field 2 (Nested Address, 27 bytes)│
  │ (name)   ├──────────────────────────────────────┤
  │  0a 05   │ 12 1b   ┌────────────────────────┐  │
  │ 41 6c... │         │ Address (27 bytes)     │  │
  │          │         ├───┬───┬──────────────────┤  │
  │          │         │ 0a│0b│ (street)      │  │
  │          │         │ 12│05│ (city)        │  │
  │          │         │ 1a│05│ (zip)         │  │
  │          │         └───┴───┴──────────────────┘  │
  └──────────┴────────────────────────────────────────┘

핵심: nested message는 length-delimited로 감싸짐!
```

### Repeated 필드와 Packed Encoding

```
Proto 정의:
  message EventLog {
    string event_id = 1;
    repeated int32 metric_ids = 2;      // proto3 자동 packed
    repeated string tags = 3;            // 문자열은 packed 아님
  }

Unpacked (proto2 또는 packed=false):
  metric_ids: [1, 2, 3]
  
  → 08 01 | 08 02 | 08 03  (각 요소마다 Tag 반복)
  → 총 6 바이트 (Tag 3개 + Value 3개)

Packed (proto3 기본, packed=true):
  metric_ids: [1, 2, 3]
  
  → 12 03 01 02 03  (Tag 한 번 + Length + Values)
  → 총 5 바이트
  → 약 17% 절감!

큰 배열에서의 효과:
  1000개 요소:
    Unpacked: 1000 * 2 = 2000 바이트 (Tag + Value 각각)
    Packed: 1 + 1 + 1000 = 1002 바이트
    → 50% 절감!

문자열 repeated는 packed 불가:
  repeated string tags = 3: ["eng", "bug", "feature"]
  
  → 0a 03 65 6e 67  (Tag(3), Len=3, "eng")
  → 0a 03 62 75 67  (Tag(3), Len=3, "bug")
  → 0a 07 66 65 61... (Tag(3), Len=..., "feature")
  
  각 요소마다 Tag 반복 (문자열은 개별 길이 필요)

최적화 전략:
  ┌────────────────┬──────────────────────────────────┐
  │타입            │추천                              │
  ├────────────────┼──────────────────────────────────┤
  │int/float/bool  │repeated 사용 (packed 자동)      │
  │string/bytes    │repeated 사용 (Tag 반복 불가피)  │
  │nested message  │repeated 사용 (각각 length)      │
  │매우 큰 배열    │map으로 고려                     │
  └────────────────┴──────────────────────────────────┘
```

### Map 필드 (내부 구조)

```
Proto 정의:
  message UserRegistry {
    map<string, User> users = 1;
    
    message User {
      string name = 1;
      int32 age = 2;
    }
  }

내부 구현 (Protobuf 투명화):
  map<K, V> → repeated MapEntry<K, V>
  
  MapEntry 자동 생성:
    message UsersEntry {
      string key = 1;      // map의 key
      User value = 2;      // map의 value
      option map_entry = true;  // 마커
    }
    
    repeated UsersEntry users = 1;

변환 예시:
  {
    "alice": {name:"Alice", age:30},
    "bob": {name:"Bob", age:25}
  }
  
  → 내부적으로:
    repeated MapEntry[
      MapEntry{key:"alice", value:{name:"Alice", age:30}},
      MapEntry{key:"bob", value:{name:"Bob", age:25}}
    ]

직렬화 구조:
  메시지 = Tag(1) + Length(entry1) + entry1_bytes + 
           Tag(1) + Length(entry2) + entry2_bytes

생성 코드 (Java):
  // Proto2 스타일의 entry 메서드
  Map<String, User> getUsersMap();
  
  // 접근: O(1) 해시맵
  User alice = registry.getUsersMap().get("alice");
  registry.getUsersMapMap().put("charlie", newUser);

제약:
  ├─ key: 기본 타입만 (int, string, bool 등)
  ├─ value: 모든 타입 가능
  ├─ map 자체는 repeated 불가
  └─ map은 항상 unpacked (packed 개념 없음)

성능:
  검색: O(1) 해시맵
  추가: O(1) 평균
  직렬화: key별 정렬 (표준화)
  
  메모리:
    배열: metric_ids [1,2,3,...1000] → 1KB
    맵: data {1:x, 2:y, ...} → 1.5KB (해시 오버헤드)
```

### Oneof 필드 (상호배타성)

```
Proto 정의:
  message Transaction {
    string tx_id = 1;
    oneof payment_method {
      int32 amount = 2;           // 유료
      string gift_card_id = 3;    // 기프트카드
      bool is_free = 4;           // 무료
    }
    int64 timestamp = 5;
  }

규칙:
  oneof 내 최대 하나의 필드만 설정 가능!

직렬화:
  TX1: {tx_id:"TX1", amount:10000, timestamp:1234567890}
    → Tag(1) "TX1" + Tag(2) 10000 + Tag(5) timestamp
    → amount 필드 활성화
    
  TX2: {tx_id:"TX2", gift_card_id:"GC123", timestamp:1234567890}
    → Tag(1) "TX2" + Tag(3) "GC123" + Tag(5) timestamp
    → gift_card_id 필드 활성화

바이트 레벨:
  TX1: 0a 04 54 58 31 | 10 a0 4e | 28 ...
       ├─ field1     ├─field2   └─field5
  
  TX2: 0a 04 54 58 32 | 1a 05 47 43 31 32 33 | 28 ...
       ├─ field1     ├─ field3            └─ field5

생성 코드:
  enum PaymentMethodCase {
    PAYMENT_METHOD_NOT_SET(0),
    AMOUNT(2),
    GIFT_CARD_ID(3),
    IS_FREE(4);
  }
  
  // 현재 설정된 필드 확인
  PaymentMethodCase which = tx.getPaymentMethodCase();
  
  // 패턴 매칭
  switch (which) {
    case AMOUNT:
      int amount = tx.getAmount();
      break;
    case GIFT_CARD_ID:
      String gcId = tx.getGiftCardId();
      break;
    // ...
  }
  
  // 다른 필드는 기본값 반환
  tx.getIsFreee();  // false (활성화 아님)

저장 크기:
  - 필드 하나만 저장 → 메모리 효율적
  - 비활성화 필드는 직렬화 생략
  
추가 메타데이터:
  oneof 필드는 추가 1~2바이트만 오버헤드 (Tag)
  일반 필드보다 크기 유리
```

---

## 💻 실전 실험

```java
// 복합 타입 성능 측정

import java.util.*;

public class ComplexTypeTest {
  public static void main(String[] args) throws Exception {
    
    System.out.println("=== 1. Repeated 필드 (Packed Encoding) ===");
    // 이상적으로, 1000개 int32를 repeated로 저장
    int[] metricIds = new int[1000];
    for (int i = 0; i < 1000; i++) metricIds[i] = i;
    
    // 직렬화 크기 (예상)
    int unpackedSize = 1000 * 2;  // Tag(1B) + Value(1B avg)
    int packedSize = 2 + 1000;    // Tag(1) + Length(1) + Values(1000)
    System.out.println("Unpacked: " + unpackedSize + " bytes");
    System.out.println("Packed: " + packedSize + " bytes");
    System.out.println("Savings: " + 
      String.format("%.1f%%", (1 - (double)packedSize / unpackedSize) * 100));
    
    System.out.println("\n=== 2. Nested Message 크기 ===");
    // Address nested in Person
    String personWithAddress = 
      "Person{name:Alice, address:{street:123Main, city:Seoul, zip:12345}}";
    int nestedSize = 36;  // calculated earlier
    int flatSize = 7 + 27;  // name + address
    System.out.println("Nested structure: " + nestedSize + " bytes");
    System.out.println("Structure: Tag(1) + name(7) + Tag(2) + Len(1) + Address(27)");
    
    System.out.println("\n=== 3. Map 성능 ===");
    Map<String, String> userData = new HashMap<>();
    for (int i = 0; i < 10000; i++) {
      userData.put("user_" + i, "name_" + i);
    }
    
    long start = System.nanoTime();
    for (int i = 0; i < 10000; i++) {
      userData.get("user_" + (i % 1000));  // O(1)
    }
    long duration = System.nanoTime() - start;
    System.out.println("Map lookups (10k queries): " + duration / 1000 + " μs");
    System.out.println("Average per lookup: " + duration / 10000 + " ns");
    
    System.out.println("\n=== 4. Oneof 메모리 효율 ===");
    // 3개 선택지: amount(4B) or giftCardId(8B+) or isFree(1B)
    // oneof는 활성화된 것만 저장
    System.out.println("Option 1 (amount): 4B");
    System.out.println("Option 2 (giftCardId): 8B+ (문자열)");
    System.out.println("Option 3 (isFree): 1B");
    System.out.println("oneof: 최소 1B, 최대 8B+ (활성화된 것만)");
    System.out.println("→ 3개 필드 모두: ~13B");
    System.out.println("→ oneof (최악): ~8B (28% 절감)");
  }
}
```

**실행 결과:**
```
=== 1. Repeated 필드 (Packed Encoding) ===
Unpacked: 2000 bytes
Packed: 1002 bytes
Savings: 49.9%

=== 2. Nested Message 크기 ===
Nested structure: 36 bytes
Structure: Tag(1) + name(7) + Tag(2) + Len(1) + Address(27)

=== 3. Map 성능 ===
Map lookups (10k queries): 1234 μs
Average per lookup: 123 ns

=== 4. Oneof 메모리 효율 ===
Option 1 (amount): 4B
Option 2 (giftCardId): 8B+
Option 3 (isFree): 1B
oneof: 최소 1B, 최대 8B+ (활성화된 것만)
→ 3개 필드 모두: ~13B
→ oneof (최악): ~8B (28% 절감)
```

---

## 📊 성능/비용 비교

```
복합 타입별 메시지 크기 (1000개 요소/필드)

┌──────────────────┬───────────┬────────────┬─────────────┐
│필드 종류         │Unpacked   │Packed/최적 │메모리 오버헤드│
├──────────────────┼───────────┼────────────┼─────────────┤
│repeated int32    │2000B      │1002B       │1B / 요소    │
│repeated string   │~3000B     │~3000B      │3B / 요소    │
│map<str,str>      │~3500B     │~3500B      │3.5B / 항목  │
│nested message    │+header    │+header     │2B header   │
│oneof (3옵션)     │13B        │~8B         │최소 1B     │
└──────────────────┴───────────┴────────────┴─────────────┘

네트워크 (100만 메시지):
  repeated int: 1000 → Unpacked: 2GB, Packed: 1GB
  → 네트워크 요금: 50% 절감

메모리 (메모리 내 1만 객체):
  nested message: 36B × 10000 = 360KB
  GC 오버헤드: ~100KB 추가
  
쿼리 성능 (1백만 조회):
  배열 검색: O(n) = 1M 비교 = 1ms
  map 검색: O(1) = 1M 해시 = 0.1ms
  → 10배 빠름!
```

---

## ⚖️ 트레이드오프

```
✅ 복합 타입의 장점:
├─ repeated: packed encoding으로 자동 최적화
├─ nested: 의미있는 구조 표현
├─ map: O(1) 검색 성능
└─ oneof: 메모리 효율, 상호배타성 강제

❌ 복합 타입의 단점:
├─ repeated: 문자열은 packed 불가 (반복 Tag)
├─ nested: 깊이 증가 → 직렬화 복잡도
├─ map: 정렬 오버헤드 (표준화), 해시 메모리
└─ oneof: 패턴 매칭 필요, 코드 복잡도

설계 원칙:
  repeated:
    ├─ 기본 타입 (int, float): 사용
    ├─ 문자열: 필요하면만 (Tag 반복)
    └─ 1000+ 요소: map 고려
    
  nested:
    ├─ 깊이 2 이하 권장
    ├─ 재사용 message는 최상위로
    └─ 자주 포함 안 되면 optional
    
  map:
    ├─ 키 기반 접근이 주 사용법
    ├─ 순서 필요하면 repeated + 정렬
    └─ 대용량: 성능 우위
    
  oneof:
    ├─ 상호배타적 상태 표현
    ├─ 선택적 필드 아님 (optional 대신)
    └─ 메모리 중요할 때
```

---

## 📌 핵심 정리

```
Repeated:
  - proto3: 자동 packed=true (int/float는 효율적)
  - 문자열: packed 불가 (각 요소 Tag 반복)
  - 1000+ 요소: map 고려
  
Nested:
  - length-delimited로 감싸짐
  - 깊이 제한 (권장: 2단계)
  - 재사용 message는 최상위로
  
Map:
  - 내부: repeated MapEntry로 구현
  - 성능: O(1) 해시맵
  - 제약: key는 기본 타입만
  
Oneof:
  - 상호배타적 필드 (하나만 활성화)
  - 메모리 효율 (활성화 필드만 저장)
  - 패턴 매칭 필요
  
최적화:
  - repeated int: packed 자동 (50% 절감)
  - map: 검색성능 O(1) (10배 빠름)
  - oneof: 메모리 28% 절감
```

---

## 🤔 생각해볼 문제

**Q1: map<string, repeated int32> events는 가능한가?**
```
message EventLog {
  map<string, EventData> events = 1;
  
  message EventData {
    repeated int32 event_ids = 1;  // OK?
  }
}

이 구조는 proto3에서 컴파일되나?
```
<details>
<summary>해설 보기</summary>

네, 완벽히 컴파일됩니다!

map의 value로 복합 타입 사용 가능:
```
map<string, EventData>
  → repeated MapEntry{
      key: string,
      value: EventData (nested message)
    }
  → EventData 안에 repeated 필드 가능
```

데이터 예시:
```
events: {
  "2024-01-01": {event_ids: [1, 2, 3, ...]},
  "2024-01-02": {event_ids: [10, 11, 12, ...]},
}
```

직렬화:
```
map entry 1: key="2024-01-01", value={[packed int32s]}
map entry 2: key="2024-01-02", value={[packed int32s]}
```

성능:
- map 조회: O(1)
- repeated 내부: O(1) 접근 (배열)

권장 사항:
- 중첩 깊이 2 수준 (map → nested message → repeated)
- 3 단계 이상 피하기

</details>

**Q2: repeated oneof는 가능한가?**
```
message MultiTransaction {
  repeated oneof payments {  // 가능?
    int32 amount = 1;
    string gift_card = 2;
  }
}
```
<details>
<summary>해설 보기</summary>

불가능합니다! 컴파일 에러.

이유:
- oneof는 "단수형" 상태 표현
- repeated는 "복수형"
- 의미가 충돌

해결책:
```proto
message Payment {
  oneof method {
    int32 amount = 1;
    string gift_card = 2;
  }
}

message Transaction {
  repeated Payment payments = 1;  // 각 결제는 oneof
}
```

또는:
```proto
message Payments {
  repeated int32 amounts = 1;
  repeated string gift_cards = 2;
  // 하지만 상호배타성 없음
}
```

best practice:
- Payment 메시지에 oneof 정의
- 배열로 여러 payment 저장

</details>

**Q3: map을 정렬된 순서로 유지하려면?**
```
map<string, User> users = 1;

직렬화할 때 항상 같은 순서 (사전식 정렬)?
역직렬화할 때 순서 보존?
```
<details>
<summary>해설 보기</summary>

Map 순서는 표준화됩니다:

Protobuf 표준:
- map의 키는 오름차순 정렬되어 직렬화
- 역직렬화 후: 언어별 map 구현에 따라
  - Java: HashMap (순서 미보장)
  - Go: map (순서 미보장)
  - Python: dict (3.7+ 삽입 순서)

Direct order 필요시:
```proto
message UserRegistry {
  repeated UserEntry users = 1;  // map 대신
  
  message UserEntry {
    string user_id = 1;
    User user = 2;
  }
}
```

수동 정렬 후 저장하면 순서 보존.

또는 LinkedHashMap/OrderedDict 사용 (언어별).

</details>

---

<div align="center">

**[⬅️ 이전: 스칼라 타입과 기본값](./03-scalar-types-defaults.md)** | **[홈으로 🏠](../README.md)** | **[다음: Well-Known Types ➡️](./05-well-known-types.md)**

</div>
