# Part 68: Network Inventory System

## สารบัญ
1. [Network Inventory Concepts](#concepts)
2. [Auto-Discovery](#discovery)
3. [Device Tracking](#tracking)
4. [Configuration Management](#config-mgmt)
5. [Change Management](#changes)
6. [Asset Lifecycle](#lifecycle)
7. [Integration with Monitoring](#monitoring)
8. [Reporting](#reporting)
9. [API Integration](#api)
10. [Lab: Network Inventory System](#lab)

---

## 1. Network Inventory Concepts {#concepts}

Network Inventory System (NIS) คือระบบที่รวบรวมและจัดการข้อมูลของอุปกรณ์เครือข่ายทั้งหมด

### ประโยชน์ของ Network Inventory

| ประโยชน์ | คำอธิบาย |
|---------|---------|
| Asset tracking | รู้ว่ามีอุปกรณ์อะไรบ้างในเครือข่าย |
| Configuration backup | เก็บ config สำรองก่อนเปลี่ยนแปลง |
| Change audit | ติดตามการเปลี่ยนแปลง config |
| Capacity planning | วางแผนการขยายระบบ |
| Compliance | ตรวจสอบ security compliance |
| Troubleshooting | ข้อมูลเพื่อ debug ปัญหา |

### Data Model

```sql
-- Device Inventory Schema
CREATE TABLE device_inventory (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    hostname VARCHAR(100) NOT NULL,
    ip_address INET NOT NULL,
    device_type VARCHAR(50),  -- router, switch, ap, firewall
    vendor VARCHAR(50),       -- mikrotik, cisco, juniper
    model VARCHAR(100),
    serial_number VARCHAR(100),
    firmware_version VARCHAR(50),
    location VARCHAR(200),
    site VARCHAR(100),
    rack VARCHAR(50),
    rack_unit INT,
    purchase_date DATE,
    warranty_expiry DATE,
    status VARCHAR(20) DEFAULT 'active'
        CHECK (status IN ('active', 'inactive', 'maintenance', 'decommissioned')),
    last_seen TIMESTAMP WITH TIME ZONE,
    last_discovery TIMESTAMP WITH TIME ZONE,
    snmp_community VARCHAR(100),
    ssh_credentials JSONB DEFAULT '{}',
    custom_fields JSONB DEFAULT '{}',
    tags TEXT[],
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE DEFAULT NOW()
);

CREATE TABLE config_backups (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    device_id UUID NOT NULL REFERENCES device_inventory(id),
    config_type VARCHAR(50) DEFAULT 'running',
    config_content TEXT NOT NULL,
    config_hash VARCHAR(64) NOT NULL,  -- SHA256
    backup_method VARCHAR(20),         -- api, ssh, snmp
    is_changed BOOLEAN DEFAULT FALSE,  -- เปลี่ยนจาก backup ก่อน
    backup_by VARCHAR(100),
    backup_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    notes TEXT
);

CREATE TABLE config_changes (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    device_id UUID NOT NULL REFERENCES device_inventory(id),
    change_type VARCHAR(30),  -- config_changed, firmware_update, interface_down
    old_value TEXT,
    new_value TEXT,
    diff_content TEXT,
    detected_at TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    detected_by VARCHAR(50),  -- system, user, api
    approved BOOLEAN,
    approved_by VARCHAR(100),
    change_request_id VARCHAR(50)
);

CREATE INDEX idx_inventory_ip ON device_inventory(ip_address);
CREATE INDEX idx_inventory_status ON device_inventory(status);
CREATE INDEX idx_config_backups_device ON config_backups(device_id, backup_at DESC);
CREATE INDEX idx_config_changes_device ON config_changes(device_id, detected_at DESC);
```

---

## 2. Auto-Discovery {#discovery}

```python
# app/services/inventory/discovery_service.py
import ipaddress
import asyncio
import socket
from typing import List, Dict, Optional
import logging
from concurrent.futures import ThreadPoolExecutor

logger = logging.getLogger(__name__)


class NetworkDiscovery:
    """Auto-discovery ของ network devices"""
    
    def __init__(self, max_workers: int = 50):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
    
    async def discover_subnet(
        self,
        subnet: str,
        methods: List[str] = None
    ) -> List[Dict]:
        """Discover devices ใน subnet"""
        if methods is None:
            methods = ['ping', 'snmp', 'mikrotik_api']
        
        network = ipaddress.ip_network(subnet, strict=False)
        hosts = list(network.hosts())
        
        logger.info(f"Starting discovery for {subnet} ({len(hosts)} hosts)")
        
        # Run discovery ใน parallel
        loop = asyncio.get_event_loop()
        tasks = [
            loop.run_in_executor(
                self.executor,
                self._probe_host,
                str(host),
                methods
            )
            for host in hosts
        ]
        
        results = await asyncio.gather(*tasks)
        discovered = [r for r in results if r is not None]
        
        logger.info(f"Discovered {len(discovered)} devices in {subnet}")
        return discovered
    
    def _probe_host(self, ip: str, methods: List[str]) -> Optional[Dict]:
        """Probe single host"""
        device_info = {"ip_address": ip, "methods_used": []}
        found = False
        
        # Ping test
        if 'ping' in methods:
            if self._ping_host(ip):
                found = True
                device_info["is_alive"] = True
                device_info["methods_used"].append("ping")
        
        if not found:
            return None
        
        # MikroTik API probe
        if 'mikrotik_api' in methods:
            mikrotik_info = self._probe_mikrotik_api(ip)
            if mikrotik_info:
                device_info.update(mikrotik_info)
                device_info["vendor"] = "MikroTik"
                device_info["methods_used"].append("mikrotik_api")
        
        # SNMP probe
        if 'snmp' in methods:
            snmp_info = self._probe_snmp(ip)
            if snmp_info:
                device_info.update(snmp_info)
                device_info["methods_used"].append("snmp")
        
        # DNS resolution
        hostname = self._resolve_hostname(ip)
        if hostname:
            device_info["hostname"] = hostname
        
        return device_info
    
    def _ping_host(self, ip: str, timeout: float = 1.0) -> bool:
        """Ping test"""
        import subprocess
        import platform
        
        param = '-n' if platform.system().lower() == 'windows' else '-c'
        command = ['ping', param, '1', '-W', str(int(timeout * 1000)), ip]
        
        try:
            return subprocess.run(
                command,
                capture_output=True,
                timeout=timeout + 1
            ).returncode == 0
        except Exception:
            return False
    
    def _probe_mikrotik_api(self, ip: str, timeout: int = 3) -> Optional[Dict]:
        """Probe MikroTik API"""
        try:
            import librouteros
            conn = librouteros.connect(
                host=ip,
                username='admin',
                password='',
                timeout=timeout
            )
            
            resources = {}
            for item in conn('/system/resource/print'):
                resources = dict(item)
                break
            
            identity = {}
            for item in conn('/system/identity/print'):
                identity = dict(item)
                break
            
            conn.close()
            
            return {
                "hostname": identity.get('name'),
                "model": resources.get('board-name'),
                "firmware_version": resources.get('version'),
                "device_type": "router",
                "vendor": "MikroTik"
            }
        except Exception:
            return None
    
    def _probe_snmp(self, ip: str, community: str = 'public') -> Optional[Dict]:
        """SNMP probe"""
        try:
            from pysnmp.hlapi import (
                getCmd, SnmpEngine, CommunityData, UdpTransportTarget,
                ContextData, ObjectType, ObjectIdentity
            )
            
            # System OIDs
            oids = {
                'sysDescr': '1.3.6.1.2.1.1.1.0',
                'sysName': '1.3.6.1.2.1.1.5.0',
                'sysContact': '1.3.6.1.2.1.1.4.0',
                'sysLocation': '1.3.6.1.2.1.1.6.0',
            }
            
            result = {}
            for name, oid in oids.items():
                error_indication, error_status, error_index, var_binds = next(
                    getCmd(
                        SnmpEngine(),
                        CommunityData(community),
                        UdpTransportTarget((ip, 161), timeout=2, retries=0),
                        ContextData(),
                        ObjectType(ObjectIdentity(oid))
                    )
                )
                
                if not error_indication and not error_status:
                    result[name] = str(var_binds[0][1])
            
            return result if result else None
        except ImportError:
            logger.debug("pysnmp not installed")
            return None
        except Exception:
            return None
    
    def _resolve_hostname(self, ip: str) -> Optional[str]:
        """Reverse DNS lookup"""
        try:
            return socket.gethostbyaddr(ip)[0]
        except Exception:
            return None
```

---

## 3. Configuration Management {#config-mgmt}

```python
# app/services/inventory/config_manager.py
import hashlib
import difflib
from datetime import datetime
from typing import Optional, Dict, List
import logging
from sqlalchemy.orm import Session
from sqlalchemy import text

from app.services.mikrotik import RouterManager

logger = logging.getLogger(__name__)


class ConfigurationManager:
    """จัดการ configuration backups"""
    
    def __init__(self, db: Session, router_manager: RouterManager):
        self.db = db
        self.router_manager = router_manager
    
    def backup_router_config(self, router_id: str, device_id: str) -> Dict:
        """สำรอง configuration ของ router"""
        try:
            with self.router_manager.get_connection(router_id) as conn:
                # Export configuration แบบ full
                config_lines = []
                
                # System identity
                for item in conn.run_command('/system/identity/print'):
                    config_lines.append(f"# System: {item.get('name', 'unknown')}")
                
                # Interfaces
                config_lines.append("\n# Interfaces")
                for iface in conn.run_command('/interface/print'):
                    config_lines.append(str(dict(iface)))
                
                # IP Addresses
                config_lines.append("\n# IP Addresses")
                for addr in conn.run_command('/ip/address/print'):
                    config_lines.append(str(dict(addr)))
                
                # Routes
                config_lines.append("\n# Routes")
                for route in conn.run_command('/ip/route/print'):
                    config_lines.append(str(dict(route)))
                
                # Firewall
                config_lines.append("\n# Firewall Filter")
                for rule in conn.run_command('/ip/firewall/filter/print'):
                    config_lines.append(str(dict(rule)))
                
                config_content = '\n'.join(config_lines)
                config_hash = hashlib.sha256(config_content.encode()).hexdigest()
                
                # ตรวจสอบว่า config เปลี่ยนไปจาก backup ล่าสุด
                last_backup = self.db.execute(text("""
                    SELECT config_hash FROM config_backups
                    WHERE device_id = :device_id::UUID
                    ORDER BY backup_at DESC
                    LIMIT 1
                """), {"device_id": device_id}).fetchone()
                
                is_changed = not last_backup or last_backup.config_hash != config_hash
                
                # บันทึก backup
                result = self.db.execute(text("""
                    INSERT INTO config_backups 
                        (device_id, config_content, config_hash, is_changed, backup_method)
                    VALUES 
                        (:device_id::UUID, :content, :hash, :changed, 'api')
                    RETURNING id
                """), {
                    "device_id": device_id,
                    "content": config_content,
                    "hash": config_hash,
                    "changed": is_changed
                })
                
                backup_id = result.fetchone().id
                
                if is_changed:
                    self._record_config_change(device_id, last_backup, config_content)
                
                self.db.commit()
                
                return {
                    "backup_id": str(backup_id),
                    "is_changed": is_changed,
                    "config_hash": config_hash,
                    "backed_up_at": datetime.utcnow().isoformat()
                }
                
        except Exception as e:
            self.db.rollback()
            logger.error(f"Config backup failed for {router_id}: {e}")
            raise
    
    def _record_config_change(self, device_id: str, last_backup, new_config: str):
        """บันทึก config change"""
        old_config = ""
        if last_backup:
            old_result = self.db.execute(text("""
                SELECT config_content FROM config_backups WHERE config_hash = :hash
            """), {"hash": last_backup.config_hash}).fetchone()
            if old_result:
                old_config = old_result.config_content
        
        # สร้าง diff
        diff = '\n'.join(difflib.unified_diff(
            old_config.splitlines(),
            new_config.splitlines(),
            lineterm='',
            n=3
        ))
        
        self.db.execute(text("""
            INSERT INTO config_changes 
                (device_id, change_type, old_value, new_value, diff_content)
            VALUES 
                (:device_id::UUID, 'config_changed', :old, :new, :diff)
        """), {
            "device_id": device_id,
            "old": old_config[:10000] if old_config else "",
            "new": new_config[:10000],
            "diff": diff[:50000]
        })
    
    def compare_configs(self, backup_id_1: str, backup_id_2: str) -> str:
        """เปรียบเทียบ configs สอง version"""
        configs = self.db.execute(text("""
            SELECT id, config_content, backup_at
            FROM config_backups
            WHERE id IN (:id1::UUID, :id2::UUID)
            ORDER BY backup_at
        """), {"id1": backup_id_1, "id2": backup_id_2}).fetchall()
        
        if len(configs) < 2:
            return ""
        
        diff = '\n'.join(difflib.unified_diff(
            configs[0].config_content.splitlines(),
            configs[1].config_content.splitlines(),
            fromfile=f"Version {configs[0].backup_at}",
            tofile=f"Version {configs[1].backup_at}",
            lineterm='',
            n=3
        ))
        
        return diff
    
    def bulk_backup_all_routers(self) -> Dict:
        """สำรอง config ทุก router"""
        routers = self.db.execute(text("""
            SELECT r.id as router_id, d.id as device_id
            FROM routers r
            JOIN device_inventory d ON d.ip_address = r.host
            WHERE r.is_active = true
        """)).fetchall()
        
        results = {"success": 0, "failed": 0, "changed": 0}
        
        for router in routers:
            try:
                result = self.backup_router_config(
                    str(router.router_id),
                    str(router.device_id)
                )
                results["success"] += 1
                if result.get("is_changed"):
                    results["changed"] += 1
            except Exception as e:
                results["failed"] += 1
                logger.error(f"Backup failed: {e}")
        
        return results
```

---

## 4. Asset Lifecycle {#lifecycle}

```python
# app/services/inventory/lifecycle_service.py
from datetime import date, timedelta
from typing import List, Dict
from sqlalchemy.orm import Session
from sqlalchemy import text


class AssetLifecycleService:
    """จัดการ asset lifecycle"""
    
    LIFECYCLE_STAGES = [
        'procurement',
        'receiving',
        'staging',
        'deployment',
        'production',
        'maintenance',
        'end_of_life',
        'decommissioned'
    ]
    
    WARNING_DAYS_BEFORE_EXPIRY = 90  # แจ้งเตือนล่วงหน้า 90 วัน
    
    def __init__(self, db: Session):
        self.db = db
    
    def get_expiring_warranties(self, days: int = None) -> List[Dict]:
        """หา devices ที่ warranty ใกล้หมด"""
        days = days or self.WARNING_DAYS_BEFORE_EXPIRY
        cutoff_date = date.today() + timedelta(days=days)
        
        devices = self.db.execute(text("""
            SELECT id, hostname, ip_address, vendor, model,
                   warranty_expiry, location,
                   (warranty_expiry - CURRENT_DATE) as days_remaining
            FROM device_inventory
            WHERE warranty_expiry BETWEEN CURRENT_DATE AND :cutoff_date
            AND status = 'active'
            ORDER BY warranty_expiry
        """), {"cutoff_date": cutoff_date}).fetchall()
        
        return [dict(d._mapping) for d in devices]
    
    def get_eol_devices(self) -> List[Dict]:
        """หา devices ที่ End of Life"""
        # MikroTik EOL information
        EOL_MODELS = {
            'RB951G-2HnD': '2023-01-01',
            'RB2011UiAS-2HnD': '2023-06-01',
            'hEX lite': '2024-01-01',
        }
        
        devices = self.db.execute(text("""
            SELECT id, hostname, ip_address, model, firmware_version, 
                   purchase_date, location
            FROM device_inventory
            WHERE status = 'active'
        """)).fetchall()
        
        eol_devices = []
        today = date.today().isoformat()
        
        for device in devices:
            model = device.model or ''
            for eol_model, eol_date in EOL_MODELS.items():
                if eol_model.lower() in model.lower() and eol_date <= today:
                    device_dict = dict(device._mapping)
                    device_dict['eol_date'] = eol_date
                    device_dict['days_past_eol'] = (
                        date.today() - date.fromisoformat(eol_date)
                    ).days
                    eol_devices.append(device_dict)
                    break
        
        return eol_devices
    
    def get_asset_summary(self) -> Dict:
        """ดึง asset summary"""
        result = self.db.execute(text("""
            SELECT
                COUNT(*) as total_devices,
                COUNT(*) FILTER (WHERE status = 'active') as active,
                COUNT(*) FILTER (WHERE status = 'maintenance') as in_maintenance,
                COUNT(*) FILTER (WHERE status = 'decommissioned') as decommissioned,
                COUNT(*) FILTER (WHERE warranty_expiry < CURRENT_DATE) as expired_warranty,
                COUNT(*) FILTER (
                    WHERE warranty_expiry BETWEEN CURRENT_DATE 
                    AND CURRENT_DATE + INTERVAL '90 days'
                ) as expiring_warranty,
                COUNT(DISTINCT vendor) as vendors,
                COUNT(DISTINCT site) as sites
            FROM device_inventory
        """)).fetchone()
        
        return dict(result._mapping)
```

---

## 5. Lab: Network Inventory System {#lab}

### Setup

```bash
# 1. Install dependencies
pip install pysnmp librouteros schedule reportlab

# 2. Create database tables
psql $DATABASE_URL -f database/inventory_schema.sql

# 3. Run initial discovery
python -c "
from app.services.inventory.discovery_service import NetworkDiscovery
import asyncio

async def main():
    discovery = NetworkDiscovery()
    devices = await discovery.discover_subnet('192.168.1.0/24')
    for device in devices:
        print(device)

asyncio.run(main())
"

# 4. Run config backup
python -c "
from app.database import SessionLocal
from app.services.inventory.config_manager import ConfigurationManager
from app.services.mikrotik import router_manager

db = SessionLocal()
mgr = ConfigurationManager(db, router_manager)
result = mgr.bulk_backup_all_routers()
print(result)
"

# 5. Check for warranty expiry
python -c "
from app.database import SessionLocal
from app.services.inventory.lifecycle_service import AssetLifecycleService

db = SessionLocal()
svc = AssetLifecycleService(db)
expiring = svc.get_expiring_warranties(90)
for device in expiring:
    print(f'{device[\"hostname\"]}: {device[\"days_remaining\"]} days remaining')
"
```

### Scheduled Jobs

```python
# scripts/inventory_jobs.py
import schedule
import time
import logging

logging.basicConfig(level=logging.INFO)


def run_discovery():
    """รัน auto-discovery ทุกคืน"""
    import asyncio
    from app.services.inventory.discovery_service import NetworkDiscovery
    
    discovery = NetworkDiscovery()
    subnets = ['192.168.1.0/24', '10.0.0.0/24']
    
    async def discover():
        for subnet in subnets:
            devices = await discovery.discover_subnet(subnet)
            print(f"Found {len(devices)} devices in {subnet}")
    
    asyncio.run(discover())


def run_config_backup():
    """รัน config backup ทุก 6 ชั่วโมง"""
    from app.database import SessionLocal
    from app.services.inventory.config_manager import ConfigurationManager
    from app.services.mikrotik import router_manager
    
    db = SessionLocal()
    try:
        mgr = ConfigurationManager(db, router_manager)
        result = mgr.bulk_backup_all_routers()
        print(f"Config backup: {result}")
    finally:
        db.close()


# Schedule
schedule.every().day.at("02:00").do(run_discovery)
schedule.every(6).hours.do(run_config_backup)

if __name__ == "__main__":
    while True:
        schedule.run_pending()
        time.sleep(60)
```

### Verification Checklist

- [ ] Auto-discovery finds devices in subnet
- [ ] MikroTik routers identified correctly
- [ ] Config backup runs and stores successfully
- [ ] Config diff shows changes accurately
- [ ] Warranty expiry alerts work
- [ ] EOL device report generated
- [ ] Asset summary dashboard shows correct data
- [ ] API endpoints return proper data

> **Tip:** ใช้ SNMP v3 แทน v2c เพื่อ security ที่ดีกว่า

> **Warning:** Auto-discovery อาจ trigger security alerts ใน network ที่มี IDS/IPS

---

## Summary

Part นี้ครอบคลุม:
- **Auto-discovery** ด้วย ping, SNMP, และ MikroTik API
- **Configuration backup** และ diff comparison
- **Change management** tracking
- **Asset lifecycle** management พร้อม warranty tracking
- **Scheduled jobs** สำหรับ automation

---

[← Part 67: ISP Platform](part-067-isp-platform.md) | [Part 69: Report Generation →](part-069-report-generation.md)
