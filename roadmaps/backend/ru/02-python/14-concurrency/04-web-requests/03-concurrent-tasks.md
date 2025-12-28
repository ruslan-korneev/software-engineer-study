# О конкурентном выполнении задач

[prev: ./02-context-managers.md](./02-context-managers.md) | [next: ./04-gather.md](./04-gather.md)

---

## Введение в конкурентность

**Конкурентность** (concurrency) - это способность программы работать с несколькими задачами одновременно. В контексте asyncio это означает, что пока одна задача ожидает I/O операцию (например, ответ от сервера), другие задачи могут выполняться.

Важно понимать разницу:
- **Параллелизм** - задачи буквально выполняются одновременно (на разных ядрах CPU)
- **Конкурентность** - задачи могут перекрываться во времени, переключаясь между собой

asyncio обеспечивает конкурентность в одном потоке, что идеально подходит для I/O-bound задач, таких как HTTP-запросы.

## Проблема последовательного выполнения

Рассмотрим последовательный подход:

```python
import aiohttp
import asyncio
import time

async def fetch(session, url):
    async with session.get(url) as response:
        return await response.json()

async def sequential():
    """Последовательное выполнение - МЕДЛЕННО"""
    start = time.time()
    urls = [
        'https://httpbin.org/delay/1',
        'https://httpbin.org/delay/1',
        'https://httpbin.org/delay/1',
    ]

    async with aiohttp.ClientSession() as session:
        results = []
        for url in urls:
            result = await fetch(session, url)  # Ждем каждый запрос
            results.append(result)

    print(f"Последовательно: {time.time() - start:.2f} сек")  # ~3 секунды
    return results

asyncio.run(sequential())
```

Каждый запрос занимает 1 секунду, и они выполняются последовательно. Общее время: ~3 секунды.

## Решение: конкурентное выполнение

```python
async def concurrent():
    """Конкурентное выполнение - БЫСТРО"""
    start = time.time()
    urls = [
        'https://httpbin.org/delay/1',
        'https://httpbin.org/delay/1',
        'https://httpbin.org/delay/1',
    ]

    async with aiohttp.ClientSession() as session:
        # Создаем корутины (еще не запущены)
        tasks = [fetch(session, url) for url in urls]
        # Запускаем все конкурентно
        results = await asyncio.gather(*tasks)

    print(f"Конкурентно: {time.time() - start:.2f} сек")  # ~1 секунда
    return results

asyncio.run(concurrent())
```

Все три запроса выполняются одновременно. Общее время: ~1 секунда.

## Способы создания задач

### 1. Корутины (Coroutines)

Корутина - это функция, объявленная с `async def`. Сама по себе она не выполняется, пока вы не вызовете `await` или не обернете в Task.

```python
async def my_coroutine():
    await asyncio.sleep(1)
    return "результат"

# Это создает объект корутины, но не запускает её
coro = my_coroutine()

# Для выполнения нужен await
result = await coro
```

### 2. Tasks

`Task` - это обертка над корутиной, которая планирует её выполнение в event loop.

```python
import asyncio

async def worker(name, delay):
    print(f"{name} начал работу")
    await asyncio.sleep(delay)
    print(f"{name} завершил работу")
    return f"Результат от {name}"

async def main():
    # Создаем задачи - они СРАЗУ начинают выполняться!
    task1 = asyncio.create_task(worker("Task-1", 2))
    task2 = asyncio.create_task(worker("Task-2", 1))

    print("Задачи созданы и выполняются в фоне")

    # Можно делать что-то еще
    await asyncio.sleep(0.5)
    print("Прошло 0.5 сек")

    # Ждем завершения задач
    result1 = await task1
    result2 = await task2

    print(result1, result2)

asyncio.run(main())
```

Вывод:
```
Task-1 начал работу
Task-2 начал работу
Задачи созданы и выполняются в фоне
Прошло 0.5 сек
Task-2 завершил работу
Task-1 завершил работу
Результат от Task-1 Результат от Task-2
```

### 3. Разница между корутинами и Tasks

