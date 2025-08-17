# Phase 4: Security & Monitoring Implementation

## Pre-Phase 4 Checklist
- Cloudflare tunnels working for all services
- Basic monitoring via Uptime Kuma operational

**Goal**: Implement comprehensive monitoring, automate your backup strategy, and validate security measures

**Time Estimate**: 2-3 hours total

---

## Part A: Backup Implementation (45 minutes)

Your backup strategy is excellent - let's implement it on the TrueNAS VM.

### A1: OVH Cloud Storage Setup

```bash
# SSH into TrueNAS VM
ssh admin@10.0.0.110

# Install required packages
sudo apt update
sudo apt install -y rclone curl jq

# Configure rclone for OVH Object Storage
rclone config
# Choose: n (new remote)
# Name: ovh-backup
# Storage: 4 (Amazon S3 Compliant)
# Provider: 15 (Other)
# Access Key ID: [Your OVH access key]
# Secret Access Key: [Your OVH secret key]
# Region: ca-east-1 (for Beauharnois, Quebec)
# Endpoint: s3.ca-east-1.cloud.ovh.net
# Advanced config: No

# Test connection
rclone lsd ovh-backup:
```

### A2: Create Backup Directory Structure

```bash
# Create backup staging areas
sudo mkdir -p /mnt/main-storage/backups/{staging,configs,vm-exports}
sudo chown -R homelab:homelab /mnt/main-storage/backups/

# Create scripts directory
sudo mkdir -p /opt/scripts
sudo chown homelab:homelab /opt/scripts
```

### A3: Simple Backup Script Implementation

```bash
# Create the main backup script
cat > /opt/scripts/simple-backup.sh << 'EOF'
#!/bin/bash

# Simple Home Lab Backup Script
# Implements 3-2-1 strategy for critical data only

BACKUP_DIR="/mnt/main-storage/backups"
STAGING_DIR="$BACKUP_DIR/staging"
LOG_FILE="/var/log/homelab-backup.log"
DATE=$(date +%Y%m%d_%H%M%S)

# Logging function
log() {
   echo "[$(date '+%Y-%m-%d %H:%M:%S')] $1" | tee -a $LOG_FILE
}

log "=== Starting Home Lab Backup - $DATE ==="

# 1. Export system configurations
log "Exporting system configurations..."
mkdir -p $STAGING_DIR/configs/$DATE

# TrueNAS config
sudo cp /data/freenas-v1.db $STAGING_DIR/configs/$DATE/truenas-config.db 2>/dev/null || {
   # For TrueNAS SCALE
   sudo cp /etc/systemd/system/truenas-config.db $STAGING_DIR/configs/$DATE/ 2>/dev/null || 
   log "Warning: Could not backup TrueNAS config"
}

# Network and system configs
sudo cp /etc/netplan/* $STAGING_DIR/configs/$DATE/ 2>/dev/null
sudo cp /etc/fstab $STAGING_DIR/configs/$DATE/
sudo cp /etc/hosts $STAGING_DIR/configs/$DATE/

# 2. Sync documents from Paperless-ngx
log "Syncing documents..."
if [ -d "/mnt/main-storage/documents" ]; then
   rsync -av --delete /mnt/main-storage/documents/ $STAGING_DIR/documents/
   log "Documents synced successfully"
else
   log "Warning: Documents directory not found"
fi

# 3. Sync music library (with incremental support)
log "Syncing music library..."
if [ -d "/mnt/main-storage/media/music" ]; then
   rsync -av --delete /mnt/main-storage/media/music/ $STAGING_DIR/music/
   log "Music library synced successfully"
else
   log "Warning: Music directory not found"
fi

# 4. Create backup manifest
log "Creating backup manifest..."
cat > $STAGING_DIR/backup-manifest.txt << MANIFEST
Backup Date: $DATE
System: tehzombijesus.ca Home Lab
Contents:
- System configurations
- Documents from Paperless-ngx
- Music library
- VM configurations (manual)

Excludes:
- Movies, TV shows, books (replaceable content)
- Media automation downloads
- Temporary files

Backup Type: Weekly staging for cloud upload
MANIFEST

# 5. Calculate staging size
STAGING_SIZE=$(du -sh $STAGING_DIR | cut -f1)
log "Staging area size: $STAGING_SIZE"

# 6. Upload to OVH (weekly on Sundays)
if [ $(date +%u) -eq 7 ]; then
   log "Sunday detected - uploading to OVH cloud storage..."
   
   # Upload with date prefix
   rclone sync $STAGING_DIR ovh-backup:homelab-backups/$DATE --progress --log-level INFO --log-file $LOG_FILE.rclone
   
   if [ $? -eq 0 ]; then
       log "Cloud upload successful"
       # Clean up old staging files (keep last 2 weeks)
       find $STAGING_DIR -type d -name "configs" -exec find {} -type d -mtime +14 -exec rm -rf {} + \;
   else
       log "ERROR: Cloud upload failed"
   fi
else
   log "Not Sunday - skipping cloud upload"
fi

# 7. Health check and cleanup
log "Running backup health check..."

# Check if critical directories exist
HEALTH_STATUS="OK"
[ ! -d "$STAGING_DIR/documents" ] && { log "ERROR: Documents backup missing"; HEALTH_STATUS="ERROR"; }
[ ! -d "$STAGING_DIR/music" ] && { log "ERROR: Music backup missing"; HEALTH_STATUS="ERROR"; }
[ ! -d "$STAGING_DIR/configs" ] && { log "ERROR: Config backup missing"; HEALTH_STATUS="ERROR"; }

# Create health status file for monitoring
echo "$HEALTH_STATUS" > /tmp/backup-status
echo "$DATE" > /tmp/backup-last-run

log "Backup completed with status: $HEALTH_STATUS"
log "=== End Backup - $DATE ==="
EOF

# Make script executable
chmod +x /opt/scripts/simple-backup.sh
```

