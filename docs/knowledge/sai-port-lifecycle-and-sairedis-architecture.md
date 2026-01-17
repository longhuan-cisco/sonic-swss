# SAI Port Lifecycle and SAI Redis Architecture

## Overview

This document describes the lifecycle of ASIC_STATE:SAI_OBJECT_TYPE_PORT entries and the underlying SAI Redis communication architecture in SONiC.

---

## 1. Who Creates/Deletes ASIC_STATE:SAI_OBJECT_TYPE_PORT Entries

### Primary Owner
- **Component:** `PortsOrch` (orchagent)
- **File:** `/home/user/sonic-swss/orchagent/portsorch.cpp`
- **Global Instance:** `gPortsOrch` created in `orchdaemon.cpp:208`

### Infrastructure Chain
```
orchagent (PortsOrch) → SAI Redis Library → syncd → ASIC_STATE DB
```

The `syncd` daemon is what actually writes to the ASIC_STATE database after receiving SAI API calls translated by libsairedis.

---

## 2. Port Creation

### Bulk Port Creation (Primary Method)
- **Entry Point:** `PortsOrch::doPortTask()` at `portsorch.cpp:4234`
- **Flow:**
  1. Receives `PORT_CONFIG_DONE` notification from portsyncd
  2. Processes port configuration from APPLICATION_DB (PORT_TABLE)
  3. Calls `PortsOrch::addPortBulk()` at `portsorch.cpp:1152-1408`
  4. Invokes SAI API: `sai_port_api->create_ports()` at `portsorch.cpp:1349`

### Key Attributes Set During Creation (lines 1152-1347)
- `SAI_PORT_ATTR_HW_LANE_LIST` - Hardware lane configuration
- `SAI_PORT_ATTR_SPEED` - Port speed in Mbps
- `SAI_PORT_ATTR_AUTO_NEG_MODE` - Auto-negotiation
- `SAI_PORT_ATTR_FEC_MODE` - Forward Error Correction
- `SAI_PORT_ATTR_TPID` - Tag Protocol Identifier

### Single Port Creation (Gearbox/Special Ports)
- `PortsOrch::gearboxPortCreate()` at `portsorch.cpp:9683, 9791`
- Uses `sai_port_api->create_port()` single variant

---

## 3. Port Deletion

### Bulk Port Deletion
- **Entry Point:** `PortsOrch::doPortTask()` at `portsorch.cpp:4234`
- **Flow:**
  1. Compares configured ports with currently created ports
  2. Calls `PortsOrch::removePortBulk()` at `portsorch.cpp:1410-1496`
  3. Pre-deletion cleanup (lines 1419-1448):
     - Sets admin status to DOWN
     - Removes port serdes attributes
     - Decrements TAM reference counts
  4. Invokes SAI API: `sai_port_api->remove_ports()` at `portsorch.cpp:1453`

### Single Port Deletion
- `PortsOrch::removePort()` at `portsorch.cpp:3804-3852`
- Uses `sai_port_api->remove_port(port_id)` at line 3841

---

## 4. Downstream Flow After SAI API Calls

