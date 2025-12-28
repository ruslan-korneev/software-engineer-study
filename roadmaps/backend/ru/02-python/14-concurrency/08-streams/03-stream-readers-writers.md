# Потоковые читатели и писатели

[prev: 02-transports-protocols.md](./02-transports-protocols.md) | [next: 04-non-blocking-stdin.md](./04-non-blocking-stdin.md)

---

## Обзор StreamReader и StreamWriter

**StreamReader** и **StreamWriter** - это основные классы высокоуровневого Streams API в asyncio. Они предоставляют удобный интерфейс для асинхронного чтения и записи данных.

- **StreamReader** - буферизованный асинхронный читатель данных
- **StreamWriter** - обертка над транспортом для записи данных

Эти объекты создаются автоматически при вызове `asyncio.open_connection()` или передаются в callback при `asyncio.start_server()`.

## StreamReader

### Основные методы чтения

```python
import asyncio

async def demonstrate_reader(reader: asyncio.StreamReader):
    # Читает до n байт (может вернуть меньше)
    data = await reader.read(100)

    # Читает ровно n байт (ждет пока не получит все)
    exact_data = await reader.readexactly(50)

    # Читает строку до \n (включая \n)
    line = await reader.readline()

    # Читает до разделителя (включая его)
    chunk = await reader.readuntil(b'\r\n')

    # Проверяет, достигнут ли конец потока
    if reader.at_eof():
        print('Конец потока')
```

### Метод read()

```python
async def read_example(reader: asyncio.StreamReader):
    """
    read(n=-1) - читает до n байт.
    - Если n=-1, читает до EOF
    - Если n>0, читает до n байт (может вернуть меньше)
    - Возвращает b'' при EOF
    """
    # Читаем до 1024 байт
    data = await reader.read(1024)

    if not data:
        print('Соединение закрыто')
        return

    print(f'Получено {len(data)} байт')
```

### Метод readline()

```python
async def readline_example(reader: asyncio.StreamReader):
    """
    readline() - читает одну строку.
    - Читает до символа \n (включительно)
    - Возвращает неполную строку если EOF без \n
    - Возвращает b'' при EOF
    """
    while True:
        line = await reader.readline()

        if not line:
            break

        # Удаляем \n и декодируем
        text = line.decode().strip()
        print(f'Строка: {text}')
```

### Метод readexactly()

```python
async def readexactly_example(reader: asyncio.StreamReader):
    """
    readexactly(n) - читает ровно n байт.
    - Ждет пока не получит все n байт
    - Бросает IncompleteReadError если EOF раньше
    """
    try:
        # Читаем заголовок протокола (например, 4 байта длины)
        header = await reader.readexactly(4)
        length = int.from_bytes(header, 'big')

        # Читаем тело сообщения
        body = await reader.readexactly(length)
        return body

    except asyncio.IncompleteReadError as e:
        print(f'Неполное чтение: получено {len(e.partial)} из {e.expected}')
        return e.partial
```

### Метод readuntil()

```python
async def readuntil_example(reader: asyncio.StreamReader):
    """
    readuntil(separator) - читает до разделителя.
    - Включает разделитель в результат
    - Бросает IncompleteReadError если EOF без разделителя
    - Бросает LimitOverrunError если лимит буфера превышен
    """
    try:
        # Читаем HTTP заголовок (до пустой строки)
        headers = await reader.readuntil(b'\r\n\r\n')
        return headers.decode()

    except asyncio.IncompleteReadError:
        print('Соединение закрыто до получения разделителя')
    except asyncio.LimitOverrunError:
        print('Превышен лимит буфера')
```

### Дополнительные свойства и методы

```python
async def reader_extras(reader: asyncio.StreamReader):
    # Проверяем, достигнут ли EOF
    if reader.at_eof():
        print('Конец потока')

    # Получаем исключение, если оно было установлено
    exc = reader.exception()
    if exc:
        print(f'Ошибка: {exc}')

    # Возвращаем данные в начало буфера (для повторного чтения)
    reader.feed_data(b'prepend this')

    # Сигнализируем об окончании данных
    reader.feed_eof()
```

