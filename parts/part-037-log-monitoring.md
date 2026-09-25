# Part 37: Log Monitoring ใน RouterOS

## บทนำ

RouterOS logging system บันทึกทุก event ที่เกิดขึ้น การอ่านและวิเคราะห์ logs ช่วยให้ตรวจพบปัญหาและความผิดปกติได้รวดเร็ว

---

## 37.1 RouterOS Logging Basics

```routeros
# ดู log topics ที่มี
/system logging topic print

# Log topics หลัก:
# info, warning, error, critical
# firewall, dhcp, pppoe, hotspot
# script, scheduler, system
# interface, radius, route

# เพิ่ม logging rule
/system logging add topics=firewall action=memory
/system logging add topics=dhcp action=memory
/system logging add topics=script action=memory

# ดู log ปัจจุบัน
/log print

# Filter log by topic
/log print where topics~"firewall"

# Filter log by message
/log print where message~"192.168.1"

# ดู last 20 entries
/log print count=20
```

---

## 37.2 Log Actions

```routeros
# Memory log (default)
/system logging action set memory memory-lines=1000

# Disk log
/system logging action add \
    name=disk-log \
    target=disk \
    disk-file-name=log/system \
    disk-lines-per-file=1000 \
    disk-file-count=5

# Remote syslog
/system logging action add \
    name=syslog-remote \
    target=remote \
    remote=192.168.1.200 \
    remote-port=514 \
    syslog-facility=local0 \
    syslog-severity=auto

# Email log (critical only)
/system logging action add \
    name=email-alerts \
    target=email \
    email-to=admin@company.com

# Assign actions to topics
/system logging add topics=firewall action=disk-log
/system logging add topics=error action=email-alerts
/system logging add topics=critical action=email-alerts,disk-log
/system logging add topics=dhcp action=syslog-remote
```

---

## 37.3 Log Parsing Scripts

```routeros
# Parse log สำหรับ failed logins
/system script add name="parse-failed-logins" source="
    :local failedLogins 0
    :local report \"\"
    
    :foreach entry in=[/log find topics~\"system\" message~\"login failure\"] do={
        :local msg [/log get \$entry message]
        :local time [/log get \$entry time]
        :set failedLogins (\$failedLogins + 1)
        :set report (\$report . \$time . \": \" . \$msg . \"\\r\\n\")
    }
    
    :if (\$failedLogins > 0) do={
        :put (\"Found \" . \$failedLogins . \" failed logins\")
        :put \$report
        
        :if (\$failedLogins > 5) do={
            /tool e-mail send to=\"admin@company.com\" \
                subject=(\"Alert: \" . \$failedLogins . \" failed logins\") \
                body=\$report
        }
    }
" comment="Parse failed login attempts"

# Parse log สำหรับ errors
/system script add name="parse-errors" source="
    :local errors 0
    
    :foreach entry in=[/log find topics~\"error\"] do={
        :local msg [/log get \$entry message]
        :local time [/log get \$entry time]
        :put (\$time . \": \" . \$msg)
        :set errors (\$errors + 1)
    }
    
    :put (\"Total errors found: \" . \$errors)
" comment="Parse error logs"

# Search log for specific IP
:local searchIP "192.168.1.100"
:foreach entry in=[/log find message~$searchIP] do={
    :local msg [/log get $entry message]
    :local time [/log get $entry time]
    :put "$time: $msg"
}
```

---

## 37.4 Syslog Configuration

```routeros
# ส่ง logs ไปยัง syslog server (rsyslog/syslog-ng)
/system logging action set [find name=remote] \
    remote=10.0.0.5 \
    remote-port=514 \
    syslog-facility=local0

# ส่ง topics ที่ต้องการ
/system logging add topics=firewall,dhcp,pppoe action=remote

# Structured logging (RouterOS 7.x)
/system logging action set [find name=remote] \
    bsd-syslog=yes

# ทดสอบ syslog
:log info "Test syslog message from RouterOS"

# Script ส่ง custom syslog message
:local sendSyslog do={
    :local severity $1
    :local message $2
    
    :if ($severity = "info") do={
        :log info $message
    }
    :if ($severity = "warning") do={
        :log warning $message
    }
    :if ($severity = "error") do={
        :log error $message
    }
}

[$sendSyslog "info" "Custom monitoring message"]
[$sendSyslog "warning" "High CPU detected"]
```

---

## 37.5 Alert on Log Events

