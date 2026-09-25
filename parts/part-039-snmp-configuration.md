# Part 39: SNMP Configuration ใน RouterOS

## บทนำ

SNMP (Simple Network Management Protocol) ใช้สำหรับ monitor และจัดการ network devices RouterOS รองรับ SNMP v1, v2c และ v3

---

## 39.1 SNMP v1/v2c Configuration

```routeros
# Enable SNMP
/snmp set enabled=yes

# Set system info
/snmp set \
    contact="admin@company.com" \
    location="Server Room Bangkok" \
    engine-id="" \
    trap-version=2 \
    trap-community=public

# Community strings
/snmp community print

# แก้ไข default community
/snmp community set [find name=public] \
    name=my-community \
    address=10.0.0.0/8 \
    read-access=yes \
    write-access=no \
    security=none

# เพิ่ม community ใหม่
/snmp community add \
    name=monitoring \
    address=192.168.1.100/32 \
    read-access=yes \
    write-access=no \
    security=none

# ดู SNMP config
/snmp print
/snmp community print
```

---

## 39.2 SNMP v3 Configuration

```routeros
# SNMP v3 มี authentication และ encryption

/snmp community add \
    name=v3-community \
    security=private \
    authentication-password=auth-pass-123 \
    encryption-password=enc-pass-456 \
    authentication-protocol=SHA1 \
    encryption-protocol=AES \
    read-access=yes

# ดู SNMP v3 config
/snmp community print detail

# Test SNMP v3 (จาก Linux)
# snmpwalk -v3 -u v3-community -l authPriv -a SHA -A auth-pass-123 -x AES -X enc-pass-456 192.168.1.1
```

---

## 39.3 Important OIDs

```routeros
# RouterOS MikroTik OIDs หลัก:
# System info
# .1.3.6.1.2.1.1.1.0 = sysDescr
# .1.3.6.1.2.1.1.3.0 = sysUpTime
# .1.3.6.1.2.1.1.5.0 = sysName

# Interface stats
# .1.3.6.1.2.1.2.2.1.2.x = ifDescr (interface name)
# .1.3.6.1.2.1.2.2.1.10.x = ifInOctets
# .1.3.6.1.2.1.2.2.1.16.x = ifOutOctets
# .1.3.6.1.2.1.31.1.1.1.6.x = ifHCInOctets (64-bit)
# .1.3.6.1.2.1.31.1.1.1.10.x = ifHCOutOctets (64-bit)

# MikroTik specific OIDs
# .1.3.6.1.4.1.14988.1.1.1.1.0 = CPU load
# .1.3.6.1.4.1.14988.1.1.1.2.0 = Free memory
# .1.3.6.1.4.1.14988.1.1.1.3.0 = Total memory
# .1.3.6.1.4.1.14988.1.1.1.4.0 = HDD free
# .1.3.6.1.4.1.14988.1.1.1.5.0 = HDD total
# .1.3.6.1.4.1.14988.1.1.1.7.0 = Voltage
# .1.3.6.1.4.1.14988.1.1.1.8.0 = Temperature

# Script ดู system info via OIDs
:put "System info via SNMP would show:"
:put "CPU: " . [/system resource get cpu-load] . "%"
:put "Memory Free: " . ([/system resource get free-memory] / 1048576) . " MB"
:put "Temperature: " . [/system health get temperature] . "C"
```

---

## 39.4 SNMP Traps

```routeros
# Configure SNMP traps
/snmp set \
    trap-target=10.0.0.200 \
    trap-version=2 \
    trap-community=public

# Trap generators
/snmp set trap-generators=temp-exception,interfaces,start-trap

# เปิด trap
/snmp set enabled=yes

# Test trap (ส่ง test trap)
/tool snmp-trap send \
    trap-type=warmStart \
    target=10.0.0.200 \
    community=public

# Script ส่ง custom trap
:local sendTrap do={
    :local message $1
    :local severity $2
    
    /tool snmp-trap send \
        trap-type=enterpriseSpecific \
        target=10.0.0.200 \
        community=public \
        value=($message . " [" . $severity . "]")
    
    :put "Trap sent: $message"
}

[$sendTrap "High CPU detected" "warning"]
```

---

## 39.5 Zabbix Integration

