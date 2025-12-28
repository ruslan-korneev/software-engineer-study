# Линейный поиск (Linear Search)

[prev: 03-recursion-vs-iteration](../01-recursion/03-recursion-vs-iteration.md) | [next: 02-binary-search](./02-binary-search.md)
---

## Определение

**Линейный поиск** (также известный как последовательный поиск) — это простейший алгоритм поиска, который проходит по всем элементам коллекции один за другим, сравнивая каждый элемент с искомым значением, пока не найдёт совпадение или не достигнет конца коллекции.

## Зачем нужен

### Области применения:
- **Неотсортированные данные** — единственный вариант для несортированных коллекций
- **Небольшие массивы** — простота важнее производительности (до ~100 элементов)
- **Связанные списки** — бинарный поиск неэффективен из-за отсутствия произвольного доступа
- **Однократный поиск** — нет смысла сортировать ради одного поиска
- **Поиск по сложным критериям** — поиск по нескольким полям или условиям

### Преимущества:
- Простота реализации
- Работает с неотсортированными данными
- Не требует дополнительной памяти
- Эффективен для маленьких коллекций

## Как работает

### Визуализация процесса

```
Поиск числа 7 в массиве [3, 8, 2, 7, 1, 9]

Шаг 1: Проверяем индекс 0
[3, 8, 2, 7, 1, 9]
 ↑
 3 ≠ 7, продолжаем

Шаг 2: Проверяем индекс 1
[3, 8, 2, 7, 1, 9]
    ↑
    8 ≠ 7, продолжаем

Шаг 3: Проверяем индекс 2
[3, 8, 2, 7, 1, 9]
       ↑
       2 ≠ 7, продолжаем

Шаг 4: Проверяем индекс 3
[3, 8, 2, 7, 1, 9]
          ↑
          7 = 7, НАЙДЕНО! Возвращаем индекс 3
```

### Худший случай: элемент в конце или отсутствует

```
Поиск числа 5 в массиве [3, 8, 2, 7, 1, 9]

[3, 8, 2, 7, 1, 9]
 ↑  ↑  ↑  ↑  ↑  ↑
 1  2  3  4  5  6  ← Все 6 сравнений

Элемент не найден, возвращаем -1
```

## Псевдокод

```
function linearSearch(array, target):
    for i from 0 to length(array) - 1:
        if array[i] == target:
            return i        # Найден: возвращаем индекс
    return -1               # Не найден
```

### Реализация на Python

```python
def linear_search(arr, target):
    """
    Линейный поиск элемента в массиве.

    Args:
        arr: список для поиска
        target: искомый элемент

    Returns:
        индекс найденного элемента или -1
    """
    for i in range(len(arr)):
        if arr[i] == target:
            return i
    return -1
```

### Рекурсивная версия

```python
def linear_search_recursive(arr, target, index=0):
    """Рекурсивный линейный поиск."""
    if index >= len(arr):
        return -1
    if arr[index] == target:
        return index
    return linear_search_recursive(arr, target, index + 1)
```

### Поиск с конца (оптимизация для часто запрашиваемых элементов в конце)

```python
def linear_search_reverse(arr, target):
    """Линейный поиск с конца массива."""
    for i in range(len(arr) - 1, -1, -1):
        if arr[i] == target:
            return i
    return -1
```

### Sentinel Linear Search (оптимизированная версия)

```python
def sentinel_linear_search(arr, target):
    """
    Линейный поиск с sentinel-элементом.
    Уменьшает количество сравнений в цикле.
    """
    n = len(arr)
    if n == 0:
        return -1

    # Сохраняем последний элемент
    last = arr[n - 1]

    # Ставим target в конец (sentinel)
    arr[n - 1] = target

    i = 0
    # Теперь не нужно проверять границу — target гарантированно будет найден
    while arr[i] != target:
        i += 1

    # Восстанавливаем последний элемент
    arr[n - 1] = last

    # Проверяем, был ли найден реальный элемент или sentinel
    if i < n - 1 or arr[n - 1] == target:
        return i
    return -1
```

## Анализ сложности

| Случай | Временная сложность | Описание |
|--------|---------------------|----------|
| **Лучший** | O(1) | Элемент на первой позиции |
| **Средний** | O(n/2) = O(n) | Элемент в середине |
| **Худший** | O(n) | Элемент в конце или отсутствует |