```routeros
# Watch for critical events and alert
/system script add name="log-alert-monitor" source="
    :global lastLogCheck
    :local currentTime [/system clock get time]
    
    # ตรวจสอบ log entries ใหม่
    :local criticalPatterns {\"critical\"; \"failed\"; \"dropped\"; \"attack\"}
    
    :foreach entry in=[/log find] do={
        :local msg [/log get \$entry message]
        :local topics [/log get \$entry topics]
        :local time [/log get \$entry time]
        
        # ตรวจสอบ patterns
        :if (\$topics~\"critical\" || \$topics~\"error\") do={
            :log warning (\"Alert pattern detected: \" . \$msg)
        }
    }
" comment="Monitor logs for critical events"

# Trigger alerts based on patterns
/system script add name="security-log-watch" source="
    :local threshold 10
    :local violations 0
    
    # หา port scan attempts
    :foreach entry in=[/log find topics~\"firewall\" message~\"input\"] do={
        :set violations (\$violations + 1)
    }
    
    :if (\$violations > \$threshold) do={
        :log warning (\"Possible attack: \" . \$violations . \" firewall events\")
        /tool e-mail send to=\"security@company.com\" \
            subject=\"Security Alert: Firewall Activity\" \
            body=(\"Detected \" . \$violations . \" firewall events in last cycle\")
    }
" comment="Security log watcher"
```

---

## 37.6 Log Rotation

```routeros
# RouterOS จัดการ log rotation อัตโนมัติ แต่เราสามารถ script เพิ่มได้

# Export logs to file then rotate
/system script add name="log-rotate" source="
    # Export current log
    :local timestamp [/system clock get date]
    :local filename (\"log-\" . \$timestamp . \".txt\")
    :local logContent \"\"
    
    :foreach entry in=[/log find] do={
        :local time [/log get \$entry time]
        :local msg [/log get \$entry message]
        :local topics [/log get \$entry topics]
        :set logContent (\$logContent . \$time . \" [\" . \$topics . \"] \" . \$msg . \"\\r\\n\")
    }
    
    # Save to file
    :do {
        /tool fetch url=\"data:,\" dst-path=\$filename
        /file set [find name=\$filename] contents=\$logContent
        :put (\"Log exported: \" . \$filename)
    } on-error={
        :put \"Error saving log file\"
    }
    
    # Clean old log files (keep 7 days)
    :foreach f in=[/file find type=file] do={
        :local fname [/file get \$f name]
        :if (\$fname~\"log-\") do={
            :local age [/file get \$f creation-time]
            # Note: RouterOS does not have direct date comparison
            # Use file count to limit
        }
    }
    
    # Keep only last 7 log files
    :local logFiles [/file find name~\"log-\" type=file]
    :if ([:len \$logFiles] > 7) do={
        /file remove [:pick \$logFiles 0 1]
    }
    
    :put \"Log rotation complete\"
" comment="Log rotation script"

/system scheduler add \
    name="log-rotate" \
    start-time=00:00:00 \
    interval=1d \
    on-event="/system script run log-rotate"
```

---

## Lab 37: Syslog + Alert System

### Solution

```routeros
# Complete Syslog + Alert System
# ================================

# 1. Configure remote syslog action
/system logging action add \
    name=syslog-server \
    target=remote \
    remote=10.0.0.100 \
    remote-port=514 \
    syslog-facility=local0

# 2. Add logging rules
/system logging add topics=firewall action=syslog-server
/system logging add topics=dhcp action=syslog-server
/system logging add topics=system action=syslog-server

# 3. Alert script for critical events
/system script add name="log-alert" source="
    :global lastAlertTime
    :local alertCooldown 300  # 5 minutes
    :local now [/system clock get time]
    :local criticalFound false
    :local alertMsg \"\"
    
    :foreach entry in=[/log find topics~\"critical\"] do={
        :local msg [/log get \$entry message]
        :local time [/log get \$entry time]
        :set criticalFound true
        :set alertMsg (\$alertMsg . \$time . \": \" . \$msg . \"\\r\\n\")
    }
    
    :if (\$criticalFound) do={
        /tool e-mail send to=\"admin@company.com\" \
            subject=\"Critical Alert from RouterOS\" \
            body=\$alertMsg
        :log info \"Alert email sent\"
    }
" comment="Critical log alerter"

# 4. Schedule alert check every 5 minutes
/system scheduler add \
    name="log-alert-check" \
    interval=5m \
    on-event="/system script run log-alert"

:put "Syslog + Alert system configured"
:put "Remote syslog: 10.0.0.100:514"
:put "Alert check interval: 5 minutes"
```

---

## Summary ของ Part 37

| Log Action | ปลายทาง | ใช้เมื่อ |
|-----------|--------|---------|
| memory | RAM | Debug/ทั่วไป |
| disk | Flash storage | Long-term |
| remote | Syslog server | Centralized |
| email | Email | Critical alerts |

> **Tip:** ใช้ remote syslog กับ Graylog หรือ ELK Stack สำหรับ log analysis

> **Warning:** Log to disk มาก จะทำให้ flash wear ควรใช้ remote log แทน

> **Note:** `/log print` แสดงเฉพาะ memory logs ถ้าต้องการ persistent ต้องใช้ disk หรือ remote

---

[← Part 36: User Manager](part-036-user-manager.md) | [Part 38: Network Monitoring →](part-038-network-monitoring.md)
