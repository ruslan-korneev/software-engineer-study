# Отладочный режим

[prev: ./08-manual-event-loop.md](./08-manual-event-loop.md) | [next: ../03-first-app/readme.md](../03-first-app/readme.md)

---

## Введение

Режим отладки asyncio помогает обнаружить распространённые ошибки в асинхронном коде:
- Забытые `await`
- Блокирующие операции в event loop
- Медленные callbacks
- Незавершённые задачи

## Включение режима отладки

### Способ 1: Через asyncio.run()

```python
import asyncio

async def main():
    await asyncio.sleep(1)

# Включаем debug mode
asyncio.run(main(), debug=True)
```

### Способ 2: Переменная окружения

```bash
# В терминале
PYTHONASYNCIODEBUG=1 python script.py
```

```python
import os
os.environ['PYTHONASYNCIODEBUG'] = '1'

import asyncio

async def main():
    await asyncio.sleep(1)

asyncio.run(main())
```

### Способ 3: Через Event Loop

```python
import asyncio

async def main():
    loop = asyncio.get_running_loop()
    loop.set_debug(True)

    print(f"Debug mode: {loop.get_debug()}")
    await asyncio.sleep(1)

asyncio.run(main())
```

### Способ 4: Глобальная настройка

```python
import asyncio

# Создаём политику с debug по умолчанию
class DebugEventLoopPolicy(asyncio.DefaultEventLoopPolicy):
    def new_event_loop(self):
        loop = super().new_event_loop()
        loop.set_debug(True)
        return loop

asyncio.set_event_loop_policy(DebugEventLoopPolicy())

async def main():
    loop = asyncio.get_running_loop()
    print(f"Debug: {loop.get_debug()}")  # True

asyncio.run(main())
```

## Что обнаруживает debug mode

### 1. Забытый await

```python
import asyncio

async def fetch_data():
    await asyncio.sleep(1)
    return "данные"

async def main():
    # Забыли await - в debug mode будет предупреждение
    result = fetch_data()
    print(result)

# Запуск с debug=True покажет:
# RuntimeWarning: coroutine 'fetch_data' was never awaited
asyncio.run(main(), debug=True)
```

### 2. Блокирующие вызовы

```python
import asyncio
import time

async def blocking_operation():
    # time.sleep блокирует event loop
    # В debug mode это будет обнаружено
    time.sleep(0.2)  # > slow_callback_duration
    return "готово"

async def main():
    await blocking_operation()

# Debug mode выведет предупреждение о медленном callback
asyncio.run(main(), debug=True)
# Executing <Task ...> took 0.201 seconds
```

### 3. Медленные callbacks

```python
import asyncio
import time

def slow_callback():
    """Медленный callback."""
    time.sleep(0.15)  # Превышает порог
    print("Callback выполнен")

async def main():
    loop = asyncio.get_running_loop()
    loop.set_debug(True)

    # Устанавливаем порог для "медленных" callbacks (по умолчанию 0.1 сек)
    loop.slow_callback_duration = 0.1

    loop.call_soon(slow_callback)
    await asyncio.sleep(0.5)

asyncio.run(main())
# Вывод предупреждения о медленном callback
```

### 4. Незавершённые задачи

```python
import asyncio

async def orphan_task():
    await asyncio.sleep(10)
    return "никогда не завершится"

async def main():
    # Создаём задачу, но не ждём её
    task = asyncio.create_task(orphan_task())
    await asyncio.sleep(0.1)
    # main завершается, task всё ещё выполняется

asyncio.run(main(), debug=True)
# Task was destroyed but it is pending!
```

## Настройка логирования

### Базовая настройка

```python
import asyncio
import logging

# Настраиваем логирование для asyncio
logging.basicConfig(
    level=logging.DEBUG,
    format='%(asctime)s - %(name)s - %(levelname)s - %(message)s'
)

# Устанавливаем уровень для asyncio
asyncio_logger = logging.getLogger('asyncio')
asyncio_logger.setLevel(logging.DEBUG)

async def main():
    await asyncio.sleep(1)

asyncio.run(main(), debug=True)
```

### Расширенное логирование

```python
import asyncio
import logging
import sys

def setup_async_logging():
    """Настройка детального логирования asyncio."""

    # Создаём форматтер
    formatter = logging.Formatter(
        '%(asctime)s [%(levelname)s] %(name)s: %(message)s',
        datefmt='%H:%M:%S'
    )

    # Handler для консоли
    console_handler = logging.StreamHandler(sys.stdout)
    console_handler.setFormatter(formatter)

    # Handler для файла
    file_handler = logging.FileHandler('asyncio_debug.log')
    file_handler.setFormatter(formatter)

    # Настраиваем логгер asyncio
    asyncio_logger = logging.getLogger('asyncio')
    asyncio_logger.setLevel(logging.DEBUG)
    asyncio_logger.addHandler(console_handler)
    asyncio_logger.addHandler(file_handler)

    return asyncio_logger

setup_async_logging()

async def main():
    logging.debug("Начало main")
    await asyncio.sleep(0.5)
    logging.debug("Конец main")

asyncio.run(main(), debug=True)
```

