# Условия

[prev: ./04-events.md](./04-events.md) | [next: ../12-queues/readme.md](../12-queues/readme.md)

---

## Что такое Condition?

**Condition** (условная переменная) - это продвинутый примитив синхронизации, который комбинирует блокировку с механизмом уведомлений. Condition позволяет корутинам ожидать выполнения определённого условия и уведомлять другие корутины об изменении состояния.

### Отличие от Event

- **Event**: простой сигнал "произошло/не произошло"
- **Condition**: ожидание конкретного условия + взаимоисключающий доступ к данным

### Аналогия

Представьте ресторан:
- **Lock** - только один посетитель может разговаривать с официантом
- **Event** - звонок, что кухня открылась
- **Condition** - официант ждёт, пока блюдо будет готово, затем обслуживает клиента

## Базовое использование

### Создание Condition

```python
import asyncio

# Condition с автоматической блокировкой
condition = asyncio.Condition()

# Condition с явной блокировкой
lock = asyncio.Lock()
condition_with_lock = asyncio.Condition(lock)
```

### Паттерн wait/notify

```python
import asyncio

condition = asyncio.Condition()
data_ready = False
data = None

async def consumer():
    global data
    async with condition:
        # Ждём, пока данные будут готовы
        while not data_ready:
            await condition.wait()
        # Данные готовы - обрабатываем
        print(f"Consumer получил: {data}")


async def producer():
    global data_ready, data
    await asyncio.sleep(1)  # Симуляция подготовки данных

    async with condition:
        data = "Важные данные"
        data_ready = True
        condition.notify()  # Уведомляем одного ожидающего


async def main():
    await asyncio.gather(
        consumer(),
        producer()
    )


asyncio.run(main())
```

## Методы Condition

### wait() - ожидание уведомления

```python
import asyncio

condition = asyncio.Condition()

async def demo():
    async with condition:
        print("Начинаю ожидание...")
        # wait() освобождает блокировку и ждёт notify
        await condition.wait()
        # После notify блокировка снова захвачена
        print("Дождался!")
```

### wait_for() - ожидание с предикатом

```python
import asyncio

condition = asyncio.Condition()
count = 0

async def wait_for_count(target: int):
    async with condition:
        # wait_for автоматически проверяет условие
        await condition.wait_for(lambda: count >= target)
        print(f"Счётчик достиг {count}")


async def increment():
    global count
    for i in range(10):
        async with condition:
            count += 1
            print(f"Счётчик: {count}")
            condition.notify_all()
        await asyncio.sleep(0.1)


async def main():
    await asyncio.gather(
        wait_for_count(5),
        increment()
    )


asyncio.run(main())
```

### notify() - уведомление одного ожидающего

```python
import asyncio

condition = asyncio.Condition()

async def waiter(name: str):
    async with condition:
        print(f"{name}: ожидаю")
        await condition.wait()
        print(f"{name}: получил уведомление!")


async def notifier():
    await asyncio.sleep(0.5)
    async with condition:
        print("Уведомляю ОДНОГО")
        condition.notify()  # Только один проснётся


async def main():
    await asyncio.gather(
        waiter("A"),
        waiter("B"),
        waiter("C"),
        notifier()
    )
    # Только один из A, B, C получит уведомление!


asyncio.run(main())
```

### notify_all() - уведомление всех ожидающих

```python
import asyncio

condition = asyncio.Condition()

async def waiter(name: str):
    async with condition:
        print(f"{name}: ожидаю")
        await condition.wait()
        print(f"{name}: получил уведомление!")


async def notifier():
    await asyncio.sleep(0.5)
    async with condition:
        print("Уведомляю ВСЕХ")
        condition.notify_all()  # Все проснутся


async def main():
    await asyncio.gather(
        waiter("A"),
        waiter("B"),
        waiter("C"),
        notifier()
    )
    # Все A, B, C получат уведомление


asyncio.run(main())
```

## Практические примеры

### 1. Producer-Consumer с буфером

