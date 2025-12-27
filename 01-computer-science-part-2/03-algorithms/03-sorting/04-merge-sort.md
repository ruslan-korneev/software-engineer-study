# Сортировка слиянием (Merge Sort)

## Определение

**Сортировка слиянием** — это эффективный алгоритм сортировки, основанный на принципе "разделяй и властвуй" (divide and conquer). Алгоритм рекурсивно делит массив пополам, сортирует каждую половину, а затем сливает отсортированные половины в один отсортированный массив.

## Зачем нужен

### Области применения:
- **Сортировка связанных списков** — эффективен без произвольного доступа
- **Внешняя сортировка** — файлы, не помещающиеся в память
- **Параллельные вычисления** — легко распараллелить
- **Требуется стабильность** — гарантированно стабильная сортировка
- **Требуется гарантия O(n log n)** — в любом случае

### Преимущества:
- Гарантированное O(n log n) в любом случае
- Стабильная сортировка
- Легко распараллеливается
- Хорошо работает со связанными списками
- Хорош для внешней сортировки

### Недостатки:
- Требует O(n) дополнительной памяти
- Не адаптивный — не ускоряется на частично отсортированных данных
- Медленнее Quick Sort на практике для массивов в памяти

## Как работает

### Визуализация процесса

```
Исходный массив: [38, 27, 43, 3, 9, 82, 10]

ФАЗА РАЗДЕЛЕНИЯ (Divide):

                [38, 27, 43, 3, 9, 82, 10]
                          |
            ┌─────────────┴─────────────┐
      [38, 27, 43, 3]             [9, 82, 10]
            |                          |
      ┌─────┴─────┐              ┌─────┴─────┐
   [38, 27]   [43, 3]         [9, 82]    [10]
      |          |               |
   ┌──┴──┐    ┌──┴──┐        ┌──┴──┐
  [38]  [27] [43]  [3]      [9]  [82]

ФАЗА СЛИЯНИЯ (Conquer):

  [38]  [27] [43]  [3]      [9]  [82]    [10]
      |          |               |          |
   └──┬──┘    └──┬──┘        └──┬──┘        |
   [27, 38]  [3, 43]         [9, 82]      [10]
        |          |               |         |
        └────┬─────┘               └────┬────┘
      [3, 27, 38, 43]             [9, 10, 82]
              |                          |
              └────────────┬─────────────┘
               [3, 9, 10, 27, 38, 43, 82]
```

### Детальная визуализация слияния

```
Слияние [27, 38] и [3, 43]:

Левый:   [27, 38]    Правый: [3, 43]    Результат: []
          ↑                   ↑
          i                   j

Шаг 1: 27 > 3, берём 3
Левый:   [27, 38]    Правый: [3, 43]    Результат: [3]
          ↑                      ↑
          i                      j

Шаг 2: 27 < 43, берём 27
Левый:   [27, 38]    Правый: [3, 43]    Результат: [3, 27]
              ↑                  ↑
              i                  j

Шаг 3: 38 < 43, берём 38
Левый:   [27, 38]    Правый: [3, 43]    Результат: [3, 27, 38]
                 ↑               ↑
                 i               j

Шаг 4: левый пуст, берём оставшееся из правого
Результат: [3, 27, 38, 43]
```

## Псевдокод

```
function mergeSort(array):
    if length(array) <= 1:
        return array

    # Разделяем
    mid = length(array) / 2
    left = mergeSort(array[0..mid])
    right = mergeSort(array[mid..end])

    # Сливаем
    return merge(left, right)

function merge(left, right):
    result = []
    i, j = 0, 0

    while i < length(left) and j < length(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i++
        else:
            result.append(right[j])
            j++

    # Добавляем оставшиеся элементы
    result.extend(left[i..end])
    result.extend(right[j..end])

    return result
```

### Реализация на Python (Top-Down)

