---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
---

# PROJECTION AUDIO MESSAGES

Message definitions for audio projection (audio streaming and control).
Follows a session model similar to video projection, supporting various audio sources and codec configurations.

---

# CONSTANTS

## AudioFourCC

```constset
constset AudioFourCC: uint32 {
  /// Opus Audio Codec
  const opus = 0x4F505553;
  /// G.711 mu-law
  const pcmu = 0x50434D55;
  /// G.711 A-law
  const pcma = 0x50434D41;
}
```

## AudioSessionFailureReason

```constset
constset AudioSessionFailureReason: uint32 {
  const unknown = 0;
  /// The requested audio source was not found
  const sourceNotFound = 1;
  /// The audio codec is not supported
  const codecNotSupported = 2;
  /// Audio capture permission was denied
  const permissionDenied = 3;
}
```

## AudioSessionChangeReason

```constset
constset AudioSessionChangeReason: uint32 {
  const unknown = 0;
  /// The audio source was changed
  const sourceChanged = 1;
  /// The codec settings were renegotiated
  const codecRenegotiated = 2;
}
```

## AudioSessionEndReason

```constset
constset AudioSessionEndReason: uint32 {
  const unknown = 0;
  /// Ended by client request
  const clientRequested = 1;
  /// The audio source is no longer available
  const sourceUnavailable = 2;
  /// Ended due to an error
  const error = 3;
}
```

---

# STRUCTS

## Audio Codec Quality Types

```protobuf
/// CBR (Constant Bit Rate) mode setting
message AudioConstantBitrateQuality {
  /// Fixed bitrate (kbps)
  uint32 bitrateKbps = 1;
}

/// VBR (Variable Bit Rate) mode setting
message AudioVariableBitrateQuality {
  /// Maximum bitrate (kbps)
  uint32 maxBitrateKbps = 1;
  /// Target bitrate (kbps)
  uint32 targetBitrateKbps = 2;
}

/// Auto quality mode (server automatically adjusts based on network conditions)
message AudioAutoQuality {
  // Reserved for future extension
}
```

## AudioCodec

```protobuf
/// Audio codec and quality settings
message AudioCodec {
  /// FourCC code for codec identification
  // @constset: AudioFourCC
  fixed32 fourCC = 1;

  /// Quality setting mode
  oneof quality {
    AudioConstantBitrateQuality constantBitrate = 2;
    AudioVariableBitrateQuality variableBitrate = 3;
    AudioAutoQuality auto = 4;
  }

  /// Sampling rate (Hz). 0 means use default.
  uint32 sampleRate = 5;
  /// Number of channels. 0 means use default.
  uint32 channelCount = 6;

  /// Per-codec additional options (e.g., "frame-size: 20ms; stream-type: voice;")
  optional string options = 15;
}
```

### IMPLEMENTATION NOTES

- When `sampleRate` or `channelCount` is 0, implementations MUST use the stream or codec default value.
- The `options` field is parsed by `CodecOptionsParser` and is a semicolon-delimited string in `key: value;` format.

## Audio Source Types

```protobuf
/// Session-wide audio loopback source
message SessionAudioSource {
}

/// Audio source for a specific application
message ApplicationAudioSource {
  oneof identifier {
    /// Process ID
    uint64 pid = 1;
    /// App bundle ID (macOS)
    string bundleId = 2;
  }
}

/// Microphone input source
message MicrophoneAudioSource {
  /// Specific microphone device ID (uses default device if empty)
  optional string deviceId = 1;
}

/// Audio projection source definition
message AudioSource {
  oneof value {
    SessionAudioSource sessionAudio = 1;
    ApplicationAudioSource applicationAudio = 2;
    MicrophoneAudioSource microphone = 3;
  }
}
```

---

# MESSAGES

## AudioProjectionRequest

```protobuf
/// Requests the start of audio projection.
// @opcode: 0x80C1
message AudioProjectionRequest {
  /// Session identifier
  SRUUID identifier = 1;

  /// Audio source to project
  AudioSource source = 2;
  /// List of preferred codecs (in priority order)
  repeated AudioCodec preferredCodecs = 3;
}
```

### DESCRIPTION

Used by the client to request the start of audio projection from the server.
`preferredCodecs` is the list of codecs preferred by the client; the server selects a supported codec from this list.

## StopAudioProjectionRequest

```protobuf
/// Requests stopping audio projection.
// @opcode: 0x80C2
message StopAudioProjectionRequest {
  /// Identifier of the session to stop
  SRUUID identifier = 1;
}
```

### DESCRIPTION

Used by the client to request the stopping of an ongoing audio projection session.

## AudioSessionCreatedEvent

```protobuf
/// Event emitted when an audio session is successfully created.
// @opcode: 0x80C3
message AudioSessionCreatedEvent {
  SRUUID identifier = 1;
  AudioSource source = 2;
  AudioCodec codec = 3;
}
```

### DESCRIPTION

Sent by the server to the client when an audio projection session has been successfully created.
The `codec` field contains the codec settings actually selected by the server.

## AudioSessionCreationFailedEvent

```protobuf
/// Event emitted when audio session creation fails.
// @opcode: 0x80C4
message AudioSessionCreationFailedEvent {
  SRUUID identifier = 1;
  /// Failure reason
  // @constset: AudioSessionFailureReason
  uint32 reason = 2;
  /// Detailed error message
  optional string message = 3;
}
```

### DESCRIPTION

Sent by the server to the client when audio projection session creation has failed.
The failure cause can be identified through the `reason` field.

## AudioSessionChangedEvent

```protobuf
/// Event emitted when audio session settings are changed.
// @opcode: 0x80C5
message AudioSessionChangedEvent {
  SRUUID identifier = 1;
  /// Change reason
  // @constset: AudioSessionChangeReason
  uint32 reason = 2;
  optional AudioSource source = 3;
  optional AudioCodec codec = 4;
}
```

### DESCRIPTION

Sent by the server to the client when the settings of an ongoing audio session have changed.
May occur due to reasons such as audio source changes or codec renegotiation.

## AudioSessionEndedEvent

```protobuf
/// Event emitted when an audio session has ended.
// @opcode: 0x80C6
message AudioSessionEndedEvent {
  SRUUID identifier = 1;
  /// End reason
  // @constset: AudioSessionEndReason
  uint32 reason = 2;
  optional string message = 3;
}
```

### DESCRIPTION

Sent by the server to the client when an audio session has ended.
May be ended for various reasons including client request, source unavailability, or errors.
