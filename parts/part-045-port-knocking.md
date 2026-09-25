# Part 45: Port Knocking บน MikroTik

## บทนำ

Port Knocking เป็นเทคนิคการรักษาความปลอดภัยที่ซ่อน service (เช่น SSH) โดยต้องส่ง packet ไปยัง port ที่กำหนดตามลำดับก่อนจึงจะเข้าถึงได้ ช่วยลด exposure จาก brute force attacks และ port scanner

---

## 45.1 Port Knocking Concept

### Port Knocking ทำงานอย่างไร?

```
ขั้นตอนปกติ (ไม่มี Port Knocking):
Client ──────────────── SSH (port 22) ────→ Server
                        ↑ firewall block

ขั้นตอนกับ Port Knocking:
Client ─ knock port 1234 ─────────────────→ Server
Client ─ knock port 5678 ─────────────────→ Server  
Client ─ knock port 9012 ─────────────────→ Server
Client ─────────────── SSH (port 22) ─────→ Server ✓
         ↑ เปิดเฉพาะ IP ที่ knock ถูก
```

### ข้อดีและข้อเสีย

| ข้อดี | ข้อเสีย |
|-------|---------|
| ซ่อน service จาก port scanner | อาจถูก replay attack |
| ลด brute force attempts | ต้องจำ knock sequence |
| ไม่ต้องเปิด port ตลอดเวลา | อาจมีปัญหาถ้า packet loss |
| เพิ่มชั้น security | ไม่ใช่ replacement สำหรับ strong auth |

> **Warning:** Port knocking เป็นแค่ "security through obscurity" ไม่ควรใช้แทน password ที่แข็งแกร่งหรือ SSH key authentication

---

## 45.2 Implementation with Firewall

### หลักการทำงานบน MikroTik

```
1. Client ส่ง packet ไปยัง port 1234 (step 1)
   → Router เพิ่ม IP ใน list "knock-1"

2. Client ส่ง packet ไปยัง port 5678 (step 2)
   → ถ้า IP อยู่ใน "knock-1" แล้ว เพิ่มใน "knock-2"
   → ลบออกจาก "knock-1"

3. Client ส่ง packet ไปยัง port 9012 (step 3)
   → ถ้า IP อยู่ใน "knock-2" แล้ว เพิ่มใน "allowed-ssh"
   → ลบออกจาก "knock-2"

4. Client เชื่อมต่อ SSH
   → ถ้า IP อยู่ใน "allowed-ssh" → ACCEPT
   → ถ้า IP ไม่อยู่ → DROP
```

---

## 45.3 Single Knock Sequence

### การตั้งค่า Port Knocking แบบง่าย (3-knock)

```routeros
# ========================================
# PORT KNOCKING SETUP - 3 KNOCK SEQUENCE
# Sequence: 1234 → 5678 → 9012 → SSH
# ========================================

/ip firewall filter

# STEP 1: Knock on port 1234
add chain=input protocol=tcp dst-port=1234 \
    action=add-src-to-address-list \
    address-list=knock-1 \
    address-list-timeout=30s \
    comment="PortKnock: Step 1 - Add to knock-1"

# STEP 2: Knock on port 5678 (must be in knock-1)
add chain=input protocol=tcp dst-port=5678 \
    src-address-list=knock-1 \
    action=add-src-to-address-list \
    address-list=knock-2 \
    address-list-timeout=30s \
    comment="PortKnock: Step 2 - Add to knock-2"

# Remove from knock-1 if they tried step 2 wrong order
add chain=input protocol=tcp dst-port=5678 \
    src-address-list=!knock-1 \
    action=add-src-to-address-list \
    address-list=knock-fail \
    address-list-timeout=1h \
    comment="PortKnock: Wrong sequence -> fail list"

# STEP 3: Knock on port 9012 (must be in knock-2)
add chain=input protocol=tcp dst-port=9012 \
    src-address-list=knock-2 \
    action=add-src-to-address-list \
    address-list=allowed-ssh \
    address-list-timeout=5m \
    comment="PortKnock: Step 3 - Add to allowed-ssh"

# Block wrong-sequence knockers
add chain=input src-address-list=knock-fail \
    action=drop \
    comment="PortKnock: Block failed knockers"

# ALLOW SSH only for successful knockers
add chain=input protocol=tcp dst-port=22 \
    src-address-list=allowed-ssh \
    action=accept \
    comment="PortKnock: Allow SSH for successful knock"

# BLOCK all other SSH attempts
add chain=input protocol=tcp dst-port=22 \
    action=drop \
    comment="PortKnock: Block all other SSH"
```

---

## 45.4 Multi-knock Sequences

### การตั้งค่า 5-knock Sequence

