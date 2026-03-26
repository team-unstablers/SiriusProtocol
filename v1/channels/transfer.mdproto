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

This channel receives the following arguments via the `args` field of the `ChannelStartRequest` message:

```typescript
type TransferChannelArgs = [
    purpose: 'file-transfer' | 'clipboard-data' | string, // transfer purpose
    direction: 'upload' | 'download', // transfer direction
    ...args: string[] // additional arguments depending on the transfer purpose (e.g., filename, clipboard data index, etc.)
];
```

---

# MESSAGES

## TransferStartNotification

Transfer start notification message.

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

## TransferDataChunk

```protobuf
/// Transfer data chunk.
// @opcode: 0x8002
message TransferDataChunk {
  /// Sequence number of this data chunk (starting from 0)
  uint64 sequenceNumber = 1;

  /// Data contained in this chunk
  bytes data = 2;

  uint32 crc32 = 3;

  /// Whether this chunk is the last chunk of the transfer
  bool isEOF = 4;
}
```
