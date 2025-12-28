# Снятие задач и тайм-ауты

[prev: ./03-tasks.md](./03-tasks.md) | [next: ./05-futures.md](./05-futures.md)

---

## Введение

В асинхронном программировании критически важно уметь:
1. **Отменять задачи** - когда результат больше не нужен или операция занимает слишком много времени
2. **Устанавливать тайм-ауты** - чтобы ограничить время выполнения операций

asyncio предоставляет мощные инструменты для управления временем жизни задач.

## Отмена задач (Cancellation)

### Базовая отмена с task.cancel()

```python
import asyncio

async def long_running_task():
    print("Задача: начинаю длительную работу...")
    try:
        await asyncio.sleep(10)
        print("Задача: работа завершена")
        return "результат"
    except asyncio.CancelledError:
        print("Задача: была отменена!")
        raise  # Важно: перебрасываем исключение

async def main():
    task = asyncio.create_task(long_running_task())

    # Даём задаче начать выполнение
    await asyncio.sleep(1)

    # Отменяем задачу
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Main: задача успешно отменена")

    print(f"Задача отменена: {task.cancelled()}")

asyncio.run(main())
```

### Отмена с сообщением (Python 3.9+)

```python
import asyncio

async def worker():
    try:
        await asyncio.sleep(10)
    except asyncio.CancelledError as e:
        print(f"Причина отмены: {e.args}")
        raise

async def main():
    task = asyncio.create_task(worker())
    await asyncio.sleep(0.1)

    # Отмена с сообщением
    task.cancel("Превышено время ожидания")

    try:
        await task
    except asyncio.CancelledError:
        pass

asyncio.run(main())
```

### Обработка отмены с очисткой ресурсов

```python
import asyncio

async def task_with_cleanup():
    resource = "открытый_ресурс"
    print(f"Задача: открыл {resource}")

    try:
        await asyncio.sleep(10)
        return "результат"
    except asyncio.CancelledError:
        print("Задача: отменена, выполняю очистку...")
        # Важно: cleanup НЕ должен быть отменяем
        await cleanup_resource(resource)
        raise
    finally:
        print("Задача: блок finally выполнен")

async def cleanup_resource(resource):
    """Очистка ресурса - не может быть отменена."""
    # asyncio.shield защищает от отмены
    print(f"Очистка: освобождаю {resource}")
    await asyncio.sleep(0.1)  # Имитация асинхронной очистки
    print("Очистка: завершена")

async def main():
    task = asyncio.create_task(task_with_cleanup())
    await asyncio.sleep(0.5)
    task.cancel()

    try:
        await task
    except asyncio.CancelledError:
        print("Main: задача отменена")

asyncio.run(main())
```

### Защита от отмены с asyncio.shield()

```python
import asyncio

async def critical_operation():
    """Критическая операция, которую нельзя прерывать."""
    print("Критическая операция: начало")
    await asyncio.sleep(2)
    print("Критическая операция: завершена")
    return "важные данные"

async def main():
    task = asyncio.create_task(
        asyncio.shield(critical_operation())
    )

    await asyncio.sleep(0.5)
    task.cancel()  # Попытка отмены

    try:
        result = await task
        print(f"Результат: {result}")
    except asyncio.CancelledError:
        print("Внешняя задача отменена, но внутренняя продолжается")
        # Внутренняя операция всё ещё выполняется!
        await asyncio.sleep(2)

asyncio.run(main())
```

## Тайм-ауты

### asyncio.wait_for() - базовый тайм-аут

```python
import asyncio

async def slow_operation():
    print("Медленная операция: начало")
    await asyncio.sleep(5)
    print("Медленная операция: конец")
    return "результат"

async def main():
    try:
        # Ограничиваем время выполнения 2 секундами
        result = await asyncio.wait_for(slow_operation(), timeout=2.0)
        print(f"Результат: {result}")
    except asyncio.TimeoutError:
        print("Операция превысила тайм-аут!")

asyncio.run(main())
```

### asyncio.timeout() - контекстный менеджер (Python 3.11+)

```python
import asyncio

async def operation1():
    await asyncio.sleep(1)
    return "op1 готово"

async def operation2():
    await asyncio.sleep(0.5)
    return "op2 готово"

async def main():
    try:
        async with asyncio.timeout(2.0):
            # Все операции внутри блока ограничены 2 секундами
            result1 = await operation1()
            result2 = await operation2()
            print(f"Результаты: {result1}, {result2}")
    except TimeoutError:
        print("Операции не успели завершиться")

asyncio.run(main())
```

### asyncio.timeout_at() - тайм-аут до определённого времени

```python
import asyncio

async def main():
    loop = asyncio.get_event_loop()

    # Тайм-аут через 3 секунды от текущего момента
    deadline = loop.time() + 3.0

    try:
        async with asyncio.timeout_at(deadline):
            await asyncio.sleep(1)
            print("Первая операция завершена")
            await asyncio.sleep(1)
            print("Вторая операция завершена")
            await asyncio.sleep(2)  # Это превысит deadline
            print("Третья операция завершена")
    except TimeoutError:
        print("Превышен deadline")

asyncio.run(main())
```

