# Part 26: Error Handling ใน RouterOS Scripting

## บทนำ

Error handling เป็นส่วนสำคัญของการเขียน production-grade scripts ใน RouterOS การจัดการ errors อย่างถูกต้องช่วยให้ scripts ทำงานได้น่าเชื่อถือ ตรวจพบปัญหาได้เร็ว และกู้คืนสถานะได้เมื่อเกิดข้อผิดพลาด

---

## 26.1 :do...on-error

### รูปแบบพื้นฐาน

```routeros
# รูปแบบพื้นฐานของ try-catch ใน RouterOS
:do {
    # ส่วนที่อาจเกิด error
    :local result [/tool fetch url="http://example.com" dst-path="test.txt"]
} on-error={
    # จัดการ error
    :put "Error occurred!"
}

# ตัวอย่าง: แปลงตัวเลข
:do {
    :local num [:tonum "not-a-number"]
    :put "Converted: $num"
} on-error={
    :put "Invalid number format"
}

# ตัวอย่าง: เข้าถึง object ที่ไม่มีอยู่
:do {
    :local iface [/interface get [find name="ether99"]]
} on-error={
    :put "Interface ether99 not found"
}

# Nested try-catch
:do {
    :do {
        :error "inner error"
    } on-error={
        :put "Inner error caught"
        :error "re-throw"
    }
} on-error={
    :put "Outer error caught"
}
```

### ดัก Error ด้วย Variable

```routeros
# ดัก error message
:local errMsg ""
:do {
    :error "My custom error"
} on-error={
    # $error มีค่า error message ใน RouterOS 6.x+
    :set errMsg $1  
    :put "Error: $errMsg"
}

# ตัวอย่างจริง
:do {
    /ip address add address=999.999.999.999/24 interface=ether1
} on-error={
    :put "Failed to add IP address"
    :log error "Invalid IP address configuration"
}
```

---

## 26.2 Error Types ใน RouterOS

### Runtime Errors

```routeros
# 1. Type conversion errors
:do {
    :local n [:tonum "abc"]
} on-error={ :put "Type error" }

# 2. Object not found errors
:do {
    /interface get [find name="nonexistent"]
} on-error={ :put "Not found error" }

# 3. Permission errors
:do {
    # พยายามทำสิ่งที่ไม่มีสิทธิ์
} on-error={ :put "Permission error" }

# 4. Network errors
:do {
    /tool fetch url="http://1.2.3.4/nonexistent" dst-path="test.txt"
} on-error={ :put "Network error" }

# 5. Custom errors (ที่เราสร้างเอง)
:local validate do={
    :local value $1
    :if ($value < 0) do={
        :error "Value cannot be negative"
    }
    :if ($value > 100) do={
        :error "Value cannot exceed 100"
    }
    :return $value
}

:do {
    :local result [$validate -5]
} on-error={
    :put "Validation error occurred"
}
```

### Error Categories

| ประเภท Error | สาเหตุ | วิธีป้องกัน |
|-------------|--------|-----------|
| Type error | แปลงประเภทข้อมูลผิด | Validate input ก่อน |
| Not found | Object/file ไม่มี | ตรวจสอบการมีอยู่ก่อน |
| Network error | Connection fail | Retry + timeout |
| Config error | Config ไม่ถูกต้อง | Validate ก่อน apply |
| Resource error | Memory/disk เต็ม | Monitor resources |
| Permission error | ไม่มีสิทธิ์ | ใช้ user ที่มีสิทธิ์ |

---

## 26.3 Graceful Degradation

```routeros
# Script ที่ทำงานต่อได้แม้ส่วนใดส่วนหนึ่ง fail

:local success true
:local results {}

# Task 1: Backup config
:do {
    /export file=backup
    :set results ($results , "backup:OK")
    :put "Task 1 (backup): OK"
} on-error={
    :set results ($results , "backup:FAILED")
    :set success false
    :put "Task 1 (backup): FAILED - continuing..."
}

# Task 2: Upload to FTP
:do {
    /tool fetch address="192.168.1.100" src-path="backup.rsc" \
        user="backup" password="pass" dst-path="/backups/backup.rsc" \
        upload=yes
    :set results ($results , "ftp:OK")
    :put "Task 2 (FTP): OK"
} on-error={
    :set results ($results , "ftp:FAILED")
    :set success false
    :put "Task 2 (FTP): FAILED - continuing..."
}

# Task 3: Send email
:do {
    /tool e-mail send to="admin@example.com" subject="Backup Done" \
        body="Backup completed"
    :set results ($results , "email:OK")
    :put "Task 3 (email): OK"
} on-error={
    :set results ($results , "email:FAILED")
    :put "Task 3 (email): FAILED - non-critical"
}

# Summary
:if ($success) do={
    :put "All critical tasks completed"
} else={
    :put "Some tasks failed: $results"
    :log warning message="Backup incomplete"
}
```

