# Потоки для счетных задач

[prev: 04-event-loop-threads.md](./04-event-loop-threads.md) | [next: ../08-streams/readme.md](../08-streams/readme.md)

---

## CPU-bound vs I/O-bound задачи

Перед выбором инструмента важно понимать тип задачи:

| Тип | Характеристика | Примеры |
|-----|----------------|---------|
| **I/O-bound** | Ожидание внешних ресурсов | Сетевые запросы, файлы, БД |
| **CPU-bound** | Интенсивные вычисления | Обработка данных, криптография |

## Проблема GIL для CPU-bound задач

**GIL (Global Interpreter Lock)** не позволяет Python-потокам выполнять байткод параллельно:

```python
import threading
import time

def cpu_intensive(n: int) -> int:
    """Вычислительно сложная функция."""
    result = 0
    for i in range(n):
        result += i ** 2
    return result

# Последовательное выполнение
start = time.time()
result1 = cpu_intensive(10_000_000)
result2 = cpu_intensive(10_000_000)
sequential_time = time.time() - start
print(f"Последовательно: {sequential_time:.2f}с")

# Параллельное выполнение с потоками
start = time.time()
t1 = threading.Thread(target=cpu_intensive, args=(10_000_000,))
t2 = threading.Thread(target=cpu_intensive, args=(10_000_000,))

t1.start()
t2.start()
t1.join()
t2.join()

threaded_time = time.time() - start
print(f"С потоками: {threaded_time:.2f}с")

# Результат: потоки НЕ ускоряют CPU-bound задачи из-за GIL!
```

## Решение: multiprocessing

Для CPU-bound задач используйте процессы вместо потоков:

```python
import multiprocessing
import time

def cpu_intensive(n: int) -> int:
    """Вычислительно сложная функция."""
    result = 0
    for i in range(n):
        result += i ** 2
    return result

if __name__ == "__main__":
    # С процессами
    start = time.time()

    with multiprocessing.Pool(processes=4) as pool:
        results = pool.map(cpu_intensive, [10_000_000] * 4)

    multiprocess_time = time.time() - start
    print(f"С процессами: {multiprocess_time:.2f}с")
    print(f"Результаты: {results}")
```

## ProcessPoolExecutor

Более современный API через `concurrent.futures`:

```python
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor
import time

def heavy_computation(x: int) -> int:
    """Тяжёлое вычисление."""
    return sum(i * i for i in range(x))

def compare_executors():
    """Сравнение Thread и Process executors."""
    tasks = [5_000_000] * 4

    # ThreadPoolExecutor (неэффективен для CPU-bound)
    start = time.time()
    with ThreadPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(heavy_computation, tasks))
    thread_time = time.time() - start
    print(f"ThreadPool: {thread_time:.2f}с")

    # ProcessPoolExecutor (эффективен для CPU-bound)
    start = time.time()
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = list(executor.map(heavy_computation, tasks))
    process_time = time.time() - start
    print(f"ProcessPool: {process_time:.2f}с")

    print(f"Ускорение: {thread_time / process_time:.2f}x")

if __name__ == "__main__":
    compare_executors()
```

## Интеграция с asyncio

Использование ProcessPoolExecutor в асинхронном коде:

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import time

def cpu_task(n: int) -> int:
    """CPU-интенсивная задача."""
    return sum(i ** 2 for i in range(n))

async def process_data(executor: ProcessPoolExecutor):
    """Асинхронная обработка с использованием процессов."""
    loop = asyncio.get_running_loop()

    # Запускаем CPU-bound задачи в процессах
    tasks = [
        loop.run_in_executor(executor, cpu_task, 5_000_000),
        loop.run_in_executor(executor, cpu_task, 5_000_000),
        loop.run_in_executor(executor, cpu_task, 5_000_000),
        loop.run_in_executor(executor, cpu_task, 5_000_000),
    ]

    results = await asyncio.gather(*tasks)
    return results

async def main():
    # Важно: ProcessPoolExecutor нужно создавать в main процессе
    with ProcessPoolExecutor(max_workers=4) as executor:
        start = time.time()
        results = await process_data(executor)
        elapsed = time.time() - start

        print(f"Результаты: {results}")
        print(f"Время: {elapsed:.2f}с")

