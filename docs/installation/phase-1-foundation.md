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

### 1.3 Complete Installation
1. Remove USB drive when prompted
2. Reboot system
3. Access web interface: `https://[IP-ADDRESS]:8006`
4. Login as `root` with the password you set

**✅ Step 1 Complete When**: You can access Proxmox web interface

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
1. In Proxmox web interface: System → Network
2. Edit your network interface (usually `vmbr0`)
3. Change from DHCP to static:
   - **IP Address**: `192.168.1.100/24` (adjust for your network)
   - **Gateway**: Your router IP (usually `192.168.1.1`)
   - **DNS**: `192.168.1.1` or `8.8.8.8`
4. Apply configuration and reboot

**✅ Step 2 Complete When**: Proxmox accessible via static IP

---

## Step 3: TrueNAS VM Creation

### 3.1 Download TrueNAS ISO
1. In Proxmox: local (pve) → ISO Images → Upload
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
   - Add EFI Disk: ✓
5. **Disks**:
   - Bus/Device: SATA 0
   - Disk size: 32 GB (OS disk)
   - Cache: Write back
6. **CPU**:
   - Cores: 2
   - Type: host
7. **Memory**: 8192 MB (8GB)
8. **Network**: Default (vmbr0)

### 3.3 Add Storage Disks to TrueNAS VM
1. Select TrueNAS VM → Hardware → Add → Hard Disk
2. **For each storage disk you want to add**:
   - Bus/Device: SATA (next available)
   - Disk size: According to your setup
   - Cache: Write back
3. **Minimum recommendation**: 2x 1TB disks for mirrored storage

**✅ Step 3 Complete When**: TrueNAS VM created with all disks attached

---

## Step 4: TrueNAS Installation & Configuration

### 4.1 Install TrueNAS
1. Start TrueNAS VM
2. Open Console
3. Follow TrueNAS installer:
   - Install on the 32GB OS disk (SATA 0)
   - Set root password (write it down!)
   - Complete installation and reboot

### 4.2 Initial TrueNAS Setup
1. Note IP address from console
2. Access web interface: `http://[TRUENAS-IP]`
3. Login as `root`
4. **Network** → **Interfaces** → Set static IP if needed
5. **System** → **General** → Set timezone

### 4.3 Create Storage Pool
1. **Storage** → **Pools** → **Add**
2. **Pool Name**: `main-storage`
3. **Layout**: Mirror (for 2 disks) or appropriate RAID
4. Select your storage disks (NOT the OS disk)
5. **Create Pool**

### 4.4 Create Datasets
1. **Storage** → **Pools** → `main-storage` → **Add Dataset**
2. Create these datasets:
   - `media` (for Plex content)
   - `documents` (for Paperless-ngx)
   - `game-data` (for game server data)
   - `backups` (for VM backups)

**✅ Step 4 Complete When**: TrueNAS accessible via web interface with storage pool created

---

## Step 5: Network Shares Setup

### 5.1 Create SMB Shares
1. **Sharing** → **Windows (SMB) Shares** → **Add**
2. For each dataset, create a share:
   - **Path**: `/mnt/main-storage/[dataset-name]`
   - **Name**: `[dataset-name]`
   - **Purpose**: Default share parameters

### 5.2 Create User Account
1. **Accounts** → **Users** → **Add**
2. **Username**: `homelab-user`
3. **Password**: Set strong password
4. **Primary Group**: Create new group `homelab`
5. **Auxiliary Groups**: Add to appropriate groups

### 5.3 Set Permissions
1. **Storage** → **Pools** → `main-storage` → **Edit Permissions**
2. Set owner to `homelab-user:homelab`
3. Apply recursively

**✅ Step 5 Complete When**: You can access SMB shares from another computer

---

## Phase 1 Completion Criteria

### ✅ Must Be Working:
- [ ] Proxmox web interface accessible via static IP
- [ ] TrueNAS VM running and accessible via web interface
- [ ] Storage pool created and healthy
- [ ] All datasets created
- [ ] SMB shares accessible from network
- [ ] Both systems have static IP addresses

### 📊 Resource Usage Check:
- **Proxmox Host**: ~2GB RAM used by hypervisor
- **TrueNAS VM**: 8GB RAM allocated
- **Total Used**: ~10GB of 64GB available
- **Storage Pool**: Healthy status, no errors

---

## Rollback Strategy

### If Proxmox Installation Fails:
1. Reboot from USB installer
2. Choose "Rescue" mode to access existing installation
3. Or completely restart installation process

### If TrueNAS VM Won't Start:
1. Proxmox → TrueNAS VM → Hardware → Check all settings
2. Verify ISO is properly attached
3. Check VM logs: TrueNAS VM → Monitor → Log
4. Delete VM and recreate if necessary

### If Storage Pool Creation Fails:
1. **Storage** → **Pools** → **Export/Disconnect** pool
2. Verify disks are healthy: **Storage** → **Disks**
3. Recreate pool with different configuration
4. Check Proxmox VM disk allocation

### Complete Reset:
1. If everything fails, reinstall Proxmox from scratch
2. VMs can be recreated easily at this stage
3. No data loss since this is foundation phase

---

## Pre-Phase 2 Snapshot

**Before proceeding to Phase 2:**

1. **Proxmox Backup**:
   - Datacenter → Backup → Create backup job for TrueNAS VM
   - Or manually: TrueNAS VM → Backup → Backup now

2. **VM Snapshot**:
   - TrueNAS VM → Snapshots → Take Snapshot
   - Name: `Phase1-Complete-Working`

3. **Configuration Backup**:
   - Export TrueNAS configuration: **System** → **General** → **Save Config**
   - Save file to external location

**Phase 1 is complete when all completion criteria are met and backups are taken.**

---

## Next: Phase 2 - Docker Services VM
Ready to proceed when Phase 1 completion criteria are met and snapshots are taken.
