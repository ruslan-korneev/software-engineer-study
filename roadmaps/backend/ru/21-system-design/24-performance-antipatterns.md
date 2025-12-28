# Антипаттерны производительности

[prev: 23-event-driven-architecture](./23-event-driven-architecture.md) | [next: 25-monitoring](./25-monitoring.md)

---

## Введение

Антипаттерны производительности — это распространённые ошибки в проектировании и реализации систем, которые приводят к снижению производительности, увеличению времени отклика и неэффективному использованию ресурсов. Понимание этих антипаттернов критически важно для создания масштабируемых и отзывчивых приложений.

В отличие от функциональных багов, проблемы производительности часто проявляются не сразу, а при увеличении нагрузки или объёма данных. Раннее распознавание антипаттернов помогает избежать дорогостоящего рефакторинга в будущем.

---

## 1. N+1 Queries Problem

### Описание проблемы

N+1 запросов — один из самых распространённых антипаттернов при работе с базами данных. Он возникает, когда для получения связанных данных выполняется один запрос для получения списка (1) и затем N дополнительных запросов для каждого элемента.

### Пример проблемы

```python
# Антипаттерн: N+1 запросов
class User:
    def __init__(self, id, name):
        self.id = id
        self.name = name

class Order:
    def __init__(self, id, user_id, total):
        self.id = id
        self.user_id = user_id
        self.total = total

# Плохо: 1 запрос для пользователей + N запросов для заказов
def get_users_with_orders_bad():
    users = db.execute("SELECT * FROM users")  # 1 запрос
    result = []
    for user in users:
        # N запросов - по одному для каждого пользователя!
        orders = db.execute(
            "SELECT * FROM orders WHERE user_id = ?",
            user.id
        )
        result.append({
            'user': user,
            'orders': orders
        })
    return result

# При 1000 пользователях = 1001 запрос к БД!
```

### Решение

```python
# Правильно: Eager Loading (жадная загрузка)
def get_users_with_orders_good():
    # Один запрос с JOIN
    query = """
        SELECT u.id, u.name, o.id as order_id, o.total
        FROM users u
        LEFT JOIN orders o ON u.id = o.user_id
    """
    rows = db.execute(query)

    # Группируем результаты
    users_dict = {}
    for row in rows:
        if row['id'] not in users_dict:
            users_dict[row['id']] = {
                'user': {'id': row['id'], 'name': row['name']},
                'orders': []
            }
        if row['order_id']:
            users_dict[row['id']]['orders'].append({
                'id': row['order_id'],
                'total': row['total']
            })

    return list(users_dict.values())

# Альтернатива: два запроса с IN clause
def get_users_with_orders_batch():
    users = db.execute("SELECT * FROM users")  # 1 запрос
    user_ids = [u.id for u in users]

    # 1 запрос для всех заказов
    orders = db.execute(
        "SELECT * FROM orders WHERE user_id IN (?)",
        user_ids
    )

    # Группировка в памяти
    orders_by_user = {}
    for order in orders:
        orders_by_user.setdefault(order.user_id, []).append(order)

    return [
        {'user': u, 'orders': orders_by_user.get(u.id, [])}
        for u in users
    ]
# Всего 2 запроса независимо от количества пользователей
```

### Решение с ORM (SQLAlchemy)

```python
from sqlalchemy import create_engine, Column, Integer, String, ForeignKey
from sqlalchemy.orm import relationship, joinedload, selectinload, Session
from sqlalchemy.ext.declarative import declarative_base

Base = declarative_base()

class User(Base):
    __tablename__ = 'users'
    id = Column(Integer, primary_key=True)
    name = Column(String)
    orders = relationship("Order", back_populates="user")

class Order(Base):
    __tablename__ = 'orders'
    id = Column(Integer, primary_key=True)
    user_id = Column(Integer, ForeignKey('users.id'))
    total = Column(Integer)
    user = relationship("User", back_populates="orders")

# Антипаттерн: lazy loading (по умолчанию)
def get_users_lazy(session: Session):
    users = session.query(User).all()
    for user in users:
        # Каждое обращение к orders = новый запрос!
        print(f"{user.name}: {len(user.orders)} orders")

# Правильно: eager loading с joinedload
def get_users_eager_join(session: Session):
    users = session.query(User).options(
        joinedload(User.orders)  # LEFT JOIN в одном запросе
    ).all()
    for user in users:
        print(f"{user.name}: {len(user.orders)} orders")

# Правильно: eager loading с selectinload
def get_users_eager_select(session: Session):
    users = session.query(User).options(
        selectinload(User.orders)  # SELECT ... WHERE id IN (...)
    ).all()
    for user in users:
        print(f"{user.name}: {len(user.orders)} orders")
```

