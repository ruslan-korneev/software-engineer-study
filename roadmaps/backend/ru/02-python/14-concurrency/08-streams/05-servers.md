# Создание серверов

[prev: 04-non-blocking-stdin.md](./04-non-blocking-stdin.md) | [next: 06-chat-server.md](./06-chat-server.md)

---

## Введение в создание серверов

Asyncio предоставляет мощные инструменты для создания высокопроизводительных сетевых серверов. С помощью Streams API можно быстро создать сервер, который обрабатывает тысячи соединений одновременно.

## Базовый TCP-сервер

### Минимальный пример

```python
import asyncio

async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Обработчик клиентского соединения."""
    addr = writer.get_extra_info('peername')
    print(f'Клиент подключен: {addr}')

    try:
        while True:
            data = await reader.read(1024)
            if not data:
                break

            message = data.decode()
            print(f'Получено от {addr}: {message!r}')

            # Отправляем ответ
            response = f'Эхо: {message}'
            writer.write(response.encode())
            await writer.drain()

    except ConnectionResetError:
        print(f'Соединение с {addr} сброшено')
    finally:
        writer.close()
        await writer.wait_closed()
        print(f'Клиент отключен: {addr}')


async def main():
    server = await asyncio.start_server(
        handle_client,
        '127.0.0.1',
        8888
    )

    addr = server.sockets[0].getsockname()
    print(f'Сервер запущен на {addr}')

    async with server:
        await server.serve_forever()


asyncio.run(main())
```

## Объект Server

`asyncio.start_server()` возвращает объект `Server`, который предоставляет методы для управления сервером:

```python
import asyncio

async def handle_client(reader, writer):
    pass

async def main():
    server = await asyncio.start_server(handle_client, '0.0.0.0', 8888)

    # Получаем адреса, на которых слушает сервер
    for sock in server.sockets:
        print(f'Слушаем на {sock.getsockname()}')

    # Проверяем, работает ли сервер
    print(f'Сервер работает: {server.is_serving()}')

    # Запускаем сервер (если start_serving=False)
    await server.start_serving()

    # Ждем завершения (блокирующий)
    # await server.serve_forever()

    # Или работаем определенное время
    await asyncio.sleep(60)

    # Останавливаем прием новых соединений
    server.close()

    # Ждем закрытия сервера
    await server.wait_closed()

    print('Сервер остановлен')
```

## Параметры start_server()

```python
import asyncio
import ssl

async def handle_client(reader, writer):
    pass

async def create_server():
    server = await asyncio.start_server(
        handle_client,           # Callback для обработки соединений
        host='0.0.0.0',          # Хост (или список хостов)
        port=8888,               # Порт
        family=0,                # Семейство адресов (0 = auto)
        flags=0,                 # Флаги сокета
        sock=None,               # Существующий сокет
        backlog=100,             # Размер очереди ожидающих соединений
        ssl=None,                # SSL контекст для HTTPS/TLS
        reuse_address=True,      # Переиспользование адреса (SO_REUSEADDR)
        reuse_port=True,         # Переиспользование порта (SO_REUSEPORT)
        ssl_handshake_timeout=None,  # Таймаут SSL рукопожатия
        start_serving=True       # Начать слушать сразу
    )
    return server
```

## SSL/TLS сервер

```python
import asyncio
import ssl

async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    data = await reader.read(1024)
    writer.write(b'Hello over TLS!')
    await writer.drain()
    writer.close()
    await writer.wait_closed()


async def main():
    # Создаем SSL контекст
    ssl_context = ssl.create_default_context(ssl.Purpose.CLIENT_AUTH)
    ssl_context.load_cert_chain('server.crt', 'server.key')

    server = await asyncio.start_server(
        handle_client,
        '0.0.0.0',
        8443,
        ssl=ssl_context
    )

    print('SSL сервер запущен на порту 8443')

    async with server:
        await server.serve_forever()


asyncio.run(main())
```

## Сервер с таймаутами

```python
import asyncio

class TimeoutHandler:
    """Обработчик с таймаутами на операции."""

    def __init__(self, read_timeout: float = 30.0, write_timeout: float = 10.0):
        self.read_timeout = read_timeout
        self.write_timeout = write_timeout

    async def handle(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        addr = writer.get_extra_info('peername')
        print(f'Клиент подключен: {addr}')

        try:
            while True:
                # Чтение с таймаутом
                try:
                    data = await asyncio.wait_for(
                        reader.readline(),
                        timeout=self.read_timeout
                    )
                except asyncio.TimeoutError:
                    print(f'Таймаут чтения для {addr}')
                    writer.write(b'ERROR: Read timeout\n')
                    break

                if not data:
                    break

                # Обработка запроса
                response = self.process_request(data)

                # Запись с таймаутом
                try:
                    writer.write(response)
                    await asyncio.wait_for(
                        writer.drain(),
                        timeout=self.write_timeout
                    )
                except asyncio.TimeoutError:
                    print(f'Таймаут записи для {addr}')
                    break

        except ConnectionResetError:
            pass
        finally:
            writer.close()
            await writer.wait_closed()
            print(f'Клиент отключен: {addr}')

    def process_request(self, data: bytes) -> bytes:
        """Обрабатывает запрос и возвращает ответ."""
        return f'Echo: {data.decode()}'.encode()


async def main():
    handler = TimeoutHandler(read_timeout=30.0, write_timeout=10.0)

    server = await asyncio.start_server(
        handler.handle,
        '127.0.0.1',
        8888
    )

    async with server:
        await server.serve_forever()


asyncio.run(main())
```

