# Part 1: Introduction to MikroTik

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner to Intermediate  
> **เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

1. [ประวัติและที่มาของ MikroTik](#1-ประวัติและที่มาของ-mikrotik)
2. [RouterOS คืออะไร](#2-routeros-คืออะไร)
3. [สถาปัตยกรรม RouterOS](#3-สถาปัตยกรรม-routeros)
4. [License Levels](#4-license-levels)
5. [Use Cases ต่างๆ](#5-use-cases-ต่างๆ)
6. [MikroTik Ecosystem](#6-mikrotik-ecosystem)
7. [ข้อดีและข้อเสีย](#7-ข้อดีและข้อเสีย)
8. [เปรียบเทียบกับคู่แข่ง](#8-เปรียบเทียบกับคู่แข่ง)
9. [Summary](#9-summary)

---

## 1. ประวัติและที่มาของ MikroTik

### 1.1 ก่อตั้งบริษัท

MikroTik (ออกเสียง: ไมโคร-ทิก) ก่อตั้งขึ้นในปี **1996** ที่กรุง **Riga ประเทศลัตเวีย** โดย:

- **John Trully** (ชาวอเมริกัน)
- **Arnis Riekstins** (ชาวลัตเวีย)

ในช่วงแรก บริษัทมุ่งเน้นการพัฒนา **software router** และ **wireless ISP (WISP)** systems เพื่อให้บริการอินเทอร์เน็ตในประเทศกำลังพัฒนา

> **ข้อเท็จจริงน่าสนใจ:** ลัตเวียในปี 1990s มีโครงสร้างพื้นฐานอินเทอร์เน็ตที่จำกัดมาก MikroTik จึงพัฒนา solution ราคาถูกที่ทำงานบน hardware ธรรมดา

### 1.2 Timeline สำคัญ

| ปี | เหตุการณ์ |
|-----|-----------|
| 1996 | ก่อตั้งบริษัท MikroTikls SIA |
| 1997 | เปิดตัว RouterOS เวอร์ชันแรก |
| 2002 | เปิดตัว RouterBOARD hardware ตัวแรก |
| 2006 | เปิดตัว Winbox GUI |
| 2010 | RouterOS v6 พร้อม IPv6 support |
| 2016 | เปิดตัว SwOS สำหรับ managed switches |
| 2018 | RouterOS v6.43 พร้อม MPLS improvements |
| 2021 | เปิดตัว RouterOS v7 stable |
| 2023 | RouterOS v7.x กลายเป็น mainstream |

### 1.3 ปัจจุบัน

ปัจจุบัน MikroTik:
- มีพนักงานกว่า **350 คน**
- จำหน่ายผลิตภัณฑ์ใน **160+ ประเทศ**
- มี **Mikrotik User Meeting (MUM)** จัดทั่วโลกทุกปี
- เป็น ISP ขนาดใหญ่ใน Latvia ด้วย

---

## 2. RouterOS คืออะไร

### 2.1 นิยาม

**RouterOS** คือ **Linux-based operating system** ที่พัฒนาโดย MikroTik โดยเฉพาะสำหรับ:
- Routing
- Switching
- Wireless networking
- VPN
- Firewall
- Bandwidth management

```
RouterOS Architecture:
┌─────────────────────────────────────┐
│          User Applications          │
│  (Winbox, WebFig, API, SSH, CLI)    │
├─────────────────────────────────────┤
│         RouterOS Services           │
│  Routing │ Firewall │ QoS │ VPN    │
├─────────────────────────────────────┤
│      Linux Kernel (modified)        │
├─────────────────────────────────────┤
│    Hardware Abstraction Layer       │
├─────────────────────────────────────┤
│     Hardware (RouterBOARD/x86)      │
└─────────────────────────────────────┘
```

### 2.2 เปรียบเทียบกับ OS อื่น

| Feature | RouterOS | Cisco IOS | Juniper JunOS | OpenWRT |
|---------|----------|-----------|---------------|---------|
| Base OS | Linux | Proprietary | BSD | Linux |
| GUI | Winbox/WebFig | Web UI | Juniper Web | LuCI |
| CLI | RouterOS CLI | IOS CLI | JunOS CLI | OpenWRT CLI |
| Cost | ถูก | แพงมาก | แพง | ฟรี |
| Learning Curve | ปานกลาง | สูง | สูงมาก | ต่ำ-ปานกลาง |
| Community | ใหญ่ | ใหญ่มาก | ปานกลาง | ใหญ่ |
| Enterprise Support | จำกัด | ดีมาก | ดีมาก | น้อย |
| ISP Usage | สูงมาก | สูงมาก | สูง | ปานกลาง |

### 2.3 RouterOS Versions

**RouterOS v6 (Legacy - EOL approaching)**
```
# ตรวจสอบ version
/system resource print
# Output:
#   version: 6.49.14 (stable)
```

**RouterOS v7 (Current - แนะนำ)**
```
# ตรวจสอบ version
/system resource print
# Output:
#   version: 7.15.3 (stable)
```

> **Warning:** RouterOS v6 และ v7 มีความแตกต่างในการ syntax บางส่วน เช่น OSPF, BGP configuration ที่ใน v7 ถูก redesign ใหม่ทั้งหมด

---

## 3. สถาปัตยกรรม RouterOS

### 3.1 Core Components

```
┌────────────────────────────────────────────────────────┐
│                    RouterOS v7                         │
├──────────────┬─────────────────┬──────────────────────┤
│   Routing    │    Firewall     │    Management        │
│  ─────────   │   ──────────   │   ──────────────     │
│  • Static    │   • Filter      │   • Winbox           │
│  • OSPF      │   • NAT         │   • WebFig           │
│  • BGP       │   • Mangle      │   • SSH/Telnet       │
│  • RIP       │   • Raw         │   • API              │
│  • MPLS      │   • Connection  │   • SNMP             │
├──────────────┼─────────────────┼──────────────────────┤
│   Tunnels    │   Wireless      │    Services          │
│  ─────────   │   ─────────    │   ──────────         │
│  • PPPoE     │   • 802.11a/b  │   • DHCP             │
│  • L2TP      │   /g/n/ac/ax   │   • DNS              │
│  • SSTP      │   • CAPsMAN    │   • NTP              │
│  • WireGuard │   • Nstreme    │   • RADIUS           │
│  • OpenVPN   │   • 802.11r    │   • Hotspot          │
│  • IPsec     │                 │   • User Manager     │
├──────────────┴─────────────────┴──────────────────────┤
│              Linux Kernel (Modified)                   │
│         Packet Processing Pipeline                     │
└────────────────────────────────────────────────────────┘
```

### 3.2 Packet Flow

เมื่อ packet เข้ามาใน RouterOS จะผ่าน pipeline ดังนี้:

```
Incoming Packet
      │
      ▼
┌──────────────┐
│  Interface   │  ← Physical/Virtual interface
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  RAW Table   │  ← Prerouting (ก่อน connection tracking)
│  Prerouting  │
└──────┬───────┘
       │
       ▼
┌──────────────┐
│  Connection  │  ← State tracking (NEW, ESTABLISHED, RELATED)
│  Tracking    │
└──────┬───────┘
       │
       ├─────────────────────┐
       │                     │
       ▼                     ▼
┌──────────────┐    ┌────────────────┐
│  Firewall    │    │    Mangle      │
│  Prerouting  │    │   Prerouting   │
└──────┬───────┘    └───────┬────────┘
       │                    │
       └──────────┬──────────┘
                  │
                  ▼
           ┌─────────────┐
           │   Routing   │  ← Route lookup
           │  Decision   │
           └──────┬──────┘
                  │
          ┌───────┴──────────┐
          │                  │
          ▼                  ▼
   ┌─────────────┐   ┌──────────────┐
   │  Local     │   │  Forward     │
   │  Process   │   │  Process     │
   └─────────────┘   └──────┬───────┘
                            │
                            ▼
                     ┌─────────────┐
                     │  Postrouting│  ← NAT, Mangle
                     └──────┬──────┘
                            │
                            ▼
                     ┌─────────────┐
                     │  Egress     │  ← Queuing, Shaping
                     │  Interface  │
                     └─────────────┘
```

### 3.3 Menu Structure

RouterOS CLI จัดระเบียบด้วย hierarchical menu:

```bash
/                          # Root
├── /interface             # Network interfaces
│   ├── /interface ethernet
│   ├── /interface wireless
│   ├── /interface bridge
│   └── /interface vlan
├── /ip                    # IPv4 settings
│   ├── /ip address        # IP assignments
│   ├── /ip route          # Routing table
│   ├── /ip firewall       # Firewall rules
│   ├── /ip dhcp-server    # DHCP server
│   ├── /ip dhcp-client    # DHCP client
│   ├── /ip dns            # DNS settings
│   └── /ip nat            # NAT rules (v6) / firewall nat (v7)
├── /ipv6                  # IPv6 settings
├── /routing               # Routing protocols
│   ├── /routing ospf
│   ├── /routing bgp
│   └── /routing rip
├── /ppp                   # PPP protocols
├── /mpls                  # MPLS
├── /system                # System settings
│   ├── /system clock
│   ├── /system identity
│   ├── /system resource
│   └── /system scheduler
└── /tool                  # Diagnostic tools
    ├── /tool ping
    ├── /tool traceroute
    └── /tool bandwidth-test
```

---

## 4. License Levels

### 4.1 License Overview

RouterOS มี License levels ทั้งหมด 7 ระดับ (0-6):

| Level | ชื่อ | ราคา (approx) | Use Case |
|-------|------|---------------|----------|
| 0 | Trial/Free | $0 | ทดสอบ 24 ชั่วโมง |
| 1 | Free (old) | $0 | Limited (ไม่แนะนำ) |
| 2 | WISP | $45 | Home/SOHO |
| 3 | WISP AP | $95 | AP สำหรับ WISP |
| 4 | WISP | $195 | Small ISP |
| 5 | WISP | $250 | Medium ISP |
| 6 | Controller | $995 | Large ISP/Enterprise |

### 4.2 Feature Comparison by Level

| Feature | L1 | L2 | L3 | L4 | L5 | L6 |
|---------|----|----|----|----|----|----|
| Wireless AP | - | 1 | 1 | ∞ | ∞ | ∞ |
| Hotspot Users | - | 1 | 1 | 200 | 500 | ∞ |
| PPPoE Tunnels | - | 1 | 200 | 200 | 500 | ∞ |
| OVPN Tunnels | - | - | - | - | - | ∞ |
| Active Users | - | - | 200 | 200 | 500 | ∞ |
| RADIUS Client | - | + | + | + | + | + |
| User Manager | - | - | - | - | + | + |
| EoIP Tunnels | - | - | - | - | - | ∞ |

> **Note:** RouterBOARD hardware มักมา pre-licensed ตาม model หมายความว่าไม่ต้องซื้อ license เพิ่ม

### 4.3 CHR License

**Cloud Hosted Router (CHR)** มี license model ต่างออกไป:

```
CHR License Types:
├── Free Trial     : ความเร็วสูงสุด 1 Mbps
├── P1  ($45/yr)   : ความเร็วสูงสุด 1 Gbps
├── P10 ($95/yr)   : ความเร็วสูงสุด 10 Gbps
├── P-Unlimited    : ความเร็วไม่จำกัด (perpetual)
└── 60-day Trial   : ความเร็วไม่จำกัด (ทดสอบ)
```

### 4.4 ตรวจสอบ License

```bash
# ดู license level ปัจจุบัน
/system license print

# Output:
#   software-id: XXXX-XXXX
#   level: 6
#   features: hotspot user-manager
```

---

## 5. Use Cases ต่างๆ

### 5.1 Home User

สำหรับ home network MikroTik เหมาะสำหรับ:
- Internet sharing (NAT/Masquerade)
- Parental control (web filtering)
- Quality of Service (QoS)
- VPN client (WireGuard, L2TP)
- Guest WiFi

**แนะนำ Hardware:** hAP ac³, hAP ax², RB750Gr3

```
Home Network Topology:
┌─────────────────────────────────────────────┐
│                Internet                     │
└──────────────────┬──────────────────────────┘
                   │ WAN (DHCP/PPPoE)
          ┌────────┴────────┐
          │   MikroTik      │  hAP ac³
          │   Router        │  192.168.1.1/24
          └────────┬────────┘
                   │ LAN 192.168.1.0/24
         ┌─────────┼──────────┐
         │         │          │
    ┌────┴──┐ ┌────┴──┐ ┌────┴──┐
    │  PC   │ │Laptop │ │Mobile │
    │.1.10  │ │.1.20  │ │.1.30  │
    └───────┘ └───────┘ └───────┘
```

### 5.2 Small Business / SOHO

```
SOHO Network Topology:
┌─────────────────────────────────────────────────┐
│                   Internet                      │
└────────────────────┬────────────────────────────┘
                     │ WAN
            ┌────────┴────────┐
            │   MikroTik      │  Level 4 License
            │   Router/FW     │  RB4011 or similar
            └──────┬──────────┘
                   │
        ┌──────────┼──────────────┐
        │          │              │
   ┌────┴───┐  ┌───┴────┐  ┌─────┴────┐
   │  LAN   │  │  DMZ   │  │  WiFi    │
   │192.168.│  │10.0.0.0│  │172.16.0.0│
   │1.0/24  │  │/24     │  │/24       │
   └────────┘  └────────┘  └──────────┘
```

### 5.3 ISP (Internet Service Provider)

MikroTik เป็นที่นิยมสูงมากในกลุ่ม ISP ขนาดกลาง-เล็ก เนื่องจาก:
- ราคาถูกกว่า Cisco/Juniper มาก
- รองรับ PPPoE สำหรับ subscriber management
- Hotspot สำหรับ public WiFi
- Bandwidth management ด้วย Queue Tree

```
ISP Core Network Topology:
                    ┌──────────────────┐
                    │  Upstream ISP    │
                    │  (Transit/IX)    │
                    └────────┬─────────┘
                             │ BGP
                    ┌────────┴─────────┐
                    │    Core Router   │  CCR2004-1G-12S+2XS
                    │    (BGP/MPLS)    │  RouterOS L6
                    └────────┬─────────┘
                             │
              ┌──────────────┴──────────────┐
              │                             │
     ┌────────┴────────┐          ┌─────────┴────────┐
     │  Distribution   │          │  Distribution    │
     │  Router 1       │          │  Router 2        │
     │  CCR1009        │          │  CCR1009         │
     └────────┬────────┘          └─────────┬────────┘
              │                             │
    ┌─────────┼──────────┐       ┌──────────┼──────────┐
    │         │          │       │          │          │
 ┌──┴──┐  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐  ┌──┴──┐
 │DSLAM│  │OLT  │  │BRAS │  │DSLAM│  │OLT  │  │BRAS │
 │     │  │     │  │PPPoE│  │     │  │     │  │PPPoE│
 └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘  └──┬──┘
    │        │         │         │         │        │
 Customers Customers Hotspot  Customers Customers Hotspot
```

### 5.4 Enterprise Network

```
Enterprise Topology:
┌────────────────────────────────────────────────────────┐
│                     Internet                           │
└─────────────────────────┬──────────────────────────────┘
                          │
               ┌──────────┴──────────┐
               │   Edge/Border       │  CCR2116-12G-4S+
               │   Router (BGP)      │
               └──────────┬──────────┘
                          │
          ┌───────────────┼───────────────┐
          │               │               │
   ┌──────┴──────┐ ┌──────┴──────┐ ┌──────┴──────┐
   │  Core SW    │ │  Firewall   │ │  DMZ        │
   │  CRS326     │ │  Cluster    │ │  Services   │
   └──────┬──────┘ └─────────────┘ └─────────────┘
          │
    ┌─────┼────────────────┐
    │     │                │
┌───┴──┐ ┌┴──────┐ ┌───────┴──┐
│Floor │ │Server │ │  WiFi    │
│Switch│ │VLAN   │ │  CAPsMAN │
│CRS   │ │       │ │          │
└──────┘ └───────┘ └──────────┘
```

---

## 6. MikroTik Ecosystem

### 6.1 RouterBOARD Series

MikroTik ผลิต hardware หลายซีรีส์:

#### hAP Series (Home Access Point)
```
รุ่นยอดนิยม:
- hAP ac³    : ราคาถูก, 802.11ac, เหมาะ home
- hAP ax²    : WiFi 6, เหมาะ home/office
- hAP ac²    : เก่า แต่ยังใช้ได้ดี
```

#### RB Series (RouterBoard)
```
- RB750Gr3  : 5-port Gigabit, basic routing
- RB4011    : 10-port, SFP+, สำหรับ SME
- RB5009    : 8-port Gig + SFP+, dual PSU option
```

#### CCR Series (Cloud Core Router)
```
- CCR1036-8G-2S+EM : 36-core Tilera, ISP grade
- CCR2004-1G-12S+2XS : 4-core ARM, high throughput
- CCR2116-12G-4S+   : 16-core ARM, datacenter
```

#### CRS Series (Cloud Router Switch)
```
- CRS305-1G-4S+IN : 4x SFP+, aggregation
- CRS326-24G-2S+  : 24x Gig + 2x SFP+
- CRS354-48G-4S+2Q+RM : 48-port datacenter
```

#### Wireless Series
```
- wAP ac   : Outdoor AP, 802.11ac
- cAP ac   : Ceiling mount AP
- mAP      : Indoor/outdoor small AP
- Cube     : CPE for WISP
- SXT      : PTMP sector antenna
- LHG      : Long-range point-to-point
```

### 6.2 SwOS (Switch OS)

SwOS คือ OS แยกต่างหากสำหรับ **managed switches** ที่ไม่ได้รัน RouterOS:

```
SwOS Features:
- Web-based management
- VLAN configuration
- Port mirroring
- RSTP/STP
- Link aggregation (LAG/LACP)
- Bandwidth limiting
- Port isolation
- MAC table
```

> **Note:** SwOS switches ไม่รองรับ routing หรือ firewall เหมือน RouterOS

### 6.3 CHR (Cloud Hosted Router)

CHR คือ RouterOS ที่ออกแบบให้รันบน Virtual Machine:

```
Supported Platforms:
├── VMware ESXi/vSphere
├── Microsoft Hyper-V
├── VirtualBox
├── KVM/QEMU
├── Proxmox VE
├── Xen
└── Cloud Providers:
    ├── Amazon AWS
    ├── Google Cloud Platform
    ├── Microsoft Azure
    ├── DigitalOcean
    └── Vultr, Linode, etc.
```

**ดาวน์โหลด CHR:**
```
https://mikrotik.com/download
→ Cloud Hosted Router (CHR)
→ เลือก format: VMDK, OVF, RAW, VHD
```

### 6.4 MikroTik Certifications

| Certification | ย่อ | ระดับ | เนื้อหา |
|--------------|-----|-------|---------|
| MikroTik Certified Network Associate | MTCNA | พื้นฐาน | Basic routing, wireless |
| MikroTik Certified Routing Engineer | MTCRE | กลาง | Advanced routing |
| MikroTik Certified Traffic Control Engineer | MTCTCE | กลาง | QoS, queuing |
| MikroTik Certified Wireless Engineer | MTCWE | กลาง | Wireless, CAPsMAN |
| MikroTik Certified Security Engineer | MTCSE | กลาง | Firewall, VPN |
| MikroTik Certified User Management Engineer | MTCUME | กลาง | Hotspot, RADIUS |
| MikroTik Certified Inter-Networking Engineer | MTCINE | สูง | BGP, MPLS |
| MikroTik Certified Enterprise Wireless Engineer | MTCEWE | สูง | Enterprise WiFi |

---

## 7. ข้อดีและข้อเสีย

### 7.1 ข้อดี (Pros)

```
✅ ราคาถูก
   - RouterBOARD hardware ราคา $30-500
   - License ราคา $45-995 (ถูกกว่า Cisco มาก)
   - Total cost of ownership ต่ำ

✅ Features ครบครัน
   - OSPF, BGP, RIP routing
   - MPLS, VPLS
   - WireGuard, IPsec, L2TP, SSTP VPN
   - Hotspot, User Manager
   - CAPsMAN wireless controller
   - QoS, Queue Tree
   - Firewall, NAT
   - VLAN, Bridging
   - IPv6 complete support

✅ Active Community
   - Forum.mikrotik.com มีผู้ใช้หลายแสนคน
   - YouTube tutorials มากมาย
   - Wiki documentation ดีมาก

✅ Winbox GUI
   - Interface ที่ใช้งานง่าย
   - Real-time monitoring
   - Drag-and-drop interface list

✅ Regular Updates
   - อัปเดตบ่อย (2-4 สัปดาห์)
   - Security patches รวดเร็ว

✅ Scripting
   - Built-in scripting engine
   - Scheduler
   - Event-based scripts
```

### 7.2 ข้อเสีย (Cons)

```
❌ Learning Curve
   - CLI syntax แตกต่างจาก Cisco IOS
   - v6 → v7 migration ซับซ้อน
   - Documentation บางส่วน outdated

❌ Enterprise Support
   - ไม่มี 24/7 TAC support แบบ Cisco
   - Bug fixes ขึ้นกับ community report
   - SLA ไม่ชัดเจน

❌ Hardware Quality
   - บางรุ่นมีปัญหา power supply
   - Plastic case บางรุ่นไม่ทนทาน
   - Fan noise ใน CCR series

❌ Software Bugs
   - บางครั้ง stable release มี bugs
   - Beta/RC versions อาจไม่เสถียร
   - ควร test ก่อน production

❌ CLI Consistency
   - บาง command ไม่ consistent ระหว่าง menu
   - v7 เปลี่ยน syntax หลายอย่าง
```

---

## 8. เปรียบเทียบกับคู่แข่ง

### 8.1 MikroTik vs Cisco

| Criteria | MikroTik | Cisco |
|----------|----------|-------|
| **Price (Entry)** | $50-200 | $500-2000+ |
| **Enterprise Router** | $500-2000 | $5,000-50,000+ |
| **License Model** | Perpetual (mostly) | Subscription (DNA) |
| **CLI** | RouterOS CLI | IOS/IOS-XE/NX-OS |
| **GUI** | Winbox (excellent) | Web UI (limited) |
| **Routing Protocols** | OSPF, BGP, RIP, MPLS | OSPF, BGP, EIGRP, IS-IS, MPLS |
| **Support** | Community + Forum | 24/7 TAC |
| **Certifications** | MTCNA, MTCRE, etc. | CCNA, CCNP, CCIE |
| **Market Share (ISP)** | สูงมากในตลาด WISP | สูงใน enterprise |
| **Documentation** | Wiki + Forum | Cisco.com (extensive) |

### 8.2 MikroTik vs Juniper

| Criteria | MikroTik | Juniper |
|----------|----------|---------|
| **Price** | ถูก | แพงมาก |
| **OS** | RouterOS | Junos |
| **CLI Philosophy** | Task-focused | Config hierarchy |
| **Automation** | Scripts, API | NETCONF, REST |
| **Target Market** | SME, WISP, Home | Service Provider, Enterprise |
| **Throughput** | สูงถึง 40+ Gbps (CCR) | สูงมาก (Tbps) |

### 8.3 MikroTik vs Ubiquiti

| Criteria | MikroTik | Ubiquiti |
|----------|----------|---------|
| **Price** | ใกล้เคียงกัน | ใกล้เคียงกัน |
| **GUI** | Winbox (powerful) | UniFi (user-friendly) |
| **CLI** | RouterOS (advanced) | EdgeOS (Vyatta-based) |
| **Wireless** | ดี | ดีมาก (UniFi) |
| **Management** | Per-device | Centralized (UniFi Controller) |
| **Learning Curve** | ปานกลาง | ต่ำ (UniFi) |
| **Community** | ใหญ่มาก | ใหญ่ |
| **Enterprise Features** | ครบ | จำกัด |

### 8.4 When to Choose MikroTik

```
✅ เลือก MikroTik เมื่อ:
- Budget จำกัด แต่ต้องการ features ครบ
- เป็น WISP หรือ small-medium ISP
- ต้องการ Hotspot/User Manager
- ต้องการ Scripting/Automation
- ทีม Admin มีความรู้ networking

❌ อย่าเลือก MikroTik เมื่อ:
- ต้องการ 24/7 enterprise support (เลือก Cisco/Juniper)
- Team ไม่มี technical knowledge
- ต้องการ plug-and-play (เลือก Ubiquiti UniFi)
- Scale ใหญ่มากระดับ datacenter backbone
```

---

## 9. Summary

### สิ่งที่เรียนรู้ใน Part 1:

1. **MikroTik** ก่อตั้งปี 1996 ที่ Latvia พัฒนา RouterOS บน Linux
2. **RouterOS** คือ full-featured network OS ที่ราคาประหยัด
3. **License Levels 0-6** กำหนด features ที่ใช้ได้ โดย hardware มักมาพร้อม license
4. **Use Cases** ครอบคลุม Home, SOHO, ISP, Enterprise
5. **Ecosystem** ประกอบด้วย RouterBOARD, SwOS switches, CHR, Wireless APs
6. **ข้อดี** คือราคา, features, community **ข้อเสีย** คือ enterprise support จำกัด

### Quick Reference

```bash
# Commands ที่ควรรู้จาก Part 1
/system resource print          # ดู resource และ version
/system license print           # ดู license level
/system identity print          # ดูชื่อ router
/system clock print             # ดูเวลา
```

### การเตรียมตัวสำหรับ Part 2

ใน **Part 2** เราจะเรียน:
- RouterBOARD hardware series ต่างๆ อย่างละเอียด
- การเลือก hardware ตามการใช้งาน
- Lab setup ทั้ง physical และ virtual (GNS3, EVE-NG)

---

## อ้างอิงเพิ่มเติม

- [MikroTik Official Wiki](https://wiki.mikrotik.com)
- [MikroTik Forum](https://forum.mikrotik.com)
- [MikroTik Downloads](https://mikrotik.com/download)
- [RouterOS Changelog](https://mikrotik.com/download/changelogs)
- [MikroTik Training](https://mt.lv/training)
- [MikroTik User Meeting Videos](https://mum.mikrotik.com)

---

**[⬅ Back to Index](../README.md)** | **[Next Part: Hardware and Products ➡](part-002-hardware-and-products.md)**
