# Part 31: Email Notifications ใน RouterOS

## บทนำ

RouterOS สามารถส่ง email notifications ได้โดยตรง ช่วยให้ admin ได้รับ alerts และรายงานสำคัญโดยอัตโนมัติ ไม่ว่าจะเป็น interface down, CPU สูง, หรือ daily reports

---

## 31.1 SMTP Configuration

### ตั้งค่า SMTP Server

```routeros
# ตั้งค่า SMTP สำหรับ Gmail
/tool e-mail set \
    server=smtp.gmail.com \
    port=587 \
    tls=starttls \
    from=router@gmail.com \
    user=router@gmail.com \
    password=YourAppPassword

# ตั้งค่า SMTP สำหรับ Microsoft Exchange
/tool e-mail set \
    server=mail.company.com \
    port=25 \
    tls=no \
    from=router@company.com \
    user=router@company.com \
    password=RouterPass

# ตั้งค่า SMTP แบบ TLS (465)
/tool e-mail set \
    server=mail.company.com \
    port=465 \
    tls=tls \
    from=router@company.com

# ดู SMTP configuration
/tool e-mail print

# Test SMTP configuration
/tool e-mail send \
    to=admin@company.com \
    subject="Test from RouterOS" \
    body="This is a test email from RouterOS"
```

### SMTP Settings Table

| Setting | คำอธิบาย | ตัวอย่าง |
|---------|---------|---------|
| server | SMTP server hostname/IP | smtp.gmail.com |
| port | SMTP port | 25, 465, 587 |
| tls | TLS mode | no, tls, starttls |
| from | From address | router@company.com |
| user | Username | router@company.com |
| password | Password/App password | ... |

---

## 31.2 Sending Emails via Script

### Basic Email Send

```routeros
# ส่ง email พื้นฐาน
/tool e-mail send \
    to="admin@company.com" \
    subject="Router Alert" \
    body="Something happened on your router"

# ส่งพร้อม CC และ BCC
/tool e-mail send \
    to="admin@company.com" \
    cc="manager@company.com" \
    subject="Router Alert" \
    body="Alert message"

# ส่งหลาย recipients
/tool e-mail send \
    to="admin1@company.com,admin2@company.com" \
    subject="Multiple recipients test" \
    body="This goes to multiple people"

# ส่งด้วย script function
:local sendEmail do={
    :local to $1
    :local subject $2
    :local body $3
    
    :do {
        /tool e-mail send to=$to subject=$subject body=$body
        :log info ("Email sent to: " . $to . " Subject: " . $subject)
        :return true
    } on-error={
        :log error ("Failed to send email to: " . $to)
        :return false
    }
}

[$sendEmail "admin@company.com" "Test Alert" "This is a test"]
```

---

## 31.3 HTML Emails

```routeros
# RouterOS รองรับ HTML emails
:local sendHTMLEmail do={
    :local to $1
    :local subject $2
    :local htmlBody $3
    
    # RouterOS อาจต้องการ content-type header
    :do {
        /tool e-mail send \
            to=$to \
            subject=$subject \
            body=$htmlBody
        :return true
    } on-error={
        :return false
    }
}

# สร้าง HTML email body
:local router [/system identity get name]
:local cpu [/system resource get cpu-load]
:local mem [/system resource get free-memory]
:local uptime [/system resource get uptime]

:local htmlBody "<html><body>"
:set htmlBody ($htmlBody . "<h2>Router Status Report</h2>")
:set htmlBody ($htmlBody . "<table border='1'>")
:set htmlBody ($htmlBody . "<tr><th>Metric</th><th>Value</th></tr>")
:set htmlBody ($htmlBody . "<tr><td>Router</td><td>" . $router . "</td></tr>")
:set htmlBody ($htmlBody . "<tr><td>CPU Load</td><td>" . $cpu . "%</td></tr>")
:set htmlBody ($htmlBody . "<tr><td>Free Memory</td><td>" . $mem . " bytes</td></tr>")
:set htmlBody ($htmlBody . "<tr><td>Uptime</td><td>" . $uptime . "</td></tr>")
:set htmlBody ($htmlBody . "</table>")
:set htmlBody ($htmlBody . "</body></html>")

[$sendHTMLEmail "admin@company.com" "Router Status" $htmlBody]
```

