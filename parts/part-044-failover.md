# Part 44: Failover System บน MikroTik

## บทนำ

Failover คือกระบวนการที่ระบบเปลี่ยนไปใช้ link สำรองโดยอัตโนมัติเมื่อ link หลักล้มเหลว การทำ failover ที่ดีต้องตรวจจับ failure ได้รวดเร็ว, เปลี่ยน route อย่างถูกต้อง, และแจ้งเตือน admin

---

## 44.1 Failover Concepts

### ประเภทของ Failover

| ประเภท | คำอธิบาย | Recovery Time |
|--------|----------|---------------|
| Active-Passive | WAN1 active, WAN2 standby | 30-60 วินาที |
| Active-Active | Load balance + failover | ทันที |
| Hot Standby | Pre-configured standby | < 5 วินาที |

### ปัจจัยที่ต้องพิจารณา

```
Detection Speed     - ตรวจจับ failure ได้เร็วแค่ไหน?
Recovery Speed      - เปลี่ยน route ได้เร็วแค่ไหน?
False Positive      - ตรวจจับ failure ผิดพลาดบ้างไหม?
Notification        - แจ้งเตือน admin ได้ไหม?
Return to Primary   - กลับมาใช้ primary อัตโนมัติหรือไม่?
```

---

## 44.2 Netwatch-based Failover

### Netwatch คืออะไร?

Netwatch เป็น built-in tool ใน RouterOS ที่ ping target ทุก N วินาที และรัน script เมื่อ status เปลี่ยน

### การตั้งค่า Netwatch พื้นฐาน

```routeros
# ตรวจสอบ WAN1 gateway ทุก 10 วินาที
/tool netwatch
add host=203.0.113.1 interval=10s \
    up-script="/system script run wan1-up" \
    down-script="/system script run wan1-down" \
    comment="WAN1 Gateway Monitor"

# ตรวจสอบ Internet connectivity ผ่าน WAN1
/tool netwatch
add host=8.8.8.8 interval=15s \
    routing-table=TO-WAN1 \
    up-script="/system script run wan1-internet-up" \
    down-script="/system script run wan1-internet-down" \
    comment="WAN1 Internet Monitor"
```

### Script WAN1 Down

```routeros
/system script
add name="wan1-down" source={
    :log warning "WAN1 is DOWN - initiating failover to WAN2"
    
    # ลบ default route ของ WAN1 (หรือ set distance สูงขึ้น)
    /ip route set [find comment="WAN1-DEFAULT"] disabled=yes
    
    # Enable WAN2 default route
    /ip route set [find comment="WAN2-DEFAULT"] disabled=no
    
    # แจ้งเตือนผ่าน email
    /tool e-mail send \
        to="admin@example.com" \
        subject="[ALERT] WAN1 Failover Activated" \
        body=("WAN1 is DOWN at " . [/system clock get time] . \
            " on " . [/system clock get date] . \
            "\n\nAutomatic failover to WAN2 has been activated." . \
            "\n\nRouter: " . [/system identity get name])
    
    # อัพเดท status variable
    /system script environment set WAN1_STATUS value=DOWN
    
    :log info "Failover to WAN2 completed"
}
```

### Script WAN1 Up

```routeros
/system script
add name="wan1-up" source={
    :log info "WAN1 is UP - restoring primary route"
    
    # Enable WAN1 route กลับ
    /ip route set [find comment="WAN1-DEFAULT"] disabled=no
    
    # Disable WAN2 default route (optional - ขึ้นอยู่กับ policy)
    /ip route set [find comment="WAN2-DEFAULT"] disabled=yes
    
    # แจ้งเตือน
    /tool e-mail send \
        to="admin@example.com" \
        subject="[RECOVERY] WAN1 Restored" \
        body=("WAN1 has been restored at " . [/system clock get time] . \
            "\n\nPrimary route has been re-activated.")
    
    /system script environment set WAN1_STATUS value=UP
    
    :log info "WAN1 route restored"
}
```

---

## 44.3 Recursive Routing Failover

### Recursive Routing คืออะไร?

การใช้ recursive route โดย default route จะ active ก็ต่อเมื่อ check route ยังมีอยู่

```routeros
# ตั้งค่า check routes (ใช้ ISP gateway IP)
/ip route
add dst-address=203.0.113.1/32 gateway=203.0.113.1 \
    scope=10 comment="WAN1-CHECK"
    
add dst-address=198.51.100.1/32 gateway=198.51.100.1 \
    scope=10 comment="WAN2-CHECK"

# Default routes ที่ depend on check routes
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 \
    check-gateway=ping distance=1 \
    comment="WAN1-DEFAULT"
    
add dst-address=0.0.0.0/0 gateway=198.51.100.1 \
    check-gateway=ping distance=2 \
    comment="WAN2-DEFAULT"
```

