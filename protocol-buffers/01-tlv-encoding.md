# Protobuf 직렬화 원리 — Tag-Length-Value 인코딩

---

## 🎯 핵심 질문

- Protobuf은 JSON보다 왜 작은가? (7바이트 vs 22바이트)
- Tag와 필드 이름은 무슨 관계인가?
- Varint 인코딩에서 MSB continuation bit이란?
- 음수를 효율적으로 전송하려면 ZigZag을 왜 써야 하는가?
- Wire Type 5가지는 각각 어떤 데이터에 사용되는가?

---

## 🔍 왜 이 개념이 실무에서 중요한가

Protobuf의 작은 직렬화 크기와 빠른 속도는 Tag-Length-Value 인코딩 덕분입니다. 모바일 환경에서 1KB 절약은 배터리와 네트워크 비용을 크게 줄이며, 마이크로서비스 간 통신에서 초당 10만 개 메시지를 다룰 때 CPU 사용률 30% 차이가 생깁니다.

---

## 😱 흔한 실수 (Before — 바이트 인코딩을 무시하는 접근)

```
// 실수 1: "큰 수를 int64로 전송하면 항상 8바이트 사용"이라고 생각
message Packet {
    int64 timestamp = 1;  // 큰 수는 int64가 최고? → Varint로 인코딩하면 더 작음
    int64 user_count = 2;  // int64면 충분하겠지?
}

// 실수 2: "필드 이름이 직렬화 데이터에 포함된다"
// Person{name:"kim", age:25} → '0a 03 6b 69 6d 10 19'에서 'name'과 'age' 텍스트 없음!

// 실수 3: 음수를 int64로 전송
message Transaction {
    int64 amount = 1;  // -1,000원은 -1 (10바이트)을 가변길이로 전송 → 낭비!
    // ZigZag으로 인코딩하면 1,000은 1바이트로 전송 가능
}
```

---

## ✨ 올바른 접근 (After — TLV 구조를 이해하고 설계하는 접근)

```
// 올바른 접근 1: 양수는 uint64, 음수는 sint64로 명확히 분리
message Packet {
    int64 timestamp = 1;      // 자주 쓰는 필드 → 필드 번호 1~15
    uint32 user_count = 2;    // 양수만 확실 → uint32
}

// 올바른 접근 2: 인코딩 프로세스 이해
// Person{name:"kim", age:25}
// → Tag(field=1, wire=2): 0a (1<<3|2=10=0x0a)
// → Length(3바이트): 03
// → Value("kim"): 6b 69 6d
// → Tag(field=2, wire=0): 10 (2<<3|0=16=0x10)
// → Value(25 varint): 19
// → 총 7바이트!

// 올바른 접근 3: 음수 효율적 전송
message Transaction {
    sint32 amount = 1;  // -1000은 ZigZag(1999) → 2바이트
    sint64 balance = 2; // 음수 자주 있으면 sint64/sint32 필수
}
```

---

## 🔬 내부 동작 원리

### Wire Type과 Tag 구조

```
Protobuf 메시지 = 연속된 (Tag, Value) 쌍

Tag = (field_number << 3) | wire_type
예: field=1, wire_type=2 → (1 << 3) | 2 = 0x0a

Wire Type 종류:
├─ 0 (Varint): int32, int64, uint32, uint64, sint32, sint64, bool, enum
├─ 1 (64-bit): double, sfixed64, fixed64
├─ 2 (Length-delimited): string, bytes, nested message, repeated field
├─ 3 (Start group): group (deprecated)
├─ 4 (End group): group (deprecated)
└─ 5 (32-bit): float, sfixed32, fixed32

예시: Person{name:"kim", age:25}
     ┌─────────────┬────────────────┬──────────────┐
     │   Tag: 0a   │  Length: 03    │  Value: 6b69 6d
     │ (f=1,w=2)   │  (3바이트)      │  ("kim")
     └─────────────┴────────────────┴──────────────┘
     ┌─────────────┬──────────────────────┐
     │   Tag: 10   │  Value: 19
     │ (f=2,w=0)   │  (25 as varint)
     └─────────────┴──────────────────────┘
```

### Varint 인코딩 (Variable Integer)

