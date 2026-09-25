# Part 95: Real-time Network Analytics

## สารบัญ
1. [Stream Processing Architecture](#architecture)
2. [Kafka for Network Events](#kafka)
3. [Stream Processing with Flink](#flink)
4. [Time-Series Analysis](#timeseries)
5. [Real-time Dashboard](#dashboard)
6. [Lab: Streaming Analytics](#lab)

---

## 1. Stream Processing Architecture {#architecture}

```
[MikroTik Routers]
    │ NetFlow/SNMP/API
    ↓
[Kafka Topics]
  ├── network.flows
  ├── network.metrics
  └── network.alerts
    │
    ↓
[Stream Processor (Flink/Kafka Streams)]
    │ Real-time aggregation
    ↓
[Time-Series DB (InfluxDB)]
    │
    ↓
[Grafana Real-time Dashboard]
```

---

## 2. Kafka for Network Events {#kafka}

```yaml
# docker-compose.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.4.0
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
  
  kafka:
    image: confluentinc/cp-kafka:7.4.0
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_LOG_RETENTION_HOURS: 24
  
  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: admin123
      DOCKER_INFLUXDB_INIT_ORG: myorg
      DOCKER_INFLUXDB_INIT_BUCKET: network
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3000:3000"
```

```python
# kafka_producer.py - Network metrics ส่งเข้า Kafka
import json
import time
import librouteros
from kafka import KafkaProducer
from datetime import datetime
from typing import List, Dict

class NetworkMetricsProducer:
    def __init__(self, kafka_bootstrap: str = "localhost:9092"):
        self.producer = KafkaProducer(
            bootstrap_servers=[kafka_bootstrap],
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            key_serializer=lambda k: k.encode('utf-8') if k else None
        )
    
    def produce_interface_metrics(self, router_host: str, username: str, 
                                   password: str):
        """ส่ง interface metrics ไป Kafka"""
        conn = librouteros.connect(host=router_host, username=username, password=password)
        
        interfaces = list(conn('/interface/print'))
        
        for iface in interfaces:
            i = dict(iface)
            name = i.get("name", "")
            
            metric = {
                "timestamp": datetime.utcnow().isoformat(),
                "router": router_host,
                "interface": name,
                "rx_bps": int(i.get("rx-bits-per-second", 0)),
                "tx_bps": int(i.get("tx-bits-per-second", 0)),
                "rx_pps": int(i.get("rx-packets-per-second", 0)),
                "tx_pps": int(i.get("tx-packets-per-second", 0)),
                "rx_errors": int(i.get("rx-errors", 0)),
                "tx_errors": int(i.get("tx-errors", 0))
            }
            
            self.producer.send(
                topic="network.metrics",
                key=f"{router_host}_{name}",
                value=metric
            )
        
        conn.close()
        self.producer.flush()
    
    def produce_system_metrics(self, router_host: str, username: str, 
                                password: str):
        """ส่ง system resource metrics"""
        conn = librouteros.connect(host=router_host, username=username, password=password)
        
        resource = dict(list(conn('/system/resource/print'))[0])
        
        metric = {
            "timestamp": datetime.utcnow().isoformat(),
            "router": router_host,
            "cpu_load": int(resource.get("cpu-load", 0)),
            "free_memory": int(resource.get("free-memory", 0)),
            "total_memory": int(resource.get("total-memory", 0)),
            "uptime": resource.get("uptime", "")
        }
        
        self.producer.send(
            topic="network.system",
            key=router_host,
            value=metric
        )
        
        conn.close()
        self.producer.flush()


def run_producer(routers: list, interval: int = 10):
    producer = NetworkMetricsProducer()
    
    while True:
        for router in routers:
            try:
                producer.produce_interface_metrics(**router)
                producer.produce_system_metrics(**router)
            except Exception as e:
                print(f"Error producing from {router['router_host']}: {e}")
        
        time.sleep(interval)


if __name__ == "__main__":
    routers = [
        {"router_host": "10.0.0.1", "username": "admin", "password": "pass"},
        {"router_host": "10.0.0.2", "username": "admin", "password": "pass"}
    ]
    run_producer(routers)
```

---

## 3. Stream Processing with Flink {#flink}

```python
# kafka_stream_processor.py - Process network metrics
from kafka import KafkaConsumer, KafkaProducer
import json
from collections import defaultdict, deque
from datetime import datetime, timedelta
import threading
import time

class NetworkStreamProcessor:
    """Process real-time network metrics stream"""
    
    def __init__(self, kafka_bootstrap: str = "localhost:9092",
                 window_seconds: int = 60):
        self.window_seconds = window_seconds
        self.windows: defaultdict = defaultdict(lambda: deque())
        
        self.consumer = KafkaConsumer(
            "network.metrics",
            bootstrap_servers=[kafka_bootstrap],
            value_deserializer=lambda v: json.loads(v.decode('utf-8')),
            group_id="stream-processor",
            auto_offset_reset="latest"
        )
        
        self.alert_producer = KafkaProducer(
            bootstrap_servers=[kafka_bootstrap],
            value_serializer=lambda v: json.dumps(v).encode('utf-8')
        )
    
    def process_metric(self, metric: dict):
        """Process single metric และ update windows"""
        key = f"{metric['router']}_{metric['interface']}"
        window = self.windows[key]
        
        now = datetime.utcnow()
        
        # Add to window
        window.append({
            "timestamp": now,
            "rx_bps": metric["rx_bps"],
            "tx_bps": metric["tx_bps"]
        })
        
        # Remove old entries
        cutoff = now - timedelta(seconds=self.window_seconds)
        while window and window[0]["timestamp"] < cutoff:
            window.popleft()
        
        # Aggregations
        if len(window) >= 2:
            rx_values = [e["rx_bps"] for e in window]
            tx_values = [e["tx_bps"] for e in window]
            
            aggregated = {
                "key": key,
                "window_seconds": self.window_seconds,
                "count": len(window),
                "rx_avg_bps": sum(rx_values) / len(rx_values),
                "tx_avg_bps": sum(tx_values) / len(tx_values),
                "rx_max_bps": max(rx_values),
                "tx_max_bps": max(tx_values),
                "timestamp": now.isoformat()
            }
            
            # Check for anomalies
            self._check_anomalies(aggregated, metric)
            
            return aggregated
        
        return None
    
    def _check_anomalies(self, aggregated: dict, current: dict):
        """ตรวจสอบ anomalies และส่ง alerts"""
        avg_rx = aggregated["rx_avg_bps"]
        max_rx = aggregated["rx_max_bps"]
        current_rx = current["rx_bps"]
        
        # Spike detection: current > 2x average
        if avg_rx > 0 and current_rx > avg_rx * 2:
            alert = {
                "type": "traffic_spike",
                "key": aggregated["key"],
                "current_bps": current_rx,
                "avg_bps": avg_rx,
                "ratio": round(current_rx / avg_rx, 2),
                "severity": "warning" if current_rx < avg_rx * 5 else "critical",
                "timestamp": datetime.utcnow().isoformat()
            }
            
            self.alert_producer.send("network.alerts", value=alert)
    
    def run(self):
        """Main processing loop"""
        print("Stream processor started")
        
        for message in self.consumer:
            metric = message.value
            result = self.process_metric(metric)
            
            if result:
                # Write to InfluxDB (simplified)
                self._write_to_influxdb(result)
    
    def _write_to_influxdb(self, data: dict):
        """Write aggregated data to InfluxDB"""
        try:
            from influxdb_client import InfluxDBClient, Point
            from influxdb_client.client.write_api import SYNCHRONOUS
            
            client = InfluxDBClient(url="http://localhost:8086", token="mytoken", org="myorg")
            write_api = client.write_api(write_options=SYNCHRONOUS)
            
            key_parts = data["key"].split("_", 1)
            router = key_parts[0] if len(key_parts) > 0 else "unknown"
            interface = key_parts[1] if len(key_parts) > 1 else "unknown"
            
            point = (Point("interface_metrics")
                     .tag("router", router)
                     .tag("interface", interface)
                     .field("rx_avg_bps", data["rx_avg_bps"])
                     .field("tx_avg_bps", data["tx_avg_bps"])
                     .field("rx_max_bps", data["rx_max_bps"]))
            
            write_api.write(bucket="network", record=point)
            client.close()
        except Exception as e:
            pass  # Continue even if write fails


if __name__ == "__main__":
    processor = NetworkStreamProcessor()
    processor.run()
```

---

## 4. Time-Series Analysis {#timeseries}

```python
# time_series_analysis.py
from influxdb_client import InfluxDBClient
import pandas as pd
import numpy as np
from datetime import datetime, timedelta

class NetworkTimeSeriesAnalyzer:
    def __init__(self, influx_url: str, token: str, org: str):
        self.client = InfluxDBClient(url=influx_url, token=token, org=org)
        self.query_api = self.client.query_api()
    
    def get_traffic_trends(self, router: str, interface: str, 
                            hours: int = 24) -> pd.DataFrame:
        """ดึง traffic trend"""
        query = f'''
        from(bucket: "network")
          |> range(start: -{hours}h)
          |> filter(fn: (r) => r["_measurement"] == "interface_metrics")
          |> filter(fn: (r) => r["router"] == "{router}")
          |> filter(fn: (r) => r["interface"] == "{interface}")
          |> filter(fn: (r) => r["_field"] == "rx_avg_bps" or r["_field"] == "tx_avg_bps")
          |> aggregateWindow(every: 5m, fn: mean)
        '''
        
        result = self.query_api.query_data_frame(query)
        return result if not result.empty else pd.DataFrame()
    
    def detect_trends(self, df: pd.DataFrame) -> dict:
        """วิเคราะห์ traffic trends"""
        if df.empty:
            return {}
        
        # Calculate hourly averages
        df['hour'] = pd.to_datetime(df['_time']).dt.hour
        hourly_avg = df.groupby('hour')['_value'].mean()
        
        # Peak hours
        peak_hour = hourly_avg.idxmax()
        off_peak_hour = hourly_avg.idxmin()
        
        # Growth trend (linear regression)
        x = np.arange(len(df))
        y = df['_value'].values
        slope, intercept = np.polyfit(x, y, 1)
        
        return {
            "peak_hour": int(peak_hour),
            "off_peak_hour": int(off_peak_hour),
            "growth_trend_bps_per_sample": round(slope, 2),
            "trending_up": slope > 0,
            "avg_bps": round(float(df['_value'].mean()), 0),
            "max_bps": round(float(df['_value'].max()), 0)
        }
    
    def forecast_bandwidth(self, router: str, interface: str, 
                            forecast_days: int = 7) -> dict:
        """พยากรณ์ bandwidth ต้องการใน X วัน"""
        df = self.get_traffic_trends(router, interface, hours=168)  # Last 7 days
        
        if df.empty:
            return {"error": "Insufficient data"}
        
        trends = self.detect_trends(df)
        
        # Simple linear extrapolation
        current_avg = trends["avg_bps"]
        growth_rate = trends["growth_trend_bps_per_sample"]
        
        # Assuming 1 sample per 5 minutes: 288 samples per day
        samples_per_day = 288
        forecast_value = current_avg + (growth_rate * samples_per_day * forecast_days)
        
        return {
            "current_avg_mbps": round(current_avg / 1e6, 2),
            "forecast_avg_mbps": round(forecast_value / 1e6, 2),
            "forecast_days": forecast_days,
            "capacity_upgrade_needed": forecast_value > (1e9 * 0.8)  # 80% of 1Gbps
        }
```

---

## 5. Real-time Dashboard {#dashboard}

```python
# realtime_api.py - FastAPI สำหรับ real-time dashboard
from fastapi import FastAPI, WebSocket
from kafka import KafkaConsumer
import asyncio
import json
from datetime import datetime

app = FastAPI(title="Network Real-time Dashboard")

@app.websocket("/ws/metrics")
async def metrics_websocket(websocket: WebSocket):
    """WebSocket endpoint สำหรับ real-time metrics"""
    await websocket.accept()
    
    consumer = KafkaConsumer(
        "network.metrics",
        bootstrap_servers=["localhost:9092"],
        value_deserializer=lambda v: json.loads(v.decode()),
        auto_offset_reset="latest",
        group_id=f"ws-{id(websocket)}"
    )
    
    try:
        while True:
            # Non-blocking poll
            records = consumer.poll(timeout_ms=100, max_records=100)
            
            if records:
                metrics = []
                for tp, msgs in records.items():
                    for msg in msgs:
                        metrics.append(msg.value)
                
                await websocket.send_json({
                    "type": "metrics",
                    "timestamp": datetime.utcnow().isoformat(),
                    "data": metrics[-10:]  # Last 10 metrics
                })
            
            await asyncio.sleep(0.1)
    
    except Exception:
        consumer.close()


@app.get("/api/alerts")
async def get_alerts():
    """ดึง recent alerts"""
    consumer = KafkaConsumer(
        "network.alerts",
        bootstrap_servers=["localhost:9092"],
        value_deserializer=lambda v: json.loads(v.decode()),
        auto_offset_reset="earliest",
        group_id="alert-reader",
        consumer_timeout_ms=2000
    )
    
    alerts = []
    for msg in consumer:
        alerts.append(msg.value)
        if len(alerts) >= 50:
            break
    
    consumer.close()
    return {"alerts": alerts[-20:]}
```

---

## 6. Lab: Streaming Analytics {#lab}

### Lab Steps

```bash
# Step 1: Start infrastructure
docker-compose up -d kafka influxdb grafana

# Step 2: Create Kafka topics
docker exec kafka kafka-topics.sh --create --topic network.metrics --bootstrap-server localhost:9092
docker exec kafka kafka-topics.sh --create --topic network.alerts --bootstrap-server localhost:9092

# Step 3: Start producer
python kafka_producer.py &

# Step 4: Start stream processor
python kafka_stream_processor.py &

# Step 5: Verify data flow
# Check Kafka
docker exec kafka kafka-console-consumer.sh --topic network.metrics --bootstrap-server localhost:9092 --max-messages 5

# Check InfluxDB
curl http://localhost:8086/api/v2/query -H "Authorization: Token mytoken" -d '{"query": "from(bucket:\"network\") |> range(start:-5m)"}'

# Step 6: View in Grafana
# http://localhost:3000
# Add InfluxDB data source
# Create dashboard with flux queries
```

### Verification Checklist

- [ ] Kafka topics created
- [ ] Producer sends metrics
- [ ] Consumer processes and aggregates
- [ ] Anomaly detection fires alerts
- [ ] InfluxDB stores time-series data
- [ ] Grafana dashboard shows real-time data

---

## Summary

Part นี้ครอบคลุม:
- **Stream processing** architecture
- **Kafka** สำหรับ event streaming
- **Stream processor** พร้อม anomaly detection
- **InfluxDB** สำหรับ time-series storage
- **Real-time dashboard** ด้วย WebSocket

---

[← Part 94: Traffic Engineering](part-094-traffic-engineering.md) | [Part 96: Machine Learning →](part-096-machine-learning.md)
