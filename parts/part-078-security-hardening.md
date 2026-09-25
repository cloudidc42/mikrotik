# Part 78: Security Hardening

## สารบัญ
1. [Security Baseline](#baseline)
2. [Firewall Hardening](#firewall)
3. [Certificate Management](#certificates)
4. [Intrusion Detection](#ids)
5. [Access Control](#access-control)
6. [Audit Logging](#logging)
7. [Security Monitoring](#monitoring)
8. [Lab: Security Hardening Checklist](#lab)

---

## 1. Security Baseline {#baseline}

### MikroTik Security Checklist

| Category | Item | Status |
|----------|------|--------|
| Authentication | เปลี่ยน default password | Required |
| Authentication | ปิด default admin account | Recommended |
| Authentication | ใช้ strong passwords | Required |
| Access | ปิด services ที่ไม่ใช้ | Required |
| Access | Restrict management IP | Required |
| Firewall | สร้าง default-deny policy | Required |
| Firmware | Update RouterOS | Required |
| Backup | Encrypt backup files | Recommended |

```bash
# ============================================
# Security Baseline Configuration
# ============================================

# 1. เปลี่ยน admin password
/user set admin password=SuperSecure@2024!

# 2. สร้าง admin user ใหม่ (ไม่ใช้ "admin")
/user add name=netadmin password=StrongPass@123 group=full

# 3. ปิด default admin account (หลังจาก login ด้วย account ใหม่)
/user disable admin

# 4. ปิด services ที่ไม่ต้องการ
/ip service
disable telnet
disable ftp
disable www
disable api
disable api-ssl  # เปิดถ้าต้องการ API access
set ssh port=2222  # เปลี่ยน port จาก default

# 5. เปิดเฉพาะ services ที่จำเป็น
/ip service
set ssh address=10.0.0.0/24 comment="Restricted to management"
set winbox address=10.0.0.0/24 comment="Restricted to management"

# 6. Disable Neighbor Discovery บน WAN
/ip neighbor discovery-settings
set discover-interface-list=LAN

# 7. ปิด IP Cloud (ถ้าไม่ต้องการ)
/ip cloud set ddns-enabled=no update-time=no

# 8. ปิด MAC-based discovery
/tool mac-server
set allowed-interface-list=LAN

/tool mac-server ping
set enabled=no
```

---

## 2. Firewall Hardening {#firewall}

```bash
# ============================================
# Comprehensive Firewall Ruleset
# ============================================

# --- INPUT CHAIN (protect router itself) ---
/ip firewall filter

# Allow established connections
add chain=input connection-state=established,related action=accept \
    comment="Allow established/related"

# Drop invalid packets
add chain=input connection-state=invalid action=drop \
    log=yes log-prefix="INVALID-IN:" comment="Drop invalid"

# Rate limit ICMP
add chain=input protocol=icmp limit=10,20:packet action=accept \
    comment="Allow ICMP (rate limited)"
add chain=input protocol=icmp action=drop comment="Drop excess ICMP"

# Allow management from trusted networks only
add chain=input src-address-list=management-access action=accept \
    comment="Allow management"

# SSH brute force protection
add chain=input protocol=tcp dst-port=2222 connection-state=new \
    src-address-list=ssh-blacklist action=drop \
    comment="Block SSH brute force"
add chain=input protocol=tcp dst-port=2222 connection-state=new \
    action=add-src-to-address-list address-list=ssh-bruteforce \
    address-list-timeout=30s comment="Track SSH attempts"
add chain=input protocol=tcp dst-port=2222 src-address-list=ssh-bruteforce \
    connection-state=new nth=3,3 action=add-src-to-address-list \
    address-list=ssh-blacklist address-list-timeout=1d \
    comment="Block after 3 attempts"

# Default drop everything else
add chain=input action=drop log=yes log-prefix="INPUT-DROP:" \
    comment="Default drop"


# --- FORWARD CHAIN ---

# Allow established
add chain=forward connection-state=established,related action=accept

# Drop invalid
add chain=forward connection-state=invalid action=drop \
    log=yes log-prefix="FWD-INVALID:"

# Block bogon sources
add chain=forward src-address-list=bogon action=drop \
    log=yes log-prefix="BOGON-DROP:"

# Port scan detection
add chain=forward protocol=tcp tcp-flags=fin,!syn,!rst,!ack \
    action=add-src-to-address-list address-list=portscan \
    address-list-timeout=2h comment="NULL scan"
add chain=forward protocol=tcp tcp-flags=fin,syn \
    action=add-src-to-address-list address-list=portscan \
    address-list-timeout=2h comment="XMAS scan"

add chain=forward src-address-list=portscan action=drop \
    log=yes log-prefix="PORTSCAN:"

# DDoS protection
add chain=forward connection-state=new \
    action=add-src-to-address-list address-list=syn-flood-detect \
    address-list-timeout=5s \
    connection-limit=100,32 comment="Detect SYN flood"
add chain=forward src-address-list=syn-flood-detect action=drop \
    comment="Drop SYN flood"

# Allow LAN to WAN
add chain=forward in-interface=ether2 out-interface=ether1 action=accept

# Drop all else
add chain=forward action=drop log=yes log-prefix="FWD-DROP:"


# Address lists
/ip firewall address-list
add list=management-access address=10.0.0.0/24 comment="Management VLAN"
add list=bogon address=10.0.0.0/8 comment="RFC1918"
add list=bogon address=172.16.0.0/12
add list=bogon address=192.168.0.0/16
add list=bogon address=127.0.0.0/8
add list=bogon address=169.254.0.0/16
add list=bogon address=0.0.0.0/8
add list=bogon address=240.0.0.0/4
```

---

## 3. Certificate Management {#certificates}

```bash
# ============================================
# Certificate สำหรับ HTTPS/SSL
# ============================================

# Generate self-signed certificate
/certificate
add name=ca-cert common-name="My CA" key-size=4096 \
    days-valid=3650 key-usage=key-cert-sign,crl-sign

sign ca-cert ca=ca-cert name=ca-cert

add name=server-cert common-name=router.company.com \
    subject-alt-name=IP:192.168.1.1 \
    key-size=2048 days-valid=365

sign server-cert ca=ca-cert name=server-cert

# ใช้ certificate สำหรับ Winbox/API
/ip service
set winbox tls-version=only-1.2
set api-ssl certificate=server-cert

# ตรวจสอบ certificates
/certificate print detail

# Import certificate จากภายนอก
/certificate import file-name=company-ca.crt \
    passphrase="" name=company-ca

# Let's Encrypt (ต้องการ DNS challenge script)
/system script
add name=letsencrypt-renew source={
    # Script to renew Let's Encrypt certificate
    # ต้องใช้ ACME client หรือ external automation
}
```

```python
# Python certificate management
import ssl
import socket
from datetime import datetime
import librouteros

def check_certificate_expiry(host: str, port: int = 443) -> dict:
    """ตรวจสอบ expiry ของ certificate"""
    context = ssl.create_default_context()
    context.check_hostname = False
    context.verify_mode = ssl.CERT_NONE
    
    with socket.create_connection((host, port), timeout=10) as sock:
        with context.wrap_socket(sock, server_hostname=host) as ssock:
            cert = ssock.getpeercert()
    
    not_after = ssl.cert_time_to_seconds(cert['notAfter'])
    expiry = datetime.fromtimestamp(not_after)
    days_left = (expiry - datetime.now()).days
    
    return {
        "host": host,
        "expiry": expiry.isoformat(),
        "days_left": days_left,
        "subject": dict(x[0] for x in cert['subject']),
        "warning": days_left < 30
    }

def monitor_certificates(routers: list):
    for router in routers:
        result = check_certificate_expiry(router["host"], router.get("port", 443))
        if result["warning"]:
            print(f"WARNING: {router['host']} cert expires in {result['days_left']} days!")
        else:
            print(f"OK: {router['host']} cert expires in {result['days_left']} days")
```

---

## 4. Intrusion Detection {#ids}

```bash
# ============================================
# IDS-like Rules บน MikroTik
# ============================================

# Detect port scanning
/ip firewall filter
add chain=input connection-state=new \
    action=add-src-to-address-list \
    address-list=scan-detect address-list-timeout=30m \
    port=23,135,139,445,1433,3306,3389,5900 protocol=tcp \
    comment="Detect attacks on common ports"

add chain=input src-address-list=scan-detect action=drop \
    log=yes log-prefix="IDS-DROP:"

# Detect DNS amplification
add chain=input protocol=udp dst-port=53 \
    connection-limit=10,32 \
    action=add-src-to-address-list \
    address-list=dns-flood address-list-timeout=1h

add chain=input src-address-list=dns-flood action=drop

# Honeypot ports - ไม่มีใคร connect ปกติ
/ip firewall filter
add chain=input protocol=tcp dst-port=4444,1337,31337 \
    action=add-src-to-address-list \
    address-list=honeypot-triggered address-list-timeout=24h \
    log=yes log-prefix="HONEYPOT:"

add chain=input src-address-list=honeypot-triggered action=drop
```

```python
# Syslog-based IDS
import asyncio
import re
from dataclasses import dataclass
from typing import Callable
from datetime import datetime

@dataclass
class SecurityEvent:
    timestamp: datetime
    source_ip: str
    event_type: str
    details: str
    severity: str

class MikroTikIDS:
    """Intrusion Detection สำหรับ MikroTik syslog"""
    
    def __init__(self):
        self.patterns = {
            "ssh_brute_force": (
                r"ssh-blacklist.*src=(\d+\.\d+\.\d+\.\d+)",
                "critical"
            ),
            "port_scan": (
                r"PORTSCAN.*src=(\d+\.\d+\.\d+\.\d+)",
                "high"
            ),
            "invalid_packet": (
                r"INVALID-IN.*src=(\d+\.\d+\.\d+\.\d+)",
                "medium"
            ),
            "honeypot": (
                r"HONEYPOT.*src=(\d+\.\d+\.\d+\.\d+)",
                "critical"
            )
        }
        self.alert_handlers: list[Callable] = []
    
    def add_alert_handler(self, handler: Callable):
        self.alert_handlers.append(handler)
    
    def analyze_log(self, log_line: str) -> SecurityEvent | None:
        for event_type, (pattern, severity) in self.patterns.items():
            match = re.search(pattern, log_line)
            if match:
                return SecurityEvent(
                    timestamp=datetime.utcnow(),
                    source_ip=match.group(1),
                    event_type=event_type,
                    details=log_line,
                    severity=severity
                )
        return None
    
    async def monitor_syslog(self, host: str = "0.0.0.0", port: int = 514):
        """Listen for syslog messages"""
        loop = asyncio.get_event_loop()
        
        transport, protocol = await loop.create_datagram_endpoint(
            lambda: SyslogProtocol(self),
            local_addr=(host, port)
        )
        
        print(f"IDS listening on {host}:{port}")
        await asyncio.sleep(float('inf'))


class SyslogProtocol(asyncio.DatagramProtocol):
    def __init__(self, ids: MikroTikIDS):
        self.ids = ids
    
    def datagram_received(self, data: bytes, addr):
        message = data.decode('utf-8', errors='ignore')
        event = self.ids.analyze_log(message)
        
        if event:
            for handler in self.ids.alert_handlers:
                asyncio.create_task(handler(event))
```

---

## 5. Access Control {#access-control}

```bash
# ============================================
# Advanced Access Control
# ============================================

# Role-based access
/user group
# Read-only group
add name=noc-readonly policy=read,test,winbox

# Operations group
add name=noc-ops policy=read,write,test,winbox,!sensitive,!reboot,!policy,!password

# Full admin (limited)
add name=senior-admin policy=!ftp,!web,read,write,test,policy,password,reboot,sensitive,winbox,api,sniff,rest-api

# Users
/user
add name=noc1 group=noc-readonly password=NOC_Pass1!
add name=ops1 group=noc-ops password=Ops_Pass1!
add name=netadmin group=senior-admin password=Admin_Pass1!

# SSH key authentication
/user ssh-keys import user=netadmin public-key-file=netadmin.pub

# ปิด password auth บน SSH (ใช้ key only)
/ip ssh set strong-crypto=yes \
    host-key-size=4096 \
    forwarding-enabled=no

# Two-factor authentication (ต้องการ RADIUS)
/radius add address=192.168.10.20 secret=radius-secret service=login

/user set netadmin radius=yes
```

---

## 6. Audit Logging {#logging}

```bash
# ============================================
# Comprehensive Audit Logging
# ============================================

# Syslog to remote server
/system logging action
add name=syslog-remote target=remote remote=192.168.10.30 \
    remote-port=514 bsd-syslog=yes syslog-facility=daemon \
    syslog-severity=info comment="Remote syslog"

/system logging
add topics=system action=syslog-remote
add topics=firewall action=syslog-remote
add topics=account action=syslog-remote  # Login/logout events
add topics=critical action=syslog-remote
add topics=error action=syslog-remote
add topics=warning action=syslog-remote
add topics=info action=syslog-remote

# Log rule สำหรับ configuration changes
/system logging
add topics=config action=syslog-remote

# Log network events
add topics=route action=syslog-remote
add topics=bgp action=syslog-remote
add topics=ospf action=syslog-remote
add topics=vrrp action=syslog-remote


# Script สำหรับ log config changes
/system script
add name=log-config-change source={
    :local user [/user get [find active] name]
    :log info ("Config changed by user: " . $user)
}
```

---

## 7. Lab: Security Hardening Checklist {#lab}

### Security Hardening Steps

```bash
# Phase 1: Authentication Hardening
/user set admin password=NewStrongPass@2024!
/user add name=secadmin group=full password=SecAdmin@2024!
/user disable admin

# Phase 2: Service Hardening
/ip service disable telnet,ftp,www,api
/ip service set ssh port=2222 address=10.0.0.0/24
/ip service set winbox address=10.0.0.0/24

# Phase 3: Firewall Hardening
/ip firewall filter add chain=input connection-state=established,related action=accept
/ip firewall filter add chain=input connection-state=invalid action=drop
/ip firewall filter add chain=input src-address-list=management-access action=accept
/ip firewall filter add chain=input action=drop log=yes

# Phase 4: Discovery Disable
/ip neighbor discovery-settings set discover-interface-list=LAN
/tool mac-server set allowed-interface-list=LAN

# Phase 5: Logging
/system logging action add name=remote target=remote remote=10.0.0.100
/system logging add topics=firewall,system,account action=remote

# Phase 6: Verify
/ip service print
/user print
/ip firewall filter print
```

### Security Score Checklist

- [ ] Default password เปลี่ยนแล้ว
- [ ] Default admin disabled
- [ ] Unused services disabled
- [ ] Management access จำกัด IP
- [ ] Firewall default-deny configured
- [ ] SSH key-based auth configured
- [ ] Syslog to remote server
- [ ] RouterOS up to date
- [ ] Backup encrypted
- [ ] SNMP community เปลี่ยน

> **Warning:** ทดสอบ firewall rules ก่อน apply บน production เสมอ

> **Tip:** ใช้ `/ip firewall connection tracking print` เพื่อ debug connectivity issues

---

## Summary

Part นี้ครอบคลุม:
- **Security baseline** configuration
- **Firewall hardening** กับ DDoS protection
- **Certificate management** สำหรับ SSL/TLS
- **IDS rules** สำหรับ threat detection
- **Access control** ด้วย RBAC
- **Audit logging** สู่ remote syslog

---

[← Part 77: Traffic Analysis](part-077-traffic-analysis.md) | [Part 79: CAPsMAN →](part-079-capsman.md)
