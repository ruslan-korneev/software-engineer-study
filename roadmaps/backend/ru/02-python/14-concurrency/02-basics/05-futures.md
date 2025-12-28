# Задачи, сопрограммы и futures

[prev: ./04-cancellation-timeouts.md](./04-cancellation-timeouts.md) | [next: ./06-timing-decorators.md](./06-timing-decorators.md)

---

## Что такое Future?

**Future** - это объект, представляющий результат асинхронной операции, который будет доступен в будущем. Future - это низкоуровневый примитив asyncio, который редко используется напрямую в прикладном коде.

### Иерархия awaitable-объектов

```
Awaitable (абстрактный базовый класс)
├── Coroutine (корутина)
├── Future (будущий результат)
│   └── Task (задача = Future + корутина)
└── Другие awaitable объекты
```

## Создание и использование Future

### Базовое использование

```python
import asyncio

async def main():
    # Создаём Future
    loop = asyncio.get_event_loop()
    future = loop.create_future()

    # Future ещё не имеет результата
    print(f"Future done: {future.done()}")

    # Устанавливаем результат
    future.set_result("Готово!")

    # Теперь результат доступен
    print(f"Future done: {future.done()}")
    result = await future
    print(f"Результат: {result}")

asyncio.run(main())
```

### Асинхронная установка результата

```python
import asyncio

async def set_result_later(future: asyncio.Future, delay: float, value):
    """Устанавливает результат Future после задержки."""
    await asyncio.sleep(delay)
    future.set_result(value)
    print(f"Результат установлен: {value}")

async def main():
    future = asyncio.get_event_loop().create_future()

    # Запускаем задачу, которая установит результат
    asyncio.create_task(set_result_later(future, 2, "данные"))

    print("Ожидаю результат...")
    result = await future
    print(f"Получен результат: {result}")

asyncio.run(main())
```

## Future vs Task vs Coroutine

### Корутина (Coroutine)

```python
import asyncio

async def my_coroutine():
    """Корутина - это функция с async def."""
    await asyncio.sleep(1)
    return "результат корутины"

async def main():
    # Корутина НЕ начинает выполняться при создании
    coro = my_coroutine()
    print(f"Тип: {type(coro)}")

    # Выполняется только при await
    result = await coro
    print(result)

asyncio.run(main())
```

### Task (Задача)

```python
import asyncio

async def my_coroutine():
    await asyncio.sleep(1)
    return "результат задачи"

async def main():
    # Task - это Future + корутина
    # Task НАЧИНАЕТ выполняться сразу при создании
    task = asyncio.create_task(my_coroutine())
    print(f"Тип: {type(task)}")
    print(f"Task - это Future: {isinstance(task, asyncio.Future)}")

    result = await task
    print(result)

asyncio.run(main())
```

### Future (Будущий результат)

```python
import asyncio

async def main():
    # Future - низкоуровневый примитив
    # НЕ содержит логики выполнения, только результат
    future = asyncio.get_event_loop().create_future()
    print(f"Тип: {type(future)}")

    # Результат должен быть установлен извне
    future.set_result("результат future")

    result = await future
    print(result)

asyncio.run(main())
```

## Сравнительная таблица

| Характеристика | Coroutine | Task | Future |
|----------------|-----------|------|--------|
| Создание | `async def` | `create_task()` | `create_future()` |
| Начало выполнения | При `await` | Сразу | Не выполняется |
| Содержит логику | Да | Да (обёртка над корутиной) | Нет |
| Можно отменить | Нет | Да | Да |
| Результат | `return` | `return` | `set_result()` |

## Методы Future

### Установка результата и исключения

```python
import asyncio

async def main():
    loop = asyncio.get_event_loop()

    # Future с успешным результатом
    future1 = loop.create_future()
    future1.set_result("успех")
    print(f"Результат: {await future1}")

    # Future с исключением
    future2 = loop.create_future()
    future2.set_exception(ValueError("ошибка"))

    try:
        await future2
    except ValueError as e:
        print(f"Исключение: {e}")

asyncio.run(main())
```

### Проверка состояния

```python
import asyncio

async def main():
    future = asyncio.get_event_loop().create_future()

    # Проверка состояния
    print(f"done(): {future.done()}")
    print(f"cancelled(): {future.cancelled()}")

    # После установки результата
    future.set_result(42)
    print(f"done(): {future.done()}")
    print(f"result(): {future.result()}")

    # Попытка получить исключение (если его нет - None)
    print(f"exception(): {future.exception()}")

asyncio.run(main())
```

### Callbacks

```python
import asyncio

def on_future_done(future: asyncio.Future):
    """Callback, вызываемый при завершении Future."""
    if future.cancelled():
        print("Future отменён")
    elif future.exception():
        print(f"Future завершился с ошибкой: {future.exception()}")
    else:
        print(f"Future завершился с результатом: {future.result()}")

async def main():
    future = asyncio.get_event_loop().create_future()

    # Добавляем callback
    future.add_done_callback(on_future_done)

    # Устанавливаем результат - callback вызовется автоматически
    future.set_result("готово")

    await future

asyncio.run(main())
```

## Практические применения Future

### Паттерн: Обещание результата (Promise-like)

```python
import asyncio

class AsyncResult:
    """Класс для передачи результата между корутинами."""

    def __init__(self):
        self._future = asyncio.get_event_loop().create_future()

    def set_result(self, value):
        if not self._future.done():
            self._future.set_result(value)

    def set_exception(self, exc):
        if not self._future.done():
            self._future.set_exception(exc)

    async def get(self):
        return await self._future

async def producer(result: AsyncResult):
    print("Producer: обрабатываю данные...")
    await asyncio.sleep(2)
    result.set_result({"data": "важные данные"})
    print("Producer: результат установлен")

async def consumer(result: AsyncResult):
    print("Consumer: жду результат...")
    data = await result.get()
    print(f"Consumer: получил {data}")

async def main():
    result = AsyncResult()

    await asyncio.gather(
        producer(result),
        consumer(result)
    )

asyncio.run(main())
```

