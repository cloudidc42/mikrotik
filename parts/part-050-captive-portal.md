# Part 50: Captive Portal บน MikroTik

## บทนำ

Captive Portal เป็นระบบที่บังคับให้ผู้ใช้เข้าหน้า login ก่อนใช้ internet เป็นระบบที่พบได้ทั่วไปใน WiFi สาธารณะ โรงแรม ร้านกาแฟ และสนามบิน MikroTik Hotspot เป็น feature built-in ที่ใช้เป็น captive portal ได้

---

## 50.1 Captive Portal Concepts

### การทำงานของ Captive Portal

```
1. User เชื่อมต่อ WiFi
2. User เปิด browser → พิมพ์ URL ใดก็ได้
3. Router redirect ไปยัง login page
4. User กรอก credentials / accept terms
5. Router ให้ access internet
```

### ส่วนประกอบหลัก

| ส่วนประกอบ | หน้าที่ |
|-----------|---------|
| Hotspot Server | จัดการ authentication |
| Login Page | HTML/CSS/JS สำหรับ UI |
| User Database | เก็บ username/password |
| RADIUS | External authentication (optional) |
| DHCP | แจก IP ให้ client |

---

## 50.2 Hotspot as Captive Portal

### ตั้งค่า Hotspot Server

```routeros
# ตั้งค่า Hotspot บน interface
/ip hotspot
add name=hotspot1 interface=wlan1 \
    address-pool=hotspot-pool \
    profile=hsprof1 \
    idle-timeout=10m \
    comment="WiFi Hotspot"

# สร้าง IP Pool
/ip pool
add name=hotspot-pool ranges=192.168.88.10-192.168.88.254

# สร้าง Hotspot Profile
/ip hotspot profile
add name=hsprof1 \
    hotspot-address=192.168.88.1 \
    dns-name=hotspot.example.com \
    login-by=http-chap \
    html-directory=hotspot \
    http-proxy=0.0.0.0:0 \
    use-radius=no

# สร้าง User Profile
/ip hotspot user profile
add name=default-unlimited \
    session-timeout=1d \
    shared-users=1 \
    rate-limit=10M/10M \
    comment="Default unlimited profile"

add name=basic-1h \
    session-timeout=1h \
    rate-limit=5M/5M \
    comment="Basic 1 hour profile"

# เพิ่ม User
/ip hotspot user
add name=testuser password=testpass123 \
    profile=default-unlimited \
    comment="Test user"
```

---

## 50.3 Custom Login Page (HTML/CSS/JS)

### โครงสร้างไฟล์ Hotspot

```
/flash/hotspot/
├── login.html          # หน้า login
├── logout.html         # หน้า logout
├── status.html         # หน้า status
├── alogin.html         # Auto login
├── redirect.html       # Redirect หลัง login
├── rlogin.html         # Login required
├── css/
│   └── style.css
├── images/
│   └── logo.png
└── js/
    └── app.js
```