## Инструменты отладки

### Трассировка корутин

```python
import asyncio
import traceback

async def fetch_user(user_id: int):
    await asyncio.sleep(0.5)
    return {"id": user_id}

async def process_users():
    loop = asyncio.get_running_loop()

    # Получаем все текущие задачи
    tasks = asyncio.all_tasks(loop)
    print(f"Активных задач: {len(tasks)}")

    for task in tasks:
        print(f"\nЗадача: {task.get_name()}")
        print(f"  Done: {task.done()}")
        print(f"  Cancelled: {task.cancelled()}")

        # Получаем stack trace
        stack = task.get_stack()
        if stack:
            print("  Stack:")
            for frame in stack:
                print(f"    {frame}")

async def main():
    task1 = asyncio.create_task(fetch_user(1), name="fetch_user_1")
    task2 = asyncio.create_task(fetch_user(2), name="fetch_user_2")

    # Даём задачам начать выполнение
    await asyncio.sleep(0.1)

    await process_users()

    await asyncio.gather(task1, task2)

asyncio.run(main(), debug=True)
```

### Кастомный exception handler

```python
import asyncio
import logging

logging.basicConfig(level=logging.ERROR)

def custom_exception_handler(loop, context):
    """Кастомный обработчик исключений для event loop."""
    exception = context.get('exception')
    message = context.get('message')
    task = context.get('task')

    logging.error(f"Async exception caught!")
    logging.error(f"  Message: {message}")
    logging.error(f"  Task: {task}")

    if exception:
        logging.error(f"  Exception: {type(exception).__name__}: {exception}")

    # Можно добавить дополнительную логику:
    # - Отправка в систему мониторинга
    # - Запись в специальный лог
    # - Уведомление разработчиков

async def failing_task():
    await asyncio.sleep(0.5)
    raise ValueError("Что-то пошло не так!")

async def main():
    loop = asyncio.get_running_loop()
    loop.set_exception_handler(custom_exception_handler)

    task = asyncio.create_task(failing_task())

    # Не ждём задачу - исключение будет обработано handler'ом
    await asyncio.sleep(1)

asyncio.run(main())
```

### Профилирование задач

```python
import asyncio
import time
from dataclasses import dataclass, field
from typing import Dict, List

@dataclass
class TaskProfile:
    name: str
    start_time: float = 0
    end_time: float = 0
    duration: float = 0

class AsyncProfiler:
    """Профайлер для asyncio задач."""

    def __init__(self):
        self.profiles: Dict[str, TaskProfile] = {}

    def track_task(self, task: asyncio.Task):
        """Начинает отслеживание задачи."""
        name = task.get_name()
        self.profiles[name] = TaskProfile(
            name=name,
            start_time=time.perf_counter()
        )

        def on_done(t):
            if name in self.profiles:
                profile = self.profiles[name]
                profile.end_time = time.perf_counter()
                profile.duration = profile.end_time - profile.start_time

        task.add_done_callback(on_done)

    def report(self):
        """Выводит отчёт о профилировании."""
        print("\n=== Task Profiling Report ===")
        for name, profile in sorted(
            self.profiles.items(),
            key=lambda x: x[1].duration,
            reverse=True
        ):
            print(f"{name}: {profile.duration:.4f}s")

profiler = AsyncProfiler()

async def worker(name: str, delay: float):
    await asyncio.sleep(delay)
    return f"{name} done"

async def main():
    tasks = []
    for i in range(5):
        task = asyncio.create_task(
            worker(f"worker_{i}", 0.5 + i * 0.1),
            name=f"worker_{i}"
        )
        profiler.track_task(task)
        tasks.append(task)

    await asyncio.gather(*tasks)
    profiler.report()

asyncio.run(main())
```

## Отладка с использованием IDE

### VS Code launch.json

```json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Debug Async",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal",
            "env": {
                "PYTHONASYNCIODEBUG": "1"
            },
            "justMyCode": false
        }
    ]
}
```

### PyCharm

```python
# В настройках Run Configuration добавьте:
# Environment variables: PYTHONASYNCIODEBUG=1
```

## Практические паттерны отладки

### Паттерн: Debug wrapper

