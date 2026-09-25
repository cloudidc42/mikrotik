# Part 29: DHCP Automation ใน RouterOS

## บทนำ

DHCP automation ช่วยให้เราสามารถทำงานอัตโนมัติเมื่อ clients เชื่อมต่อหรือตัดการเชื่อมต่อจาก network เช่น การ assign DNS record, การ log devices ใหม่, การส่ง notification และอื่นๆ

---

## 29.1 DHCP Event Scripts

### DHCP Server Event Handlers

RouterOS DHCP server รองรับ event scripts 3 ประเภท:

| Event | เกิดเมื่อ |
|-------|---------|
| `on-bound` | Client ได้รับ/ต่อ lease |
| `on-expired` | Lease หมดอายุ |
| `on-release` | Client คืน IP (DHCPRELEASE) |

### Built-in Variables ใน DHCP Scripts

| Variable | ค่า |
|---------|-----|
| `$leaseActualAddress` | IP ที่ assign ให้ |
| `$leaseServerName` | ชื่อ DHCP server |
| `$leaseClientId` | Client identifier |
| `$leaseMacAddress` | MAC address |
| `$leaseHostname` | Hostname ที่ client ส่งมา |
| `$leaseBound` | 1=bound, 0=unbound |
| `$address` | IP address |

---

## 29.2 Dynamic DNS from DHCP

### Setup DNS Records เมื่อ Client เชื่อมต่อ

```routeros
# DHCP on-bound script: สร้าง DNS record
:local dhcpBoundScript "
    :local hostname \$leaseHostname
    :local ip \$leaseActualAddress
    :local domain \"lan\"
    
    # ถ้าไม่มี hostname ใช้ MAC แทน
    :if ([:len \$hostname] = 0) do={
        :set hostname [:tostr \$leaseMacAddress]
        # แทนที่ : ด้วย - 
        :local cleanMac \"\"
        :for i from=0 to=([:len \$hostname] - 1) do={
            :local c [:pick \$hostname \$i (\$i + 1)]
            :if (\$c = \":\") do={ :set cleanMac (\$cleanMac . \"-\") } else={ :set cleanMac (\$cleanMac . \$c) }
        }
        :set hostname (\"device-\" . \$cleanMac)
    }
    
    :local fqdn (\$hostname . \".\" . \$domain)
    
    # ลบ DNS record เก่า (ถ้ามี)
    /ip dns static remove [find name=\$fqdn]
    
    # เพิ่ม DNS record ใหม่
    /ip dns static add name=\$fqdn address=\$ip ttl=5m
    :log info (\"DNS added: \" . \$fqdn . \" -> \" . \$ip)
"

# DHCP on-expired/on-release script: ลบ DNS record
:local dhcpUnboundScript "
    :local hostname \$leaseHostname
    :local ip \$leaseActualAddress
    :local domain \"lan\"
    
    :if ([:len \$hostname] = 0) do={ :return }
    
    :local fqdn (\$hostname . \".\" . \$domain)
    
    # ลบ DNS record
    /ip dns static remove [find name=\$fqdn]
    :log info (\"DNS removed: \" . \$fqdn)
"

# Apply scripts to DHCP server
/ip dhcp-server set [find name=dhcp1] \
    on-bound=$dhcpBoundScript \
    on-expired=$dhcpUnboundScript \
    on-release=$dhcpUnboundScript
```

---

## 29.3 Static Lease Automation

