# Кэширование (Caching)

[prev: 18-sharding-and-replication](./18-sharding-and-replication.md) | [next: 20-asynchronism](./20-asynchronism.md)

---

## Введение

**Кэширование** — это техника хранения копий данных в быстром хранилище (кэше) для ускорения последующего доступа к этим данным. Вместо того чтобы каждый раз выполнять дорогостоящую операцию (запрос к БД, вычисление, сетевой запрос), система возвращает заранее сохранённый результат.

### Зачем нужно кэширование?

1. **Снижение латентности** — данные из кэша возвращаются за микросекунды вместо миллисекунд
2. **Уменьшение нагрузки на backend** — меньше запросов к базе данных и API
3. **Экономия ресурсов** — меньше вычислений, меньше сетевого трафика
4. **Повышение отказоустойчивости** — кэш может отдавать данные, даже если основной источник недоступен
5. **Масштабируемость** — система обрабатывает больше запросов без увеличения нагрузки на БД

### Принцип работы

```
┌─────────┐    запрос     ┌─────────┐    cache miss    ┌──────────┐
│  Client │ ────────────► │  Cache  │ ────────────────► │ Database │
│         │ ◄──────────── │         │ ◄──────────────── │          │
└─────────┘    ответ      └─────────┘    данные        └──────────┘
                               │
                          cache hit
                               │
                               ▼
                         быстрый ответ
```

**Cache hit** — данные найдены в кэше (быстро)
**Cache miss** — данные не найдены, запрос идёт к источнику (медленно)

**Hit ratio** = cache hits / (cache hits + cache misses) — показатель эффективности кэша

---

## Уровни кэширования

Кэширование может применяться на разных уровнях системы:

```
┌─────────────────────────────────────────────────────────────┐
│                                                             │
│  ┌──────────┐   ┌─────┐   ┌─────────┐   ┌──────────────┐   │
│  │ Browser  │   │ CDN │   │ App     │   │ Database     │   │
│  │ Cache    │   │     │   │ Cache   │   │ Cache        │   │
│  └──────────┘   └─────┘   └─────────┘   └──────────────┘   │
│       ▲            ▲           ▲              ▲            │
│       │            │           │              │            │
│    Client        Edge       Server        Storage          │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### 1. Client-side Cache (Браузерный кэш)

Браузер кэширует статические ресурсы локально на устройстве пользователя.

**HTTP-заголовки для управления кэшем:**

```http
# Cache-Control — основной заголовок
Cache-Control: max-age=31536000, public, immutable

# Варианты:
Cache-Control: no-cache        # Проверять свежесть на сервере
Cache-Control: no-store        # Не кэшировать вообще
Cache-Control: private         # Только для конкретного пользователя
Cache-Control: public          # Можно кэшировать на CDN
Cache-Control: max-age=3600    # Хранить 1 час

# ETag — хеш содержимого для проверки изменений
ETag: "33a64df551425fcc55e4d42a148795d9f25f89d4"

# Last-Modified — время последнего изменения
Last-Modified: Wed, 21 Oct 2023 07:28:00 GMT

# Expires (устаревший, но поддерживается)
Expires: Thu, 01 Dec 2024 16:00:00 GMT
```

**Условные запросы:**

```http
# Клиент проверяет актуальность кэша
If-None-Match: "33a64df551425fcc55e4d42a148795d9f25f89d4"
If-Modified-Since: Wed, 21 Oct 2023 07:28:00 GMT

# Сервер отвечает:
# 304 Not Modified — используй кэш
# 200 OK + новые данные — обнови кэш
```

**Пример настройки в nginx:**

```nginx
location /static/ {
    # Агрессивное кэширование для файлов с версией в имени
    expires 1y;
    add_header Cache-Control "public, immutable";
}

location /api/ {
    # API не кэшируем
    add_header Cache-Control "no-store";
}

location / {
    # HTML — проверяем свежесть
    add_header Cache-Control "no-cache";
    etag on;
}
```

### 2. CDN Cache

CDN (Content Delivery Network) кэширует контент на edge-серверах по всему миру.

```
┌──────────┐                                    ┌──────────┐
│  User    │                                    │  Origin  │
│  Moscow  │◄───┐                          ┌───►│  Server  │
└──────────┘    │                          │    └──────────┘
                │    ┌──────────────────┐  │
┌──────────┐    │    │                  │  │
│  User    │◄───┼────┤   CDN Edge       │──┘
│  Berlin  │    │    │   (Frankfurt)    │
└──────────┘    │    │                  │
                │    └──────────────────┘
