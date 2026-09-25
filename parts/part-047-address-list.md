# Part 47: Address List Management

## บทนำ

Address List เป็น feature ที่ทรงพลังใน MikroTik Firewall ช่วยให้จัดกลุ่ม IP addresses และนำไปใช้ใน rules ได้สะดวก การจัดการ address list ด้วย scripting ทำให้สามารถ automate การ update, sync, และ maintain lists ได้อย่างมีประสิทธิภาพ

---

## 47.1 Address List Fundamentals

### ประเภทของ Address List

| ประเภท | วิธีสร้าง | Persistence | Timeout |
|--------|----------|-------------|---------|
| Static | Manual/Script | ถาวร | ไม่มี |
| Dynamic | Firewall rule | ชั่วคราว | มี |

### การสร้าง Address List พื้นฐาน

```routeros
# เพิ่ม IP เดียว
/ip firewall address-list
add list=MY-LIST address=192.168.1.100 \
    comment="Server 1"

# เพิ่ม subnet
add list=MY-LIST address=10.0.0.0/8 \
    comment="Internal network"

# เพิ่ม IP range
add list=MY-LIST address=172.16.1.1-172.16.1.10 \
    comment="IP range"

# เพิ่มพร้อม timeout
add list=TEMP-LIST address=203.0.113.50 \
    timeout=2h comment="Temporary access"

# ดู address list
/ip firewall address-list print where list=MY-LIST
```

### การใช้งานใน Firewall Rules

```routeros
/ip firewall filter

# Allow traffic from MY-LIST
add chain=input src-address-list=MY-LIST \
    action=accept comment="Allow from MY-LIST"

# Block traffic from BLOCKED-LIST
add chain=forward dst-address-list=BAD-SERVERS \
    action=drop comment="Block bad servers"

# Redirect to different route
add chain=prerouting src-address-list=PREMIUM-USERS \
    action=mark-routing new-routing-mark=PREMIUM-ROUTE \
    passthrough=yes
```

---

## 47.2 Dynamic vs Static Lists

### Static List Management

```routeros
# เพิ่ม batch ด้วย array
:local staticIPs {
    "203.0.113.10";"203.0.113.11";"203.0.113.12"
}

:foreach ip in=$staticIPs do={
    /ip firewall address-list add \
        list=OFFICE-IPS \
        address=$ip \
        comment="Office IP"
}
```

### Dynamic List Properties

```routeros
# ดู dynamic entries
/ip firewall address-list print dynamic

# ดู static entries
/ip firewall address-list print static

# ดู entries ที่ใกล้หมดอายุ (dynamic)
/ip firewall address-list print where timeout

# แปลง dynamic เป็น static
:foreach entry in=[/ip firewall address-list find dynamic=yes list=MY-LIST] do={
    :local addr [/ip firewall address-list get $entry address]
    # ลบ dynamic entry
    /ip firewall address-list remove $entry
    # สร้าง static entry ใหม่
    /ip firewall address-list add list=MY-LIST address=$addr \
        comment="Converted from dynamic"
}
```

---

## 47.3 Address List Management Scripts

### Add/Remove Script

```routeros
/system script
add name="al-add" source={
    # รับ parameters
    :local list $1
    :local address $2
    :local comment $3
    :local timeout $4
    
    :if ([:len $list] = 0 || [:len $address] = 0) do={
        :put "Usage: al-add list=<name> address=<ip> [comment=<text>] [timeout=<time>]"
        :error "Missing required parameters"
    }
    
    # ตรวจสอบ duplicate
    :local existing [/ip firewall address-list find \
        list=$list address=$address]
    
    :if ([:len $existing] > 0) do={
        :put ("Already exists: " . $address . " in " . $list)
    } else={
        :if ([:len $timeout] > 0) do={
            /ip firewall address-list add \
                list=$list address=$address \
                timeout=$timeout \
                comment=$comment
        } else={
            /ip firewall address-list add \
                list=$list address=$address \
                comment=$comment
        }
        :put ("Added " . $address . " to " . $list)
        :log info ("AL: Added " . $address . " to " . $list)
    }
}

/system script
add name="al-remove" source={
    :local list $1
    :local address $2
    
    :local entry [/ip firewall address-list find \
        list=$list address=$address]
    
    :if ([:len $entry] > 0) do={
        /ip firewall address-list remove $entry
        :put ("Removed " . $address . " from " . $list)
        :log info ("AL: Removed " . $address . " from " . $list)
    } else={
        :put ("Not found: " . $address . " in " . $list)
    }
}
```

### List Copy Script

