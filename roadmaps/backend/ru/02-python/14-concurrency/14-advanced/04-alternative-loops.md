# Альтернативные реализации циклов событий

[prev: ./03-force-iteration.md](./03-force-iteration.md) | [next: ./05-custom-event-loop.md](./05-custom-event-loop.md)

---

## Введение

Стандартный event loop в asyncio реализован на чистом Python и использует модуль `selectors` для мультиплексирования I/O. Хотя он достаточно эффективен для большинства задач, существуют **альтернативные реализации**, которые могут значительно повысить производительность за счёт использования низкоуровневых системных вызовов.

## Почему нужны альтернативные Event Loop?

### Ограничения стандартного loop

1. **Реализация на Python** — интерпретируемый код медленнее, чем нативный C
2. **Универсальность** — оптимизирован для совместимости, а не максимальной скорости
3. **Абстракция selectors** — дополнительный слой над системными вызовами

### Преимущества альтернативных реализаций

1. **Написаны на C/Cython** — значительно быстрее
2. **Прямой доступ к системным API** — epoll, kqueue, IOCP
3. **Оптимизация для конкретных сценариев** — высоконагруженные сервера

## uvloop — самый популярный альтернативный Event Loop

**uvloop** — это быстрая замена стандартного asyncio event loop, построенная на основе библиотеки **libuv** (той же, что используется в Node.js).

### Установка

```bash
pip install uvloop
```

### Базовое использование

```python
import asyncio
import uvloop

async def main():
    print("Привет из uvloop!")
    await asyncio.sleep(1)
    return "Результат"

# Способ 1: Установить политику глобально
uvloop.install()
asyncio.run(main())
```

### Альтернативные способы настройки

```python
import asyncio
import uvloop

async def main():
    print("Работаем с uvloop!")
    await asyncio.sleep(0.1)

# Способ 2: Создать event loop явно
loop = uvloop.new_event_loop()
asyncio.set_event_loop(loop)
try:
    loop.run_until_complete(main())
finally:
    loop.close()

# Способ 3: Использовать EventLoopPolicy
asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
asyncio.run(main())
```

### Python 3.11+ с asyncio.Runner

```python
import asyncio
import uvloop

async def main():
    print("uvloop с Runner!")
    await asyncio.sleep(0.1)

# Python 3.11+
with asyncio.Runner(loop_factory=uvloop.new_event_loop) as runner:
    runner.run(main())
```

## Производительность uvloop

### Бенчмарк: TCP Echo Server

```python
import asyncio
import time
import uvloop

async def handle_client(reader, writer):
    """Простой echo-обработчик."""
    while True:
        data = await reader.read(1024)
        if not data:
            break
        writer.write(data)
        await writer.drain()
    writer.close()
    await writer.wait_closed()

async def benchmark_server(host='127.0.0.1', port=8888, duration=5):
    """Запускает echo-сервер и измеряет производительность."""
    server = await asyncio.start_server(handle_client, host, port)

    print(f"Сервер запущен на {host}:{port}")
    print(f"Работает {duration} секунд...")

    await asyncio.sleep(duration)

    server.close()
    await server.wait_closed()

def run_with_standard_loop():
    """Запуск со стандартным event loop."""
    print("\n=== Стандартный asyncio ===")
    asyncio.run(benchmark_server())

def run_with_uvloop():
    """Запуск с uvloop."""
    print("\n=== uvloop ===")
    uvloop.install()
    asyncio.run(benchmark_server())

# Тестирование производительности
# run_with_standard_loop()
# run_with_uvloop()
```

### Типичные результаты бенчмарков

В реальных сценариях uvloop показывает:

- **HTTP-сервера**: 2-4x быстрее
- **TCP Echo**: 2-3x быстрее
- **DNS-резолвинг**: до 2x быстрее
- **Операции с сокетами**: 1.5-3x быстрее

## Практический пример: Высокопроизводительный HTTP-клиент

