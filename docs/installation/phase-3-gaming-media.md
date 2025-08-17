## Phase 3 Completion Checklist

- [ ] **VM2 (Pterodactyl)**: Gaming server operational with CrowdSec hardening
- [ ] **VM4 (Media Server)**: Plex Media Server with hardware transcoding
- [ ] **VM5 (Media Automation)**: Sonarr, Radarr, Lidarr, qBittorrent + VPN operational

## Overview
Phase 3 deploys the core entertainment services: gaming servers, media streaming, and automated media management. Each VM is hardened immediately after deployment to maintain security throughout the build process.

## Prerequisites
- ✅ Phase 1: Proxmox + TrueNAS operational
- ✅ Phase 2: Docker Services VM with Portainer running
- 🔑 YubiKey 5 NFC ready for 2FA setup
- 🌐 Cloudflare account with domain configured

## VM Resource Allocation

| VM | Purpose | RAM | CPU | Storage | Priority |
|----|---------|-----|-----|---------|----------|
| VM2 | Pterodactyl (Gaming) | 16GB | 4 cores | 100GB | High |
| VM4 | Plex Media Server | 8GB | 4 cores | 50GB | Medium |
| VM5 | Media Automation | 8GB | 2 cores | 50GB | Medium |

---

## Part A: Pterodactyl Gaming Server VM

### A1: VM Creation & Base Hardening

```bash
# Create VM in Proxmox (adjust VM ID as needed)
# VM2: Pterodactyl Gaming Server
# - 16GB RAM, 4 CPU cores, 100GB storage
# - Ubuntu Server 24.04 LTS with Ubuntu Pro

# After VM creation and Ubuntu installation, connect via Termius for initial setup
```

### A2: Immediate System Hardening

```bash
# Update system immediately
sudo apt update && sudo apt upgrade -y

# Install essential security packages
sudo apt install -y iptables-persistent unattended-upgrades apt-listchanges

# Configure automatic security updates
sudo dpkg-reconfigure -plow unattended-upgrades

# Install CrowdSec with specific scenarios for MariaDB (main exposure point)
curl -s https://install.crowdsec.net | sudo bash
sudo systemctl enable crowdsec
sudo systemctl start crowdsec

# Install collections for MariaDB protection
sudo cscli collections install crowdsecurity/mariadb
sudo cscli collections install crowdsecurity/nginx  
sudo cscli collections install crowdsecurity/http-cve
sudo cscli collections install crowdsecurity/base-http-scenarios

# Configure CrowdSec for MariaDB logs
sudo tee -a /etc/crowdsec/acquis.yaml << EOF
filenames:
  - /var/log/mysql/error.log
  - /var/log/mysql/mysql-slow.log
labels:
  type: mysql
---
filenames:
  - /var/log/nginx/access.log
  - /var/log/nginx/error.log  
labels:
  type: nginx
EOF

# Restart CrowdSec to apply new configurations
sudo systemctl restart crowdsec

# Configure iptables for local-only SSH and Docker compatibility
sudo iptables -P INPUT DROP
sudo iptables -P FORWARD DROP
sudo iptables -P OUTPUT ACCEPT
sudo iptables -A INPUT -i lo -j ACCEPT
sudo iptables -A INPUT -m conntrack --ctstate ESTABLISHED,RELATED -j ACCEPT
# SSH restricted to your 10.0.0.x network
sudo iptables -A INPUT -p tcp -s 10.0.0.0/24 --dport 22 -j ACCEPT
# HTTP/HTTPS for Cloudflare tunnels
sudo iptables -A INPUT -p tcp --dport 80 -j ACCEPT
sudo iptables -A INPUT -p tcp --dport 443 -j ACCEPT

# Save iptables rules
sudo netfilter-persistent save

# Configure SSH for YubiKey FIDO2 and local-only access
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.backup
sudo tee -a /etc/ssh/sshd_config << EOF

# Security hardening - YubiKey FIDO2 via Termius
PermitRootLogin no
PasswordAuthentication no
ChallengeResponseAuthentication no
PubkeyAuthentication yes
X11Forwarding no
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
# Restrict to local 10.0.0.x network only
ListenAddress 0.0.0.0
AllowUsers admin@10.0.0.*
EOF

# Note: Termius will generate FIDO2 keys directly with YubiKey
# No additional SSH configuration needed for YubiKey integration

sudo systemctl restart sshd

# Configure YubiKey for sudo (replace password entirely)
sudo apt install -y libpam-u2f

# Generate U2F keys for your user (run this as the admin user, not root)
su - admin -c "mkdir -p ~/.config/Yubico && pamu2fcfg > ~/.config/Yubico/u2f_keys"

# Configure PAM to use YubiKey for sudo
sudo tee -a /etc/pam.d/sudo << EOF
# YubiKey U2F authentication
auth sufficient pam_u2f.so authfile=/home/admin/.config/Yubico/u2f_keys cue
EOF

# Set up system monitoring
sudo apt install -y htop iotop nethogs
```

