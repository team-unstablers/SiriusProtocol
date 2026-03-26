---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.hidio"
import:
  - "common-types.proto"
  - "v1/channels/hidio/hid_mouse.proto"
---

# PEN MESSAGES

Message definitions for pen input.

---

# CONSTANTS

## PenTipType

```constset
constset PenTipType: uint32 {
  /// Normal pen tip (used for drawing / writing)
  const pen = 0;

  /// Eraser tip (used for erasing pen input)
  const eraser = 1;

  /// Pointer (used only as a pointing device; no pressure or tilt information is provided)
  const pointer = 2;
}
```

## PenProximityEventType

```constset
constset PenProximityEventType: uint32 {
  /// The pen entered the tablet's detection range
  const enter = 0;
  /// The pen left the tablet's detection range
  const leave = 1;
}
```

## PenButtonEventType

```constset
constset PenButtonEventType: uint32 {
  /// Pen button pressed
  const down = 0;

  /// Pen button released
  const up = 1;
}
```

---

# STRUCTS

## PenContext

```protobuf
message PenContext {
  /// Unique pen ID (identifier to distinguish pen inputs)
  uint32 penId = 1;

  /// Pressure information. Value in the range 0.0 to 1.0
  float pressure = 3;

  /// Pen tilt information
  /// tiltX: degree to which the pen is tilted to the left on the horizontal plane (-1.0 to 1.0)
  /// tiltY: degree to which the pen is tilted upward on the horizontal plane (-1.0 to 1.0)
  float tiltX = 4;
  float tiltY = 5;

  /// Pen rotation (azimuth) information
  /// Value in the range 0 to 360, increasing clockwise from the north direction.
  float azimuth = 6;

  /// Pen altitude information
  /// Value in the range 0 to 90, measured from the vertical direction.
  float altitude = 7;
}
```

---

# MESSAGES

## PenSetupEvent

Message for setting up a tablet pen.

```protobuf
message PenSetupEvent {
  /// Unique pen ID (self-assigned identifier to distinguish pen inputs)
  uint32 penId = 1;

  /* 2 = reserved for future use */

  /// Tablet device vendor ID (USB HID Vendor ID)
  uint32 vendorId = 3;
  /// Tablet device product ID (USB HID Product ID)
  uint32 productId = 4;

  /// Display name of the tablet device (e.g., "Nulltech Floatous Pro")
  optional string deviceName = 5;
}
```

### DESCRIPTION

- Every pen MUST send a `PenSetupEvent` upon connection to notify the server of its pen ID and device information.

## PenDestroyEvent

Message indicating that a pen connection has been terminated.

```protobuf
message PenDestroyEvent {
  /// Unique pen ID (MUST match the penId set in PenSetupEvent)
  uint32 penId = 1;
}
```

### DESCRIPTION

- `PenDestroyEvent` notifies the server that the pen is no longer in use. For example, the client may send this message when a pen is detached from a tablet or when the session ends.

## PenProximityEvent

An event that occurs when the pen enters or leaves the tablet's detection range.

```protobuf
message PenProximityEvent {
  /// Unique pen ID (MUST match the penId set in PenSetupEvent)
  uint32 penId = 1;

  // @constset: PenTipType
  uint32 tipType = 2;

  // @constset: PenProximityEventType
  uint32 eventType = 3;
}
```

### DESCRIPTION
- If `PenProximityEvent` is not sent, some systems or applications may treat pen input as mouse input.
  Therefore, it is RECOMMENDED to use this event to clearly indicate the pen's detection state.

## PenMoveEvent

```protobuf
message PenMoveEvent {
  CursorPositionScope scope = 1;
  CursorPositionPercent position = 2;

  PenContext context = 3;
}
```

### IMPLEMENTATION NOTES

- Always uses the absolute coordinate system, expressing positions as percentage-based coordinates in the range (0.0, 0.0) to (1.0, 1.0). The `scope` field specifies the reference frame for the coordinates.

## PenButtonEvent

```protobuf
message PenButtonEvent {
  uint32 penId = 1;

  // @constset: PenButtonEventType
  uint32 eventType = 2;
  uint32 button = 3;
}
```

### DESCRIPTION

Represents a pen button press/release event.
