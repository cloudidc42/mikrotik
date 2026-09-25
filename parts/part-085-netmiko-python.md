# Part 85: Netmiko และ NAPALM สำหรับ MikroTik

## สารบัญ
1. [Netmiko Overview](#netmiko)
2. [MikroTik Driver](#driver)
3. [Configuration Management](#config)
4. [NAPALM Integration](#napalm)
5. [Network Automation Patterns](#patterns)
6. [Lab: Python Network Automation](#lab)

---

## 1. Netmiko Overview {#netmiko}

Netmiko เป็น Python library สำหรับ connect กับ network devices ผ่าน SSH

```bash
pip install netmiko napalm
```

```python
# Basic Netmiko connection
from netmiko import ConnectHandler

# MikroTik RouterOS connection
device = {
    "device_type": "mikrotik_routeros",
    "host": "192.168.1.1",
    "username": "admin",
    "password": "admin123",
    "port": 22
}

conn = ConnectHandler(**device)

# Run command
output = conn.send_command("/ip address print")
print(output)

conn.disconnect()
```

---

## 2. MikroTik Driver {#driver}

```python
# ============================================
# Advanced Netmiko MikroTik Usage
# ============================================

from netmiko import ConnectHandler, NetmikoTimeoutException
from typing import Optional, List
import logging
import re

logger = logging.getLogger(__name__)


class MikroTikManager:
    """Manager class สำหรับ MikroTik automation ด้วย Netmiko"""
    
    def __init__(self, host: str, username: str, password: str, 
                 port: int = 22, timeout: int = 30):
        self.host = host
        self.device_params = {
            "device_type": "mikrotik_routeros",
            "host": host,
            "username": username,
            "password": password,
            "port": port,
            "timeout": timeout,
            "session_log": f"session_{host}.log"
        }
        self._connection: Optional[ConnectHandler] = None
    
    def connect(self) -> bool:
        """เชื่อมต่อ router"""
        try:
            self._connection = ConnectHandler(**self.device_params)
            logger.info(f"Connected to {self.host}")
            return True
        except NetmikoTimeoutException:
            logger.error(f"Timeout connecting to {self.host}")
            return False
        except Exception as e:
            logger.error(f"Error connecting to {self.host}: {e}")
            return False
    
    def disconnect(self):
        """ตัดการเชื่อมต่อ"""
        if self._connection:
            self._connection.disconnect()
            self._connection = None
    
    def send_command(self, command: str) -> str:
        """ส่ง command และรับ output"""
        if not self._connection:
            raise RuntimeError("Not connected")
        return self._connection.send_command(command)
    
    def send_config_set(self, commands: List[str]) -> str:
        """ส่ง multiple config commands"""
        if not self._connection:
            raise RuntimeError("Not connected")
        return self._connection.send_config_set(commands)
    
    def get_interfaces(self) -> List[dict]:
        """ดึงข้อมูล interfaces"""
        output = self.send_command("/interface print")
        interfaces = []
        
        # Parse output
        for line in output.split('\n'):
            if re.match(r'\s+\d+', line):
                parts = line.split()
                if len(parts) >= 4:
                    interfaces.append({
                        "index": parts[0],
                        "flags": parts[1] if len(parts[1]) > 0 else "",
                        "name": parts[2] if not parts[1].startswith('X') else parts[1],
                        "type": "ether"
                    })
        
        return interfaces
    
    def get_ip_addresses(self) -> List[dict]:
        """ดึงข้อมูล IP addresses"""
        output = self.send_command("/ip address print detail")
        addresses = []
        
        blocks = re.findall(
            r'(\d+)\s+(?:X\s+)?address=([\d./]+)\s+.*?interface=(\S+)',
            output
        )
        
        for block in blocks:
            addresses.append({
                "index": block[0],
                "address": block[1],
                "interface": block[2]
            })
        
        return addresses
    
    def backup_config(self, backup_name: str = "netmiko-backup") -> str:
        """Backup configuration"""
        self.send_command(f"/system backup save name={backup_name}")
        return self.send_command("/system backup print")
    
    def export_config(self) -> str:
        """Export configuration เป็น text"""
        return self.send_command("/export")
    
    def __enter__(self):
        self.connect()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.disconnect()
```

---

## 3. Configuration Management {#config}

```python
# ============================================
# Configuration Management with Netmiko
# ============================================

import difflib
import hashlib
from datetime import datetime
from pathlib import Path


class ConfigurationManager:
    """จัดการ configuration ด้วย version control"""
    
    def __init__(self, backup_dir: str = "./backups"):
        self.backup_dir = Path(backup_dir)
        self.backup_dir.mkdir(exist_ok=True)
    
    def save_config(self, hostname: str, config: str) -> str:
        """บันทึก config พร้อม timestamp"""
        timestamp = datetime.utcnow().strftime("%Y%m%d_%H%M%S")
        filename = f"{hostname}_{timestamp}.rsc"
        filepath = self.backup_dir / filename
        
        filepath.write_text(config)
        return str(filepath)
    
    def get_latest_config(self, hostname: str) -> Optional[str]:
        """ดึง config ล่าสุด"""
        files = sorted(self.backup_dir.glob(f"{hostname}_*.rsc"), reverse=True)
        if files:
            return files[0].read_text()
        return None
    
    def diff_configs(self, old_config: str, new_config: str, 
                     context: int = 3) -> str:
        """เปรียบเทียบ configurations"""
        diff = difflib.unified_diff(
            old_config.splitlines(keepends=True),
            new_config.splitlines(keepends=True),
            fromfile="previous",
            tofile="current",
            n=context
        )
        return "".join(diff)
    
    def has_changed(self, hostname: str, new_config: str) -> bool:
        """ตรวจสอบว่า config เปลี่ยนหรือไม่"""
        old_config = self.get_latest_config(hostname)
        if not old_config:
            return True
        
        old_hash = hashlib.sha256(old_config.encode()).hexdigest()
        new_hash = hashlib.sha256(new_config.encode()).hexdigest()
        
        return old_hash != new_hash
    
    def backup_all_routers(self, routers: List[dict]) -> dict:
        """Backup ทุก routers"""
        results = {}
        
        for router in routers:
            hostname = router["host"]
            try:
                with MikroTikManager(**router) as mgr:
                    config = mgr.export_config()
                    
                    if self.has_changed(hostname, config):
                        filepath = self.save_config(hostname, config)
                        results[hostname] = {"status": "changed", "file": filepath}
                    else:
                        results[hostname] = {"status": "unchanged"}
            
            except Exception as e:
                results[hostname] = {"status": "error", "error": str(e)}
        
        return results


# Example usage
if __name__ == "__main__":
    manager = ConfigurationManager("./router_backups")
    
    routers = [
        {"host": "10.0.0.1", "username": "admin", "password": "pass"},
        {"host": "10.0.0.2", "username": "admin", "password": "pass"},
    ]
    
    results = manager.backup_all_routers(routers)
    for host, result in results.items():
        print(f"{host}: {result['status']}")
```

---

## 4. NAPALM Integration {#napalm}

```python
# ============================================
# NAPALM สำหรับ MikroTik
# ============================================

# NAPALM ยังไม่ support MikroTik อย่างเป็นทางการ
# ใช้ napalm-routeros community driver

pip install napalm napalm-routeros

from napalm import get_network_driver

# Connect
driver = get_network_driver("routeros")
device = driver(
    hostname="192.168.1.1",
    username="admin",
    password="admin123"
)

device.open()

# Get facts
facts = device.get_facts()
print(f"Hostname: {facts['hostname']}")
print(f"OS Version: {facts['os_version']}")
print(f"Model: {facts['model']}")
print(f"Uptime: {facts['uptime']}")
print(f"Interfaces: {facts['interface_list']}")

# Get interfaces
interfaces = device.get_interfaces()
for name, data in interfaces.items():
    print(f"{name}: up={data['is_up']} speed={data['speed']}")

# Get BGP neighbors
bgp = device.get_bgp_neighbors()
for neighbor, data in bgp.get("global", {}).get("peers", {}).items():
    print(f"BGP {neighbor}: state={data['description']}")

# Get ARP table
arp = device.get_arp_table()
for entry in arp:
    print(f"ARP: {entry['ip']} -> {entry['mac']} via {entry['interface']}")

device.close()
```

---

## 5. Network Automation Patterns {#patterns}

```python
# ============================================
# Common Automation Patterns
# ============================================

import asyncio
from concurrent.futures import ThreadPoolExecutor
from typing import Callable

class BulkDeviceManager:
    """จัดการ operations บน multiple devices พร้อมกัน"""
    
    def __init__(self, max_workers: int = 10):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)
    
    def run_on_device(self, device_params: dict, 
                       operation: Callable) -> dict:
        """รัน operation บน single device"""
        hostname = device_params["host"]
        try:
            with MikroTikManager(**device_params) as mgr:
                result = operation(mgr)
                return {"host": hostname, "status": "success", "result": result}
        except Exception as e:
            return {"host": hostname, "status": "error", "error": str(e)}
    
    def run_bulk(self, devices: List[dict], 
                  operation: Callable) -> List[dict]:
        """รัน operation บน multiple devices พร้อมกัน"""
        futures = [
            self.executor.submit(self.run_on_device, device, operation)
            for device in devices
        ]
        
        results = []
        for future in futures:
            results.append(future.result())
        
        return results


# ตัวอย่างการใช้งาน
def get_router_info(mgr: MikroTikManager) -> dict:
    return {
        "identity": mgr.send_command("/system identity get name"),
        "version": mgr.send_command("/system resource get version"),
        "uptime": mgr.send_command("/system resource get uptime")
    }


if __name__ == "__main__":
    devices = [
        {"host": "10.0.0.1", "username": "admin", "password": "pass"},
        {"host": "10.0.0.2", "username": "admin", "password": "pass"},
        {"host": "10.0.0.3", "username": "admin", "password": "pass"},
    ]
    
    bulk = BulkDeviceManager(max_workers=5)
    results = bulk.run_bulk(devices, get_router_info)
    
    for result in results:
        if result["status"] == "success":
            print(f"OK: {result['host']} - {result['result']['identity']}")
        else:
            print(f"ERROR: {result['host']} - {result['error']}")
```

---

## 6. Lab: Python Network Automation {#lab}

### Lab Steps

```python
# lab_automation.py

from netmiko import ConnectHandler
import json

def run_lab():
    # Connect to router
    device = {
        "device_type": "mikrotik_routeros",
        "host": "192.168.1.1",
        "username": "admin",
        "password": "admin123"
    }
    
    with ConnectHandler(**device) as conn:
        # 1. Get router info
        identity = conn.send_command("/system identity get name")
        version = conn.send_command("/system resource get version")
        print(f"Router: {identity} v{version}")
        
        # 2. List interfaces
        ifaces = conn.send_command("/interface print terse")
        print("\nInterfaces:")
        print(ifaces)
        
        # 3. Add comment to interface
        conn.send_config_set([
            "/interface set ether1 comment=WAN"
        ])
        
        # 4. Verify change
        output = conn.send_command("/interface print where name=ether1")
        print("\nUpdated interface:")
        print(output)

run_lab()
```

### Verification Checklist

- [ ] Netmiko connect ทำงาน
- [ ] send_command ส่ง commands ได้
- [ ] send_config_set apply changes
- [ ] NAPALM facts ทำงาน
- [ ] Bulk operations เร็วกว่า sequential
- [ ] Error handling ทำงาน

> **Tip:** ใช้ `session_log` parameter เพื่อ debug connection issues

> **Note:** Netmiko MikroTik driver ใช้ SSH - ต้องเปิด SSH service

---

## Summary

Part นี้ครอบคลุม:
- **Netmiko** สำหรับ SSH automation
- **MikroTik driver** configuration
- **Configuration management** with versioning
- **NAPALM** multi-vendor abstraction
- **Bulk operations** patterns

---

[← Part 84: Terraform](part-084-terraform.md) | [Part 86: ELK Logging →](part-086-elk-logging.md)
