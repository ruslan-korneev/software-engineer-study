# GIL и asyncio

[prev: ./04-processes-threads.md](./04-processes-threads.md) | [next: ./06-single-threaded.md](./06-single-threaded.md)

---

## Что такое GIL?

**GIL (Global Interpreter Lock)** - это глобальная блокировка интерпретатора в CPython, которая позволяет только одному потоку выполнять Python-байткод в любой момент времени.

```
┌─────────────────────────────────────────────────────────────────┐
│                         CPython                                  │
│                                                                  │
│    ┌─────────┐    ┌─────────┐    ┌─────────┐                    │
│    │ Поток 1 │    │ Поток 2 │    │ Поток 3 │                    │
│    └────┬────┘    └────┬────┘    └────┬────┘                    │
│         │              │              │                          │
│         ▼              ▼              ▼                          │
│    ╔═════════════════════════════════════════╗                  │
│    ║              GIL (Замок)                ║                  │
│    ║  Только один поток может владеть       ║                  │
│    ║  GIL в любой момент времени            ║                  │
│    ╚═════════════════════════════════════════╝                  │
│                         │                                        │
│                         ▼                                        │
│              ┌─────────────────────┐                            │
│              │ Python Интерпретатор│                            │
│              └─────────────────────┘                            │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Зачем нужен GIL?

### Историческая причина

GIL был введён в ранних версиях Python для:

1. **Упрощения управления памятью** - CPython использует подсчёт ссылок (reference counting)
2. **Защиты внутренних структур данных** - словарей, списков и т.д.
3. **Упрощения интеграции с C-библиотеками** - не требует потокобезопасности

### Проблема подсчёта ссылок

```python
import sys

a = []  # Создаём список, счётчик ссылок = 1
b = a   # Добавляем ссылку, счётчик = 2

print(sys.getrefcount(a))  # 3 (a, b, и аргумент функции)

# Без GIL два потока могли бы одновременно изменить счётчик:
# Поток 1: читает счётчик = 2
# Поток 2: читает счётчик = 2
# Поток 1: увеличивает до 3
# Поток 2: увеличивает до 3 (должно быть 4!)
# Результат: утечка памяти или преждевременное освобождение
```

## Как работает GIL?

### Механизм переключения

```python
import threading
import sys

# GIL освобождается каждые N байткод-инструкций
# В Python 3.2+ используется временной интервал (5ms по умолчанию)

print(f"Интервал переключения: {sys.getswitchinterval()} сек")
# Вывод: Интервал переключения: 0.005 сек

# Можно изменить интервал
sys.setswitchinterval(0.001)  # 1ms
```

### Визуализация работы GIL

```
Время →
────────────────────────────────────────────────────────►

Поток 1: ████████░░░░░░░░████████░░░░░░░░████████
Поток 2: ░░░░░░░░████████░░░░░░░░████████░░░░░░░░
Поток 3: ░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░

████ = Владеет GIL (выполняется)
░░░░ = Ожидает GIL

Только один поток выполняет Python-код в любой момент!
```

## Влияние GIL на производительность

### CPU-bound задачи

```python
import threading
import time

def cpu_intensive(n):
    """CPU-bound: вычисление суммы квадратов"""
    total = 0
    for i in range(n):
        total += i ** 2
    return total

def test_sequential():
    """Последовательное выполнение"""
    start = time.time()
    cpu_intensive(10_000_000)
    cpu_intensive(10_000_000)
    return time.time() - start

def test_threaded():
    """Многопоточное выполнение"""
    start = time.time()

    t1 = threading.Thread(target=cpu_intensive, args=(10_000_000,))
    t2 = threading.Thread(target=cpu_intensive, args=(10_000_000,))

    t1.start()
    t2.start()
    t1.join()
    t2.join()

    return time.time() - start

# Сравнение
seq_time = test_sequential()
thread_time = test_threaded()

print(f"Последовательно: {seq_time:.2f} сек")
print(f"Многопоточно: {thread_time:.2f} сек")

# Типичный результат:
# Последовательно: 2.5 сек
# Многопоточно: 2.8 сек  <-- МЕДЛЕННЕЕ из-за накладных расходов GIL!
```

### I/O-bound задачи

```python
import threading
import time
import requests

def fetch_url(url):
    """I/O-bound: HTTP запрос"""
    response = requests.get(url)
    return response.status_code

def test_io_sequential():
    """Последовательные запросы"""
    urls = ["https://httpbin.org/delay/1"] * 4
    start = time.time()

    for url in urls:
        fetch_url(url)

    return time.time() - start

def test_io_threaded():
    """Параллельные запросы"""
    urls = ["https://httpbin.org/delay/1"] * 4
    start = time.time()

    threads = [threading.Thread(target=fetch_url, args=(url,)) for url in urls]
    for t in threads:
        t.start()
    for t in threads:
        t.join()

    return time.time() - start

# Сравнение
seq_time = test_io_sequential()
thread_time = test_io_threaded()

print(f"Последовательно: {seq_time:.2f} сек")
print(f"Многопоточно: {thread_time:.2f} сек")

# Типичный результат:
# Последовательно: 4.0 сек
# Многопоточно: 1.1 сек  <-- БЫСТРЕЕ! GIL освобождается при I/O
```

## Когда GIL освобождается?

GIL автоматически освобождается при:

1. **I/O операциях** (чтение файлов, сетевые запросы)
2. **Ожидании** (time.sleep, блокирующие вызовы)
3. **Вызове C-расширений** (numpy, PIL и др.)
4. **Системных вызовах**

```python
import time
import threading

def io_operation():
    """GIL освобождается при I/O"""
    with open("file.txt", "r") as f:
        data = f.read()  # GIL освобождён
    return data

