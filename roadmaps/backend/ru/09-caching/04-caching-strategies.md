# Стратегии кэширования

## Обзор стратегий

Выбор правильной стратегии кэширования критически важен для баланса между производительностью, консистентностью данных и сложностью реализации.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Основные стратегии                           │
├─────────────────────────────────────────────────────────────────┤
│  Cache-Aside (Lazy Loading)                                     │
│  └── Приложение управляет кэшем самостоятельно                  │
│                                                                 │
│  Read-Through                                                   │
│  └── Кэш загружает данные из БД при промахе                     │
│                                                                 │
│  Write-Through                                                  │
│  └── Запись в кэш и БД одновременно                             │
│                                                                 │
│  Write-Behind (Write-Back)                                      │
│  └── Асинхронная запись в БД после записи в кэш                 │
│                                                                 │
│  Refresh-Ahead                                                  │
│  └── Проактивное обновление до истечения TTL                    │
└─────────────────────────────────────────────────────────────────┘
```

## Cache-Aside (Lazy Loading)

### Принцип работы

Самая распространённая стратегия, где приложение напрямую управляет кэшем.

```
        Чтение:                              Запись:
        ┌─────┐                              ┌─────┐
        │ App │                              │ App │
        └──┬──┘                              └──┬──┘
           │                                    │
    1. get │     2. if miss                     │ 1. write
           ▼           │                        ▼
        ┌─────┐        │                     ┌─────┐
        │Cache│        │                     │ DB  │
        └─────┘        ▼                     └─────┘
                    ┌─────┐                     │
                    │ DB  │                     │ 2. invalidate
                    └─────┘                     ▼
                       │                     ┌─────┐
                       │ 3. set              │Cache│
                       ▼                     └─────┘
                    ┌─────┐
                    │Cache│
                    └─────┘
```

### Реализация

```python
import redis
import json
from typing import Optional, Any, Callable

class CacheAside:
    """
    Cache-Aside (Lazy Loading) стратегия.

    Плюсы:
    - Простота реализации
    - Кэшируются только запрашиваемые данные
    - Отказ кэша не блокирует приложение

    Минусы:
    - Cache miss = дополнительная задержка
    - Возможна рассинхронизация данных
    - Каждый cache miss загружает БД
    """

    def __init__(self, cache: redis.Redis, default_ttl: int = 3600):
        self.cache = cache
        self.default_ttl = default_ttl

    def get(self, key: str, loader: Callable[[], Any], ttl: int = None) -> Any:
        """
        Получить данные из кэша или загрузить из источника.

        Args:
            key: Ключ кэша
            loader: Функция для загрузки данных при промахе
            ttl: Время жизни в секундах
        """
        ttl = ttl or self.default_ttl

        # 1. Пробуем получить из кэша
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        # 2. Cache miss — загружаем из источника
        data = loader()

        # 3. Сохраняем в кэш
        if data is not None:
            self.cache.setex(key, ttl, json.dumps(data))

        return data

    def invalidate(self, key: str):
        """Инвалидировать кэш при изменении данных"""
        self.cache.delete(key)

    def invalidate_pattern(self, pattern: str):
        """Инвалидировать все ключи по паттерну"""
        keys = self.cache.keys(pattern)
        if keys:
            self.cache.delete(*keys)


# Пример использования
cache = CacheAside(redis.Redis())

def get_user(user_id: int) -> dict:
    """Получить пользователя с кэшированием"""
    return cache.get(
        key=f'user:{user_id}',
        loader=lambda: db.users.find_one({'id': user_id}),
        ttl=3600
    )

def update_user(user_id: int, data: dict):
    """Обновить пользователя с инвалидацией кэша"""
    db.users.update_one({'id': user_id}, {'$set': data})
    cache.invalidate(f'user:{user_id}')
```

### Предотвращение Cache Stampede

```python
import threading
import time

class CacheAsideWithLock:
    """Cache-Aside с защитой от cache stampede"""

    def __init__(self, cache: redis.Redis):
        self.cache = cache
        self._locks = {}
        self._lock = threading.Lock()

    def get(self, key: str, loader: Callable, ttl: int = 3600) -> Any:
        # Проверяем кэш
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        # Получаем или создаём лок для ключа
        with self._lock:
            if key not in self._locks:
                self._locks[key] = threading.Lock()
            key_lock = self._locks[key]

        # Только один поток загружает данные
        with key_lock:
            # Повторная проверка — может другой поток уже загрузил
            cached = self.cache.get(key)
            if cached:
                return json.loads(cached)

            # Загружаем данные
            data = loader()
            if data is not None:
                self.cache.setex(key, ttl, json.dumps(data))

            return data


