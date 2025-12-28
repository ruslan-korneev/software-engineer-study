# MapReduce с asyncio

[prev: 03-process-pool-executors.md](./03-process-pool-executors.md) | [next: 05-shared-data.md](./05-shared-data.md)

---

## Что такое MapReduce?

**MapReduce** — это модель программирования для обработки больших объемов данных, состоящая из двух основных фаз:

1. **Map** — применение функции к каждому элементу данных (параллельно)
2. **Reduce** — агрегация результатов map-фазы в итоговый результат

### Классическая схема MapReduce

```
Входные данные
     │
     ▼
┌─────────────────────────────────────┐
│            MAP (параллельно)        │
│  [d1, d2, d3, d4] → [r1, r2, r3, r4]│
└─────────────────────────────────────┘
     │
     ▼
┌─────────────────────────────────────┐
│           REDUCE (агрегация)        │
│  [r1, r2, r3, r4] → final_result    │
└─────────────────────────────────────┘
     │
     ▼
Итоговый результат
```

## Простой MapReduce с multiprocessing

```python
from multiprocessing import Pool
from functools import reduce as functools_reduce

def map_function(text):
    """Map: подсчет слов в тексте"""
    word_counts = {}
    for word in text.lower().split():
        word = word.strip('.,!?;:')
        word_counts[word] = word_counts.get(word, 0) + 1
    return word_counts

def reduce_function(counts1, counts2):
    """Reduce: объединение словарей подсчетов"""
    result = counts1.copy()
    for word, count in counts2.items():
        result[word] = result.get(word, 0) + count
    return result

if __name__ == "__main__":
    # Входные данные - несколько текстов
    texts = [
        "Python is great Python is powerful",
        "Python is easy to learn",
        "Python has great libraries",
        "Python is used in data science",
    ]

    with Pool(4) as pool:
        # Map фаза - параллельно
        mapped = pool.map(map_function, texts)

    # Reduce фаза - последовательно
    result = functools_reduce(reduce_function, mapped)

    # Сортировка по частоте
    sorted_words = sorted(result.items(), key=lambda x: x[1], reverse=True)
    print("Топ слов:")
    for word, count in sorted_words[:5]:
        print(f"  {word}: {count}")
```

## MapReduce с ProcessPoolExecutor

```python
from concurrent.futures import ProcessPoolExecutor
from functools import reduce
from collections import Counter

def mapper(chunk):
    """Map: подсчет символов в чанке"""
    return Counter(chunk)

def reducer(counter1, counter2):
    """Reduce: объединение счетчиков"""
    return counter1 + counter2

if __name__ == "__main__":
    # Большие данные разбиваем на чанки
    data = "abcdefghij" * 100000
    chunk_size = len(data) // 4
    chunks = [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]

    with ProcessPoolExecutor(max_workers=4) as executor:
        # Map фаза
        mapped_results = list(executor.map(mapper, chunks))

    # Reduce фаза
    final_result = reduce(reducer, mapped_results)

    print(f"Подсчет символов: {dict(final_result)}")
```

## Асинхронный MapReduce с asyncio

### Базовая реализация

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from functools import reduce

def cpu_map_function(data):
    """CPU-интенсивная map функция"""
    result = 0
    for i in data:
        result += i ** 2
    return result

async def async_map_reduce(data_chunks, executor):
    """Асинхронный MapReduce"""
    loop = asyncio.get_event_loop()

    # Map фаза - параллельно в процессах
    map_tasks = [
        loop.run_in_executor(executor, cpu_map_function, chunk)
        for chunk in data_chunks
    ]
    mapped_results = await asyncio.gather(*map_tasks)

    # Reduce фаза
    final_result = sum(mapped_results)

    return final_result

async def main():
    # Подготовка данных
    data = list(range(10_000_000))
    chunk_size = len(data) // 4
    chunks = [data[i:i+chunk_size] for i in range(0, len(data), chunk_size)]

    with ProcessPoolExecutor(max_workers=4) as executor:
        result = await async_map_reduce(chunks, executor)
        print(f"Сумма квадратов: {result}")

if __name__ == "__main__":
    asyncio.run(main())
```

### MapReduce с комбинированием I/O и CPU

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
import aiohttp
import re
from collections import Counter

def count_words(text):
    """CPU-bound: подсчет слов"""
    words = re.findall(r'\w+', text.lower())
    return Counter(words)

async def fetch_url(session, url):
    """I/O-bound: загрузка страницы"""
    async with session.get(url) as response:
        return await response.text()

async def async_word_count_mapreduce(urls):
    """
    MapReduce для подсчета слов на веб-страницах
    1. Загружаем страницы (I/O-bound, asyncio)
    2. Подсчитываем слова (CPU-bound, ProcessPoolExecutor)
    3. Объединяем результаты (Reduce)
    """
    loop = asyncio.get_event_loop()

    # Фаза 1: Загрузка (I/O-bound)
    async with aiohttp.ClientSession() as session:
        fetch_tasks = [fetch_url(session, url) for url in urls]
        pages = await asyncio.gather(*fetch_tasks, return_exceptions=True)

    # Фильтруем ошибки
    valid_pages = [p for p in pages if isinstance(p, str)]

    # Фаза 2: Map (CPU-bound)
    with ProcessPoolExecutor() as executor:
        map_tasks = [
            loop.run_in_executor(executor, count_words, page)
            for page in valid_pages
        ]
        word_counts = await asyncio.gather(*map_tasks)

    # Фаза 3: Reduce
    total_counts = Counter()
    for counts in word_counts:
        total_counts += counts

    return total_counts

async def main():
    urls = [
        "https://httpbin.org/html",
        "https://httpbin.org/robots.txt",
    ]

    result = await async_word_count_mapreduce(urls)
    print("Топ-10 слов:")
    for word, count in result.most_common(10):
        print(f"  {word}: {count}")

if __name__ == "__main__":
    asyncio.run(main())
```

