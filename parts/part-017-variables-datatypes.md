# Part 17: Variables & Data Types

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate-Advanced
**เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

- [Variable Declaration](#variable-declaration)
- [Data Types ทั้งหมด](#data-types)
  - [String (str)](#string)
  - [Number (num)](#number)
  - [Boolean (bool)](#boolean)
  - [IP Address (ip)](#ip-address)
  - [IP Prefix (ip-prefix)](#ip-prefix)
  - [MAC Address (mac)](#mac-address)
  - [Array](#array)
  - [Nothing/Nil](#nothing)
  - [Time](#time)
  - [ID](#id)
- [Type Conversion](#type-conversion)
- [Variable Scope](#variable-scope)
- [Constants](#constants)
- [Special Variables](#special-variables)
- [Nil/Nothing Handling](#nil-handling)
- [Type Checking](#type-checking)
- [Variable Naming Conventions](#naming-conventions)

---

## Variable Declaration

### :local - Local Variable

```routeros
# การประกาศ Local Variable
:local variableName
:local variableName value
:local variableName "string value"
:local variableName 42
:local variableName true

# ตัวอย่าง
:local name "John"
:local age 30
:local isActive true
:local ipAddr 192.168.1.1
```

### :global - Global Variable

```routeros
# Global Variable - ใช้ร่วมกันได้ระหว่าง Scripts
:global variableName
:global variableName "initial value"

# ตัวอย่าง
:global currentISP "ISP1"
:global failoverCount 0
:global lastBackupDate ""
```

### :set - กำหนดค่า

```routeros
# กำหนดค่าให้ Variable ที่ประกาศแล้ว
:local x
:set x 100

:local name
:set name "MikroTik"

# กำหนดพร้อมประกาศ (เป็นทางลัด)
:local x 100
:local name "MikroTik"

# กำหนดค่าใหม่
:local counter 0
:set counter ($counter + 1)  # counter = 1
:set counter ($counter + 1)  # counter = 2
```

### ตัวอย่างการใช้งาน Variable

```routeros
# ตัวอย่างที่สมบูรณ์
:local firstName "สมชาย"
:local lastName "ใจดี"
:local age 25
:local salary 50000.00
:local isEmployee true

:put ("ชื่อ: " . $firstName . " " . $lastName)
:put ("อายุ: " . $age . " ปี")
:put ("เงินเดือน: " . $salary . " บาท")
:put ("เป็นพนักงาน: " . $isEmployee)
```

---

## Data Types

### ประเภทข้อมูลทั้งหมด

| Type | ตัวอย่าง | คำอธิบาย |
|------|---------|---------|
| `str` | "hello" | String (ข้อความ) |
| `num` | 42, 3.14 | Number (ตัวเลข) |
| `bool` | true, false | Boolean |
| `ip` | 192.168.1.1 | IPv4 Address |
| `ip6` | ::1 | IPv6 Address |
| `ip-prefix` | 192.168.1.0/24 | IP Network/Prefix |
| `mac` | AA:BB:CC:DD:EE:FF | MAC Address |
| `time` | 1h30m15s | Time Duration |
| `array` | {1;2;3} | Array |
| `nothing` | - | Nil/Undefined |
| `id` | *1 | RouterOS Internal ID |

---

## String

### String พื้นฐาน

```routeros
# String ใช้ Double Quotes
:local str1 "Hello, World!"
:local str2 "MikroTik Router"

# String ว่าง
:local empty ""

# String ที่มี Special Characters
:local path "C:\\Users\\Admin"    # Backslash ต้องใช้ \\
:local newline "Line1\nLine2"     # \n = Newline
:local tab "Col1\tCol2"           # \t = Tab
:local quote "He said \"Hello\""  # \" = Quote ภายใน String
```

### String Operations

```routeros
# Concatenation (ต่อ String)
:local a "Hello"
:local b "World"
:local c ($a . ", " . $b . "!")
:put $c    # Hello, World!

# String Length
:local str "Hello"
:put [:len $str]    # 5

# Substring
:local str "Hello, World!"
:put [:pick $str 0 5]     # Hello (ตำแหน่ง 0-4)
:put [:pick $str 7 12]    # World (ตำแหน่ง 7-11)

# String ภาษาไทย
:local thai "สวัสดี MikroTik"
:put $thai
:put [:len $thai]    # นับ Characters
```

### String Functions

```routeros
# Convert to String
:local num 42
:local str [:tostr $num]
:put $str    # "42"

# String to Upper/Lower (RouterOS ไม่มี built-in แต่ทำได้)
# ใช้วิธีอื่นแทน

# Find in String
:local str "Hello, World!"
:local pos [:find $str "World"]
:put $pos    # 7 (ตำแหน่งที่พบ)

# หา String ไม่เจอ จะได้ ""
:local pos2 [:find $str "xyz"]
:if ($pos2 = "") do={
    :put "Not found"
}

# Pattern Matching (Regular Expression)
:local str "192.168.1.100"
:if ($str ~ "^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+\$") do={
    :put "Valid IP format"
}
```

### String ใน Conditions

```routeros
# เปรียบเทียบ String
:local a "hello"
:local b "hello"

:if ($a = $b) do={
    :put "Strings are equal"
}

# String ใน Pattern Match
:local ssid "Office-WiFi"
:if ($ssid ~ "Office") do={
    :put "This is an Office network"
}
```

---

## Number

### Integer และ Float

```routeros
# Integer
:local int1 42
:local int2 -10
:local int3 1000000

# Float (ทศนิยม)
:local float1 3.14
:local float2 -2.5
:local float3 0.001

# Hex (Base 16)
:local hex 0xFF      # = 255
:local hex2 0x1000   # = 4096

# Octal (Base 8)
:local oct 0o17      # = 15
```

### Arithmetic Operations

```routeros
:local a 10
:local b 3

:put ($a + $b)    # 13 (บวก)
:put ($a - $b)    # 7  (ลบ)
:put ($a * $b)    # 30 (คูณ)
:put ($a / $b)    # 3  (หาร - ตัดเศษ)
:put ($a % $b)    # 1  (โมดูลัส/เศษ)

# Float Arithmetic
:local c 10.0
:local d 3.0
:put ($c / $d)    # 3.333333...
```

### Number Conversion

```routeros
# String to Number
:local str "42"
:local num [:tonum $str]
:put ($num + 1)    # 43

# Number to String
:local num 42
:local str [:tostr $num]
:put ("Value: " . $str)

# Number Formatting
:local bigNum 1000000
:put $bigNum    # 1000000
```

### Math Operations

```routeros
# ใช้ [:tonum] สำหรับคำนวณ
:local a [:tonum "10"]
:local b [:tonum "3"]

# Integer Division
:local div ($a / $b)
:put $div    # 3

# คำนวณ Percentage
:local total 100
:local used 75
:local pct (($used * 100) / $total)
:put ($pct . "%")    # 75%
```

---

## Boolean

### Boolean Values

```routeros
# Boolean Values
:local isTrue true
:local isFalse false

:put $isTrue     # true
:put $isFalse    # false

# Boolean จาก Comparison
:local x 10
:local isPositive ($x > 0)
:put $isPositive    # true
```

### Boolean Operations

```routeros
:local a true
:local b false

:put ($a && $b)    # false (AND)
:put ($a || $b)    # true  (OR)
:put (!$a)         # false (NOT)

# เทียบกับ Boolean
:if ($a) do={ :put "a is true" }
:if (!$b) do={ :put "b is false" }

# ใช้ใน if
:local isConnected true
:if ($isConnected) do={
    :put "Connected"
} else={
    :put "Disconnected"
}
```

---

## IP Address

### IP Address Type

```routeros
# IP Address ใน RouterOS เป็น Type พิเศษ
:local ip1 192.168.1.1
:local ip2 10.0.0.1
:local ip3 172.16.0.100

# แสดง IP
:put $ip1    # 192.168.1.1

# เปรียบเทียบ IP
:if ($ip1 = 192.168.1.1) do={
    :put "Match!"
}
```

### IP Address Operations

```routeros
# ดึง IP จาก Interface
:local myIP [/ip address get [find interface=ether1] address]
:put $myIP    # เช่น 192.168.1.1/24

# ดึงเฉพาะ IP (ไม่มี Prefix)
:local ip [/ip address get [find interface=ether1] address]
:local ipOnly [:toip $ip]

# Convert String to IP
:local ipStr "192.168.1.1"
:local ip [:toip $ipStr]

# IP to String
:local ip 192.168.1.1
:local str [:tostr $ip]
:put ("IP is: " . $str)
```

### IPv6

```routeros
# IPv6 Address
:local ipv6 ::1
:local ipv6full 2001:db8::1

:put $ipv6
:put $ipv6full
```

---

## IP Prefix

### IP Prefix (Network/Mask)

```routeros
# IP Prefix
:local net1 192.168.1.0/24
:local net2 10.0.0.0/8
:local net3 172.16.0.0/12

:put $net1    # 192.168.1.0/24

# ตรวจสอบว่า IP อยู่ใน Network
:local network 192.168.1.0/24
:local testIP 192.168.1.50

:if ($testIP in $network) do={
    :put "$testIP is in $network"
}

# Convert String to IP-Prefix
:local str "192.168.1.0/24"
:local prefix [:toip-prefix $str]
```

### Prefix Operations

```routeros
# ดึง Network Address
:local prefix 192.168.1.100/24
:local network ($prefix & 255.255.255.0)

# ดูว่า IP อยู่ใน Prefix ไหม
:local prefixList {
    10.0.0.0/8;
    172.16.0.0/12;
    192.168.0.0/16
}

:local testIP 192.168.1.50
:foreach net in=$prefixList do={
    :if ($testIP in $net) do={
        :put ($testIP . " is RFC1918 (in " . $net . ")")
    }
}
```

---

## MAC Address

### MAC Address Type

```routeros
# MAC Address
:local mac1 AA:BB:CC:DD:EE:FF
:local mac2 00:11:22:33:44:55

:put $mac1    # AA:BB:CC:DD:EE:FF

# Convert String to MAC
:local macStr "AA:BB:CC:DD:EE:FF"
:local mac [:tomac $macStr]

# เปรียบเทียบ MAC
:if ($mac1 = $mac2) do={
    :put "Same MAC"
}
```

### MAC Address ในการใช้งาน

```routeros
# ดู MAC ของ Interface
:local mac [/interface get ether1 mac-address]
:put ("ether1 MAC: " . $mac)

# ค้นหา DHCP Lease โดย MAC
:local clientMAC "AA:BB:CC:DD:EE:01"
:local lease [/ip dhcp-server lease find mac-address=$clientMAC]

:if ([:len $lease] > 0) do={
    :local ip [/ip dhcp-server lease get $lease address]
    :put ("IP for " . $clientMAC . ": " . $ip)
}
```

---

## Array

### Array พื้นฐาน

```routeros
# สร้าง Array
:local arr {1; 2; 3; 4; 5}
:local strArr {"apple"; "banana"; "cherry"}
:local ipArr {192.168.1.1; 10.0.0.1; 172.16.0.1}

# ดูข้อมูลใน Array
:put $arr         # 1;2;3;4;5
:put $strArr      # apple;banana;cherry

# Array Length
:put [:len $arr]    # 5
```

### เข้าถึง Array Elements

```routeros
# ใช้ Index (เริ่มที่ 0)
:local arr {"first"; "second"; "third"}

:put ($arr->0)    # first
:put ($arr->1)    # second
:put ($arr->2)    # third

# Loop ใน Array
:foreach item in=$arr do={
    :put $item
}

# Loop พร้อม Index
:local i 0
:foreach item in=$arr do={
    :put ($i . ": " . $item)
    :set i ($i + 1)
}
```

### Array Operations

```routeros
# เพิ่ม Element (ต้อง Reconstruct Array)
:local arr {1; 2; 3}
:set arr ($arr, 4)    # เพิ่ม 4 ต่อท้าย
:put $arr    # 1;2;3;4

# Array ของ Router Objects
:local interfaces [/interface find type=ether]
:foreach iface in=$interfaces do={
    :local name [/interface get $iface name]
    :local running [/interface get $iface running]
    :put ($name . ": " . $running)
}

# กำหนด Array แบบ Key-Value
:local config {
    "hostname"="Router1";
    "admin"="admin@example.com";
    "location"="Bangkok"
}

:put ($config->"hostname")    # Router1
:put ($config->"admin")       # admin@example.com
```

### ตัวอย่าง Array ที่ใช้บ่อย

```routeros
# List ของ Hosts ที่ต้อง Ping
:local hosts {"8.8.8.8"; "1.1.1.1"; "8.8.4.4"; "208.67.222.222"}

:foreach host in=$hosts do={
    :local result [/ping $host count=2 as-value]
    :local received ($result->"received")

    :if ($received > 0) do={
        :log info ($host . " is reachable")
    } else={
        :log warning ($host . " is NOT reachable")
    }
}
```

---

## Nothing

### Nothing/Nil Type

```routeros
# Nothing = ไม่มีค่า / Undefined
:local x

:put $x    # (แสดงว่างเปล่า)
:put [:typeof $x]    # nothing

# ตรวจสอบว่าเป็น nothing
:if ($x = $nothing) do={
    :put "x has no value"
}

# หรือ
:if ([:typeof $x] = "nothing") do={
    :put "x is nothing type"
}
```

### การตั้งค่าเป็น Nothing

```routeros
# ลบค่าของ Variable
:local x 100
:set x    # ลบค่า x ออก (เป็น nothing)

# ตรวจสอบ
:if ([:typeof $x] = "nothing") do={
    :put "x is now nothing"
}
```

---

## Time

### Time Type

```routeros
# Time Duration
:local t1 1h
:local t2 30m
:local t3 15s
:local t4 1h30m15s
:local t5 00:01:30    # 1 นาที 30 วินาที

:put $t1    # 1h
:put $t4    # 1h30m15s

# ดูเวลาปัจจุบัน
:local now [/system clock get time]
:put $now    # เช่น 08:30:00

:local date [/system clock get date]
:put $date   # เช่น jan/01/2026
```

### Time Operations

```routeros
# เปรียบเทียบ Time
:local t1 1h
:local t2 30m

:if ($t1 > $t2) do={
    :put "t1 is longer than t2"
}

# Convert Time to Seconds
:local t 1h30m
:local secs [:tonum $t]   # 5400 seconds

# System Uptime
:local uptime [/system resource get uptime]
:put ("Uptime: " . $uptime)

# ตรวจว่า Uptime น้อยกว่า 5 นาที (อาจเพิ่ง Reboot)
:if ($uptime < 5m) do={
    :log warning "Router recently rebooted!"
}
```

---

## ID

### ID Type

```routeros
# ID ใช้ระบุ RouterOS Objects
# มีรูปแบบ *<number> เช่น *1, *2, ...

# ดึง ID ของ Object
:local id [/ip address find address="192.168.1.1/24"]
:put $id    # เช่น *1

# ใช้ ID
:local addr [/ip address get $id]
:put $addr

# ตรวจสอบว่า ID Valid
:if ([:len $id] > 0) do={
    :put "Found"
} else={
    :put "Not found"
}
```

---

## Type Conversion

### Conversion Functions

| Function | จาก | ไป | ตัวอย่าง |
|----------|-----|-----|---------|
| `:tostr` | any | str | `:tostr 42` → "42" |
| `:tonum` | str | num | `:tonum "42"` → 42 |
| `:toip` | str | ip | `:toip "192.168.1.1"` → ip |
| `:toip6` | str | ip6 | `:toip6 "::1"` → ip6 |
| `:toip-prefix` | str | ip-prefix | `:toip-prefix "192.168.1.0/24"` |
| `:tomac` | str | mac | `:tomac "AA:BB:CC:00:11:22"` |
| `:tobool` | any | bool | `:tobool "true"` → true |
| `:totime` | str | time | `:totime "1h30m"` → 1h30m |
| `:toarray` | str | array | `:toarray "a,b,c"` → array |

### ตัวอย่าง Type Conversion

```routeros
# String to Number
:local str "100"
:local num [:tonum $str]
:put ($num * 2)    # 200

# Number to String (สำหรับ Concatenation)
:local num 42
:local msg ("The answer is " . [:tostr $num])
:put $msg    # The answer is 42

# String to IP
:local ipStr "192.168.1.1"
:local ip [:toip $ipStr]
:if ($ip in 192.168.0.0/16) do={
    :put "Private IP"
}

# Boolean Conversion
:local boolStr "true"
:local b [:tobool $boolStr]
:if ($b) do={
    :put "It's true!"
}

# Time Conversion
:local timeStr "1h30m"
:local t [:totime $timeStr]
:put $t    # 1h30m
```

### Implicit Conversion

```routeros
# RouterOS ทำ Implicit Conversion บางกรณี

# Number + String (String จะถูกแปลงเป็น Number ถ้าเป็นไปได้)
:local a 10
:local b "5"
:put ($a + $b)    # 15 (String "5" ถูกแปลงเป็น Number)

# แต่ควรทำ Explicit Conversion เพื่อความชัดเจน
:put ($a + [:tonum $b])    # 15 (ชัดเจนกว่า)
```

---

## Variable Scope

### Local Scope

```routeros
# Local Variable จำกัดอยู่ใน Block { } ที่ประกาศ
:local x 10

{
    :local x 20   # x ใหม่ใน Block นี้
    :put $x    # 20
}

:put $x    # 10 (x ใน Block หลักยังเป็น 10)
```

### Global Scope

```routeros
# Global Variable ใช้ได้ทุกที่
:global counter 0

# Script A
:global counter
:set counter ($counter + 1)
:log info ("Counter: " . $counter)

# Script B (รันหลัง Script A)
:global counter
:put ("Counter from Script A: " . $counter)
```

### ตัวอย่าง Scope

```routeros
:local outer "outer"

:do {
    :local inner "inner"
    :put $outer    # ใช้ได้ - เห็น outer
    :put $inner    # ใช้ได้ - เห็น inner
}

:put $outer    # ใช้ได้
# :put $inner  # ERROR! - inner ไม่มีใน scope นี้
```

---

## Constants

### Simulating Constants ใน RouterOS

RouterOS ไม่มี `const` keyword แต่ทำได้โดยใช้ Convention:

```routeros
# ใช้ชื่อตัวพิมพ์ใหญ่เพื่อบ่งบอกว่าเป็น Constant
:local MAX_RETRIES 3
:local TIMEOUT 30
:local DEFAULT_GATEWAY "192.168.1.1"
:local VERSION "1.0.0"

# ใช้งาน
:local retries 0
:while ($retries < $MAX_RETRIES) do={
    # Try something
    :set retries ($retries + 1)
}
```

### Global Constants

```routeros
# กำหนด Global Constants ครั้งเดียว
# (ใส่ใน Startup Script)
:global CFG_BACKUP_SERVER "192.168.1.100"
:global CFG_ADMIN_EMAIL "admin@example.com"
:global CFG_CHECK_INTERVAL 60

# ใช้งานใน Scripts ต่างๆ
:global CFG_BACKUP_SERVER
:put $CFG_BACKUP_SERVER
```

---

## Special Variables

### Built-in Special Variables

```routeros
# $nothing - Nil value
:local x $nothing
:put [:typeof $x]    # nothing

# $true / $false - Boolean Constants
:local a $true
:local b $false

# $0 - Script Path (ถ้ามี)

# [:timestamp] - ดู Timestamp ปัจจุบัน (ไม่ใช่ Variable แต่ Function)
:put [/system clock get date]
:put [/system clock get time]
```

### Environment Variables

```routeros
# ดู Environment Variables ทั้งหมด
/system script environment print

# Variables ที่สร้างโดย Scheduler
# $date, $time, etc. (ขึ้นกับ Event)
```

---

## Nil/Nothing Handling

### ตรวจสอบ Nil

```routeros
# วิธีที่ 1: เปรียบเทียบกับ $nothing
:local x
:if ($x = $nothing) do={
    :put "x is nil"
}

# วิธีที่ 2: ตรวจสอบ Type
:if ([:typeof $x] = "nothing") do={
    :put "x is nothing type"
}

# วิธีที่ 3: ตรวจสอบ Length (สำหรับ String/Array)
:local arr [/ip address find]
:if ([:len $arr] = 0) do={
    :put "No IP addresses found"
}
```

### Default Value Pattern

```routeros
# Pattern: ใช้ค่า Default ถ้า Variable เป็น nil
:local hostname [/system identity get name]
:if ([:typeof $hostname] = "nothing") do={
    :set hostname "Unknown"
}
:put ("Router: " . $hostname)

# หรือใช้ Conditional Expression
:local hostname [/system identity get name]
:local displayName
:if ($hostname != $nothing) do={
    :set displayName $hostname
} else={
    :set displayName "Unknown"
}
```

---

## Type Checking

### :typeof Function

```routeros
# ตรวจสอบ Type ของ Variable
:local str "hello"
:local num 42
:local bool true
:local ip 192.168.1.1
:local arr {1;2;3}

:put [:typeof $str]    # str
:put [:typeof $num]    # num
:put [:typeof $bool]   # bool
:put [:typeof $ip]     # ip
:put [:typeof $arr]    # array

# ตรวจสอบก่อนใช้
:local x "not a number"
:if ([:typeof $x] = "num") do={
    :put ($x * 2)
} else={
    :put "Not a number, cannot multiply"
}
```

### ตรวจสอบ Type ก่อน Conversion

```routeros
# Safe Conversion Pattern
:local input "42abc"

:local converted
:do {
    :set converted [:tonum $input]
} on-error={
    :log warning ("Cannot convert to number: " . $input)
    :set converted 0  # Default value
}

:put ("Converted: " . $converted)
```

---

## Variable Naming Conventions

### Best Practices

```routeros
# camelCase สำหรับ Variables ทั่วไป
:local serverAddress "192.168.1.100"
:local maxRetryCount 3
:local isBackupEnabled true

# UPPER_CASE สำหรับ Constants
:local MAX_CONNECTIONS 100
:local DEFAULT_TIMEOUT 30

# ชื่อที่มีความหมาย
:local routerName [/system identity get name]    # ดี
:local r [/system identity get name]             # ไม่ดี

# Prefix บอก Type (optional)
:local strName "John"        # str prefix
:local numAge 25             # num prefix
:local boolActive true       # bool prefix
:local ipServer 192.168.1.1  # ip prefix
:local arrItems {1;2;3}      # arr prefix
```

### ตัวอย่าง Naming ที่ดี

```routeros
# ชื่อที่ดี - อธิบายตัวเอง
:local primaryDnsServer 8.8.8.8
:local backupDnsServer 8.8.4.4
:local maxFailoverAttempts 3
:local currentBandwidthMbps 100
:local isMainLinkActive true

# ชื่อที่ไม่ดี - ไม่ชัดเจน
:local dns1 8.8.8.8
:local x 3
:local bw 100
:local flag true
```

---

## ตัวอย่าง Scripts ที่ครอบคลุม Data Types

### Script แสดง Router Info ทุก Type

```routeros
# ตัวอย่าง Script ที่ใช้ Data Types หลากหลาย

# String
:local routerName [/system identity get name]
:local routerOS [/system resource get version]

# Number
:local cpuLoad [/system resource get cpu-load]
:local freeMemory [/system resource get free-memory]
:local totalMemory [/system resource get total-memory]

# Time
:local uptime [/system resource get uptime]

# Boolean (derived)
:local isOverloaded ($cpuLoad > 80)

# IP (from first interface)
:local firstIP [/ip address get [find] address]

# Array (list of interfaces)
:local interfaces [/interface find]

# Display Summary
:put "=== Router Summary ==="
:put ("Name:       " . $routerName)
:put ("OS:         " . $routerOS)
:put ("CPU:        " . $cpuLoad . "%")
:put ("Memory:     " . (($freeMemory / 1024) . " KB free"))
:put ("Uptime:     " . $uptime)
:put ("Overloaded: " . $isOverloaded)
:put ("Interfaces: " . [:len $interfaces])
:put "===================="

# Log สรุป
:log info ("Router " . $routerName . " - CPU: " . $cpuLoad . "%, Overloaded: " . $isOverloaded)
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 17:

| Data Type | Declaration | ตัวอย่าง | ใช้สำหรับ |
|-----------|------------|---------|---------|
| str | `:local s "text"` | "Hello" | ข้อความ |
| num | `:local n 42` | 42, 3.14 | ตัวเลข |
| bool | `:local b true` | true, false | เงื่อนไข |
| ip | `:local ip 192.168.1.1` | 192.168.1.1 | IP Address |
| ip-prefix | `:local net 192.168.1.0/24` | 10.0.0.0/8 | Network |
| mac | `:local mac AA:BB:CC:DD:EE:FF` | MAC Address | Hardware ID |
| array | `:local arr {1;2;3}` | {a;b;c} | List |
| nothing | (ไม่กำหนดค่า) | - | Nil/Empty |
| time | `:local t 1h30m` | 1h, 30m, 15s | Duration |
| id | (จาก `/find`) | *1, *2 | Object ID |

---

## แบบทดสอบ

1. ความแตกต่างระหว่าง `:local` และ `:global` คืออะไร?
2. ถ้าต้องการ Concatenate Number กับ String ต้องทำอย่างไร?
3. `[:typeof x]` ใช้ทำอะไร?
4. วิธีตรวจสอบว่า IP Address อยู่ใน Subnet ได้อย่างไร?
5. Array ใน RouterOS เริ่ม Index ที่เท่าไร?

---

## Navigation

[← Part 16: Intro to Scripting](part-016-intro-to-scripting.md) | [Part 18: Operators & Expressions →](part-018-operators-expressions.md)

---

*MikroTik RouterOS Administration Course - Part 17*
*Last Updated: 2026*
