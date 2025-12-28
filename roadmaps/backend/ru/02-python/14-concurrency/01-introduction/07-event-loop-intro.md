# Как работает цикл событий

[prev: ./06-single-threaded.md](./06-single-threaded.md) | [next: ../02-basics/readme.md](../02-basics/readme.md)

---

## Введение

**Event Loop (цикл событий)** - это сердце asyncio. Это бесконечный цикл, который координирует выполнение асинхронных задач, обрабатывает I/O события и управляет колбэками.

## Концепция Event Loop

```
┌─────────────────────────────────────────────────────────────────┐
│                        EVENT LOOP                                │
│                                                                  │
│    ┌──────────────────────────────────────────────────────┐     │
│    │                  Бесконечный цикл                    │     │
│    │                                                       │     │
│    │   while True:                                        │     │
│    │       1. Проверить готовые задачи                    │     │
│    │       2. Выполнить готовые колбэки                   │     │
│    │       3. Ждать I/O события                           │     │
│    │       4. Обработать I/O события                      │     │
│    │       5. Повторить                                   │     │
│    │                                                       │     │
│    └──────────────────────────────────────────────────────┘     │
│                                                                  │
│    ┌──────────────┐ ┌──────────────┐ ┌──────────────┐           │
│    │ Очередь      │ │ I/O          │ │ Таймеры      │           │
│    │ задач        │ │ мониторинг   │ │              │           │
│    └──────────────┘ └──────────────┘ └──────────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Компоненты Event Loop

### 1. Очередь готовых задач (Ready Queue)

```python
import asyncio

async def task_demo():
    print("Задача выполняется")
    await asyncio.sleep(0)  # Отдаёт контроль, возвращается в очередь
    print("Задача продолжается")

async def main():
    # Создание задач добавляет их в очередь
    task1 = asyncio.create_task(task_demo())
    task2 = asyncio.create_task(task_demo())
    task3 = asyncio.create_task(task_demo())

    # Event loop выполняет задачи из очереди
    await asyncio.gather(task1, task2, task3)

asyncio.run(main())
```

### 2. I/O Selector (мониторинг I/O)

```python
import asyncio
import socket

async def io_monitoring_demo():
    """
    Event loop использует селекторы ОС для мониторинга I/O:
    - select (старый, ограниченный)
    - poll (Linux)
    - epoll (Linux, более эффективный)
    - kqueue (BSD, macOS)
    - IOCP (Windows)
    """

    # Создаём сокет
    reader, writer = await asyncio.open_connection('example.com', 80)

    # Event loop регистрирует сокет в селекторе
    # и уведомляется, когда данные готовы

    writer.write(b'GET / HTTP/1.1\r\nHost: example.com\r\n\r\n')
    await writer.drain()

    # Ожидание данных (event loop мониторит сокет)
    data = await reader.read(1024)

    writer.close()
    await writer.wait_closed()

    return data

asyncio.run(io_monitoring_demo())
```

### 3. Таймеры и отложенные вызовы

```python
import asyncio

async def timer_demo():
    loop = asyncio.get_running_loop()

    # call_later - вызов через N секунд
    loop.call_later(2.0, print, "Через 2 секунды")

    # call_at - вызов в определённое время
    loop.call_at(loop.time() + 3.0, print, "Через 3 секунды")

    # call_soon - вызов на следующей итерации
    loop.call_soon(print, "Немедленно (на следующей итерации)")

    await asyncio.sleep(4)

asyncio.run(timer_demo())
```

## Жизненный цикл Event Loop

```python
import asyncio

# 1. Создание event loop
loop = asyncio.new_event_loop()

# 2. Установка как текущего
asyncio.set_event_loop(loop)

# 3. Запуск задач
async def my_task():
    print("Выполняется задача")
    await asyncio.sleep(1)
    return "Результат"

try:
    # 4. Выполнение до завершения
    result = loop.run_until_complete(my_task())
    print(f"Результат: {result}")
finally:
    # 5. Закрытие loop
    loop.close()

# Современный способ (Python 3.7+)
# asyncio.run() делает всё это автоматически
asyncio.run(my_task())
```

## Как работает один цикл итерации

```python
"""
Псевдокод одной итерации event loop:
"""

def one_iteration(self):
    # 1. Выполнить все готовые колбэки
    while self._ready:
        callback = self._ready.popleft()
        callback()

    # 2. Рассчитать таймаут для ожидания I/O
    timeout = self._calculate_timeout()

    # 3. Ждать I/O события (select/epoll/kqueue)
    events = self._selector.select(timeout)

    # 4. Обработать I/O события
    for key, mask in events:
        callback = key.data
        self._ready.append(callback)

    # 5. Обработать сработавшие таймеры
    now = self.time()
    while self._scheduled and self._scheduled[0].when <= now:
        timer = heapq.heappop(self._scheduled)
        self._ready.append(timer.callback)
