# Part 58: User Self-Service Portal

## บทนำ

User Self-Service Portal ช่วยให้ลูกค้า ISP หรือผู้ใช้งานเครือข่ายสามารถจัดการบัญชีของตัวเองได้โดยไม่ต้องติดต่อ Support โดยตรง บทนี้จะสร้าง Portal ครบวงจรด้วย Node.js + Vue.js

---

## 58.1 Portal Architecture

```
┌────────────────────────────────────────────┐
│          Vue.js Frontend                    │
│    (Login / Dashboard / Account / Ticket)  │
└─────────────────┬──────────────────────────┘
                  │ REST API + WebSocket
┌─────────────────┴──────────────────────────┐
│          Node.js Backend                    │
│    (Express.js + JWT + Socket.io)           │
└──────┬───────────────────┬─────────────────┘
       │                   │
┌──────┴──────┐   ┌────────┴────────┐
│   MySQL/    │   │   MikroTik      │
│   SQLite    │   │   RouterOS API  │
└─────────────┘   └─────────────────┘
```

---

## 58.2 Authentication System

### Database Schema

```sql
-- users.sql
CREATE TABLE users (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    username VARCHAR(100) UNIQUE NOT NULL,
    email VARCHAR(200) UNIQUE NOT NULL,
    password_hash VARCHAR(255) NOT NULL,
    full_name VARCHAR(200),
    phone VARCHAR(20),
    plan_id INTEGER,
    account_status ENUM('active','suspended','cancelled') DEFAULT 'active',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    last_login DATETIME
);

CREATE TABLE plans (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    name VARCHAR(100) NOT NULL,
    speed_down INTEGER NOT NULL,  -- Mbps
    speed_up INTEGER NOT NULL,    -- Mbps
    price DECIMAL(10,2) NOT NULL,
    quota_gb INTEGER DEFAULT 0,   -- 0 = unlimited
    description TEXT
);

CREATE TABLE invoices (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER REFERENCES users(id),
    amount DECIMAL(10,2) NOT NULL,
    due_date DATE NOT NULL,
    paid_date DATE,
    status ENUM('pending','paid','overdue') DEFAULT 'pending',
    description TEXT,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE support_tickets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER REFERENCES users(id),
    subject VARCHAR(255) NOT NULL,
    description TEXT NOT NULL,
    status ENUM('open','in_progress','resolved','closed') DEFAULT 'open',
    priority ENUM('low','medium','high','critical') DEFAULT 'medium',
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE ticket_replies (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    ticket_id INTEGER REFERENCES support_tickets(id),
    user_id INTEGER REFERENCES users(id),
    is_staff BOOLEAN DEFAULT FALSE,
    message TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE notifications (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    user_id INTEGER REFERENCES users(id),
    title VARCHAR(255) NOT NULL,
    message TEXT NOT NULL,
    type ENUM('info','warning','error','success') DEFAULT 'info',
    read_at DATETIME,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

### Auth Service

```javascript
// backend/services/authService.js
const bcrypt = require('bcrypt');
const jwt = require('jsonwebtoken');
const db = require('../db');

const JWT_SECRET = process.env.JWT_SECRET || 'your-secret-key';
const JWT_EXPIRES = '7d';

class AuthService {
    async register(userData) {
        const { username, email, password, full_name, phone } = userData;
        
        // ตรวจสอบว่ามี user อยู่แล้วหรือไม่
        const existing = db.prepare('SELECT id FROM users WHERE email = ? OR username = ?')
            .get(email, username);
        
        if (existing) {
            throw new Error('Username or email already exists');
        }
        
        const password_hash = await bcrypt.hash(password, 12);
        
        const result = db.prepare(`
            INSERT INTO users (username, email, password_hash, full_name, phone)
            VALUES (?, ?, ?, ?, ?)
        `).run(username, email, password_hash, full_name, phone);
        
        return { id: result.lastInsertRowid, username, email };
    }
    
    async login(email, password) {
        const user = db.prepare('SELECT * FROM users WHERE email = ?').get(email);
        
        if (!user) {
            throw new Error('Invalid credentials');
        }
        
        const valid = await bcrypt.compare(password, user.password_hash);
        if (!valid) {
            throw new Error('Invalid credentials');
        }
        
        if (user.account_status !== 'active') {
            throw new Error(`Account is ${user.account_status}`);
        }
        
        // Update last login
        db.prepare('UPDATE users SET last_login = CURRENT_TIMESTAMP WHERE id = ?').run(user.id);
        
        const token = jwt.sign(
            { userId: user.id, username: user.username, email: user.email },
            JWT_SECRET,
            { expiresIn: JWT_EXPIRES }
        );
        
        return {
            token,
            user: {
                id: user.id,
                username: user.username,
                email: user.email,
                full_name: user.full_name,
            }
        };
    }
    