## StreamWriter

### Основные методы записи

```python
async def demonstrate_writer(writer: asyncio.StreamWriter):
    # Записывает данные в буфер
    writer.write(b'Hello, World!')

    # Записывает список байтовых объектов
    writer.writelines([b'Line 1\n', b'Line 2\n', b'Line 3\n'])

    # Ждет пока данные будут отправлены
    await writer.drain()

    # Закрывает поток записи (отправляет EOF)
    writer.write_eof()

    # Закрывает соединение
    writer.close()
    await writer.wait_closed()
```

### Метод write() и drain()

```python
async def write_with_drain(writer: asyncio.StreamWriter):
    """
    write() - неблокирующий, добавляет данные в буфер.
    drain() - ждет пока буфер опустеет (flow control).

    ВАЖНО: Всегда вызывайте drain() после записи!
    """
    large_data = b'x' * 1_000_000  # 1 МБ данных

    # Записываем порциями с ожиданием
    chunk_size = 65536
    for i in range(0, len(large_data), chunk_size):
        chunk = large_data[i:i + chunk_size]
        writer.write(chunk)
        await writer.drain()  # Ждем отправки, чтобы не переполнить буфер

    print('Все данные отправлены')
```

### Получение информации о соединении

```python
async def writer_info(writer: asyncio.StreamWriter):
    """get_extra_info() - получает метаданные соединения."""

    # Адрес удаленной стороны (IP, порт)
    peername = writer.get_extra_info('peername')
    print(f'Клиент: {peername}')

    # Локальный адрес
    sockname = writer.get_extra_info('sockname')
    print(f'Сервер: {sockname}')

    # Сокет (для низкоуровневых операций)
    socket = writer.get_extra_info('socket')

    # Для SSL-соединений
    ssl_object = writer.get_extra_info('ssl_object')
    cipher = writer.get_extra_info('cipher')
    peercert = writer.get_extra_info('peercert')

    # Проверяем, закрывается ли соединение
    if writer.is_closing():
        print('Соединение закрывается')
```

### Правильное закрытие соединения

```python
async def proper_close(writer: asyncio.StreamWriter):
    """Правильный способ закрытия соединения."""

    # Отправляем оставшиеся данные
    await writer.drain()

    # Закрываем соединение
    writer.close()

    # ВАЖНО: дожидаемся полного закрытия
    await writer.wait_closed()

    print('Соединение корректно закрыто')
```

## Практические примеры

### Пример 1: Простой HTTP-клиент

```python
import asyncio

async def http_get(host: str, path: str = '/') -> str:
    """Выполняет простой HTTP GET запрос."""
    reader, writer = await asyncio.open_connection(host, 80)

    # Формируем HTTP запрос
    request = f'GET {path} HTTP/1.1\r\nHost: {host}\r\nConnection: close\r\n\r\n'
    writer.write(request.encode())
    await writer.drain()

    # Читаем ответ
    response = b''
    while True:
        chunk = await reader.read(4096)
        if not chunk:
            break
        response += chunk

    writer.close()
    await writer.wait_closed()

    return response.decode('utf-8', errors='replace')


async def main():
    response = await http_get('example.com')
    print(response[:500])

asyncio.run(main())
```

### Пример 2: Протокол с заголовком длины

```python
import asyncio
import struct

async def send_message(writer: asyncio.StreamWriter, message: bytes):
    """Отправляет сообщение с 4-байтовым заголовком длины."""
    # Формируем заголовок (длина сообщения в big-endian)
    header = struct.pack('>I', len(message))

    writer.write(header + message)
    await writer.drain()


async def receive_message(reader: asyncio.StreamReader) -> bytes:
    """Получает сообщение с 4-байтовым заголовком длины."""
    # Читаем заголовок
    header = await reader.readexactly(4)
    length = struct.unpack('>I', header)[0]

    # Читаем тело сообщения
    message = await reader.readexactly(length)
    return message


async def client_example():
    reader, writer = await asyncio.open_connection('127.0.0.1', 8888)

    # Отправляем сообщение
    await send_message(writer, b'Hello, Server!')

    # Получаем ответ
    response = await receive_message(reader)
    print(f'Ответ: {response.decode()}')

    writer.close()
    await writer.wait_closed()
```

