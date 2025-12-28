# Однопоточная конкурентность

[prev: ./05-gil.md](./05-gil.md) | [next: ./07-event-loop-intro.md](./07-event-loop-intro.md)

---

## Введение

Однопоточная конкурентность - это парадигма, при которой множество задач выполняются "одновременно" в одном потоке. Это основа работы asyncio и многих современных асинхронных фреймворков.

## Как это возможно?

Секрет однопоточной конкурентности в **кооперативной многозадачности**. Задачи добровольно уступают контроль, когда им нечего делать (например, при ожидании I/O).

```
Традиционная многопоточность (вытесняющая):
┌────────────────────────────────────────────────────────┐
│  ОС принудительно переключает потоки                  │
│                                                        │
│  Поток 1: ████──прервано──████──прервано──████        │
│  Поток 2: ░░░░████──прервано──████──прервано──        │
│                                                        │
│  ОС решает, когда переключать (каждые ~10ms)          │
└────────────────────────────────────────────────────────┘

Однопоточная конкурентность (кооперативная):
┌────────────────────────────────────────────────────────┐
│  Задачи добровольно уступают контроль                 │
│                                                        │
│  Задача 1: ████──await──████──await──████             │
│  Задача 2: ░░░░████──await──████──await──             │
│                                                        │
│  Программист решает, когда переключать (await)        │
└────────────────────────────────────────────────────────┘
```

## Аналогия: Повар на кухне

### Синхронный повар (однозадачный):

```python
def synchronous_chef():
    """Один повар, одно блюдо за раз"""
    # Готовит суп: стоит и смотрит, как варится
    boil_water()           # 5 минут ожидания
    add_ingredients()      # 1 минута
    simmer()               # 10 минут ожидания

    # Только потом начинает салат
    chop_vegetables()      # 3 минуты
    mix_salad()            # 1 минута

    # Общее время: 20 минут
```

### Асинхронный повар (конкурентный):

```python
async def async_chef():
    """Один повар, несколько блюд одновременно"""

    async def make_soup():
        await boil_water()      # Ставит воду и уходит
        add_ingredients()
        await simmer()          # Ставит на медленный огонь и уходит

    async def make_salad():
        chop_vegetables()       # Режет, пока варится суп
        mix_salad()

    # Готовит оба блюда "одновременно"
    await asyncio.gather(make_soup(), make_salad())

    # Общее время: ~11-12 минут (не 20!)
```

## Реализация однопоточной конкурентности

### Генераторы как основа

Python корутины построены на генераторах:

```python
def simple_generator():
    """Генератор - приостанавливаемая функция"""
    print("Шаг 1")
    yield 1
    print("Шаг 2")
    yield 2
    print("Шаг 3")
    yield 3

# Использование
gen = simple_generator()
print(next(gen))  # Шаг 1 -> 1
print(next(gen))  # Шаг 2 -> 2
print(next(gen))  # Шаг 3 -> 3
```

### Корутины на основе генераторов (старый стиль)

```python
def old_style_coroutine():
    """Корутина до Python 3.5"""
    while True:
        value = yield  # Приостанавливается и ждёт значение
        print(f"Получено: {value}")

coro = old_style_coroutine()
next(coro)           # Запускаем корутину
coro.send("Hello")   # Получено: Hello
coro.send("World")   # Получено: World
```

### Современные корутины (async/await)

```python
async def modern_coroutine():
    """Корутина Python 3.5+"""
    print("Начало")
    await asyncio.sleep(1)  # Приостанавливаемся
    print("Середина")
    await asyncio.sleep(1)  # Снова приостанавливаемся
    print("Конец")
    return "Результат"
```

## Как работает переключение контекста

### Ручной планировщик

