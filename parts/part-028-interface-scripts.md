# Part 28: Interface Scripts ใน RouterOS

## บทนำ

Interface management ผ่าน scripting ช่วยให้เราสามารถ automate การจัดการ network interfaces ได้อย่างมีประสิทธิภาพ ตั้งแต่การ monitor status, enable/disable interfaces, จนถึงการ configure ต่างๆ ผ่าน script

---

## 28.1 Getting Interface List

### ดูรายการ Interfaces ทั้งหมด

```routeros
# แสดง interfaces ทั้งหมด
/interface print

# แสดงเฉพาะ Ethernet
/interface print where type=ether

# แสดงเฉพาะ Wireless
/interface print where type=wlan

# แสดงเฉพาะที่ enabled
/interface print where disabled=no

# แสดงเฉพาะที่ running
/interface print where running=yes

# ดึงรายการ interface names
:foreach iface in=[/interface find] do={
    :local name [/interface get $iface name]
    :local type [/interface get $iface type]
    :put "$name ($type)"
}

# ดึงเฉพาะ Ethernet interfaces
:local ethIfaces {}
:foreach iface in=[/interface find type=ether] do={
    :set ethIfaces ($ethIfaces , [/interface get $iface name])
}
:put "Ethernet interfaces: $ethIfaces"
```

### Query Interface Properties

```routeros
# ดู properties ของ interface
:local ifaceName "ether1"
:local id [/interface find name=$ifaceName]

:if ([:len $id] > 0) do={
    :put "Name: " . [/interface get $id name]
    :put "Type: " . [/interface get $id type]
    :put "MTU: " . [/interface get $id mtu]
    :put "MAC: " . [/interface get $id mac-address]
    :put "Running: " . [/interface get $id running]
    :put "Disabled: " . [/interface get $id disabled]
    :put "Comment: " . [/interface get $id comment]
} else={
    :put "Interface $ifaceName not found"
}

# ดู properties ทุก interface
/interface print columns=name,type,mac-address,mtu,running
```

---

## 28.2 Interface Status Monitoring

### Basic Status Check

```routeros
# ตรวจสอบ interface ว่า up หรือ down
:local checkInterface do={
    :local ifaceName $1
    :local id [/interface find name=$ifaceName]
    
    :if ([:len $id] = 0) do={
        :put "Interface not found: $ifaceName"
        :return "not-found"
    }
    
    :local running [/interface get $id running]
    :local disabled [/interface get $id disabled]
    
    :if ($disabled) do={ :return "disabled" }
    :if ($running) do={ :return "up" }
    :return "down"
}

:local status [$checkInterface "ether1"]
:put "ether1 status: $status"

# Monitor all interfaces
:foreach iface in=[/interface find disabled=no] do={
    :local name [/interface get $iface name]
    :local running [/interface get $iface running]
    :local status "DOWN"
    :if ($running) do={ :set status "UP" }
    :put "$name: $status"
}
```

### Interface Up/Down History

```routeros
# ติดตาม interface state changes
:global ifaceLastState {}
:global ifaceChangeLog {}

:local monitorInterfaces do={
    :global ifaceLastState
    :global ifaceChangeLog
    
    :foreach iface in=[/interface find disabled=no] do={
        :local name [/interface get $iface name]
        :local running [/interface get $iface running]
        :local currentState "down"
        :if ($running) do={ :set currentState "up" }
        
        # Check if state changed
        :local prevState ""
        :foreach item in=$ifaceLastState do={
            :if (($item->0) = $name) do={
                :set prevState ($item->1)
            }
        }
        
        :if ($prevState != $currentState && [:len $prevState] > 0) do={
            :local changeEntry {$name; $prevState; $currentState; [/system clock get time]}
            :set ifaceChangeLog ($ifaceChangeLog , $changeEntry)
            :log warning ("Interface " . $name . " changed from " . $prevState . " to " . $currentState)
        }
        
        # Update last state
        :local newState {}
        :foreach item in=$ifaceLastState do={
            :if (($item->0) != $name) do={
                :set newState ($newState , $item)
            }
        }
        :set newState ($newState , {$name; $currentState})
        :set ifaceLastState $newState
    }
}

# รัน monitor ทุก 1 นาที
/system scheduler add \
    name="interface-monitor" \
    interval=1m \
    on-event="/system script run monitor-interfaces"
```