    verifyToken(token) {
        return jwt.verify(token, JWT_SECRET);
    }
    
    async changePassword(userId, oldPassword, newPassword) {
        const user = db.prepare('SELECT * FROM users WHERE id = ?').get(userId);
        
        const valid = await bcrypt.compare(oldPassword, user.password_hash);
        if (!valid) {
            throw new Error('Current password is incorrect');
        }
        
        if (newPassword.length < 8) {
            throw new Error('New password must be at least 8 characters');
        }
        
        const hash = await bcrypt.hash(newPassword, 12);
        db.prepare('UPDATE users SET password_hash = ? WHERE id = ?').run(hash, userId);
        
        return { success: true };
    }
}

module.exports = new AuthService();
```

---

## 58.3 Account Information

### Account API Routes

```javascript
// backend/routes/account.js
const express = require('express');
const router = express.Router();
const db = require('../db');
const { requireAuth } = require('../middleware/auth');
const RouterOSService = require('../services/routerosService');

// GET /api/account/profile
router.get('/profile', requireAuth, (req, res) => {
    const user = db.prepare(`
        SELECT u.id, u.username, u.email, u.full_name, u.phone,
               u.account_status, u.created_at, u.last_login,
               p.name as plan_name, p.speed_down, p.speed_up, p.price
        FROM users u
        LEFT JOIN plans p ON u.plan_id = p.id
        WHERE u.id = ?
    `).get(req.user.userId);
    
    if (!user) {
        return res.status(404).json({ error: 'User not found' });
    }
    
    res.json(user);
});

// PUT /api/account/profile
router.put('/profile', requireAuth, (req, res) => {
    const { full_name, phone } = req.body;
    
    db.prepare('UPDATE users SET full_name = ?, phone = ? WHERE id = ?')
        .run(full_name, phone, req.user.userId);
    
    res.json({ success: true, message: 'Profile updated' });
});

// GET /api/account/bandwidth
router.get('/bandwidth', requireAuth, async (req, res) => {
    try {
        const user = db.prepare('SELECT username FROM users WHERE id = ?')
            .get(req.user.userId);
        
        const stats = await RouterOSService.getHotspotUserStats(user.username);
        
        res.json({
            username: user.username,
            download_bytes: stats?.bytes_in || 0,
            upload_bytes: stats?.bytes_out || 0,
            session_time: stats?.uptime || '0s',
            connected: stats !== null,
        });
    } catch (err) {
        res.status(500).json({ error: 'Failed to fetch bandwidth stats' });
    }
});

module.exports = router;
```

---

## 58.4 Bandwidth Usage Display

### Vue.js Bandwidth Component

```vue
<!-- frontend/src/components/BandwidthWidget.vue -->
<template>
  <div class="bandwidth-widget">
    <h2>การใช้งาน Bandwidth</h2>
    
    <div class="stats-grid">
      <div class="stat-card download">
        <div class="icon">⬇</div>
        <div class="label">ดาวน์โหลด</div>
        <div class="value">{{ formatBytes(stats.download_bytes) }}</div>
      </div>
      
      <div class="stat-card upload">
        <div class="icon">⬆</div>
        <div class="label">อัพโหลด</div>
        <div class="value">{{ formatBytes(stats.upload_bytes) }}</div>
      </div>
      
      <div class="stat-card session">
        <div class="icon">⏱</div>
        <div class="label">เวลาเชื่อมต่อ</div>
        <div class="value">{{ stats.session_time || '-' }}</div>
      </div>
      
      <div class="stat-card status" :class="{ active: stats.connected }">
        <div class="icon">{{ stats.connected ? '🟢' : '🔴' }}</div>
        <div class="label">สถานะ</div>
        <div class="value">{{ stats.connected ? 'เชื่อมต่อแล้ว' : 'ไม่ได้เชื่อมต่อ' }}</div>
      </div>
    </div>
    
    <!-- Quota display (if plan has quota) -->
    <div v-if="plan.quota_gb > 0" class="quota-section">
      <h3>โควต้าที่ใช้</h3>
      <div class="quota-bar">
        <div class="quota-fill" :style="{ width: quotaPercent + '%' }" 
             :class="{ warning: quotaPercent > 80, danger: quotaPercent > 95 }">
        </div>
      </div>
      <p>ใช้ไป {{ usedGB }} GB จาก {{ plan.quota_gb }} GB ({{ quotaPercent.toFixed(1) }}%)</p>
    </div>
    
    <button @click="refreshStats" :disabled="loading" class="btn-refresh">
      {{ loading ? 'กำลังโหลด...' : 'รีเฟรช' }}
    </button>
  </div>
