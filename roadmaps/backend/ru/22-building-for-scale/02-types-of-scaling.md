# Типы масштабирования (Types of Scaling)

[prev: 01-migration-strategies](./01-migration-strategies.md) | [next: 03-mitigation-strategies](./03-mitigation-strategies.md)

---

## Определение

**Масштабирование** — это процесс увеличения возможностей системы для обработки растущей нагрузки. Существует несколько подходов к масштабированию, каждый из которых имеет свои преимущества, ограничения и области применения.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                         ТИПЫ МАСШТАБИРОВАНИЯ                                │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   ┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐          │
│   │   ВЕРТИКАЛЬНОЕ  │   │  ГОРИЗОНТАЛЬНОЕ │   │  ДИАГОНАЛЬНОЕ   │          │
│   │   (Scale Up)    │   │   (Scale Out)   │   │   (Комбинация)  │          │
│   │                 │   │                 │   │                 │          │
│   │   ┌───────┐     │   │  ┌──┐ ┌──┐ ┌──┐│   │   ┌───────┐     │          │
│   │   │███████│     │   │  │  │ │  │ │  ││   │   │███████│     │          │
│   │   │███████│     │   │  └──┘ └──┘ └──┘│   │   │███████│     │          │
│   │   │███████│     │   │                │   │   └───────┘     │          │
│   │   │███████│     │   │  ┌──┐ ┌──┐ ┌──┐│   │   ┌──┐ ┌──┐     │          │
│   │   └───────┘     │   │  │  │ │  │ │  ││   │   │██│ │██│     │          │
│   │   Мощнее        │   │  └──┘ └──┘ └──┘│   │   └──┘ └──┘     │          │
│   │                 │   │   Больше       │   │   Мощнее +      │          │
│   │                 │   │                │   │   Больше        │          │
│   └─────────────────┘   └─────────────────┘   └─────────────────┘          │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

---

## 1. Вертикальное масштабирование (Scale Up / Vertical Scaling)

### Определение

Вертикальное масштабирование — это увеличение мощности одного сервера путём добавления ресурсов: CPU, RAM, SSD, улучшения сетевого интерфейса.

```
┌─────────────────────────────────────────────────────────────────┐
│            ВЕРТИКАЛЬНОЕ МАСШТАБИРОВАНИЕ                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│   ДО:                          ПОСЛЕ:                           │
│   ┌─────────────┐              ┌─────────────┐                  │
│   │   Server    │              │   Server    │                  │
│   │             │              │             │                  │
│   │  CPU: 4     │      →       │  CPU: 16    │                  │
│   │  RAM: 8GB   │              │  RAM: 64GB  │                  │
│   │  SSD: 256GB │              │  SSD: 2TB   │                  │
│   │             │              │  NVMe       │                  │
│   └─────────────┘              └─────────────┘                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### Пример: Настройка PostgreSQL для использования ресурсов

```sql
-- Конфигурация PostgreSQL для мощного сервера (64GB RAM, 16 CPU)
-- Файл: postgresql.conf

-- Память
shared_buffers = 16GB                    -- 25% от RAM
effective_cache_size = 48GB              -- 75% от RAM
work_mem = 256MB                         -- Память для сортировки
maintenance_work_mem = 2GB               -- Для VACUUM, INDEX

-- Параллелизм (используем многоядерность)
max_worker_processes = 16
max_parallel_workers_per_gather = 8
max_parallel_workers = 16

-- WAL и чекпоинты
wal_buffers = 64MB
checkpoint_completion_target = 0.9
max_wal_size = 4GB
```

### Пример: Увеличение ресурсов в Docker

```yaml
# docker-compose.yml - До масштабирования
version: '3.8'
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

---
# docker-compose.yml - После вертикального масштабирования
version: '3.8'
services:
  app:
    image: myapp:latest
    deploy:
      resources:
        limits:
          cpus: '8'           # Увеличили CPU
          memory: 16G         # Увеличили RAM
        reservations:
          cpus: '4'
          memory: 8G
