# Part 54: MikroTik API ด้วย Node.js

## บทนำ

Node.js เหมาะมากสำหรับการสร้าง real-time web applications และ API services เนื่องจาก event-driven, non-blocking I/O การนำ Node.js มาใช้กับ MikroTik API ช่วยสร้าง live dashboards, REST APIs, และ real-time monitoring tools

---

## 54.1 Node.js Libraries

### ตัวเลือก Libraries

| Library | npm Package | Stars | Features |
|---------|------------|-------|----------|
| node-routeros | `node-routeros` | Popular | TypeScript, Promise |
| mikronode | `mikronode` | Older | Event-based |
| routeros-client | Custom | - | Wrapper |

### การเลือก Library

```
ใช้ node-routeros เมื่อ:
├── ต้องการ TypeScript support
├── ต้องการ modern async/await API
└── ต้องการ Promise-based interface

ใช้ mikronode เมื่อ:
├── ต้องการ event-based interface
└── Legacy code compatibility
```

---

## 54.2 Setup and Installation

### สร้าง Project

```bash
# สร้าง project directory
mkdir mikrotik-nodejs
cd mikrotik-nodejs

# Initialize npm
npm init -y

# ติดตั้ง dependencies
npm install node-routeros express socket.io
npm install --save-dev typescript @types/node nodemon ts-node

# สำหรับ TypeScript
npx tsc --init
```

### package.json

```json
{
  "name": "mikrotik-nodejs",
  "version": "1.0.0",
  "description": "MikroTik Node.js API Manager",
  "main": "dist/index.js",
  "scripts": {
    "start": "node dist/index.js",
    "dev": "nodemon src/index.ts",
    "build": "tsc",
    "test": "jest"
  },
  "dependencies": {
    "node-routeros": "^1.4.0",
    "express": "^4.18.0",
    "socket.io": "^4.7.0",
    "dotenv": "^16.0.0"
  },
  "devDependencies": {
    "typescript": "^5.0.0",
    "@types/node": "^20.0.0",
    "@types/express": "^4.17.0",
    "nodemon": "^3.0.0",
    "ts-node": "^10.9.0"
  }
}
```

### ทดสอบ Connection

```javascript
// test_connection.js
const RouterOSAPI = require('node-routeros').RouterOSAPI;

const connection = new RouterOSAPI({
    host: '192.168.1.1',
    user: 'admin',
    password: '',
    port: 8728,
});

connection.connect()
    .then(() => {
        console.log('✓ Connected!');
        
        return connection.write('/system/identity/print');
    })
    .then((data) => {
        console.log('Router name:', data[0].name);
        connection.close();
    })
    .catch((err) => {
        console.error('✗ Error:', err.message);
    });
```

---

## 54.3 Basic Operations

### RouterOS Client Class

```typescript
// src/RouterOSClient.ts
import { RouterOSAPI } from 'node-routeros';

interface RouterOSConfig {
    host: string;
    user?: string;
    password?: string;
    port?: number;
    timeout?: number;
}

export class RouterOSClient {
    private api: RouterOSAPI;
    private connected: boolean = false;
    
    constructor(private config: RouterOSConfig) {
        this.api = new RouterOSAPI({
            host: config.host,
            user: config.user || 'admin',
            password: config.password || '',
            port: config.port || 8728,
            timeout: config.timeout || 10,
        });
    }
    
    async connect(): Promise<void> {
        await this.api.connect();
        this.connected = true;
        console.log(`Connected to ${this.config.host}`);
    }
    
    async disconnect(): Promise<void> {
        if (this.connected) {
            this.api.close();
            this.connected = false;
        }
    }
    
    // ==================== READ ====================
    
    async getAll(path: string): Promise<any[]> {
        const result = await this.api.write(`${path}/print`);
        return result;
    }
    
    async getWhere(path: string, conditions: Record<string, string>): Promise<any[]> {
        const queries = Object.entries(conditions).map(([k, v]) => `?${k}=${v}`);
        const result = await this.api.write([`${path}/print`, ...queries]);
        return result;
    }
    
    // ==================== CREATE ====================
    
    async add(path: string, params: Record<string, string>): Promise<string> {
        const args = Object.entries(params).map(([k, v]) => `=${k}=${v}`);
        const result = await this.api.write([`${path}/add`, ...args]);
        return result[0]?.ret || '';
    }
    
    // ==================== UPDATE ====================
    
    async set(path: string, id: string, params: Record<string, string>): Promise<void> {
        const args = [
            `${path}/set`,
            `=.id=${id}`,
            ...Object.entries(params).map(([k, v]) => `=${k}=${v}`)
        ];
        await this.api.write(args);
    }
    
    // ==================== DELETE ====================
    
    async remove(path: string, id: string): Promise<void> {
        await this.api.write([`${path}/remove`, `=.id=${id}`]);
    }
    
    // ==================== EXECUTE ====================
    
    async execute(command: string, params: Record<string, string> = {}): Promise<any[]> {
        const args = [command, ...Object.entries(params).map(([k, v]) => `=${k}=${v}`)];
        return this.api.write(args);
    }
}

// ตัวอย่างใช้งาน
async function example() {
    const client = new RouterOSClient({ host: '192.168.1.1' });
    
    try {
        await client.connect();
        
        // ดู IP addresses
        const addresses = await client.getAll('/ip/address');
        console.log('IP Addresses:');
        addresses.forEach(addr => {
            console.log(`  ${addr.address} on ${addr.interface}`);
        });
        
        // เพิ่ม address list
        const newId = await client.add('/ip/firewall/address-list', {
            list: 'TEST',
            address: '10.0.0.1',
            comment: 'Node.js API test',
        });
        console.log(`Added entry: ${newId}`);
        
        // ลบ
        await client.remove('/ip/firewall/address-list', newId);
        
    } finally {
        await client.disconnect();
    }
}
```