## Сервер с ограничением соединений

```python
import asyncio

class LimitedConnectionsServer:
    """Сервер с ограничением количества одновременных соединений."""

    def __init__(self, max_connections: int = 100):
        self.semaphore = asyncio.Semaphore(max_connections)
        self.active_connections = 0

    async def handle_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        addr = writer.get_extra_info('peername')

        async with self.semaphore:
            self.active_connections += 1
            print(f'Подключен {addr}. Активных соединений: {self.active_connections}')

            try:
                await self._process_client(reader, writer)
            finally:
                self.active_connections -= 1
                writer.close()
                await writer.wait_closed()
                print(f'Отключен {addr}. Активных соединений: {self.active_connections}')

    async def _process_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        """Обрабатывает клиента."""
        while True:
            data = await reader.readline()
            if not data:
                break

            writer.write(f'Echo: {data.decode()}'.encode())
            await writer.drain()


async def main():
    limited_server = LimitedConnectionsServer(max_connections=10)

    server = await asyncio.start_server(
        limited_server.handle_client,
        '127.0.0.1',
        8888
    )

    print('Сервер с лимитом 10 соединений запущен')

    async with server:
        await server.serve_forever()


asyncio.run(main())
```

## Graceful Shutdown

```python
import asyncio
import signal

class GracefulServer:
    """Сервер с корректным завершением работы."""

    def __init__(self):
        self.server = None
        self.active_tasks: set[asyncio.Task] = set()
        self.shutdown_event = asyncio.Event()

    async def handle_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        task = asyncio.current_task()
        self.active_tasks.add(task)

        addr = writer.get_extra_info('peername')
        print(f'Клиент подключен: {addr}')

        try:
            while not self.shutdown_event.is_set():
                try:
                    data = await asyncio.wait_for(reader.readline(), timeout=1.0)
                except asyncio.TimeoutError:
                    continue

                if not data:
                    break

                writer.write(f'Echo: {data.decode()}'.encode())
                await writer.drain()

        except asyncio.CancelledError:
            # Отправляем прощальное сообщение при завершении
            try:
                writer.write(b'Server shutting down. Goodbye!\n')
                await writer.drain()
            except:
                pass
        finally:
            writer.close()
            await writer.wait_closed()
            self.active_tasks.discard(task)
            print(f'Клиент отключен: {addr}')

    async def start(self, host: str, port: int):
        self.server = await asyncio.start_server(
            self.handle_client,
            host,
            port
        )
        print(f'Сервер запущен на {host}:{port}')

    async def shutdown(self):
        """Корректно завершает работу сервера."""
        print('Начинаем завершение работы...')

        # Сигнализируем о завершении
        self.shutdown_event.set()

        # Прекращаем принимать новые соединения
        self.server.close()
        await self.server.wait_closed()

        # Даем время активным соединениям завершиться
        if self.active_tasks:
            print(f'Ожидание завершения {len(self.active_tasks)} соединений...')

            # Ждем завершения или отменяем через 5 секунд
            done, pending = await asyncio.wait(
                self.active_tasks,
                timeout=5.0
            )

            # Отменяем оставшиеся
            for task in pending:
                task.cancel()

            if pending:
                await asyncio.gather(*pending, return_exceptions=True)

        print('Сервер остановлен')

    async def run_forever(self):
        """Запускает сервер до получения сигнала завершения."""
        loop = asyncio.get_running_loop()

        # Устанавливаем обработчики сигналов
        for sig in (signal.SIGTERM, signal.SIGINT):
            loop.add_signal_handler(
                sig,
                lambda: asyncio.create_task(self.shutdown())
            )

        async with self.server:
            await self.server.serve_forever()


async def main():
    server = GracefulServer()
    await server.start('127.0.0.1', 8888)
    await server.run_forever()


asyncio.run(main())
```

## HTTP-подобный сервер

