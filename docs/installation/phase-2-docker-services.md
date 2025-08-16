# Phase 2: Docker Services VM (Container Platform)

## Pre-Phase Checklist
- [ ] Phase 1 completed successfully (Proxmox + TrueNAS operational)
- [ ] TrueNAS storage pools accessible and healthy
- [ ] Ubuntu Pro account ready (free personal license)
- [ ] Phase 1 snapshot taken and verified

### Phase 2 Overview
**Goal**: Create containerized services VM with core infrastructure services
**Time Estimate**: 1-2 hours
**Risk Level**: Medium (no data loss risk)

---

## Step 1: Create Docker Services VM

### 1.1 Download Ubuntu Server ISO
1. In Proxmox: `local (pve)` → `ISO Images` → `Upload`
2. Upload Ubuntu Server 22.04 LTS ISO (or download directly)
3. Wait for upload completion

### 1.2 Create VM
1. **Create VM** in Proxmox
2. **General**:
   - VM ID: `200`
   - Name: `Docker-Services`
3. **OS**:
   - ISO: Ubuntu Server 22.04 LTS
   - Guest OS: Linux 6.x - 2.6 Kernel
4. **System**:
   - Machine: q35
   - BIOS: Default (SeaBIOS)
   - Qemu Agent: ✓ Enabled
5. **Disks**:
   - Bus/Device: SCSI 0
   - Disk size: 64 GB
   - Cache: Write back
   - Discard: ✓ Enabled
6. **CPU**:
   - Cores: 4
   - Type: host
7. **Memory**: 12288 MB (12GB)
8. **Network**: Default (vmbr0)

**✅ Step 1 Complete When**: VM created and ready to start

---

## Step 2: Ubuntu Server Installation

