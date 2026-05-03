# Channel Opcode Convention

Defines the opcode convention used by Feature Channels in the Sirius protocol.

> **Scope**: These rules are guidelines strictly followed by libsirius maintainers.
> Third-party developers extending channels may use them as a reference.

---

## 1. Basic Principles

### 1.1 Opcode Ranges

| Range | Purpose |
|-------|---------|
| `0x0000 ~ 0x7FFF` | **Global Opcode** - Main channel only (handshake, authentication, channel management) |
| `0x8000 ~ 0xFFFF` | **Feature Channel Opcode** - Used by each feature channel |

### 1.2 Inter-Channel Independence

**Each channel uses a separate transport stream, so there is no risk of opcode collision between channels.**

For example:
- `0x8001` in the HIDIO channel and `0x8001` in the Projection Data channel may refer to entirely different messages
- Even with the same opcode value, channels are completely independent

---

## 2. Group + Sequential Scheme

### 2.1 Group Structure

Within each channel, opcodes are divided into **functional groups**, and **sequential numbers** are assigned within each group.

```
Group 1: 0x8001 ~ 0x801F (32 slots)
Group 2: 0x8021 ~ 0x803F (32 slots)
Group 3: 0x8041 ~ 0x805F (32 slots)
Group 4: 0x8061 ~ 0x807F (32 slots)
Group 5: 0x8081 ~ 0x809F (32 slots)
Group 6: 0x80A1 ~ 0x80BF (32 slots)
Group 7: 0x80C1 ~ 0x80DF (32 slots)
...
```

- Group size: **0x20 (32)**
- Group start address: `0x8001`, `0x8021`, `0x8041`, ... (`0x8000 + 0x20*n + 1`)

### 2.2 Message Placement Within Groups

Messages are placed using a **fully sequential scheme**, with related messages adjacent:

```
0x8001: FooRequest
0x8002: FooResponse
0x8003: BarRequest
0x8004: BarResponse
0x8005: BazEvent
...
```

**Advantages**:
- Related Request/Response pairs are adjacent, making the layout intuitive
- The functional group can be identified just by looking at the opcode

### 2.3 Group Purpose Guide (Recommended)

Group purposes can be freely defined depending on the channel's characteristics. The following is a reference example:

```
Group 1: Core/session-related messages for the channel
Group 2: Event/notification-related messages
Group 3: Query/retrieval-related messages
Group 4: Control/manipulation-related messages
Group 5+: Reserved for extensions
```

---

## 3. Per-Channel Opcode Allocation

### 3.1 HIDIO Channel

```
┌─────────────────────────────────────────────────────────────────┐
│ Group 1 (0x8001~0x801F): Core                                    │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ HIDIOPacket                                           │
└─────────┴───────────────────────────────────────────────────────┘
```

### 3.2 Projection Channel

The Projection channel encompasses Display, Window, Cursor, and Audio related functionality.

