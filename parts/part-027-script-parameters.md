# Part 27: Script Parameters ใน RouterOS

## บทนำ

Script parameters ช่วยให้ scripts ของเรา flexible มากขึ้น สามารถ reuse ได้หลาย scenarios โดยแค่เปลี่ยน parameters แทนที่จะต้องสร้าง scripts ใหม่หลายตัว

---

## 27.1 :environment (Script Parameters)

### Script Environment

```routeros
# ดู environment ปัจจุบัน
:environment print

# Environment แสดง:
# - Local variables ใน scope ปัจจุบัน
# - Global variables ที่ declare แล้ว
# - Built-in variables

# Script ที่รับ parameters (function-style)
:local myFunc do={
    :local p1 $1   # Parameter 1
    :local p2 $2   # Parameter 2
    :local p3 $3   # Parameter 3
    
    :put "Param 1: $p1"
    :put "Param 2: $p2"
    :put "Param 3: $p3"
}

[$myFunc "hello" 42 true]
```

### Parameters ใน Named Scripts

```routeros
# Script ที่รับ parameters ผ่าน global variables
/system script add name="parameterized-script" source="
    # Script รับ params ผ่าน globals ที่ set ก่อนรัน
    :global PARAM_HOST
    :global PARAM_PORT
    :global PARAM_TIMEOUT
    
    # Default values
    :if ([:typeof \$PARAM_HOST] = \"nothing\") do={ :set PARAM_HOST \"192.168.1.1\" }
    :if ([:typeof \$PARAM_PORT] = \"nothing\") do={ :set PARAM_PORT 22 }
    :if ([:typeof \$PARAM_TIMEOUT] = \"nothing\") do={ :set PARAM_TIMEOUT 30 }
    
    :put \"Connecting to \$PARAM_HOST:\$PARAM_PORT with timeout \$PARAM_TIMEOUT\"
    # ... rest of script
"

# รัน script พร้อม parameters
:global PARAM_HOST "10.0.0.1"
:global PARAM_PORT 8080
:global PARAM_TIMEOUT 60
/system script run parameterized-script
```

---

## 27.2 Passing Parameters to Scripts

### Method 1: Global Variables

```routeros
# Set globals แล้วรัน script
:global SCRIPT_TARGET_IP "192.168.1.100"
:global SCRIPT_ACTION "backup"
:global SCRIPT_OUTPUT_DIR "/backups/"

/system script run process-target

# ใน process-target script:
# :global SCRIPT_TARGET_IP
# :global SCRIPT_ACTION
# :global SCRIPT_OUTPUT_DIR
# ... use them
```

### Method 2: Function Parameters ($1, $2, ...)

```routeros
# Function ที่รับ positional parameters
:local addFirewallRule do={
    :local chain $1
    :local srcAddr $2
    :local action $3
    :local comment $4
    
    /ip firewall filter add \
        chain=$chain \
        src-address=$srcAddr \
        action=$action \
        comment=$comment
    
    :put "Added rule: $chain $srcAddr -> $action"
}

# เรียกใช้
[$addFirewallRule "input" "192.168.100.0/24" "drop" "Block suspicious subnet"]
[$addFirewallRule "forward" "10.0.0.5" "drop" "Block compromised host"]

# Named parameters แบบ dictionary
:local createUser do={
    :local params $1
    :local name ($params->"name")
    :local group ($params->"group")
    :local pass ($params->"password")
    
    /user add name=$name group=$group password=$pass
    :put "Created user: $name (group: $group)"
}

# เรียกใช้ด้วย named params
[$createUser {"name"="techsupport"; "group"="write"; "password"="Secure123"}]
```

### Method 3: Environment Script

```routeros
# สร้าง script ที่ set environment แล้วรัน main script
:local runWithParams do={
    :local host $1
    :local user $2
    :local pass $3
    
    :global _RUN_HOST $host
    :global _RUN_USER $user
    :global _RUN_PASS $pass
    
    /system script run target-script
    
    # Cleanup params
    :set _RUN_HOST
    :set _RUN_USER
    :set _RUN_PASS
}

[$runWithParams "192.168.1.100" "admin" "password123"]
```

