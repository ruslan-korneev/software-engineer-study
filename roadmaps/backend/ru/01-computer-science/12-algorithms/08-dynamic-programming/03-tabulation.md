# Табуляция (Tabulation)

## Определение

**Табуляция** — это подход "снизу вверх" (bottom-up) к динамическому программированию. Вместо рекурсии мы итеративно заполняем таблицу, начиная с базовых случаев и двигаясь к решению исходной задачи.

Табуляция = Итерация + Таблица результатов

## Зачем нужен

### Преимущества:
- **Нет рекурсии** — нет риска переполнения стека
- **Эффективность** — меньше накладных расходов, чем у рекурсии
- **Оптимизация памяти** — легче уменьшить использование памяти
- **Предсказуемость** — понятный порядок вычислений

### Когда использовать:
- Глубокая рекурсия может вызвать stack overflow
- Нужно вычислить все подзадачи
- Важна оптимизация памяти

## Как работает

### Визуализация

```
Числа Фибоначчи:

Табуляция (заполняем таблицу слева направо):

Индекс:  0   1   2   3   4   5   6   7   8   9   10
        ─────────────────────────────────────────────
dp:    │ 0 │ 1 │ 1 │ 2 │ 3 │ 5 │ 8 │13 │21 │34 │55 │
        ─────────────────────────────────────────────
              ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑   ↑
              │   │   │   │   │   │   │   │   │
            base  dp[0]+dp[1]                 ...
            case  =0+1=1

Каждая клетка зависит от двух предыдущих.
Заполняем строго слева направо.
```

### Схема работы

```
1. Создать таблицу dp нужного размера
2. Инициализировать базовые случаи
3. Заполнить таблицу в правильном порядке
4. Вернуть ответ из нужной ячейки

┌─────────────────────────────────────┐
│  for i in range(base, target):     │
│      dp[i] = f(dp[...], dp[...])   │
└─────────────────────────────────────┘
```

## Реализация на Python

### Фибоначчи: базовая табуляция

```python
def fib_tab(n):
    """Фибоначчи с табуляцией."""
    if n <= 1:
        return n

    dp = [0] * (n + 1)
    dp[0] = 0
    dp[1] = 1

    for i in range(2, n + 1):
        dp[i] = dp[i - 1] + dp[i - 2]

    return dp[n]

print(fib_tab(10))  # 55
```

### Фибоначчи: оптимизация памяти

```python
def fib_optimized(n):
    """Фибоначчи с O(1) памяти."""
    if n <= 1:
        return n

    prev2, prev1 = 0, 1

    for _ in range(2, n + 1):
        curr = prev1 + prev2
        prev2 = prev1
        prev1 = curr

    return prev1

print(fib_optimized(10))  # 55
```

### Размен монет

```python
def coin_change_tab(coins, amount):
    """Минимальное количество монет."""
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0  # Базовый случай: 0 монет для суммы 0

    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i and dp[i - coin] != float('inf'):
                dp[i] = min(dp[i], dp[i - coin] + 1)

    return dp[amount] if dp[amount] != float('inf') else -1

print(coin_change_tab([1, 5, 10, 25], 67))  # 6
```

### Количество способов разменять

```python
def coin_change_ways(coins, amount):
    """Количество способов разменять сумму."""
    dp = [0] * (amount + 1)
    dp[0] = 1  # Один способ: не брать ничего

    for coin in coins:
        for i in range(coin, amount + 1):
            dp[i] += dp[i - coin]

    return dp[amount]

print(coin_change_ways([1, 5, 10], 10))  # 4 способа
```

### Задача о рюкзаке

```python
def knapsack_tab(weights, values, capacity):
    """0/1 Рюкзак с табуляцией."""
    n = len(weights)

    # dp[i][w] = макс. ценность для первых i предметов и ёмкости w
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            # Не берём предмет i
            dp[i][w] = dp[i - 1][w]

            # Берём предмет i (если помещается)
            if weights[i - 1] <= w:
                dp[i][w] = max(
                    dp[i][w],
                    dp[i - 1][w - weights[i - 1]] + values[i - 1]
                )

    return dp[n][capacity]

weights = [2, 3, 4, 5]
values = [3, 4, 5, 6]
print(knapsack_tab(weights, values, 5))  # 7
```

### Рюкзак с оптимизацией памяти