```routeros
/system script
add name="al-copy" source={
    :local srcList $1
    :local dstList $2
    :local copyCount 0
    
    :foreach entry in=[/ip firewall address-list find list=$srcList] do={
        :local addr [/ip firewall address-list get $entry address]
        :local comment [/ip firewall address-list get $entry comment]
        
        # ตรวจสอบ duplicate
        :local existing [/ip firewall address-list find \
            list=$dstList address=$addr]
        
        :if ([:len $existing] = 0) do={
            /ip firewall address-list add \
                list=$dstList \
                address=$addr \
                comment=("Copied from " . $srcList . ": " . $comment)
            :set copyCount ($copyCount + 1)
        }
    }
    
    :put ("Copied " . $copyCount . " entries from " . $srcList . " to " . $dstList)
}
```

---

## 47.4 Syncing Address Lists

### Sync ระหว่าง Routers

```routeros
/system script
add name="al-sync-push" source={
    # Push address list ไปยัง remote router
    :local remoteIP "192.168.1.2"
    :local remoteUser "admin"
    :local remotePass "password"
    :local listName "SHARED-BLOCKLIST"
    
    # สร้าง script สำหรับ remote execution
    :local remoteScript "/ip firewall address-list remove [find list=" . \
        $listName . "];"
    
    :foreach entry in=[/ip firewall address-list find list=$listName] do={
        :local addr [/ip firewall address-list get $entry address]
        :local comment [/ip firewall address-list get $entry comment]
        
        :set remoteScript ($remoteScript . \
            "/ip firewall address-list add list=" . $listName . \
            " address=" . $addr . \
            " comment=\"" . $comment . "\";")
    }
    
    # Execute ผ่าน SSH (ต้องมี ssh-exec permission)
    /system ssh-exec address=$remoteIP \
        user=$remoteUser \
        password=$remotePass \
        command=$remoteScript
    
    :log info ("AL: Synced " . $listName . " to " . $remoteIP)
}
```

### Script ใช้ API สำหรับ Sync

```routeros
/system script
add name="al-export-to-api" source={
    :local listName "EXPORT-LIST"
    :local apiEndpoint "http://management.example.com/api/address-list"
    :local exportData ""
    
    # สร้าง CSV data
    :set exportData "list,address,comment\n"
    
    :foreach entry in=[/ip firewall address-list find list=$listName] do={
        :local list [/ip firewall address-list get $entry list]
        :local addr [/ip firewall address-list get $entry address]
        :local comment [/ip firewall address-list get $entry comment]
        
        :set exportData ($exportData . $list . "," . $addr . \
            "," . $comment . "\n")
    }
    
    # Send ไปยัง API
    /tool fetch url=$apiEndpoint \
        http-method=post \
        http-data=$exportData \
        output=none
    
    :log info "AL: Exported to API"
}
```

---

## 47.5 Importing from External Sources

### Import จาก File

```routeros
/system script
add name="al-import-from-file" source={
    :local listName $1
    :local fileName $2
    :local importCount 0
    :local skipCount 0
    
    :if ([:len $listName] = 0) do={
        :set listName "IMPORTED-LIST"
    }
    :if ([:len $fileName] = 0) do={
        :set fileName "import.txt"
    }
    
    # ตรวจสอบว่า file มีอยู่
    :if ([/file find name=$fileName] = "") do={
        :log error ("File not found: " . $fileName)
        :error "File not found"
    }
    
    # อ่าน file
    :local content [/file get [/file find name=$fileName] contents]
    
    # แยกแต่ละบรรทัด
    :local lines [:toarray $content]
    
    :foreach line in=$lines do={
        # ข้าม comments และ empty lines
        :if ([:pick $line 0 1] != "#" && [:len $line] > 4) do={
            # ดึง IP (assume first word)
            :local ip $line
            :local spacePos [:find $line " "]
            :if ($spacePos > 0) do={
                :set ip [:pick $line 0 $spacePos]
            }
            
            :do {
                /ip firewall address-list add \
                    list=$listName \
                    address=$ip \
                    comment="Imported"
                :set importCount ($importCount + 1)
            } on-error={
                :set skipCount ($skipCount + 1)
                :log debug ("AL Import: Skipped invalid entry: " . $ip)
            }
        }
    }
    
    :log info ("AL: Imported " . $importCount . " entries, " . \
        $skipCount . " skipped to " . $listName)
    :put ("Imported: " . $importCount . ", Skipped: " . $skipCount)
}
```

### Import จาก URL (HTTP)

