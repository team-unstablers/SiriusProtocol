---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.hidio"
import:
  - "common-types.proto"
---

# HID KEYBOARD MESSAGES

Message definitions for keyboard input events and layout configuration.

---

# CONSTANTS

## KeyboardLayoutId

```constset
constset KeyboardLayoutId: uint32 {
  const ansi = 0;
  const iso = 1;
  const jis = 2;
  const ks = 3;

  /// A hint value indicating that keyboard events will be sent via a virtual keyboard
  /// (software keyboard on tablets/phones such as iOS, iPhone, etc.)
  const virtual = 15;
}
```

## KeyboardEventType

```constset
constset KeyboardEventType: uint32 {
  const unknown = 0;
  /// Key down event
  const keyDown = 1;
  /// Key up event
  const keyUp = 2;
  /// UCS-4 character input event. The keyCode field contains the UCS-4 codepoint.
  const ucs4 = 3;
}
```

### DESCRIPTION

Indicates the type of keyboard event. The `ucs4` type is used to directly transmit characters such as CJK that cannot be mapped to keycodes.

## KeyboardModifier

```optionset
optionset KeyboardModifier: uint32 {
  option none = 0x0000;
  option lShift = 0x0001;
  option rShift = 0x0002;
  option lCtrl = 0x0004;
  option rCtrl = 0x0008;
  option lAlt = 0x0010;
  option rAlt = 0x0020;
  option lMeta = 0x0040;
  option rMeta = 0x0080;
}
```

---

# STRUCTS

## KeyboardLayout

```protobuf
/// Keyboard layout definition
message KeyboardLayout {
  // @constset: KeyboardLayoutId
  uint32 layoutId = 1;

  /// Locale ID (MS-LCID specification)
  /// - e.g., 0x0409 = en-US, 0x0412 = ko-KR
  /// - Reference: https://learn.microsoft.com/en-us/openspecs/windows_protocols/ms-lcid/70feba9f-294e-491e-b6eb-56532684c37f
  uint32 localeId = 2;

  uint32 flags = 15;
}
```

## KeyboardHack

```protobuf
/// Keyboard hack definition
message KeyboardHack {
  /// Unique hack identifier
  string identifier = 1;
  /// Additional argument map
  map<string, string> args = 2;
}
```

### DESCRIPTION

A Keyboard Hack refers to a configuration that modifies or supplements specific keyboard behavior for user convenience.
Each hack has a unique `identifier`, and additional arguments can be passed via the `args` map as needed.

### IMPLEMENTATION NOTES

- The server activates hack plugins based on the hack list provided via `KeyboardSetupEvent`.
- Examples of defined hack identifiers:
  - `app.noctiluca.hidio.hack.general.capslock_as_ctrl`: Makes the Caps Lock key act as the Ctrl key
  - `app.noctiluca.hidio.hack.cjk.emulate_win32_ime_switch`: Emulates input method switching on the macOS host using the Alt + Shift key combination
  - `app.noctiluca.hidio.hack.cjk.emulate_win32_hangul_toggle`: Toggles Korean/English input on the macOS host when the Hangul key is pressed

## KeyboardSetupEvent

```protobuf
/// Keyboard device setup event
message KeyboardSetupEvent {
  /// List of keyboard layouts preferred by the client (in priority order)
  repeated KeyboardLayout preferredLayouts = 1;
  /// List of keyboard hacks to activate
  repeated KeyboardHack hacks = 2;

  uint32 flags = 15;
}
```

### DESCRIPTION

An event sent by the client to convey keyboard device settings to the server.
Includes the preferred keyboard layouts and the list of keyboard hacks to activate.

### IMPLEMENTATION NOTES

- If the server does not support the `layoutId` requested by the client, the server SHOULD use the ANSI layout as the default and send a `ServerNotice` (severity: WARNING) through the main channel.
- This event is transmitted inside an `HIDIOPacket`, not directly on the HIDIO channel.

---

# MESSAGES

## KeyboardEvent

```protobuf
/// Keyboard input event
message KeyboardEvent {
  // @constset: KeyboardEventType
  uint32 eventType = 1;

  uint32 scanCode = 2;
  uint32 keyCode = 3;

  // @optionset: KeyboardModifier
  uint32 modifiers = 4;

  uint32 flags = 5;
}
```

### DESCRIPTION

Represents a keyboard input event. The `eventType` field determines the type of event: key down, key up, UCS-4 character input, etc.

### IMPLEMENTATION NOTES

- `keyCode` follows the Linux keycode specification. The server converts it to the host system's keycode (Carbon keycode on macOS) before injection.
- When `eventType` is `ucs4`, the `keyCode` field contains a UCS-4 codepoint. The server processes this as a character input.
- `scanCode` is currently unused and MUST be set to 0.
