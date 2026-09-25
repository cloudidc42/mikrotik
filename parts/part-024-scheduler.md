# Part 24: Scheduler ใน RouterOS

## บทนำ

RouterOS Scheduler เป็นเครื่องมือสำคัญสำหรับ automation ช่วยให้เราสามารถรันงานอัตโนมัติตามเวลาที่กำหนดได้โดยไม่ต้องมีคนมานั่งทำ เหมาะสำหรับงาน backup, monitoring, maintenance และอื่นๆ

---

## 24.1 Scheduler Overview

### หลักการทำงาน

```
RouterOS Scheduler
├── One-time tasks (run once at specific time)
├── Repeating tasks (run every X seconds/minutes/hours/days)
└── Cron-like tasks (run at specific time patterns)
```

### ดู Scheduled Tasks

```routeros
# แสดง schedulers ทั้งหมด
/system scheduler print

# แสดงรายละเอียด
/system scheduler print detail

# ดูเฉพาะ columns สำคัญ
/system scheduler print columns=name,start-time,interval,next-run

# ค้นหา scheduler ที่ต้องการ
/system scheduler print where name~"backup"
/system scheduler print where disabled=no
```

### Scheduler Properties

| Property | คำอธิบาย | ตัวอย่าง |
|----------|---------|---------|
| name | ชื่อ scheduler | "daily-backup" |
| start-date | วันที่เริ่ม | jan/01/2025 |
| start-time | เวลาเริ่ม | 02:00:00 |
| interval | ความถี่ | 1d, 6h, 30m |
| on-event | script ที่รัน | "/system script run ..." |
| comment | คำอธิบาย | "Daily backup" |
| disabled | เปิด/ปิด | yes/no |
| next-run | เวลารันครั้งต่อไป | (auto-calculated) |

---

## 24.2 Creating Scheduled Tasks

### Basic Scheduler

```routeros
# สร้าง scheduler พื้นฐาน
/system scheduler add \
    name="my-first-scheduler" \
    start-time=12:00:00 \
    interval=1d \
    on-event=":put \"Hello World\"" \
    comment="Test scheduler"

# สร้างแบบระบุวันเริ่ม
/system scheduler add \
    name="monthly-task" \
    start-date=jan/01/2025 \
    start-time=00:00:00 \
    interval=30d \
    on-event="/system script run monthly-report"

# สร้าง scheduler ที่รัน script
/system scheduler add \
    name="run-backup" \
    start-time=02:00:00 \
    interval=1d \
    on-event="/system script run backup-script"
```

### Interval Formats

```routeros
# รูปแบบ interval ต่างๆ
# interval=0        - run once
# interval=30s      - every 30 seconds
# interval=5m       - every 5 minutes
# interval=1h       - every 1 hour
# interval=1d       - every 1 day
# interval=1w       - every 1 week
# interval=00:05:00 - every 5 minutes (HH:MM:SS format)
# interval=1d00:00:00 - every 1 day

# ตัวอย่าง
/system scheduler add \
    name="every-30-seconds" \
    interval=30s \
    on-event=":put \"Tick\""

/system scheduler add \
    name="every-5-minutes" \
    interval=5m \
    on-event="/system script run check-health"

/system scheduler add \
    name="every-6-hours" \
    interval=6h \
    on-event="/system script run update-stats"
```

---

## 24.3 Cron-like Syntax

RouterOS ไม่ใช้ cron syntax โดยตรง แต่เราสามารถ simulate ได้:

```routeros
# Simulate cron: ทุกวันจันทร์ 08:00
# 1. สร้าง scheduler ที่รันทุกวัน
# 2. ใน script ตรวจสอบวันในสัปดาห์

# Script ที่ตรวจสอบวัน
:local weeklyScript "
:local dayOfWeek [:tostr [/system clock get time]]
# Monday = 1 (depends on RouterOS version)
:local today [/system clock get day]
:if ($today = 1) do={
    # Run Monday tasks
    /system script run monday-tasks
}
"

# Scheduler รันทุกวัน 08:00
/system scheduler add \
    name="check-weekday" \
    start-time=08:00:00 \
    interval=1d \
    on-event=$weeklyScript

# Simulate cron: ทุกต้นเดือน
:local monthlyCheckScript "
:local day [/system clock get date]
:local dayNum [:pick [:tostr $day] 4 6]
:if ($dayNum = \"01\") do={
    /system script run monthly-tasks
}
"

/system scheduler add \
    name="monthly-check" \
    start-time=00:00:00 \
    interval=1d \
    on-event=$monthlyCheckScript
```