```python
def merge_sort(arr):
    """Рекурсивная сортировка слиянием (top-down)."""
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

    return merge(left, right)

def merge(left, right):
    """Слияние двух отсортированных массивов."""
    result = []
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    # Добавляем оставшиеся элементы
    result.extend(left[i:])
    result.extend(right[j:])

    return result
```

### In-place реализация (меньше памяти)

```python
def merge_sort_inplace(arr, left=0, right=None):
    """Сортировка слиянием на месте."""
    if right is None:
        right = len(arr) - 1

    if left < right:
        mid = (left + right) // 2

        merge_sort_inplace(arr, left, mid)
        merge_sort_inplace(arr, mid + 1, right)

        merge_inplace(arr, left, mid, right)

def merge_inplace(arr, left, mid, right):
    """Слияние на месте."""
    # Создаём временные массивы
    left_arr = arr[left:mid + 1]
    right_arr = arr[mid + 1:right + 1]

    i = j = 0
    k = left

    while i < len(left_arr) and j < len(right_arr):
        if left_arr[i] <= right_arr[j]:
            arr[k] = left_arr[i]
            i += 1
        else:
            arr[k] = right_arr[j]
            j += 1
        k += 1

    while i < len(left_arr):
        arr[k] = left_arr[i]
        i += 1
        k += 1

    while j < len(right_arr):
        arr[k] = right_arr[j]
        j += 1
        k += 1
```

### Итеративная реализация (Bottom-Up)

```python
def merge_sort_iterative(arr):
    """Итеративная сортировка слиянием (bottom-up)."""
    n = len(arr)
    size = 1

    while size < n:
        for left in range(0, n, 2 * size):
            mid = min(left + size - 1, n - 1)
            right = min(left + 2 * size - 1, n - 1)

            if mid < right:
                merge_inplace(arr, left, mid, right)

        size *= 2

    return arr
```

### Сортировка связанного списка

```python
class ListNode:
    def __init__(self, val=0, next=None):
        self.val = val
        self.next = next

def merge_sort_list(head):
    """Merge Sort для связанного списка."""
    if not head or not head.next:
        return head

    # Находим середину
    slow = fast = head
    prev = None
    while fast and fast.next:
        prev = slow
        slow = slow.next
        fast = fast.next.next

    prev.next = None  # Разделяем список

    left = merge_sort_list(head)
    right = merge_sort_list(slow)

    return merge_lists(left, right)

def merge_lists(l1, l2):
    """Слияние двух отсортированных списков."""
    dummy = ListNode()
    current = dummy

    while l1 and l2:
        if l1.val <= l2.val:
            current.next = l1
            l1 = l1.next
        else:
            current.next = l2
            l2 = l2.next
        current = current.next

    current.next = l1 or l2

    return dummy.next
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
| **Память** | O(n) — для массивов |
| **Память** | O(log n) — для связанных списков |
| **Стабильность** | Да |

### Почему O(n log n)?
```
Глубина рекурсии: log₂(n)
Работа на каждом уровне: O(n)

Уровень 0:  n элементов в 1 слиянии
Уровень 1:  n элементов в 2 слияниях
Уровень 2:  n элементов в 4 слияниях
...
Уровень k:  n элементов в 2^k слияниях

Всего уровней: log₂(n)
Всего работы: n × log₂(n) = O(n log n)
```

## Пример с пошаговым разбором

### Сортировка массива [12, 11, 13, 5, 6, 7]

```
mergeSort([12, 11, 13, 5, 6, 7])
│
├── left = mergeSort([12, 11, 13])
│   │
│   ├── left = mergeSort([12])
│   │   └── return [12]
│   │
│   ├── right = mergeSort([11, 13])
│   │   │
│   │   ├── left = mergeSort([11]) → [11]
│   │   ├── right = mergeSort([13]) → [13]
│   │   └── merge([11], [13]) → [11, 13]
│   │
│   └── merge([12], [11, 13]) → [11, 12, 13]
│
├── right = mergeSort([5, 6, 7])
│   │
│   ├── left = mergeSort([5])
│   │   └── return [5]
│   │
│   ├── right = mergeSort([6, 7])
│   │   │
│   │   ├── left = mergeSort([6]) → [6]
│   │   ├── right = mergeSort([7]) → [7]
│   │   └── merge([6], [7]) → [6, 7]
│   │
│   └── merge([5], [6, 7]) → [5, 6, 7]
│
└── merge([11, 12, 13], [5, 6, 7])

