# Конкурентное выполнение с задачами

[prev: ./02-sleep.md](./02-sleep.md) | [next: ./04-cancellation-timeouts.md](./04-cancellation-timeouts.md)

---

## Что такое Task?

**Task (задача)** в asyncio - это обёртка над корутиной, которая позволяет запустить её выполнение в фоновом режиме. Task наследуется от `Future` и представляет собой запланированное выполнение корутины в event loop.

Основное отличие от простого `await`:
- `await coroutine()` - выполняет корутину последовательно, ожидая её завершения
- `asyncio.create_task(coroutine())` - запускает корутину в фоне, позволяя продолжить выполнение

## Создание задач

### asyncio.create_task() (рекомендуемый способ)

```python
import asyncio

async def my_task(name: str, delay: float) -> str:
    print(f"Задача {name}: начало")
    await asyncio.sleep(delay)
    print(f"Задача {name}: завершена")
    return f"Результат {name}"

async def main():
    # Создаём задачу - корутина начинает выполняться немедленно
    task = asyncio.create_task(my_task("A", 2))

    # Можем делать другую работу, пока задача выполняется
    print("Главная функция: делаю другую работу")
    await asyncio.sleep(1)
    print("Главная функция: закончила другую работу")

    # Ожидаем завершения задачи
    result = await task
    print(f"Получен результат: {result}")

asyncio.run(main())
```

### asyncio.ensure_future() (для совместимости)

```python
import asyncio

async def my_coroutine():
    await asyncio.sleep(1)
    return "готово"

async def main():
    # ensure_future работает как с корутинами, так и с Future
    task = asyncio.ensure_future(my_coroutine())
    result = await task
    print(result)

asyncio.run(main())
```

### Именование задач

```python
import asyncio

async def worker():
    await asyncio.sleep(1)

async def main():
    # Создание задачи с именем
    task = asyncio.create_task(worker(), name="worker_task")

    print(f"Имя задачи: {task.get_name()}")

    # Можно изменить имя
    task.set_name("renamed_worker")
    print(f"Новое имя: {task.get_name()}")

    await task

asyncio.run(main())
```

## Параллельное выполнение задач

### Запуск нескольких задач одновременно

```python
import asyncio
import time

async def fetch_data(source: str, delay: float) -> dict:
    print(f"Начинаю загрузку из {source}")
    await asyncio.sleep(delay)
    print(f"Завершена загрузка из {source}")
    return {"source": source, "data": f"данные из {source}"}

async def main():
    start = time.perf_counter()

    # Создаём задачи - все начинают выполняться сразу
    task1 = asyncio.create_task(fetch_data("API", 2))
    task2 = asyncio.create_task(fetch_data("Database", 1))
    task3 = asyncio.create_task(fetch_data("Cache", 0.5))

    # Ожидаем все задачи
    result1 = await task1
    result2 = await task2
    result3 = await task3

    elapsed = time.perf_counter() - start
    print(f"Все задачи выполнены за {elapsed:.2f} сек")
    # Вывод: ~2 секунды (параллельно), а не 3.5 секунды (последовательно)

asyncio.run(main())
```

## asyncio.gather() - группировка задач

`asyncio.gather()` - удобный способ запуска нескольких корутин параллельно и ожидания их завершения:

```python
import asyncio

async def task_a():
    await asyncio.sleep(1)
    return "A"

async def task_b():
    await asyncio.sleep(2)
    return "B"

async def task_c():
    await asyncio.sleep(0.5)
    return "C"

async def main():
    # gather запускает все корутины параллельно
    results = await asyncio.gather(task_a(), task_b(), task_c())
    print(results)  # ['A', 'B', 'C'] - порядок сохраняется

asyncio.run(main())
```

### Обработка исключений в gather

```python
import asyncio

async def successful_task():
    await asyncio.sleep(0.5)
    return "успех"

async def failing_task():
    await asyncio.sleep(0.5)
    raise ValueError("Ошибка в задаче!")

async def main():
    # По умолчанию: первое исключение прерывает gather
    try:
        results = await asyncio.gather(
            successful_task(),
            failing_task(),
            successful_task()
        )
    except ValueError as e:
        print(f"Поймано исключение: {e}")

    # С return_exceptions=True: исключения возвращаются как результаты
    results = await asyncio.gather(
        successful_task(),
        failing_task(),
        successful_task(),
        return_exceptions=True
    )
    print(results)  # ['успех', ValueError('Ошибка в задаче!'), 'успех']

    # Обработка смешанных результатов
    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Задача {i} завершилась с ошибкой: {result}")
        else:
            print(f"Задача {i} вернула: {result}")

asyncio.run(main())
```