---

## 2. Chatty I/O (Болтливый ввод-вывод)

### Описание проблемы

Chatty I/O — это антипаттерн, при котором приложение делает множество мелких сетевых запросов или операций ввода-вывода вместо нескольких крупных. Каждый вызов имеет накладные расходы (latency, установка соединения, сериализация), которые накапливаются.

### Пример проблемы

```python
import redis
import requests

# Антипаттерн: множество мелких запросов к Redis
def get_user_data_chatty(redis_client, user_id: str):
    # 5 отдельных запросов к Redis
    name = redis_client.get(f"user:{user_id}:name")
    email = redis_client.get(f"user:{user_id}:email")
    age = redis_client.get(f"user:{user_id}:age")
    city = redis_client.get(f"user:{user_id}:city")
    status = redis_client.get(f"user:{user_id}:status")

    return {
        'name': name,
        'email': email,
        'age': age,
        'city': city,
        'status': status
    }

# Антипаттерн: множество HTTP запросов
def fetch_items_chatty(item_ids: list):
    items = []
    for item_id in item_ids:
        # Каждый запрос = TCP handshake + HTTP overhead
        response = requests.get(f"http://api.example.com/items/{item_id}")
        items.append(response.json())
    return items
```

### Решение

```python
import redis
import requests
from concurrent.futures import ThreadPoolExecutor

# Правильно: пакетные операции Redis
def get_user_data_batch(redis_client, user_id: str):
    # Один запрос с pipeline
    pipe = redis_client.pipeline()
    pipe.get(f"user:{user_id}:name")
    pipe.get(f"user:{user_id}:email")
    pipe.get(f"user:{user_id}:age")
    pipe.get(f"user:{user_id}:city")
    pipe.get(f"user:{user_id}:status")

    results = pipe.execute()  # Один round-trip к серверу

    return {
        'name': results[0],
        'email': results[1],
        'age': results[2],
        'city': results[3],
        'status': results[4]
    }

# Ещё лучше: использовать hash
def get_user_data_hash(redis_client, user_id: str):
    # Все данные пользователя в одном hash
    return redis_client.hgetall(f"user:{user_id}")

# Правильно: batch API запрос
def fetch_items_batch(item_ids: list):
    # Один запрос для всех items
    response = requests.post(
        "http://api.example.com/items/batch",
        json={"ids": item_ids}
    )
    return response.json()

# Альтернатива: параллельные запросы если batch недоступен
def fetch_items_parallel(item_ids: list, max_workers: int = 10):
    def fetch_one(item_id):
        response = requests.get(f"http://api.example.com/items/{item_id}")
        return response.json()

    with ThreadPoolExecutor(max_workers=max_workers) as executor:
        items = list(executor.map(fetch_one, item_ids))
    return items
```

### Агрегация запросов на уровне API

```python
from fastapi import FastAPI
from pydantic import BaseModel
from typing import List

app = FastAPI()

class BatchRequest(BaseModel):
    operations: List[dict]

class BatchResponse(BaseModel):
    results: List[dict]

# Предоставляем batch endpoint для клиентов
@app.post("/api/batch")
async def batch_operations(request: BatchRequest) -> BatchResponse:
    """
    Позволяет клиентам отправить несколько операций в одном запросе
    """
    results = []
    for op in request.operations:
        if op['type'] == 'get_user':
            result = await get_user(op['user_id'])
        elif op['type'] == 'get_orders':
            result = await get_orders(op['user_id'])
        elif op['type'] == 'get_products':
            result = await get_products(op['product_ids'])
        results.append(result)

    return BatchResponse(results=results)
```

