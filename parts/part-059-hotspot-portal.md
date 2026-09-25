# Part 59: Hotspot Portal

## บทนำ

Hotspot Portal คือหน้า login ที่ผู้ใช้งาน WiFi สาธารณะจะพบเมื่อเชื่อมต่อ บทนี้ครอบคลุมตั้งแต่ custom design, registration, payment integration จนถึง white-label solution

---

## 59.1 Custom Portal Design

### Hotspot Login Page Structure

```html
<!DOCTYPE html>
<html lang="th">
<head>
    <meta charset="UTF-8">
    <meta name="viewport" content="width=device-width, initial-scale=1.0">
    <title>$(link-hostname) - WiFi Login</title>
    <style>
        :root {
            --primary: #2563eb;
            --secondary: #1e40af;
            --accent: #3b82f6;
            --bg: #f0f9ff;
            --card-bg: #ffffff;
            --text: #1e293b;
            --text-light: #64748b;
        }
        
        * { box-sizing: border-box; margin: 0; padding: 0; }
        
        body {
            font-family: 'Sarabun', sans-serif;
            background: var(--bg);
            min-height: 100vh;
            display: flex;
            align-items: center;
            justify-content: center;
        }
        
        .portal-container {
            width: 100%;
            max-width: 420px;
            padding: 1rem;
        }
        
        .portal-card {
            background: var(--card-bg);
            border-radius: 16px;
            padding: 2rem;
            box-shadow: 0 20px 60px rgba(0,0,0,0.1);
        }
        
        .logo-area {
            text-align: center;
            margin-bottom: 1.5rem;
        }
        
        .logo-area img {
            max-width: 120px;
            height: auto;
        }
        
        .portal-title {
            font-size: 1.5rem;
            font-weight: 700;
            color: var(--text);
            text-align: center;
            margin-bottom: 0.5rem;
        }
        
        .portal-subtitle {
            color: var(--text-light);
            text-align: center;
            margin-bottom: 1.5rem;
        }
        
        .tabs {
            display: flex;
            border-bottom: 2px solid #e2e8f0;
            margin-bottom: 1.5rem;
        }
        
        .tab-btn {
            flex: 1;
            padding: 0.75rem;
            background: none;
            border: none;
            cursor: pointer;
            font-size: 0.9rem;
            color: var(--text-light);
            border-bottom: 2px solid transparent;
            margin-bottom: -2px;
            transition: all 0.2s;
        }
        
        .tab-btn.active {
            color: var(--primary);
            border-bottom-color: var(--primary);
            font-weight: 600;
        }
        
        .tab-content { display: none; }
        .tab-content.active { display: block; }
        
        .form-group { margin-bottom: 1rem; }
        
        .form-group label {
            display: block;
            margin-bottom: 0.4rem;
            font-weight: 600;
            color: var(--text);
        }
        
        .form-group input {
            width: 100%;
            padding: 0.75rem 1rem;
            border: 2px solid #e2e8f0;
            border-radius: 8px;
            font-size: 1rem;
            transition: border-color 0.2s;
        }
        
        .form-group input:focus {
            outline: none;
            border-color: var(--primary);
        }
        
        .btn-primary {
            width: 100%;
            padding: 0.875rem;
            background: var(--primary);
            color: white;
            border: none;
            border-radius: 8px;
            font-size: 1rem;
            font-weight: 600;
            cursor: pointer;
            transition: background 0.2s;
        }
        
        .btn-primary:hover { background: var(--secondary); }
        
        .divider {
            text-align: center;
            color: var(--text-light);
            margin: 1rem 0;
            position: relative;
        }
        
        .divider::before, .divider::after {
            content: '';
            position: absolute;
            top: 50%;
            width: 40%;
            height: 1px;
            background: #e2e8f0;
        }
        
        .divider::before { left: 0; }
        .divider::after { right: 0; }
        
        .social-buttons { display: flex; gap: 0.75rem; }
        
        .btn-social {
            flex: 1;
            padding: 0.75rem;
            border: 2px solid #e2e8f0;
            border-radius: 8px;
            background: white;
            cursor: pointer;
            font-size: 0.9rem;
            transition: border-color 0.2s;
        }
        
        .btn-social:hover { border-color: var(--primary); }
        
        .terms {
            text-align: center;
            font-size: 0.8rem;
            color: var(--text-light);
            margin-top: 1rem;
        }
        
        .terms a { color: var(--primary); }
        
        @media (max-width: 480px) {
            .portal-card { padding: 1.5rem; border-radius: 0; }
        }
    </style>
    <link href="https://fonts.googleapis.com/css2?family=Sarabun:wght@400;600;700&display=swap" rel="stylesheet">
</head>
<body>
    <div class="portal-container">
        <div class="portal-card">
            <div class="logo-area">
                <img src="$(link-logo)" alt="Logo" onerror="this.style.display='none'">
                <h1 class="portal-title">$(link-hostname)</h1>
                <p class="portal-subtitle">เชื่อมต่อ WiFi ฟรี</p>
            </div>
            
            <div class="tabs">
                <button class="tab-btn active" onclick="showTab('login')">เข้าสู่ระบบ</button>
                <button class="tab-btn" onclick="showTab('register')">สมัครสมาชิก</button>
                <button class="tab-btn" onclick="showTab('voucher')">Voucher</button>
            </div>
            
            <!-- Login Tab -->
            <div id="tab-login" class="tab-content active">
                <form name="login" action="$(link-login)" method="post">
                    <input type="hidden" name="dst" value="$(link-orig)">
                    <input type="hidden" name="popup" value="true">
                    
                    <div class="form-group">
                        <label>ชื่อผู้ใช้</label>
                        <input name="username" type="text" placeholder="Username" required autofocus>
                    </div>
                    
                    <div class="form-group">
                        <label>รหัสผ่าน</label>
                        <input name="password" type="password" placeholder="Password" required>
                    </div>
                    
                    <button type="submit" class="btn-primary">เข้าสู่ระบบ</button>
                    
                    <div class="divider">หรือ</div>
                    
                    <div class="social-buttons">
                        <button type="button" class="btn-social" onclick="facebookLogin()">
                            📘 Facebook
                        </button>
                        <button type="button" class="btn-social" onclick="googleLogin()">
                            🔍 Google
                        </button>
                    </div>
                </form>
            </div>
            
            <!-- Register Tab -->
            <div id="tab-register" class="tab-content">
                <form id="registerForm" onsubmit="handleRegister(event)">
                    <div class="form-group">
                        <label>ชื่อ-นามสกุล</label>
                        <input id="reg-name" type="text" placeholder="ชื่อ นามสกุล" required>
                    </div>
                    <div class="form-group">
                        <label>เบอร์โทรศัพท์</label>
                        <input id="reg-phone" type="tel" placeholder="0812345678" required>
                    </div>
                    <div class="form-group">
                        <label>อีเมล (ไม่บังคับ)</label>
                        <input id="reg-email" type="email" placeholder="email@example.com">
                    </div>
                    <button type="submit" class="btn-primary">ส่ง OTP ยืนยัน</button>
                </form>
            </div>
            
            <!-- Voucher Tab -->
            <div id="tab-voucher" class="tab-content">
                <form name="login" action="$(link-login)" method="post">
                    <input type="hidden" name="dst" value="$(link-orig)">
                    <div class="form-group">
                        <label>รหัส Voucher</label>
                        <input name="username" type="text" placeholder="XXXX-XXXX-XXXX" 
                               style="letter-spacing: 2px; text-align: center;" required>
                        <input name="password" type="hidden" value="voucher">
                    </div>
                    <button type="submit" class="btn-primary">ใช้งาน Voucher</button>
                </form>
            </div>
            
            <p class="terms">
                โดยการเชื่อมต่อ คุณยอมรับ
                <a href="/terms" target="_blank">เงื่อนไขการใช้งาน</a>
            </p>
        </div>
    </div>
    
    <script>
        function showTab(tab) {
            document.querySelectorAll('.tab-content').forEach(t => t.classList.remove('active'));
            document.querySelectorAll('.tab-btn').forEach(b => b.classList.remove('active'));
            
            document.getElementById('tab-' + tab).classList.add('active');
            event.target.classList.add('active');
        }
        
        async function handleRegister(e) {
            e.preventDefault();
            const phone = document.getElementById('reg-phone').value;
            
            const res = await fetch('/api/register', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({
                    name: document.getElementById('reg-name').value,
                    phone,
                    email: document.getElementById('reg-email').value,
                })
            });
            
            const data = await res.json();
            if (data.success) {
                showOTPDialog(phone);
            } else {
                alert(data.error || 'เกิดข้อผิดพลาด');
            }
        }
        
        function showOTPDialog(phone) {
            const otp = prompt(`กรุณากรอก OTP ที่ส่งไปยัง ${phone}`);
            if (otp) {
                verifyOTP(phone, otp);
            }
        }
        
        async function verifyOTP(phone, otp) {
            const res = await fetch('/api/verify-otp', {
                method: 'POST',
                headers: { 'Content-Type': 'application/json' },
                body: JSON.stringify({ phone, otp })
            });
            
            const data = await res.json();
            if (data.credentials) {
                // Auto-login with created credentials
                document.querySelector('[name="username"]').value = data.credentials.username;
                document.querySelector('[name="password"]').value = data.credentials.password;
                document.querySelector('form[name="login"]').submit();
            }
        }
        
        function facebookLogin() {
            window.location.href = '/api/auth/facebook?mac=$(mac)&dst=$(link-orig)';
        }
        
        function googleLogin() {
            window.location.href = '/api/auth/google?mac=$(mac)&dst=$(link-orig)';
        }
    </script>
</body>
</html>
```

