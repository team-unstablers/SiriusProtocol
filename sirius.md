# NAME

Sirius - macOS용 원격 제어 소프트웨어 'Noctiluca'의 기반 프로토콜

# SYNOPSIS (DRAFT)

```swift
class SampleServerApplication {
    var client: ClientSession!
    
    func onClientConnect() async throws {
        // HIDIO를 설정한다 -> 키보드, 마우스 입력을 받을 수 있다
        let hidioChannel = try await self.client.openChannel(for: .hidio, identifier: UUID())
        hidioChannel.delegate = self
        
        // 프로젝션 채널을 연다. -> 화면 전송 제어용
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
        let projectionDataChannel = try await channel.openChannel(for: .projectionData, identifier: channelIdentifier)
        projector.startProjection(to: projectionDataChannel, windowId: request.windowId)
        
        let event = ProjectionStartedEvent(
            channelIdentifier: channelIdentifier,
            codec: projector.codec,
            ...
        )
        
        channel.sendProjectionStartedEvent(event)
    }
}
```

# STRUCTURE OVERVIEW

Sirius 프로토콜은 서버-클라이언트 모델을 기반으로 합니다.

```
+--------------------+
|    Projects GUI    |
|       Session      |                     +-----------------+
+--------------------+---------------------+    Redirects    |
|  ScreenCaptureKit  | User Authentication |    HID Event    |
+--------------------+---------------------+-----------------+-----------+
|    VideoToolbox    |       PAM API       | Cocoa Event Tap |    ...    |
+------------------------------------------------------------------------+
|                           Server Application                           | ------- 여기서부턴 Sirius 프로토콜의 영역 바깥 (각 서버가 직접 구현) -------
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

<!-- TODO: 채널 / 스트림의 용어 분리. 스트림은 트랜스포트 레이어의 단위고 채널은 Sirius 프로토콜의 단위임을 명확히 할 것. -->
<!-- TODO: 하나의 채널은 하나의 스트림만 가진다고 명시 -->
<!-- TODO: 채널 식별자에 대한 설명 추가 -->
<!-- TODO: 채널은 서버/클라이언트가 필요할 때마다 서로 생성 / 종료할 수 있다고 명시 -->

- 서버-클라이언트 간의 커넥션은 여러 개의 멀티플랙스된 스트림 + 채널로 구성됩니다.
- 각 채널은 특정 기능(예: 화면 전송, HID 이벤트 전송 등)을 담당하며, 서로 독립적으로 동작합니다.
  - 각 기능(feature)은 한개 이상의 채널을 통해 데이터를 주고 받을 수 있습니다.
  - 예: 화면 전송 기능은 제어 채널과 데이터 채널로 구성될 수 있습니다.

- Sirius 프로토콜에서 사용할 트랜스포트 레이어(기반 전송 프로토콜)는 멀티플랙싱을 지원하는 프로토콜 사용을 권장합니다. (예: QUIC)
  - 멀티플랙싱을 지원하지 않는 프로토콜을 사용하는 경우, 수동으로 멀티플랙싱을 구현하면 사용할 수는 있으나 권장하지 않습니다.

### MAIN CHANNEL

- 모든 Sirius 커넥션에는 반드시 하나의 메인 채널이 존재해야 합니다.
- Sirius 프로토콜에서는 커넥션 직후 수립된 가장 먼저 생성된 채널을 메인 채널로 간주합니다.

- 메인 채널은 다음과 같은 역할을 담당합니다.
    - 프로토콜 핸드셰이크 수행
    - 사용자 인증(Authentication) 처리
    - `ServerNotice` 등의 전역 알림 이벤트 전송


# CHANNEL MESSAGES

- 모든 Sirius 프로토콜 메시지는 Protobuf 3로 정의되어 있습니다.
- 메시지 정의 파일은 `msgdef/` 디렉토리에 위치해 있습니다.

## OPCODES

- 모든 Sirius 프로토콜 메시지에는 고유한 opcode가 할당되어 있습니다.
- opcode는 2바이트 고정 길이 필드로, 각 메시지의 시작 부분에 위치합니다.
- opcode는 Big-Endian 형식으로 인코딩됩니다.

### 'GLOBAL' OPCODES VS 'FEATURE CHANNEL' OPCODES

- `0x0000` ~ `0x7FFF` 범위의 opcode는 'GLOBAL' opcode로 예약되어 있습니다.
  - 이러한 opcode는 메인 채널 및 모든 채널에서 공통적으로 사용됩니다.
  - 예: 핸드셰이크 메시지, 인증 메시지, 채널 시작 메시지 등
- `0x8000` ~ `0xFFFF` 범위의 opcode는 채널별로 정의된 opcode로 사용됩니다.
  - 예를 들어, A 채널의 opcode `0x8001`은 B 채널에서 다른 메시지를 나타낼 수 있습니다.
  - 각 기능(feature)은 자체적으로 opcode 범위를 관리해야 합니다.
    - 예: HIDIO 채널, Projection 채널 등

## FRAME STRUCTURE
- 모든 Sirius 프로토콜 메시지는 다음과 같은 프레임 구조를 가집니다.

```
+----------------+----------------+----------------+
|    Opcode      |  Payload Len   |    Payload     |
|   (2 bytes)    |   (4 bytes)    |  (variable)    |
+----------------+----------------+----------------+
```

- Opcode (2 bytes): 메시지의 종류를 나타내는 고유 식별자입니다. Big-Endian 형식으로 인코딩됩니다.
- Payload Len (4 bytes): 페이로드의 길이를 나타내는 필드입니다. Big-Endian 형식으로 인코딩됩니다.
- Payload (variable): 실제 Protobuf 메시지 데이터를 포함하는 가변 길이 필드입니다. 길이는 Payload Len 필드에 의해 결정됩니다.

# PROTOCOL FLOW

Sirius 프로토콜의 기본 플로우는 다음과 같습니다.

## 1. 커넥션 수립

- 서버와 클라이언트 간에 트랜스포트 레이어(예: QUIC)를 통해 커넥션이 수립됩니다.
- 서버의 인증서 지문이 신뢰할 수 있는지 검증합니다.
  - 만약 트러스트 리스트에 존재하지 경우, 사용자에게 경고 메시지를 표시하고 계속 진행할지 여부를 묻습니다.
    - 시스템의 트러스트 스토어를 사용하는 것 보다는, 내부적으로 직접 트러스트 리스트를 관리하는 방식을 권장합니다.
    - 왜냐하면, RDP 역시 그렇듯이 대부분의 Sirius 호스트는 자가 서명된 인증서를 사용할 것이기 때문입니다.
  - 만약 트러스트 리스트에 호스트가 있지만 지문이 다른 경우, '서버가 위장된 것일 수 있습니다' 같은 내용의 경고 메시지를 표시하고 계속 진행할지 여부를 묻습니다.
  - 만약 사용자가 계속 진행하지 않기로 한 경우, 커넥션을 종료합니다.

## 2. 메인 채널 생성 

- 커넥션이 수립된 직후, 클라이언트는 메인 채널을 생성해야 합니다.
- 만약, 지정된 시간 이내에 메인 채널이 생성되지 않으면, 서버는 커넥션을 종료할 수 있습니다.

## 3. 핸드셰이크

### 3-1. 클라이언트 핸드셰이크 요청

- 메인 채널이 생성된 후, **클라이언트**는 서버에 핸드셰이크 요청 메시지(`ClientHello`)를 전송해야 합니다.

```swift
mainChannel.send(ClientHello(
    // Sirius V1
    protocolVersion: 0x0100,
    
    agentName: "Noctiluca Client/1.0",
))
```

- `agentName` 필드는 아래 규칙을 따르는 것을 권장합니다.

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

### 3-2. 서버 핸드셰이크 응답

- 서버는 클라이언트의 핸드셰이크 요청을 수신한 후, 아래 사항을 검증합니다.
  - 지원하지 않거나 너무 오래된 프로토콜 버전인지 여부
  - 기타 핸드셰이크 조건 충족 여부 (예: 서버 용량 초과 등)

- 검증이 완료되면, **서버**는 클라이언트에게 핸드셰이크 응답 메시지(`ServerHello`)를 전송합니다.
- Acceptable하지 않은 요청인 경우, 서버는 `ServerNotice(severity=FATAL)` 메시지를 전송한 후 커넥션을 종료할 수 있습니다.

```swift
// acceptable한 요청인 경우
mainChannel.send(ServerHello(
    // Sirius V1
    protocolVersion: 0x0100,
    
    serverName: "Noctiluca Server/1.0",
    
    supportedFeatures: [ .hidio, .projection, ... ],
))

