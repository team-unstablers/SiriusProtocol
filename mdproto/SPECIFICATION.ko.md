# MDProto 사양서 (초안 1.0)

MDProto(`.mdproto`)는 Markdown 문서 내에서 Protocol Buffers 메시지를 정의하기 위한 문학적 프로그래밍(Literate Programming) 형식입니다. 이를 통해 개발자는 프로토콜 문서와 정의를 단일 파일에서 유지 관리하여 일관성과 가독성을 높일 수 있습니다.

## 1. 파일 구조

`.mdproto` 파일은 다음 두 부분으로 구성됩니다.
1.  **YAML Frontmatter**: 메타데이터, 패키지 정보, 임포트 등을 정의합니다.
2.  **Markdown Body**: 문서 내용과 정의를 위한 코드 블록을 포함합니다.

```markdown
---
syntax: "proto3"
package: "sirius.example"
import:
  - "msgdef/general.proto"
options:
  java_package: "com.sirius.example"
---

# 문서 제목

설명...

```protobuf
message MyMessage { ... }
```
```

## 2. YAML Frontmatter

Frontmatter는 파일 시작 부분의 `---`로 구분됩니다.

| 필드 | 타입 | 설명 | 필수 여부 |
|-------|------|-------------|----------|
| `syntax` | String | Protobuf 구문 버전 (예: "proto3"). | 예 |
| `package` | String | 생성될 proto 파일의 패키지 이름. | 예 |
| `import` | List<String> | 임포트할 `.proto` 파일 목록. | 아니오 |
| `options` | Map<String, String> | 파일 수준 옵션 (예: `csharp_namespace`). | 아니오 |

## 3. 코드 블록

MDProto는 특정 코드 블록을 처리하여 최종 `.proto` 파일을 생성합니다.

### 3.1 `protobuf` 블록

표준 Protobuf 메시지, 열거형(enum) 또는 서비스 정의를 포함합니다.

-   **추출 (Extraction)**: 내용은 추출되어 생성 파일에 추가됩니다.
-   **어노테이션 (Annotations)**: 특수 주석(`// @`로 시작)은 메타데이터로 파싱됩니다.

### 3.2 `optionset` 블록

비트 플래그(Bit flags) 집합을 정의합니다. 표준 Protobuf `enum`과 달리, 생성된 코드에서 명시적인 비트 연산이 사용될 것을 전제로 합니다.

#### 문법 (Syntax)

```ebnf
optionset_block ::= "optionset" identifying_name [":" underlying_type] "{" { option_def } "}"
underlying_type ::= "uint8" | "uint16" | "uint32" | "uint64"
option_def      ::= "option" option_name "=" value_expression ";"
value_expression::= literal | shift_expr
shift_expr      ::= literal "<<" literal
literal         ::= decimal_lit | hex_lit | binary_lit
```

-   **기반 타입 (Underlying Type)**: 선택 사항. 생략 시 기본값은 `uint32`입니다.
-   **값 표현식 (Value Expression)**:
    -   정수 리터럴 지원: `1`, `0x01`, `0b001`.
    -   왼쪽 시프트 연산 지원: `1 << 0`, `1 << 1`.
    -   **참고**: 정의 시 비트 OR(`|`)를 사용한 복합 플래그는 지원하지 **않습니다**.

#### 예시

```optionset
optionset FilePermissions : uint8 {
    /// 읽기 권한
    option READ = 1 << 0;     // 0x01
    
    /// 쓰기 권한
    option WRITE = 1 << 1;    // 0x02
    
    /// 실행 권한
    option EXEC = 0x04;       // 혼용 사용 가능
}
```

### 3.3 `constset` 블록

특정 타입에 대한 명명된 상수 집합을 정의합니다. `enum`과 달리 이들은 "개방형(open)" 집합으로 취급되며, 이를 사용하는 필드는 집합에 정의되지 않은 값도 가질 수 있습니다.

#### 문법 (Syntax)

```ebnf
constset_block  ::= "constset" identifying_name ":" underlying_type "{" { const_def } "}"
underlying_type ::= "string" | "int32" | "int64" | "uint32" | "uint64" | "float" | "double" | "bytes"
const_def       ::= "const" const_name "=" literal ";"
```

-   **기반 타입 (Underlying Type)**: 필수. 숫자, 문자열, 바이트 타입을 사용할 수 있습니다.
-   **개방형 집합 (Open Set)**: 상수를 위한 네임스페이스 역할을 합니다. 변수가 *오직* 이 값들만 가지도록 제한하지 않습니다.

#### 예시

```constset
constset CodecID : string {
    const OPUS = "OPUS";
    const H264 = "AVCC";
}
```

#### 생성 출력 규칙

1.  **Swift**: `static let` 상수를 포함하고 `RawRepresentable`을 준수하는 `struct`로 생성됩니다.
2.  **C++**: `static constexpr` 상수를 포함하는 `struct`로 생성됩니다.

## 4. 어노테이션 (Annotations)

어노테이션은 `protobuf` 블록 내에서 메시지나 필드에 메타데이터를 첨부하기 위해 사용되는 특수 주석입니다.

### 4.1 메시지 어노테이션

#### `@opcode: <value>`
메시지에 고유한 작업 코드(Opcode)를 할당합니다.
-   **값 (Value)**: 16진수(`0x1000`) 또는 정수(`4096`).
-   **용도**: 메시지를 식별하기 위해 디스패칭 로직에서 사용됩니다.

```protobuf
// @opcode: 0x8001
message PlayRequest { ... }
```

### 4.2 필드 어노테이션

#### `@optionset: <Name>`
정수 필드가 `optionset`으로 정의된 비트마스크를 나타냄을 명시합니다.

```protobuf
message FileAttribute {
    // @optionset: FilePermissionFlags
    uint32 permissions = 1;
}
```

#### `@constset: <Name>`
필드를 `constset` 정의와 연결합니다. 이는 필드가 주로 정의된 상수 중 하나를 갖지만, 기반 타입의 임의의 값도 허용함을 힌트로 제공합니다.

```protobuf
message VideoStream {
    // @constset: CodecID
    string codecID = 1;
}
```

#### `@ensure_byte_order: <Order>`
필드에 필요한 바이트 순서를 지정합니다 (`fourCC`나 raw 데이터에 유용).
-   **순서 (Order)**: `BIG_ENDIAN` 또는 `LITTLE_ENDIAN`.

```protobuf
message Codec {
    // @ensure_byte_order: BIG_ENDIAN
    fixed32 fourCC = 1;
}
```

#### `@see: <Reference>`
다른 메시지, 필드 또는 문서 섹션으로 연결되는 링크를 제공합니다.

```protobuf
// @see: PlayRequest
message PlayResponse { ... }
```

## 5. 주석 처리

-   **슬래시 세 개 (`///`)**: 문서화 주석으로 취급됩니다. 이는 보존되어 생성된 코드에서 적절한 문서 형식(예: Javadoc, Doxygen)으로 변환됩니다.
-   **슬래시 두 개 (`//`)**: 구현 주석으로 취급됩니다. 생성기 설정에 따라 삭제되거나 보존될 수 있습니다.
