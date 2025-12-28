# Что такое asyncio?

[prev: ./readme.md](./readme.md) | [next: ./02-io-bound-cpu-bound.md](./02-io-bound-cpu-bound.md)

---

## Введение

**asyncio** - это стандартная библиотека Python для написания асинхронного кода с использованием синтаксиса `async`/`await`. Она была добавлена в Python 3.4 (2014 год) и значительно улучшена в последующих версиях.

Asyncio предоставляет инфраструктуру для:
- Написания однопоточного конкурентного кода с использованием корутин
- Мультиплексирования ввода-вывода через сокеты и другие ресурсы
- Запуска сетевых клиентов и серверов
- Работы с подпроцессами

## Зачем нужен asyncio?

### Проблема синхронного кода

Рассмотрим типичную ситуацию - нужно сделать 3 HTTP-запроса:

```python
import time
import requests

def fetch_url(url):
    """Синхронный запрос"""
    response = requests.get(url)
    return len(response.text)

def main_sync():
    urls = [
        "https://python.org",
        "https://pypi.org",
        "https://docs.python.org"
    ]

    start = time.time()

    for url in urls:
        size = fetch_url(url)
        print(f"{url}: {size} bytes")

    print(f"Общее время: {time.time() - start:.2f} сек")

# Каждый запрос выполняется последовательно
# Если каждый занимает 1 секунду, общее время = 3 секунды
```

### Решение с asyncio

```python
import asyncio
import aiohttp
import time

async def fetch_url_async(session, url):
    """Асинхронный запрос"""
    async with session.get(url) as response:
        text = await response.text()
        return len(text)

async def main_async():
    urls = [
        "https://python.org",
        "https://pypi.org",
        "https://docs.python.org"
    ]

    start = time.time()

    async with aiohttp.ClientSession() as session:
        # Запускаем все запросы конкурентно
        tasks = [fetch_url_async(session, url) for url in urls]
        results = await asyncio.gather(*tasks)

        for url, size in zip(urls, results):
            print(f"{url}: {size} bytes")

    print(f"Общее время: {time.time() - start:.2f} сек")

# Запросы выполняются конкурентно
# Общее время ≈ время самого медленного запроса (около 1 секунды)
asyncio.run(main_async())
```

## Основные концепции

### 1. Корутины (Coroutines)

Корутина - это функция, объявленная с помощью `async def`. Она может приостанавливать своё выполнение с помощью `await`:

```python
async def say_hello(name: str) -> str:
    """Это корутина"""
    await asyncio.sleep(1)  # Приостанавливаем выполнение на 1 секунду
    return f"Привет, {name}!"

# Вызов корутины НЕ выполняет её, а создаёт объект корутины
coro = say_hello("Мир")  # <coroutine object say_hello at 0x...>

# Для выполнения нужен event loop
result = asyncio.run(say_hello("Мир"))
print(result)  # Привет, Мир!
```

### 2. Задачи (Tasks)

Task - это обёртка над корутиной, которая позволяет запустить её в фоне:

```python
async def background_task(name: str, delay: int):
    print(f"Задача {name} начата")
    await asyncio.sleep(delay)
    print(f"Задача {name} завершена")
    return f"Результат {name}"

async def main():
    # Создаём задачи - они начинают выполняться сразу
    task1 = asyncio.create_task(background_task("A", 2))
    task2 = asyncio.create_task(background_task("B", 1))

    print("Задачи созданы, ждём завершения...")

    # Ждём завершения обеих задач
    result1 = await task1
    result2 = await task2

    print(f"Результаты: {result1}, {result2}")

asyncio.run(main())
```

Вывод:
```
Задача A начата
Задача B начата
Задачи созданы, ждём завершения...
Задача B завершена
Задача A завершена
Результаты: Результат A, Результат B
```

### 3. Event Loop (Цикл событий)

Event loop - это сердце asyncio. Он управляет выполнением корутин и переключает контекст между ними:

```python
import asyncio

async def my_coroutine():
    print("Старт")
    await asyncio.sleep(1)
    print("Финиш")

# Способ 1: asyncio.run() (рекомендуется, Python 3.7+)
asyncio.run(my_coroutine())

# Способ 2: Ручное управление (для особых случаев)
loop = asyncio.new_event_loop()
try:
    loop.run_until_complete(my_coroutine())
finally:
    loop.close()
```

## Ключевые слова async и await

### async def

Объявляет асинхронную функцию (корутину):

```python
# Обычная функция
def regular_function():
    return "Привет"

# Корутина
async def async_function():
    return "Привет асинхронно"

# Разница в типах возвращаемых значений
print(type(regular_function()))  # <class 'str'>
print(type(async_function()))    # <class 'coroutine'>
```

### await

Приостанавливает выполнение корутины до получения результата:

```python
async def get_data():
    await asyncio.sleep(1)  # Ждём 1 секунду
    return {"data": "value"}

async def process():
    # await можно использовать только внутри async def
    result = await get_data()
    print(result)
```

## Что можно await'ить?

Можно использовать `await` с:

1. **Корутинами** - функции с `async def`
2. **Tasks** - созданными через `asyncio.create_task()`
3. **Futures** - низкоуровневые awaitable объекты
4. **Любыми awaitable объектами** - имеющими метод `__await__`

```python
async def example():
    # Await корутины
    result = await some_coroutine()

    # Await задачи
    task = asyncio.create_task(some_coroutine())
    result = await task

    # Await нескольких задач
    results = await asyncio.gather(coro1(), coro2(), coro3())
```

## Частые ошибки начинающих

### 1. Забыли await

```python
async def fetch_data():
    await asyncio.sleep(1)
    return "data"

async def main():
    # НЕПРАВИЛЬНО - получаем объект корутины, не результат
    result = fetch_data()
    print(result)  # <coroutine object fetch_data at 0x...>

    # ПРАВИЛЬНО
    result = await fetch_data()
    print(result)  # "data"
```

### 2. Использование await вне async функции

```python
# НЕПРАВИЛЬНО - SyntaxError
def regular_function():
    await asyncio.sleep(1)

# ПРАВИЛЬНО
async def async_function():
    await asyncio.sleep(1)
```

### 3. Блокирующие вызовы в async коде

```python
import time

async def bad_example():
    # ПЛОХО - блокирует весь event loop!
    time.sleep(5)

async def good_example():
    # ХОРОШО - позволяет другим корутинам работать
    await asyncio.sleep(5)
```

## Когда использовать asyncio?

**Подходит для:**
- Сетевых операций (HTTP-запросы, WebSocket)
- Работы с базами данных (с async-драйверами)
- Файлового ввода-вывода (с aiofiles)
- Серверов и клиентов (FastAPI, aiohttp)
- Задач, связанных с ожиданием (I/O-bound)

**Не подходит для:**
- CPU-интенсивных вычислений (используйте multiprocessing)
- Простых скриптов без параллельных операций
- Когда все зависимости синхронные

## Заключение

Asyncio - мощный инструмент для написания эффективного конкурентного кода в Python. Он позволяет обрабатывать тысячи одновременных соединений в одном потоке, что делает его идеальным для сетевых приложений и серверов.

В следующих разделах мы подробнее рассмотрим, когда применять asyncio и как он работает под капотом.

---

[prev: ./readme.md](./readme.md) | [next: ./02-io-bound-cpu-bound.md](./02-io-bound-cpu-bound.md)
