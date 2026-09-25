# Part 23: File Operations ใน RouterOS

## บทนำ

RouterOS มี file system ที่ช่วยให้เราจัดเก็บข้อมูล, scripts, backups และไฟล์ต่างๆ ได้ การเข้าใจ file operations จะช่วยให้สร้าง automation scripts ที่ทำงานกับไฟล์ได้อย่างมีประสิทธิภาพ

---

## 23.1 RouterOS File System

### โครงสร้าง File System

```
RouterOS File System:
/
├── flash/          (primary storage)
│   ├── disk/       (USB disk if connected)
│   └── rtos/       (system files - read only)
├── disk1/          (external storage)
└── tmp/            (temporary files)
```

### ดู File System

```routeros
# แสดงไฟล์ทั้งหมด
/file print

# แสดงเฉพาะไฟล์บางประเภท
/file print where type=".backup"
/file print where type=".rsc"
/file print where name~"config"

# รายละเอียดไฟล์
/file print detail

# ขนาดไฟล์แต่ละไฟล์
/file print where size>0

# แสดงเฉพาะ column ที่ต้องการ
/file print columns=name,size,creation-time
```

### File Properties

| Property | คำอธิบาย |
|----------|---------|
| name | ชื่อไฟล์ |
| type | ประเภทไฟล์ (.backup, .rsc, .jpg, etc.) |
| size | ขนาดเป็น bytes |
| creation-time | วันเวลาที่สร้าง |

---

## 23.2 Reading Files

### อ่านไฟล์ด้วย Script

```routeros
# อ่านไฟล์ .rsc (script file)
/import file-name=myscript.rsc

# อ่านเนื้อหาไฟล์ด้วย :read (ไม่มีใน RouterOS โดยตรง)
# ใช้วิธีผ่าน file operations

# ตรวจสอบว่าไฟล์มีอยู่หรือไม่
:if ([:len [/file find name="config.txt"]] > 0) do={
    :put "File exists"
} else={
    :put "File not found"
}

# อ่าน config ที่ export ไว้
/export file=current-config
# จะสร้างไฟล์ current-config.rsc

# อ่าน file content (RouterOS 6.44+)
:local content [/file get [/file find name="config.txt"] contents]
:put $content
```

### ดึง File Contents

```routeros
# ต้องใช้ /file get contents
:local fileName "mydata.txt"
:local fileId [/file find name=$fileName]

:if ([:len $fileId] > 0) do={
    :local content [/file get $fileId contents]
    :put "File content:"
    :put $content
} else={
    :put "File $fileName not found"
}

# อ่านไฟล์และ parse line by line
:local parseFile do={
    :local fileName $1
    :local fileId [/file find name=$fileName]
    :if ([:len $fileId] = 0) do={
        :put "File not found"
        :return
    }
    
    :local content [/file get $fileId contents]
    :local lines [:toarray $content]
    
    :foreach line in=$lines do={
        :put "Line: $line"
    }
}
```

---

## 23.3 Writing Files

### สร้างและเขียนไฟล์

```routeros
# สร้างไฟล์ด้วย /tool fetch (เขียน content)
# Method 1: ผ่าน tool fetch
/tool fetch url="http://example.com/file.txt" dst-path="downloaded.txt"

# Method 2: สร้างไฟล์จาก script output
# RouterOS script สามารถเขียนผ่าน /file set contents

# เขียน content ลงไฟล์
:local fileName "myfile.txt"
:local content "Hello, RouterOS!\r\nThis is line 2.\r\nLine 3."

# ลบไฟล์เก่าถ้ามี
/file remove [find name=$fileName]

# สร้างไฟล์ใหม่ด้วย fetch
/tool fetch url="data:," dst-path=$fileName

# เขียน content
/file set [find name=$fileName] contents=$content

:put "File written successfully"

# Append ข้อมูลลงไฟล์
:local appendToFile do={
    :local fileName $1
    :local newContent $2
    
    :local fileId [/file find name=$fileName]
    :local existingContent ""
    
    :if ([:len $fileId] > 0) do={
        :set existingContent [/file get $fileId contents]
    }
    
    :local fullContent ($existingContent . $newContent . "\r\n")
    
    :if ([:len $fileId] > 0) do={
        /file set $fileId contents=$fullContent
    } else={
        /tool fetch url="data:," dst-path=$fileName
        /file set [find name=$fileName] contents=$fullContent
    }
}

[$appendToFile "log.txt" "2025-01-01 System started"]
[$appendToFile "log.txt" "2025-01-01 Interface up"]
```

