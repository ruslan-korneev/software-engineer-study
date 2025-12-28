# I/O-bound vs CPU-bound

[prev: ./01-what-is-asyncio.md](./01-what-is-asyncio.md) | [next: ./03-concurrency-parallelism.md](./03-concurrency-parallelism.md)

---

## Введение

Понимание разницы между I/O-bound и CPU-bound задачами критически важно для выбора правильного подхода к конкурентности. Это определяет, какой инструмент использовать: asyncio, threading или multiprocessing.

## I/O-bound задачи

**I/O-bound (ограниченные вводом-выводом)** - это задачи, которые большую часть времени ждут внешних ресурсов:

- Сетевые запросы (HTTP, API, базы данных)
- Чтение/запись файлов
- Ввод от пользователя
- Обращения к внешним сервисам

### Характеристики I/O-bound:

```
Время выполнения:
|████░░░░░░░░░░░░░░░░| CPU работает
|░░░░████████████░░░░| Ожидание I/O
|░░░░░░░░░░░░░░░░████| CPU работает

Большая часть времени - ожидание!
```

### Пример I/O-bound задачи:

```python
import time
import requests

def fetch_user_data(user_id: int) -> dict:
    """
    I/O-bound: большую часть времени ждём ответа от сервера
    """
    print(f"Запрашиваем данные пользователя {user_id}...")

    # Отправляем запрос и ЖДЁМ ответа
    # CPU практически не работает во время ожидания
    response = requests.get(f"https://api.example.com/users/{user_id}")

    print(f"Получен ответ для пользователя {user_id}")
    return response.json()

def main():
    start = time.time()

    # Каждый запрос занимает ~500мс, но 95% этого времени - ожидание
    users = []
    for user_id in range(1, 6):
        users.append(fetch_user_data(user_id))

    print(f"Общее время: {time.time() - start:.2f} сек")
    # ~2.5 секунды, хотя CPU работал всего ~0.1 секунды
```

### I/O-bound с asyncio:

```python
import asyncio
import aiohttp
import time

async def fetch_user_data_async(session, user_id: int) -> dict:
    """
    Асинхронная версия - во время ожидания выполняются другие задачи
    """
    print(f"Запрашиваем данные пользователя {user_id}...")

    async with session.get(f"https://api.example.com/users/{user_id}") as response:
        data = await response.json()

    print(f"Получен ответ для пользователя {user_id}")
    return data

async def main():
    start = time.time()

    async with aiohttp.ClientSession() as session:
        # Все запросы выполняются конкурентно
        tasks = [fetch_user_data_async(session, i) for i in range(1, 6)]
        users = await asyncio.gather(*tasks)

    print(f"Общее время: {time.time() - start:.2f} сек")
    # ~0.5 секунды вместо 2.5!

asyncio.run(main())
```

## CPU-bound задачи

**CPU-bound (ограниченные процессором)** - это задачи, которые интенсивно используют CPU:

- Математические вычисления
- Обработка изображений/видео
- Шифрование/хеширование
- Сжатие данных
- Машинное обучение

### Характеристики CPU-bound:

```
Время выполнения:
|████████████████████| CPU работает постоянно
|░░░░░░░░░░░░░░░░░░░░| Нет ожидания I/O

Весь процесс - работа CPU!
```

### Пример CPU-bound задачи:

```python
import time
import math

def calculate_primes(limit: int) -> list:
    """
    CPU-bound: интенсивные вычисления без ожидания
    """
    primes = []
    for num in range(2, limit):
        is_prime = True
        for i in range(2, int(math.sqrt(num)) + 1):
            if num % i == 0:
                is_prime = False
                break
        if is_prime:
            primes.append(num)
    return primes

def fibonacci(n: int) -> int:
    """
    CPU-bound: рекурсивные вычисления
    """
    if n <= 1:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)

def process_image(image_data: bytes) -> bytes:
    """
    CPU-bound: обработка каждого пикселя
    """
    # Представим, что обрабатываем каждый пиксель
    result = bytearray()
    for pixel in image_data:
        # Применяем фильтр (CPU-интенсивная операция)
        new_pixel = (pixel * 2) % 256
        result.append(new_pixel)
    return bytes(result)

# Замер времени CPU-bound операции
start = time.time()
primes = calculate_primes(100000)
print(f"Найдено {len(primes)} простых чисел за {time.time() - start:.2f} сек")
```

## Почему asyncio НЕ ускоряет CPU-bound?

```python
import asyncio
import time

def cpu_intensive_work(n: int) -> int:
    """Симуляция CPU-bound задачи"""
    result = 0
    for i in range(n):
        result += i ** 2
    return result

async def async_cpu_work(n: int) -> int:
    """Обёртка в async НЕ делает код быстрее!"""
    return cpu_intensive_work(n)

async def main():
    start = time.time()

    # Это НЕ будет выполняться параллельно!
    # Каждая задача полностью блокирует event loop
    tasks = [async_cpu_work(10_000_000) for _ in range(4)]
    results = await asyncio.gather(*tasks)

    print(f"Результаты: {results}")
    print(f"Время: {time.time() - start:.2f} сек")
    # Время = сумма всех задач, не максимальное!

asyncio.run(main())
```

