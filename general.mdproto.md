---
syntax: "proto3"
package: "sirius.msgdef"
import:
  - "common-types.proto"
---

# GENERAL MESSAGE DEFINITIONS

General message definitions used on the main channel of the Sirius protocol.
Feature-specific message definitions are located in their respective mdproto files.

Every Sirius protocol message is assigned a fixed 2-byte opcode, encoded in Big-Endian byte order.
The message framing structure is as follows:

- 2 bytes: opcode (Big-Endian)
- 4 bytes: payload length (Big-Endian, excluding the opcode and length fields)
- N bytes: payload (Protobuf v3 serialized data)

For example, a message with opcode `0x0001` and a payload length of 10 bytes is encoded as
`00 01 | 00 00 00 0A | [10 bytes payload]`.

---

# CONSTANTS

## SiriusProtocolVersion

```constset
/// The lowest 1 byte is reserved for identifying variants such as development versions.
constset SiriusProtocolVersion: uint32 {
  /// v1.0 (draft version)
  const v1_0_draft = 0x00010000;

  /// v1.0
  const v1_0 = 0x00010001;
}
```

### DESCRIPTION

- A set of constants representing the Sirius protocol version.
- The currently active version is `v1_0_draft`. `v1_0` is reserved for the official release.

## ClosureCode

```constset
constset ClosureCode: uint32 {
  /// The session was closed normally.
  const successful = 0;

  /// The session was closed due to a protocol error.
  /// This occurs when an invalid message is received or the protocol specification is violated.
  /// Additional information MAY be provided via the message field if needed.
  const protocolError = 1;

  /// The session was closed due to an internal server error.
  const internalServerError = 2;

  /// The session was closed due to an authentication failure.
  const authenticationFailed = 3;

  /// Failed to allocate a session seat.
  /// This typically occurs due to insufficient server resources or exceeding the maximum connection limit.
  const sessionAllocationFailed = 4;
}
```

---

# ENUM TYPES

## NoticeSeverity

```protobuf
enum NoticeSeverity {
  INFO = 0;
  WARNING = 1;
  ERROR = 2;
  FATAL = 3;
}
```

### DESCRIPTION

Indicates the severity level of a `ServerNotice` message.

- `INFO`: General informational message
- `WARNING`: Warning message requiring attention
- `ERROR`: Error message indicating that a problem has occurred
- `FATAL`: Fatal error message indicating that the session can no longer be maintained

---

# MESSAGES

## ServerNotice

```protobuf
/// A message sent by the server to deliver notices to the client.
// @opcode: 0x0001
message ServerNotice {
  NoticeSeverity severity = 1;
  uint32 code = 2;
  string message = 3;
  uint64 timestamp = 4;
}
```

### DESCRIPTION

- Used by the server to deliver notices to the client.
- This message is used to communicate errors and warnings in various situations, such as handshake rejection, authentication failure, and protocol errors.
- When the `severity` is `FATAL`, the server MAY terminate the connection after sending this message.

### IMPLEMENTATION NOTES

- The `code` field is a numeric code defined at the application level. The Sirius protocol itself only recommends a range for code values; the specific meaning is determined by the server implementation.
- The `timestamp` field is in milliseconds since the Unix epoch.

## ClientHello

```protobuf
/// Initial handshake message sent by the client when connecting to the server.
// @opcode: 0x0002
message ClientHello {
  // @constset: SiriusProtocolVersion
  uint32 protocolVersion = 1;

  // TODO: should be changed to optional
  string agentName = 2;
}
```

### DESCRIPTION

- The initial handshake message sent by the client when connecting to the server.
- The client MUST send this message immediately after the main channel is opened. Upon receiving it, the server responds with either a `ServerHello` or a `ServerNotice`.
- The `protocolVersion` field informs the server of the protocol version supported by the client.

### IMPLEMENTATION NOTES

- Server implementations MUST validate the `protocolVersion` to ensure compatibility. If the version is incompatible, the server MUST send a `ServerNotice` with `FATAL` severity and close the connection.

## ServerHello

```protobuf
/// Response message sent by the server to the client's connection request.
// @opcode: 0x0003
message ServerHello {
  // @constset: SiriusProtocolVersion
  uint32 protocolVersion = 1;

  /// List of features supported by the server (array of SiriusFeatureID UUIDs)
  repeated SRUUID supportedFeatures = 2;

  /// Server name (MAY be omitted depending on server policy)
  optional string serverName = 3;

  /// Message of the day (MAY be omitted depending on server policy)
  optional string motd = 4;
}
```

### DESCRIPTION

- The server's response to a `ClientHello` message.
- The `protocolVersion` field informs the client of the protocol version the server will use.
- The `supportedFeatures` field contains a list of UUIDs representing the features supported by the server (see SiriusFeatureID).

## Goodbye

```protobuf
/// Connection closure message.
// @opcode: 0x0004
message Goodbye {
  // @constset: ClosureCode
  uint32 code = 1;
  optional string message = 2;
}
```

### DESCRIPTION

- Sent by either the server or the client when closing the connection.
- The `code` field indicates the reason for closure (see ClosureCode), and the `message` field MAY optionally contain additional explanation.
- Upon receiving this message, the recipient MUST close the connection immediately.

### IMPLEMENTATION NOTES

- Goodbye is a bidirectional message. It can be sent by either the server or the client.
