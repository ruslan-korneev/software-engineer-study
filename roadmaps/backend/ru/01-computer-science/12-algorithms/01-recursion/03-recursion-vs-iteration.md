# Рекурсия vs Итерация

[prev: 02-tail-recursion](./02-tail-recursion.md) | [next: 01-linear-search](../02-searching/01-linear-search.md)
---

## Определение

**Рекурсия** — метод решения задачи путём разбиения её на меньшие подзадачи того же типа. Функция вызывает сама себя.

**Итерация** — метод решения задачи путём повторения набора инструкций с использованием циклов (for, while).

Любой рекурсивный алгоритм можно преобразовать в итеративный и наоборот, но удобство и эффективность могут отличаться.

## Зачем понимать различия

### Когда использовать рекурсию:
- Древовидные структуры (обход деревьев, графов)
- Задачи "разделяй и властвуй" (merge sort, quick sort)
- Математические определения (факториал, Фибоначчи)
- Задачи с возвратом (backtracking)
- Когда код становится значительно проще

### Когда использовать итерацию:
- Простые линейные обходы
- Критичные к производительности участки
- Глубокие вычисления (избежание Stack Overflow)
- Когда важна экономия памяти

## Как работают

### Визуальное сравнение

```
РЕКУРСИЯ (Факториал 5):            ИТЕРАЦИЯ (Факториал 5):

Стек вызовов:                      Переменные:
┌─────────────────┐                result = 1
│ factorial(5)    │
│ ┌─────────────┐ │                Цикл:
│ │ factorial(4)│ │                i=1: result = 1*1 = 1
│ │ ┌─────────┐ │ │                i=2: result = 1*2 = 2
│ │ │ fact(3) │ │ │                i=3: result = 2*3 = 6
│ │ │ ┌─────┐ │ │ │                i=4: result = 6*4 = 24
│ │ │ │f(2) │ │ │ │                i=5: result = 24*5 = 120
│ │ │ │┌───┐│ │ │ │
│ │ │ ││f1 ││ │ │ │                Память: O(1)
│ │ │ │└───┘│ │ │ │                (только переменные)
│ │ │ └─────┘ │ │ │
│ │ └─────────┘ │ │
│ └─────────────┘ │
└─────────────────┘

Память: O(n)
(стек вызовов)
```

### Выполнение во времени

```
Время →

РЕКУРСИЯ:
call(5)──┬──call(4)──┬──call(3)──┬──call(2)──┬──call(1)
         │          │          │          │     ↓
         │          │          │          │  return 1
         │          │          │     ←────┴──2*1=2
         │          │     ←────┴──3*2=6
         │     ←────┴──4*6=24
    ←────┴──5*24=120

ИТЕРАЦИЯ:
start → i=1 → i=2 → i=3 → i=4 → i=5 → end
        r=1   r=2   r=6   r=24  r=120
```

## Сравнение кода

### Факториал

```python
# РЕКУРСИЯ
def factorial_recursive(n):
    if n <= 1:
        return 1
    return n * factorial_recursive(n - 1)

# ИТЕРАЦИЯ
def factorial_iterative(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

### Числа Фибоначчи

```python
# РЕКУРСИЯ (наивная) - O(2^n)
def fib_recursive(n):
    if n <= 1:
        return n
    return fib_recursive(n - 1) + fib_recursive(n - 2)

