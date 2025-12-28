# Семафоры

[prev: ./02-locks.md](./02-locks.md) | [next: ./04-events.md](./04-events.md)

---

## Что такое Semaphore?

**Семафор** (Semaphore) - это примитив синхронизации, который ограничивает количество корутин, одновременно имеющих доступ к ресурсу. В отличие от Lock, который позволяет только одной корутине владеть им, семафор может быть захвачен несколькими корутинами одновременно (до указанного лимита).

### Аналогия

Представьте парковку на 5 мест:
- **Lock** - парковка на 1 место (только одна машина)
- **Semaphore(5)** - парковка на 5 мест (до 5 машин одновременно)

## Базовое использование

### Создание семафора

```python
import asyncio

# Семафор на 3 одновременных доступа
semaphore = asyncio.Semaphore(3)

async def limited_task(name: str):
    async with semaphore:
        print(f"{name}: начал работу")
        await asyncio.sleep(1)
        print(f"{name}: завершил работу")


async def main():
    # Запускаем 10 задач, но одновременно работают только 3
    tasks = [limited_task(f"Task-{i}") for i in range(10)]
    await asyncio.gather(*tasks)


asyncio.run(main())
```

### Вывод (упрощённо)

```
Task-0: начал работу
Task-1: начал работу
Task-2: начал работу
# Пауза 1 секунда
Task-0: завершил работу
Task-3: начал работу
Task-1: завершил работу
Task-4: начал работу
...
```

## Типы семафоров в asyncio

### 1. Semaphore - обычный семафор

```python
import asyncio

semaphore = asyncio.Semaphore(2)

async def demo():
    # Можно освободить больше раз, чем захватили (опасно!)
    semaphore.release()  # Счётчик становится 3!
    print(f"Доступных слотов: {semaphore._value}")  # 3


asyncio.run(demo())
```

### 2. BoundedSemaphore - ограниченный семафор

```python
import asyncio

bounded_semaphore = asyncio.BoundedSemaphore(2)

async def demo():
    try:
        # Ошибка! Нельзя освободить больше, чем начальное значение
        bounded_semaphore.release()
    except ValueError as e:
        print(f"Ошибка: {e}")  # BoundedSemaphore released too many times


asyncio.run(demo())
```

**Рекомендация**: Всегда используйте `BoundedSemaphore`, чтобы отлавливать ошибки в логике программы.

## Практические примеры

### 1. Ограничение параллельных HTTP-запросов

```python
import asyncio
import aiohttp

class RateLimitedClient:
    def __init__(self, max_concurrent: int = 5):
        self._semaphore = asyncio.BoundedSemaphore(max_concurrent)
        self._session = None

    async def __aenter__(self):
        self._session = aiohttp.ClientSession()
        return self

    async def __aexit__(self, *args):
        await self._session.close()

    async def get(self, url: str) -> str:
        async with self._semaphore:
            print(f"Запрос: {url}")
            async with self._session.get(url) as response:
                return await response.text()


async def main():
    urls = [f"https://httpbin.org/delay/1?id={i}" for i in range(20)]

    async with RateLimitedClient(max_concurrent=5) as client:
        # 20 запросов, но одновременно не более 5
        tasks = [client.get(url) for url in urls]
        results = await asyncio.gather(*tasks)

    print(f"Получено {len(results)} ответов")


asyncio.run(main())
```

### 2. Пул подключений к базе данных

```python
import asyncio
from contextlib import asynccontextmanager

class ConnectionPool:
    def __init__(self, max_connections: int = 10):
        self._semaphore = asyncio.BoundedSemaphore(max_connections)
        self._connections = []
        self._max = max_connections

    async def _create_connection(self):
        """Симуляция создания подключения"""
        await asyncio.sleep(0.1)
        return {"id": len(self._connections), "connected": True}

    @asynccontextmanager
    async def acquire(self):
        async with self._semaphore:
            # Получаем или создаём подключение
            if self._connections:
                conn = self._connections.pop()
            else:
                conn = await self._create_connection()

            try:
                yield conn
            finally:
                # Возвращаем подключение в пул
                self._connections.append(conn)


async def worker(pool: ConnectionPool, worker_id: int):
    async with pool.acquire() as conn:
        print(f"Worker-{worker_id} использует подключение {conn['id']}")
        await asyncio.sleep(0.5)  # Работа с БД
        print(f"Worker-{worker_id} освободил подключение")


async def main():
    pool = ConnectionPool(max_connections=3)

    # 10 воркеров, но только 3 подключения
    await asyncio.gather(*[worker(pool, i) for i in range(10)])


asyncio.run(main())
```

### 3. Rate Limiting (ограничение частоты запросов)

