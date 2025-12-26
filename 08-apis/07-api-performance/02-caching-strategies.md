# Caching Strategies

## Введение

Кэширование — это техника хранения копий часто запрашиваемых данных в быстром хранилище для уменьшения времени доступа и снижения нагрузки на основные системы. Правильно реализованное кэширование может улучшить производительность API на порядки величин.

## Типы кэширования

### Архитектура многоуровневого кэширования

```
┌─────────────────────────────────────────────────────────────────┐
│                         CLIENT                                   │
│                   (Browser Cache, App Cache)                     │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                          CDN                                     │
│                (Edge Caching, Geographic Distribution)           │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                     REVERSE PROXY                                │
│               (Nginx, Varnish, HAProxy)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                   APPLICATION CACHE                              │
│              (Redis, Memcached, In-Memory)                       │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                    DATABASE CACHE                                │
│              (Query Cache, Buffer Pool)                          │
└─────────────────────────────────────────────────────────────────┘
                              │
                              ▼
┌─────────────────────────────────────────────────────────────────┐
│                      DATABASE                                    │
│                  (Primary Data Store)                            │
└─────────────────────────────────────────────────────────────────┘
```

## HTTP Caching

### Cache-Control Headers

```python
from fastapi import FastAPI, Response
from fastapi.responses import JSONResponse
from datetime import datetime, timedelta
import hashlib
import json

app = FastAPI()

class CacheHeaders:
    """Класс для управления HTTP-заголовками кэширования"""

    @staticmethod
    def public(max_age: int = 3600) -> dict:
        """
        Public cache — может кэшироваться CDN и браузерами
        Используйте для публичных данных без персонализации
        """
        return {
            "Cache-Control": f"public, max-age={max_age}",
            "Vary": "Accept-Encoding"
        }

    @staticmethod
    def private(max_age: int = 300) -> dict:
        """
        Private cache — только браузер пользователя
        Для персонализированных данных
        """
        return {
            "Cache-Control": f"private, max-age={max_age}",
            "Vary": "Accept-Encoding, Authorization"
        }

    @staticmethod
    def no_store() -> dict:
        """
        Запрет любого кэширования
        Для конфиденциальных данных
        """
        return {
            "Cache-Control": "no-store, no-cache, must-revalidate",
            "Pragma": "no-cache"
        }

    @staticmethod
    def stale_while_revalidate(max_age: int = 60, stale: int = 3600) -> dict:
        """
        Позволяет отдавать устаревшие данные пока обновляются свежие
        Отличный баланс между производительностью и свежестью
        """
        return {
            "Cache-Control": f"public, max-age={max_age}, stale-while-revalidate={stale}"
        }

@app.get("/api/products")
async def get_products(response: Response):
    """Публичный список товаров — активно кэшируем"""
    products = await fetch_products()

    headers = CacheHeaders.stale_while_revalidate(max_age=60, stale=3600)
    for key, value in headers.items():
        response.headers[key] = value

    return products

@app.get("/api/users/me")
async def get_current_user(response: Response):
    """Персональные данные — только private cache"""
    user = await get_user_from_token()

    headers = CacheHeaders.private(max_age=300)
    for key, value in headers.items():
        response.headers[key] = value

    return user
```

### ETag и Conditional Requests

```python
from fastapi import FastAPI, Request, Response, HTTPException
import hashlib
import json

app = FastAPI()

def generate_etag(data: dict) -> str:
    """Генерация ETag на основе содержимого"""
    content = json.dumps(data, sort_keys=True)
    return f'"{hashlib.md5(content.encode()).hexdigest()}"'

@app.get("/api/articles/{article_id}")
async def get_article(
    article_id: int,
    request: Request,
    response: Response
):
    article = await fetch_article(article_id)

    # Генерируем ETag
    etag = generate_etag(article)

    # Проверяем If-None-Match от клиента
    if_none_match = request.headers.get("If-None-Match")
    if if_none_match == etag:
        # Данные не изменились — возвращаем 304
        raise HTTPException(status_code=304)

    response.headers["ETag"] = etag
    response.headers["Cache-Control"] = "private, max-age=0, must-revalidate"

    return article

@app.put("/api/articles/{article_id}")
async def update_article(
    article_id: int,
    request: Request,
    article_data: dict
):
    # Проверяем If-Match для предотвращения конфликтов
    if_match = request.headers.get("If-Match")

    current_article = await fetch_article(article_id)
    current_etag = generate_etag(current_article)

    if if_match and if_match != current_etag:
        # Данные изменились с момента последнего чтения
        raise HTTPException(
            status_code=412,
            detail="Precondition Failed: Article was modified"
        )

    updated = await save_article(article_id, article_data)
    return updated
```

