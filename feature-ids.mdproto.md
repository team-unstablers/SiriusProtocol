---
syntax: "proto3"
package: "sirius.msgdef"
---

# FEATURE IDENTIFIERS

A Feature ID is a 128-bit UUID used to identify a specific feature in the Sirius protocol.
Each feature has a unique Feature ID, which is used during feature negotiation (`ServerHello`)
and channel activation (`ChannelStartRequest`).

The Sirius protocol is designed to be extensible beyond the standard features defined in the
`SiriusFeatureID` constset — implementors MAY generate arbitrary UUIDs to define and register
new features.

---

# CONSTANTS

## SiriusFeatureID

```constset
constset SiriusFeatureID: UUID {
  /// HIDIO: Redirects HID input such as keyboard and mouse.
  const hidio = uuid"405E6E75-8B64-4E8F-91FF-E9E5A2C6BC77";

  /// Projection: Configures and controls screen capture.
  const projection = uuid"362A7AF0-2AC0-48DE-8F6B-5DB5D01E99DC";

  /// ProjectionData: Transmits screen capture data.
  const projectionData = uuid"CDE57E7B-9528-47F7-8FEC-14389301A990";

  /// Clipboard: Synchronizes clipboard contents between server and client.
  const clipboard = uuid"8A3D9F2E-6B1C-4E75-B8A0-D4F2C7E19A3B";

  /// Transfer: Transfers large data such as files and clipboard data.
  const transfer = uuid"9F4EE026-35B4-4B3D-AD0B-168EE4CDB175";

  /// FileSystemAccess: Control plane for the file-system access channel (discovery and consent).
  const fileSystemAccess = uuid"3F64ED71-E64C-4D5F-ACF3-77EA8731EDE3";

  /// FileSystemAccessMount: Per-mount data plane for the file-system access channel (handle, I/O, directory and file operations).
  const fileSystemAccessMount = uuid"31035A4B-AB82-4F7D-A165-F763C382C1B9";
}
```

### DESCRIPTION

- The list of standard Feature IDs defined by the Sirius protocol.
- These are used in `ServerHello.supportedFeatures` to advertise the features supported by the server, and in `ChannelStartRequest.featureId` to specify which feature a channel will serve.
