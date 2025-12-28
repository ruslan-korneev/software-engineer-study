# Создание собственного цикла событий

[prev: ./04-alternative-loops.md](./04-alternative-loops.md) | [next: ../../15-frameworks/01-fastapi.md](../../15-frameworks/01-fastapi.md)

---

## Введение

Создание собственного event loop — это продвинутая техника, которая позволяет глубже понять работу asyncio и создавать специализированные решения. Хотя на практике чаще используют стандартный loop или uvloop, понимание внутренней архитектуры event loop критически важно для отладки и оптимизации асинхронного кода.

## Архитектура Event Loop

### Основные компоненты

Event loop состоит из следующих ключевых элементов:

1. **Очередь готовых задач (ready queue)** — задачи, готовые к выполнению
2. **Очередь отложенных задач (scheduled)** — задачи с таймером
3. **I/O мультиплексор (selector)** — отслеживание файловых дескрипторов
4. **Обработчики сигналов** — реакция на системные сигналы

```python
import asyncio
from collections import deque
import selectors
import time
import heapq
from typing import Callable, Any, Optional

class SimpleEventLoop:
    """Упрощённая реализация event loop для понимания концепции."""

    def __init__(self):
        # Очередь готовых к выполнению callbacks
        self._ready: deque = deque()

        # Heap для отложенных callbacks (время, callback)
        self._scheduled: list = []

        # Selector для I/O мультиплексирования
        self._selector = selectors.DefaultSelector()

        # Флаг остановки
        self._stopping = False

        # Текущее время loop
        self._clock = time.monotonic

    def time(self) -> float:
        """Возвращает текущее время loop."""
        return self._clock()

    def call_soon(self, callback: Callable, *args) -> None:
        """Планирует callback для выполнения в следующей итерации."""
        self._ready.append((callback, args))

    def call_later(self, delay: float, callback: Callable, *args) -> None:
        """Планирует callback для выполнения через delay секунд."""
        when = self.time() + delay
        heapq.heappush(self._scheduled, (when, callback, args))

    def call_at(self, when: float, callback: Callable, *args) -> None:
        """Планирует callback для выполнения в указанное время."""
        heapq.heappush(self._scheduled, (when, callback, args))

    def stop(self) -> None:
        """Останавливает event loop."""
        self._stopping = True

    def _run_once(self) -> None:
        """Выполняет одну итерацию event loop."""
        # 1. Перемещаем готовые scheduled callbacks в ready queue
        now = self.time()
        while self._scheduled and self._scheduled[0][0] <= now:
            _, callback, args = heapq.heappop(self._scheduled)
            self._ready.append((callback, args))

        # 2. Вычисляем timeout для select
        if self._ready:
            timeout = 0
        elif self._scheduled:
            timeout = max(0, self._scheduled[0][0] - now)
        else:
            timeout = None  # Блокируемся навсегда

        # 3. Ждём I/O событий
        events = self._selector.select(timeout)
        for key, mask in events:
            callback = key.data
            self._ready.append((callback, ()))

        # 4. Выполняем готовые callbacks
        ntodo = len(self._ready)
        for _ in range(ntodo):
            callback, args = self._ready.popleft()
            callback(*args)

    def run_forever(self) -> None:
        """Запускает event loop до вызова stop()."""
        self._stopping = False
        while not self._stopping:
            self._run_once()

    def run_until_complete(self, coro) -> Any:
        """Запускает event loop до завершения корутины."""
        # Создаём Task для корутины
        task = Task(coro, self)

        # Запускаем loop до завершения task
        self._stopping = False
        while not task.done() and not self._stopping:
            self._run_once()

        return task.result()

    def close(self) -> None:
        """Закрывает event loop."""
        self._selector.close()
```

## Реализация Task

Task — это обёртка над корутиной, управляющая её выполнением:

```python
from typing import Generator, Any

class Task:
    """Простая реализация Task для понимания концепции."""

    def __init__(self, coro, loop: SimpleEventLoop):
        self._coro = coro
        self._loop = loop
        self._result = None
        self._exception = None
        self._done = False

        # Планируем первый шаг корутины
        self._loop.call_soon(self._step)

    def _step(self, value=None, exc=None):
        """Выполняет один шаг корутины."""
        try:
            if exc is not None:
                result = self._coro.throw(exc)
            else:
                result = self._coro.send(value)
        except StopIteration as e:
            # Корутина завершилась
            self._result = e.value
            self._done = True
        except Exception as e:
            # Корутина выбросила исключение
            self._exception = e
            self._done = True
        else:
            # Корутина вернула awaitable
            if isinstance(result, Sleep):
                # Планируем продолжение после delay
                self._loop.call_later(result.delay, self._step)
            else:
                # Неизвестный awaitable — продолжаем сразу
                self._loop.call_soon(self._step)

    def done(self) -> bool:
        """Проверяет, завершён ли Task."""
        return self._done

    def result(self) -> Any:
        """Возвращает результат Task."""
        if self._exception:
            raise self._exception
        return self._result

class Sleep:
    """Простой awaitable для задержки."""

    def __init__(self, delay: float):
        self.delay = delay

    def __await__(self):
        yield self  # Возвращаем себя для обработки в Task._step
```

## Полная реализация минимального Event Loop