# Альтернатива: Redis-based lock
class CacheAsideWithRedisLock:
    def get_with_lock(self, key: str, loader: Callable, ttl: int = 3600) -> Any:
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        lock_key = f'lock:{key}'

        # Пытаемся захватить лок
        if self.cache.set(lock_key, '1', nx=True, ex=10):
            try:
                data = loader()
                if data is not None:
                    self.cache.setex(key, ttl, json.dumps(data))
                return data
            finally:
                self.cache.delete(lock_key)
        else:
            # Лок занят — ждём и повторяем
            time.sleep(0.1)
            return self.get_with_lock(key, loader, ttl)
```

## Read-Through

### Принцип работы

Кэш сам отвечает за загрузку данных при промахе. Приложение всегда обращается только к кэшу.

```
        ┌─────┐
        │ App │
        └──┬──┘
           │ 1. get (всегда)
           ▼
        ┌─────┐
        │Cache│──────────────┐
        └─────┘              │
           │                 │ 2. if miss, load
           │                 ▼
           │              ┌─────┐
           │              │ DB  │
           │              └─────┘
           │                 │
           │◄────────────────┘
           │ 3. return (cached or fresh)
           ▼
        ┌─────┐
        │ App │
        └─────┘
```

### Реализация

```python
from abc import ABC, abstractmethod

class DataLoader(ABC):
    """Интерфейс загрузчика данных"""

    @abstractmethod
    def load(self, key: str) -> Optional[Any]:
        pass


class DatabaseLoader(DataLoader):
    """Загрузчик из базы данных"""

    def __init__(self, db):
        self.db = db

    def load(self, key: str) -> Optional[dict]:
        # Парсим ключ: "user:123" -> collection="users", id=123
        entity, entity_id = key.split(':')
        collection = entity + 's'  # user -> users
        return self.db[collection].find_one({'id': int(entity_id)})


class ReadThroughCache:
    """
    Read-Through стратегия.

    Плюсы:
    - Приложение упрощено — работает только с кэшем
    - Логика загрузки инкапсулирована в кэше

    Минусы:
    - Кэш должен знать о структуре данных
    - Сложнее кастомизировать для разных типов данных
    """

    def __init__(self, cache: redis.Redis, loader: DataLoader, default_ttl: int = 3600):
        self.cache = cache
        self.loader = loader
        self.default_ttl = default_ttl

    def get(self, key: str) -> Optional[Any]:
        """Получить данные (кэш загрузит автоматически при промахе)"""

        # Проверяем кэш
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        # Загружаем через loader
        data = self.loader.load(key)

        # Сохраняем в кэш
        if data is not None:
            self.cache.setex(key, self.default_ttl, json.dumps(data))

        return data


# Использование
loader = DatabaseLoader(db)
cache = ReadThroughCache(redis.Redis(), loader)

# Приложение работает только с кэшем
user = cache.get('user:123')
product = cache.get('product:456')
```

## Write-Through

### Принцип работы

Данные записываются в кэш и БД одновременно (синхронно). Гарантирует консистентность.

```
        ┌─────┐
        │ App │
        └──┬──┘
           │ 1. write
           ▼
        ┌─────┐
        │Cache│
        └──┬──┘
           │
           ├──────────────┬──────────────┐
           │              │              │
           ▼              ▼              │
    2. write cache   3. write DB         │
           │              │              │
           └──────────────┴──────────────┘
                          │
                          ▼
                   4. return (после записи в оба)
