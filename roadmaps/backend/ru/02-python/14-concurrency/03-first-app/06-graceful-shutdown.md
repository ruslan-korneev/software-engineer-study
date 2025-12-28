# Корректная остановка

[prev: ./05-echo-server.md](./05-echo-server.md) | [next: ../04-web-requests/readme.md](../04-web-requests/readme.md)

---

## Введение

Graceful shutdown (корректное завершение работы) - это процесс остановки сервера, при котором:
1. Новые соединения не принимаются
2. Текущие запросы завершаются
3. Ресурсы освобождаются корректно
4. Данные не теряются

## Почему это важно?

При резком завершении (kill -9, SIGKILL):
- Запросы обрываются на середине
- Данные могут быть потеряны
- Соединения с БД остаются открытыми
- Клиенты получают ошибки

При корректном завершении:
- Клиенты получают ответы
- Транзакции завершаются
- Соединения закрываются
- Логи записываются

## Сигналы Unix

Основные сигналы для управления процессами:

| Сигнал | Номер | Описание | Действие по умолчанию |
|--------|-------|----------|----------------------|
| SIGTERM | 15 | Завершение | Завершить |
| SIGINT | 2 | Прерывание (Ctrl+C) | Завершить |
| SIGKILL | 9 | Принудительное завершение | Нельзя перехватить |
| SIGHUP | 1 | Отключение терминала | Завершить |
| SIGUSR1 | 10 | Пользовательский 1 | Игнорировать |
| SIGUSR2 | 12 | Пользовательский 2 | Игнорировать |

## Обработка сигналов в asyncio

### Базовый пример

```python
import asyncio
import signal

async def main():
    # Создаём событие для завершения
    shutdown_event = asyncio.Event()

    # Обработчик сигналов
    def signal_handler():
        print("Получен сигнал завершения")
        shutdown_event.set()

    # Регистрируем обработчики
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, signal_handler)

    print("Сервер запущен. Нажмите Ctrl+C для остановки")

    # Ждём сигнала
    await shutdown_event.wait()

    print("Корректное завершение...")


if __name__ == '__main__':
    asyncio.run(main())
```

### Важно для Windows

Windows не поддерживает сигналы Unix. Используйте альтернативный подход:

```python
import asyncio
import sys

async def main():
    shutdown_event = asyncio.Event()

    if sys.platform != 'win32':
        import signal
        loop = asyncio.get_running_loop()
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(sig, shutdown_event.set)
    else:
        # Windows: обрабатываем Ctrl+C через try/except
        pass

    try:
        await shutdown_event.wait()
    except KeyboardInterrupt:
        pass

    print("Завершение...")
```

## Graceful Shutdown для сервера

### Полный пример

```python
import asyncio
import signal
from typing import Set

class GracefulServer:
    """Сервер с корректным завершением"""

    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.server: asyncio.Server | None = None
        self.shutdown_event = asyncio.Event()
        self.active_connections: Set[asyncio.StreamWriter] = set()

    async def start(self):
        """Запуск сервера"""
        # Регистрируем обработчики сигналов
        loop = asyncio.get_running_loop()
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(sig, self._signal_handler)

        # Запускаем сервер
        self.server = await asyncio.start_server(
            self._handle_client,
            self.host,
            self.port
        )

        print(f"Сервер запущен на {self.host}:{self.port}")
        print("Нажмите Ctrl+C для остановки")

        try:
            # Запускаем сервер и ждём сигнала
            async with self.server:
                await self.shutdown_event.wait()
        finally:
            await self._shutdown()

    def _signal_handler(self):
        """Обработчик сигналов"""
        print("\nПолучен сигнал завершения")
        self.shutdown_event.set()

    async def _handle_client(self, reader: asyncio.StreamReader,
                             writer: asyncio.StreamWriter):
        """Обработка клиента"""
        addr = writer.get_extra_info('peername')
        self.active_connections.add(writer)
        print(f"[{addr}] Подключён (активных: {len(self.active_connections)})")

        try:
            while not self.shutdown_event.is_set():
                try:
                    data = await asyncio.wait_for(reader.readline(), timeout=1.0)
                except asyncio.TimeoutError:
                    continue

                if not data:
                    break

                message = data.decode().strip()
                writer.write(f"Echo: {message}\r\n".encode())
                await writer.drain()

        except ConnectionResetError:
            pass
        finally:
            self.active_connections.discard(writer)
            print(f"[{addr}] Отключён (активных: {len(self.active_connections)})")
            writer.close()
            await writer.wait_closed()

    async def _shutdown(self):
        """Корректное завершение"""
        print("Начинаем корректное завершение...")

        # 1. Прекращаем принимать новые соединения
        if self.server:
            self.server.close()
            await self.server.wait_closed()
            print("Новые соединения отклонены")

        # 2. Уведомляем активные соединения
        for writer in list(self.active_connections):
            try:
                writer.write(b"Server is shutting down. Goodbye!\r\n")
                await writer.drain()
            except:
                pass

        # 3. Ждём завершения активных соединений (с таймаутом)
        if self.active_connections:
            print(f"Ожидание {len(self.active_connections)} соединений...")

            # Даём время на завершение
            for _ in range(30):  # 30 секунд максимум
                if not self.active_connections:
                    break
                await asyncio.sleep(1)
            else:
                print(f"Принудительное закрытие {len(self.active_connections)} соединений")
                for writer in list(self.active_connections):
                    writer.close()

        print("Сервер остановлен")


async def main():
    server = GracefulServer('127.0.0.1', 8080)
    await server.start()


if __name__ == '__main__':
    asyncio.run(main())
```

