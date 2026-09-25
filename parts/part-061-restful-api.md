# Part 61: REST API Design สำหรับ MikroTik Management

## สารบัญ
1. [REST API Design Principles](#rest-api-design)
2. [FastAPI Setup](#fastapi-setup)
3. [Express.js Alternative](#expressjs)
4. [Endpoint Design](#endpoints)
5. [Authentication Middleware](#authentication)
6. [Request Validation](#validation)
7. [Response Formatting](#response)
8. [Swagger Documentation](#swagger)
9. [Testing with Postman](#postman)
10. [Deployment](#deployment)
11. [Lab: Complete REST API](#lab)

---

## 1. REST API Design Principles {#rest-api-design}

REST (Representational State Transfer) API สำหรับ MikroTik management ต้องการการออกแบบที่ดีเพื่อให้ใช้งานได้จริงในระดับ production

### หลักการสำคัญ

| หลักการ | คำอธิบาย | ตัวอย่าง |
|---------|----------|---------|
| Stateless | ทุก request มีข้อมูลครบ | JWT token ในทุก request |
| Resource-based | URL แทน resource ไม่ใช่ action | `/routers/{id}/interfaces` |
| HTTP Methods | ใช้ GET/POST/PUT/DELETE ตาม semantics | GET = อ่าน, POST = สร้าง |
| Versioning | version ใน URL | `/api/v1/routers` |
| Consistent format | format เดียวกันทุก endpoint | JSON response ทั้งหมด |

### Architecture Overview

```
┌─────────────────────────────────────────────────────────┐
│                    Client Applications                    │
│              (Web UI, Mobile, CLI tools)                 │
└─────────────────────┬───────────────────────────────────┘
                      │ HTTPS
┌─────────────────────▼───────────────────────────────────┐
│                   API Gateway / Nginx                     │
│            (Rate Limiting, SSL Termination)              │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│                   FastAPI Application                     │
│  ┌─────────────┐ ┌──────────────┐ ┌──────────────────┐  │
│  │   Routers   │ │  Interfaces  │ │   Firewall Rules │  │
│  │  Endpoints  │ │  Endpoints   │ │   Endpoints      │  │
│  └─────────────┘ └──────────────┘ └──────────────────┘  │
└─────────────────────┬───────────────────────────────────┘
                      │
┌─────────────────────▼───────────────────────────────────┐
│              RouterOS API Library (librouteros)           │
└─────────────────────┬───────────────────────────────────┘
                      │ RouterOS API (port 8728/8729)
┌─────────────────────▼───────────────────────────────────┐
│                   MikroTik Routers                        │
│         (Router1, Router2, ... RouterN)                  │
└─────────────────────────────────────────────────────────┘
```

---

## 2. FastAPI Setup {#fastapi-setup}

### การติดตั้ง Environment

```bash
# สร้าง virtual environment
python3 -m venv venv
source venv/bin/activate

# ติดตั้ง dependencies
pip install fastapi uvicorn librouteros python-jose[cryptography] \
            passlib[bcrypt] python-multipart redis pydantic-settings \
            httpx pytest pytest-asyncio

# สร้างไฟล์ requirements.txt
pip freeze > requirements.txt
```

### โครงสร้างโปรเจค

```
mikrotik-api/
├── app/
│   ├── __init__.py
│   ├── main.py                 # FastAPI application entry point
│   ├── config.py               # Configuration settings
│   ├── dependencies.py         # Shared dependencies
│   ├── database.py             # Database connection
│   ├── models/
│   │   ├── __init__.py
│   │   ├── router.py           # Router models
│   │   ├── interface.py        # Interface models
│   │   ├── user.py             # User models
│   │   └── firewall.py         # Firewall models
│   ├── routers/
│   │   ├── __init__.py
│   │   ├── auth.py             # Authentication endpoints
│   │   ├── routers_api.py      # Router management
│   │   ├── interfaces.py       # Interface management
│   │   ├── firewall.py         # Firewall management
│   │   ├── dhcp.py             # DHCP management
│   │   └── monitoring.py       # Monitoring endpoints
│   ├── services/
│   │   ├── __init__.py
│   │   ├── mikrotik.py         # MikroTik connection service
│   │   ├── auth_service.py     # Authentication logic
│   │   └── cache.py            # Caching service
│   └── middleware/
│       ├── __init__.py
│       ├── auth.py             # JWT middleware
│       ├── rate_limit.py       # Rate limiting
│       └── logging.py          # Request logging
├── tests/
│   ├── __init__.py
│   ├── test_auth.py
│   ├── test_routers.py
│   └── test_interfaces.py
├── docker/
│   ├── Dockerfile
│   └── docker-compose.yml
├── .env.example
├── requirements.txt
└── README.md
```

### Main Application File

```python
# app/main.py
from fastapi import FastAPI, Request
from fastapi.middleware.cors import CORSMiddleware
from fastapi.middleware.trustedhost import TrustedHostMiddleware
from fastapi.responses import JSONResponse
import time
import logging
from contextlib import asynccontextmanager

from app.config import settings
from app.routers import auth, routers_api, interfaces, firewall, dhcp, monitoring
from app.middleware.logging import LoggingMiddleware
from app.middleware.rate_limit import RateLimitMiddleware

# Configure logging
logging.basicConfig(
    level=logging.INFO,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)
logger = logging.getLogger(__name__)


@asynccontextmanager
async def lifespan(app: FastAPI):
    """Application lifespan events"""
    logger.info("Starting MikroTik Management API...")
    # Startup: initialize connections, cache, etc.
    yield
    # Shutdown: cleanup connections
    logger.info("Shutting down MikroTik Management API...")


# สร้าง FastAPI application
app = FastAPI(
    title="MikroTik Management API",
    description="""
    ## REST API สำหรับจัดการ MikroTik Routers
    
    ### Features
    - **Router Management**: จัดการ router หลายตัว
    - **Interface Control**: configure interfaces
    - **Firewall Rules**: manage firewall
    - **DHCP Management**: IP address management
    - **Real-time Monitoring**: ดู metrics แบบ real-time
    
    ### Authentication
    ใช้ JWT Bearer token สำหรับ authentication
    """,
    version="1.0.0",
    openapi_tags=[
        {"name": "auth", "description": "Authentication operations"},
        {"name": "routers", "description": "Router management"},
        {"name": "interfaces", "description": "Interface management"},
        {"name": "firewall", "description": "Firewall rule management"},
        {"name": "monitoring", "description": "Monitoring and metrics"},
    ],
    lifespan=lifespan
)

# CORS middleware
app.add_middleware(
    CORSMiddleware,
    allow_origins=settings.ALLOWED_ORIGINS,
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)

# Custom middleware
app.add_middleware(LoggingMiddleware)
app.add_middleware(RateLimitMiddleware, max_requests=100, window_seconds=60)

# Include routers
app.include_router(auth.router, prefix="/api/v1/auth", tags=["auth"])
app.include_router(routers_api.router, prefix="/api/v1/routers", tags=["routers"])
app.include_router(interfaces.router, prefix="/api/v1/routers/{router_id}/interfaces", tags=["interfaces"])
app.include_router(firewall.router, prefix="/api/v1/routers/{router_id}/firewall", tags=["firewall"])
app.include_router(dhcp.router, prefix="/api/v1/routers/{router_id}/dhcp", tags=["dhcp"])
app.include_router(monitoring.router, prefix="/api/v1/monitoring", tags=["monitoring"])


@app.get("/", include_in_schema=False)
async def root():
    return {
        "message": "MikroTik Management API",
        "version": "1.0.0",
        "docs": "/docs",
        "redoc": "/redoc"
    }


@app.get("/health")
async def health_check():
    """Health check endpoint"""
    return {
        "status": "healthy",
        "timestamp": time.time(),
        "version": "1.0.0"
    }


@app.exception_handler(Exception)
async def global_exception_handler(request: Request, exc: Exception):
    """Global exception handler"""
    logger.error(f"Unhandled exception: {exc}", exc_info=True)
    return JSONResponse(
        status_code=500,
        content={
            "error": "internal_server_error",
            "message": "An unexpected error occurred",
            "path": str(request.url)
        }
    )
```

### Configuration Settings

```python
# app/config.py
from pydantic_settings import BaseSettings
from typing import List, Optional
import os


class Settings(BaseSettings):
    # API Settings
    APP_NAME: str = "MikroTik Management API"
    APP_VERSION: str = "1.0.0"
    DEBUG: bool = False
    
    # Security
    SECRET_KEY: str = "your-secret-key-change-in-production"
    ALGORITHM: str = "HS256"
    ACCESS_TOKEN_EXPIRE_MINUTES: int = 30
    REFRESH_TOKEN_EXPIRE_DAYS: int = 7
    
    # Database
    DATABASE_URL: str = "postgresql://user:password@localhost/mikrotik_db"
    
    # Redis
    REDIS_URL: str = "redis://localhost:6379"
    CACHE_TTL: int = 300  # 5 minutes
    
    # CORS
    ALLOWED_ORIGINS: List[str] = [
        "http://localhost:3000",
        "http://localhost:8080",
        "https://admin.yourdomain.com"
    ]
    
    # MikroTik defaults
    MIKROTIK_DEFAULT_PORT: int = 8728
    MIKROTIK_SSL_PORT: int = 8729
    MIKROTIK_TIMEOUT: int = 10
    
    # Rate limiting
    RATE_LIMIT_REQUESTS: int = 100
    RATE_LIMIT_WINDOW: int = 60
    
    class Config:
        env_file = ".env"
        case_sensitive = True


settings = Settings()
```

---

## 3. MikroTik Service Layer

```python
# app/services/mikrotik.py
import librouteros
from librouteros import connect
from librouteros.exceptions import TrapError, ConnectionClosed
import asyncio
from typing import Dict, Any, List, Optional
import logging
from contextlib import contextmanager

logger = logging.getLogger(__name__)


class MikroTikConnection:
    """Class สำหรับจัดการ connection กับ MikroTik"""
    
    def __init__(self, host: str, username: str, password: str, 
                 port: int = 8728, use_ssl: bool = False):
        self.host = host
        self.username = username
        self.password = password
        self.port = port
        self.use_ssl = use_ssl
        self._connection = None
    
    def connect(self):
        """สร้าง connection กับ MikroTik"""
        try:
            if self.use_ssl:
                import ssl
                ctx = ssl.create_default_context()
                ctx.check_hostname = False
                ctx.verify_mode = ssl.CERT_NONE
                self._connection = connect(
                    host=self.host,
                    username=self.username,
                    password=self.password,
                    port=self.port,
                    ssl_wrapper=ctx.wrap_socket
                )
            else:
                self._connection = connect(
                    host=self.host,
                    username=self.username,
                    password=self.password,
                    port=self.port
                )
            logger.info(f"Connected to MikroTik: {self.host}")
            return self._connection
        except Exception as e:
            logger.error(f"Failed to connect to {self.host}: {e}")
            raise
    
    def disconnect(self):
        """ปิด connection"""
        if self._connection:
            try:
                self._connection.close()
            except:
                pass
            self._connection = None
    
    def __enter__(self):
        self.connect()
        return self
    
    def __exit__(self, exc_type, exc_val, exc_tb):
        self.disconnect()
    
    def get_interfaces(self) -> List[Dict]:
        """ดึงรายการ interfaces"""
        if not self._connection:
            raise RuntimeError("Not connected")
        
        result = []
        for item in self._connection('/interface/print'):
            result.append(dict(item))
        return result
    
    def get_interface_stats(self) -> List[Dict]:
        """ดึง traffic statistics"""
        if not self._connection:
            raise RuntimeError("Not connected")
        
        result = []
        for item in self._connection('/interface/print', 
                                      **{'=.proplist': 'name,rx-byte,tx-byte,rx-packet,tx-packet,rx-error,tx-error'}):
            result.append(dict(item))
        return result
    
    def get_ip_addresses(self) -> List[Dict]:
        """ดึง IP addresses ทั้งหมด"""
        result = []
        for item in self._connection('/ip/address/print'):
            result.append(dict(item))
        return result
    
    def get_routing_table(self) -> List[Dict]:
        """ดึง routing table"""
        result = []
        for item in self._connection('/ip/route/print'):
            result.append(dict(item))
        return result
    
    def get_firewall_rules(self, chain: str = None) -> List[Dict]:
        """ดึง firewall rules"""
        result = []
        if chain:
            for item in self._connection('/ip/firewall/filter/print', 
                                          **{'?chain': chain}):
                result.append(dict(item))
        else:
            for item in self._connection('/ip/firewall/filter/print'):
                result.append(dict(item))
        return result
    
    def add_firewall_rule(self, rule: Dict) -> Dict:
        """เพิ่ม firewall rule"""
        params = {f'={k}': v for k, v in rule.items()}
        result = self._connection('/ip/firewall/filter/add', **params)
        return {'id': result}
    
    def get_dhcp_leases(self) -> List[Dict]:
        """ดึง DHCP leases"""
        result = []
        for item in self._connection('/ip/dhcp-server/lease/print'):
            result.append(dict(item))
        return result
    
    def get_system_resources(self) -> Dict:
        """ดึง system resources"""
        result = {}
        for item in self._connection('/system/resource/print'):
            result = dict(item)
            break
        return result
    
    def run_command(self, command: str, **kwargs) -> List[Dict]:
        """รัน arbitrary command"""
        result = []
        for item in self._connection(command, **kwargs):
            result.append(dict(item))
        return result


class RouterManager:
    """Manager สำหรับจัดการ routers หลายตัว"""
    
    def __init__(self):
        self._routers: Dict[str, Dict] = {}
    
    def add_router(self, router_id: str, host: str, username: str, 
                   password: str, port: int = 8728, use_ssl: bool = False):
        """เพิ่ม router เข้า manager"""
        self._routers[router_id] = {
            'host': host,
            'username': username,
            'password': password,
            'port': port,
            'use_ssl': use_ssl
        }
    
    @contextmanager
    def get_connection(self, router_id: str):
        """Get connection สำหรับ router ที่ระบุ"""
        if router_id not in self._routers:
            raise ValueError(f"Router {router_id} not found")
        
        config = self._routers[router_id]
        conn = MikroTikConnection(**config)
        try:
            conn.connect()
            yield conn
        finally:
            conn.disconnect()
    
    def test_connection(self, router_id: str) -> bool:
        """ทดสอบ connection กับ router"""
        try:
            with self.get_connection(router_id) as conn:
                resources = conn.get_system_resources()
                return bool(resources)
        except Exception as e:
            logger.error(f"Connection test failed for {router_id}: {e}")
            return False


# Singleton instance
router_manager = RouterManager()
```

---

## 4. Endpoints Design {#endpoints}

### Router Management Endpoints

```python
# app/routers/routers_api.py
from fastapi import APIRouter, Depends, HTTPException, status, Query
from typing import List, Optional
from pydantic import BaseModel, IPvAnyAddress
import logging

from app.services.mikrotik import router_manager
from app.dependencies import get_current_user
from app.models.router import RouterCreate, RouterResponse, RouterUpdate

logger = logging.getLogger(__name__)
router = APIRouter()


class RouterCreate(BaseModel):
    name: str
    host: str
    username: str
    password: str
    port: int = 8728
    use_ssl: bool = False
    description: Optional[str] = None
    location: Optional[str] = None
    
    class Config:
        schema_extra = {
            "example": {
                "name": "Core Router 1",
                "host": "192.168.1.1",
                "username": "admin",
                "password": "secure_password",
                "port": 8728,
                "use_ssl": False,
                "description": "Main core router",
                "location": "Server Room A"
            }
        }


class RouterResponse(BaseModel):
    id: str
    name: str
    host: str
    status: str
    description: Optional[str]
    location: Optional[str]
    created_at: str
    updated_at: str


@router.get("/", response_model=List[RouterResponse])
async def list_routers(
    skip: int = Query(0, ge=0, description="จำนวน records ที่ข้าม"),
    limit: int = Query(10, ge=1, le=100, description="จำนวน records สูงสุด"),
    status: Optional[str] = Query(None, description="กรองตาม status"),
    location: Optional[str] = Query(None, description="กรองตาม location"),
    current_user = Depends(get_current_user)
):
    """
    ดึงรายการ routers ทั้งหมด
    
    - **skip**: จำนวน records ที่ข้าม (สำหรับ pagination)
    - **limit**: จำนวน records สูงสุดที่ return
    - **status**: กรองตาม status (online/offline)
    - **location**: กรองตาม location
    """
    # ดึงจาก database
    from app.database import get_db
    # ... database query logic
    pass


@router.post("/", response_model=RouterResponse, status_code=status.HTTP_201_CREATED)
async def create_router(
    router_data: RouterCreate,
    current_user = Depends(get_current_user)
):
    """
    เพิ่ม router ใหม่เข้าระบบ
    
    ต้องการ role: **admin** หรือ **network_admin**
    """
    # ทดสอบ connection ก่อนบันทึก
    test_conn = MikroTikConnection(
        host=router_data.host,
        username=router_data.username,
        password=router_data.password,
        port=router_data.port,
        use_ssl=router_data.use_ssl
    )
    
    try:
        test_conn.connect()
        resources = test_conn.get_system_resources()
        test_conn.disconnect()
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_400_BAD_REQUEST,
            detail=f"Cannot connect to router: {str(e)}"
        )
    
    # บันทึกลง database
    # ...
    pass


@router.get("/{router_id}", response_model=RouterResponse)
async def get_router(
    router_id: str,
    current_user = Depends(get_current_user)
):
    """ดึงข้อมูล router ตาม ID"""
    pass


@router.put("/{router_id}", response_model=RouterResponse)
async def update_router(
    router_id: str,
    router_data: RouterCreate,
    current_user = Depends(get_current_user)
):
    """อัพเดต router configuration"""
    pass


@router.delete("/{router_id}", status_code=status.HTTP_204_NO_CONTENT)
async def delete_router(
    router_id: str,
    current_user = Depends(get_current_user)
):
    """ลบ router ออกจากระบบ"""
    pass


@router.get("/{router_id}/status")
async def get_router_status(
    router_id: str,
    current_user = Depends(get_current_user)
):
    """ดึงสถานะปัจจุบันของ router"""
    try:
        with router_manager.get_connection(router_id) as conn:
            resources = conn.get_system_resources()
            return {
                "router_id": router_id,
                "status": "online",
                "resources": {
                    "cpu_load": resources.get('cpu-load', '0'),
                    "memory_used": resources.get('total-memory', '0'),
                    "uptime": resources.get('uptime', 'unknown'),
                    "version": resources.get('version', 'unknown'),
                    "board_name": resources.get('board-name', 'unknown')
                }
            }
    except Exception as e:
        return {
            "router_id": router_id,
            "status": "offline",
            "error": str(e)
        }


@router.post("/{router_id}/backup")
async def create_router_backup(
    router_id: str,
    backup_name: Optional[str] = None,
    current_user = Depends(get_current_user)
):
    """สร้าง backup ของ router configuration"""
    try:
        with router_manager.get_connection(router_id) as conn:
            if not backup_name:
                from datetime import datetime
                backup_name = f"backup_{datetime.now().strftime('%Y%m%d_%H%M%S')}"
            
            conn.run_command('/system/backup/save', **{'=name': backup_name})
            
            return {
                "success": True,
                "backup_name": backup_name,
                "message": f"Backup created: {backup_name}"
            }
    except Exception as e:
        raise HTTPException(
            status_code=status.HTTP_500_INTERNAL_SERVER_ERROR,
            detail=f"Backup failed: {str(e)}"
        )
```

### Interface Endpoints

```python
# app/routers/interfaces.py
from fastapi import APIRouter, Depends, HTTPException, status, Path
from typing import List, Optional, Dict, Any
from pydantic import BaseModel
import logging

from app.services.mikrotik import router_manager
from app.dependencies import get_current_user

logger = logging.getLogger(__name__)
router = APIRouter()


class InterfaceUpdate(BaseModel):
    disabled: Optional[bool] = None
    comment: Optional[str] = None
    mtu: Optional[int] = None
    name: Optional[str] = None


@router.get("/")
async def list_interfaces(
    router_id: str = Path(..., description="Router ID"),
    include_stats: bool = False,
    current_user = Depends(get_current_user)
):
    """
    ดึงรายการ interfaces ทั้งหมดของ router
    
    - **include_stats**: รวม traffic statistics ด้วย
    """
    try:
        with router_manager.get_connection(router_id) as conn:
            interfaces = conn.get_interfaces()
            
            if include_stats:
                stats = conn.get_interface_stats()
                stats_map = {s['name']: s for s in stats}
                
                for iface in interfaces:
                    iface_name = iface.get('name', '')
                    if iface_name in stats_map:
                        iface['stats'] = stats_map[iface_name]
            
            return {
                "router_id": router_id,
                "count": len(interfaces),
                "interfaces": interfaces
            }
    except ValueError as e:
        raise HTTPException(status_code=404, detail=str(e))
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@router.get("/{interface_name}")
async def get_interface(
    router_id: str = Path(...),
    interface_name: str = Path(...),
    current_user = Depends(get_current_user)
):
    """ดึงข้อมูล interface เฉพาะตัว"""
    try:
        with router_manager.get_connection(router_id) as conn:
            interfaces = conn.get_interfaces()
            
            for iface in interfaces:
                if iface.get('name') == interface_name:
                    # เพิ่ม IP addresses
                    ip_addresses = conn.get_ip_addresses()
                    iface_ips = [
                        ip for ip in ip_addresses 
                        if ip.get('interface') == interface_name
                    ]
                    iface['ip_addresses'] = iface_ips
                    
                    return iface
            
            raise HTTPException(
                status_code=404,
                detail=f"Interface {interface_name} not found"
            )
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@router.put("/{interface_name}")
async def update_interface(
    router_id: str = Path(...),
    interface_name: str = Path(...),
    update_data: InterfaceUpdate = None,
    current_user = Depends(get_current_user)
):
    """อัพเดต interface configuration"""
    try:
        with router_manager.get_connection(router_id) as conn:
            # หา interface ID ก่อน
            interfaces = conn.get_interfaces()
            target_id = None
            
            for iface in interfaces:
                if iface.get('name') == interface_name:
                    target_id = iface.get('.id')
                    break
            
            if not target_id:
                raise HTTPException(status_code=404, detail="Interface not found")
            
            # อัพเดต
            params = {'=.id': target_id}
            if update_data.disabled is not None:
                params['=disabled'] = 'yes' if update_data.disabled else 'no'
            if update_data.comment is not None:
                params['=comment'] = update_data.comment
            if update_data.mtu is not None:
                params['=mtu'] = str(update_data.mtu)
            
            conn.run_command('/interface/set', **params)
            
            return {"success": True, "message": f"Interface {interface_name} updated"}
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))


@router.post("/{interface_name}/enable")
async def enable_interface(
    router_id: str = Path(...),
    interface_name: str = Path(...),
    current_user = Depends(get_current_user)
):
    """เปิดใช้งาน interface"""
    try:
        with router_manager.get_connection(router_id) as conn:
            interfaces = conn.get_interfaces()
            
            for iface in interfaces:
                if iface.get('name') == interface_name:
                    conn.run_command('/interface/enable', 
                                   **{'=.id': iface.get('.id')})
                    return {"success": True, "message": f"Interface {interface_name} enabled"}
            
            raise HTTPException(status_code=404, detail="Interface not found")
    except HTTPException:
        raise
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))
```

---

## 5. Authentication Middleware {#authentication}

```python
# app/middleware/auth.py
from fastapi import HTTPException, Depends, status
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from jose import JWTError, jwt
from passlib.context import CryptContext
from datetime import datetime, timedelta
from typing import Optional
import logging

from app.config import settings
from app.models.user import User, TokenData

logger = logging.getLogger(__name__)

# Password hashing
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")

# OAuth2 scheme
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/api/v1/auth/token")


def verify_password(plain_password: str, hashed_password: str) -> bool:
    """ตรวจสอบ password"""
    return pwd_context.verify(plain_password, hashed_password)


def get_password_hash(password: str) -> str:
    """Hash password"""
    return pwd_context.hash(password)


def create_access_token(data: dict, expires_delta: Optional[timedelta] = None) -> str:
    """สร้าง JWT access token"""
    to_encode = data.copy()
    
    if expires_delta:
        expire = datetime.utcnow() + expires_delta
    else:
        expire = datetime.utcnow() + timedelta(
            minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES
        )
    
    to_encode.update({"exp": expire, "type": "access"})
    encoded_jwt = jwt.encode(
        to_encode, 
        settings.SECRET_KEY, 
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt


def create_refresh_token(data: dict) -> str:
    """สร้าง JWT refresh token"""
    to_encode = data.copy()
    expire = datetime.utcnow() + timedelta(days=settings.REFRESH_TOKEN_EXPIRE_DAYS)
    to_encode.update({"exp": expire, "type": "refresh"})
    
    encoded_jwt = jwt.encode(
        to_encode,
        settings.SECRET_KEY,
        algorithm=settings.ALGORITHM
    )
    return encoded_jwt


async def get_current_user(token: str = Depends(oauth2_scheme)):
    """Dependency สำหรับดึง current user จาก token"""
    credentials_exception = HTTPException(
        status_code=status.HTTP_401_UNAUTHORIZED,
        detail="Could not validate credentials",
        headers={"WWW-Authenticate": "Bearer"},
    )
    
    try:
        payload = jwt.decode(
            token, 
            settings.SECRET_KEY, 
            algorithms=[settings.ALGORITHM]
        )
        username: str = payload.get("sub")
        token_type: str = payload.get("type")
        
        if username is None or token_type != "access":
            raise credentials_exception
        
        token_data = TokenData(username=username)
    except JWTError as e:
        logger.warning(f"JWT validation failed: {e}")
        raise credentials_exception
    
    # ดึง user จาก database
    # user = get_user(db, username=token_data.username)
    # if user is None:
    #     raise credentials_exception
    
    return token_data


async def require_role(*roles: str):
    """Dependency factory สำหรับตรวจสอบ role"""
    async def check_role(current_user = Depends(get_current_user)):
        if current_user.role not in roles:
            raise HTTPException(
                status_code=status.HTTP_403_FORBIDDEN,
                detail=f"Insufficient permissions. Required: {roles}"
            )
        return current_user
    return check_role


# app/routers/auth.py
from fastapi import APIRouter, Depends, HTTPException, status
from fastapi.security import OAuth2PasswordRequestForm
from pydantic import BaseModel
from datetime import timedelta
from typing import Optional

router = APIRouter()


class Token(BaseModel):
    access_token: str
    refresh_token: str
    token_type: str
    expires_in: int


class TokenRefresh(BaseModel):
    refresh_token: str


@router.post("/token", response_model=Token)
async def login_for_access_token(
    form_data: OAuth2PasswordRequestForm = Depends()
):
    """
    Login และรับ JWT tokens
    
    - **username**: ชื่อผู้ใช้
    - **password**: รหัสผ่าน
    """
    # ตรวจสอบ credentials (ดึงจาก database จริงๆ)
    # user = authenticate_user(db, form_data.username, form_data.password)
    
    # ตัวอย่างแบบง่าย
    if form_data.username != "admin" or form_data.password != "password":
        raise HTTPException(
            status_code=status.HTTP_401_UNAUTHORIZED,
            detail="Incorrect username or password",
            headers={"WWW-Authenticate": "Bearer"},
        )
    
    access_token_expires = timedelta(minutes=settings.ACCESS_TOKEN_EXPIRE_MINUTES)
    access_token = create_access_token(
        data={"sub": form_data.username},
        expires_delta=access_token_expires
    )
    refresh_token = create_refresh_token(data={"sub": form_data.username})
    
    return {
        "access_token": access_token,
        "refresh_token": refresh_token,
        "token_type": "bearer",
        "expires_in": settings.ACCESS_TOKEN_EXPIRE_MINUTES * 60
    }


@router.post("/refresh", response_model=Token)
async def refresh_access_token(token_data: TokenRefresh):
    """Refresh access token ด้วย refresh token"""
    try:
        payload = jwt.decode(
            token_data.refresh_token,
            settings.SECRET_KEY,
            algorithms=[settings.ALGORITHM]
        )
        
        if payload.get("type") != "refresh":
            raise HTTPException(
                status_code=400,
                detail="Invalid token type"
            )
        
        username = payload.get("sub")
        new_access_token = create_access_token(data={"sub": username})
        new_refresh_token = create_refresh_token(data={"sub": username})
        
        return {
            "access_token": new_access_token,
            "refresh_token": new_refresh_token,
            "token_type": "bearer",
            "expires_in": settings.ACCESS_TOKEN_EXPIRE_MINUTES * 60
        }
    except JWTError:
        raise HTTPException(
            status_code=401,
            detail="Invalid refresh token"
        )
```

---

## 6. Request Validation {#validation}

```python
# app/models/firewall.py
from pydantic import BaseModel, validator, Field
from typing import Optional, List
from enum import Enum


class FirewallChain(str, Enum):
    INPUT = "input"
    FORWARD = "forward"
    OUTPUT = "output"


class FirewallAction(str, Enum):
    ACCEPT = "accept"
    DROP = "drop"
    REJECT = "reject"
    LOG = "log"
    PASSTHROUGH = "passthrough"
    JUMP = "jump"
    RETURN = "return"
    TARPIT = "tarpit"


class FirewallProtocol(str, Enum):
    TCP = "tcp"
    UDP = "udp"
    ICMP = "icmp"
    GRE = "gre"
    ESP = "esp"
    AH = "ah"


class FirewallRuleCreate(BaseModel):
    chain: FirewallChain
    action: FirewallAction
    src_address: Optional[str] = Field(None, example="192.168.1.0/24")
    dst_address: Optional[str] = Field(None, example="0.0.0.0/0")
    src_address_list: Optional[str] = None
    dst_address_list: Optional[str] = None
    protocol: Optional[FirewallProtocol] = None
    src_port: Optional[str] = Field(None, example="1024-65535")
    dst_port: Optional[str] = Field(None, example="80,443")
    in_interface: Optional[str] = None
    out_interface: Optional[str] = None
    comment: Optional[str] = Field(None, max_length=255)
    disabled: bool = False
    log: bool = False
    log_prefix: Optional[str] = Field(None, max_length=20)
    
    @validator('src_address', 'dst_address')
    def validate_ip_address(cls, v):
        """ตรวจสอบรูปแบบ IP address/CIDR"""
        if v is None:
            return v
        
        import ipaddress
        try:
            ipaddress.ip_network(v, strict=False)
        except ValueError:
            try:
                ipaddress.ip_address(v)
            except ValueError:
                raise ValueError(f"Invalid IP address or CIDR: {v}")
        return v
    
    @validator('dst_port', 'src_port')
    def validate_port(cls, v):
        """ตรวจสอบรูปแบบ port"""
        if v is None:
            return v
        
        parts = v.split(',')
        for part in parts:
            part = part.strip()
            if '-' in part:
                start, end = part.split('-', 1)
                if not (1 <= int(start) <= 65535 and 1 <= int(end) <= 65535):
                    raise ValueError(f"Invalid port range: {part}")
            else:
                if not 1 <= int(part) <= 65535:
                    raise ValueError(f"Invalid port: {part}")
        return v
    
    class Config:
        schema_extra = {
            "example": {
                "chain": "input",
                "action": "accept",
                "protocol": "tcp",
                "dst_port": "80,443",
                "src_address": "192.168.1.0/24",
                "comment": "Allow web traffic from LAN"
            }
        }
```

---

## 7. Response Formatting {#response}

```python
# app/utils/response.py
from typing import Any, Optional, Dict, List
from pydantic import BaseModel
import time


class APIResponse(BaseModel):
    """Standard API response format"""
    success: bool
    data: Optional[Any] = None
    message: Optional[str] = None
    errors: Optional[List[Dict]] = None
    meta: Optional[Dict] = None
    timestamp: float = None
    
    def __init__(self, **data):
        if 'timestamp' not in data:
            data['timestamp'] = time.time()
        super().__init__(**data)


class PaginatedResponse(APIResponse):
    """Paginated response format"""
    meta: Dict = {
        "page": 1,
        "per_page": 10,
        "total": 0,
        "total_pages": 0
    }


def success_response(data: Any, message: str = None, meta: Dict = None) -> Dict:
    """สร้าง success response"""
    response = {
        "success": True,
        "data": data,
        "timestamp": time.time()
    }
    if message:
        response["message"] = message
    if meta:
        response["meta"] = meta
    return response


def error_response(message: str, errors: List = None, code: str = None) -> Dict:
    """สร้าง error response"""
    response = {
        "success": False,
        "message": message,
        "timestamp": time.time()
    }
    if errors:
        response["errors"] = errors
    if code:
        response["code"] = code
    return response


def paginated_response(
    data: List, 
    total: int, 
    page: int, 
    per_page: int
) -> Dict:
    """สร้าง paginated response"""
    import math
    return {
        "success": True,
        "data": data,
        "meta": {
            "page": page,
            "per_page": per_page,
            "total": total,
            "total_pages": math.ceil(total / per_page)
        },
        "timestamp": time.time()
    }
```

---

## 8. Swagger Documentation {#swagger}

```python
# app/main.py - เพิ่ม custom OpenAPI
from fastapi.openapi.utils import get_openapi


def custom_openapi():
    if app.openapi_schema:
        return app.openapi_schema
    
    openapi_schema = get_openapi(
        title="MikroTik Management API",
        version="1.0.0",
        description="""
## MikroTik Management REST API

### Authentication
API ใช้ JWT Bearer authentication. ต้อง login ที่ `/api/v1/auth/token` ก่อนเพื่อได้รับ token

```
Authorization: Bearer <your_token>
```

### Rate Limiting
- 100 requests ต่อ minute ต่อ IP
- Header `X-RateLimit-Remaining` บอกจำนวน requests ที่เหลือ

### Error Codes

| Code | Description |
|------|-------------|
| 400 | Bad Request - ข้อมูลที่ส่งมาไม่ถูกต้อง |
| 401 | Unauthorized - ต้อง authentication |
| 403 | Forbidden - ไม่มีสิทธิ์ |
| 404 | Not Found - ไม่พบ resource |
| 429 | Too Many Requests - เกิน rate limit |
| 500 | Internal Server Error |
        """,
        routes=app.routes,
    )
    
    # เพิ่ม security scheme
    openapi_schema["components"]["securitySchemes"] = {
        "BearerAuth": {
            "type": "http",
            "scheme": "bearer",
            "bearerFormat": "JWT"
        }
    }
    
    # ใส่ security ใน global
    openapi_schema["security"] = [{"BearerAuth": []}]
    
    app.openapi_schema = openapi_schema
    return app.openapi_schema


app.openapi = custom_openapi
```

---

## 9. Testing with Postman {#postman}

### Postman Collection

```json
{
  "info": {
    "name": "MikroTik Management API",
    "description": "Complete API collection for MikroTik management",
    "schema": "https://schema.getpostman.com/json/collection/v2.1.0/collection.json"
  },
  "auth": {
    "type": "bearer",
    "bearer": [
      {
        "key": "token",
        "value": "{{access_token}}",
        "type": "string"
      }
    ]
  },
  "variable": [
    {
      "key": "base_url",
      "value": "http://localhost:8000"
    },
    {
      "key": "access_token",
      "value": ""
    }
  ],
  "item": [
    {
      "name": "Authentication",
      "item": [
        {
          "name": "Login",
          "request": {
            "method": "POST",
            "url": "{{base_url}}/api/v1/auth/token",
            "body": {
              "mode": "urlencoded",
              "urlencoded": [
                {"key": "username", "value": "admin"},
                {"key": "password", "value": "password"}
              ]
            }
          },
          "event": [
            {
              "listen": "test",
              "script": {
                "exec": [
                  "var jsonData = pm.response.json();",
                  "pm.environment.set('access_token', jsonData.access_token);"
                ]
              }
            }
          ]
        }
      ]
    },
    {
      "name": "Routers",
      "item": [
        {
          "name": "List Routers",
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/v1/routers?skip=0&limit=10"
          }
        },
        {
          "name": "Get Router Status",
          "request": {
            "method": "GET",
            "url": "{{base_url}}/api/v1/routers/{{router_id}}/status"
          }
        }
      ]
    }
  ]
}
```

### Automated Testing

```python
# tests/test_routers.py
import pytest
from fastapi.testclient import TestClient
from app.main import app
import json

client = TestClient(app)


@pytest.fixture
def auth_headers():
    """Get authentication headers"""
    response = client.post(
        "/api/v1/auth/token",
        data={"username": "admin", "password": "password"}
    )
    token = response.json()["access_token"]
    return {"Authorization": f"Bearer {token}"}


def test_health_check():
    """ทดสอบ health check endpoint"""
    response = client.get("/health")
    assert response.status_code == 200
    assert response.json()["status"] == "healthy"


def test_list_routers_unauthenticated():
    """ทดสอบ access โดยไม่มี token"""
    response = client.get("/api/v1/routers/")
    assert response.status_code == 401


def test_list_routers_authenticated(auth_headers):
    """ทดสอบ list routers ด้วย authentication"""
    response = client.get("/api/v1/routers/", headers=auth_headers)
    assert response.status_code == 200
    data = response.json()
    assert "data" in data


def test_create_router_validation(auth_headers):
    """ทดสอบ validation ของ router creation"""
    # ข้อมูลไม่ครบ
    response = client.post(
        "/api/v1/routers/",
        headers=auth_headers,
        json={"name": "Test Router"}  # ขาด host, username, password
    )
    assert response.status_code == 422  # Validation error


def test_create_firewall_rule_invalid_ip(auth_headers):
    """ทดสอบ validation ของ IP address"""
    response = client.post(
        "/api/v1/routers/test-id/firewall/rules",
        headers=auth_headers,
        json={
            "chain": "input",
            "action": "accept",
            "src_address": "invalid_ip"  # IP ไม่ถูกต้อง
        }
    )
    assert response.status_code == 422
```

---

## 10. Deployment {#deployment}

### Docker Setup

```dockerfile
# Dockerfile
FROM python:3.11-slim

WORKDIR /app

# ติดตั้ง system dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    && rm -rf /var/lib/apt/lists/*

# Copy requirements
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt

# Copy application
COPY . .

# สร้าง non-root user
RUN adduser --disabled-password --gecos '' appuser && \
    chown -R appuser:appuser /app
USER appuser

EXPOSE 8000

CMD ["uvicorn", "app.main:app", "--host", "0.0.0.0", "--port", "8000"]
```

```yaml
# docker-compose.yml
version: '3.8'

services:
  api:
    build: .
    ports:
      - "8000:8000"
    environment:
      - DATABASE_URL=postgresql://postgres:password@db/mikrotik_db
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY=${SECRET_KEY}
    depends_on:
      - db
      - redis
    restart: unless-stopped
    healthcheck:
      test: ["CMD", "curl", "-f", "http://localhost:8000/health"]
      interval: 30s
      timeout: 10s
      retries: 3

  db:
    image: postgres:15
    environment:
      - POSTGRES_DB=mikrotik_db
      - POSTGRES_USER=postgres
      - POSTGRES_PASSWORD=password
    volumes:
      - postgres_data:/var/lib/postgresql/data
    restart: unless-stopped

  redis:
    image: redis:7-alpine
    restart: unless-stopped
    volumes:
      - redis_data:/data

  nginx:
    image: nginx:alpine
    ports:
      - "80:80"
      - "443:443"
    volumes:
      - ./nginx.conf:/etc/nginx/nginx.conf
      - ./ssl:/etc/nginx/ssl
    depends_on:
      - api
    restart: unless-stopped

volumes:
  postgres_data:
  redis_data:
```

```nginx
# nginx.conf
events {
    worker_connections 1024;
}

http {
    upstream api {
        server api:8000;
    }
    
    # Rate limiting
    limit_req_zone $binary_remote_addr zone=api:10m rate=10r/s;
    
    server {
        listen 80;
        server_name api.yourdomain.com;
        return 301 https://$server_name$request_uri;
    }
    
    server {
        listen 443 ssl http2;
        server_name api.yourdomain.com;
        
        ssl_certificate /etc/nginx/ssl/cert.pem;
        ssl_certificate_key /etc/nginx/ssl/key.pem;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-RSA-AES256-GCM-SHA512:DHE-RSA-AES256-GCM-SHA512;
        
        location /api/ {
            limit_req zone=api burst=20 nodelay;
            
            proxy_pass http://api;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            
            # Timeouts
            proxy_connect_timeout 60s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;
        }
        
        location /docs {
            proxy_pass http://api;
            proxy_set_header Host $host;
        }
    }
}
```

---

## 11. Lab: Complete REST API {#lab}

### Lab Objectives
1. ติดตั้ง FastAPI และ dependencies ทั้งหมด
2. เชื่อมต่อกับ MikroTik router จริง
3. Implement endpoints หลักทั้งหมด
4. เพิ่ม authentication และ authorization
5. Deploy ด้วย Docker

### Lab Steps

```bash
# Step 1: Setup project
mkdir mikrotik-api && cd mikrotik-api
python3 -m venv venv
source venv/bin/activate
pip install fastapi uvicorn librouteros python-jose passlib[bcrypt] pydantic-settings

# Step 2: สร้างโครงสร้างไฟล์
mkdir -p app/{routers,services,models,middleware}
touch app/__init__.py app/main.py app/config.py

# Step 3: Run development server
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000

# Step 4: Test API
curl http://localhost:8000/health
curl -X POST http://localhost:8000/api/v1/auth/token \
  -d "username=admin&password=password"

# Step 5: Build and run with Docker
docker build -t mikrotik-api .
docker run -p 8000:8000 mikrotik-api
```

### Verification Checklist

- [ ] API starts successfully
- [ ] Health check returns 200
- [ ] Authentication works (returns JWT token)
- [ ] Protected endpoints return 401 without token
- [ ] Can list routers with valid token
- [ ] Can create firewall rules
- [ ] Input validation works (rejects invalid data)
- [ ] Swagger docs available at `/docs`
- [ ] Docker container runs successfully

> **Note:** ใน production ต้องเปลี่ยน `SECRET_KEY` เป็นค่าที่ random และ secure

> **Warning:** อย่าเปิด debug mode ใน production เพราะจะ expose stack traces

---

## Summary

Part นี้ครอบคลุม:
- **REST API Design** ด้วย FastAPI สำหรับ MikroTik management
- **Authentication** ด้วย JWT tokens
- **Validation** ของ input data ด้วย Pydantic
- **Service Layer** สำหรับ MikroTik connections
- **Deployment** ด้วย Docker และ Nginx

สิ่งที่ต้องจำ:
1. ใช้ HTTPS เสมอใน production
2. ใส่ rate limiting เพื่อป้องกัน abuse
3. Log ทุก request สำหรับ audit
4. Test ทุก endpoint ก่อน deploy

---

[← Part 60: Advanced Monitoring](part-060-advanced-monitoring.md) | [Part 62: WebSocket →](part-062-websocket.md)
