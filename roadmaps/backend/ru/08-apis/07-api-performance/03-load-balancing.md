# Load Balancing

## Введение

Load Balancing (балансировка нагрузки) — это техника распределения входящего трафика между несколькими серверами для обеспечения высокой доступности, масштабируемости и оптимального использования ресурсов.

## Архитектура Load Balancing

```
                            ┌─────────────────┐
                            │    Internet     │
                            └────────┬────────┘
                                     │
                            ┌────────▼────────┐
                            │  DNS (Route 53) │
                            │   Round Robin   │
                            └────────┬────────┘
                                     │
              ┌──────────────────────┼──────────────────────┐
              │                      │                      │
     ┌────────▼────────┐   ┌────────▼────────┐   ┌────────▼────────┐
     │   CDN Edge 1    │   │   CDN Edge 2    │   │   CDN Edge 3    │
     └────────┬────────┘   └────────┬────────┘   └────────┬────────┘
              │                      │                      │
              └──────────────────────┼──────────────────────┘
                                     │
                            ┌────────▼────────┐
                            │  Load Balancer  │
                            │  (L4/L7 - HA)   │
                            └────────┬────────┘
                                     │
         ┌───────────────────────────┼───────────────────────────┐
         │                           │                           │
┌────────▼────────┐        ┌────────▼────────┐        ┌────────▼────────┐
│   API Server 1  │        │   API Server 2  │        │   API Server 3  │
└─────────────────┘        └─────────────────┘        └─────────────────┘
```

## Типы Load Balancers

### L4 (Transport Layer) vs L7 (Application Layer)

```python
"""
L4 Load Balancer (TCP/UDP):
- Работает на уровне TCP/UDP
- Не видит содержимое HTTP запросов
- Очень быстрый, низкая задержка
- Простые алгоритмы балансировки

L7 Load Balancer (HTTP/HTTPS):
- Работает на уровне приложения
- Видит HTTP headers, cookies, URL paths
- Может делать content-based routing
- SSL termination
- Более гибкий, но немного медленнее
"""

from enum import Enum
from dataclasses import dataclass
from typing import List, Optional, Dict

class LoadBalancerType(Enum):
    L4 = "transport"
    L7 = "application"

@dataclass
class LoadBalancerConfig:
    type: LoadBalancerType

    # L4 specific
    tcp_port: int = 80
    health_check_interval: int = 30

    # L7 specific
    ssl_termination: bool = True
    sticky_sessions: bool = False
    path_based_routing: bool = False
    header_based_routing: bool = False

# L4 подходит для:
l4_use_cases = [
    "High-throughput TCP services",
    "Database connections",
    "Non-HTTP protocols",
    "UDP-based services (DNS, gaming)"
]

# L7 подходит для:
l7_use_cases = [
    "HTTP/HTTPS APIs",
    "Microservices routing",
    "A/B testing",
    "Canary deployments",
    "Authentication offloading"
]
```

## Алгоритмы балансировки

### Round Robin

```python
from typing import List
from itertools import cycle

class RoundRobinBalancer:
    """
    Round Robin — простейший алгоритм

    Каждый следующий запрос направляется на следующий сервер по кругу.
    Не учитывает нагрузку на серверы.
    """

    def __init__(self, servers: List[str]):
        self.servers = servers
        self.iterator = cycle(servers)

    def get_server(self) -> str:
        return next(self.iterator)

# Использование
balancer = RoundRobinBalancer(["server1", "server2", "server3"])
# Запросы: server1 -> server2 -> server3 -> server1 -> ...
```

### Weighted Round Robin

