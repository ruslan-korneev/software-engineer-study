# Подключение через telnet

[prev: ./01-blocking-sockets.md](./01-blocking-sockets.md) | [next: ./03-non-blocking-sockets.md](./03-non-blocking-sockets.md)

---

## Введение

**Telnet** - это сетевой протокол и утилита командной строки для двунаправленной текстовой связи. Telnet незаменим при разработке и отладке сетевых приложений, так как позволяет вручную отправлять данные на сервер и видеть его ответы в реальном времени.

## Что такое Telnet?

Telnet (TELetype NETwork) - протокол прикладного уровня, изначально разработанный для удалённого доступа к компьютерам. Сегодня он в основном используется для:

- **Тестирования серверов** - проверка работоспособности TCP-сервисов
- **Отладки протоколов** - ручной ввод HTTP, SMTP и других текстовых протоколов
- **Проверки портов** - быстрая проверка доступности сервиса

## Установка Telnet

### macOS

```bash
# Telnet входит в состав macOS, но может потребоваться установка
brew install telnet
```

### Linux (Ubuntu/Debian)

```bash
sudo apt-get install telnet
```

### Windows

```powershell
# Включение через PowerShell (с правами администратора)
dism /online /Enable-Feature /FeatureName:TelnetClient
```

## Использование Telnet

### Базовый синтаксис

```bash
telnet <хост> <порт>
```

### Подключение к HTTP-серверу

```bash
telnet example.com 80
```

После подключения можно вручную ввести HTTP-запрос:

```
GET / HTTP/1.1
Host: example.com

```

> Важно: после заголовков нужно дважды нажать Enter (пустая строка).

### Подключение к локальному серверу

```bash
telnet localhost 8080
```

## Создание сервера для тестирования с Telnet

Создадим простой эхо-сервер, с которым можно взаимодействовать через telnet:

```python
import socket

def create_echo_server():
    """Простой эхо-сервер для тестирования с telnet"""
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('127.0.0.1', 8080))
    server.listen(5)

    print("Эхо-сервер запущен на 127.0.0.1:8080")
    print("Подключитесь через: telnet localhost 8080")

    while True:
        client, address = server.accept()
        print(f"Подключился клиент: {address}")

        # Отправляем приветствие
        client.send(b"Welcome to Echo Server!\r\n")
        client.send(b"Type 'quit' to disconnect.\r\n")

        try:
            while True:
                # Получаем данные от клиента
                data = client.recv(1024)
                if not data:
                    break

                message = data.decode().strip()
                print(f"Получено: {message}")

                if message.lower() == 'quit':
                    client.send(b"Goodbye!\r\n")
                    break

                # Отправляем эхо обратно
                response = f"Echo: {message}\r\n"
                client.send(response.encode())

        finally:
            client.close()
            print(f"Клиент {address} отключился")

if __name__ == '__main__':
    create_echo_server()
```

### Тестирование сервера

```bash
# В первом терминале запускаем сервер
python echo_server.py

# Во втором терминале подключаемся через telnet
telnet localhost 8080
```

Пример сессии:

```
Trying 127.0.0.1...
Connected to localhost.
Escape character is '^]'.
Welcome to Echo Server!
Type 'quit' to disconnect.
Hello, World!
Echo: Hello, World!
test message
Echo: test message
quit
Goodbye!
Connection closed by foreign host.
```

## Интерактивный чат-сервер

Более сложный пример - чат-сервер с несколькими клиентами:

```python
import socket
import threading

clients = []
clients_lock = threading.Lock()

def broadcast(message, sender=None):
    """Отправить сообщение всем подключённым клиентам"""
    with clients_lock:
        for client in clients:
            if client != sender:
                try:
                    client.send(message)
                except:
                    clients.remove(client)

def handle_client(client, address):
    """Обработка одного клиента"""
    # Запрашиваем имя
    client.send(b"Enter your name: ")
    name = client.recv(1024).decode().strip()

    welcome = f"\r\n*** {name} joined the chat! ***\r\n"
    broadcast(welcome.encode())
    print(f"{name} ({address}) connected")

    client.send(b"Type 'quit' to leave.\r\n")

    try:
        while True:
            data = client.recv(1024)
            if not data:
                break

            message = data.decode().strip()
            if message.lower() == 'quit':
                break

            # Транслируем сообщение всем
            broadcast_msg = f"{name}: {message}\r\n"
            broadcast(broadcast_msg.encode(), client)
            print(f"{name}: {message}")

    finally:
        with clients_lock:
            if client in clients:
                clients.remove(client)

        goodbye = f"\r\n*** {name} left the chat ***\r\n"
        broadcast(goodbye.encode())
        client.close()
        print(f"{name} disconnected")

def main():
    server = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server.bind(('127.0.0.1', 8080))
    server.listen(10)

    print("Chat server started on 127.0.0.1:8080")

    while True:
        client, address = server.accept()
        with clients_lock:
            clients.append(client)

        thread = threading.Thread(target=handle_client, args=(client, address))
        thread.daemon = True
        thread.start()

if __name__ == '__main__':
    main()
```

