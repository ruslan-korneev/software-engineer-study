# Мемоизация (Memoization)

## Определение

**Мемоизация** — это техника оптимизации, при которой результаты вычислений сохраняются в кэше для повторного использования. Это подход "сверху вниз" (top-down) к динамическому программированию.

Мемоизация = Рекурсия + Кэширование результатов

## Зачем нужен

### Преимущества:
- **Простота** — добавляем кэш к обычной рекурсии
- **Ленивые вычисления** — считаем только нужные подзадачи
- **Интуитивность** — легко понять и отладить

### Когда использовать:
- Рекурсивная структура задачи очевидна
- Не все подзадачи нужны для ответа
- Удобнее думать "сверху вниз"

## Как работает

### Визуализация

```
Числа Фибоначчи без мемоизации:

                    fib(5)
                   /      \
               fib(4)     fib(3)      ← fib(3) считается дважды
              /     \     /    \
          fib(3)  fib(2) fib(2) fib(1)  ← fib(2) считается трижды
          /   \
       fib(2) fib(1)

С мемоизацией:

fib(5) → ?
  fib(4) → ?
    fib(3) → ?
      fib(2) → ?
        fib(1) → 1  (базовый случай)
        fib(0) → 0  (базовый случай)
      fib(2) = 1, сохраняем в memo[2]
      fib(1) → 1  (базовый случай)
    fib(3) = 2, сохраняем в memo[3]
    fib(2) → memo[2] = 1  (из кэша!)
  fib(4) = 3, сохраняем в memo[4]
  fib(3) → memo[3] = 2  (из кэша!)
fib(5) = 5
```

### Схема работы

```
function memoized_function(args):
    ┌──────────────────────────────┐
    │  Аргументы в кэше?           │
    └──────────────┬───────────────┘
                   │
         ┌─────────┴─────────┐
         ▼                   ▼
       [Да]               [Нет]
         │                   │
         ▼                   ▼
    Вернуть            Вычислить
    из кэша            рекурсивно
                            │
                            ▼
                      Сохранить в кэш
                            │
                            ▼
                       Вернуть
```

## Реализация на Python

### Способ 1: Словарь

```python
def fib_memo(n, memo=None):
    """Фибоначчи с мемоизацией через словарь."""
    if memo is None:
        memo = {}

    if n in memo:
        return memo[n]

    if n <= 1:
        return n

    memo[n] = fib_memo(n - 1, memo) + fib_memo(n - 2, memo)
    return memo[n]


# Использование
print(fib_memo(100))  # Мгновенно!
# Без мемоизации: 2^100 операций (невозможно)
```

### Способ 2: Декоратор @lru_cache

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib_lru(n):
    """Фибоначчи с lru_cache — встроенная мемоизация."""
    if n <= 1:
        return n
    return fib_lru(n - 1) + fib_lru(n - 2)


# Использование
print(fib_lru(100))

# Информация о кэше
print(fib_lru.cache_info())
# CacheInfo(hits=98, misses=101, maxsize=None, currsize=101)

# Очистка кэша
fib_lru.cache_clear()
```

### Способ 3: Собственный декоратор

```python
def memoize(func):
    """Универсальный декоратор мемоизации."""
    cache = {}

    def wrapper(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]

    wrapper.cache = cache
    wrapper.clear_cache = lambda: cache.clear()

    return wrapper


@memoize
def fib_custom(n):
    if n <= 1:
        return n
    return fib_custom(n - 1) + fib_custom(n - 2)


print(fib_custom(100))
print(f"Размер кэша: {len(fib_custom.cache)}")
```

### Способ 4: Класс с кэшем

```python
class Fibonacci:
    def __init__(self):
        self.cache = {0: 0, 1: 1}

    def __call__(self, n):
        if n not in self.cache:
            self.cache[n] = self(n - 1) + self(n - 2)
        return self.cache[n]


fib = Fibonacci()
print(fib(100))
```

## Примеры применения

### Пути в сетке

```python
@lru_cache(maxsize=None)
def unique_paths(m, n):
    """Количество путей из (0,0) в (m-1, n-1), двигаясь только вправо/вниз."""
    if m == 1 or n == 1:
        return 1
    return unique_paths(m - 1, n) + unique_paths(m, n - 1)

print(unique_paths(3, 7))  # 28
```

### Размен монет

```python
def coin_change_memo(coins, amount):
    """Минимальное количество монет."""

    @lru_cache(maxsize=None)
    def dp(remaining):
        if remaining == 0:
            return 0
        if remaining < 0:
            return float('inf')

        min_coins = float('inf')
        for coin in coins:
            result = dp(remaining - coin)
            if result != float('inf'):
                min_coins = min(min_coins, result + 1)

        return min_coins

    result = dp(amount)
    return result if result != float('inf') else -1

