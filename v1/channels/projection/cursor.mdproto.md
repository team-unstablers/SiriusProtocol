---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
  - "v1/channels/projection/projection_typedef.proto"
---

# PROJECTION CURSOR

Message definitions for cursor (mouse pointer) event subscription and delivery.
Clients can subscribe to cursor events to synchronize the cursor position and image from the remote host.

---

# CONSTANTS

## SubscribeCursorEventsFlags

```optionset
optionset SubscribeCursorEventsFlags: uint32 {
  /// Informs the server that the client will not cache cursor images.
  /// When this flag is set, the server implementation MUST always send cursor image data.
  option disableImageCache = 0x1;
}
```

---

# STRUCTS

## CursorMoveEvent

```protobuf
message CursorMoveEvent {
  /// The display ID where the cursor is currently located.
  /// Special values:
  ///   - `-1` means 'default display' in other messages, but MUST NOT be used here (assertion).
  ///   - `-2` means the entire display area (the full viewport of the graphics session).
  uint32 displayID = 1;

  /// The new cursor position (in pixels).
  SRPoint position = 2;
}
```

### DESCRIPTION

- Indicates that the mouse pointer position has changed.
- The server sends this event to subscribed clients to synchronize the current cursor position.

### IMPLEMENTATION NOTES

- Server implementations SHOULD consider throttling or buffering this event before sending. Excessively frequent cursor move events MAY waste network bandwidth.

## CursorImageEvent

```protobuf
message CursorImageEvent {
  /// Cursor type identifier.
  /// This value MAY differ across server implementations and operating systems. Use it only for cursor image caching.
  uint64 cursorType = 1;

  /// MIME type of the cursor image (e.g., "image/png").
  string mimeType = 2;

  /// Size of the cursor image.
  SRSize size = 3;

  /// The click point (hotspot) within the cursor image (in pixels, relative to the top-left corner of the image).
  SRPoint hotspot = 4;

  /// Cursor image data (binary).
  /// Only sent once for each cursor type, unless the `disableImageCache` flag is set.
  optional bytes imageData = 5;
}
```

### IMPLEMENTATION NOTES

- Cursor image data SHOULD be kept under 16 KB whenever possible. Excessively large images MAY cause transmission delays.

---

# MESSAGES

## SubscribeCursorEventsRequest

```protobuf
/// Requests subscription to cursor events.
// @opcode: 0x80A1
message SubscribeCursorEventsRequest {
  uint64 requestId = 1;

  // @optionset: SubscribeCursorEventsFlags
  uint32 flags = 16;
}
```

### DESCRIPTION

- Used by the client to subscribe to cursor events from the server.
- When the server receives this request, it MUST respond with a `SubscribeCursorEventsResponse` containing a subscription identifier.

## SubscribeCursorEventsResponse

```protobuf
/// Response to a cursor event subscription request.
// @opcode: 0x80A2
message SubscribeCursorEventsResponse {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
}
```

### DESCRIPTION

- Sent by the server in response to a `SubscribeCursorEventsRequest`.
- The returned `subscriptionId` is used when unsubscribing from cursor events.

## UnsubscribeCursorEventsRequest

```protobuf
/// Requests unsubscription from cursor events.
// @opcode: 0x80A3
message UnsubscribeCursorEventsRequest {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
}
```

### DESCRIPTION

- Used by the client to cancel an existing cursor event subscription.

## UnsubscribeCursorEventsResponse

```protobuf
/// Response to a cursor event unsubscription request.
// @opcode: 0x80A4
message UnsubscribeCursorEventsResponse {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
  /// Whether the unsubscription was successful.
  bool isSuccess = 3;
}
```

### DESCRIPTION

- Sent by the server in response to an `UnsubscribeCursorEventsRequest`.

## CursorEvent

```protobuf
/// Container message for cursor events.
// @opcode: 0x80A5
message CursorEvent {
  oneof event {
    CursorMoveEvent moveEvent = 1;
    CursorImageEvent imageEvent = 2;
  }
}
```

### DESCRIPTION

- Sent by the server to subscribed clients as a cursor event container.
- Contains either a cursor move event (`CursorMoveEvent`) or a cursor image change event (`CursorImageEvent`).