### Time-based Conditions ใน Scripts

```routeros
# ตรวจสอบชั่วโมงปัจจุบัน
:local currentTime [/system clock get time]
:local hour [:pick [:tostr $currentTime] 0 2]

:if ($hour = "09") do={
    :put "Morning tasks..."
}

:if ($hour >= "17") do={
    :put "Evening tasks..."
}

# Business hours check
:local isBusinessHours do={
    :local time [/system clock get time]
    :local hour [:toint [:pick [:tostr $time] 0 2]]
    :return ($hour >= 9 && $hour < 18)
}

:if ([$isBusinessHours]) do={
    :put "Business hours - full service"
} else={
    :put "Off hours - reduced service"
}
```

---

## 24.4 One-time Tasks

```routeros
# รัน script ครั้งเดียวในอนาคต
# ใช้ interval=0 (หรือไม่ระบุ interval)

# รัน ณ เวลาที่กำหนด (ครั้งเดียว)
/system scheduler add \
    name="one-time-maintenance" \
    start-date=jan/15/2025 \
    start-time=23:00:00 \
    interval=0 \
    on-event="/system script run maintenance" \
    comment="One-time maintenance window"

# รัน delay 5 นาทีจากตอนนี้ (ครั้งเดียว)
:local delayScript "
# ใช้ scheduler สร้าง delayed execution
:local currentTime [/system clock get time]
:local currentDate [/system clock get date]
/system scheduler add \
    name=\"delayed-task\" \
    start-date=currentDate \
    start-time=currentTime \
    interval=0 \
    on-event=\"/system script run my-script\"
"

# ลบ scheduler หลังรัน (self-deleting)
:local selfDeleteScript "
# ทำงาน
:put \"One-time task running\"
/system script run actual-task

# ลบตัวเอง
/system scheduler remove [find name=\"one-time-task\"]
"

/system scheduler add \
    name="one-time-task" \
    start-date=[/system clock get date] \
    start-time=([/system clock get time] + 60) \
    interval=0 \
    on-event=$selfDeleteScript
```

---

## 24.5 Repeating Tasks

```routeros
# Heartbeat ทุก 1 นาที
/system scheduler add \
    name="heartbeat" \
    interval=1m \
    on-event="
        :local uptime [/system resource get uptime]
        :local cpu [/system resource get cpu-load]
        /log info message=(\"Heartbeat: CPU=\" . $cpu . \"% uptime=\" . $uptime)
    " \
    comment="System heartbeat every minute"

# Check interfaces ทุก 5 นาที
/system scheduler add \
    name="interface-check" \
    interval=5m \
    on-event="/system script run monitor-interfaces" \
    comment="Interface status check"

# Bandwidth stats ทุก 1 ชั่วโมง
/system scheduler add \
    name="hourly-stats" \
    interval=1h \
    on-event="/system script run collect-bandwidth-stats" \
    comment="Hourly bandwidth collection"

# Daily backup ตี 2
/system scheduler add \
    name="daily-backup" \
    start-time=02:00:00 \
    interval=1d \
    on-event="/system script run daily-backup" \
    comment="Daily system backup at 2AM"

# Weekly full report ทุกวันจันทร์
/system scheduler add \
    name="weekly-report" \
    start-time=08:00:00 \
    interval=7d \
    on-event="/system script run weekly-report" \
    comment="Weekly report every Monday 8AM"
```

---

## 24.6 Managing Schedules

### Enable/Disable Schedulers

```routeros
# Disable scheduler
/system scheduler disable [find name="daily-backup"]

# Enable scheduler
/system scheduler enable [find name="daily-backup"]

# Disable ทั้งหมด
/system scheduler disable [find disabled=no]

# Enable ทั้งหมด
/system scheduler enable [find disabled=yes]

# Toggle scheduler
:local toggleScheduler do={
    :local name $1
    :local id [/system scheduler find name=$name]
    :if ([:len $id] = 0) do={
        :put "Scheduler not found: $name"
        :return
    }
    
    :local status [/system scheduler get $id disabled]
    :if ($status) do={
        /system scheduler enable $id
        :put "$name enabled"
    } else={
        /system scheduler disable $id
        :put "$name disabled"
    }
}

[$toggleScheduler "daily-backup"]
```

### แก้ไข Scheduler

