# Эхо-сервер на asyncio

[prev: ./04-selectors.md](./04-selectors.md) | [next: ./06-graceful-shutdown.md](./06-graceful-shutdown.md)

---

## Введение

Эхо-сервер - это классический пример для изучения сетевого программирования. Он просто отправляет обратно всё, что получает. В этой теме мы построим полноценный эхо-сервер на `asyncio`, применив все изученные ранее концепции.

## Простейший эхо-сервер

Начнём с минимального примера:

```python
import asyncio

async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Обработчик клиентского соединения"""
    addr = writer.get_extra_info('peername')
    print(f"Подключён: {addr}")

    try:
        while True:
            data = await reader.read(1024)
            if not data:
                break

            message = data.decode().strip()
            print(f"Получено от {addr}: {message}")

            writer.write(f"Echo: {message}\n".encode())
            await writer.drain()

    except ConnectionResetError:
        print(f"Соединение сброшено: {addr}")
    finally:
        print(f"Отключён: {addr}")
        writer.close()
        await writer.wait_closed()

async def main():
    server = await asyncio.start_server(
        handle_client,
        '127.0.0.1',
        8080
    )

    addr = server.sockets[0].getsockname()
    print(f"Сервер запущен на {addr}")

    async with server:
        await server.serve_forever()

if __name__ == '__main__':
    asyncio.run(main())
```

## Понимание компонентов

### StreamReader и StreamWriter

`asyncio` предоставляет высокоуровневые абстракции для работы с потоками:

```python
# StreamReader - для чтения данных
data = await reader.read(n)      # Читать до n байт
data = await reader.readline()   # Читать до \n
data = await reader.readexactly(n)  # Читать ровно n байт
data = await reader.readuntil(separator)  # Читать до разделителя

# StreamWriter - для записи данных
writer.write(data)               # Записать в буфер (не async!)
await writer.drain()             # Дождаться отправки
writer.close()                   # Начать закрытие
await writer.wait_closed()       # Дождаться закрытия

# Информация о соединении
addr = writer.get_extra_info('peername')  # Адрес клиента
sock = writer.get_extra_info('socket')    # Низкоуровневый сокет
```

### asyncio.start_server

```python
server = await asyncio.start_server(
    client_connected_cb,  # Callback при подключении
    host='127.0.0.1',     # Хост
    port=8080,            # Порт
    reuse_address=True,   # SO_REUSEADDR
    reuse_port=False,     # SO_REUSEPORT
    backlog=100,          # Размер очереди
    start_serving=True    # Начать слушать сразу
)
```

## Эхо-сервер с протоколом сообщений

Реальные серверы используют протоколы. Реализуем простой протокол:
- Каждое сообщение заканчивается `\n`
- Команда `QUIT` отключает клиента
- Команда `TIME` возвращает текущее время

```python
import asyncio
from datetime import datetime

class EchoProtocol:
    """Простой протокол для эхо-сервера"""

    def __init__(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        self.reader = reader
        self.writer = writer
        self.addr = writer.get_extra_info('peername')

    async def handle(self):
        """Основной цикл обработки"""
        await self.send("Welcome to Echo Server!")
        await self.send("Commands: TIME, QUIT, or any text for echo")

        try:
            while True:
                message = await self.receive()
                if message is None:
                    break

                response = await self.process_command(message)
                if response is None:
                    break

                await self.send(response)

        except ConnectionResetError:
            print(f"[{self.addr}] Соединение сброшено")
        finally:
            await self.close()

    async def receive(self) -> str | None:
        """Получить сообщение от клиента"""
        try:
            data = await self.reader.readline()
            if not data:
                return None
            return data.decode().strip()
        except Exception:
            return None

    async def send(self, message: str):
        """Отправить сообщение клиенту"""
        self.writer.write(f"{message}\r\n".encode())
        await self.writer.drain()

    async def process_command(self, command: str) -> str | None:
        """Обработка команды"""
        cmd = command.upper()

        if cmd == 'QUIT':
            await self.send("Goodbye!")
            return None

        if cmd == 'TIME':
            return f"Server time: {datetime.now().isoformat()}"

        if cmd == 'HELP':
            return "Commands: TIME, QUIT, HELP"

        return f"Echo: {command}"

    async def close(self):
        """Закрыть соединение"""
        print(f"[{self.addr}] Отключён")
        self.writer.close()
        await self.writer.wait_closed()


async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Фабрика для создания протокола"""
    addr = writer.get_extra_info('peername')
    print(f"[{addr}] Подключён")

    protocol = EchoProtocol(reader, writer)
    await protocol.handle()


async def main():
    server = await asyncio.start_server(
        handle_client,
        '127.0.0.1',
        8080
    )

    print("Echo Server запущен на 127.0.0.1:8080")

    async with server:
        await server.serve_forever()


if __name__ == '__main__':
    asyncio.run(main())
```

## Эхо-сервер с низкоуровневым API

Для большего контроля можно использовать низкоуровневый API:

```python
import asyncio
import socket

class LowLevelEchoServer:
    """Эхо-сервер на низкоуровневом asyncio API"""

    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.server_socket: socket.socket | None = None
        self.clients: dict[socket.socket, tuple] = {}

    async def start(self):
        """Запуск сервера"""
        loop = asyncio.get_running_loop()

        # Создаём сокет вручную
        self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server_socket.bind((self.host, self.port))
        self.server_socket.listen(100)
        self.server_socket.setblocking(False)

        print(f"Low-level сервер на {self.host}:{self.port}")

        while True:
            # Ждём нового соединения
            client, addr = await loop.sock_accept(self.server_socket)
            self.clients[client] = addr
            print(f"[{addr}] Подключён")

            # Запускаем обработчик
            asyncio.create_task(self.handle_client(client, addr))

    async def handle_client(self, client: socket.socket, addr: tuple):
        """Обработка клиента"""
        loop = asyncio.get_running_loop()

        try:
            # Приветствие
            await loop.sock_sendall(client, b"Welcome!\r\n")

            while True:
                data = await loop.sock_recv(client, 1024)
                if not data:
                    break

                message = data.decode().strip()
                print(f"[{addr}] {message}")

                response = f"Echo: {message}\r\n".encode()
                await loop.sock_sendall(client, response)

        except ConnectionResetError:
            print(f"[{addr}] Соединение сброшено")
        finally:
            print(f"[{addr}] Отключён")
            del self.clients[client]
            client.close()

    def stop(self):
        """Остановка сервера"""
        for client in list(self.clients.keys()):
            client.close()
        if self.server_socket:
            self.server_socket.close()


async def main():
    server = LowLevelEchoServer('127.0.0.1', 8080)
    await server.start()


if __name__ == '__main__':
    asyncio.run(main())
```

## Чат-сервер на asyncio

Развитие эхо-сервера - чат с несколькими клиентами:

```python
import asyncio
from dataclasses import dataclass, field
from typing import Set

@dataclass
class ChatServer:
    """Простой чат-сервер"""
    host: str
    port: int
    clients: Set[asyncio.StreamWriter] = field(default_factory=set)
    names: dict = field(default_factory=dict)

    async def broadcast(self, message: str, exclude: asyncio.StreamWriter = None):
        """Отправить сообщение всем клиентам"""
        disconnected = []

        for client in self.clients:
            if client != exclude:
                try:
                    client.write(f"{message}\r\n".encode())
                    await client.drain()
                except:
                    disconnected.append(client)

        # Удаляем отключённых
        for client in disconnected:
            self.clients.discard(client)

    async def handle_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        """Обработка клиента"""
        addr = writer.get_extra_info('peername')

        # Запрашиваем имя
        writer.write(b"Enter your name: ")
        await writer.drain()

        name_data = await reader.readline()
        name = name_data.decode().strip() or f"User_{addr[1]}"

        self.clients.add(writer)
        self.names[writer] = name

        print(f"[{addr}] {name} подключился")
        await self.broadcast(f"*** {name} joined the chat ***")

        try:
            while True:
                data = await reader.readline()
                if not data:
                    break

                message = data.decode().strip()
                if message.lower() == '/quit':
                    break

                if message.lower() == '/users':
                    users = ', '.join(self.names.values())
                    writer.write(f"Online: {users}\r\n".encode())
                    await writer.drain()
                    continue

                print(f"[{name}] {message}")
                await self.broadcast(f"{name}: {message}", exclude=writer)

        except ConnectionResetError:
            pass
        finally:
            self.clients.discard(writer)
            if writer in self.names:
                name = self.names.pop(writer)
                await self.broadcast(f"*** {name} left the chat ***")
                print(f"[{addr}] {name} отключился")

            writer.close()
            await writer.wait_closed()

    async def start(self):
        """Запуск сервера"""
        server = await asyncio.start_server(
            self.handle_client,
            self.host,
            self.port
        )

        print(f"Chat Server на {self.host}:{self.port}")

        async with server:
            await server.serve_forever()


async def main():
    chat = ChatServer('127.0.0.1', 8080)
    await chat.start()


if __name__ == '__main__':
    asyncio.run(main())
```

## Эхо-сервер с таймаутами

Добавим таймауты для защиты от зависших соединений:

```python
import asyncio

IDLE_TIMEOUT = 60  # Секунд без активности

async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Обработчик с таймаутом"""
    addr = writer.get_extra_info('peername')
    print(f"[{addr}] Подключён")

    try:
        while True:
            try:
                # Ждём данные с таймаутом
                data = await asyncio.wait_for(
                    reader.readline(),
                    timeout=IDLE_TIMEOUT
                )
            except asyncio.TimeoutError:
                print(f"[{addr}] Таймаут неактивности")
                writer.write(b"Connection timeout. Goodbye!\r\n")
                await writer.drain()
                break

            if not data:
                break

            message = data.decode().strip()
            print(f"[{addr}] {message}")

            writer.write(f"Echo: {message}\r\n".encode())
            await writer.drain()

    except ConnectionResetError:
        print(f"[{addr}] Сброс соединения")
    finally:
        print(f"[{addr}] Отключён")
        writer.close()
        await writer.wait_closed()


async def main():
    server = await asyncio.start_server(
        handle_client,
        '127.0.0.1',
        8080
    )

    print(f"Echo Server с таймаутом {IDLE_TIMEOUT}с")

    async with server:
        await server.serve_forever()


if __name__ == '__main__':
    asyncio.run(main())
```

