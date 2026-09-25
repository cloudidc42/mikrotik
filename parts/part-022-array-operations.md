# Part 22: Array Operations ใน RouterOS Scripting

## บทนำ

Arrays ใน RouterOS เป็นโครงสร้างข้อมูลพื้นฐานที่ช่วยให้เราจัดการข้อมูลหลายๆ รายการได้อย่างมีประสิทธิภาพ RouterOS ใช้ array ในรูปแบบ list/tuple ซึ่งมีความยืดหยุ่นสูง

---

## 22.1 Array Creation

### การสร้าง Array พื้นฐาน

```routeros
# สร้าง empty array
:local emptyArr {}

# สร้าง array พร้อมข้อมูล
:local fruits {"apple"; "banana"; "cherry"}

# Array ของตัวเลข
:local numbers {1; 2; 3; 4; 5}

# Array ของ IP addresses
:local ipList {"192.168.1.1"; "192.168.1.2"; "192.168.1.3"}

# Array แบบ comma-separated (ใช้ :toarray)
:local csvData "one,two,three,four"
:local arr [:toarray $csvData]

# Array ของ mixed types
:local mixed {"hello"; 42; true; 192.168.1.1}
```

### Array ใน RouterOS: รูปแบบพิเศษ

```routeros
# RouterOS array ใช้ semicolon เป็น separator
:local servers {"web"; "mail"; "db"; "cache"}

# สร้าง array ด้วย loop
:local nums {}
:for i from=1 to=10 do={
    :set nums ($nums , $i)
}
:put $nums

# สร้าง array จาก query result
:local interfaces [/interface find type=ether]
:put [:len $interfaces]
```

---

## 22.2 Array Access

### การเข้าถึงข้อมูลใน Array

```routeros
:local arr {"zero"; "one"; "two"; "three"; "four"}

# Access by index (zero-based)
:put ($arr->0)    # "zero"
:put ($arr->1)    # "one"
:put ($arr->4)    # "four"

# Access ด้วย :pick
:local firstEl [:pick $arr 0 1]
:put $firstEl    # "zero"

# Access range
:local subArr [:pick $arr 1 3]
:put $subArr    # "one" "two"

# ความยาวของ array
:put [:len $arr]    # 5

# Last element
:local lastIdx ([:len $arr] - 1)
:put ($arr->$lastIdx)    # "four"
```

### Named Array (Dictionary-like)

```routeros
# RouterOS ยังไม่รองรับ true dictionary
# แต่ใช้ array indexes ได้

# Array of arrays (nested)
:local config {
    {"name"; "ether1"; "ip"; "192.168.1.1/24"};
    {"name"; "ether2"; "ip"; "10.0.0.1/8"}
}

# Access nested
:put (($config->0)->0)    # "name"
:put (($config->0)->1)    # "ether1"
```

---

## 22.3 Array Manipulation

### เพิ่มข้อมูลเข้า Array

```routeros
:local arr {"one"; "two"; "three"}

# เพิ่มด้วย , (comma operator)
:set arr ($arr , "four")
:put $arr

# เพิ่มหลาย elements
:set arr ($arr , "five" , "six")
:put $arr

# เพิ่มที่ต้น (prepend)
:set arr ("zero" , $arr)
:put $arr

# Merge สอง arrays
:local arr1 {"a"; "b"; "c"}
:local arr2 {"d"; "e"; "f"}
:local merged ($arr1 , $arr2)
:put $merged
```

### ลบข้อมูลออกจาก Array

```routeros
# RouterOS ไม่มี direct delete element
# ต้องสร้าง array ใหม่

:local removeElement do={
    :local arr $1
    :local idx $2
    :local newArr {}
    
    :for i from=0 to=([:len $arr] - 1) do={
        :if ($i != $idx) do={
            :set newArr ($newArr , ($arr->$i))
        }
    }
    :return $newArr
}

:local fruits {"apple"; "banana"; "cherry"; "date"}
:set fruits [$removeElement $fruits 1]
:put $fruits
# Output: "apple" "cherry" "date"

# Remove by value
:local removeValue do={
    :local arr $1
    :local val $2
    :local newArr {}
    
    :foreach el in=$arr do={
        :if ($el != $val) do={
            :set newArr ($newArr , $el)
        }
    }
    :return $newArr
}

:set fruits [$removeValue $fruits "cherry"]
:put $fruits
# Output: "apple" "date"
```

### แทนที่ element

