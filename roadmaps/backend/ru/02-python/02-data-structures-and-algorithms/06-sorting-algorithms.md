# Алгоритмы сортировки (Sorting Algorithms)

[prev: 05-recursion](./05-recursion.md) | [next: 03-modules](../03-modules.md)
---

Сортировка — упорядочивание элементов по определённому критерию.

## Обзор алгоритмов

| Алгоритм       | Лучшее     | Среднее    | Худшее     | Память   | Стабильность |
| -------------- | ---------- | ---------- | ---------- | -------- | ------------ |
| Bubble Sort    | O(n)       | O(n²)      | O(n²)      | O(1)     | Стабильный   |
| Selection Sort | O(n²)      | O(n²)      | O(n²)      | O(1)     | Нестабильный |
| Insertion Sort | O(n)       | O(n²)      | O(n²)      | O(1)     | Стабильный   |
| Merge Sort     | O(n log n) | O(n log n) | O(n log n) | O(n)     | Стабильный   |
| Quick Sort     | O(n log n) | O(n log n) | O(n²)      | O(log n) | Нестабильный |
| Heap Sort      | O(n log n) | O(n log n) | O(n log n) | O(1)     | Нестабильный |
| Counting Sort  | O(n + k)   | O(n + k)   | O(n + k)   | O(k)     | Стабильный   |
| Radix Sort     | O(nk)      | O(nk)      | O(nk)      | O(n + k) | Стабильный   |

**Стабильность** — сохраняется ли порядок равных элементов.

---

## Bubble Sort (Пузырьковая сортировка)

Идея: соседние элементы сравниваются и меняются местами, "всплывая" как пузырьки.

```python
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        swapped = False
        for j in range(n - 1 - i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
                swapped = True
        # Оптимизация: если обменов не было — массив отсортирован
        if not swapped:
            break
    return arr
```

```
Проход 1: [5, 3, 8, 1, 2]
          [3, 5, 8, 1, 2]  5 > 3, меняем
          [3, 5, 8, 1, 2]  5 < 8, ок
          [3, 5, 1, 8, 2]  8 > 1, меняем
          [3, 5, 1, 2, 8]  8 > 2, меняем ← 8 "всплыл"

Проход 2: [3, 5, 1, 2, 8]
          [3, 1, 5, 2, 8]
          [3, 1, 2, 5, 8]  ← 5 "всплыл"
...
```

**Сложность:** O(n²) — плохо для больших массивов.
**Когда использовать:** Учебные цели, почти отсортированные данные (O(n) с оптимизацией).

---

## Selection Sort (Сортировка выбором)

Идея: находим минимум в неотсортированной части, ставим в начало.

```python
def selection_sort(arr):
    n = len(arr)
    for i in range(n):
        min_idx = i
        for j in range(i + 1, n):
            if arr[j] < arr[min_idx]:
                min_idx = j
        arr[i], arr[min_idx] = arr[min_idx], arr[i]
    return arr
```

```
[5, 3, 8, 1, 2]
 ^           ^
 i         min=1

[1, 3, 8, 5, 2]  ← 1 на место
    ^        ^
    i      min=2

[1, 2, 8, 5, 3]  ← 2 на место
       ^     ^
       i   min=3
...
```

**Сложность:** O(n²) — всегда, даже на отсортированном массиве.
**Преимущество:** Минимум обменов — O(n).

---

## Insertion Sort (Сортировка вставками)

Идея: берём элемент и вставляем на правильное место в отсортированную часть.

```python
def insertion_sort(arr):
    for i in range(1, len(arr)):
        key = arr[i]
        j = i - 1
        # Сдвигаем элементы, пока не найдём место для key
        while j >= 0 and arr[j] > key:
            arr[j + 1] = arr[j]
            j -= 1
        arr[j + 1] = key
    return arr
```

```
[5, 3, 8, 1, 2]
    ^
    key=3, вставляем перед 5
[3, 5, 8, 1, 2]

[3, 5, 8, 1, 2]
       ^
       key=8, уже на месте

[3, 5, 8, 1, 2]
          ^
          key=1, вставляем в начало
[1, 3, 5, 8, 2]
...
```

**Сложность:** O(n²), но O(n) на почти отсортированных данных.
**Когда использовать:** Небольшие массивы, онлайн-сортировка (данные приходят по одному).

---

## Merge Sort (Сортировка слиянием)

Идея: "Разделяй и властвуй" — делим пополам, сортируем, сливаем.

