# Part 33: PPPoE Server ใน RouterOS

## บทนำ

PPPoE (Point-to-Point Protocol over Ethernet) ใช้กันแพร่หลายใน ISP networks สำหรับ broadband connections RouterOS สามารถ setup เป็น PPPoE Server ได้สมบูรณ์แบบ รองรับ authentication, bandwidth management, และ accounting

---

## 33.1 PPPoE Server Setup

### ขั้นตอน Setup PPPoE Server

```routeros
# Step 1: สร้าง IP Pool สำหรับ PPPoE clients
/ip pool add name=pppoe-pool ranges=10.0.0.2-10.0.0.254

# Step 2: สร้าง PPPoE Profile
/ppp profile add \
    name=pppoe-default \
    local-address=10.0.0.1 \
    remote-address=pppoe-pool \
    use-compression=no \
    use-encryption=no \
    only-one=default \
    session-timeout=0s \
    idle-timeout=0s

# Step 3: สร้าง PPPoE Server
/interface pppoe-server server add \
    service-name=MYISP \
    interface=ether1 \
    default-profile=pppoe-default \
    authentication=pap,chap,mschap1,mschap2 \
    max-sessions=200 \
    disabled=no

# ตรวจสอบ server
/interface pppoe-server server print

# ดู active sessions
/ppp active print
```

### Multiple PPPoE Servers บน interfaces ต่างๆ

```routeros
# PPPoE บน VLAN interfaces
/interface vlan add name=vlan100 vlan-id=100 interface=ether1 comment="ISP VLAN"

/interface pppoe-server server add \
    service-name=ISP-VLAN100 \
    interface=vlan100 \
    default-profile=pppoe-default \
    disabled=no

# ตรวจสอบ servers ทั้งหมด
/interface pppoe-server server print columns=service-name,interface,disabled,sessions
```

---

## 33.2 PPPoE Profiles

```routeros
# Profile สำหรับ tiers ต่างๆ
# Basic - 10Mbps
/ppp profile add \
    name=basic-10mbps \
    local-address=10.0.0.1 \
    remote-address=pppoe-pool \
    rate-limit=5M/10M \
    session-timeout=0s \
    comment="Basic 10Mbps package"

# Standard - 30Mbps
/ppp profile add \
    name=standard-30mbps \
    local-address=10.0.0.1 \
    remote-address=pppoe-pool \
    rate-limit=15M/30M \
    session-timeout=0s \
    comment="Standard 30Mbps package"

# Premium - 100Mbps
/ppp profile add \
    name=premium-100mbps \
    local-address=10.0.0.1 \
    remote-address=pppoe-pool \
    rate-limit=50M/100M \
    session-timeout=0s \
    comment="Premium 100Mbps package"

# Script สร้าง profiles จาก list
:local createPPPoEProfiles do={
    :local profiles $1
    :local localIP $2
    :local pool $3
    
    :foreach profile in=$profiles do={
        :local name ($profile->"name")
        :local dl ($profile->"download")
        :local ul ($profile->"upload")
        
        /ppp profile add \
            name=$name \
            local-address=$localIP \
            remote-address=$pool \
            rate-limit=($ul . "/" . $dl)
        
        :put "Created profile: $name ($dl/$ul)"
    }
}

:local packageList {
    {"name"="pkg-10mbps"; "download"="10M"; "upload"="5M"};
    {"name"="pkg-30mbps"; "download"="30M"; "upload"="15M"};
    {"name"="pkg-100mbps"; "download"="100M"; "upload"="50M"};
    {"name"="pkg-unlimited"; "download"="1G"; "upload"="500M"}
}

[$createPPPoEProfiles $packageList "10.0.0.1" "pppoe-pool"]
```

---

## 33.3 PPPoE Client Management

