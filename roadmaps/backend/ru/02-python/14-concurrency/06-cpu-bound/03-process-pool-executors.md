# ProcessPoolExecutor с asyncio

[prev: 02-process-pools.md](./02-process-pools.md) | [next: 04-map-reduce.md](./04-map-reduce.md)

---

## Введение в concurrent.futures

Модуль `concurrent.futures` предоставляет высокоуровневый интерфейс для асинхронного выполнения задач. Он был введен в Python 3.2 и предоставляет унифицированный API для работы как с потоками, так и с процессами.

### Два типа Executor

```python
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

# Для I/O-bound задач (сетевые запросы, файлы)
with ThreadPoolExecutor(max_workers=10) as executor:
    pass

# Для CPU-bound задач (вычисления)
with ProcessPoolExecutor(max_workers=4) as executor:
    pass
```

## ProcessPoolExecutor

`ProcessPoolExecutor` — это обертка над `multiprocessing.Pool`, предоставляющая современный API с поддержкой `Future` объектов.

### Базовое использование

```python
from concurrent.futures import ProcessPoolExecutor
import time

def compute_factorial(n):
    """Вычисление факториала"""
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result

if __name__ == "__main__":
    numbers = [100000, 110000, 120000, 130000]

    with ProcessPoolExecutor(max_workers=4) as executor:
        # submit() отправляет одну задачу
        future = executor.submit(compute_factorial, 1000)
        result = future.result()
        print(f"1000! имеет {len(str(result))} цифр")
```

## Метод submit() и объект Future

`submit()` возвращает объект `Future`, который представляет отложенное вычисление:

```python
from concurrent.futures import ProcessPoolExecutor
import time

def slow_compute(x):
    time.sleep(1)
    return x ** 2

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as executor:
        # Отправляем задачи
        futures = []
        for i in range(8):
            future = executor.submit(slow_compute, i)
            futures.append(future)

        # Работаем с futures
        for f in futures:
            # done() - проверка завершения (не блокирует)
            print(f"Завершено: {f.done()}")

            # result() - получение результата (блокирует)
            result = f.result(timeout=10)
            print(f"Результат: {result}")
```

### Методы объекта Future

```python
from concurrent.futures import ProcessPoolExecutor
import time

def task(x):
    time.sleep(2)
    return x * 2

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=2) as executor:
        future = executor.submit(task, 10)

        # Проверка статуса
        print(f"Запущено: {future.running()}")
        print(f"Завершено: {future.done()}")
        print(f"Отменено: {future.cancelled()}")

        # Попытка отмены (может не сработать если уже выполняется)
        cancelled = future.cancel()
        print(f"Отмена успешна: {cancelled}")

        # Ожидание результата с таймаутом
        try:
            result = future.result(timeout=5)
            print(f"Результат: {result}")
        except TimeoutError:
            print("Таймаут!")
```

## Метод map()

`map()` в `ProcessPoolExecutor` аналогичен `Pool.map()`, но возвращает итератор:

```python
from concurrent.futures import ProcessPoolExecutor
import time

def process(x):
    time.sleep(0.5)
    return x ** 2

if __name__ == "__main__":
    items = list(range(12))

    with ProcessPoolExecutor(max_workers=4) as executor:
        # map() возвращает итератор результатов
        results = executor.map(process, items)

        # Результаты в порядке входных данных
        for item, result in zip(items, results):
            print(f"{item}^2 = {result}")
```

### map() с несколькими итерируемыми

```python
from concurrent.futures import ProcessPoolExecutor

def add(a, b):
    return a + b

if __name__ == "__main__":
    with ProcessPoolExecutor() as executor:
        list_a = [1, 2, 3, 4, 5]
        list_b = [10, 20, 30, 40, 50]

        # map() с несколькими итерируемыми
        results = executor.map(add, list_a, list_b)
        print(list(results))  # [11, 22, 33, 44, 55]
```

## Функции as_completed() и wait()

### as_completed()

Возвращает итератор, который выдает `Future` объекты **по мере их завершения**:

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import time
import random

