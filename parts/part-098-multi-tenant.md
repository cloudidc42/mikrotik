# Part 98: Multi-tenancy และ Network-as-a-Service

## สารบัญ
1. [Multi-tenancy Concepts](#concepts)
2. [Network Isolation](#isolation)
3. [NaaS Architecture](#naas)
4. [Tenant Management API](#api)
5. [Billing Integration](#billing)
6. [Lab: Multi-tenant Setup](#lab)

---

## 1. Multi-tenancy Concepts {#concepts}

### Isolation Levels

| Level | Method | Use Case |
|-------|--------|----------|
| Physical | Dedicated hardware per tenant | Maximum security |
| VLAN | 802.1Q VLANs | Cost-effective isolation |
| VRF | Virtual routing instances | L3 isolation |
| MPLS/VPN | L3VPN กับ BGP | ISP multi-tenant |
| Overlay | VXLAN, GRE tunnels | Cloud networking |

### NaaS Model

```
[Tenant A] ──── VLAN 100 ──┐
[Tenant B] ──── VLAN 200 ──┤ MikroTik + VRF → Internet
[Tenant C] ──── VLAN 300 ──┘
```

---

## 2. Network Isolation {#isolation}

```bash
# ============================================
# Multi-tenant VLAN + VRF Setup
# ============================================

# Tenant A - VLAN 100, VRF tenant-a
/interface vlan
add name=vlan100-tenant-a vlan-id=100 interface=ether2-trunk

/ip vrf
add name=tenant-a interfaces=vlan100-tenant-a

/ip address
add address=10.100.0.1/24 interface=vlan100-tenant-a \
    comment="Tenant A gateway"

# Tenant B - VLAN 200, VRF tenant-b
/interface vlan
add name=vlan200-tenant-b vlan-id=200 interface=ether2-trunk

/ip vrf
add name=tenant-b interfaces=vlan200-tenant-b

/ip address
add address=10.200.0.1/24 interface=vlan200-tenant-b \
    comment="Tenant B gateway"

# Tenant C - VLAN 300, VRF tenant-c
/interface vlan
add name=vlan300-tenant-c vlan-id=300 interface=ether2-trunk

/ip vrf
add name=tenant-c interfaces=vlan300-tenant-c

/ip address
add address=10.300.0.1/24 interface=vlan300-tenant-c \
    comment="Tenant C gateway"

# Per-tenant NAT (แต่ละ tenant ใช้ IP pool ของตัวเอง)
/ip firewall nat
add chain=srcnat out-interface=ether1-wan \
    src-address=10.100.0.0/24 \
    action=src-nat to-addresses=203.0.113.10 \
    comment="Tenant A NAT"

add chain=srcnat out-interface=ether1-wan \
    src-address=10.200.0.0/24 \
    action=src-nat to-addresses=203.0.113.11 \
    comment="Tenant B NAT"

add chain=srcnat out-interface=ether1-wan \
    src-address=10.300.0.0/24 \
    action=src-nat to-addresses=203.0.113.12 \
    comment="Tenant C NAT"

# Block cross-tenant traffic
/ip firewall filter
add chain=forward in-interface=vlan100-tenant-a \
    out-interface=vlan200-tenant-b action=drop \
    comment="Block A→B"
add chain=forward in-interface=vlan100-tenant-a \
    out-interface=vlan300-tenant-c action=drop \
    comment="Block A→C"
add chain=forward in-interface=vlan200-tenant-b \
    out-interface=vlan100-tenant-a action=drop \
    comment="Block B→A"
add chain=forward in-interface=vlan200-tenant-b \
    out-interface=vlan300-tenant-c action=drop \
    comment="Block B→C"

# Per-tenant bandwidth limits
/queue simple
add name=tenant-a-limit target=10.100.0.0/24 \
    max-limit=100M/100M comment="Tenant A: 100Mbps"
add name=tenant-b-limit target=10.200.0.0/24 \
    max-limit=200M/200M comment="Tenant B: 200Mbps"
add name=tenant-c-limit target=10.300.0.0/24 \
    max-limit=50M/50M comment="Tenant C: 50Mbps"


# ============================================
# DHCP per tenant
# ============================================

/ip dhcp-server
add name=dhcp-tenant-a interface=vlan100-tenant-a \
    address-pool=pool-tenant-a lease-time=12h

add name=dhcp-tenant-b interface=vlan200-tenant-b \
    address-pool=pool-tenant-b lease-time=12h

add name=dhcp-tenant-c interface=vlan300-tenant-c \
    address-pool=pool-tenant-c lease-time=12h

/ip pool
add name=pool-tenant-a ranges=10.100.0.10-10.100.0.200
add name=pool-tenant-b ranges=10.200.0.10-10.200.0.200
add name=pool-tenant-c ranges=10.300.0.10-10.300.0.200

/ip dhcp-server network
add address=10.100.0.0/24 gateway=10.100.0.1 dns-server=8.8.8.8
add address=10.200.0.0/24 gateway=10.200.0.1 dns-server=8.8.8.8
add address=10.300.0.0/24 gateway=10.300.0.1 dns-server=8.8.8.8
```

---

## 3. NaaS Architecture {#naas}

```python
# naas_controller.py - Network-as-a-Service controller
from dataclasses import dataclass, field
from typing import List, Optional, Dict
import librouteros
import ipaddress

@dataclass
class TenantNetwork:
    tenant_id: str
    name: str
    vlan_id: int
    subnet: str
    gateway: str
    bandwidth_mbps: int
    public_ip: Optional[str] = None
    status: str = "pending"

class NaaSController:
    """NaaS controller สำหรับจัดการ multi-tenant networks"""
    
    def __init__(self, router_host: str, username: str, password: str):
        self.router_host = router_host
        self.username = username
        self.password = password
        self.tenants: Dict[str, TenantNetwork] = {}
    
    def _connect(self):
        return librouteros.connect(
            host=self.router_host,
            username=self.username,
            password=self.password
        )
    
    def provision_tenant(self, tenant: TenantNetwork) -> dict:
        """Provision new tenant network"""
        conn = self._connect()
        
        results = {"tenant_id": tenant.tenant_id, "steps": []}
        
        try:
            # Step 1: Create VLAN interface
            conn('/interface/vlan/add', **{
                "name": f"vlan{tenant.vlan_id}-{tenant.tenant_id}",
                "vlan-id": str(tenant.vlan_id),
                "interface": "ether2-trunk",
                "comment": f"NaaS tenant {tenant.tenant_id}"
            })
            results["steps"].append("vlan_created")
            
            # Step 2: Assign gateway IP
            conn('/ip/address/add', **{
                "address": tenant.gateway + "/" + str(ipaddress.IPv4Network(tenant.subnet, strict=False).prefixlen),
                "interface": f"vlan{tenant.vlan_id}-{tenant.tenant_id}",
                "comment": f"Gateway for {tenant.tenant_id}"
            })
            results["steps"].append("gateway_assigned")
            
            # Step 3: Create DHCP pool
            pool_range = self._calculate_dhcp_range(tenant.subnet)
            conn('/ip/pool/add', **{
                "name": f"pool-{tenant.tenant_id}",
                "ranges": pool_range
            })
            results["steps"].append("dhcp_pool_created")
            
            # Step 4: Create DHCP server
            conn('/ip/dhcp-server/add', **{
                "name": f"dhcp-{tenant.tenant_id}",
                "interface": f"vlan{tenant.vlan_id}-{tenant.tenant_id}",
                "address-pool": f"pool-{tenant.tenant_id}",
                "lease-time": "12h"
            })
            
            network = ipaddress.IPv4Network(tenant.subnet, strict=False)
            conn('/ip/dhcp-server/network/add', **{
                "address": str(network),
                "gateway": tenant.gateway,
                "dns-server": "8.8.8.8,8.8.4.4"
            })
            results["steps"].append("dhcp_configured")
            
            # Step 5: NAT if public IP assigned
            if tenant.public_ip:
                conn('/ip/firewall/nat/add', **{
                    "chain": "srcnat",
                    "src-address": tenant.subnet,
                    "out-interface": "ether1-wan",
                    "action": "src-nat",
                    "to-addresses": tenant.public_ip,
                    "comment": f"NAT {tenant.tenant_id}"
                })
                results["steps"].append("nat_configured")
            
            # Step 6: Bandwidth limit
            conn('/queue/simple/add', **{
                "name": f"bw-{tenant.tenant_id}",
                "target": tenant.subnet,
                "max-limit": f"{tenant.bandwidth_mbps}M/{tenant.bandwidth_mbps}M",
                "comment": f"Bandwidth {tenant.tenant_id}"
            })
            results["steps"].append("bandwidth_configured")
            
            tenant.status = "active"
            self.tenants[tenant.tenant_id] = tenant
            results["status"] = "success"
            
        except Exception as e:
            results["status"] = "failed"
            results["error"] = str(e)
        
        conn.close()
        return results
    
    def deprovision_tenant(self, tenant_id: str) -> dict:
        """Remove tenant network"""
        conn = self._connect()
        results = {"tenant_id": tenant_id, "removed": []}
        
        try:
            # Remove queue
            queues = list(conn('/queue/simple/print', **{
                "?comment": f"Bandwidth {tenant_id}"
            }))
            for q in queues:
                conn('/queue/simple/remove', **{"numbers": dict(q).get(".id")})
            results["removed"].append("bandwidth")
            
            # Remove NAT
            nat_rules = list(conn('/ip/firewall/nat/print', **{
                "?comment": f"NAT {tenant_id}"
            }))
            for r in nat_rules:
                conn('/ip/firewall/nat/remove', **{"numbers": dict(r).get(".id")})
            results["removed"].append("nat")
            
            # Remove DHCP server
            servers = list(conn('/ip/dhcp-server/print', **{
                "?name": f"dhcp-{tenant_id}"
            }))
            for s in servers:
                conn('/ip/dhcp-server/remove', **{"numbers": dict(s).get(".id")})
            results["removed"].append("dhcp")
            
            # Remove IP address
            addresses = list(conn('/ip/address/print', **{
                "?comment": f"Gateway for {tenant_id}"
            }))
            for a in addresses:
                conn('/ip/address/remove', **{"numbers": dict(a).get(".id")})
            results["removed"].append("ip_address")
            
            # Remove VLAN
            tenant = self.tenants.get(tenant_id)
            if tenant:
                vlans = list(conn('/interface/vlan/print', **{
                    "?comment": f"NaaS tenant {tenant_id}"
                }))
                for v in vlans:
                    conn('/interface/vlan/remove', **{"numbers": dict(v).get(".id")})
                results["removed"].append("vlan")
                
                del self.tenants[tenant_id]
            
            results["status"] = "success"
        
        except Exception as e:
            results["status"] = "failed"
            results["error"] = str(e)
        
        conn.close()
        return results
    
    def get_tenant_stats(self, tenant_id: str) -> dict:
        """ดึง statistics ของ tenant"""
        tenant = self.tenants.get(tenant_id)
        if not tenant:
            return {"error": "Tenant not found"}
        
        conn = self._connect()
        
        try:
            iface_name = f"vlan{tenant.vlan_id}-{tenant_id}"
            
            interfaces = list(conn('/interface/print', **{
                "?name": iface_name
            }))
            
            if interfaces:
                iface = dict(interfaces[0])
                stats = {
                    "tenant_id": tenant_id,
                    "vlan_id": tenant.vlan_id,
                    "rx_bps": int(iface.get("rx-bits-per-second", 0)),
                    "tx_bps": int(iface.get("tx-bits-per-second", 0)),
                    "rx_bytes": int(iface.get("rx-bytes", 0)),
                    "tx_bytes": int(iface.get("tx-bytes", 0)),
                    "status": "active" if iface.get("running") == "true" else "inactive"
                }
            else:
                stats = {"error": "Interface not found"}
        
        except Exception as e:
            stats = {"error": str(e)}
        
        conn.close()
        return stats
    
    def _calculate_dhcp_range(self, subnet: str) -> str:
        """คำนวณ DHCP range จาก subnet"""
        network = ipaddress.IPv4Network(subnet, strict=False)
        hosts = list(network.hosts())
        
        # Use IPs from .10 to second-to-last
        start = str(hosts[9]) if len(hosts) > 10 else str(hosts[0])
        end = str(hosts[-2]) if len(hosts) > 2 else str(hosts[-1])
        
        return f"{start}-{end}"
```

---

## 4. Tenant Management API {#api}

```python
# tenant_api.py - FastAPI สำหรับ NaaS management
from fastapi import FastAPI, HTTPException
from pydantic import BaseModel
from typing import Optional

app = FastAPI(title="NaaS Management API")

controller = NaaSController("10.0.0.1", "admin", "secure_password")

class TenantCreateRequest(BaseModel):
    tenant_id: str
    name: str
    vlan_id: int
    subnet: str
    gateway: str
    bandwidth_mbps: int
    public_ip: Optional[str] = None

@app.post("/tenants")
async def create_tenant(req: TenantCreateRequest):
    """Provision new tenant"""
    if req.tenant_id in controller.tenants:
        raise HTTPException(400, "Tenant already exists")
    
    tenant = TenantNetwork(
        tenant_id=req.tenant_id,
        name=req.name,
        vlan_id=req.vlan_id,
        subnet=req.subnet,
        gateway=req.gateway,
        bandwidth_mbps=req.bandwidth_mbps,
        public_ip=req.public_ip
    )
    
    result = controller.provision_tenant(tenant)
    
    if result.get("status") != "success":
        raise HTTPException(500, f"Provisioning failed: {result.get('error')}")
    
    return {"message": "Tenant provisioned", "details": result}

@app.get("/tenants/{tenant_id}/stats")
async def get_tenant_stats(tenant_id: str):
    """ดึง tenant statistics"""
    stats = controller.get_tenant_stats(tenant_id)
    
    if "error" in stats:
        raise HTTPException(404, stats["error"])
    
    return stats

@app.delete("/tenants/{tenant_id}")
async def delete_tenant(tenant_id: str):
    """Remove tenant"""
    result = controller.deprovision_tenant(tenant_id)
    
    if result.get("status") != "success":
        raise HTTPException(500, f"Deprovisioning failed: {result.get('error')}")
    
    return {"message": "Tenant removed", "details": result}

@app.get("/tenants")
async def list_tenants():
    """List all tenants"""
    return {
        "tenants": [
            {
                "tenant_id": tid,
                "name": t.name,
                "vlan_id": t.vlan_id,
                "subnet": t.subnet,
                "bandwidth_mbps": t.bandwidth_mbps,
                "status": t.status
            }
            for tid, t in controller.tenants.items()
        ],
        "total": len(controller.tenants)
    }
```

---

## 5. Billing Integration {#billing}

```python
# billing.py - Usage-based billing สำหรับ NaaS
from datetime import datetime, timedelta
from typing import Dict

class UsageBilling:
    """คำนวณค่าบริการตาม usage"""
    
    RATES = {
        "data_gb": 5.00,       # บาทต่อ GB
        "bandwidth_mbps": 50.00,  # บาทต่อ Mbps/เดือน
        "tenant_base": 500.00  # ค่า base ต่อเดือน
    }
    
    def calculate_monthly_bill(self, tenant_id: str, 
                                usage_data: dict) -> dict:
        """คำนวณค่าบริการเดือนนี้"""
        base_fee = self.RATES["tenant_base"]
        
        # Bandwidth fee
        bandwidth_mbps = usage_data.get("allocated_bandwidth_mbps", 0)
        bandwidth_fee = bandwidth_mbps * self.RATES["bandwidth_mbps"]
        
        # Data usage fee
        data_gb = usage_data.get("data_used_gb", 0)
        data_fee = data_gb * self.RATES["data_gb"]
        
        total = base_fee + bandwidth_fee + data_fee
        
        return {
            "tenant_id": tenant_id,
            "period": datetime.utcnow().strftime("%Y-%m"),
            "base_fee": base_fee,
            "bandwidth_fee": bandwidth_fee,
            "data_fee": data_fee,
            "total_thb": round(total, 2),
            "breakdown": {
                "bandwidth_mbps": bandwidth_mbps,
                "data_used_gb": round(data_gb, 2)
            }
        }
    
    def bytes_to_gb(self, bytes_count: int) -> float:
        return bytes_count / (1024 ** 3)
    
    def generate_invoice(self, tenant_id: str, bill: dict) -> str:
        """สร้าง invoice text"""
        lines = [
            f"INVOICE - {bill['period']}",
            f"Tenant: {tenant_id}",
            "=" * 40,
            f"Base Fee:          {bill['base_fee']:>10.2f} THB",
            f"Bandwidth ({bill['breakdown']['bandwidth_mbps']}Mbps): {bill['bandwidth_fee']:>10.2f} THB",
            f"Data ({bill['breakdown']['data_used_gb']:.1f}GB): {bill['data_fee']:>10.2f} THB",
            "=" * 40,
            f"TOTAL:             {bill['total_thb']:>10.2f} THB"
        ]
        return "\n".join(lines)
```

---

## 6. Lab: Multi-tenant Setup {#lab}

```bash
# Step 1: Provision tenants via API
curl -X POST http://localhost:8000/tenants \
  -H "Content-Type: application/json" \
  -d '{
    "tenant_id": "tenant-a",
    "name": "Company A",
    "vlan_id": 100,
    "subnet": "10.100.0.0/24",
    "gateway": "10.100.0.1",
    "bandwidth_mbps": 100,
    "public_ip": "203.0.113.10"
  }'

curl -X POST http://localhost:8000/tenants \
  -H "Content-Type: application/json" \
  -d '{
    "tenant_id": "tenant-b",
    "name": "Company B",
    "vlan_id": 200,
    "subnet": "10.200.0.0/24",
    "gateway": "10.200.0.1",
    "bandwidth_mbps": 200
  }'

# Step 2: Verify isolation
# เชื่อม client ไป VLAN 100 (tenant-a)
# ลอง ping 10.200.0.x - ต้อง fail
# ลอง ping 8.8.8.8 - ต้อง success

# Step 3: Check stats
curl http://localhost:8000/tenants/tenant-a/stats

# Step 4: List tenants
curl http://localhost:8000/tenants

# Step 5: Verify MikroTik config
# บน router:
# /interface vlan print
# /queue simple print
# /ip firewall nat print
```

### Verification Checklist

- [ ] VLANs สร้างแล้ว (100, 200)
- [ ] DHCP server ทำงานต่อ tenant
- [ ] Cross-tenant traffic blocked
- [ ] Bandwidth limits ทำงาน
- [ ] API endpoints ตอบสนอง
- [ ] Usage statistics แสดงถูกต้อง

> **Note:** VLAN ID ต้องไม่ซ้ำกัน และ switch trunk port ต้องรองรับ VLANs ทั้งหมด

---

## Summary

Part นี้ครอบคลุม:
- **Multi-tenancy** ด้วย VLAN + VRF
- **NaaS controller** สำหรับ automated provisioning
- **Tenant API** สำหรับ self-service
- **Billing** based on usage
- **Network isolation** ระหว่าง tenants

---

[← Part 97: Full-stack ISP Platform](part-097-fullstack-isp.md) | [Part 99: Global Orchestration →](part-099-global-orchestration.md)
