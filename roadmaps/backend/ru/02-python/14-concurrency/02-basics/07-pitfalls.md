# Ловушки сопрограмм и задач

[prev: ./06-timing-decorators.md](./06-timing-decorators.md) | [next: ./08-manual-event-loop.md](./08-manual-event-loop.md)

---

## Введение

Асинхронное программирование в Python имеет множество подводных камней, которые могут привести к неожиданному поведению, ошибкам производительности или сложно отлаживаемым багам. В этом разделе рассмотрим наиболее распространённые ловушки и способы их избежать.

## Ловушка 1: Забытый await

Самая распространённая ошибка - забыть `await` при вызове корутины:

```python
import asyncio

async def fetch_data():
    await asyncio.sleep(1)
    return "данные"

async def main():
    # НЕПРАВИЛЬНО - забыли await
    result = fetch_data()
    print(f"Результат: {result}")
    # Вывод: Результат: <coroutine object fetch_data at 0x...>
    # RuntimeWarning: coroutine 'fetch_data' was never awaited

    # ПРАВИЛЬНО
    result = await fetch_data()
    print(f"Результат: {result}")
    # Вывод: Результат: данные

asyncio.run(main())
```

### Как обнаружить

Python выдаёт `RuntimeWarning`, но его легко пропустить. Используйте режим отладки:

```python
import asyncio
import warnings

# Превращаем предупреждения в ошибки
warnings.filterwarnings("error", category=RuntimeWarning)

# Или включаем debug mode
asyncio.run(main(), debug=True)
```

## Ловушка 2: Блокирующий код в async-функции

Использование синхронных блокирующих операций блокирует весь event loop:

```python
import asyncio
import time
import requests  # Синхронная библиотека!

async def bad_fetch():
    # НЕПРАВИЛЬНО - блокирует event loop
    response = requests.get("https://httpbin.org/delay/2")
    return response.json()

async def bad_sleep():
    # НЕПРАВИЛЬНО - блокирует event loop
    time.sleep(2)
    return "готово"

async def good_fetch():
    # ПРАВИЛЬНО - используем асинхронную библиотеку
    import aiohttp
    async with aiohttp.ClientSession() as session:
        async with session.get("https://httpbin.org/delay/2") as response:
            return await response.json()

async def good_sleep():
    # ПРАВИЛЬНО - используем asyncio.sleep
    await asyncio.sleep(2)
    return "готово"

# Если нужно использовать синхронный код:
async def run_blocking_in_executor():
    loop = asyncio.get_event_loop()
    # Запускаем блокирующий код в отдельном потоке
    result = await loop.run_in_executor(
        None,  # Используем дефолтный ThreadPoolExecutor
        requests.get,
        "https://httpbin.org/delay/2"
    )
    return result.json()
```

### Демонстрация проблемы

```python
import asyncio
import time

async def blocking_task():
    print("Blocking: начало")
    time.sleep(2)  # БЛОКИРУЕТ!
    print("Blocking: конец")

async def other_task():
    print("Other: начало")
    await asyncio.sleep(1)
    print("Other: конец")

async def main():
    start = time.perf_counter()

    # Задачи выполняются ПОСЛЕДОВАТЕЛЬНО из-за time.sleep
    await asyncio.gather(blocking_task(), other_task())

    elapsed = time.perf_counter() - start
    print(f"Общее время: {elapsed:.2f} сек")  # ~3 сек вместо 2!

asyncio.run(main())
```

## Ловушка 3: Создание задачи без ожидания

Создание Task без сохранения ссылки и ожидания может привести к потере задачи:

```python
import asyncio

async def important_task():
    await asyncio.sleep(1)
    print("Важная задача выполнена!")
    return "результат"

async def main():
    # НЕПРАВИЛЬНО - задача может быть собрана GC
    asyncio.create_task(important_task())
    # Если main() завершится раньше, задача не успеет выполниться

    # ПРАВИЛЬНО - сохраняем ссылку
    task = asyncio.create_task(important_task())
    # ... другой код ...
    await task  # Ждём завершения

asyncio.run(main())
```

### Правильное управление фоновыми задачами

