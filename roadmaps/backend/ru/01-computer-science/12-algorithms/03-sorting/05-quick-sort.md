# Быстрая сортировка (Quick Sort)

[prev: 04-merge-sort](./04-merge-sort.md) | [next: 06-heap-sort](./06-heap-sort.md)
---

## Определение

**Быстрая сортировка** — это эффективный алгоритм сортировки, основанный на принципе "разделяй и властвуй". Алгоритм выбирает опорный элемент (pivot), разделяет массив на две части — элементы меньше и больше опорного — и рекурсивно сортирует каждую часть.

Quick Sort — один из самых быстрых алгоритмов сортировки на практике, несмотря на O(n^2) в худшем случае.

## Зачем нужен

### Области применения:
- **Универсальная сортировка** — эффективен для большинства данных
- **Системные библиотеки** — часть `qsort()` в C
- **In-place сортировка** — когда память ограничена
- **Базы данных** — сортировка записей

### Преимущества:
- Очень быстрый на практике — O(n log n) в среднем
- In-place — O(log n) дополнительной памяти (стек рекурсии)
- Cache-friendly — хорошо работает с кэшем процессора
- Легко оптимизировать

### Недостатки:
- O(n^2) в худшем случае (уже отсортированный массив)
- Нестабильная сортировка
- Рекурсия может привести к Stack Overflow

## Как работает

### Визуализация процесса

```
Исходный массив: [10, 80, 30, 90, 40, 50, 70]
Pivot = 70 (последний элемент)

PARTITION:
[10, 80, 30, 90, 40, 50, 70]
  i                      pivot

Шаг 1: 10 < 70, меняем arr[0] с arr[0] (ничего)
[10, 80, 30, 90, 40, 50, 70]
      i

Шаг 2: 80 > 70, пропускаем

Шаг 3: 30 < 70, меняем arr[1] с arr[2]
[10, 30, 80, 90, 40, 50, 70]
          i

Шаг 4: 90 > 70, пропускаем

Шаг 5: 40 < 70, меняем arr[2] с arr[4]
[10, 30, 40, 90, 80, 50, 70]
              i

Шаг 6: 50 < 70, меняем arr[3] с arr[5]
[10, 30, 40, 50, 80, 90, 70]
                  i

Финал: меняем pivot с arr[4]
[10, 30, 40, 50, 70, 90, 80]
                  ↑
                pivot на своём месте!

Теперь рекурсивно:
[10, 30, 40, 50] | 70 | [90, 80]
```

### Схема разделения (Lomuto partition)

```
                     Pivot
                       ↓
[  < pivot  |  > pivot  | ? ... | P ]
             ↑          ↑
             i          j

i — граница между меньшими и большими
j — текущий проверяемый элемент
```

## Псевдокод

```
function quickSort(array, low, high):
    if low < high:
        pivotIndex = partition(array, low, high)
        quickSort(array, low, pivotIndex - 1)
        quickSort(array, pivotIndex + 1, high)

function partition(array, low, high):
    pivot = array[high]
    i = low - 1

    for j from low to high - 1:
        if array[j] < pivot:
            i++
            swap(array[i], array[j])

    swap(array[i + 1], array[high])
    return i + 1
```

### Базовая реализация на Python

```python
def quick_sort(arr, low=0, high=None):
    """Quick Sort с Lomuto partition."""
    if high is None:
        high = len(arr) - 1

    if low < high:
        pivot_idx = partition(arr, low, high)
        quick_sort(arr, low, pivot_idx - 1)
        quick_sort(arr, pivot_idx + 1, high)

    return arr

def partition(arr, low, high):
    """Lomuto partition scheme."""
    pivot = arr[high]
    i = low - 1

    for j in range(low, high):
        if arr[j] < pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1
```

### Hoare partition (оригинальная, более эффективная)

```python
def quick_sort_hoare(arr, low=0, high=None):
    """Quick Sort с Hoare partition."""
    if high is None:
        high = len(arr) - 1

    if low < high:
        pivot_idx = partition_hoare(arr, low, high)
        quick_sort_hoare(arr, low, pivot_idx)
        quick_sort_hoare(arr, pivot_idx + 1, high)

    return arr

def partition_hoare(arr, low, high):
    """Hoare partition scheme."""
    pivot = arr[(low + high) // 2]
    i = low - 1
    j = high + 1

    while True:
        i += 1
        while arr[i] < pivot:
            i += 1

        j -= 1
        while arr[j] > pivot:
            j -= 1

        if i >= j:
            return j

        arr[i], arr[j] = arr[j], arr[i]
```

