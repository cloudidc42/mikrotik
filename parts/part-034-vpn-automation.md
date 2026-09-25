# Part 34: VPN Automation ใน RouterOS

## บทนำ

RouterOS รองรับ VPN protocols หลายประเภท ตั้งแต่ PPTP แบบ legacy ไปจนถึง WireGuard ที่ทันสมัย ในบทนี้เราจะ configure และ automate VPN servers และ clients ทุกประเภท

---

## 34.1 PPTP Server (Legacy)

> **Warning:** PPTP มีความปลอดภัยต่ำ ไม่แนะนำสำหรับ production ใช้เฉพาะถ้า compatibility จำเป็น

```routeros
# Setup PPTP Server
/interface pptp-server server set \
    enabled=yes \
    authentication=mschap1,mschap2 \
    mtu=1450 \
    mru=1450 \
    mrru=disabled \
    keepalive-timeout=30 \
    default-profile=default-encryption

# สร้าง user สำหรับ PPTP
/ppp secret add \
    name=vpnuser1 \
    password=VPNpass123 \
    service=pptp \
    profile=default-encryption \
    remote-address=10.100.0.1

# Firewall สำหรับ PPTP
/ip firewall filter add chain=input \
    protocol=tcp \
    dst-port=1723 \
    action=accept \
    comment="Allow PPTP"

/ip firewall filter add chain=input \
    protocol=gre \
    action=accept \
    comment="Allow GRE (PPTP)"

# ดู PPTP sessions
/interface pptp-server print
/ppp active print where service=pptp
```

---

## 34.2 L2TP/IPSec Server

```routeros
# ตั้งค่า L2TP Server
/interface l2tp-server server set \
    enabled=yes \
    authentication=mschap1,mschap2 \
    mtu=1450 \
    mru=1450 \
    max-sessions=100 \
    use-ipsec=required \
    ipsec-secret=MySharedSecret \
    default-profile=default-encryption

# IP Pool สำหรับ L2TP clients
/ip pool add name=l2tp-pool ranges=192.168.200.1-192.168.200.100

# L2TP Profile
/ppp profile add \
    name=l2tp-profile \
    local-address=192.168.200.0 \
    remote-address=l2tp-pool \
    dns-server=8.8.8.8,1.1.1.1 \
    use-compression=no \
    use-encryption=yes

# สร้าง users
/ppp secret add \
    name=l2tp-user1 \
    password=L2TPpass123 \
    service=l2tp \
    profile=l2tp-profile

# IPSec Policies
/ip ipsec proposal add \
    name=l2tp-proposal \
    auth-algorithms=sha1 \
    enc-algorithms=aes-256-cbc \
    pfs-group=modp1024

/ip ipsec policy add \
    src-address=0.0.0.0/0 \
    dst-address=0.0.0.0/0 \
    protocol=all \
    proposal=l2tp-proposal \
    action=encrypt \
    level=require

# Firewall สำหรับ L2TP/IPSec
/ip firewall filter add chain=input protocol=udp dst-port=1701 action=accept comment="L2TP"
/ip firewall filter add chain=input protocol=udp dst-port=500 action=accept comment="IKE"
/ip firewall filter add chain=input protocol=udp dst-port=4500 action=accept comment="IPSec NAT-T"
/ip firewall filter add chain=input protocol=ipsec-esp action=accept comment="IPSec ESP"
/ip firewall filter add chain=input protocol=ipsec-ah action=accept comment="IPSec AH"

# ดู L2TP/IPSec connections
/interface l2tp-server print
/ip ipsec active-peers print
```

---

## 34.3 OpenVPN Server

