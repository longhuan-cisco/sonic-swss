# Orchagent <-> GBSyncd Communication & Issues Analysis

> Based on [sonic-net/sonic-swss @ `f39134c`](https://github.com/sonic-net/sonic-swss/tree/f39134cbb25b6cf27358437a88de6c55c6dc16a1)

---

## Table of Contents

- [1. Communication Architecture](#1-communication-architecture)
  - [1.1 Database Separation](#11-database-separation)
  - [1.2 Context-Based Routing](#12-context-based-routing)
  - [1.3 ASIC_STATE Hash Table Ownership](#13-asic_state-hash-table-ownership)
  - [1.4 sairedis.rec Recording](#14-sairedisrec-recording)
  - [1.5 gearsyncd vs gbsyncd](#15-gearsyncd-vs-gbsyncd)
- [2. Port Lifecycle: NPU Port vs Gearbox Port](#2-port-lifecycle-npu-port-vs-gearbox-port)
  - [2.1 Creation Order](#21-creation-order)
  - [2.2 Gearbox Port Objects Created](#22-gearbox-port-objects-created)
  - [2.3 Dependency Direction](#23-dependency-direction)
  - [2.4 Deletion: Gearbox Ports Are NOT Cleaned Up](#24-deletion-gearbox-ports-are-not-cleaned-up)
- [3. Sync Mode: Blocking Behavior](#3-sync-mode-blocking-behavior)
- [4. SAI Call Timeout & Error Handling](#4-sai-call-timeout--error-handling)
  - [4.1 Timeout Mechanism](#41-timeout-mechanism)
  - [4.2 Retry Policy by SAI Status](#42-retry-policy-by-sai-status)
  - [4.3 Two-Layer Retry for Port Deletion](#43-two-layer-retry-for-port-deletion)
  - [4.4 handleSaiFailure Behavior](#44-handlesaifailure-behavior)
  - [4.5 initGearboxPort Return Value Ignored](#45-initgearboxport-return-value-ignored)
- [5. Debugging Mechanisms for Slow/Stuck SAI Calls](#5-debugging-mechanisms-for-slowstuck-sai-calls)
- [6. Supervisord Monitoring](#6-supervisord-monitoring)
  - [6.1 Configuration](#61-configuration)
  - [6.2 Behavior on Missed Heartbeats](#62-behavior-on-missed-heartbeats)
  - [6.3 What Actually Kills Orchagent](#63-what-actually-kills-orchagent)
- [7. End-to-End Stuck/Timeout Scenario Timeline](#7-end-to-end-stucktimeout-scenario-timeline)
- [8. Gearbox-Specific Timeout Consequences](#8-gearbox-specific-timeout-consequences)
- [9. `config interface breakout` CLI — GB_ASIC_DB Verification Gap](#9-config-interface-breakout-cli--gb_asic_db-verification-gap)
- [10. Summary of Identified Gaps](#10-summary-of-identified-gaps)

---

## 1. Communication Architecture

### 1.1 Database Separation

Orchagent communicates with syncd and gbsyncd through **separate Redis databases** using the same SAI Redis library (libsairedis):

| DB Purpose | Regular syncd | GBSyncd (Gearbox) |
|---|---|---|
| ASIC state | `ASIC_DB` (index 1) | `GB_ASIC_DB` (index 9) |
| Counters | `COUNTERS_DB` (index 2) | `GB_COUNTERS_DB` (index 10) |
| Flex counters | `FLEX_COUNTER_DB` (index 5) | `GB_FLEX_COUNTER_DB` (index 11) |

### 1.2 Context-Based Routing

Both syncd and gbsyncd use the **same SAI Redis request/response pattern** (ProducerTable on ASIC_STATE, Redis list for notifications), but routing is determined by `SAI_REDIS_SWITCH_ATTR_CONTEXT`:

- **Main NPU switch** (`orchagent/main.cpp:723`): context = `gSwitchId` → routes to syncd via ASIC_DB
- **Gearbox PHY switch** (`orchagent/saihelper.cpp:527`): context = `phy->context_id` → routes to gbsyncd via GB_ASIC_DB

The `context_config.json` on the device maps each context ID to its syncd/gbsyncd instance.

### 1.3 ASIC_STATE Hash Table Ownership

The `ASIC_STATE` hash entries in both ASIC_DB and GB_ASIC_DB are written/deleted **exclusively by the server side** (syncd/gbsyncd), not by orchagent:

- **Client (orchagent via libsairedis)**: sends "create"/"remove"/"set" **commands** via Redis lists (`RedisRemoteSaiInterface.cpp:840-893`)
- **Server (syncd/gbsyncd)**: picks up commands, calls vendor SAI API, then writes hash entries on success (`syncd/RedisClient.cpp:503-598`)
  - `createAsicObject()` → `m_dbAsic->hset(key, ...)`
  - `removeAsicObject()` → `m_dbAsic->del(key)`

### 1.4 sairedis.rec Recording

Both syncd and gbsyncd SAI calls are recorded to the **same sairedis.rec file**. There is one global `Recorder::Instance()` singleton per orchagent process. Recording attributes are configured only on the main switch (`saihelper.cpp:346-389`); gearbox switches inherit the global recorder.

To distinguish entries: correlate the **switch OID** in each recorded operation. Identify the gearbox switch OID from the `create_switch` call that has the gearbox `context_id`.

### 1.5 gearsyncd vs gbsyncd

- **gearsyncd** (`sonic-swss/gearsyncd/`): a **config loader only** — reads gearbox config from ConfigDB, writes to APPL_DB, signals `GearboxConfigDone`. Not a SAI process.
- **gbsyncd** (in `sonic-sairedis`): the actual SAI syncd process serving GB_ASIC_DB, analogous to syncd for ASIC_DB.

---

## 2. Port Lifecycle: NPU Port vs Gearbox Port

### 2.1 Creation Order

NPU port is created first, then gearbox port. Call chain in `portsorch.cpp`:

```
doPortTask (line 4301)
  -> addPortBulk (line 4490)       -- creates NPU port via sai_port_api->create_port(gSwitchId)
  -> initPortsBulk (line 4495)
    -> initializePorts (line 3965)  -- host interfaces, queues
    -> registerPort (line 3974)
      -> initGearboxPort (line 4004) -- creates gearbox ports via sai_port_api->create_port(phyOid)
```

### 2.2 Gearbox Port Objects Created

Per interface, `initGearboxPort()` (`portsorch.cpp:9718-10017`) creates three objects in GB_ASIC_DB:
- **System-side port** (`sai_port_api->create_port(&systemPort, phyOid, ...)` — line 9816)
- **Line-side port** (`sai_port_api->create_port(&linePort, phyOid, ...)` — line 9924)
- **Port connector** (`sai_port_api->create_port_connector(&connector, phyOid, ...)` — line 9948)

### 2.3 Dependency Direction

- **NPU port → Gearbox port**: One-way. NPU port must exist first (gearbox init happens during `registerPort` after NPU port has `m_port_id`).
- **Gearbox port → NPU port**: No SAI-level reference. The gearbox ports are created independently within the PHY context.
- **Correlation is administrative only**: orchagent links them via `Port.m_system_side_id`, `Port.m_line_side_id`, and `m_gearboxPortListLaneMap[npu_port_id] = (system_oid, line_oid)`.

### 2.4 Deletion: Gearbox Ports Are NOT Cleaned Up

The port DEL handler (`portsorch.cpp:5363-5449`) flow:

```
1. Check ref_count > 0?           -> retry (keep in queue)
2. Check bridge_port != NULL?      -> retry (keep in queue)
3. deInitPort() + remove hostif
4. unsetPortPtTam()
5. removePort(port_id)             -> SAI remove NPU port only
6. Clean up internal maps (m_portList, m_portConfigMap, saiOidToAlias)
```

**Missing**: No `deinitGearboxPort()` — system-side port, line-side port, port_connector, and serdes objects in GB_ASIC_DB are **never removed**. The `m_gearboxPortListLaneMap` entry keyed by the old NPU port OID is also never erased.

**Note**: Gearbox port existence is **NOT** a ref count blocker for NPU port deletion.

**Consequence**: On gearbox platforms, every port breakout cycle **leaks gearbox SAI objects** in GB_ASIC_DB.

---

## 3. Sync Mode: Blocking Behavior

Orchagent is **single-threaded**. All SAI calls (to both syncd and gbsyncd) are synchronous and sequential on the main orch loop thread.

**Impact**: If a call to gbsyncd is stuck or slow, orchagent is blocked. No other work proceeds — including pending requests to syncd, heartbeat emission, and all other orch task processing.

---

## 4. SAI Call Timeout & Error Handling

### 4.1 Timeout Mechanism

`RedisChannel::wait()` / `ZeroMQChannel::wait()` has a **60-second default timeout** (`SAI_REDIS_DEFAULT_SYNC_OPERATION_RESPONSE_TIMEOUT`), configurable via `SAI_REDIS_SWITCH_ATTR_SYNC_OPERATION_RESPONSE_TIMEOUT`.

On timeout:
```
RedisChannel:   SWSS_LOG_ERROR("SELECT operation result: TIMEOUT on <command>")
                SWSS_LOG_ERROR("failed to get response for <command>")
                -> returns SAI_STATUS_FAILURE

ZeroMQChannel:  SWSS_LOG_ERROR("zmq_poll timed out for: <command>")
                -> returns SAI_STATUS_FAILURE
```

### 4.2 Retry Policy by SAI Status

**For remove operations** (`handleSaiRemoveStatus` in `saihelper.cpp:652-693`):

| SAI Status | Result | Retry? |
|---|---|---|
| `SAI_STATUS_SUCCESS` | `task_success` | No |
| `SAI_STATUS_ITEM_NOT_FOUND` | `task_success` | No (treated as already gone) |
| `SAI_STATUS_OBJECT_IN_USE` | `task_need_retry` | **Yes** (no retry limit) |
| `SAI_STATUS_FAILURE` (timeout) | `task_failed` | **No** — task dropped |
| Other errors | `task_failed` | **No** — task dropped |

**For create operations** (`handleSaiCreateStatus` in `saihelper.cpp:579-603`):

| SAI Status | Result | Retry? |
|---|---|---|
| `SAI_STATUS_SUCCESS` | `task_success` | No |
| `SAI_STATUS_ITEM_ALREADY_EXISTS` | `task_success` | No |
| `SAI_STATUS_INSUFFICIENT_RESOURCES` / `TABLE_FULL` / `NO_MEMORY` | `task_need_retry` | **Yes** |
| `SAI_STATUS_FAILURE` (timeout) | `task_failed` | **No** — task dropped |

### 4.3 Two-Layer Retry for Port Deletion

Port deletion has two layers of retry, both with **no retry limit** (indefinite):

**Layer 1: Orchagent pre-checks (before SAI call)** — `portsorch.cpp:5376-5397`

Orchagent checks internal state before even calling SAI. If conditions aren't met, the task stays in `m_toSync` for the next orch loop iteration (~1s):

- `m_port_ref_count[alias] > 0`:
  - `SWSS_LOG_WARN("Unable to remove port %s: ref count %u", ...)`
  - Retry next cycle (other orchs like sub-interface/LAG still hold a reference)
- `bridge_port_oid != SAI_NULL_OBJECT_ID`:
  - `SWSS_LOG_WARN("Cannot remove port as bridge port OID is present %" PRIx64, ...)`
  - Retry next cycle (port still in a VLAN; VLAN member removal hasn't completed)

**Layer 2: SAI returns OBJECT_IN_USE (after SAI call)** — `portsorch.cpp:5429-5438`

If the pre-checks pass but the vendor SAI still has references (e.g., FDB entries, mirror sessions, routes):

```cpp
sai_status_t status = removePort(port_id);
if (SAI_STATUS_SUCCESS != status)
{
    if (SAI_STATUS_OBJECT_IN_USE != status)
    {
        throw runtime_error("Delete port failed");  // fatal for other errors
    }
    SWSS_LOG_WARN("Failed to remove port %" PRIx64 ", as the object is in use", port_id);
    it++;      // keep in queue
    continue;  // retry next cycle
}
```

Any **other** SAI error (including `SAI_STATUS_FAILURE` from timeout) on port removal triggers `throw runtime_error("Delete port failed")` → unhandled exception → orchagent crash → container restart.

Note: `removePort()` (`portsorch.cpp:3833-3881`) returns the raw `sai_status_t` from `sai_port_api->remove_port()` directly — it does **not** go through `handleSaiRemoveStatus`. The caller's only special case is `OBJECT_IN_USE`.

Additionally, `removePort()` calls `setPortAdminStatus(port, false)` (line 3846) **before** the actual `remove_port`. If that SAI call also times out, there could be **two** consecutive 60s timeouts (admin-down + remove) totaling ~120s of blocking before the crash.

This crash path currently only applies to **NPU port removal** (syncd), since gearbox ports are never removed today. If a `deinitGearboxPort()` were added with similar throw-on-failure logic, gbsyncd timeouts would also trigger crashes.

### 4.4 handleSaiFailure Behavior

`saihelper.cpp:747-779` — triggered on `SAI_STATUS_FAILURE` (timeout) for non-port-removal SAI calls:

1. Sets `gOrchUnhealthy = true`
2. Logs: `"Encountered failure in <op> operation, SAI API: <api>, status: SAI_STATUS_FAILURE"`
3. Publishes structured event: `"sai-operation-failure"`
4. Triggers syncd dump via `SAI_REDIS_NOTIFY_SYNCD_INVOKE_DUMP`
5. Does **NOT** abort (`abort_on_failure = false` for runtime operations)

### 4.5 initGearboxPort Return Value Ignored

`registerPort()` at line 4004:
```cpp
initGearboxPort(p);   // return value NOT checked
```

If gearbox port creation times out, orchagent proceeds as if the port is fine.

---

## 5. Debugging Mechanisms for Slow/Stuck SAI Calls

| Mechanism | Location | Default Threshold | What It Does |
|---|---|---|---|
| **sairedis.rec timestamps** | `sonic-sairedis/lib/Recorder.cpp` | Always on | Microsecond-precision timestamps on every SAI call. Gap between request (lowercase `c`/`s`/`r`) and response (uppercase `C`/`S`/`R`) shows syncd/gbsyncd latency. Correlate by switch OID. |
| **TimerWatchdog** | `sonic-sairedis/syncd/TimerWatchdog.cpp` | **30 seconds** (syncd `-w` flag) | Background thread in syncd/gbsyncd. Logs `"time span WD exceeded %ld ms for %s"` if call still running; `"event '%s' took %ld ms to execute"` on completion. Fires in both syncd and gbsyncd (shared binary). |
| **Sync mode response timeout** | `sonic-sairedis/lib/RedisChannel.cpp` | **60 seconds** | `SAI_REDIS_DEFAULT_SYNC_OPERATION_RESPONSE_TIMEOUT`. Configurable via `SAI_REDIS_SWITCH_ATTR_SYNC_OPERATION_RESPONSE_TIMEOUT`. VOQ: 5x default; Fabric: 10x default. |
| **Orchagent heartbeat** | `orchagent/orchdaemon.cpp:1213` | **10 seconds** (`-I` flag) | Emits `<!--XSUPERVISOR:BEGIN-->heartbeat<!--XSUPERVISOR:END-->` to stdout. Read from `CONFIG_DB HEARTBEAT\|orchagent`. |
| **PerformanceIntervalTimer** | `sonic-sairedis/meta/PerformanceIntervalTimer.cpp` | Per 10,000 ops | Tracks cumulative timing for bulk operations. |

---

## 6. Supervisord Monitoring

### 6.1 Configuration

- `sonic-buildimage/dockers/docker-orchagent/supervisord.conf.j2`: orchagent configured with `stdout_capture_maxbytes=1MB` (enables PROCESS_COMMUNICATION_STDOUT events)
- `sonic-buildimage/dockers/docker-orchagent/watchdog_processes.j2`: lists `program:orchagent`
- Event listener: `supervisor-proc-exit-listener-rs` listens for `PROCESS_STATE_EXITED`, `PROCESS_STATE_RUNNING`, `PROCESS_COMMUNICATION_STDOUT`
- Alert threshold: **60 seconds** default (`ALERTING_INTERVAL_SECS = 60`), configurable via `CONFIG_DB HEARTBEAT|orchagent alert_interval` (in milliseconds)

### 6.2 Behavior on Missed Heartbeats

**Alerting only — NO kill:**
- After 60s of no heartbeat: `WARNING: "Process 'orchagent' is stuck in namespace '...' (N minutes)."`
- Warning repeats periodically
- Orchagent is **not killed or restarted**

### 6.3 What Actually Kills Orchagent

| Condition | Kill mechanism | Container restart? |
|---|---|---|
| Heartbeat missed (stuck on SAI) | **No kill** — syslog warning only | No |
| SAI timeout (60s) | **No kill** — task dropped, `gOrchUnhealthy=true` | No |
| `abort_on_failure=true` (init-time critical) | `abort()` in `handleSaiFailure` | Yes |
| `throw runtime_error("Delete port failed")` (non-OBJECT_IN_USE SAI error on port delete) | Unhandled exception → crash | Yes |
| Unhandled exception / segfault | Process exits | Yes |
| OOM killer | Kernel kills process | Yes |

When orchagent **exits** (any reason), the `supervisor-proc-exit-listener` detects `PROCESS_STATE_EXITED` and sends `SIGTERM` to the supervisord parent PID → kills the entire swss container → systemd restarts it.

---

## 7. End-to-End Stuck/Timeout Scenario Timeline

```
T+0s     Orchagent makes sync SAI call (e.g., create_port to gbsyncd)
         Main thread blocks on RedisChannel::wait()
         All other orch processing frozen (routes, neighbors, ACLs, syncd tasks)
         heartBeat() unreachable

T+10s    First missed heartbeat emission

T+30s    syncd/gbsyncd TimerWatchdog fires (default 30s threshold):
         ERROR: "time span WD exceeded 30000 ms for <SAI_API_call>"
         (This means the vendor SAI call is STILL running on the syncd/gbsyncd side)
         If the call eventually completes after this warning:
         ERROR: "event '<SAI_API_call>' took <N> ms to execute"
         (These messages appear in syncd/gbsyncd syslog, not orchagent syslog)

T+60s    supervisor-proc-exit-listener:
         WARNING: "Process 'orchagent' is stuck in namespace '...' (1 minutes)."
         (No kill — alerting only)

T+60s    RedisChannel::wait() timeout fires:
         ERROR: "SELECT operation result: TIMEOUT on <command>"
         ERROR: "failed to get response for <command>"
         -> SAI_STATUS_FAILURE returned

         handleSaiFailure():
         ERROR: "Encountered failure in <op> operation, SAI API: <api>, status: SAI_STATUS_FAILURE"
         -> gOrchUnhealthy = true
         -> event "sai-operation-failure" published
         -> syncd dump triggered

         Task processing (depends on operation type):
         -> General SAI calls: task_failed -> task ERASED from m_toSync (gone forever)
         -> Port removal specifically: throw runtime_error("Delete port failed") -> crash
            -> PROCESS_STATE_EXITED -> SIGTERM to supervisord -> container restart

T+60s+   If orchagent did NOT crash:
         Orchagent resumes main loop
         heartBeat() resumes -> stuck warning eventually clears
         Processes queued tasks again
         But: gOrchUnhealthy=true, failed task lost, potential orphaned/missing objects
```

---

## 8. Gearbox-Specific Timeout Consequences

### Full gearbox creation timeout

- `initGearboxPort` return value ignored (`portsorch.cpp:4004`)
- `m_system_side_id = 0`, `m_line_side_id = 0` on Port object
- NPU port appears operational; gearbox PHY has no path configured
- **Traffic blackhole**: packets reach NPU but have no PHY-level forwarding through gearbox
- Flex counter setup safely skipped (checks `if (p.m_system_side_id)`)

### Partial gearbox creation timeout (system-side OK, line-side timeout)

- `port.m_system_side_id` set, `port.m_line_side_id = 0`
- System-side port exists in GB_ASIC_DB; line-side doesn't
- Port connector creation skipped (needs both)
- **Half-configured gearbox PHY**, no rollback of system-side port
- Admin-state/FEC/speed changes via `setGearboxPortsAttr` apply to system-side only, silently skip line-side

### Late completion by gbsyncd (after orchagent timeout)

- gbsyncd may complete the operation after 60s timeout
- Orchagent never received the OID → gearbox port is an **orphan** in GB_ASIC_DB
- Subsequent operations not applied to orphaned gearbox port
- On port deletion/breakout: orphaned objects never cleaned up

---

## 9. `config interface breakout` CLI — GB_ASIC_DB Verification Gap

### Current verification (`sonic-utilities/config/config_mgmt.py`)

The `_verifyAsicDB()` method (line 381) waits up to 60 seconds for port deletion from **ASIC_DB only**:

```python
self.oidKey = 'ASIC_STATE:SAI_OBJECT_TYPE_PORT:oid:0x'  # line 322

def _checkNoPortsInAsicDb(self, db, ports, portMap):
    db.connect(db.ASIC_DB)            # Only ASIC_DB, never GB_ASIC_DB
    for port in ports:
        key = self.oidKey + portMap[port]
        if self._checkKeyinAsicDB(key, db):
            return False
    return True
```

### Gap

- **GB_ASIC_DB is never checked** during breakout verification
- CLI reports deletion complete when NPU port is gone from ASIC_DB
- Gearbox entries (system-side port, line-side port, port_connector, serdes) in GB_ASIC_DB are not verified
- Combined with the missing `deinitGearboxPort()`, gearbox objects are **never deleted** during breakout

### Risk assessment

In **sync mode**: orchagent won't process new port config until old gearbox deletion completes (single-threaded), so ordering is safe if gearbox deletion were implemented. The CLI timeout is misleading but not a correctness bug.

In any future **async/pipeline mode**: this would be a real race condition.

---

## 10. Summary of Identified Gaps

| Issue | Impact | Location |
|---|---|---|
| No `deinitGearboxPort()` in port deletion | Gearbox SAI objects leaked in GB_ASIC_DB on every breakout | `portsorch.cpp:5363-5449` |
| `initGearboxPort()` return value ignored | Silent failure, traffic blackhole on gearbox timeout | `portsorch.cpp:4004` |
| No rollback on partial gearbox creation | Half-configured PHY, leaked system-side port | `portsorch.cpp:9718-10017` |
| `m_gearboxPortListLaneMap` not cleaned on port delete | Memory leak | `portsorch.cpp:5440-5446` |
| Breakout CLI doesn't verify GB_ASIC_DB | False "deletion complete" signal on gearbox platforms | `config_mgmt.py:355-416` |
| Stuck orchagent not killed by supervisord | Orchagent limps along unhealthy indefinitely | `supervisor-proc-exit-listener` |
| SAI timeout → task dropped, no retry | Lost configuration requiring manual re-push | `saihelper.cpp:579-603` |
| Port removal SAI timeout → crash | `throw runtime_error` on non-OBJECT_IN_USE error kills orchagent | `portsorch.cpp:5432-5434` |
