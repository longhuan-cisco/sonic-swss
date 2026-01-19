# Orchagent Consumer Architecture

This document explains how orchagent consumes data from Redis databases using ConsumerStateTable and SubscriberStateTable, and how the main event loop processes these events.

## Table of Contents

1. [Overview](#overview)
2. [Threading Model](#threading-model)
3. [Select Loop Architecture](#select-loop-architecture)
4. [ConsumerStateTable vs SubscriberStateTable](#consumerstatetable-vs-subscriberstatetable)
5. [ConsumerStateTable Deep Dive](#consumerstatetable-deep-dive)
6. [SubscriberStateTable Deep Dive](#subscriberstatetable-deep-dive)
7. [Redis Notification Mechanisms](#redis-notification-mechanisms)
8. [Reliability Comparison](#reliability-comparison)
9. [Redis Data Structures](#redis-data-structures)
10. [Producer-Consumer Model](#producer-consumer-model)
11. [Code References](#code-references)

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

## ConsumerStateTable vs SubscriberStateTable

| Aspect | ConsumerStateTable | SubscriberStateTable |
|--------|-------------------|---------------------|
| **Used for** | APPL_DB | CONFIG_DB, STATE_DB |
| **Subscription** | Custom channel: `TABLE_CHANNEL@0` | Redis keyspace: `__keyspace@N__:TABLE:*` |
| **Notification content** | Just `"G"` (meaningless signal) | Contains key name + operation |
| **Data staging** | Temporary hash `_TABLE:key` | Direct table `TABLE\|key` |
| **Atomicity** | Lua script for atomic pop | No atomicity guarantees |
| **Producer** | Must use `ProducerStateTable` | Any Redis client (CLI, etc.) |

### When Each Is Used

**File:** `orchagent/orch.cpp:1090-1100`

```cpp
void Orch::addConsumer(DBConnector *db, string tableName, int pri)
{
    if (db->getDbId() == CONFIG_DB || db->getDbId() == STATE_DB) {
        // Uses keyspace notifications
        addExecutor(new Consumer(new SubscriberStateTable(...)));
    } else {
        // APPL_DB uses producer-consumer queue
        addExecutor(new Consumer(new ConsumerStateTable(...)));
    }
}
```

---

## ConsumerStateTable Deep Dive

### Two-Phase Data Retrieval

ConsumerStateTable uses a two-phase approach:

1. **`readData()`** - Read pub/sub notification (just counts, discards content)
2. **`pops()`** - Fetch actual data via Lua script

### Why Two Phases?

| Phase | Method | Purpose |
|-------|--------|---------|
| 1 | `readData()` | **Notification** - "something changed" (lightweight) |
| 2 | `pops()` | **Retrieval** - get actual data (heavier) |

**Reasons for separation:**
- `Select` class is generic - it handles any `Selectable`, only knows to call `readData()`
- Batching - multiple notifications can arrive, `pops()` fetches all at once
- Notification content is just `"G"` - no useful data to save
- Fair scheduling - `m_queueLength` enables round-robin across tables

### readData() Implementation

**File:** `sonic-swss-common/common/redisselect.cpp:25-53`

```cpp
uint64_t RedisSelect::readData()
{
    redisReply *reply = nullptr;

    // Read one notification from the Redis SUBSCRIBE socket
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

### pops() Implementation

**File:** `sonic-swss-common/common/consumerstatetable.cpp:36-94`

```cpp
void ConsumerStateTable::pops(deque<KeyOpFieldsValuesTuple> &vkco, ...)
{
    RedisCommand command;
    command.format(
        "EVALSHA %s 3 %s %s%s %s %d %s",
        m_shaPop.c_str(),           // Lua script SHA
        getKeySetName().c_str(),    // KEYS[1]: PORT_TABLE_KEY_SET
        getTableName().c_str(),     // KEYS[2]: PORT_TABLE:
        getTableNameSeparator().c_str(),
        getDelKeySetName().c_str(), // KEYS[3]: PORT_TABLE_DEL_SET
        POP_BATCH_SIZE,             // ARGV[1]: batch size
        getStateHashPrefix().c_str()); // ARGV[2]: "_" prefix

    RedisReply r(m_db, command);
    // ... parse results
}
```

### The Lua Script

**File:** `sonic-swss-common/common/consumer_state_table_pops.lua`

```lua
local ret = {}
local tablename = KEYS[2]                           -- "PORT_TABLE:"
local stateprefix = ARGV[2]                         -- "_"

-- 1. Pop keys from the KEY_SET (atomically)
local keys = redis.call('SPOP', KEYS[1], ARGV[1])   -- SPOP PORT_TABLE_KEY_SET <batch>

for i = 1, n do
   local key = keys[i]

   -- 2. Check if key was marked for deletion
   local num = redis.call('SREM', KEYS[3], key)     -- SREM PORT_TABLE_DEL_SET key
   if num == 1 then
      redis.call('DEL', tablename..key)             -- DEL PORT_TABLE:Ethernet0
   end

   -- 3. Get field-values from temporary STAGING hash
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

**Key insight:** The **consumer** writes to the actual DB table, not the producer. Producer only writes to the staging hash with `_` prefix.

---

## SubscriberStateTable Deep Dive

### Why It Has Its Own readData()

SubscriberStateTable overrides `readData()` because the notification **contains the key name**:

```
Channel: "__keyspace@4__:PORT_TABLE|Ethernet0"
Data:    "hset" or "del"
```

### readData() - Saves Notifications

```cpp
uint64_t SubscriberStateTable::readData() {
    redisGetReply(..., &reply);
    m_keyspace_event_buffer.emplace_back(reply);  // SAVES the notification!

    // Drain more buffered notifications
    do {
        redisGetReplyFromReader(..., &reply);
        if (reply != nullptr) {
            m_keyspace_event_buffer.emplace_back(reply);  // SAVES each one
        }
    } while (reply != nullptr);
}
```

### pops() - Parses Saved Notifications

```cpp
void SubscriberStateTable::pops(...) {
    while (auto event = popEventBuffer()) {           // Uses saved notifications
        auto message = event->getReply<RedisMessage>();

        // Parse key from channel: "__keyspace@4__:PORT_TABLE|Ethernet0"
        string key = parseKeyFromChannel(message.channel);

        if (message.data == "del") {
            kfvOp(kco) = DEL_COMMAND;
        } else {
            m_table.get(key, kfvFieldsValues(kco));   // Fetch from actual table
            kfvOp(kco) = SET_COMMAND;
        }
        vkco.push_back(kco);
    }
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

**Key insight:** SubscriberStateTable's reliability limitations stem from Redis keyspace notifications only providing key + event type in the payload, not actual field values. If Redis keyspace notifications included the data values, SubscriberStateTable would be as reliable as NotificationConsumer.

### Redis Pub/Sub with Signal (used by ConsumerStateTable)

**Explicit messaging** - producer must explicitly publish.

```
Producer:                              Consumer:
PUBLISH PORT_TABLE_CHANNEL@0 "G"  ──►  SUBSCRIBE PORT_TABLE_CHANNEL@0
```

#### Characteristics

| Aspect | Description |
|--------|-------------|
| **Trigger** | Explicit `PUBLISH` command by producer |
| **Content** | Producer controls message content (SONiC uses `"G"`) |
| **Channel** | Custom channel name: `TABLE_CHANNEL@N` |
| **Coupling** | Producer must know to publish |
| **Used by** | `ProducerStateTable` → `ConsumerStateTable` |

#### Example

```bash
# Producer
redis-cli PUBLISH PORT_TABLE_CHANNEL@0 "G"

# Consumer receives:
1) "message"
2) "PORT_TABLE_CHANNEL@0"
3) "G"                        # Just a signal, no useful data
```

### Redis Keyspace Notifications (used by SubscriberStateTable)

**Automatic notifications** - Redis automatically notifies when keys change.

```
Any client:                            Consumer:
HSET PORT|Eth0 speed 100000  ──►  Redis auto-generates notification
                                       │
                                       ▼
                              PSUBSCRIBE __keyspace@4__:PORT|*
```

#### Characteristics

| Aspect | Description |
|--------|-------------|
| **Trigger** | Automatic on any key modification |
| **Content** | Contains key name and operation type |
| **Channel** | Auto-generated: `__keyspace@N__:TABLE\|key` |
| **Coupling** | No producer awareness needed |
| **Used by** | Any writer → `SubscriberStateTable` |

#### Example

```bash
# Any client writes to Redis
redis-cli HSET "PORT|Ethernet0" speed 100000

# Consumer receives (automatic):
1) "pmessage"
2) "__keyspace@4__:PORT|*"              # Pattern subscribed
3) "__keyspace@4__:PORT|Ethernet0"      # Actual key that changed
4) "hset"                               # Operation type
```

#### Must Be Enabled in Redis

```bash
# Enable keyspace notifications (CONFIG_DB uses this)
redis-cli CONFIG SET notify-keyspace-events AKE
```

### Redis Pub/Sub with JSON Payload (used by NotificationConsumer)

**Self-contained messages** - full data embedded in JSON payload.

```
Producer (syncd):                              Consumer (orchagent):
PUBLISH NOTIFICATIONS '{"op":"port_state",     SUBSCRIBE NOTIFICATIONS
  "data":"oid:0x1000", "state":"up"}'    ──►   receives full JSON data
```

#### Characteristics

| Aspect | Description |
|--------|-------------|
| **Trigger** | Explicit `PUBLISH` command by producer |
| **Content** | Full JSON payload with all event data |
| **Channel** | Custom channel name (e.g., `NOTIFICATIONS`) |
| **Coupling** | Producer must use `NotificationProducer` |
| **Used by** | `syncd` → `orchagent` for SAI events |

#### Example

```bash
# Producer (syncd)
redis-cli PUBLISH NOTIFICATIONS '[["port_state_change","oid:0x1000"],["state","up"]]'

# Consumer receives:
1) "message"
2) "NOTIFICATIONS"
3) '[["port_state_change","oid:0x1000"],["state","up"]]'   # Full data in message!
```

#### Use Cases in Orchagent

| Orch | Channel | Purpose |
|------|---------|---------|
| `PortsOrch` | NOTIFICATIONS | Port oper status changes from SAI |
| `FdbOrch` | NOTIFICATIONS | FDB learn/age events from hardware |
| `BfdOrch` | NOTIFICATIONS | BFD session state changes |
| `PfcWdOrch` | NOTIFICATIONS | PFC watchdog events |
| `MACsecOrch` | NOTIFICATIONS | MACsec completion events |
| `SwitchOrch` | RESTARTCHECK | Warm restart check |

#### Why NotificationConsumer for SAI Events?

SAI events are **asynchronous hardware events** - the data exists only at the moment of the callback. The event data must be captured immediately.

```
Hardware event (link up)
        │
        ▼
    SAI callback in syncd
        │
        ├─► Serialize event to JSON (captures state NOW)
        └─► PUBLISH to NOTIFICATIONS channel
                │
                ▼
        orchagent (NotificationConsumer)
                │
                └─► Data is IN the message (safe from race conditions)