```python
import asyncio
import uvloop
import time
from typing import List

# Устанавливаем uvloop
uvloop.install()

async def fetch_url(session, url: str) -> tuple:
    """Загружает URL и возвращает время выполнения."""
    start = time.monotonic()

    # Имитация HTTP-запроса
    await asyncio.sleep(0.01)  # Симуляция сетевой задержки

    elapsed = time.monotonic() - start
    return url, elapsed

async def fetch_many(urls: List[str]) -> List[tuple]:
    """Параллельная загрузка множества URL."""
    tasks = [fetch_url(None, url) for url in urls]
    return await asyncio.gather(*tasks)

async def benchmark():
    """Бенчмарк параллельных запросов."""
    urls = [f"http://example.com/page{i}" for i in range(1000)]

    start = time.monotonic()
    results = await fetch_many(urls)
    total_time = time.monotonic() - start

    print(f"Обработано {len(results)} URL за {total_time:.3f} секунд")
    print(f"Средняя скорость: {len(results) / total_time:.0f} req/sec")

asyncio.run(benchmark())
```

## Интеграция с веб-фреймворками

### FastAPI с uvloop

```python
# main.py
from fastapi import FastAPI
import uvloop

# uvloop автоматически используется при запуске через uvicorn
app = FastAPI()

@app.get("/")
async def root():
    return {"message": "Hello from FastAPI with uvloop!"}

# Запуск: uvicorn main:app --loop uvloop
```

### aiohttp с uvloop

```python
import asyncio
import uvloop
from aiohttp import web

async def handle(request):
    return web.Response(text="Hello, World!")

async def init_app():
    app = web.Application()
    app.router.add_get('/', handle)
    return app

if __name__ == '__main__':
    uvloop.install()
    app = asyncio.run(init_app())
    web.run_app(app, host='0.0.0.0', port=8080)
```

### Использование в Gunicorn

```python
# gunicorn_config.py
import uvloop

def post_fork(server, worker):
    uvloop.install()

# Запуск: gunicorn -c gunicorn_config.py myapp:app -k uvicorn.workers.UvicornWorker
```

## Другие альтернативные Event Loops

### tokio (через pyo3-asyncio)

**Tokio** — это асинхронный runtime для Rust. Можно использовать через Rust-биндинги:

```python
# Требует pyo3-asyncio
# Это более экспериментальный подход

# Пример использования Rust async из Python
# from my_rust_lib import async_function
# result = await async_function()
```

### winloop (для Windows)

**winloop** — версия uvloop, оптимизированная для Windows:

```bash
pip install winloop
```

```python
import asyncio
import sys

if sys.platform == 'win32':
    import winloop
    winloop.install()
else:
    import uvloop
    uvloop.install()

async def main():
    print("Кроссплатформенный код!")

asyncio.run(main())
```

## Сравнение Event Loop реализаций

| Характеристика | asyncio (stdlib) | uvloop | winloop |
|----------------|------------------|--------|---------|
| Платформа | Все | Linux, macOS | Windows |
| Реализация | Python | Cython + libuv | Cython + libuv |
| Производительность | Базовая | ~2-4x быстрее | ~2-3x быстрее |
| Совместимость | 100% | ~99% | ~95% |
| Отладка | Простая | Сложнее | Сложнее |

## Проверка совместимости с uvloop

Некоторые функции asyncio работают по-разному или не поддерживаются в uvloop:

```python
import asyncio
import uvloop

async def check_compatibility():
    """Проверка совместимости с uvloop."""

    loop = asyncio.get_running_loop()

    # Проверяем тип event loop
    print(f"Тип loop: {type(loop).__name__}")

    # Поддержка subprocess
    try:
        proc = await asyncio.create_subprocess_shell(
            'echo "test"',
            stdout=asyncio.subprocess.PIPE
        )
        stdout, _ = await proc.communicate()
        print(f"Subprocess работает: {stdout.decode().strip()}")
    except NotImplementedError:
        print("Subprocess не поддерживается")

    # Поддержка signal handlers
    import signal
    try:
        loop.add_signal_handler(signal.SIGUSR1, lambda: None)
        loop.remove_signal_handler(signal.SIGUSR1)
        print("Signal handlers поддерживаются")
    except (NotImplementedError, AttributeError):
        print("Signal handlers не поддерживаются")

# Тест без uvloop
print("=== Стандартный asyncio ===")
asyncio.run(check_compatibility())

# Тест с uvloop
print("\n=== uvloop ===")
uvloop.install()
asyncio.run(check_compatibility())
```

## Условное использование uvloop

Паттерн для опционального использования uvloop:

```python
import asyncio
import sys

def setup_event_loop():
    """Настраивает оптимальный event loop для платформы."""

    if sys.platform == 'win32':
        # Windows: пробуем winloop или оставляем стандартный
        try:
            import winloop
            winloop.install()
            return "winloop"
        except ImportError:
            # ProactorEventLoop для лучшей совместимости на Windows
            asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())
            return "ProactorEventLoop"
    else:
        # Unix: пробуем uvloop
        try:
            import uvloop
            uvloop.install()
            return "uvloop"
        except ImportError:
            return "asyncio (default)"

async def main():
    loop_type = setup_event_loop()
    print(f"Используется: {loop_type}")

    # Ваш асинхронный код
    await asyncio.sleep(0.1)

# Сначала настраиваем loop, потом запускаем
loop_type = setup_event_loop()
print(f"Event loop: {loop_type}")
asyncio.run(main())
```

## Отладка и профилирование

### Debug-режим с uvloop

```python
import asyncio
import uvloop
import logging

# Настройка логирования
logging.basicConfig(level=logging.DEBUG)

async def potentially_slow():
    """Функция, которая может блокировать loop."""
    import time
    time.sleep(0.2)  # Блокирующая операция!

async def main():
    await potentially_slow()

# uvloop с debug
uvloop.install()

# Включаем debug-режим asyncio
# PYTHONASYNCIODEBUG=1 python script.py
# Или программно:
loop = asyncio.new_event_loop()
loop.set_debug(True)
asyncio.set_event_loop(loop)

try:
    loop.run_until_complete(main())
finally:
    loop.close()
```

## Best Practices

### 1. Устанавливайте uvloop как можно раньше

```python
# ХОРОШО: в начале main.py
import uvloop
uvloop.install()

# Далее весь код использует uvloop
import asyncio
from myapp import main
asyncio.run(main())
```

### 2. Тестируйте с обоими event loops

```python
import pytest
import asyncio

@pytest.fixture(params=['asyncio', 'uvloop'])
def event_loop_policy(request):
    if request.param == 'uvloop':
        import uvloop
        return uvloop.EventLoopPolicy()
    return asyncio.DefaultEventLoopPolicy()

@pytest.fixture
def event_loop(event_loop_policy):
    asyncio.set_event_loop_policy(event_loop_policy)
    loop = asyncio.new_event_loop()
    yield loop
    loop.close()

async def test_my_async_code(event_loop):
    # Тест выполнится дважды: с asyncio и uvloop
    result = await some_async_function()
    assert result == expected
```

### 3. Документируйте зависимость от uvloop

```python
# requirements.txt
uvloop>=0.17.0; sys_platform != 'win32'
winloop>=0.0.4; sys_platform == 'win32'

# pyproject.toml
[project.optional-dependencies]
fast = [
    "uvloop>=0.17.0; sys_platform != 'win32'",
    "winloop>=0.0.4; sys_platform == 'win32'",
]
```

## Распространённые ошибки

### 1. Установка uvloop после создания loop

```python
import asyncio

# НЕПРАВИЛЬНО: loop уже создан
loop = asyncio.new_event_loop()
import uvloop
uvloop.install()  # Не повлияет на существующий loop!

# ПРАВИЛЬНО: сначала uvloop
import uvloop
uvloop.install()
import asyncio
loop = asyncio.new_event_loop()  # Это будет uvloop
```

### 2. Смешивание разных event loops

```python
import asyncio
import uvloop

# НЕПРАВИЛЬНО: разные loops
loop1 = asyncio.new_event_loop()
uvloop.install()
loop2 = asyncio.new_event_loop()  # Это uvloop

# Task из loop1 не может выполняться в loop2!
```

### 3. Игнорирование ограничений uvloop

```python
# uvloop не поддерживает некоторые методы
# Например, loop.add_reader с файлами (не сокетами)
```

## Резюме

- **uvloop** — самая популярная альтернатива, ускоряет код в 2-4 раза
- Основан на **libuv** (Node.js runtime)
- **winloop** — аналог для Windows
- Устанавливайте uvloop **до** импорта asyncio
- Не все функции asyncio поддерживаются в uvloop
- Используйте условную загрузку для кроссплатформенности
- Тестируйте код с обоими event loops

---

[prev: ./03-force-iteration.md](./03-force-iteration.md) | [next: ./05-custom-event-loop.md](./05-custom-event-loop.md)
