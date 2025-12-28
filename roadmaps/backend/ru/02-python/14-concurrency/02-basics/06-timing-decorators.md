# Измерение времени с декораторами

[prev: ./05-futures.md](./05-futures.md) | [next: ./07-pitfalls.md](./07-pitfalls.md)

---

## Введение

Измерение времени выполнения асинхронного кода - важный аспект оптимизации и отладки. Декораторы позволяют элегантно добавлять замер времени без изменения основной логики функций.

## Базовый декоратор для измерения времени

### Синхронная версия

```python
import time
from functools import wraps

def sync_timer(func):
    """Декоратор для измерения времени синхронных функций."""
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} выполнена за {elapsed:.4f} сек")
        return result
    return wrapper

@sync_timer
def slow_sync_function():
    time.sleep(1)
    return "готово"

result = slow_sync_function()
# Вывод: slow_sync_function выполнена за 1.0012 сек
```

### Асинхронная версия

```python
import asyncio
import time
from functools import wraps

def async_timer(func):
    """Декоратор для измерения времени асинхронных функций."""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = await func(*args, **kwargs)
        elapsed = time.perf_counter() - start
        print(f"{func.__name__} выполнена за {elapsed:.4f} сек")
        return result
    return wrapper

@async_timer
async def slow_async_function():
    await asyncio.sleep(1)
    return "готово"

asyncio.run(slow_async_function())
# Вывод: slow_async_function выполнена за 1.0010 сек
```

## Универсальный декоратор

Декоратор, работающий как с синхронными, так и с асинхронными функциями:

```python
import asyncio
import time
from functools import wraps
from typing import Callable, Any

def timer(func: Callable) -> Callable:
    """Универсальный декоратор для измерения времени."""
    if asyncio.iscoroutinefunction(func):
        @wraps(func)
        async def async_wrapper(*args, **kwargs) -> Any:
            start = time.perf_counter()
            result = await func(*args, **kwargs)
            elapsed = time.perf_counter() - start
            print(f"[ASYNC] {func.__name__}: {elapsed:.4f} сек")
            return result
        return async_wrapper
    else:
        @wraps(func)
        def sync_wrapper(*args, **kwargs) -> Any:
            start = time.perf_counter()
            result = func(*args, **kwargs)
            elapsed = time.perf_counter() - start
            print(f"[SYNC] {func.__name__}: {elapsed:.4f} сек")
            return result
        return sync_wrapper

@timer
def sync_operation():
    time.sleep(0.5)
    return "sync"

@timer
async def async_operation():
    await asyncio.sleep(0.5)
    return "async"

# Тест
sync_operation()
asyncio.run(async_operation())
```

## Декоратор с параметрами

### С настраиваемым выводом

```python
import asyncio
import time
import logging
from functools import wraps
from typing import Callable, Optional

def timed(
    name: Optional[str] = None,
    logger: Optional[logging.Logger] = None,
    level: int = logging.INFO
):
    """
    Декоратор с параметрами для измерения времени.

    Args:
        name: Имя для вывода (по умолчанию - имя функции)
        logger: Logger для вывода (по умолчанию - print)
        level: Уровень логирования
    """
    def decorator(func: Callable) -> Callable:
        func_name = name or func.__name__

        def log_time(elapsed: float):
            message = f"{func_name} выполнена за {elapsed:.4f} сек"
            if logger:
                logger.log(level, message)
            else:
                print(message)

        if asyncio.iscoroutinefunction(func):
            @wraps(func)
            async def async_wrapper(*args, **kwargs):
                start = time.perf_counter()
                try:
                    return await func(*args, **kwargs)
                finally:
                    log_time(time.perf_counter() - start)
            return async_wrapper
        else:
            @wraps(func)
            def sync_wrapper(*args, **kwargs):
                start = time.perf_counter()
                try:
                    return func(*args, **kwargs)
                finally:
                    log_time(time.perf_counter() - start)
            return sync_wrapper

    return decorator

# Использование
logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@timed(name="Загрузка данных", logger=logger)
async def fetch_data():
    await asyncio.sleep(1)
    return "данные"

asyncio.run(fetch_data())
```

### С порогом предупреждения

```python
import asyncio
import time
from functools import wraps
from typing import Callable, Optional
import logging

def timed_with_threshold(
    threshold: float = 1.0,
    warn_message: str = "Операция заняла слишком много времени!"
):
    """Декоратор с предупреждением при превышении порога."""
    def decorator(func: Callable) -> Callable:
        if asyncio.iscoroutinefunction(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                start = time.perf_counter()
                result = await func(*args, **kwargs)
                elapsed = time.perf_counter() - start

                if elapsed > threshold:
                    logging.warning(
                        f"{func.__name__}: {warn_message} "
                        f"({elapsed:.4f} сек > {threshold} сек)"
                    )
                else:
                    logging.debug(f"{func.__name__}: {elapsed:.4f} сек")

                return result
            return wrapper
        else:
            @wraps(func)
            def wrapper(*args, **kwargs):
                start = time.perf_counter()
                result = func(*args, **kwargs)
                elapsed = time.perf_counter() - start

                if elapsed > threshold:
                    logging.warning(
                        f"{func.__name__}: {warn_message} "
                        f"({elapsed:.4f} сек > {threshold} сек)"
                    )
                else:
                    logging.debug(f"{func.__name__}: {elapsed:.4f} сек")

                return result
            return wrapper
    return decorator

@timed_with_threshold(threshold=0.5)
async def maybe_slow_operation():
    await asyncio.sleep(0.8)  # Превысит порог
    return "готово"

logging.basicConfig(level=logging.WARNING)
asyncio.run(maybe_slow_operation())
```