---

## 26.4 Logging Errors

```routeros
# Error logging function
:local logError do={
    :local level $1      # error/warning/critical
    :local source $2     # script name
    :local message $3    # error message
    
    :local fullMsg ("[$source] " . $message)
    
    :if ($level = "error") do={
        :log error message=$fullMsg
    }
    :if ($level = "warning") do={
        :log warning message=$fullMsg
    }
    :if ($level = "critical") do={
        :log critical message=$fullMsg
    }
    
    :put "[$level] $fullMsg"
}

# ใช้งาน
[$logError "error" "backup-script" "FTP connection failed"]
[$logError "warning" "monitor" "CPU above 80%"]
[$logError "critical" "firewall" "Unexpected rule change detected"]

# Log ด้วย stack trace (simplified)
:local logWithTrace do={
    :local source $1
    :local message $2
    :local context $3
    
    :local trace ("Source: " . $source . " | Context: " . $context)
    :log error message=(message . " | " . $trace)
    
    # บันทึกลงไฟล์
    :local logEntry ([/system clock get date] . " " . \
                    [/system clock get time] . " [" . $source . "] " . \
                    $message . " | " . $trace . "\r\n")
    
    :local fileId [/file find name="error.log"]
    :local existing ""
    :if ([:len $fileId] > 0) do={
        :set existing [/file get $fileId contents]
    }
    
    /file remove [find name="error.log"]
    /tool fetch url="data:," dst-path="error.log"
    /file set [find name="error.log"] contents=($existing . $logEntry)
}

[$logWithTrace "ftp-backup" "Connection timeout" "host=192.168.1.100 port=21"]
```

---

## 26.5 Email Alerts on Errors

```routeros
# ส่ง email เมื่อเกิด error สำคัญ
:local sendErrorAlert do={
    :local subject $1
    :local message $2
    :local severity $3
    
    :local router [/system identity get name]
    :local time [/system clock get date]
    :local body ("ALERT from Router: " . $router . "\r\n" . \
                "Time: " . $time . "\r\n" . \
                "Severity: " . $severity . "\r\n" . \
                "Message: " . $message . "\r\n")
    
    :do {
        /tool e-mail send \
            to="admin@example.com" \
            subject=("[" . $severity . "] " . $subject . " - " . $router) \
            body=$body
        :log info "Alert email sent"
    } on-error={
        :log error "Failed to send alert email"
    }
}

# ตัวอย่างใช้งาน
:do {
    /tool fetch url="http://1.2.3.4" dst-path="test.txt"
} on-error={
    [$sendErrorAlert "Connection Failed" "Cannot reach primary server 1.2.3.4" "ERROR"]
}

# Alert สำหรับ critical events
:local cpu [/system resource get cpu-load]
:if ($cpu > 95) do={
    [$sendErrorAlert "High CPU Alert" ("CPU at " . $cpu . "%") "CRITICAL"]
}

# Throttle alerts (ไม่ส่งถี่เกินไป)
:global lastAlertTime
:local alertCooldown 300  # 5 minutes

:local sendThrottledAlert do={
    :global lastAlertTime
    :local now [/system resource get uptime]
    
    :if ([:typeof $lastAlertTime] = "nothing") do={
        :set lastAlertTime 0
    }
    
    # Check cooldown
    :if (($now - $lastAlertTime) > $alertCooldown) do={
        [$sendErrorAlert $1 $2 $3]
        :set lastAlertTime $now
    } else={
        :log info "Alert throttled (too frequent)"
    }
}
```

---

## 26.6 Retry Logic

