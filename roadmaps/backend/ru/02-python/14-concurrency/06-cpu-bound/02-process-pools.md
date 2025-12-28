# Использование пулов процессов

[prev: 01-multiprocessing-intro.md](./01-multiprocessing-intro.md) | [next: 03-process-pool-executors.md](./03-process-pool-executors.md)

---

## Что такое пул процессов?

**Пул процессов (Pool)** — это заранее созданная группа рабочих процессов, которые ожидают задачи. Вместо создания нового процесса для каждой задачи, задачи распределяются между уже существующими процессами.

### Преимущества пула процессов

- **Экономия ресурсов** — процессы создаются один раз
- **Быстрое выполнение** — нет затрат на создание/уничтожение процессов
- **Удобное API** — встроенные методы для параллельного выполнения
- **Автоматическое управление** — пул сам распределяет задачи

## Класс Pool

```python
import multiprocessing

# Создание пула с указанием количества процессов
pool = multiprocessing.Pool(processes=4)

# Или пул с количеством процессов = количеству ядер CPU
pool = multiprocessing.Pool()  # по умолчанию cpu_count()
```

## Метод apply()

Выполняет функцию в одном из процессов пула и **блокирует** до получения результата:

```python
import multiprocessing
import time

def square(x):
    time.sleep(1)  # Имитация вычислений
    return x ** 2

if __name__ == "__main__":
    with multiprocessing.Pool(4) as pool:
        # apply() - синхронный вызов (блокирующий)
        result = pool.apply(square, args=(5,))
        print(f"Результат: {result}")  # 25
```

**Недостаток**: `apply()` блокирует, поэтому задачи выполняются последовательно.

## Метод apply_async()

Асинхронная версия `apply()`. Возвращает объект `AsyncResult`:

```python
import multiprocessing
import time

def compute(x):
    time.sleep(1)
    return x ** 2

if __name__ == "__main__":
    with multiprocessing.Pool(4) as pool:
        # Запускаем задачи асинхронно
        results = []
        for i in range(8):
            result = pool.apply_async(compute, args=(i,))
            results.append(result)

        # Получаем результаты
        for r in results:
            print(r.get(timeout=10))  # get() блокирует до готовности результата
```

### Методы AsyncResult

```python
import multiprocessing

def slow_task(x):
    import time
    time.sleep(2)
    return x * 2

if __name__ == "__main__":
    with multiprocessing.Pool(2) as pool:
        async_result = pool.apply_async(slow_task, args=(10,))

        # Проверка готовности (не блокирует)
        print(f"Готово: {async_result.ready()}")  # False

        # Проверка успешности (только после ready() == True)
        # async_result.successful()

        # Ожидание завершения (блокирует)
        async_result.wait(timeout=5)

        # Получение результата (блокирует если не готово)
        result = async_result.get(timeout=10)
        print(f"Результат: {result}")  # 20
```

## Метод map()

`map()` применяет функцию к каждому элементу итерируемого объекта **параллельно**:

```python
import multiprocessing
import time

def process_item(item):
    """Обработка одного элемента"""
    time.sleep(0.5)  # Имитация работы
    return item ** 2

if __name__ == "__main__":
    items = list(range(16))

    # Последовательная обработка
    start = time.time()
    sequential_results = [process_item(x) for x in items]
    print(f"Последовательно: {time.time() - start:.2f} сек")

    # Параллельная обработка с пулом
    start = time.time()
    with multiprocessing.Pool(4) as pool:
        parallel_results = pool.map(process_item, items)
    print(f"Параллельно: {time.time() - start:.2f} сек")

    # Проверка результатов
    assert sequential_results == parallel_results
    print(f"Результаты: {parallel_results}")
```

### Параметр chunksize

`chunksize` определяет, сколько элементов отправляется каждому процессу за раз:

```python
import multiprocessing

def process(x):
    return x ** 2

if __name__ == "__main__":
    items = list(range(1000))

    with multiprocessing.Pool(4) as pool:
        # Маленький chunksize: больше накладных расходов на IPC
        result1 = pool.map(process, items, chunksize=1)

        # Большой chunksize: меньше накладных расходов
        result2 = pool.map(process, items, chunksize=100)

        # Автоматический расчет (по умолчанию)
        result3 = pool.map(process, items)  # chunksize = len(items) // (4 * pool_size)
```

**Рекомендация**: Для быстрых функций используйте большой `chunksize`, для медленных — маленький.

## Метод map_async()

Асинхронная версия `map()`:

```python
import multiprocessing
import time

def compute(x):
    time.sleep(0.1)
    return x ** 2

if __name__ == "__main__":
    items = list(range(20))

    with multiprocessing.Pool(4) as pool:
        # Запускаем map асинхронно
        async_result = pool.map_async(compute, items)

        # Можем делать что-то другое пока идут вычисления
        print("Вычисления запущены...")

        # Ждем результаты
        results = async_result.get(timeout=30)
        print(f"Результаты: {results}")
```

## Метод imap()

`imap()` — ленивая версия `map()`. Возвращает итератор:

```python
import multiprocessing
import time

def slow_square(x):
    time.sleep(0.5)
    return x ** 2

if __name__ == "__main__":
    items = list(range(10))

    with multiprocessing.Pool(4) as pool:
        # imap возвращает итератор, результаты приходят по мере готовности
        # НО в порядке входных данных!
        for result in pool.imap(slow_square, items):
            print(f"Получен результат: {result}")
```

## Метод imap_unordered()

Как `imap()`, но результаты возвращаются по мере готовности, **без сохранения порядка**:

```python
import multiprocessing
import time
import random

def variable_task(x):
    """Задача с переменным временем выполнения"""
    sleep_time = random.uniform(0.1, 1.0)
    time.sleep(sleep_time)
    return x ** 2

if __name__ == "__main__":
    items = list(range(10))

    with multiprocessing.Pool(4) as pool:
        # Результаты приходят по мере готовности
        for result in pool.imap_unordered(variable_task, items):
            print(f"Получен результат: {result}")
```

**Когда использовать `imap_unordered()`**:
- Когда порядок результатов не важен
- Когда нужно обрабатывать результаты сразу по мере готовности

## Метод starmap()

`starmap()` — как `map()`, но распаковывает аргументы:

```python
import multiprocessing

def add(a, b):
    return a + b

if __name__ == "__main__":
    # Список кортежей аргументов
    args = [(1, 2), (3, 4), (5, 6), (7, 8)]

    with multiprocessing.Pool(4) as pool:
        # starmap распаковывает кортежи как аргументы
        results = pool.starmap(add, args)
        print(results)  # [3, 7, 11, 15]

        # Эквивалент без starmap:
        # results = pool.map(lambda x: add(*x), args)
```

### starmap_async()

```python
import multiprocessing

def power(base, exp):
    return base ** exp

if __name__ == "__main__":
    args = [(2, 3), (3, 3), (4, 2), (5, 2)]

    with multiprocessing.Pool(4) as pool:
        async_result = pool.starmap_async(power, args)
        results = async_result.get()
        print(results)  # [8, 27, 16, 25]
```

## Инициализация воркеров

Можно задать функцию инициализации, которая выполнится один раз при создании каждого процесса:

```python
import multiprocessing
import os

def initializer():
    """Выполняется один раз при создании процесса"""
    print(f"Инициализация воркера {os.getpid()}")
    # Здесь можно открыть соединение с БД, загрузить модель и т.д.

def worker(x):
    return x ** 2

if __name__ == "__main__":
    with multiprocessing.Pool(
        processes=4,
        initializer=initializer
    ) as pool:
        results = pool.map(worker, range(10))
        print(results)
```

### Инициализация с аргументами

```python
import multiprocessing

# Глобальная переменная для хранения состояния воркера
worker_data = None

def init_worker(data):
    """Инициализация с аргументами"""
    global worker_data
    worker_data = data
    print(f"Воркер инициализирован с: {data}")

def process(x):
    global worker_data
    return x * worker_data

if __name__ == "__main__":
    with multiprocessing.Pool(
        processes=4,
        initializer=init_worker,
        initargs=(10,)
    ) as pool:
        results = pool.map(process, range(5))
        print(results)  # [0, 10, 20, 30, 40]
```

## Практический пример: обработка изображений

```python
import multiprocessing
from pathlib import Path
import time

def process_image(image_path):
    """
    Имитация обработки изображения
    В реальности здесь была бы работа с PIL/OpenCV
    """
    # Имитация CPU-интенсивной обработки
    result = 0
    for i in range(1_000_000):
        result += i ** 0.5

    return f"Обработано: {image_path}"

if __name__ == "__main__":
    # Имитация списка изображений
    images = [f"image_{i}.jpg" for i in range(20)]

    # Последовательная обработка
    start = time.time()
    sequential = [process_image(img) for img in images]
    print(f"Последовательно: {time.time() - start:.2f} сек")

    # Параллельная обработка
    start = time.time()
    with multiprocessing.Pool() as pool:
        parallel = pool.map(process_image, images)
    print(f"Параллельно: {time.time() - start:.2f} сек")
```

## Обработка ошибок

```python
import multiprocessing

def risky_function(x):
    if x == 5:
        raise ValueError(f"Ошибка при обработке {x}")
    return x ** 2

if __name__ == "__main__":
    with multiprocessing.Pool(4) as pool:
        # Ошибка в одном из процессов
        try:
            results = pool.map(risky_function, range(10))
        except ValueError as e:
            print(f"Поймана ошибка: {e}")

        # Для более гибкой обработки используйте apply_async
        results = []
        for i in range(10):
            r = pool.apply_async(risky_function, args=(i,))
            results.append(r)

        for i, r in enumerate(results):
            try:
                value = r.get()
                print(f"{i}: {value}")
            except ValueError as e:
                print(f"{i}: Ошибка - {e}")
```

## Сравнение методов Pool

| Метод | Блокирует | Порядок | Результат | Использование |
|-------|-----------|---------|-----------|---------------|
| `apply()` | Да | - | Значение | Одна задача |
| `apply_async()` | Нет | - | AsyncResult | Одна задача, асинхронно |
| `map()` | Да | Сохраняет | Список | Много задач |
| `map_async()` | Нет | Сохраняет | AsyncResult | Много задач, асинхронно |
| `imap()` | Частично | Сохраняет | Итератор | Большие данные |
| `imap_unordered()` | Частично | Не сохраняет | Итератор | Большие данные, порядок не важен |
| `starmap()` | Да | Сохраняет | Список | Много аргументов |

## Best Practices

1. **Используйте менеджер контекста** — гарантирует корректное закрытие пула
2. **Количество процессов = количеству ядер** — для CPU-bound задач
3. **Подбирайте chunksize** — влияет на производительность
4. **Обрабатывайте исключения** — ошибки в процессах не всегда очевидны
5. **Используйте `imap_unordered()`** — когда порядок не важен

---

[prev: 01-multiprocessing-intro.md](./01-multiprocessing-intro.md) | [next: 03-process-pool-executors.md](./03-process-pool-executors.md)
