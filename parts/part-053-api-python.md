# Part 53: MikroTik API ด้วย Python

## บทนำ

Python เป็นภาษาที่เหมาะมากสำหรับ network automation เนื่องจากมี libraries ที่หลากหลาย, syntax ที่อ่านง่าย, และ ecosystem ที่ครบครัน การใช้ Python กับ MikroTik API ช่วยสร้าง automation scripts และ monitoring tools ได้อย่างมีประสิทธิภาพ

---

## 53.1 Python Libraries Overview

### ตัวเลือก Libraries

| Library | Package | Features | Status |
|---------|---------|----------|--------|
| routeros-api | `pip install routeros-api` | Simple, widely-used | Active |
| librouteros | `pip install librouteros` | Modern, async support | Active |
| paramiko | `pip install paramiko` | SSH-based | Alternative |
| netmiko | `pip install netmiko` | Multi-vendor | Alternative |

---

## 53.2 Installation and Setup

### ติดตั้ง Libraries

```bash
# สร้าง virtual environment
python3 -m venv mikrotik-env
source mikrotik-env/bin/activate  # Linux/Mac
# หรือ
mikrotik-env\Scripts\activate  # Windows

# ติดตั้ง libraries
pip install routeros-api librouteros pandas matplotlib

# ตรวจสอบ installation
python3 -c "import routeros_api; print('routeros-api OK')"
python3 -c "import librouteros; print('librouteros OK')"
```

### requirements.txt

```
routeros-api>=0.1.3
librouteros>=3.2.1
pandas>=2.0.0
matplotlib>=3.7.0
aiohttp>=3.9.0
asyncio>=3.4.3
```

### ทดสอบ Connection พื้นฐาน

```python
#!/usr/bin/env python3
# test_connection.py

import routeros_api

def test_connection():
    connection = routeros_api.RouterOsApiPool(
        '192.168.1.1',
        username='admin',
        password='',
        port=8728
    )
    
    try:
        api = connection.get_api()
        
        # ทดสอบ query
        identity = api.get_resource('/system/identity')
        result = identity.get()
        print(f"✓ Connected! Router: {result[0]['name']}")
        
        return api
    except Exception as e:
        print(f"✗ Connection failed: {e}")
        raise
    finally:
        connection.disconnect()

if __name__ == '__main__':
    test_connection()
```

---

## 53.3 Basic CRUD Operations

### Complete CRUD Class

