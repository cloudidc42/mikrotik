# Part 41: BGP Scripting และ Automation

## บทนำ

BGP (Border Gateway Protocol) เป็น routing protocol หลักที่ใช้บน Internet ซึ่งมีความซับซ้อนสูง การใช้ scripting ช่วยให้การจัดการ BGP เป็นไปอย่างอัตโนมัติ ลดข้อผิดพลาด และตอบสนองได้รวดเร็วขึ้น

---

## 41.1 BGP Fundamentals Review

### BGP คืออะไร?

BGP (Border Gateway Protocol) เป็น Exterior Gateway Protocol (EGP) ที่ใช้แลกเปลี่ยน routing information ระหว่าง Autonomous Systems (AS)

| คุณสมบัติ | รายละเอียด |
|-----------|------------|
| Protocol Type | Path-vector routing protocol |
| Transport | TCP port 179 |
| Administrative Distance | 20 (eBGP), 200 (iBGP) |
| Metric | AS-PATH, MED, LOCAL_PREF |
| Scalability | Internet-scale (800,000+ prefixes) |

### BGP Session Types

```
eBGP (External BGP)
├── เชื่อมต่อระหว่าง AS ต่างกัน
├── TTL = 1 โดยปกติ
└── Administrative Distance = 20

iBGP (Internal BGP)
├── เชื่อมต่อภายใน AS เดียวกัน
├── TTL = 255
└── Administrative Distance = 200
```

### BGP States Machine

```
IDLE → CONNECT → ACTIVE → OPENSENT → OPENCONFIRM → ESTABLISHED
```

> **Note:** BGP จะ exchange routing information ได้ก็ต่อเมื่ออยู่ใน ESTABLISHED state เท่านั้น

---

## 41.2 BGP Configuration บน MikroTik

### การตั้งค่า BGP พื้นฐาน

```routeros
# กำหนด AS Number
/routing bgp instance
set default as=65001 router-id=1.1.1.1

# เพิ่ม BGP Peer
/routing bgp peer
add name=ISP1 remote-address=203.0.113.1 remote-as=65000 \
    in-filter=BGP-IN out-filter=BGP-OUT \
    nexthop-choice=force-self

# ดู BGP Peer Status
/routing bgp peer print
```

### BGP Network Advertisement

```routeros
# ประกาศ Network ผ่าน BGP
/routing bgp network
add network=203.0.114.0/24 synchronize=no
add network=203.0.115.0/24 synchronize=no

# ดู BGP Networks
/routing bgp network print
```

### การตั้งค่า Prefix Lists

```routeros
# สร้าง prefix list สำหรับ filter
/routing filter
add chain=BGP-IN prefix=0.0.0.0/0 prefix-length=0-32 action=accept comment="Allow all"
add chain=BGP-IN action=reject

add chain=BGP-OUT prefix=203.0.114.0/24 action=accept
add chain=BGP-OUT prefix=203.0.115.0/24 action=accept
add chain=BGP-OUT action=reject
```

---

## 41.3 BGP Attributes

### Path Attributes สำคัญ

| Attribute | Type | Description | Default |
|-----------|------|-------------|---------|
| AS-PATH | Well-known mandatory | รายการ AS ที่ route ผ่านมา | - |
| NEXT_HOP | Well-known mandatory | IP ที่จะส่ง traffic ต่อ | - |
| LOCAL_PREF | Well-known discretionary | ค่า preference ใน iBGP | 100 |
| MED | Optional non-transitive | Metric สำหรับ eBGP | 0 |
| COMMUNITY | Optional transitive | Tag สำหรับ grouping routes | - |
| ORIGIN | Well-known mandatory | แหล่งที่มาของ route | - |

### การแก้ไข BGP Attributes ด้วย Scripting

