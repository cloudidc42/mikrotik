# Part 25: Global Variables ใน RouterOS Scripting

## บทนำ

Global variables ใน RouterOS ช่วยให้ scripts ต่างๆ สามารถแชร์ข้อมูลกันได้ เป็นเครื่องมือสำคัญสำหรับ inter-script communication, state management, และการเก็บข้อมูลชั่วคราวที่ต้องใช้ข้ามหลาย scripts

---

## 25.1 :global Declaration

### การประกาศ Global Variables

```routeros
# ประกาศ global variable
:global myGlobal

# ประกาศพร้อมค่าเริ่มต้น
:global counter 0
:global routerName "MikroTik-01"
:global serverList {}

# ประกาศหลายตัวพร้อมกัน
:global status "running"
:global startTime [/system clock get time]
:global errorCount 0

# ตรวจสอบ type
:global myVar "hello"
:put [:typeof $myVar]    # str

# Reassign
:global counter
:set counter 10
:put $counter    # 10

# Global จะ persist ตลอด session (จนกว่าจะ reboot)
```

### Local vs Global

```routeros
# Local variable - ใช้ได้เฉพาะใน script block นั้น
:local localVar "I am local"

# Global variable - ใช้ได้จากทุกที่
:global globalVar "I am global"

# ตัวอย่างความแตกต่าง
/system script add name="test-scope" source="
    :local testLocal \"local value\"
    :global testGlobal \"global value\"
    :put \"Inside script:\"
    :put \$testLocal     # OK
    :put \$testGlobal    # OK
"

/system script run test-scope

# หลังจาก script รัน
:global testGlobal
:put $testGlobal    # "global value" - ยังอยู่!

# :local testLocal ใช้ไม่ได้จาก outside script
```

---

## 25.2 Sharing Data Between Scripts

### Script 1 Set, Script 2 Read

```routeros
# Script 1: เก็บข้อมูล
/system script add name="collector" source="
    :global collectedData
    :global collectionTime
    
    :set collectedData {
        \"cpu\": [/system resource get cpu-load];
        \"mem\": [/system resource get free-memory];
        \"uptime\": [/system resource get uptime]
    }
    :set collectionTime [/system clock get time]
    :log info \"Data collected\"
"

# Script 2: ใช้ข้อมูล
/system script add name="reporter" source="
    :global collectedData
    :global collectionTime
    
    :if ([:typeof \$collectedData] = \"nothing\") do={
        :put \"No data collected yet\"
        :return
    }
    
    :put \"Report at: \$collectionTime\"
    :put \"CPU: \" . (\$collectedData->\"cpu\") . \"%\"
    :put \"Memory: \" . (\$collectedData->\"mem\")
    :put \"Uptime: \" . (\$collectedData->\"uptime\")
"

# รัน Script 1 ก่อน แล้วค่อย Script 2
/system script run collector
/system script run reporter
```

### Shared State Between Scripts

```routeros
# Global state machine
:global appState "stopped"
:global appData {}
:global appErrors {}

# Script: Start application
/system script add name="app-start" source="
    :global appState
    :global appData
    :global appErrors
    
    :if (\$appState = \"running\") do={
        :put \"Already running\"
        :return
    }
    
    :set appState \"running\"
    :set appData {}
    :set appErrors {}
    :log info \"Application started\"
"

# Script: Process data
/system script add name="app-process" source="
    :global appState
    :global appData
    
    :if (\$appState != \"running\") do={
        :put \"Application not running\"
        :return
    }
    
    # เก็บข้อมูล
    :local newData [/system resource get cpu-load]
    :set appData (\$appData , \$newData)
    :log info (\"Processed: \" . \$newData)
"

# Script: Stop application
/system script add name="app-stop" source="
    :global appState
    :global appData
    
    :set appState \"stopped\"
    :log info (\"Application stopped with \" . [:len \$appData] . \" data points\")
"
```

---

## 25.3 Environment Variables

