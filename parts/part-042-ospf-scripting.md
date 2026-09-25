# Part 42: OSPF Scripting และ Network Automation

## บทนำ

OSPF (Open Shortest Path First) เป็น Link-State routing protocol ที่นิยมใช้ภายใน Enterprise network การใช้ scripting กับ OSPF ช่วยให้การจัดการ OSPF areas, redistribution, และ monitoring เป็นไปโดยอัตโนมัติ

---

## 42.1 OSPF Fundamentals

### OSPF คืออะไร?

OSPF เป็น Interior Gateway Protocol (IGP) ที่ใช้ Dijkstra's Shortest Path First (SPF) algorithm ในการคำนวณ routing

| คุณสมบัติ | รายละเอียด |
|-----------|------------|
| Protocol Type | Link-State |
| Transport | IP Protocol 89 |
| Algorithm | Dijkstra SPF |
| Metric | Cost (100/bandwidth Mbps) |
| Convergence | เร็วกว่า RIP มาก |
| Multicast | 224.0.0.5 (All OSPF Routers), 224.0.0.6 (DR/BDR) |

### OSPF Router Types

```
Internal Router     - ทุก interface อยู่ใน area เดียว
Backbone Router     - มี interface ใน Area 0
ABR (Area Border)   - เชื่อมต่อหลาย areas
ASBR (AS Boundary)  - redistribute จาก external protocol
```

### OSPF Neighbor States

```
Down → Init → 2-Way → ExStart → Exchange → Loading → Full
```

> **Note:** OSPF จะแลกเปลี่ยน routing information ได้เมื่ออยู่ใน Full state

---

## 42.2 OSPF Areas

### Area Types

| Area Type | LSA Types | External Routes | Summary LSAs |
|-----------|-----------|-----------------|--------------|
| Backbone (Area 0) | 1,2,3,4,5 | Yes | Yes |
| Standard | 1,2,3,4,5 | Yes | Yes |
| Stub | 1,2,3 | No | Yes |
| Totally Stub | 1,2 | No | No |
| NSSA | 1,2,3,7 | Type 7 | Yes |

### การตั้งค่า OSPF Areas

```routeros
# สร้าง OSPF Instance
/routing ospf instance
add name=default router-id=1.1.1.1

# สร้าง Backbone Area
/routing ospf area
add area-id=0.0.0.0 instance=default name=backbone

# สร้าง Area 1 (Standard)
/routing ospf area
add area-id=0.0.0.1 instance=default name=area1

# สร้าง Area 2 (Stub)
/routing ospf area
add area-id=0.0.0.2 instance=default name=area2 \
    type=stub default-cost=10

# สร้าง Area 3 (NSSA)
/routing ospf area
add area-id=0.0.0.3 instance=default name=area3 \
    type=nssa
```

---

## 42.3 OSPF Configuration

### การตั้งค่า OSPF Network

```routeros
# เพิ่ม networks เข้า OSPF
/routing ospf network
add network=10.1.1.0/24 area=backbone
add network=10.1.2.0/24 area=backbone
add network=10.2.0.0/24 area=area1
add network=10.3.0.0/24 area=area2
```

### OSPF Interface Settings

```routeros
# ปรับแต่ง OSPF parameters บน interface
/routing ospf interface
add interface=ether1 cost=10 priority=100 \
    hello-interval=10s dead-interval=40s \
    retransmit-interval=5s transmit-delay=1s \
    network-type=broadcast

# P2P interface
/routing ospf interface
add interface=ether2 network-type=point-to-point cost=5

# Passive interface (ไม่ส่ง OSPF hello)
/routing ospf interface
add interface=ether3 passive=yes
```

### OSPF Authentication

```routeros
# ตั้งค่า MD5 authentication
/routing ospf interface
add interface=ether1 \
    authentication=md5 \
    authentication-key="MySecureKey123"

# หรือ Simple password
/routing ospf interface
add interface=ether1 \
    authentication=simple \
    authentication-key="password"
```

---

## 42.4 Dynamic OSPF Manipulation

### Script สำหรับ OSPF Management

