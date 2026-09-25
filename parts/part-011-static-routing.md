# Part 11: Static Routing

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate
**เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

- [Routing Fundamentals](#routing-fundamentals)
- [Routing Table](#routing-table)
- [Static Route Syntax](#static-route-syntax)
- [Default Route](#default-route)
- [Floating Static Route (Backup)](#floating-static-route)
- [Policy-Based Routing](#policy-based-routing)
- [Route Distances และ Metrics](#route-distances)
- [ECMP (Equal Cost Multi-Path)](#ecmp)
- [Blackhole Routes](#blackhole-routes)
- [Route Monitoring](#route-monitoring)
- [Lab: Multi-Router Static Routing](#lab)

---

## Routing Fundamentals

### Routing คืออะไร?

**Routing** คือกระบวนการที่ Router ตัดสินใจว่าจะส่ง Packet ไปทางใด โดยอ้างอิงจาก Destination IP Address ของ Packet นั้น

```
                    ┌─────────┐
   PC-A ─────────── │ Router1 │ ─────────── Router2 ─── PC-B
   192.168.1.10      └─────────┘              │
                                          PC-C
```

Router ต้องรู้ว่า:
- Network ปลายทางอยู่ที่ Interface ไหน
- หรือต้องส่งต่อไปยัง Next-Hop Router ตัวใด

### ประเภทของ Routing

| ประเภท | คำอธิบาย | ข้อดี | ข้อเสีย |
|--------|----------|-------|---------|
| **Static Routing** | กำหนด Route ด้วยมือ | ควบคุมได้ 100%, ไม่ใช้ CPU | ดูแลยากในเครือข่ายใหญ่ |
| **Dynamic Routing** | ใช้ Protocol (OSPF, BGP) แลกเปลี่ยนข้อมูล Auto | Scale ได้ดี | ซับซ้อน, ใช้ Resource มากกว่า |
| **Connected Routes** | เพิ่มอัตโนมัติเมื่อ Interface มี IP | ไม่ต้องตั้ง | จำกัดเฉพาะ Network ที่เชื่อมต่อโดยตรง |

### Routing Decision Process

เมื่อ Router ได้รับ Packet จะทำตามขั้นตอนนี้:

```
Packet มาถึง
      │
      ▼
ดู Destination IP
      │
      ▼
ค้นหาใน Routing Table (Longest Prefix Match)
      │
      ├── พบ Route ─► ส่ง Packet ออก Interface ที่ระบุ
      │
      └── ไม่พบ Route ─► Drop Packet (หรือใช้ Default Route)
```

### Longest Prefix Match

RouterOS ใช้หลัก **Longest Prefix Match** คือเลือก Route ที่มี Prefix ยาวที่สุด (จำเพาะที่สุด)

**ตัวอย่าง:**
```
Routing Table:
  10.0.0.0/8     via 192.168.1.1
  10.10.0.0/16   via 192.168.1.2
  10.10.10.0/24  via 192.168.1.3

Packet ไปยัง 10.10.10.5 จะใช้:
  → 10.10.10.0/24 (prefix ยาวสุด = จำเพาะที่สุด)
```

---

## Routing Table

### ดู Routing Table

```routeros
# ดู Routing Table ทั้งหมด
/ip route print

# ดูแบบ Detail
/ip route print detail

# Filter เฉพาะ Active routes
/ip route print where active=yes

# Filter เฉพาะ Static routes
/ip route print where static=yes

# ดูแบบ Brief
/ip route print brief
```

### ทำความเข้าใจ Columns

```
Flags: X - disabled, A - active, D - dynamic, C - connect,
       S - static, r - rip, b - bgp, o - ospf, m - mme, B - blackhole,
       U - unreachable, P - prohibit

 #      DST-ADDRESS        PREF-SRC          GATEWAY            DISTANCE
 0 A S  0.0.0.0/0                            203.0.113.1              1
 1 ADC  192.168.1.0/24    192.168.1.1        ether1                   0
 2 A S  10.0.0.0/8                           192.168.2.1              1
```

**ความหมายของ Flags:**
- `A` = Active (ใช้งานได้)
- `D` = Dynamic (เพิ่มโดย Protocol อัตโนมัติ)
- `C` = Connected (Interface เชื่อมต่อโดยตรง)
- `S` = Static (กำหนดเอง)
- `B` = Blackhole (ทิ้ง Packet)

### Routing Table Lookup

```routeros
# ทดสอบว่า Packet ไปยัง IP นี้จะใช้ Route ใด
/ip route check 8.8.8.8

# ดู Route สำหรับ Destination เฉพาะ
/ip route get [find dst-address=192.168.10.0/24]
```

---

## Static Route Syntax

### คำสั่งพื้นฐาน

```routeros
# รูปแบบ
/ip route add dst-address=<NETWORK/PREFIX> gateway=<NEXT-HOP|INTERFACE>

# ตัวอย่าง: Route ไปยัง Network 10.0.0.0/8 ผ่าน Gateway 192.168.1.1
/ip route add dst-address=10.0.0.0/8 gateway=192.168.1.1

# ตัวอย่าง: Route ผ่าน Interface โดยตรง
/ip route add dst-address=172.16.0.0/16 gateway=ether2

# ตัวอย่าง: Route พร้อมกำหนด Distance
/ip route add dst-address=10.0.0.0/8 gateway=192.168.1.1 distance=10
```

### Parameters ทั้งหมด

| Parameter | คำอธิบาย | ค่าเริ่มต้น |
|-----------|----------|------------|
| `dst-address` | Destination Network/Host | - |
| `gateway` | Next-Hop IP หรือ Interface | - |
| `distance` | Administrative Distance | 1 |
| `scope` | Route Scope | 30 |
| `target-scope` | Target Scope | 10 |
| `comment` | หมายเหตุ | - |
| `disabled` | เปิด/ปิด Route | no |
| `routing-mark` | สำหรับ PBR | - |

### ตัวอย่าง Static Routes ที่ใช้บ่อย

```routeros
# 1. Route ไปยัง Branch Office
/ip route add \
    dst-address=192.168.100.0/24 \
    gateway=10.0.0.2 \
    comment="Branch Office Bangkok"

# 2. Route ผ่าน VPN Tunnel
/ip route add \
    dst-address=172.16.0.0/12 \
    gateway=vpn-tunnel1 \
    comment="VPN to HQ"

# 3. Route หลายเส้นทาง
/ip route add dst-address=10.0.0.0/8 gateway=192.168.1.1
/ip route add dst-address=10.0.0.0/8 gateway=192.168.2.1

# 4. Route สำหรับ Host เดียว
/ip route add dst-address=8.8.8.8/32 gateway=203.0.113.1
```

### การแก้ไขและลบ Route

```routeros
# แก้ไข Route
/ip route set [find dst-address=10.0.0.0/8] gateway=192.168.3.1

# ปิด Route ชั่วคราว
/ip route disable [find dst-address=10.0.0.0/8]

# เปิด Route
/ip route enable [find dst-address=10.0.0.0/8]

# ลบ Route
/ip route remove [find dst-address=10.0.0.0/8]

# ลบ Route ทั้งหมดที่เป็น Static
/ip route remove [find static=yes !dynamic]
```

> **Warning:** ระวังการลบ Route ที่ใช้งานอยู่ อาจทำให้ขาดการเชื่อมต่อได้

---

## Default Route

### Default Route คืออะไร?

**Default Route** คือ Route ที่ใช้เมื่อไม่พบ Route ที่ตรงกับ Destination ใน Routing Table เปรียบเหมือน "ทางออกสุดท้าย"

- Destination: `0.0.0.0/0`
- ครอบคลุม IP ทุก Address

### การตั้งค่า Default Route

```routeros
# Default Route พื้นฐาน
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1

# Default Route พร้อม Comment
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=203.0.113.1 \
    comment="ISP1 Default Gateway"

# ตรวจสอบ
/ip route print where dst-address=0.0.0.0/0
```

### Default Route สำหรับ Dual ISP

```routeros
# ISP1 เป็น Primary (Distance = 1)
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=203.0.113.1 \
    distance=1 \
    comment="ISP1 Primary"

# ISP2 เป็น Backup (Distance = 10)
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=198.51.100.1 \
    distance=10 \
    comment="ISP2 Backup"
```

> **Note:** Route ที่มี Distance ต่ำกว่าจะได้รับเลือกก่อนเสมอ

---

## Floating Static Route

### Floating Static Route คืออะไร?

**Floating Static Route** คือ Static Route สำรองที่จะถูกใช้งานเมื่อ Route หลักล้มเหลว โดยกำหนด Distance ให้สูงกว่า Route หลัก

### การตั้งค่า Backup Route

```routeros
# Route หลัก ผ่าน ISP1
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=203.0.113.1 \
    distance=1 \
    check-gateway=ping \
    comment="Primary ISP1"

# Floating Static Route ผ่าน ISP2 (Backup)
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=198.51.100.1 \
    distance=100 \
    comment="Backup ISP2 - Floating"
```

### Gateway Check

RouterOS มีฟีเจอร์ `check-gateway` ที่ Monitor Gateway:

```routeros
# ใช้ Ping ตรวจสอบ Gateway
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=203.0.113.1 \
    check-gateway=ping \
    distance=1

# ใช้ ARP ตรวจสอบ (เร็วกว่า แต่ใช้ได้เฉพาะ Same Subnet)
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=203.0.113.1 \
    check-gateway=arp \
    distance=1

# ตั้งค่า Timeout สำหรับ Gateway Check
/ip settings set route-cache=yes
```

### Script สำหรับ Auto Failover

```routeros
# Script ตรวจสอบ Link และ Failover
:local primary "203.0.113.1"
:local backup "198.51.100.1"
:local testHost "8.8.8.8"

# ทดสอบ Primary Link
:if ([/ping $testHost count=3 interface=ether1 routing-table=main] = 0) do={
    :log warning "Primary ISP down! Switching to backup..."
    /ip route disable [find gateway=$primary dst-address=0.0.0.0/0]
    /ip route enable [find gateway=$backup dst-address=0.0.0.0/0]
} else={
    /ip route enable [find gateway=$primary dst-address=0.0.0.0/0]
    /ip route disable [find gateway=$backup dst-address=0.0.0.0/0]
    :log info "Primary ISP is up"
}
```

---

## Policy-Based Routing

### Policy-Based Routing คืออะไร?

**Policy-Based Routing (PBR)** คือการกำหนด Route โดยอิงกับ Policy ไม่ใช่แค่ Destination IP เช่น:
- Route ตาม Source IP
- Route ตาม Protocol (TCP/UDP)
- Route ตาม Port
- Route ตาม Interface ที่เข้ามา

### Routing Marks

PBR ใน RouterOS ทำผ่าน **Routing Marks** โดย:
1. Mark Packet ใน Mangle
2. กำหนด Route ตาม Mark

```routeros
# Step 1: กำหนด Routing Table
/routing table add name=ISP2-table fib

# Step 2: Mark Packet จาก Subnet 192.168.2.0/24 ให้ใช้ ISP2
/ip firewall mangle add \
    chain=prerouting \
    src-address=192.168.2.0/24 \
    action=mark-routing \
    new-routing-mark=ISP2-table \
    passthrough=yes \
    comment="PBR: Route subnet 2 via ISP2"

# Step 3: เพิ่ม Route ใน ISP2-table
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=198.51.100.1 \
    routing-table=ISP2-table \
    comment="ISP2 for subnet 2"
```

### PBR ตาม Service (Port)

```routeros
# Mark HTTP/HTTPS ให้ใช้ ISP1
/ip firewall mangle add \
    chain=prerouting \
    protocol=tcp \
    dst-port=80,443 \
    action=mark-routing \
    new-routing-mark=ISP1-table \
    comment="Web traffic via ISP1"

# Mark VoIP (SIP) ให้ใช้ ISP2
/ip firewall mangle add \
    chain=prerouting \
    protocol=udp \
    dst-port=5060,10000-20000 \
    action=mark-routing \
    new-routing-mark=ISP2-table \
    comment="VoIP via ISP2"
```

### ตรวจสอบ Routing Tables

```routeros
# ดู Routing Tables ทั้งหมด
/routing table print

# ดู Routes ใน Table เฉพาะ
/ip route print routing-table=ISP2-table

# ทดสอบ Route โดยระบุ Table
/ip route check 8.8.8.8 routing-table=ISP2-table
```

---

## Route Distances

### Administrative Distance คืออะไร?

**Administrative Distance (AD)** คือค่าความน่าเชื่อถือของ Route Source ยิ่งต่ำยิ่งดี

| Route Source | Default Distance | RouterOS Value |
|-------------|-----------------|----------------|
| Connected | 0 | 0 |
| Static | 1 | 1 |
| OSPF Internal | 110 | 110 |
| RIP | 120 | 120 |
| BGP External | 20 | 20 |
| BGP Internal | 200 | 200 |

### การใช้ Distance ในทางปฏิบัติ

```routeros
# Route หลัก Distance = 1
/ip route add dst-address=10.0.0.0/8 gateway=192.168.1.1 distance=1

# Route สำรอง Distance = 50 (ใช้เมื่อ Route หลักล้มเหลว)
/ip route add dst-address=10.0.0.0/8 gateway=192.168.2.1 distance=50

# Route สำรองสุดท้าย Distance = 200
/ip route add dst-address=10.0.0.0/8 gateway=192.168.3.1 distance=200
```

### Scope และ Target-Scope

```routeros
# Scope กำหนดว่า Route นี้จะถูก Export/Share ไปหรือไม่
# Target-Scope กำหนดว่า Route ที่ใช้เป็น Recursive Gateway ต้องมี Scope ไม่เกินเท่าไร

/ip route add \
    dst-address=10.0.0.0/8 \
    gateway=192.168.1.1 \
    scope=30 \
    target-scope=10

# ค่า Scope ที่ใช้บ่อย:
# 10 = Host route (connected)
# 30 = Default scope สำหรับ Static routes
# 200 = BGP routes
```

---

## ECMP (Equal Cost Multi-Path)

### ECMP คืออะไร?

**ECMP** คือการใช้หลาย Route ที่มี Cost เท่ากันพร้อมกัน เพื่อกระจาย Load (Load Balancing)

### การตั้งค่า ECMP

```routeros
# เพิ่ม 2 Routes ด้วย Distance เท่ากัน
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1
/ip route add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=1

# ดู Routes ที่เป็น ECMP
/ip route print where dst-address=0.0.0.0/0
```

ผลลัพธ์:
```
#   DST-ADDRESS    GATEWAY         DISTANCE
0 AS 0.0.0.0/0    203.0.113.1         1
1 AS 0.0.0.0/0    198.51.100.1        1
```

### ECMP Load Balancing Modes

```routeros
# กำหนดวิธี Load Balance (ตั้งค่าใน /ip settings)
/ip settings set route-cache=yes

# ดู per-connection load balance statistics
/ip route print stats
```

### ข้อจำกัดของ ECMP

> **Note:** ECMP จะกระจาย Traffic ต่อ Connection (per-flow) ไม่ใช่ per-packet ดังนั้นถ้ามี Connection เดียวที่ใช้ Bandwidth สูง ECMP จะไม่ช่วย

### ECMP พร้อม Weights (RouterOS v7)

```routeros
# RouterOS v7 รองรับ Weighted ECMP
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=203.0.113.1 \
    distance=1 \
    ecmp-weight=3    # ISP1 รับ Traffic 3 ส่วน

/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=198.51.100.1 \
    distance=1 \
    ecmp-weight=1    # ISP2 รับ Traffic 1 ส่วน
```

---

## Blackhole Routes

### Blackhole Route คืออะไร?

**Blackhole Route** คือ Route ที่ทิ้ง Packet ทันทีโดยไม่ส่งต่อ และไม่ส่ง ICMP Error กลับ ใช้เพื่อ:
- Block Traffic สำหรับ Network ที่ไม่มีอยู่จริง
- ป้องกัน Routing Loop
- Rate Limiting / Traffic Engineering

### การสร้าง Blackhole Route

```routeros
# Blackhole - ทิ้ง Packet เงียบๆ
/ip route add dst-address=10.0.0.0/8 type=blackhole

# Unreachable - ส่ง ICMP "Host Unreachable" กลับ
/ip route add dst-address=10.0.0.0/8 type=unreachable

# Prohibit - ส่ง ICMP "Communication Administratively Prohibited" กลับ
/ip route add dst-address=10.0.0.0/8 type=prohibit
```

### Use Case: Null Route สำหรับ Security

```routeros
# Block IP ที่ถูก DDoS โดยใช้ Blackhole (RTBH - Remote Triggered Blackhole)
/ip route add \
    dst-address=203.0.113.100/32 \
    type=blackhole \
    comment="DDoS mitigation - block attacker"

# Block Bogon Networks (IP ที่ไม่ควรมีใน Internet)
/ip route add dst-address=10.0.0.0/8 type=blackhole comment="RFC1918"
/ip route add dst-address=172.16.0.0/12 type=blackhole comment="RFC1918"
/ip route add dst-address=192.168.0.0/16 type=blackhole comment="RFC1918"
```

### Blackhole สำหรับ Aggregate Routes

```routeros
# Summary Route พร้อม Blackhole (ป้องกัน Routing Loop)
# เมื่อ Router มี 192.168.1.0/24, 192.168.2.0/24, 192.168.3.0/24

# Blackhole สำหรับ Aggregate
/ip route add dst-address=192.168.0.0/22 type=blackhole distance=254 comment="Aggregate blackhole"

# Routes จริงๆ (มี Distance ต่ำกว่า จะถูกเลือกก่อน)
/ip route add dst-address=192.168.1.0/24 gateway=10.0.0.1 distance=1
/ip route add dst-address=192.168.2.0/24 gateway=10.0.0.2 distance=1
/ip route add dst-address=192.168.3.0/24 gateway=10.0.0.3 distance=1
```

---

## Route Monitoring

### การ Monitor Routes

```routeros
# ดู Route แบบ Real-time
/ip route print interval=2

# Watch Specific Route
:while (true) do={
    :local r [/ip route get [find dst-address=0.0.0.0/0 active=yes] gateway]
    :log info "Default route via: $r"
    :delay 10
}
```

### ตรวจสอบสถานะ Route

```routeros
# ดู Route Statistics
/ip route print stats

# ดูว่า Route ไหน Active
/ip route print where active=yes

# ดู Route ที่ Disabled
/ip route print where disabled=yes

# ดู Route ที่เป็น Gateway-down
/ip route print where active=no
```

### Netwatch สำหรับ Monitor Host

```routeros
# ใช้ Netwatch เพื่อ Monitor Host และ ทำ Failover
/tool netwatch add \
    host=8.8.8.8 \
    interval=30s \
    timeout=3s \
    up-script={
        /ip route enable [find comment="Primary ISP"]
        /ip route disable [find comment="Backup ISP"]
        :log info "Primary ISP restored"
    } \
    down-script={
        /ip route disable [find comment="Primary ISP"]
        /ip route enable [find comment="Backup ISP"]
        :log warning "Primary ISP DOWN - Switched to Backup"
    }
```

### Route Change Notification

```routeros
# Script ส่ง Email เมื่อ Default Route เปลี่ยน
:local currentGW [/ip route get [find dst-address=0.0.0.0/0 active=yes] gateway]
:local lastGW [/system script get route-monitor source]

:if ($currentGW != $lastGW) do={
    /tool e-mail send \
        to="admin@example.com" \
        subject="Route Change Alert" \
        body="Default gateway changed from $lastGW to $currentGW"
    :log warning "Default gateway changed: $lastGW -> $currentGW"
}
```

---

## Lab: Multi-Router Static Routing

### Network Topology

```
Internet (8.8.8.8)
        │
   203.0.113.1 (ISP)
        │
   ether1 [203.0.113.2]
   ┌─────────────┐
   │   Router-A  │ ─── ether3 [10.0.0.1] ─── [10.0.0.2] ether1 ─┐
   └─────────────┘                                                  │
   ether2 [192.168.1.1]                                    ┌───────────────┐
        │                                                   │   Router-B    │
   PC-LAN-A                                                └───────────────┘
   192.168.1.0/24                                          ether2 [192.168.2.1]
                                                                │
                                                           PC-LAN-B
                                                           192.168.2.0/24
```

### Objectives

1. PC-LAN-A สามารถ Ping PC-LAN-B ได้
2. ทั้งสอง LAN สามารถเข้า Internet ได้
3. เมื่อ Link หลักล้มเหลว มี Backup Link

### Step 1: ตั้งค่า Router-A

```routeros
# --- Router-A Configuration ---

# กำหนดชื่อ Router
/system identity set name=Router-A

# ตั้งค่า IP Addresses
/ip address add address=203.0.113.2/30 interface=ether1 comment="WAN - ISP"
/ip address add address=192.168.1.1/24 interface=ether2 comment="LAN-A"
/ip address add address=10.0.0.1/30 interface=ether3 comment="Link to Router-B"

# Default Route ออก Internet
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 comment="Default via ISP"

# Route ไปยัง LAN-B ผ่าน Router-B
/ip route add dst-address=192.168.2.0/24 gateway=10.0.0.2 distance=1 comment="Route to LAN-B"

# NAT สำหรับ Internet Access
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="NAT for Internet"
```

### Step 2: ตั้งค่า Router-B

```routeros
# --- Router-B Configuration ---

# กำหนดชื่อ Router
/system identity set name=Router-B

# ตั้งค่า IP Addresses
/ip address add address=10.0.0.2/30 interface=ether1 comment="Link to Router-A"
/ip address add address=192.168.2.1/24 interface=ether2 comment="LAN-B"

# Default Route ผ่าน Router-A (ออก Internet)
/ip route add dst-address=0.0.0.0/0 gateway=10.0.0.1 distance=1 comment="Default via Router-A"

# Route ไปยัง LAN-A ผ่าน Router-A
/ip route add dst-address=192.168.1.0/24 gateway=10.0.0.1 distance=1 comment="Route to LAN-A"
```

### Step 3: ทดสอบ Connectivity

```routeros
# ทดสอบจาก Router-A
/ping 192.168.2.1 count=4     # Ping Router-B LAN interface
/ping 192.168.2.10 count=4    # Ping PC ใน LAN-B
/ping 8.8.8.8 count=4         # Ping Internet

# ทดสอบจาก Router-B
/ping 192.168.1.1 count=4     # Ping Router-A LAN interface
/ping 8.8.8.8 count=4         # Ping Internet

# ดู Routing Table
/ip route print
```

### Step 4: เพิ่ม Floating Static Route (Backup)

```routeros
# สมมติว่ามี Backup Link ระหว่าง Router-A และ Router-B ผ่าน ether4
# Router-A: ether4 = 10.0.1.1/30
# Router-B: ether4 = 10.0.1.2/30

# Router-A: เพิ่ม Backup Route ไปยัง LAN-B
/ip route add \
    dst-address=192.168.2.0/24 \
    gateway=10.0.1.2 \
    distance=100 \
    comment="Backup route to LAN-B via ether4"

# Router-B: เพิ่ม Backup Default Route
/ip route add \
    dst-address=0.0.0.0/0 \
    gateway=10.0.1.1 \
    distance=100 \
    comment="Backup default via Router-A ether4"
```

### Step 5: ตั้งค่า Gateway Monitoring

```routeros
# Router-A: Monitor link ไปยัง Router-B
/tool netwatch add \
    host=10.0.0.2 \
    interval=10s \
    timeout=3s \
    up-script={
        :log info "Primary link to Router-B is UP"
    } \
    down-script={
        :log warning "Primary link to Router-B is DOWN"
    }
```

### Step 6: Verification Commands

```routeros
# ตรวจสอบ Routing Table
/ip route print
/ip route print where active=yes

# Traceroute เพื่อดู Path
/tool traceroute 192.168.2.10
/tool traceroute 8.8.8.8

# ดู ARP Table
/ip arp print

# ดู Neighbor
/ip neighbor print
```

### ผลลัพธ์ที่คาดหวัง

```
Router-A# /ip route print
Flags: X - disabled, A - active, D - dynamic, C - connect, S - static

 #      DST-ADDRESS        GATEWAY        DISTANCE
 0 ADC  203.0.113.0/30    203.0.113.2       0
 1 ADC  192.168.1.0/24    192.168.1.1       0
 2 ADC  10.0.0.0/30       10.0.0.1          0
 3 A S  0.0.0.0/0         203.0.113.1       1      ← Default route
 4 A S  192.168.2.0/24    10.0.0.2          1      ← Route to LAN-B
 5   S  192.168.2.0/24    10.0.1.2        100      ← Backup (inactive)
```

---

## Troubleshooting

### ปัญหาที่พบบ่อย

**1. Route ไม่ Active**
```routeros
# ตรวจสอบว่า Gateway Reachable ไหม
/ping <gateway-ip>

# ตรวจสอบว่า Interface Up ไหม
/interface print

# ดู Route Detail
/ip route print detail where dst-address=<network>
```

**2. Routing Loop**
```routeros
# ใช้ Traceroute หาจุดที่วน
/tool traceroute <destination>

# ตรวจสอบ TTL ใน Ping
/ping <destination> ttl=5
```

**3. Asymmetric Routing**
```routeros
# ดูว่า Traffic ออกทาง Interface ใด
/ip route check <destination>

# Monitor Traffic per interface
/interface monitor-traffic ether1,ether2 interval=1
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 11:

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| Routing Fundamentals | Router ตัดสินใจโดยใช้ Longest Prefix Match |
| Routing Table | เก็บข้อมูล Routes ทั้งหมด, ดูด้วย `/ip route print` |
| Static Routes | กำหนด Route ด้วยมือ, ควบคุมได้ 100% |
| Default Route | `0.0.0.0/0` ใช้เมื่อไม่มี Route ที่ Match |
| Floating Static | Backup route ที่มี Distance สูงกว่า |
| Policy-Based Routing | Route ตาม Policy ผ่าน Routing Marks |
| Distance | ยิ่งต่ำยิ่งดี, ใช้เลือก Route เมื่อมีหลาย Route |
| ECMP | Load balance ผ่านหลาย Gateway พร้อมกัน |
| Blackhole | ทิ้ง Packet เงียบๆ |

---

## แบบทดสอบ

1. หาก Routing Table มี Route `10.0.0.0/8` และ `10.10.0.0/16` Packet ไปยัง `10.10.5.1` จะใช้ Route ใด?
2. อธิบายความแตกต่างระหว่าง Blackhole, Unreachable, และ Prohibit Route
3. ทำไม Floating Static Route ต้องมี Distance สูงกว่า Route หลัก?
4. ECMP ต่างจาก Failover อย่างไร?
5. Policy-Based Routing ใช้กลไกอะไรในการ Mark Packet?

---

## Navigation

[← Part 10: Firewall Basics](part-010-firewall-basics.md) | [Part 12: Bridge Configuration →](part-012-bridge-configuration.md)

---

*MikroTik RouterOS Administration Course - Part 11*
*Last Updated: 2026*
