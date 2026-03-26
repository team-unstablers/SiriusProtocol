# MDProto 사양서 (초안 1.0)

MDProto(`.mdproto.md`)는 Markdown 문서 내에서 Protocol Buffers 메시지를 정의하기 위한 문학적 프로그래밍(Literate Programming) 형식입니다. 이를 통해 개발자는 프로토콜 문서와 정의를 단일 파일에서 유지 관리하여 일관성과 가독성을 높일 수 있습니다.

## 1. 파일 구조

`.mdproto.md` 파일은 다음 두 부분으로 구성됩니다.
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
underlying_type ::= "string" | "int32" | "int64" | "uint32" | "uint64" | "float" | "double" | "bytes" | "UUID"
const_def       ::= "const" const_name "=" literal ";"
literal         ::= decimal_lit | hex_lit | binary_lit | uuid_lit | string_literal
uuid_lit        ::= "uuid" string_literal
```

-   **기반 타입 (Underlying Type)**: 필수. 숫자, 문자열, 바이트, **UUID** 타입을 사용할 수 있습니다.
-   **개방형 집합 (Open Set)**: 상수를 위한 네임스페이스 역할을 합니다. 변수가 *오직* 이 값들만 가지도록 제한하지 않습니다.
-   **UUID 지원**: `UUID` 타입을 사용할 경우, 상수는 `uuid"..."` 리터럴 형식을 사용해야 합니다.

#### 예시

```constset
constset CodecID : string {
    const OPUS = "OPUS";
    const H264 = "AVCC";
}

constset FeatureID : UUID {
    const HIDIO = uuid"405E6E75-8B64-4E8F-91FF-E9E5A2C6BC77";
}
```

#### 생성 출력 규칙

1.  **Swift**: `static let` 상수를 포함하고 `RawRepresentable`을 준수하는 `struct`로 생성됩니다.
2.  **C++**: `static constexpr` 상수를 포함하는 `struct`로 생성됩니다.

## 4. 어노테이션 (Annotations)

어노테이션은 `protobuf` 블록 내에서 메시지나 필드에 임의의 메타데이터를 첨부하기 위해 사용되는 특수 주석입니다. MDProto는 어노테이션의 **문법**만을 정의하며, 구체적인 어노테이션 키의 의미는 각 프로젝트 또는 프로토콜이 정의합니다.

### 4.1 문법 (Syntax)

어노테이션은 `// @`로 시작하는 한 줄 주석이며, 키와 콜론으로 구분된 선택적 값으로 구성됩니다.

```ebnf
annotation      ::= "//" ws "@" key [ ":" ws value ]
key             ::= identifier
value           ::= <줄 끝까지의 임의 문자열>
```

-   **배치**: 어노테이션은 바로 **다음에 오는** 메시지 또는 필드 선언에 적용됩니다.
-   **복수 어노테이션**: 하나의 선언 앞에 여러 어노테이션을 연속으로 배치할 수 있습니다.
-   **키**: 임의의 식별자. MDProto는 유효한 키를 제한하지 않습니다.
-   **값**: 임의의 문자열. 숫자, 이름, 참조 등으로의 해석은 이를 소비하는 도구 또는 프로토콜에 맡깁니다.

### 4.2 예시

```protobuf
// @opcode: 0x8001
message PlayRequest {
    // @constset: CodecID
    string codec_id = 1;

    // @ensure_byte_order: BIG_ENDIAN
    fixed32 fourCC = 2;
}

// @see: PlayRequest
message PlayResponse { ... }
```

### 4.3 보존 (Preservation)

어노테이션은 생성된 `.proto` 출력에 주석으로 보존됩니다. 다운스트림 도구(예: `protoc` 플러그인, 코드 생성기)는 이 주석을 파싱하여 후처리에 필요한 메타데이터를 추출할 수 있습니다.

## 5. 주석 처리

-   **슬래시 세 개 (`///`)**: 문서화 주석으로 취급됩니다. 이는 보존되어 생성된 코드에서 적절한 문서 형식(예: Javadoc, Doxygen)으로 변환됩니다.
-   **슬래시 두 개 (`//`)**: 구현 주석으로 취급됩니다. 생성기 설정에 따라 삭제되거나 보존될 수 있습니다.
