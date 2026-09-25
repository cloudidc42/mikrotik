# Part 52: MikroTik API ด้วย PHP

## บทนำ

PHP เป็นภาษาที่นิยมมากสำหรับการสร้าง web applications การใช้ PHP กับ MikroTik API ช่วยให้สร้าง web-based management tools, dashboards, และ portals ได้อย่างรวดเร็ว

---

## 52.1 PHP API Library Setup

### ติดตั้ง routeros-api ด้วย Composer

```bash
# สร้าง project
mkdir mikrotik-php
cd mikrotik-php

# Initialize Composer
composer init

# ติดตั้ง library
composer require evilfreelancer/routeros-api

# หรือ clone RouterOS API directly
git clone https://github.com/BenMenking/routeros-api.git
```

### Manual Installation (ไม่ใช้ Composer)

```bash
# Download class.routeros_api.php
wget https://raw.githubusercontent.com/BenMenking/routeros-api/master/routeros_api.class.php

# ใช้ในไฟล์ PHP
# require_once 'routeros_api.class.php';
```

### ตรวจสอบการติดตั้ง

```php
<?php
// test_connection.php
require_once 'vendor/autoload.php';

use RouterOS\Client;
use RouterOS\Query;

try {
    $client = new Client([
        'host' => '192.168.1.1',
        'user' => 'admin',
        'pass' => '',
        'port' => 8728,
    ]);
    
    echo "✓ Connected to MikroTik!\n";
    
    // Test query
    $query = new Query('/system/identity/print');
    $response = $client->query($query)->read();
    
    echo "Router name: " . $response[0]['name'] . "\n";
    
} catch (Exception $e) {
    echo "✗ Error: " . $e->getMessage() . "\n";
}
```

---

## 52.2 Connection Handling

### Connection Class

```php
<?php
// MikroTikConnection.php

namespace App\MikroTik;

use RouterOS\Client;
use RouterOS\Exceptions\ConnectException;

class MikroTikConnection
{
    private ?Client $client = null;
    private array $config;
    private int $retries;
    
    public function __construct(array $config, int $retries = 3)
    {
        $this->config = array_merge([
            'host' => '192.168.1.1',
            'user' => 'admin',
            'pass' => '',
            'port' => 8728,
            'timeout' => 10,
        ], $config);
        
        $this->retries = $retries;
    }
    
    public function connect(): void
    {
        $attempts = 0;
        $lastException = null;
        
        while ($attempts < $this->retries) {
            try {
                $this->client = new Client($this->config);
                return;
            } catch (ConnectException $e) {
                $attempts++;
                $lastException = $e;
                
                if ($attempts < $this->retries) {
                    sleep(2 ** $attempts); // Exponential backoff
                }
            }
        }
        
        throw new \RuntimeException(
            "Failed to connect after {$this->retries} attempts: " . 
            $lastException->getMessage()
        );
    }
    
    public function getClient(): Client
    {
        if (!$this->client) {
            $this->connect();
        }
        return $this->client;
    }
    
    public function isConnected(): bool
    {
        return $this->client !== null;
    }
    
    public function disconnect(): void
    {
        $this->client = null;
    }
    
    public function __destruct()
    {
        $this->disconnect();
    }
}
```

### Connection Pool

```php
<?php
// ConnectionPool.php

class ConnectionPool
{
    private array $pool = [];
    private int $maxConnections;
    
    public function __construct(int $maxConnections = 5)
    {
        $this->maxConnections = $maxConnections;
    }
    
    public function get(string $host, array $config = []): MikroTikConnection
    {
        $key = $host;
        
        if (!isset($this->pool[$key])) {
            if (count($this->pool) >= $this->maxConnections) {
                // ลบ connection เก่าสุด
                $oldKey = array_key_first($this->pool);
                $this->pool[$oldKey]->disconnect();
                unset($this->pool[$oldKey]);
            }
            
            $this->pool[$key] = new MikroTikConnection(
                array_merge(['host' => $host], $config)
            );
            $this->pool[$key]->connect();
        }
        
        return $this->pool[$key];
    }
    
    public function release(string $host): void
    {
        if (isset($this->pool[$host])) {
            $this->pool[$host]->disconnect();
            unset($this->pool[$host]);
        }
    }
}
```

---

## 52.3 Basic Operations (GET, SET, ADD, REMOVE)

