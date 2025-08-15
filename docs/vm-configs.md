# 🖥️ VM Configurations

Detailed configuration specifications for all 5 virtual machines in the home lab setup.

## 📊 Resource Allocation Overview

| VM | OS | RAM | Disk | CPU Cores | Priority | License |
|----|----|----|------|-----------|----------|---------|
| TrueNAS | TrueNAS SCALE | 8GB | 50GB | 2-4 | Medium | N/A |
| Pterodactyl | Ubuntu 22.04 LTS | 24-32GB | 200GB | 4-6 | High | Ubuntu Pro |
| Docker Services | Ubuntu 22.04 LTS | 12GB | 100GB | 2-4 | Medium | Ubuntu Pro |
| Media Server | Ubuntu 22.04 LTS | 8GB | 100GB | 2-4 | Medium | Ubuntu Pro |
| Media Automation | Ubuntu 22.04 LTS | 8GB | 150GB | 2-4 | Low | Ubuntu Pro |
| **Totals** | | **60-68GB** | **600GB** | | | **4/5 Licenses** |

## 🗄️ VM1: TrueNAS Storage

### System Configuration
- **Operating System:** TrueNAS SCALE (Debian-based)
- **RAM:** 8GB (recommended minimum for ZFS)
- **Storage:** 50GB system disk + remaining RAID 1 space for datasets
- **CPU:** 2-4 cores
- **Network:** Static IP on management network

### ZFS Dataset Structure
```
Pool: main-pool (RAID 1 - ~1.4TB usable)
├── datasets/
│   ├── media/
│   │   ├── movies/          (~600GB)
│   │   ├── tvshows/         (~500GB)
│   │   └── music/           (~200GB)
│   ├── downloads/
│   │   ├── complete/        (~50GB buffer)
│   │   └── incomplete/      (~50GB buffer)
│   └── backups/
│       ├── vm-backups/      (~50GB)
│       ├── desktop-backups/ (~50GB)
│       └── configs/         (~10GB)
```

### Share Configuration
| Share Name | Path | Access | Purpose |
|------------|------|--------|---------|
| `media-movies` | `/datasets/media/movies` | Read-Only | Plex media consumption |
| `media-tv` | `/datasets/media/tvshows` | Read-Only | Plex media consumption |
| `media-music` | `/datasets/media/music` | Read-Only | Plex media consumption |
| `downloads` | `/datasets/downloads` | Read-Write | Media automation downloads |
| `vm-backups` | `/datasets/backups/vm-backups` | Read-Write | Proxmox backup storage |

### Performance Settings
- **ARC Cache:** 4GB (50% of RAM)
- **Compression:** LZ4 (fast compression)
- **Snapshots:** Daily retention for 30 days
- **Scrub Schedule:** Monthly on first Sunday

---

## 🎮 VM2: Pterodactyl Gaming

### System Configuration
- **Operating System:** Ubuntu Server 22.04 LTS + Pro
- **RAM:** 24-32GB (allocate based on game server needs)
- **Storage:** 200GB (game files, maps, plugins)
- **CPU:** 4-6 cores with high priority
- **Network:** Static IP + Cloudflare tunnel endpoint

### Resource Allocation by Game Type
| Game Server | RAM | Storage | Notes |
|-------------|-----|---------|-------|
| Minecraft (Vanilla) | 4-6GB | 2-5GB | Per server instance |
| Minecraft (Modded) | 8-12GB | 5-15GB | Depends on mod pack |
| Rust Server | 8-12GB | 15-25GB | Large map files |
| Discord Bot | 512MB-1GB | 1GB | Node.js/Python bots |
| **Buffer** | 4-6GB | 50GB | OS + Panel overhead |

### Pterodactyl Configuration
```bash
# Panel Installation
Panel URL: https://games.tehzombijesus.ca
Database: MariaDB (local)
Cache: Redis (local)
Queue: Redis (local)

# Wings Configuration  
Node: Local Node (same VM)
SFTP: Port 2022 (internal only)
Daemon Port: 8080 (internal only)
SSL: Cloudflare tunnel termination
```

### Game Server Templates
- **Minecraft Eggs:** Paper, Fabric, Forge, Vanilla, Bedrock
- **Rust Egg:** Official Rust server with oxide support
- **Bot Eggs:** Node.js, Python, generic applications
- **Custom Eggs:** Community-created server types

### Security Configuration
- **Firewall:** UFW with restrictive rules
- **Container Isolation:** Each game server in isolated container
- **Resource Limits:** Per-server CPU/RAM limits enforced
- **Backup Integration:** Automatic backups to TrueNAS

---

## 🐳 VM3: Docker Services Hub

### System Configuration
- **Operating System:** Ubuntu Server 22.04 LTS + Pro
- **RAM:** 12GB (distributed across containers)
- **Storage:** 100GB (container data and logs)
- **CPU:** 2-4 cores
- **Network:** Static IP with multiple service ports

### Container Stack
| Service | RAM | Storage | Ports | Purpose |
|---------|-----|---------|-------|---------|
| Portainer | 256MB | 2GB | 9000 | Docker management |
| AdGuard Home | 256MB | 2GB | 53,80,443,3000 | DNS/Ad blocking |
| Uptime Kuma | 256MB | 2GB | 3001 | Service monitoring |
| Homer | 128MB | 1GB | 8080 | Service dashboard |
| Netdata | 512MB | 5GB | 19999 | System monitoring |
| Paperless-ngx | 2GB | 20GB | 8000 | Document management |
| **Overhead** | 1GB | 10GB | | Docker + OS |

