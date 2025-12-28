# Моделирование с помощью sleep

[prev: ./01-coroutines.md](./01-coroutines.md) | [next: ./03-tasks.md](./03-tasks.md)

---

## Что такое asyncio.sleep?

**`asyncio.sleep()`** - это асинхронная функция, которая приостанавливает выполнение корутины на указанное количество секунд, не блокируя event loop. Это ключевое отличие от `time.sleep()`, который блокирует весь поток выполнения.

## Синтаксис

```python
import asyncio

await asyncio.sleep(delay, result=None)
```

- **delay** - время задержки в секундах (может быть float)
- **result** - значение, которое будет возвращено после задержки (опционально)

## Различие между asyncio.sleep и time.sleep

### time.sleep (блокирующий)

```python
import time

def blocking_example():
    print("Начало")
    time.sleep(2)  # Блокирует ВЕСЬ поток на 2 секунды
    print("Конец")

blocking_example()
```

### asyncio.sleep (неблокирующий)

```python
import asyncio

async def non_blocking_example():
    print("Начало")
    await asyncio.sleep(2)  # Освобождает event loop на 2 секунды
    print("Конец")

asyncio.run(non_blocking_example())
```

## Практическая демонстрация различий

```python
import asyncio
import time

# НЕПРАВИЛЬНО: использование time.sleep в асинхронном коде
async def bad_example():
    print("Задача 1: начало")
    time.sleep(2)  # Блокирует event loop!
    print("Задача 1: конец")

async def task2():
    print("Задача 2: начало")
    await asyncio.sleep(1)
    print("Задача 2: конец")

async def main_bad():
    await asyncio.gather(bad_example(), task2())

# Вывод будет ПОСЛЕДОВАТЕЛЬНЫМ из-за time.sleep:
# Задача 1: начало
# (пауза 2 секунды)
# Задача 1: конец
# Задача 2: начало
# (пауза 1 секунда)
# Задача 2: конец

# ПРАВИЛЬНО: использование asyncio.sleep
async def good_example():
    print("Задача 1: начало")
    await asyncio.sleep(2)  # Не блокирует event loop
    print("Задача 1: конец")

async def main_good():
    await asyncio.gather(good_example(), task2())

# Вывод будет КОНКУРЕНТНЫМ:
# Задача 1: начало
# Задача 2: начало
# (через 1 секунду)
# Задача 2: конец
# (ещё через 1 секунду)
# Задача 1: конец
```

## Использование для моделирования задержек

`asyncio.sleep()` идеально подходит для имитации операций ввода-вывода при разработке и тестировании:

```python
import asyncio

async def fetch_from_database(query: str) -> dict:
    """Имитация запроса к базе данных."""
    print(f"Выполняю запрос: {query}")
    await asyncio.sleep(0.5)  # Имитация задержки БД
    return {"result": f"Данные для {query}"}

async def call_external_api(endpoint: str) -> dict:
    """Имитация вызова внешнего API."""
    print(f"Вызываю API: {endpoint}")
    await asyncio.sleep(1.0)  # Имитация сетевой задержки
    return {"status": "ok", "endpoint": endpoint}

async def read_file(filename: str) -> str:
    """Имитация чтения файла."""
    print(f"Читаю файл: {filename}")
    await asyncio.sleep(0.2)  # Имитация дискового I/O
    return f"Содержимое {filename}"

async def main():
    # Запускаем все операции конкурентно
    results = await asyncio.gather(
        fetch_from_database("SELECT * FROM users"),
        call_external_api("/api/data"),
        read_file("config.json")
    )

    for result in results:
        print(f"Результат: {result}")

asyncio.run(main())
```

## Возврат значения из asyncio.sleep

Параметр `result` позволяет вернуть значение после задержки:

```python
import asyncio

async def delayed_value():
    # Вернёт "готово" после 2 секунд
    result = await asyncio.sleep(2, result="готово")
    return result

async def main():
    value = await delayed_value()
    print(value)  # "готово"

asyncio.run(main())
```

## Нулевая задержка - asyncio.sleep(0)

`asyncio.sleep(0)` - особый случай, который передаёт управление event loop без реальной задержки:

```python
import asyncio

async def cpu_intensive_task():
    """Длительная CPU-задача с точками передачи управления."""
    result = 0
    for i in range(1000000):
        result += i
        if i % 100000 == 0:
            # Даём event loop шанс обработать другие задачи
            await asyncio.sleep(0)
            print(f"Обработано: {i}")
    return result

async def background_task():
    """Фоновая задача."""
    for i in range(5):
        print(f"Фоновая задача: итерация {i}")
        await asyncio.sleep(0.1)

async def main():
    await asyncio.gather(
        cpu_intensive_task(),
        background_task()
    )

asyncio.run(main())
```

## Паттерны использования

### Паттерн 1: Периодическое выполнение