```routeros
# เปลี่ยนเวลารัน
/system scheduler set [find name="daily-backup"] start-time=03:00:00

# เปลี่ยน interval
/system scheduler set [find name="heartbeat"] interval=2m

# เปลี่ยน script ที่รัน
/system scheduler set [find name="daily-backup"] \
    on-event="/system script run new-backup-script"

# Update comment
/system scheduler set [find name="daily-backup"] \
    comment="Updated: Daily backup at 3AM"
```

### ลบ Scheduler

```routeros
# ลบ scheduler เดี่ยว
/system scheduler remove [find name="daily-backup"]

# ลบ scheduler ที่ disabled
/system scheduler remove [find disabled=yes]

# ลบ scheduler ที่ชื่อขึ้นต้นด้วย "temp-"
:foreach s in=[/system scheduler find] do={
    :local name [/system scheduler get $s name]
    :if ([:find $name "temp-"] = 0) do={
        /system scheduler remove $s
        :put "Removed: $name"
    }
}
```

---

## 24.7 Scheduler Scripts

### Inline Script

```routeros
# Script สั้นๆ ใส่ใน on-event ได้เลย
/system scheduler add \
    name="simple-log" \
    interval=1h \
    on-event=":log info \"Hourly check OK\""

# Multi-line inline script
/system scheduler add \
    name="complex-inline" \
    interval=5m \
    on-event="
        :local cpu [/system resource get cpu-load]
        :local mem [/system resource get free-memory]
        :if ($cpu > 80) do={
            :log warning message=(\"High CPU: \" . $cpu . \"%\")
        }
        :if ($mem < 10000000) do={
            :log warning message=\"Low memory!\"
        }
    " \
    comment="System health check every 5 min"
```

### External Script Reference

```routeros
# สร้าง script แยกก่อน
/system script add \
    name="check-health" \
    source="
        :local cpu [/system resource get cpu-load]
        :local mem [/system resource get free-memory]
        :local disk [/system resource get total-hdd-space]
        
        :put \"=== System Health ===\"
        :put \"CPU: $cpu%\"
        :put \"Free Memory: $mem bytes\"
        :put \"Disk: $disk bytes\"
        
        :if ($cpu > 90) do={
            :log critical message=(\"CRITICAL CPU: \" . $cpu . \"%\")
        }
    " \
    comment="System health check script"

# แล้ว scheduler อ้างอิง script
/system scheduler add \
    name="health-check" \
    interval=5m \
    on-event="/system script run check-health"
```

---

## 24.8 Common Automation Patterns

### Pattern 1: Conditional Scheduler

```routeros
# Script ที่ทำงานเฉพาะเงื่อนไขบางอย่าง
/system script add name="conditional-task" source="
    # ตรวจสอบว่ามี interface ที่ down อยู่หรือไม่
    :local downIfaces [/interface find running=no disabled=no]
    :if ([:len $downIfaces] > 0) do={
        # มี interface down - ส่ง alert
        :foreach iface in=$downIfaces do={
            :local name [/interface get $iface name]
            :log error message=(\"Interface DOWN: \" . $name)
        }
    }
"

/system scheduler add \
    name="interface-monitor" \
    interval=1m \
    on-event="/system script run conditional-task"
```

### Pattern 2: Time Window Script

```routeros
# รัน script เฉพาะช่วงเวลา business hours
/system script add name="business-hours-task" source="
    :local time [/system clock get time]
    :local hour [:toint [:pick [:tostr $time] 0 2]]
    
    # ทำงานเฉพาะ 09:00-18:00
    :if ($hour >= 9 && $hour < 18) do={
        # Business hours tasks
        /system script run monitoring-script
    }
"

/system scheduler add \
    name="business-task" \
    interval=30m \
    on-event="/system script run business-hours-task"
```

### Pattern 3: Adaptive Scheduler

```routeros
# ปรับ interval ตาม load
/system script add name="adaptive-check" source="
    :global checkInterval
    :local cpu [/system resource get cpu-load]
    
    # ถ้า CPU สูง เพิ่ม interval เพื่อลด overhead
    :if ($cpu > 70) do={
        :set checkInterval 5m
    } else={
        :set checkInterval 1m
    }
    
    # ทำงานหลัก
    /system script run main-monitoring
    
    # Update scheduler interval
    /system scheduler set [find name=\"adaptive-check-runner\"] \
        interval=$checkInterval
"
```

### Pattern 4: Retry Mechanism

