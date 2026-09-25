# Part 35: Bandwidth Management ใน RouterOS

## บทนำ

Bandwidth management เป็นหัวใจของ network QoS (Quality of Service) RouterOS มี queuing system ที่ทรงพลังช่วยให้จัดสรร bandwidth ได้อย่างยุติธรรมและมีประสิทธิภาพ

---

## 35.1 Simple Queue

### พื้นฐาน Simple Queue

```routeros
# Simple queue สำหรับ single IP
/queue simple add \
    name=client-001 \
    target=192.168.1.10/32 \
    max-limit=10M/5M \
    comment="Client 001 - 10Mbps down / 5Mbps up"

# Simple queue สำหรับ subnet
/queue simple add \
    name=lan-queue \
    target=192.168.1.0/24 \
    max-limit=100M/50M \
    comment="LAN network total"

# Queue พร้อม burst
/queue simple add \
    name=client-burst \
    target=192.168.1.20/32 \
    max-limit=10M/5M \
    burst-limit=20M/10M \
    burst-threshold=8M/4M \
    burst-time=10s/10s \
    comment="Client with burst"

# ดู queues
/queue simple print

# Script เพิ่ม queue สำหรับ client ใหม่
:local addClientQueue do={
    :local name $1
    :local ip $2
    :local downLimit $3
    :local upLimit $4
    :local comment $5
    
    # ลบ queue เก่าถ้ามี
    /queue simple remove [find target=($ip . "/32")]
    
    # สร้าง queue ใหม่
    /queue simple add \
        name=$name \
        target=($ip . "/32") \
        max-limit=($downLimit . "/" . $upLimit) \
        comment=$comment
    
    :put "Queue added: $name - $ip ($downLimit down / $upLimit up)"
}

[$addClientQueue "user-001" "192.168.1.100" "20M" "10M" "Premium user"]
[$addClientQueue "user-002" "192.168.1.101" "5M" "2M" "Basic user"]
```

### Bulk Queue Management

```routeros
# สร้าง queues จาก DHCP leases
:local createQueuesFromDHCP do={
    :local profile $1
    :local downLimit $2
    :local upLimit $3
    
    :foreach lease in=[/ip dhcp-server lease find status=bound] do={
        :local ip [/ip dhcp-server lease get $lease address]
        :local mac [/ip dhcp-server lease get $lease mac-address]
        :local hostname [/ip dhcp-server lease get $lease host-name]
        
        # ตรวจสอบว่ามี queue แล้วหรือยัง
        :if ([:len [/queue simple find target=($ip . "/32")]] = 0) do={
            /queue simple add \
                name=("dhcp-" . $mac) \
                target=($ip . "/32") \
                max-limit=($downLimit . "/" . $upLimit) \
                comment=($hostname . " (DHCP)")
            :put "Created queue for: $ip ($hostname)"
        }
    }
}

[$createQueuesFromDHCP "basic" "10M" "5M"]
```

---

## 35.2 Queue Tree

### Queue Tree สำหรับ Complex Traffic Shaping

```routeros
# Queue Tree ต้องมี parent-child relationship

# ก่อนอื่น ต้องมี mangle marks

# 1. Mark traffic
/ip firewall mangle add \
    chain=prerouting \
    in-interface=ether2 \
    action=mark-packet \
    new-packet-mark=download \
    passthrough=no \
    comment="Mark download traffic"

/ip firewall mangle add \
    chain=postrouting \
    out-interface=ether1 \
    action=mark-packet \
    new-packet-mark=upload \
    passthrough=no \
    comment="Mark upload traffic"

# 2. สร้าง Queue Tree
# Parent queue
/queue tree add \
    name=download-main \
    parent=ether2 \
    max-limit=100M \
    comment="Total download bandwidth"

/queue tree add \
    name=upload-main \
    parent=ether1 \
    max-limit=50M \
    comment="Total upload bandwidth"

# Child queues สำหรับ traffic classes
/queue tree add \
    name=voip-dl \
    parent=download-main \
    packet-mark=voip-traffic \
    priority=1 \
    max-limit=10M \
    comment="VoIP - high priority"

/queue tree add \
    name=web-dl \
    parent=download-main \
    packet-mark=web-traffic \
    priority=5 \
    max-limit=80M \
    comment="Web - medium priority"

/queue tree add \
    name=bulk-dl \
    parent=download-main \
    packet-mark=bulk-traffic \
    priority=8 \
    max-limit=20M \
    comment="Downloads - low priority"

# Mangle rules สำหรับ VoIP
/ip firewall mangle add \
    chain=forward \
    protocol=udp \
    dst-port=5060,10000-20000 \
    action=mark-packet \
    new-packet-mark=voip-traffic \
    passthrough=no

# Mangle rules สำหรับ web
/ip firewall mangle add \
    chain=forward \
    protocol=tcp \
    dst-port=80,443 \
    action=mark-packet \
    new-packet-mark=web-traffic \
    passthrough=no

# Mangle rules สำหรับ bulk
/ip firewall mangle add \
    chain=forward \
    action=mark-packet \
    new-packet-mark=bulk-traffic \
    passthrough=no \
    comment="Default - bulk"
```