### High-Level Architecture
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              ORCHAGENT                                       │
│  SAI API call (e.g., sai_port_api->create_ports())                          │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         SAIREDIS LIBRARY                                     │
│  (libsairedis.so - transparent abstraction layer)                           │
│  - Serializes SAI API calls to Redis commands                               │
│  - Manages communication mode (async/sync/zmq)                              │
│  - Generates VID (Virtual ID) for new objects                               │
│  - Writes CREATE COMMAND to request channel (NOT final ASIC_STATE entry)    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ASIC_DB (Redis DB 1)                              │
│                                                                             │
│  REQUEST Channel/Key (written by sairedis, consumed by syncd):              │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Command: CREATE                                                     │   │
│  │  Object Type: SAI_OBJECT_TYPE_PORT                                   │   │
│  │  VID: 0x1000000000003                                                │   │
│  │  Attributes: lanes, speed, fec, etc.                                 │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                              SYNCD                                           │
│  (syncd process - SAI daemon)                                               │
│  - Consumes commands from ASIC_DB request channel                           │
│  - Translates to vendor-specific SAI implementation                         │
│  - Maintains VID ↔ RID mapping                                              │
│  - Programs actual ASIC hardware                                            │
│  - Writes FINAL ASIC_STATE entry after successful creation                  │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                         VENDOR SAI LIBRARY                                   │
│  (libsai.so - ASIC vendor implementation)                                   │
│  - Broadcom, Mellanox, etc.                                                 │
│  - Returns RID (Real ID) from actual hardware                               │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                            ASIC HARDWARE                                     │
│  (Memory, TCAM, forwarding tables, etc.)                                    │
└─────────────────────────────────────────────────────────────────────────────┘
                                    │
                                    ▼
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ASIC_DB - FINAL STATE (written by syncd)                  │
│                                                                             │
│  ASIC_STATE Entry (after syncd processes and HW succeeds):                  │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  Key: ASIC_STATE:SAI_OBJECT_TYPE_PORT|oid:0x1000000000003           │   │
│  │    SAI_PORT_ATTR_HW_LANE_LIST: 1,2,3,4                               │   │
│  │    SAI_PORT_ATTR_SPEED: 100000                                       │   │
│  │    SAI_PORT_ATTR_OPER_STATUS: SAI_PORT_OPER_STATUS_UP                │   │
│  │    ...                                                               │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
│  VID↔RID Mapping (internal to syncd, persisted for warm reboot):            │
│  ┌─────────────────────────────────────────────────────────────────────┐   │
│  │  VIDTORID|oid:0x1000000000003 → oid:0x<actual_hardware_oid>         │   │
│  │  RIDTOVID|oid:0x<actual_hardware_oid> → oid:0x1000000000003         │   │
│  └─────────────────────────────────────────────────────────────────────┘   │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Clarification: Request vs Final State

**Q: Are the sairedis write and syncd write acting on the same DB table entry?**

**A: No, they are different operations:**

| Step | Component | What it writes | Purpose |
|------|-----------|----------------|---------|
| Request | sairedis | CREATE command to request channel/key | IPC to syncd |
| Final State | syncd | ASIC_STATE:SAI_OBJECT_TYPE_PORT entry | Persistent state after HW success |

The request channel is consumed by syncd and is transient. The ASIC_STATE entry is the final, persistent representation of the object after it has been successfully created in hardware.

---

## 5. VID vs RID Architecture

### Definitions
| Term | Full Name | Description |
|------|-----------|-------------|
| **VID** | Virtual ID | Generated by sairedis library, used by orchagent |
| **RID** | Real ID | Actual hardware OID from vendor SAI/ASIC |

### VID/RID Flow
```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         VID/RID Lifecycle                                    │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  ORCHAGENT (sairedis library)                 SYNCD                         │
│  ─────────────────────────────                ─────                         │
│                                                                             │
│  1. sai_port_api->create_port()                                             │
│          │                                                                  │
│          ▼                                                                  │
│  2. sairedis generates VID locally ◄─── VID created HERE                    │
│          │                                                                  │
│          ▼                                                                  │
│  3. Request sent: "create port, VID=0x1000000000003"                        │
│          │                                                                  │
│          ├─────────────────────────────────►  4. syncd receives request     │
│          │                                            │                     │
│          │                                            ▼                     │
│  5. Returns VID to caller                     6. Calls ASIC SDK             │
│     (ASYNC: immediate)                                │                     │
│     (SYNC: waits)                                     ▼                     │
│          │                                    7. Gets RID from hardware     │
│          │                                            │                     │
│          │                                            ▼                     │
│          │                                    8. Maps VID ↔ RID             │
│          │                                            │                     │
│          │                                            ▼                     │
│          │◄───────────────────────────────────9. Response: STATUS           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Key Points
- VID is generated at **request time** by sairedis library (in orchagent process)
- RID is created when **syncd calls ASIC SDK**
- Orchagent **never sees RID** - only works with VID
- VID↔RID mapping maintained by syncd (in memory and/or ASIC_DB for warm reboot)

---

## 6. Communication Modes

Configured in `main.cpp:58` and `saihelper.cpp:334-337`:

| Mode | Flag | Behavior |
|------|------|----------|
| **REDIS_ASYNC** (default) | `-z redis_async` | Non-blocking, uses pipelining, returns immediately |
| **REDIS_SYNC** | `-z redis_sync` | Blocking, waits for syncd/ASIC response |
| **ZMQ_SYNC** | `-z zmq_sync` | Uses ZeroMQ instead of Redis |

### ASYNC Mode Behavior
```
sai_port_api->create_ports() returns immediately
            │
            ▼
    ┌───────────────────────────────────────┐
    │  Only means: sairedis QUEUED the      │
    │  request in Redis pipeline            │
    │                                        │
    │  syncd has NOT processed it yet!      │
    │  ASIC SDK has NOT been called yet!    │
    └───────────────────────────────────────┘