---

## 54.4 Promise/Async-await Patterns

### Service Layer

```typescript
// src/services/RouterOSService.ts

import { RouterOSClient } from '../RouterOSClient';

interface SystemInfo {
    name: string;
    version: string;
    uptime: string;
    cpuLoad: number;
    freeMemory: number;
    totalMemory: number;
}

interface InterfaceInfo {
    name: string;
    type: string;
    running: boolean;
    rxBytes: number;
    txBytes: number;
}

export class RouterOSService {
    constructor(private client: RouterOSClient) {}
    
    async getSystemInfo(): Promise<SystemInfo> {
        const [identity, resources] = await Promise.all([
            this.client.getAll('/system/identity'),
            this.client.getAll('/system/resource'),
        ]);
        
        const resource = resources[0] || {};
        
        return {
            name: identity[0]?.name || 'Unknown',
            version: resource.version || 'Unknown',
            uptime: resource.uptime || '0s',
            cpuLoad: parseInt(resource['cpu-load'] || '0'),
            freeMemory: parseInt(resource['free-memory'] || '0'),
            totalMemory: parseInt(resource['total-memory'] || '0'),
        };
    }
    
    async getInterfaces(onlyRunning = false): Promise<InterfaceInfo[]> {
        const interfaces = onlyRunning
            ? await this.client.getWhere('/interface', { running: 'true' })
            : await this.client.getAll('/interface');
        
        return interfaces.map(iface => ({
            name: iface.name,
            type: iface.type,
            running: iface.running === 'true',
            rxBytes: parseInt(iface['rx-byte'] || '0'),
            txBytes: parseInt(iface['tx-byte'] || '0'),
        }));
    }
    
    async getFirewallStats(): Promise<any[]> {
        const rules = await this.client.getAll('/ip/firewall/filter');
        return rules.map(rule => ({
            chain: rule.chain,
            action: rule.action,
            packets: parseInt(rule.packets || '0'),
            bytes: parseInt(rule.bytes || '0'),
            comment: rule.comment || '',
        }));
    }
}
```

---

## 54.5 Real-time Data Streaming

### WebSocket Server สำหรับ Real-time Data