```routeros
/system script
add name="al-import-from-url" source={
    :local listName "THREAT-LIST"
    :local url "https://raw.githubusercontent.com/example/blocklist/main/ips.txt"
    :local tmpFile "threat-import.txt"
    
    # Download file
    :log info "Downloading threat list..."
    :do {
        /tool fetch url=$url dst-path=$tmpFile
    } on-error={
        :log error "Failed to download threat list"
        :error "Download failed"
    }
    
    # ล้าง list เก่า
    /ip firewall address-list remove [find list=$listName]
    
    # Import
    /system script run al-import-from-file \
        list=$listName file=$tmpFile
    
    # Cleanup
    /file remove [find name=$tmpFile]
    
    :log info ("AL: Threat list updated")
}

# Schedule ทุก 6 ชั่วโมง
/system scheduler
add name="update-threat-list" interval=6h \
    on-event="/system script run al-import-from-url"
```

---

## 47.6 Address List Monitoring

### Monitor Script

```routeros
/system script
add name="al-monitor" source={
    :put "=== ADDRESS LIST MONITOR ==="
    :put ""
    
    # รายการ lists ทั้งหมด
    :local allLists {}
    :foreach entry in=[/ip firewall address-list find] do={
        :local list [/ip firewall address-list get $entry list]
        # เพิ่มลงใน unique set
        :if ([:find $allLists $list] < 0) do={
            :set allLists ($allLists, $list)
        }
    }
    
    # แสดงแต่ละ list
    :foreach listName in=$allLists do={
        :local entries [/ip firewall address-list find list=$listName]
        :local staticCount 0
        :local dynamicCount 0
        
        :foreach e in=$entries do={
            :if ([/ip firewall address-list get $e dynamic]) do={
                :set dynamicCount ($dynamicCount + 1)
            } else={
                :set staticCount ($staticCount + 1)
            }
        }
        
        :put ($listName . ": " . [:len $entries] . " entries " . \
            "(static=" . $staticCount . ", dynamic=" . $dynamicCount . ")")
    }
    
    :put ""
    :put ("Total lists: " . [:len $allLists])
}
```

### Alert on List Size Change

```routeros
/system script
add name="al-size-alert" source={
    :local monitoredLists {
        "BLOCKED-IPS";"THREAT-LIST";"WHITELIST"
    }
    :local alertThreshold 1000
    
    :foreach listName in=$monitoredLists do={
        :local entries [/ip firewall address-list find list=$listName]
        :local count [:len $entries]
        
        :if ($count > $alertThreshold) do={
            :log warning ("AL Alert: " . $listName . \
                " has " . $count . " entries (threshold: " . $alertThreshold . ")")
            
            /tool e-mail send \
                to="admin@example.com" \
                subject=("AL Alert: " . $listName . " size warning") \
                body=($listName . " has " . $count . " entries")
        }
        
        :log info ("AL Monitor: " . $listName . " = " . $count . " entries")
    }
}
```

---

## 47.7 Bulk Operations

### Batch Add ด้วย Array

```routeros
/system script
add name="al-bulk-add" source={
    :local listName "BATCH-TEST"
    :local addresses {
        "10.0.0.1";"10.0.0.2";"10.0.0.3";"10.0.0.4";"10.0.0.5";
        "10.0.1.1";"10.0.1.2";"10.0.1.3";"10.0.1.4";"10.0.1.5"
    }
    
    :local added 0
    :local skipped 0
    
    :foreach addr in=$addresses do={
        :local existing [/ip firewall address-list find \
            list=$listName address=$addr]
        
        :if ([:len $existing] = 0) do={
            /ip firewall address-list add \
                list=$listName address=$addr \
                comment="Bulk import"
            :set added ($added + 1)
        } else={
            :set skipped ($skipped + 1)
        }
    }
    
    :put ("Bulk add complete: " . $added . " added, " . $skipped . " skipped")
}
```

### Bulk Remove

```routeros
/system script
add name="al-bulk-remove" source={
    :local listName "CLEANUP-LIST"
    :local removeOlderThan 30  # ลบ entries ที่เก่ากว่า 30 วัน
    
    # ลบ dynamic entries ทั้งหมด
    /ip firewall address-list remove [find list=$listName dynamic=yes]
    
    # นับ entries ที่เหลือ
    :local remaining [/ip firewall address-list find list=$listName]
    :log info ("AL: Cleanup complete, " . [:len $remaining] . " static entries remain")
}
```

---

## 47.8 Address List Backup/Restore

### Backup Script