```routeros
# Auto-create static lease จาก existing lease
:local makeStaticLease do={
    :local mac $1
    :local ip $2
    :local hostname $3
    :local comment $4
    
    # ตรวจสอบว่า static lease มีอยู่แล้วหรือไม่
    :local existing [/ip dhcp-server lease find \
        mac-address=$mac static-only=yes]
    
    :if ([:len $existing] > 0) do={
        :put "Static lease already exists for $mac"
        :return false
    }
    
    # สร้าง static lease
    /ip dhcp-server lease add \
        mac-address=$mac \
        address=$ip \
        comment=$comment \
        server=dhcp1
    
    :put "Created static lease: $mac -> $ip ($hostname)"
    :return true
}

# Convert dynamic leases to static
:local convertToStatic do={
    :foreach lease in=[/ip dhcp-server lease find \
        dynamic=yes status=bound] do={
        
        :local mac [/ip dhcp-server lease get $lease mac-address]
        :local ip [/ip dhcp-server lease get $lease address]
        :local hostname [/ip dhcp-server lease get $lease host-name]
        :local comment ("Auto-static: " . $hostname)
        
        [$makeStaticLease $mac $ip $hostname $comment]
    }
}

# ตัวอย่างการใช้งาน
[$makeStaticLease "AA:BB:CC:DD:EE:FF" "192.168.1.50" "printer-main" "Main Printer"]
[$makeStaticLease "11:22:33:44:55:66" "192.168.1.51" "camera-01" "IP Camera 1"]

# Bulk import static leases จาก CSV
:local importStaticLeases do={
    :local csvData "AA:BB:CC:00:01;192.168.1.10;server1;Web Server\r\n" .
                   "AA:BB:CC:00:02;192.168.1.11;server2;DB Server\r\n" .
                   "AA:BB:CC:00:03;192.168.1.12;printer;Office Printer"
    
    :local lines [:toarray $csvData]
    :foreach line in=$lines do={
        :local parts [:toarray $line]
        :if ([:len $parts] >= 4) do={
            [$makeStaticLease ($parts->0) ($parts->1) ($parts->2) ($parts->3)]
        }
    }
}
```

---

## 29.4 DHCP Lease Reporting

```routeros
# สร้าง report ของ DHCP leases
:local generateDHCPReport do={
    :local report ""
    :local activeCount 0
    :local staticCount 0
    :local dynamicCount 0
    
    :set report "=== DHCP Lease Report ===\r\n"
    :set report ($report . "Date: " . [/system clock get date] . "\r\n\r\n")
    
    :set report ($report . "IP Address       MAC Address         Hostname          Type\r\n")
    :set report ($report . "------------------------------------------------\r\n")
    
    :foreach lease in=[/ip dhcp-server lease find] do={
        :local ip [/ip dhcp-server lease get $lease address]
        :local mac [/ip dhcp-server lease get $lease mac-address]
        :local hostname [/ip dhcp-server lease get $lease host-name]
        :local status [/ip dhcp-server lease get $lease status]
        :local dynamic [/ip dhcp-server lease get $lease dynamic]
        :local type "static"
        :if ($dynamic) do={ :set type "dynamic" }
        
        :if ($status = "bound") do={
            :set activeCount ($activeCount + 1)
            :set report ($report . $ip . "  " . $mac . "  " . $hostname . "  " . $type . "\r\n")
        }
        
        :if ($dynamic) do={ :set dynamicCount ($dynamicCount + 1) }
        :if (!$dynamic) do={ :set staticCount ($staticCount + 1) }
    }
    
    :set report ($report . "\r\n--- Summary ---\r\n")
    :set report ($report . "Active leases: " . $activeCount . "\r\n")
    :set report ($report . "Static leases: " . $staticCount . "\r\n")
    :set report ($report . "Dynamic leases: " . $dynamicCount . "\r\n")
    
    :return $report
}

:put [$generateDHCPReport]

# Export report ลงไฟล์
:local report [$generateDHCPReport]
/file remove [find name="dhcp-report.txt"]
/tool fetch url="data:," dst-path="dhcp-report.txt"
/file set [find name="dhcp-report.txt"] contents=$report

# ส่งรายงานทาง email ทุกวัน
/system scheduler add \
    name="dhcp-daily-report" \
    start-time=08:00:00 \
    interval=1d \
    on-event="
        :local report [$generateDHCPReport]
        /tool e-mail send to=\"admin@company.com\" \
            subject=\"Daily DHCP Report\" body=\$report
    "
```

---

## 29.5 Network Inventory from DHCP

