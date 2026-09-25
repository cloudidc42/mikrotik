# Part 56: Web Dashboard - ตอนที่ 2

## บทนำ

ในตอนที่ 2 นี้เราจะสร้าง features ที่สมบูรณ์ยิ่งขึ้น ได้แก่ interface statistics, real-time traffic graphs, connected clients list, firewall log viewer, multi-router support, alerts system, user management UI, และ export reports

---

## 56.1 Interface Statistics Display

### Interface Stats API

```javascript
// src/routes/interfaces.js

const express = require('express');
const router = express.Router();
const { requireAuth } = require('../auth/jwt');
const connectionManager = require('../router/ConnectionManager');
const db = require('../database');

// GET /api/routers/:id/interfaces
router.get('/:routerId/interfaces', requireAuth, async (req, res) => {
    try {
        const routerConfig = getRouterConfig(req.params.routerId);
        const api = await connectionManager.getConnection(req.params.routerId, routerConfig);
        
        const interfaces = await api.write('/interface/print');
        
        const enriched = interfaces.map(iface => ({
            id: iface['.id'],
            name: iface.name,
            type: iface.type,
            running: iface.running === 'true',
            disabled: iface.disabled === 'true',
            rxBytes: parseInt(iface['rx-byte'] || 0),
            txBytes: parseInt(iface['tx-byte'] || 0),
            rxPackets: parseInt(iface['rx-packet'] || 0),
            txPackets: parseInt(iface['tx-packet'] || 0),
            rxErrors: parseInt(iface['rx-error'] || 0),
            txErrors: parseInt(iface['tx-error'] || 0),
            rxDrops: parseInt(iface['rx-drop'] || 0),
            txDrops: parseInt(iface['tx-drop'] || 0),
            mtu: parseInt(iface.mtu || 1500),
            comment: iface.comment || '',
        }));
        
        res.json(enriched);
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// GET /api/routers/:id/interfaces/:name/stats  
router.get('/:routerId/interfaces/:name/stats', requireAuth, async (req, res) => {
    try {
        const routerConfig = getRouterConfig(req.params.routerId);
        const api = await connectionManager.getConnection(req.params.routerId, routerConfig);
        
        const stats = await api.write([
            '/interface/monitor-traffic',
            `=interface=${req.params.name}`,
            '=once='
        ]);
        
        if (!stats.length) {
            return res.status(404).json({ error: 'Interface not found' });
        }
        
        res.json({
            name: req.params.name,
            rxBitsPerSecond: parseInt(stats[0]['rx-bits-per-second'] || 0),
            txBitsPerSecond: parseInt(stats[0]['tx-bits-per-second'] || 0),
            rxPacketsPerSecond: parseInt(stats[0]['rx-packets-per-second'] || 0),
            txPacketsPerSecond: parseInt(stats[0]['tx-packets-per-second'] || 0),
            timestamp: new Date().toISOString(),
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

module.exports = router;
```

### Interface Table Component

```javascript
// public/js/components/InterfaceTable.js

class InterfaceTable {
    constructor(containerId, routerId) {
        this.container = document.getElementById(containerId);
        this.routerId = routerId;
        this.data = [];
    }
    
    async load() {
        try {
            const response = await apiGet(`/routers/${this.routerId}/interfaces`);
            this.data = response;
            this.render();
        } catch (error) {
            console.error('Failed to load interfaces:', error);
        }
    }
    
    render() {
        const html = `
            <table class="data-table">
                <thead>
                    <tr>
                        <th>Interface</th>
                        <th>Type</th>
                        <th>Status</th>
                        <th>RX</th>
                        <th>TX</th>
                        <th>Errors</th>
                    </tr>
                </thead>
                <tbody>
                    ${this.data.map(iface => this.renderRow(iface)).join('')}
                </tbody>
            </table>
        `;
        this.container.innerHTML = html;
    }
    
    renderRow(iface) {
        const statusClass = iface.disabled ? 'badge-gray' : 
                           iface.running ? 'badge-green' : 'badge-red';
        const status = iface.disabled ? 'Disabled' : iface.running ? 'Up' : 'Down';
        
        return `
            <tr>
                <td>
                    <strong>${iface.name}</strong>
                    ${iface.comment ? `<br><small>${iface.comment}</small>` : ''}
                </td>
                <td>${iface.type}</td>
                <td><span class="badge ${statusClass}">${status}</span></td>
                <td>${formatBytes(iface.rxBytes)}</td>
                <td>${formatBytes(iface.txBytes)}</td>
                <td>${iface.rxErrors + iface.txErrors}</td>
            </tr>
        `;
    }
}
```