```
┌─────────────────────────────────────────────────────────────────┐
│ Group 1 (0x8001~0x801F): Projection Session                      │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ ProjectionRequest                                     │
│ 0x8002  │ StopProjectionRequest                                 │
│ 0x8003  │ ProjectionPerformanceReport                           │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 2 (0x8021~0x803F): Projection Session Events               │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8021  │ ProjectionSessionCreatedEvent                         │
│ 0x8022  │ ProjectionSessionCreationFailedEvent                  │
│ 0x8023  │ ProjectionSessionChangedEvent                         │
│ 0x8024  │ ProjectionSessionEndedEvent                           │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 3 (0x8041~0x805F): Display Management                      │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8041  │ DisplayListRequest                                    │
│ 0x8042  │ DisplayListResponse                                   │
│ 0x8043  │ SubscribeDisplayChangesRequest                        │
│ 0x8044  │ SubscribeDisplayChangesResponse                       │
│ 0x8045  │ UnsubscribeDisplayChangesRequest                      │
│ 0x8046  │ UnsubscribeDisplayChangesResponse                     │
│ 0x8047  │ DisplayChangedEvent                                   │
│ 0x8048  │ DisplayTransactionRequest                             │
│ 0x8049  │ DisplayTransactionResponse                            │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 4 (0x8061~0x807F): Window Query                            │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8061  │ WindowListRequest                                     │
│ 0x8062  │ WindowListResponse                                    │
│ 0x8063  │ GetWindowInfoRequest                                  │
│ 0x8064  │ GetWindowInfoResponse                                 │
│ 0x8065  │ GetWindowIconRequest                                  │
│ 0x8066  │ GetWindowIconResponse                                 │
│ 0x8067  │ GetWindowThumbnailRequest                             │
│ 0x8068  │ GetWindowThumbnailResponse                            │
│ 0x8069  │ SubscribeWindowEventsRequest                          │
│ 0x806A  │ SubscribeWindowEventsResponse                         │
│ 0x806B  │ UnsubscribeWindowEventsRequest                        │
│ 0x806C  │ UnsubscribeWindowEventsResponse                       │
│ 0x806D  │ WindowChangedEvent                                    │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 5 (0x8081~0x809F): Window Manipulation                     │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8081  │ WindowManipulationRequest                             │
│ 0x8082  │ WindowManipulationResponse                            │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 6 (0x80A1~0x80BF): Cursor                                  │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x80A1  │ SubscribeCursorEventsRequest                          │
│ 0x80A2  │ SubscribeCursorEventsResponse                         │
│ 0x80A3  │ UnsubscribeCursorEventsRequest                        │
│ 0x80A4  │ UnsubscribeCursorEventsResponse                       │
│ 0x80A5  │ CursorEvent                                           │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 7 (0x80C1~0x80DF): Audio Projection                        │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x80C1  │ AudioProjectionRequest                                │
│ 0x80C2  │ StopAudioProjectionRequest                            │
│ 0x80C3  │ AudioSessionCreatedEvent                              │
│ 0x80C4  │ AudioSessionCreationFailedEvent                       │
│ 0x80C5  │ AudioSessionChangedEvent                              │
│ 0x80C6  │ AudioSessionEndedEvent                                │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 8 (0x80E1~0x80FF): Application Management (appman)         │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x80E1  │ ApplicationListRequest                                │
│ 0x80E2  │ ApplicationListResponse                               │
│ 0x80E3  │ ApplicationLaunchRequest                              │
│ 0x80E4  │ ApplicationLaunchResponse                             │
│ 0x80E5  │ ApplicationTerminateRequest                           │
│ 0x80E6  │ ApplicationTerminateResponse                          │
│ 0x80E7  │ SubscribeApplicationEventsRequest                     │
│ 0x80E8  │ SubscribeApplicationEventsResponse                    │
│ 0x80E9  │ UnsubscribeApplicationEventsRequest                   │
│ 0x80EA  │ UnsubscribeApplicationEventsResponse                  │
│ 0x80EB  │ ApplicationChangedEvent                               │
│ 0x80F1  │ StartAppStreamRequest                                 │
│ 0x80F2  │ StartAppStreamResponse                                │
│ 0x80F3  │ StopAppStreamRequest                                  │
│ 0x80F4  │ StopAppStreamResponse                                 │
│ 0x80F5  │ AppStreamWindowEvent                                  │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 9 (0x8101~0x811F): Accessibility                           │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8101  │ GetAccessibilityTreeRequest                           │
│ 0x8102  │ GetAccessibilityTreeResponse                          │
│ 0x8103  │ SubscribeAccessibilityTreeUpdatesRequest              │
│ 0x8104  │ SubscribeAccessibilityTreeUpdatesResponse             │
│ 0x8105  │ UnsubscribeAccessibilityTreeUpdatesRequest            │
│ 0x8106  │ UnsubscribeAccessibilityTreeUpdatesResponse           │
│ 0x8107  │ AccessibilityTreeUpdateEvent                          │
│ 0x8108  │ DispatchActionRequest                                 │
│ 0x8109  │ DispatchActionResponse                                │
└─────────┴───────────────────────────────────────────────────────┘
```

### 3.3 Projection Data Channel

```
┌─────────────────────────────────────────────────────────────────┐
│ Group 1 (0x8001~0x801F): Frame Data                              │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ FrameDataHeader                                       │
│ 0x8002  │ CodecParameterSetMessage                              │
└─────────┴───────────────────────────────────────────────────────┘
```

### 3.4 Clipboard Channel

```
┌─────────────────────────────────────────────────────────────────┐
│ Group 1 (0x8001~0x801F): Core                                    │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ ClipboardEvent                                        │
└─────────┴───────────────────────────────────────────────────────┘
```

### 3.5 File-System Access Control Channel (fsaccess)

The File-System Access (fsaccess) control channel is the discovery-and-consent plane of the file-system access feature. It exposes file systems on the remote peer in a manner inspired by RDP's `\\tsclient\C` drive redirection, and is split across a single functional group covering listing, mount, and unmount. Per-mount data-plane operations live on the companion `fsaccess_mount` channel (see Section 3.6).

