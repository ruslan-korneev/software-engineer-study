# Создание чат-сервера

[prev: 05-servers.md](./05-servers.md) | [next: ../09-web-apps/readme.md](../09-web-apps/readme.md)

---

## Введение

Чат-сервер - это классический пример применения asyncio streams. Он демонстрирует:
- Управление множеством соединений
- Broadcast сообщений всем клиентам
- Обработку подключений и отключений
- Работу с состоянием приложения

## Архитектура чат-сервера

```
┌──────────────────────────────────────────────────────────────┐
│                        Чат-сервер                            │
│                                                              │
│  ┌─────────────────────────────────────────────────────┐    │
│  │              Менеджер соединений                      │    │
│  │  - clients: Dict[str, StreamWriter]                   │    │
│  │  - broadcast(message)                                 │    │
│  │  - add_client(name, writer)                           │    │
│  │  - remove_client(name)                                │    │
│  └─────────────────────────────────────────────────────┘    │
│                           ▲                                  │
│         ┌─────────────────┼─────────────────┐               │
│         │                 │                 │               │
│    ┌────┴────┐       ┌────┴────┐       ┌────┴────┐         │
│    │ Клиент 1│       │ Клиент 2│       │ Клиент N│         │
│    └─────────┘       └─────────┘       └─────────┘         │
└──────────────────────────────────────────────────────────────┘
```

## Базовая реализация

```python
import asyncio
from datetime import datetime
from typing import Dict

class ChatServer:
    """Простой чат-сервер на asyncio streams."""

    def __init__(self):
        # Словарь: имя клиента -> StreamWriter
        self.clients: Dict[str, asyncio.StreamWriter] = {}

    async def handle_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        """Обрабатывает подключение клиента."""
        addr = writer.get_extra_info('peername')

        # Запрашиваем имя
        writer.write(b'Welcome! Enter your name: ')
        await writer.drain()

        name_data = await reader.readline()
        if not name_data:
            writer.close()
            await writer.wait_closed()
            return

        name = name_data.decode().strip()

        # Проверяем уникальность имени
        if name in self.clients:
            writer.write(b'Name already taken. Goodbye!\n')
            await writer.drain()
            writer.close()
            await writer.wait_closed()
            return

        # Регистрируем клиента
        self.clients[name] = writer
        print(f'[{addr}] {name} joined the chat')

        # Уведомляем всех
        await self.broadcast(f'*** {name} joined the chat ***\n', exclude=name)
        writer.write(f'Hello, {name}! You can start chatting.\n'.encode())
        await writer.drain()

        try:
            # Основной цикл чтения сообщений
            while True:
                data = await reader.readline()
                if not data:
                    break

                message = data.decode().strip()
                if not message:
                    continue

                # Обработка команд
                if message.startswith('/'):
                    await self.handle_command(name, message, writer)
                else:
                    # Обычное сообщение
                    timestamp = datetime.now().strftime('%H:%M')
                    formatted = f'[{timestamp}] {name}: {message}\n'
                    await self.broadcast(formatted, exclude=name)

        except asyncio.CancelledError:
            pass
        except ConnectionResetError:
            pass
        finally:
            # Удаляем клиента
            del self.clients[name]
            await self.broadcast(f'*** {name} left the chat ***\n')
            writer.close()
            await writer.wait_closed()
            print(f'[{addr}] {name} left the chat')

    async def broadcast(self, message: str, exclude: str = None):
        """Отправляет сообщение всем клиентам."""
        data = message.encode()
        disconnected = []

        for name, writer in self.clients.items():
            if name == exclude:
                continue

            try:
                writer.write(data)
                await writer.drain()
            except (ConnectionResetError, BrokenPipeError):
                disconnected.append(name)

        # Удаляем отключившихся клиентов
        for name in disconnected:
            if name in self.clients:
                del self.clients[name]

    async def handle_command(self, sender: str, command: str, writer: asyncio.StreamWriter):
        """Обрабатывает команды чата."""
        parts = command.split(maxsplit=1)
        cmd = parts[0].lower()

        if cmd == '/help':
            help_text = (
                'Available commands:\n'
                '  /help - show this help\n'
                '  /list - list online users\n'
                '  /msg <user> <text> - private message\n'
                '  /quit - leave chat\n'
            )
            writer.write(help_text.encode())
            await writer.drain()

        elif cmd == '/list':
            users = ', '.join(self.clients.keys())
            writer.write(f'Online users: {users}\n'.encode())
            await writer.drain()

        elif cmd == '/msg' and len(parts) > 1:
            # Приватное сообщение
            msg_parts = parts[1].split(maxsplit=1)
            if len(msg_parts) >= 2:
                target, text = msg_parts
                if target in self.clients:
                    target_writer = self.clients[target]
                    target_writer.write(f'[PM from {sender}]: {text}\n'.encode())
                    await target_writer.drain()
                    writer.write(f'[PM to {target}]: {text}\n'.encode())
                    await writer.drain()
                else:
                    writer.write(f'User {target} not found.\n'.encode())
                    await writer.drain()

        elif cmd == '/quit':
            writer.write(b'Goodbye!\n')
            await writer.drain()
            writer.close()

        else:
            writer.write(f'Unknown command: {cmd}. Type /help for help.\n'.encode())
            await writer.drain()

    async def run(self, host: str = '127.0.0.1', port: int = 8888):
        """Запускает сервер."""
        server = await asyncio.start_server(
            self.handle_client,
            host,
            port
        )

        print(f'Chat server running on {host}:{port}')
        print('Press Ctrl+C to stop')

        async with server:
            await server.serve_forever()


if __name__ == '__main__':
    chat = ChatServer()
    asyncio.run(chat.run())
```

