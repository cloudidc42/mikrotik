# Part 96: Machine Learning สำหรับ Network Management

## สารบัญ
1. [ML in Networking Overview](#overview)
2. [Anomaly Detection with ML](#anomaly)
3. [Traffic Classification](#classification)
4. [Predictive Maintenance](#predictive)
5. [Automated Optimization](#optimization)
6. [Lab: ML-Powered Monitoring](#lab)

---

## 1. ML in Networking Overview {#overview}

### ML Use Cases ใน Network Management

| Use Case | Algorithm | Benefit |
|----------|-----------|---------|
| Anomaly Detection | Isolation Forest, LSTM | ตรวจจับ DDoS, outages |
| Traffic Classification | Random Forest, SVM | QoS policy automation |
| Failure Prediction | Time-series, ARIMA | Predictive maintenance |
| Capacity Planning | Linear Regression | เพิ่ม capacity ล่วงหน้า |
| Route Optimization | Reinforcement Learning | ลด latency |

---

## 2. Anomaly Detection with ML {#anomaly}

```python
# anomaly_detection.py
import numpy as np
import pandas as pd
from sklearn.ensemble import IsolationForest
from sklearn.preprocessing import StandardScaler
import joblib
import librouteros
from datetime import datetime, timedelta

class NetworkAnomalyDetector:
    """ML-based network anomaly detection"""
    
    def __init__(self, model_path: str = "anomaly_model.pkl"):
        self.model_path = model_path
        self.scaler = StandardScaler()
        self.model = IsolationForest(
            contamination=0.05,  # 5% expected anomalies
            random_state=42,
            n_estimators=100
        )
        self.is_trained = False
        self.feature_names = [
            "rx_bps", "tx_bps", "rx_pps", "tx_pps",
            "rx_errors", "tx_errors", "cpu_load"
        ]
    
    def collect_training_data(self, router_host: str, username: str,
                               password: str, samples: int = 1000) -> pd.DataFrame:
        """เก็บ training data จาก router"""
        import time
        data = []
        
        conn = librouteros.connect(host=router_host, username=username, password=password)
        
        for i in range(samples):
            try:
                interfaces = list(conn('/interface/print'))
                resource = dict(list(conn('/system/resource/print'))[0])
                
                for iface in interfaces:
                    i_data = dict(iface)
                    name = i_data.get("name", "")
                    
                    if name.startswith("ether") or name.startswith("pppoe"):
                        row = {
                            "timestamp": datetime.utcnow().isoformat(),
                            "interface": name,
                            "rx_bps": int(i_data.get("rx-bits-per-second", 0)),
                            "tx_bps": int(i_data.get("tx-bits-per-second", 0)),
                            "rx_pps": int(i_data.get("rx-packets-per-second", 0)),
                            "tx_pps": int(i_data.get("tx-packets-per-second", 0)),
                            "rx_errors": int(i_data.get("rx-errors", 0)),
                            "tx_errors": int(i_data.get("tx-errors", 0)),
                            "cpu_load": int(resource.get("cpu-load", 0))
                        }
                        data.append(row)
            except Exception as e:
                print(f"Sample {i} error: {e}")
            
            if i < samples - 1:
                time.sleep(5)  # ทุก 5 วินาที
        
        conn.close()
        return pd.DataFrame(data)
    
    def train(self, df: pd.DataFrame):
        """Train anomaly detection model"""
        features = df[self.feature_names].fillna(0)
        X = self.scaler.fit_transform(features)
        self.model.fit(X)
        self.is_trained = True
        
        # Save model
        joblib.dump({
            'model': self.model,
            'scaler': self.scaler
        }, self.model_path)
        
        print(f"Model trained on {len(df)} samples")
    
    def load_model(self):
        """Load saved model"""
        saved = joblib.load(self.model_path)
        self.model = saved['model']
        self.scaler = saved['scaler']
        self.is_trained = True
    
    def predict(self, metrics: dict) -> dict:
        """Predict anomaly สำหรับ current metrics"""
        if not self.is_trained:
            return {"error": "Model not trained"}
        
        features = [[
            metrics.get(f, 0) for f in self.feature_names
        ]]
        
        X = self.scaler.transform(features)
        prediction = self.model.predict(X)[0]  # 1=normal, -1=anomaly
        score = self.model.score_samples(X)[0]  # anomaly score
        
        return {
            "is_anomaly": prediction == -1,
            "anomaly_score": round(float(score), 4),
            "severity": "normal" if prediction == 1 else (
                "warning" if score > -0.5 else "critical"
            )
        }
    
    def continuous_monitoring(self, router_host: str, username: str, 
                               password: str, interval: int = 10):
        """Continuous anomaly monitoring"""
        import time
        
        if not self.is_trained:
            print("Model not trained. Training on live data first...")
            df = self.collect_training_data(router_host, username, password, 100)
            self.train(df)
        
        conn = librouteros.connect(host=router_host, username=username, password=password)
        
        print(f"Monitoring {router_host}...")
        
        while True:
            try:
                interfaces = list(conn('/interface/print'))
                resource = dict(list(conn('/system/resource/print'))[0])
                
                for iface in interfaces:
                    i_data = dict(iface)
                    name = i_data.get("name", "")
                    
                    metrics = {
                        "rx_bps": int(i_data.get("rx-bits-per-second", 0)),
                        "tx_bps": int(i_data.get("tx-bits-per-second", 0)),
                        "rx_pps": int(i_data.get("rx-packets-per-second", 0)),
                        "tx_pps": int(i_data.get("tx-packets-per-second", 0)),
                        "rx_errors": int(i_data.get("rx-errors", 0)),
                        "tx_errors": int(i_data.get("tx-errors", 0)),
                        "cpu_load": int(resource.get("cpu-load", 0))
                    }
                    
                    result = self.predict(metrics)
                    
                    if result.get("is_anomaly"):
                        print(f"ANOMALY DETECTED: {name} score={result['anomaly_score']} severity={result['severity']}")
                        # Send alert
            
            except Exception as e:
                print(f"Error: {e}")
            
            time.sleep(interval)
```

---

## 3. Traffic Classification {#classification}

```python
# traffic_classifier.py - จำแนกประเภท traffic ด้วย ML
from sklearn.ensemble import RandomForestClassifier
from sklearn.model_selection import train_test_split
from sklearn.metrics import classification_report
import numpy as np
import pandas as pd

class TrafficClassifier:
    """ML-based traffic classification สำหรับ QoS"""
    
    def __init__(self):
        self.model = RandomForestClassifier(
            n_estimators=100,
            max_depth=10,
            random_state=42
        )
        self.classes = ["streaming", "gaming", "voip", "web", "bulk", "unknown"]
        self.feature_names = [
            "avg_packet_size", "packets_per_second", "bytes_per_second",
            "inter_arrival_time_ms", "flow_duration_sec",
            "dst_port", "tcp_flags_ratio"
        ]
    
    def create_synthetic_dataset(self) -> pd.DataFrame:
        """สร้าง synthetic dataset สำหรับ demo"""
        np.random.seed(42)
        n_samples = 5000
        
        data = []
        
        # Streaming (Netflix, YouTube) - large packets, consistent rate
        for _ in range(n_samples // 6):
            data.append({
                "avg_packet_size": np.random.normal(1400, 50),
                "packets_per_second": np.random.normal(100, 20),
                "bytes_per_second": np.random.normal(140000, 10000),
                "inter_arrival_time_ms": np.random.normal(10, 2),
                "flow_duration_sec": np.random.normal(1800, 300),
                "dst_port": np.random.choice([443, 80]),
                "tcp_flags_ratio": np.random.normal(0.1, 0.02),
                "label": "streaming"
            })
        
        # Gaming - small packets, low latency, consistent
        for _ in range(n_samples // 6):
            data.append({
                "avg_packet_size": np.random.normal(200, 50),
                "packets_per_second": np.random.normal(60, 15),
                "bytes_per_second": np.random.normal(12000, 3000),
                "inter_arrival_time_ms": np.random.normal(16, 3),
                "flow_duration_sec": np.random.normal(3600, 600),
                "dst_port": np.random.choice([27015, 3074, 3659]),
                "tcp_flags_ratio": np.random.normal(0.05, 0.01),
                "label": "gaming"
            })
        
        # VoIP - very small packets, strict timing
        for _ in range(n_samples // 6):
            data.append({
                "avg_packet_size": np.random.normal(160, 20),
                "packets_per_second": np.random.normal(50, 5),
                "bytes_per_second": np.random.normal(8000, 500),
                "inter_arrival_time_ms": np.random.normal(20, 0.5),
                "flow_duration_sec": np.random.normal(300, 120),
                "dst_port": np.random.choice([5060, 16384, 20000]),
                "tcp_flags_ratio": np.random.normal(0.02, 0.005),
                "label": "voip"
            })
        
        # Web browsing - bursty, mixed sizes
        for _ in range(n_samples // 6):
            data.append({
                "avg_packet_size": np.random.normal(800, 300),
                "packets_per_second": np.random.normal(30, 20),
                "bytes_per_second": np.random.normal(24000, 15000),
                "inter_arrival_time_ms": np.random.normal(50, 30),
                "flow_duration_sec": np.random.normal(30, 20),
                "dst_port": np.random.choice([443, 80]),
                "tcp_flags_ratio": np.random.normal(0.2, 0.05),
                "label": "web"
            })
        
        # Bulk transfer - large packets, high throughput
        for _ in range(n_samples // 6):
            data.append({
                "avg_packet_size": np.random.normal(1450, 30),
                "packets_per_second": np.random.normal(300, 50),
                "bytes_per_second": np.random.normal(435000, 50000),
                "inter_arrival_time_ms": np.random.normal(3, 1),
                "flow_duration_sec": np.random.normal(600, 200),
                "dst_port": np.random.choice([21, 22, 8080]),
                "tcp_flags_ratio": np.random.normal(0.1, 0.02),
                "label": "bulk"
            })
        
        return pd.DataFrame(data)
    
    def train(self, df: pd.DataFrame):
        """Train traffic classifier"""
        X = df[self.feature_names].fillna(0)
        y = df["label"]
        
        X_train, X_test, y_train, y_test = train_test_split(
            X, y, test_size=0.2, random_state=42
        )
        
        self.model.fit(X_train, y_train)
        
        # Evaluate
        y_pred = self.model.predict(X_test)
        print(classification_report(y_test, y_pred))
        
        return classification_report(y_test, y_pred, output_dict=True)
    
    def classify_flow(self, flow_features: dict) -> dict:
        """จำแนกประเภท traffic flow"""
        features = [[flow_features.get(f, 0) for f in self.feature_names]]
        
        prediction = self.model.predict(features)[0]
        probabilities = self.model.predict_proba(features)[0]
        
        # QoS recommendation
        qos_map = {
            "voip": {"priority": 1, "dscp": "ef", "queue": "high"},
            "gaming": {"priority": 2, "dscp": "af41", "queue": "high"},
            "streaming": {"priority": 3, "dscp": "af21", "queue": "medium"},
            "web": {"priority": 4, "dscp": "af11", "queue": "medium"},
            "bulk": {"priority": 5, "dscp": "be", "queue": "low"},
            "unknown": {"priority": 6, "dscp": "be", "queue": "default"}
        }
        
        return {
            "traffic_type": prediction,
            "confidence": round(float(max(probabilities)), 3),
            "qos_recommendation": qos_map.get(prediction, qos_map["unknown"])
        }
    
    def generate_mikrotik_qos_rules(self, flows: list) -> str:
        """Generate MikroTik mangle rules จาก flow classifications"""
        rules = ["/ip firewall mangle"]
        
        for flow in flows:
            classification = self.classify_flow(flow)
            qos = classification["qos_recommendation"]
            
            rules.append(
                f"add chain=prerouting "
                f"src-address={flow.get('src_ip', '0.0.0.0')} "
                f"dst-port={int(flow.get('dst_port', 0))} "
                f"protocol=tcp "
                f"action=mark-packet new-packet-mark={classification['traffic_type']} "
                f"comment=\"ML-QoS: {classification['traffic_type']} conf={classification['confidence']}\""
            )
        
        return "\n".join(rules)
```

---

## 4. Predictive Maintenance {#predictive}

```python
# predictive_maintenance.py - ทำนายก่อน failure
import pandas as pd
import numpy as np
from sklearn.linear_model import LinearRegression
from datetime import datetime, timedelta

class PredictiveMaintenance:
    """ทำนาย hardware failure ก่อนเกิดขึ้น"""
    
    def __init__(self):
        self.models = {}
    
    def analyze_interface_errors(self, error_history: list) -> dict:
        """วิเคราะห์ error trend"""
        if len(error_history) < 10:
            return {"status": "insufficient_data"}
        
        x = np.arange(len(error_history)).reshape(-1, 1)
        y = np.array(error_history)
        
        model = LinearRegression()
        model.fit(x, y)
        
        slope = model.coef_[0]
        
        # Predict when errors will reach threshold (1000 errors/sample)
        threshold = 1000
        current = error_history[-1]
        
        if slope > 0 and current < threshold:
            samples_to_threshold = (threshold - current) / slope
            days_to_failure = samples_to_threshold / 288  # 288 samples/day
        else:
            days_to_failure = None
        
        return {
            "current_errors": current,
            "error_trend": "increasing" if slope > 0 else "stable",
            "slope_per_sample": round(slope, 4),
            "days_to_threshold": round(days_to_failure, 1) if days_to_failure else None,
            "action": "replace_cable" if slope > 1 else "monitor"
        }
    
    def predict_cpu_overload(self, cpu_history: list) -> dict:
        """ทำนาย CPU overload"""
        if len(cpu_history) < 20:
            return {"status": "insufficient_data"}
        
        x = np.arange(len(cpu_history)).reshape(-1, 1)
        y = np.array(cpu_history)
        
        model = LinearRegression()
        model.fit(x, y)
        
        slope = model.coef_[0]
        current_cpu = cpu_history[-1]
        
        # Predict when CPU will hit 80%
        if slope > 0 and current_cpu < 80:
            samples_to_80 = (80 - current_cpu) / slope
            hours_to_80 = samples_to_80 / 12  # 12 samples/hour (5min interval)
        else:
            hours_to_80 = None
        
        return {
            "current_cpu_pct": current_cpu,
            "avg_cpu_pct": round(float(np.mean(cpu_history)), 1),
            "trend": "increasing" if slope > 0.1 else "stable",
            "hours_to_80pct": round(hours_to_80, 1) if hours_to_80 else None,
            "recommendation": (
                "upgrade_hardware" if current_cpu > 70 else
                "review_processes" if slope > 0.1 else
                "normal"
            )
        }
    
    def capacity_forecast(self, bandwidth_history: list, 
                           link_capacity_mbps: float) -> dict:
        """พยากรณ์ bandwidth capacity"""
        if len(bandwidth_history) < 30:
            return {"status": "insufficient_data"}
        
        x = np.arange(len(bandwidth_history)).reshape(-1, 1)
        y = np.array(bandwidth_history)
        
        model = LinearRegression()
        model.fit(x, y)
        
        slope = model.coef_[0]
        current = bandwidth_history[-1]
        utilization_pct = (current / (link_capacity_mbps * 1e6)) * 100
        
        # When will utilization hit 80%?
        threshold = link_capacity_mbps * 1e6 * 0.8
        
        if slope > 0 and current < threshold:
            samples_to_threshold = (threshold - current) / slope
            days_to_80pct = samples_to_threshold / 288
        else:
            days_to_80pct = None
        
        return {
            "current_mbps": round(current / 1e6, 1),
            "link_capacity_mbps": link_capacity_mbps,
            "utilization_pct": round(utilization_pct, 1),
            "growth_mbps_per_day": round(slope * 288 / 1e6, 2),
            "days_to_80pct_utilization": round(days_to_80pct, 0) if days_to_80pct else None,
            "upgrade_recommended": days_to_80pct is not None and days_to_80pct < 30
        }
```

---

## 5. Automated Optimization {#optimization}

```python
# auto_optimizer.py - Automated network optimization ด้วย ML
import librouteros
from sklearn.cluster import KMeans
import numpy as np

class NetworkOptimizer:
    """Automated optimization recommendations"""
    
    def __init__(self, router_host: str, username: str, password: str):
        self.router_host = router_host
        self.username = username
        self.password = password
    
    def get_current_state(self) -> dict:
        """ดึง current network state"""
        conn = librouteros.connect(
            host=self.router_host,
            username=self.username,
            password=self.password
        )
        
        state = {
            "interfaces": [dict(i) for i in conn('/interface/print')],
            "routes": [dict(r) for r in conn('/ip/route/print')],
            "firewall_rules": [dict(r) for r in conn('/ip/firewall/filter/print')],
            "resource": dict(list(conn('/system/resource/print'))[0])
        }
        
        conn.close()
        return state
    
    def analyze_firewall_rules(self, rules: list) -> list:
        """หา redundant/inefficient firewall rules"""
        recommendations = []
        
        for i, rule in enumerate(rules):
            r = dict(rule)
            
            # Check if rule has any matches
            packets = int(r.get("packets", 0))
            bytes_count = int(r.get("bytes", 0))
            
            if packets == 0 and bytes_count == 0:
                recommendations.append({
                    "type": "unused_rule",
                    "rule_id": r.get(".id"),
                    "comment": r.get("comment", ""),
                    "action": "Consider removing unused rule"
                })
        
        return recommendations
    
    def optimize_routes(self, routes: list) -> list:
        """Optimize routing table"""
        recommendations = []
        
        # Find duplicate routes
        route_prefixes = {}
        for route in routes:
            r = dict(route)
            dst = r.get("dst-address", "")
            
            if dst in route_prefixes:
                recommendations.append({
                    "type": "duplicate_route",
                    "prefix": dst,
                    "action": "Review duplicate routes"
                })
            else:
                route_prefixes[dst] = r
        
        return recommendations
    
    def generate_report(self) -> dict:
        """Generate optimization report"""
        state = self.get_current_state()
        
        firewall_recs = self.analyze_firewall_rules(state["firewall_rules"])
        route_recs = self.optimize_routes(state["routes"])
        
        cpu = int(state["resource"].get("cpu-load", 0))
        free_mem = int(state["resource"].get("free-memory", 0))
        total_mem = int(state["resource"].get("total-memory", 1))
        mem_pct = ((total_mem - free_mem) / total_mem) * 100
        
        system_recs = []
        if cpu > 70:
            system_recs.append("CPU ใช้งาน > 70% - ลด load หรือ upgrade hardware")
        if mem_pct > 80:
            system_recs.append("Memory ใช้งาน > 80% - optimize หรือ upgrade")
        
        return {
            "timestamp": str(np.datetime64("now")),
            "router": self.router_host,
            "firewall_recommendations": firewall_recs,
            "routing_recommendations": route_recs,
            "system_recommendations": system_recs,
            "total_recommendations": len(firewall_recs) + len(route_recs) + len(system_recs)
        }
```

---

## 6. Lab: ML-Powered Monitoring {#lab}

```bash
# Setup
pip install scikit-learn pandas numpy joblib librouteros

# Train model
python3 << 'EOF'
from anomaly_detection import NetworkAnomalyDetector

detector = NetworkAnomalyDetector()
df = detector.collect_training_data("192.168.1.1", "admin", "password", samples=200)
detector.train(df)
print("Model trained!")
EOF

# Run monitoring
python3 << 'EOF'
from anomaly_detection import NetworkAnomalyDetector

detector = NetworkAnomalyDetector()
detector.load_model()
detector.continuous_monitoring("192.168.1.1", "admin", "password")
EOF
```

### Verification Checklist

- [ ] Training data collected
- [ ] Model trained without errors
- [ ] Anomaly detection running
- [ ] Traffic classification working
- [ ] Predictive analysis showing results
- [ ] Optimization report generated

> **Tip:** ใช้ minimum 1000+ training samples สำหรับ production accuracy

> **Note:** ML models ต้องการ retraining เป็นระยะเมื่อ network patterns เปลี่ยน

---

## Summary

Part นี้ครอบคลุม:
- **Anomaly detection** ด้วย Isolation Forest
- **Traffic classification** ด้วย Random Forest
- **Predictive maintenance** ด้วย Linear Regression
- **Automated optimization** recommendations
- **Integration** กับ MikroTik API

---

[← Part 95: Real-time Analytics](part-095-realtime-analytics.md) | [Part 97: Full-stack ISP Platform →](part-097-fullstack-isp.md)
