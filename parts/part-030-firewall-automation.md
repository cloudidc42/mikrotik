# Part 30: Firewall Automation ใน RouterOS

## บทนำ

Firewall automation ช่วยให้เราตอบสนองต่อ threats ได้อย่างรวดเร็วและอัตโนมัติ ตั้งแต่การ block IPs ที่ทำ port scan ไปจนถึงการ integrate กับ threat intelligence feeds

---

## 30.1 Dynamic Address List Management

### พื้นฐาน Address List

```routeros
# สร้าง address list
/ip firewall address-list add list=whitelist address=192.168.1.0/24
/ip firewall address-list add list=blacklist address=1.2.3.4
/ip firewall address-list add list=monitored address=10.0.0.5

# แสดง address list
/ip firewall address-list print where list=blacklist

# ลบ entry
/ip firewall address-list remove [find list=blacklist address=1.2.3.4]

# เพิ่ม address พร้อม timeout (auto-expire)
/ip firewall address-list add list=temp-block address=5.6.7.8 \
    timeout=1h comment="Temporary block"

# Script เพิ่ม IP เข้า address list
:local addToList do={
    :local list $1
    :local ip $2
    :local comment $3
    :local timeout $4
    
    # ตรวจสอบ duplicate
    :if ([:len [/ip firewall address-list find list=$list address=$ip]] > 0) do={
        :put "IP $ip already in $list"
        :return false
    }
    
    :if ([:len $timeout] > 0) do={
        /ip firewall address-list add \
            list=$list address=$ip comment=$comment timeout=$timeout
    } else={
        /ip firewall address-list add \
            list=$list address=$ip comment=$comment
    }
    
    :log info ("Added to " . $list . ": " . $ip . " (" . $comment . ")")
    :return true
}

[$addToList "blacklist" "1.2.3.4" "Manually blocked" ""]
[$addToList "temp-block" "5.6.7.8" "Suspicious activity" "2h"]
```

---

## 30.2 Automatic Ban by Pattern

```routeros
# ตั้ง firewall rules สำหรับ auto-detect และ ban
# ก่อนอื่น ต้องมี rules เหล่านี้ใน filter table

# Rule 1: นับ connections ต่อ IP
/ip firewall filter add chain=input \
    protocol=tcp \
    connection-state=new \
    src-address-list=!whitelist \
    action=add-src-to-address-list \
    address-list=scan-stage-1 \
    address-list-timeout=1m \
    comment="Port scan detection stage 1"

/ip firewall filter add chain=input \
    protocol=tcp \
    connection-state=new \
    src-address-list=scan-stage-1 \
    action=add-src-to-address-list \
    address-list=scan-stage-2 \
    address-list-timeout=1m \
    comment="Port scan detection stage 2"

/ip firewall filter add chain=input \
    protocol=tcp \
    connection-state=new \
    src-address-list=scan-stage-2 \
    action=add-src-to-address-list \
    address-list=blacklist \
    address-list-timeout=1d \
    comment="Add port scanners to blacklist"

# Rule 2: Block blacklisted IPs
/ip firewall filter add chain=input \
    src-address-list=blacklist \
    action=drop \
    comment="Block blacklisted IPs"

# Script ตรวจ auto-ban list และแจ้ง admin
/system script add name="check-autobans" source="
    :local newBans [/ip firewall address-list find list=blacklist \
        where comment~\"scan\"]
    
    :if ([:len \$newBans] > 0) do={
        :local banList \"\"
        :foreach ban in=\$newBans do={
            :local ip [/ip firewall address-list get \$ban address]
            :set banList (\$banList . \$ip . \"\\r\\n\")
        }
        :log warning (\"Auto-banned \" . [:len \$newBans] . \" IPs for port scanning\")
    }
"
```

---

## 30.3 Port Scan Detection and Ban

