# Модуль selectors

[prev: ./03-non-blocking-sockets.md](./03-non-blocking-sockets.md) | [next: ./05-echo-server.md](./05-echo-server.md)

---

## Введение

Модуль `selectors` предоставляет высокоуровневый интерфейс для мультиплексирования ввода-вывода. Он автоматически выбирает наиболее эффективный механизм для вашей операционной системы (epoll на Linux, kqueue на macOS/BSD, select на других).

## Зачем нужен selectors?

При работе с множеством сокетов возникает вопрос: как узнать, какой сокет готов к чтению или записи? Есть несколько низкоуровневых механизмов:

| Механизм | ОС | Сложность | Масштабируемость |
|----------|-----|-----------|------------------|
| `select` | Все | O(n) | До ~1024 сокетов |
| `poll` | Unix | O(n) | Неограниченно |
| `epoll` | Linux | O(1) | Миллионы соединений |
| `kqueue` | BSD/macOS | O(1) | Миллионы соединений |

Модуль `selectors` абстрагирует эти различия, предоставляя единый API.

## Базовое использование

```python
import selectors
import socket

# Создаём селектор (автоматически выбирает лучший для ОС)
sel = selectors.DefaultSelector()

# Создаём серверный сокет
server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server.bind(('127.0.0.1', 8080))
server.listen(100)
server.setblocking(False)

# Регистрируем сокет для отслеживания событий чтения
sel.register(server, selectors.EVENT_READ, data=None)

print("Сервер запущен на 127.0.0.1:8080")

while True:
    # Ждём событий (timeout=None означает бесконечное ожидание)
    events = sel.select(timeout=None)

    for key, mask in events:
        if key.data is None:
            # Это серверный сокет - новое соединение
            client, addr = server.accept()
            print(f"Подключён: {addr}")
            client.setblocking(False)
            # Регистрируем клиентский сокет
            sel.register(client, selectors.EVENT_READ, data=addr)
        else:
            # Это клиентский сокет - есть данные
            sock = key.fileobj
            addr = key.data
            try:
                data = sock.recv(1024)
                if data:
                    print(f"От {addr}: {data.decode().strip()}")
                    sock.send(f"Echo: {data.decode()}".encode())
                else:
                    print(f"Отключён: {addr}")
                    sel.unregister(sock)
                    sock.close()
            except ConnectionResetError:
                print(f"Сброс соединения: {addr}")
                sel.unregister(sock)
                sock.close()
```

## Ключевые компоненты

### SelectorKey

При регистрации сокета возвращается `SelectorKey`:

```python
import selectors

sel = selectors.DefaultSelector()

# register возвращает SelectorKey
key = sel.register(socket, selectors.EVENT_READ, data="my_data")

print(key.fileobj)  # Зарегистрированный объект (сокет)
print(key.fd)       # Файловый дескриптор
print(key.events)   # Отслеживаемые события (EVENT_READ, EVENT_WRITE)
print(key.data)     # Пользовательские данные
```

### События

```python
import selectors

# Доступные события
selectors.EVENT_READ   # Готов к чтению
selectors.EVENT_WRITE  # Готов к записи

# Комбинация событий
both = selectors.EVENT_READ | selectors.EVENT_WRITE
```

### Методы селектора

```python
sel = selectors.DefaultSelector()

# Регистрация
sel.register(fileobj, events, data=None)

# Изменение отслеживаемых событий
sel.modify(fileobj, events, data=None)

# Удаление из отслеживания
sel.unregister(fileobj)

# Ожидание событий
events = sel.select(timeout=None)  # None = бесконечно

# Получение информации о зарегистрированном объекте
key = sel.get_key(fileobj)

# Карта всех зарегистрированных объектов
for fileobj, key in sel.get_map().items():
    print(fileobj, key)

# Закрытие селектора
sel.close()
```

## Полноценный сервер с selectors

