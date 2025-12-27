# GIL (Global Interpreter Lock)

## Что это?

Глобальная блокировка — только один поток выполняет Python-байткод в любой момент.

---

## Почему существует?

- **Reference counting** — управление памятью через подсчёт ссылок
- **Простота** — не нужны блокировки для каждого объекта
- **C-расширения** — проще писать

---

## Влияние GIL

### CPU-bound — GIL мешает

```python
import threading

def cpu_task():
    sum(range(50_000_000))

# 2 потока НЕ быстрее 1 потока!
t1 = threading.Thread(target=cpu_task)
t2 = threading.Thread(target=cpu_task)
# Выполняется последовательно из-за GIL
```

### I/O-bound — GIL не мешает

```python
# GIL освобождается во время I/O
def io_task():
    requests.get(url)  # GIL отпущен

# Потоки работают параллельно
threads = [threading.Thread(target=io_task) for _ in range(5)]
# 5 запросов за ~1s, не за 5s
```

**GIL освобождается при:**
- Сетевых операциях
- Чтении/записи файлов
- `time.sleep()`
- Вызовах C-библиотек (NumPy)

---

## Как обойти?

| Подход | Когда |
|--------|-------|
| `threading` | I/O-bound |
| `multiprocessing` | CPU-bound |
| `asyncio` | Много I/O |
| NumPy, Cython | Вычисления в C |

---

## Python 3.13+ Free-threaded

```python
import sys
sys._is_gil_enabled()  # False (экспериментально)
```

---

## Резюме

| Задача | GIL? | Решение |
|--------|------|---------|
| CPU-bound | Мешает | `multiprocessing` |
| I/O-bound | Не мешает | `threading`, `asyncio` |