```routeros
# Script ที่ retry ถ้า fail
/system script add name="retry-task" source="
    :global retryCount
    :local maxRetries 3
    :local success false
    
    :if ([:typeof $retryCount] = \"nothing\") do={
        :set retryCount 0
    }
    
    :do {
        # Try main task
        /tool fetch url=\"http://192.168.1.100/api/update\" \
            dst-path=\"update-result.txt\"
        :set success true
        :set retryCount 0
        :log info \"Task completed successfully\"
    } on-error={
        :set retryCount ($retryCount + 1)
        :log warning message=(\"Task failed, attempt \" . $retryCount . \"/\" . $maxRetries)
        
        :if ($retryCount >= $maxRetries) do={
            :log error \"Max retries reached - sending alert\"
            :set retryCount 0
            # ส่ง alert
        }
    }
"
```

---

## 24.9 Troubleshooting Schedules

### Debug Scheduled Tasks

```routeros
# ดูว่า scheduler รันล่าสุดเมื่อไหร่
/system scheduler print detail where name="daily-backup"

# เปิด logging สำหรับ scheduler
/system logging add topics=scheduler action=memory

# ดู log ของ scheduler
/log print where topics~"scheduler"

# Test run script ด้วยตัวเอง
/system script run daily-backup

# ดู error ใน script
/system script print where name="daily-backup"
```

### Common Issues และ Solutions

```routeros
# Issue: Script ไม่รัน
# Solution: ตรวจสอบ disabled status
/system scheduler print where disabled=no

# Issue: Script รันแต่ error
# Solution: ดู log
/log print where message~"daily-backup"

# Issue: Scheduler หายหลัง reboot
# Solution: ตรวจสอบว่า scheduler อยู่ใน config
/system scheduler export

# Issue: Script ใช้เวลานาน block scheduler อื่น
# Solution: Script ต้องรันใน background หรือ แยก scheduler
# (RouterOS scripts รัน sequential ไม่ parallel)

# Verify scheduler is running
:local checkSchedulerHealth do={
    :foreach s in=[/system scheduler find disabled=no] do={
        :local name [/system scheduler get $s name]
        :local nextRun [/system scheduler get $s next-run]
        :put "Scheduler: $name, Next run: $nextRun"
    }
}

[$checkSchedulerHealth]
```

---

## Lab 24: Daily Backup Scheduler

### โจทย์
สร้างระบบ backup อัตโนมัติที่:
1. Backup ทุกวันตี 2
2. Weekly backup ทุกวันอาทิตย์
3. Monthly backup วันที่ 1 ของเดือน
4. ส่ง email แจ้งผล
5. ลบ backups เก่า

### Solution

