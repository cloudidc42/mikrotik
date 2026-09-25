# Part 38: Network Monitoring ใน RouterOS

## บทนำ

Network monitoring เป็นส่วนสำคัญในการดูแลระบบ RouterOS มีเครื่องมือหลายอย่างสำหรับ monitor network ทั้ง built-in และ script-based

---

## 38.1 Ping Monitoring

```routeros
# Ping พื้นฐาน
/tool ping 8.8.8.8 count=4

# Ping พร้อม options
/tool ping address=8.8.8.8 count=10 size=1472 interval=1s

# Ping test script
:local pingTest do={
    :local host $1
    :local count $2
    :local result "PASS"
    :local failed 0
    
    :for i from=1 to=$count do={
        :local response [/tool ping address=$host count=1 as-value]
        :if (($response->"sent" != $response->"received")) do={
            :set failed ($failed + 1)
        }
    }
    
    :if ($failed > 0) do={
        :set result "FAIL ($failed/$count packet loss)"
    }
    
    :put "$host: $result"
    :return $result
}

:local r1 [$pingTest "8.8.8.8" 5]
:local r2 [$pingTest "192.168.1.1" 5]

# Continuous monitoring with alert
/system script add name="ping-monitor" source="
    :local hosts {\"8.8.8.8\"; \"8.8.4.4\"; \"1.1.1.1\"}
    :local threshold 3
    
    :foreach host in=\$hosts do={
        :local result [/tool ping address=\$host count=\$threshold as-value]
        :local sent (\$result->\"sent\")
        :local received (\$result->\"received\")
        :local loss (\$sent - \$received)
        
        :if (\$loss >= \$threshold) do={
            :log warning (\"Host unreachable: \" . \$host . \" (\" . \$loss . \"/\" . \$sent . \" packets lost)\")
            /tool e-mail send to=\"admin@company.com\" \
                subject=(\"Network Alert: \" . \$host . \" unreachable\") \
                body=(\"Host \" . \$host . \" is not responding. \" . \$loss . \" of \" . \$sent . \" packets lost.\")
        } else={
            :log info (\"Host OK: \" . \$host . \" (\" . \$received . \"/\" . \$sent . \" received)\")
        }
    }
" comment="Ping monitoring script"

/system scheduler add \
    name="ping-check" \
    interval=5m \
    on-event="/system script run ping-monitor"
```

---

## 38.2 Bandwidth Monitoring

```routeros
# Real-time bandwidth stats per interface
:local showBandwidth do={
    :put "=== Interface Bandwidth ==="
    :foreach iface in=[/interface find disabled=no] do={
        :local name [/interface get $iface name]
        :local rxRate [/interface get $iface rx-bits-per-second]
        :local txRate [/interface get $iface tx-bits-per-second]
        :local rxMbps ($rxRate / 1000000)
        :local txMbps ($txRate / 1000000)
        :put "$name: RX=$rxMbps Mbps / TX=$txMbps Mbps"
    }
}

[$showBandwidth]

# Monitor bandwidth threshold
/system script add name="bw-threshold-monitor" source="
    :local wanIface \"ether1\"
    :local threshold 90000000  # 90Mbps
    :local id [/interface find name=\$wanIface]
    
    :if ([:len \$id] > 0) do={
        :local rxRate [/interface get \$id rx-bits-per-second]
        :local txRate [/interface get \$id tx-bits-per-second]
        
        :if (\$rxRate > \$threshold || \$txRate > \$threshold) do={
            :local rxMbps (\$rxRate / 1000000)
            :local txMbps (\$txRate / 1000000)
            :log warning (\"High bandwidth: \" . \$wanIface . \" RX=\" . \$rxMbps . \"Mbps TX=\" . \$txMbps . \"Mbps\")
            /tool e-mail send to=\"admin@company.com\" \
                subject=\"Bandwidth Alert: High usage on \" . \$wanIface \
                body=(\"Interface \" . \$wanIface . \" is near capacity:\\r\\nRX: \" . \$rxMbps . \" Mbps\\r\\nTX: \" . \$txMbps . \" Mbps\")
        }
    }
" comment="Bandwidth threshold monitor"
```

