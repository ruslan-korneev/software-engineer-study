# Создание подпроцесса

[prev: ./readme.md](./readme.md) | [next: ./02-subprocess-communication.md](./02-subprocess-communication.md)

---

## Введение

Модуль `asyncio` предоставляет мощные инструменты для асинхронного управления подпроцессами. В отличие от стандартного модуля `subprocess`, asyncio позволяет запускать и управлять внешними процессами без блокировки основного потока выполнения, что особенно важно для высоконагруженных приложений.

## Основные функции создания подпроцессов

### asyncio.create_subprocess_exec()

Функция `create_subprocess_exec()` используется для запуска программы с явным указанием аргументов командной строки. Это наиболее безопасный способ запуска подпроцессов.

```python
import asyncio

async def run_command():
    # Создание подпроцесса с явными аргументами
    process = await asyncio.create_subprocess_exec(
        'ls', '-la', '/tmp',
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )

    # Ожидание завершения и получение вывода
    stdout, stderr = await process.communicate()

    print(f'Код возврата: {process.returncode}')
    print(f'Вывод: {stdout.decode()}')

    if stderr:
        print(f'Ошибки: {stderr.decode()}')

asyncio.run(run_command())
```

### asyncio.create_subprocess_shell()

Функция `create_subprocess_shell()` запускает команду через системную оболочку (shell). Это позволяет использовать переменные окружения, конвейеры и другие возможности shell.

```python
import asyncio

async def run_shell_command():
    # Запуск команды через shell
    process = await asyncio.create_subprocess_shell(
        'echo "Текущая дата:" && date',
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )

    stdout, stderr = await process.communicate()
    print(stdout.decode())

asyncio.run(run_shell_command())
```

> **Внимание!** Использование `create_subprocess_shell()` с пользовательским вводом может привести к уязвимостям shell injection. Всегда предпочитайте `create_subprocess_exec()`, когда это возможно.

## Параметры создания подпроцессов

### Перенаправление потоков

При создании подпроцесса можно настроить перенаправление стандартных потоков:

```python
import asyncio

async def redirect_streams():
    # PIPE - создает канал для чтения/записи
    # DEVNULL - отбрасывает вывод
    # STDOUT - перенаправляет stderr в stdout

    process = await asyncio.create_subprocess_exec(
        'python', '-c', 'print("stdout"); import sys; print("stderr", file=sys.stderr)',
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.STDOUT  # stderr -> stdout
    )

    stdout, _ = await process.communicate()
    print(f'Объединённый вывод: {stdout.decode()}')

asyncio.run(redirect_streams())
```

### Настройка рабочей директории и окружения

```python
import asyncio
import os

async def custom_environment():
    # Создание кастомного окружения
    env = os.environ.copy()
    env['MY_VARIABLE'] = 'my_value'

    process = await asyncio.create_subprocess_exec(
        'python', '-c', 'import os; print(os.environ.get("MY_VARIABLE"))',
        stdout=asyncio.subprocess.PIPE,
        env=env,
        cwd='/tmp'  # Рабочая директория
    )

    stdout, _ = await process.communicate()
    print(f'Значение переменной: {stdout.decode().strip()}')

asyncio.run(custom_environment())
```

## Объект Process

После создания подпроцесса возвращается объект `asyncio.subprocess.Process`, который предоставляет:

```python
import asyncio

async def process_attributes():
    process = await asyncio.create_subprocess_exec(
        'sleep', '1',
        stdout=asyncio.subprocess.PIPE
    )

    # Атрибуты объекта Process
    print(f'PID процесса: {process.pid}')

    # Ожидание завершения
    await process.wait()

    # Код возврата (доступен после завершения)
    print(f'Код возврата: {process.returncode}')

asyncio.run(process_attributes())
```

### Основные методы Process

| Метод | Описание |
|-------|----------|
| `wait()` | Ожидает завершения процесса |
| `communicate(input)` | Отправляет данные и получает вывод |
| `send_signal(signal)` | Отправляет сигнал процессу |
| `terminate()` | Отправляет SIGTERM |
| `kill()` | Отправляет SIGKILL |

## Параллельный запуск подпроцессов

Одно из главных преимуществ asyncio - возможность запускать несколько подпроцессов параллельно:

```python
import asyncio
import time

async def run_task(name: str, duration: int) -> str:
    """Запускает команду sleep и возвращает результат."""
    process = await asyncio.create_subprocess_exec(
        'sleep', str(duration),
        stdout=asyncio.subprocess.PIPE
    )
    await process.wait()
    return f'Задача {name} завершена (sleep {duration}s)'

async def run_parallel():
    start = time.time()

    # Запуск нескольких подпроцессов параллельно
    tasks = [
        run_task('A', 2),
        run_task('B', 2),
        run_task('C', 2),
    ]

    results = await asyncio.gather(*tasks)

    elapsed = time.time() - start
    print(f'Общее время: {elapsed:.2f}s')  # ~2 секунды, а не 6

    for result in results:
        print(result)

asyncio.run(run_parallel())
```

## Таймауты для подпроцессов

Важно устанавливать таймауты для предотвращения зависания программы:

```python
import asyncio

async def run_with_timeout():
    process = await asyncio.create_subprocess_exec(
        'sleep', '10',
        stdout=asyncio.subprocess.PIPE
    )

    try:
        # Ожидание с таймаутом
        await asyncio.wait_for(process.wait(), timeout=2.0)
        print('Процесс завершился вовремя')
    except asyncio.TimeoutError:
        print('Таймаут! Завершаем процесс...')
        process.terminate()

        # Даём время на graceful shutdown
        try:
            await asyncio.wait_for(process.wait(), timeout=1.0)
        except asyncio.TimeoutError:
            print('Принудительное завершение...')
            process.kill()
            await process.wait()

asyncio.run(run_with_timeout())
```

## Практический пример: Параллельное выполнение команд

```python
import asyncio
from typing import List, Tuple

async def execute_command(cmd: str) -> Tuple[str, int, str, str]:
    """
    Выполняет команду и возвращает результат.

    Returns:
        Tuple[команда, код_возврата, stdout, stderr]
    """
    process = await asyncio.create_subprocess_shell(
        cmd,
        stdout=asyncio.subprocess.PIPE,
        stderr=asyncio.subprocess.PIPE
    )

    stdout, stderr = await process.communicate()

    return (
        cmd,
        process.returncode,
        stdout.decode().strip(),
        stderr.decode().strip()
    )

async def run_commands_parallel(commands: List[str]):
    """Выполняет список команд параллельно."""
    tasks = [execute_command(cmd) for cmd in commands]
    results = await asyncio.gather(*tasks, return_exceptions=True)

    for result in results:
        if isinstance(result, Exception):
            print(f'Ошибка: {result}')
        else:
            cmd, code, stdout, stderr = result
            print(f'Команда: {cmd}')
            print(f'Код возврата: {code}')
            if stdout:
                print(f'Вывод: {stdout}')
            if stderr:
                print(f'Ошибки: {stderr}')
            print('-' * 40)

# Пример использования
commands = [
    'echo "Hello"',
    'python --version',
    'date',
    'whoami'
]

asyncio.run(run_commands_parallel(commands))
```

## Best Practices

1. **Предпочитайте `create_subprocess_exec()`** вместо `create_subprocess_shell()` для безопасности
2. **Всегда используйте таймауты** для предотвращения зависания
3. **Обрабатывайте код возврата** - `returncode != 0` обычно означает ошибку
4. **Закрывайте потоки** после использования
5. **Используйте `communicate()`** вместо прямого чтения из pipe для избежания deadlock
6. **Ограничивайте количество параллельных процессов** с помощью `asyncio.Semaphore`

## Частые ошибки

### Deadlock при чтении из pipe

```python
# НЕПРАВИЛЬНО - может привести к deadlock
async def bad_example():
    process = await asyncio.create_subprocess_exec(
        'cat', '/large/file',
        stdout=asyncio.subprocess.PIPE
    )
    await process.wait()  # Deadlock! Буфер переполнится
    stdout = await process.stdout.read()

# ПРАВИЛЬНО - используйте communicate()
async def good_example():
    process = await asyncio.create_subprocess_exec(
        'cat', '/large/file',
        stdout=asyncio.subprocess.PIPE
    )
    stdout, stderr = await process.communicate()
```

### Забытый await

```python
# НЕПРАВИЛЬНО
process = asyncio.create_subprocess_exec('ls')  # Возвращает coroutine!

# ПРАВИЛЬНО
process = await asyncio.create_subprocess_exec('ls')
```

---

[prev: ./readme.md](./readme.md) | [next: ./02-subprocess-communication.md](./02-subprocess-communication.md)
