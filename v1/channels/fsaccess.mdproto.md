---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.fsaccess"
import:
  - "common-types.proto"
---

# FILE-SYSTEM ACCESS CONTROL CHANNEL

Message definitions for the control plane of the file-system access (fsaccess) feature, which exposes file systems on the remote peer in a manner inspired by RDP's `\\tsclient\C` drive redirection.
The protocol design is symmetric, but the primary use case in Noctiluca is the client-to-server direction (the navigator client exposing local files to the remote macOS host).

This channel is the discovery-and-consent surface only. It defines `List`, `Mount`, and `Unmount` and carries the shared error model. Per-mount data-plane operations (`Open`, `Read`, `Write`, `ReadDir`, ...) are defined in the companion **`fsaccess_mount`** channel; the exposing peer opens one `fsaccess_mount` channel per granted mount session, mirroring the relationship between `projection` and `projection_data`.

---

# CONSTANTS

## AccessMode

```constset
constset AccessMode: uint32 {
  /// Read-only access. Write operations on a handle obtained with this mode MUST fail with `permissionDenied`.
  const read = 0;
  /// Write-only access. Read operations on a handle obtained with this mode MUST fail with `permissionDenied`.
  const write = 1;
  /// Read-write access.
  const readWrite = 2;
}
```

### DESCRIPTION

Access level negotiated by `FileSystemMountRequest` / `FileSystemMountResponse`. The granted access scopes every operation that the exposing peer subsequently accepts on the corresponding `fsaccess_mount` channel.

## FileSystemErrorCode

```constset
constset FileSystemErrorCode: uint32 {
  /// Reserved for unset or unknown error codes (e.g., field not populated).
  const unspecified = 0;

  // General
  /// Internal implementation error (uncaught exception, etc.).
  const internal = 1;
  /// One or more arguments are invalid (negative length, malformed structure, etc.).
  const invalidArgument = 2;
  /// The operation is not supported by this entry or filesystem.
  const notSupported = 3;

  // Handle / Session
  /// The handle does not exist or has expired. (POSIX: EBADF)
  const invalidHandle = 10;
  /// The session does not exist (e.g., the `sessionId` was never issued, or its mount channel is no longer active).
  const invalidSession = 11;
  /// The mount session was released while the operation was in flight.
  const notMounted = 12;

  // Path / Entry
  /// File or directory was not found. (POSIX: ENOENT)
  const notFound = 20;
  /// File or directory already exists. (POSIX: EEXIST)
  const alreadyExists = 21;
  /// Target is not a directory. (POSIX: ENOTDIR)
  const notDirectory = 22;
  /// Target is a directory. (POSIX: EISDIR)
  const isDirectory = 23;
  /// Directory is not empty. (POSIX: ENOTEMPTY)
  const notEmpty = 24;
  /// Path string is too long. (POSIX: ENAMETOOLONG)
  const pathTooLong = 25;
  /// Symbolic link loop detected. (POSIX: ELOOP)
  const loop = 26;
  /// Path normalization failed (e.g., contains `..` traversal, NUL bytes, or invalid UTF-8).
  const invalidPath = 27;
  /// Cross-device rename was attempted. (POSIX: EXDEV)
  const crossDevice = 28;

  // Permission / Policy
  /// Filesystem-level permission denied. (POSIX: EACCES)
  const accessDenied = 40;
  /// The operation itself is not permitted. (POSIX: EPERM)
  const permissionDenied = 41;
  /// The filesystem or mount is read-only. (POSIX: EROFS)
  const readOnlyFilesystem = 42;
  /// The exposing peer's policy denied the operation.
  const policyViolation = 43;
  /// The user denied the mount consent dialog.
  const consentDenied = 44;

  // Resource
  /// No space left on device. (POSIX: ENOSPC)
  const diskFull = 60;
  /// File would exceed the maximum allowed size. (POSIX: EFBIG)
  const fileTooLarge = 61;
  /// Quota exceeded (POSIX: EDQUOT, or Pattern A hard threshold violation).
  const quotaExceeded = 62;
  /// Too many open handles for this mount channel. (POSIX: EMFILE)
  const tooManyHandles = 63;

  // State
  /// Resource is busy and the operation cannot proceed. (POSIX: EBUSY)
  const busy = 80;
  /// Handle refers to an entry that no longer exists or has been replaced. (POSIX: ESTALE)
  const staleHandle = 81;

  // I/O
  /// Low-level I/O error. (POSIX: EIO)
  const ioError = 90;

  // Channel Lifecycle
  /// The channel is in the process of closing and rejects new operations.
  const channelClosing = 100;
}
```

