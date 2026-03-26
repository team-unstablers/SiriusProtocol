---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
  - "v1/channels/projection/projection_typedef.proto"
---

# PROJECTION WINDOW MANAGEMENT (QUERY)

Message definitions for the read (query) side of window management,
including window list retrieval, window info/icon/thumbnail queries, and window change event subscriptions.

A `WindowFilter` can be used to filter windows matching specific conditions:
- `(windowID IS 1234567890) OR (pid IS 9876)`
- `(application ICONTAINS "Google Chrome") OR (application ICONTAINS "chrome.exe")`

---

# CONSTANTS

## WindowInfoFlags

```optionset
optionset WindowInfoFlags: uint32 {
  /// The window currently has focus.
  option isFocused = 0x0001;
  /// The window is configured to be hidden from the user.
  option isHidden = 0x0002;
  /// The window has no window decorations (e.g., title bar).
  option noWindowDecoration = 0x0004;
  /// The window does not appear in the taskbar (Windows, Linux) or Dock (macOS).
  option skipWindowEntry = 0x0008;
  /// The window is minimized.
  option isMinimized = 0x0010;
  /// The window is maximized.
  option isMaximized = 0x0020;
  /// The window is displayed in fullscreen mode.
  option isFullscreen = 0x0040;
  /// The window cannot be minimized.
  option cannotMinimize = 0x0100;
  /// The window cannot be maximized.
  option cannotMaximize = 0x0200;
  /// The window cannot enter fullscreen mode.
  option cannotFullscreen = 0x0400;
  /// The window is in an always-on-top (topmost) state.
  option isTopmost = 0x1000;
  /// The window is in an always-on-bottom (bottommost) state.
  option isBottommost = 0x2000;
  /// Linux: The window bypasses the compositor.
  /// Windows: The window bypasses DWM (Desktop Window Manager),
  /// meaning Aero Glass effects and similar are not applied.
  /// This may be set on applications with known compatibility issues
  /// or those requiring low-latency rendering.
  option skipCompositor = 0x00010000;
}
```

## WindowRole

```constset
constset WindowRole: uint32 {
  const unknown = 0;
  /// Normal window
  const normal = 1;
  /// Dialog window
  const dialog = 2;
  /// Tooltip window
  const tooltip = 3;
  /// Menu
  const menu = 4;
  /// Notification window
  const notification = 5;
  // TODO: Additional window roles may need to be defined (e.g., popup, splash, toolbar)
}
```

## WindowHint

```optionset
optionset WindowHint: uint32 {
  /// The window is part of the system UI.
  /// Examples: navigation bar, status bar, notification center, IME panel, etc.
  option systemUI = 0x0001;
  /// The window has been reported as not responding by the system.
  option notResponding = 0x0002;
  /// The window is expected to be inaccessible due to system policy.
  /// (e.g., windows from another user session, windows of elevated processes, UAC, etc.)
  option inaccessible = 0x0004;
  /// The window is expected to be uncapturable due to system policy (e.g., DRM-protected content).
  /// This hint does NOT guarantee that capture is impossible.
  /// Capture may still occur, but the result may appear as a black screen.
  option protectedContent = 0x0008;
  /// The window appears to be aggressively protected by a third-party DRM solution,
  /// anti-cheat solution, or similar.
  /// Attempting to capture or remotely control this window may cause the server
  /// implementation's process to crash or be forcefully terminated,
  /// so projection is NOT recommended.
  /// (e.g., AhnLab Safe Transaction, Fasoo DRM, etc.)
  option aggressiveProtectedContent = 0x0010;
  /// The window is currently not visible on screen (e.g., occluded by other windows).
  option notVisibleOnScreen = 0x0020;
  /// The window contains static content that does not change frequently.
  option staticContent = 0x0100;
  /// The window contains rapidly updating dynamic content (e.g., video player, game).
  option dynamicContent = 0x0200;
  /// The window contains a shadow.
  /// Useful when the window has an outer shadow region.
  /// The client may choose to draw the shadow on its own instead.
  option hasShadow = 0x1000;
  /// The window contains transparent regions.
  /// Useful for windows with border-radius or partially transparent UI elements.
  option hasTransparency = 0x2000;
}
```

## WindowFilterExpressionOperator

```constset
constset WindowFilterExpressionOperator: uint32 {
  const matchExact = 0;
  const matchContains = 1;
  const matchIContains = 2;
  /// The REGEX implementation SHOULD use Perl-compatible regular expressions (PCRE),
  /// but this is not a strict requirement.
  const matchRegex = 3;
}
```

## WindowFilterOperator

```constset
constset WindowFilterOperator: uint32 {
  /// All conditions must be satisfied
  const and = 0;
  /// At least one condition must be satisfied
  const or = 1;
}
```

## WindowListRequestFlags

