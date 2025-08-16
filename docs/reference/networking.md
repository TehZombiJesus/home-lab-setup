# Network & Security - tehzombijesus.ca Home Lab

## 🛡️ Security Overview

This home lab implements a **Zero Trust security model** with multiple layers of protection, eliminating the need for traditional port forwarding while providing enterprise-grade security and access control.

## 🌐 Network Architecture

### **ISP Constraints & Solutions**

**Port Restrictions (ISP Blocked):**
```
Incoming Blocked Ports:
├── 25    (SMTP - Email)
├── 53    (DNS - Domain Name System)
├── 55    (DSF - Directory Services)
├── 77    (RJE - Remote Job Entry)
├── 135   (Microsoft RPC)
├── 139   (NetBIOS Session)
├── 161   (SNMP)
├── 162   (SNMP Trap)
├── 445   (Microsoft SMB)
├── 1080  (SOCKS Proxy)
└── 4444  (Metasploit Default)
```

**Solution: Cloudflare Tunnels**
- **No inbound ports required** - All connections are outbound from your lab
- **Enterprise-grade DDoS protection** included
- **Automatic SSL/TLS termination** with certificates
- **Geographic load balancing** and failover capabilities

### **Internal Network Layout**

```
Network Topology: 10.0.0.0/24

├── Gateway: 10.0.0.1 (ISP Router)
├── Proxmox Host: 10.0.0.5
├── VM1 (TrueNAS): 10.0.0.10
├── VM2 (Pterodactyl): 10.0.0.20  
├── VM3 (Docker Services): 10.0.0.30
├── VM4 (Plex): 10.0.0.40
└── VM5 (Media Automation): 10.0.0.50
```

**Network Services:**
- **Primary DNS**: AdGuard Home (10.0.0.30:53)
- **DHCP**: ISP Router (maintains current setup)
- **Internal Resolution**: Custom DNS records for service discovery
- **Storage Access**: SMB/NFS shares from TrueNAS

---

## 🌩️ Cloudflare Configuration

### **Domain Setup: tehzombijesus.ca**

**DNS Records Configuration:**
```
DNS Zone Configuration:
├── A Record: tehzombijesus.ca → Cloudflare Proxy (Orange Cloud)
├── CNAME: *.tehzombijesus.ca → tehzombijesus.ca
├── MX Records: (if email needed)
└── TXT Records: Domain verification, SPF, DKIM
```

**Subdomain Strategy:**
| Service Category | Subdomain | Target VM | Purpose |
|------------------|-----------|-----------|---------|
| **Management** | `admin.tehzombijesus.ca` | VM3 | Portainer Dashboard |
| **Gaming** | `games.tehzombijesus.ca` | VM2 | Pterodactyl Panel |
| **Media** | `plex.tehzombijesus.ca` | VM4 | Plex Media Server |
| **Documents** | `docs.tehzombijesus.ca` | VM3 | Paperless-ngx |
| **Monitoring** | `status.tehzombijesus.ca` | VM3 | Uptime Kuma |
| **Downloads** | `downloads.tehzombijesus.ca` | VM5 | *arr Services |
| **Network** | `dns.tehzombijesus.ca` | VM3 | AdGuard Home |

### **Cloudflare Tunnel Setup**

**Step 1: Install Cloudflared on VM3**
```bash
# Download and install cloudflared
wget https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
sudo dpkg -i cloudflared-linux-amd64.deb

# Authenticate with Cloudflare
cloudflared tunnel login

# Create tunnel
cloudflared tunnel create homelab-tunnel

# Configure tunnel
sudo mkdir -p /etc/cloudflared
```