### GET - อ่านข้อมูล

```php
<?php
// RouterOSRepository.php

use RouterOS\Client;
use RouterOS\Query;

class RouterOSRepository
{
    public function __construct(private Client $client) {}
    
    /**
     * Get all items from a resource
     */
    public function getAll(string $path): array
    {
        $query = new Query("$path/print");
        return $this->client->query($query)->read();
    }
    
    /**
     * Get with filter
     */
    public function getWhere(string $path, array $conditions): array
    {
        $query = new Query("$path/print");
        
        foreach ($conditions as $key => $value) {
            $query->where($key, $value);
        }
        
        return $this->client->query($query)->read();
    }
    
    /**
     * Get single item by ID
     */
    public function getById(string $path, string $id): ?array
    {
        $result = $this->getWhere($path, ['.id' => $id]);
        return $result[0] ?? null;
    }
}

// ตัวอย่างการใช้งาน
$repo = new RouterOSRepository($client);

// ดู IP addresses ทั้งหมด
$addresses = $repo->getAll('/ip/address');
foreach ($addresses as $addr) {
    echo $addr['address'] . ' on ' . $addr['interface'] . "\n";
}

// ดู interfaces ที่ running
$runningInterfaces = $repo->getWhere('/interface', ['running' => 'true']);
```

### SET - แก้ไขข้อมูล

```php
<?php
/**
 * Set/Update values
 */
public function set(string $path, string $id, array $values): bool
{
    $query = new Query("$path/set");
    $query->equal('.id', $id);
    
    foreach ($values as $key => $value) {
        $query->equal($key, $value);
    }
    
    try {
        $this->client->query($query)->read();
        return true;
    } catch (Exception $e) {
        error_log("Set failed: " . $e->getMessage());
        return false;
    }
}

// ตัวอย่างการแก้ไข interface comment
$repo->set('/interface', '*1', [
    'comment' => 'Updated comment',
    'mtu' => '1500'
]);
```

### ADD - เพิ่มข้อมูล

```php
<?php
/**
 * Add new item
 */
public function add(string $path, array $values): ?string
{
    $query = new Query("$path/add");
    
    foreach ($values as $key => $value) {
        $query->equal($key, $value);
    }
    
    try {
        $response = $this->client->query($query)->read();
        return $response[0]['ret'] ?? null;
    } catch (Exception $e) {
        error_log("Add failed: " . $e->getMessage());
        return null;
    }
}

// ตัวอย่างเพิ่ม IP address
$id = $repo->add('/ip/address', [
    'address' => '10.0.0.1/24',
    'interface' => 'ether2',
    'comment' => 'Added via PHP API'
]);

echo "Added with ID: $id\n";
```

### REMOVE - ลบข้อมูล

```php
<?php
/**
 * Remove item
 */
public function remove(string $path, string $id): bool
{
    $query = new Query("$path/remove");
    $query->equal('.id', $id);
    
    try {
        $this->client->query($query)->read();
        return true;
    } catch (Exception $e) {
        error_log("Remove failed: " . $e->getMessage());
        return false;
    }
}

// ลบ IP address
$repo->remove('/ip/address', '*5');
```

---

## 52.4 Asynchronous Operations

### Async with pcntl_fork

```php
<?php
// async_operations.php

function fetchMultipleRouters(array $routers): array
{
    $results = [];
    $pipes = [];
    $pids = [];
    
    foreach ($routers as $index => $router) {
        // สร้าง pipe สำหรับ IPC
        socket_pair(AF_UNIX, SOCK_STREAM, 0, $socketPair);
        
        $pid = pcntl_fork();
        
        if ($pid === 0) {
            // Child process
            socket_close($socketPair[0]);
            
            try {
                $client = new Client([
                    'host' => $router['host'],
                    'user' => $router['user'],
                    'pass' => $router['pass'],
                ]);
                
                $query = new Query('/system/resource/print');
                $data = $client->query($query)->read();
                
                socket_write($socketPair[1], json_encode([
                    'success' => true,
                    'data' => $data,
                    'host' => $router['host']
                ]));
            } catch (Exception $e) {
                socket_write($socketPair[1], json_encode([
                    'success' => false,
                    'error' => $e->getMessage(),
                    'host' => $router['host']
                ]));
            }
            
            socket_close($socketPair[1]);
            exit(0);
        } else {
            // Parent process
            socket_close($socketPair[1]);
            $pipes[$pid] = $socketPair[0];
            $pids[] = $pid;
        }
    }
    
    // รอผลจาก children
    foreach ($pids as $pid) {
        $response = '';
        while ($chunk = socket_read($pipes[$pid], 4096)) {
            $response .= $chunk;
        }
        socket_close($pipes[$pid]);
        
        $results[] = json_decode($response, true);
        pcntl_waitpid($pid, $status);
    }
    
    return $results;
}
```

