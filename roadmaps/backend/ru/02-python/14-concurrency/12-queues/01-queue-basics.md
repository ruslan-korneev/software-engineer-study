# Основы асинхронных очередей

[prev: ./readme.md](./readme.md) | [next: ./02-priority-queues.md](./02-priority-queues.md)

---

## Введение

Асинхронные очереди (`asyncio.Queue`) - это мощный инструмент для организации обмена данными между асинхронными задачами (корутинами). Они реализуют паттерн "производитель-потребитель" (producer-consumer) и обеспечивают безопасную передачу данных между конкурентно выполняющимися корутинами.

## Зачем нужны асинхронные очереди?

В асинхронном программировании часто возникает необходимость:
- Передавать данные между независимыми корутинами
- Балансировать нагрузку между несколькими обработчиками
- Буферизовать данные, когда производитель работает быстрее потребителя
- Организовать конвейерную обработку данных

## Создание очереди

```python
import asyncio

# Очередь без ограничения размера
queue = asyncio.Queue()

# Очередь с максимальным размером
bounded_queue = asyncio.Queue(maxsize=10)
```

Параметр `maxsize` определяет максимальное количество элементов в очереди:
- `maxsize=0` (по умолчанию) - очередь без ограничений
- `maxsize > 0` - очередь блокирует производителя при заполнении

## Основные методы

### Добавление элементов

```python
import asyncio

async def producer_example():
    queue = asyncio.Queue()

    # Асинхронное добавление (ждет, если очередь полная)
    await queue.put("item1")

    # Неблокирующее добавление (вызывает исключение, если очередь полная)
    try:
        queue.put_nowait("item2")
    except asyncio.QueueFull:
        print("Очередь переполнена!")

asyncio.run(producer_example())
```

### Извлечение элементов

```python
import asyncio

async def consumer_example():
    queue = asyncio.Queue()
    await queue.put("item1")

    # Асинхронное извлечение (ждет, если очередь пустая)
    item = await queue.get()
    print(f"Получен элемент: {item}")

    # Неблокирующее извлечение (вызывает исключение, если очередь пустая)
    try:
        item = queue.get_nowait()
    except asyncio.QueueEmpty:
        print("Очередь пуста!")

asyncio.run(consumer_example())
```

### Проверка состояния очереди

```python
import asyncio

async def check_queue_state():
    queue = asyncio.Queue(maxsize=5)

    await queue.put("item1")
    await queue.put("item2")

    # Текущий размер очереди
    print(f"Размер очереди: {queue.qsize()}")  # 2

    # Проверка на пустоту
    print(f"Очередь пуста: {queue.empty()}")  # False

    # Проверка на заполненность (только для ограниченных очередей)
    print(f"Очередь полна: {queue.full()}")  # False

asyncio.run(check_queue_state())
```

## Паттерн "производитель-потребитель"

Классический пример использования асинхронных очередей:

```python
import asyncio
import random

async def producer(queue: asyncio.Queue, name: str, count: int):
    """Производитель - создает задачи и помещает их в очередь"""
    for i in range(count):
        item = f"{name}_task_{i}"
        await queue.put(item)
        print(f"[{name}] Добавлена задача: {item}")
        await asyncio.sleep(random.uniform(0.1, 0.5))
    print(f"[{name}] Завершил работу")

async def consumer(queue: asyncio.Queue, name: str):
    """Потребитель - обрабатывает задачи из очереди"""
    while True:
        item = await queue.get()
        print(f"[{name}] Обрабатывает: {item}")
        # Имитация обработки
        await asyncio.sleep(random.uniform(0.2, 0.8))
        # Сигнализируем о завершении обработки
        queue.task_done()
        print(f"[{name}] Завершил обработку: {item}")

async def main():
    queue = asyncio.Queue()

    # Запускаем производителей
    producers = [
        asyncio.create_task(producer(queue, "Producer1", 3)),
        asyncio.create_task(producer(queue, "Producer2", 3)),
    ]

    # Запускаем потребителей
    consumers = [
        asyncio.create_task(consumer(queue, "Consumer1")),
        asyncio.create_task(consumer(queue, "Consumer2")),
    ]

    # Ждем завершения всех производителей
    await asyncio.gather(*producers)

    # Ждем, пока все задачи в очереди будут обработаны
    await queue.join()

    # Отменяем потребителей (они работают бесконечно)
    for c in consumers:
        c.cancel()

asyncio.run(main())
```

## Методы task_done() и join()

Эти методы позволяют отслеживать завершение обработки всех элементов очереди:

```python
import asyncio

async def worker(queue: asyncio.Queue, worker_id: int):
    while True:
        task = await queue.get()
        print(f"Worker {worker_id}: обрабатывает {task}")
        await asyncio.sleep(0.5)  # Имитация работы
        queue.task_done()  # Важно! Сигнализируем о завершении
        print(f"Worker {worker_id}: завершил {task}")

async def main():
    queue = asyncio.Queue()

    # Добавляем задачи
    for i in range(5):
        await queue.put(f"task_{i}")

    # Создаем воркеров
    workers = [
        asyncio.create_task(worker(queue, i))
        for i in range(2)
    ]

    # Ждем обработки ВСЕХ задач
    await queue.join()
    print("Все задачи обработаны!")

    # Отменяем воркеров
    for w in workers:
        w.cancel()

asyncio.run(main())
```