```routeros
# Retry function พร้อม exponential backoff
:local withRetry do={
    :local action $1
    :local maxRetries $2
    :local baseDelay $3
    
    :local attempt 0
    :local success false
    
    :while (!$success && $attempt < $maxRetries) do={
        :set attempt ($attempt + 1)
        :put "Attempt $attempt/$maxRetries..."
        
        :do {
            [$action]
            :set success true
            :put "Success on attempt $attempt"
        } on-error={
            :if ($attempt < $maxRetries) do={
                :local delay ($baseDelay * $attempt)
                :put "Failed, retrying in ${delay}s..."
                :delay ($delay . "s")
            } else={
                :put "All retries exhausted"
                :error "Max retries reached"
            }
        }
    }
}

# ตัวอย่าง: retry FTP upload
:local ftpUpload do={
    /tool fetch address="192.168.1.100" \
        src-path="backup.rsc" \
        user="backup" \
        password="pass" \
        dst-path="/backups/backup.rsc" \
        upload=yes
}

:do {
    [$withRetry $ftpUpload 3 5]
} on-error={
    :put "FTP upload failed after all retries"
    [$sendErrorAlert "FTP Upload Failed" "Cannot upload backup" "ERROR"]
}

# Retry with specific conditions
:local retryWithCondition do={
    :local maxAttempts $1
    :local delaySeconds $2
    :local attempts 0
    
    :while ($attempts < $maxAttempts) do={
        :set attempts ($attempts + 1)
        
        :do {
            # พยายาม ping server
            /tool ping 192.168.1.100 count=3
            :put "Server reachable on attempt $attempts"
            :return true
        } on-error={
            :put "Ping failed on attempt $attempts"
            :if ($attempts < $maxAttempts) do={
                :delay ($delaySeconds . "s")
            }
        }
    }
    :return false
}

:if ([$retryWithCondition 5 10]) do={
    :put "Server is up!"
} else={
    :put "Server unreachable after 5 attempts"
}
```

---

## 26.7 Validation Before Action

```routeros
# Validate ก่อนทำการเปลี่ยนแปลง
:local validateIPAddress do={
    :local ip $1
    :local valid true
    
    # ตรวจสอบ format
    :local parts {}
    :local remaining $ip
    :local dotCount 0
    
    :while ([:find $remaining "."] != -1) do={
        :local dotPos [:find $remaining "."]
        :local octet [:pick $remaining 0 $dotPos]
        :local octetNum [:toint $octet]
        
        :if ($octetNum < 0 || $octetNum > 255) do={
            :set valid false
        }
        
        :set remaining [:pick $remaining ($dotPos + 1) [:len $remaining]]
        :set dotCount ($dotCount + 1)
    }
    
    # Check last octet
    :local lastOctet [:toint $remaining]
    :if ($lastOctet < 0 || $lastOctet > 255) do={ :set valid false }
    
    # ต้องมี 4 octets (3 dots)
    :if ($dotCount != 3) do={ :set valid false }
    
    :return $valid
}

# Validate IP ก่อนเพิ่ม
:local addIPSafe do={
    :local ip $1
    :local iface $2
    :local prefix $3
    
    :if (![$validateIPAddress $ip]) do={
        :put "Invalid IP address: $ip"
        :return false
    }
    
    :if ([:len [/interface find name=$iface]] = 0) do={
        :put "Interface not found: $iface"
        :return false
    }
    
    :do {
        /ip address add address=($ip . "/" . $prefix) interface=$iface
        :put "Added IP: $ip/$prefix on $iface"
        :return true
    } on-error={
        :put "Failed to add IP: $ip"
        :return false
    }
}

[$addIPSafe "192.168.1.1" "ether1" "24"]
[$addIPSafe "999.999.0.1" "ether1" "24"]  # จะ fail validation

# Validate firewall rule ก่อน apply
:local validateFirewallRule do={
    :local chain $1
    :local action $2
    :local srcIP $3
    
    :local validChains {"input"; "forward"; "output"}
    :local validActions {"accept"; "drop"; "reject"; "log"}
    
    # Check chain
    :local chainValid false
    :foreach c in=$validChains do={
        :if ($c = $chain) do={ :set chainValid true }
    }
    :if (!$chainValid) do={
        :put "Invalid chain: $chain"
        :return false
    }
    
    # Check action
    :local actionValid false
    :foreach a in=$validActions do={
        :if ($a = $action) do={ :set actionValid true }
    }
    :if (!$actionValid) do={
        :put "Invalid action: $action"
        :return false
    }
    
    # Check source IP if provided
    :if ([:len $srcIP] > 0) do={
        :if (![$validateIPAddress $srcIP]) do={
            :put "Invalid source IP: $srcIP"
            :return false
        }
    }
    
    :return true
}
```

