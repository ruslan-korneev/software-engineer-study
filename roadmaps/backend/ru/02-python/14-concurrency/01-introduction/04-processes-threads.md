# Процессы, потоки, многопоточность

[prev: ./03-concurrency-parallelism.md](./03-concurrency-parallelism.md) | [next: ./05-gil.md](./05-gil.md)

---

## Введение

Для понимания asyncio необходимо разобраться в базовых концепциях операционных систем: процессах и потоках. Это фундамент, на котором строится вся конкурентность.

## Процессы (Processes)

**Процесс** - это экземпляр выполняющейся программы. Каждый процесс имеет:

- Собственное адресное пространство памяти
- Системные ресурсы (файловые дескрипторы, сокеты)
- Как минимум один поток выполнения
- Уникальный идентификатор (PID)

```
┌─────────────────────────────────────────────────────────────┐
│                         ПРОЦЕСС                             │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │     Код         │  │     Стек        │                  │
│  │  программы      │  │  вызовов        │                  │
│  └─────────────────┘  └─────────────────┘                  │
│  ┌─────────────────┐  ┌─────────────────┐                  │
│  │     Куча        │  │   Данные        │                  │
│  │    (heap)       │  │   (data)        │                  │
│  └─────────────────┘  └─────────────────┘                  │
│                                                             │
│  PID: 12345                                                 │
└─────────────────────────────────────────────────────────────┘
```

### Работа с процессами в Python

```python
import os
import multiprocessing
import time

def worker(name: str):
    """Функция, выполняемая в отдельном процессе"""
    pid = os.getpid()
    ppid = os.getppid()
    print(f"Процесс {name}: PID={pid}, родитель={ppid}")
    time.sleep(2)
    return f"Результат от {name}"

def main():
    print(f"Главный процесс: PID={os.getpid()}")

    # Создание процессов
    processes = []
    for i in range(3):
        p = multiprocessing.Process(target=worker, args=(f"Worker-{i}",))
        processes.append(p)
        p.start()

    # Ожидание завершения
    for p in processes:
        p.join()

    print("Все процессы завершены")

if __name__ == "__main__":
    main()
```

### Вывод:
```
Главный процесс: PID=12345
Процесс Worker-0: PID=12346, родитель=12345
Процесс Worker-1: PID=12347, родитель=12345
Процесс Worker-2: PID=12348, родитель=12345
Все процессы завершены
```

### Pool процессов

```python
from multiprocessing import Pool
import os

def compute(x: int) -> int:
    """CPU-интенсивное вычисление"""
    pid = os.getpid()
    result = sum(i ** 2 for i in range(x))
    print(f"PID {pid}: compute({x}) = {result}")
    return result

def main():
    # Pool автоматически управляет процессами
    with Pool(processes=4) as pool:
        # map - применяет функцию к каждому элементу
        results = pool.map(compute, [100000, 200000, 300000, 400000])
        print(f"Результаты: {results}")

if __name__ == "__main__":
    main()
```

## Потоки (Threads)

**Поток** - это единица выполнения внутри процесса. Потоки одного процесса:

- Разделяют общее адресное пространство памяти
- Имеют собственный стек вызовов
- Могут выполняться конкурентно
- Дешевле в создании, чем процессы

```
┌─────────────────────────────────────────────────────────────┐
│                         ПРОЦЕСС                             │
│                                                             │
│  ┌─────────────────────────────────────────────────────┐   │
│  │              Общая память (heap, data)              │   │
│  └─────────────────────────────────────────────────────┘   │
│                                                             │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │   Поток 1   │  │   Поток 2   │  │   Поток 3   │        │
│  │   ┌─────┐   │  │   ┌─────┐   │  │   ┌─────┐   │        │
│  │   │Стек │   │  │   │Стек │   │  │   │Стек │   │        │
│  │   └─────┘   │  │   └─────┘   │  │   └─────┘   │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                             │
└─────────────────────────────────────────────────────────────┘
```

### Работа с потоками в Python

```python
import threading
import time

def worker(name: str, duration: int):
    """Функция, выполняемая в потоке"""
    thread_id = threading.current_thread().name
    print(f"[{thread_id}] Поток {name} начал работу")
    time.sleep(duration)
    print(f"[{thread_id}] Поток {name} завершил работу")

def main():
    print(f"Главный поток: {threading.current_thread().name}")

    # Создание потоков
    threads = []
    for i in range(3):
        t = threading.Thread(
            target=worker,
            args=(f"Worker-{i}", 2),
            name=f"Thread-{i}"
        )
        threads.append(t)
        t.start()

    # Ожидание завершения
    for t in threads:
        t.join()

    print("Все потоки завершены")

if __name__ == "__main__":
    main()
```

### ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor
import requests
import time

def fetch_url(url: str) -> dict:
    """Загрузка URL (I/O-bound)"""
    start = time.time()
    response = requests.get(url)
    return {
        "url": url,
        "status": response.status_code,
        "time": time.time() - start
    }

def main():
    urls = [
        "https://python.org",
        "https://pypi.org",
        "https://docs.python.org",
        "https://realpython.com"
    ]

    start = time.time()

    # ThreadPoolExecutor управляет пулом потоков
    with ThreadPoolExecutor(max_workers=4) as executor:
        # submit возвращает Future
        futures = [executor.submit(fetch_url, url) for url in urls]

        # Получаем результаты
        for future in futures:
            result = future.result()
            print(f"{result['url']}: {result['status']} ({result['time']:.2f}s)")

    print(f"Общее время: {time.time() - start:.2f}s")

if __name__ == "__main__":
    main()
```

## Сравнение процессов и потоков

| Характеристика | Процесс | Поток |
|---------------|---------|-------|
| Память | Изолированная | Общая |
| Создание | Дорогое | Дешёвое |
| Переключение контекста | Медленное | Быстрое |
| Обмен данными | Через IPC | Через общую память |
| Отказоустойчивость | Высокая | Низкая |
| GIL | Не влияет | Блокирует CPU-bound |

## Межпроцессное взаимодействие (IPC)

Процессы не разделяют память, поэтому для обмена данными используют специальные механизмы:

### Queue (Очередь)

```python
from multiprocessing import Process, Queue
import time

def producer(queue: Queue, items: list):
    """Производитель - добавляет элементы в очередь"""
    for item in items:
        print(f"Производитель: отправляю {item}")
        queue.put(item)
        time.sleep(0.5)
    queue.put(None)  # Сигнал завершения

def consumer(queue: Queue):
    """Потребитель - забирает элементы из очереди"""
    while True:
        item = queue.get()
        if item is None:
            break
        print(f"Потребитель: получил {item}")

def main():
    queue = Queue()
    items = ["яблоко", "банан", "апельсин", "груша"]

    p1 = Process(target=producer, args=(queue, items))
    p2 = Process(target=consumer, args=(queue,))

    p1.start()
    p2.start()

    p1.join()
    p2.join()

if __name__ == "__main__":
    main()
```

### Pipe (Канал)

```python
from multiprocessing import Process, Pipe

def sender(conn):
    """Отправитель данных"""
    messages = ["Привет", "Мир", "Python"]
    for msg in messages:
        conn.send(msg)
        print(f"Отправлено: {msg}")
    conn.close()

def receiver(conn):
    """Получатель данных"""
    while True:
        try:
            msg = conn.recv()
            print(f"Получено: {msg}")
        except EOFError:
            break

def main():
    # Создаём двусторонний канал
    parent_conn, child_conn = Pipe()

    p1 = Process(target=sender, args=(parent_conn,))
    p2 = Process(target=receiver, args=(child_conn,))

    p1.start()
    p2.start()

    p1.join()
    p2.join()

if __name__ == "__main__":
    main()
```

### Shared Memory (Общая память)

```python
from multiprocessing import Process, Value, Array
import time

def increment_counter(counter, lock):
    """Увеличивает счётчик с блокировкой"""
    for _ in range(10000):
        with lock:
            counter.value += 1

def fill_array(arr):
    """Заполняет массив"""
    for i in range(len(arr)):
        arr[i] = i * 2

def main():
    from multiprocessing import Lock

    # Общий счётчик (Value) с типом 'i' (int)
    counter = Value('i', 0)
    lock = Lock()

    # Общий массив (Array) с типом 'd' (double)
    shared_array = Array('d', [0.0] * 10)

    # Демонстрация общего счётчика
    processes = [
        Process(target=increment_counter, args=(counter, lock))
        for _ in range(4)
    ]

    for p in processes:
        p.start()
    for p in processes:
        p.join()

    print(f"Счётчик: {counter.value}")  # Должно быть 40000

    # Демонстрация общего массива
    p = Process(target=fill_array, args=(shared_array,))
    p.start()
    p.join()

    print(f"Массив: {list(shared_array)}")

if __name__ == "__main__":
    main()
```

## Синхронизация потоков

### Lock (Блокировка)

```python
import threading
import time

