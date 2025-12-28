# Введение в модуль threading

[prev: readme.md](./readme.md) | [next: 02-threads-asyncio.md](./02-threads-asyncio.md)

---

## Что такое потоки (threads)?

**Поток (thread)** — это минимальная единица выполнения внутри процесса. Процесс может содержать несколько потоков, которые разделяют общее адресное пространство памяти, но имеют собственный стек вызовов и локальные переменные.

### Основные характеристики потоков

| Характеристика | Описание |
|---------------|----------|
| Общая память | Потоки одного процесса разделяют память |
| Легковесность | Создание потока дешевле, чем процесса |
| Параллельность | Потоки могут выполняться параллельно на разных ядрах |
| Синхронизация | Требуется защита общих данных |

## Модуль threading в Python

Модуль `threading` предоставляет высокоуровневый API для работы с потоками:

```python
import threading
import time

def worker(name: str, delay: float) -> None:
    """Рабочая функция, выполняемая в потоке."""
    print(f"Поток {name}: начинаю работу")
    time.sleep(delay)
    print(f"Поток {name}: завершаю работу")

# Создание потоков
thread1 = threading.Thread(target=worker, args=("A", 2))
thread2 = threading.Thread(target=worker, args=("B", 1))

# Запуск потоков
thread1.start()
thread2.start()

# Ожидание завершения
thread1.join()
thread2.join()

print("Все потоки завершены")
```

## Создание потоков через наследование

Можно создать собственный класс потока:

```python
import threading
import time

class WorkerThread(threading.Thread):
    """Пользовательский класс потока."""

    def __init__(self, name: str, iterations: int):
        super().__init__()
        self.name = name
        self.iterations = iterations
        self.result = 0

    def run(self) -> None:
        """Основной метод потока."""
        for i in range(self.iterations):
            self.result += i
            time.sleep(0.1)
        print(f"Поток {self.name}: результат = {self.result}")

# Использование
threads = [
    WorkerThread("Calc-1", 10),
    WorkerThread("Calc-2", 5),
]

for t in threads:
    t.start()

for t in threads:
    t.join()
    print(f"Результат {t.name}: {t.result}")
```

## GIL (Global Interpreter Lock)

**GIL** — это механизм в CPython, который позволяет только одному потоку выполнять Python-байткод в любой момент времени.

### Влияние GIL

```python
import threading
import time

counter = 0

def increment():
    global counter
    for _ in range(1_000_000):
        counter += 1

# Без GIL это привело бы к гонке данных
threads = [threading.Thread(target=increment) for _ in range(2)]

start = time.time()
for t in threads:
    t.start()
for t in threads:
    t.join()

print(f"Ожидалось: 2_000_000, получено: {counter}")
print(f"Время: {time.time() - start:.2f}с")
```

> **Важно:** Несмотря на GIL, гонка данных всё равно возможна, так как `counter += 1` — это несколько байткод-инструкций.

## Когда использовать потоки?

### Потоки эффективны для:

1. **I/O-bound задач** — операции ввода-вывода, сетевые запросы
2. **Блокирующих вызовов** — работа с файлами, базами данных
3. **Ожидания событий** — таймеры, обработка сигналов

```python
import threading
import urllib.request
import time

def fetch_url(url: str) -> str:
    """Загрузка URL в отдельном потоке."""
    start = time.time()
    response = urllib.request.urlopen(url)
    data = response.read()
    elapsed = time.time() - start
    print(f"Загружено {url}: {len(data)} байт за {elapsed:.2f}с")
    return data

urls = [
    "https://python.org",
    "https://github.com",
    "https://stackoverflow.com",
]

# Параллельная загрузка
threads = []
for url in urls:
    t = threading.Thread(target=fetch_url, args=(url,))
    threads.append(t)
    t.start()

for t in threads:
    t.join()
```

### Потоки НЕ эффективны для:

1. **CPU-bound задач** — вычисления, обработка данных
2. **Чистых вычислений** — математические операции

## Управление потоками

### Daemon-потоки

Daemon-потоки автоматически завершаются при выходе из основной программы:

```python
import threading
import time

def background_task():
    """Фоновая задача."""
    while True:
        print("Фоновая работа...")
        time.sleep(1)

# Создание daemon-потока
daemon = threading.Thread(target=background_task, daemon=True)
daemon.start()

# Основная программа
time.sleep(3)
print("Основная программа завершена")
# Daemon-поток автоматически остановится
```

### Получение информации о потоках

```python
import threading

def worker():
    current = threading.current_thread()
    print(f"Имя потока: {current.name}")
    print(f"ID потока: {current.ident}")
    print(f"Daemon: {current.daemon}")

# Информация о главном потоке
main = threading.main_thread()
print(f"Главный поток: {main.name}")

# Количество активных потоков
print(f"Активных потоков: {threading.active_count()}")

# Список всех потоков
for t in threading.enumerate():
    print(f"  - {t.name}")
```

## Передача данных между потоками

### Использование очереди (Queue)

```python
import threading
import queue
import time
import random

def producer(q: queue.Queue, items: int):
    """Производитель данных."""
    for i in range(items):
        item = f"item-{i}"
        q.put(item)
        print(f"Произведено: {item}")
        time.sleep(random.uniform(0.1, 0.3))
    q.put(None)  # Сигнал завершения

def consumer(q: queue.Queue):
    """Потребитель данных."""
    while True:
        item = q.get()
        if item is None:
            break
        print(f"Потреблено: {item}")
        q.task_done()

# Создание очереди
data_queue = queue.Queue(maxsize=5)

# Запуск потоков
producer_thread = threading.Thread(target=producer, args=(data_queue, 10))
consumer_thread = threading.Thread(target=consumer, args=(data_queue,))

producer_thread.start()
consumer_thread.start()

producer_thread.join()
consumer_thread.join()
```

## Best Practices

1. **Используйте контекстные менеджеры** для синхронизации
2. **Избегайте глобальных переменных** — передавайте данные через очереди
3. **Устанавливайте daemon=True** для фоновых задач
4. **Всегда вызывайте join()** для не-daemon потоков
5. **Обрабатывайте исключения** внутри потоков

```python
import threading
import logging

logging.basicConfig(level=logging.INFO)

def safe_worker():
    """Безопасная рабочая функция с обработкой ошибок."""
    try:
        # Рабочий код
        result = 10 / 0
    except Exception as e:
        logging.error(f"Ошибка в потоке: {e}")

thread = threading.Thread(target=safe_worker)
thread.start()
thread.join()
```

## Распространённые ошибки

1. **Забыть вызвать start()** — поток не запустится
2. **Забыть вызвать join()** — программа может завершиться раньше потока
3. **Модифицировать общие данные без синхронизации** — гонка данных
4. **Создавать слишком много потоков** — overhead на переключение контекста

---

[prev: readme.md](./readme.md) | [next: 02-threads-asyncio.md](./02-threads-asyncio.md)