---

## 56.2 Traffic Graphs (Real-time)

### Real-time Traffic Chart

```javascript
// public/js/components/RealtimeTrafficGraph.js

class RealtimeTrafficGraph {
    constructor(canvasId, title = 'Traffic') {
        this.canvas = document.getElementById(canvasId);
        this.maxDataPoints = 60;
        this.rxData = new Array(this.maxDataPoints).fill(0);
        this.txData = new Array(this.maxDataPoints).fill(0);
        this.labels = new Array(this.maxDataPoints).fill('');
        
        this.chart = new Chart(this.canvas.getContext('2d'), {
            type: 'line',
            data: {
                labels: this.labels,
                datasets: [
                    {
                        label: '↓ Download',
                        data: this.rxData,
                        borderColor: '#3b82f6',
                        backgroundColor: 'rgba(59, 130, 246, 0.1)',
                        borderWidth: 2,
                        pointRadius: 0,
                        tension: 0.4,
                        fill: true,
                    },
                    {
                        label: '↑ Upload',
                        data: this.txData,
                        borderColor: '#ef4444',
                        backgroundColor: 'rgba(239, 68, 68, 0.1)',
                        borderWidth: 2,
                        pointRadius: 0,
                        tension: 0.4,
                        fill: true,
                    }
                ]
            },
            options: {
                responsive: true,
                maintainAspectRatio: true,
                animation: false,
                interaction: {
                    mode: 'index',
                    intersect: false,
                },
                plugins: {
                    title: {
                        display: true,
                        text: title,
                    },
                    tooltip: {
                        callbacks: {
                            label: (ctx) => ` ${ctx.dataset.label}: ${formatBps(ctx.parsed.y)}`
                        }
                    }
                },
                scales: {
                    x: {
                        display: false,
                    },
                    y: {
                        beginAtZero: true,
                        ticks: {
                            callback: (val) => formatBps(val)
                        }
                    }
                }
            }
        });
    }
    
    addDataPoint(rxBps, txBps) {
        const now = new Date().toLocaleTimeString();
        
        this.rxData.push(rxBps);
        this.txData.push(txBps);
        this.labels.push(now);
        
        if (this.rxData.length > this.maxDataPoints) {
            this.rxData.shift();
            this.txData.shift();
            this.labels.shift();
        }
        
        this.chart.update('none');
    }
    
    clear() {
        this.rxData.fill(0);
        this.txData.fill(0);
        this.labels.fill('');
        this.chart.update();
    }
}

function formatBps(bps) {
    if (bps >= 1e9) return (bps/1e9).toFixed(1) + ' Gbps';
    if (bps >= 1e6) return (bps/1e6).toFixed(1) + ' Mbps';
    if (bps >= 1e3) return (bps/1e3).toFixed(1) + ' Kbps';
    return bps + ' bps';
}
```

---

## 56.3 Connected Clients List

### Hotspot/DHCP Clients API

```javascript
// src/routes/clients.js

const express = require('express');
const router = express.Router();

// GET /api/routers/:id/clients
router.get('/:routerId/clients', requireAuth, async (req, res) => {
    try {
        const api = await getRouterAPI(req.params.routerId);
        
        const [hotspotActive, dhcpLeases, arpTable] = await Promise.all([
            api.write('/ip/hotspot/active/print').catch(() => []),
            api.write('/ip/dhcp-server/lease/print').catch(() => []),
            api.write('/ip/arp/print').catch(() => []),
        ]);
        
        // สร้าง client map จาก ARP
        const arpMap = {};
        arpTable.forEach(entry => {
            arpMap[entry['mac-address']] = entry.address;
        });
        
        // Hotspot clients
        const hotspotClients = hotspotActive.map(client => ({
            type: 'hotspot',
            username: client.user,
            mac: client['mac-address'],
            ip: client.address,
            uptime: client.uptime,
            rxBytes: parseInt(client['bytes-in'] || 0),
            txBytes: parseInt(client['bytes-out'] || 0),
            comment: '',
        }));
        
        // DHCP clients
        const dhcpClients = dhcpLeases
            .filter(lease => lease.status === 'bound')
            .map(lease => ({
                type: 'dhcp',
                username: lease['host-name'] || 'Unknown',
                mac: lease['mac-address'],
                ip: lease.address,
                uptime: lease['expires-after'] || 'N/A',
                rxBytes: 0,
                txBytes: 0,
                comment: lease.comment || '',
            }));
        
        res.json({
            hotspot: hotspotClients,
            dhcp: dhcpClients,
            total: hotspotClients.length + dhcpClients.length,
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});

// POST /api/routers/:id/clients/:mac/disconnect
router.post('/:routerId/clients/:mac/disconnect', requireAuth, async (req, res) => {
    try {
        const api = await getRouterAPI(req.params.routerId);
        const mac = req.params.mac;
        
        // หา hotspot session ที่มี MAC นี้
        const sessions = await api.write([
            '/ip/hotspot/active/print',
            `?mac-address=${mac}`
        ]);
        
        if (sessions.length > 0) {
            await api.write([
                '/ip/hotspot/active/remove',
                `=.id=${sessions[0]['.id']}`
            ]);
            res.json({ success: true, message: 'Client disconnected' });
        } else {
            res.status(404).json({ error: 'Client session not found' });
        }
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});
```