---

## 59.2 Registration System

### Registration Backend

```python
#!/usr/bin/env python3
# registration/app.py

from flask import Flask, request, jsonify, session
import routeros_api
import random
import string
import time
import requests
import sqlite3
import hashlib
import os

app = Flask(__name__)
app.secret_key = os.environ.get('SECRET_KEY', 'dev-secret-key')

DB_PATH = '/var/db/hotspot_users.db'
MIKROTIK_HOST = os.environ.get('MIKROTIK_HOST', '192.168.1.1')
MIKROTIK_USER = os.environ.get('MIKROTIK_USER', 'admin')
MIKROTIK_PASS = os.environ.get('MIKROTIK_PASS', '')

# SMS Provider settings
SMS_API_URL = os.environ.get('SMS_API_URL', '')
SMS_API_KEY = os.environ.get('SMS_API_KEY', '')


def init_db():
    conn = sqlite3.connect(DB_PATH)
    conn.execute('''
        CREATE TABLE IF NOT EXISTS registered_users (
            id INTEGER PRIMARY KEY,
            phone VARCHAR(20) UNIQUE,
            name VARCHAR(200),
            email VARCHAR(200),
            mikrotik_user VARCHAR(100),
            registered_at DATETIME DEFAULT CURRENT_TIMESTAMP
        )
    ''')
    conn.execute('''
        CREATE TABLE IF NOT EXISTS otp_codes (
            phone VARCHAR(20),
            code VARCHAR(6),
            created_at INTEGER,
            verified INTEGER DEFAULT 0
        )
    ''')
    conn.commit()
    conn.close()


def send_sms_otp(phone: str, code: str) -> bool:
    """ส่ง OTP via SMS provider"""
    if not SMS_API_URL:
        # Demo mode - print to console
        print(f"OTP for {phone}: {code}")
        return True
    
    try:
        res = requests.post(SMS_API_URL, json={
            'apiKey': SMS_API_KEY,
            'to': phone,
            'message': f'รหัส OTP ของคุณคือ: {code}\nหมดอายุใน 5 นาที'
        }, timeout=10)
        return res.status_code == 200
    except Exception as e:
        print(f"SMS failed: {e}")
        return False


def create_mikrotik_user(username: str, password: str, profile: str = 'default') -> bool:
    """สร้าง hotspot user ใน MikroTik"""
    try:
        pool = routeros_api.RouterOsApiPool(
            MIKROTIK_HOST,
            username=MIKROTIK_USER,
            password=MIKROTIK_PASS,
        )
        api = pool.get_api()
        
        api.get_resource('/ip/hotspot/user').add(
            name=username,
            password=password,
            profile=profile,
            comment=f'Self-registered',
        )
        
        pool.disconnect()
        return True
    except Exception as e:
        print(f"MikroTik user creation failed: {e}")
        return False


@app.route('/api/register', methods=['POST'])
def register():
    data = request.json
    phone = data.get('phone', '').strip()
    name = data.get('name', '').strip()
    
    if not phone or not name:
        return jsonify({'error': 'Phone and name required'}), 400
    
    # สร้าง OTP
    otp = ''.join(random.choices(string.digits, k=6))
    
    conn = sqlite3.connect(DB_PATH)
    conn.execute('DELETE FROM otp_codes WHERE phone = ?', (phone,))
    conn.execute('INSERT INTO otp_codes (phone, code, created_at) VALUES (?, ?, ?)',
                 (phone, otp, int(time.time())))
    conn.execute('INSERT OR REPLACE INTO registered_users (phone, name, email) VALUES (?, ?, ?)',
                 (phone, name, data.get('email', '')))
    conn.commit()
    conn.close()
    
    sent = send_sms_otp(phone, otp)
    
    if not sent:
        return jsonify({'error': 'Failed to send OTP'}), 500
    
    return jsonify({'success': True, 'message': 'OTP sent'})


@app.route('/api/verify-otp', methods=['POST'])
def verify_otp():
    data = request.json
    phone = data.get('phone')
    code = data.get('otp')
    
    conn = sqlite3.connect(DB_PATH)
    otp_row = conn.execute(
        'SELECT code, created_at FROM otp_codes WHERE phone = ? AND verified = 0',
        (phone,)
    ).fetchone()
    
    if not otp_row:
        conn.close()
        return jsonify({'error': 'OTP not found'}), 400
    
    # ตรวจสอบอายุ OTP (5 นาที)
    if int(time.time()) - otp_row[1] > 300:
        conn.close()
        return jsonify({'error': 'OTP expired'}), 400
    
    if otp_row[0] != code:
        conn.close()
        return jsonify({'error': 'Invalid OTP'}), 400
    
    # สร้าง credentials
    username = 'u' + phone[-8:]
    password = ''.join(random.choices(string.ascii_letters + string.digits, k=8))
    
    if create_mikrotik_user(username, password):
        conn.execute('UPDATE otp_codes SET verified = 1 WHERE phone = ?', (phone,))
        conn.execute('UPDATE registered_users SET mikrotik_user = ? WHERE phone = ?',
                     (username, phone))
        conn.commit()
        conn.close()
        
        return jsonify({
            'success': True,
            'credentials': {'username': username, 'password': password}
        })
    
    conn.close()
    return jsonify({'error': 'Account creation failed'}), 500


if __name__ == '__main__':
    init_db()
    app.run(host='0.0.0.0', port=5000)
```