```python
#!/usr/bin/env python3
# mikrotik_crud.py

import routeros_api
from typing import Optional, List, Dict, Any
import logging

logger = logging.getLogger(__name__)

class MikroTikCRUD:
    """CRUD operations สำหรับ MikroTik RouterOS"""
    
    def __init__(self, host: str, username: str = 'admin', 
                 password: str = '', port: int = 8728):
        self.host = host
        self.port = port
        self._connection = routeros_api.RouterOsApiPool(
            host, username=username, password=password, port=port
        )
        self._api = None
    
    def connect(self) -> 'MikroTikCRUD':
        """เชื่อมต่อ"""
        self._api = self._connection.get_api()
        logger.info(f"Connected to {self.host}:{self.port}")
        return self
    
    def disconnect(self) -> None:
        """ปิดการเชื่อมต่อ"""
        self._connection.disconnect()
        self._api = None
        logger.info("Disconnected")
    
    def __enter__(self):
        return self.connect()
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.disconnect()
    
    # ==================== READ ====================
    
    def get_all(self, path: str) -> List[Dict[str, Any]]:
        """ดูข้อมูลทั้งหมด"""
        resource = self._api.get_resource(path)
        return resource.get()
    
    def get_where(self, path: str, **conditions) -> List[Dict[str, Any]]:
        """ดูข้อมูลแบบมี filter"""
        resource = self._api.get_resource(path)
        return resource.get(**conditions)
    
    def get_by_id(self, path: str, item_id: str) -> Optional[Dict[str, Any]]:
        """ดูข้อมูลตาม ID"""
        resource = self._api.get_resource(path)
        results = resource.get(**{'.id': item_id})
        return results[0] if results else None
    
    # ==================== CREATE ====================
    
    def add(self, path: str, **values) -> Optional[str]:
        """เพิ่มข้อมูล"""
        resource = self._api.get_resource(path)
        try:
            result = resource.add(**values)
            return result.get('ret')
        except Exception as e:
            logger.error(f"Add failed at {path}: {e}")
            raise
    
    # ==================== UPDATE ====================
    
    def update(self, path: str, item_id: str, **values) -> bool:
        """แก้ไขข้อมูล"""
        resource = self._api.get_resource(path)
        try:
            resource.set(id=item_id, **values)
            return True
        except Exception as e:
            logger.error(f"Update failed at {path}: {e}")
            return False
    
    # ==================== DELETE ====================
    
    def remove(self, path: str, item_id: str) -> bool:
        """ลบข้อมูล"""
        resource = self._api.get_resource(path)
        try:
            resource.remove(id=item_id)
            return True
        except Exception as e:
            logger.error(f"Remove failed at {path}: {e}")
            return False
    
    # ==================== EXECUTE ====================
    
    def execute(self, path: str, **params) -> List[Dict]:
        """Execute command"""
        resource = self._api.get_resource(path)
        return resource.call('', **params) if params else resource.call('')


# ตัวอย่างการใช้งาน
def example_usage():
    with MikroTikCRUD('192.168.1.1', username='admin') as api:
        
        # READ - ดู interfaces
        print("=== Interfaces ===")
        interfaces = api.get_all('/interface')
        for iface in interfaces:
            status = "UP" if iface.get('running') == 'true' else "DOWN"
            print(f"  [{status}] {iface['name']} ({iface['type']})")
        
        # READ - ดู IP addresses
        print("\n=== IP Addresses ===")
        addresses = api.get_where('/ip/address', invalid='false')
        for addr in addresses:
            print(f"  {addr['address']} on {addr['interface']}")
        
        # CREATE - เพิ่ม address list entry
        new_id = api.add('/ip/firewall/address-list',
                        list='TEST-LIST',
                        address='192.168.100.1',
                        comment='Python API test')
        print(f"\n=== Added entry with ID: {new_id} ===")
        
        # UPDATE - แก้ไข comment
        if new_id:
            api.update('/ip/firewall/address-list', new_id,
                      comment='Updated by Python')
        
        # DELETE - ลบที่เพิ่ม
        if new_id:
            api.remove('/ip/firewall/address-list', new_id)
            print("Entry removed")


if __name__ == '__main__':
    example_usage()
```

---

## 53.4 Streaming Data

### Real-time Traffic Monitoring

```python
#!/usr/bin/env python3
# streaming_monitor.py

import routeros_api
import time
from datetime import datetime

def stream_interface_stats(host: str, interface: str, duration: int = 60):
    """Stream interface statistics แบบ real-time"""
    
    connection = routeros_api.RouterOsApiPool(
        host, username='admin', password='', port=8728
    )
    api = connection.get_api()
    
    print(f"Streaming stats for {interface} on {host}")
    print(f"{'Time':<20} {'RX bps':<15} {'TX bps':<15} {'RX drops':<12} {'TX drops':<12}")
    print("-" * 75)
    
    start_time = time.time()
    
    try:
        while time.time() - start_time < duration:
            resource = api.get_resource('/interface')
            stats = resource.call('monitor-traffic',
                                 interface=interface,
                                 once='')
            
            if stats:
                stat = stats[0]
                timestamp = datetime.now().strftime('%H:%M:%S')
                rx_bps = int(stat.get('rx-bits-per-second', 0))
                tx_bps = int(stat.get('tx-bits-per-second', 0))
                rx_drop = int(stat.get('rx-drop', 0))
                tx_drop = int(stat.get('tx-drop', 0))
                
                # แปลงเป็น human-readable
                rx_str = format_bps(rx_bps)
                tx_str = format_bps(tx_bps)
                
                print(f"{timestamp:<20} {rx_str:<15} {tx_str:<15} {rx_drop:<12} {tx_drop:<12}")
            
            time.sleep(1)
    
    except KeyboardInterrupt:
        print("\nMonitoring stopped")
    finally:
        connection.disconnect()


def format_bps(bps: int) -> str:
    """แปลง bits per second เป็น human-readable"""
    if bps >= 1_000_000_000:
        return f"{bps/1_000_000_000:.2f} Gbps"
    elif bps >= 1_000_000:
        return f"{bps/1_000_000:.2f} Mbps"
    elif bps >= 1_000:
        return f"{bps/1_000:.2f} Kbps"
    else:
        return f"{bps} bps"


if __name__ == '__main__':
    stream_interface_stats('192.168.1.1', 'ether1', duration=30)
```

---

## 53.5 Async with asyncio

### Async MikroTik Client

