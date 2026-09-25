# Part 55: Web Dashboard สำหรับ MikroTik - ตอนที่ 1

## บทนำ

การสร้าง Web Dashboard ช่วยให้การจัดการ MikroTik router เป็นเรื่องง่ายขึ้น แทนที่จะต้องใช้ Winbox หรือ SSH ผู้ดูแลระบบสามารถดู status, monitor traffic, และจัดการระบบผ่าน web browser ได้จากทุกที่

---

## 55.1 Project Architecture

### Overall Architecture

```
┌─────────────────────────────────────────────┐
│                 Web Browser                  │
│  Vue.js/React Frontend + Chart.js            │
└──────────────────┬──────────────────────────┘
                   │ HTTP/WebSocket
┌──────────────────┴──────────────────────────┐
│              Backend API Server              │
│  Node.js/Express + Socket.io                │
│  Authentication + Caching                   │
└──────────────────┬──────────────────────────┘
                   │ RouterOS API
┌──────────────────┴──────────────────────────┐
│            MikroTik Routers                  │
│  Router 1, Router 2, Router 3...            │
└─────────────────────────────────────────────┘
```

### Layer Separation

| Layer | Role | Technology |
|-------|------|------------|
| Presentation | UI, Charts, Forms | Vue.js, Chart.js |
| API | Business logic, Auth | Node.js/Express |
| Data | Router communication | RouterOS API library |
| Cache | Performance | Redis/In-memory |
| Storage | Users, configs | MySQL/SQLite |

---

## 55.2 Tech Stack Selection

### Frontend Options

| Framework | Pros | Cons | เหมาะสำหรับ |
|-----------|------|------|------------|
| Vue.js 3 | เรียนง่าย, reactive | Community เล็กกว่า React | Mid-size projects |
| React | ใหญ่, ecosystem ดี | Learning curve สูง | Large projects |
| Vanilla JS | ไม่มี dependency | ยากกับ complex UI | Simple dashboards |

### Backend Options

| Framework | Language | Pros |
|-----------|----------|------|
| Express.js | Node.js | เร็ว, ecosystem ดี |
| Fastify | Node.js | เร็วมาก, TypeScript |
| Laravel | PHP | Full-featured, ORM |

### เราจะใช้

```
Frontend:  Vue.js 3 + Vite + Chart.js
Backend:   Node.js + Express + Socket.io
Database:  SQLite (development) / MySQL (production)
Auth:      JWT tokens
Cache:     In-memory Map
```

---

## 55.3 Database Schema Design

### Schema สำหรับ Dashboard

```sql
-- users table
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(50) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    role VARCHAR(20) DEFAULT 'viewer',  -- admin, operator, viewer
    email VARCHAR(100),
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_login DATETIME
);

-- routers table
CREATE TABLE routers (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(100) NOT NULL,
    host VARCHAR(100) NOT NULL,
    port INTEGER DEFAULT 8728,
    api_user VARCHAR(50) DEFAULT 'admin',
    api_password_encrypted VARCHAR(255),
    location VARCHAR(200),
    description TEXT,
    active BOOLEAN DEFAULT 1,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- router_metrics table (time-series data)
CREATE TABLE router_metrics (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    router_id INTEGER REFERENCES routers(id),
    timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
    cpu_load INTEGER,
    free_memory INTEGER,
    total_memory INTEGER,
    uptime VARCHAR(50),
    interface_data TEXT  -- JSON blob
);

-- alerts table
CREATE TABLE alerts (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    router_id INTEGER REFERENCES routers(id),
    severity VARCHAR(20),  -- critical, warning, info
    message TEXT,
    acknowledged BOOLEAN DEFAULT 0,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

-- sessions table
CREATE TABLE sessions (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER REFERENCES users(id),
    token_hash VARCHAR(255),
    expires_at DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

---

## 55.4 Authentication System

### JWT Authentication

```javascript
// src/auth/jwt.js

const jwt = require('jsonwebtoken');
const bcrypt = require('bcryptjs');

const JWT_SECRET = process.env.JWT_SECRET || 'change-this-in-production';
const JWT_EXPIRY = '8h';

class AuthService {
    /**
     * Hash password
     */
    static async hashPassword(password) {
        return bcrypt.hash(password, 12);
    }
    