### 2.1 Install Ubuntu Server
1. Start Docker Services VM
2. Open Console
3. Follow Ubuntu installer:
   - **Language**: English
   - **Keyboard**: Your layout
   - **Network**: Use DHCP (we'll set static later)
   - **Proxy**: Leave blank
   - **Mirror**: Default
   - **Storage**: Use entire disk (guided)
   - **Profile Setup**:
     - Name: `Admin User`
     - Server name: `docker-services`
     - Username: `homelab`
     - Password: Strong password (write it down!)
   - **SSH Setup**: Install OpenSSH server ✓
   - **Snap packages**: Skip for now
4. Reboot when prompted

### 2.2 Post-Installation Setup
1. Login as `homelab` user
2. Update system:
```bash
sudo apt update && sudo apt upgrade -y
```

3. Install Qemu Guest Agent:
```bash
sudo apt install qemu-guest-agent -y
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

4. Set static IP:
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Edit to:
```yaml
network:
  ethernets:
    ens18:
      addresses:
        - 192.168.1.101/24
      nameservers:
        addresses: [192.168.1.1, 8.8.8.8]
      routes:
        - to: default
          via: 192.168.1.1
  version: 2
```

Apply changes:
```bash
sudo netplan apply
```

**✅ Step 2 Complete When**: Ubuntu accessible via SSH at static IP

---

## Step 3: Ubuntu Pro Setup

### 3.1 Enable Ubuntu Pro
1. Get your Ubuntu Pro token from https://ubuntu.com/pro
2. Attach the subscription:
```bash
sudo pro attach [YOUR-TOKEN]
```

3. Enable additional services:
```bash
sudo pro enable esm-infra
sudo pro enable esm-apps
```

4. Verify status:
```bash
sudo pro status
```

**✅ Step 3 Complete When**: Ubuntu Pro services enabled and active

---

## Step 4: Docker Installation

### 4.1 Install Docker
1. Install prerequisites:
```bash
sudo apt update
sudo apt install apt-transport-https ca-certificates curl software-properties-common -y
```

2. Add Docker GPG key:
```bash
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg
```

3. Add Docker repository:
```bash
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null
```

4. Install Docker:
```bash
sudo apt update
sudo apt install docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
```

5. Add user to docker group:
```bash
sudo usermod -aG docker homelab
newgrp docker
```

6. Enable Docker service:
```bash
sudo systemctl enable docker
sudo systemctl start docker
```

### 4.2 Verify Docker Installation
```bash
docker --version
docker compose version
docker run hello-world
```

**✅ Step 4 Complete When**: Docker commands work without sudo

---

## Step 5: Directory Structure Setup

### 5.1 Create Application Directories
```bash
# Create main application directory
mkdir -p /home/homelab/docker-apps

# Create individual service directories
mkdir -p /home/homelab/docker-apps/portainer
mkdir -p /home/homelab/docker-apps/adguard
mkdir -p /home/homelab/docker-apps/paperless
mkdir -p /home/homelab/docker-apps/monitoring

# Create shared directories
mkdir -p /home/homelab/docker-apps/shared/config
mkdir -p /home/homelab/docker-apps/shared/logs
```

### 5.2 Mount TrueNAS Storage
1. Install NFS client:
```bash
sudo apt install nfs-common -y
```

2. Create mount points:
```bash
sudo mkdir -p /mnt/truenas/documents
sudo mkdir -p /mnt/truenas/media
sudo mkdir -p /mnt/truenas/backups
```

3. Add to fstab for persistent mounting:
```bash
sudo nano /etc/fstab
```

Add lines (replace `192.168.1.100` with your TrueNAS IP):
```
192.168.1.100:/mnt/main-storage/documents /mnt/truenas/documents nfs defaults 0 0
192.168.1.100:/mnt/main-storage/media /mnt/truenas/media nfs defaults 0 0
192.168.1.100:/mnt/main-storage/backups /mnt/truenas/backups nfs defaults 0 0
```

4. Mount all:
```bash
sudo mount -a
```

**✅ Step 5 Complete When**: TrueNAS shares mounted and accessible

---

## Step 6: Deploy Portainer (Container Management)

### 6.1 Create Portainer Docker Compose
```bash
cd /home/homelab/docker-apps/portainer
nano docker-compose.yml
```

Create file:
```yaml
version: '3.8'

services:
  portainer:
    image: portainer/portainer-ce:latest
    container_name: portainer
    restart: unless-stopped
    security_opt:
      - no-new-privileges:true
    volumes:
      - /etc/localtime:/etc/localtime:ro
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./data:/data
    ports:
      - 9000:9000
    environment:
      - PUID=1000
      - PGID=1000
```

### 6.2 Deploy Portainer
```bash
docker compose up -d
```

### 6.3 Access Portainer Setup
1. Open browser: `http://192.168.1.101:9000`
2. Create admin account (username: `admin`)
3. Choose "Docker" environment
4. Connect to local Docker instance

**✅ Step 6 Complete When**: Portainer accessible and managing local Docker

---

## Step 7: Deploy AdGuard Home (Network-wide Ad Blocking)

### 7.1 Create AdGuard Docker Compose
```bash
cd /home/homelab/docker-apps/adguard
nano docker-compose.yml
```

Create file:
```yaml
version: '3.8'

services:
  adguardhome:
    image: adguard/adguardhome:latest
    container_name: adguardhome
    restart: unless-stopped
    ports:
      - 53:53/tcp
      - 53:53/udp
      - 3000:3000/tcp
    volumes:
      - ./work:/opt/adguardhome/work
      - ./conf:/opt/adguardhome/conf
    environment:
      - PUID=1000
      - PGID=1000
```

### 7.2 Deploy AdGuard Home
```bash
docker compose up -d
```

### 7.3 Configure AdGuard Home
1. Open browser: `http://192.168.1.101:3000`
2. Follow setup wizard:
   - **Admin interface**: Port 3000 (default)
   - **DNS server**: Port 53 (default)
   - **Admin user**: Create username/password
   - **DNS settings**: 
     - Upstream DNS: `8.8.8.8`, `1.1.1.1`
     - Enable DNS-over-HTTPS
3. Apply configuration

### 7.4 Test DNS Blocking
```bash
# Test from another machine or this VM
nslookup doubleclick.net 192.168.1.101
# Should return blocked IP (0.0.0.0)
```

**✅ Step 7 Complete When**: AdGuard Home blocking ads and accessible via web interface

---

## Step 8: Deploy Basic Monitoring (Uptime Kuma)

### 8.1 Create Monitoring Docker Compose
```bash
cd /home/homelab/docker-apps/monitoring
nano docker-compose.yml
```

Create file:
```yaml
version: '3.8'

services:
  uptime-kuma:
    image: louislam/uptime-kuma:latest
    container_name: uptime-kuma
    restart: unless-stopped
    ports:
      - 3001:3001
    volumes:
      - ./data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - PUID=1000
      - PGID=1000
```

### 8.2 Deploy Uptime Kuma
```bash
docker compose up -d
```

### 8.3 Configure Basic Monitoring
1. Open browser: `http://192.168.1.101:3001`
2. Create admin account
3. Add basic monitors:
   - **Proxmox**: `https://192.168.1.100:8006`
   - **TrueNAS**: `http://192.168.1.100`
   - **Portainer**: `http://192.168.1.101:9000`
   - **AdGuard**: `http://192.168.1.101:3000`

**✅ Step 8 Complete When**: Monitoring dashboard shows all services as UP

---

## Step 9: Deploy Paperless-ngx (Document Management)

### 9.1 Create Paperless Docker Compose
```bash
cd /home/homelab/docker-apps/paperless
nano docker-compose.yml
```

Create file:
```yaml
version: '3.8'

services:
  paperless-redis:
    image: docker.io/library/redis:7
    restart: unless-stopped
    volumes:
      - ./redis-data:/data

  paperless-db:
    image: docker.io/library/postgres:15
    restart: unless-stopped
    volumes:
      - ./pgdata:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: paperless_db_password

  paperless-webserver:
    image: paperlessngx/paperless-ngx:latest
    restart: unless-stopped
    depends_on:
      - paperless-db
      - paperless-redis
    ports:
      - 8010:8000
    volumes:
      - ./data:/usr/src/paperless/data
      - ./media:/usr/src/paperless/media
      - /mnt/truenas/documents:/usr/src/paperless/export
      - ./consume:/usr/src/paperless/consume
    environment:
      PAPERLESS_REDIS: redis://paperless-redis:6379
      PAPERLESS_DBHOST: paperless-db
      PAPERLESS_DBNAME: paperless
      PAPERLESS_DBUSER: paperless
      PAPERLESS_DBPASS: paperless_db_password
      PAPERLESS_SECRET_KEY: change-this-secret-key-in-production
      PAPERLESS_URL: http://192.168.1.101:8010
      PAPERLESS_OCR_LANGUAGE: eng
      PAPERLESS_TIME_ZONE: America/New_York
```

### 9.2 Deploy Paperless-ngx
```bash
docker compose up -d
```

### 9.3 Create Superuser
```bash
docker compose exec paperless-webserver python3 manage.py createsuperuser
```

### 9.4 Access Paperless-ngx
1. Open browser: `http://192.168.1.101:8010`
2. Login with superuser account
3. Configure basic settings:
   - Set up document types
   - Configure OCR settings
   - Test document upload

**✅ Step 9 Complete When**: Paperless-ngx accessible and can process documents

---

## Phase 2 Completion Criteria

### ✅ Must Be Working:
- [ ] Docker Services VM accessible via SSH and web interfaces
- [ ] Portainer managing Docker containers
- [ ] AdGuard Home blocking ads network-wide
- [ ] Uptime Kuma monitoring all Phase 1 & 2 services
- [ ] Paperless-ngx processing documents with OCR
- [ ] TrueNAS storage accessible from Docker Services VM
- [ ] All containers starting automatically on reboot

### 📊 Resource Usage Check:
- **Docker Services VM**: 12GB RAM allocated, ~6-8GB in use
- **Container Count**: 6 containers running
- **Storage Usage**: ~2GB local, connected to TrueNAS
- **Network Services**: DNS (port 53), Web interfaces (various ports)

### 🌐 Service Access Points:
- **Portainer**: `http://192.168.1.101:9000`
- **AdGuard Home**: `http://192.168.1.101:3000`
- **Uptime Kuma**: `http://192.168.1.101:3001`
- **Paperless-ngx**: `http://192.168.1.101:8010`

---

## Rollback Strategy

### If Docker Installation Fails:
```bash
# Remove Docker completely
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo rm -rf /var/lib/docker
# Restart from Step 4
```

### If Container Won't Start:
```bash
# Check logs
docker compose logs [service-name]
# Check system resources
free -h && df -h
# Restart individual service
docker compose restart [service-name]
```

### If Storage Mount Fails:
```bash
# Check TrueNAS NFS service status
# Verify network connectivity to TrueNAS
ping 192.168.1.100
# Remount manually
sudo umount /mnt/truenas/*
sudo mount -a
```

### Complete VM Reset:
1. Shutdown VM in Proxmox
2. Restore from Phase 1 snapshot (if taken)
3. Or delete VM and recreate from Step 1
4. TrueNAS data remains safe

---

## Pre-Phase 3 Snapshot

**Before proceeding to Phase 3:**

1. **Test All Services**:
   - Verify each service accessible via web interface
   - Test AdGuard DNS blocking from another device
   - Upload test document to Paperless-ngx
   - Check all containers in Portainer

2. **VM Snapshot**:
   - Docker Services VM → Snapshots → Take Snapshot
   - Name: `Phase2-Complete-All-Services-Working`

3. **Configuration Backup**:
   - Export Portainer configuration
   - Backup docker-compose files: `tar -czf docker-configs-backup.tar.gz /home/homelab/docker-apps/*/docker-compose.yml`
   - Save to TrueNAS backup share

4. **Router DNS Configuration** (Optional):
   - Point router DNS to `192.168.1.101` for network-wide ad blocking
   - Test DNS resolution from multiple devices

**Phase 2 is complete when all completion criteria are met and snapshots are taken.**

---

## Next: Phase 3 - Gaming & Media VMs
Ready to proceed when Phase 2 completion criteria are met and snapshots are taken.

Your Docker Services VM now provides the foundation for container management, network-wide ad blocking, system monitoring, and document management - all essential services for the home lab infrastructure.
