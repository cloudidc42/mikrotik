# Part 6: Network Configuration

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner to Intermediate  
> **เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

1. [Interface Types ใน RouterOS](#1-interface-types-ใน-routeros)
2. [IP Address Assignment](#2-ip-address-assignment)
3. [Multiple IPs on One Interface](#3-multiple-ips-on-one-interface)
4. [Interface Aliases](#4-interface-aliases)
5. [VLAN Configuration](#5-vlan-configuration)
6. [IP Pool](#6-ip-pool)
7. [ARP Settings](#7-arp-settings)
8. [Network Diagnostics](#8-network-diagnostics)
9. [IP Services](#9-ip-services)
10. [Lab Exercise: Setup 3-Router Network](#10-lab-exercise-setup-3-router-network)
11. [Summary](#11-summary)

---

## 1. Interface Types ใน RouterOS

### 1.1 Ethernet Interfaces

```bash
# ดู Ethernet interfaces
/interface ethernet print

# Output:
# Flags: X - disabled, R - running, S - slave
#  #    NAME          MTU  MAC-ADDRESS        SPEED         FULL-DUPLEX
#  0  R ether1        1500 XX:XX:XX:XX:XX:01  1Gbps         yes
#  1  R ether2        1500 XX:XX:XX:XX:XX:02  1Gbps         yes
#  2    ether3        1500 XX:XX:XX:XX:XX:03  (no link)

# ตรวจสอบ details
/interface ethernet print detail
```

**Ethernet Settings:**
```bash
# ตั้ง Speed และ Duplex (บังคับ)
/interface ethernet set ether1 \
    speed=1Gbps \
    full-duplex=yes \
    auto-negotiation=no

# หรือใช้ auto-negotiation (แนะนำ)
/interface ethernet set ether1 auto-negotiation=yes

# ตั้ง MTU
/interface ethernet set ether1 mtu=9000  # Jumbo frames

# ดู statistics
/interface ethernet monitor ether1

# Monitor real-time
/interface ethernet monitor ether1 once
```

### 1.2 VLAN Interfaces

```bash
# สร้าง VLAN interface
/interface vlan add \
    name=vlan10 \
    vlan-id=10 \
    interface=ether2 \
    comment="Management VLAN"

/interface vlan add \
    name=vlan20 \
    vlan-id=20 \
    interface=ether2 \
    comment="User VLAN"

/interface vlan add \
    name=vlan30 \
    vlan-id=30 \
    interface=ether2 \
    comment="Guest VLAN"

# ดู VLAN interfaces
/interface vlan print
```

### 1.3 Bridge Interfaces

```bash
# สร้าง Bridge
/interface bridge add \
    name=bridge1 \
    comment="LAN Bridge" \
    stp=no \
    fast-forward=yes

# เพิ่ม ports เข้า bridge
/interface bridge port add \
    bridge=bridge1 \
    interface=ether2

/interface bridge port add \
    bridge=bridge1 \
    interface=ether3

/interface bridge port add \
    bridge=bridge1 \
    interface=wlan1

# ดู bridge
/interface bridge print
/interface bridge port print

# ดู bridge MAC table
/interface bridge host print
```

### 1.4 Wireless Interfaces

```bash
# ดู wireless interfaces
/interface wireless print

# ตั้ง wireless mode
/interface wireless set wlan1 \
    mode=ap-bridge \
    ssid=MyNetwork \
    band=2ghz-b/g/n \
    frequency=auto \
    channel-width=20/40mhz-Ce \
    wireless-protocol=802.11

# ตั้ง wireless security
/interface wireless security-profiles set default \
    mode=dynamic-keys \
    authentication-types=wpa2-psk \
    wpa2-pre-shared-key=MyPassword123

# Apply security profile
/interface wireless set wlan1 security-profile=default
```

### 1.5 Loopback Interface

```bash
# RouterOS ไม่มี loopback interface จริงๆ แบบ Linux
# แต่ใช้ Bridge interface เป็น loopback แทน

# สร้าง loopback
/interface bridge add \
    name=loopback \
    comment="Loopback interface"

# ตั้ง IP
/ip address add \
    address=10.255.255.1/32 \
    interface=loopback \
    comment="Router ID"
```

### 1.6 Tunnel Interfaces

```bash
# EoIP (Ethernet over IP)
/interface eoip add \
    name=eoip1 \
    remote-address=10.0.0.2 \
    tunnel-id=1 \
    comment="EoIP to R2"

# GRE (Generic Routing Encapsulation)
/interface gre add \
    name=gre1 \
    remote-address=10.0.0.2 \
    comment="GRE tunnel to R2"

# IPIP
/interface ipip add \
    name=ipip1 \
    remote-address=10.0.0.2 \
    comment="IPIP tunnel to R2"

# WireGuard (RouterOS 7+)
/interface wireguard add \
    name=wg0 \
    listen-port=51820 \
    comment="WireGuard VPN"
```

### 1.7 PPPoE Interface

```bash
# PPPoE Client (สำหรับ ISP connection)
/interface pppoe-client add \
    name=pppoe-out1 \
    interface=ether1 \
    user=isp-username \
    password=isp-password \
    add-default-route=yes \
    dial-on-demand=no \
    use-peer-dns=yes \
    comment="ISP PPPoE"

# ดู PPPoE status
/interface pppoe-client print detail
/interface pppoe-client monitor pppoe-out1
```

---

## 2. IP Address Assignment

### 2.1 Static IP Assignment

```bash
# เพิ่ม IP address
/ip address add \
    address=192.168.1.1/24 \
    interface=ether1 \
    comment="LAN"

# ดู IP addresses
/ip address print

# Output:
# Flags: X - disabled, I - invalid, D - dynamic
#  #   ADDRESS            NETWORK         INTERFACE  COMMENT
#  0   192.168.1.1/24     192.168.1.0     ether1     LAN
```

### 2.2 DHCP Client (Dynamic IP)

```bash
# ตั้ง DHCP client บน WAN interface
/ip dhcp-client add \
    interface=ether1 \
    disabled=no \
    add-default-route=yes \
    default-route-distance=1 \
    use-peer-dns=yes \
    comment="WAN DHCP"

# ดู DHCP client status
/ip dhcp-client print
/ip dhcp-client print detail

# Release และ Renew DHCP
/ip dhcp-client release [find interface=ether1]
/ip dhcp-client renew [find interface=ether1]
```

### 2.3 IP Address Management

```bash
# ลบ IP address
/ip address remove [find interface=ether3]

# Disable IP address
/ip address disable [find interface=ether3]

# Enable IP address
/ip address enable [find interface=ether3]

# แก้ไข IP address
/ip address set [find interface=ether1] address=192.168.2.1/24

# ดู IP พร้อม network info
/ip address print detail

# Output (detail):
#  0   address=192.168.1.1/24
#      network=192.168.1.0
#      broadcast=192.168.1.255
#      interface=ether1
#      actual-interface=ether1
#      invalid=no
#      dynamic=no
#      disabled=no
#      comment=LAN
```

---

## 3. Multiple IPs on One Interface

### 3.1 Secondary IP Addresses

```bash
# เพิ่ม IP addresses หลายตัวบน interface เดียว
/ip address add address=192.168.1.1/24 interface=ether2
/ip address add address=10.0.0.1/8 interface=ether2
/ip address add address=172.16.0.1/12 interface=ether2

# ดูทั้งหมด
/ip address print where interface=ether2

# Output:
#  0   192.168.1.1/24     192.168.1.0     ether2
#  1   10.0.0.1/8         10.0.0.0        ether2
#  2   172.16.0.1/12      172.16.0.0      ether2
```

### 3.2 Use Case: Multiple Subnets

```bash
# Scenario: Router ให้บริการหลาย subnet บน interface เดียว
# ใช้ใน Multi-tenant hosting, หรือ VLAN over single port

# ISP ที่มีหลาย customer subnet บน interface เดียว
/ip address add address=10.1.0.1/24 interface=ether2 comment="Customer A"
/ip address add address=10.2.0.1/24 interface=ether2 comment="Customer B"
/ip address add address=10.3.0.1/24 interface=ether2 comment="Customer C"
```

---

## 4. Interface Aliases

### 4.1 Naming Interfaces

```bash
# เปลี่ยนชื่อ interface
/interface set ether1 name=WAN
/interface set ether2 name=LAN
/interface set ether3 name=DMZ
/interface set ether4 name=WIFI
/interface set sfp-sfpplus1 name=UPLINK

# ดูชื่อใหม่
/interface print
```

### 4.2 Interface Comment

```bash
# เพิ่ม comment
/interface set WAN comment="ISP True Business 1Gbps"
/interface set LAN comment="Office LAN 192.168.1.0/24"
/interface set DMZ comment="DMZ for web/mail servers"

# ดู comments
/interface print detail where comment!=""
```

### 4.3 Interface Lists

```bash
# สร้าง interface lists สำหรับ firewall/NAT
/interface list add name=WAN comment="Internet-facing interfaces"
/interface list add name=LAN comment="Internal network interfaces"

# เพิ่ม members
/interface list member add interface=ether1 list=WAN
/interface list member add interface=ether2 list=LAN
/interface list member add interface=ether3 list=LAN
/interface list member add interface=bridge1 list=LAN
/interface list member add interface=vlan10 list=LAN

# ดู lists
/interface list print
/interface list member print

# ใช้ใน firewall
/ip firewall filter add \
    chain=forward \
    in-interface-list=WAN \
    out-interface-list=LAN \
    action=drop \
    comment="Block unsolicited from WAN to LAN"
```

---

## 5. VLAN Configuration

### 5.1 VLAN Overview

```
VLAN Concepts:
├── Access Port: port ที่เชื่อมต่อ end device (untagged)
├── Trunk Port: port ที่ส่ง multiple VLANs (tagged)
├── Native VLAN: VLAN ที่ไม่มี tag บน trunk port
└── VLAN ID: 1-4094
```

### 5.2 Router-on-a-Stick Configuration

```bash
# Topology:
# PC1 (VLAN 10) ─┐
# PC2 (VLAN 20) ─┤ Switch ─── ether2 (trunk) ─── MikroTik
# PC3 (VLAN 30) ─┘

# สร้าง VLAN interfaces
/interface vlan add name=vlan10 vlan-id=10 interface=ether2
/interface vlan add name=vlan20 vlan-id=20 interface=ether2
/interface vlan add name=vlan30 vlan-id=30 interface=ether2

# ตั้ง IP addresses
/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20
/ip address add address=192.168.30.1/24 interface=vlan30

# ตั้ง DHCP server สำหรับแต่ละ VLAN
/ip dhcp-server add name=dhcp-vlan10 interface=vlan10 address-pool=pool-vlan10
/ip dhcp-server add name=dhcp-vlan20 interface=vlan20 address-pool=pool-vlan20
/ip dhcp-server add name=dhcp-vlan30 interface=vlan30 address-pool=pool-vlan30

# สร้าง IP pools
/ip pool add name=pool-vlan10 ranges=192.168.10.10-192.168.10.254
/ip pool add name=pool-vlan20 ranges=192.168.20.10-192.168.20.254
/ip pool add name=pool-vlan30 ranges=192.168.30.10-192.168.30.254

# ตั้ง DHCP network
/ip dhcp-server network add address=192.168.10.0/24 gateway=192.168.10.1
/ip dhcp-server network add address=192.168.20.0/24 gateway=192.168.20.1
/ip dhcp-server network add address=192.168.30.0/24 gateway=192.168.30.1
```

### 5.3 Bridge with VLANs (Modern MikroTik Way)

```bash
# RouterOS 6.41+ แนะนำ Bridge VLAN Filtering
# Topology:
# ether1 = WAN
# ether2 = Trunk (switch)
# ether3 = Access port VLAN 10
# ether4 = Access port VLAN 20

# สร้าง bridge พร้อม VLAN filtering
/interface bridge add \
    name=bridge1 \
    vlan-filtering=yes \
    frame-types=admit-all

# เพิ่ม ports
/interface bridge port add interface=ether2 bridge=bridge1 frame-types=admit-all
/interface bridge port add interface=ether3 bridge=bridge1 pvid=10 frame-types=admit-only-untagged-and-priority-tagged
/interface bridge port add interface=ether4 bridge=bridge1 pvid=20 frame-types=admit-only-untagged-and-priority-tagged

# กำหนด VLANs บน bridge
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=bridge1,ether2 untagged=ether3
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=bridge1,ether2 untagged=ether4

# เพิ่ม IP สำหรับ management และ routing
/interface vlan add name=vlan10 vlan-id=10 interface=bridge1
/interface vlan add name=vlan20 vlan-id=20 interface=bridge1

/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20
```

### 5.4 VLAN Troubleshooting

```bash
# ดู VLAN configuration
/interface bridge vlan print

# ดู bridge ports
/interface bridge port print detail

# ดู MAC table
/interface bridge host print

# ตรวจสอบ traffic บน VLAN interface
/interface monitor vlan10

# Packet capture บน VLAN
/tool sniffer start interface=vlan10 file-name=vlan10-capture
/tool sniffer stop
```

---

## 6. IP Pool

### 6.1 สร้าง IP Pool

```bash
# สร้าง pool เดี่ยว
/ip pool add \
    name=dhcp-pool-lan \
    ranges=192.168.1.10-192.168.1.254 \
    comment="LAN DHCP Pool"

# สร้าง pool หลาย ranges
/ip pool add \
    name=dhcp-pool-wifi \
    ranges=10.0.0.1-10.0.0.50,10.0.0.100-10.0.0.200 \
    comment="WiFi DHCP Pool (skip 51-99)"

# สร้าง pool สำหรับ VPN
/ip pool add \
    name=vpn-pool \
    ranges=10.10.0.1-10.10.0.254 \
    next-pool=vpn-pool2  # overflow ไปที่ pool ถัดไป

/ip pool add \
    name=vpn-pool2 \
    ranges=10.10.1.1-10.10.1.254 \
    comment="VPN Pool overflow"
```

### 6.2 ดูและจัดการ Pool

```bash
# ดู pools
/ip pool print

# ดู pool usage
/ip pool used print

# Output:
# POOL            ADDRESS         OWNER               INFO
# dhcp-pool-lan   192.168.1.10    dhcp server (LAN)   AA:BB:CC:DD:EE:FF
# dhcp-pool-lan   192.168.1.11    dhcp server (LAN)   11:22:33:44:55:66

# ดูจำนวน IPs ที่ใช้
/ip pool used print count-only
```

---

## 7. ARP Settings

### 7.1 ARP Overview

```bash
# ดู ARP table
/ip arp print

# Output:
# Flags: X - disabled, I - invalid, H - DHCP, D - dynamic, P - published, C - complete
#  #    ADDRESS         MAC-ADDRESS        INTERFACE  STATUS
#  0 DC 192.168.1.2     AA:BB:CC:DD:EE:FF  ether2     reachable
#  1 DC 192.168.1.3     11:22:33:44:55:66  ether2     reachable

# เพิ่ม static ARP entry
/ip arp add address=192.168.1.100 mac-address=AA:BB:CC:DD:EE:FF interface=ether2
```

### 7.2 ARP Modes บน Interface

```bash
# ARP Modes:
# enabled   - standard ARP (default)
# disabled  - ไม่ reply ARP (ใช้กับ static ARP เท่านั้น)
# proxy-arp - reply ARP แทน hosts ใน network อื่น
# local-proxy-arp - proxy ARP สำหรับ hosts ใน subnet เดียวกัน
# reply-only - ไม่ส่ง ARP request, แต่ reply ได้

# ตั้ง ARP mode
/interface set ether2 arp=enabled
/interface set ether2 arp=proxy-arp

# ใช้ proxy-arp เมื่อ:
# - Router ต้องการ forward ระหว่าง subnets ที่อยู่บน interface เดียวกัน
# - บาง VPN scenarios
```

### 7.3 ARP Table Management

```bash
# ดู ARP table พร้อมรายละเอียด
/ip arp print detail

# ลบ ARP entries ที่ invalid
/ip arp remove [find invalid=yes]

# Flush ARP table ทั้งหมด (dynamic entries)
/ip arp remove [find dynamic=yes]

# Monitor ARP table
/ip arp print interval=1
```

---

## 8. Network Diagnostics

### 8.1 Ping

```bash
# Basic ping
/ping 8.8.8.8

# Ping ด้วย count
/ping 8.8.8.8 count=10

# Ping ด้วย size
/ping 8.8.8.8 size=1400 count=5

# Ping ด้วย interval
/ping 8.8.8.8 interval=0.5 count=20

# Ping จาก specific interface
/ping 8.8.8.8 src-address=192.168.1.1 count=5

# Ping IPv6
/ping 2001:4860:4860::8888 count=5

# Output:
#   SEQ HOST            SIZE TTL TIME       STATUS
#     0 8.8.8.8          56  55  14ms        echo reply
#     1 8.8.8.8          56  55  12ms        echo reply
#     2 8.8.8.8          56  55  13ms        echo reply
#  3 packets transmitted, 3 received, 0% packet loss
#  round-trip min/avg/max = 12/13.0/14 ms
```

### 8.2 Traceroute

```bash
# Basic traceroute
/tool traceroute 8.8.8.8

# Traceroute ด้วย options
/tool traceroute 8.8.8.8 \
    count=3 \
    timeout=2 \
    max-hops=30 \
    src-address=192.168.1.1

# Output:
#  # ADDRESS        LOSS  LAST   AVG   BEST  WORST  STD-DEV STATUS
#  1 10.0.0.1       0%    1ms    1.1ms  1ms  2ms     0.2ms   echo reply
#  2 100.64.0.1     0%    5ms    5.2ms  5ms  6ms     0.4ms   echo reply
#  3 203.0.113.1    0%    10ms   10ms   10ms 11ms    0.3ms   echo reply
#  4 8.8.8.8        0%    15ms   15ms   15ms 16ms    0.4ms   echo reply
```

### 8.3 Bandwidth Test

```bash
# ทดสอบ bandwidth ระหว่าง routers
# ต้องเปิด bandwidth-test server บน target

# เปิด server
/tool bandwidth-server set enabled=yes

# ทดสอบจาก client
/tool bandwidth-test address=192.168.1.2 \
    user=admin \
    password=password \
    direction=both \
    duration=10s

# Output:
#    STATUS: running
#    DURATION: 10s
#    TX-CURRENT: 940Mbps
#    RX-CURRENT: 930Mbps
#    TX-TOTAL-AVERAGE: 938Mbps
#    RX-TOTAL-AVERAGE: 928Mbps
```

### 8.4 Packet Sniffer / Torch

```bash
# Torch - real-time traffic monitor
/tool torch interface=ether1

# Filter torch by protocol
/tool torch interface=ether1 ip-protocol=tcp

# Filter by port
/tool torch interface=ether1 port=80,443

# Filter by IP
/tool torch interface=ether1 src-address=192.168.1.100

# Packet Sniffer (pcap capture)
# เริ่ม capture
/tool sniffer start \
    interface=ether1 \
    filter-ip-address=8.8.8.8 \
    filter-port=53 \
    file-name=dns-capture.pcap \
    file-limit=10MB

# หยุด capture
/tool sniffer stop

# ดูไฟล์
/file print where name~"capture"

# Download ไปวิเคราะห์ใน Wireshark
```

### 8.5 DNS Testing

```bash
# ดู DNS configuration
/ip dns print

# ทดสอบ DNS resolution
/ip dns cache flush
/ping google.com count=1

# ดู DNS cache
/ip dns cache print

# ดู DNS entries ที่ resolve แล้ว
/ip dns cache print detail

# Test DNS ด้วย nslookup-like command
/tool fetch url="https://8.8.8.8/resolve?name=google.com&type=A" \
    mode=https output=user
```

### 8.6 Connection Tracking

```bash
# ดู active connections
/ip firewall connection print

# ดู connections แบบ real-time
/ip firewall connection print interval=1

# ดู connections ไปยัง specific host
/ip firewall connection print where dst-address~"8.8.8.8"

# ดู TCP connections
/ip firewall connection print where protocol=tcp

# ดู connection count
/ip firewall connection print count-only

# Output:
# Flags: S - seen reply, A - assured, C - confirmed, E - expiring
#  # PROTOCOL  SRC-ADDRESS          DST-ADDRESS          TCP-STATE
#  0 tcp       192.168.1.10:54321   8.8.8.8:443          established
#  1 tcp       192.168.1.20:12345   1.2.3.4:80            established
```

---

## 9. IP Services

### 9.1 ดูและจัดการ Services

```bash
# ดู services ทั้งหมด
/ip service print

# Output:
# NAME     PORT  ADDRESS      CERT    INVALID  DISABLED
# telnet   23                 none    no       no
# ftp      21                 none    no       no
# www      80                 none    no       no
# ssh      22                 none    no       no
# www-ssl  443                none    no       no
# api      8728               none    no       no
# winbox   8291               none    no       no
# api-ssl  8729               none    no       no
```

### 9.2 Security Best Practices

```bash
# Disable ไม่ใช้
/ip service disable telnet
/ip service disable ftp
/ip service disable api  # ถ้าไม่ใช้ API

# เปลี่ยน port (port obfuscation - ป้องกัน script kiddies)
/ip service set ssh port=2222
/ip service set www-ssl port=8443
/ip service set winbox port=8292

# จำกัด access จาก specific networks เท่านั้น
/ip service set ssh address=192.168.1.0/24,10.0.0.0/8
/ip service set winbox address=192.168.1.0/24
/ip service set www address=192.168.1.0/24

# ตั้ง HTTPS certificate
/ip service set www-ssl certificate=my-cert tls-version=only-1.2

# ตรวจสอบ
/ip service print
```

### 9.3 SSH Service Configuration

```bash
# ดู SSH config
/ip ssh print

# ตั้ง SSH options
/ip ssh set \
    forwarding-enabled=no \
    host-key-size=4096 \
    strong-crypto=yes

# Generate new host key
/ip ssh regenerate-host-key
```

---

## 10. Lab Exercise: Setup 3-Router Network

### Lab Overview

```
Lab Topology:

    Internet (Simulated)
         │
    ┌────┴────┐
    │ Router1 │  WAN: DHCP
    │ (Edge)  │  LAN1: 192.168.1.1/24
    └────┬────┘
         │ 192.168.1.0/24 (LAN1)
    ┌────┴────┐
    │ Router2 │  LAN1: 192.168.1.2/24 (uplink)
    │(Branch1)│  LAN2: 192.168.2.1/24
    └────┬────┘
         │ 192.168.2.0/24 (LAN2)
    ┌────┴────┐
    │ Router3 │  LAN2: 192.168.2.2/24 (uplink)
    │(Branch2)│  LAN3: 192.168.3.1/24
    └─────────┘
         │ 192.168.3.0/24 (LAN3)
     PCs (192.168.3.x)
```

### Step 1: Router 1 Configuration (Edge Router)

```bash
# Login ไปยัง Router 1

# ตั้งชื่อ
/system identity set name=Router1-Edge

# ตั้ง timezone
/system clock set time-zone-name=Asia/Bangkok

# ตั้งชื่อ interfaces
/interface set ether1 name=WAN comment="ISP Connection"
/interface set ether2 name=LAN1 comment="Internal Network"

# WAN - DHCP client
/ip dhcp-client add interface=WAN disabled=no

# LAN - Static IP
/ip address add address=192.168.1.1/24 interface=LAN1

# Default route (ถ้า DHCP ไม่ auto)
# /ip route add dst-address=0.0.0.0/0 gateway=[WAN-gateway]

# DHCP server สำหรับ LAN1
/ip pool add name=pool-lan1 ranges=192.168.1.100-192.168.1.200
/ip dhcp-server add name=dhcp-lan1 interface=LAN1 address-pool=pool-lan1
/ip dhcp-server network add address=192.168.1.0/24 gateway=192.168.1.1 dns-server=8.8.8.8

# DNS
/ip dns set servers=8.8.8.8,8.8.4.4 allow-remote-requests=yes

# NAT (Masquerade สำหรับ Internet access)
/ip firewall nat add chain=srcnat out-interface=WAN action=masquerade

# Static routes ไปยัง networks ด้านใน
/ip route add dst-address=192.168.2.0/24 gateway=192.168.1.2 comment="To Router2 LAN2"
/ip route add dst-address=192.168.3.0/24 gateway=192.168.1.2 comment="To Router3 LAN3"

# Basic firewall
/ip firewall filter add chain=input connection-state=established,related action=accept
/ip firewall filter add chain=input in-interface=WAN action=drop
/ip firewall filter add chain=forward in-interface=WAN connection-state=new action=drop
```

### Step 2: Router 2 Configuration (Branch 1)

```bash
# Login ไปยัง Router 2

/system identity set name=Router2-Branch1
/system clock set time-zone-name=Asia/Bangkok

/interface set ether1 name=UPLINK comment="To Router1"
/interface set ether2 name=LAN2 comment="Branch1 LAN"

# Uplink IP (ใน subnet ของ Router1)
/ip address add address=192.168.1.2/24 interface=UPLINK

# LAN2 IP
/ip address add address=192.168.2.1/24 interface=LAN2

# Default route ผ่าน Router1
/ip route add dst-address=0.0.0.0/0 gateway=192.168.1.1

# Static route ไปยัง LAN3 ผ่าน Router3
/ip route add dst-address=192.168.3.0/24 gateway=192.168.2.2

# DHCP สำหรับ LAN2
/ip pool add name=pool-lan2 ranges=192.168.2.100-192.168.2.200
/ip dhcp-server add name=dhcp-lan2 interface=LAN2 address-pool=pool-lan2
/ip dhcp-server network add address=192.168.2.0/24 gateway=192.168.2.1 dns-server=8.8.8.8

/ip dns set servers=8.8.8.8,8.8.4.4
```

### Step 3: Router 3 Configuration (Branch 2)

```bash
# Login ไปยัง Router 3

/system identity set name=Router3-Branch2
/system clock set time-zone-name=Asia/Bangkok

/interface set ether1 name=UPLINK comment="To Router2"
/interface set ether2 name=LAN3 comment="Branch2 LAN"

# Uplink IP
/ip address add address=192.168.2.2/24 interface=UPLINK

# LAN3 IP
/ip address add address=192.168.3.1/24 interface=LAN3

# Default route ผ่าน Router2
/ip route add dst-address=0.0.0.0/0 gateway=192.168.2.1

# DHCP สำหรับ LAN3
/ip pool add name=pool-lan3 ranges=192.168.3.100-192.168.3.200
/ip dhcp-server add name=dhcp-lan3 interface=LAN3 address-pool=pool-lan3
/ip dhcp-server network add address=192.168.3.0/24 gateway=192.168.3.1 dns-server=8.8.8.8

/ip dns set servers=8.8.8.8,8.8.4.4
```

### Step 4: Verification

```bash
# บน Router 1 - ทดสอบ connectivity
/ping 192.168.1.2 count=3  # ไปยัง Router2
/ping 192.168.2.1 count=3  # ไปยัง Router2 LAN
/ping 192.168.2.2 count=3  # ไปยัง Router3
/ping 192.168.3.1 count=3  # ไปยัง Router3 LAN
/ping 8.8.8.8 count=3      # Internet

# บน Router 3 - ทดสอบ
/ping 8.8.8.8 count=3      # Internet ผ่าน Router1
/ping 192.168.1.1 count=3  # Router1
/ping 192.168.2.1 count=3  # Router2

# Traceroute จาก Router3 ไป Internet
/tool traceroute 8.8.8.8
# ควรเห็น: R3 → R2 → R1 → ISP → 8.8.8.8

# ดู routing table
/ip route print

# ดู ARP table
/ip arp print
```

### Lab Verification Checklist

```
✅ Router1:
□ WAN ได้ IP จาก DHCP
□ LAN1 IP: 192.168.1.1/24
□ Ping 8.8.8.8 ได้จาก Router1
□ DHCP server ทำงานบน LAN1
□ Routes ไปยัง 192.168.2.0/24 และ 192.168.3.0/24

✅ Router2:
□ UPLINK IP: 192.168.1.2/24
□ LAN2 IP: 192.168.2.1/24
□ Ping 8.8.8.8 ได้จาก Router2
□ Ping 192.168.3.1 ได้จาก Router2
□ DHCP server ทำงานบน LAN2

✅ Router3:
□ UPLINK IP: 192.168.2.2/24
□ LAN3 IP: 192.168.3.1/24
□ Ping 8.8.8.8 ได้จาก Router3
□ Ping 192.168.1.1 ได้จาก Router3
□ DHCP server ทำงานบน LAN3
```

---

## 11. Summary

### สิ่งที่เรียนรู้ใน Part 6:

1. **Interface Types** RouterOS รองรับ Ethernet, VLAN, Bridge, Wireless, Tunnel interfaces
2. **IP Assignment** ทำได้ทั้ง Static และ DHCP client
3. **Multiple IPs** สามารถเพิ่มหลาย IP addresses บน interface เดียว
4. **VLAN** ใช้ Router-on-a-stick หรือ Bridge VLAN Filtering
5. **IP Pool** สำหรับ DHCP และ VPN address allocation
6. **ARP** ควบคุม ARP behavior ด้วย modes ต่างๆ
7. **Diagnostics** ใช้ ping, traceroute, torch, sniffer
8. **IP Services** จำกัด และ harden management services

### Quick Reference

```bash
# Interface management
/interface print
/interface set ether1 name=WAN

# IP management
/ip address add address=x.x.x.x/y interface=ethN
/ip address print
/ip address remove [find interface=ethN]

# Routing
/ip route add dst-address=0.0.0.0/0 gateway=x.x.x.x
/ip route print where active=yes

# VLAN
/interface vlan add name=vlanN vlan-id=N interface=ethN
/ip address add address=x.x.x.x/y interface=vlanN

# Diagnostics
/ping 8.8.8.8 count=5
/tool traceroute 8.8.8.8
/tool torch interface=ether1
```

---

**[⬅ Previous: CLI Basics](part-005-cli-basics.md)** | **[Next: DHCP Server & Client ➡](part-007-dhcp-server-client.md)**