```

### Реализация

```python
class WriteThroughCache:
    """
    Write-Through стратегия.

    Плюсы:
    - Кэш всегда консистентен с БД
    - Данные не теряются при сбое кэша

    Минусы:
    - Высокая latency записи (ждём и кэш, и БД)
    - Записываются даже редко читаемые данные
    """

    def __init__(self, cache: redis.Redis, db, default_ttl: int = 3600):
        self.cache = cache
        self.db = db
        self.default_ttl = default_ttl

    def get(self, collection: str, doc_id: str) -> Optional[dict]:
        """Чтение — стандартный read-through"""
        key = f'{collection}:{doc_id}'

        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        data = self.db[collection].find_one({'_id': doc_id})
        if data:
            self.cache.setex(key, self.default_ttl, json.dumps(data))

        return data

    def set(self, collection: str, doc_id: str, data: dict) -> bool:
        """Запись — синхронно в кэш и БД"""
        key = f'{collection}:{doc_id}'

        try:
            # Начинаем транзакцию (если БД поддерживает)
            # 1. Записываем в БД
            self.db[collection].update_one(
                {'_id': doc_id},
                {'$set': data},
                upsert=True
            )

            # 2. Записываем в кэш
            self.cache.setex(key, self.default_ttl, json.dumps(data))

            return True

        except Exception as e:
            # При ошибке инвалидируем кэш для консистентности
            self.cache.delete(key)
            raise

    def delete(self, collection: str, doc_id: str) -> bool:
        """Удаление — из обоих хранилищ"""
        key = f'{collection}:{doc_id}'

        self.db[collection].delete_one({'_id': doc_id})
        self.cache.delete(key)

        return True


# Использование
cache = WriteThroughCache(redis.Redis(), db)

# Запись — автоматически в кэш и БД
cache.set('users', '123', {'name': 'John', 'email': 'john@example.com'})

# Чтение — сначала из кэша
user = cache.get('users', '123')
```

## Write-Behind (Write-Back)

### Принцип работы

Данные сначала записываются в кэш, а в БД — асинхронно, с задержкой.

```
        ┌─────┐
        │ App │
        └──┬──┘
           │ 1. write
           ▼
        ┌─────┐
        │Cache│◄─── return немедленно
        └──┬──┘
           │
           │ 2. async (batch, delay)
           ▼
        ┌─────┐
        │Queue│
        └──┬──┘
           │
           │ 3. background flush
           ▼
        ┌─────┐
        │ DB  │
        └─────┘
```

### Реализация

```python
import threading
import queue
from collections import defaultdict

class WriteBehindCache:
    """
    Write-Behind (Write-Back) стратегия.

    Плюсы:
    - Очень быстрая запись (только в кэш)
    - Batch writes снижают нагрузку на БД
    - Сглаживание пиков нагрузки

    Минусы:
    - Риск потери данных при сбое кэша
    - Временная рассинхронизация
    - Сложная реализация
    """

    def __init__(self, cache: redis.Redis, db,
                 flush_interval: int = 5,
                 batch_size: int = 100):
        self.cache = cache
        self.db = db
        self.flush_interval = flush_interval
        self.batch_size = batch_size

        # Очередь записей для flush
        self._write_queue = queue.Queue()
        self._pending_writes = defaultdict(dict)
        self._lock = threading.Lock()

        # Запускаем фоновый writer
        self._start_background_writer()

    def _start_background_writer(self):
        """Фоновый поток для записи в БД"""
        def writer():
            while True:
                time.sleep(self.flush_interval)
                self._flush_to_db()

        thread = threading.Thread(target=writer, daemon=True)
        thread.start()

    def set(self, collection: str, doc_id: str, data: dict, ttl: int = 3600):
        """Быстрая запись — только в кэш"""
        key = f'{collection}:{doc_id}'

        # Записываем в кэш
        self.cache.setex(key, ttl, json.dumps(data))

        # Добавляем в очередь на запись в БД
        with self._lock:
            self._pending_writes[collection][doc_id] = data

        # Если накопилось много — flush немедленно
        total_pending = sum(len(v) for v in self._pending_writes.values())
        if total_pending >= self.batch_size:
            self._flush_to_db()

    def _flush_to_db(self):
        """Записать накопленные данные в БД"""
        with self._lock:
            if not self._pending_writes:
                return

            writes = dict(self._pending_writes)
            self._pending_writes = defaultdict(dict)

        # Batch write в БД
        for collection, documents in writes.items():
            operations = [
                UpdateOne(
                    {'_id': doc_id},
                    {'$set': data},
                    upsert=True
                )
                for doc_id, data in documents.items()
            ]

            if operations:
                try:
                    self.db[collection].bulk_write(operations, ordered=False)
                except Exception as e:
                    # При ошибке возвращаем в очередь
                    with self._lock:
                        for doc_id, data in documents.items():
                            self._pending_writes[collection][doc_id] = data
                    raise

    def get(self, collection: str, doc_id: str) -> Optional[dict]:
        """Чтение — из кэша или pending writes"""
        key = f'{collection}:{doc_id}'

        # Сначала проверяем pending writes
        with self._lock:
            if doc_id in self._pending_writes.get(collection, {}):
                return self._pending_writes[collection][doc_id]

        # Затем кэш
        cached = self.cache.get(key)
        if cached:
            return json.loads(cached)

        # Наконец БД
        data = self.db[collection].find_one({'_id': doc_id})
        if data:
            self.cache.setex(key, 3600, json.dumps(data))

        return data

    def flush_sync(self):
        """Принудительный синхронный flush (для graceful shutdown)"""
        self._flush_to_db()
