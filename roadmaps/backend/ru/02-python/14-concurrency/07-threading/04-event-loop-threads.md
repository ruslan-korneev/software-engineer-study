# Циклы событий в отдельных потоках

[prev: 03-locks-deadlocks.md](./03-locks-deadlocks.md) | [next: 05-cpu-bound-threads.md](./05-cpu-bound-threads.md)

---

## Зачем запускать event loop в отдельном потоке?

Иногда требуется запустить asyncio event loop в отдельном потоке:

| Сценарий | Причина |
|----------|---------|
| GUI-приложения | Основной поток занят UI-циклом |
| Интеграция с синхронным кодом | Legacy-система требует async функционал |
| Фоновые async-задачи | Основной поток не должен блокироваться |
| Тестирование async кода | Изоляция тестов |

## Базовый пример: event loop в потоке

```python
import asyncio
import threading
import time

def start_event_loop(loop: asyncio.AbstractEventLoop):
    """Запуск event loop в текущем потоке."""
    asyncio.set_event_loop(loop)
    loop.run_forever()

# Создаём новый event loop
loop = asyncio.new_event_loop()

# Запускаем его в отдельном потоке
loop_thread = threading.Thread(target=start_event_loop, args=(loop,), daemon=True)
loop_thread.start()

# Теперь можно отправлять задачи из основного потока
async def async_task(name: str) -> str:
    print(f"Async задача {name} начата")
    await asyncio.sleep(1)
    print(f"Async задача {name} завершена")
    return f"Результат {name}"

# Отправляем задачу в loop из другого потока
future = asyncio.run_coroutine_threadsafe(async_task("test"), loop)

# Получаем результат (блокирующий вызов)
result = future.result(timeout=5)
print(f"Получен результат: {result}")

# Останавливаем loop
loop.call_soon_threadsafe(loop.stop)
loop_thread.join()
```

## Класс-обёртка для async worker'а

```python
import asyncio
import threading
from typing import Coroutine, Any, Optional
from concurrent.futures import Future

class AsyncWorker:
    """Класс для управления event loop в отдельном потоке."""

    def __init__(self):
        self._loop: Optional[asyncio.AbstractEventLoop] = None
        self._thread: Optional[threading.Thread] = None

    def start(self) -> None:
        """Запуск worker'а."""
        if self._thread is not None:
            raise RuntimeError("Worker уже запущен")

        self._loop = asyncio.new_event_loop()
        self._thread = threading.Thread(target=self._run_loop, daemon=True)
        self._thread.start()

    def _run_loop(self) -> None:
        """Внутренний метод запуска loop."""
        asyncio.set_event_loop(self._loop)
        self._loop.run_forever()

    def submit(self, coro: Coroutine) -> Future:
        """Отправить корутину на выполнение."""
        if self._loop is None:
            raise RuntimeError("Worker не запущен")

        return asyncio.run_coroutine_threadsafe(coro, self._loop)

    def stop(self) -> None:
        """Остановка worker'а."""
        if self._loop is None:
            return

        self._loop.call_soon_threadsafe(self._loop.stop)
        if self._thread:
            self._thread.join(timeout=5)

        self._loop = None
        self._thread = None

    def __enter__(self):
        self.start()
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        self.stop()

# Использование
async def fetch_data(url: str) -> dict:
    await asyncio.sleep(1)  # Симуляция запроса
    return {"url": url, "status": "ok"}

with AsyncWorker() as worker:
    # Отправляем задачи
    future1 = worker.submit(fetch_data("https://api.example.com/users"))
    future2 = worker.submit(fetch_data("https://api.example.com/posts"))

    # Получаем результаты
    print(future1.result(timeout=5))
    print(future2.result(timeout=5))
```

## Взаимодействие GUI и asyncio

Пример интеграции с tkinter:

```python
import asyncio
import threading
import tkinter as tk
from tkinter import ttk

class AsyncGUI:
    """GUI-приложение с async backend'ом."""

    def __init__(self):
        self.root = tk.Tk()
        self.root.title("Async GUI Demo")

        # Создаём event loop для async операций
        self.async_loop = asyncio.new_event_loop()
        self.async_thread = threading.Thread(
            target=self._run_async_loop,
            daemon=True
        )

        self._setup_ui()

    def _setup_ui(self):
        """Настройка интерфейса."""
        self.status_label = ttk.Label(self.root, text="Готов")
        self.status_label.pack(pady=10)

        self.fetch_button = ttk.Button(
            self.root,
            text="Загрузить данные",
            command=self._on_fetch_click
        )
        self.fetch_button.pack(pady=10)

        self.result_text = tk.Text(self.root, height=10, width=40)
        self.result_text.pack(pady=10)

    def _run_async_loop(self):
        """Запуск async loop в отдельном потоке."""
        asyncio.set_event_loop(self.async_loop)
        self.async_loop.run_forever()

    def _on_fetch_click(self):
        """Обработчик клика по кнопке."""
        self.status_label.config(text="Загрузка...")
        self.fetch_button.config(state="disabled")

        # Отправляем async задачу
        future = asyncio.run_coroutine_threadsafe(
            self._fetch_data(),
            self.async_loop
        )

        # Проверяем завершение через after()
        self._check_future(future)

    async def _fetch_data(self) -> str:
        """Async операция загрузки данных."""
        await asyncio.sleep(2)  # Симуляция сетевого запроса
        return "Данные успешно загружены!\n" + "=" * 30 + "\nПример данных..."

    def _check_future(self, future):
        """Проверка завершения future."""
        if future.done():
            try:
                result = future.result()
                self.result_text.delete(1.0, tk.END)
                self.result_text.insert(tk.END, result)
                self.status_label.config(text="Готов")
            except Exception as e:
                self.status_label.config(text=f"Ошибка: {e}")
            finally:
                self.fetch_button.config(state="normal")
        else:
            # Проверяем снова через 100мс
            self.root.after(100, self._check_future, future)

    def run(self):
        """Запуск приложения."""
        self.async_thread.start()
        try:
            self.root.mainloop()
        finally:
            self.async_loop.call_soon_threadsafe(self.async_loop.stop)

# Запуск
if __name__ == "__main__":
    app = AsyncGUI()
    app.run()
```