```routeros
# ตั้งค่า RouterOS สำหรับ Zabbix monitoring

# 1. SNMP สำหรับ Zabbix
/snmp set enabled=yes
/snmp community set [find name=public] address=0.0.0.0/0 read-access=yes

# 2. Firewall อนุญาต SNMP จาก Zabbix server
/ip firewall filter add \
    chain=input \
    protocol=udp \
    dst-port=161 \
    src-address=10.0.0.5 \
    action=accept \
    comment="Allow SNMP from Zabbix"

# 3. Zabbix template MikroTik ใช้ OIDs:
# Interface traffic: ifHCInOctets, ifHCOutOctets
# CPU: .1.3.6.1.4.1.14988.1.1.1.1.0
# Memory: .1.3.6.1.4.1.14988.1.1.1.2.0

# 4. Script ทดสอบ SNMP ใน RouterOS
/snmp community print detail
/snmp print

# Zabbix template items
:put "Zabbix SNMP Items for MikroTik:"
:put "CPU Load: OID .1.3.6.1.4.1.14988.1.1.1.1.0"
:put "Free RAM: OID .1.3.6.1.4.1.14988.1.1.1.2.0"
:put "Total RAM: OID .1.3.6.1.4.1.14988.1.1.1.3.0"
:put "Temperature: OID .1.3.6.1.4.1.14988.1.1.1.8.0"
:put "Uptime: OID .1.3.6.1.2.1.1.3.0"
```

---

## 39.6 SNMP Security Hardening

```routeros
# Hardening SNMP

# 1. ปิด default public community
/snmp community remove [find name=public]

# 2. สร้าง community ที่ตั้งชื่อซับซ้อน
/snmp community add \
    name=x9k2m7-monitoring \
    address=10.0.0.5/32 \
    read-access=yes \
    write-access=no

# 3. Block SNMP จาก outside
/ip firewall filter add \
    chain=input \
    protocol=udp \
    dst-port=161 \
    src-address=!10.0.0.0/8 \
    action=drop \
    comment="Block SNMP from outside"

# 4. ใช้ SNMP v3 เสมอสำหรับ sensitive data
/snmp community add \
    name=secure-v3 \
    security=private \
    authentication-password=Auth@Pass#123 \
    encryption-password=Enc@Pass#456 \
    authentication-protocol=SHA1 \
    encryption-protocol=AES

# 5. Log SNMP access
/system logging add topics=snmp action=memory
```

---

## Lab 39: Zabbix Monitoring Setup

### Solution

```routeros
# Zabbix SNMP Monitoring Setup
# ==============================

# Assumptions:
# Zabbix server IP: 10.0.0.5
# RouterOS IP: 192.168.1.1

# 1. Configure SNMP
/snmp set enabled=yes \
    contact="netadmin@company.com" \
    location="Main Office" \
    trap-target=10.0.0.5 \
    trap-version=2 \
    trap-community=zabbix-trap

# 2. Community สำหรับ Zabbix
/snmp community add \
    name=zabbix-read \
    address=10.0.0.5/32 \
    read-access=yes \
    write-access=no \
    security=none

# 3. Firewall rules
/ip firewall filter add \
    chain=input protocol=udp dst-port=161 \
    src-address=10.0.0.5/32 action=accept \
    comment="Zabbix SNMP" place-before=0

# 4. SNMP traps สำหรับ alerts
/snmp set trap-generators=temp-exception,interfaces

# 5. Verify
/snmp print
/snmp community print

# =============================
# Zabbix configuration (บน server):
# Host: 192.168.1.1
# SNMP version: 2
# Community: zabbix-read
# Template: Template Net MikroTik SNMPv2
# =============================

:put "Zabbix SNMP setup complete"
:put "Community: zabbix-read"
:put "Allowed from: 10.0.0.5"
:put "Traps to: 10.0.0.5"
```

---

## Summary ของ Part 39

| SNMP Version | Security | ใช้เมื่อ |
|-------------|---------|---------|
| v1 | None | Legacy only |
| v2c | Community string | Standard monitoring |
| v3 | Auth + Encrypt | Secure environments |

> **Warning:** ห้ามใช้ community "public" ใน production เพราะเป็น default ที่ทุกคนรู้

> **Best Practice:** ใช้ SNMP v3 เสมอ จำกัด source IP และ log SNMP access

> **Tip:** Zabbix มี MikroTik template สำเร็จรูป ไม่ต้องสร้าง OIDs เอง

---

[← Part 38: Network Monitoring](part-038-network-monitoring.md) | [Part 40: Netwatch →](part-040-netwatch.md)
