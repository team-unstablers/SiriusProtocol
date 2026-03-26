---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
  - "v1/channels/projection/projection_codec.proto"
  - "v1/channels/projection/projection_source.proto"
---

# PROJECTION SESSION MESSAGES

Message definitions for video projection session creation, stopping, status reporting, and events.

---

# CONSTANTS

## VideoSessionEndReason

```constset
constset VideoSessionEndReason: int32 {
  const unknown = 0;
  /// Ended by client request
  const clientRequested = 1;
  /// Display disconnected
  const displayDisconnected = 2;
  /// Ended due to an internal error
  const internalError = 3;
  /// Recorder (screen capture) failure
  const recorderFailed = 4;
  /// Data channel error
  const dataChannelError = 5;
}
```

### DESCRIPTION

- Indicates the reason why a projection session was ended.
- Used in the `ProjectionSessionEndedEvent.reason` field.

## VideoSessionFailureReason

```constset
constset VideoSessionFailureReason: int32 {
  const unknown = 0;
  /// The requested source was not found
  const sourceNotFound = 1;
  /// Codec negotiation failed
  const codecNegotiationFailed = 2;
  /// Internal server error
  const serverError = 3;
}
```

### DESCRIPTION

- Indicates the reason why a projection session creation failed.
- Used in the `ProjectionSessionCreationFailedEvent.reason` field.

---

# MESSAGES

## ProjectionRequest

```protobuf
/// Requests creation of a video projection session.
// @opcode: 0x8001
message ProjectionRequest {
  /// Session identifier
  SRUUID identifier = 1;

  /// Source to project (display, region, or single window)
  ProjectionSource viewport = 2;
  /// List of codecs preferred by the client (in priority order)
  repeated Codec preferredCodecs = 3;
}
```

### DESCRIPTION

- Used by the client to request the creation of a video projection session from the server.
- The server performs codec negotiation based on `preferredCodecs` and responds with `ProjectionSessionCreatedEvent` on success or `ProjectionSessionCreationFailedEvent` on failure.

### IMPLEMENTATION NOTES

- Upon successful codec negotiation, the server opens a separate `ProjectionDataChannel` to transmit encoded video frames.
- The actual resolution is determined by comparing the content size of the source specified in `viewport` with the `size` field of `preferredCodecs`.

## StopProjectionRequest

```protobuf
/// Requests stopping a video projection session.
// @opcode: 0x8002
message StopProjectionRequest {
  /// Identifier of the session to stop
  SRUUID identifier = 1;
}
```

### DESCRIPTION

- Used by the client to request the stopping of an ongoing video projection session.
- The server terminates the session and closes the associated data channel.

## ProjectionPerformanceReport

```protobuf
/// Client's projection performance report.
// @opcode: 0x8003
message ProjectionPerformanceReport {
  /// Projection session identifier
  SRUUID identifier = 1;

  /// Number of frames received during the interval
  uint32 receivedFrameCount = 2;
  /// Number of frames successfully decoded during the interval
  uint32 decodedFrameCount = 3;
  /// Number of frames dropped during the interval
  uint32 droppedFrameCount = 4;

  /// Average decoding time during the interval (in milliseconds)
  uint32 averageDecodeTimeMs = 5;
}
```

### DESCRIPTION

- A performance report sent periodically by the client to the server during a projection session.
- The server can use this report to gauge the client's decoding performance and network conditions, adjusting video quality as needed.

### IMPLEMENTATION NOTES

- All frame count fields represent frames received/processed since the last report (interval-based values, not cumulative).
- The server's AutoQuality planner uses this report to dynamically adjust bitrate and resolution.

## ProjectionSessionCreatedEvent

```protobuf
/// Event emitted when a projection session is successfully created.
// @opcode: 0x8021
message ProjectionSessionCreatedEvent {
  /// Session identifier
  SRUUID identifier = 1;

  /// Actual projection source
  ProjectionSource source = 2;
  /// Negotiated final codec
  Codec codec = 3;
}
```

### DESCRIPTION

- Sent by the server to the client when a session has been successfully created after processing a `ProjectionRequest`.
- The `codec` field contains the final codec settings determined through negotiation. The client MUST initialize its decoder based on this.

## ProjectionSessionCreationFailedEvent

```protobuf
/// Event emitted when a projection session creation fails.
// @opcode: 0x8022
message ProjectionSessionCreationFailedEvent {
  /// Session identifier
  SRUUID identifier = 1;
  /// Failure reason
  // @constset: VideoSessionFailureReason
  int32 reason = 2;

  /// Detailed error message
  optional string message = 3;
}
```

### DESCRIPTION

- Sent by the server when session creation fails while processing a `ProjectionRequest`.
- Possible reasons include codec negotiation failure, source not found, and internal server errors.

## ProjectionSessionChangedEvent

```protobuf
/// Event emitted when projection session settings are changed.
// @opcode: 0x8023
message ProjectionSessionChangedEvent {
  /// Session identifier
  SRUUID identifier = 1;
  int32 reason = 2; // TODO: constset definition needed

  /// Changed projection source (not set if unchanged)
  ProjectionSource source = 3;
  /// Changed codec (not set if unchanged)
  Codec codec = 4;
}
```

### DESCRIPTION

- Sent by the server to the client when the source or codec settings of a projection session have been changed.
- If the `codec` field is set, the client MUST reconfigure its decoder.

### IMPLEMENTATION NOTES

- The current server implementation sends `reason = 1` for resolution changes (resolutionChanged). This will be defined as a constset in the future.

## ProjectionSessionEndedEvent

```protobuf
/// Event emitted when a projection session has ended.
// @opcode: 0x8024
message ProjectionSessionEndedEvent {
  /// Session identifier
  SRUUID identifier = 1;
  /// End reason
  // @constset: VideoSessionEndReason
  int32 reason = 2;

  /// Detailed message
  optional string message = 3;
}
```

### DESCRIPTION

- Sent by the server to the client when a projection session has ended.
- May be sent not only for client-requested termination, but also for server-side reasons such as display disconnection, internal errors, or data channel errors.