### Write Config Backup

```routeros
# Export current config
/export file=config-backup

# Export specific section
/ip address export file=ip-addresses
/ip firewall filter export file=firewall-rules
/ip route export file=routes

# Export ด้วย timestamp
:local timestamp [/system clock get date]
:local filename ("backup-" . $timestamp)
/export file=$filename
:put "Config exported to $filename.rsc"

# Export compact
/export compact file=compact-backup
```

---

## 23.4 File Operations

### ลบไฟล์

```routeros
# ลบไฟล์เดี่ยว
/file remove [find name="old-backup.rsc"]

# ลบหลายไฟล์
:local filesToDelete {"old1.txt"; "old2.txt"; "temp.rsc"}
:foreach fileName in=$filesToDelete do={
    :local fileId [/file find name=$fileName]
    :if ([:len $fileId] > 0) do={
        /file remove $fileId
        :put "Deleted: $fileName"
    }
}

# ลบไฟล์ที่เก่ากว่า N วัน
:local deleteOldFiles do={
    :local daysOld $1
    :local cutoffTime ([/system clock get time] - ($daysOld * 86400))
    
    :foreach file in=[/file find] do={
        :local creationTime [/file get $file creation-time]
        # ใช้ date comparison
        :put "Checking: [/file get $file name]"
    }
}

# ลบ backup files เก่า (เก็บไว้แค่ล่าสุด 5 ไฟล์)
:local cleanOldBackups do={
    :local backupFiles [/file find where name~".backup"]
    :local count [:len $backupFiles]
    :local keepCount 5
    
    :if ($count > $keepCount) do={
        :local deleteCount ($count - $keepCount)
        :local oldFiles [:pick $backupFiles 0 $deleteCount]
        :foreach f in=$oldFiles do={
            :local fname [/file get $f name]
            /file remove $f
            :put "Deleted old backup: $fname"
        }
    }
}
```

### Copy และ Rename ไฟล์

```routeros
# RouterOS ไม่มี direct copy/rename ต้อง download และ upload
# หรือใช้ FTP

# Rename ผ่าน FTP (local)
# ต้องมี FTP server บน router

# Copy backup file
:local copyFile do={
    :local srcFile $1
    :local dstFile $2
    
    # อ่าน source
    :local srcId [/file find name=$srcFile]
    :if ([:len $srcId] = 0) do={
        :put "Source file not found: $srcFile"
        :return
    }
    
    :local content [/file get $srcId contents]
    
    # เขียน destination
    /file remove [find name=$dstFile]
    /tool fetch url="data:," dst-path=$dstFile
    /file set [find name=$dstFile] contents=$content
    
    :put "Copied $srcFile to $dstFile"
}
```

---

## 23.5 FTP Operations

### Upload ไฟล์ผ่าน FTP

```routeros
# Upload ไฟล์ไปยัง FTP server
/tool fetch \
    address=192.168.1.100 \
    src-path=backup.rsc \
    user=ftpuser \
    password=ftppass \
    dst-path=/backups/router-backup.rsc \
    upload=yes

# Upload ด้วยตัวแปร
:local ftpServer "192.168.1.100"
:local ftpUser "admin"
:local ftpPass "password"
:local localFile "config.rsc"
:local remotePath "/backups/"
:local remoteFile ($remotePath . "backup-" . [/system clock get date] . ".rsc")

/tool fetch \
    address=$ftpServer \
    src-path=$localFile \
    user=$ftpUser \
    password=$ftpPass \
    dst-path=$remoteFile \
    upload=yes

:put "Uploaded to FTP: $remoteFile"

# Download จาก FTP
/tool fetch \
    address=192.168.1.100 \
    src-path=/configs/latest.rsc \
    user=ftpuser \
    password=ftppass \
    dst-path=downloaded-config.rsc

:put "Downloaded from FTP"
```

### FTP Backup Automation