---

## 3. Synchronous I/O (Синхронный ввод-вывод)

### Описание проблемы

Синхронный I/O блокирует поток выполнения во время ожидания ответа от внешних ресурсов (база данных, HTTP запросы, файловая система). Это приводит к неэффективному использованию ресурсов сервера и снижению пропускной способности.

### Пример проблемы

```python
import requests
import time

# Антипаттерн: синхронные последовательные запросы
def fetch_all_data_sync():
    start = time.time()

    # Каждый запрос блокирует поток
    users = requests.get("http://api.example.com/users").json()
    orders = requests.get("http://api.example.com/orders").json()
    products = requests.get("http://api.example.com/products").json()
    reviews = requests.get("http://api.example.com/reviews").json()

    print(f"Total time: {time.time() - start:.2f}s")
    # Если каждый запрос занимает 1 секунду, общее время = 4 секунды

    return {'users': users, 'orders': orders, 'products': products, 'reviews': reviews}
```

### Решение с asyncio

```python
import asyncio
import aiohttp
import time

# Правильно: асинхронные параллельные запросы
async def fetch_all_data_async():
    start = time.time()

    async with aiohttp.ClientSession() as session:
        # Все запросы выполняются параллельно
        tasks = [
            fetch_json(session, "http://api.example.com/users"),
            fetch_json(session, "http://api.example.com/orders"),
            fetch_json(session, "http://api.example.com/products"),
            fetch_json(session, "http://api.example.com/reviews"),
        ]

        users, orders, products, reviews = await asyncio.gather(*tasks)

    print(f"Total time: {time.time() - start:.2f}s")
    # Общее время ≈ время самого долгого запроса (≈1 секунда)

    return {'users': users, 'orders': orders, 'products': products, 'reviews': reviews}

async def fetch_json(session: aiohttp.ClientSession, url: str):
    async with session.get(url) as response:
        return await response.json()

# Запуск
asyncio.run(fetch_all_data_async())
```

### Асинхронная работа с базой данных

```python
import asyncio
import asyncpg
from typing import List, Dict, Any

class AsyncDatabasePool:
    def __init__(self, dsn: str, min_size: int = 5, max_size: int = 20):
        self.dsn = dsn
        self.min_size = min_size
        self.max_size = max_size
        self.pool = None

    async def connect(self):
        self.pool = await asyncpg.create_pool(
            self.dsn,
            min_size=self.min_size,
            max_size=self.max_size
        )

    async def close(self):
        await self.pool.close()

    async def fetch_many(self, queries: List[str]) -> List[List[Dict[str, Any]]]:
        """Выполняет несколько запросов параллельно"""
        async with self.pool.acquire() as conn:
            tasks = [conn.fetch(query) for query in queries]
            return await asyncio.gather(*tasks)

# Использование
async def get_dashboard_data(db: AsyncDatabasePool, user_id: int):
    queries = [
        f"SELECT * FROM orders WHERE user_id = {user_id} ORDER BY created_at DESC LIMIT 10",
        f"SELECT * FROM notifications WHERE user_id = {user_id} AND read = false",
        f"SELECT SUM(total) as total FROM orders WHERE user_id = {user_id}",
    ]

    orders, notifications, stats = await db.fetch_many(queries)

    return {
        'recent_orders': orders,
        'notifications': notifications,
        'total_spent': stats[0]['total'] if stats else 0
    }
```

### Неблокирующий файловый I/O