---

## 38.3 Interface Utilization

```routeros
# Interface utilization monitoring
/system script add name="interface-stats" source="
    :local stats \"=== Interface Utilization ===\\r\\n\"
    :local wanIface \"ether1\"
    :local wanSpeed 100  # 100Mbps
    
    :foreach iface in=[/interface find disabled=no type=ether] do={
        :local name [/interface get \$iface name]
        :local rxRate [/interface get \$iface rx-bits-per-second]
        :local txRate [/interface get \$iface tx-bits-per-second]
        :local rxMbps (\$rxRate / 1000000)
        :local txMbps (\$txRate / 1000000)
        :local rxPct (\$rxMbps * 100 / \$wanSpeed)
        :local txPct (\$txMbps * 100 / \$wanSpeed)
        
        :set stats (\$stats . \$name . \": RX=\" . \$rxMbps . \"Mbps (\" . \$rxPct . \"%) TX=\" . \$txMbps . \"Mbps (\" . \$txPct . \"%)\\r\\n\")
    }
    
    :put \$stats
" comment="Interface utilization stats"

# Track interface errors
/system script add name="interface-errors" source="
    :local errorThreshold 100
    :foreach iface in=[/interface find disabled=no type=ether] do={
        :local name [/interface get \$iface name]
        :local rxErrors [/interface get \$iface rx-error]
        :local txErrors [/interface get \$iface tx-error]
        
        :if (\$rxErrors > \$errorThreshold || \$txErrors > \$errorThreshold) do={
            :log warning (\"Interface errors: \" . \$name . \" RX-err=\" . \$rxErrors . \" TX-err=\" . \$txErrors)
        }
    }
" comment="Interface error monitor"
```

---

## 38.4 Netwatch

```routeros
# Netwatch - built-in up/down monitoring
/tool netwatch add \
    host=8.8.8.8 \
    interval=30s \
    timeout=1s \
    up-script="
        :log info \"8.8.8.8 is UP\"
    " \
    down-script="
        :log warning \"8.8.8.8 is DOWN\"
        /tool e-mail send to=\"admin@company.com\" subject=\"Alert: 8.8.8.8 DOWN\" body=\"Google DNS is unreachable\"
    "

/tool netwatch add \
    host=192.168.1.1 \
    interval=10s \
    up-script=":log info \"Gateway UP\"" \
    down-script=":log error \"Gateway DOWN - network issue!\""

# ดู netwatch status
/tool netwatch print

# Netwatch สำหรับ multiple hosts
:local hosts {"8.8.8.8"; "1.1.1.1"; "9.9.9.9"}
:foreach host in=$hosts do={
    :local existing [/tool netwatch find host=$host]
    :if ([:len $existing] = 0) do={
        /tool netwatch add \
            host=$host \
            interval=30s \
            up-script=(":log info \"" . $host . " UP\"") \
            down-script=(":log warning \"" . $host . " DOWN\"")
        :put "Added netwatch for: $host"
    }
}
```

---

## 38.5 SNMP Polling (Local)

```routeros
# Enable SNMP
/snmp set enabled=yes

# Community configuration
/snmp community add name=monitoring address=10.0.0.0/8 security=none read-access=yes

# Check SNMP is working
/snmp print

# Script query SNMP OIDs (via external tool or script)
# RouterOS ไม่มี built-in SNMP client แต่ใช้ /tool fetch ส่ง HTTP ไปยัง SNMP proxy ได้

# ดู system uptime via SNMP (ต้องใช้ external client)
# OID: .1.3.6.1.2.1.1.3.0 = sysUpTime
```

---

## 38.6 Connection Monitoring

```routeros
# Monitor active connections
:local monitorConnections do={
    :local total [:len [/ip firewall connection find]]
    :local established [:len [/ip firewall connection find tcp-state=established]]
    :local syn [:len [/ip firewall connection find tcp-state=syn-sent]]
    
    :put "=== Connection Stats ==="
    :put "Total: $total"
    :put "Established: $established"
    :put "SYN: $syn"
    
    # Alert if too many connections
    :if ($total > 10000) do={
        :log warning "High connection count: $total"
    }
}

[$monitorConnections]

# Track PPPoE connections
:local monitorPPPoE do={
    :local active [:len [/ppp active find]]
    :put "PPPoE Active: $active"
    :log info "PPPoE sessions: $active"
}

[$monitorPPPoE]
```

