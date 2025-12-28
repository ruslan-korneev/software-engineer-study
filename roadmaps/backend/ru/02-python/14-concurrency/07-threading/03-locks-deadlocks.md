# Блокировки и взаимоблокировки

[prev: 02-threads-asyncio.md](./02-threads-asyncio.md) | [next: 04-event-loop-threads.md](./04-event-loop-threads.md)

---

## Проблема гонки данных (Race Condition)

**Гонка данных** возникает, когда несколько потоков одновременно обращаются к общим данным, и хотя бы один из них выполняет запись.

```python
import threading
import time

# Общий ресурс
balance = 1000

def withdraw(amount: int) -> bool:
    """Снятие денег со счёта (небезопасная версия)."""
    global balance

    if balance >= amount:
        # Имитация задержки между проверкой и операцией
        time.sleep(0.001)
        balance -= amount
        return True
    return False

# Создаём множество потоков
threads = []
for _ in range(10):
    t = threading.Thread(target=withdraw, args=(200,))
    threads.append(t)

for t in threads:
    t.start()

for t in threads:
    t.join()

print(f"Итоговый баланс: {balance}")
# Может быть отрицательным! Race condition!
```

## Lock (Мьютекс)

**Lock** — это примитив синхронизации, позволяющий только одному потоку владеть блокировкой в любой момент времени.

```python
import threading

balance = 1000
balance_lock = threading.Lock()

def safe_withdraw(amount: int) -> bool:
    """Безопасное снятие денег со счёта."""
    global balance

    # Захватываем блокировку
    balance_lock.acquire()
    try:
        if balance >= amount:
            balance -= amount
            return True
        return False
    finally:
        # Всегда освобождаем блокировку
        balance_lock.release()

# Более элегантный вариант с контекстным менеджером
def safe_withdraw_v2(amount: int) -> bool:
    """Безопасное снятие с использованием контекстного менеджера."""
    global balance

    with balance_lock:
        if balance >= amount:
            balance -= amount
            return True
        return False
```

### Неблокирующий захват

```python
import threading
import time

lock = threading.Lock()

def try_acquire_lock():
    """Попытка захватить блокировку без ожидания."""
    if lock.acquire(blocking=False):
        try:
            print(f"{threading.current_thread().name}: захватил блокировку")
            time.sleep(1)
        finally:
            lock.release()
    else:
        print(f"{threading.current_thread().name}: блокировка занята, пропускаю")

# Запуск потоков
threads = [threading.Thread(target=try_acquire_lock) for _ in range(5)]
for t in threads:
    t.start()
    time.sleep(0.1)  # Небольшая задержка между стартами

for t in threads:
    t.join()
```

### Захват с таймаутом

```python
import threading
import time

lock = threading.Lock()

def acquire_with_timeout():
    """Попытка захватить блокировку с таймаутом."""
    name = threading.current_thread().name

    # Ожидаем максимум 2 секунды
    acquired = lock.acquire(timeout=2)

    if acquired:
        try:
            print(f"{name}: успешно захватил блокировку")
            time.sleep(3)
        finally:
            lock.release()
            print(f"{name}: освободил блокировку")
    else:
        print(f"{name}: таймаут ожидания блокировки")
```

## RLock (Реентерабельная блокировка)

**RLock** позволяет одному потоку захватывать блокировку несколько раз:

```python
import threading

class BankAccount:
    """Банковский счёт с реентерабельной блокировкой."""

    def __init__(self, balance: float):
        self.balance = balance
        self._lock = threading.RLock()

    def deposit(self, amount: float) -> None:
        with self._lock:
            self.balance += amount

    def withdraw(self, amount: float) -> bool:
        with self._lock:
            if self.balance >= amount:
                self.balance -= amount
                return True
            return False

    def transfer_to(self, other: "BankAccount", amount: float) -> bool:
        """Перевод между счетами."""
        with self._lock:
            # Здесь вызывается withdraw, который тоже захватывает _lock
            # С обычным Lock это привело бы к deadlock!
            if self.withdraw(amount):
                other.deposit(amount)
                return True
            return False
```

### Сравнение Lock и RLock

```python
import threading

# Lock — нельзя захватить повторно
lock = threading.Lock()
lock.acquire()
# lock.acquire()  # DEADLOCK! Поток заблокируется навсегда
lock.release()

# RLock — можно захватить повторно
rlock = threading.RLock()
rlock.acquire()
rlock.acquire()  # OK! Увеличивается счётчик
rlock.acquire()  # OK! Счётчик = 3
rlock.release()  # Счётчик = 2
rlock.release()  # Счётчик = 1
rlock.release()  # Счётчик = 0, блокировка освобождена
```

## Deadlock (Взаимоблокировка)