```python
import selectors
import socket
from dataclasses import dataclass

@dataclass
class ConnectionData:
    """Данные о соединении"""
    address: tuple
    recv_buffer: bytes = b''
    send_buffer: bytes = b''

def accept_connection(sel, server_socket):
    """Обработка нового соединения"""
    client, addr = server_socket.accept()
    print(f"Подключён: {addr}")
    client.setblocking(False)

    # Создаём данные соединения
    conn_data = ConnectionData(address=addr)
    conn_data.send_buffer = b"Welcome to the server!\r\n"

    # Регистрируем для чтения И записи
    sel.register(client, selectors.EVENT_READ | selectors.EVENT_WRITE, data=conn_data)

def handle_client(sel, key, mask):
    """Обработка клиентского сокета"""
    sock = key.fileobj
    data = key.data

    if mask & selectors.EVENT_READ:
        # Можно читать
        try:
            recv_data = sock.recv(1024)
            if recv_data:
                data.recv_buffer += recv_data

                # Обрабатываем полные строки
                while b'\n' in data.recv_buffer:
                    line, data.recv_buffer = data.recv_buffer.split(b'\n', 1)
                    message = line.decode().strip()
                    print(f"{data.address}: {message}")

                    if message.lower() == 'quit':
                        data.send_buffer += b"Goodbye!\r\n"
                        # Отключим после отправки
                    else:
                        data.send_buffer += f"Echo: {message}\r\n".encode()
            else:
                # Пустые данные = клиент отключился
                print(f"Отключён: {data.address}")
                sel.unregister(sock)
                sock.close()
                return
        except ConnectionResetError:
            print(f"Сброс соединения: {data.address}")
            sel.unregister(sock)
            sock.close()
            return

    if mask & selectors.EVENT_WRITE:
        # Можно писать
        if data.send_buffer:
            try:
                sent = sock.send(data.send_buffer)
                data.send_buffer = data.send_buffer[sent:]
            except (ConnectionResetError, BrokenPipeError):
                sel.unregister(sock)
                sock.close()
                return

def run_server():
    """Запуск сервера"""
    sel = selectors.DefaultSelector()

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('127.0.0.1', 8080))
    server.listen(100)
    server.setblocking(False)

    # Регистрируем серверный сокет
    sel.register(server, selectors.EVENT_READ, data=None)

    print("Сервер с selectors запущен на 127.0.0.1:8080")

    try:
        while True:
            events = sel.select(timeout=1)

            for key, mask in events:
                if key.data is None:
                    # Серверный сокет
                    accept_connection(sel, key.fileobj)
                else:
                    # Клиентский сокет
                    handle_client(sel, key, mask)

    except KeyboardInterrupt:
        print("\nОстановка сервера...")
    finally:
        sel.close()
        server.close()

if __name__ == '__main__':
    run_server()
```

## Динамическое изменение событий

Иногда нужно менять отслеживаемые события в процессе работы:

```python
import selectors
import socket

sel = selectors.DefaultSelector()

def handle_client(key, mask):
    sock = key.fileobj
    data = key.data

    if mask & selectors.EVENT_READ:
        received = sock.recv(1024)
        if received:
            data['buffer'] += received
            # Есть данные для отправки - добавляем EVENT_WRITE
            sel.modify(sock, selectors.EVENT_READ | selectors.EVENT_WRITE, data=data)

    if mask & selectors.EVENT_WRITE:
        if data['buffer']:
            sent = sock.send(data['buffer'])
            data['buffer'] = data['buffer'][sent:]

            if not data['buffer']:
                # Буфер пуст - убираем EVENT_WRITE
                sel.modify(sock, selectors.EVENT_READ, data=data)
```

## Типы селекторов

```python
import selectors

# Автоматический выбор лучшего для ОС
sel = selectors.DefaultSelector()

# Явный выбор типа
sel = selectors.SelectSelector()    # select() - везде работает
sel = selectors.PollSelector()      # poll() - Unix
sel = selectors.EpollSelector()     # epoll() - Linux
sel = selectors.KqueueSelector()    # kqueue() - macOS, BSD
sel = selectors.DevpollSelector()   # /dev/poll - Solaris

# Проверка доступности
print(selectors.DefaultSelector)  # <class 'selectors.EpollSelector'>
```

## Сравнение производительности

```python
import selectors
import time

# Тест: сколько сокетов можно обработать
def benchmark_selector(selector_class, num_sockets=1000):
    sel = selector_class()

    sockets = []
    for _ in range(num_sockets):
        s = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        s.setblocking(False)
        try:
            s.connect(('127.0.0.1', 80))
        except BlockingIOError:
            pass
        sel.register(s, selectors.EVENT_READ)
        sockets.append(s)

    start = time.time()
    for _ in range(1000):
        sel.select(timeout=0)
    elapsed = time.time() - start

    for s in sockets:
        sel.unregister(s)
        s.close()
    sel.close()

    return elapsed

# epoll O(1), select O(n)
```

## Эхо-сервер с классами

