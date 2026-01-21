# SONiC Redis Consumer Architecture

This document explains how SONiC components consume data from Redis databases, covering the consumer mechanisms in sonic-swss-common and how orchagent uses them in its event loop.

## Table of Contents

1. [Overview](#overview)
2. [Threading Model](#threading-model)
3. [Event Loop and Processing](#event-loop-and-processing)
4. [Redis Notification Mechanisms](#redis-notification-mechanisms)
5. [Consumer Classes](#consumer-classes)
6. [Comparison and Trade-offs](#comparison-and-trade-offs)
7. [Code References](#code-references)

---

## Overview

Orchagent is the core component in SONiC that translates high-level configuration from Redis databases into SAI (Switch Abstraction Interface) API calls. It uses a **single-threaded event loop** to process changes from multiple Redis tables.

### Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `Select` | swss-common | Event multiplexer using epoll |
| `Selectable` | swss-common | Base interface for event sources |
| `ConsumerTable` | swss-common | Consumes via LIST queue (ordered, no coalescing) |
| `ConsumerStateTable` | swss-common | Consumes via KEY_SET + staging (coalescing) |
| `SubscriberStateTable` | swss-common | Consumes via Redis keyspace notifications |
| `NotificationConsumer` | swss-common | Consumes JSON messages via pub/sub |
| `Consumer` | orchagent | Wrapper connecting a table to an Orch |
| `Orch` | orchagent | Base class for orchestrators (PortsOrch, RouteOrch, etc.) |

---

## Threading Model

Orchagent uses a **single-threaded event loop** where all Orchs share the main thread.

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
```

### Implications

- **No parallelism** between Orchs - they're cooperative
- **Blocking** - if `doPortTask()` takes long, all other Orchs wait
- **SAI calls** - all SAI API calls happen on main thread (serialized)
- **Simplicity** - no locks needed between Orchs

---

## Event Loop and Processing

### Main Loop

**File:** `orchagent/orchdaemon.cpp`

```cpp
void OrchDaemon::start()
{
    for (Orch *o : m_orchList) {
        m_select->addSelectables(o->getSelectables());
    }

    while (true) {
        Selectable *s;
        int ret = m_select->select(&s, SELECT_TIMEOUT);

        if (ret == Select::OBJECT) {
            auto *c = (Executor *)s;
            c->execute();
        }

        for (Orch *o : m_orchList)
            o->doTask();
    }
}
```

### Two-Phase Processing

All consumer classes use a two-phase approach:

| Phase | Called By | Purpose |
|-------|-----------|---------|
| `readData()` | `Select` (generic infrastructure) | Drain socket, count/buffer notifications |
| `pops()` | `Consumer::execute()` (application) | Fetch/parse actual data, return to caller |

**Why two phases?**

1. **Architectural separation** - `Select` only knows the `Selectable` interface; doesn't know about Lua scripts or JSON parsing
2. **Fair scheduling** - `readData()` updates queue length, enabling `hasCachedData()` for fair scheduling
3. **Batching** - Multiple notifications buffered by `readData()`, processed in one `pops()` call
4. **Non-blocking** - `readData()` is fast (drains socket); heavy work deferred to `pops()`

```
Select::select()                          Consumer::execute()
      │                                          │
      ├─► epoll_wait()                           │
      ├─► readData()  ◄─── Phase 1               │
      │     (drain socket, count/buffer)         │
      │                                          │
      └─► returns Selectable* ──────────────────►│
                                                 ├─► pops()  ◄─── Phase 2
                                                 │     (fetch/parse data)
                                                 ├─► addToSync()
                                                 └─► drain() → doTask()
```

### Fair Scheduling

**File:** `sonic-swss-common/common/select.cpp`

```cpp
int Select::poll_descriptors(Selectable **c, ...)
{
    ret = ::epoll_wait(m_epoll_fd, events.data(), sz, timeout);

    for (int i = 0; i < ret; ++i) {
        Selectable* sel = m_objects[fd];
        sel->readData();
        m_ready.insert(sel);
    }

    while (!m_ready.empty()) {
        auto sel = *m_ready.begin();
        m_ready.erase(sel);

        if (!sel->hasData())
            continue;

        *c = sel;

        if (sel->hasCachedData())    // More data pending?
            m_ready.insert(sel);     // Re-add to back of ready set

        sel->updateAfterRead();
        return Select::OBJECT;
    }
}
```

The `hasCachedData()` check prevents one busy table from monopolizing the loop:

```
Initial state:
  PORT_TABLE:  m_queueLength = 100
  ROUTE_TABLE: m_queueLength = 2
  m_ready = {PORT_TABLE, ROUTE_TABLE}

Iteration 1:
  → Return PORT_TABLE
  → hasCachedData()? YES → re-add to m_ready
  → m_ready = {ROUTE_TABLE, PORT_TABLE}  ← PORT moved to back

Iteration 2:
  → Return ROUTE_TABLE (gets a turn)
  ...tables get interleaved turns...
```

### Batch Size Throttling

Each `pops()` is limited by `POP_BATCH_SIZE` (default 128):

```cpp
// ConsumerStateTable: Lua script limits SPOP count
SPOP KEY_SET 128

// NotificationConsumer: stops when batch full
if (vkco.size() >= POP_BATCH_SIZE)
    return;
```

This prevents a single table with massive data from blocking others indefinitely.

---

## Redis Notification Mechanisms

All consumer mechanisms are based on **Redis Pub/Sub**:

```
                        Redis Pub/Sub (foundation)
                               │
        ┌──────────────────────┼──────────────────────┐
        │                      │                      │
        ▼                      ▼                      ▼
  Custom Channel         Custom Channel         Keyspace Notifications
  Payload: "G"           Payload: JSON          Payload: key + event type
  (signal only)          (full data)            (no values)
        │                      │                      │
        ▼                      ▼                      ▼
  ConsumerTable         NotificationConsumer    SubscriberStateTable
  ConsumerStateTable      (self-contained)       (must fetch data)
  (data in Redis)
```

### 1. Pub/Sub with Signal + Queue (ConsumerTable / ConsumerStateTable)

Producer publishes a simple signal; data stored separately in Redis.

```bash
# Producer
PUBLISH PORT_TABLE_CHANNEL@0 "G"

# Consumer receives:
1) "message"
2) "PORT_TABLE_CHANNEL@0"
3) "G"                        # Just a signal
```

- **ConsumerTable** stores data in a LIST queue (ordered)
- **ConsumerStateTable** stores data in KEY_SET + staging hash (coalesced)

### 2. Pub/Sub with JSON Payload (NotificationConsumer)

Full data embedded in message - self-contained.

```bash
# Producer (syncd)
PUBLISH NOTIFICATIONS '[["port_state_change","oid:0x1000"],["state","up"]]'

# Consumer receives:
1) "message"
2) "NOTIFICATIONS"
3) '[["port_state_change","oid:0x1000"],["state","up"]]'
```

### 3. Keyspace Notifications (SubscriberStateTable)

Redis automatically notifies when keys change.

```bash
# Any client writes
HSET "PORT|Ethernet0" speed 100000

# Consumer receives (automatic):
1) "pmessage"
2) "__keyspace@4__:PORT|*"
3) "__keyspace@4__:PORT|Ethernet0"
4) "hset"                     # Operation type only - NO VALUES
```

**Why keyspace notifications don't include values:**
- Performance - copying values to all subscribers is expensive
- Buffer pressure - large values cause more dropped notifications
- Design philosophy - "what changed" not "what + value"

---

## Consumer Classes

### ConsumerTable

Used for **ordered queue processing** - preserves operation order, no coalescing.

Paired with **ProducerTable**.

#### How It Works

Uses a Redis **LIST** as a FIFO queue. Each entry contains (key, value, op) as three list elements.

```
Producer                                 Redis LIST
   │
   ├─► RPUSH TABLE_KEY_VALUE_OP_QUEUE key1
   ├─► RPUSH TABLE_KEY_VALUE_OP_QUEUE value1_json
   ├─► RPUSH TABLE_KEY_VALUE_OP_QUEUE "Sset"
   └─► PUBLISH channel "G"
                                    LIST = [key1, value1, op1, key2, value2, op2, ...]