```python
import asyncio
import aiofiles
from pathlib import Path

# Антипаттерн: блокирующее чтение файлов
def read_files_sync(file_paths: list):
    contents = []
    for path in file_paths:
        with open(path, 'r') as f:
            contents.append(f.read())
    return contents

# Правильно: асинхронное чтение файлов
async def read_files_async(file_paths: list):
    async def read_one(path: str):
        async with aiofiles.open(path, 'r') as f:
            return await f.read()

    tasks = [read_one(path) for path in file_paths]
    return await asyncio.gather(*tasks)

# Для CPU-bound операций используем ProcessPoolExecutor
from concurrent.futures import ProcessPoolExecutor

async def process_files_cpu_bound(file_paths: list):
    loop = asyncio.get_event_loop()

    with ProcessPoolExecutor() as executor:
        tasks = [
            loop.run_in_executor(executor, heavy_processing, path)
            for path in file_paths
        ]
        return await asyncio.gather(*tasks)

def heavy_processing(file_path: str) -> dict:
    """CPU-интенсивная обработка файла"""
    # ... сложные вычисления
    pass
```

---

## 4. No Caching (Отсутствие кэширования)

### Описание проблемы

Отсутствие кэширования приводит к повторным вычислениям или запросам к медленным источникам данных. Это увеличивает нагрузку на базу данных и внешние сервисы, а также время отклика приложения.

### Пример проблемы

```python
# Антипаттерн: каждый раз запрашиваем данные из БД
def get_product_no_cache(product_id: int):
    # Этот запрос выполняется каждый раз, даже если данные не менялись
    return db.execute(
        "SELECT * FROM products WHERE id = ?",
        product_id
    )

# Антипаттерн: повторные вычисления
def calculate_statistics_no_cache(user_id: int):
    # Тяжёлые вычисления при каждом вызове
    orders = db.execute("SELECT * FROM orders WHERE user_id = ?", user_id)

    total = sum(o.total for o in orders)
    avg = total / len(orders) if orders else 0
    max_order = max((o.total for o in orders), default=0)

    return {'total': total, 'average': avg, 'max': max_order}
```

### Многоуровневое кэширование

```python
import functools
import time
import redis
import json
from typing import Optional, Callable, Any

# Уровень 1: In-memory кэш (для hot data)
class InMemoryCache:
    def __init__(self, max_size: int = 1000, ttl: int = 60):
        self.cache = {}
        self.max_size = max_size
        self.ttl = ttl

    def get(self, key: str) -> Optional[Any]:
        if key in self.cache:
            value, timestamp = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return value
            del self.cache[key]
        return None

    def set(self, key: str, value: Any):
        if len(self.cache) >= self.max_size:
            # Удаляем самый старый элемент
            oldest_key = min(self.cache, key=lambda k: self.cache[k][1])
            del self.cache[oldest_key]
        self.cache[key] = (value, time.time())

# Уровень 2: Redis кэш (распределённый)
class RedisCache:
    def __init__(self, redis_client: redis.Redis, prefix: str = "cache"):
        self.redis = redis_client
        self.prefix = prefix

    def get(self, key: str) -> Optional[Any]:
        value = self.redis.get(f"{self.prefix}:{key}")
        return json.loads(value) if value else None

    def set(self, key: str, value: Any, ttl: int = 300):
        self.redis.setex(
            f"{self.prefix}:{key}",
            ttl,
            json.dumps(value)
        )

# Многоуровневый кэш
class MultiLevelCache:
    def __init__(self, l1_cache: InMemoryCache, l2_cache: RedisCache):
        self.l1 = l1_cache
        self.l2 = l2_cache

    def get(self, key: str) -> Optional[Any]:
        # Сначала проверяем L1 (быстрый)
        value = self.l1.get(key)
        if value is not None:
            return value

        # Затем L2 (медленнее, но распределённый)
        value = self.l2.get(key)
        if value is not None:
            # Заполняем L1 для следующих запросов
            self.l1.set(key, value)
            return value

        return None

    def set(self, key: str, value: Any, l1_ttl: int = 60, l2_ttl: int = 300):
        self.l1.set(key, value)
        self.l2.set(key, value, l2_ttl)

# Декоратор для кэширования
def cached(cache: MultiLevelCache, key_func: Callable = None, l1_ttl: int = 60, l2_ttl: int = 300):
    def decorator(func):
        @functools.wraps(func)
        def wrapper(*args, **kwargs):
            # Генерируем ключ кэша
            if key_func:
                cache_key = key_func(*args, **kwargs)
            else:
                cache_key = f"{func.__name__}:{args}:{kwargs}"

            # Пробуем получить из кэша
            result = cache.get(cache_key)
            if result is not None:
                return result

            # Вычисляем и сохраняем
            result = func(*args, **kwargs)
            cache.set(cache_key, result, l1_ttl, l2_ttl)
            return result

        return wrapper
    return decorator

# Использование
memory_cache = InMemoryCache(max_size=1000, ttl=60)
redis_cache = RedisCache(redis.Redis(), prefix="app")
multi_cache = MultiLevelCache(memory_cache, redis_cache)

@cached(multi_cache, key_func=lambda product_id: f"product:{product_id}")
def get_product(product_id: int):
    return db.execute("SELECT * FROM products WHERE id = ?", product_id)
```

