# Part 48: Queue Management บน MikroTik

## บทนำ

Queue Management เป็นการควบคุม bandwidth ของ users หรือ services ต่างๆ MikroTik มี Queue หลายประเภทให้เลือกใช้ตามความต้องการ การใช้ scripting ช่วยให้สามารถสร้างและจัดการ queues ได้แบบ dynamic

---

## 48.1 Queue Types Comparison

### ประเภทของ Queue

| Queue Type | Use Case | Pros | Cons |
|-----------|----------|------|------|
| Simple Queue | User bandwidth limit | ง่าย, GUI-friendly | ไม่ flexible มาก |
| Queue Tree | Complex QoS hierarchy | Flexible, powerful | ซับซ้อน |
| PCQ | Fair usage per-user | Auto-fair | ไม่รองรับ per-user limit |

### Queue Architecture

```
Traffic Flow:
Incoming → [Mangle] → [Queue Tree/PCQ] → [Scheduler] → Outgoing
                              ↑
                       [Simple Queue]
```

---

## 48.2 Simple Queue Scripts

### สร้าง Simple Queue สำหรับ User

```routeros
# สร้าง queue เดี่ยว
/queue simple
add name="User-001" target=192.168.1.100/32 \
    max-limit=10M/10M \
    comment="User 001 - 10Mbps"

# สร้าง queue พร้อม burst
/queue simple
add name="User-002" target=192.168.1.101/32 \
    max-limit=10M/10M \
    burst-limit=20M/20M \
    burst-threshold=8M/8M \
    burst-time=10s/10s \
    comment="User 002 - 10Mbps with 20Mbps burst"
```

### Script สร้าง Simple Queues แบบ Bulk

```routeros
/system script
add name="create-user-queues" source={
    :local users {
        {"ip"="192.168.1.100"; "name"="User-001"; "bw"="10M"};
        {"ip"="192.168.1.101"; "name"="User-002"; "bw"="5M"};
        {"ip"="192.168.1.102"; "name"="User-003"; "bw"="20M"};
        {"ip"="192.168.1.103"; "name"="User-004"; "bw"="10M"}
    }
    
    :foreach user in=$users do={
        :local userName ($user->"name")
        :local userIP ($user->"ip")
        :local userBW ($user->"bw")
        
        # ตรวจสอบว่า queue มีอยู่แล้วหรือไม่
        :local existing [/queue simple find name=$userName]
        
        :if ([:len $existing] = 0) do={
            /queue simple add \
                name=$userName \
                target=($userIP . "/32") \
                max-limit=($userBW . "/" . $userBW) \
                comment=("Auto-created: " . $userName)
            
            :log info ("Queue created: " . $userName . " " . $userBW)
        } else={
            # Update bandwidth ถ้ามีอยู่แล้ว
            /queue simple set $existing \
                max-limit=($userBW . "/" . $userBW)
            :log info ("Queue updated: " . $userName . " " . $userBW)
        }
    }
    
    :put "User queues configured!"
}
```

### Script ลบ Queues ที่ Inactive

```routeros
/system script
add name="cleanup-inactive-queues" source={
    :local inactiveTime 7d
    
    :foreach queue in=[/queue simple find] do={
        :local qName [/queue simple get $queue name]
        :local qTarget [/queue simple get $queue target]
        
        # ตรวจสอบว่ามี active traffic
        :local rxBytes [/queue simple get $queue bytes]
        
        :if ($rxBytes = "0/0") do={
            :log info ("Removing inactive queue: " . $qName)
            /queue simple remove $queue
        }
    }
}
```

---

## 48.3 Queue Tree Scripts

### Queue Tree Structure

```routeros
# ====================================
# QUEUE TREE - HIERARCHICAL BANDWIDTH
# ====================================

# Parent queue (Total WAN bandwidth)
/queue tree
add name="TOTAL-WAN" parent=global \
    max-limit=100M \
    comment="Total WAN bandwidth: 100Mbps"

# Child: Internet traffic
add name="INTERNET" parent=TOTAL-WAN \
    max-limit=80M \
    comment="Internet: 80Mbps"

# Child: VoIP (guaranteed)
add name="VOIP" parent=TOTAL-WAN \
    max-limit=20M priority=1 \
    comment="VoIP: 20Mbps guaranteed"

# Child within Internet
add name="HTTP" parent=INTERNET \
    max-limit=50M priority=3 \
    packet-mark=HTTP-MARK \
    comment="HTTP/HTTPS: 50Mbps"

add name="OTHER" parent=INTERNET \
    max-limit=30M priority=5 \
    comment="Other traffic: 30Mbps"
```

### Dynamic Queue Tree