---

## 35.3 PCQ (Per Connection Queue)

```routeros
# PCQ ทำให้แต่ละ connection/user ได้ bandwidth เท่าๆ กัน

# สร้าง PCQ queues
/queue type add \
    name=pcq-download \
    kind=pcq \
    pcq-rate=0 \
    pcq-limit=50KiB \
    pcq-classifier=dst-address

/queue type add \
    name=pcq-upload \
    kind=pcq \
    pcq-rate=0 \
    pcq-limit=50KiB \
    pcq-classifier=src-address

# Queue tree ใช้ PCQ
/queue tree add \
    name=pcq-dl \
    parent=ether2 \
    queue=pcq-download \
    max-limit=100M

/queue tree add \
    name=pcq-ul \
    parent=ether1 \
    queue=pcq-upload \
    max-limit=50M

# PCQ บน Simple Queue
/queue simple add \
    name=shared-internet \
    target=192.168.1.0/24 \
    max-limit=100M/50M \
    queue=pcq-download/pcq-upload \
    comment="PCQ - fair share for all users"
```

---

## 35.4 Burst

```routeros
# Burst ช่วยให้ users ได้ bandwidth สูงกว่า max ชั่วคราว

# Simple Burst
/queue simple add \
    name=burst-test \
    target=192.168.1.50/32 \
    max-limit=10M/5M \
    burst-limit=30M/15M \       # burst ไม่เกิน 30M/15M
    burst-threshold=8M/4M \     # burst เมื่อใช้ต่ำกว่า 8M/4M
    burst-time=10s/10s \        # burst time window
    comment="User with 30M burst for 10 seconds"

# อธิบาย Burst:
# ถ้า average usage (ใน burst-time) ต่ำกว่า burst-threshold 
# user จะได้ burst-limit แทน max-limit
# ถ้า average สูงกว่า burst-threshold จะ throttle กลับ max-limit

# Burst โดยไม่มี max-limit (burst แล้วหยุด)
/queue simple add \
    name=temporary-burst \
    target=192.168.1.60/32 \
    max-limit=5M/2M \
    burst-limit=50M/25M \
    burst-threshold=4M/2M \
    burst-time=30s/30s
```

---

## 35.5 Priority Queuing

```routeros
# Priority 1 = highest, 8 = lowest

# Per-user priorities ใน Queue Tree
/queue tree add name=vip-users parent=ether2 priority=1 max-limit=50M comment="VIP users"
/queue tree add name=regular-users parent=ether2 priority=5 max-limit=40M comment="Regular users"
/queue tree add name=guest-users parent=ether2 priority=8 max-limit=10M comment="Guest users"

# Mangle marks สำหรับแต่ละ group
/ip firewall mangle add \
    chain=forward \
    src-address-list=vip-users \
    action=mark-packet \
    new-packet-mark=vip-traffic

/ip firewall mangle add \
    chain=forward \
    src-address-list=regular-users \
    action=mark-packet \
    new-packet-mark=regular-traffic

# Address lists สำหรับ user groups
/ip firewall address-list add list=vip-users address=192.168.1.10
/ip firewall address-list add list=vip-users address=192.168.1.11
/ip firewall address-list add list=regular-users address=192.168.1.0/24

# Dynamic priority ตาม time of day
/system script add name="adjust-priority" source="
    :local hour [:toint [:pick [:tostr [/system clock get time]] 0 2]]
    
    :if (\$hour >= 9 && \$hour < 18) do={
        # Business hours: ให้ priority แก่ work traffic
        /ip firewall mangle set [find chain=forward dst-port=80,443 protocol=tcp] \
            new-packet-mark=priority-traffic
    } else={
        # Off hours: normal priority
        /ip firewall mangle set [find chain=forward dst-port=80,443 protocol=tcp] \
            new-packet-mark=regular-traffic
    }
"
```

