# Part 43: Load Balancing บน MikroTik

## บทนำ

Load Balancing ช่วยกระจาย traffic ออกหลาย WAN link เพื่อเพิ่ม throughput และลด bottleneck MikroTik รองรับหลาย method ในการทำ load balancing ทั้ง ECMP, PCC, nth, และ Mangle-based

---

## 43.1 Load Balancing Methods

### การเปรียบเทียบ Methods

| Method | Algorithm | Session Affinity | Complexity | Use Case |
|--------|-----------|-----------------|------------|----------|
| ECMP | Hash-based | Partial | Low | Simple multi-WAN |
| PCC | Per-connection | Yes | Medium | ISP-grade |
| nth | Round-robin | No | Low | Simple balancing |
| Mangle-based | Custom | Yes | High | Complex scenarios |

### เลือก Method ที่เหมาะสม

```
มีผู้ใช้จำนวนมาก + ต้องการ session affinity
    └── ใช้ PCC (Per Connection Classifier)

ต้องการความเรียบง่าย + ไม่สนใจ session
    └── ใช้ ECMP หรือ nth

ต้องการ custom logic ตาม protocol/user
    └── ใช้ Mangle-based
```

---

## 43.2 ECMP Routing

### Equal-Cost Multi-Path คืออะไร?

ECMP (Equal-Cost Multi-Path) คือการมี multiple routes ไปยัง destination เดียวกัน โดย RouterOS จะ distribute traffic ระหว่าง paths

```routeros
# ตั้งค่า ECMP - เพิ่ม default routes หลายเส้น
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 \
    comment="WAN1 Default Route"
add dst-address=0.0.0.0/0 gateway=198.51.100.1 \
    comment="WAN2 Default Route"
```

### ตรวจสอบ ECMP Routes

```routeros
# ดู routing table
/ip route print where dst-address=0.0.0.0/0

# ดู ECMP statistics
/ip route print detail where dst-address=0.0.0.0/0
```

> **Note:** ECMP ใน RouterOS จะ hash ตาม src/dst IP ทำให้ connection เดียวกันไปทาง WAN เดิม แต่ connections ต่างกันอาจไปคนละ WAN

---

## 43.3 PCC (Per Connection Classifier)

### PCC คืออะไร?

PCC แบ่ง traffic ตาม hash ของ src-address, dst-address, src-port, หรือ dst-port ทำให้ connection เดียวกันไปทาง WAN เดิม (session affinity)

### PCC สำหรับ Dual WAN

```routeros
# =========================================
# PCC DUAL-WAN LOAD BALANCING
# =========================================

# --- IP Addresses ---
# WAN1: 203.0.113.10/24 gateway 203.0.113.1
# WAN2: 198.51.100.10/24 gateway 198.51.100.1
# LAN: 192.168.1.0/24

# Step 1: Marking connections
/ip firewall mangle

# Mark connections ที่มาจาก WAN1
add chain=input in-interface=WAN1 action=mark-connection \
    new-connection-mark=WAN1-CONN passthrough=yes \
    comment="Mark connections from WAN1"

# Mark connections ที่มาจาก WAN2
add chain=input in-interface=WAN2 action=mark-connection \
    new-connection-mark=WAN2-CONN passthrough=yes \
    comment="Mark connections from WAN2"

# Step 2: PCC-based connection marking
# Split outgoing traffic using PCC
add chain=prerouting in-interface=LAN \
    per-connection-classifier=src-address:2/0 \
    action=mark-connection new-connection-mark=WAN1-CONN \
    passthrough=yes \
    comment="PCC: First half to WAN1"

add chain=prerouting in-interface=LAN \
    per-connection-classifier=src-address:2/1 \
    action=mark-connection new-connection-mark=WAN2-CONN \
    passthrough=yes \
    comment="PCC: Second half to WAN2"

# Step 3: Mark routing
add chain=prerouting connection-mark=WAN1-CONN \
    action=mark-routing new-routing-mark=TO-WAN1 \
    passthrough=yes

add chain=prerouting connection-mark=WAN2-CONN \
    action=mark-routing new-routing-mark=TO-WAN2 \
    passthrough=yes

# Step 4: Routing tables
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 \
    routing-mark=TO-WAN1 \
    comment="WAN1 route for marked traffic"

add dst-address=0.0.0.0/0 gateway=198.51.100.1 \
    routing-mark=TO-WAN2 \
    comment="WAN2 route for marked traffic"

# Default fallback
add dst-address=0.0.0.0/0 gateway=203.0.113.1 \
    comment="Default fallback to WAN1"

# Step 5: NAT สำหรับทั้งสอง WAN
/ip firewall nat
add chain=srcnat out-interface=WAN1 \
    action=masquerade \
    comment="NAT for WAN1"

add chain=srcnat out-interface=WAN2 \
    action=masquerade \
    comment="NAT for WAN2"
```