### Cache-Aside Pattern

```python
class ProductService:
    def __init__(self, db, cache: MultiLevelCache):
        self.db = db
        self.cache = cache

    def get_product(self, product_id: int) -> dict:
        cache_key = f"product:{product_id}"

        # 1. Попытка чтения из кэша
        product = self.cache.get(cache_key)
        if product:
            return product

        # 2. Cache miss - читаем из БД
        product = self.db.execute(
            "SELECT * FROM products WHERE id = ?",
            product_id
        )

        # 3. Сохраняем в кэш
        if product:
            self.cache.set(cache_key, product)

        return product

    def update_product(self, product_id: int, data: dict):
        # Обновляем БД
        self.db.execute(
            "UPDATE products SET name = ?, price = ? WHERE id = ?",
            data['name'], data['price'], product_id
        )

        # Инвалидируем кэш
        cache_key = f"product:{product_id}"
        self.cache.delete(cache_key)

        # Опционально: прогреваем кэш новыми данными
        # self.cache.set(cache_key, self.get_product(product_id))
```

---

## 5. Improper Threading (Неправильное использование потоков)

### Описание проблемы

Неправильное использование потоков включает: создание слишком большого количества потоков, отсутствие пулов потоков, гонки данных (race conditions), взаимные блокировки (deadlocks), и неэффективную синхронизацию.

### Пример проблем

```python
import threading
import time

# Антипаттерн 1: Создание потока на каждый запрос
def handle_request_bad(request):
    # Создание потока дорогое - 1-2MB памяти на поток
    thread = threading.Thread(target=process_request, args=(request,))
    thread.start()
    thread.join()  # Блокирует до завершения - зачем тогда поток?

# Антипаттерн 2: Race condition
counter = 0

def increment_counter_bad():
    global counter
    for _ in range(100000):
        counter += 1  # Не атомарная операция!

# При запуске из нескольких потоков результат непредсказуем
threads = [threading.Thread(target=increment_counter_bad) for _ in range(10)]
for t in threads: t.start()
for t in threads: t.join()
print(counter)  # Ожидаем 1000000, получаем меньше

# Антипаттерн 3: Deadlock
lock_a = threading.Lock()
lock_b = threading.Lock()

def worker_1():
    with lock_a:
        time.sleep(0.1)  # Симуляция работы
        with lock_b:  # Ждёт lock_b, который держит worker_2
            pass

def worker_2():
    with lock_b:
        time.sleep(0.1)
        with lock_a:  # Ждёт lock_a, который держит worker_1
            pass

# Deadlock!
t1 = threading.Thread(target=worker_1)
t2 = threading.Thread(target=worker_2)
t1.start(); t2.start()
t1.join(); t2.join()  # Зависнет навсегда
```

### Правильное использование потоков

