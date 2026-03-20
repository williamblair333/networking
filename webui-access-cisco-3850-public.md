# Cisco Catalyst 3850 — Web UI Access Guide

**Platform:** WS-C3850-48F-L (IOS XE)  
**Purpose:** Enable and access the built-in browser-based management interface  
**Scope:** Initial setup through first login

---

## Table of Contents

- [Overview](#overview)
- [Prerequisites](#prerequisites)
- [Step 1 — Management IP](#step-1--management-ip)
- [Step 2 — Local User](#step-2--local-user)
- [Step 3 — Enable HTTP/HTTPS](#step-3--enable-httphttps)
- [Step 4 — Browser Access](#step-4--browser-access)
- [Hardening](#hardening)
- [Troubleshooting](#troubleshooting)
- [Known Limitations](#known-limitations)

---

## Overview

The 3850 runs IOS XE and includes a built-in Web UI accessible via HTTPS. It provides a dashboard view of interfaces, VLANs, port status, and basic diagnostics — useful for quick visual checks without dropping to CLI.

The Web UI is **read/write** but limited compared to CLI. Use it for monitoring and simple configuration. Complex VLAN, EtherChannel, or spanning-tree work should still be done via CLI.

---

## Prerequisites

| Requirement | Detail |
|-------------|--------|
| IOS XE version | 3.x or later (any current 3850 image) |
| Management reachability | Layer 3 path from your workstation to the switch SVI or GigabitEthernet0/0 |
| Local user | Privilege level **15** — lower levels produce a blank page or login error |
| Browser | Chrome or Firefox — avoid IE/Edge Legacy |

---

## Step 1 — Management IP

The 3850 has two management options:

### Option A — VLAN SVI (recommended for in-band management)

```
interface Vlan<mgmt-vlan>
 ip address 192.168.1.X 255.255.255.0
 no shutdown
!
ip default-gateway 192.168.1.1
```

Replace `192.168.1.X` with the address you're assigning and `<mgmt-vlan>` with your management VLAN ID.

---

### Option B — Dedicated Management Port (GigabitEthernet0/0)

The 3850 has a dedicated out-of-band management port on the rear panel. It runs in a separate VRF (`Mgmt-vrf`) and is isolated from the data plane.

```
interface GigabitEthernet0/0
 vrf forwarding Mgmt-vrf
 ip address 192.168.1.X 255.255.255.0
 no shutdown
!
ip route vrf Mgmt-vrf 0.0.0.0 0.0.0.0 192.168.1.1
```

> **Note:** If you use Gi0/0, your browser must reach the switch via that interface's subnet. SSH/HTTP commands that don't specify a VRF won't work from the data plane without additional config. For simplicity, Option A (SVI) is easier to manage alongside existing infrastructure.

---

## Step 2 — Local User

The Web UI requires a local user at **privilege 15**. Use `algorithm-type scrypt` for modern password hashing (IOS XE 16.x+):

```
username admin privilege 15 algorithm-type scrypt secret <password>
```

If your IOS XE version doesn't support `algorithm-type scrypt`, fall back to:

```
username admin privilege 15 secret <password>
```

Verify the privilege level:

```
show running-config | include username
```

Expected output:
```
username admin privilege 15 algorithm-type scrypt secret 9 $9$...
```

---

## Step 3 — Enable HTTP/HTTPS

```
ip http server
ip http secure-server
ip http authentication local
ip http max-connections 16
```

Save:

```
copy running-config startup-config
```

> `ip http server` enables plain HTTP (port 80). Enable it during initial setup for browser compatibility testing, then disable it once HTTPS is confirmed working. See [Hardening](#hardening).

Verify:

```
show ip http server status
```

Look for:
```
HTTP server status: Enabled
HTTP secure server status: Enabled
```

---

## Step 4 — Browser Access

Navigate to:

```
https://192.168.1.X
```

**A self-signed certificate warning is expected.** The 3850 generates a self-signed cert at boot. Accept the exception in your browser:

- Chrome: `Advanced → Proceed to 192.168.1.X (unsafe)`
- Firefox: `Advanced → Accept the Risk and Continue`

Login with the credentials from Step 2.

**Landing page after login:**

| Section | What It Shows |
|---------|--------------|
| Dashboard | CPU/memory utilization, uptime, interface summary |
| Configuration | VLAN, interface, IP settings |
| Administration | System info, users, logging |
| Monitoring | Interface counters, MAC table, ARP table |

---

## Hardening

Once Web UI access is confirmed working:

```
no ip http server
ip http access-class ipv4 99
access-list 99 permit 192.168.1.0 0.0.0.255
access-list 99 deny   any
```

This disables plain HTTP and restricts HTTPS access to management subnet hosts only. Adjust the ACL to match your actual management subnet.

> ⚠️ If you apply `ip http access-class` and your workstation is outside the permitted range, you'll lock yourself out of the Web UI. Verify the ACL before applying.

---

## Troubleshooting

### Browser returns "Connection Refused"

```
show ip http server status
```

If HTTP secure server shows `Disabled`, re-run:
```
ip http secure-server
```

---

### Login accepted but blank page loads

Almost always a privilege level issue. Confirm:

```
show running-config | include username
```

The user must show `privilege 15`. If not, fix it:

```
username admin privilege 15 secret <password>
```

---

### Can't reach the switch IP at all

```
show interface Vlan<mgmt-vlan>
show ip interface brief
ping <default-gateway>
```

Check:
- SVI is `up/up`
- Default gateway is reachable
- No ACL on VTY or HTTP blocking the source

---

### Certificate error won't dismiss

A few browsers (especially corporate-managed Chrome installs) block self-signed certs on internal IPs without offering a bypass. Use Firefox as a fallback — it provides an explicit "Accept the Risk" override.

---

## Known Limitations

| Limitation | Detail |
|------------|--------|
| EtherChannel config | Not configurable via Web UI — CLI only |
| Spanning-tree tuning | Priority, port cost, BPDU guard — CLI only |
| SNMP | Configurable in Web UI but limited; use CLI for full control |
| Stacking | Stack management (if stacked) is CLI only |
| ACL config | Basic permit/deny possible; complex ACLs use CLI |
| Session timeout | Web UI session idles out quickly; no persistent session config in GUI |
