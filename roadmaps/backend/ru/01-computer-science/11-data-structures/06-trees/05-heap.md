# Куча (Heap)

[prev: 04-balanced-trees](./04-balanced-trees.md) | [next: 06-trie](./06-trie.md)
---

## Определение

**Куча (Heap)** — это специализированная древовидная структура данных, представляющая собой полное бинарное дерево, удовлетворяющее **свойству кучи**: значение каждого узла не меньше (max-heap) или не больше (min-heap) значений его детей.

```
Max-Heap:                    Min-Heap:
       90                        10
      /  \                      /  \
    80    70                  20    30
   / \   / \                 / \   / \
  50 60 40 30               40 50 60 70

Родитель ≥ дети            Родитель ≤ дети
```

## Зачем нужно

### Практическое применение:
- **Priority Queue** — приоритетные очереди
- **Heapsort** — сортировка за O(n log n)
- **Top-K элементов** — поиск K наибольших/наименьших
- **Median of stream** — медиана потока данных
- **Алгоритм Дейкстры** — кратчайшие пути
- **Алгоритм Прима** — минимальное остовное дерево
- **Планировщики задач** — операционные системы

## Свойства кучи

### 1. Полное бинарное дерево

```
Все уровни заполнены, кроме последнего.
Последний уровень заполняется слева направо.

         90
        /  \
      80    70
     / \   /
    50 60 40
```

### 2. Свойство кучи (Heap Property)

```
Max-Heap: parent.value ≥ child.value
Min-Heap: parent.value ≤ child.value

Важно: нет упорядоченности между siblings!
В max-heap: 80 > 70, но 80 и 70 — не упорядочены между собой.
```

### 3. Представление в массиве

```
Дерево:
          90 [0]
        /      \
    80 [1]    70 [2]
    /   \     /   \
 50[3] 60[4] 40[5] 30[6]

Массив: [90, 80, 70, 50, 60, 40, 30]
Index:    0   1   2   3   4   5   6

Формулы для индекса i:
- Parent: (i - 1) // 2
- Left child: 2 * i + 1
- Right child: 2 * i + 2
```

## Основные операции

### Sift Up (Подъём)

```python
def sift_up(self, i):
    """
    Поднимает элемент вверх, если он нарушает свойство кучи.
    Используется после вставки.
    """
    while i > 0:
        parent = (i - 1) // 2

        if self.heap[i] > self.heap[parent]:  # для max-heap
            self.heap[i], self.heap[parent] = self.heap[parent], self.heap[i]
            i = parent
        else:
            break
```

```
Sift Up после insert(85):

[90, 80, 70, 50, 60, 40, 85]
                        ↑
                       new

85 > 70? Да → swap
[90, 80, 85, 50, 60, 40, 70]
         ↑

85 > 90? Нет → stop
```

### Sift Down (Спуск)

```python
def sift_down(self, i):
    """
    Опускает элемент вниз, если он нарушает свойство кучи.
    Используется после извлечения корня.
    """
    size = len(self.heap)

    while True:
        largest = i
        left = 2 * i + 1
        right = 2 * i + 2

        if left < size and self.heap[left] > self.heap[largest]:
            largest = left
        if right < size and self.heap[right] > self.heap[largest]:
            largest = right

        if largest != i:
            self.heap[i], self.heap[largest] = self.heap[largest], self.heap[i]
            i = largest
        else:
            break
```

```
Sift Down после extract_max:

Перемещаем последний элемент в корень:
[30, 80, 70, 50, 60, 40]  (90 удалён, 30 в корне)
 ↓

30 < max(80, 70)? Да, 80 больше → swap
[80, 30, 70, 50, 60, 40]
     ↓

30 < max(50, 60)? Да, 60 больше → swap
[80, 60, 70, 50, 30, 40]
                 ↓

30 — лист, stop
```

## Полная реализация

```python
class MaxHeap:
    def __init__(self):
        self.heap = []

    def __len__(self):
        return len(self.heap)

    def is_empty(self):
        return len(self.heap) == 0

    def insert(self, value):
        """Вставка элемента - O(log n)"""
        self.heap.append(value)
        self._sift_up(len(self.heap) - 1)

    def extract_max(self):
        """Извлечение максимума - O(log n)"""
        if self.is_empty():
            raise IndexError("Heap is empty")

        if len(self.heap) == 1:
            return self.heap.pop()

        max_val = self.heap[0]
        self.heap[0] = self.heap.pop()  # последний элемент в корень
        self._sift_down(0)

        return max_val

    def peek(self):
        """Просмотр максимума - O(1)"""
        if self.is_empty():
            raise IndexError("Heap is empty")
        return self.heap[0]

    def _sift_up(self, i):
        while i > 0:
            parent = (i - 1) // 2
            if self.heap[i] > self.heap[parent]:
                self.heap[i], self.heap[parent] = self.heap[parent], self.heap[i]
                i = parent
            else:
                break

    def _sift_down(self, i):
        size = len(self.heap)
        while True:
            largest = i
            left = 2 * i + 1
            right = 2 * i + 2

            if left < size and self.heap[left] > self.heap[largest]:
                largest = left
            if right < size and self.heap[right] > self.heap[largest]:
                largest = right

            if largest != i:
                self.heap[i], self.heap[largest] = self.heap[largest], self.heap[i]
                i = largest
            else:
                break

    @staticmethod
    def heapify(arr):
        """Построение кучи из массива - O(n)"""
        heap = MaxHeap()
        heap.heap = arr[:]

        # Начинаем с последнего не-листового узла
        for i in range(len(arr) // 2 - 1, -1, -1):
            heap._sift_down(i)

        return heap
```

