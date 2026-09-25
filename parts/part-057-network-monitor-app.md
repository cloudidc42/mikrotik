# Part 57: Network Monitoring Application

## บทนำ

Network Monitoring Application ที่สมบูรณ์ต้องการ stack หลายชั้น ตั้งแต่ data collection, time-series storage, visualization ไปจนถึง alerting MikroTik network monitoring สามารถสร้างได้ด้วย InfluxDB + Grafana stack หรือ custom solution

---

## 57.1 Monitoring Architecture

### Stack Components

```
┌─────────────────────────────────────────────┐
│            Grafana Dashboards                │
│     (Visualization + Alerting UI)            │
└──────────────────┬──────────────────────────┘
                   │ Query
┌──────────────────┴──────────────────────────┐
│              InfluxDB                        │
│         (Time-series Database)               │
└──────────────────┬──────────────────────────┘
                   │ Write
┌──────────────────┴──────────────────────────┐
│          Data Collector Daemon               │
│     (Node.js / Python script)               │
└──────────────────┬──────────────────────────┘
                   │ RouterOS API
┌──────────────────┴──────────────────────────┐
│            MikroTik Routers                  │
└─────────────────────────────────────────────┘
```

### Data Flow

```
Router → Collector (every 30s) → InfluxDB → Grafana → Users
                                    ↓
                               Alert Rules → Notifications
```

---

## 57.2 Data Collection Daemon

### Collector Service