```python
#!/usr/bin/env python3
# async_client.py

import asyncio
import librouteros
from typing import List, Dict, Any

class AsyncMikroTikClient:
    """Async MikroTik client สำหรับ high-performance operations"""
    
    def __init__(self, host: str, username: str = 'admin',
                 password: str = '', port: int = 8728):
        self.host = host
        self.username = username
        self.password = password
        self.port = port
        self._api = None
    
    async def connect(self):
        """เชื่อมต่อแบบ async"""
        self._api = await librouteros.create_connection(
            self.host,
            username=self.username,
            password=self.password,
            port=self.port,
        )
        return self
    
    async def disconnect(self):
        """ปิดการเชื่อมต่อ"""
        if self._api:
            self._api.close()
    
    async def __aenter__(self):
        return await self.connect()
    
    async def __aexit__(self, *args):
        await self.disconnect()
    
    async def get_all(self, path: str) -> List[Dict]:
        """ดูข้อมูลแบบ async"""
        api_path = self._api.path(*path.split('/')[1:])
        return [item async for item in api_path]
    
    async def add(self, path: str, **params):
        """เพิ่มข้อมูลแบบ async"""
        api_path = self._api.path(*path.split('/')[1:])
        return await api_path.add(**params)


async def monitor_multiple_routers(routers: List[Dict]):
    """Monitor หลาย routers พร้อมกัน"""
    
    async def get_router_stats(config: Dict) -> Dict:
        async with AsyncMikroTikClient(**config) as client:
            resources = await client.get_all('/system/resource')
            interfaces = await client.get_all('/interface')
            
            return {
                'host': config['host'],
                'cpu': resources[0].get('cpu-load', 0) if resources else 0,
                'memory': resources[0].get('free-memory', 0) if resources else 0,
                'interfaces': len(interfaces),
            }
    
    # รัน parallel
    tasks = [get_router_stats(router) for router in routers]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    
    for result in results:
        if isinstance(result, Exception):
            print(f"Error: {result}")
        else:
            print(f"Router {result['host']}: CPU={result['cpu']}%, "
                  f"Interfaces={result['interfaces']}")


# ตัวอย่าง
async def main():
    routers = [
        {'host': '192.168.1.1', 'username': 'admin', 'password': ''},
        {'host': '192.168.1.2', 'username': 'admin', 'password': ''},
    ]
    
    await monitor_multiple_routers(routers)


if __name__ == '__main__':
    asyncio.run(main())
```

---

## 53.6 Error Handling

### Comprehensive Error Handler

```python
#!/usr/bin/env python3
# error_handling.py

import routeros_api
import logging
from functools import wraps
from typing import Callable, Any

logger = logging.getLogger(__name__)

class RouterOSError(Exception):
    pass

class RouterOSConnectionError(RouterOSError):
    pass

class RouterOSCommandError(RouterOSError):
    def __init__(self, message: str, category: str = None):
        super().__init__(message)
        self.category = category

def retry_on_disconnect(max_retries: int = 3, delay: float = 2.0):
    """Decorator สำหรับ retry เมื่อ connection หลุด"""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(self, *args, **kwargs):
            for attempt in range(max_retries):
                try:
                    return func(self, *args, **kwargs)
                except (ConnectionResetError, BrokenPipeError) as e:
                    if attempt < max_retries - 1:
                        logger.warning(f"Connection lost, retrying ({attempt + 1}/{max_retries})")
                        import time
                        time.sleep(delay * (attempt + 1))
                        self.reconnect()
                    else:
                        raise RouterOSConnectionError(f"Failed after {max_retries} retries: {e}")
        return wrapper
    return decorator


class RobustMikroTikClient:
    """Client ที่มี error handling ครบถ้วน"""
    
    def __init__(self, host: str, **kwargs):
        self.host = host
        self.kwargs = kwargs
        self._api = None
        self._connection = None
        self.connect()
    
    def connect(self):
        try:
            self._connection = routeros_api.RouterOsApiPool(
                self.host, **self.kwargs
            )
            self._api = self._connection.get_api()
            logger.info(f"Connected to {self.host}")
        except Exception as e:
            raise RouterOSConnectionError(f"Cannot connect to {self.host}: {e}")
    
    def reconnect(self):
        """Re-establish connection"""
        logger.info("Reconnecting...")
        try:
            self._connection.disconnect()
        except Exception:
            pass
        self.connect()
    
    @retry_on_disconnect(max_retries=3)
    def safe_get(self, path: str, **conditions) -> list:
        """Get ด้วย error handling"""
        resource = self._api.get_resource(path)
        return resource.get(**conditions) if conditions else resource.get()
    
    @retry_on_disconnect(max_retries=3)
    def safe_add(self, path: str, **values) -> str:
        """Add ด้วย error handling"""
        resource = self._api.get_resource(path)
        try:
            return resource.add(**values).get('ret', '')
        except routeros_api.exceptions.TrapError as e:
            # Parse RouterOS error
            error_msg = str(e)
            if 'already have' in error_msg:
                raise RouterOSCommandError("Duplicate entry", category='duplicate')
            raise RouterOSCommandError(error_msg)
```

