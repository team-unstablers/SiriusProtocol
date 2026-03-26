---
syntax: "proto3"
package: "sirius.msgdef"
---

# COMMON TYPES

Common data type definitions shared across the Sirius protocol.

---

# STRUCTS

## SRUUID

```protobuf
/// UUID container
message SRUUID {
  /// 16-byte UUID value
  bytes value = 1;
}
```

### DESCRIPTION

- A container structure for serializing UUIDs in the Sirius protocol.
- The `value` field MUST contain a UUID value that is exactly 16 bytes in length.

### IMPLEMENTATION NOTES

- All Sirius protocol implementations MUST validate that `value` is exactly 16 bytes.
- UUID values that are not 16 bytes MUST be treated as invalid and rejected.
