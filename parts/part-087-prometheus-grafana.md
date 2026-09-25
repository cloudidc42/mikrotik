# Part 87: Prometheus และ Grafana สำหรับ MikroTik

## สารบัญ
1. [Monitoring Stack Overview](#overview)
2. [SNMP Exporter](#snmp-exporter)
3. [Custom MikroTik Exporter](#custom-exporter)
4. [Prometheus Configuration](#prometheus)
5. [Grafana Dashboards](#grafana)
6. [Alerting with AlertManager](#alertmanager)
7. [Lab: Full Monitoring Stack](#lab)

---

## 1. Monitoring Stack Overview {#overview}

```
[MikroTik Router]
    │ SNMP / API
    ↓
[SNMP Exporter / Custom Exporter]
    │ HTTP /metrics
    ↓
[Prometheus]
    │ Query
    ↓
[Grafana] ── [AlertManager] ── [Email/Slack/PagerDuty]
```

---

## 2. SNMP Exporter {#snmp-exporter}

```yaml
# docker-compose.yml
version: '3.8'

services:
  prometheus:
    image: prom/prometheus:latest
    ports:
      - "9090:9090"
    volumes:
      - ./prometheus.yml:/etc/prometheus/prometheus.yml
      - prom_data:/prometheus

  snmp-exporter:
    image: prom/snmp-exporter:latest
    ports:
      - "9116:9116"
    volumes:
      - ./snmp.yml:/etc/snmp_exporter/snmp.yml

  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
    volumes:
      - grafana_data:/var/lib/grafana
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin123

  alertmanager:
    image: prom/alertmanager:latest
    ports:
      - "9093:9093"
    volumes:
      - ./alertmanager.yml:/etc/alertmanager/alertmanager.yml

volumes:
  prom_data:
  grafana_data:
```

```yaml
# snmp.yml - SNMP exporter config สำหรับ MikroTik
modules:
  mikrotik:
    walk:
      - 1.3.6.1.2.1.1          # system
      - 1.3.6.1.2.1.2.2        # ifTable
      - 1.3.6.1.2.1.31.1.1     # ifXTable
      - 1.3.6.1.4.1.14988.1    # MikroTik specific
    
    metrics:
      - name: sysUpTime
        oid: 1.3.6.1.2.1.1.3.0
        type: gauge
        help: System uptime
      
      - name: ifHCInOctets
        oid: 1.3.6.1.2.1.31.1.1.1.6
        type: counter
        help: Interface inbound bytes
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels:
              - ifIndex
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
      
      - name: ifHCOutOctets
        oid: 1.3.6.1.2.1.31.1.1.1.10
        type: counter
        help: Interface outbound bytes
        indexes:
          - labelname: ifIndex
            type: gauge
        lookups:
          - labels:
              - ifIndex
            labelname: ifName
            oid: 1.3.6.1.2.1.31.1.1.1.1
            type: DisplayString
    
    auth:
      community: public
      version: 2
```

```bash
# MikroTik SNMP configuration
/snmp set enabled=yes community=public location="DataCenter" contact="NOC"
/snmp community
set public name=public addresses=10.0.0.0/24 security=none read-access=yes
```

---

## 3. Custom MikroTik Exporter {#custom-exporter}

```python
# custom_exporter.py - MikroTik metrics exporter สำหรับ Prometheus
from prometheus_client import start_http_server, Gauge, Counter, Info
import librouteros
import time
import threading
from typing import Dict, List

# Prometheus metrics
interface_rx_bytes = Counter('mikrotik_interface_rx_bytes_total',
                              'Interface RX bytes', ['router', 'interface'])
interface_tx_bytes = Counter('mikrotik_interface_tx_bytes_total',
                              'Interface TX bytes', ['router', 'interface'])
interface_rx_bps = Gauge('mikrotik_interface_rx_bps',
                          'Interface RX bits per second', ['router', 'interface'])
interface_tx_bps = Gauge('mikrotik_interface_tx_bps',
                          'Interface TX bits per second', ['router', 'interface'])
interface_errors = Counter('mikrotik_interface_rx_errors_total',
                            'Interface RX errors', ['router', 'interface'])
router_uptime = Gauge('mikrotik_uptime_seconds',
                       'Router uptime in seconds', ['router'])
router_cpu = Gauge('mikrotik_cpu_load_percent',
                    'CPU load percentage', ['router'])
router_memory = Gauge('mikrotik_memory_used_bytes',
                       'Memory used bytes', ['router'])
router_memory_free = Gauge('mikrotik_memory_free_bytes',
                            'Memory free bytes', ['router'])
active_connections = Gauge('mikrotik_active_connections',
                            'Active NAT/firewall connections', ['router'])
bgp_peer_up = Gauge('mikrotik_bgp_peer_up',
                     'BGP peer state (1=established)', ['router', 'peer', 'remote_as'])
ppp_active_sessions = Gauge('mikrotik_ppp_active_sessions',
                             'Active PPPoE/PPP sessions', ['router'])


class RouterExporter:
    def __init__(self, host: str, username: str, password: str, 
                 name: str = None):
        self.host = host
        self.username = username
        self.password = password
        self.name = name or host
    
    def connect(self):
        return librouteros.connect(
            host=self.host,
            username=self.username,
            password=self.password,
            timeout=10
        )
    
    def collect_metrics(self):
        try:
            conn = self.connect()
            
            # System resource
            resource = list(conn('/system/resource/print'))[0]
            r = dict(resource)
            
            uptime_str = r.get('uptime', '0s')
            uptime_sec = self._parse_uptime(uptime_str)
            router_uptime.labels(router=self.name).set(uptime_sec)
            router_cpu.labels(router=self.name).set(float(r.get('cpu-load', 0)))
            
            mem_used = int(r.get('total-memory', 0)) - int(r.get('free-memory', 0))
            router_memory.labels(router=self.name).set(mem_used)
            router_memory_free.labels(router=self.name).set(int(r.get('free-memory', 0)))
            
            # Interfaces
            ifaces = list(conn('/interface/print'))
            for iface in ifaces:
                i = dict(iface)
                name = i.get('name', '')
                
                interface_rx_bps.labels(router=self.name, interface=name).set(
                    int(i.get('rx-bits-per-second', 0))
                )
                interface_tx_bps.labels(router=self.name, interface=name).set(
                    int(i.get('tx-bits-per-second', 0))
                )
            
            # Connection tracking
            try:
                conn_count = list(conn('/ip/firewall/connection/print', count_only=''))
                if conn_count:
                    active_connections.labels(router=self.name).set(
                        int(str(conn_count[0]))
                    )
            except:
                pass
            
            # BGP peers
            try:
                peers = list(conn('/routing/bgp/peer/print'))
                for peer in peers:
                    p = dict(peer)
                    is_up = 1 if p.get('state') == 'established' else 0
                    bgp_peer_up.labels(
                        router=self.name,
                        peer=p.get('name', ''),
                        remote_as=p.get('remote-as', '')
                    ).set(is_up)
            except:
                pass
            
            # PPP sessions
            try:
                ppp_sessions = list(conn('/ppp/active/print'))
                ppp_active_sessions.labels(router=self.name).set(len(ppp_sessions))
            except:
                pass
            
            conn.close()
        
        except Exception as e:
            print(f"Error collecting from {self.name}: {e}")
    
    def _parse_uptime(self, uptime_str: str) -> int:
        """Convert uptime string to seconds"""
        import re
        total = 0
        matches = re.findall(r'(\d+)([wdhms])', uptime_str)
        units = {'w': 604800, 'd': 86400, 'h': 3600, 'm': 60, 's': 1}
        for value, unit in matches:
            total += int(value) * units.get(unit, 0)
        return total


class MultiRouterExporter:
    def __init__(self, routers: List[Dict]):
        self.exporters = [
            RouterExporter(
                host=r['host'],
                username=r['username'],
                password=r['password'],
                name=r.get('name', r['host'])
            )
            for r in routers
        ]
    
    def collect_all(self):
        threads = [threading.Thread(target=e.collect_metrics) for e in self.exporters]
        for t in threads:
            t.start()
        for t in threads:
            t.join()
    
    def start(self, port: int = 8000, interval: int = 15):
        start_http_server(port)
        print(f"Exporter running on port {port}")
        
        while True:
            self.collect_all()
            time.sleep(interval)


if __name__ == "__main__":
    routers = [
        {"host": "10.0.0.1", "username": "admin", "password": "pass", "name": "core-01"},
        {"host": "10.0.0.2", "username": "admin", "password": "pass", "name": "core-02"},
    ]
    
    exporter = MultiRouterExporter(routers)
    exporter.start(port=8000, interval=15)
```

---

## 4. Prometheus Configuration {#prometheus}

```yaml
# prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

rule_files:
  - "alerts/*.yml"

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

scrape_configs:
  # Custom MikroTik exporter
  - job_name: 'mikrotik-custom'
    static_configs:
      - targets: ['mikrotik-exporter:8000']
    
  # SNMP exporter
  - job_name: 'mikrotik-snmp'
    static_configs:
      - targets:
          - '10.0.0.1'
          - '10.0.0.2'
    metrics_path: /snmp
    params:
      module: [mikrotik]
    relabel_configs:
      - source_labels: [__address__]
        target_label: __param_target
      - source_labels: [__param_target]
        target_label: instance
      - target_label: __address__
        replacement: snmp-exporter:9116
```

---

## 5. Grafana Dashboards {#grafana}

```json
// Dashboard JSON (สำหรับ import)
{
  "title": "MikroTik Network Overview",
  "panels": [
    {
      "title": "Interface Traffic",
      "type": "timeseries",
      "targets": [
        {
          "expr": "rate(mikrotik_interface_rx_bytes_total{router=\"core-01\"}[5m]) * 8",
          "legendFormat": "RX {{interface}}"
        },
        {
          "expr": "rate(mikrotik_interface_tx_bytes_total{router=\"core-01\"}[5m]) * 8",
          "legendFormat": "TX {{interface}}"
        }
      ]
    },
    {
      "title": "CPU Load",
      "type": "gauge",
      "targets": [
        {
          "expr": "mikrotik_cpu_load_percent",
          "legendFormat": "{{router}}"
        }
      ]
    },
    {
      "title": "Active Connections",
      "type": "stat",
      "targets": [
        {
          "expr": "mikrotik_active_connections",
          "legendFormat": "{{router}}"
        }
      ]
    },
    {
      "title": "BGP Peer Status",
      "type": "table",
      "targets": [
        {
          "expr": "mikrotik_bgp_peer_up",
          "legendFormat": "{{router}} - {{peer}}"
        }
      ]
    }
  ]
}
```

---

## 6. Alerting with AlertManager {#alertmanager}

```yaml
# alerts/mikrotik.yml
groups:
  - name: mikrotik
    rules:
      - alert: RouterDown
        expr: up{job="mikrotik-custom"} == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Router {{ $labels.instance }} is down"
          description: "Cannot collect metrics from {{ $labels.instance }}"
      
      - alert: HighCPULoad
        expr: mikrotik_cpu_load_percent > 80
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High CPU on {{ $labels.router }}: {{ $value }}%"
      
      - alert: BGPPeerDown
        expr: mikrotik_bgp_peer_up == 0
        for: 2m
        labels:
          severity: critical
        annotations:
          summary: "BGP peer {{ $labels.peer }} down on {{ $labels.router }}"
      
      - alert: HighInterfaceTraffic
        expr: rate(mikrotik_interface_rx_bytes_total[5m]) * 8 > 900000000
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "Interface {{ $labels.interface }} near capacity"


# alertmanager.yml
global:
  smtp_smarthost: 'smtp.gmail.com:587'
  smtp_from: 'alerts@company.com'

route:
  group_by: ['alertname', 'router']
  group_wait: 30s
  group_interval: 5m
  repeat_interval: 12h
  receiver: 'noc-team'
  routes:
    - match:
        severity: critical
      receiver: 'pagerduty-critical'

receivers:
  - name: 'noc-team'
    email_configs:
      - to: 'noc@company.com'
        subject: '[ALERT] {{ .GroupLabels.alertname }}'
        body: |
          {{ range .Alerts }}
          Alert: {{ .Annotations.summary }}
          Details: {{ .Annotations.description }}
          {{ end }}
  
  - name: 'pagerduty-critical'
    pagerduty_configs:
      - routing_key: 'your-pagerduty-key'
        description: '{{ .CommonAnnotations.summary }}'
```

---

## 7. Lab: Full Monitoring Stack {#lab}

### Lab Steps

```bash
# Step 1: Configure SNMP on MikroTik
/snmp set enabled=yes
/snmp community set public addresses=0.0.0.0/0

# Step 2: Start monitoring stack
docker-compose up -d

# Step 3: Verify SNMP
curl "http://localhost:9116/snmp?target=192.168.1.1&module=mikrotik"

# Step 4: Verify custom exporter
curl http://localhost:8000/metrics | grep mikrotik_

# Step 5: Verify Prometheus
curl http://localhost:9090/api/v1/targets

# Step 6: Import Grafana dashboard
# http://localhost:3000 → Dashboard → Import → paste JSON
```

### Verification Checklist

- [ ] SNMP enabled บน MikroTik
- [ ] SNMP exporter ดึงค่าได้
- [ ] Custom exporter metrics ปรากฏ
- [ ] Prometheus scrapes targets
- [ ] Grafana dashboards แสดง metrics
- [ ] Alerts fire เมื่อ threshold exceeded

---

## Summary

Part นี้ครอบคลุม:
- **SNMP Exporter** สำหรับ MikroTik
- **Custom Python Exporter** ด้วย librouteros
- **Prometheus** configuration
- **Grafana dashboards** สำหรับ network metrics
- **AlertManager** สำหรับ notifications

---

[← Part 86: ELK Logging](part-086-elk-logging.md) | [Part 88: CI/CD →](part-088-cicd.md)