```python
import threading
from concurrent.futures import ThreadPoolExecutor, as_completed
from queue import Queue
import time

# Правильно 1: Использование пула потоков
class RequestProcessor:
    def __init__(self, max_workers: int = 10):
        self.executor = ThreadPoolExecutor(max_workers=max_workers)

    def process_batch(self, requests: list):
        futures = [
            self.executor.submit(self.process_one, req)
            for req in requests
        ]

        results = []
        for future in as_completed(futures):
            try:
                results.append(future.result())
            except Exception as e:
                results.append({'error': str(e)})

        return results

    def process_one(self, request):
        # Обработка запроса
        return {'status': 'processed', 'data': request}

    def shutdown(self):
        self.executor.shutdown(wait=True)

# Правильно 2: Атомарные операции и блокировки
class ThreadSafeCounter:
    def __init__(self):
        self._value = 0
        self._lock = threading.Lock()

    def increment(self):
        with self._lock:
            self._value += 1

    def get(self) -> int:
        with self._lock:
            return self._value

# Правильно 3: Избегаем deadlock через упорядочивание блокировок
class BankAccount:
    _lock_order = {}
    _order_counter = 0
    _order_lock = threading.Lock()

    def __init__(self, account_id: str, balance: float):
        self.account_id = account_id
        self.balance = balance
        self.lock = threading.Lock()

        # Присваиваем порядок блокировки при создании
        with BankAccount._order_lock:
            BankAccount._lock_order[id(self)] = BankAccount._order_counter
            BankAccount._order_counter += 1

    @staticmethod
    def transfer(from_account: 'BankAccount', to_account: 'BankAccount', amount: float):
        # Всегда блокируем в одном порядке (по id)
        first, second = sorted(
            [from_account, to_account],
            key=lambda acc: BankAccount._lock_order[id(acc)]
        )

        with first.lock:
            with second.lock:
                if from_account.balance >= amount:
                    from_account.balance -= amount
                    to_account.balance += amount
                    return True
                return False

# Правильно 4: Producer-Consumer с Queue
class WorkerPool:
    def __init__(self, num_workers: int = 4):
        self.queue = Queue()
        self.workers = []
        self.running = True

        for _ in range(num_workers):
            worker = threading.Thread(target=self._worker_loop)
            worker.daemon = True
            worker.start()
            self.workers.append(worker)

    def _worker_loop(self):
        while self.running:
            try:
                task = self.queue.get(timeout=1)
                if task is None:
                    break

                func, args, kwargs, result_queue = task
                try:
                    result = func(*args, **kwargs)
                    result_queue.put(('success', result))
                except Exception as e:
                    result_queue.put(('error', str(e)))
                finally:
                    self.queue.task_done()
            except:
                continue

    def submit(self, func, *args, **kwargs):
        result_queue = Queue()
        self.queue.put((func, args, kwargs, result_queue))
        return result_queue

    def shutdown(self):
        self.running = False
        for _ in self.workers:
            self.queue.put(None)
        for worker in self.workers:
            worker.join()
```

### Выбор между потоками и процессами

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import multiprocessing
import time

def io_bound_task(url: str) -> str:
    """I/O-bound: используем потоки"""
    import requests
    return requests.get(url).text[:100]

def cpu_bound_task(n: int) -> int:
    """CPU-bound: используем процессы"""
    total = 0
    for i in range(n):
        total += i ** 2
    return total

# Для I/O-bound задач - потоки
def process_urls(urls: list):
    with ThreadPoolExecutor(max_workers=20) as executor:
        results = list(executor.map(io_bound_task, urls))
    return results

# Для CPU-bound задач - процессы (обходим GIL)
def heavy_computations(numbers: list):
    with ProcessPoolExecutor(max_workers=multiprocessing.cpu_count()) as executor:
        results = list(executor.map(cpu_bound_task, numbers))
    return results
```

---

## 6. Memory Leaks (Утечки памяти)

### Описание проблемы

Утечки памяти в Python возникают когда объекты остаются в памяти, хотя они больше не нужны. Причины: циклические ссылки, незакрытые ресурсы, глобальные коллекции, которые только растут, неправильное использование кэшей.

### Пример проблем

```python
import gc

