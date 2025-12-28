# API, допускающие сопрограммы (Awaitable API)

[prev: ./readme.md](./readme.md) | [next: ./02-context-vars.md](./02-context-vars.md)

---

## Введение

В Python asyncio существует концепция **awaitable-объектов** — это объекты, которые можно использовать с ключевым словом `await`. Понимание того, как создавать собственные awaitable-объекты, открывает широкие возможности для построения гибких асинхронных API.

## Что такое Awaitable?

Awaitable — это любой объект, который можно "ожидать" с помощью `await`. В Python существует три основных типа awaitable-объектов:

1. **Корутины (Coroutines)** — функции, определённые с помощью `async def`
2. **Задачи (Tasks)** — обёртки над корутинами для параллельного выполнения
3. **Футуры (Futures)** — низкоуровневые awaitable-объекты для результатов операций

```python
import asyncio
from collections.abc import Awaitable

# Корутина — это awaitable
async def my_coroutine():
    return 42

# Проверка типа
coro = my_coroutine()
print(isinstance(coro, Awaitable))  # True
coro.close()  # Закрываем неиспользованную корутину
```

## Протокол `__await__`

Любой класс может стать awaitable, если он реализует метод `__await__`. Этот метод должен возвращать **итератор**.

### Базовый пример

```python
import asyncio

class SimpleAwaitable:
    """Простой awaitable-объект, возвращающий значение."""

    def __init__(self, value):
        self.value = value

    def __await__(self):
        # Возвращаем итератор, который сразу завершается
        return iter([])

    # Альтернативный способ — делегировать к корутине
    # def __await__(self):
    #     return self._async_impl().__await__()

    # async def _async_impl(self):
    #     return self.value

async def main():
    result = await SimpleAwaitable(42)
    print(f"Результат: {result}")

asyncio.run(main())
```

### Awaitable с задержкой

```python
import asyncio

class DelayedValue:
    """Awaitable, который возвращает значение после задержки."""

    def __init__(self, value, delay: float):
        self.value = value
        self.delay = delay

    def __await__(self):
        # Делегируем к корутине через её __await__
        return self._wait().__await__()

    async def _wait(self):
        await asyncio.sleep(self.delay)
        return self.value

async def main():
    print("Начало ожидания...")
    result = await DelayedValue("Готово!", 2.0)
    print(f"Результат: {result}")

asyncio.run(main())
```

## Класс `asyncio.Future`

`Future` — это низкоуровневый awaitable-объект, представляющий результат асинхронной операции, который будет доступен в будущем.

### Работа с Future

```python
import asyncio

async def set_future_result(future: asyncio.Future, value, delay: float):
    """Устанавливает результат future после задержки."""
    await asyncio.sleep(delay)
    future.set_result(value)

async def main():
    # Создаём Future
    loop = asyncio.get_running_loop()
    future = loop.create_future()

    # Запускаем задачу, которая установит результат
    asyncio.create_task(set_future_result(future, "Результат готов", 1.0))

    # Ожидаем Future
    print("Ожидаем результат...")
    result = await future
    print(f"Получили: {result}")

asyncio.run(main())
```

### Future с исключением

```python
import asyncio

async def set_future_exception(future: asyncio.Future, delay: float):
    """Устанавливает исключение в future."""
    await asyncio.sleep(delay)
    future.set_exception(ValueError("Произошла ошибка!"))

async def main():
    loop = asyncio.get_running_loop()
    future = loop.create_future()

    asyncio.create_task(set_future_exception(future, 0.5))

    try:
        result = await future
    except ValueError as e:
        print(f"Поймали исключение: {e}")

asyncio.run(main())
```

## Создание сложных Awaitable-объектов

### Awaitable с повторными попытками

```python
import asyncio
import random

class RetryableOperation:
    """Awaitable с автоматическими повторными попытками."""

    def __init__(self, operation, max_retries: int = 3, delay: float = 1.0):
        self.operation = operation
        self.max_retries = max_retries
        self.delay = delay

    def __await__(self):
        return self._execute_with_retry().__await__()

    async def _execute_with_retry(self):
        last_exception = None

        for attempt in range(self.max_retries):
            try:
                return await self.operation()
            except Exception as e:
                last_exception = e
                print(f"Попытка {attempt + 1} не удалась: {e}")
                if attempt < self.max_retries - 1:
                    await asyncio.sleep(self.delay * (attempt + 1))

        raise last_exception

async def unreliable_operation():
    """Операция, которая случайно падает."""
    if random.random() < 0.7:
        raise ConnectionError("Соединение потеряно")
    return "Успех!"

async def main():
    result = await RetryableOperation(unreliable_operation, max_retries=5)
    print(f"Результат: {result}")

asyncio.run(main())
```

