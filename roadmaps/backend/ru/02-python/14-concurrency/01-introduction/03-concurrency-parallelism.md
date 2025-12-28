# Конкурентность и параллелизм

[prev: ./02-io-bound-cpu-bound.md](./02-io-bound-cpu-bound.md) | [next: ./04-processes-threads.md](./04-processes-threads.md)

---

## Введение

Конкурентность (concurrency) и параллелизм (parallelism) - два понятия, которые часто путают. Понимание разницы между ними критически важно для написания эффективного асинхронного кода.

## Определения

### Конкурентность (Concurrency)

**Конкурентность** - это способность системы работать с несколькими задачами, переключаясь между ними. Задачи могут выполняться "одновременно" на логическом уровне, но не обязательно в один момент времени.

> "Dealing with lots of things at once" - Rob Pike

```
Конкурентность (один исполнитель):

Задача A: ████░░░░░░████░░░░░░████
Задача B: ░░░░████░░░░░░████░░░░░░░
Задача C: ░░░░░░░░████░░░░░░████░░░
          ─────────────────────────→ Время

Один работник переключается между задачами
```

### Параллелизм (Parallelism)

**Параллелизм** - это одновременное выполнение нескольких задач в буквальном смысле, на нескольких процессорах/ядрах.

> "Doing lots of things at once" - Rob Pike

```
Параллелизм (несколько исполнителей):

Ядро 1 - Задача A: ████████████████████
Ядро 2 - Задача B: ████████████████████
Ядро 3 - Задача C: ████████████████████
                   ─────────────────────→ Время

Несколько работников выполняют задачи одновременно
```

## Аналогия с реальным миром

### Конкурентность: Один официант, много столиков

```python
# Конкурентность: один официант обслуживает много столиков
async def waiter_concurrent():
    """
    Официант переключается между столиками:
    1. Принял заказ у столика 1
    2. Пока кухня готовит - принял заказ у столика 2
    3. Принёс еду столику 1
    4. Принял заказ у столика 3
    ...
    """
    async def serve_table(table_id: int):
        print(f"Принимаю заказ у столика {table_id}")
        await asyncio.sleep(1)  # Ожидание кухни
        print(f"Подаю еду столику {table_id}")

    # Один официант, много столиков
    await asyncio.gather(
        serve_table(1),
        serve_table(2),
        serve_table(3)
    )
```

### Параллелизм: Много официантов

```python
from multiprocessing import Pool

def serve_table_parallel(table_id: int):
    """Каждый официант обслуживает свой столик"""
    print(f"Официант обслуживает столик {table_id}")
    time.sleep(1)  # Работа с клиентом
    return f"Столик {table_id} обслужен"

# Параллелизм: много официантов одновременно
with Pool(processes=3) as pool:
    results = pool.map(serve_table_parallel, [1, 2, 3])
```

## Связь конкурентности и параллелизма

```
                    ┌───────────────────────────────────────┐
                    │         КОНКУРЕНТНОСТЬ                │
                    │  (структура программы)                │
                    │                                       │
                    │  ┌─────────────────────────────────┐  │
                    │  │      ПАРАЛЛЕЛИЗМ                │  │
                    │  │  (способ выполнения)            │  │
                    │  │                                 │  │
                    │  │  - Требует нескольких ядер     │  │
                    │  │  - Буквально одновременно      │  │
                    │  └─────────────────────────────────┘  │
                    │                                       │
                    │  Параллелизм - подмножество           │
                    │  конкурентности                       │
                    └───────────────────────────────────────┘
```

### Возможные комбинации:

1. **Не конкурентно, не параллельно** - последовательное выполнение
2. **Конкурентно, не параллельно** - asyncio, переключение задач
3. **Параллельно, конкурентно** - multiprocessing с несколькими процессами
4. **Параллельно, не конкурентно** - один поток на каждую задачу (редко)

## Примеры в Python

### Последовательное выполнение (ни то, ни другое)

```python
import time

def task(name: str, duration: int):
    print(f"Задача {name} начата")
    time.sleep(duration)
    print(f"Задача {name} завершена")

def sequential():
    start = time.time()

    task("A", 2)
    task("B", 2)
    task("C", 2)

    print(f"Общее время: {time.time() - start:.1f} сек")  # ~6 секунд

sequential()
```

### Конкурентность с asyncio

```python
import asyncio

async def async_task(name: str, duration: int):
    print(f"Задача {name} начата")
    await asyncio.sleep(duration)  # Неблокирующее ожидание
    print(f"Задача {name} завершена")

async def concurrent():
    start = time.time()

    # Все задачи запускаются "одновременно"
    await asyncio.gather(
        async_task("A", 2),
        async_task("B", 2),
        async_task("C", 2)
    )

    print(f"Общее время: {time.time() - start:.1f} сек")  # ~2 секунды

asyncio.run(concurrent())
```

### Параллелизм с multiprocessing

```python
from multiprocessing import Pool
import time
import os

def parallel_task(args):
    name, duration = args
    print(f"Задача {name} на процессе {os.getpid()}")
    time.sleep(duration)
    return f"Задача {name} завершена"

def parallel():
    start = time.time()

    with Pool(processes=3) as pool:
        # Реально одновременное выполнение на разных ядрах
        results = pool.map(parallel_task, [("A", 2), ("B", 2), ("C", 2)])

    print(results)
    print(f"Общее время: {time.time() - start:.1f} сек")  # ~2 секунды

if __name__ == "__main__":
    parallel()
```