# Антипаттерн 1: Бесконечно растущий кэш
class BadCache:
    def __init__(self):
        self.cache = {}  # Никогда не очищается!

    def get_or_compute(self, key: str, compute_func):
        if key not in self.cache:
            self.cache[key] = compute_func()
        return self.cache[key]

# Антипаттерн 2: Циклические ссылки
class Node:
    def __init__(self, value):
        self.value = value
        self.parent = None
        self.children = []

    def add_child(self, child):
        child.parent = self  # Цикл: parent -> child -> parent
        self.children.append(child)

# Без weak references, удаление parent не освободит children

# Антипаттерн 3: Незакрытые ресурсы
def read_files_bad(file_paths: list):
    contents = []
    for path in file_paths:
        f = open(path, 'r')  # Файл никогда не закрывается!
        contents.append(f.read())
    return contents

# Антипаттерн 4: Глобальные списки без ограничений
_event_log = []

def log_event(event: dict):
    _event_log.append(event)  # Растёт бесконечно!
```

### Решения

```python
import weakref
import gc
from functools import lru_cache
from collections import OrderedDict
from contextlib import contextmanager
import tracemalloc
from typing import Optional, Any

# Правильно 1: LRU Cache с ограничением размера
class LRUCache:
    def __init__(self, max_size: int = 100):
        self.cache = OrderedDict()
        self.max_size = max_size

    def get(self, key: str) -> Optional[Any]:
        if key in self.cache:
            self.cache.move_to_end(key)
            return self.cache[key]
        return None

    def set(self, key: str, value: Any):
        if key in self.cache:
            self.cache.move_to_end(key)
        self.cache[key] = value

        while len(self.cache) > self.max_size:
            self.cache.popitem(last=False)  # Удаляем самый старый

# Встроенный декоратор с ограничением
@lru_cache(maxsize=1000)
def expensive_computation(n: int) -> int:
    return sum(i ** 2 for i in range(n))

# Правильно 2: Weak References для циклических структур
class Node:
    def __init__(self, value):
        self.value = value
        self._parent = None  # Weak reference
        self.children = []

    @property
    def parent(self):
        return self._parent() if self._parent else None

    @parent.setter
    def parent(self, node):
        self._parent = weakref.ref(node) if node else None

    def add_child(self, child):
        child.parent = self
        self.children.append(child)

# Правильно 3: Context Manager для ресурсов
def read_files_good(file_paths: list):
    contents = []
    for path in file_paths:
        with open(path, 'r') as f:  # Автоматическое закрытие
            contents.append(f.read())
    return contents

# Правильно 4: Ring Buffer для логов
from collections import deque

class RingBuffer:
    def __init__(self, max_size: int = 10000):
        self.buffer = deque(maxlen=max_size)

    def append(self, item):
        self.buffer.append(item)  # Старые элементы автоматически удаляются

    def get_all(self) -> list:
        return list(self.buffer)

event_log = RingBuffer(max_size=10000)

def log_event(event: dict):
    event_log.append(event)  # Теперь ограничено!

# Правильно 5: Периодическая очистка
class CacheWithTTL:
    def __init__(self, ttl: int = 300, cleanup_interval: int = 60):
        self.cache = {}
        self.ttl = ttl
        self.cleanup_interval = cleanup_interval
        self._last_cleanup = time.time()

    def get(self, key: str) -> Optional[Any]:
        self._maybe_cleanup()

        if key in self.cache:
            value, timestamp = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return value
            del self.cache[key]
        return None

    def set(self, key: str, value: Any):
        self.cache[key] = (value, time.time())

    def _maybe_cleanup(self):
        now = time.time()
        if now - self._last_cleanup > self.cleanup_interval:
            self._cleanup()
            self._last_cleanup = now

    def _cleanup(self):
        now = time.time()
        expired_keys = [
            key for key, (value, timestamp) in self.cache.items()
            if now - timestamp >= self.ttl
        ]
        for key in expired_keys:
            del self.cache[key]
```

### Инструменты диагностики

```python
import tracemalloc
import gc
import sys
from pympler import asizeof, tracker