```routeros
# สร้าง network inventory จาก DHCP data
:local buildInventory do={
    :local inventory {}
    
    :foreach lease in=[/ip dhcp-server lease find status=bound] do={
        :local ip [/ip dhcp-server lease get $lease address]
        :local mac [/ip dhcp-server lease get $lease mac-address]
        :local hostname [/ip dhcp-server lease get $lease host-name]
        :local expires [/ip dhcp-server lease get $lease expires-after]
        :local server [/ip dhcp-server lease get $lease server]
        
        # OUI lookup (first 3 octets of MAC = vendor)
        :local oui [:toupper [:pick $mac 0 8]]
        :local vendor "Unknown"
        
        # Simple OUI lookup table
        :if ($oui = "AA:BB:CC") do={ :set vendor "Cisco" }
        :if ($oui = "00:50:56") do={ :set vendor "VMware" }
        :if ($oui = "08:00:27") do={ :set vendor "VirtualBox" }
        :if ([:find $oui "DC:A6:32"] != -1) do={ :set vendor "Raspberry Pi" }
        :if ([:find $oui "B8:27:EB"] != -1) do={ :set vendor "Raspberry Pi" }
        
        :local entry {
            "ip"=$ip;
            "mac"=$mac;
            "hostname"=$hostname;
            "vendor"=$vendor;
            "server"=$server;
            "expires"=$expires
        }
        
        :set inventory ($inventory , $entry)
    }
    
    :return $inventory
}

:local inventory [$buildInventory]
:put "Network Inventory:"
:put "Total devices: " . [:len $inventory]
:foreach device in=$inventory do={
    :put ("  " . ($device->"ip") . " - " . ($device->"hostname") . \
          " [" . ($device->"vendor") . "]")
}
```

---

## 29.6 Notification on New Devices

```routeros
# สร้าง known devices list
:global KNOWN_DEVICES {}

# Load known devices จากไฟล์
:local loadKnownDevices do={
    :global KNOWN_DEVICES
    :local fileId [/file find name="known-devices.txt"]
    :if ([:len $fileId] > 0) do={
        :local content [/file get $fileId contents]
        :set KNOWN_DEVICES [:toarray $content]
    }
}

# Save known devices ลงไฟล์
:local saveKnownDevices do={
    :global KNOWN_DEVICES
    /file remove [find name="known-devices.txt"]
    /tool fetch url="data:," dst-path="known-devices.txt"
    /file set [find name="known-devices.txt"] contents=[:tostr $KNOWN_DEVICES]
}

# DHCP on-bound script ที่ detect new devices
:local newDeviceScript "
    :global KNOWN_DEVICES
    :local mac \$leaseMacAddress
    :local ip \$leaseActualAddress
    :local hostname \$leaseHostname
    
    # ตรวจสอบว่าเป็น device ใหม่
    :local isKnown false
    :foreach known in=\$KNOWN_DEVICES do={
        :if (\$known = \$mac) do={ :set isKnown true }
    }
    
    :if (!\$isKnown) do={
        # New device detected!
        :log warning (\"NEW DEVICE: MAC=\" . \$mac . \" IP=\" . \$ip . \" HOST=\" . \$hostname)
        
        # เพิ่มเข้า known devices
        :set KNOWN_DEVICES (\$KNOWN_DEVICES , \$mac)
        
        # ส่ง alert
        /tool e-mail send \
            to=\"admin@company.com\" \
            subject=\"New Device Detected!\" \
            body=(\"New device connected to network\\r\\n\" . \
                 \"MAC: \" . \$mac . \"\\r\\n\" . \
                 \"IP: \" . \$ip . \"\\r\\n\" . \
                 \"Hostname: \" . \$hostname . \"\\r\\n\" . \
                 \"Time: \" . [/system clock get time])
    }
"

# Apply ไปยัง DHCP server
/ip dhcp-server set [find name=dhcp1] on-bound=$newDeviceScript
```

---

## 29.7 DHCP Lease Cleanup