    /**
     * Verify password
     */
    static async verifyPassword(password, hash) {
        return bcrypt.compare(password, hash);
    }
    
    /**
     * Generate JWT token
     */
    static generateToken(user) {
        return jwt.sign(
            {
                id: user.id,
                username: user.username,
                role: user.role,
            },
            JWT_SECRET,
            { expiresIn: JWT_EXPIRY }
        );
    }
    
    /**
     * Verify JWT token
     */
    static verifyToken(token) {
        try {
            return jwt.verify(token, JWT_SECRET);
        } catch (error) {
            return null;
        }
    }
}

/**
 * Auth middleware
 */
function requireAuth(req, res, next) {
    const authHeader = req.headers.authorization;
    
    if (!authHeader || !authHeader.startsWith('Bearer ')) {
        return res.status(401).json({ error: 'No token provided' });
    }
    
    const token = authHeader.split(' ')[1];
    const payload = AuthService.verifyToken(token);
    
    if (!payload) {
        return res.status(401).json({ error: 'Invalid or expired token' });
    }
    
    req.user = payload;
    next();
}

/**
 * Role middleware
 */
function requireRole(...roles) {
    return (req, res, next) => {
        if (!roles.includes(req.user?.role)) {
            return res.status(403).json({ error: 'Insufficient permissions' });
        }
        next();
    };
}

module.exports = { AuthService, requireAuth, requireRole };
```

### Auth Routes

```javascript
// src/routes/auth.js

const express = require('express');
const router = express.Router();
const { AuthService, requireAuth } = require('../auth/jwt');
const db = require('../database');

// POST /api/auth/login
router.post('/login', async (req, res) => {
    const { username, password } = req.body;
    
    if (!username || !password) {
        return res.status(400).json({ error: 'Username and password required' });
    }
    
    try {
        // ดู user จาก database
        const user = db.prepare('SELECT * FROM users WHERE username = ?')
                       .get(username);
        
        if (!user) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }
        
        // ตรวจสอบ password
        const valid = await AuthService.verifyPassword(password, user.password_hash);
        if (!valid) {
            return res.status(401).json({ error: 'Invalid credentials' });
        }
        
        // อัพเดท last login
        db.prepare('UPDATE users SET last_login = ? WHERE id = ?')
          .run(new Date().toISOString(), user.id);
        
        // Generate token
        const token = AuthService.generateToken(user);
        
        res.json({
            token,
            user: {
                id: user.id,
                username: user.username,
                role: user.role,
                email: user.email,
            }
        });
        
    } catch (error) {
        console.error('Login error:', error);
        res.status(500).json({ error: 'Internal server error' });
    }
});

// POST /api/auth/logout
router.post('/logout', requireAuth, (req, res) => {
    // JWT is stateless, just tell client to remove token
    res.json({ success: true });
});

// GET /api/auth/me
router.get('/me', requireAuth, (req, res) => {
    const user = db.prepare('SELECT id, username, role, email, created_at FROM users WHERE id = ?')
                   .get(req.user.id);
    res.json(user);
});

module.exports = router;
```

---

## 55.5 Router Connection Management

### Connection Manager

```javascript
// src/router/ConnectionManager.js

const { RouterOSAPI } = require('node-routeros');

class ConnectionManager {
    constructor() {
        this._connections = new Map();
    }
    
    async getConnection(routerId, config) {
        // ตรวจสอบว่ามี connection อยู่แล้ว
        if (this._connections.has(routerId)) {
            const conn = this._connections.get(routerId);
            // ทดสอบ connection ว่ายังใช้งานได้
            try {
                await conn.write('/system/identity/print');
                return conn;
            } catch (e) {
                // Connection หลุด - สร้างใหม่
                this._connections.delete(routerId);
            }
        }
        
        // สร้าง connection ใหม่
        const api = new RouterOSAPI({
            host: config.host,
            user: config.user || 'admin',
            password: config.password || '',
            port: config.port || 8728,
            timeout: 15,
        });
        
        await api.connect();
        this._connections.set(routerId, api);
        console.log(`Connected to router ${routerId} (${config.host})`);
        
        return api;
    }
    
    disconnect(routerId) {
        if (this._connections.has(routerId)) {
            const conn = this._connections.get(routerId);
            try {
                conn.close();
            } catch (e) {}
            this._connections.delete(routerId);
            console.log(`Disconnected router ${routerId}`);
        }
    }
    
