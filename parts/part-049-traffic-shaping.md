# Part 49: Traffic Shaping บน MikroTik

## บทนำ

Traffic Shaping เป็นการควบคุมและจัดการ traffic flow เพื่อให้ network ทำงานอย่างมีประสิทธิภาพ ต่างจาก Policing ที่เพียงแค่ drop packets เกิน limit, Shaping จะ buffer packets ไว้แล้วส่งออกตาม rate ที่กำหนด

---

## 49.1 Traffic Shaping vs Policing

### ความแตกต่าง

| คุณสมบัติ | Traffic Shaping | Traffic Policing |
|-----------|----------------|-----------------|
| Action เมื่อเกิน limit | Buffer แล้วส่งทีหลัง | Drop ทันที |
| Packet loss | น้อย | มาก |
| Latency | เพิ่มขึ้น (buffering) | คงที่ |
| Memory usage | สูงกว่า | ต่ำกว่า |
| Use case | Client-facing | Network protection |

### เมื่อใช้ Shaping vs Policing

```
Traffic Shaping เหมาะสำหรับ:
├── User bandwidth control
├── Download throttling
└── QoS implementations

Traffic Policing เหมาะสำหรับ:
├── DDoS protection
├── Rate limiting ฝั่ง input
└── Network abuse prevention
```

---

## 49.2 Mangle Marks

### Packet Marks คืออะไร?

Packet Mark เป็น tag ที่ติดกับ packet เพื่อให้ queue/routing รู้ว่าต้องทำอะไรกับ packet นั้น

```routeros
/ip firewall mangle

# Mark HTTP traffic
add chain=forward \
    protocol=tcp dst-port=80 \
    action=mark-packet \
    new-packet-mark=HTTP \
    passthrough=yes \
    comment="Mark HTTP traffic"

# Mark HTTPS traffic
add chain=forward \
    protocol=tcp dst-port=443 \
    action=mark-packet \
    new-packet-mark=HTTPS \
    passthrough=yes \
    comment="Mark HTTPS traffic"

# Mark streaming traffic
add chain=forward \
    dst-address-list=STREAMING-SITES \
    action=mark-packet \
    new-packet-mark=STREAMING \
    passthrough=yes \
    comment="Mark streaming traffic"

# Mark VoIP
add chain=forward \
    protocol=udp dst-port=5060,10000-20000 \
    action=mark-packet \
    new-packet-mark=VOIP \
    passthrough=yes \
    comment="Mark VoIP traffic"
```

---

## 49.3 DSCP Marking

### DSCP คืออะไร?

DSCP (Differentiated Services Code Point) เป็น 6-bit field ใน IP header ที่บอก network devices ว่า packet ควรได้รับ treatment แบบใด

### DSCP Values ที่ใช้บ่อย

| DSCP Class | Value | Use Case |
|-----------|-------|----------|
| EF (Expedited Forwarding) | 46 | VoIP, real-time |
| AF41 | 34 | Video conferencing |
| AF21 | 18 | Business critical |
| CS1 | 8 | Background traffic |
| BE (Best Effort) | 0 | Normal traffic |

```routeros
/ip firewall mangle

# Set DSCP for VoIP (EF = 46)
add chain=forward \
    packet-mark=VOIP \
    action=change-dscp \
    new-dscp=46 \
    passthrough=yes \
    comment="DSCP: VoIP = EF(46)"

# Set DSCP for video (AF41 = 34)
add chain=forward \
    packet-mark=VIDEO-CONF \
    action=change-dscp \
    new-dscp=34 \
    passthrough=yes \
    comment="DSCP: Video = AF41(34)"

# Set DSCP for normal traffic (BE = 0)
add chain=forward \
    action=change-dscp \
    new-dscp=0 \
    passthrough=yes \
    comment="DSCP: Default = BE(0)"
```

---

## 49.4 Application Identification

### สร้าง Address List สำหรับ Applications

```routeros
# Netflix/Streaming sites
/ip firewall address-list
add list=STREAMING-SITES address=44.224.0.0/11 comment="Netflix"
add list=STREAMING-SITES address=52.1.0.0/16 comment="Netflix CDN"
add list=STREAMING-SITES address=35.156.0.0/14 comment="Amazon Prime"
add list=STREAMING-SITES address=216.58.0.0/16 comment="YouTube"

# VoIP services
add list=VOIP-SERVICES address=216.115.69.0/24 comment="Skype"
add list=VOIP-SERVICES address=185.76.145.0/24 comment="Teams"

# Gaming
add list=GAMING-SERVERS address=54.240.0.0/18 comment="Steam"
```

