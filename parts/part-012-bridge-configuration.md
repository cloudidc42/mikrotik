# Part 12: Bridge Configuration

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate
**เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

- [Bridge Concepts](#bridge-concepts)
- [Creating Bridges](#creating-bridges)
- [Bridge Ports](#bridge-ports)
- [Bridge STP](#bridge-stp)
- [RSTP (Rapid STP)](#rstp)
- [MSTP](#mstp)
- [Bridge VLAN Filtering](#bridge-vlan-filtering)
- [Bridge Hardware Offloading](#bridge-hardware-offloading)
- [Bridge Monitoring](#bridge-monitoring)
- [Lab: Office Network with Bridge](#lab)

---

## Bridge Concepts

### Bridge คืออะไร?

**Bridge** คืออุปกรณ์ Layer 2 ที่เชื่อม Network Segments เข้าด้วยกัน โดยส่ง Traffic โดยอ้างอิงจาก MAC Address ไม่ใช่ IP Address

```
Without Bridge:
  Segment A ──────────────────── Segment B
  (แยกกัน ไม่สื่อสารกัน)

With Bridge:
  Segment A ──── [Bridge] ──── Segment B
  (สื่อสารกันได้เหมือนอยู่ใน Network เดียวกัน)
```

### Bridge vs Switch vs Router

| Feature | Hub | Bridge/Switch | Router |
|---------|-----|--------------|--------|
| OSI Layer | 1 | 2 | 3 |
| Addressing | - | MAC Address | IP Address |
| Broadcast Domain | 1 | 1 per VLAN | แยกต่างหาก |
| Collision Domain | 1 | แยกต่อ Port | แยกต่างหาก |
| Intelligent | ไม่มี | มีบ้าง | มีมาก |

### MikroTik Bridge Overview

ใน RouterOS **Bridge** ทำงานเป็น Virtual Interface ที่:
- รวม Physical Interfaces หลายอันเข้าด้วยกัน
- มี MAC Address และ IP Address ของตัวเอง
- รองรับ STP, RSTP, MSTP
- รองรับ VLAN Filtering
- รองรับ Hardware Offloading (ใน Switch-chip Router)

### Bridge Forwarding Database (FDB)

Bridge เรียนรู้ MAC Address ของ Devices และสร้าง **Forwarding Database**:

```
FDB:
  Port1 ← MAC: AA:BB:CC:DD:EE:01 (PC-A)
  Port1 ← MAC: AA:BB:CC:DD:EE:02 (PC-B)
  Port2 ← MAC: AA:BB:CC:DD:EE:03 (PC-C)
  Port3 ← MAC: AA:BB:CC:DD:EE:04 (Server-1)
```

เมื่อ PC-A ส่ง Packet ไป PC-C:
1. Bridge ค้นหา MAC ของ PC-C ใน FDB
2. พบว่าอยู่ที่ Port2
3. ส่ง Packet ออก Port2 เท่านั้น (ไม่ Flood ทุก Port)

---

## Creating Bridges

### สร้าง Bridge เบื้องต้น

```routeros
# สร้าง Bridge Interface
/interface bridge add name=bridge1 comment="Main LAN Bridge"

# ดู Bridge Interfaces
/interface bridge print

# ดู Bridge แบบ Detail
/interface bridge print detail
```

### Bridge Parameters สำคัญ

```routeros
/interface bridge add \
    name=bridge1 \
    comment="Main LAN Bridge" \
    protocol-mode=rstp \        # STP Protocol (none/stp/rstp/mstp)
    priority=0x8000 \           # Bridge Priority (0-65535, default 32768)
    forward-delay=15s \         # Forward Delay
    max-message-age=20s \       # Max Message Age
    ageing-time=5m \            # MAC Address Ageing Time
    vlan-filtering=no \         # Enable VLAN Filtering
    dhcp-snooping=no            # Enable DHCP Snooping
```

### ตั้งค่า IP บน Bridge

```routeros
# เพิ่ม IP ให้ Bridge (ทำหน้าที่เป็น Gateway)
/ip address add address=192.168.1.1/24 interface=bridge1 comment="LAN Gateway"

# หรือใช้ DHCP บน Bridge
/ip dhcp-server add interface=bridge1 address-pool=dhcp-pool name=dhcp1
```

---

## Bridge Ports

### เพิ่ม Interface เข้า Bridge

```routeros
# เพิ่ม Port เข้า Bridge
/interface bridge port add interface=ether2 bridge=bridge1
/interface bridge port add interface=ether3 bridge=bridge1
/interface bridge port add interface=ether4 bridge=bridge1
/interface bridge port add interface=wlan1 bridge=bridge1

# ดู Bridge Ports
/interface bridge port print

# ดู Port แบบ Detail (รวม STP Status)
/interface bridge port print detail
```

### Bridge Port Parameters

```routeros
/interface bridge port add \
    interface=ether2 \
    bridge=bridge1 \
    priority=0x80 \         # Port Priority (0-255)
    path-cost=10 \          # Path Cost สำหรับ STP
    horizon=none \          # Split Horizon (none/1-429496729)
    learn=auto \            # Learning Mode (auto/yes/no)
    discover=yes \          # Discover (for STP)
    flood=yes \             # Flood Unknown Unicast
    point-to-point=auto \   # Point-to-Point link (auto/yes/no)
    edge=auto               # Edge Port (auto/yes/no)
```

### การลบ Port จาก Bridge

```routeros
# ลบ Port ออกจาก Bridge
/interface bridge port remove [find interface=ether2]

# หรือ Disable Port ชั่วคราว
/interface bridge port disable [find interface=ether2]
```

### Bridge Port Status

```routeros
# ดู Status ของ Bridge Ports
/interface bridge port print

# Status ที่พบ:
# designated  = Port นี้เป็น Designated Port
# root        = Port นี้เป็น Root Port
# disabled    = Port ถูก Blocked โดย STP
# forwarding  = Port ส่ง Traffic ได้ปกติ
```

---

## Bridge STP

### STP คืออะไร?

**Spanning Tree Protocol (STP)** คือ Protocol ที่ป้องกัน **Broadcast Storm** ในเครือข่ายที่มีหลาย Path (Redundant Links)

**ปัญหาที่ STP แก้ไข:**
```
PC-A ─── Switch1 ─── Switch2 ─── PC-B
              └────────────────┘
              (Redundant Link = ปัญหา Loop!)
```

หากไม่มี STP, Broadcast จาก PC-A จะวนซ้ำไปเรื่อยๆ จนเครือข่ายล่ม

### STP Election Process

1. **เลือก Root Bridge** - Bridge ที่มี Priority ต่ำสุด (แล้ว MAC ต่ำสุด)
2. **เลือก Root Port** - Port ที่ถูกที่สุดในการไปถึง Root Bridge
3. **เลือก Designated Port** - Port ที่ดีที่สุดในแต่ละ Segment
4. **Block Ports ที่เหลือ** - เพื่อป้องกัน Loop

### ตั้งค่า STP

```routeros
# เปิดใช้ STP
/interface bridge set bridge1 protocol-mode=stp

# กำหนด Bridge Priority (เพื่อบังคับให้เป็น Root Bridge)
# ค่าต่ำ = Priority สูง = โอกาสเป็น Root Bridge มากกว่า
/interface bridge set bridge1 priority=0x1000  # Priority = 4096

# กำหนด Path Cost สำหรับ Port (เพื่อกำหนด Root Port)
/interface bridge port set [find interface=ether2] path-cost=4

# ดู STP State
/interface bridge print detail
/interface bridge port print detail
```

### STP Port States

| State | คำอธิบาย | รับ Traffic? | ส่ง Traffic? | Learning? |
|-------|----------|-------------|-------------|----------|
| Disabled | Port ถูก Shutdown | ไม่ | ไม่ | ไม่ |
| Blocking | Block โดย STP | ไม่ | ไม่ | ไม่ |
| Listening | กำลัง Monitor BPDUs | ไม่ | ไม่ | ไม่ |
| Learning | กำลัง Learn MAC | ไม่ | ไม่ | ใช่ |
| Forwarding | ทำงานปกติ | ใช่ | ใช่ | ใช่ |

### STP Timers

```routeros
/interface bridge set bridge1 \
    forward-delay=15s \     # เวลาอยู่ใน Listening + Learning state
    max-message-age=20s \   # เวลา BPDU ยังมีผล
    hello-time=2s           # ความถี่ส่ง BPDU (Root Bridge เท่านั้น)
```

> **Note:** การ Converge ของ STP ใช้เวลา ~30-50 วินาที ซึ่งช้ามาก จึงแนะนำให้ใช้ RSTP แทน

---

## RSTP (Rapid STP)

### RSTP คืออะไร?

**Rapid Spanning Tree Protocol (RSTP - IEEE 802.1w)** คือ STP เวอร์ชันปรับปรุง ที่ Converge เร็วกว่ามาก (< 1 วินาที)

**ความแตกต่างหลักจาก STP:**
- Port States ลดเหลือ 3 (Discarding, Learning, Forwarding)
- Edge Ports Converge ทันที
- Proposal/Agreement Mechanism แทน Timer-based

### ตั้งค่า RSTP

```routeros
# เปิดใช้ RSTP (แนะนำสำหรับเครือข่ายทั่วไป)
/interface bridge set bridge1 protocol-mode=rstp

# กำหนด Edge Port (สำหรับ Port ที่เชื่อมต่อกับ End Device โดยตรง)
# Edge Port จะ Skip การรอ STP Convergence
/interface bridge port set [find interface=ether2] edge=yes point-to-point=yes

# Auto-detect Edge Port (PortFast equivalent)
/interface bridge port set [find interface=ether2] edge=auto
```

### BPDU Guard

```routeros
# BPDU Guard - ปิด Port ทันทีหากได้รับ BPDU บน Edge Port
/interface bridge port set [find interface=ether2] \
    edge=yes \
    bpdu-guard=yes \
    comment="Access port with BPDU Guard"

# ดู Port ที่ถูก Disable โดย BPDU Guard
/interface bridge port print where status=bpdu-guard-blocked

# Recovery - Enable Port กลับมาด้วยมือ
/interface bridge port enable [find interface=ether2]
```

### Root Guard

```routeros
# Root Guard - ป้องกัน Port นี้จากการเป็น Root Port
# ใช้ใน Ports ที่ไม่ควรเชื่อมไปยัง Root Bridge
/interface bridge port set [find interface=ether3] root-path-cost=1000000
```

### RSTP Topology ตัวอย่าง

```routeros
# --- Topology ---
# Core-Switch (Root Bridge) ─── Access-Switch-1 ─── PC
#                          └─── Access-Switch-2 ─── PC
#                Access-Switch-1 ─── Access-Switch-2 (Redundant)

# Core-Switch: ตั้งเป็น Root Bridge
/interface bridge set bridge1 priority=0x1000 protocol-mode=rstp

# Access-Switch-1
/interface bridge set bridge1 priority=0x8000 protocol-mode=rstp
# Uplink ports (ไปหา Core) - ปกติ
/interface bridge port set [find interface=ether1] path-cost=10
# Access ports (ไปหา PC) - Edge Port
/interface bridge port set [find interface=ether2] edge=yes point-to-point=yes
/interface bridge port set [find interface=ether3] edge=yes point-to-point=yes
```

---

## MSTP

### MSTP คืออะไร?

**Multiple Spanning Tree Protocol (MSTP - IEEE 802.1s)** ให้มี STP Instance หลายตัว แต่ละ Instance ดูแล VLAN ต่างๆ ทำให้สามารถ Load Balance ได้

```
VLAN 10 ── Instance 1 ── เส้นทาง A เป็น Active
VLAN 20 ── Instance 2 ── เส้นทาง B เป็น Active
```

### ตั้งค่า MSTP

```routeros
# เปิดใช้ MSTP
/interface bridge set bridge1 protocol-mode=mstp

# กำหนด MST Region
/interface bridge set bridge1 \
    region-name=MY-REGION \
    region-revision=1

# กำหนด VLAN-Instance Mapping
/interface bridge mst-override add \
    bridge=bridge1 \
    msti=1 \
    vlan-mapping=10-19 \
    comment="VLAN 10-19 in MST Instance 1"

/interface bridge mst-override add \
    bridge=bridge1 \
    msti=2 \
    vlan-mapping=20-29 \
    comment="VLAN 20-29 in MST Instance 2"

# กำหนด Priority ต่อ Instance
/interface bridge mst-override set [find msti=1] priority=0x1000
```

---

## Bridge VLAN Filtering

### VLAN Filtering คืออะไร?

**Bridge VLAN Filtering** ทำให้ Bridge ทำงานเหมือน Managed Switch ที่รองรับ VLAN

```
Access Port ──── Bridge ──── Trunk Port
VLAN 10          │            VLAN 10,20,30
                 │
Access Port ──── ┘
VLAN 20
```

### เปิดใช้ VLAN Filtering

```routeros
# เปิด VLAN Filtering บน Bridge
/interface bridge set bridge1 vlan-filtering=yes

# สำคัญ: ต้องตั้งค่า VLAN Table ก่อนเปิด VLAN Filtering
# มิฉะนั้น Traffic จะถูก Block ทั้งหมด!
```

### ตั้งค่า VLAN Table

```routeros
# เพิ่ม VLAN ลงใน Bridge VLAN Table
/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=10 \
    tagged=ether1 \        # Tagged (Trunk) Ports
    untagged=ether2,ether3  # Untagged (Access) Ports

/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=20 \
    tagged=ether1 \
    untagged=ether4,ether5

/interface bridge vlan add \
    bridge=bridge1 \
    vlan-ids=30 \
    tagged=ether1 \
    untagged=ether6
```

### กำหนด PVID (Port VLAN ID)

```routeros
# PVID = Default VLAN สำหรับ Untagged Traffic
/interface bridge port set [find interface=ether2] pvid=10
/interface bridge port set [find interface=ether3] pvid=10
/interface bridge port set [find interface=ether4] pvid=20
/interface bridge port set [find interface=ether5] pvid=20
/interface bridge port set [find interface=ether6] pvid=30

# ether1 เป็น Trunk Port (ไม่ต้องกำหนด PVID)
```

### ตัวอย่างสมบูรณ์: Bridge VLAN Setup

```routeros
# Network Design:
# ether1 = Uplink/Trunk (tagged VLAN 10,20,30)
# ether2,3 = VLAN 10 (Management)
# ether4,5 = VLAN 20 (Staff)
# ether6,7 = VLAN 30 (Guest)

# Step 1: สร้าง Bridge
/interface bridge add name=bridge1 vlan-filtering=no protocol-mode=rstp

# Step 2: เพิ่ม Ports
/interface bridge port add interface=ether1 bridge=bridge1
/interface bridge port add interface=ether2 bridge=bridge1 pvid=10
/interface bridge port add interface=ether3 bridge=bridge1 pvid=10
/interface bridge port add interface=ether4 bridge=bridge1 pvid=20
/interface bridge port add interface=ether5 bridge=bridge1 pvid=20
/interface bridge port add interface=ether6 bridge=bridge1 pvid=30
/interface bridge port add interface=ether7 bridge=bridge1 pvid=30

# Step 3: สร้าง VLAN Interfaces สำหรับ IP
/interface vlan add interface=bridge1 vlan-id=10 name=vlan10
/interface vlan add interface=bridge1 vlan-id=20 name=vlan20
/interface vlan add interface=bridge1 vlan-id=30 name=vlan30

# Step 4: กำหนด VLAN Table
/interface bridge vlan add bridge=bridge1 vlan-ids=10 tagged=ether1,bridge1 untagged=ether2,ether3
/interface bridge vlan add bridge=bridge1 vlan-ids=20 tagged=ether1,bridge1 untagged=ether4,ether5
/interface bridge vlan add bridge=bridge1 vlan-ids=30 tagged=ether1,bridge1 untagged=ether6,ether7

# Step 5: ตั้งค่า IP บน VLAN Interfaces
/ip address add address=192.168.10.1/24 interface=vlan10
/ip address add address=192.168.20.1/24 interface=vlan20
/ip address add address=192.168.30.1/24 interface=vlan30

# Step 6: เปิด VLAN Filtering
/interface bridge set bridge1 vlan-filtering=yes
```

### ตรวจสอบ VLAN Filtering

```routeros
# ดู VLAN Table
/interface bridge vlan print

# ดู Port VLAN Settings
/interface bridge port print

# Test VLAN
/ping 192.168.10.1    # Test จาก VLAN 10
/ping 192.168.20.1    # Test จาก VLAN 20
```

---

## Bridge Hardware Offloading

### Hardware Offloading คืออะไร?

ใน RouterBoard ที่มี Switch-chip (เช่น CRS, CCR, RB4xx) การ Bridge สามารถทำผ่าน Hardware โดยตรง ไม่ผ่าน CPU ทำให้:
- Throughput สูงกว่ามาก (Wire Speed)
- CPU Load ต่ำกว่า
- Latency ต่ำกว่า

### ตรวจสอบว่า Hardware Offloading รองรับไหม

```routeros
# ดูว่า Port รองรับ HW Offloading ไหม
/interface ethernet print detail

# ดูสถานะ HW Offloading
/interface bridge port print detail

# สังเกตคอลัมน์ H (Hardware)
# H = Hardware Offloading กำลังทำงาน
```

### เปิดใช้ Hardware Offloading

```routeros
# เปิด HW Offloading บน Bridge
/interface bridge set bridge1 \
    fast-forward=yes \
    frame-types=admit-all

# เปิด HW Offloading บน Port
/interface bridge port set [find bridge=bridge1] hw=yes

# ตรวจสอบ
/interface bridge port print detail
# สังเกต hw=yes และ hw-offload=yes
```

### ข้อจำกัดของ Hardware Offloading

> **Warning:** Hardware Offloading มีข้อจำกัด:
> - บาง Features ไม่ทำงานเมื่อเปิด HW Offloading เช่น Bridge Filter, certain STP features
> - ขึ้นกับ Hardware Platform
> - ตรวจสอบ Wiki ของ RouterBoard นั้นๆ ก่อนใช้งาน

```routeros
# ตรวจสอบ Switch Chip Info
/interface ethernet switch print
/interface ethernet switch port print
```

---

## Bridge Monitoring

### ดูสถานะ Bridge

```routeros
# ดู Bridge Interfaces
/interface bridge print

# ดู Bridge แบบ Realtime
/interface bridge monitor bridge1

# ดู Port Statistics
/interface bridge port print detail

# ดู MAC Table (FDB)
/interface bridge host print

# ดู MAC Table แบบ Realtime
/interface bridge host print interval=5
```

### Bridge Host Table

```routeros
# ดู MAC ที่ Learn ใน Bridge
/interface bridge host print

# Output ตัวอย่าง:
# Flags: L - local, E - external
#  #   MAC-ADDRESS       VID  ON-INTERFACE  BRIDGE  AGE
#  0 L AA:BB:CC:DD:EE:01  -   ether2        bridge1  10s
#  1   AA:BB:CC:DD:EE:02  -   ether2        bridge1  45s
#  2   AA:BB:CC:DD:EE:03  -   ether3        bridge1  1m20s

# ค้นหา MAC เฉพาะ
/interface bridge host print where mac-address="AA:BB:CC:DD:EE:01"
```

### STP Monitoring

```routeros
# ดู STP State ของ Bridge
/interface bridge monitor bridge1

# Output ตัวอย่าง:
#           state: enabled
#     current-mac: AA:BB:CC:DD:EE:FF
# root-bridge-id: 8000.AA:BB:CC:DD:EE:FF
#   root-path-cost: 0
#        root-port: none
#    port-count: 4
# designated-port-count: 4

# ดู STP Status ของ Ports
/interface bridge port print detail
```

### Bridge Bandwidth Monitoring

```routeros
# Monitor Traffic บน Bridge
/interface monitor-traffic bridge1 interval=1

# ดู Statistics ของ Bridge
/interface print stats where type=bridge
```

---

## Lab: Office Network with Bridge

### Network Topology

```
Internet
    │
[Router/Firewall]
    │ ether1 (WAN)
    │
    ├── ether2 ── bridge1 ── Management VLAN 10 (192.168.10.0/24)
    │                    ├── Staff VLAN 20 (192.168.20.0/24)
    │                    └── Guest VLAN 30 (192.168.30.0/24)
    │
[CRS/Switch]
    │ Trunk (VLAN 10,20,30)
    │
    ├── Port2-4: VLAN 10 (Management)
    ├── Port5-8: VLAN 20 (Staff)
    └── Port9-12: VLAN 30 (Guest WiFi)
```

### Objectives

1. สร้าง Bridge พร้อม VLAN Filtering
2. แยก Traffic 3 VLAN ออกจากกัน
3. ทุก VLAN สามารถออก Internet ได้
4. VLAN Guest ไม่สามารถเข้าถึง VLAN อื่นได้
5. มี DHCP Server สำหรับแต่ละ VLAN

### Step 1: สร้าง Bridge

```routeros
# สร้าง Bridge
/interface bridge add \
    name=office-bridge \
    protocol-mode=rstp \
    vlan-filtering=no \
    comment="Office Network Bridge"

# ดูผลลัพธ์
/interface bridge print
```

### Step 2: เพิ่ม Ports

```routeros
# เพิ่ม Ports พร้อมกำหนด PVID
/interface bridge port add interface=ether2 bridge=office-bridge pvid=10 comment="Mgmt Port1"
/interface bridge port add interface=ether3 bridge=office-bridge pvid=10 comment="Mgmt Port2"
/interface bridge port add interface=ether4 bridge=office-bridge pvid=20 comment="Staff Port1"
/interface bridge port add interface=ether5 bridge=office-bridge pvid=20 comment="Staff Port2"
/interface bridge port add interface=ether6 bridge=office-bridge pvid=30 comment="Guest Port1"
/interface bridge port add interface=ether7 bridge=office-bridge pvid=30 comment="Guest Port2"

# Trunk Port ไปยัง Switch
/interface bridge port add interface=ether8 bridge=office-bridge comment="Trunk to Switch"
```

### Step 3: สร้าง VLAN Interfaces

```routeros
# สร้าง VLAN Interfaces
/interface vlan add name=vlan10-mgmt interface=office-bridge vlan-id=10
/interface vlan add name=vlan20-staff interface=office-bridge vlan-id=20
/interface vlan add name=vlan30-guest interface=office-bridge vlan-id=30

# ตั้งค่า IP บน VLAN Interfaces
/ip address add address=192.168.10.1/24 interface=vlan10-mgmt comment="Mgmt Gateway"
/ip address add address=192.168.20.1/24 interface=vlan20-staff comment="Staff Gateway"
/ip address add address=192.168.30.1/24 interface=vlan30-guest comment="Guest Gateway"
```

### Step 4: ตั้งค่า VLAN Table

```routeros
# VLAN Table
/interface bridge vlan add \
    bridge=office-bridge \
    vlan-ids=10 \
    tagged=ether8,office-bridge \
    untagged=ether2,ether3

/interface bridge vlan add \
    bridge=office-bridge \
    vlan-ids=20 \
    tagged=ether8,office-bridge \
    untagged=ether4,ether5

/interface bridge vlan add \
    bridge=office-bridge \
    vlan-ids=30 \
    tagged=ether8,office-bridge \
    untagged=ether6,ether7

# เปิด VLAN Filtering
/interface bridge set office-bridge vlan-filtering=yes
```

### Step 5: ตั้งค่า DHCP

```routeros
# สร้าง IP Pool สำหรับแต่ละ VLAN
/ip pool add name=pool-vlan10 ranges=192.168.10.100-192.168.10.200
/ip pool add name=pool-vlan20 ranges=192.168.20.100-192.168.20.200
/ip pool add name=pool-vlan30 ranges=192.168.30.100-192.168.30.200

# สร้าง DHCP Network
/ip dhcp-server network add \
    address=192.168.10.0/24 \
    gateway=192.168.10.1 \
    dns-server=8.8.8.8,8.8.4.4 \
    comment="VLAN 10 Management"

/ip dhcp-server network add \
    address=192.168.20.0/24 \
    gateway=192.168.20.1 \
    dns-server=8.8.8.8,8.8.4.4 \
    comment="VLAN 20 Staff"

/ip dhcp-server network add \
    address=192.168.30.0/24 \
    gateway=192.168.30.1 \
    dns-server=8.8.8.8 \
    comment="VLAN 30 Guest"

# สร้าง DHCP Server
/ip dhcp-server add name=dhcp-vlan10 interface=vlan10-mgmt address-pool=pool-vlan10 disabled=no
/ip dhcp-server add name=dhcp-vlan20 interface=vlan20-staff address-pool=pool-vlan20 disabled=no
/ip dhcp-server add name=dhcp-vlan30 interface=vlan30-guest address-pool=pool-vlan30 disabled=no
```

### Step 6: ตั้งค่า Firewall สำหรับ Guest Isolation

```routeros
# ป้องกัน Guest (VLAN30) เข้าถึง VLAN อื่น
/ip firewall filter add \
    chain=forward \
    src-address=192.168.30.0/24 \
    dst-address=192.168.10.0/24 \
    action=drop \
    comment="Block Guest to Management"

/ip firewall filter add \
    chain=forward \
    src-address=192.168.30.0/24 \
    dst-address=192.168.20.0/24 \
    action=drop \
    comment="Block Guest to Staff"

# อนุญาต Guest ออก Internet
/ip firewall filter add \
    chain=forward \
    src-address=192.168.30.0/24 \
    connection-state=new \
    action=accept \
    comment="Allow Guest to Internet"

# NAT สำหรับทุก VLAN
/ip firewall nat add \
    chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="NAT all VLANs"
```

### Step 7: ตรวจสอบ

```routeros
# ดู Bridge Status
/interface bridge print
/interface bridge port print

# ดู VLAN Table
/interface bridge vlan print

# ดู IP Addresses
/ip address print

# ดู DHCP Leases
/ip dhcp-server lease print

# ทดสอบ Connectivity
/ping 192.168.10.1
/ping 192.168.20.1
/ping 192.168.30.1
/ping 8.8.8.8
```

### Verification Results

```
Expected Output:

/interface bridge vlan print
Flags: X - disabled, D - dynamic
 #  BRIDGE        VLAN-IDS  CURRENT-TAGGED          CURRENT-UNTAGGED
 0  office-bridge 10        ether8,office-bridge    ether2,ether3
 1  office-bridge 20        ether8,office-bridge    ether4,ether5
 2  office-bridge 30        ether8,office-bridge    ether6,ether7
```

---

## Troubleshooting

### ปัญหาที่พบบ่อย

**1. หลังเปิด VLAN Filtering แล้ว Traffic หาย**
```routeros
# ตรวจสอบ VLAN Table ครบถ้วนไหม
/interface bridge vlan print

# ตรวจสอบ PVID ของแต่ละ Port
/interface bridge port print detail

# ตรวจสอบว่า Bridge Interface อยู่ใน Tagged list ของ VLAN ไหม
# (สำคัญมาก! ถ้าไม่มี Bridge Interface ใน Tagged, Router ไม่สามารถ Route ได้)
```

**2. Broadcast Storm**
```routeros
# ตรวจสอบ STP Status
/interface bridge port print detail

# ดู Loop Detection Logs
/log print where message~"topology"

# ชั่วคราว: เปิด STP บน Bridge
/interface bridge set bridge1 protocol-mode=rstp
```

**3. MAC Learning ไม่ทำงาน**
```routeros
# ตรวจสอบ Bridge Host Table
/interface bridge host print

# Clear Host Table
/interface bridge host remove [find !local]

# ตรวจสอบ Port Learning Mode
/interface bridge port print detail
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 12:

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| Bridge Concepts | Layer 2, ส่งโดยอ้างอิง MAC Address |
| Creating Bridges | `/interface bridge add` |
| Bridge Ports | เพิ่ม Interface เข้า Bridge ด้วย `/interface bridge port add` |
| STP | ป้องกัน Loop, Converge ช้า ~30-50s |
| RSTP | STP ที่เร็วขึ้น < 1s, แนะนำให้ใช้ |
| MSTP | หลาย STP Instance ต่อ VLAN Group |
| VLAN Filtering | ทำให้ Bridge ทำงานเหมือน Managed Switch |
| HW Offloading | ใช้ Switch-chip แทน CPU, ประสิทธิภาพสูง |

---

## แบบทดสอบ

1. Bridge ต่างจาก Router อย่างไร?
2. STP ป้องกันอะไร? อธิบายกระบวนการทำงาน
3. ทำไม RSTP ถึงเร็วกว่า STP?
4. PVID คืออะไร ใช้เพื่ออะไร?
5. ทำไมต้องใส่ `bridge1` ใน Tagged list ของ VLAN Table?

---

## Navigation

[← Part 11: Static Routing](part-011-static-routing.md) | [Part 13: Wireless Basics →](part-013-wireless-basics.md)

---

*MikroTik RouterOS Administration Course - Part 12*
*Last Updated: 2026*