print(coin_change_memo([1, 5, 10, 25], 67))  # 6
```

### Longest Common Subsequence (LCS)

```python
def lcs_memo(s1, s2):
    """Длина наибольшей общей подпоследовательности."""

    @lru_cache(maxsize=None)
    def dp(i, j):
        if i == 0 or j == 0:
            return 0

        if s1[i - 1] == s2[j - 1]:
            return dp(i - 1, j - 1) + 1

        return max(dp(i - 1, j), dp(i, j - 1))

    return dp(len(s1), len(s2))

print(lcs_memo("ABCDGH", "AEDFHR"))  # 3 ("ADH")
```

### Разбиение на слова (Word Break)

```python
def word_break(s, word_dict):
    """Можно ли разбить строку на слова из словаря?"""
    word_set = set(word_dict)

    @lru_cache(maxsize=None)
    def dp(start):
        if start == len(s):
            return True

        for end in range(start + 1, len(s) + 1):
            if s[start:end] in word_set and dp(end):
                return True

        return False

    return dp(0)

print(word_break("leetcode", ["leet", "code"]))  # True
```

## Анализ сложности

```
Без мемоизации:
- Фибоначчи: O(2^n)
- LCS: O(2^(n+m))

С мемоизацией:
- Фибоначчи: O(n) время, O(n) память
- LCS: O(n×m) время, O(n×m) память

Накладные расходы мемоизации:
- Хеширование аргументов
- Поиск в словаре: O(1) амортизированно
- Память под кэш
```

## Типичные ошибки и Edge Cases

### 1. Изменяемые аргументы

```python
# ОШИБКА: списки нельзя хешировать
@lru_cache(maxsize=None)
def wrong_func(arr):  # TypeError: unhashable type: 'list'
    pass

# РЕШЕНИЕ 1: преобразовать в кортеж
@lru_cache(maxsize=None)
def correct_func(arr_tuple):
    pass

# Вызов
correct_func(tuple([1, 2, 3]))

# РЕШЕНИЕ 2: использовать индексы
def better_approach(arr):
    @lru_cache(maxsize=None)
    def dp(i, j):  # Используем индексы, не сам массив
        pass
    return dp(0, len(arr) - 1)
```

### 2. Мутируемый аргумент по умолчанию

```python
# ОШИБКА: memo сохраняется между вызовами!
def fib_wrong(n, memo={}):
    # memo общий для всех вызовов

# ПРАВИЛЬНО
def fib_correct(n, memo=None):
    if memo is None:
        memo = {}
    # ...
```

### 3. Переполнение стека

```python
import sys
sys.setrecursionlimit(10000)  # Увеличить лимит

# Или использовать итеративный подход (табуляцию)
```

### 4. Память под кэш

```python
# Ограничение размера кэша
@lru_cache(maxsize=1000)  # Хранит последние 1000 результатов
def limited_cache(n):
    pass

# Очистка кэша после использования
result = some_function(args)
some_function.cache_clear()
```

### 5. Сравнение с табуляцией

```python
"""
МЕМОИЗАЦИЯ (Top-Down):
+ Простота реализации
+ Ленивые вычисления
+ Естественный рекурсивный стиль
- Накладные расходы на рекурсию
- Риск переполнения стека

ТАБУЛЯЦИЯ (Bottom-Up):
+ Нет рекурсии
+ Легче оптимизировать память
+ Может быть быстрее (меньше overhead)
- Нужно вычислить ВСЕ подзадачи
- Порядок вычисления не всегда очевиден
"""
```

### 6. Мемоизация с несколькими возвращаемыми значениями

```python
@lru_cache(maxsize=None)
def dp_with_path(n):
    """Возвращает (значение, путь)."""
    if n <= 1:
        return (n, [n])

    prev1_val, prev1_path = dp_with_path(n - 1)
    prev2_val, prev2_path = dp_with_path(n - 2)

    if prev1_val > prev2_val:
        return (prev1_val + n, prev1_path + [n])
    else:
        return (prev2_val + n, prev2_path + [n])
```

### Рекомендации:
1. Используйте `@lru_cache` для простых случаев
2. Преобразуйте списки в кортежи для хеширования
3. Следите за размером кэша при работе с большими данными
4. Очищайте кэш после использования, если память критична
5. При глубокой рекурсии рассмотрите табуляцию