---

## 28.3 Enabling/Disabling Interfaces

```routeros
# Enable interface
/interface enable ether2

# Disable interface
/interface disable ether2

# Enable/Disable ด้วย script
:local toggleInterface do={
    :local ifaceName $1
    :local action $2  # enable/disable
    
    :local id [/interface find name=$ifaceName]
    :if ([:len $id] = 0) do={
        :put "Interface not found: $ifaceName"
        :return false
    }
    
    :if ($action = "enable") do={
        /interface enable $id
        :put "Enabled: $ifaceName"
        :log info ("Interface enabled: " . $ifaceName)
    }
    :if ($action = "disable") do={
        /interface disable $id
        :put "Disabled: $ifaceName"
        :log info ("Interface disabled: " . $ifaceName)
    }
    :return true
}

[$toggleInterface "ether2" "disable"]
[$toggleInterface "ether2" "enable"]

# Disable all unused interfaces
:foreach iface in=[/interface find running=no disabled=no type=ether] do={
    :local name [/interface get $iface name]
    # ยกเว้น ether1 (uplink)
    :if ($name != "ether1") do={
        /interface disable $iface
        :put "Disabled unused: $name"
    }
}

# Scheduled port lockdown ช่วง off-hours
/system script add name="lockdown-guest-ports" source="
    :local ports {\"ether5\"; \"ether6\"; \"ether7\"; \"ether8\"}
    :local hour [:toint [:pick [:tostr [/system clock get time]] 0 2]]
    
    :if (\$hour >= 22 || \$hour < 6) do={
        # Off-hours: disable guest ports
        :foreach port in=\$ports do={
            :local id [/interface find name=\$port]
            :if ([:len \$id] > 0 && ![/interface get \$id disabled]) do={
                /interface disable \$id
                :log info (\"Off-hours lockdown: \" . \$port)
            }
        }
    } else={
        # Business hours: enable guest ports
        :foreach port in=\$ports do={
            :local id [/interface find name=\$port]
            :if ([:len \$id] > 0 && [/interface get \$id disabled]) do={
                /interface enable \$id
                :log info (\"Business hours unlock: \" . \$port)
            }
        }
    }
"

/system scheduler add \
    name="port-lockdown" \
    interval=1h \
    on-event="/system script run lockdown-guest-ports"
```

---

## 28.4 Configuring Interfaces via Script

```routeros
# ตั้งค่า interface comment
/interface set ether1 comment="WAN - ISP 1"
/interface set ether2 comment="LAN - Office"

# ตั้งค่า MTU
/interface set ether1 mtu=1500

# Script configure multiple interfaces
:local configureInterface do={
    :local name $1
    :local ip $2
    :local comment $3
    :local mtu $4
    
    :if ([:len $mtu] = 0) do={ :set mtu 1500 }
    
    :local id [/interface find name=$name]
    :if ([:len $id] = 0) do={
        :put "Interface not found: $name"
        :return
    }
    
    # Set interface properties
    /interface set $id comment=$comment mtu=$mtu
    
    # Add IP address (remove existing first)
    /ip address remove [find interface=$name]
    /ip address add address=$ip interface=$name comment=$comment
    
    :put "Configured: $name ($ip)"
}

[$configureInterface "ether1" "203.0.113.1/30" "WAN - ISP1" "1500"]
[$configureInterface "ether2" "192.168.1.1/24" "LAN - Main" "1500"]
[$configureInterface "ether3" "192.168.2.1/24" "LAN - Guest" "1500"]

# Bulk rename interfaces
:local renameInterfaces do={
    :local mappings {
        {"ether1"; "WAN1"};
        {"ether2"; "LAN-MAIN"};
        {"ether3"; "LAN-GUEST"}
    }
    
    :foreach mapping in=$mappings do={
        :local oldName ($mapping->0)
        :local newName ($mapping->1)
        :local id [/interface find name=$oldName]
        :if ([:len $id] > 0) do={
            /interface set $id name=$newName
            :put "Renamed: $oldName -> $newName"
        }
    }
}
```

---

## 28.5 Interface Traffic Statistics

