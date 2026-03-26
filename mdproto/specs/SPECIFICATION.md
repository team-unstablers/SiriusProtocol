# MDProto Specification (Draft 1.0)

MDProto(`.mdproto`) is a literate programming format for defining Protocol Buffers messages within Markdown documentation. It allows developers to maintain protocol documentation and definitions in a single file, ensuring consistency and readability.

## 1. File Structure

An `.mdproto` file consists of two parts:
1.  **YAML Frontmatter**: Defines metadata, package information, and imports.
2.  **Markdown Body**: Contains documentation and code blocks for definitions.

```markdown
---
syntax: "proto3"
package: "sirius.example"
import:
  - "msgdef/general.proto"
options:
  java_package: "com.sirius.example"
---

# Documentation Title

Description...

```protobuf
message MyMessage { ... }
```
```

## 2. YAML Frontmatter

The frontmatter is delimited by `---` at the beginning of the file.

| Field | Type | Description | Required |
|-------|------|-------------|----------|
| `syntax` | String | Protobuf syntax version (e.g., "proto3"). | Yes |
| `package` | String | The package name for the generated proto file. | Yes |
| `import` | List<String> | List of `.proto` files to import. | No |
| `options` | Map<String, String> | File-level options (e.g., `csharp_namespace`). | No |

## 3. Code Blocks

MDProto processes specific code blocks to generate the final `.proto` file.

### 3.1 `protobuf` Block

Contains standard Protobuf message, enum, or service definitions.

-   **Extraction**: Contents are extracted and appended to the generated file.
-   **Annotations**: Special comments (starting with `// @`) are parsed as metadata.

### 3.2 `optionset` Block

Defines a set of bit flags. Unlike standard Protobuf `enum`, explicit bitwise operations are expected in the generated code.

#### Syntax

```ebnf
optionset_block ::= "optionset" identifying_name [":" underlying_type] "{" { option_def } "}"
underlying_type ::= "uint8" | "uint16" | "uint32" | "uint64"
option_def      ::= "option" option_name "=" value_expression ";"
value_expression::= literal | shift_expr
shift_expr      ::= literal "<<" literal
literal         ::= decimal_lit | hex_lit | binary_lit
```

-   **Underlying Type**: Optional. Defaults to `uint32` if omitted.
-   **Value Expression**:
    -   Supports Integer Literals: `1`, `0x01`, `0b001`.
    -   Supports Left Shift: `1 << 0`, `1 << 1`.
    -   **Note**: Composite flags using bitwise OR (`|`) are **NOT** supported in the definition.

#### Example

```optionset
optionset FilePermissions : uint8 {
    /// Read permission
    option READ = 1 << 0;     // 0x01
    
    /// Write permission
    option WRITE = 1 << 1;    // 0x02
    
    /// Execute permission
    option EXEC = 0x04;       // Mixed usage is allowed
}
```

### 3.3 `constset` Block

Defines a set of named constants for a specific type. Unlike `enum`, these are treated as "open" sets, meaning fields using them can hold values not explicitly defined in the set.

#### Syntax

```ebnf
constset_block  ::= "constset" identifying_name ":" underlying_type "{" { const_def } "}"
underlying_type ::= "string" | "int32" | "int64" | "uint32" | "uint64" | "float" | "double" | "bytes" | "UUID"
const_def       ::= "const" const_name "=" literal ";"
literal         ::= decimal_lit | hex_lit | binary_lit | uuid_lit | string_literal
uuid_lit        ::= "uuid" string_literal
```

-   **Underlying Type**: Required. Can be numeric, string, bytes, or **UUID**.
-   **Open Set**: Acts as a namespace for constants. It does not restrict the variable to *only* these values.
-   **UUID Support**: When using the `UUID` type, constants must use the `uuid"..."` literal format.

#### Example

```constset
constset CodecID : string {
    const OPUS = "OPUS";
    const H264 = "AVCC";
}

constset FeatureID : UUID {
    const HIDIO = uuid"405E6E75-8B64-4E8F-91FF-E9E5A2C6BC77";
}
```

#### Generated Output Rules

1.  **Swift**: Generated as a `struct` conforming to `RawRepresentable` with `static let` constants.
2.  **C++**: Generated as a `struct` containing `static constexpr` constants.

## 4. Annotations

Annotations are special comments used within `protobuf` blocks to attach arbitrary metadata to messages or fields. MDProto defines only the **syntax** for annotations; the semantics of specific annotation keys are left to each project or protocol to define.

### 4.1 Syntax

An annotation is a single-line comment beginning with `// @`, followed by a key and an optional value separated by a colon.

```ebnf
annotation      ::= "//" ws "@" key [ ":" ws value ]
key             ::= identifier
value           ::= <any characters until end of line>
```

-   **Placement**: An annotation applies to the **next** message or field declaration that follows it.
-   **Multiple annotations**: Multiple annotations can be stacked on consecutive lines before a single declaration.
-   **Key**: An arbitrary identifier. MDProto does not restrict which keys are valid.
-   **Value**: An arbitrary string. Interpretation (e.g., as a number, name, or reference) is up to the consuming tool or protocol.

### 4.2 Examples

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

### 4.3 Preservation

Annotations are preserved in the generated `.proto` output as comments. Downstream tools (e.g., `protoc` plugins, code generators) may parse these comments to extract metadata for further processing.

## 5. Comment Processing

-   **Triple Slash (`///`)**: Treated as documentation comments. These are preserved and converted to the appropriate documentation format in the generated code (e.g., Javadoc, Doxygen).
-   **Double Slash (`//`)**: Treated as implementation comments. These may be discarded or preserved depending on the generator settings.
