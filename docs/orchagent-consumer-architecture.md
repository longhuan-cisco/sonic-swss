# Orchagent Consumer Architecture

This document explains how orchagent consumes data from Redis databases using different consumer mechanisms, and how the main event loop processes these events.

## Table of Contents

1. [Overview](#overview)
2. [Threading Model](#threading-model)
3. [Select Loop Architecture](#select-loop-architecture)
4. [Redis Notification Mechanisms](#redis-notification-mechanisms)
5. [Consumer Classes](#consumer-classes)
6. [Reliability Comparison](#reliability-comparison)
7. [Producer-Consumer Model](#producer-consumer-model)
8. [Code References](#code-references)

---

## Overview

Orchagent is the core component in SONiC that translates high-level configuration from Redis databases into SAI (Switch Abstraction Interface) API calls. It uses a **single-threaded event loop** to process changes from multiple Redis tables.

### Key Components

| Component | Purpose |
|-----------|---------|
| `Select` | Event multiplexer using epoll |
| `Selectable` | Base interface for event sources |
| `ConsumerStateTable` | Consumes from APPL_DB (producer-consumer queue pattern) |
| `SubscriberStateTable` | Consumes from CONFIG_DB/STATE_DB (keyspace notifications) |
| `NotificationConsumer` | Consumes async events from syncd (JSON in pub/sub) |
| `Consumer` | Wrapper that connects a table to an Orch |
| `Orch` | Base class for all orchestrators (PortsOrch, RouteOrch, etc.) |

---

## Threading Model

Orchagent uses a **single-threaded event loop** design where all Orchs share the same main thread.

```
┌─────────────────────────────────────────────────────────────────┐
│                     MAIN THREAD                                 │
│                                                                 │
│  OrchDaemon::start()                                            │
│      │                                                          │
│      └─► while(true)                                            │
│              ├─► Select::select()  ← blocks waiting for events  │
│              ├─► Consumer::execute()                            │
│              └─► for each Orch: doTask()                        │
│                                                                 │
│  All Orchs run here: PortsOrch, RouteOrch, NeighOrch, etc.     │
└─────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────┐
│                 RING BUFFER THREAD (optional)                   │
│                                                                 │
│  Only for high-volume route processing                          │
│  Decouples route updates from main select loop                  │
└─────────────────────────────────────────────────────────────────┘
```

### Implications

- **No parallelism** between Orchs - they're cooperative
- **Blocking** - if `doPortTask()` takes long, all other Orchs wait
- **SAI calls** - all SAI API calls happen on main thread (serialized)
- **Simplicity** - no locks needed between Orchs

---

## Select Loop Architecture

### Main Event Loop

**File:** `orchagent/orchdaemon.cpp:911-1050`

```cpp
void OrchDaemon::start()
{
    // 1. Register all selectables from each orch
    for (Orch *o : m_orchList) {
        m_select->addSelectables(o->getSelectables());
    }

    while (true) {
        Selectable *s;
        int ret = m_select->select(&s, SELECT_TIMEOUT);  // Blocks here

        if (ret == Select::TIMEOUT) {
            for (Orch *o : m_orchList)
                o->doTask();
            continue;
        }

        // Event received - dispatch to handler
        auto *c = (Executor *)s;
        c->execute();

        // Process remaining tasks for all orchs
        for (Orch *o : m_orchList)
            o->doTask();
    }
}
```

### How Select::select() Works

**File:** `sonic-swss-common/common/select.cpp:92-165`

```cpp
int Select::poll_descriptors(Selectable **c, unsigned int timeout, ...)
{
    // 1. Wait for any fd to become readable
    ret = ::epoll_wait(m_epoll_fd, events.data(), sz_selectables, timeout);

    // 2. For each ready fd, call readData()
    for (int i = 0; i < ret; ++i) {
        Selectable* sel = m_objects[fd];
        sel->readData();          // Read notification from socket
        m_ready.insert(sel);
    }

    // 3. Return one Selectable that hasData()
    while (!m_ready.empty()) {
        auto sel = *m_ready.begin();
        m_ready.erase(sel);

        if (!sel->hasData())
            continue;

        *c = sel;

        if (sel->hasCachedData())    // More data pending
            m_ready.insert(sel);     // Re-add for fair scheduling

        sel->updateAfterRead();      // Decrement queue length
        return Select::OBJECT;
    }
    return Select::TIMEOUT;
}
```

---

## Redis Notification Mechanisms

All three consumer mechanisms in SONiC are fundamentally based on **Redis Pub/Sub**. Redis Keyspace Notifications are a specialized form of pub/sub where Redis defines the payload format.

```
                        Redis Pub/Sub (foundation)
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
  Custom Channel         Custom Channel         Keyspace Notifications
  Payload: "G"           Payload: JSON          Payload: key + event type
  (signal only)          (full data)            (no values - Redis limitation)
        │                      │                      │
        ▼                      ▼                      ▼
ConsumerStateTable    NotificationConsumer    SubscriberStateTable
  + KEY_SET              (self-contained)       (must fetch data)
  + Staging hash
```

### 1. Pub/Sub with Signal (ConsumerStateTable)

**Explicit messaging** - producer publishes a simple signal.

```bash
# Producer
PUBLISH PORT_TABLE_CHANNEL@0 "G"

# Consumer receives:
1) "message"
2) "PORT_TABLE_CHANNEL@0"
3) "G"                        # Just a signal, no useful data
```

Data is stored separately in staging hash + KEY_SET, fetched by Lua script during `pops()`.

### 2. Pub/Sub with JSON Payload (NotificationConsumer)

**Self-contained messages** - full data embedded in JSON payload.

```bash
# Producer (syncd)
PUBLISH NOTIFICATIONS '[["port_state_change","oid:0x1000"],["state","up"]]'

# Consumer receives:
1) "message"
2) "NOTIFICATIONS"
3) '[["port_state_change","oid:0x1000"],["state","up"]]'   # Full data!
```

### 3. Keyspace Notifications (SubscriberStateTable)

**Automatic notifications** - Redis automatically notifies when keys change.

```bash
# Any client writes to Redis
HSET "PORT|Ethernet0" speed 100000

# Consumer receives (automatic):
1) "pmessage"
2) "__keyspace@4__:PORT|*"
3) "__keyspace@4__:PORT|Ethernet0"      # Key that changed
4) "hset"                               # Operation type (NO VALUES!)
```

Must be enabled: `redis-cli CONFIG SET notify-keyspace-events AKE`

### Why Redis Keyspace Notifications Don't Include Values

Redis designed keyspace notifications as lightweight signals, not data carriers. This is a deliberate design choice:

#### 1. Performance / Efficiency

```
Scenario: HSET with 100 fields, 1000 subscribers

With values:    100 fields × 1000 subscribers = 100,000 field copies
Without values: 1 event × 1000 subscribers = 1,000 tiny messages
```

#### 2. Pub/Sub Buffer Pressure

```
Redis pub/sub has client-output-buffer-limit:
  client-output-buffer-limit pubsub 32mb 8mb 60

With values: Buffer fills quickly → MORE dropped notifications
Without values: Tiny messages → Buffer lasts longer
```

Ironically, adding values could make reliability **worse**.

#### 3. Not All Consumers Need Values

```
Consumer A: "I just need to know something changed" (cache invalidation)
Consumer B: "I need the actual data" (sync)

Current design lets Consumer A stay lightweight.
```

#### 4. Data Type Complexity

```
STRING:  Simple value
HASH:    Which fields changed? All fields? Just modified ones?
LIST:    Which elements? Head? Tail? Index?
SET:     Which members added/removed?
```

No universal "value" representation works for all types.

#### 5. Design Philosophy

Redis keyspace notifications = **"WHAT changed"** (event signal)
Not designed for = **"WHAT + VALUE"** (change data capture)

For full change data capture, Redis offers Redis Streams or replication.

#### Impact on SONiC

This Redis limitation is why `SubscriberStateTable` has reliability issues - it must fetch data separately, creating race conditions. SONiC works around this with `ConsumerStateTable` (staging + KEY_SET) for critical paths.

### Comparison Table

| Aspect | ConsumerStateTable | NotificationConsumer | SubscriberStateTable |
|--------|-------------------|---------------------|---------------------|
| **Subscription** | Custom channel | Custom channel | Keyspace pattern |
| **Trigger** | Explicit `PUBLISH` | Explicit `PUBLISH` | Automatic on write |
| **Message content** | `"G"` (signal) | Full JSON data | Key + event type |
| **Data source** | Staging hash | Message itself | Must fetch from table |
| **Producer coupling** | ProducerStateTable | NotificationProducer | Any client |

---

## Consumer Classes

### ConsumerStateTable

Used for **APPL_DB** - high-reliability table synchronization.

#### Two-Phase Data Retrieval

1. **`readData()`** - Read pub/sub notification (counts, discards content)
2. **`pops()`** - Fetch actual data via atomic Lua script

#### readData() Implementation

**File:** `sonic-swss-common/common/redisselect.cpp:25-53`

```cpp
uint64_t RedisSelect::readData()
{
    redisReply *reply = nullptr;
    redisGetReply(m_subscribe->getContext(), reinterpret_cast<void**>(&reply));

    freeReplyObject(reply);     // Discard - content is just "G"
    m_queueLength++;            // Just count notifications

    // Drain any additional buffered notifications
    do {
        status = redisGetReplyFromReader(..., &reply);
        if (reply != nullptr && status == REDIS_OK) {
            m_queueLength++;
            freeReplyObject(reply);
        }
    } while (reply != nullptr && status == REDIS_OK);
    return 0;
}
```

#### pops() Lua Script

**File:** `sonic-swss-common/common/consumer_state_table_pops.lua`

```lua
local ret = {}
local tablename = KEYS[2]                           -- "PORT_TABLE:"
local stateprefix = ARGV[2]                         -- "_"

-- 1. Pop keys from the KEY_SET (atomically)
local keys = redis.call('SPOP', KEYS[1], ARGV[1])

for i = 1, n do
   local key = keys[i]

   -- 2. Check if key was marked for deletion
   local num = redis.call('SREM', KEYS[3], key)
   if num == 1 then
      redis.call('DEL', tablename..key)
   end

   -- 3. Get field-values from STAGING hash
   local fieldvalues = redis.call('HGETALL', stateprefix..tablename..key)
   table.insert(ret, {key, fieldvalues})

   -- 4. Copy to PERMANENT table
   for i = 1, #fieldvalues, 2 do
      redis.call('HSET', tablename..key, fieldvalues[i], fieldvalues[i + 1])
   end

   -- 5. Clean up staging hash
   redis.call('DEL', stateprefix..tablename..key)
end
return ret
```

**Key insight:** The **consumer** writes to the actual DB table, not the producer.

#### Redis Data Structures

| Redis Key | Type | Written By | Read By | Purpose |
|-----------|------|------------|---------|---------|
| `_PORT_TABLE:Ethernet0` | HASH | Producer | Consumer | Staging area |
| `PORT_TABLE:Ethernet0` | HASH | Consumer | Anyone | Real table |
| `PORT_TABLE_KEY_SET` | SET | Producer | Consumer | Pending keys |
| `PORT_TABLE_DEL_SET` | SET | Producer | Consumer | Keys to delete |
| `PORT_TABLE_CHANNEL@0` | PUBSUB | Producer | Consumer | Notification |

---

### SubscriberStateTable

Used for **CONFIG_DB / STATE_DB** - simpler but less reliable.

#### Why It Overrides readData()

The notification **contains the key name**, so it must be saved:

```cpp
uint64_t SubscriberStateTable::readData() {
    redisGetReply(..., &reply);
    m_keyspace_event_buffer.emplace_back(reply);  // SAVES the notification!
    // ... drain more
}
```

#### pops() - Must Fetch Data Separately

```cpp
void SubscriberStateTable::pops(...) {
    while (auto event = popEventBuffer()) {
        string key = parseKeyFromChannel(message.channel);

        if (message.data == "del") {
            kfvOp(kco) = DEL_COMMAND;
        } else {
            m_table.get(key, kfvFieldsValues(kco));  // FETCH from table!
            kfvOp(kco) = SET_COMMAND;
        }
    }
}
```

**Problem:** By the time `pops()` runs, the table data may have changed.

---

### NotificationConsumer

Used for **SAI async events** from syncd - self-contained messages.

#### Use Cases

| Orch | Channel | Purpose |
|------|---------|---------|
| `PortsOrch` | NOTIFICATIONS | Port oper status from SAI |
| `FdbOrch` | NOTIFICATIONS | FDB learn/age events |
| `BfdOrch` | NOTIFICATIONS | BFD session state changes |
| `PfcWdOrch` | NOTIFICATIONS | PFC watchdog events |
| `MACsecOrch` | NOTIFICATIONS | MACsec completion events |
| `SwitchOrch` | RESTARTCHECK | Warm restart check |

#### Why NotificationConsumer for SAI Events?

SAI events are **asynchronous hardware events** - the data exists only at the moment of the callback:

```
Hardware event (link up)
        │
        ▼
    SAI callback in syncd
        │
        ├─► Serialize to JSON (captures state NOW)
        └─► PUBLISH to NOTIFICATIONS
                │
                ▼
        orchagent (NotificationConsumer)
                │
                └─► Data is IN the message (safe)
```

---

## Reliability Comparison

### Overview

| Scenario | ConsumerStateTable | NotificationConsumer | SubscriberStateTable |
|----------|-------------------|---------------------|---------------------|
| SET then quick DEL | ✅ Safe (staging) | ✅ Safe (data in msg) | ❌ **Data lost** |
| Multiple rapid updates | ✅ Gets latest | ⚠️ Each processed | ⚠️ Each processed |
| Consumer slow/blocked | ✅ Data persists | ❌ **Buffer overflow** | ❌ **Buffer overflow** |
| Consumer restart | ✅ KEY_SET persists | ❌ **Missed** | ❌ **Missed** |
| Producer crash mid-write | ✅ Atomic Lua | ✅ Atomic publish | ⚠️ Partial state |

### Failure Scenarios

#### SET Followed by Quick DEL

**SubscriberStateTable (fails):**
```
Time 0ms: HSET PORT|Eth0 speed 100000 → notification "hset"
Time 1ms: DEL PORT|Eth0 → notification "del"
Time 5ms: Consumer pops "hset" → HGETALL PORT|Eth0 → EMPTY! ❌
```

**ConsumerStateTable (safe):**
```
Time 0ms: HSET _PORT_TABLE:Eth0, SADD KEY_SET
Time 1ms: SADD DEL_SET Eth0
Time 5ms: Lua script checks DEL_SET → correctly handles as DEL ✅
```

#### Consumer Restart

**SubscriberStateTable (misses data):**
```
Time 0: Consumer subscribed
Time 1: HSET PORT|Eth0 → notification sent
Time 2: Consumer crashes
Time 3: HSET PORT|Eth4 → notification lost! ❌
Time 4: Consumer restarts
```

**ConsumerStateTable (recovers):**
```
Time 0-3: Producer writes → KEY_SET = {Eth0, Eth4, Eth8}
Time 4: Consumer restarts
Time 5: pops() → Gets ALL pending keys ✅
```

### Full Comparison

| Aspect | ConsumerStateTable | NotificationConsumer | SubscriberStateTable |
|--------|-------------------|---------------------|---------------------|
| **Reliability** | ✅ High | ⚠️ Medium | ⚠️ Lower |
| **Data persistence** | ✅ Staging + KEY_SET | ❌ Transient | ❌ Transient |
| **Crash recovery** | ✅ Full recovery | ❌ Missed | ❌ Missed |
| **SET→DEL race** | ✅ Handled correctly | ✅ Data in message | ❌ Data loss |
| **Atomicity** | ✅ Lua script | ✅ Single publish | ❌ None |
| **Rapid updates** | ✅ Coalesced | ⚠️ Each processed | ⚠️ Each processed |
| **Buffer overflow** | ✅ KEY_SET unlimited | ❌ Buffer limited | ❌ Buffer limited |
| **Producer coupling** | ❌ ProducerStateTable | ❌ NotificationProducer | ✅ Any client |
| **Setup complexity** | ⚠️ More complex | ✅ Simple | ✅ Simple |
| **Best use case** | Table sync (APPL_DB) | Async events (SAI) | Table monitoring |

### Why SONiC Uses Each

| Use Case | Consumer Class | Rationale |
|----------|---------------|-----------|
| **APPL_DB** | ConsumerStateTable | High-volume, cannot lose updates, must survive restarts |
| **SAI Events** | NotificationConsumer | Data must be captured at event time, self-contained |
| **CONFIG_DB** | SubscriberStateTable | Lower volume, operator can re-apply, any client can write |

---

## Producer-Consumer Model

### Multiple Producers, Single Consumer

ConsumerStateTable supports **multiple producers** but requires **single consumer**:

```
portsyncd (Producer 1)                    Redis
        │                                   │
        ├─► HSET _PORT_TABLE:Eth0 speed 100000
        ├─► SADD PORT_TABLE_KEY_SET Eth0
        └─► PUBLISH channel "G"

portmgrd (Producer 2)
        │
        ├─► HSET _PORT_TABLE:Eth4 mtu 9100
        ├─► SADD PORT_TABLE_KEY_SET Eth4
        └─► PUBLISH channel "G"

                                     KEY_SET = {Eth0, Eth4}
```

**Multiple producers safe because:**
- `SADD` is atomic
- SET ensures no duplicate keys
- Same key: last writer wins

**Single consumer required because:**
- `SPOP` is destructive
- Lua script deletes staging hash after reading

### Guaranteed Delivery

```
Time 0ms: HSET _PORT_TABLE:Eth0 speed 10000, SADD KEY_SET
Time 1ms: HSET _PORT_TABLE:Eth0 speed 25000, SADD KEY_SET (no-op)
Time 2ms: HSET _PORT_TABLE:Eth0 speed 100000, SADD KEY_SET (no-op)

Time 5ms: Consumer pops → KEY_SET has ONE entry → gets 100000 (latest) ✅
```

---

## Code References

### Class Hierarchy

```
Selectable (swss-common)
    └── RedisSelect
            └── ConsumerTableBase
                    ├── ConsumerStateTable
                    └── SubscriberStateTable

Selectable (swss-common)
    └── NotificationConsumer

Executor (orch.h:106)
    └── ConsumerBase (orch.h:155)
            └── Consumer (orch.h:230)
```

### Key Files

| File | Purpose |
|------|---------|
| `orchagent/orchdaemon.cpp` | Main event loop |
| `orchagent/orch.cpp` | Orch base class, Consumer::execute() |
| `orchagent/portsorch.cpp` | PortsOrch::doPortTask() |
| `sonic-swss-common/common/select.cpp` | Select class |
| `sonic-swss-common/common/consumerstatetable.cpp` | ConsumerStateTable |
| `sonic-swss-common/common/subscriberstatetable.cpp` | SubscriberStateTable |
| `sonic-swss-common/common/notificationconsumer.cpp` | NotificationConsumer |
| `sonic-swss-common/common/consumer_state_table_pops.lua` | Atomic pop script |

### Execution Flow

```
OrchDaemon::start()
    │
    └─► Select::select()
            │
            ├─► epoll_wait()
            ├─► Selectable::readData()
            └─► returns Selectable*
                    │
                    ▼
            Consumer::execute()           // orch.cpp:503
                │
                ├─► pops()                // Fetch data
                ├─► addToSync()           // Queue in m_toSync
                └─► drain()
                        │
                        ▼
                    Orch::doTask(Consumer&)
                        │
                        ▼
                    PortsOrch::doTask()
                        │
                        └─► doPortTask() → SAI API calls
```

### Complete Data Flow (ConsumerStateTable)

```
Producer (portsyncd)                    Redis APPL_DB                     Consumer (orchagent)
        │                                     │                                    │
        │  HSET _PORT_TABLE:Eth0 speed 100000 │                                    │
        │─────────────────────────────────────►│ _PORT_TABLE:Eth0 (staging)        │
        │                                     │                                    │
        │  SADD PORT_TABLE_KEY_SET Eth0       │                                    │
        │─────────────────────────────────────►│ KEY_SET = {Eth0}                  │
        │                                     │                                    │
        │  PUBLISH channel "G"                │                                    │
        │─────────────────────────────────────►│                                    │
        │                                     │          readData() ◄──────────────│
        │                                     │          (count notification)      │
        │                                     │                                    │
        │                                     │          pops() ◄──────────────────│
        │                                     │◄── EVALSHA lua_script ─────────────│
        │                                     │    ├─ SPOP KEY_SET → [Eth0]        │
        │                                     │    ├─ HGETALL _PORT_TABLE:Eth0     │
        │                                     │    ├─ HSET PORT_TABLE:Eth0 ...     │
        │                                     │    └─ DEL _PORT_TABLE:Eth0         │
        │                                     │                                    │
        │                                     │ PORT_TABLE:Eth0 (real table)       │
        │                                     │   created by CONSUMER              │
        │                                     │                                    │
        │                                     │          doPortTask() ─────────────│
        │                                     │          SAI API calls             │
```