```python
from collections import deque

class SimpleScheduler:
    """Простой планировщик для демонстрации"""

    def __init__(self):
        self.ready = deque()  # Очередь готовых задач
        self.current = None

    def add_task(self, coro):
        """Добавляет корутину в очередь"""
        self.ready.append(coro)

    def run(self):
        """Выполняет все задачи"""
        while self.ready:
            self.current = self.ready.popleft()
            try:
                # Выполняем до следующего yield
                self.current.send(None)
                # Если не закончилась - возвращаем в очередь
                self.ready.append(self.current)
            except StopIteration:
                # Корутина завершилась
                pass

def task(name, steps):
    """Задача, которая уступает контроль на каждом шаге"""
    for i in range(steps):
        print(f"{name}: шаг {i + 1}")
        yield  # Уступаем контроль планировщику

# Демонстрация
scheduler = SimpleScheduler()
scheduler.add_task(task("Задача A", 3))
scheduler.add_task(task("Задача B", 3))
scheduler.add_task(task("Задача C", 3))
scheduler.run()

# Вывод:
# Задача A: шаг 1
# Задача B: шаг 1
# Задача C: шаг 1
# Задача A: шаг 2
# Задача B: шаг 2
# Задача C: шаг 2
# ...
```

## Точки переключения в asyncio

### Явные точки переключения

```python
import asyncio

async def example():
    # Каждый await - потенциальная точка переключения

    await asyncio.sleep(1)      # Переключение

    result = await fetch(url)   # Переключение при ожидании I/O

    async with lock:            # Переключение при ожидании блокировки
        pass

    async for item in stream:   # Переключение при ожидании данных
        pass
```

### Где НЕ происходит переключение

```python
async def no_switch_example():
    # Обычный Python-код выполняется без переключений

    result = 0
    for i in range(1000000):
        result += i  # Нет await = нет переключения!

    # Этот код БЛОКИРУЕТ event loop!
    # Другие задачи не смогут выполняться

    return result
```

### Исправление блокирующего кода

```python
async def non_blocking_computation():
    """Добавляем точки переключения в тяжёлые вычисления"""
    result = 0
    for i in range(1000000):
        result += i

        # Периодически уступаем контроль
        if i % 10000 == 0:
            await asyncio.sleep(0)  # "Нулевое" ожидание

    return result

# Или выносим в отдельный поток/процесс
async def proper_heavy_computation():
    loop = asyncio.get_running_loop()
    result = await loop.run_in_executor(None, heavy_sync_function)
    return result
```

## Преимущества однопоточной конкурентности

### 1. Отсутствие race conditions

```python
# Многопоточность: race condition возможен
counter = 0

def thread_unsafe():
    global counter
    temp = counter
    temp += 1
    counter = temp  # Другой поток мог изменить counter!

# Asyncio: race condition невозможен между await
counter = 0

async def async_safe():
    global counter
    temp = counter
    temp += 1
    counter = temp  # Гарантированно атомарно!

    await something()  # Только здесь может произойти переключение
```

### 2. Нет необходимости в блокировках

```python
# Многопоточность: нужны блокировки
import threading

lock = threading.Lock()
shared_data = []

def thread_append(item):
    with lock:  # Обязательно!
        shared_data.append(item)

# Asyncio: блокировки обычно не нужны
shared_data = []

async def async_append(item):
    shared_data.append(item)  # Безопасно!
    await something()
    # Только после await данные могут измениться
```

### 3. Меньше потребление памяти

```python
import asyncio
import threading
import sys

# Поток занимает ~8MB стека
thread = threading.Thread(target=lambda: None)
# thread.stack_size()  # ~8MB по умолчанию

# Корутина занимает ~1KB
async def tiny_coroutine():
    await asyncio.sleep(0)

coro = tiny_coroutine()
# Размер объекта корутины минимален

# Результат: можно создать 100,000+ корутин
# но только ~1000 потоков
```

### 4. Предсказуемое поведение

```python
async def predictable():
    print("A")
    # Точно известно: между этими print'ами
    # никто не вмешается
    print("B")

    await asyncio.sleep(0)
    # А вот здесь могут выполниться другие задачи

    print("C")
```

