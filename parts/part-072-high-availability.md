# Part 72: High Availability

## สารบัญ
1. [HA Concepts](#concepts)
2. [Active-Passive Setup](#active-passive)
3. [Active-Active Setup](#active-active)
4. [State Synchronization](#state-sync)
5. [Configuration Sync](#config-sync)
6. [Monitoring in HA](#monitoring)
7. [Failover Testing](#failover-testing)
8. [Maintenance Procedures](#maintenance)
9. [Recovery Time Objectives](#rto-rpo)
10. [Lab: HA Router Pair](#lab)

---

## 1. HA Concepts {#concepts}

### HA Terminology

| Term | คำอธิบาย |
|------|---------|
| RTO | Recovery Time Objective - เวลา recover สูงสุดที่ยอมรับได้ |
| RPO | Recovery Point Objective - ข้อมูลที่ยอมสูญเสียได้ |
| MTBF | Mean Time Between Failures |
| MTTR | Mean Time To Repair |
| SLA | Service Level Agreement |

### HA Architecture Comparison

| Type | Failover Time | Complexity | Cost |
|------|--------------|------------|------|
| Active-Passive | 1-30 seconds | Low | Medium |
| Active-Active | < 1 second | High | High |
| ECMP Load Balance | < 1 second | Medium | Medium |

---

## 2. Active-Passive Setup {#active-passive}

```bash
# ============================================
# Active Node Configuration
# ============================================

# VRRP สำหรับ gateway redundancy
/interface vrrp
add interface=ether1 name=vrrp1 vrid=1 priority=200 \
    preemption-mode=yes on-backup="/system script run ha-standby" \
    on-master="/system script run ha-active" \
    authentication=ah password=VRRPpassword123

# Virtual IP address
/ip address
add address=192.168.1.1/24 interface=ether1 comment="Physical IP - Active"
add address=192.168.1.254/24 interface=vrrp1 comment="Virtual IP - Shared"

# HA Status script - Active
/system script
add name=ha-active source={
    :log info "HA: Became MASTER"
    # Enable WAN interface
    /interface enable ether1
    # Update monitoring
    /tool netwatch
    :foreach i in=[find] do={
        :if ([get $i comment]="ha-monitor") do={
            :set [get $i disabled] no
        }
    }
}

add name=ha-standby source={
    :log info "HA: Became BACKUP"
}

# Keepalive check script
/system script
add name=check-active-node source={
    :local pingResult [/ping 192.168.1.1 count=3 interface=ether3]
    :if ($pingResult = 0) do={
        :log warning "Active node not reachable - checking failover"
    }
}

/system scheduler
add name=ha-monitor interval=10s on-event=check-active-node \
    comment="HA monitoring"


# ============================================
# Passive Node Configuration
# ============================================

/interface vrrp
add interface=ether1 name=vrrp1 vrid=1 priority=100 \
    preemption-mode=yes

/ip address
add address=192.168.1.2/24 interface=ether1 comment="Physical IP - Passive"
add address=192.168.1.254/24 interface=vrrp1 comment="Virtual IP - Shared"
```

---

## 3. Active-Active Setup {#active-active}

```bash
# ============================================
# Active-Active with ECMP Load Balancing
# ============================================

# Router 1 (Primary)
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 distance=1 comment="ISP1-Primary"
add dst-address=0.0.0.0/0 gateway=198.51.100.1 distance=1 comment="ISP2-Secondary"
# Same distance = ECMP load balancing

# Firewall mangle สำหรับ per-connection load balancing
/ip firewall mangle
add chain=input in-interface=ether1 action=mark-connection \
    new-connection-mark=isp1-conn passthrough=yes comment="Mark ISP1 connections"

add chain=input in-interface=ether2 action=mark-connection \
    new-connection-mark=isp2-conn passthrough=yes comment="Mark ISP2 connections"

add chain=output connection-mark=isp1-conn action=mark-routing \
    new-routing-mark=to-isp1 passthrough=yes

add chain=output connection-mark=isp2-conn action=mark-routing \
    new-routing-mark=to-isp2 passthrough=yes

# Per-connection load balancing สำหรับ new connections
add chain=prerouting dst-address-type=!local in-interface=ether3-lan \
    action=mark-connection new-connection-mark=isp1-conn \
    connection-state=new nth=2,1 comment="New conn - ISP1"

add chain=prerouting dst-address-type=!local in-interface=ether3-lan \
    action=mark-connection new-connection-mark=isp2-conn \
    connection-state=new nth=2,2 comment="New conn - ISP2"

# Routing tables
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.1 routing-mark=to-isp1
add dst-address=0.0.0.0/0 gateway=198.51.100.1 routing-mark=to-isp2

# Source NAT per ISP
/ip firewall nat
add chain=srcnat out-interface=ether1 routing-mark=to-isp1 \
    action=masquerade comment="NAT for ISP1"
add chain=srcnat out-interface=ether2 routing-mark=to-isp2 \
    action=masquerade comment="NAT for ISP2"
```

---

## 4. State Synchronization {#state-sync}

```bash
# ============================================
# Connection Tracking Sync (RouterOS 7.x)
# ============================================

# สร้าง dedicated sync interface
/interface ethernet
set ether5 name=sync-link comment="HA Sync Link"

/ip address
add address=169.254.0.1/30 interface=sync-link comment="HA Sync IP - Active"
# Passive: 169.254.0.2/30

# Conntrack sync ผ่าน TCP
/ip firewall connection tracking
set sync-enable=yes sync-peer=169.254.0.2 sync-type=active

# ตรวจสอบ sync status
/ip firewall connection tracking print stats
```

---

## 5. Configuration Sync {#config-sync}

```bash
# ============================================
# Config Sync Script
# ============================================

/system script
add name=sync-config-to-backup source={
    :local backupIP "192.168.1.2"
    :local backupUser "admin"
    :local backupPass "backup-password"
    
    # Export config
    /export file=config-sync
    
    # Upload to backup router via FTP/SFTP
    :local result [/tool fetch address=$backupIP user=$backupUser \
        password=$backupPass src-path="config-sync.rsc" \
        dst-path="config-received.rsc" upload=yes]
    
    :if ($result = "finished") do={
        :log info "Config sync successful"
    } else={
        :log error "Config sync FAILED"
    }
}

# ตั้ง schedule sync ทุก 1 ชั่วโมง
/system scheduler
add name=sync-config interval=1h on-event=sync-config-to-backup
```

```python
# Python script สำหรับ config sync
import paramiko
import hashlib
import time
from typing import Dict, List
import logging

logger = logging.getLogger(__name__)


class ConfigSyncService:
    """ซิงค์ config ระหว่าง HA pair"""
    
    def __init__(self, primary: Dict, secondary: Dict):
        self.primary = primary
        self.secondary = secondary
    
    def export_config(self, host: str, username: str, password: str) -> str:
        """Export config จาก router"""
        ssh = paramiko.SSHClient()
        ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
        
        try:
            ssh.connect(host, username=username, password=password, timeout=30)
            stdin, stdout, stderr = ssh.exec_command('/export')
            config = stdout.read().decode('utf-8')
            return config
        finally:
            ssh.close()
    
    def compare_configs(self, config1: str, config2: str) -> bool:
        """เปรียบเทียบ configs"""
        hash1 = hashlib.sha256(config1.encode()).hexdigest()
        hash2 = hashlib.sha256(config2.encode()).hexdigest()
        return hash1 == hash2
    
    def sync_if_different(self) -> Dict:
        """Sync config ถ้าแตกต่างกัน"""
        primary_config = self.export_config(**self.primary)
        secondary_config = self.export_config(**self.secondary)
        
        if not self.compare_configs(primary_config, secondary_config):
            logger.warning("Config difference detected between HA pair!")
            return {
                "in_sync": False,
                "primary_hash": hashlib.sha256(primary_config.encode()).hexdigest()[:8],
                "secondary_hash": hashlib.sha256(secondary_config.encode()).hexdigest()[:8]
            }
        
        return {"in_sync": True}
```

---

## 6. Failover Testing {#failover-testing}

```bash
# ============================================
# Failover Test Procedures
# ============================================

# Test 1: Graceful failover
# 1. ลด VRRP priority บน Active node
/interface vrrp set vrrp1 priority=50

# 2. สังเกต VRRP state change
/interface vrrp print
# Backup node ควร become Master

# 3. ทดสอบ connectivity
/ping 8.8.8.8

# 4. Restore
/interface vrrp set vrrp1 priority=200

# Test 2: Hard failover (unplug cable simulation)
# ใช้ NetWatch เพื่อ detect
/tool netwatch
add host=192.168.1.1 interval=5s \
    up-script="/log info \"Primary up\"" \
    down-script="/log warning \"Primary DOWN - failover initiated\""

# Test 3: ตรวจสอบ failover time
/system script
add name=failover-test source={
    :local startTime [/system clock get time]
    # Simulate failure...
    :delay 5s
    :local endTime [/system clock get time]
    :log info ("Failover time: " . $startTime . " to " . $endTime)
}
```

---

## 7. Maintenance Procedures {#maintenance}

```bash
# ============================================
# Zero-downtime Maintenance Procedure
# ============================================

# Step 1: ยืนยันว่า backup node พร้อม
/interface vrrp print
# ต้องเห็น backup node ใน BACKUP state

# Step 2: Fail over ไป backup node gracefully
# ลด priority จนต่ำกว่า backup node
/interface vrrp set vrrp1 priority=50

# Wait 5 seconds สำหรับ failover
:delay 5s

# ยืนยัน backup กลายเป็น master
/interface vrrp print

# Step 3: Perform maintenance บน primary node
# ... ทำ maintenance ...

# Step 4: Restore primary
/interface vrrp set vrrp1 priority=200

# ถ้า preemption-mode=yes primary จะ reclaim master role
```

---

## 8. Lab: HA Router Pair {#lab}

### Lab Setup

```bash
# ============================================
# Lab: Active-Passive HA with VRRP
# ============================================

# Prerequisites:
# - 2x MikroTik routers (CCR หรือ RB)
# - 3 networks: WAN, LAN, Sync-link
# - ทั้งคู่ connected ถึง WAN และ LAN

# Active Router (R1)
# WAN: 203.0.113.2/29
# LAN: 192.168.1.1/24
# SYNC: 169.254.0.1/30
# VRRP VIP: 192.168.1.254/24

# Passive Router (R2)
# WAN: 203.0.113.3/29
# LAN: 192.168.1.2/24
# SYNC: 169.254.0.2/30
# VRRP VIP: 192.168.1.254/24 (shared)

# Test failover:
# 1. ping -t 8.8.8.8 จาก client
# 2. unplug R1's LAN cable หรือ disable VRRP priority
# 3. สังเกต ping drops
# 4. Measure failover time
```

### Expected Results

| Metric | Target | Acceptable |
|--------|--------|-----------|
| Failover Time | < 3 sec | < 10 sec |
| Packet Loss | < 5 packets | < 20 packets |
| Config Sync | < 5 sec lag | < 60 sec |
| VRRP Detection | < 3 sec | < 10 sec |

### Verification Checklist

- [ ] VRRP negotiates master/backup correctly
- [ ] Virtual IP accessible from client
- [ ] Failover occurs when primary fails
- [ ] Traffic resumes after failover
- [ ] Preemption restores primary when recovered
- [ ] Config sync working
- [ ] Monitoring alerts on failover event

> **Tip:** ทดสอบ failover ในช่วง maintenance window เสมอ

> **Warning:** VRRP authentication ต้องตรงกันทั้งสอง routers

---

## Summary

Part นี้ครอบคลุม:
- **Active-Passive HA** ด้วย VRRP
- **Active-Active** ด้วย ECMP
- **State synchronization** สำหรับ connection tracking
- **Config sync** ระหว่าง HA pair
- **Failover testing** procedures
- **Zero-downtime maintenance**

---

[← Part 71: Enterprise Design](part-071-enterprise-design.md) | [Part 73: VRRP →](part-073-vrrp.md)