## Реализация класса MapReduce

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from typing import Callable, Iterable, Any, TypeVar
from functools import reduce

T = TypeVar('T')
R = TypeVar('R')

class AsyncMapReduce:
    """Универсальный асинхронный MapReduce"""

    def __init__(self, max_workers: int = None):
        self.max_workers = max_workers

    async def execute(
        self,
        data: Iterable[T],
        map_func: Callable[[T], R],
        reduce_func: Callable[[R, R], R],
        initial: R = None
    ) -> R:
        """
        Выполнить MapReduce

        Args:
            data: Входные данные
            map_func: Функция для map-фазы
            reduce_func: Функция для reduce-фазы
            initial: Начальное значение для reduce

        Returns:
            Результат reduce-фазы
        """
        loop = asyncio.get_event_loop()
        data_list = list(data)

        with ProcessPoolExecutor(max_workers=self.max_workers) as executor:
            # Map фаза
            map_tasks = [
                loop.run_in_executor(executor, map_func, item)
                for item in data_list
            ]
            mapped = await asyncio.gather(*map_tasks)

        # Reduce фаза
        if initial is not None:
            return reduce(reduce_func, mapped, initial)
        return reduce(reduce_func, mapped)

    async def execute_chunked(
        self,
        data: Iterable[T],
        chunk_size: int,
        map_func: Callable[[list[T]], R],
        reduce_func: Callable[[R, R], R],
        initial: R = None
    ) -> R:
        """
        MapReduce с разбиением на чанки
        """
        data_list = list(data)
        chunks = [
            data_list[i:i+chunk_size]
            for i in range(0, len(data_list), chunk_size)
        ]

        return await self.execute(chunks, map_func, reduce_func, initial)


# Пример использования
def square_sum(numbers):
    """Map: сумма квадратов"""
    return sum(x ** 2 for x in numbers)

def add(a, b):
    """Reduce: сложение"""
    return a + b

async def main():
    mr = AsyncMapReduce(max_workers=4)

    data = list(range(1_000_000))

    result = await mr.execute_chunked(
        data=data,
        chunk_size=250_000,
        map_func=square_sum,
        reduce_func=add,
        initial=0
    )

    print(f"Результат: {result}")

if __name__ == "__main__":
    asyncio.run(main())
```

## Распределенный MapReduce с shuffle

В более сложных сценариях между Map и Reduce есть фаза **Shuffle** (группировка по ключам):

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from collections import defaultdict
from itertools import chain

def mapper(document):
    """
    Map: (документ) -> [(слово, 1), (слово, 1), ...]
    """
    words = document.lower().split()
    return [(word.strip('.,!?'), 1) for word in words if word.strip('.,!?')]

def reducer(word_counts):
    """
    Reduce: [(слово, 1), (слово, 1), ...] -> (слово, сумма)
    """
    word, counts = word_counts
    return (word, sum(counts))

def shuffle(mapped_results):
    """
    Shuffle: группировка по ключу
    [(k1, v1), (k2, v2), (k1, v3)] -> {k1: [v1, v3], k2: [v2]}
    """
    grouped = defaultdict(list)
    for key, value in chain.from_iterable(mapped_results):
        grouped[key].append(value)
    return list(grouped.items())

async def word_count_mapreduce(documents):
    """Полный MapReduce с shuffle"""
    loop = asyncio.get_event_loop()

    with ProcessPoolExecutor() as executor:
        # MAP
        map_tasks = [
            loop.run_in_executor(executor, mapper, doc)
            for doc in documents
        ]
        mapped = await asyncio.gather(*map_tasks)

        # SHUFFLE (в главном процессе)
        shuffled = shuffle(mapped)

        # REDUCE
        reduce_tasks = [
            loop.run_in_executor(executor, reducer, item)
            for item in shuffled
        ]
        reduced = await asyncio.gather(*reduce_tasks)

    return dict(reduced)

async def main():
    documents = [
        "Python is a great programming language",
        "Python is used for web development",
        "Python is popular for data science",
        "Java is also a great language",
        "Programming is fun with Python",
    ]

    result = await word_count_mapreduce(documents)

    # Сортировка по частоте
    sorted_words = sorted(result.items(), key=lambda x: x[1], reverse=True)
    print("Подсчет слов:")
    for word, count in sorted_words[:10]:
        print(f"  {word}: {count}")

if __name__ == "__main__":
    asyncio.run(main())
```