**Step 2: Tunnel Configuration File**
```yaml
# /etc/cloudflared/config.yml
tunnel: homelab-tunnel
credentials-file: /root/.cloudflared/tunnel-credentials.json

ingress:
  # Portainer (Container Management)
  - hostname: admin.tehzombijesus.ca
    service: http://10.0.0.30:9000
    originRequest:
      httpHostHeader: admin.tehzombijesus.ca
      
  # Pterodactyl Gaming Panel
  - hostname: games.tehzombijesus.ca
    service: http://10.0.0.20:80
    originRequest:
      httpHostHeader: games.tehzombijesus.ca
      
  # Plex Media Server
  - hostname: plex.tehzombijesus.ca
    service: http://10.0.0.40:32400
    originRequest:
      httpHostHeader: plex.tehzombijesus.ca
      
  # Paperless Document Management
  - hostname: docs.tehzombijesus.ca
    service: http://10.0.0.30:8000
    originRequest:
      httpHostHeader: docs.tehzombijesus.ca
      
  # Uptime Monitoring
  - hostname: status.tehzombijesus.ca
    service: http://10.0.0.30:3001
    originRequest:
      httpHostHeader: status.tehzombijesus.ca
      
  # Media Automation (*arr stack)
  - hostname: downloads.tehzombijesus.ca
    service: http://10.0.0.50:8080
    originRequest:
      httpHostHeader: downloads.tehzombijesus.ca
      
  # AdGuard Home
  - hostname: dns.tehzombijesus.ca
    service: http://10.0.0.30:3000
    originRequest:
      httpHostHeader: dns.tehzombijesus.ca

  # Default rule - must be last
  - service: http_status:404
```

**Step 3: DNS Configuration**
```bash
# Create DNS records pointing to tunnel
cloudflared tunnel route dns homelab-tunnel admin.tehzombijesus.ca
cloudflared tunnel route dns homelab-tunnel games.tehzombijesus.ca
cloudflared tunnel route dns homelab-tunnel plex.tehzombijesus.ca
cloudflared tunnel route dns homelab-tunnel docs.tehzombijesus.ca
cloudflared tunnel route dns homelab-tunnel status.tehzombijesus.ca
cloudflared tunnel route dns homelab-tunnel downloads.tehzombijesus.ca
cloudflared tunnel route dns homelab-tunnel dns.tehzombijesus.ca
```

**Step 4: Service Installation**
```bash
# Install tunnel as system service
sudo cloudflared service install
sudo systemctl enable cloudflared
sudo systemctl start cloudflared

# Check status
sudo systemctl status cloudflared
```

### **SSL/TLS Configuration**

**Cloudflare SSL Settings:**
- **SSL/TLS Encryption**: Full (Strict)
- **Always Use HTTPS**: On
- **HTTP Strict Transport Security (HSTS)**: Enabled
- **Minimum TLS Version**: 1.2
- **Opportunistic Encryption**: On

**Certificate Management:**
- **Origin Certificates**: Generated for backend services
- **Edge Certificates**: Automatically managed by Cloudflare
- **Certificate Transparency**: Monitored for unauthorized certificates

---

## 🔐 Zero Trust Security

### **Cloudflare Access Setup**

**Step 1: Create Access Application**
```
Application Configuration:
├── Application Name: "Home Lab Services"
├── Application Domain: "*.tehzombijesus.ca"
├── Policy Name: "Admin Access"
└── Policy Rules: Email matches admin@tehzombijesus.ca
```

**Step 2: Authentication Methods**
```
Identity Providers:
├── One-Time PIN (Email-based)
├── Google OAuth (Optional backup)
└── GitHub OAuth (For development services)
```

**Step 3: Access Policies**

**Admin Full Access:**
```yaml
Name: Admin Full Access
Action: Allow
Rules:
  - Include:
      Email: admin@tehzombijesus.ca
  - Require:
      Country: Canada, United States
Session Duration: 24 hours
```

**Friends Gaming Access:**
```yaml
Name: Gaming Access
Action: Allow
Applications: games.tehzombijesus.ca
Rules:
  - Include:
      Email domains: 
        - friend1@example.com
        - friend2@example.com
Session Duration: 7 days
```

**Media Sharing Access:**
```yaml
Name: Plex Access  
Action: Allow
Applications: plex.tehzombijesus.ca
Rules:
  - Include:
      Email domains: (trusted friends/family)
Session Duration: 30 days
```

### **Application-Level Security**

**Service-Specific Authentication:**

**Portainer:**
- Local admin account with strong password
- RBAC for team access if expanded
- Session timeout: 8 hours
- Two-factor authentication enabled

**Pterodactyl:**
- Separate user accounts for each game server admin
- API keys for automation with limited scopes
- Audit logging for all administrative actions
- Resource limits per user/server

**Plex:**
- Plex authentication + Cloudflare Zero Trust
- Individual user libraries and access controls
- Sharing restrictions and parental controls
- Device authorization required

