# sairedis.rec Recording Flow in SONiC

This document explains the end-to-end flow of how `sairedis.rec` gets written as part of the SONiC SAI SDK architecture.

## Overview

`sairedis.rec` is a recording file that captures **all SAI API calls** made by orchagent through the sairedis library. It serves as a comprehensive audit trail for debugging, troubleshooting hardware configuration issues, and replay testing.

## Architecture Diagram

```
┌────────────────────────────────────────────────────────────────────────────┐
│                              ORCHAGENT                                      │
│   (Network Orchestration Agent - translates APPL_DB to SAI calls)          │
└────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    │ SAI API calls (create, remove, set, get)
                                    ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                    LIBSAIREDIS (Client SAI Library)                         │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │                    RedisRemoteSaiInterface                           │   │
│  │  • Intercepts all SAI API calls                                      │   │
│  │  • Serializes object attributes                                      │   │
│  │  • Records to sairedis.rec via Recorder class    ──────────────────────────┐
│  │  • Sends commands to Redis ASIC_STATE table                          │   │ │
│  └─────────────────────────────────────────────────────────────────────┘   │ │
└────────────────────────────────────────────────────────────────────────────┘ │
                                    │                                          │
                                    │ Redis/ZMQ                                │
                                    ▼                                          │
┌────────────────────────────────────────────────────────────────────────────┐ │
│                         REDIS (ASIC_STATE_DB)                               │ │
│   Stores SAI object state and serves as IPC between orchagent & syncd      │ │
└────────────────────────────────────────────────────────────────────────────┘ │
                                    │                                          │
                                    │ SelectableChannel polling               │
                                    ▼                                          │
┌────────────────────────────────────────────────────────────────────────────┐ │
│                              SYNCD                                          │ │
│   • Receives SAI commands from Redis                                        │ │
│   • Translates VID (Virtual ID) ↔ RID (Real ID)                            │ │
│   • Calls actual hardware SAI SDK via VendorSai                            │ │
└────────────────────────────────────────────────────────────────────────────┘ │
                                    │                                          │
                                    │ SAI API (vendor-specific)               │
                                    ▼                                          │
┌────────────────────────────────────────────────────────────────────────────┐ │
│                     HARDWARE SAI SDK (VendorSai)                            │ │
│   (Broadcom, Mellanox, Marvell, etc.)                                       │ │
└────────────────────────────────────────────────────────────────────────────┘ │
                                    │                                          │
                                    │                                          │
                                    ▼                                          │
┌────────────────────────────────────────────────────────────────────────────┐ │
│                          ASIC HARDWARE                                      │ │
│                  (Network switching silicon)                                │ │
└────────────────────────────────────────────────────────────────────────────┘ │
                                                                               │
                                                                               │
                ┌──────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────────────────────┐
│                          sairedis.rec                                       │
│   • Timestamp-prefixed log of all SAI operations                           │
│   • Format: YYYY-MM-DD.HH:MM:SS.microseconds|opcode|data                   │
│   • Located in configurable directory (default: current directory)         │
└────────────────────────────────────────────────────────────────────────────┘
```

## Who Writes sairedis.rec?

**The sairedis library (libsairedis)** writes the `sairedis.rec` file, NOT orchagent directly.

### Key Components:

| Component | Location | Role |
|-----------|----------|------|
| `Recorder` class | `sonic-sairedis/lib/Recorder.cpp` | Handles file operations, timestamps, thread-safe writes |
| `RedisRemoteSaiInterface` | `sonic-sairedis/lib/RedisRemoteSaiInterface.cpp` | Intercepts SAI calls and invokes recorder |
| Orchagent | `sonic-swss/orchagent/saihelper.cpp` | Configures recording via SAI attributes |

## When Does Recording Happen?

### 1. Configuration Phase (Orchagent Startup)

During orchagent initialization, `initSaiRedis()` in `orchagent/saihelper.cpp:315-388` configures recording:

```cpp
// Step 1: Set recording output directory
attr.id = SAI_REDIS_SWITCH_ATTR_RECORDING_OUTPUT_DIR;
attr.value.s8list.count = (uint32_t)record_location.size();
attr.value.s8list.list = (int8_t*)const_cast<char *>(record_location.c_str());
sai_switch_api->set_switch_attribute(gSwitchId, &attr);

// Step 2: Set recording filename
attr.id = SAI_REDIS_SWITCH_ATTR_RECORDING_FILENAME;
attr.value.s8list.count = (uint32_t)record_filename.size();
attr.value.s8list.list = (int8_t*)const_cast<char *>(record_filename.c_str());
sai_switch_api->set_switch_attribute(gSwitchId, &attr);

// Step 3: Enable recording
attr.id = SAI_REDIS_SWITCH_ATTR_RECORD;
attr.value.booldata = Recorder::Instance().sairedis.isRecord();
sai_switch_api->set_switch_attribute(gSwitchId, &attr);
```

### 2. Recording Phase (Runtime)

Every SAI API call is intercepted in `RedisRemoteSaiInterface.cpp`. Example for CREATE:

```cpp
// RedisRemoteSaiInterface.cpp:809-847
sai_status_t RedisRemoteSaiInterface::create(...) {
    // 1. Serialize attributes
    auto entry = SaiAttributeList::serialize_attr_list(object_type, attr_count, attr_list, false);

    const std::string key = serializedObjectType + ":" + serializedObjectId;

    // 2. RECORD the request BEFORE sending to Redis
    m_recorder->recordGenericCreate(key, entry);

    // 3. Send command to syncd via Redis
    m_communicationChannel->set(key, entry, REDIS_ASIC_STATE_COMMAND_CREATE);

    // 4. Wait for response from syncd
    auto status = waitForResponse(SAI_COMMON_API_CREATE);

    // 5. RECORD the response
    m_recorder->recordGenericCreateResponse(status);

    return status;
}
```

