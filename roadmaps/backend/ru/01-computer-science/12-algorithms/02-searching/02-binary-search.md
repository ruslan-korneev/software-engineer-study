# Бинарный поиск (Binary Search)

[prev: 01-linear-search](./01-linear-search.md) | [next: 01-bubble-sort](../03-sorting/01-bubble-sort.md)
---

## Определение

**Бинарный поиск** — это эффективный алгоритм поиска элемента в **отсортированном** массиве. Алгоритм многократно делит область поиска пополам, сравнивая искомый элемент со средним элементом. Если искомый элемент меньше среднего, поиск продолжается в левой половине, иначе — в правой.

## Зачем нужен

### Области применения:
- **Поиск в отсортированных массивах** — основное применение
- **Поиск границ** — первое/последнее вхождение элемента
- **Поиск позиции вставки** — где вставить элемент, сохранив сортировку
- **Оптимизационные задачи** — поиск минимума/максимума по условию
- **Базы данных** — индексы на основе B-деревьев используют принцип бинарного поиска

### Преимущества:
- Время O(log n) — значительно быстрее линейного поиска
- Эффективен для больших массивов
- Не требует дополнительной памяти (итеративная версия)

### Ограничения:
- Требует отсортированных данных
- Требует произвольного доступа (не подходит для связанных списков)

## Как работает

### Визуализация процесса

```
Поиск числа 23 в отсортированном массиве:
[2, 5, 8, 12, 16, 23, 38, 56, 72, 91]

Шаг 1: left=0, right=9, mid=4
[2, 5, 8, 12, 16, 23, 38, 56, 72, 91]
 L           M                    R
             ↓
            16 < 23, ищем справа
            left = mid + 1 = 5

Шаг 2: left=5, right=9, mid=7
[2, 5, 8, 12, 16, 23, 38, 56, 72, 91]
                  L       M       R
                          ↓
                         56 > 23, ищем слева
                         right = mid - 1 = 6

Шаг 3: left=5, right=6, mid=5
[2, 5, 8, 12, 16, 23, 38, 56, 72, 91]
                  L   R
                  M
                  ↓
                 23 = 23, НАЙДЕНО! Индекс: 5
```

### Наглядная схема деления

```
Начало: весь массив
[░░░░░░░░░░░░░░░░░░░░░░░░░░░░░░]

После шага 1: правая половина
              [░░░░░░░░░░░░░░░]

После шага 2: левая четверть
              [░░░░░░░]

После шага 3: найден элемент
              [░]

Количество шагов = log₂(n)
Для n=1000: ~10 шагов
Для n=1000000: ~20 шагов
```

## Псевдокод

```
function binarySearch(array, target):
    left = 0
    right = length(array) - 1

    while left <= right:
        mid = left + (right - left) / 2  # Защита от переполнения

        if array[mid] == target:
            return mid
        else if array[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1  # Не найден
```

### Итеративная реализация на Python

```python
def binary_search(arr, target):
    """
    Бинарный поиск элемента в отсортированном массиве.

    Args:
        arr: отсортированный список
        target: искомый элемент

    Returns:
        индекс найденного элемента или -1
    """
    left, right = 0, len(arr) - 1

    while left <= right:
        mid = left + (right - left) // 2  # Избегаем переполнения

        if arr[mid] == target:
            return mid
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return -1
```

### Рекурсивная реализация

```python
def binary_search_recursive(arr, target, left=None, right=None):
    """Рекурсивный бинарный поиск."""
    if left is None:
        left, right = 0, len(arr) - 1

    if left > right:
        return -1

    mid = left + (right - left) // 2

    if arr[mid] == target:
        return mid
    elif arr[mid] < target:
        return binary_search_recursive(arr, target, mid + 1, right)
    else:
        return binary_search_recursive(arr, target, left, mid - 1)
```

### Поиск первого вхождения (leftmost)

```python
def binary_search_first(arr, target):
    """Находит индекс первого вхождения элемента."""
    left, right = 0, len(arr) - 1
    result = -1

    while left <= right:
        mid = left + (right - left) // 2

        if arr[mid] == target:
            result = mid
            right = mid - 1  # Продолжаем искать слева
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return result
```

### Поиск последнего вхождения (rightmost)

```python
def binary_search_last(arr, target):
    """Находит индекс последнего вхождения элемента."""
    left, right = 0, len(arr) - 1
    result = -1

    while left <= right:
        mid = left + (right - left) // 2

        if arr[mid] == target:
            result = mid
            left = mid + 1  # Продолжаем искать справа
        elif arr[mid] < target:
            left = mid + 1
        else:
            right = mid - 1

    return result
```

### Поиск позиции вставки (bisect_left)

```python
def bisect_left(arr, target):
    """Находит позицию для вставки, сохраняющую сортировку."""
    left, right = 0, len(arr)

    while left < right:
        mid = left + (right - left) // 2

        if arr[mid] < target:
            left = mid + 1
        else:
            right = mid

    return left
```

## Анализ сложности

| Случай | Временная сложность | Описание |
|--------|---------------------|----------|
| **Лучший** | O(1) | Элемент в середине |
| **Средний** | O(log n) | Типичный случай |
| **Худший** | O(log n) | Элемент на краю или отсутствует |

