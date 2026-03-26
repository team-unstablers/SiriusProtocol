# MDProto Style Guide

This document defines the authoring rules for `.mdproto` files in the `msgdef/` directory.

---

## 1. File Structure

Every `.mdproto` file MUST follow the section order below. Sections that do not apply MAY be omitted.

```
---
(YAML front-matter)
---

# <Document Title>

<Intro paragraph>

---

# CONSTANTS

---

# ENUM TYPES

---

# STRUCTS

---

# MESSAGES

---

# <Appendix Section>          (optional, may repeat)

```

### Section Roles

| Section | Purpose |
|---------|---------|
| `# <Document Title>` | First H1 in the file. Names the channel or feature this file covers. |
| `# CONSTANTS` | `constset` / `optionset` block definitions |
| `# ENUM TYPES` | Protobuf `enum` definitions (if applicable) |
| `# STRUCTS` | Helper `message` types without opcodes (data structures) |
| `# MESSAGES` | Protocol messages with opcodes |
| `# <Appendix Section>` | (Optional) Supplementary explanation spanning the entire channel/feature |

### Appendix Sections (Optional)

After `# MESSAGES`, additional H1 sections MAY be added for **channel/feature-level supplementary explanation** that is not specific to a single type or message.

- Use these for content that is too broad for an individual `### IMPLEMENTATION NOTES` but too detailed for the intro paragraph.
- Section names MUST be uppercase English phrases that clearly describe the content.
- Examples: `# DEALING WITH LARGE-SIZE DATA`, `# CHANNEL ARGUMENTS`, `# SYNOPSIS`

---

## 2. Heading Level Rules

| Level | Purpose | Examples |
|-------|---------|----------|
| `#` (H1) | Document title + major section dividers | `# CONSTANTS`, `# MESSAGES` |
| `##` (H2) | Individual type/message name or subgroup | `## AuthChallenge`, `## DisplayKind` |
| `###` (H3) | Subsections within a type/message | `### DESCRIPTION`, `### IMPLEMENTATION NOTES` |

- H1 is used for the **document title**, **major section dividers** (CONSTANTS / ENUM TYPES / STRUCTS / MESSAGES), and optionally **appendix sections**.
- A `---` horizontal rule MUST be inserted between each major section.

---

## 3. Intro Paragraph

Immediately below the H1 document title, provide 1–3 sentences describing the role of this file.

```markdown
# PROJECTION AUDIO MESSAGES

Message definitions for audio projection (audio streaming and control).
Follows a session model similar to video projection, supporting various audio sources and codec configurations.
```

- Protocol flow, framing, and other meta-level descriptions SHOULD be concentrated in `general.mdproto` or a dedicated document.
- Do not scatter meta-level descriptions across individual feature files.

---

## 4. Per-Type / Per-Message Structure

```markdown
## <TypeName>

\```protobuf          (or constset / optionset)
...
\```

### DESCRIPTION          ← required

### IMPLEMENTATION NOTES  ← optional
```

### DESCRIPTION (Required)

- MUST be present for every message (in `# MESSAGES`) and every enum (in `# ENUM TYPES`).
- 1–3 sentences describing what this type is and when it is used.
- For `constset` / `optionset`: MAY be omitted if the `///` comments within the block are sufficiently self-explanatory. If additional context is needed, include a DESCRIPTION.
- For helper types in `# STRUCTS`: MAY be omitted if the `///` comments within the block are sufficient.

### IMPLEMENTATION NOTES (Optional)

- Include only when there are implementation concerns that consumers MUST be aware of.
- Examples: default value handling for specific fields, behavioral differences between server and client, security considerations, etc.

---

## 5. Code Block Rules

### 5-1. Block Separation Principle

- **Messages (with opcode):** One message per block as a rule. Closely related request/response pairs MAY be grouped under the same H2 in a single block.
- **Helper types (without opcode):** Closely related types MAY be grouped under the same H2 in a single block.
- **constset / optionset:** One type per block.

### 5-2. Indentation

- Code inside `protobuf` / `constset` / `optionset` blocks MUST use **2-space** indentation.

### 5-3. Comment Style

| Purpose | Format | Example |
|---------|--------|---------|
| Field/constant description (doc comment) | `///` | `/// Session identifier` |
| Annotation | `//` | `// @opcode: 0x8001` |
| Reference annotation | `//` | `// @constset: AudioFourCC` |
| Inline note | `//` | `uint32 flags = 16; // reserved` |

- `///` (doc comment) and `//` (general comment/annotation) serve distinct roles. Do not mix styles for the same purpose.
- `@opcode` MUST be placed on the line **immediately above** the `message` declaration.
- `@constset` and `@optionset` MUST be placed on the line **immediately above** the corresponding field.

### 5-4. No Informal Language

- Emojis, humor, and informal language MUST NOT appear in code blocks, DESCRIPTION, or IMPLEMENTATION NOTES.
- TODOs MUST follow the format `// TODO: <description>`.

