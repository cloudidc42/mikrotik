# Part 2: Hardware and Products

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner to Intermediate  
> **เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

1. [RouterBOARD Series Overview](#1-routerboard-series-overview)
2. [การเลือก Hardware ตามการใช้งาน](#2-การเลือก-hardware-ตามการใช้งาน)
3. [Spec ที่สำคัญ](#3-spec-ที่สำคัญ)
4. [SFP และ QSFP Modules](#4-sfp-และ-qsfp-modules)
5. [Lab Setup: Physical](#5-lab-setup-physical)
6. [Lab Setup: Virtual (GNS3 และ EVE-NG)](#6-lab-setup-virtual-gns3-และ-eve-ng)
7. [CHR (Cloud Hosted Router)](#7-chr-cloud-hosted-router)
8. [Pricing Guide](#8-pricing-guide)
9. [Firmware Update Process](#9-firmware-update-process)
10. [Summary](#10-summary)

---

## 1. RouterBOARD Series Overview

### 1.1 hAP Series (Home Access Point)

hAP series ออกแบบมาสำหรับ home และ small office:

| Model | CPU | RAM | Ports | WiFi | Price (USD) |
|-------|-----|-----|-------|------|-------------|
| hAP mini | 650 MHz | 32MB | 3x100M | 2.4GHz N | ~$25 |
| hAP lite | 650 MHz | 32MB | 4x100M | 2.4GHz N | ~$30 |
| hAP ac | 720 MHz | 128MB | 5x100M | 2.4+5GHz ac | ~$60 |
| hAP ac² | 716 MHz | 128MB | 5x Gig | 2.4+5GHz ac | ~$65 |
| hAP ac³ | 716 MHz | 256MB | 5x Gig | 2.4+5GHz ac | ~$75 |
| hAP ax² | 1.8 GHz | 256MB | 5x Gig | WiFi 6 | ~$89 |
| hAP ax³ | 1.8 GHz | 1GB | 5x Gig + SFP | WiFi 6 | ~$119 |

**คุณสมบัติ hAP ax³ (รุ่นล่าสุด):**
```
Hardware Specs:
├── CPU: Qualcomm IPQ-6010 (4-core ARM, 1.8GHz)
├── RAM: 1 GB DDR3
├── Storage: 128 MB NAND flash
├── Ethernet: 5x Gigabit + 1x SFP
├── WiFi: 802.11ax (WiFi 6) Dual Band
│   ├── 2.4GHz: 574 Mbps (2x2 MU-MIMO)
│   └── 5GHz: 2402 Mbps (2x2 MU-MIMO)
├── USB: 1x USB-A 3.0
├── Power: DC jack or PoE-in (802.3at)
└── Dimensions: 200 x 125 x 28 mm
```

### 1.2 RB Series (RouterBoard)

RB Series เป็น fixed-form factor routers ไม่มี WiFi built-in:

#### RB750 Family
```
RB750Gr3 (Hex):
├── CPU: MediaTek MT7621A (2-core MIPS, 880MHz)
├── RAM: 256MB DDR3
├── Ports: 5x Gigabit Ethernet
├── USB: 1x USB-A
├── License: Level 4
└── ราคา: ~$59

RB760iGS (Hex S):
├── CPU: MediaTek MT7621A
├── RAM: 256MB DDR3
├── Ports: 5x Gigabit + 1x SFP
├── License: Level 4
└── ราคา: ~$79
```

#### RB4011
```
RB4011iGS+RM:
├── CPU: Alpine AL21400 (4-core ARM, 1.4GHz)
├── RAM: 1GB DDR3
├── Ports: 10x Gigabit + 1x SFP+
├── Storage: 512MB NAND
├── License: Level 5
└── ราคา: ~$199
```

#### RB5009
```
RB5009UG+S+IN:
├── CPU: Marvell Armada 88F7040 (4-core ARM, 1.4GHz)
├── RAM: 1GB DDR4
├── Ports: 7x Gigabit + 1x 2.5G + 1x SFP+
├── USB: 1x USB-A 3.0
├── Storage: 1GB NAND
├── License: Level 5
└── ราคา: ~$169
```

### 1.3 CCR Series (Cloud Core Router)

CCR ออกแบบสำหรับ ISP และ Enterprise:

| Model | CPU | Cores | RAM | 1G Ports | SFP | SFP+ | QSFP | Price |
|-------|-----|-------|-----|----------|-----|------|------|-------|
| CCR1009-7G-1C-1S+ | Tilera | 9-core | 1GB | 7 | 1 | 1 | - | ~$450 |
| CCR1016-12G | Tilera | 16-core | 2GB | 12 | - | - | - | ~$650 |
| CCR1036-8G-2S+EM | Tilera | 36-core | 16GB | 8 | - | 2 | - | ~$1400 |
| CCR2004-1G-12S+2XS | ARM | 4-core | 4GB | 1 | - | 12 | 2 | ~$750 |
| CCR2116-12G-4S+ | ARM | 16-core | 16GB | 12 | - | 4 | - | ~$1800 |

**CCR2004 Detail:**
```
CCR2004-1G-12S+2XS:
├── CPU: Marvell Armada 88F7040 (4-core 1.4GHz)
├── RAM: 4GB DDR4
├── Storage: 128MB NAND
├── Ports:
│   ├── 1x Gigabit Ethernet
│   ├── 12x SFP+ (1G/10G)
│   └── 2x QSFP+ (100G)
├── LCD Panel: Yes (2x24 char)
├── Dual PSU: Optional (redundant)
├── License: Level 6
└── ราคา: ~$750
```

### 1.4 CRS Series (Cloud Router Switch)

CRS รวม routing และ switching ในอุปกรณ์เดียว:

#### CRS สำหรับ Access Layer
```
CRS326-24G-2S+RM:
├── CPU: ARM Cortex-A7 (800MHz)
├── RAM: 512MB DDR3
├── Ports: 24x Gigabit + 2x SFP+ (10G)
├── Switching Capacity: 52 Gbps
├── Forwarding Rate: 38.7 Mpps
├── VLAN: 4094
├── MAC Table: 16K
├── License: Level 5
└── ราคา: ~$299
```

#### CRS สำหรับ Core Layer
```
CRS354-48G-4S+2Q+RM:
├── CPU: ARM Cortex-A7
├── RAM: 1GB DDR3
├── Ports: 48x Gigabit + 4x SFP+ (10G) + 2x QSFP+ (40G)
├── Switching Capacity: 254.4 Gbps
├── License: Level 5
└── ราคา: ~$699

CRS518-16XS-2XQ-RM:
├── Ports: 16x SFP28 (25G) + 2x QSFP28 (100G)
├── Switching Capacity: 1.04 Tbps
├── ราคา: ~$2,500
```

### 1.5 Wireless / CPE Products

#### Outdoor Access Points
```
wAP ac:
├── 2x 2.4GHz (300Mbps) + 5GHz (867Mbps)
├── Outdoor weatherproof
├── PoE-in support
└── ราคา: ~$89

LHG 5 (Long range Point-to-Point):
├── 5GHz 802.11ac
├── Built-in 24.5 dBi antenna
├── Range: 15-20 km
└── ราคา: ~$79
```

---

## 2. การเลือก Hardware ตามการใช้งาน

### 2.1 Decision Matrix

```
Use Case Decision Tree:

                ┌─────────────────────┐
                │  What do you need?  │
                └──────────┬──────────┘
                           │
           ┌───────────────┼───────────────┐
           │               │               │
           ▼               ▼               ▼
    ┌─────────────┐  ┌──────────┐  ┌──────────────┐
    │  Home/SOHO  │  │   ISP    │  │  Enterprise  │
    └──────┬──────┘  └────┬─────┘  └──────┬───────┘
           │              │               │
           ▼              ▼               ▼
    ┌─────────────┐  ┌──────────┐  ┌──────────────┐
    │ hAP ax²/ax³ │  │ CCR2004  │  │ CCR2116 +    │
    │ RB5009      │  │ CCR2116  │  │ CRS354       │
    └─────────────┘  └──────────┘  └──────────────┘
```

### 2.2 Throughput Guide

| Router | Max Throughput | Recommended Use |
|--------|---------------|-----------------|
| hAP ax³ | ~1 Gbps | Home, SOHO |
| RB4011 | ~3.8 Gbps | SME, Branch |
| RB5009 | ~3 Gbps | SME, SOHO |
| CCR1009 | ~9 Gbps | Small ISP |
| CCR2004 | ~15 Gbps | Medium ISP |
| CCR2116 | ~40 Gbps | Large ISP/Enterprise |

> **Note:** Throughput ขึ้นกับ workload (routing only vs firewall+NAT+QoS ซึ่งลดลงมาก)

### 2.3 ตัวอย่าง: การเลือก Router สำหรับ ISP

**Scenario:** ISP ขนาดเล็กมี bandwidth 1 Gbps, customer 500 คน, PPPoE

```
Requirements:
- Throughput: 1+ Gbps
- PPPoE connections: 500
- NAT: ใช้ (Masquerade)
- Firewall: Basic rules
- QoS: Queue Tree

แนะนำ: CCR2004-1G-12S+2XS
Reason:
- 4-core ARM ทำ 15+ Gbps routing
- PPPoE: รองรับได้หลายพัน connections
- License Level 6: ไม่มี limit
- SFP+ ports: เหมาะต่อ uplink
```

---

## 3. Spec ที่สำคัญ

### 3.1 CPU Architecture

MikroTik ใช้ CPU หลายสถาปัตยกรรม:

| Architecture | รุ่นที่ใช้ | ข้อดี | ข้อเสีย |
|-------------|----------|-------|---------|
| MIPS | RB750, hAP lite | ราคาถูก | Performance จำกัด |
| ARM (32-bit) | hAP, CRS | Power efficient | Performance ปานกลาง |
| ARM (64-bit) | RB5009, CCR2004 | Performance ดี | ราคาสูงขึ้น |
| Tilera (MIPS) | CCR1036 | Multi-core ISP | เก่า แต่ยังใช้ |
| x86/x86_64 | CHR, Routerboard | Performance สูง | ไม่ใช่ router grade |

### 3.2 RAM Requirements

```
Minimum RAM ตาม workload:

Light routing (static routes):    32MB
Moderate (OSPF + small table):    64MB
Heavy (BGP full table):           512MB+
ISP (PPPoE + Hotspot):            256MB+
Enterprise (BGP + MPLS):          1GB+

Rule of thumb: ยิ่ง RAM มาก ยิ่งดีสำหรับ BGP tables
Full BGP table (900K+ routes) ต้องการ 512MB+ RAM
```

### 3.3 Storage

```
RouterOS Storage requirements:
- RouterOS 7.x: ~40-50 MB
- Logs: 1-100 MB (ตาม config)
- User Manager DB: ขึ้นกับจำนวน users
- Script files: น้อยมาก

Minimum: 64MB flash (เพียงพอ)
Recommended: 128MB+
```

### 3.4 Port Types

```
Port Standards ใน MikroTik:

Fast Ethernet (100M):
├── RJ45, Cat5 cable
├── ใช้ใน hAP mini, hAP lite (older)
└── ปัจจุบัน rare ใน new models

Gigabit Ethernet (1G):
├── RJ45, Cat5e/Cat6
├── Standard ใน ส่วนใหญ่ของ MikroTik
└── Support Auto-MDI/MDIX

2.5G Ethernet:
├── RJ45, Cat6 cable
├── ใช้ใน RB5009, hAP ax³
└── Backward compatible กับ 1G

SFP (1G):
├── Fiber transceiver (LC connector)
├── รองรับ SMF และ MMF
└── ใช้ใน RB760iGS, RB4011, hAP ax³

SFP+ (10G):
├── Fiber หรือ DAC cable
├── รองรับ 1G และ 10G
└── ใช้ใน CCR2004, CRS326

QSFP+ (40G):
├── 4x SFP+ (40G aggregate)
└── ใช้ใน CCR2004 (2x 40G ports)

QSFP28 (100G):
├── ใช้ใน CRS518, CCR2116 high-end
└── รองรับ 100G fiber

RJ45 Copper 10G:
├── ใช้ใน บางรุ่น CRS
└── Cat6A cable, สูงสุด 100m
```

---

## 4. SFP และ QSFP Modules

### 4.1 SFP Module Types

```
MikroTik SFP Modules:

S-85DLC05D (1G MMF):
├── Type: SFP
├── Speed: 1 Gbps
├── Fiber: Multimode (850nm)
├── Connector: LC
├── Distance: 550m
└── ราคา: ~$15

S-31DLC20D (1G SMF 20km):
├── Type: SFP
├── Speed: 1 Gbps
├── Fiber: Singlemode (1310nm)
├── Connector: LC
└── Distance: 20km

S+85DLC03D (10G MMF):
├── Type: SFP+
├── Speed: 10 Gbps
├── Fiber: Multimode (850nm)
├── Distance: 300m
└── ราคา: ~$35

S+31DLC10D (10G SMF 10km):
├── Type: SFP+
├── Speed: 10 Gbps
├── Fiber: Singlemode (1310nm)
└── Distance: 10km

S+DA0001 (10G DAC 1m):
├── Type: SFP+ Direct Attach Copper
├── Speed: 10 Gbps
├── Cable: Passive DAC
└── Distance: 1m
```

### 4.2 Third-party SFP Compatibility

```
Compatible vendors (ทดสอบแล้ว):
- Finisar/II-VI
- Cisco (marked MikroTik compatible)
- Ubiquiti UF-SM-10G
- Intel SFP+ modules

⚠️ Warning: บาง SFP vendor อาจไม่ compatible
ให้ทดสอบก่อนซื้อจำนวนมาก
```

---

## 5. Lab Setup: Physical

### 5.1 Minimum Lab สำหรับการเรียน

**ราคาประหยัด Lab:**
```
Option A (Basic - ~$150):
├── 1x RB750Gr3 (Hex) - Router
├── 1x hAP ac² - Wireless AP/Router
└── Network cables + switch

Option B (Intermediate - ~$400):
├── 2x RB5009 - Edge Routers
├── 1x CRS305 - Core Switch
├── SFP+ cables
└── USB serial adapter (optional)

Option C (Advanced - ~$1000):
├── 3x RB4011 - Routers
├── 1x CRS326 - Managed Switch
├── 1x hAP ax² - Wireless
└── CHR on VM (free)
```

### 5.2 Lab Topology แนะนำ

```
Physical Lab Topology:

    ┌─────────────────────────────────────────┐
    │           Lab Network                   │
    │                                         │
    │  ┌────────┐    ┌────────┐    ┌────────┐ │
    │  │Router 1│────│Switch  │────│Router 2│ │
    │  │RB5009  │    │CRS326  │    │RB5009  │ │
    │  │R1      │    │SW1     │    │R2      │ │
    │  └────────┘    └────┬───┘    └────────┘ │
    │    192.168          │                   │
    │    .10.x/24    ┌────┴───┐               │
    │               │Router 3 │               │
    │               │hAP ax²  │               │
    │               │R3       │               │
    │               └─────────┘               │
    │              192.168.30.x/24            │
    └─────────────────────────────────────────┘

Management VLAN (VLAN 99): 10.0.99.0/24
Lab Network 1: 192.168.10.0/24
Lab Network 2: 192.168.20.0/24
Lab Network 3: 192.168.30.0/24
```

### 5.3 Console Access

```
Serial Console Setup:
├── USB-to-Serial adapter (CH340 or FTDI)
├── Null-modem cable หรือ RJ45-DB9 adapter
├── Terminal settings:
│   ├── Speed: 115200 baud
│   ├── Data bits: 8
│   ├── Parity: None
│   ├── Stop bits: 1
│   └── Flow control: None

Linux command:
$ minicom -D /dev/ttyUSB0 -b 115200

macOS:
$ screen /dev/tty.usbserial-XXXX 115200

Windows:
Use PuTTY → Serial → COM port → 115200
```

---

## 6. Lab Setup: Virtual (GNS3 และ EVE-NG)

### 6.1 GNS3 Setup

**ขั้นตอนการ setup GNS3 สำหรับ MikroTik:**

**Step 1: ดาวน์โหลด Resources**
```bash
# ดาวน์โหลด RouterOS CHR image
wget https://download.mikrotik.com/routeros/7.15.3/chr-7.15.3.img.zip
unzip chr-7.15.3.img.zip

# หรือดาวน์โหลด GNS3 VM appliance
# ไปที่ https://mikrotik.com/download → GNS3 appliance
```

**Step 2: Import ใน GNS3**
```
GNS3 → File → Import Appliance
→ เลือก MikroTik CHR .gns3a file
→ Next → Select image file
→ เลือก chr-X.X.X.img
→ Install → Finish
```

**Step 3: GNS3 Topology Example**
```
                ┌─────────────────────────────────┐
                │        GNS3 Workspace           │
                │                                 │
    Cloud       │   ┌──────┐     ┌──────┐         │
    (Internet)──┼───│ R1   │─────│ R2   │         │
                │   │CHR   │     │CHR   │         │
                │   └──┬───┘     └──┬───┘         │
                │      │           │              │
                │      └─────┬─────┘              │
                │          ┌─┴──┐                 │
                │          │ R3 │                 │
                │          │CHR │                 │
                │          └────┘                 │
                └─────────────────────────────────┘
```

### 6.2 EVE-NG Setup

**EVE-NG สำหรับ MikroTik:**

**Step 1: เตรียม EVE-NG**
```bash
# ติดตั้ง EVE-NG Community (free)
# ดาวน์โหลดจาก https://www.eve-ng.net/downloads/

# หรือใช้ EVE-NG Pro (paid)
```

**Step 2: Upload RouterOS Image**
```bash
# SSH เข้า EVE-NG server
ssh root@eve-ng-server

# สร้างโฟลเดอร์สำหรับ MikroTik
mkdir -p /opt/unetlab/addons/qemu/mikrotik-7.15.3/

# Upload CHR image
# Rename เป็น virtioa.qcow2
cp chr-7.15.3.img /opt/unetlab/addons/qemu/mikrotik-7.15.3/virtioa.qcow2

# Convert format (ถ้าจำเป็น)
/opt/qemu/bin/qemu-img convert -f raw -O qcow2 \
    chr-7.15.3.img \
    /opt/unetlab/addons/qemu/mikrotik-7.15.3/virtioa.qcow2

# Fix permissions
/opt/unetlab/wrappers/unl_wrapper -a fixpermissions
```

**Step 3: สร้าง Lab ใน EVE-NG**
```
EVE-NG Web UI:
1. Login → New Lab
2. Add Node → MikroTik RouterOS
3. เลือก image version
4. กำหนด CPU, RAM
5. เชื่อมต่อ nodes
6. Start lab
```

### 6.3 VirtualBox Setup

สำหรับผู้เรียนที่ต้องการ simple lab:

```
Step 1: ดาวน์โหลด CHR
https://mikrotik.com/download
→ Cloud Hosted Router
→ เลือก VDI format

Step 2: สร้าง VM ใน VirtualBox
- Type: Linux
- Version: Other Linux (64-bit)
- RAM: 256MB minimum
- Disk: ใช้ VDI ที่ดาวน์โหลด
- Network:
  - Adapter 1: Bridged (WAN)
  - Adapter 2: Internal Network (LAN)
  - Adapter 3: Internal Network (optional)

Step 3: Boot และ configure
Default login: admin (no password)
```

---

## 7. CHR (Cloud Hosted Router)

### 7.1 CHR Overview

CHR คือ RouterOS ที่รันบน Virtual Machine โดยมี:
- Features เดียวกับ RouterBOARD
- ความสามารถ scale ได้ตาม VM resources
- รองรับ cloud providers ทุกรายใหญ่

### 7.2 Deploy CHR บน KVM/Proxmox

```bash
# ดาวน์โหลด CHR
wget https://download.mikrotik.com/routeros/7.15.3/chr-7.15.3.img.zip
unzip chr-7.15.3.img.zip

# Convert สำหรับ KVM
qemu-img convert -f raw -O qcow2 chr-7.15.3.img chr-7.15.3.qcow2

# Resize disk (optional)
qemu-img resize chr-7.15.3.qcow2 +4G

# สร้าง VM ด้วย virt-install
virt-install \
  --name MikroTik-CHR \
  --ram 512 \
  --vcpus 2 \
  --disk path=/var/lib/libvirt/images/chr-7.15.3.qcow2,format=qcow2 \
  --network bridge=br0,model=virtio \
  --network bridge=br1,model=virtio \
  --import \
  --os-variant linux2022 \
  --noautoconsole

# Connect console
virsh console MikroTik-CHR
```

### 7.3 Deploy CHR บน AWS

```bash
# Upload CHR image ไปยัง S3
aws s3 cp chr-7.15.3.img s3://my-bucket/mikrotik/

# Import เป็น AMI
aws ec2 import-image \
  --description "MikroTik CHR 7.15.3" \
  --disk-containers "Format=RAW,UserBucket={S3Bucket=my-bucket,S3Key=mikrotik/chr-7.15.3.img}"

# ติดตาม status
aws ec2 describe-import-image-tasks

# Launch instance จาก AMI
aws ec2 run-instances \
  --image-id ami-XXXXXXXXXX \
  --instance-type t3.micro \
  --key-name my-key \
  --security-group-ids sg-XXXXXXXXXX \
  --subnet-id subnet-XXXXXXXXXX
```

### 7.4 CHR License Activation

```bash
# บน CHR, request trial license
/system license renew level=p-unlimited

# หรือ activate paid license
/system license apply-key key=XXXX-XXXX-XXXX-XXXX

# ตรวจสอบ license
/system license print

# Output example:
#   software-id: ABCD-1234
#   upgradable-to: p-unlimited
#   level: p1
#   deadline: jan/01/2026
```

---

## 8. Pricing Guide

### 8.1 ราคา RouterBOARD (USD approximate)

```
Home/SOHO:
├── hAP mini         : $25
├── hAP lite         : $30
├── hAP ac²          : $65
├── hAP ax²          : $89
├── hAP ax³          : $119
├── RB750Gr3 (Hex)   : $59
├── RB760iGS (Hex S) : $79
└── RB5009UG+S+IN    : $169

SME/Enterprise:
├── RB4011iGS+RM     : $199
├── CCR1009-7G-1C-1S+: $449
├── CCR2004-1G-12S+  : $750
└── CCR2116-12G-4S+  : $1,800

Switches:
├── CRS305-1G-4S+IN  : $149
├── CRS326-24G-2S+RM : $299
├── CRS354-48G-4S+2Q+RM: $699
└── CRS518-16XS-2XQ  : $2,500
```

> **หมายเหตุ:** ราคาอาจแตกต่างตาม reseller และภาษีนำเข้า ราคาในไทยอาจสูงกว่า 20-40%

### 8.2 ราคา License

```
RouterOS Licenses:
├── Level 2 (WISP)     : $45
├── Level 3 (WISP AP)  : $95
├── Level 4 (WISP)     : $195
├── Level 5 (WISP)     : $250
└── Level 6 (Controller): $995

CHR Licenses:
├── P1 (1Gbps)         : $45/year
├── P10 (10Gbps)       : $95/year
└── P-Unlimited        : $250 perpetual
```

---

## 9. Firmware Update Process

### 9.1 ตรวจสอบ Version ปัจจุบัน

```bash
# ใน RouterOS CLI
/system resource print
# หรือ
/system package print

# Output:
#   NAME                VERSION    SCHEDULED
#   routeros            7.15.3
#   wireless            7.15.3
```

### 9.2 Update ผ่าน Internet (Automatic)

```bash
# ตรวจสอบ update ที่มีใหม่
/system package update check-for-updates

# Output:
#   channel: stable
#   installed-version: 7.15.2
#   latest-version: 7.15.3
#   status: New version is available

# ดาวน์โหลดและติดตั้ง
/system package update install

# Router จะ reboot อัตโนมัติ
```

### 9.3 Update แบบ Manual (Upload)

```bash
# Step 1: ดาวน์โหลด RouterOS package จาก
# https://mikrotik.com/download

# Step 2: Upload ไฟล์ไปที่ router
# ผ่าน Winbox: Files → ลาก .npk file ไปวาง
# ผ่าน FTP:
ftp admin@192.168.88.1
put routeros-7.15.3-arm64.npk

# ผ่าน SCP:
scp routeros-7.15.3-arm64.npk admin@192.168.88.1:/

# Step 3: Reboot
/system reboot
```

### 9.4 Update Channel Selection

```bash
# เลือก channel
/system package update set channel=stable
# หรือ
/system package update set channel=long-term
# หรือ
/system package update set channel=testing

# Channel types:
# stable    : แนะนำ สำหรับ production
# long-term : เวอร์ชัน stable เก่า, minimal changes
# testing   : beta, อย่าใช้ใน production
# development: alpha, ห้ามใช้ใน production
```

### 9.5 Downgrade

```bash
# ถ้า update แล้วมีปัญหา สามารถ downgrade ได้
# Step 1: ดาวน์โหลด version เก่าจาก changelog
# https://mikrotik.com/download/changelogs/routeros-7-changelog

# Step 2: Upload .npk file เก่า
scp routeros-7.14.3-arm64.npk admin@router:/

# Step 3: Downgrade
/system package downgrade

# Router จะ reboot และใช้ version เก่า
```

### 9.6 Firmware (Bootloader) Update

```bash
# RouterBOARD มี firmware แยกจาก RouterOS
/system routerboard print

# Output:
#   routerboard: yes
#   model: RB5009UG+S+IN
#   serial-number: XXXXXXXXXXXX
#   firmware-type: amt
#   factory-firmware: 7.6
#   current-firmware: 7.6
#   upgrade-firmware: 7.15

# Upgrade bootloader
/system routerboard upgrade

# หลัง reboot ตรวจสอบ
/system routerboard print
```

> **Warning:** ควร backup configuration ก่อน update ทุกครั้ง

### 9.7 Backup ก่อน Update

```bash
# สร้าง backup
/system backup save name=before-update

# Export config (text format)
/export file=before-update-config

# Download files
# Winbox: Files → เลือก backup/export file → Download
```

---

## 10. Summary

### สิ่งที่เรียนรู้ใน Part 2:

1. **RouterBOARD Series** มีหลายซีรีส์ตาม use case: hAP (home), RB (routing), CCR (ISP), CRS (switching)
2. **Hardware Selection** ขึ้นกับ throughput ที่ต้องการ และ feature set
3. **SFP/QSFP** ใช้สำหรับ fiber connectivity, MikroTik มี module ของตัวเองและ compatible กับ third-party
4. **Lab Setup** ทำได้ทั้ง physical (RouterBOARD) และ virtual (GNS3, EVE-NG, VirtualBox)
5. **CHR** ใช้สำหรับ cloud deployment มี license ราคาถูก
6. **Firmware Update** ควรทำผ่าน stable channel และ backup ก่อนเสมอ

### Quick Commands Reference

```bash
# ตรวจสอบ hardware info
/system resource print
/system routerboard print

# ตรวจสอบ version
/system package print

# Update check
/system package update check-for-updates

# Backup
/system backup save name=my-backup

# Reboot
/system reboot
```

---

**[⬅ Previous: Introduction to MikroTik](part-001-introduction-to-mikrotik.md)** | **[Next: RouterOS Installation ➡](part-003-routeros-installation.md)**