# Отслеживание аллокаций памяти
def find_memory_leaks():
    tracemalloc.start()

    # ... выполнение подозрительного кода ...

    snapshot = tracemalloc.take_snapshot()
    top_stats = snapshot.statistics('lineno')

    print("Top 10 memory allocations:")
    for stat in top_stats[:10]:
        print(stat)

# Отслеживание объектов
def track_objects():
    gc.collect()

    # Подсчёт объектов по типам
    type_counts = {}
    for obj in gc.get_objects():
        obj_type = type(obj).__name__
        type_counts[obj_type] = type_counts.get(obj_type, 0) + 1

    # Топ-10 типов
    sorted_types = sorted(type_counts.items(), key=lambda x: x[1], reverse=True)
    for obj_type, count in sorted_types[:10]:
        print(f"{obj_type}: {count}")

# Поиск циклических ссылок
def find_cycles():
    gc.collect()
    gc.set_debug(gc.DEBUG_SAVEALL)
    gc.collect()

    print(f"Unreachable objects: {len(gc.garbage)}")
    for obj in gc.garbage[:5]:
        print(f"  {type(obj)}: {obj}")

# Использование pympler для детального анализа
def analyze_memory():
    tr = tracker.SummaryTracker()

    # ... код ...

    tr.print_diff()  # Показывает изменения в памяти
```

---

## Best Practices

### 1. Профилирование перед оптимизацией

```python
import cProfile
import pstats
from functools import wraps
import time

def profile(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        profiler = cProfile.Profile()
        result = profiler.runcall(func, *args, **kwargs)

        stats = pstats.Stats(profiler)
        stats.sort_stats('cumulative')
        stats.print_stats(20)

        return result
    return wrapper

# Простой таймер
def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} took {elapsed:.4f} seconds")
        return result
    return wrapper
```

### 2. Мониторинг в production

```python
from prometheus_client import Counter, Histogram, Gauge

# Метрики для отслеживания антипаттернов
db_query_count = Counter('db_queries_total', 'Total database queries', ['operation'])
db_query_duration = Histogram('db_query_duration_seconds', 'Query duration', ['operation'])
cache_hits = Counter('cache_hits_total', 'Cache hits')
cache_misses = Counter('cache_misses_total', 'Cache misses')
active_threads = Gauge('active_threads', 'Number of active threads')
memory_usage = Gauge('memory_usage_bytes', 'Current memory usage')
```

### 3. Чек-лист для код-ревью

```markdown
## Performance Review Checklist

### Database
- [ ] Нет N+1 запросов (используется eager loading)
- [ ] Запросы используют индексы
- [ ] Нет SELECT * (выбираем только нужные поля)
- [ ] Batch операции где возможно

### Caching
- [ ] Частые неизменяемые данные кэшируются
- [ ] Кэш имеет TTL и ограничение размера
- [ ] Есть стратегия инвалидации

### I/O
- [ ] Асинхронные операции для I/O-bound задач
- [ ] Connection pooling для БД и HTTP
- [ ] Таймауты на все внешние вызовы

### Threading
- [ ] Используются пулы потоков (не создаются ad-hoc)
- [ ] Нет race conditions (защита shared state)
- [ ] CPU-bound задачи в процессах, I/O-bound в потоках

### Memory
- [ ] Коллекции имеют ограничение размера
- [ ] Ресурсы закрываются (with statement)
- [ ] Нет очевидных циклических ссылок
```

---

## Заключение

Антипаттерны производительности могут незаметно накапливаться в кодовой базе и проявляться только под нагрузкой. Ключевые принципы:

1. **Измеряй, затем оптимизируй** — профилируй код перед оптимизацией
2. **Batch операции** — объединяй мелкие запросы в крупные
3. **Кэшируй разумно** — с TTL и ограничением размера
4. **Асинхронность для I/O** — не блокируй потоки на ожидании
5. **Пулы ресурсов** — для потоков, соединений, воркеров
6. **Управляй памятью** — weak references, context managers, ограниченные коллекции
7. **Мониторь в production** — метрики помогут выявить проблемы рано