```routeros
:local arr {"one"; "two"; "THREE"; "four"}

# Replace element ที่ index 2
:set ($arr->2) "three"
:put $arr
# Output: "one" "two" "three" "four"

# Replace ด้วย function
:local replaceAt do={
    :local arr $1
    :local idx $2
    :local val $3
    :set ($arr->$idx) $val
    :return $arr
}
```

---

## 22.4 Iterating Arrays

### foreach loop

```routeros
:local fruits {"apple"; "banana"; "cherry"; "mango"}

# Basic iteration
:foreach fruit in=$fruits do={
    :put "Fruit: $fruit"
}

# Iteration with index
:local idx 0
:foreach fruit in=$fruits do={
    :put "$idx: $fruit"
    :set idx ($idx + 1)
}

# Iteration with condition
:foreach fruit in=$fruits do={
    :if ([:len $fruit] > 5) do={
        :put "Long name: $fruit"
    }
}
```

### for loop กับ Array

```routeros
:local arr {10; 20; 30; 40; 50}

# Iterate with for loop
:for i from=0 to=([:len $arr] - 1) do={
    :put "arr[$i] = " . ($arr->$i)
}

# Process every other element
:for i from=0 to=([:len $arr] - 1) step=2 do={
    :put ($arr->$i)
}
```

### while loop กับ Array

```routeros
:local queue {"task1"; "task2"; "task3"; "task4"}

# Process queue
:while ([:len $queue] > 0) do={
    :local task ($queue->0)
    :put "Processing: $task"
    # Remove first element
    :set queue [:pick $queue 1 [:len $queue]]
}
```

---

## 22.5 Array of IPs

```routeros
# สร้าง array ของ IP addresses
:local blockedIPs {"192.168.100.1"; "10.0.0.5"; "172.16.0.100"}

# ตรวจสอบว่า IP อยู่ใน array
:local isBlocked do={
    :local ip $1
    :local list $2
    :foreach item in=$list do={
        :if ($item = $ip) do={ :return true }
    }
    :return false
}

:if ([$isBlocked "10.0.0.5" $blockedIPs]) do={
    :put "IP is blocked!"
}

# Generate IP range
:local generateRange do={
    :local baseIP $1
    :local start $2
    :local end $3
    :local result {}
    
    :for i from=$start to=$end do={
        :set result ($result , ($baseIP . [:tostr $i]))
    }
    :return $result
}

:local ipRange [$generateRange "192.168.1." 1 10]
:foreach ip in=$ipRange do={
    :put $ip
}

# เพิ่ม IPs เข้า address list
:local badIPs {"1.2.3.4"; "5.6.7.8"; "9.10.11.12"}
:foreach ip in=$badIPs do={
    /ip firewall address-list add list=blacklist address=$ip
    :put "Added $ip to blacklist"
}
```

---

## 22.6 Array Functions

### :len กับ Arrays

```routeros
:local arr {"a"; "b"; "c"; "d"; "e"}

:put [:len $arr]
# Output: 5

# ตรวจสอบ empty array
:if ([:len $arr] = 0) do={
    :put "Array is empty"
} else={
    :put "Array has " . [:len $arr] . " elements"
}

# Loop based on length
:for i from=0 to=([:len $arr] - 1) do={
    :put ($arr->$i)
}
```

### :pick กับ Arrays

```routeros
:local arr {0; 1; 2; 3; 4; 5; 6; 7; 8; 9}

# ดึง elements 2-5
:local sub [:pick $arr 2 5]
:put $sub
# Output: 2 3 4

# ดึง 3 elements แรก
:local first3 [:pick $arr 0 3]
:put $first3

# ดึง elements หลัง
:local last3 [:pick $arr 7 10]
:put $last3

# Paginate array
:local paginate do={
    :local arr $1
    :local page $2
    :local pageSize $3
    :local start ($page * $pageSize)
    :local end ($start + $pageSize)
    :if ($end > [:len $arr]) do={ :set end [:len $arr] }
    :return [:pick $arr $start $end]
}

:local bigArr {1;2;3;4;5;6;7;8;9;10;11;12;13;14;15}
:local page1 [$paginate $bigArr 0 5]
:local page2 [$paginate $bigArr 1 5]
:local page3 [$paginate $bigArr 2 5]
:put "Page 1: $page1"
:put "Page 2: $page2"
:put "Page 3: $page3"
```

---

## 22.7 Nested Arrays

