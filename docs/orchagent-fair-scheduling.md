# Orchagent Fair Scheduling for DB/Table Notifications

This document explains how orchagent ensures fair scheduling for different database/table notifications and their corresponding orchestration agents (orchs) in its selectable loop.

## Table of Contents

1. [Overview](#overview)
2. [Main Event Loop](#main-event-loop)
3. [Two-Stage Mechanism: readData and pops](#two-stage-mechanism-readdata-and-pops)
4. [Select Class Internals (sonic-swss-common)](#select-class-internals-sonic-swss-common)
5. [Fairness Mechanisms](#fairness-mechanisms)
6. [Retry Task Processing](#retry-task-processing)
7. [BGP Routes: A Real-World Example](#bgp-routes-a-real-world-example)
8. [Key Configuration Parameters](#key-configuration-parameters)
9. [Summary](#summary)

---

## Overview

Orchagent uses a **multi-layered fairness approach** to prevent any single busy table from starving others. The core challenge is handling scenarios where one data source (e.g., BGP routes) can generate massive volumes of notifications while ensuring other critical operations (e.g., port state changes) remain responsive.

### Key Design Principles

| Principle | Implementation |
|-----------|----------------|
| Decouple notification detection from processing | Two-stage `readData()`/`pops()` mechanism |
| Limit work per iteration | Batch size limits on `pops()` |
| Ensure all orchs get scheduled | Explicit `doTask()` loop on all orchs |
| Handle dependencies without blocking | Retry task queue with constraint resolution |

---

## Main Event Loop

The core event loop is in `OrchDaemon::start()` (`orchagent/orchdaemon.cpp:911-1050`):

```cpp
void OrchDaemon::start()
{
    // Register ALL consumers from ALL orchs to Select
    for (Orch *o : m_orchList)
    {
        m_select->addSelectables(o->getSelectables());
    }

    while (true)
    {
        Selectable *s;
        int ret;

        // Wait for events (up to 1000ms timeout)
        ret = m_select->select(&s, SELECT_TIMEOUT);

        if (ret == Select::TIMEOUT)
        {
            // Timeout: process all orchs anyway
            for (Orch *o : m_orchList)
                o->doTask();
            continue;
        }

        if (ret == Select::ERROR)
        {
            continue;
        }

        // Process the ONE consumer that triggered
        auto *c = (Executor *)s;
        c->execute();

        // Process ALL orchs (for retry tasks)
        for (Orch *o : m_orchList)
            o->doTask();
    }
}
```

### Key Points

- **All consumers registered**: Every consumer from every orch is added to Select
- **1000ms timeout guarantee**: Even with no events, all orchs get processed periodically
- **Single consumer per select**: `select()` returns ONE ready consumer
- **All orchs processed**: `doTask()` called on ALL orchs every iteration

---

## Two-Stage Mechanism: readData and pops

The two-stage mechanism decouples **notification detection** from **notification processing**.

### Stage 1: readData() - Notification Detection & Buffering

```cpp
// orch.h:120 - Executor decorates underlying Selectable
uint64_t readData() override { return m_selectable->readData(); }
```

- Called by `Select::select()` when Redis fd becomes ready
- Reads notifications from Redis into an **internal buffer**
- **Does NOT process data** - only buffers it
- Cheap operation

### Stage 2: execute() - Data Extraction & Processing

```cpp
// orch.cpp:503-518
void Consumer::execute()
{
    auto entries = std::make_shared<std::deque<KeyOpFieldsValuesTuple>>();
    getConsumerTable()->pops(*entries);  // Extract from buffer (limited by batch size)

    processAnyTask([=](){
        addToSync(entries);  // Queue to m_toSync
        drain();             // Process via doTask(Consumer&)
    });
}
```

### Cost Analysis

```
readData()  →  pops()  →  addToSync()  →  drain()  →  doTask()
    │            │                          │
    │            │                          └── EXPENSIVE (SAI API calls)
    │            │
    │            └── CHEAP (extract from buffer)
    │
    └── CHEAP (notification detection + buffering)
```

**Key Insight**: The expensive part is `drain()` which calls `doTask(Consumer&)` - this is where actual SAI API calls happen. The `pops()` batch limit acts as a **throttle gate** for `drain()`.

---

## Select Class Internals (sonic-swss-common)

This section explains how the `Select` class implements fair scheduling using `readData()`, `hasCachedData()`, and priority-based selection.

### Core Data Structures

```cpp
// select.h (sonic-swss-common)
class Select
{
private:
    int m_epoll_fd;                                    // epoll file descriptor
    std::unordered_map<int, Selectable *> m_objects;   // fd -> Selectable mapping
    std::set<Selectable *, Select::cmp> m_ready;       // Priority-ordered ready set

    // Priority comparator for m_ready set
    struct cmp
    {
        bool operator()(const Selectable* a, const Selectable* b) const
        {
            // 1. Higher priority wins
            if (a->getPri() > b->getPri()) return true;
            if (a->getPri() < b->getPri()) return false;

            // 2. If same priority: least recently used wins (fair scheduling)
            if (a->getLastUsedTime() < b->getLastUsedTime()) return true;
            if (a->getLastUsedTime() > b->getLastUsedTime()) return false;

            return false;
        }
    };
};
```

**Key insight**: The `m_ready` set is ordered by:
1. **Priority** (higher first)
2. **Last used time** (older first, for fairness among same priority)

### Selectable Interface

```cpp
// selectable.h (sonic-swss-common)
class Selectable
{
public:
    virtual int getFd() = 0;              // File descriptor for epoll
    virtual uint64_t readData() = 0;      // Read from fd into internal buffer

    virtual bool hasData() { return true; }        // Has data to process?
    virtual bool hasCachedData() { return false; } // Has MORE data after this read?
    virtual void updateAfterRead() { }             // Called after data is consumed

    int getPri() const { return m_priority; }

private:
    int m_priority;
    std::chrono::time_point<std::chrono::steady_clock> m_last_used_time;
};
```

### Consumer Class Implementations

Different consumer classes implement `hasCachedData()` differently based on their buffering strategy.

#### ConsumerStateTable & ConsumerTable (via RedisSelect)

These classes inherit from `RedisSelect` through the inheritance chain:

```
ConsumerStateTable → ConsumerTableBase → TableConsumable → RedisSelect
ConsumerTable      → ConsumerTableBase → TableConsumable → RedisSelect
```

```cpp
// redisselect.cpp (sonic-swss-common)
class RedisSelect : public Selectable
{
protected:
    long long int m_queueLength;  // Notification counter

public:
    uint64_t readData() override
    {
        redisReply *reply = nullptr;

        // Read first reply (blocking read that triggered epoll)
        redisGetReply(m_subscribe->getContext(), &reply);
        freeReplyObject(reply);
        m_queueLength++;

        // Drain any additional buffered replies (non-blocking)
        do {
            status = redisGetReplyFromReader(m_subscribe->getContext(), &reply);
            if (reply != nullptr && status == REDIS_OK) {
                m_queueLength++;
                freeReplyObject(reply);
            }
        } while (reply != nullptr && status == REDIS_OK);

        return 0;
    }

    bool hasData() override
    {
        return m_queueLength > 0;  // Has at least one notification
    }

    bool hasCachedData() override
    {
        return m_queueLength > 1;  // Has MORE than one notification
    }

    void updateAfterRead() override
    {
        m_queueLength--;  // Decrement after each select() return
    }
};
```

**Key insight**: `m_queueLength` tracks pending notifications:
- `readData()` increments for each notification read from Redis
- `updateAfterRead()` decrements after Select returns the Selectable
- `hasCachedData()` returns true if more than one notification remains

#### SubscriberStateTable (Dual Buffer)

`SubscriberStateTable` overrides with its own implementation using two buffers:

```cpp
// subscriberstatetable.cpp (sonic-swss-common)
class SubscriberStateTable : public ConsumerTableBase
{
private:
    std::deque<KeyOpFieldsValuesTuple> m_buffer;                    // Processed entries
    std::deque<std::shared_ptr<RedisReply>> m_keyspace_event_buffer; // Raw keyspace events

public:
    bool hasData() override
    {
        return m_buffer.size() > 0 || m_keyspace_event_buffer.size() > 0;
    }

    bool hasCachedData() override
    {
        return m_buffer.size() + m_keyspace_event_buffer.size() > 1;  // Combined count
    }
};
```

**Key difference**: Counts actual buffered items across both buffers.

#### NotificationConsumer (Message Queue)

`NotificationConsumer` uses a message queue:

```cpp
// notificationconsumer.cpp (sonic-swss-common)
class NotificationConsumer : public Selectable
{
private:
    std::queue<std::string> m_queue;  // Processed notification messages

public:
    bool hasData() override
    {
        return m_queue.size() > 0;
    }

    bool hasCachedData() override
    {
        return m_queue.size() > 1;  // More than one message queued
    }
};
```

#### ZmqConsumerStateTable (Always Re-insert)

`ZmqConsumerStateTable` uses a **different strategy**:

```cpp
// zmqconsumerstatetable.h (sonic-swss-common)
class ZmqConsumerStateTable : public Selectable
{
private:
    std::queue<KeyOpFieldsValuesTuple> m_receivedOperationQueue;

public:
    bool hasData() override
    {
        std::lock_guard<std::mutex> lock(m_receivedQueueMutex);
        return !m_receivedOperationQueue.empty();
    }

    bool hasCachedData() override
    {
        return hasData();  // NOTE: Returns true if ANY data exists
    }
};
```

**Key difference**: Returns `true` whenever there's any data, causing **always re-insert** until completely drained.

#### Implementation Comparison

| Class | hasCachedData() Logic | Re-insertion Behavior |
|-------|----------------------|----------------------|
| **ConsumerStateTable** | `m_queueLength > 1` | Re-insert if 2+ notifications |
| **ConsumerTable** | `m_queueLength > 1` | Re-insert if 2+ notifications |
| **SubscriberStateTable** | `buffers.size() > 1` | Re-insert if 2+ items |
| **NotificationConsumer** | `m_queue.size() > 1` | Re-insert if 2+ messages |
| **ZmqConsumerStateTable** | `hasData()` | **Always** re-insert if any data |

The `> 1` pattern ensures re-insertion only if work remains after current iteration. ZmqConsumerStateTable always re-inserts, relying on `pops()` batch limit for fairness.

#### Critical: readData() vs pops() Data Sources

A key architectural detail is whether `readData()` and `pops()` operate on the **same buffer**:

| Class | readData() target | pops() source | Same buffer? |
|-------|------------------|---------------|--------------|
| **ConsumerStateTable** | `m_queueLength` (notification counter) | Redis Lua script (direct query) | **NO** |
| **ConsumerTable** | `m_queueLength` (notification counter) | Redis Lua script (direct query) | **NO** |
| **SubscriberStateTable** | `m_keyspace_event_buffer` | `m_keyspace_event_buffer` | **YES** |
| **NotificationConsumer** | `m_queue` | `m_queue` | **YES** |
| **ZmqConsumerStateTable** | `m_receivedOperationQueue` | `m_receivedOperationQueue` | **YES** |

**ConsumerStateTable/ConsumerTable Design:**

```
readData():                              pops():
┌──────────────────────────┐            ┌──────────────────────────┐
│ Redis SUBSCRIBE channel  │            │ EVALSHA Lua script       │
│ (notification socket)    │            │ (direct Redis query)     │
│                          │            │                          │
│ m_queueLength++          │            │ Reads from Redis SET     │
│ per notification         │            │ up to POP_BATCH_SIZE     │
└──────────────────────────┘            └──────────────────────────┘
         │                                        │
         │ DIFFERENT DATA SOURCES                 │
         └────────────────────────────────────────┘
```

**Implication**: For ConsumerStateTable, `hasCachedData()` checks `m_queueLength` (notification count), but this does NOT accurately reflect actual items in Redis:

```
Example:
- 3 notifications arrive → m_queueLength = 3
- But those 3 notifications could represent 5000 key changes in Redis
- hasCachedData() returns: 3 > 1 = true
- pops(128) fetches 128 items from Redis
- 4872 items STILL in Redis, but m_queueLength is now 2

The notification count is an APPROXIMATION, not an accurate indicator.
```

**SubscriberStateTable/NotificationConsumer/ZmqConsumerStateTable Design:**

```
readData():                              pops():
┌──────────────────────────┐            ┌──────────────────────────┐
│ Reads into buffer        │───────────▶│ Reads from SAME buffer   │
│ (m_queue, m_buffer, etc) │            │                          │
└──────────────────────────┘            └──────────────────────────┘
                    │
                    │ SAME DATA SOURCE
                    │
         hasCachedData() checks this buffer
         → More accurate indicator
```

**Why ConsumerStateTable uses this design:**

1. **Efficiency**: Data stays in Redis until needed; no memory duplication
2. **Atomicity**: Lua script ensures atomic pop operations
3. **Batch control**: `POP_BATCH_SIZE` enforced at Redis level
4. **Trade-off**: `hasCachedData()` is approximate, but `pops()` batch limit ensures fairness regardless

#### Critical Timing: hasCachedData() is Checked BEFORE pops()

A crucial detail is **when** `hasCachedData()` is evaluated relative to `pops()`:

```
Timeline:
─────────────────────────────────────────────────────────────────────────────
1. Select::poll_descriptors()
   │
   ├─ readData()              ← Buffer notifications / increment counter
   ├─ hasCachedData()         ← Checked HERE (BEFORE pops!)
   ├─ updateAfterRead()       ← Decrement counter
   └─ return selectable to orchagent

2. Consumer::execute()
   │
   └─ pops(128)               ← Actual data extraction happens HERE (AFTER!)
      └─ drain()              ← Processing happens here
─────────────────────────────────────────────────────────────────────────────
```

**Implication**: `hasCachedData()` serves as a **hint** that there might be more work, but it is **not a precise indicator** of remaining items:

- For **ConsumerStateTable**: The hint is based on notification count, which may not reflect actual Redis data
- For **SubscriberStateTable/NotificationConsumer**: The hint is based on buffer size before `pops()` drains it

**The `pops()` batch limit is the true fairness enforcer**, not `hasCachedData()`. Even if `hasCachedData()` incorrectly returns false (causing no re-insertion), the next Redis notification will trigger a new `readData()` cycle. Conversely, if it incorrectly returns true, the batch limit still caps work per iteration.

### Select::select() - The Entry Point

```cpp
// select.cpp (sonic-swss-common)
int Select::select(Selectable **c, int timeout, bool interrupt_on_signal)
{
    *c = NULL;

    // FIRST: Check if any selectable already has cached data (non-blocking)
    int ret = poll_descriptors(c, 0);  // timeout=0: immediate return

    // Return immediately if we have data, error, or caller wanted non-blocking
    if (ret != Select::TIMEOUT || timeout == 0)
        return ret;

    // SECOND: Wait for new data with actual timeout
    ret = poll_descriptors(c, timeout, interrupt_on_signal);

    return ret;
}
```

**Two-phase approach**:
1. First check for already-cached data (from previous `hasCachedData()` re-insertions)
2. Only block on epoll if no cached data exists

### Select::poll_descriptors() - The Core Logic

```cpp
// select.cpp (sonic-swss-common)
int Select::poll_descriptors(Selectable **c, unsigned int timeout, bool interrupt_on_signal)
{
    int sz_selectables = static_cast<int>(m_objects.size());
    std::vector<struct epoll_event> events(sz_selectables);

    // ┌─────────────────────────────────────────────────────────────┐
    // │ STEP 1: epoll_wait - Wait for fd events                     │
    // └─────────────────────────────────────────────────────────────┘
    int ret = ::epoll_wait(m_epoll_fd, events.data(), sz_selectables, timeout);

    if (ret < 0)
        return Select::ERROR;

    // ┌─────────────────────────────────────────────────────────────┐
    // │ STEP 2: readData() - Buffer notifications from ready fds    │
    // └─────────────────────────────────────────────────────────────┘
    for (int i = 0; i < ret; ++i)
    {
        int fd = events[i].data.fd;
        Selectable* sel = m_objects[fd];

        sel->readData();      // Read from Redis into internal buffer
        m_ready.insert(sel);  // Add to priority-ordered ready set
    }

    // ┌─────────────────────────────────────────────────────────────┐
    // │ STEP 3: Select highest priority Selectable with data        │
    // └─────────────────────────────────────────────────────────────┘
    while (!m_ready.empty())
    {
        auto sel = *m_ready.begin();  // Highest priority (due to comparator)
        m_ready.erase(sel);

        sel->updateLastUsedTime();    // Update for fair scheduling

        // Skip if no actual data (edge case)
        if (!sel->hasData())
            continue;

        *c = sel;  // Return this selectable

        // ┌─────────────────────────────────────────────────────────┐
        // │ STEP 4: Re-insert if more cached data remains           │
        // └─────────────────────────────────────────────────────────┘
        if (sel->hasCachedData())
        {
            m_ready.insert(sel);  // Will be selected again (with updated time)
        }

        sel->updateAfterRead();  // Decrement notification counter

        return Select::OBJECT;
    }

    return Select::TIMEOUT;
}
```

### Complete Flow Diagram

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         Select::select() Entry                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│ poll_descriptors(timeout=0)         │  Check for already-cached data        │
│   - Skip epoll_wait (timeout=0)     │  from previous hasCachedData()        │
│   - Check m_ready set               │  re-insertions                        │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                     ┌────────────────┴────────────────┐
                     │                                 │
              [has cached data]                 [no cached data]
                     │                                 │
                     ▼                                 ▼
              Return immediately          ┌───────────────────────────────────┐
                                          │ poll_descriptors(timeout=1000ms)  │
                                          └───────────────────────────────────┘
                                                       │
                                                       ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STEP 1: epoll_wait()                                      │
│                                                                              │
│   Block until one or more fds have data (or timeout)                        │
│   Returns: list of ready file descriptors                                   │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STEP 2: readData() on each ready fd                       │
│                                                                              │
│   for each ready_fd:                                                        │
│       selectable = m_objects[ready_fd]                                      │
│       selectable->readData()     ← Reads Redis replies, increments counter  │
│       m_ready.insert(selectable) ← Add to priority-ordered set              │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STEP 3: Pick highest priority from m_ready                │
│                                                                              │
│   m_ready is std::set with custom comparator:                               │
│     1. Higher m_priority first                                              │
│     2. If equal priority: older m_last_used_time first (fairness)           │
│                                                                              │
│   sel = *m_ready.begin()         ← Get highest priority                     │
│   m_ready.erase(sel)                                                        │
│   sel->updateLastUsedTime()      ← Mark as "just used" for fairness         │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    STEP 4: Check hasCachedData()                             │
│                                                                              │
│   if (sel->hasCachedData())      ← m_queueLength > 1 ?                       │
│       m_ready.insert(sel)        ← Re-insert for next iteration             │
│                                    (with updated lastUsedTime)              │
│                                                                              │
│   sel->updateAfterRead()         ← Decrement m_queueLength                  │
│   return sel                     ← Return to orchagent                      │
└─────────────────────────────────────────────────────────────────────────────┘
                                      │
                                      ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    Back to OrchDaemon::start()                               │
│                                                                              │
│   Executor* c = (Executor*)sel;                                             │
│   c->execute();                  ← Calls pops() + addToSync() + drain()     │
└─────────────────────────────────────────────────────────────────────────────┘
```

### hasCachedData() Re-insertion Example

```
Scenario: RouteOrch consumer receives 300 notifications

Time T1: epoll_wait returns (RouteOrch fd ready)
         │
         ▼
    readData() on RouteOrch
         - Reads all 300 Redis replies
         - m_queueLength = 300
         │
         ▼
    m_ready.insert(RouteOrch)
         │
         ▼
    Pick RouteOrch (highest priority with data)
         - updateLastUsedTime() → now
         - hasCachedData()? → YES (300 > 1)
         - m_ready.insert(RouteOrch)  ← RE-INSERTED
         - updateAfterRead() → m_queueLength = 299
         │
         ▼
    Return RouteOrch to orchagent
         │
         ▼
    execute() → pops(128) → drain()  ← Process 128 routes
         │
         ▼
Time T2: select() called again
         │
         ▼
    poll_descriptors(timeout=0)  ← Non-blocking check first
         - m_ready already contains RouteOrch (from re-insertion)
         - No epoll_wait needed!
         │
         ▼
    Pick RouteOrch again
         - hasCachedData()? → YES (299 > 1)
         - m_ready.insert(RouteOrch)  ← RE-INSERTED again
         - updateAfterRead() → m_queueLength = 298
         │
         ▼
    ... continues until m_queueLength <= 1 ...
```

### Fair Scheduling with Same Priority

```
Scenario: Two consumers (A and B) with same priority, both have data

Time T1: Both A and B added to m_ready
         m_ready order: [A, B] (A is older, selected first)
         │
         ▼
    Select A
         - updateLastUsedTime(A) → A.time = T1
         - Return A
         │
         ▼
Time T2: Both still have data, both in m_ready
         m_ready order: [B, A] (B is now older than A)
         │
         ▼
    Select B  ← Fair! B gets a turn
         - updateLastUsedTime(B) → B.time = T2
         │
         ▼
Time T3: Both still have data
         m_ready order: [A, B] (A is now older than B)
         │
         ▼
    Select A  ← Alternates fairly
```

This is validated by the unit test in `sonic-swss-common/tests/selectable_priority.cpp:246-249`:
```cpp
// "we have fair scheduler. we've read different selectables on the second read"
EXPECT_NE(selectcs1, selectcs2);
```

### Interface Summary

| Method | When Called | What It Does |
|--------|-------------|--------------|
| `readData()` | By `poll_descriptors()` after epoll | Read from Redis, increment `m_queueLength` |
| `hasData()` | By `poll_descriptors()` before returning | Check if `m_queueLength > 0` |
| `hasCachedData()` | By `poll_descriptors()` after selection | Check if `m_queueLength > 1`, re-insert if true |
| `updateAfterRead()` | By `poll_descriptors()` before return | Decrement `m_queueLength` |
| `updateLastUsedTime()` | By `poll_descriptors()` on selection | Update timestamp for fair scheduling |

---

## Fairness Mechanisms

### 1. Batch Size Limit (Primary Throttle)

```cpp
// Default: gBatchSize = 128
getConsumerTable()->pops(*entries);  // Extracts at most 128 items
```

Each `pops()` call extracts at most `gBatchSize` entries, limiting how much work flows into `drain()` per iteration.

**Without throttle:**
- 100,000 routes arrive
- All processed in one `drain()` call
- Other orchs starve for seconds

**With throttle:**
- 100,000 routes arrive
- 128 processed per iteration
- ~780 iterations, other orchs interleaved

### 2. Table Priorities

Tables are registered with explicit priorities (`orchagent/orchdaemon.cpp`):

```cpp
const int routeorch_pri = 5;       // LOW - routes can tolerate latency
const int portsorch_base_pri = 40; // HIGH - port state is time-critical
const int natorch_pri = 50;        // HIGHEST
```

Priority affects which consumer Select returns when multiple have data ready.

### 3. Consumer Ordering Within Orch

Consumers within an orch are stored in `std::map<string, Executor>` (`orch.h:153`), providing **lexicographic ordering by table name**.

### 4. Select Timeout Guarantee

```cpp
#define SELECT_TIMEOUT 1000  // 1 second
```

Even if no fd events occur, all orchs get a `doTask()` call every 1000ms maximum.

---

## Retry Task Processing

### Why doTask() Loop is Needed

The `execute()` path only handles **new Redis notifications**:

```
Redis notification → readData() → pops() → addToSync() → drain()
```

But tasks can fail due to unmet dependencies and enter the **retry queue**:

```cpp
// Task fails due to missing dependency
addToRetry(task, constraint);  // Goes to m_toRetry

// m_toRetry has NO Redis fd - no Select event will wake it
```

### The doTask() Solution

```cpp
// orch.cpp:836-849
void Orch::doTask()
{
    auto threshold = gBatchSize == 0 ? 30000 : gBatchSize;
    size_t count = 0;

    for (auto &it : m_consumerMap)
    {
        count += retryToSync(it.first, threshold - count);  // Move retries back
        it.second->drain();  // Process them
    }
}
```

### Retry Flow Example

```
1. RouteOrch: Route 10.0.0.0/24 via 192.168.1.1
   - Neighbor 192.168.1.1 doesn't exist yet
   - addToRetry(route, waitFor(NEIGHBOR, "192.168.1.1"))

2. NeighOrch::execute(): Neighbor 192.168.1.1 created
   - notifyRetry() marks constraint resolved

3. Next iteration: doTask() on RouteOrch
   - retryToSync() moves route back to m_toSync
   - drain() processes it successfully
```

### Summary: execute() vs doTask()

| Aspect | execute() via Select | doTask() on all orchs |
|--------|---------------------|----------------------|
| Trigger | Redis fd ready | Every iteration |
| Scope | ONE consumer | ALL consumers of ALL orchs |
| Handles new notifications | Yes | No (just drains) |
| Handles retry tasks | No | Yes (retryToSync) |

---

## BGP Routes: A Real-World Example

BGP routes flow through APPL_DB, making throttling essential.

### Route Flow

```
FRR/BGP Daemon (Zebra)
       │
       │ FPM protocol (netlink over TCP port 2620)
       ▼
   fpmsyncd
       │
       │ ProducerStateTable.set()
       ▼
APPL_DB:ROUTE_TABLE
       │
       │ ConsumerStateTable
       ▼
   RouteOrch
       │
       │ SAI API calls
       ▼
   ASIC Hardware
```

### Why Throttling Matters for BGP

| Event | Without Throttling | With Throttling |
|-------|-------------------|-----------------|
| BGP peer flap (50k routes) | 10s+ orchagent freeze | Routes processed over ~400 iterations |
| Simultaneous link failure | Blocked until routes done | Processed within 1-2 select cycles |
| Control plane responsiveness | Dead | Maintained |

### Priority Design Rationale

```cpp
const int routeorch_pri = 5;       // LOW
const int portsorch_base_pri = 40; // HIGH
```

- **Routes**: Can tolerate convergence latency (seconds acceptable)
- **Ports**: Link state changes are time-critical (milliseconds matter)

---

## Key Configuration Parameters

| Parameter | Default | Location | Purpose |
|-----------|---------|----------|---------|
| `gBatchSize` | 128 | main.cpp (`-b` flag) | Pop limit per consumer per execute() |
| `SELECT_TIMEOUT` | 1000ms | orchdaemon.cpp | Max time before forced doTask() on all orchs |
| Table Priorities | 5-50 | orchdaemon.cpp | Priority hints to Select |
| Retry Quota | gBatchSize | orch.cpp:840 | Limits retries moved per doTask() call |

---

## Summary

### Multi-Layered Fairness Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 1: Select Loop                                                │
│  - All consumers registered with priorities                          │
│  - 1000ms timeout guarantee                                          │
│  - Returns ONE consumer per iteration                                │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 2: Two-Stage Mechanism                                        │
│  - readData(): notification detection + buffering (cheap)            │
│  - pops(): data extraction with batch limit (cheap)                  │
│  - drain(): actual processing via doTask() (EXPENSIVE)               │
│  - Batch limit throttles work flowing to drain()                     │
└─────────────────────────────────────────────────────────────────────┘
                                   │
                                   ▼
┌─────────────────────────────────────────────────────────────────────┐
│  Layer 3: Retry Task Handling                                        │
│  - Failed tasks go to m_toRetry with constraints                     │
│  - doTask() on ALL orchs moves retries back via retryToSync()        │
│  - Enables cross-orch dependency resolution                          │
└─────────────────────────────────────────────────────────────────────┘
```

### Key Takeaways

1. **Notification detection is decoupled from processing** - `readData()` buffers, `drain()` processes
2. **pops() batch limit is the primary throttle** - gates how much work enters expensive `drain()`
3. **hasCachedData() is a hint, not precise** - checked BEFORE `pops()`, based on notification count (not actual data remaining)
4. **doTask() loop exists for retry tasks** - they have no Redis fd to trigger Select
5. **Priorities favor time-critical operations** - ports > routes
6. **1000ms timeout ensures liveness** - no orch completely starved

---

## References

### sonic-swss (this repository)

- `orchagent/orchdaemon.cpp:911-1050` - Main event loop
- `orchagent/orch.cpp:503-518` - Consumer::execute() with pops()
- `orchagent/orch.cpp:836-849` - Orch::doTask() with retryToSync()
- `orchagent/orch.h:119-123` - Executor decorator methods
- `fpmsyncd/routesync.cpp` - BGP route producer

### sonic-swss-common

- `common/select.cpp` - Select class with poll_descriptors() implementation
- `common/select.h` - Select class with m_ready set and priority comparator
- `common/selectable.h` - Selectable interface (readData, hasCachedData, etc.)
- `common/redisselect.cpp` - RedisSelect with m_queueLength counter
- `common/consumerstatetable.cpp` - ConsumerStateTable pops() implementation
- `tests/selectable_priority.cpp` - Unit tests for priority and fair scheduling
