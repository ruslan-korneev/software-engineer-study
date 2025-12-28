# Ручное управление циклом событий

[prev: ./07-pitfalls.md](./07-pitfalls.md) | [next: ./09-debug-mode.md](./09-debug-mode.md)

---

## Введение

В большинстве случаев рекомендуется использовать высокоуровневый API `asyncio.run()`. Однако понимание ручного управления event loop необходимо для:
- Интеграции с существующими фреймворками
- Создания кастомных event loop
- Работы в средах с особыми требованиями
- Отладки и профилирования

## Получение Event Loop

### Современный способ (Python 3.10+)

```python
import asyncio

async def main():
    # Внутри корутины
    loop = asyncio.get_running_loop()
    print(f"Текущий loop: {loop}")

asyncio.run(main())
```

### Устаревший способ (не рекомендуется)

```python
import asyncio

# До Python 3.10 - вызывает DeprecationWarning
loop = asyncio.get_event_loop()

# Если loop не существует, создаёт новый
# Это поведение устарело
```

### Создание нового Event Loop

```python
import asyncio

# Создание нового loop
loop = asyncio.new_event_loop()

# Установка как текущего для потока
asyncio.set_event_loop(loop)

try:
    # Использование loop
    async def my_coro():
        return "результат"

    result = loop.run_until_complete(my_coro())
    print(result)
finally:
    loop.close()
```

## Методы управления Event Loop

### run_until_complete()

Запускает корутину до завершения:

```python
import asyncio

async def fetch_data():
    await asyncio.sleep(1)
    return "данные"

loop = asyncio.new_event_loop()
try:
    result = loop.run_until_complete(fetch_data())
    print(f"Результат: {result}")
finally:
    loop.close()
```

### run_forever()

Запускает event loop бесконечно:

```python
import asyncio

async def periodic_task():
    while True:
        print("Работаю...")
        await asyncio.sleep(1)

def main():
    loop = asyncio.new_event_loop()

    # Запускаем задачу
    loop.create_task(periodic_task())

    try:
        # Запускаем loop навсегда
        loop.run_forever()
    except KeyboardInterrupt:
        print("Прервано пользователем")
    finally:
        loop.close()

# main()  # Запустите и нажмите Ctrl+C для выхода
```

### stop()

Останавливает event loop:

```python
import asyncio

async def stop_after_delay(loop, delay):
    await asyncio.sleep(delay)
    print("Останавливаю loop...")
    loop.stop()

def main():
    loop = asyncio.new_event_loop()

    # Задача остановит loop через 3 секунды
    loop.create_task(stop_after_delay(loop, 3))

    print("Запускаю loop...")
    loop.run_forever()
    print("Loop остановлен")

    loop.close()

main()
```

### is_running() и is_closed()

```python
import asyncio

async def check_loop_state():
    loop = asyncio.get_running_loop()

    print(f"Loop запущен: {loop.is_running()}")  # True
    print(f"Loop закрыт: {loop.is_closed()}")    # False

asyncio.run(check_loop_state())
```

## Планирование вызовов

### call_soon()

Планирует callback для выполнения на следующей итерации:

```python
import asyncio

def callback(message):
    print(f"Callback: {message}")

async def main():
    loop = asyncio.get_running_loop()

    # Планируем callback
    loop.call_soon(callback, "первый")
    loop.call_soon(callback, "второй")

    print("Callbacks запланированы")
    await asyncio.sleep(0)  # Даём выполниться callbacks
    print("Готово")

asyncio.run(main())
# Вывод:
# Callbacks запланированы
# Callback: первый
# Callback: второй
# Готово
```

### call_later()

Планирует callback через указанное время:

```python
import asyncio

def delayed_callback(name):
    print(f"Callback {name} выполнен")

async def main():
    loop = asyncio.get_running_loop()

    # Выполнится через 2 секунды
    loop.call_later(2, delayed_callback, "A")

    # Выполнится через 1 секунду
    loop.call_later(1, delayed_callback, "B")

    print("Callbacks запланированы")
    await asyncio.sleep(3)
    print("Готово")

asyncio.run(main())
# Вывод:
# Callbacks запланированы
# (через 1 сек) Callback B выполнен
# (через 2 сек) Callback A выполнен
# Готово
```

### call_at()

Планирует callback на конкретное время (loop time):

```python
import asyncio

def timed_callback(scheduled_time):
    loop = asyncio.get_running_loop()
    actual_time = loop.time()
    print(f"Запланировано: {scheduled_time:.2f}, Выполнено: {actual_time:.2f}")

async def main():
    loop = asyncio.get_running_loop()
    now = loop.time()

    # Выполнить через 1.5 секунды от текущего времени loop
    loop.call_at(now + 1.5, timed_callback, now + 1.5)

    await asyncio.sleep(2)

asyncio.run(main())
```

