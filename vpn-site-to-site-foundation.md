# Site-to-Site VPN Foundations

> **A vendor-neutral reference for IPsec site-to-site VPN — the concepts, protocols, and decisions that apply regardless of what hardware sits at either end of the tunnel.**
>
> | | |
> |---|---|
> | **Document Type** | Reference / Knowledge Base |
> | **Date** | April 2026 |
> | **Status** | ✅ Complete |
> | **Audience** | Network administrators building or maintaining IPsec VPN tunnels |
> | **Companion Docs** | *Meraki MX Site-to-Site VPN* (vendor-specific procedures) |

---

## Table of Contents

- [What Is a Site-to-Site VPN?](#what-is-a-site-to-site-vpn)
- [IPsec — The Protocol Suite](#ipsec--the-protocol-suite)
  - [Why IPsec?](#why-ipsec)
  - [Transport Mode vs Tunnel Mode](#transport-mode-vs-tunnel-mode)
  - [ESP vs AH](#esp-vs-ah)
- [IKE — How Tunnels Are Built](#ike--how-tunnels-are-built)
  - [Phase 1 — The Handshake](#phase-1--the-handshake)
  - [Phase 2 — The Data Tunnel](#phase-2--the-data-tunnel)
  - [IKEv1 vs IKEv2](#ikev1-vs-ikev2)
  - [The Full Negotiation — Visual Summary](#the-full-negotiation--visual-summary)
- [Cryptographic Parameters](#cryptographic-parameters)
  - [Encryption Algorithms](#encryption-algorithms)
  - [Hash / Integrity Algorithms](#hash--integrity-algorithms)
  - [Diffie-Hellman Groups](#diffie-hellman-groups)
  - [Recommended Parameter Sets](#recommended-parameter-sets)
  - [The Golden Rule — Both Sides Must Match](#the-golden-rule--both-sides-must-match)
- [Authentication](#authentication)
  - [Pre-Shared Keys (PSK)](#pre-shared-keys-psk)
  - [Certificate-Based Authentication](#certificate-based-authentication)
  - [Which to Use](#which-to-use)
- [Peer Identity](#peer-identity)
- [Traffic Selectors — What Goes in the Tunnel](#traffic-selectors--what-goes-in-the-tunnel)
  - [The Non-Overlapping Subnet Rule](#the-non-overlapping-subnet-rule)
  - [Split Tunnel vs Full Tunnel](#split-tunnel-vs-full-tunnel)
- [Routing with VPN Tunnels](#routing-with-vpn-tunnels)
  - [Policy-Based VPN](#policy-based-vpn)
  - [Route-Based VPN](#route-based-vpn)
  - [When It Matters](#when-it-matters)
- [NAT Traversal (NAT-T)](#nat-traversal-nat-t)
- [MTU, MSS, and Fragmentation](#mtu-mss-and-fragmentation)
  - [The Problem](#the-problem)
  - [The Fix — MSS Clamping](#the-fix--mss-clamping)
  - [Overhead Reference Table](#overhead-reference-table)
- [Dead Peer Detection (DPD)](#dead-peer-detection-dpd)
- [Perfect Forward Secrecy (PFS)](#perfect-forward-secrecy-pfs)
- [SA Lifetimes and Rekeying](#sa-lifetimes-and-rekeying)
- [Firewall Requirements](#firewall-requirements)
- [DNS Across the Tunnel](#dns-across-the-tunnel)
- [Redundancy and Failover](#redundancy-and-failover)
- [Troubleshooting Methodology](#troubleshooting-methodology)
  - [Step 1 — Is the Tunnel Up?](#step-1--is-the-tunnel-up)
  - [Step 2 — Which Phase Failed?](#step-2--which-phase-failed)
  - [Step 3 — Common Failures by Phase](#step-3--common-failures-by-phase)
  - [Step 4 — Traffic Flows but Something Is Wrong](#step-4--traffic-flows-but-something-is-wrong)
  - [One-Sided Troubleshooting](#one-sided-troubleshooting)
- [Coordination Checklist — Parameter Exchange](#coordination-checklist--parameter-exchange)
- [Glossary](#glossary)

---

## What Is a Site-to-Site VPN?

A site-to-site VPN creates an encrypted tunnel between two networks across an untrusted transport (typically the public internet). Traffic entering the tunnel at one site exits at the other, decrypted and delivered to the remote LAN as if the two sites were directly connected.

Unlike remote-access VPN (where individual clients connect), site-to-site VPN is **always-on** and operates between **network devices** — firewalls, routers, or dedicated VPN appliances. End users and their devices have no awareness the tunnel exists; their traffic is encapsulated transparently.

```
┌──────────────┐                                    ┌──────────────┐
│   Site A      │         Encrypted Tunnel           │   Site B      │
│  10.1.0.0/24  │◄══════════════════════════════════►│  10.2.0.0/24  │
│   Firewall A  │──── Public Internet (untrusted) ───│   Firewall B  │
└──────────────┘                                    └──────────────┘
```

> 💡 **Key concept:** The tunnel is between the *firewalls* (or VPN gateways). LAN devices at each site route to their local gateway, which decides whether traffic goes into the tunnel or out to the internet.

---

## IPsec — The Protocol Suite

### Why IPsec?

IPsec (Internet Protocol Security) is the industry standard for site-to-site VPN because it operates at **Layer 3** — it encrypts and authenticates IP packets regardless of the application or transport protocol above it. Everything from web traffic to VoIP to file shares gets the same protection without any application-level changes.

### Transport Mode vs Tunnel Mode

| Mode | What's Encrypted | Use Case |
|:-----|:----------------|:---------|
| **Transport** | Only the payload (data) of the original IP packet; the original IP header is preserved | Host-to-host encryption (rare in site-to-site) |
| **Tunnel** | The **entire** original IP packet, including the header; a new outer IP header is added with the tunnel endpoints' public IPs | Site-to-site VPN (**this is what you use**) |

> ℹ️ For site-to-site VPN, you are **always** using tunnel mode. If a configuration asks you to choose, select tunnel mode.

### ESP vs AH

| Protocol | IP Protocol # | Provides | Used In Practice? |
|:---------|:-------------|:---------|:-----------------|
| **ESP** (Encapsulating Security Payload) | 50 | Encryption **and** integrity | ✅ Yes — this is what you use |
| **AH** (Authentication Header) | 51 | Integrity only (no encryption) | ❌ Rarely — legacy, incompatible with NAT |

> ⚠️ **AH does not encrypt traffic.** It only ensures packets aren't tampered with. Because AH includes the IP header in its integrity check, it breaks when NAT changes the source IP. ESP is the standard for all modern site-to-site VPN deployments.

---

## IKE — How Tunnels Are Built

IKE (Internet Key Exchange) is the protocol that **negotiates** the tunnel's security parameters and **builds** the Security Associations (SAs) that IPsec uses to encrypt traffic. It runs on **UDP port 500** (or UDP 4500 when NAT is detected).

IKE operates in two distinct phases, and understanding the boundary between them is essential for troubleshooting.

### Phase 1 — The Handshake

**Purpose:** Authenticate the two peers to each other and establish a secure, encrypted management channel (the IKE SA) through which Phase 2 negotiation will occur.

**What's negotiated:**

| Parameter | What It Means |
|:----------|:-------------|
| Encryption algorithm | How the IKE management channel is encrypted (e.g., AES-256) |
| Hash / integrity algorithm | How IKE messages are verified as untampered (e.g., SHA-256) |
| Diffie-Hellman group | The key exchange math used to derive shared secret material (e.g., Group 14) |
| Authentication method | How peers prove identity — pre-shared key or certificate |
| SA lifetime | How long before Phase 1 renegotiates (typically 28,800 seconds / 8 hours) |

**Result:** An authenticated, encrypted channel between the two peers. No user traffic flows yet — this channel exists solely to protect the Phase 2 negotiation.

> 💡 **Analogy:** Phase 1 is two people verifying each other's identity and agreeing on a secret language before they discuss the actual business.

### Phase 2 — The Data Tunnel

**Purpose:** Negotiate the IPsec SA that will actually encrypt user traffic. This negotiation happens *inside* the Phase 1 channel.

**What's negotiated:**

| Parameter | What It Means |
|:----------|:-------------|
| Encryption algorithm | How user traffic is encrypted (e.g., AES-256) |
| Hash / integrity algorithm | How user traffic is verified (e.g., SHA-256) |
| Traffic selectors | Which subnets/networks are allowed through the tunnel (e.g., 10.1.0.0/24 ↔ 10.2.0.0/24) |
| SA lifetime | How long before Phase 2 renegotiates (typically 3,600 seconds / 1 hour) |
| PFS group | Whether to perform a fresh DH exchange for Phase 2 keys (optional but recommended) |
| Protocol | ESP (always, in practice) |

**Result:** One or more IPsec SAs carrying encrypted user traffic between the defined subnets.

> ⚠️ **Critical:** Phase 1 and Phase 2 have **separate** parameter sets. It is common for Phase 1 to succeed and Phase 2 to fail because of a mismatch — check both independently during troubleshooting.

### IKEv1 vs IKEv2

| Feature | IKEv1 | IKEv2 |
|:--------|:------|:------|
| **Messages to establish tunnel** | 9 (Main Mode) or 6 (Aggressive Mode) | 4 |
| **NAT-T** | Bolt-on extension | Built-in |
| **Reliability** | No message acknowledgment | Built-in retry/acknowledgment |
| **Multiple subnets** | One Phase 2 SA per subnet pair (each requires separate negotiation) | One SA can carry multiple traffic selectors |
| **EAP support** | No | Yes |
| **Compatibility** | Universal (legacy) | Most modern equipment (2012+) |
| **Recommendation** | Use when the remote end doesn't support IKEv2 | **Use IKEv2 when both sides support it** |

> ℹ️ **Practical note:** When working with a remote admin who controls the other end, confirm IKE version early. If either side is constrained to IKEv1, both sides must use IKEv1 — they don't interoperate.

### The Full Negotiation — Visual Summary

```
Site A (Initiator)                              Site B (Responder)
       │                                              │
       │──── Phase 1: IKE SA Proposal ───────────────►│
       │◄─── Phase 1: Accepted Proposal ──────────────│
       │──── Phase 1: DH Key Exchange ───────────────►│
       │◄─── Phase 1: DH Key Exchange ────────────────│
       │──── Phase 1: Authentication ────────────────►│
       │◄─── Phase 1: Authentication ─────────────────│
       │                                              │
       │    ══════ IKE SA Established ══════          │
       │    (encrypted management channel)            │
       │                                              │
       │──── Phase 2: IPsec SA Proposal ─────────────►│  ┐
       │◄─── Phase 2: Accepted Proposal ──────────────│  ├ Inside the
       │──── Phase 2: Confirmation ──────────────────►│  │ Phase 1 channel
       │◄─── Phase 2: Confirmation ───────────────────│  ┘
       │                                              │
       │    ══════ IPsec SA Established ══════        │
       │    (encrypted user traffic flows)            │
       │                                              │
       │◄═══════ Encrypted Data Traffic ═════════════►│
```

---

## Cryptographic Parameters

### Encryption Algorithms

| Algorithm | Key Length | Status | Notes |
|:----------|:----------|:-------|:------|
| **AES-256** | 256-bit | ✅ Recommended | Current standard; use this |
| **AES-128** | 128-bit | ✅ Acceptable | Slightly faster; still secure |
| **AES-192** | 192-bit | ✅ Acceptable | Rarely used — pick 128 or 256 |
| **3DES** | 168-bit | ⚠️ Deprecated | Slow, effectively 112-bit security; legacy only |
| **DES** | 56-bit | 🚫 Do not use | Broken since the 1990s |

### Hash / Integrity Algorithms

| Algorithm | Output Size | Status | Notes |
|:----------|:-----------|:-------|:------|
| **SHA-256** | 256-bit | ✅ Recommended | Current standard |
| **SHA-384** | 384-bit | ✅ Acceptable | Higher overhead, marginal benefit over SHA-256 |
| **SHA-512** | 512-bit | ✅ Acceptable | Same as above |
| **SHA-1** | 160-bit | ⚠️ Deprecated | Collision attacks proven; avoid in new deployments |
| **MD5** | 128-bit | 🚫 Do not use | Broken; trivially exploitable |

### Diffie-Hellman Groups

| Group | Key Size | Status | Notes |
|:------|:---------|:-------|:------|
| **Group 14** | 2048-bit | ✅ Minimum recommended | Most compatible secure option |
| **Group 19** | 256-bit ECP | ✅ Recommended | Elliptic curve; faster, equally secure |
| **Group 20** | 384-bit ECP | ✅ Recommended | Elliptic curve; higher security margin |
| **Group 5** | 1536-bit | ⚠️ Deprecated | Below current security recommendations |
| **Group 2** | 1024-bit | 🚫 Do not use | Breakable by well-funded adversaries |
| **Group 1** | 768-bit | 🚫 Do not use | Trivially broken |

### Recommended Parameter Sets

For new deployments where both sides support modern crypto:

| Parameter | Phase 1 Value | Phase 2 Value |
|:----------|:-------------|:-------------|
| **Encryption** | AES-256 | AES-256 |
| **Integrity** | SHA-256 | SHA-256 |
| **DH Group** | 14 (or 19/20 if supported) | 14 (PFS) |
| **Lifetime** | 28,800 seconds (8 hours) | 3,600 seconds (1 hour) |
| **IKE Version** | IKEv2 | — |

For interop with older or constrained hardware:

| Parameter | Phase 1 Value | Phase 2 Value |
|:----------|:-------------|:-------------|
| **Encryption** | AES-256 | AES-256 |
| **Integrity** | SHA-256 | SHA-256 |
| **DH Group** | 14 | 14 (PFS) |
| **Lifetime** | 28,800 seconds | 3,600 seconds |
| **IKE Version** | IKEv1 | — |

### The Golden Rule — Both Sides Must Match

> 🚫 **This is the #1 cause of VPN tunnel failures.**
>
> Every cryptographic parameter — encryption, hash, DH group, and lifetime — must be **identical** on both ends for each phase independently. A mismatch in any single parameter causes that phase to fail. There is no negotiation fallback; the sides either agree exactly or the tunnel does not establish.

If you control only one side, you must coordinate these values with the remote administrator **before** either side starts configuring. See the [Coordination Checklist](#coordination-checklist--parameter-exchange) at the end of this document.

---

## Authentication

### Pre-Shared Keys (PSK)

A shared passphrase configured identically on both sides. During Phase 1, each peer proves it knows the PSK without transmitting it directly (it's used as input to the key derivation).

**Strengths:**

- Simple to configure
- No infrastructure required (no CA, no certificate management)

**Weaknesses:**

- The PSK must be exchanged out-of-band (securely, before configuration)
- If compromised, all tunnels using that PSK are compromised
- Changing the PSK requires coordinated action on both ends simultaneously

**PSK best practices:**

- Minimum 20 characters
- Randomly generated — not a passphrase a human would create
- Exchanged via encrypted channel (not plaintext email)
- Unique per tunnel (don't reuse PSKs across different site pairs)
- Rotated on a defined schedule or after any personnel change at either site

### Certificate-Based Authentication

Each peer presents an X.509 certificate signed by a Certificate Authority (CA) that the other peer trusts. No shared secret to exchange.

**Strengths:**

- No PSK to share or rotate
- Can be tied to device identity
- Scales better for many tunnels

**Weaknesses:**

- Requires PKI infrastructure (CA, certificate issuance, renewal, revocation)
- More complex initial setup
- Certificate expiration can silently kill tunnels if not monitored

### Which to Use

For most small-to-midsize site-to-site deployments — particularly cross-organization where you don't control both ends — **PSK is the practical choice.** Certificate auth becomes worthwhile when you're managing many tunnels and have PKI infrastructure already in place.

---

## Peer Identity

During Phase 1, each side needs to identify itself to the other. The peer identity (also called Local ID or Remote ID) tells the other end *who* is connecting. Common identity types:

| Type | Value | Typical Use |
|:-----|:------|:-----------|
| **IP Address** | Public WAN IP of the VPN gateway | Most common for PSK auth |
| **FQDN** | `vpn.example.com` | Useful when WAN IP is dynamic |
| **User FQDN** | `admin@example.com` | Email-style identifier |

> ⚠️ **Both sides must agree on identity type and value.** If Site A identifies itself as its WAN IP but Site B expects an FQDN, Phase 1 authentication will fail even if the PSK is correct.

When you control only one side, explicitly confirm with the remote admin:

1. What identity **you** should present (your Local ID)
2. What identity **they** will present (your Remote ID / their Local ID)

---

## Traffic Selectors — What Goes in the Tunnel

Traffic selectors (also called proxy IDs, encryption domains, or interesting traffic) define **which subnets** are permitted to traverse the tunnel. Both sides must agree.

```
Site A traffic selector:   Local = 10.1.0.0/24,  Remote = 10.2.0.0/24
Site B traffic selector:   Local = 10.2.0.0/24,  Remote = 10.1.0.0/24
```

> ℹ️ These are **mirror images** of each other. Site A's "local" is Site B's "remote" and vice versa.

### The Non-Overlapping Subnet Rule

> 🚫 **The subnets at each site MUST NOT overlap.**
>
> If both sites use the same subnet, the VPN gateways cannot distinguish local traffic from tunnel-bound traffic. Routing breaks. There is no workaround short of renumbering one side or adding a NAT translation layer within the tunnel (complex and fragile).

**Before building any tunnel, confirm:**

- [ ] Local LAN subnet is documented
- [ ] Remote LAN subnet is documented
- [ ] There is **zero overlap** between them — including all VLANs and secondary subnets at both sites

### Split Tunnel vs Full Tunnel

| Mode | What Goes in the Tunnel | Internet Traffic |
|:-----|:----------------------|:----------------|
| **Split tunnel** | Only traffic destined for the remote site's subnets | Goes out the local internet connection directly |
| **Full tunnel** | All traffic, including internet-bound | Exits at the remote site's internet connection |

**Site-to-site VPN almost always uses split tunnel.** Full tunnel is a remote-access VPN concept where you want to inspect all of a remote user's traffic. For site-to-site, you only tunnel inter-site traffic.

---

## Routing with VPN Tunnels

Once the tunnel is up, each site needs to know that traffic for the remote subnet should be sent into the tunnel. Two approaches:

### Policy-Based VPN

The VPN configuration itself defines which traffic enters the tunnel based on source and destination subnet. No route table entry is needed — the VPN policy acts as an implicit route.

**Characteristics:**

- Traffic selectors = routing policy
- Simpler configuration
- Limited flexibility (can't use dynamic routing protocols across the tunnel)
- Most Meraki and consumer-grade firewalls use this model

### Route-Based VPN

The tunnel creates a **virtual interface** (VTI — Virtual Tunnel Interface). A route in the routing table points the remote subnet at this interface. The VPN encrypts anything routed into the VTI.

**Characteristics:**

- Tunnel is a routable interface
- Supports dynamic routing protocols (OSPF, BGP) across the tunnel
- More flexible (multiple subnets without multiple Phase 2 SAs)
- pfSense, Cisco IOS, and most enterprise platforms support this

### When It Matters

If both sides are Meraki, this is invisible to you — Meraki handles it. If you're connecting Meraki to pfSense, you need to understand that Meraki uses a policy-based model while pfSense supports both. Phase 2 traffic selectors must align regardless of the underlying model.

---

## NAT Traversal (NAT-T)

### The Problem

ESP (IP Protocol 50) is not TCP or UDP — it has no port numbers. Most NAT devices only know how to track connections with port numbers (TCP/UDP). When ESP packets hit a NAT device, the NAT device either drops them or can't properly track the return traffic.

### The Solution

NAT-T detects NAT between the peers during Phase 1 and automatically encapsulates ESP packets inside **UDP port 4500**. This gives NAT devices port numbers to track.

```
Without NAT-T:   [IP Header] [ESP Header] [Encrypted Payload]
                   ↑ NAT changes source IP, ESP has no port → breaks

With NAT-T:      [IP Header] [UDP 4500] [ESP Header] [Encrypted Payload]
                   ↑ NAT changes source IP and tracks via UDP port → works
```

### When You Need It

- An ISP router doing NAT sits between your firewall and the public internet
- Your VPN gateway has a private WAN IP (e.g., 192.168.x.x or 10.x.x.x) and the public IP is on an upstream device
- You're behind a carrier-grade NAT (CGNAT — common in mobile/4G/5G ISPs)

> ✅ **Recommendation:** Always enable NAT-T (it's on by default in most platforms). It has negligible overhead when NAT isn't present, and it prevents a silent failure if NAT is introduced later (e.g., ISP change).

---

## MTU, MSS, and Fragmentation

### The Problem

IPsec adds overhead to every packet. A standard Ethernet frame has a maximum payload (MTU) of **1500 bytes**. After IPsec encapsulation (ESP header, IV, padding, authentication, and the outer IP header), the available space for actual user data is smaller — typically **1400–1422 bytes** depending on the specific cipher suite and whether NAT-T is active.

If an application sends a 1500-byte packet and the VPN gateway tries to encapsulate it, the result exceeds 1500 bytes. The gateway must fragment or drop the packet. Fragmentation causes performance degradation; many firewalls and some ISPs drop fragments entirely.

### The Fix — MSS Clamping

**MSS (Maximum Segment Size)** is a TCP option negotiated during the TCP handshake. MSS clamping tells TCP clients to limit their payload size so that the resulting IP packet, after IPsec encapsulation, stays within the path MTU.

| Setting | Typical Value | Where It's Set |
|:--------|:-------------|:--------------|
| **TCP MSS Clamping** | 1360–1400 bytes | On the VPN gateway (both sides) |
| **Interface MTU** | 1400 (on VTI/tunnel interface) | On platforms with virtual tunnel interfaces |

> ⚠️ **MSS clamping only fixes TCP traffic.** UDP and other protocols don't negotiate MSS. For UDP-heavy workloads (VoIP, video), ensure the applications use packets small enough to avoid fragmentation, or set DF (Don't Fragment) bit handling appropriately on the VPN gateway.

### Overhead Reference Table

| Configuration | IPsec Overhead | Effective MTU | Recommended MSS |
|:-------------|:--------------|:-------------|:----------------|
| ESP + AES-256 + SHA-256, no NAT-T | ~58 bytes | ~1442 | 1400 |
| ESP + AES-256 + SHA-256, with NAT-T | ~66 bytes | ~1434 | 1360–1380 |
| ESP + AES-128 + SHA-256, no NAT-T | ~58 bytes | ~1442 | 1400 |

> 💡 **When in doubt, set MSS to 1360.** It's conservative but safe for virtually any combination of encryption, NAT-T, and path characteristics.

---

## Dead Peer Detection (DPD)

DPD sends periodic probes between peers. If the remote end stops responding, the local side tears down the SAs and can attempt to re-establish the tunnel.

| Parameter | Typical Value | Notes |
|:----------|:-------------|:------|
| **DPD interval** | 10–30 seconds | How often probes are sent when idle |
| **DPD timeout / retries** | 3–5 retries | How many missed probes before declaring the peer dead |
| **DPD action** | Restart / Clear | What happens when a peer is declared dead |

> ✅ **Always enable DPD.** Without it, a tunnel can appear "up" on your side for hours after the remote end has crashed, rebooted, or lost connectivity. Traffic sent into a dead tunnel black-holes silently.

---

## Perfect Forward Secrecy (PFS)

Without PFS, Phase 2 keys are derived from Phase 1 key material. If an attacker compromises the Phase 1 key, they can retroactively decrypt all Phase 2 traffic.

With PFS enabled, Phase 2 performs **its own fresh Diffie-Hellman exchange**, generating new key material independent of Phase 1. Compromising Phase 1 does not compromise Phase 2 sessions.

> ✅ **Recommendation:** Enable PFS for Phase 2 using the same DH group as Phase 1 (e.g., Group 14). The performance cost is minimal. Both sides must agree on the PFS group.

---

## SA Lifetimes and Rekeying

Security Associations have configurable lifetimes. When a lifetime expires, the SA is renegotiated (rekeyed) before it expires to avoid a traffic interruption.

| SA Type | Default Lifetime | What Happens at Expiry |
|:--------|:----------------|:----------------------|
| **Phase 1 (IKE SA)** | 28,800 seconds (8 hours) | Rekeyed; Phase 2 SAs remain active if rekey succeeds |
| **Phase 2 (IPsec SA)** | 3,600 seconds (1 hour) | Rekeyed inside the existing Phase 1 channel |

> ⚠️ **Lifetime mismatch between sides:** If Side A has a Phase 2 lifetime of 3,600 seconds and Side B has 7,200 seconds, the shorter side initiates the rekey. This *usually* works but can cause issues on some platforms. Best practice: **match lifetimes exactly on both sides.**

---

## Firewall Requirements

The following ports and protocols must be permitted **in both directions** on any firewall or ACL between the two VPN peers (including ISP routers, upstream firewalls, and cloud security groups):

| Protocol | Port / Number | Purpose |
|:---------|:-------------|:--------|
| **UDP** | **500** | IKE negotiation |
| **UDP** | **4500** | NAT Traversal (IKE + ESP over UDP) |
| **IP Protocol** | **50** (ESP) | Encrypted data transport (only if NAT-T is not in use) |

> ℹ️ If NAT-T is active (which it usually is), **all** IPsec traffic — including ESP — is encapsulated in UDP 4500. In that case, only UDP 500 and UDP 4500 are needed. However, permitting ESP (protocol 50) as well is harmless and prevents issues if NAT-T is not detected.

---

## DNS Across the Tunnel

Once the tunnel is up and IP traffic flows between sites, DNS is often the next thing that breaks.

**The problem:** Site A's devices use a local DNS resolver (e.g., the firewall or a local DNS server). They can reach Site B's IP addresses through the tunnel, but they can't *resolve* Site B's hostnames because they're not querying Site B's DNS.

**Solutions:**

| Approach | How It Works | Complexity |
|:---------|:------------|:-----------|
| **Conditional forwarding** | Site A's DNS forwards queries for the remote site's zone to that site's DNS server through the tunnel | Low — one rule per remote zone |
| **Shared DNS infrastructure** | Both sites use the same DNS server (typically at the hub site, reachable through the tunnel) | Low — but creates a dependency (if tunnel goes down, spoke loses DNS) |
| **Hosts file** | Static entries for critical remote hosts | None — but doesn't scale and requires manual maintenance |

> ✅ **Recommendation:** Conditional forwarding is the cleanest approach. The hub site's DNS server handles its own zone, the spoke site's DNS server handles its zone, and each forwards the other's zone through the tunnel. Internet DNS resolution remains local to each site.

> ⚠️ **Don't forget:** If you use the VPN gateway as the DNS server (common with Meraki MX and pfSense), the DNS server's IP must be **reachable through the tunnel** from the remote site. Confirm the DNS server's IP is within the traffic selectors.

---

## Redundancy and Failover

### Dual-WAN Failover

If the VPN gateway has two WAN connections, the tunnel can be configured to fail over to the secondary WAN. Considerations:

- **Public IP change:** The secondary WAN likely has a different public IP. The remote end must be configured to accept the new peer IP — or use FQDN-based peer identity with dynamic DNS.
- **Tunnel renegotiation:** When the WAN changes, the tunnel tears down and re-establishes. Expect 30–120 seconds of downtime depending on DPD timers and rekey speed.
- **Remote admin notification:** If you control only one side and your public IP changes during failover, the remote admin's config may need updating unless their side accepts connections from both of your WAN IPs.

### Dual-Tunnel Redundancy

Two parallel tunnels — one per WAN — with routing to prefer the primary and fail over to the secondary. More complex but eliminates the renegotiation delay.

---

## Troubleshooting Methodology

### Step 1 — Is the Tunnel Up?

Check the VPN status on your device. Look for:

- **Phase 1 SA** — is it established? Most platforms show this as IKE SA or ISAKMP SA.
- **Phase 2 SA** — is it established? Look for IPsec SA with traffic counters.

If both show "established" with incrementing byte/packet counters, the tunnel is up and carrying traffic.

### Step 2 — Which Phase Failed?

| Symptom | Likely Phase |
|:--------|:------------|
| No IKE SA present at all | Phase 1 — peers aren't communicating or can't agree on parameters |
| IKE SA established but no IPsec SA | Phase 2 — IKE negotiated but traffic selector or crypto mismatch |
| Both SAs present but no traffic flowing | Routing, firewall, or traffic selector mismatch |

### Step 3 — Common Failures by Phase

**Phase 1 Failures:**

| Error / Symptom | Probable Cause | Fix |
|:---------------|:--------------|:----|
| No response from remote peer | Firewall blocking UDP 500/4500; wrong peer IP; remote end not configured | Verify reachability, port rules, and peer IP on both sides |
| `INVALID_KEY_INFORMATION` | PSK mismatch | Re-enter the PSK on both sides; watch for trailing spaces and character encoding |
| `NO_PROPOSAL_CHOSEN` | Encryption, hash, or DH group mismatch | Compare Phase 1 parameters side-by-side; see the Coordination Checklist |
| Authentication failed | PSK wrong, or peer identity mismatch | Verify PSK and Local/Remote ID settings |
| `INVALID_ID_INFORMATION` | Peer identity type or value mismatch | Confirm both sides agree on identity type (IP, FQDN) and value |

**Phase 2 Failures:**

| Error / Symptom | Probable Cause | Fix |
|:---------------|:--------------|:----|
| `NO_PROPOSAL_CHOSEN` | Phase 2 encryption, hash, or PFS mismatch | Compare Phase 2 parameters; PFS group is a common miss |
| `INVALID_ID_INFORMATION` | Traffic selector mismatch | Verify subnets are mirror images on each side |
| Phase 2 constantly rekeying | Lifetime mismatch (shorter side rekeys, longer side hasn't expired yet) | Match lifetimes |

### Step 4 — Traffic Flows but Something Is Wrong

| Symptom | Probable Cause | Fix |
|:--------|:--------------|:----|
| Ping works, but applications are slow or fail | MTU/fragmentation issue | Implement MSS clamping (see [MTU section](#mtu-mss-and-fragmentation)) |
| Can reach some remote IPs but not others | Traffic selectors too narrow | Expand traffic selectors to include all required subnets |
| DNS resolution fails across the tunnel | DNS not configured to forward across the tunnel | See [DNS Across the Tunnel](#dns-across-the-tunnel) |
| Tunnel drops periodically | DPD timeouts, flapping WAN, or SA lifetime mismatch | Check DPD settings, WAN stability, and lifetime alignment |
| One-way traffic (A→B works, B→A doesn't) | Asymmetric routing, or one side's firewall blocking return traffic | Check routing and firewall rules on both sides |

### One-Sided Troubleshooting

When you only control one end of the tunnel, your diagnostic tools are limited. Here's what you can determine from your side alone:

**You CAN determine:**

- Whether your Phase 1 is initiating (sending proposals)
- Whether you're receiving any response from the remote peer
- Which specific error your side reports (`NO_PROPOSAL_CHOSEN`, `INVALID_KEY`, etc.)
- Whether Phase 1 succeeds but Phase 2 fails (narrows the problem)
- Your local parameters (to compare against what the remote admin claims)

**You CANNOT determine (must ask the remote admin):**

- Whether the remote side is even receiving your proposals
- What error the remote side sees
- Whether the remote side has your correct peer IP / PSK / identity
- Whether there's an upstream firewall on *their* side blocking traffic

**Effective communication with the remote admin:**

When reporting a VPN issue to the remote admin, provide actionable detail rather than "the VPN isn't working":

1. State which phase your side reached (Phase 1 initiating, Phase 1 complete, Phase 2 failing, etc.)
2. Include the specific error message or log entry from your device
3. Confirm the peer IP you're targeting and the identity you're presenting
4. Ask them to check their logs for corresponding entries from your IP
5. Ask whether UDP 500/4500 is permitted inbound on their side

---

## Coordination Checklist — Parameter Exchange

Before either side configures anything, **both administrators fill in their column** of this table and exchange it:

| Parameter | Your Site | Remote Site |
|:----------|:---------|:-----------|
| **Public WAN IP** | __________________ | __________________ |
| **Backup WAN IP** (if applicable) | __________________ | __________________ |
| **Local LAN Subnet(s)** | __________________ | __________________ |
| **IKE Version** | __________________ | __________________ |
| **Phase 1 Encryption** | __________________ | __________________ |
| **Phase 1 Hash** | __________________ | __________________ |
| **Phase 1 DH Group** | __________________ | __________________ |
| **Phase 1 Lifetime** | __________________ | __________________ |
| **Authentication Method** | PSK / Certificate | PSK / Certificate |
| **Pre-Shared Key** | *(exchanged securely)* | *(exchanged securely)* |
| **Local ID Type & Value** | __________________ | __________________ |
| **Remote ID Type & Value** | __________________ | __________________ |
| **Phase 2 Encryption** | __________________ | __________________ |
| **Phase 2 Hash** | __________________ | __________________ |
| **Phase 2 PFS Group** | __________________ | __________________ |
| **Phase 2 Lifetime** | __________________ | __________________ |
| **NAT between peers?** | Yes / No / Unknown | Yes / No / Unknown |
| **DPD enabled?** | __________________ | __________________ |
| **Contact name & phone** | __________________ | __________________ |
| **Maintenance window** | __________________ | __________________ |

> 💡 **Tip:** Send this as a simple table in a secure channel (encrypted email or out-of-band). Get the remote admin to fill in their column and return it. Do not start configuring until both columns are complete and matching.

---

## Glossary

| Term | Definition |
|:-----|:----------|
| **AES** | Advanced Encryption Standard — symmetric encryption algorithm |
| **AH** | Authentication Header — IPsec protocol providing integrity without encryption |
| **CGNAT** | Carrier-Grade NAT — ISP-level NAT that can interfere with VPN |
| **DH** | Diffie-Hellman — key exchange algorithm for deriving shared secrets |
| **DPD** | Dead Peer Detection — keepalive mechanism for IPsec tunnels |
| **ESP** | Encapsulating Security Payload — IPsec protocol providing encryption and integrity |
| **FQDN** | Fully Qualified Domain Name |
| **IKE** | Internet Key Exchange — protocol that negotiates and establishes IPsec SAs |
| **IPsec** | Internet Protocol Security — protocol suite for encrypted IP communication |
| **MSS** | Maximum Segment Size — TCP option limiting payload size |
| **MTU** | Maximum Transmission Unit — largest packet size an interface can transmit |
| **NAT** | Network Address Translation |
| **NAT-T** | NAT Traversal — encapsulates ESP in UDP to survive NAT |
| **PFS** | Perfect Forward Secrecy — independent key generation per Phase 2 SA |
| **PKI** | Public Key Infrastructure — certificate-based trust framework |
| **PSK** | Pre-Shared Key — shared secret for IKE authentication |
| **SA** | Security Association — the negotiated security relationship between two peers |
| **SPI** | Security Parameter Index — identifies an SA on a given peer |
| **VTI** | Virtual Tunnel Interface — logical interface representing an IPsec tunnel |

---

*— End of Document —*