```routeros
# 5-knock sequence: 1111 → 2222 → 3333 → 4444 → 5555 → SSH
/ip firewall filter

# Step 1
add chain=input protocol=tcp dst-port=1111 \
    action=add-src-to-address-list \
    address-list=pk-step1 \
    address-list-timeout=15s \
    comment="PK5: Step 1"

# Step 2
add chain=input protocol=tcp dst-port=2222 \
    src-address-list=pk-step1 \
    action=add-src-to-address-list \
    address-list=pk-step2 \
    address-list-timeout=15s \
    comment="PK5: Step 2"

# Step 3
add chain=input protocol=tcp dst-port=3333 \
    src-address-list=pk-step2 \
    action=add-src-to-address-list \
    address-list=pk-step3 \
    address-list-timeout=15s \
    comment="PK5: Step 3"

# Step 4
add chain=input protocol=tcp dst-port=4444 \
    src-address-list=pk-step3 \
    action=add-src-to-address-list \
    address-list=pk-step4 \
    address-list-timeout=15s \
    comment="PK5: Step 4"

# Step 5 - Final
add chain=input protocol=tcp dst-port=5555 \
    src-address-list=pk-step4 \
    action=add-src-to-address-list \
    address-list=pk-allowed \
    address-list-timeout=10m \
    comment="PK5: Step 5 - Grant access"

# Allow SSH
add chain=input protocol=tcp dst-port=22 \
    src-address-list=pk-allowed \
    action=accept \
    comment="PK5: Allow SSH"

# Default deny SSH
add chain=input protocol=tcp dst-port=22 \
    action=drop \
    comment="PK5: Block SSH"
```

### Mixed Protocol Knocking (TCP + UDP)

```routeros
# ใช้ mix ของ TCP และ UDP ทำให้ยากขึ้น
# Sequence: TCP/1234 → UDP/5678 → TCP/9012

/ip firewall filter

# Knock 1: TCP 1234
add chain=input protocol=tcp dst-port=1234 \
    action=add-src-to-address-list \
    address-list=pk-mix-1 address-list-timeout=20s \
    comment="PKmix: TCP knock 1"

# Knock 2: UDP 5678 (ต้องอยู่ใน list 1)
add chain=input protocol=udp dst-port=5678 \
    src-address-list=pk-mix-1 \
    action=add-src-to-address-list \
    address-list=pk-mix-2 address-list-timeout=20s \
    comment="PKmix: UDP knock 2"

# Knock 3: TCP 9012 (ต้องอยู่ใน list 2)
add chain=input protocol=tcp dst-port=9012 \
    src-address-list=pk-mix-2 \
    action=add-src-to-address-list \
    address-list=pk-mix-allowed address-list-timeout=5m \
    comment="PKmix: TCP knock 3"

# Allow SSH
add chain=input protocol=tcp dst-port=22 \
    src-address-list=pk-mix-allowed \
    action=accept comment="PKmix: Allow SSH"

add chain=input protocol=tcp dst-port=22 \
    action=drop comment="PKmix: Default deny SSH"
```

---

## 45.5 Time-based Access

### เปิด SSH เฉพาะช่วงเวลาทำงาน

```routeros
/ip firewall filter

# Allow SSH เฉพาะ 8:00 - 18:00 จันทร์-ศุกร์
add chain=input protocol=tcp dst-port=22 \
    src-address-list=allowed-ssh \
    time=8h-18h,mon,tue,wed,thu,fri \
    action=accept \
    comment="PK+Time: Allow SSH during work hours"

# นอกเวลา - ต้องมี emergency access
add chain=input protocol=tcp dst-port=22 \
    src-address-list=emergency-access \
    action=accept \
    comment="PK+Time: Emergency after-hours access"

# Block ทั้งหมดที่เหลือ
add chain=input protocol=tcp dst-port=22 \
    action=drop \
    comment="PK+Time: Block SSH"
```

### Script ตั้งค่า Time-based Access

```routeros
/system script
add name="time-based-ssh-control" source={
    :local currentHour [:tonum [:pick [/system clock get time] 0 2]]
    :local currentDay [/system clock get day-of-week]
    :local workDays {"mon";"tue";"wed";"thu";"fri"}
    :local isWorkDay false
    
    :foreach day in=$workDays do={
        :if ($day = $currentDay) do={
            :set isWorkDay true
        }
    }
    
    :if ($isWorkDay && $currentHour >= 8 && $currentHour < 18) do={
        # Work hours - require knock sequence
        /ip firewall filter set [find comment="SSH-TIMEGATE"] disabled=no
        :log info "SSH: Time gate opened for work hours"
    } else={
        # Off hours - require emergency access
        /ip firewall filter set [find comment="SSH-TIMEGATE"] disabled=yes
        :log info "SSH: Time gate closed (off hours)"
    }
}

# ตรวจสอบทุก 5 นาที
/system scheduler
add name="time-ssh" interval=5m \
    on-event="/system script run time-based-ssh-control"
```