```routeros
# Advanced port scan detection
# ใช้ connection tracking

# นับ destination ports ต่อ source IP
/ip firewall filter add chain=forward \
    protocol=tcp \
    connection-state=new \
    psd=21,3s,5,25 \
    action=add-src-to-address-list \
    address-list=port-scanners \
    address-list-timeout=1d \
    comment="PSD: port scan detection"

/ip firewall filter add chain=input \
    protocol=tcp \
    connection-state=new \
    psd=21,3s,5,25 \
    action=add-src-to-address-list \
    address-list=port-scanners \
    address-list-timeout=1d \
    comment="PSD: input port scan detection"

# Block port scanners
/ip firewall filter add chain=forward \
    src-address-list=port-scanners \
    action=drop \
    comment="Block port scanners (forward)"

/ip firewall filter add chain=input \
    src-address-list=port-scanners \
    action=drop \
    comment="Block port scanners (input)"

# Script ตรวจสอบและรายงาน
/system script add name="scan-report" source="
    :local scanners [/ip firewall address-list find list=port-scanners]
    :local count [:len \$scanners]
    
    :put \"Port scan report:\"
    :put \"Total IPs blocked: \$count\"
    
    :foreach scanner in=\$scanners do={
        :local ip [/ip firewall address-list get \$scanner address]
        :local comment [/ip firewall address-list get \$scanner comment]
        :put \" - \$ip (\$comment)\"
    }
"
```

---

## 30.4 Rate Limiting Automation

```routeros
# Auto-block IPs ที่ exceed rate limit

# Connection rate limit
/ip firewall filter add chain=input \
    protocol=tcp \
    dst-port=22 \
    connection-limit=5,32 \
    action=add-src-to-address-list \
    address-list=ssh-brute \
    address-list-timeout=1d \
    comment="SSH brute force: too many connections"

/ip firewall filter add chain=input \
    protocol=tcp \
    dst-port=22 \
    src-address-list=ssh-brute \
    action=drop \
    comment="Block SSH brute force"

# HTTP rate limiting
/ip firewall filter add chain=forward \
    protocol=tcp \
    dst-port=80,443 \
    connection-rate=100 \
    action=add-src-to-address-list \
    address-list=http-flood \
    address-list-timeout=30m \
    comment="HTTP flood detection"

# Script monitor rate-limited IPs
:local monitorRateLimits do={
    :local lists {"ssh-brute"; "http-flood"; "port-scanners"; "blacklist"}
    :local report "=== Rate Limit Report ===\r\n"
    
    :foreach list in=$lists do={
        :local count [:len [/ip firewall address-list find list=$list]]
        :set report ($report . $list . ": " . $count . " IPs blocked\r\n")
    }
    
    :put $report
    :log info "Rate limit check complete"
    :return $report
}

[$monitorRateLimits]
```

---

## 30.5 GeoIP Blocking Script

```routeros
# GeoIP blocking (ต้องมี IP list ของแต่ละประเทศ)
# ตัวอย่างนี้ใช้ script download IP ranges

# สร้าง address list สำหรับ country blocks
/ip firewall address-list add list=country-cn address=1.0.1.0/24 comment="China"
/ip firewall address-list add list=country-cn address=1.0.2.0/23 comment="China"
# ... (ต้อง import ทั้ง list)

# Script download และ update GeoIP list
/system script add name="update-geoip" source="
    :local countries {\"cn\"; \"kp\"; \"ru\"}  # Countries to block
    
    :foreach country in=\$countries do={
        :local url (\"http://www.ipdeny.com/ipblocks/data/aggregated/\" . \$country . \"-aggregated.zone\")
        :local listName (\"country-\" . \$country)
        
        :do {
            /tool fetch url=\$url dst-path=(\$country . \".txt\")
            
            # Remove old entries
            /ip firewall address-list remove [find list=\$listName]
            
            # Import new entries
            :local content [/file get [find name=(\$country . \".txt\")] contents]
            :local lines [:toarray \$content]
            
            :foreach line in=\$lines do={
                :if ([:len \$line] > 0) do={
                    :do {
                        /ip firewall address-list add \
                            list=\$listName address=\$line comment=\$country
                    } on-error={}
                }
            }
            
            :log info (\"GeoIP updated: \" . \$country . \" (\" . [:len \$lines] . \" ranges)\")
        } on-error={
            :log warning (\"Failed to update GeoIP for: \" . \$country)
        }
    }
"

# Rules สำหรับ block ตาม country
/ip firewall filter add chain=forward \
    src-address-list=country-cn \
    action=drop \
    comment="Block China (GeoIP)"

/ip firewall filter add chain=input \
    src-address-list=country-cn \
    action=drop \
    comment="Block China input (GeoIP)"

# Schedule อัพเดท GeoIP ทุกเดือน
/system scheduler add \
    name="geoip-update" \
    start-time=03:00:00 \
    interval=30d \
    on-event="/system script run update-geoip"
```