### DESCRIPTION

The shared error vocabulary used across both the `fsaccess` control channel and the `fsaccess_mount` data channel. This is an open enum: receivers MUST treat unknown values as `internal` so that new error categories can be introduced in future revisions without breaking wire compatibility.

The `platformCode` and `platformName` fields on `ErrorInfo` carry the raw POSIX `errno` or Windows error code separately for debugging. Receivers MUST act on `code` and treat the platform fields as advisory only.

---

# STRUCTS

## FileSystemEntry

```protobuf
/// Represents a single mountable file-system entry exposed by a peer.
message FileSystemEntry {
  /// Stable identifier for this entry. Issued by the exposing peer and used as the `entryId` argument to `FileSystemMountRequest`.
  SRUUID id = 1;

  /// Human-readable label for display in the requester's UI (e.g., "내 작업물", "/home/cheesekun", "C:\\").
  /// This field is informational only and MUST NOT be parsed for path semantics.
  string name = 2;

  /// Free-form metadata for UI hints (icon names, capacity strings, etc.). Format is implementation-defined.
  map<string, string> metadata = 14;

  /// Reserved for future flag bits (e.g., volatile / removable indicators). MUST be zero in the current revision.
  uint32 flags = 15;
}
```

### DESCRIPTION

Returned in `FileSystemListResponse.entries` to advertise the entries that the exposing peer is willing to share.
The `id` is the only field carrying protocol semantics; `name` and `metadata` exist for display purposes only and MUST NOT influence path resolution on the requester side.

## ErrorInfo

```protobuf
/// Detailed error information returned in failed responses across both fsaccess and fsaccess_mount channels.
message ErrorInfo {
  /// Categorized error code. MUST be set whenever the parent response indicates failure.
  // @constset: FileSystemErrorCode
  uint32 code = 1;

  /// Human-readable diagnostic message. SHOULD be detailed enough to debug from logs alone, including concrete limit values, offending paths, or originating subsystems.
  string message = 2;

  /// Raw platform error code (POSIX `errno` or Windows `GetLastError`). Optional, provided for debugging only.
  optional int32 platformCode = 3;

  /// Symbolic name of the platform error (e.g., "EACCES", "ERROR_ACCESS_DENIED"). Optional.
  optional string platformName = 4;
}
```

### DESCRIPTION

Returned in the `error` field of every fsaccess and fsaccess_mount response when the response indicates failure (`success = false`).

### IMPLEMENTATION NOTES

`platformCode` and `platformName` MUST NOT carry protocol semantics. Receivers MUST act on `code` and SHOULD treat the platform fields as advisory diagnostics only.
The `message` field is intended to be verbose; per the project's spec-violation policy, fatal-close paths in particular benefit from detailed reasons.

---

# MESSAGES

## FileSystemListRequest

```protobuf
/// Requests the list of file-system entries the remote peer is willing to expose.
// @opcode: 0x8001
message FileSystemListRequest {
  uint64 requestId = 1;

  /// Reserved for future flag bits. MUST be zero in the current revision.
  uint32 flags = 15;
}
```

### DESCRIPTION

Sent by either peer to discover which file-system entries the remote peer is currently willing to share.
Listing alone does NOT grant access to any entry; access is only granted by a successful `FileSystemMountRequest`, which itself gates on user consent on the exposing side.

## FileSystemListResponse