```optionset
optionset WindowListRequestFlags: uint32 {
  /// Requests paginated results.
  /// The page size MAY vary depending on the server implementation.
  option paginated = 0x0001;
  /// Includes windows that are normally hidden.
  option includeHiddenWindows = 0x0002;
}
```

## WindowChangeEventType

```optionset
optionset WindowChangeEventType: uint32 {
  /// The window gained focus.
  option focused = 0x0001;
  /// The window lost focus.
  option unfocused = 0x0002;
  /// The window was moved.
  option moved = 0x0004;
  /// The window was resized.
  option resized = 0x0008;
  /// The window was closed.
  option closed = 0x0010;
  /// The window's metadata changed significantly.
  option metadataChanged = 0x0020;
}
```

## WindowEventSubscriptionFlags

```optionset
optionset WindowEventSubscriptionFlags: uint32 {
  /// Immediately sends an initial state snapshot for the subscribed window events.
  option sendInitialSnapshot = 0x0001;
  /// Attempts to include only minimal data.
  /// For all events except WINDOW_CHANGE_EVENT_TYPE_METADATA_CHANGED,
  /// irrelevant WindowInfo fields SHOULD be omitted or excluded entirely.
  /// When this flag is not set, rich WindowInfo data is included in all events.
  option minimalData = 0x0002;
}
```

---

# STRUCTS

## WindowFilterExpression

```protobuf
/// A single window filter condition
message WindowFilterExpression {
  // @constset: WindowFilterExpressionOperator
  uint32 operator = 1;
  bool invert = 2;

  oneof field {
    /// Matches by window handle ID.
    /// Uses uint64 because window handle ID sizes may vary across OS / display managers.
    uint64 windowId = 3;
    /// Matches by process ID.
    uint64 pid = 4;
    /// Matches by window title.
    string windowTitle = 5;
    /// Matches by application name (executable name).
    /// Platform differences:
    /// - Windows: executable file name (e.g., "chrome.exe")
    /// - macOS / Linux: process name (ARGV[0]). May be a full path
    ///   or a simple executable name.
    string applicationName = 6;
    /// Matches by bundle ID.
    /// The meaning MAY vary depending on the server implementation.
    /// (e.g., macOS "com.apple.Safari", Android "com.google.android.youtube")
    string applicationBundleId = 7;
    /// Matches by window class name.
    /// macOS does not support window class names, so this field MAY be ignored.
    string windowClass = 8;
  }
}
```

## WindowFilter

```protobuf
/// Window filter (recursive tree structure)
message WindowFilter {
  // @constset: WindowFilterOperator
  uint32 operator = 1;

  /// List of conditions (compound conditions can be expressed via recursive structure)
  /// Example: (A OR B) AND C
  repeated WindowFilter expressions = 2;

  /// Single condition (leaf node)
  optional WindowFilterExpression expression = 3;
}
```

## WindowInfo

```protobuf
/// Detailed information about a window
message WindowInfo {
  uint64 windowId = 1;
  uint64 pid = 2;

  string windowTitle = 3;
  string applicationName = 4;
  string applicationBundleId = 5;
  string windowClass = 6;

  // @constset: WindowRole
  uint32 role = 7;

  SRRect bounds = 8;

  /// Icon hash for this window.
  optional fixed64 iconHash = 9;

  optional uint64 parentWindowId = 10;

  /// Thumbnail image for this window (PNG8 with alpha channel, aspect ratio preserved, max 512x512)
  optional bytes thumbnail = 12;

  /// Additional metadata.
  /// All metadata keys are optional. Not all server implementations are guaranteed
  /// to support them, and not all keys are always provided.
  /// Implementation-specific metadata keys SHOULD follow the reverse domain name format.
  /// (e.g., "com.contoso.superremote.virtual-display-purpose")
  map<string, string> metadata = 13;

  /// Window hints
  // @optionset: WindowHint
  uint32 hints = 14;
  /// Window state flags
  // @optionset: WindowInfoFlags
  uint32 flags = 15;
}
```

---

# MESSAGES

## WindowListRequest

```protobuf
/// Requests a list of windows.
// @opcode: 0x8061
message WindowListRequest {
  uint64 requestId = 1;

  /// Filter conditions
  optional WindowFilter filter = 2;

  /// Window list request flags
  // @optionset: WindowListRequestFlags
  uint32 flags = 15;
}
```

### DESCRIPTION

- Used by the client to request a window list from the server.
- When `filter` is specified, only windows matching the conditions are returned.
- The server MUST respond with a `WindowListResponse`.

## WindowListResponse

```protobuf
/// Response to a window list request.
// @opcode: 0x8062
message WindowListResponse {
  uint64 requestId = 1;
  repeated WindowInfo windows = 2;

  /// Whether this is the last page of paginated results
  bool isLastPage = 3;
}
```

### DESCRIPTION

