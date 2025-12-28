# Обработка результатов по мере поступления

[prev: ./04-gather.md](./04-gather.md) | [next: ./06-wait.md](./06-wait.md)

---

## Что такое asyncio.as_completed?

`asyncio.as_completed()` - это функция, которая возвращает итератор, выдающий задачи в порядке их завершения (а не в порядке их создания). Это позволяет обрабатывать результаты сразу, как только они становятся доступны.

```python
import asyncio

async def task(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"Задача {name} (задержка {delay}с)"

async def main():
    tasks = [
        asyncio.create_task(task("A", 3)),
        asyncio.create_task(task("B", 1)),
        asyncio.create_task(task("C", 2)),
    ]

    # Обрабатываем в порядке завершения
    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(result)

asyncio.run(main())

# Вывод (в порядке завершения):
# Задача B (задержка 1с)
# Задача C (задержка 2с)
# Задача A (задержка 3с)
```

## Сравнение с gather

| Характеристика | gather | as_completed |
|---------------|--------|--------------|
| Порядок результатов | Сохраняется | По мере завершения |
| Когда возвращает | После завершения всех | Сразу по готовности |
| Результат | Список | Итератор Future |
| Когда использовать | Нужны все результаты | Нужна быстрая реакция |

```python
import asyncio
import time

async def slow_task(id: int, delay: float):
    await asyncio.sleep(delay)
    return f"Задача {id}"

async def with_gather():
    """gather ждет ВСЕ задачи"""
    start = time.time()

    results = await asyncio.gather(
        slow_task(1, 3),
        slow_task(2, 1),
        slow_task(3, 2),
    )

    # Результаты доступны только через 3 секунды
    for result in results:
        print(f"{time.time() - start:.1f}с: {result}")

async def with_as_completed():
    """as_completed выдает результаты по мере готовности"""
    start = time.time()

    tasks = [
        asyncio.create_task(slow_task(1, 3)),
        asyncio.create_task(slow_task(2, 1)),
        asyncio.create_task(slow_task(3, 2)),
    ]

    for coro in asyncio.as_completed(tasks):
        result = await coro
        print(f"{time.time() - start:.1f}с: {result}")

# gather:
# 3.0с: Задача 1
# 3.0с: Задача 2
# 3.0с: Задача 3

# as_completed:
# 1.0с: Задача 2
# 2.0с: Задача 3
# 3.0с: Задача 1
```

## Сигнатура функции

```python
asyncio.as_completed(aws, *, timeout=None)
```

- `aws` - итерируемый объект с awaitable (корутины или задачи)
- `timeout` - максимальное время ожидания (секунды). Если превышено, вызывает `asyncio.TimeoutError`

## Практический пример с aiohttp

### Загрузка URL с отображением прогресса

```python
import aiohttp
import asyncio
import time

async def fetch(session: aiohttp.ClientSession, url: str) -> dict:
    async with session.get(url) as response:
        data = await response.json()
        return {'url': url, 'status': response.status, 'data': data}

async def main():
    urls = [
        'https://httpbin.org/delay/3',
        'https://httpbin.org/delay/1',
        'https://httpbin.org/delay/2',
        'https://httpbin.org/get',
    ]

    start = time.time()

    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(fetch(session, url)) for url in urls]

        completed = 0
        total = len(tasks)

        for coro in asyncio.as_completed(tasks):
            result = await coro
            completed += 1
            elapsed = time.time() - start
            print(f"[{completed}/{total}] {elapsed:.1f}с - {result['url']}")

asyncio.run(main())

# Вывод:
# [1/4] 0.2с - https://httpbin.org/get
# [2/4] 1.0с - https://httpbin.org/delay/1
# [3/4] 2.0с - https://httpbin.org/delay/2
# [4/4] 3.0с - https://httpbin.org/delay/3
```

### Обработка первого успешного результата

```python
import aiohttp
import asyncio

async def fetch_from_mirror(session: aiohttp.ClientSession, url: str) -> dict:
    """Попытка загрузить данные с зеркала"""
    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=5)) as response:
            if response.status == 200:
                return {'url': url, 'data': await response.json(), 'success': True}
            return {'url': url, 'data': None, 'success': False}
    except Exception as e:
        return {'url': url, 'data': None, 'success': False, 'error': str(e)}

async def fetch_first_success(urls: list[str]) -> dict:
    """Возвращает первый успешный результат"""
    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(fetch_from_mirror(session, url)) for url in urls]

        for coro in asyncio.as_completed(tasks):
            result = await coro
            if result['success']:
                # Отменяем оставшиеся задачи
                for task in tasks:
                    if not task.done():
                        task.cancel()
                return result

        return {'url': None, 'data': None, 'success': False}

async def main():
    mirrors = [
        'https://httpbin.org/delay/3',  # Медленный
        'https://httpbin.org/delay/1',  # Быстрый - будет выбран
        'https://httpbin.org/delay/2',
    ]

    result = await fetch_first_success(mirrors)
    if result['success']:
        print(f"Успех! Загружено с {result['url']}")
    else:
        print("Все зеркала недоступны")

asyncio.run(main())
```

