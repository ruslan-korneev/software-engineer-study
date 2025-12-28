# Динамическое программирование (Dynamic Programming)

[prev: 03-huffman-coding](../07-greedy/03-huffman-coding.md) | [next: 02-memoization](./02-memoization.md)
---

## Определение

**Динамическое программирование (DP)** — это метод решения сложных задач путём разбиения их на более простые подзадачи. Ключевая идея — **сохранение результатов подзадач** для избежания повторных вычислений.

DP применяется, когда задача имеет:
1. **Оптимальную подструктуру** — оптимальное решение содержит оптимальные решения подзадач
2. **Перекрывающиеся подзадачи** — одни и те же подзадачи решаются многократно

## Зачем нужен

### Области применения:
- **Оптимизационные задачи** — рюкзак, размен монет
- **Подсчёт вариантов** — пути, разбиения, комбинаторика
- **Строковые алгоритмы** — LCS, редакционное расстояние
- **Биоинформатика** — выравнивание последовательностей
- **Финансы** — оптимизация портфеля, ценообразование опционов
- **Машинное обучение** — алгоритм Витерби, CRF

### Сравнение подходов

```
Задача: Числа Фибоначчи

Рекурсия (наивная):     O(2^n) — повторные вычисления
DP (мемоизация):        O(n) — каждое число вычисляется один раз
DP (табуляция):         O(n) — итеративно снизу вверх
Оптимизированный:       O(n) время, O(1) память
```

## Как работает

### Два подхода к DP

```
1. СВЕРХУ ВНИЗ (Top-Down / Memoization)
   - Рекурсия с кэшированием
   - Вычисляем только нужные подзадачи
   - Проще понять и написать

2. СНИЗУ ВВЕРХ (Bottom-Up / Tabulation)
   - Итеративно заполняем таблицу
   - Вычисляем все подзадачи
   - Часто эффективнее по памяти
```

### Визуализация: Числа Фибоначчи

```
Наивная рекурсия (повторные вычисления):

                fib(5)
               /      \
           fib(4)      fib(3)
          /     \      /    \
       fib(3)  fib(2) fib(2) fib(1)
       /   \
    fib(2) fib(1)

fib(2) вычисляется 3 раза!
fib(3) вычисляется 2 раза!

Мемоизация:
- Вычисляем fib(2) → сохраняем в memo[2]
- При повторном вызове fib(2) → берём из memo
- Каждое значение вычисляется ОДИН раз

Табуляция:
dp[0] = 0
dp[1] = 1
dp[2] = dp[0] + dp[1] = 1
dp[3] = dp[1] + dp[2] = 2
dp[4] = dp[2] + dp[3] = 3
dp[5] = dp[3] + dp[4] = 5
```

### Шаги решения DP задачи

```
1. ОПРЕДЕЛИТЬ ПОДЗАДАЧИ
   Что мы хотим вычислить? Как параметризовать?
   Пример: dp[i] = i-е число Фибоначчи

2. НАЙТИ РЕКУРРЕНТНОЕ СООТНОШЕНИЕ
   Как связаны решения подзадач?
   Пример: dp[i] = dp[i-1] + dp[i-2]

3. ОПРЕДЕЛИТЬ БАЗОВЫЕ СЛУЧАИ
   Какие подзадачи решаются напрямую?
   Пример: dp[0] = 0, dp[1] = 1

4. ОПРЕДЕЛИТЬ ПОРЯДОК ВЫЧИСЛЕНИЯ
   В каком порядке решать подзадачи?
   Пример: от меньших к большим (0, 1, 2, ...)

5. ИЗВЛЕЧЬ ОТВЕТ
   Где находится решение исходной задачи?
   Пример: dp[n]
```

## Псевдокод

### Сверху вниз (Memoization)

```
function solve(n, memo):
    if n in memo:
        return memo[n]

    if is_base_case(n):
        return base_value(n)

    result = combine(solve(subproblem1, memo),
                     solve(subproblem2, memo), ...)

    memo[n] = result
    return result
```

### Снизу вверх (Tabulation)

```
function solve(n):
    dp = array of size n+1

    # Базовые случаи
    dp[0] = base_value_0
    dp[1] = base_value_1

    # Заполняем таблицу
    for i from 2 to n:
        dp[i] = combine(dp[...], dp[...])

    return dp[n]
```

### Примеры на Python

```python
# Фибоначчи: Мемоизация
def fib_memo(n, memo={}):
    if n in memo:
        return memo[n]
    if n <= 1:
        return n
    memo[n] = fib_memo(n-1, memo) + fib_memo(n-2, memo)
    return memo[n]

# Фибоначчи: Табуляция
def fib_tab(n):
    if n <= 1:
        return n
    dp = [0] * (n + 1)
    dp[1] = 1
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
    return dp[n]

# Фибоначчи: Оптимизация памяти
def fib_optimized(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b
```

## Классические задачи DP

### 1. Размен монет