**Paperless-ngx:**
- Document-level permissions
- OCR processing in isolated containers
- API access for automation scripts
- Regular backup of document database

---

## 🏠 Internal Network Security

### **AdGuard Home Configuration**

**DNS Filtering Strategy:**
```
Blocklist Categories:
├── Ads & Tracking: Steven Black, EasyList
├── Malware & Phishing: Malware Domain List
├── Adult Content: (Optional family filter)
├── Cryptocurrency Mining: NoCoin filter list
├── Social Media Tracking: Fanboy's Enhanced
└── IoT Telemetry: Smart TV ads, analytics
```

**Custom DNS Rules:**
```bash
# Internal service resolution
10.0.0.5     proxmox.local
10.0.0.10    truenas.local  
10.0.0.20    pterodactyl.local
10.0.0.30    docker.local
10.0.0.40    plex.local
10.0.0.50    automation.local

# Whitelist for services
@@||plex.tv^$important
@@||plexapp.com^$important
@@||cloudflare.com^$important
@@||github.com^$important
```

**Upstream DNS Configuration:**
```
Primary DNS Servers:
├── 1.1.1.1 (Cloudflare)
├── 1.0.0.1 (Cloudflare)
└── Fallback: 8.8.8.8, 8.8.4.4 (Google)

Query Processing:
├── Cache TTL: 300 seconds
├── Query Logging: 90 days retention
├── Statistics: Enabled
└── EDNS Client Subnet: Disabled (privacy)
```

### **Firewall Configuration**

**Proxmox Host Firewall:**
```bash
# Enable Proxmox firewall
pve-firewall status

# Basic rules (configured via web interface)
INPUT:
  - Accept SSH (22) from local network
  - Accept Web Interface (8006) from local network  
  - Accept Proxmox Cluster (5404-5412) if clustering
  - Drop all other inbound traffic

OUTPUT:
  - Accept all outbound traffic
```

**VM-Level Firewalls:**

**TrueNAS (VM1) Firewall:**
```
Inbound Rules:
├── Allow SMB (445) from VM network
├── Allow NFS (2049) from VM network
├── Allow SSH (22) from Proxmox host
├── Allow Web UI (80, 443) from VM network
└── Deny all other inbound

Services Required:
├── Storage protocols (SMB, NFS)
├── Management interfaces
└── Backup sync ports
```

**Gaming VM (VM2) Firewall:**
```
Inbound Rules:
├── Allow HTTP (80, 443) from tunnel
├── Allow game server ports (varies)
├── Allow SSH (22) from Proxmox host
└── Deny direct internet access

Special Considerations:
├── Game server ports exposed via Cloudflare
├── SFTP access for file management
└── API access for automation
```

**Docker Services (VM3) Firewall:**
```
Inbound Rules:
├── Allow container ports from tunnel
├── Allow DNS (53) from all VMs
├── Allow SSH (22) from Proxmox host
└── Allow inter-container communication

Container Network Isolation:
├── Separate networks for different services
├── No direct internet access for sensitive containers
└── Proxy all external requests through tunnel
```

---

## 🔒 VPN Integration

### **Download Protection (VM5)**

**VPN Provider Selection Criteria:**
- No-logs policy with third-party audits
- European jurisdiction (GDPR compliance)
- High-speed servers for large downloads  
- Support for port forwarding (torrent optimization)
- OpenVPN and WireGuard protocol support

**VPN Configuration in qBittorrent:**
```bash
# Install VPN client on VM5
sudo apt install openvpn

# VPN connection script
#!/bin/bash
# /opt/vpn/connect.sh
openvpn --config /opt/vpn/provider.ovpn --auth-user-pass /opt/vpn/auth.txt

# Kill switch implementation
iptables -A OUTPUT -o tun+ -j ACCEPT
iptables -A OUTPUT -d VPN_SERVER_IP -j ACCEPT
iptables -A OUTPUT -o lo -j ACCEPT
iptables -A OUTPUT -j DROP
```

**Service Binding:**
```yaml
# qBittorrent configuration
Network Interface: VPN tunnel interface only
Bind Address: VPN tunnel IP
Kill Switch: Enabled (stops downloads if VPN disconnects)
Port Forwarding: Configured with VPN provider
```

### **Network Monitoring**