```routeros
# ลบ expired leases
:local cleanupExpiredLeases do={
    :local removed 0
    :foreach lease in=[/ip dhcp-server lease find status=expired dynamic=yes] do={
        :local ip [/ip dhcp-server lease get $lease address]
        :local mac [/ip dhcp-server lease get $lease mac-address]
        /ip dhcp-server lease remove $lease
        :set removed ($removed + 1)
        :log info ("Removed expired lease: " . $ip . " " . $mac)
    }
    :put "Removed $removed expired leases"
    :return $removed
}

# ลบ leases ที่ไม่ active นานเกิน X วัน
:local cleanupInactiveLeases do={
    :local maxInactiveDays $1
    :local removed 0
    
    :foreach lease in=[/ip dhcp-server lease find dynamic=yes] do={
        :local status [/ip dhcp-server lease get $lease status]
        :if ($status = "waiting") do={
            /ip dhcp-server lease remove $lease
            :set removed ($removed + 1)
        }
    }
    :put "Removed $removed inactive leases"
}

# Scheduled cleanup
/system scheduler add \
    name="dhcp-cleanup" \
    start-time=03:00:00 \
    interval=1d \
    on-event="/system script run dhcp-cleanup" \
    comment="Daily DHCP lease cleanup"

/system script add name="dhcp-cleanup" source="
    # ลบ expired dynamic leases
    :local removed 0
    :foreach lease in=[/ip dhcp-server lease find status=expired dynamic=yes] do={
        /ip dhcp-server lease remove \$lease
        :set removed (\$removed + 1)
    }
    :log info (\"DHCP cleanup: removed \" . \$removed . \" expired leases\")
"
```

---

## 29.8 Multiple DHCP Server Management

```routeros
# จัดการหลาย DHCP servers
:local listAllDHCPServers do={
    :foreach server in=[/ip dhcp-server find] do={
        :local name [/ip dhcp-server get $server name]
        :local iface [/ip dhcp-server get $server interface]
        :local pool [/ip dhcp-server get $server address-pool]
        :local disabled [/ip dhcp-server get $server disabled]
        :local status "enabled"
        :if ($disabled) do={ :set status "disabled" }
        
        # นับ active leases
        :local leaseCount [:len [/ip dhcp-server lease find \
            server=$name status=bound]]
        
        :put "$name on $iface (pool: $pool) [$status] - $leaseCount leases"
    }
}

[$listAllDHCPServers]

# สร้าง DHCP server ใหม่แบบ automated
:local createDHCPServer do={
    :local name $1
    :local iface $2
    :local network $3
    :local startIP $4
    :local endIP $5
    :local gateway $6
    :local dns $7
    
    # สร้าง IP pool
    :local poolName ($name . "-pool")
    /ip pool add name=$poolName ranges=($startIP . "-" . $endIP)
    
    # สร้าง DHCP network
    /ip dhcp-server network add \
        address=$network \
        gateway=$gateway \
        dns-server=$dns \
        comment=$name
    
    # สร้าง DHCP server
    /ip dhcp-server add \
        name=$name \
        interface=$iface \
        address-pool=$poolName \
        disabled=no
    
    :put "DHCP server created: $name on $iface"
}

# สร้าง DHCP servers สำหรับหลาย VLANs
[$createDHCPServer "dhcp-vlan10" "vlan10" "10.10.10.0/24" \
    "10.10.10.10" "10.10.10.200" "10.10.10.1" "8.8.8.8"]

[$createDHCPServer "dhcp-vlan20" "vlan20" "10.20.0.0/24" \
    "10.20.0.10" "10.20.0.200" "10.20.0.1" "8.8.8.8"]

[$createDHCPServer "dhcp-vlan30" "vlan30" "10.30.0.0/24" \
    "10.30.0.10" "10.30.0.200" "10.30.0.1" "1.1.1.1,8.8.8.8"]
```

---

## 29.9 Integration with External Systems