### Min-Heap

```python
class MinHeap:
    """Аналогично MaxHeap, но с обратными сравнениями"""

    def _sift_up(self, i):
        while i > 0:
            parent = (i - 1) // 2
            if self.heap[i] < self.heap[parent]:  # < вместо >
                self.heap[i], self.heap[parent] = self.heap[parent], self.heap[i]
                i = parent
            else:
                break

    def _sift_down(self, i):
        size = len(self.heap)
        while True:
            smallest = i
            left = 2 * i + 1
            right = 2 * i + 2

            if left < size and self.heap[left] < self.heap[smallest]:
                smallest = left
            if right < size and self.heap[right] < self.heap[smallest]:
                smallest = right

            if smallest != i:
                self.heap[i], self.heap[smallest] = self.heap[smallest], self.heap[i]
                i = smallest
            else:
                break
```

## Использование heapq в Python

```python
import heapq

# heapq — это min-heap!

# Создание кучи
heap = []
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 2)
# heap = [1, 3, 2]

# Извлечение минимума
min_val = heapq.heappop(heap)  # 1

# Просмотр минимума
min_val = heap[0]

# Построение кучи из списка
arr = [3, 1, 4, 1, 5, 9, 2, 6]
heapq.heapify(arr)  # O(n)

# Push + Pop за одну операцию
heapq.heappushpop(heap, value)  # push, затем pop
heapq.heapreplace(heap, value)  # pop, затем push

# N наименьших/наибольших
heapq.nsmallest(3, arr)  # [1, 1, 2]
heapq.nlargest(3, arr)   # [9, 6, 5]

# Max-heap через отрицание
max_heap = []
heapq.heappush(max_heap, -value)
max_val = -heapq.heappop(max_heap)
```

## Анализ сложности

| Операция | Время | Примечание |
|----------|-------|------------|
| insert | O(log n) | sift up |
| extract_max/min | O(log n) | sift down |
| peek | O(1) | доступ к корню |
| heapify | **O(n)** | не O(n log n)! |
| increase/decrease key | O(log n) | sift up/down |

### Почему heapify за O(n)?

```
Для n элементов:
- n/2 листьев: 0 операций каждый
- n/4 узлов на уровне выше: 1 операция
- n/8 узлов: 2 операции
- ...

Сумма = O(n)
```

## Примеры с разбором

### Пример 1: Heapsort

```python
def heapsort(arr):
    """Сортировка кучей - O(n log n)"""
    # Шаг 1: Построение max-heap
    n = len(arr)
    for i in range(n // 2 - 1, -1, -1):
        sift_down(arr, i, n)

    # Шаг 2: Извлечение элементов по одному
    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]  # max в конец
        sift_down(arr, 0, i)

    return arr

def sift_down(arr, i, size):
    while True:
        largest = i
        left = 2 * i + 1
        right = 2 * i + 2

        if left < size and arr[left] > arr[largest]:
            largest = left
        if right < size and arr[right] > arr[largest]:
            largest = right

        if largest != i:
            arr[i], arr[largest] = arr[largest], arr[i]
            i = largest
        else:
            break
```

### Пример 2: Top K элементов

```python
import heapq

def top_k_largest(nums, k):
    """
    Находит k наибольших элементов.
    Используем min-heap размера k.
    Время: O(n log k)
    """
    if k >= len(nums):
        return nums

    heap = nums[:k]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num > heap[0]:
            heapq.heapreplace(heap, num)

    return heap

def top_k_smallest(nums, k):
    """
    Находит k наименьших элементов.
    Используем max-heap размера k.
    """
    if k >= len(nums):
        return nums

    # Max-heap через отрицание
    heap = [-x for x in nums[:k]]
    heapq.heapify(heap)

    for num in nums[k:]:
        if num < -heap[0]:
            heapq.heapreplace(heap, -num)

    return [-x for x in heap]
```

### Пример 3: Медиана потока данных