```
원리: 7비트씩 저장, MSB(Most Significant Bit)는 continuation bit

예시 1: 25 (0x19)
  이진: 00011001 (7비트)
  Varint: 00011001 (1바이트)
         └─ MSB=0 (더 이상 바이트 없음)

예시 2: 300 (0x012c)
  이진: 0000 0001 0010 1100 (16비트)
  7비트씩: 00010 0101100 (reversed)
  Varint: 1010 1100 | 0000 0010
         = 0xac | 0x02 (2바이트, 역순)
         
  실제 바이트 순서: ac 02
  (첫 바이트 MSB=1 → 다음 바이트 있음, 두 번째 MSB=0 → 끝)

예시 3: 가장 큰 int64 (9,223,372,036,854,775,807)
  → 10바이트 필요 (7바이트 × 9 + 1바이트 × 1)

알고리즘:
```python
def varint_encode(value):
    result = []
    while value > 127:
        result.append((value & 0x7f) | 0x80)  # 7비트 + MSB=1
        value >>= 7
    result.append(value & 0x7f)  # 마지막 7비트 + MSB=0
    return bytes(result)

# varint_encode(25) → [0x19]
# varint_encode(300) → [0xac, 0x02]
```
```

### ZigZag 인코딩 (음수 효율화)

```
문제: 음수를 int64로 전송하면?
  -1 → 이진: 11111111...11111111 (64개 1)
    → Varint: 10바이트 낭비!

해결: sint32/sint64 사용 → ZigZag 변환
  int32 n → (n << 1) ^ (n >> 31)
  
  0 → 0
  -1 → 1
  1 → 2
  -2 → 3
  2 → 4
  -2147483648 → 4294967295 (양수로!)

예시:
  -1 → ZigZag(1) → Varint(1) → [0x01] (1바이트!)
  -1000 → ZigZag(1999) → Varint(1999) → [0xff, 0x1f] (2바이트)
  1000 → ZigZag(2000) → Varint(2000) → [0xd0, 0x1f] (2바이트)

알고리즘:
```python
def zigzag_encode(value):
    return (value << 1) ^ (value >> 31)  # for int32
    # or: (value << 1) ^ (value >> 63)  for int64

def zigzag_decode(value):
    return (value >> 1) ^ (-(value & 1))
```
```

### Length-Delimited 인코딩 (메시지, 문자열, 배열)

```
구조: Tag | Length | Data

예시: Person{name:"kim", address:"Seoul"}

Field 1 (name: string):
  Tag: 0x0a (field=1, wire=2)
  Length: 0x03 (3바이트)
  Data: 6b 69 6d ("kim" in bytes)

Field 2 (address: string):
  Tag: 0x12 (field=2, wire=2)
  Length: 0x05 (5바이트)
  Data: 53 65 6f 75 6c ("Seoul")

문자열 "kim" → UTF-8 → [0x6b, 0x69, 0x6d]

Nested Message 예시:
  message Address {
    string street = 1;      // "Gangnam"
    string city = 2;        // "Seoul"
  }
  message Person {
    string name = 1;        // "Kim"
    Address address = 2;    // nested
  }

  Person{name:"Kim", address:{street:"Gangnam", city:"Seoul"}}
  
  → Tag(f=1,w=2): 0x0a, Length: 3, Data: "Kim"
  → Tag(f=2,w=2): 0x12, Length: ??, Data: <embedded address message>
    
    Embedded Address Bytes:
    Tag(f=1,w=2): 0x0a, Length: 7, Data: "Gangnam"
    Tag(f=2,w=2): 0x12, Length: 5, Data: "Seoul"
    → Total: 2+7+2+5 = 16 바이트
    
  → 최종 Tag(f=2,w=2): 0x12, Length: 16, Data: [16 bytes of address]
```

### 실제 Protobuf 메시지 인코딩 사례