### Awaitable-обёртка для синхронного кода

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor

class AsyncWrapper:
    """Превращает синхронную функцию в awaitable."""

    def __init__(self, sync_func, *args, **kwargs):
        self.sync_func = sync_func
        self.args = args
        self.kwargs = kwargs
        self._executor = ThreadPoolExecutor(max_workers=1)

    def __await__(self):
        return self._run_in_executor().__await__()

    async def _run_in_executor(self):
        loop = asyncio.get_running_loop()
        return await loop.run_in_executor(
            self._executor,
            lambda: self.sync_func(*self.args, **self.kwargs)
        )

def heavy_computation(n: int) -> int:
    """Синхронная тяжёлая операция."""
    import time
    time.sleep(1)  # Имитация работы
    return sum(i * i for i in range(n))

async def main():
    print("Запускаем вычисление...")
    result = await AsyncWrapper(heavy_computation, 1000000)
    print(f"Результат: {result}")

asyncio.run(main())
```

## Awaitable как контекстный менеджер

Можно комбинировать `__await__` с протоколом контекстного менеджера:

```python
import asyncio

class AsyncResource:
    """Асинхронный ресурс с awaitable-инициализацией."""

    def __init__(self, name: str):
        self.name = name
        self._initialized = False

    def __await__(self):
        return self._initialize().__await__()

    async def _initialize(self):
        print(f"Инициализация ресурса '{self.name}'...")
        await asyncio.sleep(0.5)
        self._initialized = True
        print(f"Ресурс '{self.name}' готов!")
        return self

    async def __aenter__(self):
        if not self._initialized:
            await self
        return self

    async def __aexit__(self, exc_type, exc_val, exc_tb):
        print(f"Закрытие ресурса '{self.name}'")
        await asyncio.sleep(0.1)
        self._initialized = False

async def main():
    # Использование через await
    resource = await AsyncResource("database")

    # Или через async with
    async with AsyncResource("cache") as cache:
        print(f"Работаем с {cache.name}")

asyncio.run(main())
```

## Best Practices

### 1. Делегирование к корутинам

Самый надёжный способ реализации `__await__` — делегировать к корутине:

```python
class MyAwaitable:
    def __await__(self):
        return self._async_logic().__await__()

    async def _async_logic(self):
        # Вся асинхронная логика здесь
        await asyncio.sleep(1)
        return "result"
```

### 2. Проверка типа Awaitable

```python
from collections.abc import Awaitable
import inspect

def is_awaitable(obj):
    """Проверяет, является ли объект awaitable."""
    return isinstance(obj, Awaitable) or inspect.iscoroutine(obj)
```

### 3. Не забывайте закрывать корутины

```python
async def main():
    coro = my_coroutine()

    # Если корутина не будет await'иться, её нужно закрыть
    if some_condition:
        coro.close()
    else:
        result = await coro
```

## Распространённые ошибки

### 1. Бесконечный итератор в `__await__`

```python
# НЕПРАВИЛЬНО — зависнет навсегда
class BadAwaitable:
    def __await__(self):
        while True:
            yield

# ПРАВИЛЬНО — итератор должен завершаться
class GoodAwaitable:
    def __await__(self):
        return iter([])
```

### 2. Попытка await синхронного объекта

```python
# НЕПРАВИЛЬНО — TypeError
result = await 42

# ПРАВИЛЬНО — оборачиваем в корутину
async def async_value(value):
    return value

result = await async_value(42)
```

## Резюме

- Awaitable — это объект, который можно использовать с `await`
- Для создания awaitable класса реализуйте метод `__await__`
- `__await__` должен возвращать итератор
- Лучший подход — делегировать логику `__await__` к корутине
- `Future` — низкоуровневый awaitable для представления будущих результатов
- Awaitable-объекты можно комбинировать с другими протоколами Python

---

[prev: ./readme.md](./readme.md) | [next: ./02-context-vars.md](./02-context-vars.md)
