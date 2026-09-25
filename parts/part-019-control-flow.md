# Part 19: Control Flow

**หลักสูตร MikroTik RouterOS Administration**
**ระดับ:** Intermediate-Advanced
**เวลาเรียน:** 4-5 ชั่วโมง

---

## สารบัญ

- [if/else/elsif Syntax](#if-else)
- [Nested Conditions](#nested-conditions)
- [:foreach Loop](#foreach-loop)
- [:while Loop](#while-loop)
- [Loop Control (Break)](#loop-control)
- [:do...while Equivalent](#do-while)
- [Switch/Case Patterns](#switch-case)
- [Practical Examples](#practical-examples)
- [Common Patterns](#common-patterns)
- [Performance Considerations](#performance)
- [10 Exercises](#exercises)

---

## if/else/elsif Syntax

### รูปแบบ if Statement

```routeros
# รูปแบบที่ 1: if เพียงอย่างเดียว
:if (condition) do={
    # code เมื่อ condition เป็น true
}

# รูปแบบที่ 2: if-else
:if (condition) do={
    # code เมื่อ true
} else={
    # code เมื่อ false
}

# รูปแบบที่ 3: if-elseif-else (ใช้ :if ซ้อนกัน)
:if (condition1) do={
    # code
} else={
    :if (condition2) do={
        # code
    } else={
        :if (condition3) do={
            # code
        } else={
            # default code
        }
    }
}
```

### ตัวอย่าง if พื้นฐาน

```routeros
# ตัวอย่าง 1: ตรวจสอบ CPU Load
:local cpuLoad [/system resource get cpu-load]

:if ($cpuLoad > 90) do={
    :log error ("CRITICAL: CPU Load = " . $cpuLoad . "%")
} else={
    :if ($cpuLoad > 70) do={
        :log warning ("WARNING: CPU Load = " . $cpuLoad . "%")
    } else={
        :log info ("OK: CPU Load = " . $cpuLoad . "%")
    }
}
```

### ตัวอย่าง if สำหรับ Network

```routeros
# ตรวจสอบสถานะ Interface
:local eth1Running [/interface get ether1 running]
:local eth2Running [/interface get ether2 running]

:if ($eth1Running && $eth2Running) do={
    :put "Both interfaces are UP"
} else={
    :if (!$eth1Running && !$eth2Running) do={
        :log error "All interfaces are DOWN!"
        # ส่ง Alert
    } else={
        :if (!$eth1Running) do={
            :log warning "ether1 is DOWN"
        } else={
            :log warning "ether2 is DOWN"
        }
    }
}
```

### Short Circuit Evaluation

```routeros
# RouterOS ทำ Short-Circuit Evaluation
# && - ถ้า Left เป็น false ไม่ต้องตรวจ Right
# || - ถ้า Left เป็น true ไม่ต้องตรวจ Right

:local x 0

# วิธีที่ปลอดภัยกว่า: ตรวจสอบ x ก่อน
:if ($x != 0 && (10 / $x) > 2) do={
    :put "Result is greater than 2"
}

# ถ้า $x = 0, condition หยุดที่ ($x != 0) = false
# และไม่ evaluate (10 / $x) ซึ่งจะ Error
```

---

## Nested Conditions

### Nested if ลึกหลายระดับ

```routeros
# ตัวอย่าง: ตรวจสอบ Network Health
:local eth1Up [/interface get ether1 running]
:local ping1 [/ping 8.8.8.8 count=2 as-value]
:local received ($ping1->"received")

:if ($eth1Up) do={
    :if ($received > 0) do={
        :if ($received = 2) do={
            :log info "Network: Excellent (2/2 packets)"
        } else={
            :log warning "Network: Degraded (1/2 packets)"
        }
    } else={
        :log error "Network: No internet connectivity"
    }
} else={
    :log error "Network: Interface ether1 is DOWN"
}
```

### หลีกเลี่ยง Deep Nesting

```routeros
# ไม่ดี: Nested ลึกเกิน
:if (cond1) do={
    :if (cond2) do={
        :if (cond3) do={
            :if (cond4) do={
                # code
            }
        }
    }
}

# ดีกว่า: Guard Clause Pattern (Early Return)
:if (!cond1) do={ :log warning "cond1 failed"; :error "exit" }
:if (!cond2) do={ :log warning "cond2 failed"; :error "exit" }
:if (!cond3) do={ :log warning "cond3 failed"; :error "exit" }
:if (cond4) do={
    # main code
}
```

### ตัวอย่างการ Validate Input

```routeros
# Validate ทีละเงื่อนไข (ชัดเจนกว่า Nested)
:local hostname [/system identity get name]
:local version [/system resource get version]
:local uptime [/system resource get uptime]

# Check hostname
:if ([:len $hostname] = 0) do={
    :log error "Hostname not set!"
    :error "Configuration error"
}

# Check version
:if ($version ~ "^6\\.") do={
    :log warning "Running RouterOS 6.x - consider upgrading"
}

# Check uptime
:if ($uptime < 5m) do={
    :log info "Router recently rebooted"
}

:put "All checks passed"
```

---

## :foreach Loop

### รูปแบบ :foreach

```routeros
# รูปแบบพื้นฐาน
:foreach item in=<list/array> do={
    # ใช้ $item
}

# ตัวอย่าง: Loop ใน Array
:local fruits {"apple"; "banana"; "cherry"; "date"}

:foreach fruit in=$fruits do={
    :put ("Fruit: " . $fruit)
}
```

### :foreach กับ RouterOS Objects

```routeros
# Loop ผ่าน Interfaces
:foreach iface in=[/interface find] do={
    :local name [/interface get $iface name]
    :local running [/interface get $iface running]
    :local type [/interface get $iface type]

    :put ($name . " (" . $type . "): " . $running)
}

# Loop ผ่าน IP Addresses
:foreach addr in=[/ip address find] do={
    :local ip [/ip address get $addr address]
    :local iface [/ip address get $addr interface]
    :put ($ip . " on " . $iface)
}

# Loop ผ่าน DHCP Leases
:foreach lease in=[/ip dhcp-server lease find] do={
    :local mac [/ip dhcp-server lease get $lease mac-address]
    :local ip [/ip dhcp-server lease get $lease address]
    :put ($mac . " = " . $ip)
}
```

### :foreach พร้อม Filter

```routeros
# Loop เฉพาะ Interface ที่ Running
:foreach iface in=[/interface find running=yes] do={
    :local name [/interface get $iface name]
    :put ("Running: " . $name)
}

# Loop เฉพาะ Static Routes
:foreach route in=[/ip route find static=yes] do={
    :local dst [/ip route get $route dst-address]
    :local gw [/ip route get $route gateway]
    :put ($dst . " via " . $gw)
}
```

### :foreach กับ Counter

```routeros
# นับ Items ใน Loop
:local count 0
:foreach iface in=[/interface find] do={
    :set count ($count + 1)
}
:put ("Total interfaces: " . $count)

# ทำดัชนี
:local i 0
:local items {"a"; "b"; "c"; "d"; "e"}
:foreach item in=$items do={
    :put ($i . ": " . $item)
    :set i ($i + 1)
}
```

### :foreach แบบ Accumulate ผลลัพธ์

```routeros
# รวม Bandwidth จากทุก Interface
:local totalRxBytes 0
:local totalTxBytes 0

:foreach iface in=[/interface find] do={
    :local rx [/interface get $iface rx-byte]
    :local tx [/interface get $iface tx-byte]
    :set totalRxBytes ($totalRxBytes + $rx)
    :set totalTxBytes ($totalTxBytes + $tx)
}

:put ("Total RX: " . ($totalRxBytes / 1048576) . " MB")
:put ("Total TX: " . ($totalTxBytes / 1048576) . " MB")
```

---

## :while Loop

### รูปแบบ :while

```routeros
# รูปแบบพื้นฐาน
:while (condition) do={
    # code
}

# ตัวอย่าง: Count Up
:local i 1
:while ($i <= 10) do={
    :put $i
    :set i ($i + 1)
}
```

### :while สำหรับ Retry Logic

```routeros
# Retry ด้วย :while
:local maxRetries 3
:local retryDelay 5s
:local success false
:local attempt 0

:while (!$success && $attempt < $maxRetries) do={
    :set attempt ($attempt + 1)
    :log info ("Attempt " . $attempt . "/" . $maxRetries)

    # ลอง Ping
    :local result [/ping 8.8.8.8 count=3 as-value]
    :if (($result->"received") > 0) do={
        :set success true
        :log info "Success!"
    } else={
        :log warning "Attempt failed, waiting..."
        :if ($attempt < $maxRetries) do={
            :delay $retryDelay
        }
    }
}

:if ($success) do={
    :put "Connection restored"
} else={
    :log error "All retry attempts failed!"
}
```

### :while สำหรับ Polling

```routeros
# Poll จนกว่า Condition จะตรง
:local targetIP 192.168.1.100
:local timeout 60    # วินาที
:local elapsed 0
:local interval 5
:local found false

:while (!$found && $elapsed < $timeout) do={
    # ตรวจสอบว่า Device Online ไหม
    :local result [/ping $targetIP count=1 as-value]
    :if (($result->"received") > 0) do={
        :set found true
        :log info ($targetIP . " is now online after " . $elapsed . "s")
    } else={
        :delay ($interval . "s")
        :set elapsed ($elapsed + $interval)
    }
}

:if (!$found) do={
    :log warning ($targetIP . " not found within " . $timeout . "s")
}
```

### :while Infinite Loop (with Break)

```routeros
# Infinite Loop ที่มี Break Condition
:local running true

:while ($running) do={
    # ตรวจสอบ Condition
    :local cpuLoad [/system resource get cpu-load]

    :if ($cpuLoad > 95) do={
        :log error "CPU critically high, stopping monitor"
        :set running false
    } else={
        :log info ("CPU: " . $cpuLoad . "%")
        :delay 10s
    }
}
```

---

## Loop Control (Break)

### Break ใน :foreach

```routeros
# RouterOS ไม่มี break/continue โดยตรง
# แต่ทำได้ผ่าน Variable Flag หรือ :error

# วิธีที่ 1: Variable Flag
:local found false
:foreach item in={"a";"b";"c";"target";"d";"e"} do={
    :if (!$found) do={
        :if ($item = "target") do={
            :put ("Found: " . $item)
            :set found true
        } else={
            :put ("Checking: " . $item)
        }
    }
}

# วิธีที่ 2: :do ... :error (break-like)
:do {
    :foreach item in={"a";"b";"c";"target";"d";"e"} do={
        :if ($item = "target") do={
            :put ("Found: " . $item)
            :error "break"
        }
        :put ("Processing: " . $item)
    }
} on-error={
    # Loop exited via "break"
    :put "Loop ended (found target)"
}
```

### Break ใน :while

```routeros
# Break :while ด้วย Variable
:local i 0
:local shouldBreak false

:while ($i < 100 && !$shouldBreak) do={
    :if ($i = 5) do={
        :set shouldBreak true
    } else={
        :put $i
        :set i ($i + 1)
    }
}
:put ("Stopped at: " . $i)
```

### Continue Pattern (Skip Iteration)

```routeros
# Skip Iteration (continue equivalent)
:foreach iface in=[/interface find] do={
    :local name [/interface get $iface name]
    :local type [/interface get $iface type]

    # Skip ถ้าไม่ใช่ Ethernet
    :if ($type != "ether") do={
        # Skip (not ethernet)
    } else={
        # Process ethernet interfaces only
        :put ("Ethernet: " . $name)
    }
}
```

---

## :do...while Equivalent

### Do-While Pattern ใน RouterOS

RouterOS ไม่มี `do...while` โดยตรง แต่ทำได้:

```routeros
# Pattern 1: ใช้ :while พร้อม initial execution
# แทน do { ... } while (condition)

# ทำงานครั้งแรกก่อนแล้วค่อยเช็ค Condition

# แบบ do-while:
:local i 0

# ทำงานครั้งแรก
:put $i
:set i ($i + 1)

# แล้วค่อย Loop
:while ($i < 5) do={
    :put $i
    :set i ($i + 1)
}
```

### Do-While สำหรับ Menu

```routeros
# Do-While Pattern: ทำงานอย่างน้อย 1 ครั้ง
:local continueLoop true

:while ($continueLoop) do={
    # ทำงาน (body executes at least once)
    :local cpuLoad [/system resource get cpu-load]
    :put ("CPU: " . $cpuLoad . "%")

    # Check condition ที่ส่วนท้าย
    :if ($cpuLoad < 10) do={
        :set continueLoop false
    } else={
        :delay 5s
    }
}
```

---

## Switch/Case Patterns

### Switch/Case ใน RouterOS

RouterOS ไม่มี `switch/case` แต่ทำได้หลายวิธี:

```routeros
# วิธีที่ 1: if-elseif Chain
:local day [/system clock get day-of-week]

:if ($day = "mon") do={
    :put "Monday - Start of work week"
} else={
    :if ($day = "tue") do={
        :put "Tuesday"
    } else={
        :if ($day = "wed") do={
            :put "Wednesday"
        } else={
            :if ($day = "thu") do={
                :put "Thursday"
            } else={
                :if ($day = "fri") do={
                    :put "Friday - TGIF!"
                } else={
                    :put "Weekend!"
                }
            }
        }
    }
}
```

### Switch/Case ด้วย Array Lookup

```routeros
# วิธีที่ 2: Array ของ Values (ดีกว่า)
:local day [/system clock get day-of-week]

:local dayMessages {
    "mon"="Monday - Start of work week";
    "tue"="Tuesday";
    "wed"="Wednesday";
    "thu"="Thursday";
    "fri"="Friday - TGIF!";
    "sat"="Saturday - Weekend!";
    "sun"="Sunday - Weekend!"
}

:local message ($dayMessages->$day)
:if ([:typeof $message] != "nothing") do={
    :put $message
} else={
    :put "Unknown day"
}
```

### Switch/Case สำหรับ HTTP Status

```routeros
# Status Code Handler
:local processStatus do={
    :local status $1
    :local msg

    :if ($status = 200) do={
        :set msg "OK"
    } else={
        :if ($status = 301) do={
            :set msg "Moved Permanently"
        } else={
            :if ($status = 404) do={
                :set msg "Not Found"
            } else={
                :if ($status = 500) do={
                    :set msg "Internal Server Error"
                } else={
                    :set msg ("Unknown status: " . $status)
                }
            }
        }
    }
    :return $msg
}

:put [$processStatus 200]    # OK
:put [$processStatus 404]    # Not Found
:put [$processStatus 503]    # Unknown status: 503
```

### Switch/Case สำหรับ Interface Type

```routeros
# ทำงานต่างกันตาม Interface Type
:foreach iface in=[/interface find] do={
    :local name [/interface get $iface name]
    :local type [/interface get $iface type]

    :local actions {
        "ether"="Apply Ethernet QoS";
        "wlan"="Apply Wireless settings";
        "bridge"="Check Bridge status";
        "vlan"="Check VLAN routing";
        "pppoe-out"="Monitor PPPoE connection"
    }

    :local action ($actions->$type)
    :if ([:typeof $action] != "nothing") do={
        :log info ($name . ": " . $action)
    }
}
```

---

## Practical Examples

### 1. Interface Monitor Script

```routeros
# ตรวจสอบ Interfaces ทั้งหมดและ Report
:local report ""
:local downCount 0
:local upCount 0

:foreach iface in=[/interface find type=ether] do={
    :local name [/interface get $iface name]
    :local running [/interface get $iface running]
    :local disabled [/interface get $iface disabled]

    :if ($disabled) do={
        # Skip disabled interfaces
    } else={
        :if ($running) do={
            :set upCount ($upCount + 1)
            :set report ($report . "✓ " . $name . " UP\n")
        } else={
            :set downCount ($downCount + 1)
            :set report ($report . "✗ " . $name . " DOWN\n")
        }
    }
}

:put "=== Interface Status ==="
:put $report
:put ("UP: " . $upCount . ", DOWN: " . $downCount)

:if ($downCount > 0) do={
    :log warning ($downCount . " interface(s) are DOWN!")
}
```

### 2. DHCP Lease Monitor

```routeros
# ตรวจสอบ DHCP Leases พร้อมรายละเอียด
:local activeLeases 0
:local expiredLeases 0
:local totalLeases 0

:foreach lease in=[/ip dhcp-server lease find] do={
    :set totalLeases ($totalLeases + 1)
    :local status [/ip dhcp-server lease get $lease status]

    :if ($status = "bound") do={
        :set activeLeases ($activeLeases + 1)
    } else={
        :if ($status = "expired") do={
            :set expiredLeases ($expiredLeases + 1)
        }
    }
}

:put ("DHCP Leases - Active: " . $activeLeases . ", Expired: " . $expiredLeases . ", Total: " . $totalLeases)
```

### 3. Firewall Rule Counter Check

```routeros
# ตรวจสอบ Firewall Rules ที่มี Packets สูงผิดปกติ
:local threshold 10000

:foreach rule in=[/ip firewall filter find] do={
    :local chain [/ip firewall filter get $rule chain]
    :local action [/ip firewall filter get $rule action]
    :local bytes [/ip firewall filter get $rule bytes]
    :local packets [/ip firewall filter get $rule packets]
    :local comment [/ip firewall filter get $rule comment]

    :if ($packets > $threshold) do={
        :log warning ("High traffic rule: " . $chain . "/" . $action . " - " . $packets . " packets - " . $comment)
    }
}
```

### 4. Bandwidth Threshold Alert

```routeros
# ตรวจสอบ Bandwidth บน Interface
:local interface "ether1"
:local warnThreshold 80    # 80% ของ 1Gbps
:local critThreshold 95    # 95%
:local linkSpeed 1000000000  # 1 Gbps in bps

:local rxRate [/interface get $interface rx-byte]
:delay 1s
:local rxRateNew [/interface get $interface rx-byte]
:local rxBps ($rxRateNew - $rxRate)
:local rxPct (($rxBps * 100) / $linkSpeed)

:if ($rxPct > $critThreshold) do={
    :log error ("CRITICAL: " . $interface . " RX = " . $rxPct . "%")
} else={
    :if ($rxPct > $warnThreshold) do={
        :log warning ("WARNING: " . $interface . " RX = " . $rxPct . "%")
    } else={
        :log info ("OK: " . $interface . " RX = " . $rxPct . "%")
    }
}
```

---

## Common Patterns

### Pattern 1: Find-or-Default

```routeros
# ค้นหา Object แล้วใช้ Default ถ้าไม่พบ
:local target "ether5"
:local iface [/interface find name=$target]

:local ifaceName
:if ([:len $iface] > 0) do={
    :set ifaceName [/interface get ($iface->0) name]
} else={
    :set ifaceName "ether1"    # Default
    :log warning ($target . " not found, using default")
}

:put ("Using interface: " . $ifaceName)
```

### Pattern 2: Process with Error Handling

```routeros
# Pattern สำหรับ Process ที่อาจล้มเหลว
:local processItem do={
    :local item $1
    :local result "failed"

    :do {
        # Process the item
        :if ([:len $item] = 0) do={
            :error "Empty item"
        }

        # Do actual work
        :set result "success"
    } on-error={
        :log warning ("Failed to process: " . $item)
    }

    :return $result
}

# ใช้งาน
:local items {"valid"; ""; "also-valid"; ""}
:foreach item in=$items do={
    :local result [$processItem $item]
    :put ($item . ": " . $result)
}
```

### Pattern 3: Accumulate with Transform

```routeros
# สะสมและแปลงข้อมูล
:local output ""
:local separator ""

:foreach iface in=[/interface find running=yes] do={
    :local name [/interface get $iface name]
    :set output ($output . $separator . $name)
    :set separator ","
}

:put ("Running interfaces: " . $output)
# Output: Running interfaces: ether1,ether2,wlan1
```

### Pattern 4: Timeout Pattern

```routeros
# ทำงานในเวลาที่กำหนด
:local startTime [/system clock get time]
:local maxRuntime 30    # วินาที
:local timedOut false

:while (!$timedOut) do={
    # ทำงาน
    :local work true    # เปลี่ยนเป็น actual work

    # ตรวจสอบ Timeout
    :local currentTime [/system clock get time]
    # คำนวณ elapsed time (ง่ายๆ)
    :local elapsed 0
    # ... เปรียบเทียบ time ...

    :if ($elapsed > $maxRuntime) do={
        :set timedOut true
        :log warning "Operation timed out"
    }
}
```

### Pattern 5: Configuration Batch

```routeros
# Apply Configuration ใน Batch พร้อม Validation
:local configs {
    {"interface"="ether1"; "description"="WAN"; "speed"="auto"};
    {"interface"="ether2"; "description"="LAN"; "speed"="auto"};
    {"interface"="ether3"; "description"="DMZ"; "speed"="100M"}
}

:local successCount 0
:local failCount 0

:foreach config in=$configs do={
    :local iface ($config->"interface")
    :local desc ($config->"description")

    :do {
        :local found [/interface find name=$iface]
        :if ([:len $found] = 0) do={
            :error ("Interface " . $iface . " not found")
        }

        /interface comment [find name=$iface] comment=$desc
        :log info ("Configured " . $iface . ": " . $desc)
        :set successCount ($successCount + 1)
    } on-error={
        :log warning ("Failed to configure " . $iface)
        :set failCount ($failCount + 1)
    }
}

:put ("Done: " . $successCount . " success, " . $failCount . " failed")
```

---

## Performance Considerations

### ประสิทธิภาพของ Loops

```routeros
# ไม่ดี: Query ใน Loop
:foreach iface in=[/interface find] do={
    # แต่ละ Iteration Query ซ้ำ
    :local allInterfaces [/interface find]  # BAD! ไม่ต้องทำซ้ำ
    :local name [/interface get $iface name]
}

# ดีกว่า: Query ครั้งเดียวก่อน Loop
:local interfaces [/interface find]  # Query ครั้งเดียว
:foreach iface in=$interfaces do={
    :local name [/interface get $iface name]
    # process...
}
```

### ลด Loop Iterations

```routeros
# ไม่ดี: Loop ทั้งหมดแล้วค่อย Filter
:foreach route in=[/ip route find] do={
    :local static [/ip route get $route static]
    :if ($static) do={
        # process static routes
    }
}

# ดีกว่า: Filter ก่อน Loop
:foreach route in=[/ip route find static=yes] do={
    # process static routes only
}
```

### ใช้ Count แทน Loop

```routeros
# ไม่ดี: นับ Elements ด้วย Loop
:local count 0
:foreach item in=[/ip address find] do={
    :set count ($count + 1)
}

# ดีกว่า: ใช้ count-only
:local count [/ip address print count-only]
```

### Delay อย่างเหมาะสม

```routeros
# ระวัง: Delay สั้นเกินไปใน Loop อาจกิน CPU
:while (true) do={
    # Check every 100ms - CPU intensive!
    :delay 100ms
    # Do check...
}

# ดีกว่า: ใช้ Delay ที่เหมาะสม
:while (true) do={
    # Check every 10s - better
    :delay 10s
    # Do check...
}

# ดีที่สุด: ใช้ Scheduler แทน Infinite Loop
/system scheduler add interval=10s on-event={ # check code }
```

---

## 10 Exercises

### Exercise 1: FizzBuzz

```routeros
# FizzBuzz: 1-30
# Fizz สำหรับ หาร 3 ลงตัว
# Buzz สำหรับ หาร 5 ลงตัว
# FizzBuzz สำหรับ หาร 15 ลงตัว

:for i from=1 to=30 do={
    :if (($i % 15) = 0) do={
        :put "FizzBuzz"
    } else={
        :if (($i % 3) = 0) do={
            :put "Fizz"
        } else={
            :if (($i % 5) = 0) do={
                :put "Buzz"
            } else={
                :put $i
            }
        }
    }
}
```

### Exercise 2: Sum ของ Array

```routeros
# หาผลรวมของตัวเลขใน Array
:local numbers {5; 10; 15; 20; 25; 30}
:local sum 0

:foreach n in=$numbers do={
    :set sum ($sum + $n)
}

:put ("Sum: " . $sum)    # 105
:put ("Average: " . ($sum / [:len $numbers]))    # 17
```

### Exercise 3: Find Maximum

```routeros
# หาค่า Maximum ใน Array
:local numbers {45; 12; 78; 23; 56; 90; 34}
:local max ($numbers->0)

:foreach n in=$numbers do={
    :if ($n > $max) do={
        :set max $n
    }
}

:put ("Maximum: " . $max)    # 90
```

### Exercise 4: Count Interfaces by Type

```routeros
# นับ Interfaces แยกตาม Type
:local counts {
    "ether"=0;
    "wlan"=0;
    "bridge"=0;
    "vlan"=0;
    "other"=0
}

:foreach iface in=[/interface find] do={
    :local type [/interface get $iface type]

    :if ($type = "ether") do={
        :set ($counts->"ether") (($counts->"ether") + 1)
    } else={
        :if ($type = "wlan") do={
            :set ($counts->"wlan") (($counts->"wlan") + 1)
        } else={
            :set ($counts->"other") (($counts->"other") + 1)
        }
    }
}

:put ("Ethernet: " . ($counts->"ether"))
:put ("Wireless: " . ($counts->"wlan"))
:put ("Other: " . ($counts->"other"))
```

### Exercise 5: Prime Numbers

```routeros
# หา Prime Numbers จาก 2 ถึง 50
:local primes ""
:local sep ""

:for n from=2 to=50 do={
    :local isPrime true

    :for i from=2 to=($n - 1) do={
        :if (($n % $i) = 0) do={
            :set isPrime false
        }
    }

    :if ($isPrime) do={
        :set primes ($primes . $sep . $n)
        :set sep ","
    }
}

:put ("Primes: " . $primes)
```

### Exercise 6: Fibonacci Sequence

```routeros
# สร้าง Fibonacci Sequence
:local a 0
:local b 1
:local n 15    # จำนวน Elements

:put $a
:put $b

:for i from=2 to=$n do={
    :local c ($a + $b)
    :put $c
    :set a $b
    :set b $c
}
```

### Exercise 7: Reverse Array

```routeros
# กลับ Array
:local arr {"a"; "b"; "c"; "d"; "e"}
:local reversed ""
:local sep ""

:local len [:len $arr]
:for i from=($len - 1) to=0 do={
    :set reversed ($reversed . $sep . ($arr->$i))
    :set sep ";"
}

:put ("Reversed: " . $reversed)
```

### Exercise 8: Ping Multiple Hosts

```routeros
# Ping หลาย Hosts และรายงานผล
:local hosts {
    "8.8.8.8";
    "1.1.1.1";
    "8.8.4.4";
    "208.67.222.222";
    "192.168.1.1"
}

:local upCount 0
:local downCount 0

:foreach host in=$hosts do={
    :local result [/ping $host count=2 as-value]
    :local received ($result->"received")

    :if ($received > 0) do={
        :put ("[UP]   " . $host . " (" . $received . "/2 packets)")
        :set upCount ($upCount + 1)
    } else={
        :put ("[DOWN] " . $host)
        :set downCount ($downCount + 1)
    }
}

:put ("---")
:put ("Results: " . $upCount . " up, " . $downCount . " down")
```

### Exercise 9: Config Validation

```routeros
# ตรวจสอบ Configuration ว่าครบถ้วนไหม
:local issues ""
:local issueCount 0

# ตรวจ Hostname
:local hostname [/system identity get name]
:if ($hostname = "MikroTik" || [:len $hostname] = 0) do={
    :set issues ($issues . "- Hostname not configured\n")
    :set issueCount ($issueCount + 1)
}

# ตรวจว่ามี Password บน admin
:foreach user in=[/user find name=admin group=full] do={
    :local pass [/user get $user password]
    :if ($pass = "" || $pass = "admin") do={
        :set issues ($issues . "- Default admin password!\n")
        :set issueCount ($issueCount + 1)
    }
}

# ตรวจ NTP
:local ntpServer [/system ntp client get server-dns-names]
:if ([:len $ntpServer] = 0) do={
    :set issues ($issues . "- NTP not configured\n")
    :set issueCount ($issueCount + 1)
}

# รายงาน
:if ($issueCount = 0) do={
    :put "Configuration: All checks PASSED"
} else={
    :put ("Configuration Issues (" . $issueCount . "):")
    :put $issues
}
```

### Exercise 10: Interface Statistics Report

```routeros
# รายงาน Interface Statistics
:put "=== Interface Statistics Report ==="
:put ""

:foreach iface in=[/interface find !disabled] do={
    :local name [/interface get $iface name]
    :local type [/interface get $iface type]
    :local running [/interface get $iface running]
    :local rxBytes [/interface get $iface rx-byte]
    :local txBytes [/interface get $iface tx-byte]
    :local rxPackets [/interface get $iface rx-packet]
    :local txPackets [/interface get $iface tx-packet]
    :local rxErrors [/interface get $iface rx-error]
    :local txErrors [/interface get $iface tx-error]

    :local rxMB ($rxBytes / 1048576)
    :local txMB ($txBytes / 1048576)

    :put ("Interface: " . $name . " (" . $type . ")")
    :put ("  Status:  " . [:pick ("DOWN UP  ") ([:tonum (!$running)] * 5) ([:tonum (!$running)] * 5 + 4)])
    :put ("  RX: " . $rxMB . " MB (" . $rxPackets . " pkts, " . $rxErrors . " errors)")
    :put ("  TX: " . $txMB . " MB (" . $txPackets . " pkts, " . $txErrors . " errors)")
    :put ""
}
```

---

## Summary

สิ่งที่ได้เรียนรู้ใน Part 19:

| หัวข้อ | Syntax | ใช้สำหรับ |
|--------|--------|---------|
| if/else | `:if (cond) do={...} else={...}` | ตัดสินใจ |
| foreach | `:foreach x in=list do={...}` | วน Loop ใน Collection |
| while | `:while (cond) do={...}` | Loop ตาม Condition |
| Break | Variable flag หรือ `:error` | ออกจาก Loop |
| Switch | if-chain หรือ Array lookup | Multiple branches |

---

## Navigation

[← Part 18: Operators & Expressions](part-018-operators-expressions.md) | [Part 20: Functions & Procedures →](part-020-functions-procedures.md)

---

*MikroTik RouterOS Administration Course - Part 19*
*Last Updated: 2026*