```python
import asyncio

class TaskManager:
    """Менеджер фоновых задач."""

    def __init__(self):
        self.tasks: set[asyncio.Task] = set()

    def create_task(self, coro):
        task = asyncio.create_task(coro)
        self.tasks.add(task)
        task.add_done_callback(self.tasks.discard)
        return task

    async def cancel_all(self):
        for task in self.tasks:
            task.cancel()
        await asyncio.gather(*self.tasks, return_exceptions=True)

async def background_worker(name: str):
    while True:
        print(f"Worker {name}: работаю")
        await asyncio.sleep(1)

async def main():
    manager = TaskManager()

    manager.create_task(background_worker("A"))
    manager.create_task(background_worker("B"))

    await asyncio.sleep(3)

    await manager.cancel_all()

asyncio.run(main())
```

## Ловушка 4: Неправильная обработка исключений

```python
import asyncio

async def failing_task():
    await asyncio.sleep(0.5)
    raise ValueError("Ошибка!")

async def main_bad():
    # НЕПРАВИЛЬНО - исключение "проглатывается"
    task = asyncio.create_task(failing_task())
    await asyncio.sleep(2)
    # Task:exception was never retrieved

async def main_good():
    # ПРАВИЛЬНО - обрабатываем исключение
    task = asyncio.create_task(failing_task())

    try:
        await task
    except ValueError as e:
        print(f"Поймано исключение: {e}")

# Или с callback
def handle_exception(task: asyncio.Task):
    if not task.cancelled() and task.exception():
        print(f"Task failed: {task.exception()}")

async def main_with_callback():
    task = asyncio.create_task(failing_task())
    task.add_done_callback(handle_exception)
    await asyncio.sleep(2)

asyncio.run(main_with_callback())
```

## Ловушка 5: Неправильное использование gather

```python
import asyncio

async def task_a():
    await asyncio.sleep(1)
    return "A"

async def task_b():
    await asyncio.sleep(0.5)
    raise ValueError("B failed")

async def task_c():
    await asyncio.sleep(2)
    return "C"

async def main_bad():
    # ПРОБЛЕМА: при ошибке в одной задаче, другие продолжают выполняться
    # но результаты не доступны
    try:
        results = await asyncio.gather(task_a(), task_b(), task_c())
    except ValueError:
        print("Ошибка! Но task_c всё ещё выполняется...")
        await asyncio.sleep(3)

async def main_good():
    # ПРАВИЛЬНО: используем return_exceptions или TaskGroup
    results = await asyncio.gather(
        task_a(), task_b(), task_c(),
        return_exceptions=True
    )

    for i, result in enumerate(results):
        if isinstance(result, Exception):
            print(f"Задача {i} упала: {result}")
        else:
            print(f"Задача {i} вернула: {result}")

asyncio.run(main_good())
```

## Ловушка 6: Модификация общего состояния

Хотя asyncio использует один поток, race conditions всё равно возможны:

```python
import asyncio

counter = 0

async def increment():
    global counter
    # ПРОБЛЕМА: между чтением и записью может произойти переключение
    temp = counter
    await asyncio.sleep(0)  # Точка переключения!
    counter = temp + 1

async def main_bad():
    global counter
    counter = 0

    # Ожидаем counter = 100, но получим меньше
    await asyncio.gather(*[increment() for _ in range(100)])
    print(f"Counter: {counter}")  # Скорее всего < 100

# ПРАВИЛЬНО: используем Lock
lock = asyncio.Lock()

async def safe_increment():
    global counter
    async with lock:
        temp = counter
        await asyncio.sleep(0)
        counter = temp + 1

async def main_good():
    global counter
    counter = 0

    await asyncio.gather(*[safe_increment() for _ in range(100)])
    print(f"Counter: {counter}")  # Точно 100

asyncio.run(main_good())
```

## Ловушка 7: Рекурсивный вызов asyncio.run()

```python
import asyncio

async def inner():
    return "inner"

async def outer():
    # НЕПРАВИЛЬНО - asyncio.run() создаёт новый event loop
    result = asyncio.run(inner())  # RuntimeError!
    return result

async def outer_correct():
    # ПРАВИЛЬНО - используем await
    result = await inner()
    return result

# asyncio.run(outer())  # RuntimeError: cannot be called from a running event loop
asyncio.run(outer_correct())  # OK
```

## Ловушка 8: Использование глобального event loop

```python
import asyncio

# НЕПРАВИЛЬНО - устаревший подход
loop = asyncio.get_event_loop()  # Deprecated warning в Python 3.10+

async def my_coro():
    return "результат"

# НЕПРАВИЛЬНО
# loop.run_until_complete(my_coro())

# ПРАВИЛЬНО - используем asyncio.run()
result = asyncio.run(my_coro())
```

## Ловушка 9: Неправильная отмена задач

