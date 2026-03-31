---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.hidio"
import:
  - "common-types.proto"
  - "v1/channels/hidio/hid_keyboard.proto"
  - "v1/channels/hidio/hid_mouse.proto"
  - "v1/channels/hidio/hid_pen.proto"
  - "v1/channels/hidio/hid_raw.proto"
  - "v1/channels/hidio/hid_touch.proto"
---

# HIDIO CHANNEL MESSAGES

The HIDIO channel is used to transmit HID (Human Interface Device) input events.
It captures events from input devices such as keyboards and mice and transmits them to the remote system.

---

# STRUCTS

## HIDEvent

```protobuf
message HIDEvent {
  oneof event {
    RawEvent rawEvent = 3;

    KeyboardSetupEvent keyboardSetupEvent = 4;
    KeyboardEvent keyboardEvent = 5;

    MouseMoveEvent mouseMoveEvent = 6;
    MouseButtonEvent mouseButtonEvent = 7;
    MouseWheelEvent mouseWheelEvent = 8;

    PenSetupEvent penSetupEvent = 17;
    PenDestroyEvent penDestroyEvent = 18;
    PenProximityEvent penProximityEvent = 19;
    PenMoveEvent penMoveEvent = 20;
    PenButtonEvent penButtonEvent = 21;

    TouchEvent touchEvent = 22;
  }
}
```

### DESCRIPTION

A container type that can hold various HID events.
The `oneof event` field selectively includes one of keyboard, mouse, pen, touch, or raw events.

---

# MESSAGES

## HIDIOPacket

```protobuf
/// HID event packet transmitted through the HIDIO channel.
// @opcode: 0x8001
message HIDIOPacket {
  /// Packet sequence number
  uint64 sequenceNumber = 1;
  /// Timestamp of when the event occurred (Unix timestamp in milliseconds)
  uint64 timestamp = 2;

  /// List of HID events
  repeated HIDEvent events = 3;
}
```

### DESCRIPTION

The sole protocol message of the HIDIO channel, which bundles one or more HID events for transmission.
Sent from the client to the server, which then injects the received events into the host system.

### IMPLEMENTATION NOTES

- Applications SHOULD batch multiple events into a single packet whenever possible.
- `sequenceNumber` MUST be set to a monotonically increasing value on the client side.
- `timestamp` is the Unix timestamp (in milliseconds) of when the event occurred.
- The HIDIO channel is unidirectional from client to server only. The server MUST NOT send messages through this channel.
