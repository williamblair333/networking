# 📁 Cisco Catalyst 2960: TFTP Operations Guide

**Focus:** Remote Configuration & IOS Management
**Target Server:** `192.168.1.22`

---

## 📌 Part 1: Cisco Catalyst 2960 Series Overview

The **Cisco Catalyst 2960 Series** switches are reliable Layer 2 fixed-configuration switches. For management tasks like firmware upgrades or configuration backups, they utilize Trivial File Transfer Protocol (TFTP) over UDP port 69.

### 📊 Sub-Series Comparison (TFTP/Management Context)

| Model           | Management Port     | Stacking Support | Typical Use       |
|:--------------- |:------------------- |:---------------- |:----------------- |
| **Cisco 2960X** | Dedicated Mgmt Port | FlexStack-Plus   | Enterprise access |
| **Cisco 2960S** | Vlan1 Interface     | FlexStack        | Campus / branch   |
| **Cisco 2960L** | Vlan1 Interface     | No               | Entry-level SMB   |

---

## 🔑 Part 2: Preparing for TFTP Connectivity

Before initiating a transfer to `192.168.1.22`, the switch must have a Layer 3 path to the server.

### Step 1: Interface Configuration

If you are not using a dedicated management port, you must assign an IP to a SVI (Switched Virtual Interface), typically **VLAN 1**.



```bash
Switch# configure terminal
Switch(config)# interface vlan 1
Switch(config-if)# ip address 192.168.1.5 255.255.255.0
Switch(config-if)# no shutdown
Switch(config-if)# end
```

### Step 2: Connectivity Verification

Ensure the TFTP server is reachable.

```bash
Switch# ping 192.168.1.22
Type escape sequence to abort.
Sending 5, 100-byte ICMP Echos to 192.168.1.22, timeout is 2 seconds:
!!!!!
Success rate is 100 percent (5/5)
```

---

## 📥 Part 3: TFTP Operations (Backup & Restore)

### 📤 1. Copy FROM Switch TO TFTP (Backup)

Use this to archive your configuration or the current IOS image to your server at `192.168.1.22`.

```bash
# Backup Running Config
Switch# copy running-config tftp:
Address or name of remote host []? 192.168.1.22
Destination filename [Switch-confg]? backup-config.txt

# Backup Startup Config
Switch# copy startup-config tftp:
Address or name of remote host []? 192.168.1.22
Destination filename [Switch-confg]? startup-config-backup.txt

# Backup IOS Image
# Note: Use 'dir flash:' to find the exact .bin filename
Switch# copy flash:c2960-lanbasek9-mz.150-2.SE4.bin tftp:
Address or name of remote host []? 192.168.1.22
Destination filename [c2960-lanbasek9-mz.150-2.SE4.bin]? 2960-ios-archive.bin
```

### 📥 2. Copy FROM TFTP TO Switch (Restore/Upgrade)

Use this to push a configuration file or upgrade the firmware.

```bash
# Restore Configuration to Startup (NVRAM)
Switch# copy tftp: startup-config
Address or name of remote host []? 192.168.1.22
Source filename []? backup-config.txt
Destination filename [startup-config]? 

# Copy new IOS to Flash (Upgrade)
Switch# copy tftp: flash:
Address or name of remote host []? 192.168.1.22
Source filename []? c2960-new-firmware.bin
Destination filename [c2960-new-firmware.bin]? 
```

---

## ⚠️ Part 4: Common Issues During TFTP

- [x] **TFTP Timeout:** Ensure the TFTP server software is running on `192.168.1.22` and UDP Port 69 is allowed through the Windows/Linux firewall.
- [x] **Permission Denied:** Ensure the TFTP root directory on the server has "Write" permissions enabled for the backup process.
- [x] **Incomplete Transfer:** For large IOS files, ensure the connection is stable. TFTP does not support "Resume" if interrupted.

---

## 🛡️ Part 5: Best Practices

* **Standardization:** Name your backup files with dates (e.g., `2960-core-03-03-26.cfg`).
* **Security:** TFTP is unencrypted. Do not perform transfers over public or untrusted networks. Use SCP for secure transfers if your IOS version supports it.
* **Verification:** Always verify the MD5 hash of an IOS file after a TFTP transfer to ensure it wasn't corrupted: `verify /md5 flash:filename.bin`.

---

## 🏁 Part 6: Conclusion

Utilizing a TFTP server like `192.168.1.22` is the most efficient way to manage configurations across the Cisco Catalyst 2960 series. Regularly backing up your `running-config` ensures rapid recovery in the event of hardware failure.
