# Part 89: Disaster Recovery

## สารบัญ
1. [DR Planning](#planning)
2. [RTO/RPO Objectives](#rto-rpo)
3. [Backup Strategies](#backup)
4. [DR Procedures](#procedures)
5. [DR Drill](#drill)
6. [Lab: DR Exercise](#lab)

---

## 1. DR Planning {#planning}

### DR Categories

| Category | Description | Example |
|----------|-------------|---------|
| Device Failure | Hardware breakdown | Replace router |
| Site Failure | Data center outage | Failover to DR site |
| Network Failure | ISP/link failure | Failover ISP |
| Software Failure | OS/firmware issue | Rollback firmware |
| Ransomware/Attack | Security incident | Restore from backup |

---

## 2. RTO/RPO Objectives {#rto-rpo}

```
RTO (Recovery Time Objective) = เวลาสูงสุดที่ยอมให้ service หยุด
RPO (Recovery Point Objective) = ข้อมูลล่าสุดที่ยอมสูญเสีย

ตัวอย่าง ISP:
┌─────────────────┬───────────┬────────────┐
│ Service         │ RTO       │ RPO        │
├─────────────────┼───────────┼────────────┤
│ Core Routing    │ 15 min    │ 0 (HA)     │
│ Edge Routing    │ 30 min    │ 1 hour     │
│ Customer Portal │ 2 hours   │ 1 hour     │
│ Billing System  │ 4 hours   │ 15 min     │
│ NOC Dashboard   │ 1 hour    │ 5 min      │
└─────────────────┴───────────┴────────────┘
```

---

## 3. Backup Strategies {#backup}

```bash
# ============================================
# MikroTik Backup Procedures
# ============================================

# Method 1: RouterOS Binary Backup
/system backup save name=router-backup password=BackupPass@2024!
# ดาวน์โหลดผ่าน FTP/Winbox

# Method 2: Export (Text/Rsc format)
/export file=router-config-export
# ได้ .rsc file ที่สามารถ import ได้

# Method 3: Export specific sections
/ip address export file=ip-addresses
/ip firewall filter export file=firewall-rules
/routing bgp export file=bgp-config

# Method 4: Automatic backup script
/system script
add name=auto-backup source={
    :local date [/system clock get date]
    :local time [/system clock get time]
    :local backupName ("backup-" . $date . "-" . $time)
    :set backupName [:pick $backupName 0 20]
    
    # Binary backup
    /system backup save name=$backupName password=BackupPass@2024!
    
    # Export
    /export file=($backupName . ".rsc")
    
    :log info ("Backup created: " . $backupName)
}

/system scheduler
add name=daily-backup interval=1d on-event=auto-backup start-time=02:00:00
```

```python
# Python backup automation
import librouteros
import paramiko
import os
from datetime import datetime
from pathlib import Path

class BackupManager:
    def __init__(self, backup_dir: str = "./backups"):
        self.backup_dir = Path(backup_dir)
        self.backup_dir.mkdir(exist_ok=True)
    
    def backup_router(self, host: str, username: str, password: str,
                       ssh_port: int = 22) -> dict:
        """Backup configuration จาก router"""
        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        hostname_dir = self.backup_dir / host
        hostname_dir.mkdir(exist_ok=True)
        
        results = {}
        
        # Export configuration via RouterOS API
        try:
            conn = librouteros.connect(host=host, username=username, password=password)
            
            # Trigger export
            export = conn('/export')
            
            export_file = hostname_dir / f"config_{timestamp}.rsc"
            export_file.write_text(str(export))
            results["export"] = str(export_file)
            
            # System info
            resource = list(conn('/system/resource/print'))[0]
            identity = list(conn('/system/identity/print'))[0]
            
            results["hostname"] = dict(identity).get("name")
            results["version"] = dict(resource).get("version")
            
            conn.close()
        except Exception as e:
            results["error"] = str(e)
        
        # Download .backup file via SSH/SCP
        try:
            ssh = paramiko.SSHClient()
            ssh.set_missing_host_key_policy(paramiko.AutoAddPolicy())
            ssh.connect(host, port=ssh_port, username=username, password=password)
            
            # Trigger backup creation
            stdin, stdout, stderr = ssh.exec_command(
                f"/system backup save name=backup-{timestamp}"
            )
            stdout.read()
            
            # Download via SFTP
            sftp = ssh.open_sftp()
            remote_file = f"/backup-{timestamp}.backup"
            local_file = hostname_dir / f"backup_{timestamp}.backup"
            
            sftp.get(remote_file, str(local_file))
            sftp.close()
            ssh.close()
            
            results["backup_file"] = str(local_file)
        except Exception as e:
            results["sftp_error"] = str(e)
        
        return results
    
    def backup_all(self, routers: list) -> dict:
        """Backup ทุก routers"""
        all_results = {}
        for router in routers:
            print(f"Backing up {router['host']}...")
            result = self.backup_router(**router)
            all_results[router['host']] = result
        return all_results
    
    def upload_to_s3(self, local_file: str, bucket: str, key: str):
        """Upload backup ไป AWS S3"""
        import boto3
        s3 = boto3.client('s3')
        s3.upload_file(local_file, bucket, key)
        print(f"Uploaded {local_file} to s3://{bucket}/{key}")
```

---

## 4. DR Procedures {#procedures}

```bash
# ============================================
# DR Runbook - Core Router Failure
# ============================================

# DR Runbook: CORE-ROUTER-FAILURE
# Severity: Critical
# RTO: 15 minutes
# Last Updated: 2024-01-01

# 1. DETECTION (0-2 minutes)
# - Monitoring alert: Core router unreachable
# - Verify: /ping core-01-mgmt
# - Check physical connections

# 2. ASSESSMENT (2-5 minutes)
# - Is hardware dead or software issue?
# - Is DR router available?
# - How many customers affected?

# 3. FAILOVER (5-10 minutes)
# Scenario A: Software issue → Reboot/rollback
/system reboot

# Scenario B: Hardware failure → Activate DR router
# - Power on DR router (pre-configured)
# - Apply saved configuration
/system backup load name=latest-backup password=BackupPass@2024!

# Verify DR router config
/ip address print
/ip route print
/routing bgp peer print

# 4. VERIFICATION (10-12 minutes)
/ping 8.8.8.8
/routing bgp peer print  # Check BGP established
/ip firewall connection print count-only  # Check customer traffic

# 5. COMMUNICATION (12-15 minutes)
# - Notify NOC team
# - Update status page
# - Log incident

# 6. POST-INCIDENT (after restoration)
# - Root cause analysis
# - Update runbook
# - Preventive measures
```

---

## 5. DR Drill {#drill}

```python
# DR Drill automation script
import time
import logging
from dataclasses import dataclass
from typing import List, Callable
from datetime import datetime

logger = logging.getLogger(__name__)

@dataclass
class DrillStep:
    name: str
    action: Callable
    timeout_seconds: int
    rollback: Callable = None

class DRDrill:
    """Automated DR drill execution"""
    
    def __init__(self, name: str):
        self.name = name
        self.steps: List[DrillStep] = []
        self.results = []
        self.start_time = None
    
    def add_step(self, step: DrillStep):
        self.steps.append(step)
    
    def run(self) -> dict:
        self.start_time = datetime.utcnow()
        logger.info(f"Starting DR Drill: {self.name}")
        
        passed = 0
        failed = 0
        
        for step in self.steps:
            step_start = time.time()
            logger.info(f"Executing: {step.name}")
            
            try:
                result = step.action()
                elapsed = time.time() - step_start
                
                if elapsed > step.timeout_seconds:
                    status = "TIMEOUT"
                    failed += 1
                else:
                    status = "PASS"
                    passed += 1
                
                self.results.append({
                    "step": step.name,
                    "status": status,
                    "elapsed_seconds": round(elapsed, 2),
                    "timeout": step.timeout_seconds,
                    "result": result
                })
                
                logger.info(f"  {status}: {elapsed:.1f}s (target: {step.timeout_seconds}s)")
            
            except Exception as e:
                failed += 1
                self.results.append({
                    "step": step.name,
                    "status": "FAIL",
                    "error": str(e)
                })
                logger.error(f"  FAIL: {e}")
                
                if step.rollback:
                    logger.info("  Executing rollback...")
                    step.rollback()
        
        total_time = (datetime.utcnow() - self.start_time).total_seconds()
        
        return {
            "drill": self.name,
            "passed": passed,
            "failed": failed,
            "total_seconds": round(total_time, 2),
            "steps": self.results,
            "success": failed == 0
        }


# Example drill สำหรับ ISP edge router
def create_edge_router_dr_drill():
    from netmiko import ConnectHandler
    
    primary = {
        "device_type": "mikrotik_routeros",
        "host": "10.0.0.1",
        "username": "admin",
        "password": "admin123"
    }
    
    secondary = {
        "device_type": "mikrotik_routeros",
        "host": "10.0.0.2",
        "username": "admin",
        "password": "admin123"
    }
    
    drill = DRDrill("Edge Router Failover")
    
    def simulate_primary_failure():
        with ConnectHandler(**primary) as conn:
            conn.send_command("/interface vrrp set vrrp1 priority=50")
        return "VRRP priority reduced"
    
    def verify_secondary_becomes_master():
        import time
        time.sleep(3)
        with ConnectHandler(**secondary) as conn:
            output = conn.send_command("/interface vrrp print")
            assert "master" in output.lower()
        return "Secondary is master"
    
    def verify_connectivity():
        with ConnectHandler(**secondary) as conn:
            output = conn.send_command("/ping 8.8.8.8 count=3")
            assert "3 received" in output
        return "Internet connectivity OK"
    
    def restore_primary():
        with ConnectHandler(**primary) as conn:
            conn.send_command("/interface vrrp set vrrp1 priority=200")
    
    drill.add_step(DrillStep("Simulate Primary Failure", simulate_primary_failure, 30, restore_primary))
    drill.add_step(DrillStep("Verify Secondary Becomes Master", verify_secondary_becomes_master, 10))
    drill.add_step(DrillStep("Verify Internet Connectivity", verify_connectivity, 15))
    drill.add_step(DrillStep("Restore Primary", restore_primary, 30))
    
    return drill
```

---

## 6. Lab: DR Exercise {#lab}

### Lab Steps

```bash
# Pre-drill preparation
# 1. ตรวจสอบ backup ล่าสุด
ls -la backups/
# 2. ตรวจสอบ DR router พร้อม
/interface vrrp print  # ควรเห็น backup state

# Execute drill
python dr_drill.py --scenario edge-router-failure

# Post-drill checks
# 1. ตรวจสอบ customer traffic กลับมาปกติ
/ip firewall connection print count-only

# 2. ตรวจสอบ BGP sessions
/routing bgp peer print

# 3. บันทึก drill results
cat dr_drill_results_$(date +%Y%m%d).json
```

### DR Drill Checklist

- [ ] Backup files พร้อมและทดสอบ restore แล้ว
- [ ] DR router pre-configured
- [ ] Failover time ตาม RTO
- [ ] Customer traffic restored หลัง failover
- [ ] BGP sessions re-established
- [ ] Monitoring alerts ถูก
- [ ] DR runbook updated

> **Tip:** ทำ DR drill อย่างน้อย quarterly (ทุก 3 เดือน)

> **Warning:** แจ้ง customers ก่อนทำ planned DR drill เสมอ

---

## Summary

Part นี้ครอบคลุม:
- **DR planning** และ categories
- **RTO/RPO** objectives
- **Backup strategies** แบบ automated
- **DR procedures** แบบ runbook
- **DR drill** ด้วย automation

---

[← Part 88: CI/CD](part-088-cicd.md) | [Part 90: Performance →](part-090-performance.md)
