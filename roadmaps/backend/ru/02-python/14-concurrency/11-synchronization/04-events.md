# Уведомление задач с помощью событий

[prev: ./03-semaphores.md](./03-semaphores.md) | [next: ./05-conditions.md](./05-conditions.md)

---

## Что такое Event?

**Event** (событие) - это примитив синхронизации, который позволяет одной или нескольким корутинам ожидать наступления определённого события. Событие имеет два состояния:
- **Установлено (set)** - сигнал "можно продолжать"
- **Сброшено (cleared)** - сигнал "нужно ждать"

### Аналогия

Представьте стартовый пистолет на соревнованиях:
- Бегуны ждут выстрела (`await event.wait()`)
- Судья стреляет (`event.set()`)
- Все бегуны одновременно начинают бег

## Базовое использование

### Создание и использование Event

```python
import asyncio

event = asyncio.Event()

async def waiter(name: str):
    print(f"{name}: жду сигнала...")
    await event.wait()
    print(f"{name}: сигнал получен!")


async def setter():
    print("Setter: готовлюсь...")
    await asyncio.sleep(2)
    print("Setter: отправляю сигнал!")
    event.set()


async def main():
    # Запускаем ожидающие корутины и установщик
    await asyncio.gather(
        waiter("Worker-1"),
        waiter("Worker-2"),
        waiter("Worker-3"),
        setter()
    )


asyncio.run(main())
```

### Вывод

```
Worker-1: жду сигнала...
Worker-2: жду сигнала...
Worker-3: жду сигнала...
Setter: готовлюсь...
Setter: отправляю сигнал!
Worker-1: сигнал получен!
Worker-2: сигнал получен!
Worker-3: сигнал получен!
```

## Методы Event

### set() - установка события

```python
import asyncio

event = asyncio.Event()

async def demo():
    print(f"Установлено: {event.is_set()}")  # False
    event.set()
    print(f"Установлено: {event.is_set()}")  # True


asyncio.run(demo())
```

### clear() - сброс события

```python
import asyncio

event = asyncio.Event()

async def demo():
    event.set()
    print(f"Установлено: {event.is_set()}")  # True

    event.clear()
    print(f"Установлено: {event.is_set()}")  # False


asyncio.run(demo())
```

### wait() - ожидание события

```python
import asyncio

event = asyncio.Event()

async def waiter():
    print("Ожидаю...")
    await event.wait()  # Блокируется до set()
    print("Дождался!")


async def main():
    task = asyncio.create_task(waiter())
    await asyncio.sleep(1)
    event.set()  # Разблокируем waiter
    await task


asyncio.run(main())
```

### is_set() - проверка состояния

```python
import asyncio

event = asyncio.Event()

async def conditional_action():
    if event.is_set():
        print("Событие уже произошло")
    else:
        print("Ждём события...")
        await event.wait()
        print("Событие произошло!")
```

## Практические примеры

### 1. Инициализация приложения

```python
import asyncio

class Application:
    def __init__(self):
        self._ready = asyncio.Event()
        self._shutdown = asyncio.Event()
        self._db = None
        self._cache = None

    async def start(self):
        """Инициализация компонентов"""
        print("Запуск приложения...")

        # Параллельная инициализация
        await asyncio.gather(
            self._init_database(),
            self._init_cache()
        )

        self._ready.set()
        print("Приложение готово!")

    async def _init_database(self):
        await asyncio.sleep(1)  # Симуляция подключения
        self._db = {"connected": True}
        print("База данных подключена")

    async def _init_cache(self):
        await asyncio.sleep(0.5)  # Симуляция подключения
        self._cache = {"connected": True}
        print("Кэш подключён")

    async def wait_ready(self):
        """Ожидание готовности приложения"""
        await self._ready.wait()

    async def shutdown(self):
        """Graceful shutdown"""
        print("Начинаем завершение...")
        self._shutdown.set()

    async def wait_shutdown(self):
        await self._shutdown.wait()


async def worker(app: Application, worker_id: int):
    print(f"Worker-{worker_id}: жду готовности приложения...")
    await app.wait_ready()
    print(f"Worker-{worker_id}: начинаю работу!")

    # Работаем до сигнала завершения
    while not app._shutdown.is_set():
        await asyncio.sleep(0.5)
        print(f"Worker-{worker_id}: работаю...")

    print(f"Worker-{worker_id}: завершаю работу")


async def main():
    app = Application()

    # Запускаем воркеры до инициализации приложения
    workers = [
        asyncio.create_task(worker(app, i))
        for i in range(3)
    ]

    # Инициализируем приложение
    await app.start()

    # Даём поработать
    await asyncio.sleep(2)

    # Graceful shutdown
    await app.shutdown()
    await asyncio.gather(*workers)

    print("Приложение завершено")


asyncio.run(main())
```

### 2. Координация этапов работы