```routeros
/system script
add name="dynamic-queue-tree" source={
    # สร้าง Queue Tree แบบ dynamic ตาม time of day
    :local currentHour [:tonum [:pick [/system clock get time] 0 2]]
    :local totalBW 100M
    
    :if ($currentHour >= 8 && $currentHour < 18) do={
        # Business hours - ลด entertainment bandwidth
        /queue tree set [find name=HTTP] max-limit=60M
        /queue tree set [find name=VOIP] max-limit=30M
        :log info "Queue: Business hours profile applied"
    } else={
        # Off hours - ให้ entertainment bandwidth เพิ่ม
        /queue tree set [find name=HTTP] max-limit=80M
        /queue tree set [find name=VOIP] max-limit=10M
        :log info "Queue: Off-hours profile applied"
    }
}

/system scheduler
add name="queue-time-policy" interval=1h \
    on-event="/system script run dynamic-queue-tree"
```

---

## 48.4 PCQ Configuration

### PCQ คืออะไร?

PCQ (Per Connection Queue) แบ่ง bandwidth อย่างเป็นธรรมให้แต่ละ user/connection โดยอัตโนมัติ

```routeros
# สร้าง PCQ type
/queue type
add name="PCQ-DOWNLOAD" kind=pcq \
    pcq-classifier=dst-address \
    pcq-rate=2M \
    pcq-limit=100KiB \
    pcq-burst-rate=4M \
    pcq-burst-threshold=1536KiB \
    pcq-burst-time=10s

add name="PCQ-UPLOAD" kind=pcq \
    pcq-classifier=src-address \
    pcq-rate=512k \
    pcq-limit=50KiB

# ใช้ PCQ ใน Queue Tree
/queue tree
add name="PCQ-DOWN" parent=global \
    queue=PCQ-DOWNLOAD \
    packet-mark=all \
    comment="PCQ Download"

add name="PCQ-UP" parent=global \
    queue=PCQ-UPLOAD \
    packet-mark=all \
    comment="PCQ Upload"
```

---

## 48.5 Dynamic Queue Creation

### Auto-create Queue เมื่อ User Connect

```routeros
/system script
add name="auto-create-queue" source={
    # รับ IP address ของ user ใหม่
    :local userIP $1
    :local userName $2
    :local bwLimit $3
    
    :if ([:len $userIP] = 0) do={
        :error "userIP required"
    }
    
    :if ([:len $bwLimit] = 0) do={
        :set bwLimit "5M"
    }
    
    :if ([:len $userName] = 0) do={
        :set userName ("User-" . $userIP)
    }
    
    # สร้าง queue name ที่ unique
    :local queueName ("Q-" . $userIP)
    
    # ตรวจสอบว่ามีอยู่แล้ว
    :local existing [/queue simple find name=$queueName]
    
    :if ([:len $existing] = 0) do={
        /queue simple add \
            name=$queueName \
            target=($userIP . "/32") \
            max-limit=($bwLimit . "/" . $bwLimit) \
            comment=("Auto: " . $userName . " " . [/system clock get time])
        
        :log info ("Queue auto-created for " . $userIP . " limit=" . $bwLimit)
    } else={
        :log debug ("Queue already exists for " . $userIP)
    }
}
```

### Queue ตาม DHCP Lease

```routeros
/system script
add name="dhcp-queue-manager" source={
    # ตรวจสอบ DHCP leases และสร้าง queue
    :foreach lease in=[/ip dhcp-server lease find status=bound] do={
        :local ip [/ip dhcp-server lease get $lease address]
        :local mac [/ip dhcp-server lease get $lease mac-address]
        :local hostname [/ip dhcp-server lease get $lease host-name]
        
        :local queueName ("DHCP-" . $ip)
        :local existing [/queue simple find name=$queueName]
        
        :if ([:len $existing] = 0) do={
            # Default bandwidth: 5Mbps
            /queue simple add \
                name=$queueName \
                target=($ip . "/32") \
                max-limit=5M/5M \
                comment=("DHCP: " . $hostname . " [" . $mac . "]")
            
            :log info ("Auto queue for DHCP user: " . $ip . " " . $hostname)
        }
    }
    
    # ลบ queues ที่ DHCP lease หมดแล้ว
    :foreach queue in=[/queue simple find comment~"DHCP:"] do={
        :local qName [/queue simple get $queue name]
        :local qTarget [/queue simple get $queue target]
        :local ip [:pick $qTarget 0 [:find $qTarget "/"]]
        
        # ตรวจสอบว่ายังมี active lease
        :local activeLease [/ip dhcp-server lease find address=$ip status=bound]
        
        :if ([:len $activeLease] = 0) do={
            /queue simple remove $queue
            :log info ("Removed expired queue: " . $qName)
        }
    }
}

/system scheduler
add name="dhcp-queue-sync" interval=5m \
    on-event="/system script run dhcp-queue-manager"
```

---

## 48.6 User-based Queuing

### Queue ตาม User Profile