```typescript
// src/server/websocket.ts

import { Server as SocketIOServer } from 'socket.io';
import { RouterOSService } from '../services/RouterOSService';
import { RouterOSClient } from '../RouterOSClient';

export function setupWebSocket(io: SocketIOServer, client: RouterOSClient) {
    const service = new RouterOSService(client);
    
    io.on('connection', (socket) => {
        console.log(`Client connected: ${socket.id}`);
        
        let metricsInterval: NodeJS.Timer | null = null;
        
        // เริ่ม streaming metrics
        socket.on('start-monitoring', async (config: { interval: number }) => {
            const interval = config.interval || 5000;
            
            metricsInterval = setInterval(async () => {
                try {
                    const [systemInfo, interfaces] = await Promise.all([
                        service.getSystemInfo(),
                        service.getInterfaces(true),
                    ]);
                    
                    socket.emit('metrics', {
                        timestamp: new Date().toISOString(),
                        system: systemInfo,
                        interfaces,
                    });
                } catch (error) {
                    socket.emit('error', { message: (error as Error).message });
                }
            }, interval);
            
            socket.emit('monitoring-started', { interval });
        });
        
        // หยุด streaming
        socket.on('stop-monitoring', () => {
            if (metricsInterval) {
                clearInterval(metricsInterval);
                metricsInterval = null;
            }
            socket.emit('monitoring-stopped');
        });
        
        // Request specific data
        socket.on('get-interfaces', async () => {
            const interfaces = await service.getInterfaces();
            socket.emit('interfaces', interfaces);
        });
        
        socket.on('disconnect', () => {
            if (metricsInterval) {
                clearInterval(metricsInterval);
            }
            console.log(`Client disconnected: ${socket.id}`);
        });
    });
}
```

---

## 54.6 Express.js Integration

### REST API Server

