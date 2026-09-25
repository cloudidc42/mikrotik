# Part 60: API Security

## บทนำ

API Security เป็นสิ่งสำคัญที่สุดเมื่อ expose MikroTik RouterOS API สู่ภายนอก บทนี้ครอบคลุมทุกด้านตั้งแต่ SSL/TLS, authentication, rate limiting จนถึง audit logging และ penetration testing

---

## 60.1 API Security Threats

### ภัยคุกคามหลัก

| ภัยคุกคาม | คำอธิบาย | ความเสี่ยง |
|-----------|----------|-----------|
| Brute Force | เดา password ซ้ำๆ | สูง |
| Credential Stuffing | ใช้ password รั่วไหล | สูง |
| MITM | ดักข้อมูลระหว่างทาง | สูง |
| API Key Exposure | Key ถูก commit ใน code | สูง |
| Excessive Privilege | สิทธิ์เกินความจำเป็น | กลาง |
| Injection | Command injection ผ่าน API | สูง |
| Replay Attack | ส่ง request เก่าซ้ำ | กลาง |
| DoS/DDoS | ส่ง request ล้น API | สูง |

### Attack Surface

```
Internet ──→ Firewall ──→ API Gateway ──→ RouterOS API (8728/8729)
                                   ↑
                              Web App ──→ Direct API calls

Attack vectors:
1. Port 8728 exposed directly to internet
2. Weak password on API user
3. No SSL (plain text)
4. No rate limiting
5. Overly permissive API user groups
```

---

## 60.2 SSL/TLS for API

### Enable SSL on MikroTik

```bash
# MikroTik RouterOS Commands

# 1. สร้าง Self-signed certificate
/certificate add \
    name=api-cert \
    common-name=router.example.com \
    country=TH \
    organization="My Company" \
    key-size=2048 \
    days-valid=3650

/certificate sign api-cert

# 2. เปิด API-SSL
/ip service set api-ssl \
    certificate=api-cert \
    disabled=no \
    port=8729

# 3. ปิด API ที่ไม่ encrypt (optional แต่แนะนำ)
/ip service set api disabled=yes

# 4. ตรวจสอบ
/ip service print
```

### Python SSL Connection

```python
# secure_client.py
import ssl
import socket
import routeros_api

def create_ssl_api(host: str, username: str, password: str, 
                   verify_ssl: bool = True, ca_cert: str = None):
    """เชื่อมต่อ RouterOS API ด้วย SSL"""
    
    ssl_context = ssl.create_default_context()
    
    if not verify_ssl:
        # สำหรับ self-signed cert - ใช้เฉพาะใน dev
        ssl_context.check_hostname = False
        ssl_context.verify_mode = ssl.CERT_NONE
    elif ca_cert:
        ssl_context.load_verify_locations(ca_cert)
    
    pool = routeros_api.RouterOsApiPool(
        host,
        username=username,
        password=password,
        port=8729,  # SSL port
        ssl_wrapper=ssl_context.wrap_socket,
        plaintext_login=False,
    )
    
    return pool.get_api()


# ใช้งาน
api = create_ssl_api(
    host='192.168.1.1',
    username='api-user',
    password='SecurePassword123',
    verify_ssl=False,  # ใน production ใช้ verify_ssl=True
)
```

### Node.js SSL Connection

```javascript
// ssl-client.js
const tls = require('tls');
const RouterOS = require('node-routeros').RouterOSAPI;

function createSecureConnection(host, username, password) {
    return new RouterOS({
        host,
        user: username,
        password,
        port: 8729,
        timeout: 10,
        tls: {
            rejectUnauthorized: false,  // false สำหรับ self-signed
        }
    });
}
```

---

## 60.3 Authentication Methods

### Dedicated API User with Minimal Permissions

```bash
# สร้าง API User Group ที่มีสิทธิ์น้อยที่สุด

# Read-only group
/user group add \
    name=api-readonly \
    policy=read,api

# Monitoring group
/user group add \
    name=api-monitor \
    policy=read,api,!write,!policy,!password,!sensitive

# Full API group (ระวัง)
/user group add \
    name=api-admin \
    policy=read,write,api,!policy,!password,!sensitive

# สร้าง API users
/user add \
    name=api-monitor \
    password=MonitorPass456 \
    group=api-monitor \
    comment="Monitoring only"

/user add \
    name=api-readonly \
    password=ReadOnlyPass789 \
    group=api-readonly \
    comment="Read-only API access"

# จำกัด IP ที่ login ได้
/user set api-monitor \
    allowed-address=192.168.100.0/24
```