## Таймаут с as_completed

```python
import asyncio

async def slow_task(id: int, delay: float):
    await asyncio.sleep(delay)
    return f"Задача {id}"

async def main():
    tasks = [
        asyncio.create_task(slow_task(1, 1)),
        asyncio.create_task(slow_task(2, 5)),  # Не успеет
        asyncio.create_task(slow_task(3, 2)),
    ]

    try:
        # Таймаут 3 секунды на все задачи
        for coro in asyncio.as_completed(tasks, timeout=3):
            result = await coro
            print(result)
    except asyncio.TimeoutError:
        print("Превышен таймаут!")

        # Какие задачи не завершились?
        for task in tasks:
            if not task.done():
                print(f"Задача {task.get_name()} не завершена")
                task.cancel()

asyncio.run(main())

# Вывод:
# Задача 1
# Задача 3
# Превышен таймаут!
# Задача Task-2 не завершена
```

## Обработка ошибок

```python
import asyncio
import aiohttp

async def fetch_may_fail(session: aiohttp.ClientSession, url: str) -> dict:
    async with session.get(url) as response:
        response.raise_for_status()
        return {'url': url, 'data': await response.json()}

async def main():
    urls = [
        'https://httpbin.org/get',
        'https://httpbin.org/status/404',  # Ошибка
        'https://httpbin.org/delay/1',
    ]

    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(fetch_may_fail(session, url)) for url in urls]

        successful = []
        failed = []

        for coro in asyncio.as_completed(tasks):
            try:
                result = await coro
                successful.append(result)
                print(f"OK: {result['url']}")
            except Exception as e:
                failed.append(str(e))
                print(f"ОШИБКА: {e}")

        print(f"\nУспешно: {len(successful)}, Ошибок: {len(failed)}")

asyncio.run(main())
```

## Отслеживание какая задача завершилась

Проблема `as_completed` в том, что вы не знаете, какая именно задача вернула результат:

```python
import asyncio
import aiohttp

async def fetch_with_id(session: aiohttp.ClientSession, id: int, url: str) -> dict:
    """Возвращает результат с идентификатором"""
    async with session.get(url) as response:
        return {
            'id': id,
            'url': url,
            'status': response.status,
            'data': await response.json()
        }

async def main():
    urls = [
        ('user_1', 'https://api.github.com/users/octocat'),
        ('user_2', 'https://api.github.com/users/torvalds'),
        ('delay', 'https://httpbin.org/delay/1'),
    ]

    async with aiohttp.ClientSession() as session:
        tasks = [
            asyncio.create_task(fetch_with_id(session, id, url))
            for id, url in urls
        ]

        for coro in asyncio.as_completed(tasks):
            result = await coro
            print(f"Завершена задача '{result['id']}': {result['url']}")

asyncio.run(main())
```

## Практический пример: параллельный пинг серверов

```python
import aiohttp
import asyncio
import time
from dataclasses import dataclass
from typing import Optional

@dataclass
class PingResult:
    host: str
    response_time: Optional[float]
    status: Optional[int]
    error: Optional[str]

async def ping_server(
    session: aiohttp.ClientSession,
    host: str
) -> PingResult:
    """Пингует сервер и возвращает время ответа"""
    url = f"https://{host}"
    start = time.time()

    try:
        async with session.get(url, timeout=aiohttp.ClientTimeout(total=5)) as response:
            return PingResult(
                host=host,
                response_time=time.time() - start,
                status=response.status,
                error=None
            )
    except asyncio.TimeoutError:
        return PingResult(host=host, response_time=None, status=None, error="Таймаут")
    except Exception as e:
        return PingResult(host=host, response_time=None, status=None, error=str(e))

async def ping_all_servers(hosts: list[str]) -> list[PingResult]:
    """Пингует все серверы и выводит результаты по мере получения"""
    results = []

    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(ping_server(session, host)) for host in hosts]

        print("Пингуем серверы...\n")

        for coro in asyncio.as_completed(tasks):
            result = await coro
            results.append(result)

            if result.error:
                print(f"  {result.host}: ОШИБКА - {result.error}")
            else:
                print(f"  {result.host}: {result.response_time*1000:.0f}мс (статус {result.status})")

    return results

async def main():
    hosts = [
        'api.github.com',
        'httpbin.org',
        'google.com',
        'nonexistent.example.com',
    ]

    results = await ping_all_servers(hosts)

    # Статистика
    successful = [r for r in results if not r.error]
    if successful:
        avg_time = sum(r.response_time for r in successful) / len(successful)
        print(f"\nСреднее время ответа: {avg_time*1000:.0f}мс")
        print(f"Успешно: {len(successful)}/{len(results)}")

asyncio.run(main())
```