---

## 30.6 Firewall Rule Generation

```routeros
# Script สร้าง firewall rules จาก config

:local createFirewallRules do={
    :local rules $1
    
    :foreach rule in=$rules do={
        :local chain ($rule->"chain")
        :local srcAddr ($rule->"src-address")
        :local dstAddr ($rule->"dst-address")
        :local protocol ($rule->"protocol")
        :local port ($rule->"port")
        :local action ($rule->"action")
        :local comment ($rule->"comment")
        
        # Build command
        :local cmd "/ip firewall filter add chain=$chain action=$action"
        :if ([:len $srcAddr] > 0) do={
            :set cmd ($cmd . " src-address=" . $srcAddr)
        }
        :if ([:len $dstAddr] > 0) do={
            :set cmd ($cmd . " dst-address=" . $dstAddr)
        }
        :if ([:len $protocol] > 0) do={
            :set cmd ($cmd . " protocol=" . $protocol)
        }
        :if ([:len $port] > 0) do={
            :set cmd ($cmd . " dst-port=" . $port)
        }
        :if ([:len $comment] > 0) do={
            :set cmd ($cmd . " comment=\"" . $comment . "\"")
        }
        
        :do {
            /ip firewall filter add \
                chain=$chain \
                action=$action \
                comment=$comment
            :put "Rule added: $comment"
        } on-error={
            :put "Failed to add rule: $comment"
        }
    }
}

# Define rules as config
:local firewallConfig {
    {"chain"="input"; "protocol"="tcp"; "port"="22"; "action"="accept"; "comment"="Allow SSH from management"},
    {"chain"="input"; "protocol"="tcp"; "port"="8291"; "action"="accept"; "comment"="Allow Winbox"},
    {"chain"="input"; "protocol"="tcp"; "port"="80,443"; "action"="accept"; "comment"="Allow HTTP/HTTPS"},
    {"chain"="input"; "action"="drop"; "comment"="Drop all other input"}
}

[$createFirewallRules $firewallConfig]

# Export firewall rules เป็น rsc
/ip firewall filter export file=firewall-rules
:put "Firewall rules exported"

# Validate firewall rules
:local validateFirewall do={
    :local rules [/ip firewall filter find chain=input action=accept]
    :local dropRules [/ip firewall filter find chain=input action=drop]
    
    :put "Firewall Validation:"
    :put "Accept rules in input: " . [:len $rules]
    :put "Drop rules in input: " . [:len $dropRules]
    
    :if ([:len $dropRules] = 0) do={
        :put "WARNING: No drop rules in input chain!"
        :log warning "Firewall has no drop rules!"
    }
}
```

---

## 30.7 Whitelist Management

```routeros
# จัดการ whitelist

# เพิ่ม IP เข้า whitelist
:local addToWhitelist do={
    :local ip $1
    :local reason $2
    :local adminUser $3
    
    # ตรวจสอบ duplicate
    :if ([:len [/ip firewall address-list find list=whitelist address=$ip]] > 0) do={
        :put "IP $ip already whitelisted"
        :return false
    }
    
    :local comment ("WL: " . $reason . " by " . $adminUser . " on " . [/system clock get date])
    /ip firewall address-list add list=whitelist address=$ip comment=$comment
    
    :log info ("Whitelisted: " . $ip . " (" . $reason . ") by " . $adminUser)
    :put "Added $ip to whitelist"
    :return true
}

# ลบ IP ออกจาก whitelist
:local removeFromWhitelist do={
    :local ip $1
    :local reason $2
    
    :local entry [/ip firewall address-list find list=whitelist address=$ip]
    :if ([:len $entry] = 0) do={
        :put "IP $ip not in whitelist"
        :return false
    }
    
    /ip firewall address-list remove $entry
    :log info ("Removed from whitelist: " . $ip . " (" . $reason . ")")
    :put "Removed $ip from whitelist"
    :return true
}

# ตรวจสอบว่า IP อยู่ใน whitelist
:local isWhitelisted do={
    :local ip $1
    :return ([:len [/ip firewall address-list find list=whitelist address=$ip]] > 0)
}

# ใช้งาน
[$addToWhitelist "203.0.113.10" "Management server" "admin"]
[$addToWhitelist "192.168.100.0/24" "Internal management subnet" "admin"]

:if ([$isWhitelisted "203.0.113.10"]) do={
    :put "IP is whitelisted"
}

# Whitelist cleanup - ลบ entries เก่า (comment ระบุวันที่)
:local cleanupOldWhitelist do={
    :put "Whitelist entries:"
    :foreach entry in=[/ip firewall address-list find list=whitelist] do={
        :local ip [/ip firewall address-list get $entry address]
        :local comment [/ip firewall address-list get $entry comment]
        :put "  $ip - $comment"
    }
}

[$cleanupOldWhitelist]
```

