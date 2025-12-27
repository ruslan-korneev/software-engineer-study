# Space Complexity (Пространственная сложность)

## Определение

**Пространственная сложность** — это мера количества памяти, которое использует алгоритм в зависимости от размера входных данных. Она показывает, как растёт потребление памяти при увеличении n.

### Что учитывается

1. **Auxiliary Space (Вспомогательная память)** — дополнительная память, выделяемая алгоритмом
2. **Input Space (Память для входных данных)** — память для хранения входа

```
Space Complexity = Auxiliary Space + Input Space
```

Обычно, когда говорят о пространственной сложности, имеют в виду **Auxiliary Space** — сколько дополнительной памяти нужно сверх входных данных.

## Зачем нужно

### Практическое применение

1. **Ограничения памяти** — встраиваемые системы, мобильные устройства
2. **Большие данные** — когда данные не помещаются в RAM
3. **Trade-off время/память** — иногда можно ускорить алгоритм за счёт памяти
4. **Рекурсия** — глубина стека вызовов ограничена
5. **Кэш-эффективность** — меньше памяти = лучше использование кэша

### Пример trade-off

```python
# Вариант 1: O(n) времени, O(n) памяти
def has_duplicate_set(arr):
    seen = set()
    for x in arr:
        if x in seen:
            return True
        seen.add(x)
    return False

# Вариант 2: O(n²) времени, O(1) памяти
def has_duplicate_naive(arr):
    for i in range(len(arr)):
        for j in range(i + 1, len(arr)):
            if arr[i] == arr[j]:
                return True
    return False
```

## Как работает

### Подсчёт памяти

Учитываем:
- Примитивные типы (int, float) — константа
- Массивы/списки — пропорционально длине
- Объекты — сумма полей
- Стек вызовов (для рекурсии) — глубина рекурсии

### Правила оценки

1. **Игнорируем константы** — O(100) = O(1)
2. **Берём старший член** — O(n + log n) = O(n)
3. **Вход не считаем** (обычно) — только дополнительная память

## Визуализация

### Классы пространственной сложности

```
Память
    ↑
    │                              ╱ O(n²)
    │                           ╱
    │                        ╱
    │                     ╱   _____ O(n)
    │                  ╱ ____╱
    │               ╱_╱
    │            __╱    ......... O(log n)
    │       ___-╱   ............
    │  ___--╱     .............
    │_-╱════════════════════════ O(1)
    └────────────────────────────→ n
```

### Сравнение сложностей

| Сложность | 10 | 100 | 1000 | 10⁶ |
|-----------|-----|------|-------|------|
| O(1) | 1 | 1 | 1 | 1 |
| O(log n) | 3 | 7 | 10 | 20 |
| O(n) | 10 | 100 | 1000 | 10⁶ |
| O(n²) | 100 | 10⁴ | 10⁶ | 10¹² |

---

## Примеры по классам сложности

### O(1) — Константная память

```python
# Сумма массива — только переменная total
def sum_array(arr):
    total = 0            # O(1)
    for x in arr:
        total += x
    return total

# Поиск максимума — только переменная max_val
def find_max(arr):
    max_val = arr[0]     # O(1)
    for x in arr:
        if x > max_val:
            max_val = x
    return max_val

# Swap двух переменных
def swap(a, b):
    a, b = b, a          # O(1)
    return a, b

# Bubble sort — in-place, O(1) памяти
def bubble_sort(arr):
    n = len(arr)
    for i in range(n):
        for j in range(n - 1 - i):
            if arr[j] > arr[j + 1]:
                arr[j], arr[j + 1] = arr[j + 1], arr[j]
    # Сортировка на месте, без доп. памяти
```

### O(log n) — Логарифмическая память

```python
# Рекурсивный бинарный поиск
def binary_search(arr, target, left, right):
    if left > right:
        return -1

    mid = (left + right) // 2

    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        return binary_search(arr, target, mid + 1, right)
    else:
        return binary_search(arr, target, left, mid - 1)

# Глубина рекурсии = log n
# Каждый вызов использует константную память
# Итого: O(log n) памяти на стеке
```

