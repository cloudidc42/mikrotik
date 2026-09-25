# Part 51: MikroTik API Introduction

## บทนำ

MikroTik API เป็น binary protocol ที่ช่วยให้ external programs สามารถสื่อสารกับ RouterOS ได้โดยตรง ช่วยให้สร้าง custom applications สำหรับ monitoring, management, และ automation ได้

---

## 51.1 MikroTik API Protocol

### API Protocol Overview

```
RouterOS API ทำงานบน:
├── Port: 8728 (API, unencrypted)
├── Port: 8729 (API-SSL, encrypted)
├── Protocol: Binary/Text hybrid
└── Transport: TCP
```

### API Request-Response Flow

```
Client                    RouterOS
  |                          |
  |-- Connect TCP 8728 ─────>|
  |<─ Connection Open ───────|
  |                          |
  |-- Login Request ────────>|
  |<─ Login Response ────────|
  |                          |
  |-- Command ──────────────>|
  |<─ Response (data) ───────|
  |<─ !done ─────────────────|
  |                          |
  |-- Disconnect ───────────>|
```

---

## 51.2 API vs CLI Comparison

| คุณสมบัติ | API | CLI (SSH/Winbox) |
|-----------|-----|-----------------|
| ความเร็ว | เร็วมาก (binary) | ช้ากว่า (text) |
| Integration | ง่าย (library) | ยาก (screen scraping) |
| Automation | เหมาะมาก | พอใช้ |
| Learning Curve | สูงกว่า | ต่ำกว่า |
| Real-time Data | รองรับ | จำกัด |
| Error Handling | Structured | Text-based |

---

## 51.3 API Connection Methods

### การเชื่อมต่อ API

```python
# Python ตัวอย่างพื้นฐาน
import socket

class MikroTikAPI:
    def __init__(self, host, port=8728):
        self.host = host
        self.port = port
        self.socket = None
    
    def connect(self):
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.socket.connect((self.host, self.port))
        print(f"Connected to {self.host}:{self.port}")
    
    def disconnect(self):
        if self.socket:
            self.socket.close()
            self.socket = None
```

### SSL Connection

```python
import ssl

def connect_ssl(host, port=8729):
    context = ssl.create_default_context()
    context.check_hostname = False
    context.verify_mode = ssl.CERT_NONE
    
    raw_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    ssl_socket = context.wrap_socket(raw_socket, server_hostname=host)
    ssl_socket.connect((host, port))
    return ssl_socket
```

---

## 51.4 API Security

### Security Best Practices

| ด้าน | แนวทาง |
|------|--------|
| Port | ใช้ port 8729 (SSL) แทน 8728 |
| Credentials | สร้าง API user แยกต่างหาก |
| Permissions | จำกัด rights เฉพาะที่จำเป็น |
| IP Restriction | Whitelist IP ที่อนุญาต |
| Password | ใช้ strong password |

### สร้าง API User ที่มี Limited Rights

```routeros
# สร้าง group สำหรับ API access
/user group
add name="api-readonly" \
    policy=read,winbox,api \
    comment="Read-only API access"

add name="api-fullaccess" \
    policy=read,write,api,winbox \
    comment="Full API access"

# สร้าง user สำหรับ API
/user
add name="api-monitor" \
    group=api-readonly \
    password="SecureApiPass123!" \
    comment="API monitoring user"

add name="api-mgmt" \
    group=api-fullaccess \
    password="SuperSecurePass456!" \
    address=192.168.1.100 \
    comment="API management user - restricted to 192.168.1.100"
```

---

## 51.5 API Sentence Structure

### โครงสร้าง Sentence

```
API ส่งข้อมูลเป็น "sentences" (ชุดของ words)

Sentence = Word + Word + Word + ... + Empty Word

Word = Length-encoded string
Length = 1-4 bytes (depends on string length)

Example Sentence:
/ip/address/print   <- command word
=count-only=yes     <- attribute word
                    <- empty word (end of sentence)
```

### Word Encoding