```python
import selectors
import socket
from typing import Optional

class Server:
    """Сервер на основе selectors"""

    def __init__(self, host: str, port: int):
        self.host = host
        self.port = port
        self.sel = selectors.DefaultSelector()
        self.server_socket: Optional[socket.socket] = None

    def start(self):
        """Запуск сервера"""
        self.server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
        self.server_socket.bind((self.host, self.port))
        self.server_socket.listen(100)
        self.server_socket.setblocking(False)

        self.sel.register(
            self.server_socket,
            selectors.EVENT_READ,
            data=self._accept
        )

        print(f"Сервер запущен на {self.host}:{self.port}")
        self._event_loop()

    def _event_loop(self):
        """Главный цикл обработки событий"""
        try:
            while True:
                events = self.sel.select(timeout=1)
                for key, mask in events:
                    callback = key.data
                    callback(key.fileobj, mask)
        except KeyboardInterrupt:
            print("\nОстановка...")
        finally:
            self.stop()

    def _accept(self, sock: socket.socket, mask: int):
        """Обработка нового соединения"""
        client, addr = sock.accept()
        print(f"Подключён: {addr}")
        client.setblocking(False)

        # Создаём обработчик для клиента
        handler = ClientHandler(client, addr, self.sel)
        self.sel.register(
            client,
            selectors.EVENT_READ | selectors.EVENT_WRITE,
            data=handler.handle
        )

    def stop(self):
        """Остановка сервера"""
        self.sel.close()
        if self.server_socket:
            self.server_socket.close()


class ClientHandler:
    """Обработчик клиентского соединения"""

    def __init__(self, sock: socket.socket, addr: tuple, sel: selectors.BaseSelector):
        self.sock = sock
        self.addr = addr
        self.sel = sel
        self.recv_buffer = b''
        self.send_buffer = b'Welcome!\r\n'

    def handle(self, sock: socket.socket, mask: int):
        """Обработка событий сокета"""
        if mask & selectors.EVENT_READ:
            self._read()

        if mask & selectors.EVENT_WRITE:
            self._write()

    def _read(self):
        """Чтение данных"""
        try:
            data = self.sock.recv(1024)
            if data:
                self.recv_buffer += data
                self._process_messages()
            else:
                self._close()
        except ConnectionResetError:
            self._close()

    def _write(self):
        """Отправка данных"""
        if self.send_buffer:
            try:
                sent = self.sock.send(self.send_buffer)
                self.send_buffer = self.send_buffer[sent:]
            except (ConnectionResetError, BrokenPipeError):
                self._close()

    def _process_messages(self):
        """Обработка полученных сообщений"""
        while b'\n' in self.recv_buffer:
            line, self.recv_buffer = self.recv_buffer.split(b'\n', 1)
            message = line.decode().strip()
            print(f"{self.addr}: {message}")
            self.send_buffer += f"Echo: {message}\r\n".encode()

    def _close(self):
        """Закрытие соединения"""
        print(f"Отключён: {self.addr}")
        self.sel.unregister(self.sock)
        self.sock.close()


if __name__ == '__main__':
    server = Server('127.0.0.1', 8080)
    server.start()
```

## Интеграция с asyncio

Модуль `asyncio` внутри использует `selectors`:

```python
import asyncio
import selectors

# asyncio использует DefaultSelector
loop = asyncio.new_event_loop()

# Можно получить доступ к селектору
selector = loop._selector  # Внутренний атрибут

# Или создать loop с конкретным селектором
class CustomSelectorEventLoop(asyncio.SelectorEventLoop):
    def __init__(self):
        selector = selectors.SelectSelector()  # Явный select
        super().__init__(selector)
```

## Таймауты и периодические задачи

```python
import selectors
import socket
import time

sel = selectors.DefaultSelector()
last_check = time.time()

# ... регистрация сокетов ...

while True:
    # Таймаут 1 секунда - для периодических задач
    events = sel.select(timeout=1.0)

    for key, mask in events:
        handle_event(key, mask)

    # Периодическая задача каждые 5 секунд
    if time.time() - last_check > 5:
        print("Периодическая проверка...")
        cleanup_stale_connections()
        last_check = time.time()
```

## Лучшие практики

1. **Используйте DefaultSelector** - он выберет лучший вариант для ОС
2. **Храните данные в key.data** - не создавайте отдельные словари
3. **Обрабатывайте все события** - не игнорируйте EVENT_WRITE
4. **Закрывайте селектор** - используйте try/finally или контекстный менеджер
5. **Устанавливайте таймауты** - для периодических задач и проверок

```python
# Контекстный менеджер
with selectors.DefaultSelector() as sel:
    sel.register(sock, selectors.EVENT_READ)
    events = sel.select()
# Автоматически закрывается
```

## Распространённые ошибки

1. **Забыли unregister** - при закрытии сокета нужно снять регистрацию
2. **Блокирующий сокет** - setblocking(False) обязателен
3. **Не обработали EVENT_WRITE** - данные не отправляются
4. **Слишком маленький таймаут** - высокая нагрузка на CPU

## Заключение

Модуль `selectors` - удобная абстракция над низкоуровневыми механизмами мультиплексирования. Он автоматически выбирает лучший механизм для вашей ОС и предоставляет простой API. Это основа для понимания работы `asyncio`.

В следующей теме мы построим полноценный эхо-сервер на `asyncio`, который использует все изученные концепции.

---

[prev: ./03-non-blocking-sockets.md](./03-non-blocking-sockets.md) | [next: ./05-echo-server.md](./05-echo-server.md)
