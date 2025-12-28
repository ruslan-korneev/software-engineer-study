# Работа с блокирующими сокетами

[prev: ./readme.md](./readme.md) | [next: ./02-telnet.md](./02-telnet.md)

---

## Введение

Сокеты - это низкоуровневый механизм сетевого взаимодействия между процессами. В Python модуль `socket` предоставляет доступ к BSD socket API. По умолчанию все сокеты работают в **блокирующем режиме** - это означает, что операции чтения и записи приостанавливают выполнение программы до их завершения.

## Что такое блокирующий сокет?

Блокирующий сокет - это сокет, который при выполнении операций ввода-вывода (I/O) останавливает выполнение текущего потока до тех пор, пока операция не будет завершена.

### Основные блокирующие операции:

| Операция | Описание | Что блокирует |
|----------|----------|---------------|
| `accept()` | Ожидание входящего соединения | Блокирует до подключения клиента |
| `recv()` | Чтение данных из сокета | Блокирует до получения данных |
| `send()` | Отправка данных | Блокирует до отправки (буфер полон) |
| `connect()` | Установка соединения | Блокирует до установки соединения |

## Создание простого TCP-сервера

```python
import socket

# Создаём TCP-сокет
server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Позволяет повторно использовать адрес (избегаем ошибки "Address already in use")
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)

# Привязываем сокет к адресу и порту
server_socket.bind(('127.0.0.1', 8080))

# Начинаем слушать входящие соединения (очередь до 5 соединений)
server_socket.listen(5)

print("Сервер запущен на 127.0.0.1:8080")

while True:
    # БЛОКИРУЮЩИЙ вызов - программа ждёт подключения клиента
    client_socket, client_address = server_socket.accept()
    print(f"Подключился клиент: {client_address}")

    # БЛОКИРУЮЩИЙ вызов - ждём данные от клиента
    data = client_socket.recv(1024)
    print(f"Получены данные: {data.decode()}")

    # Отправляем ответ
    client_socket.send(b"Hello from server!")

    # Закрываем соединение с клиентом
    client_socket.close()
```

## Создание TCP-клиента

```python
import socket

# Создаём TCP-сокет
client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# БЛОКИРУЮЩИЙ вызов - подключаемся к серверу
client_socket.connect(('127.0.0.1', 8080))
print("Подключились к серверу")

# Отправляем данные
client_socket.send(b"Hello from client!")

# БЛОКИРУЮЩИЙ вызов - ждём ответ от сервера
response = client_socket.recv(1024)
print(f"Ответ сервера: {response.decode()}")

# Закрываем соединение
client_socket.close()
```

## Проблема блокирующих сокетов

Главная проблема блокирующих сокетов - **невозможность обслуживать несколько клиентов одновременно**:

```python
import socket
import time

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_socket.bind(('127.0.0.1', 8080))
server_socket.listen(5)

print("Сервер запущен...")

while True:
    # Пока мы обрабатываем одного клиента,
    # другие клиенты ждут в очереди!
    client_socket, address = server_socket.accept()
    print(f"Подключился: {address}")

    # Симулируем долгую обработку
    time.sleep(5)  # Другие клиенты ждут 5 секунд!

    data = client_socket.recv(1024)
    client_socket.send(b"Done!")
    client_socket.close()
```

### Демонстрация проблемы

Запустите сервер выше, затем в двух терминалах одновременно:

```bash
# Терминал 1
python -c "import socket; s=socket.socket(); s.connect(('127.0.0.1',8080)); print('Connected 1')"

# Терминал 2 (сразу после первого)
python -c "import socket; s=socket.socket(); s.connect(('127.0.0.1',8080)); print('Connected 2')"
```

Второй клиент будет ждать, пока первый не будет обслужен!

## Решения проблемы блокировки

### 1. Многопоточность (threading)

```python
import socket
import threading

def handle_client(client_socket, address):
    """Обработчик для каждого клиента в отдельном потоке"""
    print(f"Обработка клиента: {address}")

    try:
        while True:
            data = client_socket.recv(1024)
            if not data:
                break
            print(f"От {address}: {data.decode()}")
            client_socket.send(data.upper())  # Echo в верхнем регистре
    finally:
        client_socket.close()
        print(f"Клиент {address} отключился")

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
server_socket.bind(('127.0.0.1', 8080))
server_socket.listen(100)

print("Многопоточный сервер запущен на 127.0.0.1:8080")

while True:
    client_socket, address = server_socket.accept()
    # Создаём новый поток для каждого клиента
    thread = threading.Thread(target=handle_client, args=(client_socket, address))
    thread.start()
```

