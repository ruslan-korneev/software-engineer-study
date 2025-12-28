# Транспортные механизмы и протоколы

[prev: 01-streams-intro.md](./01-streams-intro.md) | [next: 03-stream-readers-writers.md](./03-stream-readers-writers.md)

---

## Низкоуровневый API asyncio

Asyncio предоставляет два уровня API для работы с сетью:
- **Высокоуровневый**: Streams (StreamReader/StreamWriter) - простой и удобный
- **Низкоуровневый**: Transports и Protocols - гибкий и производительный

Транспорты и протоколы - это основа, на которой построены Streams. Понимание этого уровня помогает:
- Писать более эффективный код
- Создавать собственные протоколы
- Отлаживать проблемы с сетью

## Транспорты (Transports)

**Transport** - это абстракция, представляющая канал связи. Транспорт отвечает за:
- Отправку данных по сети
- Получение данных из сети
- Управление соединением (открытие, закрытие)
- Управление потоком данных (flow control)

### Типы транспортов

| Тип | Описание | Класс |
|-----|----------|-------|
| TCP | Потоковый транспорт для TCP-соединений | `Transport` |
| UDP | Датаграммный транспорт для UDP | `DatagramTransport` |
| SSL | Транспорт с шифрованием | `Transport` |
| Subprocess | Транспорт для работы с процессами | `SubprocessTransport` |

### Методы Transport

```python
class Transport:
    def write(self, data: bytes) -> None:
        """Записывает данные в транспорт (неблокирующий)."""
        pass

    def writelines(self, list_of_data: List[bytes]) -> None:
        """Записывает список байтовых объектов."""
        pass

    def write_eof(self) -> None:
        """Закрывает запись (отправляет EOF)."""
        pass

    def can_write_eof(self) -> bool:
        """Проверяет, поддерживает ли транспорт write_eof()."""
        pass

    def close(self) -> None:
        """Закрывает транспорт."""
        pass

    def is_closing(self) -> bool:
        """Проверяет, закрывается ли транспорт."""
        pass

    def get_extra_info(self, name: str, default=None):
        """Получает дополнительную информацию о транспорте."""
        pass

    def set_write_buffer_limits(self, high=None, low=None) -> None:
        """Устанавливает лимиты буфера записи."""
        pass

    def get_write_buffer_size(self) -> int:
        """Возвращает текущий размер буфера записи."""
        pass

    def pause_reading(self) -> None:
        """Приостанавливает чтение данных."""
        pass

    def resume_reading(self) -> None:
        """Возобновляет чтение данных."""
        pass
```

## Протоколы (Protocols)

**Protocol** - это класс, который обрабатывает события транспорта. Вы переопределяете методы протокола для реакции на события:

### Базовые методы Protocol

```python
import asyncio

class MyProtocol(asyncio.Protocol):
    def connection_made(self, transport: asyncio.Transport) -> None:
        """Вызывается при установке соединения."""
        self.transport = transport
        print('Соединение установлено')

    def data_received(self, data: bytes) -> None:
        """Вызывается при получении данных."""
        print(f'Получено: {data}')

    def eof_received(self) -> bool:
        """Вызывается при получении EOF."""
        print('EOF получен')
        return False  # False = закрыть соединение

    def connection_lost(self, exc: Exception) -> None:
        """Вызывается при потере соединения."""
        if exc:
            print(f'Соединение потеряно с ошибкой: {exc}')
        else:
            print('Соединение закрыто')
```

### Жизненный цикл протокола

```
1. connection_made()    - соединение установлено
        |
        v
2. data_received()      - получены данные (может вызываться многократно)
        |
        v
3. eof_received()       - получен EOF (опционально)
        |
        v
4. connection_lost()    - соединение закрыто
```

## Пример: Эхо-сервер на протоколах

```python
import asyncio

class EchoServerProtocol(asyncio.Protocol):
    """Протокол эхо-сервера."""

    def connection_made(self, transport: asyncio.Transport):
        self.transport = transport
        self.peername = transport.get_extra_info('peername')
        print(f'Новое соединение от {self.peername}')

    def data_received(self, data: bytes):
        message = data.decode()
        print(f'Получено от {self.peername}: {message!r}')

        # Эхо - отправляем данные обратно
        self.transport.write(f'Эхо: {message}'.encode())

        # Можно закрыть соединение после ответа
        # self.transport.close()

    def connection_lost(self, exc):
        print(f'Соединение с {self.peername} закрыто')


async def main():
    loop = asyncio.get_running_loop()

    # Создаем сервер с использованием протокола
    server = await loop.create_server(
        lambda: EchoServerProtocol(),  # Фабрика протоколов
        '127.0.0.1',
        8888
    )

    print('Сервер запущен на порту 8888')

    async with server:
        await server.serve_forever()


asyncio.run(main())
```

## Пример: TCP-клиент на протоколах