### PCC แบบ Weighted (70/30 split)

```routeros
# ใช้ modulo ที่ต่างกัน เช่น 10 เพื่อ 70/30 split
/ip firewall mangle

# 7 ใน 10 connections ไป WAN1
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/0 \
    action=mark-connection new-connection-mark=WAN1-CONN passthrough=yes
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/1 \
    action=mark-connection new-connection-mark=WAN1-CONN passthrough=yes
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/2 \
    action=mark-connection new-connection-mark=WAN1-CONN passthrough=yes
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/3 \
    action=mark-connection new-connection-mark=WAN1-CONN passthrough=yes
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/4 \
    action=mark-connection new-connection-mark=WAN1-CONN passthrough=yes
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/5 \
    action=mark-connection new-connection-mark=WAN1-CONN passthrough=yes
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/6 \
    action=mark-connection new-connection-mark=WAN1-CONN passthrough=yes

# 3 ใน 10 connections ไป WAN2
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/7 \
    action=mark-connection new-connection-mark=WAN2-CONN passthrough=yes
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/8 \
    action=mark-connection new-connection-mark=WAN2-CONN passthrough=yes
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:10/9 \
    action=mark-connection new-connection-mark=WAN2-CONN passthrough=yes
```

---

## 43.4 nth-based Load Balancing

### nth (Every nth Packet)

nth ทำ round-robin โดยส่งทุก N packet ไปทาง WAN ที่กำหนด เหมาะสำหรับ simple balancing

```routeros
/ip firewall mangle

# ทุก 2 connections: 1 ไป WAN1, 1 ไป WAN2
add chain=prerouting in-interface=LAN \
    nth=2,1 action=mark-connection \
    new-connection-mark=WAN1-CONN passthrough=yes \
    comment="nth: Connection 1 of 2 to WAN1"

add chain=prerouting in-interface=LAN \
    nth=2,2 action=mark-connection \
    new-connection-mark=WAN2-CONN passthrough=yes \
    comment="nth: Connection 2 of 2 to WAN2"
```

> **Warning:** nth load balancing ไม่มี session affinity ดังนั้นอาจทำให้ connection ถูก reset บาง applications เช่น banking websites

---

## 43.5 Mangle-based Load Balancing

### Custom Logic ด้วย Mangle

```routeros
# ========================================
# ADVANCED MANGLE-BASED LOAD BALANCING
# ========================================

/ip firewall mangle

# 1. Mark streaming traffic ไป WAN2 (bandwidth มากกว่า)
add chain=prerouting in-interface=LAN \
    dst-port=80,443 protocol=tcp \
    dst-address-list=STREAMING-SITES \
    action=mark-routing new-routing-mark=TO-WAN2 \
    passthrough=no \
    comment="Streaming to WAN2"

# 2. Mark video conference ไป WAN1 (latency ต่ำกว่า)
add chain=prerouting in-interface=LAN \
    dst-port=3478,19302 protocol=udp \
    action=mark-routing new-routing-mark=TO-WAN1 \
    passthrough=no \
    comment="Video conference to WAN1"

# 3. Default PCC สำหรับ traffic ที่เหลือ
add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:2/0 \
    action=mark-connection new-connection-mark=WAN1-CONN \
    passthrough=yes

add chain=prerouting in-interface=LAN \
    per-connection-classifier=both-addresses:2/1 \
    action=mark-connection new-connection-mark=WAN2-CONN \
    passthrough=yes

add chain=prerouting connection-mark=WAN1-CONN \
    action=mark-routing new-routing-mark=TO-WAN1 passthrough=yes

add chain=prerouting connection-mark=WAN2-CONN \
    action=mark-routing new-routing-mark=TO-WAN2 passthrough=yes
```

---

## 43.6 Monitoring Load Balance

### Script ตรวจสอบ WAN Traffic