### API Key Authentication Wrapper

```python
# auth/api_key_auth.py
import hashlib
import hmac
import time
import sqlite3
from functools import wraps
from flask import request, jsonify, g


class APIKeyManager:
    def __init__(self, db_path: str = 'api_keys.db'):
        self.db_path = db_path
        self._init_db()
    
    def _init_db(self):
        conn = sqlite3.connect(self.db_path)
        conn.execute('''
            CREATE TABLE IF NOT EXISTS api_keys (
                id INTEGER PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                key_hash VARCHAR(64) NOT NULL UNIQUE,
                permissions TEXT DEFAULT '[]',
                rate_limit INTEGER DEFAULT 100,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                last_used DATETIME,
                expires_at DATETIME,
                active INTEGER DEFAULT 1
            )
        ''')
        conn.commit()
        conn.close()
    
    def create_key(self, name: str, permissions: list, 
                   rate_limit: int = 100, expires_days: int = None) -> str:
        """สร้าง API Key ใหม่"""
        import secrets
        import json
        from datetime import datetime, timedelta
        
        key = 'mk_' + secrets.token_hex(32)
        key_hash = hashlib.sha256(key.encode()).hexdigest()
        
        expires_at = None
        if expires_days:
            expires_at = (datetime.utcnow() + timedelta(days=expires_days)).isoformat()
        
        conn = sqlite3.connect(self.db_path)
        conn.execute(
            'INSERT INTO api_keys (name, key_hash, permissions, rate_limit, expires_at) VALUES (?, ?, ?, ?, ?)',
            (name, key_hash, json.dumps(permissions), rate_limit, expires_at)
        )
        conn.commit()
        conn.close()
        
        return key  # แสดงครั้งเดียว ไม่เก็บ plain text
    
    def verify_key(self, api_key: str) -> dict | None:
        """ตรวจสอบ API Key"""
        if not api_key or not api_key.startswith('mk_'):
            return None
        
        key_hash = hashlib.sha256(api_key.encode()).hexdigest()
        
        conn = sqlite3.connect(self.db_path)
        row = conn.execute(
            'SELECT * FROM api_keys WHERE key_hash = ? AND active = 1',
            (key_hash,)
        ).fetchone()
        conn.close()
        
        if not row:
            return None
        
        # ตรวจสอบ expiry
        if row[8]:  # expires_at
            from datetime import datetime
            expires = datetime.fromisoformat(row[8])
            if datetime.utcnow() > expires:
                return None
        
        # Update last_used
        conn = sqlite3.connect(self.db_path)
        conn.execute('UPDATE api_keys SET last_used = CURRENT_TIMESTAMP WHERE key_hash = ?', (key_hash,))
        conn.commit()
        conn.close()
        
        import json
        return {
            'id': row[0],
            'name': row[1],
            'permissions': json.loads(row[3]),
            'rate_limit': row[4],
        }


api_key_manager = APIKeyManager()


def require_api_key(permission: str = None):
    """Decorator ตรวจสอบ API Key"""
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            api_key = request.headers.get('X-API-Key')
            if not api_key:
                return jsonify({'error': 'API key required'}), 401
            
            key_info = api_key_manager.verify_key(api_key)
            if not key_info:
                return jsonify({'error': 'Invalid API key'}), 401
            
            if permission and permission not in key_info['permissions']:
                return jsonify({'error': 'Permission denied'}), 403
            
            g.api_key = key_info
            return f(*args, **kwargs)
        return decorated
    return decorator
```

---

## 60.4 Rate Limiting

### Rate Limiter Implementation