```python
from dataclasses import dataclass
from typing import List

@dataclass
class WeightedServer:
    address: str
    weight: int  # Чем больше вес, тем больше запросов

class WeightedRoundRobin:
    """
    Weighted Round Robin

    Серверы получают запросы пропорционально своему весу.
    Полезно когда серверы имеют разную производительность.
    """

    def __init__(self, servers: List[WeightedServer]):
        self.servers = servers
        self.current_index = 0
        self.current_weight = 0
        self.max_weight = max(s.weight for s in servers)
        self.gcd_weight = self._gcd_of_weights()

    def _gcd(self, a: int, b: int) -> int:
        while b:
            a, b = b, a % b
        return a

    def _gcd_of_weights(self) -> int:
        result = self.servers[0].weight
        for s in self.servers[1:]:
            result = self._gcd(result, s.weight)
        return result

    def get_server(self) -> str:
        while True:
            self.current_index = (self.current_index + 1) % len(self.servers)

            if self.current_index == 0:
                self.current_weight -= self.gcd_weight
                if self.current_weight <= 0:
                    self.current_weight = self.max_weight

            server = self.servers[self.current_index]
            if server.weight >= self.current_weight:
                return server.address

# Пример: server1(w=5) получит в 5 раз больше запросов чем server3(w=1)
servers = [
    WeightedServer("server1:8080", weight=5),
    WeightedServer("server2:8080", weight=3),
    WeightedServer("server3:8080", weight=1),
]
balancer = WeightedRoundRobin(servers)
```

### Least Connections

```python
from dataclasses import dataclass, field
from typing import Dict
import heapq
import threading

@dataclass(order=True)
class ServerConnection:
    connections: int
    address: str = field(compare=False)

class LeastConnectionsBalancer:
    """
    Least Connections

    Запрос направляется на сервер с наименьшим количеством
    активных соединений. Лучше адаптируется к неравномерной нагрузке.
    """

    def __init__(self, servers: List[str]):
        self.lock = threading.Lock()
        self.servers: Dict[str, int] = {s: 0 for s in servers}

    def get_server(self) -> str:
        with self.lock:
            # Находим сервер с минимальным количеством соединений
            server = min(self.servers, key=self.servers.get)
            self.servers[server] += 1
            return server

    def release_connection(self, server: str):
        with self.lock:
            if server in self.servers:
                self.servers[server] = max(0, self.servers[server] - 1)

    def get_stats(self) -> Dict[str, int]:
        with self.lock:
            return dict(self.servers)

# Использование с контекстным менеджером
class ConnectionContext:
    def __init__(self, balancer: LeastConnectionsBalancer):
        self.balancer = balancer
        self.server = None

    def __enter__(self):
        self.server = self.balancer.get_server()
        return self.server

    def __exit__(self, exc_type, exc_val, exc_tb):
        if self.server:
            self.balancer.release_connection(self.server)

# Использование
balancer = LeastConnectionsBalancer(["server1", "server2", "server3"])

with ConnectionContext(balancer) as server:
    # Делаем запрос к server
    response = make_request(server)
# Соединение автоматически освобождается
```

### Least Response Time

```python
from dataclasses import dataclass
from typing import Dict, List
from collections import deque
import time
import statistics

@dataclass
class ServerStats:
    address: str
    response_times: deque  # Последние N времён ответа
    active_connections: int = 0

    def __init__(self, address: str, window_size: int = 100):
        self.address = address
        self.response_times = deque(maxlen=window_size)
        self.active_connections = 0

    @property
    def avg_response_time(self) -> float:
        if not self.response_times:
            return 0.0
        return statistics.mean(self.response_times)

    @property
    def score(self) -> float:
        """
        Комбинированный скор: время ответа + активные соединения
        Меньше = лучше
        """
        return self.avg_response_time * (1 + self.active_connections * 0.1)

class LeastResponseTimeBalancer:
    """
    Least Response Time

    Выбирает сервер с наименьшим средним временем ответа
    и наименьшим количеством активных соединений.
    """

    def __init__(self, servers: List[str]):
        self.stats: Dict[str, ServerStats] = {
            s: ServerStats(s) for s in servers
        }

    def get_server(self) -> str:
        # Выбираем сервер с лучшим скором
        best = min(self.stats.values(), key=lambda s: s.score)
        best.active_connections += 1
        return best.address

    def record_response(self, server: str, response_time: float):
        """Записываем время ответа"""
        if server in self.stats:
            stats = self.stats[server]
            stats.response_times.append(response_time)
            stats.active_connections = max(0, stats.active_connections - 1)

# Использование
balancer = LeastResponseTimeBalancer(["server1", "server2", "server3"])

server = balancer.get_server()
start = time.time()
try:
    response = make_request(server)
finally:
    elapsed = time.time() - start
    balancer.record_response(server, elapsed)
```