---

## 30.8 Threat Intelligence Integration

```routeros
# Download blocklists จาก threat intelligence sources
/system script add name="update-threat-feeds" source="
    :local feeds {
        {\"list\"=\"threat-spamhaus\"; \"url\"=\"https://www.spamhaus.org/drop/drop.txt\"};
        {\"list\"=\"threat-abuse\"; \"url\"=\"https://feodotracker.abuse.ch/downloads/ipblocklist.csv\"}
    }
    
    :foreach feed in=\$feeds do={
        :local listName (\$feed->\"list\")
        :local url (\$feed->\"url\")
        :local filename (\$listName . \".txt\")
        
        :put \"Updating feed: \$listName\"
        
        :do {
            /tool fetch url=\$url dst-path=\$filename mode=https
            
            :local content [/file get [find name=\$filename] contents]
            :local lines [:toarray \$content]
            
            # ลบรายการเก่า
            /ip firewall address-list remove [find list=\$listName]
            
            :local added 0
            :foreach line in=\$lines do={
                # Skip comments
                :if ([:find \$line \"#\"] != 0) do={
                    :if ([:len \$line] > 0) do={
                        :do {
                            /ip firewall address-list add \
                                list=\$listName address=\$line \
                                comment=(\"TI: \" . \$listName)
                            :set added (\$added + 1)
                        } on-error={}
                    }
                }
            }
            
            :log info (\"Threat feed updated: \" . \$listName . \" (\" . \$added . \" entries)\")
        } on-error={
            :log error (\"Failed to update threat feed: \" . \$listName)
        }
    }
"

# Block threat IPs
/ip firewall filter add chain=forward \
    src-address-list=threat-spamhaus \
    action=drop \
    comment="Block Spamhaus DROP list"

/ip firewall filter add chain=input \
    src-address-list=threat-spamhaus \
    action=drop \
    comment="Block Spamhaus DROP list (input)"

# Schedule อัพเดท daily
/system scheduler add \
    name="threat-feed-update" \
    start-time=04:00:00 \
    interval=1d \
    on-event="/system script run update-threat-feeds"
```

---

## 30.9 Firewall Audit Scripts