## Контекстный менеджер для замера времени

```python
import asyncio
import time
from contextlib import contextmanager, asynccontextmanager
from typing import Optional

class Timer:
    """Класс для измерения времени."""

    def __init__(self, name: str = "Операция"):
        self.name = name
        self.start: float = 0
        self.elapsed: float = 0

    def __enter__(self):
        self.start = time.perf_counter()
        return self

    def __exit__(self, *args):
        self.elapsed = time.perf_counter() - self.start
        print(f"{self.name}: {self.elapsed:.4f} сек")

    async def __aenter__(self):
        self.start = time.perf_counter()
        return self

    async def __aexit__(self, *args):
        self.elapsed = time.perf_counter() - self.start
        print(f"{self.name}: {self.elapsed:.4f} сек")

# Синхронное использование
with Timer("Синхронная операция"):
    time.sleep(0.5)

# Асинхронное использование
async def main():
    async with Timer("Асинхронная операция"):
        await asyncio.sleep(0.5)

asyncio.run(main())
```

## Сбор статистики времени выполнения

```python
import asyncio
import time
from functools import wraps
from typing import Callable, Dict, List
from dataclasses import dataclass, field
from statistics import mean, stdev

@dataclass
class TimingStats:
    """Статистика времени выполнения."""
    name: str
    times: List[float] = field(default_factory=list)

    @property
    def count(self) -> int:
        return len(self.times)

    @property
    def total(self) -> float:
        return sum(self.times)

    @property
    def average(self) -> float:
        return mean(self.times) if self.times else 0

    @property
    def std_dev(self) -> float:
        return stdev(self.times) if len(self.times) > 1 else 0

    @property
    def min_time(self) -> float:
        return min(self.times) if self.times else 0

    @property
    def max_time(self) -> float:
        return max(self.times) if self.times else 0

    def __str__(self) -> str:
        return (
            f"{self.name}: calls={self.count}, "
            f"total={self.total:.4f}s, "
            f"avg={self.average:.4f}s, "
            f"min={self.min_time:.4f}s, "
            f"max={self.max_time:.4f}s"
        )

class TimingCollector:
    """Коллектор статистики времени."""

    _stats: Dict[str, TimingStats] = {}

    @classmethod
    def record(cls, name: str, elapsed: float):
        if name not in cls._stats:
            cls._stats[name] = TimingStats(name)
        cls._stats[name].times.append(elapsed)

    @classmethod
    def get_stats(cls, name: str) -> Optional[TimingStats]:
        return cls._stats.get(name)

    @classmethod
    def all_stats(cls) -> Dict[str, TimingStats]:
        return cls._stats.copy()

    @classmethod
    def report(cls):
        print("\n=== Timing Report ===")
        for stats in cls._stats.values():
            print(stats)

    @classmethod
    def reset(cls):
        cls._stats.clear()

def collect_timing(name: Optional[str] = None):
    """Декоратор для сбора статистики времени."""
    def decorator(func: Callable) -> Callable:
        timing_name = name or func.__name__

        if asyncio.iscoroutinefunction(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                start = time.perf_counter()
                try:
                    return await func(*args, **kwargs)
                finally:
                    TimingCollector.record(
                        timing_name,
                        time.perf_counter() - start
                    )
            return wrapper
        else:
            @wraps(func)
            def wrapper(*args, **kwargs):
                start = time.perf_counter()
                try:
                    return func(*args, **kwargs)
                finally:
                    TimingCollector.record(
                        timing_name,
                        time.perf_counter() - start
                    )
            return wrapper
    return decorator

# Использование
@collect_timing("fetch_user")
async def fetch_user(user_id: int):
    await asyncio.sleep(0.1 + user_id * 0.05)
    return {"id": user_id}

async def main():
    # Выполняем несколько вызовов
    for i in range(5):
        await fetch_user(i)

    # Выводим отчёт
    TimingCollector.report()

asyncio.run(main())
```

## Профилирование группы операций

