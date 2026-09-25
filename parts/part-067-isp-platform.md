# Part 67: ISP Platform Architecture

## สารบัญ
1. [ISP Platform Architecture](#architecture)
2. [Customer Management](#customers)
3. [Network Management](#network)
4. [Billing Integration](#billing)
5. [Support System](#support)
6. [NOC Dashboard](#noc)
7. [Automated Provisioning](#provisioning)
8. [SLA Management](#sla)
9. [Reporting and Analytics](#reporting)
10. [Lab: ISP Platform Deployment](#lab)

---

## 1. ISP Platform Architecture {#architecture}

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                      ISP Management Platform                      │
├──────────────┬──────────────┬──────────────┬────────────────────┤
│  Web Portal  │  Mobile App  │  Admin Panel │  API Gateway       │
│  (React)     │  (React Native)│ (React)    │  (Kong/Nginx)      │
└──────┬───────┴──────┬───────┴──────┬───────┴────────┬───────────┘
       │              │              │                 │
┌──────▼──────────────▼──────────────▼─────────────────▼──────────┐
│                         Microservices Layer                       │
├──────────────┬───────────────┬───────────────┬──────────────────┤
│  Customer    │   Network     │   Billing     │   Support        │
│  Service     │   Service     │   Service     │   Service        │
│  :8001       │   :8002       │   :8003       │   :8004          │
└──────┬───────┴───────┬───────┴───────┬───────┴──────────────────┘
       │               │               │
┌──────▼───────────────▼───────────────▼──────────────────────────┐
│                        Data Layer                                 │
├──────────────┬───────────────┬───────────────┬──────────────────┤
│  PostgreSQL  │     Redis     │   InfluxDB    │   Elasticsearch   │
│  (Main DB)   │   (Cache)     │  (Metrics)    │   (Logs/Search)  │
└──────────────┴───────────────┴───────────────┴──────────────────┘
```

### Technology Stack

| Component | Technology | Purpose |
|-----------|-----------|---------|
| Frontend | React + TypeScript | Admin Panel, Customer Portal |
| API Gateway | Kong / Nginx | Rate limiting, Auth, Routing |
| Backend | FastAPI (Python) | Core business logic |
| WebSocket | Node.js + Socket.io | Real-time features |
| Database | PostgreSQL 15 | Main data storage |
| Cache | Redis 7 | Caching, Sessions |
| Message Queue | RabbitMQ / Redis | Async tasks |
| Monitoring | Prometheus + Grafana | Metrics |
| Logging | ELK Stack | Log management |
| Container | Docker + Kubernetes | Deployment |

---

## 2. Customer Management {#customers}

```python
# app/services/customer/customer_service.py
from typing import List, Optional, Dict
from uuid import UUID
from sqlalchemy.orm import Session
from sqlalchemy import text, or_
import logging

logger = logging.getLogger(__name__)


class CustomerService:
    """Service สำหรับจัดการ customers"""
    
    def __init__(self, db: Session):
        self.db = db
    
    def search_customers(
        self,
        query: str = None,
        status: str = None,
        customer_type: str = None,
        page: int = 1,
        per_page: int = 20
    ) -> Dict:
        """ค้นหา customers แบบ advanced"""
        
        base_query = """
            SELECT 
                c.*,
                COUNT(cs.id) FILTER (WHERE cs.status = 'active') as active_services,
                COUNT(i.id) FILTER (WHERE i.status = 'pending') as pending_invoices,
                SUM(i.total_amount) FILTER (WHERE i.status = 'pending') as outstanding_balance
            FROM customers c
            LEFT JOIN customer_services cs ON cs.customer_id = c.id
            LEFT JOIN invoices i ON i.customer_id = c.id
            WHERE 1=1
        """
        
        params = {}
        conditions = []
        
        if query:
            conditions.append("""(
                c.first_name ILIKE :query OR
                c.last_name ILIKE :query OR
                c.customer_code ILIKE :query OR
                c.email ILIKE :query OR
                c.phone ILIKE :query
            )""")
            params['query'] = f"%{query}%"
        
        if status:
            conditions.append("c.status = :status")
            params['status'] = status
        
        if customer_type:
            conditions.append("c.customer_type = :customer_type")
            params['customer_type'] = customer_type
        
        if conditions:
            base_query += " AND " + " AND ".join(conditions)
        
        base_query += " GROUP BY c.id ORDER BY c.created_at DESC"
        
        # Count total
        count_query = f"SELECT COUNT(*) FROM ({base_query}) sub"
        total = self.db.execute(text(count_query), params).scalar()
        
        # Paginate
        offset = (page - 1) * per_page
        base_query += f" LIMIT {per_page} OFFSET {offset}"
        
        customers = self.db.execute(text(base_query), params).fetchall()
        
        return {
            "customers": [dict(c._mapping) for c in customers],
            "total": total,
            "page": page,
            "per_page": per_page,
            "total_pages": -(-total // per_page)  # ceil division
        }
    
    def get_customer_details(self, customer_id: str) -> Dict:
        """ดึงข้อมูล customer แบบละเอียด"""
        customer = self.db.execute(text("""
            SELECT c.*,
                   COALESCE(json_agg(
                       json_build_object(
                           'id', cs.id,
                           'plan_name', sp.name,
                           'status', cs.status,
                           'username', cs.username,
                           'ip_address', cs.ip_address,
                           'start_date', cs.start_date
                       ) ORDER BY cs.created_at DESC
                   ) FILTER (WHERE cs.id IS NOT NULL), '[]') as services
            FROM customers c
            LEFT JOIN customer_services cs ON cs.customer_id = c.id
            LEFT JOIN service_plans sp ON sp.id = cs.plan_id
            WHERE c.id = :customer_id::UUID
            GROUP BY c.id
        """), {"customer_id": customer_id}).fetchone()
        
        if not customer:
            return None
        
        return dict(customer._mapping)
    
    def provision_new_customer(
        self,
        customer_data: Dict,
        service_data: Dict,
        router_id: str
    ) -> Dict:
        """Provision customer ใหม่ทั้งหมดในครั้งเดียว"""
        try:
            # 1. สร้าง customer
            customer_id = self._create_customer(customer_data)
            
            # 2. สร้าง PPPoE credentials
            username = self._generate_pppoe_username(customer_data)
            password = self._generate_password()
            
            # 3. สร้าง service record
            service_id = self._create_service(
                customer_id=customer_id,
                plan_id=service_data['plan_id'],
                username=username,
                router_id=router_id
            )
            
            # 4. Create PPPoE user ใน MikroTik
            self._provision_mikrotik_user(
                router_id=router_id,
                username=username,
                password=password,
                profile=service_data.get('profile', 'default')
            )
            
            # 5. สร้าง invoice แรก
            from app.services.billing.invoice_service import InvoiceService
            invoice_svc = InvoiceService(self.db)
            # invoice_svc.create_monthly_invoice(...)
            
            self.db.commit()
            
            return {
                "success": True,
                "customer_id": customer_id,
                "service_id": service_id,
                "username": username,
                "password": password  # แสดงครั้งเดียว
            }
        except Exception as e:
            self.db.rollback()
            logger.error(f"Failed to provision customer: {e}")
            raise
    
    def _generate_pppoe_username(self, customer_data: Dict) -> str:
        """Generate PPPoE username จาก customer data"""
        code = customer_data.get('customer_code', '').lower()
        return f"user_{code}"
    
    def _generate_password(self, length: int = 12) -> str:
        """Generate secure password"""
        import secrets
        import string
        alphabet = string.ascii_letters + string.digits
        return ''.join(secrets.choice(alphabet) for _ in range(length))
    
    def _provision_mikrotik_user(
        self,
        router_id: str,
        username: str,
        password: str,
        profile: str
    ):
        """สร้าง PPPoE user ใน MikroTik"""
        from app.services.mikrotik import router_manager
        
        with router_manager.get_connection(router_id) as conn:
            conn.run_command('/ppp/secret/add', **{
                '=name': username,
                '=password': password,
                '=service': 'pppoe',
                '=profile': profile,
                '=comment': f"Auto-provisioned"
            })
```

---

## 3. Automated Provisioning {#provisioning}

```python
# app/services/provisioning/provisioner.py
from typing import Dict, Optional
import logging
from enum import Enum

logger = logging.getLogger(__name__)


class ProvisioningStatus(str, Enum):
    PENDING = "pending"
    IN_PROGRESS = "in_progress"
    COMPLETED = "completed"
    FAILED = "failed"
    ROLLBACK = "rollback"


class AutoProvisioner:
    """Automated provisioning สำหรับ ISP services"""
    
    def __init__(self, router_manager, db_session):
        self.router_manager = router_manager
        self.db = db_session
    
    def provision_pppoe_service(
        self,
        customer_id: str,
        router_id: str,
        plan_code: str,
        username: str,
        password: str
    ) -> Dict:
        """Provision PPPoE service แบบครบวงจร"""
        
        steps = []
        
        try:
            # Step 1: ตรวจสอบ plan
            plan = self._get_plan(plan_code)
            steps.append({"step": "get_plan", "status": "done"})
            
            # Step 2: หา IP Pool
            ip_pool = self._find_ip_pool(router_id, plan)
            steps.append({"step": "find_ip_pool", "status": "done"})
            
            # Step 3: สร้าง PPPoE Profile ถ้าจำเป็น
            profile_name = self._ensure_pppoe_profile(router_id, plan)
            steps.append({"step": "ensure_profile", "status": "done"})
            
            # Step 4: สร้าง PPPoE Secret
            self._create_pppoe_secret(router_id, username, password, profile_name)
            steps.append({"step": "create_pppoe_secret", "status": "done"})
            
            # Step 5: ตั้งค่า Queue (ถ้ามี)
            if plan.get('speed_limit'):
                self._setup_simple_queue(router_id, username, plan)
                steps.append({"step": "setup_queue", "status": "done"})
            
            # Step 6: อัพเดต database
            self._update_service_status(customer_id, "active")
            steps.append({"step": "update_database", "status": "done"})
            
            logger.info(f"Provisioning completed for {username}")
            return {
                "status": ProvisioningStatus.COMPLETED,
                "username": username,
                "profile": profile_name,
                "steps": steps
            }
            
        except Exception as e:
            logger.error(f"Provisioning failed for {username}: {e}")
            # Rollback
            self._rollback_provisioning(router_id, username, steps)
            return {
                "status": ProvisioningStatus.FAILED,
                "error": str(e),
                "steps": steps
            }
    
    def _get_plan(self, plan_code: str) -> Dict:
        from sqlalchemy import text
        plan = self.db.execute(
            text("SELECT * FROM service_plans WHERE code = :code"),
            {"code": plan_code}
        ).fetchone()
        
        if not plan:
            raise ValueError(f"Plan {plan_code} not found")
        return dict(plan._mapping)
    
    def _ensure_pppoe_profile(self, router_id: str, plan: Dict) -> str:
        """สร้าง PPPoE Profile ถ้ายังไม่มี"""
        profile_name = f"plan_{plan['code']}"
        
        with self.router_manager.get_connection(router_id) as conn:
            existing = conn.run_command('/ppp/profile/print', **{'?name': profile_name})
            
            if not existing:
                # สร้าง profile ใหม่
                rate_limit = (
                    f"{plan['download_speed_mbps']}M/{plan['upload_speed_mbps']}M"
                )
                
                conn.run_command('/ppp/profile/add', **{
                    '=name': profile_name,
                    '=rate-limit': rate_limit,
                    '=local-address': '10.0.0.1',
                    '=dns-server': '8.8.8.8,8.8.4.4'
                })
        
        return profile_name
    
    def _create_pppoe_secret(
        self,
        router_id: str,
        username: str,
        password: str,
        profile: str
    ):
        """สร้าง PPPoE secret"""
        with self.router_manager.get_connection(router_id) as conn:
            conn.run_command('/ppp/secret/add', **{
                '=name': username,
                '=password': password,
                '=service': 'pppoe',
                '=profile': profile
            })
    
    def _rollback_provisioning(self, router_id: str, username: str, steps: List):
        """Rollback การ provision"""
        completed_steps = [s['step'] for s in steps if s['status'] == 'done']
        
        if 'create_pppoe_secret' in completed_steps:
            try:
                with self.router_manager.get_connection(router_id) as conn:
                    secrets = conn.run_command('/ppp/secret/print', **{'?name': username})
                    for secret in secrets:
                        conn.run_command('/ppp/secret/remove', **{'=.id': secret.get('.id')})
                logger.info(f"Rolled back PPPoE secret for {username}")
            except Exception as e:
                logger.error(f"Rollback failed: {e}")
```

---

## 4. NOC Dashboard {#noc}

```python
# app/services/noc/dashboard_service.py
from datetime import datetime, timedelta
from typing import Dict, List
from sqlalchemy.orm import Session
from sqlalchemy import text


class NOCDashboardService:
    """Service สำหรับ NOC Dashboard data"""
    
    def __init__(self, db: Session, redis_client, router_manager):
        self.db = db
        self.redis = redis_client
        self.router_manager = router_manager
    
    def get_summary(self) -> Dict:
        """ดึงข้อมูล summary สำหรับ NOC Dashboard"""
        
        # Cache สำหรับ 30 วินาที
        cache_key = "noc:summary"
        cached = self.redis.get_json(cache_key)
        if cached:
            return cached
        
        summary = {
            "routers": self._get_router_summary(),
            "customers": self._get_customer_summary(),
            "traffic": self._get_traffic_summary(),
            "alerts": self._get_active_alerts(),
            "top_talkers": self._get_top_talkers(),
            "generated_at": datetime.utcnow().isoformat()
        }
        
        self.redis.set(cache_key, summary, ttl=30)
        return summary
    
    def _get_router_summary(self) -> Dict:
        result = self.db.execute(text("""
            SELECT 
                COUNT(*) as total,
                COUNT(*) FILTER (WHERE last_check_status = 'online') as online,
                COUNT(*) FILTER (WHERE last_check_status = 'offline') as offline,
                COUNT(*) FILTER (WHERE last_check_status = 'unknown') as unknown
            FROM routers WHERE is_active = true
        """)).fetchone()
        
        return dict(result._mapping)
    
    def _get_customer_summary(self) -> Dict:
        result = self.db.execute(text("""
            SELECT
                COUNT(*) as total,
                COUNT(*) FILTER (WHERE status = 'active') as active,
                COUNT(*) FILTER (WHERE status = 'suspended') as suspended,
                COUNT(DISTINCT cs.id) FILTER (WHERE cs.status = 'active') as active_services
            FROM customers c
            LEFT JOIN customer_services cs ON cs.customer_id = c.id
        """)).fetchone()
        
        return dict(result._mapping)
    
    def _get_active_alerts(self) -> List[Dict]:
        alerts = self.db.execute(text("""
            SELECT id, router_id, severity, event_type, title, created_at
            FROM system_events
            WHERE is_acknowledged = false
            AND severity IN ('warning', 'critical')
            ORDER BY 
                CASE severity WHEN 'critical' THEN 1 ELSE 2 END,
                created_at DESC
            LIMIT 20
        """)).fetchall()
        
        return [dict(a._mapping) for a in alerts]
    
    def get_router_health(self, router_id: str) -> Dict:
        """ดึงข้อมูล health ของ router"""
        try:
            with self.router_manager.get_connection(router_id) as conn:
                resources = conn.get_system_resources()
                interfaces = conn.get_interfaces()
                
                return {
                    "router_id": router_id,
                    "status": "online",
                    "cpu_load": resources.get('cpu-load', '0'),
                    "memory_percent": self._calc_memory_percent(resources),
                    "uptime": resources.get('uptime'),
                    "interface_count": len(interfaces),
                    "down_interfaces": len([i for i in interfaces if i.get('running') == 'false'])
                }
        except Exception as e:
            return {
                "router_id": router_id,
                "status": "offline",
                "error": str(e)
            }
    
    def _calc_memory_percent(self, resources: Dict) -> float:
        total = int(resources.get('total-memory', 1))
        free = int(resources.get('free-memory', 0))
        return round((total - free) / total * 100, 1)
```

---

## 5. SLA Management {#sla}

```python
# app/services/sla/sla_service.py
from datetime import datetime, timedelta
from decimal import Decimal
from typing import Dict, List
import logging

logger = logging.getLogger(__name__)


class SLADefinition:
    """กำหนด SLA targets"""
    
    # Default SLA targets
    TARGETS = {
        'residential': {
            'uptime_percent': 99.0,      # 99% uptime
            'response_time_hours': 24,    # 24 ชั่วโมง
            'resolution_time_hours': 72   # 72 ชั่วโมง
        },
        'business': {
            'uptime_percent': 99.5,
            'response_time_hours': 4,
            'resolution_time_hours': 8
        },
        'enterprise': {
            'uptime_percent': 99.9,
            'response_time_hours': 1,
            'resolution_time_hours': 4
        }
    }


class SLATracker:
    """ติดตาม SLA compliance"""
    
    def __init__(self, db, redis_client):
        self.db = db
        self.redis = redis_client
    
    def calculate_uptime(
        self,
        router_id: str,
        start_date: datetime,
        end_date: datetime
    ) -> Dict:
        """คำนวณ uptime percentage"""
        from sqlalchemy import text
        
        # ดึง downtime events
        downtime_events = self.db.execute(text("""
            SELECT created_at, resolved_at
            FROM system_events
            WHERE router_id = :router_id::UUID
            AND event_type = 'router_offline'
            AND created_at BETWEEN :start_date AND :end_date
            ORDER BY created_at
        """), {
            "router_id": router_id,
            "start_date": start_date,
            "end_date": end_date
        }).fetchall()
        
        total_seconds = (end_date - start_date).total_seconds()
        downtime_seconds = 0
        
        for event in downtime_events:
            resolved = event.resolved_at or end_date
            downtime = (min(resolved, end_date) - max(event.created_at, start_date)).total_seconds()
            downtime_seconds += max(0, downtime)
        
        uptime_seconds = total_seconds - downtime_seconds
        uptime_percent = (uptime_seconds / total_seconds) * 100 if total_seconds > 0 else 0
        
        return {
            "router_id": router_id,
            "period": {
                "start": start_date.isoformat(),
                "end": end_date.isoformat(),
                "total_hours": total_seconds / 3600
            },
            "uptime": {
                "seconds": uptime_seconds,
                "hours": uptime_seconds / 3600,
                "percent": round(uptime_percent, 4)
            },
            "downtime": {
                "seconds": downtime_seconds,
                "hours": downtime_seconds / 3600,
                "incidents": len(downtime_events)
            },
            "sla_met": uptime_percent >= 99.0  # Default target
        }
    
    def generate_sla_report(self, month: datetime) -> List[Dict]:
        """สร้าง SLA report รายเดือน"""
        from sqlalchemy import text
        
        start_date = month.replace(day=1, hour=0, minute=0, second=0)
        end_date = (start_date + timedelta(days=32)).replace(day=1) - timedelta(seconds=1)
        
        routers = self.db.execute(text("""
            SELECT id, name, location FROM routers WHERE is_active = true
        """)).fetchall()
        
        report = []
        for router in routers:
            uptime_data = self.calculate_uptime(
                str(router.id), start_date, end_date
            )
            
            report.append({
                "router_name": router.name,
                "location": router.location,
                **uptime_data
            })
        
        return report
```

---

## 6. Lab: ISP Platform Deployment {#lab}

### Complete Deployment

```bash
#!/bin/bash
# deploy_isp_platform.sh

echo "=== Deploying ISP Platform ==="

# 1. Prerequisites check
check_prerequisites() {
    command -v docker >/dev/null || { echo "Docker required"; exit 1; }
    command -v docker-compose >/dev/null || { echo "docker-compose required"; exit 1; }
    [ -f ".env" ] || { echo ".env file required"; exit 1; }
    echo "Prerequisites OK"
}

# 2. Build all images
build_images() {
    docker-compose build --no-cache
    echo "Images built"
}

# 3. Initialize database
init_database() {
    docker-compose up -d postgres redis
    sleep 10  # รอ postgres
    docker-compose run --rm api alembic upgrade head
    echo "Database initialized"
}

# 4. Start all services
start_services() {
    docker-compose up -d
    echo "Services started"
    docker-compose ps
}

# Main
check_prerequisites
build_images
init_database
start_services

echo "=== ISP Platform is running ==="
echo "Admin Panel: http://localhost/admin"
echo "API Docs: http://localhost/api/docs"
echo "Grafana: http://localhost:3000"
```

### Verification Checklist

- [ ] All microservices start and are healthy
- [ ] Customer CRUD operations work
- [ ] PPPoE provisioning creates users in MikroTik
- [ ] Invoice generation works
- [ ] Suspension/activation workflow tested
- [ ] NOC dashboard shows real-time data
- [ ] SLA reports generate correctly
- [ ] Email notifications sent successfully
- [ ] API documentation accessible

> **Tip:** เริ่มจาก monolith ก่อน แล้วค่อย split เป็น microservices เมื่อ scale จำเป็น

---

## Summary

Part นี้ครอบคลุม:
- **Complete ISP platform** architecture
- **Customer management** with provisioning
- **Automated provisioning** ด้วย MikroTik API
- **NOC Dashboard** แบบ real-time
- **SLA tracking** และ reporting

---

[← Part 66: Billing System](part-066-billing-system.md) | [Part 68: Inventory System →](part-068-inventory-system.md)