def sleep_operation():
    """GIL освобождается при sleep"""
    time.sleep(1)  # GIL освобождён

def numpy_operation():
    """GIL освобождается при работе numpy"""
    import numpy as np
    a = np.random.rand(1000, 1000)
    b = np.random.rand(1000, 1000)
    c = np.dot(a, b)  # GIL освобождён (C-код)
    return c
```

## Способы обхода GIL

### 1. Multiprocessing

```python
from multiprocessing import Pool
import os

def cpu_task(n):
    """CPU-bound в отдельном процессе"""
    pid = os.getpid()
    total = sum(i ** 2 for i in range(n))
    return pid, total

def main():
    # Каждый процесс имеет свой GIL
    with Pool(processes=4) as pool:
        results = pool.map(cpu_task, [5_000_000] * 4)

    for pid, total in results:
        print(f"Процесс {pid}: {total}")

if __name__ == "__main__":
    main()
```

### 2. C-расширения

```python
# Пример с Cython (псевдокод)
# my_module.pyx

# nogil позволяет освободить GIL
cdef int heavy_computation(int n) nogil:
    cdef int i, total = 0
    for i in range(n):
        total += i * i
    return total

def parallel_compute(int n):
    cdef int result
    with nogil:  # Освобождаем GIL
        result = heavy_computation(n)
    return result
```

### 3. Альтернативные интерпретаторы

- **PyPy** - JIT-компилятор, но тоже имеет GIL
- **Jython** - Python на JVM, без GIL
- **IronPython** - Python на .NET, без GIL
- **GraalPython** - экспериментальный, без GIL

### 4. Subinterpreters (Python 3.12+)

```python
# Python 3.12+ - экспериментальная фича
# Несколько интерпретаторов в одном процессе

import _xxsubinterpreters as interpreters

# Создаём отдельные интерпретаторы
interp1 = interpreters.create()
interp2 = interpreters.create()

# Каждый имеет свой GIL
interpreters.run_string(interp1, "print('Hello from interp1')")
interpreters.run_string(interp2, "print('Hello from interp2')")
```

## GIL и asyncio

### Почему asyncio работает с GIL?

Asyncio работает в **одном потоке**, поэтому GIL не является проблемой:

```python
import asyncio

async def task1():
    print("Задача 1 начата")
    await asyncio.sleep(1)  # Не держит GIL
    print("Задача 1 завершена")

async def task2():
    print("Задача 2 начата")
    await asyncio.sleep(1)
    print("Задача 2 завершена")

async def main():
    # Всё в одном потоке, GIL не переключается
    await asyncio.gather(task1(), task2())

asyncio.run(main())
```

### Преимущества asyncio над threading

| Аспект | Threading | Asyncio |
|--------|-----------|---------|
| GIL переключения | Да (накладные расходы) | Нет |
| Race conditions | Возможны | Контролируемы |
| Потребление памяти | ~8MB на поток | ~1KB на корутину |
| Максимум соединений | ~1000 | ~100000 |
| Сложность отладки | Высокая | Средняя |

### Комбинация asyncio и multiprocessing

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import os

def cpu_bound_task(n):
    """CPU-bound в отдельном процессе (обходит GIL)"""
    pid = os.getpid()
    result = sum(i ** 2 for i in range(n))
    return pid, result

async def main():
    loop = asyncio.get_running_loop()

    # ProcessPoolExecutor для CPU-bound
    with ProcessPoolExecutor(max_workers=4) as executor:
        # Запускаем CPU-bound задачи в процессах
        tasks = [
            loop.run_in_executor(executor, cpu_bound_task, 5_000_000)
            for _ in range(4)
        ]

        # Ждём асинхронно (не блокируем event loop)
        results = await asyncio.gather(*tasks)

    for pid, result in results:
        print(f"Процесс {pid}: {result}")

asyncio.run(main())
```

## Будущее GIL

### PEP 703: Making the GIL Optional

Python 3.13+ планирует сделать GIL опциональным:

```python
# Возможное будущее (Python 3.13+)
# python --no-gil script.py

import threading

def true_parallel_task():
    """Настоящий параллелизм без GIL"""
    result = 0
    for i in range(10_000_000):
        result += i ** 2
    return result

# Все потоки реально работают параллельно
threads = [threading.Thread(target=true_parallel_task) for _ in range(4)]
# ...
```

## Best Practices

### 1. Для CPU-bound используйте multiprocessing

```python
# ПРАВИЛЬНО
from multiprocessing import Pool

with Pool(4) as p:
    results = p.map(cpu_task, data)
```

### 2. Для I/O-bound используйте asyncio

```python
# ПРАВИЛЬНО
async def main():
    tasks = [fetch(url) for url in urls]
    results = await asyncio.gather(*tasks)
```

### 3. Избегайте threading для CPU-bound

```python
# НЕПРАВИЛЬНО - GIL не даст ускорения
threads = [threading.Thread(target=cpu_task) for _ in range(4)]

# ПРАВИЛЬНО - используйте процессы
processes = [Process(target=cpu_task) for _ in range(4)]
```

### 4. Профилируйте перед оптимизацией

```python
import cProfile

# Определите узкое место
cProfile.run('my_function()')
```

## Заключение

GIL - это особенность CPython, которая:

- **Упрощает** реализацию интерпретатора
- **Ограничивает** параллелизм для CPU-bound задач
- **Не мешает** I/O-bound задачам и asyncio
- **Можно обойти** с помощью multiprocessing

Понимание GIL критически важно для выбора правильного подхода к конкурентности в Python.

---

[prev: ./04-processes-threads.md](./04-processes-threads.md) | [next: ./06-single-threaded.md](./06-single-threaded.md)