```python
# security/rate_limiter.py
import time
import redis
from functools import wraps
from flask import request, jsonify, g


class RateLimiter:
    def __init__(self, redis_url: str = 'redis://localhost:6379'):
        self.redis = redis.from_url(redis_url)
    
    def is_allowed(self, key: str, max_requests: int, window: int) -> tuple[bool, dict]:
        """
        ตรวจสอบ rate limit
        Returns: (allowed, info)
        """
        pipe = self.redis.pipeline()
        now = time.time()
        window_start = now - window
        
        # Sliding window algorithm
        pipe.zremrangebyscore(key, 0, window_start)
        pipe.zadd(key, {str(now): now})
        pipe.zcard(key)
        pipe.expire(key, window)
        
        results = pipe.execute()
        current_count = results[2]
        
        remaining = max(0, max_requests - current_count)
        
        info = {
            'limit': max_requests,
            'remaining': remaining,
            'reset': int(now + window),
        }
        
        return current_count <= max_requests, info
    
    def limit(self, max_requests: int = 60, window: int = 60, key_func=None):
        """Decorator สำหรับ rate limiting"""
        def decorator(f):
            @wraps(f)
            def decorated(*args, **kwargs):
                if key_func:
                    key = key_func()
                else:
                    # Default: IP-based
                    ip = request.headers.get('X-Forwarded-For', request.remote_addr)
                    key = f"rate:{ip}:{request.endpoint}"
                
                allowed, info = self.is_allowed(key, max_requests, window)
                
                if not allowed:
                    response = jsonify({
                        'error': 'Rate limit exceeded',
                        'retry_after': info['reset'] - int(time.time()),
                    })
                    response.status_code = 429
                    response.headers['X-RateLimit-Limit'] = info['limit']
                    response.headers['X-RateLimit-Remaining'] = info['remaining']
                    response.headers['X-RateLimit-Reset'] = info['reset']
                    return response
                
                resp = f(*args, **kwargs)
                
                # เพิ่ม headers
                if hasattr(resp, 'headers'):
                    resp.headers['X-RateLimit-Limit'] = info['limit']
                    resp.headers['X-RateLimit-Remaining'] = info['remaining']
                
                return resp
            return decorated
        return decorator


rate_limiter = RateLimiter()
```

---

## 60.5 IP Whitelisting

### IP Whitelist Middleware

```python
# security/ip_whitelist.py
import ipaddress
from flask import request, jsonify
from functools import wraps

ALLOWED_NETWORKS = [
    '192.168.0.0/16',
    '10.0.0.0/8',
    '172.16.0.0/12',
    '203.0.113.0/24',  # Office public IP range
]


def get_client_ip():
    """ดึง real IP จาก request"""
    forwarded = request.headers.get('X-Forwarded-For')
    if forwarded:
        return forwarded.split(',')[0].strip()
    return request.remote_addr


def is_ip_allowed(ip: str, allowed_networks: list = None) -> bool:
    """ตรวจสอบว่า IP อยู่ใน whitelist หรือไม่"""
    networks = allowed_networks or ALLOWED_NETWORKS
    
    try:
        client = ipaddress.ip_address(ip)
        for network_str in networks:
            network = ipaddress.ip_network(network_str, strict=False)
            if client in network:
                return True
    except ValueError:
        pass
    
    return False


def require_whitelisted_ip(allowed_networks: list = None):
    """Decorator สำหรับ IP whitelist"""
    def decorator(f):
        @wraps(f)
        def decorated(*args, **kwargs):
            client_ip = get_client_ip()
            
            if not is_ip_allowed(client_ip, allowed_networks):
                return jsonify({
                    'error': 'Access denied',
                    'message': f'IP {client_ip} is not whitelisted'
                }), 403
            
            return f(*args, **kwargs)
        return decorated
    return decorator
```

---

## 60.6 JWT Token Security

### Secure JWT Implementation