## Управление задачами при завершении

### Отмена всех задач

```python
import asyncio

async def shutdown(loop):
    """Отмена всех задач при завершении"""
    print("Отмена задач...")

    # Получаем все задачи кроме текущей
    tasks = [t for t in asyncio.all_tasks() if t is not asyncio.current_task()]

    print(f"Задач для отмены: {len(tasks)}")

    # Отменяем все задачи
    for task in tasks:
        task.cancel()

    # Ждём завершения
    await asyncio.gather(*tasks, return_exceptions=True)

    print("Все задачи отменены")
```

### Graceful отмена с таймаутом

```python
import asyncio

async def graceful_shutdown(timeout: float = 30.0):
    """Корректная отмена задач с таймаутом"""
    print(f"Graceful shutdown (таймаут: {timeout}с)")

    # Получаем все задачи
    tasks = [t for t in asyncio.all_tasks() if t is not asyncio.current_task()]

    if not tasks:
        print("Нет активных задач")
        return

    print(f"Ожидание {len(tasks)} задач...")

    # Ждём завершения с таймаутом
    done, pending = await asyncio.wait(tasks, timeout=timeout)

    print(f"Завершено: {len(done)}, Ожидают: {len(pending)}")

    # Отменяем незавершённые
    for task in pending:
        task.cancel()

    # Ждём отмены
    if pending:
        await asyncio.gather(*pending, return_exceptions=True)
        print(f"Отменено: {len(pending)} задач")
```

## Паттерн с TaskGroup (Python 3.11+)

```python
import asyncio
import signal

async def worker(name: str, shutdown_event: asyncio.Event):
    """Воркер, который реагирует на shutdown"""
    print(f"[{name}] Запущен")
    try:
        while not shutdown_event.is_set():
            # Делаем работу
            await asyncio.sleep(1)
            print(f"[{name}] Работаю...")
    except asyncio.CancelledError:
        print(f"[{name}] Отменён")
        raise
    finally:
        print(f"[{name}] Завершён")


async def main():
    shutdown_event = asyncio.Event()

    # Обработчик сигналов
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, shutdown_event.set)

    try:
        async with asyncio.TaskGroup() as tg:
            # Запускаем воркеры
            tg.create_task(worker("Worker-1", shutdown_event))
            tg.create_task(worker("Worker-2", shutdown_event))
            tg.create_task(worker("Worker-3", shutdown_event))

            # Ждём сигнала
            await shutdown_event.wait()

            # Отменяем все задачи в группе
            raise asyncio.CancelledError()

    except* asyncio.CancelledError:
        print("Все воркеры остановлены")

    print("Программа завершена")


if __name__ == '__main__':
    asyncio.run(main())
```

## Graceful Shutdown с базой данных

```python
import asyncio
import signal
from contextlib import asynccontextmanager

class DatabasePool:
    """Имитация пула соединений с БД"""

    def __init__(self):
        self.connections = []
        self.is_closed = False

    async def connect(self):
        print("Подключение к БД...")
        await asyncio.sleep(0.5)  # Имитация
        print("Подключено")

    async def close(self):
        print("Закрытие соединений с БД...")
        self.is_closed = True
        await asyncio.sleep(0.5)  # Имитация
        print("Соединения закрыты")

    async def execute(self, query: str):
        if self.is_closed:
            raise RuntimeError("Пул закрыт")
        await asyncio.sleep(0.1)  # Имитация
        return f"Result: {query}"


class Application:
    """Приложение с graceful shutdown"""

    def __init__(self):
        self.db = DatabasePool()
        self.shutdown_event = asyncio.Event()

    @asynccontextmanager
    async def lifespan(self):
        """Контекстный менеджер жизненного цикла"""
        # Startup
        await self.db.connect()
        yield
        # Shutdown
        await self.db.close()

    async def handle_request(self, request_id: int):
        """Обработка запроса"""
        print(f"[Request-{request_id}] Начало")
        try:
            result = await self.db.execute(f"SELECT * FROM users WHERE id={request_id}")
            print(f"[Request-{request_id}] Завершён: {result}")
        except Exception as e:
            print(f"[Request-{request_id}] Ошибка: {e}")

    async def run(self):
        """Запуск приложения"""
        # Регистрируем сигналы
        loop = asyncio.get_running_loop()
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(sig, self._signal_handler)

        async with self.lifespan():
            print("Приложение запущено")

            # Симулируем запросы
            request_id = 0
            while not self.shutdown_event.is_set():
                request_id += 1
                asyncio.create_task(self.handle_request(request_id))
                await asyncio.sleep(2)

            print("Ожидание завершения запросов...")
            # Даём время на завершение
            await asyncio.sleep(1)

        print("Приложение остановлено")

    def _signal_handler(self):
        print("\nПолучен сигнал завершения")
        self.shutdown_event.set()


async def main():
    app = Application()
    await app.run()


if __name__ == '__main__':
    asyncio.run(main())
```