```python
def merge_sort(arr):
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

    return merge(left, right)


def merge(left, right):
    result = []
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

```
[5, 3, 8, 1, 2]
       /     \
   [5, 3]   [8, 1, 2]
    / \       /   \
  [5] [3]   [8]  [1, 2]
    \ /       \   / \
   [3, 5]    [8] [1] [2]
      \        \ / \  /
       \      [1, 2]
        \        |
         \    [1, 2, 8]
          \     /
       [1, 2, 3, 5, 8]
```

**Сложность:** O(n log n) — всегда!
**Память:** O(n) — нужен дополнительный массив.
**Когда использовать:** Когда нужна гарантированная O(n log n), для связных списков, внешняя сортировка.

---

## Quick Sort (Быстрая сортировка)

Идея: выбираем опорный элемент (pivot), разделяем на меньшие и большие.

```python
def quick_sort(arr):
    if len(arr) <= 1:
        return arr

    pivot = arr[len(arr) // 2]  # или arr[0], или random
    left = [x for x in arr if x < pivot]
    middle = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]

    return quick_sort(left) + middle + quick_sort(right)
```

### In-place версия (без доп. памяти)

```python
def quick_sort_inplace(arr, low=0, high=None):
    if high is None:
        high = len(arr) - 1

    if low < high:
        pivot_idx = partition(arr, low, high)
        quick_sort_inplace(arr, low, pivot_idx - 1)
        quick_sort_inplace(arr, pivot_idx + 1, high)

    return arr


def partition(arr, low, high):
    pivot = arr[high]  # последний элемент как pivot
    i = low - 1

    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1
```

```
[5, 3, 8, 1, 2], pivot = 3 (середина)

left = [1, 2]      (< 3)
middle = [3]       (= 3)
right = [5, 8]     (> 3)

quick_sort([1, 2]) + [3] + quick_sort([5, 8])
[1, 2] + [3] + [5, 8]
= [1, 2, 3, 5, 8]
```

**Сложность:** O(n log n) в среднем, O(n²) в худшем (плохой pivot).
**Память:** O(log n) — стек рекурсии.
**Когда использовать:** Универсальный выбор, быстрее Merge Sort на практике (меньше памяти, лучше кэш).

### Выбор pivot

```python
# Плохо: всегда первый/последний — O(n²) на отсортированных данных
pivot = arr[0]