---

## 31.4 Email Templates

```routeros
# Template system สำหรับ emails
:local emailTemplates {}

# Alert template
:local alertTemplate "=== ROUTER ALERT ===\r\n\r\nRouter: {ROUTER}\r\nTime: {TIME}\r\nSeverity: {SEVERITY}\r\n\r\nAlert: {MESSAGE}\r\n\r\nDetails:\r\n{DETAILS}\r\n\r\n---\r\nThis is an automated message."

# Daily report template
:local reportTemplate "=== DAILY REPORT ===\r\n\r\nRouter: {ROUTER}\r\nDate: {DATE}\r\n\r\nSystem Status:\r\n  CPU: {CPU}%\r\n  Memory: {MEMORY}\r\n  Uptime: {UPTIME}\r\n\r\n{EXTRA}\r\n---\r\nGenerated automatically."

# Fill template function
:local fillTemplate do={
    :local template $1
    :local vars $2
    
    :local result $template
    :foreach var in=$vars do={
        :local key ("{" . ($var->0) . "}")
        :local val ($var->1)
        
        :while ([:find $result $key] != -1) do={
            :local pos [:find $result $key]
            :set result ([:pick $result 0 $pos] . $val . \
                       [:pick $result ($pos + [:len $key]) [:len $result]])
        }
    }
    :return $result
}

# ใช้งาน template
:local router [/system identity get name]
:local alertBody [$fillTemplate $alertTemplate {
    {"ROUTER"; $router};
    {"TIME"; [/system clock get time]};
    {"SEVERITY"; "HIGH"};
    {"MESSAGE"; "Interface ether1 is DOWN"};
    {"DETAILS"; "Interface was up until 14:30, now showing as down"}
}]

/tool e-mail send \
    to="admin@company.com" \
    subject="[HIGH] Interface Down Alert" \
    body=$alertBody
```

---

## 31.5 Alerts: CPU, Memory, Interface Down

