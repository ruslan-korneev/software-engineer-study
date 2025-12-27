# Пирамидальная сортировка (Heap Sort)

## Определение

**Пирамидальная сортировка** — это алгоритм сортировки, основанный на структуре данных "куча" (heap). Алгоритм строит max-heap из массива, затем многократно извлекает максимальный элемент (корень) и помещает его в конец массива, восстанавливая свойства кучи после каждого извлечения.

## Зачем нужен

### Области применения:
- **Гарантированное O(n log n)** — когда нужна предсказуемость
- **Системы реального времени** — нет деградации до O(n^2)
- **Ограниченная память** — O(1) дополнительной памяти
- **Частичная сортировка** — найти k наибольших/наименьших элементов
- **Очередь с приоритетом** — основа для priority queue

### Преимущества:
- Гарантированное O(n log n) в любом случае
- In-place — O(1) дополнительной памяти
- Нет проблем с рекурсией (итеративная реализация)

### Недостатки:
- Нестабильная сортировка
- Плохая локальность кэша (много "прыжков" по памяти)
- На практике медленнее Quick Sort

## Как работает

### Структура кучи (Heap)

```
Max-Heap: родитель >= дети

Массив: [16, 14, 10, 8, 7, 9, 3, 2, 4, 1]

Дерево:
              16
           /      \
         14        10
        /  \      /  \
       8    7    9    3
      / \  /
     2  4 1

Индексация (0-based):
- Родитель i: (i-1) / 2
- Левый ребёнок i: 2*i + 1
- Правый ребёнок i: 2*i + 2
```

### Визуализация процесса

```
Исходный массив: [4, 10, 3, 5, 1]

ШАГ 1: Построение max-heap (heapify)

    4           10          10
   / \    →    / \    →    / \
  10  3       4   3       5   3
 / \         / \         / \
5   1       5   1       4   1

Массив после построения кучи: [10, 5, 3, 4, 1]

ШАГ 2: Извлечение максимума

[10, 5, 3, 4, 1] → swap(10, 1) → [1, 5, 3, 4 | 10]
                                        ↑
                                   отсортирован

Heapify [1, 5, 3, 4]:
    1           5
   / \    →    / \
  5   3       4   3
 /           /
4           1

[5, 4, 3, 1 | 10]

[5, 4, 3, 1] → swap(5, 1) → [1, 4, 3 | 5, 10]
Heapify → [4, 1, 3 | 5, 10]

[4, 1, 3] → swap(4, 3) → [3, 1 | 4, 5, 10]
Heapify → [3, 1 | 4, 5, 10]

[3, 1] → swap(3, 1) → [1 | 3, 4, 5, 10]

Результат: [1, 3, 4, 5, 10]
```

## Псевдокод

```
function heapSort(array):
    n = length(array)

    # Построение max-heap
    for i from n/2 - 1 down to 0:
        heapify(array, n, i)

    # Извлечение элементов
    for i from n-1 down to 1:
        swap(array[0], array[i])
        heapify(array, i, 0)

function heapify(array, n, i):
    largest = i
    left = 2*i + 1
    right = 2*i + 2

    if left < n and array[left] > array[largest]:
        largest = left

    if right < n and array[right] > array[largest]:
        largest = right

    if largest != i:
        swap(array[i], array[largest])
        heapify(array, n, largest)
```

### Реализация на Python

```python
def heap_sort(arr):
    """Пирамидальная сортировка."""
    n = len(arr)

    # Построение max-heap
    for i in range(n // 2 - 1, -1, -1):
        heapify(arr, n, i)

    # Извлечение элементов один за другим
    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]  # Перемещаем макс в конец
        heapify(arr, i, 0)  # Восстанавливаем кучу

    return arr

def heapify(arr, n, i):
    """Просеивание вниз (sift down)."""
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

### Итеративная версия heapify

```python
def heapify_iterative(arr, n, i):
    """Итеративное просеивание вниз."""
    while True:
        largest = i
        left = 2 * i + 1
        right = 2 * i + 2

        if left < n and arr[left] > arr[largest]:
            largest = left

        if right < n and arr[right] > arr[largest]:
            largest = right

        if largest == i:
            break

        arr[i], arr[largest] = arr[largest], arr[i]
        i = largest