---

## 27.3 Default Values

```routeros
# Pattern 1: ternary-like default
:local myParam
:local effectiveParam
:if ([:typeof $myParam] = "nothing") do={
    :set effectiveParam "default-value"
} else={
    :set effectiveParam $myParam
}

# Pattern 2: helper function
:local getWithDefault do={
    :local val $1
    :local default $2
    :if ([:typeof $val] = "nothing" || [:len [:tostr $val]] = 0) do={
        :return $default
    }
    :return $val
}

# ใช้งาน
:global USER_CONFIG_INTERVAL
:local interval [$getWithDefault $USER_CONFIG_INTERVAL 60]
:put "Using interval: $interval"

# Pattern 3: Defaults ใน function
:local backupWithDefaults do={
    :local host $1
    :local user $2
    :local pass $3
    :local path $4
    :local keepCount $5
    
    # Apply defaults
    :if ([:len [:tostr $path]] = 0) do={ :set path "/backups/" }
    :if ([:len [:tostr $keepCount]] = 0) do={ :set keepCount 7 }
    
    :put "Backup to $host as $user"
    :put "Path: $path, Keep: $keepCount copies"
}

[$backupWithDefaults "192.168.1.1" "backup" "pass" "" ""]
# Output: Backup to 192.168.1.1 as backup
#         Path: /backups/, Keep: 7 copies
```

---

## 27.4 Parameter Validation

```routeros
# Comprehensive parameter validation
:local validateParams do={
    :local errors {}
    
    # ตรวจสอบ host
    :local host $1
    :if ([:len $host] = 0) do={
        :set errors ($errors , "host is required")
    }
    
    # ตรวจสอบ port
    :local port $2
    :if ([:typeof $port] = "nothing") do={
        :set errors ($errors , "port is required")
    } else={
        :if ($port < 1 || $port > 65535) do={
            :set errors ($errors , "port must be 1-65535")
        }
    }
    
    # ตรวจสอบ action
    :local action $3
    :local validActions {"read"; "write"; "delete"}
    :local actionValid false
    :foreach a in=$validActions do={
        :if ($a = $action) do={ :set actionValid true }
    }
    :if (!$actionValid) do={
        :set errors ($errors , "action must be read/write/delete")
    }
    
    :return $errors
}

:local errors [$validateParams "192.168.1.1" 80 "read"]
:if ([:len $errors] > 0) do={
    :foreach e in=$errors do={
        :put "Validation error: $e"
    }
} else={
    :put "Validation passed!"
}

# ตัวอย่าง error cases
:local errors2 [$validateParams "" 99999 "invalid"]
:foreach e in=$errors2 do={
    :put "Error: $e"
}
# Output:
# Error: host is required
# Error: port must be 1-65535
# Error: action must be read/write/delete
```

---

## 27.5 Scheduler with Parameters

```routeros
# ส่ง parameters ให้ scheduled scripts ผ่าน globals

# Setup config globals
:global BACKUP_HOST "192.168.1.100"
:global BACKUP_INTERVAL "daily"
:global BACKUP_KEEP 7

# Scheduled script ที่ใช้ globals
/system script add name="scheduled-backup" source="
    :global BACKUP_HOST
    :global BACKUP_INTERVAL
    :global BACKUP_KEEP
    
    # ใช้ค่า globals
    :put \"Running \$BACKUP_INTERVAL backup to \$BACKUP_HOST\"
    :put \"Keeping \$BACKUP_KEEP copies\"
    
    # ทำ backup...
"

# Scheduler เรียก script
/system scheduler add \
    name="daily-backup" \
    interval=1d \
    start-time=02:00:00 \
    on-event="/system script run scheduled-backup"

# เปลี่ยน parameters โดยไม่ต้องแก้ script
:set BACKUP_HOST "10.0.0.50"
:set BACKUP_KEEP 14

# Scheduler ที่มี inline parameters
/system scheduler add \
    name="custom-backup" \
    interval=6h \
    on-event="
        :global BACKUP_HOST \"192.168.1.100\"
        :global BACKUP_TYPE \"incremental\"
        /system script run backup-script
    "
```