---

## 56.4 Firewall Log Viewer

### Log Viewer Component

```javascript
// src/routes/logs.js

// GET /api/routers/:id/logs
router.get('/:routerId/logs', requireAuth, async (req, res) => {
    const { topics, limit = 100, search } = req.query;
    
    try {
        const api = await getRouterAPI(req.params.routerId);
        
        let args = ['/log/print'];
        if (topics) {
            args.push(`?topics=${topics}`);
        }
        
        const logs = await api.write(args);
        
        // Filter and limit
        let filtered = logs;
        if (search) {
            filtered = logs.filter(log => 
                log.message?.toLowerCase().includes(search.toLowerCase())
            );
        }
        
        // Sort newest first and limit
        const limited = filtered.slice(-parseInt(limit)).reverse();
        
        res.json({
            logs: limited,
            total: filtered.length,
        });
    } catch (error) {
        res.status(500).json({ error: error.message });
    }
});
```

### Log Viewer HTML/JS

```html
<!-- Firewall Log Viewer -->
<div class="card" id="log-viewer">
    <div class="card-header">
        <h3>Firewall Logs</h3>
        <div class="log-controls">
            <input type="text" id="log-search" 
                   placeholder="Search logs..." 
                   class="input-sm">
            <select id="log-filter" class="input-sm">
                <option value="">All topics</option>
                <option value="firewall">Firewall</option>
                <option value="warning">Warnings</option>
                <option value="error">Errors</option>
            </select>
            <button onclick="refreshLogs()" class="btn-sm">Refresh</button>
            <button onclick="toggleAutoRefresh()" class="btn-sm" id="auto-refresh-btn">
                Auto: OFF
            </button>
        </div>
    </div>
    
    <div id="log-container" class="log-container">
        <!-- Logs will be loaded here -->
    </div>
</div>

<script>
let autoRefreshInterval = null;

async function loadLogs() {
    const search = document.getElementById('log-search').value;
    const topics = document.getElementById('log-filter').value;
    
    const params = new URLSearchParams();
    if (search) params.append('search', search);
    if (topics) params.append('topics', topics);
    params.append('limit', 200);
    
    const data = await apiGet(`/routers/${currentRouterId}/logs?${params}`);
    
    const container = document.getElementById('log-container');
    container.innerHTML = data.logs.map(log => {
        const severity = getSeverity(log.topics);
        return `
            <div class="log-entry log-${severity}">
                <span class="log-time">${log.time}</span>
                <span class="log-topics">[${log.topics}]</span>
                <span class="log-message">${escapeHtml(log.message)}</span>
            </div>
        `;
    }).join('');
    
    // Scroll to bottom for newest
    container.scrollTop = container.scrollHeight;
}

function getSeverity(topics) {
    if (!topics) return 'info';
    if (topics.includes('critical') || topics.includes('error')) return 'error';
    if (topics.includes('warning')) return 'warning';
    return 'info';
}

function toggleAutoRefresh() {
    const btn = document.getElementById('auto-refresh-btn');
    if (autoRefreshInterval) {
        clearInterval(autoRefreshInterval);
        autoRefreshInterval = null;
        btn.textContent = 'Auto: OFF';
        btn.classList.remove('active');
    } else {
        autoRefreshInterval = setInterval(loadLogs, 5000);
        btn.textContent = 'Auto: ON';
        btn.classList.add('active');
    }
}
</script>
```

---

## 56.5 Router System Resources

### System Resource Widget