---

## 59.3 Payment Integration

### PromptPay QR Code

```python
# payment/promptpay.py
import qrcode
import io
import base64


def generate_promptpay_qr(mobile_or_national_id: str, amount: float) -> str:
    """สร้าง PromptPay QR Code"""
    
    def crc16(data: str) -> str:
        crc = 0xFFFF
        for char in data:
            crc ^= ord(char) << 8
            for _ in range(8):
                if crc & 0x8000:
                    crc = (crc << 1) ^ 0x1021
                else:
                    crc <<= 1
        return format(crc & 0xFFFF, '04X')
    
    def tlv(tag: str, value: str) -> str:
        return f"{tag}{len(value):02d}{value}"
    
    # PromptPay target
    is_mobile = mobile_or_national_id.startswith('0') and len(mobile_or_national_id) == 10
    
    if is_mobile:
        number = '0066' + mobile_or_national_id[1:]
        merchant_id = tlv('01', '13') + tlv('02', number)
    else:
        merchant_id = tlv('01', '02') + tlv('02', mobile_or_national_id.zfill(13))
    
    merchant_account = tlv('00', 'A000000677010111') + tlv('01', merchant_id if is_mobile else '')
    
    amount_str = f"{amount:.2f}"
    
    payload = (
        tlv('00', '01') +              # Payload format
        tlv('01', '12') +              # Point of initiation (dynamic)
        tlv('29', merchant_account) +  # Merchant account
        '5303764' +                    # Transaction currency (THB)
        tlv('54', amount_str) +        # Transaction amount
        '5802TH' +                     # Country
        '6304'                         # CRC placeholder
    )
    
    crc = crc16(payload)
    payload += crc
    
    qr = qrcode.QRCode(error_correction=qrcode.constants.ERROR_CORRECT_M)
    qr.add_data(payload)
    qr.make(fit=True)
    
    img = qr.make_image(fill_color='black', back_color='white')
    buffer = io.BytesIO()
    img.save(buffer, format='PNG')
    encoded = base64.b64encode(buffer.getvalue()).decode()
    
    return f"data:image/png;base64,{encoded}"
```