## Application-Level Caching

### Redis как кэш

```python
import redis.asyncio as redis
import json
from typing import Optional, Any, Callable
from datetime import timedelta
from functools import wraps
import hashlib

class RedisCache:
    """Асинхронный Redis кэш с расширенными возможностями"""

    def __init__(self, redis_url: str = "redis://localhost:6379"):
        self.redis = redis.from_url(redis_url, decode_responses=True)

    async def get(self, key: str) -> Optional[Any]:
        """Получение значения из кэша"""
        value = await self.redis.get(key)
        if value:
            return json.loads(value)
        return None

    async def set(
        self,
        key: str,
        value: Any,
        ttl: int = 3600,
        nx: bool = False  # Set if Not eXists
    ) -> bool:
        """Сохранение значения в кэш"""
        serialized = json.dumps(value, default=str)
        if nx:
            return await self.redis.setnx(key, serialized)
        await self.redis.setex(key, ttl, serialized)
        return True

    async def delete(self, key: str) -> bool:
        """Удаление ключа"""
        return await self.redis.delete(key) > 0

    async def delete_pattern(self, pattern: str) -> int:
        """Удаление ключей по паттерну"""
        cursor = 0
        deleted = 0
        while True:
            cursor, keys = await self.redis.scan(cursor, match=pattern, count=100)
            if keys:
                deleted += await self.redis.delete(*keys)
            if cursor == 0:
                break
        return deleted

    async def get_or_set(
        self,
        key: str,
        factory: Callable,
        ttl: int = 3600
    ) -> Any:
        """Получить из кэша или вычислить и сохранить"""
        value = await self.get(key)
        if value is not None:
            return value

        # Cache miss — вычисляем значение
        value = await factory()
        await self.set(key, value, ttl)
        return value

# Декоратор для кэширования
def cached(
    cache: RedisCache,
    prefix: str,
    ttl: int = 3600,
    key_builder: Optional[Callable] = None
):
    """Декоратор для автоматического кэширования результатов функции"""

    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            # Строим ключ кэша
            if key_builder:
                cache_key = key_builder(*args, **kwargs)
            else:
                key_parts = [prefix, func.__name__]
                key_parts.extend(str(arg) for arg in args)
                key_parts.extend(f"{k}={v}" for k, v in sorted(kwargs.items()))
                cache_key = ":".join(key_parts)

            # Пробуем получить из кэша
            cached_value = await cache.get(cache_key)
            if cached_value is not None:
                return cached_value

            # Вычисляем и кэшируем
            result = await func(*args, **kwargs)
            await cache.set(cache_key, result, ttl)
            return result

        # Добавляем метод для инвалидации
        wrapper.invalidate = lambda *a, **kw: cache.delete(
            key_builder(*a, **kw) if key_builder else
            ":".join([prefix, func.__name__] + [str(x) for x in a])
        )

        return wrapper
    return decorator

# Использование
cache = RedisCache()

@cached(cache, prefix="products", ttl=300)
async def get_product(product_id: int) -> dict:
    """Результат кэшируется на 5 минут"""
    return await db.fetch_product(product_id)

# Инвалидация при обновлении
async def update_product(product_id: int, data: dict):
    await db.update_product(product_id, data)
    await get_product.invalidate(product_id)
```

### Cache-Aside Pattern (Lazy Loading)