// acceptable하지 않은 요청인 경우
mainChannel.send(ServerNotice(
    severity: .fatal,
    code: .unsupportedProtocolVersion,
    message: "Unsupported protocol version",
    timestamp: UInt64(Date().timeIntervalSince1970 * 1000)
))

mainChannel.close()
transport.close()
```

## 4. 사용자 인증 (Authentication)

- 핸드셰이크가 성공적으로 완료된 후, **서버**는 클라이언트에게 인증 필요 메시지 (`AuthChallenge`)를 전송합니다.

```swift
mainChannel.send(AuthChallenge(
    acceptedMethods: [.password, .sshKey],
    message: '이 컴퓨터는 Acme Corp. 소유입니다. 허가 받지 않은 접근을 금지합니다.'
))
```

- **클라이언트**는 서버의 인증 요청을 수신한 후, 아래 시퀀스의 동작을 취합니다.
  - acceptedMethods 중에서 사용할 인증 방식을 선택합니다.
    - 만약 사용할 수 있는 인증 방식이 없는 경우, 커넥션을 종료합니다.

```swift
mainChannel.send(AuthRequest(
    method: .password,
    payload: encryptedPayloadData
)
```

- **서버**는 클라이언트의 인증 요청을 수신한 후, 인증 정보를 검증합니다.
  - 인증 정보가 유효하지 않은 경우, 서버는 AuthChallenge 메시지를 다시 전송하여 재시도를 요청할 수 있습니다.
  - 인증 정보가 유효한 경우, 인증이 성공적으로 완료되었음을 나타내는 메시지를 전송합니다.

```swift
let authRequest: AuthRequest
assert(authRequest.method == .password)
let payload = JSON.parse(authRequest.payload) as AuthPasswordPayload

guard pam.authenticate(payload.username, payload.password) else {
    mainChannel.send(AuthChallenge(
        acceptedMethods: [.password, .sshKey],
        message: '인증에 실패했습니다. 다시 시도해 주세요.'
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

// 인증 성공
mainChannel.send(AuthResponse(
    sessionId: UUID()
))
```

## 5. 채널 생성 및 기능 사용

- 인증이 성공적으로 완료된 후, 클라이언트와 서버는 각각 필요한 기능을 사용하기 위해 새로운 채널을 생성할 수 있습니다.

- 원격 제어에 필수적인 기능(feature)에 대한 채널은 클라이언트가 생성합니다.
  - HIDIO
  - Projection

### 채널 생성 예시

```swift
class ClientSideSession {
    let transport: Transport
    
    func openChannel<Ch>(for feature: SiriusFeature, identifier: UUID, args: string[] = []) async throws -> Ch where Ch: SiriusChannel {
        // 새 스트림을 연다
        let stream = try await transport.openStream()
        
        // '이제부터 이 스트림은 어떠어떠한 feature를 위한 채널이다' 라고 상대편에게 알린다
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
            identifier: identifier,
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
                code: 0,
            ))
        }
    }
}
```

### high-level example

```swift
/// 키보드와 마우스 입력을 전송하기 위한 HIDIO 채널 생성
let hidioChannel = try await session.openChannel(
    for: .hidio,
    identifier: UUID()
)

/// 화면 전송 제어를 위한 Projection 채널 생성
let projectionChannel = try await session.openChannel(
    for: .projection,
    identifier: UUID()
)

/// 화면 전송 데이터를 수신하기 위한 ProjectionData 채널 생성
let projectionDataChannel = try await session.openChannel(
    for: .projectionData,
    identifier: UUID()
)

hidioChannel.moveMouse(x: 100, y: 200)
hidioChannel.keyDown(keyCode: 0x04) // 'A' key down
hidioChannel.keyUp(keyCode: 0x04)   // 'A' key

let windowList = projectionChannel.requestWindowList(
    filter: [.titleContains("Xcode")]
)

// ...

projectionChannel.requestProjection(
    viewport: .singleWindow(someWindowId),
    preferredCodecs: [
      .h265(quality: .auto, frameRate: 60.000, width: -1, height: -1, options: "color-format: 'YUV444'; hardware-acceleration: 'auto'; profile: 'high'; level: '4.2';"),
      .h264(quality: .high, frameRate: 30.000, width: 1280, height: 720, options: "color-format: 'YUV420'; hardware-acceleration: 'auto'; profile: 'main'; level: '4.0';")
    ]
)

// 서버가 ProjectionStarted 이벤트를 보냄과 동시에 'ProjectionData' 채널이 열린다
// ProjectionData 채널을 통해 H.264 / H.265 인코딩된 화면 데이터가 전송된다
// ...

projectionDataChannel.onVideoFrameReceived { frame in
    renderVideoFrame(frame)
}
```