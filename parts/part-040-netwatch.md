# Part 40: Netwatch ใน RouterOS

## บทนำ

Netwatch เป็น built-in tool ของ RouterOS สำหรับ monitor hosts โดยการ ping และ trigger scripts เมื่อ host เปลี่ยน state ระหว่าง up และ down เหมาะสำหรับ failover automation และ alerting

---

## 40.1 Netwatch Overview

```routeros
# Netwatch ทำงานอย่างไร:
# 1. Ping host ตาม interval ที่กำหนด
# 2. ถ้า host ไม่ตอบภายใน timeout -> down
# 3. ถ้า host ตอบ -> up
# 4. เมื่อ state เปลี่ยน -> run up-script หรือ down-script

# ดู netwatch hosts
/tool netwatch print

# Netwatch status
/tool netwatch print status
```

---

## 40.2 Basic Host Monitoring

```routeros
# Monitor gateway
/tool netwatch add \
    host=192.168.1.1 \
    interval=10s \
    timeout=2s \
    up-script=":log info \"Gateway 192.168.1.1 is UP\"" \
    down-script=":log error \"Gateway 192.168.1.1 is DOWN\""

# Monitor Google DNS
/tool netwatch add \
    host=8.8.8.8 \
    interval=30s \
    timeout=3s \
    up-script=":log info \"Internet UP (8.8.8.8)\"" \
    down-script="
        :log warning \"Internet DOWN (8.8.8.8)\"
        /tool e-mail send to=\"admin@company.com\" \
            subject=\"Alert: Internet is DOWN\" \
            body=\"Cannot reach 8.8.8.8 - Internet may be down\"
    "

# Monitor server
/tool netwatch add \
    host=10.0.0.100 \
    interval=1m \
    timeout=5s \
    up-script=":log info \"Server 10.0.0.100 OK\"" \
    down-script="
        :log error \"Server 10.0.0.100 DOWN\"
        /tool e-mail send to=\"admin@company.com\" \
            subject=\"Critical: Server Down\" \
            body=\"Server 10.0.0.100 is not responding\"
    "

# Script เพิ่ม netwatch หลาย hosts
:local hosts {"8.8.8.8"; "8.8.4.4"; "1.1.1.1"; "9.9.9.9"}
:foreach host in=$hosts do={
    :local existing [/tool netwatch find host=$host]
    :if ([:len $existing] = 0) do={
        /tool netwatch add \
            host=$host \
            interval=30s \
            up-script=(":log info \"" . $host . " UP\"") \
            down-script=(":log warning \"" . $host . " DOWN\"")
        :put "Added monitoring: $host"
    }
}
```

---

## 40.3 Up/Down Scripts

```routeros
# Up script - run เมื่อ host กลับมา UP
/tool netwatch set [find host=8.8.8.8] up-script="
    :global internetDown
    :if (\$internetDown) do={
        :set internetDown false
        :log info \"Internet RESTORED\"
        /tool e-mail send to=\"admin@company.com\" \
            subject=\"Internet RESTORED\" \
            body=\"Internet connection has been restored at \" . [/system clock get time]
    }
"

# Down script - run เมื่อ host DOWN
/tool netwatch set [find host=8.8.8.8] down-script="
    :global internetDown
    :set internetDown true
    :log warning \"Internet DOWN detected\"
    /tool e-mail send to=\"admin@company.com\" \
        subject=\"Internet DOWN\" \
        body=\"Internet connection lost at \" . [/system clock get time]
"
```

---

## 40.4 Latency Monitoring

```routeros
# Netwatch ไม่ track latency โดยตรง แต่ใช้ ping script แทน
/system script add name="latency-check" source="
    :local hosts {\"8.8.8.8\"; \"1.1.1.1\"}
    :local threshold 100  # ms
    
    :foreach host in=\$hosts do={
        :local result [/tool ping address=\$host count=3 as-value]
        :local avgRtt (\$result->\"avg-rtt\")
        
        :if ([:len \$avgRtt] > 0) do={
            :local avgMs (\$avgRtt / 1000)  # microsec to ms
            :put (\$host . \": \" . \$avgMs . \"ms\")
            
            :if (\$avgMs > \$threshold) do={
                :log warning (\"High latency to \" . \$host . \": \" . \$avgMs . \"ms\")
            }
        } else={
            :put (\$host . \": unreachable\")
            :log error (\$host . \" unreachable\")
        }
    }
" comment="Latency monitoring"

/system scheduler add \
    name="latency-check" \
    interval=5m \
    on-event="/system script run latency-check"
```