</template>

<script>
export default {
  name: 'BandwidthWidget',
  props: {
    plan: {
      type: Object,
      default: () => ({})
    }
  },
  
  data() {
    return {
      stats: {
        download_bytes: 0,
        upload_bytes: 0,
        session_time: '',
        connected: false,
      },
      loading: false,
    };
  },
  
  computed: {
    usedGB() {
      return ((this.stats.download_bytes + this.stats.upload_bytes) / 1024**3).toFixed(2);
    },
    
    quotaPercent() {
      if (!this.plan.quota_gb) return 0;
      return Math.min(100, (parseFloat(this.usedGB) / this.plan.quota_gb) * 100);
    }
  },
  
  methods: {
    formatBytes(bytes) {
      if (bytes >= 1024**3) return (bytes / 1024**3).toFixed(2) + ' GB';
      if (bytes >= 1024**2) return (bytes / 1024**2).toFixed(2) + ' MB';
      return (bytes / 1024).toFixed(2) + ' KB';
    },
    
    async refreshStats() {
      this.loading = true;
      try {
        const res = await fetch('/api/account/bandwidth', {
          headers: { Authorization: `Bearer ${localStorage.getItem('token')}` }
        });
        this.stats = await res.json();
      } catch (err) {
        console.error('Failed to fetch stats:', err);
      } finally {
        this.loading = false;
      }
    }
  },
  
  mounted() {
    this.refreshStats();
    // Auto-refresh ทุก 60 วินาที
    this.refreshInterval = setInterval(this.refreshStats, 60000);
  },
  
  beforeUnmount() {
    clearInterval(this.refreshInterval);
  }
};
</script>
```

---

## 58.5 Invoice Display

### Invoice API

```javascript
// backend/routes/invoices.js
const express = require('express');
const router = express.Router();
const db = require('../db');
const { requireAuth } = require('../middleware/auth');

// GET /api/invoices - รายการ invoices ทั้งหมด
router.get('/', requireAuth, (req, res) => {
    const { page = 1, limit = 10, status } = req.query;
    const offset = (page - 1) * limit;
    
    let query = 'SELECT * FROM invoices WHERE user_id = ?';
    const params = [req.user.userId];
    
    if (status) {
        query += ' AND status = ?';
        params.push(status);
    }
    
    query += ' ORDER BY created_at DESC LIMIT ? OFFSET ?';
    params.push(parseInt(limit), offset);
    
    const invoices = db.prepare(query).all(...params);
    const total = db.prepare('SELECT COUNT(*) as count FROM invoices WHERE user_id = ?')
        .get(req.user.userId).count;
    
    res.json({ invoices, total, page: parseInt(page), limit: parseInt(limit) });
});

// GET /api/invoices/:id - รายละเอียด invoice
router.get('/:id', requireAuth, (req, res) => {
    const invoice = db.prepare(`
        SELECT i.*, u.full_name, u.email, p.name as plan_name
        FROM invoices i
        JOIN users u ON i.user_id = u.id
        LEFT JOIN plans p ON u.plan_id = p.id
        WHERE i.id = ? AND i.user_id = ?
    `).get(req.params.id, req.user.userId);
    
    if (!invoice) {
        return res.status(404).json({ error: 'Invoice not found' });
    }
    
    res.json(invoice);
});

module.exports = router;
```

---

## 58.6 Password Change

### Change Password Component

```vue
<!-- frontend/src/components/ChangePassword.vue -->
<template>
  <div class="change-password">
    <h2>เปลี่ยนรหัสผ่าน</h2>
    
    <form @submit.prevent="handleSubmit" class="password-form">
      <div class="form-group">
        <label>รหัสผ่านปัจจุบัน</label>
        <input v-model="form.current" type="password" required
               placeholder="รหัสผ่านปัจจุบัน" />
      </div>
      
      <div class="form-group">
        <label>รหัสผ่านใหม่</label>
        <input v-model="form.newPassword" type="password" required
               placeholder="อย่างน้อย 8 ตัวอักษร" minlength="8" />
        <div class="password-strength" :class="strengthClass">
          {{ strengthText }}
        </div>
      </div>
      
      <div class="form-group">
        <label>ยืนยันรหัสผ่านใหม่</label>
        <input v-model="form.confirm" type="password" required
               placeholder="ยืนยันรหัสผ่านใหม่" />
        <span v-if="form.newPassword && form.confirm && !passwordsMatch" class="error-text">
          รหัสผ่านไม่ตรงกัน
        </span>
      </div>
      
      <div v-if="message" :class="['message', messageType]">
        {{ message }}
      </div>
      
      <button type="submit" :disabled="!canSubmit || loading" class="btn-primary">
        {{ loading ? 'กำลังบันทึก...' : 'เปลี่ยนรหัสผ่าน' }}
      </button>
    </form>
  </div>