```

### Преимущества вертикального масштабирования

| Преимущество | Описание |
|-------------|----------|
| Простота | Не требует изменения архитектуры приложения |
| Целостность данных | Все данные на одном сервере |
| Низкая задержка | Нет сетевых вызовов между узлами |
| Транзакции | ACID-транзакции работают без распределённых блокировок |

### Ограничения вертикального масштабирования

| Ограничение | Описание |
|------------|----------|
| Потолок роста | Физический предел мощности одного сервера |
| Стоимость | Экспоненциальный рост цены |
| Single Point of Failure | Отказ сервера = отказ всей системы |
| Простой при апгрейде | Требуется остановка для замены железа |

---

## 2. Горизонтальное масштабирование (Scale Out / Horizontal Scaling)

### Определение

Горизонтальное масштабирование — это добавление новых серверов/инстансов для распределения нагрузки между ними.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ГОРИЗОНТАЛЬНОЕ МАСШТАБИРОВАНИЕ                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│                          ┌──────────────────┐                               │
│                          │  Load Balancer   │                               │
│                          │    (nginx/HAProxy)│                              │
│                          └────────┬─────────┘                               │
│                                   │                                         │
│              ┌────────────────────┼────────────────────┐                    │
│              │                    │                    │                    │
│              ▼                    ▼                    ▼                    │
│        ┌──────────┐        ┌──────────┐        ┌──────────┐                │
│        │ Server 1 │        │ Server 2 │        │ Server 3 │                │
│        │          │        │          │        │          │                │
│        │ CPU: 4   │        │ CPU: 4   │        │ CPU: 4   │                │
│        │ RAM: 8GB │        │ RAM: 8GB │        │ RAM: 8GB │                │
│        └──────────┘        └──────────┘        └──────────┘                │
│                                                                             │
│        Каждый сервер обрабатывает часть запросов                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Stateless vs Stateful приложения

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                     STATELESS vs STATEFUL                                   │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   STATELESS (Без состояния):             STATEFUL (С состоянием):          │
│   ┌─────────────────────────┐            ┌─────────────────────────┐       │
│   │                         │            │                         │       │
│   │  Любой запрос может     │            │  Запрос должен попасть  │       │
│   │  обработать любой       │            │  на "свой" сервер       │       │
│   │  сервер                 │            │                         │       │
│   │                         │            │  Сессия привязана       │       │
│   │  Сессии в Redis/DB      │            │  к конкретному серверу  │       │
│   │                         │            │                         │       │
│   │  ✓ Легко масштабировать │            │  ✗ Сложно масштабировать│       │
│   │  ✓ Отказоустойчиво      │            │  ✗ Sticky sessions      │       │
│   │                         │            │                         │       │
│   └─────────────────────────┘            └─────────────────────────┘       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Пример: Stateless приложение с Redis для сессий

```python
# app.py - Stateless FastAPI приложение
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import HTTPBearer
import redis
import json
import uuid

app = FastAPI()
security = HTTPBearer()

# Подключение к Redis (общее хранилище сессий)
redis_client = redis.Redis(
    host='redis-cluster',  # Redis доступен всем инстансам
    port=6379,
    decode_responses=True
)

class SessionManager:
    """Менеджер сессий - состояние хранится в Redis, не на сервере"""

    def __init__(self, redis: redis.Redis):
        self.redis = redis
        self.session_ttl = 3600  # 1 час

    def create_session(self, user_id: int, user_data: dict) -> str:
        """Создаёт сессию и возвращает токен"""
        session_id = str(uuid.uuid4())
        session_data = {
            "user_id": user_id,
            "user_data": user_data
        }
        # Сохраняем в Redis - доступно всем серверам
        self.redis.setex(
            f"session:{session_id}",
            self.session_ttl,
            json.dumps(session_data)
        )
        return session_id

    def get_session(self, session_id: str) -> dict | None:
        """Получает сессию из Redis"""
        data = self.redis.get(f"session:{session_id}")
        return json.loads(data) if data else None

    def delete_session(self, session_id: str):
        """Удаляет сессию"""
        self.redis.delete(f"session:{session_id}")

session_manager = SessionManager(redis_client)

@app.post("/login")
async def login(username: str, password: str):
    # Проверка пользователя (упрощённо)
    user_id = 123
    user_data = {"username": username, "role": "user"}

    # Создаём сессию в Redis
    token = session_manager.create_session(user_id, user_data)
    return {"token": token}