```python
import asyncio

class Pipeline:
    def __init__(self):
        self._stage_complete = {}

    def create_stage(self, name: str) -> asyncio.Event:
        event = asyncio.Event()
        self._stage_complete[name] = event
        return event

    async def wait_for_stage(self, name: str):
        if name in self._stage_complete:
            await self._stage_complete[name].wait()


async def data_loader(pipeline: Pipeline):
    print("Загрузка данных...")
    await asyncio.sleep(1)
    print("Данные загружены")
    pipeline._stage_complete["data_loaded"].set()


async def data_processor(pipeline: Pipeline):
    await pipeline.wait_for_stage("data_loaded")
    print("Обработка данных...")
    await asyncio.sleep(1)
    print("Данные обработаны")
    pipeline._stage_complete["data_processed"].set()


async def data_saver(pipeline: Pipeline):
    await pipeline.wait_for_stage("data_processed")
    print("Сохранение данных...")
    await asyncio.sleep(0.5)
    print("Данные сохранены")
    pipeline._stage_complete["data_saved"].set()


async def main():
    pipeline = Pipeline()

    # Создаём события для каждого этапа
    pipeline.create_stage("data_loaded")
    pipeline.create_stage("data_processed")
    pipeline.create_stage("data_saved")

    # Запускаем все этапы - они сами определят порядок
    await asyncio.gather(
        data_saver(pipeline),    # Ждёт data_processed
        data_processor(pipeline), # Ждёт data_loaded
        data_loader(pipeline)     # Начинает сразу
    )

    print("Pipeline завершён!")


asyncio.run(main())
```

### 3. Периодические задачи с паузой

```python
import asyncio

class PeriodicTask:
    def __init__(self, interval: float):
        self._interval = interval
        self._pause_event = asyncio.Event()
        self._pause_event.set()  # Изначально не на паузе
        self._stop_event = asyncio.Event()
        self._task = None

    async def start(self, callback):
        self._task = asyncio.create_task(self._run(callback))

    async def _run(self, callback):
        while not self._stop_event.is_set():
            # Ждём, если на паузе
            await self._pause_event.wait()

            if self._stop_event.is_set():
                break

            await callback()
            await asyncio.sleep(self._interval)

    def pause(self):
        """Приостановить выполнение"""
        self._pause_event.clear()
        print("Задача приостановлена")

    def resume(self):
        """Возобновить выполнение"""
        self._pause_event.set()
        print("Задача возобновлена")

    async def stop(self):
        """Остановить задачу"""
        self._stop_event.set()
        self._pause_event.set()  # Разблокировать, если на паузе
        if self._task:
            await self._task


async def my_job():
    print(f"Выполняю работу в {asyncio.get_event_loop().time():.1f}")


async def main():
    task = PeriodicTask(interval=0.5)
    await task.start(my_job)

    await asyncio.sleep(2)
    task.pause()

    await asyncio.sleep(2)  # Пауза - ничего не выполняется
    task.resume()

    await asyncio.sleep(2)
    await task.stop()

    print("Задача остановлена")


asyncio.run(main())
```

### 4. Барьер синхронизации

```python
import asyncio

class Barrier:
    """Барьер - все должны дойти до точки, прежде чем продолжить"""

    def __init__(self, parties: int):
        self._parties = parties
        self._count = 0
        self._event = asyncio.Event()
        self._lock = asyncio.Lock()

    async def wait(self):
        async with self._lock:
            self._count += 1
            if self._count >= self._parties:
                self._event.set()

        await self._event.wait()

    def reset(self):
        self._count = 0
        self._event.clear()


async def worker(barrier: Barrier, worker_id: int):
    # Фаза 1
    print(f"Worker-{worker_id}: завершил фазу 1")
    await asyncio.sleep(worker_id * 0.3)  # Разное время завершения

    await barrier.wait()  # Ждём всех
    print(f"Worker-{worker_id}: все завершили фазу 1, начинаю фазу 2")


async def main():
    barrier = Barrier(parties=3)

    await asyncio.gather(
        worker(barrier, 1),
        worker(barrier, 2),
        worker(barrier, 3)
    )


asyncio.run(main())
```

### 5. Отмена с graceful shutdown

```python
import asyncio

class GracefulService:
    def __init__(self):
        self._shutdown_event = asyncio.Event()

    async def run(self):
        """Основной цикл сервиса"""
        print("Сервис запущен")

        while not self._shutdown_event.is_set():
            try:
                # Ждём с таймаутом, чтобы периодически проверять shutdown
                await asyncio.wait_for(
                    self._do_work(),
                    timeout=1.0
                )
            except asyncio.TimeoutError:
                pass

        print("Сервис остановлен")

    async def _do_work(self):
        # Основная работа
        await asyncio.sleep(0.1)
        print(".", end="", flush=True)

    def shutdown(self):
        print("\nПолучен сигнал завершения")
        self._shutdown_event.set()


async def main():
    service = GracefulService()

    # Запускаем сервис
    service_task = asyncio.create_task(service.run())

    # Симуляция работы
    await asyncio.sleep(3)

    # Graceful shutdown
    service.shutdown()
    await service_task


asyncio.run(main())
```

## Event с таймаутом

