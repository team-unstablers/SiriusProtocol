# MDProto Style Guide

이 문서는 `msgdef/` 디렉토리 내 `.mdproto` 파일의 작성 규칙을 정의합니다.

---

## 1. 파일 전체 구조

모든 `.mdproto` 파일은 아래 순서를 따릅니다. 해당 없는 섹션은 생략할 수 있습니다.

```
---
(YAML front-matter)
---

# <문서 제목>

<도입 설명>

---

# CONSTANTS

---

# ENUM TYPES

---

# STRUCTS

---

# MESSAGES

---

# <부록 섹션>          (선택, 복수 가능)

```

### 각 섹션의 역할

| 섹션 | 내용 |
|------|------|
| `# <문서 제목>` | 파일의 첫 H1. 이 파일이 다루는 채널/기능 이름 |
| `# CONSTANTS` | `constset` / `optionset` 블록 정의 |
| `# ENUM TYPES` | protobuf `enum` 정의 (해당 시) |
| `# STRUCTS` | opcode가 없는 보조 `message` 타입 (데이터 구조체) |
| `# MESSAGES` | opcode가 있는 실제 프로토콜 메시지 |
| `# <부록 섹션>` | (선택) 채널/기능 전체에 걸치는 보충 설명 |

### 부록 섹션 (선택)

`# MESSAGES` 이후에, 특정 타입이나 메시지에 종속되지 않는 **채널/기능 레벨의 보충 설명**이 필요한 경우 추가 H1 섹션을 둘 수 있습니다.

- 개별 메시지의 `### IMPLEMENTATION NOTES`에 넣기엔 범위가 넓고, 도입 설명에 넣기엔 너무 긴 내용에 적합합니다.
- 섹션 이름은 내용을 명확히 나타내는 대문자 영문 구문을 사용합니다.
- 예: `# DEALING WITH LARGE-SIZE DATA`, `# CHANNEL ARGUMENTS`, `# SYNOPSIS`

---

## 2. 헤딩 레벨 규칙

| 레벨 | 용도 | 예시 |
|------|------|------|
| `#` (H1) | 문서 제목 + 대섹션 구분자 | `# CONSTANTS`, `# MESSAGES` |
| `##` (H2) | 개별 타입/메시지 이름 또는 소그룹 | `## AuthChallenge`, `## DisplayKind` |
| `###` (H3) | 부속 설명 섹션 | `### DESCRIPTION`, `### IMPLEMENTATION NOTES` |

- H1은 **문서 제목**, **대섹션 구분자**(CONSTANTS / ENUM TYPES / STRUCTS / MESSAGES), 그리고 필요 시 **부록 섹션**에 사용합니다.
- 각 대섹션 사이에는 `---` 수평선을 삽입합니다.

---

## 3. 도입 설명

H1 문서 제목 바로 아래에 1~3문장으로 이 파일의 역할을 서술합니다.

```markdown
# PROJECTION AUDIO MESSAGES

오디오 프로젝션(오디오 전송 및 제어)을 위한 메시지 정의입니다.
비디오 프로젝션과 유사한 세션 모델을 따르며, 다양한 오디오 소스와 코덱 설정을 지원합니다.
```

- 프로토콜 흐름이나 프레이밍 같은 메타 설명은 `general.mdproto` 도입부 또는 별도 문서에 집중합니다.
- 각 feature 파일에 메타 설명을 분산시키지 않습니다.

---

## 4. 개별 타입/메시지 단위 구조

```markdown
## <TypeName>

\```protobuf          (또는 constset / optionset)
...
\```

### DESCRIPTION          ← 필수

### IMPLEMENTATION NOTES  ← 선택
```

### DESCRIPTION (필수)

- 모든 메시지(`# MESSAGES` 섹션)와 enum(`# ENUM TYPES` 섹션)에 필수입니다.
- 1~3문장으로 이 타입이 무엇이고, 언제 사용되는지를 서술합니다.
- `constset` / `optionset`은 블록 내 `///` 주석이 충분히 자기 설명적이라면 DESCRIPTION 섹션을 생략할 수 있습니다. 단, 추가적인 맥락 설명이 필요하면 DESCRIPTION을 작성합니다.
- `# STRUCTS` 섹션의 보조 타입은 블록 내 `///` 주석이 충분하다면 생략할 수 있습니다.

