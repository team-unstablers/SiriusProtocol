---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.fsaccess_mount"
import:
  - "common-types.proto"
  - "v1/channels/fsaccess.proto"
---

# FILE-SYSTEM ACCESS MOUNT CHANNEL

Message definitions for the per-mount data plane of the file-system access (fsaccess) feature. Each `fsaccess_mount` channel is bound to exactly one mount session previously granted by the companion `fsaccess` control channel. The exposing peer opens an `fsaccess_mount` channel for every approved `FileSystemMountRequest`, mirroring the relationship between `projection` and `projection_data`.

The channel surface covers POSIX-style file handles with bounded inline read / write, directory and metadata operations, and an opt-in handoff to the Transfer channel for sequential bulk transfer. The shared error vocabulary (`FileSystemErrorCode`, `ErrorInfo`) is imported from the control channel; per-message documentation references those types directly.

---

# CONSTANTS

## CreateDisposition

```constset
constset CreateDisposition: uint32 {
  /// Open the existing file. Fails with `notFound` if the file does not exist.
  const openExisting = 0;
  /// Create a new file. Fails with `alreadyExists` if the file already exists. (POSIX: O_CREAT | O_EXCL)
  const createNew = 1;
  /// Create a new file, or truncate to zero length if it already exists. (POSIX: O_CREAT | O_TRUNC)
  const createAlways = 2;
  /// Open the existing file, or create it if it does not exist. (POSIX: O_CREAT)
  const openOrCreate = 3;
  /// Open and truncate the existing file. Fails with `notFound` if the file does not exist. (POSIX: O_TRUNC)
  const truncateExisting = 4;
}
```

## OpenFlags

```optionset
optionset OpenFlags: uint32 {
  /// Subsequent writes always append to the end of the file, ignoring `WriteRequest.offset`. (POSIX: O_APPEND)
  option append = 0x1;
  /// If the final path component is a symbolic link, the open MUST fail with `loop`. (POSIX: O_NOFOLLOW)
  option noFollowSymlinks = 0x2;
  /// The handle is opened for directory access only and may be used with `ReadDir`. Fails with `notDirectory` if the path is not a directory.
  option directoryOnly = 0x4;
}
```

## FileType

```constset
constset FileType: uint32 {
  /// Type could not be determined or is intentionally unspecified.
  const unknown = 0;
  /// Regular file.
  const file = 1;
  /// Directory.
  const directory = 2;
  /// Symbolic link.
  const symlink = 3;
  /// Other type (block device, character device, FIFO, socket, etc.).
  const other = 99;
}
```

### DESCRIPTION

This is an open enum. Receivers MUST treat unknown values as `unknown` rather than rejecting the message, so that future revisions can introduce new types without breaking wire compatibility.

## FileAttributes

```optionset
optionset FileAttributes: uint32 {
  /// File is hidden. (POSIX: dotfile convention; Windows: FILE_ATTRIBUTE_HIDDEN)
  option hidden = 0x1;
  /// File is read-only. (POSIX: synthesized from permission bits; Windows: FILE_ATTRIBUTE_READONLY)
  option readOnly = 0x2;
  /// File is a system file. (POSIX: synthesized; Windows: FILE_ATTRIBUTE_SYSTEM)
  option system = 0x4;
  /// File is marked for archiving. (POSIX: synthesized; Windows: FILE_ATTRIBUTE_ARCHIVE)
  option archive = 0x8;
}
```

## LockType

```constset
constset LockType: uint32 {
  /// Shared (read) lock. Multiple shared locks MAY coexist on overlapping ranges across different handles, but no exclusive lock MAY coexist with a shared lock on overlapping ranges. (POSIX: F_RDLCK)
  const shared = 0;
  /// Exclusive (write) lock. Only one exclusive lock MAY exist on a given range, and it MUST NOT coexist with any other lock (shared or exclusive) on overlapping ranges. (POSIX: F_WRLCK)
  const exclusive = 1;
}
```

### DESCRIPTION

Used by `FileSystemLockRequest`, `FileSystemTestLockRequest`, and `FileSystemTestLockResponse` to identify the kind of byte-range lock. Acquiring a `shared` lock requires the underlying handle to have been opened with read access; acquiring an `exclusive` lock requires write access. See the `LOCK SEMANTICS` appendix for the full ownership and lifetime model.

---

# STRUCTS

## FileStat

```protobuf
/// Cross-platform stat information for a single file-system entry.
message FileStat {
  /// Type of the entry.
  // @constset: FileType
  uint32 type = 1;

  /// Size of the file in bytes. For directories, the value is implementation-defined and MAY be zero.
  uint64 size = 2;

  /// POSIX permission bits (mode_t). On Windows, the value is synthesized from file attributes (e.g., 0o444 for read-only files).
  uint32 mode = 3;

  /// Last data-modification time in milliseconds since the Unix epoch. MUST be populated.
  int64 mtimeMs = 4;

  /// Last access time in milliseconds since the Unix epoch. Best-effort; MAY be zero if the platform does not track it.
  int64 atimeMs = 5;

  /// Birth (creation) time in milliseconds since the Unix epoch. Best-effort; MAY be zero if the platform does not track it (e.g., on filesystems that lack a creation timestamp).
  int64 btimeMs = 6;

  /// Platform-style attribute bitmask. POSIX implementations synthesize the bits from naming and permission conventions.
  // @optionset: FileAttributes
  uint32 attributes = 7;

  /// Target of the symbolic link. Populated only when `type` is `symlink` and the originating stat operation did not follow links.
  optional string symlinkTarget = 8;

  /// Platform-specific extras (e.g., POSIX `uid` / `gid`, NTFS reparse-point class). Format is implementation-defined.
  map<string, string> metadata = 14;
}
```

### DESCRIPTION

Carries common file-system metadata across POSIX (macOS, Linux) and Windows in a single shape.
`mtimeMs` is the only timestamp that MUST be populated; the others are best-effort and MAY be zero when the underlying filesystem does not track them.

### IMPLEMENTATION NOTES

The POSIX `ctime` (status-change time) is intentionally omitted because it conflicts with the Windows interpretation of "creation time". Implementations that need to surface POSIX `ctime` SHOULD do so via the `metadata` map.

## DirEntry

```protobuf
/// A single entry returned from a directory listing.
message DirEntry {
  /// Basename of the entry (e.g., "foo.txt", "subdir"). MUST NOT contain '/' or NUL bytes. The "." and ".." entries are NEVER returned.
  string name = 1;

  /// Stat information for this entry. Populated regardless of entry type.
  FileStat stat = 2;
}
```

### DESCRIPTION

Returned in repeated form by `FileSystemReadDirResponse.entries`.
The full `FileStat` is included so that callers performing UI-style listings (e.g., `ls -l`) do not need a separate `Stat` round-trip per entry.

---

# MESSAGES

## FileSystemOpenRequest