</template>

<script>
export default {
  name: 'ChangePassword',
  
  data() {
    return {
      form: { current: '', newPassword: '', confirm: '' },
      loading: false,
      message: '',
      messageType: 'info',
    };
  },
  
  computed: {
    passwordsMatch() {
      return this.form.newPassword === this.form.confirm;
    },
    
    strengthLevel() {
      const p = this.form.newPassword;
      if (p.length < 8) return 0;
      let score = 0;
      if (/[A-Z]/.test(p)) score++;
      if (/[0-9]/.test(p)) score++;
      if (/[^A-Za-z0-9]/.test(p)) score++;
      if (p.length >= 12) score++;
      return score;
    },
    
    strengthClass() {
      return ['', 'weak', 'fair', 'good', 'strong'][this.strengthLevel];
    },
    
    strengthText() {
      return ['', 'อ่อนแอ', 'พอใช้', 'ดี', 'แข็งแกร่ง'][this.strengthLevel] || '';
    },
    
    canSubmit() {
      return this.form.current && this.form.newPassword.length >= 8 
          && this.passwordsMatch;
    }
  },
  
  methods: {
    async handleSubmit() {
      this.loading = true;
      this.message = '';
      
      try {
        const res = await fetch('/api/account/change-password', {
          method: 'POST',
          headers: {
            'Content-Type': 'application/json',
            Authorization: `Bearer ${localStorage.getItem('token')}`
          },
          body: JSON.stringify({
            currentPassword: this.form.current,
            newPassword: this.form.newPassword,
          })
        });
        
        const data = await res.json();
        
        if (res.ok) {
          this.message = 'เปลี่ยนรหัสผ่านสำเร็จ';
          this.messageType = 'success';
          this.form = { current: '', newPassword: '', confirm: '' };
        } else {
          this.message = data.error || 'เกิดข้อผิดพลาด';
          this.messageType = 'error';
        }
      } catch (err) {
        this.message = 'ไม่สามารถเชื่อมต่อได้';
        this.messageType = 'error';
      } finally {
        this.loading = false;
      }
    }
  }
};
</script>
```

---

## 58.7 Support Tickets

### Ticket API

```javascript
// backend/routes/tickets.js
const express = require('express');
const router = express.Router();
const db = require('../db');
const { requireAuth } = require('../middleware/auth');

// POST /api/tickets - สร้าง ticket ใหม่
router.post('/', requireAuth, (req, res) => {
    const { subject, description, priority = 'medium' } = req.body;
    
    if (!subject || !description) {
        return res.status(400).json({ error: 'Subject and description are required' });
    }
    
    const result = db.prepare(`
        INSERT INTO support_tickets (user_id, subject, description, priority)
        VALUES (?, ?, ?, ?)
    `).run(req.user.userId, subject, description, priority);
    
    // Notify admins (send email or notification)
    // sendAdminNotification(...)
    
    res.status(201).json({ 
        id: result.lastInsertRowid, 
        message: 'Ticket created successfully' 
    });
});

// GET /api/tickets - รายการ tickets ของ user
router.get('/', requireAuth, (req, res) => {
    const { status } = req.query;
    
    let query = 'SELECT * FROM support_tickets WHERE user_id = ?';
    const params = [req.user.userId];
    
    if (status) {
        query += ' AND status = ?';
        params.push(status);
    }
    
    query += ' ORDER BY created_at DESC';
    
    const tickets = db.prepare(query).all(...params);
    res.json(tickets);
});

// POST /api/tickets/:id/reply
router.post('/:id/reply', requireAuth, (req, res) => {
    const ticket = db.prepare('SELECT * FROM support_tickets WHERE id = ? AND user_id = ?')
        .get(req.params.id, req.user.userId);
    
    if (!ticket) return res.status(404).json({ error: 'Ticket not found' });
    if (ticket.status === 'closed') return res.status(400).json({ error: 'Ticket is closed' });
    
    const { message } = req.body;
    
    db.prepare(`
        INSERT INTO ticket_replies (ticket_id, user_id, is_staff, message)
        VALUES (?, ?, FALSE, ?)
    `).run(req.params.id, req.user.userId, message);
    
    db.prepare("UPDATE support_tickets SET status = 'open', updated_at = CURRENT_TIMESTAMP WHERE id = ?")
        .run(req.params.id);
    
    res.json({ success: true });
});

