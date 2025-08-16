# Recovery Procedures - Emergency Reference Guide

## 🚨 Quick Emergency Contacts

```
Primary Contact: admin@tehzombijesus.ca
OVH Support: https://help.ovhcloud.com/
Backup System Status: /var/log/ovh-backup.log
Cost Dashboard: /opt/scripts/backups/ovh-cost-monitor.sh --report
```

---

## 🎵 Music Library Recovery (Priority #1)

### **Scenario 1: Single Music File Recovery**

**⏱️ Expected Time: 5-30 minutes**

```bash
# Step 1: Check TrueNAS snapshots first (fastest)
ssh truenas-vm "ls /mnt/pool/music/.zfs/snapshot/"
ssh truenas-vm "find /mnt/pool/music/.zfs/snapshot/ -name '*your-song-name*'"

# Step 2: Copy from snapshot if found
ssh truenas-vm "cp '/mnt/pool/music/.zfs/snapshot/hourly-2025-XX-XX/path/to/song.mp3' /mnt/pool/music/restored/"

# Step 3: If not in snapshots, check recent incremental backup
rclone ls ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/ | grep music-incremental

# Step 4: Download and decrypt if needed
rclone copy "ovh-s3:homelab-backups/2025/08/music-incremental-YYYYMMDD.tar.gz.gpg" /tmp/
gpg --decrypt /tmp/music-incremental-YYYYMMDD.tar.gz.gpg > /tmp/music-backup.tar.gz
tar -xzf /tmp/music-backup.tar.gz -C /tmp/
# Find and restore your file
```

### **Scenario 2: Complete Music Library Loss**

**⏱️ Expected Time: 2-4 hours**

```bash
# Step 1: Create recovery workspace
mkdir -p /tmp/music-recovery
cd /tmp/music-recovery

# Step 2: Download latest monthly full backup
LATEST_FULL=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/ | grep music-full | tail -1 | awk '{print $2}')
echo "Downloading: $LATEST_FULL"
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$LATEST_FULL" ./

# Step 3: Download all incremental backups since full backup
FULL_DATE=$(echo $LATEST_FULL | grep -o '[0-9]\{8\}')
rclone ls ovh-s3:homelab-backups/$(date +%Y)/ | grep music-incremental | \
    awk -v date="$FULL_DATE" '$2 > "music-incremental-"date {print $2}' | \
    while read backup; do
        echo "Downloading incremental: $backup"
        rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$backup" ./
    done

# Step 4: Decrypt and restore base library
gpg --decrypt $LATEST_FULL > music-full.tar.gz
tar -xzf music-full.tar.gz
cp -r mnt/pool/media/music/* /mnt/pool/music/

# Step 5: Apply incremental changes (in chronological order)
for incremental in music-incremental-*.tar.gz.gpg; do
    echo "Applying incremental: $incremental"
    gpg --decrypt "$incremental" > temp-incremental.tar.gz
    tar -xzf temp-incremental.tar.gz
    rsync -av mnt/pool/media/music/ /mnt/pool/music/
    rm -rf mnt/ temp-incremental.tar.gz
done

# Step 6: Verify restoration
find /mnt/pool/music -name "*.mp3" -o -name "*.flac" -o -name "*.m4a" | wc -l
echo "Music files restored. Compare with inventory file if available."

# Cleanup
cd /
rm -rf /tmp/music-recovery
```

---

## 📄 Document Recovery

### **Scenario 1: Single Document Recovery**

**⏱️ Expected Time: 10-20 minutes**