```routeros
# ตั้งค่า LOCAL_PREF สำหรับ BGP routes จาก ISP1
/routing filter
add chain=BGP-IN-ISP1 bgp-communities=65000:100 \
    action=accept bgp-local-pref=150 \
    comment="Prefer ISP1 for certain communities"

add chain=BGP-IN-ISP1 bgp-communities=65000:200 \
    action=accept bgp-local-pref=50 \
    comment="Deprioritize backup routes"

add chain=BGP-IN-ISP1 action=accept bgp-local-pref=100
```

### BGP Communities

```routeros
# เพิ่ม Community ขณะ advertise ออก
/routing filter
add chain=BGP-OUT-ISP1 prefix=203.0.114.0/24 \
    action=accept bgp-communities=65001:100 \
    comment="Tag main network"

# ลบ Community เมื่อ receive จาก peer
/routing filter
add chain=BGP-IN-ISP1 action=accept \
    bgp-remove-communities=65000:999 \
    comment="Remove blackhole community from peer"
```

---

## 41.4 Route Filtering with Scripts

### Script-based Route Filter

```routeros
# สร้าง script สำหรับสร้าง BGP filter rules
/system script
add name="bgp-filter-update" source={
    # ลบ filter rules เก่า
    /routing filter remove [find chain="DYNAMIC-BGP-IN"]
    
    # อ่าน prefix list จาก file
    :local prefixList {
        "10.0.0.0/8";
        "172.16.0.0/12";
        "192.168.0.0/16";
        "100.64.0.0/10"
    }
    
    # สร้าง rule สำหรับแต่ละ prefix (bogon filter)
    :foreach prefix in $prefixList do={
        /routing filter add chain=DYNAMIC-BGP-IN \
            prefix=$prefix prefix-length=0-32 \
            action=reject \
            comment=("Bogon: " . $prefix)
    }
    
    # Accept ที่เหลือ
    /routing filter add chain=DYNAMIC-BGP-IN action=accept
    
    :log info "BGP filter updated with bogon prefixes"
}
```

### Filtering by AS-PATH

```routeros
# Filter prefixes จาก specific AS
/routing filter
add chain=BGP-IN \
    bgp-as-path=".*65100.*" \
    action=reject \
    comment="Block routes through AS65100"

# Allow only specific AS path length
/routing filter
add chain=BGP-IN \
    bgp-as-path-length=10 bgp-as-path-length-min=1 \
    action=reject \
    comment="Reject paths longer than 10 hops"
```

---

## 41.5 Dynamic BGP Prefix Lists

### Script สร้าง Prefix List แบบ Dynamic

```routeros
/system script
add name="build-bgp-prefix-list" source={
    :local listName "MY-NETWORKS"
    :local networks {
        "203.0.114.0/24";
        "203.0.115.0/24";
        "203.0.116.0/24"
    }
    
    # ลบ filter chain เก่า
    /routing filter remove [find chain=$listName]
    
    :foreach net in $networks do={
        /routing filter add \
            chain=$listName \
            prefix=$net \
            action=accept \
            comment=("Permit " . $net)
    }
    
    # Default deny
    /routing filter add chain=$listName action=reject \
        comment="Default deny"
    
    :log info ("Prefix list " . $listName . " updated with " . \
        [:len $networks] . " networks")
}
```

### อ่าน Prefix List จาก Environment

```routeros
/system script
add name="load-prefix-from-env" source={
    # สร้าง prefix list จาก address list
    :local srcList "BGP-ADVERTISE"
    :local filterChain "EXPORT-FILTER"
    
    /routing filter remove [find chain=$filterChain]
    
    :foreach entry in=[/ip firewall address-list find list=$srcList] do={
        :local addr [/ip firewall address-list get $entry address]
        
        /routing filter add \
            chain=$filterChain \
            prefix=$addr \
            action=accept \
            comment=("Auto: " . $addr)
        
        :log debug ("Added BGP filter for: " . $addr)
    }
    
    /routing filter add chain=$filterChain action=reject
}
```

---

## 41.6 BGP Monitoring Scripts

### ตรวจสอบ BGP Session Status