```protobuf
/// Opens a file or directory and returns a handle bound to this mount channel.
// @opcode: 0x8001
message FileSystemOpenRequest {
  uint64 requestId = 1;

  /// Path to open, relative to the mount root. MUST use '/' as the separator and MUST be valid UTF-8. Leading '/' is allowed but interpreted as relative to the mount root (not the host filesystem root).
  string path = 2;

  /// Access mode requested for the handle. MUST NOT exceed the `grantedAccess` of the mount session that this channel is bound to; otherwise the open MUST fail with `permissionDenied`.
  // @constset: sirius.msgdef.v1.channels.fsaccess.AccessMode
  uint32 accessMode = 3;

  /// Disposition controlling whether the file is opened, created, or truncated.
  // @constset: CreateDisposition
  uint32 createDisposition = 4;

  /// Additional flags controlling open behavior (append, noFollowSymlinks, directoryOnly).
  // @optionset: OpenFlags
  uint32 flags = 5;

  /// POSIX permission bits to apply when a new file is created. Used only when `createDisposition` is `createNew`, `createAlways`, or `openOrCreate` and the file is actually created. On Windows, this is best-effort.
  uint32 mode = 6;
}
```

### DESCRIPTION

Opens a file or directory under the channel's bound mount session and returns a handle for subsequent operations.
The handle's lifetime is bound to this `fsaccess_mount` channel and an explicit `FileSystemCloseRequest`. When the channel closes (graceful close, transport failure, or cascade close from the parent fsaccess control channel), the handle is released and any pending stream transfers attached to it are aborted.

To obtain a directory handle for use with `FileSystemReadDirRequest`, the `directoryOnly` flag MUST be set; otherwise opening a directory MUST fail with `isDirectory`. Conversely, opening a regular file with `directoryOnly` set MUST fail with `notDirectory`.

### IMPLEMENTATION NOTES

When `flags & append` is set, the resulting handle ignores the `offset` field of any `FileSystemWriteRequest` and always writes at the current end of file. This mirrors POSIX `O_APPEND` semantics.

Path validation and path-length limits are described in the `PATH HANDLING` appendix section.

## FileSystemOpenResponse

```protobuf
/// Response to a FileSystemOpenRequest.
// @opcode: 0x8002
message FileSystemOpenResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Identifier of the opened handle. Issued by the exposing peer and unique within this channel. Valid only when `success` is true.
  uint64 handleId = 3;

  /// Stat snapshot of the opened entry at the moment of opening, provided as a convenience so callers do not need to issue a separate `FStat` round-trip.
  FileStat stat = 4;

  /// Populated when `success` is false.
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 5;
}
```

### DESCRIPTION

Sent by the exposing peer after the open is processed.
The included `stat` is a snapshot at open time and reflects the state observed by the underlying `open` / `CreateFile` call. Subsequent changes to the entry on disk are not reflected here; callers needing a fresh view MUST issue `FileSystemFStatRequest`.

### IMPLEMENTATION NOTES

Common failure modes and their codes:

- `notFound` — `createDisposition = openExisting` or `truncateExisting` and the file does not exist.
- `alreadyExists` — `createDisposition = createNew` and the file exists.
- `isDirectory` — the path resolves to a directory but `directoryOnly` is not set.
- `notDirectory` — `directoryOnly` is set but the path is not a directory.
- `loop` — `flags & noFollowSymlinks` is set and the final component is a symbolic link.
- `accessDenied` — the underlying filesystem rejected the open.
- `permissionDenied` — `accessMode` exceeds the session's `grantedAccess`, or the mount is read-only and write access was requested.
- `tooManyHandles` — the per-channel handle soft limit was reached.

## FileSystemCloseRequest

```protobuf
/// Closes an open file or directory handle.
// @opcode: 0x8003
message FileSystemCloseRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;
}
```

### DESCRIPTION

Releases the handle identified by `handleId`. The exposing peer MUST perform an implicit flush before closing (equivalent to `FileSystemFlushRequest` followed by `close`).

If a stream transfer is in progress on this handle (initiated through `FileSystemRequestStreamReadRequest` or `FileSystemRequestStreamWriteRequest`), it MUST be aborted as part of the close. The associated Transfer channel SHOULD be closed by the side that opened it.

After a successful close, the `handleId` MUST NOT be reused by the exposing peer for a different open within the same channel; this prevents accidental cross-talk between a freshly issued handle and stale references on the requester side.

## FileSystemCloseResponse

```protobuf
/// Response to a FileSystemCloseRequest.
// @opcode: 0x8004
message FileSystemCloseResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Populated when `success` is false. The most common code is `invalidHandle` (the handle was already closed or never existed).
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Sent in response to `FileSystemCloseRequest`.

### IMPLEMENTATION NOTES

Even when `success` is false, the requester SHOULD treat the handle as closed and clean up local state. The handle is considered no longer usable regardless of the response outcome.

If the implicit flush fails (for example, due to `diskFull`), the close MAY return `success = false` with the underlying I/O error code, but the handle MUST still be released on the exposing side.

## FileSystemReadRequest

```protobuf
/// Reads bytes from an open file handle at an absolute offset.
// @opcode: 0x8021
message FileSystemReadRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;

  /// Absolute offset within the file at which to begin reading.
  uint64 offset = 3;

  /// Number of bytes the requester wishes to read. The exposing peer MAY return fewer bytes (partial read) and MUST NOT return more.
  uint32 length = 4;
}
```

### DESCRIPTION

Reads up to `length` bytes from the file at `offset`. The read is independent of any prior read on the same handle; multiple reads MAY be pipelined concurrently with distinct `requestId`s, and they MAY proceed in any order on the wire.

A `length` of zero is a valid request and MUST be answered with an empty `data` field, `isEof = false`, and `success = true`. It is treated as a no-op probe.

### IMPLEMENTATION NOTES

Typical values for `length` fall in the 64 KiB – 1 MiB range, balancing memory pressure against round-trip count. The maximum is bounded by the protocol-wide 16 MiB frame size limit defined in the protocol introduction.

Reads on a handle that currently has an active stream transfer (initiated by `FileSystemRequestStreamRead` / `FileSystemRequestStreamWrite`) are permitted; consistency between the inline read and the stream is the requester's responsibility.

## FileSystemReadResponse

```protobuf
/// Response to a FileSystemReadRequest.
// @opcode: 0x8022
message FileSystemReadResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Bytes read. MUST be no longer than the requested `length`. MAY be shorter (partial read).
  bytes data = 3;

  /// Set to true when this read reached or passed the end of file. The requester MAY rely on either this flag or `data.length() < requested length` as an EOF signal; both MUST agree.
  bool isEof = 4;

  /// Populated when `success` is false.
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 5;
}
```

### DESCRIPTION

Returned in response to `FileSystemReadRequest`.
A successful response with empty `data` and `isEof = true` indicates a clean EOF (the read started at or past the end of file). A successful response with empty `data` and `isEof = false` MUST NOT occur except in response to a zero-length read.

## FileSystemWriteRequest

```protobuf
/// Writes bytes to an open file handle at an absolute offset.
// @opcode: 0x8023
message FileSystemWriteRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;

  /// Absolute offset within the file at which to begin writing. Ignored when the handle was opened with `OpenFlags.append`; in that case the write always lands at the current end of file.
  uint64 offset = 3;

  /// Bytes to write. The exposing peer MAY accept fewer bytes than provided (partial write) and MUST report the actual count in `bytesWritten`.
  bytes data = 4;
}
```

### DESCRIPTION

Writes `data` to the file at `offset` (or at the current EOF if the handle is in append mode). The exposing peer MUST NOT silently extend a partial write; if only some bytes were written, the response reports the actual count in `bytesWritten`, and the requester is responsible for re-issuing the remainder.

### IMPLEMENTATION NOTES

Typical chunk sizes for `data` fall in the 64 KiB – 1 MiB range, balancing memory pressure against round-trip count. The maximum is bounded by the protocol-wide 16 MiB frame size limit defined in the protocol introduction.

Write durability is not guaranteed by `success = true` alone. Callers requiring durability MUST issue `FileSystemFlushRequest` after the relevant writes (or close the handle, which performs an implicit flush).

## FileSystemWriteResponse

```protobuf
/// Response to a FileSystemWriteRequest.
// @opcode: 0x8024
message FileSystemWriteResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Number of bytes actually written. MUST be no greater than `data.length()` from the request. May be less (partial write) on conditions such as approaching `diskFull` or `quotaExceeded`.
  uint32 bytesWritten = 3;

  /// Populated when `success` is false. A response with `success = false` MAY still carry `bytesWritten > 0` if the failure occurred mid-write (e.g., `diskFull` after partial progress).
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 4;
}
```

### DESCRIPTION

Returned in response to `FileSystemWriteRequest`. Mirrors POSIX `write(2)` semantics in that partial writes are normal and the requester is expected to handle them.

## FileSystemFlushRequest

```protobuf
/// Forces buffered writes on a handle to be persisted to durable storage.
// @opcode: 0x8025
message FileSystemFlushRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;
}
```

### DESCRIPTION

Equivalent in intent to POSIX `fsync(2)`. The exposing peer MUST flush all buffered writes for the handle to durable storage before responding with `success = true`.

A flush on a handle opened in read-only mode is a no-op and MUST succeed.

## FileSystemFlushResponse

```protobuf
/// Response to a FileSystemFlushRequest.
// @opcode: 0x8026
message FileSystemFlushResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Populated when `success` is false. Common codes include `ioError` (low-level write failure during sync) and `diskFull` (writes that had previously appeared to succeed could not be persisted).
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Returned in response to `FileSystemFlushRequest`. A `success = false` response signals that prior writes which the requester believed had succeeded may in fact be lost or partially persisted; the requester SHOULD treat this as a serious data-integrity event.