Actual processing happens at flush():
    OrchDaemon::flush() → SAI_REDIS_SWITCH_ATTR_FLUSH
```

### SYNC Mode Behavior
```
sai_port_api->create_ports() blocks until complete
            │
            ▼
    ┌───────────────────────────────────────┐
    │  Means: Full round-trip complete      │
    │                                        │
    │  ✓ syncd received request             │
    │  ✓ ASIC SDK processed it              │
    │  ✓ Response returned                  │
    └───────────────────────────────────────┘
```

---

## 7. Status Return Value Meaning

### What Immediate Status Means by Mode

| Mode | `SAI_STATUS_SUCCESS` means | Real ASIC errors caught? |
|------|---------------------------|--------------------------|
| **ASYNC** | sairedis validated & queued request | NO - only at flush() |
| **SYNC** | Full round-trip success | YES |

### Errors Caught Immediately in ASYNC Mode

| Error Type | Caught Immediately? |
|------------|---------------------|
| Invalid parameter (NULL pointer) | ✓ Yes |
| Invalid OID format | ✓ Yes |
| Serialization failure | ✓ Yes |
| Memory allocation failure | ✓ Yes |
| **ASIC doesn't support feature** | ✗ NO |
| **Hardware resource exhausted** | ✗ NO |
| **Invalid attribute for HW** | ✗ NO |

---

## 8. sairedis.rec Log File

### Configuration
- **Enabled via:** `SAI_REDIS_SWITCH_ATTR_RECORD` (boolean)
- **Output directory:** `SAI_REDIS_SWITCH_ATTR_RECORDING_OUTPUT_DIR`
- **Filename:** `SAI_REDIS_SWITCH_ATTR_RECORDING_FILENAME`
- **Default:** `sairedis.rec` in current directory
- **Written by:** syncd (not orchagent)

### Contents
Records **both requests AND responses** (full transaction log):

```
# REQUEST - VID already known (generated by sairedis client)
timestamp|c|SAI_OBJECT_TYPE_PORT|oid:0x1000000000003|attrs...

# RESPONSE - Same VID, plus STATUS
timestamp|C|SAI_STATUS_SUCCESS|oid:0x1000000000003
```

| Code | Meaning |
|------|---------|
| `c` | Create request |
| `C` | Create response |
| `s` | Set request |
| `S` | Set response |
| `r` | Remove request |
| `R` | Remove response |
| `g` | Get request |
| `G` | Get response |

### OID in sairedis.rec
- Contains **VID** (Virtual ID), not RID
- VID is generated at request time by sairedis client
- Response contains same VID for correlation, plus status

---

## 9. Error Handling and Dump Mechanism

### handleSaiFailure() Function
**File:** `saihelper.cpp:744-770`

```cpp
void handleSaiFailure(sai_api_t api, string oper, sai_status_t status)
{
    // 1. Log the error
    SWSS_LOG_ERROR("Encountered failure in %s operation...");

    // 2. Publish structured syslog event
    event_publish(g_events_handle, "sai-operation-failure", &params);

    // 3. Request syncd to invoke dump
    attr.id = SAI_REDIS_SWITCH_ATTR_NOTIFY_SYNCD;
    attr.value.s32 = SAI_REDIS_NOTIFY_SYNCD_INVOKE_DUMP;
    sai_switch_api->set_switch_attribute(gSwitchId, &attr);
}
```

### When Dump is Triggered

| Status | Action |
|--------|--------|
| `SAI_STATUS_SUCCESS` | task_success (no dump) |
| `SAI_STATUS_ITEM_ALREADY_EXISTS` | task_success (no dump) |
| `SAI_STATUS_INSUFFICIENT_RESOURCES` | task_need_retry (no dump) |
| `SAI_STATUS_TABLE_FULL` | task_need_retry (no dump) |
| Other errors | `handleSaiFailure()` → **DUMP** |
| GET operation failures | Logged only (no dump to avoid overwhelming) |

### ASYNC Mode Limitation

```
SYNC Mode:
  orchagent SAI call → syncd → ASIC error → returned to orchagent → DUMP ✓