```

#### pops() - Lua Script

**File:** `sonic-swss-common/common/consumer_table_pops.lua`

```lua
local popsize = ARGV[1] * 3           -- key, value, op per entry
local keys = redis.call('LRANGE', KEYS[1], -popsize, -1)
redis.call('LTRIM', KEYS[1], 0, -popsize-1)

for i = n, 1, -3 do
   local op = keys[i-2]
   local value = keys[i-1]              -- JSON encoded
   local key = keys[i]

   -- Decode JSON value
   local jj = cjson.decode(value)

   -- Write to real table (if op is set/create/remove)
   if op == 'set' or op == 'SET' then
       redis.call('HSET', tablename..key, ...)
   elseif op == 'DEL' then
       redis.call('DEL', tablename..key)
   end
end
```

#### Key Characteristics

| Aspect | ConsumerTable |
|--------|---------------|
| Queue type | Redis LIST |
| Ordering | ✅ FIFO preserved |
| Coalescing | ❌ Every operation processed |
| Bulk operations | ✅ Supports bulkset, bulkcreate, bulkremove |
| Query/Response | ✅ Supports get, getresponse, flush, etc. |

---

### ConsumerStateTable

Used for **APPL_DB** - high-reliability table synchronization with coalescing.

Paired with **ProducerStateTable**.

#### How It Works

Uses **KEY_SET** + **DEL_SET** + **staging hashes**. Multiple updates to same key are coalesced.

#### Why This Design? (KEY_SET + DEL_SET + Staging Hashes)

**Problem 1: Staging hashes are scattered (one per key)**

```
_PORT_TABLE:Ethernet0   ← one hash
_PORT_TABLE:Ethernet4   ← another hash
_PORT_TABLE:Ethernet8   ← another hash
```

Without KEY_SET, consumer would need `SCAN 0 MATCH _PORT_TABLE:*` to discover pending keys - expensive and non-atomic.

**Solution:** KEY_SET acts as an **index/manifest** of all pending staging hashes.

```
KEY_SET = {Ethernet0, Ethernet4, Ethernet8}
SPOP KEY_SET 128  ← O(1) per key, atomic
```

**Problem 2: DEL operation removes the staging hash**

```
SET Ethernet0:
  HSET _PORT_TABLE:Ethernet0 speed 100000   ← staging hash created
  SADD KEY_SET Ethernet0