### A4: Setup Backup Scheduling

```bash
# Create backup health check script
cat > /opt/scripts/backup-health.sh << 'EOF'
#!/bin/bash

# Simple health check for monitoring integration
LAST_RUN=$(cat /tmp/backup-last-run 2>/dev/null || echo "never")
STATUS=$(cat /tmp/backup-status 2>/dev/null || echo "unknown")

if [ "$STATUS" = "OK" ]; then
   echo "Backup healthy - last run: $LAST_RUN"
   exit 0
else
   echo "Backup failed - status: $STATUS, last run: $LAST_RUN"
   exit 1
fi
EOF

chmod +x /opt/scripts/backup-health.sh

# Add to crontab
(crontab -l 2>/dev/null; cat << CRON
# Home Lab Backup Schedule
# Weekly backup (Sunday 2 AM)
0 2 * * 0 /opt/scripts/simple-backup.sh

# Daily health check (6 AM)
0 6 * * * /opt/scripts/backup-health.sh
CRON
) | crontab -

# Test the backup script
log "Testing backup script..."
/opt/scripts/simple-backup.sh
```

---

## Part B: Enhanced Monitoring Stack (1 hour)

Let's add comprehensive system monitoring to complement your Uptime Kuma.

### B1: Deploy Monitoring Stack via Portainer

SSH into your Docker Services VM (10.0.0.101) and add this to Portainer:

**Stack Name**: `enhanced-monitoring`

```yaml
version: '3.8'

networks:
 monitoring:
   driver: bridge

services:
 # Prometheus for metrics collection
 prometheus:
   image: prom/prometheus:latest
   container_name: prometheus
   restart: unless-stopped
   ports:
     - 9090:9090
   volumes:
     - prometheus_data:/prometheus
     - ./prometheus.yml:/etc/prometheus/prometheus.yml:ro
   command:
     - '--config.file=/etc/prometheus/prometheus.yml'
     - '--storage.tsdb.path=/prometheus'
     - '--web.console.libraries=/etc/prometheus/console_libraries'
     - '--web.console.templates=/etc/prometheus/consoles'
     - '--web.enable-lifecycle'
   networks:
     - monitoring

 # Grafana for visualization
 grafana:
   image: grafana/grafana:latest
   container_name: grafana
   restart: unless-stopped
   ports:
     - 3002:3000
   volumes:
     - grafana_data:/var/lib/grafana
   environment:
     - GF_SECURITY_ADMIN_PASSWORD=your_secure_password
     - GF_USERS_ALLOW_SIGN_UP=false
   networks:
     - monitoring

 # Node Exporter for system metrics
 node-exporter:
   image: prom/node-exporter:latest
   container_name: node-exporter
   restart: unless-stopped
   ports:
     - 9100:9100
   volumes:
     - /proc:/host/proc:ro
     - /sys:/host/sys:ro
     - /:/rootfs:ro
   command:
     - '--path.procfs=/host/proc'
     - '--path.rootfs=/rootfs'
     - '--path.sysfs=/host/sys'
     - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
   networks:
     - monitoring

 # cAdvisor for container metrics
 cadvisor:
   image: gcr.io/cadvisor/cadvisor:latest
   container_name: cadvisor
   restart: unless-stopped
   ports:
     - 8080:8080
   volumes:
     - /:/rootfs:ro
     - /var/run:/var/run:rw
     - /sys:/sys:ro
     - /var/lib/docker/:/var/lib/docker:ro
     - /dev/disk/:/dev/disk:ro
   privileged: true
   devices:
     - /dev/kmsg
   networks:
     - monitoring

volumes:
 prometheus_data:
 grafana_data:
```