## asyncio.TaskGroup (Python 3.11+)

TaskGroup - современный способ группировки задач с автоматической обработкой ошибок:

```python
import asyncio

async def worker(name: str, delay: float) -> str:
    print(f"Worker {name}: начало")
    await asyncio.sleep(delay)
    print(f"Worker {name}: конец")
    return f"Результат {name}"

async def main():
    async with asyncio.TaskGroup() as tg:
        task1 = tg.create_task(worker("A", 1))
        task2 = tg.create_task(worker("B", 2))
        task3 = tg.create_task(worker("C", 0.5))

    # Все задачи завершены к этому моменту
    print(f"Результаты: {task1.result()}, {task2.result()}, {task3.result()}")

asyncio.run(main())
```

### Обработка ошибок в TaskGroup

```python
import asyncio

async def failing_worker():
    await asyncio.sleep(0.5)
    raise RuntimeError("Worker упал!")

async def main():
    try:
        async with asyncio.TaskGroup() as tg:
            tg.create_task(asyncio.sleep(2))
            tg.create_task(failing_worker())  # Эта задача упадёт
            tg.create_task(asyncio.sleep(1))
    except* RuntimeError as eg:
        # ExceptionGroup содержит все исключения
        for exc in eg.exceptions:
            print(f"Поймано: {exc}")

asyncio.run(main())
```

## Состояние задачи

```python
import asyncio

async def long_task():
    await asyncio.sleep(5)
    return "готово"

async def main():
    task = asyncio.create_task(long_task())

    # Проверка состояния
    print(f"Выполняется: {not task.done()}")
    print(f"Завершена: {task.done()}")
    print(f"Отменена: {task.cancelled()}")

    await asyncio.sleep(0.1)

    # Отменяем задачу
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Задача была отменена")

    print(f"После отмены - cancelled(): {task.cancelled()}")

asyncio.run(main())
```

## Получение результата задачи

```python
import asyncio

async def compute(x: int) -> int:
    await asyncio.sleep(0.5)
    return x * 2

async def main():
    task = asyncio.create_task(compute(21))

    # Способ 1: await задачи
    result = await task
    print(f"Результат через await: {result}")

    # Способ 2: task.result() (только после завершения!)
    task2 = asyncio.create_task(compute(10))
    await task2  # Сначала дождёмся завершения
    print(f"Результат через result(): {task2.result()}")

    # ОШИБКА: вызов result() до завершения
    task3 = asyncio.create_task(compute(5))
    try:
        task3.result()  # InvalidStateError!
    except asyncio.InvalidStateError:
        print("Нельзя получить результат незавершённой задачи")
    await task3

asyncio.run(main())
```

## Callback при завершении задачи

```python
import asyncio

def task_done_callback(task: asyncio.Task):
    """Callback вызывается при завершении задачи."""
    if task.cancelled():
        print(f"Задача {task.get_name()} была отменена")
    elif task.exception():
        print(f"Задача {task.get_name()} завершилась с ошибкой: {task.exception()}")
    else:
        print(f"Задача {task.get_name()} завершена с результатом: {task.result()}")

async def worker(name: str, should_fail: bool = False):
    await asyncio.sleep(0.5)
    if should_fail:
        raise ValueError(f"Ошибка в {name}")
    return f"успех {name}"

async def main():
    # Успешная задача
    task1 = asyncio.create_task(worker("task1"), name="task1")
    task1.add_done_callback(task_done_callback)

    # Задача с ошибкой
    task2 = asyncio.create_task(worker("task2", should_fail=True), name="task2")
    task2.add_done_callback(task_done_callback)

    # Ждём завершения, обрабатывая исключения
    await asyncio.gather(task1, task2, return_exceptions=True)

asyncio.run(main())
```

## Получение всех задач

