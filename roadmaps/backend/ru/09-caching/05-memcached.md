# Memcached

[prev: 04-caching-strategies](./04-caching-strategies.md) | [next: 01-https](../10-web-security/01-https.md)

---

## Что такое Memcached?

**Memcached** — это высокопроизводительная распределённая система кэширования объектов в памяти. Изначально разработана для LiveJournal, сейчас используется Facebook, Twitter, YouTube и многими другими крупными проектами.

## Memcached vs Redis

```
┌─────────────────────┬─────────────────────┬─────────────────────┐
│     Характеристика  │     Memcached       │       Redis         │
├─────────────────────┼─────────────────────┼─────────────────────┤
│ Структуры данных    │ Только key-value    │ Strings, Hashes,    │
│                     │                     │ Lists, Sets, etc.   │
│                     │                     │                     │
│ Многопоточность     │ Да (multi-threaded) │ Нет (single-thread) │
│                     │                     │                     │
│ Персистентность     │ Нет                 │ RDB, AOF            │
│                     │                     │                     │
│ Репликация          │ Нет (сторонние)     │ Встроенная          │
│                     │                     │                     │
│ Pub/Sub             │ Нет                 │ Да                  │
│                     │                     │                     │
│ Lua скрипты         │ Нет                 │ Да                  │
│                     │                     │                     │
│ Макс. размер ключа  │ 250 байт            │ 512 MB              │
│                     │                     │                     │
│ Макс. размер значен.│ 1 MB (по умолчанию) │ 512 MB              │
│                     │                     │                     │
│ Потребление памяти  │ Меньше              │ Больше              │
│                     │                     │                     │
│ Сложность           │ Проще               │ Сложнее             │
└─────────────────────┴─────────────────────┴─────────────────────┘
```

### Когда выбирать Memcached?

- **Простое key-value кэширование** — не нужны сложные структуры данных
- **Многопоточная нагрузка** — лучше утилизирует многоядерные CPU
- **Кэширование сессий** — простой и быстрый
- **Горизонтальное масштабирование** — легко добавлять ноды
- **Минимальное потребление памяти** — важно при ограниченных ресурсах

## Установка и настройка

### Установка

```bash
# Ubuntu/Debian
sudo apt-get install memcached libmemcached-tools

# macOS
brew install memcached

# Docker
docker run -d --name memcached -p 11211:11211 memcached:latest

# С параметрами
docker run -d --name memcached \
    -p 11211:11211 \
    memcached:latest \
    memcached -m 512 -c 1024 -t 4
```

### Конфигурация

```bash
# /etc/memcached.conf

# Память (MB)
-m 512

# Максимальное количество соединений
-c 1024

# Количество потоков
-t 4

# Порт
-p 11211

# Слушать на всех интерфейсах (осторожно в продакшене!)
# -l 0.0.0.0

# Слушать только localhost
-l 127.0.0.1

# Максимальный размер объекта (байты)
-I 2m

# Запуск от пользователя
-u memcache

# Вербозный лог (для отладки)
# -vv
```

## Работа с Python

### Установка клиента

```bash
# pymemcache — рекомендуемый клиент
pip install pymemcache

# python-memcached — альтернатива
pip install python-memcached
```

### Базовые операции