DEL Ethernet0:
  DEL _PORT_TABLE:Ethernet0                  ← staging hash GONE
  SADD KEY_SET Ethernet0                     ← still need to process this key
  -- But how does consumer know it's a deletion?
```

**Solution:** DEL_SET marks keys that should be deleted.

```
DEL Ethernet0:
  DEL _PORT_TABLE:Ethernet0
  SADD DEL_SET Ethernet0                     ← marker for deletion
  SADD KEY_SET Ethernet0
```

Consumer logic:
```lua
key = SPOP KEY_SET                -- "Ethernet0"
if SREM DEL_SET key == 1 then     -- was in DEL_SET?
    DEL real_table:key            -- yes, delete real table
end
fv = HGETALL staging:key          -- {} empty (staging was deleted)
```

**Summary:**

| Structure | Purpose |
|-----------|---------|
| KEY_SET | Index of pending keys (discovery - avoids expensive SCAN) |
| DEL_SET | Marker for deletions (staging hash is already gone) |
| Staging hashes | Actual field-value data (one hash per key, prefix `_`) |

#### readData()

**File:** `sonic-swss-common/common/redisselect.cpp`

```cpp
uint64_t RedisSelect::readData()
{
    redisReply *reply = nullptr;
    redisGetReply(m_subscribe->getContext(), (void**)&reply);

    freeReplyObject(reply);     // Discard - content is just "G"
    m_queueLength++;            // Just count notifications

    // Drain additional buffered notifications
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

-- 1. Pop keys from KEY_SET (up to batch size)
local keys = redis.call('SPOP', KEYS[1], ARGV[1])

for i = 1, n do
   local key = keys[i]

   -- 2. Check if marked for deletion
   local num = redis.call('SREM', KEYS[3], key)
   if num == 1 then
      redis.call('DEL', tablename..key)
   end

   -- 3. Get from STAGING hash
   local fieldvalues = redis.call('HGETALL', stateprefix..tablename..key)
   table.insert(ret, {key, fieldvalues})

   -- 4. Copy to REAL table (consumer writes!)
   for i = 1, #fieldvalues, 2 do
      redis.call('HSET', tablename..key, fieldvalues[i], fieldvalues[i + 1])
   end

   -- 5. Clean up staging
   redis.call('DEL', stateprefix..tablename..key)
end
return ret
```

#### Redis Data Structures

| Redis Key | Type | Written By | Purpose |
|-----------|------|------------|---------|
| `_PORT_TABLE:Ethernet0` | HASH | Producer | Staging area |
| `PORT_TABLE:Ethernet0` | HASH | Consumer | Real table |
| `PORT_TABLE_KEY_SET` | SET | Producer | Pending keys (coalesced) |
| `PORT_TABLE_DEL_SET` | SET | Producer | Keys to delete |
| `PORT_TABLE_CHANNEL@0` | PUBSUB | Producer | Notification |

#### Producer-Consumer Model

Supports **multiple producers** but requires **single consumer**:

```
portsyncd (Producer 1)                    Redis
        │
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

**Multiple producers safe:** `SADD` is atomic, SET ensures no duplicates.

**Single consumer required:** `SPOP` is destructive, Lua deletes staging after reading.

---

### SubscriberStateTable

Used for **CONFIG_DB / STATE_DB** - simpler but less reliable.

No paired producer - works with **any Redis client**.

#### readData()

The notification **contains the key name**, so it must be saved:

```cpp
uint64_t SubscriberStateTable::readData() {
    redisGetReply(..., &reply);
    m_keyspace_event_buffer.emplace_back(reply);  // SAVES notification

    do {
        redisGetReplyFromReader(..., &reply);
        if (reply != nullptr)
            m_keyspace_event_buffer.emplace_back(reply);
    } while (reply != nullptr);
}

bool hasData()       { return !m_keyspace_event_buffer.empty(); }
bool hasCachedData() { return m_keyspace_event_buffer.size() > 1; }
```

#### pops()

```cpp
void SubscriberStateTable::pops(...) {
    while (auto event = popEventBuffer()) {
        string key = parseKeyFromChannel(message.channel);

        if (message.data == "del") {
            kfvOp(kco) = DEL_COMMAND;
        } else {
            m_table.get(key, kfvFieldsValues(kco));  // FETCH from table
            kfvOp(kco) = SET_COMMAND;
        }
        vkco.push_back(kco);
    }
}
```

**Problem:** By the time `pops()` runs, table data may have changed (race condition).

---

### NotificationConsumer

Used for **SAI async events** from syncd - self-contained messages.

Paired with **NotificationProducer**.

#### readData()

**File:** `sonic-swss-common/common/notificationconsumer.cpp`

```cpp
uint64_t NotificationConsumer::readData()
{
    redisReply *reply = nullptr;
    redisGetReply(m_subscribe->getContext(), (void**)&reply);

    processReply(reply);    // Queue the message

    do {
        status = redisGetReplyFromReader(..., &reply);
        if (reply != nullptr && status == REDIS_OK)
            processReply(reply);
    } while (reply != nullptr && status == REDIS_OK);
    return 0;
}

void processReply(redisReply *reply) {
    std::string msg = reply->element[2]->str;  // JSON payload
    m_queue.push(msg);
}

bool hasData()       { return m_queue.size() > 0; }
bool hasCachedData() { return m_queue.size() > 1; }
```

#### pops()

```cpp
void NotificationConsumer::pops(std::deque<KeyOpFieldsValuesTuple> &vkco)
{
    while (!m_queue.empty()) {
        std::string msg = m_queue.front();
        m_queue.pop();

        JSon::readJson(msg, values);
        op = fvField(values.at(0));
        data = fvValue(values.at(0));

        vkco.emplace_back(data, op, values);

        if (vkco.size() >= POP_BATCH_SIZE)
            return;
    }
}
```

#### Use Cases

| Orch | Channel | Purpose |
|------|---------|---------|
| `PortsOrch` | NOTIFICATIONS | Port oper status from SAI |
| `FdbOrch` | NOTIFICATIONS | FDB learn/age events |
| `BfdOrch` | NOTIFICATIONS | BFD session state changes |
| `PfcWdOrch` | NOTIFICATIONS | PFC watchdog events |

SAI events are **asynchronous hardware events** - the data exists only at callback time, so it must be serialized into the message.

---

## Comparison and Trade-offs

### Full Comparison

| Aspect | ConsumerTable | ConsumerStateTable | NotificationConsumer | SubscriberStateTable |
|--------|---------------|-------------------|---------------------|---------------------|
| **Queue type** | LIST | SET + staging | In-memory queue | Event buffer |
| **Ordering** | ✅ FIFO | ❌ Unordered | ✅ FIFO | ✅ FIFO |
| **Coalescing** | ❌ No | ✅ Yes | ❌ No | ❌ No |
| **Reliability** | ✅ High | ✅ High | ⚠️ Medium | ⚠️ Lower |
| **Data persistence** | ✅ LIST persists | ✅ KEY_SET persists | ❌ Transient | ❌ Transient |
| **Crash recovery** | ✅ Full | ✅ Full | ❌ Missed | ❌ Missed |
| **SET→DEL race** | ✅ Both seen | ✅ Final state | ✅ Both seen | ❌ Data loss |
| **Bulk operations** | ✅ Yes | ❌ No | ❌ No | ❌ No |
| **Producer coupling** | ProducerTable | ProducerStateTable | NotificationProducer | Any client |
| **Best use case** | Ordered ops | State sync | Async events | Monitoring |

### ConsumerTable vs ConsumerStateTable

| Aspect | ConsumerTable | ConsumerStateTable |
|--------|---------------|-------------------|
| **Use when** | Order matters, every op needed | Final state matters, coalescing OK |
| **Example** | ASIC_STATE ops to syncd | PORT_TABLE config to orchagent |
| **100 writes same key** | 100 entries processed | 1 entry (latest value) |
| **Ordering** | ✅ Guaranteed FIFO | ❌ SET is unordered |

### Coalescing Behavior

| Consumer Class | Notification Count vs Data Items | Fair Scheduling Accuracy |
|----------------|----------------------------------|-------------------------|
| **ConsumerTable** | 100 notifications = 100 ops | ✅ Accurate |
| **ConsumerStateTable** | 100 notifications may = 5 keys (coalesced) | ⚠️ May have empty iterations |
| **NotificationConsumer** | 100 notifications = 100 events | ✅ Accurate |
| **SubscriberStateTable** | 100 notifications = 100 events | ✅ Accurate |

### SET Followed by Quick DEL

#### SubscriberStateTable (data loss)

```
Time 0ms: HSET PORT|Eth0 speed 100000 → notification "hset"
Time 1ms: DEL PORT|Eth0 → notification "del"
Time 5ms: Consumer pops "hset" → HGETALL PORT|Eth0 → EMPTY! ❌
```

#### ConsumerStateTable (correct final state)

```
Time 0ms: SET Eth0 speed=100000
  - HSET _PORT_TABLE:Eth0 speed 100000  (staging)
  - SADD KEY_SET Eth0
  - PUBLISH "G"

Time 1ms: DEL Eth0
  - DEL _PORT_TABLE:Eth0
  - SADD DEL_SET Eth0
  - PUBLISH "G"

Time 5ms: Consumer pops() via Lua:
  - SPOP KEY_SET → {Eth0}
  - SREM DEL_SET Eth0 → 1 (was in DEL_SET)
  - DEL PORT_TABLE:Eth0
  - HGETALL _PORT_TABLE:Eth0 → {}
  - Returns: {key=Eth0, fields={}} → DEL ✅
```

**Note:** The SET event is "lost" (not processed separately), but the **final state is correct**. ConsumerStateTable is for state synchronization, not event tracking.

#### ConsumerTable (both events seen)

```
Time 0ms: SET Eth0 → RPUSH to LIST
Time 1ms: DEL Eth0 → RPUSH to LIST

Time 5ms: Consumer pops() via Lua:
  - LRANGE gets both entries in order
  - Entry 1: SET Eth0  ✅
  - Entry 2: DEL Eth0  ✅
```

### When to Use Each

| Need | Consumer Class | Rationale |
|------|---------------|-----------|
| Ordered operations | ConsumerTable | FIFO preserved, every op processed |
| Final state sync | ConsumerStateTable | Coalescing efficient, correct final state |
| Async event tracking | NotificationConsumer | Self-contained, every event preserved |
| Simple monitoring | SubscriberStateTable | OK for low-volume, non-critical |

---

## Code References

### Class Hierarchy

```
Selectable (swss-common)
    └── RedisSelect
            └── ConsumerTableBase
                    ├── ConsumerTable
                    ├── ConsumerStateTable
                    └── SubscriberStateTable

Selectable (swss-common)
    └── NotificationConsumer

Executor (orchagent)
    └── ConsumerBase
            └── Consumer
```

### Key Files

| File | Purpose |
|------|---------|
| `orchagent/orchdaemon.cpp` | Main event loop |
| `orchagent/orch.cpp` | Orch base class, Consumer::execute() |
| `sonic-swss-common/common/select.cpp` | Select class, fair scheduling |
| `sonic-swss-common/common/consumertable.cpp` | ConsumerTable |
| `sonic-swss-common/common/consumerstatetable.cpp` | ConsumerStateTable |
| `sonic-swss-common/common/subscriberstatetable.cpp` | SubscriberStateTable |
| `sonic-swss-common/common/notificationconsumer.cpp` | NotificationConsumer |
| `sonic-swss-common/common/consumer_table_pops.lua` | ConsumerTable Lua script |
| `sonic-swss-common/common/consumer_state_table_pops.lua` | ConsumerStateTable Lua script |
