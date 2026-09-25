# Part 21: String Operations ใน RouterOS Scripting

## บทนำ

String operations เป็นพื้นฐานสำคัญในการเขียน RouterOS scripts การจัดการข้อความช่วยให้เราสามารถ parse ข้อมูล, สร้าง messages, และประมวลผล configuration ได้อย่างมีประสิทธิภาพ

---

## 21.1 String Functions พื้นฐาน

### :len - ความยาวของ String

```routeros
# วัดความยาวของ string
:local myStr "Hello, MikroTik!"
:put [:len $myStr]
# Output: 16

# ใช้กับ IP address
:local ip "192.168.1.100"
:put [:len $ip]
# Output: 13

# ตรวจสอบว่า string ว่างหรือไม่
:if ([:len $myStr] = 0) do={
    :put "String is empty"
} else={
    :put "String length: [:len $myStr]"
}
```

### :tostr - แปลงเป็น String

```routeros
# แปลง number เป็น string
:local num 42
:local str [:tostr $num]
:put $str
# Output: "42"

# แปลง IP เป็น string
:local ip 192.168.1.1
:local ipStr [:tostr $ip]
:put [:typeof $ipStr]
# Output: str

# แปลง boolean เป็น string
:local flag true
:put [:tostr $flag]
# Output: "true"

# แปลง time เป็น string
:local t [/system clock get time]
:put [:tostr $t]
```

### :tonum - แปลงเป็น Number

```routeros
# แปลง string เป็น number
:local str "123"
:local num [:tonum $str]
:put ($num + 10)
# Output: 133

# ระวัง! ถ้าไม่ใช่ตัวเลขจะ error
:local str2 "abc"
:do {
    :local n [:tonum $str2]
} on-error={
    :put "Cannot convert to number"
}
```

### :toint - แปลงเป็น Integer

```routeros
# แปลง string เป็น integer
:local strNum "255"
:local intNum [:toint $strNum]
:put $intNum
# Output: 255

# ใช้ใน arithmetic
:local a [:toint "100"]
:local b [:toint "200"]
:put ($a + $b)
# Output: 300

# แปลง hex string เป็น integer
:local hexStr "0xFF"
:put [:toint $hexStr]
# Output: 255
```

---

## 21.2 String Search Functions

### :find - ค้นหาใน String

```routeros
# หา substring ใน string
:local text "RouterOS is awesome"
:local pos [:find $text "is"]
:put $pos
# Output: 9 (zero-based index)

# หาไม่เจอ return -1
:local pos2 [:find $text "xyz"]
:put $pos2
# Output: -1

# ตรวจสอบว่ามี substring อยู่หรือไม่
:if ([:find $text "RouterOS"] != -1) do={
    :put "Found RouterOS in text"
}

# หา substring เริ่มจาก position ที่กำหนด
:local email "user@example.com"
:local atPos [:find $email "@"]
:put "@ is at position: $atPos"
# Output: @ is at position: 4
```

### :pick - ดึง substring

```routeros
# ดึง substring จาก string
:local text "Hello, World!"
:local sub [:pick $text 7 12]
:put $sub
# Output: "World"

# ดึงตั้งแต่ position 0 ถึง position 5
:local first5 [:pick $text 0 5]
:put $first5
# Output: "Hello"

# ดึง IP address ส่วน octets
:local ip "192.168.1.100"
:local firstOctetEnd [:find $ip "."]
:local firstOctet [:pick $ip 0 $firstOctetEnd]
:put $firstOctet
# Output: "192"

# ดึงส่วนหลังของ string
:local domain "user@example.com"
:local atPos [:find $domain "@"]
:local domainPart [:pick $domain ($atPos + 1) [:len $domain]]
:put $domainPart
# Output: "example.com"
```

---

## 21.3 String Case Functions

### :toupper - แปลงเป็น uppercase

```routeros
# แปลงเป็นตัวพิมพ์ใหญ่
:local text "hello world"
:put [:toupper $text]
# Output: "HELLO WORLD"

# ใช้ใน comparison (case-insensitive)
:local input "admin"
:local expected "ADMIN"
:if ([:toupper $input] = $expected) do={
    :put "Match!"
}
```