### IP Hash

```python
import hashlib
from typing import List

class IPHashBalancer:
    """
    IP Hash (Source IP Affinity)

    Запросы от одного клиента всегда направляются на один сервер.
    Обеспечивает session persistence без sticky sessions.
    """

    def __init__(self, servers: List[str]):
        self.servers = sorted(servers)  # Сортируем для консистентности

    def get_server(self, client_ip: str) -> str:
        hash_value = int(hashlib.md5(client_ip.encode()).hexdigest(), 16)
        index = hash_value % len(self.servers)
        return self.servers[index]

# Использование
balancer = IPHashBalancer(["server1", "server2", "server3"])

# Один и тот же клиент всегда попадёт на один сервер
server1 = balancer.get_server("192.168.1.100")  # -> server2
server2 = balancer.get_server("192.168.1.100")  # -> server2 (тот же)
```

### Consistent Hashing

```python
import hashlib
from typing import List, Optional
from bisect import bisect_left

class ConsistentHashBalancer:
    """
    Consistent Hashing

    Минимизирует перераспределение при добавлении/удалении серверов.
    Использует виртуальные узлы для равномерного распределения.
    """

    def __init__(self, servers: List[str], replicas: int = 100):
        self.replicas = replicas
        self.ring: List[int] = []
        self.hash_to_server: dict = {}

        for server in servers:
            self.add_server(server)

    def _hash(self, key: str) -> int:
        return int(hashlib.sha256(key.encode()).hexdigest(), 16)

    def add_server(self, server: str):
        for i in range(self.replicas):
            hash_key = self._hash(f"{server}:{i}")
            self.ring.append(hash_key)
            self.hash_to_server[hash_key] = server
        self.ring.sort()

    def remove_server(self, server: str):
        for i in range(self.replicas):
            hash_key = self._hash(f"{server}:{i}")
            if hash_key in self.hash_to_server:
                self.ring.remove(hash_key)
                del self.hash_to_server[hash_key]

    def get_server(self, key: str) -> Optional[str]:
        if not self.ring:
            return None

        hash_value = self._hash(key)
        idx = bisect_left(self.ring, hash_value)

        if idx >= len(self.ring):
            idx = 0

        return self.hash_to_server[self.ring[idx]]

# Пример: при удалении сервера только ~1/N ключей перераспределяется
balancer = ConsistentHashBalancer(["server1", "server2", "server3"])
server = balancer.get_server("user:12345")
```

## Health Checks

### Active Health Checks

