# Распространённые классы сложности (Common Complexities)

[prev: 04-big-theta](./04-big-theta.md) | [next: 06-space-complexity](./06-space-complexity.md)

---
## Определение

**Классы сложности** — это категории алгоритмов, сгруппированные по скорости роста времени выполнения или потребления памяти. Понимание этих классов позволяет быстро оценить эффективность алгоритма.

## Иерархия классов сложности

От самых быстрых к самым медленным:

```
O(1) < O(log n) < O(n) < O(n log n) < O(n²) < O(n³) < O(2ⁿ) < O(n!)
```

## Визуализация

### График роста функций

```
Операции
    ↑
10⁸ │                                          · n!
    │                                       ·
    │                                    ·
    │                                 · 2ⁿ
10⁶ │                              ·
    │                           ·
    │                        ·      _____ n²
    │                     · ____----
10⁴ │                  · -╱
    │               ·-╱      _____ n log n
    │            ·╱     ___-╱
    │         ·╱   __--╱
10² │      ·╱  __-╱    _________ n
    │   ·╱___-╱    ____╱
    │·╱__-╱   ____╱ .............. log n
    │-_______╱     ........................
  1 │══════════════════════════════════════ 1
    └──────┬──────┬──────┬──────┬──────→ n
          10    100   1000  10000
```

### Таблица роста

| n | O(1) | O(log n) | O(n) | O(n log n) | O(n²) | O(2ⁿ) | O(n!) |
|---|------|----------|------|------------|-------|-------|-------|
| 1 | 1 | 0 | 1 | 0 | 1 | 2 | 1 |
| 10 | 1 | 3 | 10 | 33 | 100 | 1024 | 3.6M |
| 100 | 1 | 7 | 100 | 664 | 10K | 10³⁰ | ∞ |
| 1000 | 1 | 10 | 1K | 10K | 1M | 10³⁰⁰ | ∞ |
| 10⁶ | 1 | 20 | 10⁶ | 2×10⁷ | 10¹² | ∞ | ∞ |

---

## O(1) — Константная сложность

### Описание
Время выполнения **не зависит** от размера входных данных.

### Примеры

```python
# Доступ по индексу
def get_first(arr):
    return arr[0]

# Арифметические операции
def calculate(a, b):
    return a * b + a - b

# Доступ к хеш-таблице (средний случай)
def lookup(dictionary, key):
    return dictionary.get(key)

# Проверка чётности
def is_even(n):
    return n % 2 == 0
```

### Типичные операции O(1)
- Доступ к элементу массива: `arr[i]`
- Push/pop в конец списка: `list.append()`, `list.pop()`
- Операции с хеш-таблицей: `dict[key]`, `set.add()`
- Математические вычисления
- Сравнения и присваивания

---

## O(log n) — Логарифмическая сложность

### Описание
Время растёт **логарифмически**. Каждый шаг уменьшает задачу в константное число раз (обычно вдвое).

### Визуализация

```
n = 16:    [████████████████]
           ↓ делим пополам
n = 8:     [████████]
           ↓
n = 4:     [████]
           ↓
n = 2:     [██]
           ↓
n = 1:     [█]

Всего 4 шага = log₂(16)
```

### Примеры

```python
# Бинарный поиск
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
    return -1

# Поиск в BST
def search_bst(node, value):
    if node is None or node.val == value:
        return node
    if value < node.val:
        return search_bst(node.left, value)
    return search_bst(node.right, value)

# Возведение в степень
def power(base, exp):
    if exp == 0:
        return 1
    if exp % 2 == 0:
        half = power(base, exp // 2)
        return half * half
    return base * power(base, exp - 1)
```

### Типичные операции O(log n)
- Бинарный поиск
- Операции с BST (сбалансированным)
- Деление пополам в цикле
- Поиск в B-tree

---

## O(n) — Линейная сложность

### Описание
Время **пропорционально** размеру входных данных. Нужно обработать каждый элемент.

### Примеры

```python
# Поиск максимума
def find_max(arr):
    max_val = arr[0]
    for x in arr:
        if x > max_val:
            max_val = x
    return max_val

# Линейный поиск
def linear_search(arr, target):
    for i, x in enumerate(arr):
        if x == target:
            return i
    return -1

# Суммирование
def sum_array(arr):
    return sum(arr)

# Подсчёт элементов
def count_if(arr, predicate):
    count = 0
    for x in arr:
        if predicate(x):
            count += 1
    return count
```

### Типичные операции O(n)
- Проход по массиву/списку
- Поиск min/max
- Проверка `x in list`
- Копирование массива
- Подсчёт элементов

---

## O(n log n) — Линейно-логарифмическая сложность

