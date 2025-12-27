# Multiprocessing (Процессы)

## Зачем?

- Обход GIL — каждый процесс имеет свой
- CPU-bound задачи — настоящая параллельность

---

## Основы

```python
from multiprocessing import Process

def worker(name):
    print(f"Процесс {name}")

if __name__ == "__main__":  # Обязательно!
    p = Process(target=worker, args=("A",))
    p.start()
    p.join()
```

---

## Pool

```python
from multiprocessing import Pool

def square(x):
    return x ** 2

if __name__ == "__main__":
    with Pool(4) as pool:
        results = pool.map(square, range(10))
        # [0, 1, 4, 9, 16, 25, 36, 49, 64, 81]
```

---

## ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor

def task(n):
    return sum(range(n))

if __name__ == "__main__":
    with ProcessPoolExecutor(max_workers=4) as executor:
        results = executor.map(task, [1000000] * 4)
```

---

## Обмен данными

### Queue

```python
from multiprocessing import Queue

queue = Queue()
queue.put(item)
item = queue.get()
```

### Value / Array — разделяемая память

```python
from multiprocessing import Value, Array

counter = Value('i', 0)       # int
arr = Array('d', [1.0, 2.0])  # double
```

### Manager — сложные структуры

```python
from multiprocessing import Manager

with Manager() as m:
    shared_list = m.list([1, 2])
    shared_dict = m.dict({"a": 1})
```

---

## Lock

```python
from multiprocessing import Lock

lock = Lock()
with lock:
    # Только один процесс
    counter.value += 1
```

---

## Резюме

| Инструмент | Назначение |
|------------|-----------|
| `Process` | Один процесс |
| `Pool` | Пул процессов |
| `ProcessPoolExecutor` | concurrent.futures API |
| `Queue` | Очередь между процессами |
| `Value`, `Array` | Разделяемая память |
| `Manager` | Сложные структуры |
| `Lock` | Синхронизация |

Используй для **CPU-bound** задач.
