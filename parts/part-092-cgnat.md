# Part 92: CGNAT - Carrier Grade NAT

## สารบัญ
1. [CGNAT Concepts](#concepts)
2. [MikroTik CGNAT](#mikrotik-cgnat)
3. [Port Allocation](#port-allocation)
4. [Logging](#logging)
5. [Performance Considerations](#performance)
6. [Lab: CGNAT Implementation](#lab)

---

## 1. CGNAT Concepts {#concepts}

CGNAT (Carrier Grade NAT) ใช้เมื่อ ISP ไม่มี IPv4 addresses เพียงพอสำหรับ customers ทุกคน

### CGNAT Address Space (RFC 6598)

```
100.64.0.0/10 = Shared Address Space
├── 100.64.0.0 - 100.127.255.255
└── 4,194,304 addresses

ใช้สำหรับ: ISP → CPE link
ไม่ route บน internet public
```

### CGNAT Architecture

```
[Customer CPE]         [ISP CGNAT Router]    [Internet]
192.168.1.x  ─NAT1─>  100.64.x.x   ─NAT2─>  1.2.3.4
              (CPE NAT)  (CGNAT)
```

### Port Allocation Approaches

| Method | Description | Ports per subscriber |
|--------|-------------|----------------------|
| Static PAT | กำหนด port range คงที่ | 512-2048 |
| Dynamic PAT | แจก port ตาม demand | Variable |
| PCP (Port Control Protocol) | Client-requested | On demand |

---

## 2. MikroTik CGNAT {#mikrotik-cgnat}

```bash
# ============================================
# CGNAT Configuration บน MikroTik
# ============================================

# ============================================
# Basic CGNAT Setup
# ============================================

# WAN interface กับ public IP
/ip address
add address=203.0.113.1/30 interface=ether1-wan comment="Public IP"

# Customer subnet (RFC 6598 shared space)
/ip address
add address=100.64.0.1/10 interface=ether2-customers comment="CGNAT pool"

# Basic CGNAT (masquerade)
/ip firewall nat
add chain=srcnat out-interface=ether1-wan \
    src-address=100.64.0.0/10 \
    action=masquerade \
    comment="CGNAT masquerade"

# ============================================
# Advanced CGNAT with Port Allocation
# ============================================

# กำหนด port range ต่อ subscriber (512 ports each)
# Subscriber 1: 100.64.0.1 → Public: 203.0.113.1:1024-1535
# Subscriber 2: 100.64.0.2 → Public: 203.0.113.1:1536-2047
# ... และต่อไป

/ip firewall nat
# Subscriber 1 port range
add chain=srcnat src-address=100.64.0.1 \
    out-interface=ether1-wan \
    action=src-nat \
    to-addresses=203.0.113.1 \
    to-ports=1024-1535 \
    comment="CGNAT Sub1"

# Subscriber 2 port range
add chain=srcnat src-address=100.64.0.2 \
    out-interface=ether1-wan \
    action=src-nat \
    to-addresses=203.0.113.1 \
    to-ports=1536-2047 \
    comment="CGNAT Sub2"

# หลาย public IPs (ขยาย capacity)
/ip address
add address=203.0.113.5/32 interface=ether1-wan comment="CGNAT IP-2"
add address=203.0.113.6/32 interface=ether1-wan comment="CGNAT IP-3"

# กลุ่ม customers ต่อ public IP
/ip firewall nat
add chain=srcnat src-address=100.64.0.0/24 \
    out-interface=ether1-wan action=src-nat \
    to-addresses=203.0.113.1

add chain=srcnat src-address=100.64.1.0/24 \
    out-interface=ether1-wan action=src-nat \
    to-addresses=203.0.113.5
```

---

## 3. Port Allocation {#port-allocation}

```python
# ============================================
# CGNAT Port Allocation Manager
# ============================================

from dataclasses import dataclass
from typing import Optional, Dict, Tuple
import ipaddress

@dataclass
class PortAllocation:
    subscriber_ip: str
    public_ip: str
    port_start: int
    port_end: int

class CGNATManager:
    """จัดการ CGNAT port allocation"""
    
    def __init__(self, public_ips: list, ports_per_subscriber: int = 512):
        self.public_ips = public_ips
        self.ports_per_subscriber = ports_per_subscriber
        self.allocations: Dict[str, PortAllocation] = {}
        
        # Calculate capacity
        # Ports 1024-65535 = 64512 usable ports
        # Per public IP: 64512 / 512 = 126 subscribers
        self.usable_ports_start = 1024
        self.usable_ports_end = 65535
        self.max_per_ip = (self.usable_ports_end - self.usable_ports_start) // ports_per_subscriber
        self.max_subscribers = len(public_ips) * self.max_per_ip
    
    def allocate(self, subscriber_ip: str) -> Optional[PortAllocation]:
        """จัดสรร port range ให้ subscriber"""
        if subscriber_ip in self.allocations:
            return self.allocations[subscriber_ip]
        
        total_allocated = len(self.allocations)
        
        if total_allocated >= self.max_subscribers:
            return None  # No capacity
        
        # Calculate which public IP and which port range
        public_ip_index = total_allocated // self.max_per_ip
        slot_in_ip = total_allocated % self.max_per_ip
        
        public_ip = self.public_ips[public_ip_index]
        port_start = self.usable_ports_start + (slot_in_ip * self.ports_per_subscriber)
        port_end = port_start + self.ports_per_subscriber - 1
        
        allocation = PortAllocation(
            subscriber_ip=subscriber_ip,
            public_ip=public_ip,
            port_start=port_start,
            port_end=port_end
        )
        
        self.allocations[subscriber_ip] = allocation
        return allocation
    
    def generate_mikrotik_rules(self) -> str:
        """สร้าง MikroTik NAT rules"""
        rules = ["/ip firewall nat"]
        
        for sub_ip, alloc in self.allocations.items():
            rules.append(
                f"add chain=srcnat src-address={sub_ip} "
                f"out-interface=ether1-wan action=src-nat "
                f"to-addresses={alloc.public_ip} "
                f"to-ports={alloc.port_start}-{alloc.port_end} "
                f"comment=\"CGNAT {sub_ip}\""
            )
        
        return "\n".join(rules)
    
    def get_subscriber_from_public(self, public_ip: str, port: int) -> Optional[str]:
        """หา subscriber จาก public IP:port"""
        for sub_ip, alloc in self.allocations.items():
            if alloc.public_ip == public_ip and alloc.port_start <= port <= alloc.port_end:
                return sub_ip
        return None


# Example usage
manager = CGNATManager(
    public_ips=["203.0.113.1", "203.0.113.2"],
    ports_per_subscriber=512
)

# Allocate for subscribers
for i in range(1, 11):
    alloc = manager.allocate(f"100.64.0.{i}")
    if alloc:
        print(f"{alloc.subscriber_ip} → {alloc.public_ip}:{alloc.port_start}-{alloc.port_end}")

# Generate MikroTik rules
print("\nMikroTik Rules:")
print(manager.generate_mikrotik_rules())
```

---

## 4. Logging {#logging}

```bash
# ============================================
# CGNAT Logging สำหรับ Legal Compliance
# ============================================

# CGNAT logging เป็นข้อกำหนดทางกฎหมายในหลายประเทศ
# ต้องบันทึก: timestamp, subscriber IP, public IP:port, protocol

/ip firewall nat
# Enable log บน CGNAT rules
add chain=srcnat src-address=100.64.0.0/10 \
    out-interface=ether1-wan \
    action=masquerade \
    log=yes log-prefix="CGNAT:"

/system logging
add topics=firewall action=remote
```

```python
# CGNAT Log Processor
import re
from datetime import datetime
from elasticsearch import Elasticsearch

class CGNATLogProcessor:
    """Process CGNAT syslog สำหรับ compliance"""
    
    def __init__(self, es_url: str = "http://localhost:9200"):
        self.es = Elasticsearch(es_url)
        self.pattern = re.compile(
            r'CGNAT:.*src-mac.*proto (\w+), '
            r'(\d+\.\d+\.\d+\.\d+):(\d+)->'
            r'(\d+\.\d+\.\d+\.\d+):(\d+)'
        )
    
    def parse_log(self, log_line: str) -> Optional[dict]:
        match = self.pattern.search(log_line)
        if not match:
            return None
        
        return {
            "timestamp": datetime.utcnow().isoformat(),
            "proto": match.group(1),
            "subscriber_ip": match.group(2),
            "subscriber_port": int(match.group(3)),
            "public_ip": match.group(4),
            "public_port": int(match.group(5)),
            "log_line": log_line[:500]
        }
    
    def store_log(self, log_entry: dict):
        """บันทึก log ลง Elasticsearch"""
        self.es.index(
            index=f"cgnat-{datetime.utcnow().strftime('%Y.%m')}",
            body=log_entry
        )
    
    def lookup_subscriber(self, public_ip: str, public_port: int, 
                          timestamp: str) -> Optional[dict]:
        """หา subscriber จาก public IP:port+time"""
        query = {
            "query": {
                "bool": {
                    "must": [
                        {"term": {"public_ip": public_ip}},
                        {"term": {"public_port": public_port}},
                        {"range": {"timestamp": {
                            "gte": timestamp,
                            "lte": timestamp
                        }}}
                    ]
                }
            }
        }
        
        result = self.es.search(index="cgnat-*", body=query)
        hits = result["hits"]["hits"]
        
        if hits:
            return hits[0]["_source"]
        return None
```

---

## 5. Performance Considerations {#performance}

```bash
# ============================================
# CGNAT Performance Optimization
# ============================================

# MikroTik NAT performance tips

# 1. ใช้ connection tracking แบบ loose
/ip firewall connection tracking
set enabled=yes loose-tcp-tracking=yes

# 2. Increase connection limits
/ip firewall connection tracking
set max-fasttrack-entries=1000000

# 3. เพิ่ม NAT max entries
/ip firewall nat
# ใช้ action=src-nat แทน masquerade
# src-nat ใช้ memory น้อยกว่า

# 4. FastTrack สำหรับ NAT
/ip firewall filter
add chain=forward connection-state=established,related \
    action=fasttrack-connection

# 5. Hardware NAT (CCR models)
# CCR2216 รองรับ hardware NAT offloading

# Performance metrics สำหรับ planning
# CCR2004: ~500K NAT sessions, ~2Gbps with NAT
# CCR2216: ~5M NAT sessions, ~20Gbps with NAT

# Monitoring NAT table
/ip firewall connection print count-only
/ip firewall connection tracking print stats
```

---

## 6. Lab: CGNAT Implementation {#lab}

### Lab Topology

```
[Customer 1: 100.64.0.1]  ─┐
[Customer 2: 100.64.0.2]  ─┤
[Customer 3: 100.64.0.3]  ─┘
                            │
                   [MikroTik CGNAT]
                    100.64.0.0/10
                       │ CGNAT
                   203.0.113.1
                       │
                   [Internet]
```

### Lab Steps

```bash
# Step 1: Setup interfaces
/ip address add address=203.0.113.1/30 interface=ether1-wan
/ip address add address=100.64.0.254/10 interface=ether2-customers

# Step 2: CGNAT rules
/ip firewall nat add chain=srcnat src-address=100.64.0.0/10 out-interface=ether1-wan action=masquerade log=yes log-prefix="CGNAT:"

# Step 3: Routing
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.254

# Step 4: Test
# จาก customer host:
# ping 8.8.8.8  # ควรผ่าน CGNAT

# Step 5: Verify translation
/ip firewall nat print stats
/ip firewall connection print where src-address~"100.64"
```

### Verification Checklist

- [ ] CGNAT NAT rules configured
- [ ] Customers ใน 100.64.0.0/10 รับ internet
- [ ] Public IP แสดงเป็น 203.0.113.x
- [ ] Logging enabled สำหรับ compliance
- [ ] Port allocation per subscriber documented
- [ ] Performance metrics ยอมรับได้

> **Note:** CGNAT ทำให้ port-forwarding (ส่ง traffic จาก internet มา customer) ทำได้ยาก - ต้องใช้ PCP หรือ static mapping

> **Warning:** บาง applications ไม่รองรับ CGNAT เช่น P2P, gaming, hosting servers

---

## Summary

Part นี้ครอบคลุม:
- **CGNAT concepts** และ RFC 6598
- **MikroTik CGNAT** configuration
- **Port allocation** ต่อ subscriber
- **Logging** สำหรับ legal compliance
- **Performance** optimization

---

[← Part 91: ISP from Scratch](part-091-isp-from-scratch.md) | [Part 93: DDoS Protection →](part-093-ddos-protection.md)