```

### Сортировка по убыванию (min-heap)

```python
def heap_sort_descending(arr):
    """Сортировка по убыванию с min-heap."""
    n = len(arr)

    # Построение min-heap
    for i in range(n // 2 - 1, -1, -1):
        min_heapify(arr, n, i)

    for i in range(n - 1, 0, -1):
        arr[0], arr[i] = arr[i], arr[0]
        min_heapify(arr, i, 0)

    return arr

def min_heapify(arr, n, i):
    smallest = i
    left = 2 * i + 1
    right = 2 * i + 2

    if left < n and arr[left] < arr[smallest]:
        smallest = left

    if right < n and arr[right] < arr[smallest]:
        smallest = right

    if smallest != i:
        arr[i], arr[smallest] = arr[smallest], arr[i]
        min_heapify(arr, n, smallest)
```

### Поиск k наибольших элементов

```python
import heapq

def k_largest(arr, k):
    """Найти k наибольших элементов за O(n log k)."""
    # Используем min-heap размером k
    min_heap = []

    for num in arr:
        if len(min_heap) < k:
            heapq.heappush(min_heap, num)
        elif num > min_heap[0]:
            heapq.heapreplace(min_heap, num)

    return sorted(min_heap, reverse=True)

arr = [3, 2, 1, 5, 6, 4]
print(k_largest(arr, 2))  # [6, 5]
```

## Анализ сложности

| Случай | Временная сложность | Описание |
|--------|---------------------|----------|
| **Лучший** | O(n log n) | Любой массив |
| **Средний** | O(n log n) | Любой массив |
| **Худший** | O(n log n) | Любой массив |

| Ресурс | Сложность |
|--------|-----------|
| **Время** | O(n log n) |
| **Память** | O(1) — in-place |
| **Стабильность** | Нет |

### Почему O(n log n)?
```
Построение кучи: O(n)  (не O(n log n)!)
  - Хотя каждый heapify O(log n)
  - Большинство узлов близко к листьям
  - Математически: sum(h * nodes_at_height_h) = O(n)

Извлечение: n элементов × O(log n) heapify = O(n log n)

Итого: O(n) + O(n log n) = O(n log n)
```

## Пример с пошаговым разбором

### Сортировка массива [12, 11, 13, 5, 6, 7]

```
Исходный: [12, 11, 13, 5, 6, 7]

=== Построение max-heap ===

Начинаем с i = n/2 - 1 = 2 (первый нелистовой узел)

i=2: heapify(arr, 6, 2)
    13
   /  \        Сравниваем 13, 7
  11   13      13 > 7, ничего не меняем
 /  \  /
5   6 7

[12, 11, 13, 5, 6, 7] — без изменений

i=1: heapify(arr, 6, 1)
    12
   /  \        Сравниваем 11, 5, 6
  11   13      11 > 5 и 11 > 6, ничего не меняем
 /  \  /
5   6 7

[12, 11, 13, 5, 6, 7] — без изменений

i=0: heapify(arr, 6, 0)
    12
   /  \        Сравниваем 12, 11, 13
  11   13      13 > 12, меняем 12 и 13
 /  \  /
5   6 7

    13
   /  \        Рекурсивно: heapify(arr, 6, 2)
  11   12      12 > 7, ничего не меняем
 /  \  /
5   6 7

Max-heap: [13, 11, 12, 5, 6, 7]

=== Извлечение ===

i=5: swap(13, 7) → [7, 11, 12, 5, 6 | 13]
     heapify → [12, 11, 7, 5, 6 | 13]

i=4: swap(12, 6) → [6, 11, 7, 5 | 12, 13]
     heapify → [11, 6, 7, 5 | 12, 13]

i=3: swap(11, 5) → [5, 6, 7 | 11, 12, 13]
     heapify → [7, 6, 5 | 11, 12, 13]

i=2: swap(7, 5) → [5, 6 | 7, 11, 12, 13]
     heapify → [6, 5 | 7, 11, 12, 13]

i=1: swap(6, 5) → [5 | 6, 7, 11, 12, 13]

Результат: [5, 6, 7, 11, 12, 13]
```

## Типичные ошибки и Edge Cases

### 1. Неправильная индексация

```python
# ОШИБКА: начинаем heapify с неправильного индекса
def wrong_heap_sort(arr):
    n = len(arr)
    for i in range(n - 1, -1, -1):  # Листья не нужно heapify!
        heapify(arr, n, i)

# ПРАВИЛЬНО: начинаем с первого нелистового узла
def correct_heap_sort(arr):
    n = len(arr)
    for i in range(n // 2 - 1, -1, -1):  # n//2 - 1
        heapify(arr, n, i)
```

### 2. Выход за границы массива

```python
def heapify_safe(arr, n, i):
    largest = i
    left = 2 * i + 1
    right = 2 * i + 2

    # Обязательно проверяем границы!
    if left < n and arr[left] > arr[largest]:
        largest = left

    if right < n and arr[right] > arr[largest]:
        largest = right

    if largest != i:
        arr[i], arr[largest] = arr[largest], arr[i]
        heapify_safe(arr, n, largest)
```

### 3. Пустой массив и один элемент

```python
def heap_sort_safe(arr):
    if len(arr) <= 1:
        return arr

    # ... основной алгоритм
```

### 4. Использование heapq модуля Python

```python
import heapq

def heap_sort_builtin(arr):
    """Heap Sort с использованием heapq."""
    heap = arr.copy()
    heapq.heapify(heap)  # O(n)
    return [heapq.heappop(heap) for _ in range(len(heap))]

# Примечание: heapq создаёт min-heap, не max-heap!
arr = [3, 1, 4, 1, 5, 9, 2, 6]
print(heap_sort_builtin(arr))  # [1, 1, 2, 3, 4, 5, 6, 9]
```

### 5. Нестабильность — проблема с равными элементами

```python
# Heap Sort нестабилен!
arr = [(1, 'a'), (2, 'b'), (1, 'c')]
# После сортировки (1, 'a') и (1, 'c') могут поменяться местами

# Для стабильности можно добавить индекс:
def stable_heap_sort(arr, key=lambda x: x):
    indexed = [(key(item), i, item) for i, item in enumerate(arr)]
    heap_sort(indexed)  # Сортируем по (key, index)
    return [item for _, _, item in indexed]
```

### 6. Оптимизация: Floyd's heap construction

```python
def build_heap_floyd(arr):
    """
    Построение кучи методом Флойда (снизу вверх).
    Это стандартный подход, O(n).
    """
    n = len(arr)
    # Начинаем с последнего нелистового узла
    for i in range(n // 2 - 1, -1, -1):
        heapify(arr, n, i)
```

### Рекомендации:
1. Heap Sort хорош, когда нужна гарантия O(n log n)
2. Для частичной сортировки (k элементов) используйте heapq
3. Heap Sort не cache-friendly — на практике медленнее Quick Sort
4. Используйте heapq для priority queue операций
5. Помните о нестабильности при сортировке объектов