### B2: Create Prometheus Configuration

```bash
# Create prometheus.yml in Docker Services VM
mkdir -p /opt/monitoring/prometheus
cat > /opt/monitoring/prometheus/prometheus.yml << 'EOF'
global:
 scrape_interval: 15s
 evaluation_interval: 15s

scrape_configs:
 # Docker Services VM (this machine)
 - job_name: 'docker-services-node'
   static_configs:
     - targets: ['node-exporter:9100']
   scrape_interval: 5s

 - job_name: 'docker-containers'
   static_configs:
     - targets: ['cadvisor:8080']
   scrape_interval: 5s

 # Gaming VM monitoring
 - job_name: 'gaming-vm'
   static_configs:
     - targets: ['10.0.0.102:9100']  # Pterodactyl VM
   scrape_interval: 15s

 # Media VMs monitoring  
 - job_name: 'plex-vm'
   static_configs:
     - targets: ['10.0.0.104:9100']  # Plex VM
   scrape_interval: 15s

 - job_name: 'media-automation-vm'
   static_configs:
     - targets: ['10.0.0.105:9100']  # Media Automation VM
   scrape_interval: 15s

 # TrueNAS monitoring
 - job_name: 'truenas-vm'
   static_configs:
     - targets: ['10.0.0.110:9100']  # TrueNAS VM
   scrape_interval: 15s

 # Backup monitoring
 - job_name: 'backup-health'
   static_configs:
     - targets: ['10.0.0.110:9101']  # Custom backup exporter
   scrape_interval: 60s
EOF
```

### B3: Install Node Exporters on All VMs

Create a quick script to deploy node exporters:

```bash
# Create node exporter installer script
cat > /opt/scripts/install-node-exporters.sh << 'EOF'
#!/bin/bash

VMS=("10.0.0.102" "10.0.0.104" "10.0.0.105" "10.0.0.110")
VM_NAMES=("Gaming-Pterodactyl" "Media-Plex" "Media-Automation" "TrueNAS-Storage")

for i in "${!VMS[@]}"; do
   VM_IP="${VMS[$i]}"
   VM_NAME="${VM_NAMES[$i]}"
   
   echo "Installing Node Exporter on $VM_NAME ($VM_IP)..."
   
   ssh homelab@$VM_IP << 'REMOTE'
       # Download and install node exporter
       wget -q https://github.com/prometheus/node_exporter/releases/download/v1.6.1/node_exporter-1.6.1.linux-amd64.tar.gz
       tar xzf node_exporter-1.6.1.linux-amd64.tar.gz
       sudo mv node_exporter-1.6.1.linux-amd64/node_exporter /usr/local/bin/
       rm -rf node_exporter-1.6.1.linux-amd64*
       
       # Create systemd service
       sudo tee /etc/systemd/system/node_exporter.service << SERVICE
[Unit]
Description=Node Exporter
After=network.target

[Service]
User=nobody
Group=nobody
Type=simple
ExecStart=/usr/local/bin/node_exporter --web.listen-address=:9100

[Install]
WantedBy=multi-user.target
SERVICE

       # Start and enable service
       sudo systemctl daemon-reload
       sudo systemctl enable node_exporter
       sudo systemctl start node_exporter
       
       # Update firewall to allow Prometheus scraping
       sudo ufw allow from 10.0.0.101 to any port 9100
       
       echo "Node Exporter installed and started"
REMOTE

   if [ $? -eq 0 ]; then
       echo " $VM_NAME setup failed"
   fi
done
EOF

chmod +x /opt/scripts/install-node-exporters.sh
/opt/scripts/install-node-exporters.sh
```