```routeros
# CPU Alert
/system script add name="alert-high-cpu" source="
    :local threshold 85
    :local cpu [/system resource get cpu-load]
    :local router [/system identity get name]
    
    :if (\$cpu > \$threshold) do={
        :local subject (\"[CPU ALERT] \" . \$router . \": CPU at \" . \$cpu . \"%\")
        :local body (\"High CPU Alert\\r\\n\" . \
                    \"Router: \" . \$router . \"\\r\\n\" . \
                    \"CPU Load: \" . \$cpu . \"%\" . \"\\r\\n\" . \
                    \"Threshold: \" . \$threshold . \"%\\r\\n\" . \
                    \"Time: \" . [/system clock get time] . \"\\r\\n\\r\\n\" . \
                    \"Top processes affecting CPU may include:\\r\\n\" . \
                    \"Please investigate immediately.\")
        
        /tool e-mail send to=\"admin@company.com\" subject=\$subject body=\$body
        :log critical (\"High CPU alert: \" . \$cpu . \"%\")
    }
" comment="CPU alert script"

# Memory Alert  
/system script add name="alert-low-memory" source="
    :local threshold 10000000  # 10MB
    :local freeMem [/system resource get free-memory]
    :local totalMem [/system resource get total-memory]
    :local router [/system identity get name]
    
    :if (\$freeMem < \$threshold) do={
        :local usedPct (((\$totalMem - \$freeMem) * 100) / \$totalMem)
        :local subject (\"[MEMORY ALERT] \" . \$router . \": Low memory\")
        :local body (\"Low Memory Alert\\r\\n\" . \
                    \"Router: \" . \$router . \"\\r\\n\" . \
                    \"Free Memory: \" . \$freeMem . \" bytes\\r\\n\" . \
                    \"Total Memory: \" . \$totalMem . \" bytes\\r\\n\" . \
                    \"Used: \" . \$usedPct . \"%\\r\\n\" . \
                    \"Time: \" . [/system clock get time])
        
        /tool e-mail send to=\"admin@company.com\" subject=\$subject body=\$body
        :log critical (\"Low memory alert: \" . \$freeMem . \" bytes free\")
    }
" comment="Memory alert script"

# Interface Down Alert
/system script add name="alert-interface-down" source="
    :global LAST_IFACE_STATE
    :local router [/system identity get name]
    
    :foreach iface in=[/interface find disabled=no] do={
        :local name [/interface get \$iface name]
        :local running [/interface get \$iface running]
        :local key (\"iface_\" . \$name)
        
        # ตรวจสอบ previous state
        :local prevRunning true
        :if ([:typeof (\$LAST_IFACE_STATE->\$key)] != \"nothing\") do={
            :set prevRunning (\$LAST_IFACE_STATE->\$key)
        }
        
        # Interface just went DOWN
        :if (\$prevRunning && !\$running) do={
            :local subject (\"[INTERFACE DOWN] \" . \$router . \": \" . \$name)
            :local body (\"Interface DOWN Alert\\r\\n\" . \
                        \"Router: \" . \$router . \"\\r\\n\" . \
                        \"Interface: \" . \$name . \"\\r\\n\" . \
                        \"Status: DOWN\\r\\n\" . \
                        \"Time: \" . [/system clock get time])
            
            /tool e-mail send to=\"admin@company.com\" subject=\$subject body=\$body
            :log error (\"Interface DOWN alert sent: \" . \$name)
        }
        
        # Interface just came UP
        :if (!\$prevRunning && \$running) do={
            :local subject (\"[INTERFACE UP] \" . \$router . \": \" . \$name)
            :local body (\"Interface UP (recovered)\\r\\n\" . \
                        \"Router: \" . \$router . \"\\r\\n\" . \
                        \"Interface: \" . \$name . \"\\r\\n\" . \
                        \"Status: UP\\r\\n\" . \
                        \"Time: \" . [/system clock get time])
            
            /tool e-mail send to=\"admin@company.com\" subject=\$subject body=\$body
            :log info (\"Interface UP alert sent: \" . \$name)
        }
        
        # Update state
        :set (\$LAST_IFACE_STATE->\$key) \$running
    }
" comment="Interface down/up alert"

# Setup monitoring schedulers
/system scheduler add name="cpu-monitor" interval=5m \
    on-event="/system script run alert-high-cpu"
/system scheduler add name="memory-monitor" interval=5m \
    on-event="/system script run alert-low-memory"
/system scheduler add name="iface-monitor" interval=1m \
    on-event="/system script run alert-interface-down"
```

---

## 31.6 Daily Reports via Email