## Клиент для чата

```python
import asyncio
import sys

async def create_stdin_reader():
    """Создает StreamReader для stdin."""
    loop = asyncio.get_running_loop()
    reader = asyncio.StreamReader()
    protocol = asyncio.StreamReaderProtocol(reader)
    await loop.connect_read_pipe(lambda: protocol, sys.stdin)
    return reader


async def chat_client(host: str, port: int):
    """Клиент для чат-сервера."""
    try:
        reader, writer = await asyncio.open_connection(host, port)
    except ConnectionRefusedError:
        print(f'Cannot connect to {host}:{port}')
        return

    stdin_reader = await create_stdin_reader()

    async def receive_messages():
        """Получает сообщения от сервера."""
        while True:
            data = await reader.read(4096)
            if not data:
                return 'server_closed'
            # Очищаем строку ввода и печатаем сообщение
            print(f'\r{data.decode()}', end='')
            print('> ', end='', flush=True)

    async def send_messages():
        """Отправляет сообщения на сервер."""
        while True:
            print('> ', end='', flush=True)
            line = await stdin_reader.readline()
            if not line:
                return 'stdin_closed'

            message = line.decode().strip()
            if message.lower() == '/quit':
                writer.write(b'/quit\n')
                await writer.drain()
                return 'user_quit'

            writer.write(f'{message}\n'.encode())
            await writer.drain()

    # Запускаем обе задачи
    receive_task = asyncio.create_task(receive_messages())
    send_task = asyncio.create_task(send_messages())

    done, pending = await asyncio.wait(
        [receive_task, send_task],
        return_when=asyncio.FIRST_COMPLETED
    )

    for task in pending:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass

    writer.close()
    await writer.wait_closed()

    result = done.pop().result()
    if result == 'server_closed':
        print('\nServer closed connection')
    elif result == 'user_quit':
        print('\nGoodbye!')


if __name__ == '__main__':
    host = sys.argv[1] if len(sys.argv) > 1 else '127.0.0.1'
    port = int(sys.argv[2]) if len(sys.argv) > 2 else 8888
    asyncio.run(chat_client(host, port))
```

## Расширенная версия с комнатами

