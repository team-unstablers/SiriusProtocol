---
syntax: "proto3"
package: "sirius.extras.mdproto_example.sysmon"
import:
  - "msgdef/general.proto"
options:
  java_package: "com.sirius.extras.mdproto.sysmon"
  csharp_namespace: "Sirius.Extras.Mdproto.SysMon"
---

# System Monitoring Protocol Example

이 문서는 **MDProto** 형식을 사용하여 시스템 모니터링 서비스의 프로토콜을 정의하는 실용적인 예제입니다.
MDProto의 주요 기능인 `protobuf`, `optionset`, `constset` 코드 블록과 어노테이션 문법을 포괄적으로 다룹니다.

## 1. Constants & Enums

시스템 아키텍처와 프로세스 상태를 정의합니다.

```constset
constset Architecture : string {
    const X86_64 = "x86_64";
    const ARM64 = "arm64";
    const RISCV64 = "riscv64";
    const WASM = "wasm";
}
```

```protobuf
enum ProcessState {
    UNKNOWN = 0;
    RUNNING = 1;
    SLEEPING = 2;
    STOPPED = 3;
    ZOMBIE = 4;
}
```

## 2. Flags (Optionset)

프로세스의 속성을 비트마스크로 표현하기 위해 `optionset`을 사용합니다.

```optionset
optionset ProcessAttributeFlags {
    /// 루트 권한으로 실행 중
    option IS_ROOT = 0x01;
    
    /// 백그라운드 데몬 프로세스
    option IS_DAEMON = 0x02;
    
    /// 디버거가 붙어있음
    option IS_DEBUGGED = 0x04;
    
    /// 샌드박스 환경 내에서 실행 중
    option IS_SANDBOXED = 0x08;
}
```

## 3. Data Structures

### 3.1 Kernel Information (Low-level)

커널 매직 넘버와 같이 특정 엔디안 처리가 필요한 필드를 정의합니다.
어노테이션을 활용하여 필드에 바이트 오더 등의 메타데이터를 첨부할 수 있습니다.

```protobuf
message KernelInfo {
    /// 커널 매직 넘버 (Big Endian 강제)
    // @ensure_byte_order: BIG_ENDIAN
    fixed32 magic_number = 1;
    
    /// 커널 버전 문자열
    string version_string = 2;
    
    /// 시스템 아키텍처
    // @constset: Architecture
    string arch = 3;
}
```

### 3.2 Process Information

개별 프로세스의 상태 정보를 담는 메시지입니다.

```protobuf
message ProcessInfo {
    /// 프로세스 ID
    uint32 pid = 1;
    
    /// 부모 프로세스 ID
    uint32 ppid = 2;
    
    /// 실행 파일 이름
    string name = 3;
    
    /// 현재 상태
    ProcessState state = 4;
    
    /// 프로세스 속성 플래그
    // @optionset: ProcessAttributeFlags
    uint32 attributes = 5;
    
    /// 메모리 사용량 (Bytes)
    uint64 memory_usage = 6;
    
    /// CPU 사용률 (0.0 ~ 100.0)
    float cpu_usage = 7;
}
```

## 4. Requests & Responses (Opcodes)

### 4.1 Start Monitoring

모니터링 세션을 시작하는 요청입니다.
Opcode `0x2001`을 사용합니다.

```protobuf
// @opcode: 0x2001
message StartMonitorRequest {
    /// 모니터링 대상 PID 목록 (비어있으면 전체 시스템)
    repeated uint32 target_pids = 1;
    
    /// 업데이트 주기 (ms)
    uint32 interval_ms = 2;
    
    /// 커널 정보 포함 여부
    bool include_kernel_info = 3;
}
```

### 4.2 Monitor Update

주기적으로 전송되는 시스템 상태 업데이트입니다.
Opcode `0x2002`를 사용합니다.

```protobuf
// @opcode: 0x2002
message MonitorUpdate {
    /// 타임스탬프 (Unix ms)
    uint64 timestamp = 1;
    
    /// 전체 시스템 CPU 로드 (1분 평균)
    float system_load_1min = 2;
    
    /// 활성 프로세스 목록
    repeated ProcessInfo processes = 3;
    
    /// 커널 정보 (요청 시에만 포함)
    optional KernelInfo kernel_info = 4;
}
```
