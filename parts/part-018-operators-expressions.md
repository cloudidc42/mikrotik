# Part 18: Operators & Expressions

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate-Advanced
**เวลาเรียน:** 3-4 ชั่วโมง

---

## สารบัญ

- [Arithmetic Operators](#arithmetic-operators)
- [Comparison Operators](#comparison-operators)
- [Logical Operators](#logical-operators)
- [String Operators](#string-operators)
- [Bitwise Operators](#bitwise-operators)
- [IP/Prefix Operators](#ip-prefix-operators)
- [Operator Precedence](#operator-precedence)
- [Expressions ใน Conditions](#expressions-in-conditions)
- [Type Coercion](#type-coercion)
- [Examples ที่ใช้บ่อย](#common-examples)

---

## Arithmetic Operators

### ตัวดำเนินการทางคณิตศาสตร์

| Operator | ความหมาย | ตัวอย่าง | ผลลัพธ์ |
|----------|---------|---------|--------|
| `+` | บวก | `5 + 3` | 8 |
| `-` | ลบ | `10 - 4` | 6 |
| `*` | คูณ | `4 * 5` | 20 |
| `/` | หาร (Integer) | `10 / 3` | 3 |
| `%` | โมดูลัส (เศษ) | `10 % 3` | 1 |

### ตัวอย่างการใช้งาน

```routeros
# Arithmetic พื้นฐาน
:local a 10
:local b 3

:put ($a + $b)    # 13
:put ($a - $b)    # 7
:put ($a * $b)    # 30
:put ($a / $b)    # 3 (Integer division)
:put ($a % $b)    # 1 (Remainder)

# สูตรคำนวณทั่วไป
:local price 100
:local qty 5
:local discount 10    # 10%
:local total ($price * $qty)
:local discountAmt (($total * $discount) / 100)
:local finalPrice ($total - $discountAmt)

:put ("Total: " . $total)
:put ("Discount: " . $discountAmt)
:put ("Final: " . $finalPrice)
```

### Arithmetic กับ Bandwidth

```routeros
# คำนวณ Bandwidth
:local rxBytes [/interface get ether1 rx-byte]
:local txBytes [/interface get ether1 tx-byte]
:local totalBytes ($rxBytes + $txBytes)

# แปลงเป็น KB, MB, GB
:local kb ($totalBytes / 1024)
:local mb ($kb / 1024)
:local gb ($mb / 1024)

:put ("Total: " . $gb . " GB")
```

### Arithmetic กับ Memory

```routeros
# คำนวณ Memory Usage
:local totalMem [/system resource get total-memory]
:local freeMem [/system resource get free-memory]
:local usedMem ($totalMem - $freeMem)
:local usedPct (($usedMem * 100) / $totalMem)

:put ("Memory: " . $usedPct . "% used")
:put ("Free: " . ($freeMem / 1048576) . " MB")
:put ("Used: " . ($usedMem / 1048576) . " MB")
```

### Integer vs Float Division

```routeros
# Integer Division ตัดทศนิยมทิ้ง
:put (7 / 2)    # 3 (ไม่ใช่ 3.5)
:put (1 / 3)    # 0 (ไม่ใช่ 0.333...)

# เพื่อได้ทศนิยม ต้องคูณก่อน
:local result ((7 * 100) / 2)
:put ($result . "%")    # 350% - แล้วค่อยหาร 100 ตอนแสดงผล

# Percentage ที่ถูกต้อง
:local used 7
:local total 10
:local pct (($used * 100) / $total)
:put ($pct . "%")    # 70%
```

---

## Comparison Operators

### ตัวดำเนินการเปรียบเทียบ

| Operator | ความหมาย | ตัวอย่าง | ผลลัพธ์ |
|----------|---------|---------|--------|
| `=` | เท่ากับ | `5 = 5` | true |
| `!=` | ไม่เท่ากับ | `5 != 3` | true |
| `<` | น้อยกว่า | `3 < 5` | true |
| `>` | มากกว่า | `5 > 3` | true |
| `<=` | น้อยกว่าหรือเท่า | `5 <= 5` | true |
| `>=` | มากกว่าหรือเท่า | `5 >= 6` | false |
| `~` | Pattern Match (Regex) | `"hello" ~ "he"` | true |
| `!~` | ไม่ Match Pattern | `"hello" !~ "xyz"` | true |
| `in` | อยู่ใน Set/Range | `ip in prefix` | bool |

### ตัวอย่างการใช้งาน

```routeros
:local x 10
:local y 20

:put ($x = y)     # false
:put ($x != $y)   # true
:put ($x < $y)    # true
:put ($x > $y)    # false
:put ($x <= $y)   # true
:put ($x >= $y)   # false
:put ($x = 10)    # true
```

### เปรียบเทียบ String

```routeros
:local a "hello"
:local b "HELLO"
:local c "hello"

:put ($a = $b)    # false (Case Sensitive)
:put ($a = $c)    # true

# Pattern Matching (~)
:local ssid "Office-WiFi-2G"
:if ($ssid ~ "Office") do={
    :put "This is an Office SSID"
}

:if ($ssid ~ "^Office.*2G\$") do={
    :put "Office 2.4GHz SSID"
}
```

### เปรียบเทียบ IP

```routeros
:local ip1 192.168.1.1
:local ip2 192.168.1.2

:put ($ip1 = $ip2)    # false
:put ($ip1 < $ip2)    # true (IP เปรียบเทียบตาม Numeric Value)
:put ($ip1 > $ip2)    # false
```

### เปรียบเทียบ Time

```routeros
:local t1 1h
:local t2 30m
:local t3 90m

:put ($t1 > $t2)    # true (1h > 30m)
:put ($t1 = $t3)    # true (1h = 60m = 90... wait, 1h=60m, 90m>1h)
:put ($t3 > $t1)    # true (90m > 60m = 1h)
```

---

## Logical Operators

### ตัวดำเนินการ Logic

| Operator | ความหมาย | ตัวอย่าง | ผลลัพธ์ |
|----------|---------|---------|--------|
| `&&` | AND | `true && false` | false |
| `\|\|` | OR | `true \|\| false` | true |
| `!` | NOT | `!true` | false |

### Truth Table

```
AND (&&):
  true  && true  = true
  true  && false = false
  false && true  = false
  false && false = false

OR (||):
  true  || true  = true
  true  || false = true
  false || true  = true
  false || false = false

NOT (!):
  !true  = false
  !false = true
```

### ตัวอย่างการใช้งาน

```routeros
:local a true
:local b false
:local c true

:put ($a && $b)    # false
:put ($a || $b)    # true
:put (!$a)         # false
:put ($a && $c)    # true
:put (!$b && $c)   # true

# Compound Conditions
:local age 25
:local hasLicense true

:if ($age >= 18 && $hasLicense) do={
    :put "Can drive"
}

# Multiple Conditions
:local x 15
:if ($x > 10 && $x < 20) do={
    :put "x is between 10 and 20"
}
```

### Logical ใน Network Conditions

```routeros
# ตรวจสอบ Interface Status
:local eth1Up [/interface get ether1 running]
:local eth2Up [/interface get ether2 running]

:if ($eth1Up && $eth2Up) do={
    :log info "Both uplinks are UP"
} else={
    :if (!$eth1Up && !$eth2Up) do={
        :log error "Both uplinks are DOWN!"
    } else={
        :log warning "One uplink is DOWN"
    }
}

# ตรวจสอบ CPU Load และ Memory
:local cpuLoad [/system resource get cpu-load]
:local freeMem [/system resource get free-memory]

:if ($cpuLoad > 90 || $freeMem < 10000000) do={
    :log warning "System resources critical!"
}
```

---

## String Operators

### Concatenation

```routeros
# ต่อ String ด้วย . (dot)
:local a "Hello"
:local b "World"
:local c ($a . ", " . $b . "!")
:put $c    # Hello, World!

# ต่อ String กับ Number
:local count 5
:local msg ("Found " . $count . " items")
:put $msg    # Found 5 items

# ต่อหลาย Variables
:local firstName "John"
:local lastName "Doe"
:local age 30
:local info ($firstName . " " . $lastName . " (Age: " . $age . ")")
:put $info
```

### String Pattern Matching

```routeros
# ~ = Match Pattern (Regex)
:local str "192.168.1.100"

# เช็คว่าเป็น IP format
:if ($str ~ "^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+\$") do={
    :put "Valid IP format"
}

# เช็คว่ามี keyword
:local log "Error: Connection failed"
:if ($log ~ "[Ee]rror") do={
    :put "Log contains error"
}

# !~ = Not Match
:local filename "backup.rsc"
:if ($filename !~ "\\.backup\$") do={
    :put "Not a backup file"
}
```

### String ที่ใช้บ่อย

```routeros
# [:len] - ความยาว String
:local str "Hello"
:put [:len $str]    # 5

# [:pick] - ดึง Substring
:local str "Hello, World!"
:put [:pick $str 0 5]      # Hello
:put [:pick $str 7 12]     # World

# [:find] - หาตำแหน่ง
:local str "Hello, World!"
:local pos [:find $str ","]
:put $pos    # 5

# ดึง Domain จาก Email
:local email "admin@example.com"
:local atPos [:find $email "@"]
:local domain [:pick $email ($atPos + 1) [:len $email]]
:put $domain    # example.com
```

### in Operator สำหรับ String

```routeros
# ตรวจสอบว่า String มี Substring
:local str "Hello, World!"
:if ("World" in $str) do={
    :put "Contains World"
}

# ตรวจสอบว่า IP อยู่ใน List
:local allowedIPs {"192.168.1.10"; "192.168.1.20"; "10.0.0.1"}
:local clientIP 192.168.1.10

:if ($clientIP in $allowedIPs) do={
    :put "IP is allowed"
}
```

---

## Bitwise Operators

### ตัวดำเนินการ Bitwise

| Operator | ความหมาย | ตัวอย่าง |
|----------|---------|---------|
| `&` | Bitwise AND | `0xFF & 0x0F` = `0x0F` |
| `\|` | Bitwise OR | `0xF0 \| 0x0F` = `0xFF` |
| `^` | Bitwise XOR | `0xFF ^ 0x0F` = `0xF0` |
| `~` | Bitwise NOT | `~0` = all 1s |
| `<<` | Left Shift | `1 << 4` = 16 |
| `>>` | Right Shift | `16 >> 4` = 1 |

### ตัวอย่างการใช้งาน

```routeros
# Bitwise AND
:local a 0xFF    # 11111111
:local b 0x0F    # 00001111
:local result ($a & $b)
:put $result    # 15 (00001111)

# Bitwise OR
:local c 0xF0   # 11110000
:local d 0x0F   # 00001111
:put ($c | $d)  # 255 (11111111)

# Bitwise XOR
:local e 0xFF   # 11111111
:local f 0x0F   # 00001111
:put ($e ^ $f)  # 240 (11110000)

# Left Shift (คูณด้วย 2^n)
:put (1 << 8)   # 256 (1 * 2^8)

# Right Shift (หารด้วย 2^n)
:put (256 >> 4) # 16 (256 / 2^4)
```

### Bitwise สำหรับ IP/Network

```routeros
# คำนวณ Network Address จาก IP + Mask
:local ip 192.168.1.100
:local mask 255.255.255.0

# Network Address = IP & Mask
:local network ($ip & $mask)
:put $network    # 192.168.1.0

# Broadcast = Network | ~Mask
:local invMask (~$mask)
:local broadcast ($network | $invMask)
:put $broadcast    # 192.168.1.255
```

---

## IP/Prefix Operators

### IP Operators

```routeros
# Arithmetic บน IP
:local ip 192.168.1.1
:put ($ip + 1)     # 192.168.1.2
:put ($ip + 10)    # 192.168.1.11
:put ($ip - 1)     # 192.168.1.0

# เปรียบเทียบ IP
:local ip1 192.168.1.1
:local ip2 192.168.2.1

:if ($ip1 < $ip2) do={
    :put "ip1 is lower"
}

# ตรวจสอบว่า IP อยู่ใน Prefix
:local clientIP 192.168.1.50
:local network 192.168.1.0/24

:if ($clientIP in $network) do={
    :put "Client is in LAN"
}
```

### Prefix/Subnet Operations

```routeros
# ตรวจสอบ Bogon Networks
:local testIP 192.168.100.50

:local bogons {
    "10.0.0.0/8";
    "172.16.0.0/12";
    "192.168.0.0/16";
    "127.0.0.0/8";
    "0.0.0.0/8";
    "100.64.0.0/10"
}

:local isBogon false
:foreach net in=$bogons do={
    :if ($testIP in [:toip-prefix $net]) do={
        :set isBogon true
    }
}

:if ($isBogon) do={
    :put ($testIP . " is a private/bogon IP")
} else={
    :put ($testIP . " is a public IP")
}
```

### Range Operations

```routeros
# สร้าง IP Range
:local startIP 192.168.1.100
:local count 10

:for i from=0 to=($count - 1) do={
    :local ip ($startIP + $i)
    :put $ip
}

# Output:
# 192.168.1.100
# 192.168.1.101
# ...
# 192.168.1.109
```

---

## Operator Precedence

### ลำดับความสำคัญ (สูงไปต่ำ)

```
1. () Parentheses (วงเล็บ - สูงสุด)
2. ! (NOT)
3. ~ << >> (Bitwise Shift)
4. & (Bitwise AND)
5. ^ (Bitwise XOR)
6. | (Bitwise OR)
7. * / % (Multiply, Divide, Modulus)
8. + - (Add, Subtract)
9. . (String Concatenation)
10. < > <= >= (Comparison)
11. = != ~ !~ in (Equality, Pattern)
12. && (Logical AND)
13. || (Logical OR)  - ต่ำที่สุด
```

### ตัวอย่าง Precedence

```routeros
# ไม่มีวงเล็บ - ใช้ Precedence
:put (2 + 3 * 4)    # 14 (3*4=12, 2+12=14)

# ใช้วงเล็บ - บังคับ Order
:put ((2 + 3) * 4)  # 20 (2+3=5, 5*4=20)

# Logical Precedence
:local a true
:local b false
:local c true

# && มีความสำคัญสูงกว่า ||
:put ($a || $b && $c)    # true ($b && $c = false, $a || false = true)
:put (($a || $b) && $c)  # true (($a || $b) = true, true && $c = true)

# แนะนำให้ใส่วงเล็บเสมอเพื่อความชัดเจน
:if (($cpuLoad > 80) || ($freeMem < 5000000)) do={
    :log warning "System resources critical"
}
```

### ข้อผิดพลาดที่พบบ่อย

```routeros
# ผิด: ขาดวงเล็บ
:put 2 + 3 * 4      # Error!

# ถูก: ใส่วงเล็บ
:put (2 + 3 * 4)    # 14

# ผิด: ลืม $ สำหรับ Variable
:put x + 1          # Error!

# ถูก: ใส่ $ ก่อน Variable
:put ($x + 1)
```

---

## Expressions ใน Conditions

### if Statement

```routeros
# รูปแบบ if
:if (condition) do={
    # code when true
}

# if-else
:if (condition) do={
    # code when true
} else={
    # code when false
}

# if-elseif-else
:if (condition1) do={
    # code for condition1
} else={
    :if (condition2) do={
        # code for condition2
    } else={
        # default code
    }
}
```

### Complex Conditions

```routeros
# หลาย Conditions
:local cpuLoad [/system resource get cpu-load]
:local memFree [/system resource get free-memory]
:local uptime [/system resource get uptime]

# Compound condition
:if (($cpuLoad > 90) && ($memFree < 10000000)) do={
    :log error "System is critically overloaded!"
}

# Complex condition with multiple checks
:local isWorkHour ($uptime > 8h && $uptime < 17h)
:local isWeekend false  # สมมติ

:if (!$isWeekend && $isWorkHour) do={
    :put "During work hours on weekday"
}
```

### Condition ใน :while

```routeros
# Loop จนกว่า Condition จะเป็น false
:local x 0
:while ($x < 10) do={
    :put $x
    :set x ($x + 1)
}

# Loop พร้อม Multiple Conditions
:local retries 0
:local success false
:while (!$success && $retries < 3) do={
    # Try something
    :local result [/ping 8.8.8.8 count=1 as-value]
    :if (($result->"received") > 0) do={
        :set success true
    } else={
        :set retries ($retries + 1)
        :delay 2s
    }
}
```

### Ternary Expression Pattern

```routeros
# RouterOS ไม่มี Ternary Operator (condition ? a : b)
# แต่ทำได้ผ่าน Function

:local ternary do={
    :if ($1) do={ :return $2 } else={ :return $3 }
}

:local x 10
:local result [$ternary ($x > 5) "big" "small"]
:put $result    # big
```

---

## Type Coercion

### Automatic Type Conversion

```routeros
# RouterOS บางครั้งทำ Implicit Conversion

# String + Number → Number
:local n 10
:local s "5"
:put ($n + $s)    # 15 (String "5" แปลงเป็น Number)

# Boolean in Arithmetic
:put (true + 0)    # 1
:put (false + 0)   # 0

# IP + Number
:local ip 192.168.1.0
:put ($ip + 1)    # 192.168.1.1

# Number ใน Boolean Context
:if (1) do={ :put "1 is true" }     # true
:if (0) do={ :put "zero" } else={ :put "0 is false" }  # false
```

### Explicit Conversion ที่แนะนำ

```routeros
# ทำ Explicit Conversion เพื่อความชัดเจน

# String to Number
:local strNum "42"
:local num [:tonum $strNum]

# Number to String
:local n 100
:local str [:tostr $n]

# IP to String
:local ip 192.168.1.1
:local ipStr [:tostr $ip]

# String to IP
:local ipStr "192.168.1.1"
:local ip [:toip $ipStr]

# Boolean to Number
:local b true
:local n [:tonum $b]    # 1 or 0
```

### Coercion Pitfalls

```routeros
# ระวัง: String ที่ไม่ใช่ตัวเลขจะเกิด Error เมื่อ Convert เป็น Number
:local badStr "abc"
:do {
    :local num [:tonum $badStr]
} on-error={
    :put "Cannot convert 'abc' to number"
}

# ระวัง: Empty String ไม่เท่ากับ 0
:local empty ""
:put ($empty = 0)    # false!
:put ($empty = "")   # true

# Safe Conversion
:local input "42abc"
:local isNum ($input ~ "^[0-9]+\$")
:if ($isNum) do={
    :local n [:tonum $input]
    :put ($n * 2)
} else={
    :put "Not a pure number"
}
```

---

## Common Examples

### ตัวอย่างที่ใช้บ่อยในชีวิตจริง

### 1. คำนวณ Bandwidth Usage

```routeros
:local rxPrev 0
:local txPrev 0
:delay 1s
:local rxNow [/interface get ether1 rx-byte]
:local txNow [/interface get ether1 tx-byte]

:local rxBps ($rxNow - $rxPrev)
:local txBps ($txNow - $txPrev)

:put ("Download: " . ($rxBps / 1024) . " KB/s")
:put ("Upload: " . ($txBps / 1024) . " KB/s")
```

### 2. ตรวจสอบ IP Range

```routeros
# ตรวจว่า IP อยู่ใน Private Range
:local testIP 192.168.1.50

:if (($testIP in 10.0.0.0/8) || ($testIP in 172.16.0.0/12) || ($testIP in 192.168.0.0/16)) do={
    :put ($testIP . " is a Private IP")
} else={
    :put ($testIP . " is a Public IP")
}
```

### 3. Time-based Conditions

```routeros
# ทำงานต่างกันตามเวลา
:local hour [:tonum [:pick [/system clock get time] 0 2]]
:local minute [:tonum [:pick [/system clock get time] 3 5]]

:if ($hour >= 9 && $hour < 18) do={
    :put "Business hours"
} else={
    :put "After hours"
}

# วันในสัปดาห์
:local dayOfWeek [/system clock get day-of-week]
:if ($dayOfWeek = "sat" || $dayOfWeek = "sun") do={
    :put "Weekend"
} else={
    :put "Weekday"
}
```

### 4. String Manipulation

```routeros
# แยก Hostname จาก FQDN
:local fqdn "router.branch1.example.com"
:local dotPos [:find $fqdn "."]
:local hostname [:pick $fqdn 0 $dotPos]
:put ("Hostname: " . $hostname)    # router

# แยก Extension จาก Filename
:local filename "backup-2026.tar.gz"
:local dotPos [:find $filename "." [:len $filename] true]  # หา . จากหลัง
:local ext [:pick $filename $dotPos [:len $filename]]
:put ("Extension: " . $ext)    # .gz

# ตรวจสอบ Email Format
:local email "admin@example.com"
:if ($email ~ "^[a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\\.[a-zA-Z]{2,}\$") do={
    :put "Valid email format"
}
```

### 5. Network Calculations

```routeros
# คำนวณ Subnet
:local ip 192.168.1.100
:local prefix 24
:local mask (0xFFFFFFFF << (32 - $prefix))

# Network Address
:local network ($ip & $mask)
:put ("Network: " . $network)

# Subnet Mask
:local subnetMask [:toip $mask]
:put ("Mask: " . $subnetMask)

# Broadcast
:local broadcast ($network | (~$mask))
:put ("Broadcast: " . $broadcast)
```

### 6. Monitoring with Thresholds

```routeros
# Monitor และ Alert ตาม Threshold
:local thresholds {
    "cpu"=80;
    "memory"=90;
    "disk"=85
}

:local cpuLoad [/system resource get cpu-load]
:local freeMem [/system resource get free-memory]
:local totalMem [/system resource get total-memory]
:local memUsed (100 - (($freeMem * 100) / $totalMem))

# ตรวจสอบแต่ละ Threshold
:if ($cpuLoad > ($thresholds->"cpu")) do={
    :log warning ("CPU overloaded: " . $cpuLoad . "%")
}

:if ($memUsed > ($thresholds->"memory")) do={
    :log warning ("Memory high: " . $memUsed . "%")
}
```

### 7. Pattern-based Configuration

```routeros
# กำหนดค่าตาม Pattern
:local interfaceName "ether1"

:if ($interfaceName ~ "^ether") do={
    :put "Ethernet Interface - apply Ethernet rules"
} else={
    :if ($interfaceName ~ "^wlan") do={
        :put "Wireless Interface - apply Wireless rules"
    } else={
        :if ($interfaceName ~ "^bridge") do={
            :put "Bridge Interface"
        }
    }
}
```

### 8. Bitwise สำหรับ Flags

```routeros
# ใช้ Bits เป็น Flags (เช่น Feature Flags)
:local FEATURE_BACKUP 1      # Bit 0
:local FEATURE_MONITOR 2     # Bit 1
:local FEATURE_NOTIFY 4      # Bit 2
:local FEATURE_CLEANUP 8     # Bit 3

# เปิด Features ที่ต้องการ
:local enabledFeatures ($FEATURE_BACKUP | $FEATURE_MONITOR | $FEATURE_NOTIFY)

# ตรวจสอบว่า Feature เปิดอยู่ไหม
:if (($enabledFeatures & $FEATURE_BACKUP) != 0) do={
    :put "Backup is enabled"
}

:if (($enabledFeatures & $FEATURE_CLEANUP) != 0) do={
    :put "Cleanup is enabled"
} else={
    :put "Cleanup is disabled"
}
```

---

## แบบฝึกหัด

```routeros
# Exercise 1: คำนวณ Circle
# พื้นที่วงกลม = π * r²
:local radius 5
:local pi 314    # 3.14 * 100
:local area (($pi * $radius * $radius) / 100)
:put ("Area: " . $area)

# Exercise 2: ตรวจสอบ Even/Odd
:local n 42
:if (($n % 2) = 0) do={
    :put ($n . " is even")
} else={
    :put ($n . " is odd")
}

# Exercise 3: Swap ค่าสอง Variables
:local x 10
:local y 20
:local temp $x
:set x $y
:set y $temp
:put ("x=" . $x . ", y=" . $y)    # x=20, y=10

# Exercise 4: ตรวจสอบ Leap Year
:local year 2024
:if ((($year % 4) = 0 && ($year % 100) != 0) || ($year % 400) = 0) do={
    :put ($year . " is a leap year")
} else={
    :put ($year . " is not a leap year")
}

# Exercise 5: IP Address Validation
:local ip "192.168.1.abc"
:if ($ip ~ "^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+\$") do={
    :put "Valid IP format"
} else={
    :put "Invalid IP format"
}
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 18:

| Operator Type | Symbols | ใช้สำหรับ |
|--------------|---------|---------|
| Arithmetic | `+ - * / %` | คำนวณตัวเลข |
| Comparison | `= != < > <= >= ~ in` | เปรียบเทียบค่า |
| Logical | `&& \|\| !` | Logic Boolean |
| String | `. ~ in` | จัดการ String |
| Bitwise | `& \| ^ ~ << >>` | จัดการ Bits |
| IP | `in + -` | คำนวณ IP/Network |

---

## Navigation

[← Part 17: Variables & Data Types](part-017-variables-datatypes.md) | [Part 19: Control Flow →](part-019-control-flow.md)

---

*MikroTik RouterOS Administration Course - Part 18*
*Last Updated: 2026*
