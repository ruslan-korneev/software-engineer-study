# Совместное использование потоков и asyncio

[prev: 01-threading-intro.md](./01-threading-intro.md) | [next: 03-locks-deadlocks.md](./03-locks-deadlocks.md)

---

## Зачем комбинировать потоки и asyncio?

Asyncio отлично подходит для асинхронных I/O операций, но не все библиотеки поддерживают асинхронность. Когда нужно использовать **блокирующий код** в асинхронном приложении, на помощь приходят потоки.

### Типичные сценарии

| Сценарий | Решение |
|----------|---------|
| Блокирующая библиотека в async коде | `run_in_executor()` |
| Async код в синхронном приложении | `asyncio.run()` в потоке |
| GUI + async операции | Event loop в отдельном потоке |
| Legacy код + современный async | Мост между парадигмами |

## run_in_executor: запуск блокирующего кода

Метод `loop.run_in_executor()` позволяет выполнять блокирующие функции в пуле потоков:

```python
import asyncio
import time
from concurrent.futures import ThreadPoolExecutor

def blocking_io_operation(url: str) -> str:
    """Симуляция блокирующей операции."""
    print(f"Начинаю загрузку {url}")
    time.sleep(2)  # Блокирующий вызов
    return f"Данные с {url}"

async def fetch_data(url: str) -> str:
    """Асинхронная обёртка над блокирующей функцией."""
    loop = asyncio.get_running_loop()

    # Выполняем блокирующую функцию в пуле потоков
    result = await loop.run_in_executor(
        None,  # None = использовать default executor
        blocking_io_operation,
        url
    )
    return result

async def main():
    urls = ["https://site1.com", "https://site2.com", "https://site3.com"]

    # Параллельная загрузка через потоки
    tasks = [fetch_data(url) for url in urls]
    results = await asyncio.gather(*tasks)

    for result in results:
        print(result)

asyncio.run(main())
```

## Использование кастомного пула потоков

Можно создать собственный пул с нужными параметрами:

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import time

def cpu_intensive_task(n: int) -> int:
    """Вычислительная задача."""
    result = sum(i * i for i in range(n))
    return result

async def process_with_custom_executor():
    loop = asyncio.get_running_loop()

    # Создаём пул с 4 потоками
    with ThreadPoolExecutor(max_workers=4, thread_name_prefix="Worker") as executor:
        # Множество задач
        tasks = [
            loop.run_in_executor(executor, cpu_intensive_task, 1_000_000),
            loop.run_in_executor(executor, cpu_intensive_task, 2_000_000),
            loop.run_in_executor(executor, cpu_intensive_task, 500_000),
        ]

        results = await asyncio.gather(*tasks)
        print(f"Результаты: {results}")

asyncio.run(process_with_custom_executor())
```

## asyncio.to_thread (Python 3.9+)

Начиная с Python 3.9, есть удобная функция `asyncio.to_thread()`:

```python
import asyncio
import time

def blocking_function(name: str, duration: float) -> str:
    """Блокирующая функция."""
    time.sleep(duration)
    return f"Задача {name} выполнена за {duration}с"

async def main():
    # Простой и чистый синтаксис
    results = await asyncio.gather(
        asyncio.to_thread(blocking_function, "A", 2),
        asyncio.to_thread(blocking_function, "B", 1),
        asyncio.to_thread(blocking_function, "C", 1.5),
    )

    for result in results:
        print(result)

asyncio.run(main())
```

### Сравнение подходов

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

def sync_task(x: int) -> int:
    return x ** 2

async def main():
    loop = asyncio.get_running_loop()

    # Способ 1: run_in_executor с None (default executor)
    result1 = await loop.run_in_executor(None, sync_task, 5)

    # Способ 2: run_in_executor с кастомным executor
    with ThreadPoolExecutor() as executor:
        result2 = await loop.run_in_executor(executor, sync_task, 5)

    # Способ 3: to_thread (Python 3.9+)
    result3 = await asyncio.to_thread(sync_task, 5)

    print(f"Все результаты: {result1}, {result2}, {result3}")

asyncio.run(main())
```

## Потокобезопасное взаимодействие с event loop

При работе из другого потока нужно использовать специальные методы:

```python
import asyncio
import threading
import time

async def async_task(name: str) -> str:
    await asyncio.sleep(1)
    return f"Async задача {name} завершена"

def thread_worker(loop: asyncio.AbstractEventLoop):
    """Функция, выполняемая в отдельном потоке."""

    # НЕПРАВИЛЬНО: loop.create_task() — не потокобезопасно!
    # task = loop.create_task(async_task("wrong"))

    # ПРАВИЛЬНО: использовать run_coroutine_threadsafe
    future = asyncio.run_coroutine_threadsafe(
        async_task("from_thread"),
        loop
    )

    # Ожидаем результат (блокирующий вызов)
    result = future.result(timeout=5)
    print(f"Поток получил результат: {result}")

async def main():
    loop = asyncio.get_running_loop()

    # Запускаем поток, который будет взаимодействовать с loop
    thread = threading.Thread(target=thread_worker, args=(loop,))
    thread.start()

    # Продолжаем асинхронную работу
    await asyncio.sleep(2)
    thread.join()

asyncio.run(main())
```