### :tolower - แปลงเป็น lowercase

```routeros
# แปลงเป็นตัวพิมพ์เล็ก
:local text "HELLO WORLD"
:put [:tolower $text]
# Output: "hello world"

# Normalize username
:local username "Admin"
:local normalized [:tolower $username]
:put $normalized
# Output: "admin"
```

---

## 21.4 String Concatenation

```routeros
# การต่อ string ด้วย . (dot)
:local first "Hello"
:local last "World"
:local combined ($first . " " . $last)
:put $combined
# Output: "Hello World"

# ต่อกับตัวแปรอื่น
:local hostname [/system identity get name]
:local msg ("Router: " . $hostname . " is online")
:put $msg

# ต่อ string หลายๆ ตัว
:local a "192."
:local b "168."
:local c "1."
:local d "1"
:local ip ($a . $b . $c . $d)
:put $ip
# Output: "192.168.1.1"

# String concatenation ใน loop
:local result ""
:for i from=1 to=5 do={
    :set result ($result . [:tostr $i] . ",")
}
:put $result
# Output: "1,2,3,4,5,"
```

---

## 21.5 String Splitting

RouterOS ไม่มี built-in split function แต่เราสามารถทำได้ด้วย find และ pick:

```routeros
# แยก string ด้วย delimiter
:local csv "192.168.1.1,192.168.1.2,192.168.1.3"
:local delimiter ","
:local result [:toarray $csv]

# Method 1: ใช้ :toarray สำหรับ comma-separated
:local ips "192.168.1.1,192.168.1.2,192.168.1.3"
:local ipArr [:toarray $ips]
:foreach ip in=$ipArr do={
    :put $ip
}

# Method 2: Manual splitting
:local str "one:two:three"
:local sep ":"
:local remaining $str
:local parts ""

:while ([:find $remaining $sep] != -1) do={
    :local sepPos [:find $remaining $sep]
    :local part [:pick $remaining 0 $sepPos]
    :put "Part: $part"
    :set remaining [:pick $remaining ($sepPos + 1) [:len $remaining]]
}
:put "Last part: $remaining"

# Split IP into octets
:local ip "192.168.10.50"
:local oct1End [:find $ip "."]
:local oct1 [:pick $ip 0 $oct1End]

:local remaining [:pick $ip ($oct1End + 1) [:len $ip]]
:local oct2End [:find $remaining "."]
:local oct2 [:pick $remaining 0 $oct2End]

:set remaining [:pick $remaining ($oct2End + 1) [:len $remaining]]
:local oct3End [:find $remaining "."]
:local oct3 [:pick $remaining 0 $oct3End]
:local oct4 [:pick $remaining ($oct3End + 1) [:len $remaining]]

:put "Octet 1: $oct1"
:put "Octet 2: $oct2"
:put "Octet 3: $oct3"
:put "Octet 4: $oct4"
```

---

## 21.6 Regular Expressions ใน RouterOS

RouterOS support regular expressions ใน firewall และ find operations บางส่วน:

```routeros
# ใช้ regex ใน find
# RouterOS ใช้ POSIX regex

# Firewall - match by regex
/ip firewall filter add chain=forward \
    comment~"BLOCKED" action=drop

# Address list ที่มี regex
/ip firewall address-list print where list~"blacklist"

# Find objects ที่ match regex
/interface print where name~"ether"
/interface print where name~"^ether[0-9]$"

# Log filter ด้วย regex
/log print where message~"login"
/log print where message~"failed.*admin"

# Wireless - match SSID
/interface wireless print where ssid~"Office"

# Certificate matching
/certificate print where name~".*CA.*"
```

```routeros
# Script ที่ใช้ regex สำหรับ validation
:local validateEmail do={
    :local email $1
    # Basic email validation
    :local atPos [:find $email "@"]
    :if ($atPos = -1) do={ :return false }
    :local domain [:pick $email ($atPos + 1) [:len $email]]
    :local dotPos [:find $domain "."]
    :if ($dotPos = -1) do={ :return false }
    :return true
}

:if ([$validateEmail "user@example.com"]) do={
    :put "Valid email"
} else={
    :put "Invalid email"
}
```