## Безопасное взаимодействие с loop

### Проверка текущего потока

```python
import asyncio
import threading

async def safe_async_operation():
    loop = asyncio.get_running_loop()

    # Проверяем, что мы в правильном потоке
    if threading.current_thread() is not threading.main_thread():
        print("Выполняемся в отдельном потоке")

    await asyncio.sleep(1)
    return "done"

def run_from_another_thread(loop: asyncio.AbstractEventLoop):
    """Безопасный запуск async кода из другого потока."""

    # НЕПРАВИЛЬНО: прямой вызов
    # result = loop.run_until_complete(safe_async_operation())

    # ПРАВИЛЬНО: через run_coroutine_threadsafe
    future = asyncio.run_coroutine_threadsafe(safe_async_operation(), loop)
    return future.result(timeout=5)
```

### Отмена задач при остановке

```python
import asyncio
import threading
import signal

class GracefulAsyncRunner:
    """Runner с graceful shutdown."""

    def __init__(self):
        self._loop = asyncio.new_event_loop()
        self._thread = None
        self._tasks = []
        self._shutdown = threading.Event()

    def start(self):
        self._thread = threading.Thread(target=self._run, daemon=True)
        self._thread.start()

    def _run(self):
        asyncio.set_event_loop(self._loop)
        self._loop.run_forever()

    def submit(self, coro):
        """Отправить задачу с отслеживанием."""
        future = asyncio.run_coroutine_threadsafe(coro, self._loop)
        self._tasks.append(future)
        return future

    def shutdown(self, timeout: float = 5.0):
        """Graceful shutdown."""
        print("Начинаю graceful shutdown...")

        # Отменяем все pending задачи
        for task in self._tasks:
            if not task.done():
                self._loop.call_soon_threadsafe(task.cancel)

        # Даём время на отмену
        self._shutdown.wait(timeout=1)

        # Останавливаем loop
        self._loop.call_soon_threadsafe(self._loop.stop)
        self._thread.join(timeout=timeout)

        print("Shutdown завершён")

# Пример использования
async def long_running_task(name: str):
    try:
        for i in range(10):
            print(f"Задача {name}: шаг {i}")
            await asyncio.sleep(1)
    except asyncio.CancelledError:
        print(f"Задача {name}: отменена")
        raise

runner = GracefulAsyncRunner()
runner.start()

runner.submit(long_running_task("A"))
runner.submit(long_running_task("B"))

# Работаем некоторое время
import time
time.sleep(3)

# Graceful shutdown
runner.shutdown()
```

## Паттерн: Async Service

```python
import asyncio
import threading
from typing import Dict, Any
from dataclasses import dataclass
from queue import Queue
import uuid

@dataclass
class Request:
    id: str
    method: str
    params: Dict[str, Any]

@dataclass
class Response:
    id: str
    result: Any = None
    error: str = None

class AsyncService:
    """Сервис с async backend'ом."""

    def __init__(self):
        self._loop = None
        self._thread = None
        self._response_queues: Dict[str, Queue] = {}

    def start(self):
        self._loop = asyncio.new_event_loop()
        self._thread = threading.Thread(target=self._run, daemon=True)
        self._thread.start()

    def _run(self):
        asyncio.set_event_loop(self._loop)
        self._loop.run_forever()

    def call(self, method: str, **params) -> Any:
        """Синхронный вызов async метода."""
        request_id = str(uuid.uuid4())
        response_queue = Queue()
        self._response_queues[request_id] = response_queue

        # Отправляем запрос
        asyncio.run_coroutine_threadsafe(
            self._handle_request(Request(request_id, method, params)),
            self._loop
        )

        # Ожидаем ответ
        response = response_queue.get(timeout=30)

        del self._response_queues[request_id]

        if response.error:
            raise Exception(response.error)
        return response.result

    async def _handle_request(self, request: Request):
        """Обработка запроса."""
        try:
            # Диспетчеризация по методу
            handler = getattr(self, f"_method_{request.method}", None)
            if handler is None:
                raise ValueError(f"Unknown method: {request.method}")

            result = await handler(**request.params)
            response = Response(request.id, result=result)
        except Exception as e:
            response = Response(request.id, error=str(e))

        # Отправляем ответ
        queue = self._response_queues.get(request.id)
        if queue:
            queue.put(response)

    async def _method_fetch(self, url: str) -> dict:
        """Пример async метода."""
        await asyncio.sleep(1)  # Симуляция запроса
        return {"url": url, "data": "example"}

    async def _method_compute(self, x: int, y: int) -> int:
        """Пример async вычисления."""
        await asyncio.sleep(0.5)
        return x + y

    def stop(self):
        if self._loop:
            self._loop.call_soon_threadsafe(self._loop.stop)
        if self._thread:
            self._thread.join(timeout=5)

# Использование
service = AsyncService()
service.start()

try:
    result = service.call("fetch", url="https://api.example.com")
    print(f"Fetch result: {result}")

    result = service.call("compute", x=10, y=20)
    print(f"Compute result: {result}")
finally:
    service.stop()
```