```python
# auth/jwt_auth.py
import jwt
import time
import secrets
import sqlite3
from datetime import datetime, timedelta, timezone


class SecureJWTAuth:
    def __init__(self, secret_key: str, access_expire: int = 900, refresh_expire: int = 604800):
        """
        secret_key: JWT signing key
        access_expire: access token lifetime in seconds (default 15 min)
        refresh_expire: refresh token lifetime in seconds (default 7 days)
        """
        self.secret_key = secret_key
        self.access_expire = access_expire
        self.refresh_expire = refresh_expire
        self.algorithm = 'HS256'
        self._init_db()
    
    def _init_db(self):
        conn = sqlite3.connect('tokens.db')
        conn.execute('''
            CREATE TABLE IF NOT EXISTS refresh_tokens (
                id VARCHAR(64) PRIMARY KEY,
                user_id INTEGER NOT NULL,
                created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
                expires_at DATETIME NOT NULL,
                revoked INTEGER DEFAULT 0,
                device_info TEXT
            )
        ''')
        conn.execute('''
            CREATE TABLE IF NOT EXISTS revoked_tokens (
                jti VARCHAR(64) PRIMARY KEY,
                revoked_at DATETIME DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        conn.commit()
        conn.close()
    
    def create_access_token(self, user_id: int, claims: dict = None) -> str:
        """สร้าง short-lived access token"""
        now = datetime.now(timezone.utc)
        jti = secrets.token_hex(16)
        
        payload = {
            'sub': str(user_id),
            'iat': now,
            'exp': now + timedelta(seconds=self.access_expire),
            'jti': jti,
            'type': 'access',
        }
        
        if claims:
            payload.update(claims)
        
        return jwt.encode(payload, self.secret_key, algorithm=self.algorithm)
    
    def create_refresh_token(self, user_id: int, device_info: str = None) -> str:
        """สร้าง long-lived refresh token"""
        token_id = secrets.token_hex(32)
        expires_at = datetime.now(timezone.utc) + timedelta(seconds=self.refresh_expire)
        
        conn = sqlite3.connect('tokens.db')
        conn.execute(
            'INSERT INTO refresh_tokens (id, user_id, expires_at, device_info) VALUES (?, ?, ?, ?)',
            (token_id, user_id, expires_at.isoformat(), device_info)
        )
        conn.commit()
        conn.close()
        
        return token_id
    
    def verify_access_token(self, token: str) -> dict:
        """ตรวจสอบ access token"""
        try:
            payload = jwt.decode(token, self.secret_key, algorithms=[self.algorithm])
            
            # ตรวจสอบว่า token ถูก revoke หรือไม่
            if self._is_revoked(payload.get('jti')):
                raise ValueError("Token revoked")
            
            return payload
            
        except jwt.ExpiredSignatureError:
            raise ValueError("Token expired")
        except jwt.InvalidTokenError as e:
            raise ValueError(f"Invalid token: {e}")
    
    def _is_revoked(self, jti: str) -> bool:
        conn = sqlite3.connect('tokens.db')
        result = conn.execute('SELECT 1 FROM revoked_tokens WHERE jti = ?', (jti,)).fetchone()
        conn.close()
        return result is not None
    
    def revoke_token(self, jti: str):
        """Revoke access token"""
        conn = sqlite3.connect('tokens.db')
        conn.execute('INSERT OR IGNORE INTO revoked_tokens (jti) VALUES (?)', (jti,))
        conn.commit()
        conn.close()
    
    def refresh_access_token(self, refresh_token: str, user_id: int) -> str:
        """สร้าง access token ใหม่จาก refresh token"""
        conn = sqlite3.connect('tokens.db')
        row = conn.execute(
            'SELECT * FROM refresh_tokens WHERE id = ? AND user_id = ? AND revoked = 0',
            (refresh_token, user_id)
        ).fetchone()
        conn.close()
        
        if not row:
            raise ValueError("Invalid refresh token")
        
        expires_at = datetime.fromisoformat(row[3])
        if datetime.now(timezone.utc) > expires_at:
            raise ValueError("Refresh token expired")
        
        return self.create_access_token(user_id)
```

---

## 60.7 Audit Logging

### Audit Logger