counter = 0
lock = threading.Lock()

def increment_without_lock():
    global counter
    for _ in range(100000):
        counter += 1  # Race condition!

def increment_with_lock():
    global counter
    for _ in range(100000):
        with lock:  # Безопасно
            counter += 1

def demo():
    global counter

    # Без блокировки (race condition)
    counter = 0
    threads = [threading.Thread(target=increment_without_lock) for _ in range(4)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"Без блокировки: {counter}")  # Меньше 400000!

    # С блокировкой
    counter = 0
    threads = [threading.Thread(target=increment_with_lock) for _ in range(4)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    print(f"С блокировкой: {counter}")  # Ровно 400000

if __name__ == "__main__":
    demo()
```

### Semaphore (Семафор)

```python
import threading
import time

# Ограничиваем количество одновременных подключений
connection_semaphore = threading.Semaphore(3)

def access_resource(thread_id: int):
    print(f"Поток {thread_id} ожидает доступа...")

    with connection_semaphore:
        print(f"Поток {thread_id} получил доступ")
        time.sleep(2)  # Работа с ресурсом
        print(f"Поток {thread_id} освободил ресурс")

def main():
    threads = [
        threading.Thread(target=access_resource, args=(i,))
        for i in range(6)
    ]

    for t in threads:
        t.start()
    for t in threads:
        t.join()

if __name__ == "__main__":
    main()
```

### Event (Событие)

```python
import threading
import time

event = threading.Event()

def waiter(name: str):
    print(f"{name} ждёт сигнала...")
    event.wait()  # Блокируется до set()
    print(f"{name} получил сигнал!")

def signaler():
    print("Сигнальщик: жду 3 секунды...")
    time.sleep(3)
    print("Сигнальщик: подаю сигнал!")
    event.set()  # Разблокирует всех ожидающих

def main():
    threads = [
        threading.Thread(target=waiter, args=(f"Официант-{i}",))
        for i in range(3)
    ]
    threads.append(threading.Thread(target=signaler))

    for t in threads:
        t.start()
    for t in threads:
        t.join()

if __name__ == "__main__":
    main()
```

## Проблемы многопоточности

### 1. Race Condition (Состояние гонки)

```python
# ПРОБЛЕМА: Race condition
balance = 1000

def withdraw(amount):
    global balance
    if balance >= amount:
        # Между проверкой и списанием другой поток может изменить balance
        time.sleep(0.001)  # Симуляция задержки
        balance -= amount
        return True
    return False

# РЕШЕНИЕ: Использование блокировки
balance_lock = threading.Lock()

def safe_withdraw(amount):
    global balance
    with balance_lock:
        if balance >= amount:
            time.sleep(0.001)
            balance -= amount
            return True
        return False
```

### 2. Deadlock (Взаимная блокировка)

```python
# ПРОБЛЕМА: Deadlock
lock_a = threading.Lock()
lock_b = threading.Lock()

def thread_1():
    with lock_a:
        time.sleep(0.1)
        with lock_b:  # Ждёт lock_b
            print("Thread 1")

def thread_2():
    with lock_b:
        time.sleep(0.1)
        with lock_a:  # Ждёт lock_a - DEADLOCK!
            print("Thread 2")

# РЕШЕНИЕ: Всегда захватывать блокировки в одном порядке
def safe_thread_1():
    with lock_a:
        with lock_b:
            print("Thread 1")

def safe_thread_2():
    with lock_a:  # Тот же порядок!
        with lock_b:
            print("Thread 2")
```

## Когда использовать что?

| Задача | Инструмент | Причина |
|--------|------------|---------|
| CPU-bound | multiprocessing | Обходит GIL |
| I/O-bound (простой) | threading | Простота |
| I/O-bound (сложный) | asyncio | Эффективность |
| Изолированные задачи | multiprocessing | Безопасность |
| Общие данные | threading | Простой обмен |

## Заключение

- **Процессы** - изолированные единицы выполнения, идеальны для CPU-bound
- **Потоки** - легковесные единицы внутри процесса, подходят для I/O-bound
- **Синхронизация** - необходима при работе с общими ресурсами
- **asyncio** - альтернатива потокам для I/O-bound с лучшей масштабируемостью

В следующем разделе мы рассмотрим GIL и почему он так важен для понимания конкурентности в Python.

---

[prev: ./03-concurrency-parallelism.md](./03-concurrency-parallelism.md) | [next: ./05-gil.md](./05-gil.md)