> **Note:** RouterOS จะใช้ route ที่มี distance ต่ำกว่าก่อน เมื่อ WAN1 (distance=1) ล้มเหลว จะ fallback ไป WAN2 (distance=2) อัตโนมัติ

### ตรวจสอบ Recursive Routing

```routeros
# ดู routing table
/ip route print detail where dst-address=0.0.0.0/0

# ทดสอบ
/ping 8.8.8.8 count=5
```

---

## 44.4 IP SLA Equivalent

### SLA Monitoring ด้วย Script

```routeros
/system script
add name="sla-monitor" source={
    :local targets {
        "8.8.8.8";"8.8.4.4";"1.1.1.1"
    }
    :local successCount 0
    :local totalTargets [:len $targets]
    
    :foreach target in=$targets do={
        :local result [/ping $target count=3 interval=500ms as-value]
        :local received ($result->"received")
        :local avgRtt ($result->"avg-rtt")
        
        :if ($received >= 2) do={
            :set successCount ($successCount + 1)
            :log debug ("SLA: " . $target . " OK, avg RTT: " . $avgRtt . "ms")
        } else={
            :log warning ("SLA: " . $target . " FAILED, received=" . $received)
        }
    }
    
    # คำนวณ success rate
    :local successRate (($successCount * 100) / $totalTargets)
    :log info ("SLA: Success rate " . $successRate . "% (" . \
        $successCount . "/" . $totalTargets . " targets)")
    
    # ถ้า success rate < 50% ถือว่า WAN มีปัญหา
    :if ($successRate < 50) do={
        :log error "SLA: Internet connectivity degraded!"
        /system script run activate-failover
    }
}
```

---

## 44.5 Automatic Failover Scripts

### Complete Failover Manager

```routeros
/system script
add name="failover-manager" source={
    :local wan1GW "203.0.113.1"
    :local wan2GW "198.51.100.1"
    :local checkTarget "8.8.8.8"
    :local pingCount 5
    :local pingInterval 200ms
    :local failThreshold 3  # ต้อง fail ติดกัน 3 ครั้ง
    
    # อ่าน current state
    :local wan1Fails 0
    :local wan2Fails 0
    
    :do {
        :set wan1Fails [/system script environment get WAN1_FAIL_COUNT value-of=value]
    } on-error={}
    
    :do {
        :set wan2Fails [/system script environment get WAN2_FAIL_COUNT value-of=value]
    } on-error={}
    
    # ตรวจสอบ WAN1
    :local wan1Result [/ping $checkTarget routing-table=TO-WAN1 \
        count=$pingCount interval=$pingInterval as-value]
    :local wan1Received ($wan1Result->"received")
    
    :if ($wan1Received < 3) do={
        :set wan1Fails ($wan1Fails + 1)
        :log warning ("WAN1 check failed (count: " . $wan1Fails . ")")
    } else={
        :if ($wan1Fails > 0) do={
            :log info "WAN1 check passed, resetting failure count"
            :set wan1Fails 0
        }
    }
    
    # ตรวจสอบ WAN2
    :local wan2Result [/ping $checkTarget routing-table=TO-WAN2 \
        count=$pingCount interval=$pingInterval as-value]
    :local wan2Received ($wan2Result->"received")
    
    :if ($wan2Received < 3) do={
        :set wan2Fails ($wan2Fails + 1)
        :log warning ("WAN2 check failed (count: " . $wan2Fails . ")")
    } else={
        :if ($wan2Fails > 0) do={
            :log info "WAN2 check passed, resetting failure count"
            :set wan2Fails 0
        }
    }
    
    # บันทึก counters
    /system script environment set WAN1_FAIL_COUNT value=$wan1Fails
    /system script environment set WAN2_FAIL_COUNT value=$wan2Fails
    
    # Failover logic
    :if ($wan1Fails >= $failThreshold) do={
        :local currentRoute [/ip route get [find comment="WAN1-DEFAULT"] disabled]
        :if (!$currentRoute) do={
            /ip route set [find comment="WAN1-DEFAULT"] disabled=yes
            :log error "FAILOVER: WAN1 disabled after " . $wan1Fails . " failures"
            /system script run send-failover-alert
        }
    } else if ($wan1Fails = 0) do={
        :local currentRoute [/ip route get [find comment="WAN1-DEFAULT"] disabled]
        :if ($currentRoute) do={
            /ip route set [find comment="WAN1-DEFAULT"] disabled=no
            :log info "FAILOVER: WAN1 restored"
            /system script run send-recovery-alert
        }
    }
}
```