```routeros
/system script
add name="ospf-auto-setup" source={
    :local routerID [/ip address get [find interface=loopback] address]
    :set routerID [:pick $routerID 0 [:find $routerID "/"]]
    
    # ตั้งค่า router ID
    /routing ospf instance set default router-id=$routerID
    :log info ("OSPF router ID set to: " . $routerID)
    
    # เพิ่ม networks จาก interfaces ที่ up
    :foreach iface in=[/interface find running=yes type!=loopback] do={
        :local ifName [/interface get $iface name]
        :local ipAddr [/ip address find interface=$ifName]
        
        :if ([:len $ipAddr] > 0) do={
            :local network [/ip address get $ipAddr network]
            :local mask [/ip address get $ipAddr address]
            :set mask [:pick $mask ([:find $mask "/"] + 1) [:len $mask]]
            
            :local netPrefix ($network . "/" . $mask)
            
            # ตรวจสอบว่า network นี้มีใน OSPF แล้วหรือยัง
            :local existing [/routing ospf network find network=$netPrefix]
            
            :if ([:len $existing] = 0) do={
                /routing ospf network add network=$netPrefix area=backbone
                :log info ("OSPF: Added network " . $netPrefix)
            }
        }
    }
}
```

### Dynamic Cost Adjustment

```routeros
/system script
add name="ospf-cost-adjuster" source={
    # ปรับ OSPF cost ตาม interface speed
    :foreach iface in=[/interface find type=ether] do={
        :local ifName [/interface get $iface name]
        :local speed [/interface ethernet get $ifName speed]
        :local cost 100
        
        :if ($speed = "100Mbps") do={ :set cost 10 }
        :if ($speed = "1Gbps") do={ :set cost 1 }
        :if ($speed = "10Gbps") do={ :set cost 1 }
        
        # อัพเดท OSPF interface cost
        :local ospfIface [/routing ospf interface find interface=$ifName]
        :if ([:len $ospfIface] > 0) do={
            /routing ospf interface set $ospfIface cost=$cost
            :log info ("OSPF: Interface " . $ifName . " cost set to " . $cost)
        }
    }
}
```

---

## 42.5 OSPF Monitoring Scripts

### ตรวจสอบ OSPF Neighbors

```routeros
/system script
add name="ospf-neighbor-check" source={
    :local alertThreshold 0
    :local downNeighbors 0
    
    :put "=== OSPF Neighbor Status ==="
    
    :foreach neighbor in=[/routing ospf neighbor find] do={
        :local routerID [/routing ospf neighbor get $neighbor router-id]
        :local state [/routing ospf neighbor get $neighbor state]
        :local area [/routing ospf neighbor get $neighbor area]
        :local iface [/routing ospf neighbor get $neighbor interface]
        :local priority [/routing ospf neighbor get $neighbor priority]
        
        :put ("Neighbor: " . $routerID)
        :put ("  Interface: " . $iface)
        :put ("  Area: " . $area)
        :put ("  State: " . $state)
        :put ("  Priority: " . $priority)
        :put ""
        
        :if ($state != "full" && $state != "2-way") do={
            :set downNeighbors ($downNeighbors + 1)
            :log warning ("OSPF: Neighbor " . $routerID . " is in " . $state . " state!")
        }
    }
    
    :if ($downNeighbors > 0) do={
        :log error ("OSPF: " . $downNeighbors . " neighbor(s) not in Full state!")
    } else={
        :log info "OSPF: All neighbors are in Full state"
    }
}
```

### OSPF Route Monitor

```routeros
/system script
add name="ospf-route-monitor" source={
    :local totalRoutes 0
    :local externalRoutes 0
    :local interAreaRoutes 0
    :local intraAreaRoutes 0
    
    :foreach route in=[/ip route find protocol=ospf] do={
        :set totalRoutes ($totalRoutes + 1)
        :local ospfType [/ip route get $route ospf-type]
        
        :if ($ospfType = "ext-type-1" || $ospfType = "ext-type-2") do={
            :set externalRoutes ($externalRoutes + 1)
        }
        :if ($ospfType = "inter-area") do={
            :set interAreaRoutes ($interAreaRoutes + 1)
        }
        :if ($ospfType = "intra-area") do={
            :set intraAreaRoutes ($intraAreaRoutes + 1)
        }
    }
    
    :log info ("OSPF Routes: Total=" . $totalRoutes . \
        " IntraArea=" . $intraAreaRoutes . \
        " InterArea=" . $interAreaRoutes . \
        " External=" . $externalRoutes)
}
```

