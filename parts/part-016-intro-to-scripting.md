# Part 16: Introduction to Scripting

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate-Advanced
**เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

- [RouterOS Scripting Overview](#overview)
- [Script Editor](#script-editor)
- [Running Scripts](#running-scripts)
- [Script Console](#script-console)
- [Script Environment](#script-environment)
- [Basic Print และ Log Commands](#basic-print-log)
- [Comments](#comments)
- [Script Storage](#script-storage)
- [Security Considerations](#security)
- [Best Practices](#best-practices)
- [Hello World แบบต่างๆ](#hello-world)

---

## RouterOS Scripting Overview

### RouterOS Scripting Language คืออะไร?

**RouterOS Scripting** เป็น Built-in Scripting Language ที่ใช้ Automate งานใน MikroTik Router โดยมีลักษณะเฉพาะคือ:

- ทำงานบน RouterOS โดยตรง ไม่ต้องติดตั้งเพิ่ม
- Syntax คล้าย Bash/Shell Script แต่มีความเฉพาะตัว
- ใช้สั่งการ RouterOS CLI commands ได้โดยตรง
- รองรับ Variables, Loops, Conditions, Functions
- มี Built-in IP/MAC/Time handling

### ความสามารถของ RouterOS Scripting

```
1. Automation
   - Backup อัตโนมัติ
   - Configuration changes ตาม Schedule
   - Auto Failover

2. Monitoring
   - ตรวจสอบ Link Status
   - Monitor Bandwidth
   - Check Service Availability

3. Configuration Management
   - Dynamic Configuration
   - Template-based Deployment
   - Bulk Changes

4. Integration
   - ส่งข้อมูลไปยัง External Systems
   - รับข้อมูลจาก API
   - Email/SMS Notifications
```

### สิ่งที่ต้องรู้ก่อนเริ่ม

```routeros
# RouterOS Scripting พื้นฐาน
# 1. ทุก Command เหมือน CLI Command ปกติ
# 2. Variable เริ่มด้วย : (colon)
# 3. Comment ใช้ # (hash)
# 4. String ใส่ใน "" หรือ ""
# 5. คำสั่ง Built-in เริ่มด้วย : เช่น :put, :log, :if, :for, :while

# ตัวอย่าง Script ง่ายๆ
:put "Hello, MikroTik!"
:log info "Script is running"
/ip address print
```

---

## Script Editor

### วิธีเข้า Script Editor

**ผ่าน WinBox:**
1. ไปที่ Menu → System → Scripts
2. กดปุ่ม `+` เพื่อสร้าง Script ใหม่
3. ใส่ชื่อ Script ใน `Name` field
4. พิมพ์ Code ใน `Source` field

**ผ่าน Terminal:**
```routeros
# เปิด Script Editor ใน Terminal
/system script edit source myscript

# หรือสร้าง Script โดยตรง
/system script add name=myscript source="# Script code here\n:put 'Hello'"
```

**ผ่าน WebFig:**
1. System → Scripts
2. Add → ใส่ชื่อและ Source

### Script Properties

```routeros
# ดู Script Properties
/system script print

# Script Properties ที่สำคัญ:
# name     = ชื่อ Script
# source   = Code ของ Script
# policy   = สิทธิ์ที่ Script ต้องใช้
# comment  = หมายเหตุ
# dont-require-permissions = ไม่ต้องตรวจสอบ Permission
```

### กำหนด Script Policies

```routeros
# Script Policy กำหนดสิ่งที่ Script ทำได้
/system script add \
    name=backup-script \
    policy=ftp,reboot,read,write,policy,test,password,sniff,sensitive,romon \
    source={
        # Script code here
    }

# Policies ที่ Script ใช้บ่อย:
# read     = อ่าน Configuration
# write    = เขียน Configuration
# ftp      = ใช้ FTP
# password = เปลี่ยน Password
# reboot   = Reboot Router
# test     = ทดสอบ (Ping, etc.)
# sensitive = เข้าถึง Sensitive Data
```

---

## Running Scripts

### วิธีรัน Script

```routeros
# วิธีที่ 1: รันจาก Terminal
/system script run myscript

# วิธีที่ 2: รัน Script อีก Script หนึ่ง
/system script run other-script

# วิธีที่ 3: ผ่าน Scheduler (Auto Run)
/system scheduler add \
    name=run-myscript \
    interval=1h \
    on-event=/system/script/run/myscript

# วิธีที่ 4: ผ่าน Netwatch
/tool netwatch add \
    host=8.8.8.8 \
    down-script=/system/script/run/failover-script

# วิธีที่ 5: รัน Script ด้วย Source โดยตรง (Inline)
/system script run [find name=myscript]
```

### รัน Script ใน Terminal โดยตรง

```routeros
# พิมพ์ Code โดยตรงใน Terminal
# (ไม่ต้องสร้าง Script ก่อน)
:local name "World"
:put ("Hello, " . $name . "!")

# หรือใช้ { } เพื่อรัน Block
{
    :local x 10
    :local y 20
    :put ($x + $y)
}
```

---

## Script Console

### Terminal/Console พื้นฐาน

```routeros
# ใน Console ของ RouterOS สามารถ:
# 1. รัน Command CLI ปกติ
/ip address print

# 2. รัน Script Command
:put "Hello"
:log info "Testing"

# 3. รัน Block of Code
{
    :local i 1
    :while ($i <= 5) do={
        :put $i
        :set i ($i + 1)
    }
}
```

### Console Output

```routeros
# แสดงผลลัพธ์ใน Console
:put "ข้อความแสดงใน Console"

# ตัวอย่าง
:local ipAddr [/ip address get [find interface=ether1] address]
:put ("Interface ether1 IP: " . $ipAddr)

# แสดงหลาย Values
:local x 100
:local y 200
:put ("x = " . $x . ", y = " . $y . ", sum = " . ($x + $y))
```

---

## Script Environment

### Environment Variables

```routeros
# ดู Environment Variables ทั้งหมด
/system script environment print

# Environment Variables พิเศษ:
# $nothing    = nil/null value
# $true       = Boolean true
# $false      = Boolean false
# $null       = nil/null value

# ตัวอย่างการใช้
:if ($x = $nothing) do={
    :put "x is undefined/nothing"
}
```

### Global vs Local Scope

```routeros
# Local Variable - ใช้ได้เฉพาะใน Script นั้น
:local myLocalVar "local value"

# Global Variable - ใช้ได้ทุก Script
:global myGlobalVar "global value"

# Script A: กำหนด Global
:global sharedData "Hello from Script A"

# Script B: อ่าน Global จาก Script A
:global sharedData
:put $sharedData    # แสดง "Hello from Script A"
```

### Script Parameters

```routeros
# Script รับ Parameters ผ่านตัวแปร $0, $1, ...
# (ใน RouterOS ใช้ Global Variable แทน)

# วิธีส่ง Parameters
:global param1 "value1"
:global param2 "value2"
/system script run myscript

# ใน Script รับ Parameters
:global param1
:global param2
:put ("Param1: " . $param1 . ", Param2: " . $param2)
```

---

## Basic Print และ Log Commands

### :put Command

```routeros
# แสดงข้อความใน Console
:put "Hello, World!"

# แสดง Variable
:local name "MikroTik"
:put $name

# แสดงผลของ Expression
:local x 10
:local y 20
:put ($x + $y)

# แสดงหลายค่าใน String
:put ("Result: " . $x . " + " . $y . " = " . ($x + $y))

# แสดง Boolean
:put true
:put false

# แสดง IP Address
:local ip 192.168.1.1
:put $ip
```

### :log Command

```routeros
# บันทึก Log ระดับต่างๆ
:log debug "Debug message - ใช้ตอน Debug เท่านั้น"
:log info "Info message - ข้อมูลทั่วไป"
:log warning "Warning - ข้อเตือน"
:log error "Error - ข้อผิดพลาด"
:log critical "Critical - วิกฤต"

# ดู Log
/log print

# ดู Log ที่เกี่ยวกับ Script
/log print where topics~"script"

# ดู Log แบบ Realtime
/log print follow
```

### Log Levels

| Level | คำอธิบาย | ใช้เมื่อ |
|-------|---------|---------|
| debug | ข้อมูล Debug | กำลัง Debug Script |
| info | ข้อมูลทั่วไป | ขั้นตอนสำคัญ |
| warning | ข้อเตือน | มีความผิดปกติแต่ยังทำงานได้ |
| error | ข้อผิดพลาด | ทำงานผิดพลาด |
| critical | วิกฤต | ระบบมีปัญหาร้ายแรง |

### :error Command

```routeros
# ออกจาก Script พร้อมแสดง Error
:error "Something went wrong!"

# ใช้ร่วมกับ Condition
:local x 0
:if ($x = 0) do={
    :error "Division by zero!"
}

# Error Handling ด้วย :do ... :on-error
:do {
    :error "Intentional error"
} on-error={
    :log warning "Caught an error!"
    :put "Script continues after error handling"
}
```

---

## Comments

### การใช้ Comments

```routeros
# นี่คือ Comment แบบ Single Line

# ===== Section Header =====
# ใช้ # ได้ทุกที่ที่ต้องการ

:local x 10  # Inline comment หลังคำสั่ง

# Comment หลายบรรทัด (ทำทีละบรรทัด)
# บรรทัดที่ 1
# บรรทัดที่ 2
# บรรทัดที่ 3

# Documentation Comment สำหรับ Script
# Script: auto-backup.rsc
# Author: Admin
# Version: 1.0
# Date: 2026-01-01
# Description: Automated daily backup script
# Usage: /system script run auto-backup
```

### Best Practice สำหรับ Comments

```routeros
# ===================================================
# Script: bandwidth-monitor.rsc
# Version: 2.1
# Author: Network Team
# Created: 2026-01-01
# Modified: 2026-03-15
# ===================================================
# Purpose:
#   Monitor bandwidth usage per interface and
#   send alert when threshold exceeded
# ===================================================
# Configuration Variables
# ===================================================

:local threshold 90        # Alert when above 90%
:local checkInterval 60    # Check every 60 seconds
:local alertEmail "noc@company.com"

# ===================================================
# Main Logic
# ===================================================

# Get interface stats
:local txBps [/interface get ether1 tx-byte]

# Check threshold
:if ($txBps > $threshold) do={
    # Send alert
    :log warning "Bandwidth threshold exceeded"
}
```

---

## Script Storage

### จัดเก็บ Script

```routeros
# ดู Scripts ที่มีอยู่
/system script print

# Script เก็บไว้ใน Router Memory
# ดูขนาดและรายละเอียด
/system script print detail

# ดูจำนวน Script
/system script print count-only
```

### นำเข้า/ส่งออก Script

```routeros
# Export Script เป็นไฟล์
/system script export file=my-scripts

# Import Script จากไฟล์
/import file=my-scripts.rsc

# เขียน Script ลงไฟล์ในเครื่อง
/file print
/file set [find name=myscript.rsc] contents="# Script code\n:put 'Hello'"
```

### Script ใน Scheduler

```routeros
# Script ใน Scheduler (On-event)
/system scheduler add \
    name=hourly-check \
    interval=1h \
    on-event={
        # Script Code ใส่ตรงนี้ได้เลย
        :local uptime [/system resource get uptime]
        :log info ("Router uptime: " . $uptime)
    }

# หรือเรียก Script ที่มีอยู่แล้ว
/system scheduler add \
    name=daily-backup \
    interval=1d \
    on-event=/system/script/run/backup-script
```

---

## Security Considerations

### Script Security Risks

```routeros
# 1. อย่ารัน Script ที่ไม่รู้ที่มา
# 2. ตรวจสอบ Code ก่อนรัน
# 3. ใช้ Policies ที่จำเป็นเท่านั้น

# ตัวอย่าง: Script ที่มี Minimal Permissions
/system script add \
    name=read-only-script \
    policy=read \         # เฉพาะ read เท่านั้น
    source={
        /ip address print   # OK
        /ip address add address=1.2.3.4/32  # FAIL - ไม่มี write permission
    }
```

### ป้องกัน Script Injection

```routeros
# อย่าใช้ Input จากภายนอกโดยตรงใน Command
# BAD - เสี่ยง Injection
:global userInput
/ip address add address=$userInput  # อันตราย!

# GOOD - Validate ก่อนใช้
:global userInput
:if ($userInput ~ "^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+/[0-9]+\$") do={
    /ip address add address=$userInput
} else={
    :log error "Invalid IP address format"
}
```

### Sensitive Data Handling

```routeros
# อย่า Log Passwords
:local password "secret123"
:log info ("Password: " . $password)  # BAD!

# ใช้ Log แค่ที่จำเป็น
:log info "Authentication successful"  # GOOD - ไม่แสดง Password

# Global Variables ที่มี Sensitive Data ควร Clear หลังใช้งาน
:global apiKey "my-secret-key"
# ... use apiKey ...
:set apiKey ""  # Clear หลังใช้
```

---

## Best Practices

### การตั้งชื่อ

```routeros
# ตั้งชื่อ Script ให้มีความหมาย
# BAD
/system script add name=script1 source=...
/system script add name=test source=...

# GOOD
/system script add name=backup-daily source=...
/system script add name=failover-isp source=...
/system script add name=notify-bandwidth source=...
```

### Error Handling

```routeros
# ใช้ :do ... :on-error เสมอสำหรับ Critical Operations
:do {
    /system backup save name=my-backup
    :log info "Backup created successfully"
} on-error={
    :log error "Backup failed!"
    /tool e-mail send to="admin@example.com" subject="Backup Failed" body="..."
}
```

### Code Organization

```routeros
# แยก Configuration ออกจาก Logic
# ===== CONFIG =====
:local backupServer "192.168.1.100"
:local backupUser "backup"
:local backupPath "/backups/"
:local maxBackups 30

# ===== FUNCTIONS =====
# (กำหนด Functions ที่นี่)

# ===== MAIN =====
# (Logic หลักที่นี่)
```

### Testing

```routeros
# ใช้ :put สำหรับ Debug
:local debug true

:if ($debug) do={
    :put ("Debug: Variable x = " . $x)
}

# ใน Production ปิด Debug
:local debug false
```

### Version Control

```routeros
# ใส่ Version ใน Script
# Version: 1.0.0
# Changelog:
#   1.0.0 - Initial release
#   1.1.0 - Added email notification
#   1.2.0 - Fixed FTP upload bug

:local scriptVersion "1.2.0"
:log info ("Running script version: " . $scriptVersion)
```

---

## Hello World แบบต่างๆ

### 1. Hello World พื้นฐาน

```routeros
# วิธีที่ 1: Simple :put
:put "Hello, World!"
```

### 2. Hello World ใน Log

```routeros
# วิธีที่ 2: Log ทุก Level
:log debug "Hello, World! (Debug)"
:log info "Hello, World! (Info)"
:log warning "Hello, World! (Warning)"
:log error "Hello, World! (Error)"
```

### 3. Hello World พร้อม Variable

```routeros
# วิธีที่ 3: Variable
:local greeting "Hello"
:local name "MikroTik World"
:put ($greeting . ", " . $name . "!")
```

### 4. Hello World จาก Router Info

```routeros
# วิธีที่ 4: ดึงข้อมูลจาก Router
:local routerName [/system identity get name]
:local routerOS [/system resource get version]
:put ("Hello from " . $routerName . " running RouterOS " . $routerOS)
```

### 5. Hello World แบบ Loop

```routeros
# วิธีที่ 5: Loop
:for i from=1 to=5 do={
    :put ("Hello, World! (ครั้งที่ " . $i . ")")
}
```

### 6. Hello World แบบมีเงื่อนไข

```routeros
# วิธีที่ 6: Conditional
:local hour [:tonum [:pick [/system clock get time] 0 2]]

:if ($hour >= 5 && $hour < 12) do={
    :put "Good Morning, World!"
} else={
    :if ($hour >= 12 && $hour < 17) do={
        :put "Good Afternoon, World!"
    } else={
        :put "Good Evening, World!"
    }
}
```

### 7. Hello World แบบ Function

```routeros
# วิธีที่ 7: Function
:local greet do={
    :local target $1
    :put ("Hello, " . $target . "!")
}

$greet "MikroTik"
$greet "RouterOS"
$greet "World"
```

### 8. Hello World แบบ Email

```routeros
# วิธีที่ 8: ส่ง Email
/tool e-mail send \
    to="admin@example.com" \
    subject="Hello from MikroTik!" \
    body="Hello, World! This is a test email from RouterOS."
```

### 9. Hello World แบบ Log + Terminal

```routeros
# วิธีที่ 9: ทั้ง Console และ Log
:local msg "Hello, World!"
:put $msg
:log info $msg
```

### 10. Hello World พร้อม Timestamp

```routeros
# วิธีที่ 10: พร้อม Timestamp
:local now [/system clock get date]
:local time [/system clock get time]
:put ("Hello, World! - " . $now . " " . $time)
:log info ("Hello World Script executed at: " . $now . " " . $time)
```

### 11. Hello World แบบ Script เต็มรูปแบบ

```routeros
# ===================================================
# Hello World - Complete Example
# Version: 1.0
# Author: MikroTik Admin
# ===================================================

# Configuration
:local debug true

# ===== MAIN =====
:local routerName [/system identity get name]
:local uptime [/system resource get uptime]
:local cpuLoad [/system resource get cpu-load]

# Build Message
:local message "=== Hello, World! ===\n"
:set message ($message . "Router: " . $routerName . "\n")
:set message ($message . "Uptime: " . $uptime . "\n")
:set message ($message . "CPU Load: " . $cpuLoad . "%\n")
:set message ($message . "==================")

# Output
:put $message
:log info $message

:if ($debug) do={
    :put "[DEBUG] Script completed successfully"
}
```

---

## ตัวอย่าง Scripts ที่ใช้งานได้จริง

### Script 1: System Status Report

```routeros
/system script add name=system-status source={
    :local routerName [/system identity get name]
    :local version [/system resource get version]
    :local uptime [/system resource get uptime]
    :local cpuLoad [/system resource get cpu-load]
    :local freeMemory [/system resource get free-memory]
    :local totalMemory [/system resource get total-memory]
    :local memUsedPct (100 - (($freeMemory * 100) / $totalMemory))

    :put "==================="
    :put "  System Status"
    :put "==================="
    :put ("Name:     " . $routerName)
    :put ("Version:  " . $version)
    :put ("Uptime:   " . $uptime)
    :put ("CPU Load: " . $cpuLoad . "%")
    :put ("Memory:   " . $memUsedPct . "% used")
    :put "==================="
}
```

### Script 2: Interface Down Alert

```routeros
/system script add name=check-interfaces source={
    :foreach iface in=[/interface find type=ether] do={
        :local ifName [/interface get $iface name]
        :local ifStatus [/interface get $iface running]

        :if (!$ifStatus) do={
            :log warning ("Interface DOWN: " . $ifName)
        }
    }
}
```

### Script 3: Simple Ping Test

```routeros
/system script add name=ping-test source={
    :local hosts {
        "8.8.8.8";
        "1.1.1.1";
        "192.168.1.1"
    }

    :foreach host in=$hosts do={
        :local result [/ping $host count=3 as-value]
        :local received ($result->"received")

        :if ($received > 0) do={
            :put ($host . ": OK (" . $received . "/3)")
        } else={
            :put ($host . ": FAILED")
            :log warning ("Host unreachable: " . $host)
        }
    }
}
```

---

## Exercises

1. เขียน Script แสดงชื่อ Router + IP ของ Interface ether1
2. เขียน Script นับ Interfaces ทั้งหมด
3. เขียน Script แสดง Uptime และแจ้งเตือนถ้า Uptime น้อยกว่า 1 ชั่วโมง (อาจ Rebooted)
4. เขียน Script แสดง Active DHCP Leases ทั้งหมด
5. เขียน Script ตรวจสอบว่ามีไฟล์ Backup ของวันนี้ไหม

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 16:

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| Script Overview | Automation Language ใน RouterOS |
| Script Editor | WinBox, Terminal, WebFig |
| Running Scripts | `/system script run`, Scheduler, Netwatch |
| :put | แสดงข้อมูลใน Console |
| :log | บันทึก Log ระดับต่างๆ |
| Comments | # สำหรับ Documentation |
| Security | ตรวจสอบ Policies, Validate Input |
| Best Practices | Error Handling, Naming, Organization |

---

## Navigation

[← Part 15: Backup & Restore](part-015-backup-restore.md) | [Part 17: Variables & Data Types →](part-017-variables-datatypes.md)

---

*MikroTik RouterOS Administration Course - Part 16*
*Last Updated: 2026*