- Response message to a `WindowListRequest`.
- When the `paginated` flag is set, `isLastPage` indicates whether additional pages exist.

## GetWindowInfoRequest

```protobuf
/// Requests detailed information about a specific window.
// @opcode: 0x8063
message GetWindowInfoRequest {
  uint64 requestId = 1;
  uint64 windowId = 2;
}
```

### DESCRIPTION

- Used by the client to request detailed information about a specific window.
- The server MUST respond with a `GetWindowInfoResponse`.

## GetWindowInfoResponse

```protobuf
/// Response to a window info request.
// @opcode: 0x8064
message GetWindowInfoResponse {
  uint64 requestId = 1;
  optional WindowInfo info = 2;
}
```

### DESCRIPTION

- Response message to a `GetWindowInfoRequest`.
- If the requested window does not exist, `info` MAY be absent.

## GetWindowIconRequest

```protobuf
/// Requests the icon of a specific window.
// @opcode: 0x8065
message GetWindowIconRequest {
  uint64 requestId = 1;
  uint64 windowId = 2;
}
```

### DESCRIPTION

- Used by the client to request the icon image of a specific window.

## GetWindowIconResponse

```protobuf
/// Response to a window icon request.
// @opcode: 0x8066
message GetWindowIconResponse {
  uint64 requestId = 1;
  uint64 windowId = 2;
  /// Icon image for this window (PNG8 with alpha channel, square, max 256x256)
  optional bytes icon = 3;
}
```

### DESCRIPTION

- Response message to a `GetWindowIconRequest`.
- If the window has no icon, `icon` MAY be absent.

## GetWindowThumbnailRequest

```protobuf
/// Requests the thumbnail of a specific window.
// @opcode: 0x8067
message GetWindowThumbnailRequest {
  uint64 requestId = 1;
  uint64 windowId = 2;
}
```

### DESCRIPTION

- Used by the client to request the thumbnail image of a specific window.

## GetWindowThumbnailResponse

```protobuf
/// Response to a window thumbnail request.
// @opcode: 0x8068
message GetWindowThumbnailResponse {
  uint64 requestId = 1;
  uint64 windowId = 2;
  /// Thumbnail image for this window (PNG8 with alpha channel, aspect ratio preserved, max 512x512)
  optional bytes thumbnail = 3;
}
```

### DESCRIPTION

- Response message to a `GetWindowThumbnailRequest`.
- If no thumbnail is available, `thumbnail` MAY be absent.

## SubscribeWindowEventsRequest

```protobuf
/// Subscribes to window change events.
// @opcode: 0x8069
message SubscribeWindowEventsRequest {
  uint64 requestId = 1;
  /// When set to 0x0000, subscribes to all event types.
  // @optionset: WindowChangeEventType
  uint32 eventMask = 2;
  /// When absent, subscribes to events for all windows.
  optional WindowFilter filter = 3;

  // @optionset: WindowEventSubscriptionFlags
  uint32 flags = 16;
}
```

### DESCRIPTION

- Used by the client to subscribe to window change events.
- `eventMask` can be used to filter specific event types, and `filter` can be used to target specific windows.

## SubscribeWindowEventsResponse

```protobuf
/// Response to a window event subscription request.
// @opcode: 0x806A
message SubscribeWindowEventsResponse {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
}
```

### DESCRIPTION

- Response message to a `SubscribeWindowEventsRequest`.
- The returned `subscriptionId` is used to unsubscribe later.

## UnsubscribeWindowEventsRequest

```protobuf
/// Unsubscribes from window change events.
// @opcode: 0x806B
message UnsubscribeWindowEventsRequest {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
}
```

### DESCRIPTION

- Used by the client to cancel an existing window event subscription.

## UnsubscribeWindowEventsResponse

```protobuf
/// Response to a window event unsubscription request.
// @opcode: 0x806C
message UnsubscribeWindowEventsResponse {
  uint64 requestId = 1;
  /// Subscription identifier.
  SRUUID subscriptionId = 2;
  /// Whether the unsubscription was successful.
  bool isSuccess = 3;
}
```

### DESCRIPTION

- Response message to an `UnsubscribeWindowEventsRequest`.

## WindowChangedEvent

```protobuf
/// Window change event notification.
// @opcode: 0x806D
message WindowChangedEvent {
  /// Event type code
  // @optionset: WindowChangeEventType
  uint32 eventType = 1;
  uint64 windowId = 2;

  /// Information about the changed window
  optional WindowInfo info = 3;
}
```

### DESCRIPTION

- Sent by the server to subscribed clients when a window state change occurs.
- Covers events such as focus change, move, resize, close, and metadata change.

### IMPLEMENTATION NOTES

- When `WindowEventSubscriptionFlags.minimalData` is set, `info` includes only minimal fields for all events except `metadataChanged`.
