# Очереди с приоритетами

[prev: ./01-queue-basics.md](./01-queue-basics.md) | [next: ./03-lifo-queues.md](./03-lifo-queues.md)

---

## Введение

`asyncio.PriorityQueue` - это асинхронная очередь, которая извлекает элементы в порядке приоритета, а не в порядке добавления. Элементы с наименьшим значением приоритета извлекаются первыми (min-heap).

## Принцип работы

В отличие от обычной FIFO-очереди, `PriorityQueue` сортирует элементы при извлечении:

```python
import asyncio

async def basic_priority_example():
    pq = asyncio.PriorityQueue()

    # Добавляем элементы (приоритет, данные)
    await pq.put((3, "низкий приоритет"))
    await pq.put((1, "высокий приоритет"))
    await pq.put((2, "средний приоритет"))

    # Извлекаем - элементы выходят по приоритету!
    while not pq.empty():
        priority, item = await pq.get()
        print(f"Приоритет {priority}: {item}")
        pq.task_done()

asyncio.run(basic_priority_example())
# Вывод:
# Приоритет 1: высокий приоритет
# Приоритет 2: средний приоритет
# Приоритет 3: низкий приоритет
```

## Формат элементов

Элементы в `PriorityQueue` должны быть сравнимыми. Обычно используют кортежи:

```python
# (приоритет, данные)
await queue.put((1, "task"))

# (приоритет, порядок_добавления, данные) - для стабильной сортировки
await queue.put((1, 0, "task"))
```

**Важно:** Если приоритеты равны, Python попытается сравнить вторые элементы кортежа. Если они не сравнимы, возникнет ошибка.

## Решение проблемы сравнения

### Проблема: несравнимые данные

```python
import asyncio

async def problem_example():
    pq = asyncio.PriorityQueue()

    # Словари нельзя сравнивать между собой
    await pq.put((1, {"name": "task1"}))
    await pq.put((1, {"name": "task2"}))  # Ошибка при одинаковом приоритете!

    # TypeError: '<' not supported between instances of 'dict' and 'dict'

# asyncio.run(problem_example())  # Вызовет ошибку
```

### Решение 1: Добавить счетчик порядка

```python
import asyncio
from dataclasses import dataclass, field
from typing import Any

counter = 0

async def with_counter():
    global counter
    pq = asyncio.PriorityQueue()

    def add_task(priority: int, data: Any):
        global counter
        counter += 1
        return (priority, counter, data)

    await pq.put(add_task(1, {"name": "task1"}))
    await pq.put(add_task(1, {"name": "task2"}))
    await pq.put(add_task(1, {"name": "task3"}))

    while not pq.empty():
        priority, order, data = await pq.get()
        print(f"Приоритет: {priority}, Порядок: {order}, Данные: {data}")
        pq.task_done()

asyncio.run(with_counter())
```

### Решение 2: Использовать dataclass

```python
import asyncio
from dataclasses import dataclass, field
from typing import Any

@dataclass(order=True)
class PrioritizedItem:
    priority: int
    order: int = field(compare=True)
    data: Any = field(compare=False)  # Исключаем из сравнения

async def with_dataclass():
    pq = asyncio.PriorityQueue()
    order = 0

    def create_item(priority: int, data: Any) -> PrioritizedItem:
        nonlocal order
        order += 1
        return PrioritizedItem(priority, order, data)

    await pq.put(create_item(2, {"task": "low"}))
    await pq.put(create_item(1, {"task": "high"}))
    await pq.put(create_item(1, {"task": "high2"}))

    while not pq.empty():
        item = await pq.get()
        print(f"Приоритет: {item.priority}, Данные: {item.data}")
        pq.task_done()

asyncio.run(with_dataclass())
```

## Практический пример: Система задач с приоритетами

