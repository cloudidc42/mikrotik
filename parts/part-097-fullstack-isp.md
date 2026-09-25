# Part 97: Full-stack ISP Platform

## สารบัญ
1. [Platform Architecture](#architecture)
2. [Backend API (FastAPI)](#backend)
3. [Database Layer](#database)
4. [Frontend Dashboard](#frontend)
5. [Microservices Integration](#microservices)
6. [Lab: ISP Platform Deployment](#lab)

---

## 1. Platform Architecture {#architecture}

```
[React Frontend]
    │ HTTP/WebSocket
    ↓
[FastAPI Gateway] ←── Auth (JWT)
    │
    ├── [Router Service] ──── librouteros ──── MikroTik Routers
    ├── [Customer Service] ── PostgreSQL ───── Customer DB
    ├── [Billing Service] ─── Stripe API ───── Payment
    ├── [Monitor Service] ─── InfluxDB ──────── Metrics
    └── [Alert Service] ───── Redis Pub/Sub ─── Notifications
```

### Technology Stack

| Layer | Technology | Purpose |
|-------|-----------|---------|
| Frontend | React + TypeScript | Dashboard UI |
| API Gateway | FastAPI + Python | REST API |
| Database | PostgreSQL | Customer/config data |
| Cache | Redis | Sessions, pub/sub |
| Metrics | InfluxDB | Time-series data |
| Container | Docker + Compose | Deployment |
| Auth | JWT + bcrypt | Authentication |

---

## 2. Backend API (FastAPI) {#backend}

```python
# main.py - ISP Platform API
from fastapi import FastAPI, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel
from typing import Optional, List
import jwt
import bcrypt
from datetime import datetime, timedelta
import asyncpg
import redis.asyncio as aioredis

# ============================================
# App Setup
# ============================================
app = FastAPI(title="ISP Management Platform", version="1.0.0")

app.add_middleware(
    CORSMiddleware,
    allow_origins=["http://localhost:3000"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"]
)

SECRET_KEY = "your-secret-key-change-in-production"
ALGORITHM = "HS256"

oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/auth/token")

# ============================================
# Pydantic Models
# ============================================

class Customer(BaseModel):
    id: Optional[int] = None
    username: str
    name: str
    email: str
    phone: str
    address: str
    package_id: int
    status: str = "active"

class Package(BaseModel):
    id: Optional[int] = None
    name: str
    download_mbps: int
    upload_mbps: int
    price_monthly: float
    description: str

class RouterDevice(BaseModel):
    id: Optional[int] = None
    name: str
    host: str
    username: str
    location: str
    model: str

class Token(BaseModel):
    access_token: str
    token_type: str

# ============================================
# Database Connection
# ============================================

DATABASE_URL = "postgresql://isp:password@localhost/isp_platform"

async def get_db():
    conn = await asyncpg.connect(DATABASE_URL)
    try:
        yield conn
    finally:
        await conn.close()

async def get_redis():
    r = await aioredis.from_url("redis://localhost")
    try:
        yield r
    finally:
        await r.close()

# ============================================
# Authentication
# ============================================

def create_token(data: dict, expires_delta: timedelta = timedelta(hours=8)):
    to_encode = data.copy()
    to_encode["exp"] = datetime.utcnow() + expires_delta
    return jwt.encode(to_encode, SECRET_KEY, algorithm=ALGORITHM)

async def get_current_user(token: str = Depends(oauth2_scheme)):
    try:
        payload = jwt.decode(token, SECRET_KEY, algorithms=[ALGORITHM])
        username = payload.get("sub")
        if not username:
            raise HTTPException(status_code=401, detail="Invalid token")
        return {"username": username, "role": payload.get("role", "viewer")}
    except jwt.ExpiredSignatureError:
        raise HTTPException(status_code=401, detail="Token expired")
    except jwt.InvalidTokenError:
        raise HTTPException(status_code=401, detail="Invalid token")

@app.post("/auth/token", response_model=Token)
async def login(form: OAuth2PasswordRequestForm = Depends(), 
                db = Depends(get_db)):
    """Login และรับ JWT token"""
    user = await db.fetchrow(
        "SELECT * FROM admin_users WHERE username = $1",
        form.username
    )
    
    if not user or not bcrypt.checkpw(form.password.encode(), user["password_hash"].encode()):
        raise HTTPException(status_code=401, detail="Incorrect credentials")
    
    token = create_token({"sub": form.username, "role": user["role"]})
    return {"access_token": token, "token_type": "bearer"}

# ============================================
# Customer Endpoints
# ============================================

@app.get("/customers", response_model=List[Customer])
async def list_customers(db = Depends(get_db), 
                          current_user = Depends(get_current_user)):
    rows = await db.fetch("SELECT * FROM customers ORDER BY id")
    return [dict(r) for r in rows]

@app.post("/customers", response_model=Customer)
async def create_customer(customer: Customer, db = Depends(get_db),
                           current_user = Depends(get_current_user)):
    """สร้าง customer ใหม่"""
    row = await db.fetchrow("""
        INSERT INTO customers (username, name, email, phone, address, package_id, status)
        VALUES ($1, $2, $3, $4, $5, $6, $7)
        RETURNING *
    """, customer.username, customer.name, customer.email,
        customer.phone, customer.address, customer.package_id, customer.status)
    
    return dict(row)

@app.get("/customers/{customer_id}")
async def get_customer(customer_id: int, db = Depends(get_db),
                        current_user = Depends(get_current_user)):
    row = await db.fetchrow("SELECT * FROM customers WHERE id = $1", customer_id)
    if not row:
        raise HTTPException(status_code=404, detail="Customer not found")
    return dict(row)

@app.put("/customers/{customer_id}")
async def update_customer(customer_id: int, customer: Customer,
                           db = Depends(get_db),
                           current_user = Depends(get_current_user)):
    """อัปเดต customer"""
    row = await db.fetchrow("""
        UPDATE customers SET name=$1, email=$2, phone=$3, 
        package_id=$4, status=$5 WHERE id=$6 RETURNING *
    """, customer.name, customer.email, customer.phone,
        customer.package_id, customer.status, customer_id)
    
    if not row:
        raise HTTPException(status_code=404, detail="Customer not found")
    return dict(row)

# ============================================
# Router Management Endpoints
# ============================================

@app.get("/routers")
async def list_routers(db = Depends(get_db), 
                        current_user = Depends(get_current_user)):
    rows = await db.fetch("SELECT * FROM router_devices")
    return [dict(r) for r in rows]

@app.get("/routers/{router_id}/status")
async def get_router_status(router_id: int, db = Depends(get_db),
                              current_user = Depends(get_current_user)):
    """ดึง real-time status จาก router"""
    row = await db.fetchrow(
        "SELECT * FROM router_devices WHERE id = $1", router_id
    )
    if not row:
        raise HTTPException(status_code=404, detail="Router not found")
    
    router = dict(row)
    
    try:
        import librouteros
        conn = librouteros.connect(
            host=router["host"],
            username=router["username"],
            password=router.get("password", "admin"),
            timeout=5
        )
        resource = dict(list(conn('/system/resource/print'))[0])
        conn.close()
        
        return {
            "id": router_id,
            "name": router["name"],
            "status": "online",
            "cpu_load": resource.get("cpu-load"),
            "uptime": resource.get("uptime"),
            "version": resource.get("version")
        }
    except Exception as e:
        return {
            "id": router_id,
            "name": router["name"],
            "status": "offline",
            "error": str(e)
        }

@app.post("/routers/{router_id}/pppoe/disconnect/{username}")
async def disconnect_pppoe_session(router_id: int, username: str,
                                    db = Depends(get_db),
                                    current_user = Depends(get_current_user)):
    """Disconnect PPPoE session ของ customer"""
    row = await db.fetchrow(
        "SELECT * FROM router_devices WHERE id = $1", router_id
    )
    if not row:
        raise HTTPException(status_code=404, detail="Router not found")
    
    try:
        import librouteros
        conn = librouteros.connect(
            host=row["host"],
            username=row["username"],
            password=row.get("password", "admin")
        )
        
        sessions = list(conn('/ppp/active/print', **{"?name": username}))
        
        for s in sessions:
            session = dict(s)
            conn('/ppp/active/remove', **{"numbers": session.get(".id")})
        
        conn.close()
        return {"message": f"Disconnected {len(sessions)} session(s) for {username}"}
    
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

## 3. Database Layer {#database}

```sql
-- schema.sql - PostgreSQL schema สำหรับ ISP Platform

-- Admin users
CREATE TABLE admin_users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'viewer',
    email VARCHAR(100),
    created_at TIMESTAMP DEFAULT NOW()
);

-- Service packages
CREATE TABLE packages (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    download_mbps INTEGER NOT NULL,
    upload_mbps INTEGER NOT NULL,
    price_monthly DECIMAL(10,2) NOT NULL,
    description TEXT,
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Customers
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) UNIQUE NOT NULL,
    name VARCHAR(200) NOT NULL,
    email VARCHAR(100),
    phone VARCHAR(20),
    address TEXT,
    package_id INTEGER REFERENCES packages(id),
    status VARCHAR(20) DEFAULT 'active',
    pppoe_password VARCHAR(100),
    ip_address INET,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

-- Router devices
CREATE TABLE router_devices (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,
    host VARCHAR(100) NOT NULL,
    username VARCHAR(50) DEFAULT 'admin',
    password VARCHAR(100),
    location VARCHAR(200),
    model VARCHAR(100),
    is_active BOOLEAN DEFAULT true,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Billing
CREATE TABLE invoices (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    amount DECIMAL(10,2) NOT NULL,
    period_start DATE NOT NULL,
    period_end DATE NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    paid_at TIMESTAMP,
    created_at TIMESTAMP DEFAULT NOW()
);

-- Indexes
CREATE INDEX idx_customers_username ON customers(username);
CREATE INDEX idx_customers_status ON customers(status);
CREATE INDEX idx_invoices_customer ON invoices(customer_id);
CREATE INDEX idx_invoices_status ON invoices(status);

-- Default data
INSERT INTO packages (name, download_mbps, upload_mbps, price_monthly, description)
VALUES 
    ('Basic 30M', 30, 10, 299.00, 'Internet 30/10 Mbps'),
    ('Plus 100M', 100, 30, 499.00, 'Internet 100/30 Mbps'),
    ('Pro 300M', 300, 100, 799.00, 'Internet 300/100 Mbps'),
    ('Ultra 1G', 1000, 500, 1299.00, 'Fiber 1Gbps/500Mbps');
```

---

## 4. Frontend Dashboard {#frontend}

```html
<!-- dashboard.html - Simple HTML dashboard -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>ISP Management Dashboard</title>
    <style>
        :root {
            --primary: #1a56db;
            --success: #057a55;
            --warning: #c27803;
            --danger: #e02424;
            --bg: #f9fafb;
        }
        
        * { box-sizing: border-box; margin: 0; padding: 0; }
        body { font-family: 'Segoe UI', sans-serif; background: var(--bg); }
        
        .header {
            background: var(--primary);
            color: white;
            padding: 1rem 2rem;
            display: flex;
            justify-content: space-between;
            align-items: center;
        }
        
        .container { max-width: 1200px; margin: 0 auto; padding: 2rem; }
        
        .stats-grid {
            display: grid;
            grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
            gap: 1rem;
            margin-bottom: 2rem;
        }
        
        .stat-card {
            background: white;
            padding: 1.5rem;
            border-radius: 8px;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
        }
        
        .stat-card h3 { color: #6b7280; font-size: 0.875rem; margin-bottom: 0.5rem; }
        .stat-card .value { font-size: 2rem; font-weight: bold; color: #111; }
        
        .table-container {
            background: white;
            border-radius: 8px;
            box-shadow: 0 1px 3px rgba(0,0,0,0.1);
            overflow-x: auto;
        }
        
        table { width: 100%; border-collapse: collapse; }
        th { background: #f3f4f6; padding: 1rem; text-align: left; }
        td { padding: 0.75rem 1rem; border-top: 1px solid #f3f4f6; }
        
        .badge {
            padding: 0.25rem 0.75rem;
            border-radius: 9999px;
            font-size: 0.75rem;
            font-weight: 600;
        }
        .badge-success { background: #d1fae5; color: var(--success); }
        .badge-danger { background: #fee2e2; color: var(--danger); }
        
        .btn {
            padding: 0.5rem 1rem;
            border: none;
            border-radius: 6px;
            cursor: pointer;
            font-size: 0.875rem;
        }
        .btn-primary { background: var(--primary); color: white; }
        .btn-danger { background: var(--danger); color: white; }
    </style>
</head>
<body>
    <div class="header">
        <h1>ISP Management Platform</h1>
        <span id="user-info">Loading...</span>
    </div>
    
    <div class="container">
        <div class="stats-grid">
            <div class="stat-card">
                <h3>Active Customers</h3>
                <div class="value" id="stat-customers">-</div>
            </div>
            <div class="stat-card">
                <h3>Online Routers</h3>
                <div class="value" id="stat-routers">-</div>
            </div>
            <div class="stat-card">
                <h3>PPPoE Sessions</h3>
                <div class="value" id="stat-pppoe">-</div>
            </div>
            <div class="stat-card">
                <h3>Revenue This Month</h3>
                <div class="value" id="stat-revenue">-</div>
            </div>
        </div>
        
        <h2 style="margin-bottom: 1rem;">Customers</h2>
        <div class="table-container">
            <table>
                <thead>
                    <tr>
                        <th>Name</th>
                        <th>Username</th>
                        <th>Package</th>
                        <th>Status</th>
                        <th>Actions</th>
                    </tr>
                </thead>
                <tbody id="customers-table">
                    <tr><td colspan="5">Loading...</td></tr>
                </tbody>
            </table>
        </div>
    </div>

    <script>
        const API = "http://localhost:8000";
        let token = localStorage.getItem("jwt_token");
        
        async function fetchWithAuth(url) {
            const res = await fetch(API + url, {
                headers: { "Authorization": `Bearer ${token}` }
            });
            if (!res.ok) throw new Error(`HTTP ${res.status}`);
            return res.json();
        }
        
        async function loadDashboard() {
            try {
                const customers = await fetchWithAuth("/customers");
                
                document.getElementById("stat-customers").textContent = 
                    customers.filter(c => c.status === "active").length;
                
                const tbody = document.getElementById("customers-table");
                tbody.innerHTML = customers.map(c => `
                    <tr>
                        <td>${c.name}</td>
                        <td>${c.username}</td>
                        <td>Package ${c.package_id}</td>
                        <td>
                            <span class="badge badge-${c.status === 'active' ? 'success' : 'danger'}">
                                ${c.status}
                            </span>
                        </td>
                        <td>
                            <button class="btn btn-primary" 
                                onclick="viewCustomer(${c.id})">View</button>
                        </td>
                    </tr>
                `).join("");
                
            } catch (e) {
                console.error("Error loading dashboard:", e);
            }
        }
        
        function viewCustomer(id) {
            window.location.href = `/customers/${id}`;
        }
        
        // Load on startup
        if (token) {
            document.getElementById("user-info").textContent = "Admin";
            loadDashboard();
        } else {
            window.location.href = "/login";
        }
        
        // Auto-refresh every 30 seconds
        setInterval(loadDashboard, 30000);
    </script>
</body>
</html>
```

---

## 5. Microservices Integration {#microservices}

```yaml
# docker-compose.yml - Complete ISP Platform
version: '3.8'

services:
  api:
    build: ./api
    ports:
      - "8000:8000"
    environment:
      DATABASE_URL: postgresql://isp:password@postgres/isp_platform
      REDIS_URL: redis://redis:6379
    depends_on:
      - postgres
      - redis
    command: uvicorn main:app --host 0.0.0.0 --port 8000 --reload
  
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: isp
      POSTGRES_PASSWORD: password
      POSTGRES_DB: isp_platform
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./schema.sql:/docker-entrypoint-initdb.d/schema.sql
    ports:
      - "5432:5432"
  
  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"
  
  influxdb:
    image: influxdb:2.7
    ports:
      - "8086:8086"
    environment:
      DOCKER_INFLUXDB_INIT_MODE: setup
      DOCKER_INFLUXDB_INIT_USERNAME: admin
      DOCKER_INFLUXDB_INIT_PASSWORD: admin123
      DOCKER_INFLUXDB_INIT_ORG: isp
      DOCKER_INFLUXDB_INIT_BUCKET: network
    volumes:
      - influxdb_data:/var/lib/influxdb2
  
  grafana:
    image: grafana/grafana:latest
    ports:
      - "3001:3000"
    environment:
      GF_SECURITY_ADMIN_PASSWORD: admin
    volumes:
      - grafana_data:/var/lib/grafana
  
  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./frontend:/usr/share/nginx/html
    depends_on:
      - api

volumes:
  postgres_data:
  influxdb_data:
  grafana_data:
```

```nginx
# nginx.conf
events { worker_connections 1024; }

http {
    upstream api {
        server api:8000;
    }
    
    server {
        listen 80;
        
        location /api/ {
            proxy_pass http://api/;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
        }
        
        location /auth/ {
            proxy_pass http://api/auth/;
        }
        
        location / {
            root /usr/share/nginx/html;
            index dashboard.html;
            try_files $uri $uri/ /dashboard.html;
        }
    }
}
```

---

## 6. Lab: ISP Platform Deployment {#lab}

```bash
# Deploy complete ISP platform

# Step 1: Prepare directories
mkdir -p isp-platform/{api,frontend,monitoring}
cd isp-platform

# Step 2: Start database + redis
docker-compose up -d postgres redis

# Step 3: Initialize database
docker-compose exec postgres psql -U isp -d isp_platform -f /docker-entrypoint-initdb.d/schema.sql

# Step 4: Create admin user
python3 << 'EOF'
import bcrypt
import asyncpg
import asyncio

async def create_admin():
    conn = await asyncpg.connect("postgresql://isp:password@localhost/isp_platform")
    password_hash = bcrypt.hashpw(b"admin123", bcrypt.gensalt()).decode()
    await conn.execute(
        "INSERT INTO admin_users (username, password_hash, role) VALUES ($1, $2, $3)",
        "admin", password_hash, "admin"
    )
    await conn.close()
    print("Admin user created!")

asyncio.run(create_admin())
EOF

# Step 5: Start all services
docker-compose up -d

# Step 6: Test API
curl -X POST http://localhost:8000/auth/token \
  -d "username=admin&password=admin123" \
  -H "Content-Type: application/x-www-form-urlencoded"

# Step 7: Access dashboard
# http://localhost/dashboard.html
# http://localhost:3001  # Grafana

echo "ISP Platform deployed!"
```

### Verification Checklist

- [ ] PostgreSQL running + schema created
- [ ] API responds to /auth/token
- [ ] Customer CRUD endpoints working
- [ ] Router status endpoint working
- [ ] Dashboard loads in browser
- [ ] Grafana connected to InfluxDB

---

## Summary

Part นี้ครอบคลุม:
- **Full-stack architecture** - FastAPI + React + PostgreSQL
- **REST API** พร้อม JWT authentication
- **Database schema** สำหรับ ISP platform
- **Frontend dashboard** HTML/JS
- **Docker Compose** deployment

---

[← Part 96: Machine Learning](part-096-machine-learning.md) | [Part 98: Multi-tenant →](part-098-multi-tenant.md)
