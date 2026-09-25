# Part 63: Database Integration สำหรับ Network Data

## สารบัญ
1. [Database Selection](#db-selection)
2. [Schema Design](#schema)
3. [Connection Pooling](#pooling)
4. [Historical Data Storage](#historical)
5. [Data Retention Policies](#retention)
6. [Query Optimization](#optimization)
7. [ORM Usage](#orm)
8. [Backup Strategies](#backup)
9. [Migration Management](#migration)
10. [Lab: Network Data Warehouse](#lab)

---

## 1. Database Selection {#db-selection}

### เปรียบเทียบ Database Options

| Feature | MySQL/MariaDB | PostgreSQL | SQLite | InfluxDB | TimescaleDB |
|---------|--------------|------------|--------|----------|-------------|
| ใช้งานทั่วไป | ✓ | ✓ | ✓ (dev only) | ✗ | ✓ |
| Time-series | ปานกลาง | ดี | ไม่แนะนำ | ดีมาก | ดีมาก |
| JSON support | ✓ | ✓✓ | ✓ | N/A | ✓✓ |
| Scaling | ปานกลาง | ดี | ไม่ดี | ดีมาก | ดีมาก |
| ISP use case | ดี | ดีมาก | ไม่แนะนำ | สำหรับ metrics | ดีมาก |

### แนะนำ Architecture

```
┌─────────────────────────────────────────────────────────┐
│                  Application Layer                        │
└────────┬────────────┬────────────────┬──────────────────┘
         │            │                │
         ▼            ▼                ▼
┌──────────────┐ ┌──────────┐ ┌──────────────┐
│  PostgreSQL  │ │  Redis   │ │  InfluxDB    │
│ (Main Data)  │ │ (Cache)  │ │ (Metrics)    │
│              │ │          │ │              │
│ - Customers  │ │ - Sessions│ │ - Traffic    │
│ - Routers    │ │ - Tokens │ │ - CPU/Mem    │
│ - Config     │ │ - Live   │ │ - SNMP       │
│ - Billing    │ │   data   │ │ - Alerts     │
└──────────────┘ └──────────┘ └──────────────┘
```

---

## 2. Schema Design {#schema}

### PostgreSQL Schema

```sql
-- ============================================
-- Database Schema สำหรับ MikroTik Management
-- ============================================

-- Extensions
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";  -- สำหรับ full-text search
CREATE EXTENSION IF NOT EXISTS "pgcrypto";  -- สำหรับ encryption

-- ============================================
-- Users and Authentication
-- ============================================
CREATE TABLE users (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    username VARCHAR(50) UNIQUE NOT NULL,
    email VARCHAR(255) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) NOT NULL DEFAULT 'viewer' 
        CHECK (role IN ('admin', 'network_admin', 'operator', 'viewer')),
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_login TIMESTAMP WITH TIME ZONE,
    failed_login_attempts INT DEFAULT 0,
    locked_until TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_users_username ON users(username);
CREATE INDEX idx_users_email ON users(email);

-- ============================================
-- Routers
-- ============================================
CREATE TABLE routers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) NOT NULL,
    host INET NOT NULL,
    api_port INT NOT NULL DEFAULT 8728,
    use_ssl BOOLEAN NOT NULL DEFAULT FALSE,
    username VARCHAR(50) NOT NULL,
    -- password เก็บแบบ encrypted
    password_encrypted BYTEA NOT NULL,
    location VARCHAR(200),
    description TEXT,
    tags JSONB DEFAULT '[]',
    is_active BOOLEAN NOT NULL DEFAULT TRUE,
    last_seen TIMESTAMP WITH TIME ZONE,
    last_check_status VARCHAR(20) DEFAULT 'unknown'
        CHECK (last_check_status IN ('online', 'offline', 'unknown')),
    router_identity VARCHAR(100),
    ros_version VARCHAR(50),
    board_name VARCHAR(100),
    created_by UUID REFERENCES users(id),
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_routers_host ON routers(host);
CREATE INDEX idx_routers_status ON routers(last_check_status);
CREATE INDEX idx_routers_tags ON routers USING gin(tags);

-- ============================================
-- Router Interfaces
-- ============================================
CREATE TABLE router_interfaces (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    router_id UUID NOT NULL REFERENCES routers(id) ON DELETE CASCADE,
    name VARCHAR(100) NOT NULL,
    type VARCHAR(50),
    mac_address MACADDR,
    mtu INT,
    is_disabled BOOLEAN DEFAULT FALSE,
    comment TEXT,
    last_updated TIMESTAMP WITH TIME ZONE DEFAULT NOW(),
    
    UNIQUE(router_id, name)
);

CREATE INDEX idx_interfaces_router ON router_interfaces(router_id);

-- ============================================
-- IP Addresses
-- ============================================
CREATE TABLE ip_addresses (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    router_id UUID NOT NULL REFERENCES routers(id) ON DELETE CASCADE,
    interface_id UUID REFERENCES router_interfaces(id),
    address CIDR NOT NULL,
    network CIDR,
    comment TEXT,
    is_disabled BOOLEAN DEFAULT FALSE,
    is_dynamic BOOLEAN DEFAULT FALSE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_ip_addresses_router ON ip_addresses(router_id);
CREATE INDEX idx_ip_addresses_address ON ip_addresses(address);

-- ============================================
-- Customers (ISP)
-- ============================================
CREATE TABLE customers (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_code VARCHAR(20) UNIQUE NOT NULL,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    company_name VARCHAR(200),
    email VARCHAR(255),
    phone VARCHAR(20),
    address TEXT,
    national_id VARCHAR(20),
    customer_type VARCHAR(20) DEFAULT 'residential'
        CHECK (customer_type IN ('residential', 'business', 'enterprise')),
    status VARCHAR(20) DEFAULT 'active'
        CHECK (status IN ('active', 'suspended', 'terminated')),
    notes TEXT,
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_customers_code ON customers(customer_code);
CREATE INDEX idx_customers_email ON customers(email);
CREATE INDEX idx_customers_status ON customers(status);
CREATE INDEX idx_customers_name ON customers 
    USING gin(to_tsvector('english', first_name || ' ' || last_name));

-- ============================================
-- Service Plans
-- ============================================
CREATE TABLE service_plans (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    name VARCHAR(100) NOT NULL,
    code VARCHAR(20) UNIQUE NOT NULL,
    download_speed_mbps INT NOT NULL,
    upload_speed_mbps INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL,
    setup_fee DECIMAL(10, 2) DEFAULT 0,
    billing_cycle VARCHAR(20) DEFAULT 'monthly'
        CHECK (billing_cycle IN ('monthly', 'quarterly', 'annually')),
    data_limit_gb INT,  -- NULL = unlimited
    is_active BOOLEAN DEFAULT TRUE,
    features JSONB DEFAULT '{}',
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

-- ============================================
-- Customer Services / Subscriptions
-- ============================================
CREATE TABLE customer_services (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    customer_id UUID NOT NULL REFERENCES customers(id),
    plan_id UUID NOT NULL REFERENCES service_plans(id),
    router_id UUID REFERENCES routers(id),
    username VARCHAR(100),  -- PPPoE/Hotspot username
    password_encrypted BYTEA,
    ip_address INET,
    ip_pool VARCHAR(100),
    start_date DATE NOT NULL,
    end_date DATE,
    status VARCHAR(20) DEFAULT 'active'
        CHECK (status IN ('active', 'suspended', 'cancelled', 'pending')),
    suspend_reason VARCHAR(200),
    notes TEXT,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    updated_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_services_customer ON customer_services(customer_id);
CREATE INDEX idx_services_router ON customer_services(router_id);
CREATE INDEX idx_services_status ON customer_services(status);
CREATE INDEX idx_services_username ON customer_services(username);

-- ============================================
-- Traffic Statistics (Partitioned by month)
-- ============================================
CREATE TABLE traffic_stats (
    id BIGSERIAL,
    router_id UUID NOT NULL,
    interface_name VARCHAR(100) NOT NULL,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL,
    rx_bytes BIGINT DEFAULT 0,
    tx_bytes BIGINT DEFAULT 0,
    rx_packets BIGINT DEFAULT 0,
    tx_packets BIGINT DEFAULT 0,
    rx_errors INT DEFAULT 0,
    tx_errors INT DEFAULT 0,
    rx_drops INT DEFAULT 0,
    tx_drops INT DEFAULT 0,
    
    PRIMARY KEY (id, timestamp)
) PARTITION BY RANGE (timestamp);

-- สร้าง partitions สำหรับแต่ละเดือน
CREATE TABLE traffic_stats_2024_01 
    PARTITION OF traffic_stats 
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE traffic_stats_2024_02 
    PARTITION OF traffic_stats 
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- Function สร้าง partition อัตโนมัติ
CREATE OR REPLACE FUNCTION create_traffic_partition(target_date DATE)
RETURNS VOID AS $$
DECLARE
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    start_date := DATE_TRUNC('month', target_date);
    end_date := start_date + INTERVAL '1 month';
    partition_name := 'traffic_stats_' || TO_CHAR(start_date, 'YYYY_MM');
    
    IF NOT EXISTS (
        SELECT 1 FROM pg_tables 
        WHERE tablename = partition_name
    ) THEN
        EXECUTE FORMAT(
            'CREATE TABLE %I PARTITION OF traffic_stats FOR VALUES FROM (%L) TO (%L)',
            partition_name,
            start_date,
            end_date
        );
        
        EXECUTE FORMAT(
            'CREATE INDEX ON %I (router_id, timestamp)',
            partition_name
        );
        
        RAISE NOTICE 'Created partition: %', partition_name;
    END IF;
END;
$$ LANGUAGE plpgsql;

CREATE INDEX idx_traffic_router_time ON traffic_stats(router_id, timestamp);
CREATE INDEX idx_traffic_interface ON traffic_stats(interface_name);

-- ============================================
-- Firewall Rules History
-- ============================================
CREATE TABLE firewall_rule_history (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    router_id UUID NOT NULL REFERENCES routers(id),
    action_type VARCHAR(20) NOT NULL 
        CHECK (action_type IN ('create', 'update', 'delete', 'enable', 'disable')),
    rule_data JSONB NOT NULL,
    changed_by UUID REFERENCES users(id),
    change_reason TEXT,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
);

CREATE INDEX idx_fw_history_router ON firewall_rule_history(router_id);
CREATE INDEX idx_fw_history_time ON firewall_rule_history(timestamp);

-- ============================================
-- System Events / Alerts
-- ============================================
CREATE TABLE system_events (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    router_id UUID REFERENCES routers(id),
    severity VARCHAR(20) NOT NULL 
        CHECK (severity IN ('info', 'warning', 'critical', 'resolved')),
    event_type VARCHAR(100) NOT NULL,
    title VARCHAR(200) NOT NULL,
    description TEXT,
    details JSONB DEFAULT '{}',
    is_acknowledged BOOLEAN DEFAULT FALSE,
    acknowledged_by UUID REFERENCES users(id),
    acknowledged_at TIMESTAMP WITH TIME ZONE,
    created_at TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW(),
    resolved_at TIMESTAMP WITH TIME ZONE
);

CREATE INDEX idx_events_severity ON system_events(severity);
CREATE INDEX idx_events_router ON system_events(router_id);
CREATE INDEX idx_events_time ON system_events(created_at DESC);
CREATE INDEX idx_events_unack ON system_events(is_acknowledged) 
    WHERE is_acknowledged = FALSE;

-- ============================================
-- Audit Log
-- ============================================
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    user_id UUID REFERENCES users(id),
    action VARCHAR(100) NOT NULL,
    resource_type VARCHAR(50),
    resource_id VARCHAR(100),
    changes JSONB,
    ip_address INET,
    user_agent TEXT,
    timestamp TIMESTAMP WITH TIME ZONE NOT NULL DEFAULT NOW()
) PARTITION BY RANGE (timestamp);

-- ============================================
-- Functions and Triggers
-- ============================================

-- Auto-update updated_at
CREATE OR REPLACE FUNCTION update_updated_at()
RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER update_routers_updated_at
    BEFORE UPDATE ON routers
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

CREATE TRIGGER update_customers_updated_at
    BEFORE UPDATE ON customers
    FOR EACH ROW EXECUTE FUNCTION update_updated_at();

-- Views สำหรับ reporting
CREATE VIEW customer_summary AS
SELECT 
    c.id,
    c.customer_code,
    c.first_name || ' ' || c.last_name AS full_name,
    c.customer_type,
    c.status,
    COUNT(cs.id) AS service_count,
    SUM(sp.price) AS total_monthly_revenue
FROM customers c
LEFT JOIN customer_services cs ON cs.customer_id = c.id AND cs.status = 'active'
LEFT JOIN service_plans sp ON sp.id = cs.plan_id
GROUP BY c.id, c.customer_code, c.first_name, c.last_name, c.customer_type, c.status;
```

---

## 3. Connection Pooling {#pooling}

### Python SQLAlchemy Connection Pool

```python
# app/database.py
from sqlalchemy import create_engine, event, text
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
from sqlalchemy.pool import QueuePool
from typing import Generator
import logging
import time

from app.config import settings

logger = logging.getLogger(__name__)

# สร้าง engine พร้อม connection pool
engine = create_engine(
    settings.DATABASE_URL,
    poolclass=QueuePool,
    pool_size=20,              # จำนวน connections ใน pool
    max_overflow=10,           # connections เพิ่มเติมได้อีก 10
    pool_timeout=30,           # รอ connection ได้สูงสุด 30 วินาที
    pool_recycle=3600,         # Recycle connection ทุก 1 ชั่วโมง
    pool_pre_ping=True,        # ตรวจสอบ connection ก่อนใช้
    echo=settings.DEBUG,       # Log SQL ใน debug mode
    connect_args={
        "connect_timeout": 10,
        "application_name": "mikrotik-api",
        "options": "-c statement_timeout=30000"  # 30s query timeout
    }
)

# Session factory
SessionLocal = sessionmaker(
    autocommit=False,
    autoflush=False,
    bind=engine
)

Base = declarative_base()


# Dependency สำหรับ FastAPI
def get_db() -> Generator[Session, None, None]:
    """Get database session"""
    db = SessionLocal()
    try:
        yield db
    except Exception as e:
        logger.error(f"Database error: {e}")
        db.rollback()
        raise
    finally:
        db.close()


# Event listeners สำหรับ monitoring
@event.listens_for(engine, "checkout")
def receive_checkout(dbapi_connection, connection_record, connection_proxy):
    """Log connection checkout"""
    connection_record.checkout_timestamp = time.time()


@event.listens_for(engine, "checkin")
def receive_checkin(dbapi_connection, connection_record):
    """Log connection checkin and duration"""
    if hasattr(connection_record, 'checkout_timestamp'):
        duration = time.time() - connection_record.checkout_timestamp
        if duration > 5:  # Log slow connections
            logger.warning(f"Long connection duration: {duration:.2f}s")


# Health check function
def check_database_connection() -> bool:
    """ตรวจสอบว่า database เชื่อมต่อได้"""
    try:
        with engine.connect() as conn:
            conn.execute(text("SELECT 1"))
        return True
    except Exception as e:
        logger.error(f"Database health check failed: {e}")
        return False
```

---

## 4. Historical Data Storage {#historical}

### Storing Traffic Data Efficiently

```python
# app/services/data_storage.py
from sqlalchemy.orm import Session
from sqlalchemy import text, func
from datetime import datetime, timedelta
from typing import List, Dict, Optional
import logging
from dataclasses import dataclass

logger = logging.getLogger(__name__)


@dataclass
class TrafficRecord:
    router_id: str
    interface_name: str
    timestamp: datetime
    rx_bytes: int
    tx_bytes: int
    rx_packets: int
    tx_packets: int


class TrafficDataStorage:
    """Service สำหรับจัดการ traffic data storage"""
    
    def __init__(self, db: Session):
        self.db = db
    
    def bulk_insert_traffic(self, records: List[TrafficRecord]) -> int:
        """Insert traffic records แบบ bulk สำหรับ performance"""
        if not records:
            return 0
        
        # ตรวจสอบว่า partition มีอยู่ก่อน insert
        dates = {r.timestamp.date() for r in records}
        for date in dates:
            self.db.execute(
                text("SELECT create_traffic_partition(:date)"),
                {"date": date}
            )
        
        # Bulk insert ด้วย COPY หรือ batch INSERT
        data = [
            {
                "router_id": r.router_id,
                "interface_name": r.interface_name,
                "timestamp": r.timestamp,
                "rx_bytes": r.rx_bytes,
                "tx_bytes": r.tx_bytes,
                "rx_packets": r.rx_packets,
                "tx_packets": r.tx_packets
            }
            for r in records
        ]
        
        self.db.execute(
            text("""
                INSERT INTO traffic_stats 
                    (router_id, interface_name, timestamp, rx_bytes, tx_bytes, rx_packets, tx_packets)
                VALUES 
                    (:router_id, :interface_name, :timestamp, :rx_bytes, :tx_bytes, :rx_packets, :tx_packets)
                ON CONFLICT DO NOTHING
            """),
            data
        )
        self.db.commit()
        
        return len(records)
    
    def get_hourly_traffic(
        self, 
        router_id: str, 
        interface_name: str,
        start_time: datetime,
        end_time: datetime
    ) -> List[Dict]:
        """ดึงข้อมูล traffic แบบ hourly aggregated"""
        result = self.db.execute(
            text("""
                SELECT 
                    DATE_TRUNC('hour', timestamp) AS hour,
                    SUM(rx_bytes) AS total_rx_bytes,
                    SUM(tx_bytes) AS total_tx_bytes,
                    AVG(rx_bytes) AS avg_rx_bytes,
                    AVG(tx_bytes) AS avg_tx_bytes,
                    MAX(rx_bytes) AS max_rx_bytes,
                    MAX(tx_bytes) AS max_tx_bytes
                FROM traffic_stats
                WHERE 
                    router_id = :router_id
                    AND interface_name = :interface_name
                    AND timestamp BETWEEN :start_time AND :end_time
                GROUP BY DATE_TRUNC('hour', timestamp)
                ORDER BY hour
            """),
            {
                "router_id": router_id,
                "interface_name": interface_name,
                "start_time": start_time,
                "end_time": end_time
            }
        )
        
        return [dict(row) for row in result.fetchall()]
    
    def get_top_interfaces_by_traffic(
        self,
        router_id: str,
        hours: int = 24,
        limit: int = 10
    ) -> List[Dict]:
        """ดึง top interfaces ตาม traffic"""
        cutoff = datetime.utcnow() - timedelta(hours=hours)
        
        result = self.db.execute(
            text("""
                SELECT 
                    interface_name,
                    SUM(rx_bytes + tx_bytes) AS total_bytes,
                    SUM(rx_bytes) AS total_rx,
                    SUM(tx_bytes) AS total_tx,
                    COUNT(*) AS sample_count
                FROM traffic_stats
                WHERE 
                    router_id = :router_id
                    AND timestamp > :cutoff
                GROUP BY interface_name
                ORDER BY total_bytes DESC
                LIMIT :limit
            """),
            {"router_id": router_id, "cutoff": cutoff, "limit": limit}
        )
        
        return [dict(row) for row in result.fetchall()]
```

---

## 5. Data Retention Policies {#retention}

```python
# app/services/retention.py
from sqlalchemy.orm import Session
from sqlalchemy import text
from datetime import datetime, timedelta
import logging
import schedule
import time
import threading

logger = logging.getLogger(__name__)


class DataRetentionService:
    """Service จัดการ data retention policy"""
    
    RETENTION_POLICIES = {
        'traffic_stats': {
            'raw': timedelta(days=7),          # Raw data: 7 วัน
            'hourly': timedelta(days=90),      # Hourly aggregated: 90 วัน
            'daily': timedelta(days=365),      # Daily aggregated: 1 ปี
            'monthly': timedelta(days=365*5)   # Monthly: 5 ปี
        },
        'system_events': {
            'info': timedelta(days=30),
            'warning': timedelta(days=90),
            'critical': timedelta(days=365)
        },
        'audit_log': timedelta(days=365),
        'firewall_rule_history': timedelta(days=90)
    }
    
    def __init__(self, db: Session):
        self.db = db
    
    def apply_traffic_retention(self):
        """ลบ traffic data ที่เก่าเกินกำหนด"""
        policy = self.RETENTION_POLICIES['traffic_stats']
        cutoff = datetime.utcnow() - policy['raw']
        
        # ลบ raw data เก่า
        result = self.db.execute(
            text("DELETE FROM traffic_stats WHERE timestamp < :cutoff"),
            {"cutoff": cutoff}
        )
        deleted_count = result.rowcount
        
        # ลบ partitions ที่ว่างเปล่า
        self._drop_empty_partitions()
        
        self.db.commit()
        logger.info(f"Deleted {deleted_count} old traffic records")
        return deleted_count
    
    def aggregate_traffic_data(self):
        """รวม raw data เป็น hourly และ daily aggregates"""
        # สร้าง hourly aggregated table ถ้ายังไม่มี
        self.db.execute(text("""
            CREATE TABLE IF NOT EXISTS traffic_hourly (
                router_id UUID NOT NULL,
                interface_name VARCHAR(100) NOT NULL,
                hour TIMESTAMP WITH TIME ZONE NOT NULL,
                rx_bytes BIGINT,
                tx_bytes BIGINT,
                max_rx_bps BIGINT,
                max_tx_bps BIGINT,
                avg_rx_bps BIGINT,
                avg_tx_bps BIGINT,
                PRIMARY KEY (router_id, interface_name, hour)
            )
        """))
        
        # Aggregate ข้อมูลเมื่อวาน
        yesterday = (datetime.utcnow() - timedelta(days=1)).date()
        
        self.db.execute(text("""
            INSERT INTO traffic_hourly 
                (router_id, interface_name, hour, rx_bytes, tx_bytes)
            SELECT 
                router_id,
                interface_name,
                DATE_TRUNC('hour', timestamp) as hour,
                SUM(rx_bytes),
                SUM(tx_bytes)
            FROM traffic_stats
            WHERE DATE(timestamp) = :date
            GROUP BY router_id, interface_name, DATE_TRUNC('hour', timestamp)
            ON CONFLICT (router_id, interface_name, hour) DO UPDATE SET
                rx_bytes = EXCLUDED.rx_bytes,
                tx_bytes = EXCLUDED.tx_bytes
        """), {"date": yesterday})
        
        self.db.commit()
    
    def _drop_empty_partitions(self):
        """ลบ partitions ที่ว่างเปล่า (ช่วยประหยัด disk space)"""
        result = self.db.execute(text("""
            SELECT 
                schemaname,
                tablename
            FROM pg_tables 
            WHERE tablename LIKE 'traffic_stats_%'
            ORDER BY tablename
        """))
        
        for row in result.fetchall():
            partition_name = row[1]
            count_result = self.db.execute(
                text(f"SELECT COUNT(*) FROM {partition_name}")
            ).fetchone()
            
            if count_result[0] == 0:
                # ตรวจสอบว่าเป็น partition เก่า (ไม่ใช่ current month)
                # ก่อน drop จริงๆ
                logger.info(f"Empty partition found: {partition_name}")


def start_retention_scheduler(db_session_factory):
    """เริ่ม scheduler สำหรับ data retention"""
    def run_retention():
        with db_session_factory() as db:
            service = DataRetentionService(db)
            service.apply_traffic_retention()
            service.aggregate_traffic_data()
    
    # รัน retention ทุกคืนเวลา 02:00
    schedule.every().day.at("02:00").do(run_retention)
    
    def scheduler_thread():
        while True:
            schedule.run_pending()
            time.sleep(60)
    
    thread = threading.Thread(target=scheduler_thread, daemon=True)
    thread.start()
    logger.info("Data retention scheduler started")
```

---

## 6. Query Optimization {#optimization}

```sql
-- ============================================
-- Query Optimization Examples
-- ============================================

-- 1. Explain analyze สำหรับหา slow queries
EXPLAIN (ANALYZE, BUFFERS, FORMAT TEXT)
SELECT 
    r.name as router_name,
    ts.interface_name,
    SUM(ts.rx_bytes + ts.tx_bytes) as total_bytes
FROM traffic_stats ts
JOIN routers r ON r.id = ts.router_id
WHERE ts.timestamp > NOW() - INTERVAL '24 hours'
GROUP BY r.name, ts.interface_name
ORDER BY total_bytes DESC;

-- 2. Partial index สำหรับ active customers
CREATE INDEX idx_active_customers 
    ON customers(customer_code, email)
    WHERE status = 'active';

-- 3. Covering index สำหรับ common queries
CREATE INDEX idx_services_lookup
    ON customer_services(customer_id, status, plan_id)
    INCLUDE (ip_address, username, start_date);

-- 4. BRIN index สำหรับ time-series data (compact กว่า B-tree)
CREATE INDEX idx_traffic_brin 
    ON traffic_stats USING BRIN (timestamp)
    WITH (pages_per_range = 128);

-- 5. Materialized view สำหรับ dashboard queries
CREATE MATERIALIZED VIEW daily_traffic_summary AS
SELECT 
    DATE(timestamp) as date,
    router_id,
    interface_name,
    SUM(rx_bytes) as daily_rx,
    SUM(tx_bytes) as daily_tx,
    MAX(rx_bytes) as peak_rx,
    MAX(tx_bytes) as peak_tx,
    COUNT(*) as sample_count
FROM traffic_stats
GROUP BY DATE(timestamp), router_id, interface_name
WITH DATA;

CREATE INDEX idx_daily_summary_date ON daily_traffic_summary(date, router_id);

-- Refresh materialized view
CREATE OR REPLACE FUNCTION refresh_daily_summary()
RETURNS void AS $$
BEGIN
    REFRESH MATERIALIZED VIEW CONCURRENTLY daily_traffic_summary;
END;
$$ LANGUAGE plpgsql;

-- 6. Vacuuming strategy
-- ตั้งค่า autovacuum สำหรับ high-write tables
ALTER TABLE traffic_stats SET (
    autovacuum_vacuum_scale_factor = 0.01,  -- Vacuum when 1% rows dead
    autovacuum_analyze_scale_factor = 0.005,
    autovacuum_vacuum_cost_delay = 10
);
```

### Python Query Optimization

```python
# app/repositories/traffic_repository.py
from sqlalchemy.orm import Session
from sqlalchemy import text, func, and_, select
from datetime import datetime, timedelta
from typing import List, Dict, Optional
import logging
from functools import lru_cache

logger = logging.getLogger(__name__)


class TrafficRepository:
    def __init__(self, db: Session):
        self.db = db
    
    def get_interface_summary(
        self,
        router_id: str,
        start_time: datetime,
        end_time: datetime
    ) -> List[Dict]:
        """Query พร้อม optimization"""
        
        # ใช้ prepared statement เพื่อ performance
        query = text("""
            SELECT 
                interface_name,
                SUM(rx_bytes)::BIGINT as total_rx,
                SUM(tx_bytes)::BIGINT as total_tx,
                MAX(rx_bytes)::BIGINT as peak_rx,
                MAX(tx_bytes)::BIGINT as peak_tx,
                AVG(rx_bytes)::BIGINT as avg_rx,
                AVG(tx_bytes)::BIGINT as avg_tx
            FROM traffic_stats
            WHERE 
                router_id = :router_id::UUID
                AND timestamp >= :start_time
                AND timestamp < :end_time
            GROUP BY interface_name
            ORDER BY total_rx + total_tx DESC
        """).bindparams(
            router_id=router_id,
            start_time=start_time,
            end_time=end_time
        )
        
        result = self.db.execute(query)
        return [dict(zip(result.keys(), row)) for row in result.fetchall()]
    
    def get_time_series(
        self,
        router_id: str,
        interface_name: str,
        start_time: datetime,
        end_time: datetime,
        resolution: str = '5 minutes'
    ) -> List[Dict]:
        """ดึง time series data แบบ downsampled"""
        
        # ใช้ time_bucket function (ถ้าใช้ TimescaleDB)
        # หรือ DATE_TRUNC สำหรับ PostgreSQL ปกติ
        
        valid_resolutions = {
            '1 minute': 'minute',
            '5 minutes': '5 minutes',
            '15 minutes': '15 minutes',
            '1 hour': 'hour',
            '1 day': 'day'
        }
        
        if resolution not in valid_resolutions:
            resolution = '5 minutes'
        
        query = text(f"""
            SELECT 
                DATE_TRUNC('{valid_resolutions[resolution]}', timestamp) as bucket,
                AVG(rx_bytes)::BIGINT as avg_rx,
                AVG(tx_bytes)::BIGINT as avg_tx,
                MAX(rx_bytes)::BIGINT as max_rx,
                MAX(tx_bytes)::BIGINT as max_tx
            FROM traffic_stats
            WHERE 
                router_id = :router_id::UUID
                AND interface_name = :interface_name
                AND timestamp BETWEEN :start_time AND :end_time
            GROUP BY DATE_TRUNC('{valid_resolutions[resolution]}', timestamp)
            ORDER BY bucket
        """)
        
        result = self.db.execute(query, {
            "router_id": router_id,
            "interface_name": interface_name,
            "start_time": start_time,
            "end_time": end_time
        })
        
        return [dict(zip(result.keys(), row)) for row in result.fetchall()]
```

---

## 7. ORM Usage with SQLAlchemy {#orm}

```python
# app/models/orm_models.py
from sqlalchemy import Column, String, Boolean, Integer, DateTime, ForeignKey, Text, ARRAY
from sqlalchemy.dialects.postgresql import UUID, INET, JSONB, CIDR, MACADDR
from sqlalchemy.orm import relationship, backref
from sqlalchemy.sql import func
import uuid

from app.database import Base


class Router(Base):
    __tablename__ = "routers"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    name = Column(String(100), nullable=False)
    host = Column(INET, nullable=False)
    api_port = Column(Integer, default=8728)
    use_ssl = Column(Boolean, default=False)
    username = Column(String(50), nullable=False)
    password_encrypted = Column(String(500))
    location = Column(String(200))
    description = Column(Text)
    is_active = Column(Boolean, default=True)
    last_seen = Column(DateTime(timezone=True))
    last_check_status = Column(String(20), default='unknown')
    ros_version = Column(String(50))
    board_name = Column(String(100))
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
    
    # Relationships
    interfaces = relationship("RouterInterface", back_populates="router", cascade="all, delete-orphan")
    ip_addresses = relationship("IPAddress", back_populates="router", cascade="all, delete-orphan")
    
    def to_dict(self):
        return {
            "id": str(self.id),
            "name": self.name,
            "host": str(self.host),
            "location": self.location,
            "status": self.last_check_status,
            "ros_version": self.ros_version,
            "board_name": self.board_name,
            "last_seen": self.last_seen.isoformat() if self.last_seen else None
        }
    
    def __repr__(self):
        return f"<Router {self.name} ({self.host})>"


class Customer(Base):
    __tablename__ = "customers"
    
    id = Column(UUID(as_uuid=True), primary_key=True, default=uuid.uuid4)
    customer_code = Column(String(20), unique=True, nullable=False)
    first_name = Column(String(100), nullable=False)
    last_name = Column(String(100), nullable=False)
    company_name = Column(String(200))
    email = Column(String(255))
    phone = Column(String(20))
    address = Column(Text)
    customer_type = Column(String(20), default='residential')
    status = Column(String(20), default='active')
    created_at = Column(DateTime(timezone=True), server_default=func.now())
    updated_at = Column(DateTime(timezone=True), server_default=func.now(), onupdate=func.now())
    
    # Relationships
    services = relationship("CustomerService", back_populates="customer")
    
    @property
    def full_name(self):
        return f"{self.first_name} {self.last_name}"
    
    def __repr__(self):
        return f"<Customer {self.customer_code}: {self.full_name}>"
```

---

## 8. Backup Strategies {#backup}

```bash
#!/bin/bash
# backup_database.sh

# Configuration
DB_HOST="${DB_HOST:-localhost}"
DB_PORT="${DB_PORT:-5432}"
DB_NAME="${DB_NAME:-mikrotik_db}"
DB_USER="${DB_USER:-postgres}"
BACKUP_DIR="/var/backups/postgresql"
S3_BUCKET="${S3_BUCKET:-s3://your-backup-bucket}"
RETENTION_DAYS=30

# สร้าง directory ถ้ายังไม่มี
mkdir -p "$BACKUP_DIR"

# Timestamp
TIMESTAMP=$(date +%Y%m%d_%H%M%S)
BACKUP_FILE="${BACKUP_DIR}/${DB_NAME}_${TIMESTAMP}.pgdump"

echo "Starting backup of ${DB_NAME}..."

# Full backup พร้อม compression
pg_dump \
    -h "$DB_HOST" \
    -p "$DB_PORT" \
    -U "$DB_USER" \
    -d "$DB_NAME" \
    --format=custom \
    --compress=9 \
    --no-password \
    --verbose \
    --file="$BACKUP_FILE"

if [ $? -eq 0 ]; then
    echo "Backup created: $BACKUP_FILE"
    
    # Upload to S3
    aws s3 cp "$BACKUP_FILE" "${S3_BUCKET}/daily/${DB_NAME}_${TIMESTAMP}.pgdump" \
        --storage-class STANDARD_IA
    
    echo "Uploaded to S3"
    
    # ลบ backup เก่าเกิน RETENTION_DAYS วัน
    find "$BACKUP_DIR" -name "*.pgdump" -mtime +"$RETENTION_DAYS" -delete
    
    # ลบ S3 backups เก่า
    CUTOFF_DATE=$(date -d "$RETENTION_DAYS days ago" +%Y-%m-%d)
    aws s3 ls "${S3_BUCKET}/daily/" | \
        awk '{print $4}' | \
        while read f; do
            FILE_DATE=$(echo $f | grep -oP '\d{8}')
            if [[ "$FILE_DATE" < "${CUTOFF_DATE//-/}" ]]; then
                aws s3 rm "${S3_BUCKET}/daily/$f"
            fi
        done
else
    echo "ERROR: Backup failed!"
    # ส่ง alert
    curl -X POST "${ALERT_WEBHOOK}" \
        -H "Content-Type: application/json" \
        -d '{"text": "Database backup FAILED for '"${DB_NAME}"'"}'
    exit 1
fi

echo "Backup completed successfully"
```

---

## 9. Migration Management {#migration}

```python
# alembic/env.py
from alembic import context
from sqlalchemy import engine_from_config, pool
from logging.config import fileConfig
import logging

from app.database import Base
from app.models.orm_models import *  # Import all models

# Alembic config
config = context.config
fileConfig(config.config_file_name)
logger = logging.getLogger('alembic.env')

target_metadata = Base.metadata


def run_migrations_offline():
    """Run migrations in offline mode"""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
        compare_type=True,
        compare_server_default=True
    )
    
    with context.begin_transaction():
        context.run_migrations()


def run_migrations_online():
    """Run migrations in online mode"""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )
    
    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
            compare_server_default=True
        )
        
        with context.begin_transaction():
            context.run_migrations()


if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

```bash
# Migration commands
# สร้าง migration ใหม่
alembic revision --autogenerate -m "Add customer table"

# รัน migrations
alembic upgrade head

# Rollback 1 version
alembic downgrade -1

# ดู migration history
alembic history --verbose

# ดู current version
alembic current
```

---

## 10. Lab: Network Data Warehouse {#lab}

### Lab Setup

```bash
# 1. Setup PostgreSQL
docker run -d \
    --name mikrotik-postgres \
    -e POSTGRES_DB=mikrotik_db \
    -e POSTGRES_USER=admin \
    -e POSTGRES_PASSWORD=securepassword \
    -p 5432:5432 \
    -v postgres_data:/var/lib/postgresql/data \
    postgres:15

# 2. Run migrations
export DATABASE_URL="postgresql://admin:securepassword@localhost/mikrotik_db"
alembic upgrade head

# 3. Insert test data
psql $DATABASE_URL -f tests/seed_data.sql

# 4. Test queries
psql $DATABASE_URL -c "SELECT COUNT(*) FROM routers;"
psql $DATABASE_URL -c "SELECT COUNT(*) FROM customers;"
```

### Performance Test

```python
# tests/test_db_performance.py
import pytest
import time
from datetime import datetime, timedelta
from sqlalchemy.orm import Session
from app.database import SessionLocal
from app.repositories.traffic_repository import TrafficRepository

def test_bulk_insert_performance():
    """ทดสอบ performance ของ bulk insert"""
    db = SessionLocal()
    repo = TrafficRepository(db)
    
    # สร้าง test data 10,000 records
    from app.services.data_storage import TrafficRecord
    records = []
    base_time = datetime.utcnow()
    
    for i in range(10000):
        records.append(TrafficRecord(
            router_id="test-router-id",
            interface_name="ether1",
            timestamp=base_time - timedelta(seconds=i),
            rx_bytes=1000 * i,
            tx_bytes=500 * i,
            rx_packets=i,
            tx_packets=i // 2
        ))
    
    start = time.time()
    count = repo.bulk_insert(records)
    duration = time.time() - start
    
    print(f"Inserted {count} records in {duration:.2f}s")
    print(f"Rate: {count/duration:.0f} records/second")
    
    assert count == 10000
    assert duration < 5  # ควรเสร็จภายใน 5 วินาที
    
    db.close()
```

### Verification Checklist

- [ ] Database schema created successfully
- [ ] All indexes created and verified with EXPLAIN
- [ ] Connection pooling configured
- [ ] Bulk insert performance > 5000 records/second
- [ ] Partitioning works correctly
- [ ] Data retention job runs without errors
- [ ] Backup script creates valid backup
- [ ] Restore from backup successful
- [ ] Query response time < 100ms for common queries

> **Tip:** ใช้ `pg_stat_statements` extension เพื่อ monitor slow queries ใน production

> **Warning:** อย่าลืมตั้ง `work_mem` และ `shared_buffers` ใน postgresql.conf ให้เหมาะสมกับ RAM

---

## Summary

Part นี้ครอบคลุม:
- **Database selection** สำหรับ network data
- **Schema design** พร้อม partitioning
- **Connection pooling** ด้วย SQLAlchemy
- **Data retention** policies
- **Query optimization** ด้วย indexes และ materialized views
- **Backup strategies** พร้อม S3 upload
- **Migration management** ด้วย Alembic

---

[← Part 62: WebSocket](part-062-websocket.md) | [Part 64: Redis Caching →](part-064-redis-caching.md)