```python
from pymemcache.client.base import Client
from pymemcache.client.hash import HashClient
import json

# Сериализатор для JSON
def json_serializer(key, value):
    if isinstance(value, str):
        return value.encode('utf-8'), 1
    return json.dumps(value).encode('utf-8'), 2

def json_deserializer(key, value, flags):
    if flags == 1:
        return value.decode('utf-8')
    if flags == 2:
        return json.loads(value.decode('utf-8'))
    return value


class MemcachedCache:
    def __init__(self, servers=None, default_ttl=3600):
        servers = servers or [('localhost', 11211)]
        self.default_ttl = default_ttl

        # Для одного сервера
        if len(servers) == 1:
            self.client = Client(
                servers[0],
                serializer=json_serializer,
                deserializer=json_deserializer,
                connect_timeout=5,
                timeout=3
            )
        else:
            # Для кластера
            self.client = HashClient(
                servers,
                serializer=json_serializer,
                deserializer=json_deserializer,
                use_pooling=True,
                max_pool_size=10
            )

    def get(self, key: str):
        """Получить значение"""
        return self.client.get(key)

    def set(self, key: str, value, ttl: int = None) -> bool:
        """Установить значение"""
        ttl = ttl or self.default_ttl
        return self.client.set(key, value, expire=ttl)

    def delete(self, key: str) -> bool:
        """Удалить ключ"""
        return self.client.delete(key)

    def get_multi(self, keys: list) -> dict:
        """Получить несколько ключей за один запрос"""
        return self.client.get_multi(keys)

    def set_multi(self, mapping: dict, ttl: int = None) -> list:
        """Установить несколько ключей"""
        ttl = ttl or self.default_ttl
        failed = self.client.set_multi(mapping, expire=ttl)
        return failed  # Возвращает список ключей, которые не удалось записать

    def add(self, key: str, value, ttl: int = None) -> bool:
        """Добавить только если ключ не существует"""
        ttl = ttl or self.default_ttl
        return self.client.add(key, value, expire=ttl)

    def replace(self, key: str, value, ttl: int = None) -> bool:
        """Заменить только если ключ существует"""
        ttl = ttl or self.default_ttl
        return self.client.replace(key, value, expire=ttl)

    def incr(self, key: str, delta: int = 1) -> int:
        """Инкремент числового значения"""
        return self.client.incr(key, delta)

    def decr(self, key: str, delta: int = 1) -> int:
        """Декремент числового значения"""
        return self.client.decr(key, delta)

    def touch(self, key: str, ttl: int) -> bool:
        """Обновить TTL без изменения значения"""
        return self.client.touch(key, ttl)

    def flush_all(self) -> bool:
        """Очистить весь кэш (осторожно!)"""
        return self.client.flush_all()


# Использование
cache = MemcachedCache()

# Базовые операции
cache.set('user:123', {'name': 'John', 'email': 'john@example.com'})
user = cache.get('user:123')

# Массовые операции
cache.set_multi({
    'user:1': {'name': 'Alice'},
    'user:2': {'name': 'Bob'},
    'user:3': {'name': 'Charlie'}
})

users = cache.get_multi(['user:1', 'user:2', 'user:3'])

# Счётчики
cache.set('page_views', 0)
cache.incr('page_views')
cache.incr('page_views', 10)
```

### CAS (Check-And-Set) операции

```python
from pymemcache.client.base import Client

def update_with_cas(cache, key, update_fn, max_retries=5):
    """
    Атомарное обновление с CAS (Compare-And-Swap).
    Предотвращает race conditions при конкурентных обновлениях.
    """
    for attempt in range(max_retries):
        # gets возвращает значение и CAS token
        result = cache.client.gets(key)
        if result is None:
            return False

        value, cas_token = result

        # Применяем функцию обновления
        new_value = update_fn(value)

        # cas записывает только если token совпадает
        success = cache.client.cas(key, new_value, cas_token, expire=3600)

        if success:
            return True

        # Если не удалось — значение изменилось, повторяем

    return False  # Не удалось за max_retries попыток


# Пример: атомарное добавление в список
def add_item_to_list(cache, key, item):
    def update_fn(current_list):
        if current_list is None:
            return [item]
        if item not in current_list:
            current_list.append(item)
        return current_list

    return update_with_cas(cache, key, update_fn)
```

## Паттерны использования

### Кэширование результатов функций

```python
from functools import wraps
import hashlib

def memcached_cache(key_prefix: str, ttl: int = 3600):
    """Декоратор для кэширования результатов функций"""

    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            # Генерируем ключ из аргументов
            cache_key = f"{key_prefix}:{generate_cache_key(args, kwargs)}"

            # Пробуем получить из кэша
            cached = cache.get(cache_key)
            if cached is not None:
                return cached

            # Вызываем функцию
            result = func(*args, **kwargs)

            # Кэшируем результат
            if result is not None:
                cache.set(cache_key, result, ttl=ttl)

            return result
        return wrapper
    return decorator

def generate_cache_key(args, kwargs) -> str:
    """Генерация уникального ключа из аргументов"""
    key_parts = [str(arg) for arg in args]
    key_parts.extend(f"{k}={v}" for k, v in sorted(kwargs.items()))
    key_string = ":".join(key_parts)
    return hashlib.md5(key_string.encode()).hexdigest()[:16]


# Использование
@memcached_cache('user', ttl=600)
def get_user_by_id(user_id: int) -> dict:
    return db.users.find_one({'id': user_id})

@memcached_cache('products_list', ttl=300)
def get_products(category: str, page: int = 1) -> list:
    return db.products.find({'category': category}).skip((page-1)*20).limit(20)
```