```python
import asyncio
import time

class RateLimiter:
    """Ограничитель: не более N операций в секунду"""

    def __init__(self, rate: int):
        self._rate = rate
        self._semaphore = asyncio.BoundedSemaphore(rate)
        self._reset_task = None

    async def start(self):
        """Запуск периодического сброса семафора"""
        self._reset_task = asyncio.create_task(self._reset_loop())

    async def stop(self):
        if self._reset_task:
            self._reset_task.cancel()
            try:
                await self._reset_task
            except asyncio.CancelledError:
                pass

    async def _reset_loop(self):
        while True:
            await asyncio.sleep(1)
            # Сбрасываем семафор до начального значения
            while self._semaphore._value < self._rate:
                self._semaphore.release()

    async def acquire(self):
        await self._semaphore.acquire()

    def __aiter__(self):
        return self

    async def __anext__(self):
        await self.acquire()
        return time.time()


async def main():
    limiter = RateLimiter(rate=5)  # 5 операций в секунду
    await limiter.start()

    start = time.time()
    for i in range(15):
        await limiter.acquire()
        print(f"Операция {i} в {time.time() - start:.2f}с")

    await limiter.stop()


asyncio.run(main())
```

### 4. Контроль загрузки CPU/памяти

```python
import asyncio

class ResourceController:
    """Контроль использования ресурсов"""

    def __init__(self, max_heavy_tasks: int = 2, max_light_tasks: int = 10):
        self._heavy = asyncio.BoundedSemaphore(max_heavy_tasks)
        self._light = asyncio.BoundedSemaphore(max_light_tasks)

    async def run_heavy(self, coro):
        """Для ресурсоёмких задач (обработка изображений, ML)"""
        async with self._heavy:
            return await coro

    async def run_light(self, coro):
        """Для лёгких задач (API вызовы, чтение файлов)"""
        async with self._light:
            return await coro


async def heavy_task(task_id: int):
    print(f"Heavy task {task_id} started")
    await asyncio.sleep(2)  # Симуляция тяжёлой работы
    print(f"Heavy task {task_id} finished")
    return f"heavy_{task_id}"


async def light_task(task_id: int):
    print(f"Light task {task_id} started")
    await asyncio.sleep(0.5)
    print(f"Light task {task_id} finished")
    return f"light_{task_id}"


async def main():
    controller = ResourceController(max_heavy_tasks=2, max_light_tasks=5)

    tasks = []
    # 10 тяжёлых задач (по 2 одновременно)
    for i in range(10):
        tasks.append(controller.run_heavy(heavy_task(i)))

    # 20 лёгких задач (по 5 одновременно)
    for i in range(20):
        tasks.append(controller.run_light(light_task(i)))

    results = await asyncio.gather(*tasks)
    print(f"Завершено {len(results)} задач")


asyncio.run(main())
```

## Методы Semaphore

### acquire() и release()

```python
import asyncio

semaphore = asyncio.Semaphore(2)

async def demo():
    print(f"Начальное значение: {semaphore._value}")  # 2

    await semaphore.acquire()
    print(f"После acquire: {semaphore._value}")  # 1

    await semaphore.acquire()
    print(f"После второго acquire: {semaphore._value}")  # 0

    # Следующий acquire заблокируется до release
    # await semaphore.acquire()  # Заблокируется!

    semaphore.release()
    print(f"После release: {semaphore._value}")  # 1


asyncio.run(demo())
```

### locked() - проверка заблокированности

```python
import asyncio

semaphore = asyncio.Semaphore(2)

async def demo():
    print(f"Заблокирован: {semaphore.locked()}")  # False

    await semaphore.acquire()
    print(f"Заблокирован: {semaphore.locked()}")  # False (ещё есть 1 слот)

    await semaphore.acquire()
    print(f"Заблокирован: {semaphore.locked()}")  # True (все слоты заняты)


asyncio.run(demo())
```

## Продвинутые паттерны

### Динамический семафор

```python
import asyncio

class DynamicSemaphore:
    """Семафор с изменяемым лимитом"""

    def __init__(self, initial_value: int):
        self._value = initial_value
        self._max_value = initial_value
        self._waiters = []
        self._lock = asyncio.Lock()

    async def acquire(self):
        async with self._lock:
            while self._value <= 0:
                waiter = asyncio.get_event_loop().create_future()
                self._waiters.append(waiter)
                self._lock.release()
                try:
                    await waiter
                finally:
                    await self._lock.acquire()
            self._value -= 1

    def release(self):
        self._value += 1
        if self._value > self._max_value:
            self._max_value = self._value
        if self._waiters:
            waiter = self._waiters.pop(0)
            if not waiter.done():
                waiter.set_result(None)

    def increase_limit(self, amount: int = 1):
        """Увеличить лимит семафора"""
        for _ in range(amount):
            self.release()

    def decrease_limit(self, amount: int = 1):
        """Уменьшить лимит (не ниже текущего использования)"""
        self._max_value = max(1, self._max_value - amount)

    async def __aenter__(self):
        await self.acquire()
        return self

    async def __aexit__(self, *args):
        self.release()
```

### Семафор с таймаутом

```python
import asyncio

semaphore = asyncio.Semaphore(1)

async def acquire_with_timeout(timeout: float) -> bool:
    try:
        await asyncio.wait_for(semaphore.acquire(), timeout=timeout)
        return True
    except asyncio.TimeoutError:
        return False


async def demo():
    async with semaphore:
        print("Семафор захвачен")

        # Попытка захватить занятый семафор с таймаутом
        success = await acquire_with_timeout(1.0)
        print(f"Захват с таймаутом: {success}")  # False


asyncio.run(demo())
```

