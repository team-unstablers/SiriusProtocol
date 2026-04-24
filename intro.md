# NAME

Sirius - The underlying protocol for 'Noctiluca', a remote desktop software for macOS

Sirius is an application-level protocol for establishing and managing remote desktop sessions
between a server and a client.
It handles common procedures such as handshake, authentication, and channel creation,
while individual features like screen streaming and input forwarding are separated into dedicated channels.

# SYNOPSIS (DRAFT)

```swift
class SampleServerApplication {
    var client: ClientSession!

    func onClientConnect() async throws {
        // Open an HIDIO channel — enables keyboard and mouse input exchange.
        let hidioChannel = try await self.client.openChannel(for: .hidio, identifier: UUID())
        hidioChannel.delegate = self

        // Open a Projection channel — used for screen streaming control.
        let projectionChannel = try await self.client.openChannel(for: .projection, identifier: UUID())
        projectionChannel.delegate = self
    }
}

extension SampleServerApplication: HIDIOChannelDelegate {
    func hidioChannelDidReceiveKeyEvent(_ channel: HIDIOChannel, event: KeyEvent) {
        self.handleKeyEvent(event)
    }

    func hidioChannelDidReceiveMouseMoveEvent(_ channel: HIDIOChannel, event: MouseMoveEvent) {
        self.handleMouseMoveEvent(event)
    }
}

extension SampleServerApplication: ProjectionChannelDelegate {
    func projectionChannelDidReceiveProjectionRequest(_ channel: ProjectionChannel, request: ProjectionRequest) {
        guard windowManager.windowExists(request.windowId), ... else {
            return
        }

        let projector = SessionProjector()
        projector.configure(codec: request.codec, ...)

        let channelIdentifier = UUID()

        // Open a data channel for transmitting actual video frames.
        let projectionDataChannel = try await channel.openChannel(
            for: .projectionData,
            identifier: channelIdentifier
        )

        projector.startProjection(to: projectionDataChannel, windowId: request.windowId)

        let event = ProjectionStartedEvent(
            channelIdentifier: channelIdentifier,
            codec: projector.codec,
            ...
        )

        // Send a projection-started event through the control channel.
        channel.sendProjectionStartedEvent(event)
    }
}
```

# STRUCTURE OVERVIEW

The Sirius protocol is based on a server-client model.
The diagram below illustrates the layered structure between a macOS server application,
the transport layer, and the client application, along with the relationship between
channels and streams that operate on top of them.

```
+--------------------+
|    Projects GUI    |
|       Session      |                     +-----------------+
+--------------------+---------------------+    Redirects    |
|  ScreenCaptureKit  | User Authentication |    HID Event    |
+--------------------+---------------------+-----------------+-----------+
|    VideoToolbox    |       PAM API       | Cocoa Event Tap |    ...    |
+------------------------------------------------------------------------+
|                           Server Application                           | ------- Outside the scope of the Sirius protocol (implemented by each server) -------
+--------------------+------------------+--------------------+-----------+
|    Main Channel    |    Channel #N    |    Channel #N+1    |    ...    |
+--------------------+------------------+--------------------+-----------+
|     Stream  #1     |    Stream  #N    |    Stream  #N+1    |    ...    |
+--------------------+------------------+--------------------+-----------+
|                      Transport Layer (e.g., QUIC)                      |
+------------------------------------------------------------------------+
                                   ||
                                   ||
                                   ||
+------------------------------------------------------------------------+
|                      Transport Layer (e.g., QUIC)                      |
+--------------------+------------------+--------------------+-----------+
|     Stream  #1     |    Stream  #N    |    Stream  #N+1    |    ...    |
+--------------------+------------------+--------------------+-----------+
|    Main Channel    |    Channel #N    |    Channel #N+1    |    ...    |
+--------------------+------------------+--------------------+-----------+
|                           Client Application                           |
+------------------------------------------------------------------------+
```

## MULTI-CHANNEL ARCHITECTURE

Sirius uses the concept of channels to handle multiple features simultaneously
over a single transport connection.