### Пример 3: Построчный обмен данными

```python
import asyncio

async def line_client(host: str, port: int):
    """Клиент, работающий с построчным протоколом."""
    reader, writer = await asyncio.open_connection(host, port)

    try:
        # Отправляем команды
        commands = ['HELLO', 'STATUS', 'QUIT']

        for cmd in commands:
            # Отправляем команду
            writer.write(f'{cmd}\n'.encode())
            await writer.drain()

            # Читаем ответ
            response = await reader.readline()
            print(f'{cmd} -> {response.decode().strip()}')

    finally:
        writer.close()
        await writer.wait_closed()


async def line_server(reader: asyncio.StreamReader, writer: asyncio.StreamWriter):
    """Сервер с построчным протоколом."""
    addr = writer.get_extra_info('peername')
    print(f'Клиент подключен: {addr}')

    try:
        while True:
            line = await reader.readline()
            if not line:
                break

            command = line.decode().strip()
            print(f'Команда от {addr}: {command}')

            # Обрабатываем команды
            if command == 'HELLO':
                response = 'HELLO, Client!'
            elif command == 'STATUS':
                response = 'OK'
            elif command == 'QUIT':
                response = 'BYE'
                writer.write(f'{response}\n'.encode())
                await writer.drain()
                break
            else:
                response = f'UNKNOWN: {command}'

            writer.write(f'{response}\n'.encode())
            await writer.drain()

    finally:
        writer.close()
        await writer.wait_closed()
        print(f'Клиент отключен: {addr}')
```

## Обработка ошибок

```python
import asyncio

async def safe_read(reader: asyncio.StreamReader):
    """Безопасное чтение с обработкой всех ошибок."""
    try:
        data = await reader.readexactly(100)
        return data

    except asyncio.IncompleteReadError as e:
        # EOF до получения всех данных
        print(f'Неполное чтение: {len(e.partial)} байт из 100')
        return e.partial

    except asyncio.LimitOverrunError as e:
        # Превышен лимит буфера
        print(f'Лимит буфера превышен на {e.consumed} байт')
        return None

    except ConnectionResetError:
        # Соединение сброшено
        print('Соединение сброшено клиентом')
        return None

    except Exception as e:
        print(f'Неожиданная ошибка: {e}')
        return None
```

## Best Practices

### 1. Всегда используйте drain()

```python
# Плохо
writer.write(data)

# Хорошо
writer.write(data)
await writer.drain()
```

### 2. Правильно закрывайте соединения

```python
# Плохо
writer.close()

# Хорошо
writer.close()
await writer.wait_closed()
```

### 3. Проверяйте пустые данные

```python
async def read_loop(reader):
    while True:
        data = await reader.read(1024)
        if not data:  # Важная проверка!
            break
        process(data)
```

### 4. Используйте таймауты

```python
async def read_with_timeout(reader, timeout=30):
    try:
        return await asyncio.wait_for(reader.read(1024), timeout)
    except asyncio.TimeoutError:
        print('Таймаут чтения')
        return None
```

### 5. Используйте context manager для соединений

```python
async def safe_connection():
    reader, writer = await asyncio.open_connection('example.com', 80)
    try:
        # Работа с соединением
        pass
    finally:
        writer.close()
        await writer.wait_closed()
```

## Ограничения и лимиты

```python
async def main():
    # Установка лимита буфера при создании соединения
    reader, writer = await asyncio.open_connection(
        'example.com', 80,
        limit=2**20  # 1 МБ вместо 64 КБ по умолчанию
    )

    # При превышении лимита readuntil() бросит LimitOverrunError
```

---

[prev: 02-transports-protocols.md](./02-transports-protocols.md) | [next: 04-non-blocking-stdin.md](./04-non-blocking-stdin.md)