### IMPLEMENTATION NOTES (선택)

- 구현체가 주의해야 할 사항이 있을 때만 작성합니다.
- 예: 특정 필드의 기본값 처리, 서버/클라이언트 간 동작 차이, 보안 고려사항 등.

---

## 5. 코드 블록 규칙

### 5-1. 블록 분리 원칙

- **메시지 (opcode 있는 것)**: 원칙적으로 **1 메시지 = 1 블록**. 밀접하게 관련된 request/response 쌍은 같은 H2 아래 한 블록에 묶는 것을 허용합니다.
- **보조 타입 (opcode 없는 것)**: 밀접한 관련 타입은 같은 H2 아래 한 블록에 묶을 수 있습니다.
- **constset / optionset**: **1 타입 = 1 블록**.

### 5-2. 들여쓰기

- `protobuf` / `constset` / `optionset` 블록 내부는 **스페이스 2칸** 들여쓰기를 사용합니다.

### 5-3. 주석 스타일

| 용도 | 형식 | 예시 |
|------|------|------|
| 필드/상수 설명 (doc comment) | `///` | `/// 세션 식별자` |
| 어노테이션 | `//` | `// @opcode: 0x8001` |
| 참조 어노테이션 | `//` | `// @constset: AudioFourCC` |
| 인라인 참고 | `//` | `uint32 flags = 16; // reserved` |

- `///` (doc comment)과 `//` (일반 주석/어노테이션)은 역할에 따라 구분합니다. 같은 용도에 두 스타일을 혼용하지 않습니다.
- `@opcode`는 `message` 선언 **바로 위** 줄에 배치합니다.
- `@constset`, `@optionset`은 해당 필드 **바로 위** 줄에 배치합니다.

### 5-4. 비격식 표현 금지

- 코드 블록 및 DESCRIPTION / IMPLEMENTATION NOTES 내에 이모티콘, 유머, 비격식 표현을 사용하지 않습니다.
- TODO는 `// TODO: <설명>` 형식으로 통일합니다.

---

## 6. 모범 예시

아래는 이 스타일 가이드를 따르는 파일의 전체 예시입니다.

```markdown
---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.example"
import:
  - "msgdef/common-types.proto"
---

# EXAMPLE CHANNEL MESSAGES

예시 채널의 메시지 정의입니다. 서버와 클라이언트 간 예시 데이터를 교환하는 데 사용됩니다.

---

# CONSTANTS

## ExampleKind

\```constset
constset ExampleKind: uint32 {
  /// 기본 타입
  const default = 0;
  /// 확장 타입
  const extended = 1;
}
\```

---

# STRUCTS

## ExamplePayload

\```protobuf
/// 예시 데이터 페이로드
message ExamplePayload {
  /// 데이터 식별자
  uint32 id = 1;
  /// 데이터 내용
  bytes data = 2;
}
\```

---

# MESSAGES

## ExampleRequest

\```protobuf
/// 예시 데이터를 요청합니다.
// @opcode: 0x8001
message ExampleRequest {
  /// 요청 식별자
  uint64 requestId = 1;
  /// 요청할 데이터 종류
  // @constset: ExampleKind
  uint32 kind = 2;
}
\```

### DESCRIPTION

- 클라이언트가 서버에게 예시 데이터를 요청할 때 사용하는 메시지입니다.
- 서버는 이 메시지를 수신한 후 `ExampleResponse`로 응답합니다.

## ExampleResponse

\```protobuf
/// 예시 데이터 요청에 대한 응답입니다.
// @opcode: 0x8002
message ExampleResponse {
  /// 요청 식별자 (ExampleRequest의 requestId와 일치)
  uint64 requestId = 1;
  /// 응답 데이터
  ExamplePayload payload = 2;
}
\```

### DESCRIPTION

- `ExampleRequest`에 대한 응답 메시지입니다.
- `requestId`는 원본 요청의 식별자와 일치해야 합니다.

### IMPLEMENTATION NOTES

- 서버가 요청한 `kind`를 지원하지 않는 경우, `payload`를 비워서 응답합니다.
```