```routeros
# ตั้งค่า OpenVPN Server (RouterOS รองรับ UDP และ TCP)

# สร้าง certificates ก่อน
/certificate add \
    name=ca-cert \
    common-name=CA \
    key-size=2048 \
    days-valid=3650 \
    key-usage=key-cert-sign,crl-sign

/certificate sign ca-cert ca=ca-cert

/certificate add \
    name=server-cert \
    common-name=server \
    key-size=2048 \
    days-valid=3650 \
    key-usage=digital-signature,key-encipherment,tls-server

/certificate sign server-cert ca=ca-cert

/certificate add \
    name=client1-cert \
    common-name=client1 \
    key-size=2048 \
    days-valid=3650 \
    key-usage=digital-signature,key-encipherment,tls-client

/certificate sign client1-cert ca=ca-cert

# IP Pool สำหรับ OpenVPN
/ip pool add name=ovpn-pool ranges=10.200.0.1-10.200.0.100

# OpenVPN Profile
/ppp profile add \
    name=ovpn-profile \
    local-address=10.200.0.0 \
    remote-address=ovpn-pool

# OpenVPN Server
/interface ovpn-server server set \
    enabled=yes \
    port=1194 \
    mode=ip \
    protocol=udp \
    certificate=server-cert \
    ca=ca-cert \
    require-client-certificate=yes \
    auth=sha1 \
    cipher=aes256 \
    tls-version=any \
    netmask=255.255.255.0 \
    default-profile=ovpn-profile \
    max-mtu=1500

# สร้าง user
/ppp secret add \
    name=ovpn-user1 \
    password=OVPNpass123 \
    service=ovpn \
    profile=ovpn-profile

# Firewall
/ip firewall filter add chain=input protocol=udp dst-port=1194 action=accept comment="OpenVPN UDP"

# Export client certificate
/certificate export-certificate client1-cert type=pkcs12 export-passphrase=ClientPass
# ไฟล์ will be at /cert_export/ directory

# Script สร้าง OpenVPN client config
:local genOVPNConfig do={
    :local serverIP $1
    :local clientName $2
    
    :local config "client\r\n"
    :set config ($config . "dev tun\r\n")
    :set config ($config . "proto udp\r\n")
    :set config ($config . "remote " . $serverIP . " 1194\r\n")
    :set config ($config . "resolv-retry infinite\r\n")
    :set config ($config . "nobind\r\n")
    :set config ($config . "persist-key\r\n")
    :set config ($config . "persist-tun\r\n")
    :set config ($config . "remote-cert-tls server\r\n")
    :set config ($config . "auth SHA1\r\n")
    :set config ($config . "cipher AES-256-CBC\r\n")
    :set config ($config . "verb 3\r\n")
    :set config ($config . "auth-user-pass\r\n")
    
    # Save to file
    /file remove [find name=($clientName . ".ovpn")]
    /tool fetch url="data:," dst-path=($clientName . ".ovpn")
    /file set [find name=($clientName . ".ovpn")] contents=$config
    :put "Config generated: " . $clientName . ".ovpn"
}

[$genOVPNConfig "203.0.113.1" "client1"]
```

---

## 34.4 WireGuard (RouterOS 7.x)

```routeros
# WireGuard ใช้ได้ใน RouterOS 7.x เท่านั้น
# ตรวจสอบ version ก่อน
:local version [/system resource get version]
:put "RouterOS version: $version"

# Setup WireGuard Interface
/interface wireguard add \
    name=wg0 \
    listen-port=51820 \
    comment="WireGuard VPN"

# ดู public key (สร้างอัตโนมัติ)
/interface wireguard print

# เพิ่ม peer (client)
/interface wireguard peers add \
    interface=wg0 \
    public-key="CLIENT_PUBLIC_KEY_HERE" \
    allowed-address=10.0.0.2/32 \
    comment="Client 1"

# IP address สำหรับ wg interface
/ip address add address=10.0.0.1/24 interface=wg0

# Firewall
/ip firewall filter add chain=input \
    protocol=udp \
    dst-port=51820 \
    action=accept \
    comment="WireGuard"

# Script generate WireGuard keys
:local genWGKeys "
# สร้าง WireGuard key pair ผ่าน script
# RouterOS 7.x
/interface wireguard add name=wg-temp
:local privKey [/interface wireguard get wg-temp private-key]
:local pubKey [/interface wireguard get wg-temp public-key]
:put \"Private Key: \$privKey\"
:put \"Public Key: \$pubKey\"
/interface wireguard remove wg-temp
"

# สร้าง WireGuard config สำหรับ client
:local genWGClientConfig do={
    :local serverPubKey $1
    :local serverEndpoint $2
    :local clientPrivKey $3
    :local clientIP $4
    :local dns $5
    
    :if ([:len $dns] = 0) do={ :set dns "1.1.1.1" }
    
    :local config "[Interface]\r\n"
    :set config ($config . "PrivateKey = " . $clientPrivKey . "\r\n")
    :set config ($config . "Address = " . $clientIP . "\r\n")
    :set config ($config . "DNS = " . $dns . "\r\n\r\n")
    :set config ($config . "[Peer]\r\n")
    :set config ($config . "PublicKey = " . $serverPubKey . "\r\n")
    :set config ($config . "Endpoint = " . $serverEndpoint . ":51820\r\n")
    :set config ($config . "AllowedIPs = 0.0.0.0/0\r\n")
    :set config ($config . "PersistentKeepalive = 25\r\n")
    
    :return $config
}
```

