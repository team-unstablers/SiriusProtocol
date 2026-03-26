---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
  - "v1/channels/projection/projection_typedef.proto"
---

# PROJECTION DISPLAY MANAGEMENT

Message definitions for display management, including display list retrieval and subscription to display change events.

---

# CONSTANTS

## DisplayKind

```constset
constset DisplayKind: uint32 {
  /// Unknown display type.
  const unknown = 0;
  /// Internal display (e.g., built-in laptop display)
  const internal = 1;
  /// External display (e.g., monitor equipment)
  const external = 2;
  /// Virtual display (e.g., wireless display adapter)
  const virtual = 3;
}
```

## DisplayColorDepth

```constset
constset DisplayColorDepth: uint32 {
  /// Unknown color depth
  const unknown = 0;
  /// Grayscale color depth
  const grayscale = 1;
  /// Indexed (palette-defined) 16-color
  const indexed16Color = 2;
  /// Indexed (palette-defined) 256-color
  const indexed256Color = 3;
  /// 8-bit color (RGB8, 0–255)
  const bit8 = 4;
  /// 10-bit color (RGB10, 0–1023)
  const bit10 = 5;
}
```

## DisplayColorProfile

```constset
constset DisplayColorProfile: string {
  /// sRGB color profile
  const sRGB = "sRGB";
  /// Adobe RGB color profile
  const adobeRGB = "AdobeRGB";
  /// Display P3 color profile
  const displayP3 = "Display P3";
}
```

## DisplayDynamicRange

```constset
constset DisplayDynamicRange: uint32 {
  /// Standard Dynamic Range (SDR)
  const sdr = 0;
  /// High Dynamic Range (HDR)
  const hdr = 1;
}
```

## DisplayChangeEventType

```optionset
optionset DisplayChangeEventType: uint32 {
  /// A display was connected.
  option connected = 0x0001;
  /// A display was disconnected.
  option disconnected = 0x0002;
  /// The display's geometry, color information, etc. was modified.
  option modified = 0x0004;
  /// The display became the primary display.
  option becamePrimary = 0x0008;
}
```

## DisplayListRequestFlags

```optionset
optionset DisplayListRequestFlags: uint32 {
  /// Includes thumbnails of each display's contents. Consumes significant computing resources; do not abuse.
  /// This flag MAY be ignored depending on the server implementation.
  /// If an aggressive DRM solution that detects capture attempts and terminates the process is running,
  /// the server implementation MAY ignore this flag.
  option includeThumbnails = 0x0001;
}
```

---

# STRUCTS

## DisplayState

```protobuf
/// Current state information of a display
message DisplayState {
  /// Whether the display is the primary display
  bool isPrimary = 1;
  /// Whether the display is currently connected
  bool isConnected = 2;
  /// Whether the display is currently active (powered on)
  bool isActive = 3;
}
```

## DisplayPhysicalSizeInfo

```protobuf
/// Physical size information of a display
message DisplayPhysicalSizeInfo {
  /// Physical size of the display (in millimeters)
  SRSize physicalSize = 1;
  /// Pixel density of the display (DPI: Dots Per Inch)
  uint32 dpi = 2;
}
```

## DisplayInfo

