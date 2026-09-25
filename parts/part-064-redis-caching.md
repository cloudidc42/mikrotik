# Part 64: Redis Caching สำหรับ Network Applications

## สารบัญ
1. [Redis for Network Applications](#intro)
2. [Caching Router Data](#caching)
3. [Session Management](#sessions)
4. [Rate Limiting with Redis](#rate-limiting)
5. [Pub/Sub for Real-time](#pubsub)
6. [Data Expiry Strategies](#expiry)
7. [Redis Cluster](#cluster)
8. [Cache Invalidation Patterns](#invalidation)
9. [Performance Optimization](#performance)
10. [Lab: Cached Network Dashboard](#lab)

---

## 1. Redis for Network Applications {#intro}

Redis เป็น in-memory data store ที่เหมาะมากสำหรับ network management applications เพราะความเร็วสูงและ features หลากหลาย

### ทำไมต้องใช้ Redis

| Use Case | ทำไม Redis เหมาะ |
|---------|----------------|
| Router data caching | ลด API calls ไปยัง MikroTik |
| Session management | Fast token validation |
| Rate limiting | Atomic operations |
| Real-time metrics | Low latency pub/sub |
| Job queues | Background task processing |
| Leaderboards | Sorted sets |

### Architecture

```
┌─────────────────────────────────────────────────────────┐
│                    FastAPI Application                    │
│                                                          │
│  GET /routers/1/interfaces                               │
│       │                                                  │
│       ├─ Cache HIT ──────> Return cached data (fast!)   │
│       │                                                  │
│       └─ Cache MISS ──────> Fetch from MikroTik         │
│                              │                           │
│                              └─> Store in Redis          │
│                                  Return data             │
└─────────────────────────────────────────────────────────┘
           │                              │
           ▼                              ▼
    ┌──────────┐                   ┌──────────────┐
    │  Redis   │                   │   MikroTik   │
    │ (Cache)  │                   │   Routers    │
    └──────────┘                   └──────────────┘
```

---

## 2. Setup Redis

```bash
# ติดตั้ง redis library
pip install redis[hiredis] aioredis

# หรือ Node.js
npm install ioredis
```

```python
# app/services/redis_client.py
import redis
import aioredis
from typing import Optional, Any, Union
import json
import logging
from contextlib import asynccontextmanager

from app.config import settings

logger = logging.getLogger(__name__)


class RedisClient:
    """Redis client wrapper"""
    
    def __init__(self, url: str = None):
        self.url = url or settings.REDIS_URL
        self._sync_client: Optional[redis.Redis] = None
        self._async_pool = None
    
    @property
    def sync_client(self) -> redis.Redis:
        """Synchronous Redis client"""
        if not self._sync_client:
            self._sync_client = redis.from_url(
                self.url,
                decode_responses=True,
                socket_connect_timeout=5,
                socket_timeout=5,
                retry_on_timeout=True,
                health_check_interval=30
            )
        return self._sync_client
    
    async def get_async_client(self):
        """Asynchronous Redis client"""
        if not self._async_pool:
            self._async_pool = await aioredis.from_url(
                self.url,
                decode_responses=True,
                max_connections=20
            )
        return self._async_pool
    
    def ping(self) -> bool:
        """ทดสอบ connection"""
        try:
            return self.sync_client.ping()
        except Exception as e:
            logger.error(f"Redis ping failed: {e}")
            return False
    
    def get(self, key: str) -> Optional[str]:
        """Get value"""
        try:
            return self.sync_client.get(key)
        except Exception as e:
            logger.error(f"Redis GET error: {e}")
            return None
    
    def set(self, key: str, value: Any, ttl: int = None) -> bool:
        """Set value with optional TTL"""
        try:
            serialized = json.dumps(value) if not isinstance(value, str) else value
            if ttl:
                return self.sync_client.setex(key, ttl, serialized)
            return self.sync_client.set(key, serialized)
        except Exception as e:
            logger.error(f"Redis SET error: {e}")
            return False
    
    def get_json(self, key: str) -> Optional[Any]:
        """Get JSON value"""
        value = self.get(key)
        if value:
            try:
                return json.loads(value)
            except json.JSONDecodeError:
                return value
        return None
    
    def delete(self, *keys: str) -> int:
        """Delete keys"""
        try:
            return self.sync_client.delete(*keys)
        except Exception as e:
            logger.error(f"Redis DELETE error: {e}")
            return 0
    
    def exists(self, key: str) -> bool:
        """Check if key exists"""
        try:
            return bool(self.sync_client.exists(key))
        except Exception:
            return False
    
    def expire(self, key: str, ttl: int) -> bool:
        """Set TTL on existing key"""
        try:
            return self.sync_client.expire(key, ttl)
        except Exception:
            return False
    
    def ttl(self, key: str) -> int:
        """Get remaining TTL"""
        try:
            return self.sync_client.ttl(key)
        except Exception:
            return -1
    
    def keys_by_pattern(self, pattern: str):
        """Find keys by pattern (ระวัง: ช้าใน production)"""
        try:
            return self.sync_client.keys(pattern)
        except Exception:
            return []
    
    def scan_keys(self, pattern: str, count: int = 100):
        """Scan keys by pattern (ปลอดภัยกว่า KEYS)"""
        cursor = 0
        while True:
            cursor, keys = self.sync_client.scan(
                cursor=cursor, 
                match=pattern, 
                count=count
            )
            for key in keys:
                yield key
            if cursor == 0:
                break


# Singleton instance
redis_client = RedisClient()
```

---

## 3. Caching Router Data {#caching}

```python
# app/services/cache_service.py
from typing import Any, Optional, Callable, TypeVar, Dict
from functools import wraps
import json
import hashlib
import time
import logging

from app.services.redis_client import redis_client

logger = logging.getLogger(__name__)

T = TypeVar('T')

# Cache TTL values (seconds)
CACHE_TTL = {
    'router_list': 60,           # 1 minute
    'router_status': 30,         # 30 seconds
    'interfaces': 60,            # 1 minute
    'interface_stats': 5,        # 5 seconds (near real-time)
    'firewall_rules': 300,       # 5 minutes
    'dhcp_leases': 30,           # 30 seconds
    'routing_table': 120,        # 2 minutes
    'system_resources': 10,      # 10 seconds
    'arp_table': 30,             # 30 seconds
}


def cache_key(prefix: str, *args, **kwargs) -> str:
    """สร้าง cache key จาก arguments"""
    key_parts = [prefix]
    key_parts.extend(str(a) for a in args)
    key_parts.extend(f"{k}={v}" for k, v in sorted(kwargs.items()))
    
    key_str = ':'.join(key_parts)
    
    # Hash ถ้า key ยาวเกินไป
    if len(key_str) > 200:
        key_hash = hashlib.md5(key_str.encode()).hexdigest()
        return f"{prefix}:hash:{key_hash}"
    
    return key_str


def cached(ttl_key: str = None, ttl: int = None):
    """Decorator สำหรับ cache function results"""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            # สร้าง cache key
            actual_ttl = ttl or CACHE_TTL.get(ttl_key, 60)
            func_name = f"{func.__module__}.{func.__name__}"
            key = cache_key(func_name, *args, **kwargs)
            
            # ลอง get จาก cache
            cached_value = redis_client.get_json(key)
            if cached_value is not None:
                logger.debug(f"Cache HIT: {key}")
                return cached_value
            
            # Cache miss - เรียก function จริง
            logger.debug(f"Cache MISS: {key}")
            result = func(*args, **kwargs)
            
            # Store ใน cache
            if result is not None:
                redis_client.set(key, result, ttl=actual_ttl)
            
            return result
        
        # เพิ่ม method สำหรับ invalidate cache
        wrapper.invalidate = lambda *args, **kwargs: redis_client.delete(
            cache_key(f"{func.__module__}.{func.__name__}", *args, **kwargs)
        )
        
        return wrapper
    return decorator


class RouterDataCache:
    """Cache service สำหรับ router data"""
    
    def __init__(self):
        self.redis = redis_client
    
    def get_interfaces(self, router_id: str) -> Optional[list]:
        """Get cached interfaces"""
        key = f"router:{router_id}:interfaces"
        return self.redis.get_json(key)
    
    def set_interfaces(self, router_id: str, data: list):
        """Cache interfaces data"""
        key = f"router:{router_id}:interfaces"
        self.redis.set(key, data, ttl=CACHE_TTL['interfaces'])
    
    def get_interface_stats(self, router_id: str) -> Optional[dict]:
        """Get cached interface statistics"""
        key = f"router:{router_id}:interface_stats"
        return self.redis.get_json(key)
    
    def set_interface_stats(self, router_id: str, data: dict):
        """Cache interface stats (short TTL)"""
        key = f"router:{router_id}:interface_stats"
        self.redis.set(key, data, ttl=CACHE_TTL['interface_stats'])
    
    def get_firewall_rules(self, router_id: str, chain: str = 'all') -> Optional[list]:
        """Get cached firewall rules"""
        key = f"router:{router_id}:firewall:{chain}"
        return self.redis.get_json(key)
    
    def set_firewall_rules(self, router_id: str, data: list, chain: str = 'all'):
        """Cache firewall rules"""
        key = f"router:{router_id}:firewall:{chain}"
        self.redis.set(key, data, ttl=CACHE_TTL['firewall_rules'])
    
    def get_system_resources(self, router_id: str) -> Optional[dict]:
        """Get cached system resources"""
        key = f"router:{router_id}:resources"
        return self.redis.get_json(key)
    
    def set_system_resources(self, router_id: str, data: dict):
        """Cache system resources"""
        key = f"router:{router_id}:resources"
        self.redis.set(key, data, ttl=CACHE_TTL['system_resources'])
    
    def invalidate_router(self, router_id: str):
        """ลบ cache ทั้งหมดของ router"""
        pattern = f"router:{router_id}:*"
        count = 0
        for key in self.redis.scan_keys(pattern):
            self.redis.delete(key)
            count += 1
        logger.info(f"Invalidated {count} cache entries for router {router_id}")
        return count
    
    def invalidate_by_type(self, router_id: str, data_type: str):
        """ลบ cache เฉพาะ type"""
        pattern = f"router:{router_id}:{data_type}*"
        for key in self.redis.scan_keys(pattern):
            self.redis.delete(key)
    
    def get_cache_stats(self) -> Dict:
        """ดึง cache statistics"""
        try:
            info = redis_client.sync_client.info('stats')
            memory = redis_client.sync_client.info('memory')
            
            return {
                "hits": info.get('keyspace_hits', 0),
                "misses": info.get('keyspace_misses', 0),
                "hit_rate": self._calc_hit_rate(
                    info.get('keyspace_hits', 0),
                    info.get('keyspace_misses', 0)
                ),
                "memory_used": memory.get('used_memory_human', '0'),
                "total_keys": redis_client.sync_client.dbsize()
            }
        except Exception as e:
            logger.error(f"Failed to get cache stats: {e}")
            return {}
    
    def _calc_hit_rate(self, hits: int, misses: int) -> float:
        total = hits + misses
        if total == 0:
            return 0.0
        return round((hits / total) * 100, 2)


# Singleton
router_cache = RouterDataCache()


# Example usage ใน API endpoint
class RouterService:
    """Service layer พร้อม caching"""
    
    def __init__(self, router_manager, cache: RouterDataCache):
        self.router_manager = router_manager
        self.cache = cache
    
    def get_interfaces(self, router_id: str, force_refresh: bool = False):
        """Get interfaces พร้อม caching"""
        if not force_refresh:
            # ลอง get จาก cache
            cached = self.cache.get_interfaces(router_id)
            if cached is not None:
                return {"data": cached, "source": "cache"}
        
        # Fetch จาก router จริง
        with self.router_manager.get_connection(router_id) as conn:
            interfaces = conn.get_interfaces()
        
        # Store ใน cache
        self.cache.set_interfaces(router_id, interfaces)
        
        return {"data": interfaces, "source": "router"}
    
    def get_system_resources(self, router_id: str):
        """Get system resources พร้อม caching"""
        cached = self.cache.get_system_resources(router_id)
        if cached:
            return cached
        
        with self.router_manager.get_connection(router_id) as conn:
            resources = conn.get_system_resources()
        
        self.cache.set_system_resources(router_id, resources)
        return resources
```

---

## 4. Session Management {#sessions}

```python
# app/services/session_service.py
import uuid
import json
import time
from typing import Optional, Dict, Any
from app.services.redis_client import redis_client


class SessionService:
    """จัดการ user sessions ด้วย Redis"""
    
    SESSION_PREFIX = "session:"
    DEFAULT_TTL = 86400  # 24 hours
    
    def create_session(self, user_id: str, user_data: Dict) -> str:
        """สร้าง session ใหม่"""
        session_id = str(uuid.uuid4())
        key = f"{self.SESSION_PREFIX}{session_id}"
        
        session_data = {
            "user_id": user_id,
            "created_at": time.time(),
            "last_active": time.time(),
            **user_data
        }
        
        redis_client.set(key, session_data, ttl=self.DEFAULT_TTL)
        
        # เก็บ list of sessions ต่อ user
        user_sessions_key = f"user_sessions:{user_id}"
        redis_client.sync_client.sadd(user_sessions_key, session_id)
        redis_client.sync_client.expire(user_sessions_key, self.DEFAULT_TTL)
        
        return session_id
    
    def get_session(self, session_id: str) -> Optional[Dict]:
        """Get session data"""
        key = f"{self.SESSION_PREFIX}{session_id}"
        data = redis_client.get_json(key)
        
        if data:
            # Update last active time
            data['last_active'] = time.time()
            redis_client.set(key, data, ttl=self.DEFAULT_TTL)
        
        return data
    
    def delete_session(self, session_id: str) -> bool:
        """ลบ session (logout)"""
        key = f"{self.SESSION_PREFIX}{session_id}"
        session = redis_client.get_json(key)
        
        if session:
            user_id = session.get('user_id')
            if user_id:
                redis_client.sync_client.srem(f"user_sessions:{user_id}", session_id)
        
        return bool(redis_client.delete(key))
    
    def delete_all_user_sessions(self, user_id: str) -> int:
        """ลบทุก sessions ของ user (force logout ทุก device)"""
        user_sessions_key = f"user_sessions:{user_id}"
        session_ids = redis_client.sync_client.smembers(user_sessions_key)
        
        count = 0
        for session_id in session_ids:
            redis_client.delete(f"{self.SESSION_PREFIX}{session_id}")
            count += 1
        
        redis_client.delete(user_sessions_key)
        return count
    
    def extend_session(self, session_id: str, ttl: int = None) -> bool:
        """ต่ออายุ session"""
        key = f"{self.SESSION_PREFIX}{session_id}"
        return redis_client.expire(key, ttl or self.DEFAULT_TTL)


session_service = SessionService()
```

---

## 5. Rate Limiting with Redis {#rate-limiting}

```python
# app/middleware/rate_limit.py
from fastapi import Request, Response, HTTPException, status
from starlette.middleware.base import BaseHTTPMiddleware
import time
import logging

from app.services.redis_client import redis_client

logger = logging.getLogger(__name__)


class RateLimitMiddleware(BaseHTTPMiddleware):
    """Sliding window rate limiter ด้วย Redis"""
    
    def __init__(self, app, max_requests: int = 100, window_seconds: int = 60):
        super().__init__(app)
        self.max_requests = max_requests
        self.window_seconds = window_seconds
    
    async def dispatch(self, request: Request, call_next):
        # ดึง client identifier
        client_ip = request.client.host
        user = getattr(request.state, 'user', None)
        identifier = f"user:{user.id}" if user else f"ip:{client_ip}"
        
        # ข้าม rate limit สำหรับ health check
        if request.url.path in ['/health', '/']:
            return await call_next(request)
        
        # ตรวจสอบ rate limit
        allowed, remaining, reset_time = self.check_rate_limit(identifier)
        
        # เพิ่ม rate limit headers
        response = await call_next(request)
        response.headers['X-RateLimit-Limit'] = str(self.max_requests)
        response.headers['X-RateLimit-Remaining'] = str(remaining)
        response.headers['X-RateLimit-Reset'] = str(reset_time)
        
        if not allowed:
            return Response(
                content='{"error": "rate_limit_exceeded", "message": "Too many requests"}',
                status_code=429,
                headers={
                    'Content-Type': 'application/json',
                    'X-RateLimit-Limit': str(self.max_requests),
                    'X-RateLimit-Remaining': '0',
                    'X-RateLimit-Reset': str(reset_time),
                    'Retry-After': str(reset_time - int(time.time()))
                }
            )
        
        return response
    
    def check_rate_limit(self, identifier: str):
        """Sliding window rate limit check"""
        now = time.time()
        window_start = now - self.window_seconds
        
        key = f"rate_limit:{identifier}"
        
        pipe = redis_client.sync_client.pipeline()
        
        # ลบ requests ที่เก่ากว่า window
        pipe.zremrangebyscore(key, '-inf', window_start)
        
        # นับ requests ใน window ปัจจุบัน
        pipe.zcard(key)
        
        # เพิ่ม request ปัจจุบัน
        pipe.zadd(key, {str(now): now})
        
        # ตั้ง expiry
        pipe.expire(key, self.window_seconds + 1)
        
        results = pipe.execute()
        
        request_count = results[1]
        remaining = max(0, self.max_requests - request_count - 1)
        reset_time = int(now + self.window_seconds)
        
        allowed = request_count < self.max_requests
        
        return allowed, remaining, reset_time


class APIKeyRateLimiter:
    """Rate limiter สำหรับ API keys (different limits per key)"""
    
    API_KEY_LIMITS = {
        'basic': {'requests': 100, 'window': 60},
        'standard': {'requests': 1000, 'window': 60},
        'premium': {'requests': 10000, 'window': 60},
        'enterprise': {'requests': 100000, 'window': 60},
    }
    
    def check(self, api_key: str, tier: str = 'basic') -> Dict:
        limits = self.API_KEY_LIMITS.get(tier, self.API_KEY_LIMITS['basic'])
        
        key = f"api_key_rate:{api_key}"
        now = int(time.time())
        window_start = now - limits['window']
        
        pipe = redis_client.sync_client.pipeline()
        pipe.zremrangebyscore(key, '-inf', window_start)
        pipe.zcard(key)
        pipe.zadd(key, {str(now + id(api_key)): now})
        pipe.expire(key, limits['window'] + 1)
        results = pipe.execute()
        
        count = results[1]
        allowed = count < limits['requests']
        
        return {
            "allowed": allowed,
            "limit": limits['requests'],
            "remaining": max(0, limits['requests'] - count - 1),
            "reset": now + limits['window']
        }
```

---

## 6. Pub/Sub for Real-time {#pubsub}

```python
# app/services/pubsub_service.py
import json
import threading
import logging
from typing import Callable, Dict, List
import redis

from app.services.redis_client import RedisClient

logger = logging.getLogger(__name__)


class PubSubService:
    """Redis Pub/Sub สำหรับ real-time event broadcasting"""
    
    CHANNELS = {
        'router_alerts': 'mikrotik:alerts:*',
        'router_status': 'mikrotik:status:*',
        'traffic_updates': 'mikrotik:traffic:*',
        'config_changes': 'mikrotik:config:*',
        'system_events': 'mikrotik:system',
    }
    
    def __init__(self, redis_url: str):
        self.redis_url = redis_url
        self._publisher = redis.from_url(redis_url, decode_responses=True)
        self._subscribers: Dict[str, List[Callable]] = {}
        self._pubsub = None
        self._listener_thread = None
    
    def publish(self, channel: str, message: Dict) -> int:
        """Publish message ไปยัง channel"""
        try:
            payload = json.dumps(message)
            recipients = self._publisher.publish(channel, payload)
            logger.debug(f"Published to {channel}: {recipients} recipients")
            return recipients
        except Exception as e:
            logger.error(f"Publish error: {e}")
            return 0
    
    def publish_alert(self, router_id: str, alert_data: Dict) -> int:
        """Publish router alert"""
        channel = f"mikrotik:alerts:{router_id}"
        return self.publish(channel, {
            "type": "alert",
            "router_id": router_id,
            **alert_data
        })
    
    def publish_status_change(self, router_id: str, status: str) -> int:
        """Publish router status change"""
        channel = f"mikrotik:status:{router_id}"
        return self.publish(channel, {
            "type": "status_change",
            "router_id": router_id,
            "status": status
        })
    
    def subscribe(self, pattern: str, callback: Callable):
        """Subscribe to channel pattern"""
        if pattern not in self._subscribers:
            self._subscribers[pattern] = []
        self._subscribers[pattern].append(callback)
        
        # เริ่ม listener ถ้ายังไม่มี
        if not self._listener_thread or not self._listener_thread.is_alive():
            self._start_listener()
    
    def _start_listener(self):
        """เริ่ม listener thread"""
        subscriber = redis.from_url(self.redis_url, decode_responses=True)
        self._pubsub = subscriber.pubsub()
        
        # Subscribe to all patterns
        patterns = list(self._subscribers.keys())
        if patterns:
            self._pubsub.psubscribe(*patterns)
        
        def listen():
            for message in self._pubsub.listen():
                if message['type'] in ['pmessage', 'message']:
                    channel = message['channel']
                    pattern = message.get('pattern', channel)
                    
                    try:
                        data = json.loads(message['data'])
                    except:
                        data = message['data']
                    
                    # เรียก callbacks
                    callbacks = self._subscribers.get(pattern, [])
                    for callback in callbacks:
                        try:
                            callback(channel, data)
                        except Exception as e:
                            logger.error(f"Callback error: {e}")
        
        self._listener_thread = threading.Thread(target=listen, daemon=True)
        self._listener_thread.start()
        logger.info("PubSub listener started")


# ตัวอย่างการใช้งานกับ WebSocket
class WebSocketPubSubBridge:
    """Bridge ระหว่าง Redis PubSub และ WebSocket"""
    
    def __init__(self, pubsub: PubSubService, socket_io_server):
        self.pubsub = pubsub
        self.io = socket_io_server
        self._setup_subscriptions()
    
    def _setup_subscriptions(self):
        """Setup Redis subscriptions"""
        self.pubsub.subscribe(
            'mikrotik:alerts:*',
            self._handle_alert
        )
        self.pubsub.subscribe(
            'mikrotik:status:*',
            self._handle_status_change
        )
    
    def _handle_alert(self, channel: str, data: Dict):
        """Forward alert ไปยัง WebSocket clients"""
        router_id = channel.split(':')[-1]
        
        # Broadcast ไปยัง room ที่ subscribe
        self.io.to(f"alerts:{router_id}").emit('alert', data)
        self.io.to('alerts:all').emit('alert', data)
    
    def _handle_status_change(self, channel: str, data: Dict):
        """Forward status change"""
        router_id = channel.split(':')[-1]
        self.io.to(f"status:{router_id}").emit('status_change', data)
```

---

## 7. Data Expiry Strategies {#expiry}

```python
# app/services/expiry_strategy.py
"""
Data Expiry Strategies สำหรับ Different Types of Data

1. Volatile - ข้อมูลที่ expire เร็ว (interface stats)
2. Persistent - ข้อมูลที่ expire ช้า (firewall rules)
3. Event-driven - ข้อมูลที่ expire เมื่อมี event
4. Sliding window - expire ต่อจากการใช้งานล่าสุด
"""

from enum import Enum
from typing import Optional
import time


class ExpiryStrategy(str, Enum):
    FIXED_TTL = "fixed_ttl"          # TTL คงที่
    SLIDING = "sliding"               # ต่ออายุทุกครั้งที่ access
    LAZY = "lazy"                     # ไม่มี TTL แต่ check validity
    STALE_WHILE_REVALIDATE = "swr"   # Return stale data แต่ refresh ใน background


class CacheConfig:
    """Configuration สำหรับ cache แต่ละประเภท"""
    
    CONFIGS = {
        # Real-time data - TTL สั้นมาก
        'interface_stats': {
            'strategy': ExpiryStrategy.FIXED_TTL,
            'ttl': 5,
            'max_stale': 15
        },
        'system_resources': {
            'strategy': ExpiryStrategy.FIXED_TTL,
            'ttl': 10,
            'max_stale': 30
        },
        
        # Semi-static data - TTL ปานกลาง
        'interfaces': {
            'strategy': ExpiryStrategy.FIXED_TTL,
            'ttl': 60,
            'max_stale': 300
        },
        'routing_table': {
            'strategy': ExpiryStrategy.FIXED_TTL,
            'ttl': 120,
            'max_stale': 600
        },
        
        # Static data - TTL ยาว
        'firewall_rules': {
            'strategy': ExpiryStrategy.STALE_WHILE_REVALIDATE,
            'ttl': 300,
            'max_stale': 3600
        },
        'service_plans': {
            'strategy': ExpiryStrategy.FIXED_TTL,
            'ttl': 3600,
            'max_stale': 86400
        },
        
        # User sessions - Sliding window
        'user_session': {
            'strategy': ExpiryStrategy.SLIDING,
            'ttl': 3600,
            'max_idle': 1800
        },
    }


class StaleWhileRevalidateCache:
    """SWR Cache - Return stale data แต่ refresh ใน background"""
    
    def __init__(self, redis_client, revalidator_func):
        self.redis = redis_client
        self.revalidator = revalidator_func
    
    def get(self, key: str, ttl: int, max_stale: int):
        """
        Get data:
        - ถ้า fresh: return data
        - ถ้า stale แต่ยัง valid: return data + trigger background refresh
        - ถ้า expired: fetch fresh data
        """
        data_key = f"swr:{key}:data"
        expires_key = f"swr:{key}:expires"
        
        data = self.redis.get_json(data_key)
        expires_at = self.redis.get(expires_key)
        
        if data is None:
            # Cache miss - fetch และ store
            fresh_data = self.revalidator(key)
            self.set(key, fresh_data, ttl, max_stale)
            return fresh_data
        
        now = time.time()
        is_stale = expires_at and float(expires_at) < now
        
        if is_stale:
            # Stale data - return แต่ refresh ใน background
            import threading
            def refresh():
                try:
                    fresh = self.revalidator(key)
                    self.set(key, fresh, ttl, max_stale)
                except Exception:
                    pass
            
            threading.Thread(target=refresh, daemon=True).start()
        
        return data
    
    def set(self, key: str, data, ttl: int, max_stale: int):
        """Store data พร้อม expiry information"""
        data_key = f"swr:{key}:data"
        expires_key = f"swr:{key}:expires"
        
        expires_at = time.time() + ttl
        
        self.redis.set(data_key, data, ttl=ttl + max_stale)
        self.redis.set(expires_key, str(expires_at), ttl=ttl + max_stale)
```

---

## 8. Redis Cluster {#cluster}

```python
# app/services/redis_cluster.py
from redis.cluster import RedisCluster
from redis.cluster import ClusterNode
import logging

logger = logging.getLogger(__name__)


class RedisClusterClient:
    """Redis Cluster client สำหรับ high availability"""
    
    def __init__(self, startup_nodes: list):
        """
        startup_nodes: list of {'host': ..., 'port': ...}
        """
        nodes = [ClusterNode(n['host'], n['port']) for n in startup_nodes]
        
        self.client = RedisCluster(
            startup_nodes=nodes,
            decode_responses=True,
            skip_full_coverage_check=True,
            socket_timeout=5,
            socket_connect_timeout=5,
            retry_on_timeout=True,
            max_connections_per_node=20
        )
    
    def get(self, key: str):
        return self.client.get(key)
    
    def set(self, key: str, value, ttl: int = None):
        if ttl:
            return self.client.setex(key, ttl, value)
        return self.client.set(key, value)
    
    def cluster_info(self):
        """ดู cluster status"""
        return self.client.cluster_info()
    
    def cluster_nodes(self):
        """ดู nodes ทั้งหมด"""
        return self.client.cluster_nodes()


# docker-compose.yml สำหรับ Redis Cluster
REDIS_CLUSTER_COMPOSE = """
version: '3.8'
services:
  redis-1:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf
             --cluster-node-timeout 5000 --appendonly yes --port 6379
    ports: ["6379:6379"]
  
  redis-2:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf
             --cluster-node-timeout 5000 --appendonly yes --port 6379
    ports: ["6380:6379"]
  
  redis-3:
    image: redis:7-alpine
    command: redis-server --cluster-enabled yes --cluster-config-file nodes.conf
             --cluster-node-timeout 5000 --appendonly yes --port 6379
    ports: ["6381:6379"]
  
  redis-cluster-init:
    image: redis:7-alpine
    command: >
      sh -c "redis-cli --cluster create
      redis-1:6379 redis-2:6379 redis-3:6379
      --cluster-replicas 0 --cluster-yes"
    depends_on: [redis-1, redis-2, redis-3]
"""
```

---

## 9. Performance Optimization {#performance}

```python
# app/services/cache_warming.py
"""Cache Warming - โหลดข้อมูลเข้า cache ตอน startup"""
import asyncio
import logging
from typing import List

logger = logging.getLogger(__name__)


class CacheWarmer:
    """Warm up cache ตอน application startup"""
    
    def __init__(self, router_manager, cache_service):
        self.router_manager = router_manager
        self.cache = cache_service
    
    async def warm_all_routers(self, router_ids: List[str]):
        """Warm cache สำหรับทุก router"""
        tasks = [self.warm_router(router_id) for router_id in router_ids]
        results = await asyncio.gather(*tasks, return_exceptions=True)
        
        success = sum(1 for r in results if not isinstance(r, Exception))
        logger.info(f"Cache warming complete: {success}/{len(router_ids)} routers")
    
    async def warm_router(self, router_id: str):
        """Warm cache สำหรับ router เดียว"""
        try:
            loop = asyncio.get_event_loop()
            
            # Run synchronous operations ใน thread pool
            with self.router_manager.get_connection(router_id) as conn:
                interfaces = await loop.run_in_executor(None, conn.get_interfaces)
                resources = await loop.run_in_executor(None, conn.get_system_resources)
                firewall = await loop.run_in_executor(None, conn.get_firewall_rules)
            
            # Store ใน cache
            self.cache.set_interfaces(router_id, interfaces)
            self.cache.set_system_resources(router_id, resources)
            self.cache.set_firewall_rules(router_id, firewall)
            
            logger.info(f"Cache warmed for router: {router_id}")
        except Exception as e:
            logger.warning(f"Failed to warm cache for {router_id}: {e}")
            raise


# Redis Pipeline สำหรับ batch operations
class BatchCacheOperations:
    """ใช้ Pipeline เพื่อ batch Redis operations"""
    
    def __init__(self, redis_client):
        self.redis = redis_client
    
    def batch_set(self, items: dict, ttl: int = 300):
        """Set หลาย keys ในครั้งเดียว"""
        import json
        
        pipe = self.redis.sync_client.pipeline(transaction=False)
        
        for key, value in items.items():
            serialized = json.dumps(value) if not isinstance(value, str) else value
            pipe.setex(key, ttl, serialized)
        
        return pipe.execute()
    
    def batch_get(self, keys: list) -> dict:
        """Get หลาย keys ในครั้งเดียว"""
        import json
        
        pipe = self.redis.sync_client.pipeline(transaction=False)
        for key in keys:
            pipe.get(key)
        
        results = pipe.execute()
        
        return {
            key: json.loads(value) if value else None
            for key, value in zip(keys, results)
        }
```

---

## 10. Lab: Cached Network Dashboard {#lab}

```python
# tests/test_cache_performance.py
import time
import pytest
from app.services.cache_service import RouterDataCache


def test_cache_vs_direct_fetch():
    """เปรียบเทียบ performance: cache vs direct fetch"""
    cache = RouterDataCache()
    router_id = "test-router-1"
    
    # Simulate data
    mock_interfaces = [
        {"name": f"ether{i}", "type": "ether", "disabled": False}
        for i in range(1, 10)
    ]
    
    # Direct fetch simulation
    start = time.time()
    for _ in range(100):
        # Simulated router API call (10ms latency)
        time.sleep(0.01)
    direct_time = time.time() - start
    
    # Cache operations
    cache.set_interfaces(router_id, mock_interfaces)
    
    start = time.time()
    for _ in range(100):
        data = cache.get_interfaces(router_id)
    cache_time = time.time() - start
    
    improvement = direct_time / cache_time
    print(f"Direct: {direct_time:.3f}s | Cache: {cache_time:.3f}s | Improvement: {improvement:.0f}x")
    
    assert improvement > 100  # Cache ควรเร็วกว่า 100x
```

```bash
# Lab Setup Steps

# 1. Start Redis
docker run -d --name mikrotik-redis \
    -p 6379:6379 \
    -v redis_data:/data \
    redis:7-alpine redis-server --appendonly yes

# 2. Test connection
redis-cli ping
# Expected: PONG

# 3. Monitor Redis
redis-cli monitor

# 4. Check stats
redis-cli info stats | grep keyspace
redis-cli info memory | grep used_memory_human
```

### Verification Checklist

- [ ] Redis เชื่อมต่อได้
- [ ] Cache HIT/MISS logging ทำงาน
- [ ] Rate limiting blocks เมื่อ exceed limit
- [ ] Session management สร้างและ invalidate ได้
- [ ] Pub/Sub delivery ทำงาน
- [ ] Cache warming เสร็จใน startup
- [ ] Memory usage อยู่ในขอบเขตที่กำหนด
- [ ] Eviction policy ทำงานถูกต้อง

> **Tip:** ใช้ `redis-cli --latency` เพื่อ measure Redis latency

> **Warning:** อย่าใช้ KEYS command ใน production เพราะทำให้ Redis block

---

## Summary

Part นี้ครอบคลุม:
- **Caching strategies** สำหรับ router data
- **Session management** ด้วย Redis
- **Rate limiting** แบบ sliding window
- **Pub/Sub** สำหรับ real-time events
- **Data expiry** strategies หลากหลายรูปแบบ
- **Redis Cluster** สำหรับ high availability

---

[← Part 63: Database Integration](part-063-database-integration.md) | [Part 65: Docker Deployment →](part-065-docker-deployment.md)
