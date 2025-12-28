# Классические задачи динамического программирования

[prev: 03-tabulation](./03-tabulation.md) | [next: 01-decision-problems](../../13-complexity-classes/01-decision-problems.md)
---

## Обзор

Рассмотрим три фундаментальные задачи DP: числа Фибоначчи, задача о рюкзаке и наибольшая общая подпоследовательность (LCS).

---

## 1. Числа Фибоначчи (Fibonacci)

### Определение
F(0) = 0, F(1) = 1, F(n) = F(n-1) + F(n-2)

### Визуализация

```
n:    0   1   2   3   4   5   6   7   8   9   10
F(n): 0   1   1   2   3   5   8   13  21  34  55

Зависимости:
F(5) = F(4) + F(3)
     = (F(3) + F(2)) + (F(2) + F(1))
     = ...
```

### Реализации

```python
# 1. Наивная рекурсия — O(2^n)
def fib_naive(n):
    if n <= 1:
        return n
    return fib_naive(n - 1) + fib_naive(n - 2)

# 2. Мемоизация — O(n) время, O(n) память
from functools import lru_cache

@lru_cache(maxsize=None)
def fib_memo(n):
    if n <= 1:
        return n
    return fib_memo(n - 1) + fib_memo(n - 2)

# 3. Табуляция — O(n) время, O(n) память
def fib_tab(n):
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]
    return dp[n]

# 4. Оптимизированный — O(n) время, O(1) память
def fib_opt(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b

# 5. Матричное возведение в степень — O(log n)
def fib_matrix(n):
    if n <= 1:
        return n

    def multiply(A, B):
        return [
            [A[0][0]*B[0][0] + A[0][1]*B[1][0],
             A[0][0]*B[0][1] + A[0][1]*B[1][1]],
            [A[1][0]*B[0][0] + A[1][1]*B[1][0],
             A[1][0]*B[0][1] + A[1][1]*B[1][1]]
        ]

    def power(M, n):
        if n == 1:
            return M
        if n % 2 == 0:
            half = power(M, n // 2)
            return multiply(half, half)
        else:
            return multiply(M, power(M, n - 1))

    M = [[1, 1], [1, 0]]
    result = power(M, n)
    return result[0][1]
```

### Сравнение подходов

| Метод | Время | Память | Примечание |
|-------|-------|--------|------------|
| Наивный | O(2^n) | O(n) | Неприменим для n > 40 |
| Мемоизация | O(n) | O(n) | Просто добавить |
| Табуляция | O(n) | O(n) | Без рекурсии |
| Оптимизированный | O(n) | O(1) | Лучший для практики |
| Матричный | O(log n) | O(1) | Для очень больших n |

---

## 2. Задача о рюкзаке (Knapsack Problem)

### Варианты задачи

```
1. 0/1 Рюкзак: каждый предмет можно взять только один раз
2. Неограниченный: каждый предмет можно брать сколько угодно
3. Дробный: можно брать части предметов (жадный алгоритм)
```

### 0/1 Рюкзак

```python
def knapsack_01(weights, values, capacity):
    """
    Максимизировать ценность при ограниченной ёмкости.
    Каждый предмет можно взять не более одного раза.
    """
    n = len(weights)

    # dp[i][w] = макс. ценность для первых i предметов и ёмкости w
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            # Не берём предмет i
            dp[i][w] = dp[i - 1][w]

            # Берём предмет i
            if weights[i - 1] <= w:
                dp[i][w] = max(
                    dp[i][w],
                    dp[i - 1][w - weights[i - 1]] + values[i - 1]
                )

    return dp[n][capacity]


# Оптимизация памяти: O(capacity)
def knapsack_01_optimized(weights, values, capacity):
    n = len(weights)
    dp = [0] * (capacity + 1)

    for i in range(n):
        # Обход справа налево!
        for w in range(capacity, weights[i] - 1, -1):
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[capacity]


# С восстановлением решения
def knapsack_01_with_items(weights, values, capacity):
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            dp[i][w] = dp[i - 1][w]
            if weights[i - 1] <= w:
                dp[i][w] = max(
                    dp[i][w],
                    dp[i - 1][w - weights[i - 1]] + values[i - 1]
                )

    # Восстановление
    selected = []
    w = capacity
    for i in range(n, 0, -1):
        if dp[i][w] != dp[i - 1][w]:
            selected.append(i - 1)
            w -= weights[i - 1]

    return dp[n][capacity], selected[::-1]


# Пример
weights = [2, 3, 4, 5]
values = [3, 4, 5, 6]
capacity = 5

max_value, items = knapsack_01_with_items(weights, values, capacity)
print(f"Максимальная ценность: {max_value}")  # 7
print(f"Выбранные предметы: {items}")         # [0, 1] (веса 2+3=5, ценность 3+4=7)
```

### Неограниченный рюкзак

```python
def knapsack_unbounded(weights, values, capacity):
    """Каждый предмет можно брать неограниченно."""
    dp = [0] * (capacity + 1)

    for w in range(1, capacity + 1):
        for i in range(len(weights)):
            if weights[i] <= w:
                dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[capacity]
```

### Визуализация 0/1 Рюкзака

```
weights = [1, 2, 3], values = [6, 10, 12], capacity = 5

       ёмкость w
       0   1   2   3   4   5
     ┌───┬───┬───┬───┬───┬───┐
i=0  │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │  (пустой рюкзак)
     ├───┼───┼───┼───┼───┼───┤
i=1  │ 0 │ 6 │ 6 │ 6 │ 6 │ 6 │  (только предмет 1: w=1, v=6)
     ├───┼───┼───┼───┼───┼───┤
i=2  │ 0 │ 6 │10 │16 │16 │16 │  (+ предмет 2: w=2, v=10)
     ├───┼───┼───┼───┼───┼───┤
i=3  │ 0 │ 6 │10 │16 │18 │22 │  (+ предмет 3: w=3, v=12)
     └───┴───┴───┴───┴───┴───┘

Ответ: dp[3][5] = 22 (предметы 2 и 3: 10 + 12)
```

