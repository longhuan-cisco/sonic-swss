# SONiC SWSS Operational Status Down Flow

This document explains the operational status (oper status) down flow in SONiC SWSS, covering both hardware-initiated (link failure) and user-initiated (admin shutdown) scenarios.

## Table of Contents
1. [Overview](#overview)
2. [Key Data Structures](#key-data-structures)
3. [Path 1: Oper Status Down Due to Link Failure](#path-1-oper-status-down-due-to-link-failure-hardware-initiated)
4. [Path 2: Oper Status Down Due to User Shutdown](#path-2-oper-status-down-due-to-user-shutdown-admin-initiated)
5. [Central Orchestration Function: updatePortOperStatus](#central-orchestration-function-updateportoperstatus)
6. [Downstream Components Affected](#downstream-components-affected)
7. [Sequence Diagrams](#sequence-diagrams)
8. [Key Code References](#key-code-references)

---

## Overview

There are two distinct paths that lead to a port's operational status going down:

| Trigger | Origin | Flow Path |
|---------|--------|-----------|
| Link Failure | Hardware/PHY layer detects loss of signal | SAI SDK -> syncd -> ASIC_DB NOTIFICATIONS -> orchagent |
| Admin Shutdown | User configuration (CONFIG_DB) | CONFIG_DB -> portsorch -> SAI API -> SAI generates notification -> orchagent |

Both paths ultimately converge at the `updatePortOperStatus()` function in portsorch.cpp, which handles all downstream updates.

---

## Key Data Structures

### Port Structure (orchagent/port.h)

```cpp
class Port {
    sai_port_oper_status_t m_oper_status;   // Operational status (UP/DOWN/UNKNOWN/TESTING/NOT_PRESENT)
    bool m_admin_state_up;                   // Administrative status (user-configurable)
    uint32_t m_oper_error_status;            // Bitmap of error conditions
    uint32_t m_flap_count;                   // Tracks interface state changes
    // ...
};
```

### PortOperStateUpdate Structure (orchagent/portsorch.h:92)

```cpp
struct PortOperStateUpdate
{
    Port port;
    sai_port_oper_status_t operStatus;
};
```

This structure is used to notify observers about port operational state changes.

---

## Path 1: Oper Status Down Due to Link Failure (Hardware-Initiated)

This flow is triggered when the physical link fails (e.g., cable unplugged, transceiver failure, remote side down).

### Flow Diagram

```
Physical Link Failure
         |
         v
+-------------------+
|   SAI/SDK Layer   |  Hardware detects link loss
+-------------------+
         |
         v (callback registered in main.cpp:554-555)
+-------------------+
| on_port_state_    |  notifications.cpp:29
| change()          |
+-------------------+
         |
         v (sends to ASIC_DB NOTIFICATIONS channel)
+-------------------+
| NotificationProducer |  notifications.cpp:33-39
+-------------------+
         |
         v
+-------------------+
| ASIC_DB           |  Redis pub/sub channel "NOTIFICATIONS"
| NOTIFICATIONS     |
+-------------------+
         |
         v (NotificationConsumer in portsorch)
+-------------------+
| PortsOrch::       |  portsorch.cpp:8879-8901
| doTask()          |
+-------------------+
         |
         v
+-------------------+
| PortsOrch::       |  portsorch.cpp:8903-8969
| handleNotification|
+-------------------+
         |
         v
+-------------------+
| PortsOrch::       |  portsorch.cpp:9041-9114
| updatePortOperStatus|
+-------------------+
         |
         v
   [Downstream Updates]
```

### Key Code Path

1. **SAI Notification Registration** (`main.cpp:554-555`)
   ```cpp
   attr.id = SAI_SWITCH_ATTR_PORT_STATE_CHANGE_NOTIFY;
   attr.value.ptr = (void *)on_port_state_change;
   ```

2. **Notification Callback** (`notifications.cpp:29-39`)
   ```cpp
   void on_port_state_change(uint32_t count, sai_port_oper_status_notification_t *data)
   {
       swss::NotificationProducer port_state_change(&db, "NOTIFICATIONS");
       std::string sdata = sai_serialize_port_oper_status_ntf(count, data);
       port_state_change.send("port_state_change", sdata, values);
   }
   ```

3. **NotificationConsumer Processing** (`portsorch.cpp:8879-8901`)
   ```cpp
   void PortsOrch::doTask(NotificationConsumer &consumer)
   {
       std::deque<KeyOpFieldsValuesTuple> entries;
       consumer.pops(entries);
       for (auto& entry : entries) {
           handleNotification(consumer, entry);
       }
   }
   ```

4. **Notification Handling** (`portsorch.cpp:8909-8933`)
   ```cpp
   if (&consumer == m_portStatusNotificationConsumer && op == "port_state_change")
   {
       sai_deserialize_port_oper_status_ntf(data, count, &portoperstatus);
       for (uint32_t i = 0; i < count; i++) {
           sai_port_oper_status_t status = portoperstatus[i].port_state;
           updatePortOperStatus(port, status);
       }
   }
   ```

---

## Path 2: Oper Status Down Due to User Shutdown (Admin-Initiated)

This flow is triggered when a user administratively shuts down an interface (e.g., `config interface shutdown Ethernet0`).

### Flow Diagram

```
User Command: config interface shutdown Ethernet0
         |
         v
+-------------------+
| CONFIG_DB         |  PORT table updated: admin_status = down
+-------------------+
         |
         v (SubscriberStateTable)
+-------------------+
| PortsOrch::       |  portsorch.cpp:4156 onwards
| doPortTask()      |
+-------------------+
         |
         v (when admin_status field is processed)
+-------------------+
| PortsOrch::       |  portsorch.cpp:5156-5170
| setPortAdminStatus|
+-------------------+
         |
         v
+-------------------+
| sai_port_api->    |  portsorch.cpp:2125
| set_port_attribute|  (SAI_PORT_ATTR_ADMIN_STATE = false)
+-------------------+
         |
         v (SAI/SDK brings port down, generates notification)
+-------------------+
| SAI SDK           |  Hardware executes admin down
+-------------------+
         |
         v (flows back through Path 1)
   [Notification flow from SAI -> orchagent]
         |
         v
+-------------------+
| updatePortOperStatus|  portsorch.cpp:9041-9114
+-------------------+
```

### Key Code Path

1. **CONFIG_DB Change Processing** (`portsorch.cpp:5156-5170`)
   ```cpp
   if (pCfg.admin_status.is_set)
   {
       if (p.m_admin_state_up != pCfg.admin_status.value)
       {
           if (!setPortAdminStatus(p, pCfg.admin_status.value))
           {
               SWSS_LOG_ERROR("Failed to set port %s admin status...");
           }
           p.m_admin_state_up = pCfg.admin_status.value;
       }
   }
   ```

2. **Set Port Admin Status** (`portsorch.cpp:2106-2160`)
   ```cpp
   bool PortsOrch::setPortAdminStatus(Port &port, bool state)
   {
       sai_attribute_t attr;
       attr.id = SAI_PORT_ATTR_ADMIN_STATE;
       attr.value.booldata = state;

       // Update host_tx_ready before admin down
       if (!state && !m_cmisModuleAsicSyncSupported) {
           setHostTxReady(port, "false");
       }

       sai_status_t status = sai_port_api->set_port_attribute(port.m_port_id, &attr);

       // Update host_tx_ready after admin up
       if (state && status == SAI_STATUS_SUCCESS) {
           setHostTxReady(port, "true");
       }

       return true;
   }
   ```

### Important Note: Admin Status vs Oper Status

- **Admin Status**: User-configurable. Can be set to UP/DOWN via CONFIG_DB.
- **Oper Status**: Reflects actual link state. Determined by hardware.
- When admin_status is set to DOWN, SAI/SDK will bring the port down, which triggers an oper_status notification.
- When admin_status is set to UP, the port MAY come up (oper_status = UP) only if the physical link is healthy.

---

## Central Orchestration Function: updatePortOperStatus

This is the central function that handles all oper status changes (`portsorch.cpp:9041-9114`).

```cpp
void PortsOrch::updatePortOperStatus(Port &port, sai_port_oper_status_t status)
{
    // 1. Log the state transition
    SWSS_LOG_NOTICE("Port %s oper state set from %s to %s",
            port.m_alias.c_str(),
            oper_status_strings.at(port.m_oper_status).c_str(),
            oper_status_strings.at(status).c_str());

    // 2. Skip if no change
    if (status == port.m_oper_status) return;

    // 3. Update STATE_DB
    if (port.m_type == Port::PHY || port.m_type == Port::TUNNEL) {
        updateDbPortOperStatus(port, status);
    }

    // 4. Update flap count and timestamps
    if (port.m_type == Port::PHY) {
        updateDbPortFlapCount(port, status);
        updateGearboxPortOperStatus(port);

        // Refresh auto-negotiation state
        if (port.m_autoneg > 0) {
            refreshPortStateAutoNeg(port);
        }
        // Refresh link training state
        if (port.m_link_training > 0) {
            refreshPortStateLinkTraining(port);
        }
    }

    // 5. Update internal state
    port.m_oper_status = status;

    // 6. Update host interface (kernel netdev)
    bool isUp = status == SAI_PORT_OPER_STATUS_UP;
    if (port.m_type == Port::PHY) {
        setHostIntfsOperStatus(port, isUp);  // portsorch.cpp:3649
    }

    // 7. Inform NeighOrch about next hop changes
    gNeighOrch->ifChangeInformNextHop(port.m_alias, isUp);

    // Also for child ports (sub-interfaces)
    for (const auto &child_port : port.m_child_ports) {
        gNeighOrch->ifChangeInformNextHop(child_port, isUp);
    }

    // 8. VOQ chassis: sync interface state
    if (gMySwitchType == "voq") {
        gIntfsOrch->voqSyncIntfState(port.m_alias, isUp);
    }

    // 9. Notify observers (FdbOrch, FgNhgOrch, etc.)
    PortOperStateUpdate update = {port, status};
    notify(SUBJECT_TYPE_PORT_OPER_STATE_CHANGE, static_cast<void *>(&update));
}
```

---

## Downstream Components Affected

When a port goes operationally down, the following components are notified/updated:

### 1. STATE_DB Updates
- `PORT_TABLE|<port_name>`: `oper_status` field updated to "down"
- `PORT_TABLE|<port_name>`: `flap_count` incremented
- `PORT_TABLE|<port_name>`: `last_down_time` timestamp updated

### 2. Host Interface (Kernel netdev)
**File:** `portsorch.cpp:3649-3671`

```cpp
bool PortsOrch::setHostIntfsOperStatus(const Port& port, bool isUp) const
{
    sai_attribute_t attr;
    attr.id = SAI_HOSTIF_ATTR_OPER_STATUS;
    attr.value.booldata = isUp;
    sai_hostif_api->set_hostif_attribute(port.m_hif_id, &attr);

    // Publish event for external consumers
    event_params_t params = {{"ifname", port.m_alias}, {"status", isUp ? "up" : "down"}};
    event_publish(g_events_handle, "if-state", &params);
}
```

This updates the Linux kernel interface (e.g., `ip link show` will show the interface as DOWN).

### 3. NeighOrch - Next Hop Invalidation
**File:** `neighorch.cpp:523-555`

```cpp
bool NeighOrch::ifChangeInformNextHop(const string &alias, bool if_up)
{
    for (auto nhop = m_syncdNextHops.begin(); nhop != m_syncdNextHops.end(); ++nhop)
    {
        if (nhop->first.alias != alias) continue;

        if (if_up) {
            clearNextHopFlag(nhop->first, NHFLAGS_IFDOWN);
        } else {
            setNextHopFlag(nhop->first, NHFLAGS_IFDOWN);  // Mark NH as unusable
        }
    }
}
```

When a port goes down:
- All next hops associated with that interface are marked with `NHFLAGS_IFDOWN`
- Routes using these next hops will use alternate paths (if available via ECMP)
- Traffic is redirected away from the failed port

### 4. FdbOrch - MAC Address Flush
**File:** `fdborch.cpp:1204-1239`

```cpp
void FdbOrch::updatePortOperState(const PortOperStateUpdate& update)
{
    if (update.operStatus == SAI_PORT_OPER_STATUS_DOWN)
    {
        swss::Port p = update.port;

        // Skip flush for MCLAG interfaces (they handle failover differently)
        if (gMlagOrch->isMlagInterface(p.m_alias)) {
            SWSS_LOG_NOTICE("Ignoring fdb flush on MCLAG port:%s", p.m_alias.c_str());
            return;
        }

        // Flush FDB entries for this port
        if (p.m_bridge_port_id != SAI_NULL_OBJECT_ID) {
            flushFDBEntries(p.m_bridge_port_id, SAI_NULL_OBJECT_ID);
        }

        // Notify observers about FDB flush for each VLAN
        vlan_members_t vlan_members;
        m_portsOrch->getPortVlanMembers(p, vlan_members);
        for (const auto& vlan_member: vlan_members) {
            notifyObserversFDBFlush(p, vlan.m_vlan_info.vlan_oid);
        }
    }
}
```

When a port goes down:
- All MAC addresses learned on that port are flushed from the FDB table
- This ensures traffic is not forwarded to the dead port
- MCLAG interfaces are excluded (they have special handling for failover)

### 5. FgNhgOrch - Fine-Grained Next Hop Group Updates
**File:** `fgnhgorch.cpp:46-48`

```cpp
case SUBJECT_TYPE_PORT_OPER_STATE_CHANGE:
{
    PortOperStateUpdate *update = reinterpret_cast<PortOperStateUpdate *>(cntx);
    // Update fine-grained ECMP groups based on port state
}
```

### 6. VOQ System (Modular Chassis)
For VOQ-based distributed systems (`portsorch.cpp:9103-9109`):

```cpp
if (gMySwitchType == "voq")
{
    if (gIntfsOrch->isLocalSystemPortIntf(port.m_alias))
    {
        gIntfsOrch->voqSyncIntfState(port.m_alias, isUp);
    }
}
```

The interface state is synchronized across the chassis to all line cards.

---

## Sequence Diagrams

### Sequence 1: Link Failure (Hardware-Initiated)

```
Hardware    SAI/SDK    syncd    ASIC_DB    orchagent    STATE_DB    NeighOrch    FdbOrch
   |           |         |         |           |            |           |           |
   |--Link Down|         |         |           |            |           |           |
   |           |--notify-|         |           |            |           |           |
   |           |         |--pub--> |           |            |           |           |
   |           |         |         |--sub----> |            |           |           |
   |           |         |         |           |--update--> |           |           |
   |           |         |         |           |            |           |           |
   |           |         |         |           |---ifChangeInformNextHop->|           |
   |           |         |         |           |            |           |           |
   |           |         |         |           |---notify(PORT_OPER_STATE)-------->  |
   |           |         |         |           |            |           |--flushFDB |
```

### Sequence 2: Admin Shutdown (User-Initiated)

```
User    CONFIG_DB    orchagent    SAI/SDK    syncd    ASIC_DB    (then back to orchagent)
  |         |            |           |         |          |
  |--config-|            |           |         |          |
  |         |--event---> |           |         |          |
  |         |            |--set_attr->|         |          |
  |         |            |(ADMIN_STATE)|        |          |
  |         |            |           |--exec--> |          |
  |         |            |           |--notify->|          |
  |         |            |           |         |--pub----> |
  |         |            |           |         |          |--sub-->orchagent
  |         |            |           |         |          |   (updatePortOperStatus)
```

---

## Key Code References

| Component | File | Line | Description |
|-----------|------|------|-------------|
| SAI notification registration | `main.cpp` | 554-555 | Registers `on_port_state_change` callback |
| Notification callback | `notifications.cpp` | 29-39 | Forwards to ASIC_DB NOTIFICATIONS |
| Notification consumer | `portsorch.cpp` | 8879-8901 | `doTask(NotificationConsumer&)` |
| Notification handling | `portsorch.cpp` | 8903-8969 | `handleNotification()` |
| **Central oper status update** | `portsorch.cpp` | 9041-9114 | `updatePortOperStatus()` |
| Admin status setting | `portsorch.cpp` | 2106-2160 | `setPortAdminStatus()` |
| Host interface oper status | `portsorch.cpp` | 3649-3671 | `setHostIntfsOperStatus()` |
| Next hop invalidation | `neighorch.cpp` | 523-555 | `ifChangeInformNextHop()` |
| FDB flush | `fdborch.cpp` | 1204-1239 | `updatePortOperState()` |
| Observer pattern | `portsorch.cpp` | 9112-9113 | `notify(SUBJECT_TYPE_PORT_OPER_STATE_CHANGE)` |
| Observer registration | `fdborch.cpp` | 39 | `m_portsOrch->attach(this)` |

---

## Summary

1. **Two paths to oper status down:**
   - **Hardware-initiated (link failure):** Detected by SAI/SDK, notification sent through syncd to orchagent
   - **User-initiated (admin shutdown):** CONFIG_DB change triggers SAI API call, which causes hardware to generate oper status notification

2. **Central orchestration:** Both paths converge at `updatePortOperStatus()` which handles all downstream updates

3. **Downstream effects when port goes down:**
   - STATE_DB updated with new status and flap count
   - Kernel netdev marked as DOWN
   - Next hops marked with NHFLAGS_IFDOWN (routing updates)
   - FDB entries flushed (L2 cleanup)
   - Fine-grained ECMP groups updated
   - VOQ chassis state synchronized

4. **Observer pattern:** PortsOrch uses the observer pattern to notify interested components (FdbOrch, FgNhgOrch) about state changes