```routeros
# ส่งข้อมูล DHCP lease ไปยัง external system
:local sendLeaseToAPI do={
    :local ip $1
    :local mac $2
    :local hostname $3
    :local action $4  # add/remove
    
    :local jsonData ("{\"ip\":\"" . $ip . "\"," . \
                    "\"mac\":\"" . $mac . "\"," . \
                    "\"hostname\":\"" . $hostname . "\"," . \
                    "\"action\":\"" . $action . "\"," . \
                    "\"router\":\"" . [/system identity get name] . "\"}")
    
    :do {
        /tool fetch \
            url="http://inventory.company.com/api/dhcp" \
            http-method=post \
            http-header-field="Content-Type: application/json" \
            http-data=$jsonData \
            dst-path="api-response.txt"
        :log info ("Lease sent to API: " . $ip . " " . $action)
    } on-error={
        :log warning ("Failed to send lease to API: " . $ip)
    }
}

# DHCP script ที่ integrate กับ external systems
:local integratedBoundScript "
    :local ip \$leaseActualAddress
    :local mac \$leaseMacAddress
    :local hostname \$leaseHostname
    
    # 1. สร้าง DNS record
    /ip dns static remove [find name=(\$hostname . \".lan\")]
    /ip dns static add name=(\$hostname . \".lan\") address=\$ip ttl=5m
    
    # 2. เพิ่ม address tag สำหรับ QoS
    /ip firewall address-list remove [find list=known-devices address=\$ip]
    /ip firewall address-list add list=known-devices address=\$ip comment=\$hostname
    
    # 3. Log ลง syslog
    /log info (\"DHCP bound: \" . \$ip . \" MAC=\" . \$mac . \" HOST=\" . \$hostname)
    
    # 4. ส่งไปยัง monitoring API (ถ้าสามารถเชื่อมต่อได้)
    :do {
        /tool fetch url=\"http://192.168.1.100/api/lease\" \
            http-method=post \
            http-data=(\"ip=\" . \$ip . \"&mac=\" . \$mac . \"&action=add\") \
            dst-path=\"api.tmp\"
    } on-error={ }
"
```

---

## Lab 29: Auto-DNS from DHCP Script

### โจทย์
สร้างระบบ Dynamic DNS ที่:
1. สร้าง DNS record อัตโนมัติเมื่อ client เชื่อมต่อ
2. ลบ DNS record เมื่อ lease หมดอายุ
3. จัดการ hostname conflicts
4. Log ทุกการเปลี่ยนแปลง
5. Report ประจำวัน

### Solution