```

### Надёжная реализация с Redis Streams

```python
class ReliableWriteBehind:
    """Write-Behind с использованием Redis Streams для надёжности"""

    def __init__(self, cache: redis.Redis, db):
        self.cache = cache
        self.db = db
        self.stream_key = 'write_behind:stream'
        self.consumer_group = 'db_writers'

    def set(self, collection: str, doc_id: str, data: dict, ttl: int = 3600):
        """Запись с гарантией доставки"""
        key = f'{collection}:{doc_id}'

        # Записываем в кэш
        self.cache.setex(key, ttl, json.dumps(data))

        # Добавляем в stream (персистентная очередь)
        self.cache.xadd(self.stream_key, {
            'collection': collection,
            'doc_id': doc_id,
            'data': json.dumps(data),
            'timestamp': str(time.time())
        })

    def process_writes(self):
        """Consumer для обработки записей"""
        while True:
            # Читаем batch из stream
            messages = self.cache.xreadgroup(
                self.consumer_group,
                'worker-1',
                {self.stream_key: '>'},
                count=100,
                block=5000
            )

            if not messages:
                continue

            for stream, entries in messages:
                operations = defaultdict(list)

                for msg_id, data in entries:
                    collection = data['collection']
                    doc_id = data['doc_id']
                    doc_data = json.loads(data['data'])

                    operations[collection].append((msg_id, doc_id, doc_data))

                # Batch write
                for collection, ops in operations.items():
                    bulk_ops = [
                        UpdateOne({'_id': doc_id}, {'$set': doc_data}, upsert=True)
                        for _, doc_id, doc_data in ops
                    ]

                    self.db[collection].bulk_write(bulk_ops)

                    # ACK успешно обработанные
                    msg_ids = [msg_id for msg_id, _, _ in ops]
                    self.cache.xack(self.stream_key, self.consumer_group, *msg_ids)
```

## Refresh-Ahead

### Принцип работы

Проактивное обновление кэша до истечения TTL для предотвращения cache miss.

```
        TTL = 3600 секунд
        ├────────────────────────────────────────────────────────┤
        │                                                        │
        │        80% TTL (2880 сек)                              │
        │              │                                         │
        │              ▼                                         │
        │         ┌────────┐                                     │
        │         │Refresh │ <── Фоновое обновление              │
        │         └────────┘                                     │
        │                                                        │
        └────────────────────────────────────────────────────────┘
```

### Реализация

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class RefreshAheadCache:
    """
    Refresh-Ahead стратегия.

    Плюсы:
    - Минимизирует cache miss
    - Пользователи всегда получают данные из кэша
    - Предсказуемая latency

    Минусы:
    - Потребляет ресурсы на refresh неиспользуемых данных
    - Сложность определения порога refresh
    """

    def __init__(self, cache: redis.Redis, default_ttl: int = 3600,
                 refresh_threshold: float = 0.8):
        self.cache = cache
        self.default_ttl = default_ttl
        self.refresh_threshold = refresh_threshold  # 80% TTL
        self.executor = ThreadPoolExecutor(max_workers=4)
        self._refreshing = set()

    def get(self, key: str, loader: Callable, ttl: int = None) -> Any:
        """Получить данные с проактивным refresh"""
        ttl = ttl or self.default_ttl

        # Получаем значение и TTL
        pipe = self.cache.pipeline()
        pipe.get(key)
        pipe.ttl(key)
        value, remaining_ttl = pipe.execute()

        if value:
            # Проверяем нужен ли refresh
            if remaining_ttl > 0:
                threshold = ttl * (1 - self.refresh_threshold)
                if remaining_ttl < threshold:
                    self._schedule_refresh(key, loader, ttl)

            return json.loads(value)

        # Cache miss — загружаем синхронно
        data = loader()
        if data is not None:
            self.cache.setex(key, ttl, json.dumps(data))

        return data

    def _schedule_refresh(self, key: str, loader: Callable, ttl: int):
        """Запланировать фоновое обновление"""
        if key in self._refreshing:
            return

        self._refreshing.add(key)

        def refresh():
            try:
                data = loader()
                if data is not None:
                    self.cache.setex(key, ttl, json.dumps(data))
            finally:
                self._refreshing.discard(key)

        self.executor.submit(refresh)


# Использование
cache = RefreshAheadCache(redis.Redis(), default_ttl=3600, refresh_threshold=0.8)

def get_product(product_id: int):
    return cache.get(
        key=f'product:{product_id}',
        loader=lambda: db.products.find_one({'id': product_id}),
        ttl=3600
    )
```