@app.get("/profile")
async def get_profile(token: str = Depends(security)):
    # Любой сервер может обработать этот запрос
    session = session_manager.get_session(token.credentials)
    if not session:
        raise HTTPException(status_code=401, detail="Invalid session")
    return session["user_data"]
```

### Стратегии Load Balancing

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    СТРАТЕГИИ БАЛАНСИРОВКИ НАГРУЗКИ                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│  1. ROUND ROBIN                     2. LEAST CONNECTIONS                    │
│  ┌─────────────────────┐            ┌─────────────────────┐                │
│  │ Запрос 1 → Server A │            │ Новый запрос →      │                │
│  │ Запрос 2 → Server B │            │   Server с мин.     │                │
│  │ Запрос 3 → Server C │            │   активными         │                │
│  │ Запрос 4 → Server A │            │   соединениями      │                │
│  │ ...                 │            │                     │                │
│  └─────────────────────┘            └─────────────────────┘                │
│                                                                             │
│  3. IP HASH                         4. WEIGHTED                             │
│  ┌─────────────────────┐            ┌─────────────────────┐                │
│  │ hash(IP) % N        │            │ Server A (w=5): 50% │                │
│  │   → один и тот же   │            │ Server B (w=3): 30% │                │
│  │   сервер для IP     │            │ Server C (w=2): 20% │                │
│  │                     │            │                     │                │
│  └─────────────────────┘            └─────────────────────┘                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Пример: Конфигурация Nginx Load Balancer

```nginx
# /etc/nginx/nginx.conf

# Пул серверов приложения
upstream app_servers {
    # Стратегия: Least Connections
    least_conn;

    # Серверы с весами
    server app1.internal:8000 weight=5;   # Мощный сервер
    server app2.internal:8000 weight=3;   # Средний сервер
    server app3.internal:8000 weight=2;   # Слабый сервер

    # Health check параметры
    # Сервер недоступен после 3 неудач за 30 секунд
    server app4.internal:8000 max_fails=3 fail_timeout=30s;

    # Резервный сервер (используется только если основные недоступны)
    server backup.internal:8000 backup;

    # Keepalive соединения для снижения latency
    keepalive 32;
}

server {
    listen 80;
    server_name api.example.com;

    location / {
        proxy_pass http://app_servers;

        # Заголовки для backend
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;

        # HTTP/1.1 для keepalive
        proxy_http_version 1.1;
        proxy_set_header Connection "";

        # Таймауты
        proxy_connect_timeout 5s;
        proxy_read_timeout 60s;
    }

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "OK";
    }
}
```

### Пример: HAProxy конфигурация

```haproxy
# /etc/haproxy/haproxy.cfg

global
    maxconn 50000
    log stdout format raw local0

defaults
    mode http
    timeout connect 5s
    timeout client 30s
    timeout server 30s
    option httplog
    option dontlognull

# Frontend - принимает входящие запросы
frontend http_front
    bind *:80
    default_backend app_servers

    # Маршрутизация по URL
    acl is_api path_beg /api
    acl is_static path_beg /static

    use_backend api_servers if is_api
    use_backend static_servers if is_static

# Backend - пул серверов API
backend api_servers
    balance roundrobin
    option httpchk GET /health
    http-check expect status 200

    # Серверы с health check
    server api1 10.0.0.1:8000 check inter 5s fall 3 rise 2
    server api2 10.0.0.2:8000 check inter 5s fall 3 rise 2
    server api3 10.0.0.3:8000 check inter 5s fall 3 rise 2

# Backend - статические файлы
backend static_servers
    balance roundrobin
    server static1 10.0.1.1:80 check
    server static2 10.0.1.2:80 check

# Статистика HAProxy
listen stats
    bind *:8404
    stats enable
    stats uri /stats
    stats refresh 10s