```routeros
/system script
add name="bgp-session-check" source={
    :local alertEmail "admin@example.com"
    :local issues 0
    
    :foreach peer in=[/routing bgp peer find] do={
        :local peerName [/routing bgp peer get $peer name]
        :local peerState [/routing bgp peer get $peer state]
        :local peerAddr [/routing bgp peer get $peer remote-address]
        :local uptime [/routing bgp peer get $peer uptime]
        
        :if ($peerState != "established") do={
            :set issues ($issues + 1)
            :log warning ("BGP peer " . $peerName . " (" . $peerAddr . \
                ") is " . $peerState)
            
            # ส่ง notification
            /tool e-mail send to=$alertEmail \
                subject=("BGP ALERT: " . $peerName . " down") \
                body=("Peer " . $peerAddr . " is " . $peerState . \
                    "\nTime: " . [/system clock get time] . \
                    " " . [/system clock get date])
        } else={
            :log info ("BGP peer " . $peerName . " OK, uptime: " . $uptime)
        }
    }
    
    :if ($issues = 0) do={
        :log info "All BGP peers are established"
    }
}
```

### ตรวจสอบ BGP Route Count

```routeros
/system script
add name="bgp-route-monitor" source={
    :local threshold 1000
    :local minRoutes 100
    
    :foreach peer in=[/routing bgp peer find] do={
        :local peerName [/routing bgp peer get $peer name]
        :local prefixCount [/routing bgp peer get $peer prefix-count]
        
        :if ($prefixCount < $minRoutes) do={
            :log warning ("BGP peer " . $peerName . \
                " has only " . $prefixCount . " routes (min: " . $minRoutes . ")")
        }
        
        :if ($prefixCount > $threshold) do={
            :log warning ("BGP peer " . $peerName . \
                " has " . $prefixCount . " routes (threshold: " . $threshold . ")")
        }
        
        :log info ("BGP peer " . $peerName . ": " . $prefixCount . " prefixes")
    }
}
```

### BGP Statistics Collector

```routeros
/system script
add name="bgp-stats-collector" source={
    :local timestamp [/system clock get time]
    :local date [/system clock get date]
    :local logFile "bgp-stats.log"
    
    :foreach peer in=[/routing bgp peer find] do={
        :local peerName [/routing bgp peer get $peer name]
        :local state [/routing bgp peer get $peer state]
        :local prefixes [/routing bgp peer get $peer prefix-count]
        :local uptime [/routing bgp peer get $peer uptime]
        :local remoteAddr [/routing bgp peer get $peer remote-address]
        :local remoteAs [/routing bgp peer get $peer remote-as]
        
        :local logLine ($date . " " . $timestamp . " | " . \
            $peerName . " | " . $remoteAddr . " | AS" . $remoteAs . \
            " | " . $state . " | prefixes=" . $prefixes . \
            " | uptime=" . $uptime)
        
        :log info $logLine
    }
}

# Schedule การเก็บ statistics ทุก 5 นาที
/system scheduler
add name="bgp-stats" interval=5m \
    on-event="/system script run bgp-stats-collector" \
    comment="Collect BGP statistics"
```

---

## 41.7 Prefix Advertisement Automation

### Auto-advertise Networks

```routeros
/system script
add name="bgp-auto-advertise" source={
    # ตรวจสอบ interface และ advertise networks ที่ up
    :foreach iface in=[/interface find running=yes] do={
        :local ifName [/interface get $iface name]
        
        # ดูว่ามี IP address บน interface นี้หรือไม่
        :foreach ipEntry in=[/ip address find interface=$ifName] do={
            :local network [/ip address get $ipEntry network]
            :local prefix [/ip address get $ipEntry address]
            
            # แปลง prefix เป็น network/mask
            :local netAddr ($network . "/" . \
                [:pick $prefix ([:find $prefix "/"] + 1) [:len $prefix]])
            
            # ตรวจสอบว่า network นี้ใน BGP แล้วหรือยัง
            :local existing [/routing bgp network find network=$netAddr]
            
            :if ([:len $existing] = 0) do={
                :if ($network != "0.0.0.0") do={
                    /routing bgp network add network=$netAddr synchronize=no
                    :log info ("Auto-advertised: " . $netAddr)
                }
            }
        }
    }
}
```