- A channel is a **logical communication unit** defined at the Sirius protocol level.
- Each channel maps **1:1** to a stream in the transport layer (e.g., QUIC).
- Multiple streams and channels can coexist within a single server-client connection.

Each channel is responsible for a specific feature and operates independently.

- For example, the following features may be implemented as separate channels:
  - HIDIO: Keyboard, mouse, and other input event transmission
  - Projection: Screen streaming control
  - ProjectionData: Encoded video frame transmission
- A single feature may also be split across multiple channels.
  - Example: Separating screen streaming into a control channel and a data channel

The transport layer used by Sirius SHOULD support multiplexing.

- Example: QUIC
- While it is possible to emulate streams at the application level over a transport
  that does not support multiplexing, this approach is NOT recommended.

### Channel Lifecycle Overview

- The server and client can **open and close channels as needed**.
- Channels may be initiated from either side (server or client).
  When starting a channel, the initiator MUST inform the remote peer **which feature the channel is for**.
- When a channel is closed, the associated stream is also terminated.

Detailed message definitions and state transitions are defined in `v1/channels.proto`.

### MAIN CHANNEL

Every Sirius connection MUST have exactly one **main channel**.

- The first stream/channel created immediately after the transport connection is established
  is designated as the main channel.
- The main channel handles the common control flow across the entire session.

The main channel is primarily responsible for the following:

- Performing the protocol handshake
  - Exchanging `ClientHello` and `ServerHello` messages
- Handling user authentication
  - `AuthChallenge`, `AuthRequest`, `AuthResponse`
- Delivering session-wide global notifications such as `ServerNotice`

If the main channel is closed due to exceptional circumstances (e.g., protocol version mismatch,
fatal error), the transport connection is generally terminated as well.

# CHANNEL MESSAGES

All messages in the Sirius protocol are defined using **Protocol Buffers v3**.

- Common messages and main channel messages: `general.proto`
- Session management and authentication messages: `v1/session.proto`
- Channel start/close messages: `v1/channels.proto`
- Feature-specific channel messages: `v1/channels/**.proto`
  - Examples: HIDIO, Projection, Window Management, etc.

Each message is assigned an opcode. Framing rules are defined in the **OPCODES** and
**FRAME STRUCTURE** sections below.

## OPCODES

Every Sirius protocol message has a 2-byte opcode.

- An opcode is a unique identifier indicating the type of message.
- Opcodes are encoded in Big-Endian byte order.

### GLOBAL OPCODES vs FEATURE CHANNEL OPCODES

The opcode space is divided into three ranges:

- `0x0000` – `0x7FFF`: **Global opcodes**
  - Assigned to messages **shared across all channels**, including the main channel.
  - Examples:
    - Handshake messages (`ClientHello`, `ServerHello`)
    - Authentication messages (`AuthChallenge`, `AuthRequest`, `AuthResponse`)
    - Channel management messages (`ChannelStartRequest`, `ChannelStartResponse`, `ChannelCloseRequest`)
- `0x8000` – `0xFF00`: **Feature channel opcodes**
  - Used for **channel-specific messages** defined by each feature.
  - Examples:
    - Input event messages for the HIDIO channel
    - Projection request/event messages for the Projection channel
  - The same numeric opcode value MAY refer to different messages depending on the channel/feature.

- `0xFF01` – `0xFFFF`: **Special Use**
  - `0xFFFE`: encapsulated message (see 'PROTOCOL UPGRADE' section)

Each feature is responsible for managing its own range of feature channel opcodes.

## FRAME STRUCTURE

All Sirius messages use the following fixed framing structure:

```
+----------------+----------------+----------------+
|    Opcode      |  Payload Len   |    Payload     |
|   (2 bytes)    |   (4 bytes)    |  (variable)    |
+----------------+----------------+----------------+
```

- **Opcode (2 bytes)**
  A unique identifier indicating the message type. Encoded in Big-Endian byte order.

- **Payload Len (4 bytes)**
  The length of the payload. An unsigned integer in Big-Endian format, representing
  **the number of Protobuf-serialized bytes excluding the opcode and length fields**.

