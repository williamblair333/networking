# 🏢 HQ Network Modernization — Cisco Catalyst 3850 Deployment
### Single-Switch Consolidation with Meraki MX Uplink
> **Published:** March 2026 &nbsp;|&nbsp; **Status:** ✅ Production Verified &nbsp;|&nbsp; **Platform:** Cisco IOS XE 16.12 + Meraki MX

---

## 📋 Table of Contents

- [Overview](#overview)
- [Use Case](#use-case)
- [Before & After](#before--after)
  - [What Was Retired](#what-was-retired)
  - [What Was Deployed](#what-was-deployed)
- [Network Topology](#network-topology)
  - [Physical Layout](#physical-layout)
  - [Logical Design](#logical-design)
- [VLAN Design](#vlan-design)
- [Switch Configuration](#switch-configuration)
  - [Platform Notes](#platform-notes)
  - [Port Map](#port-map)
  - [Annotated Config](#annotated-config)
- [Critical IOS XE Gotchas](#critical-ios-xe-gotchas)
  - [IP Routing Is Enabled by Default](#1--ip-routing-is-enabled-by-default)
  - [Meraki Uplink Requires bpdufilter](#2--meraki-uplink-requires-bpdufilter)
  - [portfast trunk Alone Is Not Enough](#3--portfast-trunk-alone-is-not-enough)
- [Troubleshooting Guide](#troubleshooting-guide)
  - [PVID_Inc Blocking](#pvid_inc-blocking)
  - [ARP Resolves but Ping Fails](#arp-resolves-but-ping-fails)
  - [VLAN101 Line Protocol Down](#vlan101-line-protocol-down)
- [Lessons Learned](#lessons-learned)
- [Phase 2 Staging — Netgate pfSense](#phase-2-staging--netgate-pfsense)
- [Reference — Show Commands](#reference--show-commands)
- [Appendix — Environment Details](#appendix--environment-details)

---

## Overview

This guide documents the **complete process of consolidating a multi-switch HQ environment into a single Cisco Catalyst 3850**, connected upstream to a Meraki MX firewall appliance. The implementation replaced three aging switches with one modern platform while staging 10G uplinks for a future pfSense migration.

The guide is structured as a **practical how-to** with annotated configuration, a troubleshooting log of every issue encountered, and the lessons learned from each. It is applicable to any environment running a Cisco IOS XE access/distribution switch behind a Meraki MX.

---

## Use Case

### Environment Profile

| Attribute | Value |
|-----------|-------|
| **Site type** | Single-building headquarters |
| **Firewall** | Meraki MX (any model) |
| **Upstream topology** | MX → single L2 switch → endpoints |
| **VLAN count** | 4 (employee, wireless SSIDs, guest) |
| **Switch role** | Pure L2 — no inter-VLAN routing |
| **Management** | SSH only — HTTP/HTTPS disabled |
| **Future plan** | pfSense replacing MX, 10G uplinks to firewall |

### Why a 3850?

| Feature | Benefit |
|---------|---------|
| 48x PoE+ ports | Powers APs and IP phones without injectors |
| 4x 10G SFP+ | Future-ready — stage trunks to pfSense now |
| IOS XE | Modern, maintained codebase |
| Single RU | Replaces multiple switches in the same rack space |
| No stacking required | Full port density from one unit |

---

## Before & After

### What Was Retired

Three switches removed from production:

| Device | Model | Role |
|--------|-------|------|
| ~~Distribution Switch~~ | Meraki MS120-24P | L2 distribution — 24 PoE ports |
| ~~Access Switch A~~ | Cisco WS-C2960-48 | 48 access ports |
| ~~Access Switch B~~ | Cisco WS-C2960-24 | 24 access ports — redundant |

> ***The 24-port 2960 was entirely redundant.** With the MS120 already providing 24 ports and the 48-port 2960 providing 48, removing the 24-port and replacing everything with one 48-port PoE+ switch results in no net port loss.*

---

### What Was Deployed

| Device | Model | Management IP | Role |
|--------|-------|--------------|------|
| **hq-sw-01** | Cisco WS-C3850-48P | `192.168.10.252` | Sole HQ switch |

---

## Network Topology

### Physical Layout

```
                    ┌─────────────────────────────────────────┐
                    │            Meraki MX Firewall            │
                    │         Gateway: 192.168.10.1            │
                    │                                          │
                    │   Port 2: (reserved)                     │
                    │   Port 3: (legacy — temp)                │
                    │   Port 4: (free)                         │
                    │   Port 5: ──────────────────────────►   │
                    └─────────────────────────────────────────┘
                                         │
                                   Gi1/0/48
                             Trunk · Native VLAN 101
                             bpdufilter + portfast trunk
                                         │
                    ┌────────────────────┴────────────────────┐
                    │         Cisco Catalyst 3850-48P          │
                    │               hq-sw-01                   │
                    │        Mgmt: 192.168.10.252              │
                    │                                          │
                    │  Gi1/0/1–3   ── AP Trunks               │
                    │  Gi1/0/4–44  ── VLAN 101 Access         │
                    │  Gi1/0/45–47 ── AP Trunks               │
                    │  Gi1/0/48    ── MX Uplink               │
                    │  Te1/1/1–4   ── Future pfSense (shut)   │
                    └─────────────────────────────────────────┘
                          │         │         │
                       ┌──┴──┐   ┌──┴──┐   ┌──┴──┐
                       │ AP  │   │ AP  │   │ AP  │
                       └─────┘   └─────┘   └─────┘
```

---

### Logical Design

```
Internet
    │
    ▼
┌──────────────────────────────────────────────────────┐
│                  Meraki MX (L3 Gateway)               │
│  VLAN 101 → 192.168.10.1   Employee primary          │
│  VLAN 200 → 192.168.20.1   Employee wireless         │
│  VLAN 201 → 192.168.21.1   Training wireless         │
│  VLAN 202 → 192.168.22.1   Guest wireless (isolated) │
│  DHCP server for all VLANs                           │
│  IPsec VPN hub → remote sites                        │
└───────────────────────┬──────────────────────────────┘
                        │ 802.1Q Trunk — All VLANs
                        │ Native VLAN 101
                        ▼
           ┌────────────────────────┐
           │   hq-sw-01 (L2 Only)   │
           │   no ip routing        │
           │   Pure switched fabric │
           └────────────────────────┘
                 │            │
            VLAN 101     VLANs 200/201/202
            Employees    Wireless (tagged to APs)
```

> 💡 **No `ip helper-address` needed.** With `no ip routing` set, the 3850 is a pure L2 device. DHCP broadcasts flood natively up the trunk to the MX, which serves all DHCP scopes directly.

---

## VLAN Design

### VLAN Table

| VLAN | Name | Subnet | Gateway | Purpose |
|------|------|--------|---------|---------|
| **101** | Employee-Primary | 192.168.10.0/24 | 192.168.10.1 | Wired employees + switch management |
| **200** | Employee-Wireless | 192.168.20.0/24 | 192.168.20.1 | Employee SSID |
| **201** | Training-Wireless | 192.168.21.0/24 | 192.168.21.1 | Training SSID |
| **202** | Guest-Wireless | 192.168.22.0/24 | 192.168.22.1 | Guest SSID — firewall isolated |

### VLAN Flow on the Switch

```
AP Port (Trunk)
    │
    │  Tagged: VLAN 200, 201, 202
    │  Untagged (native): VLAN 101  ← AP management traffic
    ▼
hq-sw-01
    │
    │  All VLANs carried on uplink trunk
    ▼
MX Firewall
    │
    ├── VLAN 101 → route normally
    ├── VLAN 200 → route normally
    ├── VLAN 201 → route normally
    └── VLAN 202 → deny to internal subnets → internet only
```

---

## Switch Configuration

### Platform Notes

> ⚠️ **The Catalyst 3850 is fundamentally different from the 2960 series in two ways that affect this deployment:**

1. **L3 routing is enabled by default** on IOS XE. The switch will attempt to route traffic unless explicitly disabled. See [Critical IOS XE Gotchas](#critical-ios-xe-gotchas).
2. **IOS XE PVST+ handles Meraki RSTP BPDUs differently** than 2960 IOS. The `bpdufilter` workaround required is specific to IOS XE — document your Meraki firmware version if behavior differs.

---

### Port Map

#### Access Ports

| Range | Mode | VLAN | PortFast | BPDU Guard | PoE |
|-------|------|------|----------|-----------|-----|
| Gi1/0/4–44 | Access | 101 | ✅ | ✅ | ✅ |

#### AP Trunk Ports

| Range | Mode | Native VLAN | Allowed VLANs | PortFast |
|-------|------|------------|--------------|----------|
| Gi1/0/1–3 | Trunk | 101 | All | Trunk |
| Gi1/0/45–47 | Trunk | 101 | All | Trunk |

> 💡 **AP port native VLAN must be the employee VLAN.** The AP uses the native (untagged) VLAN for its own management traffic to reach the wireless controller or cloud dashboard. Wireless client traffic from each SSID is tagged with its respective VLAN ID. Setting native VLAN to anything other than the management VLAN will take the AP offline.

#### Uplink to Firewall

| Port | Mode | Native VLAN | Special Config |
|------|------|------------|---------------|
| Gi1/0/48 | Trunk | 101 | `bpdufilter enable` + `portfast trunk` |

#### Future Firewall Trunks (10G)

| Range | Mode | Native VLAN | Status |
|-------|------|------------|--------|
| Te1/1/1–4 | Trunk | 101 | **Shutdown — activate in Phase 2** |

---

### Annotated Config

```
! ════════════════════════════════════════════════════════════════════
! hq-sw-01 — Cisco WS-C3850-48P — IOS XE 16.12.8
! Single-switch HQ consolidation — Meraki MX upstream
! ════════════════════════════════════════════════════════════════════

hostname hq-sw-01

! ── VTP ──────────────────────────────────────────────────────────────
! Server mode: this is the only switch — manages its own VLAN database
! Pruning: best practice even with single switch; ready for expansion
vtp mode server
vtp pruning

! ── VLANs ────────────────────────────────────────────────────────────
vlan 101
 name Employee-Primary
vlan 200
 name Employee-Wireless
vlan 201
 name Training-Wireless
vlan 202
 name Guest-Wireless

! ── Spanning Tree ────────────────────────────────────────────────────
! Rapid-PVST+: Cisco standard — faster convergence than classic STP
! portfast default: enables PortFast on all access ports globally
! bpduguard default: err-disables any access port that receives a BPDU
! Priority 4096: lock this switch as root — prevent rogue root election
spanning-tree mode rapid-pvst
spanning-tree extend system-id
spanning-tree portfast default
spanning-tree portfast bpduguard default
spanning-tree vlan 101,200,201,202 priority 4096

! ── CRITICAL: Disable IP Routing ────────────────────────────────────
! 3850 enables ip routing by default. Disable for pure L2 operation.
! Without this: ARP resolves but pings fail — routing table has no path
no ip routing
ip default-gateway 192.168.10.1

! ── Management SVI ───────────────────────────────────────────────────
interface Vlan1
 no ip address
 shutdown

interface Vlan101
 description Management
 ip address 192.168.10.252 255.255.255.0
 no shutdown

! ── Security ─────────────────────────────────────────────────────────
no ip http server
no ip http secure-server
service password-encryption
enable secret 9 <strong-hash>
username admin privilege 15 secret 9 <strong-hash>

! ── SSH ──────────────────────────────────────────────────────────────
no ip domain lookup
ip domain name your-domain.example
ip ssh version 2
ip ssh time-out 60
ip ssh authentication-retries 3
! Run interactively after hostname + domain are set:
! crypto key generate rsa modulus 2048

line con 0
 login local
line vty 0 4
 login local
 transport input ssh
line vty 5 15
 login local
 transport input ssh

! ── SNMP (optional) ──────────────────────────────────────────────────
! Use SNMPv3 in new deployments — v2c shown for legacy compatibility
snmp-server community YOUR-COMMUNITY-STRING RO

! ── MX Uplink ────────────────────────────────────────────────────────
! bpdufilter: required — Meraki RSTP BPDUs cause PVID_Inc blocking
!   on IOS XE PVST+ without this. Safe because MX is L3, not a switch.
! portfast trunk: eliminates 30s STP delay on a router uplink
interface GigabitEthernet1/0/48
 description Trunk-to-MX-Firewall
 switchport trunk native vlan 101
 switchport mode trunk
 spanning-tree portfast trunk
 spanning-tree bpdufilter enable
 no shutdown

! ── AP Trunk Ports ───────────────────────────────────────────────────
! Native VLAN 101: AP management traffic — must match employee VLAN
! portfast trunk: eliminates STP delay for AP uplinks (no loop risk)
! No bpduguard on trunk ports
interface range GigabitEthernet1/0/1-3
 description AP-Trunk
 switchport trunk native vlan 101
 switchport mode trunk
 spanning-tree portfast trunk
 no shutdown

interface range GigabitEthernet1/0/45-47
 description AP-Trunk
 switchport trunk native vlan 101
 switchport mode trunk
 spanning-tree portfast trunk
 no shutdown

! ── Access Ports ─────────────────────────────────────────────────────
! portfast: individual port command redundant with global default,
!   but explicit — shows intent clearly in the config
! bpduguard: err-disables port if a switch/hub is connected
interface range GigabitEthernet1/0/4-44
 switchport mode access
 switchport access vlan 101
 spanning-tree portfast
 spanning-tree bpduguard enable
 no shutdown

! ── Future pfSense Trunks (10G) ──────────────────────────────────────
! Pre-configured and shutdown — activate when pfSense is deployed
! Trunk config already applied — just `no shutdown` when ready
interface TenGigabitEthernet1/1/1
 description Future-pfSense-Trunk
 switchport trunk native vlan 101
 switchport mode trunk
 shutdown

interface TenGigabitEthernet1/1/2
 description Future-pfSense-Trunk
 switchport trunk native vlan 101
 switchport mode trunk
 shutdown

interface TenGigabitEthernet1/1/3
 description Future-pfSense-Trunk
 switchport trunk native vlan 101
 switchport mode trunk
 shutdown

interface TenGigabitEthernet1/1/4
 description Future-pfSense-Trunk
 switchport trunk native vlan 101
 switchport mode trunk
 shutdown

end

! Save — IOS XE does NOT auto-save
copy running-config startup-config
```

---

## Critical IOS XE Gotchas

> These issues are **unique to the Catalyst 3850 / IOS XE platform** and will not be encountered on a 2960. Read before starting.

---

### 1 — IP Routing Is Enabled by Default

**The 3850 is an L3 switch.** Unlike the 2960, IOS XE enables `ip routing` out of the box.

#### Symptoms of This Problem

| Symptom | Misleading Because |
|---------|-------------------|
| `show arp` resolves the gateway MAC correctly | Looks like L2 is working |
| Ping from the switch to gateway fails 100% | Looks like a trunk/STP problem |
| `ping [gateway] source vlan 101` also fails | Looks like STP is still blocking |
| Traffic from connected hosts also fails | Looks like a VLAN problem |

#### Why It Happens

With `ip routing` enabled, the switch processes ICMP packets through the routing engine. The routing table has no valid path to the gateway (because the switch isn't configured as a router), so packets are dropped silently. ARP operates independently of routing and resolves correctly regardless.

#### The Fix

```
conf t
no ip routing
end
```

Verify:
```
show ip route
! Should show no routes (or only connected/local)
ping 192.168.10.1
! Should now succeed
```

---

### 2 — Meraki Uplink Requires bpdufilter

#### The Problem

Meraki MX appliances send **IEEE 802.1D/802.1w standard RSTP BPDUs**. Cisco PVST+ (used on IOS XE) expects BPDUs with a per-VLAN Cisco proprietary tag. When the 3850 receives an untagged IEEE RSTP BPDU on a PVST+ trunk, it detects what it interprets as a PVID inconsistency and **blocks the port**.

```
! Symptom in show spanning-tree vlan 101:
Interface           Role Sts Cost      Prio.Nbr Type
Gi1/0/48            Desg BKN*4         128.48   P2p *PVID_Inc
                              ^^^
                         PORT BLOCKED
```

#### Why bpdufilter Fixes It

`spanning-tree bpdufilter enable` stops the port from **sending or processing BPDUs entirely**. Since the MX is a router — not a switch — STP should not cross this boundary. bpdufilter effectively removes the uplink from the STP domain, which is correct.

```
! After applying bpdufilter:
Interface           Role Sts Cost      Prio.Nbr Type
Gi1/0/48            Desg FWD 4         128.48   P2p Edge
                              ^^^
                         FORWARDING ✅
```

> ⚠️ **Do not use bpdufilter on switch-to-switch links.** It is only safe here because the MX is a Layer 3 router. Using it between switches disables STP loop protection.

---

### 3 — portfast trunk Alone Is Not Enough

`spanning-tree portfast trunk` moves the port to **Edge** state but does **not** resolve the `PVID_Inc` error on IOS XE. The port will still show `BKN*` (blocking) with `PVID_Inc` after portfast trunk is applied.

**Both commands are required on the MX uplink:**

```
spanning-tree portfast trunk    ! Eliminates 30s STP delay
spanning-tree bpdufilter enable ! Resolves PVID_Inc blocking
```

---

## Troubleshooting Guide

### PVID_Inc Blocking

> **Full symptom:** Port is up/up, trunk is negotiated, native VLANs match on both sides, but STP shows `BKN* PVID_Inc`

```
! Confirm the issue:
show spanning-tree vlan 101
! Look for: BKN* and PVID_Inc on the uplink port

! Fix:
conf t
interface GigabitEthernet1/0/48
 spanning-tree portfast trunk
 spanning-tree bpdufilter enable
end

! Verify:
show spanning-tree vlan 101
! Expect: Desg FWD, Edge type, no PVID_Inc
```

---

### ARP Resolves but Ping Fails

> **Full symptom:** `show arp` shows the gateway MAC address, but `ping [gateway]` fails 100%

```
! Confirm ARP is working:
show arp
! If gateway MAC is present — L2 is working

! Check if ip routing is the cause:
show ip route
! If you see routes being processed — this is the problem

! Fix:
conf t
no ip routing
end

! Verify:
ping 192.168.10.1
! Should now succeed
```

---

### VLAN101 Line Protocol Down

> **Full symptom:** `Vlan101 is up, line protocol is down` even with trunk configured

**Cause:** No active ports exist in VLAN 101 yet. The SVI line protocol mirrors the state of the VLAN — if no physical port is up and assigned to VLAN 101 (either as access port or trunk member), the SVI stays down.

**Resolution path:**
1. Check trunk state: `show interfaces trunk` — is the uplink port in STP forwarding state?
2. If STP is blocking: resolve `PVID_Inc` per above
3. If trunk is forwarding but VLAN101 still down: check that at least one port is up in VLAN 101

```
! After resolving STP:
show interfaces Vlan101
! Expect: Vlan101 is up, line protocol is up
```

---

## Lessons Learned

> Five things that would have saved hours if known before starting.

---

### 🔴 Lesson 1 — Check `no ip routing` First on Any 3850 Deployment

If you're deploying a 3850 as a pure L2 switch, `no ip routing` is the first command you should run after basic VLAN setup. It prevents a category of confusing failures where the switch appears to have working L2 but broken connectivity.

---

### 🔴 Lesson 2 — Meraki Uplinks Need bpdufilter on IOS XE

The `PVID_Inc` blocking issue when connecting IOS XE PVST+ to a Meraki MX is a known incompatibility. The fix is `spanning-tree bpdufilter enable` on the uplink port. This is safe because the MX is a router. Document this explicitly in your configs so future engineers understand why it's there.

---

### 🟡 Lesson 3 — Meraki Dashboard Does Not Auto-Save

Changes in the Meraki Dashboard are **not saved until you explicitly click Save/Update**. Navigating away discards your changes silently. Always verify the port configuration reflects your intent after navigating back to the port list. This caused a significant troubleshooting detour in this deployment.

---

### 🟡 Lesson 4 — Always Verify Both Ends of a Trunk Before Deep Troubleshooting

Before investigating STP, VLANs, or routing issues on a new trunk, confirm both endpoints are correctly configured and saved. A misconfigured remote end is faster to find than to diagnose from the local side.

---

### 🟢 Lesson 5 — VTP Revision Check Is Mandatory on Refurbished Hardware

Always run `show vtp status` on a refurbished or used switch before connecting it to any trunk. A switch with a VTP revision number higher than your existing switches will silently overwrite the VLAN database across your entire domain the moment a trunk comes up — with no warning or confirmation prompt.

```
! Always run before first trunk connection:
show vtp status
! If revision > 0 and domain matches yours:
conf t
vtp mode transparent
! Or reset revision by changing domain name and changing back
```

---

## Phase 2 Staging — Netgate pfSense

The 10G SFP+ ports (Te1/1/1–4) are pre-configured as trunks and shut down. When pfSense is deployed to replace the MX:

```
! Activate when pfSense is physically connected:
conf t
interface TenGigabitEthernet1/1/1
 no shutdown
end
```

No other switch configuration changes are needed. The trunk config (native VLAN 101, all VLANs allowed) is already applied and will activate immediately when the port comes up.

> ✅ **pfSense note:** Unlike Meraki, pfSense does not send any BPDUs. The `bpdufilter` on the current MX uplink (Gi1/0/48) can be removed when pfSense replaces the MX — though leaving it in place is also harmless.

---

## Reference — Show Commands

### Daily Health Check

```bash
# All interface states at a glance
show interfaces status

# Confirm all VLANs present
show vlan brief

# Confirm trunk is up and carrying correct VLANs
show interfaces trunk

# STP root confirmation and port states
show spanning-tree vlan 101

# Interface error counters — watch for CRC, input errors
show interfaces counters errors

# CPU load
show processes cpu sorted | head 20

# ARP table — verify gateway reachable
show arp

# Active management sessions
show users
```

### Trunk Verification Checklist

```
show interfaces trunk
```

Expected output — uplink port:

```
Port        Mode             Encapsulation  Status        Native vlan
Gi1/0/48    on               802.1q         trunking      101

Port        Vlans allowed on trunk
Gi1/0/48    1-4094

Port        Vlans allowed and active in management domain
Gi1/0/48    101,200,201,202

Port        Vlans in spanning tree forwarding state and not pruned
Gi1/0/48    101,200,201,202
```

### BPDU Guard Recovery

If an access port gets err-disabled by BPDU Guard (a switch or hub was connected):

```bash
# Identify the affected port
show interfaces status | include err

# Manual recovery after removing offending device
conf t
interface GigabitEthernet1/0/X
 shutdown
 no shutdown
end

# Optional: automatic recovery after 5 minutes
conf t
errdisable recovery cause bpduguard
errdisable recovery interval 300
```

---

## Appendix — Environment Details

### Example IP Scheme

> 📝 Substitute your own values. These are illustrative only.

| Parameter | Example Value |
|-----------|--------------|
| **Employee subnet** | 192.168.10.0/24 |
| **Employee gateway** | 192.168.10.1 |
| **Switch management IP** | 192.168.10.252 |
| **Wireless subnet** | 192.168.20.0/24 |
| **Training subnet** | 192.168.21.0/24 |
| **Guest subnet** | 192.168.22.0/24 |

### Hardware Tested

| Component | Model | Version |
|-----------|-------|---------|
| Access switch | Cisco WS-C3850-48P | IOS XE 16.12.8 |
| Firewall | Meraki MX (any) | MX 14.x / 16.x |
| APs | Meraki MR series | Current |

### Compatibility Notes

| IOS XE Version | Known Issues |
|---------------|-------------|
| 16.12.x | Tested — all workarounds confirmed working |
| 16.9.x | `bpdufilter` workaround expected to work — untested |
| < 16.6 | Not recommended — upgrade before deployment |

---

## Checklist

### Pre-Deployment

- [ ] `show vtp status` — confirm revision = 0 or set transparent
- [ ] Confirm MX uplink port is configured as trunk, native VLAN correct, saved
- [ ] Identify management IP from available pool
- [ ] RSA key generated: `crypto key generate rsa modulus 2048`

### Configuration

- [ ] `no ip routing` applied
- [ ] VLANs created and named
- [ ] STP priority set — switch is root
- [ ] `bpdufilter enable` on MX uplink
- [ ] `portfast trunk` on MX uplink and AP ports
- [ ] `portfast` + `bpduguard` on all access ports
- [ ] SSH v2 configured — Telnet disabled
- [ ] HTTP/HTTPS disabled: `no ip http server` + `no ip http secure-server`
- [ ] Management SVI up/up

### Verification

- [ ] `show vlan brief` — all VLANs present
- [ ] `show interfaces trunk` — uplink trunking, correct native VLAN
- [ ] `show spanning-tree vlan 101` — this switch is root, uplink is FWD
- [ ] `ping [gateway]` — succeeds
- [ ] `ping [internet host]` — succeeds
- [ ] SSH login from a remote host — succeeds
- [ ] `copy running-config startup-config` — saved

---

> **Document Version:** 1.0
> **Platform:** Cisco Catalyst 3850 · IOS XE 16.12 · Meraki MX
> *All configurations verified in production. Issues and resolutions documented from live deployment.*