```python
#!/usr/bin/env python3
# collector/daemon.py

import routeros_api
import time
import logging
import json
from datetime import datetime
from typing import Dict, List
from influxdb_client import InfluxDBClient, Point
from influxdb_client.client.write_api import SYNCHRONOUS

logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s [%(levelname)s] %(name)s: %(message)s',
    handlers=[
        logging.StreamHandler(),
        logging.FileHandler('collector.log'),
    ]
)
logger = logging.getLogger('collector')


class RouterCollector:
    """Collect metrics จาก single router"""
    
    def __init__(self, router_config: Dict):
        self.config = router_config
        self.api_pool = None
        self.api = None
        self.router_id = router_config['id']
        self.router_name = router_config['name']
    
    def connect(self):
        self.api_pool = routeros_api.RouterOsApiPool(
            self.config['host'],
            username=self.config.get('username', 'admin'),
            password=self.config.get('password', ''),
            port=self.config.get('port', 8728),
        )
        self.api = self.api_pool.get_api()
        logger.info(f"Connected to {self.router_name} ({self.config['host']})")
    
    def disconnect(self):
        if self.api_pool:
            try:
                self.api_pool.disconnect()
            except Exception:
                pass
    
    def collect_all(self) -> Dict:
        """เก็บข้อมูลทั้งหมดจาก router"""
        timestamp = datetime.utcnow()
        
        try:
            data = {
                'timestamp': timestamp,
                'router_id': self.router_id,
                'router_name': self.router_name,
            }
            
            # System resources
            resources = self.api.get_resource('/system/resource').get()
            if resources:
                r = resources[0]
                data['system'] = {
                    'cpu_load': int(r.get('cpu-load', 0)),
                    'free_memory': int(r.get('free-memory', 0)),
                    'total_memory': int(r.get('total-memory', 0)),
                    'free_hdd': int(r.get('free-hdd-space', 0)),
                    'total_hdd': int(r.get('total-hdd-space', 0)),
                    'uptime': r.get('uptime', '0s'),
                    'version': r.get('version', ''),
                }
            
            # Interface stats
            interfaces = self.api.get_resource('/interface').get()
            data['interfaces'] = []
            for iface in interfaces:
                if iface.get('running') == 'true':
                    data['interfaces'].append({
                        'name': iface['name'],
                        'type': iface.get('type', 'unknown'),
                        'rx_byte': int(iface.get('rx-byte', 0)),
                        'tx_byte': int(iface.get('tx-byte', 0)),
                        'rx_packet': int(iface.get('rx-packet', 0)),
                        'tx_packet': int(iface.get('tx-packet', 0)),
                        'rx_error': int(iface.get('rx-error', 0)),
                        'tx_error': int(iface.get('tx-error', 0)),
                        'rx_drop': int(iface.get('rx-drop', 0)),
                        'tx_drop': int(iface.get('tx-drop', 0)),
                    })
            
            # Hotspot users
            try:
                hotspot = self.api.get_resource('/ip/hotspot/active').get()
                data['hotspot_users'] = len(hotspot)
            except Exception:
                data['hotspot_users'] = 0
            
            # BGP peers (if configured)
            try:
                bgp_peers = self.api.get_resource('/routing/bgp/peer').get()
                data['bgp_peers'] = {
                    'total': len(bgp_peers),
                    'established': sum(1 for p in bgp_peers if p.get('state') == 'established'),
                }
            except Exception:
                pass
            
            return data
            
        except Exception as e:
            logger.error(f"Collection failed for {self.router_name}: {e}")
            raise


class InfluxDBWriter:
    """เขียน metrics ไปยัง InfluxDB"""
    
    def __init__(self, url: str, token: str, org: str, bucket: str):
        self.client = InfluxDBClient(url=url, token=token, org=org)
        self.write_api = self.client.write_api(write_options=SYNCHRONOUS)
        self.org = org
        self.bucket = bucket
    
    def write_metrics(self, data: Dict):
        points = []
        
        router_name = data['router_name']
        timestamp = data['timestamp']
        
        # System metrics
        if 'system' in data:
            sys = data['system']
            
            # CPU
            cpu_point = (
                Point("system_metrics")
                .tag("router", router_name)
                .tag("metric", "cpu")
                .field("cpu_load", sys['cpu_load'])
                .time(timestamp)
            )
            points.append(cpu_point)
            
            # Memory
            if sys['total_memory'] > 0:
                mem_used = sys['total_memory'] - sys['free_memory']
                mem_pct = (mem_used / sys['total_memory']) * 100
                
                mem_point = (
                    Point("system_metrics")
                    .tag("router", router_name)
                    .tag("metric", "memory")
                    .field("free_bytes", sys['free_memory'])
                    .field("total_bytes", sys['total_memory'])
                    .field("used_pct", round(mem_pct, 2))
                    .time(timestamp)
                )
                points.append(mem_point)
        
        # Interface metrics
        for iface in data.get('interfaces', []):
            iface_point = (
                Point("interface_metrics")
                .tag("router", router_name)
                .tag("interface", iface['name'])
                .tag("type", iface['type'])
                .field("rx_byte", iface['rx_byte'])
                .field("tx_byte", iface['tx_byte'])
                .field("rx_packet", iface['rx_packet'])
                .field("tx_packet", iface['tx_packet'])
                .field("rx_error", iface['rx_error'])
                .field("tx_error", iface['tx_error'])
                .time(timestamp)
            )
            points.append(iface_point)
        
        # Hotspot users
        if 'hotspot_users' in data:
            user_point = (
                Point("hotspot_metrics")
                .tag("router", router_name)
                .field("active_users", data['hotspot_users'])
                .time(timestamp)
            )
            points.append(user_point)
        
        # Write to InfluxDB
        self.write_api.write(bucket=self.bucket, org=self.org, record=points)
        logger.debug(f"Written {len(points)} points for {router_name}")
    
    def close(self):
        self.client.close()


class CollectorDaemon:
    """Main daemon ที่รัน collectors"""
    
    def __init__(self, config_file: str = 'config.json'):
        with open(config_file) as f:
            self.config = json.load(f)
        
        self.influx = InfluxDBWriter(
            url=self.config['influxdb']['url'],
            token=self.config['influxdb']['token'],
            org=self.config['influxdb']['org'],
            bucket=self.config['influxdb']['bucket'],
        )
        
        self.collectors = {}
        self.running = False
    
    def setup_collectors(self):
        for router_cfg in self.config['routers']:
            collector = RouterCollector(router_cfg)
            try:
                collector.connect()
                self.collectors[router_cfg['id']] = collector
                logger.info(f"Collector ready: {router_cfg['name']}")
            except Exception as e:
                logger.error(f"Failed to setup collector for {router_cfg['name']}: {e}")
    
    def run(self, interval: int = 30):
        self.running = True
        self.setup_collectors()
        
        logger.info(f"Daemon started with {len(self.collectors)} collectors")
        logger.info(f"Collection interval: {interval}s")
        
        while self.running:
            start = time.time()
            
            for router_id, collector in list(self.collectors.items()):
                try:
                    data = collector.collect_all()
                    self.influx.write_metrics(data)
                    logger.debug(f"Collected from {collector.router_name}")
                    
                except Exception as e:
                    logger.error(f"Collection error for router {router_id}: {e}")
                    # Try to reconnect
                    try:
                        collector.disconnect()
                        collector.connect()
                    except Exception:
                        pass
            
            # Sleep ให้ครบ interval
            elapsed = time.time() - start
            sleep_time = max(0, interval - elapsed)
            time.sleep(sleep_time)
    
    def stop(self):
        self.running = False
        for collector in self.collectors.values():
            collector.disconnect()
        self.influx.close()
        logger.info("Daemon stopped")


if __name__ == '__main__':
    daemon = CollectorDaemon('config.json')
    try:
        daemon.run(interval=30)
    except KeyboardInterrupt:
        daemon.stop()
```