## Работа с потоками

### call_soon_threadsafe()

Безопасно планирует callback из другого потока:

```python
import asyncio
import threading
import time

def thread_worker(loop):
    """Функция, выполняющаяся в другом потоке."""
    time.sleep(1)
    print("Thread: планирую callback")
    loop.call_soon_threadsafe(lambda: print("Callback из потока!"))

async def main():
    loop = asyncio.get_running_loop()

    # Запускаем поток
    thread = threading.Thread(target=thread_worker, args=(loop,))
    thread.start()

    # Ждём завершения
    await asyncio.sleep(2)
    thread.join()

asyncio.run(main())
```

### run_in_executor()

Выполняет блокирующий код в отдельном потоке или процессе:

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import time

def blocking_io():
    """Блокирующая I/O операция."""
    time.sleep(1)
    return "I/O завершён"

def cpu_intensive(n):
    """CPU-интенсивная операция."""
    return sum(i * i for i in range(n))

async def main():
    loop = asyncio.get_running_loop()

    # С ThreadPoolExecutor (по умолчанию)
    result1 = await loop.run_in_executor(None, blocking_io)
    print(f"Thread result: {result1}")

    # С кастомным ThreadPoolExecutor
    with ThreadPoolExecutor(max_workers=4) as executor:
        result2 = await loop.run_in_executor(executor, blocking_io)
        print(f"Custom thread result: {result2}")

    # С ProcessPoolExecutor для CPU-задач
    with ProcessPoolExecutor() as executor:
        result3 = await loop.run_in_executor(executor, cpu_intensive, 1000000)
        print(f"Process result: {result3}")

asyncio.run(main())
```

## Кастомный Event Loop

### Использование uvloop (высокопроизводительный loop)

```python
# pip install uvloop
import asyncio

try:
    import uvloop
    asyncio.set_event_loop_policy(uvloop.EventLoopPolicy())
    print("Используется uvloop")
except ImportError:
    print("uvloop не установлен, используется стандартный loop")

async def main():
    loop = asyncio.get_running_loop()
    print(f"Тип loop: {type(loop)}")

asyncio.run(main())
```

### Создание собственной политики Event Loop

```python
import asyncio

class CustomEventLoopPolicy(asyncio.DefaultEventLoopPolicy):
    """Кастомная политика для event loop."""

    def new_event_loop(self):
        loop = super().new_event_loop()
        # Настройка loop
        loop.set_debug(True)
        return loop

# Установка кастомной политики
asyncio.set_event_loop_policy(CustomEventLoopPolicy())

async def main():
    loop = asyncio.get_running_loop()
    print(f"Debug mode: {loop.get_debug()}")

asyncio.run(main())
```

## Интеграция с фреймворками

### Интеграция с GUI (Tkinter)

```python
import asyncio
import tkinter as tk

class AsyncTkApp:
    def __init__(self):
        self.root = tk.Tk()
        self.root.title("Async Tkinter")

        self.label = tk.Label(self.root, text="Нажмите кнопку")
        self.label.pack()

        self.button = tk.Button(
            self.root,
            text="Запустить async задачу",
            command=self.on_button_click
        )
        self.button.pack()

        self.loop = asyncio.new_event_loop()

    def on_button_click(self):
        self.label.config(text="Загрузка...")
        # Запускаем async задачу
        self.loop.run_until_complete(self.async_task())

    async def async_task(self):
        await asyncio.sleep(2)  # Имитация загрузки
        self.label.config(text="Готово!")

    def run(self):
        self.root.mainloop()
        self.loop.close()

# app = AsyncTkApp()
# app.run()
```

### Интеграция с существующим синхронным кодом

```python
import asyncio
from contextlib import contextmanager

@contextmanager
def async_context():
    """Контекстный менеджер для запуска async кода в синхронном контексте."""
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)
    try:
        yield loop
    finally:
        # Завершаем все pending задачи
        pending = asyncio.all_tasks(loop)
        for task in pending:
            task.cancel()
        loop.run_until_complete(asyncio.gather(*pending, return_exceptions=True))
        loop.close()

def sync_function():
    """Синхронная функция, которая хочет вызвать async код."""
    async def async_operation():
        await asyncio.sleep(1)
        return "async результат"

    with async_context() as loop:
        result = loop.run_until_complete(async_operation())
        return result

print(sync_function())
```

## Обработчики сигналов

```python
import asyncio
import signal