## Шаблон для веб-серверов

```python
import asyncio
import signal
from aiohttp import web

async def handle(request):
    """Обработчик запросов"""
    await asyncio.sleep(1)  # Имитация работы
    return web.Response(text="Hello, World!")


async def on_startup(app):
    """Действия при запуске"""
    print("Инициализация...")
    # Подключение к БД, кэшу и т.д.


async def on_cleanup(app):
    """Действия при остановке"""
    print("Очистка ресурсов...")
    # Закрытие соединений


async def on_shutdown(app):
    """Действия при начале остановки"""
    print("Начало остановки...")
    # Уведомление сервисов


def create_app():
    """Создание приложения"""
    app = web.Application()

    # Регистрируем хуки жизненного цикла
    app.on_startup.append(on_startup)
    app.on_shutdown.append(on_shutdown)
    app.on_cleanup.append(on_cleanup)

    # Маршруты
    app.router.add_get('/', handle)

    return app


if __name__ == '__main__':
    app = create_app()
    web.run_app(
        app,
        host='127.0.0.1',
        port=8080,
        shutdown_timeout=30.0  # Таймаут для graceful shutdown
    )
```

## Контекстный менеджер для приложения

```python
import asyncio
import signal
from contextlib import asynccontextmanager

@asynccontextmanager
async def application_lifecycle():
    """Управление жизненным циклом приложения"""
    shutdown_event = asyncio.Event()

    # Регистрируем сигналы
    loop = asyncio.get_running_loop()
    for sig in (signal.SIGTERM, signal.SIGINT):
        loop.add_signal_handler(sig, shutdown_event.set)

    # Startup
    print("=== Startup ===")
    resources = await startup()

    try:
        yield shutdown_event, resources
    finally:
        # Shutdown
        print("=== Shutdown ===")
        await shutdown(resources)


async def startup():
    """Инициализация ресурсов"""
    print("Инициализация...")
    return {"db": "connected", "cache": "connected"}


async def shutdown(resources):
    """Освобождение ресурсов"""
    print(f"Освобождение: {resources}")


async def main():
    async with application_lifecycle() as (shutdown_event, resources):
        print(f"Приложение запущено с ресурсами: {resources}")
        await shutdown_event.wait()


if __name__ == '__main__':
    asyncio.run(main())
```

## Лучшие практики

1. **Обрабатывайте SIGTERM и SIGINT** - стандартные сигналы для остановки
2. **Устанавливайте таймауты** - не ждите вечно
3. **Логируйте этапы** - для отладки проблем
4. **Уведомляйте клиентов** - о предстоящем завершении
5. **Закрывайте ресурсы в правильном порядке** - сначала соединения, потом пулы

```python
# Порядок закрытия
async def shutdown():
    # 1. Прекратить принимать новые запросы
    server.close()

    # 2. Дождаться завершения активных запросов
    await wait_for_requests(timeout=30)

    # 3. Закрыть соединения с внешними сервисами
    await close_external_connections()

    # 4. Закрыть пулы соединений с БД
    await db_pool.close()

    # 5. Сохранить состояние если нужно
    await save_state()
```

## Типичные ошибки

1. **Не обрабатывать сигналы** - резкое завершение
2. **Бесконечное ожидание** - процесс не завершается
3. **Закрывать ресурсы в неправильном порядке** - ошибки
4. **Не отменять задачи** - утечка ресурсов
5. **Игнорировать Windows** - код не работает на Windows

## Заключение

Graceful shutdown - важная часть production-ready приложения. Правильная реализация:

- Предотвращает потерю данных
- Обеспечивает хороший UX для клиентов
- Упрощает деплой и обновления
- Корректно освобождает ресурсы

Это завершает раздел "Первое приложение asyncio". Вы изучили путь от блокирующих сокетов до полноценного асинхронного сервера с корректным завершением работы.

---

[prev: ./05-echo-server.md](./05-echo-server.md) | [next: ../04-web-requests/readme.md](../04-web-requests/readme.md)