---

## 52.5 Error Handling

### Comprehensive Error Handling

```php
<?php
// RouterOSException.php

class RouterOSException extends \RuntimeException
{
    private ?string $category;
    
    public function __construct(string $message, ?string $category = null, int $code = 0)
    {
        parent::__construct($message, $code);
        $this->category = $category;
    }
    
    public function getCategory(): ?string
    {
        return $this->category;
    }
}

// ใน repository
public function safeExecute(callable $operation, $default = null)
{
    try {
        return $operation();
    } catch (RouterOSException $e) {
        switch ($e->getCategory()) {
            case 'no_such_item':
                return $default;
            case 'access_denied':
                throw new \UnauthorizedException("Access denied to RouterOS resource");
            default:
                $this->logger->error("RouterOS API error: " . $e->getMessage(), [
                    'category' => $e->getCategory(),
                ]);
                throw $e;
        }
    } catch (\Exception $e) {
        $this->logger->critical("Unexpected error: " . $e->getMessage());
        throw $e;
    }
}
```

---

## 52.6 Building a PHP Class Wrapper

### RouterOS Manager Class

```php
<?php
// RouterOSManager.php

namespace App\MikroTik;

use RouterOS\Client;
use RouterOS\Query;
use Psr\Log\LoggerInterface;

class RouterOSManager
{
    private Client $client;
    
    public function __construct(
        private string $host,
        private string $username,
        private string $password,
        private LoggerInterface $logger,
        private int $port = 8728
    ) {
        $this->connect();
    }
    
    private function connect(): void
    {
        $this->client = new Client([
            'host' => $this->host,
            'user' => $this->username,
            'pass' => $this->password,
            'port' => $this->port,
        ]);
    }
    
    // ===== SYSTEM =====
    
    public function getSystemIdentity(): string
    {
        $result = $this->query('/system/identity/print');
        return $result[0]['name'] ?? 'Unknown';
    }
    
    public function getSystemResources(): array
    {
        $result = $this->query('/system/resource/print');
        return $result[0] ?? [];
    }
    
    public function getSystemUptime(): string
    {
        $resources = $this->getSystemResources();
        return $resources['uptime'] ?? '0s';
    }
    
    // ===== INTERFACES =====
    
    public function getInterfaces(bool $onlyRunning = false): array
    {
        $query = new Query('/interface/print');
        if ($onlyRunning) {
            $query->where('running', 'true');
        }
        return $this->client->query($query)->read();
    }
    
    public function getInterfaceStats(string $interfaceName): array
    {
        $query = new Query('/interface/monitor-traffic');
        $query->equal('interface', $interfaceName);
        $query->equal('once', '');
        return $this->client->query($query)->read();
    }
    
    // ===== IP =====
    
    public function getIPAddresses(): array
    {
        return $this->query('/ip/address/print');
    }
    
    public function addIPAddress(string $address, string $interface, string $comment = ''): string
    {
        $query = new Query('/ip/address/add');
        $query->equal('address', $address)
              ->equal('interface', $interface);
        
        if ($comment) {
            $query->equal('comment', $comment);
        }
        
        $result = $this->client->query($query)->read();
        return $result[0]['ret'] ?? '';
    }
    
    // ===== FIREWALL =====
    
    public function getFirewallRules(string $chain = ''): array
    {
        $query = new Query('/ip/firewall/filter/print');
        if ($chain) {
            $query->where('chain', $chain);
        }
        return $this->client->query($query)->read();
    }
    
    public function addToAddressList(string $list, string $address, string $timeout = ''): void
    {
        $query = new Query('/ip/firewall/address-list/add');
        $query->equal('list', $list)
              ->equal('address', $address);
        
        if ($timeout) {
            $query->equal('timeout', $timeout);
        }
        
        $this->client->query($query)->read();
    }
    
    // ===== HOTSPOT =====
    
    public function getActiveHotspotUsers(): array
    {
        return $this->query('/ip/hotspot/active/print');
    }
    
    public function createHotspotUser(
        string $username,
        string $password,
        string $profile = 'default'
    ): void {
        $query = new Query('/ip/hotspot/user/add');
        $query->equal('name', $username)
              ->equal('password', $password)
              ->equal('profile', $profile);
        
        $this->client->query($query)->read();
    }
    
    // ===== HELPER =====
    
    private function query(string $command, array $params = []): array
    {
        $query = new Query($command);
        foreach ($params as $key => $value) {
            $query->equal($key, $value);
        }
        
        $this->logger->debug("RouterOS Query: $command");
        
        return $this->client->query($query)->read();
    }
}
```