---

## 35.6 Bandwidth Monitoring Scripts

```routeros
# Monitor bandwidth usage per interface
:local monitorBandwidth do={
    :local interfaces {"ether1"; "ether2"; "ether3"}
    
    :foreach ifaceName in=$interfaces do={
        :local id [/interface find name=$ifaceName]
        :if ([:len $id] > 0) do={
            :local rxBytes [/interface get $id rx-byte]
            :local txBytes [/interface get $id tx-byte]
            :local rxRate [/interface get $id rx-bits-per-second]
            :local txRate [/interface get $id tx-bits-per-second]
            
            :put "$ifaceName: RX=" . ($rxRate/1000000) . "Mbps TX=" . ($txRate/1000000) . "Mbps"
        }
    }
}

[$monitorBandwidth]

# Monitor specific interface continuously (manual - use tool traffic-monitor)
/tool traffic-monitor

# Collect bandwidth stats ทุกชั่วโมง
:global bwStats {}
:local collectBWStats do={
    :global bwStats
    :local timestamp [/system clock get time]
    :local stats {}
    
    :foreach iface in=[/interface find disabled=no type=ether] do={
        :local name [/interface get $iface name]
        :local rxBytes [/interface get $iface rx-byte]
        :local txBytes [/interface get $iface tx-byte]
        :set stats ($stats , {$name; $rxBytes; $txBytes})
    }
    
    :set bwStats ($bwStats , {$timestamp; $stats})
    
    # Keep only 24 hours (24 snapshots)
    :if ([:len $bwStats] > 24) do={
        :set bwStats [:pick $bwStats 1 [:len $bwStats]]
    }
    :put "BW stats collected at $timestamp"
}

/system scheduler add \
    name="bw-stats" \
    interval=1h \
    on-event="[$collectBWStats]"
```

---

## 35.7 Dynamic Bandwidth Allocation

```routeros
# ปรับ bandwidth ตาม time of day
/system script add name="dynamic-bandwidth" source="
    :local hour [:toint [:pick [:tostr [/system clock get time]] 0 2]]
    :local peakLimit \"50M\"
    :local offPeakLimit \"100M\"
    :local currentLimit \$offPeakLimit
    
    # Peak hours: 8AM-10PM
    :if (\$hour >= 8 && \$hour < 22) do={
        :set currentLimit \$peakLimit
    }
    
    # อัพเดท main queue limit
    /queue simple set [find name=main-queue] max-limit=(\$currentLimit . \"/\" . \$currentLimit)
    :log info (\"Bandwidth adjusted: \" . \$currentLimit . \" (hour=\" . \$hour . \")\")
"

/system scheduler add \
    name="dynamic-bw" \
    interval=1h \
    on-event="/system script run dynamic-bandwidth"

# ปรับ bandwidth ตาม network load
/system script add name="load-based-bandwidth" source="
    :local cpu [/system resource get cpu-load]
    :local activeSessions [:len [/ppp active find]]
    
    # ถ้า sessions เยอะ และ CPU สูง - limit bandwidth
    :if (\$activeSessions > 100 && \$cpu > 80) do={
        /queue simple set [find name=shared-bandwidth] max-limit=\"80M/40M\"
        :log info \"Bandwidth reduced due to high load\"
    } else={
        /queue simple set [find name=shared-bandwidth] max-limit=\"100M/50M\"
    }
"
```

---

## 35.8 Fair Usage Policy Scripts