---

## 57.3 Time-series Database (InfluxDB)

### InfluxDB Setup

```yaml
# docker-compose.yml
version: '3.8'

services:
  influxdb:
    image: influxdb:2.7
    container_name: mikrotik-influxdb
    ports:
      - "8086:8086"
    volumes:
      - influxdb-data:/var/lib/influxdb2
      - influxdb-config:/etc/influxdb2
    environment:
      - DOCKER_INFLUXDB_INIT_MODE=setup
      - DOCKER_INFLUXDB_INIT_USERNAME=admin
      - DOCKER_INFLUXDB_INIT_PASSWORD=password123
      - DOCKER_INFLUXDB_INIT_ORG=myorg
      - DOCKER_INFLUXDB_INIT_BUCKET=mikrotik
      - DOCKER_INFLUXDB_INIT_RETENTION=30d

  grafana:
    image: grafana/grafana:latest
    container_name: mikrotik-grafana
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=admin123
    depends_on:
      - influxdb

volumes:
  influxdb-data:
  influxdb-config:
  grafana-data:
```

### Collector Config

```json
{
  "influxdb": {
    "url": "http://localhost:8086",
    "token": "your-influxdb-token",
    "org": "myorg",
    "bucket": "mikrotik"
  },
  "routers": [
    {
      "id": "router-1",
      "name": "HQ-Router",
      "host": "192.168.1.1",
      "username": "api-monitor",
      "password": "SecurePass123",
      "port": 8728
    },
    {
      "id": "router-2",
      "name": "Branch-Router",
      "host": "192.168.2.1",
      "username": "api-monitor",
      "password": "SecurePass123",
      "port": 8728
    }
  ]
}
```

---

## 57.4 Grafana Integration

### Grafana Provisioning - Datasource

```yaml
# grafana/provisioning/datasources/influxdb.yml
apiVersion: 1

datasources:
  - name: InfluxDB-MikroTik
    type: influxdb
    access: proxy
    url: http://influxdb:8086
    jsonData:
      version: Flux
      organization: myorg
      defaultBucket: mikrotik
    secureJsonData:
      token: your-influxdb-token
```

### Grafana Flux Queries

```flux
// CPU Load Query
from(bucket: "mikrotik")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "system_metrics")
  |> filter(fn: (r) => r.metric == "cpu")
  |> filter(fn: (r) => r.router == "${router}")
  |> filter(fn: (r) => r._field == "cpu_load")
  |> aggregateWindow(every: v.windowPeriod, fn: mean)

// Interface Traffic Query  
from(bucket: "mikrotik")
  |> range(start: v.timeRangeStart, stop: v.timeRangeStop)
  |> filter(fn: (r) => r._measurement == "interface_metrics")
  |> filter(fn: (r) => r.router == "${router}")
  |> filter(fn: (r) => r.interface == "${interface}")
  |> filter(fn: (r) => r._field == "rx_byte" or r._field == "tx_byte")
  |> derivative(unit: 8s, nonNegative: true)
  |> map(fn: (r) => ({r with _value: r._value * 8.0}))
  |> aggregateWindow(every: v.windowPeriod, fn: mean)
```

---

## 57.5 Custom Dashboards

### Grafana Dashboard JSON (excerpt)

```json
{
  "title": "MikroTik Network Overview",
  "uid": "mikrotik-overview",
  "panels": [
    {
      "id": 1,
      "title": "CPU Load",
      "type": "gauge",
      "gridPos": {"h": 8, "w": 6, "x": 0, "y": 0},
      "options": {
        "reduceOptions": {"calcs": ["lastNotNull"]},
        "orientation": "auto",
        "thresholds": {
          "steps": [
            {"color": "green", "value": 0},
            {"color": "yellow", "value": 60},
            {"color": "red", "value": 80}
          ]
        }
      }
    },
    {
      "id": 2,
      "title": "WAN Traffic",
      "type": "timeseries",
      "gridPos": {"h": 10, "w": 12, "x": 6, "y": 0}
    }
  ]
}
```

---

## 57.6 Alert Rules

### InfluxDB Alert Setup (Flux)

```flux
// CPU Alert Task
option task = {
  name: "CPU High Alert",
  every: 1m,
}

from(bucket: "mikrotik")
  |> range(start: -2m)
  |> filter(fn: (r) => r._measurement == "system_metrics" and r.metric == "cpu")
  |> filter(fn: (r) => r._field == "cpu_load")
  |> mean()
  |> filter(fn: (r) => r._value > 80.0)
  |> map(fn: (r) => ({r with _value: r._value, message: "High CPU: ${r.router} at ${r._value}%"}))
  |> to(bucket: "alerts")
```

