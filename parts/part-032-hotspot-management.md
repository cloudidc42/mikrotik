# Part 32: Hotspot Management ใน RouterOS

## บทนำ

RouterOS Hotspot เป็น feature ที่ใช้สร้าง Captive Portal สำหรับ WiFi networks โดยทั่วไปในร้านกาแฟ, โรงแรม, สนามบิน หรือสถานที่สาธารณะต่างๆ

---

## 32.1 Hotspot Overview

### Hotspot Architecture

```
Client Device
    ↓ (HTTP redirect)
Captive Portal (Login Page)
    ↓ (Authentication)
RADIUS / Local User DB
    ↓ (Authorized)
Internet Access
```

### Hotspot Components

| Component | คำอธิบาย |
|-----------|---------|
| Hotspot Server | ควบคุม authentication flow |
| IP Pool | IP addresses สำหรับ clients |
| User Profiles | กำหนด bandwidth, time limits |
| Walled Garden | Sites ที่เข้าได้ก่อน login |
| SMTP Server | สำหรับส่ง vouchers |

---

## 32.2 Hotspot Server Setup

### ติดตั้ง Hotspot ด้วย Wizard

```routeros
# Setup hotspot ผ่าน IP Hotspot menu
/ip hotspot setup

# หรือ manual setup
# Step 1: สร้าง IP Pool
/ip pool add name=hotspot-pool ranges=192.168.88.10-192.168.88.200

# Step 2: สร้าง DHCP Server
/ip dhcp-server add \
    name=dhcp-hotspot \
    interface=wlan1 \
    address-pool=hotspot-pool \
    lease-time=1h \
    disabled=no

/ip dhcp-server network add \
    address=192.168.88.0/24 \
    gateway=192.168.88.1 \
    dns-server=192.168.88.1

# Step 3: กำหนด IP ให้ interface
/ip address add address=192.168.88.1/24 interface=wlan1

# Step 4: สร้าง Hotspot Server Profile
/ip hotspot profile add \
    name=default \
    hotspot-address=192.168.88.1 \
    dns-name=hotspot.local \
    html-directory=hotspot \
    http-proxy=0.0.0.0:64872 \
    login-by=http-chap \
    use-radius=no

# Step 5: สร้าง Hotspot Server
/ip hotspot add \
    name=hotspot1 \
    interface=wlan1 \
    address-pool=hotspot-pool \
    profile=default \
    disabled=no

# ตรวจสอบ hotspot
/ip hotspot print
/ip hotspot active print
```

---

## 32.3 User Profiles

```routeros
# สร้าง user profiles สำหรับ bandwidth tiers

# Free tier
/ip hotspot user profile add \
    name=free-tier \
    rate-limit=2M/2M \
    session-timeout=1h \
    shared-users=1 \
    idle-timeout=10m \
    keepalive-timeout=2m \
    on-login="" \
    on-logout=""

# Basic tier
/ip hotspot user profile add \
    name=basic-tier \
    rate-limit=5M/5M \
    session-timeout=4h \
    shared-users=2 \
    idle-timeout=30m

# Premium tier
/ip hotspot user profile add \
    name=premium-tier \
    rate-limit=20M/20M \
    session-timeout=24h \
    shared-users=5 \
    idle-timeout=2h

# Unlimited tier
/ip hotspot user profile add \
    name=unlimited \
    rate-limit=0/0 \
    session-timeout=0s \
    shared-users=10

# ดู profiles
/ip hotspot user profile print

# Script สร้าง profiles จาก config
:local createProfiles do={
    :local profiles $1
    :foreach profile in=$profiles do={
        :local name ($profile->"name")
        :local dl ($profile->"download")
        :local ul ($profile->"upload")
        :local sessionTime ($profile->"session")
        :local users ($profile->"users")
        
        /ip hotspot user profile add \
            name=$name \
            rate-limit=($ul . "/" . $dl) \
            session-timeout=$sessionTime \
            shared-users=$users
        
        :put "Created profile: $name ($dl down / $ul up)"
    }
}

:local profileConfig {
    {"name"="1hour-5mbps"; "download"="5M"; "upload"="2M"; "session"="1h"; "users"=1};
    {"name"="1day-10mbps"; "download"="10M"; "upload"="5M"; "session"="24h"; "users"=3};
    {"name"="7day-unlimited"; "download"="0"; "upload"="0"; "session"="7d"; "users"=5}
}

[$createProfiles $profileConfig]
```

---

## 32.4 Vouchers

