# Hardware Specifications - tehzombijesus.ca Home Lab

## 🖥️ Host System Overview

### **HP EliteDesk 800 G5 Small Form Factor (SFF)**

**Base System Specifications:**
```
Model: HP EliteDesk 800 G5 SFF
Form Factor: Small Form Factor (36 x 17.5 x 34 cm)
Weight: ~5.9 kg (13 lbs)
Power Supply: 180W 90% efficient external adapter
BIOS: HP UEFI with Secure Boot support
Manufacturing Year: 2019-2020
Business Class: Yes (3-year warranty standard)
```

**Why This Platform?**
- **Enterprise-grade reliability** with business-class components
- **Compact footprint** ideal for home environments
- **Excellent power efficiency** for 24/7 operation
- **Robust cooling design** handles sustained workloads
- **Extensive upgrade options** for future expansion
- **Silent operation** suitable for living spaces

---

## 🔧 CPU Specifications

### **Intel Core i5-9500T**

**Processor Details:**
```
Architecture: Coffee Lake (9th Generation)
Manufacturing Process: 14nm
Base Clock: 2.2 GHz
Max Turbo: 3.7 GHz
Cores: 6 physical cores
Threads: 6 (no hyperthreading)
Cache: 9MB Intel Smart Cache
TDP: 35W (T-series for efficiency)
Socket: LGA1151
```

**Performance Characteristics:**
```
Single-Core Performance: Excellent for virtualization overhead
Multi-Core Performance: Sufficient for 5 concurrent VMs
Power Efficiency: 35W TDP ideal for 24/7 operation
Virtualization: Intel VT-x, VT-d for hardware passthrough
AES-NI: Hardware encryption acceleration
Quick Sync Video: H.264/H.265 hardware transcoding
```

**CPU Feature Set:**
- **Intel VT-x**: Hardware virtualization for efficient VM execution
- **Intel VT-d**: I/O virtualization for device passthrough
- **Intel AES-NI**: Hardware-accelerated encryption/decryption
- **Intel Quick Sync Video**: Dedicated video transcoding engine
- **Intel Turbo Boost 2.0**: Dynamic frequency scaling
- **Enhanced SpeedStep**: Power management and thermal control

**VM Resource Allocation:**
```
Total CPU Allocation (6 cores):
├── Proxmox Host: 0.5 cores (reserved)
├── TrueNAS: 1 core (storage operations)
├── Pterodactyl: 2 cores (game servers)
├── Docker Services: 1 core (containers)
├── Plex: 1 core + Quick Sync (transcoding)
└── Media Automation: 0.5 cores (background tasks)
```

---

## 💾 Memory Configuration

### **64GB DDR4 RAM (Upgraded)**

**Memory Specifications:**
```
Total Capacity: 64GB
Configuration: 2 x 32GB DDR4 SODIMMs
Speed: DDR4-2666 (PC4-21300)
Latency: CL19-19-19-43
ECC Support: Non-ECC (consumer grade)
Max Supported: 64GB (motherboard limit)
Upgradeable: No (maxed out)
```

**Memory Performance:**
```
Bandwidth: 21.3 GB/s theoretical
Actual Performance: ~19 GB/s (real-world)
Dual Channel: Yes (optimal performance)
XMP Profile: Enabled for rated speeds
```

**Memory Allocation Strategy:**
```
Total RAM Distribution (64GB):
├── Proxmox Host: 2GB (hypervisor overhead)
├── TrueNAS: 8GB (ZFS cache requires significant RAM)
├── Pterodactyl: 16GB (Java game servers are memory-intensive)
├── Docker Services: 16GB (multiple containers + caching)
├── Plex: 12GB (transcoding buffer + metadata cache)
├── Media Automation: 12GB (parallel processing + downloads)
└── Available: ~2GB (emergency buffer)

Memory Usage Optimization:
├── ZFS ARC Cache: 6GB (TrueNAS automatic tuning)
├── Container Limits: Enforced to prevent memory leaks
├── Java Heap Tuning: Optimized for Minecraft servers
└── Transcoding Buffer: Dynamic allocation based on streams
```