```python
import asyncio
from collections import deque
import selectors
import time
import heapq
import socket
from typing import Callable, Any, Optional, Coroutine

class Handle:
    """Обёртка над callback для отмены."""

    def __init__(self, callback: Callable, args: tuple, loop: 'MinimalEventLoop'):
        self._callback = callback
        self._args = args
        self._loop = loop
        self._cancelled = False

    def cancel(self):
        """Отменяет выполнение callback."""
        self._cancelled = True

    def _run(self):
        """Выполняет callback если не отменён."""
        if not self._cancelled:
            self._callback(*self._args)

class TimerHandle(Handle):
    """Handle с временем выполнения."""

    def __init__(self, when: float, callback: Callable, args: tuple, loop: 'MinimalEventLoop'):
        super().__init__(callback, args, loop)
        self._when = when

    def __lt__(self, other):
        return self._when < other._when

class MinimalEventLoop:
    """
    Минимальная, но функциональная реализация event loop.

    Поддерживает:
    - call_soon, call_later, call_at
    - run_until_complete, run_forever
    - add_reader, remove_reader
    - create_task
    """

    def __init__(self):
        self._ready: deque[Handle] = deque()
        self._scheduled: list[TimerHandle] = []
        self._selector = selectors.DefaultSelector()
        self._stopping = False
        self._running = False
        self._current_handle: Optional[Handle] = None

    def time(self) -> float:
        """Монотонное время loop."""
        return time.monotonic()

    # === Планирование callbacks ===

    def call_soon(self, callback: Callable, *args) -> Handle:
        """Планирует callback на следующую итерацию."""
        handle = Handle(callback, args, self)
        self._ready.append(handle)
        return handle

    def call_later(self, delay: float, callback: Callable, *args) -> TimerHandle:
        """Планирует callback через delay секунд."""
        when = self.time() + delay
        return self.call_at(when, callback, *args)

    def call_at(self, when: float, callback: Callable, *args) -> TimerHandle:
        """Планирует callback на указанное время."""
        handle = TimerHandle(when, callback, args, self)
        heapq.heappush(self._scheduled, handle)
        return handle

    # === I/O мониторинг ===

    def add_reader(self, fd: int, callback: Callable, *args) -> None:
        """Добавляет callback для чтения из fd."""
        handle = Handle(callback, args, self)
        try:
            key = self._selector.get_key(fd)
            # Обновляем существующий
            mask = key.events | selectors.EVENT_READ
            self._selector.modify(fd, mask, handle)
        except KeyError:
            self._selector.register(fd, selectors.EVENT_READ, handle)

    def remove_reader(self, fd: int) -> bool:
        """Удаляет callback для чтения из fd."""
        try:
            key = self._selector.get_key(fd)
            mask = key.events & ~selectors.EVENT_READ
            if mask:
                self._selector.modify(fd, mask, key.data)
            else:
                self._selector.unregister(fd)
            return True
        except KeyError:
            return False

    def add_writer(self, fd: int, callback: Callable, *args) -> None:
        """Добавляет callback для записи в fd."""
        handle = Handle(callback, args, self)
        try:
            key = self._selector.get_key(fd)
            mask = key.events | selectors.EVENT_WRITE
            self._selector.modify(fd, mask, handle)
        except KeyError:
            self._selector.register(fd, selectors.EVENT_WRITE, handle)

    def remove_writer(self, fd: int) -> bool:
        """Удаляет callback для записи в fd."""
        try:
            key = self._selector.get_key(fd)
            mask = key.events & ~selectors.EVENT_WRITE
            if mask:
                self._selector.modify(fd, mask, key.data)
            else:
                self._selector.unregister(fd)
            return True
        except KeyError:
            return False

    # === Выполнение ===

    def _run_once(self) -> None:
        """Одна итерация event loop."""
        # Перемещаем готовые таймеры
        now = self.time()
        while self._scheduled:
            handle = self._scheduled[0]
            if handle._when > now:
                break
            heapq.heappop(self._scheduled)
            if not handle._cancelled:
                self._ready.append(handle)

        # Вычисляем timeout
        if self._ready:
            timeout = 0
        elif self._scheduled:
            timeout = max(0, self._scheduled[0]._when - now)
        else:
            timeout = None

        # Ждём I/O
        events = self._selector.select(timeout)
        for key, mask in events:
            handle = key.data
            if not handle._cancelled:
                self._ready.append(handle)

        # Выполняем callbacks
        ntodo = len(self._ready)
        for _ in range(ntodo):
            handle = self._ready.popleft()
            self._current_handle = handle
            handle._run()
            self._current_handle = None

    def run_forever(self) -> None:
        """Запускает loop до вызова stop()."""
        if self._running:
            raise RuntimeError("Loop уже запущен")

        self._running = True
        self._stopping = False

        try:
            while not self._stopping:
                self._run_once()
        finally:
            self._running = False

    def run_until_complete(self, future) -> Any:
        """Запускает loop до завершения future."""
        from asyncio import ensure_future, Future

        # Преобразуем в Future если нужно
        if not isinstance(future, Future):
            future = ensure_future(future, loop=self)

        # Добавляем callback для остановки
        future.add_done_callback(lambda f: self.stop())

        try:
            self.run_forever()
        finally:
            future.remove_done_callback(lambda f: self.stop())

        return future.result()

    def stop(self) -> None:
        """Останавливает loop."""
        self._stopping = True

    def close(self) -> None:
        """Закрывает loop."""
        if self._running:
            raise RuntimeError("Нельзя закрыть работающий loop")
        self._selector.close()

    def is_running(self) -> bool:
        """Проверяет, запущен ли loop."""
        return self._running

    def is_closed(self) -> bool:
        """Проверяет, закрыт ли loop."""
        return self._selector._map is None

    # === Создание задач ===

    def create_task(self, coro: Coroutine) -> 'asyncio.Task':
        """Создаёт Task из корутины."""
        from asyncio import Task
        task = Task(coro, loop=self)
        return task

    def create_future(self) -> 'asyncio.Future':
        """Создаёт Future."""
        from asyncio import Future
        return Future(loop=self)

# Демонстрация использования
async def demo():
    print("Начало")
    await asyncio.sleep(1)
    print("После sleep")
    return 42

# Использование нашего loop
# loop = MinimalEventLoop()
# result = loop.run_until_complete(demo())
# print(f"Результат: {result}")
# loop.close()
```

## Реализация EventLoopPolicy

Для интеграции с asyncio нужно реализовать `EventLoopPolicy`:

```python
import asyncio
from typing import Optional

class CustomEventLoopPolicy(asyncio.AbstractEventLoopPolicy):
    """Политика для использования нашего event loop."""

    def __init__(self):
        self._local = asyncio.threading.local()

    def get_event_loop(self) -> MinimalEventLoop:
        """Получает event loop для текущего потока."""
        try:
            return self._local.loop
        except AttributeError:
            loop = self.new_event_loop()
            self.set_event_loop(loop)
            return loop

    def set_event_loop(self, loop: Optional[MinimalEventLoop]) -> None:
        """Устанавливает event loop для текущего потока."""
        self._local.loop = loop

    def new_event_loop(self) -> MinimalEventLoop:
        """Создаёт новый event loop."""
        return MinimalEventLoop()

# Использование
# asyncio.set_event_loop_policy(CustomEventLoopPolicy())
# asyncio.run(demo())
```

