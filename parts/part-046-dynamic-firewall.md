# Part 46: Dynamic Firewall และ Automated Threat Protection

## บทนำ

Dynamic Firewall คือระบบที่สร้างและจัดการ firewall rules โดยอัตโนมัติตาม traffic pattern และ threat intelligence ช่วยให้ router ป้องกันตัวเองได้โดยไม่ต้องรอ manual intervention

---

## 46.1 Dynamic Address Lists

### Address List คืออะไร?

Address List เป็น named group ของ IP addresses ที่ใช้ใน firewall rules สามารถเพิ่ม/ลบ IP ได้แบบ dynamic ด้วย firewall rules หรือ scripts

```routeros
# สร้าง static address list
/ip firewall address-list
add list=TRUSTED-IPS address=192.168.1.100
add list=TRUSTED-IPS address=10.0.0.0/8

# สร้าง dynamic list ด้วย firewall
/ip firewall filter
add chain=input protocol=tcp dst-port=22 \
    action=add-src-to-address-list \
    address-list=SSH-ATTEMPTS \
    address-list-timeout=1h \
    comment="Track SSH connection attempts"
```

### Dynamic Address List Types

| ประเภท | คำอธิบาย | Timeout |
|--------|----------|---------|
| Static | เพิ่มด้วย manual | ไม่มีหมดอายุ |
| Dynamic (firewall) | เพิ่มโดย firewall rule | มี timeout |
| Dynamic (script) | เพิ่มโดย script | Optional timeout |
| Dynamic (netwatch) | เพิ่มโดย netwatch | Optional timeout |

---

## 46.2 Auto-ban Based on Patterns

### Brute Force SSH Detection

```routeros
/ip firewall filter

# ตรวจจับ SSH brute force - ถ้า connect มากกว่า 3 ครั้งใน 1 นาที
add chain=input protocol=tcp dst-port=22 \
    state=new \
    src-address-list=SSH-STAGE-1 \
    action=add-src-to-address-list \
    address-list=SSH-BLACKLIST \
    address-list-timeout=1d \
    log=yes log-prefix="SSH-BANNED:" \
    comment="Auto-ban: 3rd SSH attempt"

add chain=input protocol=tcp dst-port=22 \
    state=new \
    src-address-list=SSH-STAGE-0 \
    action=add-src-to-address-list \
    address-list=SSH-STAGE-1 \
    address-list-timeout=1m \
    comment="SSH track: 2nd attempt"

add chain=input protocol=tcp dst-port=22 \
    state=new \
    action=add-src-to-address-list \
    address-list=SSH-STAGE-0 \
    address-list-timeout=1m \
    comment="SSH track: 1st attempt"

# Drop blacklisted IPs
add chain=input \
    src-address-list=SSH-BLACKLIST \
    action=drop \
    log=yes log-prefix="BLACKLIST-DROP:" \
    comment="Drop blacklisted IPs"

# Allow SSH (after brute force check)
add chain=input protocol=tcp dst-port=22 \
    action=accept \
    comment="Allow SSH"
```

### HTTP Flood Detection

```routeros
/ip firewall filter

# ตรวจจับ HTTP flood
add chain=forward protocol=tcp dst-port=80,443 \
    connection-limit=100,32 \
    action=add-src-to-address-list \
    address-list=HTTP-FLOOD \
    address-list-timeout=30m \
    log=yes log-prefix="HTTP-FLOOD:" \
    comment="HTTP flood detection"

# Block HTTP flood
add chain=forward \
    src-address-list=HTTP-FLOOD \
    action=drop \
    comment="Block HTTP flood"
```

---

## 46.3 Brute Force Protection

### Multi-service Brute Force Protection

