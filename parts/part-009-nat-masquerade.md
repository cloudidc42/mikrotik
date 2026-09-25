# Part 9: NAT & Masquerade

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner to Intermediate  
> **เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

1. [NAT Fundamentals](#1-nat-fundamentals)
2. [Masquerade (SNAT)](#2-masquerade-snat)
3. [DNAT (Port Forwarding)](#3-dnat-port-forwarding)
4. [1:1 NAT (Static NAT)](#4-11-nat-static-nat)
5. [Hairpin NAT](#5-hairpin-nat)
6. [NAT Traversal](#6-nat-traversal)
7. [Double NAT](#7-double-nat)
8. [NAT Logging](#8-nat-logging)
9. [Common NAT Scenarios](#9-common-nat-scenarios)
10. [Lab: Complete NAT Setup](#10-lab-complete-nat-setup)
11. [Summary](#11-summary)

---

## 1. NAT Fundamentals

### 1.1 NAT คืออะไร

**NAT (Network Address Translation)** คือการเปลี่ยน IP address ใน packet header เพื่อ:
- ให้ internal hosts ใช้ internet ผ่าน IP เดียว (SNAT/Masquerade)
- รับ connections จาก internet ไปยัง internal servers (DNAT)
- แก้ปัญหา IP address space

```
NAT Types:
├── SNAT (Source NAT)
│   ├── Masquerade     - dynamic SNAT (ใช้ IP ของ interface)
│   └── src-nat        - static SNAT (กำหนด IP)
├── DNAT (Destination NAT)
│   └── dst-nat        - Port forwarding, load balancing
├── Redirect           - DNAT ไปยัง router ตัวเอง
└── Netmap             - 1:1 NAT (อาจ deprecated)
```

### 1.2 NAT Chain ใน RouterOS

```
RouterOS NAT Chains:
├── srcnat  - SNAT (after routing decision, before forwarding out)
└── dstnat  - DNAT (before routing decision)

Packet Flow:
                    ┌─────────────┐
Incoming ────────── │  dstnat     │ ← DNAT/Port Forwarding ทำที่นี่
Packet             └──────┬──────┘
                           │
                    ┌──────┴──────┐
                    │  Routing    │
                    │  Decision   │
                    └──────┬──────┘
                           │
                    ┌──────┴──────┐
Outgoing ─────────── │  srcnat     │ ← Masquerade/SNAT ทำที่นี่
Packet              └─────────────┘
```

### 1.3 Connection Tracking สำหรับ NAT

```bash
# NAT ต้องการ Connection Tracking
# ดู connection tracking state
/ip firewall connection print

# Connection States:
# NEW        - packet แรกของ connection
# ESTABLISHED - connection ที่ established แล้ว
# RELATED    - connection ที่ related กับ existing connection
# INVALID    - packet ที่ไม่ fit state ใดๆ

# ดู NAT translations
/ip firewall connection print where connection-nat-state~"srcnat"
```

---

## 2. Masquerade (SNAT)

### 2.1 Basic Masquerade

```bash
# Masquerade ทุก traffic ออก WAN interface
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="Default Masquerade"

# หรือใช้ interface list
/ip firewall nat add \
    chain=srcnat \
    out-interface-list=WAN \
    action=masquerade \
    comment="Masquerade WAN"

# ดู NAT rules
/ip firewall nat print
```

### 2.2 Masquerade กับ src-nat

```bash
# Masquerade: IP เปลี่ยนตาม interface IP (dynamic)
# ดี: ใช้กับ dynamic IP (DHCP/PPPoE)
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade

# src-nat: IP ที่กำหนดไว้ตายตัว (static)
# ดี: เมื่อมี static public IP
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=src-nat \
    to-addresses=203.0.113.10 \
    comment="Static SNAT"

# src-nat กับ range
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=src-nat \
    to-addresses=203.0.113.10-203.0.113.20 \
    comment="SNAT with IP range"
```

### 2.3 Selective Masquerade

```bash
# Masquerade เฉพาะ source subnet
/ip firewall nat add \
    chain=srcnat \
    src-address=192.168.1.0/24 \
    out-interface=ether1 \
    action=masquerade \
    comment="Masquerade LAN only"

# Masquerade ยกเว้น VPN traffic
/ip firewall nat add \
    chain=srcnat \
    dst-address=10.0.0.0/8 \
    action=accept \
    comment="Don't NAT VPN traffic (place before masquerade)"

/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="Masquerade (after VPN exclusion)"

# ⚠️ Rule order สำคัญมาก! เพิ่ม accept ก่อน masquerade
```

### 2.4 Source IP Logging

```bash
# Log NAT translations
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    log=yes \
    log-prefix="NAT-OUT: "
```

---

## 3. DNAT (Port Forwarding)

### 3.1 Basic Port Forwarding

```bash
# Port Forward: WAN:8080 → Internal Server:80
/ip firewall nat add \
    chain=dstnat \
    dst-address=203.0.113.10 \
    protocol=tcp \
    dst-port=8080 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    to-ports=80 \
    comment="HTTP Port Forward to Web Server"

# Port Forward: WAN:443 → Internal Server:443
/ip firewall nat add \
    chain=dstnat \
    in-interface=ether1 \
    protocol=tcp \
    dst-port=443 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    to-ports=443 \
    comment="HTTPS Port Forward"
```

### 3.2 Port Forwarding Examples

```bash
# Web Server (HTTP/HTTPS)
/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp \
    dst-port=80 action=dst-nat to-addresses=192.168.1.10 to-ports=80 \
    comment="HTTP"

/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp \
    dst-port=443 action=dst-nat to-addresses=192.168.1.10 to-ports=443 \
    comment="HTTPS"

# SSH Server (เปลี่ยน public port)
/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp \
    dst-port=2222 action=dst-nat to-addresses=192.168.1.10 to-ports=22 \
    comment="SSH to internal server (port 2222 → 22)"

# Mail Server
/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp \
    dst-port=25 action=dst-nat to-addresses=192.168.1.20 to-ports=25 \
    comment="SMTP"

/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp \
    dst-port=993 action=dst-nat to-addresses=192.168.1.20 to-ports=993 \
    comment="IMAPS"

# Game Server (UDP)
/ip firewall nat add chain=dstnat in-interface=WAN protocol=udp \
    dst-port=27015 action=dst-nat to-addresses=192.168.1.50 to-ports=27015 \
    comment="Game Server"

# RDP (Remote Desktop)
/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp \
    dst-port=3389 action=dst-nat to-addresses=192.168.1.100 to-ports=3389 \
    comment="RDP"

# NAS Web Interface
/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp \
    dst-port=5001 action=dst-nat to-addresses=192.168.1.200 to-ports=5001 \
    comment="Synology NAS"
```

### 3.3 Port Range Forwarding

```bash
# Forward port range
/ip firewall nat add \
    chain=dstnat \
    in-interface=WAN \
    protocol=tcp \
    dst-port=10000-20000 \
    action=dst-nat \
    to-addresses=192.168.1.50 \
    to-ports=10000-20000 \
    comment="Port range 10000-20000"

# Forward multiple ports
/ip firewall nat add \
    chain=dstnat \
    in-interface=WAN \
    protocol=tcp \
    dst-port=80,443,8080,8443 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    comment="Multiple web ports"
```

### 3.4 DNAT กับ Firewall

```bash
# หลังจาก DNAT ต้องเพิ่ม firewall rule อนุญาต traffic
# (ถ้า firewall block incoming connections)

# Allow forwarded traffic ไปยัง web server
/ip firewall filter add \
    chain=forward \
    dst-address=192.168.1.10 \
    protocol=tcp \
    dst-port=80,443 \
    action=accept \
    comment="Allow forwarded HTTP/HTTPS to web server"

# ⚠️ เพิ่ม rule นี้ก่อน rule drop general!
```

---

## 4. 1:1 NAT (Static NAT)

### 4.1 1:1 NAT คืออะไร

```
1:1 NAT (Bidirectional NAT):
- Internal IP ↔ Public IP แบบ 1 ต่อ 1
- ทุก port forward อัตโนมัติ
- ใช้เมื่อมี multiple public IPs

Topology:
Public IPs: 203.0.113.10, 203.0.113.11, 203.0.113.12
Internal IPs: 192.168.1.10, 192.168.1.11, 192.168.1.12

203.0.113.10 ↔ 192.168.1.10 (all ports)
203.0.113.11 ↔ 192.168.1.11 (all ports)
203.0.113.12 ↔ 192.168.1.12 (all ports)
```

### 4.2 1:1 NAT Configuration

```bash
# Public IP 203.0.113.11 → Internal 192.168.1.11

# DNAT: Incoming traffic 203.0.113.11 → 192.168.1.11
/ip firewall nat add \
    chain=dstnat \
    dst-address=203.0.113.11 \
    action=dst-nat \
    to-addresses=192.168.1.11 \
    comment="1:1 NAT DNAT for server2"

# SNAT: Outgoing traffic from 192.168.1.11 → 203.0.113.11
/ip firewall nat add \
    chain=srcnat \
    src-address=192.168.1.11 \
    action=src-nat \
    to-addresses=203.0.113.11 \
    comment="1:1 NAT SNAT for server2"

# เพิ่ม IP บน WAN interface
/ip address add address=203.0.113.11/24 interface=ether1 comment="1:1 NAT IP for server2"
```

### 4.3 1:1 NAT สำหรับหลาย Servers

```bash
# ตาราง mapping:
# 203.0.113.10 ↔ 192.168.1.10 (Web Server)
# 203.0.113.11 ↔ 192.168.1.11 (Mail Server)
# 203.0.113.12 ↔ 192.168.1.12 (FTP Server)

# เพิ่ม public IPs บน WAN interface
/ip address add address=203.0.113.10/29 interface=ether1 comment="1:1 Web"
/ip address add address=203.0.113.11/29 interface=ether1 comment="1:1 Mail"
/ip address add address=203.0.113.12/29 interface=ether1 comment="1:1 FTP"

# DNAT rules
/ip firewall nat add chain=dstnat dst-address=203.0.113.10 \
    action=dst-nat to-addresses=192.168.1.10 comment="1:1 Web DNAT"
/ip firewall nat add chain=dstnat dst-address=203.0.113.11 \
    action=dst-nat to-addresses=192.168.1.11 comment="1:1 Mail DNAT"
/ip firewall nat add chain=dstnat dst-address=203.0.113.12 \
    action=dst-nat to-addresses=192.168.1.12 comment="1:1 FTP DNAT"

# SNAT rules
/ip firewall nat add chain=srcnat src-address=192.168.1.10 \
    action=src-nat to-addresses=203.0.113.10 comment="1:1 Web SNAT"
/ip firewall nat add chain=srcnat src-address=192.168.1.11 \
    action=src-nat to-addresses=203.0.113.11 comment="1:1 Mail SNAT"
/ip firewall nat add chain=srcnat src-address=192.168.1.12 \
    action=src-nat to-addresses=203.0.113.12 comment="1:1 FTP SNAT"

# General masquerade (สำหรับ clients อื่น)
/ip firewall nat add chain=srcnat out-interface=ether1 \
    action=masquerade comment="General Masquerade"
```

---

## 5. Hairpin NAT

### 5.1 Hairpin NAT คืออะไร

```
ปัญหา:
Internal user เข้า web.company.com (203.0.113.10) 
→ ต้องผ่าน internet กลับมา internal server

Without Hairpin:
[Internal Client] → [Router WAN] → [Internet] → [Router WAN] → [Server]
(อาจไม่ work หรือ latency สูง)

With Hairpin NAT:
[Internal Client] → [Router] → [Server (direct)]
(ไม่ต้องออก internet)
```

### 5.2 Hairpin NAT Configuration

```bash
# Hairpin NAT: Internal clients เข้า public IP → redirect ไป internal server

# สมมติ:
# Public IP: 203.0.113.10
# Internal Server: 192.168.1.10
# Internal Clients: 192.168.1.0/24

# Step 1: DNAT (เหมือนปกติ)
/ip firewall nat add \
    chain=dstnat \
    in-interface=WAN \
    dst-address=203.0.113.10 \
    protocol=tcp \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    to-ports=80 \
    comment="DNAT from WAN"

# Step 2: Hairpin NAT (สำหรับ internal clients)
/ip firewall nat add \
    chain=dstnat \
    in-interface=LAN \              ← จาก LAN
    dst-address=203.0.113.10 \     ← ไปยัง public IP
    protocol=tcp \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    to-ports=80 \
    comment="Hairpin NAT HTTP"

# Step 3: Masquerade hairpin traffic
/ip firewall nat add \
    chain=srcnat \
    src-address=192.168.1.0/24 \   ← traffic มาจาก LAN
    dst-address=192.168.1.10 \     ← ไปยัง internal server
    action=masquerade \
    comment="Hairpin Masquerade"
```

### 5.3 Split-horizon DNS แทน Hairpin NAT

```bash
# วิธีที่ดีกว่า: ใช้ Split-horizon DNS
# internal clients ได้ internal IP โดยตรง ไม่ต้องผ่าน NAT

/ip dns static add \
    name=web.company.com \
    address=192.168.1.10 \
    comment="Internal: bypass public IP"

# Internal clients จะ resolve web.company.com เป็น 192.168.1.10 โดยตรง
```

---

## 6. NAT Traversal

### 6.1 NAT Traversal ใน RouterOS

```bash
# RouterOS มี built-in NAT traversal helpers

# ดู helpers ที่ enabled
/ip firewall service-port print

# Output:
# NAME    PORTS    ENABLED
# ftp     21       yes
# tftp    69       yes
# irc     6667     no
# h323             yes
# sip     5060,5061 yes
# pptp              yes
# dccp              no
# sctp              no
# udplite           no

# Enable/Disable helpers
/ip firewall service-port enable ftp
/ip firewall service-port disable irc
```

### 6.2 SIP NAT Traversal (VoIP)

```bash
# SIP มักมีปัญหากับ NAT
# RouterOS มี SIP helper

# Enable SIP helper
/ip firewall service-port enable sip

# SIP helper จัดการ:
# - Rewrite SIP headers ที่มี private IPs
# - Open RTP ports สำหรับ media streams
# - Track SIP sessions

# ถ้ายังมีปัญหา ลอง disable helper และใช้ STUN
/ip firewall service-port disable sip

# หรือใช้ SIPROXD (external)
```

### 6.3 PPTP Passthrough

```bash
# Enable PPTP passthrough
/ip firewall service-port enable pptp

# PPTP helper จัดการ GRE protocol
# ทำให้ internal PPTP clients ใช้ได้

# L2TP Passthrough (ไม่ต้องการ helper)
# L2TP ใช้ UDP ซึ่ง NAT รองรับ native
```

---

## 7. Double NAT

### 7.1 Double NAT คืออะไร

```
Double NAT situation:
[ISP Modem Router] NAT1 ← → [MikroTik Router] NAT2 ← → [Internal Clients]

ISP Modem: 192.168.0.0/24 (NAT1)
MikroTik WAN: 192.168.0.x (private IP จาก ISP Modem)
MikroTik LAN: 192.168.1.0/24 (NAT2)
```

### 7.2 ปัญหาของ Double NAT

```
ปัญหา:
- Port forwarding ซับซ้อน (ต้อง forward ใน modem ด้วย)
- บาง applications ไม่ทำงาน (peer-to-peer)
- VPN อาจมีปัญหา
- NAT loopback ซับซ้อนขึ้น

วิธีแก้:
1. ตั้ง ISP modem เป็น Bridge Mode → MikroTik รับ public IP โดยตรง
2. ตั้ง ISP modem เป็น DMZ → forward ทุก traffic ไปยัง MikroTik
3. ยอมรับ double NAT (ถ้าไม่มี inbound connections)
```

### 7.3 Bridge Mode / DMZ Setup

```bash
# ถ้า ISP modem ตั้ง DMZ หรือ Bridge ไม่ได้:
# ตั้ง Double NAT อย่างถูกต้อง

# บน ISP Modem:
# Port Forward ALL → 192.168.0.2 (MikroTik WAN IP)
# หรือ DMZ → 192.168.0.2

# บน MikroTik:
# ทำ port forwarding ตามปกติ
/ip firewall nat add \
    chain=dstnat \
    in-interface=WAN \
    protocol=tcp \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    comment="HTTP through double NAT"
```

---

## 8. NAT Logging

### 8.1 Log NAT Events

```bash
# เพิ่ม log action ใน NAT rules
/ip firewall nat add \
    chain=srcnat \
    out-interface=WAN \
    action=masquerade \
    log=yes \
    log-prefix="NAT-MASQ: " \
    comment="Masquerade with logging"

# Log DNAT
/ip firewall nat add \
    chain=dstnat \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.10 \
    log=yes \
    log-prefix="NAT-DNAT-HTTP: "
```

### 8.2 Log ไปยัง Remote Syslog

```bash
# ตั้ง logging ไปยัง syslog server
/system logging action add \
    name=syslog \
    type=remote \
    remote=192.168.1.200 \
    remote-port=514 \
    src-address=0.0.0.0 \
    bsd-syslog=yes

# ตั้ง topics ที่จะ log
/system logging add \
    topics=firewall \
    action=syslog

# NAT logs จะส่งไปยัง 192.168.1.200:514
```

### 8.3 Accounting with NAT

```bash
# ใช้ Mangle สำหรับ NAT accounting
/ip firewall mangle add \
    chain=prerouting \
    in-interface=WAN \
    action=mark-connection \
    new-connection-mark=inbound

/ip firewall mangle add \
    chain=postrouting \
    out-interface=WAN \
    action=mark-connection \
    new-connection-mark=outbound

# ดู marked connections
/ip firewall connection print where connection-mark=outbound
```

---

## 9. Common NAT Scenarios

### 9.1 Home Router Setup

```bash
# Standard home router NAT
/ip dhcp-client add interface=ether1 disabled=no  # WAN DHCP

/ip address add address=192.168.1.1/24 interface=ether2  # LAN

/ip dhcp-server setup  # Setup DHCP for LAN

/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="Home Internet NAT"
```

### 9.2 ISP BRAS (PPPoE)

```bash
# ISP ที่รับ PPPoE clients และ NAT ให้
# (CGN - Carrier Grade NAT)

/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    src-address=100.64.0.0/10 \     # Shared address space (RFC 6598)
    action=masquerade \
    comment="CGN for PPPoE clients"
```

### 9.3 Multiple WAN (Load Balancing)

```bash
# Multi-WAN ต้องการ NAT ต่าง interface

# WAN1 (ISP1): ether1
# WAN2 (ISP2): ether2
# LAN: ether3

/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="NAT via WAN1"

/ip firewall nat add \
    chain=srcnat \
    out-interface=ether2 \
    action=masquerade \
    comment="NAT via WAN2"

# Routing จัดการ traffic ไปแต่ละ WAN
# (ดูรายละเอียดใน Part routing)
```

### 9.4 DMZ Setup

```bash
# DMZ (Demilitarized Zone) สำหรับ public servers

# Interfaces:
# WAN: ether1 (203.0.113.1/24)
# LAN: ether2 (192.168.1.0/24)
# DMZ: ether3 (10.0.0.0/24)

# NAT สำหรับ DMZ servers (different public IPs)
/ip address add address=203.0.113.20/24 interface=ether1 comment="DMZ Web IP"
/ip address add address=203.0.113.21/24 interface=ether1 comment="DMZ Mail IP"

# DNAT ไปยัง DMZ servers
/ip firewall nat add chain=dstnat dst-address=203.0.113.20 \
    action=dst-nat to-addresses=10.0.0.10 comment="Web server in DMZ"
/ip firewall nat add chain=dstnat dst-address=203.0.113.21 \
    action=dst-nat to-addresses=10.0.0.11 comment="Mail server in DMZ"

# SNAT สำหรับ DMZ outbound
/ip firewall nat add chain=srcnat src-address=10.0.0.10 \
    action=src-nat to-addresses=203.0.113.20 comment="Web server SNAT"
/ip firewall nat add chain=srcnat src-address=10.0.0.11 \
    action=src-nat to-addresses=203.0.113.21 comment="Mail server SNAT"

# LAN masquerade
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

### 9.5 VPN + NAT

```bash
# VPN traffic ไม่ควร NAT
# เพิ่ม accept rule ก่อน masquerade

# ไม่ NAT traffic ที่ไปยัง VPN subnets
/ip firewall nat add \
    chain=srcnat \
    dst-address=10.0.0.0/8 \
    action=accept \
    comment="Don't NAT VPN traffic" \
    place-before=0  # ต้องอยู่ก่อน masquerade

/ip firewall nat add \
    chain=srcnat \
    dst-address=172.16.0.0/12 \
    action=accept \
    comment="Don't NAT private networks"

# Masquerade หลังจาก exclusions
/ip firewall nat add \
    chain=srcnat \
    out-interface=WAN \
    action=masquerade
```

---

## 10. Lab: Complete NAT Setup

### Lab Topology

```
          Internet
              │
    ┌─────────┴─────────┐
    │   MikroTik Edge   │
    │   Router          │
    │   WAN: Dynamic IP │
    │   (from DHCP)     │
    └─────────┬─────────┘
              │ ether2 (LAN) 192.168.1.1/24
    ┌─────────┼──────────────────┐
    │         │                  │
┌───┴────┐ ┌──┴────┐ ┌──────────┴──┐
│  PC1   │ │  PC2  │ │ Web Server  │
│DHCP    │ │DHCP   │ │ Static      │
│.1.10   │ │.1.20  │ │ .1.100:80   │
└────────┘ └───────┘ └─────────────┘
```

### Step 1: Basic Setup

```bash
# ตั้งชื่อ interfaces
/interface set ether1 name=WAN
/interface set ether2 name=LAN

# WAN - DHCP
/ip dhcp-client add interface=WAN disabled=no

# LAN - Static
/ip address add address=192.168.1.1/24 interface=LAN

# DHCP Server สำหรับ LAN
/ip pool add name=lan-pool ranges=192.168.1.10-192.168.1.90
/ip dhcp-server add name=lan-dhcp interface=LAN address-pool=lan-pool
/ip dhcp-server network add address=192.168.1.0/24 gateway=192.168.1.1 dns-server=8.8.8.8
```

### Step 2: Masquerade (Internet Access)

```bash
# Masquerade สำหรับ clients อินเทอร์เน็ต
/ip firewall nat add \
    chain=srcnat \
    out-interface=WAN \
    action=masquerade \
    comment="Internet Masquerade"
```

### Step 3: Port Forwarding (Web Server)

```bash
# Forward HTTP ไปยัง Web Server ภายใน
/ip firewall nat add \
    chain=dstnat \
    in-interface=WAN \
    protocol=tcp \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.100 \
    to-ports=80 \
    comment="HTTP to Web Server"

/ip firewall nat add \
    chain=dstnat \
    in-interface=WAN \
    protocol=tcp \
    dst-port=443 \
    action=dst-nat \
    to-addresses=192.168.1.100 \
    to-ports=443 \
    comment="HTTPS to Web Server"

# Forward SSH (port 2222 → 22)
/ip firewall nat add \
    chain=dstnat \
    in-interface=WAN \
    protocol=tcp \
    dst-port=2222 \
    action=dst-nat \
    to-addresses=192.168.1.100 \
    to-ports=22 \
    comment="SSH to Web Server"
```

### Step 4: Firewall Rules สำหรับ Port Forward

```bash
# Allow forwarded traffic ไปยัง web server
/ip firewall filter add \
    chain=forward \
    dst-address=192.168.1.100 \
    protocol=tcp \
    dst-port=80,443,22 \
    connection-state=new \
    action=accept \
    comment="Allow forwarded traffic to Web Server"
```

### Step 5: Hairpin NAT

```bash
# Internal users เข้า web server ผ่าน public IP
# ต้องรู้ public IP ก่อน
:local pubIP [/ip dhcp-client get [find interface=WAN] address]
:local pubIPclean [:pick $pubIP 0 [:find $pubIP "/"]]

# Hairpin DNAT
/ip firewall nat add \
    chain=dstnat \
    in-interface=LAN \
    dst-address=$pubIPclean \
    protocol=tcp \
    dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.100 \
    comment="Hairpin HTTP"

# Hairpin Masquerade
/ip firewall nat add \
    chain=srcnat \
    src-address=192.168.1.0/24 \
    dst-address=192.168.1.100 \
    action=masquerade \
    comment="Hairpin Masquerade"
```

### Step 6: Verification

```bash
# ดู NAT rules
/ip firewall nat print

# ดู active NAT connections
/ip firewall connection print where connection-nat-state~"srcnat"

# ทดสอบ Internet access
/ping 8.8.8.8 count=3
/ping google.com count=3

# ทดสอบ port forwarding (จาก outside)
# เปิด browser → http://[WAN-IP]:80
# หรือ
# curl http://[WAN-IP]:80

# ดู masquerade connections
/ip firewall connection print count-only

# Monitor NAT traffic
/tool torch interface=WAN protocol=tcp
```

### Lab Troubleshooting

```bash
# ปัญหา: Internet ใช้ไม่ได้

# Check 1: WAN interface ได้ IP?
/ip dhcp-client print detail

# Check 2: Default route มีไหม?
/ip route print where dst-address="0.0.0.0/0"

# Check 3: NAT rule ถูกต้อง?
/ip firewall nat print

# Check 4: Packet capture บน WAN
/tool sniffer start interface=WAN
# ping 8.8.8.8 จาก client
/tool sniffer stop
/tool sniffer packet print

# ปัญหา: Port forwarding ไม่ทำงาน

# Check 1: DNAT rule
/ip firewall nat print where chain=dstnat

# Check 2: Firewall forward rule
/ip firewall filter print where chain=forward

# Check 3: ทดสอบ connection ภายใน
/ping 192.168.1.100 count=3

# Check 4: Packet capture
/tool sniffer start interface=LAN filter-ip-address=192.168.1.100
```

---

## 11. Summary

### สิ่งที่เรียนรู้ใน Part 9:

1. **Masquerade** ใช้สำหรับ internet sharing, ดีสำหรับ dynamic IP
2. **src-nat** ใช้เมื่อมี static public IP
3. **DNAT** ใช้สำหรับ port forwarding
4. **1:1 NAT** map public IP ↔ internal IP ทุก port
5. **Hairpin NAT** แก้ปัญหา internal users เข้า server ผ่าน public IP
6. **NAT Helpers** ช่วย protocols ที่ embed IPs ใน payload
7. **Double NAT** ปัญหาที่เกิดเมื่อมี router สองชั้น

### NAT Rules Order (สำคัญ!)

```bash
# Order ที่ถูกต้องสำหรับ NAT rules:
1. DNAT rules (chain=dstnat)
2. VPN exclusions (chain=srcnat, action=accept)
3. Hairpin masquerade (ก่อน general masquerade)
4. General masquerade/src-nat (chain=srcnat)

# ตรวจสอบ order
/ip firewall nat print
```

### Quick Reference

```bash
# Masquerade
/ip firewall nat add chain=srcnat out-interface=WAN action=masquerade

# Port Forward
/ip firewall nat add chain=dstnat in-interface=WAN protocol=tcp \
    dst-port=80 action=dst-nat to-addresses=192.168.1.10 to-ports=80

# 1:1 NAT (DNAT side)
/ip firewall nat add chain=dstnat dst-address=PUBLIC_IP \
    action=dst-nat to-addresses=INTERNAL_IP

# 1:1 NAT (SNAT side)
/ip firewall nat add chain=srcnat src-address=INTERNAL_IP \
    action=src-nat to-addresses=PUBLIC_IP

# ดู NAT connections
/ip firewall connection print where connection-nat-state~"srcnat"
```

---

**[⬅ Previous: DNS Configuration](part-008-dns-configuration.md)** | **[Next: Firewall Basics ➡](part-010-firewall-basics.md)**