```typescript
// src/server/index.ts

import express from 'express';
import { createServer } from 'http';
import { Server as SocketIOServer } from 'socket.io';
import { RouterOSClient } from '../RouterOSClient';
import { RouterOSService } from '../services/RouterOSService';
import { setupWebSocket } from './websocket';

const app = express();
const httpServer = createServer(app);
const io = new SocketIOServer(httpServer, {
    cors: { origin: '*' }
});

app.use(express.json());

// Middleware - สร้าง connection ต่อ request
app.use(async (req, res, next) => {
    const client = new RouterOSClient({
        host: process.env.MIKROTIK_HOST || '192.168.1.1',
        user: process.env.MIKROTIK_USER || 'admin',
        password: process.env.MIKROTIK_PASS || '',
    });
    
    try {
        await client.connect();
        (req as any).routerClient = client;
        (req as any).routerService = new RouterOSService(client);
        next();
    } catch (error) {
        res.status(503).json({ error: 'Cannot connect to router' });
    }
});

// Cleanup middleware
app.use(async (req, res, next) => {
    res.on('finish', async () => {
        const client = (req as any).routerClient;
        if (client) await client.disconnect();
    });
    next();
});

// ==================== ROUTES ====================

// GET /api/system
app.get('/api/system', async (req, res) => {
    try {
        const service = (req as any).routerService as RouterOSService;
        const info = await service.getSystemInfo();
        res.json(info);
    } catch (error) {
        res.status(500).json({ error: (error as Error).message });
    }
});

// GET /api/interfaces
app.get('/api/interfaces', async (req, res) => {
    try {
        const service = (req as any).routerService as RouterOSService;
        const onlyRunning = req.query.running === 'true';
        const interfaces = await service.getInterfaces(onlyRunning);
        res.json(interfaces);
    } catch (error) {
        res.status(500).json({ error: (error as Error).message });
    }
});

// GET /api/firewall/rules
app.get('/api/firewall/rules', async (req, res) => {
    try {
        const client = (req as any).routerClient as RouterOSClient;
        const rules = await client.getAll('/ip/firewall/filter');
        res.json(rules);
    } catch (error) {
        res.status(500).json({ error: (error as Error).message });
    }
});

// POST /api/address-list
app.post('/api/address-list', async (req, res) => {
    const { list, address, timeout, comment } = req.body;
    
    if (!list || !address) {
        return res.status(400).json({ error: 'list and address are required' });
    }
    
    try {
        const client = (req as any).routerClient as RouterOSClient;
        const params: Record<string, string> = { list, address };
        if (timeout) params.timeout = timeout;
        if (comment) params.comment = comment;
        
        const id = await client.add('/ip/firewall/address-list', params);
        res.status(201).json({ id, list, address });
    } catch (error) {
        res.status(500).json({ error: (error as Error).message });
    }
});

// DELETE /api/address-list/:id
app.delete('/api/address-list/:id', async (req, res) => {
    try {
        const client = (req as any).routerClient as RouterOSClient;
        await client.remove('/ip/firewall/address-list', req.params.id);
        res.json({ success: true });
    } catch (error) {
        res.status(500).json({ error: (error as Error).message });
    }
});

// Setup WebSocket
const globalClient = new RouterOSClient({
    host: process.env.MIKROTIK_HOST || '192.168.1.1',
    user: process.env.MIKROTIK_USER || 'admin',
    password: process.env.MIKROTIK_PASS || '',
});

globalClient.connect().then(() => {
    setupWebSocket(io, globalClient);
});

// Start server
const PORT = process.env.PORT || 3000;
httpServer.listen(PORT, () => {
    console.log(`Server running on http://localhost:${PORT}`);
});
```

---

## 54.7 Socket.io for Live Updates

### Client-side JavaScript

```html
<!-- public/dashboard.html -->
<!DOCTYPE html>
<html>
<head>
    <title>MikroTik Dashboard</title>
    <script src="/socket.io/socket.io.js"></script>
    <style>
        body { font-family: Arial, sans-serif; padding: 20px; }
        .metric { background: #f0f0f0; padding: 15px; margin: 10px; border-radius: 8px; }
        .up { color: green; }
        .down { color: red; }
        table { width: 100%; border-collapse: collapse; }
        th, td { padding: 8px; text-align: left; border: 1px solid #ddd; }
    </style>
</head>
<body>
    <h1>MikroTik Live Dashboard</h1>
    
    <div id="system-info" class="metric">
        <h3>System</h3>
        <p>Router: <span id="router-name">-</span></p>
        <p>CPU: <span id="cpu-load">-</span>%</p>
        <p>Memory: <span id="memory-info">-</span></p>
        <p>Uptime: <span id="uptime">-</span></p>
    </div>
    
    <div class="metric">
        <h3>Interfaces</h3>
        <table id="interfaces-table">
            <thead>
                <tr><th>Interface</th><th>Status</th><th>RX</th><th>TX</th></tr>
            </thead>
            <tbody id="interfaces-body"></tbody>
        </table>
    </div>
    
    <script>
        const socket = io();
        
        // เริ่ม monitoring เมื่อ connect
        socket.on('connect', () => {
            console.log('Connected to server');
            socket.emit('start-monitoring', { interval: 3000 });
        });
        
        // รับ metrics
        socket.on('metrics', (data) => {
            updateSystem(data.system);
            updateInterfaces(data.interfaces);
        });
        
        function updateSystem(system) {
            document.getElementById('router-name').textContent = system.name;
            document.getElementById('cpu-load').textContent = system.cpuLoad;
            
            const memUsed = (system.totalMemory - system.freeMemory) / 1024 / 1024;
            const memTotal = system.totalMemory / 1024 / 1024;
            document.getElementById('memory-info').textContent = 
                `${memUsed.toFixed(0)}MB / ${memTotal.toFixed(0)}MB`;
            
            document.getElementById('uptime').textContent = system.uptime;
        }
        
        function updateInterfaces(interfaces) {
            const tbody = document.getElementById('interfaces-body');
            tbody.innerHTML = '';
            
            interfaces.forEach(iface => {
                const tr = document.createElement('tr');
                const statusClass = iface.running ? 'up' : 'down';
                const status = iface.running ? 'UP' : 'DOWN';
                const rx = formatBytes(iface.rxBytes);
                const tx = formatBytes(iface.txBytes);
                
                tr.innerHTML = `
                    <td>${iface.name}</td>
                    <td class="${statusClass}">${status}</td>
                    <td>${rx}</td>
                    <td>${tx}</td>
                `;
                tbody.appendChild(tr);
            });
        }
        
        function formatBytes(bytes) {
            if (bytes >= 1073741824) return (bytes/1073741824).toFixed(2) + ' GB';
            if (bytes >= 1048576) return (bytes/1048576).toFixed(2) + ' MB';
            if (bytes >= 1024) return (bytes/1024).toFixed(2) + ' KB';
            return bytes + ' B';
        }
    </script>
</body>
</html>
```

---

## 54.8 Building REST API Wrapper

### Comprehensive REST API

```typescript
// src/routes/api.ts

import { Router } from 'express';

const router = Router();

// GET /api/hotspot/users
router.get('/hotspot/users', async (req, res) => {
    const client = (req as any).routerClient;
    const users = await client.getAll('/ip/hotspot/user');
    
    const enhanced = users.map((user: any) => ({
        id: user['.id'],
        username: user.name,
        profile: user.profile,
        bytesIn: parseInt(user['bytes-in'] || '0'),
        bytesOut: parseInt(user['bytes-out'] || '0'),
        uptime: user.uptime || '0s',
        comment: user.comment || '',
    }));
    
    res.json({ users: enhanced, count: enhanced.length });
});

// POST /api/hotspot/users
router.post('/hotspot/users', async (req, res) => {
    const { username, password, profile = 'default', comment = '' } = req.body;
    
    if (!username || !password) {
        return res.status(400).json({ error: 'username and password required' });
    }
    
    const client = (req as any).routerClient;
    const id = await client.add('/ip/hotspot/user', {
        name: username, password, profile, comment
    });
    
    res.status(201).json({ id, username, profile });
});

// DELETE /api/hotspot/users/:id
router.delete('/hotspot/users/:id', async (req, res) => {
    const client = (req as any).routerClient;
    await client.remove('/ip/hotspot/user', req.params.id);
    res.json({ success: true });
});

export default router;
```

---

## 54.9 Production Deployment

### Docker Configuration

```dockerfile
# Dockerfile
FROM node:20-alpine

WORKDIR /app

COPY package*.json ./
RUN npm ci --production

COPY dist/ ./dist/

EXPOSE 3000

ENV NODE_ENV=production

CMD ["node", "dist/server/index.js"]
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  mikrotik-api:
    build: .
    ports:
      - "3000:3000"
    environment:
      MIKROTIK_HOST: "192.168.1.1"
      MIKROTIK_USER: "admin"
      MIKROTIK_PASS: ""
      PORT: "3000"
    restart: unless-stopped
```

---

## 54.10 Lab: Node.js Network Dashboard

### Lab Overview

| ระบบ | รายละเอียด |
|------|------------|
| Backend | Node.js + Express + Socket.io |
| Features | REST API, WebSocket real-time |
| Dashboard | HTML/JS live updating |

### Quick Start Lab

```bash
# 1. สร้าง project
mkdir mikrotik-lab && cd mikrotik-lab
npm init -y
npm install node-routeros express socket.io dotenv

# 2. สร้าง .env
echo "MIKROTIK_HOST=192.168.1.1
MIKROTIK_USER=admin
MIKROTIK_PASS=
PORT=3000" > .env

# 3. สร้าง server.js
cat > server.js << 'EOF'
require('dotenv').config();
const express = require('express');
const { createServer } = require('http');
const { Server } = require('socket.io');
const { RouterOSAPI } = require('node-routeros');

const app = express();
const httpServer = createServer(app);
const io = new Server(httpServer);

app.use(express.static('public'));
app.use(express.json());

function createClient() {
    return new RouterOSAPI({
        host: process.env.MIKROTIK_HOST,
        user: process.env.MIKROTIK_USER,
        password: process.env.MIKROTIK_PASS,
        port: 8728
    });
}

// REST API
app.get('/api/system', async (req, res) => {
    const client = createClient();
    try {
        await client.connect();
        const [identity, resources] = await Promise.all([
            client.write('/system/identity/print'),
            client.write('/system/resource/print')
        ]);
        res.json({ name: identity[0].name, ...resources[0] });
    } catch(e) {
        res.status(500).json({ error: e.message });
    } finally {
        client.close();
    }
});

// WebSocket
io.on('connection', (socket) => {
    const client = createClient();
    let interval;
    
    client.connect().then(() => {
        interval = setInterval(async () => {
            const res = await client.write('/system/resource/print');
            socket.emit('metrics', res[0]);
        }, 3000);
    });
    
    socket.on('disconnect', () => {
        clearInterval(interval);
        client.close();
    });
});

httpServer.listen(process.env.PORT, () => {
    console.log(`Server running on http://localhost:${process.env.PORT}`);
});
EOF

# 4. รัน
node server.js
```

### ทดสอบ API

```bash
# ทดสอบ REST endpoint
curl http://localhost:3000/api/system | python3 -m json.tool

# เปิด dashboard
open http://localhost:3000
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Libraries | node-routeros, mikronode |
| Setup | npm, TypeScript config |
| Basic Operations | CRUD ด้วย async/await |
| Service Layer | Business logic separation |
| Real-time | Socket.io streaming |
| Express | REST API endpoints |
| WebSocket Client | Live dashboard HTML/JS |
| REST Wrapper | Comprehensive API routes |
| Production | Docker deployment |

---

[← Part 53: API Python](part-053-api-python.md) | [Part 55: Web Dashboard Part 1 →](part-055-web-dashboard-1.md)