| Ресурс | Итеративный | Рекурсивный |
|--------|-------------|-------------|
| **Время** | O(log n) | O(log n) |
| **Память** | O(1) | O(log n) — стек вызовов |

### Сравнение с линейным поиском

| n | Линейный O(n) | Бинарный O(log n) |
|---|---------------|-------------------|
| 10 | 10 | 4 |
| 100 | 100 | 7 |
| 1,000 | 1,000 | 10 |
| 1,000,000 | 1,000,000 | 20 |
| 1,000,000,000 | 1,000,000,000 | 30 |

## Пример с пошаговым разбором

### Задача: Найти число 7 в массиве [1, 3, 5, 7, 9, 11, 13]

```python
arr = [1, 3, 5, 7, 9, 11, 13]
target = 7

# Итерация 1:
# left=0, right=6, mid=3
# arr[3]=7 == 7, НАЙДЕНО!

# Результат: индекс 3
```

### Задача: Найти число 6 в массиве [1, 3, 5, 7, 9, 11, 13]

```python
arr = [1, 3, 5, 7, 9, 11, 13]
target = 6

# Итерация 1:
# left=0, right=6, mid=3
# arr[3]=7 > 6, ищем слева
# right = 2

# Итерация 2:
# left=0, right=2, mid=1
# arr[1]=3 < 6, ищем справа
# left = 2

# Итерация 3:
# left=2, right=2, mid=2
# arr[2]=5 < 6, ищем справа
# left = 3

# left > right, элемент не найден
# Результат: -1
```

### Подсчёт вхождений

```python
def count_occurrences(arr, target):
    """Считает количество вхождений элемента."""
    first = binary_search_first(arr, target)
    if first == -1:
        return 0
    last = binary_search_last(arr, target)
    return last - first + 1

arr = [1, 2, 2, 2, 2, 3, 4, 5]
print(count_occurrences(arr, 2))  # 4
```

## Типичные ошибки и Edge Cases

### 1. Переполнение при вычислении mid

```python
# ОШИБКА: может переполниться в некоторых языках
mid = (left + right) // 2

# ПРАВИЛЬНО: безопасное вычисление
mid = left + (right - left) // 2
```

### 2. Неправильные границы цикла

```python
# ОШИБКА: бесконечный цикл или пропуск элементов
while left < right:  # Не включает случай left == right
    mid = (left + right) // 2
    if arr[mid] < target:
        left = mid  # ОШИБКА: должно быть mid + 1

# ПРАВИЛЬНО
while left <= right:
    mid = left + (right - left) // 2
    if arr[mid] < target:
        left = mid + 1
    else:
        right = mid - 1
```

### 3. Пустой массив

```python
def binary_search_safe(arr, target):
    if not arr:
        return -1
    # ... остальной код
```

### 4. Массив из одного элемента

```python
arr = [42]
print(binary_search(arr, 42))  # 0
print(binary_search(arr, 99))  # -1
```

### 5. Использование на неотсортированном массиве

```python
# ОШИБКА: неопределённое поведение!
arr = [5, 2, 8, 1, 9]
binary_search(arr, 2)  # Может вернуть неверный результат!

# ПРАВИЛЬНО: сначала отсортировать
arr = sorted([5, 2, 8, 1, 9])  # [1, 2, 5, 8, 9]
binary_search(arr, 2)  # 1
```

### 6. Поиск в массиве с дубликатами

```python
arr = [1, 2, 2, 2, 2, 3]

# Стандартный бинарный поиск вернёт какой-то из индексов 2
# Для конкретного индекса используйте специальные версии:
first = binary_search_first(arr, 2)  # 1
last = binary_search_last(arr, 2)    # 4
```

### 7. Использование встроенного модуля bisect

```python
import bisect

arr = [1, 3, 5, 7, 9]

# Позиция для вставки слева
pos = bisect.bisect_left(arr, 5)  # 2

# Позиция для вставки справа
pos = bisect.bisect_right(arr, 5)  # 3

# Проверка наличия элемента
def contains(arr, target):
    i = bisect.bisect_left(arr, target)
    return i < len(arr) and arr[i] == target

print(contains(arr, 5))  # True
print(contains(arr, 6))  # False
```

### 8. Бинарный поиск по ответу

```python
def sqrt_binary_search(n, precision=1e-10):
    """Вычисление квадратного корня бинарным поиском."""
    if n < 0:
        raise ValueError("Отрицательное число")
    if n == 0:
        return 0

    left, right = 0, max(1, n)

    while right - left > precision:
        mid = (left + right) / 2
        if mid * mid < n:
            left = mid
        else:
            right = mid

    return (left + right) / 2

print(sqrt_binary_search(2))  # ~1.4142135623730951
```

### Рекомендации:
1. Всегда используйте `left + (right - left) // 2` для вычисления mid
2. Убедитесь, что массив отсортирован перед поиском
3. Используйте `while left <= right` для стандартного поиска
4. Для поиска границ модифицируйте условия обновления left/right
5. Рассмотрите использование `bisect` модуля в Python
6. Проверяйте edge cases: пустой массив, один элемент, дубликаты

---
[prev: 01-linear-search](./01-linear-search.md) | [next: 01-bubble-sort](../03-sorting/01-bubble-sort.md)