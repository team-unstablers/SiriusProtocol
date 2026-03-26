---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
  - "v1/channels/projection/projection_typedef.proto"
---

# PROJECTION SOURCE

Defines the sources for screen projection (displays, specific regions, single windows, etc.).

---

# CONSTANTS

## SingleWindowProjectionSourceFlags

```optionset
optionset SingleWindowProjectionSourceFlags: uint32 {
  /// Tracks changes to the source window (movement, resizing, etc.) and updates the projection area.
  option followSource = 1;
}
```

## ProjectionSourceFlags

```optionset
optionset ProjectionSourceFlags: uint32 {
  /// Shows the cursor on the host screen.
  /// NOTE: This flag MAY be ignored depending on the server implementation and policy configuration.
  option showCursor = 0x1;

  /// Shows a curtain (screen blanking) on the host's console screen.
  /// NOTE: This flag MAY be ignored depending on the server implementation and policy configuration.
  option showCurtain = 0x2;
}
```

---

# STRUCTS

## EntireDisplayProjectionSource

```protobuf
/// Used when projecting an entire display.
message EntireDisplayProjectionSource {
  /// Display ID.
  /// Setting to -1 refers to the default display.
  /// Setting to -2 refers to the entire display area.
  sint32 displayId = 1;
}
```

## DisplayRegionProjectionSource

```protobuf
/// Used when projecting a manually specified region within a specific display.
message DisplayRegionProjectionSource {
  /// Display ID.
  /// Setting to -1 refers to the default display.
  /// Setting to -2 refers to the entire display area.
  sint32 displayId = 1;

  /// Rectangular region to project (in pixels)
  SRRect region = 2;
}
```

## SingleWindowProjectionSource

```protobuf
/// Used when projecting a single window.
message SingleWindowProjectionSource {
  /// Unique ID (handle) of the window to project
  optional sint64 windowId = 1;

  // TODO: consider adding additional window identification methods such as process ID, window title, etc.

  /// Flags related to window projection
  // @optionset: SingleWindowProjectionSourceFlags
  uint32 flags = 16;
}
```

## ProjectionSource

```protobuf
/// Projection source definition (one of: display, region, or single window)
message ProjectionSource {
  oneof value {
    EntireDisplayProjectionSource entireDisplay = 1;
    DisplayRegionProjectionSource region = 2;
    SingleWindowProjectionSource singleWindow = 3;
  }

  /// Common flags for projection sources
  // @optionset: ProjectionSourceFlags
  uint32 flags = 16;
}
```

### DESCRIPTION

- The top-level structure that defines the source for projection.
- Selects one of `entireDisplay` (entire display), `region` (specific region within a display), or `singleWindow` (single window).
- `flags` specifies common projection source options such as cursor display and curtain display.