```routeros
# Global variables ที่ใช้เป็น environment config
:global CONFIG_FTP_SERVER "192.168.1.100"
:global CONFIG_FTP_USER "backup"
:global CONFIG_FTP_PASS "BackupPass"
:global CONFIG_SMTP_SERVER "mail.company.com"
:global CONFIG_ADMIN_EMAIL "admin@company.com"
:global CONFIG_BACKUP_PATH "/backups/"
:global CONFIG_MAX_BACKUPS 7
:global CONFIG_LOG_LEVEL "info"

# Scripts อื่นๆ ใช้ config เหล่านี้
/system script add name="use-config" source="
    :global CONFIG_FTP_SERVER
    :global CONFIG_FTP_USER
    :global CONFIG_FTP_PASS
    :global CONFIG_BACKUP_PATH
    
    # ใช้ config ที่ define ไว้
    /tool fetch address=\$CONFIG_FTP_SERVER \
        src-path=\"backup.rsc\" \
        user=\$CONFIG_FTP_USER \
        password=\$CONFIG_FTP_PASS \
        dst-path=(\$CONFIG_BACKUP_PATH . \"backup.rsc\") \
        upload=yes
"
```

### Initialize Config

```routeros
# Script เพื่อ initialize config ทั้งหมด
/system script add name="init-config" source="
    # FTP Settings
    :global CONFIG_FTP_SERVER \"192.168.1.100\"
    :global CONFIG_FTP_USER \"backup\"
    :global CONFIG_FTP_PASS \"BackupPass\"
    :global CONFIG_FTP_PATH \"/router-backups/\"
    
    # Email Settings
    :global CONFIG_SMTP_SERVER \"smtp.gmail.com\"
    :global CONFIG_SMTP_PORT 587
    :global CONFIG_EMAIL_FROM \"router@company.com\"
    :global CONFIG_EMAIL_TO \"admin@company.com\"
    
    # Monitoring Settings
    :global CONFIG_CHECK_INTERVAL 5
    :global CONFIG_ALERT_CPU 80
    :global CONFIG_ALERT_MEM 20000000
    
    # Feature Flags
    :global FEATURE_FTP_BACKUP true
    :global FEATURE_EMAIL_ALERTS true
    :global FEATURE_AUTO_BLOCK false
    
    :log info \"Configuration initialized\"
    :put \"Config loaded successfully\"
"

# รัน init ก่อนรัน scripts อื่นๆ
/system script run init-config
```

---

## 25.4 Script Environment

```routeros
# ดู global variables ปัจจุบัน
:environment print

# ดูเฉพาะ globals ที่ set ไว้
:global testVar "hello"
:environment print

# ล้าง global variable
:global testVar
:set testVar ""

# Check ว่า global exists
:global myGlobal
:if ([:typeof $myGlobal] = "nothing") do={
    :put "Variable not set"
    :set myGlobal "default value"
}
:put $myGlobal

# Global ที่เป็น function
:global greet
:set greet do={
    :local name $1
    :return ("Hello, " . $name . "!")
}

:put [$greet "MikroTik"]
# Output: "Hello, MikroTik!"
```

---

## 25.5 Persistent Variables (Using Files)

Global variables หายไปเมื่อ reboot ถ้าต้องการ persistent ต้องใช้ files:

```routeros
# Save global variable ลงไฟล์
:local saveGlobal do={
    :local varName $1
    :local varValue $2
    :local fileName ("global-" . $varName . ".txt")
    
    /file remove [find name=$fileName]
    /tool fetch url="data:," dst-path=$fileName
    /file set [find name=$fileName] contents=[:tostr $varValue]
}

# Load global variable จากไฟล์
:local loadGlobal do={
    :local varName $1
    :local fileName ("global-" . $varName . ".txt")
    :local fileId [/file find name=$fileName]
    
    :if ([:len $fileId] = 0) do={
        :return ""
    }
    
    :return [/file get $fileId contents]
}

# ตัวอย่างการใช้งาน
:global persistCounter

# Load จากไฟล์
:set persistCounter [$loadGlobal "counter"]
:if ([:len $persistCounter] = 0) do={
    :set persistCounter 0
}

# ใช้งาน
:set persistCounter ($persistCounter + 1)
:put "Counter: $persistCounter"

# Save กลับลงไฟล์
[$saveGlobal "counter" $persistCounter]
```

### Persistent Config Manager