```routeros
# สร้าง 2D array (array of arrays)
:local matrix {
    {1; 2; 3};
    {4; 5; 6};
    {7; 8; 9}
}

# Access element [row][col]
:put (($matrix->0)->0)    # 1
:put (($matrix->1)->1)    # 5
:put (($matrix->2)->2)    # 9

# Iterate nested array
:foreach row in=$matrix do={
    :local rowStr ""
    :foreach col in=$row do={
        :set rowStr ($rowStr . $col . " ")
    }
    :put $rowStr
}

# Network topology as nested array
:local topology {
    {"router1"; "192.168.1.1"; {"ether1"; "ether2"}};
    {"router2"; "192.168.2.1"; {"ether1"; "ether3"}};
    {"router3"; "192.168.3.1"; {"ether1"; "ether4"}}
}

:foreach node in=$topology do={
    :put "Router: " . ($node->0)
    :put "IP: " . ($node->1)
    :put "Interfaces: " . ($node->2)
}
```

---

## 22.8 Converting Between Types and Arrays

```routeros
# String to Array
:local csv "apple,banana,cherry"
:local arr [:toarray $csv]
:put [:len $arr]    # 3

# Array to String
:local arr {"one"; "two"; "three"}
:local str [:tostr $arr]
:put $str    # "one;two;three"

# Number array
:local numArr {1; 2; 3; 4; 5}
:local sum 0
:foreach n in=$numArr do={
    :set sum ($sum + $n)
}
:put "Sum: $sum"
:put "Average: " . ($sum / [:len $numArr])

# Array from RouterOS objects
:local ethIfaces [/interface find type=ether]
:local ifaceNames {}
:foreach iface in=$ethIfaces do={
    :set ifaceNames ($ifaceNames , [/interface get $iface name])
}
:put $ifaceNames

# Unique values filter
:local unique do={
    :local arr $1
    :local result {}
    :foreach item in=$arr do={
        :local found false
        :foreach existing in=$result do={
            :if ($existing = $item) do={ :set found true }
        }
        :if (!$found) do={
            :set result ($result , $item)
        }
    }
    :return $result
}

:local withDups {"a"; "b"; "a"; "c"; "b"; "d"}
:put [$unique $withDups]
# Output: a b c d
```

---

## 22.9 Common Array Patterns

### Sorting Array

```routeros
# Bubble sort (simple implementation)
:local bubbleSort do={
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

:local nums {5; 3; 8; 1; 9; 2; 7; 4; 6}
:set nums [$bubbleSort $nums]
:put $nums
# Output: 1 2 3 4 5 6 7 8 9
```

### Array Search

```routeros
# Linear search
:local indexOf do={
    :local arr $1
    :local val $2
    :local idx 0
    :foreach item in=$arr do={
        :if ($item = $val) do={ :return $idx }
        :set idx ($idx + 1)
    }
    :return -1
}

:local arr {"red"; "green"; "blue"; "yellow"}
:put [$indexOf $arr "blue"]     # 2
:put [$indexOf $arr "purple"]   # -1

# Contains check
:local contains do={
    :local arr $1
    :local val $2
    :return ([$indexOf $arr $val] != -1)
}

:if ([$contains $arr "green"]) do={
    :put "Found green!"
}
```

### Array Filtering

```routeros
# Filter array by condition
:local filter do={
    :local arr $1
    :local minLen $2
    :local result {}
    :foreach item in=$arr do={
        :if ([:len [:tostr $item]] >= $minLen) do={
            :set result ($result , $item)
        }
    }
    :return $result
}

:local words {"hi"; "hello"; "hey"; "greetings"; "yo"}
:local longWords [$filter $words 4]
:put $longWords
# Output: "hello" "greetings"

# Filter numbers greater than threshold
:local filterGT do={
    :local arr $1
    :local threshold $2
    :local result {}
    :foreach item in=$arr do={
        :if ($item > $threshold) do={
            :set result ($result , $item)
        }
    }
    :return $result
}

:local nums {10; 25; 5; 40; 15; 30}
:put [$filterGT $nums 20]
# Output: 25 40 30
```

### Array Map (Transform)

```routeros
# ทำ operation กับทุก element
:local doubleArr do={
    :local arr $1
    :local result {}
    :foreach item in=$arr do={
        :set result ($result , ($item * 2))
    }
    :return $result
}

:local nums {1; 2; 3; 4; 5}
:put [$doubleArr $nums]
# Output: 2 4 6 8 10

# Convert all to uppercase
:local toUpperArr do={
    :local arr $1
    :local result {}
    :foreach item in=$arr do={
        :set result ($result , [:toupper $item])
    }
    :return $result
}

:local words {"apple"; "banana"; "cherry"}
:put [$toUpperArr $words]
# Output: APPLE BANANA CHERRY
```