---

## 59.4 Voucher Generation

### Voucher System

```python
# voucher/generator.py
import routeros_api
import random
import string
import csv
import io


class VoucherGenerator:
    def __init__(self, host: str, username: str, password: str):
        self.pool = routeros_api.RouterOsApiPool(host, username=username, password=password)
        self.api = self.pool.get_api()
    
    def generate_code(self, length: int = 12) -> str:
        """สร้าง voucher code แบบ XXXX-XXXX-XXXX"""
        chars = string.ascii_uppercase + string.digits
        chars = chars.replace('0', '').replace('O', '').replace('I', '').replace('1', '')
        
        parts = []
        for _ in range(3):
            part = ''.join(random.choices(chars, k=4))
            parts.append(part)
        
        return '-'.join(parts)
    
    def create_vouchers(self, count: int, profile: str = '1hour', comment: str = '') -> list:
        """สร้าง vouchers จำนวนมาก"""
        users_resource = self.api.get_resource('/ip/hotspot/user')
        vouchers = []
        
        for i in range(count):
            code = self.generate_code()
            
            try:
                users_resource.add(
                    name=code,
                    password=code,
                    profile=profile,
                    comment=comment or f'Voucher batch #{i+1}',
                )
                vouchers.append({'code': code, 'profile': profile})
            except Exception as e:
                print(f"Failed to create voucher {i+1}: {e}")
        
        return vouchers
    
    def export_csv(self, vouchers: list) -> str:
        """Export vouchers เป็น CSV"""
        output = io.StringIO()
        writer = csv.DictWriter(output, fieldnames=['code', 'profile'])
        writer.writeheader()
        writer.writerows(vouchers)
        return output.getvalue()
    
    def list_unused_vouchers(self) -> list:
        """รายการ vouchers ที่ยังไม่ได้ใช้"""
        users = self.api.get_resource('/ip/hotspot/user').get()
        active = {u['name'] for u in self.api.get_resource('/ip/hotspot/active').get()}
        
        unused = []
        for u in users:
            name = u.get('name', '')
            # Voucher มีรูปแบบ XXXX-XXXX-XXXX
            if '-' in name and u.get('uptime', '0s') == '0s' and name not in active:
                unused.append({'code': name, 'profile': u.get('profile', '')})
        
        return unused
    
    def close(self):
        self.pool.disconnect()
```