### Custom Login Page (login.html)

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>Welcome - Free WiFi</title>
    <style>
        * { box-sizing: border-box; margin: 0; padding: 0; }
        
        body {
            font-family: 'Sarabun', Arial, sans-serif;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        
        .container {
            background: white;
            border-radius: 16px;
            padding: 40px;
            width: 100%;
            max-width: 420px;
            box-shadow: 0 20px 60px rgba(0,0,0,0.2);
        }
        
        .logo {
            text-align: center;
            margin-bottom: 30px;
        }
        
        .logo img {
            width: 80px;
            height: 80px;
        }
        
        h1 {
            color: #333;
            font-size: 24px;
            text-align: center;
            margin-bottom: 8px;
        }
        
        .subtitle {
            color: #666;
            text-align: center;
            font-size: 14px;
            margin-bottom: 30px;
        }
        
        .form-group {
            margin-bottom: 20px;
        }
        
        label {
            display: block;
            color: #333;
            font-size: 14px;
            margin-bottom: 6px;
            font-weight: 600;
        }
        
        input[type="text"],
        input[type="password"] {
            width: 100%;
            padding: 12px 16px;
            border: 2px solid #e0e0e0;
            border-radius: 8px;
            font-size: 16px;
            transition: border-color 0.3s;
            outline: none;
        }
        
        input:focus {
            border-color: #667eea;
        }
        
        .btn-login {
            width: 100%;
            padding: 14px;
            background: linear-gradient(135deg, #667eea 0%, #764ba2 100%);
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 16px;
            font-weight: 600;
            cursor: pointer;
            transition: opacity 0.3s;
        }
        
        .btn-login:hover {
            opacity: 0.9;
        }
        
        .error-msg {
            color: #e74c3c;
            font-size: 13px;
            text-align: center;
            margin-top: 10px;
        }
        
        .trial-btn {
            width: 100%;
            padding: 12px;
            background: transparent;
            color: #667eea;
            border: 2px solid #667eea;
            border-radius: 8px;
            font-size: 14px;
            cursor: pointer;
            margin-top: 12px;
            font-weight: 600;
        }
        
        .footer-text {
            text-align: center;
            color: #999;
            font-size: 12px;
            margin-top: 20px;
        }
    </style>
</head>
<body>
    <div class="container">
        <div class="logo">
            <img src="images/logo.png" alt="Logo" onerror="this.style.display='none'">
        </div>
        
        <h1>ยินดีต้อนรับ</h1>
        <p class="subtitle">เข้าสู่ระบบเพื่อใช้งาน Free WiFi</p>
        
        <!-- MikroTik Hotspot Login Form -->
        <form name="sendin" action="$(link-login-only)" method="post">
            <input type="hidden" name="dst" value="$(link-orig)">
            <input type="hidden" name="popup" value="true">
            
            <div class="form-group">
                <label for="username">Username</label>
                <input type="text" id="username" name="username" 
                       placeholder="กรอก username"
                       value="$(username)" autocomplete="username">
            </div>
            
            <div class="form-group">
                <label for="password">Password</label>
                <input type="password" id="password" name="password"
                       placeholder="กรอก password"
                       autocomplete="current-password">
            </div>
            
            $(if chap-id)
            <input type="hidden" name="chap-id" value="$(chap-id)">
            <input type="hidden" name="chap-challenge" value="$(chap-challenge)">
            <input type="hidden" name="sha1-challenge" value="$(sha1-challenge)">
            $(endif)
            
            <button type="submit" class="btn-login">เข้าสู่ระบบ</button>
        </form>
        
        $(if error)
        <p class="error-msg">$(error)</p>
        $(endif)
        
        <button class="trial-btn" onclick="trialAccess()">
            ทดลองใช้ฟรี 30 นาที
        </button>
        
        <p class="footer-text">
            โดยการเชื่อมต่อ คุณยอมรับ<br>
            <a href="terms.html">เงื่อนไขการใช้งาน</a>
        </p>
    </div>
    
    <script>
    function trialAccess() {
        // Auto-login ด้วย trial account
        document.querySelector('[name="username"]').value = 'trial';
        document.querySelector('[name="password"]').value = 'trial123';
        document.querySelector('form').submit();
    }
    </script>
</body>
</html>
```

---

## 50.4 Social Login Integration

### Facebook Login Integration

```html
<!-- Facebook Login Button (ต้องใช้ FB App ID) -->
<div id="fb-root"></div>
<script>
window.fbAsyncInit = function() {
    FB.init({
        appId: 'YOUR_APP_ID',
        cookie: true,
        xfbml: true,
        version: 'v18.0'
    });
};

(function(d, s, id) {
    var js, fjs = d.getElementsByTagName(s)[0];
    if (d.getElementById(id)) return;
    js = d.createElement(s); js.id = id;
    js.src = "https://connect.facebook.net/en_US/sdk.js";
    fjs.parentNode.insertBefore(js, fjs);
}(document, 'script', 'facebook-jssdk'));

function loginWithFacebook() {
    FB.login(function(response) {
        if (response.authResponse) {
            // ส่ง token ไปยัง backend สำหรับ verification
            var token = response.authResponse.accessToken;
            
            // Redirect พร้อม token
            window.location.href = '/social-login?provider=facebook&token=' + token;
        }
    }, {scope: 'public_profile,email'});
}
</script>

<button class="btn-facebook" onclick="loginWithFacebook()">
    <svg><!-- Facebook icon --></svg>
    เข้าสู่ระบบด้วย Facebook
</button>
```

> **Note:** การทำ Social Login ต้องการ backend server ที่รับ OAuth token และ authenticate กับ MikroTik Hotspot ผ่าน RADIUS หรือ API

---

## 50.5 SMS Verification

### ขั้นตอน SMS Verification

```
1. User ใส่เบอร์โทรศัพท์
2. ระบบส่ง OTP ผ่าน SMS gateway
3. User ใส่ OTP
4. Backend verify และ activate hotspot user
5. User ได้ internet access
```

### Backend Script (PHP/Python ตัวอย่าง)

```python
# sms_verify.py - ตัวอย่าง SMS verification backend
import routeros_api
import random
import datetime

def generate_otp():
    return str(random.randint(100000, 999999))

def send_sms(phone_number, otp_code):
    # ใช้ SMS gateway API (เช่น AWS SNS, Twilio)
    message = f"รหัส OTP ของคุณคือ: {otp_code} (หมดอายุใน 5 นาที)"
    # implement SMS sending here
    print(f"SMS sent to {phone_number}: {message}")
    return True

def create_hotspot_user(router_ip, username, password, session_timeout="1h"):
    """สร้าง MikroTik Hotspot User ผ่าน API"""
    connection = routeros_api.RouterOsApiPool(
        router_ip, username='admin', password='', port=8728
    )
    api = connection.get_api()
    
    hotspot_api = api.get_resource('/ip/hotspot/user')
    hotspot_api.add(
        name=username,
        password=password,
        profile='basic-1h',
        comment=f'SMS Verified: {datetime.datetime.now()}'
    )
    
    connection.disconnect()
    print(f"Created hotspot user: {username}")

# Example usage flow:
# 1. User requests OTP
otp = generate_otp()
phone = "+66812345678"
send_sms(phone, otp)

# 2. User enters OTP - verify it
entered_otp = input("Enter OTP: ")
if entered_otp == otp:
    # สร้าง user ใน MikroTik
    username = f"sms_{phone[-8:]}"
    password = otp
    create_hotspot_user("192.168.1.1", username, password)
    print("Access granted!")
```

---

## 50.6 Voucher-based Access

### Voucher System

```routeros
/system script
add name="generate-vouchers" source={
    :local count 10
    :local duration "2h"
    :local bandwidth "10M"
    :local prefix "VCH"
    :local generated 0
    
    :put "Generating vouchers..."
    :put ""
    :put "Code          | Duration | Bandwidth"
    :put "------------- | -------- | ---------"
    
    :for i from=1 to=$count do={
        # สร้าง random code
        :local code ($prefix . \
            [:tostr ([:rndnum from=1000 to=9999])] . \
            [:tostr ([:rndnum from=1000 to=9999])])
        
        # สร้าง user ใน Hotspot
        :do {
            /ip hotspot user add \
                name=$code \
                password=$code \
                profile=("voucher-" . $duration) \
                comment=("Voucher " . $i . " | BW:" . $bandwidth)
            
            :set generated ($generated + 1)
            :put ($code . " | " . $duration . "     | " . $bandwidth)
        } on-error={
            :log error ("Failed to create voucher: " . $code)
        }
    }
    
    :put ""
    :put ("Generated " . $generated . " vouchers")
}

# สร้าง voucher profile
/ip hotspot user profile
add name="voucher-2h" session-timeout=2h \
    rate-limit=10M/10M shared-users=1 \
    comment="2 hour voucher"

add name="voucher-1d" session-timeout=1d \
    rate-limit=10M/10M shared-users=1 \
    comment="1 day voucher"
```

---

## 50.7 Time-limited Access

### สร้าง Time-limited Users

```routeros
/system script
add name="create-time-limited-user" source={
    :local username $1
    :local hours $2
    
    :if ([:len $hours] = 0) do={ :set hours 1 }
    :local sessionTimeout ($hours . "h")
    
    # สร้าง user
    /ip hotspot user add \
        name=$username \
        password=$username \
        profile=("time-" . $hours . "h") \
        comment=("Created: " . [/system clock get time] . \
            " Exp: " . $hours . "h")
    
    :log info ("Time-limited user created: " . $username . \
        " for " . $hours . " hours")
    :put ("User " . $username . " created with " . $hours . "h access")
}
```

---

## 50.8 Bandwidth Control

### Hotspot User Profiles ด้วย Bandwidth Control

```routeros
/ip hotspot user profile

# Free tier - 1Mbps
add name="free" \
    session-timeout=30m \
    rate-limit=1M/512k \
    shared-users=3 \
    comment="Free tier: 30min/1Mbps"

# Basic - 5Mbps
add name="basic" \
    session-timeout=2h \
    rate-limit=5M/2M \
    shared-users=1 \
    comment="Basic: 2h/5Mbps"

# Premium - 20Mbps
add name="premium" \
    session-timeout=1d \
    rate-limit=20M/10M \
    shared-users=1 \
    comment="Premium: 24h/20Mbps"

# Corporate - Unlimited
add name="corporate" \
    rate-limit=100M/50M \
    shared-users=10 \
    comment="Corporate: Unlimited"
```

---

## 50.9 Analytics and Reporting

### Hotspot User Activity Report

```routeros
/system script
add name="hotspot-report" source={
    :put "============================================"
    :put " HOTSPOT ACTIVITY REPORT"
    :put "============================================"
    :put ("Date: " . [/system clock get date])
    :put ""
    
    # Active users
    :local activeUsers [/ip hotspot active find]
    :put ("Active Sessions: " . [:len $activeUsers])
    :put ""
    
    :put "--- Active Users ---"
    :foreach user in=$activeUsers do={
        :local name [/ip hotspot active get $user user]
        :local ip [/ip hotspot active get $user address]
        :local mac [/ip hotspot active get $user mac-address]
        :local uptime [/ip hotspot active get $user uptime]
        :local bytesIn [/ip hotspot active get $user bytes-in]
        :local bytesOut [/ip hotspot active get $user bytes-out]
        
        :put ($name . " | " . $ip . " | Up: " . $uptime)
        :put ("  Data: ↓" . $bytesIn . " ↑" . $bytesOut)
    }
    :put ""
    
    # Total registered users
    :local totalUsers [/ip hotspot user find]
    :put ("Total Registered Users: " . [:len $totalUsers])
    
    # Users by profile
    :put "--- Users by Profile ---"
    :foreach profile in=[/ip hotspot user profile find] do={
        :local profileName [/ip hotspot user profile get $profile name]
        :local usersInProfile [/ip hotspot user find profile=$profileName]
        :put ($profileName . ": " . [:len $usersInProfile] . " users")
    }
}

# Schedule รายงานทุกชั่วโมง
/system scheduler
add name="hotspot-hourly-report" interval=1h \
    on-event="/system script run hotspot-report"
```

---

## 50.10 Lab: Full Captive Portal System

### Lab Overview

| ระบบ | รายละเอียด |
|------|------------|
| Interface | wlan1 (WiFi) |
| Pool | 192.168.88.10-254 |
| Portal | Custom login page |
| Features | Vouchers, Time-limited, Bandwidth control |

### Complete Setup

```routeros
# ========================================
# COMPLETE CAPTIVE PORTAL SETUP
# ========================================

# 1. IP Pool
/ip pool
add name="hotspot-pool" ranges=192.168.88.10-192.168.88.200

# 2. DHCP Server
/ip dhcp-server
add name="hotspot-dhcp" interface=wlan1 \
    address-pool=hotspot-pool \
    authoritative=yes

/ip dhcp-server network
add address=192.168.88.0/24 gateway=192.168.88.1 \
    dns-server=192.168.88.1 \
    comment="Hotspot network"

# 3. Interface IP
/ip address
add address=192.168.88.1/24 interface=wlan1

# 4. Hotspot Setup
/ip hotspot setup

# หรือ manual setup
/ip hotspot profile
add name="main-profile" \
    hotspot-address=192.168.88.1 \
    dns-name="wifi.example.com" \
    html-directory=flash/hotspot \
    login-by=http-chap \
    use-radius=no

/ip hotspot
add name="main-hotspot" \
    interface=wlan1 \
    address-pool=hotspot-pool \
    profile=main-profile \
    idle-timeout=10m

# 5. User Profiles
/ip hotspot user profile
add name="free-30min" session-timeout=30m \
    rate-limit=2M/1M shared-users=2 \
    comment="Free 30 minutes"

add name="paid-1day" session-timeout=1d \
    rate-limit=10M/5M shared-users=1 \
    comment="Paid 1 day plan"

# 6. Walled Garden (sites accessible without login)
/ip hotspot walled-garden
add dst-host=*.facebook.com action=allow comment="Facebook"
add dst-host=*.google.com action=allow comment="Google"

# 7. เพิ่ม test users
/ip hotspot user
add name="free" password="free" profile=free-30min
add name="testuser" password="test123" profile=paid-1day

:log info "Captive portal setup complete!"
:put "Captive portal configured successfully!"
```

### Upload Custom Login Page

```bash
# บน management PC
# Upload hotspot files ผ่าน FTP
ftp 192.168.88.1
# login as admin
cd flash/hotspot
put login.html
put logout.html
put status.html
bye
```

### Verification

```routeros
# ดู hotspot status
/ip hotspot print

# ดู active users
/ip hotspot active print

# ดู user list
/ip hotspot user print

# ดู logs
/log print where topics~"hotspot"
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Hotspot Setup | Basic captive portal configuration |
| Custom Login | HTML/CSS/JS login page |
| Social Login | Facebook OAuth integration |
| SMS Verification | OTP-based authentication |
| Vouchers | Code-based access |
| Time-limited | Hourly/daily access control |
| Bandwidth Control | Per-profile rate limiting |
| Analytics | User activity reporting |
| Walled Garden | Pre-auth accessible sites |

---

[← Part 49: Traffic Shaping](part-049-traffic-shaping.md) | [Part 51: MikroTik API Intro →](part-051-mikrotik-api-intro.md)
