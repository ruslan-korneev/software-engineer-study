# Взаимодействие с подпроцессами

[prev: ./01-create-subprocess.md](./01-create-subprocess.md) | [next: ../14-advanced/readme.md](../14-advanced/readme.md)

---

## Введение

После создания подпроцесса часто требуется организовать двустороннюю коммуникацию: отправлять данные на вход процесса и читать его вывод. Asyncio предоставляет удобные механизмы для асинхронного взаимодействия с подпроцессами через стандартные потоки ввода-вывода.

## Основные потоки коммуникации

Каждый процесс имеет три стандартных потока:
- **stdin** (standard input) - стандартный ввод
- **stdout** (standard output) - стандартный вывод
- **stderr** (standard error) - поток ошибок

```python
import asyncio

async def basic_communication():
    # Создание процесса с подключением всех потоков
    process = await asyncio.create_subprocess_exec(
        'cat',  # cat просто возвращает то, что получает на вход
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )

    # Отправка данных и получение результата
    input_data = b'Hello, subprocess!\n'
    stdout, stderr = await process.communicate(input=input_data)

    print(f'Отправлено: {input_data.decode().strip()}')
    print(f'Получено: {stdout.decode().strip()}')

asyncio.run(basic_communication())
```

## Метод communicate()

Метод `communicate()` - основной способ взаимодействия с подпроцессом. Он:
1. Отправляет данные в stdin (если указаны)
2. Читает stdout и stderr до EOF
3. Ожидает завершения процесса

```python
import asyncio

async def communicate_example():
    # Python-скрипт, который читает ввод и обрабатывает его
    python_code = '''
import sys
data = sys.stdin.read()
print(f"Получено {len(data)} байт")
print(f"Содержимое: {data.upper()}")
'''

    process = await asyncio.create_subprocess_exec(
        'python', '-c', python_code,
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )

    # Отправляем данные
    stdout, stderr = await process.communicate(b'hello world')

    print(f'Результат:\n{stdout.decode()}')
    if stderr:
        print(f'Ошибки: {stderr.decode()}')

asyncio.run(communicate_example())
```

## Потоковое чтение вывода

Для длительных процессов или обработки данных по мере поступления используется потоковое чтение:

```python
import asyncio

async def stream_reading():
    """Чтение вывода построчно в реальном времени."""

    # Процесс, который выводит данные постепенно
    process = await asyncio.create_subprocess_exec(
        'python', '-c', '''
import time
import sys
for i in range(5):
    print(f"Строка {i + 1}", flush=True)
    time.sleep(0.5)
''',
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )

    # Чтение построчно
    while True:
        line = await process.stdout.readline()
        if not line:
            break
        print(f'Получено: {line.decode().strip()}')

    await process.wait()
    print(f'Процесс завершён с кодом: {process.returncode}')

asyncio.run(stream_reading())
```

### Чтение фиксированного количества байт

```python
import asyncio

async def read_chunks():
    """Чтение данных блоками."""

    process = await asyncio.create_subprocess_exec(
        'dd', 'if=/dev/urandom', 'bs=1024', 'count=10',
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.DEVNULL
    )

    total_bytes = 0
    chunk_num = 0

    while True:
        chunk = await process.stdout.read(1024)  # Читаем по 1KB
        if not chunk:
            break
        chunk_num += 1
        total_bytes += len(chunk)
        print(f'Блок {chunk_num}: {len(chunk)} байт')

    await process.wait()
    print(f'Всего получено: {total_bytes} байт')

asyncio.run(read_chunks())
```

## Интерактивное взаимодействие

Для интерактивных процессов можно писать в stdin и читать из stdout поочерёдно:

```python
import asyncio

async def interactive_process():
    """Интерактивное взаимодействие с процессом."""

    # Python REPL в качестве интерактивного процесса
    process = await asyncio.create_subprocess_exec(
        'python', '-i', '-u',  # -u для небуферизованного вывода
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.STDOUT
    )

    async def send_command(cmd: str) -> str:
        """Отправляет команду и читает ответ."""
        process.stdin.write(f'{cmd}\n'.encode())
        await process.stdin.drain()

        # Читаем ответ (ожидаем приглашение >>>)
        response = b''
        while True:
            try:
                chunk = await asyncio.wait_for(
                    process.stdout.read(1024),
                    timeout=1.0
                )
                response += chunk
                if b'>>> ' in chunk or not chunk:
                    break
            except asyncio.TimeoutError:
                break

        return response.decode()

    # Пропускаем начальное приглашение
    await asyncio.sleep(0.5)
    await process.stdout.read(4096)

    # Отправляем команды
    print(await send_command('2 + 2'))
    print(await send_command('print("Hello!")'))

    # Завершаем процесс
    process.stdin.write(b'exit()\n')
    await process.stdin.drain()
    process.stdin.close()
    await process.wait()

asyncio.run(interactive_process())
```