```routeros
# เพิ่ม PPPoE users
/ppp secret add \
    name=user001 \
    password=Pass@001 \
    service=pppoe \
    profile=basic-10mbps \
    comment="Client 001 - Basic"

/ppp secret add \
    name=user002 \
    password=Pass@002 \
    service=pppoe \
    profile=premium-100mbps \
    comment="Client 002 - Premium"

# เพิ่ม user พร้อม static IP
/ppp secret add \
    name=user003 \
    password=Pass@003 \
    service=pppoe \
    profile=standard-30mbps \
    remote-address=10.0.1.1 \
    comment="Client 003 - Static IP"

# Script bulk add users
:local bulkAddPPPoE do={
    :local users $1
    :local defaultProfile $2
    
    :foreach user in=$users do={
        :local name ($user->"name")
        :local pass ($user->"password")
        :local profile ($user->"profile")
        
        :if ([:len $profile] = 0) do={ :set profile $defaultProfile }
        
        :do {
            /ppp secret add \
                name=$name \
                password=$pass \
                service=pppoe \
                profile=$profile \
                comment=("Added: " . [/system clock get date])
            :put "Added PPPoE user: $name ($profile)"
        } on-error={
            :put "Failed to add: $name (may already exist)"
        }
    }
}

:local newUsers {
    {"name"="cust001"; "password"="Cust@001"; "profile"="basic-10mbps"};
    {"name"="cust002"; "password"="Cust@002"; "profile"="standard-30mbps"};
    {"name"="cust003"; "password"="Cust@003"; "profile"="premium-100mbps"}
}

[$bulkAddPPPoE $newUsers "basic-10mbps"]

# ดู users ทั้งหมด
/ppp secret print columns=name,service,profile,comment

# ค้นหา user
/ppp secret print where name~"cust"

# แก้ไข profile ของ user
/ppp secret set [find name=cust001] profile=standard-30mbps
:put "Upgraded cust001 to standard-30mbps"
```

---

## 33.4 Dynamic IP Assignment

```routeros
# IP Pool Management
/ip pool add name=pool-basic ranges=10.1.0.1-10.1.0.254
/ip pool add name=pool-standard ranges=10.2.0.1-10.2.0.254
/ip pool add name=pool-premium ranges=10.3.0.1-10.3.0.254

# Profiles ใช้ pool ต่างกันตาม tier
/ppp profile set basic-10mbps remote-address=pool-basic
/ppp profile set standard-30mbps remote-address=pool-standard
/ppp profile set premium-100mbps remote-address=pool-premium

# Script ตรวจสอบ pool usage
:local checkPoolUsage do={
    :local pools {"pool-basic"; "pool-standard"; "pool-premium"}
    :put "=== IP Pool Usage ==="
    
    :foreach poolName in=$pools do={
        :local pool [/ip pool find name=$poolName]
        :if ([:len $pool] > 0) do={
            :local ranges [/ip pool get $pool ranges]
            :local used [:len [/ip pool used print count-only]]
            :put "$poolName: ranges=$ranges"
        }
    }
}

[$checkPoolUsage]

# Auto-extend pool ถ้า running low
:local autoExtendPool do={
    :local poolName $1
    :local threshold 10  # Alert when < 10 IPs left
    
    :local pool [/ip pool find name=$poolName]
    :if ([:len $pool] = 0) do={ :return }
    
    # นับ IPs ที่ใช้อยู่
    :local usedCount [:len [/ppp active find]]
    :put "Pool $poolName: $usedCount sessions active"
}
```

---

## 33.5 Bandwidth per User

```routeros
# Rate limiting ผ่าน PPP profile
/ppp profile set basic-10mbps rate-limit=5M/10M

# Per-user rate limiting
/ppp secret set cust001 rate-limit=3M/6M  # Override profile

# Dynamic rate limiting ผ่าน script
:local setUserBandwidth do={
    :local username $1
    :local upload $2
    :local download $3
    
    :local userId [/ppp secret find name=$username]
    :if ([:len $userId] = 0) do={
        :put "User not found: $username"
        :return false
    }
    
    :if ([:len $upload] > 0 && [:len $download] > 0) do={
        /ppp secret set $userId rate-limit=($upload . "/" . $download)
        :put "Bandwidth set for $username: $upload up / $download down"
    } else={
        # ลบ rate-limit (ใช้จาก profile)
        /ppp secret set $userId rate-limit=""
        :put "Using profile rate-limit for $username"
    }
    :return true
}

[$setUserBandwidth "cust001" "5M" "15M"]
[$setUserBandwidth "cust002" "" ""]  # Reset to profile default

# FUP (Fair Usage Policy)
# ลด bandwidth หลัง ใช้ data เกิน limit
/system script add name="fup-check" source="
    :foreach session in=[/ppp active find] do={
        :local user [/ppp active get \$session name]
        :local bytesIn [/ppp active get \$session bytes-in]
        :local limitGB 50  # 50GB limit
        :local limitBytes (\$limitGB * 1073741824)
        
        :if (\$bytesIn > \$limitBytes) do={
            # Exceeded FUP limit - reduce speed
            :local userId [/ppp secret find name=\$user]
            :if ([:len \$userId] > 0) do={
                /ppp secret set \$userId rate-limit=\"1M/2M\"  # FUP speed
                :log info (\"FUP applied to \" . \$user . \": \" . \$bytesIn . \" bytes used\")
            }
        }
    }
" comment="FUP check and throttle"
```

