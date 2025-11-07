# Phase 1: Foundation Setup – Proxmox + TrueNAS

> **This guide walks you through the initial infrastructure setup for your homelab: Proxmox installation, storage foundation with TrueNAS, core hardening, and prep for next phases.**

---

## 📑 Table of Contents

- [✅ Pre-Phase Checklist](#-pre-phase-checklist)
- [🚦 Phase 1 Overview](#-phase-1-overview)
- [⚡ Step 1: Proxmox VE Installation](#-step-1-proxmox-ve-installation)
- [🔧 Step 2: Proxmox Initial Configuration](#-step-2-proxmox-initial-configuration)
- [💾 Step 3: TrueNAS VM Creation](#-step-3-truenas-vm-creation)
- [📁 NFS and Dataset Setup](#-nfs-and-dataset-setup)
- [🧪 Verification & Troubleshooting](#-verification--troubleshooting)
- [🔒 Step 6: Initial Security Hardening](#-step-6-initial-security-hardening)
- [🗂️ Backups & Snapshots](#-backups--snapshots)
- [➡️ Next Steps](#-next-steps)

---

## ✅ Pre-Phase Checklist

- [ ] HP EliteDesk 800 G5 powered on and accessible
- [ ] USB drive with Proxmox VE ISO prepared
- [ ] Network cable connected (DHCP available)
- [ ] Monitor and keyboard connected for initial setup
- [ ] **CRITICAL:** Backup any existing data – this will wipe the system

---

## 🚦 Phase 1 Overview

- **Goal:** Install Proxmox VE as hypervisor and bring up a TrueNAS VM for central ZFS storage
- **Estimated Time:** 2–3 hours
- **Risk Level:** High (system wipe on install)

---

## ⚡ Step 1: Proxmox VE Installation

### 1.1 Boot from USB

1. Insert prepared Proxmox VE USB installer
2. Boot (usually F9 for HP; check your hardware)
3. Select "Install Proxmox VE"

### 1.2 System Configuration

- Target Hard Disk: Select your RAID 1 array
- Set country/timezone, *strong* admin password, and valid email for alerts
- Use DHCP for management interface, note the assigned IP

> ✅ **Complete when** you can access the Proxmox web UI from LAN

---

## 🔧 Step 2: Proxmox Initial Configuration

### 2.1 Repository & Updates

```
# Remove enterprise repo
rm /etc/apt/sources.list.d/pve-enterprise.list

# Add the free no-subscription repo
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Update system
apt update && apt upgrade -y
```

### 2.2 Configure Static IP

- Edit the node → System → Network in the Proxmox web UI
- Assign a static IP within your 10.0.0.x subnet

> ✅ **Complete when** Proxmox is reachable via its static IP

---

## 💾 Step 3: TrueNAS VM Creation

### 3.1 Upload TrueNAS ISO

- Download [TrueNAS SCALE ISO](https://www.truenas.com/download-truenas-scale/)
- In Proxmox web UI → local storage → Upload ISO

### 3.2 Proxmox VM Settings

- Create VM:
  - **General:** Name: TrueNAS-Storage, VM ID: 100
  - **OS:** ISO Image: TrueNAS SCALE, Guest: Linux 6.x or later
  - **System:** q35, BIOS: OVMF (UEFI). Add EFI disk
  - **Disks:** Attach your RAID disks (SATA, suitable size)
- Allocate RAM/CPU generously

### 3.3 Install TrueNAS

1. Boot new VM and install TrueNAS per on-screen instructions.
2. After install, access TrueNAS web interface (note assigned IP).

---

## 📁 NFS and Dataset Setup

1. **Create ZFS Pool(s):** Use your attached disks (mirrored if possible).
2. **Create Datasets:**
    - `media` (for Plex content)
    - `documents` (for Paperless-ngx)
    - `backups` (for configs/snapshots)
3. **Configure NFS Shares** for each dataset for your Proxmox/Docker hosts:
    - Enable NFSv4, restrict to local subnet
    - Set permissions for target user/group (e.g. homelab-user:homelab)

---

## 🧪 Verification & Troubleshooting

- Test mounting NFS shares from a Linux system:
  ```
  showmount -e 10.0.0.11
  # Should display exports, try mounting manually if needed
  ```
- If pool creation or shares fail:
  - Check disks: *Storage* → *Disks*
  - Test network: `ping 10.0.0.11`
  - Verify NFS perms/Firewall

**If all else fails:**  
- Repair pool, recreate VM, or (before moving on) **reinstall Proxmox/TrueNAS** if it’s a new setup (safer to start fresh now).

---

## 🔒 Step 6: Initial Security Hardening

### 6.1 Proxmox 2FA

- Go to Datacenter → Authentication → Add Yubico OTP or TOTP
- Link YubiKey ([Yubico portal](https://www.yubico.com/))
- Enable 2FA for your user: Datacenter → Users → Edit → Two Factor
- Test login with 2FA

### 6.2 Harden Proxmox SSH

Edit `/etc/ssh/sshd_config`:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222
Protocol 2
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

- Create a separate `proxmox-admin` user and set up SSH pubkey auth

```
useradd -m -s /bin/bash -G sudo proxmox-admin
passwd proxmox-admin
mkdir /home/proxmox-admin/.ssh
echo "your-ssh-public-key" > /home/proxmox-admin/.ssh/authorized_keys
chown -R proxmox-admin:proxmox-admin /home/proxmox-admin/.ssh
chmod 700 /home/proxmox-admin/.ssh
chmod 600 /home/proxmox-admin/.ssh/authorized_keys
systemctl restart sshd
```

### 6.3 TrueNAS Hardening

- *Disable console menu access*: System Access → Console menu: Disable
- *Set up alert emails* and test delivery

---

## 🗂️ Backups & Snapshots

**Before moving on to Phase 2:**

- Create a Proxmox VM backup job for TrueNAS or manual backup
- Take a VM snapshot: `Phase1-Complete-Working`
- Download/export TrueNAS config (System → Save Config), store it externally

---

## ➡️ Next Steps

- Once all the above is verified, secure, and backed up, proceed to [Phase 2: Docker Services VM](./phase-2-docker-services.md).

---

**HomeLab Tip:**  
If at any point a step fails and you’re not in production, it is safer to restart from scratch than to try patching a broken foundation.

---

*For questions, see the [Installation README](../README.md) or open an issue!*
```