### Перемещение deadline (Python 3.11+)

```python
import asyncio

async def main():
    try:
        async with asyncio.timeout(5.0) as cm:
            print(f"Начальный deadline: {cm.when()}")

            await asyncio.sleep(1)

            # Продлеваем тайм-аут на 3 секунды от текущего момента
            cm.reschedule(asyncio.get_event_loop().time() + 3.0)
            print(f"Новый deadline: {cm.when()}")

            await asyncio.sleep(2)
            print("Операция завершена")
    except TimeoutError:
        print("Тайм-аут!")

asyncio.run(main())
```

## Комбинирование тайм-аутов и отмены

### Тайм-аут для группы задач

```python
import asyncio

async def fetch_data(source: str, delay: float) -> str:
    print(f"Загрузка из {source}...")
    await asyncio.sleep(delay)
    return f"Данные из {source}"

async def main():
    tasks = [
        fetch_data("API", 1),
        fetch_data("Database", 2),
        fetch_data("Cache", 5),  # Эта задача не успеет
    ]

    try:
        # Ограничиваем общее время 3 секундами
        results = await asyncio.wait_for(
            asyncio.gather(*tasks),
            timeout=3.0
        )
        print(f"Все результаты: {results}")
    except asyncio.TimeoutError:
        print("Не все задачи успели завершиться")

asyncio.run(main())
```

### Индивидуальные тайм-ауты для задач

```python
import asyncio

async def fetch_with_timeout(source: str, delay: float, timeout: float):
    try:
        result = await asyncio.wait_for(
            fetch_data_impl(source, delay),
            timeout=timeout
        )
        return {"source": source, "data": result, "status": "success"}
    except asyncio.TimeoutError:
        return {"source": source, "data": None, "status": "timeout"}

async def fetch_data_impl(source: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"Данные из {source}"

async def main():
    results = await asyncio.gather(
        fetch_with_timeout("API", 1, 2),      # Успеет
        fetch_with_timeout("Database", 3, 2),  # Тайм-аут
        fetch_with_timeout("Cache", 0.5, 2),   # Успеет
    )

    for result in results:
        print(f"{result['source']}: {result['status']}")

asyncio.run(main())
```

## asyncio.wait() - гибкое ожидание

```python
import asyncio

async def task(name: str, delay: float):
    await asyncio.sleep(delay)
    return f"{name} завершена"

async def main():
    tasks = {
        asyncio.create_task(task("A", 1), name="task_A"),
        asyncio.create_task(task("B", 2), name="task_B"),
        asyncio.create_task(task("C", 3), name="task_C"),
    }

    # Ждём первую завершённую задачу
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )

    print("Первая завершённая:")
    for t in done:
        print(f"  {t.get_name()}: {t.result()}")

    print(f"Ещё выполняются: {len(pending)}")

    # Отменяем оставшиеся
    for t in pending:
        t.cancel()

    # Ждём отмены
    await asyncio.gather(*pending, return_exceptions=True)

asyncio.run(main())
```

### Режимы wait()

```python
import asyncio

async def task_success(name: str, delay: float):
    await asyncio.sleep(delay)
    return f"{name} успех"

async def task_fail(name: str, delay: float):
    await asyncio.sleep(delay)
    raise ValueError(f"{name} ошибка")

async def main():
    tasks = [
        asyncio.create_task(task_success("A", 1)),
        asyncio.create_task(task_fail("B", 0.5)),
        asyncio.create_task(task_success("C", 2)),
    ]

    # FIRST_COMPLETED - до первой завершённой (успех или ошибка)
    done, pending = await asyncio.wait(
        tasks,
        return_when=asyncio.FIRST_COMPLETED
    )
    print(f"FIRST_COMPLETED: {len(done)} done, {len(pending)} pending")

    # FIRST_EXCEPTION - до первого исключения или всех завершённых
    # ALL_COMPLETED - до завершения всех (по умолчанию)

    # С тайм-аутом
    tasks2 = [
        asyncio.create_task(task_success("X", 5)),
        asyncio.create_task(task_success("Y", 1)),
    ]

    done, pending = await asyncio.wait(
        tasks2,
        timeout=2.0  # Ждём максимум 2 секунды
    )
    print(f"С тайм-аутом: {len(done)} done, {len(pending)} pending")

    # Очистка
    for t in pending:
        t.cancel()
    await asyncio.gather(*pending, return_exceptions=True)

asyncio.run(main())
```

## Паттерны обработки тайм-аутов

### Паттерн: Retry с тайм-аутом

