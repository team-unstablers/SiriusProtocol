---
syntax: "proto3"
package: "sirius.msgdef"
---

# COMMON TYPES

Common data type definitions shared across the Sirius protocol.

---

# CONSTANTS

## CompressionMethod

```constset
constset CompressionMethod: string {
  /// No compression. Payload is transmitted as raw bytes.
  const none = "none";
  /// Zstandard (RFC 8478).
  const zstd = "zstd";
  /// Raw DEFLATE (RFC 1951) without zlib or gzip wrappers.
  const deflate = "deflate";
  /// Brotli (RFC 7932).
  const brotli = "brotli";
}
```

### DESCRIPTION

Identifies the compression algorithm applied to a payload. Used wherever the Sirius protocol negotiates or carries compressed bytes (e.g., the `transfer` channel data plane and the `fsaccess_mount` channel data plane).

A `string`-typed identifier was chosen so that future revisions can introduce additional methods without consuming a finite numeric space, and so that on-the-wire captures remain self-explanatory during debugging.

### IMPLEMENTATION NOTES

The `none` value is reserved to represent the explicit decision NOT to compress. Negotiation flows SHOULD prefer emitting `none` over omitting the field, so that "no compression chosen" is distinguishable from "field not yet populated by an older peer". Concrete channels MAY define an absence-as-default convention; in that case absence and `none` MUST be treated as equivalent.

Receivers iterating over a candidate list MUST ignore unrecognized identifiers and continue with the next candidate; when none of the candidates is recognized, the receiver falls back to `none`. For single-value fields carrying an already-selected method, an unrecognized identifier MUST be treated as if `none` had been selected. This common policy lets a peer running a newer revision of the registry remain interoperable with an older peer that has not yet learned a new identifier, without requiring per-message escape hatches. Surrounding messages MAY tighten this policy (for example, by promoting an unrecognized identifier in a specific field to a channel-fatal violation) but MUST do so explicitly.

The set of identifiers above is the canonical registry as of this revision. Implementations adding a new method SHOULD update this constset rather than minting an ad-hoc string, so that all peers share a single source of truth for spelling and meaning.

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
