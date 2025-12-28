# Принудительный запуск итерации Event Loop

[prev: ./02-context-vars.md](./02-context-vars.md) | [next: ./04-alternative-loops.md](./04-alternative-loops.md)

---

## Введение

Event loop в asyncio работает по принципу **кооперативной многозадачности**: каждая задача добровольно уступает управление с помощью `await`. Однако в некоторых случаях требуется **принудительно запустить итерацию** event loop для обработки накопившихся событий. Это продвинутая техника, которая требует глубокого понимания работы asyncio.

## Зачем нужна принудительная итерация?

### Основные сценарии использования

1. **Интеграция с синхронным кодом** — когда нужно выполнить async-код из синхронного контекста
2. **Длительные вычисления** — чтобы дать event loop возможность обработать другие события
3. **Тестирование** — для пошаговой отладки асинхронного кода
4. **Специфичные библиотеки** — интеграция с GUI-фреймворками или legacy-кодом

## Метод `asyncio.sleep(0)`

Самый простой и рекомендуемый способ "отпустить" event loop для обработки других задач:

```python
import asyncio

async def long_computation():
    """Длительное вычисление с периодическим освобождением event loop."""
    result = 0
    for i in range(1000000):
        result += i * i

        # Каждые 10000 итераций даём event loop возможность работать
        if i % 10000 == 0:
            await asyncio.sleep(0)
            print(f"Прогресс: {i / 10000}%")

    return result

async def background_task():
    """Фоновая задача."""
    while True:
        print("Фоновая задача работает!")
        await asyncio.sleep(0.5)

async def main():
    # Запускаем фоновую задачу
    bg = asyncio.create_task(background_task())

    # Длительное вычисление не блокирует фоновую задачу
    result = await long_computation()
    print(f"Результат: {result}")

    bg.cancel()

asyncio.run(main())
```

### Почему это работает?

`await asyncio.sleep(0)` создаёт "точку переключения контекста". Event loop:
1. Приостанавливает текущую корутину
2. Проверяет и выполняет готовые задачи
3. Обрабатывает I/O события
4. Возвращает управление исходной корутине

## Низкоуровневые методы управления Event Loop

### `loop.run_until_complete()`

Запускает event loop до завершения указанной корутины:

```python
import asyncio

async def my_coroutine():
    await asyncio.sleep(1)
    return "Результат"

# Получаем или создаём event loop
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)

try:
    # Запускаем корутину синхронно
    result = loop.run_until_complete(my_coroutine())
    print(f"Получили: {result}")
finally:
    loop.close()
```

### `loop.run_forever()` и `loop.stop()`

Для более тонкого контроля над event loop:

```python
import asyncio

async def schedule_stop(loop, delay):
    """Останавливает event loop через указанное время."""
    await asyncio.sleep(delay)
    loop.stop()

async def periodic_task():
    """Периодическая задача."""
    counter = 0
    while True:
        counter += 1
        print(f"Итерация {counter}")
        await asyncio.sleep(0.3)

def main():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    try:
        # Планируем задачи
        loop.create_task(periodic_task())
        loop.create_task(schedule_stop(loop, 2.0))

        # Запускаем event loop "навсегда"
        loop.run_forever()
        print("Event loop остановлен")
    finally:
        # Отменяем все задачи перед закрытием
        pending = asyncio.all_tasks(loop)
        for task in pending:
            task.cancel()
        loop.run_until_complete(asyncio.gather(*pending, return_exceptions=True))
        loop.close()

main()
```

### `loop._run_once()` (Приватный метод)

**ВНИМАНИЕ**: Это приватный метод, который может измениться в будущих версиях Python!

```python
import asyncio

async def simple_task():
    print("Задача выполнена")
    return 42

def manual_iteration_example():
    """Пример ручной итерации event loop."""
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    try:
        # Создаём Future для результата
        future = asyncio.ensure_future(simple_task(), loop=loop)

        # Выполняем итерации пока future не завершится
        while not future.done():
            # ВНИМАНИЕ: приватный метод!
            loop._run_once()

        print(f"Результат: {future.result()}")
    finally:
        loop.close()

# Не рекомендуется для продакшена!
# manual_iteration_example()
```

## Практические паттерны

