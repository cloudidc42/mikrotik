# Part 15: Backup & Restore

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate
**เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

- [Backup Types](#backup-types)
- [Manual Backup](#manual-backup)
- [Automated Backup Scripts](#automated-backup-scripts)
- [Restore Procedure](#restore-procedure)
- [Configuration Export/Import](#configuration-exportimport)
- [Safe Mode](#safe-mode)
- [Config Rollback](#config-rollback)
- [Version Control for Configs](#version-control)
- [Disaster Recovery](#disaster-recovery)
- [Lab: Backup Automation Script](#lab)

---

## Backup Types

### ประเภทของ Backup ใน RouterOS

| ประเภท | Extension | คำอธิบาย | เปิดอ่านได้? |
|--------|-----------|---------|------------|
| Binary Backup | `.backup` | เต็ม Configuration ทั้งหมด (Binary) | ไม่ |
| Export (RSC) | `.rsc` | Script ที่อ่านได้ด้วยตาคน | ใช่ |
| Sensitive Export | `.rsc` | Export รวม Passwords และ Keys | ใช่ |

### เปรียบเทียบ Binary vs Export

| Feature | Binary Backup | Export (RSC) |
|---------|--------------|-------------|
| Format | Binary | Plain Text Script |
| Passwords | เก็บไว้ | ซ่อน (default) หรือ เก็บ |
| RouterOS Version | ต้องใกล้เคียงกัน | ยืดหยุ่นกว่า |
| Selective Restore | ไม่ | ใช่ |
| Human Readable | ไม่ | ใช่ |
| Version Control | ยาก | ง่าย (Git) |
| Speed | เร็ว | ช้ากว่าเล็กน้อย |

### เมื่อไรควรใช้ประเภทไหน

```
Binary Backup:
  ✓ Full Backup ก่อนทำการเปลี่ยนแปลง
  ✓ Disaster Recovery
  ✓ Clone Router

Export (RSC):
  ✓ Version Control ด้วย Git
  ✓ Documentation
  ✓ Transfer Config บางส่วนไป Router อื่น
  ✓ Review ก่อน Apply
```

---

## Manual Backup

### Binary Backup

```routeros
# Backup แบบ Binary (เก็บไว้ใน Router)
/system backup save name=myrouter-backup

# ระบุชื่อพร้อมวันที่
/system backup save name=("backup-" . [:tostr [/system clock get date]])

# Backup พร้อม Password (เข้ารหัสไฟล์)
/system backup save name=secure-backup password="BackupPass123!"

# ดูไฟล์ Backup ใน Router
/file print

# Backup ที่รวม Sensitive Information
/system backup save name=full-backup include-password=yes
```

### Download Backup File

```bash
# Download ผ่าน SCP
scp -P 2222 admin@192.168.1.1:/myrouter-backup.backup ~/backups/

# Download ผ่าน FTP
ftp 192.168.1.1
# login แล้ว get myrouter-backup.backup

# ผ่าน WinBox: Files → ลาก File มา Desktop
```

### Export (RSC)

```routeros
# Export ทั้งหมด
/export file=full-export

# Export พร้อม Passwords (Sensitive)
/export file=full-export-sensitive hide-sensitive=no

# Export เฉพาะส่วนที่ต้องการ
/ip address export file=ip-addresses
/ip route export file=routes
/ip firewall filter export file=firewall-rules
/interface export file=interfaces

# Export แบบ Compact (ไม่แสดง Default Values)
/export compact file=compact-export

# Export พร้อมวันที่ในชื่อ
/export file=("export-" . [:tostr [/system clock get date]])
```

### ดูผลของ Export

```routeros
# Export ออกหน้าจอ (Terminal)
/export

# Export เฉพาะส่วน
/ip address export
/ip route export
/interface wireless export

# Export ส่วนใดส่วนหนึ่ง
/ip firewall filter export
```

ตัวอย่าง Output ของ Export:
```
# jan/01/2026 08:00:00 by RouterOS 7.x
# software id = XXXX-XXXX
#
# model = RouterBOARD 952Ui-5ac2nD
# serial number = XXXXXXXXXXXX
/interface bridge
add comment="Main Bridge" name=bridge1 protocol-mode=rstp
/interface vlan
add interface=bridge1 name=vlan10 vlan-id=10
/ip address
add address=192.168.1.1/24 interface=bridge1 network=192.168.1.0
/ip route
add distance=1 dst-address=0.0.0.0/0 gateway=203.0.113.1
```

---

## Automated Backup Scripts

### Script Backup รายวัน

```routeros
# Script สร้าง Backup รายวัน
:local backupName ("daily-" . [:tostr [/system clock get date]])
:local backupFile ($backupName . ".backup")
:local exportFile ($backupName . ".rsc")

# สร้าง Binary Backup
/system backup save name=$backupName
:log info "Binary backup created: $backupFile"

# สร้าง Export
/export file=$backupName
:log info "Export created: $exportFile"

# ลบ Backup ที่เก่ากว่า 30 วัน
:foreach f in=[/file find name~"daily-"] do={
    :local fileAge [/file get $f creation-time]
    # ลบถ้าเก่ากว่า 30 วัน
    /file remove $f
}
```

### สร้าง Scheduler สำหรับ Auto Backup

```routeros
# สร้าง Script ชื่อ "auto-backup"
/system script add \
    name=auto-backup \
    policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon \
    source={
        :local date [/system clock get date]
        :local time [/system clock get time]
        :local name ("backup-" . $date)

        # Binary Backup
        /system backup save name=$name

        # Export
        /export file=$name

        :log info ("Auto backup completed: " . $name)
    }

# สร้าง Scheduler รัน Script ทุกคืน 02:00
/system scheduler add \
    name=daily-backup \
    interval=1d \
    on-event=auto-backup \
    start-time=02:00:00 \
    comment="Daily Backup at 2AM"
```

### ส่ง Backup ไปยัง FTP Server

```routeros
# Script Backup พร้อมส่ง FTP
/system script add \
    name=backup-to-ftp \
    source={
        :local ftpServer "192.168.1.100"
        :local ftpUser "backupuser"
        :local ftpPass "FTPPass123"
        :local date [/system clock get date]
        :local routerName [/system identity get name]
        :local backupName ($routerName . "-" . $date)

        # สร้าง Backup
        /system backup save name=$backupName
        /export file=$backupName

        # Upload ผ่าน FTP
        /tool fetch \
            url=("ftp://" . $ftpUser . ":" . $ftpPass . "@" . $ftpServer . "/" . $backupName . ".backup") \
            src-path=($backupName . ".backup") \
            upload=yes \
            mode=ftp

        /tool fetch \
            url=("ftp://" . $ftpUser . ":" . $ftpPass . "@" . $ftpServer . "/" . $backupName . ".rsc") \
            src-path=($backupName . ".rsc") \
            upload=yes \
            mode=ftp

        :log info ("Backup uploaded to FTP: " . $backupName)
    }
```

### ส่ง Backup ผ่าน Email

```routeros
# ตั้งค่า Email ก่อน
/tool e-mail set \
    server=smtp.gmail.com \
    port=587 \
    from=router@example.com \
    user=router@example.com \
    password=AppPassword123

# Script Backup พร้อมส่ง Email
/system script add \
    name=backup-email \
    source={
        :local date [/system clock get date]
        :local routerName [/system identity get name]
        :local backupName ($routerName . "-" . $date)

        # สร้าง Backup
        /system backup save name=$backupName
        /export file=($backupName . ".rsc")

        # ส่ง Email พร้อม Attachment
        /tool e-mail send \
            to="admin@example.com" \
            subject=("[Backup] " . $routerName . " - " . $date) \
            body=("Router backup completed.\nDate: " . $date . "\nRouter: " . $routerName) \
            file=($backupName . ".backup")

        :log info "Backup emailed successfully"
    }
```

### Cleanup Script

```routeros
# Script ลบ Backup เก่า (เก็บไว้ 7 ชุดล่าสุด)
/system script add \
    name=cleanup-backups \
    source={
        :local maxBackups 7
        :local backupFiles [/file find name~"backup-"]
        :local count [:len $backupFiles]

        :if ($count > $maxBackups) do={
            :local toDelete ($count - $maxBackups)
            :for i from=0 to=($toDelete - 1) do={
                :local file ($backupFiles->$i)
                /file remove $file
                :log info ("Deleted old backup: " . [/file get $file name])
            }
        }
    }
```

---

## Restore Procedure

### Restore จาก Binary Backup

```routeros
# วิธีที่ 1: ผ่าน RouterOS Command
# Upload ไฟล์ไปยัง Router ก่อน (ผ่าน FTP/SCP/WinBox)
/system backup load name=myrouter-backup

# หากมี Password
/system backup load name=secure-backup password="BackupPass123!"

# Router จะ Reboot อัตโนมัติหลัง Restore
```

```bash
# วิธีที่ 2: Upload ผ่าน SCP แล้ว Restore
scp -P 2222 ~/backups/myrouter-backup.backup admin@192.168.1.1:/

# แล้ว Login และ Restore
ssh -p 2222 admin@192.168.1.1
# /system backup load name=myrouter-backup
```

### Restore จาก Export (RSC)

```routeros
# วิธีที่ 1: Import ไฟล์
/import file=full-export.rsc

# วิธีที่ 2: Copy-paste ใน Terminal
# เปิดไฟล์ .rsc และ Copy-paste ใน Terminal

# วิธีที่ 3: ผ่าน WinBox Terminal
# File → Import ใน WinBox
```

### Import เฉพาะส่วน

```routeros
# ลบ Config เดิมก่อน (ถ้าต้องการ Clean Import)
# ระวัง!! คำสั่งนี้ล้าง Config ทั้งหมด
/system reset-configuration no-defaults=yes

# Import เฉพาะ Firewall Rules
/import file=firewall-rules.rsc

# Import เฉพาะ Routes
/import file=routes.rsc
```

### Restore Verification

```routeros
# หลัง Restore ตรวจสอบ:

# 1. IP Addresses
/ip address print

# 2. Routes
/ip route print

# 3. Firewall Rules
/ip firewall filter print
/ip firewall nat print

# 4. Users
/user print

# 5. Services
/ip service print

# 6. Interfaces
/interface print

# 7. Test Connectivity
/ping 8.8.8.8
```

---

## Configuration Export/Import

### Export เฉพาะส่วน

```routeros
# Export ตาราง Firewall
/ip firewall filter export file=fw-filter
/ip firewall nat export file=fw-nat
/ip firewall mangle export file=fw-mangle

# Export Routing
/ip route export file=routes
/routing ospf instance export file=ospf

# Export DHCP
/ip dhcp-server export file=dhcp-server
/ip dhcp-server network export file=dhcp-network

# Export Wireless
/interface wireless export file=wireless-config
/interface wireless security-profiles export file=wireless-security

# Export Users
/user export file=users  # ไม่รวม Passwords (Sensitive)
/user export file=users-full hide-sensitive=no  # รวม Passwords
```

### Selective Import

```routeros
# Import Firewall Rules ใหม่ (ไม่ลบอันเดิม - เพิ่มต่อท้าย)
/import file=fw-filter.rsc

# หากต้องการ Clean Import Firewall
/ip firewall filter remove [find]  # ลบทั้งหมด
/import file=fw-filter.rsc         # Import ใหม่
```

### Export เพื่อ Documentation

```routeros
# Export ทุกอย่างเป็น Human-readable Script
/export verbose file=full-verbose-export

# Export เฉพาะที่ต่างจาก Default
/export compact file=changes-only

# ดู Export ใน Terminal
/export compact
```

---

## Safe Mode

### Safe Mode คืออะไร?

**Safe Mode** คือ Mode พิเศษที่ Router จะ Rollback Configuration อัตโนมัติถ้าไม่ได้รับการยืนยันภายใน Timeout

### ใช้ Safe Mode

```routeros
# เข้า Safe Mode
# ใน WinBox: กดปุ่ม [Safe Mode] หรือ Ctrl+X
# ใน Terminal: กด Ctrl+X

# เมื่อเข้า Safe Mode จะเห็น:
# [Safe Mode taken]
# ทุกการเปลี่ยนแปลงจะ Rollback หากเราออกจาก Terminal โดยไม่ยืนยัน

# ทำการเปลี่ยนแปลงที่ต้องการ
/ip firewall filter add chain=input action=drop

# ถ้าต้องการ Save การเปลี่ยนแปลง: ออกจาก Safe Mode (Ctrl+X อีกครั้ง)
# ถ้าต้องการ Rollback: ปิด Terminal หรือรอ Timeout
```

### Safe Mode ใน Script

```routeros
# ใช้ Safe Mode ใน Script ด้วย :do ... :on-error
:do {
    # เปลี่ยนแปลงที่อาจมีปัญหา
    /ip firewall filter add chain=input src-address=10.0.0.0/8 action=drop

    # Test ว่ายังเชื่อมต่อได้ไหม
    :local test [/ping 8.8.8.8 count=3]
    :if ($test = 0) do={
        :error "Lost connectivity - rolling back"
    }

    :log info "Changes applied successfully"
} on-error={
    # Rollback เมื่อเกิด Error
    /ip firewall filter remove [find src-address=10.0.0.0/8]
    :log warning "Changes rolled back due to error"
}
```

---

## Config Rollback

### Manual Rollback

```routeros
# กรณีทำการเปลี่ยนแปลงแล้วมีปัญหา

# วิธีที่ 1: Restore จาก Backup ล่าสุด
/system backup load name=pre-change-backup

# วิธีที่ 2: Undo การเปลี่ยนแปลงด้วยมือ
# ตรวจ Log เพื่อดูว่าเปลี่ยนอะไร
/log print where topics~"system"

# วิธีที่ 3: Reset to Default
/system reset-configuration no-defaults=no
```

### Pre-Change Backup Workflow

```routeros
# Best Practice: Backup ก่อนเปลี่ยนแปลงทุกครั้ง

# 1. Backup ก่อน
/system backup save name=("pre-change-" . [/system clock get date] . "-" . [/system clock get time])

# 2. Export ก่อน
/export file=("pre-change-export-" . [/system clock get date])

# 3. ทำการเปลี่ยนแปลง
# ... (your changes here) ...

# 4. ทดสอบ
/ping 8.8.8.8

# 5. ถ้ามีปัญหา
/system backup load name=pre-change-backup-date
```

---

## Version Control for Configs

### ใช้ Git สำหรับ Config

```bash
# สร้าง Git Repository สำหรับ Config
mkdir ~/router-configs
cd ~/router-configs
git init

# สร้างโครงสร้าง
mkdir -p routers/router1
mkdir -p routers/router2

# Script ดึง Config จาก Router
#!/bin/bash
ROUTER_IP="192.168.1.1"
ROUTER_USER="sysadmin"
DATE=$(date +%Y-%m-%d)
OUTPUT_DIR="routers/router1"

# Export Config
ssh -p 2222 $ROUTER_USER@$ROUTER_IP "/export hide-sensitive=no" > $OUTPUT_DIR/config-$DATE.rsc

# Git Commit
git add $OUTPUT_DIR/
git commit -m "Auto-backup $DATE: Router1"
```

### Script Export และ Push Git

```routeros
# RouterOS Script ส่ง Export ไปยัง Git Server
/system script add \
    name=git-backup \
    source={
        :local date [/system clock get date]
        :local routerName [/system identity get name]
        :local filename ($routerName . "-" . $date . ".rsc")

        # Export
        /export file=$filename hide-sensitive=no

        # Upload ไปยัง Server ด้วย SFTP/HTTP
        /tool fetch \
            url=("http://backup-server/upload?router=" . $routerName . "&date=" . $date) \
            src-path=$filename \
            upload=yes \
            method=post

        :log info ("Config exported and uploaded: " . $filename)
    }
```

### ตัวอย่าง Git Config Structure

```
router-configs/
├── README.md
├── scripts/
│   ├── backup.sh         # Script ดึง Config
│   └── restore.sh        # Script Restore
├── routers/
│   ├── router1/
│   │   ├── config.rsc    # Latest Config
│   │   └── history/      # เก็บ History
│   └── router2/
│       ├── config.rsc
│       └── history/
└── templates/
    ├── firewall.rsc      # Firewall Template
    └── basic-setup.rsc   # Basic Setup Template
```

---

## Disaster Recovery

### Disaster Recovery Plan

```
ขั้นตอน Disaster Recovery:

1. ประเมินความเสียหาย
   - Router Crash หรือแค่ Config ผิด?
   - มีอุปกรณ์สำรองไหม?

2. Recovery Options:
   a. Software Recovery: Restore Config
   b. Hardware Recovery: เปลี่ยน Router + Restore Config
   c. Netinstall: Reset + Reinstall RouterOS

3. ลำดับการ Restore:
   1. ติดตั้ง RouterOS (ถ้าจำเป็น)
   2. ตั้งค่า IP เพื่อ Management
   3. Restore Backup
   4. ตรวจสอบ Connectivity
   5. ทดสอบ Services

4. Post-Recovery:
   - ทดสอบทุก Services
   - อัปเดต Documentation
   - หา Root Cause
   - ปรับปรุง Backup Strategy
```

### Netinstall (Reset + Reinstall)

```
Netinstall ใช้เมื่อ:
- RouterOS เสียหายรุนแรง
- ลืม Password ทั้งหมด
- ต้องการ Fresh Install

ขั้นตอน:
1. Download Netinstall จาก mikrotik.com
2. ต่อสาย Ethernet PC กับ Port 1 ของ Router
3. กดค้าง Reset Button ขณะเปิดไฟ
4. รัน Netinstall บน PC
5. เลือก Package และ Install
```

### Reset Configuration

```routeros
# Reset เป็น Factory Default (ระวัง! ลบทุกอย่าง)
/system reset-configuration

# Reset แต่ยังเก็บ Key Pairs
/system reset-configuration keep-users=yes

# Reset และ Run Default Script
/system reset-configuration run-after-reset=default-script.rsc

# Reset โดยไม่มี Default Config
/system reset-configuration no-defaults=yes
```

---

## Lab: Backup Automation Script

### Objectives

1. สร้าง Complete Backup Script ที่ทำงานอัตโนมัติ
2. Backup ทั้ง Binary และ Export
3. ส่งไปยัง FTP Server
4. ส่ง Email Notification
5. Cleanup Backup เก่า
6. สร้าง Scheduler

### Script สมบูรณ์

```routeros
# ===================================================
# Auto Backup Script - Complete Version
# ===================================================
# นำไปใส่ใน: /system script name=auto-backup-full
# ===================================================

:local routerName [/system identity get name]
:local date [/system clock get date]
:local time [/system clock get time]
:local timestamp ($date . "_" . [:pick $time 0 5])
:local backupName ($routerName . "_" . $timestamp)

# Configuration
:local ftpServer "192.168.1.100"
:local ftpUser "backupuser"
:local ftpPass "FTPBackup123"
:local emailTo "admin@example.com"
:local maxLocalBackups 10

# ===== สร้าง Backup =====
:log info ("Starting backup: " . $backupName)

# Binary Backup
:do {
    /system backup save name=$backupName password=""
    :log info ("Binary backup created: " . $backupName . ".backup")
} on-error={
    :log error "Failed to create binary backup"
}

# Export (RSC)
:do {
    /export file=$backupName hide-sensitive=no
    :log info ("Export created: " . $backupName . ".rsc")
} on-error={
    :log error "Failed to create export"
}

# ===== Upload ไปยัง FTP =====
:do {
    /tool fetch \
        url=("ftp://" . $ftpUser . ":" . $ftpPass . "@" . $ftpServer . "/backups/" . $backupName . ".backup") \
        src-path=($backupName . ".backup") \
        upload=yes \
        mode=ftp
    :log info "Binary backup uploaded to FTP"
} on-error={
    :log warning "Failed to upload binary backup to FTP"
}

:do {
    /tool fetch \
        url=("ftp://" . $ftpUser . ":" . $ftpPass . "@" . $ftpServer . "/backups/" . $backupName . ".rsc") \
        src-path=($backupName . ".rsc") \
        upload=yes \
        mode=ftp
    :log info "Export uploaded to FTP"
} on-error={
    :log warning "Failed to upload export to FTP"
}

# ===== Cleanup Local Backups =====
:local backupFiles [/file find name~($routerName . "_")]
:local backupCount [:len $backupFiles]

:if ($backupCount > $maxLocalBackups) do={
    :local toDelete ($backupCount - $maxLocalBackups)
    :log info ("Cleaning up " . $toDelete . " old backup files")
    :for i from=0 to=($toDelete - 1) do={
        :local filename [/file get ($backupFiles->$i) name]
        /file remove ($backupFiles->$i)
        :log info ("Deleted: " . $filename)
    }
}

# ===== ส่ง Email Notification =====
:local emailBody ("Router Backup Report\n")
:set emailBody ($emailBody . "========================\n")
:set emailBody ($emailBody . "Router: " . $routerName . "\n")
:set emailBody ($emailBody . "Date: " . $date . "\n")
:set emailBody ($emailBody . "Time: " . $time . "\n")
:set emailBody ($emailBody . "Backup: " . $backupName . "\n")
:set emailBody ($emailBody . "========================\n")
:set emailBody ($emailBody . "Backup completed successfully!")

:do {
    /tool e-mail send \
        to=$emailTo \
        subject=("[Backup OK] " . $routerName . " - " . $date) \
        body=$emailBody
    :log info "Backup notification email sent"
} on-error={
    :log warning "Failed to send backup notification email"
}

:log info ("Backup completed: " . $backupName)
```

### สร้าง Scheduler

```routeros
# สร้าง Scheduler รัน Script ทุกคืน 02:30
/system scheduler add \
    name=nightly-backup \
    interval=1d \
    start-time=02:30:00 \
    on-event=auto-backup-full \
    policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon \
    comment="Nightly backup at 2:30 AM"

# สร้าง Scheduler Weekly (ทุกอาทิตย์)
/system scheduler add \
    name=weekly-backup \
    interval=7d \
    start-time=sun 01:00:00 \
    on-event=auto-backup-full \
    comment="Weekly backup on Sunday at 1 AM"

# ดู Schedulers
/system scheduler print
```

### ทดสอบ Script

```routeros
# รัน Script ด้วยมือ
/system script run auto-backup-full

# ดู Log หลังรัน
/log print where message~"backup"

# ดูไฟล์ที่สร้าง
/file print where name~"Router1"
```

### ตัวอย่างผลลัพธ์

```
/file print
 #  NAME                              TYPE         SIZE        CREATION-TIME
 0  Router1_jan/01/2026_02:30.backup  backup       245.3 KiB   jan/01/2026 02:30:15
 1  Router1_jan/01/2026_02:30.rsc     script       48.7 KiB    jan/01/2026 02:30:16
 2  Router1_dec/31/2025_02:30.backup  backup       244.8 KiB   dec/31/2025 02:30:14
 3  Router1_dec/31/2025_02:30.rsc     script       48.5 KiB    dec/31/2025 02:30:15
```

---

## Backup Verification Script

```routeros
# Script ตรวจสอบว่า Backup ทำได้สำเร็จ
/system script add \
    name=verify-backup \
    source={
        :local routerName [/system identity get name]
        :local today [/system clock get date]
        :local expectedFile ($routerName . "_" . $today)
        :local found false

        # ตรวจหาไฟล์ Backup ของวันนี้
        :foreach f in=[/file find] do={
            :if ([/file get $f name] ~ $expectedFile) do={
                :set found true
            }
        }

        :if ($found) do={
            :log info "Backup verification PASSED: Backup found for today"
        } else={
            :log error "Backup verification FAILED: No backup found for today!"
            # ส่ง Alert Email
            /tool e-mail send \
                to="admin@example.com" \
                subject=("[ALERT] Backup Missing: " . $routerName) \
                body=("WARNING: No backup found for " . $today . " on " . $routerName)
        }
    }

# Schedule ตรวจสอบตอนเช้า
/system scheduler add \
    name=verify-backup-morning \
    interval=1d \
    start-time=08:00:00 \
    on-event=verify-backup \
    comment="Verify backup at 8 AM"
```

---

## Troubleshooting

### ปัญหาที่พบบ่อย

**1. ไม่สามารถ Restore Backup ได้**
```routeros
# ตรวจสอบว่า RouterOS Version ตรงกัน
/system resource print | find version

# ตรวจสอบ Backup Password
/system backup load name=my-backup password="correct-password"

# ถ้ายังไม่ได้ ลอง Export/Import แทน
```

**2. Export ไม่ครบ**
```routeros
# Export รวม Sensitive Data
/export hide-sensitive=no file=full-export

# ตรวจสอบว่า Export ครบ
/export | count-lines
```

**3. FTP Upload ล้มเหลว**
```routeros
# ทดสอบ FTP Connection
/tool fetch url=ftp://username:password@192.168.1.100/ mode=ftp

# ตรวจสอบ Firewall บน FTP Server
# ตรวจสอบ Passive/Active FTP Mode
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 15:

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| Backup Types | Binary (.backup) vs Export (.rsc) |
| Manual Backup | `/system backup save` และ `/export` |
| Automated Backup | Script + Scheduler |
| Restore | `/system backup load` และ `/import` |
| Safe Mode | Ctrl+X ใน Terminal, Auto-rollback |
| Version Control | Git สำหรับ Config Management |
| Disaster Recovery | มีขั้นตอนชัดเจน ทดสอบสม่ำเสมอ |

---

## แบบทดสอบ

1. ความแตกต่างระหว่าง Binary Backup กับ Export คืออะไร?
2. เมื่อไรควรใช้ Safe Mode?
3. อธิบายวิธีตั้งค่า Backup อัตโนมัติทุกคืน
4. ถ้า Router ลืม Password ทุกช่องทาง จะ Recover ได้อย่างไร?
5. ทำไมต้องส่ง Backup ออกไปเก็บที่ภายนอก Router?

---

## Navigation

[← Part 14: User Management](part-014-user-management.md) | [Part 16: Intro to Scripting →](part-016-intro-to-scripting.md)

---

*MikroTik RouterOS Administration Course - Part 15*
*Last Updated: 2026*
