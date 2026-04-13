# Meraki MX — Site-to-Site VPN

> **Configuring and managing IPsec site-to-site VPN on the Meraki MX platform — with emphasis on cross-organization and third-party (non-Meraki) peers.**
>
> | | |
> |---|---|
> | **Document Type** | How-To + Reference Guide |
> | **Platform** | Cisco Meraki MX Security Appliance |
> | **Date** | April 2026 |
> | **Status** | ✅ Complete |
> | **Audience** | Administrators managing the Meraki side of an IPsec tunnel |
> | **Prerequisite** | [Site-to-Site VPN Foundations](./Site-to-Site_VPN_Foundations.md) — read this first for IPsec concepts referenced throughout |

---

## Table of Contents

- [Overview — Two VPN Modes](#overview--two-vpn-modes)
- [Auto VPN — Same-Organization Tunnels](#auto-vpn--same-organization-tunnels)
  - [What Auto VPN Is](#what-auto-vpn-is)
  - [How It Works](#how-it-works)
  - [Hub-and-Spoke vs Full Mesh](#hub-and-spoke-vs-full-mesh)
  - [Configuring Auto VPN](#configuring-auto-vpn)
  - [When Auto VPN Doesn't Apply](#when-auto-vpn-doesnt-apply)
- [Non-Meraki VPN — Third-Party and Cross-Org Tunnels](#non-meraki-vpn--third-party-and-cross-org-tunnels)
  - [When to Use Non-Meraki VPN](#when-to-use-non-meraki-vpn)
  - [Prerequisites](#prerequisites)
  - [What to Collect Before You Start](#what-to-collect-before-you-start)
  - [Dashboard Configuration Walkthrough](#dashboard-configuration-walkthrough)
  - [What to Send the Remote Admin](#what-to-send-the-remote-admin)
- [IPsec Parameters — Meraki's Supported Proposals](#ipsec-parameters--merakis-supported-proposals)
  - [Phase 1 (IKE)](#phase-1-ike)
  - [Phase 2 (IPsec)](#phase-2-ipsec)
  - [Meraki Defaults vs Recommended](#meraki-defaults-vs-recommended)
  - [Interop Notes — What the Remote End Needs to Know](#interop-notes--what-the-remote-end-needs-to-know)
- [Peer Identity on the MX](#peer-identity-on-the-mx)
- [NAT-T and the MX](#nat-t-and-the-mx)
- [Monitoring and Status](#monitoring-and-status)
  - [VPN Status Page](#vpn-status-page)
  - [Color Indicators](#color-indicators)
  - [Event Log](#event-log)
  - [Packet Capture](#packet-capture)
- [Troubleshooting from the Meraki Side](#troubleshooting-from-the-meraki-side)
  - [Tunnel Won't Establish — Checklist](#tunnel-wont-establish--checklist)
  - [Tunnel Established but No Traffic](#tunnel-established-but-no-traffic)
  - [Intermittent Tunnel Drops](#intermittent-tunnel-drops)
  - [Reading the Event Log for IKE Errors](#reading-the-event-log-for-ike-errors)
  - [Communicating Findings to the Remote Admin](#communicating-findings-to-the-remote-admin)
- [Transition Planning — Replacing Your MX](#transition-planning--replacing-your-mx)
  - [What Changes](#what-changes)
  - [What Stays the Same](#what-stays-the-same)
  - [What to Tell the Remote Admin Before Cutover](#what-to-tell-the-remote-admin-before-cutover)
  - [Cutover Procedure](#cutover-procedure)
  - [Rollback Plan](#rollback-plan)
- [DNS Across the Tunnel](#dns-across-the-tunnel)
- [Dual-WAN and VPN Failover](#dual-wan-and-vpn-failover)
- [Known Quirks and Gotchas](#known-quirks-and-gotchas)
- [Quick Reference — Dashboard Navigation](#quick-reference--dashboard-navigation)
- [Appendix A — Parameter Exchange Template](#appendix-a--parameter-exchange-template)
- [Appendix B — Pre-Cutover Communication Template](#appendix-b--pre-cutover-communication-template)

---

## Overview — Two VPN Modes

The Meraki MX supports two fundamentally different site-to-site VPN modes. Understanding which one applies to your situation is the first decision.

| Feature | Auto VPN | Non-Meraki VPN |
|:--------|:---------|:--------------|
| **Peers** | Other Meraki MX devices **in the same organization** | Any IPsec-capable device — different org Meraki, pfSense, Cisco IOS, Fortinet, etc. |
| **Configuration** | Cloud-brokered; minimal manual config | Manual IPsec parameter configuration |
| **Key exchange** | Automatic (Meraki cloud handles it) | Pre-shared key (PSK) exchanged out-of-band |
| **Peer discovery** | Automatic via Meraki cloud | Static peer IP required |
| **Crypto parameters** | Automatic (Meraki selects) | You configure Phase 1 and Phase 2 proposals |
| **Dashboard section** | `Security & SD-WAN → Site-to-site VPN` → Hub/Spoke | `Security & SD-WAN → Site-to-site VPN` → Organization-wide → Non-Meraki VPN peers |
| **Use when** | All tunnel endpoints are MX devices in your org | Any cross-org or non-Meraki peer |

> ℹ️ **Both modes can coexist.** An MX configured as an Auto VPN hub can simultaneously maintain Non-Meraki VPN tunnels to third-party peers.

---

## Auto VPN — Same-Organization Tunnels

### What Auto VPN Is

Auto VPN is Meraki's proprietary cloud-brokered VPN. It eliminates manual IPsec configuration by having the Meraki cloud coordinate tunnel setup between MX devices in the same Dashboard organization. You choose hub-and-spoke or full mesh, select which networks participate, and Meraki handles the rest — key exchange, peer discovery, failover, and tunnel health monitoring.

### How It Works

```
┌─────────────┐         ┌──────────────────┐         ┌──────────────┐
│  Site A MX   │◄───────►│   Meraki Cloud    │◄───────►│  Site B MX    │
│  (Hub)       │         │  Brokers tunnel   │         │  (Spoke)      │
└─────────────┘         │  parameters &     │         └──────────────┘
                        │  peer discovery   │
                        └──────────────────┘
```

1. Each MX registers with the Meraki cloud and reports its public IP
2. The cloud distributes tunnel parameters to all participating MX devices
3. MX devices establish IPsec tunnels directly to each other (data does **not** flow through the cloud)
4. If a WAN IP changes (e.g., during failover), the cloud notifies peers automatically

> 💡 **The Meraki cloud is the control plane, not the data plane.** User traffic flows directly between MX devices over IPsec — the cloud only coordinates the setup.

### Hub-and-Spoke vs Full Mesh

| Topology | Traffic Flow | When to Use |
|:---------|:------------|:-----------|
| **Hub-and-spoke** | All spoke traffic routes through the hub; spokes don't talk directly | Central services at HQ; internet exit at hub; most common topology |
| **Full mesh** | Every site tunnels directly to every other site | Sites need direct inter-site communication without hairpinning through hub |

> ⚠️ **Hub-and-spoke has a single point of failure.** If the hub goes down, all spoke-to-spoke communication stops because there's no direct path. Consider dual-hub for critical environments.

### Configuring Auto VPN

**Dashboard path:** `Security & SD-WAN` → `Site-to-site VPN`

1. **Set the VPN role:**

    | Site Type | Setting |
    |:----------|:--------|
    | Hub (e.g., HQ) | **Hub (Mesh)** |
    | Spoke (e.g., remote station) | **Spoke** |

2. **Select VPN subnets:** Under "VPN settings," toggle each local subnet to participate in VPN:

    | Subnet | Use VPN |
    |:-------|:--------|
    | Primary employee LAN | ✅ Yes |
    | Wireless VLAN | ✅ Yes (if remote sites need to reach it) |
    | Guest VLAN | ❌ No (guests should not traverse the tunnel) |

3. **Spoke hub configuration:** On each spoke, under "Hub settings," select which hub(s) to connect to and set the hub priority.

4. **Save:** Click **Update** at the bottom of the page.

    > ⚠️ **Meraki dashboard save behavior:** You **must** click "Update" to save. Unsaved changes silently revert. This has caused outages in the past — always confirm the save completed.

5. **Verify:** Navigate to `Security & SD-WAN` → `VPN status`. Tunnels should show green within 60 seconds.

### When Auto VPN Doesn't Apply

Auto VPN **cannot** be used when:

- The remote peer is a non-Meraki device (pfSense, Cisco IOS, Fortinet, etc.)
- The remote MX is in a **different Meraki Dashboard organization**
- You need granular control over IPsec parameters (encryption, DH group, lifetime)
- You need to interoperate with a specific IKE version or proposal set

For all of these scenarios, use **Non-Meraki VPN** — the next section.

---

## Non-Meraki VPN — Third-Party and Cross-Org Tunnels

### When to Use Non-Meraki VPN

This is the correct mode when:

- The remote end is a Meraki MX in a **different organization** (cross-org)
- The remote end is a **non-Meraki device** (pfSense, Cisco IOS, Fortinet, SonicWall, etc.)
- You need to configure specific IPsec parameters to match the remote end
- You are preparing to **replace your MX** with non-Meraki hardware and want the tunnel config to translate cleanly

> ℹ️ For cross-organization Meraki-to-Meraki tunnels, both sides configure the other as a "Non-Meraki VPN peer" — even though both are Meraki devices. The Auto VPN cloud broker only works within a single org.

### Prerequisites

- [ ] Meraki Dashboard access — **Organization Administrator** role
- [ ] Your MX is configured as a VPN **Hub** or **Spoke** (the VPN role must be set even for Non-Meraki tunnels)
- [ ] Remote peer's public IP, local subnets, and IPsec parameters are known
- [ ] PSK has been exchanged securely with the remote admin
- [ ] Firewall rules on both sides permit UDP 500, UDP 4500, and ESP (Protocol 50)
- [ ] [Parameter Exchange Checklist](./Site-to-Site_VPN_Foundations.md#coordination-checklist--parameter-exchange) is completed for both sides

### What to Collect Before You Start

Before touching the dashboard, gather all of the following from the remote admin:

| Item | What You Need | Why |
|:-----|:-------------|:----|
| **Remote public IP** | The WAN IP of the remote VPN gateway | The MX needs a target to initiate or accept IKE |
| **Remote LAN subnets** | All subnets behind the remote gateway that should be reachable through the tunnel | Defines your traffic selectors (what goes in the tunnel) |
| **PSK** | The pre-shared key (agreed upon by both admins) | Authentication during Phase 1 |
| **IKE version** | IKEv1 or IKEv2 | Must match both sides |
| **Phase 1 proposals** | Encryption, hash, DH group, lifetime | Must match both sides exactly |
| **Phase 2 proposals** | Encryption, hash, PFS group, lifetime | Must match both sides exactly |
| **Peer identity type** | IP address, FQDN, or email | How the remote end identifies itself |
| **NAT situation** | Is the remote end behind NAT? | Affects NAT-T behavior |
| **Contact info** | Name, phone, email, maintenance window | You will need this during troubleshooting |

> 💡 **Use the Parameter Exchange Template** in [Appendix A](#appendix-a--parameter-exchange-template) — send it to the remote admin and have them fill in their column before you start.

### Dashboard Configuration Walkthrough

#### Step 1 — Enable Site-to-Site VPN

**Dashboard path:** `Security & SD-WAN` → `Site-to-site VPN`

Set the VPN type to **Hub (Mesh)** or **Spoke**, depending on your site's role in the overall topology. This must be done even if you're only using Non-Meraki VPN — the MX won't expose the Non-Meraki VPN configuration section unless a VPN role is set.

#### Step 2 — Define Local Subnets

Under the **VPN settings** section, enable VPN for each local subnet that should be accessible through the tunnel:

| Local Subnet | VLAN | Enable VPN? | Notes |
|:------------|:-----|:-----------|:------|
| Employee LAN | 101 | ✅ | Primary data network — remote sites need this |
| Wireless VLAN | 711 | Depends | Only if remote sites need to reach wireless clients |
| Guest VLAN | 714 | ❌ | Never route guest traffic across the tunnel |

> ⚠️ **Every subnet you enable here becomes part of your "interesting traffic" (traffic selectors).** The remote end's configuration must include the same subnets as your "remote" subnets. Adding a subnet on your side without coordinating with the remote admin will cause a Phase 2 mismatch.

#### Step 3 — Add a Non-Meraki VPN Peer

Scroll down to the **Non-Meraki VPN peers** section (below the Auto VPN settings). Click **Add a peer**.

Fill in the following fields:

| Field | Value | Notes |
|:------|:------|:------|
| **Name** | Descriptive label (e.g., `County-HQ-Tunnel`) | Dashboard display only — not transmitted to the peer |
| **Public IP** | The remote peer's WAN IP address | Must match exactly; if they have dual WAN, use the IP they're initiating from |
| **Remote subnets** | The remote site's LAN subnets | One entry per subnet; these become your Phase 2 traffic selectors |
| **IPsec parameters** | See Step 4 | |
| **Preshared secret** | The PSK agreed upon with the remote admin | Case-sensitive; watch for trailing spaces |
| **Availability** | Which of your networks can use this tunnel | Select the specific network or "All networks" |

#### Step 4 — Configure IPsec Parameters

Within the peer configuration, expand the **IPsec policies** section. The MX provides dropdown menus for each parameter.

**Phase 1 (IKE) settings:**

| Parameter | Field Name in Dashboard | Set To |
|:----------|:----------------------|:-------|
| IKE version | IKE version | Match the remote end (IKEv1 or IKEv2) |
| Encryption | Encryption | Match the remote end (e.g., AES-256) |
| Hash | Authentication | Match the remote end (e.g., SHA-256) |
| DH group | Diffie-Hellman group | Match the remote end (e.g., Group 14) |
| Lifetime | Lifetime | Match the remote end (e.g., 28800 seconds) |

**Phase 2 (IPsec) settings:**

| Parameter | Field Name in Dashboard | Set To |
|:----------|:----------------------|:-------|
| Encryption | Encryption | Match the remote end (e.g., AES-256) |
| Hash | Authentication | Match the remote end (e.g., SHA-256) |
| PFS | PFS group | Match the remote end (e.g., Group 14, or Disabled if they don't use PFS) |
| Lifetime | Lifetime | Match the remote end (e.g., 3600 seconds) |

> 🚫 **Do not use "Default" for any parameter unless you have confirmed Meraki's default matches the remote end exactly.** See [Meraki Defaults vs Recommended](#meraki-defaults-vs-recommended) below to understand what "Default" actually selects.

#### Step 5 — Define Remote Subnets

In the peer configuration's **Remote subnets** field, enter each remote subnet in CIDR notation:

```
10.2.0.0/24
10.3.0.0/24
```

Each entry becomes a Phase 2 traffic selector. The remote end must have **your** subnets listed as **their** remote subnets — the entries are mirror images.

> ⚠️ **IKEv1 limitation:** Each remote subnet creates a separate Phase 2 SA. If you have 5 remote subnets and 3 local subnets, that's 15 Phase 2 SAs. With IKEv2, multiple subnets can share a single SA.

#### Step 6 — Save and Verify

1. Click **Update** to save the configuration

    > ⚠️ **Repeat: you must click "Update."** Navigating away without saving silently discards all changes. There is no draft or auto-save.

2. Navigate to `Security & SD-WAN` → `VPN status`
3. The new peer should appear in the **Non-Meraki VPN peers** section
4. Status should progress from gray (configuring) to green (established) within 1–3 minutes
5. If status remains gray or turns red, proceed to [Troubleshooting](#troubleshooting-from-the-meraki-side)

### What to Send the Remote Admin

After saving your configuration, provide the remote admin with the following so they can configure their side:

| Item | Your Value |
|:-----|:----------|
| **Your public WAN IP** | *(from `Security & SD-WAN` → `Appliance status`)* |
| **Your local subnets** | *(all subnets you enabled for VPN in Step 2)* |
| **PSK** | *(the agreed-upon key — confirm they have it)* |
| **Phase 1 parameters** | *(exactly as configured in Step 4)* |
| **Phase 2 parameters** | *(exactly as configured in Step 4)* |
| **Your peer identity** | *(see [Peer Identity](#peer-identity-on-the-mx))* |
| **NAT status** | *(is there NAT between your MX and the internet?)* |

> 💡 **Use the completed Parameter Exchange Template** from [Appendix A](#appendix-a--parameter-exchange-template) to communicate this cleanly.

---

## IPsec Parameters — Meraki's Supported Proposals

### Phase 1 (IKE)

| Parameter | Supported Values on MX |
|:----------|:----------------------|
| **IKE Version** | IKEv1, IKEv2 |
| **Encryption** | DES, 3DES, AES-128, AES-192, AES-256 |
| **Authentication (Hash)** | MD5, SHA-1, SHA-256 |
| **DH Group** | 1, 2, 5, 14, 19, 20 |
| **Lifetime** | Configurable in seconds (default: 28800) |

### Phase 2 (IPsec)

| Parameter | Supported Values on MX |
|:----------|:----------------------|
| **Encryption** | DES, 3DES, AES-128, AES-192, AES-256 |
| **Authentication (Hash)** | MD5, SHA-1, SHA-256 |
| **PFS Group** | Disabled, 1, 2, 5, 14, 19, 20 |
| **Lifetime** | Configurable in seconds (default: 3600) |

### Meraki Defaults vs Recommended

When you select "Default" in the dashboard dropdowns, the MX proposes **multiple** algorithms in a single offer and lets the peer select from the list. This is convenient but opaque — you don't control what gets selected, and it can lead to weaker-than-intended crypto if the remote end picks the weakest common option.

| Parameter | "Default" Proposes (typical) | Recommended Override |
|:----------|:---------------------------|:--------------------|
| Phase 1 Encryption | AES-256, AES-192, AES-128, 3DES | **AES-256** (explicit) |
| Phase 1 Hash | SHA-256, SHA-1, MD5 | **SHA-256** (explicit) |
| Phase 1 DH Group | 14, 5, 2, 1 | **14** (explicit) |
| Phase 2 Encryption | AES-256, AES-192, AES-128, 3DES | **AES-256** (explicit) |
| Phase 2 Hash | SHA-256, SHA-1, MD5 | **SHA-256** (explicit) |
| Phase 2 PFS | Disabled by default | **14** (explicit) |

> ✅ **Best practice:** Never use "Default" for Non-Meraki VPN peers. Explicitly set every parameter to match the remote end. This eliminates ambiguity and makes troubleshooting dramatically simpler — if a parameter mismatch causes a failure, you know exactly what both sides are proposing.

### Interop Notes — What the Remote End Needs to Know

When the remote end is **not** a Meraki device, they need to be aware of the following:

1. **Meraki uses policy-based VPN.** The MX does not create a virtual tunnel interface (VTI). Traffic selectors (proxy IDs / encryption domains) drive the tunnel. The remote end — especially pfSense or Cisco IOS — may default to route-based VPN and need to be configured with matching traffic selectors.

2. **Phase 2 traffic selectors must be exact mirrors.** If the MX defines local subnet `10.1.0.0/24` and remote subnet `10.2.0.0/24`, the remote end must define local `10.2.0.0/24` and remote `10.1.0.0/24`. A mismatch — even a different CIDR mask — causes `INVALID_ID_INFORMATION` in Phase 2.

3. **IKEv1 creates one Phase 2 SA per subnet pair.** If you have 3 local subnets and 2 remote subnets, the MX initiates 6 Phase 2 negotiations. The remote end must accept all of them. IKEv2 can consolidate these.

4. **Meraki identifies itself by WAN IP.** The MX uses its public WAN IP as its IKE identity (Local ID). The remote end's "Remote ID" or "Peer ID" should be set to the MX's WAN IP address. See [Peer Identity](#peer-identity-on-the-mx).

5. **NAT-T is always enabled on the MX.** The MX always negotiates NAT-T during Phase 1. The remote end should also have NAT-T enabled (or set to auto).

6. **DPD is always enabled on the MX.** The remote end should also have DPD enabled to ensure both sides detect dead peers.

---

## Peer Identity on the MX

The MX identifies itself using its **public WAN IP address** as the IKE Local ID. This is not configurable in the dashboard for Non-Meraki VPN.

| Direction | Identity |
|:----------|:--------|
| **MX Local ID** | MX's public WAN IP (automatic) |
| **MX Remote ID** | The public IP entered in the peer configuration |

**What this means for the remote admin:**

The remote admin must configure their device's **Remote ID** (or **Peer ID**) to the MX's WAN IP address, and their **Local ID** to their own WAN IP. If the remote end uses FQDN-based identity, there will be a mismatch unless special handling is applied.

> ⚠️ **Dual-WAN consideration:** If your MX has dual WAN and fails over, the WAN IP changes — and so does the MX's IKE identity. The remote admin must accept both IPs or the tunnel will fail to re-establish after failover.

---

## NAT-T and the MX

The MX **always** enables NAT-T. You cannot disable it from the dashboard. During IKE Phase 1, the MX detects whether NAT exists between itself and the remote peer:

| NAT Detected? | Behavior |
|:-------------|:---------|
| **No** | IKE and ESP traffic use standard ports (UDP 500, ESP Protocol 50) |
| **Yes** | All traffic encapsulated in UDP 4500 (NAT-T) |

**If there's an ISP router upstream doing NAT before your MX:**

The MX's WAN IP will be private (192.168.x.x, 10.x.x.x, etc.), and the actual public IP is on the ISP router. NAT-T will activate automatically. The remote admin needs to target the ISP router's **public** IP, not the MX's private WAN IP.

> 💡 **To determine your actual public IP:** `Security & SD-WAN` → `Appliance status` → look for the "Public IP" field (this may differ from the WAN interface IP if NAT is present).

---

## Monitoring and Status

### VPN Status Page

**Dashboard path:** `Security & SD-WAN` → `VPN status`

This page shows:

- All Auto VPN tunnels and their current state
- All Non-Meraki VPN peers and their current state
- Latency and packet loss per tunnel
- Uptime for each tunnel

### Color Indicators

| Color | Meaning | Action |
|:------|:--------|:-------|
| 🟢 **Green** | Tunnel is established and passing traffic | None — healthy |
| ⚪ **Gray** | Tunnel is configured but not established; may be negotiating | Wait 2–3 minutes; if persists, check Phase 1 connectivity |
| 🔴 **Red** | Tunnel failed to establish or was torn down | Investigate — see [Troubleshooting](#troubleshooting-from-the-meraki-side) |
| 🟡 **Yellow** | Tunnel is up but experiencing high latency or packet loss | Monitor; may indicate WAN issue, not VPN misconfiguration |

### Event Log

**Dashboard path:** `Network-wide` → `Event log`

Filter by event type: **VPN**

Key events to watch for:

| Event | Meaning |
|:------|:--------|
| `VPN tunnel established` | Phase 1 and Phase 2 both succeeded — tunnel is up |
| `VPN tunnel terminated` | Tunnel was torn down (DPD timeout, rekey failure, or admin action) |
| `VPN peer unreachable` | DPD probes are failing — remote end may be down or blocked |
| `IKE negotiation failed` | Phase 1 or Phase 2 mismatch — check parameters |

> 💡 **The event log is your primary diagnostic tool when you control only one side.** The specific event message tells you which phase failed and often includes the reason code.

### Packet Capture

**Dashboard path:** `Network-wide` → `Packet capture`

You can capture on the MX's WAN interface to see IKE negotiation packets. Filter by:

- Port: UDP 500 (IKE) or UDP 4500 (NAT-T)
- Remote IP: the peer's public IP

This is useful for confirming whether your MX is sending proposals and whether the remote end is responding.

---

## Troubleshooting from the Meraki Side

> **Cross-reference:** For protocol-level troubleshooting (which phase failed, what errors mean), refer to the [Troubleshooting Methodology](./Site-to-Site_VPN_Foundations.md#troubleshooting-methodology) section of the Foundations document.

### Tunnel Won't Establish — Checklist

Work through these in order:

- [ ] **Confirm the VPN role is set.** The MX must be set to Hub or Spoke under `Site-to-site VPN`. If set to "None," the Non-Meraki VPN section is hidden and all tunnels are disabled.

- [ ] **Confirm the configuration was saved.** Check that your peer appears in the dashboard and that the parameters displayed match what you entered. If they reverted, you didn't click "Update."

- [ ] **Confirm the remote peer's public IP is reachable.** From the MX, there's no direct ping tool — but you can check event logs for `VPN peer unreachable`. Alternatively, from a computer on the LAN, verify the remote IP responds to ping (if the remote admin allows ICMP).

- [ ] **Confirm firewall rules.** UDP 500 and 4500 must be permitted inbound and outbound on any device between the MX and the public internet (ISP routers, upstream firewalls, cloud security groups).

- [ ] **Confirm the PSK.** Re-enter it on both sides. Watch for:
    - Trailing spaces (copy-paste often adds them)
    - Character encoding differences (special characters may render differently)
    - The PSK is case-sensitive

- [ ] **Confirm Phase 1 parameters match.** Compare your encryption, hash, DH group, lifetime, and IKE version against the remote admin's configuration.

- [ ] **Confirm Phase 2 parameters match.** Compare encryption, hash, PFS group, and lifetime. PFS group mismatch is one of the most common Phase 2 failures — if one side has PFS enabled and the other doesn't, it fails silently.

- [ ] **Confirm traffic selectors (subnets) are mirror images.** Your local subnets must appear as their remote subnets, and vice versa. Exact CIDR match required.

### Tunnel Established but No Traffic

The VPN status page shows green, but devices can't reach the remote site.

- [ ] **Routing:** Are devices at your site using the MX as their default gateway? If devices point to a different gateway, traffic won't enter the tunnel.
- [ ] **Traffic selectors:** Is the specific traffic (source IP / destination IP) covered by the tunnel's subnet definitions? Traffic outside the selectors bypasses the tunnel.
- [ ] **Remote-side firewall:** The remote site may have the tunnel up but a local firewall blocking traffic that exits the tunnel. Ask the remote admin to check.
- [ ] **DNS:** Can you ping the remote site by IP? If ping works but name resolution doesn't, the issue is DNS, not VPN. See [DNS Across the Tunnel](#dns-across-the-tunnel).

### Intermittent Tunnel Drops

- [ ] **SA lifetime mismatch:** If lifetimes don't match, one side tries to rekey before the other expects it. This can cause periodic drops. Match lifetimes exactly.
- [ ] **DPD timeout:** If the WAN link is marginal (high latency or packet loss), DPD probes may fail intermittently. Check the `VPN peer unreachable` events in the log and correlate with WAN health.
- [ ] **WAN failover:** If your MX has dual WAN and fails over, the tunnel tears down and must re-establish with the new WAN IP. The remote end must accept the new IP.
- [ ] **ISP issues:** MTU problems on the WAN path can cause intermittent failures. IPsec overhead pushes packets past 1500 bytes, and if the ISP drops oversized packets or fragments, the tunnel may appear unstable.

### Reading the Event Log for IKE Errors

| Event Log Message | What It Means | Likely Cause |
|:-----------------|:-------------|:------------|
| `IKE negotiation failed: no proposal chosen` | Phase 1 crypto mismatch | Encryption, hash, or DH group doesn't match |
| `IKE negotiation failed: invalid key` | PSK mismatch | PSK is different on each side |
| `IKE negotiation failed: invalid ID` | Peer identity mismatch | Local/Remote ID type or value doesn't match |
| `IKE negotiation failed: timeout` | No response from remote peer | Peer is down, IP is wrong, or firewall is blocking |
| `IPsec SA negotiation failed: no proposal chosen` | Phase 2 crypto mismatch | Phase 2 encryption, hash, or PFS mismatch |
| `IPsec SA negotiation failed: invalid ID` | Traffic selector mismatch | Subnets don't mirror correctly |
| `VPN peer unreachable` | DPD failure | Remote end is down, network path issue, or firewall blocking |

### Communicating Findings to the Remote Admin

When you've identified an issue from your event log, translate it into something actionable for the remote admin:

<details>
<summary><strong>Template: Reporting a Phase 1 failure</strong></summary>

> Subject: VPN Tunnel — Phase 1 Negotiation Failure
>
> Our Meraki MX at [your WAN IP] is attempting to establish an IKE Phase 1 SA with your device at [their WAN IP].
>
> Our event log shows: `[exact error message from the event log]`
>
> Our Phase 1 configuration:
> - IKE version: [version]
> - Encryption: [algorithm]
> - Hash: [algorithm]
> - DH group: [number]
> - Lifetime: [seconds]
> - Local ID: [your WAN IP]
>
> Could you verify that your device is:
> 1. Configured with a VPN peer pointing to our IP ([your WAN IP])?
> 2. Using matching Phase 1 parameters?
> 3. Permitting inbound UDP 500 and 4500?
> 4. Showing any corresponding log entries from our IP?

</details>

<details>
<summary><strong>Template: Reporting a Phase 2 failure</strong></summary>

> Subject: VPN Tunnel — Phase 2 Negotiation Failure (Phase 1 is UP)
>
> IKE Phase 1 between our MX ([your WAN IP]) and your device ([their WAN IP]) is established successfully.
>
> Phase 2 (IPsec SA) is failing. Our event log shows: `[exact error message]`
>
> Our Phase 2 configuration:
> - Encryption: [algorithm]
> - Hash: [algorithm]
> - PFS: [group or "Disabled"]
> - Lifetime: [seconds]
> - Local subnets (our side): [list]
> - Remote subnets (your side as we see it): [list]
>
> Could you verify:
> 1. Your Phase 2 parameters match the above?
> 2. Your local subnets match what we have as remote?
> 3. PFS is set to [group / Disabled] on your side?

</details>

---

## Transition Planning — Replacing Your MX

When you replace your Meraki MX with a different platform (e.g., Netgate pfSense), the existing Non-Meraki VPN tunnel must be rebuilt on the new hardware. The goal is minimal downtime and no confusion with the remote admin.

### What Changes

| Item | MX Value | New Device Value | Impact |
|:-----|:---------|:----------------|:-------|
| **Public WAN IP** | May change | Depends on ISP config | Remote admin must update their peer IP |
| **Peer identity** | MX's WAN IP (automatic) | Configurable — set to new WAN IP | Remote admin must update their Remote ID |
| **Dashboard management** | Meraki Dashboard | New platform's management interface | You lose Meraki VPN status page, event log, packet capture |

### What Stays the Same

| Item | Notes |
|:-----|:------|
| **PSK** | Keep the same PSK — no reason to change during a hardware swap unless you want to rotate it |
| **Phase 1 and Phase 2 parameters** | Configure the new device with identical crypto proposals |
| **Local subnets** | Your LAN doesn't change — same subnets, same traffic selectors |
| **Remote subnets** | The remote end's network doesn't change |

### What to Tell the Remote Admin Before Cutover

Send this **at least 48 hours before the maintenance window.** See [Appendix B](#appendix-b--pre-cutover-communication-template) for a ready-to-send template.

Key points to communicate:

1. **Date and time** of the cutover (with timezone)
2. **Expected downtime** (typically 15–60 minutes)
3. **New public WAN IP** (if it changes)
4. **Everything else stays the same** — PSK, crypto, subnets
5. **Request they update their peer IP** (if it changes) at the start of the maintenance window
6. **Your contact info** for the window — phone number, not just email

### Cutover Procedure

1. **Before the window:**
    - [ ] New device is staged and configured with the IPsec tunnel (matching all parameters from the MX)
    - [ ] Remote admin has been notified and has your new WAN IP (if changed)
    - [ ] Remote admin is available during the window

2. **During the window:**
    - [ ] Disconnect the MX WAN cable
    - [ ] Connect the new device to the WAN circuit
    - [ ] Verify the new device obtains the correct WAN IP
    - [ ] Notify the remote admin to update their peer IP (if changed)
    - [ ] Monitor the new device's VPN status — tunnel should establish within 1–3 minutes
    - [ ] Test bidirectional traffic (ping across the tunnel from both sides)
    - [ ] Test application-level connectivity (not just ping — verify the services that matter)

3. **After the window:**
    - [ ] Monitor for 24 hours — watch for intermittent drops, rekey failures, or traffic issues
    - [ ] Confirm with the remote admin that their side is healthy

### Rollback Plan

If the new device fails to establish the tunnel within the maintenance window:

1. Disconnect the new device
2. Reconnect the MX
3. Verify the MX re-establishes the tunnel (it should, since the config is still in the Meraki cloud)
4. Notify the remote admin to revert their peer IP (if they changed it)
5. Investigate the failure offline and reschedule

> ⚠️ **Keep the MX powered on and configured until the new device has been stable for at least one week.** Don't decommission it the same day.

---

## DNS Across the Tunnel

> **Cross-reference:** [DNS Across the Tunnel](./Site-to-Site_VPN_Foundations.md#dns-across-the-tunnel) in the Foundations document for concepts.

**On the Meraki MX specifically:**

The MX can act as a DNS forwarder. Clients at your site point to the MX as their DNS server (this is the default when DHCP is served by the MX). The MX forwards queries to upstream DNS servers (typically public DNS like 8.8.8.8 or your ISP's DNS).

To resolve hostnames at the **remote** site through the tunnel:

| Approach | How to Configure on MX | Notes |
|:---------|:----------------------|:------|
| **Remote DNS server in DHCP** | Add the remote site's DNS server IP to the DHCP DNS field alongside your existing DNS servers | Clients query the remote DNS directly through the tunnel; remote DNS IP must be within the traffic selectors |
| **Hosts file on individual devices** | Add static entries for critical remote hosts on the devices that need them | Doesn't scale; use only for a handful of critical systems |
| **Dedicated DNS server** | Deploy a local DNS server with conditional forwarding rules | Most flexible; the MX points to this server, which handles forwarding logic |

> ⚠️ **Meraki MX does not support conditional DNS forwarding natively.** The MX forwards all DNS queries to the configured upstream servers. For conditional forwarding (different DNS servers for different domains), you need a dedicated DNS server on the LAN.

---

## Dual-WAN and VPN Failover

**How Non-Meraki VPN behaves during WAN failover:**

Unlike Auto VPN (where the Meraki cloud re-brokers the tunnel automatically), Non-Meraki VPN tunnels **do not fail over gracefully** when the active WAN changes:

| Event | What Happens |
|:------|:------------|
| WAN 1 goes down, MX fails over to WAN 2 | MX's public IP changes; IKE identity changes; existing SAs are torn down |
| MX initiates new Phase 1 from WAN 2 IP | If remote end only accepts the WAN 1 IP, the tunnel stays down |
| WAN 1 comes back, MX fails back | Public IP reverts; another renegotiation cycle |

**Mitigation options:**

1. **Provide both WAN IPs to the remote admin.** Ask them to configure two VPN peers — one for each of your WAN IPs. Only one will be active at a time, but either can initiate.

2. **Use FQDN-based identity with dynamic DNS.** If the remote platform supports FQDN as the peer identity, a DDNS hostname that updates when the WAN changes can help. Note: the MX itself uses IP-based identity, so this helps the remote end find you — but the identity mismatch may still cause issues.

3. **Accept the downtime.** For most environments, a 1–5 minute tunnel re-establishment after WAN failover is acceptable. The remote admin just needs to accept connections from both of your WAN IPs.

> ⚠️ **If the remote admin's device only allows a single peer IP, your tunnel WILL be down after a WAN failover until someone manually updates the configuration.** Plan for this.

---

## Known Quirks and Gotchas

### 1 — Dashboard Save Behavior

**"Update" is the save action.** There is no auto-save, no draft, and no confirmation dialog if you navigate away. Unsaved changes silently revert. After every change, click "Update" and visually confirm the configuration persisted.

### 2 — VPN Role Must Be Set

The Non-Meraki VPN configuration section is invisible in the dashboard unless the MX's VPN role is set to Hub or Spoke. If you don't see the "Non-Meraki VPN peers" section, check the VPN type dropdown at the top of the page.

### 3 — Subnet Must Be Assigned to a Port

A VLAN must be assigned to at least one MX LAN port before it appears in the VPN subnet list or the DHCP configuration page. This is undocumented Meraki behavior — if you create a VLAN in dashboard but don't assign it to a port, it won't show up where you expect.

### 4 — Phase 2 SA Count with IKEv1

IKEv1 creates a separate Phase 2 SA for every combination of local subnet × remote subnet. If you're advertising 4 local subnets and the remote end has 3, that's 12 Phase 2 SAs. Some devices limit the number of concurrent SAs — confirm with the remote admin.

### 5 — MX Firmware Ceiling

Some MX hardware (notably MX80) cannot run firmware above MX 14.x. The available cipher suites and IKE features depend on firmware version. Verify what your specific firmware supports before promising parameters to the remote admin.

### 6 — Meraki Does Not Expose Full IKE Logs

The Meraki event log shows high-level VPN events but does not expose raw IKE negotiation logs (unlike pfSense or Cisco IOS, where you can see every proposal and response). Your troubleshooting resolution is limited to the event messages listed in the [Event Log section](#reading-the-event-log-for-ike-errors).

### 7 — PSK Visibility

The PSK is hidden (masked) in the dashboard after saving. If you need to verify the PSK, you must re-enter it — there's no "show" option. Keep a secure copy outside the dashboard.

### 8 — No MSS Clamping Control

The MX handles MTU/MSS internally. There is no dashboard control for MSS clamping. If you experience MTU-related issues (large transfers failing, SSH working but SCP/SFTP failing), you may need to clamp MSS on the remote end or on individual hosts.

---

## Quick Reference — Dashboard Navigation

| Task | Dashboard Path |
|:-----|:-------------|
| Set VPN role (Hub/Spoke/None) | `Security & SD-WAN` → `Site-to-site VPN` → Type dropdown |
| Enable subnets for VPN | `Security & SD-WAN` → `Site-to-site VPN` → VPN settings → toggle per subnet |
| Add/edit Non-Meraki VPN peer | `Security & SD-WAN` → `Site-to-site VPN` → scroll to "Non-Meraki VPN peers" |
| View VPN tunnel status | `Security & SD-WAN` → `VPN status` |
| View VPN events | `Network-wide` → `Event log` → filter by VPN |
| Packet capture on WAN | `Network-wide` → `Packet capture` |
| Check public IP | `Security & SD-WAN` → `Appliance status` |
| View connected clients | `Network-wide` → `Clients` |

---

## Appendix A — Parameter Exchange Template

Copy and fill in your column, then send to the remote admin for them to complete:

```
╔════════════════════════════════════════════════════════════════════╗
║              SITE-TO-SITE VPN — PARAMETER EXCHANGE                ║
╠════════════════════════╦═══════════════════╦═══════════════════════╣
║ Parameter              ║ Your Site         ║ Remote Site           ║
╠════════════════════════╬═══════════════════╬═══════════════════════╣
║ Organization           ║                   ║                       ║
║ Contact Name           ║                   ║                       ║
║ Contact Phone          ║                   ║                       ║
║ Contact Email          ║                   ║                       ║
╠════════════════════════╬═══════════════════╬═══════════════════════╣
║ VPN Platform           ║ Meraki MX         ║                       ║
║ Firmware/Version       ║                   ║                       ║
║ Public WAN IP          ║                   ║                       ║
║ Backup WAN IP          ║                   ║                       ║
║ NAT Present?           ║ Yes / No          ║ Yes / No              ║
╠════════════════════════╬═══════════════════╬═══════════════════════╣
║ Local LAN Subnet(s)    ║                   ║                       ║
║                        ║                   ║                       ║
║                        ║                   ║                       ║
╠════════════════════════╬═══════════════════╬═══════════════════════╣
║ IKE Version            ║                   ║                       ║
╠════════════════════════╬═══════════════════╬═══════════════════════╣
║ PHASE 1                ║                   ║                       ║
║   Encryption           ║                   ║                       ║
║   Hash                 ║                   ║                       ║
║   DH Group             ║                   ║                       ║
║   Lifetime (sec)       ║                   ║                       ║
║   Auth Method          ║ PSK               ║ PSK                   ║
║   Local ID (type)      ║ IP Address        ║                       ║
║   Local ID (value)     ║                   ║                       ║
╠════════════════════════╬═══════════════════╬═══════════════════════╣
║ PHASE 2                ║                   ║                       ║
║   Encryption           ║                   ║                       ║
║   Hash                 ║                   ║                       ║
║   PFS Group            ║                   ║                       ║
║   Lifetime (sec)       ║                   ║                       ║
╠════════════════════════╬═══════════════════╬═══════════════════════╣
║ PRE-SHARED KEY         ║ (exchanged securely — do not email       ║
║                        ║  in plaintext)                            ║
╠════════════════════════╬═══════════════════╬═══════════════════════╣
║ MAINTENANCE WINDOW     ║                   ║                       ║
╚════════════════════════╩═══════════════════╩═══════════════════════╝
```

---

## Appendix B — Pre-Cutover Communication Template

Send this to the remote admin before replacing your MX:

<details>
<summary><strong>Click to expand — Ready-to-send template</strong></summary>

> **Subject: VPN Tunnel Maintenance — Hardware Replacement at [Your Site Name]**
>
> Hi [Remote Admin Name],
>
> We are replacing our VPN gateway at [Your Site Name] during a scheduled maintenance window. I wanted to coordinate with you to ensure minimal disruption to our site-to-site tunnel.
>
> **Maintenance Window:**
> - Date: [Date]
> - Time: [Start Time] – [End Time] ([Timezone])
> - Expected tunnel downtime: approximately [15–60] minutes
>
> **What's changing:**
> - Our VPN gateway hardware is being replaced
> - Our public WAN IP will change from [Old IP] to [New IP]
>   *(or: Our public WAN IP will remain the same: [IP])*
>
> **What's NOT changing:**
> - Pre-shared key (same as current)
> - IPsec parameters (same encryption, hash, DH group, lifetimes)
> - Our local subnets (same traffic selectors)
>
> **What I need from you:**
> - At the start of the maintenance window, please update the peer IP in your VPN configuration from [Old IP] to [New IP]
>   *(or: No action required on your side if the IP isn't changing)*
> - Be available by phone at [Your Phone Number] during the window in case we need to troubleshoot
>
> **Rollback plan:**
> If we encounter issues, we will revert to the existing hardware. If you've already updated the peer IP, I'll ask you to change it back to [Old IP].
>
> Please confirm this window works for your schedule and that you'll be available.
>
> Thanks,
> [Your Name]
> [Your Phone]
> [Your Email]

</details>

---

*— End of Document —*