### Медиана трёх (оптимизация выбора pivot)

```python
def median_of_three(arr, low, high):
    """Выбирает медиану из первого, среднего и последнего элементов."""
    mid = (low + high) // 2

    if arr[low] > arr[mid]:
        arr[low], arr[mid] = arr[mid], arr[low]
    if arr[low] > arr[high]:
        arr[low], arr[high] = arr[high], arr[low]
    if arr[mid] > arr[high]:
        arr[mid], arr[high] = arr[high], arr[mid]

    # Помещаем медиану в конец
    arr[mid], arr[high - 1] = arr[high - 1], arr[mid]
    return arr[high - 1]
```

### Итеративная версия (без рекурсии)

```python
def quick_sort_iterative(arr):
    """Итеративный Quick Sort со стеком."""
    if len(arr) <= 1:
        return arr

    stack = [(0, len(arr) - 1)]

    while stack:
        low, high = stack.pop()

        if low < high:
            pivot_idx = partition(arr, low, high)

            # Добавляем подмассивы в стек
            # Меньший подмассив первым (оптимизация памяти)
            if pivot_idx - low < high - pivot_idx:
                stack.append((pivot_idx + 1, high))
                stack.append((low, pivot_idx - 1))
            else:
                stack.append((low, pivot_idx - 1))
                stack.append((pivot_idx + 1, high))

    return arr
```

### Трёхстороннее разбиение (для дубликатов)

```python
def quick_sort_3way(arr, low=0, high=None):
    """
    Quick Sort с трёхсторонним разбиением.
    Эффективен для массивов с множеством дубликатов.
    """
    if high is None:
        high = len(arr) - 1

    if low >= high:
        return

    lt, gt = partition_3way(arr, low, high)

    quick_sort_3way(arr, low, lt - 1)
    quick_sort_3way(arr, gt + 1, high)

def partition_3way(arr, low, high):
    """Dutch National Flag partition."""
    pivot = arr[low]
    lt = low      # arr[low..lt-1] < pivot
    gt = high     # arr[gt+1..high] > pivot
    i = low + 1   # arr[lt..i-1] == pivot

    while i <= gt:
        if arr[i] < pivot:
            arr[lt], arr[i] = arr[i], arr[lt]
            lt += 1
            i += 1
        elif arr[i] > pivot:
            arr[gt], arr[i] = arr[i], arr[gt]
            gt -= 1
        else:
            i += 1

    return lt, gt
```

## Анализ сложности

| Случай | Временная сложность | Описание |
|--------|---------------------|----------|
| **Лучший** | O(n log n) | Сбалансированное разбиение |
| **Средний** | O(n log n) | Случайные данные |
| **Худший** | O(n^2) | Уже отсортированный массив (при плохом pivot) |

| Ресурс | Сложность |
|--------|-----------|
| **Время** | O(n log n) средний, O(n^2) худший |
| **Память** | O(log n) — стек рекурсии |
| **Стабильность** | Нет |

### Сравнение partition schemes:

| Схема | Обменов | Особенности |
|-------|---------|-------------|
| Lomuto | Больше | Проще, понятнее |
| Hoare | Меньше | Эффективнее на практике |

## Пример с пошаговым разбором

### Сортировка массива [3, 6, 8, 10, 1, 2, 1]

```
quickSort([3, 6, 8, 10, 1, 2, 1])

pivot = 1 (последний элемент)
partition:
  [3, 6, 8, 10, 1, 2, 1]
   ↑                  pivot
   j

  3 > 1, пропуск
  6 > 1, пропуск
  8 > 1, пропуск
  10 > 1, пропуск
  1 <= 1, swap(arr[0], arr[4]): [1, 6, 8, 10, 3, 2, 1]
  2 > 1, пропуск

  swap pivot: [1, 1, 8, 10, 3, 2, 6]
              ← pivot (idx=1)

Рекурсия:
  left: [1] — уже отсортирован
  right: [8, 10, 3, 2, 6]

  pivot = 6
  partition [8, 10, 3, 2, 6]:
    8 > 6, пропуск
    10 > 6, пропуск
    3 < 6, swap → [3, 10, 8, 2, 6]
    2 < 6, swap → [3, 2, 8, 10, 6]
    swap pivot → [3, 2, 6, 10, 8]

  Рекурсия:
    left: [3, 2] → [2, 3]
    right: [10, 8] → [8, 10]

Результат: [1, 1, 2, 3, 6, 8, 10]
```