```routeros
# Complete FTP backup script
:local ftpBackup do={
    :local ftpHost $1
    :local ftpUser $2
    :local ftpPass $3
    :local backupPath $4
    
    # สร้าง backup ก่อน
    :local date [/system clock get date]
    :local hostname [/system identity get name]
    :local backupName ($hostname . "-" . $date . ".backup")
    :local rscName ($hostname . "-" . $date . ".rsc")
    
    :put "Creating backup..."
    /system backup save name=$backupName password=""
    /export file=$rscName
    
    :delay 2s
    
    # Upload ทั้งสองไฟล์
    :do {
        /tool fetch \
            address=$ftpHost \
            src-path=$backupName \
            user=$ftpUser \
            password=$ftpPass \
            dst-path=($backupPath . $backupName) \
            upload=yes
        :put "Uploaded: $backupName"
    } on-error={
        :put "ERROR: Failed to upload $backupName"
    }
    
    :do {
        /tool fetch \
            address=$ftpHost \
            src-path=$rscName \
            user=$ftpUser \
            password=$ftpPass \
            dst-path=($backupPath . $rscName) \
            upload=yes
        :put "Uploaded: $rscName"
    } on-error={
        :put "ERROR: Failed to upload $rscName"
    }
    
    # ลบไฟล์ local (optional)
    # /file remove [find name=$backupName]
    # /file remove [find name=$rscName]
    
    :put "Backup complete!"
}

[$ftpBackup "192.168.1.100" "ftpuser" "ftppass" "/backups/"]
```

---

## 23.6 HTTP Operations

### HTTP GET ดาวน์โหลดไฟล์

```routeros
# Download จาก HTTP
/tool fetch url="http://example.com/config.txt" dst-path="config.txt"

# Download จาก HTTPS
/tool fetch \
    url="https://example.com/updates/latest.rsc" \
    dst-path="update.rsc" \
    mode=https

# Download พร้อม authentication
/tool fetch \
    url="http://192.168.1.100/api/config" \
    http-method=get \
    http-header-field="Authorization: Bearer mytoken" \
    dst-path="api-response.txt"

# Download และ ตรวจสอบ
:do {
    /tool fetch url="http://example.com/file.txt" dst-path="temp.txt"
    :put "Download successful"
} on-error={
    :put "Download failed"
}
```

### HTTP POST ส่งข้อมูล

```routeros
# POST data ไปยัง server
/tool fetch \
    url="http://192.168.1.100/api/update" \
    http-method=post \
    http-data="router=mikrotik&status=online" \
    dst-path="response.txt"

# POST JSON data
:local jsonData "{\"router\":\"MikroTik-01\",\"status\":\"up\",\"uptime\":\"5d\"}"
/tool fetch \
    url="http://192.168.1.100/api/status" \
    http-method=post \
    http-header-field="Content-Type: application/json" \
    http-data=$jsonData \
    dst-path="api-response.json"

# อ่าน response
:local response [/file get [/file find name="api-response.json"] contents]
:put "Server response: $response"
```

---

## 23.7 Log File Management

### การจัดการ Log

```routeros
# ดู log ปัจจุบัน
/log print

# Export log ลงไฟล์
/log print file=system-log

# Filter log ก่อน export
/log print where topics~"firewall" file=firewall-log

# สร้าง log action บันทึกลงไฟล์
/system logging action add name=toDisk \
    target=disk \
    disk-file-name=router-log \
    disk-lines-per-file=1000 \
    disk-file-count=5

# กำหนด topics ให้ใช้ action นี้
/system logging add topics=error action=toDisk
/system logging add topics=warning action=toDisk
/system logging add topics=critical action=toDisk

# ดู log files
/file print where name~"router-log"

# ลบ log files เก่า
:foreach logFile in=[/file find where name~"router-log"] do={
    :local size [/file get $logFile size]
    :if ($size < 100) do={
        /file remove $logFile
        :put "Removed empty log file"
    }
}
```

### Custom Log Rotation

```routeros
# Log rotation script
:local rotateLog do={
    :local logName $1
    :local maxFiles $2
    
    # ดูรายการ log files
    :local logFiles [/file find where name~$logName]
    :local count [:len $logFiles]
    
    :put "Found $count log files"
    
    # ถ้ามีมากเกิน maxFiles ให้ลบเก่าสุด
    :if ($count >= $maxFiles) do={
        :local toDelete [:pick $logFiles 0 ($count - $maxFiles + 1)]
        :foreach f in=$toDelete do={
            :local fname [/file get $f name]
            /file remove $f
            :put "Rotated: $fname"
        }
    }
}

# รัน rotation ทุกวัน
[$rotateLog "router-log" 10]
```

---

## 23.8 Config Files

### Export/Import Config