```python
import asyncio

async def slow_operation():
    await asyncio.sleep(1)
    return "done"

async def with_coroutines():
    """Корутины выполняются ПОСЛЕДОВАТЕЛЬНО"""
    start = time.time()

    # Это НЕ запускает корутины параллельно!
    coro1 = slow_operation()
    coro2 = slow_operation()

    result1 = await coro1  # Ждем 1 секунду
    result2 = await coro2  # Ждем еще 1 секунду

    print(f"Корутины: {time.time() - start:.2f} сек")  # ~2 секунды

async def with_tasks():
    """Tasks выполняются КОНКУРЕНТНО"""
    start = time.time()

    # Создание Task сразу планирует выполнение!
    task1 = asyncio.create_task(slow_operation())
    task2 = asyncio.create_task(slow_operation())

    result1 = await task1
    result2 = await task2

    print(f"Tasks: {time.time() - start:.2f} сек")  # ~1 секунда
```

## Практический пример: загрузка нескольких URL

```python
import aiohttp
import asyncio
from typing import List, Dict, Any

async def fetch_url(session: aiohttp.ClientSession, url: str) -> Dict[str, Any]:
    """Загружает один URL и возвращает результат с метаданными"""
    try:
        async with session.get(url) as response:
            data = await response.json()
            return {
                'url': url,
                'status': response.status,
                'data': data,
                'error': None
            }
    except Exception as e:
        return {
            'url': url,
            'status': None,
            'data': None,
            'error': str(e)
        }

async def fetch_all(urls: List[str]) -> List[Dict[str, Any]]:
    """Загружает все URL конкурентно"""
    async with aiohttp.ClientSession() as session:
        # Создаем задачи для всех URL
        tasks = [
            asyncio.create_task(fetch_url(session, url))
            for url in urls
        ]

        # Ждем завершения всех задач
        results = await asyncio.gather(*tasks)

    return results

async def main():
    urls = [
        'https://api.github.com/users/octocat',
        'https://api.github.com/users/torvalds',
        'https://api.github.com/users/gvanrossum',
        'https://httpbin.org/status/404',  # Ошибка 404
    ]

    results = await fetch_all(urls)

    for result in results:
        if result['error']:
            print(f"ОШИБКА: {result['url']} - {result['error']}")
        else:
            print(f"OK: {result['url']} - статус {result['status']}")

asyncio.run(main())
```

## Управление количеством одновременных задач

Запуск слишком большого количества одновременных запросов может перегрузить сервер или исчерпать локальные ресурсы. Используйте семафор для ограничения:

```python
import aiohttp
import asyncio
from typing import List

async def fetch_with_semaphore(
    session: aiohttp.ClientSession,
    url: str,
    semaphore: asyncio.Semaphore
) -> dict:
    """Загружает URL с ограничением конкурентности"""
    async with semaphore:  # Ограничиваем количество одновременных запросов
        print(f"Начинаем загрузку: {url}")
        async with session.get(url) as response:
            data = await response.json()
            print(f"Завершена загрузка: {url}")
            return data

async def fetch_all_limited(urls: List[str], max_concurrent: int = 5) -> List[dict]:
    """Загружает все URL с ограничением конкурентности"""
    semaphore = asyncio.Semaphore(max_concurrent)

    async with aiohttp.ClientSession() as session:
        tasks = [
            asyncio.create_task(fetch_with_semaphore(session, url, semaphore))
            for url in urls
        ]
        return await asyncio.gather(*tasks)

async def main():
    # Создаем 20 URL
    urls = [f'https://httpbin.org/delay/1?id={i}' for i in range(20)]

    import time
    start = time.time()

    # Ограничиваем до 5 одновременных запросов
    results = await fetch_all_limited(urls, max_concurrent=5)

    print(f"Загружено {len(results)} URL за {time.time() - start:.2f} сек")
    # Без ограничений: ~1 сек (все 20 одновременно)
    # С ограничением 5: ~4 сек (4 раунда по 5 запросов)

asyncio.run(main())
```

## Отмена задач