```

## Практическая демонстрация

### Визуализация работы event loop

```python
import asyncio
import time

async def task(name: str, work_time: float, wait_time: float):
    """Задача с работой и ожиданием"""
    print(f"[{time.strftime('%H:%M:%S')}] {name}: начало работы")

    # Симуляция CPU-работы (не отдаёт контроль)
    end = time.time() + work_time
    while time.time() < end:
        pass

    print(f"[{time.strftime('%H:%M:%S')}] {name}: ожидание I/O")

    # Симуляция I/O (отдаёт контроль)
    await asyncio.sleep(wait_time)

    print(f"[{time.strftime('%H:%M:%S')}] {name}: завершение")
    return name

async def main():
    start = time.time()

    results = await asyncio.gather(
        task("A", work_time=0.1, wait_time=1.0),
        task("B", work_time=0.1, wait_time=1.0),
        task("C", work_time=0.1, wait_time=1.0),
    )

    print(f"\nОбщее время: {time.time() - start:.2f} сек")
    print(f"Результаты: {results}")

asyncio.run(main())
```

Вывод:
```
[12:00:00] A: начало работы
[12:00:00] A: ожидание I/O
[12:00:00] B: начало работы
[12:00:00] B: ожидание I/O
[12:00:00] C: начало работы
[12:00:00] C: ожидание I/O
[12:00:01] A: завершение
[12:00:01] B: завершение
[12:00:01] C: завершение

Общее время: 1.30 сек
Результаты: ['A', 'B', 'C']
```

## Методы Event Loop

### Получение текущего loop

```python
import asyncio

async def get_loop_demo():
    # Внутри async функции
    loop = asyncio.get_running_loop()
    print(f"Текущий loop: {loop}")

    # Информация о loop
    print(f"Время loop: {loop.time()}")
    print(f"Запущен: {loop.is_running()}")
    print(f"Закрыт: {loop.is_closed()}")

asyncio.run(get_loop_demo())
```

### Планирование вызовов

```python
import asyncio

async def scheduling_demo():
    loop = asyncio.get_running_loop()
    results = []

    def callback(msg):
        results.append(msg)
        print(f"Callback: {msg}")

    # call_soon - следующая итерация
    loop.call_soon(callback, "soon")

    # call_later - через N секунд
    loop.call_later(0.5, callback, "later_0.5")
    loop.call_later(0.2, callback, "later_0.2")

    # call_at - в определённое время
    loop.call_at(loop.time() + 0.3, callback, "at_0.3")

    await asyncio.sleep(1)

    print(f"\nПорядок выполнения: {results}")
    # ['soon', 'later_0.2', 'at_0.3', 'later_0.5']

asyncio.run(scheduling_demo())
```

### Выполнение в executor

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import time

def blocking_io():
    """Блокирующая I/O операция"""
    time.sleep(1)
    return "I/O завершён"

def cpu_intensive(n):
    """CPU-интенсивная операция"""
    return sum(i ** 2 for i in range(n))

async def executor_demo():
    loop = asyncio.get_running_loop()

    # ThreadPoolExecutor для I/O (по умолчанию)
    result1 = await loop.run_in_executor(None, blocking_io)
    print(f"ThreadPool: {result1}")

    # Явный ThreadPoolExecutor
    with ThreadPoolExecutor(max_workers=4) as executor:
        result2 = await loop.run_in_executor(executor, blocking_io)
        print(f"Explicit ThreadPool: {result2}")

    # ProcessPoolExecutor для CPU-bound
    with ProcessPoolExecutor(max_workers=4) as executor:
        result3 = await loop.run_in_executor(executor, cpu_intensive, 1000000)
        print(f"ProcessPool: {result3}")

asyncio.run(executor_demo())
```

## Event Loop и сигналы

```python
import asyncio
import signal

async def signal_demo():
    loop = asyncio.get_running_loop()
    stop_event = asyncio.Event()

    def signal_handler():
        print("Получен сигнал, завершаем...")
        stop_event.set()

    # Регистрация обработчика сигнала
    loop.add_signal_handler(signal.SIGINT, signal_handler)
    loop.add_signal_handler(signal.SIGTERM, signal_handler)

    print("Нажмите Ctrl+C для выхода")

    try:
        await stop_event.wait()
    finally:
        loop.remove_signal_handler(signal.SIGINT)
        loop.remove_signal_handler(signal.SIGTERM)

# asyncio.run(signal_demo())  # Раскомментируйте для теста
```