async def main():
    loop = asyncio.get_running_loop()

    # Обработчик для SIGINT (Ctrl+C)
    def handle_sigint():
        print("Получен SIGINT, завершаю...")
        # Отменяем все задачи
        for task in asyncio.all_tasks(loop):
            task.cancel()

    # Регистрируем обработчик (только Unix)
    try:
        loop.add_signal_handler(signal.SIGINT, handle_sigint)
        loop.add_signal_handler(signal.SIGTERM, handle_sigint)
    except NotImplementedError:
        # Windows не поддерживает add_signal_handler
        pass

    try:
        print("Работаю... (Ctrl+C для выхода)")
        while True:
            await asyncio.sleep(1)
            print("Tick")
    except asyncio.CancelledError:
        print("Задача отменена")

# asyncio.run(main())
```

## Мониторинг Event Loop

```python
import asyncio
import time

class EventLoopMonitor:
    """Мониторинг производительности event loop."""

    def __init__(self, loop):
        self.loop = loop
        self.slow_callbacks = []

    def start(self):
        # Устанавливаем callback для медленных операций
        self.loop.slow_callback_duration = 0.1  # 100ms

        # Включаем отладку для отслеживания медленных callbacks
        self.loop.set_debug(True)

    def get_stats(self):
        return {
            "is_running": self.loop.is_running(),
            "is_closed": self.loop.is_closed(),
            "time": self.loop.time()
        }

async def monitored_operation():
    loop = asyncio.get_running_loop()
    monitor = EventLoopMonitor(loop)
    monitor.start()

    # Некоторые операции
    await asyncio.sleep(1)

    print(monitor.get_stats())

asyncio.run(monitored_operation())
```

## Паттерны использования

### Паттерн: Graceful Shutdown

```python
import asyncio
import signal

class Application:
    def __init__(self):
        self.loop = None
        self.tasks = set()

    async def start_background_tasks(self):
        """Запуск фоновых задач."""
        async def worker(name):
            while True:
                print(f"Worker {name}: работаю")
                await asyncio.sleep(1)

        for i in range(3):
            task = asyncio.create_task(worker(f"W{i}"))
            self.tasks.add(task)
            task.add_done_callback(self.tasks.discard)

    async def shutdown(self):
        """Корректное завершение."""
        print("Завершаю работу...")

        # Отменяем все задачи
        for task in self.tasks:
            task.cancel()

        # Ждём завершения с обработкой исключений
        await asyncio.gather(*self.tasks, return_exceptions=True)
        print("Все задачи завершены")

    def run(self):
        self.loop = asyncio.new_event_loop()
        asyncio.set_event_loop(self.loop)

        try:
            self.loop.run_until_complete(self.start_background_tasks())
            self.loop.run_forever()
        except KeyboardInterrupt:
            pass
        finally:
            self.loop.run_until_complete(self.shutdown())
            self.loop.close()

# app = Application()
# app.run()
```

### Паттерн: Event Loop в отдельном потоке

```python
import asyncio
import threading
from queue import Queue

class AsyncWorker:
    """Async worker в отдельном потоке."""

    def __init__(self):
        self.loop = None
        self.thread = None
        self.queue = Queue()

    def start(self):
        self.thread = threading.Thread(target=self._run_loop)
        self.thread.start()

    def _run_loop(self):
        self.loop = asyncio.new_event_loop()
        asyncio.set_event_loop(self.loop)
        self.loop.run_forever()
        self.loop.close()

    def submit(self, coro):
        """Отправляет корутину на выполнение."""
        future = asyncio.run_coroutine_threadsafe(coro, self.loop)
        return future

    def stop(self):
        self.loop.call_soon_threadsafe(self.loop.stop)
        self.thread.join()

# Использование
async def async_task(x):
    await asyncio.sleep(1)
    return x * 2

worker = AsyncWorker()
worker.start()

# Отправляем задачи из основного потока
future = worker.submit(async_task(21))
result = future.result(timeout=5)
print(f"Результат: {result}")

worker.stop()
```

## Best Practices

1. **Используйте `asyncio.run()`** когда возможно - это наиболее безопасный способ
2. **Не создавайте loop в модуле** - создавайте при необходимости
3. **Всегда закрывайте loop** в блоке finally
4. **Используйте `run_in_executor`** для блокирующего кода
5. **Применяйте `call_soon_threadsafe`** для межпоточного взаимодействия

```python
import asyncio

def best_practice_example():
    """Пример правильного ручного управления event loop."""
    loop = asyncio.new_event_loop()
    asyncio.set_event_loop(loop)

    async def main():
        # Ваш async код
        await asyncio.sleep(1)
        return "готово"

    try:
        result = loop.run_until_complete(main())
        return result
    finally:
        # Очистка
        pending = asyncio.all_tasks(loop)
        for task in pending:
            task.cancel()
        loop.run_until_complete(asyncio.gather(*pending, return_exceptions=True))
        loop.close()

print(best_practice_example())
```

---

[prev: ./07-pitfalls.md](./07-pitfalls.md) | [next: ./09-debug-mode.md](./09-debug-mode.md)
