---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.transfer"
import:
- "common-types.proto"
---

# TRANSFER CHANNEL

Message definitions for exchanging files, clipboard data, or other large data with the remote host via the transfer channel.

- **Multiple transfer channels can exist**: A separate channel MAY be created for each file transfer or purpose-specific data transfer.
- **Transfer channels are single-use**: Once the transfer is complete, the channel is closed. A new channel MUST be created for a new transfer.
- **Transfer channels are unidirectional**: Each channel transmits data in one direction only. If bidirectional communication is needed, two channels may be required.
    - The sending side (uploader) creates the channel, and the receiving side (downloader) may accept the channel creation request.
    - Conversely, the receiving side may create the channel, and the sending side may accept the channel creation request.

Like other Sirius channels, this channel is designed to operate over a reliable transport.

---

# CHANNEL ARGUMENTS

This channel receives a fixed-arity, four-element argument list via the `args` field of the `ChannelStartRequest` message:

```typescript
type TransferChannelArgs = [
    purpose: 'file-transfer' | 'clipboard-data' | string, // transfer purpose
    direction: 'upload' | 'download',                     // transfer direction
    purposeArgs: string,    // 'key=value&key2=value2' query-string; see Encoding below
    transferArgs: string,   // 'key=value&key2=value2' query-string; see Encoding below
];
```

The first two slots remain plain identifiers as in earlier revisions. The last two slots carry **URI-style query strings** so that purpose-specific argument schemas and channel-wide options can each evolve as named keys without rearranging the argument list.

`args.length` MUST be exactly 4. Receivers MUST reject `ChannelStartRequest` whose `args` length differs from 4.

## Encoding

`purposeArgs` and `transferArgs` follow **RFC 3986 query-component** conventions, with the following clarifications:

- Pairs are delimited by `&`. Within a pair, key and value are delimited by the first `=`.
- Keys are bare ASCII identifiers matching `^[a-z][a-z0-9-]*$` and are NOT percent-encoded.
- Values are percent-encoded for any character outside the unreserved set (`A-Z` / `a-z` / `0-9` / `-` / `_` / `.` / `~`). In particular, `&`, `=`, `,`, `/`, and space inside a value MUST be percent-encoded as `%26`, `%3D`, `%2C`, `%2F`, and `%20` respectively.
- `+` is treated as a literal plus sign (RFC 3986 query semantics) and NOT as an alias for space (form-urlencoded semantics). Senders that want a space MUST emit `%20`.
- An empty value (`key=`) is allowed and means "the key is present with an empty string value". A bare key (`key` without `=`) is a parse error: senders MUST NOT emit one, and receivers MUST reject the `ChannelStartRequest` with a `ServerNotice` indicating a protocol violation when one is observed.
- List-valued options use `,` as the in-value separator. Commas inside an individual list item MUST be percent-encoded as `%2C`.
- An empty `purposeArgs` or `transferArgs` slot is signaled by an empty string `""` (the slot itself is mandatory and MUST be present).

Receivers MUST percent-decode each value after splitting on `&` and `=`, and MUST ignore tokens whose key they do not recognize so that future revisions can introduce additional keys without breaking forward compatibility.

## Defined `purposeArgs` keys

For `purpose = 'file-transfer'`:

| Key | Value format | Required | Meaning |
|-----|--------------|----------|---------|
| `name` | percent-encoded UTF-8 string | yes | Display name of the file being transferred. |
| `path` | percent-encoded absolute path | no | Sender-side absolute path. Advisory only; see the channel's security notes. |
| `offset` | decimal `uint64` | no (default `0`) | Starting byte offset of the requested range. |
| `length` | decimal `uint64`. The sentinel value `18446744073709551615` (`0xFFFFFFFFFFFFFFFF`, `UINT64_MAX`) means "transfer from `offset` until EOF"; any other value is the exact number of bytes to transfer. Senders MUST NOT emit `0` for an open-ended transfer (use the sentinel). | yes | Number of bytes to transfer from `offset`. |