```routeros
# Export full config
/export file=full-config

# Export ส่วนต่างๆ แยกกัน
/ip address export file=ip-config
/ip route export file=route-config
/ip firewall filter export file=firewall-config
/interface export file=interface-config
/system identity export file=identity-config

# Export compact (ไม่รวม default values)
/export compact file=compact-config

# Import config
/import file-name=full-config.rsc

# Import และดูผลลัพธ์
:do {
    /import file-name=config.rsc
    :put "Config imported successfully"
} on-error={
    :put "Config import failed!"
}

# Validate config ก่อน import
:local validateConfig do={
    :local fileName $1
    :local fileId [/file find name=$fileName]
    
    :if ([:len $fileId] = 0) do={
        :put "Config file not found"
        :return false
    }
    
    :local content [/file get $fileId contents]
    
    # ตรวจสอบว่ามี required sections
    :if ([:find $content "/ip address"] = -1) do={
        :put "WARNING: No IP addresses in config"
    }
    
    :put "Config validation passed"
    :return true
}
```

---

## 23.9 Script Files

### จัดการ Script Files

```routeros
# ดู scripts ที่มีอยู่
/system script print

# Export script เป็นไฟล์
/system script export file=all-scripts

# Import script จากไฟล์
/import file-name=scripts.rsc

# สร้าง script ใหม่ผ่านไฟล์
:local scriptContent "# My Script\r\n:put \"Hello from script\""

/tool fetch url="data:," dst-path="myscript.rsc"
/file set [find name="myscript.rsc"] contents=$scriptContent

# Run script file โดยตรง
/import file-name="myscript.rsc"

# สร้าง script จาก template
:local generateScript do={
    :local name $1
    :local action $2
    :local template ("# Script: " . $name . "\r\n" . \
                    "# Generated: " . [/system clock get date] . "\r\n\r\n" . \
                    $action)
    
    :local filename ($name . ".rsc")
    /tool fetch url="data:," dst-path=$filename
    /file set [find name=$filename] contents=$template
    :put "Script created: $filename"
}

[$generateScript "daily-backup" ":put \"Backup started\"\r\n/system backup save name=daily\r\n:put \"Done\""]
```

---

## Lab 23: Config Backup to FTP

### โจทย์
สร้าง automated backup script ที่:
1. Export config เป็น .backup และ .rsc
2. Upload ไปยัง FTP server
3. เก็บ backup 7 วันล่าสุดบน router
4. Log ผลลัพธ์

### Solution