```routeros
/system script
add name="setup-user-profiles" source={
    # Define user profiles
    :local profiles {
        {"name"="BASIC"; "bw"="5M"; "burst"="10M"};
        {"name"="STANDARD"; "bw"="10M"; "burst"="20M"};
        {"name"="PREMIUM"; "bw"="20M"; "burst"="40M"};
        {"name"="BUSINESS"; "bw"="50M"; "burst"="100M"}
    }
    
    # สร้าง address lists สำหรับแต่ละ profile
    :foreach profile in=$profiles do={
        :local profileName ($profile->"name")
        :local profileBW ($profile->"bw")
        :local profileBurst ($profile->"burst")
        
        # สร้าง queue tree สำหรับ profile นี้
        :local existing [/queue tree find name=("PROF-" . $profileName)]
        
        :if ([:len $existing] = 0) do={
            /queue tree add \
                name=("PROF-" . $profileName) \
                parent=global \
                max-limit=$profileBW \
                queue=default \
                comment=("Profile: " . $profileName)
        }
        
        :log info ("Profile setup: " . $profileName . " " . $profileBW)
    }
}
```

---

## 48.7 Time-based QoS

### Script เปลี่ยน QoS ตามเวลา

```routeros
/system script
add name="time-based-qos" source={
    :local currentHour [:tonum [:pick [/system clock get time] 0 2]]
    :local currentMin [:tonum [:pick [/system clock get time] 3 5]]
    :local currentDay [/system clock get day-of-week]
    
    # กำหนด profile ตามเวลา
    :local profile "DEFAULT"
    
    # Business hours: Mon-Fri 8:00-18:00
    :if ($currentDay != "sat" && $currentDay != "sun") do={
        :if ($currentHour >= 8 && $currentHour < 18) do={
            :set profile "BUSINESS-HOURS"
        }
    }
    
    # Peak hours: 19:00-23:00 everyday
    :if ($currentHour >= 19 && $currentHour < 23) do={
        :set profile "PEAK-HOURS"
    }
    
    # Night: 00:00-06:00
    :if ($currentHour < 6) do={
        :set profile "NIGHT"
    }
    
    # Apply profile
    :if ($profile = "BUSINESS-HOURS") do={
        /queue tree set [find name=INTERNET] max-limit=80M
        /queue tree set [find name=VOIP] max-limit=20M
    } else if ($profile = "PEAK-HOURS") do={
        /queue tree set [find name=INTERNET] max-limit=60M
        /queue tree set [find name=VOIP] max-limit=10M
    } else if ($profile = "NIGHT") do={
        /queue tree set [find name=INTERNET] max-limit=95M
        /queue tree set [find name=VOIP] max-limit=5M
    } else={
        /queue tree set [find name=INTERNET] max-limit=70M
        /queue tree set [find name=VOIP] max-limit=15M
    }
    
    :log info ("QoS profile applied: " . $profile)
}

# ตรวจสอบทุก 15 นาที
/system scheduler
add name="time-qos" interval=15m \
    on-event="/system script run time-based-qos"
```

---

## 48.8 Queue Monitoring Scripts

### Monitor Queue Statistics

```routeros
/system script
add name="queue-monitor" source={
    :put "=== QUEUE MONITORING ==="
    :put ""
    :put ("Time: " . [/system clock get time])
    :put ""
    
    :put "--- Simple Queues (Top 10 by usage) ---"
    
    :local queues [/queue simple find]
    :local count 0
    
    :foreach q in=$queues do={
        :if ($count < 10) do={
            :local name [/queue simple get $q name]
            :local target [/queue simple get $q target]
            :local bytes [/queue simple get $q bytes]
            :local rate [/queue simple get $q rate]
            :local maxLimit [/queue simple get $q max-limit]
            :local dropped [/queue simple get $q dropped]
            
            :put ($name . " [" . $target . "]")
            :put ("  Rate: " . $rate)
            :put ("  Max: " . $maxLimit)
            :put ("  Bytes: " . $bytes)
            :put ("  Dropped: " . $dropped)
            :put ""
            
            :set count ($count + 1)
        }
    }
    
    :put "Total queues: " . [:len $queues]
}
```

### Alert เมื่อ Queue Drop สูง

```routeros
/system script
add name="queue-drop-alert" source={
    :local dropThreshold 1000
    
    :foreach q in=[/queue simple find] do={
        :local name [/queue simple get $q name]
        :local dropped [/queue simple get $q dropped]
        :local droppedNum [:tonum [:pick $dropped 0 [:find $dropped "/"]]]
        
        :if ($droppedNum > $dropThreshold) do={
            :log warning ("Queue " . $name . " has " . \
                $droppedNum . " dropped packets!")
        }
    }
}
```

---

## 48.9 Queue Optimization

### ปรับ Queue Parameters อัตโนมัติ

