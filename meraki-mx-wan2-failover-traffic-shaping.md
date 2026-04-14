# Meraki MX — Dual WAN, Failover & Traffic Shaping
## Adding a Secondary ISP, Configuring Automatic Failover, and Shaping Traffic Priority

> **Document Type:** How-To + Reference Guide
> **Platform:** Cisco Meraki MX Security Appliance (MX 14.x firmware)
> **Date:** March 2026
> **Status:** ✅ Complete — All steps verified in production
> **Audience:** Network administrators managing a Meraki MX in a hub-and-spoke VPN environment

---

## 📋 Table of Contents

- [Overview](#overview)
- [Environment & Prerequisites](#environment--prerequisites)
- [Architecture](#architecture)
  - [Before — Single WAN](#before--single-wan)
  - [After — Dual WAN with Failover](#after--dual-wan-with-failover)
  - [Port Map](#port-map)
- [Phase 1 — Convert LAN Port to WAN 2](#phase-1--convert-lan-port-to-wan-2)
  - [Step 1 — Convert LAN Port to WAN 2](#step-1--convert-lan-port-to-wan-2)
  - [Step 2 — Verify WAN 2 Status in Dashboard](#step-2--verify-wan-2-status-in-dashboard)
- [Phase 2 — Configure Failover](#phase-2--configure-failover)
  - [Step 3 — SD-WAN & Traffic Shaping Settings](#step-3--sd-wan--traffic-shaping-settings)
  - [Step 4 — Test Failover](#step-4--test-failover)
- [Phase 3 — Traffic Shaping](#phase-3--traffic-shaping)
  - [Step 5 — Protect Critical Traffic](#step-5--protect-critical-traffic)
  - [Step 6 — Cap Non-Critical Traffic](#step-6--cap-non-critical-traffic)
- [Platform Limitations — MX 14.x](#platform-limitations--mx-14x)
  - [No Per-Uplink Traffic Shaping](#no-per-uplink-traffic-shaping)
  - [No Application-Based SD-WAN Steering](#no-application-based-sd-wan-steering)
- [VPN Failover Behavior](#vpn-failover-behavior)
  - [Meraki Auto VPN — No Hard-Coded Peer IPs](#meraki-auto-vpn--no-hard-coded-peer-ips)
  - [Expired License Warning](#expired-license-warning)
- [Firmware Note — MX 14.56](#firmware-note--mx-1456)
- [Reference Tables](#reference-tables)
  - [Traffic Shaping Rules Summary](#traffic-shaping-rules-summary)
  - [WAN Uplink Summary](#wan-uplink-summary)
  - [MX Port Map](#mx-port-map)
- [Troubleshooting](#troubleshooting)
- [Lessons Learned](#lessons-learned)
- [Checklist](#checklist)

---

## Overview

This guide documents the complete process of adding a **secondary ISP connection** to a Meraki MX appliance, configuring **automatic WAN failover**, and implementing **traffic shaping rules** to prioritize critical business traffic during failover events.

The environment uses a **hub-and-spoke Auto VPN topology** — the MX at headquarters acts as the VPN hub for all remote sites. Secondary WAN provides internet continuity during primary ISP outages and during planned maintenance windows such as firmware upgrades.

### What This Guide Covers

- Converting an existing MX LAN port to a WAN 2 port (via Dashboard on MX80; via local status page on other MX models)
- Verifying secondary ISP DHCP assignment
- Configuring WAN 1 as primary with WAN 2 as standby failover
- Testing automatic failover and VPN tunnel renegotiation
- Shaping traffic to prioritize VoIP and business-critical applications
- Capping non-critical traffic (streaming, social media, gaming) on the secondary circuit
- Understanding MX 14.x traffic shaping limitations and workarounds

### What This Guide Does Not Cover

- Configuring the secondary ISP gateway device itself
- Per-uplink traffic shaping (not available on MX 14.x — see [Platform Limitations](#platform-limitations--mx-14x))
- Application-based SD-WAN steering (requires MX 15+)
- Load balancing across both uplinks

---

## Environment & Prerequisites

### Tested Hardware

| Role | Device | Firmware |
|------|--------|----------|
| Firewall / VPN Hub | Meraki MX appliance | MX 14.x |
| Distribution Switch | Meraki MS switch | Current |
| Remote Sites | Meraki MX appliances | Current |

### Prerequisites

- [ ] Meraki Dashboard access — **Organization Administrator** role
- [ ] Primary WAN (WAN 1) active and confirmed working
- [ ] Secondary ISP gateway device physically connected to the MX LAN port being converted
- [ ] Secondary ISP gateway confirmed powered on and serving DHCP
- [ ] Secondary ISP public IP address known (obtain from ISP or gateway admin page)
- [ ] **MX80:** Dashboard access is sufficient for port conversion
- [ ] **Other MX models:** LAN-connected device available to reach MX local status page (`http://[MX-LAN-IP]`) if Dashboard does not expose the conversion control
- [ ] Maintenance window scheduled — port conversion causes a brief service interruption

### Example Network Addressing

> 📝 Substitute your own values throughout this guide.

| Parameter | Example Value | Your Value |
|-----------|--------------|------------|
| **MX LAN Gateway** | `10.0.1.1` | __________ |
| **WAN 1 IP (static)** | `203.0.113.10` | __________ |
| **WAN 1 Gateway** | `203.0.113.9` | __________ |
| **WAN 1 Speed** | `250 / 250 Mbps` | __________ |
| **WAN 2 IP (DHCP from ISP2 gateway)** | `192.168.0.x` | __________ |
| **WAN 2 Public IP** | `198.51.100.50` | __________ |
| **WAN 2 Speed** | `50 / 25 Mbps` | __________ |
| **MX Port used for WAN 2** | `Port 2` | __________ |
| **MX Port used for LAN trunk** | `Port 3` | __________ |

---

## Architecture

### Before — Single WAN

```
┌─────────────────────────────────────────────┐
│             Primary ISP                      │
│         203.0.113.10 (static)                │
└─────────────────────┬───────────────────────┘
                      │ Internet 1
              ┌───────┴───────┐
              │   Meraki MX   │
              │               │
              │  Port 2: LAN  │ ◄── ISP2 gateway plugged in but inactive
              │  Port 3: LAN  │ ──► MS Switch trunk
              │  Port 4: LAN  │
              │  Port 5: LAN  │
              └───────┬───────┘
                      │ LAN
              ┌───────┴───────┐
              │   MS Switch   │
              └───────────────┘
```

### After — Dual WAN with Failover

```
┌───────────────────┐        ┌───────────────────┐
│   Primary ISP     │        │  Secondary ISP    │
│  203.0.113.10     │        │  198.51.100.50    │
│  (static)         │        │  (via gateway     │
│                   │        │   DHCP: 192.168.x)│
└─────────┬─────────┘        └────────┬──────────┘
          │ Internet 1                │ ISP2 Gateway
          │ (WAN 1)                   │
   ┌──────┴───────────────────────────┴──────┐
   │               Meraki MX                  │
   │                                          │
   │  WAN 1:  203.0.113.10  — Active         │
   │  WAN 2:  192.168.0.x   — Ready/Standby  │
   │                                          │
   │  Port 2: WAN 2 ◄── ISP2 Gateway         │
   │  Port 3: LAN Trunk ──► MS Switch        │
   │  Port 4: LAN (reserved)                 │
   │  Port 5: LAN (reserved)                 │
   └──────────────────┬───────────────────────┘
                      │ LAN
              ┌───────┴───────┐
              │   MS Switch   │
              └───────────────┘
```

### Port Map

| MX Port | Role | Connected To | Notes |
|---------|------|-------------|-------|
| Internet 1 | WAN 1 | Primary ISP | Static IP — do not touch |
| Port 2 | **WAN 2** | ISP2 Gateway (LAN port) | Converted from LAN in this guide |
| Port 3 | LAN Trunk | MS Switch uplink | Do not touch |
| Port 4 | LAN | Reserved | Free |
| Port 5 | LAN | Reserved | Free |

> ⚠️ **Critical:** Know your port assignments before starting. Converting the wrong port will drop your LAN trunk and take the network down. Verify Port 3 is the MS Switch uplink before touching Port 2.

---

## Phase 1 — Convert LAN Port to WAN 2

### Step 1 — Convert LAN Port to WAN 2

> **MX80:** Port conversion is available directly in the Meraki Dashboard — no local status page required.
> **Other MX models:** The Dashboard may not expose this control. Use the local status page method below.

#### Method A — Meraki Dashboard (MX80 and supported models)

**Dashboard path:** `Security & SD-WAN → Configure → Addressing & VLANs → MX Ports`

1. Locate the port connected to the ISP2 gateway
2. Change the port type from **LAN** to **WAN**
3. Set connection type: **DHCP**
4. Save

#### Method B — Local Status Page (other MX models)

1. From a **LAN-connected device**, open a browser and navigate to the MX local status page:

```
http://[your-MX-LAN-gateway-IP]
```

*Example: `http://10.0.1.1`*

2. Click the **Uplink** tab
3. Locate the port currently assigned as LAN that has the ISP2 gateway connected
4. Click **"Convert port 2 from LAN to WAN"**
5. Set connection type: **DHCP**
6. Click **Save**

> ✅ The port is now WAN 2. The ISP2 gateway will issue a DHCP address to the MX on this port. Confirm the gateway device is powered on and its LAN port is connected.

> 💡 To undo this at any time, the same control will read **"Revert port 2 from WAN to LAN"** (local status page) or allow you to change the port type back to LAN (Dashboard).

---

### Step 2 — Verify WAN 2 Status in Dashboard

**Dashboard path:** `Security & SD-WAN → Monitor → Appliance Status → Uplink tab`

Confirm the following:

| Uplink | Type | Status | IP Address |
|--------|------|--------|------------|
| WAN 1 | Static | **Active** | `203.0.113.10` (your static IP) |
| WAN 2 | Dynamic (DHCP) | **Ready** | `192.168.0.x` (from ISP2 gateway) |

> ℹ️ **"Ready" is correct.** WAN 2 status will show **Ready** (not Active) when WAN 1 is healthy. It transitions to Active only when WAN 1 fails. This is expected behavior.

> ⚠️ If WAN 2 shows **Not connected** or no IP address, verify: (1) ISP2 gateway is powered on, (2) cable is seated at both ends, (3) ISP2 gateway DHCP is enabled.

---

## Phase 2 — Configure Failover

### Step 3 — SD-WAN & Traffic Shaping Settings

**Dashboard path:** `Security & SD-WAN → Configure → SD-WAN & Traffic Shaping`

Configure the uplink settings:

| Setting | Value | Notes |
|---------|-------|-------|
| **Primary uplink** | WAN 1 | All traffic uses WAN 1 by default |
| **Load balancing** | Disabled | WAN 2 is standby only, not load balanced |
| **WAN 1 — Download** | 250 Mbps | Match your ISP contract speed |
| **WAN 1 — Upload** | 250 Mbps | Match your ISP contract speed |
| **WAN 2 — Download** | 50 Mbps | Match your ISP2 contract speed |
| **WAN 2 — Upload** | 25 Mbps | Match your ISP2 contract speed |

> 💡 **Why speed values matter:** The MX uses these values to calculate traffic shaping percentages and to make failover decisions. Inaccurate speeds result in suboptimal shaping behavior. Set them to match your actual contracted speeds, not theoretical maximums.

> ⚠️ **Do not enable load balancing** unless you have a specific reason. With load balancing enabled, traffic is split across both uplinks at all times, which can cause session persistence issues for VoIP and applications that are sensitive to IP changes mid-session.

Save when complete.

---

### Step 4 — Test Failover

> 🧪 Perform this test immediately after configuring failover, before closing the maintenance window.

**Test procedure:**

1. Confirm internet access is working normally on WAN 1
2. **Physically unplug** the WAN 1 cable from the MX Internet 1 port
3. Wait **~30 seconds**
4. From a LAN device, confirm internet access is still available
5. Check Dashboard → Appliance Status → Uplink tab:
   - WAN 1: should show **Failed** or **Not connected**
   - WAN 2: should show **Active**
6. Confirm VPN tunnels to remote sites have renegotiated (check Dashboard → VPN Status)
7. **Reconnect WAN 1 cable**
8. Wait ~30 seconds — traffic falls back to WAN 1 automatically
9. Confirm WAN 1 returns to **Active**, WAN 2 returns to **Ready**

**Expected failover time:** ~15–30 seconds from WAN 1 loss to WAN 2 active

**Expected fallback time:** ~15–30 seconds after WAN 1 restoration

> ✅ No manual intervention is required for failover or fallback. The MX handles both automatically.

---

## Phase 3 — Traffic Shaping

### Overview

The goal is to ensure that **critical business traffic** (VoIP, Microsoft 365, cloud services) receives priority bandwidth during normal operation and gets access to WAN 2 capacity during failover, while **non-critical traffic** (streaming, gaming, social media) is deprioritized and capped.

> ⚠️ **MX 14.x Limitation:** Traffic shaping rules apply **globally across both uplinks**. It is not possible to apply rules only during failover or only on WAN 2. See [Platform Limitations](#platform-limitations--mx-14x) for full details and the recommended workaround for next-generation hardware.

**Dashboard path:** `Security & SD-WAN → Configure → SD-WAN & Traffic Shaping → Traffic shaping rules`

> 💡 Set **Default Rules** to **"Disable default traffic shaping rules"**. The default rules apply DSCP tags only — they do not restrict bandwidth and add noise without contributing to the shaping goals here.

---

### Step 5 — Protect Critical Traffic

**Add Rule #1:**

| Field | Value |
|-------|-------|
| **Definition** | All VoIP & video conferencing + All Databases & cloud services + Microsoft OneDrive + Windows Live Hotmail and Outlook + Office 365 + SharePoint + Windows Live Office + Remote desktop + All Security + All Software & anti-virus updates |
| **Bandwidth limit** | Obey network per-client limit (unlimited / unlimited) |
| **Priority** | **High** |
| **DSCP tagging** | Do not change DSCP tag |

> 💡 Add each application category by clicking **Add +** in the Definition field. You can stack multiple categories into a single rule — they are evaluated with OR logic (traffic matching *any* of the listed categories is subject to the rule).

> 💡 **High priority** means this traffic wins contention over normal and low priority traffic during saturation. On a 50 Mbps failover circuit, voice calls and Microsoft 365 sessions will be preserved even if other traffic is competing for bandwidth.

---

### Step 6 — Cap Non-Critical Traffic

**Add Rule #2:**

| Field | Value |
|-------|-------|
| **Definition** | All Video & music + All Advertising + All Social web & photo sharing + All Gaming + All Web file sharing + All News + All Peer-to-peer (P2P) + All Sports + All Blogging |
| **Bandwidth limit** | **5 Mbps** (set via slider) |
| **Priority** | **Low** |
| **DSCP tagging** | Do not change DSCP tag |

> ⚠️ **Bandwidth limit field behavior:** The dropdown selector controls per-client limit behavior. The **slider below it** sets the actual Mbps cap. Confirm the slider reads your intended value (e.g., 5 Mbps) before saving — the dropdown label does not reflect the slider value.

> 💡 5 Mbps is sufficient to make streaming services technically functional but degraded on WAN 1 (250 Mbps) while making them largely unusable on WAN 2 (50 Mbps shared with all other traffic). Adjust this value to match your tolerance for streaming on the primary circuit.

**Rule priority (evaluation order):**

| # | Rule | Priority | Bandwidth |
|---|------|----------|-----------|
| 1 | Critical traffic (VoIP, M365, cloud) | High | Unlimited |
| 2 | Non-critical traffic (streaming, gaming, social) | Low | 5 Mbps cap |
| — | Everything else (general web browsing) | Normal | Remaining |

> 💡 General web browsing requires no explicit rule. It gets whatever bandwidth remains after critical traffic takes priority and non-critical traffic hits its cap.

---

## Platform Limitations — MX 14.x

### No Per-Uplink Traffic Shaping

> **This is a confirmed hardware ceiling for MX 14.x firmware.**

Traffic shaping rules on MX 14.x apply globally — there is no mechanism to restrict or shape traffic only when WAN 2 is active. The 5 Mbps streaming cap in Rule #2 applies at all times, on both WAN 1 and WAN 2.

**Impact:** Streaming is capped to 5 Mbps during normal operation on WAN 1, not only during failover. For most business environments this is acceptable. If unrestricted streaming on WAN 1 is required while still restricting it on WAN 2, the workaround is **next-generation hardware** running MX 15+.

---

### No Application-Based SD-WAN Steering

MX 14.x SD-WAN policies support only:

- VPN traffic uplink preferences
- Custom performance classes (latency, jitter, loss thresholds)

**Application-based uplink steering** — the ability to say "send video streaming only out WAN 1, with no failover to WAN 2" — requires **MX 15+ firmware**, which in turn requires newer MX hardware (MX 68, MX 85, MX 95, etc.).

On MX 14.x, the SD-WAN policies section will not contain any application or traffic-type steering controls. This is not a configuration error — it is a firmware version limitation tied to the hardware generation.

**Workaround on MX 14.x:** The 5 Mbps bandwidth cap with Low priority on non-critical traffic (Rule #2) is the closest available approximation. During WAN 2 failover on a 50 Mbps circuit, 5 Mbps for streaming combined with contention from high-priority critical traffic effectively degrades streaming to unusable.

**Resolution:** Full per-uplink traffic shaping is available on pfSense/OPNsense and on MX 15+ hardware. Schedule for Phase 2 hardware replacement.

---

## VPN Failover Behavior

### Meraki Auto VPN — No Hard-Coded Peer IPs

Meraki Auto VPN does **not** use static peer IP addresses in its configuration. Hub-and-spoke peer discovery is cloud-mediated — when the hub MX fails over to a new WAN IP, the Meraki cloud notifies all spoke devices of the updated peer address and tunnels renegotiate automatically.

**No manual intervention is required at remote sites during a failover event.** Tunnel renegotiation typically completes within 30–60 seconds of WAN 2 becoming active.

This is a significant operational advantage over traditional IPsec with static peer IPs, where a hub IP change would require manual reconfiguration at every spoke site.

---

### Expired License Warning

> ⚠️ **Read this before testing failover if any devices in your VPN topology have expired Meraki licenses.**

Auto VPN renegotiation is **cloud-mediated**. If a spoke-site MX has an expired license, it may lose cloud connectivity. A spoke that cannot reach the Meraki cloud cannot receive the updated hub IP when the hub fails over to WAN 2.

**Symptom:** WAN 2 failover succeeds at HQ, but VPN tunnels to one or more remote sites remain down until WAN 1 is restored.

**Check:** In Meraki Dashboard, verify all spoke devices show green (online) status before performing failover testing. If any spoke shows offline or degraded, renew the license before testing.

**Verify license status:** `Organization → License info`

---

## Firmware Note — MX 14.56

> This section applies specifically to MX 80 and other MX hardware with a **MX 14.x firmware ceiling**.

Some MX hardware models cannot run MX 15 or higher firmware regardless of dashboard prompts. If your MX hardware is at or near end-of-support, it is likely in this category.

**Why MX 14.56 specifically matters:**

MX 14.56 is the minimum required version for **AMP (Advanced Malware Protection)** and **ThreatGrid** cloud communications to function correctly. Running a firmware version below 14.56 on an AMP-enabled network will result in silent failures of threat intelligence services.

**Recommended procedure:**

1. Schedule firmware update during a maintenance window
2. Confirm WAN 2 is active and failover-tested before starting
3. Apply the MX 14.56 update via Dashboard → Network-wide → General → Firmware upgrades
4. The MX will reboot — internet continuity is maintained via WAN 2 during the reboot
5. After reboot, confirm WAN 1 returns to Active and VPN tunnels renegotiate

**Expected reboot time:** ~3–5 minutes

---

## Reference Tables

### Traffic Shaping Rules Summary

| Rule | Definition | Bandwidth | Priority | Effect |
|------|-----------|-----------|----------|--------|
| #1 | VoIP, M365, cloud services, security updates | Unlimited | **High** | Protected during bandwidth contention on either uplink |
| #2 | Video/music, gaming, social, P2P, ads | **5 Mbps cap** | **Low** | Capped globally — effectively unusable on 50 Mbps WAN 2 under load |
| — | General web browsing (no rule) | Remaining | Normal | Gets leftover capacity |

---

### WAN Uplink Summary

| Uplink | Type | Normal Status | Failover Status | Speed |
|--------|------|--------------|----------------|-------|
| WAN 1 | Static | **Active** | Failed | 250/250 Mbps |
| WAN 2 | DHCP (ISP2 gateway) | **Ready** | Active | 50/25 Mbps |

---

### MX Port Map

| Port | Role | Assignment | Notes |
|------|------|-----------|-------|
| Internet 1 | WAN 1 | Primary ISP | Static IP — do not modify |
| Port 2 | WAN 2 | ISP2 Gateway | Converted from LAN in this guide |
| Port 3 | LAN Trunk | Distribution switch uplink | Do not modify |
| Port 4 | LAN | Reserved | Available |
| Port 5 | LAN | Reserved | Available |

---

## Troubleshooting

### WAN 2 Shows "Not Connected" After Port Conversion

| Check | Action |
|-------|--------|
| ISP2 gateway powered on? | Confirm LEDs, power cycle if needed |
| Cable seated at both ends? | Reseat cable at MX Port 2 and gateway LAN port |
| Gateway DHCP enabled? | Log into gateway admin page and verify DHCP server is active |
| Correct port converted? | Dashboard → Addressing & VLANs → MX Ports (MX80) or local status page → Uplink tab — confirm Port 2 is listed as WAN |

---

### Failover Does Not Occur When WAN 1 Cable Unplugged

**Possible causes:**

- **Uplink health check not configured** — Dashboard may not be actively monitoring WAN 1 health. Check `SD-WAN & Traffic Shaping → Uplink connectivity monitoring`.
- **Primary uplink not set** — Verify WAN 1 is selected as primary uplink in SD-WAN settings.
- **WAN 2 speed not configured** — MX may not treat WAN 2 as a viable failover target without speed values set.

---

### VPN Tunnels Don't Renegotiate After Failover

| Check | Action |
|-------|--------|
| Spoke license expired? | `Organization → License info` — renew if expired |
| Spoke showing offline in Dashboard? | Check device status at each spoke site |
| Failover IP blocked by spoke-side ISP? | Confirm spoke-side ISP doesn't block inbound IPsec (UDP 500/4500) |
| Wait time | Allow 60–90 seconds after failover before concluding tunnels are stuck |

---

### Traffic Shaping Rule Not Matching Expected Traffic

**Symptom:** Streaming services are not being capped to 5 Mbps.

**Possible causes:**

- Application not covered by "Video & music" category — add specific app or domain to the definition
- Rule order — verify Rule #1 (critical) is above Rule #2 (non-critical); rules are evaluated top-down
- HTTPS traffic may require SSL inspection to be correctly categorized at L7 — MX performs L7 classification heuristically without full SSL inspection

---

## Lessons Learned

> Hard-won knowledge from production implementation.

---

### 🟡 Lesson 1 — Port Conversion Method Depends on MX Model

On the **MX80**, LAN-to-WAN port conversion is available directly in the Meraki Dashboard under `Security & SD-WAN → Configure → Addressing & VLANs → MX Ports`. No local status page required.

On **other MX models**, the Dashboard may not expose this control. In that case, navigate to `http://[MX-LAN-IP]` → Uplink tab to perform the conversion.

Do not assume the local status page is the only method — check Dashboard first on MX80-class hardware.

---

### 🔴 Lesson 2 — Know Your Port Assignments Before Converting

Verify which port connects to the distribution switch **before** converting any LAN port to WAN. Converting the wrong port will immediately drop the LAN trunk and take down the entire network segment. There is no confirmation prompt.

---

### 🟡 Lesson 3 — WAN 2 "Ready" Status Is Correct

WAN 2 will show **Ready** rather than **Active** while WAN 1 is healthy. This is expected behavior and does not indicate a problem. WAN 2 becomes **Active** only when WAN 1 fails.

---

### 🟡 Lesson 4 — Default Traffic Shaping Rules Are DSCP Tags Only

The "Enable default traffic shaping rules" option applies DSCP marking to SIP, WebEx, video, and software update traffic. It does **not** apply bandwidth limits or change priority. For this use case, disable default rules and create explicit rules instead.

---

### 🟡 Lesson 5 — Bandwidth Limit Slider Is Independent of the Dropdown

The bandwidth limit dropdown in traffic shaping rules controls per-client behavior, not the cap value. The actual Mbps cap is set by the **slider below the dropdown**. The slider value and dropdown selection are independent. Always confirm the slider reads your intended value before saving.

---

### 🟢 Lesson 6 — Auto VPN Failover Requires No Spoke-Side Configuration

When the hub MX fails over to WAN 2, spoke sites automatically renegotiate their VPN tunnels via the Meraki cloud — no manual reconfiguration at remote sites is needed. The only prerequisite is that all spoke devices maintain active cloud connectivity (valid license, internet access).

---

### 🟢 Lesson 7 — WAN 2 Provides Firmware Upgrade Coverage

A verified WAN 2 failover path is the correct way to cover internet continuity during a planned MX firmware upgrade reboot. Schedule firmware updates only after WAN 2 is confirmed working.

---

### 🔴 Lesson 8 — Secondary ISP Gateway Must Be in IP Passthrough Mode, Not Bridge

Bridge mode on a secondary ISP gateway typically still performs NAT — the MX WAN 2 interface receives a private IP (e.g., 192.168.x.x) from the gateway's DHCP rather than the real public IP. This results in double NAT: the MX NATs outbound traffic once, and the ISP gateway NATs it again. Double NAT frequently breaks IPsec-based tunnels and other protocols sensitive to source IP changes.

**Fix:** Configure the secondary ISP gateway for **IP passthrough** (terminology varies by vendor — may also be called DMZ host, bridge mode, or transparent mode). This causes the gateway to pass the real public IP directly to the MX via DHCP. WAN 2 on the Appliance Status page should show a real public IP rather than a private address after the change.

**Confirm:** Dashboard → Security & SD-WAN → Monitor → Appliance Status → Uplink tab → WAN 2 IP should be a publicly routable address, not 192.168.x.x or 10.x.x.x.

---

### 🔴 Lesson 9 — IP Passthrough Requires MX Connected to Gateway's LAN1 Port

Discovered in production: IP passthrough on the secondary ISP gateway only functions correctly when the MX is connected to the gateway's **LAN1** port specifically. Other LAN ports on the gateway may not pass through the public IP. If IP passthrough appears configured but WAN 2 still shows a private IP, verify the physical cable is in LAN1.

---

### 🔴 Lesson 10 — Third-Party IPsec Appliances on the LAN Are Affected by WAN Failover

Any device on the LAN that maintains its own independent IPsec tunnel to a remote endpoint — such as an alerting, dispatch, or public-safety appliance — uses the MX as its default gateway. When the MX fails over to WAN 2, that device's IPsec traffic exits through a different public IP. The remote endpoint's firewall may reject the connection if it only accepts traffic from known source IPs.

This is entirely separate from Meraki Auto VPN, which is cloud-brokered and IP-agnostic. Third-party IPsec appliances are not aware of Meraki failover and must re-establish their tunnels independently over the new path.

**Resolution options:**

- Configure the secondary ISP gateway for IP passthrough (eliminates double NAT — see Lessons 8 and 9)
- Request that the remote endpoint's firewall accept the secondary ISP's public IP as an additional allowed peer
- Expect a reconnect delay of 1–2 minutes after failover while the IPsec tunnel re-establishes — this is normal behavior once the path is clean

**Before relying on WAN 2 failover operationally:** Identify any third-party IPsec appliances on the LAN, confirm their reconnect behavior over the secondary path, and coordinate with the remote endpoint operator if source IP allowlisting is required.

---

### 🟢 Lesson 11 — Meraki Auto VPN Is Not Affected by Source IP Changes

Unlike third-party IPsec appliances, Meraki Auto VPN tunnels are cloud-brokered. When the hub or spoke MX fails over and its WAN IP changes, the Meraki cloud notifies all VPN peers of the new address and tunnels renegotiate automatically. No source IP allowlisting is required at the remote end and no manual reconfiguration is needed. The only prerequisite is that all devices maintain active Meraki cloud connectivity (valid license, internet reachable).

Do not conflate this with third-party IPsec tunnels running on LAN-connected appliances — they behave entirely differently during a WAN failover event.

---

## Checklist

### Phase 1 — WAN 2 Setup

- [ ] ISP2 gateway connected to MX **LAN1** port (required for IP passthrough)
- [ ] ISP2 gateway configured for **IP passthrough** — not bridge mode
- [ ] ISP2 gateway powered on and DHCP enabled
- [ ] LAN trunk port confirmed (e.g., Port 3) — do not convert this port
- [ ] **MX80:** Dashboard → Addressing & VLANs → MX Ports → set Port 2 to WAN, DHCP → Save
- [ ] **Other models:** From LAN device, navigate to `http://[MX-LAN-IP]` → Uplink tab → Convert Port 2 from LAN to WAN → DHCP → Save
- [ ] Dashboard → Appliance Status → Uplink tab → WAN 2 shows **Ready** with a publicly routable IP (not 192.168.x.x or 10.x.x.x)
- [ ] Third-party IPsec appliances on LAN identified and failover behavior confirmed

### Phase 2 — Failover Configuration

- [ ] Dashboard → SD-WAN & Traffic Shaping → Primary uplink = **WAN 1**
- [ ] Load balancing = **Disabled**
- [ ] WAN 1 speed configured (download / upload)
- [ ] WAN 2 speed configured (download / upload)
- [ ] Saved
- [ ] Failover test: unplug WAN 1 → internet remains up via WAN 2 ✅
- [ ] VPN tunnels to all remote sites renegotiated ✅
- [ ] WAN 1 reconnected → traffic fell back automatically ✅

### Phase 3 — Traffic Shaping

- [ ] Default traffic shaping rules = **Disabled**
- [ ] Rule #1 added: critical traffic categories, High priority, unlimited bandwidth
- [ ] Rule #2 added: non-critical traffic categories, Low priority, 5 Mbps cap
- [ ] Rule #1 is above Rule #2 in the list
- [ ] Saved
- [ ] Verified streaming is degraded / capped
- [ ] Verified VoIP and business applications perform normally

### Ongoing

- [ ] Firmware updated to MX 14.56 minimum (required for AMP/ThreatGrid)
- [ ] Firmware update scheduled during maintenance window with WAN 2 active
- [ ] ISP2 license / account details documented for support contacts
- [ ] WAN 2 public IP documented (`198.51.100.50` in this example)
- [ ] Phase 2 hardware replacement includes MX 15+ for per-uplink shaping capability

---

> **Document Version:** 1.0
> **Platform:** Cisco Meraki MX — Dual WAN, Auto Failover, Traffic Shaping
> **Firmware:** MX 14.x (MX 80 and equivalent generation)
> *Verified in production. Platform limitations confirmed. Workarounds documented.*
