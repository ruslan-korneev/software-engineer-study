# Серверное кэширование

## Что такое серверное кэширование?

**Серверное кэширование** — это техника хранения часто запрашиваемых данных в быстром хранилище на стороне сервера для уменьшения нагрузки на базу данных и ускорения ответов API.

## Уровни серверного кэширования

```
┌─────────────────────────────────────────────────────────────────┐
│                    Уровни кэширования                           │
├─────────────────────────────────────────────────────────────────┤
│  L1: In-Memory Cache (в памяти приложения)                      │
│      └── Самый быстрый, но локальный для процесса               │
│                                                                 │
│  L2: Distributed Cache (Redis, Memcached)                       │
│      └── Быстрый, разделяемый между инстансами                  │
│                                                                 │
│  L3: Database Query Cache                                       │
│      └── Кэш на уровне СУБД                                     │
│                                                                 │
│  L4: Database (источник данных)                                 │
│      └── Самый медленный, но авторитетный                       │
└─────────────────────────────────────────────────────────────────┘
```

## Redis — основной инструмент серверного кэширования

### Установка и подключение

```python
# Python с redis-py
import redis
from typing import Optional, Any
import json

class RedisCache:
    def __init__(self, host='localhost', port=6379, db=0, password=None):
        self.client = redis.Redis(
            host=host,
            port=port,
            db=db,
            password=password,
            decode_responses=True,  # Автоматическое декодирование в строки
            socket_timeout=5,
            socket_connect_timeout=5
        )

    def get(self, key: str) -> Optional[Any]:
        """Получить значение из кэша"""
        value = self.client.get(key)
        if value:
            return json.loads(value)
        return None

    def set(self, key: str, value: Any, ttl: int = 3600) -> bool:
        """Сохранить значение в кэш"""
        return self.client.setex(
            key,
            ttl,
            json.dumps(value, ensure_ascii=False)
        )

    def delete(self, key: str) -> bool:
        """Удалить ключ из кэша"""
        return bool(self.client.delete(key))

    def exists(self, key: str) -> bool:
        """Проверить существование ключа"""
        return bool(self.client.exists(key))

# Использование
cache = RedisCache()
cache.set('user:123', {'name': 'John', 'email': 'john@example.com'}, ttl=3600)
user = cache.get('user:123')
```

### Продвинутые паттерны работы с Redis

```python
import redis
import json
import hashlib
from functools import wraps
from typing import Callable, Any

class AdvancedRedisCache:
    def __init__(self, redis_client: redis.Redis):
        self.client = redis_client

    def cache_aside(self, key: str, ttl: int = 3600):
        """Декоратор для паттерна Cache-Aside"""
        def decorator(func: Callable) -> Callable:
            @wraps(func)
            def wrapper(*args, **kwargs):
                # Пробуем получить из кэша
                cached = self.client.get(key)
                if cached:
                    return json.loads(cached)

                # Cache miss - вызываем функцию
                result = func(*args, **kwargs)

                # Сохраняем в кэш
                self.client.setex(key, ttl, json.dumps(result))

                return result
            return wrapper
        return decorator

    def mget_with_fallback(self, keys: list, fallback_fn: Callable) -> dict:
        """Массовое получение с fallback для отсутствующих ключей"""
        # Получаем все ключи одним запросом
        values = self.client.mget(keys)

        result = {}
        missing_keys = []

        for key, value in zip(keys, values):
            if value:
                result[key] = json.loads(value)
            else:
                missing_keys.append(key)

        # Загружаем отсутствующие данные
        if missing_keys:
            missing_data = fallback_fn(missing_keys)
            # Сохраняем в кэш и добавляем в результат
            pipeline = self.client.pipeline()
            for key, value in missing_data.items():
                pipeline.setex(key, 3600, json.dumps(value))
                result[key] = value
            pipeline.execute()

        return result

    def get_or_set(self, key: str, default_fn: Callable, ttl: int = 3600) -> Any:
        """Получить значение или вычислить и сохранить"""
        value = self.client.get(key)
        if value:
            return json.loads(value)

        # Вычисляем значение
        result = default_fn()

        # Сохраняем
        self.client.setex(key, ttl, json.dumps(result))

        return result

# Пример использования
cache = AdvancedRedisCache(redis.Redis())

@cache.cache_aside('expensive_query_result', ttl=600)
def expensive_database_query():
    """Тяжёлый запрос к БД"""
    return db.execute("SELECT * FROM big_table WHERE complex_condition")
```

