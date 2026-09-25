# Part 79: CAPsMAN - Centralized AP Management

## สารบัญ
1. [CAPsMAN Architecture](#architecture)
2. [CAPsMAN Server Setup](#server)
3. [CAP Registration](#registration)
4. [Profiles Configuration](#profiles)
5. [Roaming](#roaming)
6. [Security](#security)
7. [Monitoring](#monitoring)
8. [Lab: CAPsMAN Deployment](#lab)

---

## 1. CAPsMAN Architecture {#architecture}

CAPsMAN (Controlled Access Point system MANager) ช่วยจัดการ WiFi APs ทั้งหมดจาก central controller

```
                    [CAPsMAN Controller]
                    192.168.1.1
                         │
           ┌─────────────┼─────────────┐
           │             │             │
      [CAP-AP1]     [CAP-AP2]     [CAP-AP3]
     Floor 1        Floor 2       Meeting Room
     192.168.1.11   192.168.1.12  192.168.1.13
```

### CAPsMAN Components

| Component | Description |
|-----------|-------------|
| CAPsMAN | Controller (MikroTik router/switch) |
| CAP | Controlled Access Point (hAP, wAP, etc.) |
| Master Configuration | รวม all settings profiles |
| Channel | RF channel configuration |
| Security | WPA2/WPA3 settings |
| Datapath | VLAN, bridge configuration |

---

## 2. CAPsMAN Server Setup {#server}

```bash
# ============================================
# CAPsMAN Controller Configuration
# ============================================

# Enable CAPsMAN
/caps-man manager
set enabled=yes
set ca-certificate=auto  # หรือใช้ certificate ที่ generate เอง

# สร้าง Channel profiles
/caps-man channel
add name=ch-2.4ghz-auto band=2ghz-b/g/n frequency=auto \
    extension-channel=Ce width=20/40 comment="2.4GHz auto"

add name=ch-5ghz-auto band=5ghz-a/n/ac frequency=auto \
    extension-channel=Ceee width=80 comment="5GHz auto"

add name=ch-5ghz-ch36 band=5ghz-a/n/ac frequency=5180 \
    extension-channel=Ceee width=80 comment="5GHz Ch36"

# Security profiles
/caps-man security
add name=wpa2-corp authentication-types=wpa2-psk \
    encryption=aes-ccm passphrase=Corp@WiFi2024! \
    group-key-update=5m comment="Corporate WPA2"

add name=wpa3-corp authentication-types=wpa2-psk,wpa3-psk \
    encryption=aes-ccm passphrase=Corp@WiFi2024! \
    comment="WPA2/WPA3 transition"

add name=guest-open authentication-types=wpa2-psk \
    encryption=aes-ccm passphrase=Guest@Wifi! \
    comment="Guest network"

# Datapath profiles (bridge/VLAN)
/caps-man datapath
add name=dp-corporate bridge=bridge-corp client-to-client-forwarding=yes \
    vlan-id=100 vlan-mode=use-tag comment="Corporate VLAN"

add name=dp-guest bridge=bridge-guest client-to-client-forwarding=no \
    vlan-id=200 vlan-mode=use-tag comment="Guest VLAN"

# Master configurations
/caps-man configuration
add name=corp-2.4ghz \
    ssid="CORP-WIFI" \
    channel=ch-2.4ghz-auto \
    security=wpa2-corp \
    datapath=dp-corporate \
    rates.ht-basic-mcs=mcs-0,mcs-1,mcs-2,mcs-7 \
    tx-power=17 \
    comment="Corporate 2.4GHz"

add name=corp-5ghz \
    ssid="CORP-WIFI-5G" \
    channel=ch-5ghz-auto \
    security=wpa2-corp \
    datapath=dp-corporate \
    tx-power=20 \
    comment="Corporate 5GHz"

add name=guest-2.4ghz \
    ssid="Guest-WiFi" \
    channel=ch-2.4ghz-auto \
    security=guest-open \
    datapath=dp-guest \
    tx-power=15 \
    comment="Guest 2.4GHz"

# Provisioning rules
/caps-man provisioning
add master-configuration=corp-2.4ghz \
    name-format=cap-%I \
    hw-supported-modes=g,gn comment="2.4GHz APs"

add master-configuration=corp-5ghz \
    name-format=cap-5g-%I \
    hw-supported-modes=ac comment="5GHz APs"
```

---

## 3. CAP Registration {#registration}

```bash
# ============================================
# CAP (Access Point) Configuration
# ============================================

# บน CAP router (เช่น hAP ac2)

# Enable CAP mode
/interface wireless cap
set enabled=yes interfaces=wlan1,wlan2 \
    discovery-interfaces=ether1 \
    caps-man-addresses=192.168.1.1 \
    certificate=request \
    comment="Register to CAPsMAN"

# หรือใช้ DHCP discovery (ถ้า CAPsMAN และ CAP อยู่ subnet เดียวกัน)
/interface wireless cap
set enabled=yes interfaces=wlan1,wlan2 \
    discovery-interfaces=ether1


# บน CAPsMAN - verify CAP registration
/caps-man remote-cap print
# Expected: เห็น CAP ใน list พร้อม state=running

/caps-man registration-table print
# Expected: เห็น wireless clients

# ตรวจสอบ interfaces บน CAPsMAN
/caps-man interface print
```

---

## 4. Profiles Configuration {#profiles}

```bash
# ============================================
# Advanced Profile Configuration
# ============================================

# Radio rates optimization
/caps-man rates
add name=rates-11n-2.4 basic=6Mbps,12Mbps,24Mbps \
    ht-basic-mcs=mcs-0,mcs-8 \
    ht-supported-mcs=mcs-0,mcs-1,mcs-2,mcs-3,mcs-4,mcs-5,mcs-6,mcs-7,mcs-8,mcs-9,mcs-10,mcs-11,mcs-12,mcs-13,mcs-14,mcs-15

# Advanced security
/caps-man security
add name=wpa2-enterprise \
    authentication-types=wpa2-eap \
    eap-methods=passthrough \
    encryption=aes-ccm \
    comment="WPA2-Enterprise via RADIUS"

# RADIUS สำหรับ 802.1X
/radius add address=192.168.10.20 secret=wpa2radius service=wireless

# Datapath กับ VLAN dynamic (per-user)
/caps-man datapath
add name=dp-dynamic vlan-mode=use-tag \
    local-forwarding=no comment="Dynamic VLAN"

# Access list สำหรับ filter clients
/caps-man access-list
add action=accept interface=cap-01 \
    mac-address=AA:BB:CC:DD:EE:FF \
    comment="Allow specific device"

add action=reject comment="Block others (fallthrough)"

# Band steering (prefer 5GHz)
/caps-man channel
set ch-2.4ghz-auto band-steering=yes
```

---

## 5. Roaming {#roaming}

```bash
# ============================================
# Seamless Roaming Configuration
# ============================================

# Fast BSS Transition (802.11r)
/caps-man security
set wpa2-corp ft=yes ft-over-ds=yes \
    comment="Enable 802.11r Fast Transition"

# 802.11k (Neighbor reports) - ให้ client รู้ว่ามี AP อื่น
# 802.11v (BSS Transition Management) - แนะนำ client ย้าย AP

# เพื่อให้ roaming seamless:
# 1. ทุก AP ใช้ SSID เดียวกัน
# 2. ทุก AP ใช้ security configuration เดียวกัน  
# 3. CAPsMAN จัดการ key caching

# PMKSA caching สำหรับ fast reconnect
/caps-man security
set wpa2-corp group-key-update=0  # Disable periodic rekey สำหรับ roaming test

# ตรวจสอบ roaming events
/log print where topics~"wireless"
# Expected: เห็น "Station roamed from cap-01 to cap-02"
```

---

## 6. Monitoring {#monitoring}

```bash
# ============================================
# CAPsMAN Monitoring
# ============================================

# Remote CAPs status
/caps-man remote-cap print detail

# Registration table (connected clients)
/caps-man registration-table print detail

# Client statistics
/caps-man registration-table print stats

# Interface statistics
/caps-man interface print stats

# AP utilization
/caps-man channel print
```

```python
# Python CAPsMAN monitoring
import librouteros
from datetime import datetime

def monitor_capsman(controller_host: str, username: str, password: str) -> dict:
    conn = librouteros.connect(host=controller_host, username=username, password=password)
    
    # Remote CAPs
    caps = list(conn('/caps-man/remote-cap/print'))
    cap_summary = []
    for cap in caps:
        c = dict(cap)
        cap_summary.append({
            "name": c.get("name"),
            "address": c.get("address"),
            "state": c.get("state"),
            "radio-count": c.get("radio-count"),
            "uptime": c.get("uptime")
        })
    
    # Registered clients
    clients = list(conn('/caps-man/registration-table/print'))
    client_count = len(clients)
    
    # Client by SSID
    ssid_count = {}
    for client in clients:
        c = dict(client)
        ssid = c.get("ssid", "unknown")
        ssid_count[ssid] = ssid_count.get(ssid, 0) + 1
    
    conn.close()
    
    return {
        "timestamp": datetime.utcnow().isoformat(),
        "controller": controller_host,
        "total_aps": len(cap_summary),
        "aps_running": sum(1 for c in cap_summary if c.get("state") == "running"),
        "total_clients": client_count,
        "clients_by_ssid": ssid_count,
        "aps": cap_summary
    }


if __name__ == "__main__":
    import json
    status = monitor_capsman("192.168.1.1", "admin", "password")
    print(json.dumps(status, indent=2))
```

---

## 7. Lab: CAPsMAN Deployment {#lab}

### Lab Topology

```
[CAPsMAN Controller]
192.168.1.1 (RB4011 or CCR)
     │
     │ Layer 2 (ether3-10 trunk)
     │
[Management Switch]
     │
     ├── [CAP-AP1] hAP ac3 - Floor 1
     ├── [CAP-AP2] wAP ac   - Floor 2
     └── [CAP-AP3] Audience - Conference Room
```

### Lab Steps

```bash
# Step 1: Controller setup
/caps-man manager set enabled=yes
/caps-man channel add name=ch1 band=2ghz-b/g/n frequency=auto
/caps-man security add name=sec1 authentication-types=wpa2-psk passphrase=TestWifi123
/caps-man configuration add name=conf1 ssid=TestSSID channel=ch1 security=sec1

# Step 2: CAP setup (on each AP)
/interface wireless cap set enabled=yes interfaces=wlan1 caps-man-addresses=192.168.1.1

# Step 3: Verify
/caps-man remote-cap print  # บน controller
/caps-man registration-table print  # เห็น clients

# Step 4: Test roaming
# Connect client to AP1, walk to AP2
# Client ควร roam โดยไม่ disconnect
```

### Verification Checklist

- [ ] CAPsMAN manager enabled
- [ ] CAPs registered กับ controller
- [ ] SSID broadcast ถูกต้อง
- [ ] Clients connect ได้
- [ ] VLAN tagging ทำงาน
- [ ] Roaming ทำงาน seamlessly
- [ ] Monitoring dashboard แสดง APs และ clients

> **Tip:** ใช้ `/caps-man remote-cap print` เพื่อ debug CAP registration

> **Note:** CAP และ CAPsMAN ต้องมี layer 2 connectivity หรือ config `caps-man-addresses` เป็น IP

---

## Summary

Part นี้ครอบคลุม:
- **CAPsMAN architecture** และ components
- **Controller setup** กับ profiles
- **CAP registration** procedures
- **VLAN และ security** profiles
- **Seamless roaming** ด้วย 802.11r
- **Monitoring** CAPsMAN deployment

---

[← Part 78: Security Hardening](part-078-security-hardening.md) | [Part 80: CHR Cloud →](part-080-chr-cloud.md)