### A3: Pterodactyl Installation

```bash
# Install dependencies
sudo apt install -y software-properties-common curl apt-transport-https ca-certificates gnupg

# Add PHP 8.3 repository (Pterodactyl recommended version)
sudo add-apt-repository -y ppa:ondrej/php
sudo apt update

# Install PHP 8.3 and extensions (as per Pterodactyl requirements)
sudo apt install -y php8.3 php8.3-cli php8.3-gd php8.3-mysql php8.3-pdo php8.3-mbstring php8.3-tokenizer php8.3-bcmath php8.3-xml php8.3-fpm php8.3-curl php8.3-zip

# Install Composer
curl -sS https://getcomposer.org/installer | sudo php -- --install-dir=/usr/local/bin --filename=composer

# Install MariaDB (protected by CrowdSec)
sudo apt install -y mariadb-server mariadb-client

# Secure MariaDB installation
sudo mariadb-secure-installation
# Choose: Y, new_root_password, Y, Y, Y, Y

# Create Pterodactyl database with strong password using MariaDB commands
DB_PASSWORD=$(openssl rand -base64 32)
sudo mariadb -u root -p << EOF
CREATE USER 'pterodactyl'@'127.0.0.1' IDENTIFIED BY '${DB_PASSWORD}';
CREATE DATABASE panel;
GRANT ALL PRIVILEGES ON panel.* TO 'pterodactyl'@'127.0.0.1' WITH GRANT OPTION;
FLUSH PRIVILEGES;
EXIT;
EOF

# Save database password securely
echo "Pterodactyl DB Password: ${DB_PASSWORD}" | sudo tee /root/pterodactyl_db_password.txt
sudo chmod 600 /root/pterodactyl_db_password.txt

# Download and install Pterodactyl
cd /var/www
sudo curl -Lo panel.tar.gz https://github.com/pterodactyl/panel/releases/latest/download/panel.tar.gz
sudo tar -xzvf panel.tar.gz
sudo chmod -R 755 storage/* bootstrap/cache/

# Install dependencies
sudo composer install --no-dev --optimize-autoloader

# Environment setup (use the generated password)
sudo php artisan p:environment:setup
sudo php artisan p:environment:database

# Run setup commands
sudo php artisan migrate --seed --force
sudo php artisan p:user:make

# Set correct permissions
sudo chown -R www-data:www-data /var/www/pterodactyl/*
```

### A4: Nginx Configuration & Cloudflare Integration

```bash
# Install Nginx
sudo apt install -y nginx

# Create Pterodactyl site configuration (CF Tunnel compatible)
sudo tee /etc/nginx/sites-available/pterodactyl.conf << EOF
server {
    listen 80;
    server_name games.tehzombijesus.ca;

    root /var/www/pterodactyl/public;
    index index.html index.htm index.php;
    charset utf-8;

    # Cloudflare specific headers (as per Pterodactyl docs)
    real_ip_header CF-Connecting-IP;
    set_real_ip_from 103.21.244.0/22;
    set_real_ip_from 103.22.200.0/22;
    set_real_ip_from 103.31.4.0/22;
    set_real_ip_from 104.16.0.0/13;
    set_real_ip_from 104.24.0.0/14;
    set_real_ip_from 108.162.192.0/18;
    set_real_ip_from 131.0.72.0/22;
    set_real_ip_from 141.101.64.0/18;
    set_real_ip_from 162.158.0.0/15;
    set_real_ip_from 172.64.0.0/13;
    set_real_ip_from 173.245.48.0/20;
    set_real_ip_from 188.114.96.0/20;
    set_real_ip_from 190.93.240.0/20;
    set_real_ip_from 197.234.240.0/22;
    set_real_ip_from 198.41.128.0/17;
    set_real_ip_from 2400:cb00::/32;
    set_real_ip_from 2606:4700::/32;
    set_real_ip_from 2803:f800::/32;
    set_real_ip_from 2405:b500::/32;
    set_real_ip_from 2405:8100::/32;
    set_real_ip_from 2c0f:f248::/32;
    set_real_ip_from 2a06:98c0::/29;

    location / {
        try_files \$uri \$uri/ /index.php?\$query_string;
    }

    location = /favicon.ico { access_log off; log_not_found off; }
    location = /robots.txt  { access_log off; log_not_found off; }

    access_log off;
    error_log  /var/log/nginx/pterodactyl.app-error.log error;

    location ~ \.php\$ {
        fastcgi_pass unix:/run/php/php8.3-fpm.sock;
        fastcgi_param SCRIPT_FILENAME \$realpath_root\$fastcgi_script_name;
        include fastcgi_params;
    }

    location ~ /\.(?!well-known).* {
        deny all;
    }
}
EOF

# Enable site and restart Nginx
sudo ln -s /etc/nginx/sites-available/pterodactyl.conf /etc/nginx/sites-enabled/pterodactyl.conf
sudo rm /etc/nginx/sites-enabled/default
sudo systemctl restart nginx
```