### Conditional Advertisement

```routeros
/system script
add name="bgp-conditional-advertise" source={
    # Advertise เฉพาะเมื่อมี route ใน routing table
    :local targetNetwork "203.0.114.0/24"
    :local checkRoute "203.0.114.1"
    
    # ตรวจสอบว่า route มีอยู่ใน routing table
    :local routeExists [/ip route find dst-address=$targetNetwork active=yes]
    
    :if ([:len $routeExists] > 0) do={
        # ตรวจสอบว่า advertise อยู่แล้วหรือไม่
        :local bgpNet [/routing bgp network find network=$targetNetwork]
        
        :if ([:len $bgpNet] = 0) do={
            /routing bgp network add network=$targetNetwork synchronize=no
            :log info ("Conditional advertise: " . $targetNetwork . " added")
        }
    } else={
        # ถ้า route ไม่มี ให้ถอน advertisement
        :local bgpNet [/routing bgp network find network=$targetNetwork]
        
        :if ([:len $bgpNet] > 0) do={
            /routing bgp network remove $bgpNet
            :log warning ("Conditional withdraw: " . $targetNetwork . " removed")
        }
    }
}

# Schedule ตรวจสอบทุกนาที
/system scheduler
add name="bgp-conditional" interval=1m \
    on-event="/system script run bgp-conditional-advertise"
```

---

## 41.8 BGP Session Monitoring

### Automatic Session Recovery

```routeros
/system script
add name="bgp-session-recovery" source={
    :local maxRetries 3
    
    :foreach peer in=[/routing bgp peer find] do={
        :local peerName [/routing bgp peer get $peer name]
        :local peerState [/routing bgp peer get $peer state]
        :local peerAddr [/routing bgp peer get $peer remote-address]
        
        :if ($peerState != "established") do={
            :log warning ("BGP peer " . $peerName . " is " . $peerState . \
                ", attempting reset...")
            
            # Reset peer เพื่อพยายาม re-establish
            /routing bgp peer reset $peer
            
            # รอ 10 วินาทีแล้วตรวจสอบ
            :delay 10s
            
            :local newState [/routing bgp peer get $peer state]
            :if ($newState = "established") do={
                :log info ("BGP peer " . $peerName . " recovered!")
            } else={
                :log error ("BGP peer " . $peerName . \
                    " still down after reset: " . $newState)
            }
        }
    }
}
```

### BGP Uptime Tracker

```routeros
/system script
add name="bgp-uptime-tracker" source={
    :local varName "bgp-last-check"
    
    :foreach peer in=[/routing bgp peer find] do={
        :local peerName [/routing bgp peer get $peer name]
        :local peerState [/routing bgp peer get $peer state]
        :local uptime [/routing bgp peer get $peer uptime]
        
        # เก็บ state ปัจจุบันใน script environment
        :local stateVar ("bgp-state-" . $peerName)
        :local lastState [/system script environment get $stateVar value-of=value]
        
        :if ($peerState != $lastState) do={
            :log info ("BGP peer " . $peerName . " state changed: " . \
                $lastState . " -> " . $peerState)
            
            /system script environment set $stateVar value=$peerState
        }
    }
}
```

---

## 41.9 Route Leak Prevention

### Script ป้องกัน Route Leak