module.exports = router;
```

---

## 58.8 Notification Preferences

### Notification Settings

```javascript
// backend/routes/notifications.js
const express = require('express');
const router = express.Router();
const db = require('../db');
const { requireAuth } = require('../middleware/auth');

// GET /api/notifications - รายการ notifications
router.get('/', requireAuth, (req, res) => {
    const { unread_only } = req.query;
    
    let query = 'SELECT * FROM notifications WHERE user_id = ?';
    const params = [req.user.userId];
    
    if (unread_only === 'true') {
        query += ' AND read_at IS NULL';
    }
    
    query += ' ORDER BY created_at DESC LIMIT 50';
    
    const notifications = db.prepare(query).all(...params);
    
    const unreadCount = db.prepare('SELECT COUNT(*) as count FROM notifications WHERE user_id = ? AND read_at IS NULL')
        .get(req.user.userId).count;
    
    res.json({ notifications, unread_count: unreadCount });
});

// POST /api/notifications/read-all - mark all as read
router.post('/read-all', requireAuth, (req, res) => {
    db.prepare('UPDATE notifications SET read_at = CURRENT_TIMESTAMP WHERE user_id = ? AND read_at IS NULL')
        .run(req.user.userId);
    
    res.json({ success: true });
});

module.exports = router;
```

---

## 58.9 Mobile App (PWA)

### PWA Manifest

```json
{
  "name": "Network User Portal",
  "short_name": "MyNetwork",
  "description": "จัดการบัญชี Network ของคุณ",
  "start_url": "/",
  "display": "standalone",
  "background_color": "#ffffff",
  "theme_color": "#3b82f6",
  "icons": [
    {
      "src": "/icons/icon-192.png",
      "sizes": "192x192",
      "type": "image/png",
      "purpose": "any maskable"
    },
    {
      "src": "/icons/icon-512.png",
      "sizes": "512x512",
      "type": "image/png"
    }
  ]
}
```

### Service Worker

```javascript
// public/sw.js
const CACHE_NAME = 'portal-v1';
const STATIC_ASSETS = ['/', '/css/app.css', '/js/app.js', '/manifest.json'];

self.addEventListener('install', event => {
    event.waitUntil(
        caches.open(CACHE_NAME)
            .then(cache => cache.addAll(STATIC_ASSETS))
    );
});

self.addEventListener('fetch', event => {
    if (event.request.url.includes('/api/')) {
        // ไม่ cache API calls
        return;
    }
    
    event.respondWith(
        caches.match(event.request)
            .then(cached => cached || fetch(event.request))
    );
});
```

---

## 58.10 Lab: Complete User Portal

```bash
#!/bin/bash
# setup_user_portal.sh

echo "Setting up User Self-Service Portal..."

mkdir -p portal/{backend/{routes,services,middleware},frontend/src/{components,views}}
cd portal

# Backend package.json
cat > backend/package.json << 'EOF'
{
  "name": "user-portal-backend",
  "version": "1.0.0",
  "scripts": {
    "start": "node server.js",
    "dev": "nodemon server.js"
  },
  "dependencies": {
    "express": "^4.18.0",
    "better-sqlite3": "^8.0.0",
    "bcrypt": "^5.1.0",
    "jsonwebtoken": "^9.0.0",
    "routeros-api": "^0.1.3"
  },
  "devDependencies": {
    "nodemon": "^3.0.0"
  }
}
EOF

# Install dependencies
cd backend
npm install
cd ..

echo ""
echo "Portal setup complete!"
echo ""
echo "Structure:"
echo "  portal/backend/  - Node.js API server"
echo "  portal/frontend/ - Vue.js frontend"
echo ""
echo "Start backend: cd portal/backend && npm start"
echo "Start frontend: cd portal/frontend && npm run dev"
```

---

## Summary

| Feature | Technology |
|---------|-----------|
| Authentication | JWT + bcrypt |
| Profile Management | REST API |
| Bandwidth Display | RouterOS API + Vue.js |
| Invoice Display | SQLite + REST API |
| Password Change | bcrypt re-hash |
| Support Tickets | CRUD API |
| Notifications | Push + in-app |
| Mobile App | PWA |

---

[← Part 57: Network Monitor App](part-057-network-monitor-app.md) | [Part 59: Hotspot Portal →](part-059-hotspot-portal.md)