### Layer 7 Application Matching

```routeros
# สร้าง L7 protocol สำหรับ application detection
/ip firewall layer7-protocol
add name="YouTube" regexp="^.*(youtube|googlevideo).*\$"
add name="Netflix" regexp="^.*(netflix|nflxvideo).*\$"
add name="BitTorrent" regexp="^.*(torrent|peer_id|info_hash).*\$"

# ใช้ใน mangle
/ip firewall mangle
add chain=forward \
    layer7-protocol=BitTorrent \
    action=mark-packet \
    new-packet-mark=TORRENT \
    passthrough=yes \
    comment="Mark BitTorrent traffic"

add chain=forward \
    layer7-protocol=YouTube \
    action=mark-packet \
    new-packet-mark=YOUTUBE \
    passthrough=yes \
    comment="Mark YouTube traffic"
```

> **Note:** L7 matching มี performance overhead สูง ควรใช้เฉพาะกรณีจำเป็น และ match ก่อน accept rules อื่นๆ

---

## 49.5 Per-protocol Shaping

### Queue Tree สำหรับ Protocol Shaping

```routeros
# ====================================
# PER-PROTOCOL TRAFFIC SHAPING
# ====================================

# Root queue
/queue tree
add name="SHAPING-ROOT" parent=global max-limit=100M

# VoIP - highest priority, guaranteed bandwidth
add name="SHAPE-VOIP" parent=SHAPING-ROOT \
    max-limit=10M priority=1 \
    packet-mark=VOIP \
    comment="VoIP: 10M guaranteed"

# Video conference
add name="SHAPE-VIDEO" parent=SHAPING-ROOT \
    max-limit=20M priority=2 \
    packet-mark=VIDEO-CONF \
    comment="Video: 20M"

# Business apps
add name="SHAPE-BUSINESS" parent=SHAPING-ROOT \
    max-limit=40M priority=3 \
    packet-mark=BUSINESS-APP \
    comment="Business: 40M"

# Web browsing
add name="SHAPE-WEB" parent=SHAPING-ROOT \
    max-limit=60M priority=4 \
    packet-mark=HTTP \
    comment="Web: 60M"

# Streaming
add name="SHAPE-STREAM" parent=SHAPING-ROOT \
    max-limit=50M priority=5 \
    packet-mark=STREAMING \
    comment="Streaming: 50M"

# BitTorrent - lowest priority
add name="SHAPE-TORRENT" parent=SHAPING-ROOT \
    max-limit=5M priority=8 \
    packet-mark=TORRENT \
    comment="Torrent: 5M limit"
```

---

## 49.6 Burst Handling

### Burst คืออะไร?

Burst ช่วยให้ user ได้ bandwidth สูงกว่า limit ชั่วคราว เหมาะสำหรับ web browsing ที่ต้องการ burst download

```routeros
/queue simple

# Queue พร้อม burst configuration
add name="BURST-USER" target=192.168.1.100/32 \
    max-limit=10M/10M \
    burst-limit=30M/30M \
    burst-threshold=7M/7M \
    burst-time=10s/10s \
    comment="10Mbps with 30Mbps burst for 10s"
```

### Burst Parameters อธิบาย

| Parameter | คำอธิบาย |
|-----------|----------|
| max-limit | Bandwidth สูงสุดปกติ |
| burst-limit | Bandwidth สูงสุดระหว่าง burst |
| burst-threshold | Threshold ที่จะ trigger burst |
| burst-time | ระยะเวลา burst สูงสุด |

### Script คำนวณ Burst Parameters

```routeros
/system script
add name="calculate-burst" source={
    :local maxLimit 10
    :local burstFactor 3
    :local burstDuration 10
    
    :local burstLimit ($maxLimit * $burstFactor)
    :local burstThreshold ($maxLimit * 0.8)
    
    :put ("Max Limit: " . $maxLimit . "Mbps")
    :put ("Burst Limit: " . $burstLimit . "Mbps (x" . $burstFactor . ")")
    :put ("Burst Threshold: " . $burstThreshold . "Mbps (80%)")
    :put ("Burst Duration: " . $burstDuration . "s")
    
    # สร้าง queue ด้วย calculated values
    /queue simple add \
        name="CALC-USER" \
        target=192.168.1.100/32 \
        max-limit=($maxLimit . "M/" . $maxLimit . "M") \
        burst-limit=($burstLimit . "M/" . $burstLimit . "M") \
        burst-threshold=($burstThreshold . "M/" . $burstThreshold . "M") \
        burst-time=($burstDuration . "s/" . $burstDuration . "s")
}
```

