# ASIC DB Design Rationale: Past, Present, and Future

## Table of Contents

- [Introduction](#introduction)
- [Historical Context](#historical-context)
  - [Timeline](#timeline)
  - [Technology Available in 2015-2016](#technology-available-in-2015-2016)
- [Why LIST + Pub/Sub Design?](#why-list--pubsub-design)
  - [Requirements](#requirements)
  - [Design Decisions](#design-decisions)
  - [The Atomic State Update Problem](#the-atomic-state-update-problem)
- [Why Not Redis Streams?](#why-not-redis-streams)
  - [Redis Streams Overview](#redis-streams-overview)
  - [Comparison](#comparison)
  - [Migration Cost Analysis](#migration-cost-analysis)
- [Modern Architecture Options](#modern-architecture-options)
  - [Option 1: gRPC (Recommended)](#option-1-grpc-recommended)
  - [Option 2: Redis Streams + Lua](#option-2-redis-streams--lua)
  - [Option 3: Hybrid gRPC + Redis](#option-3-hybrid-grpc--redis)
  - [Option 4: Shared Memory + Lock-free Queue](#option-4-shared-memory--lock-free-queue)
- [Architecture Comparison Matrix](#architecture-comparison-matrix)
- [Recommended Modern Design](#recommended-modern-design)
- [Migration Path](#migration-path)
- [Conclusion](#conclusion)

---

## Introduction

This document explains the design rationale behind SONiC's ASIC DB communication mechanism between `orchagent` and `syncd`, and explores what modern alternatives would be considered if designing the system today.

The current design uses:
- **Redis LIST** as a message queue
- **Redis Pub/Sub** for wakeup notifications
- **Redis HASH** for state snapshots
- **Lua scripts** for atomic operations

---

## Historical Context

### Timeline

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           SONiC Development Timeline                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  2015      SONiC project initiated at Microsoft                            │
│     │      Redis 3.0 available (LIST, HASH, Pub/Sub, Lua scripting)        │
│     │                                                                       │
│  2016      Initial architecture design                                      │
│     │      Decision: Redis as central database + IPC mechanism             │
│     │      Pattern: ProducerTable/ConsumerTable with LIST + Pub/Sub        │
│     │                                                                       │
│  2017      SONiC open-sourced                                              │
│     │      Architecture becomes de-facto standard                          │
│     │                                                                       │
│  2018      Redis 5.0 released with Streams                                 │
│     │      SONiC architecture already mature and deployed                  │
│     │                                                                       │
│  2019+     gRPC/gNMI adopted for management plane                          │
│     │      Data plane still uses Redis-based IPC                           │
│     │                                                                       │
│  Today     Legacy design remains due to stability and migration cost       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Technology Available in 2015-2016

| Technology | Status | Notes |
|------------|--------|-------|
| Redis LIST | ✓ Available | Stable since Redis 1.0 |
| Redis HASH | ✓ Available | Stable since Redis 2.0 |
| Redis Pub/Sub | ✓ Available | Stable since Redis 2.0 |
| Redis Lua | ✓ Available | Since Redis 2.6 (2012) |
| Redis Streams | ✗ Not available | Released in Redis 5.0 (2018) |
| gRPC | ✓ Available | Released 2015, but less mature |
| ZeroMQ | ✓ Available | Mature, but adds dependency |

**Decision factors:**
1. Redis already chosen as central state database
2. Natural to extend Redis for IPC
3. Lua scripting enabled atomic operations
4. Minimize external dependencies

---

## Why LIST + Pub/Sub Design?

### Requirements

The orchagent-syncd communication required:

1. **Reliable message delivery** - No message loss
2. **Ordered processing** - FIFO semantics
3. **State persistence** - For warm boot recovery
4. **Low latency** - Hardware programming is time-sensitive
5. **Debugging visibility** - Operators need to inspect state
6. **Atomic state updates** - Consistent view of ASIC state

### Design Decisions

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Design Decision Tree                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Q: How to transport requests from orchagent to syncd?                     │
│     │                                                                       │
│     ├─► Option A: Direct socket/pipe                                       │
│     │   └─► Rejected: Need persistence for warm boot                       │
│     │                                                                       │
│     ├─► Option B: Message queue (RabbitMQ, Kafka)                          │
│     │   └─► Rejected: Additional dependency, operational complexity        │
│     │                                                                       │
│     └─► Option C: Redis-based queue ✓                                      │
│         └─► Accepted: Already using Redis, Lua for atomicity              │
│                                                                             │
│  Q: How to notify syncd of new messages?                                   │
│     │                                                                       │
│     ├─► Option A: Polling                                                  │
│     │   └─► Rejected: Wasteful, adds latency                              │
│     │                                                                       │
│     └─► Option B: Pub/Sub notification ✓                                   │
│         └─► Accepted: Low latency wakeup, minimal overhead                │
│                                                                             │
│  Q: How to maintain current state for warm boot?                           │
│     │                                                                       │
│     └─► Option: HASH tables updated atomically with queue pop ✓            │
│         └─► Accepted: Lua script ensures atomicity                        │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### The Atomic State Update Problem

A critical requirement is updating the state HASH atomically with the queue pop:

```lua
-- consumer_table_pops.lua
-- This MUST be atomic - either both happen or neither

-- 1. Pop from queue
local keys = redis.call('LRANGE', KEYS[1], -popsize, -1)
redis.call('LTRIM', KEYS[1], 0, -popsize-1)

-- 2. Update state HASH
if op == 'set' or op == 'create' then
    redis.call('HSET', keyname, attr, value)
elseif dbop == 'D' then
    redis.call('DEL', keyname)
end
```

**Why atomicity matters:**
- If pop succeeds but HASH update fails → inconsistent state
- If HASH update succeeds but pop fails → duplicate processing
- Lua script guarantees all-or-nothing execution

---

## Why Not Redis Streams?

### Redis Streams Overview

Redis Streams (available since Redis 5.0, 2018) provide:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Redis Streams Features                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ✓ Consumer groups      - Multiple consumers can share workload            │
│  ✓ Message acknowledgment - Explicit XACK for reliable delivery            │
│  ✓ Message persistence  - Messages retained until explicitly deleted       │
│  ✓ Message replay       - Re-read old messages                             │
│  ✓ Automatic IDs        - Timestamp-based message IDs                      │
│  ✓ Blocking reads       - XREADGROUP blocks until data available           │
│  ✓ Monitoring           - XINFO for queue depth, consumer lag              │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Comparison

| Feature | Current LIST Approach | Redis Streams |
|---------|----------------------|---------------|
| Available since | Redis 1.0 | Redis 5.0 (2018) |
| Message ordering | FIFO ✓ | FIFO ✓ |
| Persistence | ✓ (until popped) | ✓ (configurable) |
| Consumer groups | ✗ (single consumer) | ✓ Built-in |
| Message acknowledgment | Implicit (pop removes) | Explicit (XACK) |
| Message replay | ✗ (removed after pop) | ✓ |
| Blocking read | Requires Pub/Sub | Built-in (XREADGROUP BLOCK) |
| Atomic with HASH update | ✓ (Lua script) | Requires Lua anyway |
| Monitoring | Manual | XINFO, XLEN |

### Migration Cost Analysis

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Migration Cost Assessment                              │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Code Changes Required:                                                     │
│  ├── sonic-swss-common                                                      │
│  │   ├── Rewrite ProducerTable (LPUSH → XADD)                              │
│  │   ├── Rewrite ConsumerTable (LRANGE → XREADGROUP)                       │
│  │   ├── Rewrite consumer_table_pops.lua                                   │
│  │   └── Update all table base classes                                     │
│  │                                                                          │
│  ├── sonic-sairedis                                                         │
│  │   ├── Update RedisChannel                                               │
│  │   ├── Update syncd consumer logic                                       │
│  │   └── Handle XACK acknowledgments                                       │
│  │                                                                          │
│  └── sonic-swss                                                             │
│      └── Minimal changes (uses abstractions)                               │
│                                                                             │
│  Testing Required:                                                          │
│  ├── Full regression on all platforms                                      │
│  ├── Warm boot validation                                                  │
│  ├── Fast boot validation                                                  │
│  ├── Performance benchmarking                                              │
│  └── All vendor ASIC validation                                            │
│                                                                             │
│  Risk Assessment:                                                           │
│  ├── HIGH: Breaking change to fundamental IPC                              │
│  ├── HIGH: Warm boot state format changes                                  │
│  └── MEDIUM: Performance characteristics may differ                        │
│                                                                             │
│  Benefit:                                                                   │
│  └── Cleaner code, but functionally equivalent                             │
│                                                                             │
│  Conclusion: Cost outweighs benefit for mature, production system          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Modern Architecture Options

If designing SONiC today, these are the viable options:

### Option 1: gRPC (Recommended)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           gRPC-based Architecture                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────┐         gRPC (HTTP/2 + Protobuf)        ┌───────────┐       │
│  │ orchagent │◄───────────────────────────────────────►│   syncd   │       │
│  │           │   • Bidirectional streaming              │           │       │
│  │           │   • Request/Response for SAI ops         │           │       │
│  │           │   • Server streaming for notifications   │           │       │
│  └───────────┘                                          └───────────┘       │
│        │                                                      │             │
│        │              ┌─────────────┐                         │             │
│        └─────────────►│    Redis    │◄────────────────────────┘             │
│                       │  (State DB) │                                       │
│                       │             │  • HASH for current state             │
│                       │             │  • Updated after SAI success          │
│                       │             │  • Used for warm boot only            │
│                       └─────────────┘                                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Protobuf Schema Example:**

```protobuf
syntax = "proto3";

package sai;

service SaiService {
    // Unary RPCs for synchronous operations
    rpc Create(CreateRequest) returns (CreateResponse);
    rpc Remove(RemoveRequest) returns (RemoveResponse);
    rpc Set(SetRequest) returns (SetResponse);
    rpc Get(GetRequest) returns (GetResponse);

    // Bidirectional streaming for bulk operations
    rpc BulkOperations(stream SaiRequest) returns (stream SaiResponse);

    // Server streaming for async notifications
    rpc SubscribeNotifications(NotificationFilter) returns (stream Notification);
}

message SetRequest {
    ObjectType object_type = 1;
    string object_id = 2;
    repeated Attribute attributes = 3;
}

message SetResponse {
    Status status = 1;
}

message Notification {
    NotificationType type = 1;
    oneof data {
        PortStateChange port_state = 2;
        FdbEvent fdb_event = 3;
        // ... other notification types
    }
}
```

**Advantages:**
| Aspect | Benefit |
|--------|---------|
| Type safety | Protobuf prevents serialization bugs |
| Performance | HTTP/2 multiplexing, binary protocol |
| Bidirectional | Native support for notifications |
| Already in SONiC | gNMI/gNOI use gRPC |
| Code generation | Auto-generate C++/Python stubs |
| Testability | Easy to mock for unit tests |

### Option 2: Redis Streams + Lua

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                       Redis Streams Architecture                            │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────┐                                          ┌───────────┐       │
│  │ orchagent │                                          │   syncd   │       │
│  └─────┬─────┘                                          └─────┬─────┘       │
│        │                                                      │             │
│        │ XADD                                        XREADGROUP│             │
│        │                                                      │             │
│        ▼                                                      ▼             │
│  ┌─────────────────────────────────────────────────────────────────┐       │
│  │                         Redis                                    │       │
│  │  ┌─────────────────────────────────────────────────────────┐    │       │
│  │  │  ASIC_STATE_STREAM                                      │    │       │
│  │  │  • Consumer group: syncd_group                          │    │       │
│  │  │  • Automatic message IDs                                │    │       │
│  │  │  • Explicit acknowledgment (XACK)                       │    │       │
│  │  └─────────────────────────────────────────────────────────┘    │       │
│  │                                                                  │       │
│  │  ┌─────────────────────────────────────────────────────────┐    │       │
│  │  │  ASIC_STATE:* (HASH)                                    │    │       │
│  │  │  • Updated atomically via Lua                           │    │       │
│  │  └─────────────────────────────────────────────────────────┘    │       │
│  └─────────────────────────────────────────────────────────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Lua Script for Atomic Stream Consumer:**

```lua
-- stream_consumer.lua
local stream_key = KEYS[1]
local hash_prefix = KEYS[2]
local group = ARGV[1]
local consumer = ARGV[2]
local count = ARGV[3]

-- Read from stream
local messages = redis.call('XREADGROUP', 'GROUP', group, consumer,
                            'COUNT', count, 'STREAMS', stream_key, '>')

if not messages or #messages == 0 then
    return {}
end

local results = {}
for _, stream_data in ipairs(messages) do
    local entries = stream_data[2]
    for _, entry in ipairs(entries) do
        local id = entry[1]
        local fields = entry[2]

        -- Parse and update HASH atomically
        local key = fields[2]  -- object key
        local op = fields[4]   -- operation
        local attrs = fields[6] -- attributes

        local hash_key = hash_prefix .. ':' .. key

        if op == 'create' or op == 'set' then
            -- Parse attributes and HSET
            redis.call('HSET', hash_key, unpack(attrs))
        elseif op == 'remove' then
            redis.call('DEL', hash_key)
        end

        -- Acknowledge message
        redis.call('XACK', stream_key, group, id)

        table.insert(results, {id, key, op, attrs})
    end
end

return results
```

### Option 3: Hybrid gRPC + Redis

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      Hybrid gRPC + Redis Architecture                       │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                        ┌─────────────────────┐                              │
│                        │   Control Plane     │                              │
│                        │   (gNMI/gNOI)       │                              │
│                        └──────────┬──────────┘                              │
│                                   │                                         │
│                                   ▼                                         │
│  ┌───────────┐      gRPC       ┌───────────┐      SAI      ┌──────────┐   │
│  │ orchagent │◄───────────────►│   syncd   │◄─────────────►│   ASIC   │   │
│  │           │  (Fast path)    │           │               │          │   │
│  │           │  • SAI CRUD     │           │               │          │   │
│  │           │  • Bulk ops     │           │               │          │   │
│  │           │  • Notifications│           │               │          │   │
│  └─────┬─────┘                 └─────┬─────┘               └──────────┘   │
│        │                             │                                     │
│        │                             │                                     │
│        │      ┌─────────────┐        │                                     │
│        └─────►│    Redis    │◄───────┘                                     │
│               │             │                                              │
│               │ • STATE_DB  │  State persistence (no queue)                │
│               │ • CONFIG_DB │  Configuration                               │
│               │ • COUNTER_DB│  Statistics                                  │
│               │             │                                              │
│               └─────────────┘                                              │
│                     │                                                       │
│                     ▼                                                       │
│              ┌─────────────┐                                               │
│              │  Debugging  │  redis-cli, warm boot recovery                │
│              │  Monitoring │                                               │
│              └─────────────┘                                               │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Key Design Points:**
1. **gRPC for communication**: All SAI requests/responses
2. **Redis for state only**: No queue role, just persistence
3. **State updated after SAI success**: Always consistent
4. **Clean separation**: Transport (gRPC) vs Storage (Redis)

### Option 4: Shared Memory + Lock-free Queue

For ultra-low latency requirements:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Shared Memory Architecture                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ┌───────────┐                                          ┌───────────┐       │
│  │ orchagent │                                          │   syncd   │       │
│  └─────┬─────┘                                          └─────┬─────┘       │
│        │                                                      │             │
│        │              Shared Memory Region                    │             │
│        │    ┌─────────────────────────────────────────┐      │             │
│        └───►│  Lock-free SPSC Queue                   │◄─────┘             │
│             │  (Single Producer, Single Consumer)     │                     │
│             │                                         │                     │
│             │  • Memory-mapped file for persistence   │                     │
│             │  • Eventfd for notification             │                     │
│             │  • ~1μs latency                         │                     │
│             └─────────────────────────────────────────┘                     │
│                                                                             │
│             ┌─────────────────────────────────────────┐                     │
│             │  Redis (STATE_DB)                       │                     │
│             │  • Updated asynchronously               │                     │
│             │  • For warm boot / debugging only       │                     │
│             └─────────────────────────────────────────┘                     │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

**Trade-offs:**
| Aspect | Consideration |
|--------|---------------|
| Latency | Lowest (~1μs vs ~100μs for Redis) |
| Complexity | Higher - memory management, synchronization |
| Debugging | Harder - no redis-cli visibility |
| Persistence | Requires memory-mapped file |
| Portability | Platform-specific |

---

## Architecture Comparison Matrix

| Criteria | Current (LIST+Pub/Sub) | Redis Streams | gRPC | gRPC+Redis | Shared Memory |
|----------|------------------------|---------------|------|------------|---------------|
| **Latency** | ~100μs | ~100μs | ~10-50μs | ~10-50μs | ~1μs |
| **Type safety** | None | None | Protobuf ✓ | Protobuf ✓ | Manual |
| **Sync mode** | Extra code | Extra code | Native ✓ | Native ✓ | Native |
| **Notifications** | Separate | Separate | Built-in ✓ | Built-in ✓ | Eventfd |
| **Debugging** | redis-cli ✓ | redis-cli ✓ | grpcurl | redis-cli ✓ | Custom tools |
| **Warm boot** | HASH ✓ | HASH ✓ | Need Redis | HASH ✓ | mmap file |
| **Code complexity** | High (Lua) | Medium | Low ✓ | Low ✓ | High |
| **Schema evolution** | Manual | Manual | Protobuf ✓ | Protobuf ✓ | Manual |
| **Existing in SONiC** | Yes ✓ | No | Yes (gNMI) | Partial | No |
| **Multi-consumer** | No | Yes ✓ | Yes ✓ | Yes ✓ | No |

---

## Recommended Modern Design

**If designing SONiC today, the recommended architecture is: gRPC + Redis (Option 3)**

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     RECOMMENDED ARCHITECTURE                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Communication Layer: gRPC                                                  │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  • Protobuf for type-safe serialization                            │    │
│  │  • Bidirectional streaming for notifications                       │    │
│  │  • Native sync/async support                                       │    │
│  │  • HTTP/2 for multiplexing                                         │    │
│  │  • Already familiar to SONiC team (gNMI/gNOI)                      │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  State Layer: Redis (no queue role)                                        │
│  ┌────────────────────────────────────────────────────────────────────┐    │
│  │  • HASH tables for current ASIC state                              │    │
│  │  • Updated AFTER successful SAI call (always consistent)           │    │
│  │  • Used for warm boot recovery                                     │    │
│  │  • Debugging with redis-cli                                        │    │
│  │  • No complex Lua scripts                                          │    │
│  └────────────────────────────────────────────────────────────────────┘    │
│                                                                             │
│  Benefits:                                                                  │
│  ├── Clean separation of concerns (transport vs state)                    │
│  ├── Strong typing prevents serialization bugs                            │
│  ├── State always consistent (updated after SAI success)                  │
│  ├── Easier testing (mock gRPC service)                                   │
│  ├── Future-proof (streaming telemetry, distributed syncd)               │
│  └── Lower latency than Redis-based queue                                 │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Migration Path

For existing SONiC deployments wanting to modernize:

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                        Incremental Migration Path                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  Phase 1: Add gRPC Interface (Parallel)                                    │
│  ├── Add gRPC server to syncd alongside existing Redis consumer           │
│  ├── Define Protobuf schemas for SAI operations                           │
│  ├── Implement gRPC handlers that call existing SAI processing            │
│  └── No changes to orchagent yet                                          │
│                                                                             │
│  Phase 2: Orchagent gRPC Client                                            │
│  ├── Add gRPC client option to sairedis                                   │
│  ├── Feature flag to choose Redis or gRPC                                 │
│  ├── Extensive testing with gRPC path                                     │
│  └── Keep Redis path as fallback                                          │
│                                                                             │
│  Phase 3: Simplify Redis Usage                                             │
│  ├── Redis becomes state-only (no queue)                                  │
│  ├── State updates moved to after SAI success                             │
│  ├── Remove Lua scripts for queue operations                              │
│  └── Keep HASH tables for warm boot                                       │
│                                                                             │
│  Phase 4: Deprecate Redis Queue                                            │
│  ├── Remove LIST-based queue code                                         │
│  ├── Remove Pub/Sub notification code                                     │
│  ├── gRPC becomes the only communication path                             │
│  └── Redis purely for state persistence                                   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## Conclusion

The current LIST + Pub/Sub design in SONiC was a pragmatic choice given the technology available in 2015-2016:

1. **Redis was already the central database** - Natural to extend for IPC
2. **Lua scripts enabled atomicity** - Critical for consistent state updates
3. **No Redis Streams** - Not available until 2018
4. **Simplicity** - Fewer external dependencies

**If designing today**, the recommended approach is **gRPC + Redis**:

1. **gRPC for communication** - Type-safe, bidirectional, low-latency
2. **Redis for state only** - No complex queue logic, always consistent
3. **Clean architecture** - Separation of transport and storage
4. **Future-proof** - Supports distributed syncd, streaming telemetry

The current design works well and is proven in production. Migration should only be considered if there are compelling requirements (e.g., distributed syncd, strict consistency) that justify the engineering investment.