```python
import asyncio
from datetime import datetime

async def periodic_task(interval: float, name: str):
    """Задача, выполняющаяся периодически."""
    while True:
        print(f"[{datetime.now().strftime('%H:%M:%S')}] {name}: выполняю работу")
        await asyncio.sleep(interval)

async def main():
    # Создаём несколько периодических задач
    task1 = asyncio.create_task(periodic_task(1.0, "Задача A"))
    task2 = asyncio.create_task(periodic_task(2.0, "Задача B"))

    # Даём им поработать 5 секунд
    await asyncio.sleep(5)

    # Отменяем задачи
    task1.cancel()
    task2.cancel()

asyncio.run(main())
```

### Паттерн 2: Экспоненциальная задержка (backoff)

```python
import asyncio
import random

async def fetch_with_retry(url: str, max_retries: int = 5) -> str:
    """Запрос с экспоненциальной задержкой при ошибках."""
    base_delay = 1.0

    for attempt in range(max_retries):
        try:
            print(f"Попытка {attempt + 1}: запрос к {url}")
            # Имитация запроса, который иногда падает
            await asyncio.sleep(0.5)
            if random.random() < 0.7:  # 70% шанс ошибки
                raise ConnectionError("Сервер недоступен")
            return "Данные получены успешно"
        except ConnectionError as e:
            if attempt == max_retries - 1:
                raise
            delay = base_delay * (2 ** attempt) + random.uniform(0, 1)
            print(f"Ошибка: {e}. Повтор через {delay:.2f} сек...")
            await asyncio.sleep(delay)

async def main():
    try:
        result = await fetch_with_retry("https://api.example.com")
        print(result)
    except ConnectionError:
        print("Не удалось получить данные после всех попыток")

asyncio.run(main())
```

### Паттерн 3: Throttling (ограничение частоты)

```python
import asyncio
from collections import deque
from time import time

class RateLimiter:
    """Ограничитель частоты запросов."""

    def __init__(self, max_requests: int, time_window: float):
        self.max_requests = max_requests
        self.time_window = time_window
        self.requests: deque = deque()

    async def acquire(self):
        """Ожидание разрешения на выполнение запроса."""
        now = time()

        # Удаляем устаревшие записи
        while self.requests and self.requests[0] < now - self.time_window:
            self.requests.popleft()

        if len(self.requests) >= self.max_requests:
            sleep_time = self.requests[0] - (now - self.time_window)
            if sleep_time > 0:
                print(f"Rate limit: ожидание {sleep_time:.2f} сек")
                await asyncio.sleep(sleep_time)

        self.requests.append(time())

async def make_request(limiter: RateLimiter, request_id: int):
    await limiter.acquire()
    print(f"Запрос {request_id} выполняется")
    await asyncio.sleep(0.1)  # Имитация запроса

async def main():
    # Максимум 3 запроса в секунду
    limiter = RateLimiter(max_requests=3, time_window=1.0)

    tasks = [make_request(limiter, i) for i in range(10)]
    await asyncio.gather(*tasks)

asyncio.run(main())
```

## Распространённые ошибки

### Ошибка 1: Использование time.sleep в async-коде

```python
import asyncio
import time

async def bad():
    # НЕПРАВИЛЬНО - блокирует event loop
    time.sleep(1)

async def good():
    # ПРАВИЛЬНО - не блокирует event loop
    await asyncio.sleep(1)
```

### Ошибка 2: Забыли await перед asyncio.sleep

```python
import asyncio

async def example():
    # НЕПРАВИЛЬНО - задержки не будет
    asyncio.sleep(1)  # Создаёт корутину, но не выполняет

    # ПРАВИЛЬНО
    await asyncio.sleep(1)  # Ждёт 1 секунду
```

### Ошибка 3: Отрицательная задержка

```python
import asyncio

async def example(delay: float):
    # Может вызвать ValueError при отрицательном значении
    if delay < 0:
        delay = 0
    await asyncio.sleep(delay)
```

## Best Practices

1. **Всегда используйте `asyncio.sleep()`** вместо `time.sleep()` в асинхронном коде
2. **Используйте `asyncio.sleep(0)`** для передачи управления event loop в CPU-интенсивных задачах
3. **Применяйте разумные задержки** при моделировании - слишком большие замедлят тесты
4. **Используйте экспоненциальную задержку** при повторных попытках
5. **Не забывайте про `await`** перед `asyncio.sleep()`

```python
import asyncio

async def well_designed_function():
    """Пример хорошо спроектированной асинхронной функции."""
    # Используем asyncio.sleep для имитации I/O
    await asyncio.sleep(0.1)

    # Даём event loop шанс при длительных вычислениях
    for i in range(10):
        # ... вычисления ...
        if i % 5 == 0:
            await asyncio.sleep(0)

    return "Готово"

asyncio.run(well_designed_function())
```

---

[prev: ./01-coroutines.md](./01-coroutines.md) | [next: ./03-tasks.md](./03-tasks.md)