```routeros
# สร้าง strict outbound filter
/routing filter
add chain=STRICT-BGP-OUT prefix=10.0.0.0/8 prefix-length=8-32 \
    action=reject comment="Block RFC1918 10.x.x.x"
add chain=STRICT-BGP-OUT prefix=172.16.0.0/12 prefix-length=12-32 \
    action=reject comment="Block RFC1918 172.16-31.x.x"
add chain=STRICT-BGP-OUT prefix=192.168.0.0/16 prefix-length=16-32 \
    action=reject comment="Block RFC1918 192.168.x.x"
add chain=STRICT-BGP-OUT prefix=100.64.0.0/10 prefix-length=10-32 \
    action=reject comment="Block Shared Address Space"
add chain=STRICT-BGP-OUT prefix=127.0.0.0/8 prefix-length=8-32 \
    action=reject comment="Block Loopback"
add chain=STRICT-BGP-OUT prefix=0.0.0.0/8 prefix-length=8-32 \
    action=reject comment="Block This Network"
add chain=STRICT-BGP-OUT prefix=169.254.0.0/16 prefix-length=16-32 \
    action=reject comment="Block Link-Local"
add chain=STRICT-BGP-OUT prefix=192.0.2.0/24 prefix-length=24-32 \
    action=reject comment="Block TEST-NET-1"
add chain=STRICT-BGP-OUT prefix=198.51.100.0/24 prefix-length=24-32 \
    action=reject comment="Block TEST-NET-2"
add chain=STRICT-BGP-OUT prefix=203.0.113.0/24 prefix-length=24-32 \
    action=reject comment="Block TEST-NET-3"
add chain=STRICT-BGP-OUT prefix=240.0.0.0/4 prefix-length=4-32 \
    action=reject comment="Block Reserved"
add chain=STRICT-BGP-OUT prefix=0.0.0.0/0 prefix-length=0-7 \
    action=reject comment="Block too-short prefixes"
add chain=STRICT-BGP-OUT prefix=0.0.0.0/0 prefix-length=25-32 \
    action=reject comment="Block too-long prefixes"
```

### Max Prefix Protection

```routeros
# ตั้งค่า maximum prefix limit
/routing bgp peer
set [find name=ISP1] in-filter=BGP-IN \
    max-prefix=1000 max-prefix-restart-time=5m \
    comment="Limit prefixes from ISP1"
```

### Script ตรวจสอบ Route Leak

```routeros
/system script
add name="route-leak-detector" source={
    :local bogonPrefixes {
        "10.0.0.0/8";
        "172.16.0.0/12";
        "192.168.0.0/16"
    }
    
    :foreach bgpRoute in=[/ip route find protocol=bgp] do={
        :local dst [/ip route get $bgpRoute dst-address]
        
        :foreach bogon in=$bogonPrefixes do={
            :local bogonNet [:pick $bogon 0 [:find $bogon "/"]]
            :local dstNet [:pick $dst 0 [:find $dst "/"]]
            
            # Simple prefix check
            :if ([:find $dst $bogonNet] = 0) do={
                :log error ("ROUTE LEAK DETECTED: BGP route " . $dst . \
                    " is a bogon prefix!")
            }
        }
    }
}
```

---

## 41.10 Lab: BGP with Dynamic Filtering

### Lab Overview

| รายการ | รายละเอียด |
|--------|------------|
| Topology | 3 routers: R1 (AS65001), R2 (AS65002), R3 (AS65003) |
| Objective | ตั้งค่า BGP พร้อม dynamic filtering และ monitoring |
| Duration | 60 นาที |

### Network Topology

```
Internet
    |
[R1: AS65001] ---- eBGP ---- [R2: AS65002]
    |                              |
[LAN: 10.1.1.0/24]       [LAN: 10.2.2.0/24]
                              |
                         [R3: AS65003]
                              |
                      [LAN: 10.3.3.0/24]
```

### Step 1: ตั้งค่า BGP Instance

```routeros
# บน R1
/routing bgp instance
set default as=65001 router-id=1.1.1.1

/routing bgp peer
add name=R2 remote-address=172.16.12.2 remote-as=65002 \
    in-filter=BGP-IN-R2 out-filter=BGP-OUT-R2

/routing bgp network
add network=10.1.1.0/24 synchronize=no
```

### Step 2: สร้าง Dynamic Prefix List Script