```python
import asyncio
from typing import Dict, Callable, Awaitable

class SimpleHTTPServer:
    """Простой HTTP-подобный сервер."""

    def __init__(self):
        self.routes: Dict[str, Callable[..., Awaitable[str]]] = {}

    def route(self, path: str):
        """Декоратор для регистрации маршрутов."""
        def decorator(func):
            self.routes[path] = func
            return func
        return decorator

    async def handle_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        addr = writer.get_extra_info('peername')

        try:
            # Читаем первую строку запроса
            request_line = await reader.readline()
            if not request_line:
                return

            # Парсим запрос (METHOD /path HTTP/1.x)
            parts = request_line.decode().strip().split()
            if len(parts) < 2:
                await self._send_error(writer, 400, 'Bad Request')
                return

            method, path = parts[0], parts[1]

            # Читаем заголовки
            headers = {}
            while True:
                line = await reader.readline()
                if line == b'\r\n' or line == b'\n':
                    break
                if b':' in line:
                    key, value = line.decode().split(':', 1)
                    headers[key.strip().lower()] = value.strip()

            print(f'{addr} - {method} {path}')

            # Находим обработчик
            handler = self.routes.get(path)
            if handler:
                body = await handler(method, headers)
                await self._send_response(writer, 200, body)
            else:
                await self._send_error(writer, 404, 'Not Found')

        except Exception as e:
            print(f'Ошибка обработки {addr}: {e}')
            await self._send_error(writer, 500, 'Internal Server Error')
        finally:
            writer.close()
            await writer.wait_closed()

    async def _send_response(self, writer: asyncio.StreamWriter, status: int, body: str):
        response = (
            f'HTTP/1.1 {status} OK\r\n'
            f'Content-Type: text/plain\r\n'
            f'Content-Length: {len(body)}\r\n'
            f'Connection: close\r\n'
            f'\r\n'
            f'{body}'
        )
        writer.write(response.encode())
        await writer.drain()

    async def _send_error(self, writer: asyncio.StreamWriter, status: int, message: str):
        response = (
            f'HTTP/1.1 {status} {message}\r\n'
            f'Content-Type: text/plain\r\n'
            f'Content-Length: {len(message)}\r\n'
            f'Connection: close\r\n'
            f'\r\n'
            f'{message}'
        )
        writer.write(response.encode())
        await writer.drain()

    async def run(self, host: str = '127.0.0.1', port: int = 8080):
        server = await asyncio.start_server(
            self.handle_client,
            host,
            port
        )

        print(f'HTTP сервер запущен на http://{host}:{port}')

        async with server:
            await server.serve_forever()


# Использование
app = SimpleHTTPServer()

@app.route('/')
async def index(method, headers):
    return 'Hello, World!'

@app.route('/status')
async def status(method, headers):
    return 'OK'

@app.route('/time')
async def time_handler(method, headers):
    import datetime
    return str(datetime.datetime.now())

# asyncio.run(app.run())
```

## Многопортовый сервер

```python
import asyncio

async def handle_http(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Обработчик HTTP на порту 80."""
    writer.write(b'HTTP/1.1 200 OK\r\nContent-Length: 5\r\n\r\nHTTP!')
    await writer.drain()
    writer.close()
    await writer.wait_closed()


async def handle_https(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Обработчик HTTPS на порту 443."""
    writer.write(b'HTTP/1.1 200 OK\r\nContent-Length: 6\r\n\r\nHTTPS!')
    await writer.drain()
    writer.close()
    await writer.wait_closed()


async def main():
    # Создаем несколько серверов на разных портах
    servers = await asyncio.gather(
        asyncio.start_server(handle_http, '0.0.0.0', 8080),
        asyncio.start_server(handle_https, '0.0.0.0', 8443),
    )

    print('Серверы запущены на портах 8080 и 8443')

    # Ждем завершения всех серверов
    try:
        await asyncio.gather(
            *[server.serve_forever() for server in servers]
        )
    finally:
        for server in servers:
            server.close()


asyncio.run(main())
```

## Best Practices для серверов

### 1. Всегда обрабатывайте исключения

```python
async def safe_handle(reader, writer):
    try:
        await actual_handle(reader, writer)
    except ConnectionResetError:
        pass  # Клиент отключился
    except Exception as e:
        print(f'Ошибка: {e}')
    finally:
        writer.close()
        await writer.wait_closed()
```

### 2. Используйте таймауты

```python
try:
    data = await asyncio.wait_for(reader.read(1024), timeout=30.0)
except asyncio.TimeoutError:
    # Клиент слишком долго не отвечает
    pass
```

### 3. Ограничивайте количество соединений

```python
semaphore = asyncio.Semaphore(1000)

async def handle(reader, writer):
    async with semaphore:
        await process(reader, writer)
```

### 4. Логируйте важные события

```python
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

async def handle(reader, writer):
    addr = writer.get_extra_info('peername')
    logger.info(f'Подключение: {addr}')
    # ...
    logger.info(f'Отключение: {addr}')
```

### 5. Реализуйте graceful shutdown

```python
# Не прерывайте активные соединения резко
# Дайте им время завершить работу
await asyncio.wait(active_tasks, timeout=5.0)
```

---

[prev: 04-non-blocking-stdin.md](./04-non-blocking-stdin.md) | [next: 06-chat-server.md](./06-chat-server.md)