```python
import asyncio

async def background_task(name: str):
    while True:
        await asyncio.sleep(1)
        print(f"{name}: работаю")

async def main():
    # Создаём несколько фоновых задач
    task1 = asyncio.create_task(background_task("Task1"))
    task2 = asyncio.create_task(background_task("Task2"))

    await asyncio.sleep(0.1)

    # Получаем все текущие задачи
    all_tasks = asyncio.all_tasks()
    print(f"Всего задач: {len(all_tasks)}")

    for task in all_tasks:
        print(f"  - {task.get_name()}: done={task.done()}")

    # Получаем текущую задачу
    current = asyncio.current_task()
    print(f"Текущая задача: {current.get_name()}")

    # Отменяем фоновые задачи
    task1.cancel()
    task2.cancel()

    # Ждём отмены
    await asyncio.gather(task1, task2, return_exceptions=True)

asyncio.run(main())
```

## Паттерны работы с задачами

### Паттерн: Producer-Consumer

```python
import asyncio
import random

async def producer(queue: asyncio.Queue, name: str):
    """Производитель: добавляет элементы в очередь."""
    for i in range(5):
        item = f"{name}-item-{i}"
        await queue.put(item)
        print(f"Producer {name}: добавил {item}")
        await asyncio.sleep(random.uniform(0.1, 0.5))

async def consumer(queue: asyncio.Queue, name: str):
    """Потребитель: обрабатывает элементы из очереди."""
    while True:
        try:
            item = await asyncio.wait_for(queue.get(), timeout=2.0)
            print(f"Consumer {name}: обрабатывает {item}")
            await asyncio.sleep(random.uniform(0.1, 0.3))
            queue.task_done()
        except asyncio.TimeoutError:
            print(f"Consumer {name}: очередь пуста, завершаюсь")
            break

async def main():
    queue = asyncio.Queue()

    # Запускаем производителей и потребителей
    producers = [
        asyncio.create_task(producer(queue, f"P{i}"))
        for i in range(2)
    ]
    consumers = [
        asyncio.create_task(consumer(queue, f"C{i}"))
        for i in range(3)
    ]

    # Ждём завершения производителей
    await asyncio.gather(*producers)

    # Ждём обработки всех элементов
    await queue.join()

    # Отменяем потребителей
    for c in consumers:
        c.cancel()

asyncio.run(main())
```

### Паттерн: Пул задач с ограничением

```python
import asyncio

async def limited_task_pool(tasks: list, max_concurrent: int):
    """Выполняет задачи с ограничением параллелизма."""
    semaphore = asyncio.Semaphore(max_concurrent)

    async def run_with_semaphore(coro):
        async with semaphore:
            return await coro

    return await asyncio.gather(
        *[run_with_semaphore(task) for task in tasks]
    )

async def fetch_url(url: str) -> str:
    print(f"Начинаю загрузку: {url}")
    await asyncio.sleep(1)  # Имитация запроса
    print(f"Завершена загрузка: {url}")
    return f"Данные с {url}"

async def main():
    urls = [f"https://example.com/page{i}" for i in range(10)]
    tasks = [fetch_url(url) for url in urls]

    # Максимум 3 параллельных запроса
    results = await limited_task_pool(tasks, max_concurrent=3)
    print(f"Получено {len(results)} результатов")

asyncio.run(main())
```

## Best Practices

1. **Используйте `asyncio.create_task()`** для создания задач (не `ensure_future()`)
2. **Всегда ожидайте задачи** - неожидаемые задачи могут быть собраны GC
3. **Обрабатывайте исключения** - используйте `return_exceptions=True` или try/except
4. **Именуйте задачи** для упрощения отладки
5. **Используйте `TaskGroup`** в Python 3.11+ для автоматического управления

```python
import asyncio

async def best_practices_example():
    # Хороший пример создания и управления задачами
    async with asyncio.TaskGroup() as tg:
        # Именованные задачи
        task1 = tg.create_task(
            asyncio.sleep(1),
            name="sleep_task"
        )

    # Или для Python < 3.11
    tasks = [
        asyncio.create_task(asyncio.sleep(0.5), name=f"task_{i}")
        for i in range(3)
    ]

    results = await asyncio.gather(*tasks, return_exceptions=True)

    for task, result in zip(tasks, results):
        if isinstance(result, Exception):
            print(f"{task.get_name()} failed: {result}")
        else:
            print(f"{task.get_name()} succeeded")

asyncio.run(best_practices_example())
```

---

[prev: ./02-sleep.md](./02-sleep.md) | [next: ./04-cancellation-timeouts.md](./04-cancellation-timeouts.md)