**Deadlock** — ситуация, когда два или более потоков ожидают друг друга бесконечно.

### Классический пример deadlock

```python
import threading
import time

lock_a = threading.Lock()
lock_b = threading.Lock()

def thread_1():
    print("Поток 1: пытаюсь захватить lock_a")
    with lock_a:
        print("Поток 1: захватил lock_a")
        time.sleep(0.1)  # Имитация работы

        print("Поток 1: пытаюсь захватить lock_b")
        with lock_b:  # DEADLOCK!
            print("Поток 1: захватил lock_b")

def thread_2():
    print("Поток 2: пытаюсь захватить lock_b")
    with lock_b:
        print("Поток 2: захватил lock_b")
        time.sleep(0.1)  # Имитация работы

        print("Поток 2: пытаюсь захватить lock_a")
        with lock_a:  # DEADLOCK!
            print("Поток 2: захватил lock_a")

# Эти потоки заблокируются навсегда!
t1 = threading.Thread(target=thread_1)
t2 = threading.Thread(target=thread_2)

t1.start()
t2.start()
# t1.join()  # Никогда не завершится
# t2.join()  # Никогда не завершится
```

### Условия возникновения Deadlock (Coffman conditions)

1. **Mutual Exclusion** — ресурс может принадлежать только одному потоку
2. **Hold and Wait** — поток удерживает ресурс и ожидает другой
3. **No Preemption** — ресурс нельзя отобрать принудительно
4. **Circular Wait** — циклическое ожидание ресурсов

## Предотвращение Deadlock

### 1. Упорядочивание блокировок

```python
import threading

lock_a = threading.Lock()
lock_b = threading.Lock()

# Определяем глобальный порядок блокировок
LOCK_ORDER = {id(lock_a): 0, id(lock_b): 1}

def acquire_locks_in_order(*locks):
    """Захват блокировок в фиксированном порядке."""
    sorted_locks = sorted(locks, key=lambda l: LOCK_ORDER.get(id(l), id(l)))
    for lock in sorted_locks:
        lock.acquire()
    return sorted_locks

def release_locks(*locks):
    """Освобождение блокировок."""
    for lock in locks:
        lock.release()

def safe_thread_1():
    locks = acquire_locks_in_order(lock_a, lock_b)
    try:
        print("Поток 1: работаю с обоими блокировками")
    finally:
        release_locks(*locks)

def safe_thread_2():
    locks = acquire_locks_in_order(lock_b, lock_a)  # Порядок не важен
    try:
        print("Поток 2: работаю с обоими блокировками")
    finally:
        release_locks(*locks)
```

### 2. Использование таймаутов

```python
import threading
import time

lock_a = threading.Lock()
lock_b = threading.Lock()

def thread_with_timeout():
    """Поток с обработкой таймаута."""
    name = threading.current_thread().name

    while True:
        if lock_a.acquire(timeout=1):
            try:
                if lock_b.acquire(timeout=1):
                    try:
                        print(f"{name}: успешно захватил обе блокировки")
                        return  # Успех
                    finally:
                        lock_b.release()
                else:
                    print(f"{name}: таймаут на lock_b, повторяю...")
            finally:
                lock_a.release()
        else:
            print(f"{name}: таймаут на lock_a, повторяю...")

        time.sleep(0.1)  # Небольшая пауза перед повтором
```

### 3. Lock-free структуры данных

```python
import threading
import queue

# Queue — потокобезопасная структура без явных блокировок
task_queue = queue.Queue()

def producer():
    for i in range(10):
        task_queue.put(f"task-{i}")
        print(f"Произведено: task-{i}")

def consumer():
    while True:
        try:
            task = task_queue.get(timeout=1)
            print(f"Обработано: {task}")
            task_queue.task_done()
        except queue.Empty:
            break

# Нет риска deadlock!
t1 = threading.Thread(target=producer)
t2 = threading.Thread(target=consumer)

t1.start()
t2.start()

t1.join()
t2.join()
```

## Semaphore (Семафор)

**Semaphore** ограничивает количество потоков, которые могут одновременно обращаться к ресурсу:

```python
import threading
import time
import random

# Ограничиваем доступ 3 потоками
connection_semaphore = threading.Semaphore(3)

def database_query(query_id: int):
    """Симуляция запроса к базе данных."""
    name = threading.current_thread().name

    print(f"{name}: ожидаю подключения для запроса {query_id}")

    with connection_semaphore:
        print(f"{name}: выполняю запрос {query_id}")
        time.sleep(random.uniform(0.5, 1.5))  # Симуляция работы
        print(f"{name}: запрос {query_id} завершён")

# Запуск 10 потоков (но только 3 одновременно смогут работать)
threads = [
    threading.Thread(target=database_query, args=(i,))
    for i in range(10)
]

for t in threads:
    t.start()

for t in threads:
    t.join()
```