---

## 33.6 User Authentication

```routeros
# Authentication methods
/interface pppoe-server server set [find] \
    authentication=mschap2,mschap1,chap,pap

# ตรวจสอบ authentication ที่ใช้
/interface pppoe-server server print detail

# Challenge ด้วย CHAP
# RouterOS ใช้ MD5 hash สำหรับ CHAP authentication

# ตรวจสอบ user credentials
:local checkCredentials do={
    :local username $1
    :local password $2
    
    :local userId [/ppp secret find name=$username]
    :if ([:len $userId] = 0) do={
        :put "User not found: $username"
        :return false
    }
    
    :local storedPass [/ppp secret get $userId password]
    :if ($storedPass = $password) do={
        :put "Authentication: SUCCESS for $username"
        :return true
    } else={
        :put "Authentication: FAILED for $username"
        :return false
    }
}

# Change password
:local changePassword do={
    :local username $1
    :local newPassword $2
    
    :local userId [/ppp secret find name=$username]
    :if ([:len $userId] = 0) do={
        :put "User not found"
        :return false
    }
    
    /ppp secret set $userId password=$newPassword
    :put "Password changed for: $username"
    :return true
}

# Force re-authentication (disconnect and reconnect)
:local forceReauth do={
    :local username $1
    :local session [/ppp active find name=$username]
    :if ([:len $session] > 0) do={
        /ppp active remove $session
        :put "Disconnected $username (will reconnect automatically)"
    }
}
```

---

## 33.7 PPPoE Scripts

```routeros
# on-up script - เมื่อ session เชื่อมต่อ
/ppp profile set default on-up="
    :log info (\"PPPoE connected: \" . \$user . \" from \" . \$\"caller-id\" . \" IP: \" . \$\"remote-address\")
    
    # เพิ่ม route สำหรับ client (ถ้าจำเป็น)
    # /ip route add dst-address=(\$\"remote-address\"/32) gateway=\$\"local-address\"
    
    # ส่ง notification
    :do {
        /tool fetch url=(\"http://billing.isp.com/api/connected?user=\" . \$user . \"&ip=\" . \$\"remote-address\") \
            dst-path=\"pppoe-notify.tmp\"
    } on-error={}
"

# on-down script - เมื่อ session ตัดการเชื่อมต่อ
/ppp profile set default on-down="
    :local user \$user
    :local uptime \$\"session-time\"
    :local bytesIn \$\"bytes-in\"
    :local bytesOut \$\"bytes-out\"
    
    :log info (\"PPPoE disconnected: \" . \$user . \" uptime=\" . \$uptime . \" rx=\" . \$bytesIn . \" tx=\" . \$bytesOut)
    
    # บันทึก usage
    :local logEntry (\$user . \",\" . \$uptime . \",\" . \$bytesIn . \",\" . \$bytesOut . \"\\r\\n\")
    :local fileId [/file find name=\"pppoe-usage.csv\"]
    :local existing \"\"
    :if ([:len \$fileId] > 0) do={
        :set existing [/file get \$fileId contents]
    }
    /file remove [find name=\"pppoe-usage.csv\"]
    /tool fetch url=\"data:,\" dst-path=\"pppoe-usage.csv\"
    /file set [find name=\"pppoe-usage.csv\"] contents=(\$existing . \$logEntry)
"
```