```routeros
# Daily Backup Scheduler Lab
# ==========================

# --- Setup Scripts ---

# 1. Daily backup script
/system script add name="daily-backup-task" source="
    :local type \"daily\"
    :local hostname [/system identity get name]
    :local date [/system clock get date]
    :local time [/system clock get time]
    
    # สร้าง filename
    :local dateStr [:tostr \$date]
    :local cleanDate \"\"
    :for i from=0 to=([:len \$dateStr] - 1) do={
        :local c [:pick \$dateStr \$i (\$i + 1)]
        :if (\$c = \"/\") do={ :set cleanDate (\$cleanDate . \"-\") } else={ :set cleanDate (\$cleanDate . \$c) }
    }
    :local filename (\$hostname . \"-\" . \$type . \"-\" . \$cleanDate)
    
    # Export config
    :do {
        /system backup save name=\$filename
        /export file=\$filename
        :delay 3s
        :log info message=(\"Backup created: \" . \$filename)
    } on-error={
        :log error message=(\"Backup FAILED: \" . \$filename)
    }
    
    # Cleanup (keep 7 daily backups)
    :local backups [/file find where name~(\$hostname . \"-daily-\") and name~\".backup\"]
    :local count [:len \$backups]
    :if (\$count > 7) do={
        :local toDelete [:pick \$backups 0 (\$count - 7)]
        :foreach f in=\$toDelete do={
            /file remove \$f
        }
    }
" comment="Daily backup task"

# 2. Weekly backup script
/system script add name="weekly-backup-task" source="
    :local type \"weekly\"
    :local hostname [/system identity get name]
    :local date [/system clock get date]
    :local dateStr [:tostr \$date]
    :local cleanDate \"\"
    :for i from=0 to=([:len \$dateStr] - 1) do={
        :local c [:pick \$dateStr \$i (\$i + 1)]
        :if (\$c = \"/\") do={ :set cleanDate (\$cleanDate . \"-\") } else={ :set cleanDate (\$cleanDate . \$c) }
    }
    :local filename (\$hostname . \"-\" . \$type . \"-\" . \$cleanDate)
    
    /system backup save name=\$filename
    /export file=\$filename
    :delay 3s
    
    # Upload to FTP
    :do {
        /tool fetch address=\"192.168.1.100\" src-path=(\$filename . \".backup\") \
            user=\"backup\" password=\"backup123\" \
            dst-path=(\"/weekly/\" . \$filename . \".backup\") upload=yes
        :log info message=(\"Weekly backup uploaded: \" . \$filename)
    } on-error={
        :log error message=\"Weekly FTP upload failed\"
    }
    
    # Cleanup (keep 4 weekly backups)
    :local backups [/file find where name~(\$hostname . \"-weekly-\") and name~\".backup\"]
    :local count [:len \$backups]
    :if (\$count > 4) do={
        :local toDelete [:pick \$backups 0 (\$count - 4)]
        :foreach f in=\$toDelete do={
            /file remove \$f
        }
    }
" comment="Weekly backup task"

# 3. Monthly backup script
/system script add name="monthly-backup-task" source="
    # รัน script นี้ทุกวัน แต่ทำงานจริงแค่วันที่ 1
    :local date [/system clock get date]
    :local dateStr [:tostr \$date]
    :local dayStr [:pick \$dateStr 4 6]
    
    :if (\$dayStr != \"01\") do={ :return }
    
    :local type \"monthly\"
    :local hostname [/system identity get name]
    :local filename (\$hostname . \"-\" . \$type . \"-\" . \$dateStr)
    
    /system backup save name=\$filename
    /export file=\$filename
    :delay 3s
    
    # Upload to FTP
    /tool fetch address=\"192.168.1.100\" src-path=(\$filename . \".backup\") \
        user=\"backup\" password=\"backup123\" \
        dst-path=(\"/monthly/\" . \$filename . \".backup\") upload=yes
    
    :log info message=(\"Monthly backup created: \" . \$filename)
" comment="Monthly backup task (runs daily, acts on day 1)"

# --- Setup Schedulers ---

# Daily backup - ทุกวันตี 2
/system scheduler add \
    name="daily-backup" \
    start-time=02:00:00 \
    interval=1d \
    on-event="/system script run daily-backup-task" \
    comment="Daily backup at 2AM"

# Weekly backup - ทุกวันอาทิตย์เที่ยงคืน
# interval=7d จะทำให้รันทุก 7 วัน
/system scheduler add \
    name="weekly-backup" \
    start-date=jan/05/2025 \
    start-time=00:30:00 \
    interval=7d \
    on-event="/system script run weekly-backup-task" \
    comment="Weekly backup on Sundays"

# Monthly backup check - ทุกวันตี 1
/system scheduler add \
    name="monthly-backup" \
    start-time=01:00:00 \
    interval=1d \
    on-event="/system script run monthly-backup-task" \
    comment="Monthly backup on 1st of month"

# Verify schedulers
/system scheduler print

:put "Backup schedulers configured successfully!"
:put ""
:put "Summary:"
:put "- Daily backup: 2:00 AM every day"
:put "- Weekly backup: 12:30 AM every Sunday"
:put "- Monthly backup: 1:00 AM on 1st of each month"
```

---

## Summary ของ Part 24

| ประเภท | interval | ตัวอย่าง |
|--------|---------|---------|
| Seconds | Xs | 30s, 60s |
| Minutes | Xm | 5m, 30m |
| Hours | Xh | 1h, 6h |
| Days | Xd | 1d, 7d |
| One-time | 0 | รันครั้งเดียว |
| HH:MM:SS | HH:MM:SS | 00:05:00 |

> **Best Practice:** แยก logic ไว้ใน scripts และให้ scheduler เรียกใช้ ไม่ควรใส่ logic ทั้งหมดใน on-event

> **Warning:** Scripts หลายตัวที่รันพร้อมกันอาจทำให้ router ช้าหรือ resource ขาด

> **Tip:** ใช้ comment อธิบาย scheduler เสมอเพื่อให้ maintain ได้ง่าย

---

[← Part 23: File Operations](part-023-file-operations.md) | [Part 25: Global Variables →](part-025-global-variables.md)