```routeros
# ดู traffic statistics
/interface monitor-traffic ether1 once

# ดู interface stats ผ่าน script
:local getIfaceStats do={
    :local ifaceName $1
    :local id [/interface find name=$ifaceName]
    
    :if ([:len $id] = 0) do={
        :put "Interface not found"
        :return
    }
    
    :local rxBytes [/interface get $id rx-byte]
    :local txBytes [/interface get $id tx-byte]
    :local rxPackets [/interface get $id rx-packet]
    :local txPackets [/interface get $id tx-packet]
    :local rxErrors [/interface get $id rx-error]
    :local txErrors [/interface get $id tx-error]
    :local rxDrops [/interface get $id rx-drop]
    :local txDrops [/interface get $id tx-drop]
    
    :put "=== $ifaceName Statistics ==="
    :put "RX: $rxBytes bytes ($rxPackets pkts)"
    :put "TX: $txBytes bytes ($txPackets pkts)"
    :put "RX Errors: $rxErrors, Drops: $rxDrops"
    :put "TX Errors: $txErrors, Drops: $txDrops"
}

[$getIfaceStats "ether1"]

# Collect stats ทุกชั่วโมง
:global ifaceStatsHistory {}

:local collectStats do={
    :global ifaceStatsHistory
    :local snapshot {}
    :local timestamp [/system clock get time]
    
    :foreach iface in=[/interface find disabled=no] do={
        :local name [/interface get $iface name]
        :local rxBytes [/interface get $iface rx-byte]
        :local txBytes [/interface get $iface tx-byte]
        :set snapshot ($snapshot , {$name; $rxBytes; $txBytes; $timestamp})
    }
    
    :set ifaceStatsHistory ($ifaceStatsHistory , $snapshot)
    # Keep only last 24 snapshots (24 hours)
    :if ([:len $ifaceStatsHistory] > 24) do={
        :set ifaceStatsHistory [:pick $ifaceStatsHistory 1 [:len $ifaceStatsHistory]]
    }
}

# Calculate throughput between snapshots
:local calcThroughput do={
    :global ifaceStatsHistory
    :if ([:len $ifaceStatsHistory] < 2) do={
        :put "Not enough data"
        :return
    }
    
    :local last ($ifaceStatsHistory->([:len $ifaceStatsHistory]-1))
    :local prev ($ifaceStatsHistory->([:len $ifaceStatsHistory]-2))
    
    :foreach lastIface in=$last do={
        :local name ($lastIface->0)
        :local lastRx ($lastIface->1)
        :local prevRx 0
        
        :foreach prevIface in=$prev do={
            :if (($prevIface->0) = $name) do={
                :set prevRx ($prevIface->1)
            }
        }
        
        :local rxDiff ($lastRx - $prevRx)
        :put "$name: RX $rxDiff bytes/hour"
    }
}
```

---

## 28.6 MAC Address Manipulation

```routeros
# ดู MAC address
:foreach iface in=[/interface find type=ether] do={
    :local name [/interface get $iface name]
    :local mac [/interface get $iface mac-address]
    :put "$name: $mac"
}

# เปลี่ยน MAC address
/interface set ether1 mac-address=AA:BB:CC:DD:EE:FF

# เปลี่ยน MAC address ด้วย script
:local changeMac do={
    :local ifaceName $1
    :local newMac $2
    
    # Validate MAC format (basic)
    :if ([:len $newMac] != 17) do={
        :put "Invalid MAC format: $newMac"
        :return false
    }
    
    :local id [/interface find name=$ifaceName]
    :if ([:len $id] = 0) do={
        :put "Interface not found: $ifaceName"
        :return false
    }
    
    :local oldMac [/interface get $id mac-address]
    /interface set $id mac-address=$newMac
    :put "Changed MAC on $ifaceName: $oldMac -> $newMac"
    :return true
}

# Reset MAC to factory default
:local resetMac do={
    :local ifaceName $1
    :local id [/interface find name=$ifaceName]
    :if ([:len $id] > 0) do={
        /interface set $id mac-address=[/interface get $id orig-mac-address]
        :put "Reset MAC on $ifaceName to factory default"
    }
}

# Clone MAC from interface
:local cloneMac do={
    :local srcIface $1
    :local dstIface $2
    :local srcId [/interface find name=$srcIface]
    :local dstId [/interface find name=$dstIface]
    
    :if ([:len $srcId] > 0 && [:len $dstId] > 0) do={
        :local mac [/interface get $srcId mac-address]
        /interface set $dstId mac-address=$mac
        :put "Cloned MAC from $srcIface to $dstIface: $mac"
    }
}
```