### A5: Cloudflare Tunnel Setup for Pterodactyl

```bash
# Install cloudflared
wget -q https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Follow Cloudflare Dashboard instructions:
# 1. Go to Cloudflare Zero Trust > Networks > Tunnels
# 2. Create tunnel: "pterodactyl-tunnel"  
# 3. Copy and paste the provided commands from the dashboard
# 4. Add public hostname: games.tehzombijesus.ca -> http://localhost:80

# The dashboard will provide commands similar to:
# cloudflared service install [token]
# sudo systemctl enable cloudflared
# sudo systemctl start cloudflared
```

---

## Part B: Plex Media Server VM (Portainer-Managed)

### B1: VM Creation & Hardening

```bash
# Create VM4 in Proxmox
# - 8GB RAM, 4 CPU cores, 50GB storage
# - Ubuntu Server 24.04 LTS

# Connect via Termius and apply base hardening

# Apply same base hardening as Part A2 (CrowdSec + iptables)
# (Repeat hardening steps from A2)
```

### B2: Docker Installation & Portainer Agent

```bash
# Install Docker
sudo apt update
sudo apt install -y ca-certificates curl gnupg lsb-release

curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /usr/share/keyrings/docker-archive-keyring.gpg

echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/docker-archive-keyring.gpg] https://download.docker.com/linux/ubuntu $(lsb_release -cs) stable" | sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

sudo apt update
sudo apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin

# Install Portainer Agent to connect to Phase 2 Portainer
docker run -d \
  --name portainer-agent \
  --restart unless-stopped \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# No CrowdSec needed - services behind CF Tunnels with Zero Trust
```

### B3: Plex Setup via Portainer

```bash
# Create media mount points for Docker volumes
sudo mkdir -p /opt/plex/{config,media/{movies,tv,music,books}}
sudo chown -R 1000:1000 /opt/plex

# Mount TrueNAS shares
sudo apt install -y nfs-common
sudo tee -a /etc/fstab << EOF
<truenas-ip>:/mnt/tank/media/movies /opt/plex/media/movies nfs defaults 0 0
<truenas-ip>:/mnt/tank/media/tv /opt/plex/media/tv nfs defaults 0 0
<truenas-ip>:/mnt/tank/media/music /opt/plex/media/music nfs defaults 0 0
<truenas-ip>:/mnt/tank/media/books /opt/plex/media/books nfs defaults 0 0
EOF

sudo mount -a
```

**Continue in Portainer Web Interface:**
1. Add this VM as endpoint: `<vm-ip>:9001`
2. Deploy Plex stack with this compose:

```yaml
version: '3.8'
services:
  plex:
    image: lscr.io/linuxserver/plex:latest
    container_name: plex
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
      - VERSION=docker
    volumes:
      - /opt/plex/config:/config
      - /opt/plex/media:/data
    ports:
      - "32400:32400"
    devices:
      - /dev/dri:/dev/dri  # Intel Quick Sync
```

### B4: Cloudflare Tunnel for Plex

```bash
# Follow Cloudflare Dashboard for plex tunnel:
# 1. Create tunnel: "plex-tunnel"
# 2. Add hostname: plex.tehzombijesus.ca -> http://localhost:32400
# 3. Copy and run provided commands
```

---

## Part C: Media Automation VM (Portainer-Managed)

### C1: VM Creation & Hardening

```bash
# Create VM5 in Proxmox
# - 8GB RAM, 2 CPU cores, 50GB storage
# - Ubuntu Server 24.04 LTS

# Apply base hardening (same as A2: CrowdSec + iptables)
```

### C2: Docker Installation & Portainer Agent