### Redis для кэширования сессий

```python
from flask import Flask, session
from flask_session import Session
import redis

app = Flask(__name__)

# Конфигурация сессий в Redis
app.config['SESSION_TYPE'] = 'redis'
app.config['SESSION_PERMANENT'] = True
app.config['SESSION_USE_SIGNER'] = True
app.config['SESSION_KEY_PREFIX'] = 'session:'
app.config['SESSION_REDIS'] = redis.Redis(
    host='localhost',
    port=6379,
    db=1  # Отдельная БД для сессий
)

Session(app)

@app.route('/login', methods=['POST'])
def login():
    user = authenticate(request.form['username'], request.form['password'])
    if user:
        session['user_id'] = user.id
        session['role'] = user.role
        return {'status': 'ok'}
    return {'status': 'error'}, 401
```

## Структуры данных Redis для кэширования

### Хэши (Hashes) — для объектов

```python
def cache_user_profile(user_id: int, profile: dict):
    """Кэширование профиля пользователя в хэше"""
    key = f'user:{user_id}:profile'

    # Сохраняем все поля разом
    cache.client.hset(key, mapping=profile)
    cache.client.expire(key, 3600)

def get_user_profile(user_id: int) -> dict:
    """Получение профиля"""
    key = f'user:{user_id}:profile'
    return cache.client.hgetall(key)

def update_user_field(user_id: int, field: str, value: str):
    """Обновление одного поля (без перезаписи всего объекта)"""
    key = f'user:{user_id}:profile'
    cache.client.hset(key, field, value)

# Использование
cache_user_profile(123, {
    'name': 'John Doe',
    'email': 'john@example.com',
    'avatar': 'https://example.com/avatar.jpg'
})

update_user_field(123, 'name', 'John Smith')  # Обновляем только имя
```

### Sorted Sets — для рейтингов и лидербордов

```python
def add_score(user_id: int, score: int):
    """Добавить очки пользователю"""
    cache.client.zadd('leaderboard', {f'user:{user_id}': score})

def get_top_players(limit: int = 10) -> list:
    """Получить топ игроков"""
    # ZREVRANGE возвращает от большего к меньшему
    return cache.client.zrevrange('leaderboard', 0, limit - 1, withscores=True)

def get_player_rank(user_id: int) -> int:
    """Получить ранг игрока"""
    rank = cache.client.zrevrank('leaderboard', f'user:{user_id}')
    return rank + 1 if rank is not None else None

# Использование
add_score(1, 1500)
add_score(2, 2000)
add_score(3, 1800)

top = get_top_players(3)
# [('user:2', 2000), ('user:3', 1800), ('user:1', 1500)]
```

### Lists — для очередей и последних элементов

```python
def add_to_recent(user_id: int, item_id: int, max_items: int = 10):
    """Добавить в список недавно просмотренных"""
    key = f'user:{user_id}:recent_views'

    pipeline = cache.client.pipeline()
    # Удаляем дубликаты
    pipeline.lrem(key, 0, item_id)
    # Добавляем в начало
    pipeline.lpush(key, item_id)
    # Обрезаем до max_items
    pipeline.ltrim(key, 0, max_items - 1)
    # Устанавливаем TTL
    pipeline.expire(key, 86400 * 7)  # 7 дней
    pipeline.execute()

def get_recent_views(user_id: int, limit: int = 10) -> list:
    """Получить недавно просмотренные"""
    key = f'user:{user_id}:recent_views'
    return cache.client.lrange(key, 0, limit - 1)
```

