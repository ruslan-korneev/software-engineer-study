# Приоритетная очередь (Priority Queue)

[prev: 02-deque](./02-deque.md) | [next: 01-hash-functions](../05-hash-tables/01-hash-functions.md)

---
## Определение

**Приоритетная очередь** — это абстрактная структура данных, в которой каждый элемент имеет приоритет. Элементы извлекаются в порядке приоритета, а не в порядке добавления. Обычно реализуется на основе **кучи (heap)**.

```
Обычная очередь (FIFO):          Приоритетная очередь:
Порядок добавления:              Порядок по приоритету:
[A(1), B(3), C(2)]               [B(3), C(2), A(1)]
    ↓                                ↓
Выход: A, B, C                   Выход: B, C, A
(по порядку добавления)          (по приоритету)
```

## Зачем нужно

### Практическое применение:
- **Планировщики ОС** — процессы с разными приоритетами
- **Алгоритм Дейкстры** — поиск кратчайшего пути
- **Алгоритм A*** — поиск пути в играх
- **Huffman coding** — сжатие данных
- **Merge k sorted lists** — слияние отсортированных списков
- **Task scheduling** — выполнение задач по приоритету
- **Event-driven simulation** — обработка событий по времени
- **Медиана потока** — нахождение медианы в реальном времени

## Как работает

### Два типа приоритетных очередей

```
MIN-HEAP (минимальный элемент наверху):
     1
    / \
   3   2
  / \
 5   4

Извлечение: 1, 2, 3, 4, 5 (по возрастанию)


MAX-HEAP (максимальный элемент наверху):
     5
    / \
   4   3
  / \
 1   2

Извлечение: 5, 4, 3, 2, 1 (по убыванию)
```

### Структура кучи

Куча — это полное бинарное дерево, хранящееся в массиве:

```
Дерево:           Массив:
       1          [1, 3, 2, 5, 4, 6, 7]
      / \          0  1  2  3  4  5  6
     3   2
    / \ / \
   5  4 6  7

Формулы:
- Родитель i: (i - 1) // 2
- Левый ребёнок i: 2 * i + 1
- Правый ребёнок i: 2 * i + 2
```

### Основные операции

```
INSERT (добавление):
1. Добавляем элемент в конец массива
2. "Всплываем" (sift up) — меняем с родителем, если нарушено свойство кучи

     1                    1                    1
    / \    insert(0)     / \    sift up       / \
   3   2   ────────►    3   2   ────────►    0   2
  / \                  / \ /                / \ /
 5   4                5  4 0               5  4 3


EXTRACT (извлечение минимума/максимума):
1. Сохраняем корень
2. Перемещаем последний элемент в корень
3. "Просеиваем" (sift down) — меняем с меньшим ребёнком

     1                    4                    2
    / \    extract       / \    sift down     / \
   3   2   ────────►    3   2   ────────►    3   4
  / \                  /                    /
 5   4                5                    5
```

## Псевдокод основных операций

### Реализация Min-Heap

```python
class MinHeap:
    def __init__(self):
        self.heap = []

    def __len__(self):
        return len(self.heap)

    def is_empty(self):
        return len(self.heap) == 0

    def _parent(self, i):
        return (i - 1) // 2

    def _left_child(self, i):
        return 2 * i + 1

    def _right_child(self, i):
        return 2 * i + 2

    def _swap(self, i, j):
        self.heap[i], self.heap[j] = self.heap[j], self.heap[i]

    def _sift_up(self, i):
        """Всплытие элемента вверх"""
        while i > 0:
            parent = self._parent(i)
            if self.heap[i] < self.heap[parent]:
                self._swap(i, parent)
                i = parent
            else:
                break

    def _sift_down(self, i):
        """Просеивание элемента вниз"""
        size = len(self.heap)
        while True:
            smallest = i
            left = self._left_child(i)
            right = self._right_child(i)

            if left < size and self.heap[left] < self.heap[smallest]:
                smallest = left
            if right < size and self.heap[right] < self.heap[smallest]:
                smallest = right

            if smallest != i:
                self._swap(i, smallest)
                i = smallest
            else:
                break

    def insert(self, value):
        """Добавление элемента - O(log n)"""
        self.heap.append(value)
        self._sift_up(len(self.heap) - 1)

    def extract_min(self):
        """Извлечение минимума - O(log n)"""
        if self.is_empty():
            raise IndexError("Heap is empty")

        if len(self.heap) == 1:
            return self.heap.pop()

        min_val = self.heap[0]
        self.heap[0] = self.heap.pop()  # перемещаем последний в корень
        self._sift_down(0)

        return min_val

    def peek(self):
        """Просмотр минимума - O(1)"""
        if self.is_empty():
            raise IndexError("Heap is empty")
        return self.heap[0]

    def heapify(self, arr):
        """Построение кучи из массива - O(n)"""
        self.heap = arr[:]
        # Начинаем с последнего не-листового узла
        for i in range(len(self.heap) // 2 - 1, -1, -1):
            self._sift_down(i)
```