```routeros
/system script add name="daily-email-report" source="
    :local router [/system identity get name]
    :local date [/system clock get date]
    :local cpu [/system resource get cpu-load]
    :local freeMem [/system resource get free-memory]
    :local totalMem [/system resource get total-memory]
    :local uptime [/system resource get uptime]
    :local usedMem ((\$totalMem - \$freeMem) * 100 / \$totalMem)
    
    :local body \"\"
    :set body (\$body . \"Daily Status Report\\r\\n\")
    :set body (\$body . \"================================\\r\\n\")
    :set body (\$body . \"Router: \" . \$router . \"\\r\\n\")
    :set body (\$body . \"Date: \" . \$date . \"\\r\\n\\r\\n\")
    
    :set body (\$body . \"System Resources:\\r\\n\")
    :set body (\$body . \"  CPU Load: \" . \$cpu . \"%\\r\\n\")
    :set body (\$body . \"  Memory Used: \" . \$usedMem . \"%\\r\\n\")
    :set body (\$body . \"  Uptime: \" . \$uptime . \"\\r\\n\\r\\n\")
    
    :set body (\$body . \"Interfaces:\\r\\n\")
    :foreach iface in=[/interface find disabled=no] do={
        :local name [/interface get \$iface name]
        :local running [/interface get \$iface running]
        :local status \"DOWN\"
        :if (\$running) do={ :set status \"UP\" }
        :local rxBytes [/interface get \$iface rx-byte]
        :local txBytes [/interface get \$iface tx-byte]
        :set body (\$body . \"  \" . \$name . \": \" . \$status . \
                  \" RX=\" . \$rxBytes . \" TX=\" . \$txBytes . \"\\r\\n\")
    }
    
    :set body (\$body . \"\\r\\nDHCP Leases:\\r\\n\")
    :local activeLeases [:len [/ip dhcp-server lease find status=bound]]
    :set body (\$body . \"  Active leases: \" . \$activeLeases . \"\\r\\n\")
    
    :set body (\$body . \"\\r\\nFirewall:\\r\\n\")
    :local blocked [:len [/ip firewall address-list find list=blacklist]]
    :set body (\$body . \"  Blocked IPs: \" . \$blocked . \"\\r\\n\")
    
    :set body (\$body . \"\\r\\n--- End of report ---\\r\\n\")
    
    /tool e-mail send \
        to=\"admin@company.com\" \
        subject=(\"[Daily Report] \" . \$router . \" - \" . \$date) \
        body=\$body
    
    :log info \"Daily report sent\"
" comment="Daily status report via email"

/system scheduler add \
    name="daily-report" \
    start-time=08:00:00 \
    interval=1d \
    on-event="/system script run daily-email-report"
```

---

## 31.7 Attachment Support

```routeros
# RouterOS รองรับ file attachments ใน email
:local sendEmailWithAttachment do={
    :local to $1
    :local subject $2
    :local body $3
    :local attachFile $4
    
    :do {
        /tool e-mail send \
            to=$to \
            subject=$subject \
            body=$body \
            file=$attachFile
        :put "Email sent with attachment: $attachFile"
        :return true
    } on-error={
        :put "Failed to send email with attachment"
        :return false
    }
}

# ส่ง backup เป็น attachment
:local hostname [/system identity get name]
:local date [/system clock get date]
:local backupFile ($hostname . "-" . $date . ".rsc")

/export file=($hostname . "-" . $date)
:delay 2s

[$sendEmailWithAttachment \
    "admin@company.com" \
    "Config Backup - " . $hostname . " - " . $date \
    "Please find attached the daily configuration backup." \
    $backupFile]

# ส่ง log file เป็น attachment
:local sendLogEmail do={
    :local logContent ""
    
    # เก็บ log 100 entries ล่าสุด
    :foreach entry in=[/log find] do={
        :local msg [/log get $entry message]
        :local time [/log get $entry time]
        :set logContent ($logContent . $time . " " . $msg . "\r\n")
    }
    
    # บันทึกเป็นไฟล์
    /file remove [find name="daily-log.txt"]
    /tool fetch url="data:," dst-path="daily-log.txt"
    /file set [find name="daily-log.txt"] contents=$logContent
    
    /tool e-mail send \
        to="admin@company.com" \
        subject="Daily Log Export" \
        body="Please find attached the daily log export." \
        file="daily-log.txt"
}

[$sendLogEmail]
```

---

## 31.8 Multiple Recipients