```

---

## 3. Диагональное масштабирование (Diagonal Scaling)

### Определение

Диагональное масштабирование — это комбинация вертикального и горизонтального подходов для достижения оптимального баланса производительности и стоимости.

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                      ДИАГОНАЛЬНОЕ МАСШТАБИРОВАНИЕ                           │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Этап 1: Вертикальное           Этап 2: Горизонтальное                    │
│   ┌─────────┐                    ┌─────────┐  ┌─────────┐                  │
│   │█████████│                    │█████████│  │█████████│                  │
│   │█████████│  Упёрлись в        │█████████│  │█████████│                  │
│   │█████████│  потолок   →       │█████████│  │█████████│                  │
│   │█████████│                    │█████████│  │█████████│                  │
│   └─────────┘                    └─────────┘  └─────────┘                  │
│   1 мощный сервер                2 мощных сервера                          │
│                                                                             │
│   Этап 3: Снова горизонтальное                                             │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                       │
│   │█████████│  │█████████│  │█████████│  │█████████│                       │
│   │█████████│  │█████████│  │█████████│  │█████████│                       │
│   │█████████│  │█████████│  │█████████│  │█████████│                       │
│   │█████████│  │█████████│  │█████████│  │█████████│                       │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘                       │
│   4 мощных сервера                                                         │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Стратегия диагонального масштабирования

```python
# scaling_strategy.py - Логика принятия решений о масштабировании

from dataclasses import dataclass
from enum import Enum

class ScalingAction(Enum):
    SCALE_UP = "vertical"      # Увеличить мощность
    SCALE_OUT = "horizontal"   # Добавить инстансы
    SCALE_DOWN = "reduce"      # Уменьшить ресурсы
    NO_ACTION = "none"

@dataclass
class ServerMetrics:
    cpu_percent: float
    memory_percent: float
    network_io_percent: float
    disk_io_percent: float
    instance_count: int
    instance_size: str  # "small", "medium", "large", "xlarge"

@dataclass
class ScalingThresholds:
    cpu_high: float = 80.0
    cpu_low: float = 20.0
    memory_high: float = 85.0
    max_instances: int = 10
    max_size: str = "xlarge"

def determine_scaling_action(
    metrics: ServerMetrics,
    thresholds: ScalingThresholds
) -> ScalingAction:
    """
    Определяет оптимальное действие масштабирования.

    Стратегия:
    1. Сначала пробуем вертикальное масштабирование (дешевле управлять)
    2. Если достигли потолка - горизонтальное
    3. При низкой нагрузке - уменьшаем ресурсы
    """

    # Проверяем необходимость увеличения ресурсов
    needs_more_resources = (
        metrics.cpu_percent > thresholds.cpu_high or
        metrics.memory_percent > thresholds.memory_high
    )

    if needs_more_resources:
        # Можем ли ещё увеличить размер инстанса?
        if can_scale_up(metrics.instance_size, thresholds.max_size):
            return ScalingAction.SCALE_UP
        # Если нет - добавляем инстансы
        elif metrics.instance_count < thresholds.max_instances:
            return ScalingAction.SCALE_OUT

    # Проверяем возможность уменьшения ресурсов
    if (metrics.cpu_percent < thresholds.cpu_low and
        metrics.memory_percent < thresholds.cpu_low):
        return ScalingAction.SCALE_DOWN

    return ScalingAction.NO_ACTION

def can_scale_up(current_size: str, max_size: str) -> bool:
    """Проверяет возможность вертикального масштабирования"""
    sizes = ["small", "medium", "large", "xlarge"]
    current_idx = sizes.index(current_size)
    max_idx = sizes.index(max_size)
    return current_idx < max_idx
```

---

## 4. Масштабирование баз данных

### 4.1. Репликация (Replication)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    MASTER-SLAVE РЕПЛИКАЦИЯ                                  │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Приложение                                                                │
│       │                                                                     │
│       ├──── Write ────►  ┌─────────────┐                                   │
│       │                  │   MASTER    │                                   │
│       │                  │  (Primary)  │                                   │
│       │                  └──────┬──────┘                                   │
│       │                         │ WAL / Binlog                             │
│       │                         │ (репликация)                             │
│       │              ┌──────────┼──────────┐                               │
│       │              │          │          │                               │
│       │              ▼          ▼          ▼                               │
│       │         ┌────────┐ ┌────────┐ ┌────────┐                           │
│       └─ Read ─►│ SLAVE  │ │ SLAVE  │ │ SLAVE  │                           │
│                 │   #1   │ │   #2   │ │   #3   │                           │
│                 └────────┘ └────────┘ └────────┘                           │
│                                                                             │
│   Write: только на Master    Read: распределяется по Slaves                │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Пример: PostgreSQL репликация

```bash
# На MASTER сервере - postgresql.conf
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB
hot_standby = on

