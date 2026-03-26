---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
  - "v1/channels/projection/projection_typedef.proto"
---

# PROJECTION WINDOW MANAGEMENT (MANIPULATION)

Message definitions for window manipulation operations such as state changes, focus control, and geometry adjustments.

---

# CONSTANTS

## WindowStateCommand

```constset
constset WindowStateCommand: uint32 {
  /// Unknown operation
  const unknown = 0;
  /// Close the window.
  const close = 1;
  /// Minimize the window.
  const minimize = 2;
  /// Maximize the window.
  const maximize = 3;
  /// Restore the window to its original size.
  const restore = 4;
}
```

---

# MESSAGES

## WindowManipulationRequest

```protobuf
/// Requests a manipulation operation on a window.
// @opcode: 0x8081
message WindowManipulationRequest {
  uint64 windowId = 1;

  oneof operation {
    /// Performs a state change operation such as close, minimize, maximize, or restore.
    /// This operation MAY not succeed
    /// (e.g., a confirmation dialog is shown, or the window role prevents closing).
    // @constset: WindowStateCommand
    uint32 stateCommand = 2;

    /// Sets the focus state of this window.
    /// When set to true, the window receives focus;
    /// when set to false, focus is removed from the window.
    /// On most window managers, focusing a window brings it to the foreground.
    bool focus = 3;

    /// Changes the geometry of this window.
    SRRect setGeometry = 4;

    /// Sets flags on this window (window->flags() | flagMask).
    /// For safety, only application-level flags as defined in `WindowInfoFlags` MAY be set.
    /// The server implementation MAY silently ignore flags deemed unsafe.
    uint64 setFlags = 13;

    /// Clears flags on this window (window->flags() & ~flagMask).
    /// For safety, only application-level flags as defined in `WindowInfoFlags` MAY be cleared.
    /// The server implementation MAY silently ignore flags deemed unsafe.
    uint64 clearFlags = 14;
  }

  /// Reserved for future extension
  map<string, string> extraArgs = 15;
  /// Reserved for future extension
  uint32 flags = 16;
}
```

### DESCRIPTION

- Used by the client to request a manipulation operation on a specific window.
- Supported operations include state changes (close/minimize/maximize/restore), focus control, geometry adjustment, and flag set/clear.

### IMPLEMENTATION NOTES

- Closing a window via `stateCommand` MAY not succeed immediately, as the application may present a confirmation dialog.
- `setFlags` and `clearFlags` MUST only target application-level flags defined in `WindowInfoFlags`, not OS-level flags.

## WindowManipulationResponse

```protobuf
/// Response to a window manipulation request.
// @opcode: 0x8082
message WindowManipulationResponse {
  bool isSuccess = 1;

  uint32 code = 2;
  optional string message = 3;
}
```

### DESCRIPTION

- Response message for `WindowManipulationRequest`.
- The `isSuccess` field indicates whether the operation succeeded. On failure, `code` and `message` provide details about the reason.