---

## 59.5 Social Login

### OAuth2 Integration (Google)

```python
# social/google_auth.py
from flask import Flask, redirect, url_for, session, request
from authlib.integrations.flask_client import OAuth
import routeros_api
import os

app = Flask(__name__)
oauth = OAuth(app)

google = oauth.register(
    name='google',
    client_id=os.environ['GOOGLE_CLIENT_ID'],
    client_secret=os.environ['GOOGLE_CLIENT_SECRET'],
    server_metadata_url='https://accounts.google.com/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid email profile'},
)


@app.route('/api/auth/google')
def google_login():
    mac = request.args.get('mac', '')
    dst = request.args.get('dst', 'http://google.com')
    session['mac'] = mac
    session['dst'] = dst
    
    redirect_uri = url_for('google_callback', _external=True)
    return google.authorize_redirect(redirect_uri)


@app.route('/api/auth/google/callback')
def google_callback():
    token = google.authorize_access_token()
    user_info = token.get('userinfo')
    
    if not user_info or not user_info.get('email'):
        return redirect('/error?msg=auth_failed')
    
    email = user_info['email']
    name = user_info.get('name', email)
    
    # สร้าง/ค้นหา hotspot user
    username = 'g_' + email.replace('@', '_').replace('.', '_')[:20]
    password = ''.join(__import__('random').choices(__import__('string').ascii_letters, k=12))
    
    create_or_update_hotspot_user(username, password)
    
    # Redirect ไปยัง hotspot login
    mac = session.get('mac', '')
    dst = session.get('dst', 'http://google.com')
    
    login_url = f"http://192.168.88.1/login?username={username}&password={password}&dst={dst}"
    return redirect(login_url)


def create_or_update_hotspot_user(username: str, password: str):
    pool = routeros_api.RouterOsApiPool(
        os.environ.get('MIKROTIK_HOST', '192.168.1.1'),
        username=os.environ.get('MIKROTIK_USER', 'admin'),
        password=os.environ.get('MIKROTIK_PASS', ''),
    )
    api = pool.get_api()
    users = api.get_resource('/ip/hotspot/user')
    
    existing = users.get(name=username)
    if existing:
        users.set(id=existing[0]['id'], password=password)
    else:
        users.add(name=username, password=password, profile='social-login', comment='Google Auth')
    
    pool.disconnect()
```

