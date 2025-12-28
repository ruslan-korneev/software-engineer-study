# Неблокирующий ввод из командной строки

[prev: 03-stream-readers-writers.md](./03-stream-readers-writers.md) | [next: 05-servers.md](./05-servers.md)

---

## Проблема блокирующего ввода

Стандартные функции ввода в Python (`input()`, `sys.stdin.read()`) являются **блокирующими**. Это означает, что при их вызове выполнение программы останавливается до получения данных от пользователя.

В асинхронном коде это создает серьезную проблему - блокирующий вызов останавливает весь event loop, не позволяя выполняться другим задачам.

```python
import asyncio

async def problematic_input():
    """Это заблокирует весь event loop!"""
    user_input = input("Введите текст: ")  # БЛОКИРУЕТ!
    return user_input

async def background_task():
    """Эта задача не сможет выполняться во время input()."""
    while True:
        print("Tick...")
        await asyncio.sleep(1)

async def main():
    # background_task не будет работать пока пользователь не введет текст
    await asyncio.gather(
        problematic_input(),
        background_task()
    )

# asyncio.run(main())  # Не делайте так!
```

## Решение: asyncio и stdin

Asyncio позволяет добавить файловый дескриптор stdin в event loop и читать из него асинхронно.

### Базовый подход с add_reader()

```python
import asyncio
import sys

async def async_input(prompt: str = "") -> str:
    """Асинхронный ввод с использованием add_reader()."""
    loop = asyncio.get_running_loop()
    future = loop.create_future()

    if prompt:
        print(prompt, end='', flush=True)

    def on_stdin_ready():
        line = sys.stdin.readline()
        if not future.done():
            future.set_result(line.strip())
        loop.remove_reader(sys.stdin)

    loop.add_reader(sys.stdin, on_stdin_ready)

    return await future


async def main():
    print("Программа запущена. Введите что-нибудь:")

    # Запускаем фоновую задачу
    async def ticker():
        for i in range(10):
            print(f"  [Tick {i}]")
            await asyncio.sleep(1)

    ticker_task = asyncio.create_task(ticker())

    # Асинхронный ввод - не блокирует ticker!
    user_input = await async_input(">>> ")
    print(f"Вы ввели: {user_input}")

    ticker_task.cancel()
    try:
        await ticker_task
    except asyncio.CancelledError:
        pass


asyncio.run(main())
```

## Использование StreamReader для stdin

Более элегантный подход - создать StreamReader для stdin:

```python
import asyncio
import sys

async def create_stdin_reader() -> asyncio.StreamReader:
    """Создает StreamReader для стандартного ввода."""
    loop = asyncio.get_running_loop()

    # Создаем StreamReader
    reader = asyncio.StreamReader()

    # Создаем протокол, который будет наполнять reader
    protocol = asyncio.StreamReaderProtocol(reader)

    # Подключаем stdin к event loop
    await loop.connect_read_pipe(lambda: protocol, sys.stdin)

    return reader


async def main():
    print("Асинхронный ввод через StreamReader")
    print("Введите строки (Ctrl+D для выхода):")

    reader = await create_stdin_reader()

    # Фоновая задача продолжает работать
    async def background():
        count = 0
        while True:
            count += 1
            print(f"  [Background: {count}]")
            await asyncio.sleep(2)

    bg_task = asyncio.create_task(background())

    try:
        while True:
            print(">>> ", end='', flush=True)
            line = await reader.readline()

            if not line:  # EOF (Ctrl+D)
                break

            text = line.decode().strip()
            print(f"Получено: {text!r}")

            if text.lower() == 'quit':
                break

    finally:
        bg_task.cancel()


asyncio.run(main())
```

## Практический пример: Интерактивный клиент