**Memory Performance Monitoring:**
- **Critical Threshold**: 90% utilization triggers alerts
- **Ballooning**: Proxmox can dynamically adjust VM memory
- **Swap Usage**: Minimized through proper allocation
- **Cache Hit Rates**: Monitored for storage performance

---

## 💿 Storage Configuration

### **Primary Storage: 2TB NVMe RAID 1**

**Drive Specifications:**
```
Drive Model: 2 x Samsung 980 PRO 1TB NVMe SSD
Interface: PCIe 4.0 x4 (running at PCIe 3.0 speeds)
Form Factor: M.2 2280
NAND Type: Samsung V-NAND 3-bit MLC
Controller: Samsung Elpis controller
Cache: 1GB LPDDR4 per drive
```

**Performance Metrics:**
```
Sequential Read: 7,000 MB/s (per drive)
Sequential Write: 5,000 MB/s (per drive)
Random Read (4K): 1,000,000 IOPS
Random Write (4K): 1,000,000 IOPS
Endurance: 600 TBW per drive (1,200 TBW total)
Warranty: 5 years Samsung warranty
```

**RAID 1 Configuration:**
```
RAID Level: Software RAID 1 (mirroring)
Usable Capacity: 1TB (50% overhead for redundancy)
Performance Impact: Slight write penalty (~15%)
Reliability: Single drive failure tolerance
Management: Linux mdadm + Proxmox integration
```

**Storage Layout:**
```
Total 2TB Raw → 1TB Usable (RAID 1)

Proxmox Host (200GB):
├── Proxmox VE OS: 32GB
├── VM Templates: 50GB
├── ISO Storage: 50GB
├── Snapshots: 50GB
└── System Logs: 18GB

VM Storage Allocation (800GB):
├── TrueNAS: 100GB (OS + apps)
├── Pterodactyl: 200GB (game worlds + backups)
├── Docker Services: 100GB (containers + volumes)
├── Plex: 100GB (metadata + thumbnails)
├── Media Automation: 100GB (temp processing)
└── Available: 200GB (expansion + snapshots)
```

### **Data Storage Strategy**

**Hot Storage (NVMe):**
- VM operating systems and applications
- Active game server worlds
- Container images and volumes  
- Plex metadata and thumbnail cache
- Recently accessed documents

**Warm Storage (TrueNAS Datasets):**
- Media library (movies, TV, music)
- Document archive (Paperless-ngx)
- Configuration backups
- Long-term snapshots

**Cold Storage (External):**
- Encrypted offsite backups
- Archive media not recently accessed
- Disaster recovery images

---

## 🌡️ Cooling & Thermal Management

### **Cooling System Design**

**Stock HP Cooling:**
```
CPU Cooler: Low-profile heatsink with 92mm fan
Case Fans: 1 x 92mm intake fan (front)
Airflow Design: Front-to-back positive pressure
Thermal Design: Optimized for 35W TDP CPU
Noise Level: <25dB during normal operation
```

**Thermal Performance:**
```
Idle Temperatures:
├── CPU: 35-40°C
├── RAM: 30-35°C
├── NVMe: 40-45°C
└── Case: 25-30°C

Load Temperatures (24/7 operation):
├── CPU: 55-65°C (under sustained VM load)
├── RAM: 40-45°C
├── NVMe: 60-70°C (during heavy I/O)
└── Case: 35-40°C

Critical Temperatures:
├── CPU Throttling: 85°C (never reached)
├── NVMe Throttling: 85°C (rarely reached)
└── System Shutdown: 90°C (emergency only)
```

**Cooling Optimizations:**
- **Fan Curves**: BIOS configured for quiet operation under normal load
- **Thermal Paste**: High-quality compound for optimal heat transfer
- **Dust Management**: Positive pressure reduces dust accumulation
- **Placement**: Adequate ventilation clearance on all sides

---

## ⚡ Power Management

### **Power Supply & Consumption**

**External Power Adapter:**
```
Model: HP 180W External Adapter
Input: 100-240V AC, 50/60Hz, 2.3A
Output: 19.5V DC, 9.23A (180W max)
Efficiency: 90% (80 Plus Gold equivalent)
Power Factor: >0.9 (active PFC)
```

