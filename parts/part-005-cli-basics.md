# Part 5: CLI Basics

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner  
> **เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

1. [Terminal Emulator Setup](#1-terminal-emulator-setup)
2. [RouterOS CLI Structure](#2-routeros-cli-structure)
3. [Menu Navigation](#3-menu-navigation)
4. [Command Syntax](#4-command-syntax)
5. [Filtering และ Searching](#5-filtering-และ-searching)
6. [Tab Completion](#6-tab-completion)
7. [Command History](#7-command-history)
8. [Aliases](#8-aliases)
9. [Output Formatting](#9-output-formatting)
10. [Export และ Import Configuration](#10-export-และ-import-configuration)
11. [Quick Reference Card](#11-quick-reference-card)
12. [Practice Exercises](#12-practice-exercises)

---

## 1. Terminal Emulator Setup

### 1.1 Windows - PuTTY

```
ดาวน์โหลด PuTTY:
https://www.putty.org/

การตั้งค่า SSH:
├── Host Name: 192.168.1.1
├── Port: 22
├── Connection type: SSH
├── Saved Sessions: "My MikroTik"
└── กด Save

Settings แนะนำ:
Window → Appearance:
├── Font: Courier New 12pt (หรือ Consolas)
└── Font quality: ClearType

Window → Translation:
└── Remote character set: UTF-8

Connection → SSH:
├── Enable SSH compression: ✓
└── Preferred SSH protocol version: 2

Terminal:
└── Keyboard → The Backspace key: Control-H
```

### 1.2 Windows - Windows Terminal

```powershell
# ติดตั้ง Windows Terminal จาก Microsoft Store
# หรือ
winget install Microsoft.WindowsTerminal

# SSH ไปยัง MikroTik
ssh admin@192.168.1.1

# ตั้ง profile ใน Windows Terminal settings.json:
{
    "name": "MikroTik",
    "commandline": "ssh admin@192.168.1.1",
    "icon": "🔧",
    "colorScheme": "One Half Dark"
}
```

### 1.3 macOS - Terminal

```bash
# Terminal.app มาพร้อม macOS
# หรือใช้ iTerm2 (แนะนำ)
brew install --cask iterm2

# SSH
ssh admin@192.168.1.1

# สร้าง SSH config
cat >> ~/.ssh/config << 'EOF'
Host mikrotik
    HostName 192.168.1.1
    User admin
    Port 22
    IdentityFile ~/.ssh/mikrotik_key
    ServerAliveInterval 30
    ServerAliveCountMax 3
EOF

# ใช้งาน
ssh mikrotik
```

### 1.4 Linux - Terminal Emulators

```bash
# GNOME Terminal (Ubuntu default)
gnome-terminal -- ssh admin@192.168.1.1

# Terminator (multi-pane)
sudo apt install terminator

# Tmux สำหรับ session management
sudo apt install tmux
tmux new-session -d -s mikrotik
tmux attach -t mikrotik

# SSH ไปยัง MikroTik
ssh admin@192.168.1.1 -i ~/.ssh/mikrotik_key
```

### 1.5 Font Recommendations

```
แนะนำ Monospace fonts:
├── Consolas (Windows built-in)
├── Cascadia Code (Windows Terminal default)
├── JetBrains Mono (Free)
├── Fira Code (Free, ligatures)
├── Hack (Free)
└── Source Code Pro (Adobe, Free)

เหตุผล: RouterOS CLI ใช้ | ├ └ characters 
ต้องการ monospace font ที่ render ถูกต้อง
```

---

## 2. RouterOS CLI Structure

### 2.1 Prompt Structure

```
[admin@Router] >
  │      │      │
  │      │      └── ตำแหน่ง menu ปัจจุบัน (> = root)
  │      └────────── Hostname ของ router
  └───────────────── Username ที่ login

ตัวอย่างเมื่อเข้า /ip:
[admin@Router] /ip>

ตัวอย่างเมื่อเข้า /ip address:
[admin@Router] /ip/address>
```

### 2.2 Command Structure

```
RouterOS Command Format:
/<menu-path> <verb> [<parameters>]

Examples:
/ip address add address=192.168.1.1/24 interface=ether1
/interface print
/ip route add dst-address=0.0.0.0/0 gateway=10.0.0.1
/system reboot
```

### 2.3 Verbs (Actions)

```
Primary Verbs:
├── add      - เพิ่ม item ใหม่
├── set      - แก้ไข item ที่มีอยู่
├── remove   - ลบ item
├── print    - แสดงข้อมูล
├── enable   - เปิดใช้งาน item
├── disable  - ปิดใช้งาน item
├── move     - ย้าย item (เปลี่ยน order)
├── comment  - เพิ่ม comment
├── export   - export config
└── monitor  - monitor real-time

Special Verbs:
├── reset    - reset item
├── check    - ตรวจสอบ validity
├── find     - หา item ที่ match condition
├── getall   - ดึง item ทั้งหมด
└── run      - รัน script/command
```

---

## 3. Menu Navigation

### 3.1 การ Navigate

```bash
# ไปที่ menu
[admin@Router] > /ip address
[admin@Router] /ip/address>

# ขึ้นไปหนึ่งระดับ
[admin@Router] /ip/address> ..
[admin@Router] /ip>

# ขึ้นสองระดับ
[admin@Router] /ip/address> ../..
[admin@Router] >

# กลับ root
[admin@Router] /ip/address> /
[admin@Router] >

# ใช้ absolute path จากทุกที่
[admin@Router] /ip/address> /system identity print
[admin@Router] /ip/address>   ← ยังอยู่ที่เดิม
```

### 3.2 Menu Tree

```bash
# Root menus สำคัญ
/caps-man          # CAPsMAN wireless controller (v6)
/certificate       # PKI certificates
/container         # Docker-like containers (v7)
/disk              # External storage
/file              # File management
/interface         # Network interfaces
  /bridge          # Bridge interfaces
  /ethernet        # Ethernet interfaces
  /l2tp-client     # L2TP client tunnels
  /l2tp-server     # L2TP server
  /list            # Interface lists
  /ovpn-client     # OpenVPN client
  /ovpn-server     # OpenVPN server
  /pppoe-client    # PPPoE client
  /sstp-client     # SSTP client
  /vlan            # VLAN interfaces
  /vxlan           # VXLAN (v7)
  /wireguard       # WireGuard (v7)
  /wireless        # WiFi interfaces
/ip                # IPv4 settings
  /address         # IP addresses
  /arp             # ARP table
  /cloud           # MikroTik Cloud
  /dhcp-client     # DHCP client
  /dhcp-relay      # DHCP relay
  /dhcp-server     # DHCP server
  /dns             # DNS settings
  /firewall        # Firewall
    /address-list  # Address lists
    /connection    # Connection tracking
    /filter        # Filter rules
    /layer7-protocol # L7 patterns
    /mangle        # Mangle rules
    /nat           # NAT rules
    /raw           # Raw rules
    /service-port  # Service ports
  /hotspot         # Hotspot system
  /ipsec           # IPsec
  /neighbor        # Neighbor discovery
  /pool            # IP pools
  /proxy           # Web proxy
  /route           # Routing table
  /service         # IP services
  /smb             # SMB/CIFS
  /snmp            # SNMP
  /ssh             # SSH (keys)
  /tftp            # TFTP server
  /traffic-flow    # NetFlow
  /upnp            # UPnP
  /vrf             # VRF (v7)
/ipv6              # IPv6 settings
/log               # System log
/mpls              # MPLS
/ppp               # PPP protocols
/queue             # Traffic queuing
/radius            # RADIUS client
/routing           # Routing protocols
  /bgp             # BGP (v7 redesigned)
  /filter          # Route filters
  /ospf            # OSPF (v7 redesigned)
  /rip             # RIP
  /table           # Routing tables
/snmp              # SNMP (root level)
/system            # System settings
  /backup          # Backup/restore
  /clock           # Time/timezone
  /console         # Console settings
  /default-configuration # Factory defaults
  /health          # Hardware health
  /history         # Command history
  /identity        # Hostname
  /leds            # LED config
  /license         # License info
  /logging         # Log configuration
  /note            # Notes
  /ntp             # NTP client/server
  /package         # Package management
  /password        # Password change
  /resource        # System resources
  /routerboard     # RouterBOARD info
  /scheduler       # Task scheduler
  /script          # Script manager
  /upgrade         # System upgrade
  /watchdog        # Watchdog timer
/tool              # Diagnostic tools
  /bandwidth-test  # Speed test
  /e-mail          # Email notifications
  /fetch           # HTTP client
  /graphing        # Graphing
  /mac-server      # MAC telnet/winbox
  /netwatch        # Network watch
  /ping            # ICMP ping
  /profile         # CPU profiling
  /romon           # RoMON
  /sms             # SMS (modem)
  /sniffer         # Packet capture
  /torch           # Real-time monitor
  /traceroute      # Path trace
/user              # User management
  /group           # User groups
  /ssh-keys        # SSH public keys
```

---

## 4. Command Syntax

### 4.1 add Command

```bash
# Syntax: add [parameters]
# สร้าง item ใหม่

# เพิ่ม IP address
/ip address add \
    address=192.168.1.1/24 \
    interface=ether2 \
    comment="LAN Interface"

# เพิ่ม static route
/ip route add \
    dst-address=10.0.0.0/8 \
    gateway=192.168.1.254 \
    distance=10 \
    comment="Corporate network"

# เพิ่ม firewall rule
/ip firewall filter add \
    chain=input \
    protocol=tcp \
    dst-port=22 \
    src-address=192.168.1.0/24 \
    action=accept \
    comment="Allow SSH from LAN" \
    place-before=0

# เพิ่ม DHCP static lease
/ip dhcp-server lease add \
    address=192.168.1.100 \
    mac-address=AA:BB:CC:DD:EE:FF \
    comment="Printer"
```

### 4.2 set Command

```bash
# Syntax: set [number|name] [parameters]
# แก้ไข item ที่มีอยู่

# แก้ไขด้วย index number
/ip address set 0 address=192.168.2.1/24

# แก้ไขด้วย comment/name
/ip address set [find comment="LAN Interface"] address=10.0.0.1/24

# แก้ไข system identity
/system identity set name=MyRouter

# แก้ไข interface
/interface set ether1 name=WAN mtu=1500

# แก้ไข multiple items
/ip address set 0,1,2 disabled=yes
```

### 4.3 remove Command

```bash
# Syntax: remove [number|name|find]
# ลบ item

# ลบด้วย index
/ip address remove 0

# ลบด้วย condition
/ip address remove [find interface=ether3]

# ลบหลาย items
/ip address remove 0,1,2

# ลบทั้งหมด (ระวัง!)
/ip address remove [find]
```

### 4.4 print Command

```bash
# Syntax: print [flags] [options]
# แสดงข้อมูล

# Print ทั้งหมด
/ip address print

# Print พร้อม details
/ip address print detail

# Print format table
/ip address print terse

# Print พร้อม count
/ip firewall filter print count-only

# Print แบบ brief
/ip address print brief

# Print โดย filter
/ip address print where interface=ether1

# Print ด้วย columns เฉพาะ
/ip address print show-ids

# Monitor (real-time update)
/interface print interval=1
```

### 4.5 enable / disable Commands

```bash
# Enable item
/ip address enable 0
/ip address enable [find interface=ether3]
/interface enable ether3

# Disable item
/ip address disable 0
/ip address disable [find address~"192.168.2"]
/interface disable ether3

# Toggle หลาย items
/ip firewall filter disable 0,1,2
/ip firewall filter enable 0,1,2
```

### 4.6 move Command

```bash
# Syntax: move [source] [destination]
# เปลี่ยน order ของ rules (สำคัญสำหรับ firewall)

# ย้าย rule 5 ไปก่อน rule 0
/ip firewall filter move 5 destination=0

# ย้าย rule ล่าสุดไปเป็น rule แรก
/ip firewall filter move [find comment="Important Rule"] destination=0
```

---

## 5. Filtering และ Searching

### 5.1 where Clause

```bash
# filter ด้วย exact match
/ip address print where interface=ether1

# filter ด้วย contains (~)
/ip address print where address~"192.168"

# filter ด้วย not (!)
/ip address print where !disabled

# filter ด้วย greater/less than
/ip route print where distance>10

# filter ด้วย หลาย conditions
/ip firewall filter print where chain=input && action=drop

# filter ด้วย OR
/ip firewall filter print where chain=input || chain=forward
```

### 5.2 find Command

```bash
# หา index ของ item
/ip address find where interface=ether1
# Output: 0 2 (index numbers)

# ใช้ใน set/remove
/ip address set [find interface=ether1] disabled=yes
/ip address remove [find disabled=yes]

# หา firewall rule
/ip firewall filter find where comment~"allow"
```

### 5.3 ตัวอย่าง Advanced Filtering

```bash
# หา interfaces ที่ running
/interface print where running=yes

# หา routes ที่ active
/ip route print where active=yes

# หา DHCP leases ที่ active
/ip dhcp-server lease print where status=bound

# หา firewall rules ที่ disable
/ip firewall filter print where disabled=yes

# หา users ที่ login อยู่
/user active print

# หา interfaces ที่มี traffic
/interface print where tx-byte>0

# Filter แบบ range
/log print where time>12:00:00 && time<13:00:00
```

---

## 6. Tab Completion

### 6.1 การใช้ Tab

```
กด Tab เพื่อ auto-complete:

[admin@Router] > /ip addr<Tab>
→ /ip address

[admin@Router] > /ip address ad<Tab>
→ /ip address add

[admin@Router] /ip/address> add inter<Tab>
→ add interface=

[admin@Router] /ip/address> add interface=<Tab><Tab>
→ แสดง available interfaces:
   ether1    ether2    ether3    bridge1   wlan1
```

### 6.2 ? สำหรับ Help

```bash
# ดู help สำหรับ command ปัจจุบัน
[admin@Router] > ?
# แสดงทุก top-level commands

[admin@Router] /ip/address> add ?
# แสดง parameters ทั้งหมดสำหรับ add

[admin@Router] /ip/address> add interface=?
# แสดง available interfaces

[admin@Router] /ip/address> add action=?
# แสดง valid actions
```

### 6.3 Double Tab

```bash
# กด Tab สองครั้งเพื่อดูทางเลือกทั้งหมด
[admin@Router] > /ip <Tab><Tab>
Output:
  address        arp             cloud
  dhcp-client    dhcp-relay      dhcp-server
  dns            firewall        hotspot
  ipsec          neighbor        pool
  proxy          route           service
  settings       smb             snmp
  ssh            tftp            traffic-flow
  upnp           vrf
```

---

## 7. Command History

### 7.1 ดู History

```bash
# ใช้ Up/Down arrow keys เพื่อเลื่อน history

# ดู command history
/system history print

# Output:
# ACTION   BY      POLICY
# add      admin   write
# set      admin   write
# remove   admin   write

# Undo ล่าสุด (RouterOS ไม่รองรับ undo โดยตรง)
# ต้องทำ inverse command manually
```

### 7.2 Command History ใน Terminal

```bash
# กด ↑ ↓ เพื่อเลื่อน history
# Ctrl+R เพื่อ reverse search (ไม่ supported ใน RouterOS)

# ดู history ใน terminal buffer
# scroll up ในใน terminal emulator
```

### 7.3 Script Logging

```bash
# RouterOS log ทุก configuration change
/log print where topics~"system"

# ดู last changes
/system history print

# Output แสดง:
# - ใครทำอะไร
# - เมื่อไหร่
# - Action ที่ทำ
```

---

## 8. Aliases

### 8.1 RouterOS Aliases

RouterOS ไม่มี bash-style aliases โดยตรง แต่ใช้ scripts แทน:

```bash
# สร้าง script ที่ทำงานเหมือน alias
/system script add name=show-ip source={
    /ip address print
    /ip route print
}

# รัน script
/system script run show-ip

# หรือสร้าง shortcut ด้วย Scheduler
/system scheduler add name=run-script \
    on-event=/system/script/run show-ip \
    interval=0  # 0 = run once
```

### 8.2 Function-like Scripts

```bash
# สร้าง reusable script
/system script add name=backup-config source={
    :local dateStr [/system clock get date]
    :local filename ("backup-" . $dateStr)
    /system backup save name=$filename
    /export file=$filename
    :log info ("Config backed up: " . $filename)
}

# รัน
/system script run backup-config
```

### 8.3 Environment Variables

```bash
# ตั้งค่า global variable
:global myDNS "8.8.8.8"
:put $myDNS
# Output: 8.8.8.8

# ใช้ใน commands
/ip dns set servers=$myDNS

# ตั้ง local variable ใน script
:local interface "ether1"
:local ipAddr "192.168.1.1"
/ip address add address=$ipAddr interface=$interface
```

---

## 9. Output Formatting

### 9.1 Print Formats

```bash
# Standard print
/ip address print

# Detail format (verbose)
/ip address print detail

# Terse format (compact)
/ip address print terse
# Output (single line per item):
# /ip address add address=192.168.1.1/24 interface=ether1 disabled=no

# Brief format
/ip address print brief

# Print ด้วย specific columns
/ip address print column=address,interface

# Print จำนวนนับ
/ip address print count-only
# Output: 3
```

### 9.2 Output Filtering

```bash
# แสดงเฉพาะ fields ที่สนใจ
/ip address print column=address,interface,disabled

# Sort output
/ip route print order-by=dst-address

# Output หลาย levels
/ip firewall filter print verbose
```

### 9.3 Redirect Output

```bash
# RouterOS ไม่มี pipe (|) เหมือน Linux
# แต่ใช้ find และ where แทน

# เปรียบเทียบ:
# Linux:   ip route | grep default
# RouterOS: /ip route print where dst-address="0.0.0.0/0"

# Count matching items:
# Linux:   iptables -L | grep DROP | wc -l
# RouterOS: /ip firewall filter print count-only where action=drop
```

### 9.4 Terminal Colors และ Special Characters

```bash
# RouterOS ใช้ ANSI colors ใน terminal
# แสดง:
# - R = Running (สีเขียว)
# - D = Dynamic (สีน้ำเงิน)
# - X = Disabled (สีแดง/หรี่)

# ตัวอย่าง interface print:
/interface print
# Output:
# Flags: D - dynamic, X - disabled, R - running, S - slave
#  #     NAME       TYPE     MTU  ...
#  0  R  ether1     ether    1500 ...   ← R = Running
#  1  X  ether2     ether    1500 ...   ← X = Disabled  
#  2 RS  bridge1    bridge   1500 ...   ← R+S = Running+Slave
```

---

## 10. Export และ Import Configuration

### 10.1 Export Config

```bash
# Export ทั้ง config (ไปที่ terminal)
/export

# Export เฉพาะ section
/ip address export
/ip firewall filter export
/ip route export

# Export ไปที่ file
/export file=my-config

# Export พร้อม sensitive info (passwords)
/export verbose file=full-config

# Export แบบ compact (ไม่มี defaults)
/export compact file=compact-config

# ดู exported files
/file print
```

### 10.2 Import Config

```bash
# Import จาก file
/import file-name=my-config.rsc

# Import พร้อม verbose output
/import verbose file-name=my-config.rsc

# Import แบบ dry-run (ดูว่าจะทำอะไร)
# RouterOS ไม่รองรับ dry-run โดยตรง
# ใช้วิธี review file ก่อน import
```

### 10.3 Backup และ Restore

```bash
# สร้าง binary backup
/system backup save name=full-backup

# สร้าง backup พร้อม password
/system backup save name=secure-backup password=BackupPass123

# ดู backup files
/file print where name~"backup"

# Restore จาก backup
/system backup load name=full-backup

# ⚠️ Warning: restore จะ reboot router อัตโนมัติ

# สร้าง backup + export (แนะนำทั้งคู่)
:local dateStr [/system clock get date]
/system backup save name=("backup-" . $dateStr)
/export file=("export-" . $dateStr)
```

### 10.4 Remote Backup

```bash
# ส่ง backup ไปยัง FTP server
/tool fetch address=192.168.1.100 \
    src-path=backup.backup \
    user=ftpuser \
    password=ftppass \
    mode=ftp \
    port=21 \
    upload=yes

# ส่งผ่าน Email
/tool e-mail send \
    to=admin@example.com \
    subject="Router Backup" \
    body="Automated backup" \
    file=backup.backup

# Automated backup script
/system script add name=daily-backup source={
    :local date [/system clock get date]
    :local fname ("backup-" . $date)
    /system backup save name=$fname
    /export file=$fname
    /tool fetch address=10.0.0.1 \
        src-path=($fname . ".backup") \
        user=backup \
        password=backuppass \
        mode=ftp \
        upload=yes
}

/system scheduler add \
    name=daily-backup \
    on-event=/system/script/run\ daily-backup \
    start-time=02:00:00 \
    interval=1d
```

---

## 11. Quick Reference Card

### Core Commands Table

| Command | Description | Example |
|---------|-------------|---------|
| `/ip address print` | ดู IP addresses | `/ip address print` |
| `/ip address add` | เพิ่ม IP address | `/ip address add address=x.x.x.x/y interface=etherN` |
| `/ip address set` | แก้ IP address | `/ip address set 0 address=x.x.x.x/y` |
| `/ip address remove` | ลบ IP address | `/ip address remove 0` |
| `/ip route print` | ดู routing table | `/ip route print` |
| `/ip route add` | เพิ่ม static route | `/ip route add dst-address=0.0.0.0/0 gateway=x.x.x.x` |
| `/interface print` | ดู interfaces | `/interface print` |
| `/interface enable` | เปิด interface | `/interface enable ether1` |
| `/interface disable` | ปิด interface | `/interface disable ether1` |
| `/ip firewall filter print` | ดู firewall rules | `/ip firewall filter print` |
| `/ip firewall nat print` | ดู NAT rules | `/ip firewall nat print` |
| `/ip dns print` | ดู DNS config | `/ip dns print` |
| `/ip dhcp-server print` | ดู DHCP servers | `/ip dhcp-server print` |
| `/ip dhcp-client print` | ดู DHCP clients | `/ip dhcp-client print` |
| `/system identity print` | ดูชื่อ router | `/system identity print` |
| `/system resource print` | ดู system info | `/system resource print` |
| `/system clock print` | ดูเวลา | `/system clock print` |
| `/system reboot` | Reboot router | `/system reboot` |
| `/export` | Export config | `/export file=backup` |
| `/import` | Import config | `/import file-name=backup.rsc` |
| `/ping` | Ping test | `/ping 8.8.8.8 count=4` |
| `/tool traceroute` | Traceroute | `/tool traceroute 8.8.8.8` |
| `/log print` | ดู logs | `/log print` |

### Navigation Quick Reference

```bash
/                    # ไป root
..                   # ขึ้น 1 level
../..                # ขึ้น 2 levels
/ip address          # ไปที่ /ip address
Tab                  # Auto-complete
?                    # Help
Ctrl+C               # Cancel
↑ ↓                  # Command history
```

### Useful Filters

```bash
# Active items เท่านั้น
print where !disabled

# Running interfaces เท่านั้น
/interface print where running=yes

# Active routes เท่านั้น
/ip route print where active=yes

# Specific interface
/ip address print where interface=ether1

# Contains string
/ip address print where address~"192.168"
```

---

## 12. Practice Exercises

### Exercise 1: Basic Navigation

**โจทย์:** Navigate ผ่าน menu tree และ print ข้อมูล

```bash
# ทำตามขั้นตอน:
1. Login ไปยัง Router
2. ไปที่ /ip address
3. Print addresses
4. กลับไป root
5. ไปที่ /interface
6. Print interfaces
7. ไปที่ /system
8. Print resource info
```

**เฉลย:**
```bash
[admin@Router] > /ip address
[admin@Router] /ip/address> print
[admin@Router] /ip/address> /
[admin@Router] > /interface
[admin@Router] /interface> print
[admin@Router] /interface> /system
[admin@Router] /system> resource print
```

---

### Exercise 2: IP Address Management

**โจทย์:** เพิ่ม, ดู, และลบ IP addresses

```bash
# เพิ่ม IP addresses เหล่านี้:
# ether1: 10.0.0.1/30
# ether2: 192.168.1.1/24
# ether3: 172.16.0.1/16

# ดู IP addresses ที่เพิ่ม
# ลบ IP ของ ether3
# ตรวจสอบว่าลบแล้ว
```

**เฉลย:**
```bash
/ip address add address=10.0.0.1/30 interface=ether1
/ip address add address=192.168.1.1/24 interface=ether2
/ip address add address=172.16.0.1/16 interface=ether3
/ip address print
/ip address remove [find interface=ether3]
/ip address print
```

---

### Exercise 3: Interface Renaming

**โจทย์:** เปลี่ยนชื่อ interfaces ให้ชัดเจน

```bash
# เปลี่ยนชื่อ:
# ether1 → WAN
# ether2 → LAN
# ether3 → DMZ

# ตรวจสอบ
```

**เฉลย:**
```bash
/interface set ether1 name=WAN
/interface set ether2 name=LAN
/interface set ether3 name=DMZ
/interface print
```

---

### Exercise 4: Filtering Output

**โจทย์:** ใช้ filter เพื่อดูข้อมูลเฉพาะ

```bash
# ดู:
# 1. Interfaces ที่ running เท่านั้น
# 2. IP addresses ของ LAN interface
# 3. Active routes เท่านั้น
# 4. Firewall rules ที่ chain=input เท่านั้น
```

**เฉลย:**
```bash
/interface print where running=yes
/ip address print where interface=LAN
/ip route print where active=yes
/ip firewall filter print where chain=input
```

---

### Exercise 5: Export และ Import

**โจทย์:** Export config และ Import กลับ

```bash
# 1. Export config ไปที่ file ชื่อ "test-export"
# 2. ดู file ที่ export
# 3. ดูเนื้อหา file (อ่านผ่าน terminal)
# 4. เพิ่ม comment ใน CLI
# 5. Export อีกครั้งพร้อม timestamp
```

**เฉลย:**
```bash
/export file=test-export
/file print
/file print detail where name~"test-export"

# อ่าน file content
:local content [/file get [find name="test-export.rsc"] contents]
:put $content

# Export พร้อม date
:local date [/system clock get date]
/export file=("config-" . $date)
```

---

### Exercise 6: Script Basics

**โจทย์:** สร้าง script แสดงสถานะ network

```bash
# สร้าง script ชื่อ "show-status" ที่แสดง:
# 1. Hostname
# 2. RouterOS version
# 3. Uptime
# 4. IP addresses
# 5. Default routes
```

**เฉลย:**
```bash
/system script add name=show-status source={
    :put "=== Router Status ==="
    :put ("Hostname: " . [/system identity get name])
    :put ("Version:  " . [/system resource get version])
    :put ("Uptime:   " . [/system resource get uptime])
    :put ""
    :put "--- IP Addresses ---"
    /ip address print
    :put ""
    :put "--- Default Routes ---"
    /ip route print where dst-address="0.0.0.0/0"
}

/system script run show-status
```

---

### Exercise 7: Backup Script

**โจทย์:** สร้าง automated backup

```bash
# สร้าง script ที่:
# 1. สร้าง backup ด้วยชื่อที่มี date
# 2. Export config ด้วยชื่อที่มี date
# 3. Log ว่า backup สำเร็จ
```

**เฉลย:**
```bash
/system script add name=auto-backup source={
    :local date [/system clock get date]
    :local backupName ("backup-" . $date)
    :local exportName ("export-" . $date)
    
    /system backup save name=$backupName
    /export file=$exportName
    
    :log info ("Backup completed: " . $backupName)
    :log info ("Export completed: " . $exportName)
    :put ("Backup done: " . $backupName)
}

/system script run auto-backup
```

---

### Exercise 8: Find และ Modify

**โจทย์:** หาและแก้ไข items ด้วย find

```bash
# 1. Find interfaces ที่ disabled
# 2. Enable ทุก disabled interfaces
# 3. Find IP addresses ที่มี "192.168" ใน address
# 4. Disable IP addresses เหล่านั้น
```

**เฉลย:**
```bash
/interface print where disabled=yes
/interface enable [find disabled=yes]

/ip address print where address~"192.168"
/ip address disable [find address~"192.168"]
```

---

### Exercise 9: Scheduler

**โจทย์:** ตั้ง scheduled tasks

```bash
# ตั้ง Scheduler:
# 1. Backup ทุกวัน เวลา 02:00 น.
# 2. Reboot ทุกสัปดาห์ วันอาทิตย์ เวลา 03:00 น.
```

**เฉลย:**
```bash
# Daily backup
/system scheduler add \
    name=daily-backup \
    start-date=jan/01/2024 \
    start-time=02:00:00 \
    interval=1d \
    on-event="/system backup save name=auto-daily" \
    comment="Daily config backup"

# Weekly reboot
/system scheduler add \
    name=weekly-reboot \
    start-date=jan/07/2024 \
    start-time=03:00:00 \
    interval=7d \
    on-event=/system/reboot \
    comment="Weekly maintenance reboot"

# ตรวจสอบ
/system scheduler print
```

---

### Exercise 10: Complete Config Audit

**โจทย์:** สร้าง script ตรวจสอบ security

```bash
# สร้าง script ที่ตรวจสอบ:
# 1. Admin password ถูกตั้งหรือไม่
# 2. Telnet enabled หรือไม่
# 3. SSH enabled หรือไม่
# 4. Winbox accessible จาก WAN หรือไม่
# 5. มี NTP configured หรือไม่
```

**เฉลย:**
```bash
/system script add name=security-audit source={
    :put "=== Security Audit ==="
    
    # Check admin password
    :local adminPass [/user get [find name=admin] password]
    :if ($adminPass = "") do={
        :put "⚠️  WARNING: Admin password is NOT set!"
    } else={
        :put "✅ Admin password is set"
    }
    
    # Check Telnet
    :local telnetEnabled [/ip service get [find name=telnet] disabled]
    :if (!$telnetEnabled) do={
        :put "⚠️  WARNING: Telnet is ENABLED (insecure!)"
    } else={
        :put "✅ Telnet is disabled"
    }
    
    # Check SSH
    :local sshEnabled [/ip service get [find name=ssh] disabled]
    :if (!$sshEnabled) do={
        :put "✅ SSH is enabled"
    } else={
        :put "⚠️  INFO: SSH is disabled"
    }
    
    # Check NTP
    :local ntpServers [/system ntp client get servers]
    :if ($ntpServers = "") do={
        :put "⚠️  WARNING: NTP is not configured"
    } else={
        :put ("✅ NTP configured: " . $ntpServers)
    }
    
    :put "=== Audit Complete ==="
}

/system script run security-audit
```

---

## Summary

### สิ่งที่เรียนรู้ใน Part 5:

1. **Terminal Setup** ใช้ PuTTY (Windows) หรือ native terminal (Mac/Linux) ต่อ SSH ไปยัง RouterOS
2. **CLI Structure** RouterOS ใช้ hierarchical menu, prompt แสดง location ปัจจุบัน
3. **Core Verbs** `add`, `set`, `remove`, `print`, `enable`, `disable`, `move` ใช้บ่อยที่สุด
4. **Filtering** ใช้ `where`, `find`, `~` สำหรับ pattern matching
5. **Tab Completion** กด Tab เพื่อ complete command, `?` เพื่อดู help
6. **Export/Import** ใช้สำหรับ backup และ migration
7. **Scripts** RouterOS มี scripting language built-in

### Cheat Sheet

```bash
# Navigation
/          → root
..         → up one level
/ip        → jump to /ip

# CRUD
add param=value
set N param=value
remove N
print [detail|terse|brief]

# Filter
print where param=value
print where param~"pattern"
set [find param=value] new-param=new-value

# Essential
/export file=name
/import file-name=name.rsc
/system backup save name=backup-name
/system reboot
/ping 8.8.8.8
```

---

**[⬅ Previous: Winbox, WebFig, CLI](part-004-winbox-webfig-cli.md)** | **[Next: Network Configuration ➡](part-006-network-configuration.md)**