## Сравнение стратегий

```
┌────────────────┬───────────┬───────────┬───────────┬──────────────┐
│   Стратегия    │ Latency   │ Консистен-│ Сложность │ Когда исполь-│
│                │ чтения    │ тность    │           │ зовать       │
├────────────────┼───────────┼───────────┼───────────┼──────────────┤
│ Cache-Aside    │ Средняя   │ Eventual  │ Низкая    │ Универсально │
│                │           │           │           │              │
│ Read-Through   │ Средняя   │ Eventual  │ Средняя   │ Когда нужна  │
│                │           │           │           │ абстракция   │
│                │           │           │           │              │
│ Write-Through  │ Высокая   │ Strong    │ Средняя   │ Когда важна  │
│                │ (запись)  │           │           │ консистент.  │
│                │           │           │           │              │
│ Write-Behind   │ Низкая    │ Eventual  │ Высокая   │ Много записи,│
│                │ (запись)  │           │           │ можно терять │
│                │           │           │           │              │
│ Refresh-Ahead  │ Низкая    │ Eventual  │ Средняя   │ Горячие      │
│                │           │           │           │ данные       │
└────────────────┴───────────┴───────────┴───────────┴──────────────┘
```

## Комбинирование стратегий

```python
class HybridCache:
    """Комбинация стратегий для разных типов данных"""

    def __init__(self, cache: redis.Redis, db):
        self.cache_aside = CacheAside(cache)
        self.write_through = WriteThroughCache(cache, db)
        self.write_behind = WriteBehindCache(cache, db)

    def get_user(self, user_id: int) -> dict:
        """Пользователи: Cache-Aside (часто читаются, редко меняются)"""
        return self.cache_aside.get(
            f'user:{user_id}',
            lambda: db.users.find_one({'id': user_id})
        )

    def update_user(self, user_id: int, data: dict):
        """Профиль: Write-Through (важна консистентность)"""
        self.write_through.set('users', str(user_id), data)

    def log_activity(self, user_id: int, activity: dict):
        """Логи: Write-Behind (много записей, можно терять)"""
        self.write_behind.set('activities', f'{user_id}:{time.time()}', activity)
```

## Типичные ошибки

### 1. Неправильная инвалидация в Cache-Aside

```python
# НЕПРАВИЛЬНО: сначала кэш, потом БД
def update_user_wrong(user_id: int, data: dict):
    cache.delete(f'user:{user_id}')  # 1. Удаляем кэш
    db.users.update({'id': user_id}, data)  # 2. Обновляем БД
    # Между 1 и 2 другой запрос может закэшировать старые данные!

# ПРАВИЛЬНО: сначала БД, потом кэш
def update_user_correct(user_id: int, data: dict):
    db.users.update({'id': user_id}, data)  # 1. Обновляем БД
    cache.delete(f'user:{user_id}')  # 2. Удаляем кэш
```

### 2. Отсутствие обработки отказа кэша

```python
# НЕПРАВИЛЬНО: падаем при недоступности кэша
def get_data(key):
    return cache.get(key)  # Exception если Redis недоступен

# ПРАВИЛЬНО: graceful degradation
def get_data_safe(key, loader):
    try:
        cached = cache.get(key)
        if cached:
            return json.loads(cached)
    except redis.RedisError:
        pass  # Кэш недоступен, идём в БД

    return loader()
```

## Резюме

Выбор стратегии кэширования зависит от:

- **Соотношения чтение/запись** — много чтений → Cache-Aside, много записей → Write-Behind
- **Требований к консистентности** — строгая → Write-Through, eventual → остальные
- **Толерантности к потере данных** — можно терять → Write-Behind
- **Требований к latency** — критична → Refresh-Ahead

Часто оптимальное решение — комбинация стратегий для разных типов данных.