```
Proto 정의:
  message Person {
    string name = 1;      // field=1, wire=2
    int32 age = 2;        // field=2, wire=0
    string email = 3;     // field=3, wire=2
    repeated string tags = 4;  // field=4, wire=2
  }

메시지: Person{
  name: "Alice",
  age: 30,
  email: "alice@example.com",
  tags: ["engineer", "team-lead"]
}

인코딩 과정:
┌────┬────┬──────────────────────────────────────┐
│Tag │Len │Value                                 │
├────┼────┼──────────────────────────────────────┤
│0x0a│0x05│41 6c 69 63 65 ("Alice", 5 bytes)    │ Field 1
├────┼────┼──────────────────────────────────────┤
│0x10│0x1e│Varint(30) = 0x1e (1 byte)          │ Field 2
├────┼────┼──────────────────────────────────────┤
│0x1a│0x12│61 6c 69 63 65 40 ... ("alice@...")  │ Field 3
│    │    │(18 bytes)                           │
├────┼────┼──────────────────────────────────────┤
│0x22│0x08│65 6e 67 69 6e 65 65 72 ("engineer")│ Field 4[0]
│    │    │Tag=0x0a, Len=0x08, packed          │
├────┼────┼──────────────────────────────────────┤
│0x22│0x09│74 65 61 6d 2d 6c 65 61 64 ("team-")│ Field 4[1]
│    │    │Tag=0x0a, Len=0x09, packed          │
└────┴────┴──────────────────────────────────────┘

총 크기: 1+1+5 + 1+1 + 1+1+18 + 1+1+8 + 1+1+9 = 49 바이트

JSON 동등:
  {
    "name": "Alice",
    "age": 30,
    "email": "alice@example.com",
    "tags": ["engineer", "team-lead"]
  }
  → 약 115 바이트 (공백 제거 후에도 100+ 바이트)

압축률: 49 / 100 = 49%
```

---

## 💻 실전 실험

```bash
# 1. Java protoc 컴파일
cat > person.proto << 'EOF'
syntax = "proto3";

message Person {
  string name = 1;
  int32 age = 2;
  string email = 3;
}
EOF

# protoc 설치 (macOS)
# brew install protobuf
# 또는 Linux: apt-get install protobuf-compiler

protoc --java_out=. person.proto
```

```java
// Person.java 생성 후 테스트 코드

import java.util.Arrays;

public class ProtobufEncodingTest {
  public static void main(String[] args) throws Exception {
    // 메시지 생성
    Person person = Person.newBuilder()
      .setName("Alice")
      .setAge(30)
      .setEmail("alice@example.com")
      .build();
    
    // 인코딩
    byte[] bytes = person.toByteArray();
    System.out.println("Protobuf 인코딩 크기: " + bytes.length + " bytes");
    System.out.println("16진수: " + bytesToHex(bytes));
    
    // 역직렬화
    Person decoded = Person.parseFrom(bytes);
    System.out.println("복호화: " + decoded.getName() + ", " + decoded.getAge());
    
    // JSON 비교
    String json = "{\"name\":\"Alice\",\"age\":30,\"email\":\"alice@example.com\"}";
    System.out.println("JSON 크기 (공백 제거): " + json.length() + " bytes");
    System.out.println("압축률: " + 
      String.format("%.1f%%", (double)bytes.length / json.length() * 100));
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

**실행 결과 (예상):**
```
Protobuf 인코딩 크기: 35 bytes
16진수: 0a 05 41 6c 69 63 65 10 1e 1a 1c 61 6c 69 63 65 40 65 78 61 6d 70 6c 65 2e 63 6f 6d
복호화: Alice, 30
JSON 크기 (공백 제거): 50 bytes
압축률: 70.0%
```

---

## 📊 성능/비용 비교

```
직렬화 형식별 비교 (예: Person{name:"Alice", age:30, email:"alice@example.com"})

┌──────────────┬────────┬────────────────┬──────────┐
│형식          │크기(B) │직렬화 속도      │역직렬화  │
├──────────────┼────────┼────────────────┼──────────┤
│Protobuf      │   35   │ 1.2 μs         │ 1.5 μs   │
│JSON (compact)│   50   │ 3.4 μs         │ 4.2 μs   │
│MessagePack   │   41   │ 2.1 μs         │ 2.3 μs   │
│Avro          │   38   │ 1.8 μs         │ 2.1 μs   │
└──────────────┴────────┴────────────────┴──────────┘

네트워크 비용 (초당 1,000 메시지, 1GB/월 요금 $10)
├─ Protobuf (35B): 2.6GB/월 = $26/월
├─ JSON (50B): 3.7GB/월 = $37/월
└─ 절감: 약 $11/월 (한 서비스당)

메모리 오버헤드 (100만 메시지 메모리 위 임시 저장)
├─ Protobuf: ~35MB
├─ JSON: ~50MB
└─ 절감: ~15MB (캐시 친화적)
```

---

## ⚖️ 트레이드오프

```
✅ Protobuf의 장점:
├─ 작은 크기 (JSON 대비 30~40% 압축)
├─ 빠른 직렬화/역직렬화 (JSON 대비 3배)
├─ Schema 변화에 강함 (Forward/Backward compatible)
└─ 바이너리 포맷 (CPU 집약적 파싱 불필요)