---

## 45.6 Port Knocking + VPN

### เปิด VPN Port ด้วย Port Knocking

```routeros
# ซ่อน WireGuard/OpenVPN port ด้วย knocking

# Knock sequence สำหรับ WireGuard (port 51820)
/ip firewall filter

add chain=input protocol=tcp dst-port=8080 \
    action=add-src-to-address-list \
    address-list=vpn-knock-1 \
    address-list-timeout=30s \
    comment="VPN Knock: Step 1"

add chain=input protocol=tcp dst-port=8443 \
    src-address-list=vpn-knock-1 \
    action=add-src-to-address-list \
    address-list=vpn-allowed \
    address-list-timeout=1h \
    comment="VPN Knock: Step 2 - Grant VPN access"

# Allow WireGuard
add chain=input protocol=udp dst-port=51820 \
    src-address-list=vpn-allowed \
    action=accept \
    comment="VPN: Allow WireGuard for knocked IPs"

# Default deny WireGuard
add chain=input protocol=udp dst-port=51820 \
    action=drop \
    comment="VPN: Default deny WireGuard"
```

---

## 45.7 Security Analysis

### ความเสี่ยงของ Port Knocking

```
1. Replay Attack
   - Attacker capture knock packets แล้ว replay
   - แก้ไข: ใช้ time-based sequences หรือ OTP knocking

2. Packet Loss
   - ถ้า knock packet หาย จะต้อง knock ใหม่ทั้งหมด
   - แก้ไข: ส่ง knock หลาย packet

3. Wrong Sequence Detection
   - Attacker อาจ brute force sequence
   - แก้ไข: ใช้ high port numbers และหลาย steps
```

### เพิ่มความปลอดภัย

```routeros
# เพิ่ม rate limiting สำหรับ knock attempts
/ip firewall filter

# Block IPs ที่ knock ผิดเกิน 5 ครั้ง
add chain=input \
    src-address-list=knock-fail \
    action=drop \
    comment="PK Security: Block failed knockers"

# Tarpit ผู้ที่พยายาม port scan
add chain=input protocol=tcp \
    psd=21,3s,3,1 \
    action=tarpit \
    comment="PK Security: Tarpit port scanners"
```

---

## 45.8 Logging and Monitoring

### Log Knock Activities

```routeros
/ip firewall filter

# Log knock attempts
add chain=input protocol=tcp dst-port=1234 \
    action=add-src-to-address-list \
    address-list=knock-1 \
    address-list-timeout=30s \
    log=yes \
    log-prefix="KNOCK-1:" \
    comment="PortKnock: Step 1 (logged)"

# Log successful access
add chain=input protocol=tcp dst-port=22 \
    src-address-list=allowed-ssh \
    action=accept \
    log=yes \
    log-prefix="SSH-ALLOWED:" \
    comment="PortKnock: SSH allowed (logged)"

# Log blocked SSH
add chain=input protocol=tcp dst-port=22 \
    action=drop \
    log=yes \
    log-prefix="SSH-BLOCKED:" \
    comment="PortKnock: SSH blocked (logged)"
```

### Monitoring Script

```routeros
/system script
add name="pk-monitor" source={
    :put "=== Port Knocking Status ==="
    :put ""
    
    # ดู IPs ที่กำลัง knock
    :local knock1List [/ip firewall address-list find list=knock-1]
    :local knock2List [/ip firewall address-list find list=knock-2]
    :local allowedList [/ip firewall address-list find list=allowed-ssh]
    :local failList [/ip firewall address-list find list=knock-fail]
    
    :put ("Currently in Step 1: " . [:len $knock1List] . " IPs")
    :put ("Currently in Step 2: " . [:len $knock2List] . " IPs")
    :put ("Currently Allowed:   " . [:len $allowedList] . " IPs")
    :put ("Failed/Blocked:      " . [:len $failList] . " IPs")
    :put ""
    
    :put "=== Allowed IPs ==="
    :foreach entry in=$allowedList do={
        :local addr [/ip firewall address-list get $entry address]
        :local timeout [/ip firewall address-list get $entry timeout]
        :put ($addr . " (expires: " . $timeout . ")")
    }
    
    :put ""
    :put "=== Blocked IPs ==="
    :foreach entry in=$failList do={
        :local addr [/ip firewall address-list get $entry address]
        :put ($addr)
    }
}
```

---

## 45.9 Script Automation

### Script สำหรับ Port Knock ฝั่ง Client (RouterOS)