```routeros
# ส่งไปหลายคน
:local sendToMultiple do={
    :local recipients $1  # array ของ email addresses
    :local subject $2
    :local body $3
    
    :local toList ""
    :foreach email in=$recipients do={
        :if ([:len $toList] > 0) do={
            :set toList ($toList . ",")
        }
        :set toList ($toList . $email)
    }
    
    :do {
        /tool e-mail send to=$toList subject=$subject body=$body
        :put "Email sent to: $toList"
    } on-error={
        :put "Failed to send email"
    }
}

:local adminList {"admin1@company.com"; "admin2@company.com"; "manager@company.com"}
[$sendToMultiple $adminList "System Alert" "Alert body here"]

# Escalation matrix
:local escalate do={
    :local severity $1
    :local subject $2
    :local body $3
    
    :local recipients {}
    
    :if ($severity = "info") do={
        :set recipients {"monitoring@company.com"}
    }
    :if ($severity = "warning") do={
        :set recipients {"admin@company.com"; "monitoring@company.com"}
    }
    :if ($severity = "critical") do={
        :set recipients {"admin@company.com"; "manager@company.com"; "oncall@company.com"}
    }
    
    [$sendToMultiple $recipients ("[" . $severity . "] " . $subject) $body]
}

[$escalate "critical" "Primary WAN Down" "WAN interface ether1 is down, failover activated"]
[$escalate "warning" "High CPU" "CPU usage at 85% for 10 minutes"]
[$escalate "info" "Backup Complete" "Daily backup completed successfully"]
```

---

## 31.9 Email Queuing

```routeros
# Email queue สำหรับ reliability
:global emailQueue {}
:global emailSendRetry 3

# เพิ่ม email เข้า queue
:global queueEmail do={
    :global emailQueue
    :local email {
        "to"=$1;
        "subject"=$2;
        "body"=$3;
        "retries"=0;
        "timestamp"=[/system clock get time]
    }
    :set emailQueue ($emailQueue , $email)
    :put "Email queued: " . $1
}

# Process email queue
:global processEmailQueue do={
    :global emailQueue
    :global emailSendRetry
    
    :if ([:len $emailQueue] = 0) do={
        :return
    }
    
    :local newQueue {}
    
    :foreach email in=$emailQueue do={
        :local retries ($email->"retries")
        
        :if ($retries < $emailSendRetry) do={
            :do {
                /tool e-mail send \
                    to=($email->"to") \
                    subject=($email->"subject") \
                    body=($email->"body")
                :log info ("Email sent from queue: " . ($email->"to"))
                # Success - don't add back to queue
            } on-error={
                # Failed - increment retries and keep in queue
                :local updated {
                    "to"=($email->"to");
                    "subject"=($email->"subject");
                    "body"=($email->"body");
                    "retries"=($retries + 1);
                    "timestamp"=($email->"timestamp")
                }
                :set newQueue ($newQueue , $updated)
                :log warning ("Email retry " . ($retries + 1) . ": " . ($email->"to"))
            }
        } else={
            :log error ("Email dropped after max retries: " . ($email->"to"))
        }
    }
    
    :set emailQueue $newQueue
}

# Queue email แทนส่งตรง
[$queueEmail "admin@company.com" "Alert" "Something happened"]
[$queueEmail "manager@company.com" "Report" "Daily report"]

# Process queue ทุก 5 นาที
/system scheduler add \
    name="process-email-queue" \
    interval=5m \
    on-event="[$processEmailQueue]"
```

---

## Lab 31: Complete Alert System

### โจทย์
สร้าง Complete Alert System ที่ monitor:
1. CPU > 80%
2. Memory < 20%
3. Interface down
4. DHCP pool nearly full
5. ส่ง digest report ทุกชั่วโมง
6. Daily summary

### Solution