```bash
# Step 1: Check Paperless-ngx snapshots
ssh truenas-vm "ls /mnt/pool/documents/.zfs/snapshot/"

# Step 2: Browse recent snapshots for document
ssh truenas-vm "find /mnt/pool/documents/.zfs/snapshot/hourly-$(date +%Y-%m-%d)* -name '*.pdf' | grep -i 'document-name'"

# Step 3: If not in snapshots, download from OVH
LATEST_DOC=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/ | grep documents | tail -1 | awk '{print $2}')
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/$LATEST_DOC" /tmp/
gpg --decrypt "/tmp/$LATEST_DOC" > /tmp/documents.tar.gz
tar -tzf /tmp/documents.tar.gz | grep -i 'document-name'
tar -xzf /tmp/documents.tar.gz --wildcards '*document-name*'
```

### **Scenario 2: Complete Document Database Recovery**

**⏱️ Expected Time: 1-2 hours**

```bash
# Step 1: Stop Paperless-ngx service
docker stop paperless-ngx

# Step 2: Download latest document backup
LATEST_DOC=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/ | grep documents | tail -1 | awk '{print $2}')
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/$LATEST_DOC" /tmp/

# Step 3: Decrypt and extract
gpg --decrypt "/tmp/$LATEST_DOC" > /tmp/documents-restore.tar.gz
tar -xzf /tmp/documents-restore.tar.gz -C /tmp/

# Step 4: Restore database and files
cp /tmp/mnt/pool/backups/staging/paperless-metadata-*.sql /tmp/restore.sql
cp -r /tmp/mnt/pool/documents/* /mnt/pool/documents/

# Step 5: Import database
docker start paperless-ngx
sleep 30
docker exec paperless-ngx python manage.py shell < /tmp/restore.sql

# Step 6: Verify restoration
docker logs paperless-ngx | grep -i error
echo "Document recovery complete. Check Paperless-ngx web interface."
```

---

## 🖥️ System Configuration Recovery

### **Scenario 1: VM Configuration Recovery**

**⏱️ Expected Time: 30-60 minutes**

```bash
# Step 1: Download latest config backup
LATEST_CONFIG=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/ | grep configs | tail -1 | awk '{print $2}')
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/$LATEST_CONFIG" /tmp/

# Step 2: Decrypt and extract
gpg --decrypt "/tmp/$LATEST_CONFIG" > /tmp/configs.tar.gz
tar -xzf /tmp/configs.tar.gz -C /tmp/

# Step 3: Restore Proxmox configuration
cp -r /tmp/opt/backups/configs/*/proxmox-config/* /etc/pve/

# Step 4: Recreate VMs from configuration files
for vm_config in /tmp/opt/backups/configs/*/vm-*.conf; do
    vm_id=$(basename "$vm_config" .conf | grep -o '[0-9]*')
    echo "Restoring VM $vm_id configuration..."
    qm create $vm_id --config "$vm_config"
done

# Step 5: Restore network configuration
cp /tmp/opt/backups/configs/*/interfaces /etc/network/
cp /tmp/opt/backups/configs/*/hosts /etc/hosts

# Step 6: Restart networking and verify
systemctl restart networking
qm list
```

### **Scenario 2: TrueNAS Configuration Recovery**

**⏱️ Expected Time: 20-30 minutes**

```bash
# Step 1: Download TrueNAS config from staging
ssh truenas-vm "ls -la /mnt/pool/backups/staging/truenas-config-*.db"

# Step 2: If staging not available, get from OVH
LATEST_DOC=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/ | grep documents | tail -1 | awk '{print $2}')
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/$LATEST_DOC" /tmp/
gpg --decrypt "/tmp/$LATEST_DOC" > /tmp/documents.tar.gz
tar -xzf /tmp/documents.tar.gz
CONFIG_FILE=$(find /tmp -name "truenas-config-*.db" | head -1)

# Step 3: Upload and restore configuration via TrueNAS web interface
echo "1. Go to TrueNAS web interface (https://truenas-vm)"
echo "2. Navigate to System > General > Save Config"
echo "3. Click 'Upload Config' and select: $CONFIG_FILE"
echo "4. Reboot TrueNAS after import"
echo "5. Verify datasets and shares are restored"
```

---