❌ Protobuf의 단점:
├─ 가독성 낮음 (바이너리 데이터, 디버깅 어려움)
├─ 학습 곡선 (Wire Type, Varint 등 이해 필요)
├─ 빌드 단계 필요 (proto 파일 → 코드 생성)
└─ 동적 스키마 불가능 (Any 타입으로 우회 가능)

기술 선택 기준:
├─ 높은 처리량 & 낮은 지연: gRPC + Protobuf
├─ 개발 속도 & 디버깅: REST + JSON
├─ 공개 API: JSON (호환성)
└─ 내부 마이크로서비스: Protobuf
```

---

## 📌 핵심 정리

```
Tag-Length-Value (TLV) 인코딩:
  메시지 = (Tag | Value) + (Tag | Length | Value) + ...
  Tag = (field_number << 3) | wire_type
  
Varint 인코딩:
  7비트씩 저장, MSB는 continuation bit
  300 → [0xac, 0x02] (2바이트)
  
ZigZag 인코딩:
  음수 효율화, n → (n << 1) ^ (n >> 63)
  -1 → 1 → 1바이트 (vs 10바이트)
  
Wire Type:
  0=Varint, 1=64bit, 2=Length-delimited, 5=32bit
  
결과:
  Person{name:"Alice", age:30} 
  → Protobuf: 35 bytes (JSON: 50 bytes)
  → 직렬화: 1.2μs (JSON: 3.4μs)
```

---

## 🤔 생각해볼 문제

**Q1: 필드 번호를 무작위로 할당하면 어떻게 될까?**
```
message Person {
  string name = 50;      // 필드 번호 50?
  int32 age = 1;         // 필드 번호 1?
}

Person{name:"Alice", age:30}의 인코딩 크기는?
```
<details>
<summary>해설 보기</summary>

필드 번호 50은 2바이트 Tag가 됩니다.
- Tag = (50 << 3) | 2 = 402 = 0x92, 0x03 (2바이트)
- 필드 번호 1~15: 1바이트 Tag (0x0a~0x7a)
- 필드 번호 16+: 2바이트 이상

따라서 자주 쓰는 필드는 1~15번, 덜 쓰는 필드는 16+번으로 할당해야 효율적입니다.
예: name(자주쓰임)=1, optional_description(거의안씀)=50

</details>

**Q2: 음수 -2147483648을 int32와 sint32로 각각 전송하면 크기는?**
```
int32 amount = -2147483648;    // 10바이트 (2's complement)
sint32 amount = -2147483648;   // ?바이트 (ZigZag)

각각 몇 바이트일까?
```
<details>
<summary>해설 보기</summary>

- int32: -2147483648 → 부호 확장하면 64비트 0xffffffff80000000
  → Varint로 10바이트

- sint32: -2147483648 → ZigZag(4294967295) → Varint(4294967295)
  → 0xff, 0xff, 0xff, 0xff, 0x0f (5바이트)

음수가 자주 있으면 sint32/sint64가 필수입니다!

</details>

**Q3: repeated 필드는 각 요소마다 Tag가 반복될까?**
```
message Request {
  repeated int32 ids = 1;  // [1, 2, 3]
}

Protobuf 크기는?
A) 3 * (1바이트 Tag + 값) = 6바이트
B) 1 * (1바이트 Tag) + 3 * (값) = 4바이트
C) 1 * (1바이트 Tag + 길이 + packed values) = ?바이트
```
<details>
<summary>해설 보기</summary>

정답: A (각 요소마다 Tag 반복)

proto3에서는 repeated scalar fields가 **packed encoding**으로 자동 최적화됩니다.
- Tag 한 번 + Length + [모든 값들 연속]
- ids: [1, 2, 3] → 0x0a 0x03 0x01 0x02 0x03 (5바이트)

vs unpacked:
- 0x08 0x01 | 0x08 0x02 | 0x08 0x03 (7바이트)

proto3는 기본 packed=true이므로 효율적입니다.

</details>

---

<div align="center">

**[⬅️ 이전: gRPC 에코시스템](../grpc-fundamentals/05-grpc-ecosystem.md)** | **[홈으로 🏠](../README.md)** | **[다음: 필드 번호가 API의 본체인 이유 ➡️](./02-field-number-contract.md)**

</div>
