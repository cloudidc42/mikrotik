# Part 3: RouterOS Installation

> **หลักสูตร:** MikroTik Network Engineering  
> **ระดับ:** Beginner  
> **เวลาเรียน:** 2-3 ชั่วโมง

---

## สารบัญ

1. [Download RouterOS](#1-download-routeros)
2. [Netinstall ขั้นตอนละเอียด](#2-netinstall-ขั้นตอนละเอียด)
3. [Installation บน VMware/VirtualBox](#3-installation-บน-vmwarevirtualbox)
4. [CHR Installation](#4-chr-installation)
5. [First Boot Configuration](#5-first-boot-configuration)
6. [Default Credentials และ Security](#6-default-credentials-และ-security)
7. [Network Interface Setup เบื้องต้น](#7-network-interface-setup-เบื้องต้น)
8. [License Activation](#8-license-activation)
9. [RouterOS 7.x vs 6.x Differences](#9-routeros-7x-vs-6x-differences)
10. [Upgrade/Downgrade Procedures](#10-upgradedowngrade-procedures)
11. [Summary และ Lab Exercise](#11-summary-และ-lab-exercise)

---

## 1. Download RouterOS

### 1.1 เว็บไซต์ดาวน์โหลด

```
URL: https://mikrotik.com/download

หน้า Download จะแสดง:
├── RouterOS
│   ├── Stable (แนะนำ)
│   ├── Long-term
│   └── Testing (beta)
├── Cloud Hosted Router (CHR)
├── Winbox
└── Netinstall
```

### 1.2 Package Types

**RouterOS มา 2 แบบ:**

```
1. Complete Package (.npk):
   routeros-7.15.3-arm.npk      ← สำหรับ ARM router
   routeros-7.15.3-arm64.npk    ← สำหรับ ARM64 router
   routeros-7.15.3-mipsbe.npk   ← สำหรับ MIPS router
   routeros-7.15.3-mmips.npk    ← สำหรับ MMIPS router
   routeros-7.15.3-x86.npk      ← สำหรับ x86 (CHR/PC)
   routeros-7.15.3-tile.npk     ← สำหรับ Tilera (CCR)

2. Extra Packages (optional):
   advanced-tools-7.15.3-arm.npk
   calea-7.15.3-arm.npk
   container-7.15.3-arm.npk     ← Docker-like containers
   dude-7.15.3-arm.npk          ← Network monitoring tool
   gps-7.15.3-arm.npk
   iot-7.15.3-arm.npk
   lora-7.15.3-arm.npk
   rose-storage-7.15.3-arm.npk
   ups-7.15.3-arm.npk
   user-manager-7.15.3-arm.npk  ← RADIUS/Hotspot user management
   wireless-7.15.3-arm.npk      ← WiFi support
   zerotier-7.15.3-arm.npk
```

### 1.3 ตรวจสอบ Architecture ของ Router

```bash
# ใน RouterOS CLI
/system resource print

# ดูที่ 'architecture-name' field
# ตัวอย่าง output:
#   uptime: 1d4h30m
#   version: 7.15.3 (stable)
#   build-time: Sep/25/2024 12:00:00
#   free-memory: 850.3MiB
#   total-memory: 1024.0MiB
#   cpu: ARM64
#   cpu-count: 4
#   cpu-frequency: 1400MHz
#   cpu-load: 3%
#   free-hdd-space: 850.0MiB
#   total-hdd-space: 1000.0MiB
#   architecture-name: arm64   ← ดูตรงนี้
#   board-name: RB5009UG+S+IN
#   platform: MikroTik
```

---

## 2. Netinstall ขั้นตอนละเอียด

### 2.1 Netinstall คืออะไร

**Netinstall** คือโปรแกรมสำหรับ:
- ติดตั้ง RouterOS ใหม่ (fresh install)
- Recovery เมื่อ router ใช้งานไม่ได้
- Reset router กลับเป็น default

> **เมื่อต้องใช้ Netinstall:**
> - Router boot ไม่ขึ้น
> - ลืม password
> - RouterOS corrupt
> - ต้องการลง version ใหม่ clean

### 2.2 Requirements

```
Netinstall Requirements:
├── PC ที่รัน Windows (Netinstall เป็น Windows app)
│   └── หรือใช้ Wine บน Linux/macOS
├── Ethernet cable (ต่อ PC → Port 1 ของ Router)
├── Netinstall application
│   └── ดาวน์โหลดจาก mikrotik.com/download
├── RouterOS .npk file
│   └── ตรงกับ architecture ของ router
└── Static IP บน PC (จะตั้งในขั้นตอน)
```

### 2.3 ขั้นตอน Netinstall แบบละเอียด

**Step 1: เตรียม PC**

```
1. ปิด Windows Firewall ชั่วคราว
   Control Panel → Windows Firewall → Turn Off

2. ตั้ง Static IP บน network adapter
   IP: 192.168.88.2
   Subnet: 255.255.255.0
   Gateway: (ว่างไว้)
   DNS: (ว่างไว้)
```

**Step 2: เปิด Netinstall**

```
1. เปิดโปรแกรม Netinstall
2. เลือก network interface ที่เชื่อมต่อ Router
3. ตั้ง "Routers/Clients IP: 192.168.88.3"
4. กด "Net Booting" → เปิด ON
```

**Step 3: Boot Router เข้า Netinstall Mode**

```
Method A - Reset Button:
1. ถอดไฟ Router
2. กด Reset button (หลัง)
3. ใส่ไฟ พร้อมกดค้าง Reset button
4. รอ LED กระพริบ (ประมาณ 5 วินาที)
5. ปล่อย Reset button

Method B - RouterOS CLI (ถ้า login ได้):
/system routerboard settings set boot-device=try-ethernet-once-then-nand

แล้ว Reboot
/system reboot
```

**Step 4: Netinstall ตรวจพบ Router**

```
Netinstall จะแสดง:
MAC Address: XX:XX:XX:XX:XX:XX
RouterBOARD: RB5009UG+S+IN

กด "Browse" เลือก .npk file
เลือก packages ที่ต้องการ:
☑ routeros-7.15.3-arm64.npk
☑ wireless-7.15.3-arm64.npk  (ถ้าต้องการ WiFi)
☑ user-manager-7.15.3-arm64.npk (ถ้าต้องการ)
```

**Step 5: ติดตั้ง**

```
กด "Install"
รอประมาณ 2-5 นาที
Router จะ reboot อัตโนมัติ
```

**Step 6: ตรวจสอบ**

```bash
# หลัง install เสร็จ login ด้วย:
# Username: admin
# Password: (blank - กด Enter)

# ตรวจสอบ version
/system resource print
```

### 2.4 Troubleshooting Netinstall

```
ปัญหา: Router ไม่ปรากฏใน Netinstall

สาเหตุและวิธีแก้:
1. Firewall บน PC → ปิด Windows Firewall
2. IP address ไม่ถูก → ตรวจสอบ 192.168.88.2/24
3. Cable ผิด port → ต้องต่อที่ Port 1 (BootP port)
4. Reset ไม่สำเร็จ → ลอง reset ใหม่ ถือ button นานขึ้น
5. Network adapter ผิด → ตรวจสอบใน Netinstall dropdown
```

---

## 3. Installation บน VMware/VirtualBox

### 3.1 VMware Workstation/Player

**ดาวน์โหลด OVA:**
```
mikrotik.com/download → Cloud Hosted Router → OVA format
```

**Import OVA:**
```
VMware → File → Open
เลือก chr-7.15.3.ova
Import (อาจมี warning เรื่อง spec, กด Retry)
```

**หรือสร้าง VM ใหม่:**
```
1. New Virtual Machine → Custom
2. Guest OS: Linux → Other Linux 5.x kernel 64-bit
3. VM Name: MikroTik-CHR
4. Processors: 1-2 vCPU
5. Memory: 256MB (minimum) / 512MB (recommended)
6. Network:
   - Adapter 1: Bridged (หรือ NAT สำหรับ WAN)
   - Adapter 2: Custom VMnet2 (สำหรับ LAN)
7. Disk: Use existing disk → chr-7.15.3.vmdk
8. Finish
```

### 3.2 VirtualBox

**Step 1: ดาวน์โหลด VDI**
```
mikrotik.com/download → Cloud Hosted Router → VDI format
```

**Step 2: สร้าง VM**
```
New → ตั้งชื่อ: MikroTik-CHR
Type: Linux
Version: Other Linux (64-bit)
Memory: 256MB
Hard disk: Use existing → chr-7.15.3.vdi
```

**Step 3: Network Adapters**
```
Settings → Network

Adapter 1:
├── Enable Network Adapter: ✓
├── Attached to: Bridged Adapter
└── Name: [your physical adapter]

Adapter 2:
├── Enable Network Adapter: ✓
├── Attached to: Internal Network
└── Name: intnet1

Adapter 3: (optional)
├── Enable Network Adapter: ✓
├── Attached to: Internal Network
└── Name: intnet2
```

**Step 4: Start VM**
```
กด Start
Login:
Username: admin
Password: (กด Enter)
```

### 3.3 Proxmox VE

```bash
# บน Proxmox server

# ดาวน์โหลด CHR image
wget https://download.mikrotik.com/routeros/7.15.3/chr-7.15.3.img.zip
unzip chr-7.15.3.img.zip

# Convert เป็น qcow2
qemu-img convert -f raw -O qcow2 chr-7.15.3.img chr-7.15.3.qcow2

# สร้าง VM (ผ่าน CLI)
qm create 100 --name "MikroTik-CHR" \
  --memory 512 \
  --cores 2 \
  --net0 virtio,bridge=vmbr0 \
  --net1 virtio,bridge=vmbr1 \
  --ostype l26

# Import disk
qm importdisk 100 chr-7.15.3.qcow2 local-lvm

# ตั้งค่า disk
qm set 100 --virtio0 local-lvm:vm-100-disk-0
qm set 100 --boot c --bootdisk virtio0

# Start VM
qm start 100
```

---

## 4. CHR Installation

### 4.1 CHR บน DigitalOcean

```bash
# ผ่าน DigitalOcean API
# Step 1: สร้าง Space (S3-compatible)
# Step 2: Upload CHR image

# หรือใช้ custom image feature
# Settings → Images → Custom Images
# Upload URL: https://download.mikrotik.com/routeros/7.15.3/chr-7.15.3.img.zip

# สร้าง Droplet จาก custom image
doctl compute droplet create mikrotik-chr \
  --size s-1vcpu-1gb \
  --image [IMAGE_ID] \
  --region nyc3 \
  --ssh-keys [KEY_ID]
```

### 4.2 CHR บน AWS EC2

```bash
# ดาวน์โหลดและ upload ไปยัง S3
aws s3 cp chr-7.15.3.img s3://mybucket/mikrotik/

# สร้าง import task
aws ec2 import-image \
  --description "MikroTik CHR 7.15.3" \
  --disk-containers '[
    {
      "Description": "MikroTik CHR",
      "Format": "RAW",
      "UserBucket": {
        "S3Bucket": "mybucket",
        "S3Key": "mikrotik/chr-7.15.3.img"
      }
    }
  ]'

# ตรวจสอบ status
aws ec2 describe-import-image-tasks --import-task-ids import-ami-XXXX

# Launch instance
aws ec2 run-instances \
  --image-id ami-XXXXXXXXXX \
  --instance-type t3.micro \
  --key-name mykey \
  --subnet-id subnet-XXXX \
  --security-group-ids sg-XXXX \
  --tag-specifications 'ResourceType=instance,Tags=[{Key=Name,Value=MikroTik-CHR}]'
```

---

## 5. First Boot Configuration

### 5.1 Initial Login

```
Default credentials:
Username: admin
Password: (blank)

หมายเหตุ:
- RouterOS 7.3+ จะ prompt ให้ตั้ง password ใหม่
- ถ้าไม่ตั้ง password จะมี warning ใน log
```

### 5.2 Setup Wizard (Quick Set)

เมื่อ login ครั้งแรก ผ่าน Winbox จะมี Setup Wizard:

```
Quick Set หน้าแรก:
├── Mode: Router
├── Internet (WAN):
│   ├── Address Acquisition: DHCP
│   ├── หรือ PPPoE (สำหรับ ADSL/Fiber)
│   └── หรือ Static
├── Local Network (LAN):
│   ├── IP: 192.168.88.1/24 (default)
│   └── DHCP Server: enabled
└── Wireless:
    ├── SSID: mikrotik
    └── Security: WPA2
```

### 5.3 Basic CLI Setup

```bash
# Step 1: ตั้ง hostname
/system identity set name=MyRouter

# Step 2: ตั้ง timezone
/system clock set time-zone-name=Asia/Bangkok

# Step 3: ตั้ง admin password
/user set admin password=StrongP@ssw0rd!

# Step 4: ตั้ง IP address บน interface
/ip address add address=192.168.1.1/24 interface=ether1

# Step 5: ตั้ง default gateway (ถ้ามี WAN)
/ip route add gateway=203.0.113.1

# Step 6: ตั้ง DNS
/ip dns set servers=8.8.8.8,8.8.4.4

# Step 7: ทดสอบ
/ping 8.8.8.8 count=4
```

### 5.4 Initial Security Hardening

```bash
# Step 1: เปลี่ยน admin password (สำคัญมาก!)
/user set admin password=STRONG_PASSWORD_HERE

# Step 2: ปิด services ที่ไม่ใช้
/ip service disable telnet
/ip service disable ftp
/ip service disable api

# Step 3: เปลี่ยน port services
/ip service set ssh port=2222
/ip service set www port=8080
/ip service set api-ssl port=8729

# Step 4: จำกัด access ไปยัง management services
/ip service set ssh address=192.168.1.0/24
/ip service set www address=192.168.1.0/24
/ip service set winbox address=192.168.1.0/24

# Step 5: เปิด SSH key-based authentication (แนะนำ)
# ก่อน: upload public key
/user ssh-keys import public-key-file=id_rsa.pub user=admin

# Step 6: ปิด Neighbor Discovery บน WAN
/ip neighbor discovery-settings set discover-interface-list=!WAN

# Step 7: Disable Bandwidth Test server
/tool bandwidth-server set enabled=no
```

---

## 6. Default Credentials และ Security

### 6.1 Default Configuration

```
Default Settings หลัง Fresh Install:
├── Username: admin
├── Password: (none/blank)
├── IP Address: 192.168.88.1/24 บน ether1 (ถ้ามี bridge)
├── DHCP Server: enabled (ถ้ามี bridge config)
│   └── Pool: 192.168.88.10-192.168.88.254
├── SSH: enabled (port 22)
├── Telnet: enabled (port 23) - ควรปิด!
├── HTTP: enabled (port 80)
├── Winbox: enabled (port 8291)
└── Firewall: basic rules อาจมีอยู่แล้ว
```

### 6.2 Default Firewall Rules (RouterOS v7)

```bash
# ดู default rules
/ip firewall filter print

# ปกติจะมี rules เหล่านี้:
# chain=input action=accept connection-state=established,related,untracked
# chain=input action=drop in-interface-list=WAN
# chain=forward action=accept ipsec-policy=in,ipsec
# chain=forward action=accept ipsec-policy=out,ipsec
# chain=forward action=fasttrack connection-state=established,related
# chain=forward action=accept connection-state=established,related,untracked
# chain=forward action=drop connection-state=invalid
# chain=forward action=drop in-interface-list=WAN out-interface-list=!WAN
```

### 6.3 Security Checklist

```
✅ Security Checklist:
□ เปลี่ยน admin password
□ สร้าง user account ใหม่ที่ไม่ใช่ admin (optional)
□ ปิด Telnet
□ ปิด FTP (ถ้าไม่ใช้)
□ จำกัด management access จาก trusted IPs เท่านั้น
□ ปิด API ถ้าไม่ใช้
□ Enable SSH key authentication
□ Disable WinBox MAC access (ถ้าไม่ต้องการ)
□ ตั้ง Firewall input rules
□ ปิด unused packages
□ Update RouterOS ให้เป็น version ล่าสุด
□ ปิด Bandwidth Test server
□ ปิด DNS recursive queries (ถ้าไม่ต้องการ)
```

---

## 7. Network Interface Setup เบื้องต้น

### 7.1 ดู Interface ทั้งหมด

```bash
# ดู interface list
/interface print

# Output:
# Flags: D - dynamic, X - disabled, R - running, S - slave
#  #     NAME      TYPE     ACTUAL-MTU L2MTU  MAX-MTU MAC-ADDRESS       LAST-LINK-UP-TIME
#  0  R  ether1    ether    1500       1598   65535   XX:XX:XX:XX:XX:01 Sep/25/2024 10:00:00
#  1  R  ether2    ether    1500       1598   65535   XX:XX:XX:XX:XX:02 Sep/25/2024 10:00:00
#  2  R  ether3    ether    1500       1598   65535   XX:XX:XX:XX:XX:03 Sep/25/2024 10:00:00
#  3  R  ether4    ether    1500       1598   65535   XX:XX:XX:XX:XX:04 Sep/25/2024 10:00:00
#  4  R  ether5    ether    1500       1598   65535   XX:XX:XX:XX:XX:05 Sep/25/2024 10:00:00
#  5     sfp-sfpplus1 ether  1500      1598   65535   XX:XX:XX:XX:XX:06

# ดูรายละเอียด
/interface ethernet print detail
```

### 7.2 ตั้งชื่อ Interface

```bash
# เปลี่ยนชื่อ interface ให้จดจำง่าย
/interface set ether1 name=WAN
/interface set ether2 name=LAN
/interface set ether3 name=DMZ
/interface set sfp-sfpplus1 name=UPLINK

# ตรวจสอบ
/interface print
```

### 7.3 ตั้ง IP Address

```bash
# เพิ่ม IP address
/ip address add address=192.168.1.1/24 interface=LAN comment="LAN interface"
/ip address add address=10.0.0.2/30 interface=WAN comment="WAN interface"

# ดู IP addresses
/ip address print

# Output:
# Flags: X - disabled, I - invalid, D - dynamic
#  #   ADDRESS            NETWORK         INTERFACE
#  0   192.168.1.1/24     192.168.1.0     LAN
#  1   10.0.0.2/30        10.0.0.0        WAN

# ลบ IP address
/ip address remove [find address="192.168.1.1/24"]

# ปิด/เปิด IP address
/ip address disable 0
/ip address enable 0
```

### 7.4 Interface Lists

```bash
# สร้าง interface list สำหรับ firewall
/interface list add name=WAN comment="WAN interfaces"
/interface list add name=LAN comment="LAN interfaces"

# เพิ่ม interface เข้า list
/interface list member add interface=ether1 list=WAN
/interface list member add interface=ether2 list=LAN
/interface list member add interface=ether3 list=LAN

# ดู lists
/interface list print
/interface list member print
```

### 7.5 Default Route (Gateway)

```bash
# เพิ่ม default route
/ip route add dst-address=0.0.0.0/0 gateway=10.0.0.1 comment="Default Route"

# ดู routing table
/ip route print

# Output:
# Flags: X - disabled, A - active, D - dynamic
#  #     DST-ADDRESS       GATEWAY       DISTANCE
#  0 AD  0.0.0.0/0         10.0.0.1      1
#  1 A   10.0.0.0/30       WAN           0
#  2 A   192.168.1.0/24    LAN           0
```

---

## 8. License Activation

### 8.1 RouterBOARD License

RouterBOARD มา pre-licensed ไม่ต้องทำอะไร:

```bash
# ตรวจสอบ license
/system license print

# Output:
#   software-id: XXXX-XXXX
#   level: 6
#   features: hotspot user-manager
```

### 8.2 ซื้อและ Activate License ใหม่

```bash
# บน Router ที่ต้องการ license
/system license print
# บันทึก software-id

# ไปซื้อ license บน https://mikrotik.com/client
# ใส่ software-id และ ชำระเงิน
# จะได้ license key

# Activate online (Router ต้องมี internet)
/system license apply-key key=XXXX-XXXX-XXXX-XXXX

# หรือ activate ผ่าน command line
/system license set level=6

# ตรวจสอบ
/system license print
```

### 8.3 License Renewal (CHR)

```bash
# CHR ต่ออายุ subscription
/system license renew level=p1

# ตรวจสอบ
/system license print
# Output:
#   software-id: XXXX-XXXX
#   level: p1
#   deadline: jan/01/2027
```

---

## 9. RouterOS 7.x vs 6.x Differences

### 9.1 Major Changes

| Feature | RouterOS 6.x | RouterOS 7.x |
|---------|-------------|-------------|
| OSPF | /routing ospf | /routing ospf (redesigned) |
| BGP | /routing bgp | /routing bgp (redesigned) |
| Firewall NAT | /ip firewall nat | /ip firewall nat (เหมือนเดิม) |
| IPv6 Firewall | /ipv6 firewall | /ipv6 firewall (เหมือนเดิม) |
| VLAN | /interface vlan | /interface vlan (เหมือนเดิม) |
| WireGuard | ไม่รองรับ | /interface wireguard |
| Container | ไม่รองรับ | /container (Docker-like) |
| ZeroTier | ไม่รองรับ | /zerotier |
| CAPsMAN | v2 | v3 (redesigned) |
| Dot1X | จำกัด | ปรับปรุง |

### 9.2 Routing Protocol Changes (สำคัญมาก)

**RouterOS 6 OSPF:**
```bash
# RouterOS 6.x (เก่า)
/routing ospf instance set default router-id=1.1.1.1
/routing ospf network add network=192.168.1.0/24 area=backbone

# ดู neighbors
/routing ospf neighbor print
```

**RouterOS 7 OSPF:**
```bash
# RouterOS 7.x (ใหม่ - ต่างกันมาก!)
/routing ospf instance add name=ospf1 router-id=1.1.1.1

/routing ospf area add name=backbone instance=ospf1 area-id=0.0.0.0

/routing ospf interface-template add interfaces=LAN area=backbone

# ดู neighbors
/routing ospf neighbor print
```

### 9.3 BGP Changes

**RouterOS 6 BGP:**
```bash
# RouterOS 6.x
/routing bgp instance set default as=65001 router-id=1.1.1.1
/routing bgp peer add name=peer1 remote-address=10.0.0.2 remote-as=65002
```

**RouterOS 7 BGP:**
```bash
# RouterOS 7.x
/routing bgp template add name=ibgp as=65001

/routing bgp connection add name=peer1 \
  remote.address=10.0.0.2/32 \
  remote.as=65002 \
  local.role=ebgp \
  templates=ibgp
```

### 9.4 Migration Tips

```
Tips เมื่อ upgrade 6 → 7:

1. ทำ export config บน v6:
   /export file=v6-backup

2. ทำ backup:
   /system backup save name=before-v7-upgrade

3. Upgrade RouterOS
4. ตรวจสอบ routing protocols ทันที
   - OSPF อาจต้อง reconfigure ใหม่
   - BGP อาจต้อง reconfigure ใหม่

5. Check firewall rules
6. Test connectivity ทุก service

⚠️ Warning: ไม่แนะนำ downgrade หลังจาก config database ถูก migrate แล้ว
```

---

## 10. Upgrade/Downgrade Procedures

### 10.1 Pre-Upgrade Checklist

```bash
# Step 1: Backup configuration
/system backup save name=pre-upgrade-$(date +%Y%m%d)
/export file=pre-upgrade-config-$(date +%Y%m%d)

# Step 2: ตรวจสอบ current version
/system resource print

# Step 3: Note สิ่งที่อาจเปลี่ยนแปลง
# - อ่าน changelog ก่อน upgrade
# https://mikrotik.com/download/changelogs/

# Step 4: Plan maintenance window
# - ควร upgrade ในช่วงที่ traffic น้อย
# - แจ้ง users ล่วงหน้า
```

### 10.2 Upgrade via Internet

```bash
# ตรวจสอบ update
/system package update check-for-updates

# ดาวน์โหลดและติดตั้ง
/system package update install

# Router จะ reboot อัตโนมัติ
# หลัง reboot ตรวจสอบ version
/system resource print
```

### 10.3 Upgrade via File Upload

```bash
# Method 1: SCP
scp routeros-7.15.3-arm64.npk admin@192.168.1.1:/

# Method 2: FTP
ftp admin@192.168.1.1
put routeros-7.15.3-arm64.npk

# Method 3: Winbox
# Files → Drag and drop .npk file

# หลัง upload, reboot
/system reboot
# RouterOS จะ install package อัตโนมัติระหว่าง boot
```

### 10.4 Downgrade

```bash
# ดาวน์โหลด version เก่าจาก:
# https://mikrotik.com/download/changelogs/

# Upload .npk ของ version เก่า
scp routeros-7.14.3-arm64.npk admin@192.168.1.1:/

# Downgrade
/system package downgrade

# Router จะ reboot และใช้ version เก่า
```

### 10.5 Upgrade Extra Packages

```bash
# ถ้าต้องการ packages เพิ่มเติม
# ดาวน์โหลดจาก https://mikrotik.com/download
# เลือก "Extra packages" สำหรับ architecture ของ router

# Upload extra package
scp user-manager-7.15.3-arm64.npk admin@192.168.1.1:/

# Reboot เพื่อ install
/system reboot

# ตรวจสอบ packages
/system package print
```

---

## 11. Summary และ Lab Exercise

### Summary

สิ่งที่เรียนรู้ใน Part 3:

1. **Download** RouterOS จาก mikrotik.com/download โดยเลือก architecture ให้ถูกต้อง
2. **Netinstall** ใช้สำหรับ fresh install หรือ recovery ต้องใช้ PC Windows และต่อสาย
3. **Virtual Installation** ใช้ CHR image สำหรับ VMware, VirtualBox, Proxmox
4. **First Boot** ต้องตั้ง password, timezone, IP address ทันที
5. **Security** ปิด services ที่ไม่ใช้, จำกัด management access
6. **RouterOS 7 vs 6** routing protocols เปลี่ยน syntax สำคัญมาก
7. **Upgrade** backup ก่อนทุกครั้ง, อ่าน changelog

### Lab Exercise: Basic Installation

**Lab Goal:** ติดตั้ง RouterOS บน VirtualBox และทำ basic configuration

**Requirements:**
- VirtualBox ติดตั้งแล้ว
- ดาวน์โหลด CHR VDI จาก mikrotik.com

**Lab Steps:**

```
Lab Exercise 3-1: VirtualBox CHR Installation

Step 1: สร้าง VM
- Name: Lab-Router1
- Type: Linux, 64-bit
- RAM: 256MB
- Disk: chr-7.15.3.vdi

Step 2: Network Setup
- Adapter 1: NAT (WAN)
- Adapter 2: Internal Network "lab-net"

Step 3: First Boot
- Login: admin (no password)
- ตั้ง password ใหม่
- ตั้งชื่อ router

Step 4: Basic Config
- ตั้ง hostname: Lab-R1
- ตั้ง timezone: Asia/Bangkok
- Configure ether1 as DHCP client (WAN)
- Configure ether2 as 192.168.100.1/24 (LAN)
- ตั้ง NAT masquerade

Step 5: ทดสอบ
- ping 8.8.8.8
- ping google.com
```

**Expected Commands:**

```bash
# ตั้งชื่อ
/system identity set name=Lab-R1

# ตั้ง timezone
/system clock set time-zone-name=Asia/Bangkok

# ตั้ง password
/user set admin password=Lab123!@#

# WAN interface (DHCP)
/ip dhcp-client add interface=ether1 disabled=no

# LAN interface
/ip address add address=192.168.100.1/24 interface=ether2

# NAT
/ip firewall nat add chain=srcnat out-interface=ether1 action=masquerade

# DNS
/ip dns set servers=8.8.8.8,8.8.4.4 allow-remote-requests=no

# Test
/ping 8.8.8.8 count=4
/ping google.com count=4
```

**Verification:**

```bash
# ตรวจสอบ config
/system identity print
/ip address print
/ip route print
/ip dns print
/ip dhcp-client print
```

---

**[⬅ Previous: Hardware and Products](part-002-hardware-and-products.md)** | **[Next: Winbox, WebFig, CLI ➡](part-004-winbox-webfig-cli.md)**