```html
<div class="resource-grid">
    <!-- CPU Widget -->
    <div class="resource-widget">
        <div class="resource-header">
            <span>CPU Load</span>
            <span id="cpu-percent">0%</span>
        </div>
        <div class="progress-bar">
            <div class="progress-fill cpu-fill" id="cpu-bar" style="width: 0%"></div>
        </div>
        <div class="resource-details">
            <span id="cpu-count">-</span> CPUs
        </div>
    </div>
    
    <!-- Memory Widget -->
    <div class="resource-widget">
        <div class="resource-header">
            <span>Memory</span>
            <span id="mem-percent">0%</span>
        </div>
        <div class="progress-bar">
            <div class="progress-fill mem-fill" id="mem-bar" style="width: 0%"></div>
        </div>
        <div class="resource-details">
            <span id="mem-used">-</span> / <span id="mem-total">-</span>
        </div>
    </div>
    
    <!-- Disk Widget -->
    <div class="resource-widget">
        <div class="resource-header">
            <span>Storage</span>
            <span id="disk-percent">0%</span>
        </div>
        <div class="progress-bar">
            <div class="progress-fill disk-fill" id="disk-bar" style="width: 0%"></div>
        </div>
        <div class="resource-details">
            <span id="disk-used">-</span> / <span id="disk-total">-</span>
        </div>
    </div>
</div>

<script>
function updateResources(data) {
    const { cpuLoad, freeMemory, totalMemory, 
            freeDisk, totalDisk, cpuCount } = data;
    
    // CPU
    document.getElementById('cpu-percent').textContent = `${cpuLoad}%`;
    document.getElementById('cpu-bar').style.width = `${cpuLoad}%`;
    document.getElementById('cpu-bar').className = 
        `progress-fill ${cpuLoad > 80 ? 'danger' : cpuLoad > 50 ? 'warning' : 'success'}`;
    document.getElementById('cpu-count').textContent = cpuCount || '-';
    
    // Memory
    const memUsed = totalMemory - freeMemory;
    const memPct = Math.round((memUsed / totalMemory) * 100);
    document.getElementById('mem-percent').textContent = `${memPct}%`;
    document.getElementById('mem-bar').style.width = `${memPct}%`;
    document.getElementById('mem-used').textContent = formatBytes(memUsed);
    document.getElementById('mem-total').textContent = formatBytes(totalMemory);
    
    // Disk
    if (totalDisk) {
        const diskUsed = totalDisk - freeDisk;
        const diskPct = Math.round((diskUsed / totalDisk) * 100);
        document.getElementById('disk-percent').textContent = `${diskPct}%`;
        document.getElementById('disk-bar').style.width = `${diskPct}%`;
        document.getElementById('disk-used').textContent = formatBytes(diskUsed);
        document.getElementById('disk-total').textContent = formatBytes(totalDisk);
    }
}
</script>
```

---

## 56.6 Multi-router Support

### Router Selector Component

```javascript
// public/js/components/RouterSelector.js

class RouterSelector {
    constructor(containerId, onChange) {
        this.container = document.getElementById(containerId);
        this.onChange = onChange;
        this.currentRouter = null;
        this.routers = [];
    }
    
    async load() {
        const data = await apiGet('/routers');
        this.routers = data;
        this.render();
    }
    
    render() {
        const html = `
            <div class="router-selector">
                <label>Router:</label>
                <select id="router-select" onchange="routerSelector.select(this.value)">
                    ${this.routers.map(r => `
                        <option value="${r.id}" 
                                ${this.currentRouter?.id === r.id ? 'selected' : ''}>
                            ${r.name} (${r.host})
                            ${r.connected ? '✓' : '⚠'}
                        </option>
                    `).join('')}
                </select>
                <span class="router-status ${this.currentRouter?.connected ? 'online' : 'offline'}">
                    ${this.currentRouter?.connected ? 'Online' : 'Offline'}
                </span>
            </div>
        `;
        this.container.innerHTML = html;
    }
    
    select(routerId) {
        this.currentRouter = this.routers.find(r => r.id === parseInt(routerId));
        this.render();
        
        if (this.onChange) {
            this.onChange(this.currentRouter);
        }
    }
}
```

---

## 56.7 Alerts and Notifications

### Alert System