## Работа с Telnet из Python

Модуль `telnetlib` позволяет программно работать с telnet-сессиями:

```python
import telnetlib

# Подключение к серверу
tn = telnetlib.Telnet('localhost', 8080)

# Читаем до определённого текста
response = tn.read_until(b"name: ")
print(response.decode())

# Отправляем данные
tn.write(b"PythonBot\n")

# Читаем ответ
response = tn.read_until(b"quit' to leave.\r\n")
print(response.decode())

# Отправляем сообщение
tn.write(b"Hello from Python!\n")

# Читаем с таймаутом
response = tn.read_until(b"\r\n", timeout=5)
print(response.decode())

# Закрываем соединение
tn.write(b"quit\n")
tn.close()
```

### Асинхронная версия с telnetlib3

```python
import asyncio
import telnetlib3

async def telnet_client():
    reader, writer = await telnetlib3.open_connection('localhost', 8080)

    # Читаем приветствие
    output = await reader.read(1024)
    print(output)

    # Отправляем данные
    writer.write('Hello from async client!\n')

    # Читаем ответ
    output = await reader.read(1024)
    print(output)

    writer.close()

asyncio.run(telnet_client())
```

## Telnet для отладки протоколов

### Тестирование SMTP-сервера

```bash
telnet smtp.example.com 25
```

```
HELO mydomain.com
MAIL FROM:<sender@example.com>
RCPT TO:<recipient@example.com>
DATA
Subject: Test email

This is a test message.
.
QUIT
```

### Тестирование POP3-сервера

```bash
telnet pop.example.com 110
```

```
USER myusername
PASS mypassword
LIST
RETR 1
QUIT
```

### Тестирование Redis

```bash
telnet localhost 6379
```

```
PING
SET mykey "Hello"
GET mykey
QUIT
```

## Альтернативы Telnet

### Netcat (nc)

Более мощный инструмент с дополнительными возможностями:

```bash
# Подключение к серверу
nc localhost 8080

# Прослушивание порта (создание простого сервера)
nc -l 8080

# Передача файла
nc -l 8080 > received_file.txt  # На сервере
nc localhost 8080 < file.txt     # На клиенте
```

### Curl для HTTP

```bash
# Более удобный инструмент для HTTP
curl http://localhost:8080

# С подробным выводом
curl -v http://localhost:8080
```

## Обработка специальных символов Telnet

Telnet использует специальные управляющие последовательности:

```python
import socket

def handle_telnet_client(client):
    """Обработка telnet-клиента с управляющими последовательностями"""

    # Telnet команды
    IAC = 255   # Interpret As Command
    DONT = 254
    DO = 253
    WONT = 252
    WILL = 251

    while True:
        data = client.recv(1024)
        if not data:
            break

        # Фильтруем telnet-команды
        cleaned = bytearray()
        i = 0
        while i < len(data):
            if data[i] == IAC and i + 2 < len(data):
                # Пропускаем telnet-команду (3 байта)
                i += 3
            else:
                cleaned.append(data[i])
                i += 1

        if cleaned:
            message = bytes(cleaned).decode().strip()
            if message:
                print(f"Received: {message}")
                client.send(f"Echo: {message}\r\n".encode())
```

## Лучшие практики

1. **Используйте `\r\n`** - telnet ожидает CRLF как разделитель строк
2. **Отправляйте приветствие** - помогает понять, что соединение установлено
3. **Обрабатывайте telnet-команды** - клиенты могут отправлять управляющие последовательности
4. **Добавляйте таймауты** - защита от зависших соединений
5. **Логируйте соединения** - упрощает отладку

## Распространённые проблемы

| Проблема | Причина | Решение |
|----------|---------|---------|
| `Connection refused` | Сервер не запущен | Запустите сервер |
| Пустой ответ | Сервер не отправляет данные | Проверьте логику сервера |
| Обрезанный текст | Нет `\r\n` в конце | Добавьте `\r\n` к сообщениям |
| Странные символы | Telnet-команды | Фильтруйте IAC-последовательности |

## Заключение

Telnet - незаменимый инструмент при разработке сетевых приложений. Он позволяет вручную тестировать TCP-серверы, отлаживать протоколы и проверять работоспособность сервисов. Хотя telnet устарел как средство удалённого доступа (заменён на SSH), он остаётся полезным для разработки и тестирования.

---

[prev: ./01-blocking-sockets.md](./01-blocking-sockets.md) | [next: ./03-non-blocking-sockets.md](./03-non-blocking-sockets.md)
