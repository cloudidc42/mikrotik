# Part 73: VRRP - Virtual Router Redundancy Protocol

## สารบัญ
1. [VRRP Protocol](#protocol)
2. [VRRP Configuration](#configuration)
3. [Multiple VRRP Groups](#multiple-groups)
4. [VRRP with Tracking](#tracking)
5. [VRRP Scripts](#scripts)
6. [Pre-emption](#preemption)
7. [Authentication](#authentication)
8. [Testing VRRP](#testing)
9. [VRRP Monitoring](#monitoring)
10. [Lab: Gateway Redundancy with VRRP](#lab)

---

## 1. VRRP Protocol {#protocol}

VRRP (Virtual Router Redundancy Protocol) เป็น protocol ที่ช่วยให้มี gateway redundancy โดยหลาย routers แชร์ virtual IP address เดียวกัน

### How VRRP Works

```
Client Gateway: 192.168.1.254 (Virtual IP)
                      │
          ┌───────────┴───────────┐
          │                       │
   [Router 1 - MASTER]     [Router 2 - BACKUP]
   Priority: 200            Priority: 100
   IP: 192.168.1.1          IP: 192.168.1.2
   Virtual IP: 192.168.1.254 Virtual IP: 192.168.1.254
          │                       │
          └───────────┬───────────┘
                      │
                  [LAN Switch]
                      │
                  [Clients]
```

### VRRP States

| State | Description |
|-------|-------------|
| Initialize | เริ่มต้น, รอ configuration |
| Master | Active, owns virtual IP |
| Backup | Standby, monitors Master |

### VRRP Advertisement
- Master ส่ง advertisement ทุก 1 วินาที (default)
- Backup รอ 3x advertisement interval ก่อน takeover
- Multicast address: 224.0.0.18
- Protocol: 112

---

## 2. VRRP Configuration {#configuration}

```bash
# ============================================
# Basic VRRP Setup
# ============================================

# Router 1 (Primary/Master)
/interface vrrp
add name=vrrp-gw1 interface=ether2-lan vrid=1 priority=200 \
    preemption-mode=yes interval=1s \
    comment="LAN Gateway VRRP"

/ip address
# Physical IP
add address=192.168.1.1/24 interface=ether2-lan comment="Physical"
# Virtual IP (will be owned by Master)
add address=192.168.1.254/24 interface=vrrp-gw1 comment="Virtual GW"


# Router 2 (Secondary/Backup)
/interface vrrp
add name=vrrp-gw1 interface=ether2-lan vrid=1 priority=100 \
    preemption-mode=yes interval=1s

/ip address
add address=192.168.1.2/24 interface=ether2-lan comment="Physical"
add address=192.168.1.254/24 interface=vrrp-gw1 comment="Virtual GW"

# ตรวจสอบ status
/interface vrrp print detail

# Expected output:
# Flags: X - disabled, R - running
#  0  R  name="vrrp-gw1" interface=ether2-lan vrid=1 priority=200
#        status=master virtual-address=192.168.1.254
```

---

## 3. Multiple VRRP Groups {#multiple-groups}

```bash
# ============================================
# Multiple VRRP Groups สำหรับ Load Sharing
# ============================================

# Router 1 - Master สำหรับ VLAN100, Backup สำหรับ VLAN200
/interface vrrp
# VLAN100 - R1 เป็น Master
add name=vrrp-vlan100 interface=vlan100 vrid=100 priority=200

# VLAN200 - R1 เป็น Backup
add name=vrrp-vlan200 interface=vlan200 vrid=200 priority=100

# VLAN300 - R1 เป็น Master
add name=vrrp-vlan300 interface=vlan300 vrid=300 priority=200


# Router 2 - Backup สำหรับ VLAN100, Master สำหรับ VLAN200
/interface vrrp
# VLAN100 - R2 เป็น Backup
add name=vrrp-vlan100 interface=vlan100 vrid=100 priority=100

# VLAN200 - R2 เป็น Master
add name=vrrp-vlan200 interface=vlan200 vrid=200 priority=200

# VLAN300 - R2 เป็น Backup
add name=vrrp-vlan300 interface=vlan300 vrid=300 priority=100


# Virtual IPs
/ip address
# Router 1
add address=10.100.0.1/24 interface=vlan100 comment="R1 Physical - VLAN100"
add address=10.100.0.254/24 interface=vrrp-vlan100 comment="VRRP VIP VLAN100"
add address=10.200.0.2/24 interface=vlan200 comment="R1 Physical - VLAN200"
add address=10.200.0.254/24 interface=vrrp-vlan200 comment="VRRP VIP VLAN200"

# DHCP pool ชี้ไปยัง Virtual IP
/ip dhcp-server network
add address=10.100.0.0/24 gateway=10.100.0.254 dns-server=8.8.8.8
add address=10.200.0.0/24 gateway=10.200.0.254 dns-server=8.8.8.8
```

---

## 4. VRRP with Tracking {#tracking}

```bash
# ============================================
# VRRP Tracking สำหรับ Intelligent Failover
# ============================================

# Script สำหรับ monitor WAN link และ adjust VRRP priority

/system script
add name=vrrp-track-wan source={
    :local primaryWAN "203.0.113.1"
    :local currentPriority [/interface vrrp get vrrp-gw1 priority]
    
    # Ping WAN gateway
    :local pingResult [/ping address=$primaryWAN count=3]
    
    :if ($pingResult > 0) do={
        # WAN up - ถ้า priority ลดลงไปแล้ว ให้ restore
        :if ($currentPriority < 200) do={
            /interface vrrp set vrrp-gw1 priority=200
            :log info "VRRP: WAN restored, priority=200"
        }
    } else={
        # WAN down - ลด priority
        :if ($currentPriority >= 200) do={
            /interface vrrp set vrrp-gw1 priority=50
            :log warning "VRRP: WAN down, priority=50 (failover to backup)"
        }
    }
}

# Schedule ทุก 5 วินาที
/system scheduler
add name=vrrp-wan-track interval=5s on-event=vrrp-track-wan


# VRRP Track Interface (RouterOS 7.x built-in tracking)
/interface vrrp
set vrrp-gw1 on-backup="" on-master=""
# Newer versions support:
# add ... tracked-interfaces=ether1-wan

# Advanced tracking script
/system script
add name=vrrp-multi-track source={
    :local wanUp 0
    :local ispUp 0
    :local score 0
    
    # Check WAN interface
    :local wanState [/interface get ether1-wan running]
    :if ($wanState = true) do={ :set wanUp 1 }
    
    # Check ISP gateway
    :local ispPing [/ping address=203.0.113.1 count=2]
    :if ($ispPing > 0) do={ :set ispUp 1 }
    
    # Check internet
    :local internetPing [/ping address=8.8.8.8 count=2]
    
    # Calculate score
    :set score ($wanUp * 100 + $ispUp * 100 + $internetPing * 50)
    
    :if ($score >= 200) do={
        /interface vrrp set vrrp-gw1 priority=200
    } else-if ($score >= 100) do={
        /interface vrrp set vrrp-gw1 priority=150
    } else={
        /interface vrrp set vrrp-gw1 priority=50
        :log warning "VRRP: Low score, failover initiated"
    }
}
```

---

## 5. VRRP Scripts {#scripts}

```bash
# ============================================
# VRRP Event Scripts
# ============================================

/system script
# Script รันเมื่อ become Master
add name=vrrp-on-master source={
    :log info "VRRP: Became MASTER"
    
    # Enable services ที่ต้องการเฉพาะบน Master
    /ip dhcp-server enable [find name="dhcp-lan"]
    
    # Update DNS record (ถ้ามี dynamic DNS)
    # ...
    
    # Send alert
    /tool e-mail send to="noc@company.com" \
        subject="VRRP Master Transition" \
        body="Router became VRRP Master at [/system clock get time]"
    
    # Log to syslog
    /log info "VRRP transition: This router is now MASTER"
}

# Script รันเมื่อ become Backup
add name=vrrp-on-backup source={
    :log info "VRRP: Became BACKUP"
    
    # Disable services บน Backup node
    /ip dhcp-server disable [find name="dhcp-lan"]
    
    :log info "VRRP transition: This router is now BACKUP"
}

# ใส่ scripts ใน VRRP interface
/interface vrrp
set vrrp-gw1 on-master=vrrp-on-master on-backup=vrrp-on-backup
```

---

## 6. Testing VRRP {#testing}

```bash
# ============================================
# VRRP Testing Procedures
# ============================================

# Test 1: Basic VRRP negotiation
# ตรวจสอบ Master election
/interface vrrp print detail
# Expected: vrid=1 status=master priority=200

# Test 2: Failover test
# Method A: ลด priority บน Master
/interface vrrp set vrrp-gw1 priority=50
# Expected: Backup กลายเป็น Master

# Method B: Disable VRRP interface
/interface vrrp disable vrrp-gw1
# Expected: Backup กลายเป็น Master ภายใน 3 วินาที

# Test 3: Preemption test
# Restore priority
/interface vrrp set vrrp-gw1 priority=200
/interface vrrp enable vrrp-gw1
# Expected: Router กลับเป็น Master เพราะ priority สูงกว่า

# Test 4: Packet capture เพื่อ verify VRRP
/tool sniffer quick filter-ip-protocol=vrrp
# Expected: เห็น VRRP advertisement packets ทุก 1 วินาที

# Test 5: ทดสอบจาก client
# ระหว่างทำ failover, ping จาก client ไม่ควรหยุดนานเกิน 3 seconds
# ping -t 8.8.8.8 (Windows)
# ping -i 0.5 8.8.8.8 (Linux)
```

---

## 7. VRRP Monitoring {#monitoring}

```python
# Python monitoring script
import subprocess
import json
import time
from datetime import datetime

def get_vrrp_status(router_host: str, username: str, password: str) -> dict:
    """ดึง VRRP status จาก router"""
    try:
        import librouteros
        conn = librouteros.connect(
            host=router_host,
            username=username,
            password=password
        )
        
        vrrp_interfaces = list(conn('/interface/vrrp/print'))
        
        status = []
        for vrrp in vrrp_interfaces:
            vrrp_data = dict(vrrp)
            status.append({
                "name": vrrp_data.get("name"),
                "vrid": vrrp_data.get("vrid"),
                "priority": vrrp_data.get("priority"),
                "status": vrrp_data.get("status"),  # master/backup
                "interface": vrrp_data.get("interface"),
                "virtual_address": vrrp_data.get("virtual-address")
            })
        
        conn.close()
        return {"router": router_host, "vrrp": status, "timestamp": datetime.utcnow().isoformat()}
    
    except Exception as e:
        return {"router": router_host, "error": str(e)}


def monitor_vrrp_pair(r1_config: dict, r2_config: dict):
    """Monitor VRRP pair"""
    while True:
        r1_status = get_vrrp_status(**r1_config)
        r2_status = get_vrrp_status(**r2_config)
        
        # ตรวจสอบว่าไม่มี split-brain (ทั้งคู่ master)
        r1_masters = [v for v in r1_status.get("vrrp", []) if v.get("status") == "master"]
        r2_masters = [v for v in r2_status.get("vrrp", []) if v.get("status") == "master"]
        
        for r1_vrrp in r1_masters:
            for r2_vrrp in r2_masters:
                if r1_vrrp.get("vrid") == r2_vrrp.get("vrid"):
                    print(f"WARNING: Split-brain detected for VRID {r1_vrrp['vrid']}!")
        
        time.sleep(30)


if __name__ == "__main__":
    r1 = {"router_host": "192.168.1.1", "username": "admin", "password": "pass"}
    r2 = {"router_host": "192.168.1.2", "username": "admin", "password": "pass"}
    monitor_vrrp_pair(r1, r2)
```

---

## 8. Lab: Gateway Redundancy with VRRP {#lab}

### Lab Objectives
1. ตั้งค่า VRRP บน router คู่
2. ทดสอบ failover
3. ตั้งค่า tracking สำหรับ WAN monitoring
4. Implement VRRP scripts สำหรับ event handling
5. Monitor VRRP state

### Lab Steps

```bash
# Step 1: Setup VRRP บน R1
/interface vrrp add name=vrrp1 interface=ether2 vrid=1 priority=200 preemption-mode=yes
/ip address add address=192.168.1.254/24 interface=vrrp1

# Step 2: Setup VRRP บน R2
/interface vrrp add name=vrrp1 interface=ether2 vrid=1 priority=100 preemption-mode=yes
/ip address add address=192.168.1.254/24 interface=vrrp1

# Step 3: Verify
/interface vrrp print detail

# Step 4: Test failover
/interface vrrp set vrrp1 priority=50  # บน R1
# R2 ควร become Master

# Step 5: Restore
/interface vrrp set vrrp1 priority=200  # บน R1
# R1 ควร reclaim Master (preemption)
```

### Verification Checklist

- [ ] VRRP master/backup negotiation ถูกต้อง
- [ ] Virtual IP accessible จาก client
- [ ] Failover เกิดภายใน 3 วินาที
- [ ] Preemption works หลัง restore
- [ ] Tracking scripts ทำงาน
- [ ] VRRP authentication configured
- [ ] Event scripts ทำงานทั้ง on-master/on-backup

> **Tip:** ใช้ Wireshark packet capture เพื่อ verify VRRP advertisements

> **Warning:** VRID ต้องไม่ซ้ำกันในเครือข่ายเดียวกัน และ interval ต้องตรงกันทั้งสอง routers

---

## Summary

Part นี้ครอบคลุม:
- **VRRP fundamentals** และ state machine
- **Basic VRRP configuration** บน MikroTik
- **Multiple VRRP groups** สำหรับ load sharing
- **VRRP tracking** สำหรับ intelligent failover
- **Event scripts** สำหรับ automation
- **Testing** และ **monitoring** procedures

---

[← Part 72: High Availability](part-072-high-availability.md) | [Part 74: MPLS VPN →](part-074-mpls-vpn.md)