┌──────────┐    │
│  User    │◄───┘
│  Paris   │
└──────────┘
```

**Что кэширует CDN:**
- Статические файлы (JS, CSS, изображения)
- Видео и медиа
- API-ответы (с настройкой)
- HTML-страницы

**Пример конфигурации Cloudflare:**

```
# Page Rules
URL: example.com/api/*
Cache Level: Standard
Edge Cache TTL: 1 hour

URL: example.com/static/*
Cache Level: Cache Everything
Edge Cache TTL: 1 year
```

### 3. Application Cache

Кэширование на уровне приложения — Redis, Memcached, in-memory.

```python
import redis
import json
from functools import wraps

redis_client = redis.Redis(host='localhost', port=6379, db=0)

def cache(ttl=300):
    """Декоратор для кэширования результатов функции"""
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Формируем ключ из имени функции и аргументов
            cache_key = f"{func.__name__}:{args}:{kwargs}"

            # Пробуем получить из кэша
            cached = redis_client.get(cache_key)
            if cached:
                return json.loads(cached)

            # Вычисляем результат
            result = func(*args, **kwargs)

            # Сохраняем в кэш
            redis_client.setex(
                cache_key,
                ttl,
                json.dumps(result)
            )

            return result
        return wrapper
    return decorator

# Использование
@cache(ttl=600)
def get_user_profile(user_id: int) -> dict:
    """Тяжёлый запрос к БД"""
    return db.query(
        "SELECT * FROM users WHERE id = %s",
        [user_id]
    )

# Первый вызов — запрос к БД
profile = get_user_profile(123)  # ~50ms

# Второй вызов — из кэша
profile = get_user_profile(123)  # ~1ms
```

**In-memory кэш (для простых случаев):**

```python
from functools import lru_cache
from cachetools import TTLCache
import time

# Встроенный LRU кэш Python
@lru_cache(maxsize=1000)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n-1) + fibonacci(n-2)

# Кэш с TTL
cache = TTLCache(maxsize=1000, ttl=300)

def get_user(user_id):
    if user_id in cache:
        return cache[user_id]

    user = db.get_user(user_id)
    cache[user_id] = user
    return user
```

### 4. Database Cache

**Query Cache (MySQL):**

```sql
-- Включение query cache (устарело в MySQL 8.0+)
SET GLOBAL query_cache_type = ON;
SET GLOBAL query_cache_size = 268435456;  -- 256MB

-- Запрос будет кэшироваться
SELECT SQL_CACHE * FROM products WHERE category_id = 5;

-- Запрос не будет кэшироваться
SELECT SQL_NO_CACHE * FROM products WHERE category_id = 5;
```

**Buffer Pool (InnoDB):**

```sql
-- Настройка буферного пула
SET GLOBAL innodb_buffer_pool_size = 8589934592;  -- 8GB

-- Мониторинг эффективности
SHOW STATUS LIKE 'Innodb_buffer_pool%';
```

**PostgreSQL Shared Buffers:**

```sql
-- postgresql.conf
shared_buffers = 4GB
effective_cache_size = 12GB

-- Анализ использования кэша
SELECT
    sum(heap_blks_read) as heap_read,
    sum(heap_blks_hit) as heap_hit,
    sum(heap_blks_hit) / (sum(heap_blks_hit) + sum(heap_blks_read)) as ratio
FROM pg_statio_user_tables;
```

### 5. CPU Cache (L1, L2, L3)

Аппаратный кэш процессора — самый быстрый, но маленький.

```
┌─────────────────────────────────────────────────┐
│                    CPU                           │
│  ┌─────────────────────────────────────────┐    │
│  │  Core 0           Core 1                 │    │
│  │  ┌─────┐         ┌─────┐                │    │
│  │  │ L1d │ 32KB    │ L1d │  ← Данные      │    │
│  │  │ L1i │ 32KB    │ L1i │  ← Инструкции  │    │
│  │  └──┬──┘         └──┬──┘                │    │
│  │     │               │                    │    │
│  │  ┌──┴──┐         ┌──┴──┐                │    │
│  │  │ L2  │ 256KB   │ L2  │  ← Per-core    │    │
│  │  └──┬──┘         └──┬──┘                │    │
│  │     └───────┬───────┘                    │    │
│  │          ┌──┴──┐                         │    │
│  │          │ L3  │ 8-30MB  ← Shared        │    │
│  │          └─────┘                         │    │
│  └─────────────────────────────────────────┘    │
└─────────────────────────────────────────────────┘

Скорость доступа:
L1:  ~1 ns    (4 такта)
L2:  ~4 ns    (12 тактов)
L3:  ~12 ns   (36 тактов)
RAM: ~100 ns  (300+ тактов)
```

**Оптимизация для CPU кэша:**

```python
# Плохо — случайный доступ к памяти (cache misses)
matrix = [[0] * 1000 for _ in range(1000)]
for j in range(1000):
    for i in range(1000):
        matrix[i][j] = i + j  # Прыгаем по памяти

# Хорошо — последовательный доступ (cache-friendly)
for i in range(1000):
    for j in range(1000):
        matrix[i][j] = i + j  # Идём по строке
```

---

## Стратегии кэширования

### 1. Cache-Aside (Lazy Loading)

Приложение само управляет кэшем — проверяет, загружает, обновляет.

```
┌─────────┐  1. get   ┌─────────┐
│   App   │ ────────► │  Cache  │
│         │ ◄──────── │         │
│         │  2. miss  └─────────┘
│         │
│         │  3. query ┌─────────┐
│         │ ────────► │   DB    │
│         │ ◄──────── │         │
│         │  4. data  └─────────┘
│         │
│         │  5. set   ┌─────────┐
│         │ ────────► │  Cache  │
└─────────┘           └─────────┘
```

```python
class UserRepository:
    def __init__(self, db, cache):
        self.db = db
        self.cache = cache

    def get_user(self, user_id: int) -> dict:
        # 1. Проверяем кэш
        cache_key = f"user:{user_id}"
        cached = self.cache.get(cache_key)

        if cached:
            return cached  # Cache hit

        # 2. Cache miss — идём в БД
        user = self.db.query(
            "SELECT * FROM users WHERE id = %s",
            [user_id]
        )

        if user:
            # 3. Сохраняем в кэш
            self.cache.set(cache_key, user, ttl=3600)

        return user

    def update_user(self, user_id: int, data: dict):
        # Обновляем БД
        self.db.update("users", user_id, data)

        # Инвалидируем кэш
        self.cache.delete(f"user:{user_id}")
```

**Плюсы:**
- Простота реализации
- Кэшируются только нужные данные
- Отказоустойчивость — если кэш упал, система работает

**Минусы:**
- Cache miss = задержка
- Возможна рассинхронизация данных

### 2. Read-Through

Кэш сам загружает данные при miss. Приложение работает только с кэшем.

```
┌─────────┐  1. get   ┌─────────┐  2. load  ┌─────────┐
│   App   │ ────────► │  Cache  │ ────────► │   DB    │
│         │ ◄──────── │         │ ◄──────── │         │
└─────────┘  3. data  └─────────┘  data     └─────────┘
```

```python
class ReadThroughCache:
    def __init__(self, cache, loader):
        self.cache = cache
        self.loader = loader  # Функция загрузки из БД

    def get(self, key: str):
        value = self.cache.get(key)

        if value is None:
            # Кэш сам загружает данные
            value = self.loader(key)
            if value:
                self.cache.set(key, value)

        return value

# Использование
def load_user_from_db(key: str):
    user_id = key.split(":")[1]
    return db.query("SELECT * FROM users WHERE id = %s", [user_id])

cache = ReadThroughCache(redis_client, load_user_from_db)
user = cache.get("user:123")  # Автоматическая загрузка при miss
```

**Плюсы:**
- Приложение не знает о логике загрузки
- Упрощённый код

**Минусы:**
- Сложнее реализовать
- Первый запрос всё ещё медленный

### 3. Write-Through

При записи данные сохраняются синхронно и в кэш, и в БД.

```
┌─────────┐  1. write ┌─────────┐  2. write ┌─────────┐
│   App   │ ────────► │  Cache  │ ────────► │   DB    │
│         │ ◄──────── │         │ ◄──────── │         │
└─────────┘  4. ack   └─────────┘  3. ack   └─────────┘
```

```python
class WriteThroughCache:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db

    def write(self, key: str, value: dict):
        # Синхронная запись в оба хранилища
        self.db.save(key, value)
        self.cache.set(key, value)

    def get(self, key: str):
        # Данные всегда актуальны в кэше
        return self.cache.get(key)
```

**Плюсы:**
- Консистентность данных
- Кэш всегда актуален

**Минусы:**
- Высокая латентность записи
- Кэшируются даже неиспользуемые данные

### 4. Write-Behind (Write-Back)

Запись сначала идёт в кэш, затем асинхронно в БД.

```
┌─────────┐  1. write ┌─────────┐
│   App   │ ────────► │  Cache  │
│         │ ◄──────── │         │
└─────────┘  2. ack   │         │
                      │  async  │
                      │    ▼    │
                      │ ┌─────┐ │  3. batch write
                      │ │Queue│ │ ───────────────► DB
                      │ └─────┘ │
                      └─────────┘
```

```python
import asyncio
from collections import deque
import time

class WriteBehindCache:
    def __init__(self, cache, db, flush_interval=5, batch_size=100):
        self.cache = cache
        self.db = db
        self.write_queue = deque()
        self.flush_interval = flush_interval
        self.batch_size = batch_size

    async def write(self, key: str, value: dict):
        # Быстрая запись в кэш
        self.cache.set(key, value)

        # Добавляем в очередь на запись в БД
        self.write_queue.append((key, value, time.time()))

        # Если очередь большая — сбрасываем сразу
        if len(self.write_queue) >= self.batch_size:
            await self._flush()

    async def _flush(self):
        """Пакетная запись в БД"""
        if not self.write_queue:
            return

        batch = []
        while self.write_queue and len(batch) < self.batch_size:
            batch.append(self.write_queue.popleft())

        # Batch insert в БД
        await self.db.batch_upsert(batch)

    async def background_flush(self):
        """Фоновый процесс сброса"""
        while True:
            await asyncio.sleep(self.flush_interval)
            await self._flush()
```

**Плюсы:**
- Очень быстрая запись
- Batch-оптимизация для БД

**Минусы:**
- Риск потери данных при падении кэша
- Сложность реализации
- Eventual consistency

### 5. Refresh-Ahead

Упреждающее обновление кэша до истечения TTL.

```
TTL = 60 сек
Refresh threshold = 50 сек (83%)

0s ──────────────────────────── 50s ────────────── 60s
│                                │                  │
▼                                ▼                  ▼
[────── Данные актуальны ──────][─ Refresh zone ─][Expired]
                                 │
                            Фоновое обновление
```

```python
import asyncio
import time

class RefreshAheadCache:
    def __init__(self, cache, loader, ttl=60, refresh_threshold=0.8):
        self.cache = cache
        self.loader = loader
        self.ttl = ttl
        self.refresh_threshold = refresh_threshold  # 80% от TTL
        self.refreshing = set()  # Ключи в процессе обновления

    async def get(self, key: str):
        data, created_at = self.cache.get_with_metadata(key)

        if data is None:
            # Cache miss — синхронная загрузка
            return await self._load_and_cache(key)

        # Проверяем, нужно ли обновить заранее
        age = time.time() - created_at
        if age > self.ttl * self.refresh_threshold:
            if key not in self.refreshing:
                # Запускаем фоновое обновление
                asyncio.create_task(self._refresh(key))

        return data

    async def _refresh(self, key: str):
        """Фоновое обновление"""
        self.refreshing.add(key)
        try:
            await self._load_and_cache(key)
        finally:
            self.refreshing.discard(key)

    async def _load_and_cache(self, key: str):
        data = await self.loader(key)
        self.cache.set_with_metadata(key, data, time.time(), self.ttl)
        return data
```

**Плюсы:**
- Всегда свежие данные
- Нет задержки при обновлении

**Минусы:**
- Избыточные обновления
- Сложность реализации

---

## Политики вытеснения (Eviction Policies)

Когда кэш заполнен, нужно решить, какие данные удалить.

### 1. LRU (Least Recently Used)

Удаляется элемент, к которому дольше всего не обращались.

```
Кэш (max=3): [A, B, C]

get(A) → [B, C, A]    # A перемещается в конец
set(D) → [C, A, D]    # B удаляется (самый старый)
get(C) → [A, D, C]    # C перемещается в конец
```

```python
from collections import OrderedDict

class LRUCache:
    def __init__(self, capacity: int):
        self.cache = OrderedDict()
        self.capacity = capacity

    def get(self, key: str):
        if key not in self.cache:
            return None

        # Перемещаем в конец (самый свежий)
        self.cache.move_to_end(key)
        return self.cache[key]

    def set(self, key: str, value):
        if key in self.cache:
            self.cache.move_to_end(key)
        else:
            if len(self.cache) >= self.capacity:
                # Удаляем самый старый (первый)
                self.cache.popitem(last=False)

        self.cache[key] = value
```

**Redis LRU:**

```bash
# Конфигурация Redis
maxmemory 4gb
maxmemory-policy allkeys-lru

# Варианты политик:
# volatile-lru    — LRU только для ключей с TTL
# allkeys-lru     — LRU для всех ключей
# volatile-random — случайное удаление ключей с TTL
# allkeys-random  — случайное удаление всех ключей
# volatile-ttl    — удаление ключей с наименьшим TTL
# noeviction      — отказ в записи при переполнении
```

### 2. LFU (Least Frequently Used)

Удаляется элемент с наименьшим количеством обращений.

```python
from collections import defaultdict
import heapq

class LFUCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}           # key -> value
        self.freq = {}            # key -> frequency
        self.freq_to_keys = defaultdict(list)  # freq -> [keys]
        self.min_freq = 0
        self.time = 0             # Для разрешения ничьих

    def get(self, key: str):
        if key not in self.cache:
            return None

        self._update_freq(key)
        return self.cache[key]

    def set(self, key: str, value):
        if self.capacity == 0:
            return

        if key in self.cache:
            self.cache[key] = value
            self._update_freq(key)
            return

        if len(self.cache) >= self.capacity:
            self._evict()

        self.cache[key] = value
        self.freq[key] = 1
        self.freq_to_keys[1].append((self.time, key))
        self.min_freq = 1
        self.time += 1

    def _update_freq(self, key: str):
        f = self.freq[key]
        self.freq[key] = f + 1
        self.freq_to_keys[f + 1].append((self.time, key))
        self.time += 1

    def _evict(self):
        # Находим минимальную частоту
        while self.min_freq not in self.freq_to_keys or not self.freq_to_keys[self.min_freq]:
            self.min_freq += 1

        # Удаляем самый старый элемент с минимальной частотой
        while self.freq_to_keys[self.min_freq]:
            _, key = self.freq_to_keys[self.min_freq].pop(0)
            if key in self.cache and self.freq[key] == self.min_freq:
                del self.cache[key]
                del self.freq[key]
                return
```

**Redis LFU:**

```bash
maxmemory-policy allkeys-lfu

# LFU в Redis использует вероятностный счётчик
# lfu-log-factor — скорость затухания счётчика
# lfu-decay-time — время между декрементами
```

### 3. FIFO (First In, First Out)

Удаляется самый старый элемент (по времени добавления).

```python
from collections import deque

class FIFOCache:
    def __init__(self, capacity: int):
        self.capacity = capacity
        self.cache = {}
        self.queue = deque()

    def get(self, key: str):
        return self.cache.get(key)

    def set(self, key: str, value):
        if key in self.cache:
            self.cache[key] = value
            return

        if len(self.cache) >= self.capacity:
            oldest = self.queue.popleft()
            del self.cache[oldest]

        self.cache[key] = value
        self.queue.append(key)
```

### 4. TTL (Time To Live)

Данные автоматически удаляются после истечения времени жизни.

```python
import time
import threading

class TTLCache:
    def __init__(self):
        self.cache = {}  # key -> (value, expire_time)
        self.lock = threading.Lock()

        # Фоновая очистка
        self._start_cleanup()

    def set(self, key: str, value, ttl: int):
        expire_at = time.time() + ttl
        with self.lock:
            self.cache[key] = (value, expire_at)

    def get(self, key: str):
        with self.lock:
            if key not in self.cache:
                return None

            value, expire_at = self.cache[key]

            if time.time() > expire_at:
                del self.cache[key]
                return None

            return value

    def _cleanup(self):
        """Периодическая очистка истёкших ключей"""
        while True:
            time.sleep(60)  # Каждую минуту
            now = time.time()
            with self.lock:
                expired = [k for k, (_, exp) in self.cache.items() if now > exp]
                for key in expired:
                    del self.cache[key]

    def _start_cleanup(self):
        thread = threading.Thread(target=self._cleanup, daemon=True)
        thread.start()
```

**Redis TTL:**

```bash
# Установка TTL
SET user:123 "data" EX 3600       # 1 час
SETEX user:123 3600 "data"        # То же самое
PSETEX user:123 3600000 "data"    # В миллисекундах

# Проверка оставшегося времени
TTL user:123      # В секундах
PTTL user:123     # В миллисекундах

# Удаление TTL
PERSIST user:123

# Обновление TTL
EXPIRE user:123 7200    # Новый TTL — 2 часа
```

### Сравнение политик

| Политика | Сложность | Когда использовать |
|----------|-----------|-------------------|
| LRU | O(1) с OrderedDict | Общий случай, хорошо работает для большинства нагрузок |
| LFU | O(log n) | Когда важна частота, а не время последнего доступа |
| FIFO | O(1) | Простые случаи, когда порядок доступа не важен |
| TTL | O(1) | Данные с естественным временем жизни (сессии, токены) |

---

## Инструменты кэширования

### 1. Redis

Высокопроизводительное in-memory хранилище с поддержкой различных структур данных.

**Основные возможности:**

```bash
# Строки
SET user:123 '{"name":"John"}'
GET user:123

# Hash — для объектов
HSET user:123 name "John" age 30 email "john@example.com"
HGET user:123 name
HGETALL user:123

# Списки — для очередей
LPUSH queue:tasks "task1"
RPOP queue:tasks

# Sets — для уникальных значений
SADD online:users "user1" "user2"
SMEMBERS online:users

# Sorted Sets — для ранжирования
ZADD leaderboard 100 "player1" 200 "player2"
ZRANGE leaderboard 0 -1 WITHSCORES

# Pub/Sub — для событий
SUBSCRIBE channel
PUBLISH channel "message"
```

**Пример с Python:**

```python
import redis
from redis import Redis
from typing import Optional
import json

class RedisCache:
    def __init__(self, host='localhost', port=6379, db=0):
        self.client = Redis(
            host=host,
            port=port,
            db=db,
            decode_responses=True,
            socket_timeout=5,
            socket_connect_timeout=5
        )
        self.default_ttl = 3600

    def get(self, key: str) -> Optional[dict]:
        data = self.client.get(key)
        return json.loads(data) if data else None

    def set(self, key: str, value: dict, ttl: int = None):
        self.client.setex(
            key,
            ttl or self.default_ttl,
            json.dumps(value)
        )

    def delete(self, key: str):
        self.client.delete(key)

    def exists(self, key: str) -> bool:
        return self.client.exists(key) > 0

    def increment(self, key: str, amount: int = 1) -> int:
        return self.client.incrby(key, amount)

    def get_or_set(self, key: str, loader, ttl: int = None):
        """Атомарное получение или загрузка"""
        data = self.get(key)
        if data is None:
            data = loader()
            self.set(key, data, ttl)
        return data

    def mget(self, keys: list) -> dict:
        """Batch получение"""
        values = self.client.mget(keys)
        return {
            k: json.loads(v) if v else None
            for k, v in zip(keys, values)
        }

    def mset(self, data: dict, ttl: int = None):
        """Batch установка с TTL через pipeline"""
        pipe = self.client.pipeline()
        for key, value in data.items():
            pipe.setex(key, ttl or self.default_ttl, json.dumps(value))
        pipe.execute()
```

**Redis Cluster:**

```python
from redis.cluster import RedisCluster

# Подключение к кластеру
cluster = RedisCluster(
    startup_nodes=[
        {"host": "node1", "port": 7000},
        {"host": "node2", "port": 7001},
        {"host": "node3", "port": 7002},
    ]
)

# Использование как обычного Redis
cluster.set("key", "value")
cluster.get("key")
```

### 2. Memcached

Простой, быстрый кэш для строк и объектов.

```python
import pylibmc
import json

class MemcachedCache:
    def __init__(self, servers=['localhost:11211']):
        self.client = pylibmc.Client(
            servers,
            behaviors={
                "tcp_nodelay": True,
                "ketama": True,  # Consistent hashing
                "connect_timeout": 1000,
                "send_timeout": 500000,
                "receive_timeout": 500000,
            }
        )

    def get(self, key: str):
        data = self.client.get(key)
        return json.loads(data) if data else None

    def set(self, key: str, value, ttl: int = 3600):
        self.client.set(key, json.dumps(value), time=ttl)

    def delete(self, key: str):
        self.client.delete(key)

    def get_multi(self, keys: list) -> dict:
        """Batch получение"""
        data = self.client.get_multi(keys)
        return {k: json.loads(v) for k, v in data.items()}

    def incr(self, key: str, delta: int = 1):
        return self.client.incr(key, delta)
```

### Redis vs Memcached

| Характеристика | Redis | Memcached |
|---------------|-------|-----------|
| Структуры данных | Много (strings, lists, sets, hashes, sorted sets) | Только strings |
| Персистентность | Да (RDB, AOF) | Нет |
| Репликация | Да | Нет (только через прокси) |
| Кластеризация | Встроенная | Через прокси |
| Pub/Sub | Да | Нет |
| Lua-скрипты | Да | Нет |
| Потребление памяти | Выше | Ниже |
| Multi-threading | Нет (но I/O threads в 6.0+) | Да |

### 3. Varnish

HTTP-кэш для веб-приложений (reverse proxy cache).

```vcl
# /etc/varnish/default.vcl

vcl 4.1;

backend default {
    .host = "127.0.0.1";
    .port = "8080";
}

sub vcl_recv {
    # Не кэшируем запросы с куками
    if (req.http.Cookie) {
        return (pass);
    }

    # Не кэшируем POST
    if (req.method != "GET" && req.method != "HEAD") {
        return (pass);
    }

    # Кэшируем API на 5 минут
    if (req.url ~ "^/api/") {
        return (hash);
    }

    return (hash);
}

sub vcl_backend_response {
    # Кэшируем статику на 1 день
    if (bereq.url ~ "\.(png|gif|jpg|js|css)$") {
        set beresp.ttl = 1d;
        set beresp.http.Cache-Control = "public, max-age=86400";
    }

    # API кэшируем на 5 минут
    if (bereq.url ~ "^/api/") {
        set beresp.ttl = 5m;
    }

    return (deliver);
}

sub vcl_deliver {
    # Добавляем debug-заголовки
    if (obj.hits > 0) {
        set resp.http.X-Cache = "HIT";
        set resp.http.X-Cache-Hits = obj.hits;
    } else {
        set resp.http.X-Cache = "MISS";
    }
}
```

### 4. CDN (Cloudflare, Akamai, Fastly)

**Cloudflare Cache Rules (пример):**

```yaml
# Правила кэширования
rules:
  - name: "Cache API responses"
    expression: "(http.request.uri.path matches \"^/api/v1/products\")"
    action:
      cache:
        eligible: true
        edge_ttl: 3600
        browser_ttl: 300
        cache_key:
          query_string:
            include: ["category", "page"]

  - name: "Cache static assets"
    expression: "(http.request.uri.path matches \"\\.(js|css|png|jpg|woff2)$\")"
    action:
      cache:
        eligible: true
        edge_ttl: 31536000
        browser_ttl: 31536000
```

**Заголовки для управления CDN:**

```nginx
# nginx.conf

location /api/ {
    # Cloudflare-specific
    add_header CDN-Cache-Control "max-age=3600";

    # Стандартный Cache-Control
    add_header Cache-Control "public, max-age=300, s-maxage=3600";

    # Vary — кэш зависит от этих заголовков
    add_header Vary "Accept, Accept-Encoding, Authorization";

    # Surrogate-Control для Varnish/Fastly
    add_header Surrogate-Control "max-age=3600";
}
```

---

## Проблемы кэширования

### 1. Cache Invalidation

> "There are only two hard things in Computer Science: cache invalidation and naming things."
> — Phil Karlton

**Стратегии инвалидации:**

```python
class CacheInvalidation:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db

    # 1. TTL-based — самый простой
    def set_with_ttl(self, key: str, value, ttl: int = 300):
        self.cache.set(key, value, ttl=ttl)

    # 2. Event-based — при изменении данных
    def update_user(self, user_id: int, data: dict):
        self.db.update_user(user_id, data)

        # Удаляем связанные ключи
        self.cache.delete(f"user:{user_id}")
        self.cache.delete(f"user_profile:{user_id}")
        self.cache.delete(f"user_permissions:{user_id}")

    # 3. Versioned keys — версионирование
    def get_user_v(self, user_id: int):
        version = self.cache.get(f"user_version:{user_id}") or "1"
        return self.cache.get(f"user:{user_id}:v{version}")

    def invalidate_user(self, user_id: int):
        # Инкрементируем версию — старый кэш просто не найдётся
        self.cache.incr(f"user_version:{user_id}")

    # 4. Tag-based — групповая инвалидация
    def set_with_tags(self, key: str, value, tags: list):
        self.cache.set(key, value)
        for tag in tags:
            self.cache.sadd(f"tag:{tag}", key)

    def invalidate_by_tag(self, tag: str):
        keys = self.cache.smembers(f"tag:{tag}")
        if keys:
            self.cache.delete(*keys)
            self.cache.delete(f"tag:{tag}")

# Пример tag-based инвалидации
cache = CacheInvalidation(redis_client, db)

# Кэшируем с тегами
cache.set_with_tags("product:123", product_data, ["products", "category:electronics"])
cache.set_with_tags("product:456", product_data, ["products", "category:electronics"])

# Инвалидируем все продукты в категории
cache.invalidate_by_tag("category:electronics")
```

### 2. Cache Stampede (Thundering Herd)

Множество запросов одновременно идут в БД при истечении кэша.

```
TTL истёк
     │
     ▼
[Request 1] ───┐
[Request 2] ───┤
[Request 3] ───┼──► [Database] 💥 Перегрузка!
[Request 4] ───┤
[Request 5] ───┘
```

**Решения:**

```python
import threading
import time
import hashlib

class AntiStampede:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db
        self.locks = {}
        self._lock = threading.Lock()

    # 1. Locking — только один запрос идёт в БД
    def get_with_lock(self, key: str, loader, ttl: int = 300):
        value = self.cache.get(key)
        if value:
            return value

        lock_key = f"lock:{key}"

        # Пытаемся захватить лок
        if self.cache.set(lock_key, "1", nx=True, ex=10):
            try:
                # Мы получили лок — загружаем данные
                value = loader()
                self.cache.set(key, value, ttl=ttl)
                return value
            finally:
                self.cache.delete(lock_key)
        else:
            # Лок занят — ждём и повторяем
            time.sleep(0.1)
            return self.get_with_lock(key, loader, ttl)

    # 2. Probabilistic early expiration
    def get_with_early_recompute(self, key: str, loader, ttl: int = 300, beta: float = 1.0):
        """
        XFetch algorithm: с некоторой вероятностью
        обновляем кэш до истечения TTL
        """
        data = self.cache.get_with_metadata(key)

        if data is None:
            value = loader()
            self.cache.set(key, value, ttl=ttl)
            return value

        value, expiry, created = data
        now = time.time()
        remaining = expiry - now
        age = now - created

        # Вероятность раннего обновления растёт с возрастом
        # gap = ttl * beta * log(random())
        import random
        import math

        delta = ttl - remaining
        should_recompute = delta * beta * math.log(random.random()) >= remaining

        if should_recompute:
            value = loader()
            self.cache.set(key, value, ttl=ttl)

        return value

    # 3. Background refresh
    def get_with_background_refresh(self, key: str, loader, ttl: int = 300):
        value = self.cache.get(key)
        remaining_ttl = self.cache.ttl(key)

        if value:
            # Если осталось мало времени — обновляем в фоне
            if remaining_ttl < ttl * 0.2:  # Менее 20% TTL
                self._async_refresh(key, loader, ttl)
            return value

        # Cache miss
        value = loader()
        self.cache.set(key, value, ttl=ttl)
        return value

    def _async_refresh(self, key: str, loader, ttl: int):
        def refresh():
            value = loader()
            self.cache.set(key, value, ttl=ttl)

        thread = threading.Thread(target=refresh)
        thread.start()
```

### 3. Cache Penetration

Запросы несуществующих данных всегда идут в БД.

```
Атакующий запрашивает user_id=-1, -2, -3...
Таких пользователей нет → всегда cache miss → БД перегружена
```

**Решения:**

```python
class CachePenetrationProtection:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db

    # 1. Кэшируем null-результаты
    def get_user(self, user_id: int):
        cache_key = f"user:{user_id}"

        # Проверяем, есть ли в кэше (включая null)
        cached = self.cache.get(cache_key)
        if cached == "NULL":
            return None
        if cached:
            return cached

        # Запрос к БД
        user = self.db.get_user(user_id)

        if user:
            self.cache.set(cache_key, user, ttl=3600)
        else:
            # Кэшируем отсутствие данных с коротким TTL
            self.cache.set(cache_key, "NULL", ttl=60)

        return user

    # 2. Bloom Filter — быстрая проверка существования
    def get_user_with_bloom(self, user_id: int):
        # Bloom filter говорит:
        # "точно нет" или "возможно есть"
        if not self.bloom_filter.might_contain(f"user:{user_id}"):
            return None  # Точно нет в БД

        # Возможно есть — проверяем кэш и БД
        return self.get_user(user_id)

    # 3. Rate limiting для подозрительных запросов
    def get_user_protected(self, user_id: int, client_ip: str):
        # Лимит на miss-ы с одного IP
        miss_key = f"misses:{client_ip}"
        misses = self.cache.incr(miss_key)

        if misses == 1:
            self.cache.expire(miss_key, 60)

        if misses > 100:  # Больше 100 miss-ов в минуту
            raise RateLimitError("Too many cache misses")

        return self.get_user(user_id)
```

**Bloom Filter пример:**

```python
import mmh3
from bitarray import bitarray

class BloomFilter:
    def __init__(self, size: int = 1000000, hash_count: int = 7):
        self.size = size
        self.hash_count = hash_count
        self.bit_array = bitarray(size)
        self.bit_array.setall(0)

    def add(self, item: str):
        for seed in range(self.hash_count):
            index = mmh3.hash(item, seed) % self.size
            self.bit_array[index] = 1

    def might_contain(self, item: str) -> bool:
        for seed in range(self.hash_count):
            index = mmh3.hash(item, seed) % self.size
            if not self.bit_array[index]:
                return False  # Точно нет
        return True  # Возможно есть (может быть false positive)

# Заполняем при старте приложения
bloom = BloomFilter(size=10_000_000, hash_count=7)
for user_id in db.get_all_user_ids():
    bloom.add(f"user:{user_id}")

# Использование
def get_user(user_id: int):
    if not bloom.might_contain(f"user:{user_id}"):
        return None  # Быстрый ответ без похода в БД/кэш

    # Обычная логика с кэшем
    ...
```

### 4. Cache Avalanche

Массовое истечение TTL приводит к лавине запросов в БД.

```
12:00:00 — все ключи закэшированы с TTL=1h
13:00:00 — все ключи истекают одновременно
         → 10000 запросов в БД → 💥
```

**Решения:**

```python
import random

class CacheAvalancheProtection:
    def __init__(self, cache):
        self.cache = cache

    # 1. Случайный jitter в TTL
    def set_with_jitter(self, key: str, value, base_ttl: int = 3600):
        # TTL = base_ttl ± 10%
        jitter = int(base_ttl * 0.1)
        actual_ttl = base_ttl + random.randint(-jitter, jitter)
        self.cache.set(key, value, ttl=actual_ttl)

    # 2. Разные TTL для разных типов данных
    def set_user(self, user_id: int, data: dict):
        self.set_with_jitter(f"user:{user_id}", data, base_ttl=3600)

    def set_product(self, product_id: int, data: dict):
        self.set_with_jitter(f"product:{product_id}", data, base_ttl=7200)

    # 3. Прогрев кэша при старте (с распределением)
    async def warm_cache(self, items: list, base_ttl: int = 3600):
        for i, item in enumerate(items):
            # Распределяем TTL, чтобы не истекали одновременно
            offset = (i / len(items)) * base_ttl * 0.5  # 0-50% от TTL
            ttl = int(base_ttl + offset)
            await self.cache.set(item['key'], item['value'], ttl=ttl)
```

### 5. Consistency (Согласованность)

Данные в кэше могут отличаться от данных в БД.

```python
class ConsistencyStrategies:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db

    # 1. Strong consistency — синхронное обновление
    def update_user_strong(self, user_id: int, data: dict):
        # Транзакция
        try:
            self.db.begin()
            self.db.update_user(user_id, data)
            self.cache.delete(f"user:{user_id}")
            self.db.commit()
        except Exception:
            self.db.rollback()
            raise

    # 2. Eventual consistency — асинхронное обновление через события
    async def update_user_eventual(self, user_id: int, data: dict):
        self.db.update_user(user_id, data)

        # Публикуем событие
        await self.event_bus.publish("user.updated", {
            "user_id": user_id,
            "timestamp": time.time()
        })

    async def handle_user_updated(self, event: dict):
        """Обработчик события — инвалидирует кэш"""
        user_id = event["user_id"]
        self.cache.delete(f"user:{user_id}")

    # 3. Read-your-writes — пользователь видит свои изменения
    def get_user_with_ryw(self, user_id: int, session_writes: dict):
        # Если пользователь только что обновил данные
        if f"user:{user_id}" in session_writes:
            # Читаем из БД, игнорируя кэш
            return self.db.get_user(user_id)

        # Обычная логика с кэшем
        return self.get_user(user_id)

    def update_user_with_ryw(self, user_id: int, data: dict, session_writes: dict):
        self.db.update_user(user_id, data)
        self.cache.delete(f"user:{user_id}")

        # Запоминаем в сессии
        session_writes[f"user:{user_id}"] = time.time()
```

---

## Распределённое кэширование (Distributed Caching)

### Консистентное хеширование

```
                    Hash Ring
                  ┌───────────┐
                 ╱             ╲
               ╱                 ╲
              │    ┌───┐         │
              │    │ N1│         │
             ╱     └───┘          ╲
            │                      │
        ┌───┤                      ├───┐
        │ N4│                      │ N2│
        └───┤                      ├───┘
            │                      │
             ╲     ┌───┐          ╱
              │    │ N3│         │
              │    └───┘         │
               ╲                 ╱
                 ╲             ╱
                  └───────────┘

Ключ "user:123" → hash → попадает на N2
При добавлении N5 перераспределяется минимум ключей
```

```python
import hashlib
from bisect import bisect_left

class ConsistentHash:
    def __init__(self, nodes: list = None, virtual_nodes: int = 100):
        self.virtual_nodes = virtual_nodes
        self.ring = {}
        self.sorted_keys = []

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """Добавляем виртуальные узлы для равномерного распределения"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            self.ring[hash_value] = node
            self.sorted_keys.append(hash_value)

        self.sorted_keys.sort()

    def remove_node(self, node: str):
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)
            del self.ring[hash_value]
            self.sorted_keys.remove(hash_value)

    def get_node(self, key: str) -> str:
        if not self.ring:
            return None

        hash_value = self._hash(key)

        # Ищем первый узел по часовой стрелке
        idx = bisect_left(self.sorted_keys, hash_value)
        if idx == len(self.sorted_keys):
            idx = 0

        return self.ring[self.sorted_keys[idx]]

# Использование
ring = ConsistentHash(["redis-1", "redis-2", "redis-3"])

# Определяем, на какой узел идёт запрос
node = ring.get_node("user:123")  # → "redis-2"

# Добавление нового узла перераспределяет минимум ключей
ring.add_node("redis-4")
```

### Репликация и партиционирование

```python
class DistributedCache:
    def __init__(self, nodes: list, replicas: int = 2):
        self.consistent_hash = ConsistentHash(nodes)
        self.replicas = replicas
        self.connections = {node: Redis(node) for node in nodes}

    def _get_nodes(self, key: str) -> list:
        """Получаем primary и replica узлы"""
        primary = self.consistent_hash.get_node(key)
        nodes = [primary]

        # Добавляем реплики
        idx = self.consistent_hash.sorted_keys.index(
            self.consistent_hash._hash(f"{primary}:0")
        )

        while len(nodes) < self.replicas + 1:
            idx = (idx + 1) % len(self.consistent_hash.sorted_keys)
            node = self.consistent_hash.ring[self.consistent_hash.sorted_keys[idx]]
            if node not in nodes:
                nodes.append(node)

        return nodes

    def get(self, key: str):
        """Читаем с любой реплики"""
        nodes = self._get_nodes(key)

        for node in nodes:
            try:
                value = self.connections[node].get(key)
                if value:
                    return value
            except ConnectionError:
                continue

        return None

    def set(self, key: str, value, ttl: int = 3600):
        """Пишем на все реплики"""
        nodes = self._get_nodes(key)

        for node in nodes:
            try:
                self.connections[node].setex(key, ttl, value)
            except ConnectionError:
                # Логируем ошибку, продолжаем
                pass
```

### Redis Cluster

```python
from redis.cluster import RedisCluster

# Автоматическое шардирование и репликация
cluster = RedisCluster(
    startup_nodes=[
        {"host": "redis-1", "port": 7000},
        {"host": "redis-2", "port": 7001},
        {"host": "redis-3", "port": 7002},
    ],
    decode_responses=True,
    skip_full_coverage_check=True
)

# Использование как обычного Redis
cluster.set("user:123", "data")
cluster.get("user:123")

# Для атомарных операций с несколькими ключами
# используем hash tags — ключи на одном слоте
cluster.set("{user:123}:profile", "...")
cluster.set("{user:123}:settings", "...")

# Pipeline для batch-операций
pipe = cluster.pipeline()
pipe.get("key1")
pipe.get("key2")
results = pipe.execute()
```

---

## Best Practices

### 1. Именование ключей

```python
# Хорошо — иерархическая структура
"user:123:profile"
"user:123:orders"
"product:456:details"
"session:abc123"

# Плохо — неструктурированные ключи
"u123p"
"mydata"
"temp"
```

### 2. Сериализация

```python
import json
import pickle
import msgpack

# JSON — читаемый, совместимый
data = json.dumps({"name": "John", "age": 30})

# MessagePack — компактный, быстрый
data = msgpack.packb({"name": "John", "age": 30})

# Pickle — только для Python, поддерживает сложные объекты
data = pickle.dumps(complex_object)

# Сравнение размеров
obj = {"users": [{"id": i, "name": f"User {i}"} for i in range(100)]}
len(json.dumps(obj))      # ~3800 bytes
len(msgpack.packb(obj))   # ~2400 bytes
```

### 3. Мониторинг

```python
import time
from datadog import statsd

class MonitoredCache:
    def __init__(self, cache):
        self.cache = cache

    def get(self, key: str):
        start = time.time()
        value = self.cache.get(key)
        duration = time.time() - start

        # Метрики
        statsd.timing("cache.get.latency", duration * 1000)
        statsd.increment("cache.get.total")

        if value:
            statsd.increment("cache.hit")
        else:
            statsd.increment("cache.miss")

        return value

    def set(self, key: str, value, ttl: int = 3600):
        start = time.time()
        self.cache.set(key, value, ttl=ttl)
        duration = time.time() - start

        statsd.timing("cache.set.latency", duration * 1000)
        statsd.increment("cache.set.total")
        statsd.gauge("cache.key.ttl", ttl, tags=[f"key:{key}"])
```

**Redis мониторинг:**

```bash
# Статистика
redis-cli INFO stats
redis-cli INFO memory

# Ключевые метрики:
# - keyspace_hits / keyspace_misses → hit ratio
# - used_memory / maxmemory → использование памяти
# - evicted_keys → количество вытесненных ключей
# - connected_clients → активные подключения

# Slowlog
redis-cli SLOWLOG GET 10

# Мониторинг в реальном времени
redis-cli MONITOR
```

### 4. Graceful Degradation

```python
class ResilientCache:
    def __init__(self, cache, db):
        self.cache = cache
        self.db = db
        self.cache_available = True

    def get(self, key: str, loader):
        if not self.cache_available:
            return loader()

        try:
            value = self.cache.get(key)
            if value:
                return value

            data = loader()
            self.cache.set(key, data)
            return data

        except ConnectionError:
            self.cache_available = False
            self._schedule_health_check()
            return loader()

    def _schedule_health_check(self):
        """Периодически проверяем доступность кэша"""
        async def check():
            while not self.cache_available:
                await asyncio.sleep(5)
                try:
                    self.cache.ping()
                    self.cache_available = True
                except ConnectionError:
                    pass

        asyncio.create_task(check())
```

### 5. Размер значений

```python
# Ограничиваем размер кэшируемых данных
MAX_CACHE_VALUE_SIZE = 1024 * 1024  # 1MB

def set_safe(cache, key: str, value, ttl: int = 3600):
    serialized = json.dumps(value)

    if len(serialized) > MAX_CACHE_VALUE_SIZE:
        # Слишком большой объект — не кэшируем
        logger.warning(f"Value too large to cache: {key}, size: {len(serialized)}")
        return False

    cache.set(key, serialized, ttl=ttl)
    return True

# Для больших объектов — компрессия
import gzip

def set_compressed(cache, key: str, value, ttl: int = 3600):
    serialized = json.dumps(value).encode()
    compressed = gzip.compress(serialized)

    cache.set(key, compressed, ttl=ttl)

def get_compressed(cache, key: str):
    compressed = cache.get(key)
    if compressed:
        decompressed = gzip.decompress(compressed)
        return json.loads(decompressed)
    return None
```

---

## Примеры использования

### 1. Кэширование API-ответов

```python
from fastapi import FastAPI, Request
from functools import wraps
import hashlib
import json

app = FastAPI()

def cache_response(ttl: int = 300):
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Формируем ключ из аргументов
            request: Request = kwargs.get('request') or args[0]
            cache_key = f"api:{request.url.path}:{request.query_params}"

            # Проверяем кэш
            cached = redis.get(cache_key)
            if cached:
                return json.loads(cached)

            # Выполняем запрос
            result = await func(*args, **kwargs)

            # Кэшируем результат
            redis.setex(cache_key, ttl, json.dumps(result))

            return result
        return wrapper
    return decorator

@app.get("/api/products")
@cache_response(ttl=300)
async def get_products(category: str = None):
    products = await db.get_products(category)
    return {"products": products}
```

### 2. Кэширование сессий

```python
from fastapi import FastAPI, Depends, HTTPException
from fastapi.security import HTTPBearer
import secrets

class SessionStore:
    def __init__(self, redis_client, ttl: int = 86400):
        self.redis = redis_client
        self.ttl = ttl

    def create_session(self, user_id: int, data: dict = None) -> str:
        session_id = secrets.token_urlsafe(32)
        session_data = {
            "user_id": user_id,
            "created_at": time.time(),
            **(data or {})
        }

        self.redis.setex(
            f"session:{session_id}",
            self.ttl,
            json.dumps(session_data)
        )

        return session_id

    def get_session(self, session_id: str) -> dict:
        data = self.redis.get(f"session:{session_id}")
        if not data:
            return None

        # Продлеваем TTL при обращении
        self.redis.expire(f"session:{session_id}", self.ttl)

        return json.loads(data)

    def destroy_session(self, session_id: str):
        self.redis.delete(f"session:{session_id}")

# Использование
session_store = SessionStore(redis)

@app.post("/login")
async def login(credentials: LoginRequest):
    user = await authenticate(credentials)
    session_id = session_store.create_session(user.id)
    return {"session_id": session_id}

async def get_current_user(token: str = Depends(HTTPBearer())):
    session = session_store.get_session(token.credentials)
    if not session:
        raise HTTPException(401, "Invalid session")
    return session
```

### 3. Rate Limiting

```python
class RateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    def is_allowed(self, key: str, limit: int, window: int) -> tuple[bool, int]:
        """
        Sliding window rate limiter

        Args:
            key: идентификатор (IP, user_id)
            limit: максимум запросов
            window: окно в секундах

        Returns:
            (allowed, remaining)
        """
        now = time.time()
        window_start = now - window

        pipe = self.redis.pipeline()

        # Удаляем старые записи
        pipe.zremrangebyscore(key, 0, window_start)

        # Считаем текущие запросы
        pipe.zcard(key)

        # Добавляем текущий запрос
        pipe.zadd(key, {str(now): now})

        # Устанавливаем TTL
        pipe.expire(key, window)

        results = pipe.execute()
        current_count = results[1]

        if current_count >= limit:
            return False, 0

        return True, limit - current_count - 1

# Использование
limiter = RateLimiter(redis)

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_ip = request.client.host

    allowed, remaining = limiter.is_allowed(
        f"rate:{client_ip}",
        limit=100,
        window=60
    )

    if not allowed:
        return JSONResponse(
            {"error": "Too many requests"},
            status_code=429,
            headers={"Retry-After": "60"}
        )

    response = await call_next(request)
    response.headers["X-RateLimit-Remaining"] = str(remaining)
    return response
```

### 4. Распределённая блокировка

```python
import uuid
import time

class DistributedLock:
    def __init__(self, redis_client, name: str, timeout: int = 10):
        self.redis = redis_client
        self.name = f"lock:{name}"
        self.timeout = timeout
        self.token = str(uuid.uuid4())

    def acquire(self) -> bool:
        """Попытка захватить блокировку"""
        return self.redis.set(
            self.name,
            self.token,
            nx=True,  # Только если не существует
            ex=self.timeout
        )

    def release(self) -> bool:
        """Освобождение блокировки (атомарно)"""
        script = """
        if redis.call("get", KEYS[1]) == ARGV[1] then
            return redis.call("del", KEYS[1])
        else
            return 0
        end
        """
        return self.redis.eval(script, 1, self.name, self.token)

    def __enter__(self):
        acquired = self.acquire()
        if not acquired:
            raise LockError(f"Could not acquire lock: {self.name}")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()

# Использование
def process_order(order_id: int):
    with DistributedLock(redis, f"order:{order_id}", timeout=30):
        # Критическая секция — только один процесс
        order = db.get_order(order_id)
        process(order)
        db.update_order(order_id, status="processed")
```

### 5. Leaderboard

```python
class Leaderboard:
    def __init__(self, redis_client, name: str):
        self.redis = redis_client
        self.key = f"leaderboard:{name}"

    def add_score(self, player_id: str, score: float):
        """Добавить/обновить счёт игрока"""
        self.redis.zadd(self.key, {player_id: score})

    def increment_score(self, player_id: str, delta: float):
        """Увеличить счёт"""
        return self.redis.zincrby(self.key, delta, player_id)

    def get_rank(self, player_id: str) -> int:
        """Получить место игрока (0 = первое место)"""
        rank = self.redis.zrevrank(self.key, player_id)
        return rank + 1 if rank is not None else None

    def get_top(self, n: int = 10) -> list:
        """Топ N игроков"""
        results = self.redis.zrevrange(
            self.key,
            0,
            n - 1,
            withscores=True
        )
        return [
            {"player_id": player, "score": score, "rank": i + 1}
            for i, (player, score) in enumerate(results)
        ]

    def get_around_player(self, player_id: str, n: int = 5) -> list:
        """Игроки вокруг указанного"""
        rank = self.redis.zrevrank(self.key, player_id)
        if rank is None:
            return []

        start = max(0, rank - n)
        end = rank + n

        results = self.redis.zrevrange(
            self.key,
            start,
            end,
            withscores=True
        )

        return [
            {"player_id": player, "score": score, "rank": start + i + 1}
            for i, (player, score) in enumerate(results)
        ]

# Использование
leaderboard = Leaderboard(redis, "weekly")

leaderboard.add_score("player1", 1000)
leaderboard.increment_score("player1", 50)

top10 = leaderboard.get_top(10)
my_rank = leaderboard.get_rank("player1")
nearby = leaderboard.get_around_player("player1", n=5)
```

---

## Чек-лист по кэшированию

### При проектировании

- [ ] Определить, какие данные кэшировать (read-heavy, expensive queries)
- [ ] Выбрать стратегию кэширования (cache-aside, read-through, etc.)
- [ ] Выбрать политику вытеснения (LRU, LFU, TTL)
- [ ] Определить TTL для разных типов данных
- [ ] Спланировать инвалидацию кэша
- [ ] Продумать graceful degradation

### При реализации

- [ ] Использовать консистентное именование ключей
- [ ] Добавить jitter к TTL для предотвращения avalanche
- [ ] Защититься от cache stampede (locking, early recompute)
- [ ] Защититься от cache penetration (null caching, bloom filter)
- [ ] Настроить мониторинг (hit ratio, latency, memory)
- [ ] Логировать cache miss для анализа

### При эксплуатации

- [ ] Мониторить hit ratio (должен быть >90%)
- [ ] Следить за использованием памяти
- [ ] Анализировать slowlog
- [ ] Настроить алерты на аномалии
- [ ] Периодически ревьюить стратегию кэширования

---

## Полезные ресурсы

1. [Redis Documentation](https://redis.io/documentation)
2. [Memcached Wiki](https://github.com/memcached/memcached/wiki)
3. [Varnish Cache](https://varnish-cache.org/docs/)
4. [Cloudflare Cache](https://developers.cloudflare.com/cache/)
5. [Caching Strategies and How to Choose the Right One](https://codeahoy.com/2017/08/11/caching-strategies-and-how-to-choose-the-right-one/)
6. [System Design Primer - Caching](https://github.com/donnemartin/system-design-primer#cache)