```python
import asyncio
from dataclasses import dataclass, field
from typing import Any, Callable
from enum import IntEnum
import random

class Priority(IntEnum):
    """Уровни приоритета (меньше = важнее)"""
    CRITICAL = 0
    HIGH = 1
    NORMAL = 2
    LOW = 3

@dataclass(order=True)
class Task:
    priority: Priority
    order: int = field(compare=True)
    name: str = field(compare=False)
    handler: Callable = field(compare=False, repr=False)

class TaskQueue:
    def __init__(self, maxsize: int = 0):
        self._queue = asyncio.PriorityQueue(maxsize=maxsize)
        self._order = 0

    async def add_task(self, priority: Priority, name: str, handler: Callable):
        self._order += 1
        task = Task(priority, self._order, name, handler)
        await self._queue.put(task)
        print(f"Добавлена задача: {name} (приоритет: {priority.name})")

    async def get_task(self) -> Task:
        return await self._queue.get()

    def task_done(self):
        self._queue.task_done()

    async def join(self):
        await self._queue.join()

    def empty(self) -> bool:
        return self._queue.empty()

async def worker(task_queue: TaskQueue, worker_id: int):
    """Воркер, обрабатывающий задачи по приоритету"""
    while True:
        try:
            task = await task_queue.get_task()
            print(f"Worker {worker_id}: обрабатывает '{task.name}' "
                  f"(приоритет: {task.priority.name})")

            # Выполняем обработчик задачи
            await task.handler()

            task_queue.task_done()
            print(f"Worker {worker_id}: завершил '{task.name}'")

        except asyncio.CancelledError:
            print(f"Worker {worker_id}: остановлен")
            raise

async def main():
    task_queue = TaskQueue()

    # Создаем обработчики задач
    async def process_payment():
        await asyncio.sleep(0.5)

    async def send_notification():
        await asyncio.sleep(0.2)

    async def generate_report():
        await asyncio.sleep(1.0)

    async def cleanup_logs():
        await asyncio.sleep(0.3)

    # Добавляем задачи с разными приоритетами
    await task_queue.add_task(Priority.LOW, "Очистка логов", cleanup_logs)
    await task_queue.add_task(Priority.CRITICAL, "Обработка платежа", process_payment)
    await task_queue.add_task(Priority.NORMAL, "Генерация отчета", generate_report)
    await task_queue.add_task(Priority.HIGH, "Отправка уведомления", send_notification)
    await task_queue.add_task(Priority.CRITICAL, "Срочный платеж", process_payment)

    # Запускаем воркеров
    workers = [
        asyncio.create_task(worker(task_queue, i))
        for i in range(2)
    ]

    # Ждем обработки всех задач
    await task_queue.join()

    # Останавливаем воркеров
    for w in workers:
        w.cancel()

    await asyncio.gather(*workers, return_exceptions=True)
    print("Все задачи обработаны!")

asyncio.run(main())
```

## Пример: Планировщик запросов к API

```python
import asyncio
from dataclasses import dataclass, field
from typing import Any
import time

@dataclass(order=True)
class APIRequest:
    priority: int
    timestamp: float = field(compare=True)
    endpoint: str = field(compare=False)
    params: dict = field(compare=False, default_factory=dict)

class RateLimitedAPIClient:
    """Клиент API с ограничением запросов и приоритетами"""

    def __init__(self, requests_per_second: float = 2.0):
        self._queue = asyncio.PriorityQueue()
        self._interval = 1.0 / requests_per_second
        self._last_request_time = 0.0
        self._order = 0

    async def request(self, endpoint: str, params: dict = None, priority: int = 5):
        """Добавить запрос в очередь (приоритет 1-10, где 1 - самый важный)"""
        self._order += 1
        request = APIRequest(
            priority=priority,
            timestamp=self._order,
            endpoint=endpoint,
            params=params or {}
        )
        await self._queue.put(request)
        print(f"Запрос в очередь: {endpoint} (приоритет: {priority})")

    async def _execute_request(self, request: APIRequest):
        """Имитация выполнения запроса"""
        print(f"Выполняется запрос: {request.endpoint}")
        await asyncio.sleep(0.1)  # Имитация сетевого запроса
        return {"status": "ok", "endpoint": request.endpoint}

    async def process_queue(self):
        """Обработка очереди с учетом rate limiting"""
        while True:
            request = await self._queue.get()

            # Rate limiting
            now = time.monotonic()
            time_since_last = now - self._last_request_time
            if time_since_last < self._interval:
                await asyncio.sleep(self._interval - time_since_last)

            result = await self._execute_request(request)
            self._last_request_time = time.monotonic()

            self._queue.task_done()
            print(f"Завершен запрос: {request.endpoint}, результат: {result}")

async def main():
    client = RateLimitedAPIClient(requests_per_second=2.0)

    # Запускаем обработчик очереди
    processor = asyncio.create_task(client.process_queue())

    # Добавляем запросы с разными приоритетами
    await client.request("/api/analytics", priority=10)  # Низкий приоритет
    await client.request("/api/user/profile", priority=5)
    await client.request("/api/payment/process", priority=1)  # Высокий приоритет
    await client.request("/api/notifications", priority=3)
    await client.request("/api/logs", priority=10)

    # Ждем обработки всех запросов
    await client._queue.join()

    processor.cancel()
    print("Все запросы обработаны!")

asyncio.run(main())
```