---

## 26.8 Testing Error Conditions

```routeros
# Unit test framework สำหรับ error handling
:local runTest do={
    :local testName $1
    :local testFunc $2
    :local expectedResult $3
    
    :local result ""
    :do {
        :set result [$testFunc]
    } on-error={
        :set result "ERROR"
    }
    
    :if ($result = $expectedResult) do={
        :put "PASS: $testName"
    } else={
        :put "FAIL: $testName (expected: $expectedResult, got: $result)"
    }
}

# Test cases
:local testValidIP do={ :return [$validateIPAddress "192.168.1.1"] }
:local testInvalidIP do={ :return [$validateIPAddress "999.0.0.1"] }
:local testEmptyIP do={ :return [$validateIPAddress ""] }

[$runTest "Valid IP" $testValidIP true]
[$runTest "Invalid IP" $testInvalidIP false]
[$runTest "Empty IP" $testEmptyIP false]

# Test retry behavior
:local callCount 0
:local testFlaky do={
    :set callCount ($callCount + 1)
    :if ($callCount < 3) do={
        :error "Not ready yet"
    }
    :return "success"
}

:do {
    [$withRetry $testFlaky 5 1]
} on-error={
    :put "Test failed"
}
:put "Call count: $callCount"    # Should be 3
```

---

## 26.9 Production Error Handling Patterns

### Pattern 1: Circuit Breaker

```routeros
# Circuit breaker pattern
:global circuitState "closed"    # closed/open/half-open
:global circuitFailures 0
:global circuitLastFailure 0
:local circuitThreshold 5
:local circuitTimeout 60  # seconds

:local callWithCircuitBreaker do={
    :global circuitState
    :global circuitFailures
    :global circuitLastFailure
    
    :local action $1
    :local now [/system resource get uptime]
    
    # Check circuit state
    :if ($circuitState = "open") do={
        # Check if timeout passed
        :if (($now - $circuitLastFailure) > $circuitTimeout) do={
            :set circuitState "half-open"
            :put "Circuit half-open, trying..."
        } else={
            :put "Circuit OPEN - request blocked"
            :return false
        }
    }
    
    # Try the action
    :do {
        [$action]
        # Success - reset circuit
        :set circuitFailures 0
        :if ($circuitState = "half-open") do={
            :set circuitState "closed"
            :put "Circuit CLOSED - service recovered"
        }
        :return true
    } on-error={
        :set circuitFailures ($circuitFailures + 1)
        :set circuitLastFailure $now
        
        :if ($circuitFailures >= $circuitThreshold) do={
            :set circuitState "open"
            :put "Circuit OPEN - too many failures"
            [$sendErrorAlert "Circuit Breaker Opened" "Service unavailable" "CRITICAL"]
        }
        :return false
    }
}
```

### Pattern 2: Error Budget

```routeros
# Error budget tracking
:global errorBudget 100  # 100 credits = 100% uptime
:global errorCount 0

:local recordError do={
    :global errorBudget
    :global errorCount
    
    :set errorBudget ($errorBudget - 1)
    :set errorCount ($errorCount + 1)
    
    :if ($errorBudget < 10) do={
        :log critical "Error budget critically low!"
        [$sendErrorAlert "Error Budget Critical" \
            ("Only " . $errorBudget . " credits remaining") "CRITICAL"]
    }
    
    :if ($errorBudget <= 0) do={
        :log critical "Error budget exhausted - entering degraded mode"
    }
}

:local getHealthStatus do={
    :global errorBudget
    :local health ($errorBudget)
    
    :if ($health >= 80) do={ :return "healthy" }
    :if ($health >= 50) do={ :return "degraded" }
    :return "critical"
}
```

### Pattern 3: Dead Man's Switch

