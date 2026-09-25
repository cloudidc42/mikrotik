# Part 4: Winbox, WebFig และ CLI

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner  
> **เวลาเรียน:** 2-3 ชั่วโมง

---

## สารบัญ

1. [Winbox Installation และ Usage](#1-winbox-installation-และ-usage)
2. [Winbox Features ทั้งหมด](#2-winbox-features-ทั้งหมด)
3. [WebFig Overview](#3-webfig-overview)
4. [SSH Access Setup](#4-ssh-access-setup)
5. [Serial Console](#5-serial-console)
6. [Telnet](#6-telnet)
7. [API Access Overview](#7-api-access-overview)
8. [Keyboard Shortcuts](#8-keyboard-shortcuts)
9. [Managing Multiple Routers](#9-managing-multiple-routers)
10. [Troubleshooting Connection Issues](#10-troubleshooting-connection-issues)
11. [Summary](#11-summary)

---

## 1. Winbox Installation และ Usage

### 1.1 ดาวน์โหลด Winbox

```
ดาวน์โหลดจาก:
https://mikrotik.com/download

มีให้เลือก:
├── Winbox64.exe  → Windows 64-bit (แนะนำ)
├── Winbox.exe    → Windows 32-bit
└── Winbox.jar    → Java version (Mac/Linux, ต้องมี JRE)
```

### 1.2 Windows Installation

```
Winbox ไม่ต้องติดตั้ง (portable executable)
1. ดาวน์โหลด Winbox64.exe
2. วาง file ที่ไหนก็ได้
3. Double-click เพื่อเปิด
4. ยืนยัน UAC (ถ้าถาม)
```

### 1.3 Linux / macOS

```bash
# Method 1: Wine (Linux)
sudo apt install wine
wine Winbox64.exe

# Method 2: Docker (Linux/Mac)
docker run --rm -it \
  -e DISPLAY=$DISPLAY \
  -v /tmp/.X11-unix:/tmp/.X11-unix \
  ghcr.io/mikrotik/winbox

# Method 3: Java version
java -jar Winbox.jar

# Method 4: native macOS app (unofficial)
# ดาวน์โหลดจาก https://github.com/yourname/winbox-mac
```

### 1.4 การ Login ครั้งแรก

```
เปิด Winbox:

1. ป้อน IP address หรือ MAC address ของ Router
   - IP: 192.168.88.1 (default)
   - MAC: ใช้ได้เมื่ออยู่ใน Layer 2 เดียวกัน

2. Username: admin
3. Password: (blank สำหรับ fresh install)

4. กด Connect
```

**Connection Window:**

```
┌────────────────────────────────────────┐
│             WinBox                     │
├────────────────────────────────────────┤
│  Connect To: [192.168.88.1     ▼]      │
│              [                  ]      │
│  Login:      [admin             ]      │
│  Password:   [                  ]      │
│              [✓] Keep Password         │
│                                        │
│  [Neighbors] [Managed]                 │
│                                        │
│  ┌──────────────────────────────────┐  │
│  │ MAC Address    IP Address   Ver  │  │
│  │ XX:XX:XX:XX:XX 192.168.88.1 7.15 │  │
│  └──────────────────────────────────┘  │
│                                        │
│         [Connect]  [Close]             │
└────────────────────────────────────────┘
```

### 1.5 Winbox via MAC Address

```
ประโยชน์ของการ connect ผ่าน MAC:
- ไม่ต้องรู้ IP address
- ใช้ได้แม้ Router ไม่มี IP
- ใช้ได้ระหว่าง Netinstall setup

ข้อจำกัด:
- ต้องอยู่ใน subnet เดียวกัน (Layer 2)
- ไม่สามารถใช้ผ่าน router/internet
```

---

## 2. Winbox Features ทั้งหมด

### 2.1 Main Menu

```
Winbox Main Menu (ซ้ายมือ):
├── Quick Set       - ตั้งค่าด่วน
├── CAPsMAN         - Wireless controller
├── Interfaces      - Network interfaces
├── Wireless        - WiFi settings
├── Bridge          - Bridge config
├── PPP             - PPP tunnels
├── Mesh            - Mesh networking
├── IP              - IPv4 settings
│   ├── Addresses   - IP addresses
│   ├── Routes      - Routing table
│   ├── DHCP Client - DHCP client
│   ├── DHCP Server - DHCP server
│   ├── DNS         - DNS settings
│   ├── Firewall    - Firewall rules
│   ├── ARP         - ARP table
│   ├── Neighbors   - CDP/LLDP
│   ├── Services    - IP services
│   └── ...
├── IPv6            - IPv6 settings
├── MPLS            - MPLS routing
├── Routing         - Routing protocols
├── System          - System settings
│   ├── Clock       - Time/timezone
│   ├── Identity    - Hostname
│   ├── License     - License info
│   ├── Packages    - Package mgmt
│   ├── Scheduler   - Task scheduler
│   ├── Scripts     - Script manager
│   └── ...
├── Queue           - Traffic queuing
├── Files           - File manager
├── Log             - System logs
├── Radius          - RADIUS client
├── Tools           - Diagnostic tools
│   ├── Ping        - ICMP ping
│   ├── Traceroute  - Path trace
│   ├── Bandwidth Test - Speed test
│   ├── Packet Sniffer - Capture
│   └── ...
└── New Terminal    - CLI terminal
```

### 2.2 Interface Management

```
Interfaces Window:
┌──────────────────────────────────────────────────────┐
│ Interface List                          + Add Filter │
├──────────────────────────────────────────────────────┤
│ # Name     Type   Actual MTU TX Rate   RX Rate  Status│
│ 0 ether1   Ether  1500       1.2Mbps   856kbps  R    │
│ 1 ether2   Ether  1500       24kbps    12kbps   R    │
│ 2 ether3   Ether  1500       -         -        D    │
│ 3 wlan1    Wlan   1500       2.1Mbps   1.5Mbps  R    │
└──────────────────────────────────────────────────────┘

คลิก interface เพื่อ:
- ดู statistics real-time
- เปลี่ยนชื่อ
- Enable/Disable
- ดู MAC address
```

### 2.3 IP Firewall

```
Firewall Filter Rules:
┌─────────────────────────────────────────────────────────────────┐
│ Filter Rules   NAT   Mangle   Raw   Service Ports   Connections │
├─────────────────────────────────────────────────────────────────┤
│ # Action  Chain   Protocol  Src Addr    Dst Addr    Comment     │
│ 0 accept  input   -         -           -           defconf     │
│ 1 accept  forward -         -           -           defconf     │
│ 2 drop    input   -         -           -           defconf     │
└─────────────────────────────────────────────────────────────────┘

Double-click rule เพื่อแก้ไข
ลาก-วาง rule เพื่อเปลี่ยน order
```

### 2.4 Traffic Monitor

```
Winbox → Interfaces → เลือก interface → Traffic

แสดง real-time graph:
┌─────────────────────────────────────────┐
│ TX/RX Traffic - ether1                  │
│                                         │
│ 10 Mbps┤                   ╭──╮        │
│  8 Mbps┤          ╭╮      ╭╯  ╰──     │
│  6 Mbps┤      ╭───╯╰──────╯           │
│  4 Mbps┤   ╭──╯                        │
│  2 Mbps┤───╯                           │
│  0     └─────────────────────────────  │
│         TX: 4.2 Mbps  RX: 1.8 Mbps    │
└─────────────────────────────────────────┘
```

### 2.5 Terminal (CLI ใน Winbox)

```
New Terminal เปิด CLI ใน Winbox window:
- รองรับ copy/paste
- ไม่ต้องการ SSH
- Multi-window support
- ทำงานเหมือน SSH
```

### 2.6 File Manager

```
Winbox → Files

เห็นไฟล์ทั้งหมดใน Router flash:
├── flash/           - Main storage
├── *.backup         - Backup files
├── *.rsc            - Script files
├── *.npk            - Package files
└── log/             - Log files

Actions:
- Upload file: ลาก-วางจาก PC
- Download file: Double-click → Download
- Delete file: เลือก → Delete
```

### 2.7 Log Viewer

```
Winbox → Log

แสดง real-time log:
┌──────────────────────────────────────────────────────┐
│ 09:00:01 system  router started                      │
│ 09:00:02 dhcp    192.168.1.10 assigned to XX:XX:..   │
│ 09:00:05 firewall dropped 1.2.3.4:12345 → ..:80      │
│ 09:00:10 wireless client XX:XX connected             │
└──────────────────────────────────────────────────────┘

Filter logs:
- by Topic: system, dhcp, firewall, etc.
- by Time range
- Search text
```

---

## 3. WebFig Overview

### 3.1 เข้าถึง WebFig

```
Method 1: HTTP
http://192.168.88.1

Method 2: HTTPS
https://192.168.88.1

Default port: 80 (HTTP), 443 (HTTPS)
```

### 3.2 WebFig vs Winbox

| Feature | Winbox | WebFig |
|---------|--------|--------|
| Installation | ต้องดาวน์โหลด | Browser (ไม่ต้อง install) |
| OS | Windows (หลัก) | ทุก OS |
| Speed | เร็ว | ช้ากว่า |
| Features | ครบมาก | ครบพอสมควร |
| Real-time Monitor | ดีมาก | ดี |
| MAC Login | ใช่ | ไม่ |
| Mobile | ไม่เหมาะ | เหมาะกว่า |
| HTTPS | ใช่ | ใช่ |

### 3.3 WebFig Features

```
WebFig Menu:
├── Quick Set
├── Interfaces
├── Bridge
├── PPP
├── IP
│   ├── Addresses
│   ├── Routes
│   ├── DHCP Server
│   ├── DHCP Client
│   ├── DNS
│   ├── Firewall
│   └── ...
├── Routing
├── System
├── Queue
├── Files
├── Log
└── Tools
```

### 3.4 ความแตกต่างหลัก WebFig

```
สิ่งที่ WebFig ทำได้แต่ Winbox ไม่ได้:
- เข้าถึงผ่าน mobile browser
- ไม่ต้องติดตั้งโปรแกรม
- เหมาะสำหรับ quick config จาก PC อื่น

สิ่งที่ Winbox ทำได้แต่ WebFig ไม่ได้:
- Connect ผ่าน MAC address
- ดู real-time traffic graph สวยกว่า
- จัดการ multiple routers ได้ดีกว่า
- Packet sniffer built-in
```

---

## 4. SSH Access Setup

### 4.1 เปิด SSH Service

```bash
# ตรวจสอบ SSH status
/ip service print

# Output:
# NAME    PORT  ADDRESS       CERT  ENABLED
# telnet  23                  none  yes
# ftp     21                  none  yes
# www     80                  none  yes
# ssh     22                  none  yes
# www-ssl 443                 none  yes
# api     8728                none  no
# winbox  8291                none  yes
# api-ssl 8729                none  no

# Enable SSH (ถ้ายังไม่เปิด)
/ip service enable ssh

# เปลี่ยน port
/ip service set ssh port=2222

# จำกัด access จาก IP เฉพาะ
/ip service set ssh address=192.168.1.0/24,10.0.0.0/8
```

### 4.2 SSH Key-based Authentication

```bash
# บน client PC - สร้าง SSH key pair
ssh-keygen -t rsa -b 4096 -f ~/.ssh/mikrotik_key -C "admin@myrouter"

# ดู public key
cat ~/.ssh/mikrotik_key.pub
# Output:
# ssh-rsa AAAAB3NzaC1yc2EAAAA... admin@myrouter

# Upload public key ไปยัง Router
# Method 1: ผ่าน Winbox Files → drag id_rsa.pub
# Method 2: SCP
scp ~/.ssh/mikrotik_key.pub admin@192.168.1.1:/

# บน Router - import public key
/user ssh-keys import public-key-file=mikrotik_key.pub user=admin

# ตรวจสอบ keys
/user ssh-keys print

# ทดสอบ SSH
ssh -i ~/.ssh/mikrotik_key admin@192.168.1.1
```

### 4.3 SSH Client Configuration

```bash
# ~/.ssh/config บน client PC
Host mikrotik-r1
    HostName 192.168.1.1
    Port 22
    User admin
    IdentityFile ~/.ssh/mikrotik_key
    StrictHostKeyChecking no
    ServerAliveInterval 30

# ใช้งาน
ssh mikrotik-r1
```

### 4.4 SSH Hardening

```bash
# ปิด password authentication (ถ้าใช้ key-based)
# RouterOS ไม่มี option นี้โดยตรง
# แต่สามารถใช้ firewall block ได้

# จำกัด SSH access
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    src-address=192.168.1.0/24 \
    action=accept \
    comment="Allow SSH from LAN"

/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    action=drop \
    comment="Drop SSH from WAN"
```

---

## 5. Serial Console

### 5.1 Hardware Requirements

```
Serial Console Connection:
Router ──── RJ45-to-DB9 adapter ──── USB-to-Serial ──── PC

หรือ:
Router ──── USB Micro/Type-A (ขึ้นกับรุ่น) ──── PC

Terminal Settings:
├── Baud rate: 115200
├── Data bits: 8
├── Parity: None
├── Stop bits: 1
└── Flow control: None
```

### 5.2 Terminal Programs

**Windows:**
```
PuTTY:
1. เปิด PuTTY
2. Connection type: Serial
3. Serial line: COM3 (ตรวจสอบใน Device Manager)
4. Speed: 115200
5. กด Open
```

**Linux:**
```bash
# minicom
sudo apt install minicom
sudo minicom -D /dev/ttyUSB0 -b 115200

# screen
sudo screen /dev/ttyUSB0 115200

# picocom
sudo picocom -b 115200 /dev/ttyUSB0
```

**macOS:**
```bash
# ดู port
ls /dev/tty.*
# /dev/tty.usbserial-XXX

# screen
screen /dev/tty.usbserial-XXX 115200

# minicom
brew install minicom
minicom -D /dev/tty.usbserial-XXX -b 115200
```

### 5.3 Serial Console Use Cases

```
เมื่อต้องใช้ Serial Console:
✓ Router boot ไม่ขึ้น
✓ ลืม IP address
✓ ลืม password (ทำ password recovery)
✓ Netinstall กำลังทำงาน
✓ Network interface ทุกอันปิด

Password Recovery via Serial:
1. ต่อ Serial console
2. Reboot router
3. กด any key เพื่อเข้า boot menu
4. เลือก "Reset all passwords"
5. Router reset password เป็น blank
```

---

## 6. Telnet

### 6.1 Telnet Overview

```
⚠️ Warning: Telnet ไม่ encrypted ไม่ควรใช้ใน production!
ใช้ SSH แทนเสมอ

Telnet ยังมีประโยชน์เมื่อ:
- Network ภายในที่ trusted มาก
- Lab environment
- Emergency access เมื่อ SSH ไม่ได้
- เรียน CLI
```

### 6.2 Disable Telnet

```bash
# ปิด Telnet (แนะนำ)
/ip service disable telnet

# ตรวจสอบ
/ip service print where name=telnet
# NAME    PORT  ADDRESS  CERT  ENABLED
# telnet  23             none  no
```

### 6.3 Telnet ใช้งาน (ถ้าจำเป็น)

```bash
# จาก PC (Windows/Linux/macOS)
telnet 192.168.1.1

# Login:
# MikroTik 7.15.3 (stable)
# [admin@Router] >
```

---

## 7. API Access Overview

### 7.1 RouterOS API

RouterOS มี API สำหรับ automation:

```
API Types:
├── API (port 8728)     - ไม่ encrypted (legacy)
└── API-SSL (port 8729) - TLS encrypted (แนะนำ)
```

### 7.2 Enable API

```bash
# Enable API
/ip service enable api
/ip service enable api-ssl

# Set certificate สำหรับ API-SSL
/certificate add name=api-cert common-name=router.local

/ip service set api-ssl certificate=api-cert

# จำกัด access
/ip service set api address=192.168.1.0/24
/ip service set api-ssl address=192.168.1.0/24
```

### 7.3 Python API Example

```python
#!/usr/bin/env python3
# ต้องติดตั้ง: pip install routeros-api

import routeros_api

# Connect
connection = routeros_api.RouterOsApiPool(
    '192.168.1.1',
    username='admin',
    password='password',
    port=8728,
    plaintext_login=True
)
api = connection.get_api()

# Get IP addresses
addresses = api.get_resource('/ip/address')
for addr in addresses.get():
    print(f"Interface: {addr['interface']}, Address: {addr['address']}")

# Get routing table
routes = api.get_resource('/ip/route')
for route in routes.get():
    print(f"Route: {route['dst-address']} via {route.get('gateway', 'local')}")

# Add IP address
ip_resource = api.get_resource('/ip/address')
ip_resource.add(address='10.0.0.1/24', interface='ether3')

# Disconnect
connection.disconnect()
```

### 7.4 REST API (RouterOS 7.1+)

```bash
# RouterOS 7.1+ มี REST API
# เปิด HTTPS service ก่อน

# ตัวอย่าง REST API calls
# Get all IP addresses
curl -k -u admin:password \
  https://192.168.1.1/rest/ip/address

# Get specific interface
curl -k -u admin:password \
  https://192.168.1.1/rest/interface/ether1

# Add IP address
curl -k -u admin:password \
  -X PUT \
  -H "Content-Type: application/json" \
  -d '{"address":"10.0.0.1/24","interface":"ether3"}' \
  https://192.168.1.1/rest/ip/address

# Delete IP address
curl -k -u admin:password \
  -X DELETE \
  https://192.168.1.1/rest/ip/address/*1
```

---

## 8. Keyboard Shortcuts

### 8.1 Winbox Shortcuts

| Shortcut | Action |
|----------|--------|
| `Ctrl+W` | ปิด window ปัจจุบัน |
| `Ctrl+T` | เปิด New Terminal |
| `Ctrl+R` | Refresh (reload data) |
| `F5` | Refresh |
| `Ctrl+F` | Find/Filter |
| `Ctrl+A` | Select All |
| `Delete` | ลบ item ที่เลือก |
| `Ctrl+D` | Disable item |
| `Ctrl+E` | Enable item |
| `Ctrl+C` | Copy |
| `Ctrl+V` | Paste |
| `Enter` | เปิด/แก้ไข item |
| `Ctrl+N` | New item |
| `Ins` | New item |

### 8.2 Terminal/CLI Shortcuts

| Shortcut | Action |
|----------|--------|
| `Tab` | Auto-complete |
| `?` | Help / show options |
| `↑` `↓` | Command history |
| `Ctrl+C` | ยกเลิก command |
| `Ctrl+Z` | Background process |
| `Ctrl+A` | ไปต้น line |
| `Ctrl+E` | ไปปลาย line |
| `Ctrl+K` | ลบตั้งแต่ cursor จนปลาย |
| `Ctrl+U` | ลบ entire line |
| `Ctrl+W` | ลบ word ก่อน cursor |
| `Ctrl+L` | Clear screen |
| `Ctrl+D` | Logout |
| `..` | ขึ้น menu หนึ่งระดับ |
| `/` | ไป root menu |
| `q` | Quit / back |

### 8.3 CLI Navigation Shortcuts

```bash
# Navigation
/ip address          # ไปที่ /ip address
..                   # ขึ้นไปหนึ่งระดับ (ไปที่ /ip)
/                    # กลับ root
/interface           # ไปที่ /interface

# Print ด้วย filter
/ip address print where interface=ether1
/ip route print where active=yes

# Find
/ip address find where address~"192.168"

# Export specific item
/ip address export where interface=LAN
```

---

## 9. Managing Multiple Routers

### 9.1 Winbox Session Manager

```
Winbox มี "Managed" tab ที่ด้านบน:
- บันทึก saved connections
- Group routers ตาม location/type
- Quick connect
- Color coding
```

### 9.2 Saved Sessions

```
บันทึก session:
1. Connect ไปที่ router
2. Tools → Settings → Sessions
3. กด Save

หรือ:
Connection window → กด Save ก่อน Connect
```

### 9.3 Multiple Winbox Windows

```
เปิด Winbox หลาย instance:
- เปิดไฟล์ Winbox64.exe หลายครั้ง
- แต่ละ instance = 1 connection

หรือใช้ New Window ใน Winbox:
- File → New Window
```

### 9.4 Winbox Organizer (Dude)

```
MikroTik The Dude คือ Network Management tool:
- Monitor หลาย routers พร้อมกัน
- Topology map
- SNMP monitoring
- Alert system
- ดาวน์โหลดจาก mikrotik.com/download (Dude package)
```

### 9.5 Script-based Management

```bash
# ใช้ SSH สำหรับ bulk management
# สร้าง script ที่รันบน multiple routers

#!/bin/bash
# manage_routers.sh

ROUTERS=("192.168.1.1" "192.168.1.2" "192.168.1.3")
COMMANDS="
/system identity print
/ip address print
"

for ROUTER in "${ROUTERS[@]}"; do
    echo "=== $ROUTER ==="
    ssh -o StrictHostKeyChecking=no admin@$ROUTER "$COMMANDS"
done
```

### 9.6 Ansible for MikroTik

```yaml
# ansible-playbook.yml
---
- name: Configure MikroTik Routers
  hosts: mikrotik
  gather_facts: no
  
  tasks:
    - name: Set hostname
      community.routeros.command:
        commands:
          - /system identity set name={{ inventory_hostname }}
    
    - name: Set DNS
      community.routeros.command:
        commands:
          - /ip dns set servers=8.8.8.8,8.8.4.4

# inventory.ini
[mikrotik]
router1 ansible_host=192.168.1.1
router2 ansible_host=192.168.1.2
router3 ansible_host=192.168.1.3

[mikrotik:vars]
ansible_user=admin
ansible_password=password
ansible_connection=network_cli
ansible_network_os=community.routeros.routeros
```

---

## 10. Troubleshooting Connection Issues

### 10.1 Cannot Connect via Winbox

```
ปัญหา: Winbox ไม่เห็น Router

Troubleshooting Steps:

1. ตรวจสอบ network connectivity:
   ping 192.168.88.1

2. ตรวจสอบ Winbox port (8291):
   telnet 192.168.88.1 8291
   หรือ
   nc -zv 192.168.88.1 8291

3. ตรวจสอบ firewall บน PC:
   Windows Defender → Allow Winbox

4. ตรวจสอบ Winbox service บน Router (ผ่าน SSH):
   /ip service print where name=winbox
   /ip service enable winbox

5. ตรวจสอบ IP address:
   /ip address print
```

### 10.2 MAC Discovery ไม่ทำงาน

```
ปัญหา: ไม่เห็น Router ใน Neighbors list

สาเหตุ:
- อยู่คนละ subnet (Layer 3)
- Firewall block UDP broadcast
- Windows Firewall block Winbox

แก้:
1. ต้องต่อสายตรงหรืออยู่ใน Layer 2 เดียวกัน
2. ปิด Windows Firewall ชั่วคราว
3. ลองใช้ IP address แทน MAC
```

### 10.3 SSH Connection Refused

```bash
# ตรวจสอบ SSH service
/ip service print where name=ssh

# Enable SSH
/ip service enable ssh

# ตรวจสอบ address restriction
/ip service print detail where name=ssh
# ถ้า address field ไม่ตรง จะ refuse

# แก้ไข address restriction
/ip service set ssh address=0.0.0.0/0

# ตรวจสอบ firewall
/ip firewall filter print where dst-port=22
```

### 10.4 Cannot Login

```
ปัญหา: ใส่ username/password ถูกแต่ login ไม่ได้

1. ตรวจสอบ user accounts:
   /user print

2. Reset password (ถ้าลืม):
   - ต้องใช้ Serial console หรือ Netinstall
   - Serial: Reboot → Boot menu → Reset passwords

3. ตรวจสอบ allowed address สำหรับ user:
   /user print detail
```

### 10.5 Winbox Crashes / Hangs

```
แก้ไข Winbox crashes:

1. อัพเดท Winbox:
   https://mikrotik.com/download

2. ล้าง cache:
   ลบ Winbox.cfg ใน folder เดียวกับ Winbox.exe

3. ใช้ version เก่า:
   https://mikrotik.com/download ให้เลือก version

4. ลอง WebFig แทน:
   http://192.168.88.1

5. ลอง SSH:
   ssh admin@192.168.88.1
```

### 10.6 Diagnostic Commands

```bash
# ตรวจสอบ services ทั้งหมด
/ip service print

# ตรวจสอบ firewall ที่อาจ block access
/ip firewall filter print where chain=input

# ตรวจสอบ interface
/interface print

# ตรวจสอบ IP address
/ip address print

# ดู log เพื่อหา errors
/log print where topics~"error"

# ดู active connections
/ip firewall connection print count-only

# Test connectivity
/ping 192.168.1.100 count=4
```

---

## 11. Summary

### สิ่งที่เรียนรู้ใน Part 4:

| Tool | ใช้เมื่อ |
|------|--------|
| **Winbox** | Primary management tool, Windows-based, full features |
| **WebFig** | Browser-based, ใช้เมื่อไม่มี Winbox, mobile access |
| **SSH** | Secure remote management, scripting, automation |
| **Serial Console** | Emergency access, password recovery, Netinstall |
| **Telnet** | Lab only (ไม่แนะนำ production) |
| **API** | Automation, custom applications |
| **REST API** | Modern automation (RouterOS 7.1+) |

### Best Practices

```
✅ ใน Production:
- ใช้ Winbox หรือ SSH สำหรับ management
- ปิด Telnet เสมอ
- จำกัด management access จาก trusted IPs
- ใช้ SSH key-based authentication
- อัพเดท Winbox เป็น version ล่าสุด

✅ ใน Lab:
- Winbox ดีสุดสำหรับ visual learning
- SSH ดีสุดสำหรับ script practice
- Serial console มีประโยชน์มากสำหรับ recovery
```

### Quick Reference

```bash
# Services management
/ip service print
/ip service enable ssh
/ip service disable telnet
/ip service set ssh port=22 address=192.168.1.0/24

# SSH keys
/user ssh-keys import public-key-file=id_rsa.pub user=admin
/user ssh-keys print

# User management
/user print
/user set admin password=NewPassword
/user add name=netadmin password=Pass group=full
```

---

**[⬅ Previous: RouterOS Installation](part-003-routeros-installation.md)** | **[Next: CLI Basics ➡](part-005-cli-basics.md)**
