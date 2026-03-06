# 🔐 Meraki MAC-Based Access Control Without RADIUS

## Locking Down a Crowded SSID Using Allow Lists + Default Deny

> **Document Type:** How-To + Reference Guide
> **Platform:** Cisco Meraki MX Security Appliance + MR Access Points
> **Applies To:** Any MX/MR environment where PSK cannot be changed immediately
> **Complexity:** Intermediate
> **Date:** March 2026

---

## 📋 Table of Contents

1. [The Problem](#1-the-problem)
2. [How This Works](#2-how-this-works)
3. [Prerequisites](#3-prerequisites)
4. [Example Environment](#4-example-environment)
5. [Step-by-Step Configuration](#5-step-by-step-configuration)
   - [Step 1 — Collect Legitimate MAC Addresses](#step-1--collect-legitimate-mac-addresses)
   - [Step 2 — Pre-Add MACs to the Clients Page](#step-2--pre-add-macs-to-the-clients-page)
   - [Step 3 — Apply the Allow List Policy](#step-3--apply-the-allow-list-policy)
   - [Step 4 — Create the Default Deny Firewall Rule](#step-4--create-the-default-deny-firewall-rule)
   - [Step 5 — Verify End-to-End](#step-5--verify-end-to-end)
6. [Creating and Managing the Allow List — Full Reference](#6-creating-and-managing-the-allow-list--full-reference)
   - [How to Create the Allow List From Scratch](#how-to-create-the-allow-list-from-scratch)
   - [How to Add a Device That Has Never Connected](#how-to-add-a-device-that-has-never-connected)
   - [How to Add a Device That Has Already Connected](#how-to-add-a-device-that-has-already-connected)
   - [How to Remove a Device From the Allow List](#how-to-remove-a-device-from-the-allow-list)
   - [How to View All Currently Allow Listed Devices](#how-to-view-all-currently-allow-listed-devices)
7. [MAC Randomization — What You Need to Know](#7-mac-randomization--what-you-need-to-know)
8. [Limitations](#8-limitations)
9. [Troubleshooting](#9-troubleshooting)
10. [Lessons Learned](#10-lessons-learned)
11. [Checklist](#11-checklist)

---

## 1. The Problem

You have an SSID where:

- The PSK has been shared too broadly and **cannot be changed immediately**
- **Hundreds of unauthorized devices** are actively connected
- Only a **small, known set of devices** should have any access
- You need a way to **block everyone else at the traffic level** without RADIUS

> ⚠️ **This is not a long-term fix.** Changing the PSK is still the correct permanent solution. This guide solves the immediate problem of locking out unauthorized devices while that process is arranged.

---

## 2. How This Works

Meraki does not support blocking devices at *association* without RADIUS — unauthorized devices will still connect to the SSID and receive a DHCP lease. However, you can block them at the *traffic level*, which produces the same functional result:

```
┌─────────────────────────────────────────────────────────────┐
│                    SSID: corp-wireless                       │
│                                                             │
│   Device A — Authorized (Allow Listed)                      │
│       ✅ Connects → Gets IP → Traffic passes → Internet OK  │
│                                                             │
│   Device B — Unauthorized (no policy / default deny)        │
│       ⚠️  Connects → Gets IP → Traffic BLOCKED → Dead end   │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

**The mechanism:**

1. A default-deny firewall rule blocks **all traffic** from the wireless SSID subnet
2. The 15 legitimate devices are placed on the **Allow List** in the Meraki Dashboard
3. Allow Listed devices are **exempt from all firewall rules** — their traffic passes
4. Everyone else hits the deny wall — they have a Wi-Fi connection with no usable network

---

## 3. Prerequisites

| Requirement          | Details                                                                                                                                            |
| -------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Dashboard access** | Network Administrator or higher                                                                                                                    |
| **MX firmware**      | Any current supported release                                                                                                                      |
| **MAC addresses**    | Must know the MAC for each legitimate device — collected from the Dashboard Clients page (see [Step 1](#step-1--collect-legitimate-mac-addresses)) |
| **Existing SSID**    | SSID already operational — this guide adds controls on top                                                                                         |

> 💡 **No RADIUS server required.** This entire configuration is done through the Meraki Dashboard only.

---

## 4. Example Environment

> 📝 The values below are **fictional examples** used throughout this guide. Substitute your own network details.

### Network Details

| Parameter                 | Example Value                        |
| ------------------------- | ------------------------------------ |
| **Organization**          | Riverside Valley Municipal Authority |
| **SSID Name**             | `rv-staff`                           |
| **Employee VLAN ID**      | `110`                                |
| **Employee Subnet**       | `172.20.10.0/24`                     |
| **Employee Gateway (MX)** | `172.20.10.254`                      |
| **DHCP Pool**             | `172.20.10.10 – 172.20.10.200`       |

### Scope of the Problem

|                                        | Count |
| -------------------------------------- | ----- |
| Total devices connected to `rv-staff`  | ~300  |
| Authorized devices (known, legitimate) | 15    |
| Devices to be blocked                  | ~285  |

### Authorized Device List (Example)

| #   | Friendly Name  | MAC Address         | Device Type   |
| --- | -------------- | ------------------- | ------------- |
| 1   | Blair-Laptop   | `A4:C3:F0:11:22:33` | Dell Latitude |
| 2   | Morel-Laptop   | `B8:27:EB:44:55:66` | HP EliteBook  |
| 3   | Front-Desk-PC  | `00:1A:2B:3C:4D:5E` | Desktop       |
| 4   | Reception-iPad | `F4:5C:89:AB:CD:EF` | iPad          |
| 5   | Conf-Room-TV   | `DC:A6:32:12:34:56` | Smart TV      |
| ... | ...            | ...                 | ...           |
| 15  | Admin-Phone    | `3C:22:FB:78:9A:BC` | iPhone        |

> ⚠️ **On mobile devices:** Collect MACs from the **Meraki Dashboard Clients page**, not from the device's settings screen. The Dashboard shows the *randomized* MAC that Meraki actually sees — which is what you need to Allow List. See [Section 6](#6-mac-randomization--what-you-need-to-know) for details.

---

## 5. Step-by-Step Configuration

---

### Step 1 — Collect Legitimate MAC Addresses

Before configuring anything, you need the MAC address for each of the 15 authorized devices **as seen by Meraki** — not from device stickers or OS settings.

**Dashboard path:** `Network-wide → Monitor → Clients`

1. Filter by **SSID** → select `rv-staff`
2. Set the time window to **Last 7 days** to ensure all authorized devices appear
3. For each authorized device, record:
   - The **MAC address** shown in the Dashboard
   - A **friendly name** you'll recognize later

> ⚠️ **Critical:** Do not collect MACs from the device itself (e.g., iOS Settings → General → About → Wi-Fi Address). iOS and Android show the physical MAC there, but Meraki may be seeing a randomized MAC. The Dashboard is the authoritative source. See [Section 6](#6-mac-randomization--what-you-need-to-know).

---

### Step 2 — Get Authorized Devices Into the Clients Page

Before you can Apply the Allow List policy in Step 3, every authorized device must be visible in the Clients list. How you get them there depends on whether they've connected before.

**Dashboard path:** `Network-wide → Monitor → Clients`

---

#### ✅ Scenario A — Device has already connected to the network

It's already in the Clients list. **No action needed for this step.** Proceed to Step 3 and check the box next to it.

> 💡 In your scenario — 300 devices already on the network — your 15 authorized devices are almost certainly already in the list. Search by MAC address or device name to confirm. If all 15 are present, skip to Step 3.

---

#### 🆕 Scenario B — Device has never connected (needs to be pre-added)

Use this only for devices that have never appeared on the network:

1. Click the **`Add client`** button in the **top right corner** of the Clients page
2. Enter a **friendly name** (e.g., `Blair-Laptop`)
3. Enter the **MAC address** exactly as recorded in Step 1
4. Set **Policy** → leave as `Normal` for now — you will bulk-update in Step 3
5. Click **Save changes**
6. Repeat for each device that needs to be pre-added

> ⚠️ The `Add client` button is easy to miss — it sits in the top right of the Clients table, not in a sidebar or settings menu.

---

### Step 3 — Apply the Allow List Policy

Now apply the Allow List policy to all 15 authorized devices at once.

**Dashboard path:** `Network-wide → Monitor → Clients`

1. Check the box next to **each of your 15 authorized devices**
2. Click the **Policy** button at the top of the client list
3. Select **Allow list**
4. Click **Save**
5. Clients will display **whitelisted** in policy now

**What Allow Listed means:**

> The Allow Listed policy exempts a device from:
> 
> - All Layer 3 and Layer 7 firewall rules
> - Splash page requirements
> - Per-client bandwidth limits
> - Content filtering rules
> - Traffic shaping rules

> ✅ This means your 15 devices will pass through the default-deny rule you create in Step 4 without any impact. Their traffic flows normally regardless of the deny rule below them.

---

### Step 4 — Create the Default Deny Firewall Rule

This is the rule that blocks all unauthorized devices. It must be configured **after** Allow Listing your 15 devices in Step 3 — if you reverse the order, you'll disrupt authorized users while you work.

**Dashboard path:** `Security & SD-WAN → Configure → Firewall → L3 Outbound Rules`

Add a new rule at the **top** of the rule list:

| Field           | Value                                                           |
| --------------- | --------------------------------------------------------------- |
| **Policy**      | `Deny`                                                          |
| **Protocol**    | `Any`                                                           |
| **Source**      | `172.20.10.0/24` *(your employee SSID subnet)*                  |
| **Destination** | `Any`                                                           |
| **Port**        | `Any`                                                           |
| **Comment**     | `Block unauthorized devices on rv-staff — Allow List overrides` |

Click **Save**.

**Resulting rule table:**

| #   | Policy   | Protocol | Source           | Destination | Port | Comment                                |
| --- | -------- | -------- | ---------------- | ----------- | ---- | -------------------------------------- |
| 1   | **Deny** | Any      | `172.20.10.0/24` | Any         | Any  | Block unauthorized devices on rv-staff |
| 2   | Allow    | Any      | Any              | Any         | Any  | Default rule                           |

> ⚠️ **Protocol must be `Any`, not `TCP`.** A TCP-only deny still allows ICMP (ping) and UDP — unauthorized devices could still reach internal hosts via those protocols.

> ⚠️ **Rule order matters.** Meraki evaluates rules top-down, first match wins. The deny rule must be above the default allow rule. Allow Listed devices bypass the rule table entirely — rule order does not affect them.

---

### Step 5 — Verify End-to-End

#### 5a — Test an Authorized Device

From one of your 15 Allow Listed devices:

```bash
# Should SUCCEED — internet access
ping 8.8.8.8

# Should SUCCEED — internal resource
ping 172.20.10.254

# Browser — should load
curl https://example.com
```

**Expected result:** Full network access, no disruption.

---

#### 5b — Test an Unauthorized Device

Connect a test device that is **not** on your Allow List to `rv-staff`:

```bash
# Should FAIL — blocked by deny rule
ping 8.8.8.8

# Should FAIL — blocked
ping 172.20.10.254

# Browser — should time out or show no connectivity
curl https://example.com
```

**Expected result:** Device receives a DHCP lease (IP assignment succeeds), but all traffic is blocked. The device shows as "connected" to Wi-Fi but has no usable network access.

> ✅ Receiving a DHCP lease is **expected and correct**. The block occurs at the firewall layer, not at association.

---

#### 5c — Dashboard Verification

`Network-wide → Monitor → Clients`

For each of your 15 authorized devices, confirm the **Policy** column shows **Allow listed**.

For any unauthorized device you tested, confirm the **Policy** column shows **Normal** (or blank) — confirming they are subject to the deny rule.

---

## 6. Creating and Managing the Allow List — Full Reference

> This section is a standalone reference for the Allow List feature. Everything here is done in **`Network-wide → Monitor → Clients`** unless otherwise noted.

---

### How to Create the Allow List From Scratch

This is the full sequence when starting with no Allow List entries.

**Phase 1 — Add each device to the Clients page**

> Skip this phase for any device that has already connected to the network — it will already appear in the Clients list. Go straight to Phase 2 for those.

1. Navigate to `Network-wide → Monitor → Clients`
2. Look for the **`Add client`** button in the **top right corner** of the page
3. Click **`Add client`**
4. Fill in the fields:

| Field           | What to Enter                                                                                   |
| --------------- | ----------------------------------------------------------------------------------------------- |
| **Name**        | A friendly label — e.g., `Blair-Laptop`, `Reception-iPad`                                       |
| **MAC address** | The MAC as seen in the Dashboard — see [Section 7](#7-mac-randomization--what-you-need-to-know) |
| **Policy**      | Leave as `Normal` — you will set this in Phase 2                                                |

5. Click **Save changes**
6. Repeat for every device on your authorized list

> 💡 The `Add client` button is easy to miss — it sits in the top right corner of the Clients table, not in a sidebar or settings menu. If you don't see it, scroll up to the top of the page.

---

**Phase 2 — Apply the Allow List policy to all authorized devices**

Once all devices are in the Clients list (either added manually or already present from prior connections):

1. Navigate to `Network-wide → Monitor → Clients`
2. Use the **search/filter bar** to locate your authorized devices — filter by SSID or search by name/MAC
3. **Check the box** to the left of each authorized device
4. Click the **`Policy`** dropdown button that appears at the top of the client list once boxes are checked
5. Select **`Allow listed`**
6. Click **Save**

> ✅ You can select and Allow List multiple devices in one action — no need to do them one at a time.

**Visual confirmation:**

After saving, the **Policy** column next to each of your authorized devices should display:

```
Allow listed
```

Any device not on your list will show `Normal` or blank — those devices will be subject to the deny rule.

---

### How to Add a Device That Has Never Connected

Use this when a new authorized device needs access before it has ever joined the network.

1. `Network-wide → Monitor → Clients → Add client` (top right)
2. Enter the friendly name and MAC address
3. Set **Policy** → `Allow listed` directly (no need to do Phase 2 separately)
4. Click **Save changes**

> 💡 The device is now pre-authorized. When it connects for the first time, it will already be Allow Listed and will pass the deny rule immediately.

---

### How to Add a Device That Has Already Connected

Use this when a device is already in the Clients list from a prior connection.

1. `Network-wide → Monitor → Clients`
2. Find the device by MAC address, name, or IP
3. **Check the box** next to the device
4. Click **`Policy`** at the top of the list
5. Select **`Allow listed`** → **Save**

Alternatively, click directly on the device name to open its detail page, then set the policy there.

---

### How to Remove a Device From the Allow List

Use this when an authorized device should no longer have access — lost device, departed employee, etc.

1. `Network-wide → Monitor → Clients`
2. Find the device
3. **Check the box** next to it
4. Click **`Policy`** → select **`Normal`** → **Save**

> ⚠️ Setting back to `Normal` means the device is now subject to the default-deny rule and loses all network access — same as any unauthorized device. This takes effect immediately.

---

### How to View All Currently Allow Listed Devices

There is no dedicated "Allow List" page in the Dashboard. To see all currently Allow Listed devices:

1. `Network-wide → Monitor → Clients`
2. Set the time window to **Last 30 days** (or longer) to capture all devices, not just currently connected ones
3. Look for the **Policy** column — sort or filter by `Allow listed`

> 💡 If the Policy column is not visible, click the **column selector** (gear icon or column toggle) on the Clients page and enable it.

---

## 7. MAC Randomization — What You Need to Know

Modern mobile devices (iOS 14+, Android 10+) randomize their MAC address per SSID by default. Understanding this behavior is critical to building a reliable Allow List.

### The Key Fact: Randomized MACs Are Stable Per SSID

| Behavior                              | What Actually Happens                                     |
| ------------------------------------- | --------------------------------------------------------- |
| Device connects to `rv-staff`         | Presents a randomized MAC (not the physical hardware MAC) |
| Device disconnects and reconnects     | **Same randomized MAC** — stable                          |
| Device "forgets" the SSID and rejoins | **Same randomized MAC** — still stable                    |
| Device connects to a *different* SSID | Different randomized MAC for that SSID                    |

> ✅ **This means MAC filtering is workable for mobile devices** — as long as you collect the MAC from the **Meraki Dashboard** (what Meraki sees) rather than from the device's hardware MAC.

### The One Exception

On iOS, if a user manually toggles **Settings → Wi-Fi → [SSID] → Private Wi-Fi Address** off and back on, a new randomized MAC is generated for that SSID. That device would need to be re-added to your Allow List.

This is an edge case, not a routine occurrence. Users are unlikely to do this unless explicitly told to.

### How to Collect the Right MAC

```
✅ Correct source:  Meraki Dashboard → Network-wide → Clients
                   (shows the MAC Meraki actually sees)

❌ Wrong source:   iOS Settings → General → About → Wi-Fi Address
                   (shows physical hardware MAC — may not match)

❌ Wrong source:   Android Settings → About Phone → Status → MAC Address
                   (same problem — physical MAC, not randomized)
```

---

## 8. Limitations

> These are confirmed platform limitations. Understand them before deploying.

---

### ⚠️ Devices Still Associate and Get DHCP Leases

Unauthorized devices **will** connect to the SSID and receive an IP address. The block is at the *traffic* layer, not the *association* layer. To block at association, RADIUS (MAB) is required.

**Operational impact:**

- Your DHCP pool will be consumed by unauthorized devices
- If your DHCP pool is small relative to 300 devices, consider shortening the lease time to reclaim IPs faster
- Devices will show as "connected" in the Dashboard even though they can't do anything

---

### ⚠️ Allow List Bypasses ALL Firewall Rules

Allow Listed devices are exempt from every firewall rule — including any VLAN isolation rules you have in place for other purposes. If your 15 authorized devices should still be subject to some restrictions (e.g., cannot reach a management VLAN), the Allow List approach removes those restrictions.

**Mitigation:** If you need granular control, use a **Group Policy** with specific allow/deny rules instead of the built-in Allow List. Assign the Group Policy to your 15 devices and use the default-deny rule for everyone else.

---

### ⚠️ 3,000 Client Limit

Meraki supports a maximum of 3,000 manually Allow Listed clients per network. Not a concern for 15 devices, but worth knowing.

---

### ⚠️ MAC Spoofing

Any technically capable unauthorized user who observes a legitimate device's MAC address can spoof it and bypass the Allow List. This approach is access control, not security. It stops casual/accidental unauthorized use — it does not stop a determined adversary.

**Long-term fix:** Change the PSK and implement WPA2-Enterprise (RADIUS) for proper per-user authentication.

---

## 9. Troubleshooting

### Authorized Device Has No Network Access After Allow Listing

> **Symptom:** A device you Allow Listed cannot reach the internet or internal resources.

| Check                                     | How                                                            |
| ----------------------------------------- | -------------------------------------------------------------- |
| Confirm Policy shows "Allow listed"       | `Network-wide → Clients → find device → Policy column`         |
| Confirm correct MAC was Allow Listed      | Compare MAC in Dashboard to MAC you recorded                   |
| Check for MAC randomization mismatch      | Device may be showing a different randomized MAC than expected |
| Confirm the device is on the correct SSID | Device may have connected to a different SSID                  |

---

### Unauthorized Device Can Still Reach the Internet

> **Symptom:** A device that is NOT Allow Listed is not being blocked.

| Check                                                 | How                                                                                         |
| ----------------------------------------------------- | ------------------------------------------------------------------------------------------- |
| Verify deny rule Protocol is `Any` not `TCP`          | `Firewall → L3 Outbound Rules → inspect rule`                                               |
| Verify deny rule is above the default Allow rule      | Rule order — deny must be #1                                                                |
| Verify the device's IP is in the blocked subnet       | `Network-wide → Clients → check IP` — if it's on a different subnet, the rule doesn't apply |
| Check if the device has a Group Policy that overrides | `Network-wide → Clients → device → Policy`                                                  |

---

### Device Shows Connected But No IP (DHCP Failure)

> **Symptom:** Device connects to SSID but cannot get a DHCP lease.

This is unrelated to the Allow List / deny rule configuration. Check:

- DHCP pool exhaustion — pool may be full from 300 devices
- DHCP server configuration on the MX

---

### Allow Listed Device Lost Access After Working Fine

> **Symptom:** A previously working Allow Listed device suddenly has no access.

Most likely cause on a mobile device: the user toggled Private Wi-Fi Address in iOS settings, generating a new randomized MAC.

**Fix:**

1. Have the device reconnect to `rv-staff`
2. Find the new MAC in `Network-wide → Clients`
3. Add it to the Allow List
4. Optionally remove the old MAC entry to keep the list clean

---

## 10. Lessons Learned

> Hard-won knowledge. Read before starting.

---

### 🔴 Lesson 1 — Build the Allow List Before Applying the Deny Rule

Always add your legitimate devices to the Allow List *before* creating the default-deny firewall rule. If you create the deny rule first, you will lock out your authorized users while you scramble to add them.

**Correct order:**

```
1. Collect MACs (Step 1)
2. Add to Allow List (Steps 2–3)
3. Verify Allow List is applied
4. THEN create the deny rule (Step 4)
```

---

### 🔴 Lesson 2 — Collect MACs from the Dashboard, Not the Device

The MAC address visible in iOS or Android settings is the physical hardware MAC. For devices using MAC randomization, this will **not** match what Meraki sees. Allow Listing the hardware MAC does nothing if Meraki is seeing the randomized MAC.

**Always collect from:** `Network-wide → Monitor → Clients`

---

### 🟡 Lesson 3 — Unauthorized Devices Will Still Get IPs

Do not be alarmed when you see 285 unauthorized devices with IP addresses after deploying this. That is expected. They have addresses and no network. Monitor DHCP pool utilization and adjust lease time if needed.

---

### 🟡 Lesson 4 — This Is a Temporary Measure

~~MAC-based Allow Lists are a permanent access control strategy.~~

**Reality:** They are a stopgap. MAC addresses can be spoofed. The correct long-term solution is changing the PSK and moving to WPA2-Enterprise. Use this approach to buy time, not to close the book on the problem.

---

## 11. Checklist

### Pre-Work Checklist

- [ ] Identify all authorized devices — minimum information: friendly name + current SSID connection
- [ ] Collect MAC addresses from `Network-wide → Monitor → Clients` — **not** from devices directly
- [ ] Document each MAC with a friendly name in a spreadsheet before touching the Dashboard
- [ ] Confirm DHCP pool range and current utilization — note if exhaustion is a risk
- [ ] Confirm no existing Group Policies conflict with Allow List behavior
- [ ] Schedule a maintenance window if possible — brief disruption risk if steps are done out of order

---

### Configuration Checklist

- [ ] All 15 authorized MACs added via `Network-wide → Clients → Add client`
- [ ] All 15 authorized devices confirmed visible in Clients list
- [ ] All 15 devices selected and Policy set to **Allow listed**
- [ ] Confirmed each device shows **Allow listed** in the Policy column
- [ ] Deny rule created: Protocol = **Any**, Source = employee subnet, Destination = **Any**
- [ ] Deny rule positioned **above** the default allow rule
- [ ] Rule saved

---

### Verification Checklist

- [ ] Authorized device — `ping 8.8.8.8` → **succeeds**
- [ ] Authorized device — internal resource ping → **succeeds**
- [ ] Authorized device — browser test → **succeeds**
- [ ] Unauthorized test device — `ping 8.8.8.8` → **fails**
- [ ] Unauthorized test device — browser test → **times out**
- [ ] Unauthorized test device — receives DHCP lease → **yes** (expected)
- [ ] Dashboard Clients page — authorized devices show **Allow listed**
- [ ] Dashboard Clients page — unauthorized test device shows **Normal**

---

### Ongoing Maintenance Checklist

- [ ] New authorized device added → collect MAC from Dashboard → add to Allow List
- [ ] User reports sudden loss of access → check for MAC randomization change → re-add new MAC
- [ ] Monitor DHCP pool utilization weekly while 300 devices remain connected
- [ ] Plan and execute PSK rotation as long-term resolution
- [ ] After PSK rotation — re-evaluate whether Allow List is still needed

---

## 📎 Quick Reference

```
Meraki Dashboard Paths Used in This Guide
─────────────────────────────────────────
Collect MACs:       Network-wide → Monitor → Clients
Add client:         Network-wide → Monitor → Clients → Add client (top right)
Apply Allow List:   Network-wide → Monitor → Clients → check boxes → Policy → Allow listed
Firewall rule:      Security & SD-WAN → Configure → Firewall → L3 Outbound Rules
```

---

> **Document Version:** 1.0
> **Platform:** Cisco Meraki MX + MR
> **Example Organization:** Riverside Valley Municipal Authority *(fictional)*
> *All subnets, MACs, and device names in this guide are examples only.*
