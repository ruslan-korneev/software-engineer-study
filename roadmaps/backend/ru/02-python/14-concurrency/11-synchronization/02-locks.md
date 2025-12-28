# Блокировки

[prev: ./01-race-conditions.md](./01-race-conditions.md) | [next: ./03-semaphores.md](./03-semaphores.md)

---

## Что такое Lock в asyncio?

**Lock** (блокировка) - это примитив синхронизации, который обеспечивает взаимоисключающий доступ к общему ресурсу. В любой момент времени только одна корутина может владеть блокировкой. Остальные корутины, пытающиеся захватить занятую блокировку, будут ожидать её освобождения.

### Основные свойства

- **Взаимоисключение**: только одна корутина может владеть блокировкой
- **Не реентерабельность**: одна корутина не может захватить блокировку дважды
- **FIFO-порядок**: корутины получают блокировку в порядке запроса (в большинстве случаев)
- **Кооперативность**: блокировка работает на уровне event loop, не блокирует поток

## Базовое использование

### Создание и захват блокировки

```python
import asyncio

lock = asyncio.Lock()

async def critical_section():
    # Захват блокировки
    await lock.acquire()
    try:
        # Критическая секция - только одна корутина здесь
        print("Выполняю защищённую операцию")
        await asyncio.sleep(1)
    finally:
        # Обязательно освобождаем блокировку!
        lock.release()


async def main():
    await asyncio.gather(
        critical_section(),
        critical_section(),
        critical_section()
    )


asyncio.run(main())
```

### Использование контекстного менеджера (рекомендуется)

```python
import asyncio

lock = asyncio.Lock()

async def safe_operation():
    async with lock:  # Автоматический acquire и release
        print("Защищённая операция")
        await asyncio.sleep(1)
        # Блокировка освобождается автоматически при выходе


async def main():
    await asyncio.gather(
        safe_operation(),
        safe_operation(),
        safe_operation()
    )


asyncio.run(main())
```

## Решение проблемы состояния гонки

### До: код с race condition

```python
import asyncio

counter = 0

async def unsafe_increment():
    global counter
    temp = counter
    await asyncio.sleep(0.001)
    counter = temp + 1


async def main():
    global counter
    counter = 0
    await asyncio.gather(*[unsafe_increment() for _ in range(100)])
    print(f"Результат: {counter}")  # Обычно 1, а не 100


asyncio.run(main())
```

### После: защищённый код с Lock

```python
import asyncio

counter = 0
lock = asyncio.Lock()

async def safe_increment():
    global counter
    async with lock:
        temp = counter
        await asyncio.sleep(0.001)
        counter = temp + 1


async def main():
    global counter
    counter = 0
    await asyncio.gather(*[safe_increment() for _ in range(100)])
    print(f"Результат: {counter}")  # Всегда 100


asyncio.run(main())
```

## Практические примеры

### 1. Защита общего ресурса

```python
import asyncio
from typing import Dict, Any

class ThreadSafeCache:
    def __init__(self):
        self._cache: Dict[str, Any] = {}
        self._lock = asyncio.Lock()

    async def get(self, key: str) -> Any:
        async with self._lock:
            return self._cache.get(key)

    async def set(self, key: str, value: Any) -> None:
        async with self._lock:
            self._cache[key] = value

    async def get_or_set(self, key: str, factory) -> Any:
        """Атомарное получение или создание значения"""
        async with self._lock:
            if key not in self._cache:
                # factory может быть корутиной
                if asyncio.iscoroutinefunction(factory):
                    self._cache[key] = await factory()
                else:
                    self._cache[key] = factory()
            return self._cache[key]


async def demo():
    cache = ThreadSafeCache()

    async def expensive_computation():
        await asyncio.sleep(1)
        return "computed_value"

    # Только одна корутина выполнит expensive_computation
    results = await asyncio.gather(
        cache.get_or_set("key", expensive_computation),
        cache.get_or_set("key", expensive_computation),
        cache.get_or_set("key", expensive_computation)
    )

    print(f"Результаты: {results}")  # Все получат одно значение


asyncio.run(demo())
```

### 2. Singleton с асинхронной инициализацией