## Параллельное чтение stdout и stderr

При работе с процессами, которые пишут и в stdout, и в stderr, важно читать оба потока параллельно:

```python
import asyncio

async def read_stream(stream, name: str):
    """Читает поток и выводит с меткой."""
    while True:
        line = await stream.readline()
        if not line:
            break
        print(f'[{name}] {line.decode().strip()}')

async def parallel_streams():
    """Параллельное чтение stdout и stderr."""

    python_code = '''
import sys
import time

for i in range(3):
    print(f"stdout: сообщение {i + 1}", flush=True)
    print(f"stderr: ошибка {i + 1}", file=sys.stderr, flush=True)
    time.sleep(0.3)
'''

    process = await asyncio.create_subprocess_exec(
        'python', '-c', python_code,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )

    # Создаём задачи для параллельного чтения
    await asyncio.gather(
        read_stream(process.stdout, 'OUT'),
        read_stream(process.stderr, 'ERR'),
    )

    await process.wait()

asyncio.run(parallel_streams())
```

## Конвейер (Pipeline) процессов

Можно организовать конвейер из нескольких процессов, передавая вывод одного на вход другому:

```python
import asyncio

async def pipeline():
    """Конвейер: echo | grep | wc"""

    # Первый процесс: генерация данных
    p1 = await asyncio.create_subprocess_exec(
        'echo', 'apple\nbanana\napricot\nblueberry\navocado',
        stdout=asyncio.subprocess.PIPE
    )

    # Второй процесс: фильтрация (слова на 'a')
    p2 = await asyncio.create_subprocess_exec(
        'grep', '^a',
        stdin=p1.stdout,
        stdout=asyncio.subprocess.PIPE
    )

    # Третий процесс: подсчёт строк
    p3 = await asyncio.create_subprocess_exec(
        'wc', '-l',
        stdin=p2.stdout,
        stdout=asyncio.subprocess.PIPE
    )

    # Закрываем stdin первого процесса в родителе
    # чтобы он получил EOF когда p1 завершится
    p1.stdout.close()
    p2.stdout.close()

    # Получаем результат
    stdout, _ = await p3.communicate()
    count = int(stdout.decode().strip())

    print(f'Количество слов на "a": {count}')

    # Ждём завершения всех процессов
    await p1.wait()
    await p2.wait()

asyncio.run(pipeline())
```

## Обработка сигналов

Подпроцессам можно отправлять сигналы для управления их выполнением:

```python
import asyncio
import signal

async def signal_handling():
    """Отправка сигналов подпроцессу."""

    # Долгий процесс
    process = await asyncio.create_subprocess_exec(
        'sleep', '60',
        stdout=asyncio.subprocess.PIPE
    )

    print(f'Запущен процесс с PID: {process.pid}')

    # Ждём немного и отправляем SIGTERM
    await asyncio.sleep(1)
    print('Отправляем SIGTERM...')
    process.terminate()  # Эквивалент process.send_signal(signal.SIGTERM)

    try:
        await asyncio.wait_for(process.wait(), timeout=2.0)
        print(f'Процесс завершился с кодом: {process.returncode}')
    except asyncio.TimeoutError:
        print('Процесс не отвечает, отправляем SIGKILL...')
        process.kill()  # Эквивалент process.send_signal(signal.SIGKILL)
        await process.wait()

asyncio.run(signal_handling())
```

## Практический пример: Исполнитель команд с логированием