```python
import asyncio
import httpx
from dataclasses import dataclass, field
from typing import List, Dict, Optional, Set
from datetime import datetime, timedelta
from enum import Enum

class HealthStatus(Enum):
    HEALTHY = "healthy"
    UNHEALTHY = "unhealthy"
    UNKNOWN = "unknown"

@dataclass
class HealthCheckConfig:
    interval: int = 10          # Интервал проверки в секундах
    timeout: int = 5            # Таймаут запроса
    unhealthy_threshold: int = 3  # Количество неудачных проверок для unhealthy
    healthy_threshold: int = 2    # Количество успешных для возврата в healthy
    path: str = "/health"       # Endpoint для проверки

@dataclass
class ServerHealth:
    address: str
    status: HealthStatus = HealthStatus.UNKNOWN
    consecutive_failures: int = 0
    consecutive_successes: int = 0
    last_check: Optional[datetime] = None
    last_response_time: Optional[float] = None

class HealthChecker:
    """Активные health checks для серверов"""

    def __init__(self, servers: List[str], config: HealthCheckConfig = None):
        self.config = config or HealthCheckConfig()
        self.servers: Dict[str, ServerHealth] = {
            s: ServerHealth(address=s) for s in servers
        }
        self._running = False

    async def check_server(self, address: str) -> bool:
        """Проверка одного сервера"""
        url = f"http://{address}{self.config.path}"

        try:
            async with httpx.AsyncClient() as client:
                start = datetime.now()
                response = await client.get(
                    url,
                    timeout=self.config.timeout
                )
                elapsed = (datetime.now() - start).total_seconds()

                server = self.servers[address]
                server.last_check = datetime.now()
                server.last_response_time = elapsed

                if response.status_code == 200:
                    return True
                return False

        except Exception:
            return False

    async def update_server_status(self, address: str, success: bool):
        """Обновление статуса сервера"""
        server = self.servers[address]

        if success:
            server.consecutive_failures = 0
            server.consecutive_successes += 1

            if server.consecutive_successes >= self.config.healthy_threshold:
                server.status = HealthStatus.HEALTHY
        else:
            server.consecutive_successes = 0
            server.consecutive_failures += 1

            if server.consecutive_failures >= self.config.unhealthy_threshold:
                server.status = HealthStatus.UNHEALTHY

    async def run_checks(self):
        """Запуск периодических проверок"""
        self._running = True

        while self._running:
            tasks = [
                self.check_and_update(address)
                for address in self.servers
            ]
            await asyncio.gather(*tasks)
            await asyncio.sleep(self.config.interval)

    async def check_and_update(self, address: str):
        success = await self.check_server(address)
        await self.update_server_status(address, success)

    def get_healthy_servers(self) -> List[str]:
        """Получение списка здоровых серверов"""
        return [
            addr for addr, health in self.servers.items()
            if health.status == HealthStatus.HEALTHY
        ]

    def stop(self):
        self._running = False
```

### Health Check Endpoints

```python
from fastapi import FastAPI, Response
from pydantic import BaseModel
from typing import Dict, List
import psutil
import asyncio

app = FastAPI()

class HealthStatus(BaseModel):
    status: str
    checks: Dict[str, dict]
    version: str

class HealthChecker:
    """Проверки здоровья приложения"""

    def __init__(self):
        self.checks = {}

    def register(self, name: str, check_func):
        self.checks[name] = check_func

    async def run_all(self) -> Dict[str, dict]:
        results = {}

        for name, check in self.checks.items():
            try:
                start = asyncio.get_event_loop().time()
                success = await check()
                elapsed = asyncio.get_event_loop().time() - start

                results[name] = {
                    "status": "pass" if success else "fail",
                    "time_ms": round(elapsed * 1000, 2)
                }
            except Exception as e:
                results[name] = {
                    "status": "fail",
                    "error": str(e)
                }

        return results

health_checker = HealthChecker()

# Регистрация проверок
async def check_database():
    # Проверка соединения с БД
    try:
        await db.execute("SELECT 1")
        return True
    except:
        return False

async def check_redis():
    try:
        await redis.ping()
        return True
    except:
        return False

async def check_disk_space():
    disk = psutil.disk_usage('/')
    return disk.percent < 90  # < 90% использования

async def check_memory():
    memory = psutil.virtual_memory()
    return memory.percent < 90

health_checker.register("database", check_database)
health_checker.register("redis", check_redis)
health_checker.register("disk", check_disk_space)
health_checker.register("memory", check_memory)

@app.get("/health")
async def health_check():
    """
    Простой health check для load balancer
    Возвращает 200 если сервис работает
    """
    return {"status": "healthy"}

@app.get("/health/live")
async def liveness_check():
    """
    Kubernetes Liveness Probe
    Проверяет, что процесс жив
    """
    return {"status": "alive"}

@app.get("/health/ready")
async def readiness_check(response: Response):
    """
    Kubernetes Readiness Probe
    Проверяет, что сервис готов принимать трафик
    """
    results = await health_checker.run_all()

    all_passed = all(r["status"] == "pass" for r in results.values())

    if not all_passed:
        response.status_code = 503

    return {
        "status": "ready" if all_passed else "not_ready",
        "checks": results
    }

@app.get("/health/detailed")
async def detailed_health():
    """Детальный статус для мониторинга"""
    results = await health_checker.run_all()

    return HealthStatus(
        status="healthy" if all(r["status"] == "pass" for r in results.values()) else "degraded",
        checks=results,
        version="1.0.0"
    )
```

