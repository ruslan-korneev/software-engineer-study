# Latency vs Throughput (Задержка vs Пропускная способность)

## Введение

Латентность и пропускная способность — два фундаментальных понятия в проектировании систем. Они тесно связаны, но описывают разные аспекты производительности. Понимание этих концепций критически важно для создания эффективных и масштабируемых backend-систем.

---

## 1. Латентность (Latency)

### Определение

**Латентность** — это время, которое проходит от момента отправки запроса до получения ответа. Измеряется в единицах времени (миллисекунды, микросекунды, наносекунды).

```
Латентность = Время получения ответа - Время отправки запроса
```

### Компоненты латентности

Общая латентность запроса складывается из нескольких составляющих:

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Полная латентность запроса                       │
├─────────┬──────────┬─────────────┬──────────┬─────────┬────────────┤
│ Сетевая │ Очередь  │  Обработка  │ Доступ   │ Сетевая │  Клиент    │
│ туда    │ ожидания │  на сервере │ к данным │ обратно │ обработка  │
└─────────┴──────────┴─────────────┴──────────┴─────────┴────────────┘
```

1. **Network Latency (Сетевая задержка)** — время передачи данных по сети
2. **Queueing Latency (Задержка в очереди)** — ожидание обработки
3. **Processing Latency (Задержка обработки)** — время выполнения бизнес-логики
4. **I/O Latency (Задержка ввода-вывода)** — чтение/запись данных

### Пример измерения латентности

```python
import time
import requests