Задачи можно отменять:

```python
import asyncio

async def long_running_task():
    try:
        print("Задача начата")
        await asyncio.sleep(10)
        print("Задача завершена")
    except asyncio.CancelledError:
        print("Задача отменена!")
        raise  # Важно: перебрасываем исключение

async def main():
    task = asyncio.create_task(long_running_task())

    await asyncio.sleep(1)  # Даем задаче поработать
    task.cancel()  # Отменяем

    try:
        await task
    except asyncio.CancelledError:
        print("Задача была отменена")

asyncio.run(main())
```

## Обработка исключений в задачах

```python
import asyncio
import aiohttp

async def risky_fetch(session, url):
    """Запрос, который может упасть"""
    async with session.get(url) as response:
        response.raise_for_status()  # Выбросит исключение при ошибке
        return await response.json()

async def main():
    urls = [
        'https://httpbin.org/status/200',
        'https://httpbin.org/status/404',  # Ошибка!
        'https://httpbin.org/status/200',
    ]

    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(risky_fetch(session, url)) for url in urls]

        # Способ 1: gather с return_exceptions=True
        results = await asyncio.gather(*tasks, return_exceptions=True)

        for url, result in zip(urls, results):
            if isinstance(result, Exception):
                print(f"ОШИБКА: {url} - {result}")
            else:
                print(f"OK: {url}")

asyncio.run(main())
```

## TaskGroup (Python 3.11+)

В Python 3.11 появился `TaskGroup` - более удобный способ управления группой задач:

```python
import asyncio

async def worker(name, delay):
    await asyncio.sleep(delay)
    return f"Результат от {name}"

async def main():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(worker("Task-1", 1))
        task2 = tg.create_task(worker("Task-2", 2))
        task3 = tg.create_task(worker("Task-3", 1))
        # При выходе из блока все задачи будут завершены

    # Результаты доступны через объекты задач
    print(task1.result())
    print(task2.result())
    print(task3.result())

asyncio.run(main())
```

Преимущества `TaskGroup`:
- Автоматическое ожидание всех задач
- Лучшая обработка ошибок (если одна задача падает, остальные отменяются)
- Более явная структура кода

## Сравнение способов запуска конкурентных задач

| Способ | Когда использовать |
|--------|-------------------|
| `asyncio.create_task()` | Когда нужен контроль над отдельными задачами |
| `asyncio.gather()` | Когда нужны все результаты после завершения |
| `asyncio.as_completed()` | Когда нужно обрабатывать результаты по мере готовности |
| `asyncio.wait()` | Когда нужен точный контроль (таймауты, первый результат) |
| `asyncio.TaskGroup` | Python 3.11+, структурированная конкурентность |

## Лучшие практики

1. **Используйте семафоры** для ограничения одновременных запросов
2. **Обрабатывайте исключения** - сетевые операции ненадежны
3. **Устанавливайте таймауты** для защиты от зависших задач
4. **Группируйте связанные задачи** логически
5. **Не создавайте слишком много задач** - это расходует память

## Частые ошибки

```python
# ОШИБКА: Создание корутин без await или Task
async def wrong():
    coros = [fetch(url) for url in urls]  # Корутины НЕ запущены
    # Ничего не происходит!

# ПРАВИЛЬНО:
async def correct():
    tasks = [asyncio.create_task(fetch(url)) for url in urls]
    results = await asyncio.gather(*tasks)

# ОШИБКА: Игнорирование исключений в задачах
async def wrong2():
    tasks = [asyncio.create_task(risky_operation()) for _ in range(10)]
    await asyncio.gather(*tasks)  # Если одна упадет - всё упадет

# ПРАВИЛЬНО:
async def correct2():
    tasks = [asyncio.create_task(risky_operation()) for _ in range(10)]
    results = await asyncio.gather(*tasks, return_exceptions=True)
    for result in results:
        if isinstance(result, Exception):
            print(f"Ошибка: {result}")
```

---

[prev: ./02-context-managers.md](./02-context-managers.md) | [next: ./04-gather.md](./04-gather.md)