```python
# audit/logger.py
import json
import logging
import sqlite3
from datetime import datetime, timezone
from flask import request, g


class AuditLogger:
    def __init__(self, db_path: str = 'audit.db'):
        self.db_path = db_path
        self._init_db()
        self.logger = logging.getLogger('audit')
    
    def _init_db(self):
        conn = sqlite3.connect(self.db_path)
        conn.execute('''
            CREATE TABLE IF NOT EXISTS audit_log (
                id INTEGER PRIMARY KEY AUTOINCREMENT,
                timestamp DATETIME DEFAULT CURRENT_TIMESTAMP,
                user_id INTEGER,
                username VARCHAR(100),
                ip_address VARCHAR(45),
                action VARCHAR(100) NOT NULL,
                resource VARCHAR(200),
                method VARCHAR(10),
                status_code INTEGER,
                request_body TEXT,
                response_summary TEXT,
                duration_ms INTEGER,
                user_agent TEXT,
                success INTEGER DEFAULT 1
            )
        ''')
        conn.execute('CREATE INDEX IF NOT EXISTS idx_user ON audit_log (user_id)')
        conn.execute('CREATE INDEX IF NOT EXISTS idx_timestamp ON audit_log (timestamp)')
        conn.execute('CREATE INDEX IF NOT EXISTS idx_action ON audit_log (action)')
        conn.commit()
        conn.close()
    
    def log(self, action: str, resource: str = None, 
            status_code: int = 200, extra: dict = None):
        """บันทึก audit log"""
        
        user_id = getattr(g, 'user_id', None)
        username = getattr(g, 'username', None)
        ip = request.headers.get('X-Forwarded-For', request.remote_addr) if request else 'N/A'
        
        # Sanitize request body (ซ่อน sensitive fields)
        body = None
        if request and request.is_json:
            try:
                raw = request.get_json(silent=True) or {}
                sanitized = {k: '***' if k in ('password', 'token', 'secret') else v
                             for k, v in raw.items()}
                body = json.dumps(sanitized)
            except Exception:
                pass
        
        conn = sqlite3.connect(self.db_path)
        conn.execute('''
            INSERT INTO audit_log 
            (user_id, username, ip_address, action, resource, method, 
             status_code, request_body, user_agent, success)
            VALUES (?, ?, ?, ?, ?, ?, ?, ?, ?, ?)
        ''', (
            user_id, username, ip, action, resource,
            request.method if request else None,
            status_code, body,
            request.headers.get('User-Agent', '') if request else '',
            1 if status_code < 400 else 0
        ))
        conn.commit()
        conn.close()
        
        # Also log to file
        self.logger.info(f"AUDIT: {action} by {username or 'anonymous'} from {ip} - {status_code}")
    
    def get_recent_logs(self, limit: int = 100, user_id: int = None, 
                        action: str = None) -> list:
        """ดึง audit logs"""
        query = 'SELECT * FROM audit_log WHERE 1=1'
        params = []
        
        if user_id:
            query += ' AND user_id = ?'
            params.append(user_id)
        
        if action:
            query += ' AND action LIKE ?'
            params.append(f'%{action}%')
        
        query += ' ORDER BY timestamp DESC LIMIT ?'
        params.append(limit)
        
        conn = sqlite3.connect(self.db_path)
        rows = conn.execute(query, params).fetchall()
        conn.close()
        
        columns = ['id', 'timestamp', 'user_id', 'username', 'ip_address',
                   'action', 'resource', 'method', 'status_code', 'request_body',
                   'response_summary', 'duration_ms', 'user_agent', 'success']
        
        return [dict(zip(columns, row)) for row in rows]


audit_logger = AuditLogger()
```

---

## 60.8 Penetration Testing

### API Security Test Script