---

## 33.8 RADIUS Integration

```routeros
# ตั้งค่า RADIUS สำหรับ PPPoE
/radius add \
    service=ppp \
    address=192.168.1.10 \
    src-address=192.168.1.1 \
    secret=pppoe-radius-secret \
    authentication-port=1812 \
    accounting-port=1813 \
    timeout=3000ms \
    disabled=no

# เปิดใช้ RADIUS ใน PPP
/ppp aaa set \
    use-radius=yes \
    accounting=yes \
    interim-update=30m

# ตรวจสอบ RADIUS
/radius print
/ppp aaa print

# Fallback to local authentication
/ppp aaa set \
    use-radius=yes \
    accounting=yes \
    radius-authentication=yes \
    use-circuit-id-in-nas-port-id=yes

# Script ตรวจสอบ RADIUS health
/system script add name="check-radius-health" source="
    :do {
        /radius test service=ppp user=healthcheck password=healthcheck
        :log info \"RADIUS: OK\"
    } on-error={
        :log error \"RADIUS: UNREACHABLE - switching to local auth\"
        /ppp aaa set use-radius=no
        
        # Alert admin
        /tool e-mail send to=\"admin@isp.com\" \
            subject=\"RADIUS Down!\" \
            body=\"RADIUS server unreachable. Switched to local authentication.\"
    }
" comment="RADIUS health check"

/system scheduler add \
    name="radius-health-check" \
    interval=5m \
    on-event="/system script run check-radius-health"
```

---

## 33.9 Monitoring PPPoE Sessions

```routeros
# ดู active sessions
/ppp active print

# ดูรายละเอียด sessions
/ppp active print detail

# ดู sessions ของ specific user
/ppp active print where name=cust001

# Disconnect specific user
/ppp active remove [find name=cust001]

# Script monitor PPPoE sessions
/system script add name="pppoe-monitor" source="
    :local activeSessions [:len [/ppp active find]]
    :local maxSessions 200
    
    :log info (\"PPPoE active sessions: \" . \$activeSessions . \"/\" . \$maxSessions)
    
    :if (\$activeSessions > (\$maxSessions * 90 / 100)) do={
        :log warning (\"PPPoE sessions near limit: \" . \$activeSessions . \"/\" . \$maxSessions)
        /tool e-mail send to=\"admin@isp.com\" \
            subject=\"PPPoE Session Warning\" \
            body=(\"Active sessions: \" . \$activeSessions . \" of \" . \$maxSessions)
    }
" comment="PPPoE session monitoring"

# PPPoE Statistics Report
:local pppoeReport do={
    :local activeSessions [/ppp active find]
    :local report "=== PPPoE Session Report ===\r\n"
    :set report ($report . "Time: " . [/system clock get time] . "\r\n")
    :set report ($report . "Active: " . [:len $activeSessions] . "\r\n\r\n")
    
    :foreach session in=$activeSessions do={
        :local user [/ppp active get $session name]
        :local ip [/ppp active get $session address]
        :local uptime [/ppp active get $session uptime]
        :local bytesIn [/ppp active get $session bytes-in]
        :local bytesOut [/ppp active get $session bytes-out]
        
        :set report ($report . $user . " | " . $ip . " | " . $uptime . \
                    " | RX:" . $bytesIn . " TX:" . $bytesOut . "\r\n")
    }
    
    :return $report
}

:put [$pppoeReport]

/system scheduler add \
    name="pppoe-monitor" \
    interval=5m \
    on-event="/system script run pppoe-monitor"
```

---

## Lab 33: ISP-like PPPoE Server

### Solution