## Недостатки однопоточной конкурентности

### 1. Блокирующий код останавливает всё

```python
import time
import asyncio

async def blocking_disaster():
    print("Начало")

    time.sleep(5)  # ПЛОХО! Блокирует весь event loop!

    print("Конец")

async def other_task():
    while True:
        print("Я работаю!")
        await asyncio.sleep(1)

async def main():
    await asyncio.gather(
        blocking_disaster(),
        other_task()
    )
    # other_task НЕ будет печатать 5 секунд!

# Решение: используйте неблокирующие альтернативы
async def non_blocking():
    await asyncio.sleep(5)  # ХОРОШО!
```

### 2. Не ускоряет CPU-bound задачи

```python
async def cpu_bound_still_slow():
    # Это НЕ будет выполняться параллельно
    results = await asyncio.gather(
        compute_heavy(1000000),
        compute_heavy(1000000),
        compute_heavy(1000000)
    )
    # Общее время = сумма всех вычислений

# Решение: ProcessPoolExecutor
async def cpu_bound_fixed():
    loop = asyncio.get_running_loop()
    with ProcessPoolExecutor() as executor:
        results = await asyncio.gather(
            loop.run_in_executor(executor, compute_heavy, 1000000),
            loop.run_in_executor(executor, compute_heavy, 1000000),
            loop.run_in_executor(executor, compute_heavy, 1000000)
        )
    # Теперь выполняется параллельно в процессах
```

## Сравнение подходов

| Аспект | Многопоточность | Однопоточная конкурентность |
|--------|----------------|---------------------------|
| Переключение | Вытесняющее (ОС) | Кооперативное (await) |
| Race conditions | Возможны | Контролируемы |
| Блокировки | Необходимы | Обычно не нужны |
| Память на задачу | ~8MB | ~1KB |
| CPU-bound | Не эффективно (GIL) | Не эффективно |
| I/O-bound | Эффективно | Очень эффективно |
| Отладка | Сложная | Проще |

## Практический пример: Chat Server

```python
import asyncio
from typing import Set

class ChatServer:
    """Однопоточный чат-сервер на тысячи соединений"""

    def __init__(self):
        self.clients: Set[asyncio.StreamWriter] = set()

    async def handle_client(self, reader: asyncio.StreamReader,
                           writer: asyncio.StreamWriter):
        """Обработка одного клиента"""
        self.clients.add(writer)
        addr = writer.get_extra_info('peername')
        print(f"Новое подключение: {addr}")

        try:
            while True:
                data = await reader.readline()  # await = точка переключения
                if not data:
                    break

                message = data.decode().strip()
                print(f"{addr}: {message}")

                # Рассылка всем клиентам
                await self.broadcast(f"{addr}: {message}\n")
        finally:
            self.clients.discard(writer)
            writer.close()
            await writer.wait_closed()
            print(f"Отключение: {addr}")

    async def broadcast(self, message: str):
        """Отправка сообщения всем клиентам"""
        data = message.encode()
        for client in self.clients:
            try:
                client.write(data)
                await client.drain()  # await = точка переключения
            except Exception:
                pass

    async def start(self, host: str = '127.0.0.1', port: int = 8888):
        server = await asyncio.start_server(
            self.handle_client, host, port
        )
        print(f"Сервер запущен на {host}:{port}")

        async with server:
            await server.serve_forever()

# Запуск
async def main():
    chat = ChatServer()
    await chat.start()

asyncio.run(main())
```

## Заключение

Однопоточная конкурентность - мощная парадигма для I/O-bound задач:

- **Кооперативная многозадачность** через await
- **Нет проблем синхронизации** между точками переключения
- **Высокая масштабируемость** (тысячи соединений)
- **Предсказуемое поведение** программы

Главное правило: **никогда не блокируйте event loop** - используйте await для всех операций ожидания.

---

[prev: ./05-gil.md](./05-gil.md) | [next: ./07-event-loop-intro.md](./07-event-loop-intro.md)