```python
import asyncio
import functools
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

def debug_async(func):
    """Декоратор для отладки async функций."""
    @functools.wraps(func)
    async def wrapper(*args, **kwargs):
        func_name = func.__name__
        logger.debug(f">>> {func_name} started with args={args}, kwargs={kwargs}")

        try:
            result = await func(*args, **kwargs)
            logger.debug(f"<<< {func_name} returned: {result}")
            return result
        except Exception as e:
            logger.error(f"!!! {func_name} raised: {type(e).__name__}: {e}")
            raise

    return wrapper

@debug_async
async def fetch_data(url: str) -> dict:
    await asyncio.sleep(0.5)
    return {"url": url, "data": "content"}

@debug_async
async def process_data(data: dict) -> str:
    await asyncio.sleep(0.3)
    return f"Processed: {data}"

async def main():
    data = await fetch_data("https://example.com")
    result = await process_data(data)
    print(result)

asyncio.run(main(), debug=True)
```

### Паттерн: Task monitor

```python
import asyncio
import time
from contextlib import asynccontextmanager

class TaskMonitor:
    """Мониторинг задач в реальном времени."""

    def __init__(self, interval: float = 1.0):
        self.interval = interval
        self._running = False
        self._task = None

    async def _monitor_loop(self):
        while self._running:
            loop = asyncio.get_running_loop()
            tasks = asyncio.all_tasks(loop)

            print(f"\n--- Task Monitor [{time.strftime('%H:%M:%S')}] ---")
            print(f"Total tasks: {len(tasks)}")

            for task in tasks:
                status = "DONE" if task.done() else "RUNNING"
                print(f"  [{status}] {task.get_name()}")

            await asyncio.sleep(self.interval)

    @asynccontextmanager
    async def monitoring(self):
        self._running = True
        self._task = asyncio.create_task(
            self._monitor_loop(),
            name="task_monitor"
        )

        try:
            yield
        finally:
            self._running = False
            self._task.cancel()
            try:
                await self._task
            except asyncio.CancelledError:
                pass

async def worker(name: str, duration: float):
    await asyncio.sleep(duration)
    return f"{name} completed"

async def main():
    monitor = TaskMonitor(interval=0.5)

    async with monitor.monitoring():
        tasks = [
            asyncio.create_task(worker(f"W{i}", 1 + i * 0.5), name=f"worker_{i}")
            for i in range(3)
        ]
        await asyncio.gather(*tasks)

asyncio.run(main())
```

### Паттерн: Assertion helpers

```python
import asyncio

async def assert_completes_within(coro, timeout: float, message: str = ""):
    """Проверяет, что корутина завершается в указанное время."""
    try:
        return await asyncio.wait_for(coro, timeout=timeout)
    except asyncio.TimeoutError:
        raise AssertionError(
            f"Operation did not complete within {timeout}s. {message}"
        )

async def assert_raises_within(coro, exception_type, timeout: float):
    """Проверяет, что корутина выбрасывает исключение в указанное время."""
    try:
        await asyncio.wait_for(coro, timeout=timeout)
        raise AssertionError(f"Expected {exception_type.__name__} was not raised")
    except exception_type:
        pass  # Ожидаемое исключение
    except asyncio.TimeoutError:
        raise AssertionError(f"Operation timed out before raising exception")

# Использование в тестах
async def test_operations():
    async def fast_operation():
        await asyncio.sleep(0.1)
        return "done"

    async def failing_operation():
        await asyncio.sleep(0.1)
        raise ValueError("Expected error")

    # Тест 1: Операция завершается вовремя
    result = await assert_completes_within(fast_operation(), 1.0)
    print(f"Test 1 passed: {result}")

    # Тест 2: Операция выбрасывает ожидаемое исключение
    await assert_raises_within(failing_operation(), ValueError, 1.0)
    print("Test 2 passed")

asyncio.run(test_operations())
```

## Best Practices

1. **Включайте debug mode при разработке** - это поможет найти ошибки раньше
2. **Настройте logging** для asyncio модуля
3. **Используйте кастомный exception handler** для централизованной обработки ошибок
4. **Профилируйте задачи** для выявления узких мест
5. **Отключайте debug mode в production** - он снижает производительность

```python
import asyncio
import os
import logging

def is_development():
    return os.environ.get('ENV', 'development') == 'development'

def setup_asyncio_debug():
    """Настройка отладки asyncio для разных окружений."""
    if is_development():
        # Полная отладка для разработки
        logging.basicConfig(level=logging.DEBUG)
        asyncio_logger = logging.getLogger('asyncio')
        asyncio_logger.setLevel(logging.DEBUG)
        return True
    else:
        # Минимальное логирование для production
        logging.basicConfig(level=logging.WARNING)
        return False

async def main():
    debug = setup_asyncio_debug()
    loop = asyncio.get_running_loop()
    loop.set_debug(debug)

    print(f"Running in {'debug' if debug else 'production'} mode")
    await asyncio.sleep(1)

asyncio.run(main())
```

---

[prev: ./08-manual-event-loop.md](./08-manual-event-loop.md) | [next: ../03-first-app/readme.md](../03-first-app/readme.md)
