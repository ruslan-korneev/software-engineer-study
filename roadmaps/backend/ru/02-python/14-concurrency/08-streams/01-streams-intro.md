# Введение в потоки данных

[prev: readme.md](./readme.md) | [next: 02-transports-protocols.md](./02-transports-protocols.md)

---

## Что такое Streams в asyncio?

**Streams (потоки данных)** - это высокоуровневый API в asyncio для работы с сетевыми соединениями. Они предоставляют простой и удобный интерфейс для асинхронного чтения и записи данных, абстрагируя сложные детали работы с сокетами и буферами.

Streams API построен поверх низкоуровневого API транспортов и протоколов, но значительно проще в использовании. Вместо того чтобы создавать собственные классы протоколов, вы работаете с двумя основными объектами:
- **StreamReader** - для чтения данных
- **StreamWriter** - для записи данных

## Зачем нужны Streams?

### Проблема низкоуровневого подхода

При работе с сокетами напрямую приходится:
- Управлять буферами вручную
- Обрабатывать частичное чтение данных
- Следить за состоянием соединения
- Писать много шаблонного кода

### Решение с помощью Streams

Streams предоставляют:
- Автоматическую буферизацию
- Удобные методы чтения (по строкам, по количеству байт, до разделителя)
- Простую обработку соединений
- Встроенную поддержку SSL/TLS

## Основные функции для создания Streams

### asyncio.open_connection()

Создает клиентское TCP-соединение и возвращает пару (reader, writer):

```python
import asyncio

async def tcp_client():
    # Устанавливаем соединение с сервером
    reader, writer = await asyncio.open_connection('example.com', 80)

    # Отправляем HTTP-запрос
    request = b'GET / HTTP/1.1\r\nHost: example.com\r\n\r\n'
    writer.write(request)
    await writer.drain()  # Ждем, пока данные будут отправлены

    # Читаем ответ
    response = await reader.read(1024)
    print(f'Получено: {response[:100]}...')

    # Закрываем соединение
    writer.close()
    await writer.wait_closed()

asyncio.run(tcp_client())
```

### asyncio.start_server()

Создает TCP-сервер, который принимает входящие соединения:

```python
import asyncio

async def handle_client(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Обработчик клиентского соединения."""
    addr = writer.get_extra_info('peername')
    print(f'Новое соединение от {addr}')

    # Читаем данные от клиента
    data = await reader.read(100)
    message = data.decode()
    print(f'Получено: {message!r}')

    # Отправляем ответ
    response = f'Эхо: {message}'
    writer.write(response.encode())
    await writer.drain()

    # Закрываем соединение
    writer.close()
    await writer.wait_closed()
    print(f'Соединение с {addr} закрыто')

async def main():
    server = await asyncio.start_server(handle_client, '127.0.0.1', 8888)

    addr = server.sockets[0].getsockname()
    print(f'Сервер запущен на {addr}')

    async with server:
        await server.serve_forever()

asyncio.run(main())
```

## Основные параметры функций

### open_connection()

```python
asyncio.open_connection(
    host=None,          # Хост для подключения
    port=None,          # Порт
    *,
    limit=2**16,        # Размер буфера чтения (64 КБ по умолчанию)
    ssl=None,           # SSL контекст или True для SSL
    family=0,           # Семейство адресов (AF_INET, AF_INET6)
    proto=0,            # Протокол
    flags=0,            # Флаги getaddrinfo
    sock=None,          # Существующий сокет
    local_addr=None,    # Локальный адрес (host, port)
    server_hostname=None,  # Имя хоста для SSL
    ssl_handshake_timeout=None,  # Таймаут SSL рукопожатия
    happy_eyeballs_delay=None,   # Задержка Happy Eyeballs
    interleave=None     # Чередование адресов
)
```

### start_server()

```python
asyncio.start_server(
    client_connected_cb,  # Callback-функция для обработки соединений
    host=None,            # Хост для прослушивания
    port=None,            # Порт
    *,
    limit=2**16,          # Размер буфера чтения
    family=socket.AF_UNSPEC,
    flags=socket.AI_PASSIVE,
    sock=None,            # Существующий сокет
    backlog=100,          # Очередь ожидающих соединений
    ssl=None,             # SSL контекст
    reuse_address=None,   # Переиспользование адреса
    reuse_port=None,      # Переиспользование порта
    ssl_handshake_timeout=None,
    start_serving=True    # Начать слушать сразу
)
```

## Пример: простой эхо-сервер и клиент

### Сервер

```python
import asyncio

async def handle_echo(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    while True:
        data = await reader.readline()
        if not data:
            break

        message = data.decode().strip()
        print(f'Получено: {message}')

        writer.write(f'Эхо: {message}\n'.encode())
        await writer.drain()

    writer.close()
    await writer.wait_closed()

async def main():
    server = await asyncio.start_server(handle_echo, '127.0.0.1', 8888)
    print('Эхо-сервер запущен на порту 8888')
    async with server:
        await server.serve_forever()

asyncio.run(main())
```

### Клиент

```python
import asyncio

async def send_messages():
    reader, writer = await asyncio.open_connection('127.0.0.1', 8888)

    messages = ['Привет', 'Как дела?', 'Пока']

    for msg in messages:
        writer.write(f'{msg}\n'.encode())
        await writer.drain()

        response = await reader.readline()
        print(f'Сервер ответил: {response.decode().strip()}')

    writer.close()
    await writer.wait_closed()

asyncio.run(send_messages())
```

## Преимущества Streams API

| Преимущество | Описание |
|--------------|----------|
| Простота | Минимум кода для создания сетевых приложений |
| Буферизация | Автоматическое управление буферами чтения/записи |
| Гибкость чтения | Чтение по строкам, байтам, до разделителя |
| Поддержка SSL | Встроенная поддержка шифрования |
| Обратное давление | Метод `drain()` для управления потоком данных |

## Типичные ошибки

### 1. Забыть вызвать drain()

```python
# Неправильно - данные могут не быть отправлены
writer.write(data)

# Правильно - ждем отправки данных
writer.write(data)
await writer.drain()
```

### 2. Не закрывать соединение

```python
# Неправильно - ресурсы не освобождаются
writer.close()

# Правильно - дожидаемся полного закрытия
writer.close()
await writer.wait_closed()
```

### 3. Не проверять конец данных

```python
# Неправильно - бесконечный цикл при закрытии соединения
while True:
    data = await reader.read(100)
    process(data)

# Правильно - проверяем, что данные получены
while True:
    data = await reader.read(100)
    if not data:
        break
    process(data)
```

## Когда использовать Streams

- **Используйте Streams** для большинства сетевых задач: клиенты, серверы, прокси
- **Используйте низкоуровневый API** когда нужен полный контроль над протоколом или максимальная производительность
- **Используйте готовые библиотеки** (aiohttp, httpx) для HTTP и других стандартных протоколов

---

[prev: readme.md](./readme.md) | [next: 02-transports-protocols.md](./02-transports-protocols.md)