---

## 53.7 Pandas Integration for Analysis

### Network Data Analysis

```python
#!/usr/bin/env python3
# network_analysis.py

import routeros_api
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime, timedelta
import json

class NetworkAnalyzer:
    """วิเคราะห์ network data ด้วย Pandas"""
    
    def __init__(self, host: str, **auth):
        self.connection = routeros_api.RouterOsApiPool(host, **auth)
        self.api = self.connection.get_api()
    
    def get_hotspot_stats(self) -> pd.DataFrame:
        """ดึง hotspot user statistics"""
        users = self.api.get_resource('/ip/hotspot/user').get()
        
        df = pd.DataFrame(users)
        
        # แปลง types
        if not df.empty:
            numeric_cols = ['bytes-in', 'bytes-out', 'packets-in', 'packets-out']
            for col in numeric_cols:
                if col in df.columns:
                    df[col] = pd.to_numeric(df[col], errors='coerce').fillna(0)
        
        return df
    
    def analyze_traffic(self, df: pd.DataFrame) -> Dict:
        """วิเคราะห์ traffic patterns"""
        if df.empty:
            return {}
        
        analysis = {
            'total_users': len(df),
            'total_bytes_in': df['bytes-in'].sum() if 'bytes-in' in df.columns else 0,
            'total_bytes_out': df['bytes-out'].sum() if 'bytes-out' in df.columns else 0,
            'top_users': df.nlargest(5, 'bytes-in')[['name', 'bytes-in']].to_dict('records'),
        }
        
        return analysis
    
    def plot_traffic(self, df: pd.DataFrame, output_file: str = 'traffic.png'):
        """สร้าง traffic graph"""
        if df.empty or 'bytes-in' not in df.columns:
            return
        
        fig, axes = plt.subplots(1, 2, figsize=(12, 5))
        
        # Top 10 users by download
        top_users = df.nlargest(10, 'bytes-in')
        axes[0].barh(top_users['name'], top_users['bytes-in'] / 1024 / 1024)
        axes[0].set_xlabel('Download (MB)')
        axes[0].set_title('Top 10 Users by Download')
        
        # Upload vs Download scatter
        if 'bytes-out' in df.columns:
            axes[1].scatter(df['bytes-in'] / 1024 / 1024,
                           df['bytes-out'] / 1024 / 1024, alpha=0.6)
            axes[1].set_xlabel('Download (MB)')
            axes[1].set_ylabel('Upload (MB)')
            axes[1].set_title('Upload vs Download')
        
        plt.tight_layout()
        plt.savefig(output_file, dpi=100, bbox_inches='tight')
        print(f"Graph saved: {output_file}")
    
    def generate_report(self) -> str:
        """สร้าง text report"""
        df = self.get_hotspot_stats()
        analysis = self.analyze_traffic(df)
        
        report = f"""
NETWORK ANALYSIS REPORT
Generated: {datetime.now().strftime('%Y-%m-%d %H:%M:%S')}
{'='*50}

Total Users: {analysis.get('total_users', 0)}
Total Download: {analysis.get('total_bytes_in', 0) / 1024 / 1024:.2f} MB
Total Upload: {analysis.get('total_bytes_out', 0) / 1024 / 1024:.2f} MB

Top 5 Users by Download:
"""
        for user in analysis.get('top_users', []):
            report += f"  - {user['name']}: {user.get('bytes-in', 0) / 1024 / 1024:.2f} MB\n"
        
        return report
    
    def close(self):
        self.connection.disconnect()


# ตัวอย่าง
if __name__ == '__main__':
    analyzer = NetworkAnalyzer('192.168.1.1', username='admin', password='')
    
    # สร้าง report
    report = analyzer.generate_report()
    print(report)
    
    # สร้าง graph
    df = analyzer.get_hotspot_stats()
    analyzer.plot_traffic(df, 'network_traffic.png')
    
    analyzer.close()
```