---

## 21.7 String Templates

```routeros
# สร้าง string template สำหรับ messages
:local makeMessage do={
    :local level $1
    :local message $2
    :local timestamp [/system clock get time]
    :local hostname [/system identity get name]
    :return ("[$timestamp][$hostname][$level] " . $message)
}

:put [$makeMessage "INFO" "System started"]
:put [$makeMessage "ERROR" "Connection failed"]
:put [$makeMessage "WARNING" "CPU usage high"]

# Email template
:local emailTemplate do={
    :local subject $1
    :local body $2
    :local router [/system identity get name]
    :local time [/system clock get date]
    
    :local fullBody ("Router: " . $router . "\r\n" . \
                    "Date: " . $time . "\r\n\r\n" . \
                    $body)
    
    /tool e-mail send to="admin@example.com" \
        subject=$subject body=$fullBody
}

[$emailTemplate "Alert!" "Interface ether1 is down"]
```

---

## 21.8 String Formatting

```routeros
# Format numbers ให้มี leading zeros
:local padLeft do={
    :local num [:tostr $1]
    :local width $2
    :local pad $3
    :while ([:len $num] < $width) do={
        :set num ($pad . $num)
    }
    :return $num
}

:put [$padLeft 5 3 "0"]
# Output: "005"

:put [$padLeft 42 5 "0"]
# Output: "00042"

# Format IP address
:local formatIP do={
    :local ip $1
    :local parts [:toarray $ip]
    :local result ""
    :foreach part in=$parts do={
        :set result ($result . [$padLeft [:tonum $part] 3 "0"] . ".")
    }
    :return [:pick $result 0 ([:len $result] - 1)]
}

# Format byte size
:local formatBytes do={
    :local bytes $1
    :if ($bytes < 1024) do={ :return ([:tostr $bytes] . " B") }
    :if ($bytes < 1048576) do={ :return ([:tostr ($bytes / 1024)] . " KB") }
    :if ($bytes < 1073741824) do={ :return ([:tostr ($bytes / 1048576)] . " MB") }
    :return ([:tostr ($bytes / 1073741824)] . " GB")
}

:put [$formatBytes 500]
# Output: "500 B"
:put [$formatBytes 2048]
# Output: "2 KB"
:put [$formatBytes 5242880]
# Output: "5 MB"
```

---

## 21.9 Parsing Output Strings

```routeros
# Parse /ip address output
:local parseIPOutput do={
    :foreach addr in=[/ip address find] do={
        :local ip [/ip address get $addr address]
        :local iface [/ip address get $addr interface]
        :put ("Interface: " . $iface . " -> IP: " . $ip)
    }
}

$parseIPOutput

# Parse log entries
:local parseLastLog do={
    :local lastLog [/log get [:pick [/log find] 0 1] message]
    :put "Last log: $lastLog"
    
    # Extract timestamp
    :if ([:find $lastLog " "] != -1) do={
        :local spacePos [:find $lastLog " "]
        :local timeStr [:pick $lastLog 0 $spacePos]
        :put "Time: $timeStr"
    }
}

$parseLastLog

# Parse interface statistics
:foreach iface in=[/interface find] do={
    :local name [/interface get $iface name]
    :local rxBytes [/interface get $iface rx-byte]
    :local txBytes [/interface get $iface tx-byte]
    :put ("$name: RX=" . $rxBytes . " TX=" . $txBytes)
}
```

---

## 21.10 IP String Manipulation