## FileSystemStatRequest

```protobuf
/// Retrieves stat information for a path within the bound mount.
// @opcode: 0x8041
message FileSystemStatRequest {
  uint64 requestId = 1;

  /// Path to stat, relative to the mount root. Same encoding rules as `FileSystemOpenRequest.path`.
  string path = 2;

  /// When true, symbolic links along the path (and at the final component) are followed and the stat reflects the link target. When false, the link itself is stat'd and `FileStat.symlinkTarget` is populated for symlink entries.
  bool followSymlinks = 3;
}
```

### DESCRIPTION

Returns metadata for the entry at `path`. Used to inspect existence, type, size, and timestamps without opening a handle.

### IMPLEMENTATION NOTES

When `followSymlinks` is true and the link resolution lands outside the mount root (or escapes through any subtree boundary), the operation MUST fail with `policyViolation` rather than returning the resolved entry's stat.

Path validation rules and length limits are described in the `PATH HANDLING` appendix section.

## FileSystemStatResponse

```protobuf
/// Response to a FileSystemStatRequest.
// @opcode: 0x8042
message FileSystemStatResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Stat information for the requested entry. Valid only when `success` is true.
  FileStat stat = 3;

  /// Populated when `success` is false. Common codes include `notFound`, `accessDenied`, and `loop`.
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 4;
}
```

### DESCRIPTION

Returned in response to `FileSystemStatRequest`.

## FileSystemFStatRequest

```protobuf
/// Retrieves stat information for an open handle.
// @opcode: 0x8043
message FileSystemFStatRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;
}
```

### DESCRIPTION

Returns metadata for the entry referenced by `handleId`. Equivalent in intent to POSIX `fstat(2)`.

Unlike `FileSystemStatRequest`, the result is stable against rename or unlink of the entry that occurs after the handle was opened: as long as the handle is still valid, `FStat` continues to reflect the same underlying file. This is the recommended way to obtain a fresh stat after performing writes that may have changed the size or modification time.

## FileSystemFStatResponse

```protobuf
/// Response to a FileSystemFStatRequest.
// @opcode: 0x8044
message FileSystemFStatResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Stat information for the entry referenced by the handle. Valid only when `success` is true.
  FileStat stat = 3;

  /// Populated when `success` is false. The most common code is `invalidHandle`.
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 4;
}
```

### DESCRIPTION

Returned in response to `FileSystemFStatRequest`.

## FileSystemReadDirRequest

```protobuf
/// Reads a batch of entries from an open directory handle.
// @opcode: 0x8061
message FileSystemReadDirRequest {
  uint64 requestId = 1;

  /// Identifier of an open directory handle, obtained by calling `FileSystemOpenRequest` with `OpenFlags.directoryOnly` set.
  uint64 handleId = 2;

  /// Maximum number of entries the requester wishes to receive in this batch. The exposing peer treats this as a hint and MAY return fewer entries to fit within the per-frame size budget.
  uint32 maxEntries = 3;
}
```

### DESCRIPTION

Reads the next batch of entries from a directory handle. The exposing peer maintains an internal cursor on the handle so that successive calls return successive ranges of entries; the requester does not pass a cursor token explicitly.

The "." and ".." entries are NEVER returned, mirroring the convention chosen for this channel: the requester already knows the parent path and the directory itself.

### IMPLEMENTATION NOTES

The exposing peer MAY return fewer than `maxEntries` even when more entries remain. The requester MUST continue to call `ReadDir` until `isEnd` is set in the response.

Calling `ReadDir` on a handle that was not opened with `directoryOnly` MUST fail with `notDirectory`.

## FileSystemReadDirResponse

```protobuf
/// Response to a FileSystemReadDirRequest.
// @opcode: 0x8062
message FileSystemReadDirResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Next batch of entries from the directory. MAY be empty when the previous batch already exhausted the directory.
  repeated DirEntry entries = 3;

  /// Set to true when the directory has been fully read and no further `ReadDir` calls will return more entries on this handle.
  bool isEnd = 4;

  /// Populated when `success` is false.
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 5;
}
```

### DESCRIPTION

Returned in response to `FileSystemReadDirRequest`.
A successful response with empty `entries` and `isEnd = true` is the canonical end-of-directory signal. A successful response with empty `entries` and `isEnd = false` is permitted but discouraged (the exposing peer SHOULD coalesce empty batches with the next non-empty batch when reasonable).

### IMPLEMENTATION NOTES