```python
import asyncio

class AsyncSingleton:
    _instance = None
    _lock = asyncio.Lock()

    @classmethod
    async def get_instance(cls):
        # Double-checked locking pattern
        if cls._instance is None:
            async with cls._lock:
                # Повторная проверка внутри блокировки
                if cls._instance is None:
                    instance = cls()
                    await instance._initialize()
                    cls._instance = instance
        return cls._instance

    async def _initialize(self):
        print("Инициализация singleton...")
        await asyncio.sleep(1)  # Симуляция асинхронной инициализации
        print("Singleton готов!")


async def worker(name: str):
    instance = await AsyncSingleton.get_instance()
    print(f"{name} получил экземпляр: {id(instance)}")


async def main():
    await asyncio.gather(
        worker("Worker-1"),
        worker("Worker-2"),
        worker("Worker-3")
    )


asyncio.run(main())
# Вывод:
# Инициализация singleton...
# Singleton готов!
# Worker-1 получил экземпляр: 140234567890
# Worker-2 получил экземпляр: 140234567890  # Тот же объект!
# Worker-3 получил экземпляр: 140234567890
```

### 3. Защита файловых операций

```python
import asyncio
import aiofiles

class AsyncFileWriter:
    def __init__(self, filename: str):
        self.filename = filename
        self._lock = asyncio.Lock()

    async def append(self, data: str) -> None:
        async with self._lock:
            async with aiofiles.open(self.filename, 'a') as f:
                await f.write(data + '\n')

    async def read_all(self) -> str:
        async with self._lock:
            async with aiofiles.open(self.filename, 'r') as f:
                return await f.read()


async def demo():
    writer = AsyncFileWriter('log.txt')

    async def log_message(msg: str):
        await writer.append(f"[{asyncio.get_event_loop().time():.3f}] {msg}")

    # Параллельная запись без race conditions
    await asyncio.gather(*[
        log_message(f"Message {i}")
        for i in range(10)
    ])

    content = await writer.read_all()
    print(content)


asyncio.run(demo())
```

## Методы Lock

### locked() - проверка состояния

```python
import asyncio

lock = asyncio.Lock()

async def demo():
    print(f"Заблокирован: {lock.locked()}")  # False

    async with lock:
        print(f"Заблокирован: {lock.locked()}")  # True

    print(f"Заблокирован: {lock.locked()}")  # False


asyncio.run(demo())
```

### acquire() и release() - ручное управление

```python
import asyncio

lock = asyncio.Lock()

async def manual_control():
    # Захват блокировки с await
    await lock.acquire()
    print("Блокировка захвачена")

    # ... критическая секция ...

    # Освобождение (НЕ корутина!)
    lock.release()
    print("Блокировка освобождена")


asyncio.run(manual_control())
```

### Неблокирующий захват

```python
import asyncio

lock = asyncio.Lock()

async def try_acquire():
    # Попытка захватить без ожидания
    if lock.locked():
        print("Блокировка занята, пропускаем")
        return False

    async with lock:
        print("Успешно захвачено!")
        await asyncio.sleep(1)
        return True


async def main():
    async with lock:
        # Блокировка занята
        result = await try_acquire()
        print(f"Результат: {result}")


asyncio.run(main())
```

## Распространённые ошибки

### 1. Deadlock при повторном захвате

```python
import asyncio

lock = asyncio.Lock()

async def deadlock_example():
    async with lock:
        print("Первый захват")
        async with lock:  # DEADLOCK! Вечное ожидание
            print("Второй захват - никогда не выполнится")


# asyncio.run(deadlock_example())  # Зависнет навсегда
```

### 2. Забытый release()

```python
import asyncio

lock = asyncio.Lock()

async def forgot_release():
    await lock.acquire()
    # Если здесь произойдёт исключение...
    raise ValueError("Ошибка!")
    lock.release()  # Никогда не выполнится!


async def main():
    try:
        await forgot_release()
    except ValueError:
        pass

    # Блокировка навсегда заблокирована!
    print(f"Заблокирован: {lock.locked()}")  # True


# Решение: используйте async with или try/finally
async def correct_approach():
    await lock.acquire()
    try:
        raise ValueError("Ошибка!")
    finally:
        lock.release()  # Всегда выполнится
```

### 3. Слишком большие критические секции