### Использование heapq в Python

```python
import heapq

# Min-heap (по умолчанию)
heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)

heapq.heappop(heap)  # 1
heapq.heappop(heap)  # 2
heapq.heappop(heap)  # 3

# Просмотр минимума без удаления
heap[0]  # peek

# Построение кучи из списка
arr = [3, 1, 4, 1, 5, 9, 2, 6]
heapq.heapify(arr)  # O(n)

# Push + Pop (эффективнее отдельных операций)
heapq.heappushpop(heap, value)  # push, затем pop
heapq.heapreplace(heap, value)  # pop, затем push

# N наименьших/наибольших
heapq.nsmallest(3, arr)  # [1, 1, 2]
heapq.nlargest(3, arr)   # [9, 6, 5]

# Max-heap через отрицание
max_heap = []
heapq.heappush(max_heap, -value)  # добавляем с минусом
-heapq.heappop(max_heap)          # извлекаем и меняем знак
```

### Priority Queue с произвольными объектами

```python
import heapq
from dataclasses import dataclass, field
from typing import Any

@dataclass(order=True)
class PrioritizedItem:
    priority: int
    item: Any = field(compare=False)  # не участвует в сравнении

# Использование
pq = []
heapq.heappush(pq, PrioritizedItem(2, "task B"))
heapq.heappush(pq, PrioritizedItem(1, "task A"))
heapq.heappush(pq, PrioritizedItem(3, "task C"))

item = heapq.heappop(pq)  # PrioritizedItem(priority=1, item='task A')
```

### Использование queue.PriorityQueue (потокобезопасная)

```python
from queue import PriorityQueue

pq = PriorityQueue()

pq.put((2, "task B"))
pq.put((1, "task A"))
pq.put((3, "task C"))

pq.get()  # (1, 'task A')
pq.get()  # (2, 'task B')
pq.get()  # (3, 'task C')

pq.empty()  # True
pq.qsize()  # размер
```

## Анализ сложности

| Операция | Время | Пояснение |
|----------|-------|-----------|
| insert | O(log n) | sift up до корня |
| extract_min/max | O(log n) | sift down до листа |
| peek | O(1) | доступ к корню |
| heapify | O(n) | не O(n log n)! |
| decrease_key | O(log n) | sift up |
| increase_key | O(log n) | sift down |

### Почему heapify за O(n)?

```
Для n элементов:
- n/2 листьев — 0 операций каждый
- n/4 узлов на предпоследнем уровне — 1 операция каждый
- n/8 узлов — 2 операции каждый
- ...

Сумма: n/4 * 1 + n/8 * 2 + n/16 * 3 + ...
     = n * (1/4 + 2/8 + 3/16 + ...)
     ≤ n * 2
     = O(n)
```

## Примеры с разбором

### Пример 1: Алгоритм Дейкстры

```python
import heapq
from collections import defaultdict

def dijkstra(graph, start):
    """
    Находит кратчайшие расстояния от start до всех вершин.
    graph: {node: [(neighbor, weight), ...]}
    """
    distances = {start: 0}
    pq = [(0, start)]  # (distance, node)

    while pq:
        current_dist, current = heapq.heappop(pq)

        # Пропускаем устаревшие записи
        if current_dist > distances.get(current, float('inf')):
            continue

        for neighbor, weight in graph[current]:
            distance = current_dist + weight

            if distance < distances.get(neighbor, float('inf')):
                distances[neighbor] = distance
                heapq.heappush(pq, (distance, neighbor))

    return distances

# Пример графа:
#     A --2-- B
#     |       |
#     1       3
#     |       |
#     C --1-- D

graph = {
    'A': [('B', 2), ('C', 1)],
    'B': [('A', 2), ('D', 3)],
    'C': [('A', 1), ('D', 1)],
    'D': [('B', 3), ('C', 1)]
}

dijkstra(graph, 'A')  # {'A': 0, 'C': 1, 'B': 2, 'D': 2}
```