---

## 27.6 Command Line Execution

```routeros
# รัน script จาก command line พร้อม environment
# (ใน RouterOS ต้องใช้ /system script run)

# สร้าง wrapper script ที่รับ inline params
/system script add name="run-task" source="
    # อ่าน parameters จาก global
    :global TASK_NAME
    :global TASK_PARAMS
    
    :put \"Running task: \$TASK_NAME\"
    :put \"Parameters: \$TASK_PARAMS\"
"

# รันผ่าน API หรือ SSH
# api: /system script run run-task

# เทียบเท่า CLI:
:global TASK_NAME "backup"
:global TASK_PARAMS {"host"="192.168.1.1"; "type"="full"}
/system script run run-task

# Script ที่รับ command line arguments แบบ string
:local parseArgs do={
    :local argStr $1
    :local args {}
    
    # Parse "key=value key=value" format
    :local remaining $argStr
    :while ([:len $remaining] > 0) do={
        :local spacePos [:find $remaining " "]
        :local pair ""
        
        :if ($spacePos != -1) do={
            :set pair [:pick $remaining 0 $spacePos]
            :set remaining [:pick $remaining ($spacePos + 1) [:len $remaining]]
        } else={
            :set pair $remaining
            :set remaining ""
        }
        
        # Parse key=value
        :local eqPos [:find $pair "="]
        :if ($eqPos != -1) do={
            :local key [:pick $pair 0 $eqPos]
            :local val [:pick $pair ($eqPos + 1) [:len $pair]]
            :set args ($args , {$key; $val})
        }
    }
    :return $args
}

:local args [$parseArgs "host=192.168.1.1 user=admin action=backup"]
:put $args
```

---

## 27.7 Parameter Templates

```routeros
# Template pattern สำหรับ reusable scripts

# Email template
:local emailTemplate do={
    :local to $1
    :local subject $2
    :local template $3
    :local vars $4
    
    # Replace placeholders ใน template
    :local body $template
    :foreach var in=$vars do={
        :local key ($var->0)
        :local val ($var->1)
        # Replace {key} ด้วย val
        :local placeholder ("{" . $key . "}")
        :while ([:find $body $placeholder] != -1) do={
            :local pos [:find $body $placeholder]
            :set body ([:pick $body 0 $pos] . $val . \
                      [:pick $body ($pos + [:len $placeholder]) [:len $body]])
        }
    }
    
    /tool e-mail send to=$to subject=$subject body=$body
}

# ใช้ template
:local alertTemplate "
Dear Admin,

Router {ROUTER_NAME} has detected an issue:
Issue: {ISSUE}
Severity: {SEVERITY}
Time: {TIME}

Please investigate immediately.
"

[$emailTemplate "admin@example.com" "Router Alert" $alertTemplate {
    {"ROUTER_NAME"; [/system identity get name]};
    {"ISSUE"; "Interface down"};
    {"SEVERITY"; "HIGH"};
    {"TIME"; [/system clock get time]}
}]

# Config template
:local configTemplate do={
    :local ifaceName $1
    :local ip $2
    :local gw $3
    
    :local config ("# Interface: " . $ifaceName . "\r\n")
    :set config ($config . "/ip address add address=" . $ip . " interface=" . $ifaceName . "\r\n")
    :set config ($config . "/ip route add gateway=" . $gw . "\r\n")
    :return $config
}

:put [$configTemplate "ether1" "192.168.1.1/24" "192.168.1.254"]
```

---

## 27.8 Common Parameter Patterns

### Pattern 1: Configuration Object