```routeros
# FUP - ลด bandwidth เมื่อ usage เกิน limit
/system script add name="fup-policy" source="
    :local fupLimit 50000000000  # 50GB
    :local fupSpeed \"2M/1M\"
    :local normalSpeed \"20M/10M\"
    
    :foreach queue in=[/queue simple find] do={
        :local name [/queue simple get \$queue name]
        :local target [/queue simple get \$queue target]
        :local bytesIn [/queue simple get \$queue bytes-in]
        
        :if (\$bytesIn > \$fupLimit) do={
            # Exceeded FUP
            :local currentLimit [/queue simple get \$queue max-limit]
            :if (\$currentLimit != \$fupSpeed) do={
                /queue simple set \$queue max-limit=\$fupSpeed
                :log info (\"FUP applied: \" . \$name . \" (\" . \$bytesIn . \" bytes used)\")
            }
        } else={
            # Within FUP
            :local currentLimit [/queue simple get \$queue max-limit]
            :if (\$currentLimit = \$fupSpeed) do={
                /queue simple set \$queue max-limit=\$normalSpeed
                :log info (\"FUP lifted: \" . \$name)
            }
        }
    }
" comment="Fair Usage Policy enforcement"

# Reset FUP counters รายเดือน
/system script add name="fup-reset" source="
    # ล้าง byte counters สำหรับทุก queues
    /queue simple reset-counters [find]
    :log info \"FUP counters reset\"
    :put \"All queue counters reset for new month\"
" comment="Monthly FUP counter reset"

# Schedule reset วันที่ 1 ของทุกเดือน
/system scheduler add \
    name="monthly-fup-reset" \
    start-time=00:00:00 \
    interval=1d \
    on-event="
        :local day [:pick [:tostr [/system clock get date]] 4 6]
        :if (\$day = \"01\") do={
            /system script run fup-reset
        }
    "
```

---

## 35.9 Reports Generation

```routeros
# Generate bandwidth usage report
/system script add name="bw-report" source="
    :local report \"=== Bandwidth Usage Report ===\\r\\n\"
    :set report (\$report . \"Date: \" . [/system clock get date] . \"\\r\\n\\r\\n\")
    
    :set report (\$report . \"Queue Statistics:\\r\\n\")
    :set report (\$report . \"Name              Target           Bytes In      Bytes Out\\r\\n\")
    :set report (\$report . \"----------------------------------------------------------------\\r\\n\")
    
    :foreach q in=[/queue simple find] do={
        :local name [/queue simple get \$q name]
        :local target [/queue simple get \$q target]
        :local bytesIn [/queue simple get \$q bytes-in]
        :local bytesOut [/queue simple get \$q bytes-out]
        
        # Format bytes to MB
        :local inMB (\$bytesIn / 1048576)
        :local outMB (\$bytesOut / 1048576)
        
        :set report (\$report . \$name . \"  \" . \$target . \"  \" . \$inMB . \"MB  \" . \$outMB . \"MB\\r\\n\")
    }
    
    # Top users
    :set report (\$report . \"\\r\\nInterface Summary:\\r\\n\")
    :foreach iface in=[/interface find disabled=no type=ether] do={
        :local name [/interface get \$iface name]
        :local rxGB ([/interface get \$iface rx-byte] / 1073741824)
        :local txGB ([/interface get \$iface tx-byte] / 1073741824)
        :set report (\$report . \$name . \": RX=\" . \$rxGB . \"GB TX=\" . \$txGB . \"GB\\r\\n\")
    }
    
    /tool e-mail send to=\"admin@company.com\" \
        subject=\"Bandwidth Usage Report\" \
        body=\$report
    
    :put \$report
" comment="Generate bandwidth report"

/system scheduler add \
    name="bw-report" \
    start-time=08:00:00 \
    interval=1d \
    on-event="/system script run bw-report"
```

---

## Lab 35: Complete QoS Setup

### Solution