```routeros
# System สำหรับ persistent settings
/system script add name="config-manager" source="
    # Save config
    :global saveConfig
    :set saveConfig do={
        :local key \$1
        :local value \$2
        :local fileName (\"config-\" . \$key . \".cfg\")
        
        /file remove [find name=\$fileName]
        /tool fetch url=\"data:,\" dst-path=\$fileName
        /file set [find name=\$fileName] contents=[:tostr \$value]
        :return true
    }
    
    # Load config
    :global loadConfig
    :set loadConfig do={
        :local key \$1
        :local default \$2
        :local fileName (\"config-\" . \$key . \".cfg\")
        :local fileId [/file find name=\$fileName]
        
        :if ([:len \$fileId] = 0) do={
            :return \$default
        }
        
        :return [/file get \$fileId contents]
    }
    
    # Delete config
    :global deleteConfig
    :set deleteConfig do={
        :local key \$1
        :local fileName (\"config-\" . \$key . \".cfg\")
        /file remove [find name=\$fileName]
    }
"

# ใช้งาน
/system script run config-manager

:global saveConfig
:global loadConfig

# บันทึก
[$saveConfig "last-backup-date" "2025-01-01"]
[$saveConfig "backup-count" "42"]

# โหลด
:local lastBackup [$loadConfig "last-backup-date" "never"]
:put "Last backup: $lastBackup"

:local count [$loadConfig "backup-count" "0"]
:put "Backup count: $count"
```

---

## 25.6 Inter-script Communication

### Callback Pattern

```routeros
# Parent script ส่ง callback ให้ child script
:global onComplete
:global onError

:set onComplete do={
    :local result $1
    :put "Task completed with: $result"
    :log info message=("Task done: " . $result)
}

:set onError do={
    :local error $1
    :put "Task failed: $error"
    :log error message=("Task error: " . $error)
}

/system script add name="async-task" source="
    :global onComplete
    :global onError
    
    :do {
        # ทำงาน
        :local result \"success-data\"
        [\$onComplete \$result]
    } on-error={
        [\$onError \"Connection failed\"]
    }
"

/system script run async-task
```

### Message Queue

```routeros
# ใช้ global array เป็น message queue
:global messageQueue {}

# Publish function
:global publish
:set publish do={
    :global messageQueue
    :local msg {$1; $2; [/system clock get time]}
    :set messageQueue ($messageQueue , $msg)
}

# Subscribe/consume function
:global consume
:set consume do={
    :global messageQueue
    :if ([:len $messageQueue] = 0) do={ :return "" }
    
    :local msg ($messageQueue->0)
    :set messageQueue [:pick $messageQueue 1 [:len $messageQueue]]
    :return $msg
}

# Publisher script
[$publish "alert" "Interface ether1 is down"]
[$publish "info" "Backup completed"]
[$publish "warning" "CPU at 85%"]

# Consumer script
:while ([:len $messageQueue] > 0) do={
    :local msg [$consume]
    :local type ($msg->0)
    :local content ($msg->1)
    :local time ($msg->2)
    :put "[$time][$type] $content"
}
```

---

## 25.7 Variable Cleanup

```routeros
# ล้าง global variables เมื่อเสร็จงาน
:local cleanupGlobals do={
    # Reset ค่าให้เป็น nothing
    :global tempData
    :global processStatus
    :global tempIpList
    
    :set tempData
    :set processStatus
    :set tempIpList
    
    :put "Globals cleaned up"
}

# หลัง script ทำงานเสร็จ
[$cleanupGlobals]

# ตัวอย่างการใช้ cleanup ใน main script
/system script add name="clean-script" source="
    # Declare globals
    :global taskRunning
    :global taskData
    :global taskResult
    
    # Set initial values
    :set taskRunning true
    :set taskData {}
    :set taskResult \"\"
    
    :do {
        # Main work
        :set taskData {\"key\": \"value\"}
        :set taskResult \"success\"
    } on-error={
        :set taskResult \"failed\"
    } 
    
    # Always cleanup
    :set taskRunning false
    :set taskData {}
    :put \"Task: \$taskResult\"
"
```

### Lifecycle Management