if __name__ == "__main__":
    asyncio.run(main())
```

## Когда потоки всё же полезны для CPU-bound

### 1. Расширения на C/C++ (NumPy, pandas)

Библиотеки, написанные на C, могут освобождать GIL:

```python
import threading
import numpy as np
import time

def numpy_computation(size: int) -> np.ndarray:
    """NumPy-вычисления — освобождают GIL."""
    arr = np.random.random((size, size))
    # Эти операции выполняются в C и освобождают GIL
    result = np.dot(arr, arr.T)
    return result

def test_numpy_threading():
    """NumPy эффективно использует потоки."""
    size = 2000

    # Последовательно
    start = time.time()
    numpy_computation(size)
    numpy_computation(size)
    seq_time = time.time() - start
    print(f"Последовательно: {seq_time:.2f}с")

    # С потоками
    start = time.time()
    t1 = threading.Thread(target=numpy_computation, args=(size,))
    t2 = threading.Thread(target=numpy_computation, args=(size,))
    t1.start()
    t2.start()
    t1.join()
    t2.join()
    thread_time = time.time() - start
    print(f"С потоками: {thread_time:.2f}с")
    print(f"Ускорение: {seq_time / thread_time:.2f}x")

test_numpy_threading()
```

### 2. Комбинированные I/O + CPU задачи

```python
import asyncio
import threading
from concurrent.futures import ThreadPoolExecutor, ProcessPoolExecutor
import time

def download_data(url: str) -> bytes:
    """Симуляция загрузки (I/O-bound)."""
    time.sleep(0.5)  # Сетевой запрос
    return b"data" * 1000

def process_data(data: bytes) -> int:
    """Обработка данных (CPU-bound)."""
    return sum(b ** 2 for b in data)

async def pipeline():
    """Конвейер: загрузка (потоки) + обработка (процессы)."""
    loop = asyncio.get_running_loop()

    urls = [f"https://api.example.com/data/{i}" for i in range(10)]

    with ThreadPoolExecutor(max_workers=5) as thread_pool, \
         ProcessPoolExecutor(max_workers=4) as process_pool:

        # Шаг 1: Параллельная загрузка в потоках
        download_tasks = [
            loop.run_in_executor(thread_pool, download_data, url)
            for url in urls
        ]
        raw_data = await asyncio.gather(*download_tasks)
        print(f"Загружено {len(raw_data)} файлов")

        # Шаг 2: Параллельная обработка в процессах
        process_tasks = [
            loop.run_in_executor(process_pool, process_data, data)
            for data in raw_data
        ]
        results = await asyncio.gather(*process_tasks)
        print(f"Обработано: {results}")

if __name__ == "__main__":
    asyncio.run(pipeline())
```

## Передача данных между процессами

### Ограничения

Данные между процессами передаются через сериализацию (pickle):

```python
import multiprocessing
import numpy as np

def worker(data: np.ndarray) -> np.ndarray:
    """Обработка массива."""
    return data ** 2

if __name__ == "__main__":
    # Большой массив
    arr = np.random.random((10000, 10000))

    with multiprocessing.Pool(4) as pool:
        # Копирование данных в каждый процесс — затратно!
        # Для больших данных используйте shared memory
        pass
```

### Shared Memory (Python 3.8+)

```python
from multiprocessing import shared_memory
import numpy as np

def create_shared_array():
    """Создание массива в разделяемой памяти."""
    # Создаём обычный массив
    arr = np.array([1, 2, 3, 4, 5], dtype=np.float64)

    # Создаём shared memory
    shm = shared_memory.SharedMemory(create=True, size=arr.nbytes)

    # Создаём numpy array, использующий shared memory
    shared_arr = np.ndarray(arr.shape, dtype=arr.dtype, buffer=shm.buf)
    shared_arr[:] = arr[:]

    return shm.name, arr.shape, arr.dtype

def read_shared_array(name: str, shape: tuple, dtype):
    """Чтение массива из разделяемой памяти."""
    # Подключаемся к существующей shared memory
    shm = shared_memory.SharedMemory(name=name)

    # Создаём numpy array
    arr = np.ndarray(shape, dtype=dtype, buffer=shm.buf)
    print(f"Прочитано: {arr}")

    # Закрываем (но не удаляем!)
    shm.close()