```routeros
# แปลง IP string เป็น components
:local parseIP do={
    :local ip $1
    :local octets {}
    :local remaining $ip
    
    :for i from=0 to=3 do={
        :local dotPos [:find $remaining "."]
        :if ($dotPos != -1) do={
            :local octet [:pick $remaining 0 $dotPos]
            :set octets ($octets , $octet)
            :set remaining [:pick $remaining ($dotPos + 1) [:len $remaining]]
        } else={
            :set octets ($octets , $remaining)
        }
    }
    :return $octets
}

# แปลง IP เป็น integer
:local ipToInt do={
    :local ip $1
    :local octets [$parseIP $ip]
    :local result 0
    :local multiplier 16777216
    
    :foreach oct in=$octets do={
        :set result ($result + ([:toint $oct] * $multiplier))
        :set multiplier ($multiplier / 256)
    }
    :return $result
}

# ตรวจสอบว่า IP อยู่ใน subnet
:local isInSubnet do={
    :local ip $1
    :local network $2
    :local prefix [:pick $network ([:find $network "/"] + 1) [:len $network]]
    :local netAddr [:pick $network 0 [:find $network "/"]]
    
    :local ipInt [$ipToInt $ip]
    :local netInt [$ipToInt $netAddr]
    :local mask (0xFFFFFFFF << (32 - [:toint $prefix]))
    
    :return (($ipInt & $mask) = ($netInt & $mask))
}

# Generate next IP
:local nextIP do={
    :local ip $1
    :local octets [$parseIP $ip]
    :local oct4 [:toint ($octets->3)]
    :set ($octets->3) [:tostr ($oct4 + 1)]
    :return ($octets->0 . "." . $octets->1 . "." . $octets->2 . "." . $octets->3)
}

:put [$nextIP "192.168.1.10"]
# Output: "192.168.1.11"

# ตรวจสอบ valid IP format
:local isValidIP do={
    :local ip $1
    :local valid true
    :local remaining $ip
    
    :for i from=0 to=3 do={
        :local dotPos [:find $remaining "."]
        :local octet ""
        
        :if ($dotPos != -1) do={
            :set octet [:pick $remaining 0 $dotPos]
            :set remaining [:pick $remaining ($dotPos + 1) [:len $remaining]]
        } else={
            :set octet $remaining
        }
        
        :local octetNum [:toint $octet]
        :if ($octetNum < 0 || $octetNum > 255) do={
            :set valid false
        }
    }
    :return $valid
}
```

---

## 21.11 Practical Examples

### Example 1: URL Parser

```routeros
# Parse URL components
:local parseURL do={
    :local url $1
    :local protocol ""
    :local host ""
    :local path ""
    
    # Extract protocol
    :local colonPos [:find $url "://"]
    :if ($colonPos != -1) do={
        :set protocol [:pick $url 0 $colonPos]
        :set url [:pick $url ($colonPos + 3) [:len $url]]
    }
    
    # Extract host and path
    :local slashPos [:find $url "/"]
    :if ($slashPos != -1) do={
        :set host [:pick $url 0 $slashPos]
        :set path [:pick $url $slashPos [:len $url]]
    } else={
        :set host $url
        :set path "/"
    }
    
    :put "Protocol: $protocol"
    :put "Host: $host"
    :put "Path: $path"
}

[$parseURL "http://192.168.1.1/api/v1/data"]
[$parseURL "https://example.com/path/to/page"]
```

### Example 2: MAC Address Formatter

```routeros
# แปลง MAC address format
:local formatMAC do={
    :local mac $1
    # ลบ separator ออกก่อน
    :local clean ""
    :for i from=0 to=([:len $mac] - 1) do={
        :local char [:pick $mac $i ($i + 1)]
        :if ($char != ":" && $char != "-") do={
            :set clean ($clean . $char)
        }
    }
    
    # เพิ่ม separator แบบ colon
    :local result ""
    :for i from=0 to=5 do={
        :if ([:len $result] > 0) do={ :set result ($result . ":") }
        :set result ($result . [:pick $clean ($i*2) ($i*2+2)])
    }
    :return [:toupper $result]
}

:put [$formatMAC "aabbccddeeff"]
# Output: "AA:BB:CC:DD:EE:FF"

:put [$formatMAC "aa-bb-cc-dd-ee-ff"]
# Output: "AA:BB:CC:DD:EE:FF"
```

### Example 3: Config Generator