```javascript
// src/services/AlertService.js

class AlertService {
    constructor(db, emailTransport) {
        this.db = db;
        this.email = emailTransport;
        this.thresholds = {
            cpuLoad: 80,       // %
            memoryUsed: 90,    // %
            diskUsed: 85,      // %
        };
    }
    
    async checkRouter(routerData) {
        const alerts = [];
        
        // CPU check
        if (routerData.cpuLoad > this.thresholds.cpuLoad) {
            alerts.push({
                severity: 'warning',
                message: `High CPU usage: ${routerData.cpuLoad}%`,
                routerId: routerData.id,
            });
        }
        
        // Memory check
        const memPct = Math.round(
            ((routerData.totalMemory - routerData.freeMemory) / 
             routerData.totalMemory) * 100
        );
        if (memPct > this.thresholds.memoryUsed) {
            alerts.push({
                severity: 'critical',
                message: `Critical memory usage: ${memPct}%`,
                routerId: routerData.id,
            });
        }
        
        // Save to database
        for (const alert of alerts) {
            this.db.prepare(`
                INSERT INTO alerts (router_id, severity, message) 
                VALUES (?, ?, ?)
            `).run(alert.routerId, alert.severity, alert.message);
        }
        
        return alerts;
    }
    
    async getRecentAlerts(routerId, limit = 50) {
        return this.db.prepare(`
            SELECT * FROM alerts 
            WHERE router_id = ? 
            ORDER BY created_at DESC 
            LIMIT ?
        `).all(routerId, limit);
    }
}
```

### Alert Notification UI

```html
<div id="alerts-container">
    <div class="alerts-header">
        <h3>Alerts</h3>
        <span class="badge badge-red" id="alert-count">0</span>
    </div>
    
    <div id="alerts-list"></div>
</div>

<script>
async function loadAlerts() {
    const data = await apiGet(`/routers/${currentRouterId}/alerts`);
    
    const list = document.getElementById('alerts-list');
    document.getElementById('alert-count').textContent = data.unacknowledged || 0;
    
    list.innerHTML = data.alerts.slice(0, 10).map(alert => `
        <div class="alert alert-${alert.severity}">
            <div class="alert-icon">${alert.severity === 'critical' ? '🔴' : '🟡'}</div>
            <div class="alert-content">
                <div class="alert-message">${escapeHtml(alert.message)}</div>
                <div class="alert-time">${formatTime(alert.created_at)}</div>
            </div>
            ${!alert.acknowledged ? `
                <button onclick="acknowledgeAlert(${alert.id})" class="btn-sm">
                    Ack
                </button>
            ` : ''}
        </div>
    `).join('');
}
</script>
```

---

## 56.8 User Management UI

### User Management Page

```html
<!-- users.html -->
<div class="page">
    <div class="page-header">
        <h2>User Management</h2>
        <button onclick="showAddUser()" class="btn-primary">+ Add User</button>
    </div>
    
    <table class="data-table" id="users-table">
        <thead>
            <tr>
                <th>Username</th>
                <th>Role</th>
                <th>Last Login</th>
                <th>Actions</th>
            </tr>
        </thead>
        <tbody id="users-body"></tbody>
    </table>
</div>

<!-- Add User Modal -->
<div id="add-user-modal" class="modal hidden">
    <div class="modal-content">
        <h3>Add New User</h3>
        <form onsubmit="addUser(event)">
            <div class="form-group">
                <label>Username</label>
                <input type="text" name="username" required>
            </div>
            <div class="form-group">
                <label>Password</label>
                <input type="password" name="password" required>
            </div>
            <div class="form-group">
                <label>Role</label>
                <select name="role">
                    <option value="viewer">Viewer (Read-only)</option>
                    <option value="operator">Operator</option>
                    <option value="admin">Admin</option>
                </select>
            </div>
            <div class="form-group">
                <label>Email</label>
                <input type="email" name="email">
            </div>
            <div class="modal-footer">
                <button type="button" onclick="closeModal()">Cancel</button>
                <button type="submit" class="btn-primary">Add User</button>
            </div>
        </form>
    </div>
</div>

<script>
async function loadUsers() {
    const data = await apiGet('/users');
    const tbody = document.getElementById('users-body');
    
    tbody.innerHTML = data.map(user => `
        <tr>
            <td>${escapeHtml(user.username)}</td>
            <td><span class="badge badge-${user.role}">${user.role}</span></td>
            <td>${user.last_login ? formatTime(user.last_login) : 'Never'}</td>
            <td>
                <button onclick="resetPassword(${user.id})" class="btn-sm">
                    Reset Password
                </button>
                <button onclick="deleteUser(${user.id})" class="btn-sm btn-danger">
                    Delete
                </button>
            </td>
        </tr>
    `).join('');
}
</script>
```