```routeros
/system script
add name="al-backup" source={
    :local backupFile ("al-backup-" . \
        [:pick [/system clock get date] 0 4] . \
        [:pick [/system clock get date] 5 7] . \
        [:pick [/system clock get date] 8 10] . \
        ".rsc")
    
    :local content "# Address List Backup\n"
    :set content ($content . "# Generated: " . \
        [/system clock get date] . " " . [/system clock get time] . "\n\n")
    
    :foreach entry in=[/ip firewall address-list find static] do={
        :local list [/ip firewall address-list get $entry list]
        :local addr [/ip firewall address-list get $entry address]
        :local comment [/ip firewall address-list get $entry comment]
        
        :set content ($content . \
            "/ip firewall address-list add list=" . $list . \
            " address=" . $addr . \
            " comment=\"" . $comment . "\"\n")
    }
    
    /file add name=$backupFile contents=$content
    :log info ("AL: Backup saved to " . $backupFile)
    :put ("Backup saved: " . $backupFile)
}
```

### Restore Script

```routeros
/system script
add name="al-restore" source={
    :local backupFile $1
    
    :if ([:len $backupFile] = 0) do={
        :put "Usage: al-restore file=<backup-file.rsc>"
        :error "No file specified"
    }
    
    :if ([/file find name=$backupFile] = "") do={
        :log error ("Backup file not found: " . $backupFile)
        :error "File not found"
    }
    
    :put ("Restoring from " . $backupFile . "...")
    
    # Import the backup script
    /import file-name=$backupFile
    
    :log info ("AL: Restored from " . $backupFile)
    :put "Restore complete!"
}
```

---

## 47.9 Integration with Firewall

### Policy-based Routing with Address Lists

```routeros
# Premium users ไปทาง fast WAN
/ip firewall mangle
add chain=prerouting \
    src-address-list=PREMIUM-USERS \
    action=mark-routing \
    new-routing-mark=PREMIUM-ROUTE \
    passthrough=yes

# Blocked users redirect to block page
/ip firewall nat
add chain=dstnat \
    src-address-list=BLOCKED-USERS \
    protocol=tcp dst-port=80 \
    action=dst-nat \
    to-addresses=192.168.1.200 \
    to-ports=8080

# VIP users exempt from QoS
/ip firewall mangle
add chain=forward \
    src-address-list=VIP-USERS \
    action=mark-packet \
    new-packet-mark=VIP-TRAFFIC \
    passthrough=yes
```

---

## 47.10 Lab: External Threat List Import

### Lab Overview

| รายการ | รายละเอียด |
|--------|------------|
| Objective | Import threat IPs จาก external source แล้ว block |
| Source | Text file กับ IP addresses |
| Update | Automated ทุก 6 ชั่วโมง |
| Action | Auto-block imported IPs |

### Step 1: สร้าง Sample Threat List File

```
# สร้าง test file ด้วย IPs สำหรับ lab
/file add name="threat-sample.txt" contents="# Test Threat List
203.0.113.100
203.0.113.101
203.0.113.102
198.51.100.50
198.51.100.51
"
```

### Step 2: Import และ Apply Firewall Rule

```routeros
/system script
add name="lab-threat-import" source={
    :local listName "LAB-THREAT"
    :local fileName "threat-sample.txt"
    
    # ล้าง list เก่า
    /ip firewall address-list remove [find list=$listName]
    
    :put "Importing threat list..."
    
    # Import
    /system script run al-import-from-file \
        list=$listName file=$fileName
    
    # สร้าง firewall rule (ถ้ายังไม่มี)
    :local existing [/ip firewall filter find comment="LAB-THREAT-BLOCK"]
    :if ([:len $existing] = 0) do={
        /ip firewall filter add \
            chain=input \
            src-address-list=LAB-THREAT \
            action=drop \
            log=yes log-prefix="THREAT-BLOCKED:" \
            comment="LAB-THREAT-BLOCK" \
            place-before=0
        :put "Firewall rule created"
    }
    
    :local count [/ip firewall address-list find list=$listName]
    :put ("Imported " . [:len $count] . " threat IPs")
    :put "Firewall rule active - these IPs are now blocked"
}

/system script run lab-threat-import
```

### Step 3: ทดสอบและ Monitor

```routeros
# ดู imported list
/ip firewall address-list print where list=LAB-THREAT

# ดู firewall rule
/ip firewall filter print where comment="LAB-THREAT-BLOCK"

# Monitor logs
/log print where message~"THREAT-BLOCKED"

# ดู statistics
/ip firewall filter print stats where comment="LAB-THREAT-BLOCK"
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Fundamentals | Static vs Dynamic lists |
| Management Scripts | Add, remove, copy operations |
| Syncing | Router-to-router sync |
| Import | File และ URL import |
| Monitoring | Size alerts, statistics |
| Bulk Operations | Batch add/remove |
| Backup/Restore | Export/import scripts |
| Firewall Integration | Policy routing, NAT, marks |

---

[← Part 46: Dynamic Firewall](part-046-dynamic-firewall.md) | [Part 48: Queue Management →](part-048-queue-management.md)
