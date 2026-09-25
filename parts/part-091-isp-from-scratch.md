# Part 91: ISP Setup จาก Scratch

## สารบัญ
1. [ISP Planning](#planning)
2. [Network Design](#design)
3. [Core Infrastructure](#core)
4. [Customer Access Network](#access)
5. [ISP Services](#services)
6. [Operations Center](#noc)
7. [Lab: Complete ISP Setup](#lab)

---

## 1. ISP Planning {#planning}

### ISP Infrastructure Requirements

| Component | Description | Tool |
|-----------|-------------|------|
| AS Number | BGP routing identity | APNIC/THNIC |
| IP Addresses | IPv4/IPv6 blocks | APNIC |
| Upstream ISP | Transit connectivity | Tier-1/Tier-2 ISP |
| Core Router | High-capacity routing | CCR2004/CCR2216 |
| Edge Router | Customer aggregation | CCR1009/CCR2004 |
| Access Layer | DSLAM/OLT for last mile | Vendor specific |
| OSS/BSS | Operations & Billing | Custom/Commercial |

### IP Address Planning

```
Assigned block: 203.0.113.0/22 (1022 usable IPs)
├── 203.0.113.0/24 - Infrastructure (router loopbacks, P2P links)
├── 203.0.114.0/24 - Customer static IPs
├── 203.0.115.0/24 - Customer PPPoE pools
└── 203.0.116.0/23 - Future expansion

CGNAT (RFC 6598): 100.64.0.0/10
├── 100.64.0.0/16 - PPPoE customers (no static IP)

IPv6 block: 2001:db8::/32 (from APNIC)
├── 2001:db8:0::/48 - Infrastructure
├── 2001:db8:1::/48 - Customer /64 pools
└── 2001:db8:100::/48 - Customer /56 delegation
```

---

## 2. Network Design {#design}

```
Internet (Upstream ISP)
    │ eBGP AS65001
    │
[Border Router / CCR2216]
AS65000 router-id=203.0.113.1
    │
    │ iBGP/OSPF
    │
[Core Router / CCR2004]
    │
    │ MPLS/OSPF
    │
[Edge Routers / CCR1009]
    │
    │ PPPoE/IPoE
    │
[Access Layer - DSLAM/OLT]
    │
[Customers]
```

---

## 3. Core Infrastructure {#core}

```bash
# ============================================
# Border Router Configuration
# ============================================

/system identity set name=border-01
/ip address
add address=203.0.113.1/30 interface=ether1-upstream comment="Upstream ISP link"
add address=10.0.0.1/30 interface=ether2-core comment="Core link"

# OSPF กับ Core
/routing ospf instance
add name=default router-id=203.0.113.1 as=65000

/routing ospf area
add name=backbone instance=default area-id=0.0.0.0

/routing ospf interface-template
add interfaces=ether2-core area=backbone

# eBGP upstream
/routing bgp instance
add name=default as=65000 router-id=203.0.113.1

/routing bgp peer
add name=upstream-isp address=203.0.113.254 remote-as=65001 \
    address-families=ip,ipv6 \
    input.filter=bgp-in-upstream \
    output.filter=bgp-out-upstream

# Filter: รับ default route จาก upstream
/routing filter rule
add chain=bgp-in-upstream rule="
    if (dst == 0.0.0.0/0) { accept; }
    reject;
"

# Filter: advertise our prefix เท่านั้น
add chain=bgp-out-upstream rule="
    if (dst in 203.0.113.0/22) { accept; }
    reject;
"

# Advertise our block
/routing bgp network
add network=203.0.113.0/22

# Default route
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.254


# ============================================
# Core Router Configuration
# ============================================

/system identity set name=core-01
/ip address
add address=10.0.0.2/30 interface=ether1-border comment="Border link"
add address=10.0.1.1/30 interface=ether2-edge1 comment="Edge-01 link"
add address=10.0.1.5/30 interface=ether3-edge2 comment="Edge-02 link"
add address=10.0.0.254/32 interface=lo0 comment="Loopback (router-id)"

# OSPF
/routing ospf interface-template
add interfaces=ether1-border,ether2-edge1,ether3-edge2 area=backbone

# iBGP route-reflector
/routing bgp instance
add name=default as=65000 router-id=10.0.0.254

/routing bgp peer
add name=edge-01 address=10.0.1.2 remote-as=65000 route-reflect=yes
add name=edge-02 address=10.0.1.6 remote-as=65000 route-reflect=yes
add name=border-01 address=10.0.0.1 remote-as=65000
```

---

## 4. Customer Access Network {#access}

```bash
# ============================================
# Edge Router - PPPoE Server
# ============================================

/system identity set name=edge-01

# Interfaces
/ip address
add address=10.0.1.2/30 interface=ether1-core comment="Core link"
add address=10.0.2.1/32 interface=lo0 comment="Loopback"

# OSPF
/routing ospf interface-template
add interfaces=ether1-core area=backbone

# PPPoE server
/interface pppoe-server server
add name=pppoe-wan interface=ether2-dslam \
    service-name=ISP \
    authentication=chap,pap \
    max-sessions=1000 \
    keepalive-timeout=60 \
    one-session-per-host=yes \
    comment="PPPoE server"

# PPP Profile
/ppp profile
add name=pppoe-standard \
    local-address=203.0.115.254 \
    remote-address=pppoe-pool \
    dns-server=8.8.8.8,8.8.4.4 \
    rate-limit=20M/20M \
    session-timeout=86400 \
    only-one=yes

# IP Pool สำหรับ PPPoE customers
/ip pool
add name=pppoe-pool ranges=203.0.115.1-203.0.115.200

# RADIUS สำหรับ authentication
/radius
add address=10.10.0.100 secret=radius-secret service=ppp timeout=3000ms

/ppp aaa
set use-radius=yes accounting=yes interim-update=5m

# Firewall สำหรับ customer isolation
/ip firewall filter
add chain=forward in-interface=pppoe-wan out-interface=pppoe-wan \
    action=drop comment="Block customer to customer"


# ============================================
# RADIUS Server (FreeRADIUS)
# ============================================
```

```
# /etc/freeradius/3.0/users
# ตัวอย่าง static user entries
user001 Cleartext-Password := "password123"
    Session-Timeout = 86400,
    Framed-IP-Address = 203.0.114.100,
    Mikrotik-Rate-Limit = "20M/20M",
    Reply-Message = "Welcome user001"

user002 Cleartext-Password := "password456"
    Mikrotik-Rate-Limit = "50M/50M",
    Reply-Message = "Welcome user002"
```

---

## 5. ISP Services {#services}

```bash
# ============================================
# DNS Resolver
# ============================================

/ip dns
set servers=8.8.8.8,1.1.1.1 allow-remote-requests=yes \
    cache-max-ttl=1d cache-size=10000KiB

# DNS Firewall (block malware domains)
/ip dns static
add name=malware.example.com address=0.0.0.0 comment="Block malware"

# ============================================
# DHCP สำหรับ IPoE customers
# ============================================

/ip dhcp-server
add name=dhcp-customer interface=ether3-customers \
    address-pool=customer-pool lease-time=24h

/ip pool add name=customer-pool ranges=203.0.115.201-203.0.115.250

/ip dhcp-server network
add address=203.0.115.0/24 gateway=203.0.115.254 \
    dns-server=10.0.2.1


# ============================================
# QoS / Traffic Shaping
# ============================================

/ip firewall mangle
# Mark PPPoE sessions by subscriber
add chain=prerouting in-interface-list=pppoe-interfaces \
    action=mark-connection new-connection-mark=pppoe-conn

/queue tree
add name=pppoe-users parent=ether2-dslam

# Per-user PCQ
/queue type
add name=pcq-user-up kind=pcq pcq-classifier=src-address pcq-rate=20M
add name=pcq-user-down kind=pcq pcq-classifier=dst-address pcq-rate=20M

/queue tree
add name=upload-users parent=ether1-core queue=pcq-user-up max-limit=1G
add name=download-users parent=ether2-dslam queue=pcq-user-down max-limit=1G
```

---

## 6. Operations Center {#noc}

```python
# ============================================
# NOC Dashboard API
# ============================================

from fastapi import FastAPI
import librouteros
from datetime import datetime

app = FastAPI(title="ISP NOC API")

ROUTERS = [
    {"host": "10.0.0.1", "name": "border-01", "username": "admin", "password": "pass"},
    {"host": "10.0.0.2", "name": "core-01", "username": "admin", "password": "pass"},
    {"host": "10.0.1.2", "name": "edge-01", "username": "admin", "password": "pass"},
]

@app.get("/api/status")
async def get_network_status():
    """ดึงสถานะของทุก routers"""
    results = []
    
    for router in ROUTERS:
        try:
            conn = librouteros.connect(
                host=router["host"],
                username=router["username"],
                password=router["password"],
                timeout=5
            )
            
            resource = dict(list(conn('/system/resource/print'))[0])
            
            # PPPoE sessions (edge only)
            pppoe_count = 0
            try:
                sessions = list(conn('/ppp/active/print'))
                pppoe_count = len(sessions)
            except:
                pass
            
            conn.close()
            
            results.append({
                "name": router["name"],
                "host": router["host"],
                "status": "online",
                "cpu_load": resource.get("cpu-load"),
                "uptime": resource.get("uptime"),
                "pppoe_sessions": pppoe_count
            })
        except Exception as e:
            results.append({
                "name": router["name"],
                "host": router["host"],
                "status": "offline",
                "error": str(e)
            })
    
    return {
        "timestamp": datetime.utcnow().isoformat(),
        "routers": results,
        "total_online": sum(1 for r in results if r["status"] == "online"),
        "total_offline": sum(1 for r in results if r["status"] == "offline")
    }

@app.get("/api/customers/active")
async def get_active_customers():
    """ดึง active PPPoE sessions"""
    edge_router = ROUTERS[2]  # edge-01
    
    try:
        conn = librouteros.connect(
            host=edge_router["host"],
            username=edge_router["username"],
            password=edge_router["password"]
        )
        
        sessions = list(conn('/ppp/active/print'))
        conn.close()
        
        return {
            "active_sessions": len(sessions),
            "sessions": [{"name": dict(s).get("name"), "address": dict(s).get("address")} 
                        for s in sessions[:50]]
        }
    except Exception as e:
        return {"error": str(e)}
```

---

## 7. Lab: Complete ISP Setup {#lab}

### Lab Steps Summary

```bash
# Phase 1: Infrastructure
/system identity set name=border-01
/ip address add address=203.0.113.1/30 interface=ether1
/routing bgp instance add name=default as=65000

# Phase 2: Core
/routing ospf instance add name=default router-id=10.0.0.254
# iBGP route-reflector setup

# Phase 3: Edge/Access
/interface pppoe-server server add interface=ether2 service-name=ISP
/radius add address=10.10.0.100 secret=radius-secret

# Phase 4: Customer test
# Connect test PPPoE client
# username: testuser / password: testpass

# Phase 5: Services
/ip dns set allow-remote-requests=yes
# Setup billing system (Part 66)

# Phase 6: Monitoring
# Deploy Prometheus + Grafana (Part 87)
```

### Checklist

- [ ] Border router BGP established กับ upstream
- [ ] Core OSPF/iBGP working
- [ ] Edge PPPoE server running
- [ ] RADIUS authentication working
- [ ] Customer gets IP address
- [ ] Customer ping internet
- [ ] QoS rate limiting applies
- [ ] NOC dashboard showing stats

---

## Summary

Part นี้ครอบคลุม:
- **ISP planning** - AS, IP, topology
- **Core infrastructure** - border, core routers
- **Customer access** - PPPoE, RADIUS
- **ISP services** - DNS, DHCP, QoS
- **NOC dashboard** API

---

[← Part 90: Performance](part-090-performance.md) | [Part 92: CGNAT →](part-092-cgnat.md)