---

## 28.7 MTU Settings

```routeros
# ดู MTU settings
/interface print columns=name,mtu,actual-mtu

# ตั้ง MTU
/interface set ether1 mtu=1500

# ตั้ง MTU สำหรับ Jumbo frames
/interface set ether1 mtu=9000

# Script ตั้ง MTU ทุก interface
:local setAllMTU do={
    :local mtu $1
    :foreach iface in=[/interface find type=ether] do={
        :local name [/interface get $iface name]
        /interface set $iface mtu=$mtu
        :put "Set MTU=$mtu on $iface"
    }
}

[$setAllMTU 1500]

# Detect optimal MTU (MTU discovery)
:local discoverMTU do={
    :local target $1
    :local maxMTU 1500
    :local minMTU 576
    :local testMTU $maxMTU
    
    :while ($testMTU >= $minMTU) do={
        :do {
            /tool ping $target size=$testMTU count=3 do-not-fragment=yes
            :put "MTU $testMTU: OK"
            :return $testMTU
        } on-error={
            :set testMTU ($testMTU - 10)
        }
    }
    :return $minMTU
}
```

---

## 28.8 VLAN Management via Script

```routeros
# สร้าง VLAN interfaces
/interface vlan add name=vlan10 vlan-id=10 interface=ether2 comment="Management"
/interface vlan add name=vlan20 vlan-id=20 interface=ether2 comment="Users"
/interface vlan add name=vlan30 vlan-id=30 interface=ether2 comment="Servers"

# Script สร้าง VLANs จาก config
:local createVLANs do={
    :local vlans $1
    :local baseIface $2
    
    :foreach vlan in=$vlans do={
        :local vid ($vlan->0)
        :local name ($vlan->1)
        :local ip ($vlan->2)
        :local comment ($vlan->3)
        :local ifaceName ("vlan" . $vid)
        
        # Create VLAN interface
        :do {
            /interface vlan add \
                name=$ifaceName \
                vlan-id=$vid \
                interface=$baseIface \
                comment=$comment
            :put "Created VLAN: $ifaceName (id=$vid)"
        } on-error={
            :put "VLAN $ifaceName already exists"
        }
        
        # Add IP address
        :if ([:len $ip] > 0) do={
            /ip address add address=$ip interface=$ifaceName comment=$comment
            :put "Added IP $ip to $ifaceName"
        }
    }
}

:local vlanConfig {
    {10; "vlan10"; "10.10.10.1/24"; "Management VLAN"};
    {20; "vlan20"; "10.20.0.1/24"; "Users VLAN"};
    {30; "vlan30"; "10.30.0.1/24"; "Servers VLAN"};
    {40; "vlan40"; "10.40.0.1/24"; "IoT VLAN"}
}

[$createVLANs $vlanConfig "ether2"]

# ลบ VLANs
:local removeVLANs do={
    :local prefix $1
    :foreach iface in=[/interface vlan find] do={
        :local name [/interface vlan get $iface name]
        :if ([:find $name $prefix] = 0) do={
            /ip address remove [find interface=$name]
            /interface vlan remove $iface
            :put "Removed VLAN: $name"
        }
    }
}
```

---

## 28.9 Wireless Management via Script