- **Payload (variable)**
  A variable-length field containing the actual message data serialized with Protobuf v3.

### FRAME SIZE LIMIT

The `Payload Len` field MUST NOT exceed **16 MiB** (16 × 1024 × 1024 = 16,777,216 bytes).

- A receiver that encounters a frame declaring a `Payload Len` greater than this limit
  MUST treat the stream as corrupted, SHOULD send a `Goodbye` message with
  `ClosureCode.protocolError`, and MUST close the transport connection.
- Applications that need to exchange larger payloads MUST split the data across multiple
  messages or use a dedicated transfer mechanism (e.g., the Transfer channel).
- This limit is intentionally conservative. Future protocol versions MAY raise it,
  but SHOULD NOT lower it, to preserve forward compatibility with existing clients.

## PROTOCOL UPGRADE

- Each feature/channel MAY upgrade the stream itself to a different protocol as needed.
  - Examples: WebSocket-over-Sirius, SSH-over-Sirius, TCP-over-Sirius
- Messages from an upgraded protocol MUST be encapsulated using the `0xFFFE` opcode.

# PROTOCOL FLOW

The basic session establishment flow of the Sirius protocol is as follows:

1. Transport connection establishment
2. Main channel creation
3. Protocol handshake
4. User authentication
5. Feature channel creation and usage

Each step is briefly described below.

## 1. Connection Establishment

- The server and client establish a connection using a transport protocol that supports
  multiplexing, such as QUIC.
- If the protocol operates over TLS, the server certificate MUST be verified to prevent
  MITM attacks and similar threats.

Recommendations for server certificate verification:

- Use an internally managed **trust list** based on server certificate fingerprints.
  - Most Sirius hosts are expected to use self-signed certificates.
  - The OS-level trust store alone is insufficient to adequately represent trust decisions.
- If the host is not in the trust list:
  - It is RECOMMENDED to display a warning to the user and provide a UI to confirm
    whether to proceed with the connection.
- If an entry exists in the trust list but the stored fingerprint does not match the actual fingerprint:
  - A warning such as "The server may have been impersonated" SHOULD be displayed, and the
    connection SHOULD be rejected unless the user explicitly permits it.
- If the user chooses not to allow the connection:
  - The transport connection MUST be terminated immediately.

## 2. Main Channel Creation

- Immediately after the connection is established, **the client MUST create the main channel**.
- Main channel creation is accomplished by opening the first stream at the transport layer.
- If the main channel is not created within a predefined timeout, the server MAY consider
  the connection timed out and close it.

## 3. Handshake

### 3-1. Client Handshake Request

Once the main channel is ready, the client sends a `ClientHello` message to the server.

```swift
mainChannel.send(ClientHello(
    // Sirius V1
    protocolVersion: 0x0100,
    agentName: "Noctiluca Client/1.0"
))
```

The `agentName` field SHOULD follow the format below:

```ebnf
AGENT              = APPLICATION_NAME , "/" , VERSION , [ " " , ADDITIONAL_INFO ] ;

APPLICATION_NAME   = 1*( ALPHA | DIGIT | "-" | "_" ) ;
VERSION            = 1*( ALPHA | DIGIT | "." | "-" ) ;

ADDITIONAL_INFO    = "(" , INFO_ITEM , *( ";" , INFO_ITEM ) , ")" ;
INFO_ITEM          = PLATFORM | LIB_INFO | FLAG ;

PLATFORM           = 1*( ALPHA | DIGIT | "-" | "_" ) ;
LIB_INFO           = LIB_NAME , "/" , VERSION ;
LIB_NAME           = 1*( ALPHA | DIGIT | "-" | "_" ) ;

FLAG               = "+" , 1*( ALPHA | DIGIT | "-" | "_" ) ;
```

For example, the following values may be used:

- `NoctilucaClient/1.0 (macOS 14.0; libsirius/1.2.3; +automated)`
  - Noctiluca Client version 1.0 running on macOS 14.0. The `+automated` flag indicates
    that the computer will be controlled by an automated script.