# pg_hba.conf - разрешаем репликацию
host    replication     replicator      10.0.0.0/24     scram-sha-256
```

```bash
# На SLAVE сервере - создание реплики
pg_basebackup -h master.internal -D /var/lib/postgresql/data \
    -U replicator -P -v -R -X stream -C -S replica_slot_1
```

### Пример: Приложение с разделением Read/Write

```python
# db_connection.py - Разделение чтения и записи

from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from contextlib import contextmanager
from typing import Generator
import random

class DatabaseRouter:
    """
    Роутер базы данных.
    Записи идут на master, чтения распределяются по репликам.
    """

    def __init__(self):
        # Master для записи
        self.master_engine = create_engine(
            "postgresql://user:pass@master.internal:5432/app",
            pool_size=20,
            max_overflow=10
        )

        # Пул реплик для чтения
        self.replica_engines = [
            create_engine(
                f"postgresql://user:pass@replica{i}.internal:5432/app",
                pool_size=10,
                max_overflow=5
            )
            for i in range(1, 4)  # 3 реплики
        ]

        self.MasterSession = sessionmaker(bind=self.master_engine)
        self.ReplicaSessions = [
            sessionmaker(bind=engine)
            for engine in self.replica_engines
        ]

    @contextmanager
    def write_session(self) -> Generator[Session, None, None]:
        """Сессия для записи (всегда master)"""
        session = self.MasterSession()
        try:
            yield session
            session.commit()
        except Exception:
            session.rollback()
            raise
        finally:
            session.close()

    @contextmanager
    def read_session(self) -> Generator[Session, None, None]:
        """Сессия для чтения (случайная реплика)"""
        replica_session_class = random.choice(self.ReplicaSessions)
        session = replica_session_class()
        try:
            yield session
        finally:
            session.close()

# Использование
db = DatabaseRouter()

async def create_user(user_data: dict):
    """Запись - идёт на master"""
    with db.write_session() as session:
        user = User(**user_data)
        session.add(user)
        return user

async def get_users():
    """Чтение - идёт на реплику"""
    with db.read_session() as session:
        return session.query(User).all()
```

### 4.2. Шардинг (Sharding)

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                           ШАРДИНГ БАЗЫ ДАННЫХ                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Ключ шардирования: user_id                                               │
│                                                                             │
│   user_id % 4 = ?                                                          │
│                                                                             │
│   ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐                       │
│   │ Shard 0 │  │ Shard 1 │  │ Shard 2 │  │ Shard 3 │                       │
│   │         │  │         │  │         │  │         │                       │
│   │ user_id │  │ user_id │  │ user_id │  │ user_id │                       │
│   │  % 4 = 0│  │  % 4 = 1│  │  % 4 = 2│  │  % 4 = 3│                       │
│   │         │  │         │  │         │  │         │                       │
│   │ 0,4,8.. │  │ 1,5,9.. │  │ 2,6,10..│  │ 3,7,11..│                       │
│   └─────────┘  └─────────┘  └─────────┘  └─────────┘                       │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Пример: Реализация шардинга

```python
# sharding.py - Шардирование по user_id

from typing import Dict, Any
from sqlalchemy import create_engine
from sqlalchemy.orm import sessionmaker, Session
from hashlib import md5

class ShardRouter:
    """
    Роутер для шардирования данных.
    Использует consistent hashing для распределения.
    """

    def __init__(self, shard_configs: list[dict]):
        self.shards: Dict[int, Any] = {}
        self.num_shards = len(shard_configs)

        for i, config in enumerate(shard_configs):
            engine = create_engine(
                config['connection_string'],
                pool_size=10
            )
            self.shards[i] = {
                'engine': engine,
                'session_factory': sessionmaker(bind=engine)
            }

    def get_shard_id(self, shard_key: int) -> int:
        """
        Определяет номер шарда для ключа.
        Простой способ: остаток от деления.
        """
        return shard_key % self.num_shards

    def get_shard_id_consistent(self, shard_key: str) -> int:
        """
        Consistent hashing - более устойчив к добавлению шардов.
        """
        hash_value = int(md5(str(shard_key).encode()).hexdigest(), 16)
        return hash_value % self.num_shards

    def get_session(self, shard_key: int) -> Session:
        """Получает сессию для конкретного шарда"""
        shard_id = self.get_shard_id(shard_key)
        return self.shards[shard_id]['session_factory']()

    def execute_on_all_shards(self, query_func):
        """Выполняет запрос на всех шардах (scatter-gather)"""
        results = []
        for shard_id, shard in self.shards.items():
            session = shard['session_factory']()
            try:
                result = query_func(session)
                results.extend(result)
            finally:
                session.close()
        return results

