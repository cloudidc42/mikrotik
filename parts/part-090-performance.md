# Part 90: Performance Tuning

## สารบัญ
1. [Performance Baselines](#baselines)
2. [FastPath Processing](#fastpath)
3. [Hardware Offloading](#offloading)
4. [Queue Optimization](#queues)
5. [Benchmarking](#benchmarking)
6. [Lab: Performance Measurement](#lab)

---

## 1. Performance Baselines {#baselines}

### MikroTik Hardware Performance Reference

| Model | L2 Throughput | L3 Throughput | Firewall | CPU |
|-------|--------------|--------------|---------|-----|
| hAP ac2 | 1 Gbps | 300 Mbps | 150 Mbps | ARM |
| RB4011 | 1 Gbps | 2 Gbps | 500 Mbps | ARM64 |
| CCR1009 | 1 Gbps | 4 Gbps | 1 Gbps | Tile |
| CCR2004 | 10 Gbps | 10 Gbps | 4 Gbps | ARM64 |
| CCR2216 | 100 Gbps | 100 Gbps | 20 Gbps | ARM64 |

```bash
# Check current performance
/system resource print

# Expected output:
# version: 7.x.x (stable)
# cpu-load: 5%
# free-memory: 512000KiB
# total-memory: 1024000KiB
# cpu: 4 core ARM
# cpu-frequency: 1200MHz

# Interface statistics
/interface print stats

# Packet processing stats
/system resource cpu print
```

---

## 2. FastPath Processing {#fastpath}

```bash
# ============================================
# FastPath - hardware-accelerated forwarding
# ============================================

# FastPath ทำงานเมื่อ:
# - Connection tracking enabled
# - Packet ผ่าน connection state = established/related
# - ไม่มี complex rules ที่ต้องตรวจสอบทุก packet

# Enable FastTrack (FastPath via firewall rule)
/ip firewall filter
# ต้องวาง rule นี้ก่อน complex rules
add chain=forward connection-state=established,related \
    action=fasttrack-connection \
    comment="FastTrack established connections"

add chain=forward connection-state=established,related \
    action=accept \
    comment="Accept after FastTrack"

# ตรวจสอบ FastPath stats
/ip firewall connection print stats
# เห็น "fasttrack" column

# Hardware FastPath (Switch chip acceleration)
# ใช้สำหรับ bridge interfaces บน hardware with switch chip
/interface bridge
set bridge1 fast-forward=yes

# ตรวจสอบ FastPath ทำงาน
/interface bridge print detail
# Expected: fast-forward=yes
```

---

## 3. Hardware Offloading {#offloading}

```bash
# ============================================
# Hardware Switch Offloading
# ============================================

# CRS/CSS switches รองรับ hardware L2/L3 offloading

# Bridge hardware offloading
/interface bridge
set bridge1 vlan-filtering=yes

/interface bridge port
set [find] hw=yes  # Enable hardware offloading per port

# ตรวจสอบ HW offloading
/interface bridge port print detail
# Expected: hw=yes ทุก port ที่ support

# Queue Hardware Offloading (บาง models)
# CRS326, CRS354 รองรับ HW queuing

# VLAN offloading
/interface bridge vlan
add bridge=bridge1 tagged=ether1-uplink \
    untagged=ether2-4 vlan-ids=100
# Hardware จะ process VLAN ใน switch chip
```

---

## 4. Queue Optimization {#queues}

```bash
# ============================================
# Queue Optimization สำหรับ Performance
# ============================================

# Simple Queue vs Queue Tree
# Simple Queue = ง่ายกว่า แต่ใช้ CPU มากกว่า
# Queue Tree = ซับซ้อน แต่ performance ดีกว่า

# PCQ (Per Connection Queuing) - แจก bandwidth ให้ทุก client เท่ากัน
/queue type
add name=pcq-up kind=pcq pcq-classifier=src-address pcq-rate=0 pcq-limit=50
add name=pcq-down kind=pcq pcq-classifier=dst-address pcq-rate=0 pcq-limit=50

# Queue Tree + PCQ
/queue tree
add name=uplink-up parent=ether1 queue=pcq-up max-limit=1G
add name=uplink-down parent=ether2 queue=pcq-down max-limit=1G

# CAKE Queue (RouterOS 7.x)
# Excellent for home/small office - reduces bufferbloat
/queue type
add name=cake-isp kind=cake cake-bandwidth=100M cake-diffserv=diffserv4

# Batch Size Optimization
/queue interface
set ether1 queue=only-hardware-queue  # For GbE interfaces

# Mangle optimization - mark packets efficiently
/ip firewall mangle
# ใช้ connection-mark แทน packet-mark เมื่อทำได้
# connection-mark mark เพียงครั้งเดียวต่อ connection
add chain=prerouting connection-state=new \
    src-address=10.100.0.0/24 \
    action=mark-connection new-connection-mark=users-conn \
    comment="Mark user connections (once per connection)"

add chain=prerouting connection-mark=users-conn \
    action=mark-packet new-packet-mark=users-traffic \
    comment="Mark all packets of user connections"
```

---

## 5. Benchmarking {#benchmarking}

```bash
# ============================================
# Performance Benchmarking
# ============================================

# 1. Bandwidth Test (built-in)
/tool bandwidth-test address=192.168.1.100 direction=both duration=30s

# 2. Ping latency
/ping 8.8.8.8 count=100
# Expected: เห็น min/avg/max latency

# 3. Throughput test ด้วย iperf3
# บน server: iperf3 -s
# บน client: iperf3 -c server-ip -t 30 -P 4

# 4. Traffic Generator
/tool traffic-generator stream
add interface=ether2 packet-size=1472 rate=100mbps duration=30s

# 5. CPU under load
/system resource print
# Monitor cpu-load ขณะทดสอบ

# 6. Packet loss test
/ping 192.168.1.100 count=1000 size=1472
# Expected: 0% packet loss


# ============================================
# Performance Monitoring Script
# ============================================

/system script
add name=perf-monitor source={
    :local cpuLoad [/system resource get cpu-load]
    :local freeMem [/system resource get free-memory]
    :local uptime [/system resource get uptime]
    
    :if ($cpuLoad > 90) do={
        :log warning ("HIGH CPU: " . $cpuLoad . "%")
    }
    
    :if ($freeMem < 50000) do={
        :log warning ("LOW MEMORY: " . $freeMem . " KiB")
    }
    
    :log info ("CPU: " . $cpuLoad . "% MEM: " . $freeMem . " KiB UP: " . $uptime)
}

/system scheduler
add name=perf-check interval=5m on-event=perf-monitor
```

```python
# Python performance benchmark
import librouteros
import time
import statistics
from datetime import datetime

class PerformanceBenchmark:
    def __init__(self, host: str, username: str, password: str):
        self.host = host
        self.conn = librouteros.connect(host=host, username=username, password=password)
    
    def measure_api_latency(self, samples: int = 100) -> dict:
        """วัด API response time"""
        latencies = []
        
        for _ in range(samples):
            start = time.perf_counter()
            list(self.conn('/system/identity/print'))
            elapsed = (time.perf_counter() - start) * 1000  # ms
            latencies.append(elapsed)
        
        return {
            "min_ms": round(min(latencies), 2),
            "max_ms": round(max(latencies), 2),
            "avg_ms": round(statistics.mean(latencies), 2),
            "p95_ms": round(sorted(latencies)[int(samples * 0.95)], 2),
            "samples": samples
        }
    
    def get_resource_snapshot(self) -> dict:
        """ดึง resource usage snapshot"""
        resource = dict(list(self.conn('/system/resource/print'))[0])
        
        return {
            "timestamp": datetime.utcnow().isoformat(),
            "cpu_load": resource.get("cpu-load"),
            "free_memory_kb": resource.get("free-memory"),
            "total_memory_kb": resource.get("total-memory"),
            "memory_usage_pct": round(
                (1 - int(resource.get("free-memory", 1)) / 
                 int(resource.get("total-memory", 1))) * 100, 1
            ),
            "uptime": resource.get("uptime"),
            "version": resource.get("version")
        }
    
    def benchmark_interface_throughput(self, interface: str, 
                                         duration: int = 10) -> dict:
        """วัด interface throughput"""
        snapshots = []
        
        for _ in range(duration):
            ifaces = list(self.conn('/interface/print', **{"?name": interface}))
            if ifaces:
                s = dict(ifaces[0])
                snapshots.append({
                    "rx_bps": int(s.get("rx-bits-per-second", 0)),
                    "tx_bps": int(s.get("tx-bits-per-second", 0))
                })
            time.sleep(1)
        
        if not snapshots:
            return {}
        
        return {
            "interface": interface,
            "avg_rx_mbps": round(statistics.mean(s["rx_bps"] for s in snapshots) / 1e6, 2),
            "avg_tx_mbps": round(statistics.mean(s["tx_bps"] for s in snapshots) / 1e6, 2),
            "peak_rx_mbps": round(max(s["rx_bps"] for s in snapshots) / 1e6, 2),
            "peak_tx_mbps": round(max(s["tx_bps"] for s in snapshots) / 1e6, 2)
        }
    
    def close(self):
        self.conn.close()


if __name__ == "__main__":
    bench = PerformanceBenchmark("10.0.0.1", "admin", "password")
    
    print("API Latency:")
    print(bench.measure_api_latency(50))
    
    print("\nResource Snapshot:")
    print(bench.get_resource_snapshot())
    
    print("\nInterface Throughput (ether1, 10s):")
    print(bench.benchmark_interface_throughput("ether1", 10))
    
    bench.close()
```

---

## 6. Lab: Performance Measurement {#lab}

### Lab Steps

```bash
# Step 1: Baseline measurement
/system resource print
/interface print stats

# Step 2: Enable FastTrack
/ip firewall filter add chain=forward connection-state=established,related action=fasttrack-connection place-before=0

# Step 3: Bandwidth test
/tool bandwidth-test address=remote-server direction=both duration=30s

# Step 4: Check CPU during load
# (run bandwidth test ค้างไว้แล้วดู CPU)
/system resource get cpu-load

# Step 5: Compare with/without FastTrack
# ปิด FastTrack แล้ว test อีกครั้ง
# Expected: throughput ต่ำลง, CPU สูงขึ้น
```

### Performance Optimization Checklist

- [ ] FastTrack enabled สำหรับ established connections
- [ ] Hardware offloading enabled บน switch ports
- [ ] PCQ configured สำหรับ fair bandwidth distribution
- [ ] Complex L7 rules เฉพาะ critical interfaces
- [ ] SNMP และ services ที่ไม่ใช้ disabled
- [ ] CPU load < 70% ภายใต้ normal load
- [ ] Latency < 1ms สำหรับ LAN traffic

> **Tip:** ใช้ `/tool profile` เพื่อดูว่า process ใดใช้ CPU มากที่สุด

> **Warning:** FastTrack bypass บาง firewall features - ตรวจสอบ security implications ก่อน

---

## Summary

Part นี้ครอบคลุม:
- **Performance baselines** สำหรับ MikroTik hardware
- **FastPath/FastTrack** acceleration
- **Hardware offloading** บน switch chips
- **Queue optimization** ด้วย PCQ/CAKE
- **Benchmarking** tools และ techniques

---

[← Part 89: Disaster Recovery](part-089-disaster-recovery.md) | [Part 91: ISP from Scratch →](part-091-isp-from-scratch.md)
