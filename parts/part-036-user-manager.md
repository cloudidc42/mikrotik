# Part 36: User Manager ใน RouterOS

## บทนำ

User Manager เป็น built-in RADIUS server ของ RouterOS ที่ใช้จัดการ users สำหรับ Hotspot และ PPPoE ช่วยให้ไม่ต้องพึ่ง external RADIUS server

---

## 36.1 User Manager Overview

```routeros
# ตรวจสอบ User Manager package
/system package print

# User Manager ต้องติดตั้ง package แยก (RouterOS < 7.x)
# RouterOS 7.x: user-manager รวมอยู่แล้ว

# Enable User Manager
/user-manager router add name=local-router address=127.0.0.1 shared-secret=your-secret

# ดู User Manager router
/user-manager router print

# Config User Manager
/user-manager set enabled=yes
```

---

## 36.2 User Database Management

```routeros
# สร้าง user
/user-manager user add name=john password=pass123 shared-users=1

# สร้าง user พร้อม profile
/user-manager user add \
    name=alice \
    password=alice123 \
    profile=basic-plan \
    comment="Alice - Basic plan"

# User พร้อม attributes
/user-manager user add \
    name=bob \
    password=bob456 \
    shared-users=1 \
    comment="Bob - Premium"

# Bulk user creation script
:local createUsers do={
    :local users {
        {"user001"; "pass001"; "basic"};
        {"user002"; "pass002"; "basic"};
        {"user003"; "pass003"; "premium"};
        {"user004"; "pass004"; "premium"};
        {"user005"; "pass005"; "basic"}
    }
    
    :foreach u in=$users do={
        :local username ($u->0)
        :local password ($u->1)
        :local profile ($u->2)
        
        :do {
            /user-manager user add name=$username password=$password profile=$profile
            :put "Created: $username ($profile)"
        } on-error={
            :put "Error creating: $username"
        }
    }
}

[$createUsers]

# ดู users
/user-manager user print

# ลบ user
/user-manager user remove [find name=john]
```

---

## 36.3 Profile Management

```routeros
# สร้าง profile พื้นฐาน
/user-manager profile add name=basic \
    name-for-users="Basic Plan" \
    shared-users=1 \
    price=0

/user-manager profile add name=standard \
    name-for-users="Standard Plan" \
    shared-users=2 \
    price=200

/user-manager profile add name=premium \
    name-for-users="Premium Plan" \
    shared-users=5 \
    price=500

# Profile limitation (bandwidth)
/user-manager profile-limitation add \
    name=basic-limit \
    rate-limit=5M/2M \
    owner=basic

/user-manager profile-limitation add \
    name=standard-limit \
    rate-limit=20M/10M \
    owner=standard

/user-manager profile-limitation add \
    name=premium-limit \
    rate-limit=50M/20M \
    owner=premium

# Profile limitation สำหรับ time-based
/user-manager profile-limitation add \
    name=day-limit \
    from-time=08:00 \
    till-time=22:00 \
    rate-limit=20M/10M

/user-manager profile-limitation add \
    name=night-limit \
    from-time=22:00 \
    till-time=08:00 \
    rate-limit=100M/50M

# Profile limitation สำหรับ quota
/user-manager profile-limitation add \
    name=daily-quota \
    download-limit=5G \
    upload-limit=1G
```

---

## 36.4 Voucher Generation

```routeros
# สร้าง voucher profile
/user-manager profile add name=voucher-1h \
    name-for-users="1 Hour Voucher" \
    price=20

/user-manager profile-limitation add \
    name=voucher-1h-limit \
    validity=1h \
    rate-limit=10M/5M \
    owner=voucher-1h

# Script สร้าง vouchers bulk
/system script add name="generate-vouchers" source="
    :local count 20
    :local prefix \"V\"
    :local profile \"voucher-1h\"
    :local vouchersCreated 0
    
    :for i from=1 to=\$count do={
        # Generate random 6-char code
        :local chars \"ABCDEFGHJKMNPQRSTVWXYZ23456789\"
        :local code \"\"
        :for j from=1 to=6 do={
            :local pos ([:rnd from=0 to=29])
            :set code (\$code . [:pick \$chars \$pos (\$pos+1)])
        }
        :local username (\$prefix . \$code)
        :local password \$code
        
        :do {
            /user-manager user add name=\$username password=\$password profile=\$profile
            :set vouchersCreated (\$vouchersCreated + 1)
        } on-error={
            :put (\"Error creating voucher: \" . \$username)
        }
    }
    :put (\"Created \" . \$vouchersCreated . \" vouchers\")
" comment="Voucher generator"

/system script run generate-vouchers

# Export vouchers to CSV
/system script add name="export-vouchers" source="
    :local csv \"Username,Password,Profile,Status\\r\\n\"
    :foreach user in=[/user-manager user find] do={
        :local name [/user-manager user get \$user name]
        :local pass [/user-manager user get \$user password]
        :local profile [/user-manager user get \$user profile]
        :local status \"active\"
        :set csv (\$csv . \$name . \",\" . \$pass . \",\" . \$profile . \",\" . \$status . \"\\r\\n\")
    }
    /file remove [find name=\"vouchers.csv\"]
    :do {
        /tool fetch url=\"data:text/csv,\" dst-path=\"vouchers.csv\"
    } on-error={}
    /file set [find name=\"vouchers.csv\"] contents=\$csv
    :put \"Vouchers exported to vouchers.csv\"
" comment="Export vouchers to CSV"
```

---