```routeros
# Audit firewall configuration
/system script add name="firewall-audit" source="
    :local report \"=== Firewall Audit Report ===\\r\\n\"
    :set report (\$report . \"Date: \" . [/system clock get date] . \"\\r\\n\")
    :set report (\$report . \"Router: \" . [/system identity get name] . \"\\r\\n\\r\\n\")
    
    # Check input chain
    :local inputRules [:len [/ip firewall filter find chain=input]]
    :local inputDropRules [:len [/ip firewall filter find chain=input action=drop]]
    :local inputAcceptRules [:len [/ip firewall filter find chain=input action=accept]]
    
    :set report (\$report . \"Input chain:\\r\\n\")
    :set report (\$report . \"  Total rules: \" . \$inputRules . \"\\r\\n\")
    :set report (\$report . \"  Accept rules: \" . \$inputAcceptRules . \"\\r\\n\")
    :set report (\$report . \"  Drop rules: \" . \$inputDropRules . \"\\r\\n\")
    
    # Check for dangerous open ports
    :local openPorts [/ip firewall filter find chain=input action=accept \
        protocol=tcp]
    :set report (\$report . \"\\r\\nOpen ports (input accept TCP):\\r\\n\")
    :foreach rule in=\$openPorts do={
        :local port [/ip firewall filter get \$rule dst-port]
        :local comment [/ip firewall filter get \$rule comment]
        :set report (\$report . \"  Port \" . \$port . \" - \" . \$comment . \"\\r\\n\")
    }
    
    # Address list statistics
    :set report (\$report . \"\\r\\nAddress Lists:\\r\\n\")
    :foreach list in={\"blacklist\"; \"whitelist\"; \"port-scanners\"; \"ssh-brute\"} do={
        :local count [:len [/ip firewall address-list find list=\$list]]
        :set report (\$report . \"  \" . \$list . \": \" . \$count . \" entries\\r\\n\")
    }
    
    # Check for disabled rules
    :local disabledRules [:len [/ip firewall filter find disabled=yes]]
    :set report (\$report . \"\\r\\nDisabled rules: \" . \$disabledRules . \"\\r\\n\")
    
    :put \$report
    /log info \"Firewall audit completed\"
    
    /tool e-mail send \
        to=\"admin@company.com\" \
        subject=\"Firewall Audit Report\" \
        body=\$report
" comment="Firewall audit and report"

/system scheduler add \
    name="firewall-audit" \
    start-time=07:00:00 \
    interval=7d \
    on-event="/system script run firewall-audit"
```

---

## Lab 30: Auto-Ban Script

### โจทย์
สร้าง Auto-Ban system ที่:
1. Detect port scans
2. Detect SSH brute force
3. Detect HTTP floods
4. Auto-ban ด้วย graduated timeouts
5. Whitelist management
6. Alert admin
7. Daily report

### Solution