```protobuf
/// Response to a FileSystemListRequest.
// @opcode: 0x8002
message FileSystemListResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Entries the responding peer is willing to share. MAY be empty even when `success` is true (the peer simply has nothing exposed at the moment).
  repeated FileSystemEntry entries = 3;

  /// Populated when `success` is false.
  optional ErrorInfo error = 4;
}
```

### DESCRIPTION

Sent in response to a `FileSystemListRequest`. An empty `entries` list with `success = true` is a valid response and indicates that the peer has no entries currently exposed; it is not an error.

### IMPLEMENTATION NOTES

The length of `entries` MUST NOT exceed 32 in normal operation (Pattern A spec limit). Receivers SHOULD truncate to 32 with a warning when up to 48 entries are received (soft limit, 1.5x), and MUST close the channel with `ServerNotice(quotaExceeded)` when more than 1024 entries are received (hard limit, 32x).

## FileSystemMountRequest

```protobuf
/// Requests to mount a file-system entry. The exposing peer prompts the user for consent before responding.
// @opcode: 0x8003
message FileSystemMountRequest {
  uint64 requestId = 1;

  /// Identifier of the entry to mount, taken from `FileSystemListResponse.entries[].id`.
  SRUUID entryId = 2;

  /// Access level requested by the requester. The exposing peer MAY downgrade this in the response (e.g., grant `read` when `readWrite` was requested).
  // @constset: AccessMode
  uint32 requestedAccess = 3;

  /// Optional human-readable reason displayed in the consent dialog (e.g., "AppStream session for Photoshop").
  /// The exposing peer SHOULD treat the value as untrusted display text and MUST NOT use it for any access decision.
  optional string reason = 4;

  /// Reserved for future flag bits. MUST be zero in the current revision.
  uint32 flags = 15;
}
```

### DESCRIPTION

Initiates a mount session for the entry identified by `entryId`.
The exposing peer is expected to display a consent UI to the local user before responding. The exact UI policy is implementation-defined: a headless server (such as a macOS host running Noctiluca) MAY apply a static allow / deny / whitelist policy instead of an interactive prompt.

A successful mount results in two follow-up actions by the exposing peer: (1) the `FileSystemMountResponse` carrying the granted `sessionId`, and (2) a `ChannelStartRequest` opening a new `fsaccess_mount` channel whose argument list addresses the same `sessionId`. The data-plane operations for the mount are then conducted on the `fsaccess_mount` channel; this control channel sees no further per-mount messages aside from `FileSystemUnmountRequest`.

### IMPLEMENTATION NOTES

The length of `reason` MUST NOT exceed 512 bytes (Pattern A spec limit). Receivers MUST close the channel with `ServerNotice(quotaExceeded)` when more than 8 KiB is received (hard limit).

The exposing peer MAY enforce a per-channel limit on the number of concurrently active mount sessions: spec limit 16, soft limit 24 (warn), hard limit 512 (channel close with `quotaExceeded`).

## FileSystemMountResponse

```protobuf
/// Response to a FileSystemMountRequest.
// @opcode: 0x8004
message FileSystemMountResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Identifier of the granted mount session. Valid only when `success` is true.
  /// The exposing peer opens an `fsaccess_mount` channel keyed by this `sessionId` shortly after sending this response; data-plane operations are conducted on that channel.
  SRUUID sessionId = 3;

  /// Access level actually granted. MUST NOT exceed `requestedAccess`; MAY be lower if the user downgraded the grant (e.g., approved read-only when read-write was requested).
  // @constset: AccessMode
  uint32 grantedAccess = 4;

  /// Populated when `success` is false. Typical codes include `consentDenied`, `notFound`, and `policyViolation`.
  optional ErrorInfo error = 5;
}
```

### DESCRIPTION

Sent by the exposing peer after the consent decision has been made.
A successful response is paired with a follow-up `ChannelStartRequest` opening a new `fsaccess_mount` channel keyed by `sessionId`. The requester correlates the incoming `fsaccess_mount` channel with this response by matching the `sessionId` carried in the channel arguments. See the `CHANNEL LIFECYCLE` appendix for the full open / close sequence.

