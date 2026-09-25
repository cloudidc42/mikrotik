# Part 94: Traffic Engineering

## สารบัญ
1. [TE Concepts](#concepts)
2. [BGP Traffic Engineering](#bgp-te)
3. [Policy-Based Routing](#pbr)
4. [Load Balancing Techniques](#load-balancing)
5. [Traffic Matrices](#matrices)
6. [Lab: Multi-Path TE](#lab)

---

## 1. TE Concepts {#concepts}

Traffic Engineering คือการควบคุมเส้นทาง traffic ให้ใช้ทรัพยากรเครือข่ายอย่างมีประสิทธิภาพ

### TE Goals

| Goal | Description |
|------|-------------|
| Load Balancing | กระจาย traffic หลาย links |
| Path Optimization | ใช้ low-latency paths สำหรับ real-time |
| Bandwidth Guarantee | สำรอง bandwidth สำหรับ critical apps |
| Fault Tolerance | Reroute เมื่อ link fail |
| Cost Optimization | ใช้ cheaper links สำหรับ bulk traffic |

---

## 2. BGP Traffic Engineering {#bgp-te}

```bash
# ============================================
# BGP Traffic Engineering สำหรับ Multi-homed ISP
# ============================================

# OUTBOUND TE: เลือก exit path สำหรับ traffic

# Method 1: LOCAL_PREF (higher = preferred)
/routing filter rule
# Prefer ISP1 สำหรับ general traffic
add chain=bgp-in-isp1 rule="set bgp-local-pref 200; accept;"
add chain=bgp-in-isp2 rule="set bgp-local-pref 100; accept;"

# Prefer ISP2 สำหรับ specific destinations
add chain=bgp-in-isp2 rule="
    if (dst in 10.20.0.0/14) { set bgp-local-pref 250; }
    accept;
"

# Method 2: MED (lower = preferred for inbound)
# ส่ง MED ต่ำกว่าให้ ISP1 เพื่อให้ internet traffic เข้าผ่าน ISP1
add chain=bgp-out-isp1 rule="set bgp-med 10; accept;"
add chain=bgp-out-isp2 rule="set bgp-med 100; accept;"

# Method 3: AS-PATH Prepend (longer = less preferred for inbound)
# ทำให้ ISP2 prefix ยาวกว่า = ดึง traffic น้อยกว่า
add chain=bgp-out-isp2 rule="prepend bgp-path-as 2; accept;"

# Method 4: Selective advertisement
# Advertise บาง prefixes ให้เฉพาะ ISP1
add chain=bgp-out-isp1 rule="
    if (dst in 203.0.113.0/25) { accept; }
    reject;
"
add chain=bgp-out-isp2 rule="
    if (dst in 203.0.113.128/25) { accept; }
    reject;
"
```

---

## 3. Policy-Based Routing {#pbr}

```bash
# ============================================
# Policy-Based Routing (PBR)
# ============================================

# PBR ส่ง traffic ออก ISP ต่างๆ ตาม policy

# ISP1: 203.0.113.254 (Primary, faster)
# ISP2: 198.51.100.254 (Secondary, cheaper)

# Routing marks
/ip firewall mangle
# VoIP traffic → ISP1 (low latency)
add chain=prerouting protocol=udp dst-port=5060,10000-20000 \
    action=mark-routing new-routing-mark=to-isp1 passthrough=yes \
    comment="VoIP via ISP1"

# Backup traffic → ISP2 (cheaper)
add chain=prerouting dst-port=8080,21 protocol=tcp \
    action=mark-routing new-routing-mark=to-isp2 passthrough=yes \
    comment="Bulk via ISP2"

# Online gaming → ISP1
add chain=prerouting dst-address-list=gaming-servers \
    action=mark-routing new-routing-mark=to-isp1 \
    comment="Gaming via ISP1"

# Per-source routing
add chain=prerouting src-address=10.100.0.0/24 \
    action=mark-routing new-routing-mark=to-isp1 \
    comment="Dept A via ISP1"
add chain=prerouting src-address=10.200.0.0/24 \
    action=mark-routing new-routing-mark=to-isp2 \
    comment="Dept B via ISP2"

# Routing tables
/ip route
add dst-address=0.0.0.0/0 gateway=203.0.113.254 routing-mark=to-isp1
add dst-address=0.0.0.0/0 gateway=198.51.100.254 routing-mark=to-isp2

# Default route (ISP1)
add dst-address=0.0.0.0/0 gateway=203.0.113.254 distance=1
add dst-address=0.0.0.0/0 gateway=198.51.100.254 distance=2

# Address lists สำหรับ gaming servers
/ip firewall address-list
add list=gaming-servers address=103.x.x.0/24 comment="Steam CDN"
add list=gaming-servers address=104.x.x.0/24 comment="PSN"


# ============================================
# ECMP Load Balancing กับ routing marks
# ============================================

/ip route
# ECMP - 2 routes ด้วย distance เท่ากัน
add dst-address=0.0.0.0/0 gateway=203.0.113.254 distance=1
add dst-address=0.0.0.0/0 gateway=198.51.100.254 distance=1

# Per-connection load balancing
/ip firewall mangle
add chain=input in-interface=ether1-isp1 \
    action=mark-connection new-connection-mark=isp1-conn
add chain=input in-interface=ether2-isp2 \
    action=mark-connection new-connection-mark=isp2-conn

add chain=output connection-mark=isp1-conn \
    action=mark-routing new-routing-mark=to-isp1
add chain=output connection-mark=isp2-conn \
    action=mark-routing new-routing-mark=to-isp2

# Nth-based distribution (round robin)
add chain=prerouting connection-state=new \
    nth=2,1 action=mark-connection new-connection-mark=isp1-conn
add chain=prerouting connection-state=new \
    nth=2,2 action=mark-connection new-connection-mark=isp2-conn
```

---

## 4. Load Balancing Techniques {#load-balancing}

```bash
# ============================================
# Server Load Balancing
# ============================================

# Real Server Pool: Web servers
/ip firewall address-list
add list=web-servers address=10.10.0.1
add list=web-servers address=10.10.0.2
add list=web-servers address=10.10.0.3

# Load balance inbound HTTP traffic
/ip firewall nat
# Round robin DST NAT
add chain=dstnat dst-address=203.0.113.100 dst-port=80,443 \
    protocol=tcp nth=3,1 \
    action=dst-nat to-addresses=10.10.0.1

add chain=dstnat dst-address=203.0.113.100 dst-port=80,443 \
    protocol=tcp nth=3,2 \
    action=dst-nat to-addresses=10.10.0.2

add chain=dstnat dst-address=203.0.113.100 dst-port=80,443 \
    protocol=tcp nth=3,3 \
    action=dst-nat to-addresses=10.10.0.3

# Health check script
/system script
add name=lb-health-check source={
    :local servers {"10.10.0.1"; "10.10.0.2"; "10.10.0.3"}
    
    :foreach server in=$servers do={
        :local pingResult [/ping address=$server count=3]
        :if ($pingResult < 1) do={
            :log warning ("LB: Server " . $server . " is DOWN")
            # Could disable the NAT rule for this server
        }
    }
}

/system scheduler
add name=lb-check interval=30s on-event=lb-health-check
```

---

## 5. Traffic Matrices {#matrices}

```python
# ============================================
# Traffic Matrix Collection and Analysis
# ============================================

import librouteros
import numpy as np
from datetime import datetime
from typing import Dict, List, Tuple

class TrafficMatrixCollector:
    """เก็บ traffic matrix ระหว่าง router pairs"""
    
    def __init__(self, routers: List[Dict]):
        self.routers = routers
        self.matrix: Dict[Tuple[str, str], float] = {}
    
    def collect_flow_data(self, router: Dict) -> Dict:
        """เก็บ flow data จาก router"""
        conn = librouteros.connect(
            host=router["host"],
            username=router["username"],
            password=router["password"]
        )
        
        interfaces = list(conn('/interface/print'))
        flows = {}
        
        for iface in interfaces:
            i = dict(iface)
            name = i.get("name", "")
            flows[name] = {
                "rx_bps": int(i.get("rx-bits-per-second", 0)),
                "tx_bps": int(i.get("tx-bits-per-second", 0))
            }
        
        conn.close()
        return flows
    
    def build_matrix(self) -> np.ndarray:
        """สร้าง traffic matrix"""
        n = len(self.routers)
        matrix = np.zeros((n, n))
        
        for i, router in enumerate(self.routers):
            flows = self.collect_flow_data(router)
            
            for j, other_router in enumerate(self.routers):
                if i != j:
                    # Simplified - use total RX from direction of other router
                    # In real TE, use NetFlow/IPFIX data
                    key = f"{router['name']}_{other_router['name']}"
                    matrix[i][j] = self.matrix.get((router['name'], other_router['name']), 0)
        
        return matrix
    
    def find_congested_links(self, capacity_gbps: float = 1.0) -> List[dict]:
        """หา links ที่ congested"""
        congested = []
        threshold = capacity_gbps * 1e9 * 0.8  # 80% utilization
        
        for i, router in enumerate(self.routers):
            flows = self.collect_flow_data(router)
            
            for iface, data in flows.items():
                rx = data["rx_bps"]
                tx = data["tx_bps"]
                
                if max(rx, tx) > threshold:
                    congested.append({
                        "router": router["name"],
                        "interface": iface,
                        "utilization_pct": round(max(rx, tx) / (capacity_gbps * 1e9) * 100, 1),
                        "rx_gbps": round(rx / 1e9, 3),
                        "tx_gbps": round(tx / 1e9, 3)
                    })
        
        return congested
    
    def generate_te_recommendations(self) -> List[str]:
        """Generate TE recommendations"""
        recommendations = []
        congested = self.find_congested_links()
        
        for link in congested:
            recommendations.append(
                f"Link {link['router']}/{link['interface']} at {link['utilization_pct']}% - "
                f"Consider rerouting traffic via alternative path"
            )
        
        return recommendations
```

---

## 6. Lab: Multi-Path TE {#lab}

### Lab Topology

```
        ISP1 (100Mbps)    ISP2 (100Mbps)
              │                 │
        203.0.113.254    198.51.100.254
              │                 │
         ether1              ether2
              └────────────────┘
                      │
              [Border Router]
              AS65000
                      │
              [LAN: 10.0.0.0/24]
```

### Lab Steps

```bash
# Step 1: Setup dual ISP routing
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.254 distance=1
/ip route add dst-address=0.0.0.0/0 gateway=198.51.100.254 distance=2

# Step 2: PBR - VoIP via ISP1
/ip firewall mangle add chain=prerouting protocol=udp dst-port=5060 action=mark-routing new-routing-mark=to-isp1
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.254 routing-mark=to-isp1

# Step 3: PBR - Bulk via ISP2
/ip firewall mangle add chain=prerouting protocol=tcp dst-port=21,8080 action=mark-routing new-routing-mark=to-isp2
/ip route add dst-address=0.0.0.0/0 gateway=198.51.100.254 routing-mark=to-isp2

# Step 4: Test
# Test VoIP traffic path:
traceroute -u -p 5060 8.8.8.8  # Should exit via ISP1

# Test bulk traffic path:
traceroute -p 8080 8.8.8.8  # Should exit via ISP2

# Step 5: Monitor utilization
/interface print stats
```

### Verification Checklist

- [ ] Default route ผ่าน ISP1
- [ ] Failover ไป ISP2 เมื่อ ISP1 down
- [ ] VoIP traffic ไปผ่าน ISP1
- [ ] Bulk traffic ไปผ่าน ISP2
- [ ] Load balancing ทำงาน
- [ ] Traffic monitoring แสดงถูกต้อง

---

## Summary

Part นี้ครอบคลุม:
- **BGP Traffic Engineering** - LOCAL_PREF, MED, prepend
- **Policy-Based Routing** - per-application, per-source
- **ECMP** Load Balancing
- **Server Load Balancing** ด้วย DST NAT
- **Traffic Matrix** collection

---

[← Part 93: DDoS Protection](part-093-ddos-protection.md) | [Part 95: Realtime Analytics →](part-095-realtime-analytics.md)
