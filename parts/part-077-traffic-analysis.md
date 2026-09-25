# Part 77: Traffic Analysis

## สารบัญ
1. [Torch Tool](#torch)
2. [NetFlow/IPFIX](#netflow)
3. [sFlow](#sflow)
4. [DPI - Deep Packet Inspection](#dpi)
5. [Anomaly Detection](#anomaly)
6. [Traffic Visualization](#visualization)
7. [Lab: Traffic Analysis Setup](#lab)

---

## 1. Torch Tool {#torch}

Torch เป็น built-in tool ของ MikroTik สำหรับ real-time traffic monitoring

```bash
# ============================================
# Torch Usage
# ============================================

# Basic torch - monitor interface
/tool torch interface=ether1

# Filter by protocol
/tool torch interface=ether1 ip-protocol=tcp

# Filter by IP
/tool torch interface=ether1 src-address=192.168.1.0/24

# Filter by port
/tool torch interface=ether1 port=80,443

# Combine filters
/tool torch interface=ether1 \
    src-address=192.168.1.100 \
    ip-protocol=tcp \
    port=443

# ดู traffic ทิศทาง upload/download
/tool torch interface=ether1 direction=both


# ============================================
# Traffic Generator สำหรับ Test
# ============================================

/tool traffic-generator quick
add interface=ether2 packet-size=1024 count=1000

# UDP traffic test
/tool traffic-generator stream
add interface=ether2 packet-size=1472 rate=100mbps


# Bandwidth test
/tool bandwidth-test address=203.0.113.100 direction=both duration=10s
```

---

## 2. NetFlow/IPFIX {#netflow}

```bash
# ============================================
# NetFlow v9 / IPFIX Configuration
# ============================================

# Traffic Flow (NetFlow exporter)
/ip traffic-flow
set enabled=yes active-flow-timeout=1m \
    inactive-flow-timeout=15s \
    interfaces=all

/ip traffic-flow target
add dst-address=192.168.10.10 port=2055 version=9 \
    comment="NetFlow collector"

# IPFIX (รองรับ RouterOS 7.x)
/ip traffic-flow target
add dst-address=192.168.10.10 port=4739 version=10 \
    comment="IPFIX collector"

# ตรวจสอบ flow stats
/ip traffic-flow print
/ip traffic-flow stat print


# ============================================
# ntopng + nProbe สำหรับ collection
# ============================================
# docker-compose.yml
```

```yaml
version: '3.8'

services:
  nprobe:
    image: ntop/nprobe:stable
    network_mode: host
    command: >
      nprobe --interface=any
      --zmq=tcp://*:5556
      --netflow-version=10
      -3 192.168.10.1:2055
      --json-mode
    restart: unless-stopped

  ntopng:
    image: ntop/ntopng:stable
    ports:
      - "3000:3000"
    command: >
      ntopng --interface=tcp://*:5556
      --redis=redis:6379
      --community
    depends_on:
      - redis
      - nprobe
    restart: unless-stopped

  redis:
    image: redis:alpine
    restart: unless-stopped
```

```python
# Python NetFlow listener
import socket
import struct
from datetime import datetime

def parse_netflow_v9(data: bytes) -> dict:
    """Parse NetFlow v9 packet"""
    if len(data) < 20:
        return {}
    
    version, count, uptime, epoch, seq, source_id = struct.unpack('!HHIII', data[:20])
    
    return {
        "version": version,
        "count": count,
        "uptime_ms": uptime,
        "timestamp": datetime.fromtimestamp(epoch).isoformat(),
        "sequence": seq,
        "source_id": source_id
    }


def listen_netflow(port: int = 2055):
    """Listen for NetFlow packets"""
    sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
    sock.bind(('0.0.0.0', port))
    
    print(f"Listening for NetFlow on port {port}")
    
    while True:
        data, addr = sock.recvfrom(65535)
        flow = parse_netflow_v9(data)
        print(f"Flow from {addr[0]}: version={flow.get('version')} count={flow.get('count')}")


if __name__ == "__main__":
    listen_netflow()
```

---

## 3. sFlow {#sflow}

```bash
# ============================================
# sFlow Configuration (บาง MikroTik models)
# ============================================

# sFlow ส่ง sample ของ packets แทน full flows
# ใช้ SNMP-based sFlow ผ่าน CRS switches

# CRS switch sFlow configuration (ผ่าน SNMP)
/snmp set enabled=yes community=public contact=admin location=datacenter

# หรือใช้ sflowtool ที่ router
# (ไม่ได้ built-in, ต้องใช้ sFlow agent บน host)
```

```python
# sFlow collector (Python)
import socket
import struct

class SFlowCollector:
    def __init__(self, port: int = 6343):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
        self.sock.bind(('0.0.0.0', port))
    
    def parse_sample(self, data: bytes, offset: int) -> dict:
        """Parse sFlow sample"""
        sample_type, = struct.unpack_from('!I', data, offset)
        offset += 4
        
        return {
            "sample_type": sample_type,
            "offset": offset
        }
    
    def listen(self):
        print("sFlow collector listening...")
        while True:
            data, addr = self.sock.recvfrom(65535)
            
            # Parse sFlow header
            if len(data) >= 28:
                version, ip_version, agent_ip, sub_agent_id, seq, uptime, num_samples = \
                    struct.unpack_from('!IIxxxx4sIII', data, 0)
                
                print(f"sFlow v{version} from agent: seq={seq} samples={num_samples}")

if __name__ == "__main__":
    collector = SFlowCollector()
    collector.listen()
```

---

## 4. DPI - Deep Packet Inspection {#dpi}

```bash
# ============================================
# Layer 7 Protocol Detection
# ============================================

# L7 Protocol matching
/ip firewall layer7-protocol
add name=youtube regexp="youtube\\.com|googlevideo\\.com"
add name=netflix regexp="netflix\\.com|nflxvideo\\.net"
add name=facebook regexp="facebook\\.com|fbcdn\\.net"
add name=bittorrent regexp="^(\\x13bittorrent|azureus|bittorrent)"
add name=skype regexp="skype\\.com"

# ใช้ L7 ใน firewall/mangle
/ip firewall mangle
add chain=prerouting layer7-protocol=youtube \
    action=mark-packet new-packet-mark=youtube-traffic \
    comment="Mark YouTube traffic"

add chain=prerouting layer7-protocol=bittorrent \
    action=mark-packet new-packet-mark=p2p-traffic \
    comment="Mark P2P traffic"

# ควบคุม bandwidth ตาม L7
/queue tree
add name=youtube-limit parent=global \
    packet-mark=youtube-traffic \
    max-limit=20M priority=5

add name=p2p-limit parent=global \
    packet-mark=p2p-traffic \
    max-limit=5M priority=8


# ============================================
# Application Identification ด้วย Port
# ============================================

/ip firewall address-list
add list=streaming-services address=208.65.152.0/22 comment="Netflix CDN"
add list=streaming-services address=103.236.162.0/24 comment="YouTube CDN"

/ip firewall mangle
add chain=prerouting dst-address-list=streaming-services \
    action=mark-packet new-packet-mark=streaming \
    comment="Mark streaming services"
```

---

## 5. Anomaly Detection {#anomaly}

```python
# ============================================
# Network Anomaly Detection
# ============================================

import asyncio
import aioredis
from collections import defaultdict
from datetime import datetime, timedelta
import statistics
import json

class AnomalyDetector:
    """ตรวจจับ network anomalies"""
    
    def __init__(self, redis_url: str = "redis://localhost"):
        self.redis_url = redis_url
        self.baselines = {}
        
    async def record_metric(self, metric: str, value: float, timestamp: datetime = None):
        """บันทึก metric ลง Redis"""
        redis = await aioredis.create_redis_pool(self.redis_url)
        ts = int((timestamp or datetime.utcnow()).timestamp())
        
        key = f"metric:{metric}"
        await redis.zadd(key, ts, f"{ts}:{value}")
        
        # Keep only last 1 hour
        cutoff = ts - 3600
        await redis.zremrangebyscore(key, '-inf', cutoff)
        redis.close()
    
    async def get_baseline(self, metric: str, window_minutes: int = 60) -> dict:
        """คำนวณ baseline จาก historical data"""
        redis = await aioredis.create_redis_pool(self.redis_url)
        
        now = int(datetime.utcnow().timestamp())
        cutoff = now - (window_minutes * 60)
        
        values_raw = await redis.zrangebyscore(f"metric:{metric}", cutoff, now)
        redis.close()
        
        if not values_raw:
            return {}
        
        values = [float(v.decode().split(':')[1]) for v in values_raw]
        
        return {
            "mean": statistics.mean(values),
            "stdev": statistics.stdev(values) if len(values) > 1 else 0,
            "min": min(values),
            "max": max(values),
            "count": len(values)
        }
    
    async def detect_anomaly(self, metric: str, current_value: float, 
                              threshold_stdev: float = 3.0) -> dict:
        """ตรวจจับ anomaly ด้วย Z-score"""
        baseline = await self.get_baseline(metric)
        
        if not baseline or baseline.get("stdev", 0) == 0:
            return {"anomaly": False, "reason": "insufficient baseline data"}
        
        z_score = abs(current_value - baseline["mean"]) / baseline["stdev"]
        
        if z_score > threshold_stdev:
            return {
                "anomaly": True,
                "metric": metric,
                "current_value": current_value,
                "baseline_mean": baseline["mean"],
                "z_score": z_score,
                "severity": "critical" if z_score > 5 else "warning"
            }
        
        return {"anomaly": False, "z_score": z_score}


class TrafficAnomalyMonitor:
    """Monitor traffic สำหรับ anomalies"""
    
    def __init__(self, router_host: str, detector: AnomalyDetector):
        self.router_host = router_host
        self.detector = detector
    
    async def check_traffic_anomalies(self, interfaces: list):
        import librouteros
        
        conn = librouteros.connect(
            host=self.router_host,
            username="admin", 
            password="password"
        )
        
        for iface in interfaces:
            stats = list(conn(f'/interface/print', **{"?name": iface}))
            if stats:
                s = dict(stats[0])
                tx_bps = int(s.get("tx-bits-per-second", 0))
                rx_bps = int(s.get("rx-bits-per-second", 0))
                
                # Record metrics
                await self.detector.record_metric(f"{iface}_tx_bps", tx_bps)
                await self.detector.record_metric(f"{iface}_rx_bps", rx_bps)
                
                # Check for anomalies
                tx_result = await self.detector.detect_anomaly(f"{iface}_tx_bps", tx_bps)
                rx_result = await self.detector.detect_anomaly(f"{iface}_rx_bps", rx_bps)
                
                for result in [tx_result, rx_result]:
                    if result.get("anomaly"):
                        print(f"ANOMALY DETECTED on {self.router_host}/{iface}: {result}")
        
        conn.close()
```

---

## 6. Traffic Visualization {#visualization}

```python
# Grafana dashboard data generator
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List
import asyncio

app = FastAPI()

class TrafficMetric(BaseModel):
    timestamp: str
    interface: str
    rx_bps: int
    tx_bps: int
    rx_packets: int
    tx_packets: int

@app.get("/metrics/{router_host}")
async def get_traffic_metrics(router_host: str, interface: str = "ether1"):
    """ดึง traffic metrics สำหรับ Grafana"""
    import librouteros
    from datetime import datetime
    
    try:
        conn = librouteros.connect(host=router_host, username="admin", password="pass")
        stats = list(conn('/interface/print', **{"?name": interface}))
        
        if not stats:
            return {"error": "Interface not found"}
        
        s = dict(stats[0])
        conn.close()
        
        return {
            "timestamp": datetime.utcnow().isoformat(),
            "interface": interface,
            "rx_bps": int(s.get("rx-bits-per-second", 0)),
            "tx_bps": int(s.get("tx-bits-per-second", 0)),
            "rx_packets": int(s.get("rx-packets-per-second", 0)),
            "tx_packets": int(s.get("tx-packets-per-second", 0))
        }
    except Exception as e:
        return {"error": str(e)}
```

---

## 7. Lab: Traffic Analysis Setup {#lab}

### Lab Architecture

```
[MikroTik Router]
      │
      │ NetFlow v9 (UDP 2055)
      │
[nProbe Collector] ──ZMQ──> [ntopng Dashboard]
                                    │
                              [Web Browser]
                              :3000
```

### Lab Steps

```bash
# Step 1: Enable NetFlow on MikroTik
/ip traffic-flow set enabled=yes interfaces=all
/ip traffic-flow target add dst-address=192.168.10.10 port=2055 version=9

# Step 2: Start ntopng + nprobe
docker-compose up -d

# Step 3: Verify flow export
/ip traffic-flow stat print

# Step 4: Enable L7 Detection
/ip firewall layer7-protocol add name=youtube regexp="youtube\\.com"
/ip firewall mangle add chain=prerouting layer7-protocol=youtube action=mark-packet new-packet-mark=youtube

# Step 5: Test with traffic
curl -I https://www.youtube.com

# Step 6: View in ntopng
# http://192.168.10.10:3000
# default: admin/admin
```

### Verification Checklist

- [ ] NetFlow packets ถูกส่งไป collector
- [ ] ntopng รับ flows และ display ได้
- [ ] L7 protocols ถูก identify ถูกต้อง
- [ ] Anomaly detection สร้าง baseline แล้ว
- [ ] Traffic visualization ทำงาน

> **Tip:** ใช้ `tcpdump port 2055` บน collector เพื่อ verify NetFlow reception

> **Note:** L7 inspection ใช้ CPU สูง แนะนำใช้เฉพาะ critical interfaces

---

## Summary

Part นี้ครอบคลุม:
- **Torch tool** สำหรับ real-time monitoring
- **NetFlow/IPFIX** สำหรับ flow collection
- **sFlow** สำหรับ packet sampling
- **DPI/L7** protocol detection
- **Anomaly detection** ด้วย Z-score
- **Traffic visualization** ด้วย ntopng

---

[← Part 76: IPv6](part-076-ipv6.md) | [Part 78: Security Hardening →](part-078-security-hardening.md)