```routeros
# Pattern: Initialize, Use, Cleanup
:global initApp do={
    :global APP_STATE
    :global APP_DATA
    :global APP_ERRORS
    :global APP_START_TIME
    
    :set APP_STATE "initializing"
    :set APP_DATA {}
    :set APP_ERRORS {}
    :set APP_START_TIME [/system clock get time]
    
    :set APP_STATE "ready"
    :log info "App initialized"
}

:global stopApp do={
    :global APP_STATE
    :global APP_DATA
    :global APP_ERRORS
    
    :set APP_STATE "stopping"
    
    # Save any important data
    :local errorCount [:len $APP_ERRORS]
    :if ($errorCount > 0) do={
        :log warning message=("App stopped with " . $errorCount . " errors")
    }
    
    # Cleanup
    :set APP_DATA {}
    :set APP_ERRORS {}
    :set APP_STATE "stopped"
    :log info "App stopped"
}

[$initApp]
# ... use app ...
[$stopApp]
```

---

## 25.8 Thread Safety Considerations

RouterOS scripts run single-threaded แต่ schedulers อาจรัน scripts ซ้อนกันได้:

```routeros
# Mutex pattern สำหรับ critical sections
:global scriptMutex false

:local acquireMutex do={
    :global scriptMutex
    :local timeout 30  # seconds
    :local waited 0
    
    :while ($scriptMutex = true && $waited < $timeout) do={
        :delay 1s
        :set waited ($waited + 1)
    }
    
    :if ($scriptMutex = true) do={
        :put "Could not acquire mutex (timeout)"
        :return false
    }
    
    :set scriptMutex true
    :return true
}

:local releaseMutex do={
    :global scriptMutex
    :set scriptMutex false
}

# ใช้ใน critical section
:if ([$acquireMutex]) do={
    :do {
        # Critical section - ทำงานที่ไม่ควรรันพร้อมกัน
        :put "In critical section"
        :delay 2s
    } on-error={
        [$releaseMutex]
        :error "Critical section failed"
    }
    [$releaseMutex]
} else={
    :put "Another script is running, skipping"
}
```

---

## 25.9 Best Practices

```routeros
# 1. ตั้งชื่อ Global variables ให้ชัดเจน (prefix ด้วย module name)
:global BACKUP_FTP_SERVER "192.168.1.100"
:global MONITOR_CPU_THRESHOLD 80
:global ALERT_EMAIL "admin@example.com"

# 2. Initialize ก่อนใช้เสมอ
:global counter
:if ([:typeof $counter] = "nothing") do={
    :set counter 0
}

# 3. Document globals ด้วย comment
# :global LOG_LEVEL - log verbosity: debug/info/warning/error
:global LOG_LEVEL "info"

# 4. หลีกเลี่ยง conflicts ด้วย naming convention
# Bad: :global x
# Good: :global BACKUP_COUNT
# Good: :global monitorLastCheck

# 5. ทำ cleanup เมื่อเสร็จงาน
:global tempBuffer {}
# ... use tempBuffer ...
:set tempBuffer {}  # cleanup

# 6. ใช้ type checking
:global userCount
:if ([:typeof $userCount] != "num") do={
    :set userCount 0
}
:set userCount ($userCount + 1)
```

---

## Lab 25: Counter with Global Variables

### โจทย์
สร้างระบบ counter ที่ persist ข้ามการ reboot:
- Script 1: Increment counter
- Script 2: Reset counter
- Script 3: Get counter value
- Persist ลงไฟล์
- Thread-safe

### Solution