```python
import asyncio
import sys

class AsyncConsole:
    """Асинхронная консоль для интерактивных приложений."""

    def __init__(self):
        self.reader: asyncio.StreamReader = None
        self.running = True

    async def start(self):
        """Инициализирует асинхронное чтение stdin."""
        loop = asyncio.get_running_loop()
        self.reader = asyncio.StreamReader()
        protocol = asyncio.StreamReaderProtocol(self.reader)
        await loop.connect_read_pipe(lambda: protocol, sys.stdin)

    async def readline(self, prompt: str = "") -> str:
        """Читает строку с опциональным приглашением."""
        if prompt:
            print(prompt, end='', flush=True)

        line = await self.reader.readline()
        return line.decode().strip() if line else None

    async def input_loop(self, callback):
        """Основной цикл ввода."""
        while self.running:
            line = await self.readline(">>> ")
            if line is None:
                break
            await callback(line)

    def stop(self):
        """Останавливает консоль."""
        self.running = False


async def tcp_client_with_console():
    """TCP-клиент с интерактивным вводом."""

    # Подключаемся к серверу
    try:
        reader, writer = await asyncio.open_connection('127.0.0.1', 8888)
    except ConnectionRefusedError:
        print("Не удалось подключиться к серверу")
        return

    console = AsyncConsole()
    await console.start()

    async def handle_server_messages():
        """Читает сообщения от сервера."""
        try:
            while True:
                data = await reader.readline()
                if not data:
                    print("\nСервер закрыл соединение")
                    console.stop()
                    break
                print(f"\nСервер: {data.decode().strip()}")
                print(">>> ", end='', flush=True)
        except asyncio.CancelledError:
            pass

    async def handle_user_input(line: str):
        """Обрабатывает ввод пользователя."""
        if line.lower() == 'quit':
            console.stop()
            return

        writer.write(f"{line}\n".encode())
        await writer.drain()

    # Запускаем обе задачи параллельно
    server_task = asyncio.create_task(handle_server_messages())

    try:
        await console.input_loop(handle_user_input)
    finally:
        server_task.cancel()
        writer.close()
        await writer.wait_closed()


# asyncio.run(tcp_client_with_console())
```

## Одновременный ввод и сетевые операции

```python
import asyncio
import sys

async def create_stdin_reader():
    loop = asyncio.get_running_loop()
    reader = asyncio.StreamReader()
    protocol = asyncio.StreamReaderProtocol(reader)
    await loop.connect_read_pipe(lambda: protocol, sys.stdin)
    return reader


async def chat_client(host: str, port: int):
    """Чат-клиент с асинхронным вводом."""

    # Подключаемся к серверу
    server_reader, server_writer = await asyncio.open_connection(host, port)
    stdin_reader = await create_stdin_reader()

    print(f"Подключено к {host}:{port}")
    print("Введите сообщение (или 'quit' для выхода):")

    async def read_from_server():
        """Читает сообщения от сервера."""
        while True:
            data = await server_reader.readline()
            if not data:
                return "server_closed"
            message = data.decode().strip()
            print(f"\r< {message}")
            print("> ", end='', flush=True)

    async def read_from_stdin():
        """Читает ввод пользователя."""
        while True:
            print("> ", end='', flush=True)
            line = await stdin_reader.readline()
            if not line:
                return "stdin_closed"

            message = line.decode().strip()
            if message.lower() == 'quit':
                return "user_quit"

            server_writer.write(f"{message}\n".encode())
            await server_writer.drain()

    # Запускаем обе задачи и ждем завершения любой из них
    server_task = asyncio.create_task(read_from_server())
    stdin_task = asyncio.create_task(read_from_stdin())

    done, pending = await asyncio.wait(
        [server_task, stdin_task],
        return_when=asyncio.FIRST_COMPLETED
    )

    # Отменяем оставшиеся задачи
    for task in pending:
        task.cancel()
        try:
            await task
        except asyncio.CancelledError:
            pass

    # Закрываем соединение
    server_writer.close()
    await server_writer.wait_closed()

    # Определяем причину завершения
    result = done.pop().result()
    if result == "server_closed":
        print("Сервер закрыл соединение")
    elif result == "user_quit":
        print("До свидания!")
    else:
        print("Ввод завершен")


# asyncio.run(chat_client('127.0.0.1', 8888))
```

## Использование run_in_executor для блокирующего ввода

Альтернативный подход - запустить блокирующий `input()` в отдельном потоке:

```python
import asyncio

async def async_input_executor(prompt: str = "") -> str:
    """Асинхронный ввод через executor (отдельный поток)."""
    loop = asyncio.get_running_loop()

    # Запускаем блокирующий input() в потоке
    return await loop.run_in_executor(None, input, prompt)


async def main():
    print("Ввод через executor")

    async def background():
        for i in range(20):
            print(f"  [Tick {i}]")
            await asyncio.sleep(0.5)

    bg_task = asyncio.create_task(background())

    try:
        while True:
            user_input = await async_input_executor(">>> ")
            print(f"Вы ввели: {user_input}")

            if user_input.lower() == 'quit':
                break
    finally:
        bg_task.cancel()


asyncio.run(main())
```

### Сравнение подходов

