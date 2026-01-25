# sairedis.rec Recording Flow in SONiC

This document explains the end-to-end flow of how `sairedis.rec` gets written as part of the SONiC SAI SDK architecture.

## Table of Contents

- [Overview](#overview)
- [Architecture Diagram](#architecture-diagram)
- [Orchagent ↔ Syncd Communication Architecture](#orchagent--syncd-communication-architecture)
  - [How ProducerTable/ConsumerTable Works](#how-producertableconsumertable-works)
  - [Queue Summary](#queue-summary)
- [Who Writes sairedis.rec?](#who-writes-sairedisrec)
- [Communication Modes: Sync vs Async](#communication-modes-sync-vs-async)
  - [Available Modes](#available-modes)
  - [The Critical Difference: waitForResponse()](#the-critical-difference-waitforresponse)
  - [Visual Flow Comparison](#visual-flow-comparison)
  - [Recording Implications](#recording-implications)
  - [Important: Syncd Always Writes Responses](#important-syncd-always-writes-responses)
  - [Mode Configuration](#mode-configuration)
  - [Trade-offs](#trade-offs)
- [When Does Recording Happen?](#when-does-recording-happen)
  - [1. Configuration Phase (Orchagent Startup)](#1-configuration-phase-orchagent-startup)
  - [2. Recording Phase (Runtime)](#2-recording-phase-runtime)
  - [3. Actual File Write](#3-actual-file-write)
- [Threading Model and Performance](#threading-model-and-performance)
  - [Recording Happens on Main Thread](#recording-happens-on-main-thread)
  - [Why Synchronous Recording?](#why-synchronous-recording)
  - [Blocking Cost per Recording](#blocking-cost-per-recording)
  - [Why It's Usually Acceptable](#why-its-usually-acceptable)
  - [When Recording Could Be A Problem](#when-recording-could-be-a-problem)
  - [Disabling Recording for Performance](#disabling-recording-for-performance)
- [Recording Format](#recording-format)
  - [Operation Codes](#operation-codes)
  - [Example Recording (Sync Mode)](#example-recording-sync-mode)
- [Command-Line Configuration](#command-line-configuration)
- [Log Rotation](#log-rotation)
- [Replay Functionality (SaiPlayer)](#replay-functionality-saiplayer)
- [Key Source Files](#key-source-files)
- [Important Notes](#important-notes)
- [Debugging Tips](#debugging-tips)

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

## Orchagent ↔ Syncd Communication Architecture

Orchagent and syncd communicate using **two separate Redis LIST-based queues** (ProducerTable/ConsumerTable pattern), one for each direction:

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                              ORCHAGENT                                        │
│                         (RedisChannel.cpp)                                    │
│                                                                               │
│   m_asicState   = ProducerTable(ASIC_STATE)      ──── SENDS requests         │
│   m_getConsumer = ConsumerTable(GETRESPONSE)     ──── RECEIVES responses     │
└──────────────────────────────────────────────────────────────────────────────┘
                          │                              ▲
                          │ LPUSH + PUBLISH              │ RPOP (pop)
                          ▼                              │
┌──────────────────────────────────────────────────────────────────────────────┐
│                              REDIS (ASIC_DB)                                  │
│                                                                               │
│   ASIC_STATE_KEY_VALUE_OP_QUEUE          GETRESPONSE_KEY_VALUE_OP_QUEUE      │
│   [request] ← [request] ← ...            [response] ← [response] ← ...       │
│                                                                               │
│   ASIC_STATE_CHANNEL (pub/sub)           GETRESPONSE_CHANNEL (pub/sub)       │
│   (wake-up signal only)                  (wake-up signal only)               │
└──────────────────────────────────────────────────────────────────────────────┘
                          │                              ▲
                          │ RPOP (pop)                   │ LPUSH + PUBLISH
                          ▼                              │
┌──────────────────────────────────────────────────────────────────────────────┐
│                              SYNCD                                            │
│                    (RedisSelectableChannel.cpp)                               │
│                                                                               │
│   m_asicState   = ConsumerTable(ASIC_STATE)      ──── RECEIVES requests      │
│   m_getResponse = ProducerTable(GETRESPONSE)     ──── SENDS responses        │
└──────────────────────────────────────────────────────────────────────────────┘
```

### How ProducerTable/ConsumerTable Works

This is **NOT pub/sub for data** - the pub/sub channel is only for wake-up notifications:

```cpp
// From sonic-swss-common/common/producertable.cpp:36-38
string luaEnque =
    "redis.call('LPUSH', KEYS[1], ARGV[1], ARGV[2], ARGV[3]);"  // Data → Redis LIST
    "redis.call('PUBLISH', KEYS[2], ARGV[4]);";                  // Wake-up signal only
```

**Why it's "point-to-point":**
- Data is stored in a Redis LIST (queue)
- Consumer uses `RPOP` - once popped, the message is **gone**
- Only ONE consumer gets each message (unlike pub/sub broadcast)
- The pub/sub channel just signals "data available" - it doesn't carry the data

### Queue Summary

| Queue | Orchagent Role | Syncd Role | Direction |
|-------|----------------|------------|-----------|
| `ASIC_STATE_KEY_VALUE_OP_QUEUE` | Producer (LPUSH) | Consumer (RPOP) | Orchagent → Syncd |
| `GETRESPONSE_KEY_VALUE_OP_QUEUE` | Consumer (RPOP) | Producer (LPUSH) | Syncd → Orchagent |

## Who Writes sairedis.rec?

**The sairedis library (libsairedis)** writes the `sairedis.rec` file, NOT orchagent directly.

### Key Components:

| Component | Location | Role |
|-----------|----------|------|
| `Recorder` class | `sonic-sairedis/lib/Recorder.cpp` | Handles file operations, timestamps, thread-safe writes |
| `RedisRemoteSaiInterface` | `sonic-sairedis/lib/RedisRemoteSaiInterface.cpp` | Intercepts SAI calls and invokes recorder |
| Orchagent | `sonic-swss/orchagent/saihelper.cpp` | Configures recording via SAI attributes |

## Communication Modes: Sync vs Async

Orchagent can operate in different communication modes that fundamentally affect how SAI calls behave and how responses are recorded.

### Available Modes

| Mode | `m_syncMode` | Buffering | Wait for Response? |
|------|--------------|-----------|-------------------|
| `SAI_REDIS_COMMUNICATION_MODE_REDIS_ASYNC` | `false` | Enabled (pipelined) | **No** - fire and forget |
| `SAI_REDIS_COMMUNICATION_MODE_REDIS_SYNC` | `true` | Disabled | **Yes** - blocks until response |
| `SAI_REDIS_COMMUNICATION_MODE_ZMQ_SYNC` | `true` | Disabled | **Yes** - blocks until response |

### The Critical Difference: waitForResponse()

Both modes call `waitForResponse()`, but the behavior differs based on `m_syncMode`:

```cpp
// RedisRemoteSaiInterface.cpp:902-924
sai_status_t RedisRemoteSaiInterface::waitForResponse(sai_common_api_t api)
{
    if (m_syncMode)
    {
        // SYNC MODE: Actually wait for syncd's response
        auto status = m_communicationChannel->wait(REDIS_ASIC_STATE_COMMAND_GETRESPONSE, kco);
        m_recorder->recordGenericResponse(status);  // Record REAL status
        return status;
    }

    // ASYNC MODE: Return immediately without waiting
    // Response is NOT recorded!
    return SAI_STATUS_SUCCESS;  // Assumed success
}
```

### Visual Flow Comparison

```
         SYNC MODE                                     ASYNC MODE
         ─────────                                     ──────────

         Orchagent                                     Orchagent
             │                                             │
             ▼                                             ▼
         create()                                      create()
             │                                             │
             ▼                                             ▼
       record request                                record request
             │                                             │
             ▼                                             ▼
       send to Redis                                 send to Redis (buffered/pipelined)
             │                                             │
             ▼                                             ▼
     waitForResponse()                               waitForResponse()
             │                                             │
             ▼                                             ▼
    ┌─────────────────┐                             ┌─────────────────┐
    │ m_syncMode=true │                             │ m_syncMode=false│
    │                 │                             │                 │
    │ BLOCK waiting   │                             │ return SUCCESS  │◄── Fire and forget!
    │ for syncd...    │                             │ immediately     │
    │                 │                             │                 │
    │ record response │                             │ (no recording)  │
    └────────┬────────┘                             └─────────────────┘
             │                                             │
             ▼                                             ▼
  [syncd processes, calls ASIC]                      return to caller
             │                                       (assumes success)
             ▼
      return REAL status
```

### Recording Implications

**Sync Mode sairedis.rec:**
```
2024-01-15.10:30:45.123456|c|SAI_OBJECT_TYPE_PORT:oid:0x1000000000001|SAI_PORT_ATTR_SPEED=100000
2024-01-15.10:30:45.234567|C|SAI_STATUS_SUCCESS    ← Real response recorded
```

**Async Mode sairedis.rec:**
```
2024-01-15.10:30:45.123456|c|SAI_OBJECT_TYPE_PORT:oid:0x1000000000001|SAI_PORT_ATTR_SPEED=100000
                                                    ← NO response line!
```

### Important: Syncd Always Writes Responses

Syncd does NOT know what mode the client is using. It **always** writes responses to `GETRESPONSE`:

```cpp
// syncd/Syncd.cpp:967 - syncd always sends response
m_selectableChannel->set(sai_serialize_status(status), {}, REDIS_ASIC_STATE_COMMAND_GETRESPONSE);
```

In async mode, these responses accumulate in the queue but orchagent never reads them (for create/set/remove operations).

### Mode Configuration

**Mode is GLOBAL** - applies to all SAI calls from orchagent, not per-request:

```cpp
// orchagent/main.cpp:63 - Global variable
sai_redis_communication_mode_t gRedisCommunicationMode = SAI_REDIS_COMMUNICATION_MODE_REDIS_ASYNC;

// Set via CLI flag at startup
case 'z':
    sai_deserialize_redis_communication_mode(optarg, gRedisCommunicationMode);
    break;
```

**Can mode change at runtime?**
- Technically: Yes, the SAI attribute can be set again
- Practically: No, orchagent doesn't expose any mechanism to change it after startup
- Changing mid-stream would cause ordering/consistency issues

### Trade-offs

| Aspect | Sync Mode | Async Mode |
|--------|-----------|------------|
| **Performance** | Slower (waits for each op) | Faster (pipelined) |
| **Reliability** | Knows if operation succeeded | Assumes success |
| **Debugging** | Full request+response in recording | Only requests recorded |
| **Error Detection** | Immediate | Delayed (via notifications) |
| **Use Case** | Production (recommended) | Bulk loading, testing |

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

    // 3. Send command to syncd via Redis (ProducerTable)
    m_communicationChannel->set(key, entry, REDIS_ASIC_STATE_COMMAND_CREATE);

    // 4. Wait for response (behavior depends on sync/async mode!)
    auto status = waitForResponse(SAI_COMMON_API_CREATE);

    // 5. RECORD the response (only in sync mode!)
    m_recorder->recordGenericCreateResponse(status);

    return status;
}
```

### 3. Actual File Write

The `Recorder::recordLine()` method in `Recorder.cpp:166-182` handles thread-safe writes:

```cpp
// Recorder.cpp:23 - MUTEX is a blocking lock_guard
#define MUTEX() std::lock_guard<std::mutex> _lock(m_mutex)

// Recorder.cpp:166-182
void Recorder::recordLine(const std::string& line) {
    MUTEX();  // Acquire lock (blocks if contention)

    if (!m_enabled) return;

    if (m_ofstream.is_open()) {
        // Write + flush (std::endl forces flush)
        m_ofstream << getTimestamp() << "|" << line << std::endl;
    }
}
```

## Threading Model and Performance

### Recording Happens on Main Thread

Recording is **synchronous and blocking** - it happens directly on orchagent's main thread:

```
Orchagent Main Thread
         │
         ▼
    SAI API call (e.g., sai_port_api->create_port())
         │
         ▼
    RedisRemoteSaiInterface::create()
         │
         ├──► m_recorder->recordGenericCreate()  ◄── BLOCKS (mutex + file I/O)
         │
         ├──► m_communicationChannel->set()      ◄── BLOCKS (Redis write)
         │
         └──► waitForResponse()                  ◄── BLOCKS (sync mode) or returns (async)
         │
         ├──► m_recorder->recordGenericCreateResponse()  ◄── BLOCKS (mutex + file I/O)
         │
         ▼
    Return to orchagent
```

### Why Synchronous Recording?

1. **Simplicity**: No async complexity, no background threads
2. **Ordering**: Recording matches exact execution order
3. **Crash Safety**: If crash occurs, last recorded line shows exactly where execution stopped

### Blocking Cost per Recording

| Operation | Typical Time |
|-----------|--------------|
| Mutex acquire (no contention) | ~nanoseconds |
| `getTimestamp()` (gettimeofday + strftime) | ~1-2 μs |
| File write + flush | ~10-100 μs (filesystem dependent) |
| **Total per SAI call** | **~10-100 μs** |

### Why It's Usually Acceptable

- SAI calls to hardware (via syncd) take **milliseconds**
- Recording overhead is typically <1% of total SAI call time
- OS filesystem usually buffers writes even with `std::endl`

### When Recording Could Be A Problem

| Scenario | Impact |
|----------|--------|
| Slow filesystem (NFS, failing disk) | Significant delays |
| Async mode bulk operations | Recording overhead becomes more visible (not waiting for syncd) |
| High-frequency stats polling | Consider disabling stats recording |

### Disabling Recording for Performance

```bash
# Disable all recording
orchagent -r 0

# Or at runtime via SAI attribute
sai_attribute_t attr;
attr.id = SAI_REDIS_SWITCH_ATTR_RECORD;
attr.value.booldata = false;
sai_switch_api->set_switch_attribute(gSwitchId, &attr);

# Disable only stats recording (high frequency)
attr.id = SAI_REDIS_SWITCH_ATTR_RECORD_STATS;
attr.value.booldata = false;
sai_switch_api->set_switch_attribute(gSwitchId, &attr);
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
| `C` | Create Response | Response status from create (sync mode only) |
| `r` | Remove | Remove a SAI object |
| `R` | Remove Response | Response status from remove (sync mode only) |
| `s` | Set | Set attribute on SAI object |
| `S` | Set Response | Response status from set (sync mode only) |
| `g` | Get | Get attribute from SAI object |
| `G` | Get Response | Response with attribute values |
| `f` | FDB Flush | Flush FDB entries |
| `F` | FDB Flush Response | Response status from flush |
| `q` | Query | Query operations (stats, capability) |
| `Q` | Query Response | Response from query |
| `n` | Notification | SAI notification (async events) |
| `#` | Comment | Log rotation markers, etc. |
| `@` | Sleep | Used by saiplayer for timing |

### Example Recording (Sync Mode)

```
2024-01-15.10:30:45.123456|c|SAI_OBJECT_TYPE_SWITCH:oid:0x21000000000000|SAI_SWITCH_ATTR_INIT_SWITCH=true
2024-01-15.10:30:45.234567|C|SAI_STATUS_SUCCESS
2024-01-15.10:30:45.345678|c|SAI_OBJECT_TYPE_PORT:oid:0x1000000000001|SAI_PORT_ATTR_SPEED=100000
2024-01-15.10:30:45.456789|C|SAI_STATUS_SUCCESS
2024-01-15.10:30:46.123456|s|SAI_OBJECT_TYPE_PORT:oid:0x1000000000001|SAI_PORT_ATTR_ADMIN_STATE=true
2024-01-15.10:30:46.234567|S|SAI_STATUS_SUCCESS
```

## Command-Line Configuration

Orchagent accepts CLI arguments to control recording and communication mode (`orchagent/main.cpp`):

| Flag | Description | Default |
|------|-------------|---------|
| `-r <type>` | Recording bitfield | `0x7` (sairedis + swss + retry) |
| `-j <filename>` | sairedis.rec filename | `sairedis.rec` |
| `-d <directory>` | Recording directory | `.` (current directory) |
| `-z <mode>` | Communication mode | `redis_async` |
| `-s` | Enable sync mode (deprecated) | disabled |

**Recording type bitfield:**
- Bit 0 (0x1): `SAIREDIS_RECORD_ENABLE` - sairedis.rec
- Bit 1 (0x2): `SWSS_RECORD_ENABLE` - swss.rec
- Bit 2 (0x4): `RESPONSE_PUBLISHER_RECORD_ENABLE` - responsepublisher.rec
- Bit 3 (0x8): `RETRY_RECORD_ENABLE` - retry.rec

**Communication mode values for `-z`:**
- `redis_async` - Async mode with pipelining (default)
- `redis_sync` - Sync mode via Redis
- `zmq_sync` - Sync mode via ZeroMQ

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
| `lib/RedisRemoteSaiInterface.cpp` | sonic-sairedis | SAI call interception, sync/async handling |
| `lib/RedisChannel.cpp` | sonic-sairedis | Orchagent-side communication (Producer/Consumer) |
| `meta/RedisSelectableChannel.cpp` | sonic-sairedis | Syncd-side communication (Consumer/Producer) |
| `lib/sairedis.h` | sonic-sairedis | SAI Redis attribute definitions |
| `saiplayer/SaiPlayer.cpp` | sonic-sairedis | Recording replay engine |
| `syncd/Syncd.cpp` | sonic-sairedis | Syncd main processing loop |
| `orchagent/saihelper.cpp` | sonic-swss | Recording and mode configuration |
| `orchagent/main.cpp` | sonic-swss | CLI argument parsing |
| `common/producertable.cpp` | sonic-swss-common | ProducerTable implementation |
| `common/consumertable.cpp` | sonic-swss-common | ConsumerTable implementation |

## Important Notes

1. **Client-Side Recording**: Recording happens in the orchagent process (via libsairedis), NOT in syncd
2. **Thread Safety**: All recordings are mutex-protected for concurrent access
3. **Performance**: Recording adds minimal overhead; each line is immediately flushed
4. **Selective Recording**: Statistics operations can be disabled via `SAI_REDIS_SWITCH_ATTR_RECORD_STATS`
5. **Skip Filtering**: Certain GET operations can be skipped via `SkipRecordAttrContainer`
6. **Async Mode Limitation**: In async mode, response status is NOT recorded for create/set/remove operations
7. **Syncd Always Responds**: Syncd writes responses regardless of client mode; async mode just doesn't read them

## Debugging Tips

1. **Enable Recording**: Ensure `-r` flag includes bit 0 (e.g., `-r 0x7`)
2. **Check File Location**: Default is current working directory of orchagent
3. **Verify Permissions**: Recording directory must be writable
4. **Watch Real-Time**: Use `tail -f sairedis.rec` to monitor live
5. **Search for Errors**: Look for response lines with non-success status
6. **Check Mode**: If responses are missing, orchagent may be in async mode
7. **Use Sync Mode for Debugging**: Run with `-z redis_sync` for complete request/response recording