---

## 59.6 Analytics Dashboard

### Analytics API

```python
# analytics/router.py
from flask import Blueprint, jsonify
import routeros_api
import sqlite3
from datetime import datetime, timedelta

analytics_bp = Blueprint('analytics', __name__)


@analytics_bp.route('/api/analytics/summary')
def get_summary():
    """สรุปสถิติ hotspot"""
    pool = routeros_api.RouterOsApiPool('192.168.1.1', username='admin', password='')
    api = pool.get_api()
    
    # Active users
    active_users = len(api.get_resource('/ip/hotspot/active').get())
    
    # Total users
    total_users = len(api.get_resource('/ip/hotspot/user').get())
    
    # Traffic stats
    interfaces = api.get_resource('/interface').get()
    wan_iface = next((i for i in interfaces if i.get('name') == 'ether1'), None)
    
    pool.disconnect()
    
    return jsonify({
        'active_users': active_users,
        'total_users': total_users,
        'wan_rx_bytes': int(wan_iface.get('rx-byte', 0)) if wan_iface else 0,
        'wan_tx_bytes': int(wan_iface.get('tx-byte', 0)) if wan_iface else 0,
        'timestamp': datetime.utcnow().isoformat(),
    })
```

---

## 59.7 Multi-location Support

### Multi-location Router