### 3-2. Server Handshake Response

Upon receiving the `ClientHello`, the server validates the following:

- Whether the protocol version is unsupported
- Whether the protocol version is too old (deprecated)
- Whether the server is in a state that can accept new sessions
  - Examples: concurrent session limits, resource constraints, etc.

After validation, the server sends either a `ServerHello` or a `ServerNotice`.

```swift
// If the request is acceptable
mainChannel.send(ServerHello(
    // Sirius V1
    protocolVersion: 0x0100,
    serverName: "Noctiluca Server/1.0",
    supportedFeatures: [ .hidio, .projection, ... ]
))

// If the request is not acceptable
mainChannel.send(ServerNotice(
    severity: .fatal,
    code: .unsupportedProtocolVersion,
    message: "Unsupported protocol version",
    timestamp: UInt64(Date().timeIntervalSince1970 * 1000)
))

mainChannel.close()
transport.close()
```

The `supportedFeatures` field contains identifiers for the features provided by the server.
In the actual message definition, feature identifiers are represented as UUIDs
(see `general.proto`). At the application level, these may be wrapped as enums
such as `.hidio`, `.projection`, etc.

## 4. User Authentication

After a successful handshake, the server MAY require the client to authenticate.

### 4-1. Presenting Authentication Methods (`AuthChallenge`)

The server presents the available authentication methods to the client via an `AuthChallenge` message.

```swift
mainChannel.send(AuthChallenge(
    acceptedMethods: [.password, .sshKey],
    message: "This computer is owned by Acme Corp. Unauthorized access is prohibited."
))
```

`acceptedMethods` may include the following values.
See `v1/session.proto` for the specific enum definition.

- `AUTH_METHOD_NONE`
- `AUTH_METHOD_PASSWORD`
- `AUTH_METHOD_SIMPLE_PASSWORD`
- `AUTH_METHOD_SSH_KEY`

### 4-2. Client Authentication Request (`AuthRequest`)

The client selects one of the supported methods from `acceptedMethods` and sends the required
authentication credentials via an `AuthRequest` message.

```swift
mainChannel.send(AuthRequest(
    method: .password,
    payload: encryptedPayloadData
))
```

- `method`: The selected authentication method
- `payload`: The authentication credentials required by the chosen method
  (e.g., username/password, signature, etc.)

If the client determines that none of the available authentication methods are supported,
it MUST close the connection.

### 4-3. Server-Side Authentication Processing

Upon receiving the `AuthRequest`, the server performs authentication through its internal
authentication system (e.g., PAM).

```swift
let authRequest: AuthRequest
assert(authRequest.method == .password)

let payload = JSON.parse(authRequest.payload) as AuthPasswordPayload

guard pam.authenticate(payload.username, payload.password) else {
    mainChannel.send(AuthChallenge(
        acceptedMethods: [.password, .sshKey],
        message: "Authentication failed. Please try again."
    ))

    maxRetryCount -= 1

    if maxRetryCount <= 0 {
        mainChannel.send(ServerNotice(
            severity: .fatal,
            code: .authenticationFailed,
            message: "Authentication failed",
            timestamp: UInt64(Date().timeIntervalSince1970 * 1000)
        ))

        mainChannel.close()
        transport.close()
    }
    return
}

// Authentication successful
mainChannel.send(AuthResponse(
    sessionId: UUID()
))
```

The server MAY freely define retry limits, lock-out policies, and other authentication
policies as needed.

## 5. Channel Creation and Feature Usage

Once authentication is complete, both the client and server can create new channels
to use the features they need.

The generally recommended pattern is as follows:

- The client creates channels for features that are **essential for remote desktop**.
  - Examples: HIDIO, Projection
- If the server needs to proactively push a feature, it may also initiate channels.

### Channel Creation Example (Conceptual Code)