**Визуализация стека:**
```
binary_search(arr, 5, 0, 15)  ← 1-й уровень
  binary_search(arr, 5, 0, 7)  ← 2-й уровень
    binary_search(arr, 5, 4, 7)  ← 3-й уровень
      binary_search(arr, 5, 6, 7)  ← 4-й уровень
        return 6  ← нашли!

log₂(16) = 4 уровня → O(log n) памяти
```

### O(n) — Линейная память

```python
# Копирование массива
def copy_array(arr):
    return arr[:]        # O(n) памяти

# Создание нового списка
def double_values(arr):
    result = []          # O(n) памяти
    for x in arr:
        result.append(x * 2)
    return result

# Хеш-таблица для подсчёта
def count_elements(arr):
    counts = {}          # O(n) в худшем случае
    for x in arr:
        counts[x] = counts.get(x, 0) + 1
    return counts

# Merge sort — требует O(n) доп. памяти
def merge_sort(arr):
    if len(arr) <= 1:
        return arr

    mid = len(arr) // 2
    left = merge_sort(arr[:mid])    # Копия — O(n/2)
    right = merge_sort(arr[mid:])   # Копия — O(n/2)

    return merge(left, right)       # Слияние — O(n)
```

### O(n²) — Квадратичная память

```python
# Матрица смежности графа
def create_adjacency_matrix(n):
    return [[0] * n for _ in range(n)]  # n × n = O(n²)

# Таблица динамического программирования
def lcs_table(s1, s2):
    m, n = len(s1), len(s2)
    dp = [[0] * (n + 1) for _ in range(m + 1)]  # O(m × n)

    for i in range(1, m + 1):
        for j in range(1, n + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])

    return dp[m][n]

# Все пары
def generate_pairs(arr):
    pairs = []
    for x in arr:
        for y in arr:
            pairs.append((x, y))  # O(n²) пар
    return pairs
```

---

## Рекурсия и стек вызовов

### Важность стека

Рекурсивные вызовы занимают место на стеке. Глубина рекурсии определяет пространственную сложность.

```python
# O(n) памяти — линейная рекурсия
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

# Стек при n=5:
# factorial(5)
#   factorial(4)
#     factorial(3)
#       factorial(2)
#         factorial(1) → return 1
#       ← return 2
#     ← return 6
#   ← return 24
# ← return 120
```

### Хвостовая рекурсия

```python
# Хвостовая рекурсия (теоретически O(1), но Python не оптимизирует)
def factorial_tail(n, acc=1):
    if n <= 1:
        return acc
    return factorial_tail(n - 1, n * acc)

# Итеративный вариант — O(1) памяти
def factorial_iterative(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

### Рекурсия с ветвлением

```python
# O(n) памяти (не O(2^n)!)
def fib(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib(n-1, memo) + fib(n-2, memo)
    return memo[n]

# Максимальная глубина стека = n
# memo хранит n значений
# Итого: O(n) памяти
```

---

## In-place алгоритмы

### Определение

**In-place алгоритм** — алгоритм, использующий O(1) дополнительной памяти (не считая рекурсивного стека).

### Примеры

```python
# In-place разворот массива
def reverse_in_place(arr):
    left, right = 0, len(arr) - 1
    while left < right:
        arr[left], arr[right] = arr[right], arr[left]
        left += 1
        right -= 1
    # O(1) доп. памяти

# In-place partition (Quick Sort)
def partition(arr, low, high):
    pivot = arr[high]
    i = low - 1

    for j in range(low, high):
        if arr[j] <= pivot:
            i += 1
            arr[i], arr[j] = arr[j], arr[i]

    arr[i + 1], arr[high] = arr[high], arr[i + 1]
    return i + 1
    # O(1) доп. памяти

# НЕ in-place — создаёт новый массив
def reverse_new(arr):
    return arr[::-1]  # O(n) памяти
```

### Сравнение сортировок по памяти

| Алгоритм | Время | Память | In-place |
|----------|-------|--------|----------|
| Bubble Sort | O(n²) | O(1) | Да |
| Insertion Sort | O(n²) | O(1) | Да |
| Selection Sort | O(n²) | O(1) | Да |
| Merge Sort | O(n log n) | O(n) | Нет |
| Quick Sort | O(n log n)* | O(log n) | Да |
| Heap Sort | O(n log n) | O(1) | Да |
| Timsort | O(n log n) | O(n) | Нет |

*Quick Sort: O(n²) в худшем случае

---

## Оптимизация памяти

### Техника 1: Переиспользование входного массива

```python
# Вместо создания нового массива
def double_new(arr):
    return [x * 2 for x in arr]  # O(n) памяти

# Модифицируем на месте
def double_in_place(arr):
    for i in range(len(arr)):
        arr[i] *= 2              # O(1) памяти
```

### Техника 2: Генераторы вместо списков

```python
# Создаёт список в памяти
def get_squares_list(n):
    return [x**2 for x in range(n)]  # O(n) памяти

# Генератор — O(1) памяти
def get_squares_gen(n):
    for x in range(n):
        yield x**2               # O(1) памяти

# Использование
for sq in get_squares_gen(1000000):
    process(sq)  # Обрабатываем по одному
```

### Техника 3: Итеративный вместо рекурсивного

```python
# Рекурсия — O(n) памяти на стеке
def sum_recursive(arr, index=0):
    if index >= len(arr):
        return 0
    return arr[index] + sum_recursive(arr, index + 1)

# Итеративно — O(1) памяти
def sum_iterative(arr):
    total = 0
    for x in arr:
        total += x
    return total
```

### Техника 4: Оптимизация DP таблиц

```python
# Стандартное DP — O(n) памяти
def fibonacci_dp(n):
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]