## Паттерны инвалидации кэша

### Tag-based инвалидация

```python
class TaggedCache:
    def __init__(self, redis_client: redis.Redis):
        self.client = redis_client

    def set_with_tags(self, key: str, value: Any, tags: list, ttl: int = 3600):
        """Сохранить значение с тегами для групповой инвалидации"""
        pipeline = self.client.pipeline()

        # Сохраняем значение
        pipeline.setex(key, ttl, json.dumps(value))

        # Добавляем ключ в множества тегов
        for tag in tags:
            pipeline.sadd(f'tag:{tag}', key)
            pipeline.expire(f'tag:{tag}', ttl)

        pipeline.execute()

    def invalidate_by_tag(self, tag: str):
        """Инвалидировать все ключи с тегом"""
        tag_key = f'tag:{tag}'

        # Получаем все ключи с этим тегом
        keys = self.client.smembers(tag_key)

        if keys:
            pipeline = self.client.pipeline()
            for key in keys:
                pipeline.delete(key)
            pipeline.delete(tag_key)
            pipeline.execute()

# Использование
cache = TaggedCache(redis.Redis())

# Кэшируем товар с тегами
cache.set_with_tags(
    'product:123',
    {'name': 'iPhone', 'price': 999},
    tags=['products', 'electronics', 'category:phones'],
    ttl=3600
)

# При обновлении категории инвалидируем все товары в ней
cache.invalidate_by_tag('category:phones')
```

### Write-through кэширование

```python
class WriteThroughCache:
    def __init__(self, cache: redis.Redis, db):
        self.cache = cache
        self.db = db

    def get(self, key: str) -> Optional[dict]:
        """Получить из кэша, при промахе — из БД"""
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        # Загружаем из БД
        data = self.db.find_one({'_id': key})
        if data:
            self.cache.setex(key, 3600, json.dumps(data))
        return data

    def set(self, key: str, value: dict):
        """Записать в БД и кэш одновременно"""
        # Сначала в БД (источник истины)
        self.db.update_one({'_id': key}, {'$set': value}, upsert=True)

        # Потом в кэш
        self.cache.setex(key, 3600, json.dumps(value))

    def delete(self, key: str):
        """Удалить из БД и кэша"""
        self.db.delete_one({'_id': key})
        self.cache.delete(key)
```

## In-Memory кэширование (локальный кэш)

### LRU Cache в Python

```python
from functools import lru_cache
from cachetools import TTLCache, cached
import time

# Встроенный LRU кэш
@lru_cache(maxsize=1000)
def get_config(key: str) -> dict:
    """Кэшированное получение конфигурации"""
    return db.configs.find_one({'key': key})

# TTL кэш с cachetools
config_cache = TTLCache(maxsize=100, ttl=300)

@cached(config_cache)
def get_feature_flags() -> dict:
    """Кэшированные feature flags"""
    return db.feature_flags.find()

# Ручная реализация с TTL
class LocalTTLCache:
    def __init__(self, default_ttl: int = 300):
        self._cache = {}
        self._ttl = default_ttl

    def get(self, key: str) -> Optional[Any]:
        if key in self._cache:
            value, expires_at = self._cache[key]
            if time.time() < expires_at:
                return value
            del self._cache[key]
        return None

    def set(self, key: str, value: Any, ttl: int = None):
        ttl = ttl or self._ttl
        self._cache[key] = (value, time.time() + ttl)

    def delete(self, key: str):
        self._cache.pop(key, None)
```

### Двухуровневый кэш (L1 + L2)

