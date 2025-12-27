# Threading (Потоки)

## Основы

```python
import threading

def worker(name):
    print(f"Поток {name}")

t = threading.Thread(target=worker, args=("A",))
t.start()   # Запустить
t.join()    # Ждать завершения
```

---

## Синхронизация

### Lock

```python
lock = threading.Lock()
counter = 0

def increment():
    global counter
    with lock:
        counter += 1
```

### Semaphore — ограничение

```python
sem = threading.Semaphore(3)  # Макс 3 потока

def worker():
    with sem:
        do_work()
```

### Event — сигнализация

```python
event = threading.Event()

def waiter():
    event.wait()  # Ждать
    print("Сигнал!")

def setter():
    event.set()  # Отправить сигнал
```

---

## ThreadPoolExecutor

```python
from concurrent.futures import ThreadPoolExecutor, as_completed

def fetch(url):
    return requests.get(url).status_code

urls = ["https://google.com", "https://github.com"]

with ThreadPoolExecutor(max_workers=5) as executor:
    # map
    results = executor.map(fetch, urls)

    # submit + as_completed
    futures = [executor.submit(fetch, url) for url in urls]
    for future in as_completed(futures):
        print(future.result())
```

---

## Thread-local

```python
local = threading.local()

def worker(value):
    local.x = value  # Своё для каждого потока
```

---

## Daemon потоки

```python
t = threading.Thread(target=func, daemon=True)
# Остановится при завершении главного потока
```

---

## Резюме

| Примитив | Назначение |
|----------|-----------|
| `Lock` | Взаимное исключение |
| `RLock` | Рекурсивная блокировка |
| `Semaphore` | Ограничение количества |
| `Event` | Сигнализация |
| `Condition` | Ожидание условия |
| `ThreadPoolExecutor` | Пул потоков |

Используй для **I/O-bound** задач (сеть, файлы).