---

## 6. Reference Example

Below is a complete file example that conforms to this style guide.

```markdown
---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.example"
import:
  - "msgdef/common-types.proto"
---

# EXAMPLE CHANNEL MESSAGES

Message definitions for the example channel. Used to exchange example data between the server and client.

---

# CONSTANTS

## ExampleKind

\```constset
constset ExampleKind: uint32 {
  /// Default type
  const default = 0;
  /// Extended type
  const extended = 1;
}
\```

---

# STRUCTS

## ExamplePayload

\```protobuf
/// Example data payload
message ExamplePayload {
  /// Data identifier
  uint32 id = 1;
  /// Data content
  bytes data = 2;
}
\```

---

# MESSAGES

## ExampleRequest

\```protobuf
/// Requests example data from the server.
// @opcode: 0x8001
message ExampleRequest {
  /// Request identifier
  uint64 requestId = 1;
  /// The kind of data to request
  // @constset: ExampleKind
  uint32 kind = 2;
}
\```

### DESCRIPTION

Used by the client to request example data from the server.
When the server receives this message, it MUST respond with an `ExampleResponse`.

## ExampleResponse

\```protobuf
/// Response to an example data request.
// @opcode: 0x8002
message ExampleResponse {
  /// Request identifier (MUST match the requestId from ExampleRequest)
  uint64 requestId = 1;
  /// Response data
  ExamplePayload payload = 2;
}
\```

### DESCRIPTION

Sent by the server in response to an `ExampleRequest`.
The `requestId` MUST match the identifier from the original request so the client can correlate it.

### IMPLEMENTATION NOTES

If the server does not support the requested `kind`, it SHOULD respond with an empty `payload` rather than rejecting the request.
```

---

## 7. Language

All text within `.mdproto` files MUST be written in **English**. This applies to:

- **Document title** (H1) and **intro paragraph**
- **Section headings** (H2, H3)
- **`### DESCRIPTION`** and **`### IMPLEMENTATION NOTES`** prose
- **Doc comments** (`///`) inside code blocks
- **Inline comments** (`//`) inside code blocks (excluding annotations like `@opcode`)
- **Appendix sections** and any other prose

YAML front-matter fields (package, import paths, etc.) are inherently English and require no special treatment.

### 7-1. Tone and Style

DESCRIPTION and IMPLEMENTATION NOTES sections MUST use **RFC-style keywords** with a **readable, approachable tone**:

- Use uppercase keywords **MUST**, **MUST NOT**, **SHOULD**, **SHOULD NOT**, and **MAY** to indicate requirement levels.
- Keep sentences concise but natural — avoid overly legalistic or stiff phrasing.
- Provide context around the keywords so the reader understands *why*, not just *what*. For example, prefer "When the server receives this request, it MUST respond with..." over a bare "The server MUST respond with...".
- Use third-person or passive voice, but favor whichever reads more naturally in context.

**Example (DESCRIPTION):**

```markdown
### DESCRIPTION

Used by the client to request example data from the server.
When the server receives this message, it MUST respond with an `ExampleResponse`.
```

**Example (doc comment):**

```protobuf
/// Request identifier
uint64 requestId = 1;
/// The kind of data to request
// @constset: ExampleKind
uint32 kind = 2;
```

---

## 8. Migration Guide (Korean → English)

This section defines the procedure for translating existing Korean `.mdproto` files to English.

### 8-1. Scope

All Korean text in `.mdproto` files MUST be translated to English, including:

- Intro paragraphs
- Doc comments (`///`)
- Inline comments (`//`)
- `### DESCRIPTION` and `### IMPLEMENTATION NOTES` sections
- Appendix section prose

### 8-2. Translation Rules

1. **Translate and rewrite simultaneously.** When translating DESCRIPTION and IMPLEMENTATION NOTES, convert the tone to RFC-style (MUST/SHOULD/MAY) at the same time. Do not produce a literal translation followed by a separate style pass.
2. **Doc comments** (`///`): Translate to concise English. One line per field description is preferred.
3. **Preserve structure.** Do not add, remove, or reorder fields, messages, or sections. Only the language of the text changes.
4. **Preserve annotations.** `@opcode`, `@constset`, `@optionset` annotations MUST NOT be modified.
5. **Ambiguous or unclear originals:** If the meaning of the Korean original is ambiguous or potentially incorrect, do NOT guess. Flag it to the user for clarification before proceeding.

### 8-3. Terminology

Most Korean terms in the codebase have straightforward English equivalents. The following mappings are canonical:

| Korean | English |
|--------|---------|
| 원격 제어 / 원격 데스크톱 | remote desktop |
| 세션 | session |
| 채널 | channel |
| 핸드셰이크 | handshake |
| 인증 | authentication |
| 재전송 공격 | replay attack |

For all other terms, use natural English translations consistent with the protocol domain.
