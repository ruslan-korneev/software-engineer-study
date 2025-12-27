# Операции над массивами (Array Operations)

## Определение

**Операции над массивами** — это базовые алгоритмы и действия, которые можно выполнять с массивами: доступ, поиск, вставка, удаление, сортировка, слияние и другие манипуляции с элементами.

## Зачем нужно

Понимание операций над массивами необходимо для:
- **Эффективного выбора структуры данных** — знание сложности операций
- **Оптимизации алгоритмов** — выбор правильного подхода
- **Решения типовых задач** — поиск, фильтрация, агрегация
- **Подготовки к собеседованиям** — частые вопросы на интервью

---

## 1. Базовые операции

### Доступ по индексу (Access)

```
Массив: [10, 20, 30, 40, 50]
Индекс:   0   1   2   3   4

arr[2] = 30  ← прямой доступ за O(1)
```

```python
def get(arr, index):
    if 0 <= index < len(arr):
        return arr[index]
    raise IndexError("Index out of bounds")

def set(arr, index, value):
    if 0 <= index < len(arr):
        arr[index] = value
    else:
        raise IndexError("Index out of bounds")
```

**Сложность**: O(1)

---

### Обход массива (Traversal)

```python
# Прямой обход
def traverse(arr):
    for i in range(len(arr)):
        process(arr[i])

# Обратный обход
def reverse_traverse(arr):
    for i in range(len(arr) - 1, -1, -1):
        process(arr[i])

# С индексом
for index, value in enumerate(arr):
    print(f"arr[{index}] = {value}")
```

**Сложность**: O(n)

---

## 2. Поиск (Search)

### Линейный поиск (Linear Search)

```
Поиск значения 30:

[10, 20, 30, 40, 50]
 ↑   ↑   ↑
 1   2   3 ← найдено на 3-й проверке
```

```python
def linear_search(arr, target):
    for i in range(len(arr)):
        if arr[i] == target:
            return i
    return -1  # не найдено
```

**Сложность**: O(n)

---

### Бинарный поиск (Binary Search)

**Требование**: массив должен быть отсортирован!

```
Поиск значения 30 в [10, 20, 30, 40, 50]:

Шаг 1: left=0, right=4, mid=2
       [10, 20, 30, 40, 50]
               ↑
        arr[2]=30 == 30 ← найдено!
```

```python
def binary_search(arr, target):
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = (left + right) // 2

        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1  # не найдено
```

**Сложность**: O(log n)

---

### Поиск минимума/максимума

```python
def find_min(arr):
    if not arr:
        raise ValueError("Empty array")

    min_val = arr[0]
    for i in range(1, len(arr)):
        if arr[i] < min_val:
            min_val = arr[i]
    return min_val

def find_max(arr):
    if not arr:
        raise ValueError("Empty array")

    max_val = arr[0]
    for i in range(1, len(arr)):
        if arr[i] > max_val:
            max_val = arr[i]
    return max_val

# Одновременный поиск (оптимизация: 1.5n сравнений вместо 2n)
def find_min_max(arr):
    if not arr:
        raise ValueError("Empty array")

    if len(arr) == 1:
        return arr[0], arr[0]

    if arr[0] < arr[1]:
        min_val, max_val = arr[0], arr[1]
    else:
        min_val, max_val = arr[1], arr[0]

    for i in range(2, len(arr)):
        if arr[i] < min_val:
            min_val = arr[i]
        elif arr[i] > max_val:
            max_val = arr[i]

    return min_val, max_val
```

**Сложность**: O(n)

---

## 3. Вставка (Insertion)

### Вставка в конец

```
До:  [10, 20, 30, _, _]  size=3, capacity=5
               ↓
После: [10, 20, 30, 40, _]  size=4
```

```python
def append(arr, value):
    arr.append(value)  # O(1) амортизированное
```

**Сложность**: O(1) амортизированное

---

### Вставка в начало

```
До:    [10, 20, 30]
        ↓   ↓   ↓   (сдвиг вправо)
После: [5, 10, 20, 30]
```

```python
def insert_at_beginning(arr, value):
    # Сдвигаем все элементы вправо
    arr.append(None)  # расширяем
    for i in range(len(arr) - 1, 0, -1):
        arr[i] = arr[i - 1]
    arr[0] = value

# Или встроенный метод:
arr.insert(0, value)
```

**Сложность**: O(n)

---

### Вставка по индексу

