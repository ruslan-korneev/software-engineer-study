# Asyncio (Асинхронное программирование)

## Что это?

Кооперативная многозадачность — один поток, но задачи отдают управление при I/O.

---

## Основы

```python
import asyncio

async def main():
    print("Hello")
    await asyncio.sleep(1)
    print("World")

asyncio.run(main())
```

---

## Параллельное выполнение

### gather

```python
async def fetch(url):
    await asyncio.sleep(1)
    return f"Результат {url}"

async def main():
    results = await asyncio.gather(
        fetch("url1"),
        fetch("url2"),
        fetch("url3"),
    )
    # ~1 сек, не 3
```

### create_task

```python
async def main():
    task1 = asyncio.create_task(fetch("url1"))
    task2 = asyncio.create_task(fetch("url2"))

    result1 = await task1
    result2 = await task2
```

### TaskGroup (3.11+)

```python
async with asyncio.TaskGroup() as tg:
    task1 = tg.create_task(fetch("url1"))
    task2 = tg.create_task(fetch("url2"))
```

---

## httpx — async HTTP

```python
import httpx

async def main():
    async with httpx.AsyncClient() as client:
        response = await client.get("https://example.com")
        print(response.status_code)
```

---

## Таймауты

```python
try:
    await asyncio.wait_for(task(), timeout=2.0)
except asyncio.TimeoutError:
    print("Таймаут!")

# Python 3.11+
async with asyncio.timeout(2.0):
    await task()
```

---

## Семафор

```python
sem = asyncio.Semaphore(5)  # Макс 5 одновременно

async def limited_fetch(url):
    async with sem:
        return await fetch(url)
```

---

## Queue

```python
queue = asyncio.Queue()
await queue.put(item)
item = await queue.get()
```

---

## Синхронный код в async

```python
# В thread pool
result = await asyncio.to_thread(blocking_function)
```

---

## Rate Limiting (ограничение запросов)

### Semaphore — только параллельность

```python
sem = asyncio.Semaphore(10)  # Макс 10 одновременно
# НЕ ограничивает "100 запросов в минуту"
```

### aiolimiter — запросы в единицу времени

```bash
pip install aiolimiter
```

```python
from aiolimiter import AsyncLimiter

# 100 запросов в 60 секунд
limiter = AsyncLimiter(100, 60)

async def fetch(client, url):
    async with limiter:
        return await client.get(url)
```

### Комбинация: rate-limit + параллельность

```python
from aiolimiter import AsyncLimiter

rate_limiter = AsyncLimiter(100, 60)  # 100/мин
semaphore = asyncio.Semaphore(10)     # Макс 10 одновременно

async def fetch(client, url):
    async with semaphore:
        async with rate_limiter:
            return await client.get(url)
```

### Обработка 429 (Too Many Requests)

```python
async def fetch_with_retry(client, url, max_retries=5):
    for attempt in range(max_retries):
        response = await client.get(url)
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", 2 ** attempt))
            await asyncio.sleep(retry_after)
            continue
        return response
    raise Exception("Rate limit exceeded")
```

### Полный пример

```python
from aiolimiter import AsyncLimiter
import asyncio
import httpx

class APIClient:
    def __init__(self, base_url: str, rate: int = 100, per: int = 60):
        self.base_url = base_url
        self.limiter = AsyncLimiter(rate, per)
        self.sem = asyncio.Semaphore(10)

    async def get(self, endpoint: str):
        async with self.sem, self.limiter:
            async with httpx.AsyncClient() as client:
                response = await client.get(f"{self.base_url}{endpoint}")
                if response.status_code == 429:
                    await asyncio.sleep(int(response.headers.get("Retry-After", 5)))
                    return await self.get(endpoint)
                return response.json()
```

---

## Резюме

| Функция | Назначение |
|---------|-----------|
| `asyncio.run()` | Точка входа |
| `await` | Ожидание корутины |
| `asyncio.gather()` | Параллельный запуск |
| `asyncio.create_task()` | Создание задачи |
| `asyncio.TaskGroup` | Группа задач (3.11+) |
| `asyncio.wait_for()` | Таймаут |
| `asyncio.Semaphore` | Ограничение параллельности |
| `asyncio.to_thread()` | Синхронный код в async |
| `aiolimiter` | Rate limiting (запросы/время) |

Используй для **I/O-bound** с большим количеством операций.