# ИТЕРАЦИЯ - O(n)
def fib_iterative(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b
```

### Обход дерева

```python
class TreeNode:
    def __init__(self, val, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

# РЕКУРСИЯ — естественно и просто
def inorder_recursive(node):
    if node is None:
        return []
    return (inorder_recursive(node.left) +
            [node.val] +
            inorder_recursive(node.right))

# ИТЕРАЦИЯ — сложнее, нужен явный стек
def inorder_iterative(root):
    result = []
    stack = []
    current = root

    while current or stack:
        while current:
            stack.append(current)
            current = current.left
        current = stack.pop()
        result.append(current.val)
        current = current.right

    return result
```

### Обратный порядок строки

```python
# РЕКУРСИЯ
def reverse_recursive(s):
    if len(s) <= 1:
        return s
    return reverse_recursive(s[1:]) + s[0]

# ИТЕРАЦИЯ
def reverse_iterative(s):
    result = ""
    for char in s:
        result = char + result
    return result

# Или проще (Python-way)
def reverse_pythonic(s):
    return s[::-1]
```

## Анализ сложности

| Аспект | Рекурсия | Итерация |
|--------|----------|----------|
| **Временная сложность** | Обычно O(n), может быть O(2^n) | Обычно O(n) |
| **Пространственная сложность** | O(n) — стек вызовов | O(1) — только переменные |
| **Накладные расходы** | Выше (создание стек-фреймов) | Ниже |
| **Читаемость** | Часто лучше для сложных структур | Лучше для простых задач |
| **Риск Stack Overflow** | Да, при глубокой рекурсии | Нет |

### Детальное сравнение для Фибоначчи

| Версия | Время | Память | Примечание |
|--------|-------|--------|------------|
| Рекурсия наивная | O(2^n) | O(n) | Много повторных вычислений |
| Рекурсия + мемоизация | O(n) | O(n) | Оптимально для рекурсии |
| Хвостовая рекурсия | O(n) | O(n)* | *O(1) с TCO |
| Итерация | O(n) | O(1) | Наиболее эффективно |

## Пример преобразования

### Задача: Сумма чисел от 1 до n

```python
# Шаг 1: Рекурсивное решение
def sum_recursive(n):
    if n <= 0:
        return 0
    return n + sum_recursive(n - 1)

# Шаг 2: Анализ паттерна
# sum(5) = 5 + sum(4) = 5 + 4 + sum(3) = ... = 5 + 4 + 3 + 2 + 1

# Шаг 3: Преобразование в итерацию
def sum_iterative(n):
    result = 0
    for i in range(1, n + 1):
        result += i
    return result

# Или математически: O(1)
def sum_formula(n):
    return n * (n + 1) // 2
```

### Общий алгоритм преобразования рекурсии в итерацию:

1. **Идентифицировать базовый случай** → условие выхода из цикла
2. **Идентифицировать параметры** → переменные цикла
3. **Идентифицировать рекурсивный вызов** → тело цикла
4. **Если нужен стек вызовов** → использовать явный стек

### Пример с явным стеком (обход дерева):

```python
# Рекурсия
def preorder_recursive(node):
    if node is None:
        return []
    return [node.val] + preorder_recursive(node.left) + preorder_recursive(node.right)

# Итерация с явным стеком
def preorder_iterative(root):
    if root is None:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)

        # Добавляем справа налево, чтобы левый обрабатывался первым
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result
```

## Типичные ошибки и Edge Cases

### 1. Игнорирование глубины рекурсии

```python
# ОШИБКА: Stack Overflow для больших n
def factorial_recursive(n):
    if n <= 1:
        return 1
    return n * factorial_recursive(n - 1)

factorial_recursive(10000)  # RecursionError!

# РЕШЕНИЕ: использовать итерацию
def factorial_iterative(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result

factorial_iterative(10000)  # Работает!
```

### 2. Неэффективная рекурсия без мемоизации

```python
import time

# ПЛОХО: экспоненциальное время
def fib_bad(n):
    if n <= 1:
        return n
    return fib_bad(n-1) + fib_bad(n-2)

start = time.time()
fib_bad(35)  # ~5 секунд
print(f"Рекурсия: {time.time() - start:.2f}s")

# ХОРОШО: итерация
def fib_good(n):
    if n <= 1:
        return n
    a, b = 0, 1
    for _ in range(2, n + 1):
        a, b = b, a + b
    return b

start = time.time()
fib_good(35)  # мгновенно
print(f"Итерация: {time.time() - start:.6f}s")
```

### 3. Усложнение простых задач рекурсией

```python
# ИЗЛИШНЕ СЛОЖНО
def sum_array_recursive(arr, index=0):
    if index >= len(arr):
        return 0
    return arr[index] + sum_array_recursive(arr, index + 1)

# ПРОСТО И ПОНЯТНО
def sum_array_iterative(arr):
    return sum(arr)  # или цикл
```

### 4. Неправильное преобразование в итерацию

```python
# Рекурсия с несколькими вызовами
def count_paths(m, n):
    if m == 1 or n == 1:
        return 1
    return count_paths(m-1, n) + count_paths(m, n-1)

# НЕПРАВИЛЬНОЕ преобразование (всё ещё рекурсия!)
def count_paths_bad(m, n):
    stack = [(m, n)]
    count = 0
    while stack:
        m, n = stack.pop()
        if m == 1 or n == 1:
            count += 1
        else:
            stack.append((m-1, n))
            stack.append((m, n-1))
    return count

# ПРАВИЛЬНОЕ преобразование (динамическое программирование)
def count_paths_good(m, n):
    dp = [[1] * n for _ in range(m)]
    for i in range(1, m):
        for j in range(1, n):
            dp[i][j] = dp[i-1][j] + dp[i][j-1]
    return dp[m-1][n-1]
```

### 5. Когда рекурсия действительно лучше

```python
# Обход файловой системы — рекурсия естественна
import os

def list_files_recursive(path):
    files = []
    for item in os.listdir(path):
        full_path = os.path.join(path, item)
        if os.path.isfile(full_path):
            files.append(full_path)
        elif os.path.isdir(full_path):
            files.extend(list_files_recursive(full_path))
    return files

# Итеративная версия — сложнее
def list_files_iterative(path):
    files = []
    stack = [path]

    while stack:
        current = stack.pop()
        for item in os.listdir(current):
            full_path = os.path.join(current, item)
            if os.path.isfile(full_path):
                files.append(full_path)
            elif os.path.isdir(full_path):
                stack.append(full_path)

    return files
```

## Рекомендации по выбору

| Ситуация | Рекомендация |
|----------|--------------|
| Простой линейный обход | Итерация |
| Обход дерева/графа | Рекурсия (или итерация со стеком) |
| Глубокие вычисления (n > 1000) | Итерация |
| Divide and Conquer | Рекурсия |
| Критичная производительность | Итерация |
| Читаемость важнее скорости | Рекурсия |
| Backtracking | Рекурсия |

### Практические советы:
1. Начните с рекурсии, если задача имеет рекурсивную природу
2. Профилируйте код — оптимизируйте только при необходимости
3. Для глубокой рекурсии в Python используйте итерацию
4. Мемоизация может сделать рекурсию столь же эффективной, как итерация
5. Не усложняйте простые задачи ради "элегантности"

---
[prev: 02-tail-recursion](./02-tail-recursion.md) | [next: 01-linear-search](../02-searching/01-linear-search.md)