---

## 42.6 Route Redistribution

### Redistribute Static Routes เข้า OSPF

```routeros
# ตั้งค่า redistribution
/routing ospf instance
set default redistribute-static=as-type-1 \
    redistribute-connected=as-type-2 \
    redistribute-bgp=as-type-1

# กำหนด metric สำหรับ redistribution
/routing ospf instance
set default redistribute-static=as-type-1 \
    metric-bgp=20 metric-static=30 metric-connected=10
```

### Script สำหรับ Conditional Redistribution

```routeros
/system script
add name="ospf-redistribution-control" source={
    :local redistributeStatic true
    :local staticRouteCount [/ip route find protocol=static]
    
    # ถ้ามี static routes มากกว่า 10 routes ให้ redistribute
    :if ([:len $staticRouteCount] > 10) do={
        /routing ospf instance set default redistribute-static=as-type-1
        :log info "OSPF: Static redistribution enabled"
    } else={
        /routing ospf instance set default redistribute-static=no
        :log info "OSPF: Static redistribution disabled (too few routes)"
    }
}
```

### Filter สำหรับ Redistribution

```routeros
# สร้าง route filter สำหรับ redistribution
/routing filter
add chain=OSPF-REDIST-IN \
    prefix=10.0.0.0/8 prefix-length=8-32 \
    action=accept \
    comment="Accept internal networks"

add chain=OSPF-REDIST-IN \
    prefix=0.0.0.0/0 prefix-length=0-32 \
    action=reject \
    comment="Reject everything else"

# ใช้ filter กับ redistribution
/routing ospf instance
set default in-filter=OSPF-REDIST-IN
```

---

## 42.7 OSPF Authentication

### Script ตั้งค่า Authentication แบบ Batch

```routeros
/system script
add name="ospf-auth-setup" source={
    :local ospfInterfaces {
        "ether1";"ether2";"ether3"
    }
    :local authKey "SecureOSPFKey2024!"
    
    :foreach iface in=$ospfInterfaces do={
        :local ospfIface [/routing ospf interface find interface=$iface]
        
        :if ([:len $ospfIface] > 0) do={
            /routing ospf interface set $ospfIface \
                authentication=md5 \
                authentication-key=$authKey
            :log info ("OSPF: Auth configured on " . $iface)
        } else={
            :log warning ("OSPF: Interface " . $iface . " not in OSPF")
        }
    }
}
```

### ตรวจสอบ Authentication Issues

```routeros
/system script
add name="ospf-auth-check" source={
    # ตรวจสอบ neighbors ที่ไม่ reach full state (อาจเป็นปัญหา auth)
    :foreach neighbor in=[/routing ospf neighbor find] do={
        :local state [/routing ospf neighbor get $neighbor state]
        :local routerID [/routing ospf neighbor get $neighbor router-id]
        :local iface [/routing ospf neighbor get $neighbor interface]
        
        :if ($state = "init" || $state = "2-way") do={
            :log warning ("OSPF: Neighbor " . $routerID . " on " . \
                $iface . " stuck in " . $state . \
                " - possible auth mismatch!")
        }
    }
}
```

---

## 42.8 Troubleshooting OSPF

### OSPF Debug Script

