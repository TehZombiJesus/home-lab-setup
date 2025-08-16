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

## Step 2: Ubuntu Server Installation with Security Partitioning

### 2.1 Install Ubuntu Server
1. Start Docker Services VM
2. Open Console
3. Follow Ubuntu installer:
   - **Language**: English
   - **Keyboard**: Your layout
   - **Network**: Use DHCP (we'll set static later)
   - **Proxy**: Leave blank
   - **Mirror**: Default

### 2.2 Custom Partitioning for Security
1. **Storage Configuration**: Choose "Custom storage layout"
2. **Create Security Partitions**:

   **Boot Partition**:
   - Size: `512M`
   - Format: `ext2`
   - Mount: `/boot`

   **Root Partition**:
   - Size: `8G`
   - Format: `ext4`
   - Mount: `/`

   **Var Partition** (Docker data):
   - Size: `16G`
   - Format: `XFS`
   - Mount: `/var`

   **Var Log Partition**:
   - Size: `4G`
   - Format: `ext4`
   - Mount: `/var/log`

   **Home Partition**:
   - Size: `8G`
   - Format: `ext4`
   - Mount: `/home`

   **Opt Partition** (Applications):
   - Size: `8G`
   - Format: `XFS`
   - Mount: `/opt`

   **Swap**:
   - Size: `4G`
   - Format: `swap`

   **Remaining Space**: Leave unallocated for future expansion

### 2.3 Complete Installation
1. **Profile Setup**:
   - Name: `Admin User`
   - Server name: `docker-services`
   - Username: `homelab`
   - Password: Strong password (write it down!)
2. **SSH Setup**: Install OpenSSH server ✓
3. **Snap packages**: Skip for now
4. Reboot when prompted

**✅ Step 2 Complete When**: Ubuntu boots successfully with custom partitions

---

## Step 3: Post-Installation Security Configuration

### 3.1 Apply Security Mount Options
1. Login as `homelab` user
2. Edit fstab for security mount options:
```bash
sudo nano /etc/fstab
```

3. Update mount options (add to existing entries):
```bash
# /boot - read-only after setup
UUID=xxx /boot ext2 defaults,ro,nodev,nosuid,noexec 0 2

# /var - no execution for Docker security
UUID=xxx /var xfs defaults,nodev,nosuid,noexec 0 2

# /var/log - strict security for logs
UUID=xxx /var/log ext4 defaults,nodev,nosuid,noexec 0 2

# /home - no devices
UUID=xxx /home ext4 defaults,nodev 0 2

# /opt - no devices for applications
UUID=xxx /opt xfs defaults,nodev 0 2
```

4. Create secure tmpfs for /tmp:
```bash
echo "tmpfs /tmp tmpfs defaults,nodev,nosuid,noexec,size=2G 0 0" | sudo tee -a /etc/fstab
```

5. Remount with new options:
```bash
sudo mount -o remount /var
sudo mount -o remount /var/log
sudo mount -o remount /home
sudo mount -o remount /opt
sudo mount tmpfs /tmp
```

### 3.2 System Updates and Basic Setup
1. Update system:
```bash
sudo apt update && sudo apt upgrade -y
```

2. Install essential packages:
```bash
sudo apt install qemu-guest-agent nfs-common curl wget git -y
sudo systemctl enable qemu-guest-agent
sudo systemctl start qemu-guest-agent
```

### 3.3 Configure Static IP with Proper Netplan
```bash
sudo nano /etc/netplan/00-installer-config.yaml
```

Create proper netplan configuration:
```yaml
network:
  renderer: networkd
  ethernets:
    ens18:
      addresses:
        - 10.0.0.101/24
      nameservers:
        addresses: [9.9.9.9]
      routes:
        - to: default
          via: 10.0.0.1
          on-link: true
  version: 2
```

Apply network changes:
```bash
sudo netplan apply
```

**✅ Step 3 Complete When**: Ubuntu accessible via SSH at static IP with security partitions active

---

## Step 4: Ubuntu Pro Setup

### 4.1 Enable Ubuntu Pro
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

**✅ Step 4 Complete When**: Ubuntu Pro services enabled and active

---

## Step 5: Docker Installation

### 5.1 Install Docker
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

5. Configure Docker for security:
```bash
# Add user to docker group
sudo usermod -aG docker homelab
newgrp docker

# Configure Docker daemon for security
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json > /dev/null <<EOF
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false,
  "no-new-privileges": true
}
EOF
```

6. Enable Docker service:
```bash
sudo systemctl enable docker
sudo systemctl start docker
sudo systemctl restart docker
```

### 5.2 Verify Docker Installation
```bash
docker --version
docker compose version
docker run hello-world
```

**✅ Step 5 Complete When**: Docker commands work without sudo

---

## Step 6: TrueNAS Storage Mount Setup

### 6.1 Create Mount Points and Connect Storage
```bash
# Create mount points for TrueNAS NFS shares
sudo mkdir -p /mnt/truenas/documents
sudo mkdir -p /mnt/truenas/media
sudo mkdir -p /mnt/truenas/backups

# Test NFS connectivity
showmount -e 10.0.0.110
```

### 6.2 Configure Persistent NFS Mounts
```bash
sudo nano /etc/fstab
```

Add NFS mount entries:
```bash
# TrueNAS NFS Shares
10.0.0.110:/mnt/main-storage/documents /mnt/truenas/documents nfs defaults,nodev 0 0
10.0.0.110:/mnt/main-storage/media /mnt/truenas/media nfs defaults,nodev 0 0
10.0.0.110:/mnt/main-storage/backups /mnt/truenas/backups nfs defaults,nodev 0 0
```

Mount all shares:
```bash
sudo mount -a
```

Verify access:
```bash
ls -la /mnt/truenas/
df -h | grep truenas
```

**✅ Step 6 Complete When**: TrueNAS shares mounted and accessible

---

## Step 7: Deploy Portainer (Container Management)

### 7.1 Deploy Portainer via Command Line
```bash
# Create Portainer volume
docker volume create portainer_data

# Deploy Portainer
docker run -d -p 9000:9000 -p 9443:9443 \
  --name portainer --restart=always \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v portainer_data:/data \
  portainer/portainer-ce:latest
```

### 7.2 Access Portainer Setup
1. Open browser: `http://10.0.0.101:9000`
2. Create admin account:
   - **Username**: `admin`
   - **Password**: Strong password (write it down!)
3. **Environment Setup**: Choose "Docker" → "Connect"
4. Verify local Docker environment is connected

### 7.3 Configure Portainer for Management
1. **Settings** → **Authentication** → Configure session timeout
2. **Users** → Create additional users if needed
3. **Environments** → Verify Docker environment is healthy

**✅ Step 7 Complete When**: Portainer accessible and managing local Docker

---

## Step 8: Deploy AdGuard Home via Portainer

### 8.1 Create AdGuard Stack in Portainer
1. In Portainer: **Stacks** → **Add Stack**
2. **Name**: `adguard-home`
3. **docker-compose.yml**:

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
      - adguard_work:/opt/adguardhome/work
      - adguard_conf:/opt/adguardhome/conf
    environment:
      - PUID=1000
      - PGID=1000

volumes:
  adguard_work:
  adguard_conf:
```

4. **Deploy the stack**

### 8.2 Configure AdGuard Home
1. Open browser: `http://10.0.0.101:3000`
2. Setup wizard:
   - **Admin interface**: Port 3000
   - **DNS server**: Port 53  
   - **Admin credentials**: Create username/password
   - **Upstream DNS**: `9.9.9.9`, `149.112.112.112` (Quad9 primary/secondary)
   - **Enable DNS-over-HTTPS**: ✓ Enabled

3. **Filters** → **DNS blocklists** → Add recommended lists:
   - AdGuard DNS filter
   - EasyList
   - Malware Domains

### 8.3 Test DNS Blocking
```bash
# Test from this VM
nslookup doubleclick.net 10.0.0.101
# Should return blocked result

# Test regular domain resolution  
nslookup google.com 10.0.0.101
# Should resolve normally
```

**✅ Step 8 Complete When**: AdGuard Home blocking ads and resolving DNS properly

---

## Step 9: Deploy Monitoring (Uptime Kuma) via Portainer

### 9.1 Create Uptime Kuma Stack
1. **Portainer** → **Stacks** → **Add Stack** 
2. **Name**: `monitoring`
3. **docker-compose.yml**:

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
      - uptime_data:/app/data
      - /var/run/docker.sock:/var/run/docker.sock:ro
    environment:
      - PUID=1000
      - PGID=1000

volumes:
  uptime_data:
```

4. **Deploy the stack**

### 9.2 Configure Basic Monitoring
1. Open browser: `http://10.0.0.101:3001`
2. Create admin account
3. **Add Monitors**:
   - **Proxmox**: `https://10.0.0.100:8006` (ignore SSL errors)
   - **TrueNAS**: `http://10.0.0.110`
   - **Portainer**: `http://10.0.0.101:9000`
   - **AdGuard**: `http://10.0.0.101:3000`
   - **This VM SSH**: `10.0.0.101:22`

4. **Notification Setup**: Configure email/Discord webhooks as desired

**✅ Step 9 Complete When**: All Phase 1 & 2 services showing as UP in monitoring

---

## Step 10: Deploy Paperless-ngx via Portainer

### 10.1 Create Paperless Stack
1. **Portainer** → **Stacks** → **Add Stack**
2. **Name**: `paperless-ngx`
3. **docker-compose.yml**:

```yaml
version: '3.8'

services:
  paperless-redis:
    image: docker.io/library/redis:7
    container_name: paperless-redis
    restart: unless-stopped
    volumes:
      - redis_data:/data

  paperless-db:
    image: docker.io/library/postgres:15
    container_name: paperless-db
    restart: unless-stopped
    volumes:
      - postgres_data:/var/lib/postgresql/data
    environment:
      POSTGRES_DB: paperless
      POSTGRES_USER: paperless
      POSTGRES_PASSWORD: ${POSTGRES_PASSWORD}

  paperless-webserver:
    image: paperlessngx/paperless-ngx:latest
    container_name: paperless-webserver
    restart: unless-stopped
    depends_on:
      - paperless-db
      - paperless-redis
    ports:
      - 8010:8000
    volumes:
      - paperless_data:/usr/src/paperless/data
      - paperless_media:/usr/src/paperless/media
      - /mnt/truenas/documents:/usr/src/paperless/export
      - paperless_consume:/usr/src/paperless/consume
    environment:
      PAPERLESS_REDIS: redis://paperless-redis:6379
      PAPERLESS_DBHOST: paperless-db
      PAPERLESS_DBNAME: paperless
      PAPERLESS_DBUSER: paperless
      PAPERLESS_DBPASS: ${POSTGRES_PASSWORD}
      PAPERLESS_SECRET_KEY: ${SECRET_KEY}
      PAPERLESS_URL: http://10.0.0.101:8010
      PAPERLESS_OCR_LANGUAGE: eng
      PAPERLESS_TIME_ZONE: America/New_York
      PAPERLESS_CONSUMER_POLLING: 30
      PAPERLESS_TASK_WORKERS: 2
      # Security settings for sensitive documents
      PAPERLESS_FORCE_SCRIPT_NAME: /
      PAPERLESS_STATIC_URL: /static/
      PAPERLESS_AUTO_LOGIN_USERNAME: ""

volumes:
  redis_data:
  postgres_data:
  paperless_data:
  paperless_media:
  paperless_consume:
```

4. **Environment Variables** (in Portainer stack deployment):
```
POSTGRES_PASSWORD=your_secure_db_password
SECRET_KEY=your_very_long_secret_key_here
```

5. **Deploy the stack**

### 10.2 Create Paperless Superuser
```bash
# Via Portainer console or SSH
docker exec -it paperless-webserver python3 manage.py createsuperuser
```

### 10.3 Configure Paperless for Sensitive Documents
1. Open browser: `http://10.0.0.101:8010`
2. Login with superuser account
3. **Settings** → **Document Types** → Create categories:
   - Tax Documents
   - Financial Records  
   - Personal IDs
   - Insurance Documents

4. **Settings** → **Storage Paths** → Configure:
   - Set up organized folder structure
   - Enable document versioning
   - Configure retention policies

**✅ Step 10 Complete When**: Paperless-ngx processing documents with proper categorization

---

## Step 11: Phase 2 Security Hardening

### 11.1 SSH Security Configuration
```bash
sudo nano /etc/ssh/sshd_config
```

Apply security hardening:
```bash
# Change default port
Port 2222

# Disable root login
PermitRootLogin no

# Key-only authentication
PasswordAuthentication no
PubkeyAuthentication yes
AuthorizedKeysFile .ssh/authorized_keys

# Security settings
Protocol 2
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
```

Restart SSH:
```bash
sudo systemctl restart sshd
```

### 11.2 Docker Security Hardening
```bash
# Configure Docker log rotation
sudo nano /etc/docker/daemon.json
```

Update Docker daemon config:
```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "userland-proxy": false,
  "no-new-privileges": true,
  "icc": false,
  "default-ulimits": {
    "nofile": {
      "soft": 65536,
      "hard": 65536
    }
  }
}
```

Restart Docker:
```bash
sudo systemctl restart docker
```

### 11.3 Firewall Configuration
```bash
# Install and configure UFW
sudo apt install ufw -y

# Default policies
sudo ufw default deny incoming
sudo ufw default allow outgoing

# Allow essential services
sudo ufw allow 2222/tcp  # SSH
sudo ufw allow 53/tcp    # DNS
sudo ufw allow 53/udp    # DNS
sudo ufw allow 9000/tcp  # Portainer
sudo ufw allow 3000/tcp  # AdGuard
sudo ufw allow 3001/tcp  # Uptime Kuma
sudo ufw allow 8010/tcp  # Paperless

# Allow from local network only
sudo ufw allow from 10.0.0.0/24

# Enable firewall
sudo ufw --force enable
```

### 11.4 System Monitoring Setup
```bash
# Configure system limits
echo "homelab soft nofile 65536" | sudo tee -a /etc/security/limits.conf
echo "homelab hard nofile 65536" | sudo tee -a /etc/security/limits.conf

# Enable audit logging
sudo apt install auditd -y
sudo systemctl enable auditd
```

**✅ Step 11 Complete When**: All security measures applied and services still accessible

---

## Phase 2 Completion Criteria

### ✅ Must Be Working:
- [ ] Docker Services VM accessible via SSH (port 2222) and web interfaces
- [ ] Security partitions mounted with proper noexec/nosuid options  
- [ ] Portainer managing Docker containers via web interface
- [ ] AdGuard Home blocking ads network-wide on port 53
- [ ] Uptime Kuma monitoring all Phase 1 & 2 services
- [ ] Paperless-ngx processing documents with OCR capabilities
- [ ] TrueNAS storage accessible via NFS mounts
- [ ] All containers starting automatically on reboot
- [ ] Firewall configured and active

### 📊 Resource Usage Check:
- **Docker Services VM**: 12GB RAM allocated, ~6-8GB in use
- **Container Count**: 6 containers running (Portainer + AdGuard + Uptime Kuma + Paperless stack)
- **Storage Usage**: ~2GB local, connected to TrueNAS via NFS
- **Partition Security**: All security mount options active

### 🌐 Service Access Points:
- **Portainer**: `http://10.0.0.101:9000`
- **AdGuard Home**: `http://10.0.0.101:3000`  
- **Uptime Kuma**: `http://10.0.0.101:3001`
- **Paperless-ngx**: `http://10.0.0.101:8010`

### 🔒 Security Validation:
- **Partition Security**: Verify `/var`, `/tmp`, `/var/log` have noexec
- **Docker Security**: No containers running as root
- **Network Security**: UFW firewall active and configured
- **Service Security**: All services behind authentication

---

## Rollback Strategy

### If Partitioning Goes Wrong:
1. Boot from Ubuntu ISO in rescue mode
2. Mount existing partitions and backup data
3. Restart installation with simpler partitioning
4. TrueNAS data remains safe

### If Docker Installation Fails:
```bash
# Complete Docker removal
sudo apt purge docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin -y
sudo rm -rf /var/lib/docker /etc/docker
# Restart from Step 5
```

### If Container Won't Start:
1. **Portainer** → **Containers** → View logs
2. Check system resources: `free -h && df -h`
3. Restart via Portainer interface
4. Check mount permissions if storage-related

### If Network Issues Occur:
```bash
# Reset netplan configuration
sudo netplan --debug apply
# Check NFS connectivity
showmount -e 10.0.0.110
ping 10.0.0.110
```

### Complete VM Reset:
1. Shutdown VM in Proxmox
2. Restore from Phase 1 snapshot if needed
3. Or delete VM and recreate from Step 1
4. All TrueNAS data remains safe

---

## Pre-Phase 3 Snapshot

**Before proceeding to Phase 3:**

1. **Comprehensive Service Test**:
   - Access each service web interface
   - Upload test document to Paperless-ngx  
   - Verify DNS blocking from another device on network
   - Check all containers running in Portainer
   - Verify NFS mounts working properly

2. **Security Validation**:
   ```bash
   # Check partition mount options
   mount | grep -E "(noexec|nosuid|nodev)"
   
   # Verify firewall status
   sudo ufw status verbose
   
   # Test SSH security
   ssh homelab@10.0.0.101 -p 2222
   ```

3. **VM Snapshot**:
   - **Docker Services VM** → **Snapshots** → **Take Snapshot**
   - **Name**: `Phase2-Complete-All-Services-Secured`

4. **Configuration Backup**:
   - Export all Portainer stacks: **Stacks** → **Export**
   - Backup system configs to TrueNAS:
   ```bash
   sudo tar -czf /mnt/truenas/backups/docker-services-configs-$(date +%Y%m%d).tar.gz \
     /etc/fstab /etc/netplan /etc/ssh/sshd_config /etc/docker/daemon.json
   ```

5. **Optional Network DNS Configuration**:
   - Point router DNS to `10.0.0.101` for network-wide ad blocking
   - Test DNS resolution from multiple devices on network
   - Verify internet connectivity maintained

**Phase 2 is complete when all completion criteria are met, security validation passes, and snapshots are taken.**

---

## Next: Phase 3 - Gaming & Media VMs

Your containerized services foundation is now complete with:
- **Web-managed containers** via Portainer (no manual docker-compose needed)
- **Network-wide ad blocking** protecting all devices
- **Comprehensive monitoring** of all infrastructure
- **Secure document management** for sensitive files
- **Hardened security** with proper partitioning and firewall
- **Integrated storage** via NFS to TrueNAS

Ready to proceed when Phase 2 completion criteria are met and snapshots are taken.