**Traffic Analysis:**
```
Monitoring Points:
├── Proxmox host interface (total bandwidth)
├── VM network interfaces (per-service usage)  
├── Cloudflare tunnel metrics (external traffic)
├── VPN tunnel metrics (download traffic)
└── Storage network (backup and sync traffic)
```

**Bandwidth Allocation:**
```
Service Priorities:
├── High: Plex streaming, game servers
├── Medium: Web interfaces, monitoring
├── Low: Downloads, backups
└── Background: System updates, sync
```

---

## 📊 Security Monitoring

### **Threat Detection**

**Log Aggregation:**
```
Log Sources:
├── Cloudflare Access logs (authentication events)
├── AdGuard Home query logs (DNS filtering)
├── Proxmox audit logs (VM management)
├── Container logs (application events)
└── System logs (security events)
```

**Security Metrics:**
```
Key Indicators:
├── Failed authentication attempts
├── Unusual traffic patterns  
├── DNS query anomalies
├── Resource usage spikes
└── Service availability issues
```

**Alerting Rules:**

**Critical Security Events:**
```yaml
Multiple Failed Logins:
  Condition: >5 failed attempts in 10 minutes
  Action: Temporary IP block via Cloudflare
  Notification: Immediate email alert

Unusual Geographic Access:
  Condition: Login from unexpected country
  Action: Require additional verification
  Notification: Email with location details

Service Compromise Indicators:
  Condition: Unexpected admin actions
  Action: Disable service, alert admin
  Notification: Immediate SMS + email
```

### **Backup Security**

**Local Backup Protection:**
```
ZFS Snapshots:
├── Hourly snapshots (24 hours retention)
├── Daily snapshots (7 days retention)
├── Weekly snapshots (4 weeks retention)
└── Monthly snapshots (12 months retention)

Snapshot Security:
├── Read-only after creation
├── Checksum verification
├── Encryption for sensitive data
└── Access logging
```

**Offsite Backup Security:**
```
European Cloud Provider:
├── Client-side encryption before upload
├── Zero-knowledge architecture
├── 3-2-1 backup rule compliance
├── Regular restoration testing
└── Legal jurisdiction protection
```

---

## 🛠️ Security Maintenance

### **Regular Security Tasks**

**Weekly:**
- Review Cloudflare Access logs for anomalies
- Check AdGuard Home query patterns
- Update container images for security patches
- Verify backup integrity

**Monthly:**  
- Update Proxmox and VM operating systems
- Review and rotate service passwords
- Test disaster recovery procedures
- Security scan of exposed services

**Quarterly:**
- Full security audit of all services
- Review and update access policies
- Penetration test of external services
- Update incident response procedures

### **Incident Response Plan**

**Security Incident Categories:**
```
Severity 1: Complete service compromise
├── Immediate isolation of affected services
├── Emergency contact notification
├── Full forensic analysis
└── Service restoration with clean backups

Severity 2: Unauthorized access attempt
├── Enhanced monitoring activation
├── Access review and tightening
├── Log analysis for breach indicators
└── Preventive measure implementation

Severity 3: Service disruption
├── Service restart and monitoring
├── Root cause analysis
├── Performance optimization
└── Process improvement updates
```

**Recovery Procedures:**
```
Complete System Recovery:
├── Phase 1: Assess damage and isolate systems
├── Phase 2: Restore from clean backups
├── Phase 3: Apply security patches and updates  
├── Phase 4: Gradually restore services
├── Phase 5: Enhanced monitoring and documentation
└── Phase 6: Incident post-mortem and improvements
```

---

## 🔮 Security Roadmap

### **Short-Term Improvements (3 months)**
- Implement certificate pinning for critical services
- Add hardware security keys for admin authentication
- Deploy network intrusion detection system
- Enhance log correlation and analysis

### **Medium-Term Enhancements (6-12 months)**
- Implement service mesh for inter-VM communication
- Add behavioral analysis for anomaly detection
- Deploy honeypots for threat intelligence
- Enhance automated incident response

### **Long-Term Vision (12+ months)**
- Full zero-trust network architecture
- AI-powered threat detection and response
- Compliance framework implementation (SOC 2, ISO 27001)
- Advanced threat hunting capabilities

---

This network and security configuration provides enterprise-grade protection while maintaining ease of use and administration. The combination of Cloudflare's edge security, Zero Trust access controls, and comprehensive monitoring creates a robust defense against both external threats and insider risks.