### Описание
Типичная сложность эффективных алгоритмов сортировки. Представляет собой n операций, каждая из которых делит задачу пополам.

### Визуализация (Merge Sort)

```
Уровень 0:  [8,3,1,5,2,7,4,6]           — 1 массив
            ↙            ↘
Уровень 1:  [8,3,1,5]    [2,7,4,6]      — 2 массива
            ↙    ↘        ↙    ↘
Уровень 2:  [8,3] [1,5]  [2,7] [4,6]    — 4 массива
            ↙↘    ↙↘      ↙↘    ↙↘
Уровень 3:  [8][3][1][5] [2][7][4][6]   — 8 массивов

log₂(8) = 3 уровня
На каждом уровне O(n) операций слияния
Итого: O(n log n)
```

### Примеры

```python
# Сортировка слиянием
def merge_sort(arr):
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])
    right = merge_sort(arr[mid:])

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

# Быстрая сортировка (средний случай)
def quicksort(arr):
    if len(arr) <= 1:
        return arr
    pivot = arr[len(arr) // 2]
    left = [x for x in arr if x < pivot]
    middle = [x for x in arr if x == pivot]
    right = [x for x in arr if x > pivot]
    return quicksort(left) + middle + quicksort(right)

# Heapsort
import heapq
def heapsort(arr):
    heapq.heapify(arr)  # O(n)
    return [heapq.heappop(arr) for _ in range(len(arr))]  # O(n log n)
```

### Типичные операции O(n log n)
- Merge Sort, Quick Sort (средний), Heap Sort
- Встроенные сортировки (Timsort)
- Некоторые алгоритмы на графах
- FFT (быстрое преобразование Фурье)

---

## O(n²) — Квадратичная сложность

### Описание
Время растёт **квадратично**. Обычно возникает при вложенных циклах, каждый из которых проходит по n элементам.

### Визуализация

```
n=5: Сравнение каждого с каждым

    1  2  3  4  5
  ┌──┬──┬──┬──┬──┐
1 │░░│▓▓│▓▓│▓▓│▓▓│
  ├──┼──┼──┼──┼──┤
2 │▓▓│░░│▓▓│▓▓│▓▓│
  ├──┼──┼──┼──┼──┤
3 │▓▓│▓▓│░░│▓▓│▓▓│
  ├──┼──┼──┼──┼──┤
4 │▓▓│▓▓│▓▓│░░│▓▓│
  ├──┼──┼──┼──┼──┤
5 │▓▓│▓▓│▓▓│▓▓│░░│
  └──┴──┴──┴──┴──┘

25 клеток = 5² сравнений
```

### Примеры

```python
# Пузырьковая сортировка
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        for j in range(n - 1 - i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]

# Поиск дубликатов (наивный)
def has_duplicates(arr):
    n = len(arr)
    for i in range(n):
        for j in range(i + 1, n):
            if arr[i] == arr[j]:
                return True
    return False

# Умножение матриц (наивное)
def matrix_multiply_naive(A, B):
    n = len(A)
    C = [[0] * n for _ in range(n)]
    for i in range(n):
        for j in range(n):
            for k in range(n):  # Это делает O(n³)!
                C[i][j] += A[i][k] * B[k][j]
    return C

# Все пары
def print_pairs(arr):
    for i in arr:
        for j in arr:
            print(i, j)
```

### Типичные операции O(n²)
- Bubble Sort, Insertion Sort, Selection Sort
- Вложенные циклы по n элементам
- Наивные алгоритмы поиска дубликатов
- Сравнение каждого с каждым

---

## O(n³) — Кубическая сложность

### Описание
Три вложенных цикла. Часто встречается в алгоритмах на матрицах и графах.

### Примеры

```python
# Умножение матриц
def matrix_multiply(A, B):
    n = len(A)
    result = [[0] * n for _ in range(n)]

    for i in range(n):
        for j in range(n):
            for k in range(n):
                result[i][j] += A[i][k] * B[k][j]

    return result

# Алгоритм Флойда-Уоршелла (кратчайшие пути)
def floyd_warshall(graph):
    n = len(graph)
    dist = [row[:] for row in graph]

    for k in range(n):
        for i in range(n):
            for j in range(n):
                dist[i][j] = min(dist[i][j],
                                 dist[i][k] + dist[k][j])
    return dist
```

### Типичные операции O(n³)
- Наивное умножение матриц
- Алгоритм Флойда-Уоршелла
- Некоторые алгоритмы динамического программирования

---

## O(2ⁿ) — Экспоненциальная сложность

### Описание
Время **удваивается** с каждым дополнительным элементом. Быстро становится неприемлемым.

### Визуализация

