# Physical Infrastructure Preparation for Monitoring

**Date:** December 21, 2025
**Author:** Systems Administrator
**Scope:** Configuration of Proxmox VE, HPE iLO 4, Supermicro BMC, and Ceph for "Least Privilege" metric scraping.

## 1. Overview & Network Inventory

This document details the "Off-Cluster" prerequisites required to allow a Kubernetes-based monitoring stack (Prometheus) to scrape metrics from the physical layer.

**Target Hardware IPs:**

* **Node 1 (HPE):** `10.10.10.111`
* **Node 2 (HPE):** `10.10.10.112`
* **Node 3 (HPE):** `10.10.10.113`
* **Node 4 (Supermicro):** `10.10.10.114`

**Design Principles:**

* **Least Privilege:** All monitoring accounts are Read-Only (User/Audit level).
* **Agentless:** BMC metrics are pulled via IPMI over LAN; Proxmox/Ceph metrics via API/HTTP.

---

## 2. Proxmox VE User & API Token

*Perform these steps via SSH on **one** Proxmox node.*

### 2.1 Create Custom Role

We create a custom role because the default `PVEAuditor` grants too much access, and creating a role starting with `PVE` is forbidden.

```bash
# Create Role 'Monitor-Role' with minimum required Audit privileges
# Sys.Audit: CPU/RAM/Node status
# Datastore.Audit: Storage usage
# VM.Audit: VM/LXC config and status (No Console access)
pveum role add Monitor-Role -privs "Sys.Audit Datastore.Audit VM.Audit"

```

### 2.2 Create User & Bind Permissions

```bash
# Create the user without a password (API Token authentication only)
pveum user add monitor@pve

# Apply the Role to the root path (/) so it propagates to all nodes/storage
pveum acl modify / -user monitor@pve -role Monitor-Role

```

### 2.3 Generate API Token

**⚠️ CRITICAL:** The `value` (Secret Key) is displayed only once. Save it immediately.

```bash
# Generate token 'monitoring-token'
pveum user token add monitor@pve monitoring-token --privsep 0

```

### 2.4 Verification

Run this command to confirm the user has permissions on `/`.

```bash
pveum user permissions monitor@pve

```

*Expected Output: A table showing `/` mapped to `Monitor-Role`.*

---

## 3. HPE iLO 4 Configuration (Nodes 1-3)

*Perform these steps on the iLO Web Interface for `10.10.10.111`, `.112`, and `.113`.*

### 3.1 Create Local User

1. Navigate to **Administration**  **User Administration**.
2. Click **New User**.
3. **User Name:** `mon_user`
4. **Login Name:** `mon_user`
5. **Password:** `<SECURE_PASSWORD>`
6. **Permissions:**
* **Login:** [Checked]
* **Remote Console:** [Unchecked] ❌
* **Virtual Power and Reset:** [Unchecked] ❌
* **Virtual Media:** [Unchecked] ❌
* **Configure iLO Settings:** [Unchecked] ❌
* **Administer User Accounts:** [Unchecked] ❌
* *Note: Unchecking "Remote Console" forces the user into strict IPMI "User" mode, which prevents cipher negotiation errors.*



### 3.2 Enable IPMI over LAN

1. Navigate to **Administration**  **Access Settings**  **Service**.
2. **IPMI/DCMI over LAN:** Enabled.
3. **Port:** 623.

### 3.3 Verification (Run from Linux Host)

**Note:** You must use `-L USER` to request Read-Only access and `-C 3` to force Cipher Suite 3 (required for iLO 4 compatibility).

```bash
# Test Chassis Status (replace IP for 112/113)
ipmitool -I lanplus -H 10.10.10.111 -U mon_user -P <PASSWORD> -L USER -C 3 chassis status

# Test Sensor Reading
ipmitool -I lanplus -H 10.10.10.111 -U mon_user -P <PASSWORD> -L USER -C 3 sdr elist

```

*Success Condition: Returns `System Power: on` and a list of sensor data.*

---

## 4. Supermicro BMC Configuration (Node 4)

*Perform these steps via SSH on the Supermicro Linux host (`10.10.10.114`). This avoids a BIOS reboot.*

### 4.1 Configure User in Slot 4

```bash
# 1. Set Username
ipmitool user set name 4 mon_user

# 2. Set Password
ipmitool user set password 4 <SECURE_PASSWORD>

# 3. Set Privilege to USER (Level 2)
# Level 2 = Read Only
ipmitool user priv 4 2 1

# 4. Enable User
ipmitool user enable 4

```

### 4.2 Enable Network Access & Apply Changes

We configure both Channel 1 (Standard LAN) and Channel 8 (Dedicated LAN) to ensure connectivity regardless of physical cabling.

```bash
# Enable access on Channel 1
ipmitool channel setaccess 1 4 callin=on ipmi=on link=on privilege=2

# Enable access on Channel 8 (Ignore "Invalid data field" errors if channel doesn't exist)
ipmitool channel setaccess 8 4 callin=on ipmi=on link=on privilege=2

# ⚠️ CRITICAL: Restart BMC to apply changes (Does not reboot OS)
ipmitool mc reset cold

```

*Wait 60 seconds after the reset command before verifying.*

### 4.3 Verification

**Note:** You must use `-L USER` to prevent `ipmitool` from attempting an Administrator login, which causes "Unable to establish session" errors.

```bash
# Verify from a remote host (e.g., Node 1)
ipmitool -I lanplus -H 10.10.10.114 -U mon_user -P <PASSWORD> -L USER sensor

```

---

## 5. Ceph Metrics Exposure

*Perform on any Proxmox node running Ceph services.*

### 5.1 Enable Module

```bash
ceph mgr module enable prometheus

```

### 5.2 Verification

By default, the module listens on port **9283**.

```bash
curl -s localhost:9283/metrics | head -n 5

```

*Success Condition: Output starts with Prometheus `# HELP` and `# TYPE` headers.*

---

## 6. Troubleshooting Guide

| Issue | Cause | Solution |
| --- | --- | --- |
| **HPE:** `Unable to Get Channel Cipher Suites` | iLO 4 encryption negotiation mismatch. | Add `-C 3` to the `ipmitool` command. |
| **Supermicro:** `Set Session Privilege Level to ADMINISTRATOR failed` | Client requested Admin rights for a User account. | Add `-L USER` to the `ipmitool` command. |
| **Supermicro:** `Unable to establish IPMI v2 session` | BMC has not refreshed user table. | Run `ipmitool mc reset cold` and wait 60s. |
| **Proxmox:** `400 Parameter verification failed` | Role name started with `PVE`. | Delete role and create one named `Monitor-Role`. |