# Лучше: средний элемент
pivot = arr[len(arr) // 2]

# Ещё лучше: медиана трёх
def median_of_three(arr, low, high):
    mid = (low + high) // 2
    if arr[low] > arr[mid]:
        arr[low], arr[mid] = arr[mid], arr[low]
    if arr[low] > arr[high]:
        arr[low], arr[high] = arr[high], arr[low]
    if arr[mid] > arr[high]:
        arr[mid], arr[high] = arr[high], arr[mid]
    return arr[mid]

# Случайный pivot — рандомизированный QuickSort
import random
pivot = arr[random.randint(low, high)]
```

---

## Heap Sort (Пирамидальная сортировка)

Идея: строим max-heap, извлекаем максимумы.

```python
def heap_sort(arr):
    n = len(arr)

    # Строим max-heap
    for i in range(n // 2 - 1, -1, -1):
        heapify(arr, n, i)

    # Извлекаем элементы
    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]  # max в конец
        heapify(arr, i, 0)

    return arr


def heapify(arr, n, i):
    largest = i
    left = 2 * i + 1
    right = 2 * i + 2

    if left < n and arr[left] > arr[largest]:
        largest = left
    if right < n and arr[right] > arr[largest]:
        largest = right

    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify(arr, n, largest)
```

**Сложность:** O(n log n) — гарантированно.
**Память:** O(1) — in-place.
**Когда использовать:** Нужна гарантия O(n log n) без доп. памяти.

---

## Counting Sort (Сортировка подсчётом)

Идея: считаем количество каждого элемента — работает только для целых чисел!

```python
def counting_sort(arr):
    if not arr:
        return arr

    min_val = min(arr)
    max_val = max(arr)
    range_size = max_val - min_val + 1

    # Считаем количество каждого элемента
    count = [0] * range_size
    for num in arr:
        count[num - min_val] += 1

    # Восстанавливаем отсортированный массив
    result = []
    for i, c in enumerate(count):
        result.extend([i + min_val] * c)

    return result
```

```
arr = [4, 2, 2, 8, 3, 3, 1]
min = 1, max = 8, range = 8

count = [1, 2, 2, 1, 0, 0, 0, 1]
         1  2  3  4  5  6  7  8

result = [1, 2, 2, 3, 3, 4, 8]
```

**Сложность:** O(n + k), где k — диапазон значений.
**Память:** O(k).
**Когда использовать:** Целые числа с небольшим диапазоном.

---

## Radix Sort (Поразрядная сортировка)

Идея: сортируем по разрядам (справа налево) с помощью стабильной сортировки.

```python
def radix_sort(arr):
    if not arr:
        return arr

    max_val = max(arr)
    exp = 1

    while max_val // exp > 0:
        counting_sort_by_digit(arr, exp)
        exp *= 10

    return arr


def counting_sort_by_digit(arr, exp):
    n = len(arr)
    output = [0] * n
    count = [0] * 10

    for num in arr:
        digit = (num // exp) % 10
        count[digit] += 1

    # Кумулятивный подсчёт
    for i in range(1, 10):
        count[i] += count[i - 1]

    # Строим output (справа налево для стабильности)
    for i in range(n - 1, -1, -1):
        digit = (arr[i] // exp) % 10
        output[count[digit] - 1] = arr[i]
        count[digit] -= 1

    for i in range(n):
        arr[i] = output[i]
```

```
arr = [170, 45, 75, 90, 802, 24, 2, 66]

По единицам (exp=1):   [170, 90, 802, 2, 24, 45, 75, 66]
По десяткам (exp=10):  [802, 2, 24, 45, 66, 170, 75, 90]
По сотням (exp=100):   [2, 24, 45, 66, 75, 90, 170, 802]
```

**Сложность:** O(nk), где k — количество разрядов.
**Когда использовать:** Числа с фиксированным количеством разрядов, строки.

---

## Python: встроенная сортировка

Python использует **Timsort** — гибрид Merge Sort и Insertion Sort.

```python
# Сортировка списка (in-place)
arr = [5, 3, 8, 1, 2]
arr.sort()  # arr = [1, 2, 3, 5, 8]

# Сортировка по убыванию
arr.sort(reverse=True)  # [8, 5, 3, 2, 1]

# Создание нового списка
arr = [5, 3, 8, 1, 2]
sorted_arr = sorted(arr)  # arr не изменён

# С ключом сортировки
words = ["banana", "apple", "cherry"]
words.sort(key=len)  # ["apple", "banana", "cherry"]

# Сортировка объектов
users = [
    {"name": "Bob", "age": 25},
    {"name": "Alice", "age": 30},
    {"name": "Charlie", "age": 20}
]
users.sort(key=lambda x: x["age"])
# Отсортировано по возрасту: Charlie, Bob, Alice

# operator.itemgetter — быстрее lambda
from operator import itemgetter
users.sort(key=itemgetter("age"))

# Множественная сортировка
users.sort(key=lambda x: (x["age"], x["name"]))
```

### Timsort особенности

- **Сложность:** O(n log n) в худшем, O(n) на частично отсортированных данных
- **Стабильный:** да
- **Адаптивный:** использует существующий порядок (runs)
- **Память:** O(n)

```python
# Timsort эффективен на "почти отсортированных" данных
arr = list(range(1000)) + [0]  # 999 элементов по порядку + 1
sorted(arr)  # Очень быстро!
```

---

## Сравнение на практике

```python
import random
import time

def benchmark(sort_func, arr):
    start = time.time()
    sort_func(arr.copy())
    return time.time() - start

arr = [random.randint(0, 10000) for _ in range(5000)]

# Результаты (примерные):
# bubble_sort:    ~2.5 сек
# selection_sort: ~1.0 сек
# insertion_sort: ~0.8 сек
# merge_sort:     ~0.02 сек
# quick_sort:     ~0.01 сек
# sorted():       ~0.001 сек (Timsort на C)
```

---

## Когда какой алгоритм?

| Ситуация                       | Алгоритм                 |
| ------------------------------ | ------------------------ |
| Общий случай                   | `sorted()` / Quick Sort  |
| Почти отсортировано            | Insertion Sort / Timsort |
| Нужна стабильность             | Merge Sort / Timsort     |
| Ограничена память              | Heap Sort / Quick Sort   |
| Целые числа с малым диапазоном | Counting Sort            |
| Числа с фиксированной длиной   | Radix Sort               |
| Связный список                 | Merge Sort               |
| Параллельная сортировка        | Merge Sort               |
| Учебные цели                   | Bubble Sort              |

---

## Советы

1. **Используй `sorted()` / `.sort()`** — Timsort оптимизирован под Python
2. **O(n²) алгоритмы** — только для маленьких массивов или учёбы
3. **Quick Sort** — лучший выбор для общего случая
4. **Merge Sort** — когда нужна стабильность или работа со списками
5. **Counting/Radix Sort** — когда данные позволяют (целые числа)
6. **Всегда проверяй** — нужна ли сортировка вообще? Может, хватит `heapq.nlargest()`?

---
[prev: 05-recursion](./05-recursion.md) | [next: 03-modules](../03-modules.md)