## call_soon_threadsafe

Для вызова обычных callback'ов из другого потока:

```python
import asyncio
import threading
import time

def callback_from_thread(loop: asyncio.AbstractEventLoop, message: str):
    """Callback, вызываемый из потока."""

    def print_message():
        print(f"[EventLoop] Получено: {message}")

    # Потокобезопасно планируем вызов
    loop.call_soon_threadsafe(print_message)

async def main():
    loop = asyncio.get_running_loop()

    def thread_function():
        for i in range(5):
            time.sleep(0.5)
            callback_from_thread(loop, f"Сообщение #{i}")

    thread = threading.Thread(target=thread_function)
    thread.start()

    # Главный цикл продолжает работу
    for i in range(6):
        print(f"[Main] Итерация {i}")
        await asyncio.sleep(0.5)

    thread.join()

asyncio.run(main())
```

## Практический пример: интеграция с requests

Библиотека `requests` — синхронная, но можно использовать её в async коде:

```python
import asyncio
import requests
from typing import List, Dict

def fetch_sync(url: str) -> Dict:
    """Синхронная загрузка с requests."""
    response = requests.get(url, timeout=10)
    return {
        "url": url,
        "status": response.status_code,
        "length": len(response.content)
    }

async def fetch_multiple(urls: List[str]) -> List[Dict]:
    """Асинхронная загрузка множества URL через потоки."""
    tasks = [
        asyncio.to_thread(fetch_sync, url)
        for url in urls
    ]
    return await asyncio.gather(*tasks, return_exceptions=True)

async def main():
    urls = [
        "https://httpbin.org/get",
        "https://httpbin.org/ip",
        "https://httpbin.org/headers",
    ]

    results = await fetch_multiple(urls)

    for result in results:
        if isinstance(result, Exception):
            print(f"Ошибка: {result}")
        else:
            print(f"{result['url']}: {result['status']}, {result['length']} bytes")

asyncio.run(main())
```

## Контекстные переменные и потоки

При использовании `contextvars` нужно учитывать особенности:

```python
import asyncio
import contextvars
from concurrent.futures import ThreadPoolExecutor

# Контекстная переменная
request_id = contextvars.ContextVar("request_id", default="unknown")

def sync_function() -> str:
    """Синхронная функция, использующая контекстную переменную."""
    current_id = request_id.get()
    return f"Обработка запроса {current_id}"

async def handle_request(req_id: str) -> str:
    # Устанавливаем контекст
    request_id.set(req_id)

    # to_thread автоматически копирует контекст
    result = await asyncio.to_thread(sync_function)
    return result

async def main():
    results = await asyncio.gather(
        handle_request("REQ-001"),
        handle_request("REQ-002"),
        handle_request("REQ-003"),
    )

    for result in results:
        print(result)

asyncio.run(main())
```

## Обработка ошибок

Правильная обработка исключений при комбинировании потоков и asyncio:

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

def risky_operation(x: int) -> int:
    """Операция, которая может упасть."""
    if x == 0:
        raise ValueError("Деление на ноль!")
    return 100 // x

async def safe_execute(x: int) -> int | None:
    """Безопасное выполнение в потоке."""
    try:
        result = await asyncio.to_thread(risky_operation, x)
        return result
    except ValueError as e:
        logger.error(f"Ошибка при x={x}: {e}")
        return None
    except Exception as e:
        logger.exception(f"Неожиданная ошибка: {e}")
        return None

async def main():
    values = [10, 5, 0, 2, 1]

    results = await asyncio.gather(
        *[safe_execute(x) for x in values]
    )

    for x, result in zip(values, results):
        print(f"f({x}) = {result}")

asyncio.run(main())
```

## Best Practices

1. **Используйте `asyncio.to_thread()`** для простых случаев (Python 3.9+)
2. **Создавайте кастомный executor** для контроля количества потоков
3. **Всегда используйте потокобезопасные методы** при взаимодействии с loop из потока
4. **Обрабатывайте исключения** на обоих уровнях (поток и asyncio)
5. **Ограничивайте количество потоков** — слишком много потоков снижает производительность

```python
# Рекомендуемый паттерн
import asyncio
from concurrent.futures import ThreadPoolExecutor

# Глобальный пул с ограничением
EXECUTOR = ThreadPoolExecutor(max_workers=10)

async def run_blocking(func, *args):
    """Универсальная обёртка для блокирующих функций."""
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(EXECUTOR, func, *args)

# Не забывайте закрывать executor при завершении
# EXECUTOR.shutdown(wait=True)
```

## Распространённые ошибки

1. **Вызов async функций в потоке** — используйте `run_coroutine_threadsafe()`
2. **Блокирование event loop** — выносите блокирующий код в потоки
3. **Утечка потоков** — всегда закрывайте executor
4. **Игнорирование потокобезопасности** — используйте специальные методы asyncio

---

[prev: 01-threading-intro.md](./01-threading-intro.md) | [next: 03-locks-deadlocks.md](./03-locks-deadlocks.md)