```routeros
# Counter System with Global Variables Lab
# =========================================

# --- Initialize Counter System ---
/system script add name="counter-init" source="
    :global counterMutex false
    :global counterValue
    :global counterLastReset
    
    # Load จากไฟล์ถ้ามี
    :local fileId [/file find name=\"counter-state.txt\"]
    :if ([:len \$fileId] > 0) do={
        :local content [/file get \$fileId contents]
        # Parse: value|lastReset
        :local sepPos [:find \$content \"|\"]
        :if (\$sepPos != -1) do={
            :set counterValue [:toint [:pick \$content 0 \$sepPos]]
            :set counterLastReset [:pick \$content (\$sepPos + 1) [:len \$content]]
        }
        :log info (\"Counter loaded: \" . \$counterValue)
    } else={
        :set counterValue 0
        :set counterLastReset [/system clock get date]
        :log info \"Counter initialized to 0\"
    }
" comment="Initialize counter system"

# --- Increment Counter ---
/system script add name="counter-increment" source="
    :global counterMutex
    :global counterValue
    
    # Acquire mutex
    :local timeout 10
    :local waited 0
    :while (\$counterMutex = true && \$waited < \$timeout) do={
        :delay 1s
        :set waited (\$waited + 1)
    }
    :if (\$counterMutex = true) do={
        :log error \"Counter: mutex timeout\"
        :return
    }
    :set counterMutex true
    
    # Increment
    :if ([:typeof \$counterValue] = \"nothing\") do={
        :set counterValue 0
    }
    :set counterValue (\$counterValue + 1)
    :put \"Counter: \$counterValue\"
    
    # Save ลงไฟล์
    :global counterLastReset
    :local state (\$counterValue . \"|\" . \$counterLastReset)
    /file remove [find name=\"counter-state.txt\"]
    /tool fetch url=\"data:,\" dst-path=\"counter-state.txt\"
    /file set [find name=\"counter-state.txt\"] contents=\$state
    
    # Release mutex
    :set counterMutex false
    :log info (\"Counter incremented to: \" . \$counterValue)
" comment="Increment counter by 1"

# --- Reset Counter ---
/system script add name="counter-reset" source="
    :global counterMutex
    :global counterValue
    :global counterLastReset
    
    :set counterMutex true
    :set counterValue 0
    :set counterLastReset [/system clock get date]
    
    # Save
    :local state (\"0|\" . \$counterLastReset)
    /file remove [find name=\"counter-state.txt\"]
    /tool fetch url=\"data:,\" dst-path=\"counter-state.txt\"
    /file set [find name=\"counter-state.txt\"] contents=\$state
    
    :set counterMutex false
    :put \"Counter reset to 0\"
    :log info \"Counter reset\"
" comment="Reset counter to 0"

# --- Get Counter ---
/system script add name="counter-get" source="
    :global counterValue
    :global counterLastReset
    
    :if ([:typeof \$counterValue] = \"nothing\") do={
        :put \"Counter not initialized\"
        :return
    }
    
    :put \"=============================\"
    :put \"Counter Value: \$counterValue\"
    :put \"Last Reset: \$counterLastReset\"
    :put \"=============================\"
" comment="Display current counter value"

# --- Demo ---
/system script run counter-init
/system script run counter-increment
/system script run counter-increment
/system script run counter-increment
/system script run counter-get

# Output:
# Counter: 1
# Counter: 2
# Counter: 3
# =============================
# Counter Value: 3
# Last Reset: jan/01/2025
# =============================

/system script run counter-reset
/system script run counter-get

# Output:
# Counter reset to 0
# =============================
# Counter Value: 0
# Last Reset: jan/01/2025
# =============================

# Setup scheduler ให้ increment ทุกชั่วโมง
/system scheduler add \
    name="counter-hourly" \
    interval=1h \
    on-event="/system script run counter-increment" \
    comment="Increment hourly counter"
```

---

## Summary ของ Part 25

| Feature | Command | หมายเหตุ |
|---------|---------|---------|
| Declare global | `:global varName` | ประกาศก่อนใช้ |
| Set global | `:set varName value` | |
| Read global | `$varName` | ต้อง declare ก่อน |
| Check exists | `[:typeof $var] = "nothing"` | |
| Clear global | `:set varName` | ไม่ใส่ค่า |
| List globals | `:environment print` | |
| Persist global | เขียนลง file | ใช้ /file operations |

> **Important:** Global variables หายหลัง reboot เสมอ หากต้องการ persistent ต้องใช้ files

> **Tip:** ตั้งชื่อ globals ด้วย SCREAMING_CASE เพื่อให้รู้ว่าเป็น global

> **Warning:** ใช้ mutex pattern เมื่อ scripts หลาย scripts อาจ modify global เดียวกัน

---

[← Part 24: Scheduler](part-024-scheduler.md) | [Part 26: Error Handling →](part-026-error-handling.md)