```routeros
# ISP-like PPPoE Server Lab
# ==========================

# --- Infrastructure Setup ---

# WAN interface (กับ upstream)
/ip address add address=203.0.113.1/30 interface=ether1 comment="WAN"

# PPPoE server interface
/ip address add address=10.0.0.1/8 interface=ether2 comment="PPPoE server"

# IP Pools for different tiers
/ip pool add name=pool-10m ranges=10.1.0.1-10.1.0.254
/ip pool add name=pool-30m ranges=10.2.0.1-10.2.0.254
/ip pool add name=pool-100m ranges=10.3.0.1-10.3.0.254

# PPPoE Profiles
/ppp profile add name=pkg-10mbps local-address=10.0.0.1 remote-address=pool-10m rate-limit=5M/10M
/ppp profile add name=pkg-30mbps local-address=10.0.0.1 remote-address=pool-30m rate-limit=15M/30M
/ppp profile add name=pkg-100mbps local-address=10.0.0.1 remote-address=pool-100m rate-limit=50M/100M

# PPPoE Server
/interface pppoe-server server add \
    service-name=MyISP \
    interface=ether2 \
    default-profile=pkg-10mbps \
    authentication=pap,chap,mschap2 \
    max-sessions=500 \
    disabled=no

# --- Sample Users ---
/ppp secret add name=cust-001 password=P@ss001 service=pppoe profile=pkg-10mbps
/ppp secret add name=cust-002 password=P@ss002 service=pppoe profile=pkg-30mbps
/ppp secret add name=cust-003 password=P@ss003 service=pppoe profile=pkg-100mbps

# --- Firewall for PPPoE ---
/ip firewall nat add chain=srcnat \
    out-interface=ether1 \
    action=masquerade \
    comment="NAT for PPPoE clients"

# Allow PPPoE traffic
/ip firewall filter add chain=forward \
    in-interface=ether2 \
    action=accept \
    comment="Allow PPPoE clients"

# --- Monitoring & Reports ---
/system script add name="isp-daily-report" source="
    :local report \"ISP Daily Operations Report\\r\\n\"
    :set report (\$report . \"Date: \" . [/system clock get date] . \"\\r\\n\\r\\n\")
    
    :local activeSessions [:len [/ppp active find]]
    :local totalUsers [:len [/ppp secret find service=pppoe]]
    
    :set report (\$report . \"User Statistics:\\r\\n\")
    :set report (\$report . \"  Total users: \" . \$totalUsers . \"\\r\\n\")
    :set report (\$report . \"  Active sessions: \" . \$activeSessions . \"\\r\\n\\r\\n\")
    
    :set report (\$report . \"Package Distribution:\\r\\n\")
    :set report (\$report . \"  10Mbps: \" . [:len [/ppp secret find profile=pkg-10mbps]] . \"\\r\\n\")
    :set report (\$report . \"  30Mbps: \" . [:len [/ppp secret find profile=pkg-30mbps]] . \"\\r\\n\")
    :set report (\$report . \"  100Mbps: \" . [:len [/ppp secret find profile=pkg-100mbps]] . \"\\r\\n\")
    
    /tool e-mail send to=\"noc@myisp.com\" \
        subject=\"Daily Operations Report\" \
        body=\$report
" comment="ISP daily operations report"

/system scheduler add \
    name="isp-daily-report" \
    start-time=07:00:00 \
    interval=1d \
    on-event="/system script run isp-daily-report"

:put "ISP PPPoE Server deployed!"
:put "Server: ether2, Service: MyISP"
:put "Packages: 10M, 30M, 100Mbps"
:put "Sample users: cust-001, cust-002, cust-003"
```

---

## Summary ของ Part 33

| Command | ใช้เมื่อ |
|---------|---------|
| `/interface pppoe-server server add` | สร้าง PPPoE server |
| `/ppp profile add` | สร้าง user profile |
| `/ppp secret add` | สร้าง user |
| `/ppp active print` | ดู active sessions |
| `/ppp active remove` | Disconnect user |
| `/ppp aaa set use-radius=yes` | ใช้ RADIUS |
| `rate-limit=UL/DL` | กำหนด bandwidth |
| `on-up=` / `on-down=` | Event scripts |

> **Tip:** ใช้ RADIUS สำหรับ large deployments (>100 users) เพื่อ centralized management

> **Warning:** PPPoE on-up script ที่ error อาจทำให้ session ไม่ connect ได้

> **Best Practice:** แยก IP pools ตาม package tier เพื่อง่ายต่อ monitoring และ QoS

---

[← Part 32: Hotspot Management](part-032-hotspot-management.md) | [Part 34: VPN Automation →](part-034-vpn-automation.md)