### Grafana Alert Rules

```yaml
# grafana/provisioning/alerting/alerts.yml
apiVersion: 1

groups:
  - name: MikroTik Alerts
    interval: 1m
    rules:
      - uid: cpu-high-alert
        title: High CPU Usage
        condition: C
        data:
          - refId: A
            queryType: ''
            datasourceUid: influxdb-mikrotik
            model:
              query: |
                from(bucket: "mikrotik")
                  |> range(start: -5m)
                  |> filter(fn: (r) => r._measurement == "system_metrics")
                  |> filter(fn: (r) => r.metric == "cpu")
                  |> mean()
        noDataState: NoData
        execErrState: Alerting
        for: 5m
        annotations:
          summary: "CPU usage > 80% on {{$labels.router}}"
        labels:
          severity: warning
```

---

## 57.7 Historical Data Analysis

### Analysis Script

```python
#!/usr/bin/env python3
# analysis/historical.py

from influxdb_client import InfluxDBClient
import pandas as pd
import matplotlib.pyplot as plt
from datetime import datetime, timedelta

class HistoricalAnalyzer:
    def __init__(self, url, token, org, bucket):
        self.client = InfluxDBClient(url=url, token=token, org=org)
        self.query_api = self.client.query_api()
        self.bucket = bucket
    
    def get_cpu_history(self, router: str, days: int = 7) -> pd.DataFrame:
        """ดึง CPU history"""
        query = f'''
        from(bucket: "{self.bucket}")
          |> range(start: -{days}d)
          |> filter(fn: (r) => r._measurement == "system_metrics")
          |> filter(fn: (r) => r.metric == "cpu")
          |> filter(fn: (r) => r.router == "{router}")
          |> filter(fn: (r) => r._field == "cpu_load")
          |> aggregateWindow(every: 1h, fn: mean)
        '''
        
        result = self.query_api.query_data_frame(query)
        if result.empty:
            return pd.DataFrame()
        
        result['_time'] = pd.to_datetime(result['_time'])
        result = result.rename(columns={'_value': 'cpu_load', '_time': 'timestamp'})
        return result[['timestamp', 'cpu_load']].sort_values('timestamp')
    
    def generate_capacity_report(self, router: str) -> dict:
        """วิเคราะห์ capacity planning"""
        cpu_df = self.get_cpu_history(router, days=30)
        
        if cpu_df.empty:
            return {'error': 'No data available'}
        
        return {
            'router': router,
            'period': '30 days',
            'cpu': {
                'avg': round(cpu_df['cpu_load'].mean(), 2),
                'max': round(cpu_df['cpu_load'].max(), 2),
                'p95': round(cpu_df['cpu_load'].quantile(0.95), 2),
                'trend': 'increasing' if cpu_df['cpu_load'].iloc[-1] > cpu_df['cpu_load'].mean() else 'stable',
            }
        }
    
    def plot_cpu_trend(self, router: str, days: int = 7, output: str = 'cpu_trend.png'):
        """สร้าง CPU trend chart"""
        df = self.get_cpu_history(router, days)
        
        if df.empty:
            return
        
        fig, ax = plt.subplots(figsize=(12, 6))
        
        ax.plot(df['timestamp'], df['cpu_load'], linewidth=2, color='#3b82f6', label='CPU Load')
        ax.axhline(y=80, color='red', linestyle='--', alpha=0.7, label='Warning (80%)')
        ax.axhline(y=df['cpu_load'].mean(), color='green', linestyle='--', alpha=0.7, 
                   label=f'Average ({df["cpu_load"].mean():.1f}%)')
        
        ax.fill_between(df['timestamp'], df['cpu_load'], alpha=0.1, color='#3b82f6')
        ax.set_xlabel('Time')
        ax.set_ylabel('CPU Load (%)')
        ax.set_title(f'CPU Load Trend - {router} (Last {days} days)')
        ax.legend()
        ax.set_ylim(0, 100)
        ax.grid(True, alpha=0.3)
        
        plt.tight_layout()
        plt.savefig(output, dpi=100, bbox_inches='tight')
        print(f"Chart saved: {output}")
```

---

## 57.8 Capacity Planning Features

### Capacity Planning Script