### IMPLEMENTATION NOTES

When the user explicitly rejects the consent dialog, the response MUST set `code = consentDenied`. Requesters SHOULD NOT automatically retry on `consentDenied`; instead, the retry decision SHOULD be surfaced to the requester's user.

The exposing peer SHOULD send `FileSystemMountResponse` before issuing the corresponding `ChannelStartRequest` for the `fsaccess_mount` channel, so that the requester has the `sessionId` in hand when validating the incoming channel arguments. If the order is reversed on the wire (e.g., due to channel-management ordering), the requester MAY buffer the unfamiliar `ChannelStartRequest` for a brief window and resolve it once the matching `FileSystemMountResponse` arrives.

## FileSystemUnmountRequest

```protobuf
/// Requests to release an active mount session.
// @opcode: 0x8005
message FileSystemUnmountRequest {
  uint64 requestId = 1;

  /// Identifier of the session to release.
  SRUUID sessionId = 2;

  /// Reserved for future flag bits. MUST be zero in the current revision.
  uint32 flags = 15;
}
```

### DESCRIPTION

Releases the mount session identified by `sessionId` together with all resources scoped to it.
On a successful unmount the exposing peer MUST close the corresponding `fsaccess_mount` channel; all open handles tied to the session are implicitly closed by that channel close, and any in-progress stream transfers (`FileSystemRequestStreamRead` / `FileSystemRequestStreamWrite`) MUST be aborted as part of the cascade.

The requester MAY also release a mount by simply closing the corresponding `fsaccess_mount` channel directly; the exposing peer MUST treat that channel close as equivalent to an `Unmount`. Sending `FileSystemUnmountRequest` on the control channel is the preferred path because it gives the exposing peer an explicit, auditable signal.

## FileSystemUnmountResponse

```protobuf
/// Response to a FileSystemUnmountRequest.
// @opcode: 0x8006
message FileSystemUnmountResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Populated when `success` is false. The most common code is `invalidSession` (the session never existed or was already released).
  optional ErrorInfo error = 3;
}
```

### DESCRIPTION

Sent in response to a `FileSystemUnmountRequest`.

### IMPLEMENTATION NOTES

Even when `success` is false, the requester SHOULD treat the session as released and clean up any local state associated with it. The session is considered no longer usable regardless of the response outcome, and the corresponding `fsaccess_mount` channel (if any) is expected to be closed.

---

# CHANNEL ARGUMENTS

The fsaccess control channel does not currently take any channel arguments via `ChannelStartRequest.args`. Requesters MUST send an empty argument list, and exposing peers MUST ignore any unknown arguments to remain forward-compatible. Future revisions MAY introduce optional arguments (for example, to negotiate per-channel feature flags) without breaking existing clients.

The companion `fsaccess_mount` channel does take channel arguments; see that channel's specification for details.

---

# CHANNEL LIFECYCLE

The fsaccess channel layers on top of the standard Sirius channel lifecycle (described in `general.mdproto.md`) with the following channel-specific constraints.

## Authentication Gate

The fsaccess channel MUST NOT be opened before the Sirius authentication phase has completed (i.e., before `AuthResponse` has been received on the main channel). A `ChannelStartRequest` for fsaccess sent earlier in the session MUST be rejected at the channel-management level. If a fsaccess message somehow reaches the channel before authentication completes, the channel MUST be closed with `ServerNotice(phaseViolation)`.

## Single-Instance Constraint

Each Sirius connection MAY have at most one active fsaccess control channel. Opening a second fsaccess channel while another is active MUST fail at the channel-management level. If for any reason a duplicate channel reaches the message phase, it MUST be closed with `ServerNotice(policyViolation)`.

The single-channel constraint simplifies sessionId allocation and ensures predictable cleanup semantics on connection teardown. There is no equivalent constraint on `fsaccess_mount`: any number of mount channels MAY coexist subject to the per-channel mount-session quota described above.