---

## 34.5 SSTP Server

```routeros
# SSTP (Secure Socket Tunneling Protocol) - SSL-based VPN

# Certificate สำหรับ SSTP
/certificate add \
    name=sstp-cert \
    common-name=vpn.company.com \
    key-size=2048 \
    days-valid=3650 \
    key-usage=digital-signature,key-encipherment,tls-server

/certificate sign sstp-cert ca=ca-cert

# SSTP Server
/interface sstp-server server set \
    enabled=yes \
    port=443 \
    certificate=sstp-cert \
    verify-client-certificate=no \
    authentication=mschap1,mschap2 \
    default-profile=default-encryption

# IP Pool สำหรับ SSTP
/ip pool add name=sstp-pool ranges=192.168.201.1-192.168.201.50

# SSTP Profile
/ppp profile add \
    name=sstp-profile \
    local-address=192.168.201.0 \
    remote-address=sstp-pool \
    dns-server=8.8.8.8

# SSTP User
/ppp secret add \
    name=sstp-user1 \
    password=SSTPass123 \
    service=sstp \
    profile=sstp-profile

# Firewall
/ip firewall filter add chain=input \
    protocol=tcp \
    dst-port=443 \
    action=accept \
    comment="SSTP VPN"
```

---

## 34.6 VPN Client Scripts

```routeros
# L2TP Client
/interface l2tp-client add \
    name=l2tp-to-hq \
    connect-to=vpn.company.com \
    user=branchuser \
    password=BranchPass \
    use-ipsec=yes \
    ipsec-secret=SharedSecret \
    profile=default-encryption \
    add-default-route=no \
    disabled=no

# PPTP Client  
/interface pptp-client add \
    name=pptp-to-office \
    connect-to=203.0.113.100 \
    user=remoteuser \
    password=RemotePass \
    profile=default-encryption \
    add-default-route=no \
    disabled=no

# OpenVPN Client
/interface ovpn-client add \
    name=ovpn-to-server \
    connect-to=203.0.113.100 \
    port=1194 \
    mode=ip \
    user=ovpnclient \
    password=OVPNclientpass \
    certificate=client-cert \
    ca=ca-cert \
    auth=sha1 \
    cipher=aes256 \
    disabled=no

# WireGuard Client (ROS 7.x)
/interface wireguard add name=wg-client

/interface wireguard peers add \
    interface=wg-client \
    public-key="SERVER_PUBLIC_KEY" \
    endpoint-address=203.0.113.100 \
    endpoint-port=51820 \
    allowed-address=0.0.0.0/0 \
    persistent-keepalive=25s

/ip address add address=10.0.0.2/24 interface=wg-client

# Script enable/disable VPN clients
:local toggleVPN do={
    :local ifaceName $1
    :local enable $2
    
    :local id [/interface find name=$ifaceName]
    :if ([:len $id] = 0) do={
        :put "VPN interface not found: $ifaceName"
        :return
    }
    
    :if ($enable) do={
        /interface enable $id
        :put "VPN enabled: $ifaceName"
    } else={
        /interface disable $id
        :put "VPN disabled: $ifaceName"
    }
}

[$toggleVPN "l2tp-to-hq" true]
```

---

## 34.7 Auto-reconnect Scripts

```routeros
# Auto-reconnect VPN ถ้า connection drop
/system script add name="vpn-watchdog" source="
    :local vpnIface \"l2tp-to-hq\"
    
    :local id [/interface find name=\$vpnIface]
    :if ([:len \$id] = 0) do={
        :put \"VPN interface not found\"
        :return
    }
    
    :local running [/interface get \$id running]
    :local disabled [/interface get \$id disabled]
    
    :if (!\$disabled && !\$running) do={
        :log warning (\"VPN disconnected: \" . \$vpnIface . \" - attempting reconnect\")
        
        # Disable and re-enable to force reconnect
        /interface disable \$id
        :delay 3s
        /interface enable \$id
        :delay 10s
        
        # Check if reconnected
        :if ([/interface get \$id running]) do={
            :log info (\"VPN reconnected: \" . \$vpnIface)
        } else={
            :log error (\"VPN reconnect failed: \" . \$vpnIface)
            /tool e-mail send to=\"admin@company.com\" \
                subject=\"VPN Reconnect Failed\" \
                body=(\"VPN interface \" . \$vpnIface . \" failed to reconnect\")
        }
    }
" comment="VPN auto-reconnect watchdog"

/system scheduler add \
    name="vpn-watchdog" \
    interval=2m \
    on-event="/system script run vpn-watchdog"
```