if __name__ == "__main__":
    name, shape, dtype = create_shared_array()
    read_shared_array(name, shape, dtype)

    # Очистка
    shm = shared_memory.SharedMemory(name=name)
    shm.close()
    shm.unlink()
```

## Паттерн: Worker Pool

```python
import multiprocessing
from multiprocessing import Queue
import time

def worker(task_queue: Queue, result_queue: Queue):
    """Рабочий процесс."""
    while True:
        task = task_queue.get()
        if task is None:  # Poison pill
            break

        task_id, data = task
        # Обработка
        result = sum(x ** 2 for x in data)
        result_queue.put((task_id, result))

def main():
    num_workers = 4
    task_queue = Queue()
    result_queue = Queue()

    # Запуск workers
    workers = []
    for _ in range(num_workers):
        p = multiprocessing.Process(target=worker, args=(task_queue, result_queue))
        p.start()
        workers.append(p)

    # Отправка задач
    tasks = [(i, range(100000)) for i in range(10)]
    for task in tasks:
        task_queue.put(task)

    # Отправка poison pills
    for _ in range(num_workers):
        task_queue.put(None)

    # Сбор результатов
    results = {}
    for _ in range(len(tasks)):
        task_id, result = result_queue.get()
        results[task_id] = result

    # Ожидание завершения
    for p in workers:
        p.join()

    print(f"Результаты: {results}")

if __name__ == "__main__":
    main()
```

## Сравнение подходов

| Подход | CPU-bound | I/O-bound | Overhead | Сложность |
|--------|-----------|-----------|----------|-----------|
| threading | Плохо (GIL) | Хорошо | Низкий | Низкая |
| multiprocessing | Хорошо | Средне | Высокий | Средняя |
| asyncio | Плохо | Отлично | Низкий | Средняя |
| ProcessPoolExecutor | Хорошо | Средне | Средний | Низкая |

## Best Practices

1. **Используйте процессы для CPU-bound задач** — избегаете GIL
2. **Минимизируйте передачу данных** между процессами — используйте shared memory
3. **Комбинируйте подходы** — потоки для I/O, процессы для CPU
4. **Учитывайте overhead** создания процессов — для мелких задач он может превысить выигрыш
5. **Профилируйте код** — определите реальный bottleneck

```python
# Рекомендуемый паттерн для смешанных нагрузок
import asyncio
from concurrent.futures import ProcessPoolExecutor, ThreadPoolExecutor

class HybridExecutor:
    """Гибридный executor для разных типов задач."""

    def __init__(self, io_workers: int = 10, cpu_workers: int = None):
        self.thread_pool = ThreadPoolExecutor(max_workers=io_workers)
        self.process_pool = ProcessPoolExecutor(max_workers=cpu_workers)

    async def run_io(self, func, *args):
        """Выполнить I/O-bound задачу."""
        loop = asyncio.get_running_loop()
        return await loop.run_in_executor(self.thread_pool, func, *args)

    async def run_cpu(self, func, *args):
        """Выполнить CPU-bound задачу."""
        loop = asyncio.get_running_loop()
        return await loop.run_in_executor(self.process_pool, func, *args)

    def shutdown(self):
        self.thread_pool.shutdown(wait=True)
        self.process_pool.shutdown(wait=True)
```

## Распространённые ошибки

1. **Использование потоков для CPU-bound** — не даёт ускорения из-за GIL
2. **Передача больших объектов между процессами** — высокий overhead сериализации
3. **Забыть `if __name__ == "__main__"`** — ошибки при создании процессов на Windows
4. **Не закрывать пулы** — утечка ресурсов
5. **Игнорирование pickling ограничений** — не все объекты можно передать между процессами

```python
# Объекты, которые нельзя передать в процесс
import multiprocessing

# Lambda-функции
# func = lambda x: x ** 2  # Нельзя pickle

# Лучше использовать обычные функции
def square(x):
    return x ** 2

# Или functools.partial
from functools import partial

def power(x, n):
    return x ** n

square = partial(power, n=2)  # Можно pickle
```

---

[prev: 04-event-loop-threads.md](./04-event-loop-threads.md) | [next: ../08-streams/readme.md](../08-streams/readme.md)