---

## 44.6 Failover Notification

### Email Notification

```routeros
/system script
add name="send-failover-alert" source={
    :local hostname [/system identity get name]
    :local time [/system clock get time]
    :local date [/system clock get date]
    :local wan1State [/ip route get [find comment="WAN1-DEFAULT"] disabled]
    :local wan2State [/ip route get [find comment="WAN2-DEFAULT"] disabled]
    
    :local body ("FAILOVER ALERT\n\n")
    :set body ($body . "Router: " . $hostname . "\n")
    :set body ($body . "Time: " . $date . " " . $time . "\n\n")
    :set body ($body . "WAN1 Status: ")
    :if ($wan1State) do={
        :set body ($body . "DOWN (disabled)\n")
    } else={
        :set body ($body . "UP (active)\n")
    }
    :set body ($body . "WAN2 Status: ")
    :if ($wan2State) do={
        :set body ($body . "DOWN (disabled)\n")
    } else={
        :set body ($body . "UP (active)\n")
    }
    
    /tool e-mail send \
        to="admin@example.com" \
        subject=("[FAILOVER] " . $hostname . " WAN change detected") \
        body=$body
    
    :log info "Failover notification sent"
}
```

### Telegram Notification

```routeros
/system script
add name="notify-telegram" source={
    :local botToken "YOUR_BOT_TOKEN"
    :local chatId "YOUR_CHAT_ID"
    :local message $1
    
    :local url ("https://api.telegram.org/bot" . $botToken . \
        "/sendMessage?chat_id=" . $chatId . \
        "&text=" . $message . \
        "&parse_mode=HTML")
    
    /tool fetch url=$url keep-result=no
    :log info ("Telegram notification sent: " . $message)
}

# ใช้งาน
/system script run notify-telegram message="WAN1 is DOWN\\!"
```

---

## 44.7 Recovery Procedures

### Auto Recovery Script

```routeros
/system script
add name="auto-recovery" source={
    :local recoveryDelay 120s
    :local maxRetries 5
    :local retryCount 0
    
    :log info "Starting auto-recovery procedure..."
    
    :while ($retryCount < $maxRetries) do={
        :delay $recoveryDelay
        
        # ทดสอบ WAN1
        :local result [/ping 8.8.8.8 routing-table=TO-WAN1 \
            count=5 as-value]
        
        :if (($result->"received") >= 4) do={
            # WAN1 กลับมาแล้ว
            /ip route set [find comment="WAN1-DEFAULT"] disabled=no
            /ip route set [find comment="WAN2-DEFAULT"] disabled=yes
            
            :log info "Auto-recovery: WAN1 restored successfully"
            /system script run send-recovery-alert
            
            # หยุด loop
            :set retryCount $maxRetries
        } else={
            :set retryCount ($retryCount + 1)
            :log warning ("Auto-recovery: Attempt " . $retryCount . \
                "/" . $maxRetries . " failed")
        }
    }
    
    :if ($retryCount >= $maxRetries) do={
        :log error "Auto-recovery: WAN1 could not be restored after " . \
            $maxRetries . " attempts"
    }
}
```

---

## 44.8 Testing Failover

### Failover Test Script

```routeros
/system script
add name="test-failover" source={
    :put "=== FAILOVER TEST ==="
    :put ""
    :put "1. Testing normal state (WAN1 primary)..."
    
    :local result [/ping 8.8.8.8 count=5 as-value]
    :put ("   Internet ping: " . ($result->"received") . "/5 packets")
    
    :put ""
    :put "2. Simulating WAN1 failure..."
    /ip route set [find comment="WAN1-DEFAULT"] disabled=yes
    :delay 2s
    
    :put "3. Testing failover to WAN2..."
    :local result2 [/ping 8.8.8.8 count=5 as-value]
    :put ("   Internet ping via WAN2: " . ($result2->"received") . "/5 packets")
    
    :if (($result2->"received") >= 4) do={
        :put "   FAILOVER TEST: PASSED"
    } else={
        :put "   FAILOVER TEST: FAILED - No connectivity via WAN2!"
    }
    
    :put ""
    :put "4. Restoring WAN1..."
    /ip route set [find comment="WAN1-DEFAULT"] disabled=no
    :delay 2s
    
    :local result3 [/ping 8.8.8.8 count=5 as-value]
    :put ("   Internet ping after restore: " . ($result3->"received") . "/5 packets")
    :put ""
    :put "=== TEST COMPLETE ==="
}
```