```routeros
/system script
add name="ospf-troubleshoot" source={
    :put "=== OSPF Troubleshooting Report ==="
    :put ("Date: " . [/system clock get date] . \
        " Time: " . [/system clock get time])
    :put ""
    
    # 1. OSPF Instance Status
    :put "--- OSPF Instances ---"
    /routing ospf instance print
    :put ""
    
    # 2. OSPF Areas
    :put "--- OSPF Areas ---"
    /routing ospf area print
    :put ""
    
    # 3. OSPF Networks
    :put "--- OSPF Networks ---"
    /routing ospf network print
    :put ""
    
    # 4. OSPF Interfaces
    :put "--- OSPF Interfaces ---"
    /routing ospf interface print
    :put ""
    
    # 5. OSPF Neighbors
    :put "--- OSPF Neighbors ---"
    /routing ospf neighbor print
    :put ""
    
    # 6. OSPF Routes
    :put "--- OSPF Routes ---"
    /ip route print where protocol=ospf
    :put ""
    
    # 7. Common Issues Check
    :put "--- Issues Summary ---"
    
    :local neighborsDown 0
    :foreach n in=[/routing ospf neighbor find] do={
        :local state [/routing ospf neighbor get $n state]
        :if ($state != "full") do={
            :set neighborsDown ($neighborsDown + 1)
        }
    }
    
    :if ($neighborsDown > 0) do={
        :put ("WARNING: " . $neighborsDown . " neighbor(s) not Full!")
    } else={
        :put "OK: All neighbors are Full"
    }
    
    :local ospfRoutes [/ip route find protocol=ospf]
    :put ("INFO: " . [:len $ospfRoutes] . " OSPF routes in table")
}
```

---

## 42.9 Multi-area OSPF

### Script ตั้งค่า Multi-area OSPF

```routeros
/system script
add name="multiarea-ospf-setup" source={
    # สร้าง OSPF areas
    :local areas {
        {"id"="0.0.0.0"; "name"="backbone"; "type"="default"};
        {"id"="0.0.0.1"; "name"="area1"; "type"="default"};
        {"id"="0.0.0.2"; "name"="area2"; "type"="stub"};
        {"id"="0.0.0.3"; "name"="area3"; "type"="nssa"}
    }
    
    :foreach area in=$areas do={
        :local areaId ($area->"id")
        :local areaName ($area->"name")
        :local areaType ($area->"type")
        
        :local existing [/routing ospf area find area-id=$areaId]
        
        :if ([:len $existing] = 0) do={
            /routing ospf area add \
                area-id=$areaId \
                name=$areaName \
                type=$areaType
            :log info ("OSPF: Created area " . $areaName . " (" . $areaId . ")")
        }
    }
}
```

### ABR Route Summarization

```routeros
# ตั้งค่า Area Range (สำหรับ ABR)
/routing ospf area range
add area=area1 prefix=10.1.0.0/16 advertise=yes cost=10
add area=area2 prefix=10.2.0.0/16 advertise=yes cost=20
add area=area3 prefix=10.3.0.0/16 advertise=yes cost=30

# ตรวจสอบ area ranges
/routing ospf area range print
```

---

## 42.10 Lab: OSPF Network Automation

### Lab Overview

| รายการ | รายละเอียด |
|--------|------------|
| Topology | 4 routers, 3 OSPF areas |
| Objective | Automate OSPF configuration และ monitoring |
| Duration | 90 นาที |

### Network Topology

```
                   Area 0 (Backbone)
                        |
              [R1: 1.1.1.1] ---- [R2: 2.2.2.2]
              /  ABR              ABR  \
             /                          \
      Area 1                          Area 2
     [R3: 3.3.3.3]               [R4: 4.4.4.4]
     10.1.0.0/24                  10.2.0.0/24
```

### Step 1: ตั้งค่า OSPF บน R1 (Backbone + ABR)

```routeros
# R1 - Backbone Router / ABR
/routing ospf instance
add name=default router-id=1.1.1.1

/routing ospf area
add area-id=0.0.0.0 name=backbone instance=default
add area-id=0.0.0.1 name=area1 instance=default

/routing ospf network
add network=192.168.12.0/30 area=backbone
add network=10.1.0.0/24 area=area1

/routing ospf interface
add interface=ether1 network-type=point-to-point
add interface=ether2 network-type=broadcast priority=100
```

### Step 2: Script Auto-configure OSPF