    disconnectAll() {
        for (const [id] of this._connections) {
            this.disconnect(id);
        }
    }
    
    getStatus() {
        return {
            connected: this._connections.size,
            routerIds: Array.from(this._connections.keys()),
        };
    }
}

module.exports = new ConnectionManager();
```

---

## 55.6 Basic Data Display

### Router Data Routes

```javascript
// src/routes/routers.js

const express = require('express');
const router = express.Router();
const { requireAuth } = require('../auth/jwt');
const connectionManager = require('../router/ConnectionManager');
const db = require('../database');

// GET /api/routers
router.get('/', requireAuth, (req, res) => {
    const routers = db.prepare('SELECT * FROM routers WHERE active = 1').all();
    
    // เพิ่ม connection status
    const withStatus = routers.map(r => ({
        ...r,
        api_password_encrypted: undefined,  // ไม่ส่ง password
        connected: connectionManager.getStatus().routerIds.includes(r.id.toString()),
    }));
    
    res.json(withStatus);
});

// GET /api/routers/:id/summary
router.get('/:id/summary', requireAuth, async (req, res) => {
    try {
        const routerConfig = db.prepare('SELECT * FROM routers WHERE id = ?')
                               .get(req.params.id);
        
        if (!routerConfig) {
            return res.status(404).json({ error: 'Router not found' });
        }
        
        const api = await connectionManager.getConnection(req.params.id, {
            host: routerConfig.host,
            user: routerConfig.api_user,
            password: decrypt(routerConfig.api_password_encrypted),
            port: routerConfig.port,
        });
        
        const [identity, resources] = await Promise.all([
            api.write('/system/identity/print'),
            api.write('/system/resource/print'),
        ]);
        
        res.json({
            id: routerConfig.id,
            name: routerConfig.name,
            host: routerConfig.host,
            identity: identity[0]?.name,
            resources: {
                cpuLoad: parseInt(resources[0]?.['cpu-load'] || 0),
                freeMemory: parseInt(resources[0]?.['free-memory'] || 0),
                totalMemory: parseInt(resources[0]?.['total-memory'] || 0),
                uptime: resources[0]?.uptime || '0s',
                version: resources[0]?.version || 'unknown',
            }
        });
        
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

module.exports = router;
```

---

## 55.7 Chart Libraries

### Chart.js Integration

```html
<!-- public/components/TrafficChart.html -->
<div class="chart-container">
    <canvas id="trafficChart"></canvas>
</div>

<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>
<script>
class TrafficChart {
    constructor(canvasId) {
        this.canvas = document.getElementById(canvasId);
        this.ctx = this.canvas.getContext('2d');
        this.maxPoints = 60;  // 60 data points
        this.labels = [];
        this.rxData = [];
        this.txData = [];
        
        this.chart = new Chart(this.ctx, {
            type: 'line',
            data: {
                labels: this.labels,
                datasets: [
                    {
                        label: 'RX (Download)',
                        data: this.rxData,
                        borderColor: 'rgb(75, 192, 192)',
                        backgroundColor: 'rgba(75, 192, 192, 0.1)',
                        tension: 0.4,
                        fill: true,
                    },
                    {
                        label: 'TX (Upload)',
                        data: this.txData,
                        borderColor: 'rgb(255, 99, 132)',
                        backgroundColor: 'rgba(255, 99, 132, 0.1)',
                        tension: 0.4,
                        fill: true,
                    }
                ]
            },
            options: {
                responsive: true,
                animation: { duration: 200 },
                scales: {
                    y: {
                        beginAtZero: true,
                        ticks: {
                            callback: (value) => this.formatBps(value)
                        }
                    }
                },
                plugins: {
                    tooltip: {
                        callbacks: {
                            label: (context) => {
                                return ` ${context.dataset.label}: ${this.formatBps(context.parsed.y)}`;
                            }
                        }
                    }
                }
            }
        });
    }
    
    addPoint(timestamp, rxBps, txBps) {
        const time = new Date(timestamp).toLocaleTimeString('th-TH');
        
        this.labels.push(time);
        this.rxData.push(rxBps);
        this.txData.push(txBps);
        
        // จำกัดจำนวน points
        if (this.labels.length > this.maxPoints) {
            this.labels.shift();
            this.rxData.shift();
            this.txData.shift();
        }
        
        this.chart.update('none');
    }
    
    formatBps(bps) {
        if (bps >= 1e9) return (bps/1e9).toFixed(2) + ' Gbps';
        if (bps >= 1e6) return (bps/1e6).toFixed(2) + ' Mbps';
        if (bps >= 1e3) return (bps/1e3).toFixed(2) + ' Kbps';
        return bps + ' bps';
    }
}
</script>
```

---

## 55.8 Responsive Layout

### Dashboard HTML Layout

```html
<!-- public/index.html -->
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>MikroTik Dashboard</title>
    <link rel="stylesheet" href="css/dashboard.css">
</head>
<body>
    <!-- Navigation -->
    <nav class="navbar">
        <div class="nav-brand">
            <img src="images/logo.png" alt="Logo" class="nav-logo">
            <span>MikroTik Manager</span>
        </div>
        <div class="nav-links">
            <a href="#dashboard" class="active">Dashboard</a>
            <a href="#interfaces">Interfaces</a>
            <a href="#firewall">Firewall</a>
            <a href="#hotspot">Hotspot</a>
        </div>
        <div class="nav-user">
            <span id="user-name">Loading...</span>
            <button onclick="logout()">Logout</button>
        </div>
    </nav>
    
    <!-- Main Content -->
    <main class="main-content">
        <!-- Stats Row -->
        <div class="stats-grid">
            <div class="stat-card">
                <div class="stat-icon cpu">⚡</div>
                <div class="stat-info">
                    <div class="stat-value" id="cpu-value">0%</div>
                    <div class="stat-label">CPU Load</div>
                </div>
            </div>
            
            <div class="stat-card">
                <div class="stat-icon memory">💾</div>
                <div class="stat-info">
                    <div class="stat-value" id="memory-value">0%</div>
                    <div class="stat-label">Memory</div>
                </div>
            </div>
            
            <div class="stat-card">
                <div class="stat-icon users">👥</div>
                <div class="stat-info">
                    <div class="stat-value" id="users-value">0</div>
                    <div class="stat-label">Active Users</div>
                </div>
            </div>
            
            <div class="stat-card">
                <div class="stat-icon uptime">🕐</div>
                <div class="stat-info">
                    <div class="stat-value" id="uptime-value">-</div>
                    <div class="stat-label">Uptime</div>
                </div>
            </div>
        </div>
        
        <!-- Charts Row -->
        <div class="charts-grid">
            <div class="chart-card">
                <h3>WAN Traffic</h3>
                <canvas id="trafficChart"></canvas>
            </div>
            
            <div class="chart-card">
                <h3>CPU History</h3>
                <canvas id="cpuChart"></canvas>
            </div>
        </div>
        
        <!-- Interfaces -->
        <div class="card">
            <h3>Interfaces</h3>
            <table class="data-table" id="interfaces-table">
                <thead>
                    <tr>
                        <th>Interface</th>
                        <th>Status</th>
                        <th>RX</th>
                        <th>TX</th>
                    </tr>
                </thead>
                <tbody></tbody>
            </table>
        </div>
    </main>
    
    <script src="js/dashboard.js"></script>
</body>
</html>
```

### CSS Responsive Layout

```css
/* public/css/dashboard.css */

:root {
    --primary: #2563eb;
    --success: #16a34a;
    --warning: #d97706;
    --danger: #dc2626;
    --bg: #f1f5f9;
    --card-bg: #ffffff;
    --text: #1e293b;
    --text-light: #64748b;
    --border: #e2e8f0;
}

* { box-sizing: border-box; margin: 0; padding: 0; }

body {
    font-family: 'Inter', 'Sarabun', sans-serif;
    background: var(--bg);
    color: var(--text);
}

/* Navbar */
.navbar {
    background: var(--primary);
    color: white;
    padding: 0 20px;
    height: 60px;
    display: flex;
    align-items: center;
    justify-content: space-between;
    position: sticky;
    top: 0;
    z-index: 100;
}

/* Main Content */
.main-content {
    padding: 20px;
    max-width: 1400px;
    margin: 0 auto;
}

/* Stats Grid */
.stats-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(200px, 1fr));
    gap: 16px;
    margin-bottom: 20px;
}

.stat-card {
    background: var(--card-bg);
    border-radius: 12px;
    padding: 20px;
    display: flex;
    align-items: center;
    gap: 16px;
    box-shadow: 0 1px 3px rgba(0,0,0,0.1);
}

.stat-value {
    font-size: 24px;
    font-weight: 700;
    color: var(--primary);
}

/* Charts Grid */
.charts-grid {
    display: grid;
    grid-template-columns: repeat(auto-fit, minmax(400px, 1fr));
    gap: 16px;
    margin-bottom: 20px;
}

/* Data Table */
.data-table {
    width: 100%;
    border-collapse: collapse;
}

.data-table th, .data-table td {
    padding: 12px;
    text-align: left;
    border-bottom: 1px solid var(--border);
}

/* Mobile Responsive */
@media (max-width: 768px) {
    .stats-grid {
        grid-template-columns: repeat(2, 1fr);
    }
    
    .charts-grid {
        grid-template-columns: 1fr;
    }
    
    .navbar .nav-links {
        display: none;
    }
}
```

---

## 55.9 Real-time Updates Setup

### Socket.io Client Setup

```javascript
// public/js/dashboard.js

const socket = io();
let authToken = localStorage.getItem('auth_token');

// ตรวจสอบ authentication
if (!authToken) {
    window.location.href = '/login.html';
}

// เพิ่ม token ใน socket handshake
const socketWithAuth = io({ 
    auth: { token: authToken }
});

// Authenticate REST requests
async function apiGet(path) {
    const response = await fetch(`/api${path}`, {
        headers: {
            'Authorization': `Bearer ${authToken}`,
            'Content-Type': 'application/json',
        }
    });
    
    if (response.status === 401) {
        localStorage.removeItem('auth_token');
        window.location.href = '/login.html';
        return null;
    }
    
    return response.json();
}

// Initialize charts
const trafficChart = new TrafficChart('trafficChart');
const cpuChart = new CPUChart('cpuChart');

// รับ real-time metrics
socketWithAuth.on('metrics', (data) => {
    updateDashboard(data);
});

function updateDashboard(data) {
    // Update stats
    document.getElementById('cpu-value').textContent = data.cpu_load + '%';
    document.getElementById('uptime-value').textContent = data.uptime;
    
    const memPct = Math.round((1 - data.free_memory / data.total_memory) * 100);
    document.getElementById('memory-value').textContent = memPct + '%';
    
    // Update charts
    trafficChart.addPoint(new Date(), data.rx_bps, data.tx_bps);
    cpuChart.addPoint(new Date(), data.cpu_load);
}

// Load initial data
async function init() {
    const user = await apiGet('/auth/me');
    if (user) {
        document.getElementById('user-name').textContent = user.username;
    }
    
    // เริ่ม real-time monitoring
    socketWithAuth.emit('start-monitoring', { interval: 3000 });
}

init();
```

---

## 55.10 Lab: Dashboard Skeleton

### Lab Overview

| ระบบ | รายละเอียด |
|------|------------|
| Objective | สร้าง Dashboard skeleton ที่ทำงานได้ |
| Stack | Node.js backend + HTML/JS frontend |
| Features | Auth, Real-time via Socket.io |

### Quick Start

```bash
# สร้าง project
mkdir dashboard-lab && cd dashboard-lab
npm init -y
npm install express socket.io node-routeros better-sqlite3 jsonwebtoken bcryptjs dotenv

# สร้าง folder structure
mkdir -p src public/css public/js

# สร้าง .env
cat > .env << 'EOF'
PORT=3000
JWT_SECRET=change-me-in-production
MIKROTIK_HOST=192.168.1.1
MIKROTIK_USER=admin
MIKROTIK_PASS=
EOF

# รัน app
node src/index.js
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Architecture | Frontend/Backend/Router layers |
| Tech Stack | Vue.js, Node.js, Socket.io |
| Database | SQLite schema design |
| Authentication | JWT + bcrypt |
| Connection Manager | Multi-router connection pool |
| Data Display | REST API endpoints |
| Charts | Chart.js integration |
| Layout | Responsive CSS grid |
| Real-time | Socket.io setup |

---

[← Part 54: API Node.js](part-054-api-nodejs.md) | [Part 56: Web Dashboard Part 2 →](part-056-web-dashboard-2.md)