```python
def coin_change(coins, amount):
    """Минимальное количество монет для суммы amount."""
    dp = [float('inf')] * (amount + 1)
    dp[0] = 0

    for i in range(1, amount + 1):
        for coin in coins:
            if coin <= i and dp[i - coin] + 1 < dp[i]:
                dp[i] = dp[i - coin] + 1

    return dp[amount] if dp[amount] != float('inf') else -1

print(coin_change([1, 5, 10, 25], 67))  # 6
```

### 2. Задача о рюкзаке 0/1

```python
def knapsack(weights, values, capacity):
    """Максимальная ценность при ограниченной ёмкости."""
    n = len(weights)
    dp = [[0] * (capacity + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for w in range(capacity + 1):
            # Не берём предмет i
            dp[i][w] = dp[i-1][w]
            # Берём предмет i (если помещается)
            if weights[i-1] <= w:
                dp[i][w] = max(dp[i][w],
                              dp[i-1][w - weights[i-1]] + values[i-1])

    return dp[n][capacity]

weights = [1, 2, 3, 4]
values = [10, 20, 30, 40]
print(knapsack(weights, values, 5))  # 50
```

### 3. Самая длинная возрастающая подпоследовательность (LIS)

```python
def lis(nums):
    """Длина LIS."""
    if not nums:
        return 0

    n = len(nums)
    dp = [1] * n

    for i in range(1, n):
        for j in range(i):
            if nums[j] < nums[i]:
                dp[i] = max(dp[i], dp[j] + 1)

    return max(dp)

print(lis([10, 9, 2, 5, 3, 7, 101, 18]))  # 4: [2, 3, 7, 18]
```

## Анализ сложности

| Задача | Время | Память |
|--------|-------|--------|
| Фибоначчи | O(n) | O(n) или O(1) |
| Размен монет | O(amount × coins) | O(amount) |
| 0/1 Рюкзак | O(n × capacity) | O(n × capacity) или O(capacity) |
| LIS | O(n^2) или O(n log n) | O(n) |
| LCS | O(n × m) | O(n × m) или O(min(n, m)) |

## Типичные ошибки и Edge Cases

### 1. Неправильный порядок заполнения

```python
# ОШИБКА: не те зависимости
def wrong_order():
    for i in range(n):
        dp[i] = dp[i+1] + dp[i+2]  # i+1, i+2 ещё не вычислены!

# ПРАВИЛЬНО: от большего к меньшему
def correct_order():
    for i in range(n-1, -1, -1):
        dp[i] = dp[i+1] + dp[i+2]
```

### 2. Забыли базовые случаи

```python
# ОШИБКА
def wrong_dp(n):
    dp = [0] * (n + 1)
    # Забыли dp[0], dp[1]!
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]  # dp[0], dp[1] = 0, неправильно!

# ПРАВИЛЬНО
def correct_dp(n):
    dp = [0] * (n + 1)
    dp[0] = 0
    dp[1] = 1  # Базовый случай!
    for i in range(2, n + 1):
        dp[i] = dp[i-1] + dp[i-2]
```

### 3. Выход за границы массива

```python
# ОШИБКА
def wrong_bounds(n):
    dp = [0] * n  # Размер n, индексы 0..n-1
    dp[n] = 1     # IndexError!

# ПРАВИЛЬНО
def correct_bounds(n):
    dp = [0] * (n + 1)  # Размер n+1, индексы 0..n
    dp[n] = 1           # OK
```

### 4. Не скопировали вложенный список

```python
# ОШИБКА: все строки ссылаются на один список
dp = [[0] * m] * n  # Неправильно!
dp[0][0] = 1        # Изменит все строки!

# ПРАВИЛЬНО
dp = [[0] * m for _ in range(n)]
```

### 5. Оптимизация памяти

```python
# До оптимизации: O(n × m) памяти
def lcs_full(s1, s2):
    n, m = len(s1), len(s2)
    dp = [[0] * (m + 1) for _ in range(n + 1)]

    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if s1[i-1] == s2[j-1]:
                dp[i][j] = dp[i-1][j-1] + 1
            else:
                dp[i][j] = max(dp[i-1][j], dp[i][j-1])

    return dp[n][m]

# После оптимизации: O(m) памяти
def lcs_optimized(s1, s2):
    n, m = len(s1), len(s2)
    prev = [0] * (m + 1)
    curr = [0] * (m + 1)

    for i in range(1, n + 1):
        for j in range(1, m + 1):
            if s1[i-1] == s2[j-1]:
                curr[j] = prev[j-1] + 1
            else:
                curr[j] = max(prev[j], curr[j-1])
        prev, curr = curr, [0] * (m + 1)

    return prev[m]
```

### Рекомендации:
1. Начните с рекурсивного решения, затем добавьте мемоизацию
2. Нарисуйте таблицу dp и проследите зависимости
3. Проверьте базовые случаи и граничные условия
4. Оптимизируйте память, если нужны только последние строки/столбцы
5. Для восстановления решения храните дополнительную информацию

---
[prev: 03-huffman-coding](../07-greedy/03-huffman-coding.md) | [next: 02-memoization](./02-memoization.md)