```routeros
# ดู wireless interfaces
/interface wireless print

# ตั้งค่า wireless
/interface wireless set wlan1 \
    ssid="CompanyWiFi" \
    mode=ap \
    frequency=2462 \
    channel-width=20mhz \
    security-profile=company-security

# Script จัดการ wireless
:local configureWireless do={
    :local ifaceName $1
    :local ssid $2
    :local secProfile $3
    :local band $4
    :local channel $5
    
    :if ([:len $band] = 0) do={ :set band "2ghz-b/g/n" }
    
    :local id [/interface wireless find name=$ifaceName]
    :if ([:len $id] = 0) do={
        :put "Wireless interface not found: $ifaceName"
        :return
    }
    
    /interface wireless set $id \
        ssid=$ssid \
        security-profile=$secProfile \
        band=$band \
        mode=ap
    
    :if ([:len $channel] > 0) do={
        /interface wireless set $id frequency=$channel
    }
    
    :put "Configured wireless: $ifaceName (SSID: $ssid)"
}

# Disable WiFi ช่วง off-hours
/system script add name="wifi-schedule" source="
    :local hour [:toint [:pick [:tostr [/system clock get time]] 0 2]]
    
    :if (\$hour >= 22 || \$hour < 7) do={
        # Night: disable all wireless
        :foreach w in=[/interface wireless find disabled=no] do={
            /interface wireless disable \$w
            :log info (\"WiFi off-hours disabled: \" . [/interface wireless get \$w name])
        }
    } else={
        # Day: enable all wireless
        :foreach w in=[/interface wireless find disabled=yes] do={
            /interface wireless enable \$w
            :log info (\"WiFi enabled: \" . [/interface wireless get \$w name])
        }
    }
"

/system scheduler add \
    name="wifi-schedule" \
    interval=1h \
    on-event="/system script run wifi-schedule"

# Wireless client count monitoring
:local getWirelessClients do={
    :local total 0
    :foreach iface in=[/interface wireless find] do={
        :local name [/interface wireless get $iface name]
        :local clients [:len [/interface wireless registration-table find \
            interface=$name]]
        :put "$name: $clients clients"
        :set total ($total + $clients)
    }
    :put "Total wireless clients: $total"
}

[$getWirelessClients]
```

---

## Lab 28: Interface Monitoring Script

### โจทย์
สร้าง comprehensive interface monitoring script ที่:
1. Monitor ทุก interfaces
2. Alert เมื่อ interface down
3. Log statistics ทุกชั่วโมง
4. ส่ง daily report
5. Auto-recover (attempt restart)

### Solution

