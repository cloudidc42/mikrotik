# Part 74: MPLS และ VPN

## สารบัญ
1. [MPLS Fundamentals](#fundamentals)
2. [LDP - Label Distribution Protocol](#ldp)
3. [MPLS Traffic Engineering](#mpls-te)
4. [L2VPN - Layer 2 VPN](#l2vpn)
5. [L3VPN - Layer 3 VPN](#l3vpn)
6. [MPLS QoS](#qos)
7. [MPLS Monitoring](#monitoring)
8. [Lab: MPLS L3VPN Setup](#lab)

---

## 1. MPLS Fundamentals {#fundamentals}

MPLS (Multiprotocol Label Switching) เป็น technology สำหรับ high-performance packet forwarding ที่ใช้ labels แทน IP addresses

### MPLS Architecture

```
CE1 ──── PE1 ──── P1 ──── P2 ──── PE2 ──── CE2
  Customer   Provider   Provider   Provider   Customer
  Edge       Edge       Core       Core       Edge

CE = Customer Edge Router
PE = Provider Edge Router  
P  = Provider Core Router
```

### Label Format

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                Label                  | Exp |S|       TTL     |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+

Label: 20 bits (0-1048575)
Exp:   3 bits (QoS/CoS)
S:     1 bit (Bottom of Stack)
TTL:   8 bits (Time to Live)
```

### MPLS Operations

| Operation | Description |
|-----------|-------------|
| Push | เพิ่ม label ลงใน packet (Ingress PE) |
| Swap | เปลี่ยน label ด้วย label ใหม่ (P router) |
| Pop | ลบ label ออก (Egress PE หรือ penultimate) |

---

## 2. LDP - Label Distribution Protocol {#ldp}

```bash
# ============================================
# LDP Configuration บน MikroTik
# ============================================

# Enable MPLS (RouterOS 7.x)
/mpls
set enabled=yes

# LDP Configuration
/mpls ldp
set enabled=yes lsr-id=10.0.0.1 transport-address=10.0.0.1

# LDP Interface
/mpls ldp interface
add interface=ether1 comment="To P1"
add interface=ether2 comment="To P2"

# ตรวจสอบ LDP neighbors
/mpls ldp neighbor print
# Expected: เห็น neighbors หลังจาก LDP sessions established

# ตรวจสอบ MPLS forwarding table
/mpls forwarding-table print
# Expected: เห็น label bindings สำหรับ prefixes ต่างๆ

# ตรวจสอบ LDP bindings
/mpls ldp local-binding print
/mpls ldp remote-binding print
```

---

## 3. MPLS Traffic Engineering {#mpls-te}

```bash
# ============================================
# MPLS-TE Configuration
# ============================================

# Enable RSVP-TE
/mpls traffic-eng
set enabled=yes

# สร้าง TE tunnel
/interface traffic-eng
add name=te-tunnel1 \
    to-address=10.0.0.4 \
    bandwidth=100M \
    primary-path=path1 \
    record-route=yes \
    comment="TE Tunnel to PE2"

# กำหนด explicit path
/mpls traffic-eng path
add name=path1 \
    hops=10.0.1.1,10.0.2.1,10.0.0.4

# ใช้ TE tunnel สำหรับ routing
/ip route
add dst-address=10.10.0.0/24 gateway=te-tunnel1

# ตรวจสอบ TE tunnels
/interface traffic-eng print detail
# Expected: เห็น tunnel state=operational
```

---

## 4. L2VPN - Layer 2 VPN {#l2vpn}

```bash
# ============================================
# VPLS (Virtual Private LAN Service) 
# ============================================

# Pseudowire configuration (Layer 2 VPN)
# PE1 Configuration
/interface vpls
add name=vpls-cust1 \
    remote-peer=10.0.0.4 \
    vpls-id=1:100 \
    pw-type=ethernet \
    comment="VPLS to PE2 for Customer 1"

# Bridge VPLS with CE interface
/interface bridge
add name=bridge-cust1

/interface bridge port
add bridge=bridge-cust1 interface=ether3 comment="CE1 port"
add bridge=bridge-cust1 interface=vpls-cust1 comment="VPLS tunnel"

# PE2 Configuration  
/interface vpls
add name=vpls-cust1 \
    remote-peer=10.0.0.1 \
    vpls-id=1:100 \
    pw-type=ethernet

/interface bridge
add name=bridge-cust1

/interface bridge port
add bridge=bridge-cust1 interface=ether3 comment="CE2 port"
add bridge=bridge-cust1 interface=vpls-cust1 comment="VPLS tunnel"

# ตรวจสอบ VPLS
/interface vpls print detail
```

---

## 5. L3VPN - Layer 3 VPN {#l3vpn}

```bash
# ============================================
# BGP/MPLS L3VPN Configuration
# ============================================

# PE1 Configuration

# VRF สำหรับ customer
/ip vrf
add name=cust1-vrf \
    interfaces=ether3 \
    import-route-target=65000:100 \
    export-route-target=65000:100 \
    route-distinguisher=65000:100

# BGP configuration บน PE
/routing bgp instance
add name=default as=65000 router-id=10.0.0.1

# MP-BGP สำหรับ VPNv4
/routing bgp peer
add name=pe2 remote-address=10.0.0.4 remote-as=65000 \
    address-families=vpnv4 update-source=lo0 \
    comment="iBGP to PE2 for VPNv4"

# IP addressing สำหรับ VRF
/ip address
add address=192.168.100.1/30 interface=ether3 comment="CE1 link"

# Static route ใน VRF
/ip route vrf
add vrf=cust1-vrf dst-address=10.10.0.0/24 gateway=192.168.100.2

# PE2 Configuration
/ip vrf
add name=cust1-vrf \
    interfaces=ether3 \
    import-route-target=65000:100 \
    export-route-target=65000:100 \
    route-distinguisher=65000:101

/ip address
add address=192.168.100.5/30 interface=ether3 comment="CE2 link"

/ip route vrf
add vrf=cust1-vrf dst-address=10.20.0.0/24 gateway=192.168.100.6

# ตรวจสอบ L3VPN
/routing bgp vpnv4-route print
/ip route vrf print
```

---

## 6. MPLS QoS {#qos}

```bash
# ============================================
# MPLS DiffServ Tunneling
# ============================================

# Mangle สำหรับ MPLS EXP bits
/ip firewall mangle
# Copy DSCP to MPLS EXP
add chain=prerouting action=change-dscp new-dscp=46 \
    in-interface=ether3 src-address-list=voip-phones \
    comment="EF (Expedited Forwarding) for VoIP"

add chain=prerouting action=change-dscp new-dscp=34 \
    in-interface=ether3 dst-address-list=video-servers \
    comment="AF41 for Video"

# QoS Queue Tree บน MPLS interfaces
/queue type
add name=mpls-pq kind=pfifo pfifo-limit=50

/queue tree
add name=mpls-root parent=ether1 max-limit=1G

add name=ef-traffic parent=mpls-root \
    packet-mark=voip priority=1 limit-at=100M

add name=af41-traffic parent=mpls-root \
    packet-mark=video priority=2 limit-at=200M

add name=be-traffic parent=mpls-root \
    packet-mark=best-effort priority=8 limit-at=700M
```

---

## 7. MPLS Monitoring {#monitoring}

```bash
# ============================================
# MPLS Monitoring Commands
# ============================================

# LDP session status
/mpls ldp session print

# MPLS forwarding table
/mpls forwarding-table print

# MPLS interface stats
/mpls interface print stats

# LDP bindings
/mpls ldp local-binding print
/mpls ldp remote-binding print

# VPLS status
/interface vpls print detail

# BGP VPNv4 routes
/routing bgp vpnv4-route print

# IP routes in VRF
/ip route vrf print where vrf=cust1-vrf

# Traffic stats บน TE tunnel
/interface traffic-eng print stats
```

```python
# Python MPLS monitoring script
import librouteros
from datetime import datetime

def monitor_mpls(host: str, username: str, password: str):
    conn = librouteros.connect(host=host, username=username, password=password)
    
    # LDP sessions
    ldp_sessions = list(conn('/mpls/ldp/session/print'))
    print(f"LDP Sessions: {len(ldp_sessions)}")
    for session in ldp_sessions:
        s = dict(session)
        print(f"  Peer: {s.get('remote-address')} State: {s.get('state')}")
    
    # MPLS forwarding table size
    fwd_table = list(conn('/mpls/forwarding-table/print'))
    print(f"MPLS FIB entries: {len(fwd_table)}")
    
    # VPLS interfaces
    vpls_ifaces = list(conn('/interface/vpls/print'))
    for vpls in vpls_ifaces:
        v = dict(vpls)
        print(f"  VPLS: {v.get('name')} Remote: {v.get('remote-peer')} Running: {v.get('running')}")
    
    conn.close()

if __name__ == "__main__":
    monitor_mpls("10.0.0.1", "admin", "password")
```

---

## 8. Lab: MPLS L3VPN Setup {#lab}

### Lab Topology

```
CE1 (192.168.100.2)
    │
    │ ether3
PE1 (10.0.0.1) ──LDP/OSPF── P1 (10.0.0.2) ──LDP/OSPF── P2 (10.0.0.3) ──LDP/OSPF── PE2 (10.0.0.4)
                                                                                          │ ether3
                                                                               CE2 (192.168.100.6)

VRF: cust1-vrf (RT: 65000:100)
CE1 network: 10.10.0.0/24
CE2 network: 10.20.0.0/24
```

### Lab Steps

```bash
# Step 1: OSPF + MPLS บน P routers
/routing ospf instance add name=default router-id=10.0.0.X
/mpls set enabled=yes
/mpls ldp set enabled=yes lsr-id=10.0.0.X
/mpls ldp interface add interface=ether1,ether2

# Step 2: MP-BGP บน PE routers
/routing bgp instance add name=default as=65000 router-id=10.0.0.X
/routing bgp peer add name=pe-peer remote-address=10.0.0.Y remote-as=65000 address-families=vpnv4

# Step 3: VRF setup
/ip vrf add name=cust1 interfaces=ether3 route-distinguisher=65000:100 import-route-target=65000:100 export-route-target=65000:100

# Step 4: Verify
/routing bgp vpnv4-route print
/ip route vrf print where vrf=cust1
```

### Verification Checklist

- [ ] LDP sessions established ระหว่าง P routers
- [ ] MPLS labels ถูก assign สำหรับ loopback prefixes
- [ ] MP-BGP VPNv4 sessions ระหว่าง PEs
- [ ] VRF routes ปรากฏทั้งสอง PE
- [ ] CE1 ping CE2 ได้ผ่าน MPLS backbone

> **Note:** MPLS ต้องการ OSPF หรือ IS-IS สำหรับ underlay routing ก่อน

> **Warning:** MikroTik MPLS support มีข้อจำกัดใน CHR version บางรุ่น

---

## Summary

Part นี้ครอบคลุม:
- **MPLS fundamentals** - labels, FEC, LSP
- **LDP** - label distribution protocol
- **MPLS-TE** - traffic engineering with RSVP
- **L2VPN/VPLS** - Layer 2 connectivity
- **L3VPN** - BGP/MPLS VPN
- **MPLS QoS** - DiffServ over MPLS

---

[← Part 73: VRRP](part-073-vrrp.md) | [Part 75: BGP Internet →](part-075-bgp-internet.md)