## Mount Channel Open Sequence

When a `FileSystemMountRequest` is approved, the exposing peer follows this sequence:

1. Send `FileSystemMountResponse` on the fsaccess control channel with `success = true`, the issued `sessionId`, and the `grantedAccess`.
2. Send a `ChannelStartRequest` opening a new `fsaccess_mount` channel whose `featureId` is `fileSystemAccessMount` and whose `args` carry the same `sessionId`.

The requester accepts the `fsaccess_mount` channel after validating that the `sessionId` matches a session it has just been granted on the control channel. A `ChannelStartRequest` whose `sessionId` does not match any pending or active mount session MUST be rejected with `ChannelStartResponse(success = false)`.

## Cascade Cleanup on Close

The control channel and its mount channels are bound by a strict parent / child cleanup relationship.

- When the fsaccess control channel closes — by graceful close, transport failure, connection drop, or remote channel close — every `fsaccess_mount` channel opened against it MUST be closed by both peers. All sessions, handles, and in-progress stream transfers are released as part of those individual mount-channel closes.
- When an `fsaccess_mount` channel closes — by graceful close, the requester closing it as an implicit unmount, or transport failure — both peers MUST treat the corresponding mount session as released. No further `FileSystemUnmountRequest` exchange is needed; if the requester nonetheless sends one, the exposing peer responds with `success = false` and `code = invalidSession`.

The requester SHOULD treat any locally cached `sessionId` value as invalid after the corresponding `fsaccess_mount` channel has closed, and MUST NOT attempt to reuse it on a freshly opened channel.

---

# SPEC VIOLATION HANDLING

This appendix collects the per-channel size and count limits that apply specifically to the fsaccess control channel. Limits that apply to the per-mount data plane are defined in the `fsaccess_mount` channel's own `SPEC VIOLATION HANDLING` section.

## Pattern A Thresholds

| Subject | spec limit | warn (≈1.5×) | hard (≈32×) |
|---|---|---|---|
| `FileSystemListResponse.entries` count | **32** | 48 | **1024** |
| Concurrently active mount sessions per connection | **16** | 24 | **512** |
| `FileSystemMountRequest.reason` byte length | **512** | — | **8 KiB** |

Interpretation:

- `value ≤ spec limit` — normal operation, no log entry.
- `spec limit < value ≤ warn limit` — likely a sender-side bug. The receiver SHOULD log a warning and either truncate to the spec limit (for `repeated` fields and byte buffers) or return the appropriate response-level error such as `tooManyHandles`. The channel remains open.
- `warn limit < value ≤ hard limit` — gray zone. The receiver SHOULD log an elevated warning and continue with truncate or response-error semantics. The channel remains open unless the implementation explicitly escalates.
- `value > hard limit` — treated as deliberate abuse. The receiver MUST close the channel and emit a `ServerNotice` carrying the specific code (`quotaExceeded` or `payloadTooLarge`) along with a detailed reason message that names the violated subject, the observed value, and the limit.

## Severity Classification

The categorization below distinguishes violations that result in a per-operation error response from violations that close the entire channel.

### Response-level errors (channel remains open)

These are normal operational failures that map to the `ErrorInfo` in the relevant response: `notFound`, `accessDenied`, `permissionDenied`, `policyViolation`, `consentDenied`, `invalidSession`, `notMounted`, `invalidArgument`, `notSupported`.

### Channel-fatal violations (channel closes with `ServerNotice`)

These are protocol-level violations that cannot be safely recovered from on the same channel:

| Trigger | `ServerNotice` code |
|---|---|
| Unknown opcode received on the channel | `unsupportedOpcode` |
| Any Pattern A subject exceeds its hard limit | `quotaExceeded` |
| fsaccess message received before the Sirius authentication phase has completed | `phaseViolation` |
| A second fsaccess control channel is opened while another is already active on the same connection | `policyViolation` |

The `message` field on `ServerNotice` SHOULD be detailed enough to identify the violation from logs alone, naming the offending subject, the observed value, and the relevant limit.