---

## 56.9 Export Reports

### Report Generation

```javascript
// src/services/ReportService.js

class ReportService {
    constructor(db) {
        this.db = db;
    }
    
    async generateRouterReport(routerId, startDate, endDate) {
        // ดึงข้อมูล metrics จาก database
        const metrics = this.db.prepare(`
            SELECT * FROM router_metrics 
            WHERE router_id = ? 
            AND timestamp BETWEEN ? AND ?
            ORDER BY timestamp ASC
        `).all(routerId, startDate, endDate);
        
        // คำนวณ statistics
        const cpuValues = metrics.map(m => m.cpu_load).filter(v => v !== null);
        const memValues = metrics.map(m => {
            if (!m.total_memory || !m.free_memory) return null;
            return ((m.total_memory - m.free_memory) / m.total_memory) * 100;
        }).filter(v => v !== null);
        
        return {
            routerId,
            period: { start: startDate, end: endDate },
            metrics: {
                cpuAvg: average(cpuValues),
                cpuMax: Math.max(...cpuValues),
                memAvg: average(memValues),
                memMax: Math.max(...memValues),
                totalDataPoints: metrics.length,
            },
            alerts: this.db.prepare(`
                SELECT severity, COUNT(*) as count 
                FROM alerts 
                WHERE router_id = ? AND created_at BETWEEN ? AND ?
                GROUP BY severity
            `).all(routerId, startDate, endDate),
        };
    }
    
    generateCSV(data) {
        const rows = [
            ['Timestamp', 'CPU%', 'Memory%'],
            ...data.metrics.map(m => [
                m.timestamp,
                m.cpu_load,
                m.total_memory ? 
                    Math.round((m.total_memory - m.free_memory) / m.total_memory * 100) : 0
            ])
        ];
        
        return rows.map(row => row.join(',')).join('\n');
    }
}
```

### Export Route

```javascript
// GET /api/routers/:id/export/csv
router.get('/:routerId/export/csv', requireAuth, async (req, res) => {
    const { from, to } = req.query;
    
    const reportService = new ReportService(db);
    const report = await reportService.generateRouterReport(
        req.params.routerId,
        from || new Date(Date.now() - 7 * 24 * 60 * 60 * 1000).toISOString(),
        to || new Date().toISOString()
    );
    
    const csv = reportService.generateCSV(report);
    
    res.setHeader('Content-Type', 'text/csv');
    res.setHeader('Content-Disposition', 
        `attachment; filename="router-${req.params.routerId}-report.csv"`);
    res.send(csv);
});
```

---

## 56.10 Lab: Complete Dashboard

### Lab Setup Script

```bash
#!/bin/bash
# setup_dashboard.sh - Complete dashboard setup

set -e

echo "Setting up MikroTik Complete Dashboard..."

# Project setup
mkdir -p mikrotik-dashboard/{src,public/{css,js,images}}
cd mikrotik-dashboard

# Package.json
cat > package.json << 'EOF'
{
  "name": "mikrotik-dashboard",
  "version": "1.0.0",
  "scripts": {
    "start": "node src/index.js",
    "dev": "nodemon src/index.js"
  },
  "dependencies": {
    "express": "^4.18.0",
    "socket.io": "^4.7.0",
    "node-routeros": "^1.4.0",
    "better-sqlite3": "^9.0.0",
    "jsonwebtoken": "^9.0.0",
    "bcryptjs": "^2.4.3",
    "dotenv": "^16.0.0"
  },
  "devDependencies": {
    "nodemon": "^3.0.0"
  }
}
EOF

npm install

# Environment file
cat > .env << 'EOF'
PORT=3000
JWT_SECRET=my-super-secret-key-change-in-production
MIKROTIK_HOST=192.168.1.1
MIKROTIK_USER=admin
MIKROTIK_PASS=
EOF

echo "Dashboard setup complete!"
echo "Run: npm start"
echo "Open: http://localhost:3000"
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Interface Stats | Real-time interface monitoring |
| Traffic Graphs | Chart.js live updating |
| Client List | Hotspot + DHCP clients |
| Log Viewer | Firewall log display with filters |
| System Resources | CPU, Memory, Disk gauges |
| Multi-router | Router selector component |
| Alerts | Threshold-based alert system |
| User Management | CRUD UI for system users |
| Export | CSV report generation |

---

[← Part 55: Web Dashboard Part 1](part-055-web-dashboard-1.md) | [Part 57: Network Monitor App →](part-057-network-monitor-app.md)