### 3. Actual File Write

The `Recorder::recordLine()` method in `Recorder.cpp:166-182` handles thread-safe writes:

```cpp
void Recorder::recordLine(const std::string& line) {
    MUTEX();  // Thread-safe lock

    if (!m_enabled) return;

    if (m_ofstream.is_open()) {
        m_ofstream << getTimestamp() << "|" << line << std::endl;
    }
}
```

## Recording Format

Each line in `sairedis.rec` follows this format:
```
YYYY-MM-DD.HH:MM:SS.microseconds|opcode|payload
```

### Operation Codes

| Code | Operation | Description |
|------|-----------|-------------|
| `c` | Create | Create a SAI object |
| `C` | Create Response | Response status from create |
| `r` | Remove | Remove a SAI object |
| `R` | Remove Response | Response status from remove |
| `s` | Set | Set attribute on SAI object |
| `S` | Set Response | Response status from set |
| `g` | Get | Get attribute from SAI object |
| `G` | Get Response | Response with attribute values |
| `f` | FDB Flush | Flush FDB entries |
| `F` | FDB Flush Response | Response status from flush |
| `q` | Query | Query operations (stats, capability) |
| `Q` | Query Response | Response from query |
| `n` | Notification | SAI notification (async events) |
| `#` | Comment | Log rotation markers, etc. |
| `@` | Sleep | Used by saiplayer for timing |

### Example Recording

```
2024-01-15.10:30:45.123456|c|SAI_OBJECT_TYPE_SWITCH:oid:0x21000000000000|SAI_SWITCH_ATTR_INIT_SWITCH=true
2024-01-15.10:30:45.234567|C|SAI_STATUS_SUCCESS
2024-01-15.10:30:45.345678|c|SAI_OBJECT_TYPE_PORT:oid:0x1000000000001|SAI_PORT_ATTR_SPEED=100000
2024-01-15.10:30:45.456789|C|SAI_STATUS_SUCCESS
2024-01-15.10:30:46.123456|s|SAI_OBJECT_TYPE_PORT:oid:0x1000000000001|SAI_PORT_ATTR_ADMIN_STATE=true
2024-01-15.10:30:46.234567|S|SAI_STATUS_SUCCESS
```

## Command-Line Configuration

Orchagent accepts CLI arguments to control recording (`orchagent/main.cpp`):

| Flag | Description | Default |
|------|-------------|---------|
| `-r <type>` | Recording bitfield | `0x7` (sairedis + swss + retry) |
| `-j <filename>` | sairedis.rec filename | `sairedis.rec` |
| `-d <directory>` | Recording directory | `.` (current directory) |

Recording type bitfield:
- Bit 0 (0x1): `SAIREDIS_RECORD_ENABLE` - sairedis.rec
- Bit 1 (0x2): `SWSS_RECORD_ENABLE` - swss.rec
- Bit 2 (0x4): `RESPONSE_PUBLISHER_RECORD_ENABLE` - responsepublisher.rec
- Bit 3 (0x8): `RETRY_RECORD_ENABLE` - retry.rec

## Log Rotation

Log rotation is supported via SIGHUP signal handler (`orchagent/main.cpp:122-131`):

```cpp
void sighup_handler(int signo) {
    Recorder::Instance().sairedis.setRotate(true);
    // ... other recorders
}
```

This can be triggered by:
- Sending `SIGHUP` to orchagent process
- Setting `SAI_REDIS_SWITCH_ATTR_PERFORM_LOG_ROTATE` attribute

## Replay Functionality (SaiPlayer)

The `saiplayer` utility (`sonic-sairedis/saiplayer/SaiPlayer.cpp`) can replay recordings:

```bash
saiplayer -r sairedis.rec
```

It parses the recording file and re-executes all SAI operations, useful for:
- Reproducing issues in test environments
- Testing SAI implementations
- Debugging hardware configuration problems

## Key Source Files

| File | Repository | Purpose |
|------|------------|---------|
| `lib/Recorder.h` | sonic-sairedis | Recorder class declaration |
| `lib/Recorder.cpp` | sonic-sairedis | Recording implementation |
| `lib/RedisRemoteSaiInterface.cpp` | sonic-sairedis | SAI call interception |
| `lib/sairedis.h` | sonic-sairedis | SAI Redis attribute definitions |
| `saiplayer/SaiPlayer.cpp` | sonic-sairedis | Recording replay engine |
| `orchagent/saihelper.cpp` | sonic-swss | Recording configuration |
| `orchagent/main.cpp` | sonic-swss | CLI argument parsing |
| `lib/recorder.cpp` | sonic-swss | Recorder singleton management |

## Important Notes

1. **Client-Side Recording**: Recording happens in the orchagent process (via libsairedis), NOT in syncd
2. **Thread Safety**: All recordings are mutex-protected for concurrent access
3. **Performance**: Recording adds minimal overhead; each line is immediately flushed
4. **Selective Recording**: Statistics operations can be disabled via `SAI_REDIS_SWITCH_ATTR_RECORD_STATS`
5. **Skip Filtering**: Certain GET operations can be skipped via `SkipRecordAttrContainer`

## Debugging Tips

1. **Enable Recording**: Ensure `-r` flag includes bit 0 (e.g., `-r 0x7`)
2. **Check File Location**: Default is current working directory of orchagent
3. **Verify Permissions**: Recording directory must be writable
4. **Watch Real-Time**: Use `tail -f sairedis.rec` to monitor live
5. **Search for Errors**: Look for response lines with non-success status
