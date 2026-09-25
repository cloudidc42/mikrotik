# Part 76: IPv6

## สารบัญ
1. [IPv6 Fundamentals](#fundamentals)
2. [IPv6 Address Types](#address-types)
3. [DHCPv6 Configuration](#dhcpv6)
4. [SLAAC](#slaac)
5. [OSPFv3](#ospfv3)
6. [Dual-Stack Configuration](#dual-stack)
7. [IPv6 Firewall](#firewall)
8. [Lab: IPv6 Dual-Stack Network](#lab)

---

## 1. IPv6 Fundamentals {#fundamentals}

IPv6 ใช้ address 128-bit แทน 32-bit ของ IPv4 รองรับ address ได้ 340 undecillion addresses

### IPv6 Header

```
 0                   1                   2                   3
 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1 2 3 4 5 6 7 8 9 0 1
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|Version| Traffic Class |           Flow Label                  |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|         Payload Length        |  Next Header  |   Hop Limit   |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                         Source Address                        |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
|                                                               |
+                                                               +
|                      Destination Address                      |
+                                                               +
|                                                               |
+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+-+
```

### IPv6 Address Format

```
2001:0db8:85a3:0000:0000:8a2e:0370:7334
   ↑    ↑    ↑    ↑    ↑    ↑    ↑    ↑
  16   16   16   16   16   16   16   16  bits (each group)

Simplified:
2001:db8:85a3::8a2e:370:7334
```

### Address Types

| Type | Prefix | Example | Use |
|------|--------|---------|-----|
| Global Unicast | 2000::/3 | 2001:db8::/32 | Global internet |
| Link-Local | fe80::/10 | fe80::1 | เฉพาะ segment |
| Unique Local | fc00::/7 | fd00::/8 | Private/internal |
| Multicast | ff00::/8 | ff02::1 | Group communication |
| Loopback | ::1/128 | ::1 | Loopback |

---

## 2. IPv6 Configuration {#address-types}

```bash
# ============================================
# IPv6 Basic Configuration
# ============================================

# Enable IPv6
/ipv6 settings
set forward=yes accept-router-advertisements=yes

# Static IPv6 address
/ipv6 address
add address=2001:db8:1::1/64 interface=ether1 comment="WAN IPv6"
add address=2001:db8:2::1/64 interface=ether2 comment="LAN IPv6"
add address=fd00:1:2:3::1/64 interface=ether2 comment="ULA for LAN"

# Link-local (auto-generated, แต่ set manually ได้)
add address=fe80::1/64 interface=ether1 link-local=yes

# ตรวจสอบ IPv6 addresses
/ipv6 address print

# IPv6 route
/ipv6 route
add dst-address=::/0 gateway=2001:db8:1::254 comment="Default IPv6 route"
add dst-address=2001:db8:3::/48 gateway=2001:db8:1::2 comment="Customer subnet"

# ตรวจสอบ IPv6 routes
/ipv6 route print
```

---

## 3. DHCPv6 {#dhcpv6}

```bash
# ============================================
# DHCPv6 Server Configuration
# ============================================

# DHCPv6 pool
/ipv6 dhcp-server
add name=dhcpv6-lan interface=ether2 \
    address-pool=ipv6-pool \
    lease-time=12h \
    comment="DHCPv6 for LAN"

/ipv6 pool
add name=ipv6-pool \
    prefix=2001:db8:2::/48 \
    prefix-length=64 \
    comment="IPv6 pool for clients"

# DHCPv6 options
/ipv6 dhcp-server option
add name=dns code=23 value="0x20010db800000000000000000000000100000000000000000000000000000001"

# Prefix Delegation สำหรับ CPE routers
/ipv6 dhcp-server
set dhcpv6-lan address-pool=pd-pool

/ipv6 pool
add name=pd-pool prefix=2001:db8::/32 prefix-length=48 comment="PD pool for CPE"


# ============================================
# DHCPv6 Client สำหรับ WAN
# ============================================

/ipv6 dhcp-client
add name=dhcpv6-wan interface=ether1 \
    request=prefix \
    pool-name=wan-pd-pool \
    pool-prefix-length=64 \
    add-default-route=yes \
    comment="Get IPv6 from ISP via PD"

# ตรวจสอบ DHCPv6 client
/ipv6 dhcp-client print
/ipv6 dhcp-client binding print
```

---

## 4. SLAAC {#slaac}

```bash
# ============================================
# SLAAC - Stateless Address Autoconfiguration
# ============================================

# Enable Router Advertisement
/ipv6 nd
add interface=ether2 \
    ra-interval=20s-200s \
    ra-lifetime=1800s \
    reachable-time=0s \
    retransmit-interval=0s \
    managed-address-configuration=no \
    other-configuration=yes \
    comment="RA for LAN clients"

# Advertise prefix สำหรับ SLAAC
/ipv6 nd prefix
add prefix=2001:db8:2::/64 \
    interface=ether2 \
    valid-lifetime=2d \
    preferred-lifetime=1d \
    on-link=yes \
    autonomous=yes \
    comment="SLAAC prefix"

# DNS via RA (RDNSS)
/ipv6 nd
set [find interface=ether2] \
    dns=2001:db8::1,2001:4860:4860::8888

# ตรวจสอบ RA
/ipv6 nd print detail
/ipv6 nd prefix print

# Client จะ generate address:
# Prefix: 2001:db8:2::/64
# Interface ID: EUI-64 from MAC หรือ random (Privacy Extensions)
# Result: 2001:db8:2::xxxx:xxxx:xxxx:xxxx/64
```

---

## 5. OSPFv3 {#ospfv3}

```bash
# ============================================
# OSPFv3 สำหรับ IPv6 Routing
# ============================================

/routing ospf instance
add name=ospfv3 version=3 router-id=10.0.0.1 \
    comment="OSPFv3 for IPv6"

/routing ospf area
add name=backbone instance=ospfv3 area-id=0.0.0.0

/routing ospf interface-template
add interfaces=ether1,ether2 area=backbone \
    type=broadcast \
    hello-interval=10s dead-interval=40s \
    priority=1

# ตรวจสอบ OSPFv3
/routing ospf neighbor print
/ipv6 route print where ospf=yes


# BGP สำหรับ IPv6 (MP-BGP)
/routing bgp peer
add name=ipv6-upstream \
    remote-address=2001:db8:1::254 \
    remote-as=65001 \
    address-families=ipv6 \
    comment="IPv6 upstream BGP"

/routing bgp network
add network=2001:db8::/32 comment="Advertise our IPv6 block"
```

---

## 6. Dual-Stack {#dual-stack}

```bash
# ============================================
# Dual-Stack Network Configuration
# ============================================

# ============================================
# Router Dual-Stack Setup
# ============================================

# IPv4 addresses
/ip address
add address=192.168.1.1/24 interface=ether2 comment="IPv4 LAN"
add address=203.0.113.1/30 interface=ether1 comment="IPv4 WAN"

# IPv6 addresses
/ipv6 address
add address=2001:db8:2::1/64 interface=ether2 comment="IPv6 LAN"
add address=2001:db8:1::1/30 interface=ether1 comment="IPv6 WAN"

# IPv4 default route
/ip route add dst-address=0.0.0.0/0 gateway=203.0.113.254

# IPv6 default route
/ipv6 route add dst-address=::/0 gateway=2001:db8:1::254

# DHCP IPv4 สำหรับ clients
/ip dhcp-server
add name=dhcp-lan interface=ether2 address-pool=pool-lan

/ip pool add name=pool-lan ranges=192.168.1.10-192.168.1.250

/ip dhcp-server network
add address=192.168.1.0/24 gateway=192.168.1.1 \
    dns-server=8.8.8.8,2001:4860:4860::8888

# SLAAC + DHCPv6 IPv6 สำหรับ clients
/ipv6 nd add interface=ether2 managed-address-configuration=no other-configuration=yes
/ipv6 nd prefix add prefix=2001:db8:2::/64 interface=ether2 autonomous=yes

# NAT64 สำหรับ IPv6-only clients ที่ต้องการ access IPv4
# (ต้องการ NAT64 gateway แยกต่างหาก)


# ============================================
# DNS64 Configuration (ถ้าใช้ BIND)
# ============================================
# named.conf option:
# dns64 64:ff9b::/96 {
#     clients { any; };
#     mapped { !rfc1918; any; };
# };
```

---

## 7. IPv6 Firewall {#firewall}

```bash
# ============================================
# IPv6 Firewall Rules
# ============================================

/ipv6 firewall filter

# INPUT chain - protect router
add chain=input connection-state=established,related action=accept \
    comment="Allow established"
add chain=input connection-state=invalid action=drop \
    comment="Drop invalid"
add chain=input protocol=icmpv6 action=accept \
    comment="Allow ICMPv6 (required for IPv6)"
add chain=input src-address=fe80::/10 action=accept \
    comment="Allow link-local"
add chain=input src-address=::1 action=accept \
    comment="Allow loopback"
add chain=input dst-port=22 protocol=tcp src-address=2001:db8:2::/64 \
    action=accept comment="SSH from LAN"
add chain=input action=drop comment="Drop all else"

# FORWARD chain
add chain=forward connection-state=established,related action=accept
add chain=forward connection-state=invalid action=drop
add chain=forward protocol=icmpv6 action=accept
# Drop packets to RFC 4193 (ULA) from WAN
add chain=forward in-interface=ether1 dst-address=fc00::/7 \
    action=drop comment="Block WAN to ULA"
# Allow LAN to WAN
add chain=forward in-interface=ether2 out-interface=ether1 action=accept
add chain=forward action=drop

# ICMPv6 - สำคัญมาก ต้องอนุญาต
/ipv6 firewall filter
add chain=input protocol=icmpv6 icmp-type=1 action=accept comment="Dest unreachable"
add chain=input protocol=icmpv6 icmp-type=2 action=accept comment="Packet too big"
add chain=input protocol=icmpv6 icmp-type=3 action=accept comment="Time exceeded"
add chain=input protocol=icmpv6 icmp-type=4 action=accept comment="Parameter problem"
add chain=input protocol=icmpv6 icmp-type=133 action=accept comment="Router solicitation"
add chain=input protocol=icmpv6 icmp-type=134 action=accept comment="Router advertisement"
add chain=input protocol=icmpv6 icmp-type=135 action=accept comment="Neighbor solicitation"
add chain=input protocol=icmpv6 icmp-type=136 action=accept comment="Neighbor advertisement"


# NAT66 (IPv6 masquerade, ใช้กรณี ULA บน LAN)
/ipv6 firewall nat
add chain=srcnat out-interface=ether1 action=masquerade \
    comment="NAT66 for ULA"
```

---

## 8. Lab: IPv6 Dual-Stack Network {#lab}

### Lab Topology

```
ISP (2001:db8:1::254/64)
     │
  [Router]
  WAN: 2001:db8:1::1/64
  LAN: 2001:db8:2::1/64
  LAN: 192.168.1.1/24 (IPv4)
     │
  [LAN Clients]
  IPv4: 192.168.1.x (DHCP)
  IPv6: 2001:db8:2::xxxx (SLAAC)
```

### Lab Steps

```bash
# Step 1: Configure interfaces
/ipv6 address add address=2001:db8:1::1/64 interface=ether1
/ipv6 address add address=2001:db8:2::1/64 interface=ether2
/ipv6 route add dst-address=::/0 gateway=2001:db8:1::254

# Step 2: SLAAC for clients
/ipv6 nd add interface=ether2
/ipv6 nd prefix add prefix=2001:db8:2::/64 interface=ether2

# Step 3: Firewall
/ipv6 firewall filter add chain=input protocol=icmpv6 action=accept
/ipv6 firewall filter add chain=forward connection-state=established,related action=accept
/ipv6 firewall filter add chain=forward in-interface=ether2 action=accept

# Step 4: Test
/ping 2001:4860:4860::8888
# From client: ping6 2001:4860:4860::8888
```

### Verification Checklist

- [ ] Router มี IPv6 address บนทั้งสอง interfaces
- [ ] Default IPv6 route ทำงาน
- [ ] Clients รับ IPv6 address จาก SLAAC
- [ ] Clients ping IPv6 internet ได้
- [ ] Firewall อนุญาต ICMPv6
- [ ] DNS resolver รองรับ AAAA records

> **Tip:** ใช้ `ipv6.google.com` ทดสอบ IPv6 connectivity

> **Warning:** ต้องอนุญาต ICMPv6 ในทุก firewall - IPv6 ใช้ ICMPv6 สำหรับ NDP และ Path MTU Discovery

---

## Summary

Part นี้ครอบคลุม:
- **IPv6 addressing** และ address types
- **DHCPv6** สำหรับ stateful assignment
- **SLAAC** สำหรับ stateless autoconfiguration
- **OSPFv3** สำหรับ IPv6 routing
- **Dual-stack** เพื่อรองรับทั้ง IPv4 และ IPv6
- **IPv6 firewall** พร้อม ICMPv6 rules

---

[← Part 75: BGP Internet](part-075-bgp-internet.md) | [Part 77: Traffic Analysis →](part-077-traffic-analysis.md)