```routeros
# Auto-Ban System Lab
# ====================

# --- Firewall Rules Setup ---

# Stage 1: SSH Brute Force Detection
/ip firewall filter add chain=input \
    protocol=tcp dst-port=22 \
    connection-state=new \
    src-address-list=!whitelist \
    action=add-src-to-address-list \
    address-list=ssh-stage1 \
    address-list-timeout=1m \
    comment="SSH BF Stage 1"

/ip firewall filter add chain=input \
    protocol=tcp dst-port=22 \
    connection-state=new \
    src-address-list=ssh-stage1 \
    action=add-src-to-address-list \
    address-list=ssh-stage2 \
    address-list-timeout=1m \
    comment="SSH BF Stage 2"

/ip firewall filter add chain=input \
    protocol=tcp dst-port=22 \
    connection-state=new \
    src-address-list=ssh-stage2 \
    action=add-src-to-address-list \
    address-list=auto-ban-1h \
    address-list-timeout=1h \
    comment="SSH BF: Ban 1 hour"

# Stage 2: Port Scan Detection
/ip firewall filter add chain=input \
    protocol=tcp \
    psd=21,3s,5,25 \
    src-address-list=!whitelist \
    action=add-src-to-address-list \
    address-list=auto-ban-24h \
    address-list-timeout=1d \
    comment="Port Scan: Ban 24 hours"

# Stage 3: Drop all banned IPs
/ip firewall filter add chain=input \
    src-address-list=auto-ban-1h \
    action=drop \
    comment="Auto-ban: 1 hour bans"

/ip firewall filter add chain=input \
    src-address-list=auto-ban-24h \
    action=drop \
    comment="Auto-ban: 24 hour bans"

/ip firewall filter add chain=forward \
    src-address-list=auto-ban-24h \
    action=drop \
    comment="Auto-ban forward: 24 hour bans"

# --- Auto-Ban Monitor Script ---
/system script add name="autoban-monitor" source="
    :global AUTOBAN_ALERT_EMAIL \"admin@company.com\"
    :global AUTOBAN_LAST_COUNT
    
    :if ([:typeof \$AUTOBAN_LAST_COUNT] = \"nothing\") do={
        :set AUTOBAN_LAST_COUNT 0
    }
    
    :local ban1h [:len [/ip firewall address-list find list=auto-ban-1h]]
    :local ban24h [:len [/ip firewall address-list find list=auto-ban-24h]]
    :local total (\$ban1h + \$ban24h)
    
    # ถ้ามีการ ban ใหม่ ส่ง alert
    :if (\$total > \$AUTOBAN_LAST_COUNT) do={
        :local newBans (\$total - \$AUTOBAN_LAST_COUNT)
        :log warning (\"New auto-bans: \" . \$newBans . \" IPs (total: \" . \$total . \")\")
        
        # ส่ง alert ถ้า bans เพิ่มขึ้น > 10
        :if (\$newBans > 10) do={
            /tool e-mail send \
                to=\$AUTOBAN_ALERT_EMAIL \
                subject=\"[ALERT] Unusual Auto-Ban Activity\" \
                body=(\"\\$newBans new IPs were auto-banned in the last minute\\r\\n\" . \
                      \"1h bans: \" . \$ban1h . \"\\r\\n\" . \
                      \"24h bans: \" . \$ban24h . \"\\r\\n\" . \
                      \"Total: \" . \$total)
        }
    }
    
    :set AUTOBAN_LAST_COUNT \$total
" comment="Auto-ban monitoring"

# --- Daily Report ---
/system script add name="autoban-daily-report" source="
    :local report \"=== Auto-Ban Daily Report ===\\r\\n\"
    :set report (\$report . \"Date: \" . [/system clock get date] . \"\\r\\n\")
    :set report (\$report . \"Router: \" . [/system identity get name] . \"\\r\\n\\r\\n\")
    
    :local ban1h [/ip firewall address-list find list=auto-ban-1h]
    :local ban24h [/ip firewall address-list find list=auto-ban-24h]
    
    :set report (\$report . \"Current Bans:\\r\\n\")
    :set report (\$report . \"  1-hour bans: \" . [:len \$ban1h] . \"\\r\\n\")
    :set report (\$report . \"  24-hour bans: \" . [:len \$ban24h] . \"\\r\\n\\r\\n\")
    
    :if ([:len \$ban24h] > 0) do={
        :set report (\$report . \"24-hour banned IPs:\\r\\n\")
        :local showCount 0
        :foreach ban in=\$ban24h do={
            :if (\$showCount < 20) do={
                :local ip [/ip firewall address-list get \$ban address]
                :set report (\$report . \"  \" . \$ip . \"\\r\\n\")
                :set showCount (\$showCount + 1)
            }
        }
        :if ([:len \$ban24h] > 20) do={
            :set report (\$report . \"  ... and \" . ([:len \$ban24h] - 20) . \" more\\r\\n\")
        }
    }
    
    /tool e-mail send \
        to=\"admin@company.com\" \
        subject=\"Daily Auto-Ban Report\" \
        body=\$report
    :log info \"Auto-ban daily report sent\"
" comment="Daily auto-ban report"

# --- Setup Schedulers ---
/system scheduler add name="autoban-monitor" interval=1m \
    on-event="/system script run autoban-monitor"

/system scheduler add name="autoban-daily-report" \
    start-time=08:00:00 interval=1d \
    on-event="/system script run autoban-daily-report"

:put "Auto-Ban system configured!"
:put "Rules added:"
:put "- SSH brute force detection (2-stage)"
:put "- Port scan detection (PSD)"
:put "- Auto-ban with graduated timeouts"
:put "- Monitoring and alerting"
:put "- Daily reports"
```

---

## Summary ของ Part 30

| Technique | ใช้เมื่อ | Duration |
|-----------|---------|---------|
| Port scan detect | PSD ใน filter rules | 1d ban |
| SSH brute force | Connection limit | 1h-24h ban |
| HTTP flood | Connection rate | 30m ban |
| GeoIP block | Block by country | Permanent |
| Threat feed | Known bad IPs | Auto-expire |
| Dynamic list | Script-based ban | Configurable |

> **Warning:** Auto-ban scripts อาจ block legitimate users ถ้า configure ผิด ทดสอบใน lab ก่อน

> **Tip:** เสมอมี whitelist สำหรับ management IPs เพื่อป้องกันตัวเองถูก lock out

> **Best Practice:** Log ทุก auto-ban action เพื่อ audit และ false positive analysis

---

[← Part 29: DHCP Automation](part-029-dhcp-automation.md) | [Part 31: Email Notifications →](part-031-email-notifications.md)