```

If SubscriberStateTable were used, by the time orchagent reads the table, the port state might have changed again.

### Side-by-Side Comparison

| Aspect | ConsumerStateTable | NotificationConsumer | SubscriberStateTable |
|--------|-------------------|---------------------|---------------------|
| **Subscription type** | Custom channel | Custom channel | Keyspace pattern |
| **Notification trigger** | Explicit `PUBLISH` | Explicit `PUBLISH` | Automatic on write |
| **Message content** | `"G"` (signal only) | Full JSON data | Key + event type |
| **Data source** | Staging hash + KEY_SET | Message itself | Must fetch from table |
| **SET→DEL race** | ✅ Safe | ✅ Safe | ❌ Data loss |
| **Producer coupling** | Must use ProducerStateTable | Must use NotificationProducer | Any client works |
| **Use case** | Table sync (APPL_DB) | Async events (SAI) | Table monitoring (CONFIG_DB) |

### Why readData() Differs Between Classes

| Class | readData() behavior | Why |
|-------|---------------------|-----|
| `ConsumerStateTable` | Discards notification | Content is just `"G"` - useless |
| `NotificationConsumer` | Saves and parses JSON | Full data is in message |
| `SubscriberStateTable` | Saves notification | Contains key name - needed for `pops()` |

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

### SubscriberStateTable Failure Scenarios

#### 1. SET Followed by Quick DEL (Data Loss)

```
Time 0ms: Writer: HSET PORT|Eth0 speed 100000
          → Redis generates: __keyspace notification (hset)