ASYNC Mode (default):
  orchagent SAI call → returns SUCCESS (queued)
  (later) syncd → ASIC error → logged by syncd only → NO DUMP from orchagent ✗
```

---

## 10. Key Source File References

| Component | File | Lines | Description |
|-----------|------|-------|-------------|
| sai_port_api global | saihelper.cpp | 41 | Port API pointer declaration |
| sai_port_api init | saihelper.cpp | 199 | Port API initialization |
| gSwitchId | main.cpp | 47 | Global switch OID |
| PortsOrch creation | orchdaemon.cpp | 208 | Global PortsOrch instance |
| addPortBulk | portsorch.cpp | 1152-1408 | Bulk port creation |
| create_ports SAI call | portsorch.cpp | 1349-1353 | SAI port creation |
| removePortBulk | portsorch.cpp | 1410-1496 | Bulk port removal |
| remove_ports SAI call | portsorch.cpp | 1453-1457 | SAI port removal |
| removePort | portsorch.cpp | 3804-3852 | Single port removal |
| doPortTask | portsorch.cpp | 4234-4470 | Port task orchestration |
| initSaiRedis | saihelper.cpp | 313-448 | SAI Redis initialization |
| handleSaiFailure | saihelper.cpp | 744-770 | Error handling with dump |
| OrchDaemon::flush | orchdaemon.cpp | 855-888 | Pipeline flush |

---

## 11. Databases Involved

| Database | Purpose |
|----------|---------|
| **CONFIG_DB** | User configuration (from CLI/config files) |
| **APPL_DB** | Application-level intent (processed by *syncd daemons) |
| **ASIC_DB** | SAI object state & IPC with syncd |
| **STATE_DB** | Operational state (oper_status, etc.) |
| **COUNTERS_DB** | Port/queue statistics |
| **FLEX_COUNTER_DB** | Flexible counter configuration |

---

## 12. Notification Flow (ASIC → Orchagent)

Events flow back from the ASIC through syncd to orchagent:

```
ASIC Hardware
    │ (interrupt/event)
    ▼
SYNCD
    │ (publishes to)
    ▼
ASIC_DB:NOTIFICATIONS channel
    │ (consumed by)
    ▼
NotificationConsumer in orchagent
```

### Key Notification Consumers

| Component | File:Line | Notifications |
|-----------|-----------|---------------|
| PortsOrch | portsorch.cpp:981-988 | Port state changes |
| FdbOrch | fdborch.cpp:40-48 | FDB learn/age/flush events |
| BfdOrch | bfdorch.cpp:63 | BFD session state changes |

---

## Summary

1. **Port objects** are created/deleted by `PortsOrch` via SAI API calls
2. **sairedis library** generates VID at request time and handles orchagent↔syncd communication
3. **syncd** processes requests, calls vendor SAI, and maintains VID↔RID mapping
4. **ASYNC mode** (default) queues requests - immediate return doesn't mean ASIC processed it
5. **sairedis.rec** contains both requests and responses with VIDs
6. **Error handling** in ASYNC mode is limited - real ASIC errors may not trigger dumps from orchagent