```python
from typing import TypeVar, Generic, Optional, Callable, Awaitable
from dataclasses import dataclass
from datetime import datetime, timedelta

T = TypeVar('T')

@dataclass
class CacheEntry(Generic[T]):
    value: T
    created_at: datetime
    ttl_seconds: int

    @property
    def is_expired(self) -> bool:
        return datetime.now() > self.created_at + timedelta(seconds=self.ttl_seconds)

class CacheAsidePattern(Generic[T]):
    """
    Cache-Aside (Lazy Loading) Pattern

    1. Приложение сначала проверяет кэш
    2. При cache miss — читает из БД
    3. Записывает результат в кэш
    4. Возвращает данные
    """

    def __init__(self, cache: RedisCache, prefix: str):
        self.cache = cache
        self.prefix = prefix

    async def get(
        self,
        key: str,
        loader: Callable[[], Awaitable[T]],
        ttl: int = 3600
    ) -> Optional[T]:
        cache_key = f"{self.prefix}:{key}"

        # 1. Проверяем кэш
        cached = await self.cache.get(cache_key)
        if cached is not None:
            return cached

        # 2. Cache miss — загружаем из источника
        value = await loader()

        # 3. Сохраняем в кэш (если значение не None)
        if value is not None:
            await self.cache.set(cache_key, value, ttl)

        return value

    async def invalidate(self, key: str):
        """Инвалидация конкретного ключа"""
        await self.cache.delete(f"{self.prefix}:{key}")

    async def invalidate_all(self):
        """Инвалидация всех ключей с данным префиксом"""
        await self.cache.delete_pattern(f"{self.prefix}:*")

# Использование
user_cache = CacheAsidePattern[dict](cache, "users")

async def get_user(user_id: int) -> dict:
    return await user_cache.get(
        key=str(user_id),
        loader=lambda: db.get_user(user_id),
        ttl=600
    )
```

### Write-Through и Write-Behind Patterns

```python
import asyncio
from collections import deque
from typing import Dict, Any, List, Tuple
from datetime import datetime

class WriteThrough:
    """
    Write-Through Pattern

    При записи данные сохраняются и в кэш, и в БД синхронно.
    Гарантирует консистентность, но увеличивает latency записи.
    """

    def __init__(self, cache: RedisCache, db):
        self.cache = cache
        self.db = db

    async def write(self, key: str, value: Any, ttl: int = 3600) -> bool:
        try:
            # Записываем в БД
            await self.db.save(key, value)

            # Записываем в кэш
            await self.cache.set(key, value, ttl)

            return True
        except Exception as e:
            # При ошибке — инвалидируем кэш
            await self.cache.delete(key)
            raise

class WriteBehind:
    """
    Write-Behind (Write-Back) Pattern

    Данные сначала записываются в кэш, затем асинхронно в БД.
    Лучшая производительность записи, но риск потери данных.
    """

    def __init__(self, cache: RedisCache, db, batch_size: int = 100):
        self.cache = cache
        self.db = db
        self.batch_size = batch_size
        self.write_queue: deque = deque()
        self._running = False

    async def write(self, key: str, value: Any, ttl: int = 3600):
        # Записываем в кэш немедленно
        await self.cache.set(key, value, ttl)

        # Добавляем в очередь для записи в БД
        self.write_queue.append((key, value, datetime.now()))

    async def start_background_writer(self):
        """Фоновый процесс записи в БД"""
        self._running = True
        while self._running:
            await self._flush_batch()
            await asyncio.sleep(1)  # Пауза между batch-ами

    async def _flush_batch(self):
        """Записываем batch в БД"""
        if not self.write_queue:
            return

        batch: List[Tuple[str, Any]] = []
        while self.write_queue and len(batch) < self.batch_size:
            key, value, _ = self.write_queue.popleft()
            batch.append((key, value))

        if batch:
            try:
                await self.db.bulk_save(batch)
            except Exception as e:
                # При ошибке — возвращаем в очередь
                for item in batch:
                    self.write_queue.appendleft((*item, datetime.now()))
                raise

    async def stop(self):
        """Остановка и финальный flush"""
        self._running = False
        while self.write_queue:
            await self._flush_batch()
```