```routeros
# สร้าง voucher users
:local generateVouchers do={
    :local count $1
    :local profile $2
    :local prefix $3
    
    :if ([:len $prefix] = 0) do={ :set prefix "VCH" }
    
    :local vouchers {}
    
    :for i from=1 to=$count do={
        # Generate random code (8 characters)
        :local code ($prefix . "-")
        :local chars "ABCDEFGHIJKLMNOPQRSTUVWXYZ0123456789"
        :for j from=1 to=8 do={
            :local idx ([:toint [:tostr $i]] + $j * 7)
            :set idx ($idx % [:len $chars])
            :set code ($code . [:pick $chars $idx ($idx + 1)])
        }
        
        # สร้าง user
        /ip hotspot user add \
            name=$code \
            password=$code \
            profile=$profile \
            comment=("Voucher generated: " . [/system clock get date])
        
        :set vouchers ($vouchers , $code)
        :put "Generated voucher: $code"
    }
    
    :return $vouchers
}

# สร้าง 10 vouchers สำหรับ 1-hour profile
:local vouchers [$generateVouchers 10 "1hour-5mbps" "DAY"]
:put "Generated vouchers: $vouchers"

# Export vouchers ไปยัง file
:local exportVouchers do={
    :local profile $1
    :local content "Voucher Code,Profile,Created\r\n"
    
    :foreach user in=[/ip hotspot user find profile=$profile] do={
        :local name [/ip hotspot user get $user name]
        :local comment [/ip hotspot user get $user comment]
        :set content ($content . $name . "," . $profile . "," . $comment . "\r\n")
    }
    
    /file remove [find name="vouchers.csv"]
    /tool fetch url="data:," dst-path="vouchers.csv"
    /file set [find name="vouchers.csv"] contents=$content
    :put "Exported vouchers to vouchers.csv"
}

[$exportVouchers "1hour-5mbps"]

# Print vouchers แบบ printable
:local printVouchers do={
    :local profile $1
    :local output ""
    
    :foreach user in=[/ip hotspot user find profile=$profile] do={
        :local name [/ip hotspot user get $user name]
        :set output ($output . "+------------------+\r\n")
        :set output ($output . "| WiFi Voucher     |\r\n")
        :set output ($output . "+------------------+\r\n")
        :set output ($output . "| Code: " . $name . " |\r\n")
        :set output ($output . "+------------------+\r\n\r\n")
    }
    
    :put $output
}
```

---

## 32.5 Login Page Customization

```routeros
# Hotspot login page อยู่ใน /hotspot/ directory บน flash
# ไฟล์หลัก: login.html, logout.html, status.html, alogin.html

# Upload custom login page
/tool fetch \
    url="http://192.168.1.100/my-login.html" \
    dst-path="hotspot/login.html"

# Create custom login page via script
:local customLogin "<!DOCTYPE html>
<html>
<head>
    <title>Welcome to Free WiFi</title>
    <style>
        body { font-family: Arial; background: #f0f0f0; }
        .login-box { 
            width: 300px; margin: 100px auto;
            background: white; padding: 20px;
            border-radius: 10px; box-shadow: 0 2px 10px rgba(0,0,0,0.1);
        }
        h1 { color: #333; text-align: center; }
        input { width: 100%; padding: 10px; margin: 5px 0; box-sizing: border-box; }
        button { width: 100%; padding: 10px; background: #4CAF50; color: white; border: none; cursor: pointer; }
    </style>
</head>
<body>
    <div class='login-box'>
        <h1>Free WiFi</h1>
        <p>Please login to access the internet</p>
        <form action='https://\$(host-name)/login' method='post'>
            <input type='text' name='username' placeholder='Username'>
            <input type='password' name='password' placeholder='Password'>
            <input type='hidden' name='dst' value='\$(link-orig)'>
            <input type='hidden' name='popup' value='true'>
            <button type='submit'>Login</button>
        </form>
    </div>
</body>
</html>"

# ตั้งค่า HTML directory
/ip hotspot profile set default html-directory=hotspot

# Walled garden - sites ที่เข้าได้ก่อน login
/ip hotspot walled-garden add dst-host=google.com
/ip hotspot walled-garden add dst-host="*.google.com"
/ip hotspot walled-garden add dst-host=facebook.com

# Walled garden IP-based
/ip hotspot walled-garden ip add \
    dst-address=8.8.8.8/32 \
    action=accept
```

---

## 32.6 Bandwidth Limits