```python
import asyncio
from collections import deque

class BoundedBuffer:
    def __init__(self, capacity: int):
        self._capacity = capacity
        self._buffer = deque()
        self._condition = asyncio.Condition()

    async def put(self, item):
        async with self._condition:
            # Ждём, пока освободится место
            while len(self._buffer) >= self._capacity:
                await self._condition.wait()

            self._buffer.append(item)
            print(f"Добавлено: {item}, размер буфера: {len(self._buffer)}")

            # Уведомляем потребителей
            self._condition.notify()

    async def get(self):
        async with self._condition:
            # Ждём, пока появятся данные
            while len(self._buffer) == 0:
                await self._condition.wait()

            item = self._buffer.popleft()
            print(f"Получено: {item}, размер буфера: {len(self._buffer)}")

            # Уведомляем производителей
            self._condition.notify()
            return item


async def producer(buffer: BoundedBuffer, producer_id: int):
    for i in range(5):
        await buffer.put(f"P{producer_id}-{i}")
        await asyncio.sleep(0.1)


async def consumer(buffer: BoundedBuffer, consumer_id: int):
    for _ in range(5):
        item = await buffer.get()
        await asyncio.sleep(0.2)  # Обработка медленнее


async def main():
    buffer = BoundedBuffer(capacity=3)

    await asyncio.gather(
        producer(buffer, 1),
        producer(buffer, 2),
        consumer(buffer, 1),
        consumer(buffer, 2)
    )


asyncio.run(main())
```

### 2. Read-Write Lock

```python
import asyncio

class ReadWriteLock:
    """Блокировка с множественным чтением и эксклюзивной записью"""

    def __init__(self):
        self._readers = 0
        self._writers = 0
        self._write_waiting = 0
        self._condition = asyncio.Condition()

    async def acquire_read(self):
        async with self._condition:
            # Ждём, пока нет писателей
            while self._writers > 0 or self._write_waiting > 0:
                await self._condition.wait()
            self._readers += 1

    async def release_read(self):
        async with self._condition:
            self._readers -= 1
            if self._readers == 0:
                self._condition.notify_all()

    async def acquire_write(self):
        async with self._condition:
            self._write_waiting += 1
            # Ждём, пока нет читателей и писателей
            while self._readers > 0 or self._writers > 0:
                await self._condition.wait()
            self._write_waiting -= 1
            self._writers += 1

    async def release_write(self):
        async with self._condition:
            self._writers -= 1
            self._condition.notify_all()

    def reader(self):
        return ReadLockContext(self)

    def writer(self):
        return WriteLockContext(self)


class ReadLockContext:
    def __init__(self, rw_lock: ReadWriteLock):
        self._lock = rw_lock

    async def __aenter__(self):
        await self._lock.acquire_read()
        return self

    async def __aexit__(self, *args):
        await self._lock.release_read()


class WriteLockContext:
    def __init__(self, rw_lock: ReadWriteLock):
        self._lock = rw_lock

    async def __aenter__(self):
        await self._lock.acquire_write()
        return self

    async def __aexit__(self, *args):
        await self._lock.release_write()


# Использование
shared_data = {"value": 0}
rw_lock = ReadWriteLock()

async def reader(reader_id: int):
    for _ in range(3):
        async with rw_lock.reader():
            print(f"Reader-{reader_id}: значение = {shared_data['value']}")
            await asyncio.sleep(0.1)


async def writer(writer_id: int):
    for i in range(3):
        async with rw_lock.writer():
            shared_data['value'] = writer_id * 10 + i
            print(f"Writer-{writer_id}: записал {shared_data['value']}")
            await asyncio.sleep(0.2)


async def main():
    await asyncio.gather(
        reader(1),
        reader(2),
        writer(1),
        reader(3)
    )


asyncio.run(main())
```

### 3. Barrier с условием