---

## 49.7 Fair Queuing

### Fair Queuing ด้วย PCQ

```routeros
# ตั้งค่า Fair PCQ
/queue type
add name="FAIR-DOWNLOAD" kind=pcq \
    pcq-classifier=dst-address \
    pcq-rate=0 \
    pcq-limit=200KiB \
    pcq-total-limit=2000KiB \
    comment="Fair download per destination"

add name="FAIR-UPLOAD" kind=pcq \
    pcq-classifier=src-address \
    pcq-rate=0 \
    pcq-limit=100KiB \
    pcq-total-limit=1000KiB \
    comment="Fair upload per source"

# นำ PCQ ไปใช้
/queue tree
add name="FAIR-QOS" parent=global \
    queue=FAIR-DOWNLOAD \
    packet-mark=all \
    comment="Fair QoS for all traffic"
```

---

## 49.8 Shaping Reports

### Generate Traffic Report

```routeros
/system script
add name="traffic-shaping-report" source={
    :put "============================================"
    :put " TRAFFIC SHAPING REPORT"
    :put "============================================"
    :put ("Date: " . [/system clock get date])
    :put ("Time: " . [/system clock get time])
    :put ""
    
    :put "=== Queue Tree Statistics ==="
    :foreach q in=[/queue tree find] do={
        :local name [/queue tree get $q name]
        :local bytes [/queue tree get $q bytes]
        :local rate [/queue tree get $q rate]
        :local dropped [/queue tree get $q dropped]
        :local maxLimit [/queue tree get $q max-limit]
        
        :put ($name . ":")
        :put ("  Max Limit: " . $maxLimit)
        :put ("  Current Rate: " . $rate)
        :put ("  Total Bytes: " . $bytes)
        :put ("  Dropped: " . $dropped)
        :put ""
    }
    
    :put "=== Top 5 Users (by dropped packets) ==="
    :local count 0
    :foreach q in=[/queue simple find] do={
        :if ($count < 5) do={
            :local name [/queue simple get $q name]
            :local dropped [/queue simple get $q dropped]
            :local target [/queue simple get $q target]
            
            :put ($name . " [" . $target . "]: " . $dropped . " dropped")
            :set count ($count + 1)
        }
    }
}
```

---

## 49.9 Advanced PCQ

### PCQ ด้วย multiple classifiers

```routeros
# PCQ แยก download/upload per IP
/queue type

# Per-IP Download (limit 5M per IP)
add name="PCQ-5M-DOWN" kind=pcq \
    pcq-classifier=dst-address \
    pcq-rate=5M \
    pcq-burst-rate=10M \
    pcq-burst-threshold=4M \
    pcq-burst-time=15s

# Per-IP Upload (limit 2M per IP)
add name="PCQ-2M-UP" kind=pcq \
    pcq-classifier=src-address \
    pcq-rate=2M \
    pcq-burst-rate=4M \
    pcq-burst-threshold=1M \
    pcq-burst-time=10s

# Mangle marks
/ip firewall mangle
add chain=forward in-interface=WAN \
    action=mark-packet new-packet-mark=DOWNLOAD passthrough=no

add chain=forward out-interface=WAN \
    action=mark-packet new-packet-mark=UPLOAD passthrough=no

# Queue Tree ใช้ PCQ
/queue tree
add name="ISP-DOWNLOAD" parent=global \
    queue=PCQ-5M-DOWN \
    packet-mark=DOWNLOAD \
    max-limit=1G

add name="ISP-UPLOAD" parent=global \
    queue=PCQ-2M-UP \
    packet-mark=UPLOAD \
    max-limit=500M
```

---

## 49.10 Lab: Complete Traffic Shaping

### Lab Overview

| รายการ | รายละเอียด |
|--------|------------|
| Total WAN | 100Mbps down / 20Mbps up |
| VoIP | 10Mbps guaranteed, priority 1 |
| Video | 20Mbps, priority 2 |
| Web | 50Mbps, priority 4 |
| Torrent | 5Mbps limit, priority 8 |
| Users | Per-user 10Mbps with burst |