For `purpose = 'clipboard-data'`:

| Key | Value format | Required | Meaning |
|-----|--------------|----------|---------|
| `item-index` | decimal `uint32` | yes | Index of the clipboard item being transferred. |
| `representation-index` | decimal `uint32` | yes | Index of the representation within the item. |

## Defined `transferArgs` keys

| Key | Value format | Meaning | Default when absent |
|-----|--------------|---------|---------------------|
| `compress` | comma-separated list of `CompressionMethod` identifiers, ordered by sender preference | The sender proposes this set of compression methods for the channel's data plane. The receiver picks one in `TransferReady`. | `none` (no compression negotiated; see WIRE FLOW for the fallback rule) |

## Examples

A file upload proposing Zstd as the preferred compression with raw DEFLATE as a fallback. The original filename contains `+` and the original path contains `/`, both of which are percent-encoded inside their values:

```
[
  'file-transfer',
  'upload',
  'name=report%2Bv2.psd&path=%2Ftmp%2Fabcd&offset=0&length=524288',
  'compress=zstd,deflate'
]
```

A clipboard-data download with no compression negotiated (the `transferArgs` slot is the empty string):

```
[
  'clipboard-data',
  'download',
  'item-index=0&representation-index=0',
  ''
]
```

---

# MESSAGES

## TransferStartNotification

```protobuf
/// Transfer start notification message.
// @opcode: 0x8001
message TransferStartNotification {
  /// Actual name of the data to be transferred
  string name = 1;

  /// Total size of the data to be transferred (in bytes)
  uint64 totalSize = 2;

  /// MIME type of the data to be transferred (e.g., "application/pdf", "text/plain")
  string contentType = 3;

  /// Description of the data to be transferred (e.g., clipboard data description)
  string description = 4;
}
```

### DESCRIPTION

Sent once by the sender to introduce the transfer and supply metadata that does not vary per chunk. Following messages on this channel are `TransferDataChunk` until end-of-file.

### IMPLEMENTATION NOTES

This message MUST NOT be sent until the corresponding `TransferReady` has been received, regardless of whether the sender proposed any compression methods via the `compress=` token in `transferArgs`. See the WIRE FLOW appendix for the full ordering.

`totalSize` reflects the **uncompressed** logical size of the payload, not the size of the compressed bytes that will travel on the wire. Receivers use this value to display progress and to validate completeness against the sum of decompressed chunk lengths; see the IMPLEMENTATION NOTES of `TransferDataChunk` for the validation rule.

## TransferReady

```protobuf
/// Acknowledgment from the receiver that confirms the negotiated compression method for this channel. Sent exactly once, before any TransferStartNotification is processed by the sender.
// @opcode: 0x8003
message TransferReady {
  /// Compression method picked from the candidates the sender proposed in
  /// the channel-start `compress=` option, or `none` when the sender did not
  /// propose any. MUST be one of the proposed candidates, or `none` (which
  /// the receiver MAY pick at any time, regardless of what was proposed).
  /// Stable for the lifetime of the channel.
  // @constset: CompressionMethod
  string compressionMethod = 1;
}
```

### DESCRIPTION

Sent by the receiver immediately after a successful `ChannelStartResponse`, addressed back to the sender on the same channel. Confirms the single compression method that will apply to every subsequent `TransferDataChunk.data` for the lifetime of this channel. The sender MUST NOT begin emitting `TransferStartNotification` or any `TransferDataChunk` until it has received `TransferReady`.

The receiver's choice MUST come from the candidate list the sender advertised in the `compress=` token of `transferArgs`, or be `none`. The receiver MAY pick any candidate it supports; conventionally it picks the first candidate it can decode. The receiver MAY also pick `none` regardless of what the sender proposed (for example, when local policy disables compression for this channel, or when the sender did not propose any methods), and the sender MUST honor that choice.

### IMPLEMENTATION NOTES