### Кэширование сессий

```python
from flask import Flask, session
from flask_session import Session

app = Flask(__name__)

# Конфигурация Memcached для сессий
app.config['SESSION_TYPE'] = 'memcached'
app.config['SESSION_PERMANENT'] = True
app.config['SESSION_USE_SIGNER'] = True
app.config['SESSION_KEY_PREFIX'] = 'session:'
app.config['SESSION_MEMCACHED'] = Client(('localhost', 11211))

Session(app)

@app.route('/login', methods=['POST'])
def login():
    user = authenticate(request.form['username'], request.form['password'])
    if user:
        session['user_id'] = user.id
        session['logged_in'] = True
        return redirect('/')
    return 'Invalid credentials', 401
```

### Distributed Lock

```python
import time
import uuid

class MemcachedLock:
    """
    Распределённая блокировка на Memcached.
    Использует add() для атомарного захвата.
    """

    def __init__(self, cache, key: str, ttl: int = 30):
        self.cache = cache
        self.key = f'lock:{key}'
        self.ttl = ttl
        self.token = str(uuid.uuid4())
        self._acquired = False

    def acquire(self, timeout: int = 10) -> bool:
        """Попытка захватить блокировку"""
        start = time.time()

        while time.time() - start < timeout:
            # add() успешен только если ключ не существует
            if self.cache.client.add(self.key, self.token, expire=self.ttl):
                self._acquired = True
                return True

            time.sleep(0.1)

        return False

    def release(self):
        """Освободить блокировку"""
        if not self._acquired:
            return

        # Проверяем, что это наш лок (по токену)
        current = self.cache.get(self.key)
        if current == self.token:
            self.cache.delete(self.key)
            self._acquired = False

    def __enter__(self):
        if not self.acquire():
            raise Exception(f"Could not acquire lock: {self.key}")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.release()


# Использование
def process_payment(order_id: int):
    lock = MemcachedLock(cache, f'order:{order_id}', ttl=60)

    with lock:
        # Критическая секция — только один процесс
        order = db.orders.find_one({'id': order_id})
        if order['status'] == 'pending':
            process_order(order)
            db.orders.update({'id': order_id}, {'$set': {'status': 'completed'}})
```

### Rate Limiting

```python
class RateLimiter:
    """Rate limiting с использованием Memcached"""

    def __init__(self, cache, requests_per_minute: int = 60):
        self.cache = cache
        self.limit = requests_per_minute
        self.window = 60  # секунд

    def is_allowed(self, identifier: str) -> tuple[bool, int]:
        """
        Проверить, разрешён ли запрос.
        Возвращает (allowed, remaining_requests)
        """
        key = f'ratelimit:{identifier}:{int(time.time()) // self.window}'

        # Пробуем инкрементировать
        try:
            count = self.cache.client.incr(key, 1)
        except Exception:
            # Ключ не существует — создаём
            self.cache.set(key, 1, ttl=self.window)
            count = 1

        remaining = max(0, self.limit - count)
        allowed = count <= self.limit

        return allowed, remaining

    def get_headers(self, identifier: str) -> dict:
        """Заголовки для ответа"""
        allowed, remaining = self.is_allowed(identifier)
        return {
            'X-RateLimit-Limit': str(self.limit),
            'X-RateLimit-Remaining': str(remaining),
            'X-RateLimit-Reset': str((int(time.time()) // self.window + 1) * self.window)
        }


# Middleware для Flask
@app.before_request
def rate_limit():
    limiter = RateLimiter(cache, requests_per_minute=100)
    client_ip = request.remote_addr

    allowed, remaining = limiter.is_allowed(client_ip)

    if not allowed:
        response = jsonify({'error': 'Rate limit exceeded'})
        response.status_code = 429
        response.headers.update(limiter.get_headers(client_ip))
        return response
```