### Пример 2: Слияние k отсортированных списков

```python
import heapq

def merge_k_sorted(lists):
    """
    Сливает k отсортированных списков в один.
    Время: O(n log k), где n — общее число элементов
    """
    result = []
    # (значение, индекс списка, индекс в списке)
    pq = []

    # Инициализация — добавляем первый элемент каждого списка
    for i, lst in enumerate(lists):
        if lst:
            heapq.heappush(pq, (lst[0], i, 0))

    while pq:
        val, list_idx, elem_idx = heapq.heappop(pq)
        result.append(val)

        # Добавляем следующий элемент из того же списка
        if elem_idx + 1 < len(lists[list_idx]):
            next_val = lists[list_idx][elem_idx + 1]
            heapq.heappush(pq, (next_val, list_idx, elem_idx + 1))

    return result

# Пример:
lists = [
    [1, 4, 7],
    [2, 5, 8],
    [3, 6, 9]
]
merge_k_sorted(lists)  # [1, 2, 3, 4, 5, 6, 7, 8, 9]
```

### Пример 3: K ближайших точек к началу координат

```python
import heapq

def k_closest(points, k):
    """
    Находит k ближайших точек к началу координат (0, 0).
    """
    # Max-heap размера k (через отрицание)
    max_heap = []

    for x, y in points:
        dist = x*x + y*y  # квадрат расстояния

        if len(max_heap) < k:
            heapq.heappush(max_heap, (-dist, x, y))
        elif dist < -max_heap[0][0]:
            heapq.heapreplace(max_heap, (-dist, x, y))

    return [(x, y) for _, x, y in max_heap]

# Пример:
points = [(1, 3), (-2, 2), (5, 8), (0, 1)]
k_closest(points, 2)  # [(0, 1), (-2, 2)]
```

### Пример 4: Медиана потока данных

```python
import heapq

class MedianFinder:
    """
    Находит медиану потока чисел.
    Использует два heap: max-heap для меньшей половины,
    min-heap для большей половины.
    """
    def __init__(self):
        self.small = []  # max-heap (отрицательные значения)
        self.large = []  # min-heap

    def add_num(self, num):
        """Добавляет число - O(log n)"""
        # Добавляем в small (max-heap)
        heapq.heappush(self.small, -num)

        # Балансировка: наибольший из small должен быть ≤ наименьшего из large
        if self.small and self.large and -self.small[0] > self.large[0]:
            val = -heapq.heappop(self.small)
            heapq.heappush(self.large, val)

        # Балансировка размеров: разница не более 1
        if len(self.small) > len(self.large) + 1:
            val = -heapq.heappop(self.small)
            heapq.heappush(self.large, val)
        elif len(self.large) > len(self.small) + 1:
            val = heapq.heappop(self.large)
            heapq.heappush(self.small, -val)

    def find_median(self):
        """Возвращает медиану - O(1)"""
        if len(self.small) > len(self.large):
            return -self.small[0]
        elif len(self.large) > len(self.small):
            return self.large[0]
        else:
            return (-self.small[0] + self.large[0]) / 2

# Пример:
mf = MedianFinder()
mf.add_num(1)  # median = 1
mf.add_num(2)  # median = 1.5
mf.add_num(3)  # median = 2
mf.find_median()  # 2
```

```
Визуализация для [1, 2, 3]:

После add(1):
small: [-1]    large: []
       ↑
    median = 1

После add(2):
small: [-1]    large: [2]
       ↑              ↑
    median = (1 + 2) / 2 = 1.5

После add(3):
small: [-2, -1]    large: [3]
       ↑
    median = 2 (из large, т.к. там больше)

Нет! После балансировки:
small: [-1]    large: [2, 3]
                     ↑
    median = 2
```

### Пример 5: Планировщик задач