## Расширение стандартного Event Loop

Часто проще расширить существующий loop:

```python
import asyncio
from asyncio import SelectorEventLoop

class ExtendedEventLoop(SelectorEventLoop):
    """Расширенный event loop с дополнительными возможностями."""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._metrics = {
            'tasks_created': 0,
            'callbacks_executed': 0,
            'io_wait_time': 0.0,
        }

    def create_task(self, coro, *, name=None):
        """Создаёт task с метриками."""
        self._metrics['tasks_created'] += 1
        return super().create_task(coro, name=name)

    def call_soon(self, callback, *args, context=None):
        """Планирует callback с метриками."""
        self._metrics['callbacks_executed'] += 1
        return super().call_soon(callback, *args, context=context)

    def get_metrics(self) -> dict:
        """Возвращает метрики loop."""
        return self._metrics.copy()

    def print_metrics(self):
        """Выводит метрики."""
        print("=== Event Loop Metrics ===")
        for key, value in self._metrics.items():
            print(f"  {key}: {value}")

class ExtendedEventLoopPolicy(asyncio.DefaultEventLoopPolicy):
    """Политика для расширенного loop."""

    def new_event_loop(self):
        return ExtendedEventLoop()

# Пример использования
async def example():
    await asyncio.sleep(0.1)
    await asyncio.gather(
        asyncio.sleep(0.1),
        asyncio.sleep(0.1),
    )

asyncio.set_event_loop_policy(ExtendedEventLoopPolicy())
loop = asyncio.new_event_loop()
asyncio.set_event_loop(loop)

try:
    loop.run_until_complete(example())
    loop.print_metrics()
finally:
    loop.close()
```

## Практический пример: Трассировка Event Loop

```python
import asyncio
import time
from functools import wraps

class TracingEventLoop(asyncio.SelectorEventLoop):
    """Event loop с трассировкой выполнения."""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._trace_log = []

    def _trace(self, event: str, details: str = ""):
        """Записывает событие в лог."""
        timestamp = time.monotonic()
        self._trace_log.append({
            'time': timestamp,
            'event': event,
            'details': details
        })

    def call_soon(self, callback, *args, context=None):
        self._trace('call_soon', f"{callback.__name__ if hasattr(callback, '__name__') else str(callback)}")
        return super().call_soon(callback, *args, context=context)

    def call_later(self, delay, callback, *args, context=None):
        self._trace('call_later', f"delay={delay}, {callback.__name__ if hasattr(callback, '__name__') else str(callback)}")
        return super().call_later(delay, callback, *args, context=context)

    def create_task(self, coro, *, name=None):
        coro_name = coro.__qualname__ if hasattr(coro, '__qualname__') else str(coro)
        self._trace('create_task', coro_name)
        return super().create_task(coro, name=name)

    def get_trace(self) -> list:
        """Возвращает лог трассировки."""
        return self._trace_log.copy()

    def print_trace(self):
        """Выводит трассировку."""
        print("=== Event Loop Trace ===")
        start_time = self._trace_log[0]['time'] if self._trace_log else 0
        for entry in self._trace_log:
            relative_time = entry['time'] - start_time
            print(f"[{relative_time:.4f}s] {entry['event']}: {entry['details']}")

# Пример использования
async def task_a():
    await asyncio.sleep(0.1)
    return "A"

async def task_b():
    await asyncio.sleep(0.05)
    return "B"

async def main():
    results = await asyncio.gather(task_a(), task_b())
    return results

loop = TracingEventLoop()
asyncio.set_event_loop(loop)

try:
    result = loop.run_until_complete(main())
    print(f"\nРезультат: {result}")
    loop.print_trace()
finally:
    loop.close()
```

## Event Loop для специфических задач

### Event Loop с приоритетами