## Кластеризация и масштабирование

### Consistent Hashing

```python
from pymemcache.client.hash import HashClient

# Кластер из нескольких серверов
servers = [
    ('memcached1.example.com', 11211),
    ('memcached2.example.com', 11211),
    ('memcached3.example.com', 11211),
]

cluster = HashClient(
    servers,
    use_pooling=True,
    max_pool_size=10,
    # Consistent hashing — минимальное перераспределение при добавлении/удалении нод
    hasher=None,  # Используется по умолчанию
    serializer=json_serializer,
    deserializer=json_deserializer,
    # Повторные попытки при ошибках
    retry_attempts=2,
    retry_timeout=1,
    # Игнорировать недоступные серверы
    ignore_exc=True,
    dead_timeout=30  # Время, после которого "мёртвый" сервер проверяется снова
)
```

### Работа с недоступными нодами

```python
class ResilientCache:
    """Кэш с обработкой отказов нод"""

    def __init__(self, servers):
        self.client = HashClient(
            servers,
            use_pooling=True,
            ignore_exc=True,  # Не падать при недоступности ноды
            dead_timeout=30
        )
        self.fallback_client = None

    def get(self, key: str, default=None):
        try:
            result = self.client.get(key)
            return result if result is not None else default
        except Exception as e:
            logger.warning(f"Memcached get error: {e}")
            return default

    def set(self, key: str, value, ttl: int = 3600) -> bool:
        try:
            return self.client.set(key, value, expire=ttl)
        except Exception as e:
            logger.warning(f"Memcached set error: {e}")
            return False

    def get_stats(self) -> dict:
        """Статистика по всем серверам"""
        stats = {}
        for server in self.client.clients.keys():
            try:
                server_stats = self.client.clients[server].stats()
                stats[str(server)] = server_stats
            except Exception:
                stats[str(server)] = {'status': 'unavailable'}
        return stats
```

## Мониторинг

### Статистика Memcached

```python
def get_memcached_stats(cache) -> dict:
    """Получить и проанализировать статистику"""
    stats = cache.client.stats()

    if not stats:
        return {}

    # Декодируем байтовые ключи
    stats = {k.decode() if isinstance(k, bytes) else k:
             v.decode() if isinstance(v, bytes) else v
             for k, v in stats.items()}

    # Вычисляем метрики
    get_hits = int(stats.get('get_hits', 0))
    get_misses = int(stats.get('get_misses', 0))
    total_gets = get_hits + get_misses

    hit_ratio = (get_hits / total_gets * 100) if total_gets > 0 else 0

    bytes_used = int(stats.get('bytes', 0))
    limit_maxbytes = int(stats.get('limit_maxbytes', 0))
    memory_usage = (bytes_used / limit_maxbytes * 100) if limit_maxbytes > 0 else 0

    evictions = int(stats.get('evictions', 0))

    return {
        'hit_ratio': f'{hit_ratio:.2f}%',
        'get_hits': get_hits,
        'get_misses': get_misses,
        'memory_usage': f'{memory_usage:.2f}%',
        'bytes_used': bytes_used,
        'limit_maxbytes': limit_maxbytes,
        'evictions': evictions,
        'curr_items': stats.get('curr_items'),
        'total_items': stats.get('total_items'),
        'curr_connections': stats.get('curr_connections'),
        'uptime': stats.get('uptime')
    }


# Использование
stats = get_memcached_stats(cache)
print(f"Hit ratio: {stats['hit_ratio']}")
print(f"Memory usage: {stats['memory_usage']}")
print(f"Evictions: {stats['evictions']}")
```

### Prometheus метрики

