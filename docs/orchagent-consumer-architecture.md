# Orchagent Consumer Architecture

This document explains how orchagent consumes data from Redis databases using different consumer mechanisms, and how the main event loop processes these events.

## Table of Contents

1. [Overview](#overview)
2. [Threading Model](#threading-model)
3. [Select Loop and Event Processing](#select-loop-and-event-processing)
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

## Select Loop and Event Processing

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

### Two-Phase Processing: readData() and pops()

All consumer classes use a two-phase approach to process events:

| Phase | Called By | Purpose |
|-------|-----------|---------|
| `readData()` | `Select` class (generic) | Drain socket, count/buffer notifications |
| `pops()` | `Consumer::execute()` (application) | Process data, return to application |

**Why two phases?**

1. **Architectural separation** - `Select` is generic infrastructure that only knows the `Selectable` interface. It doesn't know about `pops()`, Lua scripts, or JSON parsing.

2. **Fair scheduling** - `readData()` updates `m_queueLength` or buffer size, enabling `hasCachedData()` checks for fair scheduling across multiple tables.

3. **Batching** - Multiple notifications can arrive and be buffered by `readData()`, then processed in one `pops()` call.

4. **Non-blocking** - `readData()` is fast (just drains socket); heavy work is deferred to `pops()`.

```
Select::select()                          Consumer::execute()
      │                                          │
      ├─► epoll_wait()                           │
      ├─► readData()  ◄─── Phase 1: Notification │
      │     (drain socket, count/buffer)         │
      │                                          │
      └─► returns Selectable* ──────────────────►│
                                                 ├─► pops()  ◄─── Phase 2: Retrieval
                                                 │     (fetch/parse actual data)
                                                 ├─► addToSync()
                                                 └─► drain() → doTask()
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
        sel->readData();          // Phase 1: drain socket
        m_ready.insert(sel);
    }

    // 3. Return ONE Selectable at a time (fair scheduling)
    while (!m_ready.empty()) {
        auto sel = *m_ready.begin();
        m_ready.erase(sel);

        if (!sel->hasData())
            continue;

        *c = sel;

        if (sel->hasCachedData())    // More data pending?
            m_ready.insert(sel);     // Re-add to BACK of ready set

        sel->updateAfterRead();      // Decrement queue length
        return Select::OBJECT;
    }
    return Select::TIMEOUT;
}
```

### Fair Scheduling Mechanism

The `hasCachedData()` check prevents one busy table from monopolizing the event loop:

```
Initial state:
  PORT_TABLE:  m_queueLength = 100 notifications
  ROUTE_TABLE: m_queueLength = 2 notifications
  m_ready = {PORT_TABLE, ROUTE_TABLE}

Iteration 1:
  → Return PORT_TABLE
  → hasCachedData()? (99 > 1) YES → re-add to m_ready
  → m_ready = {ROUTE_TABLE, PORT_TABLE}  ← PORT moved to back!

Iteration 2:
  → Return ROUTE_TABLE (gets a turn)
  → m_ready = {PORT_TABLE, ROUTE_TABLE}

... tables get interleaved turns ...
```

**Important:** Fair scheduling is at the **Selectable level** (table level), not individual entry level. Each `pops()` call processes ALL available data up to `POP_BATCH_SIZE`.

### Batch Size Throttling

Each `pops()` call is limited by `POP_BATCH_SIZE` (default 128):

```cpp
// ConsumerStateTable pops()
command.format("EVALSHA %s ... %d ...", m_shaPop.c_str(), POP_BATCH_SIZE);
// Lua: SPOP KEY_SET 128  -- max 128 keys per call

// NotificationConsumer pops()
if (vkco.size() >= POP_BATCH_SIZE)
    return;  // Stop processing, yield to other consumers
```

This prevents a single table with massive data from blocking others indefinitely.

### Coalescing Behavior Difference

| Consumer Class | Notification Count vs Data Items | Fair Scheduling Accuracy |
|----------------|----------------------------------|-------------------------|
| **ConsumerStateTable** | 100 notifications may = 5 keys (coalesced in KEY_SET) | ⚠️ Inaccurate - may have empty iterations |
| **NotificationConsumer** | 100 notifications = 100 events (no coalescing) | ✅ Accurate - uses actual queue size |
| **SubscriberStateTable** | 100 notifications = 100 events (no coalescing) | ✅ Accurate - uses actual buffer size |

For `ConsumerStateTable`, the `m_queueLength` counts "G" signals, but `KEY_SET` coalesces duplicate keys. This can result in wasted scheduling iterations when notifications exceed actual unique keys.

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

Redis designed keyspace notifications as lightweight signals, not data carriers:

1. **Performance** - Copying values to all subscribers is expensive
2. **Buffer pressure** - Large values would cause more dropped notifications
3. **Not all consumers need values** - Some just need "something changed" signal
4. **Data type complexity** - No universal value format for STRING/HASH/LIST/SET/ZSET
5. **Design philosophy** - Keyspace notifications = "WHAT changed", not "WHAT + VALUE"

This Redis limitation is why `SubscriberStateTable` has reliability issues and why SONiC uses `ConsumerStateTable` (staging + KEY_SET) for critical paths.

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

Used for **APPL_DB** - high-reliability table synchronization with coalescing.

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

#### pops() - Lua Script

**File:** `sonic-swss-common/common/consumer_state_table_pops.lua`

```lua
local ret = {}
local tablename = KEYS[2]                           -- "PORT_TABLE:"
local stateprefix = ARGV[2]                         -- "_"

-- 1. Pop keys from the KEY_SET (atomically, up to batch size)
local keys = redis.call('SPOP', KEYS[1], ARGV[1])   -- ARGV[1] = POP_BATCH_SIZE

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

**Key insight:** The **consumer** writes to the actual DB table, not the producer. The producer only writes to the staging hash with `_` prefix.

#### Redis Data Structures

| Redis Key | Type | Written By | Read By | Purpose |
|-----------|------|------------|---------|---------|
| `_PORT_TABLE:Ethernet0` | HASH | Producer | Consumer | Staging area |
| `PORT_TABLE:Ethernet0` | HASH | Consumer | Anyone | Real table |
| `PORT_TABLE_KEY_SET` | SET | Producer | Consumer | Pending keys (coalesced) |
| `PORT_TABLE_DEL_SET` | SET | Producer | Consumer | Keys to delete |
| `PORT_TABLE_CHANNEL@0` | PUBSUB | Producer | Consumer | Notification |

---

### SubscriberStateTable

Used for **CONFIG_DB / STATE_DB** - simpler but less reliable, no coalescing.

#### readData() - Saves Notifications

The notification **contains the key name**, so it must be saved:

```cpp
uint64_t SubscriberStateTable::readData() {
    redisGetReply(..., &reply);
    m_keyspace_event_buffer.emplace_back(reply);  // SAVES the notification!

    // Drain more buffered notifications
    do {
        redisGetReplyFromReader(..., &reply);
        if (reply != nullptr) {
            m_keyspace_event_buffer.emplace_back(reply);
        }
    } while (reply != nullptr);
}

bool hasData()       { return !m_keyspace_event_buffer.empty(); }
bool hasCachedData() { return m_keyspace_event_buffer.size() > 1; }
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
        vkco.push_back(kco);
    }
}
```

**Problem:** By the time `pops()` runs, the table data may have changed (race condition).

---

### NotificationConsumer

Used for **SAI async events** from syncd - self-contained messages, no coalescing.

#### readData() - Buffers JSON Messages

**File:** `sonic-swss-common/common/notificationconsumer.cpp:68-104`

```cpp
uint64_t NotificationConsumer::readData()
{
    redisReply *reply = nullptr;
    redisGetReply(m_subscribe->getContext(), reinterpret_cast<void**>(&reply));

    RedisReply r(reply);
    processReply(reply);    // Parse and queue the message

    // Drain any additional buffered messages
    do {
        status = redisGetReplyFromReader(..., &reply);
        if (reply != nullptr && status == REDIS_OK) {
            RedisReply r(reply);
            processReply(reply);
        }
    } while (reply != nullptr && status == REDIS_OK);
    return 0;
}

void NotificationConsumer::processReply(redisReply *reply)
{
    std::string msg = reply->element[2]->str;  // JSON payload
    m_queue.push(msg);                          // Buffer for pops()
}

bool hasData()       { return m_queue.size() > 0; }
bool hasCachedData() { return m_queue.size() > 1; }
```

#### pops() - Parse JSON from Queue

```cpp
void NotificationConsumer::pops(std::deque<KeyOpFieldsValuesTuple> &vkco)
{
    while (!m_queue.empty()) {
        std::string msg = m_queue.front();
        m_queue.pop();

        // Parse JSON: [["op","key"],["field1","value1"],...]
        JSon::readJson(msg, values);
        op = fvField(values.at(0));
        data = fvValue(values.at(0));

        vkco.emplace_back(data, op, values);

        // Batch size throttling
        if (vkco.size() >= POP_BATCH_SIZE)
            return;
    }
}
```

#### Use Cases in Orchagent

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
                └─► Data is IN the message (safe from race conditions)
```

---

## Reliability Comparison

### Overview

| Scenario | ConsumerStateTable | NotificationConsumer | SubscriberStateTable |
|----------|-------------------|---------------------|---------------------|
| SET then quick DEL | ✅ Final state correct | ✅ Both events seen | ❌ **Data lost** |
| Multiple rapid updates | ✅ Gets latest (coalesced) | ⚠️ Each processed | ⚠️ Each processed |
| Consumer slow/blocked | ✅ Data persists | ❌ **Buffer overflow** | ❌ **Buffer overflow** |
| Consumer restart | ✅ KEY_SET persists | ❌ **Missed** | ❌ **Missed** |
| Producer crash mid-write | ✅ Atomic Lua | ✅ Atomic publish | ⚠️ Partial state |

### SET Followed by Quick DEL

#### SubscriberStateTable (fails - data loss)

```
Time 0ms: HSET PORT|Eth0 speed 100000 → notification "hset"
Time 1ms: DEL PORT|Eth0 → notification "del"
Time 5ms: Consumer pops "hset" → HGETALL PORT|Eth0 → EMPTY! ❌
          Consumer thinks it's a DEL, loses the SET data
```

#### ConsumerStateTable (correct final state)

```
Time 0ms: SET Eth0 speed=100000
  - HSET _PORT_TABLE:Eth0 speed 100000  (staging)
  - SADD KEY_SET Eth0
  - PUBLISH "G"

Time 1ms: DEL Eth0
  - DEL _PORT_TABLE:Eth0                 (delete staging)
  - SADD DEL_SET Eth0
  - SADD KEY_SET Eth0                    (no-op, already in set)
  - PUBLISH "G"

Time 5ms: Consumer pops() via Lua:
  - SPOP KEY_SET → {Eth0}
  - SREM DEL_SET Eth0 → 1               (was in DEL_SET)
  - DEL PORT_TABLE:Eth0                  (delete real table)
  - HGETALL _PORT_TABLE:Eth0 → {}       (empty, staging deleted)
  - Returns: {key=Eth0, fields={}}      → correctly interpreted as DEL ✅
```

**Note:** The SET event is "lost" in the sense that it's not processed separately, but the **final state is correct**. This is by design - ConsumerStateTable is for state synchronization, not event tracking.

#### Edge Case: SET→DEL→SET

```
Time 0ms: SET Eth0 speed=100000
Time 1ms: DEL Eth0
Time 2ms: SET Eth0 speed=200000

State after all operations:
  - _PORT_TABLE:Eth0 = {speed: 200000}  (staging has latest SET)
  - KEY_SET = {Eth0}
  - DEL_SET = {Eth0}                     (from the DEL)

Consumer pops():
  - SREM DEL_SET → 1 (was marked for del)
  - DEL PORT_TABLE:Eth0                  (delete real table first)
  - HGETALL _PORT_TABLE:Eth0 → {speed: 200000}
  - HSET PORT_TABLE:Eth0 speed 200000   (recreate with latest)
  - Returns: SET with speed=200000       ✅ Correct final state!
```

#### NotificationConsumer (sees both events)

```
Time 0ms: PUBLISH '{"op":"SET", "key":"Eth0", "speed":"100000"}'
Time 1ms: PUBLISH '{"op":"DEL", "key":"Eth0"}'

Time 5ms: Consumer pops():
  - Message 1: SET Eth0 speed=100000    ✅ Sees SET
  - Message 2: DEL Eth0                 ✅ Sees DEL
```

### When to Use Each

| Need | Consumer Class | Rationale |
|------|---------------|-----------|
| Final state sync | ConsumerStateTable | Coalescing is efficient, correct final state |
| Event tracking | NotificationConsumer | Every event preserved |
| Simple monitoring | SubscriberStateTable | Acceptable for low-volume, non-critical |

### Full Comparison

| Aspect | ConsumerStateTable | NotificationConsumer | SubscriberStateTable |
|--------|-------------------|---------------------|---------------------|
| **Reliability** | ✅ High | ⚠️ Medium | ⚠️ Lower |
| **Data persistence** | ✅ Staging + KEY_SET | ❌ Transient | ❌ Transient |
| **Crash recovery** | ✅ Full recovery | ❌ Missed | ❌ Missed |
| **SET→DEL race** | ✅ Final state correct | ✅ Both events seen | ❌ Data loss |
| **Coalescing** | ✅ Yes (efficient) | ❌ No (every event) | ❌ No (every event) |
| **Atomicity** | ✅ Lua script | ✅ Single publish | ❌ None |
| **Buffer overflow** | ✅ KEY_SET unlimited | ❌ Buffer limited | ❌ Buffer limited |
| **Producer coupling** | ❌ ProducerStateTable | ❌ NotificationProducer | ✅ Any client |
| **Best use case** | Table sync (APPL_DB) | Async events (SAI) | Table monitoring |

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

### Guaranteed Delivery (Coalesced)

```
Time 0ms: HSET _PORT_TABLE:Eth0 speed 10000, SADD KEY_SET
Time 1ms: HSET _PORT_TABLE:Eth0 speed 25000, SADD KEY_SET (no-op)
Time 2ms: HSET _PORT_TABLE:Eth0 speed 100000, SADD KEY_SET (no-op)

Time 5ms: Consumer pops → KEY_SET has ONE entry → gets 100000 (latest) ✅
```

100 writes to the same key = 1 entry in KEY_SET = 1 `pops()` result with latest value.

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
    └─► while(true)
            │
            ▼
        Select::select()
            │
            ├─► epoll_wait()                    ← Block for events
            ├─► Selectable::readData()          ← Phase 1: drain socket
            ├─► hasCachedData() → re-add        ← Fair scheduling
            └─► returns ONE Selectable*
                    │
                    ▼
            Consumer::execute()
                │
                ├─► pops()                      ← Phase 2: fetch data (up to batch size)
                ├─► addToSync()
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