def measure_latency(url: str, num_requests: int = 100) -> dict:
    """Измерение латентности HTTP-запросов."""
    latencies = []

    for _ in range(num_requests):
        start = time.perf_counter()
        response = requests.get(url)
        end = time.perf_counter()

        latency_ms = (end - start) * 1000
        latencies.append(latency_ms)

    latencies.sort()

    return {
        "min": latencies[0],
        "max": latencies[-1],
        "avg": sum(latencies) / len(latencies),
        "p50": latencies[len(latencies) // 2],
        "p95": latencies[int(len(latencies) * 0.95)],
        "p99": latencies[int(len(latencies) * 0.99)],
    }

# Пример использования
result = measure_latency("https://api.example.com/health")
print(f"P50: {result['p50']:.2f}ms, P99: {result['p99']:.2f}ms")
```

---

## 2. Пропускная способность (Throughput)

### Определение

**Пропускная способность** — это количество операций (запросов, транзакций, байтов), которые система может обработать за единицу времени.

```
Throughput = Количество операций / Время
```

### Единицы измерения

| Контекст | Единица | Пример |
|----------|---------|--------|
| Веб-сервер | RPS (requests per second) | 10,000 RPS |
| База данных | TPS (transactions per second) | 5,000 TPS |
| Сеть | Mbps, Gbps | 1 Gbps |
| Очередь сообщений | messages/sec | 100,000 msg/s |
| Диск | IOPS (I/O operations per second) | 10,000 IOPS |

### Пример измерения пропускной способности

```python
import asyncio
import aiohttp
import time

async def measure_throughput(url: str, duration_sec: int = 10) -> float:
    """Измерение пропускной способности в RPS."""

    async def make_request(session: aiohttp.ClientSession):
        async with session.get(url) as response:
            await response.read()

    completed = 0
    start_time = time.time()

    async with aiohttp.ClientSession() as session:
        while time.time() - start_time < duration_sec:
            # Отправляем 100 параллельных запросов
            tasks = [make_request(session) for _ in range(100)]
            await asyncio.gather(*tasks, return_exceptions=True)
            completed += 100

    elapsed = time.time() - start_time
    throughput = completed / elapsed

    return throughput

# Пример использования
rps = asyncio.run(measure_throughput("http://localhost:8080/api"))
print(f"Throughput: {rps:.0f} RPS")
```

---

## 3. Связь и различия

### Аналогия с автострадой

Представьте автостраду:

| Концепция | Аналогия |
|-----------|----------|
| **Латентность** | Время поездки из точки A в точку B |
| **Пропускная способность** | Количество машин, проезжающих за час |
| **Bandwidth (Ширина канала)** | Количество полос на дороге |

```
┌────────────────────────────────────────────────────────────────┐
│                        АВТОСТРАДА                              │
│  ═══════════════════════════════════════════════════════════  │
│  ═══════════════════════════════════════════════════════════  │
│  ═══════════════════════════════════════════════════════════  │
│                                                                │
│  Латентность: 30 минут от A до B                              │
│  Пропускная способность: 3000 машин/час                       │
│  Bandwidth: 3 полосы                                           │
└────────────────────────────────────────────────────────────────┘
```

### Формула Литтла (Little's Law)

Связь между латентностью и пропускной способностью:

```
L = λ × W

где:
L = среднее число запросов в системе
λ = пропускная способность (throughput)
W = среднее время пребывания в системе (latency)
```

Следствие: при фиксированном числе одновременных соединений:
- **Уменьшение латентности → увеличение пропускной способности**
- **Увеличение латентности → уменьшение пропускной способности**

### Ключевые различия

| Характеристика | Латентность | Пропускная способность |
|----------------|-------------|------------------------|
| Что измеряет | Время отклика | Объём работы |
| Единицы | ms, μs, ns | ops/sec, RPS, TPS |
| Фокус | Один запрос | Множество запросов |
| Оптимизация | Скорость | Эффективность |
| Для пользователя | "Как быстро?" | "Сколько одновременно?" |

---

## 4. Типичные значения латентности

### Таблица латентности (Numbers Every Programmer Should Know)

```
┌─────────────────────────────────────────────────────────────────────┐
│                    Типичные значения латентности                    │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│ L1 cache reference                              0.5 ns              │
│ Branch mispredict                               5   ns              │
│ L2 cache reference                              7   ns              │
│ Mutex lock/unlock                              25   ns              │
│ Main memory reference                         100   ns              │
│ Compress 1KB with Zippy                     3,000   ns  =    3 μs   │
│ Send 1KB over 1 Gbps network               10,000   ns  =   10 μs   │
│ Read 4KB randomly from SSD                150,000   ns  =  150 μs   │
│ Read 1MB sequentially from memory         250,000   ns  =  250 μs   │
│ Round trip within datacenter              500,000   ns  =  500 μs   │
│ Read 1MB sequentially from SSD          1,000,000   ns  =    1 ms   │
│ HDD seek                               10,000,000   ns  =   10 ms   │
│ Read 1MB sequentially from HDD         20,000,000   ns  =   20 ms   │
│ Send packet CA → Netherlands → CA     150,000,000   ns  =  150 ms   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Визуализация масштаба

Если L1 cache = 1 секунда:
```
L1 cache                    1 секунда
L2 cache                   14 секунд
Main memory                3.5 минуты
SSD random read            5 минут
Datacenter round trip     16 минут
SSD sequential 1MB        33 минуты
HDD seek                   5.5 часов
HDD sequential 1MB        11 часов
Intercontinental packet    3.5 дня
```

### Латентность по типам операций

#### Сетевые операции
```python
# Типичные значения для веб-приложений
LATENCY_BENCHMARKS = {
    # Внутри датацентра
    "localhost": "< 0.1 ms",
    "same_rack": "0.1 - 0.5 ms",
    "same_datacenter": "0.5 - 1 ms",
    "different_datacenter": "10 - 100 ms",
    "cross_continent": "100 - 300 ms",

    # База данных
    "redis_get": "0.1 - 0.5 ms",
    "postgresql_simple_query": "1 - 5 ms",
    "postgresql_complex_query": "10 - 100 ms",
    "elasticsearch_search": "5 - 50 ms",

    # Внешние сервисы
    "third_party_api": "50 - 500 ms",
    "payment_gateway": "100 - 2000 ms",
}
```

#### Операции с хранилищами

```python
# IOPS и латентность разных типов хранилищ
STORAGE_CHARACTERISTICS = {
    "RAM": {
        "latency": "100 ns",
        "iops": "10,000,000+",
        "throughput": "50+ GB/s"
    },
    "NVMe SSD": {
        "latency": "10-100 μs",
        "iops": "500,000 - 1,000,000",
        "throughput": "3-7 GB/s"
    },
    "SATA SSD": {
        "latency": "50-150 μs",
        "iops": "50,000 - 100,000",
        "throughput": "500 MB/s"
    },
    "HDD": {
        "latency": "5-10 ms",
        "iops": "100-200",
        "throughput": "100-200 MB/s"
    },
    "Network Storage": {
        "latency": "1-10 ms",
        "iops": "10,000 - 100,000",
        "throughput": "1-10 GB/s"
    }
}
```

---

## 5. Измерение латентности: перцентили

### Почему не среднее значение?

Среднее значение (average) скрывает выбросы и не показывает реальную картину:

```
Запросы: 10ms, 10ms, 10ms, 10ms, 10ms, 10ms, 10ms, 10ms, 10ms, 1000ms

Average: 109 ms  (не отражает типичный опыт пользователя)
Median (P50): 10 ms  (50% пользователей получают такую латентность)
P99: 1000 ms  (1% пользователей страдает)
```

### Перцентили

| Перцентиль | Значение | Применение |
|------------|----------|------------|
| **P50 (медиана)** | 50% запросов быстрее | "Типичный" пользователь |
| **P90** | 90% запросов быстрее | Большинство пользователей |
| **P95** | 95% запросов быстрее | Почти все пользователи |
| **P99** | 99% запросов быстрее | "Проблемные" случаи |
| **P99.9** | 99.9% запросов быстрее | Экстремальные случаи |

### Код для расчёта перцентилей

```python
import numpy as np
from dataclasses import dataclass
from typing import List

@dataclass
class LatencyStats:
    min: float
    max: float
    mean: float
    p50: float
    p90: float
    p95: float
    p99: float
    p999: float

def calculate_percentiles(latencies: List[float]) -> LatencyStats:
    """Расчёт статистики латентности."""
    arr = np.array(latencies)

    return LatencyStats(
        min=np.min(arr),
        max=np.max(arr),
        mean=np.mean(arr),
        p50=np.percentile(arr, 50),
        p90=np.percentile(arr, 90),
        p95=np.percentile(arr, 95),
        p99=np.percentile(arr, 99),
        p999=np.percentile(arr, 99.9)
    )

# Пример
latencies = [10, 12, 11, 15, 13, 10, 11, 200, 12, 10, 14, 11, 500]
stats = calculate_percentiles(latencies)

print(f"""
Latency Statistics:
  Min:   {stats.min:.1f} ms
  Mean:  {stats.mean:.1f} ms
  P50:   {stats.p50:.1f} ms
  P95:   {stats.p95:.1f} ms
  P99:   {stats.p99:.1f} ms
  Max:   {stats.max:.1f} ms
""")
```

### Гистограмма латентности (HdrHistogram)

```python
from collections import Counter
import math

def print_latency_histogram(latencies: List[float], buckets: int = 20):
    """Вывод ASCII гистограммы латентности."""
    min_lat = min(latencies)
    max_lat = max(latencies)
    bucket_size = (max_lat - min_lat) / buckets

    histogram = Counter()
    for lat in latencies:
        bucket = int((lat - min_lat) / bucket_size)
        bucket = min(bucket, buckets - 1)
        histogram[bucket] += 1

    max_count = max(histogram.values())
    bar_width = 50

    print("\nLatency Distribution:")
    print("-" * 70)

    for i in range(buckets):
        bucket_start = min_lat + i * bucket_size
        bucket_end = bucket_start + bucket_size
        count = histogram[i]
        bar_len = int(count / max_count * bar_width)

        print(f"{bucket_start:8.1f}-{bucket_end:8.1f}ms | {'█' * bar_len} ({count})")
```

### SLA/SLO на основе перцентилей

```yaml
# Пример SLO (Service Level Objective)
service_level_objectives:
  api_latency:
    - metric: "http_request_duration_seconds"
      percentile: 50
      threshold: 100ms

    - metric: "http_request_duration_seconds"
      percentile: 95
      threshold: 500ms

    - metric: "http_request_duration_seconds"
      percentile: 99
      threshold: 1000ms

  availability:
    - metric: "successful_requests / total_requests"
      threshold: 99.9%
```

---

## 6. Способы уменьшения латентности

### 6.1. Кэширование

```python
from functools import lru_cache
import redis
import json

# === In-memory кэширование ===
@lru_cache(maxsize=1000)
def get_user_expensive(user_id: int) -> dict:
    """Кэширование результата в памяти."""
    return expensive_database_query(user_id)

# === Redis кэширование ===
class CacheService:
    def __init__(self):
        self.redis = redis.Redis(host='localhost', port=6379)
        self.default_ttl = 3600  # 1 час

    def get_or_compute(self, key: str, compute_fn, ttl: int = None):
        """Получить из кэша или вычислить."""
        cached = self.redis.get(key)
        if cached:
            return json.loads(cached)

        result = compute_fn()
        self.redis.setex(
            key,
            ttl or self.default_ttl,
            json.dumps(result)
        )
        return result

# Использование
cache = CacheService()
user = cache.get_or_compute(
    f"user:{user_id}",
    lambda: fetch_user_from_db(user_id),
    ttl=300
)
```

### 6.2. Уменьшение сетевых round-trips

```python
# ❌ Плохо: N+1 запросов
async def get_users_with_orders_bad(user_ids: List[int]):
    users = []
    for user_id in user_ids:
        user = await fetch_user(user_id)
        orders = await fetch_orders(user_id)
        users.append({"user": user, "orders": orders})
    return users

# ✅ Хорошо: 2 запроса с batch/join
async def get_users_with_orders_good(user_ids: List[int]):
    # Один запрос для всех пользователей
    users = await fetch_users_batch(user_ids)
    # Один запрос для всех заказов
    orders = await fetch_orders_batch(user_ids)

    # Объединение в памяти
    orders_by_user = group_by(orders, 'user_id')
    return [
        {"user": user, "orders": orders_by_user.get(user.id, [])}
        for user in users
    ]
```

### 6.3. Географическая близость

```python
# Edge computing / CDN для статики и API
CDN_CONFIG = {
    "static_assets": {
        "provider": "cloudflare",
        "edge_caching": True,
        "ttl": 86400,  # 24 часа
    },
    "api_responses": {
        "edge_caching": True,
        "cache_key": ["path", "query_params", "accept_encoding"],
        "ttl": 60,  # 1 минута
    }
}

# Geo-routing для баз данных
DATABASE_REPLICAS = {
    "us-east-1": "primary",
    "us-west-2": "read-replica",
    "eu-west-1": "read-replica",
    "ap-southeast-1": "read-replica",
}
```

### 6.4. Асинхронная обработка

```python
import asyncio
from typing import List

# ❌ Последовательные запросы
async def fetch_data_sequential(ids: List[int]) -> List[dict]:
    results = []
    for id in ids:
        result = await fetch_from_service(id)  # Ждём каждый запрос
        results.append(result)
    return results
# Общее время: sum(latencies) = 100ms * 10 = 1000ms

# ✅ Параллельные запросы
async def fetch_data_parallel(ids: List[int]) -> List[dict]:
    tasks = [fetch_from_service(id) for id in ids]
    return await asyncio.gather(*tasks)
# Общее время: max(latencies) = 100ms
```

### 6.5. Предзагрузка (Prefetching)

```python
class PrefetchCache:
    """Предзагрузка данных на основе паттернов использования."""

    def __init__(self):
        self.cache = {}
        self.access_patterns = defaultdict(Counter)

    async def get(self, key: str, user_id: str) -> Any:
        # Записываем паттерн доступа
        self.record_access(user_id, key)

        if key in self.cache:
            return self.cache[key]

        result = await self.fetch(key)
        self.cache[key] = result

        # Предзагружаем связанные данные
        await self.prefetch_related(user_id, key)

        return result

    async def prefetch_related(self, user_id: str, key: str):
        """Предзагрузка данных, которые пользователь вероятно запросит."""
        likely_next_keys = self.predict_next_keys(user_id, key)

        for next_key in likely_next_keys[:5]:
            if next_key not in self.cache:
                asyncio.create_task(self.warm_cache(next_key))
```

### 6.6. Оптимизация запросов к БД

```sql
-- ❌ Плохо: полное сканирование таблицы
SELECT * FROM orders WHERE status = 'pending';

-- ✅ Хорошо: использование индекса
CREATE INDEX idx_orders_status ON orders(status);
SELECT id, user_id, total FROM orders WHERE status = 'pending';

-- ❌ Плохо: N+1 запросов
-- Python: for order in orders: fetch_user(order.user_id)

-- ✅ Хорошо: JOIN
SELECT o.id, o.total, u.name
FROM orders o
JOIN users u ON o.user_id = u.id
WHERE o.status = 'pending';
```

---

## 7. Способы увеличения пропускной способности

### 7.1. Горизонтальное масштабирование

```python
# Load Balancer конфигурация (nginx)
"""
upstream backend {
    least_conn;  # Выбор сервера с наименьшим числом соединений
    server backend1.example.com:8080 weight=5;
    server backend2.example.com:8080 weight=5;
    server backend3.example.com:8080 weight=5;

    keepalive 32;  # Пул постоянных соединений
}

server {
    listen 80;

    location /api {
        proxy_pass http://backend;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }
}
"""
```

### 7.2. Пулы соединений

```python
from sqlalchemy import create_engine
from sqlalchemy.pool import QueuePool

# Connection pool для PostgreSQL
engine = create_engine(
    "postgresql://user:pass@localhost/db",
    poolclass=QueuePool,
    pool_size=20,           # Базовый размер пула
    max_overflow=10,        # Дополнительные соединения при пиках
    pool_timeout=30,        # Таймаут ожидания соединения
    pool_recycle=1800,      # Переподключение каждые 30 минут
    pool_pre_ping=True,     # Проверка соединения перед использованием
)

# Redis connection pool
import redis

redis_pool = redis.ConnectionPool(
    host='localhost',
    port=6379,
    max_connections=50,
    socket_timeout=5,
    socket_connect_timeout=5,
)

redis_client = redis.Redis(connection_pool=redis_pool)
```

### 7.3. Батчинг (Batching)

```python
import asyncio
from dataclasses import dataclass
from typing import Dict, List, Any

@dataclass
class BatchRequest:
    id: str
    future: asyncio.Future

class BatchProcessor:
    """Группировка запросов для обработки пакетами."""

    def __init__(self, batch_size: int = 100, max_delay: float = 0.01):
        self.batch_size = batch_size
        self.max_delay = max_delay
        self.pending: Dict[str, BatchRequest] = {}
        self._task = None

    async def get(self, id: str) -> Any:
        """Получение элемента (запрос будет сгруппирован)."""
        loop = asyncio.get_event_loop()
        future = loop.create_future()

        self.pending[id] = BatchRequest(id=id, future=future)

        if len(self.pending) >= self.batch_size:
            await self._flush()
        elif self._task is None:
            self._task = asyncio.create_task(self._delayed_flush())

        return await future

    async def _delayed_flush(self):
        await asyncio.sleep(self.max_delay)
        await self._flush()
        self._task = None

    async def _flush(self):
        if not self.pending:
            return

        batch = self.pending
        self.pending = {}

        # Один запрос для всего батча
        ids = [req.id for req in batch.values()]
        results = await self._batch_fetch(ids)

        for id, req in batch.items():
            req.future.set_result(results.get(id))

    async def _batch_fetch(self, ids: List[str]) -> Dict[str, Any]:
        # SELECT * FROM items WHERE id IN (...)
        return await db.fetch_many(ids)

# Использование
batcher = BatchProcessor(batch_size=50, max_delay=0.005)

# Эти запросы будут сгруппированы в один SQL-запрос
results = await asyncio.gather(
    batcher.get("id1"),
    batcher.get("id2"),
    batcher.get("id3"),
    # ...
)
```

### 7.4. Асинхронная обработка с очередями

```python
from celery import Celery
import asyncio

# Celery для фоновых задач
app = Celery('tasks', broker='redis://localhost:6379/0')

@app.task
def process_order(order_id: int):
    """Фоновая обработка заказа."""
    order = db.get_order(order_id)

    # Длительные операции
    validate_inventory(order)
    process_payment(order)
    send_confirmation_email(order)
    update_analytics(order)

    return {"status": "completed", "order_id": order_id}

# API эндпоинт - мгновенный ответ
@app.route("/orders", methods=["POST"])
async def create_order():
    order = await save_order(request.json)

    # Отправляем в очередь для асинхронной обработки
    process_order.delay(order.id)

    # Мгновенный ответ клиенту
    return {"order_id": order.id, "status": "processing"}
```

### 7.5. Компрессия данных

```python
import gzip
import lz4.frame
from functools import wraps

def compress_response(f):
    """Декоратор для сжатия ответов."""
    @wraps(f)
    async def wrapper(*args, **kwargs):
        result = await f(*args, **kwargs)

        accept_encoding = request.headers.get('Accept-Encoding', '')

        if 'gzip' in accept_encoding:
            compressed = gzip.compress(result.encode())
            return Response(
                compressed,
                headers={'Content-Encoding': 'gzip'}
            )

        return result
    return wrapper

# Сравнение алгоритмов
def benchmark_compression(data: bytes):
    """Сравнение скорости и степени сжатия."""
    import time

    results = {}

    # gzip (хорошее сжатие, медленнее)
    start = time.time()
    gzip_data = gzip.compress(data)
    results['gzip'] = {
        'time': time.time() - start,
        'ratio': len(gzip_data) / len(data)
    }

    # lz4 (быстрое сжатие, меньше степень)
    start = time.time()
    lz4_data = lz4.frame.compress(data)
    results['lz4'] = {
        'time': time.time() - start,
        'ratio': len(lz4_data) / len(data)
    }

    return results
```

### 7.6. Оптимизация протокола

```python
# HTTP/2 с multiplexing
# Один TCP-connection для многих запросов

# gRPC с Protocol Buffers (бинарный формат)
"""
syntax = "proto3";

service UserService {
    rpc GetUsers(GetUsersRequest) returns (stream User);
    rpc BatchGetUsers(BatchGetUsersRequest) returns (BatchGetUsersResponse);
}

message User {
    int64 id = 1;
    string name = 2;
    string email = 3;
}
"""

# Сравнение размера данных
import json
import msgpack

data = {"users": [{"id": i, "name": f"User {i}"} for i in range(1000)]}

json_size = len(json.dumps(data))        # ~35 KB
msgpack_size = len(msgpack.packb(data))  # ~20 KB (на 43% меньше)
```

---

## 8. Trade-offs между латентностью и пропускной способностью

### 8.1. Батчинг: пропускная способность vs латентность

```
┌─────────────────────────────────────────────────────────────────┐
│                                                                 │
│  Без батчинга:                                                  │
│  ┌───┐ ┌───┐ ┌───┐ ┌───┐ ┌───┐                                 │
│  │ 1 │ │ 2 │ │ 3 │ │ 4 │ │ 5 │                                 │
│  └───┘ └───┘ └───┘ └───┘ └───┘                                 │
│  Латентность: 10ms каждый                                       │
│  Пропускная способность: 100 RPS (overhead на каждый запрос)    │
│                                                                 │
│  С батчингом:                                                   │
│  ┌─────────────────────────────────────┐                       │
│  │   1   │   2   │   3   │   4   │  5  │                       │
│  └─────────────────────────────────────┘                       │
│  Латентность: 50ms + wait time (запрос 1 ждёт, пока соберётся  │
│               батч)                                             │
│  Пропускная способность: 500 RPS (меньше overhead)             │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

```python
class AdaptiveBatcher:
    """Адаптивный батчер с балансом между латентностью и throughput."""

    def __init__(
        self,
        min_batch_size: int = 1,
        max_batch_size: int = 100,
        target_latency_ms: float = 50
    ):
        self.min_batch_size = min_batch_size
        self.max_batch_size = max_batch_size
        self.target_latency_ms = target_latency_ms
        self.current_batch_size = min_batch_size
        self.latency_history = []

    def adjust_batch_size(self, observed_latency_ms: float):
        """Автоподстройка размера батча."""
        self.latency_history.append(observed_latency_ms)

        if len(self.latency_history) < 10:
            return

        avg_latency = sum(self.latency_history[-10:]) / 10

        if avg_latency < self.target_latency_ms * 0.8:
            # Есть запас — увеличиваем batch для throughput
            self.current_batch_size = min(
                self.current_batch_size + 5,
                self.max_batch_size
            )
        elif avg_latency > self.target_latency_ms:
            # Превышаем цель — уменьшаем batch для latency
            self.current_batch_size = max(
                self.current_batch_size - 5,
                self.min_batch_size
            )
```

### 8.2. Кэширование: свежесть vs скорость

```python
class CacheStrategy:
    """Различные стратегии кэширования."""

    @staticmethod
    def write_through(key: str, value: Any, db, cache):
        """Синхронная запись в кэш и БД.

        + Консистентность
        - Высокая латентность записи
        """
        db.write(key, value)
        cache.set(key, value)

    @staticmethod
    def write_behind(key: str, value: Any, db, cache, queue):
        """Асинхронная запись в БД через очередь.

        + Низкая латентность записи
        - Риск потери данных при сбое
        """
        cache.set(key, value)
        queue.push({"key": key, "value": value})

    @staticmethod
    def cache_aside_with_stale(key: str, db, cache, stale_ttl: int):
        """Возврат устаревших данных с фоновым обновлением.

        + Очень низкая латентность (всегда из кэша)
        - Данные могут быть устаревшими
        """
        cached = cache.get(key)

        if cached and not cached.is_expired():
            return cached.value

        if cached and cached.is_stale():
            # Возвращаем stale данные, обновляем в фоне
            asyncio.create_task(refresh_cache(key, db, cache))
            return cached.value

        # Нет данных — синхронная загрузка
        value = db.read(key)
        cache.set(key, value, stale_ttl=stale_ttl)
        return value
```

### 8.3. Репликация: консистентность vs производительность

```
┌──────────────────────────────────────────────────────────────────────┐
│                     Синхронная репликация                            │
│                                                                      │
│  Client ──► Primary ──► Replica 1 ──┐                               │
│                    └──► Replica 2 ──┼──► ACK ──► Client             │
│                    └──► Replica 3 ──┘                               │
│                                                                      │
│  + Строгая консистентность                                          │
│  - Латентность = max(time to all replicas)                          │
│  - Throughput ограничен самой медленной репликой                    │
├──────────────────────────────────────────────────────────────────────┤
│                     Асинхронная репликация                           │
│                                                                      │
│  Client ──► Primary ──► ACK ──► Client                              │
│                    └──► Replica 1 (async)                           │
│                    └──► Replica 2 (async)                           │
│                    └──► Replica 3 (async)                           │
│                                                                      │
│  + Низкая латентность                                               │
│  + Высокий throughput                                               │
│  - Eventual consistency (риск потери данных при сбое)               │
└──────────────────────────────────────────────────────────────────────┘
```

### 8.4. Компрессия: CPU vs сеть

```python
import time
import gzip
import lz4.frame

def analyze_compression_tradeoff(data: bytes, network_speed_mbps: float):
    """Анализ trade-off компрессии для разных сетевых условий."""

    results = []

    for name, compress_fn in [
        ("none", lambda x: x),
        ("lz4", lz4.frame.compress),
        ("gzip-1", lambda x: gzip.compress(x, compresslevel=1)),
        ("gzip-9", lambda x: gzip.compress(x, compresslevel=9)),
    ]:
        start = time.time()
        compressed = compress_fn(data)
        compress_time = time.time() - start

        network_time = (len(compressed) * 8) / (network_speed_mbps * 1_000_000)
        total_time = compress_time + network_time

        results.append({
            "method": name,
            "original_size": len(data),
            "compressed_size": len(compressed),
            "ratio": len(compressed) / len(data),
            "compress_time_ms": compress_time * 1000,
            "network_time_ms": network_time * 1000,
            "total_time_ms": total_time * 1000,
        })

    return results

# Для медленных сетей — сжатие выгодно
# Для быстрых сетей (datacenter) — сжатие может быть лишним
```

---

## 9. Практические примеры

### Пример 1: Оптимизация API эндпоинта

```python
from fastapi import FastAPI, Depends
from sqlalchemy.ext.asyncio import AsyncSession
from functools import lru_cache
import aioredis

app = FastAPI()

# ❌ До оптимизации
# Латентность: 500ms, Throughput: 50 RPS
@app.get("/products/{product_id}")
async def get_product_slow(product_id: int, db: AsyncSession = Depends(get_db)):
    # 1. Запрос продукта (100ms)
    product = await db.execute(
        select(Product).where(Product.id == product_id)
    )

    # 2. Запрос категории отдельно (100ms)
    category = await db.execute(
        select(Category).where(Category.id == product.category_id)
    )

    # 3. Запрос отзывов отдельно (200ms)
    reviews = await db.execute(
        select(Review).where(Review.product_id == product_id)
    )

    # 4. Расчёт рейтинга (100ms CPU)
    rating = calculate_rating(reviews)

    return {
        "product": product,
        "category": category,
        "reviews": reviews,
        "rating": rating
    }


# ✅ После оптимизации
# Латентность: 50ms, Throughput: 500 RPS
@app.get("/products/{product_id}")
async def get_product_fast(
    product_id: int,
    redis: aioredis.Redis = Depends(get_redis),
    db: AsyncSession = Depends(get_db)
):
    # 1. Проверяем кэш (0.5ms)
    cache_key = f"product:{product_id}"
    cached = await redis.get(cache_key)
    if cached:
        return json.loads(cached)

    # 2. Один запрос с JOIN (50ms)
    result = await db.execute(
        select(Product, Category, func.avg(Review.rating))
        .join(Category)
        .outerjoin(Review)
        .where(Product.id == product_id)
        .group_by(Product.id, Category.id)
    )

    product, category, avg_rating = result.first()

    response = {
        "product": product.dict(),
        "category": category.dict(),
        "rating": float(avg_rating or 0)
    }

    # 3. Сохраняем в кэш (0.5ms, async)
    await redis.setex(cache_key, 300, json.dumps(response))

    return response
```

### Пример 2: Балансировка нагрузки с учётом латентности

```python
import asyncio
import random
from dataclasses import dataclass, field
from typing import List, Dict
import time

@dataclass
class Backend:
    host: str
    port: int
    weight: int = 1
    current_connections: int = 0
    avg_latency_ms: float = 0
    latency_samples: List[float] = field(default_factory=list)

    def record_latency(self, latency_ms: float):
        self.latency_samples.append(latency_ms)
        if len(self.latency_samples) > 100:
            self.latency_samples.pop(0)
        self.avg_latency_ms = sum(self.latency_samples) / len(self.latency_samples)

class LatencyAwareLoadBalancer:
    """Load balancer с учётом латентности бэкендов."""

    def __init__(self, backends: List[Backend]):
        self.backends = backends

    def select_backend(self) -> Backend:
        """Выбор бэкенда с наименьшей weighted латентностью."""
        # Формула: score = latency * (1 + connections/10)
        # Учитываем и латентность, и текущую нагрузку

        best_backend = None
        best_score = float('inf')

        for backend in self.backends:
            if backend.avg_latency_ms == 0:
                # Новый бэкенд — даём ему шанс
                return backend

            connection_penalty = 1 + (backend.current_connections / 10)
            score = backend.avg_latency_ms * connection_penalty / backend.weight

            if score < best_score:
                best_score = score
                best_backend = backend

        return best_backend

    async def request(self, path: str) -> dict:
        backend = self.select_backend()
        backend.current_connections += 1

        try:
            start = time.time()
            result = await self._do_request(backend, path)
            latency = (time.time() - start) * 1000
            backend.record_latency(latency)
            return result
        finally:
            backend.current_connections -= 1

# Использование
lb = LatencyAwareLoadBalancer([
    Backend("server1.local", 8080, weight=5),
    Backend("server2.local", 8080, weight=3),
    Backend("server3.local", 8080, weight=2),
])
```

### Пример 3: Dashboard мониторинга

```python
from prometheus_client import Histogram, Counter, Gauge
import time

# Метрики
REQUEST_LATENCY = Histogram(
    'http_request_duration_seconds',
    'HTTP request latency',
    ['method', 'endpoint', 'status'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

REQUEST_COUNT = Counter(
    'http_requests_total',
    'Total HTTP requests',
    ['method', 'endpoint', 'status']
)

CONCURRENT_REQUESTS = Gauge(
    'http_concurrent_requests',
    'Number of concurrent requests',
    ['endpoint']
)

# Middleware для сбора метрик
class MetricsMiddleware:
    def __init__(self, app):
        self.app = app

    async def __call__(self, scope, receive, send):
        if scope['type'] != 'http':
            return await self.app(scope, receive, send)

        endpoint = scope['path']
        method = scope['method']

        CONCURRENT_REQUESTS.labels(endpoint=endpoint).inc()
        start_time = time.time()

        # Перехватываем статус ответа
        status_code = 500

        async def send_wrapper(message):
            nonlocal status_code
            if message['type'] == 'http.response.start':
                status_code = message['status']
            await send(message)

        try:
            await self.app(scope, receive, send_wrapper)
        finally:
            duration = time.time() - start_time

            REQUEST_LATENCY.labels(
                method=method,
                endpoint=endpoint,
                status=status_code
            ).observe(duration)

            REQUEST_COUNT.labels(
                method=method,
                endpoint=endpoint,
                status=status_code
            ).inc()

            CONCURRENT_REQUESTS.labels(endpoint=endpoint).dec()

# Grafana Dashboard Query (PromQL)
"""
# P99 латентность по эндпоинтам
histogram_quantile(0.99,
    sum(rate(http_request_duration_seconds_bucket[5m])) by (le, endpoint)
)

# Throughput (RPS) по эндпоинтам
sum(rate(http_requests_total[1m])) by (endpoint)

# Apdex Score (Application Performance Index)
(
  sum(rate(http_request_duration_seconds_bucket{le="0.1"}[5m]))
  +
  sum(rate(http_request_duration_seconds_bucket{le="0.5"}[5m])) / 2
) / sum(rate(http_request_duration_seconds_count[5m]))
"""
```

---

## Резюме

### Ключевые выводы

1. **Латентность** — время одной операции (важно для UX)
2. **Пропускная способность** — количество операций в секунду (важно для масштабирования)
3. **Они связаны**, но это разные метрики — оптимизация одной не всегда улучшает другую
4. **Используйте перцентили** (P95, P99) для измерения латентности, не среднее
5. **Trade-offs неизбежны** — батчинг увеличивает throughput, но повышает латентность

### Чеклист оптимизации

```markdown
## Уменьшение латентности
- [ ] Кэширование (Redis, in-memory)
- [ ] Уменьшение round-trips (batching, JOIN)
- [ ] Географическая близость (CDN, edge)
- [ ] Асинхронная параллельная обработка
- [ ] Prefetching данных
- [ ] Индексы в БД

## Увеличение throughput
- [ ] Горизонтальное масштабирование
- [ ] Connection pooling
- [ ] Батчинг запросов
- [ ] Очереди для фоновых задач
- [ ] Компрессия данных
- [ ] Эффективные протоколы (HTTP/2, gRPC)
```

### Типичные SLO для веб-приложений

| Тип сервиса | P50 | P99 | Throughput |
|-------------|-----|-----|------------|
| API Gateway | < 50ms | < 200ms | 10,000+ RPS |
| CRUD API | < 100ms | < 500ms | 1,000-5,000 RPS |
| Search | < 200ms | < 1s | 100-1,000 RPS |
| Analytics | < 500ms | < 5s | 10-100 RPS |
| Batch Jobs | — | < 1 hour | N/A |