## Стратегии инвалидации кэша

### Time-Based Expiration (TTL)

```python
class TTLStrategy:
    """
    Стратегии выбора TTL для разных типов данных
    """

    # Статические справочники — долгий TTL
    STATIC_DATA = 86400  # 24 часа

    # Агрегированные данные — средний TTL
    AGGREGATED_DATA = 3600  # 1 час

    # Часто меняющиеся данные — короткий TTL
    DYNAMIC_DATA = 300  # 5 минут

    # Сессионные данные — по времени сессии
    SESSION_DATA = 1800  # 30 минут

    @staticmethod
    def get_ttl(data_type: str) -> int:
        ttl_map = {
            "countries": TTLStrategy.STATIC_DATA,
            "product_catalog": TTLStrategy.AGGREGATED_DATA,
            "user_profile": TTLStrategy.DYNAMIC_DATA,
            "shopping_cart": TTLStrategy.SESSION_DATA,
            "stock_levels": 60,  # Очень динамичные данные
            "exchange_rates": 300,
        }
        return ttl_map.get(data_type, TTLStrategy.DYNAMIC_DATA)
```

### Event-Based Invalidation

```python
from typing import Set, Dict, Callable, List
import asyncio

class CacheInvalidator:
    """Инвалидация кэша на основе событий"""

    def __init__(self, cache: RedisCache):
        self.cache = cache
        self.subscribers: Dict[str, List[Callable]] = {}

    def subscribe(self, event: str, handler: Callable):
        """Подписка на событие инвалидации"""
        if event not in self.subscribers:
            self.subscribers[event] = []
        self.subscribers[event].append(handler)

    async def emit(self, event: str, data: dict):
        """Эмиссия события инвалидации"""
        handlers = self.subscribers.get(event, [])
        await asyncio.gather(*[
            handler(data) for handler in handlers
        ])

    async def invalidate_related(self, entity: str, entity_id: int):
        """
        Инвалидация связанных кэшей

        Пример: при обновлении продукта инвалидируем:
        - Кэш самого продукта
        - Список продуктов категории
        - Корзины с этим продуктом
        """
        invalidation_rules = {
            "product": [
                f"product:{entity_id}",
                f"category:*:products",  # pattern
                f"cart:*:items",  # pattern
            ],
            "user": [
                f"user:{entity_id}",
                f"user:{entity_id}:*",  # все связанные
            ],
            "order": [
                f"order:{entity_id}",
                f"user:*:orders",
            ]
        }

        patterns = invalidation_rules.get(entity, [])
        for pattern in patterns:
            if '*' in pattern:
                await self.cache.delete_pattern(pattern)
            else:
                await self.cache.delete(pattern)

# Интеграция с ORM (пример с SQLAlchemy events)
from sqlalchemy import event

invalidator = CacheInvalidator(cache)

@event.listens_for(Product, 'after_update')
def invalidate_product_cache(mapper, connection, target):
    asyncio.create_task(
        invalidator.invalidate_related("product", target.id)
    )
```

### Cache Tags