## Конфигурация Nginx

### Basic Load Balancing

```nginx
# /etc/nginx/nginx.conf

http {
    # Upstream с серверами приложения
    upstream api_backend {
        # Алгоритм балансировки (по умолчанию round-robin)
        # least_conn;        # Least connections
        # ip_hash;           # IP hash
        # hash $request_uri; # Consistent hash по URI

        # Список серверов
        server api1.example.com:8080 weight=5;
        server api2.example.com:8080 weight=3;
        server api3.example.com:8080 weight=2;

        # Backup сервер — используется только если основные недоступны
        server api4.example.com:8080 backup;

        # Keepalive connections
        keepalive 32;
    }

    server {
        listen 80;
        server_name api.example.com;

        location / {
            proxy_pass http://api_backend;

            # Заголовки для проксирования
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;

            # Таймауты
            proxy_connect_timeout 5s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            # Keepalive
            proxy_http_version 1.1;
            proxy_set_header Connection "";
        }
    }
}
```

### Health Checks и Failover

```nginx
http {
    upstream api_backend {
        zone api_backend 64k;  # Shared memory zone для статуса

        server api1.example.com:8080 max_fails=3 fail_timeout=30s;
        server api2.example.com:8080 max_fails=3 fail_timeout=30s;
        server api3.example.com:8080 max_fails=3 fail_timeout=30s;

        # Активные health checks (Nginx Plus)
        # health_check interval=5s fails=3 passes=2 uri=/health;
    }

    # Пассивные health checks (Open Source Nginx)
    server {
        location / {
            proxy_pass http://api_backend;

            # При ошибке — пробуем следующий сервер
            proxy_next_upstream error timeout http_500 http_502 http_503;
            proxy_next_upstream_tries 3;
            proxy_next_upstream_timeout 10s;
        }
    }
}
```

### SSL Termination

```nginx
http {
    upstream api_backend {
        server api1.example.com:8080;
        server api2.example.com:8080;
    }

    server {
        listen 443 ssl http2;
        server_name api.example.com;

        # SSL сертификаты
        ssl_certificate /etc/ssl/certs/api.example.com.crt;
        ssl_certificate_key /etc/ssl/private/api.example.com.key;

        # SSL настройки
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
        ssl_prefer_server_ciphers on;
        ssl_session_cache shared:SSL:10m;
        ssl_session_timeout 10m;

        # HSTS
        add_header Strict-Transport-Security "max-age=31536000" always;

        location / {
            proxy_pass http://api_backend;
            proxy_set_header X-Forwarded-Proto https;
        }
    }

    # Redirect HTTP to HTTPS
    server {
        listen 80;
        server_name api.example.com;
        return 301 https://$server_name$request_uri;
    }
}
```

### Path-Based Routing

```nginx
http {
    upstream api_v1 {
        server api-v1-1.example.com:8080;
        server api-v1-2.example.com:8080;
    }

    upstream api_v2 {
        server api-v2-1.example.com:8080;
        server api-v2-2.example.com:8080;
    }

    upstream static_content {
        server static1.example.com:80;
        server static2.example.com:80;
    }

    server {
        listen 80;

        # API v1
        location /api/v1/ {
            proxy_pass http://api_v1/;
        }

        # API v2
        location /api/v2/ {
            proxy_pass http://api_v2/;
        }

        # Статический контент
        location /static/ {
            proxy_pass http://static_content/;
            proxy_cache_valid 200 1d;
        }

        # WebSocket
        location /ws/ {
            proxy_pass http://api_v2/ws/;
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}
```

## HAProxy Configuration

### Basic Setup

