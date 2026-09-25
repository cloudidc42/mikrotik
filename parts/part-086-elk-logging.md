# Part 86: ELK Stack สำหรับ MikroTik Logging

## สารบัญ
1. [ELK Stack Overview](#overview)
2. [Logstash for Syslog](#logstash)
3. [Elasticsearch Index](#elasticsearch)
4. [Kibana Dashboards](#kibana)
5. [Alerting](#alerting)
6. [Lab: ELK Setup](#lab)

---

## 1. ELK Stack Overview {#overview}

ELK Stack ประกอบด้วย Elasticsearch (storage), Logstash (processing), Kibana (visualization)

```
[MikroTik Router]
    │ syslog UDP:514
    ↓
[Logstash]
    │ Parse & enrich
    ↓
[Elasticsearch]
    │ Store & index
    ↓
[Kibana]
    Dashboard & Alerts
```

---

## 2. Logstash for Syslog {#logstash}

```yaml
# docker-compose.yml
version: '3.8'

services:
  elasticsearch:
    image: elasticsearch:8.10.0
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx512m
      - xpack.security.enabled=false
    ports:
      - "9200:9200"
    volumes:
      - es_data:/usr/share/elasticsearch/data

  logstash:
    image: logstash:8.10.0
    ports:
      - "514:514/udp"
      - "5044:5044"
    volumes:
      - ./logstash/pipeline:/usr/share/logstash/pipeline
    depends_on:
      - elasticsearch

  kibana:
    image: kibana:8.10.0
    ports:
      - "5601:5601"
    environment:
      ELASTICSEARCH_HOSTS: "http://elasticsearch:9200"
    depends_on:
      - elasticsearch

volumes:
  es_data:
```

```ruby
# logstash/pipeline/mikrotik.conf

input {
  udp {
    port => 514
    type => "syslog"
    codec => plain {
      charset => "UTF-8"
    }
  }
}

filter {
  if [type] == "syslog" {
    # Parse syslog format
    grok {
      match => {
        "message" => "<%{NONNEGINT:syslog_pri}>%{SYSLOGTIMESTAMP:syslog_timestamp} %{SYSLOGHOST:syslog_hostname} %{DATA:syslog_program}(?:\[%{POSINT:syslog_pid}\])?: %{GREEDYDATA:syslog_message}"
      }
      tag_on_failure => ["_grokparsefailure_syslog"]
    }
    
    # Parse MikroTik specific formats
    
    # Firewall log format: in:ether1 out:ether2 src-mac=... proto=TCP ...
    if [syslog_message] =~ /^(forward|input|output):/ {
      grok {
        match => {
          "syslog_message" => "%{WORD:fw_chain}: in:%{WORD:fw_in_iface} out:%{WORD:fw_out_iface}.*src-mac %{MAC:src_mac}, proto %{WORD:proto}, %{IP:src_ip}:%{INT:src_port}->%{IP:dst_ip}:%{INT:dst_port}"
        }
        tag_on_failure => ["_grokparsefailure_firewall"]
      }
      mutate { add_tag => ["firewall"] }
    }
    
    # DHCP log
    if [syslog_message] =~ /dhcp/ {
      grok {
        match => {
          "syslog_message" => "dhcp %{WORD}: %{IP:dhcp_ip} assigned to %{MAC:dhcp_mac}"
        }
        tag_on_failure => []
      }
      mutate { add_tag => ["dhcp"] }
    }
    
    # VPN connections
    if [syslog_message] =~ /ppp|ovpn|l2tp|pptp/ {
      mutate { add_tag => ["vpn"] }
    }
    
    # System events
    if [syslog_message] =~ /logged in|logged out|login failed/ {
      grok {
        match => {
          "syslog_message" => "%{WORD:auth_user} logged (in|out) from %{IP:auth_ip}"
        }
        tag_on_failure => []
      }
      mutate { add_tag => ["authentication"] }
    }
    
    # Geolocate source IPs
    if [src_ip] {
      geoip {
        source => "src_ip"
        target => "geoip"
        fields => ["city_name", "country_name", "location"]
      }
    }
    
    # Parse timestamp
    date {
      match => ["syslog_timestamp", "MMM dd HH:mm:ss", "MMM  d HH:mm:ss"]
      target => "@timestamp"
    }
    
    # Add router identifier
    mutate {
      add_field => {
        "router_ip" => "%{syslog_hostname}"
        "[@metadata][index]" => "mikrotik-%{+YYYY.MM.dd}"
      }
    }
  }
}

output {
  elasticsearch {
    hosts => ["http://elasticsearch:9200"]
    index => "%{[@metadata][index]}"
  }
  
  # Debug output
  # stdout { codec => rubydebug }
}
```

---

## 3. Elasticsearch Index {#elasticsearch}

```python
# Python: สร้าง Elasticsearch index template
from elasticsearch import Elasticsearch
import json

es = Elasticsearch("http://localhost:9200")

# Index template สำหรับ MikroTik logs
template = {
    "index_patterns": ["mikrotik-*"],
    "settings": {
        "number_of_shards": 1,
        "number_of_replicas": 0,
        "index.lifecycle.name": "mikrotik-ilm",
        "refresh_interval": "5s"
    },
    "mappings": {
        "properties": {
            "@timestamp": {"type": "date"},
            "router_ip": {"type": "ip"},
            "syslog_hostname": {"type": "keyword"},
            "syslog_program": {"type": "keyword"},
            "syslog_message": {"type": "text"},
            "fw_chain": {"type": "keyword"},
            "fw_in_iface": {"type": "keyword"},
            "fw_out_iface": {"type": "keyword"},
            "src_ip": {"type": "ip"},
            "dst_ip": {"type": "ip"},
            "src_port": {"type": "integer"},
            "dst_port": {"type": "integer"},
            "proto": {"type": "keyword"},
            "tags": {"type": "keyword"},
            "geoip": {
                "properties": {
                    "country_name": {"type": "keyword"},
                    "city_name": {"type": "keyword"},
                    "location": {"type": "geo_point"}
                }
            }
        }
    }
}

es.indices.put_index_template(name="mikrotik-template", body=template)
print("Index template created")

# ILM Policy (Index Lifecycle Management)
ilm_policy = {
    "policy": {
        "phases": {
            "hot": {
                "actions": {
                    "rollover": {
                        "max_age": "1d",
                        "max_size": "5GB"
                    }
                }
            },
            "warm": {
                "min_age": "7d",
                "actions": {
                    "shrink": {"number_of_shards": 1},
                    "forcemerge": {"max_num_segments": 1}
                }
            },
            "delete": {
                "min_age": "90d",
                "actions": {
                    "delete": {}
                }
            }
        }
    }
}

es.ilm.put_lifecycle(name="mikrotik-ilm", body=ilm_policy)
print("ILM policy created")
```

---

## 4. Kibana Dashboards {#kibana}

```python
# สร้าง Kibana dashboard ผ่าน API
import requests
import json

KIBANA_URL = "http://localhost:5601"

# สร้าง index pattern
headers = {"Content-Type": "application/json", "kbn-xsrf": "true"}

index_pattern = {
    "attributes": {
        "title": "mikrotik-*",
        "timeFieldName": "@timestamp"
    }
}

r = requests.post(
    f"{KIBANA_URL}/api/saved_objects/index-pattern",
    headers=headers,
    json=index_pattern
)
print(f"Index pattern: {r.status_code}")

# สร้าง visualization - Firewall drops per hour
vis_config = {
    "attributes": {
        "title": "Firewall Drops Per Hour",
        "visState": json.dumps({
            "title": "Firewall Drops Per Hour",
            "type": "histogram",
            "params": {
                "grid": {"categoryLines": False},
                "categoryAxes": [{"type": "category", "position": "bottom"}],
                "valueAxes": [{"type": "value", "position": "left"}]
            },
            "aggs": [
                {
                    "id": "1",
                    "enabled": True,
                    "type": "count",
                    "schema": "metric"
                },
                {
                    "id": "2",
                    "enabled": True,
                    "type": "date_histogram",
                    "schema": "segment",
                    "params": {
                        "field": "@timestamp",
                        "interval": "1h"
                    }
                }
            ]
        })
    }
}

r = requests.post(
    f"{KIBANA_URL}/api/saved_objects/visualization",
    headers=headers,
    json=vis_config
)
print(f"Visualization: {r.status_code}")
```

---

## 5. Alerting {#alerting}

```python
# Python alerting based on Elasticsearch queries
from elasticsearch import Elasticsearch
from datetime import datetime, timedelta
import smtplib
from email.mime.text import MIMEText

class SecurityAlerting:
    def __init__(self, es_url: str):
        self.es = Elasticsearch(es_url)
    
    def check_failed_logins(self, threshold: int = 10) -> list:
        """ตรวจสอบ failed login attempts"""
        query = {
            "query": {
                "bool": {
                    "must": [
                        {"term": {"tags": "authentication"}},
                        {"match": {"syslog_message": "login failed"}},
                        {"range": {"@timestamp": {"gte": "now-1h"}}}
                    ]
                }
            },
            "aggs": {
                "by_ip": {
                    "terms": {"field": "auth_ip.keyword", "size": 10}
                }
            }
        }
        
        result = self.es.search(index="mikrotik-*", body=query)
        
        alerts = []
        for bucket in result["aggregations"]["by_ip"]["buckets"]:
            if bucket["doc_count"] >= threshold:
                alerts.append({
                    "type": "brute_force",
                    "ip": bucket["key"],
                    "count": bucket["doc_count"],
                    "severity": "critical" if bucket["doc_count"] > 50 else "high"
                })
        
        return alerts
    
    def check_firewall_spikes(self) -> list:
        """ตรวจสอบ spike ใน firewall drops"""
        query = {
            "query": {
                "bool": {
                    "must": [
                        {"term": {"tags": "firewall"}},
                        {"range": {"@timestamp": {"gte": "now-5m"}}}
                    ]
                }
            }
        }
        
        result = self.es.count(index="mikrotik-*", body=query)
        count = result["count"]
        
        alerts = []
        if count > 1000:
            alerts.append({
                "type": "firewall_spike",
                "count": count,
                "period": "5 minutes",
                "severity": "high"
            })
        
        return alerts

if __name__ == "__main__":
    alerter = SecurityAlerting("http://localhost:9200")
    
    failed_logins = alerter.check_failed_logins()
    firewall_spikes = alerter.check_firewall_spikes()
    
    all_alerts = failed_logins + firewall_spikes
    
    for alert in all_alerts:
        print(f"ALERT [{alert['severity']}]: {alert['type']} - {json.dumps(alert)}")
```

---

## 6. Lab: ELK Setup {#lab}

### Lab Steps

```bash
# Step 1: Configure MikroTik syslog
/system logging action set remote remote=192.168.10.50 remote-port=514 bsd-syslog=yes
/system logging add topics=firewall,system,account,critical action=remote

# Step 2: Start ELK Stack
docker-compose up -d

# Step 3: Verify Logstash receives logs
docker logs logstash-1 | grep "Starting server"
# Send test log
logger -n 192.168.10.50 -P 514 "Test MikroTik log message"

# Step 4: Verify Elasticsearch
curl http://localhost:9200/_cat/indices?v

# Step 5: Open Kibana
# http://localhost:5601
# Create index pattern: mikrotik-*
# Explore logs in Discover
```

### Verification Checklist

- [ ] MikroTik ส่ง syslog ไป Logstash
- [ ] Logstash parse logs ถูกต้อง
- [ ] Elasticsearch index สร้างแล้ว
- [ ] Kibana index pattern configured
- [ ] Firewall logs แยก tag ได้
- [ ] Dashboard แสดง metrics

> **Tip:** ใช้ Kibana Dev Tools เพื่อ query Elasticsearch โดยตรง

---

## Summary

Part นี้ครอบคลุม:
- **ELK Stack** architecture
- **Logstash** pipeline สำหรับ MikroTik syslog
- **Elasticsearch** index template และ ILM
- **Kibana** dashboards
- **Alerting** สำหรับ security events

---

[← Part 85: Netmiko Python](part-085-netmiko-python.md) | [Part 87: Prometheus Grafana →](part-087-prometheus-grafana.md)