### B4: Deploy Enhanced Monitoring Stack

```bash
# Update the Portainer stack with the prometheus.yml volume mount
# In Portainer, update the enhanced-monitoring stack to include:

version: '3.8'

networks:
 monitoring:
   driver: bridge

services:
 prometheus:
   image: prom/prometheus:latest
   container_name: prometheus
   restart: unless-stopped
   ports:
     - 9090:9090
   volumes:
     - prometheus_data:/prometheus
     - /opt/monitoring/prometheus/prometheus.yml:/etc/prometheus/prometheus.yml:ro
   command:
     - '--config.file=/etc/prometheus/prometheus.yml'
     - '--storage.tsdb.path=/prometheus'
     - '--web.console.libraries=/etc/prometheus/console_libraries'
     - '--web.console.templates=/etc/prometheus/consoles'
     - '--web.enable-lifecycle'
   networks:
     - monitoring

 grafana:
   image: grafana/grafana:latest
   container_name: grafana
   restart: unless-stopped
   ports:
     - 3002:3000
   volumes:
     - grafana_data:/var/lib/grafana
   environment:
     - GF_SECURITY_ADMIN_PASSWORD=your_secure_grafana_password
     - GF_USERS_ALLOW_SIGN_UP=false
     - GF_SERVER_ROOT_URL=http://monitoring.tehzombijesus.ca
   networks:
     - monitoring

 node-exporter:
   image: prom/node-exporter:latest
   container_name: node-exporter
   restart: unless-stopped
   ports:
     - 9100:9100
   volumes:
     - /proc:/host/proc:ro
     - /sys:/host/sys:ro
     - /:/rootfs:ro
   command:
     - '--path.procfs=/host/proc'
     - '--path.rootfs=/rootfs'  
     - '--path.sysfs=/host/sys'
     - '--collector.filesystem.mount-points-exclude=^/(sys|proc|dev|host|etc)($$|/)'
   networks:
     - monitoring

 cadvisor:
   image: gcr.io/cadvisor/cadvisor:latest
   container_name: cadvisor
   restart: unless-stopped
   ports:
     - 8080:8080
   volumes:
     - /:/rootfs:ro
     - /var/run:/var/run:rw
     - /sys:/sys:ro
     - /var/lib/docker/:/var/lib/docker:ro
     - /dev/disk/:/dev/disk:ro
   privileged: true
   networks:
     - monitoring

volumes:
 prometheus_data:
 grafana_data:
```

### B5: Create Backup Health Exporter

```bash
# SSH to TrueNAS VM and create backup health exporter
ssh homelab@10.0.0.110

cat > /opt/scripts/backup-exporter.py << 'EOF'
#!/usr/bin/env python3

import http.server
import socketserver
import os
from datetime import datetime, timedelta

class BackupHealthHandler(http.server.BaseHTTPRequestHandler):
   def do_GET(self):
       if self.path == '/metrics':
           self.send_response(200)
           self.send_header('Content-type', 'text/plain')
           self.end_headers()
           
           metrics = []
           
           # Check backup status
           try:
               with open('/tmp/backup-status', 'r') as f:
                   status = f.read().strip()
                   backup_healthy = 1 if status == 'OK' else 0
           except:
               backup_healthy = 0
           
           # Check last backup time
           try:
               with open('/tmp/backup-last-run', 'r') as f:
                   last_run = f.read().strip()
                   # Parse date and calculate hours since backup
                   backup_time = datetime.strptime(last_run, '%Y%m%d_%H%M%S')
                   hours_since = (datetime.now() - backup_time).total_seconds() / 3600
                   backup_age_hours = int(hours_since)
           except:
               backup_age_hours = 999
           
           # Check staging directory size
           try:
               staging_size = sum(
                   os.path.getsize(os.path.join(dirpath, filename))
                   for dirpath, dirnames, filenames in os.walk('/mnt/main-storage/backups/staging')
                   for filename in filenames
               )
               staging_size_gb = staging_size / (1024**3)
           except:
               staging_size_gb = 0
           
           metrics.append(f'backup_healthy {backup_healthy}')
           metrics.append(f'backup_age_hours {backup_age_hours}')
           metrics.append(f'backup_staging_size_gb {staging_size_gb:.2f}')
           
           self.wfile.write('\n'.join(metrics).encode())
       else:
           self.send_response(404)
           self.end_headers()

PORT = 9101
with socketserver.TCPServer(("", PORT), BackupHealthHandler) as httpd:
   httpd.serve_forever()
EOF

# Create systemd service for backup exporter
sudo tee /etc/systemd/system/backup-exporter.service << 'EOF'
[Unit]
Description=Backup Health Exporter
After=network.target

[Service]
User=homelab
Group=homelab
Type=simple
ExecStart=/usr/bin/python3 /opt/scripts/backup-exporter.py
Restart=always

[Install]
WantedBy=multi-user.target
EOF

sudo systemctl daemon-reload
sudo systemctl enable backup-exporter
sudo systemctl start backup-exporter

# Allow Prometheus access
sudo ufw allow from 10.0.0.101 to any port 9101
```