```routeros
# Dead man's switch - alert ถ้า script ไม่รันเป็นเวลานาน
:global lastHeartbeat
:global heartbeatExpiry 3600  # 1 hour

# Heartbeat script (รันทุก 30 นาที)
/system script add name="heartbeat" source="
    :global lastHeartbeat
    :set lastHeartbeat [/system resource get uptime]
    :log info \"Heartbeat: OK\"
"

# Watchdog script (รันทุก 1 ชั่วโมง)
/system script add name="watchdog" source="
    :global lastHeartbeat
    :global heartbeatExpiry
    
    :if ([:typeof \$lastHeartbeat] = \"nothing\") do={
        :log warning \"Watchdog: No heartbeat recorded\"
        :return
    }
    
    :local now [/system resource get uptime]
    :local elapsed (\$now - \$lastHeartbeat)
    
    :if (\$elapsed > \$heartbeatExpiry) do={
        :log critical (\"Watchdog: Heartbeat overdue! \" . \$elapsed . \"s since last\")
        # ทำ recovery actions
        /system script run heartbeat
    }
"

/system scheduler add name="heartbeat" interval=30m \
    on-event="/system script run heartbeat"
/system scheduler add name="watchdog" interval=1h \
    on-event="/system script run watchdog"
```

---

## Lab 26: Robust Script with Error Handling

### โจทย์
สร้าง production-grade backup script ที่มี:
- Complete error handling
- Retry logic
- Email alerts
- Logging
- Graceful degradation

### Solution