This message is sent **exactly once per channel**, by the receiver, regardless of whether the sender included a `compress=` token in `transferArgs`. When the sender did not propose any compression methods, the receiver MUST still send `TransferReady` with `compressionMethod = "none"`. Senders that have not received `TransferReady` after a successful `ChannelStartResponse` MUST NOT proceed and SHOULD close the channel with a `ServerNotice` indicating a protocol violation after an implementation-defined timeout. Receivers MUST NOT send `TransferReady` more than once; senders that observe a duplicate MUST close the channel with a `ServerNotice` indicating a protocol violation.

If the receiver sends `TransferReady` with a `compressionMethod` that was NOT in the sender's `compress=` candidate list and is not `none`, the sender MUST close the channel with a `ServerNotice` indicating a negotiation failure. An empty `compressionMethod` string is treated as equivalent to `none`.

## TransferDataChunk

```protobuf
/// Transfer data chunk.
// @opcode: 0x8002
message TransferDataChunk {
  /// Sequence number of this data chunk (starting from 0)
  uint64 sequenceNumber = 1;

  /// Data contained in this chunk. When a non-`none` compression method has
  /// been negotiated (see TransferReady), this field carries the
  /// compressed bytes; otherwise it carries the raw payload.
  bytes data = 2;

  /// CRC32 of the wire bytes carried in `data` (i.e., the compressed bytes
  /// when compression is in effect). Optional; when absent, the sender did
  /// not compute a CRC and receivers MUST NOT validate against this field.
  optional uint32 crc32 = 3;

  /// Whether this chunk is the last chunk of the transfer
  bool isEOF = 4;
}
```

### DESCRIPTION

Carries one chunk of the transfer payload. Chunks are emitted in strictly increasing `sequenceNumber` order starting at zero, and the receiver MUST reject any chunk whose `sequenceNumber` does not match the expected next value.

### IMPLEMENTATION NOTES

When a non-`none` compression method has been negotiated via `TransferReady`, each chunk's `data` field MUST be a **self-contained, independently decodable compressed frame** in the negotiated format (a Zstandard frame for `zstd`, a raw DEFLATE block sequence for `deflate`, a Brotli stream for `brotli`). Cross-chunk shared dictionaries or split frames are NOT permitted, so that a receiver can decompress each chunk in isolation without buffering preceding chunks.

The `crc32` field, when present, is computed over the **wire bytes carried in `data`** (i.e., the compressed bytes when compression is in effect, or the raw bytes when `compressionMethod = none`), so receivers can validate integrity before invoking the decompressor. When the field is absent, receivers MUST skip the integrity check entirely rather than validating against a default value. Validation against the original uncompressed bytes is delegated to the decompressor's own internal checks (e.g., Zstandard's frame checksum) when the format provides them.

`totalSize` from `TransferStartNotification` refers to the uncompressed logical size. Receivers SHOULD validate completeness by accumulating decompressed chunk lengths and comparing the total to `totalSize` once `isEOF` is observed; a mismatch indicates a corrupt or truncated transfer and the receiver MUST treat the result as discardable.

---

# WIRE FLOW

The full ordering of messages on a transfer channel is:

1. **Sender** opens the channel with `ChannelStartRequest`, supplying `transferArgs` that MAY include a `compress=<csv>` token to propose compression methods.
2. **Receiver** replies with `ChannelStartResponse` (success or failure as defined by the channel infrastructure). On failure, the channel ends here.
3. **Receiver** sends `TransferReady` with the picked `compressionMethod`. This step is REQUIRED on every transfer channel; when the sender did not include a `compress=` token, the receiver MUST send `TransferReady` with `compressionMethod = "none"`.
4. **Sender** sends `TransferStartNotification` with the transfer's metadata.
5. **Sender** sends one or more `TransferDataChunk` messages with strictly increasing `sequenceNumber`, the last of which has `isEOF = true`.
6. The channel closes once the final chunk is acknowledged at the transport layer; either side MAY close earlier on error.

The sender MUST NOT proceed from step 2 to step 4 until step 3 has occurred. The receiver MUST NOT send `TransferReady` after step 4 has begun.