```routeros
# ตั้ง bandwidth limit ผ่าน user profile
/ip hotspot user profile set free-tier rate-limit=2M/2M

# Dynamic rate limit ตาม time of day
/system script add name="adjust-bandwidth" source="
    :local hour [:toint [:pick [:tostr [/system clock get time]] 0 2]]
    
    :if (\$hour >= 8 && \$hour < 20) do={
        # Peak hours: 8AM-8PM - limit to 2M
        /ip hotspot user profile set free-tier rate-limit=2M/2M
    } else={
        # Off-peak: unlimited
        /ip hotspot user profile set free-tier rate-limit=10M/10M
    }
"

/system scheduler add \
    name="adjust-bandwidth" \
    interval=1h \
    on-event="/system script run adjust-bandwidth"

# Per-user rate limiting
:local setUserBandwidth do={
    :local username $1
    :local downLimit $2
    :local upLimit $3
    
    :local userId [/ip hotspot user find name=$username]
    :if ([:len $userId] > 0) do={
        /ip hotspot user set $userId \
            rate-limit=($upLimit . "/" . $downLimit)
        :put "Set bandwidth for $username: $downLimit down / $upLimit up"
    }
}

[$setUserBandwidth "vip-user" "50M" "25M"]
```

---

## 32.7 Time-based Access

```routeros
# User ที่มี session timeout
/ip hotspot user add \
    name="guest-time" \
    password="guest123" \
    profile=basic-tier \
    limit-bytes-in=500000000 \    # 500MB download
    limit-bytes-out=100000000 \   # 100MB upload
    limit-uptime=8h               # 8 hours total

# Time schedule ด้วย on-login script
:local timeBasedAccess "
    :local hour [:toint [:pick [:tostr [/system clock get time]] 0 2]]
    
    # ถ้าอยู่นอกเวลา business hours
    :if (\$hour < 8 || \$hour >= 22) do={
        # Disconnect user
        /ip hotspot active remove [find user=\$username]
        :log info (\"User blocked (off-hours): \" . \$username)
    }
"

# ตรวจสอบ users ทุก 30 นาทีและ disconnect ถ้า off-hours
/system script add name="check-hotspot-time" source="
    :local hour [:toint [:pick [:tostr [/system clock get time]] 0 2]]
    
    :if (\$hour < 8 || \$hour >= 22) do={
        # Disconnect all active users
        :local active [/ip hotspot active find]
        :if ([:len \$active] > 0) do={
            :foreach session in=\$active do={
                :local user [/ip hotspot active get \$session user]
                /ip hotspot active remove \$session
                :log info (\"Off-hours disconnect: \" . \$user)
            }
            :put \"Disconnected \" . [:len \$active] . \" users (off-hours)\"
        }
    }
"
```

---

## 32.8 Hotspot Scripts (on-login, on-logout)

```routeros
# on-login script
# ตัวแปรที่ใช้ได้: $username, $address, $mac-address, $interface

:local hotspotLoginScript "
    :local user \$username
    :local ip \$address
    :local mac \$mac-address
    :local iface \$interface
    
    # Log login
    :log info (\"Hotspot login: user=\" . \$user . \" ip=\" . \$ip . \" mac=\" . \$mac)
    
    # เพิ่มเข้า address list สำหรับ tracking
    /ip firewall address-list remove [find list=hotspot-active address=\$ip]
    /ip firewall address-list add list=hotspot-active address=\$ip comment=\$user
    
    # ส่ง welcome notification (webhook)
    :do {
        /tool fetch url=(\"http://192.168.1.100/api/login?user=\" . \$user . \"&ip=\" . \$ip) \
            dst-path=\"login-notify.tmp\"
    } on-error={}
"

# on-logout script
:local hotspotLogoutScript "
    :local user \$username
    :local ip \$address
    :local mac \$mac-address
    :local uptime \$uptime
    :local bytesIn \$bytes-in
    :local bytesOut \$bytes-out
    
    # Log logout
    :log info (\"Hotspot logout: user=\" . \$user . \" ip=\" . \$ip . \" uptime=\" . \$uptime . \" rx=\" . \$bytesIn . \" tx=\" . \$bytesOut)
    
    # ลบออกจาก address list
    /ip firewall address-list remove [find list=hotspot-active address=\$ip]
    
    # บันทึก usage statistics
    :local usageFile \"hotspot-usage.log\"
    :local logEntry ([/system clock get date] . \" \" . [/system clock get time] . \" \" . \$user . \" \" . \$uptime . \" \" . \$bytesIn . \" \" . \$bytesOut . \"\\r\\n\")
    :local fileId [/file find name=\$usageFile]
    :local existing \"\"
    :if ([:len \$fileId] > 0) do={
        :set existing [/file get \$fileId contents]
    }
    /file remove [find name=\$usageFile]
    /tool fetch url=\"data:,\" dst-path=\$usageFile
    /file set [find name=\$usageFile] contents=(\$existing . \$logEntry)
"

# Apply scripts to hotspot profile
/ip hotspot user profile set default \
    on-login=$hotspotLoginScript \
    on-logout=$hotspotLogoutScript
```

