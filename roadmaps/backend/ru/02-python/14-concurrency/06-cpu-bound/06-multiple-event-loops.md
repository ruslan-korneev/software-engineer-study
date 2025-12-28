# Несколько циклов событий

[prev: 05-shared-data.md](./05-shared-data.md) | [next: ../07-threading/readme.md](../07-threading/readme.md)

---

## Введение

В некоторых сценариях требуется запускать несколько event loop одновременно. Это может быть полезно для:

- Изоляции CPU-bound и I/O-bound задач
- Масштабирования на многоядерных системах
- Интеграции asyncio с синхронным кодом

## Основные правила asyncio event loop

```python
import asyncio

# Получить текущий event loop
loop = asyncio.get_event_loop()

# Создать новый event loop
new_loop = asyncio.new_event_loop()

# Установить event loop как текущий для потока
asyncio.set_event_loop(new_loop)

# Запустить корутину (Python 3.7+)
asyncio.run(some_coroutine())  # Создает, запускает и закрывает loop
```

### Важно: один event loop на поток

В asyncio действует правило: **один event loop на один поток**. Нельзя использовать один event loop из нескольких потоков без специальных мер.

## Event Loop в отдельных потоках

```python
import asyncio
import threading
import time

async def async_task(name, delay):
    print(f"[{name}] Начало задачи")
    await asyncio.sleep(delay)
    print(f"[{name}] Конец задачи")
    return f"Результат {name}"

def run_loop_in_thread(loop_name):
    """Функция для запуска event loop в потоке"""
    # Создаем новый event loop для этого потока
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    try:
        # Запускаем задачи
        result = loop.run_until_complete(
            asyncio.gather(
                async_task(f"{loop_name}-1", 1),
                async_task(f"{loop_name}-2", 2),
            )
        )
        print(f"[{loop_name}] Результаты: {result}")
    finally:
        loop.close()

if __name__ == "__main__":
    # Запускаем два потока с разными event loops
    threads = [
        threading.Thread(target=run_loop_in_thread, args=(f"Loop-{i}",))
        for i in range(2)
    ]

    for t in threads:
        t.start()

    for t in threads:
        t.join()

    print("Все потоки завершены")
```

## Event Loop в отдельных процессах

```python
import asyncio
import multiprocessing
import os

async def async_worker(process_id):
    """Асинхронная работа в процессе"""
    print(f"Process {process_id} (PID: {os.getpid()}): начало")

    tasks = [
        asyncio.create_task(async_compute(process_id, i))
        for i in range(3)
    ]

    results = await asyncio.gather(*tasks)
    print(f"Process {process_id}: результаты {results}")
    return sum(results)

async def async_compute(process_id, task_id):
    """Имитация асинхронной работы"""
    await asyncio.sleep(0.5)
    return process_id * 10 + task_id

def process_entry(process_id):
    """Точка входа для процесса"""
    # Каждый процесс создает свой event loop
    result = asyncio.run(async_worker(process_id))
    return result

if __name__ == "__main__":
    with multiprocessing.Pool(4) as pool:
        results = pool.map(process_entry, range(4))

    print(f"Итоговые результаты: {results}")
```

## Комбинирование asyncio и multiprocessing

### ProcessPoolExecutor с asyncio

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import time

def cpu_bound_sync(n):
    """Синхронная CPU-bound функция"""
    result = 0
    for i in range(n):
        result += i ** 2
    return result

async def run_cpu_tasks():
    """Запуск CPU-bound задач из asyncio"""
    loop = asyncio.get_event_loop()

    with ProcessPoolExecutor(max_workers=4) as executor:
        # Запускаем в отдельных процессах
        tasks = [
            loop.run_in_executor(executor, cpu_bound_sync, 5_000_000)
            for _ in range(4)
        ]

        results = await asyncio.gather(*tasks)
        return results

async def main():
    # Можем выполнять I/O параллельно с CPU задачами
    cpu_task = asyncio.create_task(run_cpu_tasks())

    # Пока CPU задачи выполняются в процессах,
    # можем делать I/O операции
    await asyncio.sleep(0.1)
    print("I/O операции продолжаются...")

    results = await cpu_task
    print(f"CPU результаты: {results}")

if __name__ == "__main__":
    asyncio.run(main())