```routeros
# Complete Alert System Lab
# ==========================

# --- Configuration ---
:global ALERT_TO "admin@company.com"
:global ALERT_CC ""
:global ALERT_CPU_THRESHOLD 80
:global ALERT_MEM_THRESHOLD_PCT 80  # alert when used > 80%
:global ALERT_DHCP_THRESHOLD 90     # alert when pool is 90% full
:global ALERT_COOLDOWN 1800         # 30 minutes between same alerts
:global ALERT_LAST_SENT {}

# --- Alert Throttle ---
:global canSendAlert do={
    :global ALERT_LAST_SENT
    :global ALERT_COOLDOWN
    :local alertKey $1
    :local now [/system resource get uptime]
    
    :local lastSent 0
    :foreach item in=$ALERT_LAST_SENT do={
        :if (($item->0) = $alertKey) do={
            :set lastSent ($item->1)
        }
    }
    
    :if (($now - $lastSent) > $ALERT_COOLDOWN) do={
        # Update last sent
        :local newList {}
        :foreach item in=$ALERT_LAST_SENT do={
            :if (($item->0) != $alertKey) do={
                :set newList ($newList , $item)
            }
        }
        :set newList ($newList , {$alertKey; $now})
        :set ALERT_LAST_SENT $newList
        :return true
    }
    :return false
}

# --- Send Alert Helper ---
:global sendAlert do={
    :global ALERT_TO
    :local subject $1
    :local body $2
    :local key $3
    :global canSendAlert
    
    :if ([$canSendAlert $key]) do={
        :do {
            /tool e-mail send to=$ALERT_TO subject=$subject body=$body
            :log info ("Alert sent: " . $subject)
        } on-error={
            :log error ("Failed to send alert: " . $subject)
        }
    }
}

# --- Monitor Script ---
/system script add name="complete-monitor" source="
    :global ALERT_CPU_THRESHOLD
    :global ALERT_MEM_THRESHOLD_PCT
    :global ALERT_DHCP_THRESHOLD
    :global sendAlert
    :global canSendAlert
    
    :local router [/system identity get name]
    :local time [/system clock get time]
    
    # 1. CPU Check
    :local cpu [/system resource get cpu-load]
    :if (\$cpu > \$ALERT_CPU_THRESHOLD) do={
        [\$sendAlert \
            (\"[CPU ALERT] \" . \$router . \": \" . \$cpu . \"%\") \
            (\"CPU Load Alert\\r\\nRouter: \" . \$router . \"\\r\\nCPU: \" . \$cpu . \"%\\r\\nThreshold: \" . \$ALERT_CPU_THRESHOLD . \"%\\r\\nTime: \" . \$time) \
            \"cpu-high\"]
    }
    
    # 2. Memory Check
    :local freeMem [/system resource get free-memory]
    :local totalMem [/system resource get total-memory]
    :local usedPct ((\$totalMem - \$freeMem) * 100 / \$totalMem)
    :if (\$usedPct > \$ALERT_MEM_THRESHOLD_PCT) do={
        [\$sendAlert \
            (\"[MEMORY ALERT] \" . \$router . \": \" . \$usedPct . \"% used\") \
            (\"Memory Alert\\r\\nRouter: \" . \$router . \"\\r\\nMemory Used: \" . \$usedPct . \"%\\r\\nFree: \" . \$freeMem . \" bytes\\r\\nTime: \" . \$time) \
            \"memory-low\"]
    }
    
    # 3. Interface Check
    :foreach iface in=[/interface find disabled=no] do={
        :local name [/interface get \$iface name]
        :local running [/interface get \$iface running]
        :if (!\$running) do={
            [\$sendAlert \
                (\"[INTERFACE DOWN] \" . \$router . \": \" . \$name) \
                (\"Interface Down Alert\\r\\nRouter: \" . \$router . \"\\r\\nInterface: \" . \$name . \"\\r\\nStatus: DOWN\\r\\nTime: \" . \$time) \
                (\"iface-down-\" . \$name)]
        }
    }
    
    # 4. DHCP Pool Check
    :foreach server in=[/ip dhcp-server find disabled=no] do={
        :local serverName [/ip dhcp-server get \$server name]
        :local poolName [/ip dhcp-server get \$server address-pool]
        :local pool [/ip pool find name=\$poolName]
        
        :if ([:len \$pool] > 0) do={
            :local activeLeases [:len [/ip dhcp-server lease find server=\$serverName status=bound]]
            :local ranges [/ip pool get \$pool ranges]
            # Calculate pool size (simplified - check active vs total)
            :if (\$activeLeases > 240) do={  # Assuming /24 = 254 hosts
                [\$sendAlert \
                    (\"[DHCP ALERT] \" . \$router . \": Pool nearly full\") \
                    (\"DHCP Pool Alert\\r\\nServer: \" . \$serverName . \"\\r\\nActive Leases: \" . \$activeLeases . \"\\r\\nTime: \" . \$time) \
                    (\"dhcp-full-\" . \$serverName)]
            }
        }
    }
" comment="Complete monitoring script"

# --- Hourly Digest ---
/system script add name="hourly-digest" source="
    :global ALERT_TO
    :local router [/system identity get name]
    :local time [/system clock get time]
    
    :local report \"=== Hourly Digest ===\"
    :set report (\$report . \"\\r\\nRouter: \" . \$router)
    :set report (\$report . \"\\r\\nTime: \" . \$time . \"\\r\\n\")
    
    # CPU/Memory
    :local cpu [/system resource get cpu-load]
    :local freeMem [/system resource get free-memory]
    :local totalMem [/system resource get total-memory]
    :local memPct ((\$totalMem - \$freeMem) * 100 / \$totalMem)
    
    :set report (\$report . \"\\r\\nResources:\\r\\n\")
    :set report (\$report . \"  CPU: \" . \$cpu . \"%\\r\\n\")
    :set report (\$report . \"  Memory Used: \" . \$memPct . \"%\\r\\n\")
    
    # Interface summary
    :local upCount 0
    :local downCount 0
    :foreach iface in=[/interface find disabled=no] do={
        :if ([/interface get \$iface running]) do={
            :set upCount (\$upCount + 1)
        } else={
            :set downCount (\$downCount + 1)
        }
    }
    :set report (\$report . \"\\r\\nInterfaces:\\r\\n\")
    :set report (\$report . \"  Up: \" . \$upCount . \"  Down: \" . \$downCount . \"\\r\\n\")
    
    # Recent log errors
    :local errors [/log find where topics~\"error\"]
    :set report (\$report . \"\\r\\nRecent Errors: \" . [:len \$errors] . \"\\r\\n\")
    
    /tool e-mail send to=\$ALERT_TO \
        subject=(\"[Hourly Digest] \" . \$router) \
        body=\$report
" comment="Hourly digest email"

# --- Setup Schedulers ---
/system scheduler add name="complete-monitor" interval=5m \
    on-event="/system script run complete-monitor"
/system scheduler add name="hourly-digest" interval=1h \
    on-event="/system script run hourly-digest"
/system scheduler add name="daily-report" start-time=08:00:00 interval=1d \
    on-event="/system script run daily-email-report"

:put "Complete Alert System configured!"
:put "Monitoring: CPU, Memory, Interfaces, DHCP"
:put "Alerts: Email with throttling"
:put "Reports: Hourly digest + Daily summary"
```

---

## Summary ของ Part 31

| Feature | Command | หมายเหตุ |
|---------|---------|---------|
| Configure SMTP | `/tool e-mail set` | |
| Send email | `/tool e-mail send` | |
| With attachment | `... file=filename` | |
| Multiple recipients | `to="a@b,c@d"` | |
| With CC | `... cc="email"` | |
| Test config | Send test email | ทดสอบก่อน production |

> **Warning:** Gmail ต้องใช้ App Password ไม่ใช่ password ปกติ (ต้องเปิด 2FA ก่อน)

> **Tip:** ใช้ alert throttling เพื่อไม่ให้ inbox เต็มด้วย duplicate alerts

> **Best Practice:** แยก email สำหรับ router ต่างหาก เช่น `router-alerts@company.com`

---

[← Part 30: Firewall Automation](part-030-firewall-automation.md) | [Part 32: Hotspot Management →](part-032-hotspot-management.md)
