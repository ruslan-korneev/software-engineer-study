# Неблокирующие сокеты

[prev: ./02-telnet.md](./02-telnet.md) | [next: ./04-selectors.md](./04-selectors.md)

---

## Введение

Неблокирующие сокеты - это механизм, при котором операции ввода-вывода возвращаются немедленно, не дожидаясь завершения. Это позволяет одному потоку обрабатывать множество соединений одновременно без использования многопоточности.

## Блокирующий vs Неблокирующий режим

| Аспект | Блокирующий | Неблокирующий |
|--------|-------------|---------------|
| Поведение `recv()` | Ждёт данные | Возвращает сразу (или ошибку) |
| Поведение `accept()` | Ждёт соединение | Возвращает сразу (или ошибку) |
| Количество соединений | Одно на поток | Много в одном потоке |
| Сложность кода | Простой | Более сложный |
| Ошибка при отсутствии данных | Нет | `BlockingIOError` |

## Включение неблокирующего режима

```python
import socket

# Способ 1: setblocking(False)
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setblocking(False)

# Способ 2: settimeout(0)
sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.settimeout(0)  # 0 = неблокирующий режим

# Проверка режима
print(sock.getblocking())  # False
print(sock.gettimeout())   # 0.0
```

## Обработка неблокирующих операций

При неблокирующем режиме операции выбрасывают `BlockingIOError`, если не могут быть выполнены немедленно:

```python
import socket

sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
sock.setblocking(False)
sock.bind(('127.0.0.1', 8080))
sock.listen(5)

try:
    # Если нет ожидающих соединений - будет исключение
    client, addr = sock.accept()
except BlockingIOError:
    print("Нет входящих соединений")

# Аналогично с recv()
try:
    data = client.recv(1024)
except BlockingIOError:
    print("Нет данных для чтения")
```

## Простой неблокирующий сервер

Базовый пример сервера с polling (опросом):

```python
import socket
import time

def simple_nonblocking_server():
    """Простой неблокирующий сервер с polling"""

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('127.0.0.1', 8080))
    server.listen(5)
    server.setblocking(False)

    clients = []

    print("Неблокирующий сервер запущен на 127.0.0.1:8080")

    while True:
        # Пытаемся принять новое соединение
        try:
            client, addr = server.accept()
            client.setblocking(False)
            clients.append(client)
            print(f"Новое соединение: {addr}")
        except BlockingIOError:
            pass  # Нет новых соединений

        # Обрабатываем существующие соединения
        for client in clients[:]:  # Копия списка для безопасного удаления
            try:
                data = client.recv(1024)
                if data:
                    print(f"Получено: {data.decode().strip()}")
                    client.send(f"Echo: {data.decode()}".encode())
                else:
                    # Пустые данные = клиент отключился
                    print("Клиент отключился")
                    clients.remove(client)
                    client.close()
            except BlockingIOError:
                pass  # Нет данных от этого клиента
            except ConnectionResetError:
                print("Соединение сброшено")
                clients.remove(client)
                client.close()

        # Небольшая пауза, чтобы не загружать CPU на 100%
        time.sleep(0.01)

if __name__ == '__main__':
    simple_nonblocking_server()
```

## Проблема polling

Простой polling неэффективен:

```python
while True:
    for socket in sockets:
        try:
            data = socket.recv(1024)  # Проверяем КАЖДЫЙ сокет
        except BlockingIOError:
            pass  # 99% времени ничего нет

    time.sleep(0.01)  # Без паузы - 100% CPU
                      # С паузой - задержка отклика
```

**Проблемы:**
1. **CPU-интенсивность** - постоянный опрос даже неактивных сокетов
2. **Задержка** - пауза между циклами увеличивает время отклика
3. **Масштабируемость** - O(n) проверок на каждой итерации

## Улучшенный сервер с буферизацией