Time 1ms: Writer: DEL PORT|Eth0
          → Redis generates: __keyspace notification (del)

Time 5ms: Consumer: readData()
          → Receives both notifications, buffers them

Time 6ms: Consumer: pops() processes "hset" notification
          → HGETALL PORT|Eth0
          → Returns EMPTY! (already deleted)
          → Consumer thinks it's a DEL, not SET ❌
```

**Result:** Lost the SET operation entirely.

#### 2. Notification Buffer Overflow

```
Redis keyspace notification buffer is LIMITED (client-output-buffer-limit)

If consumer is slow:
  Notification 1: buffered
  Notification 2: buffered
  ...
  Notification 1000: buffered
  Notification 1001: DROPPED! ❌

Consumer never knows about dropped notifications.
```

#### 3. Consumer Restart (Missed Updates)

```
Time 0: Consumer running, subscribed to __keyspace@4__:*
Time 1: Writer: HSET PORT|Eth0 speed 100000 → notification sent
Time 2: Consumer crashes/restarts
Time 3: Writer: HSET PORT|Eth4 mtu 9100 → notification sent (consumer not listening)
Time 4: Consumer starts, re-subscribes
Time 5: Writer: HSET PORT|Eth8 admin up

Result: Consumer missed Eth0 and Eth4 changes ❌
```

#### 4. Rapid Updates (Inefficient Processing)

```
Time 0ms: Writer: HSET PORT|Eth0 speed 10000
          → Notification queued