```bash
# Install Docker (same as Part B2)
# Install Portainer Agent to connect to main Portainer instance

docker run -d \
  --name portainer-agent \
  --restart unless-stopped \
  -p 9001:9001 \
  -v /var/run/docker.sock:/var/run/docker.sock \
  -v /var/lib/docker/volumes:/var/lib/docker/volumes \
  portainer/agent:latest

# No CrowdSec needed - all services behind CF Tunnels

# Create directory structure for volumes
sudo mkdir -p /opt/media-automation/{config/{gluetun,qbittorrent,sabnzbd,sonarr,radarr,lidarr,prowlarr},downloads,media}
sudo chown -R 1000:1000 /opt/media-automation

# Mount TrueNAS media shares
sudo tee -a /etc/fstab << EOF
<truenas-ip>:/mnt/tank/media/movies /opt/media-automation/media/movies nfs defaults 0 0
<truenas-ip>:/mnt/tank/media/tv /opt/media-automation/media/tv nfs defaults 0 0
<truenas-ip>:/mnt/tank/media/music /opt/media-automation/media/music nfs defaults 0 0
EOF

sudo mount -a
```

### C3: Media Automation Stack via Portainer (Usenet + Torrents)

**In Portainer Web Interface:**
1. Add this VM as endpoint: `<vm-ip>:9001`
2. Deploy media automation stack with both Usenet and torrent support:

```yaml
version: "3.8"

services:
  # VPN for torrent traffic only (fallback downloads)
  gluetun:
    image: qmcgaw/gluetun:latest
    container_name: gluetun
    restart: unless-stopped
    cap_add:
      - NET_ADMIN
    devices:
      - /dev/net/tun:/dev/net/tun
    environment:
      - VPN_SERVICE_PROVIDER=<your-vpn-provider>
      - VPN_TYPE=openvpn
      - OPENVPN_USER=<vpn-username>
      - OPENVPN_PASSWORD=<vpn-password>
      - SERVER_COUNTRIES=<preferred-country>
    volumes:
      - /opt/media-automation/config/gluetun:/gluetun
    ports:
      - "8080:8080"   # qBittorrent (torrents - fallback only)

  # Usenet downloader (primary, no VPN needed - safe and legal)
  sabnzbd:
    image: lscr.io/linuxserver/sabnzbd:latest
    container_name: sabnzbd
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /opt/media-automation/config/sabnzbd:/config
      - /opt/media-automation/downloads:/downloads
    ports:
      - "8082:8080"   # SABnzbd web interface

  # Torrent client (through VPN - fallback only)
  qbittorrent:
    image: lscr.io/linuxserver/qbittorrent:latest
    container_name: qbittorrent
    restart: unless-stopped
    network_mode: "service:gluetun"
    depends_on:
      - gluetun
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
      - WEBUI_PORT=8080
    volumes:
      - /opt/media-automation/config/qbittorrent:/config
      - /opt/media-automation/downloads:/downloads

  sonarr:
    image: lscr.io/linuxserver/sonarr:latest
    container_name: sonarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /opt/media-automation/config/sonarr:/config
      - /opt/media-automation/downloads:/downloads
      - /opt/media-automation/media/tv:/tv
    ports:
      - "8989:8989"

  radarr:
    image: lscr.io/linuxserver/radarr:latest
    container_name: radarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /opt/media-automation/config/radarr:/config
      - /opt/media-automation/downloads:/downloads
      - /opt/media-automation/media/movies:/movies
    ports:
      - "7878:7878"

  lidarr:
    image: lscr.io/linuxserver/lidarr:latest
    container_name: lidarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /opt/media-automation/config/lidarr:/config
      - /opt/media-automation/downloads:/downloads
      - /opt/media-automation/media/music:/music
    ports:
      - "8686:8686"

  # Indexer manager for both torrents and Usenet
  prowlarr:
    image: lscr.io/linuxserver/prowlarr:latest
    container_name: prowlarr
    restart: unless-stopped
    environment:
      - PUID=1000
      - PGID=1000
      - TZ=America/Toronto
    volumes:
      - /opt/media-automation/config/prowlarr:/config
    ports:
      - "9696:9696"
```

**Usenet Setup Notes:**
- SABnzbd runs **without VPN** (Usenet is safe and legal)
- You'll need a Usenet provider (Frugal Usenet, Newsgroup Ninja, etc.)
- Prowlarr manages both Usenet indexers and torrent trackers  
- Primary: Usenet for speed and safety
- Fallback: Torrents through VPN for rare content not on Usenet