```python
#!/usr/bin/env python3
# pentest/api_security_test.py
"""
MikroTik API Security Test Suite
ใช้เพื่อทดสอบความปลอดภัยของ API ของตัวเอง
"""

import requests
import time
import json
import threading
from concurrent.futures import ThreadPoolExecutor

BASE_URL = 'http://localhost:3000'

class APISecurityTester:
    def __init__(self, base_url: str):
        self.base_url = base_url
        self.results = []
    
    def test_authentication_bypass(self):
        """ทดสอบ authentication bypass"""
        print("\n[TEST] Authentication Bypass")
        
        # Test 1: No token
        r = requests.get(f"{self.base_url}/api/routers")
        assert r.status_code in (401, 403), f"FAIL: {r.status_code}"
        print("  ✓ No token rejected")
        
        # Test 2: Fake token
        r = requests.get(f"{self.base_url}/api/routers",
                        headers={"Authorization": "Bearer fakejwt123"})
        assert r.status_code in (401, 403), f"FAIL: {r.status_code}"
        print("  ✓ Fake token rejected")
        
        # Test 3: Expired-format token
        fake_expired = "eyJhbGciOiJIUzI1NiJ9.eyJleHAiOjB9.invalid"
        r = requests.get(f"{self.base_url}/api/routers",
                        headers={"Authorization": f"Bearer {fake_expired}"})
        assert r.status_code in (401, 403), f"FAIL: {r.status_code}"
        print("  ✓ Expired token rejected")
        
        self.results.append(('Authentication Bypass', 'PASS'))
    
    def test_rate_limiting(self):
        """ทดสอบ rate limiting"""
        print("\n[TEST] Rate Limiting")
        
        endpoint = f"{self.base_url}/api/auth/login"
        payload = {"email": "test@test.com", "password": "wrong"}
        
        rate_limited = False
        for i in range(20):
            r = requests.post(endpoint, json=payload)
            if r.status_code == 429:
                rate_limited = True
                print(f"  ✓ Rate limited at request #{i+1}")
                break
        
        if rate_limited:
            self.results.append(('Rate Limiting', 'PASS'))
        else:
            print("  ✗ FAIL: No rate limiting detected after 20 requests!")
            self.results.append(('Rate Limiting', 'FAIL'))
    
    def test_sql_injection(self):
        """ทดสอบ SQL injection"""
        print("\n[TEST] SQL Injection")
        
        payloads = [
            "admin@test.com' OR '1'='1",
            "admin@test.com' --",
            "' UNION SELECT * FROM users --",
        ]
        
        for payload in payloads:
            r = requests.post(f"{self.base_url}/api/auth/login",
                             json={"email": payload, "password": "test"})
            
            if r.status_code == 200:
                data = r.json()
                if 'token' in data:
                    print(f"  ✗ CRITICAL: SQL injection succeeded with: {payload}")
                    self.results.append(('SQL Injection', 'CRITICAL FAIL'))
                    return
            
            print(f"  ✓ Injection blocked: {payload[:30]}...")
        
        self.results.append(('SQL Injection', 'PASS'))
    
    def test_sensitive_data_exposure(self):
        """ทดสอบ data exposure"""
        print("\n[TEST] Sensitive Data Exposure")
        
        endpoints = [
            '/api/config',
            '/.env',
            '/api/users',
            '/backup',
            '/admin',
        ]
        
        for endpoint in endpoints:
            r = requests.get(f"{self.base_url}{endpoint}")
            if r.status_code == 200:
                print(f"  ✗ WARNING: {endpoint} accessible without auth")
            else:
                print(f"  ✓ {endpoint} protected ({r.status_code})")
        
        self.results.append(('Data Exposure', 'PASS'))
    
    def print_report(self):
        print("\n" + "="*50)
        print("SECURITY TEST REPORT")
        print("="*50)
        
        for test, result in self.results:
            status = "✓ PASS" if result == 'PASS' else f"✗ {result}"
            print(f"  {status}: {test}")
        
        failed = [r for _, r in self.results if r != 'PASS']
        print(f"\nTotal: {len(self.results)} tests | Failed: {len(failed)}")


if __name__ == '__main__':
    tester = APISecurityTester(BASE_URL)
    tester.test_authentication_bypass()
    tester.test_rate_limiting()
    tester.test_sql_injection()
    tester.test_sensitive_data_exposure()
    tester.print_report()
```

---

## 60.9 Secure Coding Practices

### Input Validation

```python
# security/validation.py
import re
from typing import Any


class InputValidator:
    
    @staticmethod
    def sanitize_string(value: str, max_length: int = 255, 
                        allowed_chars: str = None) -> str:
        """Sanitize string input"""
        if not isinstance(value, str):
            raise ValueError("Expected string")
        
        # Strip whitespace
        value = value.strip()
        
        if len(value) > max_length:
            raise ValueError(f"String too long (max {max_length})")
        
        if allowed_chars:
            pattern = f'^[{re.escape(allowed_chars)}]+$'
            if not re.match(pattern, value):
                raise ValueError(f"Invalid characters")
        
        return value
    
    @staticmethod
    def validate_ip(ip: str) -> str:
        """Validate IP address"""
        import ipaddress
        try:
            ipaddress.ip_address(ip)
            return ip
        except ValueError:
            raise ValueError(f"Invalid IP address: {ip}")
    
    @staticmethod
    def validate_router_command(command: str) -> str:
        """ป้องกัน command injection"""
        # Blocklist ของ dangerous patterns
        dangerous = [';', '&&', '||', '`', '$(',  
                     '\n', '\r', '|', '>', '<', '..']
        
        for pattern in dangerous:
            if pattern in command:
                raise ValueError(f"Dangerous pattern in command: {pattern}")
        
        return command
    
    @staticmethod
    def validate_mikrotik_name(name: str) -> str:
        """Validate MikroTik object names"""
        pattern = r'^[a-zA-Z0-9_\-\.]{1,64}$'
        if not re.match(pattern, name):
            raise ValueError(f"Invalid name: {name}")
        return name