```
n=1: 2 вызова      ●
                  / \
n=2: 4 вызова    ●   ●
                /\ /\
n=3: 8 вызовов ●●●●●●●●

Каждый уровень удваивает количество
```

### Примеры

```python
# Наивный Фибоначчи
def fib_naive(n):
    if n <= 1:
        return n
    return fib_naive(n - 1) + fib_naive(n - 2)

# Все подмножества
def subsets(arr):
    if not arr:
        return [[]]

    first = arr[0]
    rest = subsets(arr[1:])

    with_first = [[first] + s for s in rest]
    return rest + with_first

# Задача о рюкзаке (наивное решение)
def knapsack_naive(weights, values, capacity, n):
    if n == 0 or capacity == 0:
        return 0

    if weights[n-1] > capacity:
        return knapsack_naive(weights, values, capacity, n-1)

    include = values[n-1] + knapsack_naive(weights, values,
                                            capacity - weights[n-1], n-1)
    exclude = knapsack_naive(weights, values, capacity, n-1)

    return max(include, exclude)
```

### Типичные операции O(2ⁿ)
- Наивная рекурсия (Фибоначчи, Ханойские башни)
- Генерация всех подмножеств
- Полный перебор (brute force)
- Некоторые NP-полные задачи

---

## O(n!) — Факториальная сложность

### Описание
**Катастрофически** медленный рост. Практически невозможно выполнить для n > 10-12.

### Таблица факториалов

| n | n! |
|---|-----|
| 5 | 120 |
| 10 | 3,628,800 |
| 15 | 1.3 × 10¹² |
| 20 | 2.4 × 10¹⁸ |

### Примеры

```python
# Все перестановки
def permutations(arr):
    if len(arr) <= 1:
        return [arr]

    result = []
    for i, x in enumerate(arr):
        rest = arr[:i] + arr[i+1:]
        for perm in permutations(rest):
            result.append([x] + perm)

    return result

# Задача коммивояжёра (полный перебор)
def tsp_brute_force(distances):
    n = len(distances)
    cities = list(range(1, n))  # Все кроме стартового

    min_dist = float('inf')

    for perm in permutations(cities):
        dist = distances[0][perm[0]]
        for i in range(len(perm) - 1):
            dist += distances[perm[i]][perm[i+1]]
        dist += distances[perm[-1]][0]
        min_dist = min(min_dist, dist)

    return min_dist
```

### Типичные операции O(n!)
- Генерация всех перестановок
- Задача коммивояжёра (полный перебор)
- Некоторые задачи комбинаторной оптимизации

---

## Практические ограничения

### Максимальный n для 1 секунды выполнения

| Сложность | Максимальный n |
|-----------|----------------|
| O(n!) | ~10-11 |
| O(2ⁿ) | ~20-25 |
| O(n³) | ~500 |
| O(n²) | ~10,000 |
| O(n log n) | ~10,000,000 |
| O(n) | ~100,000,000 |
| O(log n) | Практически неограничен |
| O(1) | Неограничен |

### Шпаргалка для собеседований

```
n ≤ 10       → O(n!), O(n · n!)
n ≤ 20-25    → O(2ⁿ)
n ≤ 500      → O(n³)
n ≤ 10⁴      → O(n²)
n ≤ 10⁶      → O(n log n)
n ≤ 10⁸      → O(n)
n > 10⁸      → O(log n), O(1)
```

## Типичные ошибки

### 1. Неправильная оценка вложенных циклов

```python
# O(n), не O(n²)!
for i in range(n):
    for j in range(10):  # Константа
        print(i, j)

# O(n²)
for i in range(n):
    for j in range(i):   # Зависит от n
        print(i, j)
```

### 2. Скрытая сложность операций

```python
# Выглядит как O(n), но это O(n²)
result = ""
for s in strings:
    result += s  # Конкатенация — O(len(result))

# O(n log n), не O(n)
for x in arr:
    sorted_arr = sorted(arr)  # O(n log n) на каждой итерации!
```

### 3. Игнорирование сложности рекурсии

```python
# Это O(2ⁿ), не O(n)!
def fib(n):
    if n <= 1:
        return n
    return fib(n-1) + fib(n-2)
```

## Резюме

- **O(1)**: мгновенно, идеально
- **O(log n)**: очень быстро, отлично для больших n
- **O(n)**: линейно, приемлемо
- **O(n log n)**: стандарт для сортировки
- **O(n²)**: медленно для больших n, осторожно
- **O(2ⁿ)**: только для малых n (< 25)
- **O(n!)**: только для очень малых n (< 12)

Знание этих классов — ключ к написанию эффективного кода!

---

[prev: 04-big-theta](./04-big-theta.md) | [next: 06-space-complexity](./06-space-complexity.md)
