# Part 14: User Management

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate
**เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

- [User Accounts](#user-accounts)
- [User Groups](#user-groups)
- [Permissions](#permissions)
- [SSH Key Authentication](#ssh-key-authentication)
- [Two-Factor Authentication](#two-factor-authentication)
- [AAA](#aaa)
- [RADIUS Client](#radius-client)
- [Password Policy](#password-policy)
- [Session Management](#session-management)
- [Audit Logging](#audit-logging)
- [Lab: Secure User Management](#lab)

---

## User Accounts

### การจัดการ User ใน RouterOS

RouterOS ใช้ระบบ Local User Management ที่จัดการผ่าน `/user` Path

### ดู Users ปัจจุบัน

```routeros
# ดู Users ทั้งหมด
/user print

# ดูแบบ Detail
/user print detail

# Output ตัวอย่าง:
# Flags: X - disabled
#  #  NAME      GROUP     ADDRESS  LAST-LOGGED-IN       COMMENT
#  0  admin     full      0.0.0.0  2026-01-01 08:00:00
#  1  operator  read      0.0.0.0  never
```

### สร้าง User ใหม่

```routeros
# สร้าง User พื้นฐาน
/user add name=john password="P@ssw0rd123!" group=full comment="John Admin"

# สร้าง User แบบ Read-Only
/user add name=monitor password="ReadOnly456!" group=read comment="Monitoring User"

# สร้าง User พร้อมกำหนด Allowed IP
/user add \
    name=sysadmin \
    password="Admin@2026!" \
    group=full \
    address=192.168.1.0/24 \
    comment="System Admin - LAN only"
```

### แก้ไข User

```routeros
# เปลี่ยน Password
/user set john password="NewP@ssw0rd789!"

# เปลี่ยน Group
/user set john group=write

# กำหนด Allowed IP Address
/user set john address=10.0.0.0/8

# Disable User ชั่วคราว
/user disable john

# Enable User
/user enable john

# ลบ User
/user remove john
```

### ลบ Default Admin Password

```routeros
# สิ่งแรกที่ควรทำ: เปลี่ยน/ลบ Default Admin
# และสร้าง User ใหม่แทน

# สร้าง Admin ใหม่
/user add name=myadmin password="SuperSecure@2026!" group=full

# เปลี่ยน Password ของ admin เดิม
/user set admin password="ChangedDefault@123!"

# หรือ Disable admin เดิม (ห้ามลบ admin ถ้ายังไม่มี user อื่น group=full)
/user disable admin
```

> **Warning:** ห้ามลบ User `admin` โดยไม่มี User group=full อื่น เพราะจะล็อคตัวเองออกจากระบบ

---

## User Groups

### Default Groups

RouterOS มี Groups พื้นฐาน 3 กลุ่ม:

| Group | สิทธิ์ | คำอธิบาย |
|-------|--------|---------|
| `full` | ทั้งหมด | ควบคุมได้ทุกอย่าง |
| `write` | Read + Write | ดูและแก้ไขได้ แต่ไม่ delete sensitive |
| `read` | Read Only | ดูได้อย่างเดียว |

### ดู Groups

```routeros
# ดู Groups ทั้งหมด
/user group print

# ดูแบบ Detail (ดู Policies)
/user group print detail

# Output ตัวอย่าง:
# Flags: * - builtin
#  #  NAME   POLICY  SKIN
#  0 *full  reboot,ssh,telnet,ftp,winbox,web,sniff,sensitive,...
#  1 *write password,policy,reboot,...
#  2 *read
```

### สร้าง Custom Group

```routeros
# สร้าง Group สำหรับ NOC (Network Operations Center)
/user group add \
    name=noc \
    policy=ssh,telnet,winbox,web,read,test,sniff \
    comment="NOC Team - Read + Test"

# สร้าง Group สำหรับ Helpdesk
/user group add \
    name=helpdesk \
    policy=ssh,winbox,web,read \
    comment="Helpdesk - Read Only with Web Access"

# สร้าง Group สำหรับ Network Engineer
/user group add \
    name=neteng \
    policy=ssh,telnet,ftp,winbox,web,read,write,test,sniff,reboot \
    comment="Network Engineer - Almost Full"
```

### Policies ทั้งหมด

| Policy | คำอธิบาย |
|--------|---------|
| `local` | Login ผ่าน Console |
| `telnet` | Login ผ่าน Telnet |
| `ssh` | Login ผ่าน SSH |
| `ftp` | Login ผ่าน FTP |
| `reboot` | สิทธิ์ Reboot Router |
| `read` | อ่าน Configuration |
| `write` | เขียน Configuration |
| `policy` | จัดการ User Policies |
| `test` | ทดสอบ (Ping, Traceroute) |
| `winbox` | Login ผ่าน WinBox |
| `password` | เปลี่ยน Password ตัวเอง |
| `web` | Login ผ่าน WebFig |
| `sniff` | Packet Sniffer |
| `sensitive` | ดู Sensitive Data (Keys, Passwords) |
| `api` | Login ผ่าน API |
| `romon` | ใช้ RoMON |
| `dude` | Dude Monitoring |
| `tikapp` | TikApp |

---

## Permissions

### IP-Based Access Control

```routeros
# จำกัด User ให้ Login ได้จาก IP เฉพาะ
/user set admin address=192.168.1.100/32
/user set admin address=10.0.0.0/8

# อนุญาตหลาย Subnet
/user set admin address=192.168.1.0/24,10.0.0.0/8

# ลบ IP Restriction (อนุญาตทุก IP)
/user set admin address=""
```

### Service Restrictions

```routeros
# จำกัด Service ที่อนุญาตให้ Login
# (ทำในระดับ /ip service)

# ปิด Telnet ทั้งหมด (ไม่ปลอดภัย)
/ip service disable telnet

# จำกัด SSH เฉพาะ IP
/ip service set ssh address=192.168.1.0/24

# จำกัด WinBox เฉพาะ IP
/ip service set winbox address=192.168.1.0/24

# ดู Services
/ip service print
```

### Allowed Services per User

```routeros
# User นี้เข้าได้ผ่าน SSH เท่านั้น (ผ่าน Group Policy)
/user group add name=ssh-only policy=ssh,read comment="SSH-only access"
/user add name=ssh-user group=ssh-only password="SSHOnly@123!"
```

---

## SSH Key Authentication

### ทำไมต้องใช้ SSH Key?

- ปลอดภัยกว่า Password มาก
- ไม่ต้องพิมพ์ Password ทุกครั้ง
- ป้องกัน Brute Force Attack
- รองรับ Automation

### สร้าง SSH Key Pair

```bash
# สร้าง Key Pair บน Linux/Mac
ssh-keygen -t rsa -b 4096 -C "admin@mycompany.com" -f ~/.ssh/mikrotik_rsa

# หรือ Ed25519 (แนะนำ)
ssh-keygen -t ed25519 -C "admin@mycompany.com" -f ~/.ssh/mikrotik_ed25519

# ดู Public Key
cat ~/.ssh/mikrotik_ed25519.pub
```

### นำ Public Key ใส่ RouterOS

```routeros
# วิธีที่ 1: ผ่าน WinBox - Files
# Upload ไฟล์ .pub ไปยัง Files แล้วใช้คำสั่ง:
/user ssh-keys import user=admin public-key-file=mikrotik_ed25519.pub

# วิธีที่ 2: ผ่าน Command Line (Copy-paste Public Key)
/user ssh-keys add user=admin key="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... admin@mycompany.com"

# ดู SSH Keys
/user ssh-keys print
```

### ทดสอบ SSH Key Login

```bash
# Login ด้วย SSH Key
ssh -i ~/.ssh/mikrotik_ed25519 admin@192.168.1.1

# หากไม่ใช้ Default Port
ssh -i ~/.ssh/mikrotik_ed25519 -p 2222 admin@192.168.1.1
```

### ปิดการใช้ Password Authentication (หลังตั้งค่า Key แล้ว)

```routeros
# หลังจาก Test SSH Key ว่าใช้งานได้แล้ว
# สามารถปิด Password Authentication ได้
# (ทำใน /ip ssh settings)
/ip ssh set strong-crypto=yes

# หมายเหตุ: RouterOS ไม่มีตัวเลือก disable password auth โดยตรง
# แต่สามารถทำผ่าน Script ที่ตรวจสอบ Key ก่อน Login
```

---

## Two-Factor Authentication

### 2FA ใน RouterOS

RouterOS v7 รองรับ TOTP (Time-based One-Time Password) สำหรับ 2FA

### เปิดใช้ 2FA

```routeros
# RouterOS v7.x
# เปิดใช้ 2FA สำหรับ User
/user set admin two-factor-auth=google-authenticator

# Generate QR Code สำหรับ Authenticator App
/user generate-secret admin
```

### Script ตรวจสอบ Login

```routeros
# Login Notification Script
/system scheduler add \
    name=login-check \
    interval=1m \
    on-event={
        :local users [/user active print count-only]
        :if ($users > 0) do={
            :log warning "Active sessions: $users"
        }
    }
```

---

## AAA

### AAA คืออะไร?

**AAA = Authentication, Authorization, Accounting**

- **Authentication**: ใครคือคุณ? (ยืนยันตัวตน)
- **Authorization**: คุณทำอะไรได้บ้าง? (สิทธิ์)
- **Accounting**: คุณทำอะไรไปบ้าง? (บันทึก)

### Local AAA

```routeros
# RouterOS ใช้ Local AAA โดย Default
# Authentication: Username + Password ใน /user
# Authorization: Group + Policy ใน /user group
# Accounting: /log และ /user active

# ดู Active Sessions (Accounting)
/user active print

# Output:
# WHEN                    NAME   ADDRESS          VIA     COMMENT
# 2026-01-01 08:00:00  admin  192.168.1.100  winbox
# 2026-01-01 08:05:00  john   10.0.0.5       ssh
```

### RADIUS AAA

```routeros
# ใช้ RADIUS สำหรับ Centralized AAA
/radius add \
    address=192.168.1.10 \
    secret=shared-secret-123 \
    service=login \
    comment="Main RADIUS Server"

# กำหนดให้ใช้ RADIUS ก่อน Local
/user aaa set use-radius=yes
```

---

## RADIUS Client

### ตั้งค่า RADIUS

```routeros
# เพิ่ม RADIUS Server
/radius add \
    address=192.168.1.10 \
    secret=MySharedSecret \
    authentication-port=1812 \
    accounting-port=1813 \
    service=login \
    timeout=3s \
    comment="Primary RADIUS"

# เพิ่ม Backup RADIUS
/radius add \
    address=192.168.1.11 \
    secret=MySharedSecret \
    service=login \
    comment="Backup RADIUS"

# ดู RADIUS Servers
/radius print
```

### เปิดใช้ RADIUS Authentication

```routeros
# เปิดใช้ RADIUS สำหรับ Login
/user aaa set use-radius=yes accounting=yes

# กำหนด Fallback เป็น Local เมื่อ RADIUS ไม่ตอบ
# (Default: ใช้ Local เมื่อ RADIUS ไม่ตอบ)

# ดูการตั้งค่า
/user aaa print
```

### RADIUS Attributes สำหรับ RouterOS

```
# ใน RADIUS Server, กำหนด Attributes สำหรับ RouterOS:
Mikrotik-Group = "full"           # กำหนด Group
Mikrotik-User-Address = "192.168.1.0/24"  # จำกัด IP
Service-Type = Administrative-User # ประเภท User
```

### ทดสอบ RADIUS

```routeros
# ทดสอบ RADIUS Authentication
/radius test address=192.168.1.10 secret=MySharedSecret user=testuser password=testpass
```

---

## Password Policy

### Password Best Practices

```routeros
# ตั้งค่า Password ที่แข็งแรง
# RouterOS ไม่มี Built-in Password Policy แต่สามารถ Enforce ได้ผ่าน Script

# Script ตรวจสอบ Password Strength เมื่อสร้าง User
:local pw "MyPassword"
:local minLength 12
:local hasUpper false
:local hasLower false
:local hasDigit false
:local hasSpecial false

:if ([:len $pw] < $minLength) do={
    :error "Password too short (min $minLength chars)"
}
```

### Password Change Policy

```routeros
# Script บังคับเปลี่ยน Password ทุก 90 วัน
# (ต้องรัน Scheduler ทุกวัน)

/system scheduler add \
    name=password-expiry-check \
    interval=1d \
    on-event={
        :local users [/user find]
        :foreach u in=$users do={
            :local lastChange [/user get $u last-logged-in]
            # ตรวจสอบและ Notify
        }
    }
```

### Password Recommendations

| เกณฑ์ | ข้อแนะนำ |
|-------|---------|
| ความยาว | อย่างน้อย 12 ตัวอักษร |
| ตัวพิมพ์ | ผสม Upper + Lower case |
| ตัวเลข | มีตัวเลขอย่างน้อย 2 ตัว |
| อักขระพิเศษ | มี !@#$%^&* อย่างน้อย 1 ตัว |
| ไม่ใช้ | ชื่อ, วันเกิด, ข้อมูลส่วนตัว |
| เปลี่ยนทุก | 90 วัน |

---

## Session Management

### ดู Active Sessions

```routeros
# ดู Users ที่ Login อยู่
/user active print

# ดูแบบ Detail
/user active print detail

# Output:
# WHEN                 NAME     ADDRESS         VIA      UPTIME
# 2026-01-01 08:00:00  admin    192.168.1.100   winbox   2h30m
# 2026-01-01 09:00:00  john     10.0.0.5        ssh      1h15m
```

### Kick User ออก

```routeros
# Kick User ที่ระบุออกจากระบบ
/user active remove [find name=john]

# Kick ทุก Session ของ User นั้น
/user active remove [find name=john]
```

### Session Timeout

```routeros
# กำหนด Idle Timeout สำหรับแต่ละ Service
/ip service set telnet timeout=5m
/ip service set ssh timeout=30m
/ip service set winbox timeout=1h
/ip service set web timeout=30m
```

### Login Banner

```routeros
# กำหนด Login Banner (แสดงก่อน Login)
/ip telnet set default-ssh-banner="AUTHORIZED ACCESS ONLY\nAll activities are logged"

# ผ่าน SSH Banner
/ip ssh set banner="WARNING: Authorized access only!\nAll connections are monitored and logged."
```

---

## Audit Logging

### เปิดใช้ Audit Log

```routeros
# ดู Log ทั้งหมด
/log print

# Filter เฉพาะ User-related Events
/log print where topics~"account"

# Filter เฉพาะ Login/Logout
/log print where message~"logged in" or message~"logged out"
```

### ตั้งค่า Logging

```routeros
# ส่ง Log ไปยัง Syslog Server
/system logging action add \
    name=syslog \
    type=remote \
    remote=192.168.1.100 \
    remote-port=514 \
    src-address=0.0.0.0

# กำหนด Topics ที่ต้องการ Log
/system logging add \
    action=syslog \
    topics=account \
    comment="User account events"

/system logging add \
    action=syslog \
    topics=system \
    comment="System events"
```

### Log Topics

| Topic | คำอธิบาย |
|-------|---------|
| `account` | Login/Logout Events |
| `system` | System Events |
| `firewall` | Firewall Events |
| `wireless` | Wireless Events |
| `script` | Script Execution |
| `warning` | Warnings |
| `error` | Errors |
| `critical` | Critical Events |
| `info` | Informational |

### Syslog Format

```
Log ตัวอย่าง:
2026-01-01 08:00:00 account,info admin logged in from 192.168.1.100 via winbox
2026-01-01 08:30:00 account,info admin logged out from 192.168.1.100 via winbox
2026-01-01 09:00:00 account,warning john login failure for user john from 10.0.0.5 via ssh
```

### Email Alerts

```routeros
# ตั้งค่า Email
/tool e-mail set \
    server=smtp.gmail.com \
    port=587 \
    from=router@example.com \
    user=router@example.com \
    password=AppPassword123

# Script ส่ง Email เมื่อมีการ Login
/system scheduler add \
    name=login-alert \
    interval=1m \
    on-event={
        :local newLogins [/user active find]
        :if ([:len $newLogins] > 0) do={
            /tool e-mail send \
                to="admin@example.com" \
                subject="[Router] New Login Alert" \
                body="New login detected on router"
        }
    }
```

---

## Lab: Secure User Management

### Objectives

1. ลบ/ปิด Default Credentials
2. สร้าง Users สำหรับแต่ละ Role
3. กำหนด Custom Groups พร้อม Policies
4. ตั้งค่า SSH Key Authentication
5. จำกัด Service Access
6. เปิดใช้ Audit Logging

### Step 1: ปิด Insecure Services

```routeros
# ปิด Telnet (ไม่มี Encryption)
/ip service disable telnet

# ปิด HTTP (ใช้ HTTPS แทน)
/ip service disable www

# เปลี่ยน SSH Port (ลด Bot Scan)
/ip service set ssh port=2222

# จำกัด Services เฉพาะ Management Network
/ip service set ssh address=192.168.10.0/24
/ip service set winbox address=192.168.10.0/24
/ip service set www-ssl address=192.168.10.0/24
/ip service set api address=192.168.10.0/24

# ดูผลลัพธ์
/ip service print
```

### Step 2: สร้าง User Groups

```routeros
# Group 1: Super Admin
/user group add \
    name=super-admin \
    policy=local,telnet,ssh,ftp,reboot,read,write,policy,test,winbox,password,web,sniff,sensitive,api,romon \
    comment="Full Administrator"

# Group 2: Network Operations Center
/user group add \
    name=noc \
    policy=ssh,read,test,winbox,web \
    comment="NOC - Read + Test Only"

# Group 3: Helpdesk
/user group add \
    name=helpdesk \
    policy=winbox,web,read \
    comment="Helpdesk - Read Only"

# Group 4: Monitoring System
/user group add \
    name=monitoring \
    policy=api,read \
    comment="Monitoring API Access"

# ดูผลลัพธ์
/user group print
```

### Step 3: สร้าง Users

```routeros
# Super Admin
/user add \
    name=sysadmin \
    password="$ysAdm1n@2026!" \
    group=super-admin \
    address=192.168.10.0/24 \
    comment="System Administrator"

# NOC Staff
/user add \
    name=noc1 \
    password="N0cUser@2026!" \
    group=noc \
    address=192.168.10.0/24 \
    comment="NOC Operator 1"

/user add \
    name=noc2 \
    password="N0cUser2@2026!" \
    group=noc \
    address=192.168.10.0/24 \
    comment="NOC Operator 2"

# Helpdesk
/user add \
    name=helpdesk1 \
    password="H3lpd3sk@2026!" \
    group=helpdesk \
    comment="Helpdesk Staff"

# Monitoring (for Zabbix/PRTG)
/user add \
    name=monitoring \
    password="M0nitor1ng@2026!" \
    group=monitoring \
    address=192.168.10.50/32 \
    comment="Monitoring System API"

# Disable Default Admin
/user disable admin
```

### Step 4: ตั้งค่า SSH Keys

```bash
# สร้าง SSH Key สำหรับ sysadmin
ssh-keygen -t ed25519 -C "sysadmin@company.com" -f ~/.ssh/router_sysadmin
```

```routeros
# Upload Public Key
# วิธี 1: ผ่าน SCP
# $ scp -P 2222 ~/.ssh/router_sysadmin.pub sysadmin@192.168.10.1:/
# แล้วใน RouterOS:
/user ssh-keys import user=sysadmin public-key-file=router_sysadmin.pub

# วิธี 2: Copy-paste Key โดยตรง
/user ssh-keys add \
    user=sysadmin \
    key="ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... sysadmin@company.com"

# ดู SSH Keys
/user ssh-keys print
```

### Step 5: ตั้งค่า Logging

```routeros
# ส่ง Log ไปยัง Syslog Server
/system logging action add \
    name=syslog-server \
    type=remote \
    remote=192.168.10.100 \
    remote-port=514

# Log Account Events
/system logging add \
    action=syslog-server \
    topics=account \
    comment="Log user logins"

# Log System Events
/system logging add \
    action=syslog-server \
    topics=system \
    comment="Log system events"

# Log Script Events (เพื่อ Track Automation)
/system logging add \
    action=syslog-server \
    topics=script \
    comment="Log script execution"
```

### Step 6: ตั้งค่า SSH Hardening

```routeros
# SSH Hardening
/ip ssh set \
    strong-crypto=yes \
    forwarding-enabled=no \
    host-key-size=4096 \
    host-key-type=ed25519

# ดูผลลัพธ์
/ip ssh print
```

### Step 7: Verification

```routeros
# ดู Users ทั้งหมด
/user print

# ดู Groups
/user group print

# ดู Services
/ip service print

# ดู SSH Keys
/user ssh-keys print

# ดู Log สำหรับ Account Events
/log print where topics~"account"
```

### Step 8: ทดสอบ Access

```bash
# ทดสอบ SSH Key Login
ssh -p 2222 -i ~/.ssh/router_sysadmin sysadmin@192.168.10.1

# ทดสอบ NOC Access
ssh -p 2222 noc1@192.168.10.1

# ทดสอบว่า Telnet ไม่ได้
telnet 192.168.10.1
# ควรได้ Connection refused
```

### ผลลัพธ์ที่คาดหวัง

```
/user print
Flags: X - disabled
 #    NAME          GROUP         ADDRESS           LAST-LOGGED-IN
 0 X  admin         full          0.0.0.0           2026-01-01 07:00:00
 1    sysadmin      super-admin   192.168.10.0/24   2026-01-01 09:00:00
 2    noc1          noc           192.168.10.0/24   never
 3    noc2          noc           192.168.10.0/24   never
 4    helpdesk1     helpdesk      0.0.0.0           never
 5    monitoring    monitoring    192.168.10.50/32  never

/ip service print
# NAME    PORT   ADDRESS            CERT  AVAILABLE  DISABLED
# telnet  23               no          yes
# ftp     21               no          yes
# www     80               no          yes
# ssh     2222   192.168.10.0/24    no    yes        no
# www-ssl 443    192.168.10.0/24          yes        no
# api     8728   192.168.10.0/24          yes        no
# winbox  8291   192.168.10.0/24          yes        no
```

---

## Troubleshooting

### ปัญหาที่พบบ่อย

**1. ล็อคตัวเองออกจาก Router**
```routeros
# Physical Console Access เท่านั้น
# ต่อสาย Serial Console และ Login
# หากไม่มี Serial: ใช้ MAC WinBox (เชื่อมต่อ Layer 2)

# ป้องกัน: ใช้ Safe Mode
# กด Ctrl+X เพื่อเข้า Safe Mode
# Router จะ Rollback หากไม่ได้กด Ctrl+X ยืนยัน
```

**2. SSH Key ไม่ทำงาน**
```bash
# Debug SSH Connection
ssh -v -p 2222 sysadmin@192.168.10.1

# ตรวจสอบ Permission ของ Key File
chmod 600 ~/.ssh/router_sysadmin
chmod 644 ~/.ssh/router_sysadmin.pub
```

**3. RADIUS Authentication ล้มเหลว**
```routeros
# ดู Log
/log print where message~"radius"

# ทดสอบ RADIUS
/radius test address=192.168.1.10 secret=shared-secret user=testuser password=testpass

# ตรวจสอบ Connectivity
/ping 192.168.1.10
```

---

## Security Checklist

```
[ ] เปลี่ยน Default Admin Password หรือ Disable แล้ว
[ ] ปิด Telnet และ HTTP
[ ] เปลี่ยน Port ของ SSH
[ ] จำกัด IP ที่สามารถ Manage ได้
[ ] ตั้งค่า SSH Key Authentication
[ ] สร้าง User ตาม Role (Principle of Least Privilege)
[ ] เปิดใช้ Audit Logging
[ ] ส่ง Log ไปยัง Syslog Server
[ ] กำหนด Session Timeout
[ ] ตรวจสอบ Active Sessions เป็นประจำ
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 14:

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| User Accounts | สร้าง/แก้ไขผ่าน `/user` |
| User Groups | กำหนด Policies ต่อ Group |
| Permissions | IP-based + Policy-based |
| SSH Keys | ปลอดภัยกว่า Password |
| 2FA | TOTP สำหรับ RouterOS v7 |
| AAA | Authentication, Authorization, Accounting |
| RADIUS | Centralized Authentication |
| Audit Log | Log ทุก Login/Logout Event |

---

## แบบทดสอบ

1. ทำไม Principle of Least Privilege ถึงสำคัญ?
2. SSH Key ต่างจาก Password Authentication อย่างไร?
3. Policy `sensitive` ให้สิทธิ์อะไร?
4. ถ้า RADIUS ไม่ตอบสนอง RouterOS จะทำอย่างไร? (Default Behavior)
5. อธิบายวิธีล็อค User ให้ Login ได้จาก IP เฉพาะเท่านั้น

---

## Navigation

[← Part 13: Wireless Basics](part-013-wireless-basics.md) | [Part 15: Backup & Restore →](part-015-backup-restore.md)

---

*MikroTik RouterOS Administration Course - Part 14*
*Last Updated: 2026*