```routeros
# Interface Monitoring Script Lab
# =================================

# --- Configuration ---
:global IFACE_ALERT_EMAIL "admin@company.com"
:global IFACE_CRITICAL {"ether1"; "ether2"}  # Critical interfaces
:global IFACE_IGNORE {"lo"; "bridge1"}       # Interfaces to ignore
:global IFACE_LAST_STATE {}
:global IFACE_DOWN_SINCE {}

# --- Functions ---

# Check interface in ignore list
:local isIgnored do={
    :global IFACE_IGNORE
    :local name $1
    :foreach ig in=$IFACE_IGNORE do={
        :if ($ig = $name) do={ :return true }
    }
    :return false
}

# Check if critical interface
:local isCritical do={
    :global IFACE_CRITICAL
    :local name $1
    :foreach crit in=$IFACE_CRITICAL do={
        :if ($crit = $name) do={ :return true }
    }
    :return false
}

# Send alert
:local sendIfaceAlert do={
    :global IFACE_ALERT_EMAIL
    :local ifaceName $1
    :local status $2  # up/down
    :local critical $3
    
    :local router [/system identity get name]
    :local time [/system clock get time]
    :local severity "WARNING"
    :if ($critical) do={ :set severity "CRITICAL" }
    
    :local subject ("[$severity] Interface " . $status . ": " . $ifaceName . " on " . $router)
    :local body ("Router: " . $router . "\r\n" . \
                "Interface: " . $ifaceName . "\r\n" . \
                "Status: " . $status . "\r\n" . \
                "Time: " . $time . "\r\n" . \
                "Severity: " . $severity)
    
    :do {
        /tool e-mail send to=$IFACE_ALERT_EMAIL subject=$subject body=$body
    } on-error={
        :log warning "Could not send interface alert email"
    }
}

# Main monitoring function
/system script add name="iface-monitor" source="
    :global IFACE_LAST_STATE
    :global IFACE_DOWN_SINCE
    :global IFACE_IGNORE
    
    :foreach iface in=[/interface find] do={
        :local name [/interface get \$iface name]
        :local disabled [/interface get \$iface disabled]
        
        # Skip ignored and disabled
        :if (\$disabled) do={ :goto next }
        
        :local ignored false
        :foreach ig in=\$IFACE_IGNORE do={
            :if (\$ig = \$name) do={ :set ignored true }
        }
        :if (\$ignored) do={ :goto next }
        
        :local running [/interface get \$iface running]
        :local currentState \"down\"
        :if (\$running) do={ :set currentState \"up\" }
        
        # Find previous state
        :local prevState \"up\"  # Default assume was up
        :foreach item in=\$IFACE_LAST_STATE do={
            :if ((\$item->0) = \$name) do={
                :set prevState (\$item->1)
            }
        }
        
        # State changed?
        :if (\$prevState != \$currentState) do={
            :log warning (\"Interface state change: \" . \$name . \" \" . \$prevState . \" -> \" . \$currentState)
            
            :if (\$currentState = \"down\") do={
                # Interface went DOWN
                :local critical false
                :foreach crit in=\$IFACE_CRITICAL do={
                    :if (\$crit = \$name) do={ :set critical true }
                }
                
                # Try to recover
                :delay 5s
                :if (![/interface get \$iface running]) do={
                    # Still down after 5s
                    :if (\$critical) do={
                        :log critical (\"CRITICAL Interface DOWN: \" . \$name)
                        # Send critical alert
                    } else={
                        :log error (\"Interface DOWN: \" . \$name)
                        # Send warning alert
                    }
                } else={
                    :log info (\"Interface recovered quickly: \" . \$name)
                }
            } else={
                # Interface came UP
                :log info (\"Interface UP: \" . \$name)
            }
        }
        
        # Update last state
        :local newState {}
        :foreach item in=\$IFACE_LAST_STATE do={
            :if ((\$item->0) != \$name) do={
                :set newState (\$newState , \$item)
            }
        }
        :set newState (\$newState , {\$name; \$currentState})
        :set IFACE_LAST_STATE \$newState
        
        :label next
    }
" comment="Interface monitoring script"

# Daily report script
/system script add name="iface-daily-report" source="
    :global IFACE_ALERT_EMAIL
    :local router [/system identity get name]
    :local date [/system clock get date]
    
    :local report (\"Daily Interface Report\\r\\n\")
    :set report (\$report . \"Router: \" . \$router . \"\\r\\n\")
    :set report (\$report . \"Date: \" . \$date . \"\\r\\n\\r\\n\")
    :set report (\$report . \"Interface Status:\\r\\n\")
    
    :foreach iface in=[/interface find disabled=no] do={
        :local name [/interface get \$iface name]
        :local running [/interface get \$iface running]
        :local rxBytes [/interface get \$iface rx-byte]
        :local txBytes [/interface get \$iface tx-byte]
        :local status \"DOWN\"
        :if (\$running) do={ :set status \"UP\" }
        
        :set report (\$report . \$name . \": \" . \$status . \
                    \" RX=\" . \$rxBytes . \" TX=\" . \$txBytes . \"\\r\\n\")
    }
    
    /tool e-mail send to=\$IFACE_ALERT_EMAIL \
        subject=(\"[Report] Daily Interface Status - \" . \$router) \
        body=\$report
    :log info \"Daily interface report sent\"
" comment="Daily interface status report"

# Setup schedulers
/system scheduler add \
    name="iface-monitor" \
    interval=1m \
    on-event="/system script run iface-monitor" \
    comment="Interface monitoring every minute"

/system scheduler add \
    name="iface-daily-report" \
    start-time=08:00:00 \
    interval=1d \
    on-event="/system script run iface-daily-report" \
    comment="Daily interface report at 8AM"

:put "Interface monitoring system configured!"
```

---

## Summary ของ Part 28

| Operation | Command | หมายเหตุ |
|-----------|---------|---------|
| List interfaces | `/interface find` | |
| Get name | `/interface get $id name` | |
| Check running | `/interface get $id running` | |
| Enable | `/interface enable $id` | |
| Disable | `/interface disable $id` | |
| Set MAC | `/interface set $id mac-address=` | |
| Set MTU | `/interface set $id mtu=` | |
| Get stats | `/interface get $id rx-byte` | |
| VLAN create | `/interface vlan add` | |
| Wireless config | `/interface wireless set` | |

> **Tip:** ใช้ `/interface monitor-traffic` เพื่อดู real-time bandwidth

> **Warning:** การ disable ether1 (WAN) อาจทำให้ lose access ได้ ระวังใน production

> **Best Practice:** ทดสอบ interface scripts ใน lab ก่อน deploy จริงเสมอ

---

[← Part 27: Script Parameters](part-027-script-parameters.md) | [Part 29: DHCP Automation →](part-029-dhcp-automation.md)