```python
import asyncio

lock = asyncio.Lock()

# ПЛОХО: блокировка удерживается слишком долго
async def bad_practice():
    async with lock:
        data = await fetch_from_api()  # Долгая операция под блокировкой
        processed = process(data)
        await save_to_database(processed)


# ХОРОШО: минимизируем критическую секцию
async def good_practice():
    data = await fetch_from_api()  # Вне блокировки
    processed = process(data)  # Вне блокировки

    async with lock:
        await save_to_database(processed)  # Только запись под блокировкой
```

### 4. Создание блокировки в неправильном месте

```python
import asyncio

# ПЛОХО: новая блокировка при каждом вызове
async def bad_function():
    lock = asyncio.Lock()  # Каждый вызов создаёт новую блокировку!
    async with lock:
        await critical_operation()


# ХОРОШО: одна блокировка на уровне модуля или класса
_lock = asyncio.Lock()

async def good_function():
    async with _lock:
        await critical_operation()
```

## Продвинутые паттерны

### Блокировка с таймаутом

```python
import asyncio

lock = asyncio.Lock()

async def acquire_with_timeout(timeout: float) -> bool:
    try:
        await asyncio.wait_for(lock.acquire(), timeout=timeout)
        return True
    except asyncio.TimeoutError:
        return False


async def demo():
    async with lock:
        # Блокировка занята на 5 секунд
        success = await acquire_with_timeout(1.0)
        print(f"Захват успешен: {success}")  # False


asyncio.run(demo())
```

### Блокировка по ключу (Key-based locking)

```python
import asyncio
from collections import defaultdict

class KeyedLock:
    def __init__(self):
        self._locks: dict[str, asyncio.Lock] = defaultdict(asyncio.Lock)
        self._meta_lock = asyncio.Lock()

    async def acquire(self, key: str):
        async with self._meta_lock:
            lock = self._locks[key]
        await lock.acquire()

    def release(self, key: str):
        self._locks[key].release()

    def __call__(self, key: str):
        return KeyedLockContext(self, key)


class KeyedLockContext:
    def __init__(self, keyed_lock: KeyedLock, key: str):
        self._keyed_lock = keyed_lock
        self._key = key

    async def __aenter__(self):
        await self._keyed_lock.acquire(self._key)
        return self

    async def __aexit__(self, *args):
        self._keyed_lock.release(self._key)


async def demo():
    keyed_lock = KeyedLock()

    async def process_user(user_id: str):
        async with keyed_lock(user_id):
            print(f"Обработка пользователя {user_id}")
            await asyncio.sleep(1)
            print(f"Завершено для {user_id}")

    # user_1 и user_2 обрабатываются параллельно
    # Два вызова для user_1 обрабатываются последовательно
    await asyncio.gather(
        process_user("user_1"),
        process_user("user_2"),
        process_user("user_1")
    )


asyncio.run(demo())
```

## Best Practices

1. **Всегда используйте `async with`** вместо ручного acquire/release
2. **Минимизируйте критические секции** - держите блокировку как можно меньше
3. **Избегайте вложенных блокировок** - это путь к deadlock
4. **Создавайте блокировку один раз** - на уровне класса или модуля
5. **Документируйте** какие ресурсы защищает блокировка
6. **Не используйте блокировку для CPU-bound операций** - используйте ProcessPoolExecutor

## Когда использовать Lock

| Ситуация | Lock подходит? |
|----------|----------------|
| Защита общего состояния | Да |
| Singleton инициализация | Да |
| Атомарные read-modify-write | Да |
| Ограничение параллелизма | Нет (используйте Semaphore) |
| Ожидание события | Нет (используйте Event) |
| Координация producer-consumer | Нет (используйте Condition/Queue) |

## Резюме

- `asyncio.Lock` обеспечивает взаимоисключающий доступ к ресурсам
- Всегда используйте `async with` для автоматического освобождения
- Минимизируйте время удержания блокировки
- Избегайте вложенных захватов - это приводит к deadlock
- Lock не реентерабелен (в отличие от `threading.RLock`)
- Для более сложных сценариев используйте Semaphore, Event, Condition

---

[prev: ./01-race-conditions.md](./01-race-conditions.md) | [next: ./03-semaphores.md](./03-semaphores.md)