---

## 40.5 Multiple Hosts Management

```routeros
# Bulk netwatch management
/system script add name="setup-netwatch" source="
    :local monitorHosts {
        {\"gw-primary\"; \"192.168.1.1\"; 10};
        {\"gw-backup\"; \"192.168.2.1\"; 10};
        {\"isp1-dns\"; \"8.8.8.8\"; 30};
        {\"isp2-dns\"; \"1.1.1.1\"; 30};
        {\"server-web\"; \"10.0.0.10\"; 60};
        {\"server-db\"; \"10.0.0.20\"; 60}
    }
    
    :foreach h in=\$monitorHosts do={
        :local name (\$h->0)
        :local ip (\$h->1)
        :local interval (\$h->2)
        
        # ลบ existing
        /tool netwatch remove [find host=\$ip]
        
        # เพิ่มใหม่
        /tool netwatch add \
            host=\$ip \
            interval=(\$interval . \"s\") \
            up-script=(\":log info \\\"\" . \$name . \" (\" . \$ip . \") UP\\\"\") \
            down-script=(\"/tool e-mail send to=\\\"admin@company.com\\\" subject=\\\"DOWN: \" . \$name . \"\\\" body=\\\"Host \" . \$ip . \" is down\\\"\")
        
        :put (\"Monitoring: \" . \$name . \" (\" . \$ip . \") every \" . \$interval . \"s\")
    }
" comment="Setup all netwatch monitors"

/system script run setup-netwatch
```

---

## 40.6 Failover Triggers

```routeros
# Dual WAN failover with Netwatch

# Primary WAN: ether1 (ISP1)
# Backup WAN: ether2 (ISP2)

# Global state
:global primaryWanUp true

# Monitor primary WAN (via ISP1 DNS)
/tool netwatch add \
    host=8.8.8.8 \
    interval=15s \
    timeout=3s \
    up-script="
        :global primaryWanUp
        :if (!(\$primaryWanUp)) do={
            :set primaryWanUp true
            
            # กลับมาใช้ primary route
            /ip route set [find dst-address=0.0.0.0/0 gateway=10.0.0.1] disabled=no
            /ip route set [find dst-address=0.0.0.0/0 gateway=10.0.1.1] disabled=yes
            
            :log info \"Failback: Primary WAN restored\"
            /tool e-mail send to=\"admin@company.com\" \
                subject=\"WAN Restored\" \
                body=\"Primary WAN connection restored. Switched back from backup.\"
        }
    " \
    down-script="
        :global primaryWanUp
        :if (\$primaryWanUp) do={
            :set primaryWanUp false
            
            # Switch to backup route
            /ip route set [find dst-address=0.0.0.0/0 gateway=10.0.0.1] disabled=yes
            /ip route set [find dst-address=0.0.0.0/0 gateway=10.0.1.1] disabled=no
            
            :log error \"Failover: Switched to backup WAN\"
            /tool e-mail send to=\"admin@company.com\" \
                subject=\"WAN FAILOVER\" \
                body=\"Primary WAN DOWN. Switched to backup ISP.\"
        }
    "

# ดู netwatch
/tool netwatch print
```

---

## 40.7 Complex Monitoring Rules

```routeros
# Monitor และ alert แบบ graduated
/system script add name="graduated-alert" source="
    :global alertSentLevel
    :local downHosts 0
    
    :foreach entry in=[/tool netwatch find] do={
        :local status [/tool netwatch get \$entry status]
        :if (\$status = \"down\") do={
            :set downHosts (\$downHosts + 1)
        }
    }
    
    :if (\$downHosts = 0) do={
        :set alertSentLevel 0
    }
    :if (\$downHosts >= 1 && \$alertSentLevel < 1) do={
        :set alertSentLevel 1
        /tool e-mail send to=\"admin@company.com\" \
            subject=(\"Warning: \" . \$downHosts . \" host(s) down\") \
            body=(\"\" . \$downHosts . \" monitored host(s) are not responding\")
    }
    :if (\$downHosts >= 3 && \$alertSentLevel < 2) do={
        :set alertSentLevel 2
        /tool e-mail send to=\"oncall@company.com\" \
            subject=(\"CRITICAL: \" . \$downHosts . \" hosts down\") \
            body=\"Multiple critical hosts are down - immediate attention required\"
    }
" comment="Graduated alert based on down host count"

/system scheduler add \
    name="graduated-alert-check" \
    interval=2m \
    on-event="/system script run graduated-alert"
```