```routeros
/system script
add name="lab-ospf-auto" source={
    :put "=== OSPF Auto Configuration ==="
    
    # ตรวจสอบว่า OSPF instance มีหรือยัง
    :local instance [/routing ospf instance find name=default]
    :if ([:len $instance] = 0) do={
        :put "Creating OSPF instance..."
        /routing ospf instance add name=default router-id=1.1.1.1
    }
    
    # ตรวจสอบและสร้าง areas
    :local requiredAreas {
        "0.0.0.0";"0.0.0.1"
    }
    :foreach areaId in=$requiredAreas do={
        :local existing [/routing ospf area find area-id=$areaId]
        :if ([:len $existing] = 0) do={
            :put ("Creating area: " . $areaId)
            /routing ospf area add area-id=$areaId instance=default
        }
    }
    
    # เพิ่ม interface
    :foreach iface in=[/interface find running=yes type=ether] do={
        :local ifName [/interface get $iface name]
        :local existing [/routing ospf interface find interface=$ifName]
        
        :if ([:len $existing] = 0) do={
            /routing ospf interface add interface=$ifName
            :put ("Added OSPF interface: " . $ifName)
        }
    }
    
    :put "OSPF configuration complete!"
}
```

### Step 3: OSPF Monitoring Dashboard

```routeros
/system script
add name="lab-ospf-dashboard" source={
    :put "╔══════════════════════════════════╗"
    :put "║     OSPF MONITORING DASHBOARD    ║"
    :put "╚══════════════════════════════════╝"
    :put ""
    
    # OSPF Instance Info
    :local routerID [/routing ospf instance get default router-id]
    :put ("Router ID: " . $routerID)
    :put ""
    
    # Neighbor Summary
    :local totalNeighbors [/routing ospf neighbor find]
    :local fullNeighbors [/routing ospf neighbor find state=full]
    
    :put ("Neighbors: " . [:len $fullNeighbors] . "/" . \
        [:len $totalNeighbors] . " Full")
    :put ""
    
    # Per-neighbor details
    :foreach n in=[/routing ospf neighbor find] do={
        :local nID [/routing ospf neighbor get $n router-id]
        :local nState [/routing ospf neighbor get $n state]
        :local nIface [/routing ospf neighbor get $n interface]
        :local nArea [/routing ospf neighbor get $n area]
        
        :local status ""
        :if ($nState = "full") do={ :set status "[OK]" }
        :if ($nState != "full") do={ :set status "[!!]" }
        
        :put ($status . " " . $nID . " on " . $nIface . " (" . $nArea . "): " . $nState)
    }
    :put ""
    
    # Route Summary
    :local ospfRoutes [/ip route find protocol=ospf]
    :put ("OSPF Routes: " . [:len $ospfRoutes])
    
    :foreach r in=$ospfRoutes do={
        :local dst [/ip route get $r dst-address]
        :local gw [/ip route get $r gateway]
        :put ("  " . $dst . " via " . $gw)
    }
}
```

### Step 4: ตั้งค่า Scheduled Monitoring

```routeros
/system scheduler
add name="ospf-monitor" interval=2m \
    on-event="/system script run ospf-neighbor-check"

/system scheduler
add name="ospf-dashboard" interval=10m \
    on-event="/system script run lab-ospf-dashboard"
```

### Verification Commands

```routeros
# ดู OSPF neighbors
/routing ospf neighbor print detail

# ดู OSPF routes
/ip route print where protocol=ospf

# ดู OSPF LSA database
/routing ospf lsa print

# Debug OSPF (ใช้ระวัง บน production)
/routing ospf interface print stats
```

> **Warning:** การเปลี่ยนแปลง OSPF configuration ใน production network ควรทำในช่วง maintenance window เพราะอาจทำให้เกิด network interruption

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| OSPF Fundamentals | Area types, router roles, states |
| Configuration | Instance, areas, networks, interfaces |
| Authentication | MD5 auth setup |
| Monitoring | Neighbor check, route monitoring |
| Redistribution | Static/connected/BGP redistribution |
| Troubleshooting | Debug scripts, issue detection |
| Multi-area | ABR setup, route summarization |
| Automation | Auto-configure และ scheduled monitoring |

---

[← Part 41: BGP Scripting](part-041-bgp-scripting.md) | [Part 43: Load Balancing →](part-043-load-balancing.md)
