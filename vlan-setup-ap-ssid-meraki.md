# 🏢 Meraki Guest Network Setup Guide
## Adding an Isolated Guest SSID to an Existing Meraki Stack

> **Document Type:** How-To + Reference  
> **Platform:** Cisco Meraki MX + MS + MR  
> **Date:** March 2026  
> **Status:** ✅ Complete — All steps verified in production

---

## 📋 Table of Contents

1. [Overview](#1-overview)
2. [Environment & Prerequisites](#2-environment--prerequisites)
3. [Architecture Diagram](#3-architecture-diagram)
4. [Step-by-Step Configuration](#4-step-by-step-configuration)
   - [Step 1 — Create the Guest VLAN](#step-1--create-the-guest-vlan)
   - [Step 2 — Unlock the DHCP Page](#step-2--unlock-the-dhcp-page)
   - [Step 3 — Configure DHCP](#step-3--configure-dhcp)
   - [Step 4 — Configure the Guest SSID](#step-4--configure-the-guest-ssid)
   - [Step 5 — Convert AP Ports to Trunk](#step-5--convert-ap-ports-to-trunk)
   - [Step 6 — Add Guest Isolation Firewall Rules](#step-6--add-guest-isolation-firewall-rules)
   - [Step 7 — Verify End-to-End](#step-7--verify-end-to-end)
5. [Reference Tables](#5-reference-tables)
6. [Troubleshooting](#6-troubleshooting)
7. [Lessons Learned](#7-lessons-learned)
8. [Checklist](#8-checklist)

---

## 1. Overview

This guide documents the complete setup of a **guest wireless network** layered on top of an existing Meraki stack (MX + MS + MR). The process applies to any organization that already has a primary employee SSID and wants to add a fully isolated guest SSID without adding hardware.

The guest network produced by this guide is:

- **Isolated** from all employee resources via L3 firewall rules
- **VLAN-segmented** on its own subnet, separate from the employee network
- **DHCP-served** by the MX firewall appliance
- **Broadcast** on existing Meraki APs as a second SSID alongside the employee network

> ⚠️ **Critical Lesson Learned:** The Meraki MX will not display a new VLAN on the DHCP configuration page until that VLAN is actively assigned to at least one physical MX LAN port. This behavior is **undocumented**. See [Step 2](#step-2--unlock-the-dhcp-page) and [Lessons Learned](#7-lessons-learned).

---

## 2. Environment & Prerequisites

### Tested Hardware

This guide was tested on the following Meraki stack. Steps apply to any equivalent MX/MS/MR combination.

| Role | Device Type | Management |
|------|-------------|------------|
| Firewall / Gateway | Meraki MX appliance | Meraki Dashboard |
| Distribution Switch | Meraki MS switch (PoE) | Meraki Dashboard |
| Access Switch | Cisco IOS switch *(optional)* | CLI / SSH |
| Wireless APs | Meraki MR access points | Meraki Dashboard |

### Prerequisites

- [ ] Meraki Dashboard access — **Organization Administrator** role required
- [ ] Guest SSID already created in `Wireless → Configure → SSIDs` *(can be blank/unconfigured)*
- [ ] Employee VLAN already operational with confirmed DHCP and internet access
- [ ] AP port numbers on the MS switch identified *(check Dashboard → Switch → Ports, verify by MAC)*

### Example Network Addressing

> 📝 **Adapt these values to your environment.** The addresses below are examples used throughout this guide for illustration. Substitute your own VLAN ID, subnet, and gateway.

| Parameter | Example Value | Your Value |
|-----------|--------------|------------|
| **Employee VLAN ID** | `101` | __________ |
| **Employee Subnet** | `10.10.1.0/24` | __________ |
| **Employee Gateway** | `10.10.1.1` | __________ |
| **Guest VLAN ID** | `200` | __________ |
| **Guest VLAN Name** | `Guest-Wireless` | __________ |
| **Guest Subnet** | `10.10.4.0/24` | __________ |
| **Guest Gateway** | `10.10.4.254` | __________ |
| **Guest DHCP Pool** | `10.10.4.10 – 10.10.4.200` | __________ |
| **DNS** | `8.8.8.8`, `8.8.4.4` | __________ |
| **Lease Time** | `1 hour` | __________ |
| **VPN Advertised** | `Disabled` | __________ |

---

## 3. Architecture Diagram

### Physical Topology

```
                    ┌──────────────────────────────────┐
                    │         Meraki MX Appliance       │
                    │   Employee Gateway: 10.10.1.1     │
                    │   Guest Gateway:   10.10.4.254    │
                    │                                   │
                    │   VLAN 101 ──► Employee traffic   │
                    │   VLAN 200 ──► Guest traffic      │
                    │                                   │
                    │   Unused Port: VLAN 200 ◄──────── │─── Required to unlock DHCP page
                    └──────────────────┬───────────────┘
                                       │ Trunk — Native Employee VLAN, All VLANs
                                       │
                    ┌──────────────────┴───────────────┐
                    │       Meraki MS Switch (PoE)      │
                    │   Root Bridge                     │
                    │                                   │
                    │   AP Ports: Trunk                 │
                    │   Native Employee VLAN, All VLANs │
                    └────┬──────────┬──────────┬───────┘
                         │          │          │
                    ┌────┴──┐  ┌────┴──┐  ┌───┴───┐
                    │ MR AP │  │ MR AP │  │ MR AP │
                    │  01   │  │  02   │  │  03   │
                    └───────┘  └───────┘  └───────┘
                         │          │          │
                    ┌────┴──────────┴──────────┴───────┐
                    │         Wireless Clients          │
                    │                                   │
                    │  corp-wifi ──► VLAN 101 ──► 10.10.1.x  │
                    │  guest-wifi ──► VLAN 200 ──► 10.10.4.x │
                    └───────────────────────────────────┘
```

### Traffic Flow — Guest Client

```
Guest Device
    │
    │  Associates to guest SSID
    ▼
Meraki MR Access Point
    │
    │  Tags frames with Guest VLAN ID (802.1Q)
    ▼
Meraki MS Switch (Trunk port — carries tagged guest VLAN)
    │
    ▼
Meraki MX (Routes guest VLAN, serves DHCP, enforces firewall)
    │
    │  ✅ Permits: Internet-bound traffic
    │  ❌ Denies:  Employee subnet (all protected resources)
    ▼
Internet
```

---

## 4. Step-by-Step Configuration

---

### Step 1 — Create the Guest VLAN

**Dashboard path:** `Security & SD-WAN → Configure → Addressing & VLANs`

1. Click **Add a local network**
2. Enter your guest VLAN values:

   | Field | Example Value | Your Value |
   |-------|--------------|------------|
   | Name | `Guest-Wireless` | __________ |
   | VLAN ID | `200` | __________ |
   | Subnet | `10.10.4.0/24` | __________ |
   | MX IP / Gateway | `10.10.4.254` | __________ |
   | VPN mode | `Disabled` | __________ |

3. Click **Save**

> 💡 **Note:** After saving, the VLAN exists in the routing table but DHCP is ***not yet configurable***. Do not skip Step 2 — the DHCP section for this VLAN will not appear until Step 2 is complete.

---

### Step 2 — Unlock the DHCP Page

> ⚠️ **This step is non-obvious and undocumented by Meraki.** Do not skip it.

The Meraki MX only displays a VLAN on the DHCP configuration page **after that VLAN is assigned to at least one physical LAN port.** Without this, the VLAN will be invisible on the DHCP page regardless of whether it exists in Addressing & VLANs.

**Dashboard path:** `Security & SD-WAN → Configure → Addressing & VLANs → MX Ports`

1. Identify any **unused physical LAN port** on your MX appliance
2. Assign that port to your guest VLAN ID
3. Save

> ✅ The port does not need to be physically connected to anything — this is a software-only assignment that satisfies the MX's requirement. Your existing trunk port to the MS switch already passes all VLANs and requires no change.

> 💡 After saving, navigate to `Security & SD-WAN → Configure → DHCP` and confirm your guest VLAN now appears as a configurable section before proceeding.

---

### Step 3 — Configure DHCP

**Dashboard path:** `Security & SD-WAN → Configure → DHCP`

Scroll to your guest VLAN section (now visible after Step 2) and configure:

| Setting | Example Value | Your Value |
|---------|--------------|------------|
| Client addressing | `Run a DHCP server` | |
| DHCP pool start | `10.10.4.10` | __________ |
| DHCP pool end | `10.10.4.200` | __________ |
| DNS nameservers | `8.8.8.8`, `8.8.4.4` | __________ |
| Lease time | `1 hour` | __________ |

Click **Save**.

> 💡 A 1-hour lease is recommended for guest networks — shorter leases reduce IP exhaustion risk from transient devices.

---

### Step 4 — Configure the Guest SSID

**Dashboard path:** `Wireless → Configure → SSIDs → [your guest SSID] → Access Control`

#### 4a — Set Client IP Assignment

> ⚠️ **Do this first.** VLAN tagging is completely inaccessible until Bridge mode is enabled.

| Field | Value |
|-------|-------|
| Client IP assignment | `Bridge mode` *(shown in dashboard as "External DHCP server assigned")* |

Click **Save**, then return to the Access Control page.

---

#### 4b — Set VLAN Tagging

After enabling Bridge mode, the VLAN tagging section becomes editable.

| Field | Value |
|-------|-------|
| VLAN tagging | `Enabled` |
| Default VLAN | *(your guest VLAN ID — e.g., `200`)* |

> ⚠️ **Use the Default VLAN field only.** Do not add rows to the AP tag table unless you specifically need per-AP VLAN steering. An AP tag row with no tag selected will cause a save error: *"vlan tag 0: must not be empty"*

Click **Save**.

---

### Step 5 — Convert AP Ports to Trunk

The MR access points were previously on access ports carrying only the employee VLAN. To carry both the employee and guest SSIDs simultaneously, the AP-facing ports on the MS switch must become **trunk ports**.

**Dashboard path:** `Switching → Switches → [your MS switch] → Ports`

For each port connected to an MR access point, apply:

| Setting | Value |
|---------|-------|
| Port type | `Trunk` |
| Native VLAN | *(your employee VLAN ID — e.g., `101`)* |
| Allowed VLANs | `All` |
| PoE | `Enabled` |

> 💡 **Identifying AP ports:** If you don't know which ports the APs are on, go to `Wireless → Access Points → [AP name] → Switch port` in the Dashboard. Alternatively, check `Switching → Switches → [switch] → Clients` and filter by the AP's MAC address.

> 💡 **Native VLAN on AP ports:** Setting native VLAN to your employee VLAN ensures the APs themselves maintain their management IPs on the employee network. Only *client traffic* from the guest SSID is tagged with the guest VLAN — the AP's own management traffic remains untagged.

---

### Step 6 — Add Guest Isolation Firewall Rules

**Dashboard path:** `Security & SD-WAN → Configure → Firewall → L3 Outbound Rules`

Add the following rules **above** the default allow rule. Meraki evaluates rules top-down — first match wins.

| # | Policy | Protocol | Source | Destination | Port | Comment |
|---|--------|----------|--------|-------------|------|---------|
| 1 | **Deny** | **Any** | *(guest subnet — e.g., `10.10.4.0/24`)* | *(employee subnet — e.g., `10.10.1.0/24`)* | Any | Block guest → employee network |
| 2 | Allow | Any | Any | Any | Any | Default rule *(pre-existing)* |

> ⚠️ **Set Protocol to `Any`, not `TCP`.** A TCP-only deny rule leaves ICMP (ping) and UDP traffic unrestricted — guest clients would still be able to ping protected hosts. Always use `Any` on deny rules.

> 💡 **Additional targeted rules (optional):** If you later need to allow specific cross-VLAN services while keeping the broad deny in place, insert targeted allow rules *above* Rule 1. Example: allow guest access to a printer at `10.10.1.50/32` on TCP/9100 by inserting an allow rule before the catch-all deny.

Click **Save**.

---

### Step 7 — Verify End-to-End

Connect a test device to the guest SSID and run through the full verification sequence.

#### 7a — Confirm DHCP Assignment

Check the device's IP address via OS settings — do **not** rely on third-party network apps, which may display the cellular interface address instead of WiFi.

**How to check on each platform:**

| Platform | Path |
|----------|------|
| Android | `Settings → WiFi → [guest SSID] → gear icon → IP address` |
| iOS | `Settings → WiFi → [guest SSID] → ⓘ → IP Address` |
| Windows | `ipconfig` in Command Prompt |
| macOS | `System Settings → WiFi → Details → TCP/IP` |

***Expected result:***

```
IP Address:  10.10.4.x       ← Must be in your guest DHCP pool
Subnet Mask: 255.255.255.0
Gateway:     10.10.4.254     ← Your guest VLAN gateway
DNS:         8.8.8.8
```

If the IP is not in the guest subnet, VLAN tagging is not working correctly — return to Step 4.

---

#### 7b — Connectivity Tests

From the test device, run the following:

```bash
# Internet access — should SUCCEED
ping 8.8.8.8

# Guest gateway — will SUCCEED
# (expected — the MX always responds to pings addressed to itself)
ping 10.10.4.254

# Employee subnet hosts — should FAIL
ping 10.10.1.1
ping 10.10.1.x    ← Any employee device
```

> ℹ️ **Why the gateway responds:** The MX gateway IP (`10.10.4.254`) will always respond to pings from guest clients regardless of firewall rules. Meraki L3 outbound rules control traffic *transiting* the MX to other subnets — they do not block traffic *destined for* the MX itself. This is expected behavior and is not a security gap.

---

#### 7c — Dashboard Verification

`Security & SD-WAN → Configure → DHCP → [guest VLAN section]`

Confirm the DHCP lease table shows your test device's MAC address with an IP in the guest pool range.

---

## 5. Reference Tables

### VLAN Design — Example vs. Template

| | Example (this guide) | Your Environment |
|---|---|---|
| **Employee VLAN ID** | `101` | __________ |
| **Employee Subnet** | `10.10.1.0/24` | __________ |
| **Employee Gateway** | `10.10.1.1` | __________ |
| **Employee SSID** | `corp-wifi` | __________ |
| **Guest VLAN ID** | `200` | __________ |
| **Guest Subnet** | `10.10.4.0/24` | __________ |
| **Guest Gateway** | `10.10.4.254` | __________ |
| **Guest SSID** | `guest-wifi` | __________ |
| **VPN advertised** | Disabled | __________ |

---

### AP Port Configuration Summary

| Setting | Access Port (before) | Trunk Port (after) |
|---------|--------------------|--------------------|
| Port type | Access | **Trunk** |
| Native VLAN | Employee VLAN | Employee VLAN *(unchanged)* |
| Allowed VLANs | Employee VLAN only | **All** |
| PoE | Enabled | Enabled *(unchanged)* |
| Effect | Carries one SSID | **Carries all SSIDs** |

---

### Firewall Rule Design Patterns

| Scenario | Rule | Notes |
|----------|------|-------|
| Block all guest → employee | Deny Any `[guest]` → `[employee]` | Catch-all — use as minimum |
| Block guest → specific host | Deny Any `[guest]` → `[host]/32` | More granular, add above catch-all |
| Allow guest → printer | Allow TCP `[guest]` → `[printer]/32` port `9100` | Insert above catch-all deny |
| Allow guest → internet only | Deny Any `[guest]` → `10.0.0.0/8` + Allow Any Any → Any | Blocks all RFC1918, allows internet |

---

### SSID Client IP Assignment Modes

| Mode | DHCP Source | VLAN Tagging | Use Case |
|------|------------|--------------|----------|
| **NAT mode** | MR AP (internal) | ❌ Not available | Simple isolated guest — no VLAN needed |
| **Bridge mode** *(External DHCP)* | MX or external DHCP server | ✅ Available | VLAN-segmented networks — **use this** |
| **Layer 3 roaming** | MX | ✅ Available | Multi-AP environments with roaming requirements |

---

## 6. Troubleshooting

### VLAN Not Appearing on DHCP Page

> **Symptom:** You created the guest VLAN in Addressing & VLANs but it doesn't appear under DHCP configuration.

**Cause:** The Meraki MX only shows a VLAN on the DHCP page after it is assigned to at least one physical LAN port. This is undocumented behavior.

**Fix:** Assign the VLAN to any unused MX LAN port under `Addressing & VLANs → MX Ports`. The port does not need to be physically connected. Then return to the DHCP page — your VLAN will now appear.

---

### VLAN Tagging Greyed Out on SSID

> **Symptom:** The VLAN tagging field on the SSID access control page is uneditable (greyed out).

**Cause:** The SSID is in **NAT mode**. VLAN tagging is only available in Bridge mode.

**Fix:** Change **Client IP assignment** to **"External DHCP server assigned"** (Bridge mode). Save, then return — the VLAN tagging field will now be editable.

---

### Save Error: "vlan tag 0: must not be empty"

> **Symptom:** Saving the SSID VLAN configuration produces this error.

**Cause:** A row exists in the AP tag VLAN table with no AP tag selected. The row is present but incomplete.

**Fix:** Delete any rows with an empty AP tag field. Enter your VLAN ID in the **Default VLAN** field instead. Only add AP tag rows if you need to steer specific APs to different VLANs.

---

### Guest Client Gets Unexpected IP Address

> **Symptom:** Test device connects to the guest SSID but receives an IP address from the wrong subnet (or an entirely unexpected range).

**Cause — Option A:** Third-party network diagnostic apps may display the cellular interface IP instead of the WiFi interface IP, making it appear the device has the wrong address.  
**Fix:** Check via OS settings directly (see [Step 7a](#7a--confirm-dhcp-assignment)).

**Cause — Option B:** VLAN tagging is not actually applied — the SSID may still be in NAT mode or the default VLAN was not saved correctly.  
**Fix:** Return to `Wireless → SSIDs → [guest SSID] → Access Control` and confirm VLAN tagging shows **Enabled** with your correct VLAN ID in the Default field.

---

### Guest Can Still Ping the Gateway

> **Symptom:** Ping to the guest VLAN gateway succeeds even with deny rules in place.

**Cause:** This is ***expected and correct behavior.*** Meraki outbound firewall rules control traffic passing *through* the MX to other subnets. They do not block traffic addressed *to the MX itself*. The gateway IP is on the guest subnet — guest clients reaching their own gateway is by design.

**This is not a security issue.** Verify instead that pings to employee subnet hosts are failing.

---

### Firewall Rules Not Blocking Employee Access

> **Symptom:** Guest client can reach employee hosts despite deny rules being in place.

**Possible causes and fixes:**

| Cause | How to Check | Fix |
|-------|-------------|-----|
| Protocol set to TCP instead of Any | Inspect the rule's Protocol column | Change to `Any` |
| Deny rule is below the default allow | Look at rule order | Move deny rule above the default allow |
| Guest client got an employee VLAN IP | Check the client's IP address | VLAN tagging not working — revisit Step 4 |
| Overlapping rule allowing the traffic | Review all rules above the deny | Remove or reorder conflicting rules |

---

### AP Not Serving Guest SSID After Port Change

> **Symptom:** The guest SSID doesn't appear on the air, or clients can't connect after converting AP ports to trunk.

**Possible causes:**

- The AP port native VLAN was changed during the trunk conversion — the AP lost its management IP and went offline
- The AP port allowed VLANs does not include the employee VLAN — AP can't reach the Dashboard

**Fix:** Verify the AP port native VLAN is set to the **employee VLAN** (not the guest VLAN, not VLAN 1). Verify allowed VLANs includes both employee and guest VLANs (or is set to All). The AP must reach Dashboard via the employee VLAN to stay online and push SSID config.

---

## 7. Lessons Learned

> These are undocumented Meraki behaviors encountered during production implementation. Read before starting.

---

### 🔴 Lesson 1 — MX DHCP Page Only Shows VLANs Assigned to a Physical Port

~~Creating a VLAN in Addressing & VLANs makes it immediately configurable in DHCP.~~

**Reality:** A VLAN must be assigned to at least one physical MX LAN port before it appears on the DHCP configuration page. The port does not need a cable. This is not documented anywhere in Meraki's official documentation. Without this step, you will waste significant time looking for a DHCP configuration section that simply will not appear.

**Workaround:** Assign the new VLAN to any unused MX port (software-only, no cable needed). The VLAN then appears on the DHCP page immediately.

---

### 🔴 Lesson 2 — NAT Mode Silently Blocks VLAN Tagging

~~VLAN tagging is always configurable on any SSID.~~

**Reality:** The VLAN tagging option is completely greyed out and inaccessible when an SSID is in NAT mode. The UI provides no explanation. You must switch to **Bridge mode ("External DHCP server assigned")** first. Only then does the VLAN tagging field become editable.

**Always check Client IP Assignment mode before attempting to configure VLAN tagging.**

---

### 🟡 Lesson 3 — VLAN Profiles Are Not Required for Basic Guest VLANs

Early troubleshooting may lead to exploring VLAN Profiles (`Organization → Early Access → Named VLANs`). These are **not** required and are **not** the cause of DHCP page visibility issues.

~~VLAN Profiles must be configured to make a guest VLAN work on the MX.~~

**Reality:** VLAN Profiles are a separate feature designed for RADIUS-based dynamic VLAN assignment. They have no bearing on basic MX VLAN/DHCP configuration. Do not create them unless you specifically need RADIUS-driven VLAN assignment.

---

### 🟡 Lesson 4 — Firewall Protocol Field May Default to TCP

When adding L3 outbound firewall rules, the Protocol field may default to **TCP**. A TCP-only deny rule does **not** block ICMP or UDP traffic — guest clients could still ping protected hosts.

**Always explicitly verify Protocol is set to `Any` on deny rules.**

---

### 🟢 Lesson 5 — AP Trunk Ports Must Keep Employee VLAN as Native

When converting AP ports from access to trunk, it is tempting to set native VLAN to the guest VLAN or to VLAN 1. Either will cause the AP to lose Dashboard connectivity and go offline.

**The AP's native VLAN must remain the employee VLAN.** The AP uses the native (untagged) VLAN for its own management traffic to reach the Dashboard. Guest client traffic is carried as tagged frames on the guest VLAN — the AP handles the tagging, not the native VLAN.

---

## 8. Checklist

### Configuration Checklist

- [ ] Guest VLAN created — subnet, gateway, VPN disabled
- [ ] Guest VLAN assigned to an unused MX LAN port — unlocks DHCP page
- [ ] Confirmed guest VLAN now visible on DHCP page
- [ ] DHCP configured for guest VLAN — pool, DNS, lease time
- [ ] Guest SSID Client IP Assignment set to Bridge mode (External DHCP)
- [ ] Guest SSID VLAN tagging enabled — Default VLAN set to guest VLAN ID
- [ ] All AP-facing MS switch ports converted to Trunk
- [ ] AP port native VLAN = employee VLAN
- [ ] AP port allowed VLANs = All (or explicitly includes both VLANs)
- [ ] AP port PoE = Enabled
- [ ] L3 outbound firewall deny rule added — guest subnet → employee subnet
- [ ] Firewall rule Protocol = **Any** (not TCP)
- [ ] Firewall deny rule positioned **above** default allow rule

### Verification Checklist

- [ ] Test device connects to guest SSID
- [ ] Test device receives IP in guest subnet range
- [ ] Gateway IP confirmed in guest subnet range
- [ ] Internet access works from guest device
- [ ] `ping [employee-host]` fails from guest device
- [ ] `ping [employee-subnet-gateway]` fails from guest device *(except MX — see note)*
- [ ] DHCP lease table in Dashboard shows guest device with correct IP
- [ ] Employee SSID still works — no disruption to existing network

### Rollback Checklist

> Use this if something goes wrong and you need to restore the original state.

- [ ] Revert AP ports back to access mode, VLAN = employee VLAN
- [ ] Confirm all APs back online in Dashboard (green)
- [ ] Remove guest SSID VLAN tagging — set back to NAT mode
- [ ] Remove guest VLAN from unused MX port assignment
- [ ] Remove guest DHCP scope
- [ ] Delete guest VLAN from Addressing & VLANs
- [ ] Remove guest isolation firewall rules
- [ ] Confirm employee network fully operational

---

> **Document Version:** 1.0  
> **Platform:** Cisco Meraki MX + MS + MR  
> **Verified in production — March 2026**  
> *All undocumented behaviors confirmed across multiple implementation attempts.*
