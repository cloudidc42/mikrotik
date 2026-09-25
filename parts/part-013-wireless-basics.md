# Part 13: Wireless Basics

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate
**เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

- [Wireless Interface Types](#wireless-interface-types)
- [802.11 Standards](#80211-standards)
- [Security Modes](#security-modes)
- [SSID Configuration](#ssid-configuration)
- [Channel and Frequency](#channel-and-frequency)
- [TX Power](#tx-power)
- [Wireless Profiles](#wireless-profiles)
- [Access Control List](#access-control-list)
- [Site Survey](#site-survey)
- [Lab: Wireless AP Setup](#lab)

---

## Wireless Interface Types

### ประเภทของ Wireless Interface

RouterOS รองรับ Wireless Interface หลายประเภท:

| Interface Type | คำอธิบาย | การใช้งาน |
|---------------|----------|----------|
| `wlan1`, `wlan2` | Physical Wireless Interface | AP, Station, Bridge |
| Virtual AP (VAP) | Virtual Interface บน Physical | Multi-SSID |
| Station | เชื่อมต่อไปยัง AP อื่น | Repeater, CPE |
| Bridge | Wireless Bridge Mode | Point-to-Point |

### Wireless Interface Modes

```routeros
# ดู Wireless Interfaces
/interface wireless print

# Modes ที่รองรับ:
# ap-bridge    = Access Point (ให้ Client เชื่อมต่อ, หลาย Client)
# station      = Station Mode (เชื่อมต่อไป AP อื่น)
# station-bridge = Station Bridge (เชื่อมต่อและ Bridge)
# bridge       = Point-to-Point Bridge
# wds-slave    = WDS Slave
# alignment-only = ปรับ Antenna
# nstreme-dual-slave = Nstreme Dual
```

### Virtual Access Point (VAP)

```routeros
# สร้าง Virtual AP (Multi-SSID)
/interface wireless add \
    master-interface=wlan1 \
    mode=ap-bridge \
    name=wlan1-guest \
    ssid="Guest-Network" \
    comment="Guest WiFi VAP"

# สร้าง VAP อีกอัน
/interface wireless add \
    master-interface=wlan1 \
    mode=ap-bridge \
    name=wlan1-staff \
    ssid="Staff-Network" \
    comment="Staff WiFi VAP"

# ดู Virtual Interfaces
/interface wireless print
```

---

## 802.11 Standards

### มาตรฐาน WiFi

| Standard | ชื่อ WiFi | ความถี่ | Speed สูงสุด | ระยะ |
|----------|----------|---------|------------|------|
| 802.11b | WiFi 1 | 2.4 GHz | 11 Mbps | 35m |
| 802.11a | WiFi 2 | 5 GHz | 54 Mbps | 35m |
| 802.11g | WiFi 3 | 2.4 GHz | 54 Mbps | 38m |
| 802.11n | WiFi 4 | 2.4/5 GHz | 600 Mbps | 70m |
| 802.11ac | WiFi 5 | 5 GHz | 3.5 Gbps | 35m |
| 802.11ax | WiFi 6 | 2.4/5/6 GHz | 9.6 Gbps | 35m |

### กำหนด Band และ Standard

```routeros
# 2.4 GHz - รองรับ b/g/n
/interface wireless set wlan1 \
    band=2ghz-b/g/n \
    channel-width=20/40mhz-Ce \
    comment="2.4GHz band"

# 5 GHz - รองรับ a/n/ac
/interface wireless set wlan1 \
    band=5ghz-a/n/ac \
    channel-width=20/40/80mhz \
    comment="5GHz band"

# 2.4 GHz เฉพาะ n
/interface wireless set wlan1 \
    band=2ghz-onlyn \
    channel-width=20/40mhz-XX

# 5 GHz เฉพาะ ac
/interface wireless set wlan1 \
    band=5ghz-onlyac \
    channel-width=20/40/80mhz
```

### Channel Width

```routeros
# 2.4 GHz Channel Widths
# 20MHz  = 150 Mbps (ลด Interference)
# 40MHz  = 300 Mbps (เพิ่ม Throughput)

# 5 GHz Channel Widths
# 20MHz  = 150 Mbps
# 40MHz  = 300 Mbps
# 80MHz  = 433 Mbps
# 160MHz = 867 Mbps (ไม่ค่อยใช้, มี Interference สูง)

/interface wireless set wlan1 channel-width=20mhz      # Conservative
/interface wireless set wlan1 channel-width=20/40mhz-Ce  # Auto 20/40
/interface wireless set wlan1 channel-width=20/40/80mhz  # 5GHz AC
```

---

## Security Modes

### ประเภทของ Security

| Mode | คำอธิบาย | ความปลอดภัย |
|------|----------|------------|
| none | ไม่มีการเข้ารหัส | ต่ำมาก |
| wep40 | WEP 40-bit | ต่ำมาก (ถูก Crack ได้ง่าย) |
| wep104 | WEP 104-bit | ต่ำ (ยังถูก Crack ได้) |
| wpa-psk | WPA Personal (TKIP) | ปานกลาง |
| wpa2-psk | WPA2 Personal (AES) | ดี |
| wpa-eap | WPA Enterprise (802.1X) | ดีมาก |
| wpa2-eap | WPA2 Enterprise (802.1X) | ดีมาก |
| wpa3 | WPA3 (SAE) | ดีที่สุด |

### ตั้งค่า WPA2

```routeros
# สร้าง Security Profile สำหรับ WPA2
/interface wireless security-profiles add \
    name=wpa2-profile \
    mode=dynamic-keys \
    authentication-types=wpa2-psk \
    unicast-ciphers=aes-ccm \
    group-ciphers=aes-ccm \
    wpa2-pre-shared-key="MySecurePassword123!" \
    comment="WPA2 Personal"

# ใช้งาน Security Profile
/interface wireless set wlan1 security-profile=wpa2-profile
```

### ตั้งค่า WPA3 (RouterOS v7)

```routeros
# WPA3 Personal (SAE)
/interface wireless security-profiles add \
    name=wpa3-profile \
    mode=dynamic-keys \
    authentication-types=wpa3-psk \
    unicast-ciphers=aes-ccm \
    group-ciphers=aes-ccm \
    wpa2-pre-shared-key="MySecurePassword123!" \
    comment="WPA3 Personal"

# WPA2/WPA3 Mixed (Transition Mode)
/interface wireless security-profiles add \
    name=wpa23-mixed \
    mode=dynamic-keys \
    authentication-types=wpa2-psk,wpa3-psk \
    unicast-ciphers=aes-ccm \
    group-ciphers=aes-ccm \
    wpa2-pre-shared-key="MySecurePassword123!" \
    comment="WPA2/WPA3 Mixed Mode"
```

### WPA2 Enterprise (802.1X + RADIUS)

```routeros
# Security Profile สำหรับ Enterprise
/interface wireless security-profiles add \
    name=enterprise-profile \
    mode=dynamic-keys \
    authentication-types=wpa2-eap \
    unicast-ciphers=aes-ccm \
    group-ciphers=aes-ccm \
    eap-methods=passthrough \
    comment="WPA2 Enterprise"

# กำหนด RADIUS Server
/radius add \
    address=192.168.10.10 \
    secret=radius-shared-secret \
    service=wireless \
    comment="RADIUS Server"

# ใช้ Security Profile Enterprise
/interface wireless set wlan1 security-profile=enterprise-profile
```

---

## SSID Configuration

### SSID คืออะไร?

**SSID (Service Set Identifier)** คือชื่อของ Wireless Network ที่ Client จะเห็นใน List ของ WiFi Networks

### ตั้งค่า SSID

```routeros
# ตั้ง SSID
/interface wireless set wlan1 ssid="MyOffice-WiFi"

# SSID ภาษาไทย (Unicode)
/interface wireless set wlan1 ssid="ออฟฟิศ WiFi"

# ซ่อน SSID (Hidden Network)
/interface wireless set wlan1 \
    ssid="HiddenNetwork" \
    hide-ssid=yes

# ดู SSID ปัจจุบัน
/interface wireless print
```

### Multi-SSID Setup

```routeros
# SSID หลัก (Staff)
/interface wireless set wlan1 \
    mode=ap-bridge \
    ssid="Staff-WiFi" \
    security-profile=wpa2-staff

# SSID สำรอง (Guest)
/interface wireless add \
    master-interface=wlan1 \
    mode=ap-bridge \
    name=wlan1-guest \
    ssid="Guest-WiFi" \
    security-profile=wpa2-guest

# แต่ละ SSID สามารถอยู่คนละ VLAN ได้
/interface bridge port add interface=wlan1 bridge=bridge1 pvid=20
/interface bridge port add interface=wlan1-guest bridge=bridge1 pvid=30
```

---

## Channel and Frequency

### ทำความเข้าใจ Channels

**2.4 GHz Channels:**
```
Channel  1: 2412 MHz
Channel  2: 2417 MHz
Channel  3: 2422 MHz
Channel  4: 2427 MHz
Channel  5: 2432 MHz
Channel  6: 2437 MHz
Channel  7: 2442 MHz
Channel  8: 2447 MHz
Channel  9: 2452 MHz
Channel 10: 2457 MHz
Channel 11: 2462 MHz
Channel 12: 2467 MHz (ไม่ใช้ในบางประเทศ)
Channel 13: 2472 MHz (ไม่ใช้ในบางประเทศ)

Non-Overlapping Channels: 1, 6, 11
```

**5 GHz Channels (ตัวอย่าง Thailand):**
```
UNII-1: 36, 40, 44, 48
UNII-2: 52, 56, 60, 64 (DFS)
UNII-2e: 100-144 (DFS)
UNII-3: 149, 153, 157, 161, 165
```

### ตั้งค่า Channel

```routeros
# กำหนด Channel เฉพาะ
/interface wireless set wlan1 frequency=2437    # Channel 6, 2.4GHz
/interface wireless set wlan1 frequency=5180    # Channel 36, 5GHz

# Auto Channel Selection
/interface wireless set wlan1 frequency=auto

# กำหนด Channel Width
/interface wireless set wlan1 channel-width=20mhz

# กำหนด Country (สำคัญสำหรับ Channel Regulations)
/interface wireless set wlan1 country="thailand"
```

### DFS (Dynamic Frequency Selection)

```routeros
# DFS ใช้สำหรับ 5GHz ช่วง Channel 52-144
# RouterOS ต้องตรวจสอบ Radar ก่อนใช้งาน (CAC - Channel Availability Check)
# ใช้เวลา 60 วินาที

/interface wireless set wlan1 \
    frequency=5260 \    # DFS Channel
    dfs-mode=radar-detect \
    comment="5GHz with DFS"
```

### Frequency Scan

```routeros
# Scan หา Channel ที่ใช้งานน้อยที่สุด
/interface wireless scan wlan1

# Scan แบบ Background
/interface wireless background-scan wlan1

# ดูผลการ Scan
/interface wireless scan-list print
```

---

## TX Power

### TX Power คืออะไร?

**TX Power** คือกำลังส่งของ Wireless Signal วัดเป็น **dBm** (Decibel-milliwatts)

| dBm | mW | คำอธิบาย |
|-----|-----|---------|
| 0 | 1 | ต่ำมาก |
| 10 | 10 | ต่ำ |
| 20 | 100 | กลาง |
| 23 | 200 | ดี |
| 30 | 1000 | สูง |
| 33 | 2000 | สูงมาก |

### ตั้งค่า TX Power

```routeros
# กำหนด TX Power
/interface wireless set wlan1 tx-power=20 tx-power-mode=all-rates-fixed

# TX Power Modes
# all-rates-fixed  = ใช้ Power เดียวกันทุก Rate
# manual-table     = กำหนดต่อ Rate
# card-rates       = ใช้ค่า Default ของ Card

# ดู TX Power ปัจจุบัน
/interface wireless print detail
```

### ผลกระทบของ TX Power

```
สูง TX Power:
  ✓ ครอบคลุมพื้นที่กว้าง
  ✗ รบกวน AP อื่น
  ✗ ใช้ไฟมากขึ้น

ต่ำ TX Power:
  ✓ ลด Interference
  ✓ ประหยัดไฟ
  ✗ ครอบคลุมพื้นที่แคบ
```

> **Note:** ในหลายประเทศมีกฎหมายควบคุม TX Power สูงสุด ตรวจสอบให้แน่ใจว่าตั้งค่าไม่เกิน Legal Limit

---

## Wireless Profiles

### Security Profile

```routeros
# ดู Security Profiles ทั้งหมด
/interface wireless security-profiles print

# สร้าง Profile สำหรับ Staff
/interface wireless security-profiles add \
    name=staff-security \
    mode=dynamic-keys \
    authentication-types=wpa2-psk \
    unicast-ciphers=aes-ccm \
    group-ciphers=aes-ccm \
    wpa2-pre-shared-key="StaffPassword2026!"

# สร้าง Profile สำหรับ Guest
/interface wireless security-profiles add \
    name=guest-security \
    mode=dynamic-keys \
    authentication-types=wpa2-psk \
    unicast-ciphers=aes-ccm \
    group-ciphers=aes-ccm \
    wpa2-pre-shared-key="GuestWifi2026"
```

### Wireless Configuration Profile

```routeros
# กำหนด Wireless Configuration ผ่าน Interface settings
/interface wireless set wlan1 \
    mode=ap-bridge \
    band=2ghz-b/g/n \
    channel-width=20/40mhz-Ce \
    frequency=auto \
    ssid="Office-2.4G" \
    security-profile=staff-security \
    country=thailand \
    installation=indoor \
    antenna-gain=0 \
    tx-power=17 \
    tx-power-mode=all-rates-fixed \
    comment="2.4GHz Main AP"
```

### WPS (WiFi Protected Setup)

```routeros
# เปิดใช้ WPS (ไม่แนะนำ - มีช่องโหว่)
# RouterOS มี WPS support แต่ควรปิดไว้

# ปิด WPS
/interface wireless set wlan1 wps-mode=disabled
```

---

## Access Control List

### Wireless ACL คืออะไร?

**Wireless Access Control List (ACL)** ควบคุมว่า Client ไหนสามารถเชื่อมต่อกับ AP ได้บ้าง

### ประเภทของ ACL

1. **Default Allow** - อนุญาตทุกคน ยกเว้นที่ Deny ไว้
2. **Default Deny** - ปฏิเสธทุกคน ยกเว้นที่ Allow ไว้

### ตั้งค่า ACL

```routeros
# Default Policy
/interface wireless access-list add \
    mac-address=FF:FF:FF:FF:FF:FF \
    interface=wlan1 \
    action=accept \
    comment="Default allow all"

# Allow เฉพาะ MAC Address
/interface wireless access-list add \
    mac-address=AA:BB:CC:DD:EE:01 \
    interface=wlan1 \
    action=accept \
    comment="Allow PC-A"

/interface wireless access-list add \
    mac-address=AA:BB:CC:DD:EE:02 \
    interface=wlan1 \
    action=accept \
    comment="Allow Laptop-1"

# Deny MAC Address ที่กำหนด
/interface wireless access-list add \
    mac-address=11:22:33:44:55:66 \
    interface=wlan1 \
    action=reject \
    comment="Block intruder"

# Default Deny (ใส่ไว้สุดท้าย)
/interface wireless set wlan1 default-authentication=no

# ดู ACL
/interface wireless access-list print
```

### กำหนด Per-Client TX Power และ Rate Limit

```routeros
# จำกัด Speed ต่อ Client
/interface wireless access-list add \
    mac-address=AA:BB:CC:DD:EE:01 \
    interface=wlan1 \
    action=accept \
    ap-tx-limit=5000000 \    # AP TX จำกัด 5 Mbps
    client-tx-limit=2000000 \ # Client TX จำกัด 2 Mbps
    comment="Limited client"
```

---

## Site Survey

### Site Survey คืออะไร?

**Site Survey** คือการสำรวจพื้นที่เพื่อวิเคราะห์ Wireless Environment ก่อนติดตั้ง AP

### Scan Networks รอบข้าง

```routeros
# Scan เพื่อดู Networks รอบข้าง
/interface wireless scan wlan1

# Output ตัวอย่าง:
# ADDRESS            SSID              BAND     CHANNEL  SIG   SNR  RADIO-NAME
# AA:BB:CC:DD:EE:01  OfficeNet         2GHz-n   6        -65   30   Mikrotik
# AA:BB:CC:DD:EE:02  HomeWifi          2GHz-g   11       -70   25   Unknown
# AA:BB:CC:DD:EE:03  Cafe-Free         2GHz-b/g 6        -80   15   Unknown
```

### Frequency Usage Monitor

```routeros
# ดู Frequency Usage
/interface wireless frequency-monitor wlan1

# Monitor Signal Quality
/interface wireless monitor wlan1
```

### Sniffer Mode

```routeros
# Packet Sniffer (ต้อง Disconnect Clients ก่อน)
/interface wireless set wlan1 mode=sniff

# กลับเป็น AP Mode
/interface wireless set wlan1 mode=ap-bridge
```

### Registration Table

```routeros
# ดู Clients ที่เชื่อมต่ออยู่
/interface wireless registration-table print

# ดูแบบ Detail (Signal, Rates, etc.)
/interface wireless registration-table print detail

# Monitor Clients แบบ Realtime
/interface wireless registration-table print interval=5

# Output ตัวอย่าง:
# INTERFACE  RADIO-NAME  MAC-ADDRESS        UPTIME  SIGNAL  TX-RATE  RX-RATE
# wlan1      -           AA:BB:CC:DD:EE:01  5m32s   -65dBm  54Mbps   54Mbps
```

---

## Lab: Wireless AP Setup

### Network Topology

```
Internet
    │
[MikroTik Router]
    ├── ether1: WAN (DHCP from ISP)
    ├── ether2: LAN (192.168.1.1/24)
    └── wlan1:  WiFi AP
                ├── SSID: "Office-Staff" (VLAN 20)
                └── SSID: "Office-Guest" (VLAN 30)
```

### Objectives

1. ตั้งค่า AP Mode บน wlan1
2. สร้าง 2 SSID (Staff และ Guest)
3. Staff ใช้ WPA2 Password แข็งแรง
4. Guest ใช้ Password ง่าย แต่ Isolated
5. ทั้งสองสามารถออก Internet ได้

### Step 1: ตั้งค่า Basic Wireless

```routeros
# ตั้งค่า Country และ Mode
/interface wireless set wlan1 \
    country=thailand \
    mode=ap-bridge \
    disabled=no

# ดู Interface
/interface wireless print
```

### Step 2: สร้าง Security Profiles

```routeros
# Staff Profile - WPA2 แข็งแรง
/interface wireless security-profiles add \
    name=staff-sec \
    mode=dynamic-keys \
    authentication-types=wpa2-psk \
    unicast-ciphers=aes-ccm \
    group-ciphers=aes-ccm \
    wpa2-pre-shared-key="S@ff_P@ss2026!" \
    comment="Staff WPA2"

# Guest Profile - WPA2 ง่าย
/interface wireless security-profiles add \
    name=guest-sec \
    mode=dynamic-keys \
    authentication-types=wpa2-psk \
    unicast-ciphers=aes-ccm \
    group-ciphers=aes-ccm \
    wpa2-pre-shared-key="GuestWifi2026" \
    comment="Guest WPA2"
```

### Step 3: ตั้งค่า Main Interface (Staff)

```routeros
# ตั้งค่า wlan1 สำหรับ Staff
/interface wireless set wlan1 \
    mode=ap-bridge \
    band=2ghz-b/g/n \
    channel-width=20/40mhz-Ce \
    frequency=auto \
    ssid="Office-Staff" \
    security-profile=staff-sec \
    country=thailand \
    installation=indoor \
    tx-power=17 \
    tx-power-mode=all-rates-fixed \
    comment="Staff WiFi 2.4GHz"
```

### Step 4: สร้าง Guest Virtual AP

```routeros
# สร้าง Virtual AP สำหรับ Guest
/interface wireless add \
    master-interface=wlan1 \
    mode=ap-bridge \
    name=wlan1-guest \
    ssid="Office-Guest" \
    security-profile=guest-sec \
    comment="Guest WiFi VAP"
```

### Step 5: ตั้งค่า Bridge และ VLAN

```routeros
# สร้าง Bridge
/interface bridge add name=br-wifi protocol-mode=none

# เพิ่ม Wireless Interfaces เข้า Bridge
/interface bridge port add interface=wlan1 bridge=br-wifi pvid=20
/interface bridge port add interface=wlan1-guest bridge=br-wifi pvid=30

# เพิ่ม ether2 เข้า Bridge (LAN)
/interface bridge port add interface=ether2 bridge=br-wifi pvid=10

# สร้าง VLAN Interfaces
/interface vlan add name=vlan20-staff interface=br-wifi vlan-id=20
/interface vlan add name=vlan30-guest interface=br-wifi vlan-id=30

# ตั้งค่า IP
/ip address add address=192.168.20.1/24 interface=vlan20-staff
/ip address add address=192.168.30.1/24 interface=vlan30-guest

# VLAN Table
/interface bridge vlan add bridge=br-wifi vlan-ids=10 tagged=br-wifi untagged=ether2
/interface bridge vlan add bridge=br-wifi vlan-ids=20 tagged=br-wifi untagged=wlan1
/interface bridge vlan add bridge=br-wifi vlan-ids=30 tagged=br-wifi untagged=wlan1-guest

# เปิด VLAN Filtering
/interface bridge set br-wifi vlan-filtering=yes
```

### Step 6: ตั้งค่า DHCP

```routeros
# IP Pool
/ip pool add name=pool-staff ranges=192.168.20.100-192.168.20.200
/ip pool add name=pool-guest ranges=192.168.30.100-192.168.30.200

# DHCP Networks
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1 dns-server=8.8.8.8
/ip dhcp-server network add address=192.168.30.0/24 gateway=192.168.30.1 dns-server=8.8.8.8

# DHCP Servers
/ip dhcp-server add name=dhcp-staff interface=vlan20-staff address-pool=pool-staff disabled=no
/ip dhcp-server add name=dhcp-guest interface=vlan30-guest address-pool=pool-guest disabled=no
```

### Step 7: Firewall Rules

```routeros
# Guest Isolation - ป้องกัน Guest เข้าถึง Staff Network
/ip firewall filter add \
    chain=forward \
    src-address=192.168.30.0/24 \
    dst-address=192.168.20.0/24 \
    action=drop \
    comment="Block Guest to Staff"

# Guest Isolation - ป้องกัน Guest เข้าถึง LAN
/ip firewall filter add \
    chain=forward \
    src-address=192.168.30.0/24 \
    dst-address=192.168.1.0/24 \
    action=drop \
    comment="Block Guest to LAN"

# NAT
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade
```

### Step 8: ตรวจสอบ

```routeros
# ดู Wireless Interfaces
/interface wireless print

# ดู Clients ที่เชื่อมต่อ
/interface wireless registration-table print

# ดู DHCP Leases
/ip dhcp-server lease print

# ทดสอบ Connectivity
/ping 8.8.8.8
```

### ผลลัพธ์ที่คาดหวัง

```
/interface wireless print
Flags: X - disabled, R - running, S - slave

 #    NAME      MTU   MAC-ADDRESS        SSID                MODE
 0  R wlan1     1500  AA:BB:CC:DD:EE:01  Office-Staff        ap-bridge
 1  RS wlan1-guest 1500 AA:BB:CC:DD:EE:01 Office-Guest      ap-bridge

/interface wireless registration-table print
 #  INTERFACE  MAC-ADDRESS        SSID          SIGNAL
 0  wlan1      11:22:33:44:55:01  Office-Staff  -60dBm
 1  wlan1-guest 11:22:33:44:55:02 Office-Guest -70dBm
```

---

## Advanced Wireless Topics

### Wireless Repeater (Range Extender)

```routeros
# Router-A เป็น AP หลัก
# Router-B เป็น Repeater (Station Bridge Mode)

# Router-B: เชื่อมต่อไปยัง Router-A
/interface wireless set wlan1 \
    mode=station-bridge \
    ssid="Office-Staff" \
    security-profile=staff-sec

# Router-B: สร้าง AP บน wlan2 สำหรับ Devices รอบข้าง
/interface wireless set wlan2 \
    mode=ap-bridge \
    ssid="Office-Ext" \
    security-profile=staff-sec

# Bridge ทั้งสองเข้าด้วยกัน
/interface bridge port add interface=wlan1 bridge=bridge1
/interface bridge port add interface=wlan2 bridge=bridge1
```

### WDS (Wireless Distribution System)

```routeros
# AP หลัก: เปิด WDS
/interface wireless set wlan1 \
    mode=ap-bridge \
    wds-mode=dynamic \
    wds-default-bridge=bridge1

# AP สาขา: WDS Slave
/interface wireless set wlan1 \
    mode=wds-slave \
    scan-list=default \
    wds-master-bridge=bridge1
```

---

## Troubleshooting

### ปัญหาที่พบบ่อย

**1. Client เชื่อมต่อไม่ได้**
```routeros
# ตรวจสอบ Security Profile
/interface wireless security-profiles print

# ตรวจสอบ ACL
/interface wireless access-list print

# ดู Log สำหรับ Authentication Failures
/log print where message~"wireless"
```

**2. Signal อ่อน**
```routeros
# เพิ่ม TX Power
/interface wireless set wlan1 tx-power=23

# เปลี่ยน Channel เพื่อลด Interference
/interface wireless set wlan1 frequency=auto

# ดู Noise Floor
/interface wireless monitor wlan1
```

**3. Throughput ต่ำ**
```routeros
# ตรวจสอบ Channel Width
/interface wireless print detail

# ดู TX/RX Rates ของ Clients
/interface wireless registration-table print detail

# ตรวจสอบว่าใช้ AES ไม่ใช่ TKIP
/interface wireless security-profiles print detail
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 13:

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| Interface Types | ap-bridge, station, Virtual AP |
| 802.11 Standards | b/g/n (2.4GHz), a/n/ac/ax (5GHz) |
| Security Modes | ใช้ WPA2/WPA3, หลีกเลี่ยง WEP |
| SSID | ชื่อ Network, รองรับ Multi-SSID |
| Channels | 2.4GHz: 1,6,11 (Non-overlapping), 5GHz: หลาย Channel |
| TX Power | กำลังส่ง, ระวัง Legal Limit |
| Security Profiles | สร้างครั้งเดียว ใช้ได้หลาย Interface |
| ACL | ควบคุม Client ที่เชื่อมต่อได้ |

---

## แบบทดสอบ

1. Virtual AP คืออะไร แตกต่างจาก Physical Interface อย่างไร?
2. ทำไม WEP ถึงไม่ควรใช้?
3. Non-Overlapping Channels ใน 2.4GHz คือ Channel อะไรบ้าง?
4. DFS ใช้ทำอะไร และ Channel ไหนต้องใช้?
5. PVID กับ VLAN Filtering มีความสัมพันธ์กันอย่างไรใน Wireless?

---

## Navigation

[← Part 12: Bridge Configuration](part-012-bridge-configuration.md) | [Part 14: User Management →](part-014-user-management.md)

---

*MikroTik RouterOS Administration Course - Part 13*
*Last Updated: 2026*