def variable_task(x):
    """Задача с переменным временем выполнения"""
    sleep_time = random.uniform(0.1, 2.0)
    time.sleep(sleep_time)
    return x, sleep_time

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as executor:
        # Создаем словарь future -> входные данные
        future_to_input = {
            executor.submit(variable_task, i): i
            for i in range(10)
        }

        # Получаем результаты по мере готовности
        for future in as_completed(future_to_input):
            input_value = future_to_input[future]
            try:
                result, duration = future.result()
                print(f"Задача {input_value}: результат={result}, время={duration:.2f}с")
            except Exception as e:
                print(f"Задача {input_value} завершилась с ошибкой: {e}")
```

### wait()

Ожидает завершения группы `Future` объектов:

```python
from concurrent.futures import ProcessPoolExecutor, wait, FIRST_COMPLETED, ALL_COMPLETED
import time

def task(x):
    time.sleep(x)
    return x * 2

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as executor:
        futures = [executor.submit(task, i) for i in [3, 1, 2, 4]]

        # Ждать первого завершенного
        done, not_done = wait(futures, return_when=FIRST_COMPLETED)
        print(f"Первый завершенный: {done.pop().result()}")
        print(f"Еще не завершены: {len(not_done)}")

        # Ждать все
        done, not_done = wait(futures, return_when=ALL_COMPLETED)
        print(f"Все завершены: {len(done)}")
```

## Интеграция с asyncio

### run_in_executor()

`asyncio` позволяет выполнять синхронные функции в отдельных процессах через `run_in_executor()`:

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import time

def cpu_bound_task(n):
    """CPU-интенсивная задача"""
    result = 0
    for i in range(n):
        result += i ** 2
    return result

async def main():
    loop = asyncio.get_event_loop()

    # Создаем пул процессов
    with ProcessPoolExecutor(max_workers=4) as executor:
        # Запускаем CPU-bound задачи в процессах
        tasks = [
            loop.run_in_executor(executor, cpu_bound_task, 5_000_000)
            for _ in range(4)
        ]

        # Параллельно ждем все результаты
        results = await asyncio.gather(*tasks)
        print(f"Результаты: {results}")

if __name__ == "__main__":
    asyncio.run(main())
```

### Комбинирование I/O и CPU задач

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import aiohttp

def process_data(data):
    """CPU-bound обработка данных"""
    # Имитация CPU-интенсивной обработки
    result = sum(ord(c) ** 2 for c in data * 1000)
    return result

async def fetch_and_process(session, url, executor, loop):
    """I/O-bound загрузка + CPU-bound обработка"""
    # Асинхронная загрузка (I/O-bound)
    async with session.get(url) as response:
        data = await response.text()

    # CPU-bound обработка в отдельном процессе
    result = await loop.run_in_executor(executor, process_data, data)
    return url, result

async def main():
    urls = [
        "https://httpbin.org/html",
        "https://httpbin.org/robots.txt",
        "https://httpbin.org/forms/post",
    ]

    loop = asyncio.get_event_loop()

    with ProcessPoolExecutor(max_workers=4) as executor:
        async with aiohttp.ClientSession() as session:
            tasks = [
                fetch_and_process(session, url, executor, loop)
                for url in urls
            ]
            results = await asyncio.gather(*tasks)

            for url, result in results:
                print(f"{url}: {result}")

if __name__ == "__main__":
    asyncio.run(main())
```

## Обработка исключений

```python
from concurrent.futures import ProcessPoolExecutor, as_completed

def might_fail(x):
    if x % 3 == 0:
        raise ValueError(f"Ошибка для {x}")
    return x ** 2

if __name__ == "__main__":
    with ProcessPoolExecutor() as executor:
        futures = {executor.submit(might_fail, i): i for i in range(10)}

        for future in as_completed(futures):
            input_val = futures[future]
            try:
                result = future.result()
                print(f"{input_val}: {result}")
            except ValueError as e:
                print(f"{input_val}: ОШИБКА - {e}")
```

### Получение исключения без его выброса

```python
from concurrent.futures import ProcessPoolExecutor

def failing_task():
    raise RuntimeError("Что-то пошло не так")