---

## 38.7 Dashboard Data Collection

```routeros
# Collect system metrics สำหรับ dashboard
/system script add name="collect-metrics" source="
    :local cpuLoad [/system resource get cpu-load]
    :local freeMemory [/system resource get free-memory]
    :local totalMemory [/system resource get total-memory]
    :local memUsedPct (100 - (\$freeMemory * 100 / \$totalMemory))
    :local uptime [/system resource get uptime]
    :local tempC [/system health get temperature]
    
    :local metrics \"CPU:\" . \$cpuLoad . \"% MEM:\" . \$memUsedPct . \"% UPTIME:\" . \$uptime
    
    :if ([:len \$tempC] > 0) do={
        :set metrics (\$metrics . \" TEMP:\" . \$tempC . \"C\")
    }
    
    :log info \$metrics
    :put \$metrics
" comment="Collect system metrics"

/system scheduler add \
    name="collect-metrics" \
    interval=1m \
    on-event="/system script run collect-metrics"
```

---

## Lab 38: Complete Monitoring Setup

### Solution

```routeros
# Complete Network Monitoring Setup
# ===================================

# 1. Netwatch สำหรับ critical hosts
/tool netwatch add host=8.8.8.8 interval=30s \
    down-script="/tool e-mail send to=\"admin@company.com\" subject=\"DNS DOWN\" body=\"8.8.8.8 is unreachable\""

/tool netwatch add host=[/ip route get [find dst-address=0.0.0.0/0] gateway] interval=10s \
    down-script=":log error \"Default gateway DOWN\""

# 2. Bandwidth monitor scheduler
/system scheduler add name="bw-monitor" interval=5m \
    on-event="/system script run bw-threshold-monitor"

# 3. Interface error monitor
/system scheduler add name="iface-error-check" interval=10m \
    on-event="/system script run interface-errors"

# 4. Daily summary report
/system script add name="daily-monitoring-report" source="
    :local report \"=== Daily Network Report ===\\r\\n\"
    :set report (\$report . \"Date: \" . [/system clock get date] . \"\\r\\n\")
    :set report (\$report . \"CPU: \" . [/system resource get cpu-load] . \"%\\r\\n\")
    :set report (\$report . \"Memory: \" . ([/system resource get free-memory] / 1048576) . \"MB free\\r\\n\")
    :set report (\$report . \"Uptime: \" . [/system resource get uptime] . \"\\r\\n\")
    :set report (\$report . \"PPPoE sessions: \" . [:len [/ppp active find]] . \"\\r\\n\")
    :set report (\$report . \"Hotspot sessions: \" . [:len [/ip hotspot active find]] . \"\\r\\n\")
    
    /tool e-mail send to=\"admin@company.com\" \
        subject=(\"Daily Network Report - \" . [/system clock get date]) \
        body=\$report
"

/system scheduler add name="daily-report" start-time=08:00:00 interval=1d \
    on-event="/system script run daily-monitoring-report"

:put "Monitoring setup complete"
:put "- Netwatch: 8.8.8.8 + gateway"
:put "- Bandwidth check: every 5 min"
:put "- Error check: every 10 min"
:put "- Daily report: 8:00 AM"
```

---

## Summary ของ Part 38

| Tool | ใช้สำหรับ |
|------|---------|
| /tool ping | Manual connectivity test |
| /tool netwatch | Automated up/down monitoring |
| /interface stats | Traffic monitoring |
| /system resource | CPU/Memory monitoring |
| Scheduler | Automated checks |

> **Tip:** Combine Netwatch กับ failover script เพื่อ automatic failover

> **Note:** RouterOS monitoring เหมาะกับ network ขนาดเล็ก-กลาง สำหรับใหญ่ใช้ Zabbix/Nagios

---

[← Part 37: Log Monitoring](part-037-log-monitoring.md) | [Part 39: SNMP Configuration →](part-039-snmp-configuration.md)