### BoundedSemaphore

```python
import threading

# BoundedSemaphore не позволяет release() без acquire()
bounded_sem = threading.BoundedSemaphore(3)

bounded_sem.acquire()
bounded_sem.release()
# bounded_sem.release()  # ValueError! Превышение начального значения
```

## Condition (Условная переменная)

**Condition** позволяет потокам ожидать определённого условия:

```python
import threading
import time
import random

class Buffer:
    """Ограниченный буфер с условной переменной."""

    def __init__(self, size: int):
        self.buffer = []
        self.size = size
        self.condition = threading.Condition()

    def put(self, item):
        """Добавить элемент (блокируется, если буфер полон)."""
        with self.condition:
            while len(self.buffer) >= self.size:
                print("Буфер полон, ожидаю...")
                self.condition.wait()

            self.buffer.append(item)
            print(f"Добавлено: {item}, размер буфера: {len(self.buffer)}")
            self.condition.notify_all()

    def get(self):
        """Получить элемент (блокируется, если буфер пуст)."""
        with self.condition:
            while len(self.buffer) == 0:
                print("Буфер пуст, ожидаю...")
                self.condition.wait()

            item = self.buffer.pop(0)
            print(f"Извлечено: {item}, размер буфера: {len(self.buffer)}")
            self.condition.notify_all()
            return item

buffer = Buffer(5)

def producer():
    for i in range(10):
        buffer.put(f"item-{i}")
        time.sleep(random.uniform(0.1, 0.3))

def consumer():
    for _ in range(10):
        item = buffer.get()
        time.sleep(random.uniform(0.2, 0.5))

t1 = threading.Thread(target=producer)
t2 = threading.Thread(target=consumer)

t1.start()
t2.start()

t1.join()
t2.join()
```

## Event (Событие)

**Event** — простейший способ сигнализации между потоками:

```python
import threading
import time

shutdown_event = threading.Event()

def worker():
    """Рабочий поток, ожидающий сигнала остановки."""
    while not shutdown_event.is_set():
        print("Работаю...")
        # Ожидаем событие с таймаутом
        shutdown_event.wait(timeout=1)

    print("Получен сигнал остановки, завершаюсь")

def controller():
    """Контроллер, отправляющий сигнал остановки."""
    time.sleep(5)
    print("Отправляю сигнал остановки")
    shutdown_event.set()

t1 = threading.Thread(target=worker)
t2 = threading.Thread(target=controller)

t1.start()
t2.start()

t1.join()
t2.join()
```

## Barrier (Барьер)

**Barrier** синхронизирует несколько потоков в определённой точке:

```python
import threading
import time
import random

# Все 3 потока должны достичь барьера перед продолжением
barrier = threading.Barrier(3)

def worker(worker_id: int):
    """Рабочий поток с синхронизацией на барьере."""
    print(f"Поток {worker_id}: начинаю подготовку")
    time.sleep(random.uniform(1, 3))  # Случайная задержка

    print(f"Поток {worker_id}: достиг барьера, ожидаю остальных")
    barrier.wait()

    print(f"Поток {worker_id}: барьер пройден, продолжаю работу")

threads = [threading.Thread(target=worker, args=(i,)) for i in range(3)]

for t in threads:
    t.start()

for t in threads:
    t.join()
```

## Best Practices

1. **Всегда используйте `with` для блокировок** — гарантирует освобождение
2. **Минимизируйте время удержания блокировки** — снижает конкуренцию
3. **Избегайте вложенных блокировок** — основной источник deadlock
4. **Используйте RLock для рекурсивных вызовов**
5. **Предпочитайте высокоуровневые примитивы** — Queue, Event вместо низкоуровневых Lock

```python
import threading

# Хорошо: минимальное время удержания блокировки
def good_pattern(lock, data):
    # Подготовка данных ВНЕ блокировки
    processed = expensive_computation(data)

    # Только короткая критическая секция
    with lock:
        shared_resource.update(processed)

# Плохо: долгое удержание блокировки
def bad_pattern(lock, data):
    with lock:
        # Вся обработка внутри блокировки
        processed = expensive_computation(data)
        shared_resource.update(processed)
```

## Распространённые ошибки

1. **Забыть освободить блокировку** — используйте `with`
2. **Захватывать блокировки в разном порядке** — deadlock
3. **Использовать Lock вместо RLock для рекурсии** — deadlock
4. **Слишком много блокировок** — сложность и риск ошибок
5. **Блокировка в callback'ах** — непредсказуемое поведение

---

[prev: 02-threads-asyncio.md](./02-threads-asyncio.md) | [next: 04-event-loop-threads.md](./04-event-loop-threads.md)