```python
class CapacityPlanner:
    def __init__(self, analyzer: HistoricalAnalyzer):
        self.analyzer = analyzer
    
    def predict_cpu_saturation(self, router: str) -> dict:
        """ทำนายว่าเมื่อไหร่ CPU จะถึง 90%"""
        df = self.analyzer.get_cpu_history(router, days=30)
        
        if df.empty or len(df) < 10:
            return {'prediction': 'insufficient_data'}
        
        # Linear regression
        import numpy as np
        
        x = np.arange(len(df))
        y = df['cpu_load'].values
        
        coefficients = np.polyfit(x, y, 1)
        slope = coefficients[0]
        intercept = coefficients[1]
        
        current_value = y[-1]
        threshold = 90.0
        
        if slope <= 0:
            return {
                'prediction': 'stable',
                'current': round(current_value, 1),
                'trend': 'decreasing',
                'message': 'CPU load is stable or decreasing'
            }
        
        # คำนวณว่าต้องใช้เวลากี่ชั่วโมงจึงถึง threshold
        data_points_needed = (threshold - current_value) / slope
        hours_until_saturation = (data_points_needed * 1)  # assuming 1 point/hour
        
        return {
            'prediction': 'warning' if hours_until_saturation < 168 else 'ok',
            'current': round(current_value, 1),
            'trend': 'increasing',
            'slope': round(slope * 24, 4),  # per day
            'hours_to_saturation': round(hours_until_saturation),
            'message': f"CPU will reach 90% in approximately {round(hours_until_saturation/24, 1)} days"
        }
```

---

## 57.9 Mobile-responsive Design

### Responsive Grafana Dashboard

```json
{
  "templating": {
    "list": [
      {
        "name": "router",
        "type": "query",
        "label": "Router",
        "query": "import \"influxdata/influxdb/schema\"\nschema.tagValues(bucket: \"mikrotik\", tag: \"router\")"
      },
      {
        "name": "interface",
        "type": "query", 
        "label": "Interface"
      }
    ]
  },
  "panels": [],
  "schemaVersion": 38,
  "style": "dark",
  "tags": ["mikrotik", "network"]
}
```

---

## 57.10 Lab: Complete Monitoring Stack

### Lab Setup

```bash
#!/bin/bash
# setup_monitoring_stack.sh

echo "Setting up Complete MikroTik Monitoring Stack..."

# 1. สร้าง directory structure
mkdir -p monitoring/{collector,grafana/provisioning/{datasources,dashboards,alerting}}
cd monitoring

# 2. docker-compose.yml
cat > docker-compose.yml << 'COMPOSE'
version: '3.8'
services:
  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"
    volumes:
      - influxdb-data:/var/lib/influxdb2
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: password123
      DOCKER_INFLUXDB_INIT_ORG: myorg
      DOCKER_INFLUXDB_INIT_BUCKET: mikrotik
      DOCKER_INFLUXDB_INIT_RETENTION: 30d
      DOCKER_INFLUXDB_INIT_ADMIN_TOKEN: mytoken123

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana-data:/var/lib/grafana
      - ./grafana/provisioning:/etc/grafana/provisioning
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123
    depends_on:
      - influxdb

volumes:
  influxdb-data:
  grafana-data:
COMPOSE

# 3. Collector requirements
cat > collector/requirements.txt << 'REQ'
routeros-api>=0.1.3
influxdb-client>=3.0.0
pandas>=2.0.0
REQ

# 4. Collector config
cat > collector/config.json << 'CONFIG'
{
  "influxdb": {
    "url": "http://localhost:8086",
    "token": "mytoken123",
    "org": "myorg",
    "bucket": "mikrotik"
  },
  "collection_interval": 30,
  "routers": [
    {
      "id": "router-1",
      "name": "Main-Router",
      "host": "192.168.1.1",
      "username": "admin",
      "password": ""
    }
  ]
}
CONFIG

# 5. Start stack
docker-compose up -d

echo ""
echo "Stack started!"
echo "InfluxDB: http://localhost:8086 (admin/password123)"
echo "Grafana:  http://localhost:3000 (admin/admin123)"
echo ""
echo "Next steps:"
echo "1. Configure datasource in Grafana"
echo "2. Start collector: cd collector && python3 daemon.py"
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Architecture | Collector → InfluxDB → Grafana |
| Collector | Python daemon สำหรับ data collection |
| InfluxDB | Time-series database setup |
| Grafana | Dashboard, alerts provisioning |
| Flux Queries | Time-series data analysis |
| Historical Analysis | Pandas สำหรับ trend analysis |
| Capacity Planning | CPU saturation prediction |
| Alerts | Threshold-based notifications |
| Docker | Complete stack deployment |

---

[← Part 56: Web Dashboard Part 2](part-056-web-dashboard-2.md) | [Part 58: User Portal →](part-058-user-portal.md)