---

## 34.8 VPN Monitoring

```routeros
# Monitor all VPN connections
/system script add name="vpn-monitor" source="
    :local report \"=== VPN Status Report ===\\r\\n\"
    :set report (\$report . \"Time: \" . [/system clock get time] . \"\\r\\n\\r\\n\")
    
    # L2TP/PPTP/SSTP active sessions
    :local activePPP [/ppp active find]
    :set report (\$report . \"Active VPN Sessions: \" . [:len \$activePPP] . \"\\r\\n\")
    
    :foreach session in=\$activePPP do={
        :local user [/ppp active get \$session name]
        :local ip [/ppp active get \$session address]
        :local uptime [/ppp active get \$session uptime]
        :local service [/ppp active get \$session service]
        :set report (\$report . \"  \" . \$user . \" \" . \$ip . \" \" . \$service . \" up=\" . \$uptime . \"\\r\\n\")
    }
    
    # VPN Client status
    :set report (\$report . \"\\r\\nVPN Clients:\\r\\n\")
    :foreach iface in=[/interface l2tp-client find] do={
        :local name [/interface l2tp-client get \$iface name]
        :local running [/interface get [/interface find name=\$name] running]
        :local status \"DISCONNECTED\"
        :if (\$running) do={ :set status \"CONNECTED\" }
        :set report (\$report . \"  L2TP \" . \$name . \": \" . \$status . \"\\r\\n\")
    }
    
    :put \$report
" comment="VPN monitoring script"

/system scheduler add \
    name="vpn-monitor" \
    interval=30m \
    on-event="/system script run vpn-monitor"
```

---

## 34.9 Site-to-site VPN Automation

```routeros
# Site-to-Site VPN ด้วย L2TP/IPSec
# HQ Router (203.0.113.1)

# IPSec Peer สำหรับ Branch
/ip ipsec peer add \
    name=branch-office \
    address=203.0.113.100 \
    exchange-mode=ike2 \
    local-address=203.0.113.1 \
    passive=no \
    send-initial-contact=yes

/ip ipsec proposal add \
    name=s2s-proposal \
    auth-algorithms=sha256 \
    enc-algorithms=aes-256-cbc \
    lifetime=1h

/ip ipsec identity add \
    peer=branch-office \
    auth-method=pre-shared-key \
    secret=S2S-SharedKey-2024

/ip ipsec policy add \
    src-address=192.168.1.0/24 \
    dst-address=192.168.2.0/24 \
    proposal=s2s-proposal \
    action=encrypt \
    level=require \
    tunnel=yes \
    sa-src-address=203.0.113.1 \
    sa-dst-address=203.0.113.100

# Route สำหรับ branch network
/ip route add dst-address=192.168.2.0/24 gateway=192.168.1.254

# Monitor S2S VPN
:local monitorS2S do={
    :local peers [/ip ipsec active-peers find]
    :put "S2S VPN Status:"
    :foreach peer in=$peers do={
        :local remote [/ip ipsec active-peers get $peer remote-address]
        :local uptime [/ip ipsec active-peers get $peer uptime]
        :put "  Peer: $remote Uptime: $uptime"
    }
    :put "Total S2S tunnels: " . [:len $peers]
}

[$monitorS2S]
```

---

## Lab 34: Complete VPN Server

### Solution