### Приоритетный семафор

```python
import asyncio
import heapq
from dataclasses import dataclass, field
from typing import Any

@dataclass(order=True)
class PrioritizedItem:
    priority: int
    future: asyncio.Future = field(compare=False)

class PrioritySemaphore:
    """Семафор с приоритетами"""

    def __init__(self, value: int):
        self._value = value
        self._heap = []
        self._lock = asyncio.Lock()

    async def acquire(self, priority: int = 0):
        async with self._lock:
            if self._value > 0:
                self._value -= 1
                return

            future = asyncio.get_event_loop().create_future()
            heapq.heappush(self._heap, PrioritizedItem(priority, future))

        await future

    def release(self):
        if self._heap:
            item = heapq.heappop(self._heap)
            if not item.future.done():
                item.future.set_result(None)
        else:
            self._value += 1


async def worker(sem: PrioritySemaphore, name: str, priority: int):
    print(f"{name} (приоритет {priority}) ожидает...")
    await sem.acquire(priority)
    print(f"{name} работает!")
    await asyncio.sleep(0.5)
    sem.release()
    print(f"{name} завершил")


async def main():
    sem = PrioritySemaphore(1)

    # Захватываем семафор
    await sem.acquire()

    # Запускаем задачи с разными приоритетами
    tasks = [
        asyncio.create_task(worker(sem, "Low", 10)),
        asyncio.create_task(worker(sem, "High", 1)),
        asyncio.create_task(worker(sem, "Medium", 5)),
    ]

    await asyncio.sleep(0.1)  # Даём задачам встать в очередь
    sem.release()  # Освобождаем - High должен получить первым

    await asyncio.gather(*tasks)


asyncio.run(main())
```

## Распространённые ошибки

### 1. Забытый release()

```python
import asyncio

semaphore = asyncio.Semaphore(2)

async def bad_function():
    await semaphore.acquire()
    # Если здесь исключение - release не вызовется!
    raise ValueError("Ошибка")
    semaphore.release()  # Недостижимый код


# Правильно: используйте async with
async def good_function():
    async with semaphore:
        raise ValueError("Ошибка")
        # release вызовется автоматически
```

### 2. Неправильное начальное значение

```python
import asyncio

# ПЛОХО: слишком большое значение
huge_semaphore = asyncio.Semaphore(10000)  # 10000 одновременных подключений?

# ПЛОХО: нулевое значение (никто не сможет захватить)
zero_semaphore = asyncio.Semaphore(0)

# ХОРОШО: разумное значение на основе ресурсов
reasonable_semaphore = asyncio.Semaphore(10)  # 10 подключений к БД
```

### 3. Использование Semaphore вместо Lock

```python
import asyncio

# ПЛОХО: если нужен единственный доступ, используйте Lock
single_semaphore = asyncio.Semaphore(1)

# ХОРОШО: явный Lock
lock = asyncio.Lock()
```

### 4. Race condition с проверкой locked()

```python
import asyncio

semaphore = asyncio.Semaphore(1)

# ПЛОХО: race condition!
async def bad_check():
    if not semaphore.locked():  # Может измениться между проверкой
        await semaphore.acquire()  # и захватом!

# ХОРОШО: просто захватываем
async def good_approach():
    async with semaphore:
        # Гарантированно держим семафор
        pass
```

## Сравнение Lock и Semaphore

| Характеристика | Lock | Semaphore |
|----------------|------|-----------|
| Одновременный доступ | 1 | N |
| Реентерабельность | Нет | Нет |
| Владение | Нет | Нет |
| Использование | Взаимоисключение | Ограничение параллелизма |
| Пример | Защита переменной | Пул подключений |

## Best Practices

1. **Используйте BoundedSemaphore** - он ловит ошибки с лишним release()
2. **Используйте `async with`** - автоматический release
3. **Выбирайте разумный лимит** - на основе реальных ограничений ресурсов
4. **Не используйте Semaphore(1)** - для этого есть Lock
5. **Документируйте** что именно ограничивает семафор
6. **Мониторьте использование** - логируйте занятость семафора

## Когда использовать Semaphore

| Сценарий | Подходит? |
|----------|-----------|
| Ограничение параллельных запросов | Да |
| Пул ресурсов (подключения, файлы) | Да |
| Rate limiting | Да (с модификациями) |
| Взаимоисключение (mutex) | Нет (используйте Lock) |
| Ожидание события | Нет (используйте Event) |
| Producer-consumer | Нет (используйте Queue) |

## Резюме

- `Semaphore(N)` позволяет N корутинам одновременно владеть им
- `BoundedSemaphore` предпочтительнее - ловит ошибки
- Основное применение - ограничение параллелизма (подключения, запросы)
- Всегда используйте `async with` для автоматического освобождения
- Для единственного доступа используйте `Lock`, не `Semaphore(1)`
- Семафор не отслеживает, кто его захватил - нет понятия владельца

---

[prev: ./02-locks.md](./02-locks.md) | [next: ./04-events.md](./04-events.md)
