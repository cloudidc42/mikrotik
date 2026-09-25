# Part 62: WebSocket และ Real-time Data Streaming

## สารบัญ
1. [WebSocket Concepts](#concepts)
2. [Socket.io Setup](#socketio)
3. [Real-time Data Streaming](#streaming)
4. [Live Traffic Graphs](#traffic-graphs)
5. [Active Connections Display](#connections)
6. [Alert Broadcasting](#alerts)
7. [Multiple Router Streams](#multi-router)
8. [Client Reconnection](#reconnection)
9. [Production Scaling](#scaling)
10. [Lab: Real-time Dashboard](#lab)

---

## 1. WebSocket Concepts {#concepts}

WebSocket เป็น protocol ที่ช่วยให้มีการสื่อสารแบบ bidirectional ระหว่าง client และ server ซึ่งเหมาะมากสำหรับ real-time monitoring

### เปรียบเทียบ HTTP vs WebSocket

| Feature | HTTP Polling | Long Polling | WebSocket |
|---------|-------------|--------------|-----------|
| Latency | สูง (ขึ้นกับ interval) | ปานกลาง | ต่ำมาก (<1ms) |
| Server Load | สูง | ปานกลาง | ต่ำ |
| Connection | ใหม่ทุก request | ค้างไว้ | Persistent |
| Bandwidth | สูง (overhead) | ปานกลาง | ต่ำ |
| Real-time | ไม่แท้จริง | เกือบ | แท้จริง |

### WebSocket Flow

```
Client                          Server
  │                               │
  │──── HTTP Upgrade Request ────>│
  │                               │
  │<─── 101 Switching Protocols ──│
  │                               │
  │══════ WebSocket Connection ═══│ (Persistent)
  │                               │
  │──── Subscribe: traffic ──────>│
  │                               │
  │<─── Traffic Data (1s) ────────│
  │<─── Traffic Data (1s) ────────│
  │<─── Alert: High CPU ──────────│
  │<─── Traffic Data (1s) ────────│
  │                               │
  │──── Unsubscribe ─────────────>│
  │                               │
  │──── Close ───────────────────>│
  │<─── Close ────────────────────│
```

---

## 2. Socket.io Setup {#socketio}

### Backend: Node.js + Socket.io

```bash
# ติดตั้ง dependencies
mkdir mikrotik-realtime && cd mikrotik-realtime
npm init -y
npm install socket.io express node-routeros redis ioredis \
            winston cors dotenv compression
npm install -D typescript @types/node @types/express nodemon ts-node
```

```typescript
// src/server.ts
import express from 'express';
import { createServer } from 'http';
import { Server as SocketIOServer, Socket } from 'socket.io';
import { createAdapter } from '@socket.io/redis-adapter';
import { createClient } from 'redis';
import cors from 'cors';
import compression from 'compression';
import { logger } from './utils/logger';
import { RouterManager } from './services/RouterManager';
import { AuthMiddleware } from './middleware/auth';

const app = express();
const httpServer = createServer(app);

// Middleware
app.use(cors({ origin: process.env.ALLOWED_ORIGINS?.split(',') || '*' }));
app.use(compression());
app.use(express.json());

// Socket.IO setup
const io = new SocketIOServer(httpServer, {
    cors: {
        origin: process.env.ALLOWED_ORIGINS?.split(',') || '*',
        methods: ['GET', 'POST'],
        credentials: true
    },
    pingTimeout: 60000,
    pingInterval: 25000,
    transports: ['websocket', 'polling']
});

// Redis adapter สำหรับ horizontal scaling
async function setupRedisAdapter() {
    const pubClient = createClient({ url: process.env.REDIS_URL || 'redis://localhost:6379' });
    const subClient = pubClient.duplicate();
    
    await Promise.all([pubClient.connect(), subClient.connect()]);
    
    io.adapter(createAdapter(pubClient, subClient));
    logger.info('Redis adapter connected');
}

// Router manager
const routerManager = new RouterManager();

// Authentication middleware สำหรับ Socket.IO
io.use(async (socket: Socket, next) => {
    try {
        const token = socket.handshake.auth.token || 
                      socket.handshake.headers.authorization?.replace('Bearer ', '');
        
        if (!token) {
            throw new Error('Authentication required');
        }
        
        const user = await AuthMiddleware.validateToken(token);
        socket.data.user = user;
        next();
    } catch (error) {
        next(new Error('Authentication failed'));
    }
});

// Connection handling
io.on('connection', (socket: Socket) => {
    const user = socket.data.user;
    logger.info(`Client connected: ${socket.id} (user: ${user.username})`);
    
    // ส่ง welcome message
    socket.emit('connected', {
        socketId: socket.id,
        timestamp: Date.now(),
        message: 'Connected to MikroTik Real-time Service'
    });
    
    // Subscribe to router traffic
    socket.on('subscribe:traffic', async (data: { routerId: string, interval?: number }) => {
        await handleTrafficSubscription(socket, data);
    });
    
    // Subscribe to system resources
    socket.on('subscribe:resources', async (data: { routerId: string }) => {
        await handleResourceSubscription(socket, data);
    });
    
    // Subscribe to active connections
    socket.on('subscribe:connections', async (data: { routerId: string }) => {
        await handleConnectionsSubscription(socket, data);
    });
    
    // Subscribe to alerts
    socket.on('subscribe:alerts', (data: { routerIds?: string[] }) => {
        const roomName = data.routerIds 
            ? data.routerIds.map(id => `alerts:${id}`)
            : ['alerts:all'];
        
        roomName.forEach(room => socket.join(room));
        logger.info(`${socket.id} subscribed to alerts: ${roomName}`);
    });
    
    // Unsubscribe
    socket.on('unsubscribe', (data: { type: string, routerId?: string }) => {
        handleUnsubscribe(socket, data);
    });
    
    // Disconnect
    socket.on('disconnect', (reason) => {
        logger.info(`Client disconnected: ${socket.id} (reason: ${reason})`);
        // Cleanup subscriptions
        cleanupSubscriptions(socket.id);
    });
});

// Subscription tracking
const activeSubscriptions = new Map<string, NodeJS.Timeout[]>();

async function handleTrafficSubscription(
    socket: Socket, 
    data: { routerId: string, interval?: number }
) {
    const { routerId, interval = 1000 } = data;
    const minInterval = 1000; // 1 second minimum
    const actualInterval = Math.max(interval, minInterval);
    
    try {
        // ทดสอบว่า router ใช้งานได้
        const router = await routerManager.getRouter(routerId);
        if (!router) {
            socket.emit('error', { message: `Router ${routerId} not found` });
            return;
        }
        
        socket.join(`traffic:${routerId}`);
        
        const timer = setInterval(async () => {
            try {
                const stats = await routerManager.getTrafficStats(routerId);
                socket.emit('traffic:update', {
                    routerId,
                    timestamp: Date.now(),
                    data: stats
                });
            } catch (error) {
                socket.emit('error', { 
                    message: 'Failed to fetch traffic data',
                    routerId
                });
            }
        }, actualInterval);
        
        // เก็บ timer เพื่อ cleanup ทีหลัง
        const existing = activeSubscriptions.get(socket.id) || [];
        existing.push(timer);
        activeSubscriptions.set(socket.id, existing);
        
        socket.emit('subscribed', { 
            type: 'traffic', 
            routerId, 
            interval: actualInterval 
        });
        
    } catch (error) {
        socket.emit('error', { message: String(error) });
    }
}

async function handleResourceSubscription(socket: Socket, data: { routerId: string }) {
    const { routerId } = data;
    
    const timer = setInterval(async () => {
        try {
            const resources = await routerManager.getSystemResources(routerId);
            socket.emit('resources:update', {
                routerId,
                timestamp: Date.now(),
                data: resources
            });
        } catch (error) {
            socket.emit('error', { message: 'Failed to fetch resources' });
        }
    }, 5000); // Every 5 seconds
    
    const existing = activeSubscriptions.get(socket.id) || [];
    existing.push(timer);
    activeSubscriptions.set(socket.id, existing);
}

async function handleConnectionsSubscription(socket: Socket, data: { routerId: string }) {
    const { routerId } = data;
    
    const timer = setInterval(async () => {
        try {
            const connections = await routerManager.getActiveConnections(routerId);
            socket.emit('connections:update', {
                routerId,
                timestamp: Date.now(),
                count: connections.length,
                data: connections.slice(0, 100) // Limit to 100 connections
            });
        } catch (error) {
            socket.emit('error', { message: 'Failed to fetch connections' });
        }
    }, 2000);
    
    const existing = activeSubscriptions.get(socket.id) || [];
    existing.push(timer);
    activeSubscriptions.set(socket.id, existing);
}

function handleUnsubscribe(socket: Socket, data: { type: string, routerId?: string }) {
    // Leave room
    if (data.routerId) {
        socket.leave(`${data.type}:${data.routerId}`);
    }
    socket.emit('unsubscribed', data);
}

function cleanupSubscriptions(socketId: string) {
    const timers = activeSubscriptions.get(socketId);
    if (timers) {
        timers.forEach(timer => clearInterval(timer));
        activeSubscriptions.delete(socketId);
    }
}

// Start server
const PORT = process.env.PORT || 3001;
httpServer.listen(PORT, async () => {
    await setupRedisAdapter();
    logger.info(`WebSocket server running on port ${PORT}`);
});
```

### Router Manager Service

```typescript
// src/services/RouterManager.ts
import RouterOS from 'node-routeros';
import { EventEmitter } from 'events';
import { logger } from '../utils/logger';

interface RouterConfig {
    id: string;
    host: string;
    username: string;
    password: string;
    port: number;
}

interface TrafficStats {
    interface: string;
    rxBitsPerSecond: number;
    txBitsPerSecond: number;
    rxPacketsPerSecond: number;
    txPacketsPerSecond: number;
}

export class RouterManager extends EventEmitter {
    private routers: Map<string, RouterConfig> = new Map();
    private connections: Map<string, any> = new Map();
    private previousStats: Map<string, any> = new Map();
    
    addRouter(config: RouterConfig) {
        this.routers.set(config.id, config);
    }
    
    async getRouter(routerId: string) {
        return this.routers.get(routerId);
    }
    
    private async getConnection(routerId: string) {
        const config = this.routers.get(routerId);
        if (!config) throw new Error(`Router ${routerId} not found`);
        
        const conn = new RouterOS({
            host: config.host,
            user: config.username,
            password: config.password,
            port: config.port || 8728
        });
        
        await conn.connect();
        return conn;
    }
    
    async getTrafficStats(routerId: string): Promise<TrafficStats[]> {
        const conn = await this.getConnection(routerId);
        
        try {
            const result = await conn.write('/interface/print', [
                '=.proplist=name,rx-byte,tx-byte,rx-packet,tx-packet'
            ]);
            
            const currentStats = result.reduce((acc: any, item: any) => {
                acc[item.name] = item;
                return acc;
            }, {});
            
            const prevKey = `traffic:${routerId}`;
            const previousData = this.previousStats.get(prevKey) || {};
            
            const stats: TrafficStats[] = Object.entries(currentStats).map(([name, curr]: any) => {
                const prev = previousData[name] || curr;
                
                const rxBytes = parseInt(curr['rx-byte'] || '0') - parseInt(prev['rx-byte'] || '0');
                const txBytes = parseInt(curr['tx-byte'] || '0') - parseInt(prev['tx-byte'] || '0');
                const rxPackets = parseInt(curr['rx-packet'] || '0') - parseInt(prev['rx-packet'] || '0');
                const txPackets = parseInt(curr['tx-packet'] || '0') - parseInt(prev['tx-packet'] || '0');
                
                return {
                    interface: name,
                    rxBitsPerSecond: Math.max(0, rxBytes * 8),
                    txBitsPerSecond: Math.max(0, txBytes * 8),
                    rxPacketsPerSecond: Math.max(0, rxPackets),
                    txPacketsPerSecond: Math.max(0, txPackets)
                };
            });
            
            this.previousStats.set(prevKey, currentStats);
            
            return stats;
        } finally {
            conn.close();
        }
    }
    
    async getSystemResources(routerId: string) {
        const conn = await this.getConnection(routerId);
        
        try {
            const result = await conn.write('/system/resource/print');
            if (result.length > 0) {
                const r = result[0];
                return {
                    cpuLoad: parseInt(r['cpu-load'] || '0'),
                    memoryUsed: parseInt(r['total-memory'] || '0') - parseInt(r['free-memory'] || '0'),
                    totalMemory: parseInt(r['total-memory'] || '0'),
                    freeMemory: parseInt(r['free-memory'] || '0'),
                    uptime: r['uptime'],
                    version: r['version'],
                    boardName: r['board-name']
                };
            }
            return null;
        } finally {
            conn.close();
        }
    }
    
    async getActiveConnections(routerId: string) {
        const conn = await this.getConnection(routerId);
        
        try {
            const result = await conn.write('/ip/firewall/connection/print', [
                '=.proplist=protocol,src-address,dst-address,state,tcp-state'
            ]);
            
            return result.map((item: any) => ({
                protocol: item.protocol,
                srcAddress: item['src-address'],
                dstAddress: item['dst-address'],
                state: item.state || item['tcp-state'],
            }));
        } finally {
            conn.close();
        }
    }
}
```

---

## 3. Live Traffic Graphs {#traffic-graphs}

### Frontend: React + Chart.js

```typescript
// src/components/TrafficGraph.tsx
import React, { useEffect, useRef, useState, useCallback } from 'react';
import { Chart, registerables } from 'chart.js';
import { io, Socket } from 'socket.io-client';

Chart.register(...registerables);

interface TrafficData {
    interface: string;
    rxBitsPerSecond: number;
    txBitsPerSecond: number;
    timestamp: number;
}

interface TrafficGraphProps {
    routerId: string;
    interfaceName: string;
    maxPoints?: number;
    token: string;
}

const TrafficGraph: React.FC<TrafficGraphProps> = ({
    routerId,
    interfaceName,
    maxPoints = 60,
    token
}) => {
    const chartRef = useRef<HTMLCanvasElement>(null);
    const chartInstance = useRef<Chart | null>(null);
    const socketRef = useRef<Socket | null>(null);
    const [isConnected, setIsConnected] = useState(false);
    const [error, setError] = useState<string | null>(null);
    
    const labels = useRef<string[]>([]);
    const rxData = useRef<number[]>([]);
    const txData = useRef<number[]>([]);
    
    const formatBits = (bits: number): string => {
        if (bits >= 1e9) return `${(bits / 1e9).toFixed(1)} Gbps`;
        if (bits >= 1e6) return `${(bits / 1e6).toFixed(1)} Mbps`;
        if (bits >= 1e3) return `${(bits / 1e3).toFixed(1)} Kbps`;
        return `${bits} bps`;
    };
    
    const initChart = useCallback(() => {
        if (!chartRef.current) return;
        
        const ctx = chartRef.current.getContext('2d');
        if (!ctx) return;
        
        if (chartInstance.current) {
            chartInstance.current.destroy();
        }
        
        chartInstance.current = new Chart(ctx, {
            type: 'line',
            data: {
                labels: labels.current,
                datasets: [
                    {
                        label: 'RX (Download)',
                        data: rxData.current,
                        borderColor: 'rgb(34, 197, 94)',
                        backgroundColor: 'rgba(34, 197, 94, 0.1)',
                        fill: true,
                        tension: 0.4,
                        pointRadius: 0,
                        borderWidth: 2
                    },
                    {
                        label: 'TX (Upload)',
                        data: txData.current,
                        borderColor: 'rgb(59, 130, 246)',
                        backgroundColor: 'rgba(59, 130, 246, 0.1)',
                        fill: true,
                        tension: 0.4,
                        pointRadius: 0,
                        borderWidth: 2
                    }
                ]
            },
            options: {
                responsive: true,
                animation: false,
                scales: {
                    y: {
                        beginAtZero: true,
                        ticks: {
                            callback: (value) => formatBits(Number(value))
                        }
                    },
                    x: {
                        ticks: {
                            maxTicksLimit: 10
                        }
                    }
                },
                plugins: {
                    legend: { display: true },
                    tooltip: {
                        callbacks: {
                            label: (context) => {
                                return `${context.dataset.label}: ${formatBits(Number(context.raw))}`;
                            }
                        }
                    }
                }
            }
        });
    }, []);
    
    const updateChart = useCallback((rxBps: number, txBps: number) => {
        const now = new Date().toLocaleTimeString();
        
        labels.current.push(now);
        rxData.current.push(rxBps);
        txData.current.push(txBps);
        
        // Remove old data
        if (labels.current.length > maxPoints) {
            labels.current.shift();
            rxData.current.shift();
            txData.current.shift();
        }
        
        if (chartInstance.current) {
            chartInstance.current.update('none');
        }
    }, [maxPoints]);
    
    useEffect(() => {
        initChart();
        
        // Connect to WebSocket
        const socket = io(process.env.REACT_APP_WS_URL || 'http://localhost:3001', {
            auth: { token },
            transports: ['websocket'],
            reconnection: true,
            reconnectionDelay: 1000,
            reconnectionDelayMax: 5000,
            maxReconnectionAttempts: 10
        });
        
        socketRef.current = socket;
        
        socket.on('connect', () => {
            setIsConnected(true);
            setError(null);
            
            // Subscribe to traffic
            socket.emit('subscribe:traffic', {
                routerId,
                interval: 1000
            });
        });
        
        socket.on('traffic:update', (data: { routerId: string, data: TrafficData[] }) => {
            const interfaceData = data.data.find(d => d.interface === interfaceName);
            if (interfaceData) {
                updateChart(
                    interfaceData.rxBitsPerSecond,
                    interfaceData.txBitsPerSecond
                );
            }
        });
        
        socket.on('disconnect', () => {
            setIsConnected(false);
        });
        
        socket.on('connect_error', (err) => {
            setError(`Connection failed: ${err.message}`);
        });
        
        return () => {
            socket.emit('unsubscribe', { type: 'traffic', routerId });
            socket.disconnect();
            chartInstance.current?.destroy();
        };
    }, [routerId, interfaceName, token, initChart, updateChart]);
    
    return (
        <div className="traffic-graph">
            <div className="graph-header">
                <h3>Traffic: {interfaceName}</h3>
                <span className={`status ${isConnected ? 'online' : 'offline'}`}>
                    {isConnected ? '● Live' : '○ Disconnected'}
                </span>
            </div>
            {error && <div className="error-banner">{error}</div>}
            <canvas ref={chartRef} width={800} height={300} />
        </div>
    );
};

export default TrafficGraph;
```

---

## 4. Alert Broadcasting {#alerts}

```typescript
// src/services/AlertService.ts
import { Server as SocketIOServer } from 'socket.io';
import { logger } from '../utils/logger';

export enum AlertSeverity {
    INFO = 'info',
    WARNING = 'warning',
    CRITICAL = 'critical'
}

export interface Alert {
    id: string;
    routerId: string;
    severity: AlertSeverity;
    type: string;
    message: string;
    details?: any;
    timestamp: number;
}

export class AlertService {
    private io: SocketIOServer;
    
    constructor(io: SocketIOServer) {
        this.io = io;
    }
    
    async broadcast(alert: Alert) {
        // Broadcast to all subscribers
        this.io.to(`alerts:${alert.routerId}`).emit('alert', alert);
        this.io.to('alerts:all').emit('alert', alert);
        
        logger.warn(`Alert broadcast: [${alert.severity}] ${alert.message}`, {
            routerId: alert.routerId,
            type: alert.type
        });
    }
    
    async checkThresholds(routerId: string, metrics: any) {
        const alerts: Alert[] = [];
        
        // CPU threshold check
        if (metrics.cpuLoad > 90) {
            alerts.push({
                id: `cpu-${Date.now()}`,
                routerId,
                severity: AlertSeverity.CRITICAL,
                type: 'high_cpu',
                message: `CPU usage critical: ${metrics.cpuLoad}%`,
                details: { value: metrics.cpuLoad, threshold: 90 },
                timestamp: Date.now()
            });
        } else if (metrics.cpuLoad > 80) {
            alerts.push({
                id: `cpu-${Date.now()}`,
                routerId,
                severity: AlertSeverity.WARNING,
                type: 'high_cpu',
                message: `CPU usage high: ${metrics.cpuLoad}%`,
                details: { value: metrics.cpuLoad, threshold: 80 },
                timestamp: Date.now()
            });
        }
        
        // Memory threshold check
        const memoryPercent = ((metrics.totalMemory - metrics.freeMemory) / metrics.totalMemory) * 100;
        if (memoryPercent > 90) {
            alerts.push({
                id: `mem-${Date.now()}`,
                routerId,
                severity: AlertSeverity.CRITICAL,
                type: 'high_memory',
                message: `Memory usage critical: ${memoryPercent.toFixed(1)}%`,
                details: { value: memoryPercent, threshold: 90 },
                timestamp: Date.now()
            });
        }
        
        // Broadcast all alerts
        for (const alert of alerts) {
            await this.broadcast(alert);
        }
    }
}
```

---

## 5. Multiple Router Streams {#multi-router}

```typescript
// src/services/MultiRouterStream.ts
import { Server as SocketIOServer } from 'socket.io';
import { RouterManager } from './RouterManager';
import { AlertService } from './AlertService';

export class MultiRouterStream {
    private io: SocketIOServer;
    private routerManager: RouterManager;
    private alertService: AlertService;
    private streamIntervals: Map<string, NodeJS.Timeout> = new Map();
    
    constructor(io: SocketIOServer, routerManager: RouterManager) {
        this.io = io;
        this.routerManager = routerManager;
        this.alertService = new AlertService(io);
    }
    
    startRouterStream(routerId: string, interval: number = 1000) {
        if (this.streamIntervals.has(routerId)) {
            return; // Already streaming
        }
        
        const timer = setInterval(async () => {
            try {
                // Traffic data
                const trafficStats = await this.routerManager.getTrafficStats(routerId);
                this.io.to(`traffic:${routerId}`).emit('traffic:update', {
                    routerId,
                    timestamp: Date.now(),
                    data: trafficStats
                });
                
                // System resources (less frequent)
                const resources = await this.routerManager.getSystemResources(routerId);
                if (resources) {
                    this.io.to(`resources:${routerId}`).emit('resources:update', {
                        routerId,
                        timestamp: Date.now(),
                        data: resources
                    });
                    
                    // Check alert thresholds
                    await this.alertService.checkThresholds(routerId, resources);
                }
            } catch (error) {
                // Router offline - notify subscribers
                this.io.to(`traffic:${routerId}`).emit('router:offline', {
                    routerId,
                    timestamp: Date.now(),
                    message: 'Router connection lost'
                });
                
                this.alertService.broadcast({
                    id: `offline-${Date.now()}`,
                    routerId,
                    severity: 'critical' as any,
                    type: 'router_offline',
                    message: `Router ${routerId} is offline`,
                    timestamp: Date.now()
                });
            }
        }, interval);
        
        this.streamIntervals.set(routerId, timer);
    }
    
    stopRouterStream(routerId: string) {
        const timer = this.streamIntervals.get(routerId);
        if (timer) {
            clearInterval(timer);
            this.streamIntervals.delete(routerId);
        }
    }
    
    getActiveStreams(): string[] {
        return Array.from(this.streamIntervals.keys());
    }
}
```

---

## 6. Client Reconnection Handling {#reconnection}

```typescript
// src/hooks/useWebSocket.ts (React Hook)
import { useEffect, useRef, useState, useCallback } from 'react';
import { io, Socket } from 'socket.io-client';

interface UseWebSocketOptions {
    url: string;
    token: string;
    onConnect?: () => void;
    onDisconnect?: (reason: string) => void;
    onError?: (error: Error) => void;
    autoReconnect?: boolean;
    maxRetries?: number;
}

interface WebSocketState {
    isConnected: boolean;
    isReconnecting: boolean;
    reconnectAttempts: number;
    error: string | null;
}

export function useWebSocket(options: UseWebSocketOptions) {
    const {
        url,
        token,
        onConnect,
        onDisconnect,
        onError,
        autoReconnect = true,
        maxRetries = 10
    } = options;
    
    const socketRef = useRef<Socket | null>(null);
    const [state, setState] = useState<WebSocketState>({
        isConnected: false,
        isReconnecting: false,
        reconnectAttempts: 0,
        error: null
    });
    
    const connect = useCallback(() => {
        if (socketRef.current?.connected) return;
        
        const socket = io(url, {
            auth: { token },
            transports: ['websocket', 'polling'],
            reconnection: autoReconnect,
            reconnectionAttempts: maxRetries,
            reconnectionDelay: 1000,
            reconnectionDelayMax: 30000,
            timeout: 20000
        });
        
        socket.on('connect', () => {
            setState(prev => ({
                ...prev,
                isConnected: true,
                isReconnecting: false,
                reconnectAttempts: 0,
                error: null
            }));
            onConnect?.();
        });
        
        socket.on('disconnect', (reason) => {
            setState(prev => ({
                ...prev,
                isConnected: false
            }));
            onDisconnect?.(reason);
        });
        
        socket.on('connect_error', (error) => {
            setState(prev => ({
                ...prev,
                error: error.message
            }));
            onError?.(error);
        });
        
        socket.on('reconnecting', (attempt) => {
            setState(prev => ({
                ...prev,
                isReconnecting: true,
                reconnectAttempts: attempt
            }));
        });
        
        socket.on('reconnect_failed', () => {
            setState(prev => ({
                ...prev,
                isReconnecting: false,
                error: 'Max reconnection attempts reached'
            }));
        });
        
        socketRef.current = socket;
    }, [url, token, autoReconnect, maxRetries, onConnect, onDisconnect, onError]);
    
    const disconnect = useCallback(() => {
        socketRef.current?.disconnect();
        socketRef.current = null;
    }, []);
    
    const emit = useCallback((event: string, data?: any) => {
        socketRef.current?.emit(event, data);
    }, []);
    
    const on = useCallback((event: string, handler: (...args: any[]) => void) => {
        socketRef.current?.on(event, handler);
        return () => socketRef.current?.off(event, handler);
    }, []);
    
    useEffect(() => {
        connect();
        return () => disconnect();
    }, [connect, disconnect]);
    
    return {
        socket: socketRef.current,
        ...state,
        connect,
        disconnect,
        emit,
        on
    };
}
```

---

## 7. Production Scaling {#scaling}

### Horizontal Scaling ด้วย Redis

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  websocket-1:
    image: mikrotik-realtime:latest
    environment:
      - REDIS_URL=redis://redis:6379
      - PORT=3001
      - NODE_ID=node-1
    deploy:
      replicas: 1
      
  websocket-2:
    image: mikrotik-realtime:latest
    environment:
      - REDIS_URL=redis://redis:6379
      - PORT=3001
      - NODE_ID=node-2
    deploy:
      replicas: 1
  
  nginx-lb:
    image: nginx:alpine
    volumes:
      - ./nginx-ws.conf:/etc/nginx/nginx.conf
    ports:
      - "3001:3001"
    depends_on:
      - websocket-1
      - websocket-2

  redis:
    image: redis:7-alpine
    command: redis-server --maxmemory 2gb --maxmemory-policy allkeys-lru
    volumes:
      - redis_data:/data
```

```nginx
# nginx-ws.conf
upstream websocket_backend {
    ip_hash;  # Sticky sessions สำหรับ WebSocket
    server websocket-1:3001;
    server websocket-2:3001;
}

server {
    listen 3001;
    
    location / {
        proxy_pass http://websocket_backend;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_read_timeout 3600s;
        proxy_send_timeout 3600s;
    }
}
```

---

## 8. Lab: Real-time Dashboard {#lab}

### Lab Setup

```bash
# Backend setup
git clone https://github.com/your-org/mikrotik-realtime
cd mikrotik-realtime
npm install

# Copy config
cp .env.example .env
# แก้ไขค่าใน .env

# Run development
npm run dev

# Frontend setup
cd frontend
npm install
npm run dev
```

### Complete Dashboard Component

```tsx
// src/pages/Dashboard.tsx
import React, { useState, useEffect } from 'react';
import { useWebSocket } from '../hooks/useWebSocket';
import TrafficGraph from '../components/TrafficGraph';
import ResourceMetrics from '../components/ResourceMetrics';
import AlertPanel from '../components/AlertPanel';
import ConnectionTable from '../components/ConnectionTable';

const Dashboard: React.FC = () => {
    const { isConnected, isReconnecting, emit, on } = useWebSocket({
        url: process.env.REACT_APP_WS_URL!,
        token: localStorage.getItem('token')!,
        onConnect: () => {
            // Re-subscribe after reconnect
            selectedRouters.forEach(id => subscribeToRouter(id));
        }
    });
    
    const [selectedRouters] = useState(['router-1', 'router-2']);
    const [alerts, setAlerts] = useState<any[]>([]);
    const [resources, setResources] = useState<Record<string, any>>({});
    
    const subscribeToRouter = (routerId: string) => {
        emit('subscribe:traffic', { routerId, interval: 1000 });
        emit('subscribe:resources', { routerId });
        emit('subscribe:connections', { routerId });
    };
    
    useEffect(() => {
        const cleanup = on('alert', (alert: any) => {
            setAlerts(prev => [alert, ...prev.slice(0, 99)]);
        });
        
        const cleanupResources = on('resources:update', (data: any) => {
            setResources(prev => ({
                ...prev,
                [data.routerId]: data.data
            }));
        });
        
        if (isConnected) {
            selectedRouters.forEach(subscribeToRouter);
            emit('subscribe:alerts', { routerIds: selectedRouters });
        }
        
        return () => {
            cleanup();
            cleanupResources();
        };
    }, [isConnected]);
    
    return (
        <div className="dashboard">
            <header>
                <h1>Network Operations Center</h1>
                <div className={`connection-status ${isConnected ? 'online' : isReconnecting ? 'reconnecting' : 'offline'}`}>
                    {isConnected ? '● Connected' : isReconnecting ? '↺ Reconnecting...' : '○ Offline'}
                </div>
            </header>
            
            <div className="alert-bar">
                <AlertPanel alerts={alerts} />
            </div>
            
            <div className="metrics-row">
                {selectedRouters.map(routerId => (
                    <ResourceMetrics
                        key={routerId}
                        routerId={routerId}
                        data={resources[routerId]}
                    />
                ))}
            </div>
            
            <div className="graphs-grid">
                {selectedRouters.map(routerId => (
                    <TrafficGraph
                        key={routerId}
                        routerId={routerId}
                        interfaceName="ether1"
                        token={localStorage.getItem('token')!}
                    />
                ))}
            </div>
        </div>
    );
};

export default Dashboard;
```

### Verification Checklist

- [ ] WebSocket server starts successfully
- [ ] Client connects and receives welcome message
- [ ] Traffic graphs update every second
- [ ] Alerts appear when thresholds exceeded
- [ ] Reconnection works after network drop
- [ ] Multiple routers stream simultaneously
- [ ] Redis adapter works for horizontal scaling
- [ ] Memory usage stays stable over time

> **Tip:** ใช้ `socket.io-admin-ui` เพื่อ monitor Socket.IO server ใน production

> **Warning:** ระวัง memory leak ถ้าไม่ cleanup interval timers ใน disconnect handler

---

## Summary

Part นี้ครอบคลุม:
- **WebSocket** สำหรับ real-time MikroTik monitoring
- **Socket.io** setup พร้อม authentication
- **Traffic graphs** แบบ live ด้วย Chart.js
- **Alert system** แบบ real-time
- **Horizontal scaling** ด้วย Redis adapter
- **Client reconnection** handling

---

[← Part 61: REST API](part-061-restful-api.md) | [Part 63: Database Integration →](part-063-database-integration.md)