---

## Part C: Integrate with Uptime Kuma (30 minutes)

### C1: Add New Monitoring Endpoints

Access Uptime Kuma at `http://10.0.0.101:3001` and add these monitors:

**System Health Monitors:**
- **Prometheus**: `http://10.0.0.101:9090` (HTTP)
- **Grafana**: `http://10.0.0.101:3002` (HTTP) 
- **Gaming VM Health**: `http://10.0.0.102:9100/metrics` (HTTP)
- **Plex VM Health**: `http://10.0.0.104:9100/metrics` (HTTP)
- **Media Automation Health**: `http://10.0.0.105:9100/metrics` (HTTP)
- **TrueNAS Health**: `http://10.0.0.110:9100/metrics` (HTTP)
- **Backup Health**: `http://10.0.0.110:9101/metrics` (HTTP - Custom keyword: "backup_healthy 1")

**Service-Specific Monitors:**
- **CrowdSec Status**: `http://10.0.0.102:8080` (Gaming VM - if CrowdSec has web interface)
- **MariaDB Health**: Ping monitor to 10.0.0.102:3306
- **Docker Health**: Monitor Docker socket on each VM

### C2: Configure Alerting

In Uptime Kuma:
1. **Settings** **Networks** **Access** Backup Implementation
- [ ] Simple backup script operational on TrueNAS VM
- [ ] OVH cloud storage configured and tested
- [ ] Weekly backup schedule active (cron)
- [ ] Backup health monitoring integrated
- [ ] 3-2-1 strategy implemented (excluding media files)

### Integration & Access
- [ ] Uptime Kuma monitoring all critical services
- [ ] Monitoring services accessible via monitoring.tehzombijesus.ca
- [ ] Zero Trust authentication protecting monitoring interfaces
- [ ] Alerting configured for critical failures

### Documentation & Maintenance
- [ ] Backup procedures documented and tested
- [ ] Monitoring procedures established
- [ ] Disaster recovery plan validated
- [ ] Update procedures documented for each service type

---

## Service Access Summary

**External Access (via Cloudflare + Zero Trust):**
- **Gaming**: `https://games.tehzombijesus.ca`
- **Media**: `https://plex.tehzombijesus.ca`
- **Documents**: `https://docs.tehzombijesus.ca`
- **Monitoring**: `https://monitoring.tehzombijesus.ca`
- **Status**: `https://status.tehzombijesus.ca`

**Internal Access (via local network):**
- **Proxmox**: `https://10.0.0.100:8006`
- **TrueNAS**: `http://10.0.0.110`
- **Portainer**: `http://10.0.0.101:9000`
- **AdGuard**: `http://10.0.0.101:3000`
- **Uptime Kuma**: `http://10.0.0.101:3001`

---

## Ongoing Maintenance

### Weekly (10 minutes)
- Check Uptime Kuma for any service alerts
- Verify backup completed successfully
- Review system resource usage in Grafana

### Monthly (30 minutes)
- Update containers via Portainer
- Review security logs and CrowdSec reports
- Test disaster recovery procedure (small scale)
- Verify backup retention and cleanup

### Quarterly (2 hours)
- Ubuntu Pro updates across all VMs
- Review and optimize resource allocation
- Test full disaster recovery scenario
- Security audit and access review

**Congratulations! Your home lab now has enterprise-grade monitoring, automated backups, and comprehensive security - all manageable through web interfaces with minimal maintenance overhead.**