For paginated directory listings, exposing peers SHOULD typically respond with batches of a few hundred entries to balance round-trips against memory pressure. The response is bounded by the protocol-wide 16 MiB frame size limit defined in the protocol introduction.

If the directory is modified concurrently with the listing, the exposing peer MAY return inconsistent results (entries duplicated or missed). The protocol does not provide snapshot-isolation semantics; consumers requiring consistency MUST coordinate at the application level.

## FileSystemMkdirRequest

```protobuf
/// Creates a new directory at the given path.
// @opcode: 0x8063
message FileSystemMkdirRequest {
  uint64 requestId = 1;

  /// Path of the directory to create, relative to the mount root.
  string path = 2;

  /// POSIX permission bits to apply to the new directory. On Windows, this is best-effort.
  uint32 mode = 3;
}
```

### DESCRIPTION

Creates a single new directory at `path`. The parent directory MUST already exist; this operation does NOT create intermediate directories (mirroring `mkdir(2)` rather than `mkdir -p`). When intermediate directories are needed, the requester is expected to issue a sequence of `Mkdir` calls.

### IMPLEMENTATION NOTES

Common failure codes:

- `alreadyExists` — an entry already exists at `path` (regardless of type).
- `notFound` — the parent directory does not exist.
- `notDirectory` — a non-directory parent component was encountered.
- `readOnlyFilesystem` — the mount or filesystem is read-only.

Path validation rules and length limits are described in the `PATH HANDLING` appendix section.

## FileSystemMkdirResponse