Финальное слияние [11, 12, 13] и [5, 6, 7]:
5 < 11 → [5]
6 < 11 → [5, 6]
7 < 11 → [5, 6, 7]
11 → [5, 6, 7, 11]
12 → [5, 6, 7, 11, 12]
13 → [5, 6, 7, 11, 12, 13]

Результат: [5, 6, 7, 11, 12, 13]
```

## Типичные ошибки и Edge Cases

### 1. Неправильное разделение массива

```python
# ОШИБКА: бесконечная рекурсия при len=2
def wrong_merge_sort(arr):
    mid = len(arr) // 2
    if mid == 0:
        return arr
    # При [1, 2]: mid=1, left=[1], right=[2] — OK
    # Но если проверка неправильная...

# ПРАВИЛЬНО
def correct_merge_sort(arr):
    if len(arr) <= 1:  # Базовый случай
        return arr
    mid = len(arr) // 2
    # ...
```

### 2. Нестабильное слияние

```python
# ОШИБКА: потеря стабильности
def unstable_merge(left, right):
    # ...
    if left[i] < right[j]:  # Должно быть <=
        result.append(left[i])
    # ...

# ПРАВИЛЬНО (стабильное слияние)
def stable_merge(left, right):
    # ...
    if left[i] <= right[j]:  # <= для стабильности!
        result.append(left[i])
    # ...
```

### 3. Пустой массив

```python
def merge_sort_safe(arr):
    if len(arr) <= 1:
        return arr.copy()  # Возвращаем копию!
    # ...

print(merge_sort_safe([]))   # []
print(merge_sort_safe([1]))  # [1]
```

### 4. Не создавать новый массив при необходимости

```python
# ОШИБКА: изменяем исходный массив
def merge_sort_inplace_wrong(arr):
    if len(arr) <= 1:
        return arr  # Возвращаем тот же объект
    # ...

# ПРАВИЛЬНО: создаём копии при разделении
def merge_sort(arr):
    if len(arr) <= 1:
        return arr[:]  # Копия!

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])   # Срез создаёт копию
    right = merge_sort(arr[mid:])

    return merge(left, right)
```

### 5. Подсчёт инверсий с Merge Sort

```python
def count_inversions(arr):
    """Считает инверсии, используя Merge Sort."""
    if len(arr) <= 1:
        return arr, 0

    mid = len(arr) // 2
    left, left_inv = count_inversions(arr[:mid])
    right, right_inv = count_inversions(arr[mid:])

    merged, split_inv = merge_and_count(left, right)

    return merged, left_inv + right_inv + split_inv

def merge_and_count(left, right):
    result = []
    inversions = 0
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] <= right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1
            # Все оставшиеся элементы left > right[j]
            inversions += len(left) - i

    result.extend(left[i:])
    result.extend(right[j:])

    return result, inversions

arr = [2, 4, 1, 3, 5]
sorted_arr, inv = count_inversions(arr)
print(f"Инверсий: {inv}")  # 3: (2,1), (4,1), (4,3)
```

### Рекомендации:
1. Merge Sort гарантирует O(n log n) в худшем случае
2. Используйте для связанных списков — не требует O(n) памяти
3. Хорош для внешней сортировки больших файлов
4. Легко распараллеливается — каждая половина независима
5. Предпочтите Python's `sorted()` — использует Timsort, который включает идеи Merge Sort