```routeros
/ip firewall filter

# ==========================================
# BRUTE FORCE PROTECTION - MULTIPLE SERVICES
# ==========================================

# SSH Brute Force
add chain=input protocol=tcp dst-port=22 \
    src-address-list=BF-SSH-STAGE2 \
    action=add-src-to-address-list \
    address-list=BLOCKED-IPS \
    address-list-timeout=24h \
    comment="BF: SSH ban after 3 attempts"

add chain=input protocol=tcp dst-port=22 \
    state=new src-address-list=BF-SSH-STAGE1 \
    action=add-src-to-address-list \
    address-list=BF-SSH-STAGE2 \
    address-list-timeout=2m \
    comment="BF: SSH stage 2"

add chain=input protocol=tcp dst-port=22 \
    state=new \
    action=add-src-to-address-list \
    address-list=BF-SSH-STAGE1 \
    address-list-timeout=2m \
    comment="BF: SSH stage 1"

# FTP Brute Force
add chain=input protocol=tcp dst-port=21 \
    src-address-list=BF-FTP-STAGE2 \
    action=add-src-to-address-list \
    address-list=BLOCKED-IPS \
    address-list-timeout=24h \
    comment="BF: FTP ban"

add chain=input protocol=tcp dst-port=21 \
    state=new src-address-list=BF-FTP-STAGE1 \
    action=add-src-to-address-list \
    address-list=BF-FTP-STAGE2 \
    address-list-timeout=2m \
    comment="BF: FTP stage 2"

add chain=input protocol=tcp dst-port=21 \
    state=new \
    action=add-src-to-address-list \
    address-list=BF-FTP-STAGE1 \
    address-list-timeout=2m \
    comment="BF: FTP stage 1"

# Winbox Brute Force
add chain=input protocol=tcp dst-port=8291 \
    src-address-list=BF-WB-STAGE2 \
    action=add-src-to-address-list \
    address-list=BLOCKED-IPS \
    address-list-timeout=24h \
    comment="BF: Winbox ban"

add chain=input protocol=tcp dst-port=8291 \
    state=new src-address-list=BF-WB-STAGE1 \
    action=add-src-to-address-list \
    address-list=BF-WB-STAGE2 \
    address-list-timeout=2m \
    comment="BF: Winbox stage 2"

add chain=input protocol=tcp dst-port=8291 \
    state=new \
    action=add-src-to-address-list \
    address-list=BF-WB-STAGE1 \
    address-list-timeout=2m \
    comment="BF: Winbox stage 1"

# Block all BLOCKED-IPS
add chain=input \
    src-address-list=BLOCKED-IPS \
    action=drop \
    log=yes log-prefix="BLOCKED:" \
    comment="Block all banned IPs"
```

---

## 46.4 Rate Limiting per IP

### Per-IP Connection Limit

```routeros
/ip firewall filter

# จำกัด connections ต่อ IP
add chain=forward \
    connection-limit=50,32 \
    action=drop \
    log=yes log-prefix="CONN-LIMIT:" \
    comment="Rate limit: max 50 connections per /32"

# จำกัด new connections per second
add chain=input \
    protocol=tcp state=new \
    limit=20,10:packet \
    action=accept \
    comment="Rate limit: accept 20 new TCP/s"

add chain=input \
    protocol=tcp state=new \
    action=drop \
    comment="Rate limit: drop excess"
```

### Packet Rate Limiting

```routeros
/ip firewall filter

# ICMP rate limiting (กัน ICMP flood)
add chain=input protocol=icmp \
    limit=5,10:packet \
    action=accept \
    comment="Allow ICMP up to 5pps"

add chain=input protocol=icmp \
    action=drop \
    comment="Drop excess ICMP"

# UDP flood protection
add chain=input protocol=udp \
    limit=200,400:packet \
    action=accept \
    comment="Allow UDP up to 200pps"

add chain=input protocol=udp \
    action=add-src-to-address-list \
    address-list=UDP-FLOOD \
    address-list-timeout=5m \
    comment="Track UDP flood sources"
```

---

## 46.5 Dynamic Whitelist

### Whitelist Management Script

```routeros
/system script
add name="whitelist-manager" source={
    # เพิ่ม IP ใน whitelist (รับ argument IP)
    :local ip $1
    :local reason $2
    :local duration $3
    
    :if ([:len $ip] = 0) do={
        :put "Usage: run whitelist-manager ip=x.x.x.x reason='text' duration=24h"
        :error "IP is required"
    }
    
    :if ([:len $duration] = 0) do={ :set duration "24h" }
    :if ([:len $reason] = 0) do={ :set reason "No reason" }
    
    # ตรวจสอบว่าอยู่ใน blacklist ก่อน
    :local inBlacklist [/ip firewall address-list find \
        list=BLOCKED-IPS address=$ip]
    
    :if ([:len $inBlacklist] > 0) do={
        # ลบออกจาก blacklist ก่อน
        /ip firewall address-list remove $inBlacklist
        :log warning ("Whitelist: Removed " . $ip . " from blacklist")
    }
    
    # เพิ่มใน whitelist
    /ip firewall address-list add \
        list=WHITELIST \
        address=$ip \
        timeout=$duration \
        comment=("Reason: " . $reason . " - Added: " . [/system clock get time])
    
    :log info ("Whitelist: Added " . $ip . " for " . $duration . " - " . $reason)
    :put ("Added " . $ip . " to whitelist for " . $duration)
}
```

### Auto-whitelist ด้วยการ authenticate

```routeros
/ip firewall filter

# Whitelist IPs ที่ connect สำเร็จ (based on successful login)
add chain=input protocol=tcp dst-port=22 \
    state=established \
    action=add-src-to-address-list \
    address-list=WHITELIST \
    address-list-timeout=7d \
    comment="Auto-whitelist: Established SSH"
```

---

## 46.6 Threat Intelligence Feeds