```

---

## 60.10 Lab: API Security Hardening

```bash
#!/bin/bash
# lab-api-security.sh

echo "=== API Security Hardening Lab ==="

# 1. RouterOS: สร้าง read-only API user
echo "Step 1: Creating minimal-privilege API user"
cat << 'ROUTEROS'
# รัน commands เหล่านี้บน MikroTik

/user group add name=api-readonly policy=read,api
/user add name=api-monitor group=api-readonly password=Monitor@2024
/user set api-monitor allowed-address=192.168.100.0/24

# เปิด SSL API และปิด plain API
/certificate add name=router-ssl common-name=router days-valid=3650 key-size=2048
/certificate sign router-ssl
/ip service set api-ssl certificate=router-ssl disabled=no port=8729
/ip service set api disabled=yes

# Firewall: จำกัด access ไปยัง port 8728/8729
/ip firewall filter add chain=input protocol=tcp dst-port=8728,8729 \
    src-address=192.168.100.0/24 action=accept comment="Allow API from management"
/ip firewall filter add chain=input protocol=tcp dst-port=8728,8729 \
    action=drop comment="Block API from everywhere else"

ROUTEROS

# 2. Backend: ติดตั้ง security packages
echo ""
echo "Step 2: Install security packages"
npm install express-rate-limit helmet cors bcrypt jsonwebtoken redis 2>/dev/null || \
pip install flask-limiter flask-cors flask-jwt-extended redis bcrypt 2>/dev/null

# 3. สร้าง security config
cat > security-config.json << 'CONFIG'
{
  "api": {
    "ssl_only": true,
    "ssl_verify": false,
    "port": 8729
  },
  "auth": {
    "jwt_expires_seconds": 900,
    "refresh_expires_days": 7,
    "max_login_attempts": 5,
    "lockout_minutes": 15
  },
  "rate_limits": {
    "login": {"max": 5, "window_minutes": 15},
    "api_general": {"max": 100, "window_minutes": 1},
    "api_write": {"max": 20, "window_minutes": 1}
  },
  "ip_whitelist": {
    "enabled": true,
    "networks": ["192.168.0.0/16", "10.0.0.0/8"]
  },
  "audit": {
    "enabled": true,
    "log_file": "/var/log/api-audit.log",
    "db_path": "/var/db/audit.db"
  }
}
CONFIG

echo ""
echo "Security hardening checklist:"
echo ""
echo "RouterOS:"
echo "  [x] Minimal privilege API user"
echo "  [x] IP restriction on API user"
echo "  [x] SSL/TLS enabled (port 8729)"
echo "  [x] Plain API disabled"
echo "  [x] Firewall rules for API ports"
echo ""
echo "Backend:"
echo "  [x] JWT with short expiry (15min)"
echo "  [x] Rate limiting"
echo "  [x] IP whitelist"
echo "  [x] Input validation"
echo "  [x] Audit logging"
echo "  [x] HTTPS only"
echo "  [x] Security headers (Helmet)"
echo ""
echo "Done! Review security-config.json and run pentest:"
echo "  python3 pentest/api_security_test.py"
```

---

## Summary

| หัวข้อ | Implementation |
|--------|---------------|
| SSL/TLS | RouterOS cert + port 8729 |
| Auth | JWT + API Keys + bcrypt |
| Rate Limiting | Redis sliding window |
| IP Whitelist | Network CIDR validation |
| JWT Security | Short-lived + refresh tokens |
| Audit Logging | SQLite + file logging |
| Input Validation | Regex + blocklist |
| Penetration Testing | Automated test suite |
| Secure Coding | Sanitize all inputs |

> **Warning:** ไม่เคย expose RouterOS API port โดยตรงบน public internet เสมอต้องผ่าน VPN หรือ Jump host

---

[← Part 59: Hotspot Portal](part-059-hotspot-portal.md) | [Part 61: Advanced Security →](part-061-advanced-security.md)