```python
import socket

def buffered_nonblocking_server():
    """Сервер с буферами для отправки и получения"""

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('127.0.0.1', 8080))
    server.listen(5)
    server.setblocking(False)

    # Для каждого клиента храним буферы
    connections = {}  # socket -> {'recv_buffer': b'', 'send_buffer': b''}

    print("Буферизованный сервер запущен")

    while True:
        # Принимаем новые соединения
        try:
            client, addr = server.accept()
            client.setblocking(False)
            connections[client] = {
                'recv_buffer': b'',
                'send_buffer': b'Welcome!\r\n',
                'address': addr
            }
            print(f"Подключён: {addr}")
        except BlockingIOError:
            pass

        # Обрабатываем клиентов
        disconnected = []

        for client, buffers in connections.items():
            # Отправляем данные из буфера
            if buffers['send_buffer']:
                try:
                    sent = client.send(buffers['send_buffer'])
                    buffers['send_buffer'] = buffers['send_buffer'][sent:]
                except BlockingIOError:
                    pass  # Буфер отправки полон, попробуем позже
                except (ConnectionResetError, BrokenPipeError):
                    disconnected.append(client)
                    continue

            # Читаем данные в буфер
            try:
                data = client.recv(1024)
                if data:
                    buffers['recv_buffer'] += data

                    # Обрабатываем полные строки
                    while b'\n' in buffers['recv_buffer']:
                        line, buffers['recv_buffer'] = \
                            buffers['recv_buffer'].split(b'\n', 1)

                        message = line.decode().strip()
                        print(f"{buffers['address']}: {message}")

                        # Добавляем эхо в буфер отправки
                        buffers['send_buffer'] += f"Echo: {message}\r\n".encode()
                else:
                    disconnected.append(client)
            except BlockingIOError:
                pass
            except (ConnectionResetError, BrokenPipeError):
                disconnected.append(client)

        # Удаляем отключённых
        for client in disconnected:
            print(f"Отключён: {connections[client]['address']}")
            del connections[client]
            client.close()

if __name__ == '__main__':
    buffered_nonblocking_server()
```

## Неблокирующий connect()

Подключение к серверу тоже может быть неблокирующим:

```python
import socket
import errno

def nonblocking_connect(host, port):
    """Неблокирующее подключение к серверу"""

    sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    sock.setblocking(False)

    try:
        sock.connect((host, port))
    except BlockingIOError:
        # Это нормально для неблокирующего connect
        pass

    # Проверяем статус подключения
    import select
    _, writable, _ = select.select([], [sock], [], 5.0)

    if writable:
        # Проверяем на ошибки
        error = sock.getsockopt(socket.SOL_SOCKET, socket.SO_ERROR)
        if error == 0:
            print("Подключение успешно!")
            return sock
        else:
            raise OSError(error, f"Ошибка подключения: {errno.errorcode.get(error, error)}")
    else:
        raise TimeoutError("Таймаут подключения")

# Использование
try:
    sock = nonblocking_connect('example.com', 80)
    sock.send(b'GET / HTTP/1.1\r\nHost: example.com\r\n\r\n')
except Exception as e:
    print(f"Ошибка: {e}")
```

## Неблокирующий клиент

```python
import socket

class NonBlockingClient:
    """Неблокирующий TCP-клиент"""

    def __init__(self, host, port):
        self.sock = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.sock.setblocking(False)
        self.host = host
        self.port = port
        self.connected = False
        self.send_buffer = b''
        self.recv_buffer = b''

    def connect(self):
        """Начать подключение (неблокирующее)"""
        try:
            self.sock.connect((self.host, self.port))
            self.connected = True
        except BlockingIOError:
            # Подключение в процессе
            pass

    def is_connected(self):
        """Проверить, завершилось ли подключение"""
        if self.connected:
            return True

        import select
        _, writable, _ = select.select([], [self.sock], [], 0)
        if writable:
            error = self.sock.getsockopt(socket.SOL_SOCKET, socket.SO_ERROR)
            if error == 0:
                self.connected = True
                return True
            else:
                raise ConnectionError(f"Ошибка подключения: {error}")
        return False

    def send(self, data):
        """Добавить данные в буфер отправки"""
        self.send_buffer += data

    def flush(self):
        """Попытаться отправить данные из буфера"""
        if not self.send_buffer:
            return

        try:
            sent = self.sock.send(self.send_buffer)
            self.send_buffer = self.send_buffer[sent:]
        except BlockingIOError:
            pass

    def recv(self):
        """Попытаться получить данные"""
        try:
            data = self.sock.recv(4096)
            if data:
                self.recv_buffer += data
                return data
            else:
                raise ConnectionError("Соединение закрыто")
        except BlockingIOError:
            return None

    def close(self):
        self.sock.close()

# Пример использования
client = NonBlockingClient('127.0.0.1', 8080)
client.connect()

import time
while not client.is_connected():
    time.sleep(0.1)
    print("Подключаемся...")

print("Подключено!")
client.send(b"Hello, Server!\r\n")

while True:
    client.flush()
    data = client.recv()
    if data:
        print(f"Получено: {data.decode()}")
        break
    time.sleep(0.1)

client.close()
```

## Паттерны работы с неблокирующими сокетами

### 1. Event Loop (Цикл событий)

