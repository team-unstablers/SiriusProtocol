---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
  - "v1/channels/projection/projection_typedef.proto"
  - "v1/channels/projection/displayman.proto"
---

# PROJECTION DISPLAY MANIPULATION

Message definitions for display manipulation, including layout changes on existing displays, enabling/disabling physical displays, and creating/destroying virtual displays. Operations are delivered as a single atomic transaction so that the server can either apply the entire batch or roll it back as a whole.

---

# STRUCTS

## DisplayLayoutChange

```protobuf
/// Requests a spec/rotation change for an existing display.
message DisplayLayoutChange {
  /// Target display identifier.
  uint32 displayID = 1;

  /// Desired display spec (resolution, refresh rate, scale factor).
  optional DisplaySpec spec = 2;

  /// Desired rotation.
  // @constset: DisplayRotation
  optional uint32 rotation = 3;

  /// Desired display origin (top-left corner in the global coordinate space).
  optional SRPoint origin = 4;
}
```

### IMPLEMENTATION NOTES

- The server MAY or MAY NOT honor `spec`. Clients MUST tolerate a different spec being selected, for example when the requested mode is not supported by the target display.

## DisplaySetEnable

```protobuf
/// Enables or disables a display.
message DisplaySetEnable {
  /// Target display identifier.
  uint32 displayID = 1;
  /// Whether the display should be enabled (true) or disabled (false).
  bool isEnabled = 2;
}
```

## VirtualDisplayCreate

```protobuf
/// Requests creation of a virtual display.
message VirtualDisplayCreate {
  /// Client-assigned identifier for this virtual display.
  SRUUID identifier = 1;

  /// Candidate specs, in order of client preference.
  repeated DisplaySpec desiredSpecs = 2;

  /// Optional description of the virtual display's intended purpose.
  optional string purpose = 3;

  /// Additional metadata.
  /// Implementation-specific keys SHOULD follow the reverse domain name format.
  map<string, string> metadata = 15;
}
```

### IMPLEMENTATION NOTES

- `identifier` MUST be unique within the current session. If the client supplies an identifier that is already in use, the server MUST reject the enclosing transaction.
- The server MAY or MAY NOT honor `desiredSpecs` and `purpose`. Clients MUST tolerate a spec that differs from any of the requested candidates.

## VirtualDisplayDestroy

```protobuf
/// Requests destruction of a virtual display.
message VirtualDisplayDestroy {
  /// Identifier of the virtual display to destroy, as assigned at creation time.
  SRUUID identifier = 1;
}
```

### IMPLEMENTATION NOTES

- If no virtual display matches the given `identifier`, the server MAY silently ignore this operation rather than failing the enclosing transaction.

## DisplayOperation

```protobuf
/// A single operation within a DisplayTransactionRequest.
message DisplayOperation {
  oneof operation {
    DisplayLayoutChange change = 1;
    DisplaySetEnable setEnable = 2;
    VirtualDisplayCreate createVirtualDisplay = 3;
    VirtualDisplayDestroy destroyVirtualDisplay = 4;
  }
}
```

---

# MESSAGES

## DisplayTransactionRequest

```protobuf
/// Applies a batch of display operations atomically.
// @opcode: 0x8048
message DisplayTransactionRequest {
  /// Transaction identifier, chosen by the client.
  SRUUID transactionID = 1;
  /// Operations to apply, in order.
  repeated DisplayOperation operations = 2;
  /// If set, the display with this ID is promoted to the main display as part of the transaction.
  optional uint32 mainDisplayID = 3;
}
```

### DESCRIPTION

Used by the client to apply one or more display operations (layout changes, enable/disable, virtual display create/destroy) as a single atomic batch. The server replies with a `DisplayTransactionResponse` carrying the same `transactionID`.

### IMPLEMENTATION NOTES

- The server MUST either apply every operation in the transaction or apply none of them.
- When the transaction fails, the server MUST roll back any partial changes and return a `DisplayTransactionResponse` with `isSuccess = false`.
- When the transaction succeeds, the server MUST emit the corresponding `DisplayChangedEvent`s (for example, `modified` or `becamePrimary`) defined in `displayman.mdproto.md` to all subscribed clients.

## DisplayTransactionResponse

```protobuf
/// Response to a DisplayTransactionRequest.
// @opcode: 0x8049
message DisplayTransactionResponse {
  /// Transaction identifier. MUST match the transactionID from the request.
  SRUUID transactionID = 1;

  /// Whether the transaction was applied successfully.
  bool isSuccess = 2;

  /// Optional human-readable error message when isSuccess is false.
  optional string reason = 3;
}
```

### DESCRIPTION

Sent by the server in response to a `DisplayTransactionRequest`. The `transactionID` MUST match the request's identifier so the client can correlate it.
