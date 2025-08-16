# Simple Home Lab Backup Strategy - tehzombijesus.ca

## 🎯 Home Lab Backup Philosophy

**Keep it simple, focus on what matters:**
- Protect configurations and personal documents (irreplaceable)
- Backup music files and metadata (important but not critical)
- **Never backup movies/TV shows** (easily replaceable, waste of money)
- Use local snapshots for quick recovery, cloud for disaster protection
- Minimize maintenance overhead - this should run itself

## 📋 What Gets Backed Up (Priority Order)

### **Priority 1: Critical Data** 
- VM configurations and containers
- System configurations (Proxmox, network, etc.)
- Personal documents from Paperless-ngx
- Game server worlds and configurations

### **Priority 2: Important Data**
- Music library (~900 songs from Spotify)
- Music metadata (playlists, ratings, play counts)

### **Priority 3: Never Backed Up**
- Movies, TV shows, books, audiobooks (replaceable content)
- Media automation downloads (Sonarr/Radarr/Lidarr)
- Temp files, logs, cache data

## 🏠 Simple 3-2-1 Implementation

### **3-2-1 Rule for Home Labs:**
```
3 Copies: Original + Local Backup + Cloud Backup
2 Media Types: NVMe (local) + OVH Object Storage (cloud)  
1 Offsite: OVH cloud storage
```

### **Copy #1: Original Data**
- Live data on TrueNAS datasets
- Active VMs on Proxmox
- Real-time snapshots for quick recovery

### **Copy #2: Local Backups (Different Media)**
**Proxmox VM Backups:**
```yaml
Target VMs: Docker Services (103), TrueNAS (104) only
Frequency: Weekly (Sundays at 2 AM)
Retention: 4 backups (1 month)
Storage: Local NVMe (/var/lib/vz/dump)
```

**TrueNAS Backup Staging:**
```yaml
Location: /mnt/pool/backups/ (different dataset)
Contents: 
  - Exported documents (weekly)
  - Music library sync (weekly)
  - System configs export (weekly)
Retention: 2 weeks (staging for cloud upload)
```

### **Copy #3: Cloud Backup (Offsite)**
**OVH Object Storage:**
- VM backups (monthly upload)
- Documents and configs (weekly)
- Music library (monthly full + weekly changes)
- Different geographic location (Beauharnois, Quebec)

### **TrueNAS Snapshots (Quick Recovery)**
**Documents Dataset:**
```yaml
Snapshots:
  hourly: 24 snapshots (1 day - quick recovery)
  daily: 7 snapshots (1 week)
  weekly: 4 snapshots (1 month)
```

**Music Dataset:**
```yaml
Snapshots:
  daily: 7 snapshots (1 week)
  weekly: 4 snapshots (1 month)
```

**Media Dataset (No 3-2-1 protection):**
```yaml
Snapshots:
  daily: 3 snapshots (accidental deletion protection only)
  
Note: Media intentionally excluded from 3-2-1 rule (cost savings)
```

## ☁️ Simple Cloud Backup (OVH)

### **One Simple Backup Script**

Instead of 7 different scripts, one weekly backup that handles everything:

**What goes to OVH:**
- Latest Proxmox VM backups (Docker + TrueNAS VMs)
- System configurations export
- Documents from Paperless-ngx
- Music library (monthly full + weekly incrementals)

**What doesn't go to OVH BHS:**
- Movies, TV, books (saves ~$200/year in storage costs)
- Media automation temp files
- Replaceable data

### **Simplified 3-2-1 Schedule**
```cron
# Weekly local backups (Copy #2) - Sunday 1 AM
0 1 * * 0 root /opt/scripts/local-backup.sh

# Weekly cloud sync (Copy #3) - Sunday 3 AM  
0 3 * * 0 root /opt/scripts/cloud-backup.sh

# Daily health check
0 6 * * * root /opt/scripts/backup-health.sh
```

### **3-2-1 Cost Optimization**
```
Estimated Monthly Storage (3 copies total):

Copy #1 (Original): No additional cost
Copy #2 (Local NVMe): ~10GB staging area
Copy #3 (OVH Cloud):
├── VM backups: 3GB (monthly upload)
├── Documents: 2GB (weekly sync)
├── Music: 1GB (monthly full + weekly changes)
├── System configs: 0.5GB
└── Total: ~6.5GB = $0.63/month = $7.56/year

True 3-2-1 compliance while still saving $200+/year by excluding media
```