### 2. Мультипроцессинг (multiprocessing)

```python
import socket
import multiprocessing

def handle_client(client_socket, address):
    """Обработчик для каждого клиента в отдельном процессе"""
    print(f"Обработка клиента: {address} (PID: {multiprocessing.current_process().pid})")

    try:
        while True:
            data = client_socket.recv(1024)
            if not data:
                break
            client_socket.send(data.upper())
    finally:
        client_socket.close()

if __name__ == '__main__':
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    server_socket.bind(('127.0.0.1', 8080))
    server_socket.listen(100)

    print("Многопроцессный сервер запущен")

    while True:
        client_socket, address = server_socket.accept()
        process = multiprocessing.Process(
            target=handle_client,
            args=(client_socket, address)
        )
        process.start()
        # Важно: закрываем сокет в родительском процессе
        client_socket.close()
```

## Таймауты для блокирующих сокетов

Можно установить таймаут, чтобы операции не блокировались бесконечно:

```python
import socket

server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
server_socket.bind(('127.0.0.1', 8080))
server_socket.listen(5)

# Устанавливаем таймаут 5 секунд
server_socket.settimeout(5.0)

try:
    client_socket, address = server_socket.accept()
except socket.timeout:
    print("Таймаут: никто не подключился за 5 секунд")
```

### Таймаут для клиентского сокета

```python
import socket

client_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
client_socket.settimeout(3.0)  # 3 секунды на подключение

try:
    client_socket.connect(('127.0.0.1', 8080))
    client_socket.settimeout(10.0)  # 10 секунд на получение данных
    data = client_socket.recv(1024)
except socket.timeout:
    print("Операция превысила таймаут")
finally:
    client_socket.close()
```

## Типы сокетов

| Тип | Константа | Протокол | Описание |
|-----|-----------|----------|----------|
| TCP | `SOCK_STREAM` | TCP | Надёжная доставка, порядок сохраняется |
| UDP | `SOCK_DGRAM` | UDP | Быстрая доставка, без гарантий |

### Пример UDP-сокета

```python
import socket

# UDP сервер
udp_socket = socket.socket(socket.AF_INET, socket.SOCK_DGRAM)
udp_socket.bind(('127.0.0.1', 8080))

print("UDP сервер запущен")

while True:
    # recvfrom возвращает данные И адрес отправителя
    data, address = udp_socket.recvfrom(1024)
    print(f"Получено от {address}: {data.decode()}")

    # Отправляем ответ на адрес отправителя
    udp_socket.sendto(b"Received!", address)
```

## Лучшие практики

1. **Всегда закрывайте сокеты** - используйте `try/finally` или контекстный менеджер
2. **Устанавливайте SO_REUSEADDR** - для избежания ошибки "Address already in use"
3. **Используйте таймауты** - не позволяйте операциям висеть бесконечно
4. **Обрабатывайте исключения** - сетевые ошибки неизбежны

```python
import socket

# Использование контекстного менеджера
with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
    s.connect(('example.com', 80))
    s.send(b'GET / HTTP/1.1\r\nHost: example.com\r\n\r\n')
    response = s.recv(4096)
    print(response.decode())
# Сокет автоматически закрывается
```

## Распространённые ошибки

1. **ConnectionRefusedError** - сервер не запущен или недоступен
2. **OSError: Address already in use** - порт занят, используйте `SO_REUSEADDR`
3. **ConnectionResetError** - соединение сброшено другой стороной
4. **socket.timeout** - превышено время ожидания

## Заключение

Блокирующие сокеты просты в использовании, но имеют существенный недостаток - они не позволяют эффективно обслуживать множество клиентов. Для масштабируемых приложений нужны либо потоки/процессы, либо неблокирующие сокеты (следующая тема).

---

[prev: ./readme.md](./readme.md) | [next: ./02-telnet.md](./02-telnet.md)
