# NAME

Sirius - macOS용 원격 제어 소프트웨어 'Noctiluca'의 기반 프로토콜

# SYNOPSIS

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

## FRAME STRUCTURE
- 모든 Sirius 프로토콜 메시지는 다음과 같은 프레임 구조를 가집니다.

```
+----------------+----------------+----------------+----------------+
|    Opcode      |  Payload Len   |    Payload     |   Checksum     |
|   (2 bytes)    |   (2 bytes)    |  (variable)    |   (4 bytes)    |
+----------------+----------------+----------------+----------------+
```

- Opcode (2 bytes): 메시지의 종류를 나타내는 고유 식별자입니다. Big-Endian 형식으로 인코딩됩니다.
- Payload Len (2 bytes): 페이로드의 길이를 나타내는 필드입니다. Big-Endian 형식으로 인코딩됩니다.
- Payload (variable): 실제 Protobuf 메시지 데이터를 포함하는 가변 길이 필드입니다. 길이는 Payload Len 필드에 의해 결정됩니다.
- Checksum (4 bytes): 메시지의 무결성을 검증하기 위한 체크섬 필드입니다. CRC32 알고리즘을 사용하여 계산됩니다.

