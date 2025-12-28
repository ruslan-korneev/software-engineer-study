# Конкурентность с gather

[prev: ./03-concurrent-tasks.md](./03-concurrent-tasks.md) | [next: ./05-as-completed.md](./05-as-completed.md)

---

## Что такое asyncio.gather?

`asyncio.gather()` - это функция для запуска нескольких корутин или задач конкурентно и ожидания завершения всех из них. Это один из самых часто используемых инструментов для параллельного выполнения асинхронных операций.

```python
import asyncio

async def task1():
    await asyncio.sleep(1)
    return "Результат 1"

async def task2():
    await asyncio.sleep(2)
    return "Результат 2"

async def main():
    # Запускаем обе задачи конкурентно
    results = await asyncio.gather(task1(), task2())
    print(results)  # ['Результат 1', 'Результат 2']

asyncio.run(main())
```

## Сигнатура функции

```python
asyncio.gather(*aws, return_exceptions=False)
```

- `*aws` - произвольное количество awaitable объектов (корутин, задач, futures)
- `return_exceptions` - если `True`, исключения возвращаются как результаты вместо того, чтобы прерывать выполнение

## Особенности gather

### 1. Порядок результатов сохраняется

Результаты возвращаются в том же порядке, в каком были переданы корутины, независимо от порядка завершения:

```python
import asyncio
import random

async def fetch_with_delay(id: int):
    delay = random.uniform(0.1, 1.0)
    await asyncio.sleep(delay)
    return f"ID-{id} (задержка {delay:.2f}с)"

async def main():
    results = await asyncio.gather(
        fetch_with_delay(1),
        fetch_with_delay(2),
        fetch_with_delay(3),
    )

    for i, result in enumerate(results, 1):
        print(f"Результат {i}: {result}")

    # Вывод (порядок сохраняется!):
    # Результат 1: ID-1 (задержка 0.83с)
    # Результат 2: ID-2 (задержка 0.21с)
    # Результат 3: ID-3 (задержка 0.56с)

asyncio.run(main())
```

### 2. Ожидание всех задач

`gather` ждет завершения ВСЕХ задач перед возвратом:

```python
import asyncio
import time

async def slow_task():
    await asyncio.sleep(3)
    return "Медленная задача"

async def fast_task():
    await asyncio.sleep(0.5)
    return "Быстрая задача"

async def main():
    start = time.time()

    results = await asyncio.gather(
        slow_task(),
        fast_task(),
    )

    print(f"Все задачи завершены за {time.time() - start:.2f}с")
    print(results)

asyncio.run(main())

# Вывод:
# Все задачи завершены за 3.00с
# ['Медленная задача', 'Быстрая задача']
```

## Обработка исключений

### Поведение по умолчанию (return_exceptions=False)

По умолчанию, если одна из задач выбрасывает исключение, `gather` немедленно прекращает ожидание и пробрасывает это исключение:

```python
import asyncio

async def success_task():
    await asyncio.sleep(1)
    return "Успех"

async def failing_task():
    await asyncio.sleep(0.5)
    raise ValueError("Что-то пошло не так!")

async def main():
    try:
        results = await asyncio.gather(
            success_task(),
            failing_task(),
        )
    except ValueError as e:
        print(f"Ошибка: {e}")

asyncio.run(main())

# Вывод:
# Ошибка: Что-то пошло не так!
```

**Важно**: Остальные задачи НЕ отменяются автоматически! Они продолжают выполняться в фоне.

### С return_exceptions=True

Когда `return_exceptions=True`, исключения возвращаются как часть результатов:

```python
import asyncio

async def success_task(id: int):
    await asyncio.sleep(0.5)
    return f"Успех {id}"

async def failing_task():
    await asyncio.sleep(0.3)
    raise ValueError("Ошибка!")

async def main():
    results = await asyncio.gather(
        success_task(1),
        failing_task(),
        success_task(2),
        return_exceptions=True  # Исключения в результатах
    )

    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Задача {i}: ОШИБКА - {result}")
        else:
            print(f"Задача {i}: {result}")

asyncio.run(main())

# Вывод:
# Задача 0: Успех 1
# Задача 1: ОШИБКА - Ошибка!
# Задача 2: Успех 2
```

## Практический пример с aiohttp

### Базовый пример: загрузка нескольких URL