if __name__ == "__main__":
    with ProcessPoolExecutor() as executor:
        future = executor.submit(failing_task)

        # Получить исключение без его выброса
        exception = future.exception(timeout=5)
        if exception:
            print(f"Задача завершилась с исключением: {exception}")
```

## Callbacks

Можно добавить функцию обратного вызова, которая выполнится после завершения `Future`:

```python
from concurrent.futures import ProcessPoolExecutor
import time

def compute(x):
    time.sleep(1)
    return x ** 2

def on_complete(future):
    """Callback функция"""
    print(f"Задача завершена! Результат: {future.result()}")

if __name__ == "__main__":
    with ProcessPoolExecutor() as executor:
        future = executor.submit(compute, 5)

        # Добавляем callback
        future.add_done_callback(on_complete)

        print("Ждем завершения...")
        time.sleep(2)
```

**Важно**: Callback выполняется в **главном процессе**, а не в worker-процессе.

## Практический пример: параллельное хеширование файлов

```python
from concurrent.futures import ProcessPoolExecutor, as_completed
import hashlib
import os

def hash_file(filepath):
    """Вычисляет SHA-256 хеш файла"""
    sha256 = hashlib.sha256()
    with open(filepath, 'rb') as f:
        while chunk := f.read(8192):
            sha256.update(chunk)
    return filepath, sha256.hexdigest()

def hash_directory(directory, max_workers=None):
    """Параллельно хеширует все файлы в директории"""
    files = []
    for root, _, filenames in os.walk(directory):
        for filename in filenames:
            files.append(os.path.join(root, filename))

    results = {}
    with ProcessPoolExecutor(max_workers=max_workers) as executor:
        future_to_file = {
            executor.submit(hash_file, f): f
            for f in files
        }

        for future in as_completed(future_to_file):
            try:
                filepath, file_hash = future.result()
                results[filepath] = file_hash
                print(f"Обработан: {filepath}")
            except Exception as e:
                filepath = future_to_file[future]
                print(f"Ошибка {filepath}: {e}")

    return results

if __name__ == "__main__":
    hashes = hash_directory("/path/to/directory")
    print(f"Обработано {len(hashes)} файлов")
```

## Сравнение Pool и ProcessPoolExecutor

| Характеристика | multiprocessing.Pool | ProcessPoolExecutor |
|----------------|---------------------|---------------------|
| API | Низкоуровневый | Высокоуровневый |
| Future объекты | Нет (AsyncResult) | Да |
| Интеграция с asyncio | Сложнее | Через run_in_executor |
| Callbacks | Нет | Да |
| as_completed | Нет | Да |
| Единый API с потоками | Нет | Да |
| imap/imap_unordered | Да | Нет |

## Best Practices

1. **Используйте ProcessPoolExecutor для CPU-bound задач** — это современный подход
2. **Используйте менеджер контекста** — гарантирует корректное завершение
3. **Комбинируйте с asyncio** — для приложений с I/O и CPU задачами
4. **Используйте as_completed()** — для обработки результатов по мере готовности
5. **Обрабатывайте исключения** — через try/except при вызове result()
6. **Ограничивайте max_workers** — обычно = количеству ядер CPU

## Типичные ошибки

```python
# НЕПРАВИЛЬНО: создание executor внутри цикла
for item in items:
    with ProcessPoolExecutor() as executor:  # Создается заново каждый раз!
        executor.submit(task, item)

# ПРАВИЛЬНО: один executor для всех задач
with ProcessPoolExecutor() as executor:
    for item in items:
        executor.submit(task, item)
```

```python
# НЕПРАВИЛЬНО: не дождались результатов
with ProcessPoolExecutor() as executor:
    for i in range(10):
        executor.submit(task, i)
# Executor закрывается, процессы убиваются!

# ПРАВИЛЬНО: ждем результаты
with ProcessPoolExecutor() as executor:
    futures = [executor.submit(task, i) for i in range(10)]
    for f in as_completed(futures):
        result = f.result()
```

---

[prev: 02-process-pools.md](./02-process-pools.md) | [next: 04-map-reduce.md](./04-map-reduce.md)