## Debug режим Event Loop

```python
import asyncio

async def slow_callback():
    # Это заблокирует event loop
    import time
    time.sleep(0.2)  # Блокирующий вызов!

async def debug_demo():
    await asyncio.gather(
        slow_callback(),
        asyncio.sleep(0.1)
    )

# Включаем debug режим
asyncio.run(debug_demo(), debug=True)

# Или через переменную окружения:
# PYTHONASYNCIODEBUG=1 python script.py
```

В debug режиме вы увидите предупреждение:
```
Executing <Task ...> took 0.200 seconds
```

## Создание собственного Event Loop

```python
import asyncio
from collections import deque
import heapq
import time

class SimpleEventLoop:
    """Упрощённая реализация event loop для понимания"""

    def __init__(self):
        self._ready = deque()      # Готовые к выполнению
        self._scheduled = []       # Отложенные (heap)
        self._stopping = False

    def time(self):
        return time.monotonic()

    def call_soon(self, callback, *args):
        """Добавить в очередь готовых"""
        self._ready.append((callback, args))

    def call_later(self, delay, callback, *args):
        """Добавить с задержкой"""
        when = self.time() + delay
        heapq.heappush(self._scheduled, (when, callback, args))

    def run_forever(self):
        """Основной цикл"""
        while not self._stopping:
            self._run_once()

    def _run_once(self):
        """Одна итерация цикла"""
        # Обрабатываем сработавшие таймеры
        now = self.time()
        while self._scheduled and self._scheduled[0][0] <= now:
            when, callback, args = heapq.heappop(self._scheduled)
            self._ready.append((callback, args))

        # Выполняем готовые колбэки
        while self._ready:
            callback, args = self._ready.popleft()
            callback(*args)

        # Небольшая пауза, если нечего делать
        if not self._ready and not self._scheduled:
            time.sleep(0.01)

    def stop(self):
        self._stopping = True

# Демонстрация
def demo():
    loop = SimpleEventLoop()

    def printer(msg):
        print(f"[{time.strftime('%H:%M:%S')}] {msg}")

    def stopper():
        loop.stop()

    loop.call_soon(printer, "Немедленно")
    loop.call_later(0.5, printer, "Через 0.5 сек")
    loop.call_later(1.0, printer, "Через 1 сек")
    loop.call_later(1.5, stopper)

    loop.run_forever()
    print("Event loop остановлен")

demo()
```

## Best Practices

### 1. Используйте asyncio.run()

```python
# ПРАВИЛЬНО (Python 3.7+)
asyncio.run(main())

# УСТАРЕВШИЙ способ
loop = asyncio.get_event_loop()
loop.run_until_complete(main())
loop.close()
```

### 2. Не создавайте event loop в async функции

```python
# НЕПРАВИЛЬНО
async def bad():
    loop = asyncio.new_event_loop()  # Зачем?

# ПРАВИЛЬНО
async def good():
    loop = asyncio.get_running_loop()  # Текущий loop
```

### 3. Обрабатывайте исключения

```python
async def robust_main():
    try:
        await risky_operation()
    except Exception as e:
        print(f"Ошибка: {e}")
    finally:
        # Очистка ресурсов
        await cleanup()

asyncio.run(robust_main())
```

### 4. Graceful shutdown

```python
import asyncio
import signal

async def main():
    tasks = []

    async def worker(n):
        try:
            while True:
                print(f"Worker {n} работает")
                await asyncio.sleep(1)
        except asyncio.CancelledError:
            print(f"Worker {n} остановлен")
            raise

    # Запускаем воркеры
    for i in range(3):
        tasks.append(asyncio.create_task(worker(i)))

    # Ждём сигнал остановки
    stop = asyncio.Event()

    loop = asyncio.get_running_loop()
    for sig in (signal.SIGINT, signal.SIGTERM):
        loop.add_signal_handler(sig, stop.set)

    await stop.wait()

    # Graceful shutdown
    print("Останавливаем воркеров...")
    for task in tasks:
        task.cancel()

    await asyncio.gather(*tasks, return_exceptions=True)
    print("Все воркеры остановлены")

# asyncio.run(main())
```

## Заключение

Event Loop - это ядро asyncio, которое:

- **Управляет выполнением** корутин и задач
- **Мониторит I/O** через селекторы ОС
- **Обрабатывает таймеры** и отложенные вызовы
- **Координирует** переключение между задачами

Понимание работы event loop помогает писать эффективный асинхронный код и избегать типичных ошибок, таких как блокировка цикла событий.

---

[prev: ./06-single-threaded.md](./06-single-threaded.md) | [next: ../02-basics/readme.md](../02-basics/readme.md)