```python
import asyncio

class EchoClientProtocol(asyncio.Protocol):
    """Протокол эхо-клиента."""

    def __init__(self, message: str, on_connection_lost: asyncio.Future):
        self.message = message
        self.on_connection_lost = on_connection_lost

    def connection_made(self, transport: asyncio.Transport):
        self.transport = transport
        # Отправляем сообщение сразу после подключения
        transport.write(self.message.encode())
        print(f'Отправлено: {self.message!r}')

    def data_received(self, data: bytes):
        print(f'Получено: {data.decode()!r}')
        # Закрываем соединение после получения ответа
        self.transport.close()

    def connection_lost(self, exc):
        print('Соединение закрыто')
        self.on_connection_lost.set_result(True)


async def main():
    loop = asyncio.get_running_loop()

    # Future для ожидания закрытия соединения
    on_connection_lost = loop.create_future()

    # Создаем соединение с использованием протокола
    transport, protocol = await loop.create_connection(
        lambda: EchoClientProtocol('Привет, сервер!', on_connection_lost),
        '127.0.0.1',
        8888
    )

    # Ждем закрытия соединения
    await on_connection_lost


asyncio.run(main())
```

## Flow Control (Управление потоком)

При быстрой записи данных буфер может переполниться. Asyncio предоставляет механизм управления потоком:

```python
import asyncio

class FlowControlProtocol(asyncio.Protocol):
    def __init__(self):
        self.transport = None
        self.paused = False

    def connection_made(self, transport):
        self.transport = transport
        # Устанавливаем лимиты буфера
        transport.set_write_buffer_limits(high=65536, low=16384)

    def pause_writing(self):
        """Вызывается когда буфер заполнен."""
        print('Буфер заполнен, запись приостановлена')
        self.paused = True

    def resume_writing(self):
        """Вызывается когда буфер освободился."""
        print('Буфер освободился, запись возобновлена')
        self.paused = False

    async def write_data(self, data: bytes):
        """Записывает данные с учетом flow control."""
        if self.paused:
            # Ждем пока буфер освободится
            waiter = asyncio.get_running_loop().create_future()
            # ... логика ожидания
        self.transport.write(data)
```

## Датаграммные протоколы (UDP)

Для UDP используется `DatagramProtocol`:

```python
import asyncio

class EchoUDPProtocol(asyncio.DatagramProtocol):
    def connection_made(self, transport: asyncio.DatagramTransport):
        self.transport = transport

    def datagram_received(self, data: bytes, addr: tuple):
        message = data.decode()
        print(f'Получено от {addr}: {message}')
        # Отправляем ответ
        self.transport.sendto(f'Эхо: {message}'.encode(), addr)

    def error_received(self, exc):
        print(f'Ошибка: {exc}')


async def main():
    loop = asyncio.get_running_loop()

    # Создаем UDP-сервер
    transport, protocol = await loop.create_datagram_endpoint(
        lambda: EchoUDPProtocol(),
        local_addr=('127.0.0.1', 9999)
    )

    print('UDP-сервер запущен на порту 9999')

    try:
        await asyncio.sleep(3600)  # Работаем час
    finally:
        transport.close()


asyncio.run(main())
```

## Сравнение Streams и Protocols

| Аспект | Streams | Protocols |
|--------|---------|-----------|
| Сложность | Простой API | Требует понимания callback'ов |
| Буферизация | Автоматическая | Ручная |
| Чтение | Удобные методы (readline, read) | Только data_received() |
| Flow control | Через drain() | Через pause/resume_writing() |
| Производительность | Хорошая | Максимальная |
| Гибкость | Достаточная | Полная |

## Когда использовать Protocols

- **Максимальная производительность**: меньше накладных расходов
- **Нестандартные протоколы**: полный контроль над обработкой данных
- **UDP-приложения**: Streams не поддерживают UDP
- **Интеграция с существующим кодом**: если уже есть инфраструктура на callbacks

## Получение информации о соединении

```python
def connection_made(self, transport):
    # Адрес удаленной стороны
    peername = transport.get_extra_info('peername')

    # Локальный адрес
    sockname = transport.get_extra_info('sockname')

    # Сокет (для низкоуровневых операций)
    socket = transport.get_extra_info('socket')

    # SSL-информация (если используется SSL)
    ssl_object = transport.get_extra_info('ssl_object')
    cipher = transport.get_extra_info('cipher')
    peercert = transport.get_extra_info('peercert')
```

## Best Practices

1. **Не блокируйте event loop** в методах протокола
2. **Используйте буферы** для накопления неполных сообщений в data_received()
3. **Обрабатывайте исключения** в connection_lost()
4. **Следите за flow control** при высокой нагрузке
5. **Используйте Streams** если не нужна максимальная производительность

---

[prev: 01-streams-intro.md](./01-streams-intro.md) | [next: 03-stream-readers-writers.md](./03-stream-readers-writers.md)