```
Вставка 25 на позицию 2:

До:    [10, 20, 30, 40]
            ↓   ↓   (сдвиг элементов с индекса 2)
После: [10, 20, 25, 30, 40]
```

```python
def insert_at(arr, index, value):
    if index < 0 or index > len(arr):
        raise IndexError("Invalid index")

    arr.append(None)  # расширяем

    # Сдвигаем элементы вправо
    for i in range(len(arr) - 1, index, -1):
        arr[i] = arr[i - 1]

    arr[index] = value

# Встроенный метод:
arr.insert(index, value)
```

**Сложность**: O(n)

---

## 4. Удаление (Deletion)

### Удаление из конца

```python
def pop(arr):
    if not arr:
        raise IndexError("Empty array")
    return arr.pop()  # O(1)
```

**Сложность**: O(1)

---

### Удаление из начала

```
До:    [10, 20, 30, 40]
        ↑   ←  ←  ←   (сдвиг влево)
После: [20, 30, 40]
```

```python
def remove_first(arr):
    if not arr:
        raise IndexError("Empty array")

    removed = arr[0]
    for i in range(len(arr) - 1):
        arr[i] = arr[i + 1]
    arr.pop()  # удаляем последний элемент

    return removed

# Встроенный метод:
arr.pop(0)
```

**Сложность**: O(n)

---

### Удаление по индексу

```python
def remove_at(arr, index):
    if index < 0 or index >= len(arr):
        raise IndexError("Invalid index")

    removed = arr[index]

    # Сдвигаем элементы влево
    for i in range(index, len(arr) - 1):
        arr[i] = arr[i + 1]

    arr.pop()
    return removed

# Встроенный метод:
arr.pop(index)
```

**Сложность**: O(n)

---

### Удаление по значению

```python
def remove_value(arr, value):
    index = linear_search(arr, value)
    if index == -1:
        raise ValueError("Value not found")
    return remove_at(arr, index)

# Встроенный метод:
arr.remove(value)  # удаляет первое вхождение
```

**Сложность**: O(n)

---

## 5. Реверс массива (Reverse)

### In-place реверс (без дополнительной памяти)

```
До:    [1, 2, 3, 4, 5]
        ↕        ↕
       [5, 2, 3, 4, 1]
           ↕  ↕
       [5, 4, 3, 2, 1]
После: [5, 4, 3, 2, 1]
```

```python
def reverse_in_place(arr):
    left, right = 0, len(arr) - 1

    while left < right:
        # Обмен элементов
        arr[left], arr[right] = arr[right], arr[left]
        left += 1
        right -= 1

    return arr

# Встроенные методы:
arr.reverse()          # in-place
reversed_arr = arr[::-1]  # создаёт новый массив
```

**Сложность**: O(n) времени, O(1) памяти

---

## 6. Ротация массива (Rotation)

### Левая ротация на k позиций

```
Ротация [1, 2, 3, 4, 5] на 2 позиции влево:

Шаг 1: [2, 3, 4, 5, 1]
Шаг 2: [3, 4, 5, 1, 2]
```

```python
# Метод 1: Наивный — O(n*k) времени
def rotate_left_naive(arr, k):
    for _ in range(k):
        first = arr[0]
        for i in range(len(arr) - 1):
            arr[i] = arr[i + 1]
        arr[-1] = first

# Метод 2: Slice — O(n) времени, O(n) памяти
def rotate_left_slice(arr, k):
    k = k % len(arr)
    return arr[k:] + arr[:k]

# Метод 3: Reversal algorithm — O(n) времени, O(1) памяти
def rotate_left_optimal(arr, k):
    n = len(arr)
    k = k % n

    def reverse(start, end):
        while start < end:
            arr[start], arr[end] = arr[end], arr[start]
            start += 1
            end -= 1

    reverse(0, k - 1)      # [2, 1, 3, 4, 5]
    reverse(k, n - 1)      # [2, 1, 5, 4, 3]
    reverse(0, n - 1)      # [3, 4, 5, 1, 2]

    return arr
```

---

## 7. Слияние массивов (Merge)

### Слияние двух отсортированных массивов

```
arr1 = [1, 3, 5]
arr2 = [2, 4, 6]

      ↓           ↓
      1 < 2 → добавить 1
         ↓        ↓
         3 > 2 → добавить 2
         ↓           ↓
         3 < 4 → добавить 3
         ...

Результат: [1, 2, 3, 4, 5, 6]
```