```python
import asyncio
import time
from contextlib import asynccontextmanager
from typing import Dict, List
from dataclasses import dataclass, field

@dataclass
class ProfileResult:
    """Результат профилирования."""
    name: str
    total_time: float
    operations: Dict[str, float] = field(default_factory=dict)

    def __str__(self) -> str:
        lines = [f"\n=== Profile: {self.name} ==="]
        lines.append(f"Total time: {self.total_time:.4f}s")
        lines.append("\nOperations:")
        for op, time_val in sorted(
            self.operations.items(),
            key=lambda x: x[1],
            reverse=True
        ):
            percentage = (time_val / self.total_time) * 100
            lines.append(f"  {op}: {time_val:.4f}s ({percentage:.1f}%)")
        return "\n".join(lines)

class AsyncProfiler:
    """Асинхронный профайлер."""

    def __init__(self, name: str):
        self.name = name
        self.operations: Dict[str, float] = {}
        self.start_time: float = 0
        self.total_time: float = 0

    async def __aenter__(self):
        self.start_time = time.perf_counter()
        return self

    async def __aexit__(self, *args):
        self.total_time = time.perf_counter() - self.start_time

    @asynccontextmanager
    async def measure(self, operation_name: str):
        """Измеряет время операции внутри профайлера."""
        start = time.perf_counter()
        try:
            yield
        finally:
            elapsed = time.perf_counter() - start
            self.operations[operation_name] = (
                self.operations.get(operation_name, 0) + elapsed
            )

    def result(self) -> ProfileResult:
        return ProfileResult(
            name=self.name,
            total_time=self.total_time,
            operations=self.operations.copy()
        )

# Использование
async def complex_operation():
    async with AsyncProfiler("Complex Operation") as profiler:
        async with profiler.measure("fetch_users"):
            await asyncio.sleep(0.3)

        async with profiler.measure("process_data"):
            await asyncio.sleep(0.2)

        async with profiler.measure("save_results"):
            await asyncio.sleep(0.1)

        # Несколько вызовов одной операции
        for _ in range(3):
            async with profiler.measure("validate"):
                await asyncio.sleep(0.05)

    print(profiler.result())

asyncio.run(complex_operation())
```

## Декоратор с кэшированием времени

```python
import asyncio
import time
from functools import wraps
from typing import Callable, Any, Dict, Tuple

class TimedCache:
    """Кэш с учётом времени выполнения."""

    def __init__(self, ttl: float = 60.0):
        self.ttl = ttl
        self.cache: Dict[Tuple, Tuple[Any, float]] = {}

    def get(self, key: Tuple) -> Tuple[Any, bool]:
        if key in self.cache:
            value, timestamp = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return value, True
            del self.cache[key]
        return None, False

    def set(self, key: Tuple, value: Any):
        self.cache[key] = (value, time.time())

def timed_cached(ttl: float = 60.0):
    """Декоратор с кэшированием и замером времени."""
    cache = TimedCache(ttl)

    def decorator(func: Callable) -> Callable:
        if asyncio.iscoroutinefunction(func):
            @wraps(func)
            async def wrapper(*args, **kwargs):
                key = (args, tuple(sorted(kwargs.items())))
                cached_value, hit = cache.get(key)

                if hit:
                    print(f"{func.__name__}: cache hit")
                    return cached_value

                start = time.perf_counter()
                result = await func(*args, **kwargs)
                elapsed = time.perf_counter() - start

                cache.set(key, result)
                print(f"{func.__name__}: cache miss, {elapsed:.4f}s")

                return result
            return wrapper
        else:
            @wraps(func)
            def wrapper(*args, **kwargs):
                key = (args, tuple(sorted(kwargs.items())))
                cached_value, hit = cache.get(key)

                if hit:
                    print(f"{func.__name__}: cache hit")
                    return cached_value

                start = time.perf_counter()
                result = func(*args, **kwargs)
                elapsed = time.perf_counter() - start

                cache.set(key, result)
                print(f"{func.__name__}: cache miss, {elapsed:.4f}s")

                return result
            return wrapper
    return decorator

@timed_cached(ttl=5.0)
async def expensive_operation(x: int) -> int:
    await asyncio.sleep(1)
    return x * 2

async def main():
    # Первый вызов - cache miss
    result1 = await expensive_operation(5)

    # Второй вызов - cache hit
    result2 = await expensive_operation(5)

    print(f"Результаты: {result1}, {result2}")

asyncio.run(main())
```

## Best Practices

1. **Используйте `time.perf_counter()`** вместо `time.time()` для точных измерений
2. **Оборачивайте в try/finally** для корректного замера даже при исключениях
3. **Сохраняйте метаданные** с помощью `@functools.wraps`
4. **Собирайте статистику** для анализа производительности
5. **Устанавливайте пороги** для автоматических предупреждений

```python
import asyncio
import time
from functools import wraps

def production_timer(threshold: float = 1.0):
    """Декоратор для production-окружения."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, **kwargs):
            start = time.perf_counter()
            try:
                return await func(*args, **kwargs)
            finally:
                elapsed = time.perf_counter() - start
                if elapsed > threshold:
                    # В production логируем только медленные операции
                    import logging
                    logging.warning(
                        f"Slow operation: {func.__name__} "
                        f"took {elapsed:.2f}s"
                    )
        return wrapper
    return decorator
```

---

[prev: ./05-futures.md](./05-futures.md) | [next: ./07-pitfalls.md](./07-pitfalls.md)