```routeros
# Config Backup to FTP Lab
# ========================

# Configuration
:local ftpServer "192.168.1.100"
:local ftpUser "backup-user"
:local ftpPass "BackupP@ss123"
:local ftpPath "/router-backups/"
:local keepDays 7
:local keepCount 7

# Get identifiers
:local hostname [/system identity get name]
:local date [/system clock get date]
:local time [/system clock get time]

# สร้าง filename base
:local dateStr [:tostr $date]
# แทนที่ / ด้วย -
:local cleanDate ""
:for i from=0 to=([:len $dateStr] - 1) do={
    :local c [:pick $dateStr $i ($i + 1)]
    :if ($c = "/") do={ :set cleanDate ($cleanDate . "-") } else={ :set cleanDate ($cleanDate . $c) }
}

:local baseFileName ($hostname . "-" . $cleanDate)
:local backupFile ($baseFileName . ".backup")
:local rscFile ($baseFileName . ".rsc")
:local logFile "backup-log.txt"

# --- Logging Function ---
:local log do={
    :global logFile
    :local msg $1
    :local timestamp [/system clock get time]
    :local logMsg ("[$timestamp] " . $msg . "\r\n")
    
    # เขียน log ลงไฟล์
    :local fid [/file find name=$logFile]
    :local existing ""
    :if ([:len $fid] > 0) do={
        :set existing [/file get $fid contents]
    }
    
    :local newContent ($existing . $logMsg)
    
    :if ([:len $fid] > 0) do={
        /file set $fid contents=$newContent
    } else={
        /tool fetch url="data:," dst-path=$logFile
        /file set [find name=$logFile] contents=$newContent
    }
    
    :put $logMsg
    /log info message=$msg
}

# --- Start Backup ---
[$log "=== Backup Started ==="]
[$log "Router: $hostname"]
[$log "Date: $date $time"]

# Step 1: Create backup files
[$log "Creating .backup file: $backupFile"]
:do {
    /system backup save name=$baseFileName password=""
    :delay 2s
    [$log "Backup file created OK"]
} on-error={
    [$log "ERROR: Failed to create backup file"]
    :error "Backup creation failed"
}

[$log "Creating .rsc file: $rscFile"]
:do {
    /export file=$baseFileName
    :delay 2s
    [$log "RSC file created OK"]
} on-error={
    [$log "ERROR: Failed to create rsc file"]
    :error "RSC export failed"
}

# Step 2: Verify files exist
:if ([:len [/file find name=$backupFile]] = 0) do={
    [$log "ERROR: Backup file not found after creation"]
    :error "Backup file missing"
}

:local backupSize [/file get [/file find name=$backupFile] size]
[$log "Backup file size: $backupSize bytes"]

# Step 3: Upload to FTP
[$log "Uploading to FTP: $ftpServer"]

:do {
    /tool fetch \
        address=$ftpServer \
        src-path=$backupFile \
        user=$ftpUser \
        password=$ftpPass \
        dst-path=($ftpPath . $backupFile) \
        upload=yes
    [$log "Uploaded $backupFile to FTP OK"]
} on-error={
    [$log "ERROR: FTP upload failed for $backupFile"]
}

:do {
    /tool fetch \
        address=$ftpServer \
        src-path=$rscFile \
        user=$ftpUser \
        password=$ftpPass \
        dst-path=($ftpPath . $rscFile) \
        upload=yes
    [$log "Uploaded $rscFile to FTP OK"]
} on-error={
    [$log "ERROR: FTP upload failed for $rscFile"]
}

# Step 4: Cleanup old backups (keep last N)
[$log "Cleaning up old backups (keeping last $keepCount)..."]

:local backupFiles [/file find where name~($hostname . "-") and name~".backup"]
:local rscFiles [/file find where name~($hostname . "-") and name~".rsc"]

:local bCount [:len $backupFiles]
:if ($bCount > $keepCount) do={
    :local toDelete [:pick $backupFiles 0 ($bCount - $keepCount)]
    :foreach f in=$toDelete do={
        :local fname [/file get $f name]
        /file remove $f
        [$log "Deleted old backup: $fname"]
    }
}

:local rCount [:len $rscFiles]
:if ($rCount > $keepCount) do={
    :local toDelete [:pick $rscFiles 0 ($rCount - $keepCount)]
    :foreach f in=$toDelete do={
        :local fname [/file get $f name]
        /file remove $f
        [$log "Deleted old rsc: $fname"]
    }
}

# Step 5: Upload log to FTP
:do {
    /tool fetch \
        address=$ftpServer \
        src-path=$logFile \
        user=$ftpUser \
        password=$ftpPass \
        dst-path=($ftpPath . "backup-log.txt") \
        upload=yes
} on-error={
    :put "WARNING: Could not upload log file"
}

[$log "=== Backup Complete ==="]
:put "Backup finished successfully!"
```

### Schedule the Backup

```routeros
# สร้าง scheduler ให้รัน backup ทุกวัน 02:00
/system scheduler add \
    name="daily-backup" \
    start-date=jan/01/2025 \
    start-time=02:00:00 \
    interval=1d \
    on-event="/system script run backup-script" \
    comment="Daily backup to FTP"
```

---

## Tips

> **Tip 1:** RouterOS file system มีขนาดจำกัด ควร monitor ด้วย `/system resource print`

> **Tip 2:** ใช้ `/export compact` เพื่อลดขนาดไฟล์ config

> **Warning:** ไฟล์ขนาดใหญ่อาจทำให้ Router หน่วยความจำเต็มได้ ควรลบไฟล์ที่ไม่ใช้สม่ำเสมอ

> **Note:** `tool fetch` รองรับ HTTP, HTTPS, FTP protocols

| Operation | Command | หมายเหตุ |
|-----------|---------|---------|
| ดูไฟล์ | `/file print` | |
| ลบไฟล์ | `/file remove` | |
| อ่าน content | `/file get ... contents` | RouterOS 6.44+ |
| เขียนไฟล์ | `/file set ... contents=` | |
| Download | `/tool fetch url=...` | |
| Upload FTP | `/tool fetch ... upload=yes` | |
| Export config | `/export file=name` | |
| Import config | `/import file-name=name.rsc` | |

---

## Summary ของ Part 23

- RouterOS มี file system สำหรับเก็บ backups, scripts, configs
- ใช้ `/file` commands จัดการไฟล์
- `/tool fetch` ช่วย download/upload ผ่าน HTTP/FTP
- `/export` และ `/import` สำหรับ config management
- Logging actions ช่วยบันทึก logs ลงไฟล์

---

[← Part 22: Array Operations](part-022-array-operations.md) | [Part 24: Scheduler →](part-024-scheduler.md)