### Паттерн: Sync-to-Async Bridge

Для вызова async-кода из синхронного контекста:

```python
import asyncio
from typing import TypeVar, Coroutine, Any

T = TypeVar('T')

def run_async(coro: Coroutine[Any, Any, T]) -> T:
    """
    Запускает корутину из синхронного кода.

    Безопасно работает даже если event loop уже существует.
    """
    try:
        # Пытаемся получить текущий event loop
        loop = asyncio.get_running_loop()
    except RuntimeError:
        # Нет запущенного loop — создаём новый
        return asyncio.run(coro)

    # Если loop уже запущен — используем его
    # Это сложный случай, требующий особого подхода
    import concurrent.futures
    import threading

    # Создаём новый event loop в отдельном потоке
    result_container = {}
    exception_container = {}

    def run_in_thread():
        new_loop = asyncio.new_event_loop()
        asyncio.set_event_loop(new_loop)
        try:
            result_container['result'] = new_loop.run_until_complete(coro)
        except Exception as e:
            exception_container['exception'] = e
        finally:
            new_loop.close()

    thread = threading.Thread(target=run_in_thread)
    thread.start()
    thread.join()

    if 'exception' in exception_container:
        raise exception_container['exception']
    return result_container['result']

# Использование
async def fetch_data():
    await asyncio.sleep(0.1)
    return {"data": "value"}

# Вызов из синхронного кода
# result = run_async(fetch_data())
# print(result)
```

### Паттерн: Chunked Processing

Обработка больших данных с периодическим освобождением event loop:

```python
import asyncio
from typing import List, TypeVar, Callable, AsyncIterator

T = TypeVar('T')
R = TypeVar('R')

async def chunked_process(
    items: List[T],
    processor: Callable[[T], R],
    chunk_size: int = 100
) -> AsyncIterator[R]:
    """
    Обрабатывает элементы чанками, освобождая event loop между чанками.
    """
    for i in range(0, len(items), chunk_size):
        chunk = items[i:i + chunk_size]

        # Обрабатываем чанк
        for item in chunk:
            yield processor(item)

        # Освобождаем event loop
        await asyncio.sleep(0)

async def main():
    data = list(range(10000))

    async for result in chunked_process(data, lambda x: x * 2, chunk_size=1000):
        pass  # Обработка результата

    print("Обработка завершена")

asyncio.run(main())
```

### Паттерн: Interruptible Long Task

Прерываемая долгая задача:

```python
import asyncio
from typing import Optional

class InterruptibleTask:
    """Задача, которую можно прервать."""

    def __init__(self):
        self._cancelled = False
        self._result: Optional[int] = None

    def cancel(self):
        """Запрос на отмену."""
        self._cancelled = True

    async def run(self, iterations: int) -> Optional[int]:
        """Выполняет вычисления с возможностью прерывания."""
        result = 0

        for i in range(iterations):
            if self._cancelled:
                print(f"Задача прервана на итерации {i}")
                return None

            result += i

            # Периодически проверяем и освобождаем event loop
            if i % 1000 == 0:
                await asyncio.sleep(0)

        self._result = result
        return result

async def cancel_after(task: InterruptibleTask, delay: float):
    """Отменяет задачу после задержки."""
    await asyncio.sleep(delay)
    task.cancel()

async def main():
    task = InterruptibleTask()

    # Запускаем задачу и отмену параллельно
    results = await asyncio.gather(
        task.run(1000000),
        cancel_after(task, 0.1),
        return_exceptions=True
    )

    print(f"Результат: {results[0]}")

asyncio.run(main())
```

## Работа с call_soon и call_later

Event loop предоставляет методы для планирования callback-функций:

```python
import asyncio

def callback_example():
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    results = []

    def my_callback(value):
        results.append(value)
        print(f"Callback: {value}")

    # call_soon — выполнить в следующей итерации
    loop.call_soon(my_callback, "first")
    loop.call_soon(my_callback, "second")

    # call_later — выполнить через указанное время
    loop.call_later(0.1, my_callback, "delayed")

    # call_at — выполнить в указанное время (относительно loop.time())
    loop.call_at(loop.time() + 0.2, my_callback, "at_time")

    # Запускаем для выполнения callbacks
    async def wait_for_callbacks():
        await asyncio.sleep(0.3)

    try:
        loop.run_until_complete(wait_for_callbacks())
        print(f"Все результаты: {results}")
    finally:
        loop.close()

callback_example()
```