```routeros
/system script
add name="queue-optimizer" source={
    # ดู WAN bandwidth ปัจจุบัน
    :local wanIface "WAN1"
    :local wanStats [/interface monitor-traffic $wanIface once as-value]
    :local currentRx ($wanStats->"rx-bits-per-second")
    :local currentTx ($wanStats->"tx-bits-per-second")
    
    # แปลงเป็น Mbps
    :local rxMbps ($currentRx / 1000000)
    :local txMbps ($currentTx / 1000000)
    
    :log info ("WAN Traffic - RX: " . $rxMbps . " Mbps, TX: " . $txMbps . " Mbps")
    
    # ถ้า utilization สูง (> 80%) ให้ลด per-user limit
    :local maxBW 100
    :local utilization (($rxMbps * 100) / $maxBW)
    
    :if ($utilization > 80) do={
        :log warning ("WAN utilization " . $utilization . "% - adjusting queues")
        
        # ลด per-user limit ชั่วคราว
        :foreach q in=[/queue simple find comment~"BASIC-USER"] do={
            /queue simple set $q max-limit=3M/3M
        }
    } else if ($utilization < 50) do={
        # คืน bandwidth เมื่อ utilization ต่ำ
        :foreach q in=[/queue simple find comment~"BASIC-USER"] do={
            /queue simple set $q max-limit=5M/5M
        }
    }
}
```

---

## 48.10 Lab: Dynamic ISP QoS

### Lab Overview

| รายการ | รายละเอียด |
|--------|------------|
| Objective | สร้าง ISP-grade QoS ด้วย dynamic queue management |
| Users | 100 users, multiple packages |
| Queuing | PCQ + Queue Tree + Simple Queue |
| Features | Time-based policies, auto-create |

### Complete ISP QoS Setup

```routeros
# ========================================
# COMPLETE ISP QoS SYSTEM
# ========================================

# 1. ตั้งค่า PCQ Types
/queue type
add name="PCQ-DN" kind=pcq pcq-classifier=dst-address \
    pcq-rate=2M pcq-limit=100KiB
add name="PCQ-UP" kind=pcq pcq-classifier=src-address \
    pcq-rate=512k pcq-limit=50KiB

# 2. Mangle - Mark traffic
/ip firewall mangle

# Mark download traffic
add chain=forward in-interface=WAN \
    action=mark-packet new-packet-mark=ISP-DN passthrough=no

# Mark upload traffic  
add chain=forward out-interface=WAN \
    action=mark-packet new-packet-mark=ISP-UP passthrough=no

# 3. Queue Tree Structure
/queue tree
add name="ISP-TOTAL" parent=global max-limit=1G comment="Total ISP bandwidth"
add name="ISP-DOWNLOAD" parent=ISP-TOTAL max-limit=800M packet-mark=ISP-DN
add name="ISP-UPLOAD" parent=ISP-TOTAL max-limit=200M packet-mark=ISP-UP

# PCQ under download
add name="PCQ-FAIR-DN" parent=ISP-DOWNLOAD queue=PCQ-DN max-limit=800M
add name="PCQ-FAIR-UP" parent=ISP-UPLOAD queue=PCQ-UP max-limit=200M

# 4. Script auto-create per-user queues
/system script
add name="isp-user-provision" source={
    :local userIP $1
    :local package $2
    
    :local packages {
        {"name"="BASIC"; "bw"="5M"};
        {"name"="STANDARD"; "bw"="10M"};
        {"name"="PREMIUM"; "bw"="30M"}
    }
    
    :local userBW "5M"
    :foreach pkg in=$packages do={
        :if (($pkg->"name") = $package) do={
            :set userBW ($pkg->"bw")
        }
    }
    
    /queue simple add \
        name=("ISP-" . $userIP) \
        target=($userIP . "/32") \
        max-limit=($userBW . "/" . $userBW) \
        comment=("ISP User: " . $package . " Package")
    
    :log info ("ISP: Provisioned user " . $userIP . " package=" . $package)
}
```

### Lab Verification

```routeros
# ดู queue stats
/queue simple print stats

# Monitor real-time
/queue tree monitor once

# ตรวจสอบ PCQ per-user
/queue type print stats where kind=pcq
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Queue Types | Simple, Tree, PCQ comparison |
| Simple Queue | Bulk creation, management scripts |
| Queue Tree | Hierarchical bandwidth allocation |
| PCQ | Fair per-user bandwidth |
| Dynamic Creation | Auto-create based on DHCP |
| Time-based QoS | Business/peak/night profiles |
| Monitoring | Statistics, drop alerts |
| Optimization | Automatic bandwidth adjustment |

---

[← Part 47: Address List Management](part-047-address-list.md) | [Part 49: Traffic Shaping →](part-049-traffic-shaping.md)