# Конфигурация
shard_configs = [
    {'connection_string': 'postgresql://user:pass@shard0.internal:5432/app'},
    {'connection_string': 'postgresql://user:pass@shard1.internal:5432/app'},
    {'connection_string': 'postgresql://user:pass@shard2.internal:5432/app'},
    {'connection_string': 'postgresql://user:pass@shard3.internal:5432/app'},
]

router = ShardRouter(shard_configs)

# Использование
def get_user_orders(user_id: int):
    """Запрос идёт на конкретный шард"""
    session = router.get_session(user_id)
    try:
        return session.query(Order).filter(Order.user_id == user_id).all()
    finally:
        session.close()

def get_all_orders_count():
    """Агрегация со всех шардов"""
    def count_query(session):
        return [session.query(Order).count()]

    counts = router.execute_on_all_shards(count_query)
    return sum(counts)
```

---

## 5. Сравнительная таблица подходов

| Критерий | Вертикальное | Горизонтальное | Диагональное |
|----------|--------------|----------------|--------------|
| **Сложность** | Низкая | Высокая | Средняя |
| **Стоимость** | Экспоненциальная | Линейная | Оптимальная |
| **Потолок роста** | Ограничен | Практически нет | Практически нет |
| **Downtime** | При апгрейде | Минимальный | Минимальный |
| **Отказоустойчивость** | Низкая (SPOF) | Высокая | Высокая |
| **Изменение кода** | Не требуется | Часто требуется | Иногда требуется |
| **Управление** | Простое | Сложное | Среднее |

---

## 6. Когда какой подход использовать

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                    ВЫБОР СТРАТЕГИИ МАСШТАБИРОВАНИЯ                          │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                             │
│   Начинающий проект / MVP:                                                  │
│   ──────────────────────────                                                │
│   → Вертикальное масштабирование                                           │
│   → Простота важнее масштабируемости                                       │
│   → Один сервер, монолит                                                   │
│                                                                             │
│   Растущий проект (10K-100K пользователей):                                │
│   ──────────────────────────────────────────                                │
│   → Диагональное масштабирование                                           │
│   → Выжимаем максимум из текущих серверов                                  │
│   → Добавляем read replicas для БД                                         │
│   → Stateless приложение + Redis                                           │
│                                                                             │
│   Большой проект (100K+ пользователей):                                    │
│   ────────────────────────────────────────                                  │
│   → Горизонтальное масштабирование                                         │
│   → Kubernetes / Auto-scaling groups                                       │
│   → Шардинг базы данных                                                    │
│   → CDN для статики                                                        │
│   → Микросервисы                                                           │
│                                                                             │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Чек-лист для выбора стратегии

```python
# scaling_decision.py - Помощник выбора стратегии

def choose_scaling_strategy(project_context: dict) -> str:
    """
    Определяет оптимальную стратегию масштабирования.
    """

    users = project_context.get('monthly_active_users', 0)
    budget = project_context.get('budget', 'low')  # low, medium, high
    team_size = project_context.get('team_size', 1)
    downtime_tolerance = project_context.get('downtime_tolerance', 'high')

    # MVP / Стартап
    if users < 10000 and team_size < 5:
        return """
        РЕКОМЕНДАЦИЯ: Вертикальное масштабирование

        - Используйте один мощный сервер
        - Монолитная архитектура
        - PostgreSQL без репликации
        - Простой деплой

        Когда переходить: CPU > 70% постоянно, отклик > 500ms
        """

    # Растущий проект
    if users < 100000:
        return """
        РЕКОМЕНДАЦИЯ: Диагональное масштабирование

        1. Увеличьте сервер до максимума
        2. Добавьте read replicas (2-3 шт)
        3. Redis для кэша и сессий
        4. CDN для статики
        5. При необходимости - 2-3 app сервера + LB

        Когда переходить: Вертикальный потолок достигнут
        """

    # Крупный проект
    return """
    РЕКОМЕНДАЦИЯ: Горизонтальное масштабирование

    1. Kubernetes с auto-scaling
    2. Шардинг базы данных
    3. Микросервисная архитектура
    4. Event-driven (Kafka/RabbitMQ)
    5. Multi-region деплой
    6. Chaos engineering

    Важно: Требуется команда DevOps/SRE
    """
