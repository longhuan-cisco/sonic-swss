# Orchagent Fair Scheduling for DB/Table Notifications

This document explains how orchagent ensures fair scheduling for different database/table notifications and their corresponding orchestration agents (orchs) in its selectable loop.

## Table of Contents

1. [Overview](#overview)
2. [Main Event Loop](#main-event-loop)
3. [Two-Stage Mechanism: readData and pops](#two-stage-mechanism-readdata-and-pops)
4. [Fairness Mechanisms](#fairness-mechanisms)
5. [Retry Task Processing](#retry-task-processing)
6. [BGP Routes: A Real-World Example](#bgp-routes-a-real-world-example)
7. [Key Configuration Parameters](#key-configuration-parameters)
8. [Summary](#summary)

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

### The Glue: hasCachedData()

```cpp
// orch.h:121
bool hasCachedData() override { return m_selectable->hasCachedData(); }
```

- Returns `true` if internal buffer still has unprocessed data
- Allows Select to return the same consumer again if it has more buffered data
- Enables draining large buffers across multiple iterations

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
3. **doTask() loop exists for retry tasks** - they have no Redis fd to trigger Select
4. **Priorities favor time-critical operations** - ports > routes
5. **1000ms timeout ensures liveness** - no orch completely starved

---

## References

- `orchagent/orchdaemon.cpp:911-1050` - Main event loop
- `orchagent/orch.cpp:503-518` - Consumer::execute() with pops()
- `orchagent/orch.cpp:836-849` - Orch::doTask() with retryToSync()
- `orchagent/orch.h:119-123` - Executor decorator methods
- `fpmsyncd/routesync.cpp` - BGP route producer