```routeros
# Auto-DNS from DHCP Lab
# ======================

# --- DNS Domain Configuration ---
:global DNS_DOMAIN "home.lan"
:global DNS_TTL "5m"
:global DNS_LOG_FILE "dhcp-dns.log"

# --- DHCP on-bound script ---
/system script add name="dhcp-dns-bound" source="
    :global DNS_DOMAIN
    :global DNS_TTL
    
    :local ip \$leaseActualAddress
    :local mac \$leaseMacAddress
    :local hostname \$leaseHostname
    
    # Generate hostname จาก MAC ถ้าไม่มี hostname
    :if ([:len \$hostname] = 0 || \$hostname = \"(unknown)\") do={
        :local cleanMac \"\"
        :for i from=0 to=([:len [:tostr \$mac]] - 1) do={
            :local c [:pick [:tostr \$mac] \$i (\$i + 1)]
            :if (\$c != \":\") do={ :set cleanMac (\$cleanMac . \$c) }
        }
        :set hostname (\"device-\" . [:tolower [:pick \$cleanMac 6 12]])
    }
    
    # Sanitize hostname (ลบ characters ที่ไม่ถูกต้อง)
    :local safeHostname \"\"
    :for i from=0 to=([:len \$hostname] - 1) do={
        :local c [:pick \$hostname \$i (\$i + 1)]
        :if ((\$c >= \"a\" && \$c <= \"z\") || (\$c >= \"A\" && \$c <= \"Z\") || \
            (\$c >= \"0\" && \$c <= \"9\") || \$c = \"-\") do={
            :set safeHostname (\$safeHostname . [:tolower \$c])
        }
    }
    
    :if ([:len \$safeHostname] = 0) do={
        :set safeHostname (\"host-\" . [:pick [:tostr \$ip] 0 3])
    }
    
    :local fqdn (\$safeHostname . \".\" . \$DNS_DOMAIN)
    
    # ตรวจสอบ IP conflict (ถ้า hostname เดิมมี IP อื่น)
    :local existingEntry [/ip dns static find name=\$fqdn]
    :if ([:len \$existingEntry] > 0) do={
        :local existingIP [/ip dns static get \$existingEntry address]
        :if (\$existingIP != \$ip) do={
            # IP changed - update
            /ip dns static remove \$existingEntry
            :log info (\"DNS update: \" . \$fqdn . \" \" . \$existingIP . \" -> \" . \$ip)
        }
    }
    
    # ลบ DNS record ที่มี IP เดียวกัน (จาก old hostname)
    /ip dns static remove [find address=\$ip]
    
    # สร้าง DNS record ใหม่
    /ip dns static add \
        name=\$fqdn \
        address=\$ip \
        ttl=\$DNS_TTL \
        comment=(\"DHCP \" . \$mac)
    
    :log info (\"DNS bound: \" . \$fqdn . \" -> \" . \$ip . \" (\" . \$mac . \")\")
" comment="DHCP on-bound: create DNS record"

# --- DHCP on-expired/on-release script ---
/system script add name="dhcp-dns-unbound" source="
    :global DNS_DOMAIN
    
    :local ip \$leaseActualAddress
    :local mac \$leaseMacAddress
    :local hostname \$leaseHostname
    
    # ลบ DNS records ที่ตรงกับ IP นี้
    :local removed 0
    :foreach dns in=[/ip dns static find address=\$ip] do={
        :local dnsName [/ip dns static get \$dns name]
        /ip dns static remove \$dns
        :set removed (\$removed + 1)
        :log info (\"DNS unbound: \" . \$dnsName . \" \" . \$ip)
    }
    
    :if (\$removed = 0) do={
        :log info (\"DNS unbound: no record for \" . \$ip)
    }
" comment="DHCP on-expired/released: remove DNS record"

# --- Apply scripts to DHCP server ---
/ip dhcp-server set [find name=dhcp1] \
    on-bound="/system script run dhcp-dns-bound" \
    on-expired="/system script run dhcp-dns-unbound" \
    on-release="/system script run dhcp-dns-unbound"

# --- DNS Report Script ---
/system script add name="dhcp-dns-report" source="
    :global DNS_DOMAIN
    
    :local report \"=== DHCP-DNS Report ===\\r\\n\"
    :set report (\$report . \"Domain: \" . \$DNS_DOMAIN . \"\\r\\n\")
    :set report (\$report . \"Date: \" . [/system clock get date] . \"\\r\\n\\r\\n\")
    
    :local count 0
    :set report (\$report . \"FQDN                          IP Address\\r\\n\")
    :set report (\$report . \"-------------------------------------------\\r\\n\")
    
    :foreach dns in=[/ip dns static find where comment~\"DHCP\"] do={
        :local name [/ip dns static get \$dns name]
        :local ip [/ip dns static get \$dns address]
        :set report (\$report . \$name . \"  \" . \$ip . \"\\r\\n\")
        :set count (\$count + 1)
    }
    
    :set report (\$report . \"\\r\\nTotal DNS records: \" . \$count)
    
    :put \$report
    
    /tool e-mail send \
        to=\"admin@company.com\" \
        subject=\"Daily DHCP-DNS Report\" \
        body=\$report
" comment="Daily DHCP-DNS report"

/system scheduler add \
    name="dhcp-dns-report" \
    start-time=09:00:00 \
    interval=1d \
    on-event="/system script run dhcp-dns-report"

:put "DHCP Auto-DNS system configured!"

# --- Test ---
# จำลอง bound event
:global leaseActualAddress "192.168.1.100"
:global leaseMacAddress "AA:BB:CC:DD:EE:FF"
:global leaseHostname "my-laptop"
:global leaseBound 1
:global leaseServerName "dhcp1"

/system script run dhcp-dns-bound
/ip dns static print where address=192.168.1.100
```

---

## Summary ของ Part 29

| Event | ตัวแปร | ใช้เมื่อ |
|-------|--------|---------|
| on-bound | leaseActualAddress, leaseMacAddress, leaseHostname | สร้าง DNS, log device |
| on-expired | leaseActualAddress, leaseMacAddress | ลบ DNS, cleanup |
| on-release | leaseActualAddress, leaseMacAddress | เหมือน expired |

> **Tip:** ใช้ `$leaseHostname` แต่ระวัง hostname อาจว่างหรือ invalid ต้อง sanitize เสมอ

> **Warning:** DHCP scripts รันทุกครั้งที่ client ต่อ ต้องมี error handling ดีพอ เพราะถ้า script error จะ affect lease process

> **Best Practice:** Log ทุก event เพื่อ audit trail และ troubleshooting

---

[← Part 28: Interface Scripts](part-028-interface-scripts.md) | [Part 30: Firewall Automation →](part-030-firewall-automation.md)