### Docker Compose Structure
```yaml
# Main services stack
services:
  portainer:
    # Container management interface
  adguard:
    # DNS and ad blocking
  uptime-kuma:
    # Service monitoring
  homer:
    # Service dashboard
  netdata:
    # System metrics
  paperless:
    # Document OCR and management
```

### Volume Mapping
- **Portainer Data:** `/opt/portainer/data`
- **AdGuard Config:** `/opt/adguard/conf`
- **Uptime Kuma:** `/opt/uptime-kuma/data`
- **Paperless Media:** `/opt/paperless/media` (mounted from TrueNAS)
- **Shared Configs:** `/opt/shared/configs`

### Network Configuration
- **Bridge Network:** Custom Docker network for inter-service communication
- **DNS Override:** AdGuard Home as primary DNS for local network
- **Cloudflare Integration:** Tunnel access for external services

---

## 🎬 VM4: Media Server

### System Configuration
- **Operating System:** Ubuntu Server 22.04 LTS + Pro
- **RAM:** 8GB (Plex transcoding and metadata)
- **Storage:** 100GB (Plex database and transcoding cache)
- **CPU:** 2-4 cores with Quick Sync support
- **Network:** Static IP with media streaming optimization

### Plex Media Server Configuration
```bash
# Installation
Installation: Native .deb package (not Docker)
Version: Plex Pass (lifetime license recommended)
Database: SQLite (local)
Transcoding: Hardware-accelerated (Intel Quick Sync)

# Library Structure
Movies: /mnt/truenas/media-movies
TV Shows: /mnt/truenas/media-tv  
Music: /mnt/truenas/media-music
```

### Hardware Transcoding Setup
- **Intel Quick Sync:** Enabled for i5-9500T
- **Transcoding Directory:** `/tmp/plex-transcoding` (tmpfs for SSD longevity)
- **Quality Settings:** Optimized for 1080p streaming
- **Bandwidth Control:** Adaptive streaming enabled

### Storage Mounts
| Mount Point | Source | Type | Options |
|-------------|--------|------|---------|
| `/mnt/truenas/media-movies` | TrueNAS movies share | NFS | ro,hard,intr |
| `/mnt/truenas/media-tv` | TrueNAS TV share | NFS | ro,hard,intr |
| `/mnt/truenas/media-music` | TrueNAS music share | NFS | ro,hard,intr |

### Performance Optimization
- **RAM Usage:** 4GB for Plex, 4GB for system cache
- **Database Optimization:** Regular vacuum and optimize
- **Thumbnail Generation:** Background processing during low usage
- **Remote Access:** Via Cloudflare tunnel only

---

## 📥 VM5: Media Automation

### System Configuration
- **Operating System:** Ubuntu Server 22.04 LTS + Pro
- **RAM:** 8GB (distributed across automation containers)
- **Storage:** 150GB (download cache and processing)
- **CPU:** 2-4 cores (lowest priority for background tasks)
- **Network:** VPN-protected container networking

### Automation Stack
| Service | RAM | Storage | VPN | Purpose |
|---------|-----|---------|-----|---------|
| Sonarr | 1GB | 10GB | No | TV show automation |
| Radarr | 1GB | 10GB | No | Movie automation |
| Lidarr | 1GB | 10GB | No | Music automation |
| Prowlarr | 512MB | 5GB | No | Indexer management |
| qBittorrent+VPN | 2GB | 50GB | Yes | Protected downloads |
| **System** | 2.5GB | 65GB | | OS and overhead |

### VPN Container Configuration
```yaml
qbittorrent-vpn:
  image: binhex/arch-qbittorrentvpn
  environment:
    - VPN_ENABLED=yes
    - VPN_PROV=custom  # Your VPN provider
    - VPN_CLIENT=openvpn
    - ENABLE_PRIVOXY=yes
    - LAN_NETWORK=192.168.1.0/24
  volumes:
    - /opt/qbittorrent/config:/config
    - /mnt/truenas/downloads:/downloads
  cap_add:
    - NET_ADMIN
```

### Download Workflow
1. **Sonarr/Radarr/Lidarr** search indexers via Prowlarr
2. **Downloads** sent to qBittorrent through VPN
3. **Completed files** moved to TrueNAS organized folders
4. **Plex** automatically detects new media
5. **Cleanup** removes processed downloads

### Storage Integration
- **Downloads:** `/mnt/truenas/downloads/` (read-write NFS mount)
- **Processing:** Local SSD for speed during organization
- **Completed:** Moved to appropriate TrueNAS media folders
- **Failed:** Quarantined in separate folder for review

---

## 🔧 VM Management Best Practices

### Resource Monitoring
- **RAM Usage:** Keep 10-15% free for system operations
- **CPU Load:** Monitor during peak usage (evening streaming + gaming)
- **Storage Growth:** Plan for media library expansion
- **Network I/O:** Monitor during large downloads/streams

### Backup Strategy
- **VM Snapshots:** Daily snapshots before major changes
- **Configuration Backups:** Weekly exports of service configs
- **Data Protection:** Separate data from application configurations
- **Recovery Testing:** Monthly restore verification

### Update Procedures
1. **System Updates:** Monthly security patches
2. **Application Updates:** Staggered deployment (test → production)
3. **Container Updates:** Automated via Watchtower (optional)
4. **Rollback Plan:** Snapshot before major updates

### Performance Tuning
- **CPU Affinity:** Pin high-priority VMs to specific cores
- **RAM Ballooning:** Enable for dynamic memory allocation
- **Disk I/O:** Use virtio drivers for best performance  
- **Network:** virtio-net for optimal throughput

---

*Return to [Main Documentation](../README.md)*