```python
def encode_length(length):
    """Encode length ตาม MikroTik API spec"""
    if length < 0x80:
        return bytes([length])
    elif length < 0x4000:
        length |= 0x8000
        return bytes([(length >> 8) & 0xFF, length & 0xFF])
    elif length < 0x200000:
        length |= 0xC00000
        return bytes([(length >> 16) & 0xFF, 
                     (length >> 8) & 0xFF, 
                     length & 0xFF])
    elif length < 0x10000000:
        length |= 0xE0000000
        return bytes([(length >> 24) & 0xFF,
                     (length >> 16) & 0xFF,
                     (length >> 8) & 0xFF,
                     length & 0xFF])
    else:
        return bytes([0xF0,
                     (length >> 24) & 0xFF,
                     (length >> 16) & 0xFF,
                     (length >> 8) & 0xFF,
                     length & 0xFF])

def encode_word(word):
    """Encode word สำหรับส่งผ่าน API"""
    encoded = word.encode('utf-8')
    return encode_length(len(encoded)) + encoded

def encode_sentence(words):
    """Encode sentence (list of words)"""
    result = b''
    for word in words:
        result += encode_word(word)
    result += b'\x00'  # Empty word
    return result
```

---

## 51.6 Words and Attributes

### ประเภทของ Words

| ประเภท | Format | ตัวอย่าง |
|--------|--------|---------|
| Command | `/path/command` | `/ip/address/print` |
| Attribute | `=name=value` | `=interface=ether1` |
| Query | `?name=value` | `?interface=ether1` |
| API Attribute | `.tag=N` | `.tag=1` |
| Reply word | `!done`, `!re`, `!trap`, `!fatal` | `!done` |

### ตัวอย่าง Sentences

```
# Print IP addresses:
/ip/address/print
                    <- empty word (end)

# Add IP address:
/ip/address/add
=address=192.168.1.100/24
=interface=ether1
                    <- empty word (end)

# Print with filter:
/ip/address/print
?interface=ether1
                    <- empty word (end)

# Response sentence:
!re
=.id=*1
=address=192.168.1.100/24
=interface=ether1
=network=192.168.1.0
=broadcast=192.168.1.255
=invalid=false
=dynamic=false
                    <- empty word (end)
```

---

## 51.7 Reply Handling

### Reply Types

```python
def parse_reply(socket_obj):
    """Parse reply จาก RouterOS API"""
    replies = []
    current_reply = {}
    
    while True:
        words = read_sentence(socket_obj)
        
        if not words:
            continue
        
        reply_type = words[0]
        
        if reply_type == '!done':
            # Command completed
            break
        elif reply_type == '!re':
            # Data reply
            attrs = {}
            for word in words[1:]:
                if word.startswith('='):
                    parts = word[1:].split('=', 1)
                    if len(parts) == 2:
                        attrs[parts[0]] = parts[1]
            replies.append(attrs)
        elif reply_type == '!trap':
            # Error
            error_msg = {}
            for word in words[1:]:
                if word.startswith('='):
                    parts = word[1:].split('=', 1)
                    if len(parts) == 2:
                        error_msg[parts[0]] = parts[1]
            raise Exception(f"API Error: {error_msg}")
        elif reply_type == '!fatal':
            raise Exception("Fatal error - connection will close")
    
    return replies
```

---

## 51.8 Error Handling

### Error Types

```python
class RouterOSAPIError(Exception):
    """Base exception for RouterOS API errors"""
    pass

class APIConnectionError(RouterOSAPIError):
    """Connection failed"""
    pass

class APIAuthenticationError(RouterOSAPIError):
    """Authentication failed"""
    pass

class APICommandError(RouterOSAPIError):
    """Command execution failed"""
    def __init__(self, message, category=None):
        super().__init__(message)
        self.category = category

class MikroTikAPI:
    def safe_execute(self, command, *args, **kwargs):
        """Execute command with error handling"""
        try:
            result = self.execute(command, *args, **kwargs)
            return result
        except APICommandError as e:
            print(f"Command error: {e}")
            if e.category == 'no_such_item':
                return []
            raise
        except APIConnectionError as e:
            print(f"Connection lost: {e}")
            # Try to reconnect
            self.reconnect()
            raise
        except Exception as e:
            print(f"Unexpected error: {e}")
            raise
```