Time 1ms: Writer: HSET PORT|Eth0 speed 25000
          → Notification queued
Time 2ms: Writer: HSET PORT|Eth0 speed 100000
          → Notification queued

Time 5ms: Consumer processes notification 1
          → HGETALL PORT|Eth0 → gets 100000 (latest, OK)
Time 6ms: Consumer processes notification 2
          → HGETALL PORT|Eth0 → gets 100000 (same, wasted work)
Time 7ms: Consumer processes notification 3
          → HGETALL PORT|Eth0 → gets 100000 (same, wasted work)
```

**Result:** 3 SAI calls for same final state (inefficient).

### ConsumerStateTable Advantages

#### 1. SET Followed by DEL (Safe)

```
Time 0ms: Producer: HSET _PORT_TABLE:Eth0 speed 100000
                    SADD PORT_TABLE_KEY_SET Eth0

Time 1ms: Producer: SADD PORT_TABLE_DEL_SET Eth0  (marks for deletion)
                    SADD PORT_TABLE_KEY_SET Eth0  (already in set, no-op)

Time 5ms: Consumer: pops() via Lua script
          → SPOP KEY_SET → {Eth0}
          → SREM DEL_SET Eth0 → returns 1 (was marked for del)
          → DEL PORT_TABLE:Eth0 (delete real table)
          → HGETALL _PORT_TABLE:Eth0 → returns {} (empty)
          → Correctly interprets as DEL operation ✅
```

#### 2. Consumer Restart (Recoverable)

```
Time 0: Producer writes Eth0 → KEY_SET = {Eth0}
Time 1: Producer writes Eth4 → KEY_SET = {Eth0, Eth4}
Time 2: Consumer crashes
Time 3: Producer writes Eth8 → KEY_SET = {Eth0, Eth4, Eth8}
Time 4: Consumer restarts
Time 5: Consumer pops() → Gets ALL pending keys ✅

KEY_SET and staging hashes PERSIST in Redis.
```

#### 3. Rapid Updates (Efficient)

```
Time 0ms: Producer: HSET _PORT_TABLE:Eth0 speed 10000
                    SADD KEY_SET Eth0
Time 1ms: Producer: HSET _PORT_TABLE:Eth0 speed 25000
                    SADD KEY_SET Eth0  (already in set)
Time 2ms: Producer: HSET _PORT_TABLE:Eth0 speed 100000
                    SADD KEY_SET Eth0  (already in set)

Time 5ms: Consumer: pops()
          → SPOP KEY_SET → {Eth0}  (only ONE entry)
          → HGETALL _PORT_TABLE:Eth0 → gets 100000 (latest)
          → ONE SAI call with final state ✅
```

### Full Pros/Cons Comparison

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
| **Storage overhead** | ⚠️ Staging + sets | ✅ None | ✅ None |
| **Latency** | ⚠️ Lua script | ✅ Direct | ✅ Direct |
| **Best use case** | Table sync | Async events | Table monitoring |

### Why SONiC Uses Each

#### APPL_DB → ConsumerStateTable

```
High-volume, critical path:
- Route updates (millions of routes)
- Port configuration
- Neighbor entries

Requirements:
- Cannot lose any update
- Must handle producer faster than consumer
- Must survive restarts
- Atomicity needed
```

#### SAI Events → NotificationConsumer

```
Asynchronous hardware events:
- Port oper status changes
- FDB learn/age events
- BFD session state changes
- PFC watchdog events

