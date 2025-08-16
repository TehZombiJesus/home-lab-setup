# Home Lab Installation Guide

## Phase 1: Proxmox + TrueNAS (Storage Foundation)

### Pre-Phase Checklist
- [ ] HP EliteDesk 800 G5 powered on and accessible
- [ ] USB drive with Proxmox VE ISO prepared
- [ ] Network cable connected (DHCP available)
- [ ] Monitor and keyboard connected for initial setup
- [ ] **CRITICAL**: Backup any existing data - this will wipe the system

### Phase 1 Overview
**Goal**: Install Proxmox hypervisor and create TrueNAS VM for centralized storage
**Time Estimate**: 2-3 hours
**Risk Level**: High (system wipe)

---

## Step 1: Proxmox VE Installation

### 1.1 Boot from USB
1. Insert Proxmox VE USB installer
2. Boot from USB (F9 on HP EliteDesk)
3. Select "Install Proxmox VE"

### 1.2 System Configuration
1. **Target Harddisk**: Select your RAID 1 array
2. **Country/Timezone**: Set appropriately  
3. **Administrator Password**: Use strong password - write it down!
4. **Email**: Your email for system alerts
5. **Management Interface**: 
  - Use DHCP initially
  - Note the IP address shown (you'll need this)

Step 1 Complete When**: You can access Proxmox web interface

---

## Step 2: Proxmox Initial Configuration

### 2.1 Repository Setup
```bash
# SSH into Proxmox or use Shell in web interface
# Remove enterprise repository (we're using free)
rm /etc/apt/sources.list.d/pve-enterprise.list

# Add no-subscription repository
echo "deb http://download.proxmox.com/debian/pve bookworm pve-no-subscription" > /etc/apt/sources.list.d/pve-no-subscription.list

# Update system
apt update && apt upgrade -y
```

### 2.2 Network Configuration (Static IP)
1. In Proxmox web interface: System Step 2 Complete When**: Proxmox accessible via static IP

---

## Step 3: TrueNAS VM Creation

### 3.1 Download TrueNAS ISO
1. In Proxmox: local (pve) Upload
2. Download TrueNAS SCALE ISO directly or upload from PC
3. Wait for upload to complete

### 3.2 Create TrueNAS VM
1. **Create VM** button in Proxmox
2. **General**:
  - VM ID: `100`
  - Name: `TrueNAS-Storage`
3. **OS**:
  - ISO: Select TrueNAS SCALE ISO
  - Guest OS: Linux 6.x - 2.6 Kernel
4. **System**:
  - Machine: q35
  - BIOS: OVMF (UEFI)
  - Add EFI Disk: Hardware Hard Disk
2. **For each storage disk you want to add**:
  - Bus/Device: SATA (next available)
  - Disk size: According to your setup
  - Cache: Write back
3. **Minimum recommendation**: 2x 1TB disks for mirrored storage

** **Interfaces** **General** **Pools** **Pools** **Add Dataset**
2. Create these datasets:
  - `media` (for Plex content)
  - `documents` (for Paperless-ngx)
  - `backups` (for VM backups and configs)

** **NFS** Start Automatically
3. **NFSv4**: **Unix (NFS) Shares** Enabled
  - **Quiet**: Enabled
  - **Quiet**: Enabled
  - **Quiet**: **Users** **Pools** **Edit Permissions**
2. Set owner to `homelab-user:homelab` (1000:1000)
3. **Apply recursively**: Step 5 Complete When**: You can mount NFS shares from another Linux system

---

## Phase 1 Completion Criteria

### TrueNAS VM Check all settings
2. Verify ISO is properly attached
3. Check VM logs: TrueNAS VM Log
4. Delete VM and recreate if necessary

### If Storage Pool Creation Fails:
1. **Storage** **Export/Disconnect** pool
2. Verify disks are healthy: **Storage** **NFS**
2. Verify network connectivity: `ping 10.0.0.110`
3. Check share permissions and network restrictions
4. Test from another Linux machine: `showmount -e 10.0.0.110`

### Complete Reset:
1. If everything fails, reinstall Proxmox from scratch
2. VMs can be recreated easily at this stage
3. No data loss since this is foundation phase

---

## Pre-Phase 2 Snapshot

**Before proceeding to Phase 2:**

1. **Proxmox Backup**:
  - Datacenter Create backup job for TrueNAS VM
  - Or manually: TrueNAS VM Backup now

2. **VM Snapshot**:
  - TrueNAS VM Take Snapshot
  - Name: `Phase1-Complete-Working`

3. **Configuration Backup**:
  - Export TrueNAS configuration: **System** **Save Config**
  - Save file to external location

---

## Step 6: Initial Security Hardening

### 6.1 Proxmox 2FA Setup (YubiKey/TOTP)
1. **Datacenter** **Add** **Authentication** **Yubico OTP**
2. **ID**: `yubikey`
3. **API ID** and **Secret Key**: From Yubico account
4. Configure your YubiKey at yubico.com

**Enable 2FA for root user**:
1. **Datacenter** **Users** **Edit**
2. **Two Factor**: Select `totp` or `yubikey`
3. **Setup TOTP**: Scan QR code with authenticator app
4. **Test login**: Logout and login with 2FA

### 6.2 Proxmox SSH Hardening
```bash
# SSH into Proxmox host
nano /etc/ssh/sshd_config

# Add/modify these settings:
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
Port 2222
Protocol 2
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
```

**Create non-root user for SSH**:
```bash
# Create admin user
useradd -m -s /bin/bash -G sudo proxmox-admin
passwd proxmox-admin

# Set up SSH key authentication (replace with your public key)
mkdir /home/proxmox-admin/.ssh
echo "your-ssh-public-key" > /home/proxmox-admin/.ssh/authorized_keys
chown -R proxmox-admin:proxmox-admin /home/proxmox-admin/.ssh
chmod 700 /home/proxmox-admin/.ssh
chmod 600 /home/proxmox-admin/.ssh/authorized_keys

# Restart SSH service
systemctl restart sshd
```

### 6.3 TrueNAS Security Hardening  
1. **System** **Access**
  - **Console Menu**: Disable (prevents local access without login)
  - **Serial Console**: Disable if not needed
  
2. **Network** **Alert Services**
  - Configure email alerts for system issues
  - Test alert delivery

4. **System** Step 6 Complete When**: 
- [ ] 2FA working for Proxmox web interface
- [ ] SSH hardened with key-only authentication
- [ ] TrueNAS secured with basic hardening
- [ ] Initial Cloudflare tunnel working for Proxmox access (optional at this stage)

---

## Next: Phase 2 - Docker Services VM
Ready to proceed when Phase 1 completion criteria are met and snapshots are taken.