```routeros
# Generate interface configuration string
:local genIfaceConfig do={
    :local ifaceName $1
    :local ipAddress $2
    :local comment $3
    
    :local config ""
    :set config ($config . "/ip address add \\\r\n")
    :set config ($config . "    address=" . $ipAddress . " \\\r\n")
    :set config ($config . "    interface=" . $ifaceName . " \\\r\n")
    :set config ($config . "    comment=\"" . $comment . "\"")
    
    :return $config
}

:put [$genIfaceConfig "ether1" "192.168.1.1/24" "LAN Interface"]
```

---

## 21.12 String Comparison

```routeros
# เปรียบเทียบ strings
:local str1 "hello"
:local str2 "HELLO"

# Case sensitive
:if ($str1 = $str2) do={
    :put "Equal (case sensitive)"
} else={
    :put "Not equal (case sensitive)"
}

# Case insensitive
:if ([:tolower $str1] = [:tolower $str2]) do={
    :put "Equal (case insensitive)"
}

# Compare prefix
:local checkPrefix do={
    :local str $1
    :local prefix $2
    :local strPrefix [:pick $str 0 [:len $prefix]]
    :return ($strPrefix = $prefix)
}

:if ([$checkPrefix "ether1-LAN" "ether"]) do={
    :put "Starts with 'ether'"
}

# Compare suffix
:local checkSuffix do={
    :local str $1
    :local suffix $2
    :local strSuffix [:pick $str ([:len $str] - [:len $suffix]) [:len $str]]
    :return ($strSuffix = $suffix)
}

:if ([$checkSuffix "config.rsc" ".rsc"]) do={
    :put "Is a .rsc file"
}
```

---

## 21.13 String Replacement (Workaround)

RouterOS ไม่มี built-in string replace แต่เราสามารถทำได้:

```routeros
# Replace all occurrences
:local strReplace do={
    :local str $1
    :local find $2
    :local replace $3
    :local result ""
    
    :while ([:find $str $find] != -1) do={
        :local pos [:find $str $find]
        :set result ($result . [:pick $str 0 $pos] . $replace)
        :set str [:pick $str ($pos + [:len $find]) [:len $str]]
    }
    :set result ($result . $str)
    :return $result
}

:put [$strReplace "Hello World" "o" "0"]
# Output: "Hell0 W0rld"

:put [$strReplace "192-168-1-1" "-" "."]
# Output: "192.168.1.1"

# Trim whitespace (leading and trailing)
:local trim do={
    :local str $1
    # Trim leading spaces
    :while ([:len $str] > 0 && [:pick $str 0 1] = " ") do={
        :set str [:pick $str 1 [:len $str]]
    }
    # Trim trailing spaces
    :while ([:len $str] > 0 && [:pick $str ([:len $str]-1) [:len $str]] = " ") do={
        :set str [:pick $str 0 ([:len $str]-1)]
    }
    :return $str
}

:put [$trim "  hello world  "]
# Output: "hello world"
```

---

## Lab 21: String Parser Script

### โจทย์
สร้าง script ที่ parse log entries และสร้าง report

### Requirements
1. อ่าน log entries ล่าสุด 100 รายการ
2. Parse timestamp, topic, message
3. Count errors และ warnings
4. Generate summary report
5. บันทึก report ลงไฟล์

### Solution