## Ограниченная очередь с приоритетами

```python
import asyncio

async def bounded_priority_queue_example():
    # Очередь максимум на 3 элемента
    pq = asyncio.PriorityQueue(maxsize=3)

    await pq.put((1, "task1"))
    await pq.put((2, "task2"))
    await pq.put((3, "task3"))

    print(f"Очередь заполнена: {pq.full()}")  # True

    # Это заблокируется, пока не освободится место
    async def add_urgent():
        print("Пытаюсь добавить срочную задачу...")
        await pq.put((0, "urgent"))  # Будет ждать!
        print("Срочная задача добавлена!")

    async def process_one():
        await asyncio.sleep(1)
        item = await pq.get()
        print(f"Обработан: {item}")
        pq.task_done()

    # Запускаем параллельно
    await asyncio.gather(add_urgent(), process_one())

    # Теперь срочная задача в очереди и будет извлечена первой!
    while not pq.empty():
        item = await pq.get()
        print(f"Извлечен: {item}")
        pq.task_done()

asyncio.run(bounded_priority_queue_example())
```

## Best Practices

### 1. Используйте enum для приоритетов

```python
from enum import IntEnum

class Priority(IntEnum):
    CRITICAL = 0
    HIGH = 1
    NORMAL = 2
    LOW = 3

# Теперь код более читаемый
await queue.put((Priority.HIGH, data))
```

### 2. Всегда добавляйте счетчик для стабильной сортировки

```python
# Плохо - может вызвать ошибку при одинаковых приоритетах
await queue.put((priority, data))

# Хорошо - гарантированный порядок
await queue.put((priority, counter, data))
```

### 3. Используйте dataclass для сложных задач

```python
@dataclass(order=True)
class Task:
    priority: int
    order: int = field(compare=True)
    name: str = field(compare=False)
    payload: dict = field(compare=False, default_factory=dict)
```

### 4. Инверсия приоритета для max-heap

По умолчанию `PriorityQueue` - это min-heap. Для max-heap используйте отрицательные значения:

```python
# Элемент с наибольшим "реальным" приоритетом извлекается первым
await queue.put((-real_priority, data))
```

## Распространенные ошибки

### 1. Забывают про сравнение данных

```python
# Ошибка при одинаковых приоритетах!
await pq.put((1, some_object))
await pq.put((1, another_object))  # TypeError
```

### 2. Неправильный порядок элементов в кортеже

```python
# Неправильно - данные сравниваются вместо приоритета
await pq.put(("task", 1))

# Правильно
await pq.put((1, "task"))
```

### 3. Изменение приоритета после добавления

```python
# Не делайте так! Приоритет фиксируется при добавлении
task = [1, "data"]  # Используем список
await pq.put(task)
task[0] = 0  # Изменение не повлияет на позицию в очереди!
```

## Сравнение типов очередей

| Тип очереди | Порядок извлечения | Использование |
|-------------|-------------------|---------------|
| `Queue` | FIFO (первый вошел - первый вышел) | Общее назначение |
| `PriorityQueue` | По приоритету (min-heap) | Задачи разной важности |
| `LifoQueue` | LIFO (последний вошел - первый вышел) | Обход в глубину, отмена операций |

---

[prev: ./01-queue-basics.md](./01-queue-basics.md) | [next: ./03-lifo-queues.md](./03-lifo-queues.md)
