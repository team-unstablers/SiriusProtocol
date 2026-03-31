---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.hidio"
import:
  - "common-types.proto"
  - "v1/channels/hidio/hid_mouse.proto"
---

# TOUCH MESSAGES

Message definitions for touch input.

---

# CONSTANTS

## TouchEventType

```constset
constset TouchEventType: uint32 {
  /// The touch point moved
  const move = 0;
  /// The touch point made contact
  const down = 1;
  /// The touch point was lifted
  const up = 2;
  /// The touch was cancelled by the system (e.g., palm rejection, system gesture)
  const cancel = 3;
}
```

---

# STRUCTS

## TouchEvent

```protobuf
message TouchEvent {
  /// Unique identifier for the touch point (consistent across events for the same touch)
  uint32 id = 1;

  /// Type of touch event (move, down, up, cancel)
  // @constset: TouchEventType
  uint32 type = 2;

  /// Target scope for the touch point
  CursorPositionScope scope = 3;
  /// Position of the touch point within the scope (0.0 to 1.0)
  CursorPositionPercent position = 4;

  /// Pressure of the touch (0.0 to 1.0)
  float pressure = 5;

  /// Major radius of the touch contact area in percent of the scope (0.0 to 1.0)
  float majorRadius = 6;
  /// Minor radius of the touch contact area in percent of the scope (0.0 to 1.0)
  float minorRadius = 7;
}
```

### DESCRIPTION

Represents a single touch point.
For multi-touch, the client SHOULD send multiple `TouchEvent` entries within a single `HIDEvent` batch in an `HIDIOPacket`.

### IMPLEMENTATION NOTES

- When the input device does not support pressure sensing, the client MUST send `1.0` as the pressure value.
- When the input device does not report contact radius, the client SHOULD send `0.0` for both `majorRadius` and `minorRadius`. The server MUST treat `0.0` radius values as "not available" and use a platform-appropriate default.