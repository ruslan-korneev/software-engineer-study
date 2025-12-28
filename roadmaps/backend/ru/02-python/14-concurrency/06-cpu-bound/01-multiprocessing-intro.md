# Введение в multiprocessing

[prev: readme.md](./readme.md) | [next: 02-process-pools.md](./02-process-pools.md)

---

## Что такое CPU-bound задачи?

CPU-bound (счетные) задачи — это задачи, которые требуют интенсивных вычислений процессора. В отличие от I/O-bound задач (ожидание сети, диска), CPU-bound задачи постоянно загружают процессор.

### Примеры CPU-bound задач

- Математические вычисления (факториал, числа Фибоначчи)
- Обработка изображений и видео
- Криптографические операции
- Сжатие/распаковка данных
- Машинное обучение и анализ данных
- Парсинг больших объемов данных

## Проблема GIL в Python

**Global Interpreter Lock (GIL)** — это механизм в CPython, который позволяет только одному потоку выполнять Python-байткод в каждый момент времени.

```python
import threading
import time

def cpu_intensive_task(n):
    """Вычисление суммы квадратов"""
    result = 0
    for i in range(n):
        result += i ** 2
    return result

# Попытка использовать потоки для CPU-bound задачи
def run_with_threads():
    start = time.time()
    threads = []
    for _ in range(4):
        t = threading.Thread(target=cpu_intensive_task, args=(10_000_000,))
        threads.append(t)
        t.start()

    for t in threads:
        t.join()

    print(f"Потоки: {time.time() - start:.2f} сек")

run_with_threads()  # Медленно из-за GIL!
```

**Вывод**: Для CPU-bound задач потоки (threading) не дают ускорения из-за GIL.

## Модуль multiprocessing

Модуль `multiprocessing` позволяет создавать отдельные процессы, каждый из которых имеет свой интерпретатор Python и не ограничен GIL.

### Базовый пример создания процесса

```python
import multiprocessing
import os

def worker(name):
    """Функция, которая будет выполняться в отдельном процессе"""
    print(f"Процесс {name} запущен")
    print(f"PID процесса: {os.getpid()}")
    print(f"PID родительского процесса: {os.getppid()}")

if __name__ == "__main__":
    print(f"Главный процесс PID: {os.getpid()}")

    # Создаем процесс
    process = multiprocessing.Process(target=worker, args=("Worker-1",))

    # Запускаем процесс
    process.start()

    # Ждем завершения
    process.join()

    print("Главный процесс завершен")
```

### Важно: `if __name__ == "__main__"`

В Windows и при использовании метода "spawn" (по умолчанию на macOS начиная с Python 3.8), модуль multiprocessing импортирует главный модуль заново. Без проверки `if __name__ == "__main__"` код создания процессов будет выполнен рекурсивно.

## Сравнение: потоки vs процессы

```python
import multiprocessing
import threading
import time

def cpu_task(n):
    """CPU-интенсивная задача"""
    total = 0
    for i in range(n):
        total += i ** 2
    return total

def run_sequential(n, count):
    """Последовательное выполнение"""
    start = time.time()
    for _ in range(count):
        cpu_task(n)
    return time.time() - start

def run_threaded(n, count):
    """Выполнение с потоками"""
    start = time.time()
    threads = [threading.Thread(target=cpu_task, args=(n,)) for _ in range(count)]
    for t in threads:
        t.start()
    for t in threads:
        t.join()
    return time.time() - start

def run_multiprocess(n, count):
    """Выполнение с процессами"""
    start = time.time()
    processes = [multiprocessing.Process(target=cpu_task, args=(n,)) for _ in range(count)]
    for p in processes:
        p.start()
    for p in processes:
        p.join()
    return time.time() - start

if __name__ == "__main__":
    N = 5_000_000
    COUNT = 4

    seq_time = run_sequential(N, COUNT)
    thread_time = run_threaded(N, COUNT)
    process_time = run_multiprocess(N, COUNT)

    print(f"Последовательно: {seq_time:.2f} сек")
    print(f"Потоки:          {thread_time:.2f} сек")
    print(f"Процессы:        {process_time:.2f} сек")

    print(f"\nУскорение процессами: {seq_time / process_time:.2f}x")
```