### Почему так происходит?

1. **Asyncio работает в одном потоке**
2. **await переключает контекст только на операциях ввода-вывода**
3. **CPU-bound код не содержит точек переключения**
4. **GIL блокирует параллельное выполнение Python-кода**

## Сравнительная таблица

| Характеристика | I/O-bound | CPU-bound |
|---------------|-----------|-----------|
| Узкое место | Внешние ресурсы | Процессор |
| Время ожидания | Высокое | Низкое |
| Использование CPU | Низкое | Высокое |
| **Решение** | **asyncio, threading** | **multiprocessing** |

## Как определить тип задачи?

### Индикаторы I/O-bound:
```python
# Признаки I/O-bound кода:
requests.get(url)           # Сетевой запрос
cursor.execute(query)       # Запрос к БД
file.read()                 # Чтение файла
socket.recv()               # Получение данных
time.sleep()                # Ожидание (симуляция I/O)
```

### Индикаторы CPU-bound:
```python
# Признаки CPU-bound кода:
for i in range(1_000_000):  # Много итераций
    result += complex_calculation(i)

np.dot(matrix1, matrix2)    # Матричные операции
hash_password(password)      # Криптография
compress(data)              # Сжатие данных
```

## Практический пример: смешанные задачи

```python
import asyncio
import aiohttp
from concurrent.futures import ProcessPoolExecutor

def cpu_heavy_processing(data: str) -> dict:
    """CPU-bound: анализ данных"""
    # Симуляция тяжёлой обработки
    word_count = len(data.split())
    char_count = len(data)
    unique_words = len(set(data.lower().split()))

    return {
        "words": word_count,
        "chars": char_count,
        "unique": unique_words
    }

async def fetch_and_process(session, url: str, executor) -> dict:
    """Комбинация I/O-bound и CPU-bound"""

    # I/O-bound: получаем данные (asyncio)
    async with session.get(url) as response:
        data = await response.text()

    # CPU-bound: обрабатываем в отдельном процессе
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(
        executor,
        cpu_heavy_processing,
        data
    )

    return {"url": url, "stats": result}

async def main():
    urls = [
        "https://python.org",
        "https://docs.python.org",
        "https://pypi.org"
    ]

    # Пул процессов для CPU-bound задач
    with ProcessPoolExecutor(max_workers=4) as executor:
        async with aiohttp.ClientSession() as session:
            tasks = [
                fetch_and_process(session, url, executor)
                for url in urls
            ]
            results = await asyncio.gather(*tasks)

    for result in results:
        print(f"{result['url']}: {result['stats']}")

asyncio.run(main())
```

## Рекомендации по выбору инструмента

### I/O-bound задачи:
```python
# Используйте asyncio
async def handle_io_bound():
    async with aiohttp.ClientSession() as session:
        # Сетевые запросы
        await session.get(url)

    async with aiofiles.open("file.txt") as f:
        # Асинхронный файловый I/O
        await f.read()
```

### CPU-bound задачи:
```python
from multiprocessing import Pool

def handle_cpu_bound():
    # Используйте multiprocessing для CPU-bound
    with Pool(processes=4) as pool:
        results = pool.map(cpu_intensive_function, data_list)
    return results
```

### Смешанные задачи:
```python
import asyncio
from concurrent.futures import ProcessPoolExecutor

async def handle_mixed():
    loop = asyncio.get_running_loop()

    with ProcessPoolExecutor() as executor:
        # CPU-bound в отдельном процессе
        cpu_result = await loop.run_in_executor(
            executor, cpu_function, args
        )

        # I/O-bound асинхронно
        io_result = await async_io_function()
```

## Best Practices

1. **Профилируйте код** - определите, где тратится время
2. **Не используйте asyncio для CPU-bound** - это не даст выигрыша
3. **Используйте правильный инструмент** - asyncio для I/O, multiprocessing для CPU
4. **Комбинируйте подходы** - используйте `run_in_executor` для CPU-bound в async коде
5. **Измеряйте результаты** - всегда проверяйте, даёт ли оптимизация эффект

## Заключение

Понимание разницы между I/O-bound и CPU-bound - ключ к эффективной конкурентности:

- **I/O-bound** = много ожидания = asyncio или threading
- **CPU-bound** = много вычислений = multiprocessing

Правильный выбор инструмента может ускорить вашу программу в разы!

---

[prev: ./01-what-is-asyncio.md](./01-what-is-asyncio.md) | [next: ./03-concurrency-parallelism.md](./03-concurrency-parallelism.md)