Requirements:
- Data must be captured at event time
- Cannot fetch later (state may change)
- Event ordering preserved
- Self-contained messages
```

#### CONFIG_DB / STATE_DB → SubscriberStateTable

```
Lower-volume, less critical:
- Configuration changes (human-initiated)
- State monitoring

Acceptable trade-offs:
- Config changes are rare
- Operator can re-apply if missed
- Simplicity over reliability
- Any tool can write (CLI, REST API)
```

---

## Redis Data Structures

### ConsumerStateTable (APPL_DB)

| Redis Key | Type | Written By | Read By | Purpose |
|-----------|------|------------|---------|---------|
| `_PORT_TABLE:Ethernet0` | HASH | Producer | Consumer | **Staging area** (temporary) |
| `PORT_TABLE:Ethernet0` | HASH | Consumer | Anyone | **Real table** (permanent) |
| `PORT_TABLE_KEY_SET` | SET | Producer | Consumer | Queue of pending keys |
| `PORT_TABLE_DEL_SET` | SET | Producer | Consumer | Keys marked for deletion |
| `PORT_TABLE_CHANNEL@0` | PUBSUB | Producer | Consumer | Notification channel |

### SubscriberStateTable (CONFIG_DB, STATE_DB)

| Redis Key | Type | Written By | Read By | Purpose |
|-----------|------|------------|---------|---------|
| `PORT_TABLE\|Ethernet0` | HASH | Any client | Consumer | Actual table |
| `__keyspace@N__:*` | PUBSUB | Redis (auto) | Consumer | Keyspace notifications |

---

## Producer-Consumer Model

### Multiple Producers, Single Consumer

This model supports **multiple producers** but requires a **single consumer** per table.

#### Multiple Producers (Safe)

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

**Why it's safe:**
- `SADD` is atomic
- `HSET` on different keys is independent
- `HSET` on same key - last writer wins (acceptable)
- SET ensures no duplicate keys

#### Single Consumer (Required)

**Why only one consumer:**
- `SPOP` is destructive - removes key from SET
- Lua script deletes staging hash after reading
- Second consumer would find nothing
- Ordering must be maintained for same key

### Guaranteed No Missed Updates

1. **KEY_SET is a Redis SET** - keys are unique, even if producer writes same key multiple times

2. **Staging hash has latest value** - producer overwrites, consumer gets latest

3. **Lua script is atomic** - no race condition during pop/read/write/delete

```
Producer updates speed three times:

Time 0ms: HSET _PORT_TABLE:Ethernet0 speed 10000   →  staging has 10000
Time 1ms: HSET _PORT_TABLE:Ethernet0 speed 25000   →  staging has 25000
Time 2ms: HSET _PORT_TABLE:Ethernet0 speed 100000  →  staging has 100000

Time 5ms: Consumer pops → HGETALL _PORT_TABLE:Ethernet0 → gets 100000 (latest!)
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

Executor (orch.h:106)
    └── ConsumerBase (orch.h:155)
            └── Consumer (orch.h:230)
```

### Key Files

| File | Purpose |
|------|---------|
| `orchagent/orchdaemon.cpp` | Main event loop |
| `orchagent/orch.cpp` | Orch base class, Consumer::execute() |
| `orchagent/orch.h` | Class definitions |
| `orchagent/portsorch.cpp` | PortsOrch::doPortTask() |
| `sonic-swss-common/common/select.cpp` | Select class implementation |
| `sonic-swss-common/common/redisselect.cpp` | RedisSelect::readData() |
| `sonic-swss-common/common/consumerstatetable.cpp` | ConsumerStateTable::pops() |
| `sonic-swss-common/common/subscriberstatetable.cpp` | SubscriberStateTable implementation |
| `sonic-swss-common/common/consumer_state_table_pops.lua` | Atomic pop Lua script |

### Execution Flow

```
OrchDaemon::start()
    │
    └─► Select::select()
            │
            ├─► epoll_wait()
            ├─► Selectable::readData()    // Count notifications
            └─► returns Selectable*
                    │
                    ▼
            Consumer::execute()           // orch.cpp:503
                │
                ├─► pops()                // Fetch data from Redis
                ├─► addToSync()           // Queue in m_toSync
                └─► drain()               // orch.cpp:556
                        │
                        ▼
                    Orch::doTask(Consumer&)
                        │
                        ▼
                    PortsOrch::doTask()   // portsorch.cpp:6206
                        │
                        ▼
                    doPortTask()          // portsorch.cpp:4301
                        │
                        └─► SAI API calls
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