---

## 52.7 Security Best Practices

### Secure PHP API Implementation

```php
<?php
// secure_config.php

class SecureConfig
{
    private array $config;
    
    public function __construct()
    {
        // โหลด config จาก environment variables
        $this->config = [
            'host' => getenv('MIKROTIK_HOST') ?: '192.168.1.1',
            'user' => getenv('MIKROTIK_USER') ?: 'api-user',
            'pass' => getenv('MIKROTIK_PASS') ?: '',
            'port' => (int)(getenv('MIKROTIK_PORT') ?: 8728),
            'ssl'  => (bool)(getenv('MIKROTIK_SSL') ?: false),
        ];
        
        // Validate
        $this->validate();
    }
    
    private function validate(): void
    {
        if (empty($this->config['host'])) {
            throw new \InvalidArgumentException("MIKROTIK_HOST is required");
        }
        
        if (!filter_var($this->config['host'], FILTER_VALIDATE_IP) &&
            !filter_var($this->config['host'], FILTER_VALIDATE_DOMAIN)) {
            throw new \InvalidArgumentException("Invalid host: " . $this->config['host']);
        }
    }
    
    public function get(): array
    {
        return $this->config;
    }
}
```

---

## 52.8 Example: PHP Dashboard

### Dashboard Controller

```php
<?php
// DashboardController.php

class DashboardController
{
    private RouterOSManager $router;
    
    public function __construct(RouterOSManager $router)
    {
        $this->router = $router;
    }
    
    public function index(): void
    {
        $data = [
            'identity' => $this->router->getSystemIdentity(),
            'resources' => $this->router->getSystemResources(),
            'interfaces' => $this->router->getInterfaces(onlyRunning: true),
            'activeHotspotUsers' => count($this->router->getActiveHotspotUsers()),
        ];
        
        header('Content-Type: application/json');
        echo json_encode($data);
    }
    
    public function getStats(): void
    {
        $resources = $this->router->getSystemResources();
        
        $stats = [
            'cpu_load' => $resources['cpu-load'] ?? 0,
            'free_memory' => $resources['free-memory'] ?? 0,
            'total_memory' => $resources['total-memory'] ?? 0,
            'uptime' => $resources['uptime'] ?? '0s',
            'version' => $resources['version'] ?? 'Unknown',
        ];
        
        header('Content-Type: application/json');
        echo json_encode($stats);
    }
}

// index.php
<?php
require_once 'vendor/autoload.php';

$config = new SecureConfig();
$manager = new RouterOSManager(
    host: $config->get()['host'],
    username: $config->get()['user'],
    password: $config->get()['pass'],
    logger: new \Monolog\Logger('mikrotik'),
);

$controller = new DashboardController($manager);

// Simple router
$path = $_GET['path'] ?? 'dashboard';
switch ($path) {
    case 'dashboard':
        $controller->index();
        break;
    case 'stats':
        $controller->getStats();
        break;
    default:
        http_response_code(404);
        echo json_encode(['error' => 'Not found']);
}
```

---

## 52.9 Example: User Management

### User Management System