```protobuf
/// Detailed information about a display
message DisplayInfo {
  /// Display identification ID assigned by the OS / DM.
  uint32 displayID = 1;

  /// Display type.
  // @constset: DisplayKind
  uint32 kind = 2;

  /// Display name of the monitor device.
  /// This value MAY be empty or non-standardized depending on the server implementation.
  string displayName = 3;

  /// Display state information
  DisplayState state = 4;

  /// Display resolution and position information
  /// x, y: top-left corner coordinates in the global virtual screen coordinate system
  /// width, height: display size in pixels
  SRRect bounds = 5;

  /// Display refresh rate (Hz)
  /// Expressed as float since values like 59.94 or 23.976 may occur.
  /// Depending on the server implementation, platform, or display type, this value
  /// may differ from the actual display's capabilities or may not be provided.
  /// (0, negative, or highly abnormal values may occur)
  float refreshRate = 6;

  /// Display color depth
  // @constset: DisplayColorDepth
  uint32 colorDepth = 7;

  /// Display dynamic range
  // @constset: DisplayDynamicRange
  uint32 dynamicRange = 8;

  /// Display color profile
  // @constset: DisplayColorProfile
  // Values not in the predefined profiles represent custom color profiles defined by the user or monitor.
  optional string colorProfile = 9;

  /// Physical size information of the display
  /// This value MAY not be provided in some cases.
  optional DisplayPhysicalSizeInfo physicalSizeInfo = 10;

  /// Display scale factor
  /// Some GUI environments (especially X11) may use inverse scaling (e.g., 0.5x).
  /// If this field is not set, Protobuf behavior will default it to 0.0.
  /// Therefore, implementations MUST treat 0.0 as equivalent to 1.0 (no scaling).
  /// It is RECOMMENDED to set this field explicitly whenever possible.
  float scaleFactor = 11;

  /// Display thumbnail
  /// This field is only provided when 'includeThumbnails' is set in the DisplayListRequest flags.
  /// Even if the flag is set, this field MAY be empty depending on server conditions.
  optional bytes thumbnail = 14;

  /// Additional metadata
  /// All metadata keys are optional. Not all server implementations are guaranteed to support them,
  /// and not all keys are always provided.
  /// Implementation-specific metadata keys SHOULD follow the reverse domain name format.
  /// (e.g., "com.contoso.superremote.virtual-display-purpose")
  /// @key "related-display-id": If this display is mirroring another display,
  ///       indicates the original display ID.
  /// @key "connection-type": Display connection type
  ///       (e.g., "HDMI", "DisplayPort", "USB-C", "Wireless")
  map<string, string> metadata = 15;

  uint32 flags = 16;
}
```

### IMPLEMENTATION NOTES

- When `scaleFactor` is 0.0, implementations MUST treat it as 1.0. The Swift implementation (`DisplayInfo.init(from:)`) also performs this conversion.
- When `colorProfile` is a value not defined in the `DisplayColorProfile` constset, it is treated as a custom color profile.

---

# MESSAGES

## DisplayListRequest

```protobuf
/// Requests a list of displays.
// @opcode: 0x8041
message DisplayListRequest {
  uint64 requestId = 1;

  // @optionset: DisplayListRequestFlags
  uint32 flags = 16; // reserved
}
```

### DESCRIPTION

Used by the client to request the list of displays connected to the server.
The server responds with `DisplayListResponse`.

## DisplayListResponse

```protobuf
/// Response to a display list request.
// @opcode: 0x8042
message DisplayListResponse {
  uint64 requestId = 1;
  repeated DisplayInfo displays = 2;
}
```

### DESCRIPTION

The response to a `DisplayListRequest`.
Contains detailed information about all displays currently connected to the server.

## SubscribeDisplayChangesRequest

```protobuf
/// Subscribes to display change events from the server.
// @opcode: 0x8043
message SubscribeDisplayChangesRequest {
  uint64 requestId = 1;
  /// When 0x0000, subscribes to all events.
  // @optionset: DisplayChangeEventType
  uint32 eventMask = 2;
  uint32 flags = 16; // reserved
}
```

### DESCRIPTION

Used by the client to subscribe to display change events.
Specific event types can be filtered via `eventMask`.

## SubscribeDisplayChangesResponse

```protobuf
/// Response to a SubscribeDisplayChangesRequest.
// @opcode: 0x8044
message SubscribeDisplayChangesResponse {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
}
```

### DESCRIPTION

The response to a `SubscribeDisplayChangesRequest`.
The returned `subscriptionId` is used for subsequent unsubscription.

## UnsubscribeDisplayChangesRequest

```protobuf
/// Unsubscribes from display change events from the server.
// @opcode: 0x8045
message UnsubscribeDisplayChangesRequest {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
}
```

### DESCRIPTION

Used by the client to cancel an existing display change event subscription.

## UnsubscribeDisplayChangesResponse

```protobuf
/// Response to an UnsubscribeDisplayChangesRequest.
// @opcode: 0x8046
message UnsubscribeDisplayChangesResponse {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
  /// Whether the unsubscription was successful.
  bool isSuccess = 3;
}
```

### DESCRIPTION

The response to an `UnsubscribeDisplayChangesRequest`.

## DisplayChangedEvent

```protobuf
/// Display change event.
// @opcode: 0x8047
message DisplayChangedEvent {
  /// Event type code
  // @optionset: DisplayChangeEventType
  uint32 eventType = 1;
  /// Information about the changed display
  DisplayInfo display = 2;
}
```

### DESCRIPTION

Sent by the server to subscribed clients to notify them of display state changes.
Includes events such as display connection/disconnection, configuration changes, and primary display changes.