## 🔧 Simple Implementation

### **Single Backup Script Structure**
```bash
#!/bin/bash
# /opt/scripts/simple-backup.sh

# 1. Export system configs (no encryption complexity)
# 2. Sync documents from TrueNAS
# 3. Weekly music sync (rsync for incrementals)
# 4. Upload VM backups if newer than last upload
# 5. Simple success/failure notification

# No GPG encryption (complexity not worth it for home use)
# No YubiKey SSH (use regular SSH keys)
# No cost monitoring (storage is cheap at this scale)
```

### **Simple 3-2-1 Recovery Approach**

**For 90% of problems** (deleted file, broken config):
1. **Copy #1**: Check TrueNAS snapshots first (30 seconds)
2. **Copy #2**: Check local backup staging if snapshots insufficient  
3. **Copy #3**: Download from OVH if needed (rare)

**For major disasters** (hardware failure, complete data loss):
1. Install fresh Proxmox on new hardware
2. Download Copy #3 (cloud backups) from OVH
3. Restore VMs and configurations  
4. Restore Copy #2 data from local backups (if hardware survived)
5. Rebuild what's missing

**Recovery Time with 3-2-1:**
- Single file: 30 seconds (snapshots)
- VM restoration: 1-2 hours (local backups)
- Full disaster recovery: 4-6 hours (cloud backups)
- Data loss: Maximum 1 week (acceptable for home use)

## 🖥️ Simplified TrueNAS Storage Layout

```
TrueNAS Pool (1.8TB):
├── documents/          200GB  [BACKED UP]
│   └── paperless data
├── music/             150GB  [BACKED UP] 
│   └── ~900 songs + metadata
├── media/            1.4TB   [NOT BACKED UP]
│   ├── movies/         ~800GB
│   ├── tv/            ~400GB  
│   └── books/          ~200GB
└── backups/           50GB   [STAGING]
    └── temp backup files before cloud upload
```

## 📊 Simple Monitoring

### **Basic Health Checks**
- Did last backup complete? (check timestamp)
- Are snapshots being created? (count recent snapshots)
- Is OVH reachable? (simple connectivity test)

**Alert only on failures, not metrics**

### **Integration with Uptime Kuma**
```yaml
Simple Monitors:
  - "Weekly Backup Status" (check for success file)
  - "TrueNAS Health" (basic ping + SSH)
  - "Document Snapshots" (verify recent snapshots exist)
```

## 🎯 Why This Approach Works Better

### **Maintenance Reality Check**
- **Setup time**: 2 hours instead of 2 weeks
- **Monthly maintenance**: 10 minutes instead of 2 hours  
- **Recovery complexity**: Web UI clicks instead of command line procedures
- **Failure modes**: Simple to troubleshoot

### **Home Lab Priorities**
- **Convenience over perfection** - Good enough protection with minimal effort
- **Cost effectiveness** - Don't backup replaceable content
- **Time value** - Your time is worth more than perfect backup strategies
- **Simplicity** - Less things to break, easier to fix when they do

### **Music Protection Strategy**
- **Local snapshots**: Quick recovery for recent changes
- **Weekly cloud backup**: Disaster protection
- **Acceptable loss**: Up to 1 week of music additions (you can re-download)
- **Focus on metadata**: Playlists and ratings matter more than files

## 🚀 Implementation Plan

### **Week 1: Simplify**
1. Remove complex backup scripts
2. Deploy single backup script
3. Test basic functionality
4. Remove YubiKey/GPG complexity

### **Week 2: Optimize Storage**
1. Stop backing up media files
2. Update TrueNAS snapshot policies  
3. Clean up old complex backups
4. Verify cost reduction

### **Week 3: Validate**
1. Test file recovery from snapshots
2. Test VM restoration process
3. Document simple procedures
4. Set up basic monitoring

**Total setup time: ~8 hours spread over 3 weeks**
**Ongoing maintenance: ~10 minutes/month**

---

*This strategy prioritizes your time and sanity over theoretical perfect protection. For a home lab, 95% protection with 10% effort is better than 99.9% protection with 500% effort.*
