# Part 99: Global Network Orchestration

## สารบัญ
1. [SDN Concepts](#sdn)
2. [Centralized Control Plane](#control-plane)
3. [Intent-Based Networking](#ibn)
4. [Global Topology Management](#topology)
5. [Multi-site Orchestration](#multi-site)
6. [Lab: Network Orchestration](#lab)

---

## 1. SDN Concepts {#sdn}

### SDN Architecture Layers

```
┌─────────────────────────────────┐
│   APPLICATION LAYER             │
│  (Business Logic, Policies)     │
├─────────────────────────────────┤
│   CONTROL PLANE                 │
│  (SDN Controller: ONOS, ODL)    │
├─────────────────────────────────┤
│   DATA PLANE                    │
│  (MikroTik Routers / Switches)  │
└─────────────────────────────────┘
```

| Component | Role | Technology |
|-----------|------|-----------|
| Northbound API | Apps → Controller | REST API, gRPC |
| Controller | Centralized logic | Python, Go |
| Southbound API | Controller → Devices | NETCONF, API |
| Data Plane | Packet forwarding | MikroTik RouterOS |

---

## 2. Centralized Control Plane {#control-plane}

```python
# sdn_controller.py - Centralized SDN Controller
from dataclasses import dataclass, field
from typing import Dict, List, Optional, Set
import librouteros
import asyncio
import json
from datetime import datetime

@dataclass
class NetworkDevice:
    device_id: str
    name: str
    host: str
    username: str
    password: str
    device_type: str  # "router", "switch", "firewall"
    location: str
    capabilities: List[str] = field(default_factory=list)
    status: str = "unknown"

@dataclass  
class NetworkLink:
    link_id: str
    src_device: str
    src_port: str
    dst_device: str
    dst_port: str
    bandwidth_mbps: int
    latency_ms: float = 0.0
    status: str = "up"

@dataclass
class NetworkPolicy:
    policy_id: str
    name: str
    priority: int
    match: dict
    action: dict
    description: str = ""

class SDNController:
    """Centralized SDN Controller"""
    
    def __init__(self):
        self.devices: Dict[str, NetworkDevice] = {}
        self.links: Dict[str, NetworkLink] = {}
        self.policies: Dict[str, NetworkPolicy] = {}
        self.topology_graph: Dict[str, Set[str]] = {}  # adjacency list
    
    def register_device(self, device: NetworkDevice) -> bool:
        """ลงทะเบียน network device"""
        try:
            conn = librouteros.connect(
                host=device.host,
                username=device.username,
                password=device.password,
                timeout=5
            )
            
            resource = dict(list(conn('/system/resource/print'))[0])
            identity = dict(list(conn('/system/identity/print'))[0])
            
            device.name = identity.get("name", device.name)
            device.status = "online"
            device.capabilities = self._detect_capabilities(conn)
            
            conn.close()
            
            self.devices[device.device_id] = device
            self.topology_graph[device.device_id] = set()
            
            return True
        
        except Exception as e:
            device.status = "offline"
            self.devices[device.device_id] = device
            return False
    
    def _detect_capabilities(self, conn) -> List[str]:
        """ตรวจสอบ capabilities ของ device"""
        caps = ["routing", "firewall", "nat"]
        
        # Check for BGP
        try:
            list(conn('/routing/bgp/instance/print'))
            caps.append("bgp")
        except:
            pass
        
        # Check for OSPF
        try:
            list(conn('/routing/ospf/instance/print'))
            caps.append("ospf")
        except:
            pass
        
        # Check for MPLS
        try:
            list(conn('/mpls/print'))
            caps.append("mpls")
        except:
            pass
        
        return caps
    
    def add_link(self, link: NetworkLink):
        """เพิ่ม link ระหว่าง devices"""
        self.links[link.link_id] = link
        
        # Update topology graph
        if link.src_device in self.topology_graph:
            self.topology_graph[link.src_device].add(link.dst_device)
        if link.dst_device in self.topology_graph:
            self.topology_graph[link.dst_device].add(link.src_device)
    
    def apply_policy(self, policy: NetworkPolicy, 
                      target_devices: List[str]) -> dict:
        """Apply policy ไปยัง devices"""
        results = {}
        
        for device_id in target_devices:
            device = self.devices.get(device_id)
            if not device or device.status != "online":
                results[device_id] = {"status": "skipped", "reason": "offline"}
                continue
            
            try:
                conn = librouteros.connect(
                    host=device.host,
                    username=device.username,
                    password=device.password
                )
                
                # Apply based on policy action type
                action_type = policy.action.get("type")
                
                if action_type == "firewall_rule":
                    self._apply_firewall_policy(conn, policy)
                elif action_type == "routing_policy":
                    self._apply_routing_policy(conn, policy)
                elif action_type == "qos_policy":
                    self._apply_qos_policy(conn, policy)
                
                conn.close()
                results[device_id] = {"status": "applied"}
                
            except Exception as e:
                results[device_id] = {"status": "failed", "error": str(e)}
        
        self.policies[policy.policy_id] = policy
        return results
    
    def _apply_firewall_policy(self, conn, policy: NetworkPolicy):
        """Apply firewall policy"""
        action = policy.action
        match = policy.match
        
        rule_params = {
            "chain": match.get("chain", "forward"),
            "action": action.get("action", "drop"),
            "comment": f"SDN-Policy: {policy.name}"
        }
        
        if "src_address" in match:
            rule_params["src-address"] = match["src_address"]
        if "dst_address" in match:
            rule_params["dst-address"] = match["dst_address"]
        if "protocol" in match:
            rule_params["protocol"] = match["protocol"]
        if "dst_port" in match:
            rule_params["dst-port"] = str(match["dst_port"])
        
        conn('/ip/firewall/filter/add', **rule_params)
    
    def _apply_routing_policy(self, conn, policy: NetworkPolicy):
        """Apply routing policy"""
        action = policy.action
        
        if action.get("action") == "add_route":
            conn('/ip/route/add', **{
                "dst-address": action["dst_address"],
                "gateway": action["gateway"],
                "comment": f"SDN: {policy.name}"
            })
    
    def _apply_qos_policy(self, conn, policy: NetworkPolicy):
        """Apply QoS policy"""
        action = policy.action
        match = policy.match
        
        conn('/queue/simple/add', **{
            "name": f"sdn-{policy.policy_id}",
            "target": match.get("target", "0.0.0.0/0"),
            "max-limit": f"{action.get('bandwidth_mbps', 100)}M/{action.get('bandwidth_mbps', 100)}M",
            "comment": f"SDN-QoS: {policy.name}"
        })
    
    def find_path(self, src_device: str, dst_device: str) -> List[str]:
        """BFS path finding ระหว่าง devices"""
        if src_device not in self.topology_graph:
            return []
        
        visited = {src_device}
        queue = [[src_device]]
        
        while queue:
            path = queue.pop(0)
            current = path[-1]
            
            if current == dst_device:
                return path
            
            for neighbor in self.topology_graph.get(current, set()):
                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append(path + [neighbor])
        
        return []
    
    def get_topology(self) -> dict:
        """ส่ง topology ทั้งหมด"""
        return {
            "devices": {
                did: {
                    "name": d.name,
                    "host": d.host,
                    "type": d.device_type,
                    "location": d.location,
                    "status": d.status,
                    "capabilities": d.capabilities
                }
                for did, d in self.devices.items()
            },
            "links": {
                lid: {
                    "src": f"{l.src_device}/{l.src_port}",
                    "dst": f"{l.dst_device}/{l.dst_port}",
                    "bandwidth_mbps": l.bandwidth_mbps,
                    "status": l.status
                }
                for lid, l in self.links.items()
            },
            "policies_count": len(self.policies)
        }
    
    async def monitor_all_devices(self, interval: int = 30):
        """Monitor all registered devices"""
        print("Starting global monitoring...")
        
        while True:
            for device_id, device in self.devices.items():
                try:
                    conn = librouteros.connect(
                        host=device.host,
                        username=device.username,
                        password=device.password,
                        timeout=3
                    )
                    conn.close()
                    
                    if device.status != "online":
                        device.status = "online"
                        print(f"Device {device.name} came online")
                
                except:
                    if device.status == "online":
                        device.status = "offline"
                        print(f"ALERT: Device {device.name} went offline!")
            
            await asyncio.sleep(interval)
```

---

## 3. Intent-Based Networking {#ibn}

```python
# intent_engine.py - Intent-based networking
from typing import Dict, Any

class IntentEngine:
    """Convert high-level intents to network policies"""
    
    def __init__(self, controller: SDNController):
        self.controller = controller
    
    def translate_intent(self, intent: dict) -> List[NetworkPolicy]:
        """แปลง intent เป็น network policies"""
        intent_type = intent.get("type")
        policies = []
        
        if intent_type == "isolate_tenant":
            policies = self._intent_isolate_tenant(intent)
        elif intent_type == "prioritize_traffic":
            policies = self._intent_prioritize_traffic(intent)
        elif intent_type == "block_access":
            policies = self._intent_block_access(intent)
        elif intent_type == "ensure_connectivity":
            policies = self._intent_ensure_connectivity(intent)
        
        return policies
    
    def _intent_isolate_tenant(self, intent: dict) -> List[NetworkPolicy]:
        """Intent: isolate tenant traffic"""
        tenant_subnet = intent["tenant_subnet"]
        other_subnets = intent.get("other_subnets", [])
        
        policies = []
        for i, other in enumerate(other_subnets):
            policies.append(NetworkPolicy(
                policy_id=f"isolate-{intent['tenant_id']}-{i}",
                name=f"Isolate {intent['tenant_id']} from {other}",
                priority=100,
                match={
                    "chain": "forward",
                    "src_address": tenant_subnet,
                    "dst_address": other
                },
                action={"type": "firewall_rule", "action": "drop"},
                description=f"Auto-generated isolation policy"
            ))
        
        return policies
    
    def _intent_prioritize_traffic(self, intent: dict) -> List[NetworkPolicy]:
        """Intent: prioritize specific traffic"""
        return [NetworkPolicy(
            policy_id=f"priority-{intent.get('traffic_type', 'default')}",
            name=f"Prioritize {intent.get('traffic_type')} traffic",
            priority=200,
            match={
                "target": intent.get("src_subnet", "0.0.0.0/0"),
                "dst_port": intent.get("dst_port")
            },
            action={
                "type": "qos_policy",
                "bandwidth_mbps": intent.get("guaranteed_mbps", 10)
            }
        )]
    
    def _intent_block_access(self, intent: dict) -> List[NetworkPolicy]:
        """Intent: block access to destination"""
        return [NetworkPolicy(
            policy_id=f"block-{intent.get('src', 'all')}-to-{intent.get('dst', 'any')}",
            name=f"Block access",
            priority=50,
            match={
                "chain": "forward",
                "src_address": intent.get("src_subnet"),
                "dst_address": intent.get("dst_address")
            },
            action={"type": "firewall_rule", "action": "drop"}
        )]
    
    def apply_intent(self, intent: dict, target_devices: List[str]) -> dict:
        """Apply intent ไปยัง network"""
        policies = self.translate_intent(intent)
        
        results = {}
        for policy in policies:
            policy_result = self.controller.apply_policy(policy, target_devices)
            results[policy.policy_id] = {
                "policy_name": policy.name,
                "devices": policy_result
            }
        
        return {
            "intent_id": intent.get("id"),
            "policies_created": len(policies),
            "results": results
        }
```

---

## 4. Global Topology Management {#topology}

```python
# topology_manager.py - Global topology visualization
import json

class TopologyManager:
    """จัดการและ visualize network topology"""
    
    def __init__(self, controller: SDNController):
        self.controller = controller
    
    def export_topology_json(self) -> str:
        """Export topology เป็น JSON"""
        topology = self.controller.get_topology()
        
        # Convert to vis.js format
        nodes = []
        edges = []
        
        for device_id, device in topology["devices"].items():
            color = {
                "online": "#10b981",  # green
                "offline": "#ef4444",  # red
                "unknown": "#9ca3af"  # gray
            }.get(device["status"], "#9ca3af")
            
            nodes.append({
                "id": device_id,
                "label": device["name"],
                "title": f"{device['type']} @ {device['location']}",
                "color": {"background": color}
            })
        
        for link_id, link in topology["links"].items():
            src = link["src"].split("/")[0]
            dst = link["dst"].split("/")[0]
            
            edges.append({
                "id": link_id,
                "from": src,
                "to": dst,
                "label": f"{link['bandwidth_mbps']}Mbps",
                "color": {"color": "#1d4ed8"}
            })
        
        return json.dumps({
            "nodes": nodes,
            "edges": edges,
            "stats": {
                "total_devices": len(nodes),
                "total_links": len(edges),
                "online": sum(1 for n in nodes if "#10b981" in str(n.get("color")))
            }
        }, indent=2)
    
    def find_redundant_paths(self, src: str, dst: str) -> List[List[str]]:
        """หา redundant paths ระหว่าง nodes"""
        all_paths = []
        visited = set()
        
        def dfs(current, target, path):
            if current == target:
                all_paths.append(path[:])
                return
            
            for neighbor in self.controller.topology_graph.get(current, set()):
                if neighbor not in visited:
                    visited.add(neighbor)
                    dfs(neighbor, target, path + [neighbor])
                    visited.discard(neighbor)
        
        visited.add(src)
        dfs(src, dst, [src])
        
        return all_paths
```

---

## 5. Multi-site Orchestration {#multi-site}

```python
# multi_site.py - Orchestrate multiple sites
from concurrent.futures import ThreadPoolExecutor
from typing import Callable

class MultiSiteOrchestrator:
    """Orchestrate สำหรับ multi-site network"""
    
    def __init__(self, sites: Dict[str, SDNController]):
        self.sites = sites  # {site_name: SDNController}
    
    def apply_global_policy(self, policy: NetworkPolicy, 
                             site_names: List[str] = None) -> dict:
        """Apply policy ไปยังทุก sites"""
        if site_names is None:
            site_names = list(self.sites.keys())
        
        results = {}
        
        with ThreadPoolExecutor(max_workers=10) as executor:
            futures = {}
            
            for site in site_names:
                controller = self.sites.get(site)
                if controller:
                    all_devices = list(controller.devices.keys())
                    future = executor.submit(
                        controller.apply_policy, policy, all_devices
                    )
                    futures[site] = future
            
            for site, future in futures.items():
                try:
                    results[site] = future.result(timeout=30)
                except Exception as e:
                    results[site] = {"error": str(e)}
        
        return results
    
    def get_global_status(self) -> dict:
        """ดึงสถานะทุก sites"""
        global_status = {}
        
        for site_name, controller in self.sites.items():
            online = sum(1 for d in controller.devices.values() 
                        if d.status == "online")
            offline = sum(1 for d in controller.devices.values() 
                         if d.status == "offline")
            
            global_status[site_name] = {
                "total_devices": len(controller.devices),
                "online": online,
                "offline": offline,
                "health_pct": round((online / max(len(controller.devices), 1)) * 100, 1)
            }
        
        return {
            "sites": global_status,
            "total_sites": len(self.sites),
            "healthy_sites": sum(1 for s in global_status.values() if s["health_pct"] == 100)
        }
```

---

## 6. Lab: Network Orchestration {#lab}

```python
# lab_setup.py - Setup lab SDN environment
import asyncio

async def main():
    # Create controller
    controller = SDNController()
    
    # Register devices
    devices = [
        NetworkDevice("core-01", "Core-01", "10.0.0.1", "admin", "pass", "router", "Bangkok"),
        NetworkDevice("edge-01", "Edge-01", "10.0.1.1", "admin", "pass", "router", "Bangkok"),
        NetworkDevice("edge-02", "Edge-02", "10.0.1.2", "admin", "pass", "router", "Chiang Mai"),
    ]
    
    for device in devices:
        success = controller.register_device(device)
        print(f"Register {device.name}: {'OK' if success else 'FAILED'}")
    
    # Add links
    controller.add_link(NetworkLink("link-1", "core-01", "ether1", "edge-01", "ether1", 1000))
    controller.add_link(NetworkLink("link-2", "core-01", "ether2", "edge-02", "ether1", 1000))
    
    # Create intent engine
    intent_engine = IntentEngine(controller)
    
    # Apply intent: block access from office to servers
    intent = {
        "id": "intent-001",
        "type": "block_access",
        "src_subnet": "10.100.0.0/24",
        "dst_address": "192.168.200.0/24"
    }
    
    result = intent_engine.apply_intent(intent, ["edge-01"])
    print(f"Intent applied: {result}")
    
    # Get topology
    topology = controller.get_topology()
    print(f"Devices: {len(topology['devices'])}, Links: {len(topology['links'])}")
    
    # Start monitoring
    await controller.monitor_all_devices(interval=60)

if __name__ == "__main__":
    asyncio.run(main())
```

```bash
# ทดสอบ orchestration
python3 lab_setup.py &

# ตรวจสอบผล
curl http://localhost:8000/topology
curl http://localhost:8000/devices
curl http://localhost:8000/policies
```

### Verification Checklist

- [ ] Controller รับ device registrations
- [ ] Topology graph สร้างถูกต้อง
- [ ] Path finding ทำงาน
- [ ] Policies apply ไปยัง devices
- [ ] Intent translation ทำงาน
- [ ] Multi-site status แสดงถูกต้อง

> **Tip:** ใน production ใช้ NETCONF/YANG แทน proprietary API สำหรับ vendor-neutral southbound

---

## Summary

Part นี้ครอบคลุม:
- **SDN architecture** - control/data plane separation
- **Centralized controller** พร้อม device management
- **Intent-based networking** - high-level policies
- **Topology management** - graph-based path finding
- **Multi-site orchestration** - global control

---

[← Part 98: Multi-tenant](part-098-multi-tenant.md) | [Part 100: Capstone Project →](part-100-capstone-project.md)