```python
import aiohttp
import asyncio

async def fetch(session: aiohttp.ClientSession, url: str) -> dict:
    async with session.get(url) as response:
        return {
            'url': url,
            'status': response.status,
            'data': await response.json()
        }

async def main():
    urls = [
        'https://api.github.com/users/octocat',
        'https://api.github.com/users/torvalds',
        'https://api.github.com/users/gvanrossum',
    ]

    async with aiohttp.ClientSession() as session:
        # Создаем корутины для всех URL
        coroutines = [fetch(session, url) for url in urls]

        # Запускаем все конкурентно
        results = await asyncio.gather(*coroutines)

        for result in results:
            print(f"{result['url']}: {result['status']}")

asyncio.run(main())
```

### Пример с обработкой ошибок

```python
import aiohttp
import asyncio
from typing import Union, Dict, Any

async def fetch_safe(
    session: aiohttp.ClientSession,
    url: str
) -> Dict[str, Any]:
    """Безопасный запрос с обработкой ошибок"""
    try:
        async with session.get(url) as response:
            response.raise_for_status()
            return {
                'url': url,
                'status': response.status,
                'data': await response.json(),
                'error': None
            }
    except aiohttp.ClientError as e:
        return {
            'url': url,
            'status': None,
            'data': None,
            'error': str(e)
        }

async def main():
    urls = [
        'https://api.github.com/users/octocat',
        'https://httpbin.org/status/404',  # Ошибка 404
        'https://api.github.com/users/torvalds',
        'https://nonexistent.example.com',  # Несуществующий домен
    ]

    async with aiohttp.ClientSession() as session:
        results = await asyncio.gather(
            *[fetch_safe(session, url) for url in urls]
        )

        for result in results:
            if result['error']:
                print(f"ОШИБКА: {result['url']} - {result['error']}")
            else:
                print(f"OK: {result['url']} - {result['status']}")

asyncio.run(main())
```

## Динамическое количество задач

Часто количество задач неизвестно заранее:

```python
import aiohttp
import asyncio

async def fetch_user(session: aiohttp.ClientSession, username: str) -> dict:
    url = f'https://api.github.com/users/{username}'
    async with session.get(url) as response:
        return await response.json()

async def fetch_all_users(usernames: list[str]) -> list[dict]:
    async with aiohttp.ClientSession() as session:
        # Динамически создаем задачи для всех пользователей
        tasks = [fetch_user(session, username) for username in usernames]
        return await asyncio.gather(*tasks)

async def main():
    usernames = ['octocat', 'torvalds', 'gvanrossum', 'kennethreitz']

    users = await fetch_all_users(usernames)

    for user in users:
        print(f"{user['login']}: {user['public_repos']} репозиториев")

asyncio.run(main())
```

## Вложенные gather

Можно комбинировать несколько `gather`:

```python
import asyncio

async def api_call(name: str, delay: float):
    await asyncio.sleep(delay)
    return f"Результат {name}"

async def batch1():
    return await asyncio.gather(
        api_call("A1", 1),
        api_call("A2", 1),
    )

async def batch2():
    return await asyncio.gather(
        api_call("B1", 0.5),
        api_call("B2", 0.5),
    )

async def main():
    # Запускаем оба batch конкурентно
    results = await asyncio.gather(
        batch1(),
        batch2(),
    )

    print("Batch 1:", results[0])
    print("Batch 2:", results[1])

asyncio.run(main())

# Вывод:
# Batch 1: ['Результат A1', 'Результат A2']
# Batch 2: ['Результат B1', 'Результат B2']
```

## Ограничение конкурентности с gather

`gather` сам по себе не ограничивает количество одновременных задач. Используйте семафор:

```python
import aiohttp
import asyncio

async def fetch_with_limit(
    session: aiohttp.ClientSession,
    url: str,
    semaphore: asyncio.Semaphore
) -> dict:
    async with semaphore:
        async with session.get(url) as response:
            data = await response.json()
            return {'url': url, 'data': data}

async def fetch_all_limited(urls: list[str], max_concurrent: int = 10) -> list[dict]:
    semaphore = asyncio.Semaphore(max_concurrent)

    async with aiohttp.ClientSession() as session:
        tasks = [
            fetch_with_limit(session, url, semaphore)
            for url in urls
        ]
        return await asyncio.gather(*tasks)

async def main():
    # 100 URL, но только 10 одновременных запросов
    urls = [f'https://httpbin.org/get?id={i}' for i in range(100)]
    results = await fetch_all_limited(urls, max_concurrent=10)
    print(f"Загружено {len(results)} URL")

asyncio.run(main())
```

## Отмена gather

При отмене `gather` все задачи внутри также отменяются:

```python
import asyncio

async def long_task(id: int):
    try:
        await asyncio.sleep(10)
        return f"Задача {id} завершена"
    except asyncio.CancelledError:
        print(f"Задача {id} отменена")
        raise

async def main():
    # Создаем gather как Task
    gather_task = asyncio.create_task(
        asyncio.gather(
            long_task(1),
            long_task(2),
            long_task(3),
        )
    )

    await asyncio.sleep(1)
    gather_task.cancel()  # Отменяем все задачи

    try:
        await gather_task
    except asyncio.CancelledError:
        print("Все задачи отменены")

asyncio.run(main())

# Вывод:
# Задача 1 отменена
# Задача 2 отменена
# Задача 3 отменена
# Все задачи отменены
```

## gather vs create_task

```python
import asyncio

async def example():
    # Способ 1: gather с корутинами (рекомендуется для простых случаев)
    results = await asyncio.gather(
        coroutine1(),
        coroutine2(),
    )

    # Способ 2: create_task + gather (когда нужен контроль над задачами)
    task1 = asyncio.create_task(coroutine1())
    task2 = asyncio.create_task(coroutine2())
    results = await asyncio.gather(task1, task2)

    # Способ 3: create_task для доступа к задачам
    task1 = asyncio.create_task(coroutine1())
    task2 = asyncio.create_task(coroutine2())

    # Можно отменить конкретную задачу
    task1.cancel()

    results = await asyncio.gather(task1, task2, return_exceptions=True)
```

## Практический пример: Web Scraper

```python
import aiohttp
import asyncio
from dataclasses import dataclass
from typing import Optional

@dataclass
class PageResult:
    url: str
    status: int
    title: Optional[str]
    error: Optional[str]

async def fetch_page(
    session: aiohttp.ClientSession,
    url: str,
    semaphore: asyncio.Semaphore
) -> PageResult:
    async with semaphore:
        try:
            async with session.get(url, timeout=aiohttp.ClientTimeout(total=10)) as response:
                html = await response.text()
                # Простой парсинг title (в реальности используйте BeautifulSoup)
                title = None
                if '<title>' in html:
                    start = html.find('<title>') + 7
                    end = html.find('</title>')
                    title = html[start:end].strip()

                return PageResult(
                    url=url,
                    status=response.status,
                    title=title,
                    error=None
                )
        except Exception as e:
            return PageResult(
                url=url,
                status=0,
                title=None,
                error=str(e)
            )

async def scrape_urls(urls: list[str], max_concurrent: int = 5) -> list[PageResult]:
    semaphore = asyncio.Semaphore(max_concurrent)

    async with aiohttp.ClientSession() as session:
        tasks = [fetch_page(session, url, semaphore) for url in urls]
        return await asyncio.gather(*tasks)

async def main():
    urls = [
        'https://www.python.org',
        'https://www.github.com',
        'https://httpbin.org/html',
        'https://nonexistent.example.com',
    ]

    results = await scrape_urls(urls)

    for result in results:
        if result.error:
            print(f"ОШИБКА: {result.url} - {result.error}")
        else:
            print(f"OK: {result.url} - '{result.title}' (статус {result.status})")

asyncio.run(main())
```

## Лучшие практики

1. **Используйте `return_exceptions=True`** для устойчивости к ошибкам
2. **Ограничивайте конкурентность** семафором для большого количества задач
3. **Всегда обрабатывайте исключения** в результатах
4. **Группируйте связанные задачи** логически
5. **Помните о порядке результатов** - он соответствует порядку входных данных

## Частые ошибки

```python
# ОШИБКА: Забыли распаковку *
results = await asyncio.gather(tasks)  # tasks - это список!

# ПРАВИЛЬНО:
results = await asyncio.gather(*tasks)

# ОШИБКА: Не проверяем исключения при return_exceptions=True
results = await asyncio.gather(*tasks, return_exceptions=True)
for result in results:
    print(result)  # Может быть Exception!

# ПРАВИЛЬНО:
results = await asyncio.gather(*tasks, return_exceptions=True)
for result in results:
    if isinstance(result, Exception):
        print(f"Ошибка: {result}")
    else:
        print(f"Успех: {result}")

# ОШИБКА: Слишком много одновременных задач
tasks = [fetch(url) for url in thousands_of_urls]
await asyncio.gather(*tasks)  # Может перегрузить систему

# ПРАВИЛЬНО: используйте семафор (см. примеры выше)
```

---

[prev: ./03-concurrent-tasks.md](./03-concurrent-tasks.md) | [next: ./05-as-completed.md](./05-as-completed.md)
