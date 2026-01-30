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
underlying_type ::= "string" | "int32" | "int64" | "uint32" | "uint64" | "float" | "double" | "bytes"
const_def       ::= "const" const_name "=" literal ";"
```

-   **Underlying Type**: Required. Can be numeric, string, or bytes.
-   **Open Set**: Acts as a namespace for constants. It does not restrict the variable to *only* these values.

#### Example

```constset
constset CodecID : string {
    const OPUS = "OPUS";
    const H264 = "AVCC";
}
```

#### Generated Output Rules

1.  **Swift**: Generated as a `struct` conforming to `RawRepresentable` with `static let` constants.
2.  **C++**: Generated as a `struct` containing `static constexpr` constants.

## 4. Annotations

Annotations are special comments used within `protobuf` blocks to attach metadata to messages or fields.

### 4.1 Message Annotations

#### `@opcode: <value>`
Assigns a unique operation code (Opcode) to a message.
-   **Value**: Hexadecimal (`0x1000`) or Integer (`4096`).
-   **Usage**: Used by the dispatching logic to identify messages.

```protobuf
// @opcode: 0x8001
message PlayRequest { ... }
```

### 4.2 Field Annotations

#### `@optionset: <Name>`
Indicates that an integer field represents a bitmask defined by an `optionset`.

```protobuf
message FileAttribute {
    // @optionset: FilePermissionFlags
    uint32 permissions = 1;
}
```

#### `@constset: <Name>`
Associates a field with a `constset` definition. This hints that the field typically holds one of the defined constants but allows arbitrary values of the underlying type.

```protobuf
message VideoStream {
    // @constset: CodecID
    string codecID = 1;
}
```

#### `@ensure_byte_order: <Order>`
Specifies the required byte order for a field (useful for `fourCC` or raw data).
-   **Order**: `BIG_ENDIAN` or `LITTLE_ENDIAN`.

```protobuf
message Codec {
    // @ensure_byte_order: BIG_ENDIAN
    fixed32 fourCC = 1;
}
```

#### `@see: <Reference>`
Links to another message, field, or documentation section.

```protobuf
// @see: PlayRequest
message PlayResponse { ... }
```

## 5. Comment Processing

-   **Triple Slash (`///`)**: Treated as documentation comments. These are preserved and converted to the appropriate documentation format in the generated code (e.g., Javadoc, Doxygen).
-   **Double Slash (`//`)**: Treated as implementation comments. These may be discarded or preserved depending on the generator settings.