```routeros
/system script
add name="do-port-knock" source={
    # Port knock script สำหรับ client (ถ้า client เป็น MikroTik)
    :local targetRouter "203.0.113.10"
    :local knockPorts {1234; 5678; 9012}
    :local knockDelay 500ms
    
    :put ("Starting port knock sequence to " . $targetRouter)
    
    :foreach port in=$knockPorts do={
        :put ("Knocking port " . $port . "...")
        
        # ส่ง TCP SYN packet
        /tool fetch url=("http://" . $targetRouter . ":" . $port . "/") \
            keep-result=no \
            http-method=get
        
        :delay $knockDelay
    }
    
    :put "Knock sequence complete! Now try SSH..."
}
```

### Auto-knock Script สำหรับ Linux Client (reference)

```bash
#!/bin/bash
# knock.sh - Port knocking client script

TARGET="203.0.113.10"
PORTS=(1234 5678 9012)
DELAY=0.5

echo "Starting port knock sequence..."

for port in "${PORTS[@]}"; do
    echo "Knocking port $port..."
    # ใช้ nmap หรือ knock tool
    knock "$TARGET" "$port"
    sleep "$DELAY"
done

echo "Sequence complete! SSH should be accessible for 5 minutes"
echo "Connecting..."
ssh -p 22 admin@"$TARGET"
```

---

## 45.10 Lab: Secure SSH with Port Knocking

### Lab Overview

| รายการ | รายละเอียด |
|--------|------------|
| Objective | ซ่อน SSH ด้วย 3-step port knocking |
| Knock Sequence | TCP 7777 → TCP 8888 → TCP 9999 |
| Access Duration | 5 นาที หลัง knock สำเร็จ |
| Failed Attempts | Block 1 ชั่วโมง |

### Step 1: ตั้งค่า Firewall Rules

```routeros
# ล้าง firewall rules เก่า
/ip firewall filter remove [find comment~"PK-Lab"]

# Step 1: Knock port 7777
/ip firewall filter add \
    chain=input protocol=tcp dst-port=7777 \
    action=add-src-to-address-list \
    address-list=lab-pk-1 \
    address-list-timeout=20s \
    log=yes log-prefix="PK-Lab-1:" \
    comment="PK-Lab: Step 1"

# Step 2: Knock port 8888 (ต้องอยู่ใน step 1)
/ip firewall filter add \
    chain=input protocol=tcp dst-port=8888 \
    src-address-list=lab-pk-1 \
    action=add-src-to-address-list \
    address-list=lab-pk-2 \
    address-list-timeout=20s \
    log=yes log-prefix="PK-Lab-2:" \
    comment="PK-Lab: Step 2"

# Step 3: Knock port 9999 (ต้องอยู่ใน step 2) → GRANT ACCESS
/ip firewall filter add \
    chain=input protocol=tcp dst-port=9999 \
    src-address-list=lab-pk-2 \
    action=add-src-to-address-list \
    address-list=lab-pk-allowed \
    address-list-timeout=5m \
    log=yes log-prefix="PK-Lab-GRANTED:" \
    comment="PK-Lab: Step 3 - Grant Access"

# Allow SSH ถ้าอยู่ใน allowed list
/ip firewall filter add \
    chain=input protocol=tcp dst-port=22 \
    src-address-list=lab-pk-allowed \
    action=accept \
    log=yes log-prefix="PK-Lab-SSH:" \
    comment="PK-Lab: Allow SSH"

# Block SSH ทั้งหมดที่เหลือ
/ip firewall filter add \
    chain=input protocol=tcp dst-port=22 \
    action=drop \
    log=yes log-prefix="PK-Lab-BLOCKED:" \
    comment="PK-Lab: Block SSH"
```

### Step 2: ทดสอบ Port Knocking

```bash
# ทดสอบจาก Linux client
# ติดตั้ง knockd client
# apt install knockd  # บน Debian/Ubuntu

# ทำ knock sequence
knock 203.0.113.10 7777
sleep 0.5
knock 203.0.113.10 8888
sleep 0.5
knock 203.0.113.10 9999
sleep 0.5

# ลอง SSH
ssh admin@203.0.113.10
```

### Step 3: Monitor และ Verify

```routeros
# ดู address lists
/ip firewall address-list print where list~"lab-pk"

# ดู logs
/log print where message~"PK-Lab"

# ดู firewall rule statistics
/ip firewall filter print stats where comment~"PK-Lab"
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Port Knocking Concept | วิธีการซ่อน service |
| Single Knock | 3-step TCP sequence |
| Multi-knock | 5-step mixed protocol |
| Time-based | เปิดปิดตามเวลา |
| VPN Integration | ซ่อน VPN port |
| Security Analysis | ข้อดี/ข้อเสีย, แก้ไขความเสี่ยง |
| Monitoring | Log และ monitor knock activities |
| Lab | Complete SSH protection setup |

---

[← Part 44: Failover](part-044-failover.md) | [Part 46: Dynamic Firewall →](part-046-dynamic-firewall.md)