```python
import asyncio
import random

async def unreliable_operation():
    """Ненадёжная операция, которая может зависнуть."""
    delay = random.uniform(0.5, 3)
    await asyncio.sleep(delay)
    if random.random() < 0.3:
        raise ConnectionError("Ошибка соединения")
    return "успех"

async def operation_with_retry(max_retries: int = 3, timeout: float = 1.5):
    for attempt in range(max_retries):
        try:
            result = await asyncio.wait_for(
                unreliable_operation(),
                timeout=timeout
            )
            return result
        except asyncio.TimeoutError:
            print(f"Попытка {attempt + 1}: тайм-аут")
        except ConnectionError as e:
            print(f"Попытка {attempt + 1}: {e}")

        if attempt < max_retries - 1:
            await asyncio.sleep(0.5)  # Пауза перед повтором

    raise RuntimeError("Все попытки исчерпаны")

async def main():
    try:
        result = await operation_with_retry()
        print(f"Результат: {result}")
    except RuntimeError as e:
        print(f"Ошибка: {e}")

asyncio.run(main())
```

### Паттерн: Graceful shutdown

```python
import asyncio
import signal

class GracefulShutdown:
    def __init__(self):
        self.shutdown_event = asyncio.Event()

    def request_shutdown(self):
        self.shutdown_event.set()

    async def wait_for_shutdown(self):
        await self.shutdown_event.wait()

async def worker(name: str, shutdown: GracefulShutdown):
    while not shutdown.shutdown_event.is_set():
        print(f"Worker {name}: работаю")
        try:
            await asyncio.wait_for(
                shutdown.wait_for_shutdown(),
                timeout=1.0
            )
        except asyncio.TimeoutError:
            continue
    print(f"Worker {name}: получил сигнал завершения")

async def main():
    shutdown = GracefulShutdown()

    workers = [
        asyncio.create_task(worker(f"W{i}", shutdown))
        for i in range(3)
    ]

    # Имитация работы
    await asyncio.sleep(3)

    # Запрос на завершение
    print("Запрос на graceful shutdown...")
    shutdown.request_shutdown()

    # Ждём завершения воркеров с тайм-аутом
    try:
        await asyncio.wait_for(
            asyncio.gather(*workers),
            timeout=5.0
        )
        print("Все воркеры завершились корректно")
    except asyncio.TimeoutError:
        print("Тайм-аут ожидания, принудительная отмена")
        for w in workers:
            w.cancel()
        await asyncio.gather(*workers, return_exceptions=True)

asyncio.run(main())
```

### Паттерн: Circuit Breaker

```python
import asyncio
from enum import Enum
from dataclasses import dataclass
from time import time

class CircuitState(Enum):
    CLOSED = "closed"      # Нормальная работа
    OPEN = "open"          # Отказ, запросы не проходят
    HALF_OPEN = "half_open"  # Пробный режим

@dataclass
class CircuitBreaker:
    failure_threshold: int = 3
    recovery_timeout: float = 5.0
    timeout: float = 2.0

    state: CircuitState = CircuitState.CLOSED
    failures: int = 0
    last_failure_time: float = 0

    async def call(self, coro):
        if self.state == CircuitState.OPEN:
            if time() - self.last_failure_time > self.recovery_timeout:
                self.state = CircuitState.HALF_OPEN
            else:
                raise RuntimeError("Circuit is OPEN")

        try:
            result = await asyncio.wait_for(coro, timeout=self.timeout)
            self._on_success()
            return result
        except (asyncio.TimeoutError, Exception) as e:
            self._on_failure()
            raise

    def _on_success(self):
        self.failures = 0
        self.state = CircuitState.CLOSED

    def _on_failure(self):
        self.failures += 1
        self.last_failure_time = time()
        if self.failures >= self.failure_threshold:
            self.state = CircuitState.OPEN

async def external_service():
    await asyncio.sleep(0.5)
    return "данные"

async def main():
    cb = CircuitBreaker(failure_threshold=2, timeout=0.3)

    for i in range(5):
        try:
            result = await cb.call(external_service())
            print(f"Запрос {i}: {result}")
        except asyncio.TimeoutError:
            print(f"Запрос {i}: тайм-аут")
        except RuntimeError as e:
            print(f"Запрос {i}: {e}")

asyncio.run(main())
```

## Best Practices

1. **Всегда обрабатывайте CancelledError** и освобождайте ресурсы
2. **Используйте asyncio.shield()** для критических операций
3. **Устанавливайте разумные тайм-ауты** для всех внешних операций
4. **Предпочитайте asyncio.timeout()** (Python 3.11+) вместо wait_for() для блоков кода
5. **Логируйте отмены и тайм-ауты** для отладки

```python
import asyncio
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

async def robust_operation(timeout: float = 5.0):
    """Пример надёжной операции с правильной обработкой."""
    try:
        async with asyncio.timeout(timeout):
            result = await do_work()
            return result
    except TimeoutError:
        logger.warning("Операция превысила тайм-аут")
        raise
    except asyncio.CancelledError:
        logger.info("Операция отменена, выполняю очистку")
        await cleanup()
        raise

async def do_work():
    await asyncio.sleep(1)
    return "готово"

async def cleanup():
    await asyncio.sleep(0.1)

asyncio.run(robust_operation())
```

---

[prev: ./03-tasks.md](./03-tasks.md) | [next: ./05-futures.md](./05-futures.md)