```
┌─────────────────────────────────────────────────────────────────┐
│ Group 1 (0x8001~0x801F): Mount Lifecycle                         │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ FileSystemListRequest                                 │
│ 0x8002  │ FileSystemListResponse                                │
│ 0x8003  │ FileSystemMountRequest                                │
│ 0x8004  │ FileSystemMountResponse                               │
│ 0x8005  │ FileSystemUnmountRequest                              │
│ 0x8006  │ FileSystemUnmountResponse                             │
└─────────┴───────────────────────────────────────────────────────┘
```

### 3.6 File-System Access Mount Channel (fsaccess_mount)

The fsaccess_mount channel carries the per-mount data plane of the file-system access feature. The exposing peer opens one fsaccess_mount channel per granted mount session (mirroring the relationship between `projection` and `projection_data`). The channel surface is split across six functional groups: handle lifecycle (Group 1), bounded inline I/O (Group 2), metadata queries (Group 3), directory operations (Group 4), file operations (Group 5), and an opt-in handoff to the Transfer channel for sequential bulk transfer (Group 6).

```
┌─────────────────────────────────────────────────────────────────┐
│ Group 1 (0x8001~0x801F): Handle Lifecycle                        │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ FileSystemOpenRequest                                 │
│ 0x8002  │ FileSystemOpenResponse                                │
│ 0x8003  │ FileSystemCloseRequest                                │
│ 0x8004  │ FileSystemCloseResponse                               │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 2 (0x8021~0x803F): Inline I/O                              │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8021  │ FileSystemReadRequest                                 │
│ 0x8022  │ FileSystemReadResponse                                │
│ 0x8023  │ FileSystemWriteRequest                                │
│ 0x8024  │ FileSystemWriteResponse                               │
│ 0x8025  │ FileSystemFlushRequest                                │
│ 0x8026  │ FileSystemFlushResponse                               │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 3 (0x8041~0x805F): Stat                                    │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8041  │ FileSystemStatRequest                                 │
│ 0x8042  │ FileSystemStatResponse                                │
│ 0x8043  │ FileSystemFStatRequest                                │
│ 0x8044  │ FileSystemFStatResponse                               │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 4 (0x8061~0x807F): Directory Operations                    │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8061  │ FileSystemReadDirRequest                              │
│ 0x8062  │ FileSystemReadDirResponse                             │
│ 0x8063  │ FileSystemMkdirRequest                                │
│ 0x8064  │ FileSystemMkdirResponse                               │
│ 0x8065  │ FileSystemRmdirRequest                                │
│ 0x8066  │ FileSystemRmdirResponse                               │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 5 (0x8081~0x809F): File Operations                         │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8081  │ FileSystemUnlinkRequest                               │
│ 0x8082  │ FileSystemUnlinkResponse                              │
│ 0x8083  │ FileSystemRenameRequest                               │
│ 0x8084  │ FileSystemRenameResponse                              │
│ 0x8085  │ FileSystemFTruncateRequest                            │
│ 0x8086  │ FileSystemFTruncateResponse                           │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ Group 6 (0x80A1~0x80BF): Stream Handoff                          │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x80A1  │ FileSystemRequestStreamReadRequest                    │
│ 0x80A2  │ FileSystemRequestStreamReadResponse                   │
│ 0x80A3  │ FileSystemRequestStreamWriteRequest                   │
│ 0x80A4  │ FileSystemRequestStreamWriteResponse                  │
└─────────┴───────────────────────────────────────────────────────┘
```

---

## 4. Guide for Adding New Messages

### 4.1 Adding to an Existing Group

1. Check the last used opcode in the group
2. Assign the next sequential number
3. Ensure you do not exceed the group boundary (32 slots)

**Example**: Adding a new message to the Projection Session group
```
Existing:
  0x8001: ProjectionRequest
  0x8002: StopProjectionRequest
  0x8003: ProjectionPerformanceReport

Adding:
  0x8004: PauseProjectionRequest    <- newly added
  0x8005: ResumeProjectionRequest   <- newly added
```

### 4.2 When a New Group Is Needed

1. Check the next available group range
2. Start allocation from the group start address (`0x8001`, `0x8021`, `0x8041`, ...)

**Example**: Adding a "Recording" group to the Projection channel
```
// Group 8 (0x80E1~0x80FF): Recording (new)
0x80E1: StartRecordingRequest
0x80E2: StartRecordingResponse
0x80E3: StopRecordingRequest
0x80E4: StopRecordingResponse
```

---

## 5. Checklist

Items to verify when adding new messages or channels:

- [ ] Is the opcode unique within the channel?
- [ ] Does the opcode comply with group boundaries (0x20 increments)?
- [ ] Do Request/Response pairs use consecutive numbers?
- [ ] Has the allocation table in this document been updated?
- [ ] Are the opcode annotations in the proto files correct?
