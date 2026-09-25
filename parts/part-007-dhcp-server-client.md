# Part 7: DHCP Server & Client

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner to Intermediate  
> **เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

1. [DHCP Fundamentals](#1-dhcp-fundamentals)
2. [DHCP Server Configuration](#2-dhcp-server-configuration)
3. [DHCP Client Configuration](#3-dhcp-client-configuration)
4. [DHCP Relay](#4-dhcp-relay)
5. [Static Leases](#5-static-leases)
6. [DHCP Options](#6-dhcp-options)
7. [Multiple DHCP Servers](#7-multiple-dhcp-servers)
8. [DHCP Server on Bridge](#8-dhcp-server-on-bridge)
9. [Lease Monitoring and Management](#9-lease-monitoring-and-management)
10. [Scripts on DHCP Events](#10-scripts-on-dhcp-events)
11. [Lab: Complete DHCP Setup](#11-lab-complete-dhcp-setup)
12. [Summary](#12-summary)

---

## 1. DHCP Fundamentals

### 1.1 DHCP Process (DORA)

```
DHCP Discovery Process:

Client                              Server
  │                                    │
  │──── DISCOVER (broadcast) ─────────>│
  │     "Who has an IP for me?"         │
  │                                    │
  │<─── OFFER (unicast/broadcast) ─────│
  │     "I have 192.168.1.10 for you"   │
  │                                    │
  │──── REQUEST (broadcast) ───────────>│
  │     "I want 192.168.1.10"           │
  │                                    │
  │<─── ACK (unicast/broadcast) ────────│
  │     "OK, it's yours for 1 hour"     │
  │                                    │
  │              ... (lease time) ...   │
  │                                    │
  │──── REQUEST (unicast, renew) ──────>│
  │     "Can I keep my IP?"             │
  │                                    │
  │<─── ACK ───────────────────────────│
  │     "Yes, renewed for 1 more hour"  │
```

### 1.2 DHCP Lease States

```
Lease States:
├── bound      - Client ได้ IP แล้ว กำลังใช้งาน
├── waiting    - รอ client กลับมา (lease expire)
├── offered    - Server offer แล้ว รอ request
└── expired    - Lease หมดอายุ
```

### 1.3 DHCP Components ใน RouterOS

```
RouterOS DHCP Components:
├── /ip dhcp-server           - DHCP server config
├── /ip dhcp-server network   - Network/subnet config
├── /ip dhcp-server lease     - Lease management
├── /ip dhcp-server config    - Global DHCP settings
├── /ip dhcp-client           - DHCP client
├── /ip dhcp-relay            - DHCP relay agent
└── /ip pool                  - IP address pools
```

---

## 2. DHCP Server Configuration

### 2.1 Basic DHCP Server Setup

```bash
# Step 1: สร้าง IP Pool
/ip pool add \
    name=dhcp-pool \
    ranges=192.168.1.100-192.168.1.200

# Step 2: สร้าง DHCP Server
/ip dhcp-server add \
    name=dhcp-server-lan \
    interface=ether2 \
    address-pool=dhcp-pool \
    lease-time=1h \
    disabled=no

# Step 3: ตั้ง Network Settings
/ip dhcp-server network add \
    address=192.168.1.0/24 \
    gateway=192.168.1.1 \
    dns-server=8.8.8.8,8.8.4.4 \
    ntp-server=time.google.com \
    domain=local \
    comment="LAN Network"

# ตรวจสอบ
/ip dhcp-server print
/ip dhcp-server network print
/ip pool print
```

### 2.2 DHCP Server Setup ผ่าน Wizard

```bash
# RouterOS มี wizard สำหรับ DHCP setup
/ip dhcp-server setup

# Wizard จะถาม:
# Select interface to run DHCP server on: ether2
# Select network for DHCP addresses: 192.168.1.0/24
# Select gateway for given network: 192.168.1.1
# Select pool of ip addresses given out by DHCP: 192.168.1.100-192.168.1.200
# Select DNS servers: 8.8.8.8
# Select lease time: 00:10:00 (10 minutes)

# Wizard จะสร้าง pool, server, network อัตโนมัติ
```

### 2.3 DHCP Server Parameters

```bash
# DHCP Server options ที่สำคัญ
/ip dhcp-server set dhcp-server-lan \
    lease-time=8h \                    # ระยะเวลา lease
    max-lease-time=1d \                # ระยะเวลา lease สูงสุด
    ping-check=yes \                   # Ping ก่อน offer IP
    ping-timeout=300ms \               # Timeout สำหรับ ping
    use-radius=no \                    # ไม่ใช้ RADIUS
    conflict-detection=yes \           # ตรวจสอบ IP conflict
    insert-queue-before=0 \            # Queue position
    authoritative=yes                  # เป็น authoritative server
```

### 2.4 Network Settings ละเอียด

```bash
# Network settings พร้อม options ทั้งหมด
/ip dhcp-server network add \
    address=192.168.1.0/24 \
    gateway=192.168.1.1 \
    dns-server=1.1.1.1,8.8.8.8 \
    ntp-server=216.239.35.0 \          # Google NTP
    wins-server=192.168.1.10 \          # Windows DNS/WINS (option 44)
    domain=office.local \               # Domain name (option 15)
    boot-file-name=pxelinux.0 \         # PXE boot file (option 67)
    next-server=192.168.1.10 \          # TFTP server (option 66)
    netmask=255.255.255.0 \             # Subnet mask (auto จาก address)
    comment="Office LAN"
```

---

## 3. DHCP Client Configuration

### 3.1 Basic DHCP Client

```bash
# เพิ่ม DHCP client บน WAN interface
/ip dhcp-client add \
    interface=ether1 \
    disabled=no \
    add-default-route=yes \
    default-route-distance=1 \
    use-peer-dns=yes \
    comment="WAN DHCP"

# ดู status
/ip dhcp-client print

# Output:
# Flags: X - disabled, I - invalid
#  #   INTERFACE  USE-PEER-DNS  ADD-DEFAULT-ROUTE  STATUS
#  0   ether1     yes           yes                bound

# ดูรายละเอียด
/ip dhcp-client print detail

# Output (detail):
#  0   interface=ether1
#      add-default-route=yes
#      default-route-distance=1
#      use-peer-dns=yes
#      status=bound
#      address=203.0.113.10/24
#      gateway=203.0.113.1
#      expires-after=11h59m58s
#      dns-server=203.0.113.53
#      dhcp-server=203.0.113.1
```

### 3.2 DHCP Client Management

```bash
# Release IP (ปล่อย IP กลับ)
/ip dhcp-client release [find interface=ether1]

# Renew IP (ขอ IP ใหม่)
/ip dhcp-client renew [find interface=ether1]

# Disable DHCP client
/ip dhcp-client disable [find interface=ether1]

# ดู DHCP options ที่ได้รับ
/ip dhcp-client option print

# ดู routes ที่ DHCP เพิ่มมาให้
/ip route print where dynamic=yes
```

### 3.3 DHCP Client with Script

```bash
# รัน script เมื่อได้ IP ใหม่
/ip dhcp-client add \
    interface=ether1 \
    disabled=no \
    script="/system script run update-dns"

# Script ที่รันเมื่อ DHCP bound/renewed
/system script add name=update-dns source={
    :local ip [/ip dhcp-client get [find interface=ether1] address]
    :log info ("New WAN IP: " . $ip)
    # อัปเดต DNS หรือ firewall rules ตาม IP ใหม่
}
```

---

## 4. DHCP Relay

### 4.1 DHCP Relay คืออะไร

```
DHCP Relay ใช้เมื่อ:
- DHCP Server อยู่ใน subnet ต่างกัน
- ต้องการ centralized DHCP server
- ISP ให้บริการ DHCP centrally

Without Relay:
[Client] ─── broadcast ───> [Router] ─X─ (ไม่ผ่าน router) ─> [DHCP Server]

With Relay:
[Client] ─── broadcast ───> [Router/Relay] ─── unicast ───> [DHCP Server]
                             │ relay agent transforms broadcast to unicast
```

### 4.2 DHCP Relay Setup

```bash
# Topology:
# Client (192.168.1.x) → Router (Relay) → DHCP Server (10.0.0.10)

# บน Relay Router:
/ip dhcp-relay add \
    name=relay-lan1 \
    interface=ether2 \              # Interface ที่ clients อยู่
    dhcp-server=10.0.0.10 \        # IP ของ DHCP server
    local-address=192.168.1.1 \    # Local IP ของ relay
    disabled=no

# ดู relay
/ip dhcp-relay print

# บน DHCP Server (ต้องรู้ subnet ของ client):
/ip dhcp-server network add \
    address=192.168.1.0/24 \       # Client subnet
    gateway=192.168.1.1 \          # Client gateway
    dns-server=8.8.8.8

/ip pool add name=pool-remote ranges=192.168.1.50-192.168.1.200

/ip dhcp-server add \
    name=dhcp-remote \
    interface=ether1 \             # Interface ของ server (ไม่ใช่ client interface)
    relay=192.168.1.1 \            # IP ของ relay agent
    address-pool=pool-remote \
    disabled=no
```

---

## 5. Static Leases

### 5.1 สร้าง Static Lease

```bash
# วิธีที่ 1: สร้างใหม่
/ip dhcp-server lease add \
    address=192.168.1.10 \
    mac-address=AA:BB:CC:DD:EE:01 \
    server=dhcp-server-lan \
    comment="Printer-HP-Main"

# วิธีที่ 2: Convert dynamic lease เป็น static
# ดู dynamic leases
/ip dhcp-server lease print

# Convert lease ที่ index 2 เป็น static
/ip dhcp-server lease make-static [find address=192.168.1.15]
# หรือ
/ip dhcp-server lease make-static 2

# ดู leases
/ip dhcp-server lease print
```

### 5.2 จัดการ Static Leases จำนวนมาก

```bash
# Import static leases จาก CSV (ด้วย script)
# สมมติว่ามี file: leases.rsc
# เนื้อหา file:
# /ip dhcp-server lease add address=192.168.1.10 mac-address=AA:BB:CC:DD:EE:01 comment=PC01
# /ip dhcp-server lease add address=192.168.1.11 mac-address=AA:BB:CC:DD:EE:02 comment=PC02

# Import
/import file-name=leases.rsc

# Export static leases
/ip dhcp-server lease export file=static-leases

# ดู static leases เท่านั้น
/ip dhcp-server lease print where dynamic=no
```

### 5.3 Static Lease Options

```bash
# Static lease พร้อม custom options
/ip dhcp-server lease add \
    address=192.168.1.20 \
    mac-address=BB:CC:DD:EE:FF:01 \
    comment="VoIP Phone" \
    always-broadcast=yes \        # บางครั้ง phones ต้องการ broadcast
    block-access=no               # ไม่ block access

# Block lease (deny DHCP)
/ip dhcp-server lease add \
    mac-address=XX:XX:XX:XX:XX:XX \
    block-access=yes \
    comment="Blocked device"
```

---

## 6. DHCP Options

### 6.1 Standard DHCP Options

| Option | หมายเลข | Description |
|--------|---------|-------------|
| Subnet Mask | 1 | Subnet mask |
| Router | 3 | Default gateway |
| DNS Server | 6 | DNS servers |
| Domain Name | 15 | DNS domain |
| NTP Server | 42 | NTP servers |
| NetBIOS Name Server | 44 | WINS server |
| NetBIOS Scope | 47 | NetBIOS scope |
| TFTP Server | 66 | TFTP boot server |
| Boot File | 67 | Boot file name |
| DHCP Client ID | 61 | Client identifier |
| Classless Static Route | 121 | Static routes |
| Vendor-specific | 43 | Vendor-specific info |
| DHCP Agent Circuit ID | 82 | Relay agent info |

### 6.2 DHCP Option 121 (Classless Static Routes)

```bash
# Option 121: ส่ง static routes ไปให้ DHCP clients
# Useful สำหรับ VPN split tunneling

# สร้าง custom DHCP option
/ip dhcp-server option add \
    name=classless-routes \
    code=121 \
    value="0x" . \
    "18" . "C0A80A" . "C0A80101" . \
    # subnet=192.168.10.0/24 via 192.168.1.1
    "00" . "C0A80101"
    # default via 192.168.1.1

# หรือใช้ option sets
/ip dhcp-server option sets add \
    name=corporate-routes \
    options=classless-routes

# Apply ไปยัง network
/ip dhcp-server network set [find address=192.168.1.0/24] \
    dhcp-option-set=corporate-routes
```

### 6.3 DHCP Option 43 (Vendor-specific)

```bash
# Option 43 ใช้บ่อยสำหรับ:
# - VoIP phones (Cisco, Polycom, Yealink)
# - Access Points (Unifi)
# - PXE boot

# สำหรับ Ubiquiti UniFi Controller
/ip dhcp-server option add \
    name=unifi-controller \
    code=43 \
    value="0x01" . [ip-to-hex 192.168.1.5]
    # แนะนำ IP ของ UniFi Controller

# สำหรับ Yealink VoIP
/ip dhcp-server option add \
    name=yealink-prov-server \
    code=43 \
    value="'http://192.168.1.100/yealink/'"

# Apply
/ip dhcp-server network set [find address=192.168.1.0/24] \
    dhcp-option=unifi-controller
```

### 6.4 DHCP Option 82 (Relay Agent Info)

```bash
# Option 82 ใช้ในระบบ ISP สำหรับ subscriber identification
# Relay agent แทรก circuit-id และ remote-id

# ตั้งค่าบน relay:
/ip dhcp-relay set relay-lan1 \
    add-relay-info=yes \
    relay-info-remote-id="" \  # ปล่อยว่างสำหรับ auto
    relay-info-circuit-id=""   # หรือตั้งค่าเอง

# บน DHCP server อาจใช้ RADIUS สำหรับ verify option 82
```

### 6.5 Custom DHCP Options

```bash
# สร้าง option แบบ custom
/ip dhcp-server option add \
    name=my-option \
    code=200 \
    value="'custom-string'"

# Option types:
# 'string'      - text string (ต้องใส่ quotes)
# 0x0102       - hex bytes
# 192.168.1.1  - IP address
# s'domain.com' - string

# สร้าง option set
/ip dhcp-server option sets add \
    name=my-option-set \
    options=my-option,classless-routes

# Apply ไปยัง network
/ip dhcp-server network set 0 dhcp-option-set=my-option-set
```

---

## 7. Multiple DHCP Servers

### 7.1 Multiple Networks, Multiple Servers

```bash
# สำหรับ multiple VLANs/subnets

# Pools
/ip pool add name=pool-vlan10 ranges=192.168.10.10-192.168.10.200
/ip pool add name=pool-vlan20 ranges=192.168.20.10-192.168.20.200
/ip pool add name=pool-vlan30 ranges=192.168.30.10-192.168.30.200

# DHCP Servers
/ip dhcp-server add name=dhcp-vlan10 interface=vlan10 address-pool=pool-vlan10
/ip dhcp-server add name=dhcp-vlan20 interface=vlan20 address-pool=pool-vlan20
/ip dhcp-server add name=dhcp-vlan30 interface=vlan30 address-pool=pool-vlan30

# Networks
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1 dns-server=8.8.8.8
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=8.8.8.8
/ip dhcp-server network add address=192.168.30.0/24 gateway=192.168.30.1 dns-server=8.8.8.8
```

### 7.2 DHCP Failover / Redundancy

RouterOS ไม่มี DHCP failover built-in แต่ทำได้ด้วย IP pool overlapping:

```bash
# Primary server: 192.168.1.10-192.168.1.100
# Secondary server: 192.168.1.101-192.168.1.200
# (ทั้งคู่ทำงานพร้อมกัน แต่ pool ไม่ซ้ำกัน)

/ip pool add name=pool-primary ranges=192.168.1.10-192.168.1.100
/ip pool add name=pool-secondary ranges=192.168.1.101-192.168.1.200
```

---

## 8. DHCP Server on Bridge

### 8.1 Bridge DHCP Setup

```bash
# Topology:
# ether2, ether3, ether4 ─── bridge1 ─── DHCP Server

# สร้าง bridge
/interface bridge add name=bridge1

# เพิ่ม ports
/interface bridge port add bridge=bridge1 interface=ether2
/interface bridge port add bridge=bridge1 interface=ether3
/interface bridge port add bridge=bridge1 interface=ether4

# ตั้ง IP บน bridge
/ip address add address=192.168.1.1/24 interface=bridge1

# สร้าง DHCP บน bridge interface
/ip pool add name=pool-bridge ranges=192.168.1.10-192.168.1.200
/ip dhcp-server add name=dhcp-bridge interface=bridge1 address-pool=pool-bridge
/ip dhcp-server network add address=192.168.1.0/24 gateway=192.168.1.1
```

### 8.2 Bridge + VLAN + DHCP

```bash
# ใช้ Bridge VLAN Filtering

# สร้าง bridge
/interface bridge add name=bridge1 vlan-filtering=yes

# Trunk port (ไปยัง switch)
/interface bridge port add interface=ether1 bridge=bridge1 frame-types=admit-all

# Access port VLAN 10
/interface bridge port add interface=ether2 bridge=bridge1 pvid=10

# Access port VLAN 20
/interface bridge port add interface=ether3 bridge=bridge1 pvid=20

# VLAN config
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=bridge1,ether1 untagged=ether2
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=bridge1,ether1 untagged=ether3

# VLAN interfaces
/interface vlan add name=vlan10 vlan-id=10 interface=bridge1
/interface vlan add name=vlan20 vlan-id=20 interface=bridge1

# IP addresses
/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20

# DHCP servers
/ip pool add name=pool-vlan10 ranges=192.168.10.10-192.168.10.200
/ip pool add name=pool-vlan20 ranges=192.168.20.10-192.168.20.200

/ip dhcp-server add name=dhcp-vlan10 interface=vlan10 address-pool=pool-vlan10
/ip dhcp-server add name=dhcp-vlan20 interface=vlan20 address-pool=pool-vlan20

/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1
```

---

## 9. Lease Monitoring and Management

### 9.1 ดู Leases

```bash
# ดู leases ทั้งหมด
/ip dhcp-server lease print

# Output:
# Flags: X - disabled, R - radius, D - dynamic, B - blocked
#  # ADDRESS         MAC-ADDRESS        HOST-NAME   SERVER    STATUS
#  0 192.168.1.10    AA:BB:CC:DD:EE:01  PC01       dhcp-lan  bound
#  1 192.168.1.11    AA:BB:CC:DD:EE:02  Laptop     dhcp-lan  bound
#  2 192.168.1.12    AA:BB:CC:DD:EE:03             dhcp-lan  waiting

# ดู detail
/ip dhcp-server lease print detail

# ดู active leases
/ip dhcp-server lease print where status=bound

# ดู leases ของ server เฉพาะ
/ip dhcp-server lease print where server=dhcp-vlan10
```

### 9.2 Lease Statistics

```bash
# นับ leases
/ip dhcp-server lease print count-only

# ดู pool usage
/ip pool used print

# ดู จำนวน IPs ที่ใช้/เหลือ
:local total 191
:local used [/ip pool used print count-only]
:local free ($total - $used)
:put ("Pool usage: " . $used . "/" . $total . " (Free: " . $free . ")")
```

### 9.3 Lease Management

```bash
# ลบ lease ที่ expired
/ip dhcp-server lease remove [find status=expired]

# ลบ lease ที่ waiting
/ip dhcp-server lease remove [find status=waiting]

# Force release lease
/ip dhcp-server lease make-static [find address=192.168.1.10]
/ip dhcp-server lease remove [find address=192.168.1.10 dynamic=no]

# Block MAC address
/ip dhcp-server lease add mac-address=XX:XX:XX:XX:XX:XX block-access=yes

# Flush all dynamic leases (ระวัง! clients จะ disconnect)
/ip dhcp-server lease remove [find dynamic=yes]
```

### 9.4 DHCP Lease Notifications

```bash
# รับ notification เมื่อ client connect/disconnect
# ใช้ DHCP script (ดู Part 10)

# Monitor leases real-time
/ip dhcp-server lease print interval=5
```

---

## 10. Scripts on DHCP Events

### 10.1 DHCP Script Overview

```bash
# RouterOS รันสคริปต์เมื่อ DHCP events เกิดขึ้น
# ตั้งค่าบน DHCP server:

/ip dhcp-server set dhcp-server-lan \
    lease-script=my-dhcp-script

# Script จะได้รับ environment variables:
# $leaseBound    - 1 = bound, 0 = unbound
# $leaseActIP    - IP address ที่ assign
# $leaseActMAC   - MAC address ของ client
# $leaseServerName - ชื่อ DHCP server
# $leaseBoundType  - "bound" หรือ "expire"
```

### 10.2 Logging Script

```bash
# Script: log เมื่อ client connect/disconnect
/system script add name=dhcp-log source={
    :if ($leaseBound = 1) do={
        :log info ("DHCP BOUND: " . $leaseActIP . " MAC: " . $leaseActMAC)
    } else={
        :log info ("DHCP UNBOUND: " . $leaseActIP . " MAC: " . $leaseActMAC)
    }
}

/ip dhcp-server set dhcp-server-lan lease-script=dhcp-log
```

### 10.3 Firewall Script (Auto Add/Remove)

```bash
# Script: เพิ่ม/ลบ firewall address-list เมื่อ DHCP event
/system script add name=dhcp-firewall source={
    :if ($leaseBound = 1) do={
        # Client connect - เพิ่มใน whitelist
        /ip firewall address-list add \
            list=dhcp-clients \
            address=$leaseActIP \
            comment=$leaseActMAC
        :log info ("Added to whitelist: " . $leaseActIP)
    } else={
        # Client disconnect - ลบจาก whitelist
        /ip firewall address-list remove \
            [find list=dhcp-clients address=$leaseActIP]
        :log info ("Removed from whitelist: " . $leaseActIP)
    }
}

/ip dhcp-server set dhcp-server-lan lease-script=dhcp-firewall
```

### 10.4 DDNS Update Script

```bash
# Script: อัปเดต DNS record เมื่อ client ได้ IP
/system script add name=dhcp-dns-update source={
    :if ($leaseBound = 1) do={
        # หา hostname จาก DHCP lease
        :local hostname [/ip dhcp-server lease get \
            [find address=$leaseActIP] host-name]
        
        :if ($hostname != "") do={
            # เพิ่ม static DNS record
            /ip dns static remove [find name=$hostname]
            /ip dns static add \
                name=($hostname . ".local") \
                address=$leaseActIP \
                ttl=1h
            :log info ("DNS updated: " . $hostname . " = " . $leaseActIP)
        }
    }
}

/ip dhcp-server set dhcp-server-lan lease-script=dhcp-dns-update
```

### 10.5 Notification Script (Email/Log)

```bash
# Script: ส่ง email เมื่อมี client ใหม่
/system script add name=dhcp-notify source={
    :if ($leaseBound = 1) do={
        :local msg ("New client: IP=" . $leaseActIP . " MAC=" . $leaseActMAC)
        :log warning $msg
        
        # ส่ง email (ต้องตั้ง /tool e-mail ก่อน)
        /tool e-mail send \
            to=admin@example.com \
            subject="New DHCP Lease" \
            body=$msg
    }
}
```

---

## 11. Lab: Complete DHCP Setup

### Lab Topology

```
                Internet
                    │
           ┌────────┴────────┐
           │   MikroTik      │ 192.168.1.1/24
           │   Router        │ ether1=WAN, ether2=LAN
           └────────┬────────┘
                    │ LAN: 192.168.1.0/24
          ┌─────────┼──────────┐
          │         │          │
    ┌─────┴───┐ ┌───┴─────┐ ┌──┴──────┐
    │  PC1    │ │  PC2    │ │  Printer│
    │DHCP     │ │DHCP     │ │Static   │
    │.1.10-50 │ │.1.10-50 │ │.1.100   │
    └─────────┘ └─────────┘ └─────────┘
```

### Lab Step 1: สร้าง IP Pool

```bash
# Pool สำหรับ dynamic clients
/ip pool add \
    name=pool-lan \
    ranges=192.168.1.10-192.168.1.50 \
    comment="LAN Dynamic Pool"

# ตรวจสอบ
/ip pool print
```

### Lab Step 2: ตั้ง DHCP Server

```bash
# สร้าง DHCP Server
/ip dhcp-server add \
    name=dhcp-lan \
    interface=ether2 \
    address-pool=pool-lan \
    lease-time=8h \
    ping-check=yes \
    disabled=no

# ตั้ง Network
/ip dhcp-server network add \
    address=192.168.1.0/24 \
    gateway=192.168.1.1 \
    dns-server=1.1.1.1,8.8.8.8 \
    domain=home.local \
    comment="Home Network"

# ตรวจสอบ
/ip dhcp-server print
/ip dhcp-server network print
```

### Lab Step 3: Static Lease สำหรับ Printer

```bash
# Printer มี MAC address: AA:BB:CC:DD:EE:FF
/ip dhcp-server lease add \
    address=192.168.1.100 \
    mac-address=AA:BB:CC:DD:EE:FF \
    server=dhcp-lan \
    comment="HP Printer Main Office"

# ตรวจสอบ
/ip dhcp-server lease print
```

### Lab Step 4: DHCP Script

```bash
# สร้าง script สำหรับ logging
/system script add name=dhcp-script source={
    :local action
    :if ($leaseBound = 1) do={ :set action "CONNECT" } else={ :set action "DISCONNECT" }
    :log info ("DHCP " . $action . ": IP=" . $leaseActIP . " MAC=" . $leaseActMAC)
}

# เชื่อม script กับ DHCP server
/ip dhcp-server set dhcp-lan lease-script=dhcp-script
```

### Lab Step 5: Verification

```bash
# ทดสอบด้วยการ connect client
# ดู leases
/ip dhcp-server lease print

# ดู log
/log print where topics~"dhcp"

# ทดสอบ ping จาก router ไปยัง client
/ping 192.168.1.10 count=3

# ตรวจสอบ pool usage
/ip pool used print count-only

# ตรวจสอบ DNS resolution
/ping PC1.home.local count=3
```

### Lab Troubleshooting

```bash
# ปัญหา: Client ไม่ได้ IP

# Check 1: DHCP server enabled?
/ip dhcp-server print

# Check 2: Interface ถูกต้อง?
/ip dhcp-server print detail

# Check 3: Pool มี IP เหลือ?
/ip pool used print

# Check 4: Network config ถูกต้อง?
/ip dhcp-server network print

# Check 5: Router IP อยู่ใน same subnet?
/ip address print where interface=ether2

# Check 6: ดู DHCP log
/log print where topics~"dhcp"

# Check 7: Packet capture
/tool sniffer start interface=ether2 filter-port=67,68
# ... ให้ client request แล้ว
/tool sniffer stop
/file print  # ดู pcap file
```

---

## 12. Summary

### สิ่งที่เรียนรู้ใน Part 7:

1. **DHCP Server** ตั้งค่าด้วย pool + server + network
2. **DHCP Client** เพิ่มบน WAN interface เพื่อรับ IP จาก ISP
3. **DHCP Relay** ส่ง DHCP requests ข้าม subnet ไปยัง centralized server
4. **Static Leases** กำหนด IP ให้ device เฉพาะตาม MAC address
5. **DHCP Options** ส่ง parameters เพิ่มเติม (routes, NTP, vendor-specific)
6. **Multiple Servers** สร้าง server แยกต่างหากสำหรับแต่ละ VLAN/subnet
7. **Scripts** อัตโนมัติ เช่น logging, firewall update, DNS update

### Quick Reference

```bash
# DHCP Server
/ip pool add name=X ranges=A-B
/ip dhcp-server add name=X interface=I address-pool=X
/ip dhcp-server network add address=N/M gateway=G dns-server=D

# DHCP Client
/ip dhcp-client add interface=I disabled=no

# Static Lease
/ip dhcp-server lease add address=IP mac-address=MAC server=X

# Monitoring
/ip dhcp-server lease print
/ip pool used print
/log print where topics~"dhcp"
```

---

**[⬅ Previous: Network Configuration](part-006-network-configuration.md)** | **[Next: DNS Configuration ➡](part-008-dns-configuration.md)**