## Когда что использовать?

### Конкурентность (asyncio) подходит для:

```python
# I/O-bound задачи
async def io_bound_example():
    async with aiohttp.ClientSession() as session:
        # Пока ждём ответ - другие задачи работают
        tasks = [
            session.get("https://api1.example.com"),
            session.get("https://api2.example.com"),
            session.get("https://api3.example.com")
        ]
        responses = await asyncio.gather(*tasks)
```

**Преимущества конкурентности:**
- Меньше накладных расходов (один поток)
- Проще обмен данными (общая память)
- Нет проблем с синхронизацией
- Идеально для I/O-bound задач

### Параллелизм (multiprocessing) подходит для:

```python
from multiprocessing import Pool

def cpu_bound_example():
    """CPU-bound задачи"""
    with Pool(processes=4) as pool:
        # Каждый процесс на своём ядре
        results = pool.map(heavy_computation, large_dataset)
    return results
```

**Преимущества параллелизма:**
- Реальное ускорение CPU-bound задач
- Использование всех ядер процессора
- Обход GIL
- Изоляция между процессами

## Визуальное сравнение производительности

### I/O-bound задача (3 запроса по 1 секунде):

```
Последовательно:      ████████████  (3 сек)
Конкурентно (asyncio): ████         (1 сек)
Параллельно:           ████         (1 сек)

Для I/O-bound: конкурентность ≈ параллелизм
```

### CPU-bound задача (3 вычисления по 1 секунде):

```
Последовательно:      ████████████  (3 сек)
Конкурентно (asyncio): ████████████  (3 сек) - НЕ ПОМОГАЕТ!
Параллельно:           ████         (1 сек)

Для CPU-bound: только параллелизм даёт ускорение
```

## Практический пример: Web Scraper

```python
import asyncio
import aiohttp
from concurrent.futures import ProcessPoolExecutor
from bs4 import BeautifulSoup

def parse_html(html: str) -> dict:
    """CPU-bound: парсинг HTML"""
    soup = BeautifulSoup(html, 'html.parser')
    return {
        'title': soup.title.string if soup.title else None,
        'links': len(soup.find_all('a')),
        'paragraphs': len(soup.find_all('p'))
    }

async def fetch_url(session: aiohttp.ClientSession, url: str) -> str:
    """I/O-bound: загрузка страницы"""
    async with session.get(url) as response:
        return await response.text()

async def scrape_with_both(urls: list[str]):
    """
    Комбинация:
    - Конкурентность для загрузки (I/O-bound)
    - Параллелизм для парсинга (CPU-bound)
    """
    results = []

    async with aiohttp.ClientSession() as session:
        # Конкурентная загрузка всех страниц
        html_pages = await asyncio.gather(*[
            fetch_url(session, url) for url in urls
        ])

    # Параллельный парсинг
    with ProcessPoolExecutor(max_workers=4) as executor:
        loop = asyncio.get_running_loop()
        parse_tasks = [
            loop.run_in_executor(executor, parse_html, html)
            for html in html_pages
        ]
        results = await asyncio.gather(*parse_tasks)

    return dict(zip(urls, results))

async def main():
    urls = [
        "https://python.org",
        "https://docs.python.org",
        "https://pypi.org",
        "https://realpython.com"
    ]

    results = await scrape_with_both(urls)
    for url, data in results.items():
        print(f"{url}: {data}")

asyncio.run(main())
```

## Типичные ошибки

### 1. Использование asyncio для CPU-bound

```python
# НЕПРАВИЛЬНО - asyncio не ускорит CPU-bound
async def bad_approach():
    # Эти задачи НЕ будут выполняться параллельно!
    await asyncio.gather(
        heavy_computation(data1),  # Блокирует event loop
        heavy_computation(data2),
        heavy_computation(data3)
    )

# ПРАВИЛЬНО - используйте ProcessPoolExecutor
async def good_approach():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as executor:
        results = await asyncio.gather(
            loop.run_in_executor(executor, heavy_computation, data1),
            loop.run_in_executor(executor, heavy_computation, data2),
            loop.run_in_executor(executor, heavy_computation, data3)
        )
```

### 2. Использование multiprocessing для I/O-bound

```python
# НЕЭФФЕКТИВНО - процессы для I/O-bound
def inefficient():
    with Pool(100) as pool:  # 100 процессов!
        pool.map(fetch_url, urls)  # Много накладных расходов

# ЭФФЕКТИВНО - asyncio для I/O-bound
async def efficient():
    async with aiohttp.ClientSession() as session:
        await asyncio.gather(*[
            fetch_url(session, url) for url in urls
        ])  # Один поток, тысячи соединений
```

## Резюме

| Аспект | Конкурентность | Параллелизм |
|--------|---------------|-------------|
| Определение | Работа с несколькими задачами | Выполнение нескольких задач одновременно |
| Требования | Один CPU | Несколько CPU/ядер |
| Python инструменты | asyncio, threading | multiprocessing |
| Лучше для | I/O-bound | CPU-bound |
| Накладные расходы | Низкие | Высокие |
| Обмен данными | Простой | Сложный (между процессами) |

**Главное правило:**
- **I/O-bound** → Конкурентность (asyncio)
- **CPU-bound** → Параллелизм (multiprocessing)
- **Смешанные задачи** → Комбинация обоих подходов

---

[prev: ./02-io-bound-cpu-bound.md](./02-io-bound-cpu-bound.md) | [next: ./04-processes-threads.md](./04-processes-threads.md)