```python
def merge_sorted(arr1, arr2):
    result = []
    i, j = 0, 0

    while i < len(arr1) and j < len(arr2):
        if arr1[i] <= arr2[j]:
            result.append(arr1[i])
            i += 1
        else:
            result.append(arr2[j])
            j += 1

    # Добавляем оставшиеся элементы
    result.extend(arr1[i:])
    result.extend(arr2[j:])

    return result
```

**Сложность**: O(n + m) времени, O(n + m) памяти

---

## 8. Агрегация (Aggregation)

```python
# Сумма элементов
def sum_array(arr):
    total = 0
    for x in arr:
        total += x
    return total
# Или: sum(arr)

# Среднее значение
def average(arr):
    if not arr:
        raise ValueError("Empty array")
    return sum(arr) / len(arr)

# Количество по условию
def count_if(arr, predicate):
    count = 0
    for x in arr:
        if predicate(x):
            count += 1
    return count

# Пример: количество чётных
evens = count_if([1, 2, 3, 4, 5], lambda x: x % 2 == 0)  # 2
```

---

## 9. Фильтрация и преобразование

### Filter
```python
def filter_array(arr, predicate):
    result = []
    for x in arr:
        if predicate(x):
            result.append(x)
    return result

# Пример: только положительные
positives = filter_array([-1, 2, -3, 4], lambda x: x > 0)  # [2, 4]

# Pythonic: list comprehension
positives = [x for x in arr if x > 0]
```

### Map (преобразование)
```python
def map_array(arr, func):
    result = []
    for x in arr:
        result.append(func(x))
    return result

# Пример: возвести в квадрат
squares = map_array([1, 2, 3], lambda x: x ** 2)  # [1, 4, 9]

# Pythonic:
squares = [x ** 2 for x in arr]
```

### Reduce (свёртка)
```python
def reduce_array(arr, func, initial):
    accumulator = initial
    for x in arr:
        accumulator = func(accumulator, x)
    return accumulator

# Пример: произведение всех элементов
product = reduce_array([1, 2, 3, 4], lambda acc, x: acc * x, 1)  # 24

# Стандартная библиотека:
from functools import reduce
product = reduce(lambda acc, x: acc * x, arr, 1)
```

---

## Сводная таблица сложности

| Операция | Время | Память |
|----------|-------|--------|
| Access (get/set) | O(1) | O(1) |
| Linear Search | O(n) | O(1) |
| Binary Search | O(log n) | O(1) |
| Insert at end | O(1)* | O(1) |
| Insert at beginning | O(n) | O(1) |
| Insert at index | O(n) | O(1) |
| Remove from end | O(1) | O(1) |
| Remove from beginning | O(n) | O(1) |
| Remove by index | O(n) | O(1) |
| Reverse | O(n) | O(1) |
| Rotate | O(n) | O(1) |
| Merge sorted | O(n+m) | O(n+m) |
| Filter/Map | O(n) | O(n) |

*Амортизированное время для динамического массива

---

## Типичные ошибки

### 1. Изменение массива во время итерации
```python
# НЕПРАВИЛЬНО
for i in range(len(arr)):
    if arr[i] < 0:
        arr.pop(i)  # IndexError!

# ПРАВИЛЬНО
arr = [x for x in arr if x >= 0]
```

### 2. Off-by-one в бинарном поиске
```python
# НЕПРАВИЛЬНО
mid = (left + right) / 2   # деление с остатком!
while left < right:        # пропустит случай left == right

# ПРАВИЛЬНО
mid = (left + right) // 2  # целочисленное деление
while left <= right:       # включая равенство
```

### 3. Бинарный поиск по неотсортированному массиву
```python
arr = [3, 1, 4, 1, 5]
binary_search(arr, 4)  # неопределённый результат!

# Сначала сортировка:
arr.sort()
binary_search(arr, 4)
```

### 4. Переполнение при вычислении mid
```python
# Проблема в языках с фиксированными целыми:
mid = (left + right) / 2  # может переполниться!

# Безопасная формула:
mid = left + (right - left) // 2
```

### 5. Забытый return в рекурсивном поиске
```python
# НЕПРАВИЛЬНО
def binary_search_recursive(arr, target, left, right):
    if left > right:
        return -1
    mid = (left + right) // 2
    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        binary_search_recursive(arr, target, mid + 1, right)  # забыли return!
    else:
        binary_search_recursive(arr, target, left, mid - 1)

# ПРАВИЛЬНО — добавить return
```
