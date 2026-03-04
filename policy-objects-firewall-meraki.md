# 🔒 Meraki Policy Objects in Firewall Rules
## Scalable Guest & VLAN Isolation Without Rule Sprawl

> **Document Type:** How-To + Reference Guide  
> **Platform:** Cisco Meraki MX Security Appliance  
> **Applies To:** Any organization running guest, IoT, or segmented VLANs  
> **Complexity:** Intermediate  

---

## 📋 Table of Contents

1. [Why This Matters](#1-why-this-matters)
2. [How It Works](#2-how-it-works)
3. [Prerequisites](#3-prerequisites)
4. [Architecture Overview](#4-architecture-overview)
5. [Step-by-Step Configuration](#5-step-by-step-configuration)
   - [Step 1 — Create Individual Network Objects](#step-1--create-individual-network-objects)
   - [Step 2 — Create Object Groups](#step-2--create-object-groups)
   - [Step 3 — Wait for Dashboard Propagation](#step-3--wait-for-dashboard-propagation)
   - [Step 4 — Reference Objects in Firewall Rules](#step-4--reference-objects-in-firewall-rules)
   - [Step 5 — Verify and Test](#step-5--verify-and-test)
6. [Maintaining Objects Over Time](#6-maintaining-objects-over-time)
7. [Reference Tables](#7-reference-tables)
8. [Known Limitations](#8-known-limitations)
9. [Troubleshooting](#9-troubleshooting)
10. [Lessons Learned](#10-lessons-learned)
11. [Checklist](#11-checklist)

---

## 1. Why This Matters

### The Problem: Rule Sprawl

Without policy objects, a typical guest/employee isolation setup looks like this:

```
Rule 1:  Deny  Any  192.168.4.0/24  →  10.0.1.10/32   (Guest → Protected Host 1)
Rule 2:  Deny  Any  192.168.4.0/24  →  10.0.1.11/32   (Guest → Protected Host 2)
Rule 3:  Deny  Any  192.168.4.0/24  →  10.0.1.12/32    (Guest → Protected Host 3)
Rule 4:  Deny  Any  192.168.4.0/24  →  10.0.1.0/24     (Guest → Employee Subnet)
Rule 5:  Deny  Any  192.168.2.0/24  →  10.0.1.10/32   (Training → Protected Host 1)
Rule 6:  Deny  Any  192.168.2.0/24  →  10.0.1.11/32   (Training → Protected Host 2)
...
```

Every new VLAN multiplies the rule count. With 3 untrusted VLANs and 4 protected destinations, that's already **12 rules**. Each one must be individually created, maintained, and audited.

### The Solution: Policy Objects

```
Rule 1:  Deny  Any  [Untrusted-Subnets]  →  [Trusted-Subnets]
Rule 2:  Allow  Any  Any  →  Any
```

**Two rules. Forever.** Add a new guest VLAN? Update the object group. The rule doesn't change.

---

## 2. How It Works

Meraki Policy Objects are **reusable named references** to IP addresses, subnets, or FQDNs. They live at the **organization level** and can be referenced by name in firewall rules across any network in the org.

```
Organization → Policy Objects
       │
       ├── Network Objects (individual IPs/CIDRs)
       │       ├── guest-vlan-1      → 192.168.4.0/24
       │       ├── guest-vlan-2      → 192.168.2.0/24
       │       └── trusted-subnet    → 10.0.1.0/24
       │
       └── Object Groups (collections of objects)
               ├── Untrusted-Subnets → [guest-vlan-1, guest-vlan-2]
               └── Trusted-Subnets   → [trusted-subnet]
```

Firewall rules reference the **group**, not the individual IPs. When the group changes, all rules referencing it update automatically — no rule edits required.

---

## 3. Prerequisites

| Requirement | Details |
|-------------|---------|
| **Dashboard access** | Organization Administrator |
| **MX firmware** | Supports Policy Objects (verify at `Security & SD-WAN → Appliance Status → Firmware`) |
| **Policy Objects feature** | Enabled at `Organization → Policy Objects` — if the menu doesn't exist, the feature may need to be enabled under Early Access |
| **Existing VLANs** | VLANs to be isolated should already be configured |
| **Existing firewall rules** | Know what rules you're replacing before you start |

> ⚠️ **Policy Objects are an org-level feature.** Objects created here are available across all networks in your organization. Name them clearly — other admins will see them.

---

## 4. Architecture Overview

### Logical Model

```
┌─────────────────────────────────────────────────────────────┐
│                    ORGANIZATION LEVEL                        │
│                                                             │
│   Policy Objects                                            │
│   ┌─────────────────────┐  ┌──────────────────────────┐    │
│   │  Untrusted-Subnets  │  │    Trusted-Subnets        │    │
│   │  (Object Group)     │  │    (Object Group)         │    │
│   │                     │  │                           │    │
│   │  ● guest-vlan       │  │  ● employee-subnet        │    │
│   │    192.168.4.0/24     │  │    10.0.1.0/24           │    │
│   │  ● training-vlan    │  │                           │    │
│   │    192.168.2.0/24     │  │                           │    │
│   │  ● iot-vlan (future)│  │                           │    │
│   │    192.168.6.0/24     │  │                           │    │
│   └─────────┬───────────┘  └──────────┬────────────────┘    │
│             │                         │                     │
└─────────────┼─────────────────────────┼─────────────────────┘
              │                         │
              ▼                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    NETWORK LEVEL (MX)                        │
│                                                             │
│   L3 Outbound Firewall Rules                                │
│   ┌──────────────────────────────────────────────────────┐  │
│   │  #1  Deny  Any  [Untrusted-Subnets] → [Trusted]      │  │
│   │  #2  Allow Any  Any               → Any              │  │
│   └──────────────────────────────────────────────────────┘  │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Before vs. After

| | Before (Manual Rules) | After (Policy Objects) |
|---|---|---|
| **Rules needed** | 1 per source/destination pair | 1 per relationship type |
| **Adding a new guest VLAN** | Add N new rules | Add 1 object to a group |
| **Removing a VLAN** | Hunt and delete matching rules | Remove 1 object from a group |
| **Audit clarity** | Read every IP on every rule | Read group names |
| **Risk of missed rule** | High | Low |

---

## 5. Step-by-Step Configuration

---

### Step 1 — Create Individual Network Objects

**Dashboard path:** `Organization → Policy Objects → All Objects tab → Add new`

Each subnet needs its own named object before it can be added to a group.

> 💡 Create **all** objects you'll need before creating groups — you can't add a subnet to a group that doesn't exist as an object yet.

**For each untrusted subnet, create an object:**

| Example Name | Type | Value |
|-------------|------|-------|
| `guest-wifi` | CIDR | `192.168.4.0/24` *(your guest subnet)* |
| `training-wifi` | CIDR | `192.168.2.0/24` *(your training subnet)* |
| `iot-devices` | CIDR | `192.168.6.0/24` *(your IoT subnet, if applicable)* |

**For each trusted subnet, create an object:**

| Example Name | Type | Value |
|-------------|------|-------|
| `employee-primary` | CIDR | `10.0.1.0/24` *(your employee subnet)* |

**How to create each object:**

1. Click **Add new**
2. Enter a descriptive name *(no special characters: `#`, `/`, `$`, `@`, `%`, `!`)*
3. In the **Contains** field, type your CIDR (e.g., `192.168.4.0/24`)
4. **Click the auto-suggested option** that appears below the field — do not press Enter or Tab
5. Click **Create**

> ⚠️ **Critical:** You must **click the suggestion** that drops down — typing a value and pressing Enter will not register it. This is the single most common input error in the Policy Objects UI.

---

### Step 2 — Create Object Groups

**Dashboard path:** `Organization → Policy Objects → Groups tab → Add new`

Groups are collections of the objects you just created. You'll create one group for untrusted sources and one for trusted destinations.

#### Group 1 — Untrusted Sources

| Field | Value |
|-------|-------|
| Group name | `Untrusted-Subnets` *(or your naming convention)* |
| Contains | Add each untrusted object by typing its name and clicking the suggestion |

#### Group 2 — Trusted Destinations

| Field | Value |
|-------|-------|
| Group name | `Trusted-Subnets` |
| Contains | Add each trusted object by typing its name and clicking the suggestion |

> 💡 **Naming convention tip:** Use a consistent prefix that makes groups easy to find later — e.g., `FW-Untrusted-*` and `FW-Trusted-*`. You'll be searching for these by name in the firewall rule UI.

---

### Step 3 — Wait for Dashboard Propagation

> ⚠️ **This step is not optional. Do not skip it.**

After creating policy objects and groups, there is a **propagation delay** before the objects become searchable in firewall rule fields. The delay is variable and has been observed to take **up to one hour** in production environments.

**What happens if you don't wait:**

- You type the group name in the firewall rule source/destination field
- No suggestion appears in the dropdown
- The field rejects your input with: *"Invalid input. Must be one of the following types: Any, IPv4 CIDR, Network object, or Object group"*
- The objects appear to not exist, even though they do

**What to do:**

```
1. Create your objects and groups
2. Leave the dashboard
3. Return after ~30–60 minutes
4. Proceed to Step 4
```

> 💡 You can verify readiness by going to `Organization → Policy Objects → All Objects` and confirming your objects show the correct CIDR values in the **Contains** column. Visibility in the list does not mean they are searchable in firewall rules yet — you must wait for full propagation regardless.

---

### Step 4 — Reference Objects in Firewall Rules

**Dashboard path:** `Security & SD-WAN → Configure → Firewall → L3 Outbound Rules`

> ⚠️ **Remove or replace your existing manual rules** once the object-based rule is confirmed working. Do not run both simultaneously or traffic will match the manual rules first (Meraki evaluates top-down, first match wins).

**Add a new deny rule:**

1. Click **Add a rule** at the top of the rule list
2. Set **Policy** → `Deny`
3. Set **Protocol** → `Any`
4. Click the **Source** field — ensure it is completely empty
5. Slowly type the first few characters of your untrusted group name (e.g., `Untr`)
6. **Wait** for the dropdown to appear — it may take 2–3 seconds
7. When you see your group name labeled as **Object group** — **click it**
8. Repeat for the **Destination** field with your trusted group name
9. Set **Dst port** → `Any`
10. Add a **Comment** describing the rule (e.g., `Block untrusted VLANs from employee resources`)
11. Ensure this rule sits **above** the default Allow rule
12. Click **Save**

**Resulting rule table:**

| # | Policy | Protocol | Source | Destination | Port | Comment |
|---|--------|----------|--------|-------------|------|---------|
| 1 | Deny | Any | `[Untrusted-Subnets]` | `[Trusted-Subnets]` | Any | Block untrusted VLANs from employee resources |
| 2 | Allow | Any | Any | Any | Any | Default rule |

---

### Step 5 — Verify and Test

#### 5a — Confirm Object Reference in Dashboard

After saving, the rule should display the **group name** in the source/destination columns — not a raw IP. If you see a raw IP, the object was not properly selected.

#### 5b — Functional Test

From a device on each untrusted VLAN, test the following:

```bash
# Should FAIL — blocked by deny rule
ping [trusted-host-IP]

# Should FAIL — blocked by deny rule
ping [any employee subnet IP]

# Should SUCCEED — internet access unaffected
ping 8.8.8.8

# Browser test — should load
curl https://example.com
```

#### 5c — Add a Second VLAN (Proof of Concept)

The real value of this approach is confirmed when you add a second subnet to the group **without touching the firewall rule**:

1. Go to `Organization → Policy Objects → Groups → Untrusted-Subnets → Edit`
2. Add a new object (e.g., `iot-devices`)
3. Save
4. Immediately test from a device on the IoT subnet — it should be blocked from trusted resources with **zero firewall rule changes**

---

## 6. Maintaining Objects Over Time

### Adding a New Untrusted VLAN

```
Organization → Policy Objects → Groups → [Untrusted-Subnets] → Edit
  └── Add new object for the new subnet CIDR
      └── Save
```

No firewall rule changes required.

---

### Adding a New Protected Host/Subnet

```
Organization → Policy Objects → Groups → [Trusted-Subnets] → Edit
  └── Add new object for the new CIDR
      └── Save
```

No firewall rule changes required.

---

### Removing a VLAN

```
Organization → Policy Objects → All Objects → [object name] → Delete
```

> ⚠️ You cannot delete an object that is currently referenced in an active group. Remove it from the group first, then delete the object.

---

### Auditing

`Organization → Policy Objects` provides a **Used in** column showing every firewall rule referencing each object or group. Use this before making changes to understand blast radius.

---

## 7. Reference Tables

### Policy Object Types

| Type | Format | Example | Supported in L3 Outbound? |
|------|--------|---------|--------------------------|
| IP Address | `/32` CIDR | `192.168.1.10/32` | ✅ Yes |
| IP Subnet | CIDR notation | `192.168.4.0/24` | ✅ Yes |
| FQDN | Domain name | `example.com` | ✅ Destination only |
| Wildcard FQDN | `*.domain.com` | `*.example.com` | ✅ Destination only |
| Object Group | Named collection | `Untrusted-Subnets` | ✅ Yes |

### Where Policy Objects Can Be Used

| Location | Objects Supported | Groups Supported | Notes |
|----------|-----------------|-----------------|-------|
| L3 Outbound Firewall Rules | ✅ | ✅ | Primary use case |
| Group Policies | ❌ | ❌ | **Not supported — hard platform limit** |
| Site-to-Site VPN Firewall | ❌ | ❌ | Not supported |
| Cellular Failover Firewall | ❌ | ❌ | Not supported |
| Traffic Shaping Rules | ✅ | ✅ | Supported |

### Common Naming Conventions

| Pattern | Example | Use Case |
|---------|---------|----------|
| Role-based | `Guest-Wifi`, `Employee-LAN` | Simple environments |
| Direction-based | `Untrusted-Sources`, `Trusted-Destinations` | Firewall-oriented |
| Zone-based | `Zone-DMZ`, `Zone-Internal` | Security zone model |
| Prefix-tagged | `FW-Block-Guest`, `FW-Allow-Corp` | Multi-admin environments |

---

## 8. Known Limitations

> These are confirmed platform limitations as of the document date. Verify against current Meraki release notes for your firmware version.

### ❌ Group Policies Do Not Support Policy Objects

This is a **hard platform limitation**, not a configuration error. The Group Policy firewall rule UI accepts only raw IPs and CIDRs — object group names are rejected with an input validation error. There is no workaround within the dashboard GUI.

*Workaround:* Use L3 Outbound Firewall Rules (which do support objects) rather than Group Policy firewall rules for subnet-based isolation.

---

### ❌ FQDN Objects Cannot Be Used as Source

FQDN and Wildcard FQDN objects are only valid in the **Destination** field of L3 outbound rules. Attempting to use them as a source will produce a validation error.

---

### ❌ IPv6 Not Supported

Policy Objects do not support IPv6 addresses. All objects must be IPv4 CIDRs or FQDNs.

---

### ⏱️ Propagation Delay Is Not Shown in the UI

The dashboard provides no indicator that a newly created object is "ready" for use in firewall rules. There is no loading indicator, no status field, and no notification. The only signal is whether the object appears in the firewall rule dropdown. Allow **up to one hour** after creation before attempting to reference objects in rules.

---

### ⚠️ Object Deletion Affects All Rules

Modifying or deleting an object immediately affects every firewall rule that references it — across all networks in the organization. There is no staging, preview, or confirmation step. Always check the **Used in** column before editing or deleting objects.

---

## 9. Troubleshooting

### Object Name Not Appearing in Firewall Rule Dropdown

> **Symptom:** You type the object or group name in the source/destination field and nothing appears, or the field shows an invalid input error.

**Cause 1 — Propagation delay**  
Objects were recently created and have not fully propagated to the firewall rule UI.  
**Fix:** Wait 30–60 minutes and try again.

**Cause 2 — Wrong field behavior**  
You pressed Enter or Tab after typing instead of clicking the dropdown suggestion.  
**Fix:** Clear the field completely, retype the name, and wait for the dropdown. Click — do not press any key.

**Cause 3 — Object not fully saved**  
The CIDR value in the object's Contains field was typed but not clicked from the suggestion.  
**Fix:** Go to `Organization → Policy Objects → All Objects`, find the object, and verify it shows the correct CIDR in the **Contains** column. If it shows nothing, edit and re-add the value by clicking the suggestion.

---

### Rule Saved with Raw IP Instead of Object Name

> **Symptom:** After saving, the firewall rule shows an IP address instead of the object/group name.

**Cause:** The object was not selected from the dropdown — a raw IP was manually typed and accepted.  
**Fix:** Edit the rule, clear the field, and re-enter by clicking the dropdown suggestion.

---

### Traffic Still Blocked After Removing VLAN from Group

> **Symptom:** A subnet was removed from the untrusted group but traffic from it is still being denied.

**Cause:** The MX may take a few minutes to push updated policy after a group change. Additionally, verify the subnet doesn't match another object still in the group (e.g., a broader /16 that encompasses your /24).  
**Fix:** Wait 2–3 minutes, retest. Check all objects in the group for overlapping ranges.

---

### Traffic Not Blocked After Adding VLAN to Group

> **Symptom:** A new subnet was added to the untrusted group but devices on it can still reach trusted hosts.

**Possible causes:**

1. Propagation delay — wait 2–3 minutes
2. The device is getting an IP from a different subnet than expected — verify the DHCP lease
3. There is a more specific allow rule earlier in the rule list that is matching first
4. The new VLAN's traffic is routing through a different path (e.g., directly to the MX gateway, bypassing the outbound rules)

---

## 10. Lessons Learned

> Hard-won knowledge from production implementation. Read before starting.

---

### 🔴 Lesson 1 — You Must Click the Suggestion, Not Press Enter

~~Type the CIDR into the Contains field and press Enter to add it.~~

**Reality:** The Contains field in Policy Objects — and the source/destination fields in firewall rules — require you to **click the auto-suggested option** that appears in the dropdown below the field. Pressing Enter, Tab, or any key does not register the value. The field will appear populated but the value is not saved.

This applies to **both** the object creation screen and the firewall rule input fields.

---

### 🔴 Lesson 2 — Propagation Delay Is Real and Unindicated

~~Once an object is saved, it is immediately available for use in firewall rules.~~

**Reality:** There is a significant propagation delay — observed up to **one hour** in production — between creating a policy object and being able to reference it in firewall rules. The dashboard provides no indication of this delay. If you try to use an object immediately after creating it, the firewall rule field will reject it as if it doesn't exist.

**Best practice:** Create all objects and groups in one session, then leave and return after an hour before configuring firewall rules.

---

### 🔴 Lesson 3 — Group Policies Cannot Use Policy Objects

~~Policy Objects can be used in Group Policy firewall rules for client-level enforcement.~~

**Reality:** This is a confirmed, hard platform limitation. Group Policy firewall rules only accept raw IPs and CIDRs. Use **L3 Outbound Firewall Rules** for object-based subnet isolation instead.

---

### 🟡 Lesson 4 — The "Invalid Input" Error Is Misleading

When the firewall rule field rejects a group name, the error reads:

> *"Invalid input. Must be one of the following types: Any, IPv4 CIDR, Network object, or Object group"*

This error suggests the field *does* support Object groups — and it does. The error does **not** mean your object group is wrong. It typically means either the object hasn't propagated yet, or you didn't click the suggestion. Don't delete and recreate objects in response to this error.

---

## 11. Checklist

### Setup Checklist

- [ ] Inventory all untrusted subnets to be blocked
- [ ] Inventory all trusted subnets/hosts to be protected
- [ ] Navigate to `Organization → Policy Objects → All Objects`
- [ ] Create one Network Object per untrusted subnet — verify CIDR shows in Contains column
- [ ] Create one Network Object per trusted subnet — verify CIDR shows in Contains column
- [ ] Navigate to `Organization → Policy Objects → Groups`
- [ ] Create `Untrusted-Subnets` group — add all untrusted objects
- [ ] Create `Trusted-Subnets` group — add all trusted objects
- [ ] **Wait 30–60 minutes before proceeding**
- [ ] Navigate to `Security & SD-WAN → Configure → Firewall → L3 Outbound Rules`
- [ ] Add deny rule: Protocol Any, Source = `[Untrusted-Subnets]`, Destination = `[Trusted-Subnets]`
- [ ] Confirm rule shows group name (not raw IP) after saving
- [ ] Place deny rule above default allow rule
- [ ] Remove or disable any superseded manual rules
- [ ] Save

### Verification Checklist

- [ ] Test device on untrusted VLAN — receives correct subnet IP
- [ ] `ping [trusted-host]` from untrusted device — **fails**
- [ ] `ping 8.8.8.8` from untrusted device — **succeeds**
- [ ] Browse internet from untrusted device — **succeeds**
- [ ] Add a second subnet to untrusted group
- [ ] Test from new subnet — **blocked without rule changes** ✅
- [ ] `Organization → Policy Objects → [group] → Used in` — confirms rule reference

### Ongoing Maintenance Checklist

- [ ] New untrusted VLAN → add object to `Untrusted-Subnets` group only
- [ ] New protected resource → add object to `Trusted-Subnets` group only
- [ ] Before deleting any object → check **Used in** column
- [ ] After any group change → retest affected VLANs within 5 minutes
- [ ] Periodic audit: `Organization → Policy Objects` — remove stale objects

---

> **Document Version:** 1.0  
> **Platform:** Cisco Meraki MX — L3 Outbound Firewall with Policy Objects  
> *Tested in production. Propagation delay confirmed. Group Policy limitation confirmed.*
