---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
  - "v1/channels/projection/projection_typedef.proto"
---

# PROJECTION CODEC DEFINITIONS

Codec and quality setting definitions for video projection.

---

# CONSTANTS

## CodecFourCC

```constset
constset CodecFourCC: uint32 {
  /// 'AVC1': Advanced Video Coding (H.264), MPEG-4 Part 10
  const avc1 = 0x41564331;

  /// 'HVC1': High Efficiency Video Coding (H.265), MPEG-H Part 2
  const hvc1 = 0x48564331;

  /// 'ZRLE': ZRLE (Zlib Run-Length Encoding), RLE + Zstd
  const zrle = 0x5A524C45;

  /// 'MJPG': Motion JPEG
  const mjpg = 0x4D4A5047;

  /// 'WEBP': WebP
  const webp = 0x57454250;

  /// 'VP80': VP8
  const vp80 = 0x56503830;
}
```

## AutoQualityMode

```constset
constset AutoQualityMode: uint32 {
  /// Balances quality and performance.
  const balancedPriority = 0;

  /// Prioritizes the highest possible quality.
  const qualityPriority = 1;

  /// Prioritizes the highest possible performance.
  const performancePriority = 2;
}
```

## LosslessQualityMode

```constset
constset LosslessQualityMode: uint32 {
  /// Balances performance and compression ratio.
  const balancedPriority = 0;

  /// Prioritizes performance. (compression ratio may decrease)
  const speedPriority = 1;

  /// Prioritizes compression ratio. (performance may decrease)
  const compressionPriority = 2;
}
```

## CodecOptionKey

```constset
constset CodecOptionKey: string {
  const colorFormat = "color-format";
  const hardwareAcceleration = "hardware-acceleration";
  const profile = "profile";
  const level = "level";
  const colorDepth = "color-depth";
  const dynamicRange = "dynamic-range";
  const colorRange = "color-range";
  const displayDensity = "display-density";
  const compressionLevel = "compression-level";
  const tileSize = "tile-size";
  const quantizeLevel = "quantize-level";
}
```

### DESCRIPTION

- Defines the keys for per-codec additional options.
- Serialized as a string in the format `"key: 'value'; key2: 'value2'"` in the `Codec.options` field.
- Appending the `!required` suffix marks the option as mandatory; both sides MUST support it during codec negotiation.

## CodecOptionValue

```constset
constset CodecOptionValue: string {
  const kColorFormatAuto = "auto";
  const kColorFormatYUV420 = "yuv420";
  const kColorFormatYUV444 = "yuv444";
  const kColorFormatRGB888 = "rgb888";
  const kColorFormatRGB565 = "rgb565";

  const kHardwareAccelerationAuto = "auto";
  const kHardwareAccelerationFalse = "false";

  const kProfileAuto = "auto";
  const kProfileH264High10 = "h264_high10";
  const kProfileH264High = "h264_high";
  const kProfileH264Main = "h264_main";
  const kProfileH264Baseline = "h264_baseline";
  const kProfileHEVCMain = "hevc_main";
  const kProfileHEVCMain10 = "hevc_main10";

  const kColorDepth8Bit = "8bit";
  const kColorDepth10Bit = "10bit";

  const kColorRangeLimited = "limited";
  const kColorRangeFull = "full";

  const kDisplayDensityAuto = "auto";
  const kDisplayDensityPerformance = "performance";
  const kDisplayDensityBest = "best";

  const kDynamicRangeSDR = "sdr";
  const kDynamicRangeHDR = "hdr";

  const kQuantizeLevelNone = "0";
  const kQuantizeLevel1 = "1";
  const kQuantizeLevel2 = "2";
  const kQuantizeLevel3 = "3";
  const kQuantizeLevel4 = "4";
  const kQuantizeLevel5 = "5";

  const kTileSize64x64 = "64";
  const kTileSize128x128 = "128";
  const kTileSize256x256 = "256";
}
```

### DESCRIPTION

- Defines value constants corresponding to `CodecOptionKey`.
- The range of allowed values differs per key, and the prefix (`kColorFormat`, `kProfile`, etc.) identifies the associated key.

### IMPLEMENTATION NOTES

- `// TODO:` MDProto should support partial constset definitions. Currently, all key values are mixed in a single constset.

---

# STRUCTS

## AutoQuality

```protobuf
/// Auto quality mode (server automatically adjusts based on network conditions)
message AutoQuality {
  // @constset: AutoQualityMode
  uint32 mode = 1;
}
```

## ConstantBitrateQuality

```protobuf
/// Constant bitrate (CBR) quality setting
message ConstantBitrateQuality {
  /// Bitrate (kbps)
  int32 bitrateKbps = 1;
}
```

## VariableBitrateQuality

```protobuf
/// Variable bitrate (VBR) quality setting
message VariableBitrateQuality {
  /// Maximum bitrate (kbps)
  int32 maxBitrateKbps = 1;
  /// Target bitrate (kbps)
  int32 targetBitrateKbps = 2;
}
```

## FixedQuality

```protobuf
/// Fixed quality (CQP/CRF) setting
/// Specifies quality as a value from 0 to 100.
message FixedQuality {
  int32 quality = 1;
}
```

### IMPLEMENTATION NOTES

- A lower value may indicate higher quality or vice versa, but generally 100 represents the highest quality. The actual behavior is implementation-dependent.

## LosslessQuality

```protobuf
/// Lossless quality setting
/// Transmits data without quality degradation using lossless compression.
message LosslessQuality {
  // @constset: LosslessQualityMode
  uint32 mode = 1;
}
```

## Codec

```protobuf
/// Video codec settings
message Codec {
  /// FourCC code value (Big-Endian)
  // @constset: CodecFourCC
  fixed32 fourCC = 1;

  /// Codec quality setting
  oneof quality {
    ConstantBitrateQuality constantBitrate = 2;
    VariableBitrateQuality variableBitrate = 3;
    FixedQuality fixedQuality = 4;
    LosslessQuality lossless = 5;
    AutoQuality auto = 6;
  }

  /// Frames per second. Setting the value to -1 follows the source (monitor) frame rate.
  optional float frameRate = 9;

  /// Resolution
  SRSize size = 10;

  /// Per-codec additional option string (e.g., "color-format: 'YUV444'; hardware-acceleration: 'true'; profile: 'high';")
  optional string options = 15;
}
```

### DESCRIPTION

- A structure that comprehensively defines the codec, quality, resolution, frame rate, and other settings for video projection.
- The client passes an array of this structure in `ProjectionRequest.preferredCodecs`, and the server determines the final codec through codec negotiation.

### IMPLEMENTATION NOTES

- The `fourCC` field MUST be encoded in big-endian byte order.
- Quality settings other than `ConstantBitrateQuality` and `VariableBitrateQuality` are optional implementation details. Server/client implementations MAY not support those quality settings.
- The `options` field follows the serialization format `"key: 'value'; key2: 'value2'"`, with the `!required` suffix distinguishing mandatory from optional options.
