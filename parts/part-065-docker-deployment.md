# Part 65: Docker Deployment สำหรับ Network Applications

## สารบัญ
1. [Docker Fundamentals](#fundamentals)
2. [Dockerfile for MikroTik Apps](#dockerfile)
3. [Docker Compose Setup](#compose)
4. [Service Dependencies](#dependencies)
5. [Volume Management](#volumes)
6. [Network Configuration](#networking)
7. [Production Deployment](#production)
8. [Container Monitoring](#monitoring)
9. [CI/CD Integration](#cicd)
10. [Lab: Dockerized ISP Platform](#lab)

---

## 1. Docker Fundamentals {#fundamentals}

### ทำไมต้องใช้ Docker สำหรับ Network Apps

| Feature | Without Docker | With Docker |
|---------|---------------|-------------|
| Deployment | Manual, error-prone | Reproducible |
| Dependencies | Conflict possible | Isolated |
| Scaling | Hard | `docker-compose scale` |
| Rollback | Difficult | Image tags |
| Dev/Prod parity | Different | Same image |

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     Docker Host                               │
│                                                               │
│  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐   │
│  │  Nginx   │  │   API    │  │ Workers  │  │ Cron     │   │
│  │ (Proxy)  │  │(FastAPI) │  │(Celery)  │  │ (Jobs)   │   │
│  │ :80/:443 │  │  :8000   │  │          │  │          │   │
│  └────┬─────┘  └────┬─────┘  └────┬─────┘  └────┬─────┘   │
│       │              │              │              │          │
│  ┌────▼──────────────▼──────────────▼──────────────▼─────┐  │
│  │                    mikrotik_network                     │  │
│  └──────┬──────────────────────────────────────────┬──────┘  │
│         │                                          │          │
│  ┌──────▼──────┐                        ┌─────────▼──────┐   │
│  │ PostgreSQL  │                        │     Redis      │   │
│  │  :5432      │                        │    :6379       │   │
│  └─────────────┘                        └────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

---

## 2. Dockerfile for MikroTik Apps {#dockerfile}

### Multi-stage Dockerfile (Python/FastAPI)

```dockerfile
# Dockerfile.api
# ============================================
# Stage 1: Builder
# ============================================
FROM python:3.11-slim as builder

# ติดตั้ง build dependencies
RUN apt-get update && apt-get install -y \
    gcc \
    g++ \
    libpq-dev \
    && rm -rf /var/lib/apt/lists/*

WORKDIR /build

# Copy และติดตั้ง Python dependencies
COPY requirements.txt .
RUN pip install --user --no-cache-dir -r requirements.txt

# ============================================
# Stage 2: Production
# ============================================
FROM python:3.11-slim as production

# ติดตั้ง runtime dependencies เท่านั้น
RUN apt-get update && apt-get install -y \
    libpq5 \
    curl \
    && rm -rf /var/lib/apt/lists/*

# Security: สร้าง non-root user
RUN groupadd -r appgroup && useradd -r -g appgroup -u 1000 appuser

WORKDIR /app

# Copy installed packages จาก builder
COPY --from=builder /root/.local /home/appuser/.local

# Copy application code
COPY --chown=appuser:appgroup . .

# ไม่เก็บ sensitive files ใน image
RUN rm -f .env .env.local tests/ -rf

# Switch to non-root user
USER appuser

# เพิ่ม local bin to PATH
ENV PATH=/home/appuser/.local/bin:$PATH
ENV PYTHONDONTWRITEBYTECODE=1
ENV PYTHONUNBUFFERED=1

# Health check
HEALTHCHECK --interval=30s --timeout=10s --start-period=40s --retries=3 \
    CMD curl -f http://localhost:8000/health || exit 1

EXPOSE 8000

# Run application
CMD ["uvicorn", "app.main:app", \
     "--host", "0.0.0.0", \
     "--port", "8000", \
     "--workers", "4", \
     "--worker-class", "uvicorn.workers.UvicornWorker"]
```

### Dockerfile สำหรับ Node.js WebSocket

```dockerfile
# Dockerfile.websocket
FROM node:20-alpine as builder

WORKDIR /build

# Copy package files
COPY package*.json ./
RUN npm ci --only=production

# ============================================
FROM node:20-alpine as production

# Security updates
RUN apk upgrade --no-cache

# Non-root user
RUN addgroup -S appgroup && adduser -S appuser -G appgroup

WORKDIR /app

# Copy node_modules จาก builder
COPY --from=builder /build/node_modules ./node_modules

# Copy source
COPY --chown=appuser:appgroup . .
RUN rm -rf tests/ .env

USER appuser

HEALTHCHECK --interval=30s --timeout=10s --retries=3 \
    CMD node -e "require('http').get('http://localhost:3001/health', r => r.statusCode === 200 ? process.exit(0) : process.exit(1))"

EXPOSE 3001

CMD ["node", "dist/server.js"]
```

### Dockerfile สำหรับ Nginx

```dockerfile
# Dockerfile.nginx
FROM nginx:1.25-alpine

# ลบ default config
RUN rm /etc/nginx/conf.d/default.conf

# Copy custom config
COPY nginx/nginx.conf /etc/nginx/nginx.conf
COPY nginx/conf.d/ /etc/nginx/conf.d/

# Copy SSL certificates (ในกรณีที่ embed ใน image)
# ใน production ควรใช้ volumes แทน
# COPY ssl/ /etc/nginx/ssl/

# Test nginx config
RUN nginx -t

# Non-root user
RUN chown -R nginx:nginx /var/cache/nginx /var/run /var/log/nginx

EXPOSE 80 443

CMD ["nginx", "-g", "daemon off;"]
```

---

## 3. Docker Compose Setup {#compose}

### Development Configuration

```yaml
# docker-compose.yml
version: '3.9'

# ============================================
# Networks
# ============================================
networks:
  mikrotik_network:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/16

# ============================================
# Volumes
# ============================================
volumes:
  postgres_data:
    driver: local
  redis_data:
    driver: local
  nginx_logs:
    driver: local
  api_logs:
    driver: local

# ============================================
# Services
# ============================================
services:

  # Database
  postgres:
    image: postgres:15-alpine
    container_name: mikrotik_postgres
    environment:
      POSTGRES_DB: ${DB_NAME:-mikrotik_db}
      POSTGRES_USER: ${DB_USER:-postgres}
      POSTGRES_PASSWORD: ${DB_PASSWORD:?DB_PASSWORD required}
      POSTGRES_INITDB_ARGS: "--encoding=UTF-8 --lc-collate=C --lc-ctype=C"
      PGDATA: /var/lib/postgresql/data/pgdata
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./database/init:/docker-entrypoint-initdb.d:ro
    networks:
      - mikrotik_network
    ports:
      - "${DB_PORT:-5432}:5432"
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER:-postgres} -d ${DB_NAME:-mikrotik_db}"]
      interval: 10s
      timeout: 5s
      retries: 5
      start_period: 30s
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 2G
        reservations:
          cpus: '0.5'
          memory: 512M

  # Redis
  redis:
    image: redis:7-alpine
    container_name: mikrotik_redis
    command: >
      redis-server
      --maxmemory 512mb
      --maxmemory-policy allkeys-lru
      --appendonly yes
      --requirepass ${REDIS_PASSWORD:-}
    volumes:
      - redis_data:/data
      - ./redis/redis.conf:/usr/local/etc/redis/redis.conf:ro
    networks:
      - mikrotik_network
    ports:
      - "${REDIS_PORT:-6379}:6379"
    healthcheck:
      test: ["CMD", "redis-cli", "ping"]
      interval: 10s
      timeout: 3s
      retries: 3
    restart: unless-stopped

  # FastAPI Backend
  api:
    build:
      context: ./backend
      dockerfile: Dockerfile.api
      target: production
    container_name: mikrotik_api
    environment:
      - DATABASE_URL=postgresql://${DB_USER:-postgres}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-mikrotik_db}
      - REDIS_URL=redis://:${REDIS_PASSWORD:-}@redis:6379
      - SECRET_KEY=${SECRET_KEY:?SECRET_KEY required}
      - DEBUG=${DEBUG:-false}
      - ALLOWED_ORIGINS=${ALLOWED_ORIGINS:-http://localhost:3000}
    volumes:
      - api_logs:/app/logs
      - ./backend:/app:ro  # Development only - remove in production
    networks:
      - mikrotik_network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped
    deploy:
      replicas: 2
      resources:
        limits:
          cpus: '1'
          memory: 512M

  # WebSocket Service
  websocket:
    build:
      context: ./websocket
      dockerfile: Dockerfile.websocket
      target: production
    container_name: mikrotik_ws
    environment:
      - REDIS_URL=redis://:${REDIS_PASSWORD:-}@redis:6379
      - NODE_ENV=production
      - PORT=3001
    networks:
      - mikrotik_network
    depends_on:
      redis:
        condition: service_healthy
    restart: unless-stopped

  # Celery Worker (Background tasks)
  worker:
    build:
      context: ./backend
      dockerfile: Dockerfile.api
      target: production
    container_name: mikrotik_worker
    command: celery -A app.celery worker --loglevel=info --concurrency=4
    environment:
      - DATABASE_URL=postgresql://${DB_USER:-postgres}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-mikrotik_db}
      - REDIS_URL=redis://:${REDIS_PASSWORD:-}@redis:6379
      - SECRET_KEY=${SECRET_KEY}
    volumes:
      - api_logs:/app/logs
    networks:
      - mikrotik_network
    depends_on:
      postgres:
        condition: service_healthy
      redis:
        condition: service_healthy
    restart: unless-stopped

  # Celery Beat (Scheduled tasks)
  beat:
    build:
      context: ./backend
      dockerfile: Dockerfile.api
      target: production
    container_name: mikrotik_beat
    command: celery -A app.celery beat --loglevel=info --scheduler django_celery_beat.schedulers:DatabaseScheduler
    environment:
      - DATABASE_URL=postgresql://${DB_USER:-postgres}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-mikrotik_db}
      - REDIS_URL=redis://:${REDIS_PASSWORD:-}@redis:6379
    networks:
      - mikrotik_network
    depends_on:
      - worker
    restart: unless-stopped

  # Nginx Reverse Proxy
  nginx:
    build:
      context: ./nginx
      dockerfile: Dockerfile.nginx
    container_name: mikrotik_nginx
    volumes:
      - nginx_logs:/var/log/nginx
      - ./ssl:/etc/nginx/ssl:ro
      - ./static:/usr/share/nginx/html:ro
    networks:
      - mikrotik_network
    ports:
      - "80:80"
      - "443:443"
    depends_on:
      - api
      - websocket
    restart: unless-stopped
    deploy:
      resources:
        limits:
          cpus: '0.5'
          memory: 256M

  # Database migration (one-shot)
  migrate:
    build:
      context: ./backend
      dockerfile: Dockerfile.api
    command: alembic upgrade head
    environment:
      - DATABASE_URL=postgresql://${DB_USER:-postgres}:${DB_PASSWORD}@postgres:5432/${DB_NAME:-mikrotik_db}
    networks:
      - mikrotik_network
    depends_on:
      postgres:
        condition: service_healthy
    profiles:
      - migration
```

### Production Overrides

```yaml
# docker-compose.prod.yml
version: '3.9'

services:
  api:
    build:
      target: production
    volumes: []  # ไม่ mount source code ใน production
    environment:
      - DEBUG=false
    deploy:
      replicas: 3
      update_config:
        parallelism: 1
        delay: 10s
        failure_action: rollback
      restart_policy:
        condition: on-failure
        delay: 5s
        max_attempts: 3

  postgres:
    volumes:
      - postgres_data:/var/lib/postgresql/data
    # ไม่ expose port ใน production
    ports: []

  redis:
    ports: []  # ไม่ expose Redis ใน production
```

---

## 4. Service Dependencies {#dependencies}

```python
# app/startup.py
"""Application startup checks"""
import asyncio
import logging
import time
from typing import Callable, List

logger = logging.getLogger(__name__)


async def wait_for_service(
    check_func: Callable,
    service_name: str,
    max_retries: int = 30,
    retry_delay: float = 2.0
):
    """รอ service พร้อมใช้งาน"""
    for attempt in range(1, max_retries + 1):
        try:
            result = await check_func()
            if result:
                logger.info(f"✓ {service_name} is ready")
                return True
        except Exception as e:
            logger.warning(f"Attempt {attempt}/{max_retries}: {service_name} not ready: {e}")
        
        if attempt < max_retries:
            await asyncio.sleep(retry_delay)
    
    raise RuntimeError(f"Service {service_name} failed to start after {max_retries} attempts")


async def check_database():
    """ตรวจสอบว่า database พร้อม"""
    from app.database import check_database_connection
    return check_database_connection()


async def check_redis():
    """ตรวจสอบว่า Redis พร้อม"""
    from app.services.redis_client import redis_client
    return redis_client.ping()


async def run_migrations():
    """รัน database migrations"""
    import subprocess
    result = subprocess.run(
        ['alembic', 'upgrade', 'head'],
        capture_output=True,
        text=True
    )
    if result.returncode != 0:
        raise RuntimeError(f"Migration failed: {result.stderr}")
    logger.info("Database migrations completed")


async def startup_sequence():
    """Startup sequence ตามลำดับ"""
    logger.info("Starting application...")
    
    # รอ dependencies
    await wait_for_service(check_database, "PostgreSQL")
    await wait_for_service(check_redis, "Redis")
    
    # Run migrations
    await run_migrations()
    
    # Initialize services
    from app.services.cache_warming import CacheWarmer
    from app.services.mikrotik import router_manager
    # warmer = CacheWarmer(router_manager, cache)
    # await warmer.warm_all_routers(router_ids)
    
    logger.info("Application started successfully")
```

---

## 5. Volume Management {#volumes}

```bash
# Volume management commands

# ดู volumes ทั้งหมด
docker volume ls

# ดูรายละเอียด volume
docker volume inspect mikrotik_postgres_data

# Backup PostgreSQL data
docker run --rm \
    -v mikrotik_postgres_data:/data \
    -v $(pwd)/backups:/backup \
    alpine tar czf /backup/postgres_backup_$(date +%Y%m%d).tar.gz -C /data .

# Restore PostgreSQL data
docker run --rm \
    -v mikrotik_postgres_data:/data \
    -v $(pwd)/backups:/backup \
    alpine tar xzf /backup/postgres_backup_20240101.tar.gz -C /data

# ลบ volumes ที่ไม่ใช้
docker volume prune

# ย้ายข้อมูลระหว่าง volumes
docker run --rm \
    -v old_volume:/from \
    -v new_volume:/to \
    alpine cp -av /from/. /to/
```

### Docker Volume Configuration

```yaml
# volumes configuration ใน docker-compose
volumes:
  postgres_data:
    driver: local
    driver_opts:
      type: none
      o: bind
      device: /data/mikrotik/postgres  # bind mount ไปยัง host path
  
  redis_data:
    driver: local
    
  # NFS Volume สำหรับ shared storage ใน cluster
  shared_logs:
    driver: local
    driver_opts:
      type: nfs
      o: addr=nfs-server.local,rw
      device: ":/exports/mikrotik-logs"
```

---

## 6. Network Configuration {#networking}

```yaml
# docker-compose networks
networks:
  # Frontend network - exposed to internet
  frontend:
    driver: bridge
    ipam:
      config:
        - subnet: 172.20.0.0/24
  
  # Backend network - internal only
  backend:
    driver: bridge
    internal: true  # ไม่มี internet access
    ipam:
      config:
        - subnet: 172.20.1.0/24
  
  # Database network - very restricted
  database:
    driver: bridge
    internal: true
    ipam:
      config:
        - subnet: 172.20.2.0/24

services:
  nginx:
    networks:
      - frontend
      - backend
  
  api:
    networks:
      - backend
      - database
  
  postgres:
    networks:
      - database  # Only accessible from backend
  
  redis:
    networks:
      - backend
```

---

## 7. Production Deployment {#production}

### Production Deployment Script

```bash
#!/bin/bash
# deploy.sh - Production deployment script

set -e  # Exit on error

COMPOSE_FILE="docker-compose.yml -f docker-compose.prod.yml"
APP_NAME="mikrotik"

echo "=== MikroTik Platform Deployment ==="
echo "Date: $(date)"

# ตรวจสอบ prerequisites
command -v docker >/dev/null 2>&1 || { echo "Docker not found"; exit 1; }
command -v docker-compose >/dev/null 2>&1 || { echo "docker-compose not found"; exit 1; }

# ตรวจสอบ environment file
if [ ! -f ".env" ]; then
    echo "ERROR: .env file not found"
    exit 1
fi

# Pull latest images
echo "Pulling latest images..."
docker-compose -f $COMPOSE_FILE pull

# Build application images
echo "Building images..."
docker-compose -f $COMPOSE_FILE build --no-cache api websocket nginx

# Run database migrations
echo "Running migrations..."
docker-compose -f $COMPOSE_FILE run --rm migrate

# Deploy with zero-downtime (rolling update)
echo "Deploying API (rolling update)..."
docker-compose -f $COMPOSE_FILE up -d --no-deps --scale api=3 api

# Wait for new containers to be healthy
echo "Waiting for health checks..."
sleep 30

# Check if all containers are healthy
UNHEALTHY=$(docker-compose -f $COMPOSE_FILE ps | grep "unhealthy" | wc -l)
if [ "$UNHEALTHY" -gt "0" ]; then
    echo "ERROR: Some containers are unhealthy!"
    docker-compose -f $COMPOSE_FILE ps
    exit 1
fi

# Deploy other services
echo "Deploying remaining services..."
docker-compose -f $COMPOSE_FILE up -d nginx websocket worker beat

echo "=== Deployment Complete ==="
docker-compose -f $COMPOSE_FILE ps
```

### Environment File

```bash
# .env.example - Copy เป็น .env และแก้ไขค่า
# Application
APP_ENV=production
DEBUG=false
SECRET_KEY=your-very-long-random-secret-key-here

# Database
DB_NAME=mikrotik_db
DB_USER=mikrotik_user
DB_PASSWORD=very-strong-database-password
DB_HOST=postgres
DB_PORT=5432

# Redis
REDIS_URL=redis://:redis-password@redis:6379
REDIS_PASSWORD=redis-password

# API
ALLOWED_ORIGINS=https://admin.yourdomain.com,https://www.yourdomain.com

# Email
SMTP_HOST=smtp.gmail.com
SMTP_PORT=587
SMTP_USER=noreply@yourdomain.com
SMTP_PASSWORD=smtp-password

# MikroTik defaults
MIKROTIK_API_TIMEOUT=10
MIKROTIK_MAX_CONNECTIONS=20

# Storage
AWS_S3_BUCKET=mikrotik-backups
AWS_ACCESS_KEY_ID=your-key
AWS_SECRET_ACCESS_KEY=your-secret
```

---

## 8. Container Monitoring {#monitoring}

```yaml
# docker-compose.monitoring.yml
version: '3.9'

services:
  # Prometheus สำหรับ metrics collection
  prometheus:
    image: prom/prometheus:latest
    container_name: prometheus
    volumes:
      - ./monitoring/prometheus.yml:/etc/prometheus/prometheus.yml
      - prometheus_data:/prometheus
    command:
      - '--config.file=/etc/prometheus/prometheus.yml'
      - '--storage.tsdb.path=/prometheus'
      - '--storage.tsdb.retention.time=30d'
    ports:
      - "9090:9090"
    networks:
      - monitoring
    restart: unless-stopped

  # Grafana สำหรับ visualization
  grafana:
    image: grafana/grafana:latest
    container_name: grafana
    environment:
      - GF_SECURITY_ADMIN_PASSWORD=${GRAFANA_PASSWORD:-admin}
      - GF_USERS_ALLOW_SIGN_UP=false
    volumes:
      - grafana_data:/var/lib/grafana
      - ./monitoring/grafana/dashboards:/etc/grafana/provisioning/dashboards
    ports:
      - "3000:3000"
    networks:
      - monitoring
    restart: unless-stopped

  # cAdvisor สำหรับ container metrics
  cadvisor:
    image: gcr.io/cadvisor/cadvisor:latest
    container_name: cadvisor
    volumes:
      - /:/rootfs:ro
      - /var/run:/var/run:ro
      - /sys:/sys:ro
      - /var/lib/docker/:/var/lib/docker:ro
    ports:
      - "8080:8080"
    networks:
      - monitoring
    restart: unless-stopped

  # Node Exporter สำหรับ host metrics
  node-exporter:
    image: prom/node-exporter:latest
    container_name: node_exporter
    volumes:
      - /proc:/host/proc:ro
      - /sys:/host/sys:ro
      - /:/rootfs:ro
    command:
      - '--path.procfs=/host/proc'
      - '--path.rootfs=/rootfs'
      - '--path.sysfs=/host/sys'
    ports:
      - "9100:9100"
    networks:
      - monitoring
    restart: unless-stopped

networks:
  monitoring:
    driver: bridge

volumes:
  prometheus_data:
  grafana_data:
```

```yaml
# monitoring/prometheus.yml
global:
  scrape_interval: 15s
  evaluation_interval: 15s

alerting:
  alertmanagers:
    - static_configs:
        - targets: ['alertmanager:9093']

rule_files:
  - "alert_rules.yml"

scrape_configs:
  - job_name: 'mikrotik_api'
    static_configs:
      - targets: ['api:8000']
    metrics_path: '/metrics'
  
  - job_name: 'cadvisor'
    static_configs:
      - targets: ['cadvisor:8080']
  
  - job_name: 'node'
    static_configs:
      - targets: ['node-exporter:9100']
  
  - job_name: 'postgres'
    static_configs:
      - targets: ['postgres-exporter:9187']
  
  - job_name: 'redis'
    static_configs:
      - targets: ['redis-exporter:9121']
```

---

## 9. CI/CD Integration {#cicd}

```yaml
# .github/workflows/deploy.yml
name: Deploy MikroTik Platform

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

env:
  REGISTRY: ghcr.io
  IMAGE_NAME: ${{ github.repository }}

jobs:
  test:
    runs-on: ubuntu-latest
    services:
      postgres:
        image: postgres:15
        env:
          POSTGRES_PASSWORD: testpassword
          POSTGRES_DB: test_db
        options: >-
          --health-cmd pg_isready
          --health-interval 10s
          --health-timeout 5s
          --health-retries 5
      
      redis:
        image: redis:7
        options: >-
          --health-cmd "redis-cli ping"
          --health-interval 10s
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Set up Python
        uses: actions/setup-python@v4
        with:
          python-version: '3.11'
      
      - name: Install dependencies
        run: |
          pip install -r requirements.txt
          pip install pytest pytest-asyncio httpx
      
      - name: Run tests
        env:
          DATABASE_URL: postgresql://postgres:testpassword@localhost/test_db
          REDIS_URL: redis://localhost:6379
          SECRET_KEY: test-secret-key
        run: pytest tests/ -v --tb=short

  build-and-push:
    needs: test
    runs-on: ubuntu-latest
    if: github.ref == 'refs/heads/main'
    
    steps:
      - uses: actions/checkout@v4
      
      - name: Log in to Container Registry
        uses: docker/login-action@v3
        with:
          registry: ${{ env.REGISTRY }}
          username: ${{ github.actor }}
          password: ${{ secrets.GITHUB_TOKEN }}
      
      - name: Build and push Docker image
        uses: docker/build-push-action@v5
        with:
          context: ./backend
          file: ./backend/Dockerfile.api
          push: true
          tags: |
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:latest
            ${{ env.REGISTRY }}/${{ env.IMAGE_NAME }}:${{ github.sha }}
          cache-from: type=gha
          cache-to: type=gha,mode=max

  deploy:
    needs: build-and-push
    runs-on: ubuntu-latest
    environment: production
    
    steps:
      - name: Deploy to server
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ secrets.DEPLOY_HOST }}
          username: ${{ secrets.DEPLOY_USER }}
          key: ${{ secrets.DEPLOY_SSH_KEY }}
          script: |
            cd /opt/mikrotik
            docker-compose pull
            docker-compose up -d --no-deps api
            docker-compose ps
```

---

## 10. Lab: Dockerized ISP Platform {#lab}

### Lab Steps

```bash
# Step 1: Clone และ setup
git clone https://github.com/your-org/mikrotik-platform
cd mikrotik-platform

# Step 2: สร้าง .env file
cp .env.example .env
# แก้ไขค่าใน .env

# Step 3: Build images
docker-compose build

# Step 4: Start services
docker-compose up -d

# Step 5: Run migrations
docker-compose --profile migration up migrate

# Step 6: Check status
docker-compose ps
docker-compose logs api

# Step 7: Test endpoints
curl http://localhost/health
curl http://localhost/api/v1/auth/token \
    -d "username=admin&password=admin123"

# Step 8: Monitor
docker stats
docker-compose logs -f api
```

### Complete docker-compose.yml for Testing

```yaml
# docker-compose.lab.yml (ใช้สำหรับ lab/testing)
version: '3.9'

services:
  api:
    build: ./backend
    environment:
      - DATABASE_URL=postgresql://postgres:password@postgres/mikrotik_db
      - REDIS_URL=redis://redis:6379
      - SECRET_KEY=lab-secret-key-not-for-production
      - DEBUG=true
    ports:
      - "8000:8000"
    volumes:
      - ./backend:/app  # Hot reload
    depends_on:
      - postgres
      - redis

  postgres:
    image: postgres:15-alpine
    environment:
      - POSTGRES_DB=mikrotik_db
      - POSTGRES_PASSWORD=password
    ports:
      - "5432:5432"
    volumes:
      - ./data/postgres:/var/lib/postgresql/data

  redis:
    image: redis:7-alpine
    ports:
      - "6379:6379"

  adminer:
    image: adminer
    ports:
      - "8080:8080"
    depends_on:
      - postgres
```

### Verification Checklist

- [ ] All containers start successfully
- [ ] Health checks pass for all services
- [ ] API accessible at http://localhost/api/v1/
- [ ] Database migrations ran successfully
- [ ] Redis connection works
- [ ] WebSocket connects successfully
- [ ] Nginx proxy routes correctly
- [ ] Container logs show no errors
- [ ] Volume data persists after restart

> **Tip:** ใช้ `docker-compose logs -f service_name` เพื่อดู logs แบบ real-time

> **Warning:** ใน production ต้องใช้ secrets management ไม่ใช่ environment variables สำหรับข้อมูล sensitive

---

## Summary

Part นี้ครอบคลุม:
- **Multi-stage Dockerfiles** สำหรับ Python และ Node.js
- **Docker Compose** พร้อม production overrides
- **Service dependencies** และ health checks
- **Volume management** สำหรับ persistent data
- **Network isolation** ระหว่าง services
- **Container monitoring** ด้วย Prometheus/Grafana
- **CI/CD integration** ด้วย GitHub Actions

---

[← Part 64: Redis Caching](part-064-redis-caching.md) | [Part 66: Billing System →](part-066-billing-system.md)