```python
import heapq

class MedianFinder:
    """
    Поддерживает медиану потока чисел.
    Две кучи: max-heap для меньшей половины,
    min-heap для большей половины.
    """
    def __init__(self):
        self.small = []  # max-heap (отрицательные)
        self.large = []  # min-heap

    def add_num(self, num):
        # Добавляем в small
        heapq.heappush(self.small, -num)

        # Балансируем: max(small) ≤ min(large)
        if self.small and self.large and -self.small[0] > self.large[0]:
            val = -heapq.heappop(self.small)
            heapq.heappush(self.large, val)

        # Балансируем размеры
        if len(self.small) > len(self.large) + 1:
            val = -heapq.heappop(self.small)
            heapq.heappush(self.large, val)
        elif len(self.large) > len(self.small) + 1:
            val = heapq.heappop(self.large)
            heapq.heappush(self.small, -val)

    def find_median(self):
        if len(self.small) > len(self.large):
            return -self.small[0]
        elif len(self.large) > len(self.small):
            return self.large[0]
        else:
            return (-self.small[0] + self.large[0]) / 2
```

### Пример 4: Слияние K отсортированных массивов

```python
import heapq

def merge_k_sorted(arrays):
    """
    Сливает k отсортированных массивов.
    Время: O(n log k)
    """
    result = []
    heap = []  # (value, array_index, element_index)

    # Добавляем первый элемент каждого массива
    for i, arr in enumerate(arrays):
        if arr:
            heapq.heappush(heap, (arr[0], i, 0))

    while heap:
        val, arr_idx, elem_idx = heapq.heappop(heap)
        result.append(val)

        # Добавляем следующий элемент из того же массива
        if elem_idx + 1 < len(arrays[arr_idx]):
            next_val = arrays[arr_idx][elem_idx + 1]
            heapq.heappush(heap, (next_val, arr_idx, elem_idx + 1))

    return result
```

### Пример 5: K ближайших точек

```python
import heapq

def k_closest_points(points, k):
    """
    Находит k ближайших точек к началу координат.
    """
    # Max-heap (храним отрицательные расстояния)
    heap = []

    for x, y in points:
        dist = -(x*x + y*y)  # отрицательное для max-heap

        if len(heap) < k:
            heapq.heappush(heap, (dist, x, y))
        elif dist > heap[0][0]:  # ближе, чем самая дальняя
            heapq.heapreplace(heap, (dist, x, y))

    return [(x, y) for _, x, y in heap]
```

## Типичные ошибки

### 1. heapq — это min-heap

```python
import heapq

# НЕПРАВИЛЬНО — ожидаем max
heap = [3, 1, 2]
heapq.heapify(heap)
heapq.heappop(heap)  # 1, не 3!

# ПРАВИЛЬНО — используем отрицание для max-heap
heap = [-3, -1, -2]
heapq.heapify(heap)
-heapq.heappop(heap)  # 3
```

### 2. Забытый heapify

```python
# НЕПРАВИЛЬНО
arr = [3, 1, 4, 1, 5]
heapq.heappop(arr)  # 3, но это не минимум!

# ПРАВИЛЬНО
arr = [3, 1, 4, 1, 5]
heapq.heapify(arr)  # сначала heapify!
heapq.heappop(arr)  # 1
```

### 3. Сравнение несравнимых объектов

```python
# НЕПРАВИЛЬНО — если приоритеты равны, сравниваются объекты
heap = []
heapq.heappush(heap, (1, {"a": 1}))
heapq.heappush(heap, (1, {"b": 2}))  # TypeError!

# ПРАВИЛЬНО — добавляем уникальный счётчик
import itertools
counter = itertools.count()
heapq.heappush(heap, (1, next(counter), {"a": 1}))
```

### 4. Изменение элемента в куче

```python
# heapq не поддерживает decrease/increase key напрямую

# Решение 1: Lazy deletion
# Помечаем старый элемент как удалённый, добавляем новый

# Решение 2: Rebuild heap
heap[i] = new_value
heapq.heapify(heap)  # O(n), но просто
```

## Сравнение структур

| Операция | Отсорт. массив | Неотсорт. массив | Heap |
|----------|----------------|------------------|------|
| Insert | O(n) | O(1) | O(log n) |
| Find min/max | O(1) | O(n) | O(1) |
| Extract min/max | O(1) | O(n) | O(log n) |
| Build | O(n log n) | O(n) | O(n) |

## Когда использовать Heap

**Используй heap, когда:**
- Частое извлечение min/max
- Приоритетная очередь
- Top-K элементов
- Слияние сортированных последовательностей

**НЕ используй heap, когда:**
- Нужен поиск произвольного элемента
- Нужна сортировка (можно, но есть методы лучше)
- Нужно удаление по значению

---
[prev: 04-balanced-trees](./04-balanced-trees.md) | [next: 06-trie](./06-trie.md)