**Типичный вывод на 4-ядерном процессоре:**
```
Последовательно: 8.45 сек
Потоки:          8.89 сек  (даже медленнее из-за GIL!)
Процессы:        2.34 сек  (почти 4x ускорение!)
```

## Методы запуска процессов

Python поддерживает три метода создания процессов:

```python
import multiprocessing

# Получить текущий метод
print(multiprocessing.get_start_method())

# Установить метод (должно быть сделано в начале программы)
# multiprocessing.set_start_method('fork')  # или 'spawn', 'forkserver'
```

| Метод | ОС | Описание |
|-------|-----|----------|
| `fork` | Unix | Копирует память родительского процесса (быстро, но может быть небезопасно) |
| `spawn` | Все ОС | Запускает новый интерпретатор Python (безопасно, но медленнее) |
| `forkserver` | Unix | Использует сервер для fork (компромисс между fork и spawn) |

## Передача аргументов и получение результатов

### Использование Queue для получения результатов

```python
import multiprocessing

def calculate_square(number, queue):
    """Вычисляет квадрат и помещает результат в очередь"""
    result = number ** 2
    queue.put((number, result))

if __name__ == "__main__":
    numbers = [1, 2, 3, 4, 5]
    queue = multiprocessing.Queue()
    processes = []

    for num in numbers:
        p = multiprocessing.Process(target=calculate_square, args=(num, queue))
        processes.append(p)
        p.start()

    for p in processes:
        p.join()

    # Получаем результаты
    while not queue.empty():
        num, result = queue.get()
        print(f"{num}^2 = {result}")
```

## Управление процессами

```python
import multiprocessing
import time

def long_running_task():
    """Долгая задача"""
    while True:
        print("Работаю...")
        time.sleep(1)

if __name__ == "__main__":
    p = multiprocessing.Process(target=long_running_task)

    # Запуск
    p.start()
    print(f"Процесс запущен: {p.is_alive()}")  # True
    print(f"PID: {p.pid}")
    print(f"Exit code: {p.exitcode}")  # None (еще работает)

    time.sleep(3)

    # Принудительное завершение
    p.terminate()  # или p.kill() для более жесткого завершения
    p.join()

    print(f"Процесс жив: {p.is_alive()}")  # False
    print(f"Exit code: {p.exitcode}")  # -15 (SIGTERM)
```

## Daemon-процессы

Daemon-процессы автоматически завершаются при завершении главного процесса:

```python
import multiprocessing
import time

def daemon_worker():
    while True:
        print("Daemon работает...")
        time.sleep(1)

if __name__ == "__main__":
    p = multiprocessing.Process(target=daemon_worker)
    p.daemon = True  # Помечаем как daemon
    p.start()

    time.sleep(3)
    print("Главный процесс завершается")
    # daemon-процесс будет автоматически убит
```

## Best Practices

1. **Всегда используйте `if __name__ == "__main__"`** — обязательно для корректной работы
2. **Не создавайте слишком много процессов** — оптимально `multiprocessing.cpu_count()`
3. **Используйте пулы процессов** — вместо ручного создания процессов
4. **Минимизируйте передачу данных** — сериализация/десериализация затратна
5. **Закрывайте ресурсы корректно** — используйте менеджеры контекста

## Типичные ошибки

```python
# НЕПРАВИЛЬНО: нет защиты if __name__
import multiprocessing

p = multiprocessing.Process(target=some_func)  # Ошибка на Windows!
p.start()

# ПРАВИЛЬНО
if __name__ == "__main__":
    p = multiprocessing.Process(target=some_func)
    p.start()
```

```python
# НЕПРАВИЛЬНО: передача несериализуемых объектов
import multiprocessing

def worker(conn):  # Нельзя передать сокет или подключение к БД
    pass

# ПРАВИЛЬНО: создавать ресурсы внутри процесса
def worker():
    conn = create_connection()  # Создаем внутри процесса
```

---

[prev: readme.md](./readme.md) | [next: 02-process-pools.md](./02-process-pools.md)