```python
def knapsack_1d(weights, values, capacity):
    """Рюкзак с O(capacity) памяти."""
    n = len(weights)
    dp = [0] * (capacity + 1)

    for i in range(n):
        # Важно: обходим справа налево!
        for w in range(capacity, weights[i] - 1, -1):
            dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

    return dp[capacity]
```

### LCS (Longest Common Subsequence)

```python
def lcs_tab(s1, s2):
    """Длина наибольшей общей подпоследовательности."""
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

print(lcs_tab("ABCDGH", "AEDFHR"))  # 3
```

### Восстановление решения

```python
def lcs_with_solution(s1, s2):
    """LCS с восстановлением самой подпоследовательности."""
    n, m = len(s1), len(s2)
    dp = [[0] * (m + 1) for _ in range(n + 1)]

    # Заполняем таблицу
    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if s1[i - 1] == s2[j - 1]:
                dp[i][j] = dp[i - 1][j - 1] + 1
            else:
                dp[i][j] = max(dp[i - 1][j], dp[i][j - 1])

    # Восстанавливаем LCS
    lcs = []
    i, j = n, m
    while i > 0 and j > 0:
        if s1[i - 1] == s2[j - 1]:
            lcs.append(s1[i - 1])
            i -= 1
            j -= 1
        elif dp[i - 1][j] > dp[i][j - 1]:
            i -= 1
        else:
            j -= 1

    return ''.join(reversed(lcs))

print(lcs_with_solution("ABCDGH", "AEDFHR"))  # "ADH"
```

## Анализ сложности

| Задача | Время | Память | Оптимизированная |
|--------|-------|--------|------------------|
| Фибоначчи | O(n) | O(n) | O(1) |
| Размен монет | O(amount × coins) | O(amount) | — |
| 0/1 Рюкзак | O(n × capacity) | O(n × capacity) | O(capacity) |
| LCS | O(n × m) | O(n × m) | O(min(n, m)) |

## Типичные ошибки и Edge Cases

### 1. Неправильный порядок обхода

```python
# 0/1 Рюкзак с 1D массивом

# ОШИБКА: слева направо — предмет может использоваться несколько раз!
for i in range(n):
    for w in range(weights[i], capacity + 1):  # Неправильно!
        dp[w] = max(dp[w], dp[w - weights[i]] + values[i])

# ПРАВИЛЬНО: справа налево
for i in range(n):
    for w in range(capacity, weights[i] - 1, -1):  # Правильно!
        dp[w] = max(dp[w], dp[w - weights[i]] + values[i])
```

### 2. Забыли инициализировать базовые случаи

```python
# ОШИБКА
def wrong_coin_change(coins, amount):
    dp = [0] * (amount + 1)  # Все нули — неправильно!
    for i in range(1, amount + 1):
        for coin in coins:
            dp[i] = min(dp[i], dp[i - coin] + 1)  # dp[0] = 0, но...

# ПРАВИЛЬНО
def correct_coin_change(coins, amount):
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0  # Базовый случай!
    # ...
```

### 3. Индексация

```python
# Частая ошибка: off-by-one

# Если dp[i] означает "для первых i элементов":
# - Размер dp: n + 1
# - Индексы: 0 до n включительно
# - Элементы массива: arr[0] до arr[n-1]
# - Связь: dp[i] использует arr[i-1]

for i in range(1, n + 1):
    # Используем arr[i-1], не arr[i]!
    dp[i] = f(dp[i-1], arr[i-1])
```

### 4. Инициализация 2D массива

```python
# ОШИБКА: все строки — один список
dp = [[0] * m] * n
dp[0][0] = 1  # Изменит ВСЕ строки!

# ПРАВИЛЬНО
dp = [[0] * m for _ in range(n)]
```

### 5. Оптимизация памяти для 2D DP

```python
# Если dp[i] зависит только от dp[i-1]:

# Было: O(n × m) памяти
dp = [[0] * m for _ in range(n)]

# Стало: O(m) памяти
prev = [0] * m
curr = [0] * m

for i in range(n):
    for j in range(m):
        curr[j] = f(prev[j], curr[j-1], ...)
    prev, curr = curr, [0] * m
```

### Рекомендации:
1. Всегда рисуйте таблицу перед реализацией
2. Проверяйте зависимости: от каких ячеек зависит текущая?
3. Выбирайте порядок обхода, чтобы зависимости были вычислены
4. Инициализируйте базовые случаи явно
5. Для оптимизации памяти определите, сколько "строк" реально нужно