---

## 53.8 Building Monitoring Tools

### Network Monitor Service

```python
#!/usr/bin/env python3
# network_monitor.py

import routeros_api
import time
import json
import logging
from datetime import datetime
from pathlib import Path
from typing import Dict, List

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(message)s'
)
logger = logging.getLogger(__name__)


class NetworkMonitor:
    """Monitor network metrics จาก MikroTik"""
    
    def __init__(self, config: Dict):
        self.config = config
        self.connection = routeros_api.RouterOsApiPool(
            config['host'],
            username=config.get('username', 'admin'),
            password=config.get('password', ''),
            port=config.get('port', 8728),
        )
        self.api = self.connection.get_api()
        self.metrics_history = []
    
    def collect_metrics(self) -> Dict:
        """เก็บ metrics ทั้งหมด"""
        timestamp = datetime.now().isoformat()
        
        # System resources
        resources = self.api.get_resource('/system/resource').get()
        sys_resource = resources[0] if resources else {}
        
        # Interface stats
        interfaces = self.api.get_resource('/interface').get()
        interface_data = []
        for iface in interfaces:
            if iface.get('running') == 'true':
                interface_data.append({
                    'name': iface['name'],
                    'type': iface.get('type', 'unknown'),
                    'rx_byte': iface.get('rx-byte', 0),
                    'tx_byte': iface.get('tx-byte', 0),
                })
        
        # Hotspot active users
        try:
            hotspot = self.api.get_resource('/ip/hotspot/active').get()
        except Exception:
            hotspot = []
        
        metrics = {
            'timestamp': timestamp,
            'system': {
                'cpu_load': int(sys_resource.get('cpu-load', 0)),
                'free_memory': int(sys_resource.get('free-memory', 0)),
                'total_memory': int(sys_resource.get('total-memory', 0)),
                'uptime': sys_resource.get('uptime', '0s'),
                'version': sys_resource.get('version', 'unknown'),
            },
            'interfaces': interface_data,
            'hotspot_users': len(hotspot),
        }
        
        self.metrics_history.append(metrics)
        
        # เก็บแค่ 1000 records ล่าสุด
        if len(self.metrics_history) > 1000:
            self.metrics_history.pop(0)
        
        return metrics
    
    def check_alerts(self, metrics: Dict) -> List[Dict]:
        """ตรวจสอบ alert conditions"""
        alerts = []
        
        # CPU alert
        if metrics['system']['cpu_load'] > 80:
            alerts.append({
                'severity': 'warning',
                'message': f"High CPU: {metrics['system']['cpu_load']}%",
                'timestamp': metrics['timestamp'],
            })
        
        # Memory alert
        total = metrics['system']['total_memory']
        free = metrics['system']['free_memory']
        if total > 0:
            used_pct = ((total - free) / total) * 100
            if used_pct > 90:
                alerts.append({
                    'severity': 'critical',
                    'message': f"High memory usage: {used_pct:.1f}%",
                    'timestamp': metrics['timestamp'],
                })
        
        return alerts
    
    def run(self, interval: int = 30, max_runs: int = None):
        """รัน monitoring loop"""
        runs = 0
        
        logger.info(f"Starting monitoring (interval={interval}s)")
        
        while max_runs is None or runs < max_runs:
            try:
                metrics = self.collect_metrics()
                alerts = self.check_alerts(metrics)
                
                # Log metrics
                logger.info(
                    f"CPU={metrics['system']['cpu_load']}% "
                    f"Mem={metrics['system']['free_memory']//1024//1024}MB free "
                    f"Users={metrics['hotspot_users']}"
                )
                
                # Log alerts
                for alert in alerts:
                    if alert['severity'] == 'critical':
                        logger.error(f"ALERT: {alert['message']}")
                    else:
                        logger.warning(f"ALERT: {alert['message']}")
                
                runs += 1
                time.sleep(interval)
                
            except KeyboardInterrupt:
                logger.info("Monitoring stopped by user")
                break
            except Exception as e:
                logger.error(f"Error collecting metrics: {e}")
                time.sleep(interval)
    
    def close(self):
        """ปิดการเชื่อมต่อ"""
        self.connection.disconnect()


if __name__ == '__main__':
    config = {
        'host': '192.168.1.1',
        'username': 'admin',
        'password': '',
    }
    
    monitor = NetworkMonitor(config)
    
    try:
        monitor.run(interval=30, max_runs=10)
    finally:
        monitor.close()
```

---