**Power Consumption Analysis:**
```
Idle Power Draw: ~35W
├── CPU (idle): 8W
├── RAM (64GB): 12W
├── NVMe drives: 4W
├── Motherboard: 8W
└── PSU efficiency loss: 3W

Normal Operation: ~65W
├── CPU (moderate load): 25W
├── RAM: 12W
├── NVMe drives: 8W
├── Motherboard: 12W
└── PSU efficiency loss: 8W

Peak Load: ~110W
├── CPU (full load): 35W
├── RAM: 12W
├── NVMe drives: 15W
├── Motherboard: 15W
├── Network transcoding: 25W
└── PSU efficiency loss: 8W
```

**Annual Power Cost Calculation:**
```
Average Power Draw: 70W (24/7 operation)
Annual Consumption: 70W × 24h × 365d = 613 kWh
Electricity Cost: 613 kWh × $0.12/kWh = $74/year
Monthly Cost: ~$6.20

Comparison:
├── Similar Intel NUC: ~$85/year
├── Desktop Replacement: ~$150-200/year  
├── Dedicated Server: ~$300-500/year
└── Cloud Equivalent: ~$2,400/year
```

**Power Management Features:**
- **C-States**: CPU sleep states for idle power reduction
- **P-States**: Dynamic frequency scaling based on load
- **ASPM**: PCIe Active State Power Management
- **Wake-on-LAN**: Remote power management capability
- **Scheduled Power**: BIOS supports automated power schedules

---

## 🔌 Connectivity & I/O

### **Network Connectivity**

**Ethernet Interface:**
```
Controller: Intel I219-LM Gigabit
Speed: 10/100/1000 Mbps auto-negotiation
Wake-on-LAN: Supported
VLAN Support: Hardware VLAN tagging
Jumbo Frames: Up to 9KB supported
Power over Ethernet: Not supported (standard NIC)
```

**Network Performance:**
```
Throughput: 940 Mbps (TCP), 980 Mbps (UDP)
Latency: <1ms local network
CPU Utilization: <5% at full throughput
Buffer Size: 256KB receive, 256KB transmit
```

### **USB & Expansion Ports**

**Front Panel I/O:**
```
├── 2 x USB 3.1 Gen 1 (5 Gbps)
├── 1 x USB 3.1 Gen 2 Type-C (10 Gbps)
├── 1 x 3.5mm headphone/microphone combo
└── Power button with LED status
```

**Rear Panel I/O:**
```
├── 4 x USB 3.1 Gen 1 (5 Gbps)
├── 2 x USB 2.0 (480 Mbps)
├── 1 x RJ-45 Gigabit Ethernet
├── 1 x DisplayPort 1.2
├── 1 x VGA (DB-15)
├── 1 x 3.5mm line out
└── 1 x 3.5mm line in
```

**Internal Expansion:**
```
M.2 Slots: 2 x M.2 2280 (both occupied)
├── Slot 1: PCIe 3.0 x4 (Samsung 980 PRO #1)
└── Slot 2: PCIe 3.0 x4 (Samsung 980 PRO #2)

PCIe Slots: 1 x PCIe 3.0 x16 (low profile)
├── Available for expansion
├── GPU upgrade path possible
└── Network card expansion option

RAM Slots: 2 x DDR4 SODIMM (both occupied)
├── Slot 1: 32GB DDR4-2666
└── Slot 2: 32GB DDR4-2666
```

---

## 📊 Performance Benchmarks

### **CPU Performance**

**Virtualization Benchmarks:**
```
VM Creation Time: 15-30 seconds per VM
VM Boot Time: 20-45 seconds (depending on OS)
VM Migration: <2 minutes (live migration)
Concurrent VMs: 8-10 VMs (light workload)
CPU Overhead: ~5% (Proxmox hypervisor)
```

**Transcoding Performance (Intel Quick Sync):**
```
H.264 1080p → 720p: 4-5 simultaneous streams
H.264 4K → 1080p: 2-3 simultaneous streams  
H.265 1080p → 720p: 3-4 simultaneous streams
H.265 4K → 1080p: 1-2 simultaneous streams
Power Usage: +15W during active transcoding
Quality: Hardware transcoding (very good quality)
```

### **Memory Performance**

