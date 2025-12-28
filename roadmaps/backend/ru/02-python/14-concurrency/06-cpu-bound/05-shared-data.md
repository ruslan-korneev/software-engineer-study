# Разделяемые данные и блокировки

[prev: 04-map-reduce.md](./04-map-reduce.md) | [next: 06-multiple-event-loops.md](./06-multiple-event-loops.md)

---

## Проблема разделяемых данных

В multiprocessing каждый процесс имеет **свое собственное адресное пространство**. Это означает, что обычные переменные Python не разделяются между процессами:

```python
import multiprocessing

counter = 0  # Глобальная переменная

def increment():
    global counter
    for _ in range(100000):
        counter += 1

if __name__ == "__main__":
    processes = [
        multiprocessing.Process(target=increment)
        for _ in range(4)
    ]

    for p in processes:
        p.start()
    for p in processes:
        p.join()

    print(f"Ожидаемо: 400000")
    print(f"Фактически: {counter}")  # 0! Каждый процесс работал со своей копией
```

## Разделяемая память (Shared Memory)

### Value — разделяемое значение

```python
import multiprocessing
import ctypes

def increment(shared_value, lock):
    for _ in range(100000):
        with lock:  # Блокировка для предотвращения гонки
            shared_value.value += 1

if __name__ == "__main__":
    # Создаем разделяемое значение (i = integer)
    counter = multiprocessing.Value('i', 0)
    lock = multiprocessing.Lock()

    processes = [
        multiprocessing.Process(target=increment, args=(counter, lock))
        for _ in range(4)
    ]

    for p in processes:
        p.start()
    for p in processes:
        p.join()

    print(f"Результат: {counter.value}")  # 400000
```

### Типы данных для Value

```python
import multiprocessing
from ctypes import c_double, c_bool, c_char_p

# Числа
int_val = multiprocessing.Value('i', 0)       # int (signed)
uint_val = multiprocessing.Value('I', 0)      # unsigned int
long_val = multiprocessing.Value('l', 0)      # long
double_val = multiprocessing.Value('d', 0.0)  # double

# Использование ctypes напрямую
float_val = multiprocessing.Value(c_double, 3.14)

# Доступ к значению
print(int_val.value)
int_val.value = 42
```

### Array — разделяемый массив

```python
import multiprocessing

def worker(shared_array, index, value):
    shared_array[index] = value

if __name__ == "__main__":
    # Создаем разделяемый массив
    arr = multiprocessing.Array('i', 10)  # массив из 10 int

    processes = []
    for i in range(10):
        p = multiprocessing.Process(target=worker, args=(arr, i, i * 10))
        processes.append(p)
        p.start()

    for p in processes:
        p.join()

    print(f"Массив: {list(arr)}")  # [0, 10, 20, 30, 40, 50, 60, 70, 80, 90]
```

### RawValue и RawArray

Версии без встроенной блокировки (быстрее, но требуют ручной синхронизации):

```python
import multiprocessing

if __name__ == "__main__":
    # Без встроенной блокировки
    raw_val = multiprocessing.RawValue('i', 0)
    raw_arr = multiprocessing.RawArray('d', 10)

    # Нужна явная блокировка при доступе!
    lock = multiprocessing.Lock()

    with lock:
        raw_val.value += 1
        raw_arr[0] = 3.14
```

## shared_memory (Python 3.8+)

Модуль `multiprocessing.shared_memory` предоставляет более гибкий способ работы с разделяемой памятью:

```python
from multiprocessing import shared_memory, Process
import numpy as np

def worker(shm_name, shape, dtype):
    """Воркер, работающий с разделяемым numpy массивом"""
    # Подключаемся к существующей разделяемой памяти
    existing_shm = shared_memory.SharedMemory(name=shm_name)
    arr = np.ndarray(shape, dtype=dtype, buffer=existing_shm.buf)

    # Изменяем данные
    arr[:] = arr * 2

    # Закрываем (не удаляем!) соединение
    existing_shm.close()

if __name__ == "__main__":
    # Создаем numpy массив
    original = np.array([1, 2, 3, 4, 5], dtype=np.float64)

    # Создаем разделяемую память
    shm = shared_memory.SharedMemory(create=True, size=original.nbytes)

    # Создаем numpy массив поверх разделяемой памяти
    shared_arr = np.ndarray(original.shape, dtype=original.dtype, buffer=shm.buf)
    shared_arr[:] = original[:]  # Копируем данные

    print(f"До: {shared_arr}")

    # Запускаем воркер
    p = Process(target=worker, args=(shm.name, original.shape, original.dtype))
    p.start()
    p.join()

    print(f"После: {shared_arr}")  # [2, 4, 6, 8, 10]

    # Очистка
    shm.close()
    shm.unlink()  # Удаляем разделяемую память
```

### ShareableList

Разделяемый список с фиксированным размером:

```python
from multiprocessing import shared_memory, Process

def modifier(shm_name):
    sl = shared_memory.ShareableList(name=shm_name)
    sl[0] = sl[0] * 2  # Изменяем первый элемент
    sl[2] = "modified"
    sl.shm.close()

if __name__ == "__main__":
    # Создаем разделяемый список
    sl = shared_memory.ShareableList([10, 20, "hello", 3.14])

    print(f"До: {list(sl)}")

    p = Process(target=modifier, args=(sl.shm.name,))
    p.start()
    p.join()

    print(f"После: {list(sl)}")  # [20, 20, 'modified', 3.14]

    sl.shm.close()
    sl.shm.unlink()
```

## Примитивы синхронизации

### Lock — простая блокировка

```python
import multiprocessing
import time

def worker(lock, worker_id):
    print(f"Worker {worker_id}: ожидает блокировку")

    with lock:  # Автоматически acquire/release
        print(f"Worker {worker_id}: получил блокировку")
        time.sleep(1)
        print(f"Worker {worker_id}: освобождает блокировку")

if __name__ == "__main__":
    lock = multiprocessing.Lock()

    processes = [
        multiprocessing.Process(target=worker, args=(lock, i))
        for i in range(3)
    ]

    for p in processes:
        p.start()
    for p in processes:
        p.join()
```

### RLock — рекурсивная блокировка

Позволяет одному процессу захватывать блокировку несколько раз:

```python
import multiprocessing

def recursive_function(lock, depth):
    with lock:
        print(f"Глубина: {depth}")
        if depth > 0:
            recursive_function(lock, depth - 1)

if __name__ == "__main__":
    # С обычным Lock это вызвало бы deadlock!
    rlock = multiprocessing.RLock()

    p = multiprocessing.Process(target=recursive_function, args=(rlock, 3))
    p.start()
    p.join()
```

### Semaphore — семафор

Ограничивает количество процессов, которые могут одновременно выполнять код:

```python
import multiprocessing
import time

def limited_resource(semaphore, worker_id):
    with semaphore:
        print(f"Worker {worker_id}: использует ресурс")
        time.sleep(2)
        print(f"Worker {worker_id}: освободил ресурс")

if __name__ == "__main__":
    # Максимум 2 процесса одновременно
    semaphore = multiprocessing.Semaphore(2)

    processes = [
        multiprocessing.Process(target=limited_resource, args=(semaphore, i))
        for i in range(5)
    ]

    for p in processes:
        p.start()
    for p in processes:
        p.join()
```

### Event — событие

Используется для сигнализации между процессами:

```python
import multiprocessing
import time

def waiter(event, name):
    print(f"{name}: ожидаю сигнал...")
    event.wait()  # Блокируется пока event не установлен
    print(f"{name}: получил сигнал!")

def setter(event):
    print("Setter: жду 3 секунды...")
    time.sleep(3)
    print("Setter: отправляю сигнал!")
    event.set()

if __name__ == "__main__":
    event = multiprocessing.Event()

    waiters = [
        multiprocessing.Process(target=waiter, args=(event, f"Waiter-{i}"))
        for i in range(3)
    ]
    setter_process = multiprocessing.Process(target=setter, args=(event,))

    for w in waiters:
        w.start()
    setter_process.start()

    for w in waiters:
        w.join()
    setter_process.join()
```

### Condition — условие

Более сложная синхронизация с условиями:

```python
import multiprocessing
import time

def consumer(condition, shared_list):
    with condition:
        while len(shared_list) == 0:
            print("Consumer: жду данные...")
            condition.wait()

        item = shared_list.pop()
        print(f"Consumer: получил {item}")

def producer(condition, shared_list):
    time.sleep(2)
    with condition:
        shared_list.append("данные")
        print("Producer: добавил данные")
        condition.notify()  # Уведомляем одного ожидающего

if __name__ == "__main__":
    manager = multiprocessing.Manager()
    shared_list = manager.list()
    condition = multiprocessing.Condition()

    c = multiprocessing.Process(target=consumer, args=(condition, shared_list))
    p = multiprocessing.Process(target=producer, args=(condition, shared_list))

    c.start()
    p.start()

    c.join()
    p.join()
```

### Barrier — барьер

Синхронизирует процессы в определенной точке:

```python
import multiprocessing
import time
import random

def worker(barrier, worker_id):
    work_time = random.uniform(0.5, 2.0)
    print(f"Worker {worker_id}: работаю {work_time:.1f} сек")
    time.sleep(work_time)

    print(f"Worker {worker_id}: достиг барьера")
    barrier.wait()  # Ждем всех

    print(f"Worker {worker_id}: продолжаю после барьера")

if __name__ == "__main__":
    # Барьер на 4 участника
    barrier = multiprocessing.Barrier(4)

    processes = [
        multiprocessing.Process(target=worker, args=(barrier, i))
        for i in range(4)
    ]

    for p in processes:
        p.start()
    for p in processes:
        p.join()
```

## Manager — разделяемые объекты Python

`Manager` позволяет создавать разделяемые структуры данных Python:

```python
import multiprocessing

def worker(shared_dict, shared_list, key, value):
    shared_dict[key] = value
    shared_list.append(value)

if __name__ == "__main__":
    with multiprocessing.Manager() as manager:
        # Разделяемые структуры данных
        shared_dict = manager.dict()
        shared_list = manager.list()

        processes = []
        for i in range(5):
            p = multiprocessing.Process(
                target=worker,
                args=(shared_dict, shared_list, f"key_{i}", i * 10)
            )
            processes.append(p)
            p.start()

        for p in processes:
            p.join()

        print(f"Dict: {dict(shared_dict)}")
        print(f"List: {list(shared_list)}")
```

### Доступные типы Manager

```python
import multiprocessing

with multiprocessing.Manager() as manager:
    # Базовые структуры
    d = manager.dict()        # Разделяемый словарь
    l = manager.list()        # Разделяемый список

    # Примитивы синхронизации
    lock = manager.Lock()
    rlock = manager.RLock()
    semaphore = manager.Semaphore(3)
    event = manager.Event()
    condition = manager.Condition()
    barrier = manager.Barrier(4)

    # Разделяемые значения
    ns = manager.Namespace()  # Пространство имен
    ns.x = 10
    ns.y = "hello"

    # Очередь
    q = manager.Queue()
```

## Межпроцессные очереди

### Queue

```python
import multiprocessing
import time

def producer(queue):
    for i in range(5):
        queue.put(f"item_{i}")
        print(f"Produced: item_{i}")
        time.sleep(0.5)
    queue.put(None)  # Сигнал завершения

def consumer(queue):
    while True:
        item = queue.get()
        if item is None:
            break
        print(f"Consumed: {item}")

if __name__ == "__main__":
    queue = multiprocessing.Queue()

    p = multiprocessing.Process(target=producer, args=(queue,))
    c = multiprocessing.Process(target=consumer, args=(queue,))

    p.start()
    c.start()

    p.join()
    c.join()
```

### JoinableQueue

Позволяет отслеживать завершение обработки элементов:

```python
import multiprocessing
import time

def worker(queue):
    while True:
        item = queue.get()
        if item is None:
            queue.task_done()
            break

        print(f"Обрабатываю: {item}")
        time.sleep(0.5)
        queue.task_done()  # Сигнализируем о завершении обработки

if __name__ == "__main__":
    queue = multiprocessing.JoinableQueue()

    # Запускаем воркеры
    workers = [
        multiprocessing.Process(target=worker, args=(queue,))
        for _ in range(3)
    ]
    for w in workers:
        w.start()

    # Добавляем задачи
    for i in range(10):
        queue.put(f"task_{i}")

    # Добавляем сигналы завершения
    for _ in range(3):
        queue.put(None)

    # Ждем обработки всех задач
    queue.join()

    for w in workers:
        w.join()

    print("Все задачи обработаны!")
```

### Pipe

Двунаправленный канал между двумя процессами:

```python
import multiprocessing

def sender(conn):
    conn.send("Hello from sender!")
    conn.send([1, 2, 3])
    conn.send({'key': 'value'})
    conn.close()

def receiver(conn):
    while True:
        try:
            msg = conn.recv()
            print(f"Received: {msg}")
        except EOFError:
            break
    conn.close()

if __name__ == "__main__":
    # Создаем двунаправленный канал
    parent_conn, child_conn = multiprocessing.Pipe()

    p1 = multiprocessing.Process(target=sender, args=(parent_conn,))
    p2 = multiprocessing.Process(target=receiver, args=(child_conn,))

    p1.start()
    p2.start()

    p1.join()
    p2.join()
```

## Best Practices

1. **Минимизируйте разделяемые данные** — передача данных между процессами дорогая
2. **Используйте Lock с менеджером контекста** — избегайте deadlock
3. **Предпочитайте Queue для коммуникации** — безопаснее, чем разделяемая память
4. **Избегайте Manager для производительности** — используйте Value/Array
5. **Всегда очищайте shared_memory** — вызывайте `unlink()`

## Типичные ошибки

```python
# НЕПРАВИЛЬНО: забыли блокировку
def bad_increment(shared_val):
    for _ in range(1000):
        shared_val.value += 1  # Гонка данных!

# ПРАВИЛЬНО: используем блокировку
def good_increment(shared_val, lock):
    for _ in range(1000):
        with lock:
            shared_val.value += 1
```

```python
# НЕПРАВИЛЬНО: не закрыли shared_memory
shm = shared_memory.SharedMemory(create=True, size=100)
# ... использование ...
# Утечка памяти!

# ПРАВИЛЬНО: закрываем и удаляем
shm = shared_memory.SharedMemory(create=True, size=100)
try:
    # ... использование ...
finally:
    shm.close()
    shm.unlink()
```

---

[prev: 04-map-reduce.md](./04-map-reduce.md) | [next: 06-multiple-event-loops.md](./06-multiple-event-loops.md)