### C4: Cloudflare Tunnels for Media Services

```bash
# Create tunnels in Cloudflare Dashboard for each service:
# 1. sabnzbd-tunnel: sabnzbd.tehzombijesus.ca -> http://localhost:8082
# 2. qbittorrent-tunnel: qbittorrent.tehzombijesus.ca -> http://localhost:8080
# 3. sonarr-tunnel: sonarr.tehzombijesus.ca -> http://localhost:8989
# 4. radarr-tunnel: radarr.tehzombijesus.ca -> http://localhost:7878  
# 5. lidarr-tunnel: lidarr.tehzombijesus.ca -> http://localhost:8686
# 6. prowlarr-tunnel: prowlarr.tehzombijesus.ca -> http://localhost:9696

# Or create one tunnel with multiple hostnames - follow CF dashboard instructions
```

---

## Security Architecture Notes

### CrowdSec Usage in This Setup

**Gaming VM Only**: CrowdSec is only installed on the gaming VM (Pterodactyl) since it's the only VM with direct database exposure.

**Protected Services**:
1. **MariaDB** - Main target for SQL injection attempts
2. **Nginx** - HTTP-based attacks before CF tunnel

**Media VMs**: No CrowdSec needed since all services are behind Cloudflare Tunnels with Zero Trust authentication and no direct database exposure.

### Cloudflare Zero Trust Setup

1. **Access Applications** (in Cloudflare Dashboard):
   - Navigate to Zero Trust > Access > Applications
   - Create applications for each subdomain:
     - games.tehzombijesus.ca (Pterodactyl)
     - plex.tehzombijesus.ca (Plex)
     - sonarr.tehzombijesus.ca, radarr.tehzombijesus.ca, etc.

2. **Authentication Methods**:
   ```
   One-Time PIN: Your email address
   TOTP: YubiKey 5 NFC authenticator app
   ```

3. **Access Policies**:
   - Create policy: "Admin Access"
   - Rule: Emails include your email
   - Require: Email + TOTP
   - Session duration: 8 hours

### YubiKey 5 NFC Configuration

1. **Install YubiKey Authenticator** on mobile device
2. **Configure TOTP** for Cloudflare Zero Trust:
   - In Cloudflare: My Team > Users > [Your User] > Configure
   - Add TOTP method using YubiKey Authenticator
3. **Test Access** to each service with YubiKey authentication

---

## Testing & Validation

### Service Health Checks

```bash
# Check Pterodactyl
curl -I https://games.tehzombijesus.ca

# Check Plex
curl -I https://plex.tehzombijesus.ca

# Check media services
curl -I https://sonarr.tehzombijesus.ca
curl -I https://radarr.tehzombijesus.ca
curl -I https://lidarr.tehzombijesus.ca
```

### Security Validation

```bash
# Check iptables rules on each VM
sudo iptables -L -n -v

# Check CrowdSec status (Gaming VM only)
sudo cscli metrics
sudo journalctl -u crowdsec -f

# Check Cloudflare tunnel status
sudo systemctl status cloudflared
```

---

## Phase 3 Completion Checklist

- [ ] **VM2 (Pterodactyl)**: Gaming server operational with CrowdSec hardening
- [ ] **VM4 (Plex)**: Media server streaming with hardware transcoding via Portainer
- [ ] **VM5 (Media Automation)**: Usenet (SABnzbd) + *arr services + torrents as fallback
- [ ] **Security**: Gaming VM hardened with iptables, CrowdSec, and YubiKey SSH
- [ ] **Security**: Media VMs hardened with iptables and YubiKey SSH (no CrowdSec needed)
- [ ] **Cloudflare**: All services accessible via Zero Trust with YubiKey 2FA
- [ ] **Storage**: TrueNAS shares mounted and accessible to all VMs
- [ ] **Usenet**: Primary download method configured (fast, safe, legal)
- [ ] **VPN**: Torrent fallback encrypted and IP protected via Gluetun
- [ ] **Portainer**: Media VMs connected as agents to Phase 2 Portainer instance
- [ ] **Monitoring**: Basic system monitoring tools installed on all VMs

## Next Steps

After Phase 3 completion:
- Phase 4: Advanced monitoring, alerting, and final security hardening
- Usenet provider subscription and indexer setup
- Service optimization and fine-tuning
- Automated backup validation
- Performance monitoring setup

---

**🔒 Security Note**: Gaming VM uses CrowdSec for MariaDB protection. Media VMs rely on Cloudflare Zero Trust + YubiKey authentication since all services are tunneled with no direct database exposure.