### Паттерн: Синхронизация между потоками

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

def blocking_operation():
    """Блокирующая операция в отдельном потоке."""
    import time
    time.sleep(2)
    return "результат из потока"

async def main():
    loop = asyncio.get_event_loop()

    # Создаём Future для получения результата из потока
    future = loop.create_future()

    def thread_worker():
        result = blocking_operation()
        # Безопасно устанавливаем результат из другого потока
        loop.call_soon_threadsafe(future.set_result, result)

    # Запускаем в потоке
    with ThreadPoolExecutor() as pool:
        pool.submit(thread_worker)

    # Ожидаем результат
    result = await future
    print(f"Получен результат: {result}")

asyncio.run(main())
```

### Паттерн: Event-подобный механизм с данными

```python
import asyncio
from typing import Any, Optional

class AsyncEvent:
    """Event с возможностью передачи данных."""

    def __init__(self):
        self._future: Optional[asyncio.Future] = None
        self._loop = asyncio.get_event_loop()

    def _reset(self):
        self._future = self._loop.create_future()

    def set(self, data: Any = None):
        """Сигнализирует о событии с данными."""
        if self._future and not self._future.done():
            self._future.set_result(data)

    async def wait(self) -> Any:
        """Ожидает события и возвращает данные."""
        self._reset()
        return await self._future

async def waiter(event: AsyncEvent, name: str):
    print(f"{name}: жду события...")
    data = await event.wait()
    print(f"{name}: получил данные: {data}")

async def setter(event: AsyncEvent):
    await asyncio.sleep(2)
    print("Setter: устанавливаю событие")
    event.set({"message": "hello", "code": 200})

async def main():
    event = AsyncEvent()

    await asyncio.gather(
        waiter(event, "Waiter1"),
        setter(event)
    )

asyncio.run(main())
```

## asyncio.ensure_future()

`ensure_future()` преобразует awaitable в Task или Future:

```python
import asyncio

async def my_coroutine():
    await asyncio.sleep(1)
    return "результат"

async def main():
    # С корутиной - создаёт Task
    coro = my_coroutine()
    task = asyncio.ensure_future(coro)
    print(f"Из корутины: {type(task)}")  # Task

    # С Future - возвращает тот же Future
    future = asyncio.get_event_loop().create_future()
    same_future = asyncio.ensure_future(future)
    print(f"Тот же объект: {future is same_future}")  # True

    # С Task - возвращает тот же Task
    task2 = asyncio.create_task(my_coroutine())
    same_task = asyncio.ensure_future(task2)
    print(f"Тот же объект: {task2 is same_task}")  # True

    # Очистка
    future.set_result(None)
    await asyncio.gather(task, same_future, same_task)

asyncio.run(main())
```

## concurrent.futures интеграция

asyncio может работать с `Future` из модуля `concurrent.futures`:

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor

def cpu_bound_task(n: int) -> int:
    """CPU-интенсивная задача."""
    return sum(i * i for i in range(n))

def io_bound_task(url: str) -> str:
    """I/O-интенсивная задача."""
    import time
    time.sleep(1)  # Имитация сетевого запроса
    return f"Данные с {url}"

async def main():
    loop = asyncio.get_event_loop()

    # Использование ThreadPoolExecutor для I/O
    with ThreadPoolExecutor() as thread_pool:
        future = loop.run_in_executor(
            thread_pool,
            io_bound_task,
            "https://example.com"
        )
        result = await future
        print(f"I/O результат: {result}")

    # Использование ProcessPoolExecutor для CPU
    with ProcessPoolExecutor() as process_pool:
        future = loop.run_in_executor(
            process_pool,
            cpu_bound_task,
            1000000
        )
        result = await future
        print(f"CPU результат: {result}")

asyncio.run(main())
```

## Распространённые ошибки

### Ошибка 1: Повторная установка результата

```python
import asyncio

async def main():
    future = asyncio.get_event_loop().create_future()

    future.set_result("первый")

    try:
        future.set_result("второй")  # InvalidStateError!
    except asyncio.InvalidStateError:
        print("Нельзя установить результат дважды")

asyncio.run(main())
```

### Ошибка 2: Получение результата до готовности

```python
import asyncio

async def main():
    future = asyncio.get_event_loop().create_future()

    try:
        result = future.result()  # InvalidStateError!
    except asyncio.InvalidStateError:
        print("Future ещё не готов")

    # Правильно - через await
    asyncio.create_task(set_later(future))
    result = await future
    print(result)

async def set_later(future):
    await asyncio.sleep(0.1)
    future.set_result("готово")

asyncio.run(main())
```

## Best Practices

1. **Используйте Task вместо Future** в большинстве случаев
2. **Future - для низкоуровневой интеграции** (потоки, callback-based API)
3. **Проверяйте состояние** перед установкой результата
4. **Используйте `run_in_executor`** для блокирующих операций
5. **Не создавайте Future напрямую** - используйте `loop.create_future()`

```python
import asyncio

async def good_practice_example():
    loop = asyncio.get_event_loop()

    # Правильное создание Future
    future = loop.create_future()

    async def worker():
        await asyncio.sleep(1)
        if not future.done():
            future.set_result("готово")

    asyncio.create_task(worker())

    result = await future
    return result

asyncio.run(good_practice_example())
```

---

[prev: ./04-cancellation-timeouts.md](./04-cancellation-timeouts.md) | [next: ./06-timing-decorators.md](./06-timing-decorators.md)