## 53.9 Production Patterns

### Configuration Management

```python
#!/usr/bin/env python3
# config.py

import os
from dataclasses import dataclass
from typing import Optional

@dataclass
class RouterConfig:
    host: str
    username: str = 'admin'
    password: str = ''
    port: int = 8728
    ssl: bool = False
    timeout: int = 30
    
    @classmethod
    def from_env(cls) -> 'RouterConfig':
        """โหลด config จาก environment variables"""
        return cls(
            host=os.environ.get('MIKROTIK_HOST', '192.168.1.1'),
            username=os.environ.get('MIKROTIK_USER', 'admin'),
            password=os.environ.get('MIKROTIK_PASS', ''),
            port=int(os.environ.get('MIKROTIK_PORT', '8728')),
            ssl=os.environ.get('MIKROTIK_SSL', 'false').lower() == 'true',
        )
    
    @classmethod
    def from_dict(cls, data: dict) -> 'RouterConfig':
        return cls(**{k: v for k, v in data.items() if k in cls.__dataclass_fields__})
```

---

## 53.10 Lab: Python Network Monitor

### Lab Overview

| ระบบ | รายละเอียด |
|------|------------|
| Language | Python 3.10+ |
| Libraries | routeros-api, pandas, matplotlib |
| Features | Real-time monitor, analytics, alerts |

### Complete Lab Script

```python
#!/usr/bin/env python3
# lab_monitor.py

"""
Lab: Python MikroTik Network Monitor
รันด้วย: python3 lab_monitor.py
"""

import routeros_api
import time
import sys
from datetime import datetime

def run_lab_monitor():
    HOST = '192.168.1.1'
    
    print("="*50)
    print(" Python MikroTik Network Monitor Lab")
    print("="*50)
    
    # เชื่อมต่อ
    try:
        conn = routeros_api.RouterOsApiPool(HOST, username='admin', password='')
        api = conn.get_api()
        print(f"✓ Connected to {HOST}")
    except Exception as e:
        print(f"✗ Connection failed: {e}")
        sys.exit(1)
    
    try:
        # 1. System Info
        resources = api.get_resource('/system/resource').get()
        if resources:
            r = resources[0]
            print(f"\n[System]")
            print(f"  Version: {r.get('version', 'N/A')}")
            print(f"  Uptime: {r.get('uptime', 'N/A')}")
            print(f"  CPU: {r.get('cpu-load', 0)}%")
            mem_total = int(r.get('total-memory', 0))
            mem_free = int(r.get('free-memory', 0))
            mem_used = mem_total - mem_free
            print(f"  Memory: {mem_used//1024//1024}MB used / {mem_total//1024//1024}MB total")
        
        # 2. Interface Monitor
        print(f"\n[Interfaces - Running]")
        interfaces = api.get_resource('/interface').get()
        for iface in interfaces:
            if iface.get('running') == 'true':
                print(f"  ✓ {iface['name']} ({iface.get('type', '?')})")
        
        # 3. IP Addresses
        print(f"\n[IP Addresses]")
        addresses = api.get_resource('/ip/address').get()
        for addr in addresses:
            if addr.get('invalid') != 'true':
                print(f"  {addr.get('address', 'N/A')} on {addr.get('interface', 'N/A')}")
        
        # 4. Routes
        print(f"\n[Active Routes]")
        routes = api.get_resource('/ip/route').get()
        for route in routes:
            if route.get('active') == 'true':
                print(f"  {route.get('dst-address', 'N/A')} via {route.get('gateway', 'direct')}")
        
        print("\n✓ Lab complete!")
        
    finally:
        conn.disconnect()
        print("✓ Disconnected")


if __name__ == '__main__':
    run_lab_monitor()
```

### การรัน Lab

```bash
# สร้าง virtual environment
python3 -m venv lab-env
source lab-env/bin/activate

# ติดตั้ง dependencies
pip install routeros-api pandas matplotlib

# รัน lab
python3 lab_monitor.py
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Libraries | routeros-api, librouteros |
| Setup | venv, pip, requirements.txt |
| CRUD | Get, Add, Update, Remove |
| Streaming | Real-time traffic stats |
| Async | asyncio parallel operations |
| Error Handling | Retry, exception hierarchy |
| Pandas | Data analysis and visualization |
| Monitoring | Metrics collection, alerts |
| Production | Config management, logging |

---

[← Part 52: API PHP](part-052-api-php.md) | [Part 54: API Node.js →](part-054-api-nodejs.md)