```python
import heapq
from collections import Counter

def least_interval(tasks, n):
    """
    Минимальное время для выполнения задач с cooldown n.
    tasks = ["A","A","A","B","B","B"], n = 2

    A -> B -> idle -> A -> B -> idle -> A -> B
    Ответ: 8
    """
    # Подсчитываем частоту каждой задачи
    freq = Counter(tasks)

    # Max-heap по частоте
    max_heap = [-count for count in freq.values()]
    heapq.heapify(max_heap)

    time = 0
    cooldown = []  # (available_time, count)

    while max_heap or cooldown:
        time += 1

        if max_heap:
            count = heapq.heappop(max_heap) + 1  # выполняем задачу

            if count < 0:  # остались повторы
                cooldown.append((time + n, count))

        # Проверяем, какие задачи вышли из cooldown
        if cooldown and cooldown[0][0] == time:
            _, count = cooldown.pop(0)
            heapq.heappush(max_heap, count)

    return time
```

## Типичные ошибки

### 1. heapq — это min-heap, не max-heap

```python
import heapq

# НЕПРАВИЛЬНО — ожидаем max-heap
heap = [3, 1, 2]
heapq.heapify(heap)
heapq.heappop(heap)  # 1, не 3!

# ПРАВИЛЬНО — используем отрицание для max-heap
max_heap = [-x for x in [3, 1, 2]]
heapq.heapify(max_heap)
-heapq.heappop(max_heap)  # 3
```

### 2. Сравнение несравнимых объектов

```python
import heapq

# НЕПРАВИЛЬНО — если приоритеты равны, сравниваются объекты
pq = []
heapq.heappush(pq, (1, {"name": "A"}))
heapq.heappush(pq, (1, {"name": "B"}))  # TypeError!

# ПРАВИЛЬНО — добавляем уникальный счётчик
import itertools
counter = itertools.count()
heapq.heappush(pq, (1, next(counter), {"name": "A"}))
heapq.heappush(pq, (1, next(counter), {"name": "B"}))
```

### 3. Неправильное использование heapreplace

```python
import heapq

heap = [1, 2, 3]
heapq.heapify(heap)

# heapreplace сначала делает pop, потом push
result = heapq.heapreplace(heap, 0)  # result = 1, heap = [0, 2, 3]

# heappushpop сначала делает push, потом pop
result = heapq.heappushpop(heap, 0)  # result = 0, heap без изменений
```

### 4. Устаревшие записи в Dijkstra

```python
# НЕПРАВИЛЬНО — обрабатываем устаревшие записи
while pq:
    dist, node = heapq.heappop(pq)
    for neighbor, weight in graph[node]:
        # ...

# ПРАВИЛЬНО — пропускаем устаревшие
while pq:
    dist, node = heapq.heappop(pq)
    if dist > distances.get(node, float('inf')):
        continue  # устаревшая запись
    # ...
```

### 5. Изменение элемента в куче

```python
# heapq не поддерживает decrease_key напрямую

# Способ 1: пометить старую запись как недействительную
# Способ 2: добавить новую запись (lazy deletion)

# Lazy deletion:
pq = [(5, 'A'), (3, 'B')]
heapq.heapify(pq)

# "Уменьшаем" приоритет A до 1
heapq.heappush(pq, (1, 'A'))  # добавляем новую запись

# При извлечении проверяем актуальность
removed = set()
while pq:
    priority, item = heapq.heappop(pq)
    if item not in removed:
        removed.add(item)
        # обрабатываем item
```

## Сравнение реализаций

| Операция | Отсортированный массив | Несортированный массив | Куча |
|----------|------------------------|------------------------|------|
| Insert | O(n) | O(1) | O(log n) |
| Extract min/max | O(1) | O(n) | O(log n) |
| Peek | O(1) | O(n) | O(1) |
| Build | O(n log n) | O(n) | O(n) |

## Когда использовать Priority Queue

**Используй priority queue, когда:**
- Нужно извлекать элементы по приоритету
- Алгоритмы графов (Dijkstra, Prim, A*)
- Top-K элементов
- Планирование задач
- Слияние отсортированных последовательностей

**НЕ используй priority queue, когда:**
- Нужен FIFO порядок (обычная очередь)
- Нужен LIFO порядок (стек)
- Частый поиск произвольных элементов
- Нужно удалять произвольные элементы (используй TreeSet)

---

[prev: 02-deque](./02-deque.md) | [next: 01-hash-functions](../05-hash-tables/01-hash-functions.md)