```python
import asyncio

class ConditionalBarrier:
    """Барьер, который срабатывает при выполнении условия"""

    def __init__(self, parties: int):
        self._parties = parties
        self._count = 0
        self._generation = 0
        self._condition = asyncio.Condition()

    async def wait(self):
        async with self._condition:
            generation = self._generation
            self._count += 1

            if self._count >= self._parties:
                # Последний участник сбрасывает барьер
                self._count = 0
                self._generation += 1
                self._condition.notify_all()
                return True  # Лидер

            # Ждём, пока не соберутся все
            await self._condition.wait_for(
                lambda: generation != self._generation
            )
            return False  # Не лидер


async def worker(barrier: ConditionalBarrier, worker_id: int):
    for phase in range(3):
        print(f"Worker-{worker_id}: выполняет фазу {phase}")
        await asyncio.sleep(worker_id * 0.1)  # Разное время

        is_leader = await barrier.wait()
        if is_leader:
            print(f"=== Фаза {phase} завершена, Worker-{worker_id} - лидер ===")


async def main():
    barrier = ConditionalBarrier(parties=3)

    await asyncio.gather(
        worker(barrier, 1),
        worker(barrier, 2),
        worker(barrier, 3)
    )


asyncio.run(main())
```

### 4. Пул ресурсов с условием

```python
import asyncio

class ResourcePool:
    """Пул ресурсов с ожиданием доступности"""

    def __init__(self, resources: list):
        self._available = list(resources)
        self._in_use = []
        self._condition = asyncio.Condition()

    async def acquire(self, predicate=None):
        """Получить ресурс, опционально с условием"""
        async with self._condition:
            while True:
                # Ищем подходящий ресурс
                for resource in self._available:
                    if predicate is None or predicate(resource):
                        self._available.remove(resource)
                        self._in_use.append(resource)
                        return resource

                # Ждём освобождения ресурса
                await self._condition.wait()

    async def release(self, resource):
        async with self._condition:
            self._in_use.remove(resource)
            self._available.append(resource)
            self._condition.notify_all()


async def demo():
    # Пул серверов с разными характеристиками
    servers = [
        {"name": "server1", "ram": 4, "cpu": 2},
        {"name": "server2", "ram": 8, "cpu": 4},
        {"name": "server3", "ram": 16, "cpu": 8},
    ]

    pool = ResourcePool(servers)

    async def task(task_id: int, min_ram: int):
        print(f"Task-{task_id}: нужно минимум {min_ram}GB RAM")

        # Запрашиваем сервер с достаточной памятью
        server = await pool.acquire(lambda s: s["ram"] >= min_ram)
        print(f"Task-{task_id}: получил {server['name']}")

        await asyncio.sleep(0.5)  # Работа

        await pool.release(server)
        print(f"Task-{task_id}: освободил {server['name']}")

    await asyncio.gather(
        task(1, 16),  # Нужен server3
        task(2, 4),   # Подойдёт любой
        task(3, 8),   # Нужен server2 или server3
        task(4, 4),   # Подойдёт любой
    )


asyncio.run(demo())
```

### 5. Состояние машины (State Machine)

```python
import asyncio
from enum import Enum, auto

class State(Enum):
    IDLE = auto()
    STARTING = auto()
    RUNNING = auto()
    STOPPING = auto()
    STOPPED = auto()

class StateMachine:
    def __init__(self):
        self._state = State.IDLE
        self._condition = asyncio.Condition()

    @property
    def state(self):
        return self._state

    async def wait_for_state(self, *states):
        """Ожидание достижения одного из состояний"""
        async with self._condition:
            await self._condition.wait_for(
                lambda: self._state in states
            )
            return self._state

    async def transition(self, new_state: State):
        async with self._condition:
            old_state = self._state
            self._state = new_state
            print(f"Переход: {old_state.name} -> {new_state.name}")
            self._condition.notify_all()


async def start_sequence(sm: StateMachine):
    await sm.transition(State.STARTING)
    await asyncio.sleep(1)  # Инициализация
    await sm.transition(State.RUNNING)


async def stop_sequence(sm: StateMachine):
    await sm.transition(State.STOPPING)
    await asyncio.sleep(0.5)  # Очистка
    await sm.transition(State.STOPPED)


async def monitor(sm: StateMachine, name: str):
    state = await sm.wait_for_state(State.RUNNING)
    print(f"{name}: сервис запущен!")

    state = await sm.wait_for_state(State.STOPPED)
    print(f"{name}: сервис остановлен!")


async def main():
    sm = StateMachine()

    # Мониторы ждут состояний
    monitors = [
        asyncio.create_task(monitor(sm, "Monitor-1")),
        asyncio.create_task(monitor(sm, "Monitor-2")),
    ]

    # Запуск
    await asyncio.sleep(0.1)  # Даём мониторам начать ждать
    await start_sequence(sm)

    # Работа
    await asyncio.sleep(2)

    # Остановка
    await stop_sequence(sm)

    await asyncio.gather(*monitors)


asyncio.run(main())
```