```python
from typing import Set, List

class TaggedCache:
    """
    Кэширование с тегами для групповой инвалидации

    Пример: кэш страницы продукта может иметь теги:
    - product:123
    - category:electronics
    - brand:apple

    При изменении категории — инвалидируем все связанные кэши
    """

    def __init__(self, cache: RedisCache):
        self.cache = cache
        self.tags_prefix = "cache:tags"

    async def set_with_tags(
        self,
        key: str,
        value: any,
        tags: List[str],
        ttl: int = 3600
    ):
        """Сохранение значения с тегами"""
        # Сохраняем само значение
        await self.cache.set(key, value, ttl)

        # Добавляем ключ к каждому тегу
        for tag in tags:
            tag_key = f"{self.tags_prefix}:{tag}"
            await self.cache.redis.sadd(tag_key, key)
            await self.cache.redis.expire(tag_key, ttl + 3600)  # Теги живут дольше

    async def invalidate_by_tag(self, tag: str) -> int:
        """Инвалидация всех ключей с определённым тегом"""
        tag_key = f"{self.tags_prefix}:{tag}"

        # Получаем все ключи с этим тегом
        keys = await self.cache.redis.smembers(tag_key)

        if not keys:
            return 0

        # Удаляем все связанные ключи
        deleted = await self.cache.redis.delete(*keys)

        # Удаляем сам тег
        await self.cache.redis.delete(tag_key)

        return deleted

# Использование
tagged_cache = TaggedCache(cache)

async def cache_product_page(product: dict):
    key = f"page:product:{product['id']}"
    tags = [
        f"product:{product['id']}",
        f"category:{product['category_id']}",
        f"brand:{product['brand']}",
    ]
    await tagged_cache.set_with_tags(key, product, tags)

# При обновлении категории — инвалидируем все продукты категории
async def update_category(category_id: int, data: dict):
    await db.update_category(category_id, data)
    await tagged_cache.invalidate_by_tag(f"category:{category_id}")
```

## Распределённое кэширование

### Consistent Hashing

```python
import hashlib
from typing import List, Optional
from bisect import bisect_left

class ConsistentHash:
    """
    Consistent Hashing для распределения ключей между серверами кэша

    Преимущества:
    - При добавлении/удалении сервера перераспределяется минимум ключей
    - Равномерное распределение нагрузки
    """

    def __init__(self, nodes: List[str], virtual_nodes: int = 150):
        self.virtual_nodes = virtual_nodes
        self.ring: List[int] = []
        self.ring_to_node: dict = {}

        for node in nodes:
            self.add_node(node)

    def _hash(self, key: str) -> int:
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node: str):
        """Добавление сервера в кольцо"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)

            # Вставляем в отсортированное кольцо
            idx = bisect_left(self.ring, hash_value)
            self.ring.insert(idx, hash_value)
            self.ring_to_node[hash_value] = node

    def remove_node(self, node: str):
        """Удаление сервера из кольца"""
        for i in range(self.virtual_nodes):
            virtual_key = f"{node}:{i}"
            hash_value = self._hash(virtual_key)

            if hash_value in self.ring_to_node:
                self.ring.remove(hash_value)
                del self.ring_to_node[hash_value]

    def get_node(self, key: str) -> Optional[str]:
        """Получение сервера для ключа"""
        if not self.ring:
            return None

        hash_value = self._hash(key)
        idx = bisect_left(self.ring, hash_value)

        if idx == len(self.ring):
            idx = 0

        return self.ring_to_node[self.ring[idx]]

# Использование
nodes = ["redis-1:6379", "redis-2:6379", "redis-3:6379"]
hash_ring = ConsistentHash(nodes)

# Определяем, на какой сервер идти за ключом
server = hash_ring.get_node("user:12345")
```

### Redis Cluster

```python
from redis.cluster import RedisCluster
from redis.cluster import ClusterNode

class RedisClusterCache:
    """Кэш на базе Redis Cluster"""

    def __init__(self, startup_nodes: List[dict]):
        nodes = [ClusterNode(**node) for node in startup_nodes]
        self.cluster = RedisCluster(
            startup_nodes=nodes,
            decode_responses=True,
            skip_full_coverage_check=True
        )

    async def get(self, key: str) -> Optional[str]:
        return self.cluster.get(key)

    async def set(self, key: str, value: str, ttl: int = 3600):
        self.cluster.setex(key, ttl, value)

    async def mget(self, keys: List[str]) -> List[Optional[str]]:
        """
        Multi-get с учётом слотов

        В Redis Cluster ключи распределены по слотам.
        Для эффективного mget используем hash tags: {user}:1, {user}:2
        """
        return self.cluster.mget(keys)

# Конфигурация
startup_nodes = [
    {"host": "redis-node-1", "port": 6379},
    {"host": "redis-node-2", "port": 6379},
    {"host": "redis-node-3", "port": 6379},
]

cluster_cache = RedisClusterCache(startup_nodes)
```

## CDN Caching