## Типичные ошибки и Edge Cases

### 1. Худший случай — уже отсортированный массив

```python
# ПРОБЛЕМА: O(n^2) на отсортированном массиве
arr = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
quick_sort(arr)  # Медленно!

# РЕШЕНИЕ: случайный pivot
import random

def partition_random(arr, low, high):
    rand_idx = random.randint(low, high)
    arr[rand_idx], arr[high] = arr[high], arr[rand_idx]
    return partition(arr, low, high)
```

### 2. Много дубликатов

```python
# ПРОБЛЕМА: стандартный partition плохо работает с дубликатами
arr = [2, 2, 2, 2, 2, 2, 2]  # O(n^2)!

# РЕШЕНИЕ: трёхстороннее разбиение
quick_sort_3way(arr)  # O(n) для массива одинаковых элементов
```

### 3. Stack Overflow на больших массивах

```python
import sys
sys.setrecursionlimit(10000)  # Не рекомендуется!

# РЕШЕНИЕ 1: итеративная версия
quick_sort_iterative(large_arr)

# РЕШЕНИЕ 2: оптимизация хвостовой рекурсии
def quick_sort_tail_optimized(arr, low, high):
    while low < high:
        pivot_idx = partition(arr, low, high)

        # Рекурсия для меньшей части, итерация для большей
        if pivot_idx - low < high - pivot_idx:
            quick_sort_tail_optimized(arr, low, pivot_idx - 1)
            low = pivot_idx + 1
        else:
            quick_sort_tail_optimized(arr, pivot_idx + 1, high)
            high = pivot_idx - 1
```

### 4. Пустой массив и один элемент

```python
def quick_sort_safe(arr, low=0, high=None):
    if not arr:
        return arr

    if high is None:
        high = len(arr) - 1

    if low < high:  # Обрабатывает len=1
        # ...
```

### 5. Интроспективная сортировка (Introsort)

```python
import math

def introsort(arr):
    """
    Гибрид Quick Sort, Heap Sort и Insertion Sort.
    Используется в std::sort в C++.
    """
    max_depth = 2 * int(math.log2(len(arr) + 1))
    _introsort(arr, 0, len(arr) - 1, max_depth)
    return arr

def _introsort(arr, low, high, max_depth):
    size = high - low + 1

    if size <= 16:
        # Insertion sort для маленьких массивов
        insertion_sort_range(arr, low, high)
        return

    if max_depth == 0:
        # Переключаемся на Heap Sort
        heapsort_range(arr, low, high)
        return

    pivot_idx = partition(arr, low, high)
    _introsort(arr, low, pivot_idx - 1, max_depth - 1)
    _introsort(arr, pivot_idx + 1, high, max_depth - 1)
```

### 6. Сортировка с ключом

```python
def quick_sort_key(arr, key=lambda x: x, low=0, high=None):
    if high is None:
        high = len(arr) - 1

    if low < high:
        pivot_idx = partition_key(arr, key, low, high)
        quick_sort_key(arr, key, low, pivot_idx - 1)
        quick_sort_key(arr, key, pivot_idx + 1, high)

    return arr

def partition_key(arr, key, low, high):
    pivot_key = key(arr[high])
    i = low - 1

    for j in range(low, high):
        if key(arr[j]) < pivot_key:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1
```

### Рекомендации:
1. Используйте случайный pivot или медиану трёх
2. Для массивов с дубликатами — трёхстороннее разбиение
3. Переключайтесь на Insertion Sort для маленьких подмассивов
4. Для гарантии O(n log n) — используйте Introsort
5. В production используйте встроенный `sort()` — он уже оптимизирован

---
[prev: 04-merge-sort](./04-merge-sort.md) | [next: 06-heap-sort](./06-heap-sort.md)