```python
class TwoLevelCache:
    """
    L1: Локальный кэш в памяти (быстрый, но локальный)
    L2: Redis (медленнее, но разделяемый)
    """
    def __init__(self, redis_client: redis.Redis, l1_size: int = 1000, l1_ttl: int = 60):
        self.l1 = TTLCache(maxsize=l1_size, ttl=l1_ttl)
        self.l2 = redis_client

    def get(self, key: str) -> Optional[Any]:
        # Проверяем L1
        if key in self.l1:
            return self.l1[key]

        # Проверяем L2
        value = self.l2.get(key)
        if value:
            parsed = json.loads(value)
            # Прогреваем L1
            self.l1[key] = parsed
            return parsed

        return None

    def set(self, key: str, value: Any, ttl: int = 3600):
        # Записываем в оба уровня
        self.l1[key] = value
        self.l2.setex(key, ttl, json.dumps(value))

    def delete(self, key: str):
        # Удаляем из обоих уровней
        self.l1.pop(key, None)
        self.l2.delete(key)

        # При работе с несколькими инстансами нужен pub/sub для инвалидации L1
        self.l2.publish('cache_invalidation', key)
```

## Лучшие практики

### 1. Правильное именование ключей

```python
# Хорошо: читаемые, структурированные ключи
'user:123:profile'
'product:456:details'
'session:abc123'
'cache:api:v1:users:list:page:1'

# Плохо: непонятные или слишком длинные ключи
'u123p'
'my_application_name:module:submodule:feature:subfeature:entity:id:attribute'
```

### 2. Сериализация

```python
import msgpack  # Быстрее и компактнее JSON

def serialize(data: Any) -> bytes:
    return msgpack.packb(data, use_bin_type=True)

def deserialize(data: bytes) -> Any:
    return msgpack.unpackb(data, raw=False)
```

### 3. Обработка ошибок

```python
def get_with_fallback(key: str, fallback_fn: Callable) -> Any:
    """Получить из кэша с fallback при ошибках"""
    try:
        value = cache.get(key)
        if value:
            return value
    except redis.RedisError as e:
        logger.warning(f"Redis error: {e}")
        # При ошибке Redis продолжаем работу без кэша

    return fallback_fn()
```

## Типичные ошибки

### 1. Кэширование null/None значений

```python
# Проблема: Cache stampede при отсутствующих данных
def get_user(user_id: int):
    cached = cache.get(f'user:{user_id}')
    if cached:
        return cached

    user = db.users.find_one({'id': user_id})
    if user:
        cache.set(f'user:{user_id}', user)
    return user  # При user=None каждый запрос идёт в БД

# Решение: кэшируем null-значения
CACHE_NULL = '__NULL__'

def get_user_fixed(user_id: int):
    cached = cache.get(f'user:{user_id}')
    if cached == CACHE_NULL:
        return None
    if cached:
        return cached

    user = db.users.find_one({'id': user_id})
    cache.set(f'user:{user_id}', user if user else CACHE_NULL, ttl=300)
    return user
```

### 2. Одинаковый TTL для всех ключей

```python
# Проблема: массовое истечение кэша (thundering herd)

# Решение: добавляем jitter к TTL
import random

def get_ttl_with_jitter(base_ttl: int, jitter_percent: float = 0.1) -> int:
    """TTL с случайным отклонением"""
    jitter = int(base_ttl * jitter_percent)
    return base_ttl + random.randint(-jitter, jitter)

cache.set('key', value, ttl=get_ttl_with_jitter(3600))  # 3240-3960 секунд
```

### 3. Отсутствие мониторинга

```python
# Обязательно отслеживайте метрики кэша
class MonitoredCache:
    def __init__(self, cache: redis.Redis, metrics):
        self.cache = cache
        self.metrics = metrics

    def get(self, key: str):
        start = time.time()
        value = self.cache.get(key)

        self.metrics.histogram('cache_latency_seconds', time.time() - start)

        if value:
            self.metrics.counter('cache_hits_total').inc()
        else:
            self.metrics.counter('cache_misses_total').inc()

        return value
```

## Резюме

Серверное кэширование — ключевая техника для построения масштабируемых приложений:

- **Redis** — основной инструмент для distributed cache
- **In-memory cache** — для горячих данных на уровне процесса
- **Правильная инвалидация** — критически важна для консистентности
- **Мониторинг** — обязателен для понимания эффективности кэша
- **Обработка ошибок** — кэш не должен быть single point of failure
