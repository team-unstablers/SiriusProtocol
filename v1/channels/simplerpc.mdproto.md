---
syntax: "proto3"
package: "sirius.msgdef.v1.channels.simplerpc"
import:
  - "common-types.proto"
---

# SIMPLE RPC CHANNEL

A generic request/response RPC bus for operations that do not warrant a dedicated channel of their own. Each operation is identified by a reverse-DNS string (e.g., `app.noctiluca.appstream.set-input-method`) and exchanges UTF-8 text payloads modeled after a POSIX program invocation: the operation name corresponds to `argv[0]`, the arguments to `argv[1..]`, the response body to `stdout`, and the response code to the program's exit code.

The channel is **bidirectional**: either peer MAY initiate a `SimpleRPCRequest`, and the other peer answers with a `SimpleRPCResponse` correlated by `requestId`. Each peer maintains its own independent in-flight request table; `requestId` values issued by each peer are local to that peer and need not be globally unique across peers.

simplerpc is intended for:

- Simple, one-shot operations that fit naturally into a request/response pattern.
- Lightweight operations that do not justify the overhead of defining a new dedicated channel with its own msgdef and per-language bindings.
- Cases where the requester wants to invoke an operation that the remote peer may or may not implement; unsupported operations are handled inline via a `notSupported` response without requiring a prior capability-negotiation round trip.

simplerpc is NOT suited for:

- High-throughput or streaming workloads — use a dedicated channel.
- Operations that need negotiated session state across multiple correlated messages — model that with a dedicated channel.
- Bulk binary payload transport — use a dedicated Transfer channel.

---

# CONSTANTS

## SimpleRPCErrorCode

```constset
constset SimpleRPCErrorCode: uint32 {
  /// The request completed successfully.
  const success = 0;

  /// The responder encountered an internal error (uncaught exception, etc.) while executing the operation.
  const internal = 1;

  /// One or more `args` elements could not be parsed or validated by the operation.
  const invalidArgs = 2;

  /// The responder does not recognize `operation`, or the operation is recognized but disabled on this peer.
  const notSupported = 3;

  /// The caller lacks the privilege required to invoke this operation.
  const permissionDenied = 4;

  /// The responder timed out while executing the operation. Senders MAY retry, but SHOULD treat repeated timeouts as a transient failure of the operation rather than the channel.
  const timeout = 5;

  /// Operation-specific error codes occupy the range >= 1000.
  /// Each operation's specification defines its own non-zero codes within this range.
}
```

### DESCRIPTION

The shared error vocabulary used in `SimpleRPCResponse.code`. This is an open enum: receivers MUST treat unknown values as `internal` so that new error categories — both protocol-level (this constset) and operation-specific (>= 1000) — can be introduced in future revisions without breaking wire compatibility.

The error model is HTTP-style: `code` carries the categorical outcome of the request, and any human-readable diagnostic message is carried inline in `SimpleRPCResponse.retval`. There is no separate `errorMessage` field; clients SHOULD render `retval` as the diagnostic when `code != success`.

---

# MESSAGES

## SimpleRPCRequest

```protobuf
/// Requests the receiving peer to execute an operation.
// @opcode: 0x8001
message SimpleRPCRequest {
  /// Identifier issued by the requesting peer. The receiving peer echoes this back
  /// in `SimpleRPCResponse.requestId` so the requester can correlate the response.
  SRUUID requestId = 1;

  /// Operation identifier in reverse-DNS form (e.g., `app.noctiluca.appstream.set-input-method`).
  /// See `OPERATION NAMING` for namespace conventions.
  string operation = 2;

  /// Positional arguments for the operation, modeled after `argv[1..]` in a POSIX program invocation.
  /// The element format (plain string, key=value, JSON, base64-encoded binary, ...) is
  /// defined by each operation's specification; the channel itself is format-agnostic.
  repeated string args = 3;
}
```

### DESCRIPTION

Either peer MAY send a `SimpleRPCRequest`. The receiving peer SHOULD respond with a corresponding `SimpleRPCResponse` matching on `requestId`.

Responses MAY arrive in any order with respect to the order in which requests were sent. The channel makes no ordering guarantees beyond per-message correlation via `requestId`.

### IMPLEMENTATION NOTES

- `requestId` MUST be unique within the set of currently in-flight requests issued by the same peer on the same channel. Senders SHOULD use random UUIDs to avoid collisions and to permit safe reuse after a response (or timeout) clears the slot.
- A peer MAY drop requests it cannot service for any reason. Silent drops leave the requester to time out at its own discretion; best practice is to respond with `notSupported` (or another appropriate code) rather than dropping the request.
- `operation` SHOULD NOT exceed 256 bytes. Senders that exceed this limit SHOULD expect a `notSupported` response.
- Individual `args[i]` elements SHOULD remain below 1 MiB. See `PAYLOAD ENCODING` for the full size policy.

## SimpleRPCResponse