### Array Reduce (Aggregate)

```routeros
# Sum array
:local sum do={
    :local arr $1
    :local total 0
    :foreach item in=$arr do={
        :set total ($total + $item)
    }
    :return $total
}

:local nums {10; 20; 30; 40; 50}
:put [$sum $nums]    # 150

# Max value
:local maxVal do={
    :local arr $1
    :local max ($arr->0)
    :foreach item in=$arr do={
        :if ($item > $max) do={ :set max $item }
    }
    :return $max
}

:put [$maxVal $nums]    # 50

# Min value
:local minVal do={
    :local arr $1
    :local min ($arr->0)
    :foreach item in=$arr do={
        :if ($item < $min) do={ :set min $item }
    }
    :return $min
}

:put [$minVal $nums]    # 10
```

---

## 22.10 Array ใน Real-world Scenarios

### Backup Multiple Routers

```routeros
# Manage multiple router IPs
:local routers {
    "192.168.1.1";
    "192.168.1.2";
    "192.168.1.3"
}

:foreach routerIP in=$routers do={
    :put "Connecting to $routerIP..."
    # ทำ backup operations
}

# Network subnets to monitor
:local subnets {
    "192.168.1.0/24";
    "192.168.2.0/24";
    "10.0.0.0/8"
}

:foreach subnet in=$subnets do={
    :put "Monitoring subnet: $subnet"
}
```

### Queue Management

```routeros
# สร้าง array เป็น queue
:local taskQueue {}

# Enqueue
:local enqueue do={
    :global taskQueue
    :set taskQueue ($taskQueue , $1)
}

# Dequeue
:local dequeue do={
    :global taskQueue
    :if ([:len $taskQueue] = 0) do={ :return "" }
    :local item ($taskQueue->0)
    :set taskQueue [:pick $taskQueue 1 [:len $taskQueue]]
    :return $item
}

# เพิ่ม tasks
[$enqueue "backup-config"]
[$enqueue "check-interfaces"]
[$enqueue "send-report"]

# Process queue
:while ([:len $taskQueue] > 0) do={
    :local task [$dequeue]
    :put "Processing: $task"
}
```

---

## Lab 22: Array Management Script

### โจทย์
สร้าง script จัดการ IP blacklist ด้วย arrays พร้อม functions: add, remove, check, list, export

### Solution

