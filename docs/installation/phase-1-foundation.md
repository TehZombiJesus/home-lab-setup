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
   - **IP Address**: `10.0.0.100/24` (adjust for your network)
   - **Gateway**: Your router IP (usually `10.0.0.1`)
   - **DNS**: `9.9.9.9` (Quad9 - privacy-focused DNS)
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
4. **Network** → **Interfaces** → Set static IP if needed (e.g., `10.0.0.110`)
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
   - `backups` (for VM backups and configs)

**✅ Step 4 Complete When**: TrueNAS accessible via web interface with storage pool created

---

## Step 5: NFS Shares Setup

### 5.1 Enable NFS Service
1. **Services** → **NFS** → **Configure**
2. **Enable**: ✓ Start Automatically
3. **NFSv4**: ✓ Enable NFSv4
4. **Bind IP Addresses**: Select your TrueNAS interface
5. **Save** and **Start** the service

### 5.2 Create NFS Shares
1. **Sharing** → **Unix (NFS) Shares** → **Add**
2. For each dataset, create an NFS share:
   
   **Media Share**:
   - **Path**: `/mnt/main-storage/media`
   - **Networks**: `10.0.0.0/24`
   - **All Directories**: ✓ Enabled
   - **Quiet**: ✓ Enabled
   
   **Documents Share**:
   - **Path**: `/mnt/main-storage/documents`  
   - **Networks**: `10.0.0.0/24`
   - **All Directories**: ✓ Enabled
   - **Quiet**: ✓ Enabled
   
   **Backups Share**:
   - **Path**: `/mnt/main-storage/backups`
   - **Networks**: `10.0.0.0/24` 
   - **All Directories**: ✓ Enabled
   - **Quiet**: ✓ Enabled

### 5.3 Create User Account
1. **Accounts** → **Users** → **Add**
2. **Username**: `homelab-user`
3. **Password**: Set strong password
4. **UID**: `1000` (to match VM user IDs)
5. **Primary Group**: Create new group `homelab` with GID `1000`

### 5.4 Set Dataset Permissions  
1. **Storage** → **Pools** → `main-storage` → **Edit Permissions**
2. Set owner to `homelab-user:homelab` (1000:1000)
3. **Apply recursively**: ✓ Enabled
4. Repeat for each dataset (media, documents, backups)

**✅ Step 5 Complete When**: You can mount NFS shares from another Linux system

---

## Phase 1 Completion Criteria

### ✅ Must Be Working:
- [ ] Proxmox web interface accessible via static IP
- [ ] TrueNAS VM running and accessible via web interface  
- [ ] Storage pool created and healthy
- [ ] All datasets created (media, documents, backups)
- [ ] NFS shares accessible from network
- [ ] Both systems have static IP addresses

### 📊 Resource Usage Check:
- **Proxmox Host**: ~2GB RAM used by hypervisor
- **TrueNAS VM**: 8GB RAM allocated
- **Total Used**: ~10GB of 64GB available
- **Storage Pool**: Healthy status, no errors
- **Network**: NFS service running, shares accessible

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

### If NFS Shares Won't Mount:
1. Check NFS service status: **Services** → **NFS**
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
   - Datacenter → Backup → Create backup job for TrueNAS VM
   - Or manually: TrueNAS VM → Backup → Backup now

2. **VM Snapshot**:
   - TrueNAS VM → Snapshots → Take Snapshot
   - Name: `Phase1-Complete-Working`

3. **Configuration Backup**:
   - Export TrueNAS configuration: **System** → **General** → **Save Config**
   - Save file to external location

---

## Step 6: Initial Security Hardening

### 6.1 Proxmox 2FA Setup (YubiKey/TOTP)
1. **Datacenter** → **Authentication** → **Add** → **TOTP**
2. **ID**: `totp`
3. **Description**: `Two-Factor Authentication`
4. **Issuer**: `Proxmox-HomeLab`
5. **Digits**: `6`
6. **Step**: `30`

**For YubiKey Users**:
1. **Datacenter** → **Authentication** → **Add** → **Yubico OTP**
2. **ID**: `yubikey`
3. **API ID** and **Secret Key**: From Yubico account
4. Configure your YubiKey at yubico.com

**Enable 2FA for root user**:
1. **Datacenter** → **Permissions** → **Users** → **root** → **Edit**
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
1. **System** → **Advanced** → **Access**
   - **Console Menu**: Disable (prevents local access without login)
   - **Serial Console**: Disable if not needed
   
2. **Network** → **Global Configuration**
   - **Enable SSH**: Only if needed, change default port
   - **Root SSH Login**: Disable
   
3. **System** → **Alert Services**
   - Configure email alerts for system issues
   - Test alert delivery

4. **System** → **Update**
   - **Check for Updates**: Enable automatic checks
   - **Download Updates**: Enable for security patches

### 6.4 Initial Cloudflare Setup (Management Access)
1. **Create Cloudflare Account** (if not already done)
2. **Add Domain** to Cloudflare DNS management
3. **Install cloudflared** on a management machine:
```bash
# On your main computer/laptop (not Proxmox)
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared.deb
```

4. **Create initial tunnel** for Proxmox management:
```bash
cloudflared tunnel login
cloudflared tunnel create proxmox-mgmt
cloudflared tunnel route dns proxmox-mgmt proxmox.yourdomain.com
```

5. **Configure tunnel** (create config file):
```yaml
# ~/.cloudflared/config.yml
tunnel: proxmox-mgmt
credentials-file: /home/user/.cloudflared/[tunnel-id].json

ingress:
  - hostname: proxmox.yourdomain.com
    service: https://10.0.0.100:8006
    originRequest:
      noTLSVerify: true
  - service: http_status:404
```

6. **Start tunnel**:
```bash
cloudflared tunnel run proxmox-mgmt
```

**✅ Step 6 Complete When**: 
- [ ] 2FA working for Proxmox web interface
- [ ] SSH hardened with key-only authentication
- [ ] TrueNAS secured with basic hardening
- [ ] Initial Cloudflare tunnel working for Proxmox access (optional at this stage)

---

## Next: Phase 2 - Docker Services VM
Ready to proceed when Phase 1 completion criteria are met and snapshots are taken.