```protobuf
/// Response to a SimpleRPCRequest.
// @opcode: 0x8002
message SimpleRPCResponse {
  /// Echoes back `SimpleRPCRequest.requestId` so the requester can correlate
  /// this response with the originating request.
  SRUUID requestId = 1;

  /// Categorical outcome of the operation. `0` indicates success;
  /// non-zero values follow `SimpleRPCErrorCode`.
  // @constset: SimpleRPCErrorCode
  uint32 code = 2;

  /// Operation result body, modeled after stdout in a POSIX program invocation.
  /// On success, the format is operation-defined. On error, the body SHOULD carry
  /// a human-readable diagnostic message.
  optional string retval = 3;
}
```

### DESCRIPTION

Sent by the peer that received a `SimpleRPCRequest` after the operation has been executed, rejected, or otherwise terminated.

The error model follows HTTP-style conventions: `code` carries the categorical outcome of the request, while any human-readable diagnostic message is carried inline in `retval`. There is no separate diagnostic field. When `code != success`, implementations SHOULD render `retval` to the user as the failure reason.

### IMPLEMENTATION NOTES

- `retval` MAY be omitted entirely or carried as an empty string. The two are equivalent at the channel level. Operation specifications MAY assign semantic distinction between "absent" and "empty" at the operation level, but the channel itself does not.
- Implementations MUST NOT emit responses whose `requestId` does not match an in-flight request received from the remote peer. Unsolicited responses MUST be ignored by the receiver.
- A given `SimpleRPCRequest` MUST receive at most one `SimpleRPCResponse`. Subsequent responses bearing the same `requestId` MUST be ignored by the receiver.

---

# PAYLOAD ENCODING

`args` and `retval` carry UTF-8 text. The intended payload format within each field is operation-defined; the channel itself is format-agnostic.

When an operation needs to carry binary data, the caller MUST encode it as base64 (or another text-safe encoding agreed upon by the operation specification). Senders that need to carry large binary payloads SHOULD use a dedicated channel (e.g., Transfer) instead; simplerpc is not designed for bulk data transport.

As a soft guideline, individual `args[i]` elements and `retval` SHOULD remain below 1 MiB. Implementations MAY enforce stricter per-operation limits as appropriate.

Each Sirius frame remains subject to the protocol-wide 16 MiB frame size limit defined in the protocol introduction.

---

# OPERATION NAMING

Operation identifiers use reverse-DNS namespacing to avoid collisions between independent operation owners.

- `sirius.*` is reserved for protocol-level operations introduced by future revisions of SiriusProtocol itself.
- `app.noctiluca.*` is used by Noctiluca operations (e.g., `app.noctiluca.appstream.set-input-method`).
- Third-party extensions SHOULD use their own reverse-DNS namespace under a domain they control.

Operation names are case-sensitive. They SHOULD be composed of lowercase ASCII letters, digits, dots, and hyphens; underscores SHOULD be avoided to keep identifiers consistent with the existing convention used for quirk keys and other Noctiluca-defined namespaces.

---

# CHANNEL ARGUMENTS

The simplerpc channel does not currently take any channel arguments via `ChannelStartRequest.args`. Requesters MUST send an empty argument list, and peers MUST ignore any unknown arguments to remain forward-compatible. Future revisions MAY introduce optional arguments (for example, to negotiate channel-level feature flags) without breaking existing peers.

---

# CHANNEL LIFECYCLE

The simplerpc channel layers on top of the standard Sirius channel lifecycle with the following channel-specific notes.

## Multi-Instance

Multiple simplerpc channels MAY coexist on a single Sirius connection. Each channel maintains its own independent in-flight request table; `requestId` values are not shared across channels.

However, multi-instance simplerpc is **not recommended** as a general design pattern. simplerpc is itself a multiplexing bus over operations, and a second layer of channel-level multiplexing is rarely justified. Implementations are not required to support multi-instance simplerpc; a peer MAY reject additional `ChannelStartRequest`s for simplerpc at the channel-management level after a first instance is open.

The principal scenario in which multi-instance simplerpc could provide a measurable benefit is QUIC stream HOL blocking — separating high-priority operations onto their own channel so that a slow operation on another channel does not stall them. However, simplerpc is not designed for frequencies at which HOL blocking would be a practical concern. Workloads that approach that frequency profile SHOULD migrate to a dedicated channel rather than spawning multiple simplerpc channels.

## In-Flight Cleanup on Close

When a simplerpc channel closes — gracefully, by transport failure, or by remote channel close — both peers MUST treat every in-flight request issued on that channel as terminated. Pending requester-side futures SHOULD be completed with a transport-level cancellation error; responders SHOULD attempt best-effort cancellation of any operation execution still in progress on their side.

No further `SimpleRPCResponse` MAY be sent on a channel after it has closed. Responses arriving from the transport after the channel close MUST be dropped silently.

---

# SAMPLE EXCHANGE

```
--> { requestId: <uuid>, operation: "app.noctiluca.appstream.set-input-method",
      args: [ "{\"lang\":\"ko_KR\",\"prefer_third_party\":true}" ] }

<-- { requestId: <same uuid>, code: 0, retval: "{}" }
```

The example above uses JSON as the per-operation payload format, but the channel itself is agnostic to the choice: an operation whose contract is `args = ["ko_KR", "true"]` with `retval = "ok"` is equally valid simplerpc usage. The format is part of the operation's specification, not the channel's.
