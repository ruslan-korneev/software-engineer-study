# Рекурсия (Recursion)

**Рекурсия** — когда функция вызывает сама себя для решения меньшей подзадачи.

```python
def countdown(n):
    if n <= 0:
        print("Пуск!")
    else:
        print(n)
        countdown(n - 1)

countdown(3)
# 3
# 2
# 1
# Пуск!
```

## Два обязательных компонента

1. **Базовый случай (base case)** — условие остановки
2. **Рекурсивный случай** — вызов себя с меньшей задачей

```python
def factorial(n):
    # Базовый случай — остановка
    if n <= 1:
        return 1

    # Рекурсивный случай — вызов себя
    return n * factorial(n - 1)

factorial(5)  # 5 * 4 * 3 * 2 * 1 = 120
```

**Без базового случая — бесконечная рекурсия!**

## Как работает (Call Stack)

Каждый вызов добавляет кадр в стек вызовов:

```
factorial(4)
├── 4 * factorial(3)
│   ├── 3 * factorial(2)
│   │   ├── 2 * factorial(1)
│   │   │   └── return 1  ← базовый случай
│   │   └── return 2 * 1 = 2
│   └── return 3 * 2 = 6
└── return 4 * 6 = 24
```

```python
# Визуализация стека
def factorial(n, depth=0):
    indent = "  " * depth
    print(f"{indent}factorial({n})")

    if n <= 1:
        print(f"{indent}return 1")
        return 1

    result = n * factorial(n - 1, depth + 1)
    print(f"{indent}return {result}")
    return result

factorial(4)
```

## Классические примеры

### Факториал — n!

```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

# factorial(5) = 5 * 4 * 3 * 2 * 1 = 120
```

### Числа Фибоначчи

```python
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

# 0, 1, 1, 2, 3, 5, 8, 13, 21, 34...
# fib(0) = 0
# fib(1) = 1
# fib(6) = 8
```

### Сумма списка

```python
def sum_list(lst):
    if not lst:  # пустой список
        return 0
    return lst[0] + sum_list(lst[1:])

sum_list([1, 2, 3, 4, 5])  # 15
```

### Степень числа

```python
def power(base, exp):
    if exp == 0:
        return 1
    return base * power(base, exp - 1)

power(2, 10)  # 1024
```

### Переворот строки

```python
def reverse(s):
    if len(s) <= 1:
        return s
    return reverse(s[1:]) + s[0]

reverse("hello")  # "olleh"
```

### Палиндром

```python
def is_palindrome(s):
    if len(s) <= 1:
        return True
    if s[0] != s[-1]:
        return False
    return is_palindrome(s[1:-1])

is_palindrome("radar")  # True
is_palindrome("hello")  # False
```

## Рекурсия vs Итерация

```python
# Рекурсивно
def factorial_rec(n):
    if n <= 1:
        return 1
    return n * factorial_rec(n - 1)

# Итеративно
def factorial_iter(n):
    result = 1
    for i in range(2, n + 1):
        result *= i
    return result
```

| Критерий | Рекурсия | Итерация |
|----------|----------|----------|
| Код | Часто проще и короче | Может быть сложнее |
| Память | Использует стек вызовов | Обычно меньше |
| Скорость | Накладные расходы на вызовы | Обычно быстрее |
| Применение | Деревья, вложенные структуры | Линейные задачи |

**Правило:** если легко написать итеративно — пиши итеративно. Рекурсия хороша для древовидных структур.

## Лимит рекурсии в Python

Python ограничивает глубину рекурсии (~1000 по умолчанию):

```python
import sys

sys.getrecursionlimit()   # 1000

# Можно увеличить (осторожно — может переполнить стек!)
sys.setrecursionlimit(5000)
```

```python
# RecursionError: maximum recursion depth exceeded
def infinite():
    return infinite()

infinite()  # Ошибка!
```

## Хвостовая рекурсия (Tail Recursion)

Когда рекурсивный вызов — последняя операция (ничего не нужно делать после возврата):

```python
# Обычная рекурсия — после вызова ещё умножение
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)  # нужно дождаться результата и умножить

# Хвостовая рекурсия — результат в аккумуляторе
def factorial_tail(n, acc=1):
    if n <= 1:
        return acc
    return factorial_tail(n - 1, n * acc)  # просто возвращаем
```

**Важно:** Python **НЕ** оптимизирует хвостовую рекурсию (в отличие от Scheme, Scala). Стек всё равно растёт.

## Мемоизация — ускорение рекурсии

### Проблема: экспоненциальная сложность

```python
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

# fib(40) — очень медленно!
# Вызывается ~2^40 раз (одни и те же значения считаются много раз)
```