```routeros
# สร้าง script สำหรับ update filter
/system script
add name="update-bgp-filters" source={
    # ลบ chain เก่า
    /routing filter remove [find chain="BGP-IN-R2"]
    /routing filter remove [find chain="BGP-OUT-R2"]
    
    # สร้าง inbound filter - รับเฉพาะ 10.2.2.0/24 และ 10.3.3.0/24
    /routing filter add chain=BGP-IN-R2 prefix=10.2.2.0/24 action=accept
    /routing filter add chain=BGP-IN-R2 prefix=10.3.3.0/24 action=accept
    /routing filter add chain=BGP-IN-R2 action=reject
    
    # สร้าง outbound filter - ส่งเฉพาะ 10.1.1.0/24
    /routing filter add chain=BGP-OUT-R2 prefix=10.1.1.0/24 action=accept
    /routing filter add chain=BGP-OUT-R2 action=reject
    
    :log info "BGP filters for R2 updated"
}

/system script run update-bgp-filters
```

### Step 3: BGP Monitoring Script

```routeros
/system script
add name="bgp-lab-monitor" source={
    :put "=== BGP Status Report ==="
    :put ("Time: " . [/system clock get time] . " " . [/system clock get date])
    :put ""
    
    :foreach peer in=[/routing bgp peer find] do={
        :local name [/routing bgp peer get $peer name]
        :local state [/routing bgp peer get $peer state]
        :local remoteAs [/routing bgp peer get $peer remote-as]
        :local prefixes [/routing bgp peer get $peer prefix-count]
        :local uptime [/routing bgp peer get $peer uptime]
        
        :put ("Peer: " . $name . " (AS" . $remoteAs . ")")
        :put ("  State: " . $state)
        :put ("  Prefixes: " . $prefixes)
        :put ("  Uptime: " . $uptime)
        :put ""
    }
    
    :put "=== BGP Routes ==="
    :foreach route in=[/ip route find protocol=bgp] do={
        :local dst [/ip route get $route dst-address]
        :local gateway [/ip route get $route gateway]
        :local bgpOrigin [/ip route get $route bgp-origin]
        
        :put ($dst . " via " . $gateway . " [" . $bgpOrigin . "]")
    }
}

/system script run bgp-lab-monitor
```

### Step 4: ตั้งค่า Scheduler

```routeros
# ตรวจสอบ BGP session ทุก 5 นาที
/system scheduler
add name="bgp-check" interval=5m \
    on-event="/system script run bgp-session-check" \
    comment="BGP session monitoring"

# Update prefix list ทุกชั่วโมง
/system scheduler
add name="bgp-filter-update" interval=1h \
    on-event="/system script run update-bgp-filters" \
    comment="Update BGP filters hourly"
```

### Lab Verification

```routeros
# ตรวจสอบ BGP peers
/routing bgp peer print detail

# ตรวจสอบ BGP routes
/ip route print where protocol=bgp

# ตรวจสอบ filter rules
/routing filter print

# ทดสอบ connectivity
/ping 10.2.2.1 count=3
/ping 10.3.3.1 count=3
```

> **Warning:** ระวังการตั้งค่า BGP ผิดพลาดบน production network เพราะอาจทำให้เกิด route leak ที่ส่งผลต่อ Internet routing

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| BGP Fundamentals | AS, eBGP/iBGP, BGP states |
| BGP Configuration | Peer setup, network advertisement |
| BGP Attributes | LOCAL_PREF, MED, COMMUNITY |
| Route Filtering | Filter chains, prefix lists |
| Dynamic Filtering | Script-based filter updates |
| Monitoring | Session check, route count monitoring |
| Route Leak Prevention | Bogon filtering, max-prefix |
| Automation | Scheduled scripts for BGP management |

### สิ่งที่ต้องจำ

1. **Always filter both inbound and outbound** - อย่าปล่อย BGP peer โดยไม่มี filter
2. **Monitor prefix counts** - เพื่อตรวจจับ route leak หรือ route withdrawal
3. **Set max-prefix limits** - ป้องกัน routing table overflow
4. **Test before production** - ทดสอบ filter rules ในสภาพแวดล้อม lab ก่อน

---

[← Part 40: Advanced Scripting Techniques](part-040-advanced-scripting.md) | [Part 42: OSPF Scripting →](part-042-ospf-scripting.md)