## Важные особенности

### 1. Проверка условия в цикле

```python
import asyncio

condition = asyncio.Condition()
resource_available = False

# ПРАВИЛЬНО: проверка в цикле while
async def correct_usage():
    async with condition:
        while not resource_available:  # Цикл!
            await condition.wait()
        # Гарантированно resource_available == True


# НЕПРАВИЛЬНО: проверка if
async def incorrect_usage():
    async with condition:
        if not resource_available:  # Опасно!
            await condition.wait()
        # resource_available может быть False!
        # (spurious wakeup или другой потребитель успел первым)
```

### 2. Использование wait_for вместо while

```python
import asyncio

condition = asyncio.Condition()
data = None

# Вместо:
async def with_while():
    async with condition:
        while data is None:
            await condition.wait()

# Используйте:
async def with_wait_for():
    async with condition:
        await condition.wait_for(lambda: data is not None)
```

### 3. Notify внутри контекста

```python
import asyncio

condition = asyncio.Condition()

# ПРАВИЛЬНО: notify внутри async with
async def correct():
    async with condition:
        # ... изменяем состояние ...
        condition.notify()  # Внутри контекста!

# НЕПРАВИЛЬНО: notify снаружи
async def incorrect():
    async with condition:
        pass  # Изменили состояние
    condition.notify()  # RuntimeError!
```

## Сравнение примитивов синхронизации

| Примитив | Для чего | Особенности |
|----------|----------|-------------|
| Lock | Взаимоисключение | Один владелец |
| Semaphore | Ограничение параллелизма | N владельцев |
| Event | Сигнализация | Без блокировки данных |
| Condition | Ожидание условия | Блокировка + уведомления |

## Когда использовать Condition

| Сценарий | Condition подходит? |
|----------|---------------------|
| Producer-Consumer | Да |
| Ожидание изменения состояния | Да |
| Read-Write Lock | Да |
| Простой сигнал | Нет (Event) |
| Взаимоисключение | Нет (Lock) |
| Ограничение параллелизма | Нет (Semaphore) |

## Best Practices

1. **Всегда используйте while** - проверяйте условие в цикле
2. **Предпочитайте wait_for()** - он автоматически проверяет условие
3. **notify внутри async with** - иначе RuntimeError
4. **notify_all для множества ожидающих** - если условие может удовлетворить нескольких
5. **notify для одного ожидающего** - экономит ресурсы
6. **Используйте Queue** - для простых producer-consumer

## Резюме

- `Condition` комбинирует Lock с механизмом уведомлений
- `wait()` освобождает блокировку и ждёт `notify()`
- `wait_for(predicate)` - удобная обёртка с проверкой условия
- `notify()` будит одного ожидающего, `notify_all()` - всех
- Всегда проверяйте условие в цикле `while`
- Идеально для: producer-consumer, ожидание изменения состояния, сложная координация
- Для простых случаев предпочтите `asyncio.Queue`

---

[prev: ./04-events.md](./04-events.md) | [next: ../12-queues/readme.md](../12-queues/readme.md)