```routeros
# ส่ง configuration เป็น single parameter (array/dict)
:local deployVPN do={
    :local config $1
    
    :local server ($config->"server")
    :local user ($config->"user")
    :local pass ($config->"password")
    :local type ($config->"type")
    
    :put "Deploying $type VPN to $server as $user"
    
    :if ($type = "l2tp") do={
        /interface l2tp-client add \
            name="vpn-l2tp" \
            connect-to=$server \
            user=$user \
            password=$pass
    }
    :if ($type = "pptp") do={
        /interface pptp-client add \
            name="vpn-pptp" \
            connect-to=$server \
            user=$user \
            password=$pass
    }
}

:local vpnConfig {
    "server"="vpn.company.com";
    "user"="vpnuser";
    "password"="VPNpass123";
    "type"="l2tp"
}

[$deployVPN $vpnConfig]
```

### Pattern 2: Options with Defaults

```routeros
# Function ที่มี optional parameters
:local createQueue do={
    # Required
    :local name $1
    :local target $2
    
    # Optional with defaults
    :local maxLimit $3
    :local burstLimit $4
    :local priority $5
    
    :if ([:len [:tostr $maxLimit]] = 0) do={ :set maxLimit "10M/10M" }
    :if ([:len [:tostr $burstLimit]] = 0) do={ :set burstLimit "20M/20M" }
    :if ([:len [:tostr $priority]] = 0) do={ :set priority 8 }
    
    /queue simple add \
        name=$name \
        target=$target \
        max-limit=$maxLimit \
        burst-limit=$burstLimit \
        priority=$priority
    
    :put "Queue created: $name for $target ($maxLimit)"
}

# เรียกใช้ - required only
[$createQueue "client-001" "192.168.1.10"]

# เรียกใช้ - all params
[$createQueue "vip-client" "192.168.1.20" "50M/20M" "100M/40M" 1]
```

### Pattern 3: Chained Configuration

```routeros
# Scripts ที่รัน sequential พร้อม shared state
:global chainConfig {}
:global chainResults {}

:local initChain do={
    :global chainConfig
    :global chainResults
    :set chainConfig $1
    :set chainResults {}
}

:local addChainResult do={
    :global chainResults
    :local step $1
    :local result $2
    :set chainResults ($chainResults , {$step; $result})
}

:local runChain do={
    :global chainConfig
    :global chainResults
    :global addChainResult
    
    :local steps ($chainConfig->"steps")
    :foreach step in=$steps do={
        :put "Running step: $step"
        :do {
            /system script run $step
            [$addChainResult $step "success"]
        } on-error={
            [$addChainResult $step "failed"]
            :if (($chainConfig->"stopOnError") = true) do={
                :put "Chain stopped at: $step"
                :return false
            }
        }
    }
    :return true
}

# ใช้งาน
[$initChain {
    "steps"={"step1-validate"; "step2-backup"; "step3-deploy"; "step4-verify"};
    "stopOnError"=true
}]

[$runChain]

:put "Chain results:"
:foreach r in=$chainResults do={
    :put ("  " . ($r->0) . ": " . ($r->1))
}
```

---

## Lab 27: Parameterized Backup Script

### โจทย์
สร้าง backup script ที่ parameterized อย่างสมบูรณ์:
- รับ parameters หลายรูปแบบ
- มี defaults
- มี validation
- รองรับ multiple backup destinations

### Solution

