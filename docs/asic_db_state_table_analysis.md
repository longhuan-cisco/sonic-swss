# ASIC DB State Table: Bridge Between Orchagent and Syncd

## Table of Contents

- [Overview](#overview)
- [Architecture](#architecture)
- [Data Structures](#data-structures)
  - [1. LIST (Request Queue)](#1-list-request-queue)
  - [2. HASH (State Snapshot)](#2-hash-state-snapshot)
  - [3. Pub/Sub Channel](#3-pubsub-channel)
- [Request Format (KeyOpFieldsValuesTuple)](#request-format-keyopfieldsvaluestuple)
- [Communication Modes](#communication-modes)
  - [ASYNC Mode](#async-mode-default-sai_redis_communication_mode_redis_async)
  - [SYNC Mode](#sync-mode-sai_redis_communication_mode_redis_sync)
- [Who Updates the HASH?](#who-updates-the-hash)
- [Key Code Locations](#key-code-locations)
  - [sonic-swss-common](#sonic-swss-common)
  - [sonic-sairedis](#sonic-sairedis)
  - [sonic-swss](#sonic-swss)
- [Race Condition Handling](#race-condition-handling)
  - [Scenario: SET followed by quick DEL](#scenario-set-followed-by-quick-del)
  - [Scenario: Rapid attribute changes](#scenario-rapid-attribute-changes)
- [Concrete Example: Port Admin State Set](#concrete-example-port-admin-state-set)
  - [Step 1: User Configuration](#step-1-user-configuration)
  - [Step 2: Orchagent Processing](#step-2-orchagent-processing)
  - [Step 3: Sairedis Serialization](#step-3-sairedis-serialization)
  - [Step 4: Push to Redis (ProducerTable)](#step-4-push-to-redis-producertable)
  - [Step 5: Redis State After Push](#step-5-redis-state-after-push)
  - [Step 6: Syncd Wakeup and Pop](#step-6-syncd-wakeup-and-pop)
  - [Step 7: Syncd Command Dispatch](#step-7-syncd-command-dispatch)
  - [Step 8: SAI API Call to Hardware](#step-8-sai-api-call-to-hardware)
  - [Step 9: Final State](#step-9-final-state)
  - [Complete Flow Diagram](#complete-flow-diagram)
  - [Key Code Files in the Flow](#key-code-files-in-the-flow)
- [Summary](#summary)

---

## Overview

The ASIC DB serves as the communication bridge between `orchagent` and `syncd` in SONiC. It uses a **hybrid mechanism** combining a request queue (Redis LIST) with pub/sub notifications for wakeup signals.

## Architecture

```
┌─────────────┐                    ┌─────────────────────────────────────┐                    ┌──────────┐
│  orchagent  │                    │            ASIC_DB (Redis)          │                    │  syncd   │
│             │                    │                                     │                    │          │
│  SAI API    │────LPUSH──────────►│  LIST: ASIC_STATE_KEY_VALUE_OP_QUEUE│───LRANGE/LTRIM───►│  pop()   │
│  (sairedis) │                    │  (Request Queue - FIFO)             │                   │          │
│             │                    │                                     │                    │          │
│             │────PUBLISH────────►│  Channel: ASIC_STATE_CHANNEL        │───SUBSCRIBE──────►│  wakeup  │
│             │   (notification)   │  (Wakeup signal only, no data)      │                   │          │
│             │                    │                                     │                    │          │
│             │                    │  HASH: ASIC_STATE:SAI_OBJECT_TYPE:OID                   │          │
│             │                    │  (Current state snapshot)           │◄──────────────────│  update  │
└─────────────┘                    └─────────────────────────────────────┘                    └──────────┘
```

## Data Structures

### 1. LIST (Request Queue)
- **Name**: `ASIC_STATE_KEY_VALUE_OP_QUEUE`
- **Purpose**: Stores all SAI requests in FIFO order
- **Operations**: `LPUSH` (producer), `LRANGE`/`LTRIM` (consumer)
- **Data preserved**: ALL operations in order, no data loss

### 2. HASH (State Snapshot)
- **Name**: `ASIC_STATE:SAI_OBJECT_TYPE_xxx:oid`
- **Purpose**: Current state view of ASIC objects
- **Usage**: Warm boot recovery, state comparison, debugging
- **Note**: Latest value wins (intermediate values not preserved)

### 3. Pub/Sub Channel
- **Name**: `ASIC_STATE_CHANNEL`
- **Purpose**: Wakeup notification only (contains no actual data)
- **Mechanism**: Signals syncd that work is available in the queue

## Request Format (KeyOpFieldsValuesTuple)

```
┌─────────────────────────────────────────────────────────────────────────┐
│  KEY:    "SAI_OBJECT_TYPE_ROUTE_ENTRY:{\"dest\":\"10.0.0.0/24\",...}"  │
│  OP:     "create" | "remove" | "set" | "get" | "bulkcreate" | ...      │
│  VALUES: [("SAI_ROUTE_ENTRY_ATTR_NEXT_HOP_ID", "oid:0x12345"), ...]    │
└─────────────────────────────────────────────────────────────────────────┘
```

## Communication Modes

### ASYNC Mode (Default: `SAI_REDIS_COMMUNICATION_MODE_REDIS_ASYNC`)

```
Orchagent                          Redis                              Syncd
    │                                │                                   │
    │──LPUSH (key,value,op)─────────►│                                   │
    │──PUBLISH (wakeup)─────────────►│                                   │
    │                                │───notification──────────────────►│
    │                                │                                   │
    │                                │◄──EVALSHA consumer_table_pops.lua─│
    │                                │   ├─ LRANGE/LTRIM (pop queue)     │
    │                                │   └─ HSET/DEL (update HASH)       │
    │                                │                                   │
    │                                │                          SAI call │
    │  (continues without waiting)   │                                   │
```

- **HASH updated by**: Lua script during `pops()` (atomic, before SAI call)
- **Consistency**: Eventual (HASH may show intended state before SAI succeeds)
- **Performance**: High throughput, pipelined operations
- **`m_modifyRedis`**: `true`

### SYNC Mode (`SAI_REDIS_COMMUNICATION_MODE_REDIS_SYNC`)

```
Orchagent                          Redis                              Syncd
    │                                │                                   │
    │──LPUSH (key,value,op)─────────►│                                   │
    │──PUBLISH (wakeup)─────────────►│                                   │
    │                                │───notification──────────────────►│
    │                                │                                   │
    │                                │◄──EVALSHA (pop only, NO HASH)────│
    │                                │                                   │
    │                                │                          SAI call │
    │                                │                                   │
    │                                │◄──HSET/DEL (only if SAI OK)──────│
    │                                │                                   │
    │◄─────────────────────────────response─────────────────────────────│
```

- **HASH updated by**: Syncd C++ code after SAI success (`syncUpdateRedisQuadEvent()`)
- **Consistency**: Strong (HASH always reflects actual ASIC state)
- **Performance**: Lower throughput, blocking operations
- **`m_modifyRedis`**: `false`

## Who Updates the HASH?

**Both modes: Syncd process controls the HASH update**

| Mode | Trigger | Execution Location | Timing |
|------|---------|-------------------|--------|
| ASYNC | `ConsumerTable::pops()` → Lua script | Lua runs in Redis server | During pop (before SAI) |
| SYNC | `syncUpdateRedisQuadEvent()` | C++ code in syncd | After SAI success |

## Key Code Locations

### sonic-swss-common
| File | Purpose |
|------|---------|
| `common/producertable.cpp` | LPUSH to queue + PUBLISH notification |
| `common/consumertable.cpp` | EVALSHA to pop + optional HASH update |
| `common/consumer_table_pops.lua` | Atomic pop + HASH update logic |
| `common/redisselect.cpp` | Pub/sub subscription handling |

### sonic-sairedis
| File | Purpose |
|------|---------|
| `syncd/Syncd.cpp` | Main event loop, command processing |
| `syncd/Syncd.cpp:3253` | `syncUpdateRedisQuadEvent()` - SYNC mode HASH update |
| `syncd/Syncd.cpp:147` | `modifyRedis` flag based on mode |
| `meta/RedisSelectableChannel.cpp` | ConsumerTable wrapper for syncd |

### sonic-swss
| File | Purpose |
|------|---------|
| `orchagent/saihelper.cpp` | SAI/Redis initialization |
| `orchagent/orchdaemon.cpp` | Flush mechanism for pipelined commands |

## Race Condition Handling

### Scenario: SET followed by quick DEL

```
Queue: [..., (key1, attrs, "Sset"), (key1, {}, "Dremove")]
              ↑ older                 ↑ newer
```

- **LIST**: Both operations preserved in order
- **HASH**: Only final state (deleted)
- **Syncd**: Processes BOTH operations, executes BOTH SAI calls
- **Result**: No data loss in request path

### Scenario: Rapid attribute changes

```
T1: SET route nexthop=10.0.0.1
T2: SET route nexthop=10.0.0.2
T3: SET route nexthop=10.0.0.3
```

- **LIST**: All 3 operations preserved
- **HASH**: Only shows nexthop=10.0.0.3
- **Syncd**: Executes all 3 SAI `set_attribute` calls
- **Result**: Correct behavior, HASH is just a snapshot

## Concrete Example: Port Admin State Set

This example traces the complete flow when an operator sets a port admin state to UP.

### Step 1: User Configuration

```bash
# User runs command to bring up Ethernet0
config interface startup Ethernet0
```

This writes to CONFIG_DB, which triggers PortsOrch in orchagent.

### Step 2: Orchagent Processing

**File: `orchagent/portsorch.cpp:2140-2154`**

```cpp
bool PortsOrch::setPortAdminStatus(Port &port, bool state)
{
    sai_attribute_t attr;
    attr.id = SAI_PORT_ATTR_ADMIN_STATE;
    attr.value.booldata = state;   // true = UP

    // This calls into sairedis library
    sai_status_t status = sai_port_api->set_port_attribute(port.m_port_id, &attr);
    ...
}
```

### Step 3: Sairedis Serialization

**File: `lib/RedisRemoteSaiInterface.cpp:872-893`**

```cpp
sai_status_t RedisRemoteSaiInterface::set(
        sai_object_type_t objectType,           // SAI_OBJECT_TYPE_PORT
        const std::string &serializedObjectId,  // "oid:0x1000000000002"
        const sai_attribute_t *attr)            // SAI_PORT_ATTR_ADMIN_STATE=true
{
    // Serialize attribute to field-value tuple
    auto entry = SaiAttributeList::serialize_attr_list(objectType, 1, attr, false);
    // entry = [("SAI_PORT_ATTR_ADMIN_STATE", "true")]

    // Build key: "SAI_OBJECT_TYPE_PORT:oid:0x1000000000002"
    std::string key = serializedObjectType + ":" + serializedObjectId;

    // Push to Redis via communication channel
    m_communicationChannel->set(key, entry, REDIS_ASIC_STATE_COMMAND_SET);
    ...
}
```

### Step 4: Push to Redis (ProducerTable)

**File: `lib/RedisChannel.cpp:117-124`**

```cpp
void RedisChannel::set(const std::string& key,
        const std::vector<swss::FieldValueTuple>& values,
        const std::string& command)
{
    m_asicState->set(key, values, command);  // ProducerTable
}
```

**File: `common/producertable.cpp:36-38` (Lua script)**

```lua
redis.call('LPUSH', KEYS[1], ARGV[1], ARGV[2], ARGV[3]);  -- Push to LIST
redis.call('PUBLISH', KEYS[2], ARGV[4]);                   -- Notify syncd
```

### Step 5: Redis State After Push

```
ASIC_STATE_KEY_VALUE_OP_QUEUE (LIST):
┌─────────────────────────────────────────────────────────────────────────────┐
│ KEY:    "SAI_OBJECT_TYPE_PORT:oid:0x1000000000002"                         │
│ VALUE:  "[\"SAI_PORT_ATTR_ADMIN_STATE\",\"true\"]"                         │
│ OP:     "Sset"  (S prefix = Set operation)                                 │
└─────────────────────────────────────────────────────────────────────────────┘

ASIC_STATE_CHANNEL: PUBLISH "G" (wakeup signal)
```

### Step 6: Syncd Wakeup and Pop

**File: `syncd/Syncd.cpp:341-362`**

```cpp
void Syncd::processEvent(sairedis::SelectableChannel& consumer)
{
    do {
        swss::KeyOpFieldsValuesTuple kco;
        consumer.pop(kco, isInitViewMode());  // Triggers Lua script
        processSingleEvent(kco);
    } while (!consumer.empty());
}
```

**File: `common/consumer_table_pops.lua:4-6,55-76`**

```lua
-- Pop from queue
local keys = redis.call('LRANGE', KEYS[1], -popsize, -1)
redis.call('LTRIM', KEYS[1], 0, -popsize-1)

-- Update HASH (in ASYNC mode, m_modifyRedis=true)
if op == 'set' then
    redis.call('HSET', keyname, ret[st], ret[st+1])
    -- HSET ASIC_STATE:SAI_OBJECT_TYPE_PORT:oid:0x1000000000002
    --      SAI_PORT_ATTR_ADMIN_STATE "true"
end
```

### Step 7: Syncd Command Dispatch

**File: `syncd/Syncd.cpp:365-391`**

```cpp
sai_status_t Syncd::processSingleEvent(const swss::KeyOpFieldsValuesTuple &kco)
{
    auto& key = kfvKey(kco);  // "SAI_OBJECT_TYPE_PORT:oid:0x1000000000002"
    auto& op = kfvOp(kco);    // "set"

    if (op == REDIS_ASIC_STATE_COMMAND_SET)
        return processQuadEvent(SAI_COMMON_API_SET, kco);
    ...
}
```

### Step 8: SAI API Call to Hardware

**File: `syncd/Syncd.cpp:3450-3484`**

```cpp
sai_status_t Syncd::processQuadEvent(sai_common_api_t api,
        const swss::KeyOpFieldsValuesTuple &kco)
{
    const std::string& key = kfvKey(kco);
    auto& values = kfvFieldsValues(kco);

    // Deserialize: key → objectType + objectId
    sai_object_meta_key_t metaKey;
    sai_deserialize_object_meta_key(key, metaKey);
    // metaKey.objecttype = SAI_OBJECT_TYPE_PORT
    // metaKey.objectkey.key.object_id = 0x1000000000002 (VID)

    // Convert attributes
    SaiAttributeList list(metaKey.objecttype, values, false);
    sai_attribute_t *attr_list = list.get_attr_list();
    // attr_list[0].id = SAI_PORT_ATTR_ADMIN_STATE
    // attr_list[0].value.booldata = true

    // Translate VID to RID and call vendor SAI
    return processEntry(metaKey, api, attr_count, attr_list);
}
```

**File: `syncd/Syncd.cpp:2313-2380`**

```cpp
sai_status_t Syncd::processEntry(sai_object_meta_key_t metaKey,
        sai_common_api_t api, ...)
{
    // Translate Virtual ID (VID) to Real ID (RID)
    m_translator->translateVidToRid(metaKey);
    // VID 0x1000000000002 → RID 0x2 (actual hardware port ID)

    // Call vendor SAI implementation
    switch (api) {
        case SAI_COMMON_API_SET:
            return info->set(&metaKey, attr_list);
            // → sai_port_api->set_port_attribute(RID, attr)
            // → Hardware register write: Port admin state = UP
    }
}
```

### Step 9: Final State

```
┌─────────────────────────────────────────────────────────────────────────────┐
│ ASIC_STATE_KEY_VALUE_OP_QUEUE (LIST): empty (popped)                       │
├─────────────────────────────────────────────────────────────────────────────┤
│ ASIC_STATE:SAI_OBJECT_TYPE_PORT:oid:0x1000000000002 (HASH):                │
│   SAI_PORT_ATTR_ADMIN_STATE = "true"                                       │
│   SAI_PORT_ATTR_SPEED = "100000"                                           │
│   ... (other attributes)                                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│ Hardware: Port Ethernet0 admin state = UP                                  │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Complete Flow Diagram

```
┌──────────────────────────────────────────────────────────────────────────────────────────┐
│                           PORT ADMIN STATE SET FLOW                                      │
└──────────────────────────────────────────────────────────────────────────────────────────┘

User: config interface startup Ethernet0
                │
                ▼
┌───────────────────────────┐
│       CONFIG_DB           │
│  PORT|Ethernet0           │
│    admin_status: up       │
└───────────────────────────┘
                │
                ▼
┌───────────────────────────┐
│       Orchagent           │
│  PortsOrch::doTask()      │
│           │               │
│           ▼               │
│  setPortAdminStatus()     │
│           │               │
│           ▼               │
│  sai_port_api->           │
│    set_port_attribute()   │
└───────────────────────────┘
                │
                ▼
┌───────────────────────────┐
│       Sairedis            │
│  RedisRemoteSaiInterface  │
│    ::set()                │
│           │               │
│           ▼               │
│  Serialize:               │
│  key="SAI_OBJECT_TYPE_    │
│       PORT:oid:0x10..."   │
│  val=["SAI_PORT_ATTR_     │
│       ADMIN_STATE","true"]│
│  op="set"                 │
└───────────────────────────┘
                │
                ▼
┌───────────────────────────┐
│    Redis (ASIC_DB)        │
│                           │
│  LPUSH to LIST ◄──────────│── Request queued
│  PUBLISH to channel       │
│           │               │
│  ┌────────┴────────┐      │
│  │ HASH updated    │      │── ASYNC: Lua updates during pop
│  │ (by Lua or      │      │── SYNC: syncd updates after SAI OK
│  │  syncd)         │      │
│  └─────────────────┘      │
└───────────────────────────┘
                │
        (notification)
                │
                ▼
┌───────────────────────────┐
│        Syncd              │
│  Syncd::processEvent()    │
│           │               │
│           ▼               │
│  ConsumerTable::pop()     │
│           │               │
│           ▼               │
│  processSingleEvent()     │
│           │               │
│           ▼               │
│  processQuadEvent()       │
│           │               │
│           ▼               │
│  Translate VID→RID        │
│           │               │
│           ▼               │
│  Vendor SAI call:         │
│  sai_port_api->           │
│    set_port_attribute()   │
└───────────────────────────┘
                │
                ▼
┌───────────────────────────┐
│     ASIC Hardware         │
│  Port register write      │
│  Admin state = ENABLED    │
└───────────────────────────┘
```

### Key Code Files in the Flow

| Step | Component | File | Function/Line |
|------|-----------|------|---------------|
| 1 | Orchagent | `orchagent/portsorch.cpp` | `setPortAdminStatus()` :2140 |
| 2 | Sairedis | `lib/RedisRemoteSaiInterface.cpp` | `set()` :872 |
| 3 | Sairedis | `lib/RedisChannel.cpp` | `set()` :117 |
| 4 | swss-common | `common/producertable.cpp` | Lua script :36 |
| 5 | swss-common | `common/consumer_table_pops.lua` | Pop + HASH update |
| 6 | Syncd | `syncd/Syncd.cpp` | `processEvent()` :341 |
| 7 | Syncd | `syncd/Syncd.cpp` | `processSingleEvent()` :365 |
| 8 | Syncd | `syncd/Syncd.cpp` | `processQuadEvent()` :3450 |
| 9 | Syncd | `syncd/Syncd.cpp` | `processEntry()` :2313 |

## Summary

| Component | Purpose | Data Loss Risk |
|-----------|---------|----------------|
| LIST (Queue) | Request transport | None - all ops preserved in FIFO order |
| HASH (State) | State snapshot | Latest wins, but not used for request processing |
| Pub/Sub | Wakeup signal | N/A - contains no data |

The ASIC DB is **NOT pure pub/sub** - it's a **queue + notification** system where:
1. The LIST ensures reliable, ordered delivery of all SAI requests
2. Pub/sub provides low-latency wakeup (no polling needed)
3. The HASH provides a convenient state view for recovery/debugging