```routeros
# Robust Backup Script with Complete Error Handling
# ==================================================

# --- Configuration ---
:global BACKUP_FTP_HOST "192.168.1.100"
:global BACKUP_FTP_USER "backup"
:global BACKUP_FTP_PASS "BackupPass"
:global BACKUP_FTP_PATH "/backups/"
:global BACKUP_EMAIL "admin@example.com"
:global BACKUP_MAX_RETRIES 3
:global BACKUP_RETRY_DELAY 30

# --- Helper Functions ---

:global backupLog do={
    :local level $1
    :local msg $2
    :local line ([/system clock get time] . " [" . $level . "] " . $msg)
    :put $line
    :if ($level = "ERROR" || $level = "CRITICAL") do={
        :log error message=$msg
    } else={
        :log info message=$msg
    }
}

:global backupAlert do={
    :global BACKUP_EMAIL
    :local subject $1
    :local body $2
    :local router [/system identity get name]
    
    :do {
        /tool e-mail send \
            to=$BACKUP_EMAIL \
            subject=("[Router: " . $router . "] " . $subject) \
            body=("Router: " . $router . "\r\n\r\n" . $body)
    } on-error={
        :log warning "Could not send alert email"
    }
}

:global ftpUploadWithRetry do={
    :global BACKUP_FTP_HOST
    :global BACKUP_FTP_USER
    :global BACKUP_FTP_PASS
    :global BACKUP_FTP_PATH
    :global BACKUP_MAX_RETRIES
    :global BACKUP_RETRY_DELAY
    :global backupLog
    
    :local localFile $1
    :local attempt 0
    
    :while ($attempt < $BACKUP_MAX_RETRIES) do={
        :set attempt ($attempt + 1)
        [$backupLog "INFO" ("FTP upload attempt " . $attempt . " of " . $BACKUP_MAX_RETRIES)]
        
        :do {
            /tool fetch \
                address=$BACKUP_FTP_HOST \
                src-path=$localFile \
                user=$BACKUP_FTP_USER \
                password=$BACKUP_FTP_PASS \
                dst-path=($BACKUP_FTP_PATH . $localFile) \
                upload=yes
            [$backupLog "INFO" ("FTP upload success: " . $localFile)]
            :return true
        } on-error={
            [$backupLog "WARNING" ("FTP upload failed (attempt " . $attempt . ")")]
            :if ($attempt < $BACKUP_MAX_RETRIES) do={
                [$backupLog "INFO" ("Retrying in " . $BACKUP_RETRY_DELAY . "s...")]
                :delay ($BACKUP_RETRY_DELAY . "s")
            }
        }
    }
    
    [$backupLog "ERROR" ("FTP upload failed after " . $BACKUP_MAX_RETRIES . " attempts")]
    :return false
}

# --- Main Backup Function ---
:global runBackup do={
    :global backupLog
    :global backupAlert
    :global ftpUploadWithRetry
    
    :local hostname [/system identity get name]
    :local date [/system clock get date]
    :local dateStr ""
    :for i from=0 to=([:len [:tostr $date]] - 1) do={
        :local c [:pick [:tostr $date] $i ($i + 1)]
        :if ($c = "/") do={ :set dateStr ($dateStr . "-") } else={ :set dateStr ($dateStr . $c) }
    }
    
    :local backupName ($hostname . "-" . $dateStr)
    :local backupFile ($backupName . ".backup")
    :local rscFile ($backupName . ".rsc")
    :local status "SUCCESS"
    :local errors {}
    
    [$backupLog "INFO" "=== Backup Started ==="]
    [$backupLog "INFO" ("Hostname: " . $hostname)]
    [$backupLog "INFO" ("Files: " . $backupFile . ", " . $rscFile)]
    
    # Step 1: Create backup
    [$backupLog "INFO" "Creating system backup..."]
    :do {
        /system backup save name=$backupName password=""
        :delay 3s
        [$backupLog "INFO" "System backup created OK"]
    } on-error={
        :set status "PARTIAL"
        :set errors ($errors , "Backup creation failed")
        [$backupLog "ERROR" "Failed to create system backup"]
    }
    
    # Step 2: Export config
    [$backupLog "INFO" "Exporting configuration..."]
    :do {
        /export file=$backupName
        :delay 2s
        [$backupLog "INFO" "Config export OK"]
    } on-error={
        :set status "PARTIAL"
        :set errors ($errors , "Config export failed")
        [$backupLog "ERROR" "Failed to export configuration"]
    }
    
    # Step 3: Verify files
    [$backupLog "INFO" "Verifying backup files..."]
    :if ([:len [/file find name=$backupFile]] = 0 && \
         [:len [/file find name=$rscFile]] = 0) do={
        :set status "FAILED"
        :set errors ($errors , "No backup files created")
        [$backupLog "CRITICAL" "No backup files found!"]
        [$backupAlert "BACKUP FAILED" "No backup files were created on " . $hostname]
        :return false
    }
    
    # Step 4: FTP Upload
    [$backupLog "INFO" "Uploading to FTP..."]
    
    :if ([:len [/file find name=$backupFile]] > 0) do={
        :if (![$ftpUploadWithRetry $backupFile]) do={
            :set status "PARTIAL"
            :set errors ($errors , "Backup FTP upload failed")
        }
    }
    
    :if ([:len [/file find name=$rscFile]] > 0) do={
        :if (![$ftpUploadWithRetry $rscFile]) do={
            :set status "PARTIAL"
            :set errors ($errors , "RSC FTP upload failed")
        }
    }
    
    # Step 5: Cleanup old files (keep 7)
    [$backupLog "INFO" "Cleaning up old backups..."]
    :do {
        :local oldBackups [/file find where name~($hostname . "-") and name~".backup"]
        :local count [:len $oldBackups]
        :if ($count > 7) do={
            :local toDelete [:pick $oldBackups 0 ($count - 7)]
            :foreach f in=$toDelete do={
                :local fname [/file get $f name]
                /file remove $f
                [$backupLog "INFO" ("Cleaned: " . $fname)]
            }
        }
    } on-error={
        [$backupLog "WARNING" "Cleanup had errors (non-critical)"]
    }
    
    # Final status
    [$backupLog "INFO" ("=== Backup " . $status . " ===")]
    
    :if ($status = "FAILED" || $status = "PARTIAL") do={
        :local errMsg ""
        :foreach e in=$errors do={
            :set errMsg ($errMsg . "- " . $e . "\r\n")
        }
        [$backupAlert ("Backup " . $status) \
            ("Status: " . $status . "\r\n\r\nErrors:\r\n" . $errMsg)]
        :return false
    }
    
    # Send success notification (optional)
    :do {
        /tool e-mail send \
            to=$BACKUP_EMAIL \
            subject=("[Router: " . $hostname . "] Backup SUCCESS") \
            body=("Backup completed successfully\r\nFiles: " . $backupFile . "\r\n")
    } on-error={ }
    
    :return true
}

# --- Run Backup ---
[$runBackup]
```

---

## Summary ของ Part 26

| Pattern | ใช้เมื่อ |
|---------|---------|
| `:do...on-error` | ทุกการ operation ที่อาจ fail |
| Validation | ก่อน modify config/state |
| Retry | Network operations |
| Circuit Breaker | External service calls |
| Graceful Degradation | Tasks หลาย steps |
| Error Logging | Production scripts |
| Email Alert | Critical errors |

> **Golden Rule:** Script production ทุกตัวต้องมี error handling

> **Tip:** Log errors ด้วย context ที่เพียงพอให้ debug ได้

> **Warning:** `:error` จะหยุด script ทันที ถ้าไม่อยู่ใน `:do` block

---

[← Part 25: Global Variables](part-025-global-variables.md) | [Part 27: Script Parameters →](part-027-script-parameters.md)