## 🔧 Complete System Recovery (Disaster Scenario)

### **⏱️ Expected Time: 8-24 hours**

**Phase 1: Hardware and Base System (2-4 hours)**

```bash
# Step 1: Install Proxmox on new/replacement hardware
# - Download Proxmox VE ISO
# - Boot from USB and install
# - Configure network settings
# - Update system: apt update && apt upgrade

# Step 2: Install essential packages
apt install -y rclone gpg bc jq curl

# Step 3: Configure YubiKey SSH access
# - Insert YubiKey
# - Generate new SSH key: ssh-keygen -t ed25519-sk -O resident
# - Test YubiKey functionality

# Step 4: Configure rclone for OVH access
rclone config
# Configure ovh-s3 remote with saved credentials
rclone ls ovh-s3:homelab-backups  # Test connectivity
```

**Phase 2: Configuration Recovery (2-3 hours)**

```bash
# Step 5: Download and restore system configurations
mkdir -p /tmp/disaster-recovery
cd /tmp/disaster-recovery

# Get latest configs
LATEST_CONFIG=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/ | grep configs | tail -1 | awk '{print $2}')
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/$LATEST_CONFIG" ./

# Decrypt and restore
gpg --decrypt "$LATEST_CONFIG" > configs.tar.gz
tar -xzf configs.tar.gz

# Restore network configuration
cp opt/backups/configs/*/interfaces /etc/network/
cp opt/backups/configs/*/hosts /etc/hosts
systemctl restart networking
```

**Phase 3: VM Recovery (3-6 hours)**

```bash
# Step 6: Download and restore VM templates
LATEST_TEMPLATES=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/ | grep vm-templates | tail -1 | awk '{print $2}')
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$LATEST_TEMPLATES" ./
gpg --decrypt "$LATEST_TEMPLATES" > vm-templates.tar.gz
tar -xzf vm-templates.tar.gz -C /var/lib/vz/

# Step 7: Recreate critical VMs
for vm_config in opt/backups/configs/*/vm-*.conf; do
    vm_id=$(basename "$vm_config" .conf | grep -o '[0-9]*')
    if [[ "$vm_id" == "103" || "$vm_id" == "104" ]]; then  # Critical VMs only
        echo "Creating VM $vm_id..."
        qm create $vm_id --config "$vm_config"
        qm start $vm_id
        sleep 60  # Wait for boot
    fi
done
```

**Phase 4: Data Recovery (2-8 hours)**

```bash
# Step 8: Restore music library (PRIORITY)
mkdir -p /tmp/music-recovery
cd /tmp/music-recovery

# Download latest full music backup
LATEST_MUSIC=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/ | grep music-full | tail -1 | awk '{print $2}')
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$LATEST_MUSIC" ./
gpg --decrypt "$LATEST_MUSIC" > music-full.tar.gz

# Wait for TrueNAS VM to be ready, then restore
while ! ssh truenas-vm "echo ready" 2>/dev/null; do
    echo "Waiting for TrueNAS VM..."
    sleep 30
done

# Extract music to TrueNAS
tar -xzf music-full.tar.gz
scp -r mnt/pool/media/music/* truenas-vm:/mnt/pool/music/

# Step 9: Restore documents
LATEST_DOC=$(rclone ls ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/ | grep documents | tail -1 | awk '{print $2}')
rclone copy "ovh-s3:homelab-backups/$(date +%Y)/$(date +%m)/$LATEST_DOC" ./
gpg --decrypt "$LATEST_DOC" > documents.tar.gz
tar -xzf documents.tar.gz
scp -r mnt/pool/documents/* truenas-vm:/mnt/pool/documents/

# Step 10: Restore backup scripts and cron jobs
git clone https://github.com/TehZombiJesus/home-lab-setup.git
cp -r home-lab-setup/scripts/backups/* /opt/scripts/backups/
cp home-lab-setup/configs/cron/homelab-ovh-backups /etc/cron.d/
chmod +x /opt/scripts/backups/*.sh
```