```swift
class ClientSideSession {
    let transport: Transport

    func openChannel<Ch>(
        for feature: SiriusFeature,
        identifier: UUID,
        args: [String] = []
    ) async throws -> Ch where Ch: SiriusChannel {
        // Open a new stream.
        let stream = try await transport.openStream()

        // Inform the remote peer that this stream is a channel for a specific feature.
        stream.send(ChannelStartRequest(
            featureId: feature.rawValue,
            channelId: identifier,
            args: args
        ))

        let data = try await stream.blockUntilReceive()
        // ... parse ...
        let response = ChannelStartResponse(data: data)

        guard response.success else {
            throw SiriusError.channelStartFailed(code: response.code, message: response.message)
        }

        let channel = Ch(
            stream: stream,
            identifier: identifier
        )

        return channel
    }
}

extension ServerSideSession: TransportDelegate {
    func transportDidOpenStream(_ transport: Transport, stream: Stream) {
        Task {
            let data = try await stream.blockUntilReceive()
            // ... parse ...
            let request = ChannelStartRequest(data: data)

            guard let feature = SiriusFeature(rawValue: request.featureId) else {
                stream.send(ChannelStartResponse(
                    success: false,
                    code: .unknownFeature,
                    message: "Unknown feature ID"
                ))
                stream.close()
                return
            }

            switch feature {
            case .hidio:
                let channel = HIDIOChannel(
                    stream: stream,
                    identifier: request.channelId
                )
                self.registerChannel(channel)

            case .projection:
                let channel = ProjectionChannel(
                    stream: stream,
                    identifier: request.channelId
                )
                self.registerChannel(channel)

            // ...
            default:
                stream.send(ChannelStartResponse(
                    success: false,
                    code: .unsupportedFeature,
                    message: "Unsupported feature"
                ))
                stream.close()
                return
            }

            stream.send(ChannelStartResponse(
                success: true,
                code: 0
            ))
        }
    }
}
```

The above example is conceptual code intended to illustrate the **channel creation flow**,
rather than representing actual implementation code.

- Client:
  - Opens a new stream → sends `ChannelStartRequest` → validates the response → creates a channel object
- Server:
  - Parses the first message on a new stream as `ChannelStartRequest`
  - Interprets the feature ID and creates the appropriate channel type
  - Replies with `ChannelStartResponse` indicating success or failure

### High-Level Usage Example

```swift
/// Create an HIDIO channel for keyboard and mouse input transmission
let hidioChannel = try await session.openChannel(
    for: .hidio,
    identifier: UUID()
)

/// Create a Projection channel for screen streaming control
let projectionChannel = try await session.openChannel(
    for: .projection,
    identifier: UUID()
)

/// Create a ProjectionData channel for receiving screen streaming data
let projectionDataChannel = try await session.openChannel(
    for: .projectionData,
    identifier: UUID()
)

hidioChannel.moveMouse(x: 100, y: 200)
hidioChannel.keyDown(keyCode: 0x04) // 'A' key down
hidioChannel.keyUp(keyCode: 0x04)   // 'A' key up

let windowList = projectionChannel.requestWindowList(
    filter: [.titleContains("Xcode")]
)

// ...

projectionChannel.requestProjection(
    viewport: .singleWindow(someWindowId),
    preferredCodecs: [
        .h265(
            quality: .auto,
            frameRate: 60.000,
            width: -1,
            height: -1,
            options: "color-format: 'YUV444'; hardware-acceleration: 'auto'; profile: 'high'; level: '4.2';"
        ),
        .h264(
            quality: .high,
            frameRate: 30.000,
            width: 1280,
            height: 720,
            options: "color-format: 'YUV420'; hardware-acceleration: 'auto'; profile: 'main'; level: '4.0';"
        )
    ]
)

// The server sends a ProjectionStarted event, and a ProjectionData channel is opened simultaneously.
// H.264 / H.265 encoded screen data is transmitted through the ProjectionData channel.
// ...

projectionDataChannel.onVideoFrameReceived { frame in
    renderVideoFrame(frame)
}
```

The above example demonstrates a typical Sirius-based remote desktop workflow from the
library user's perspective: **session → channel → feature usage**.