| Ресурс | Сложность |
|--------|-----------|
| **Время** | O(n) |
| **Память** | O(1) — итеративная версия |
| **Память** | O(n) — рекурсивная версия (стек вызовов) |

### Количество сравнений:
- Минимум: 1
- Максимум: n
- В среднем: (n + 1) / 2

## Пример с пошаговым разбором

### Задача: Найти индекс первого вхождения числа 42

```python
arr = [15, 23, 42, 8, 42, 99, 7]
target = 42

# Шаг 1: i=0, arr[0]=15, 15≠42, продолжаем
# Шаг 2: i=1, arr[1]=23, 23≠42, продолжаем
# Шаг 3: i=2, arr[2]=42, 42=42, НАЙДЕНО!

result = linear_search(arr, 42)
print(f"Индекс: {result}")  # Индекс: 2
```

### Поиск всех вхождений

```python
def linear_search_all(arr, target):
    """Находит все индексы вхождения элемента."""
    indices = []
    for i in range(len(arr)):
        if arr[i] == target:
            indices.append(i)
    return indices

arr = [15, 23, 42, 8, 42, 99, 7, 42]
result = linear_search_all(arr, 42)
print(f"Индексы: {result}")  # Индексы: [2, 4, 7]
```

### Поиск по условию

```python
def linear_search_predicate(arr, predicate):
    """Поиск первого элемента, удовлетворяющего условию."""
    for i, item in enumerate(arr):
        if predicate(item):
            return i
    return -1

# Найти первое чётное число
arr = [15, 23, 41, 8, 42, 99, 7]
result = linear_search_predicate(arr, lambda x: x % 2 == 0)
print(f"Первое чётное на индексе: {result}")  # Первое чётное на индексе: 3
```

## Типичные ошибки и Edge Cases

### 1. Пустой массив

```python
def linear_search_safe(arr, target):
    if not arr:  # Проверка на пустой массив
        return -1
    for i in range(len(arr)):
        if arr[i] == target:
            return i
    return -1

# Тест
print(linear_search_safe([], 5))  # -1
```

### 2. Массив из одного элемента

```python
arr = [42]
print(linear_search(arr, 42))  # 0
print(linear_search(arr, 99))  # -1
```

### 3. Дубликаты — какой индекс возвращать?

```python
# Первое вхождение
def find_first(arr, target):
    for i in range(len(arr)):
        if arr[i] == target:
            return i
    return -1

# Последнее вхождение
def find_last(arr, target):
    result = -1
    for i in range(len(arr)):
        if arr[i] == target:
            result = i
    return result

arr = [1, 2, 3, 2, 2, 4]
print(f"Первое 2: {find_first(arr, 2)}")  # 1
print(f"Последнее 2: {find_last(arr, 2)}")  # 4
```

### 4. Поиск None или специальных значений

```python
def linear_search_with_none(arr, target):
    for i in range(len(arr)):
        # Используем 'is' для None, '==' для остальных
        if target is None:
            if arr[i] is None:
                return i
        elif arr[i] == target:
            return i
    return -1

arr = [1, None, 3, None, 5]
print(linear_search_with_none(arr, None))  # 1
```

### 5. Поиск в строке

```python
def find_char(string, char):
    for i, c in enumerate(string):
        if c == char:
            return i
    return -1

# Или использовать встроенный метод
text = "Hello, World!"
print(text.find('o'))  # 4
print(text.find('z'))  # -1
```

### 6. Поиск объектов по атрибуту

```python
class User:
    def __init__(self, id, name):
        self.id = id
        self.name = name

def find_user_by_id(users, user_id):
    for i, user in enumerate(users):
        if user.id == user_id:
            return i
    return -1

users = [User(1, "Alice"), User(2, "Bob"), User(3, "Charlie")]
index = find_user_by_id(users, 2)
print(f"Bob на индексе: {index}")  # Bob на индексе: 1
```

### Рекомендации:
1. Используйте линейный поиск только для небольших или неотсортированных данных
2. Для больших отсортированных массивов используйте бинарный поиск
3. Рассмотрите использование хеш-таблиц для частого поиска
4. В Python часто можно использовать `in`, `index()`, `find()`
5. Обрабатывайте edge cases: пустой массив, None, дубликаты

---
[prev: 03-recursion-vs-iteration](../01-recursion/03-recursion-vs-iteration.md) | [next: 02-binary-search](./02-binary-search.md)