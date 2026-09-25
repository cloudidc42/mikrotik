# Part 8: DNS Configuration

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner to Intermediate  
> **เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

1. [DNS Fundamentals](#1-dns-fundamentals)
2. [DNS Client Configuration](#2-dns-client-configuration)
3. [DNS Server (Static Records)](#3-dns-server-static-records)
4. [Dynamic DNS Records](#4-dynamic-dns-records)
5. [DNS Cache](#5-dns-cache)
6. [DNS over HTTPS (DoH)](#6-dns-over-https-doh)
7. [Custom DNS Entries](#7-custom-dns-entries)
8. [DNS Blacklisting](#8-dns-blacklisting)
9. [Split-horizon DNS](#9-split-horizon-dns)
10. [DNS Troubleshooting](#10-dns-troubleshooting)
11. [Lab: DNS Setup สำหรับ Small Office](#11-lab-dns-setup-สำหรับ-small-office)
12. [Summary](#12-summary)

---

## 1. DNS Fundamentals

### 1.1 DNS Process

```
DNS Resolution Process:

Client                    RouterOS DNS        Upstream DNS
                          (Cache/Server)      (8.8.8.8)
  │                            │                   │
  │── Query: google.com? ─────>│                   │
  │                            │                   │
  │         (Cache HIT)        │                   │
  │<── 142.250.x.x ───────────│                   │
  │                            │                   │
  │         (Cache MISS)       │                   │
  │<──────────────────────────│── Query: google? →│
  │                            │                   │
  │                            │<── 142.250.x.x ──│
  │<── 142.250.x.x ───────────│                   │
  │                            │ (cache for TTL)   │
```

### 1.2 DNS Record Types

| Type | Description | Example |
|------|-------------|---------|
| A | IPv4 address | google.com → 142.250.1.1 |
| AAAA | IPv6 address | google.com → 2607:f8b0::200e |
| CNAME | Alias | www.example.com → example.com |
| MX | Mail server | example.com → mail.example.com |
| NS | Name server | example.com → ns1.example.com |
| PTR | Reverse lookup | 1.1.250.142.in-addr.arpa → google.com |
| TXT | Text record | SPF, DKIM records |
| SRV | Service location | _sip._tcp.example.com |
| SOA | Start of authority | Zone information |

### 1.3 RouterOS DNS Components

```
RouterOS DNS:
├── DNS Client   - ส่ง query ไปยัง upstream servers
├── DNS Cache    - Cache responses
├── DNS Server   - ให้บริการ DNS สำหรับ local clients
└── DNS Static   - Static DNS records (custom)
```

---

## 2. DNS Client Configuration

### 2.1 ตั้ง DNS Servers

```bash
# ตั้ง DNS servers (upstream)
/ip dns set servers=8.8.8.8,8.8.4.4

# หรือใช้ Cloudflare
/ip dns set servers=1.1.1.1,1.0.0.1

# หรือ OpenDNS
/ip dns set servers=208.67.222.222,208.67.220.220

# หรือ DNS ของ ISP
/ip dns set servers=203.0.113.1,203.0.113.2

# Multiple servers (fallback)
/ip dns set servers=1.1.1.1,8.8.8.8,8.8.4.4

# ดูการตั้งค่า
/ip dns print

# Output:
#               servers: 1.1.1.1
#                        8.8.8.8
#    dynamic-servers: 203.0.113.53  (จาก DHCP/PPPoE)
# use-doh-server:
#      verify-doh-cert: no
#   allow-remote-requests: no
#         max-udp-packet-size: 4096
#         query-server-timeout: 2s
#         query-total-timeout: 10s
#         max-concurrent-queries: 100
#         max-concurrent-tcp-sessions: 20
#         cache-size: 2048KiB
#         cache-used: 156KiB
```

### 2.2 DNS Client Settings

```bash
# ตั้งค่า DNS client แบบละเอียด
/ip dns set \
    servers=1.1.1.1,8.8.8.8 \
    allow-remote-requests=no \          # ไม่อนุญาตให้ clients query
    max-udp-packet-size=4096 \          # Max UDP packet size
    query-server-timeout=2s \           # Timeout ต่อ server
    query-total-timeout=10s \           # Total timeout
    max-concurrent-queries=100 \        # Max concurrent queries
    cache-size=2048KiB                  # Cache size
```

### 2.3 Dynamic DNS from DHCP/PPPoE

```bash
# เมื่อใช้ DHCP client, ISP DNS จะเพิ่มเป็น dynamic-servers
/ip dhcp-client set [find interface=ether1] use-peer-dns=yes

# เมื่อใช้ PPPoE
/interface pppoe-client set pppoe-out1 use-peer-dns=yes

# ดู dynamic servers
/ip dns print
# dynamic-servers: จะแสดง DNS จาก ISP

# Override: ใช้ servers ของเราแทน peer-dns
/ip dhcp-client set [find interface=ether1] use-peer-dns=no
/ip dns set servers=1.1.1.1,8.8.8.8
```

---

## 3. DNS Server (Static Records)

### 3.1 Enable DNS Server

```bash
# เปิด DNS server สำหรับ clients
/ip dns set allow-remote-requests=yes

# ⚠️ Warning: ถ้าเปิด allow-remote-requests
# ต้องมี firewall ป้องกัน DNS amplification attack
# Block port 53 จาก WAN!

/ip firewall filter add \
    chain=input \
    in-interface-list=WAN \
    protocol=udp \
    dst-port=53 \
    action=drop \
    comment="Block DNS from WAN"

/ip firewall filter add \
    chain=input \
    in-interface-list=WAN \
    protocol=tcp \
    dst-port=53 \
    action=drop \
    comment="Block DNS TCP from WAN"
```

### 3.2 Static DNS Records

```bash
# เพิ่ม A record
/ip dns static add \
    name=server1.local \
    address=192.168.1.10 \
    ttl=1h \
    comment="Web Server"

# เพิ่ม AAAA record (IPv6)
/ip dns static add \
    name=server1.local \
    address=2001:db8::10 \
    type=AAAA \
    comment="Web Server IPv6"

# เพิ่ม CNAME record
/ip dns static add \
    name=www.local \
    cname=server1.local \
    type=CNAME \
    comment="WWW alias"

# เพิ่ม MX record
/ip dns static add \
    name=local \
    mx-exchange=mail.local \
    mx-preference=10 \
    type=MX \
    comment="Mail server"

# เพิ่ม TXT record
/ip dns static add \
    name=local \
    text="v=spf1 mx -all" \
    type=TXT \
    comment="SPF record"

# เพิ่ม PTR record (reverse DNS)
/ip dns static add \
    name=10.1.168.192.in-addr.arpa \
    address=server1.local \
    type=PTR

# ดู static records
/ip dns static print
```

### 3.3 DNS Record Management

```bash
# ดู records แบบ detail
/ip dns static print detail

# ค้นหา record
/ip dns static print where name~"server"

# แก้ไข record
/ip dns static set [find name=server1.local] address=192.168.1.20

# ลบ record
/ip dns static remove [find name=www.local]

# Disable record
/ip dns static disable [find name=server1.local]
```

---

## 4. Dynamic DNS Records

### 4.1 DHCP ไปยัง DNS Integration

```bash
# RouterOS สามารถ auto-add DNS จาก DHCP leases
# ต้องใช้ script ใน DHCP server

/system script add name=dhcp-to-dns source={
    :if ($leaseBound = 1) do={
        # Client connected - add DNS
        :local hostname [/ip dhcp-server lease get \
            [find address=$leaseActIP] host-name]
        
        :if ($hostname != "") do={
            # ลบ record เก่าถ้ามี
            /ip dns static remove [find name=($hostname . ".local")]
            
            # เพิ่ม record ใหม่
            /ip dns static add \
                name=($hostname . ".local") \
                address=$leaseActIP \
                ttl=30m \
                comment=("DHCP: " . $leaseActMAC)
            
            :log info ("DNS added: " . $hostname . ".local = " . $leaseActIP)
        }
    } else={
        # Client disconnected - remove DNS
        :local hostname [/ip dhcp-server lease get \
            [find address=$leaseActIP] host-name]
        
        :if ($hostname != "") do={
            /ip dns static remove [find name=($hostname . ".local")]
            :log info ("DNS removed: " . $hostname . ".local")
        }
    }
}

# เชื่อม script กับ DHCP server
/ip dhcp-server set dhcp-lan lease-script=dhcp-to-dns
```

### 4.2 Dynamic DNS สำหรับ Cloud DDNS

```bash
# สำหรับ DynDNS, No-IP หรือ custom DDNS
/system script add name=update-ddns source={
    # ดู WAN IP ปัจจุบัน
    :local wanIP [/ip dhcp-client get [find interface=WAN] address]
    :local wanIPClean [:pick $wanIP 0 [:find $wanIP "/"]]
    
    # อัปเดต DDNS provider
    :local username "myuser"
    :local password "mypass"
    :local hostname "myhome.ddns.net"
    
    /tool fetch url=("https://dynupdate.no-ip.com/nic/update?hostname=" . \
        $hostname . "&myip=" . $wanIPClean) \
        user=$username \
        password=$password \
        mode=https \
        output=user
    
    :log info ("DDNS updated: " . $hostname . " = " . $wanIPClean)
}

# Schedule: อัปเดตทุก 5 นาที
/system scheduler add \
    name=ddns-update \
    interval=5m \
    on-event=/system/script/run\ update-ddns
```

---

## 5. DNS Cache

### 5.1 DNS Cache Overview

```bash
# ดู cache
/ip dns cache print

# Output:
# NAME                    ADDRESS  TYPE    TTL
# google.com              142.250. A       290s
# www.google.com          google.c CNAME   290s
# cloudflare.com          104.16.  A       296s

# ดู cache statistics
/ip dns print
# cache-size: 2048KiB
# cache-used: 234KiB

# Flush cache
/ip dns cache flush

# ดู cache หลัง flush
/ip dns cache print count-only
# 0
```

### 5.2 Cache Size Configuration

```bash
# ปรับขนาด cache
/ip dns set cache-size=8192KiB   # 8MB cache

# สำหรับ router ที่มี RAM น้อย
/ip dns set cache-size=512KiB    # 512KB cache

# ดู cache usage
:local total [/ip dns get cache-size]
:local used [/ip dns get cache-used]
:put ("Cache: " . $used . " / " . $total)
```

### 5.3 DNS Cache Monitoring

```bash
# Monitor cache
/ip dns cache print interval=10

# ดู specific cached record
/ip dns cache print where name~"google"

# ดู all CNAME records
/ip dns cache print where type=CNAME

# ดู cache hits/misses (ผ่าน torch หรือ log)
/tool torch interface=ether2 port=53
```

---

## 6. DNS over HTTPS (DoH)

### 6.1 DoH Setup

```bash
# RouterOS 7.x รองรับ DNS over HTTPS

# ตั้ง DoH server (Cloudflare)
/ip dns set use-doh-server=https://cloudflare-dns.com/dns-query

# หรือ Google
/ip dns set use-doh-server=https://dns.google/dns-query

# ตรวจสอบ certificate
/ip dns set verify-doh-cert=yes

# ตั้ง upstream servers เป็น fallback
/ip dns set servers=1.1.1.1

# ดูการตั้งค่า
/ip dns print
# use-doh-server: https://cloudflare-dns.com/dns-query
# verify-doh-cert: yes
```

### 6.2 DoH กับ Local DNS

```bash
# DoH สำหรับ upstream queries
# Local static records ยังทำงานได้ปกติ

# Workflow:
# Client query "server1.local" → Check static records → found! ตอบ
# Client query "google.com"    → Check static → not found → Query via DoH
```

### 6.3 Custom DoH Server

```bash
# ถ้าใช้ Pi-hole หรือ AdGuard Home ที่รองรับ DoH
/ip dns set use-doh-server=https://192.168.1.10/dns-query

# อาจต้อง import certificate
/certificate import file-name=ca-cert.pem
/ip dns set verify-doh-cert=yes
```

---

## 7. Custom DNS Entries

### 7.1 Local Domain Resolution

```bash
# Override public domain ให้ resolve เป็น internal IP
# Useful สำหรับ split-horizon DNS

# Override google.com (testing only - ไม่แนะนำ production!)
/ip dns static add \
    name=google.com \
    address=192.168.1.10 \
    comment="TEST ONLY"

# Override ให้ internal services
/ip dns static add name=mail.company.com address=192.168.1.20
/ip dns static add name=vpn.company.com address=192.168.1.30
/ip dns static add name=nas.local address=192.168.1.40
```

### 7.2 Wildcard DNS Entries

```bash
# RouterOS รองรับ wildcard DNS records
# *.local → 192.168.1.10

/ip dns static add \
    name=*.local \
    address=192.168.1.10 \
    comment="Wildcard for all .local domains"

# ทดสอบ
/ping anything.local count=1
/ping another-thing.local count=1
# ทั้งคู่จะ resolve เป็น 192.168.1.10
```

### 7.3 Internal Services DNS

```bash
# สร้าง DNS records สำหรับ internal services
/ip dns static add name=router.local address=192.168.1.1
/ip dns static add name=nas.local address=192.168.1.100
/ip dns static add name=plex.local address=192.168.1.101
/ip dns static add name=unifi.local address=192.168.1.102
/ip dns static add name=pihole.local address=192.168.1.103
/ip dns static add name=home.local address=192.168.1.1

# CNAME records
/ip dns static add name=dashboard.local cname=router.local type=CNAME
```

---

## 8. DNS Blacklisting

### 8.1 DNS Blacklist Method

```bash
# วิธี: Override domain ที่ต้องการ block ให้ resolve เป็น IP ที่เราควบคุม
# หรือ 0.0.0.0 (null) ซึ่งทำให้ connection ล้มเหลว

# Block โดย redirect ไปยัง block page
/ip dns static add \
    name=blocked-site.com \
    address=192.168.1.99 \
    comment="BLOCKED"

# ที่ IP 192.168.1.99 ให้มี web server แสดงหน้า "Content Blocked"

# Block โดยใช้ 0.0.0.0
/ip dns static add \
    name=malicious-site.com \
    address=0.0.0.0 \
    comment="BLOCKED - Malware"
```

### 8.2 Mass DNS Blacklisting ด้วย Script

```bash
# Import blocklist จาก file
# สมมติมี file blocklist.txt ที่มีรายชื่อ domains หนึ่งบรรทัดต่อ domain

/system script add name=import-blocklist source={
    :local blockIP "0.0.0.0"
    :local domains {
        "ads.example.com";
        "tracker.example.com";
        "malware.example.com";
        "phishing.example.com"
    }
    
    :foreach d in=$domains do={
        /ip dns static remove [find name=$d]
        /ip dns static add \
            name=$d \
            address=$blockIP \
            comment="BLOCKED"
        :log info ("Blocked: " . $d)
    }
    
    :put "Blocklist imported"
}

/system script run import-blocklist
```

### 8.3 Download Blocklist

```bash
# Script: Download blocklist จาก internet
/system script add name=update-blocklist source={
    # ดาวน์โหลด blocklist
    /tool fetch \
        url="https://raw.githubusercontent.com/StevenBlack/hosts/master/hosts" \
        dst-path=blocklist.txt \
        mode=https
    
    :log info "Blocklist downloaded"
}
```

### 8.4 DNS-based Ad Blocking (Pi-hole style)

```bash
# ใช้ RouterOS เป็น basic ad blocker
# สำหรับ comprehensive blocking แนะนำ Pi-hole หรือ AdGuard Home

# Script: block common ad domains
/system script add name=block-ads source={
    :local adDomains {
        "doubleclick.net";
        "googleadservices.com";
        "googlesyndication.com";
        "adnxs.com";
        "ads.yahoo.com";
        "amazon-adsystem.com"
    }
    
    :foreach d in=$adDomains do={
        /ip dns static add name=$d address=0.0.0.0 comment="AD-BLOCK" \
            disabled=no
    }
    :put "Ad blocking rules added"
}

/system script run block-ads
```

---

## 9. Split-horizon DNS

### 9.1 Split-horizon คืออะไร

```
Split-horizon DNS (Split-brain DNS):
- Internal clients: resolve hostname เป็น internal IP
- External clients: resolve hostname เป็น public IP

Example:
  Internal: mail.company.com → 192.168.1.20 (internal IP)
  External: mail.company.com → 203.0.113.20 (public IP)
```

### 9.2 Implementation บน RouterOS

```bash
# RouterOS DNS server ตอบสนองต่อ internal clients เท่านั้น
# (เพราะ allow-remote-requests จาก LAN เท่านั้น)

# ตั้ง internal records
/ip dns static add \
    name=mail.company.com \
    address=192.168.1.20 \
    comment="Internal: Mail server"

/ip dns static add \
    name=vpn.company.com \
    address=192.168.1.30 \
    comment="Internal: VPN server"

# Internal clients จะได้ internal IPs
# External clients ใช้ public DNS → ได้ public IPs
```

### 9.3 Hairpin NAT + Split-horizon

```bash
# ปัญหา: Internal users เข้า mail.company.com ผ่าน public IP
# Solution: Hairpin NAT หรือ Split-horizon DNS

# วิธีที่ 1: Split-horizon DNS (แนะนำ)
/ip dns static add \
    name=mail.company.com \
    address=192.168.1.20  # Internal IP โดยตรง

# วิธีที่ 2: Hairpin NAT
/ip firewall nat add \
    chain=srcnat \
    dst-address=203.0.113.20 \
    src-address=192.168.1.0/24 \
    action=masquerade \
    comment="Hairpin NAT"
```

---

## 10. DNS Troubleshooting

### 10.1 ทดสอบ DNS

```bash
# ทดสอบด้วย ping
/ping google.com count=1

# ถ้า ping ไม่ได้ อาจเป็นปัญหา DNS

# ทดสอบ DNS resolution โดยตรง
# RouterOS ไม่มี nslookup/dig built-in
# แต่ใช้ tool fetch หรือ ping แทน

/tool fetch url="http://google.com" mode=http
# ถ้า DNS ทำงาน จะ connect ได้ (หรือมี redirect)

# ดู DNS cache
/ip dns cache print

# Flush cache แล้วทดสอบใหม่
/ip dns cache flush
/ping google.com count=1
```

### 10.2 DNS Logs

```bash
# ดู DNS queries ใน log
/system logging add topics=dns action=memory

/log print where topics~"dns"

# Output:
# 12:00:01 dns  query from 192.168.1.10: google.com
# 12:00:01 dns  resolved: google.com → 142.250.x.x (cached)
```

### 10.3 Common DNS Problems

```
ปัญหา: DNS ไม่ทำงาน (cannot resolve)

Checklist:
□ ตรวจสอบ DNS servers ตั้งไว้หรือไม่
  /ip dns print

□ ตรวจสอบ connectivity ไปยัง DNS server
  /ping 8.8.8.8 count=3

□ ตรวจสอบ firewall ไม่ block port 53
  /ip firewall filter print where dst-port=53

□ ตรวจสอบ allow-remote-requests (ถ้า clients ใช้ router เป็น DNS)
  /ip dns print → allow-remote-requests: yes

□ ตรวจสอบ DHCP ส่ง DNS ให้ clients
  /ip dhcp-server network print → dns-server field

□ ลอง flush cache
  /ip dns cache flush

□ ตรวจสอบ static records ที่อาจ override
  /ip dns static print where disabled=no
```

### 10.4 Packet Capture สำหรับ DNS

```bash
# Capture DNS traffic
/tool sniffer start \
    interface=ether2 \
    filter-port=53 \
    file-name=dns-debug.pcap

# ให้ client ลอง resolve domain
# หยุด capture
/tool sniffer stop

# ดูผล (ต้อง download ไปวิเคราะห์ใน Wireshark)
/file print where name~"dns-debug"
```

### 10.5 DNS Performance Testing

```bash
# ทดสอบ DNS response time
:for i from=1 to=10 do={
    :local start [/system clock get time]
    /tool fetch url="http://google.com" mode=http output=none
    :local end [/system clock get time]
    :put ($i . ": " . ($end - $start) . "ms")
}

# ทดสอบ DNS cache hit vs miss
/ip dns cache flush
# Query ครั้งแรก (cache miss - ช้ากว่า)
/ping google.com count=1
# Query ครั้งสอง (cache hit - เร็วกว่า)
/ping google.com count=1
```

---

## 11. Lab: DNS Setup สำหรับ Small Office

### Lab Topology

```
             Internet
                │
       ┌────────┴────────┐
       │   MikroTik      │
       │   Router        │ 192.168.1.1
       │   DNS Server    │
       └────────┬────────┘
                │ 192.168.1.0/24
     ┌──────────┼──────────┐
     │          │          │
┌────┴───┐ ┌───┴────┐ ┌───┴────┐
│Server  │ │  PC1   │ │  PC2   │
│.1.10   │ │DHCP    │ │DHCP    │
│(Web,   │ │        │ │        │
│NAS,etc)│ │        │ │        │
└────────┘ └────────┘ └────────┘
```

### Lab Step 1: ตั้ง DNS Client

```bash
# ตั้ง upstream DNS servers
/ip dns set \
    servers=1.1.1.1,8.8.8.8 \
    allow-remote-requests=yes \
    cache-size=4096KiB

# ตรวจสอบ
/ip dns print
```

### Lab Step 2: Static DNS Records

```bash
# Record สำหรับ services ภายใน
/ip dns static add name=router.office address=192.168.1.1 comment="Router"
/ip dns static add name=web.office address=192.168.1.10 comment="Web Server"
/ip dns static add name=nas.office address=192.168.1.11 comment="NAS"
/ip dns static add name=printer.office address=192.168.1.12 comment="Printer"

# CNAME records
/ip dns static add name=www.office cname=web.office type=CNAME
/ip dns static add name=files.office cname=nas.office type=CNAME

# MX record สำหรับ mail
/ip dns static add name=office mx-exchange=web.office mx-preference=10 type=MX

# ดูทั้งหมด
/ip dns static print
```

### Lab Step 3: DHCP ส่ง DNS ให้ Clients

```bash
# ตั้ง DHCP network ให้ใช้ router เป็น DNS
/ip dhcp-server network set [find address=192.168.1.0/24] \
    dns-server=192.168.1.1 \
    domain=office

# หมายความว่า:
# Clients จะใช้ 192.168.1.1 เป็น DNS
# Domain ถ้า type "web" จะ resolve เป็น "web.office"
```

### Lab Step 4: DNS Blacklisting

```bash
# Block social media ในช่วงทำงาน
/system script add name=block-social source={
    :local sites {
        "facebook.com";
        "www.facebook.com";
        "twitter.com";
        "www.twitter.com";
        "tiktok.com";
        "www.tiktok.com"
    }
    
    :foreach site in=$sites do={
        /ip dns static remove [find name=$site]
        /ip dns static add \
            name=$site \
            address=192.168.1.1 \
            comment="WORK-HOURS-BLOCK"
    }
    :put "Social media blocked"
}

/system script add name=unblock-social source={
    /ip dns static remove [find comment="WORK-HOURS-BLOCK"]
    :put "Social media unblocked"
}

# Schedule: block 8am-5pm วันจันทร์-ศุกร์
/system scheduler add \
    name=block-social-hours \
    start-time=08:00:00 \
    interval=1d \
    on-event="/system script run block-social"

/system scheduler add \
    name=unblock-social-hours \
    start-time=17:00:00 \
    interval=1d \
    on-event="/system script run unblock-social"
```

### Lab Step 5: DNS Monitoring

```bash
# เปิด DNS logging
/system logging add topics=dns action=memory

# ดู DNS queries
/log print where topics~"dns"

# ดู top domains ในใน cache
/ip dns cache print

# สรุปจำนวน cached records
/ip dns cache print count-only
```

### Lab Step 6: Verification

```bash
# ทดสอบ DNS resolution จาก router
/ping web.office count=3         # ควรตอบ 192.168.1.10
/ping nas.office count=3         # ควรตอบ 192.168.1.11
/ping google.com count=3         # ควรตอบ public IP

# ทดสอบจาก client (Windows)
# nslookup web.office 192.168.1.1
# nslookup google.com 192.168.1.1

# ทดสอบ blacklist
/ping facebook.com count=1       # ควรตอบ 192.168.1.1

# ดู DNS cache
/ip dns cache print where name~"office"
```

---

## 12. Summary

### สิ่งที่เรียนรู้ใน Part 8:

1. **DNS Client** ตั้ง upstream servers เพื่อ resolve domains จาก internet
2. **DNS Server** เปิด allow-remote-requests ให้ clients ใช้ router เป็น DNS
3. **Static Records** สร้าง A, CNAME, MX, TXT records สำหรับ internal services
4. **Dynamic Records** ใช้ DHCP script อัปเดต DNS อัตโนมัติ
5. **DNS Cache** cache DNS responses ลด latency
6. **DoH** DNS over HTTPS เข้ารหัส DNS queries
7. **Blacklisting** Block domains ด้วย static records
8. **Split-horizon** ตอบ DNS ต่างกันสำหรับ internal/external clients

### Quick Reference

```bash
# DNS Client
/ip dns set servers=1.1.1.1,8.8.8.8 allow-remote-requests=yes

# Static Records
/ip dns static add name=server1.local address=192.168.1.10
/ip dns static add name=www.local cname=server1.local type=CNAME

# Cache
/ip dns cache flush
/ip dns cache print

# DoH
/ip dns set use-doh-server=https://cloudflare-dns.com/dns-query verify-doh-cert=yes

# Monitoring
/ip dns cache print
/log print where topics~"dns"
```

---

**[⬅ Previous: DHCP Server & Client](part-007-dhcp-server-client.md)** | **[Next: NAT & Masquerade ➡](part-009-nat-masquerade.md)**