```python
import asyncio

event = asyncio.Event()

async def wait_with_timeout(timeout: float) -> bool:
    try:
        await asyncio.wait_for(event.wait(), timeout=timeout)
        return True
    except asyncio.TimeoutError:
        return False


async def demo():
    # Событие не установлено - таймаут
    result = await wait_with_timeout(1.0)
    print(f"Результат: {result}")  # False

    # Устанавливаем событие
    event.set()

    # Теперь wait() вернётся сразу
    result = await wait_with_timeout(1.0)
    print(f"Результат: {result}")  # True


asyncio.run(demo())
```

## Разница между Event и Lock

| Характеристика | Event | Lock |
|----------------|-------|------|
| Состояние | set/clear | locked/unlocked |
| Количество ожидающих | Неограничено | Один получает |
| Автосброс | Нет | Да (при release) |
| Использование | Сигнализация | Взаимоисключение |

```python
import asyncio

# Event - все ожидающие получают сигнал одновременно
event = asyncio.Event()

async def event_demo():
    event.set()
    # Все wait() завершатся одновременно
    await asyncio.gather(
        event.wait(),
        event.wait(),
        event.wait()
    )
    print("Все три wait() завершились")


# Lock - только один получает доступ
lock = asyncio.Lock()

async def lock_demo():
    async def worker(n):
        async with lock:
            print(f"Worker {n} работает")
            await asyncio.sleep(0.1)

    # Выполняются последовательно!
    await asyncio.gather(
        worker(1),
        worker(2),
        worker(3)
    )


asyncio.run(event_demo())
asyncio.run(lock_demo())
```

## Распространённые ошибки

### 1. Забыли сбросить Event

```python
import asyncio

event = asyncio.Event()

async def bad_pattern():
    event.set()

    # Повторный wait() сразу вернётся!
    await event.wait()  # Не блокируется
    print("Это выполнится сразу")


async def good_pattern():
    event.set()

    # Сбрасываем для повторного использования
    event.clear()

    # Теперь wait() будет ждать
    # await event.wait()  # Будет ждать следующего set()
```

### 2. Race condition при проверке is_set()

```python
import asyncio

event = asyncio.Event()

# ПЛОХО: race condition
async def bad_check():
    if not event.is_set():  # Может измениться!
        # Делаем что-то, думая что событие не установлено
        pass

# ХОРОШО: просто ждём
async def good_approach():
    await event.wait()  # Атомарная операция
    # Гарантированно событие установлено
```

### 3. Использование Event вместо Condition

```python
import asyncio

# ПЛОХО: Event для producer-consumer
async def bad_consumer(event, data):
    while True:
        await event.wait()
        event.clear()  # Race condition с другими consumer!
        item = data.pop()  # Может быть пусто!

# ХОРОШО: используйте Condition или Queue
# (см. следующую тему)
```

## AutoResetEvent (самодельный)

Python не имеет встроенного AutoResetEvent, но его легко реализовать:

```python
import asyncio

class AutoResetEvent:
    """Event, который автоматически сбрасывается после wait()"""

    def __init__(self):
        self._event = asyncio.Event()
        self._lock = asyncio.Lock()

    async def wait(self):
        await self._event.wait()
        async with self._lock:
            self._event.clear()

    def set(self):
        self._event.set()

    def is_set(self):
        return self._event.is_set()


async def demo():
    event = AutoResetEvent()

    async def waiter(name):
        await event.wait()
        print(f"{name}: получил сигнал!")

    # Только один waiter получит каждый сигнал
    asyncio.create_task(waiter("A"))
    asyncio.create_task(waiter("B"))

    await asyncio.sleep(0.1)
    event.set()  # Только один получит

    await asyncio.sleep(0.1)
    event.set()  # Второй получит

    await asyncio.sleep(0.1)


asyncio.run(demo())
```

## Best Practices

1. **Используйте Event для сигнализации** - когда нужно уведомить о событии
2. **Помните о состоянии** - Event остаётся set() до явного clear()
3. **Не используйте для взаимоисключения** - для этого есть Lock
4. **wait_for для таймаутов** - `await asyncio.wait_for(event.wait(), timeout=...)`
5. **Документируйте семантику** - кто устанавливает, кто сбрасывает
6. **Для producer-consumer используйте Queue** - не Event

## Когда использовать Event

| Сценарий | Event подходит? |
|----------|-----------------|
| Уведомление о готовности | Да |
| Сигнал завершения (shutdown) | Да |
| Барьер синхронизации | Да |
| Взаимоисключение | Нет (Lock) |
| Producer-consumer | Нет (Queue/Condition) |
| Ограничение параллелизма | Нет (Semaphore) |

## Резюме

- `Event` - примитив для сигнализации между корутинами
- Два состояния: `set` (сигнал есть) и `cleared` (сигнала нет)
- `wait()` блокируется до `set()`
- Все ожидающие корутины разблокируются одновременно
- Событие не сбрасывается автоматически - нужен `clear()`
- Идеально для: инициализации, shutdown, координации этапов
- Не подходит для: взаимоисключения, producer-consumer

---

[prev: ./03-semaphores.md](./03-semaphores.md) | [next: ./05-conditions.md](./05-conditions.md)