## Debug-режим Event Loop

Для отладки и профилирования:

```python
import asyncio

async def slow_callback():
    """Медленный callback для демонстрации."""
    import time
    time.sleep(0.2)  # Блокирующая операция!

async def main():
    # Создаём задачу с блокирующей операцией
    await slow_callback()

# Запуск в debug-режиме
# asyncio.run(main(), debug=True)
# В debug-режиме будет предупреждение о блокирующей операции
```

### Настройка slow callback threshold

```python
import asyncio

loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)

# Устанавливаем порог для предупреждений (по умолчанию 0.1 секунды)
loop.slow_callback_duration = 0.05

# Включаем debug-режим
loop.set_debug(True)

async def demo():
    import time
    time.sleep(0.06)  # Будет предупреждение

try:
    loop.run_until_complete(demo())
finally:
    loop.close()
```

## Особенности Python 3.10+ (asyncio.Runner)

В Python 3.11+ появился новый API для управления event loop:

```python
import asyncio

async def task1():
    await asyncio.sleep(0.1)
    return "task1"

async def task2():
    await asyncio.sleep(0.1)
    return "task2"

# Python 3.11+
def using_runner():
    with asyncio.Runner() as runner:
        # Можно выполнить несколько корутин последовательно
        result1 = runner.run(task1())
        result2 = runner.run(task2())
        print(f"Результаты: {result1}, {result2}")

        # Также есть доступ к loop
        loop = runner.get_loop()
        print(f"Event loop: {loop}")

# using_runner()  # Раскомментируйте для Python 3.11+
```

## Best Practices

### 1. Предпочитайте `asyncio.sleep(0)`

```python
# ХОРОШО: явная точка переключения
await asyncio.sleep(0)

# ПЛОХО: использование приватных методов
loop._run_once()  # Может сломаться в будущих версиях
```

### 2. Не злоупотребляйте принудительными итерациями

```python
# ПЛОХО: слишком частое освобождение
for i in range(1000):
    result += i
    await asyncio.sleep(0)  # Огромный overhead!

# ХОРОШО: разумная частота
for i in range(1000):
    result += i
    if i % 100 == 0:
        await asyncio.sleep(0)
```

### 3. Используйте asyncio.run() когда возможно

```python
# ХОРОШО: asyncio.run() управляет всем
asyncio.run(main())

# СЛОЖНЕЕ: ручное управление
loop = asyncio.new_event_loop()
try:
    loop.run_until_complete(main())
finally:
    loop.close()
```

## Распространённые ошибки

### 1. Блокирующие операции в event loop

```python
import asyncio
import time

async def bad_example():
    # ПЛОХО: блокирует весь event loop!
    time.sleep(5)

async def good_example():
    # ХОРОШО: не блокирует
    await asyncio.sleep(5)

    # Или для CPU-bound операций
    loop = asyncio.get_running_loop()
    await loop.run_in_executor(None, time.sleep, 5)
```

### 2. Забытый await

```python
import asyncio

async def example():
    # ПЛОХО: корутина не выполнится!
    asyncio.sleep(1)  # Создаётся, но не await'ится

    # ХОРОШО:
    await asyncio.sleep(1)
```

### 3. Вложенный asyncio.run()

```python
import asyncio

async def inner():
    return 42

async def outer():
    # ОШИБКА: RuntimeError!
    # result = asyncio.run(inner())

    # ПРАВИЛЬНО:
    result = await inner()
    return result
```

## Резюме

- `await asyncio.sleep(0)` — основной способ освободить event loop
- Используйте `run_until_complete()` для синхронного запуска корутин
- Избегайте приватных методов вроде `_run_once()`
- Python 3.11+ предоставляет `asyncio.Runner` для удобного управления
- Не блокируйте event loop синхронными операциями
- Принудительные итерации нужны редко — обычно asyncio справляется сам

---

[prev: ./02-context-vars.md](./02-context-vars.md) | [next: ./04-alternative-loops.md](./04-alternative-loops.md)
