# ASIC DB State Table: Bridge Between Orchagent and Syncd

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