```python
from prometheus_client import Counter, Gauge, Histogram

# Метрики
cache_hits = Counter('memcached_hits_total', 'Cache hits')
cache_misses = Counter('memcached_misses_total', 'Cache misses')
cache_latency = Histogram('memcached_operation_seconds', 'Operation latency',
                          ['operation'])
cache_memory = Gauge('memcached_memory_bytes', 'Memory usage')
cache_items = Gauge('memcached_items_current', 'Current items count')

class MonitoredCache:
    def __init__(self, cache):
        self.cache = cache

    def get(self, key: str):
        with cache_latency.labels('get').time():
            result = self.cache.get(key)

        if result is not None:
            cache_hits.inc()
        else:
            cache_misses.inc()

        return result

    def set(self, key: str, value, ttl: int = 3600):
        with cache_latency.labels('set').time():
            return self.cache.set(key, value, ttl)

    def update_stats(self):
        """Обновить Prometheus метрики из статистики Memcached"""
        stats = get_memcached_stats(self.cache)
        cache_memory.set(stats['bytes_used'])
        cache_items.set(int(stats['curr_items']))
```

## Лучшие практики

### 1. Правильные ключи

```python
# Хорошо: структурированные, понятные ключи
'user:123:profile'
'session:abc123'
'product:456:v2'  # Версионирование для инвалидации

# Плохо
'u123'
'12345'
'some_random_key'

# Ограничение: ключ не более 250 байт
# Решение для длинных ключей:
def safe_key(key: str) -> str:
    if len(key) > 200:
        return hashlib.md5(key.encode()).hexdigest()
    return key
```

### 2. Обработка больших значений

```python
import zlib

def compress_if_large(value: bytes, threshold: int = 1024) -> tuple[bytes, bool]:
    """Сжимаем большие значения"""
    if len(value) > threshold:
        return zlib.compress(value), True
    return value, False

def decompress_if_needed(value: bytes, is_compressed: bool) -> bytes:
    if is_compressed:
        return zlib.decompress(value)
    return value
```

### 3. Graceful degradation

```python
def get_with_fallback(key: str, fallback_fn, ttl: int = 3600):
    """Кэш не должен быть single point of failure"""
    try:
        cached = cache.get(key)
        if cached is not None:
            return cached
    except Exception as e:
        logger.warning(f"Memcached unavailable: {e}")

    # Fallback к источнику данных
    data = fallback_fn()

    # Пытаемся закэшировать
    try:
        cache.set(key, data, ttl)
    except Exception:
        pass

    return data
```

## Типичные ошибки

### 1. Слишком большие объекты

```python
# НЕПРАВИЛЬНО: объект больше 1MB не сохранится
large_data = 'x' * 2_000_000
cache.set('large', large_data)  # Молча проигнорируется

# ПРАВИЛЬНО: проверяйте размер или увеличьте лимит
MAX_ITEM_SIZE = 1024 * 1024  # 1MB

def safe_set(key, value, ttl=3600):
    serialized = json.dumps(value).encode()
    if len(serialized) > MAX_ITEM_SIZE:
        logger.warning(f"Value too large for memcached: {len(serialized)} bytes")
        return False
    return cache.set(key, value, ttl)
```

### 2. Игнорирование TTL

```python
# НЕПРАВИЛЬНО: данные живут вечно (до eviction)
cache.set('key', value)  # TTL = 0 = вечно

# ПРАВИЛЬНО: всегда указывайте TTL
cache.set('key', value, ttl=3600)
```

### 3. Нет обработки cache miss

```python
# НЕПРАВИЛЬНО
user = cache.get(f'user:{user_id}')
print(user['name'])  # TypeError если user is None

# ПРАВИЛЬНО
user = cache.get(f'user:{user_id}')
if user is None:
    user = db.users.find_one({'id': user_id})
    cache.set(f'user:{user_id}', user)
```

## Резюме

Memcached — отличный выбор для:
- Простого key-value кэширования
- Кэширования сессий
- Систем с высокой конкурентностью (multi-threaded)
- Когда не нужны сложные структуры данных

Ключевые моменты:
- Используйте consistent hashing для кластера
- Обрабатывайте отказы нод gracefully
- Мониторьте hit ratio и evictions
- Всегда указывайте TTL
- Помните об ограничении в 1MB на значение

---

[prev: 04-caching-strategies](./04-caching-strategies.md) | [next: 01-https](../10-web-security/01-https.md)