**Phase 5: Verification and Testing (1-2 hours)**

```bash
# Step 11: Verify system functionality
qm list                                    # Check VM status
ssh truenas-vm "zpool status"             # Check ZFS health
find /mnt/pool/music -name "*.mp3" | wc -l # Count music files
docker ps                                  # Check services in Docker VM

# Step 12: Test backup system
/opt/scripts/backups/backup-health-check.sh
/opt/scripts/backups/ovh-cost-monitor.sh --report

# Step 13: Resume normal operations
systemctl enable cron
echo "Disaster recovery complete. Monitor logs for 24 hours."
echo "Begin media library re-acquisition as time permits."
```

---

## 🔍 Recovery Verification Checklist

### **Post-Recovery Validation**

**System Level:**
- [ ] All critical VMs running (103: Docker, 104: TrueNAS)
- [ ] Network connectivity working
- [ ] YubiKey SSH access to TrueNAS functional
- [ ] OVH S3 connectivity working
- [ ] Backup scripts executable and scheduled

**Data Level:**
- [ ] Music library file count matches pre-disaster inventory
- [ ] Document database accessible via Paperless-ngx
- [ ] ZFS snapshots resuming normally
- [ ] Container services in Docker VM running

**Backup System:**
- [ ] Backup scripts completing successfully
- [ ] Cost monitoring functional
- [ ] Alerts and monitoring restored
- [ ] GPG decryption working

**Testing:**
- [ ] Perform test backup to OVH
- [ ] Verify snapshot recovery works
- [ ] Test music file recovery from snapshots
- [ ] Confirm document search functionality

---

## 🆘 Emergency Troubleshooting

### **Common Recovery Issues**

**GPG Decryption Fails:**
```bash
# Check GPG key availability
gpg --list-keys backup@tehzombijesus.ca

# If key missing, restore from secure backup
gpg --import /root/secure/backup-private-key.gpg

# Test decryption with small file first
echo "test" | gpg --encrypt --recipient backup@tehzombijesus.ca | gpg --decrypt
```

**YubiKey SSH Not Working:**
```bash
# Check YubiKey detection
lsusb | grep Yubico

# Verify SSH key loaded
ssh-add -l

# Test with verbose output
ssh -v -i ~/.ssh/yubikey_backup_key truenas-vm
```

**OVH S3 Connectivity Issues:**
```bash
# Test basic connectivity
curl -I https://s3.gra.cloud.ovh.net

# Check rclone config
rclone config show ovh-s3

# Test with verbose logging
rclone -v ls ovh-s3:homelab-backups
```

**Music Recovery Incomplete:**
```bash
# Check for incremental backups missed
rclone ls ovh-s3:homelab-backups/$(date +%Y)/ | grep music-incremental | sort

# Verify extraction directory structure
find /tmp/music-recovery -type d | head -10

# Check for file corruption
find /mnt/pool/music -name "*.mp3" -exec file {} \; | grep -v "Audio file"
```

---

## 📞 Emergency Contact Protocol

### **Escalation Path**

1. **Self-Resolution** (0-2 hours): Use this guide
2. **Community Support** (2-6 hours): Reddit r/homelab, Discord servers  
3. **Professional Support** (6+ hours): OVH support, local IT consultant
4. **Data Recovery Services** (Last resort): Professional data recovery if hardware failure

### **Information to Gather Before Seeking Help**

```bash
# System info
uname -a
df -h
free -m
qm list

# Backup system status
ls -la /var/log/*backup*
tail -50 /var/log/ovh-backup.log
rclone --version
gpg --version

# Error messages
journalctl -xe | grep -i error | tail -20
```

Remember: **Music library is Priority #1** - focus recovery efforts there first, then documents, then system configs. Everything else can be rebuilt over time.
