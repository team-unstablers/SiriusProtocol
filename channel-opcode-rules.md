# Channel Opcode Convention

Sirius 프로토콜의 Feature Channel에서 사용하는 opcode 컨벤션을 정의합니다.

> **적용 범위**: 이 규칙은 libsirius 메인테이너가 strict하게 따르는 가이드라인입니다.
> 채널을 확장하는 서드파티 개발자는 참고용으로 사용할 수 있습니다.

---

## 1. 기본 원칙

### 1.1 Opcode 범위

| 범위 | 용도 |
|------|------|
| `0x0000 ~ 0x7FFF` | **Global Opcode** - 메인 채널 전용 (핸드셰이크, 인증, 채널 관리) |
| `0x8000 ~ 0xFFFF` | **Feature Channel Opcode** - 각 기능 채널에서 사용 |

### 1.2 채널 간 독립성

**각 채널은 별도의 트랜스포트 스트림을 사용하므로, 채널 간 opcode 충돌 위험이 없습니다.**

예를 들어:
- HIDIO 채널의 `0x8001`과 Projection Data 채널의 `0x8001`은 서로 다른 메시지를 의미할 수 있음
- 동일한 opcode 값이라도 채널이 다르면 완전히 독립적

---

## 2. 그룹 + 연속 방식

### 2.1 그룹 구조

각 채널 내에서 **기능별 그룹**으로 나누고, 그룹 내에서 **연속 번호**를 할당합니다.

```
그룹 1: 0x8001 ~ 0x801F (32개 슬롯)
그룹 2: 0x8021 ~ 0x803F (32개 슬롯)
그룹 3: 0x8041 ~ 0x805F (32개 슬롯)
그룹 4: 0x8061 ~ 0x807F (32개 슬롯)
그룹 5: 0x8081 ~ 0x809F (32개 슬롯)
그룹 6: 0x80A1 ~ 0x80BF (32개 슬롯)
그룹 7: 0x80C1 ~ 0x80DF (32개 슬롯)
...
```

- 그룹 크기: **0x20 (32개)**
- 그룹 시작 주소: `0x8001`, `0x8021`, `0x8041`, ... (`0x8000 + 0x20*n + 1`)

### 2.2 그룹 내 메시지 배치

**완전 연속 방식**으로 관련 메시지를 붙여서 배치합니다:

```
0x8001: FooRequest
0x8002: FooResponse
0x8003: BarRequest
0x8004: BarResponse
0x8005: BazEvent
...
```

**장점**:
- 관련 Request/Response가 연속되어 직관적
- opcode만 봐도 같은 기능 그룹인지 파악 가능

### 2.3 그룹 용도 가이드 (권장)

채널 특성에 따라 자유롭게 그룹 용도를 정할 수 있습니다. 아래는 참고용 예시:

```
그룹 1: 채널의 핵심/세션 관련 메시지
그룹 2: 이벤트/알림 관련 메시지
그룹 3: 쿼리/조회 관련 메시지
그룹 4: 제어/조작 관련 메시지
그룹 5+: 확장용
```

---

## 3. 채널별 Opcode 할당

### 3.1 HIDIO 채널

```
┌─────────────────────────────────────────────────────────────────┐
│ 그룹 1 (0x8001~0x801F): Core                                    │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ HIDIOPacket                                           │
└─────────┴───────────────────────────────────────────────────────┘
```

### 3.2 Projection 채널

Projection 채널은 Display, Window, Cursor, Audio 관련 기능을 모두 포함합니다.