```

---

## Best Practices

### 1. Планируйте масштабирование заранее

```python
# Пишите stateless код с самого начала
class UserService:
    def __init__(self, cache: Redis, db: Database):
        # Внедрение зависимостей - легко заменить компоненты
        self.cache = cache
        self.db = db

    async def get_user(self, user_id: int) -> User:
        # Кэш - можно заменить на распределённый
        cached = await self.cache.get(f"user:{user_id}")
        if cached:
            return User.from_json(cached)

        # БД - можно заменить на шардированную
        user = await self.db.get_user(user_id)
        await self.cache.set(f"user:{user_id}", user.to_json())
        return user
```

### 2. Мониторинг перед масштабированием

```yaml
# prometheus.yml - Метрики для принятия решений
scrape_configs:
  - job_name: 'app'
    static_configs:
      - targets: ['app:8000']

# Алерты для масштабирования
groups:
  - name: scaling_alerts
    rules:
      - alert: HighCPU
        expr: avg(rate(process_cpu_seconds_total[5m])) > 0.8
        for: 10m
        labels:
          severity: warning
        annotations:
          summary: "Рассмотрите масштабирование - CPU > 80%"
```

### 3. Тестируйте масштабируемость

```python
# locustfile.py - Нагрузочное тестирование
from locust import HttpUser, task, between

class LoadTest(HttpUser):
    wait_time = between(1, 3)

    @task(10)
    def read_endpoint(self):
        """Симуляция нагрузки на чтение"""
        self.client.get("/api/users/123")

    @task(1)
    def write_endpoint(self):
        """Симуляция нагрузки на запись"""
        self.client.post("/api/users", json={"name": "Test"})
```

---

## Частые ошибки

### 1. Преждевременное масштабирование

```python
# ПЛОХО: Микросервисы для MVP
# - 10 сервисов для 100 пользователей
# - Kubernetes для статического сайта

# ХОРОШО: Начинаем с монолита
# - Один сервер, простая архитектура
# - Масштабируем когда есть реальная нагрузка
```

### 2. Игнорирование состояния приложения

```python
# ПЛОХО: Хранение состояния в памяти
class BadSessionStore:
    sessions = {}  # Потеряется при рестарте/масштабировании

    def set(self, key, value):
        self.sessions[key] = value

# ХОРОШО: Внешнее хранилище
class GoodSessionStore:
    def __init__(self, redis: Redis):
        self.redis = redis

    def set(self, key, value):
        self.redis.set(key, value)
```

### 3. Отсутствие health checks

```python
# ОБЯЗАТЕЛЬНО для горизонтального масштабирования
@app.get("/health")
async def health_check():
    """Health check для load balancer"""
    # Проверяем зависимости
    try:
        await db.execute("SELECT 1")
        await redis.ping()
        return {"status": "healthy"}
    except Exception as e:
        raise HTTPException(status_code=503, detail=str(e))
```

---

## Резюме

1. **Вертикальное масштабирование** — просто, но ограничено. Подходит для начала.

2. **Горизонтальное масштабирование** — мощно, но сложно. Требует stateless архитектуры.

3. **Диагональное масштабирование** — оптимальный баланс. Комбинируем подходы.

4. **Масштабирование БД** — репликация для чтения, шардинг для записи.

5. **Планируйте заранее** — пишите stateless код, используйте внешние хранилища.

6. **Мониторинг** — масштабируйте на основе данных, не интуиции.

7. **Тестируйте** — нагрузочные тесты до и после масштабирования.