---

## 3. Наибольшая общая подпоследовательность (LCS)

### Определение
Найти самую длинную последовательность символов, которая встречается в обеих строках (не обязательно подряд).

### Визуализация

```
s1 = "ABCDGH"
s2 = "AEDFHR"

       A  B  C  D  G  H
    ┌──────────────────
A   │  A
E   │
D   │           D
F   │
H   │                 H
R   │

LCS = "ADH" (длина 3)
```

### Реализация

```python
def lcs(s1, s2):
    """Длина LCS."""
    n, m = len(s1), len(s2)

    # dp[i][j] = длина LCS для s1[:i] и s2[:j]
    dp = [[0] * (m + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if s1[i - 1] == s2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    return dp[n][m]


def lcs_string(s1, s2):
    """LCS с восстановлением подпоследовательности."""
    n, m = len(s1), len(s2)
    dp = [[0] * (m + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if s1[i - 1] == s2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    # Восстановление
    result = []
    i, j = n, m
    while i > 0 and j > 0:
        if s1[i - 1] == s2[j - 1]:
            result.append(s1[i - 1])
            i -= 1
            j -= 1
        elif dp[i - 1][j] > dp[i][j - 1]:
            i -= 1
        else:
            j -= 1

    return ''.join(reversed(result))


# Оптимизация памяти: O(min(n, m))
def lcs_optimized(s1, s2):
    """LCS с O(min(n, m)) памяти."""
    if len(s1) < len(s2):
        s1, s2 = s2, s1

    m = len(s2)
    prev = [0] * (m + 1)
    curr = [0] * (m + 1)

    for i in range(1, len(s1) + 1):
        for j in range(1, m + 1):
            if s1[i - 1] == s2[j - 1]:
                curr[j] = prev[j - 1] + 1
            else:
                curr[j] = max(prev[j], curr[j - 1])
        prev, curr = curr, [0] * (m + 1)

    return prev[m]


# Пример
s1 = "ABCDGH"
s2 = "AEDFHR"
print(f"Длина LCS: {lcs(s1, s2)}")      # 3
print(f"LCS: {lcs_string(s1, s2)}")     # "ADH"
```

### Таблица DP для LCS

```
s1 = "AGGTAB", s2 = "GXTXAYB"

        ""  G   X   T   X   A   Y   B
     ┌────┬───┬───┬───┬───┬───┬───┬───┐
""   │  0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │ 0 │
     ├────┼───┼───┼───┼───┼───┼───┼───┤
A    │  0 │ 0 │ 0 │ 0 │ 0 │ 1 │ 1 │ 1 │
     ├────┼───┼───┼───┼───┼───┼───┼───┤
G    │  0 │ 1 │ 1 │ 1 │ 1 │ 1 │ 1 │ 1 │
     ├────┼───┼───┼───┼───┼───┼───┼───┤
G    │  0 │ 1 │ 1 │ 1 │ 1 │ 1 │ 1 │ 1 │
     ├────┼───┼───┼───┼───┼───┼───┼───┤
T    │  0 │ 1 │ 1 │ 2 │ 2 │ 2 │ 2 │ 2 │
     ├────┼───┼───┼───┼───┼───┼───┼───┤
A    │  0 │ 1 │ 1 │ 2 │ 2 │ 3 │ 3 │ 3 │
     ├────┼───┼───┼───┼───┼───┼───┼───┤
B    │  0 │ 1 │ 1 │ 2 │ 2 │ 3 │ 3 │ 4 │
     └────┴───┴───┴───┴───┴───┴───┴───┘

LCS длина = 4 ("GTAB")
```

---

## Связанные задачи

### Редакционное расстояние (Edit Distance)

```python
def edit_distance(s1, s2):
    """Минимум операций для превращения s1 в s2."""
    n, m = len(s1), len(s2)
    dp = [[0] * (m + 1) for _ in range(n + 1)]

    # Базовые случаи
    for i in range(n + 1):
        dp[i][0] = i
    for j in range(m + 1):
        dp[0][j] = j

    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if s1[i - 1] == s2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1]
            else:
                dp[i][j] = 1 + min(
                    dp[i - 1][j],      # удаление
                    dp[i][j - 1],      # вставка
                    dp[i - 1][j - 1]   # замена
                )

    return dp[n][m]

print(edit_distance("kitten", "sitting"))  # 3
```

### Longest Increasing Subsequence (LIS)

```python
def lis(nums):
    """Длина наибольшей возрастающей подпоследовательности."""
    if not nums:
        return 0

    # O(n^2) версия
    n = len(nums)
    dp = [1] * n

    for i in range(1, n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)

    return max(dp)


# O(n log n) версия с бинарным поиском
import bisect

def lis_fast(nums):
    tails = []
    for num in nums:
        pos = bisect.bisect_left(tails, num)
        if pos == len(tails):
            tails.append(num)
        else:
            tails[pos] = num
    return len(tails)

print(lis([10, 9, 2, 5, 3, 7, 101, 18]))  # 4
```

---

## Рекомендации

1. **Фибоначчи** — отличная задача для понимания DP
2. **Рюкзак** — классика оптимизации; обратите внимание на направление обхода
3. **LCS** — основа для многих строковых алгоритмов
4. Всегда рисуйте таблицу для понимания зависимостей
5. Оптимизируйте память, когда таблица большая

---
[prev: 03-tabulation](./03-tabulation.md) | [next: 01-decision-problems](../../13-complexity-classes/01-decision-problems.md)