### Настройка Cache-Control для CDN

```python
from fastapi import FastAPI, Response
from enum import Enum

class CDNCachePolicy(Enum):
    """Политики кэширования для CDN"""

    # Статические ресурсы — агрессивное кэширование
    STATIC = "public, max-age=31536000, immutable"

    # API данные — умеренное кэширование с revalidation
    API_PUBLIC = "public, max-age=60, stale-while-revalidate=300"

    # Персонализированные данные — не кэшировать на CDN
    PRIVATE = "private, max-age=0, must-revalidate"

    # Чувствительные данные — полный запрет
    NO_STORE = "no-store"

app = FastAPI()

def cdn_cache(policy: CDNCachePolicy):
    """Декоратор для установки CDN кэширования"""
    def decorator(func):
        async def wrapper(*args, response: Response, **kwargs):
            result = await func(*args, **kwargs)
            response.headers["Cache-Control"] = policy.value

            # Для CloudFlare
            if policy == CDNCachePolicy.API_PUBLIC:
                response.headers["CDN-Cache-Control"] = "max-age=60"

            # Для Fastly
            response.headers["Surrogate-Control"] = policy.value

            return result
        return wrapper
    return decorator

@app.get("/api/public/catalog")
@cdn_cache(CDNCachePolicy.API_PUBLIC)
async def get_catalog(response: Response):
    return {"items": [...]}
```

### Purge API

```python
import httpx
from typing import List

class CDNPurger:
    """Инвалидация кэша CDN"""

    def __init__(self, cdn_provider: str, api_key: str):
        self.provider = cdn_provider
        self.api_key = api_key

    async def purge_urls(self, urls: List[str]):
        """Purge конкретных URL"""
        if self.provider == "cloudflare":
            await self._purge_cloudflare(urls)
        elif self.provider == "fastly":
            await self._purge_fastly(urls)

    async def _purge_cloudflare(self, urls: List[str]):
        async with httpx.AsyncClient() as client:
            await client.post(
                f"https://api.cloudflare.com/client/v4/zones/{self.zone_id}/purge_cache",
                headers={"Authorization": f"Bearer {self.api_key}"},
                json={"files": urls}
            )

    async def purge_by_tag(self, tag: str):
        """Purge по тегу (Surrogate-Key)"""
        # Fastly поддерживает purge по Surrogate-Key
        async with httpx.AsyncClient() as client:
            await client.post(
                f"https://api.fastly.com/service/{self.service_id}/purge/{tag}",
                headers={"Fastly-Key": self.api_key}
            )
```

## Best Practices

### 1. Выбор стратегии кэширования

```python
"""
Матрица выбора стратегии кэширования:

┌─────────────────────────────────────────────────────────────────┐
│ Характеристика данных        │ Рекомендуемая стратегия         │
├─────────────────────────────────────────────────────────────────┤
│ Редко меняются, часто читаются → Cache-Aside + Long TTL       │
│ Часто меняются, часто читаются → Write-Through + Short TTL    │
│ Пишутся часто, читаются редко  → Write-Behind (если допустимо) │
│ Критичная консистентность      → No Cache или очень короткий TTL│
│ Высокая нагрузка на чтение     → Multi-layer cache             │
└─────────────────────────────────────────────────────────────────┘
"""

class CacheStrategyAdvisor:
    """Помощник в выборе стратегии кэширования"""

    @staticmethod
    def recommend(
        read_frequency: str,  # low, medium, high
        write_frequency: str,  # low, medium, high
        consistency_requirement: str,  # eventual, strong
        data_size: str  # small, medium, large
    ) -> dict:

        recommendations = {
            "strategy": None,
            "ttl": None,
            "invalidation": None,
            "layers": []
        }

        if read_frequency == "high" and write_frequency == "low":
            recommendations["strategy"] = "cache-aside"
            recommendations["ttl"] = 3600
            recommendations["layers"] = ["local", "distributed"]

        elif write_frequency == "high" and consistency_requirement == "strong":
            recommendations["strategy"] = "write-through"
            recommendations["ttl"] = 300
            recommendations["invalidation"] = "event-based"

        elif write_frequency == "high" and consistency_requirement == "eventual":
            recommendations["strategy"] = "write-behind"
            recommendations["ttl"] = 600
            recommendations["invalidation"] = "time-based"

        return recommendations
```

