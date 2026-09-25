# Part 81: MikroTik ใน Cloud Environments

## สารบัญ
1. [VPC Integration](#vpc)
2. [Site-to-Cloud VPN](#s2c-vpn)
3. [Cloud-to-Cloud VPN](#c2c-vpn)
4. [Transit VPC/Gateway](#transit)
5. [SD-WAN with CHR](#sdwan)
6. [Cloud Routing](#routing)
7. [Lab: Multi-Cloud Setup](#lab)

---

## 1. VPC Integration {#vpc}

MikroTik CHR ใน VPC สามารถทำหน้าที่ router และ firewall สำหรับ cloud infrastructure

```bash
# ============================================
# VPC Integration Configuration
# ============================================

# CHR เป็น router ระหว่าง VPC subnets
# VPC: 10.0.0.0/16
# Public Subnet: 10.0.1.0/24 (DMZ)
# Private Subnet 1: 10.0.2.0/24 (Web Servers)
# Private Subnet 2: 10.0.3.0/24 (Database)

# CHR interfaces
/ip address
add address=10.0.1.10/24 interface=ether1 comment="Public/DMZ"
add address=10.0.2.1/24 interface=ether2 comment="Web VLAN"
add address=10.0.3.1/24 interface=ether3 comment="DB VLAN"

# Routing
/ip route
add dst-address=0.0.0.0/0 gateway=10.0.1.1 comment="IGW via cloud gateway"

# Firewall for VPC isolation
/ip firewall filter
# Allow Web servers to access DB
add chain=forward in-interface=ether2 out-interface=ether3 \
    dst-port=5432,3306 protocol=tcp action=accept

# Block DB from reaching internet
add chain=forward in-interface=ether3 out-interface=ether1 \
    action=drop comment="DB cannot reach internet"

# Allow internet to Web servers (80,443 only)
add chain=forward in-interface=ether1 out-interface=ether2 \
    dst-port=80,443 protocol=tcp action=accept

add chain=forward action=drop comment="Default deny"

# NAT สำหรับ private subnets
/ip firewall nat
add chain=srcnat out-interface=ether1 \
    src-address=10.0.2.0/23 action=masquerade \
    comment="NAT for private subnets"
```

---

## 2. Site-to-Cloud VPN {#s2c-vpn}

```bash
# ============================================
# IPsec Site-to-Cloud VPN (AWS)
# ============================================

# CHR (Cloud VPN endpoint)
# Public IP: 54.x.x.x
# Private VPC: 10.0.0.0/16

# On-premise Router
# Public IP: 203.0.113.1
# Local Network: 192.168.1.0/24

# ---- CHR Configuration (AWS Side) ----

/ip ipsec proposal
add name=aws-proposal auth-algorithms=sha256 \
    enc-algorithms=aes-256-cbc \
    lifetime=1h

/ip ipsec profile
add name=aws-profile hash-algorithm=sha256 \
    enc-algorithm=aes-256 \
    lifetime=8h \
    dh-group=modp2048

/ip ipsec peer
add name=onprem-peer address=203.0.113.1 \
    profile=aws-profile exchange-mode=ike2

/ip ipsec identity
add peer=onprem-peer auth-method=pre-shared-key \
    secret=SharedSecret@2024!

/ip ipsec policy
add src-address=10.0.0.0/16 dst-address=192.168.1.0/24 \
    peer=onprem-peer proposal=aws-proposal \
    action=encrypt tunnel=yes

# ---- On-Premise MikroTik Configuration ----

/ip ipsec proposal
add name=cloud-proposal auth-algorithms=sha256 \
    enc-algorithms=aes-256-cbc lifetime=1h

/ip ipsec profile
add name=cloud-profile hash-algorithm=sha256 \
    enc-algorithm=aes-256 lifetime=8h dh-group=modp2048

/ip ipsec peer
add name=aws-peer address=54.x.x.x \
    profile=cloud-profile exchange-mode=ike2

/ip ipsec identity
add peer=aws-peer auth-method=pre-shared-key \
    secret=SharedSecret@2024!

/ip ipsec policy
add src-address=192.168.1.0/24 dst-address=10.0.0.0/16 \
    peer=aws-peer proposal=cloud-proposal \
    action=encrypt tunnel=yes

# ตรวจสอบ VPN
/ip ipsec active-peers print
/ip ipsec installed-sa print
```

---

## 3. Cloud-to-Cloud VPN {#c2c-vpn}

```bash
# ============================================
# WireGuard Cloud-to-Cloud VPN
# ============================================

# CHR-AWS (10.0.0.0/16) <---> CHR-GCP (172.16.0.0/16)

# AWS CHR
/interface wireguard
add name=wg-gcp listen-port=51820 private-key="..."

/interface wireguard peers
add interface=wg-gcp public-key="GCP-Public-Key" \
    endpoint-address=35.x.x.x endpoint-port=51820 \
    allowed-address=172.16.0.0/16 \
    comment="GCP peer"

/ip address
add address=10.200.0.1/30 interface=wg-gcp comment="WG tunnel IP"

/ip route
add dst-address=172.16.0.0/16 gateway=wg-gcp \
    comment="Route to GCP via WireGuard"

# GCP CHR
/interface wireguard
add name=wg-aws listen-port=51820 private-key="..."

/interface wireguard peers
add interface=wg-aws public-key="AWS-Public-Key" \
    endpoint-address=54.x.x.x endpoint-port=51820 \
    allowed-address=10.0.0.0/16 \
    comment="AWS peer"

/ip address
add address=10.200.0.2/30 interface=wg-aws comment="WG tunnel IP"

/ip route
add dst-address=10.0.0.0/16 gateway=wg-aws \
    comment="Route to AWS via WireGuard"
```

---

## 4. Transit VPC/Gateway {#transit}

```bash
# ============================================
# CHR เป็น Transit Gateway
# ============================================

# Hub CHR เชื่อมต่อ multiple spokes
# Hub: 10.100.0.0/30 (transit network)
# Spoke 1 (AWS): 10.0.0.0/16
# Spoke 2 (GCP): 172.16.0.0/16
# Spoke 3 (On-Premise): 192.168.0.0/16

# Hub CHR - WireGuard interfaces
/interface wireguard
add name=wg-spoke1 listen-port=51821
add name=wg-spoke2 listen-port=51822
add name=wg-spoke3 listen-port=51823

# Route table บน hub
/ip route
add dst-address=10.0.0.0/16 gateway=wg-spoke1
add dst-address=172.16.0.0/16 gateway=wg-spoke2
add dst-address=192.168.0.0/16 gateway=wg-spoke3

# Enable IP forwarding ระหว่าง spokes
/ip settings set ip-forward=yes

# Firewall สำหรับ transit
/ip firewall filter
add chain=forward in-interface=wg-spoke1 out-interface=wg-spoke2 action=accept
add chain=forward in-interface=wg-spoke2 out-interface=wg-spoke1 action=accept
# Spoke 1 <---> Spoke 3
add chain=forward in-interface=wg-spoke1 out-interface=wg-spoke3 action=accept
add chain=forward in-interface=wg-spoke3 out-interface=wg-spoke1 action=accept
```

---

## 5. SD-WAN with CHR {#sdwan}

```python
# ============================================
# SD-WAN Controller สำหรับ CHR fleet
# ============================================

from fastapi import FastAPI, Depends
from pydantic import BaseModel
import librouteros
import asyncio
from typing import Dict, List

app = FastAPI(title="CHR SD-WAN Controller")

class CHRNode(BaseModel):
    name: str
    host: str
    username: str
    password: str
    location: str
    role: str  # hub, spoke

class SDWANController:
    def __init__(self):
        self.nodes: Dict[str, CHRNode] = {}
        self.topology = {}
    
    def add_node(self, node: CHRNode):
        self.nodes[node.name] = node
    
    async def get_node_status(self, node_name: str) -> dict:
        node = self.nodes.get(node_name)
        if not node:
            return {"error": "Node not found"}
        
        try:
            conn = librouteros.connect(
                host=node.host,
                username=node.username,
                password=node.password,
                timeout=5
            )
            
            # Get WireGuard peers status
            wg_peers = list(conn('/interface/wireguard/peers/print'))
            
            # Get routes
            routes = list(conn('/ip/route/print'))
            
            conn.close()
            
            return {
                "name": node_name,
                "host": node.host,
                "wg_peers": len(wg_peers),
                "active_tunnels": sum(1 for p in wg_peers if dict(p).get("rx") != "0"),
                "routes": len(routes),
                "status": "online"
            }
        except Exception as e:
            return {"name": node_name, "status": "offline", "error": str(e)}
    
    async def get_all_status(self) -> List[dict]:
        tasks = [self.get_node_status(name) for name in self.nodes]
        return await asyncio.gather(*tasks)

controller = SDWANController()

@app.get("/nodes")
async def list_nodes():
    return await controller.get_all_status()

@app.post("/nodes")
async def add_node(node: CHRNode):
    controller.add_node(node)
    return {"message": f"Node {node.name} added"}
```

---

## 6. Lab: Multi-Cloud Setup {#lab}

### Lab Architecture

```
Internet
    │
    ├── [AWS CHR Hub]          ├── [GCP CHR Spoke]
    │   54.x.x.x               │   35.x.x.x
    │   VPC: 10.0.0.0/16       │   VPC: 172.16.0.0/16
    │                          │
    │           WireGuard VPN
    │   ←─────────────────────→
    │
    └── [On-Premise]
        203.0.113.1
        LAN: 192.168.1.0/24
        IPsec to AWS CHR
```

### Lab Steps

```bash
# Step 1: Deploy CHR on AWS and GCP (Part 80)

# Step 2: Configure WireGuard on AWS CHR
/interface wireguard add name=wg0 listen-port=51820
/interface wireguard peers add interface=wg0 public-key="GCP-KEY" allowed-address=172.16.0.0/16

# Step 3: Configure WireGuard on GCP CHR
/interface wireguard add name=wg0 listen-port=51820
/interface wireguard peers add interface=wg0 public-key="AWS-KEY" endpoint-address=54.x.x.x endpoint-port=51820 allowed-address=10.0.0.0/16

# Step 4: Add routes
# AWS: /ip route add dst-address=172.16.0.0/16 gateway=wg0
# GCP: /ip route add dst-address=10.0.0.0/16 gateway=wg0

# Step 5: Test connectivity
/ping 172.16.0.1  # From AWS, ping GCP
```

### Verification Checklist

- [ ] CHR instances running บน AWS/GCP
- [ ] WireGuard tunnels established
- [ ] Routes ระหว่าง VPCs ทำงาน
- [ ] Cross-cloud ping ทำงาน
- [ ] Firewall rules อนุญาต traffic ที่ต้องการ
- [ ] VPN monitoring ทำงาน

> **Tip:** ใช้ latency metrics ตัดสินว่าใช้ route ใด

> **Note:** Cloud provider charges สำหรับ data transfer ระหว่าง regions และ clouds

---

## Summary

Part นี้ครอบคลุม:
- **VPC integration** กับ routing/firewall
- **Site-to-Cloud VPN** ด้วย IPsec
- **Cloud-to-Cloud VPN** ด้วย WireGuard
- **Transit architecture** สำหรับ multi-cloud
- **SD-WAN controller** Python API

---

[← Part 80: CHR Cloud](part-080-chr-cloud.md) | [Part 82: Kubernetes →](part-082-kubernetes.md)