```routeros
# Complete QoS Setup Lab
# =======================

# Scenario: ISP network with 100Mbps WAN
# ether1 = WAN (upstream)
# ether2 = LAN (clients)

# --- Step 1: Traffic Classification (Mangle) ---

# VoIP Traffic
/ip firewall mangle add chain=prerouting protocol=udp dst-port=5060 \
    action=mark-connection new-connection-mark=voip-conn passthrough=yes
/ip firewall mangle add chain=prerouting connection-mark=voip-conn \
    action=mark-packet new-packet-mark=voip passthrough=no

# Streaming
/ip firewall mangle add chain=prerouting protocol=tcp dst-port=1935 \
    action=mark-packet new-packet-mark=streaming passthrough=no

# Web (HTTP/HTTPS)
/ip firewall mangle add chain=prerouting protocol=tcp dst-port=80,443 \
    action=mark-packet new-packet-mark=web passthrough=no

# DNS
/ip firewall mangle add chain=prerouting protocol=udp dst-port=53 \
    action=mark-packet new-packet-mark=dns passthrough=no

# Default/Bulk
/ip firewall mangle add chain=prerouting \
    action=mark-packet new-packet-mark=bulk passthrough=no

# --- Step 2: PCQ Types ---
/queue type add name=pcq-dl kind=pcq pcq-classifier=dst-address pcq-rate=5M
/queue type add name=pcq-ul kind=pcq pcq-classifier=src-address pcq-rate=2M

# --- Step 3: Queue Tree ---
# Main parent queues
/queue tree add name=WAN-DL parent=ether2 max-limit=95M comment="Total download 95Mbps"
/queue tree add name=WAN-UL parent=ether1 max-limit=45M comment="Total upload 45Mbps"

# Priority queues under download
/queue tree add name=voip-dl parent=WAN-DL packet-mark=voip priority=1 max-limit=10M
/queue tree add name=dns-dl parent=WAN-DL packet-mark=dns priority=2 max-limit=5M
/queue tree add name=web-dl parent=WAN-DL packet-mark=web priority=4 max-limit=80M queue=pcq-dl
/queue tree add name=streaming-dl parent=WAN-DL packet-mark=streaming priority=6 max-limit=30M
/queue tree add name=bulk-dl parent=WAN-DL packet-mark=bulk priority=8 max-limit=20M queue=pcq-dl

# Priority queues under upload
/queue tree add name=voip-ul parent=WAN-UL packet-mark=voip priority=1 max-limit=5M
/queue tree add name=web-ul parent=WAN-UL packet-mark=web priority=4 max-limit=30M queue=pcq-ul
/queue tree add name=bulk-ul parent=WAN-UL packet-mark=bulk priority=8 max-limit=10M queue=pcq-ul

# --- Step 4: Per-user queues for VIPs ---
/queue simple add name=vip-001 target=192.168.1.10/32 \
    max-limit=50M/20M burst-limit=100M/40M burst-threshold=40M/15M burst-time=30s/30s \
    comment="VIP user with burst"

/queue simple add name=vip-002 target=192.168.1.11/32 \
    max-limit=50M/20M \
    comment="VIP user"

# --- Step 5: Monitoring ---
/system script add name="qos-monitor" source="
    :put \"=== QoS Queue Status ===\"
    /queue tree print stats
    :put \"\\n=== Top 5 Users ===\"
    /queue simple print stats
" comment="QoS monitoring"

:put "Complete QoS setup done!"
:put "Traffic classes: VoIP(P1) > DNS(P2) > Web(P4) > Streaming(P6) > Bulk(P8)"
:put "VIP users with burst configured"
:put "PCQ ensures fair share per user"
```

---

## Summary ของ Part 35

| Queue Type | ใช้เมื่อ | ข้อดี |
|-----------|---------|------|
| Simple Queue | Per-IP bandwidth | ง่าย, ชัดเจน |
| Queue Tree | Complex shaping | Flexible, powerful |
| PCQ | Fair share | ยุติธรรม |
| Burst | Temporary boost | Good UX |

> **Tip:** ใช้ Simple Queue สำหรับ per-user limits, Queue Tree สำหรับ traffic classes

> **Warning:** Queue tree ต้องการ mangle marks ที่ถูกต้อง ถ้า mark ไม่มีจะ queue ไม่ทำงาน

> **Best Practice:** ทดสอบ QoS ด้วย iperf3 หรือ bandwidth test tool ก่อน production

---

[← Part 34: VPN Automation](part-034-vpn-automation.md) | [Part 36: User Manager →](part-036-user-manager.md)