## Практический пример: анализ логов

```python
import asyncio
from concurrent.futures import ProcessPoolExecutor
from collections import Counter
import re
from datetime import datetime

def parse_log_chunk(log_lines):
    """
    Map: парсинг чанка логов
    Возвращает статистику: {status_code: count, hour: count}
    """
    status_counts = Counter()
    hour_counts = Counter()

    log_pattern = r'(\d+\.\d+\.\d+\.\d+).*\[(\d+/\w+/\d+):(\d+):\d+:\d+.*\] "(\w+) ([^"]+)" (\d+)'

    for line in log_lines:
        match = re.search(log_pattern, line)
        if match:
            hour = int(match.group(3))
            status = match.group(6)

            status_counts[status] += 1
            hour_counts[hour] += 1

    return {'status': status_counts, 'hours': hour_counts}

def merge_stats(stats1, stats2):
    """Reduce: объединение статистик"""
    return {
        'status': stats1['status'] + stats2['status'],
        'hours': stats1['hours'] + stats2['hours']
    }

async def analyze_logs(log_file_path, chunk_size=10000):
    """Анализ логов с MapReduce"""
    loop = asyncio.get_event_loop()

    # Чтение и разбиение на чанки
    with open(log_file_path, 'r') as f:
        lines = f.readlines()

    chunks = [
        lines[i:i+chunk_size]
        for i in range(0, len(lines), chunk_size)
    ]

    print(f"Обрабатываем {len(lines)} строк в {len(chunks)} чанках")

    with ProcessPoolExecutor() as executor:
        # Map
        map_tasks = [
            loop.run_in_executor(executor, parse_log_chunk, chunk)
            for chunk in chunks
        ]
        mapped = await asyncio.gather(*map_tasks)

    # Reduce
    from functools import reduce
    final_stats = reduce(merge_stats, mapped)

    return final_stats

async def main():
    # Для демонстрации создадим тестовый лог
    test_log = """
127.0.0.1 - - [28/Dec/2024:10:15:32 +0000] "GET /api/users HTTP/1.1" 200
127.0.0.1 - - [28/Dec/2024:10:15:33 +0000] "POST /api/login HTTP/1.1" 200
127.0.0.1 - - [28/Dec/2024:11:20:45 +0000] "GET /api/products HTTP/1.1" 404
127.0.0.1 - - [28/Dec/2024:11:25:12 +0000] "GET /api/users HTTP/1.1" 500
127.0.0.1 - - [28/Dec/2024:12:30:00 +0000] "GET /api/orders HTTP/1.1" 200
""".strip()

    # Сохраняем тестовый лог
    with open('/tmp/test_access.log', 'w') as f:
        f.write(test_log)

    stats = await analyze_logs('/tmp/test_access.log', chunk_size=2)

    print("\nСтатистика по статус-кодам:")
    for status, count in stats['status'].most_common():
        print(f"  {status}: {count}")

    print("\nСтатистика по часам:")
    for hour, count in sorted(stats['hours'].items()):
        print(f"  {hour:02d}:00 - {count} запросов")

if __name__ == "__main__":
    asyncio.run(main())
```

## Оптимизация MapReduce

### 1. Оптимальный размер чанков

```python
import multiprocessing

def calculate_chunk_size(data_size, worker_count=None):
    """Расчет оптимального размера чанка"""
    if worker_count is None:
        worker_count = multiprocessing.cpu_count()

    # Минимум 1 чанк на воркер, максимум 4
    chunks_per_worker = 4
    total_chunks = worker_count * chunks_per_worker

    return max(1, data_size // total_chunks)
```

### 2. Локальное предварительное агрегирование

```python
def mapper_with_combiner(chunk):
    """
    Map с локальным combiner:
    Агрегируем данные локально перед отправкой
    """
    local_counts = Counter()
    for item in chunk:
        # Обработка
        local_counts[item] += 1
    return local_counts  # Уже агрегировано!
```

### 3. Избегание сериализации больших объектов

```python
# НЕПРАВИЛЬНО: передаем большой DataFrame
def bad_mapper(large_dataframe):
    return process(large_dataframe)  # Сериализация дорогая!

# ПРАВИЛЬНО: передаем путь к данным
def good_mapper(file_path):
    df = pd.read_csv(file_path)  # Загружаем в процессе
    return process(df)
```

## Best Practices

1. **Разбивайте данные на равные части** — для равномерной нагрузки
2. **Минимизируйте передачу данных** — обрабатывайте локально
3. **Используйте combiner** — предварительную агрегацию
4. **Обрабатывайте ошибки** — не теряйте данные при сбоях
5. **Мониторьте производительность** — находите узкие места

---

[prev: 03-process-pool-executors.md](./03-process-pool-executors.md) | [next: 05-shared-data.md](./05-shared-data.md)