```python
import asyncio
from dataclasses import dataclass
from datetime import datetime
from typing import Optional

@dataclass
class CommandResult:
    """Результат выполнения команды."""
    command: str
    return_code: int
    stdout: str
    stderr: str
    started_at: datetime
    finished_at: datetime

    @property
    def duration(self) -> float:
        return (self.finished_at - self.started_at).total_seconds()

    @property
    def success(self) -> bool:
        return self.return_code == 0

class AsyncCommandExecutor:
    """Асинхронный исполнитель команд с логированием."""

    def __init__(self, timeout: float = 30.0):
        self.timeout = timeout
        self.results: list[CommandResult] = []

    async def execute(
        self,
        command: str,
        input_data: Optional[bytes] = None
    ) -> CommandResult:
        """Выполняет команду и возвращает результат."""

        started_at = datetime.now()

        process = await asyncio.create_subprocess_shell(
            command,
            stdin=asyncio.subprocess.PIPE if input_data else None,
            stdout=asyncio.subprocess.PIPE,
            stderr=asyncio.subprocess.PIPE
        )

        try:
            stdout, stderr = await asyncio.wait_for(
                process.communicate(input=input_data),
                timeout=self.timeout
            )
        except asyncio.TimeoutError:
            process.kill()
            await process.wait()
            stdout, stderr = b'', b'Timeout exceeded'

        finished_at = datetime.now()

        result = CommandResult(
            command=command,
            return_code=process.returncode or -1,
            stdout=stdout.decode(errors='replace'),
            stderr=stderr.decode(errors='replace'),
            started_at=started_at,
            finished_at=finished_at
        )

        self.results.append(result)
        return result

    async def execute_many(self, commands: list[str]) -> list[CommandResult]:
        """Выполняет несколько команд параллельно."""
        tasks = [self.execute(cmd) for cmd in commands]
        return await asyncio.gather(*tasks)

    def print_summary(self):
        """Выводит сводку по выполненным командам."""
        print('\n' + '=' * 60)
        print('СВОДКА ВЫПОЛНЕНИЯ КОМАНД')
        print('=' * 60)

        for i, result in enumerate(self.results, 1):
            status = 'OK' if result.success else 'ОШИБКА'
            print(f'\n{i}. [{status}] {result.command}')
            print(f'   Время: {result.duration:.3f}s')
            print(f'   Код возврата: {result.return_code}')
            if result.stdout.strip():
                print(f'   Вывод: {result.stdout.strip()[:100]}...')
            if result.stderr.strip():
                print(f'   Ошибки: {result.stderr.strip()[:100]}')

async def main():
    executor = AsyncCommandExecutor(timeout=10.0)

    # Выполняем команды параллельно
    commands = [
        'echo "Hello, World!"',
        'python --version',
        'date "+%Y-%m-%d %H:%M:%S"',
        'ls -la /tmp | head -5',
        'sleep 1 && echo "После паузы"'
    ]

    print('Запуск команд...')
    results = await executor.execute_many(commands)

    # Выводим результаты
    for result in results:
        if result.success:
            print(f'[OK] {result.command}: {result.stdout.strip()}')
        else:
            print(f'[ERR] {result.command}: {result.stderr.strip()}')

    executor.print_summary()

asyncio.run(main())
```

## Best Practices

1. **Используйте `communicate()`** для простых случаев - это безопасно и предотвращает deadlock
2. **Читайте stdout и stderr параллельно** для избежания блокировок при большом выводе
3. **Всегда устанавливайте таймауты** при работе с внешними процессами
4. **Используйте `flush=True`** в Python-подпроцессах для немедленного вывода
5. **Закрывайте потоки** после завершения работы
6. **Обрабатывайте ошибки** - проверяйте код возврата и stderr

## Частые ошибки

### Deadlock при большом выводе

```python
# НЕПРАВИЛЬНО - может зависнуть
async def bad():
    process = await asyncio.create_subprocess_exec(
        'cat', '/very/large/file',
        stdout=asyncio.subprocess.PIPE
    )
    await process.wait()  # Буфер переполнится!
    data = await process.stdout.read()

# ПРАВИЛЬНО - читаем и ждём одновременно
async def good():
    process = await asyncio.create_subprocess_exec(
        'cat', '/very/large/file',
        stdout=asyncio.subprocess.PIPE
    )
    data, _ = await process.communicate()
```

### Потеря данных при закрытии stdin

```python
# НЕПРАВИЛЬНО - данные могут не успеть отправиться
async def bad():
    process = await asyncio.create_subprocess_exec(
        'cat',
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE
    )
    process.stdin.write(b'data')
    process.stdin.close()  # Нет await drain()!

# ПРАВИЛЬНО - дожидаемся отправки
async def good():
    process = await asyncio.create_subprocess_exec(
        'cat',
        stdin=asyncio.subprocess.PIPE,
        stdout=asyncio.subprocess.PIPE
    )
    process.stdin.write(b'data')
    await process.stdin.drain()  # Дожидаемся отправки
    process.stdin.close()
    stdout, _ = await process.communicate()
```

---

[prev: ./01-create-subprocess.md](./01-create-subprocess.md) | [next: ../14-advanced/readme.md](../14-advanced/readme.md)