### 2. Мониторинг кэша

```python
from prometheus_client import Counter, Gauge, Histogram

class CacheMetrics:
    """Метрики для мониторинга эффективности кэша"""

    # Hit/Miss ratio
    cache_hits = Counter(
        'cache_hits_total',
        'Total cache hits',
        ['cache_name', 'key_pattern']
    )

    cache_misses = Counter(
        'cache_misses_total',
        'Total cache misses',
        ['cache_name', 'key_pattern']
    )

    # Размер кэша
    cache_size = Gauge(
        'cache_size_bytes',
        'Current cache size in bytes',
        ['cache_name']
    )

    # Время операций
    cache_operation_duration = Histogram(
        'cache_operation_duration_seconds',
        'Cache operation duration',
        ['cache_name', 'operation'],
        buckets=[0.001, 0.005, 0.01, 0.025, 0.05, 0.1]
    )

    @classmethod
    def get_hit_rate(cls, cache_name: str) -> float:
        """Расчёт hit rate"""
        hits = cls.cache_hits.labels(cache_name=cache_name)._value.get()
        misses = cls.cache_misses.labels(cache_name=cache_name)._value.get()
        total = hits + misses
        return hits / total if total > 0 else 0.0
```

### 3. Предотвращение проблем

```python
class CacheProtection:
    """Защита от типичных проблем кэширования"""

    def __init__(self, cache: RedisCache):
        self.cache = cache
        self.locks: dict = {}

    async def get_with_stampede_protection(
        self,
        key: str,
        loader: Callable,
        ttl: int = 3600,
        lock_timeout: int = 5
    ):
        """
        Cache Stampede Protection (Thundering Herd)

        Когда кэш истекает, множество запросов одновременно
        пытаются его обновить. Решение — распределённая блокировка.
        """
        value = await self.cache.get(key)
        if value is not None:
            return value

        lock_key = f"lock:{key}"

        # Пробуем получить блокировку
        acquired = await self.cache.set(lock_key, "1", ttl=lock_timeout, nx=True)

        if acquired:
            try:
                # Мы получили блокировку — загружаем данные
                value = await loader()
                await self.cache.set(key, value, ttl)
                return value
            finally:
                await self.cache.delete(lock_key)
        else:
            # Кто-то другой загружает — ждём
            for _ in range(lock_timeout * 10):
                await asyncio.sleep(0.1)
                value = await self.cache.get(key)
                if value is not None:
                    return value

            # Таймаут — пробуем сами
            return await loader()

    async def get_with_negative_cache(
        self,
        key: str,
        loader: Callable,
        ttl: int = 3600,
        negative_ttl: int = 60
    ):
        """
        Negative Caching (Cache Penetration Protection)

        Кэшируем также отсутствие данных, чтобы избежать
        повторных запросов к БД для несуществующих ключей.
        """
        NEGATIVE_MARKER = "__NULL__"

        value = await self.cache.get(key)

        if value == NEGATIVE_MARKER:
            return None

        if value is not None:
            return value

        # Cache miss
        value = await loader()

        if value is None:
            # Кэшируем отсутствие
            await self.cache.set(key, NEGATIVE_MARKER, negative_ttl)
        else:
            await self.cache.set(key, value, ttl)

        return value
```

## Заключение

Эффективное кэширование требует:

1. **Правильного выбора уровня кэширования** — от браузера до БД
2. **Выбора стратегии** — Cache-Aside, Write-Through или Write-Behind
3. **Продуманной инвалидации** — TTL, events, tags
4. **Мониторинга** — hit rate, latency, size
5. **Защиты от проблем** — stampede, penetration, avalanche

Кэширование — это компромисс между производительностью и консистентностью данных. Выбирайте стратегию исходя из требований вашего приложения.