---

## 32.9 Integration with RADIUS

```routeros
# ตั้งค่า RADIUS server สำหรับ Hotspot authentication
/radius add \
    service=hotspot \
    address=192.168.1.10 \
    secret=radiussecret \
    timeout=3s

# เปิดใช้ RADIUS ใน hotspot profile
/ip hotspot profile set default \
    use-radius=yes \
    radius-accounting=yes

# ดู RADIUS settings
/radius print
/radius incoming print

# RADIUS attributes ที่ส่ง
# Called-Station-Id: MAC-SSID
# Calling-Station-Id: Client MAC
# User-Name: username
# User-Password: password (encrypted)
# NAS-Identifier: router name
# NAS-IP-Address: hotspot IP

# Test RADIUS connectivity
/radius test \
    service=hotspot \
    user=testuser \
    password=testpass

# Script ตรวจสอบ RADIUS health
:local checkRADIUS do={
    :local result [/radius test service=hotspot user=ping password=ping]
    :if ($result = "ok") do={
        :put "RADIUS: OK"
    } else={
        :put "RADIUS: FAILED - $result"
        :log error "RADIUS server not responding"
    }
}

[$checkRADIUS]
```

---

## Lab 32: Full Hotspot Deployment

### โจทย์
Deploy Hotspot สมบูรณ์สำหรับโรงแรมขนาดเล็ก:
1. Free WiFi สำหรับ lobby (2Mbps/1h)
2. Premium WiFi สำหรับห้องพัก (20Mbps/24h)
3. Voucher generation สำหรับ front desk
4. Usage reporting
5. ส่ง daily report

### Solution