## Множественные event loops

```python
import asyncio
import threading
from typing import List

class MultiLoopManager:
    """Управление несколькими event loops."""

    def __init__(self, num_loops: int = 3):
        self._loops: List[asyncio.AbstractEventLoop] = []
        self._threads: List[threading.Thread] = []
        self._current_index = 0
        self._lock = threading.Lock()

        for i in range(num_loops):
            loop = asyncio.new_event_loop()
            thread = threading.Thread(
                target=self._run_loop,
                args=(loop,),
                name=f"AsyncLoop-{i}",
                daemon=True
            )
            self._loops.append(loop)
            self._threads.append(thread)

    def _run_loop(self, loop: asyncio.AbstractEventLoop):
        asyncio.set_event_loop(loop)
        loop.run_forever()

    def start(self):
        """Запуск всех loops."""
        for thread in self._threads:
            thread.start()

    def get_loop(self) -> asyncio.AbstractEventLoop:
        """Получить loop (round-robin)."""
        with self._lock:
            loop = self._loops[self._current_index]
            self._current_index = (self._current_index + 1) % len(self._loops)
            return loop

    def submit(self, coro):
        """Отправить задачу в следующий loop."""
        loop = self.get_loop()
        return asyncio.run_coroutine_threadsafe(coro, loop)

    def stop(self):
        """Остановка всех loops."""
        for loop in self._loops:
            loop.call_soon_threadsafe(loop.stop)
        for thread in self._threads:
            thread.join(timeout=5)

# Использование
async def task(name: str, delay: float) -> str:
    await asyncio.sleep(delay)
    return f"Task {name} completed in {threading.current_thread().name}"

manager = MultiLoopManager(num_loops=3)
manager.start()

futures = []
for i in range(9):
    future = manager.submit(task(f"T{i}", 1))
    futures.append(future)

for future in futures:
    print(future.result(timeout=5))

manager.stop()
```

## Best Practices

1. **Всегда используйте `run_coroutine_threadsafe()`** для отправки задач из другого потока
2. **Используйте daemon-потоки** для loop — они завершатся при выходе из программы
3. **Реализуйте graceful shutdown** — отменяйте задачи перед остановкой
4. **Храните ссылку на loop** — нужна для взаимодействия
5. **Обрабатывайте исключения** в обоих потоках

```python
# Рекомендуемый паттерн
import asyncio
import threading

class AsyncHelper:
    """Минимальный helper для async в потоке."""

    _instance = None
    _lock = threading.Lock()

    def __new__(cls):
        with cls._lock:
            if cls._instance is None:
                cls._instance = super().__new__(cls)
                cls._instance._initialized = False
            return cls._instance

    def __init__(self):
        if self._initialized:
            return
        self._loop = asyncio.new_event_loop()
        self._thread = threading.Thread(
            target=self._loop.run_forever,
            daemon=True
        )
        self._thread.start()
        self._initialized = True

    def run(self, coro):
        """Синхронный запуск корутины."""
        future = asyncio.run_coroutine_threadsafe(coro, self._loop)
        return future.result()

# Singleton для удобства
async_helper = AsyncHelper()
result = async_helper.run(some_async_function())
```

## Распространённые ошибки

1. **Создание loop без `new_event_loop()`** — используйте `new_event_loop()` для отдельных потоков
2. **Забыть `set_event_loop()`** — loop не будет текущим в потоке
3. **Прямой вызов async функций из другого потока** — используйте `run_coroutine_threadsafe()`
4. **Не останавливать loop при выходе** — ресурсы могут не освободиться
5. **Блокирующие вызовы в async коде** — выносите их в `to_thread()`

---

[prev: 03-locks-deadlocks.md](./03-locks-deadlocks.md) | [next: 05-cpu-bound-threads.md](./05-cpu-bound-threads.md)