## Потоковая обработка результатов

```python
import aiohttp
import asyncio
from typing import AsyncGenerator

async def fetch(session: aiohttp.ClientSession, url: str) -> dict:
    async with session.get(url) as response:
        return {'url': url, 'data': await response.json()}

async def stream_results(urls: list[str]) -> AsyncGenerator[dict, None]:
    """Генератор, выдающий результаты по мере готовности"""
    async with aiohttp.ClientSession() as session:
        tasks = [asyncio.create_task(fetch(session, url)) for url in urls]

        for coro in asyncio.as_completed(tasks):
            try:
                result = await coro
                yield result
            except Exception as e:
                yield {'url': 'unknown', 'error': str(e)}

async def main():
    urls = [
        'https://httpbin.org/delay/2',
        'https://httpbin.org/delay/1',
        'https://httpbin.org/get',
    ]

    # Обрабатываем результаты как поток
    async for result in stream_results(urls):
        if 'error' in result:
            print(f"Ошибка: {result['error']}")
        else:
            print(f"Получено: {result['url']}")

asyncio.run(main())
```

## Комбинирование с семафором

```python
import aiohttp
import asyncio
import time

async def fetch_limited(
    session: aiohttp.ClientSession,
    url: str,
    semaphore: asyncio.Semaphore
) -> dict:
    async with semaphore:
        start = time.time()
        async with session.get(url) as response:
            data = await response.json()
            return {
                'url': url,
                'time': time.time() - start,
                'data': data
            }

async def main():
    urls = [f'https://httpbin.org/delay/1?id={i}' for i in range(10)]
    semaphore = asyncio.Semaphore(3)  # Максимум 3 одновременных запроса

    async with aiohttp.ClientSession() as session:
        tasks = [
            asyncio.create_task(fetch_limited(session, url, semaphore))
            for url in urls
        ]

        start = time.time()
        completed = 0

        for coro in asyncio.as_completed(tasks):
            result = await coro
            completed += 1
            print(f"[{completed}/{len(tasks)}] {time.time()-start:.1f}с - завершен {result['url']}")

asyncio.run(main())
```

## Когда использовать as_completed

**Используйте as_completed когда:**
- Нужно показывать прогресс пользователю
- Нужно обрабатывать результаты как можно раньше
- Хотите остановиться после первого успешного результата
- Порядок результатов не важен
- Нужна потоковая обработка данных

**Используйте gather когда:**
- Нужны все результаты в определенном порядке
- Результаты обрабатываются вместе после завершения всех задач
- Логика обработки зависит от полного набора результатов

## Лучшие практики

1. **Создавайте Tasks явно** перед передачей в `as_completed`
2. **Включайте идентификатор** в результат для отслеживания источника
3. **Обрабатывайте ошибки** для каждой задачи отдельно
4. **Используйте таймаут** для защиты от зависания
5. **Отменяйте ненужные задачи** после получения нужного результата

## Частые ошибки

```python
# ОШИБКА: Передача корутин без создания задач
coros = [fetch(url) for url in urls]
for coro in asyncio.as_completed(coros):  # Работает, но...
    pass  # Нельзя отменить отдельные задачи

# ПРАВИЛЬНО: Создаем задачи для возможности отмены
tasks = [asyncio.create_task(fetch(url)) for url in urls]
for coro in asyncio.as_completed(tasks):
    result = await coro
    if is_good_enough(result):
        for task in tasks:
            task.cancel()  # Можем отменить оставшиеся
        break

# ОШИБКА: Не обрабатываем таймаут
for coro in asyncio.as_completed(tasks, timeout=5):
    await coro  # TimeoutError не обработан!

# ПРАВИЛЬНО:
try:
    for coro in asyncio.as_completed(tasks, timeout=5):
        await coro
except asyncio.TimeoutError:
    print("Не все задачи успели завершиться")
```

---

[prev: ./04-gather.md](./04-gather.md) | [next: ./06-wait.md](./06-wait.md)