```routeros
# Hotel Hotspot Deployment Lab
# =============================

# --- Network Setup ---

# Free WiFi - Lobby (wlan1, 192.168.10.0/24)
/ip pool add name=lobby-pool ranges=192.168.10.10-192.168.10.200
/ip address add address=192.168.10.1/24 interface=wlan1

/ip dhcp-server add \
    name=dhcp-lobby \
    interface=wlan1 \
    address-pool=lobby-pool \
    lease-time=2h \
    disabled=no

/ip dhcp-server network add \
    address=192.168.10.0/24 \
    gateway=192.168.10.1 \
    dns-server=192.168.10.1

# Premium WiFi - Rooms (wlan2, 192.168.20.0/24)  
/ip pool add name=rooms-pool ranges=192.168.20.10-192.168.20.250
/ip address add address=192.168.20.1/24 interface=wlan2

/ip dhcp-server add \
    name=dhcp-rooms \
    interface=wlan2 \
    address-pool=rooms-pool \
    lease-time=24h \
    disabled=no

/ip dhcp-server network add \
    address=192.168.20.0/24 \
    gateway=192.168.20.1 \
    dns-server=192.168.20.1

# --- Hotspot Profiles ---
/ip hotspot user profile add \
    name=lobby-free \
    rate-limit=1M/2M \
    session-timeout=1h \
    idle-timeout=15m \
    shared-users=1

/ip hotspot user profile add \
    name=rooms-premium \
    rate-limit=10M/20M \
    session-timeout=24h \
    idle-timeout=2h \
    shared-users=3

# --- Hotspot Servers ---
/ip hotspot profile add \
    name=lobby-profile \
    hotspot-address=192.168.10.1 \
    login-by=http-chap \
    use-radius=no

/ip hotspot profile add \
    name=rooms-profile \
    hotspot-address=192.168.20.1 \
    login-by=http-chap \
    use-radius=no

/ip hotspot add \
    name=lobby \
    interface=wlan1 \
    address-pool=lobby-pool \
    profile=lobby-profile \
    disabled=no

/ip hotspot add \
    name=rooms \
    interface=wlan2 \
    address-pool=rooms-pool \
    profile=rooms-profile \
    disabled=no

# --- Voucher Generation Script ---
/system script add name="generate-hotel-vouchers" source="
    :local type \$1   # lobby or rooms
    :local count \$2  # number of vouchers
    :local profile \"\"
    :local prefix \"\"
    
    :if (\$type = \"lobby\") do={
        :set profile \"lobby-free\"
        :set prefix \"LBY\"
    } else={
        :set profile \"rooms-premium\"
        :set prefix \"RM\"
    }
    
    :local vouchers {}
    :for i from=1 to=\$count do={
        # Simple unique code
        :local timestamp [:tostr [/system resource get uptime]]
        :local code (\$prefix . \"-\" . [:pick \$timestamp ([:len \$timestamp]-6) [:len \$timestamp]] . [:tostr \$i])
        
        /ip hotspot user add \
            name=\$code \
            password=\$code \
            profile=\$profile \
            comment=(\"Hotel voucher type=\" . \$type . \" created=\" . [/system clock get date])
        
        :set vouchers (\$vouchers , \$code)
    }
    
    :put (\"Generated \" . [:len \$vouchers] . \" vouchers for \" . \$type . \":\")
    :foreach v in=\$vouchers do={
        :put \"  Code: \$v\"
    }
" comment="Generate hotel WiFi vouchers"

# --- Usage Tracking ---
/system script add name="hotspot-usage-report" source="
    :local report \"Hotel WiFi Usage Report\\r\\n\"
    :set report (\$report . \"Date: \" . [/system clock get date] . \"\\r\\n\\r\\n\")
    
    :set report (\$report . \"Active Sessions:\\r\\n\")
    :foreach session in=[/ip hotspot active find] do={
        :local user [/ip hotspot active get \$session user]
        :local ip [/ip hotspot active get \$session address]
        :local uptime [/ip hotspot active get \$session uptime]
        :local bytesIn [/ip hotspot active get \$session bytes-in]
        :set report (\$report . \"  User: \" . \$user . \" IP: \" . \$ip . \" Uptime: \" . \$uptime . \"\\r\\n\")
    }
    
    :local activeSessions [:len [/ip hotspot active find]]
    :set report (\$report . \"\\r\\nTotal active: \" . \$activeSessions . \"\\r\\n\")
    
    :set report (\$report . \"\\r\\nVoucher Pool Status:\\r\\n\")
    :local lobbyVouchers [:len [/ip hotspot user find profile=lobby-free]]
    :local roomsVouchers [:len [/ip hotspot user find profile=rooms-premium]]
    :set report (\$report . \"  Lobby vouchers available: \" . \$lobbyVouchers . \"\\r\\n\")
    :set report (\$report . \"  Rooms vouchers available: \" . \$roomsVouchers . \"\\r\\n\")
    
    /tool e-mail send \
        to=\"manager@hotel.com\" \
        subject=\"Daily WiFi Usage Report\" \
        body=\$report
    
    :put \$report
" comment="Hotel WiFi daily report"

/system scheduler add \
    name="hotel-wifi-report" \
    start-time=08:00:00 \
    interval=1d \
    on-event="/system script run hotspot-usage-report"

# --- Demo: Generate vouchers ---
:global PARAM_TYPE "lobby"
:global PARAM_COUNT 5
/system script run generate-hotel-vouchers

:put "Hotel Hotspot deployed!"
:put "- Lobby WiFi: wlan1 (1Mbps/2Mbps, 1h sessions)"
:put "- Rooms WiFi: wlan2 (10Mbps/20Mbps, 24h sessions)"
:put "- Voucher generation scripts ready"
:put "- Daily reports configured"
```

---

## Summary ของ Part 32

| Component | Command | หมายเหตุ |
|-----------|---------|---------|
| Setup hotspot | `/ip hotspot add` | |
| User profile | `/ip hotspot user profile add` | |
| Create user | `/ip hotspot user add` | |
| Active sessions | `/ip hotspot active print` | |
| RADIUS | `/radius add service=hotspot` | |
| Login page | `/ip hotspot profile set html-directory=` | |
| Walled garden | `/ip hotspot walled-garden add` | |

> **Tip:** ใช้ Walled Garden เพื่อให้ access บาง sites ได้ก่อน login

> **Warning:** Hotspot เปิด port 80 interceptor ซึ่งอาจทำให้ HTTPS บาง sites มีปัญหา

> **Best Practice:** แยก interface สำหรับ hotspot network ออกจาก management network

---

[← Part 31: Email Notifications](part-031-email-notifications.md) | [Part 33: PPPoE Server →](part-033-pppoe-server.md)