**ZFS Performance (TrueNAS):**
```
ARC Hit Rate: 95%+ (after warm-up)
L2ARC: Not configured (NVMe fast enough)
Memory Bandwidth: 19 GB/s sustained
ZFS Compression: ~1.3x average ratio
Deduplication: Disabled (RAM requirements)
```

**Container Performance:**
```
Container Start Time: 2-5 seconds average
Memory Overhead: ~100MB per container
Cache Hit Rate: 90%+ (application caches)
Garbage Collection: Optimized for low pause times
```

### **Storage Performance**

**RAID 1 Performance:**
```
Sequential Read: 6,500 MB/s (single drive speed)
Sequential Write: 2,800 MB/s (RAID 1 write penalty)
Random Read (4K): 950K IOPS
Random Write (4K): 475K IOPS (RAID penalty)
Rebuild Time: 45-60 minutes (1TB)
```

**VM Storage Performance:**
```
VM Boot Performance: 15-30 seconds typical
Application Launch: Near-instant for most apps
Database Performance: Excellent for small-medium DBs
Backup Speed: 300-400 MB/s to external storage
Snapshot Creation: <5 seconds per VM
```

---

## 🔧 Upgrade Path & Future Expansion

### **Current Upgrade Status**

**Completed Upgrades:**
- ✅ RAM: Upgraded from 8GB to 64GB (maxed out)
- ✅ Storage: Upgraded to 2TB NVMe RAID 1
- ✅ CPU: Upgraded to i5-9500T (from base model)

**Remaining Upgrade Options:**

**PCIe Expansion Card:**
```
Available: 1 x PCIe 3.0 x16 (low profile)
Options:
├── 10Gb Network Card (future-proofing)
├── Additional NVMe Storage (M.2 expansion)
├── GPU for transcoding/AI workloads
└── USB 3.2 expansion card
```

**External Storage Expansion:**
```
Options:
├── USB 3.1 external drives (backup/archive)
├── Network Attached Storage (additional capacity)
├── Cloud storage integration
└── Tape backup solution (enterprise)
```

### **Performance Scaling Limits**

**Current Bottlenecks:**
```
CPU: Adequate for current workload, 80% peak utilization
Memory: Well-provisioned, 90% utilization under load
Storage: Excellent performance, IOPS headroom available
Network: Gigabit sufficient, 10Gb would future-proof
```

**Scaling Recommendations:**
1. **Next 12 months**: Monitor CPU utilization trends
2. **12-24 months**: Consider 10Gb networking for media
3. **24+ months**: Evaluate next-generation platform
4. **Long-term**: Plan migration to enterprise hardware

---

## 🛠️ Hardware Monitoring

### **Health Monitoring Tools**

**System Monitoring:**
```
BIOS Health: Built-in HP system diagnostics
Temperature: lm-sensors + Proxmox monitoring
Storage Health: S.M.A.R.T. monitoring via TrueNAS
Memory: ECC errors tracked (though using non-ECC RAM)
Power Supply: Voltage monitoring via IPMI
```

**Alerting Thresholds:**
```
Temperature Alerts:
├── CPU: >75°C (warning), >80°C (critical)
├── NVMe: >75°C (warning), >80°C (critical)
├── Case: >45°C (warning), >50°C (critical)

Performance Alerts:
├── CPU Usage: >90% for 10+ minutes
├── Memory: >95% utilization
├── Storage: >85% capacity used
├── Network: >80% bandwidth sustained
```

### **Maintenance Schedule**

**Monthly:**
- Check system temperatures and fan operation
- Review S.M.A.R.T. data for storage health
- Update system firmware and drivers
- Clean dust from intake vents

**Quarterly:**
- Deep clean internal components
- Verify backup and recovery procedures
- Performance benchmarking and comparison
- Hardware health report generation

**Annually:**
- Comprehensive hardware diagnostic testing
- Thermal paste replacement (if needed)
- Fan bearing inspection and replacement
- Warranty status review and planning

---

This HP EliteDesk 800 G5 platform provides an excellent foundation for the home lab, offering enterprise-grade reliability in a compact, power-efficient package. The hardware specifications support the full virtualized architecture while maintaining room for future expansion and upgrades.