```routeros
# Array Management Lab - IP Blacklist Manager
# =============================================

# Initialize global blacklist
:global ipBlacklist {}

# --- Core Functions ---

# เพิ่ม IP เข้า blacklist
:local addToBlacklist do={
    :global ipBlacklist
    :local ip $1
    :local reason $2
    
    # ตรวจสอบ duplicate
    :foreach item in=$ipBlacklist do={
        :if (($item->0) = $ip) do={
            :put "IP $ip already in blacklist"
            :return false
        }
    }
    
    # เพิ่มพร้อม metadata
    :local entry {$ip; $reason; [/system clock get date]}
    :set ipBlacklist ($ipBlacklist , $entry)
    
    # เพิ่มใน firewall address list ด้วย
    /ip firewall address-list add \
        list=blacklist \
        address=$ip \
        comment=$reason
    
    :put "Added $ip to blacklist (reason: $reason)"
    :return true
}

# ลบ IP ออกจาก blacklist
:local removeFromBlacklist do={
    :global ipBlacklist
    :local ip $1
    :local newList {}
    :local found false
    
    :foreach item in=$ipBlacklist do={
        :if (($item->0) != $ip) do={
            :set newList ($newList , $item)
        } else={
            :set found true
        }
    }
    
    :if ($found) do={
        :set ipBlacklist $newList
        # ลบออกจาก firewall address list ด้วย
        :foreach fw in=[/ip firewall address-list find \
            list=blacklist address=$ip] do={
            /ip firewall address-list remove $fw
        }
        :put "Removed $ip from blacklist"
    } else={
        :put "IP $ip not found in blacklist"
    }
    :return $found
}

# ตรวจสอบว่า IP อยู่ใน blacklist
:local isBlacklisted do={
    :global ipBlacklist
    :local ip $1
    :foreach item in=$ipBlacklist do={
        :if (($item->0) = $ip) do={
            :return true
        }
    }
    :return false
}

# แสดงรายการ blacklist
:local listBlacklist do={
    :global ipBlacklist
    :put "=== IP Blacklist ========================"
    :put [[:tostr [:len $ipBlacklist]] . " entries"]
    :put "IP Address        Reason              Date"
    :put "----------------------------------------"
    
    :foreach item in=$ipBlacklist do={
        :local ip ($item->0)
        :local reason ($item->1)
        :local date ($item->2)
        :put ("$ip  $reason  $date")
    }
    :put "========================================"
}

# Export blacklist เป็น CSV
:local exportBlacklist do={
    :global ipBlacklist
    :local csv "IP,Reason,Date\r\n"
    
    :foreach item in=$ipBlacklist do={
        :set csv ($csv . ($item->0) . "," . ($item->1) . "," . ($item->2) . "\r\n")
    }
    
    # บันทึกลงไฟล์
    :local filename ("blacklist-" . [/system clock get date] . ".csv")
    :put $csv
    :put "Export complete: $filename"
    :return $csv
}

# Bulk import จาก array
:local bulkImport do={
    :local ips $1
    :local reason $2
    :local count 0
    
    :foreach ip in=$ips do={
        :if ([$addToBlacklist $ip $reason]) do={
            :set count ($count + 1)
        }
    }
    :put "Imported $count IPs"
}

# --- Demo Usage ---
:put "=== Blacklist Manager Demo ==="

# เพิ่ม IPs
[$addToBlacklist "1.2.3.4" "Port scan"]
[$addToBlacklist "5.6.7.8" "Brute force"]
[$addToBlacklist "9.10.11.12" "Spam"]

# Bulk import
:local suspiciousIPs {"100.200.100.1"; "100.200.100.2"; "100.200.100.3"}
[$bulkImport $suspiciousIPs "Bulk block - suspicious range"]

# แสดงรายการ
[$listBlacklist]

# ตรวจสอบ
:if ([$isBlacklisted "5.6.7.8"]) do={
    :put "5.6.7.8 is BLOCKED"
}

:if (![$isBlacklisted "8.8.8.8"]) do={
    :put "8.8.8.8 is ALLOWED"
}

# ลบออก
[$removeFromBlacklist "1.2.3.4"]

# Export
[$exportBlacklist]
```

### การทดสอบ

```routeros
# Test all functions
:put "Testing array operations..."

# Test basic array
:local arr {1; 2; 3; 4; 5}
:if ([:len $arr] = 5) do={ :put "PASS: length check" }

# Test append
:set arr ($arr , 6)
:if ([:len $arr] = 6) do={ :put "PASS: append" }

# Test index access
:if (($arr->0) = 1) do={ :put "PASS: index access" }

# Test pick
:local sub [:pick $arr 2 4]
:if (($sub->0) = 3) do={ :put "PASS: pick" }

:put "All tests passed!"
```

---

## Tips และ Notes

> **Tip:** ใช้ `{}` สร้าง empty array แล้วค่อยเพิ่มด้วย `,` operator

> **Note:** RouterOS array เป็น 0-indexed

> **Warning:** การ modify element ด้วย `(:set ($arr->idx) val)` อาจไม่ work ในทุก RouterOS version ให้สร้าง array ใหม่แทน

> **Best Practice:** Array ขนาดใหญ่มาก (>1000 elements) อาจทำให้ script ช้า ให้ใช้ database (DHCP, address-list) แทน

| Operation | Syntax | หมายเหตุ |
|-----------|--------|---------|
| สร้าง | `{a; b; c}` | ใช้ `;` คั่น |
| เพิ่มท้าย | `$arr , "new"` | สร้าง array ใหม่ |
| Access | `$arr->0` | Zero-indexed |
| Length | `[:len $arr]` | |
| Slice | `[:pick $arr 0 3]` | End is exclusive |
| Iterate | `foreach item in=$arr` | |
| From CSV | `[:toarray "a,b,c"]` | |

---

## Summary ของ Part 22

Arrays ใน RouterOS มีความสามารถพื้นฐานที่ครบถ้วน:
- สร้างและจัดการ arrays ด้วย syntax ที่ชัดเจน
- เข้าถึงข้อมูลด้วย index หรือ iteration
- รวม arrays ด้วย `,` operator
- ใช้ `:pick`, `:len` สำหรับ manipulation
- Implement patterns เช่น queue, stack, filter, map ด้วย functions

---

[← Part 21: String Operations](part-021-string-operations.md) | [Part 23: File Operations →](part-023-file-operations.md)