```
┌─────────────────────────────────────────────────────────────────┐
│ 그룹 1 (0x8001~0x801F): Projection Session                      │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ ProjectionRequest                                     │
│ 0x8002  │ StopProjectionRequest                                 │
│ 0x8003  │ ProjectionPerformanceReport                           │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 그룹 2 (0x8021~0x803F): Projection Session Events               │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8021  │ ProjectionSessionCreatedEvent                         │
│ 0x8022  │ ProjectionSessionCreationFailedEvent                  │
│ 0x8023  │ ProjectionSessionChangedEvent                         │
│ 0x8024  │ ProjectionSessionEndedEvent                           │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 그룹 3 (0x8041~0x805F): Display Management                      │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8041  │ DisplayListRequest                                    │
│ 0x8042  │ DisplayListResponse                                   │
│ 0x8043  │ SubscribeDisplayChangesRequest                        │
│ 0x8044  │ SubscribeDisplayChangesResponse                       │
│ 0x8045  │ UnsubscribeDisplayChangesRequest                      │
│ 0x8046  │ UnsubscribeDisplayChangesResponse                     │
│ 0x8047  │ DisplayChangedEvent                                   │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 그룹 4 (0x8061~0x807F): Window Query                            │
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
│ 그룹 5 (0x8081~0x809F): Window Manipulation                     │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8081  │ WindowManipulationRequest                             │
│ 0x8082  │ WindowManipulationResponse                            │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 그룹 6 (0x80A1~0x80BF): Cursor                                  │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x80A1  │ SubscribeCursorEventsRequest                          │
│ 0x80A2  │ SubscribeCursorEventsResponse                         │
│ 0x80A3  │ UnsubscribeCursorEventsRequest                        │
│ 0x80A4  │ UnsubscribeCursorEventsResponse                       │
│ 0x80A5  │ CursorEvent                                           │
└─────────┴───────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│ 그룹 7 (0x80C1~0x80DF): Audio Projection                        │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x80C1  │ AudioProjectionRequest                                │
│ 0x80C2  │ StopAudioProjectionRequest                            │
│ 0x80C3  │ AudioSessionCreatedEvent                              │
│ 0x80C4  │ AudioSessionCreationFailedEvent                       │
│ 0x80C5  │ AudioSessionChangedEvent                              │
│ 0x80C6  │ AudioSessionEndedEvent                                │
└─────────┴───────────────────────────────────────────────────────┘
```

### 3.3 Projection Data 채널

```
┌─────────────────────────────────────────────────────────────────┐
│ 그룹 1 (0x8001~0x801F): Frame Data                              │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ FrameDataHeader                                       │
│ 0x8002  │ CodecParameterSetMessage                              │
└─────────┴───────────────────────────────────────────────────────┘
```

### 3.4 Clipboard 채널

```
┌─────────────────────────────────────────────────────────────────┐
│ 그룹 1 (0x8001~0x801F): Core                                    │
├─────────┬───────────────────────────────────────────────────────┤
│ 0x8001  │ ClipboardEvent                                        │
└─────────┴───────────────────────────────────────────────────────┘
```

---

## 4. 새 메시지 추가 가이드

### 4.1 기존 그룹에 추가할 때

1. 해당 그룹에서 마지막으로 사용된 opcode를 확인
2. 그 다음 연속된 번호를 할당
3. 그룹 경계(32개)를 넘지 않도록 주의

**예시**: Projection Session 그룹에 새 메시지 추가
```
기존:
  0x8001: ProjectionRequest
  0x8002: StopProjectionRequest
  0x8003: ProjectionPerformanceReport

추가:
  0x8004: PauseProjectionRequest    <- 새로 추가
  0x8005: ResumeProjectionRequest   <- 새로 추가
```

### 4.2 새 그룹이 필요할 때

1. 사용 가능한 다음 그룹 범위를 확인
2. 그룹 시작 주소부터 할당 시작 (`0x8001`, `0x8021`, `0x8041`, ...)

**예시**: Projection 채널에 "Recording" 그룹 추가
```
// 그룹 8 (0x80E1~0x80FF): Recording (신규)
0x80E1: StartRecordingRequest
0x80E2: StartRecordingResponse
0x80E3: StopRecordingRequest
0x80E4: StopRecordingResponse
```

---

## 5. 체크리스트

새 메시지나 채널을 추가할 때 확인할 사항:

- [ ] opcode가 해당 채널 내에서 중복되지 않는가?
- [ ] opcode가 그룹 경계(0x20 단위)를 준수하는가?
- [ ] Request/Response 쌍이 연속된 번호를 사용하는가?
- [ ] 이 문서의 할당표가 업데이트되었는가?
- [ ] proto 파일의 opcode 주석이 정확한가?