## 36.5 Hotspot Integration

```routeros
# ตั้งค่า Hotspot ให้ใช้ User Manager เป็น RADIUS
/ip hotspot profile set default use-radius=yes

/radius add \
    service=hotspot \
    address=127.0.0.1 \
    secret=your-secret \
    src-address=127.0.0.1

# Test RADIUS connection
/radius test

# User Manager router config
/user-manager router add \
    name=hotspot-router \
    address=127.0.0.1 \
    shared-secret=your-secret

# ตรวจสอบ RADIUS accounting
/radius incoming set enabled=yes port=3799

# Script ดู active sessions
:local showSessions do={
    :put "=== Active Hotspot Sessions ==="
    :foreach session in=[/ip hotspot active find] do={
        :local user [/ip hotspot active get $session user]
        :local ip [/ip hotspot active get $session address]
        :local uptime [/ip hotspot active get $session uptime]
        :put "$user - $ip (up: $uptime)"
    }
}

[$showSessions]
```

---

## 36.6 PPPoE Integration

```routeros
# ตั้งค่า PPPoE server ให้ใช้ User Manager
/ppp profile set default use-radius=yes

/radius add \
    service=ppp \
    address=127.0.0.1 \
    secret=ppp-secret

# User Manager สำหรับ PPPoE users
/user-manager user add name=pppoe-user1 password=ppp123 profile=pppoe-basic

# Profile สำหรับ PPPoE
/user-manager profile add name=pppoe-basic name-for-users="PPPoE Basic"
/user-manager profile-limitation add name=pppoe-basic-lim rate-limit=20M/10M owner=pppoe-basic

# Monitor PPPoE via User Manager sessions
:local showPPPoEStats do={
    :put "=== PPPoE Active Sessions ==="
    :foreach active in=[/ppp active find] do={
        :local name [/ppp active get $active name]
        :local address [/ppp active get $active address]
        :local uptime [/ppp active get $active uptime]
        :put "$name - $address (up: $uptime)"
    }
}

[$showPPPoEStats]
```

---

## 36.7 Usage Reports

```routeros
# Generate User Manager report
/system script add name="um-report" source="
    :local report \"=== User Manager Report ===\\r\\n\"
    :set report (\$report . \"Date: \" . [/system clock get date] . \"\\r\\n\\r\\n\")
    
    # Total users
    :local totalUsers [:len [/user-manager user find]]
    :set report (\$report . \"Total Users: \" . \$totalUsers . \"\\r\\n\")
    
    # Users per profile
    :set report (\$report . \"\\r\\nUsers by Profile:\\r\\n\")
    :foreach profile in=[/user-manager profile find] do={
        :local pname [/user-manager profile get \$profile name]
        :local count [:len [/user-manager user find profile=\$pname]]
        :set report (\$report . \"  \" . \$pname . \": \" . \$count . \"\\r\\n\")
    }
    
    # Active sessions
    :local activeSessions [:len [/ip hotspot active find]]
    :set report (\$report . \"\\r\\nActive Hotspot Sessions: \" . \$activeSessions . \"\\r\\n\")
    
    /tool e-mail send to=\"admin@company.com\" \
        subject=(\"User Manager Report - \" . [/system clock get date]) \
        body=\$report
    
    :put \$report
" comment="User Manager report"

/system scheduler add \
    name="um-daily-report" \
    start-time=08:00:00 \
    interval=1d \
    on-event="/system script run um-report"
```

---

## Lab 36: User Manager Deployment

### Solution

```routeros
# Complete User Manager deployment
# ===================================

# 1. Enable User Manager
/user-manager set enabled=yes

# 2. สร้าง profiles
/user-manager profile add name=free name-for-users="Free (1hr)"
/user-manager profile add name=daily name-for-users="Daily Pass"
/user-manager profile add name=weekly name-for-users="Weekly"
/user-manager profile add name=monthly name-for-users="Monthly"

# 3. Profile limitations
/user-manager profile-limitation add name=free-lim rate-limit=1M/512k validity=1h owner=free
/user-manager profile-limitation add name=daily-lim rate-limit=5M/2M validity=1d owner=daily
/user-manager profile-limitation add name=weekly-lim rate-limit=10M/5M validity=7d owner=weekly
/user-manager profile-limitation add name=monthly-lim rate-limit=20M/10M validity=30d owner=monthly

# 4. สร้าง demo users
/user-manager user add name=demo1 password=demo1 profile=free
/user-manager user add name=customer001 password=cust001 profile=monthly

# 5. RADIUS setup
/radius add service=hotspot address=127.0.0.1 secret=secret123 src-address=127.0.0.1

# 6. User Manager router
/user-manager router add name=main address=127.0.0.1 shared-secret=secret123

# 7. Test
:put "User Manager configured successfully"
/user-manager user print
```

---

## Summary ของ Part 36

| Component | หน้าที่ |
|-----------|--------|
| Profiles | กำหนด plan/package |
| Users | บัญชีผู้ใช้ |
| RADIUS | Authentication protocol |
| Limitations | Bandwidth/time/quota |

> **Tip:** User Manager เหมาะกับ network ขนาดกลาง ถ้าใหญ่มากควรใช้ FreeRADIUS

> **Note:** RouterOS 7.x รวม User Manager ในตัวแล้ว ไม่ต้องติดตั้ง package แยก

---

[← Part 35: Bandwidth Management](part-035-bandwidth-management.md) | [Part 37: Log Monitoring →](part-037-log-monitoring.md)
