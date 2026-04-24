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

The payload length MUST NOT exceed 16 MiB (16,777,216 bytes).
See the "FRAME SIZE LIMIT" section of the protocol introduction for the full specification.

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

  // Values 3 and 4 are reserved (previously assigned to authenticationFailed and
  // sessionAllocationFailed; those conditions are now reported via ServerNoticeCode,
  // followed by a Goodbye carrying one of the codes above).
}
```

## ServerNoticeCode

```constset
constset ServerNoticeCode: uint32 {
  // MISC (0x0700 ~ 0x0FFF)

  /// "Jackpot!" - a non-normative easter-egg notice code.
  /// MUST NOT be used outside of genuine easter-egg contexts.
  /// See the "JACKPOT (NON-NORMATIVE)" appendix for details.
  const jackpot                 = 0x0777;

  // CLIENT FAULT (0x4000 ~ 0x4FFF)

  /// The server received a message of a type it does not support or recognize.
  /// This is typically used when a server that does not support a newer version of the
  /// Sirius protocol receives a message introduced in that newer version.
  const unsupportedOpcode       = 0x4005;

  /// The client selected an authentication method that the server does not support.
  const unsupportedAuthMethod   = 0x4006;

  /// The nonce in an AuthRequest does not match the nonce from the corresponding AuthChallenge.
  const nonceMismatch           = 0x4007;

  /// The peer did not respond within the configured timeout.
  const timeout                 = 0x4008;

  /// Authentication failed.
  const authenticationFailed    = 0x4009;

  /// The payload length of an incoming frame exceeds the protocol-defined limit.
  /// See the "FRAME SIZE LIMIT" section of the protocol introduction.
  const frameTooLarge           = 0x400A;

  // SERVER FAULT (0x5000 ~ 0x5FFF)

  /// An unexpected error occurred on the server.
  const internalServerError     = 0x5000;

  /// The server could not allocate a session seat, typically due to insufficient
  /// resources or exceeding the maximum number of concurrent sessions.
  const sessionAllocationFailed = 0x5001;
}
```

### DESCRIPTION

A set of codes used in the `code` field of `ServerNotice` messages.
Codes are grouped by the originator of the fault: the `0x4000`-range indicates a
client-side fault (e.g., a malformed request or failed authentication), while the
`0x5000`-range indicates a server-side fault (e.g., resource exhaustion).

### IMPLEMENTATION NOTES

- Implementations MAY use codes outside the defined set for application-specific purposes,
  but SHOULD prefer the protocol-defined codes when applicable.
- The `0x0000`-`0x3FFF` range is reserved for future protocol use and MUST NOT be used
  by application-specific codes, except for the `0x0700`-`0x0FFF` sub-range which is
  reserved for non-normative codes.

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
  // @constset: ServerNoticeCode
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

- The `code` field carries a `ServerNoticeCode` value. Implementations MAY use codes outside the defined set for application-specific purposes, but SHOULD prefer the protocol-defined codes when applicable.
- The `timestamp` field is in milliseconds since the Unix epoch.
- When using a `FATAL`-severity notice to close the connection, the server MUST follow the sequence below:
  1. Send this `ServerNotice` with `severity = FATAL` and an appropriate `code`.
  2. Send a `Goodbye` with an appropriate `ClosureCode` (`protocolError` for client-induced faults; `internalServerError` for server-side faults).
  3. Close the transport connection.
- Representative mappings of this pattern:
  - Authentication failure: `ServerNoticeCode.authenticationFailed` → `Goodbye(ClosureCode.protocolError)`
  - Session allocation failure: `ServerNoticeCode.sessionAllocationFailed` → `Goodbye(ClosureCode.internalServerError)`
  - Frame size limit violation: `ServerNoticeCode.frameTooLarge` → `Goodbye(ClosureCode.protocolError)`

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

---

# JACKPOT (NON-NORMATIVE)

> Wait - why is this even a thing?

This appendix describes the `ServerNoticeCode.jackpot` easter egg. It is non-normative
and provided for amusement purposes only. Implementations MAY ignore this section
entirely.

Some server implementations perform a "lottery draw" for each incoming connection and
deliver the `jackpot` notice code to winning clients.

## DRAW MECHANICS

The specific mechanics of the draw are implementation-defined. Representative examples:

- A per-connection scratch-off draw triggered at connect time.
- A periodic draw performed on each server tick.
- A milestone draw triggered by a specific event (e.g., the Nth connection).
- ...

The winning probability is implementation-defined, but it SHOULD be low enough that
winning feels genuinely surprising. A probability around **1/777** is RECOMMENDED.
(The number itself looks meaningful, doesn't it?)

## REWARDS

Winning clients MAY receive implementation-defined rewards. Representative examples:

- The server MAY attach a congratulatory message in `ServerNotice.message`.
- The server MAY guarantee highest-priority scheduling for the winning session
  for the duration of that session.
- ...

## DISCLAIMER

This is a joke. **The `jackpot` code MUST NOT be used outside of genuine easter-egg
contexts.**

This easter egg MUST NOT be enabled in enterprise environments. Those users are serious.