```python
import socket
import select

def event_loop():
    """Простой event loop с select"""

    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('127.0.0.1', 8080))
    server.listen(5)
    server.setblocking(False)

    sockets = [server]
    clients = {}

    print("Event loop сервер запущен")

    while True:
        # Ждём события на сокетах
        readable, writable, errored = select.select(sockets, [], sockets, 1.0)

        for sock in readable:
            if sock is server:
                # Новое соединение
                client, addr = server.accept()
                client.setblocking(False)
                sockets.append(client)
                clients[client] = addr
                print(f"Подключён: {addr}")
            else:
                # Данные от клиента
                try:
                    data = sock.recv(1024)
                    if data:
                        print(f"От {clients[sock]}: {data.decode().strip()}")
                        sock.send(f"Echo: {data.decode()}".encode())
                    else:
                        print(f"Отключён: {clients[sock]}")
                        sockets.remove(sock)
                        del clients[sock]
                        sock.close()
                except Exception as e:
                    print(f"Ошибка: {e}")
                    sockets.remove(sock)
                    if sock in clients:
                        del clients[sock]
                    sock.close()

        for sock in errored:
            print(f"Ошибка на сокете")
            sockets.remove(sock)
            if sock in clients:
                del clients[sock]
            sock.close()

if __name__ == '__main__':
    event_loop()
```

### 2. Callback-based (На колбэках)

```python
import socket

class Connection:
    """Соединение с callback-ами"""

    def __init__(self, sock, address, on_data, on_close):
        self.sock = sock
        self.address = address
        self.on_data = on_data
        self.on_close = on_close
        self.send_buffer = b''

    def handle_read(self):
        try:
            data = self.sock.recv(1024)
            if data:
                self.on_data(self, data)
            else:
                self.on_close(self)
        except BlockingIOError:
            pass
        except Exception:
            self.on_close(self)

    def handle_write(self):
        if self.send_buffer:
            try:
                sent = self.sock.send(self.send_buffer)
                self.send_buffer = self.send_buffer[sent:]
            except BlockingIOError:
                pass

    def send(self, data):
        self.send_buffer += data

    def close(self):
        self.sock.close()

# Использование
def on_data(conn, data):
    print(f"Получено от {conn.address}: {data.decode().strip()}")
    conn.send(f"Echo: {data.decode()}".encode())

def on_close(conn):
    print(f"Отключён: {conn.address}")
    conn.close()
```

## Errno коды

Важные коды ошибок при работе с неблокирующими сокетами:

```python
import errno

# Частые коды ошибок
EAGAIN = errno.EAGAIN           # Попробуйте позже (нет данных)
EWOULDBLOCK = errno.EWOULDBLOCK # То же, что EAGAIN
EINPROGRESS = errno.EINPROGRESS # Операция в процессе (connect)
EISCONN = errno.EISCONN         # Уже подключён
ENOTCONN = errno.ENOTCONN       # Не подключён

# Проверка ошибки
import socket
sock = socket.socket()
sock.setblocking(False)

try:
    sock.connect(('127.0.0.1', 8080))
except OSError as e:
    if e.errno == errno.EINPROGRESS:
        print("Подключение в процессе...")
    elif e.errno == errno.EISCONN:
        print("Уже подключён")
    else:
        raise
```

## Лучшие практики

1. **Всегда используйте select/poll/epoll** - не делайте busy-wait
2. **Буферизуйте данные** - не теряйте частично отправленные сообщения
3. **Обрабатывайте все ошибки** - BlockingIOError, ConnectionResetError и др.
4. **Используйте высокоуровневые абстракции** - asyncio, selectors
5. **Устанавливайте таймауты** на уровне приложения

```python
# Плохо - busy wait
while True:
    try:
        data = sock.recv(1024)
    except BlockingIOError:
        pass  # CPU на 100%

# Хорошо - с select
import select
readable, _, _ = select.select([sock], [], [], 1.0)
if readable:
    data = sock.recv(1024)
```

## Сравнение подходов

| Подход | Плюсы | Минусы |
|--------|-------|--------|
| Блокирующий + потоки | Простой код | Накладные расходы на потоки |
| Неблокирующий + polling | Один поток | CPU-интенсивный |
| Неблокирующий + select | Эффективный | Ограничение на количество сокетов |
| asyncio | Удобный API | Нужно изучить async/await |

## Заключение

Неблокирующие сокеты - основа для построения высокопроизводительных сетевых приложений. Они позволяют обрабатывать тысячи соединений в одном потоке. Однако работа с ними напрямую сложна - поэтому обычно используют `select`, `selectors` или `asyncio`.

В следующей теме мы изучим модуль `selectors`, который предоставляет удобный интерфейс для мультиплексирования ввода-вывода.

---

[prev: ./02-telnet.md](./02-telnet.md) | [next: ./04-selectors.md](./04-selectors.md)