---

## 44.9 Complex Failover Scenarios

### Multi-ISP Failover

```routeros
# 3 WAN links: ISP1 (primary), ISP2 (secondary), ISP3 (emergency)
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 \
    distance=1 check-gateway=ping comment="ISP1-PRIMARY"
    
add dst-address=0.0.0.0/0 gateway=198.51.100.1 \
    distance=2 check-gateway=ping comment="ISP2-SECONDARY"
    
add dst-address=0.0.0.0/0 gateway=192.0.2.1 \
    distance=3 check-gateway=ping comment="ISP3-EMERGENCY"
```

### Application-Specific Failover

```routeros
/system script
add name="app-failover" source={
    # ตรวจสอบ specific applications/services
    :local services {
        {"name"="DNS"; "host"="8.8.8.8"; "port"=53; "proto"="udp"};
        {"name"="HTTP"; "host"="1.1.1.1"; "port"=80; "proto"="tcp"}
    }
    
    :foreach svc in=$services do={
        :local svcName ($svc->"name")
        :local svcHost ($svc->"host")
        
        :local pingResult [/ping $svcHost count=3 as-value]
        :if (($pingResult->"received") = 0) do={
            :log warning ("Service " . $svcName . " unreachable!")
        }
    }
}
```

---

## 44.10 Lab: Complete Failover System

### Lab Overview

| รายการ | รายละเอียด |
|--------|------------|
| WAN1 | ISP1 - Primary link |
| WAN2 | ISP2 - Failover link |
| Detection | Netwatch + Script |
| Recovery | Auto-recovery after 2 minutes |

### Complete Lab Setup

```routeros
# ========================================
# COMPLETE FAILOVER SYSTEM SETUP
# ========================================

# 1. IP Routes ด้วย check-gateway
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 \
    distance=1 check-gateway=ping comment="WAN1-DEFAULT"
    
add dst-address=0.0.0.0/0 gateway=198.51.100.1 \
    distance=2 comment="WAN2-DEFAULT"

# 2. Netwatch ตรวจสอบ WAN1
/tool netwatch
add host=203.0.113.1 interval=30s timeout=5s \
    up-script=":log info WAN1-GW-UP; /system script run wan1-recover" \
    down-script=":log error WAN1-GW-DOWN; /system script run wan1-fail"

# 3. Script สำหรับ WAN1 failure
/system script
add name="wan1-fail" source={
    :log warning "WAN1 Gateway unreachable!"
    
    # ลบ check gateway เพื่อให้ traffic fallback ไป WAN2
    /ip route set [find comment="WAN1-DEFAULT"] check-gateway=none distance=200
    
    :log info "Traffic redirected to WAN2"
    /system script run notify-telegram message="[ALERT] WAN1 DOWN - Failover to WAN2"
}

# 4. Script สำหรับ WAN1 recovery
/system script
add name="wan1-recover" source={
    :log info "WAN1 Gateway is back!"
    
    # Restore primary route
    /ip route set [find comment="WAN1-DEFAULT"] check-gateway=ping distance=1
    
    :log info "WAN1 route restored"
    /system script run notify-telegram message="[OK] WAN1 RESTORED - Back to primary"
}

# 5. Scheduler สำหรับ periodic check
/system scheduler
add name="failover-health" interval=1m \
    on-event="/system script run failover-manager" \
    comment="Failover health check every minute"
```

### การทดสอบ Lab

```routeros
# ทดสอบ failover
/system script run test-failover

# Monitor logs
/log print where topics~"warning" or topics~"error"

# ดู routing table
/ip route print where dst-address=0.0.0.0/0
```

> **Tip:** ควร test failover ในช่วง off-peak hours เพื่อลด impact ต่อ users

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Netwatch | Monitor-based failover |
| Recursive Routing | check-gateway failover |
| IP SLA | Custom service monitoring |
| Scripts | Automated failover/recovery |
| Notifications | Email, Telegram alerts |
| Testing | Failover testing procedure |
| Multi-ISP | Triple WAN failover |

---

[← Part 43: Load Balancing](part-043-load-balancing.md) | [Part 45: Port Knocking →](part-045-port-knocking.md)