```
fib(5)
├── fib(4)
│   ├── fib(3)
│   │   ├── fib(2)
│   │   │   ├── fib(1)
│   │   │   └── fib(0)
│   │   └── fib(1)        ← уже считали!
│   └── fib(2)            ← уже считали!
│       ├── fib(1)
│       └── fib(0)
└── fib(3)                ← уже считали!
    ...
```

### Решение: кэширование результатов

```python
from functools import lru_cache

@lru_cache(maxsize=None)
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

fib(100)  # мгновенно!
```

### Ручная мемоизация

```python
def fib_memo(n, cache={}):
    if n in cache:
        return cache[n]

    if n <= 1:
        return n

    result = fib_memo(n - 1, cache) + fib_memo(n - 2, cache)
    cache[n] = result
    return result
```

### Сравнение сложности

| Вариант | Сложность |
|---------|-----------|
| Без мемоизации | O(2^n) |
| С мемоизацией | O(n) |

## Практические примеры

### Обход дерева

```python
def traverse_preorder(node):
    if node is None:
        return
    print(node.value)
    traverse_preorder(node.left)
    traverse_preorder(node.right)

def tree_sum(node):
    if node is None:
        return 0
    return node.value + tree_sum(node.left) + tree_sum(node.right)

def tree_height(node):
    if node is None:
        return 0
    return 1 + max(tree_height(node.left), tree_height(node.right))
```

### Обход вложенного списка (flatten)

```python
def flatten(lst):
    result = []
    for item in lst:
        if isinstance(item, list):
            result.extend(flatten(item))
        else:
            result.append(item)
    return result

flatten([1, [2, [3, 4], 5], 6])  # [1, 2, 3, 4, 5, 6]
```

### Быстрая сортировка (QuickSort)

```python
def quicksort(lst):
    if len(lst) <= 1:
        return lst

    pivot = lst[0]
    less = [x for x in lst[1:] if x < pivot]
    greater = [x for x in lst[1:] if x >= pivot]

    return quicksort(less) + [pivot] + quicksort(greater)

quicksort([3, 6, 1, 8, 2, 4])  # [1, 2, 3, 4, 6, 8]
```

### Сортировка слиянием (MergeSort)

```python
def mergesort(lst):
    if len(lst) <= 1:
        return lst

    mid = len(lst) // 2
    left = mergesort(lst[:mid])
    right = mergesort(lst[mid:])

    return merge(left, right)

def merge(left, right):
    result = []
    i = j = 0

    while i < len(left) and j < len(right):
        if left[i] < right[j]:
            result.append(left[i])
            i += 1
        else:
            result.append(right[j])
            j += 1

    result.extend(left[i:])
    result.extend(right[j:])
    return result
```

### Двоичный поиск

```python
def binary_search(lst, target, low=0, high=None):
    if high is None:
        high = len(lst) - 1

    if low > high:
        return -1

    mid = (low + high) // 2

    if lst[mid] == target:
        return mid
    elif lst[mid] < target:
        return binary_search(lst, target, mid + 1, high)
    else:
        return binary_search(lst, target, low, mid - 1)
```

### Все перестановки

```python
def permutations(lst):
    if len(lst) <= 1:
        return [lst]

    result = []
    for i, elem in enumerate(lst):
        rest = lst[:i] + lst[i+1:]
        for perm in permutations(rest):
            result.append([elem] + perm)

    return result

permutations([1, 2, 3])
# [[1, 2, 3], [1, 3, 2], [2, 1, 3], [2, 3, 1], [3, 1, 2], [3, 2, 1]]
```

### Ханойские башни

```python
def hanoi(n, source, target, auxiliary):
    if n == 1:
        print(f"Переместить диск с {source} на {target}")
        return

    hanoi(n - 1, source, auxiliary, target)
    print(f"Переместить диск с {source} на {target}")
    hanoi(n - 1, auxiliary, target, source)

hanoi(3, 'A', 'C', 'B')
```

## Когда использовать рекурсию?

**Хорошо подходит:**
- Деревья и графы
- Вложенные структуры
- "Разделяй и властвуй" (quicksort, mergesort)
- Задачи с естественной рекурсивной структурой

**Лучше итерация:**
- Линейные задачи (сумма, факториал)
- Когда глубина может быть большой
- Когда важна производительность

## Советы

1. **Всегда определяй базовый случай первым**
2. **Убедись, что задача уменьшается** к базовому случаю
3. **Используй мемоизацию** при повторных вычислениях
4. **Следи за глубиной** — помни о лимите Python
5. **Если сложно понять — нарисуй дерево вызовов**