### Script ดาวน์โหลด Threat List

```routeros
/system script
add name="update-threat-list" source={
    :local listName "THREAT-INTEL"
    :local feedUrl "http://example.com/threat-ips.txt"
    :local tmpFile "threat-feed.txt"
    
    :log info "Downloading threat intelligence feed..."
    
    # ดาวน์โหลด feed
    :do {
        /tool fetch url=$feedUrl dst-path=$tmpFile
        :log info "Threat feed downloaded"
    } on-error={
        :log error "Failed to download threat feed!"
        :error "Download failed"
    }
    
    # ล้าง list เก่า
    /ip firewall address-list remove [find list=$listName]
    :log info ("Cleared old " . $listName . " entries")
    
    # อ่าน file และเพิ่ม IPs
    :local lineCount 0
    :local content [/file get $tmpFile contents]
    :local lines [:toarray $content]
    
    :foreach line in=$lines do={
        :local ip [:pick $line 0 [:find $line " "]]
        :if ([:len $ip] > 6) do={
            :do {
                /ip firewall address-list add \
                    list=$listName \
                    address=$ip \
                    comment="Threat Intel"
                :set lineCount ($lineCount + 1)
            } on-error={
                :log debug ("Skipped invalid IP: " . $ip)
            }
        }
    }
    
    :log info ("Updated " . $listName . " with " . $lineCount . " entries")
    
    # ลบ temp file
    /file remove $tmpFile
}

# Update ทุก 6 ชั่วโมง
/system scheduler
add name="threat-intel-update" interval=6h \
    on-event="/system script run update-threat-list"
```

---

## 46.7 GeoIP Integration

### Block Countries ด้วย Script

```routeros
/system script
add name="setup-geoip-blocks" source={
    :local blockedCountries {
        "CN";"RU";"KP"
    }
    
    :log info "Setting up GeoIP blocking..."
    
    :foreach country in=$blockedCountries do={
        # ตรวจสอบว่ามี list หรือยัง
        :local listName ("GEOIP-" . $country)
        
        # สร้าง filter rule สำหรับ country นี้ (ถ้ายังไม่มี)
        :local existing [/ip firewall filter find comment=("GeoIP: " . $country)]
        
        :if ([:len $existing] = 0) do={
            /ip firewall filter add \
                chain=input \
                src-address-list=$listName \
                action=drop \
                comment=("GeoIP: " . $country) \
                place-before=[/ip firewall filter find comment="Allow SSH"]
            
            :log info ("GeoIP: Block rule created for " . $country)
        }
    }
}
```

> **Note:** MikroTik RouterOS ไม่มี GeoIP built-in ต้องใช้ address-list ที่ generate จาก MaxMind หรือ ip2location database แล้ว import เข้า router

---

## 46.8 Auto-expiry Rules

### Dynamic Rules ด้วย Timeout

```routeros
# ตัวอย่าง rule ที่หมดอายุอัตโนมัติ
/ip firewall address-list
add list=TEMP-ALLOW address=203.0.113.50 timeout=2h \
    comment="Temporary access for 2 hours"

# Script ตรวจสอบ expired entries
/system script
add name="cleanup-expired-rules" source={
    # ดู address lists ที่หมดอายุ (จะถูกลบโดย RouterOS อัตโนมัติ)
    :local totalEntries [/ip firewall address-list find dynamic=yes]
    :log info ("Dynamic address list entries: " . [:len $totalEntries])
    
    # Log entries ที่ใกล้หมดอายุ (< 10 นาที)
    :foreach entry in=[/ip firewall address-list find dynamic=yes] do={
        :local addr [/ip firewall address-list get $entry address]
        :local list [/ip firewall address-list get $entry list]
        :local timeout [/ip firewall address-list get $entry timeout]
        
        :log debug ("Expiring: " . $list . " - " . $addr . \
            " (timeout: " . $timeout . ")")
    }
}
```

---

## 46.9 Dynamic Firewall Dashboard

### Status Dashboard Script

