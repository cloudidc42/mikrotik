# Part 20: Functions & Procedures

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate-Advanced
**เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

- [:local Function Declaration](#function-declaration)
- [Function Parameters](#function-parameters)
- [Return Values](#return-values)
- [Recursive Functions](#recursive-functions)
- [Built-in Functions](#built-in-functions)
- [String Functions](#string-functions)
- [Array Functions](#array-functions)
- [Math Functions](#math-functions)
- [IP Functions](#ip-functions)
- [Function Best Practices](#best-practices)
- [Reusable Function Library](#function-library)

---

## Function Declaration

### สร้าง Function ใน RouterOS

RouterOS ใช้ `:local` Declaration สำหรับ Function ผ่าน `do={ }` Block:

```routeros
# รูปแบบพื้นฐาน
:local functionName do={
    # Function Body
}

# เรียกใช้งาน
$functionName
```

### ตัวอย่าง Function ง่ายๆ

```routeros
# Function พิมพ์ Hello
:local sayHello do={
    :put "Hello, World!"
}

# เรียกใช้
$sayHello
# Output: Hello, World!

# Function แสดง System Info
:local showInfo do={
    :local name [/system identity get name]
    :local ver [/system resource get version]
    :put ("Router: " . $name . " | RouterOS: " . $ver)
}

$showInfo
```

### Function ใน :global

```routeros
# Global Function - ใช้ได้ทุก Script
:global sendAlert do={
    :local msg $1
    :log warning $msg
    /tool e-mail send to="admin@example.com" subject="Alert" body=$msg
}

# เรียกใช้จาก Script อื่น
:global sendAlert
[$sendAlert "Router CPU is too high!"]
```

---

## Function Parameters

### การส่ง Parameters

RouterOS ส่ง Parameters ผ่าน `$1`, `$2`, `$3`, ... ตามลำดับ:

```routeros
# Function รับ 1 Parameter
:local greet do={
    :local name $1
    :put ("Hello, " . $name . "!")
}

$greet "Alice"    # Hello, Alice!
$greet "Bob"      # Hello, Bob!

# Function รับ 2 Parameters
:local add do={
    :local a $1
    :local b $2
    :put ($a + $b)
}

$add 10 20    # 30
$add 5 3      # 8
```

### ตัวอย่าง Multi-Parameter Functions

```routeros
# Function สร้าง IP Address Range
:local createIPRange do={
    :local baseIP $1
    :local start $2
    :local end $3

    :for i from=$start to=$end do={
        :put ([:tostr $baseIP] . [:tostr $i])
    }
}

$createIPRange 192.168.1. 10 20
# Output:
# 192.168.1.10
# 192.168.1.11
# ...
# 192.168.1.20
```

### Default Parameters Pattern

```routeros
# RouterOS ไม่มี Default Parameters โดยตรง
# แต่ทำได้ผ่านการตรวจสอบ

:local configure do={
    :local host $1
    :local port $2
    :local protocol $3

    # Default values
    :if ([:typeof $port] = "nothing" || $port = 0) do={
        :set port 22
    }
    :if ([:typeof $protocol] = "nothing" || [:len $protocol] = 0) do={
        :set protocol "ssh"
    }

    :put ("Connecting to " . $host . ":" . $port . " via " . $protocol)
}

$configure "192.168.1.1"          # host=192.168.1.1, port=22, protocol=ssh
$configure "192.168.1.1" 80       # host=192.168.1.1, port=80, protocol=ssh
$configure "192.168.1.1" 443 https  # host=192.168.1.1, port=443, protocol=https
```

### Named Parameters Pattern

```routeros
# Named Parameters ผ่าน Array
:local createUser do={
    :local params $1

    :local name ($params->"name")
    :local group ($params->"group")
    :local address ($params->"address")

    # Default values
    :if ([:typeof $group] = "nothing") do={ :set group "read" }
    :if ([:typeof $address] = "nothing") do={ :set address "0.0.0.0/0" }

    :put ("Creating user: " . $name . " (group=" . $group . ", address=" . $address . ")")
    # /user add name=$name group=$group address=$address
}

$createUser {"name"="alice"; "group"="full"}
$createUser {"name"="bob"; "group"="read"; "address"="192.168.1.0/24"}
$createUser {"name"="charlie"}
```

---

## Return Values

### Return Value ด้วย :return

```routeros
# Function Return Value ด้วย :return
:local square do={
    :local n $1
    :return ($n * $n)
}

:local result [$square 5]
:put $result    # 25

:local r2 [$square 10]
:put $r2    # 100
```

### Return หลายค่า (ผ่าน Array)

```routeros
# Return หลายค่าโดยใช้ Array
:local divmod do={
    :local a $1
    :local b $2
    :return {
        "quotient"=($a / $b);
        "remainder"=($a % $b)
    }
}

:local result [$divmod 17 5]
:put ("Quotient: " . ($result->"quotient"))    # 3
:put ("Remainder: " . ($result->"remainder"))  # 2
```

### ตัวอย่าง Functions ที่ Return ค่าสำคัญ

```routeros
# Return IP ของ Interface
:local getIfaceIP do={
    :local iface $1
    :local found [/ip address find interface=$iface]
    :if ([:len $found] = 0) do={
        :return "0.0.0.0"
    }
    :return [/ip address get ($found->0) address]
}

:local ip [$getIfaceIP "ether1"]
:put ("ether1 IP: " . $ip)

# Return Boolean (True/False)
:local isInterfaceUp do={
    :local iface $1
    :local found [/interface find name=$iface running=yes]
    :return ([:len $found] > 0)
}

:local eth1Up [$isInterfaceUp "ether1"]
:if ($eth1Up) do={
    :put "ether1 is UP"
}
```

### Early Return

```routeros
# ออกจาก Function ก่อนกำหนดด้วย :return
:local validateIP do={
    :local ip $1

    # Validate: ต้องไม่ว่าง
    :if ([:len [:tostr $ip]] = 0) do={
        :return "ERROR: Empty IP"
    }

    # Validate: ต้องเป็น Private IP
    :if (!($ip in 10.0.0.0/8) && !($ip in 172.16.0.0/12) && !($ip in 192.168.0.0/16)) do={
        :return "ERROR: Not a private IP"
    }

    # All good
    :return "OK"
}

:put [$validateIP 192.168.1.1]    # OK
:put [$validateIP 8.8.8.8]        # ERROR: Not a private IP
```

---

## Recursive Functions

### Recursive Function ใน RouterOS

```routeros
# Recursive Function: Factorial
:global factorial do={
    :local n $1
    :if ($n <= 1) do={
        :return 1
    }
    :global factorial
    :return ($n * [$factorial ($n - 1)])
}

:global factorial
:put [$factorial 5]     # 120 (5! = 5*4*3*2*1)
:put [$factorial 10]    # 3628800
```

### Recursive Fibonacci

```routeros
# Fibonacci Recursive (ช้าสำหรับตัวเลขสูง)
:global fib do={
    :local n $1
    :if ($n <= 1) do={
        :return $n
    }
    :global fib
    :return ([$fib ($n - 1)] + [$fib ($n - 2)])
}

:global fib
:put [$fib 10]    # 55
```

### Recursive File Search

```routeros
# ค้นหาไฟล์ใน List (Recursive Pattern)
:global findItem do={
    :local list $1
    :local target $2
    :local idx 0

    :global findItem

    :foreach item in=$list do={
        :if ($item = $target) do={
            :return $idx
        }
        :set idx ($idx + 1)
    }

    :return -1    # ไม่พบ
}

:global findItem
:local myList {"apple";"banana";"cherry";"date"}
:local pos [$findItem $myList "cherry"]
:put ("Found at index: " . $pos)    # 2
```

> **Warning:** Recursive Functions ใน RouterOS อาจ Stack Overflow ได้ถ้า Depth ลึกเกินไป ใช้ด้วยความระมัดระวัง

---

## Built-in Functions

### ตาราง Built-in Functions ทั้งหมด

| Function | คำอธิบาย | ตัวอย่าง |
|----------|---------|---------|
| `[:len x]` | ความยาว String/Array | `[:len "hello"]` = 5 |
| `[:typeof x]` | Type ของ Value | `[:typeof 42]` = "num" |
| `[:tostr x]` | Convert to String | `[:tostr 42]` = "42" |
| `[:tonum x]` | Convert to Number | `[:tonum "42"]` = 42 |
| `[:toip x]` | Convert to IP | `[:toip "192.168.1.1"]` |
| `[:toip-prefix x]` | Convert to IP Prefix | `[:toip-prefix "192.168.1.0/24"]` |
| `[:tomac x]` | Convert to MAC | `[:tomac "AA:BB:CC:DD:EE:FF"]` |
| `[:tobool x]` | Convert to Boolean | `[:tobool "true"]` = true |
| `[:totime x]` | Convert to Time | `[:totime "1h30m"]` |
| `[:toarray x]` | Convert to Array | `[:toarray "a,b,c"]` |
| `[:pick x s e]` | Substring/Slice | `[:pick "hello" 1 3]` = "el" |
| `[:find x y]` | Find in String | `[:find "hello" "ll"]` = 2 |
| `[:split x y]` | Split String | `[:split "a,b,c" ","]` |
| `[:char x]` | ASCII char | `[:char 65]` = "A" |
| `[:rndnum x]` | Random Number | `[:rndnum from=1 to=100]` |
| `[:rndstr x]` | Random String | `[:rndstr length=8]` |
| `[:environment]` | Get Env Variables | |
| `[:resolve x]` | DNS Resolve | `[:resolve "google.com"]` |
| `[:timestamp]` | Current Timestamp | |

---

## String Functions

### String ทุก Built-in

```routeros
# [:len] - Length
:local str "Hello, MikroTik!"
:put [:len $str]    # 16

# [:pick] - Substring
:put [:pick $str 0 5]    # Hello
:put [:pick $str 7]      # MikroTik! (จนถึงท้าย)

# [:find] - Find substring
:put [:find $str ","]    # 5
:put [:find $str "Tik"]  # 12

# [:split] - Split String
:local csv "a,b,c,d,e"
:local parts [:split $csv ","]
:foreach p in=$parts do={
    :put $p
}

# [:char] - Character by ASCII code
:put [:char 65]    # A
:put [:char 97]    # a
:put [:char 48]    # 0
```

### String Manipulation Functions

```routeros
# Function: Trim Whitespace
:local trim do={
    :local s $1
    # ลบ Spaces ต้น
    :while ([:pick $s 0 1] = " ") do={
        :set s [:pick $s 1 [:len $s]]
    }
    # ลบ Spaces ท้าย
    :while ([:pick $s ([:len $s]-1) [:len $s]] = " ") do={
        :set s [:pick $s 0 ([:len $s]-1)]
    }
    :return $s
}

:put [$trim "  hello world  "]    # "hello world"

# Function: To Upper Case (ASCII)
:local toUpper do={
    :local s $1
    :local result ""
    :for i from=0 to=([:len $s]-1) do={
        :local c [:pick $s $i ($i+1)]
        :local code [:find "abcdefghijklmnopqrstuvwxyz" $c]
        :if ($code != "") do={
            :set result ($result . [:char ($code + 65)])
        } else={
            :set result ($result . $c)
        }
    }
    :return $result
}

# Function: Repeat String
:local repeat do={
    :local str $1
    :local n $2
    :local result ""
    :for i from=1 to=$n do={
        :set result ($result . $str)
    }
    :return $result
}

:put [$repeat "-" 20]    # --------------------
:put [$repeat "AB" 5]    # ABABABABAB

# Function: Pad String
:local padLeft do={
    :local str [:tostr $1]
    :local width $2
    :local padChar $3
    :if ([:typeof $padChar] = "nothing") do={ :set padChar " " }

    :while ([:len $str] < $width) do={
        :set str ($padChar . $str)
    }
    :return $str
}

:put [$padLeft 42 8 "0"]     # 00000042
:put [$padLeft "Hi" 10 " "]  #         Hi

# Function: Replace String
:local replace do={
    :local str $1
    :local find $2
    :local replaceWith $3
    :local result ""
    :local pos [:find $str $find]

    :while ($pos != "") do={
        :set result ($result . [:pick $str 0 $pos] . $replaceWith)
        :set str [:pick $str ($pos + [:len $find]) [:len $str]]
        :set pos [:find $str $find]
    }
    :set result ($result . $str)
    :return $result
}

:put [$replace "Hello World" "World" "MikroTik"]    # Hello MikroTik
```

---

## Array Functions

### Array Built-in Operations

```routeros
# Array Length
:local arr {10; 20; 30; 40; 50}
:put [:len $arr]    # 5

# Access by Index
:put ($arr->0)    # 10
:put ($arr->4)    # 50

# Array Concatenation
:local a {1; 2; 3}
:local b {4; 5; 6}
:local c ($a, $b)
:put $c    # 1;2;3;4;5;6
```

### Custom Array Functions

```routeros
# Function: Array Contains
:local contains do={
    :local arr $1
    :local target $2

    :foreach item in=$arr do={
        :if ($item = $target) do={
            :return true
        }
    }
    :return false
}

:local fruits {"apple"; "banana"; "cherry"}
:put [$contains $fruits "banana"]    # true
:put [$contains $fruits "grape"]     # false

# Function: Array Index Of
:local indexOf do={
    :local arr $1
    :local target $2
    :local idx 0

    :foreach item in=$arr do={
        :if ($item = $target) do={
            :return $idx
        }
        :set idx ($idx + 1)
    }
    :return -1    # ไม่พบ
}

:put [$indexOf $fruits "cherry"]    # 2
:put [$indexOf $fruits "grape"]     # -1

# Function: Filter Array
:local filter do={
    :local arr $1
    :local condition $2    # Threshold value
    :local result {}

    :foreach item in=$arr do={
        :if ($item > $condition) do={
            :set result ($result, $item)
        }
    }
    :return $result
}

:local nums {5; 10; 15; 20; 25}
:local filtered [$filter $nums 12]
:put $filtered    # 15;20;25

# Function: Map Array
:local mapDouble do={
    :local arr $1
    :local result {}

    :foreach item in=$arr do={
        :set result ($result, ($item * 2))
    }
    :return $result
}

:put [$mapDouble $nums]    # 10;20;30;40;50

# Function: Reduce/Sum
:local sum do={
    :local arr $1
    :local total 0

    :foreach item in=$arr do={
        :set total ($total + $item)
    }
    :return $total
}

:put [$sum $nums]    # 75

# Function: Sort Array (Bubble Sort)
:local sort do={
    :local arr $1
    :local n [:len $arr]

    :for i from=0 to=($n - 2) do={
        :for j from=0 to=($n - $i - 2) do={
            :if (($arr->$j) > ($arr->($j+1))) do={
                :local temp ($arr->$j)
                :set ($arr->$j) ($arr->($j+1))
                :set ($arr->($j+1)) $temp
            }
        }
    }
    :return $arr
}

:local unsorted {5; 2; 8; 1; 9; 3}
:put [$sort $unsorted]    # 1;2;3;5;8;9

# Function: Unique Elements
:local unique do={
    :local arr $1
    :local result {}

    :foreach item in=$arr do={
        :local alreadyIn false
        :foreach existing in=$result do={
            :if ($item = $existing) do={
                :set alreadyIn true
            }
        }
        :if (!$alreadyIn) do={
            :set result ($result, $item)
        }
    }
    :return $result
}

:local withDups {1; 2; 2; 3; 3; 3; 4}
:put [$unique $withDups]    # 1;2;3;4
```

---

## Math Functions

### Built-in Math

```routeros
# Absolute Value
:local abs do={
    :local n $1
    :if ($n < 0) do={
        :return ($n * -1)
    }
    :return $n
}

:put [$abs -42]    # 42
:put [$abs 10]     # 10

# Power (x^n)
:local pow do={
    :local base $1
    :local exp $2
    :local result 1

    :for i from=1 to=$exp do={
        :set result ($result * $base)
    }
    :return $result
}

:put [$pow 2 10]    # 1024 (2^10)
:put [$pow 3 4]     # 81  (3^4)

# Square Root (Newton's Method)
:local sqrt do={
    :local n $1
    :if ($n < 0) do={ :return -1 }
    :if ($n = 0) do={ :return 0 }

    :local x $n
    :for i from=0 to=20 do={
        :set x (($x + ($n / $x)) / 2)
    }
    :return $x
}

:put [$sqrt 16]    # 4
:put [$sqrt 100]   # 10

# Min, Max
:local min do={
    :if ($1 < $2) do={ :return $1 }
    :return $2
}

:local max do={
    :if ($1 > $2) do={ :return $1 }
    :return $2
}

:put [$min 10 20]    # 10
:put [$max 10 20]    # 20

# Clamp (จำกัดค่าระหว่าง min และ max)
:local clamp do={
    :local val $1
    :local minVal $2
    :local maxVal $3
    :if ($val < $minVal) do={ :return $minVal }
    :if ($val > $maxVal) do={ :return $maxVal }
    :return $val
}

:put [$clamp 5 0 100]     # 5
:put [$clamp -10 0 100]   # 0
:put [$clamp 150 0 100]   # 100

# Round (ปัดเศษ)
:local round do={
    :local n $1
    :local dec $2
    :if ([:typeof $dec] = "nothing") do={ :set dec 0 }

    :local factor [$pow 10 $dec]
    :local shifted ($n * $factor)
    :local truncated ($shifted / 1)  # Integer truncation
    :local fraction ($shifted - $truncated)

    :if ($fraction >= 0.5) do={
        :set truncated ($truncated + 1)
    }
    :return ($truncated / $factor)
}

# Random Number
:put [:rndnum from=1 to=100]
:put [:rndnum from=0 to=255]
```

---

## IP Functions

### Built-in IP Operations

```routeros
# ตรวจสอบ IP ใน Subnet
:local inSubnet do={
    :local ip $1
    :local subnet $2
    :return ($ip in $subnet)
}

:put [$inSubnet 192.168.1.50 192.168.1.0/24]    # true
:put [$inSubnet 10.0.0.1 192.168.1.0/24]        # false

# ดึง Network Address จาก IP/Prefix
:local getNetwork do={
    :local ipPrefix $1
    :local ip [:toip [:pick [:tostr $ipPrefix] 0 [:find [:tostr $ipPrefix] "/"]]]
    :local prefix [:tonum [:pick [:tostr $ipPrefix] ([:find [:tostr $ipPrefix] "/"]+1) [:len [:tostr $ipPrefix]]]]

    :local mask (0xFFFFFFFF << (32 - $prefix))
    :return [:toip ($ip & $mask)]
}

# DNS Resolve
:local resolve do={
    :local hostname $1
    :local ip [:resolve $hostname]
    :return $ip
}

:local googleIP [$resolve "google.com"]
:put ("google.com = " . $googleIP)

# ตรวจสอบ IP เป็น Private/Public
:local isPrivate do={
    :local ip $1
    :if ($ip in 10.0.0.0/8) do={ :return true }
    :if ($ip in 172.16.0.0/12) do={ :return true }
    :if ($ip in 192.168.0.0/16) do={ :return true }
    :return false
}

:put [$isPrivate 192.168.1.1]    # true
:put [$isPrivate 8.8.8.8]        # false

# ตรวจสอบ Valid IP Format
:local isValidIP do={
    :local str $1
    :if (!($str ~ "^[0-9]+\\.[0-9]+\\.[0-9]+\\.[0-9]+\$")) do={
        :return false
    }
    :do {
        :local ip [:toip $str]
        :return true
    } on-error={
        :return false
    }
}

:put [$isValidIP "192.168.1.1"]    # true
:put [$isValidIP "192.168.1.abc"]  # false
:put [$isValidIP "256.0.0.1"]      # false

# ดึง Octet จาก IP
:local getOctet do={
    :local ip [:tostr $1]
    :local octet $2    # 1-4

    :local parts [:split $ip "."]
    :return [:tonum ($parts->($octet - 1))]
}

:put [$getOctet 192.168.1.100 1]    # 192
:put [$getOctet 192.168.1.100 4]    # 100
```

---

## Function Best Practices

### 1. Single Responsibility

```routeros
# BAD: Function ทำหลายอย่าง
:local doEverything do={
    # Backup
    # Check interfaces
    # Send email
    # Log
    # Update config
}

# GOOD: Function ทำสิ่งเดียว
:local createBackup do={
    # เฉพาะ Backup
}

:local checkInterfaces do={
    # เฉพาะ Check Interfaces
}

:local sendReport do={
    # เฉพาะ Send Report
}
```

### 2. Error Handling ใน Functions

```routeros
# ใช้ :do ... :on-error ใน Functions
:local safeBackup do={
    :local name $1
    :local success false

    :do {
        /system backup save name=$name
        :set success true
    } on-error={
        :log error ("Backup failed: " . $name)
    }

    :return $success
}

:local result [$safeBackup "my-backup"]
:if ($result) do={
    :put "Backup successful"
} else={
    :put "Backup failed"
}
```

### 3. Documentation

```routeros
# Document Functions ด้วย Comments
# Function: calculateBandwidth
# Parameters:
#   $1 = interface name (str)
#   $2 = measurement interval in seconds (num)
# Returns:
#   Array {"rx"=<bytes/s>, "tx"=<bytes/s>}
# Example:
#   :local bw [$calculateBandwidth "ether1" 5]
#   :put ($bw->"rx")

:local calculateBandwidth do={
    :local iface $1
    :local interval $2

    :local rx1 [/interface get $iface rx-byte]
    :local tx1 [/interface get $iface tx-byte]
    :delay ($interval . "s")
    :local rx2 [/interface get $iface rx-byte]
    :local tx2 [/interface get $iface tx-byte]

    :return {
        "rx"=(($rx2 - $rx1) / $interval);
        "tx"=(($tx2 - $tx1) / $interval)
    }
}
```

### 4. Avoid Side Effects

```routeros
# BAD: Function มี Side Effect
:global logMessages ""

:local addLog do={
    :global logMessages
    :set logMessages ($logMessages . $1 . "\n")  # Side effect!
}

# GOOD: Function Return Value แทน Side Effect
:local buildMessage do={
    :local prefix $1
    :local msg $2
    :return ("[" . $prefix . "] " . $msg)
}

:local formattedMsg [$buildMessage "INFO" "System started"]
:log info $formattedMsg
```

---

## Reusable Function Library

### Library สมบูรณ์สำหรับใช้งาน

```routeros
# ===================================================
# MikroTik RouterOS Scripting Function Library
# Version: 1.0.0
# ===================================================
# การใช้งาน: นำ Script นี้ import เข้าระบบ
# หรือ copy section ที่ต้องการ
# ===================================================

# ----- STRING LIBRARY -----

# Function: str_len - ความยาว String
:global str_len do={
    :return [:len $1]
}

# Function: str_sub - Substring
:global str_sub do={
    :local str $1
    :local start $2
    :local end $3
    :if ([:typeof $end] = "nothing") do={
        :set end [:len $str]
    }
    :return [:pick $str $start $end]
}

# Function: str_find - Find ตำแหน่ง
:global str_find do={
    :return [:find $1 $2]
}

# Function: str_contains - เช็คว่ามี Substring
:global str_contains do={
    :return ([:find $1 $2] != "")
}

# Function: str_starts_with - เช็ค Prefix
:global str_starts_with do={
    :local str $1
    :local prefix $2
    :return ([:pick $str 0 [:len $prefix]] = $prefix)
}

# Function: str_ends_with - เช็ค Suffix
:global str_ends_with do={
    :local str $1
    :local suffix $2
    :local offset ([:len $str] - [:len $suffix])
    :if ($offset < 0) do={ :return false }
    :return ([:pick $str $offset [:len $str]] = $suffix)
}

# Function: str_repeat - ทำซ้ำ String
:global str_repeat do={
    :local str $1
    :local n $2
    :local result ""
    :for i from=1 to=$n do={
        :set result ($result . $str)
    }
    :return $result
}

# Function: str_pad_left - Pad ด้านซ้าย
:global str_pad_left do={
    :local str [:tostr $1]
    :local width $2
    :local padChar $3
    :if ([:typeof $padChar] = "nothing") do={ :set padChar " " }
    :while ([:len $str] < $width) do={
        :set str ($padChar . $str)
    }
    :return $str
}

# Function: str_split - Split String
:global str_split do={
    :return [:split $1 $2]
}

# ----- NUMBER LIBRARY -----

# Function: num_abs - Absolute Value
:global num_abs do={
    :if ($1 < 0) do={ :return ($1 * -1) }
    :return $1
}

# Function: num_min - Minimum
:global num_min do={
    :if ($1 < $2) do={ :return $1 }
    :return $2
}

# Function: num_max - Maximum
:global num_max do={
    :if ($1 > $2) do={ :return $1 }
    :return $2
}

# Function: num_clamp - Clamp Value
:global num_clamp do={
    :local val $1
    :local minVal $2
    :local maxVal $3
    :if ($val < $minVal) do={ :return $minVal }
    :if ($val > $maxVal) do={ :return $maxVal }
    :return $val
}

# Function: num_pow - Power
:global num_pow do={
    :local base $1
    :local exp $2
    :local result 1
    :for i from=1 to=$exp do={
        :set result ($result * $base)
    }
    :return $result
}

# Function: num_format - Format Number with Separator
:global num_format do={
    :local n [:tostr $1]
    :local result ""
    :local len [:len $n]
    :local count 0

    :for i from=($len-1) to=0 do={
        :set result ([:pick $n $i ($i+1)] . $result)
        :set count ($count + 1)
        :if (($count % 3 = 0) && ($i > 0)) do={
            :set result ("," . $result)
        }
    }
    :return $result
}

# ----- ARRAY LIBRARY -----

# Function: arr_contains - เช็คว่า Array มี Element
:global arr_contains do={
    :local arr $1
    :local target $2
    :foreach item in=$arr do={
        :if ($item = $target) do={ :return true }
    }
    :return false
}

# Function: arr_sum - รวมทุก Elements
:global arr_sum do={
    :local arr $1
    :local total 0
    :foreach item in=$arr do={
        :set total ($total + $item)
    }
    :return $total
}

# Function: arr_avg - ค่าเฉลี่ย
:global arr_avg do={
    :local arr $1
    :local total 0
    :local count 0
    :foreach item in=$arr do={
        :set total ($total + $item)
        :set count ($count + 1)
    }
    :if ($count = 0) do={ :return 0 }
    :return ($total / $count)
}

# Function: arr_max - Maximum ใน Array
:global arr_max do={
    :local arr $1
    :local max ($arr->0)
    :foreach item in=$arr do={
        :if ($item > $max) do={ :set max $item }
    }
    :return $max
}

# Function: arr_min - Minimum ใน Array
:global arr_min do={
    :local arr $1
    :local min ($arr->0)
    :foreach item in=$arr do={
        :if ($item < $min) do={ :set min $item }
    }
    :return $min
}

# ----- IP LIBRARY -----

# Function: ip_is_private - เช็ค Private IP
:global ip_is_private do={
    :local ip $1
    :if ($ip in 10.0.0.0/8) do={ :return true }
    :if ($ip in 172.16.0.0/12) do={ :return true }
    :if ($ip in 192.168.0.0/16) do={ :return true }
    :return false
}

# Function: ip_resolve - DNS Resolve
:global ip_resolve do={
    :do {
        :return [:resolve $1]
    } on-error={
        :return "0.0.0.0"
    }
}

# Function: ip_ping - Ping Test
:global ip_ping do={
    :local host $1
    :local count $2
    :if ([:typeof $count] = "nothing") do={ :set count 3 }
    :local result [/ping $host count=$count as-value]
    :return ($result->"received")
}

# ----- SYSTEM LIBRARY -----

# Function: sys_uptime_seconds - Uptime ใน Seconds
:global sys_uptime_seconds do={
    :local uptime [/system resource get uptime]
    :return [:tonum $uptime]
}

# Function: sys_cpu_ok - ตรวจสอบว่า CPU ปกติ
:global sys_cpu_ok do={
    :local threshold $1
    :if ([:typeof $threshold] = "nothing") do={ :set threshold 80 }
    :local cpu [/system resource get cpu-load]
    :return ($cpu < $threshold)
}

# Function: sys_mem_free_mb - Free Memory ใน MB
:global sys_mem_free_mb do={
    :local freeMem [/system resource get free-memory]
    :return ($freeMem / 1048576)
}

# Function: log_with_level - Log พร้อม Level
:global log_with_level do={
    :local level $1
    :local msg $2

    :if ($level = "error") do={ :log error $msg }
    :if ($level = "warning") do={ :log warning $msg }
    :if ($level = "info") do={ :log info $msg }
    :if ($level = "debug") do={ :log debug $msg }
}
```

### ตัวอย่างการใช้ Library

```routeros
# ===== Import Library =====
# (Run Library Script First)

# ===== Use Library =====

# String Functions
:global str_len
:global str_pad_left
:global str_contains

:local name "MikroTik"
:put [$str_len $name]                    # 8
:put [$str_pad_left $name 15 "-"]        # -------MikroTik
:put [$str_contains $name "Tik"]         # true

# Number Functions
:global num_abs
:global num_clamp
:global num_format

:put [$num_abs -42]           # 42
:put [$num_clamp 150 0 100]   # 100
:put [$num_format 1000000]    # 1,000,000

# Array Functions
:global arr_sum
:global arr_avg
:global arr_max

:local data {10; 20; 30; 40; 50}
:put [$arr_sum $data]    # 150
:put [$arr_avg $data]    # 30
:put [$arr_max $data]    # 50

# IP Functions
:global ip_is_private
:global ip_ping

:put [$ip_is_private 192.168.1.1]    # true
:put [$ip_ping "8.8.8.8" 3]          # จำนวน Packets ที่ได้รับ
```

---

## Exercises

```routeros
# Exercise 1: สร้าง Function คำนวณ BMI
# BMI = weight (kg) / height^2 (m)
:local calcBMI do={
    :local weight $1
    :local height $2
    :local bmi ($weight / ($height * $height))

    :local category
    :if ($bmi < 18.5) do={ :set category "Underweight" }
    :if ($bmi >= 18.5 && $bmi < 25) do={ :set category "Normal" }
    :if ($bmi >= 25 && $bmi < 30) do={ :set category "Overweight" }
    :if ($bmi >= 30) do={ :set category "Obese" }

    :return {"bmi"=$bmi; "category"=$category}
}

:local result [$calcBMI 70 1.75]
:put ("BMI: " . ($result->"bmi") . " - " . ($result->"category"))

# Exercise 2: สร้าง Function Format Bytes
:local formatBytes do={
    :local bytes $1
    :if ($bytes >= 1073741824) do={ :return (($bytes / 1073741824) . " GB") }
    :if ($bytes >= 1048576) do={ :return (($bytes / 1048576) . " MB") }
    :if ($bytes >= 1024) do={ :return (($bytes / 1024) . " KB") }
    :return ($bytes . " B")
}

:put [$formatBytes 1024]         # 1 KB
:put [$formatBytes 1048576]      # 1 MB
:put [$formatBytes 1073741824]   # 1 GB

# Exercise 3: Function ตรวจสอบ Router Health
:local checkHealth do={
    :local issues {}
    :local score 100

    # Check CPU
    :local cpu [/system resource get cpu-load]
    :if ($cpu > 80) do={
        :set issues ($issues, ("CPU: " . $cpu . "%"))
        :set score ($score - 20)
    }

    # Check Memory
    :local freeMem [/system resource get free-memory]
    :local totalMem [/system resource get total-memory]
    :local memPct (100 - (($freeMem * 100) / $totalMem))
    :if ($memPct > 80) do={
        :set issues ($issues, ("Memory: " . $memPct . "%"))
        :set score ($score - 20)
    }

    # Check Default Route
    :local routes [/ip route find dst-address=0.0.0.0/0 active=yes]
    :if ([:len $routes] = 0) do={
        :set issues ($issues, "No default route!")
        :set score ($score - 40)
    }

    :return {"score"=$score; "issues"=$issues}
}

:local health [$checkHealth]
:put ("Health Score: " . ($health->"score") . "/100")
:foreach issue in=($health->"issues") do={
    :put ("Issue: " . $issue)
}
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 20:

| หัวข้อ | สาระสำคัญ |
|--------|-----------|
| Function Declaration | `:local fn do={...}` |
| Parameters | `$1`, `$2`, `$3` ตามลำดับ |
| Return Values | `:return value` |
| Recursive | เรียก `:global fn` ใน Function |
| Built-in Functions | `[:len]`, `[:pick]`, `[:find]`, `[:tostr]`, etc. |
| String Functions | Concat, Split, Find, Replace |
| Array Functions | Contains, Filter, Sum, Sort |
| Math Functions | Abs, Pow, Min, Max, Clamp |
| IP Functions | In subnet, Is private, Resolve |
| Best Practices | Single responsibility, Error handling, Documentation |

---

## Navigation

[← Part 19: Control Flow](part-019-control-flow.md) | [Part 21: Advanced Scripting →](part-021-advanced-scripting.md)

---

*MikroTik RouterOS Administration Course - Part 20*
*Last Updated: 2026*