### Complete Setup Script

```routeros
/system script
add name="lab-traffic-shaping-setup" source={
    :put "Setting up Complete Traffic Shaping System..."
    
    # =========================================
    # 1. LAYER 7 PROTOCOL DETECTION
    # =========================================
    /ip firewall layer7-protocol
    add name="L7-BitTorrent" regexp="^.*(BitTorrent|info_hash|peer_id).*\$"
    add name="L7-YouTube" regexp="^.*(youtube|googlevideo).*\$"
    
    # =========================================
    # 2. MANGLE RULES - CLASSIFY TRAFFIC
    # =========================================
    /ip firewall mangle
    
    # VoIP
    add chain=forward protocol=udp dst-port=5060,10000-20000 \
        action=mark-packet new-packet-mark=PM-VOIP passthrough=yes \
        comment="SHAPE: Mark VoIP"
    
    # Video Conference
    add chain=forward protocol=udp dst-port=3478,19302 \
        action=mark-packet new-packet-mark=PM-VIDEO passthrough=yes \
        comment="SHAPE: Mark Video Conf"
    
    # BitTorrent
    add chain=forward layer7-protocol=L7-BitTorrent \
        action=mark-packet new-packet-mark=PM-TORRENT passthrough=yes \
        comment="SHAPE: Mark BitTorrent"
    
    # YouTube/Streaming
    add chain=forward layer7-protocol=L7-YouTube \
        action=mark-packet new-packet-mark=PM-STREAM passthrough=yes \
        comment="SHAPE: Mark YouTube"
    
    # HTTP/HTTPS
    add chain=forward protocol=tcp dst-port=80,443 \
        action=mark-packet new-packet-mark=PM-WEB passthrough=yes \
        comment="SHAPE: Mark Web"
    
    # Mark Download
    add chain=forward in-interface=WAN1 \
        action=mark-packet new-packet-mark=PM-DOWNLOAD passthrough=no \
        comment="SHAPE: Mark Download"
    
    # =========================================
    # 3. PCQ TYPES
    # =========================================
    /queue type
    add name="PCQ-USER-DN" kind=pcq \
        pcq-classifier=dst-address \
        pcq-rate=10M pcq-burst-rate=20M \
        pcq-burst-threshold=8M pcq-burst-time=10s
    
    add name="PCQ-USER-UP" kind=pcq \
        pcq-classifier=src-address \
        pcq-rate=2M pcq-burst-rate=5M \
        pcq-burst-threshold=1M pcq-burst-time=10s
    
    # =========================================
    # 4. QUEUE TREE
    # =========================================
    /queue tree
    add name="TOTAL" parent=global max-limit=100M
    
    # Priority classes
    add name="Q-VOIP" parent=TOTAL max-limit=10M priority=1 \
        packet-mark=PM-VOIP
    add name="Q-VIDEO" parent=TOTAL max-limit=20M priority=2 \
        packet-mark=PM-VIDEO
    add name="Q-WEB" parent=TOTAL max-limit=50M priority=4 \
        packet-mark=PM-WEB
    add name="Q-STREAM" parent=TOTAL max-limit=40M priority=5 \
        packet-mark=PM-STREAM
    add name="Q-TORRENT" parent=TOTAL max-limit=5M priority=8 \
        packet-mark=PM-TORRENT
    
    # Per-user PCQ
    add name="Q-USERS" parent=TOTAL max-limit=80M \
        queue=PCQ-USER-DN packet-mark=PM-DOWNLOAD
    
    :put "Traffic Shaping setup complete!"
    :log info "Complete traffic shaping system configured"
}

/system script run lab-traffic-shaping-setup
```

### Lab Verification

```routeros
# ดู mangle rules
/ip firewall mangle print where comment~"SHAPE:"

# ดู queue tree
/queue tree print

# Monitor real-time
/queue tree monitor

# ดู shaping stats
/queue tree print stats
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Shaping vs Policing | ความแตกต่างและการใช้งาน |
| Mangle Marks | Packet classification |
| DSCP | QoS marking standards |
| Application ID | L7 protocol detection |
| Per-protocol | Priority-based shaping |
| Burst | Burst rate configuration |
| Fair Queuing | PCQ-based fairness |
| Reports | Traffic statistics |
| Advanced PCQ | Per-IP limits |

---

[← Part 48: Queue Management](part-048-queue-management.md) | [Part 50: Captive Portal →](part-050-captive-portal.md)