**Важно:** Количество вызовов `task_done()` должно соответствовать количеству вызовов `get()`. Если вызвать `task_done()` больше раз, чем было `get()`, возникнет исключение `ValueError`.

## Ограниченные очереди (Bounded Queues)

Ограниченные очереди предотвращают переполнение памяти, когда производитель работает быстрее потребителя:

```python
import asyncio

async def fast_producer(queue: asyncio.Queue):
    for i in range(20):
        print(f"Производитель: пытается добавить item_{i}")
        await queue.put(f"item_{i}")  # Блокируется, если очередь полная
        print(f"Производитель: добавил item_{i}, размер очереди: {queue.qsize()}")

async def slow_consumer(queue: asyncio.Queue):
    while True:
        item = await queue.get()
        print(f"Потребитель: обрабатывает {item}")
        await asyncio.sleep(1)  # Медленная обработка
        queue.task_done()

async def main():
    # Очередь максимум на 3 элемента
    queue = asyncio.Queue(maxsize=3)

    producer_task = asyncio.create_task(fast_producer(queue))
    consumer_task = asyncio.create_task(slow_consumer(queue))

    await producer_task
    await queue.join()
    consumer_task.cancel()

asyncio.run(main())
```

## Конвейерная обработка (Pipeline)

Очереди отлично подходят для построения конвейеров обработки данных:

```python
import asyncio

async def stage1(input_queue: asyncio.Queue, output_queue: asyncio.Queue):
    """Первый этап: умножение на 2"""
    while True:
        data = await input_queue.get()
        result = data * 2
        print(f"Stage1: {data} -> {result}")
        await output_queue.put(result)
        input_queue.task_done()

async def stage2(input_queue: asyncio.Queue, output_queue: asyncio.Queue):
    """Второй этап: добавление 10"""
    while True:
        data = await input_queue.get()
        result = data + 10
        print(f"Stage2: {data} -> {result}")
        await output_queue.put(result)
        input_queue.task_done()

async def stage3(input_queue: asyncio.Queue):
    """Третий этап: финальный вывод"""
    while True:
        data = await input_queue.get()
        print(f"Stage3: Финальный результат = {data}")
        input_queue.task_done()

async def main():
    queue1 = asyncio.Queue()
    queue2 = asyncio.Queue()
    queue3 = asyncio.Queue()

    # Запускаем этапы конвейера
    workers = [
        asyncio.create_task(stage1(queue1, queue2)),
        asyncio.create_task(stage2(queue2, queue3)),
        asyncio.create_task(stage3(queue3)),
    ]

    # Отправляем данные в конвейер
    for i in range(1, 6):
        await queue1.put(i)

    # Ждем обработки на каждом этапе
    await queue1.join()
    await queue2.join()
    await queue3.join()

    for w in workers:
        w.cancel()

asyncio.run(main())
```

## Best Practices

### 1. Всегда используйте task_done() с join()

```python
# Правильно
item = await queue.get()
try:
    await process(item)
finally:
    queue.task_done()  # Гарантированный вызов

# Неправильно
item = await queue.get()
await process(item)  # Если здесь ошибка, task_done() не вызовется
queue.task_done()
```

### 2. Используйте ограниченные очереди для контроля памяти

```python
# Рекомендуется для продакшена
queue = asyncio.Queue(maxsize=1000)
```

### 3. Graceful shutdown для потребителей

```python
async def consumer(queue: asyncio.Queue):
    try:
        while True:
            item = await queue.get()
            await process(item)
            queue.task_done()
    except asyncio.CancelledError:
        print("Consumer: получен сигнал остановки")
        # Можно обработать оставшиеся элементы
        raise
```

## Распространенные ошибки

### 1. Забывают вызывать task_done()

```python
# Ошибка: join() будет ждать вечно
async def bad_consumer(queue):
    while True:
        item = await queue.get()
        await process(item)
        # Забыли task_done()!
```

### 2. Лишние вызовы task_done()

```python
# Ошибка: ValueError
queue.task_done()  # Без предшествующего get()
```

### 3. Использование синхронных очередей в asyncio

```python
# Неправильно! queue.Queue - синхронная очередь
from queue import Queue
sync_queue = Queue()  # Будет блокировать event loop!

# Правильно
async_queue = asyncio.Queue()
```

## Сравнение с синхронными очередями

| Аспект | asyncio.Queue | queue.Queue |
|--------|---------------|-------------|
| Использование | Асинхронный код | Многопоточный код |
| Методы ожидания | await put/get | put/get (блокирующие) |
| Безопасность | Thread-safe не требуется | Thread-safe |
| Event loop | Требуется | Не требуется |

---

[prev: ./readme.md](./readme.md) | [next: ./02-priority-queues.md](./02-priority-queues.md)