```

### Запуск event loop в каждом процессе пула

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import aiohttp

async def fetch_urls(urls):
    """Асинхронная загрузка URL в процессе"""
    async with aiohttp.ClientSession() as session:
        results = []
        for url in urls:
            try:
                async with session.get(url) as response:
                    results.append((url, response.status))
            except Exception as e:
                results.append((url, str(e)))
        return results

def process_urls(urls):
    """Синхронная обертка для запуска в процессе"""
    return asyncio.run(fetch_urls(urls))

async def distributed_fetch(all_urls, num_workers=4):
    """Распределенная загрузка URL по процессам"""
    loop = asyncio.get_event_loop()

    # Разбиваем URL на части
    chunk_size = len(all_urls) // num_workers
    chunks = [
        all_urls[i:i + chunk_size]
        for i in range(0, len(all_urls), chunk_size)
    ]

    with ProcessPoolExecutor(max_workers=num_workers) as executor:
        tasks = [
            loop.run_in_executor(executor, process_urls, chunk)
            for chunk in chunks
        ]

        results = await asyncio.gather(*tasks)

    # Объединяем результаты
    return [item for sublist in results for item in sublist]

if __name__ == "__main__":
    urls = [
        "https://httpbin.org/get",
        "https://httpbin.org/ip",
        "https://httpbin.org/headers",
        "https://httpbin.org/user-agent",
    ] * 10  # 40 URL

    results = asyncio.run(distributed_fetch(urls))
    print(f"Загружено {len(results)} URL")
```

## Паттерн: Master-Worker с asyncio

```python
import asyncio
import multiprocessing
from multiprocessing import Queue
import os

def worker_process(task_queue, result_queue):
    """
    Воркер-процесс с собственным event loop
    """
    async def process_task(task):
        # Имитация асинхронной обработки
        await asyncio.sleep(0.1)
        return task ** 2

    async def worker_main():
        while True:
            try:
                task = task_queue.get(timeout=1)
                if task is None:  # Сигнал завершения
                    break

                result = await process_task(task)
                result_queue.put((task, result, os.getpid()))
            except:
                break

    # Запускаем event loop в процессе
    asyncio.run(worker_main())

async def master(num_workers=4):
    """
    Мастер-процесс, распределяющий задачи
    """
    task_queue = multiprocessing.Queue()
    result_queue = multiprocessing.Queue()

    # Запускаем воркеров
    workers = [
        multiprocessing.Process(
            target=worker_process,
            args=(task_queue, result_queue)
        )
        for _ in range(num_workers)
    ]

    for w in workers:
        w.start()

    # Отправляем задачи
    tasks = list(range(20))
    for task in tasks:
        task_queue.put(task)

    # Отправляем сигналы завершения
    for _ in range(num_workers):
        task_queue.put(None)

    # Собираем результаты
    results = []
    for _ in range(len(tasks)):
        result = result_queue.get()
        results.append(result)
        print(f"Задача {result[0]} = {result[1]} (PID: {result[2]})")

    # Ждем завершения воркеров
    for w in workers:
        w.join()

    return results

if __name__ == "__main__":
    results = asyncio.run(master())
    print(f"Обработано {len(results)} задач")
```

## Использование uvloop для производительности

`uvloop` — это быстрая реализация event loop на основе libuv:

```python
# pip install uvloop

import asyncio
import uvloop

async def main():
    # Ваш асинхронный код
    await asyncio.sleep(1)
    return "Готово"

if __name__ == "__main__":
    # Установить uvloop как политику по умолчанию
    uvloop.install()

    # Теперь asyncio.run() использует uvloop
    result = asyncio.run(main())
    print(result)
```

### uvloop в многопроцессном приложении

```python
import asyncio
import multiprocessing

try:
    import uvloop
    USE_UVLOOP = True
except ImportError:
    USE_UVLOOP = False

async def async_task():
    await asyncio.sleep(0.1)
    return "done"

def process_entry():
    if USE_UVLOOP:
        uvloop.install()

    return asyncio.run(async_task())

if __name__ == "__main__":
    with multiprocessing.Pool(4) as pool:
        results = pool.map(lambda _: process_entry(), range(10))
    print(results)
```

## Потокобезопасные вызовы event loop

Для вызова корутин из других потоков используйте `asyncio.run_coroutine_threadsafe()`:

```python
import asyncio
import threading
import time

async def async_operation(value):
    await asyncio.sleep(1)
    return value * 2

def thread_function(loop, value):
    """Вызов корутины из другого потока"""
    # Планируем корутину в event loop
    future = asyncio.run_coroutine_threadsafe(
        async_operation(value),
        loop
    )

    # Ждем результат (блокирующий вызов)
    result = future.result(timeout=10)
    print(f"Thread {threading.current_thread().name}: результат = {result}")

async def main():
    loop = asyncio.get_event_loop()

    # Запускаем потоки, которые будут вызывать корутины
    threads = [
        threading.Thread(
            target=thread_function,
            args=(loop, i),
            name=f"Worker-{i}"
        )
        for i in range(5)
    ]

    for t in threads:
        t.start()

    # Event loop продолжает работать
    await asyncio.sleep(3)

    for t in threads:
        t.join()

if __name__ == "__main__":
    asyncio.run(main())
```

## Практический пример: Web-сервер с CPU-воркерами

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from aiohttp import web
import json

# CPU-интенсивная функция
def compute_heavy(data):
    """Тяжелые вычисления в отдельном процессе"""
    result = 0
    for i in range(data.get('iterations', 1000000)):
        result += i ** 0.5
    return {"result": result, "input": data}

# Создаем пул процессов глобально
process_pool = ProcessPoolExecutor(max_workers=4)

async def handle_compute(request):
    """Обработчик запроса с CPU-bound вычислениями"""
    data = await request.json()

    loop = asyncio.get_event_loop()

    # Выполняем CPU-bound задачу в процессе
    result = await loop.run_in_executor(
        process_pool,
        compute_heavy,
        data
    )

    return web.json_response(result)

async def handle_health(request):
    """Легкий I/O запрос"""
    return web.json_response({"status": "ok"})

def create_app():
    app = web.Application()
    app.router.add_post('/compute', handle_compute)
    app.router.add_get('/health', handle_health)
    return app

if __name__ == "__main__":
    app = create_app()
    web.run_app(app, port=8080)
```

## Паттерн: Actor Model с отдельными event loops

```python
import asyncio
import multiprocessing
from multiprocessing import Queue
import os

class Actor:
    """Актор с собственным event loop в процессе"""

    def __init__(self, name):
        self.name = name
        self.inbox = Queue()
        self.process = None

    def start(self):
        self.process = multiprocessing.Process(
            target=self._run,
            args=(self.inbox, self.name)
        )
        self.process.start()

    def send(self, message):
        self.inbox.put(message)

    def stop(self):
        self.inbox.put(None)
        self.process.join()

    @staticmethod
    def _run(inbox, name):
        async def handle_message(msg):
            # Обработка сообщения
            print(f"[{name}] Получил: {msg}")
            await asyncio.sleep(0.1)  # Имитация работы

        async def main():
            while True:
                try:
                    msg = inbox.get(timeout=0.1)
                    if msg is None:
                        break
                    await handle_message(msg)
                except:
                    await asyncio.sleep(0.01)

        asyncio.run(main())

if __name__ == "__main__":
    # Создаем акторы
    actors = [Actor(f"Actor-{i}") for i in range(3)]

    # Запускаем
    for actor in actors:
        actor.start()

    # Отправляем сообщения
    for i in range(10):
        actors[i % len(actors)].send(f"Сообщение {i}")

    import time
    time.sleep(2)

    # Останавливаем
    for actor in actors:
        actor.stop()

    print("Все акторы остановлены")
```

## Best Practices

1. **Один event loop на поток** — это базовое правило asyncio
2. **Используйте ProcessPoolExecutor** — для CPU-bound задач из asyncio
3. **run_coroutine_threadsafe** — для потокобезопасных вызовов
4. **uvloop для производительности** — значительное ускорение
5. **Изолируйте состояние** — каждый процесс должен иметь свои ресурсы

## Типичные ошибки

```python
# НЕПРАВИЛЬНО: использование одного loop из разных потоков
loop = asyncio.get_event_loop()

def bad_thread():
    loop.run_until_complete(some_coro())  # Опасно!

# ПРАВИЛЬНО: использование run_coroutine_threadsafe
def good_thread(loop):
    future = asyncio.run_coroutine_threadsafe(some_coro(), loop)
    result = future.result()
```

```python
# НЕПРАВИЛЬНО: забыли установить новый loop в потоке
def bad_worker():
    asyncio.run(some_coro())  # Может конфликтовать

# ПРАВИЛЬНО: явно создаем новый loop
def good_worker():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    try:
        loop.run_until_complete(some_coro())
    finally:
        loop.close()
```

---

[prev: 05-shared-data.md](./05-shared-data.md) | [next: ../07-threading/readme.md](../07-threading/readme.md)