```python
import asyncio
from dataclasses import dataclass, field
from typing import Dict, Set, Optional
from datetime import datetime

@dataclass
class Client:
    """Представляет клиента чата."""
    name: str
    writer: asyncio.StreamWriter
    room: str = 'lobby'
    joined_at: datetime = field(default_factory=datetime.now)


class ChatRoom:
    """Чат-комната."""

    def __init__(self, name: str):
        self.name = name
        self.clients: Set[str] = set()

    async def broadcast(self, message: str, clients_dict: Dict[str, Client], exclude: str = None):
        """Отправляет сообщение всем в комнате."""
        for client_name in self.clients:
            if client_name == exclude:
                continue
            if client_name in clients_dict:
                client = clients_dict[client_name]
                try:
                    client.writer.write(message.encode())
                    await client.writer.drain()
                except:
                    pass


class AdvancedChatServer:
    """Расширенный чат-сервер с комнатами."""

    def __init__(self):
        self.clients: Dict[str, Client] = {}
        self.rooms: Dict[str, ChatRoom] = {'lobby': ChatRoom('lobby')}

    def get_or_create_room(self, name: str) -> ChatRoom:
        """Получает или создает комнату."""
        if name not in self.rooms:
            self.rooms[name] = ChatRoom(name)
        return self.rooms[name]

    async def handle_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        addr = writer.get_extra_info('peername')
        client: Optional[Client] = None

        try:
            # Запрашиваем имя
            writer.write(b'Enter your name: ')
            await writer.drain()

            name_data = await reader.readline()
            if not name_data:
                return

            name = name_data.decode().strip()
            if name in self.clients or not name:
                writer.write(b'Invalid or taken name.\n')
                await writer.drain()
                return

            # Создаем клиента
            client = Client(name=name, writer=writer)
            self.clients[name] = client

            # Добавляем в лобби
            lobby = self.rooms['lobby']
            lobby.clients.add(name)

            print(f'{addr} connected as {name}')
            await lobby.broadcast(f'*** {name} joined lobby ***\n', self.clients, exclude=name)

            writer.write(b'Welcome! Type /help for commands.\n')
            await writer.drain()

            # Основной цикл
            while True:
                data = await reader.readline()
                if not data:
                    break

                message = data.decode().strip()
                if not message:
                    continue

                if message.startswith('/'):
                    await self.handle_command(client, message)
                else:
                    room = self.rooms.get(client.room)
                    if room:
                        timestamp = datetime.now().strftime('%H:%M')
                        formatted = f'[{timestamp}] [{client.room}] {name}: {message}\n'
                        await room.broadcast(formatted, self.clients, exclude=name)

        except (asyncio.CancelledError, ConnectionResetError):
            pass
        finally:
            if client:
                # Удаляем из комнаты
                room = self.rooms.get(client.room)
                if room:
                    room.clients.discard(client.name)
                    await room.broadcast(f'*** {client.name} left ***\n', self.clients)

                del self.clients[client.name]
                print(f'{addr} ({client.name}) disconnected')

            writer.close()
            await writer.wait_closed()

    async def handle_command(self, client: Client, command: str):
        """Обрабатывает команды."""
        parts = command.split(maxsplit=1)
        cmd = parts[0].lower()
        args = parts[1] if len(parts) > 1 else ''
        writer = client.writer

        if cmd == '/help':
            help_text = (
                'Commands:\n'
                '  /help - this help\n'
                '  /rooms - list rooms\n'
                '  /join <room> - join room\n'
                '  /who - who is in current room\n'
                '  /msg <user> <text> - private message\n'
                '  /quit - leave\n'
            )
            writer.write(help_text.encode())
            await writer.drain()

        elif cmd == '/rooms':
            rooms_info = []
            for name, room in self.rooms.items():
                rooms_info.append(f'  {name} ({len(room.clients)} users)')
            writer.write(('Rooms:\n' + '\n'.join(rooms_info) + '\n').encode())
            await writer.drain()

        elif cmd == '/join':
            if not args:
                writer.write(b'Usage: /join <room>\n')
                await writer.drain()
                return

            room_name = args.strip()

            # Покидаем текущую комнату
            old_room = self.rooms.get(client.room)
            if old_room:
                old_room.clients.discard(client.name)
                await old_room.broadcast(
                    f'*** {client.name} left to {room_name} ***\n',
                    self.clients
                )

            # Входим в новую
            new_room = self.get_or_create_room(room_name)
            new_room.clients.add(client.name)
            client.room = room_name

            await new_room.broadcast(
                f'*** {client.name} joined ***\n',
                self.clients,
                exclude=client.name
            )

            writer.write(f'Joined room: {room_name}\n'.encode())
            await writer.drain()

        elif cmd == '/who':
            room = self.rooms.get(client.room)
            if room:
                users = ', '.join(room.clients)
                writer.write(f'Users in {client.room}: {users}\n'.encode())
                await writer.drain()

        elif cmd == '/msg':
            msg_parts = args.split(maxsplit=1)
            if len(msg_parts) < 2:
                writer.write(b'Usage: /msg <user> <message>\n')
                await writer.drain()
                return

            target_name, text = msg_parts
            target = self.clients.get(target_name)

            if target:
                target.writer.write(f'[PM from {client.name}]: {text}\n'.encode())
                await target.writer.drain()
                writer.write(f'[PM to {target_name}]: {text}\n'.encode())
                await writer.drain()
            else:
                writer.write(f'User {target_name} not found.\n'.encode())
                await writer.drain()

        elif cmd == '/quit':
            writer.write(b'Goodbye!\n')
            await writer.drain()
            writer.close()

        else:
            writer.write(f'Unknown command: {cmd}\n'.encode())
            await writer.drain()

    async def run(self, host: str = '127.0.0.1', port: int = 8888):
        server = await asyncio.start_server(self.handle_client, host, port)
        print(f'Advanced chat server on {host}:{port}')
        async with server:
            await server.serve_forever()


if __name__ == '__main__':
    asyncio.run(AdvancedChatServer().run())
```