```python
# multilocation/manager.py

LOCATIONS = {
    'branch-01': {
        'name': 'สาขา 1 (สยาม)',
        'host': '192.168.1.1',
        'ssid': 'FreeWiFi-Siam',
    },
    'branch-02': {
        'name': 'สาขา 2 (อโศก)',
        'host': '192.168.2.1',
        'ssid': 'FreeWiFi-Asok',
    }
}


class MultiLocationManager:
    def __init__(self):
        self.connections = {}
    
    def get_api(self, location_id: str):
        if location_id not in LOCATIONS:
            raise ValueError(f"Unknown location: {location_id}")
        
        config = LOCATIONS[location_id]
        
        if location_id not in self.connections:
            pool = routeros_api.RouterOsApiPool(
                config['host'],
                username='admin',
                password='',
            )
            self.connections[location_id] = pool.get_api()
        
        return self.connections[location_id]
    
    def get_all_active_users(self) -> dict:
        result = {}
        for loc_id in LOCATIONS:
            try:
                api = self.get_api(loc_id)
                users = api.get_resource('/ip/hotspot/active').get()
                result[loc_id] = {
                    'name': LOCATIONS[loc_id]['name'],
                    'count': len(users),
                    'users': users,
                }
            except Exception as e:
                result[loc_id] = {'error': str(e), 'count': 0}
        return result
    
    def create_roaming_user(self, username: str, password: str, profile: str = 'default'):
        """สร้าง user บนทุก location สำหรับ roaming"""
        results = {}
        for loc_id in LOCATIONS:
            try:
                api = self.get_api(loc_id)
                api.get_resource('/ip/hotspot/user').add(
                    name=username,
                    password=password,
                    profile=profile,
                    comment='Roaming user',
                )
                results[loc_id] = 'success'
            except Exception as e:
                results[loc_id] = f'error: {e}'
        return results
```

---

## 59.8 White-label Configuration

### Brand Config

```python
# whitelabel/config.py
import json
import os

DEFAULT_BRAND = {
    'name': 'Free WiFi',
    'tagline': 'เชื่อมต่อได้ทันที',
    'primary_color': '#2563eb',
    'secondary_color': '#1e40af',
    'logo_url': '/img/logo.png',
    'terms_url': '/terms',
    'support_email': 'support@example.com',
    'languages': ['th', 'en'],
    'default_language': 'th',
    'plans_enabled': True,
    'social_login': ['facebook', 'google'],
    'sms_register': True,
}


def load_brand_config(domain: str = None) -> dict:
    config_path = f'/etc/hotspot-portal/brands/{domain}.json' if domain else None
    
    if config_path and os.path.exists(config_path):
        with open(config_path) as f:
            brand = json.load(f)
        return {**DEFAULT_BRAND, **brand}
    
    return DEFAULT_BRAND
```

---

## 59.9 Lab: Complete Hotspot Portal

```bash
#!/bin/bash
# setup_hotspot_portal.sh

echo "=== Hotspot Portal Complete Setup ==="

# 1. สร้างโครงสร้าง
mkdir -p hotspot-portal/{login,backend,payment,voucher,analytics}

# 2. Python dependencies
cat > hotspot-portal/requirements.txt << 'REQ'
flask>=2.3.0
routeros-api>=0.1.3
qrcode[pil]>=7.4.2
authlib>=1.2.0
requests>=2.31.0
REQ

# 3. Upload login.html ไปยัง MikroTik
# Files → Hotspot
# รอง: /ip hotspot profile set default html-directory=hotspot

echo ""
echo "Setup Instructions:"
echo "1. pip install -r hotspot-portal/requirements.txt"
echo "2. Copy login.html to MikroTik: /files/hotspot/"
echo "3. Configure in RouterOS:"
echo "   /ip hotspot profile set default"
echo "     html-directory=hotspot"
echo "     login-by=http-chap,mac,http-pap"
echo "4. Start backend: cd hotspot-portal && python app.py"
echo ""
echo "Default URL: http://<router-ip>/hotspot/"
```

---

## Summary

| Feature | Technology |
|---------|-----------|
| Custom Login Page | HTML/CSS/JS |
| SMS OTP Registration | Flask + SMS API |
| PromptPay Payment | QR Code Generation |
| Voucher Generation | RouterOS API |
| Social Login (Google) | OAuth2/OIDC |
| Analytics | Flask + RouterOS API |
| Multi-location | Connection Manager |
| White-label | JSON Brand Config |
| PWA | Service Worker |

---

[← Part 58: User Portal](part-058-user-portal.md) | [Part 60: API Security →](part-060-api-security.md)