```haproxy
# /etc/haproxy/haproxy.cfg

global
    log stdout format raw local0
    maxconn 50000
    tune.ssl.default-dh-param 2048

defaults
    mode http
    log global
    option httplog
    option dontlognull
    option http-server-close
    option forwardfor
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    retries 3

frontend http_front
    bind *:80
    bind *:443 ssl crt /etc/ssl/certs/haproxy.pem

    # Redirect HTTP to HTTPS
    http-request redirect scheme https unless { ssl_fc }

    # ACLs для маршрутизации
    acl is_api path_beg /api
    acl is_static path_beg /static

    # Маршрутизация по ACL
    use_backend api_servers if is_api
    use_backend static_servers if is_static
    default_backend api_servers

backend api_servers
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    # Серверы с весами
    server api1 192.168.1.10:8080 weight 100 check inter 5s fall 3 rise 2
    server api2 192.168.1.11:8080 weight 100 check inter 5s fall 3 rise 2
    server api3 192.168.1.12:8080 weight 50 check inter 5s fall 3 rise 2

backend static_servers
    balance roundrobin
    server static1 192.168.1.20:80 check
    server static2 192.168.1.21:80 check

# Статистика
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
    stats admin if LOCALHOST
```

### Advanced Features

```haproxy
frontend http_front
    bind *:443 ssl crt /etc/ssl/certs/haproxy.pem

    # Rate limiting
    stick-table type ip size 100k expire 30s store http_req_rate(10s)
    http-request track-sc0 src
    http-request deny deny_status 429 if { sc_http_req_rate(0) gt 100 }

    # Sticky sessions
    # cookie SERVERID insert indirect nocache

    # Headers
    http-request set-header X-Forwarded-Proto https if { ssl_fc }
    http-response set-header X-Frame-Options SAMEORIGIN
    http-response set-header X-Content-Type-Options nosniff

backend api_servers
    balance leastconn

    # Connection limits
    default-server maxconn 1000

    # Sticky sessions по cookie
    cookie SERVERID insert indirect nocache

    # Health checks
    option httpchk GET /health
    http-check expect status 200

    server api1 192.168.1.10:8080 cookie s1 check
    server api2 192.168.1.11:8080 cookie s2 check
```

## Cloud Load Balancers

### AWS Application Load Balancer (Terraform)

```hcl
# ALB
resource "aws_lb" "api" {
  name               = "api-alb"
  internal           = false
  load_balancer_type = "application"
  security_groups    = [aws_security_group.alb.id]
  subnets            = var.public_subnets

  enable_deletion_protection = true

  access_logs {
    bucket  = aws_s3_bucket.logs.bucket
    prefix  = "alb-logs"
    enabled = true
  }
}

# Target Group
resource "aws_lb_target_group" "api" {
  name     = "api-targets"
  port     = 8080
  protocol = "HTTP"
  vpc_id   = var.vpc_id

  health_check {
    enabled             = true
    path                = "/health"
    port                = "traffic-port"
    protocol            = "HTTP"
    healthy_threshold   = 2
    unhealthy_threshold = 3
    timeout             = 5
    interval            = 10
    matcher             = "200"
  }

  stickiness {
    type            = "lb_cookie"
    cookie_duration = 3600
    enabled         = true
  }
}

# Listener
resource "aws_lb_listener" "https" {
  load_balancer_arn = aws_lb.api.arn
  port              = 443
  protocol          = "HTTPS"
  ssl_policy        = "ELBSecurityPolicy-TLS-1-2-2017-01"
  certificate_arn   = var.certificate_arn

  default_action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api.arn
  }
}

# Path-based routing
resource "aws_lb_listener_rule" "api_v2" {
  listener_arn = aws_lb_listener.https.arn
  priority     = 100

  action {
    type             = "forward"
    target_group_arn = aws_lb_target_group.api_v2.arn
  }

  condition {
    path_pattern {
      values = ["/api/v2/*"]
    }
  }
}
```

### Kubernetes Ingress

```yaml
# Nginx Ingress Controller
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: api-ingress
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "50m"
    nginx.ingress.kubernetes.io/proxy-read-timeout: "60"
    nginx.ingress.kubernetes.io/proxy-send-timeout: "60"
    nginx.ingress.kubernetes.io/load-balance: "least_conn"

    # Rate limiting
    nginx.ingress.kubernetes.io/limit-rps: "100"
    nginx.ingress.kubernetes.io/limit-connections: "5"
spec:
  ingressClassName: nginx
  tls:
    - hosts:
        - api.example.com
      secretName: api-tls-secret
  rules:
    - host: api.example.com
      http:
        paths:
          - path: /api/v1
            pathType: Prefix
            backend:
              service:
                name: api-v1-service
                port:
                  number: 80
          - path: /api/v2
            pathType: Prefix
            backend:
              service:
                name: api-v2-service
                port:
                  number: 80

---
# Service с endpoints
apiVersion: v1
kind: Service
metadata:
  name: api-v1-service
spec:
  selector:
    app: api-v1
  ports:
    - port: 80
      targetPort: 8080
  type: ClusterIP
```