```routeros
# Complete VPN Server Lab
# ========================
# Setup: L2TP/IPSec + WireGuard (ROS 7.x)

# --- L2TP/IPSec Server ---
/ip pool add name=vpn-pool ranges=192.168.100.1-192.168.100.50

/ppp profile add \
    name=vpn-profile \
    local-address=192.168.100.0 \
    remote-address=vpn-pool \
    dns-server=8.8.8.8,1.1.1.1 \
    use-compression=no \
    use-encryption=yes

/interface l2tp-server server set \
    enabled=yes \
    use-ipsec=required \
    ipsec-secret=VPNSharedKey2024 \
    default-profile=vpn-profile \
    max-sessions=50

# VPN Users
/ppp secret add name=remote-user1 password=RU@001 service=l2tp profile=vpn-profile
/ppp secret add name=remote-user2 password=RU@002 service=l2tp profile=vpn-profile

# --- WireGuard (ROS 7.x) ---
:do {
    /interface wireguard add name=wg0 listen-port=51820
    /ip address add address=10.99.0.1/24 interface=wg0
    
    # Add client peers
    /interface wireguard peers add \
        interface=wg0 \
        public-key="PASTE_CLIENT_PUBLIC_KEY_HERE" \
        allowed-address=10.99.0.2/32 \
        comment="WireGuard Client 1"
    
    :put "WireGuard configured"
} on-error={
    :put "WireGuard requires RouterOS 7.x"
}

# --- Firewall Rules ---
/ip firewall filter add chain=input protocol=tcp dst-port=1723 action=accept comment="PPTP"
/ip firewall filter add chain=input protocol=udp dst-port=1701 action=accept comment="L2TP"
/ip firewall filter add chain=input protocol=udp dst-port=500 action=accept comment="IKE"
/ip firewall filter add chain=input protocol=udp dst-port=4500 action=accept comment="IPSec"
/ip firewall filter add chain=input protocol=ipsec-esp action=accept comment="ESP"
/ip firewall filter add chain=input protocol=ipsec-ah action=accept comment="AH"
/ip firewall filter add chain=input protocol=gre action=accept comment="GRE"
/ip firewall filter add chain=input protocol=udp dst-port=51820 action=accept comment="WireGuard"

# NAT สำหรับ VPN clients ออก Internet
/ip firewall nat add \
    chain=srcnat \
    src-address=192.168.100.0/24 \
    out-interface=ether1 \
    action=masquerade \
    comment="NAT VPN clients"

# --- VPN Monitoring System ---
/system script add name="vpn-status" source="
    :put \"=== VPN Status ===\"
    :put \"Active sessions: \" . [:len [/ppp active find]]
    :foreach session in=[/ppp active find] do={
        :local user [/ppp active get \$session name]
        :local ip [/ppp active get \$session address]
        :local svc [/ppp active get \$session service]
        :put \"  \" . \$user . \" (\" . \$svc . \") - \" . \$ip
    }
" comment="Show VPN status"

# --- Auto-reconnect Watchdog ---
/system script add name="vpn-watchdog-full" source="
    :local vpnClients [/interface l2tp-client find disabled=no]
    :foreach vpn in=\$vpnClients do={
        :local name [/interface l2tp-client get \$vpn name]
        :local running [/interface get [/interface find name=\$name] running]
        :if (!\$running) do={
            :log warning (\"VPN client down: \" . \$name . \" - reconnecting\")
            /interface disable [/interface find name=\$name]
            :delay 3s
            /interface enable [/interface find name=\$name]
        }
    }
" comment="VPN client auto-reconnect"

/system scheduler add name="vpn-watchdog" interval=2m \
    on-event="/system script run vpn-watchdog-full"

:put "VPN Server configured!"
:put "- L2TP/IPSec server with 50 session limit"
:put "- WireGuard (ROS 7.x)"
:put "- Auto-reconnect monitoring"
:put "- 2 test users: remote-user1, remote-user2"
```

---

## Summary ของ Part 34

| VPN Type | Security | Speed | Use Case |
|----------|---------|-------|---------|
| PPTP | Very Low | Fast | Legacy only |
| L2TP/IPSec | High | Medium | Remote access |
| OpenVPN | High | Medium | Flexible |
| SSTP | High | Medium | Windows compat |
| WireGuard | Very High | Very Fast | Modern (ROS7.x) |

> **Recommendation:** ใช้ WireGuard (RouterOS 7.x) หรือ L2TP/IPSec สำหรับ production

> **Warning:** PPTP ไม่ควรใช้ใน production เพราะ MPPE encryption ถูก crack แล้ว

> **Tip:** IPSec shared secret ต้องมีความยาวและ complexity เพียงพอ (>20 characters)

---

[← Part 33: PPPoE Server](part-033-pppoe-server.md) | [Part 35: Bandwidth Management →](part-035-bandwidth-management.md)