```routeros
/system script
add name="firewall-dashboard" source={
    :put "╔══════════════════════════════════════╗"
    :put "║    DYNAMIC FIREWALL DASHBOARD         ║"
    :put "╚══════════════════════════════════════╝"
    :put ""
    :put ("Date: " . [/system clock get date] . " " . [/system clock get time])
    :put ""
    
    # Threat Statistics
    :local blocked [/ip firewall address-list find list=BLOCKED-IPS]
    :local whitelist [/ip firewall address-list find list=WHITELIST]
    :local threatIntel [/ip firewall address-list find list=THREAT-INTEL]
    :local sshBf [/ip firewall address-list find list~"SSH"]
    
    :put "=== Address Lists ==="
    :put ("Blocked IPs:    " . [:len $blocked])
    :put ("Whitelisted:    " . [:len $whitelist])
    :put ("Threat Intel:   " . [:len $threatIntel])
    :put ("SSH BF Stage:   " . [:len $sshBf])
    :put ""
    
    # Firewall Statistics
    :put "=== Firewall Rule Hits ==="
    :foreach rule in=[/ip firewall filter find comment~"BF:"] do={
        :local comment [/ip firewall filter get $rule comment]
        :local packets [/ip firewall filter get $rule packets]
        :local bytes [/ip firewall filter get $rule bytes]
        :put ($comment . ": " . $packets . " packets")
    }
    :put ""
    
    # Recent blocks
    :put "=== Recent Blocked IPs (last 5) ==="
    :local recentBlocked [/ip firewall address-list find list=BLOCKED-IPS]
    :local count 0
    :foreach entry in=$recentBlocked do={
        :if ($count < 5) do={
            :local addr [/ip firewall address-list get $entry address]
            :local comment [/ip firewall address-list get $entry comment]
            :put ($addr . " - " . $comment)
            :set count ($count + 1)
        }
    }
}
```

---

## 46.10 Lab: Automated Threat Protection

### Lab Overview

| ระบบ | รายละเอียด |
|------|------------|
| Brute Force Protection | SSH, Winbox, FTP |
| Rate Limiting | Per-IP connection limit |
| Auto-ban | 3-strike rule, 24h ban |
| Whitelist | Manual + auto whitelist |
| Monitoring | Dashboard + email alerts |

### Complete Setup Script

```routeros
/system script
add name="setup-dynamic-firewall" source={
    :put "Setting up Dynamic Firewall Protection..."
    
    # ล้าง rules เก่า
    /ip firewall filter remove [find comment~"DFW:"]
    /ip firewall address-list remove [find comment~"DFW:"]
    
    :put "1. Setting up brute force protection..."
    
    # SSH Brute Force (3 attempts in 2 min = ban 1 day)
    /ip firewall filter
    add chain=input protocol=tcp dst-port=22 \
        src-address-list=DFW-SSH-2 \
        action=add-src-to-address-list \
        address-list=DFW-BLACKLIST \
        address-list-timeout=1d \
        log=yes log-prefix="DFW-BAN-SSH:" \
        comment="DFW: SSH ban"
    
    add chain=input protocol=tcp dst-port=22 \
        state=new src-address-list=DFW-SSH-1 \
        action=add-src-to-address-list \
        address-list=DFW-SSH-2 \
        address-list-timeout=2m \
        comment="DFW: SSH track 2"
    
    add chain=input protocol=tcp dst-port=22 \
        state=new \
        action=add-src-to-address-list \
        address-list=DFW-SSH-1 \
        address-list-timeout=2m \
        comment="DFW: SSH track 1"
    
    :put "2. Setting up blacklist drop rule..."
    
    # Drop blacklisted
    /ip firewall filter add \
        chain=input \
        src-address-list=DFW-BLACKLIST \
        action=drop \
        log=yes log-prefix="DFW-DROP:" \
        comment="DFW: Drop blacklisted"
    
    :put "3. Setting up rate limiting..."
    
    # ICMP rate limit
    /ip firewall filter add \
        chain=input protocol=icmp \
        limit=10,20:packet \
        action=accept \
        comment="DFW: ICMP rate limit accept"
    
    /ip firewall filter add \
        chain=input protocol=icmp \
        action=add-src-to-address-list \
        address-list=DFW-BLACKLIST \
        address-list-timeout=1h \
        comment="DFW: ICMP flood ban"
    
    :put "4. Setting up monitoring scheduler..."
    
    /system scheduler
    add name="DFW-Dashboard" interval=1h \
        on-event="/system script run firewall-dashboard" \
        comment="DFW: Hourly dashboard"
    
    :put "Dynamic Firewall setup complete!"
    :log info "Dynamic Firewall configured successfully"
}

# รัน setup
/system script run setup-dynamic-firewall
```

### Lab Verification

```routeros
# ดู address lists
/ip firewall address-list print where list~"DFW"

# ดู firewall rules
/ip firewall filter print where comment~"DFW"

# รัน dashboard
/system script run firewall-dashboard

# ดู logs
/log print where message~"DFW"
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Dynamic Address Lists | Firewall-driven IP lists |
| Auto-ban Patterns | Multi-stage brute force detection |
| Brute Force Protection | SSH, FTP, Winbox protection |
| Rate Limiting | Per-IP connection/packet limits |
| Whitelist | Manual และ auto whitelist |
| Threat Intelligence | External feed integration |
| GeoIP | Country-based blocking |
| Dashboard | Real-time firewall monitoring |

---

[← Part 45: Port Knocking](part-045-port-knocking.md) | [Part 47: Address List Management →](part-047-address-list.md)