---

## 51.9 API Libraries Overview

### Popular Libraries

| Library | Language | GitHub | Features |
|---------|----------|--------|----------|
| routeros-api | Python | jonutz/routeros-api | Full-featured |
| librouteros | Python | hellt/librouteros | Modern async |
| node-routeros | Node.js | aluisiora/routeros-api | Promise-based |
| routeros-client | PHP | EvilFreelancer/routeros-api | OOP |
| go-routeros | Go | go-routeros/routeros | Native Go |

### Python routeros-api Installation

```bash
# ติดตั้ง library
pip install routeros-api

# ทดสอบ connection
python3 -c "
import routeros_api

connection = routeros_api.RouterOsApiPool(
    '192.168.1.1',
    username='admin',
    password='',
    port=8728
)
api = connection.get_api()

# List IP addresses
addresses = api.get_resource('/ip/address')
for addr in addresses.get():
    print(addr)

connection.disconnect()
"
```

---

## 51.10 Lab: Raw API Connection

### Lab: เชื่อมต่อ API แบบ Raw (Python)

```python
#!/usr/bin/env python3
"""
MikroTik Raw API Connection Lab
ไฟล์: mikrotik_raw_api.py
"""

import socket
import hashlib

class MikroTikRawAPI:
    def __init__(self, host, port=8728):
        self.host = host
        self.port = port
        self.socket = None
    
    def connect(self):
        """เชื่อมต่อไปยัง RouterOS API"""
        self.socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.socket.settimeout(10)
        self.socket.connect((self.host, self.port))
        print(f"✓ Connected to {self.host}:{self.port}")
    
    def disconnect(self):
        """ปิดการเชื่อมต่อ"""
        if self.socket:
            self.socket.close()
            self.socket = None
            print("✓ Disconnected")
    
    def _encode_length(self, length):
        """Encode length ตาม API spec"""
        if length < 0x80:
            return bytes([length])
        elif length < 0x4000:
            length |= 0x8000
            return bytes([(length >> 8) & 0xFF, length & 0xFF])
        else:
            length |= 0xC00000
            return bytes([(length >> 16) & 0xFF,
                         (length >> 8) & 0xFF,
                         length & 0xFF])
    
    def _encode_word(self, word):
        """Encode word"""
        if isinstance(word, str):
            word = word.encode('utf-8')
        return self._encode_length(len(word)) + word
    
    def _decode_length(self, data, pos):
        """Decode length จาก data"""
        first = data[pos]
        if first < 0x80:
            return first, pos + 1
        elif first < 0xC0:
            second = data[pos + 1]
            return ((first & 0x3F) << 8) | second, pos + 2
        else:
            second = data[pos + 1]
            third = data[pos + 2]
            return ((first & 0x1F) << 16) | (second << 8) | third, pos + 3
    
    def send_sentence(self, words):
        """ส่ง sentence ไปยัง router"""
        sentence = b''
        for word in words:
            sentence += self._encode_word(word)
        sentence += b'\x00'  # Empty word
        self.socket.send(sentence)
    
    def read_word(self):
        """อ่าน word หนึ่งคำจาก socket"""
        # อ่าน length
        first_byte = self.socket.recv(1)
        if not first_byte:
            return None
        
        first = first_byte[0]
        if first < 0x80:
            length = first
        elif first < 0xC0:
            second = self.socket.recv(1)[0]
            length = ((first & 0x3F) << 8) | second
        else:
            rest = self.socket.recv(2)
            length = ((first & 0x1F) << 16) | (rest[0] << 8) | rest[1]
        
        if length == 0:
            return ''
        
        # อ่าน data
        data = b''
        while len(data) < length:
            chunk = self.socket.recv(length - len(data))
            if not chunk:
                break
            data += chunk
        
        return data.decode('utf-8', errors='replace')
    
    def read_sentence(self):
        """อ่าน sentence (list of words)"""
        words = []
        while True:
            word = self.read_word()
            if word == '' or word is None:
                break
            words.append(word)
        return words
    
    def login(self, username, password):
        """Login ไปยัง RouterOS"""
        # ส่ง login command
        self.send_sentence(['/login', f'=name={username}', f'=password={password}'])
        
        # อ่าน response
        response = self.read_sentence()
        
        if response and response[0] == '!done':
            print(f"✓ Logged in as {username}")
            return True
        elif response and response[0] == '!trap':
            error = 'Unknown error'
            for word in response:
                if word.startswith('=message='):
                    error = word[9:]
            raise Exception(f"Login failed: {error}")
        
        return False
    
    def execute(self, command, params=None, queries=None):
        """Execute command และ return results"""
        words = [command]
        
        if params:
            for key, value in params.items():
                words.append(f'={key}={value}')
        
        if queries:
            for key, value in queries.items():
                words.append(f'?{key}={value}')
        
        self.send_sentence(words)
        
        results = []
        while True:
            sentence = self.read_sentence()
            
            if not sentence:
                break
            
            reply_type = sentence[0]
            
            if reply_type == '!done':
                break
            elif reply_type == '!re':
                item = {}
                for word in sentence[1:]:
                    if word.startswith('='):
                        parts = word[1:].split('=', 1)
                        if len(parts) == 2:
                            item[parts[0]] = parts[1]
                results.append(item)
            elif reply_type == '!trap':
                error = 'Command error'
                for word in sentence:
                    if word.startswith('=message='):
                        error = word[9:]
                raise Exception(f"API Error: {error}")
        
        return results


def main():
    """ทดสอบ Raw API Connection"""
    api = MikroTikRawAPI('192.168.1.1', port=8728)
    
    try:
        # เชื่อมต่อ
        api.connect()
        
        # Login
        api.login('admin', '')
        
        # 1. ดู IP addresses
        print("\n=== IP Addresses ===")
        addresses = api.execute('/ip/address/print')
        for addr in addresses:
            print(f"  {addr.get('address', 'N/A')} on {addr.get('interface', 'N/A')}")
        
        # 2. ดู interfaces
        print("\n=== Interfaces ===")
        interfaces = api.execute('/interface/print')
        for iface in interfaces:
            status = "up" if iface.get('running') == 'true' else "down"
            print(f"  {iface.get('name', 'N/A')} ({iface.get('type', 'N/A')}) - {status}")
        
        # 3. ดู routing table
        print("\n=== Routes ===")
        routes = api.execute('/ip/route/print')
        for route in routes:
            print(f"  {route.get('dst-address', 'N/A')} via {route.get('gateway', 'N/A')}")
        
        print("\n✓ Lab complete!")
        
    except Exception as e:
        print(f"✗ Error: {e}")
    finally:
        api.disconnect()


if __name__ == '__main__':
    main()
```

### การรัน Lab

```bash
# สร้างไฟล์
nano mikrotik_raw_api.py
# วาง code ด้านบน

# รันทดสอบ (ปรับ IP ตามจริง)
python3 mikrotik_raw_api.py

# Expected output:
# ✓ Connected to 192.168.1.1:8728
# ✓ Logged in as admin
#
# === IP Addresses ===
#   192.168.1.1/24 on ether1
#
# === Interfaces ===
#   ether1 (ether) - up
#
# === Routes ===
#   0.0.0.0/0 via 203.0.113.1
#
# ✓ Lab complete!
# ✓ Disconnected
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| API Protocol | Binary protocol, TCP port 8728/8729 |
| API vs CLI | เปรียบเทียบข้อดีข้อเสีย |
| Connection | TCP connect, SSL support |
| Security | User creation, IP restriction |
| Sentence Structure | Words, attributes, queries |
| Reply Handling | !done, !re, !trap |
| Error Handling | Exception types |
| Libraries | Python, Node.js, PHP options |
| Raw API Lab | Python implementation |

---

[← Part 50: Captive Portal](part-050-captive-portal.md) | [Part 52: API PHP →](part-052-api-php.md)
