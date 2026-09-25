# Part 71: Enterprise Network Design

## สารบัญ
1. [Enterprise Network Requirements](#requirements)
2. [Three-Tier Architecture](#three-tier)
3. [Core, Distribution, Access Layers](#layers)
4. [VLAN Design](#vlan)
5. [Redundancy Planning](#redundancy)
6. [Security Zones](#security)
7. [QoS Design](#qos)
8. [Documentation](#documentation)
9. [Change Management](#change-mgmt)
10. [Lab: Enterprise Network Design](#lab)

---

## 1. Enterprise Network Requirements {#requirements}

### Business Requirements Analysis

| Requirement | ความสำคัญ | Solution |
|-------------|----------|---------|
| High Availability | Critical | Redundant links/devices |
| Scalability | High | Modular design |
| Security | Critical | Segmentation, ACLs |
| Performance | High | QoS, FastPath |
| Manageability | Medium | Centralized management |
| Cost | Medium | Efficient design |

### Network Scale Categories

```
Small Enterprise:     < 100 users    - 2-tier architecture
Medium Enterprise:  100-500 users    - 3-tier architecture  
Large Enterprise:   500-5000 users   - Full 3-tier + redundancy
Campus Network:    > 5000 users      - Hierarchical + redundancy
```

---

## 2. Three-Tier Architecture {#three-tier}

```
Internet
    │
    ├── ISP 1 (Primary)    ├── ISP 2 (Secondary)
    │                       │
    └───────────────────────┘
               │
    ┌──────────▼──────────┐
    │    CORE LAYER       │
    │  (MikroTik CCR)     │
    │  - BGP routing      │
    │  - Core switching   │
    │  - WAN termination  │
    └──────────┬──────────┘
               │ Aggregation Links
    ┌──────────▼──────────┐
    │  DISTRIBUTION LAYER │
    │   (CRS/CSS switch)  │
    │  - Inter-VLAN       │
    │  - Policy-based     │
    │  - QoS              │
    └──────────┬──────────┘
               │ Access Links
    ┌──────────▼──────────┐
    │    ACCESS LAYER     │
    │   (CRS/RB switch)   │
    │  - VLAN assignment  │
    │  - Port security    │
    │  - PoE              │
    └─────────────────────┘
```

---

## 3. Core Layer Configuration {#layers}

```bash
# Core Router Configuration (MikroTik CCR2004)

# Interface setup
/interface bridge
add name=bridge-core comment="Core bridge"

/interface bridge port
add bridge=bridge-core interface=ether1 comment="To ISP1"
add bridge=bridge-core interface=ether2 comment="To ISP2"
add bridge=bridge-core interface=ether3 comment="To Distribution-1"
add bridge=bridge-core interface=ether4 comment="To Distribution-2"

# VLAN configuration at core
/interface vlan
add interface=bridge-core name=vlan-mgmt vlan-id=10
add interface=bridge-core name=vlan-servers vlan-id=20
add interface=bridge-core name=vlan-users vlan-id=100
add interface=bridge-core name=vlan-guest vlan-id=200
add interface=bridge-core name=vlan-voip vlan-id=300
add interface=bridge-core name=vlan-security vlan-id=400

# IP addressing
/ip address
add address=10.0.0.1/24 interface=vlan-mgmt comment="Management"
add address=10.10.0.1/24 interface=vlan-servers comment="Servers"
add address=10.100.0.1/22 interface=vlan-users comment="User network"
add address=10.200.0.1/24 interface=vlan-guest comment="Guest WiFi"
add address=10.30.0.1/24 interface=vlan-voip comment="VoIP"

# OSPF for internal routing
/routing ospf instance
add name=default router-id=10.0.0.1

/routing ospf area
add instance=default name=backbone area-id=0.0.0.0

/routing ospf interface-template
add area=backbone interfaces=vlan-servers networks=10.10.0.0/24
add area=backbone interfaces=vlan-users networks=10.100.0.0/22
```

---

## 4. VLAN Design {#vlan}

### VLAN Planning Table

| VLAN ID | Name | Subnet | Purpose | Security Level |
|---------|------|--------|---------|---------------|
| 10 | Management | 10.0.0.0/24 | Network devices | High |
| 20 | Servers | 10.10.0.0/24 | Internal servers | High |
| 30 | DMZ | 10.20.0.0/24 | Public-facing servers | Medium |
| 100 | Users-1 | 10.100.0.0/23 | Office users floor 1-2 | Medium |
| 101 | Users-2 | 10.102.0.0/23 | Office users floor 3-4 | Medium |
| 200 | Guest | 10.200.0.0/24 | Guest WiFi | Low |
| 300 | VoIP | 10.30.0.0/24 | IP Phones | High |
| 400 | Security | 10.40.0.0/24 | CCTV, Access control | High |
| 500 | IoT | 10.50.0.0/24 | IoT devices | Low |

```bash
# Distribution switch VLAN configuration
/interface bridge vlan
add bridge=bridge1 tagged=ether1,ether2 untagged=ether3 vlan-ids=10
add bridge=bridge1 tagged=ether1,ether2 untagged=ether4 vlan-ids=20
add bridge=bridge1 tagged=ether1,ether2 untagged=ether5-8 vlan-ids=100
add bridge=bridge1 tagged=ether1,ether2 untagged=sfp1 vlan-ids=300

# VLAN interface for routing
/interface bridge
add name=bridge1 vlan-filtering=yes frame-types=admit-only-vlan-tagged

# Inter-VLAN routing rules
/ip firewall filter
# Allow servers to be accessed from Users
add chain=forward in-interface=vlan-users out-interface=vlan-servers \
    dst-port=80,443,22,3389 protocol=tcp action=accept \
    comment="Users to Servers"

# Deny Users to Management
add chain=forward in-interface=vlan-users out-interface=vlan-mgmt \
    action=drop comment="Block users from management"

# Allow VoIP traffic
add chain=forward in-interface=vlan-voip protocol=udp \
    dst-port=5060,10000-20000 action=accept \
    comment="Allow VoIP"
```

---

## 5. Redundancy Planning {#redundancy}

```bash
# STP/RSTP configuration for loop prevention
/interface bridge
set bridge1 protocol-mode=rstp priority=0x1000

# VRRP for gateway redundancy
/interface vrrp
add interface=vlan-users name=vrrp-users vrid=100 priority=150 \
    preemption-mode=yes

/ip address
add address=10.100.0.2/24 interface=vlan-users comment="Active VRRP"
add address=10.100.0.1/24 interface=vrrp-users comment="Virtual IP"

# Bonding for uplink redundancy
/interface bonding
add name=bond-uplink slaves=ether1,ether2 mode=active-backup \
    primary=ether1 comment="Uplink bond to core"

# ECMP routing
/ip route
add dst-address=0.0.0.0/0 gateway=192.168.1.1 distance=1 comment="ISP1"
add dst-address=0.0.0.0/0 gateway=192.168.2.1 distance=2 comment="ISP2 backup"

# Check gateway scripts
/ip route check-gateway=ping
```

---

## 6. Security Zones {#security}

```bash
# Firewall security zones
/ip firewall address-list
add list=internal-networks address=10.0.0.0/8
add list=internal-networks address=172.16.0.0/12
add list=internal-networks address=192.168.0.0/16
add list=management-hosts address=10.0.0.0/24
add list=server-subnet address=10.10.0.0/24
add list=dmz-subnet address=10.20.0.0/24

# Security zone rules
/ip firewall filter

# Chain: input (to router itself)
add chain=input connection-state=established,related action=accept
add chain=input connection-state=invalid action=drop
add chain=input src-address-list=management-hosts action=accept comment="Allow management"
add chain=input protocol=icmp action=accept comment="Allow ping"
add chain=input action=drop comment="Drop all others to router"

# Chain: forward (through router)
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop

# DMZ rules - only specific ports from outside
add chain=forward in-interface=ether1 out-interface=vlan-dmz \
    dst-port=80,443 protocol=tcp action=accept comment="HTTP/S to DMZ"

# Block DMZ to internal
add chain=forward in-interface=vlan-dmz \
    dst-address-list=internal-networks action=drop \
    comment="Block DMZ to internal"

# Log suspicious activity
add chain=forward src-address-list=suspicious action=add-dst-to-address-list \
    address-list=blocked address-list-timeout=1h comment="Block suspicious"
add chain=forward src-address-list=suspicious action=log log-prefix="SUSPICIOUS:"
add chain=forward src-address-list=suspicious action=drop
```

---

## 7. QoS Design {#qos}

```bash
# QoS เพื่อ prioritize traffic
/queue type
add name=pcq-upload kind=pcq pcq-classifier=src-address
add name=pcq-download kind=pcq pcq-classifier=dst-address

# Mangle marking
/ip firewall mangle
# VoIP traffic - highest priority
add chain=prerouting protocol=udp dst-port=5060 \
    action=mark-packet new-packet-mark=voip passthrough=no
add chain=prerouting protocol=udp dst-port=10000-20000 \
    action=mark-packet new-packet-mark=voip passthrough=no

# Video conferencing
add chain=prerouting dst-port=3478,19302-19309 protocol=udp \
    action=mark-packet new-packet-mark=video passthrough=no

# Interactive (SSH, RDP, management)
add chain=prerouting dst-port=22,23,3389 protocol=tcp \
    action=mark-packet new-packet-mark=interactive passthrough=no

# Bulk transfer (backup, software updates)
add chain=prerouting dst-port=21 protocol=tcp \
    action=mark-packet new-packet-mark=bulk passthrough=no

# Queue tree with bandwidth allocation
/queue tree
add name=root-download parent=global max-limit=1G

add name=voip-down parent=root-download packet-mark=voip \
    limit-at=50M max-limit=100M priority=1
    
add name=video-down parent=root-download packet-mark=video \
    limit-at=100M max-limit=300M priority=2

add name=interactive-down parent=root-download packet-mark=interactive \
    limit-at=50M max-limit=200M priority=3

add name=bulk-down parent=root-download packet-mark=bulk \
    limit-at=100M max-limit=500M priority=8
```

---

## 8. Documentation {#documentation}

### Network Documentation Template

```markdown
# Enterprise Network Design Document

## 1. Overview
- Organization: 
- Project: 
- Date: 
- Author: 
- Version: 

## 2. Network Topology
[Attach diagram]

## 3. IP Addressing Scheme
| VLAN | Network | Gateway | DHCP Range |
|------|---------|---------|------------|
| 10   | 10.0.0.0/24 | 10.0.0.1 | N/A (static) |
| 100  | 10.100.0.0/22 | 10.100.0.1 | 10.100.0.10-10.103.0.250 |

## 4. Device Inventory
| Hostname | Model | IP | Location | Role |
|----------|-------|-----|----------|------|
| core-1   | CCR2004 | 10.0.0.1 | DC | Core Router |

## 5. VLAN Design
[Reference VLAN table above]

## 6. Security Policies
[Firewall rules summary]

## 7. QoS Policies
[Traffic priority table]

## 8. Redundancy Design
[Failover procedures]

## 9. Change Log
[Version history]
```

---

## 9. Lab: Enterprise Network Design {#lab}

### Lab Topology

```
ISP 1 (100Mbps)     ISP 2 (100Mbps)
     │                    │
     └────────────────────┘
              │
         [Core Router]
          CCR2004-1G
         10.0.0.1/24
              │
     ┌────────┴────────┐
     │                 │
[Dist Switch 1]  [Dist Switch 2]
  CRS326           CRS326
  10.0.0.2          10.0.0.3
     │                 │
  [Access 1]       [Access 2]
  CRS112            CRS112
  Floor 1-2         Floor 3-4
```

### Implementation Checklist

- [ ] Core router BGP ทำงาน
- [ ] All VLANs created และ routing ทำงาน
- [ ] VRRP gateway redundancy ทดสอบแล้ว
- [ ] Firewall rules block unauthorized access
- [ ] QoS VoIP ได้ priority สูงสุด
- [ ] STP ไม่มี loops
- [ ] Documentation ครบถ้วน
- [ ] Change management process กำหนดแล้ว

> **Tip:** ใช้ GNS3 หรือ EVE-NG เพื่อ simulate ก่อน deploy จริง

---

## Summary

Part นี้ครอบคลุม:
- **Three-tier architecture** สำหรับ enterprise
- **VLAN design** พร้อม security zones
- **Redundancy** ด้วย VRRP และ bonding
- **QoS** สำหรับ VoIP และ video
- **Security zones** พร้อม firewall rules
- **Documentation** standards

---

[← Part 70: SMS Notification](part-070-sms-notification.md) | [Part 72: High Availability →](part-072-high-availability.md)
