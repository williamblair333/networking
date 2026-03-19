# 🔍 IGMP Multicast Flood — Root Cause & Resolution
## Cisco Catalyst 3850 + Meraki MX Environment

> **Platform:** Cisco IOS XE (WS-C3850) + Meraki MX Security Appliance &nbsp;|&nbsp; **Status:** ✅ Resolved

---

## 📋 Table of Contents

- [Executive Summary](#executive-summary)
- [Environment](#environment)
- [Background — How IGMP Snooping Is Supposed to Work](#background--how-igmp-snooping-is-supposed-to-work)
- [Root Cause Analysis](#root-cause-analysis)
  - [The Missing Querier](#the-missing-querier)
  - [The Missing Mrouter Entry](#the-missing-mrouter-entry)
  - [Why Meraki MX Makes This Worse](#why-meraki-mx-makes-this-worse)
  - [Flood Propagation](#flood-propagation)
- [Symptoms](#symptoms)
- [Diagnosis](#diagnosis)
  - [Step 1 — Identify the Flood](#step-1--identify-the-flood)
  - [Step 2 — Find the Source Port](#step-2--find-the-source-port)
  - [Step 3 — Identify the Source Device](#step-3--identify-the-source-device)
  - [Step 4 — Confirm IGMP State](#step-4--confirm-igmp-state)
- [Resolution](#resolution)
  - [Fix 1 — Static Mrouter Entry](#fix-1--static-mrouter-entry-immediate)
  - [Fix 2 — IGMP Querier](#fix-2--igmp-querier-permanent)
- [Verification](#verification)
- [Before vs. After](#before-vs-after)
- [Lessons Learned](#lessons-learned)
- [Long-Term Recommendation](#long-term-recommendation)
- [Quick Reference](#quick-reference)

---

## Executive Summary

A VoIP phone was sending continuous multicast traffic for group `224.0.1.116` (standard Polycom directory/presence protocol). The Cisco Catalyst 3850 had **IGMP snooping enabled but no IGMP querier configured**, and its default mrouter learning mode (`pim-dvmrp`) was **incompatible with the upstream Meraki MX** firewall. The result: the switch had no mrouter entry, treated all multicast as unknown, and flooded it to every port in the VLAN indefinitely.

**Fix:** Two commands.

```
ip igmp snooping vlan 101 mrouter interface GigabitEthernet1/0/48
ip igmp snooping querier
```

Post-fix, IGMP groups populated correctly and all OutDiscards zeroed.

---

## Environment

| Component | Details |
|-----------|---------|
| **Access/Distribution Switch** | Cisco WS-C3850-48P |
| **IOS XE Version** | 16.12.8 |
| **Upstream Firewall/Gateway** | Meraki MX Security Appliance |
| **Affected VLAN** | Employee data VLAN |
| **IGMP Snooping** | Enabled globally and per-VLAN — *before and after* |
| **IGMP Querier** | ❌ Not configured — root cause |
| **Mrouter Entry** | ❌ Empty — root cause |
| **Multicast Source** | Polycom IP phone (standard operation) |

---

## Background — How IGMP Snooping Is Supposed to Work

IGMP snooping prevents multicast flooding by tracking group membership at the port level. When a host wants to receive multicast traffic, it sends an **IGMP Join** for the group address. The switch records which port sent the join and forwards that multicast group only to that port — not to every port in the VLAN.

```
Without snooping:                   With snooping (working correctly):

Multicast source                    Multicast source
    │                                   │
    ▼                                   ▼
  Switch                             Switch
  ├──► Port A  ✅                    ├──► Port A  ✅  (sent IGMP Join)
  ├──► Port B  ❌ (doesn't want it)  ├──► Port B  🚫 (no join — blocked)
  ├──► Port C  ❌                    ├──► Port C  🚫
  └──► Port D  ❌                    └──► Port D  🚫
```

For snooping to work, the switch needs:

1. **An mrouter port** — the upstream port toward the router, where unregistered multicast gets forwarded
2. **An IGMP querier** — periodically queries hosts to refresh group membership records

Without either, the switch reverts to flooding all multicast to all ports — the same behavior as if snooping were disabled.

---

## Root Cause Analysis

### The Missing Querier

Without a querier, hosts have no trigger to renew their IGMP group memberships. Over time, membership records age out. Once records expire, the switch no longer knows which ports want which groups — and floods.

In a network with a persistent multicast source (like a VoIP phone sending to a paging group continuously), the flooding is **permanent and immediate** from the moment the mrouter entry is absent.

---

### The Missing Mrouter Entry

The Cisco 3850's default mrouter learning mode is `pim-dvmrp`. In this mode, the switch listens for **PIM or DVMRP hello messages** from upstream routers to automatically discover the mrouter port.

```
show ip igmp snooping mrouter

Vlan    ports
----    -----
                    ← Empty. No mrouter learned.
```

When the mrouter table is empty, the switch has no designated upstream port for multicast. Every unknown multicast group floods to every port in the VLAN.

---

### Why Meraki MX Makes This Worse

> ⚠️ **This is the critical undocumented incompatibility.**

The Meraki MX is a security appliance, not a multicast router. **It does not send PIM or DVMRP hello messages.** The 3850's `pim-dvmrp` learning mode will never discover the MX as an mrouter — regardless of how long you wait or how the MX is configured.

**In any Cisco IOS or IOS XE switch connected to a Meraki MX:**

- The mrouter table will never auto-populate
- IGMP snooping will never function correctly
- All unknown multicast will flood indefinitely

The workaround is a **static mrouter entry** pointing at the MX uplink port.

---

### Flood Propagation

```
VoIP Phone (access port)
    │
    │  Continuously sends multicast to 224.0.1.116
    │  (standard Polycom directory/presence)
    ▼
Catalyst 3850
    │
    │  No mrouter entry → unknown multicast group
    │  No querier → no membership tracking
    │  Result: flood to all VLAN ports
    │
    ├──► Workstation ports (silently discarding at NIC level)
    ├──► AP trunk ports (re-flooding over wireless)
    ├──► Port with unmanaged switch behind it
    │         └──► Unmanaged switch overrun → 327,888 OutDiscards
    └──► Every other active access port
```

The flooding was completely silent on most ports — workstation NICs discarded the frames without error. The only visible symptom was the port with an **unmanaged switch** behind it (multiple devices sharing a port, limited buffering), which accumulated massive output drops and eventually triggered storm control.

---

## Symptoms

The following symptoms were observed. Any one of them in isolation could have many causes. Together they point to multicast flooding.

| Symptom | Observation |
|---------|------------|
| Port err-disabled | Gi1/0/18 triggered storm control shutdown |
| Output drops | 327,888 on Gi1/0/18 — only port with unmanaged switch behind it |
| OutMcastPkts >> OutUcastPkts | Every active access port showing ~700k OutMcastPkts |
| IGMP groups table empty | `show ip igmp snooping groups` returned nothing |
| Mrouter table empty | `show ip igmp snooping mrouter` returned nothing |
| No error logs | Switch logs showed no relevant events — silent flood |

> ℹ️ **Why only one port showed drops:** Workstations with individual NICs silently discard frames they can't buffer. An unmanaged switch aggregating multiple devices creates a visible bottleneck that surfaces the problem. In environments with all single-device ports, this flood can run indefinitely without any obvious complaints.

---

## Diagnosis

### Step 1 — Identify the Flood

Check whether OutMcastPkts exceed OutUcastPkts on access ports. If yes, flooding is occurring.

```
show interfaces GigabitEthernet1/0/X counters
```

```
Port              OutOctets   OutUcastPkts   OutMcastPkts   OutBcastPkts
Gi1/0/18          631442306         608080         699775         243800
                                    ↑                ↑
                              Unicast normal    Multicast >> Unicast = flood
```

Check for output drops:

```
show interfaces counters errors | include OutDiscards
```

---

### Step 2 — Find the Source Port

Pull InMcastPkts across all ports. The port with the highest value is receiving multicast from its connected device — meaning it's the source.

```
show interfaces counters
```

Look for the outlier in the `InMcastPkts` column among access ports:

```
Port               InOctets    InUcastPkts    InMcastPkts    InBcastPkts
Gi1/0/6           490015917        1054495         374766    ← OUTLIER
Gi1/0/8           278376494         408178          56850
Gi1/0/9            33557216          93055           9303
```

---

### Step 3 — Identify the Source Device

```
show mac address-table interface GigabitEthernet1/0/6
```

```
Vlan    Mac Address       Type        Ports
----    -----------       --------    -----
 101    64:16:7f:bb:75:61 DYNAMIC     Gi1/0/6    ← VoIP phone OUI
 101    ac:00:f9:0c:39:bf DYNAMIC     Gi1/0/6
```

Look up the OUI of each MAC address. A VoIP phone OUI on the high-InMcast port confirms the source.

---

### Step 4 — Confirm IGMP State

```
show ip igmp snooping mrouter
show ip igmp snooping groups
show ip igmp snooping querier
```

If mrouter is empty and groups table is empty with snooping enabled, the querier is the missing component.

---

## Resolution

### Fix 1 — Static Mrouter Entry (Immediate)

Manually designate the uplink port toward the router as the mrouter port. Replace `GigabitEthernet1/0/48` with your actual uplink port. Replace `101` with your VLAN ID.

```
ip igmp snooping vlan 101 mrouter interface GigabitEthernet1/0/48
```

**Verify:**

```
show ip igmp snooping mrouter

Vlan    ports
----    -----
 101    Gi1/0/48(static)    ← Correct
```

This stops flooding of *registered* groups. Unknown groups still flood until the querier is running.

---

### Fix 2 — IGMP Querier (Permanent)

Configure the switch itself as the IGMP querier. Use the switch's management IP as the querier source address.

```
ip igmp snooping querier
ip igmp snooping querier address [switch-management-IP]
ip igmp snooping querier version 2
```

**Verify:**

```
show ip igmp snooping querier

Vlan    IP Address       IGt
---------------------------------------------
101     [switch-mgmt-IP] v2    Switch    ← Active on all VLANs
```

Once the querier sends its first query and hosts respond, the groups table populates and flooding stops.

**Save:**

```
copy running-config startup-config
```

---

## Verification

Run these after the querier has been active for 2–3 minutes and the multicast source device has reconnected:

```
show ip igmp snooping groups
show interfaces counters errors | include OutDiscards
```

**Expected results:**

| Check | Expected |
|-------|---------|
| `show ip igmp snooping groups` | Groups table populated with multicast addresses and specific port lists |
| `show interfaces counters errors \| include OutDiscards` | No output — zero drops |

---

## Before vs. After

### IGMP Groups Table

**Before:**
```
Vlan    Group    Type    Version    Port List
------------------------------------------------------
                    (empty)
```

**After:**
```
Vlan    Group             Type    Version    Port List
------------------------------------------------------
101     224.0.1.116       igmp    v2         Gi1/0/6, Gi1/0/8, Gi1/0/14 ...
101     239.255.255.250   igmp    v2         Gi1/0/2, Gi1/0/6, Gi1/0/9 ...
101     224.0.1.60        igmp    v2         Gi1/0/36, Gi1/0/46
```

Groups are now tracked. Multicast is forwarded only to subscribed ports.

### Traffic Pattern

| Metric | Before | After |
|--------|--------|-------|
| OutMcastPkts on access ports | ~700k (flood) | Proportional to actual subscriptions |
| OutDiscards | 327,888 | 0 |
| IGMP groups table | Empty | Populated |
| Mrouter entry | Empty | Gi1/0/48(static) |

---

## Lessons Learned

> ### 🔴 1 — IGMP Snooping Enabled ≠ IGMP Snooping Working
>
> A switch can report IGMP snooping as enabled globally and per-VLAN while flooding all multicast to all ports. Enabled means the feature is loaded. Working requires a querier and an mrouter entry. **Always verify the mrouter table and groups table, not just the snooping status.**

---

> ### 🔴 2 — Meraki MX Requires a Static Mrouter Entry
>
> The Meraki MX does not send PIM or DVMRP hellos. The Cisco 3850's default `pim-dvmrp` mrouter learning will never populate on a Meraki-connected switch. This is not a bug or misconfiguration — it is a design incompatibility that requires a static workaround on every Cisco IOS/IOS XE switch with a Meraki MX upstream.

---

> ### 🟡 3 — VoIP Phones Are Not the Problem
>
> Polycom and other IP phones use multicast by design. Group paging, directory services, and presence protocols all rely on multicast. A phone generating multicast is functioning correctly. The problem is always the network's inability to contain it. Configure IGMP querier **before** deploying any VoIP infrastructure that uses multicast.

---

> ### 🟡 4 — Silent Flooding Is the Dangerous Kind
>
> Multicast flooding produces no error logs, no SNMP traps, and no interface alarms on most ports. Workstations silently discard unneeded frames at the NIC level. The flood can run for days or weeks without complaints — degrading performance across the entire VLAN invisibly. The only trigger that surfaced this issue was an unmanaged switch behind one port creating visible drops.

---

## Long-Term Recommendation

### Implement a Dedicated Voice VLAN

The querier fix is correct and sufficient, but VoIP multicast traffic remains commingled with employee data traffic on the same VLAN. The best-practice architecture separates them:

```
┌──────────────────────────────────────┐
│        Data VLAN (e.g., 101)         │
│  Workstations, servers, printers     │
│  Multicast: Windows SSDP only        │
└──────────────────────────────────────┘

┌──────────────────────────────────────┐
│       Voice VLAN (e.g., 150)         │
│  IP phones, phone system             │
│  Multicast: VoIP paging, directory   │
│  QoS: DSCP EF marking, priority      │
└──────────────────────────────────────┘
```

**Benefits of voice VLAN segmentation:**

- VoIP multicast is physically contained within the voice VLAN — never touches data ports
- QoS policy applied cleanly per VLAN — no per-host configuration required
- Call quality issues are isolated and easier to troubleshoot
- Security boundary between voice and data traffic
- IGMP snooping on the voice VLAN only has to manage phone traffic — simpler group table

**Implementation scope:** Requires coordination between the network team and phone system administrator. Phone system configuration must be updated to match the new VLAN. Switchport configuration must be updated to add voice VLAN tagging. Not a quick change — plan as a scheduled maintenance item.

---

## Quick Reference

### Full Diagnosis Sequence

```
! Check mrouter state
show ip igmp snooping mrouter

! Check group membership
show ip igmp snooping groups

! Check querier status
show ip igmp snooping querier

! Find the flood — look for OutMcastPkts >> OutUcastPkts
show interfaces counters

! Find the source — highest InMcastPkts
show interfaces counters

! Check for drops
show interfaces counters errors | include OutDiscards

! Identify devices on suspect port
show mac address-table interface GigabitEthernet1/0/X
```

### Full Fix Sequence

```
! Fix 1 — static mrouter (replace VLAN and port with your values)
ip igmp snooping vlan 101 mrouter interface GigabitEthernet1/0/48

! Fix 2 — IGMP querier (replace IP with your switch management IP)
ip igmp snooping querier
ip igmp snooping querier address [switch-mgmt-IP]
ip igmp snooping querier version 2

! Save
copy running-config startup-config
```

### Verification Sequence

```
show ip igmp snooping querier
show ip igmp snooping mrouter
show ip igmp snooping groups
show interfaces counters errors | include OutDiscards
```

---

*Platform: Cisco IOS XE (WS-C3850) + Meraki MX*
*Document Version 1.0*