```routeros
# Parameterized Backup Script Lab
# =================================

# Script ที่รับ parameters ครบครัน
/system script add name="flex-backup" source="
    # --- Parameters (via globals) ---
    :global FLEX_DEST_TYPE    # ftp/tftp/email/local
    :global FLEX_DEST_HOST    # สำหรับ ftp/tftp
    :global FLEX_DEST_USER    # สำหรับ ftp
    :global FLEX_DEST_PASS    # สำหรับ ftp
    :global FLEX_DEST_PATH    # path บน server
    :global FLEX_EMAIL        # สำหรับ email
    :global FLEX_BACKUP_TYPE  # full/config/security
    :global FLEX_KEEP_COPIES  # จำนวน copies ที่เก็บ
    :global FLEX_ENCRYPT_PASS # password สำหรับ encrypt (optional)
    
    # --- Defaults ---
    :if ([:typeof \$FLEX_DEST_TYPE] = \"nothing\") do={ :set FLEX_DEST_TYPE \"local\" }
    :if ([:typeof \$FLEX_KEEP_COPIES] = \"nothing\") do={ :set FLEX_KEEP_COPIES 7 }
    :if ([:typeof \$FLEX_BACKUP_TYPE] = \"nothing\") do={ :set FLEX_BACKUP_TYPE \"full\" }
    :if ([:typeof \$FLEX_ENCRYPT_PASS] = \"nothing\") do={ :set FLEX_ENCRYPT_PASS \"\" }
    
    # --- Validation ---
    :local validDestTypes {\"ftp\"; \"email\"; \"local\"}
    :local destValid false
    :foreach dt in=\$validDestTypes do={
        :if (\$dt = \$FLEX_DEST_TYPE) do={ :set destValid true }
    }
    
    :if (!\$destValid) do={
        :put \"Invalid destination type: \$FLEX_DEST_TYPE\"
        :error \"Invalid destination type\"
    }
    
    :if (\$FLEX_DEST_TYPE = \"ftp\") do={
        :if ([:len \$FLEX_DEST_HOST] = 0) do={
            :error \"FTP host required for type=ftp\"
        }
    }
    
    :if (\$FLEX_DEST_TYPE = \"email\") do={
        :if ([:len \$FLEX_EMAIL] = 0) do={
            :error \"Email required for type=email\"
        }
    }
    
    # --- Create Backup ---
    :local hostname [/system identity get name]
    :local date [/system clock get date]
    :local dateStr \"\"
    :for i from=0 to=([:len [:tostr \$date]] - 1) do={
        :local c [:pick [:tostr \$date] \$i (\$i + 1)]
        :if (\$c = \"/\") do={ :set dateStr (\$dateStr . \"-\") } else={ :set dateStr (\$dateStr . \$c) }
    }
    :local baseName (\$hostname . \"-\" . \$dateStr)
    
    :put \"Creating backup: type=\$FLEX_BACKUP_TYPE dest=\$FLEX_DEST_TYPE\"
    
    # Create backup files based on type
    :if (\$FLEX_BACKUP_TYPE = \"full\" || \$FLEX_BACKUP_TYPE = \"backup\") do={
        :if ([:len \$FLEX_ENCRYPT_PASS] > 0) do={
            /system backup save name=\$baseName password=\$FLEX_ENCRYPT_PASS
        } else={
            /system backup save name=\$baseName
        }
        :delay 2s
    }
    
    :if (\$FLEX_BACKUP_TYPE = \"full\" || \$FLEX_BACKUP_TYPE = \"config\") do={
        /export file=\$baseName
        :delay 1s
    }
    
    # --- Upload to Destination ---
    :if (\$FLEX_DEST_TYPE = \"ftp\") do={
        :local destPath \$FLEX_DEST_PATH
        :if ([:len \$destPath] = 0) do={ :set destPath \"/\" }
        
        :do {
            /tool fetch address=\$FLEX_DEST_HOST \
                src-path=(\$baseName . \".rsc\") \
                user=\$FLEX_DEST_USER password=\$FLEX_DEST_PASS \
                dst-path=(\$destPath . \$baseName . \".rsc\") \
                upload=yes
            :put \"FTP upload OK\"
        } on-error={
            :put \"FTP upload FAILED\"
        }
    }
    
    :if (\$FLEX_DEST_TYPE = \"email\") do={
        :do {
            /tool e-mail send \
                to=\$FLEX_EMAIL \
                subject=(\"Backup from \" . \$hostname . \" \" . \$date) \
                body=(\"Router backup completed\\r\\nFile: \" . \$baseName . \".rsc\")
            :put \"Email sent to \$FLEX_EMAIL\"
        } on-error={
            :put \"Email sending FAILED\"
        }
    }
    
    :if (\$FLEX_DEST_TYPE = \"local\") do={
        :put \"Backup saved locally: \$baseName\"
    }
    
    # --- Cleanup ---
    :local backups [/file find where name~(\$hostname . \"-\") and name~\".backup\"]
    :local count [:len \$backups]
    :if (\$count > \$FLEX_KEEP_COPIES) do={
        :local toDelete [:pick \$backups 0 (\$count - \$FLEX_KEEP_COPIES)]
        :foreach f in=\$toDelete do={
            /file remove \$f
        }
        :put (\"Cleaned up \" . [:len \$toDelete] . \" old backups\")
    }
    
    :put \"=== Flex Backup Complete ===\"
" comment="Parameterized backup script"

# --- Usage Examples ---

# Example 1: Local backup
:global FLEX_DEST_TYPE "local"
:global FLEX_BACKUP_TYPE "full"
:global FLEX_KEEP_COPIES 5
/system script run flex-backup

# Example 2: FTP backup
:global FLEX_DEST_TYPE "ftp"
:global FLEX_DEST_HOST "192.168.1.100"
:global FLEX_DEST_USER "backup"
:global FLEX_DEST_PASS "BackupPass"
:global FLEX_DEST_PATH "/daily-backups/"
:global FLEX_BACKUP_TYPE "config"
:global FLEX_KEEP_COPIES 7
/system script run flex-backup

# Example 3: Email backup
:global FLEX_DEST_TYPE "email"
:global FLEX_EMAIL "admin@company.com"
:global FLEX_BACKUP_TYPE "config"
/system script run flex-backup

# --- Schedulers for different backup types ---

# Daily config backup
/system scheduler add \
    name="daily-config-backup" \
    interval=1d \
    start-time=02:00:00 \
    on-event="
        :global FLEX_DEST_TYPE \"ftp\"
        :global FLEX_DEST_HOST \"192.168.1.100\"
        :global FLEX_DEST_USER \"backup\"
        :global FLEX_DEST_PASS \"BackupPass\"
        :global FLEX_DEST_PATH \"/daily/\"
        :global FLEX_BACKUP_TYPE \"config\"
        :global FLEX_KEEP_COPIES 7
        /system script run flex-backup
    "

# Weekly full backup
/system scheduler add \
    name="weekly-full-backup" \
    interval=7d \
    start-time=00:00:00 \
    on-event="
        :global FLEX_DEST_TYPE \"ftp\"
        :global FLEX_DEST_HOST \"192.168.1.100\"
        :global FLEX_DEST_USER \"backup\"
        :global FLEX_DEST_PASS \"BackupPass\"
        :global FLEX_DEST_PATH \"/weekly/\"
        :global FLEX_BACKUP_TYPE \"full\"
        :global FLEX_ENCRYPT_PASS \"WeeklyEncrypt\"
        :global FLEX_KEEP_COPIES 4
        /system script run flex-backup
    "
```

---

## Summary ของ Part 27

| Method | ใช้เมื่อ | ข้อดี |
|--------|---------|------|
| Global variables | Scripts ที่รัน via scheduler | Simple, persistent |
| Function params ($1,$2) | Functions ใน script | Clean, scoped |
| Config objects | หลาย parameters | Organized |
| Templates | Format strings | Reusable |

> **Best Practice:** ตั้งชื่อ parameter globals ด้วย prefix ของ script เช่น `BACKUP_HOST`, `MONITOR_INTERVAL`

> **Tip:** ใช้ default values เสมอเพื่อให้ script ทำงานได้แม้ไม่ set parameters

> **Warning:** Parameters ผ่าน globals มีความเสี่ยง name collision ระหว่าง scripts ต่างๆ

---

[← Part 26: Error Handling](part-026-error-handling.md) | [Part 28: Interface Scripts →](part-028-interface-scripts.md)