## Session Persistence

### Sticky Sessions

```python
from fastapi import FastAPI, Request, Response
import uuid

app = FastAPI()

class StickySessionMiddleware:
    """Middleware для sticky sessions на уровне приложения"""

    COOKIE_NAME = "SERVER_ID"

    def __init__(self, app, server_id: str):
        self.app = app
        self.server_id = server_id

    async def __call__(self, scope, receive, send):
        if scope["type"] == "http":
            # Добавляем server_id в response cookies
            async def send_wrapper(message):
                if message["type"] == "http.response.start":
                    headers = list(message.get("headers", []))
                    cookie = f"{self.COOKIE_NAME}={self.server_id}; Path=/; HttpOnly"
                    headers.append((b"set-cookie", cookie.encode()))
                    message["headers"] = headers
                await send(message)

            await self.app(scope, receive, send_wrapper)
        else:
            await self.app(scope, receive, send)

# Использование
app = FastAPI()
server_id = os.environ.get("SERVER_ID", str(uuid.uuid4())[:8])
app = StickySessionMiddleware(app, server_id)
```

## Best Practices

### Выбор алгоритма балансировки

```python
"""
Рекомендации по выбору алгоритма:

┌─────────────────────────────────────────────────────────────────┐
│ Сценарий                      │ Рекомендуемый алгоритм         │
├─────────────────────────────────────────────────────────────────┤
│ Однородные серверы, stateless → Round Robin                   │
│ Серверы разной мощности       → Weighted Round Robin           │
│ Долгие соединения/запросы     → Least Connections              │
│ Нужна session affinity        → IP Hash или Cookie-based       │
│ Серверы часто меняются        → Consistent Hashing             │
│ Важно время ответа            → Least Response Time            │
└─────────────────────────────────────────────────────────────────┘
"""

class LoadBalancerRecommendation:
    """Помощник выбора алгоритма балансировки"""

    @staticmethod
    def recommend(
        servers_homogeneous: bool,
        session_required: bool,
        long_connections: bool,
        servers_change_frequently: bool
    ) -> str:

        if session_required:
            if servers_change_frequently:
                return "consistent_hashing"
            return "ip_hash"

        if long_connections:
            return "least_connections"

        if not servers_homogeneous:
            return "weighted_round_robin"

        return "round_robin"
```

### Мониторинг Load Balancer

```yaml
# Prometheus rules для мониторинга
groups:
  - name: load_balancer
    rules:
      - alert: HighBackendErrorRate
        expr: |
          sum(rate(haproxy_backend_http_responses_total{code="5xx"}[5m]))
          / sum(rate(haproxy_backend_http_responses_total[5m])) > 0.01
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "High error rate on backend"

      - alert: BackendDown
        expr: haproxy_backend_up == 0
        for: 1m
        labels:
          severity: critical
        annotations:
          summary: "Backend {{ $labels.backend }} is down"

      - alert: HighConnectionQueue
        expr: haproxy_backend_current_queue > 100
        for: 2m
        labels:
          severity: warning
        annotations:
          summary: "High connection queue"
```

## Заключение

Эффективная балансировка нагрузки требует:

1. **Правильного выбора алгоритма** — учитывайте характеристики нагрузки
2. **Надёжных health checks** — быстро обнаруживайте неработающие серверы
3. **Graceful degradation** — система должна работать при частичных отказах
4. **Мониторинга** — отслеживайте распределение нагрузки и latency
5. **Session management** — при необходимости используйте sticky sessions

Load balancing — критический компонент высоконагруженных систем, обеспечивающий масштабируемость и отказоустойчивость.
