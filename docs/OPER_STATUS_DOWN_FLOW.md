# SONiC SWSS Operational Status Down Flow

This document explains the operational status (oper status) down flow in SONiC SWSS, covering both hardware-initiated (link failure) and user-initiated (admin shutdown) scenarios.

## Table of Contents
1. [Overview](#overview)
2. [Key Components and Their Responsibilities](#key-components-and-their-responsibilities)
3. [Key Data Structures](#key-data-structures)
4. [Path 1: Oper Status Down Due to Link Failure](#path-1-oper-status-down-due-to-link-failure-hardware-initiated)
5. [Path 2: Oper Status Down Due to User Shutdown](#path-2-oper-status-down-due-to-user-shutdown-admin-initiated)
6. [Central Orchestration Function: updatePortOperStatus](#central-orchestration-function-updateportoperstatus)
7. [Downstream Components Affected](#downstream-components-affected)
8. [Sequence Diagrams](#sequence-diagrams)
9. [Key Code References](#key-code-references)

---

## Overview

There are two distinct paths that lead to a port's operational status going down:

| Trigger | Origin | Flow Path |
|---------|--------|-----------|
| Link Failure | Hardware/PHY layer detects loss of signal | SAI SDK -> syncd -> ASIC_DB NOTIFICATIONS -> orchagent |
| Admin Shutdown | User configuration (CONFIG_DB) | CONFIG_DB -> portsorch -> SAI API -> SAI generates notification -> orchagent |

Both paths ultimately converge at the `updatePortOperStatus()` function in portsorch.cpp, which handles all downstream updates.

---

## Key Components and Their Responsibilities

### Database Updates by Component

| Component | Database | Fields Updated | Source |
|-----------|----------|----------------|--------|
| **portsyncd** | STATE_DB | `admin_status`, `netdev_oper_status`, `state`, `mtu` | Linux kernel netlink (RTM_NEWLINK) |
| **portsorch** | APPL_DB | `oper_status`, `flap_count`, `last_down_time`, `last_up_time` | SAI notifications |
| **portsorch** | STATE_DB | `speed`, `fec`, `link_training_status`, `rmt_adv_speeds` | SAI attribute queries |

### portsyncd Role (linksync.cpp)

**portsyncd** listens to Linux kernel netlink events and updates **STATE_DB**:

```cpp
// linksync.cpp:197-205
FieldValueTuple admin_status("admin_status", (admin ? "up" : "down"));   // from IFF_UP flag
FieldValueTuple op("netdev_oper_status", oper ? "up" : "down");          // from IFF_RUNNING flag
vector.push_back(admin_status);
vector.push_back(op);
m_statePortTable.set(key, vector);  // STATE_DB PORT_TABLE
```

- `admin_status`: Reflects kernel interface admin state (IFF_UP flag)
- `netdev_oper_status`: Reflects kernel interface oper state (IFF_RUNNING flag)

### portsorch Role (portsorch.cpp)

**portsorch** receives SAI notifications and updates **APPL_DB**:

```cpp
// portsorch.cpp:3787-3802
void PortsOrch::updateDbPortOperStatus(const Port& port, sai_port_oper_status_t status) const
{
    vector<FieldValueTuple> tuples;
    FieldValueTuple tuple("oper_status", oper_status_strings.at(status));
    tuples.push_back(tuple);
    m_portTable->set(port.m_alias, tuples);  // APPL_DB PORT_TABLE
}
```

- `oper_status`: Reflects SAI/ASIC port operational state

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
         v (SubscriberStateTable in portmgrd)
+-------------------+
| PortMgr::doTask() |  cfgmgr/portmgr.cpp:138
+-------------------+
         |
         v
+-------------------+
| PortMgr::         |  cfgmgr/portmgr.cpp:60-84
| setPortAdminStatus|
+-------------------+
         |
         +---------------------------+
         |                           |
         v                           v
+-------------------+       +-------------------+
| ip link set dev   |       | writeConfigToAppDb|  cfgmgr/portmgr.cpp:71
| <port> down       |       | (APPL_DB)         |
+-------------------+       +-------------------+
         |                           |
         v                           v (SubscriberStateTable in orchagent)
+-------------------+       +-------------------+
| Kernel netdev     |       | PortsOrch::       |  portsorch.cpp:5156-5170
| IFF_UP = 0        |       | doPortTask()      |
+-------------------+       +-------------------+
         |                           |
         v (netlink RTM_NEWLINK #1)  v
+-------------------+       +-------------------+
| portsyncd         |       | PortsOrch::       |  portsorch.cpp:2106-2160
| LinkSync::onMsg() |       | setPortAdminStatus|
+-------------------+       +-------------------+
         |                           |
         v                           v
+---------------------+     +-------------------+
| STATE_DB update #1  |     | sai_port_api->    |  portsorch.cpp:2125
| admin_status=down   |     | set_port_attribute|  (SAI_PORT_ATTR_ADMIN_STATE)
| netdev_oper_status  |     +-------------------+
| =up (unchanged)     |              |
+---------------------+              v (SAI/SDK brings port down)
                            +-------------------+
                            | SAI SDK           |  Hardware executes admin down
                            +-------------------+
                                     |
                                     v (SAI generates oper status notification)
                            +-------------------+
                            | updatePortOperStatus|  portsorch.cpp:9041-9114
                            +-------------------+
                                     |
         +---------------------------+---------------------------+
         |                           |                           |
         v                           v                           v
+-------------------+       +-------------------+       +-------------------+
| APPL_DB           |       | APPL_DB           |       | setHostIntfsOper  |
| oper_status=down  |       | flap_count++      |       | Status(false)     |
| (portsorch.cpp:   |       | last_down_time    |       | (portsorch.cpp:   |
|  3787-3802)       |       | (portsorch.cpp:   |       |  3649-3671)       |
+-------------------+       |  3734-3762)       |       +-------------------+
                            +-------------------+                |
                                                                 v
                                                        +-------------------+
                                                        | SAI hostif API    |
                                                        | IFF_RUNNING = 0   |
                                                        +-------------------+
                                                                 |
                                                                 v (netlink RTM_NEWLINK #2)
                                                        +-------------------+
                                                        | portsyncd         |
                                                        | LinkSync::onMsg() |
                                                        +-------------------+
                                                                 |
                                                                 v
                                                        +---------------------+
                                                        | STATE_DB update #2  |
                                                        | admin_status=down   |
                                                        | netdev_oper_status  |
                                                        | =down               |
                                                        +---------------------+
```

### Key Code Path

1. **CONFIG_DB Change Received by portmgrd** (`portmgr.cpp:186-199`)
   ```cpp
   for (auto i : kfvFieldsValues(t))
   {
       if (fvField(i) == "admin_status")
       {
           admin_status = fvValue(i);
       }
   }
   // ...
   setPortAdminStatus(alias, admin_status == "up");
   ```

2. **portmgrd Sets Kernel Interface Down** (`portmgr.cpp:60-84`)
   ```cpp
   bool PortMgr::setPortAdminStatus(const string &alias, const bool up)
   {
       // ip link set dev <port_name> [up|down]
       cmd << IP_CMD << " link set dev " << shellquote(alias) << (up ? " up" : " down");
       int ret = swss::exec(cmd_str, res);
       if (!ret)
       {
           // Write to APPL_DB after kernel command succeeds
           return writeConfigToAppDb(alias, "admin_status", (up ? "up" : "down"));
       }
   }
   ```

3. **portsyncd Receives Netlink Event** (`linksync.cpp:111-206`)
   - Kernel interface state change triggers RTM_NEWLINK
   - portsyncd updates STATE_DB with `admin_status` and `netdev_oper_status`

4. **portsorch Receives APPL_DB Change** (`portsorch.cpp:5156-5170`)
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

5. **portsorch Calls SAI API** (`portsorch.cpp:2106-2160`)
   ```cpp
   bool PortsOrch::setPortAdminStatus(Port &port, bool state)
   {
       sai_attribute_t attr;
       attr.id = SAI_PORT_ATTR_ADMIN_STATE;
       attr.value.booldata = state;

       sai_status_t status = sai_port_api->set_port_attribute(port.m_port_id, &attr);
       return true;
   }
   ```

### Important Note: Parallel Paths

When a user shuts down an interface, two parallel paths are triggered:

1. **Kernel Path (portmgrd)**:
   - `ip link set dev <port> down` → Kernel netdev goes down
   - portsyncd receives netlink → Updates STATE_DB (`admin_status`, `netdev_oper_status`)

2. **SAI/ASIC Path (portsorch)**:
   - APPL_DB change → portsorch calls SAI API
   - SAI/SDK brings ASIC port down → Generates oper status notification
   - portsorch receives notification → Updates APPL_DB (`oper_status`)

### Admin Status vs Oper Status

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

    // 3. Update APPL_DB with oper_status
    if (port.m_type == Port::PHY || port.m_type == Port::TUNNEL) {
        updateDbPortOperStatus(port, status);  // writes to APPL_DB PORT_TABLE
    }

    // 4. Update flap count and timestamps (APPL_DB)
    if (port.m_type == Port::PHY) {
        updateDbPortFlapCount(port, status);   // writes to APPL_DB PORT_TABLE
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

    // 6. Update host interface (kernel netdev) - this triggers portsyncd to update STATE_DB
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

### 1. Database Updates

**APPL_DB** (updated by portsorch via `updateDbPortOperStatus` and `updateDbPortFlapCount`):
- `PORT_TABLE|<port_name>`: `oper_status` field updated to "down"
- `PORT_TABLE|<port_name>`: `flap_count` incremented
- `PORT_TABLE|<port_name>`: `last_down_time` timestamp updated

**STATE_DB** (updated by portsyncd via netlink events):
- `PORT_TABLE|<port_name>`: `netdev_oper_status` updated when kernel interface state changes
- `PORT_TABLE|<port_name>`: `admin_status` updated when kernel admin state changes

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
Hardware    SAI/SDK    syncd    ASIC_DB    orchagent    APPL_DB    Kernel    portsyncd    STATE_DB
   |           |         |         |           |            |          |          |           |
   |--Link Down|         |         |           |            |          |          |           |
   |           |--notify-|         |           |            |          |          |           |
   |           |         |--pub--> |           |            |          |          |           |
   |           |         |         |--sub----> |            |          |          |           |
   |           |         |         |           |            |          |          |           |
   |           |         |         |    updatePortOperStatus()         |          |           |
   |           |         |         |           |--oper_status->        |          |           |
   |           |         |         |           |            |          |          |           |
   |           |         |         |           |--setHostIntfsOperStatus--------->|           |
   |           |         |         |           |            |          |--netlink>|           |
   |           |         |         |           |            |          |          |--update-->|
   |           |         |         |           |            |          |          |(netdev_   |
   |           |         |         |           |            |          |          | oper_status)
```

### Sequence 2: Admin Shutdown (User-Initiated) - Detailed Timeline

This diagram shows the two separate netlink events and STATE_DB updates:

```
Time  CONFIG_DB   portmgrd    Kernel     portsyncd   STATE_DB    APPL_DB    portsorch    SAI/SDK
  |       |          |          |            |           |           |           |           |
  |  admin_status    |          |            |           |           |           |           |
  |   = down         |          |            |           |           |           |           |
  |       |--------->|          |            |           |           |           |           |
  |       |          |          |            |           |           |           |           |
T1|       |    ip link set      |            |           |           |           |           |
  |       |    dev <port> down  |            |           |           |           |           |
  |       |          |--------->|            |           |           |           |           |
  |       |          |  IFF_UP=0|            |           |           |           |           |
  |       |          |          |            |           |           |           |           |
T2|       |          |  RTM_NEWLINK #1       |           |           |           |           |
  |       |          |          |----------->|           |           |           |           |
  |       |          |          |            |           |           |           |           |
T3|       |          |          |    STATE_DB update #1  |           |           |           |
  |       |          |          |            |---------->|           |           |           |
  |       |          |          |            | admin_status=down     |           |           |
  |       |          |          |            | netdev_oper_status=up (unchanged) |           |
  |       |          |          |            |           |           |           |           |
  |       |    writeConfigToAppDb           |           |           |           |           |
  |       |          |------------------------------------------->  |           |           |
  |       |          |          |            |           | admin_status=down     |           |
  |       |          |          |            |           |           |           |           |
T4|       |          |          |            |           |           |<----------|           |
  |       |          |          |            |           |           | (receives APPL_DB)    |
  |       |          |          |            |           |           |           |           |
T5|       |          |          |            |           |           |--setPortAdminStatus-->|
  |       |          |          |            |           |           |  SAI_PORT_ATTR_       |
  |       |          |          |            |           |           |  ADMIN_STATE=false    |
  |       |          |          |            |           |           |           |           |
T6|       |          |          |            |           |           |           |--ASIC---->|
  |       |          |          |            |           |           |           | port down |
  |       |          |          |            |           |           |           |           |
T7|       |          |          |            |           |           |<--SAI notification----|
  |       |          |          |            |           |           |  (oper_status=down)   |
  |       |          |          |            |           |           |           |           |
T8|       |          |          |            |           |           |           |           |
  |       |          |          |            |           |   updatePortOperStatus()          |
  |       |          |          |            |           |   portsorch writes to APPL_DB:    |
  |       |          |          |            |           |     - oper_status = down          |
  |       |          |          |            |           |     - flap_count++                |
  |       |          |          |            |           |     - last_down_time              |
  |       |          |          |            |           |           |           |           |
T9|       |          |          |            |           |           |--setHostIntfsOperStatus
  |       |          |          |            |           |           |  SAI_HOSTIF_ATTR_     |
  |       |          |          |            |           |           |  OPER_STATUS=false    |
  |       |          |<--------------------------------------------|           |           |
  |       |          |IFF_RUNNING=0          |           |           |           |           |
  |       |          |          |            |           |           |           |           |
T10|      |          |  RTM_NEWLINK #2       |           |           |           |           |
  |       |          |          |----------->|           |           |           |           |
  |       |          |          |            |           |           |           |           |
T11|      |          |          |    STATE_DB update #2  |           |           |           |
  |       |          |          |            |---------->|           |           |           |
  |       |          |          |            | admin_status=down     |           |           |
  |       |          |          |            | netdev_oper_status=down           |           |
```

### Timeline Summary for Admin Shutdown:

| Time | Component | Action | Trigger |
|------|-----------|--------|---------|
| T1 | portmgrd | `ip link set dev <port> down` | CONFIG_DB change |
| T2 | Kernel | Clears IFF_UP, sends RTM_NEWLINK #1 | ip link command |
| T3 | portsyncd | STATE_DB: admin_status=down, netdev_oper_status=up | Netlink #1 |
| T4 | portsorch | Receives APPL_DB change | portmgrd writeConfigToAppDb |
| T5 | portsorch | Calls SAI setPortAdminStatus | APPL_DB change |
| T6 | SAI/SDK | Brings ASIC port down | SAI API call |
| T7 | SAI/SDK | Generates oper status notification | Hardware state change |
| T8 | portsorch | **updatePortOperStatus: writes APPL_DB** (oper_status=down, flap_count++, last_down_time) | SAI notification |
| T9 | portsorch | setHostIntfsOperStatus(false) | updatePortOperStatus |
| T10 | Kernel | Clears IFF_RUNNING, sends RTM_NEWLINK #2 | SAI hostif API |
| T11 | portsyncd | STATE_DB: admin_status=down, netdev_oper_status=down | Netlink #2 |

### Key Points:

1. **Two Netlink Events**: RTM_NEWLINK #1 (from portmgrd) and RTM_NEWLINK #2 (from portsorch)
2. **Two STATE_DB Updates**: portsyncd updates STATE_DB twice - once per netlink event
3. **Both fields updated together**: Each netlink event contains both IFF_UP and IFF_RUNNING flags
4. **Trigger chain**: `setHostIntfsOperStatus()` → SAI hostif API → Kernel IFF_RUNNING change → Netlink → portsyncd → STATE_DB

---

## Key Code References

| Component | File | Line | Description |
|-----------|------|------|-------------|
| **portmgrd - CONFIG_DB handler** | `portmgr.cpp` | 138-250 | `doTask()` - receives CONFIG_DB changes |
| **portmgrd - kernel admin set** | `portmgr.cpp` | 60-84 | `setPortAdminStatus()` - `ip link set dev <port> up/down` |
| **portmgrd - APPL_DB write** | `portmgr.cpp` | 71, 252-260 | `writeConfigToAppDb()` - writes admin_status to APPL_DB |
| SAI notification registration | `main.cpp` | 554-555 | Registers `on_port_state_change` callback |
| Notification callback | `notifications.cpp` | 29-39 | Forwards to ASIC_DB NOTIFICATIONS |
| Notification consumer | `portsorch.cpp` | 8879-8901 | `doTask(NotificationConsumer&)` |
| Notification handling | `portsorch.cpp` | 8903-8969 | `handleNotification()` |
| **Central oper status update** | `portsorch.cpp` | 9041-9114 | `updatePortOperStatus()` |
| APPL_DB oper_status update | `portsorch.cpp` | 3787-3802 | `updateDbPortOperStatus()` - writes to APPL_DB |
| APPL_DB flap count update | `portsorch.cpp` | 3734-3762 | `updateDbPortFlapCount()` - writes to APPL_DB |
| portsorch admin status (SAI) | `portsorch.cpp` | 2106-2160 | `setPortAdminStatus()` - calls SAI API |
| Host interface oper status | `portsorch.cpp` | 3649-3671 | `setHostIntfsOperStatus()` |
| **STATE_DB update (portsyncd)** | `linksync.cpp` | 197-205 | `m_statePortTable.set()` - netlink handler |
| Next hop invalidation | `neighorch.cpp` | 523-555 | `ifChangeInformNextHop()` |
| FDB flush | `fdborch.cpp` | 1204-1239 | `updatePortOperState()` |
| Observer pattern | `portsorch.cpp` | 9112-9113 | `notify(SUBJECT_TYPE_PORT_OPER_STATE_CHANGE)` |
| Observer registration | `fdborch.cpp` | 39 | `m_portsOrch->attach(this)` |

---

## Summary

1. **Two paths to oper status down:**
   - **Hardware-initiated (link failure):** Detected by SAI/SDK, notification sent through syncd to orchagent
   - **User-initiated (admin shutdown):** CONFIG_DB → portmgrd → kernel + APPL_DB → portsorch → SAI → notification

2. **User-initiated shutdown involves parallel paths:**
   - **portmgrd**: `ip link set dev <port> down` → kernel netdev down → portsyncd → STATE_DB
   - **portsorch**: APPL_DB change → SAI API → ASIC port down → SAI notification → APPL_DB

3. **Central orchestration:** Both hardware and user-initiated paths converge at `updatePortOperStatus()` which handles all downstream updates

4. **Component responsibilities:**
   - **portmgrd** → Kernel interface control (`ip link set`), writes to APPL_DB
   - **portsyncd** → STATE_DB: `admin_status`, `netdev_oper_status` (from kernel netlink)
   - **portsorch** → SAI API calls, APPL_DB: `oper_status`, `flap_count`, `last_down_time`

5. **Downstream effects when port goes down:**
   - Kernel netdev marked as DOWN (by portmgrd via `ip link set`)
   - STATE_DB updated with admin_status, netdev_oper_status (portsyncd)
   - APPL_DB updated with oper_status and flap count (portsorch)
   - Next hops marked with NHFLAGS_IFDOWN (routing updates)
   - FDB entries flushed (L2 cleanup)
   - Fine-grained ECMP groups updated
   - VOQ chassis state synchronized

6. **Observer pattern:** PortsOrch uses the observer pattern to notify interested components (FdbOrch, FgNhgOrch) about state changes
