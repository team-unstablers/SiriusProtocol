---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.projection"
import:
  - "common-types.proto"
---

# PROJECTION TYPEDEFS

Common data structure definitions shared across projection-related messages.

---

# STRUCTS

## SRRect

```protobuf
/// Defines a rectangular area.
message SRRect {
  /// X coordinate of the top-left corner
  double x = 1;
  /// Y coordinate of the top-left corner
  double y = 2;
  /// Width of the rectangle
  double width = 3;
  /// Height of the rectangle
  double height = 4;
}
```

### DESCRIPTION

- Used to define a rectangular area.
- Follows the standard graphics coordinate system. (0, 0) is the top-left corner, with x increasing to the right and y increasing downward.

## SRPoint

```protobuf
/// Defines a point in 2D space.
message SRPoint {
  /// X coordinate
  double x = 1;
  /// Y coordinate
  double y = 2;
}
```

## SRSize

```protobuf
/// Defines a size in 2D space.
message SRSize {
  /// Width
  double x = 1;
  /// Height
  double y = 2;
}
```