```php
<?php
// UserManagement.php

class UserManagement
{
    public function __construct(private RouterOSManager $router) {}
    
    public function listUsers(): array
    {
        // รวมข้อมูลจากหลายแหล่ง
        $hotspotUsers = $this->router->query('/ip/hotspot/user/print');
        $activeUsers = $this->router->getActiveHotspotUsers();
        
        // สร้าง active user map
        $activeMap = [];
        foreach ($activeUsers as $active) {
            $activeMap[$active['user']] = $active;
        }
        
        // รวมข้อมูล
        return array_map(function ($user) use ($activeMap) {
            $user['online'] = isset($activeMap[$user['name']]);
            if ($user['online']) {
                $user['session_info'] = $activeMap[$user['name']];
            }
            return $user;
        }, $hotspotUsers);
    }
    
    public function createUser(array $userData): array
    {
        // Validate input
        if (empty($userData['username'])) {
            throw new \InvalidArgumentException("Username is required");
        }
        
        // ตรวจสอบ duplicate
        $existing = $this->router->query('/ip/hotspot/user/print');
        foreach ($existing as $user) {
            if ($user['name'] === $userData['username']) {
                throw new \DuplicateEntryException("Username already exists");
            }
        }
        
        // สร้าง user
        $this->router->createHotspotUser(
            username: $userData['username'],
            password: $userData['password'],
            profile: $userData['profile'] ?? 'default'
        );
        
        return ['success' => true, 'username' => $userData['username']];
    }
    
    public function disconnectUser(string $username): bool
    {
        $active = $this->router->getActiveHotspotUsers();
        
        foreach ($active as $session) {
            if ($session['user'] === $username) {
                $query = new Query('/ip/hotspot/active/remove');
                $query->equal('.id', $session['.id']);
                // execute...
                return true;
            }
        }
        
        return false;
    }
    
    public function getUserStats(string $username): array
    {
        $users = $this->router->query('/ip/hotspot/user/print');
        
        foreach ($users as $user) {
            if ($user['name'] === $username) {
                return [
                    'username' => $user['name'],
                    'profile' => $user['profile'] ?? 'default',
                    'comment' => $user['comment'] ?? '',
                    'bytes_in' => $user['bytes-in'] ?? 0,
                    'bytes_out' => $user['bytes-out'] ?? 0,
                    'packets_in' => $user['packets-in'] ?? 0,
                    'packets_out' => $user['packets-out'] ?? 0,
                    'uptime' => $user['uptime'] ?? '0s',
                ];
            }
        }
        
        throw new \NotFoundException("User not found: $username");
    }
}
```

---

## 52.10 Lab: PHP Network Manager

### Lab Overview

| ระบบ | รายละเอียด |
|------|------------|
| Framework | PHP 8.1+ |
| Library | evilfreelancer/routeros-api |
| Features | Dashboard, User management, Monitoring |

### Complete Project Structure

```
mikrotik-manager/
├── composer.json
├── .env
├── public/
│   └── index.php
├── src/
│   ├── Config/
│   │   └── RouterConfig.php
│   ├── Manager/
│   │   └── RouterOSManager.php
│   ├── Repository/
│   │   └── RouterOSRepository.php
│   └── Controller/
│       ├── DashboardController.php
│       └── UserController.php
└── templates/
    └── dashboard.html
```

### composer.json

```json
{
    "name": "myproject/mikrotik-manager",
    "description": "MikroTik Network Manager",
    "require": {
        "php": ">=8.1",
        "evilfreelancer/routeros-api": "^1.5",
        "monolog/monolog": "^3.0"
    },
    "autoload": {
        "psr-4": {
            "App\\": "src/"
        }
    }
}
```

### ติดตั้งและรัน

```bash
# ติดตั้ง dependencies
composer install

# สร้าง .env file
cat > .env << EOF
MIKROTIK_HOST=192.168.1.1
MIKROTIK_USER=admin
MIKROTIK_PASS=
MIKROTIK_PORT=8728
EOF

# รัน PHP server
php -S localhost:8080 -t public/

# ทดสอบ
curl http://localhost:8080/?path=dashboard
curl http://localhost:8080/?path=stats
```

---

## Summary

| หัวข้อ | สิ่งที่เรียนรู้ |
|--------|----------------|
| Library Setup | Composer, routeros-api |
| Connection | Pool, retry logic |
| CRUD Operations | Get, Set, Add, Remove |
| Async | pcntl_fork parallel operations |
| Error Handling | Exception hierarchy |
| Wrapper Class | RouterOSManager |
| Security | Config, validation |
| Dashboard | API endpoint for UI |
| User Management | Create, list, disconnect |

---

[← Part 51: MikroTik API Intro](part-051-mikrotik-api-intro.md) | [Part 53: API Python →](part-053-api-python.md)