```python
import asyncio
import heapq
from dataclasses import dataclass, field
from typing import Any, Callable

@dataclass(order=True)
class PrioritizedHandle:
    priority: int
    counter: int
    handle: Any = field(compare=False)

class PriorityEventLoop(asyncio.SelectorEventLoop):
    """Event loop с поддержкой приоритетов."""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self._priority_ready = []
        self._counter = 0

    def call_soon_with_priority(self, priority: int, callback: Callable, *args) -> None:
        """Планирует callback с указанным приоритетом (меньше = выше)."""
        handle = asyncio.Handle(callback, args, self)
        self._counter += 1
        heapq.heappush(
            self._priority_ready,
            PrioritizedHandle(priority, self._counter, handle)
        )

    def _run_once(self):
        # Сначала обрабатываем приоритетные callbacks
        while self._priority_ready:
            item = heapq.heappop(self._priority_ready)
            if not item.handle._cancelled:
                item.handle._run()

        # Затем стандартная обработка
        super()._run_once()

# Использование
async def high_priority_task():
    print("Высокий приоритет!")

async def low_priority_task():
    print("Низкий приоритет!")

loop = PriorityEventLoop()
asyncio.set_event_loop(loop)

loop.call_soon_with_priority(1, lambda: print("Priority 1"))
loop.call_soon_with_priority(10, lambda: print("Priority 10"))
loop.call_soon_with_priority(5, lambda: print("Priority 5"))

async def demo():
    await asyncio.sleep(0)

try:
    loop.run_until_complete(demo())
finally:
    loop.close()
```

## Best Practices

### 1. Не создавайте свой event loop без необходимости

```python
# ХОРОШО: используйте стандартный или uvloop
import asyncio
# import uvloop; uvloop.install()

asyncio.run(main())

# ПЛОХО (для большинства случаев): свой loop
# loop = MyCustomEventLoop()
# loop.run_until_complete(main())
```

### 2. При расширении наследуйтесь от SelectorEventLoop

```python
# ХОРОШО: расширяем существующий
class MyLoop(asyncio.SelectorEventLoop):
    def create_task(self, coro, **kwargs):
        # Добавляем логику
        return super().create_task(coro, **kwargs)

# НЕ РЕКОМЕНДУЕТСЯ: полная реимплементация
class MyLoop:
    def run_until_complete(self, coro):
        # Много сложного кода...
        pass
```

### 3. Тестируйте совместимость с asyncio

```python
import asyncio

def test_custom_loop_compatibility():
    """Тест совместимости кастомного loop."""
    loop = MyCustomEventLoop()
    asyncio.set_event_loop(loop)

    try:
        # Проверяем базовые операции
        async def test_suite():
            # Tasks
            task = asyncio.create_task(asyncio.sleep(0.01))
            await task

            # gather
            await asyncio.gather(asyncio.sleep(0.01), asyncio.sleep(0.01))

            # wait_for
            await asyncio.wait_for(asyncio.sleep(0.01), timeout=1.0)

            return "OK"

        result = loop.run_until_complete(test_suite())
        assert result == "OK", "Тест не пройден"
        print("Все тесты пройдены!")
    finally:
        loop.close()
```

## Распространённые ошибки

### 1. Неполная реализация интерфейса

```python
# ОШИБКА: забыли реализовать метод
class IncompleteLoop:
    def run_until_complete(self, coro):
        pass
    # Забыли call_soon, call_later и т.д.

# ПРАВИЛЬНО: наследуемся от базового класса
class CompleteLoop(asyncio.SelectorEventLoop):
    # Переопределяем только нужные методы
    pass
```

### 2. Неправильная обработка исключений

```python
# ОШИБКА: исключение теряется
def _run_once(self):
    for handle in self._ready:
        handle._run()  # Если упадёт — loop остановится

# ПРАВИЛЬНО: обрабатываем исключения
def _run_once(self):
    for handle in self._ready:
        try:
            handle._run()
        except Exception as e:
            self.call_exception_handler({
                'message': 'Exception in callback',
                'exception': e,
            })
```

## Резюме

- Создание своего event loop требуется редко — используйте стандартный или uvloop
- Для расширения наследуйтесь от `SelectorEventLoop`
- Event loop состоит из ready queue, scheduled heap и selector
- Реализуйте `EventLoopPolicy` для интеграции с asyncio
- Тестируйте совместимость со стандартными asyncio операциями
- Трассировка и метрики — полезные расширения для отладки

---

[prev: ./04-alternative-loops.md](./04-alternative-loops.md) | [next: ../../15-frameworks/01-fastapi.md](../../15-frameworks/01-fastapi.md)