```routeros
/system script
add name="lb-monitor" source={
    :put "=== Load Balance Traffic Monitor ==="
    :put ""
    
    :local wan1Iface "WAN1"
    :local wan2Iface "WAN2"
    
    # ดู traffic statistics
    :local wan1Stats [/interface get $wan1Iface]
    :local wan2Stats [/interface get $wan2Iface]
    
    :local wan1RxBytes [/interface monitor-traffic $wan1Iface once as-value]
    :local wan2RxBytes [/interface monitor-traffic $wan2Iface once as-value]
    
    :put ("WAN1 RX: " . ($wan1RxBytes->"rx-bits-per-second") . " bps")
    :put ("WAN1 TX: " . ($wan1RxBytes->"tx-bits-per-second") . " bps")
    :put ""
    :put ("WAN2 RX: " . ($wan2RxBytes->"rx-bits-per-second") . " bps")
    :put ("WAN2 TX: " . ($wan2RxBytes->"tx-bits-per-second") . " bps")
    
    # ตรวจสอบ connection marks
    :local wan1Conns [/ip firewall connection find connection-mark=WAN1-CONN]
    :local wan2Conns [/ip firewall connection find connection-mark=WAN2-CONN]
    
    :put ""
    :put ("Active Connections - WAN1: " . [:len $wan1Conns])
    :put ("Active Connections - WAN2: " . [:len $wan2Conns])
}
```

### Log WAN Statistics

```routeros
/system script
add name="lb-log-stats" source={
    :local timestamp [/system clock get time]
    :local date [/system clock get date]
    
    :foreach wanIface in={"WAN1"; "WAN2"} do={
        :local stats [/interface monitor-traffic $wanIface once as-value]
        :local rxBps ($stats->"rx-bits-per-second")
        :local txBps ($stats->"tx-bits-per-second")
        
        :log info ("LB-STAT|" . $date . "|" . $timestamp . \
            "|" . $wanIface . "|rx=" . $rxBps . "|tx=" . $txBps)
    }
}

/system scheduler
add name="lb-stats" interval=1m \
    on-event="/system script run lb-log-stats"
```

---

## 43.7 Failover Integration

### Load Balance with Failover

```routeros
/system script
add name="lb-with-failover" source={
    :local wan1Up false
    :local wan2Up false
    :local pingTarget "8.8.8.8"
    
    # ตรวจสอบ WAN1
    :local wan1Ping [/ping $pingTarget routing-table=TO-WAN1 count=3 \
        interval=500ms as-value]
    :if (($wan1Ping->"received") > 0) do={
        :set wan1Up true
    }
    
    # ตรวจสอบ WAN2
    :local wan2Ping [/ping $pingTarget routing-table=TO-WAN2 count=3 \
        interval=500ms as-value]
    :if (($wan2Ping->"received") > 0) do={
        :set wan2Up true
    }
    
    # ปรับ load balance ตาม WAN availability
    :if ($wan1Up && $wan2Up) do={
        # ทั้งสอง WAN ใช้งานได้ - enable PCC load balancing
        /ip firewall mangle set [find comment="PCC-WAN1"] disabled=no
        /ip firewall mangle set [find comment="PCC-WAN2"] disabled=no
        :log info "LB: Both WANs up, load balancing active"
        
    } else if ($wan1Up && !$wan2Up) do={
        # เฉพาะ WAN1 ใช้งานได้
        /ip firewall mangle set [find comment="PCC-WAN2"] disabled=yes
        :log warning "LB: WAN2 down, all traffic to WAN1"
        
    } else if (!$wan1Up && $wan2Up) do={
        # เฉพาะ WAN2 ใช้งานได้
        /ip firewall mangle set [find comment="PCC-WAN1"] disabled=yes
        :log warning "LB: WAN1 down, all traffic to WAN2"
        
    } else={
        # ทั้งสอง WAN ไม่ได้ใช้งาน
        :log error "LB: Both WANs are down!"
    }
}

/system scheduler
add name="lb-failover-check" interval=30s \
    on-event="/system script run lb-with-failover"
```

---

## 43.8 Per-protocol Load Balancing

### แยก Traffic ตาม Protocol

```routeros
/ip firewall mangle

# HTTP/HTTPS ไป WAN1
add chain=prerouting in-interface=LAN \
    protocol=tcp dst-port=80,443 \
    action=mark-routing new-routing-mark=TO-WAN1 \
    passthrough=no \
    comment="HTTP/HTTPS to WAN1"

# DNS ไป WAN2
add chain=prerouting in-interface=LAN \
    protocol=udp dst-port=53 \
    action=mark-routing new-routing-mark=TO-WAN2 \
    passthrough=no \
    comment="DNS to WAN2"

# VoIP ไป WAN1 (เส้นที่ stable กว่า)
add chain=prerouting in-interface=LAN \
    protocol=udp dst-port=5060,10000-20000 \
    action=mark-routing new-routing-mark=TO-WAN1 \
    passthrough=no \
    comment="VoIP to WAN1"

# File transfer ไป WAN2 (bandwidth สูงกว่า)
add chain=prerouting in-interface=LAN \
    protocol=tcp dst-port=21,22 \
    action=mark-routing new-routing-mark=TO-WAN2 \
    passthrough=no \
    comment="FTP/SSH to WAN2"
```

