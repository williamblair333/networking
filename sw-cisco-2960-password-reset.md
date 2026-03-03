# 🛠️ Cisco Catalyst 2960: Overview & Password Recovery Guide

**Author:** Leo Peng
**Logic Level:** Scholar/Candid Peer

---

## 📌 Part 1: Series Overview

The **Cisco Catalyst 2960** is a staple Layer 2/3 fixed-configuration access switch.

### 🚀 Key Performance Metrics

* **Forwarding Bandwidth:** Up to 108 Gbps.
* **Switching Bandwidth:** Up to 216 Gbps (Full-Duplex).
* **Stacking:** FlexStack / FlexStack-Plus (Model dependent).

### 📊 Sub-Series Comparison

| Model         | Layer | Stacking | PoE/PoE+ | Context               |
|:------------- |:----- |:-------- |:-------- |:--------------------- |
| **2960X**     | L2/L3 | Yes      | Yes      | Enterprise / Scalable |
| **2960S**     | L2    | Yes      | Yes      | Campus / Branch       |
| **2960-Plus** | L2    | No       | Yes      | Standard Office       |
| **2960XR**    | L3    | Yes      | Yes      | Advanced Edge         |
| **2960L**     | L2    | No       | Yes      | Entry SMB             |

---

## 🔑 Part 2: Password Recovery (Step-by-Step)

> [!IMPORTANT]
> This method renames the config file to bypass password checks while preserving your VLANs and interface settings.

### 1️⃣ Physical & Serial Setup

Connect to the Console port. Use `vim` to log your session output if needed.

* **Baud:** 9600
* **Data bits:** 8
* **Parity:** None
* **Stop bits:** 1
* **Flow Control:** Xon/Xoff

### 2️⃣ Interrupting the Bootloader

1. **Power Off** the switch.
2. Press and **Hold** the `Mode` button.
3. **Power On**; release `Mode` when the **SYST LED** turns solid green.
4. You are now at the `switch:` prompt.

### 3️⃣ File System Manipulation

```bash
# Initialize flash drivers
switch: flash_init

# View files
switch: dir flash:

# Hide the config from the boot process
switch: rename flash:config.text flash:config.text.old

# Manual Boot
switch: boot
```

### 4️⃣ Configuration Restoration

The switch will ask to enter the initial configuration dialog. **Type `no`**.

```bash
Switch> enable

# Bring the old config back
Switch# rename flash:config.text.old flash:config.text

# Load the file into Running-Config (RAM)
Switch# copy flash:config.text system:running-config

# Enter Global Config to overwrite passwords
Switch# configure terminal
Switch(config)# enable secret <NEW_SECRET>
Switch(config)# line vty 0 15
Switch(config-line)# password <NEW_VTY_PASS>
Switch(config-line)# login
Switch(config-line)# line con 0
Switch(config-line)# password <NEW_CON_PASS>
Switch(config-line)# end

# Commit to NVRAM
Switch# write memory
```

---

## ⚠️ Part 3: Troubleshooting & Warnings

- [x] **Flash Error:** If `dir flash:` fails, repeat `flash_init`.
- [x] **LED Amber:** If the LED stays amber, the hardware may have a POST failure.
- [x] **VLANs:** If using a separate `vlan.dat`, it remains unaffected by this process.

---

## 🛡️ Part 4: Best Practices

1. **Backup:** Always keep an external copy of `config.text`.
2. **Encryption:** Use `service password-encryption` and `enable secret`.
3. **Physical:** Lock the rack; anyone with physical access to the `Mode` button can bypass your security.

---

## 🏁 Part 5: Conclusion

This procedure ensures you regain access without the nuclear option of a factory reset. For deeper technical specs, refer to the [Official Cisco 2960 Documentation](https://www.cisco.com).