---

## Lab 40: HA Failover with Netwatch

### Solution

```routeros
# Complete HA Failover with Netwatch
# =====================================
# Topology:
# ether1 = WAN1 (Primary ISP, gateway 10.1.1.1)
# ether2 = WAN2 (Backup ISP, gateway 10.2.1.1)
# ether3 = LAN

# 1. Routes (Primary active, Backup disabled)
/ip route add dst-address=0.0.0.0/0 gateway=10.1.1.1 distance=1 comment="Primary WAN"
/ip route add dst-address=0.0.0.0/0 gateway=10.2.1.1 distance=10 disabled=yes comment="Backup WAN"

# 2. Global state
:global wan1Up true
:global wan2Up true

# 3. Netwatch สำหรับ WAN1
/tool netwatch add host=8.8.8.8 interval=10s timeout=3s \
    down-script="
        :global wan1Up
        :set wan1Up false
        /ip route set [find comment=\"Primary WAN\"] disabled=yes
        /ip route set [find comment=\"Backup WAN\"] disabled=no
        :log error \"WAN1 DOWN - Failover to WAN2\"
        /tool e-mail send to=\"admin@company.com\" subject=\"FAILOVER: WAN1 DOWN\" \
            body=\"Primary WAN down, switched to backup at \" . [/system clock get time]
    " \
    up-script="
        :global wan1Up
        :if (!(\$wan1Up)) do={
            :set wan1Up true
            /ip route set [find comment=\"Backup WAN\"] disabled=yes
            /ip route set [find comment=\"Primary WAN\"] disabled=no
            :log info \"WAN1 UP - Failback to WAN1\"
            /tool e-mail send to=\"admin@company.com\" subject=\"FAILBACK: WAN1 RESTORED\" \
                body=\"Primary WAN restored at \" . [/system clock get time]
        }
    "

# 4. Netwatch สำหรับ WAN2
/tool netwatch add host=1.1.1.1 interval=10s timeout=3s \
    down-script="
        :global wan2Up
        :set wan2Up false
        :log warning \"WAN2 DOWN - backup unavailable\"
    " \
    up-script="
        :global wan2Up
        :set wan2Up true
        :log info \"WAN2 UP\"
    "

# 5. ดู status
/tool netwatch print
/ip route print

:put "HA Failover configured:"
:put "- Primary: 10.1.1.1 (monitored via 8.8.8.8)"
:put "- Backup: 10.2.1.1 (monitored via 1.1.1.1)"
:put "- Failover: automatic on 3s timeout"
:put "- Email alerts: enabled"
```

---

## Summary ของ Part 40

| Feature | Description |
|---------|------------|
| interval | ความถี่ในการ ping |
| timeout | เวลา timeout ก่อนถือว่า down |
| up-script | Script เมื่อ host กลับ UP |
| down-script | Script เมื่อ host DOWN |

> **Tip:** ใช้ Netwatch interval สั้นสำหรับ critical hosts เช่น gateway (10s)

> **Warning:** down-script ทำงานเมื่อ state เปลี่ยนเท่านั้น ไม่ใช่ทุก interval

> **Best Practice:** ทดสอบ failover script ด้วยการ disable interface ก่อน production

---

## สรุปภาพรวม Part 021-040

ใน section นี้ได้เรียนรู้:
- **021-027**: String, Array, File, Scheduler, Global Variables, Error Handling, Parameters
- **028-030**: Interface, DHCP, Firewall automation
- **031-034**: Email, Hotspot, PPPoE, VPN
- **035-037**: Bandwidth, User Manager, Logging
- **038-040**: Network Monitoring, SNMP, Netwatch

---

[← Part 39: SNMP Configuration](part-039-snmp-configuration.md) | [Part 41: Advanced Scripting →](part-041-advanced-scripting.md)
