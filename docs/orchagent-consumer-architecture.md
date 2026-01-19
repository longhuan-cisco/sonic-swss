# Orchagent Consumer Architecture

This document explains how orchagent consumes data from Redis databases using ConsumerStateTable and SubscriberStateTable, and how the main event loop processes these events.

## Table of Contents

1. [Overview](#overview)
2. [Threading Model](#threading-model)
3. [Select Loop Architecture](#select-loop-architecture)
4. [ConsumerStateTable vs SubscriberStateTable](#consumerstatetable-vs-subscriberstatetable)
5. [ConsumerStateTable Deep Dive](#consumerstatetable-deep-dive)
6. [SubscriberStateTable Deep Dive](#subscriberstatetable-deep-dive)
7. [Redis Data Structures](#redis-data-structures)
8. [Producer-Consumer Model](#producer-consumer-model)
9. [Code References](#code-references)

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