```protobuf
/// Response to a FileSystemMkdirRequest.
// @opcode: 0x8064
message FileSystemMkdirResponse {
  uint64 requestId = 1;
  bool success = 2;
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Returned in response to `FileSystemMkdirRequest`.

## FileSystemRmdirRequest

```protobuf
/// Removes an empty directory.
// @opcode: 0x8065
message FileSystemRmdirRequest {
  uint64 requestId = 1;

  /// Path of the directory to remove, relative to the mount root.
  string path = 2;
}
```

### DESCRIPTION

Removes the directory at `path`. The directory MUST be empty; otherwise the operation MUST fail with `notEmpty`.

This operation does NOT recurse into subdirectories. Recursive deletion is intentionally left to the requester (via `ReadDir` followed by `Unlink` and `Rmdir` calls), so that exposing-peer policy hooks (allow-lists, audit logging, prompts) can intervene at each step.

### IMPLEMENTATION NOTES

Common failure codes:

- `notFound` — the directory does not exist.
- `notDirectory` — `path` resolves to a non-directory.
- `notEmpty` — the directory contains entries.
- `busy` — the directory is the working directory of another process or is otherwise locked.
- `readOnlyFilesystem` — the mount or filesystem is read-only.

## FileSystemRmdirResponse

```protobuf
/// Response to a FileSystemRmdirRequest.
// @opcode: 0x8066
message FileSystemRmdirResponse {
  uint64 requestId = 1;
  bool success = 2;
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Returned in response to `FileSystemRmdirRequest`.

## FileSystemUnlinkRequest

```protobuf
/// Removes a non-directory entry (regular file, symbolic link, or other file-like entry).
// @opcode: 0x8081
message FileSystemUnlinkRequest {
  uint64 requestId = 1;

  /// Path of the entry to remove, relative to the mount root.
  string path = 2;
}
```

### DESCRIPTION

Removes the entry at `path`. Mirrors POSIX `unlink(2)`: this operation MUST NOT be used on directories (use `FileSystemRmdirRequest` instead) and operates on the symbolic link itself rather than its target when `path` resolves to a symlink.

### IMPLEMENTATION NOTES

Common failure codes:

- `notFound` — the entry does not exist.
- `isDirectory` — `path` resolves to a directory; the requester SHOULD use `FileSystemRmdirRequest`.
- `accessDenied` — filesystem-level permissions reject the unlink.
- `readOnlyFilesystem` — the mount or filesystem is read-only.
- `busy` — the entry is in use and cannot be removed (e.g., Windows lock semantics).

Path validation rules and length limits are described in the `PATH HANDLING` appendix section.

## FileSystemUnlinkResponse

```protobuf
/// Response to a FileSystemUnlinkRequest.
// @opcode: 0x8082
message FileSystemUnlinkResponse {
  uint64 requestId = 1;
  bool success = 2;
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Returned in response to `FileSystemUnlinkRequest`.

## FileSystemRenameRequest

```protobuf
/// Renames or moves an entry within this mount.
// @opcode: 0x8083
message FileSystemRenameRequest {
  uint64 requestId = 1;

  /// Current path of the entry, relative to the mount root.
  string oldPath = 2;

  /// New path for the entry, relative to the mount root.
  string newPath = 3;
}
```

### DESCRIPTION

Renames (or moves within the same mount) the entry at `oldPath` to `newPath`. The operation mirrors POSIX `rename(2)`: when both paths reside on the same underlying filesystem, the rename is atomic.

If an entry already exists at `newPath`, the behavior follows POSIX semantics:

- A regular file at `newPath` is atomically replaced.
- A non-empty directory at `newPath` causes the rename to fail with `notEmpty`.
- A type mismatch (e.g., renaming a file over a directory) causes the rename to fail with `isDirectory` or `notDirectory` as appropriate.

### IMPLEMENTATION NOTES

Cross-mount renames are structurally impossible on this channel because each `fsaccess_mount` channel is bound to exactly one mount session. Renames whose paths land on different underlying filesystems within the same mount root (e.g., a mount root spanning bind mounts) MUST be rejected with `crossDevice`; the requester is expected to perform a copy-then-delete sequence in this case.

If `oldPath` and `newPath` resolve to the same entry, the operation MUST succeed as a no-op.

Path validation rules and length limits apply to both `oldPath` and `newPath`; see the `PATH HANDLING` appendix section.

## FileSystemRenameResponse

```protobuf
/// Response to a FileSystemRenameRequest.
// @opcode: 0x8084
message FileSystemRenameResponse {
  uint64 requestId = 1;
  bool success = 2;
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Returned in response to `FileSystemRenameRequest`.

## FileSystemFTruncateRequest

```protobuf
/// Sets the length of a file referenced by an open handle.
// @opcode: 0x8085
message FileSystemFTruncateRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;

  /// New length of the file, in bytes. If smaller than the current size, the file is truncated. If larger, the file is extended and the new region MUST read as zero bytes.
  uint64 length = 3;
}
```

### DESCRIPTION

Sets the length of the file referenced by `handleId`. Equivalent in intent to POSIX `ftruncate(2)`. The handle MUST have been opened with write access (`AccessMode.write` or `AccessMode.readWrite`); otherwise the operation MUST fail with `permissionDenied`.

When `length` exceeds the current size, the file is extended and the gap MUST be observable as zero bytes by subsequent reads. This mirrors POSIX behavior and is intended to provide deterministic semantics across platforms (Windows `SetFileInformationByHandle` with `FileEndOfFileInfo` is also expected to zero-fill, but historically some filesystems left the gap undefined; implementations on such filesystems MUST explicitly write zeros).

### IMPLEMENTATION NOTES

The choice of a handle-based truncate (rather than a path-based one) is intentional: it keeps permission checking, durability, and offset bookkeeping unified with the read/write path. Callers needing to truncate by path MUST `Open` the file, `FTruncate` it, and `Close` the handle.

## FileSystemFTruncateResponse

```protobuf
/// Response to a FileSystemFTruncateRequest.
// @opcode: 0x8086
message FileSystemFTruncateResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Populated when `success` is false. Common codes include `permissionDenied` (handle is read-only), `diskFull` (extension failed), and `fileTooLarge` (target length exceeds filesystem maximum).
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Returned in response to `FileSystemFTruncateRequest`.

## FileSystemRequestStreamReadRequest

```protobuf
/// Initiates a streaming read on an open file handle. The actual bytes are carried over a separately opened Transfer channel rather than inline on the fsaccess_mount channel.
// @opcode: 0x80A1
message FileSystemRequestStreamReadRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;

  /// Absolute offset within the file at which to begin streaming.
  uint64 offset = 3;

  /// Number of bytes to stream. The sentinel value `0xFFFFFFFFFFFFFFFF` means "stream until EOF". Any other value is the maximum number of bytes to send; the actual count MAY be smaller if EOF is reached first.
  uint64 length = 4;
}
```

### DESCRIPTION

Initiates a sequential read of `[offset, offset + length)` from `handleId`, where the actual bytes are pushed over a freshly opened Transfer channel rather than returned inline. This is the recommended path for known-sequential bulk reads (whole-file downloads, large range copies); for random access or small reads, callers SHOULD use `FileSystemReadRequest` instead so that handle setup and Transfer channel teardown overhead is avoided.

The exposing peer is the data sender. After this request succeeds, the exposing peer opens a Transfer channel whose virtual path is `/noctiluca/fsaccess_mount/{transferId}` and pushes bytes through it until EOF or until `length` bytes have been delivered, then closes the Transfer channel.

### IMPLEMENTATION NOTES

Inline reads on the same handle remain permitted while the stream is in progress; consistency between the two paths is the requester's responsibility.

Closing the underlying file handle (`FileSystemCloseRequest`) aborts the stream and the associated Transfer channel.

If `offset` is at or beyond the current end of file, the request MUST succeed and the resulting Transfer channel carries zero bytes before closing. This mirrors the behavior of an inline `Read` at EOF.

## FileSystemRequestStreamReadResponse

```protobuf
/// Response to a FileSystemRequestStreamReadRequest.
// @opcode: 0x80A2
message FileSystemRequestStreamReadResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Identifier used to correlate the upcoming Transfer channel with this stream request. Issued by the exposing peer. Valid only when `success` is true.
  /// The exposing peer opens a Transfer channel addressed at the virtual path `/noctiluca/fsaccess_mount/{transferId}` shortly after sending this response.
  SRUUID transferId = 3;

  /// Populated when `success` is false. Common codes include `invalidHandle`, `permissionDenied` (handle was opened write-only), and `invalidArgument` (e.g., negative-equivalent length).
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 4;
}
```

### DESCRIPTION

Sent by the exposing peer to acknowledge the stream request. A successful response is a promise that the corresponding Transfer channel will be opened soon. Mid-stream failures (e.g., low-level `ioError`) are NOT reported back through fsaccess_mount; they propagate through the Transfer channel's own abort mechanism.

## FileSystemRequestStreamWriteRequest

```protobuf
/// Initiates a streaming write on an open file handle. The actual bytes are carried over a separately opened Transfer channel rather than inline on the fsaccess_mount channel.
// @opcode: 0x80A3
message FileSystemRequestStreamWriteRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;

  /// Absolute offset within the file at which to begin writing. Ignored when the handle was opened with `OpenFlags.append`; in that case the stream lands at the current end of file and grows as bytes arrive.
  uint64 offset = 3;

  /// Maximum number of bytes the requester intends to send. The sentinel value `0xFFFFFFFFFFFFFFFF` means "stream until the requester closes the Transfer channel" (open-ended upload). Any other value is an upper bound; the requester MAY close the Transfer channel earlier (signalling end-of-stream) and MUST NOT send more bytes than `length`.
  uint64 length = 4;
}
```

### DESCRIPTION

Initiates a sequential write to `handleId` starting at `offset`, where the actual bytes are pushed over a freshly opened Transfer channel rather than supplied inline. This is the recommended path for known-sequential bulk writes (whole-file uploads, large range copies); for random access or small writes, callers SHOULD use `FileSystemWriteRequest` instead.

The requester is the data sender. After this request succeeds, the requester opens a Transfer channel whose virtual path is `/noctiluca/fsaccess_mount/{transferId}` and pushes bytes through it.

### IMPLEMENTATION NOTES

The exposing peer MAY abort the Transfer channel mid-stream when local conditions force it (e.g., `diskFull`, `quotaExceeded`). The abort reason is conveyed through the Transfer channel's abort mechanism; partially written bytes are NOT rolled back, mirroring inline write semantics.

If the requester sends more bytes than `length` (when `length` is not the open-ended sentinel), the exposing peer MUST treat this as a protocol violation, abort the Transfer channel, and SHOULD close the fsaccess_mount channel with `ServerNotice(protocolViolation)`.

Inline writes on the same handle remain permitted while the stream is in progress; consistency between the two paths is the requester's responsibility.

Closing the underlying file handle (`FileSystemCloseRequest`) aborts the stream and the associated Transfer channel.

## FileSystemRequestStreamWriteResponse

```protobuf
/// Response to a FileSystemRequestStreamWriteRequest.
// @opcode: 0x80A4
message FileSystemRequestStreamWriteResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Identifier used to correlate the upcoming Transfer channel with this stream request. Issued by the exposing peer. Valid only when `success` is true.
  /// The requester opens a Transfer channel addressed at the virtual path `/noctiluca/fsaccess_mount/{transferId}` shortly after receiving this response.
  SRUUID transferId = 3;

  /// Populated when `success` is false. Common codes include `invalidHandle`, `permissionDenied` (handle was opened read-only), `readOnlyFilesystem`, and `invalidArgument`.
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 4;
}
```

### DESCRIPTION

Sent by the exposing peer to acknowledge the stream request. A successful response is a promise that the exposing peer is ready to consume bytes on the Transfer channel addressed at `/noctiluca/fsaccess_mount/{transferId}`. Mid-stream failures propagate through the Transfer channel's own abort mechanism rather than through a follow-up fsaccess_mount message.

## FileSystemLockRequest

```protobuf
/// Acquires a byte-range advisory lock on an open file handle. The operation is strictly non-blocking: if any conflicting lock currently exists on overlapping bytes, the request MUST fail immediately with `wouldBlock`.
// @opcode: 0x80C1
message FileSystemLockRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;

  /// Type of lock to acquire.
  // @constset: LockType
  uint32 type = 3;

  /// Absolute offset within the file at which the locked range begins.
  uint64 offset = 4;

  /// Length of the locked range in bytes. The sentinel value `0xFFFFFFFFFFFFFFFF` means "from `offset` to the maximum possible end of file" (whole-file lock when `offset = 0`). A value of zero is invalid and MUST fail with `invalidArgument`.
  uint64 length = 5;
}
```

### DESCRIPTION

Acquires an advisory byte-range lock on the file referenced by `handleId`. The operation is strictly non-blocking: callers requiring blocking semantics MUST poll by reissuing the request after a `wouldBlock` failure.

The lock is bound to the supplied `handleId` and is automatically released when the handle is closed (`FileSystemCloseRequest`) or when the surrounding `fsaccess_mount` channel closes for any reason. See the `LOCK SEMANTICS` appendix for the full ownership and lifetime model.

### IMPLEMENTATION NOTES

Acquiring a `shared` lock requires the handle to have been opened with `read` or `readWrite` access; acquiring an `exclusive` lock requires `write` or `readWrite` access. A mismatch MUST be reported as `permissionDenied`.

A `Lock` request whose `(handleId, type, offset, length)` exactly matches a currently held lock MUST fail with `invalidArgument`. The protocol does not provide an atomic shared-to-exclusive upgrade; callers needing to change a lock's type MUST explicitly `Unlock` and re-acquire, accepting the brief race window during which another caller may interleave.

The exposing peer MUST attempt to acquire the lock at the host OS level using the platform-native non-blocking call (POSIX `fcntl(F_SETLK)`, Windows `LockFileEx` with `LOCKFILE_FAIL_IMMEDIATELY`, etc.). When the host platform does not provide handle-bound byte-range locks with the semantics expected by the protocol, the exposing peer MUST signal this by setting `FileSystemMountResponse.supportsLocks = false` rather than silently substituting a weaker semantics; in that case, this request MUST fail with `notSupported`.

## FileSystemLockResponse

```protobuf
/// Response to a FileSystemLockRequest.
// @opcode: 0x80C2
message FileSystemLockResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Populated when `success` is false. Common codes include `wouldBlock` (a conflicting lock exists), `notSupported` (the mount session reports `supportsLocks = false`), `invalidHandle`, `invalidArgument` (re-lock on an identical range, `length = 0`, etc.), and `permissionDenied` (the requested lock type exceeds the handle's `accessMode`).
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Returned in response to a `FileSystemLockRequest`. A `wouldBlock` failure is the canonical "lock contended" signal and is a normal, expected outcome rather than a protocol-level error.

## FileSystemUnlockRequest

```protobuf
/// Releases byte-range locks held on an open file handle.
// @opcode: 0x80C3
message FileSystemUnlockRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;

  /// Absolute offset within the file at which the unlock range begins.
  uint64 offset = 3;

  /// Length of the unlock range in bytes. The sentinel value `0xFFFFFFFFFFFFFFFF` means "from `offset` to the maximum possible end of file". A value of zero is invalid and MUST fail with `invalidArgument`.
  uint64 length = 4;
}
```

### DESCRIPTION

Releases the portion of any byte-range locks held on `handleId` that overlaps `[offset, offset + length)`. Mirrors POSIX `fcntl` with `F_UNLCK`: the released range MAY be a strict subset of an existing lock (in which case the lock is split into the surrounding non-overlapping portions), a strict superset (in which case the entire lock is released), or have no overlap with any existing lock (in which case the operation is a no-op).

An `Unlock` covering bytes that the handle does not currently hold a lock on MUST succeed silently and MUST NOT be reported as an error. This matches POSIX semantics and avoids forcing callers to track exact lock ranges for cleanup.

### IMPLEMENTATION NOTES

The exposing peer MUST translate the released range to the platform-native call (POSIX `fcntl(F_SETLK)` with `F_UNLCK`, Windows `UnlockFileEx`). Platforms whose native unlock call rejects "not held" ranges (notably Windows `UnlockFileEx`, which returns `ERROR_NOT_LOCKED`) MUST suppress that error and report `success = true`, so that protocol semantics remain platform-independent.

## FileSystemUnlockResponse

```protobuf
/// Response to a FileSystemUnlockRequest.
// @opcode: 0x80C4
message FileSystemUnlockResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// Populated when `success` is false. Common codes include `notSupported` (the mount session reports `supportsLocks = false`), `invalidHandle`, and `invalidArgument` (`length = 0`).
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 3;
}
```

### DESCRIPTION

Returned in response to a `FileSystemUnlockRequest`.

## FileSystemTestLockRequest

```protobuf
/// Probes whether a byte-range lock could be acquired without actually acquiring it. Equivalent in intent to POSIX `fcntl(F_GETLK)`.
// @opcode: 0x80C5
message FileSystemTestLockRequest {
  uint64 requestId = 1;
  uint64 handleId = 2;

  /// Type of lock that would be attempted.
  // @constset: LockType
  uint32 type = 3;

  /// Absolute offset within the file at which the candidate range begins.
  uint64 offset = 4;

  /// Length of the candidate range in bytes. The sentinel value `0xFFFFFFFFFFFFFFFF` means "from `offset` to the maximum possible end of file". A value of zero is invalid and MUST fail with `invalidArgument`.
  uint64 length = 5;
}
```

### DESCRIPTION

Probes whether a `FileSystemLockRequest` with the same `(type, offset, length)` would succeed at the moment of the test, without actually taking the lock.

The result is advisory: even when `canAcquire` is true, a subsequent `FileSystemLockRequest` MAY still fail with `wouldBlock` if another caller acquires a conflicting lock in the interval between the test and the acquire. This message is intended for UI prompts and diagnostics, not as a substitute for handling `wouldBlock` on the actual `Lock`.

## FileSystemTestLockResponse

```protobuf
/// Response to a FileSystemTestLockRequest.
// @opcode: 0x80C6
message FileSystemTestLockResponse {
  uint64 requestId = 1;
  bool success = 2;

  /// True when a `FileSystemLockRequest` with the same `type`, `offset`, and `length` would be expected to succeed at the moment of the test. Valid only when `success` is true.
  bool canAcquire = 3;

  /// Type of an existing lock that would conflict with the requested acquisition. Populated only when `success` is true and `canAcquire` is false.
  // @constset: LockType
  uint32 conflictingType = 4;

  /// Absolute offset of an existing conflicting lock. Populated only when `success` is true and `canAcquire` is false. The reported range MAY be wider than the requested range when a single existing lock spans more bytes than were probed.
  uint64 conflictingOffset = 5;

  /// Length of an existing conflicting lock. Populated only when `success` is true and `canAcquire` is false. Uses the same sentinel encoding as `FileSystemLockRequest.length`.
  uint64 conflictingLength = 6;

  /// Populated when `success` is false. Common codes include `notSupported` (the mount session reports `supportsLocks = false`), `invalidHandle`, and `invalidArgument` (`length = 0`).
  optional sirius.msgdef.v1.channels.fsaccess.ErrorInfo error = 7;
}
```

### DESCRIPTION

Returned in response to a `FileSystemTestLockRequest`.

When a conflicting lock exists, only its `type`, `offset`, and `length` are reported. The identity of the conflicting lock owner (handle, channel, or process) is intentionally NOT exposed, both to keep the wire surface narrow and to avoid leaking information across mount sessions or across processes on the host.

When multiple existing locks conflict with the requested range, the exposing peer MAY report any one of them; it is not required to coalesce conflicting locks or pick a deterministic representative. Consumers MUST NOT rely on a particular choice.

---

# CHANNEL ARGUMENTS

This channel receives the following arguments via the `args` field of the `ChannelStartRequest` message:

```typescript
type FileSystemAccessMountChannelArgs = [
    sessionId: string, // string representation of the SRUUID issued by the matching FileSystemMountResponse
];
```

The exposing peer opens this channel immediately after sending `FileSystemMountResponse(success = true, sessionId)` on the parent fsaccess control channel. The requester accepts the channel only when `args[0]` matches a `sessionId` it has just been granted; an unmatched `sessionId` MUST cause the requester to reject the channel with `ChannelStartResponse(success = false)`.

---

# CHANNEL LIFECYCLE

The fsaccess_mount channel layers on top of the standard Sirius channel lifecycle (described in `general.mdproto.md`) and is bound by additional rules linking it to its parent fsaccess control channel.

## Parent Control Channel Required

An `fsaccess_mount` channel MUST NOT be opened independently of an active fsaccess control channel on the same Sirius connection. The exposing peer is the only side permitted to open this channel, and it does so only as a follow-up to a successful `FileSystemMountRequest` exchange. A `ChannelStartRequest` for `fsaccess_mount` initiated by the wrong side, or initiated without a matching `sessionId` granted on the control channel, MUST be rejected at the channel-management level.

## Authentication Gate

Like the parent fsaccess control channel, fsaccess_mount MUST NOT exchange messages before the Sirius authentication phase has completed. If a fsaccess_mount message somehow reaches the channel before authentication completes, the channel MUST be closed with `ServerNotice(protocolViolation)`; the notice's `message` field SHOULD identify the violated phase ordering.

## Cascade Cleanup

The control channel and its mount channels are bound by a strict parent / child cleanup relationship.

- When the fsaccess control channel closes, every `fsaccess_mount` channel opened against it MUST be closed by both peers. All handles and in-progress stream transfers are released as part of those individual mount-channel closes.
- When an `fsaccess_mount` channel closes — by graceful close, an explicit `FileSystemUnmountRequest` on the control channel, the requester closing it as an implicit unmount, or transport failure — both peers MUST treat the corresponding mount session as released. Open handles bound to the channel are closed; in-progress stream transfers (`FileSystemRequestStreamRead` / `FileSystemRequestStreamWrite`) are aborted; pending response promises are completed with `channelClosing`.

The requester SHOULD treat any locally cached `handleId` value as invalid after the corresponding `fsaccess_mount` channel has closed, and MUST NOT attempt to reuse it on a freshly opened mount channel — even if a new mount of the same entry happens to issue identical-looking handles.

---

# PATH HANDLING

All path strings transmitted on the fsaccess_mount channel are interpreted under the following rules. The validation pipeline applies to every `path`, `oldPath`, `newPath`, and any future field carrying a file-system path. A failure at any step MUST cause the surrounding operation to fail with `invalidPath`, except where a different code is explicitly noted.

## Wire Encoding

- Paths MUST be encoded in valid **UTF-8** and MUST NOT contain NUL (`U+0000`) bytes.
- The path separator is **forward slash (`/`)**, regardless of the underlying host operating system. Backslashes carry no special meaning and MUST be treated as literal characters within a path component.
- Paths are interpreted relative to the **mount root** that the surrounding channel is bound to. A leading `/` is permitted as a stylistic affordance and MUST be treated as identical to the same path without the leading slash; both forms refer to the mount root, NOT to the host filesystem root.

## Validation Pipeline

The exposing peer MUST apply the following steps in order:

1. **UTF-8 check.** Reject if the byte sequence is not valid UTF-8.
2. **NUL check.** Reject if any byte is `0x00`.
3. **Component split and normalization.** Split on `/`. Drop empty components produced by leading, trailing, or repeated slashes. Drop `.` components. Resolve `..` components by popping the previous component from the in-progress result; if `..` would pop past the start (i.e., escape the mount root), reject the path.
4. **Length check.** After normalization, fail with `pathTooLong` if the rejoined path exceeds 4096 bytes. Path-bearing requests are additionally bounded by the protocol-wide 16 MiB frame size limit, so no separate channel-close threshold is required.
5. **Host-path resolution.** Join the normalized path with the mount's host root to produce an absolute host path.
6. **Subtree re-check.** After any symlink resolution that the operation actually performs (e.g., `Stat` with `followSymlinks = true`), the resolved host path MUST still lie within the mount's host root subtree. Otherwise the operation MUST fail with `policyViolation`.

## Symlinks

- A symbolic link whose target resolves outside the mount root subtree MUST NOT be transparently followed by the exposing peer. Operations that would otherwise return the resolved entry MUST fail with `policyViolation` instead.
- A symbolic link whose target resolves inside the mount root subtree is permitted and is followed transparently when the operation requests link following.
- The `noFollowSymlinks` flag on `FileSystemOpenRequest` causes the open to fail with `loop` when the final path component is a symlink, regardless of where the link's target resides.

## Hard Links

Hard links to entries inside the mount root are followed transparently because at the filesystem level they are indistinguishable from the original entry. The protocol does NOT attempt to detect hard links that originate from outside the mount root subtree; exposing peers that need stronger isolation SHOULD prevent such links from being created in the exposed subtree out-of-band (filesystem permissions, dedicated volumes, etc.).

---

# LOCK SEMANTICS

This appendix defines the ownership, lifetime, and platform-binding rules that govern the byte-range lock operations (`FileSystemLockRequest`, `FileSystemUnlockRequest`, `FileSystemTestLockRequest`).

## Capability Negotiation

Lock operations are gated by the `supportsLocks` field on `FileSystemMountResponse` (see `fsaccess.mdproto.md`). When the exposing peer reports `supportsLocks = false`, all three lock operations on the corresponding `fsaccess_mount` channel MUST fail with `notSupported`, and the consuming peer SHOULD avoid sending them.

The intended use of `supportsLocks = false` is to signal exposing peers running on platforms or sandboxes where byte-range file locks are unavailable or unreliable (for example, certain mobile sandboxes that expose filesystem access through proxies that bypass the host kernel's lock tables). In such cases, the consuming peer MAY simulate locks in its own address space if its application semantics require some form of mutual exclusion across its own callers; such simulation provides no cross-peer guarantee and is entirely out of scope for this protocol.

## Range Encoding

A lock covers the half-open byte range `[offset, offset + length)`. The sentinel `length = 0xFFFFFFFFFFFFFFFF` denotes "from `offset` to the maximum possible end of file", which becomes a whole-file lock when combined with `offset = 0`. The exposing peer MUST translate the sentinel into the platform-equivalent representation (POSIX `l_len = 0`, Windows `(MAXDWORD, MAXDWORD)`) when issuing the host syscall.

A `length` of zero is invalid for all three lock operations and MUST fail with `invalidArgument`. This rule deliberately diverges from POSIX `fcntl`, which interprets `l_len = 0` as "until end of file"; the protocol uses the explicit sentinel instead so that there is exactly one wire encoding for a whole-file or open-ended lock.

## Ownership and Lifetime

Locks are bound to the `handleId` that acquired them. The exposing peer MUST release a handle's locks under any of the following conditions:

- The handle is explicitly closed via `FileSystemCloseRequest`.
- The handle is implicitly closed because the surrounding `fsaccess_mount` channel closes (graceful close, transport failure, parent fsaccess control-channel cascade, or implicit unmount by the requester).
- The mount session is released via `FileSystemUnmountRequest` on the parent fsaccess control channel.

There is no separate lock-leasing or heartbeat mechanism: lock lifetime is exactly the lifetime of the handle that holds it. Because handle lifetime is itself bounded by channel and transport lifetime, transport failure transitively releases all locks held by a peer, eliminating cross-session lock leakage without requiring an additional liveness protocol.

## Re-Lock and Upgrade

A `FileSystemLockRequest` whose `(handleId, type, offset, length)` exactly matches a currently held lock MUST fail with `invalidArgument`. The protocol does not provide an atomic shared-to-exclusive (or exclusive-to-shared) upgrade; callers needing to change a lock's type MUST explicitly `Unlock` the existing range and re-acquire it, accepting the brief race window during which another caller may interleave.

A `Lock` request whose range overlaps but does not exactly match a currently held lock on the same handle MAY be accepted by the exposing peer when the host OS would accept the corresponding native call (for example, POSIX `fcntl` allows extending or splitting a lock with subsequent calls). The protocol leaves the exact treatment of such cases to the host syscall semantics.

## Range Overlap on Unlock

`Unlock` follows POSIX `fcntl(F_UNLCK)` semantics: the released range MAY be a strict subset of an existing lock (in which case the lock is split into the surrounding non-overlapping portions), a strict superset (in which case the entire lock is released), or have no overlap with any existing lock (in which case the operation is a no-op and MUST succeed).

Platforms whose native unlock call rejects "not held" ranges (notably Windows `UnlockFileEx`, which returns `ERROR_NOT_LOCKED`) MUST suppress that error and report `success = true`, so that protocol semantics remain platform-independent.

## Mandatory vs Advisory

The protocol does not distinguish between advisory and mandatory locks. The exposing peer MUST acquire whatever the host OS provides:

- POSIX (Linux, macOS): advisory locks via `fcntl(F_SETLK)`, or OFD locks where preferred for handle-bound semantics.
- Windows: mandatory locks via `LockFileEx` with `LOCKFILE_FAIL_IMMEDIATELY`.

Consequently, the visible effect of a held lock on operations from outside the fsaccess channel (for example, an unrelated process on the host) varies by platform. Within a single mount session, however, a held exclusive lock MUST cause overlapping operations from other handles in the same session to behave consistently with the host OS semantics for that lock type.

---

# COMPRESSION

This appendix defines how the compression method negotiated on the parent fsaccess control channel applies to byte payloads carried on this mount channel. The choice itself is made on the parent control channel; this appendix specifies what that choice means on the data plane.

## Capability Negotiation

The compression method is selected during mount setup, by the exposing peer's pick from `FileSystemMountRequest.proposedCompressionMethods` reported in `FileSystemMountResponse.selectedCompressionMethod` (see `fsaccess.mdproto.md`). The selected method is stable for the lifetime of the mount session and applies uniformly to every byte payload carried on the corresponding `fsaccess_mount` channel itself. Transfer channels spawned for Stream IO inherit the proposal but follow their own negotiation; see Stream IO below.

When `selectedCompressionMethod = none` (or empty, treated as equivalent), the data plane MUST behave exactly as it did before this revision: no compression is applied, and every `bytes` field carries raw payload as-is.

## Inline Read / Write

When `selectedCompressionMethod` is non-`none`, the following fields carry compressed bytes:

- `FileSystemReadResponse.data` (0x8022)
- `FileSystemWriteRequest.data` (0x8023)

Each such message MUST carry exactly one self-contained, independently decodable compressed frame in the negotiated format (a Zstandard frame for `zstd`, a raw DEFLATE block sequence for `deflate`, a Brotli stream for `brotli`). Cross-message shared dictionaries or split frames are NOT permitted; receivers MUST be able to decompress each message in isolation.

The protocol-wide 16 MiB frame size limit on the size of `data` applies to the **wire bytes** of the field — that is, the compressed bytes when compression is in effect. The uncompressed length is unbounded by the protocol but is in practice bounded by the read or write request's `length` parameter.

The `length` parameter of `FileSystemReadRequest` (0x8021) and the size of `FileSystemWriteRequest.data` are interpreted in **uncompressed** terms, mirroring POSIX `read(2)` / `write(2)` semantics. Receivers MUST decompress before measuring the produced bytes against the requested length. Senders SHOULD aim to keep the uncompressed payload within the request's `length` bound; the protocol does not require an exact match because some compression formats can produce frames whose decoded size differs from the requested size by a small framing margin.

## Stream IO

Stream IO (`FileSystemRequestStreamReadRequest` / `FileSystemRequestStreamWriteRequest`) hands off the bulk byte transfer to a freshly opened Transfer channel. The mount session's compression method is propagated as the Transfer channel's **proposal**, but the Transfer channel runs its own per-channel negotiation:

- The peer that opens the Transfer channel (the exposing peer for stream reads, the consuming peer for stream writes) MUST set the Transfer channel's `transferArgs` slot to `compress=<selectedCompressionMethod>` — that is, a single-candidate proposal carrying exactly the mount session's negotiated method. When `selectedCompressionMethod = none`, the opener MAY omit the `compress=` token entirely (equivalent to proposing `none` only).
- The receiver on the Transfer channel acknowledges with `TransferReady` per the Transfer channel's own rules: it MAY pick the proposed candidate, or it MAY pick `none` (for example, when local policy disables compression for this Transfer channel). The receiver MUST NOT pick any method that was not in the proposal and is not `none`.

When the receiver picks a method that differs from the mount session's `selectedCompressionMethod`, only the Transfer channel itself is affected: subsequent inline IO on the parent `fsaccess_mount` channel continues to use the mount session's method. This avoids escalating a per-Transfer-channel policy decision into a mount-wide renegotiation while still letting an individual stream opt out of compression when required.

## Failure Handling

A decompression failure on either side (malformed frame, unknown identifier, truncated payload) is treated as a corrupted operation:

- For inline read / write, the affected operation MUST fail with `ioError`, and subsequent operations on the same handle remain valid.
- For stream IO, the affected Transfer channel MUST be aborted as defined by the Transfer channel's own abort mechanism; partial bytes already written to disk are NOT rolled back, mirroring inline write semantics for partial writes.

In both cases the underlying mount channel and parent control channel remain open: the protocol does not escalate a single corrupted payload to a session-level fault.
