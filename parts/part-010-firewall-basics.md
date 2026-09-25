# Part 10: Firewall Basics

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner to Intermediate  
> **เวลาเรียน:** 5-6 ชั่วโมง

---

## สารบัญ

1. [Firewall Overview](#1-firewall-overview)
2. [Firewall Chains](#2-firewall-chains)
3. [Action Types](#3-action-types)
4. [Connection Tracking](#4-connection-tracking)
5. [Common Rules](#5-common-rules)
6. [Basic DDoS Protection](#6-basic-ddos-protection)
7. [Port Scan Detection](#7-port-scan-detection)
8. [Input Protection](#8-input-protection)
9. [Logging Firewall Events](#9-logging-firewall-events)
10. [Firewall Testing](#10-firewall-testing)
11. [Lab: Complete Basic Firewall](#11-lab-complete-basic-firewall)
12. [Summary](#12-summary)

---

## 1. Firewall Overview

### 1.1 RouterOS Firewall Tables

```
RouterOS มี Firewall Tables ดังนี้:

/ip firewall filter  - Main packet filtering
/ip firewall nat     - NAT (จาก Part 9)
/ip firewall mangle  - Packet marking/modification
/ip firewall raw     - Early packet processing (before conntrack)
/ip firewall address-list - IP address lists

/ipv6 firewall filter - IPv6 filtering
/ipv6 firewall nat    - IPv6 NAT
/ipv6 firewall mangle - IPv6 mangle
/ipv6 firewall raw    - IPv6 raw
```

### 1.2 Packet Processing Order

```
Packet arrives → Interface
        │
        ▼
┌───────────────┐
│   RAW Table   │ ← ก่อน connection tracking
│  Prerouting   │   (filter ก่อน, ลด load)
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  Connection   │ ← Track connection state
│  Tracking     │   (NEW, ESTABLISHED, RELATED, INVALID)
└───────┬───────┘
        │
        ▼
┌───────────────┐
│  NAT Table    │ ← DNAT ที่นี่
│  Prerouting   │
└───────┬───────┘
        │
        ├────────────────────────┐
        │ (for this router)      │ (for forward)
        ▼                        ▼
┌───────────────┐         ┌───────────────┐
│ Filter: Input │         │Filter: Forward│
└───────┬───────┘         └───────┬───────┘
        │                         │
        ▼                         ▼
  Local Process           ┌───────────────┐
                          │  NAT Table    │ ← SNAT ที่นี่
                          │  Postrouting  │
                          └───────┬───────┘
                                  │
                                  ▼
                           Egress Interface
```

### 1.3 Rule Matching

```
Rules ถูก match จาก บนลงล่าง (top-down):
1. ถ้า packet match rule → ทำ action (และหยุด หรือ continue ตาม action)
2. ถ้าไม่ match → ไปดู rule ถัดไป
3. ถ้าไม่ match rule ใดเลย → default action (accept)

⚠️ สำคัญ: ลำดับ rules มีผลมาก!
- accept rules ต้องมาก่อน drop rules สำหรับ traffic ที่อนุญาต
- Drop rules ต้องอยู่หลัง accept rules
```

---

## 2. Firewall Chains

### 2.1 Chain: INPUT

```
Chain: input
- Packet ที่มีปลายทางเป็น Router ตัวเอง
- ใช้ป้องกัน management access

Examples:
- SSH connection มาที่ router
- Ping มาที่ router
- DNS query มาที่ router
- DHCP request (broadcast)
- SNMP query
- Winbox connection
```

**Input chain packet flow:**
```bash
# ดู INPUT chain rules
/ip firewall filter print where chain=input

# Rules ที่ควรมีใน INPUT:
1. Accept established/related connections
2. Drop invalid connections
3. Accept ICMP (ping) - optional
4. Accept management services from trusted IPs
5. Drop everything else from WAN
```

### 2.2 Chain: FORWARD

```
Chain: forward
- Packet ที่ผ่าน Router (ไม่ได้หยุดที่ router)
- ใช้ควบคุม traffic ระหว่าง networks

Examples:
- Client LAN → Internet
- DMZ server → Internet
- VPN → LAN
```

**Forward chain packet flow:**
```bash
# Rules ที่ควรมีใน FORWARD:
1. Accept established/related connections (performance)
2. FastTrack established connections (optional, high performance)
3. Drop invalid connections
4. Accept LAN → WAN (allow outbound)
5. Drop WAN → LAN new connections
6. Allow specific port forwards
```

### 2.3 Chain: OUTPUT

```
Chain: output
- Packet ที่ router ส่งออกไป (router เป็น source)
- ใช้ควบคุม traffic จาก router เอง

Examples:
- Router ping ออกไป
- Router ส่ง NTP query
- Router ส่ง DNS query
- Router ส่ง update
```

```bash
# Output chain (ส่วนใหญ่ไม่ต้องแก้ไข)
/ip firewall filter print where chain=output
```

### 2.4 Custom Chains

```bash
# สร้าง custom chains เพื่อ organize rules
# (เหมือน sub-routines ใน programming)

# ตัวอย่าง: create chain สำหรับ detect portscans
/ip firewall filter add chain=detect-portscan action=return comment="Port scan detection chain"

# Jump ไป chain
/ip firewall filter add \
    chain=input \
    action=jump \
    jump-target=detect-portscan \
    comment="Run port scan detection"
```

---

## 3. Action Types

### 3.1 Terminal Actions (หยุด processing)

| Action | Description |
|--------|-------------|
| `accept` | อนุญาต packet, หยุดตรวจสอบ rules |
| `drop` | ทิ้ง packet เงียบๆ (ไม่ส่ง reply) |
| `reject` | ทิ้ง packet พร้อมส่ง ICMP error |
| `tarpit` | รับ TCP แต่ไม่ส่ง data (ดัก attackers) |

### 3.2 Non-terminal Actions (ทำแล้วไปต่อ)

| Action | Description |
|--------|-------------|
| `log` | Log packet แล้วไป rule ถัดไป |
| `passthrough` | ไม่ทำอะไร แค่ count แล้วไปต่อ |
| `jump` | กระโดดไป chain อื่น |
| `return` | กลับจาก chain (ใช้กับ jump) |
| `add-src-to-address-list` | เพิ่ม source IP ไปยัง address list |
| `add-dst-to-address-list` | เพิ่ม destination IP ไปยัง address list |

### 3.3 Action Examples

```bash
# Accept
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    src-address=192.168.1.0/24 \
    action=accept \
    comment="Accept SSH from LAN"

# Drop (silent)
/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    action=drop \
    comment="Drop all from WAN"

# Reject (sends ICMP unreachable)
/ip firewall filter add \
    chain=forward \
    src-address=192.168.2.0/24 \
    dst-address=192.168.1.0/24 \
    action=reject \
    reject-with=icmp-network-unreachable \
    comment="Reject VLAN2 to VLAN1"

# Log ก่อน drop
/ip firewall filter add \
    chain=input \
    action=log \
    log-prefix="Dropped: " \
    comment="Log before drop"

/ip firewall filter add \
    chain=input \
    action=drop \
    comment="Drop (after log)"

# Add to address list
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    connection-state=new \
    action=add-src-to-address-list \
    address-list=ssh-access \
    address-list-timeout=1h \
    comment="Track SSH connections"
```

---

## 4. Connection Tracking

### 4.1 Connection States

```bash
# Connection states ที่ใช้ใน firewall:
# new        - packet แรกของ connection ใหม่
# established - ตอบกลับมาใน established connection
# related    - related connection (FTP data, ICMP errors)
# invalid    - packet ที่ไม่ match connection state ใดๆ
# untracked  - connection ที่ไม่ถูก track

# ดู connections
/ip firewall connection print

# นับ connections
/ip firewall connection print count-only

# ดู connections พร้อม details
/ip firewall connection print detail
```

### 4.2 Stateful Firewall Rules

```bash
# Pattern พื้นฐาน:
# 1. Accept established/related (ก่อน)
# 2. Drop invalid
# 3. ตรวจสอบ new connections

# Accept established/related - Performance rule
/ip firewall filter add \
    chain=input \
    connection-state=established,related \
    action=accept \
    comment="Accept established/related"

/ip firewall filter add \
    chain=forward \
    connection-state=established,related \
    action=accept \
    comment="Accept established/related forward"

# Drop invalid
/ip firewall filter add \
    chain=input \
    connection-state=invalid \
    action=drop \
    comment="Drop invalid"

/ip firewall filter add \
    chain=forward \
    connection-state=invalid \
    action=drop \
    comment="Drop invalid forward"
```

### 4.3 FastTrack

```bash
# FastTrack: bypass firewall สำหรับ established connections
# เพิ่ม performance สูงมาก แต่ traffic bypass mangle/queue

/ip firewall filter add \
    chain=forward \
    connection-state=established,related \
    connection-mark=no-mark \
    action=fasttrack-connection \
    comment="FastTrack (high performance)"

/ip firewall filter add \
    chain=forward \
    connection-state=established,related \
    action=accept \
    comment="Accept established after fasttrack"

# ⚠️ Warning: FastTrack bypass mangle rules
# ถ้าใช้ QoS/shaping อย่าใช้ FastTrack หรือ ใช้ connection-mark=no-mark เท่านั้น
```

### 4.4 Connection Tracking Settings

```bash
# ดูและตั้ง connection tracking
/ip firewall connection tracking set \
    enabled=yes \
    udp-timeout=10s \
    udp-stream-timeout=180s \
    tcp-syn-sent-timeout=5s \
    tcp-syn-received-timeout=5s \
    tcp-established-timeout=1d \
    tcp-fin-wait-timeout=120s \
    tcp-close-wait-timeout=60s \
    tcp-last-ack-timeout=30s \
    tcp-time-wait-timeout=120s \
    tcp-close-timeout=10s \
    icmp-timeout=10s \
    generic-timeout=10m

# ดู max entries
/ip firewall connection tracking print
```

---

## 5. Common Rules

### 5.1 Anti-Spoofing

```bash
# Block packets ที่ claim เป็น private IPs จาก WAN (spoofing)

/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    src-address=192.168.0.0/16 \
    action=drop \
    comment="Anti-spoof: block private 192.168.x.x from WAN"

/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    src-address=10.0.0.0/8 \
    action=drop \
    comment="Anti-spoof: block private 10.x.x.x from WAN"

/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    src-address=172.16.0.0/12 \
    action=drop \
    comment="Anti-spoof: block private 172.16-31.x.x from WAN"

/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    src-address=127.0.0.0/8 \
    action=drop \
    comment="Anti-spoof: block loopback from WAN"

/ip firewall filter add \
    chain=forward \
    in-interface=WAN \
    src-address=192.168.0.0/16 \
    action=drop \
    comment="Anti-spoof forward: block 192.168 from WAN"

/ip firewall filter add \
    chain=forward \
    in-interface=WAN \
    src-address=10.0.0.0/8 \
    action=drop \
    comment="Anti-spoof forward: block 10.x from WAN"
```

### 5.2 Bogon Filtering (Bogon IPs)

```bash
# Bogons: IP ranges ที่ไม่ควรมาจาก internet

# สร้าง address list
/ip firewall address-list add list=bogons address=0.0.0.0/8
/ip firewall address-list add list=bogons address=10.0.0.0/8
/ip firewall address-list add list=bogons address=100.64.0.0/10
/ip firewall address-list add list=bogons address=127.0.0.0/8
/ip firewall address-list add list=bogons address=169.254.0.0/16
/ip firewall address-list add list=bogons address=172.16.0.0/12
/ip firewall address-list add list=bogons address=192.0.0.0/24
/ip firewall address-list add list=bogons address=192.0.2.0/24
/ip firewall address-list add list=bogons address=192.168.0.0/16
/ip firewall address-list add list=bogons address=198.18.0.0/15
/ip firewall address-list add list=bogons address=198.51.100.0/24
/ip firewall address-list add list=bogons address=203.0.113.0/24
/ip firewall address-list add list=bogons address=224.0.0.0/4
/ip firewall address-list add list=bogons address=240.0.0.0/4
/ip firewall address-list add list=bogons address=255.255.255.255/32

# Drop bogons จาก WAN
/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    src-address-list=bogons \
    action=drop \
    comment="Drop bogons from WAN"

/ip firewall filter add \
    chain=forward \
    in-interface=WAN \
    src-address-list=bogons \
    action=drop \
    comment="Drop forwarded bogons"
```

### 5.3 ICMP Rules

```bash
# Accept ping (ICMP echo-request)
/ip firewall filter add \
    chain=input \
    protocol=icmp \
    icmp-options=8:0 \
    action=accept \
    comment="Accept ICMP echo-request (ping)"

# Accept ping replies
/ip firewall filter add \
    chain=input \
    protocol=icmp \
    icmp-options=0:0 \
    action=accept \
    comment="Accept ICMP echo-reply"

# Rate limit ICMP (ป้องกัน ICMP flood)
/ip firewall filter add \
    chain=input \
    protocol=icmp \
    limit=10,20:packet \
    action=accept \
    comment="Accept ICMP rate-limited"

/ip firewall filter add \
    chain=input \
    protocol=icmp \
    action=drop \
    comment="Drop excessive ICMP"
```

---

## 6. Basic DDoS Protection

### 6.1 SYN Flood Protection

```bash
# SYN flood: ส่ง TCP SYN จำนวนมากเพื่อ exhaust connection table

# Method 1: SYN-cookie (ใน connection tracking)
/ip firewall connection tracking set tcp-syn-sent-timeout=5s

# Method 2: Limit new TCP connections
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    tcp-flags=syn \
    connection-state=new \
    limit=50,100:packet \
    action=accept \
    comment="Accept SYN within limit"

/ip firewall filter add \
    chain=input \
    protocol=tcp \
    tcp-flags=syn \
    action=drop \
    comment="Drop excessive SYN"

# Method 3: SYN per-IP limit
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    tcp-flags=syn \
    connection-state=new \
    action=add-src-to-address-list \
    address-list=syn-flood-check \
    address-list-timeout=1m \
    comment="Track SYN sources"

# นับ hits จาก same IP
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    tcp-flags=syn \
    src-address-list=syn-flood-check \
    connection-limit=20,32 \
    action=add-src-to-address-list \
    address-list=syn-flooder \
    address-list-timeout=1h \
    comment="Identify SYN flooders"

/ip firewall filter add \
    chain=input \
    src-address-list=syn-flooder \
    action=drop \
    comment="Drop SYN flooders"
```

### 6.2 UDP Flood Protection

```bash
# UDP flood protection
/ip firewall filter add \
    chain=input \
    protocol=udp \
    limit=100,200:packet \
    action=accept \
    comment="Accept UDP within limit"

/ip firewall filter add \
    chain=input \
    protocol=udp \
    action=drop \
    comment="Drop excessive UDP"
```

### 6.3 ICMP Flood Protection

```bash
# ICMP flood (ping flood)
/ip firewall filter add \
    chain=input \
    protocol=icmp \
    limit=20,50:packet \
    action=accept \
    comment="Accept ICMP within limit"

/ip firewall filter add \
    chain=input \
    protocol=icmp \
    action=drop \
    comment="Drop ICMP flood"
```

### 6.4 Connection Flood Protection

```bash
# Block IPs ที่มี connections มากเกินไป
/ip firewall filter add \
    chain=input \
    connection-limit=100,32 \
    action=add-src-to-address-list \
    address-list=connection-flooder \
    address-list-timeout=30m \
    comment="Add high-connection IPs to list"

/ip firewall filter add \
    chain=input \
    src-address-list=connection-flooder \
    action=drop \
    comment="Drop connection flooders"
```

---

## 7. Port Scan Detection

### 7.1 Detection Method

```bash
# ตรวจจับ port scanners ด้วย address-list

# Step 1: สร้าง chain สำหรับ port scan detection
# Step 2: เพิ่ม IPs ที่ scan ลงใน blacklist
# Step 3: Block IPs ใน blacklist

# Port scan detection - นับ connection attempts
/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    protocol=tcp \
    psd=21,3s,3,1 \
    action=add-src-to-address-list \
    address-list=port-scanner \
    address-list-timeout=1d \
    comment="Detect port scanners"

# Drop port scanners
/ip firewall filter add \
    chain=input \
    src-address-list=port-scanner \
    action=drop \
    comment="Drop port scanners"
```

### 7.2 Custom Port Scan Detection

```bash
# Method: track IPs ที่ access หลาย ports ในเวลาสั้น

# Stage 1: Track stage 1 ports (common scan targets)
/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    protocol=tcp \
    dst-port=21,22,23,25,110,143,161,443,1433,3306,3389,5432,6379,27017 \
    connection-state=new \
    action=add-src-to-address-list \
    address-list=scan-stage1 \
    address-list-timeout=5m \
    comment="Stage 1: track connections to common ports"

# Stage 2: IPs ใน stage1 ที่ try อีก port
/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    protocol=tcp \
    connection-state=new \
    src-address-list=scan-stage1 \
    action=add-src-to-address-list \
    address-list=port-scanner-detected \
    address-list-timeout=1d \
    comment="Stage 2: mark as port scanner"

# Drop detected scanners
/ip firewall filter add \
    chain=input \
    src-address-list=port-scanner-detected \
    action=drop \
    log=yes \
    log-prefix="PortScan BLOCKED: " \
    comment="Drop port scanners"
```

### 7.3 Honeypot Ports

```bash
# ตั้ง honeypot ports ที่ไม่มี service จริง
# ใครที่ connect มาถือว่าเป็น scanner

/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    protocol=tcp \
    dst-port=1234,4444,7777,12345 \
    connection-state=new \
    action=add-src-to-address-list \
    address-list=honeypot-hits \
    address-list-timeout=7d \
    comment="Honeypot: add to blacklist"

/ip firewall filter add \
    chain=input \
    src-address-list=honeypot-hits \
    action=drop \
    comment="Drop honeypot blacklist"
```

---

## 8. Input Protection

### 8.1 Protect Management Interfaces

```bash
# Allow management เฉพาะจาก trusted IPs
/ip firewall address-list add list=management-hosts address=192.168.1.0/24
/ip firewall address-list add list=management-hosts address=10.0.0.1/32 comment="Admin VPN IP"

# Allow SSH จาก trusted hosts เท่านั้น
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    src-address-list=management-hosts \
    action=accept \
    comment="Allow SSH from trusted"

/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    action=drop \
    comment="Drop SSH from others"

# Allow Winbox จาก trusted hosts
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=8291 \
    src-address-list=management-hosts \
    action=accept \
    comment="Allow Winbox from trusted"

/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=8291 \
    action=drop \
    comment="Drop Winbox from others"
```

### 8.2 SSH Brute Force Protection

```bash
# ป้องกัน SSH brute force ด้วย rate limiting

# Stage 1: Add IPs ที่ connect SSH บ่อย
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    connection-state=new \
    action=add-src-to-address-list \
    address-list=ssh-attempts \
    address-list-timeout=5m \
    comment="Track SSH attempts"

# Stage 2: Block IPs ที่พยายาม > 3 ครั้งใน 5 นาที
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    connection-state=new \
    src-address-list=ssh-attempts \
    connection-limit=3,32 \
    action=add-src-to-address-list \
    address-list=ssh-brute-force \
    address-list-timeout=1d \
    comment="Mark SSH brute forcers"

/ip firewall filter add \
    chain=input \
    src-address-list=ssh-brute-force \
    action=drop \
    log=yes \
    log-prefix="SSH BRUTE-FORCE: " \
    comment="Drop SSH brute forcers"

# Allow normal SSH
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    connection-state=new \
    action=accept \
    comment="Allow SSH"
```

### 8.3 Input Default Rules

```bash
# Complete INPUT chain protection

# 1. Accept established/related (performance)
/ip firewall filter add \
    chain=input \
    connection-state=established,related \
    action=accept \
    comment="Accept established"

# 2. Accept loopback
/ip firewall filter add \
    chain=input \
    in-interface=lo \
    action=accept \
    comment="Accept loopback"

# 3. Drop invalid
/ip firewall filter add \
    chain=input \
    connection-state=invalid \
    action=drop \
    comment="Drop invalid"

# 4. Accept ICMP (ping) with rate limit
/ip firewall filter add \
    chain=input \
    protocol=icmp \
    limit=10,20:packet \
    action=accept \
    comment="Accept ICMP rate-limited"

# 5. Accept management from LAN
/ip firewall filter add \
    chain=input \
    in-interface-list=LAN \
    action=accept \
    comment="Accept all from LAN"

# 6. Accept specific services (if any)
# ... add specific rules here ...

# 7. Drop everything else
/ip firewall filter add \
    chain=input \
    action=drop \
    log=yes \
    log-prefix="INPUT DROP: " \
    comment="Drop all other input"
```

---

## 9. Logging Firewall Events

### 9.1 Log Firewall Events

```bash
# เปิด logging สำหรับ firewall
# Log จะไปที่ /log ตามปกติ

# Log ก่อน drop (non-terminal log ตามด้วย terminal drop)
/ip firewall filter add \
    chain=input \
    in-interface=WAN \
    connection-state=new \
    action=log \
    log-prefix="WAN-INPUT: " \
    comment="Log new connections from WAN"

# Log dropped packets เฉพาะ
/ip firewall filter add \
    chain=input \
    action=log \
    log-prefix="DROPPED: " \
    place-before=[find action=drop chain=input comment="Drop all other input"]
```

### 9.2 Logging Configuration

```bash
# ตั้ง logging topics
/system logging add topics=firewall action=memory
/system logging add topics=firewall action=disk  # log ไป disk ด้วย

# ดู logs
/log print where topics~"firewall"

# Real-time log monitor
/log print follow where topics~"firewall"

# Log ไปยัง syslog server
/system logging action set remote \
    name=syslog \
    type=remote \
    remote=192.168.1.200 \
    remote-port=514

/system logging add topics=firewall action=remote
```

### 9.3 Log Rotation

```bash
# ตั้งค่า log storage
/system logging action set memory \
    memory-lines=1000 \      # เก็บใน memory 1000 lines
    memory-stop-on-full=no   # ลบเก่าเมื่อเต็ม

/system logging action set disk \
    disk-lines-per-file=100 \
    disk-stop-on-full=no
```

---

## 10. Firewall Testing

### 10.1 Test Rules ด้วย Ping

```bash
# ทดสอบ INPUT rules
# จาก PC ใน LAN
ping 192.168.1.1  # ควร succeed

# จาก PC ใน WAN (simulate)
# ใช้ /ping จาก router โดยระบุ src-address เป็น WAN IP
/ping 8.8.8.8 src-address=203.0.113.1 count=3

# ทดสอบ FORWARD rules
# จาก LAN PC ping ออก internet
/ping 8.8.8.8 count=3  # จาก router เอง
```

### 10.2 Rule Statistics

```bash
# ดู bytes/packets ที่ match แต่ละ rule
/ip firewall filter print stats

# Output:
#  #    CHAIN   BYTES    PACKETS   ACTION  COMMENT
#  0    input   1234567  1234      accept  Accept established
#  1    input   0        0         drop    Drop invalid
#  2    input   8901     89        accept  Accept from LAN
#  3    input   456      4         drop    Drop all input

# ดู rule ที่ไม่เคย match (bytes=0)
/ip firewall filter print stats where bytes=0
```

### 10.3 Connection Tracking Verify

```bash
# ดู active connections ผ่าน firewall
/ip firewall connection print

# Filter connections from WAN
/ip firewall connection print where dst-address~"192.168"

# ดู connection states
/ip firewall connection print where tcp-state=established

# Monitor connections real-time
/ip firewall connection print interval=2
```

### 10.4 Packet Capture สำหรับ Debug

```bash
# Capture บน interface เพื่อดู traffic
/tool sniffer start \
    interface=WAN \
    filter-ip-address=203.0.113.10 \
    file-name=test-capture.pcap

# ทดสอบ connection ในระหว่าง capture

/tool sniffer stop

# ดู capture (ต้อง download ไปใช้ Wireshark)
/file print where name~"test-capture"
```

### 10.5 Firewall Testing Checklist

```bash
# Checklist:

□ Test: Internet access จาก LAN clients
  → /ping 8.8.8.8 (จาก client)

□ Test: DNS resolution
  → /ping google.com count=1 (จาก client)

□ Test: Port forwarding
  → Connect ไปยัง public IP:port จาก internet (ต้องใช้ phone 4G)

□ Test: Block WAN access ไปยัง router
  → /ping router-WAN-IP (จาก external device)
  → ควร timeout

□ Test: Firewall logs ทำงาน
  → /log print where topics~"firewall"
  → ควรเห็น log entries

□ Test: Anti-spoofing
  → ส่ง packet มาจาก WAN ด้วย src-address=192.168.x.x
  → ควรถูก drop

□ Test: ICMP rate limiting
  → ping flood ไปยัง router
  → ควรถูก rate limit
```

---

## 11. Lab: Complete Basic Firewall

### Lab Topology

```
    Internet
        │
   ┌────┴────┐
   │WAN      │ ether1
   │ Router  │ 192.168.1.1/24 (ether2=LAN)
   │         │ 10.0.0.1/24   (ether3=DMZ)
   └────┬────┘
        │
   ┌────┼─────────┐
   │    │         │
LAN │  DMZ      WAN
192.168.1.x  10.0.0.x
```

### Step 1: Interface Lists

```bash
/interface list add name=WAN
/interface list add name=LAN
/interface list add name=DMZ

/interface list member add interface=ether1 list=WAN
/interface list member add interface=ether2 list=LAN
/interface list member add interface=ether3 list=DMZ
```

### Step 2: Address Lists

```bash
# Bogon IPs
/ip firewall address-list add list=bogons address=10.0.0.0/8
/ip firewall address-list add list=bogons address=172.16.0.0/12
/ip firewall address-list add list=bogons address=192.168.0.0/16

# Management hosts
/ip firewall address-list add list=trusted-admin address=192.168.1.100 comment="Admin PC"
```

### Step 3: INPUT Chain

```bash
# Remove default rules ถ้ามี
/ip firewall filter remove [find]

# INPUT chain

# 1. Accept established/related
/ip firewall filter add \
    chain=input \
    connection-state=established,related \
    action=accept \
    comment="[INPUT] Accept established"

# 2. Drop invalid
/ip firewall filter add \
    chain=input \
    connection-state=invalid \
    action=drop \
    comment="[INPUT] Drop invalid"

# 3. Accept loopback
/ip firewall filter add \
    chain=input \
    in-interface=lo \
    action=accept \
    comment="[INPUT] Accept loopback"

# 4. Drop bogons from WAN
/ip firewall filter add \
    chain=input \
    in-interface-list=WAN \
    src-address-list=bogons \
    action=drop \
    comment="[INPUT] Drop bogons"

# 5. Accept ICMP with rate limit
/ip firewall filter add \
    chain=input \
    protocol=icmp \
    limit=10,20:packet \
    action=accept \
    comment="[INPUT] Accept ICMP"

/ip firewall filter add \
    chain=input \
    protocol=icmp \
    action=drop \
    comment="[INPUT] Drop ICMP flood"

# 6. SSH brute force protection
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    connection-state=new \
    action=add-src-to-address-list \
    address-list=ssh-attempts \
    address-list-timeout=5m \
    comment="[INPUT] Track SSH"

/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    connection-state=new \
    src-address-list=ssh-attempts \
    connection-limit=3,32 \
    action=add-src-to-address-list \
    address-list=ssh-bruteforce \
    address-list-timeout=1d \
    comment="[INPUT] Mark SSH bruteforce"

/ip firewall filter add \
    chain=input \
    src-address-list=ssh-bruteforce \
    action=drop \
    log=yes \
    log-prefix="SSH-BF: " \
    comment="[INPUT] Block SSH bruteforce"

# 7. Accept management from LAN
/ip firewall filter add \
    chain=input \
    in-interface-list=LAN \
    action=accept \
    comment="[INPUT] Accept from LAN"

# 8. Accept specific ports (if needed from WAN)
# ตัวอย่าง: accept VPN
/ip firewall filter add \
    chain=input \
    in-interface-list=WAN \
    protocol=udp \
    dst-port=51820 \
    action=accept \
    comment="[INPUT] Accept WireGuard VPN"

# 9. Drop everything else
/ip firewall filter add \
    chain=input \
    action=drop \
    log=yes \
    log-prefix="DROP-INPUT: " \
    comment="[INPUT] Drop all else"
```

### Step 4: FORWARD Chain

```bash
# FORWARD chain

# 1. Accept established/related (FastTrack for performance)
/ip firewall filter add \
    chain=forward \
    connection-state=established,related \
    action=fasttrack-connection \
    comment="[FWD] FastTrack established"

/ip firewall filter add \
    chain=forward \
    connection-state=established,related \
    action=accept \
    comment="[FWD] Accept established"

# 2. Drop invalid
/ip firewall filter add \
    chain=forward \
    connection-state=invalid \
    action=drop \
    comment="[FWD] Drop invalid"

# 3. Drop bogons from WAN
/ip firewall filter add \
    chain=forward \
    in-interface-list=WAN \
    src-address-list=bogons \
    action=drop \
    comment="[FWD] Drop bogons from WAN"

# 4. Allow LAN → Internet
/ip firewall filter add \
    chain=forward \
    in-interface-list=LAN \
    out-interface-list=WAN \
    action=accept \
    comment="[FWD] LAN to Internet"

# 5. Allow LAN → DMZ
/ip firewall filter add \
    chain=forward \
    in-interface-list=LAN \
    out-interface-list=DMZ \
    action=accept \
    comment="[FWD] LAN to DMZ"

# 6. Allow specific port forwards (HTTP/HTTPS to DMZ)
/ip firewall filter add \
    chain=forward \
    in-interface-list=WAN \
    dst-address=10.0.0.10 \
    protocol=tcp \
    dst-port=80,443 \
    action=accept \
    comment="[FWD] Allow HTTP/HTTPS to DMZ web server"

# 7. Allow DMZ → Internet (limited)
/ip firewall filter add \
    chain=forward \
    in-interface-list=DMZ \
    out-interface-list=WAN \
    protocol=tcp \
    dst-port=80,443,25,587,993 \
    action=accept \
    comment="[FWD] DMZ outbound services"

# 8. Block DMZ → LAN
/ip firewall filter add \
    chain=forward \
    in-interface-list=DMZ \
    out-interface-list=LAN \
    action=drop \
    log=yes \
    log-prefix="DMZ-TO-LAN-BLOCK: " \
    comment="[FWD] Block DMZ to LAN"

# 9. Block WAN → LAN new connections
/ip firewall filter add \
    chain=forward \
    in-interface-list=WAN \
    connection-state=new \
    action=drop \
    log=yes \
    log-prefix="WAN-TO-LAN-BLOCK: " \
    comment="[FWD] Block unsolicited WAN to LAN"

# 10. Drop everything else
/ip firewall filter add \
    chain=forward \
    action=drop \
    comment="[FWD] Drop all else"
```

### Step 5: NAT Rules

```bash
# Masquerade
/ip firewall nat add \
    chain=srcnat \
    out-interface-list=WAN \
    action=masquerade \
    comment="Internet NAT"

# Port forward to DMZ web server
/ip firewall nat add \
    chain=dstnat \
    in-interface-list=WAN \
    protocol=tcp \
    dst-port=80 \
    action=dst-nat \
    to-addresses=10.0.0.10 \
    to-ports=80 \
    comment="HTTP to DMZ"

/ip firewall nat add \
    chain=dstnat \
    in-interface-list=WAN \
    protocol=tcp \
    dst-port=443 \
    action=dst-nat \
    to-addresses=10.0.0.10 \
    to-ports=443 \
    comment="HTTPS to DMZ"
```

### Step 6: Verification

```bash
# ดู rules ทั้งหมด
/ip firewall filter print

# ดู NAT rules
/ip firewall nat print

# ตรวจสอบ statistics
/ip firewall filter print stats

# ทดสอบ
# 1. Ping internet จาก LAN
/ping 8.8.8.8 count=3

# 2. ดู logs
/log print where topics~"firewall"

# 3. ดู blocked connections
/log print where message~"DROP"

# 4. ดู active connections
/ip firewall connection print count-only
```

---

## 12. Summary

### สิ่งที่เรียนรู้ใน Part 10:

1. **Chains** Input (traffic มาหา router), Forward (traffic ผ่าน router), Output (traffic จาก router)
2. **Actions** Accept, Drop, Reject, Log, Add-to-address-list
3. **Connection Tracking** ใช้ state-based filtering สำหรับ stateful firewall
4. **Common Rules** Anti-spoofing, Bogon filtering, ICMP protection
5. **DDoS Protection** SYN flood, UDP flood, ICMP flood, connection flood
6. **Port Scan Detection** ใช้ address-list และ honeypot ports
7. **Input Protection** ป้องกัน management interfaces, SSH brute force
8. **Logging** Log สำคัญ events สำหรับ audit และ troubleshooting

### Firewall Rule Order Template

```bash
# INPUT Chain Order:
1. Accept established/related
2. Accept loopback
3. Drop invalid
4. Drop bogons (from WAN)
5. Rate-limit ICMP
6. Brute-force protection
7. Accept management (from trusted)
8. Accept specific services
9. Drop everything else

# FORWARD Chain Order:
1. FastTrack/Accept established
2. Drop invalid
3. Drop bogons
4. Specific allow rules
5. General LAN→WAN allow
6. Block WAN→LAN new
7. Drop everything else
```

### Quick Reference

```bash
# ดู rules
/ip firewall filter print
/ip firewall filter print stats

# ดู connections
/ip firewall connection print

# ดู address-lists
/ip firewall address-list print

# Monitor logs
/log print follow where topics~"firewall"

# Flush address-list
/ip firewall address-list remove [find list=ssh-bruteforce]
```

---

## อ้างอิง

- [MikroTik Firewall Wiki](https://wiki.mikrotik.com/wiki/Manual:IP/Firewall)
- [RouterOS Firewall Examples](https://wiki.mikrotik.com/wiki/Manual:Securing_Your_Router)
- [MikroTik Forum - Firewall](https://forum.mikrotik.com)
- [Port Scan Detection](https://wiki.mikrotik.com/wiki/Port_Scan_Protection)

---

**[⬅ Previous: NAT & Masquerade](part-009-nat-masquerade.md)** | **[Back to Index ➡](../README.md)**

---

*หลักสูตร MikroTik Network Engineering - จบ Part 10*  
*ขั้นต่อไป: Part 11 - Routing Protocols (OSPF, BGP)*