```python
import asyncio

async def cleanup_required_task():
    resource = "ресурс"
    print(f"Открыл {resource}")

    try:
        await asyncio.sleep(10)
        return "готово"
    except asyncio.CancelledError:
        # НЕПРАВИЛЬНО - не перебрасываем исключение
        print("Отменено, но не освобождаю ресурс")
        # return "cancelled"  # Плохо!

        # ПРАВИЛЬНО - очистка и переброс
        print(f"Закрываю {resource}")
        raise  # Важно!

async def main():
    task = asyncio.create_task(cleanup_required_task())
    await asyncio.sleep(0.5)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Задача корректно отменена")

asyncio.run(main())
```

## Ловушка 10: Смешивание sync и async в одном классе

```python
import asyncio

class BadAPI:
    """Плохой пример - смешивание sync и async."""

    def __init__(self):
        self.data = None

    # Синхронный метод вызывает асинхронный
    def load(self):
        # НЕПРАВИЛЬНО - не работает внутри async контекста
        asyncio.run(self._load_async())

    async def _load_async(self):
        await asyncio.sleep(1)
        self.data = "loaded"

class GoodAPI:
    """Хороший пример - полностью асинхронный интерфейс."""

    def __init__(self):
        self.data = None

    async def load(self):
        await asyncio.sleep(1)
        self.data = "loaded"

    # Или используем фабричный метод
    @classmethod
    async def create(cls):
        instance = cls()
        await instance.load()
        return instance

async def main():
    api = await GoodAPI.create()
    print(api.data)

asyncio.run(main())
```

## Ловушка 11: Бесконечные циклы без точек выхода

```python
import asyncio

async def bad_infinite_loop():
    # НЕПРАВИЛЬНО - никогда не отдаёт управление
    while True:
        pass  # Блокирует event loop навсегда!

async def good_infinite_loop():
    # ПРАВИЛЬНО - периодически отдаёт управление
    while True:
        # Делаем работу
        await asyncio.sleep(0)  # Даём event loop шанс

async def better_infinite_loop():
    # ЕЩЁ ЛУЧШЕ - используем осмысленные паузы
    while True:
        # Делаем работу
        await asyncio.sleep(0.1)  # Реальная пауза
```

## Ловушка 12: Неправильное тестирование

```python
import asyncio
import pytest

# НЕПРАВИЛЬНО - синхронный тест для async кода
def test_bad():
    async def my_coro():
        return 42

    # Это создаёт warning о неожиданной корутине
    result = my_coro()
    # assert result == 42  # Не работает!

# ПРАВИЛЬНО - используем pytest-asyncio
@pytest.mark.asyncio
async def test_good():
    async def my_coro():
        return 42

    result = await my_coro()
    assert result == 42

# Или вручную запускаем
def test_manual():
    async def my_coro():
        return 42

    result = asyncio.run(my_coro())
    assert result == 42
```

## Чек-лист для избежания ловушек

1. **Всегда используйте `await`** при вызове корутин
2. **Не используйте блокирующие операции** - `time.sleep()`, `requests`, file I/O
3. **Сохраняйте ссылки на задачи** и ожидайте их завершения
4. **Обрабатывайте исключения** в задачах
5. **Используйте `return_exceptions=True`** или `TaskGroup` в gather
6. **Применяйте Lock** для защиты общего состояния
7. **Не вызывайте `asyncio.run()`** из async-кода
8. **Используйте `asyncio.run()`** вместо устаревшего API
9. **Всегда перебрасывайте `CancelledError`** после очистки
10. **Проектируйте API как полностью асинхронный**
11. **Добавляйте точки переключения** в длительных циклах
12. **Используйте pytest-asyncio** для тестирования

```python
import asyncio

async def safe_async_function():
    """Пример безопасной асинхронной функции."""
    lock = asyncio.Lock()

    async def worker():
        async with lock:
            # Защищённый доступ к ресурсу
            await asyncio.sleep(0.1)
            return "готово"

    tasks = [asyncio.create_task(worker()) for _ in range(5)]

    try:
        results = await asyncio.gather(*tasks, return_exceptions=True)
        for result in results:
            if isinstance(result, Exception):
                print(f"Ошибка: {result}")
    except asyncio.CancelledError:
        for task in tasks:
            task.cancel()
        raise

asyncio.run(safe_async_function())
```

---

[prev: ./06-timing-decorators.md](./06-timing-decorators.md) | [next: ./08-manual-event-loop.md](./08-manual-event-loop.md)