| Подход | Преимущества | Недостатки |
|--------|--------------|------------|
| `add_reader()` | Нативный asyncio, эффективный | Сложнее в реализации |
| `StreamReader` | Удобный API, буферизация | Требует настройки |
| `run_in_executor` | Простой, работает везде | Создает поток, менее эффективен |

## Платформозависимые особенности

### Windows

На Windows `add_reader()` и `connect_read_pipe()` для stdin работают только с `ProactorEventLoop` (по умолчанию в Python 3.8+).

```python
import asyncio
import sys

if sys.platform == 'win32':
    # Используем ProactorEventLoop для Windows
    asyncio.set_event_loop_policy(asyncio.WindowsProactorEventLoopPolicy())
```

### Альтернатива для Windows: msvcrt

```python
import asyncio
import sys

if sys.platform == 'win32':
    import msvcrt

    async def async_input_windows():
        """Асинхронный ввод для Windows."""
        loop = asyncio.get_running_loop()

        chars = []
        while True:
            # Проверяем, есть ли символ в буфере
            if msvcrt.kbhit():
                char = msvcrt.getwch()
                if char == '\r':  # Enter
                    print()  # Новая строка
                    return ''.join(chars)
                elif char == '\x08':  # Backspace
                    if chars:
                        chars.pop()
                        print('\b \b', end='', flush=True)
                else:
                    chars.append(char)
                    print(char, end='', flush=True)
            else:
                # Даем другим задачам возможность выполниться
                await asyncio.sleep(0.01)
```

## Полный пример: Интерактивный REPL

```python
import asyncio
import sys

class AsyncREPL:
    """Асинхронный REPL (Read-Eval-Print Loop)."""

    def __init__(self):
        self.commands = {}
        self.running = True
        self.stdin_reader = None

    def command(self, name: str):
        """Декоратор для регистрации команд."""
        def decorator(func):
            self.commands[name] = func
            return func
        return decorator

    async def setup(self):
        """Инициализация stdin reader."""
        loop = asyncio.get_running_loop()
        self.stdin_reader = asyncio.StreamReader()
        protocol = asyncio.StreamReaderProtocol(self.stdin_reader)
        await loop.connect_read_pipe(lambda: protocol, sys.stdin)

    async def readline(self) -> str:
        """Читает строку из stdin."""
        line = await self.stdin_reader.readline()
        return line.decode().strip() if line else None

    async def run(self):
        """Основной цикл REPL."""
        await self.setup()

        print("AsyncREPL v1.0")
        print("Введите 'help' для списка команд")

        while self.running:
            print(">>> ", end='', flush=True)
            line = await self.readline()

            if line is None:
                break

            if not line:
                continue

            # Разбираем команду и аргументы
            parts = line.split()
            cmd_name = parts[0].lower()
            args = parts[1:]

            if cmd_name in self.commands:
                try:
                    result = await self.commands[cmd_name](*args)
                    if result is not None:
                        print(result)
                except Exception as e:
                    print(f"Ошибка: {e}")
            else:
                print(f"Неизвестная команда: {cmd_name}")

        print("До свидания!")


async def main():
    repl = AsyncREPL()

    @repl.command('help')
    async def help_cmd():
        """Показывает справку."""
        return "Доступные команды: help, echo, sleep, quit"

    @repl.command('echo')
    async def echo_cmd(*args):
        """Повторяет аргументы."""
        return ' '.join(args)

    @repl.command('sleep')
    async def sleep_cmd(seconds='1'):
        """Асинхронная пауза."""
        await asyncio.sleep(float(seconds))
        return f"Проспал {seconds} сек"

    @repl.command('quit')
    async def quit_cmd():
        """Выход из REPL."""
        repl.running = False
        return "Выходим..."

    # Запускаем фоновую задачу параллельно с REPL
    async def background():
        count = 0
        while True:
            count += 1
            await asyncio.sleep(5)
            print(f"\n[Background: {count}]")
            print(">>> ", end='', flush=True)

    bg_task = asyncio.create_task(background())

    try:
        await repl.run()
    finally:
        bg_task.cancel()


asyncio.run(main())
```

## Best Practices

1. **Используйте StreamReader** для stdin вместо низкоуровневых callback'ов
2. **Обрабатывайте EOF** (Ctrl+D на Unix, Ctrl+Z на Windows)
3. **Не забывайте flush** при выводе prompt'а
4. **Учитывайте платформу** - на Windows может потребоваться особый подход
5. **Используйте run_in_executor** как запасной вариант для совместимости

---

[prev: 03-stream-readers-writers.md](./03-stream-readers-writers.md) | [next: 05-servers.md](./05-servers.md)