# Оптимизированное — O(1) памяти
def fibonacci_optimized(n):
    if n <= 1:
        return n
    prev2, prev1 = 0, 1
    for _ in range(2, n + 1):
        curr = prev1 + prev2
        prev2, prev1 = prev1, curr
    return prev1
```

---

## Типичные ошибки

### 1. Забывать про стек рекурсии

```python
# Кажется O(1), но стек — O(n)!
def count_down(n):
    if n <= 0:
        return
    print(n)
    count_down(n - 1)  # O(n) на стеке
```

### 2. Скрытое копирование

```python
# arr[1:] создаёт копию — O(n)!
def process_recursive(arr):
    if not arr:
        return
    print(arr[0])
    process_recursive(arr[1:])  # Каждый вызов создаёт копию!

# Лучше использовать индексы
def process_with_index(arr, i=0):
    if i >= len(arr):
        return
    print(arr[i])
    process_with_index(arr, i + 1)  # O(n) только на стеке
```

### 3. Накопление строк

```python
# Каждая конкатенация создаёт новую строку!
def build_string_bad(n):
    result = ""
    for i in range(n):
        result += str(i)  # O(n²) памяти суммарно!
    return result

# Используем join — O(n) памяти
def build_string_good(n):
    parts = [str(i) for i in range(n)]
    return "".join(parts)
```

### 4. Неучтённые структуры данных

```python
# dict и set занимают память!
def has_unique(arr):
    return len(arr) == len(set(arr))  # set — O(n) памяти

# Оба множества — O(n) каждое
def intersection(arr1, arr2):
    set1 = set(arr1)  # O(n)
    set2 = set(arr2)  # O(m)
    return set1 & set2  # Итого O(n + m)
```

---

## Сводная таблица

| Алгоритм/Операция | Время | Память |
|-------------------|-------|--------|
| Линейный поиск | O(n) | O(1) |
| Бинарный поиск (итеративный) | O(log n) | O(1) |
| Бинарный поиск (рекурсивный) | O(log n) | O(log n) |
| Bubble Sort | O(n²) | O(1) |
| Merge Sort | O(n log n) | O(n) |
| Quick Sort | O(n log n)* | O(log n) |
| Heap Sort | O(n log n) | O(1) |
| DFS (рекурсивный) | O(V+E) | O(V) |
| BFS | O(V+E) | O(V) |
| DP (таблица) | O(n²) | O(n²) |
| DP (оптимизированный) | O(n²) | O(n) |

## Резюме

- Пространственная сложность измеряет **дополнительную память**
- Рекурсия использует память на **стеке вызовов**
- **In-place** алгоритмы используют O(1) памяти
- Генераторы экономят память по сравнению со списками
- Часто возможен **trade-off** между временем и памятью
- Оптимизация DP таблиц может снизить O(n²) до O(n) или O(1)