---

## 43.9 Load Balancing Scripts

### Script ติดตั้ง PCC Load Balance แบบสมบูรณ์

```routeros
/system script
add name="setup-pcc-lb" source={
    :local wan1Iface "WAN1"
    :local wan2Iface "WAN2"
    :local lanIface "LAN"
    :local wan1GW "203.0.113.1"
    :local wan2GW "198.51.100.1"
    
    :put "Setting up PCC Load Balancing..."
    
    # ล้าง mangle rules เก่า
    /ip firewall mangle remove [find comment~"LB:"]
    
    # ล้าง routing tables เก่า
    /ip route remove [find routing-mark~"TO-WAN"]
    
    # สร้าง mangle rules
    /ip firewall mangle
    
    # Mark inbound connections
    add chain=input in-interface=$wan1Iface \
        action=mark-connection new-connection-mark=WAN1-CONN \
        passthrough=yes comment="LB: WAN1 inbound"
    
    add chain=input in-interface=$wan2Iface \
        action=mark-connection new-connection-mark=WAN2-CONN \
        passthrough=yes comment="LB: WAN2 inbound"
    
    # PCC outbound
    add chain=prerouting in-interface=$lanIface \
        per-connection-classifier=both-addresses:2/0 \
        action=mark-connection new-connection-mark=WAN1-CONN \
        passthrough=yes comment="LB: PCC WAN1"
    
    add chain=prerouting in-interface=$lanIface \
        per-connection-classifier=both-addresses:2/1 \
        action=mark-connection new-connection-mark=WAN2-CONN \
        passthrough=yes comment="LB: PCC WAN2"
    
    # Mark routing
    add chain=prerouting connection-mark=WAN1-CONN \
        action=mark-routing new-routing-mark=TO-WAN1 \
        passthrough=yes comment="LB: Route WAN1"
    
    add chain=prerouting connection-mark=WAN2-CONN \
        action=mark-routing new-routing-mark=TO-WAN2 \
        passthrough=yes comment="LB: Route WAN2"
    
    # สร้าง routing tables
    /ip route
    add dst-address=0.0.0.0/0 gateway=$wan1GW \
        routing-mark=TO-WAN1 comment="LB: WAN1 route"
    
    add dst-address=0.0.0.0/0 gateway=$wan2GW \
        routing-mark=TO-WAN2 comment="LB: WAN2 route"
    
    :put "PCC Load Balancing configured successfully!"
    :log info "PCC Load Balancing setup completed"
}
```

---

## 43.10 Lab: Dual-WAN Load Balancing

### Lab Overview

| รายการ | รายละเอียด |
|--------|------------|
| WAN1 | 203.0.113.10/24, GW: 203.0.113.1 |
| WAN2 | 198.51.100.10/24, GW: 198.51.100.1 |
| LAN | 192.168.1.0/24 |
| Method | PCC (Per Connection Classifier) |

### Step 1: ตรวจสอบ WAN connectivity

```routeros
# ตรวจสอบ interfaces
/interface print

# ตรวจสอบ IP addresses
/ip address print

# ทดสอบ ping ผ่าน WAN1
/ping 8.8.8.8 routing-table=TO-WAN1 count=5

# ทดสอบ ping ผ่าน WAN2
/ping 8.8.8.8 routing-table=TO-WAN2 count=5
```

### Step 2: ติดตั้ง PCC

```routeros
# รัน setup script
/system script run setup-pcc-lb
```

### Step 3: ตรวจสอบการทำงาน

```routeros
# ดู mangle rules
/ip firewall mangle print

# ดู routing tables
/ip route print

# ดู connections
/ip firewall connection print

# Monitor traffic
/interface monitor-traffic WAN1,WAN2
```

### Step 4: ทดสอบ Load Balance

```routeros
# เปิด multiple connections จาก LAN และดูว่ากระจายไปทั้งสอง WAN
/ip firewall connection print where connection-mark=WAN1-CONN | count
/ip firewall connection print where connection-mark=WAN2-CONN | count
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| ECMP | Simple multi-path routing |
| PCC | Per-connection load balancing with affinity |
| nth | Round-robin load balancing |
| Mangle-based | Custom protocol/destination based LB |
| Monitoring | Traffic stats, connection counts |
| Failover | WAN availability detection + auto-switch |
| Per-protocol | VoIP, HTTP, streaming สำหรับ WAN ต่าง |

---

[← Part 42: OSPF Scripting](part-042-ospf-scripting.md) | [Part 44: Failover →](part-044-failover.md)
