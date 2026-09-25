# Part 93: DDoS Protection

## สารบัญ
1. [DDoS Detection](#detection)
2. [BGP Blackholing](#blackhole)
3. [Traffic Scrubbing](#scrubbing)
4. [MikroTik DDoS Rules](#mikrotik-rules)
5. [Automated Response](#automation)
6. [Lab: DDoS Mitigation](#lab)

---

## 1. DDoS Detection {#detection}

### DDoS Attack Types

| Type | Description | Indicator |
|------|-------------|-----------|
| SYN Flood | ส่ง SYN packets จำนวนมาก | TCP state new count สูง |
| UDP Flood | UDP packets ท่วม | UDP pps สูง |
| ICMP Flood | Ping flood | ICMP pps สูง |
| HTTP Flood | HTTP requests จำนวนมาก | HTTP connections สูง |
| Amplification | DNS/NTP/SSDP reflection | Inbound bandwidth spike |

```bash
# ============================================
# DDoS Detection บน MikroTik
# ============================================

# ตรวจสอบ traffic spike
/interface print stats
# สังเกต rx-bits-per-second spike บน WAN interface

# ตรวจสอบ connection table
/ip firewall connection print count-only
# ถ้าสูงผิดปกติ (>100K) = possible DDoS

# ดู top connections
/ip firewall connection print
# sort by packet count

# ตรวจสอบ new connection rate
/ip firewall filter print stats where chain=input
```

```python
# DDoS Detection System
import librouteros
import time
import statistics
from dataclasses import dataclass
from typing import List, Dict

@dataclass
class DDoSAlert:
    timestamp: float
    attack_type: str
    target_ip: str
    pps: float
    bps: float
    severity: str

class DDoSDetector:
    def __init__(self, router_host: str, username: str, password: str):
        self.router_host = router_host
        self.username = username
        self.password = password
        self.history: Dict[str, List[float]] = {}
        self.thresholds = {
            "pps": 100000,       # 100K packets/sec
            "bps": 1_000_000_000,  # 1 Gbps
            "new_connections": 5000,  # 5000 new conns/sec
            "z_score": 4.0        # 4 standard deviations
        }
    
    def get_metrics(self) -> dict:
        conn = librouteros.connect(
            host=self.router_host,
            username=self.username,
            password=self.password,
            timeout=5
        )
        
        interfaces = list(conn('/interface/print'))
        metrics = {}
        
        for iface in interfaces:
            i = dict(iface)
            name = i.get("name", "")
            metrics[name] = {
                "rx_pps": int(i.get("rx-packets-per-second", 0)),
                "tx_pps": int(i.get("tx-packets-per-second", 0)),
                "rx_bps": int(i.get("rx-bits-per-second", 0)),
                "tx_bps": int(i.get("tx-bits-per-second", 0))
            }
        
        conn.close()
        return metrics
    
    def update_history(self, interface: str, metric: str, value: float):
        key = f"{interface}_{metric}"
        if key not in self.history:
            self.history[key] = []
        
        self.history[key].append(value)
        
        # Keep only last 60 samples
        if len(self.history[key]) > 60:
            self.history[key] = self.history[key][-60:]
    
    def detect_anomaly(self, interface: str, metric: str, current: float) -> bool:
        key = f"{interface}_{metric}"
        history = self.history.get(key, [])
        
        if len(history) < 10:
            return False
        
        mean = statistics.mean(history)
        stdev = statistics.stdev(history)
        
        if stdev == 0:
            return False
        
        z_score = (current - mean) / stdev
        return z_score > self.thresholds["z_score"]
    
    def check_for_attacks(self) -> List[DDoSAlert]:
        metrics = self.get_metrics()
        alerts = []
        
        for iface, data in metrics.items():
            rx_pps = data["rx_pps"]
            rx_bps = data["rx_bps"]
            
            # Update history
            self.update_history(iface, "rx_pps", rx_pps)
            self.update_history(iface, "rx_bps", rx_bps)
            
            # Check absolute thresholds
            if rx_pps > self.thresholds["pps"]:
                alerts.append(DDoSAlert(
                    timestamp=time.time(),
                    attack_type="packet_flood",
                    target_ip=iface,
                    pps=rx_pps,
                    bps=rx_bps,
                    severity="critical"
                ))
            
            # Check anomaly
            if self.detect_anomaly(iface, "rx_pps", rx_pps):
                alerts.append(DDoSAlert(
                    timestamp=time.time(),
                    attack_type="traffic_anomaly",
                    target_ip=iface,
                    pps=rx_pps,
                    bps=rx_bps,
                    severity="warning"
                ))
        
        return alerts
```

---

## 2. BGP Blackholing {#blackhole}

```bash
# ============================================
# BGP Blackhole Routing
# ============================================

# เมื่อตรวจจับ DDoS target IP
# ส่ง BGP community 65535:666 ไปยัง upstream
# เพื่อ drop traffic ที่ upstream router

# กำหนด blackhole community
/routing filter rule
add chain=bgp-out-upstream rule="
    if (bgp-communities has 65535:666) {
        set bgp-communities 65535:666;
        accept;
    }
"

# Script สำหรับ blackhole target IP
/system script
add name=bgp-blackhole source={
    :local targetIP $1
    
    # Add blackhole route
    /ip route add dst-address=($targetIP . "/32") \
        blackhole comment=("DDoS-Blackhole-" . $targetIP)
    
    # Advertise via BGP with community
    /routing bgp network add network=($targetIP . "/32") \
        comment=("Blackhole-" . $targetIP)
    
    :log warning ("BGP Blackhole activated for: " . $targetIP)
}

# รัน script
/system script run bgp-blackhole 1.2.3.4

# ยกเลิก blackhole หลัง attack
/system script
add name=bgp-unblackhole source={
    :local targetIP $1
    /ip route remove [find dst-address=($targetIP . "/32") blackhole=yes]
    /routing bgp network remove [find network=($targetIP . "/32")]
    :log info ("BGP Blackhole removed for: " . $targetIP)
}
```

---

## 3. Traffic Scrubbing {#scrubbing}

```bash
# ============================================
# In-line Traffic Scrubbing บน MikroTik
# ============================================

# Rate limiting สำหรับ DDoS mitigation
/ip firewall filter

# 1. SYN Flood Protection
add chain=input protocol=tcp tcp-flags=syn \
    limit=2000,5000:packet \
    action=accept comment="SYN flood limit"
add chain=input protocol=tcp tcp-flags=syn \
    action=drop log=yes log-prefix="SYN-FLOOD:"

# 2. UDP Flood Protection
add chain=input protocol=udp \
    limit=10000,20000:packet \
    action=accept comment="UDP flood limit"
add chain=input protocol=udp \
    action=drop log=yes log-prefix="UDP-FLOOD:"

# 3. ICMP Flood Protection
add chain=input protocol=icmp \
    limit=100,200:packet \
    action=accept comment="ICMP limit"
add chain=input protocol=icmp \
    action=drop log=yes log-prefix="ICMP-FLOOD:"

# 4. Connection limit per IP
add chain=input connection-limit=100,32 \
    action=add-src-to-address-list \
    address-list=ddos-attack \
    address-list-timeout=5m \
    comment="Detect high connection source"
add chain=input src-address-list=ddos-attack \
    action=drop comment="Drop DDoS source"

# 5. TCP flags validation
add chain=input protocol=tcp \
    tcp-flags=fin,!syn,!rst,!ack \
    action=drop comment="NULL scan"
add chain=input protocol=tcp \
    tcp-flags=fin,syn action=drop \
    comment="Bogus TCP flags"
add chain=input protocol=tcp \
    tcp-flags=syn,rst action=drop \
    comment="Bogus TCP flags"
```

---

## 4. Automated Response {#automation}

```python
# ============================================
# Automated DDoS Response System
# ============================================

import asyncio
import librouteros
from datetime import datetime, timedelta
from typing import Set
import aioredis

class AutoDDoSMitigation:
    """Automated DDoS mitigation system"""
    
    def __init__(self, router_configs: list, redis_url: str = "redis://localhost"):
        self.router_configs = router_configs
        self.redis_url = redis_url
        self.mitigated_ips: Set[str] = set()
        self.detector = DDoSDetector(**router_configs[0])
    
    async def blackhole_ip(self, target_ip: str, duration_minutes: int = 30):
        """Blackhole target IP บน MikroTik"""
        for router_config in self.router_configs:
            try:
                conn = librouteros.connect(**{
                    k: v for k, v in router_config.items() 
                    if k in ['host', 'username', 'password']
                })
                
                # Add blackhole route
                conn('/ip/route/add', **{
                    "dst-address": f"{target_ip}/32",
                    "blackhole": "yes",
                    "comment": f"DDoS-BH-{datetime.utcnow().isoformat()}"
                })
                
                conn.close()
                self.mitigated_ips.add(target_ip)
                
                # Schedule removal
                asyncio.create_task(
                    self._remove_blackhole_after(target_ip, duration_minutes)
                )
                
            except Exception as e:
                print(f"Error blackholing {target_ip} on {router_config['host']}: {e}")
    
    async def _remove_blackhole_after(self, target_ip: str, minutes: int):
        """ลบ blackhole route หลัง X minutes"""
        await asyncio.sleep(minutes * 60)
        await self.remove_blackhole(target_ip)
    
    async def remove_blackhole(self, target_ip: str):
        """ลบ blackhole route"""
        for router_config in self.router_configs:
            try:
                conn = librouteros.connect(**{
                    k: v for k, v in router_config.items() 
                    if k in ['host', 'username', 'password']
                })
                
                # Find and remove blackhole route
                routes = list(conn('/ip/route/print', **{
                    "?dst-address": f"{target_ip}/32",
                    "?blackhole": "yes"
                }))
                
                for route in routes:
                    r = dict(route)
                    conn('/ip/route/remove', **{"numbers": r.get(".id")})
                
                conn.close()
                self.mitigated_ips.discard(target_ip)
                print(f"Blackhole removed for {target_ip}")
                
            except Exception as e:
                print(f"Error removing blackhole for {target_ip}: {e}")
    
    async def monitor_and_respond(self):
        """Monitor แบบ continuous และ respond automatically"""
        print("Starting automated DDoS protection...")
        
        while True:
            try:
                alerts = self.detector.check_for_attacks()
                
                for alert in alerts:
                    if alert.severity == "critical" and alert.target_ip not in self.mitigated_ips:
                        print(f"DDoS detected! Type: {alert.attack_type} PPS: {alert.pps:.0f}")
                        # Extract target IP from alert
                        # In real implementation, identify target from traffic analysis
                        await self.blackhole_ip("victim-ip", duration_minutes=30)
                        
                        # Send notification
                        await self._send_alert(alert)
            
            except Exception as e:
                print(f"Monitor error: {e}")
            
            await asyncio.sleep(5)
    
    async def _send_alert(self, alert: DDoSAlert):
        """ส่ง alert notification"""
        # Implement notification (Telegram, email, etc.)
        print(f"ALERT: DDoS {alert.attack_type} at {datetime.utcnow().isoformat()}")
```

---

## 5. Lab: DDoS Mitigation {#lab}

### Lab Topology

```
[Attack Traffic]
    │ 10Gbps flood
    ▼
[Border Router]
    │ BGP Blackhole
    ▼
[CGNAT/Core] ── Rate Limiting
    │
[Protected Service]
```

### Lab Steps

```bash
# Step 1: Setup detection rules
/ip firewall filter add chain=input protocol=tcp tcp-flags=syn limit=2000,5000:packet action=accept
/ip firewall filter add chain=input protocol=tcp tcp-flags=syn action=drop log=yes log-prefix="SYN-FLOOD:"

# Step 2: Create blackhole script
/system script add name=blackhole source={
    /ip route add dst-address=($1 . "/32") blackhole comment="DDoS"
    :log warning ("Blackhole: " . $1)
}

# Step 3: Test with simulated flood
# (ใช้ traffic generator หรือ hping3 จาก test machine)
# hping3 --syn --flood -p 80 target-ip

# Step 4: Verify mitigation
/ip firewall filter print stats
/log print topics=firewall where message~"SYN-FLOOD"

# Step 5: Activate blackhole
/system script run blackhole "1.2.3.4"
/ip route print where blackhole=yes
```

### Verification Checklist

- [ ] SYN flood rules drop malicious traffic
- [ ] UDP/ICMP rate limits applied
- [ ] BGP blackhole script works
- [ ] Automated detection running
- [ ] Alerts sent to NOC
- [ ] Scrubbing reduces attack traffic

> **Tip:** ใช้ upstream ISP's DDoS scrubbing service เมื่อ attack เกินความสามารถ router

> **Warning:** Aggressive rate limiting อาจ drop legitimate traffic - calibrate thresholds carefully

---

## Summary

Part นี้ครอบคลุม:
- **DDoS detection** ด้วย traffic analysis
- **BGP blackholing** สำหรับ upstream mitigation
- **Traffic scrubbing** rules
- **Automated response** system
- **Rate limiting** สำหรับ flood protection

---

[← Part 92: CGNAT](part-092-cgnat.md) | [Part 94: Traffic Engineering →](part-094-traffic-engineering.md)