## Эхо-сервер с ограничением соединений

```python
import asyncio

MAX_CONNECTIONS = 10
connection_semaphore = asyncio.Semaphore(MAX_CONNECTIONS)
active_connections = 0

async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Обработчик с ограничением соединений"""
    global active_connections

    addr = writer.get_extra_info('peername')

    # Пытаемся занять слот
    acquired = connection_semaphore.locked()
    if not await asyncio.wait_for(connection_semaphore.acquire(), timeout=0.1):
        writer.write(b"Server is busy. Try again later.\r\n")
        await writer.drain()
        writer.close()
        await writer.wait_closed()
        return

    active_connections += 1
    print(f"[{addr}] Подключён ({active_connections}/{MAX_CONNECTIONS})")

    try:
        while True:
            data = await reader.readline()
            if not data:
                break

            message = data.decode().strip()
            writer.write(f"Echo: {message}\r\n".encode())
            await writer.drain()

    except ConnectionResetError:
        pass
    finally:
        active_connections -= 1
        connection_semaphore.release()
        print(f"[{addr}] Отключён ({active_connections}/{MAX_CONNECTIONS})")
        writer.close()
        await writer.wait_closed()


async def main():
    server = await asyncio.start_server(
        handle_client,
        '127.0.0.1',
        8080
    )

    print(f"Echo Server (макс. {MAX_CONNECTIONS} соединений)")

    async with server:
        await server.serve_forever()


if __name__ == '__main__':
    asyncio.run(main())
```

## Тестирование эхо-сервера

### Простой клиент для тестирования

```python
import asyncio

async def echo_client(message: str):
    """Простой эхо-клиент"""
    reader, writer = await asyncio.open_connection('127.0.0.1', 8080)

    print(f"Отправляем: {message}")
    writer.write(f"{message}\n".encode())
    await writer.drain()

    response = await reader.readline()
    print(f"Получили: {response.decode().strip()}")

    writer.close()
    await writer.wait_closed()


async def main():
    await echo_client("Hello, Server!")
    await echo_client("How are you?")


if __name__ == '__main__':
    asyncio.run(main())
```

### Нагрузочное тестирование

```python
import asyncio
import time

async def client_session(client_id: int, messages: int):
    """Сессия одного клиента"""
    try:
        reader, writer = await asyncio.open_connection('127.0.0.1', 8080)

        for i in range(messages):
            msg = f"Client {client_id}, message {i}\n"
            writer.write(msg.encode())
            await writer.drain()
            await reader.readline()

        writer.close()
        await writer.wait_closed()
        return True
    except Exception as e:
        print(f"Client {client_id} error: {e}")
        return False


async def load_test(num_clients: int, messages_per_client: int):
    """Нагрузочный тест"""
    print(f"Запуск {num_clients} клиентов, {messages_per_client} сообщений каждый")

    start = time.time()

    tasks = [
        client_session(i, messages_per_client)
        for i in range(num_clients)
    ]

    results = await asyncio.gather(*tasks)
    elapsed = time.time() - start

    successful = sum(results)
    total_messages = successful * messages_per_client

    print(f"Успешных клиентов: {successful}/{num_clients}")
    print(f"Всего сообщений: {total_messages}")
    print(f"Время: {elapsed:.2f}с")
    print(f"Сообщений/сек: {total_messages/elapsed:.0f}")


if __name__ == '__main__':
    asyncio.run(load_test(100, 100))
```

## Лучшие практики

1. **Используйте async with для сервера** - гарантирует корректное закрытие
2. **Обрабатывайте все исключения** - сетевые ошибки неизбежны
3. **Используйте drain()** - для контроля отправки
4. **Добавляйте таймауты** - защита от зависших соединений
5. **Ограничивайте соединения** - защита от перегрузки
6. **Логируйте подключения** - для отладки и мониторинга

```python
# Паттерн обработчика
async def handle_client(reader, writer):
    addr = writer.get_extra_info('peername')
    try:
        # Основная логика
        pass
    except ConnectionResetError:
        pass  # Клиент отключился
    except Exception as e:
        print(f"Error: {e}")
    finally:
        writer.close()
        await writer.wait_closed()
```

## Заключение

Эхо-сервер на `asyncio` демонстрирует мощь асинхронного программирования. Один поток может обслуживать тысячи соединений без накладных расходов на потоки. Мы изучили:

- Высокоуровневый API (`start_server`, `StreamReader/Writer`)
- Низкоуровневый API (работа с сокетами напрямую)
- Протоколы и команды
- Таймауты и ограничения
- Тестирование

В следующей теме мы рассмотрим корректное завершение работы сервера (graceful shutdown).

---

[prev: ./04-selectors.md](./04-selectors.md) | [next: ./06-graceful-shutdown.md](./06-graceful-shutdown.md)
