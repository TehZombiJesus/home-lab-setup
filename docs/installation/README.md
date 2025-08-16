# Home Lab Installation Overview

This guide will take you through building a complete privacy-focused home lab with gaming servers, media automation, and document management. The installation is broken into 4 phases to ensure stability and allow for troubleshooting at each step.

## Before You Begin

### Hardware Requirements
- **Host System**: HP EliteDesk 800 G5 (or similar)
- **RAM**: 64GB total (60-68GB will be allocated)
- **Storage**: 2TB in RAID 1 configuration
- **Network**: Ethernet connection with internet access

### Prerequisites Checklist
- [ ] Hardware assembled and tested
- [ ] 5 Ubuntu Pro licenses available (free for personal use)
- [ ] Domain name registered and accessible
- [ ] VPN provider subscription for secure downloads
- [ ] Cloudflare account (free plan sufficient)

### Important Constraints
⚠️ **ISP Limitations**: This setup is designed for networks where incoming ports are blocked by your ISP. All external access uses Cloudflare tunnels - **standard port forwarding will not work**.

## Installation Phases

### Phase 1: Foundation (2-3 hours)
**Goal**: Set up Proxmox hypervisor and TrueNAS storage
- Install Proxmox VE hypervisor
- Create and configure TrueNAS VM
- Set up storage pools and network shares
- Establish backup foundation

**[Start Phase 1 →](phase-1-foundation.md)**

---

### Phase 2: Docker Services (1-2 hours)
**Goal**: Create container platform with core services
- Deploy Ubuntu Pro VM for containers
- Install Docker and Portainer
- Configure AdGuard Home for network-wide ad blocking
- Set up basic monitoring and document management
- Establish container management workflow

**[Start Phase 2 →](phase-2-docker-services.md)**

---

### Phase 3: Gaming & Media (2-4 hours)
**Goal**: Deploy entertainment and gaming infrastructure
- **Gaming VM**: Pterodactyl panel with Minecraft/Rust servers
- **Media Server VM**: Plex with hardware transcoding
- **Media Automation VM**: Sonarr, Radarr, Lidarr with VPN
- Configure inter-service communication

**[Start Phase 3 →](phase-3-gaming-media.md)**

---

### Phase 4: Security & Monitoring (1-2 hours)
**Goal**: Secure external access and implement monitoring
- Configure Cloudflare tunnels for all services
- Set up Zero Trust authentication
- Deploy comprehensive monitoring stack
- Establish automated backup procedures
- Final security hardening

**[Start Phase 4 →](phase-4-security-monitoring.md)**

## Installation Philosophy

### Phased Approach Benefits
- **Stability**: Each phase builds on a working foundation
- **Troubleshooting**: Issues isolated to specific phase
- **Flexibility**: Can pause between phases or modify later phases
- **Learning**: Understand each component before adding complexity

### Safety First
- **Snapshots**: Taken before each major change
- **Rollback Plans**: Every phase includes recovery procedures
- **Validation**: Clear success criteria before proceeding
- **Backups**: Multiple backup strategies implemented early

### Web-First Management
- **Zero SSH Required**: All administration through web interfaces
- **Mobile Friendly**: Most interfaces work well on tablets/phones
- **Centralized Access**: Single dashboard for all services
- **Secure Remote Access**: Cloudflare tunnels with authentication

## Resource Allocation Overview

| VM | RAM | CPU Cores | Storage | Purpose |
|----|-----|-----------|---------|---------|
| TrueNAS | 8GB | 2 | 32GB + Data Disks | Storage & Backups |
| Docker Services | 12GB | 4 | 64GB | Containers & Core Services |
| Pterodactyl | 16GB | 4 | 128GB | Gaming Servers |
| Plex Server | 8GB | 4 | 32GB | Media Streaming |
| Media Automation | 8GB | 2 | 32GB | Download Management |
| **Total** | **52GB** | **16** | **~300GB** | **All Services** |

*Remaining 12GB RAM reserved for Proxmox host and headroom*

## Network Architecture

```
Internet → Cloudflare → Tunnels → Home Lab Services
                    ↳ Zero Trust Auth → Authenticated Users
```

**Why This Architecture:**
- **ISP Bypass**: No ports need to be opened on router
- **Enterprise Security**: Cloudflare's edge protection
- **Global Access**: Services available worldwide with low latency
- **Free Tier**: No additional costs beyond domain registration

## Success Metrics

### Phase 1 Complete
- Proxmox web interface accessible
- TrueNAS VM running with storage pools
- Network shares accessible from other devices
- Backup system operational

### Phase 2 Complete  
- Portainer managing containers
- AdGuard Home blocking ads network-wide
- Basic monitoring dashboards functional
- Document management system ready

### Phase 3 Complete
- Game servers accessible and functional
- Media library streaming to devices
- Automated media acquisition working
- All services isolated and stable

### Phase 4 Complete
- All services accessible via custom domains
- Authentication protecting sensitive services
- Comprehensive monitoring and alerting
- Automated backup procedures running

## Getting Help

### During Installation
- Each phase includes detailed troubleshooting sections
- Rollback procedures for every major step
- Resource usage validation at each checkpoint
- Common error solutions included

### After Installation
- Service-specific troubleshooting in reference docs
- Maintenance procedures for ongoing operations
- Upgrade paths and expansion guidelines
- Community best practices and tips

## Ready to Begin?

**Start with Phase 1** if you have completed all prerequisites.

**Need to review requirements first?** Check the [Hardware Specifications](../reference/hardware.md) and [System Architecture](../reference/architecture.md) docs.

**Questions about the approach?** The [FAQ section](troubleshooting.md#frequently-asked-questions) covers common concerns about this setup.

---

**[Begin Installation: Phase 1 - Foundation →](phase-1-foundation.md)**