```routeros
# String Parser Lab - Log Analysis Script
# ========================================

# --- Helper Functions ---

:local padLeft do={
    :local str [:tostr $1]
    :local width $2
    :local pad $3
    :while ([:len $str] < $width) do={
        :set str ($pad . $str)
    }
    :return $str
}

:local countOccurrences do={
    :local str $1
    :local sub $2
    :local count 0
    :local pos 0
    
    :while (true) do={
        :local found [:find $str $sub $pos]
        :if ($found = -1) do={ :break }
        :set count ($count + 1)
        :set pos ($found + [:len $sub])
    }
    :return $count
}

# --- Main Script ---

:global logReport ""
:local errorCount 0
:local warningCount 0
:local infoCount 0
:local totalEntries 0

:local header "=================================\r\n"
:set header ($header . "   RouterOS Log Analysis Report\r\n")
:set header ($header . "   " . [/system clock get date] . " " . [/system clock get time] . "\r\n")
:set header ($header . "   Router: " . [/system identity get name] . "\r\n")
:set header ($header . "=================================\r\n\r\n")

:set logReport $header

# อ่าน log entries
:local logEntries [/log find]
:local maxEntries 100
:if ([:len $logEntries] < $maxEntries) do={
    :set maxEntries [:len $logEntries]
}

:local recentEntries [:pick $logEntries 0 $maxEntries]

:foreach entry in=$recentEntries do={
    :set totalEntries ($totalEntries + 1)
    :local msg [/log get $entry message]
    :local topics [/log get $entry topics]
    :local time [/log get $entry time]
    
    # Count by type
    :if ([:find $topics "error"] != -1) do={
        :set errorCount ($errorCount + 1)
    }
    :if ([:find $topics "warning"] != -1) do={
        :set warningCount ($warningCount + 1)
    }
    :if ([:find $topics "info"] != -1) do={
        :set infoCount ($infoCount + 1)
    }
    
    # Look for important patterns
    :if ([:find $msg "login"] != -1) do={
        :set logReport ($logReport . "[LOGIN] $time: $msg\r\n")
    }
    :if ([:find $msg "failed"] != -1) do={
        :set logReport ($logReport . "[FAILED] $time: $msg\r\n")
    }
    :if ([:find $msg "interface"] != -1 && [:find $msg "down"] != -1) do={
        :set logReport ($logReport . "[IF-DOWN] $time: $msg\r\n")
    }
}

# Summary
:local summary "\r\n--- SUMMARY ---\r\n"
:set summary ($summary . "Total entries analyzed: $totalEntries\r\n")
:set summary ($summary . "Errors:   $errorCount\r\n")
:set summary ($summary . "Warnings: $warningCount\r\n")
:set summary ($summary . "Info:     $infoCount\r\n")

:set logReport ($logReport . $summary)

# บันทึกลงไฟล์
/file remove [find name="log-report.txt"] 
/tool fetch url="" dst-path="log-report.txt"
:put $logReport

:put "\nReport generated successfully!"
:put "Total entries: $totalEntries"
:put "Errors: $errorCount, Warnings: $warningCount"
```

### ผลลัพธ์ที่คาดหวัง

```
=================================
   RouterOS Log Analysis Report
   jan/01/2025 12:00:00
   Router: MikroTik-Office
=================================

[LOGIN] 11:30:15 admin logged in from 192.168.1.100
[FAILED] 11:45:22 login failure for user guest from 10.0.0.5

--- SUMMARY ---
Total entries analyzed: 100
Errors:   3
Warnings: 7
Info:     90
```

---

## Tips และ Best Practices

> **Tip 1:** RouterOS strings เป็น immutable ทุกการ modify สร้าง string ใหม่

> **Tip 2:** ใช้ `:toarray` เมื่อต้องการ split comma-separated strings

> **Warning:** `:tonum` จะ error ถ้า string ไม่ใช่ตัวเลข ควรใช้ `:do...on-error`

> **Note:** String indexing เริ่มจาก 0

> **Best Practice:** ใช้ `[:len $str] = 0` แทน `$str = ""` เพื่อตรวจสอบ empty string

---

## Summary ของ Part 21

| Function | ใช้งาน | ตัวอย่าง |
|----------|--------|---------|
| `:len` | ความยาว string | `[:len "hello"]` → 5 |
| `:tostr` | แปลงเป็น string | `[:tostr 42]` → "42" |
| `:tonum` | แปลงเป็น number | `[:tonum "42"]` → 42 |
| `:toint` | แปลงเป็น integer | `[:toint "0xFF"]` → 255 |
| `:find` | หา substring | `[:find "hello" "ll"]` → 2 |
| `:pick` | ดึง substring | `[:pick "hello" 1 3]` → "el" |
| `:toupper` | ตัวพิมพ์ใหญ่ | `[:toupper "hi"]` → "HI" |
| `:tolower` | ตัวพิมพ์เล็ก | `[:tolower "HI"]` → "hi" |
| `.` (dot) | ต่อ string | `"a" . "b"` → "ab" |

---

[← Part 20: Advanced Loops](part-020-advanced-loops.md) | [Part 22: Array Operations →](part-022-array-operations.md)
