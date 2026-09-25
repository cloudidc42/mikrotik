# Part 75: BGP สำหรับ ISP

## สารบัญ
1. [BGP Fundamentals](#fundamentals)
2. [iBGP vs eBGP](#ibgp-ebgp)
3. [BGP Communities](#communities)
4. [RPKI Validation](#rpki)
5. [Multi-homing](#multihoming)
6. [BGP Policies](#policies)
7. [BGP Security](#security)
8. [BGP Monitoring](#monitoring)
9. [Lab: ISP BGP Setup](#lab)

---

## 1. BGP Fundamentals {#fundamentals}

BGP (Border Gateway Protocol) เป็น routing protocol หลักของ internet ใช้ AS numbers สำหรับ identify organizations

### BGP Attributes

| Attribute | Type | Description |
|-----------|------|-------------|
| AS_PATH | Well-known | รายการ AS ที่ route ผ่านมา |
| NEXT_HOP | Well-known | IP ของ next hop router |
| LOCAL_PREF | Well-known | preference ภายใน AS |
| MED | Optional | แนะนำ entry point ให้ neighbor |
| COMMUNITY | Optional | Tag สำหรับ policy |
| ORIGIN | Well-known | IGP, EGP, Incomplete |

---

## 2. BGP Configuration {#ibgp-ebgp}

```bash
# ============================================
# eBGP Configuration (ISP to ISP)
# ============================================

# สมมติ: AS65000 = เรา, AS65001 = ISP upstream

/routing bgp instance
add name=default as=65000 router-id=203.0.113.1 \
    comment="Our ISP AS"

# eBGP peer กับ upstream ISP
/routing bgp peer
add name=upstream-isp1 remote-address=203.0.113.254 \
    remote-as=65001 \
    address-families=ip \
    input.filter=bgp-in-from-isp1 \
    output.filter=bgp-out-to-isp1 \
    comment="Upstream ISP1"

# Filter: รับ routes จาก ISP1
/routing filter rule
add chain=bgp-in-from-isp1 rule="if (bgp-as-path-length < 5) { accept; } else { reject; }"

# Filter: ส่ง routes ไป ISP1 (advertise เฉพาะ our prefixes)
add chain=bgp-out-to-isp1 rule="if (dst in 203.0.113.0/24) { accept; } else { reject; }"


# ============================================
# iBGP Configuration (within AS)
# ============================================

# Route Reflector สำหรับ large iBGP mesh
/routing bgp peer
add name=rr-client1 remote-address=10.0.0.2 \
    remote-as=65000 \
    route-reflect=yes \
    comment="iBGP to PE1 (RR client)"

add name=rr-client2 remote-address=10.0.0.3 \
    remote-as=65000 \
    route-reflect=yes \
    comment="iBGP to PE2 (RR client)"

# Next-hop-self สำหรับ iBGP
/routing bgp peer
set rr-client1 nexthop-choice=force-self
set rr-client2 nexthop-choice=force-self
```

---

## 3. BGP Communities {#communities}

```bash
# ============================================
# BGP Community สำหรับ Traffic Engineering
# ============================================

# Standard communities
# 65000:100 = local routes
# 65000:200 = customer routes
# 65000:300 = peering routes
# 65000:400 = transit routes

# Tagging outbound routes ด้วย community
/routing filter rule
# Customer routes
add chain=bgp-in-from-customers rule="set bgp-communities 65000:200; accept;"

# Peering routes
add chain=bgp-in-from-peers rule="set bgp-communities 65000:300; accept;"

# ใช้ community เพื่อ control advertisement
add chain=bgp-export rule="if (bgp-communities has 65000:100) { accept; } else { reject; }"

# Well-known communities
# NO_EXPORT (65535:65281) = ไม่ export ไปนอก AS
# NO_ADVERTISE (65535:65282) = ไม่ advertise ไปใคร

/routing filter rule
add chain=bgp-out rule="if (bgp-communities has 65535:65281) { reject; }"


# Large Communities (RFC 8092)
# Format: ASN:function:value
# 65000:1:1 = blackhole
# 65000:2:100 = prepend 1x to AS100

/routing filter rule
add chain=bgp-in rule="
    if (bgp-large-communities has 65000:1:1) {
        set blackhole yes;
        accept;
    }
"
```

---

## 4. RPKI Validation {#rpki}

```bash
# ============================================
# RPKI Route Origin Validation
# ============================================

# Configure RPKI validator (RTR protocol)
/routing rpki
add name=rpki-validator \
    address=rpki.example.com \
    port=8282 \
    comment="RPKI RTR validator"

# ตรวจสอบ RPKI connection
/routing rpki print
/routing rpki session print

# BGP filter ด้วย RPKI
/routing filter rule
# Accept valid routes
add chain=bgp-in-rpki rule="if (rpki-state valid) { accept; }"

# Mark invalid routes (ไม่รับ)
add chain=bgp-in-rpki rule="if (rpki-state invalid) { reject; }"

# Allow unknown (no ROA) with lower preference
add chain=bgp-in-rpki rule="if (rpki-state unknown) { set bgp-local-pref 80; accept; }"


# ROA (Route Origin Authorization) Check
/routing bgp peer
set upstream-isp1 input.filter=bgp-in-rpki
```

---

## 5. Multi-homing {#multihoming}

```bash
# ============================================
# Multi-homed ISP Configuration
# ============================================

# เชื่อมต่อ 2 upstream ISPs
/routing bgp peer
add name=isp1 remote-address=203.0.113.254 remote-as=65001 \
    input.filter=in-isp1 output.filter=out-isp1

add name=isp2 remote-address=198.51.100.254 remote-as=65002 \
    input.filter=in-isp2 output.filter=out-isp2

# Prefer ISP1 สำหรับ outbound traffic (higher LOCAL_PREF)
/routing filter rule
add chain=in-isp1 rule="set bgp-local-pref 200; accept;"
add chain=in-isp2 rule="set bgp-local-pref 100; accept;"

# Prefer ISP1 สำหรับ inbound (ส่ง AS-PATH prepend ไป ISP2)
add chain=out-isp2 rule="prepend bgp-path-as 3; accept;"

# Selective advertising: บาง prefixes ออก ISP2
/ip prefix-list
add name=via-isp2-only prefix=203.0.114.0/24

/routing filter rule
add chain=out-isp1 rule="if (dst in [prefix-list via-isp2-only]) { reject; } else { accept; }"
add chain=out-isp2 rule="if (dst in [prefix-list via-isp2-only]) { accept; } else { reject; }"


# BFD สำหรับ fast detection
/routing bgp peer
set isp1 bfd-session=yes
set isp2 bfd-session=yes

/routing bfd configuration
add interfaces=all min-rx=300ms min-tx=300ms multiplier=3
```

---

## 6. BGP Policies {#policies}

```bash
# ============================================
# BGP Policy สำหรับ ISP
# ============================================

# Maximum prefix protection
/routing bgp peer
set isp1 input.limit-addresses=700000 \
         input.limit-action=block \
         comment="Protect against BGP prefix attacks"

# Soft reconfiguration
/routing bgp peer
set isp1 input.keep-sent-attributes=yes

# BGP Graceful Restart
/routing bgp instance
set default graceful-restart=yes

# Route flap damping
/routing bgp instance
set default routing-table=main

# AS-PATH filtering (BOGON AS)
/routing filter rule
add chain=bgp-in rule="
    if (bgp-as-path contains 64512-65534) { 
        reject; 
    }
"
# Private AS ranges: 64512-65534, 4200000000-4294967294

# Prefix filtering (BOGON prefixes)
/ip prefix-list
add name=bogon-v4 prefix=0.0.0.0/8
add name=bogon-v4 prefix=10.0.0.0/8
add name=bogon-v4 prefix=100.64.0.0/10
add name=bogon-v4 prefix=127.0.0.0/8
add name=bogon-v4 prefix=169.254.0.0/16
add name=bogon-v4 prefix=172.16.0.0/12
add name=bogon-v4 prefix=192.0.0.0/24
add name=bogon-v4 prefix=192.168.0.0/16
add name=bogon-v4 prefix=198.18.0.0/15
add name=bogon-v4 prefix=198.51.100.0/24
add name=bogon-v4 prefix=203.0.113.0/24
add name=bogon-v4 prefix=240.0.0.0/4

/routing filter rule
add chain=bgp-in rule="if (dst in [prefix-list bogon-v4]) { reject; }"
```

---

## 7. BGP Monitoring {#monitoring}

```python
# BGP monitoring script
import librouteros
from datetime import datetime
import json

def check_bgp_status(host: str, username: str, password: str) -> dict:
    conn = librouteros.connect(host=host, username=username, password=password)
    
    # BGP peers
    peers = list(conn('/routing/bgp/peer/print'))
    peer_summary = []
    
    for peer in peers:
        p = dict(peer)
        peer_summary.append({
            "name": p.get("name"),
            "remote_as": p.get("remote-as"),
            "state": p.get("state"),
            "uptime": p.get("uptime"),
            "prefixes_received": p.get("prefix-count", 0),
            "established": p.get("established", False)
        })
    
    # Route table size
    routes = list(conn('/ip/route/print'))
    bgp_routes = [r for r in routes if dict(r).get("routing-mark") == "" and 
                  dict(r).get("src-address") != ""]
    
    result = {
        "timestamp": datetime.utcnow().isoformat(),
        "router": host,
        "total_routes": len(routes),
        "peers": peer_summary
    }
    
    # Alert on peer down
    for peer in peer_summary:
        if peer["state"] != "established":
            result.setdefault("alerts", []).append(
                f"BGP peer {peer['name']} (AS{peer['remote_as']}) is {peer['state']}"
            )
    
    conn.close()
    return result


def monitor_bgp_loop(routers: list):
    import time
    while True:
        for router in routers:
            status = check_bgp_status(**router)
            if "alerts" in status:
                for alert in status["alerts"]:
                    print(f"ALERT: {alert}")
            else:
                print(f"OK: {router['host']} - {len(status['peers'])} BGP peers up")
        time.sleep(60)

if __name__ == "__main__":
    routers = [
        {"host": "10.0.0.1", "username": "admin", "password": "pass"},
    ]
    monitor_bgp_loop(routers)
```

---

## 8. Lab: ISP BGP Setup {#lab}

### Lab Topology

```
        [Internet/Upstream ISPs]
              │           │
        AS65001         AS65002
          │                 │
    203.0.113.254    198.51.100.254
          │                 │
         iBGP         iBGP
          │                 │
      [Our Router - AS65000]
       router-id=203.0.113.1
       Our Prefixes: 203.0.113.0/24
```

### Lab Steps

```bash
# Step 1: Configure BGP instance
/routing bgp instance add name=default as=65000 router-id=203.0.113.1

# Step 2: eBGP peers
/routing bgp peer add name=isp1 remote-address=203.0.113.254 remote-as=65001
/routing bgp peer add name=isp2 remote-address=198.51.100.254 remote-as=65002

# Step 3: Advertise our prefixes
/routing bgp network add network=203.0.113.0/24

# Step 4: Import filters (RPKI + BOGON)
/routing filter rule add chain=bgp-in rule="if (dst in [prefix-list bogon-v4]) { reject; }"
/routing filter rule add chain=bgp-in rule="accept;"

# Step 5: Verify
/routing bgp peer print
/routing bgp advertisements print peer=isp1
/ip route print where bgp=yes
```

### Verification Checklist

- [ ] eBGP sessions established กับ ISPs
- [ ] รับ full routing table จาก ISPs
- [ ] Advertise our prefixes ออกไปทั้งสอง ISPs
- [ ] BOGON prefixes ถูก filter
- [ ] RPKI invalid routes ถูก reject
- [ ] Multi-homing failover ทำงาน
- [ ] BGP prefix limits set

> **Tip:** ใช้ looking glass หรือ route-server ตรวจสอบว่า prefix ถูก advertise

> **Warning:** Full internet routing table มี 900,000+ prefixes ต้องการ RAM สูง

---

## Summary

Part นี้ครอบคลุม:
- **BGP iBGP/eBGP** configuration
- **Communities** สำหรับ traffic engineering
- **RPKI** สำหรับ route validation
- **Multi-homing** กับหลาย ISPs
- **BGP policies** และ security
- **Monitoring** BGP sessions

---

[← Part 74: MPLS VPN](part-074-mpls-vpn.md) | [Part 76: IPv6 →](part-076-ipv6.md)