## Добавляем историю сообщений

```python
from collections import deque

class ChatServerWithHistory:
    """Чат-сервер с историей сообщений."""

    def __init__(self, history_size: int = 50):
        self.clients: Dict[str, asyncio.StreamWriter] = {}
        self.history: deque = deque(maxlen=history_size)

    async def handle_client(self, reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
        # ... регистрация клиента ...

        # Отправляем историю новому клиенту
        if self.history:
            writer.write(b'=== Recent messages ===\n')
            for msg in self.history:
                writer.write(msg.encode())
            writer.write(b'=== End of history ===\n')
            await writer.drain()

        # ... основной цикл ...

    async def broadcast(self, message: str, exclude: str = None):
        # Сохраняем в историю
        self.history.append(message)

        # Отправляем всем
        for name, writer in self.clients.items():
            if name != exclude:
                writer.write(message.encode())
                await writer.drain()
```

## Тестирование с помощью telnet/netcat

```bash
# Подключение через telnet
telnet 127.0.0.1 8888

# Подключение через netcat
nc 127.0.0.1 8888

# Или с rlwrap для удобства редактирования
rlwrap nc 127.0.0.1 8888
```

## Best Practices для чат-серверов

### 1. Обрабатывайте отключения корректно

```python
try:
    writer.write(message)
    await writer.drain()
except (ConnectionResetError, BrokenPipeError):
    # Клиент отключился
    await self.remove_client(name)
```

### 2. Валидируйте входные данные

```python
if not name or len(name) > 20 or not name.isalnum():
    writer.write(b'Invalid name.\n')
    return
```

### 3. Ограничивайте размер сообщений

```python
data = await reader.read(1024)  # Максимум 1 КБ
if len(data) > 500:
    writer.write(b'Message too long.\n')
    continue
```

### 4. Добавляйте rate limiting

```python
import time

class RateLimiter:
    def __init__(self, max_messages: int = 5, period: float = 1.0):
        self.max_messages = max_messages
        self.period = period
        self.timestamps: deque = deque()

    def allow(self) -> bool:
        now = time.time()
        while self.timestamps and now - self.timestamps[0] > self.period:
            self.timestamps.popleft()

        if len(self.timestamps) < self.max_messages:
            self.timestamps.append(now)
            return True
        return False
```

### 5. Логируйте все действия

```python
import logging

logger = logging.getLogger('chat')

async def handle_client(self, reader, writer):
    logger.info(f'New connection from {addr}')
    # ...
    logger.info(f'{name}: {message}')
```

---

[prev: 05-servers.md](./05-servers.md) | [next: ../09-web-apps/readme.md](../09-web-apps/readme.md)
