# Метод возврата (Backtracking)

## Определение

**Backtracking (метод возврата, поиск с возвратом)** — это алгоритмическая техника для решения задач путём пошагового построения решения. Если на каком-то шаге обнаруживается, что текущий путь не ведёт к решению, алгоритм "откатывается" (backtrack) к предыдущему шагу и пробует другой вариант.

Backtracking — это усовершенствованный перебор, который отсекает заведомо бесперспективные ветви поиска (pruning).

## Зачем нужен

### Области применения:
- **Комбинаторные задачи** — перестановки, сочетания, подмножества
- **Головоломки** — судоку, N ферзей, лабиринты
- **Задачи на графах** — гамильтонов путь, раскраска графа
- **Синтаксический анализ** — парсинг регулярных выражений
- **Криптография** — взлом паролей перебором

### Ключевые концепции:
- **Пространство решений** — все возможные решения
- **Частичное решение** — промежуточное состояние
- **Отсечение (pruning)** — исключение бесперспективных ветвей
- **Возврат (backtrack)** — откат к предыдущему состоянию

## Как работает

### Общая схема

```
                      []
                     /  \
                   /      \
                 /          \
              [1]            [2]
             /   \          /   \
           /       \      /       \
        [1,2]     [1,3]  [2,1]   [2,3]
         |          |      |       |
        ...        ...    ...     ...

Дерево решений: каждый узел — частичное решение,
каждое ребро — выбор/шаг.
```

### Визуализация backtracking

```
Задача: найти все перестановки [1, 2, 3]

                    []
                  / | \
                /   |   \
              /     |     \
           [1]     [2]     [3]
           / \     / \     / \
          /   \   /   \   /   \
       [1,2] [1,3] [2,1] [2,3] [3,1] [3,2]
         |     |     |     |     |     |
       [1,2,3][1,3,2][2,1,3][2,3,1][3,1,2][3,2,1]
          ✓     ✓     ✓     ✓     ✓     ✓

Все 6 перестановок найдены!
```

### Backtracking с отсечением

```
Задача: подмножества с суммой = 10, элементы [1, 5, 7, 3]

                      []  sum=0
                     /  \
                   /      \
            выбрали 1    не выбрали 1
                /            \
             [1]              []
            /    \           /  \
         [1,5]   [1]      [5]    []
        sum=6   sum=1    sum=5  sum=0
          |       |        |      |
       [1,5,7]  [1,7]   [5,7]   [7]
       sum=13   sum=8   sum=12  sum=7
          X       |       X      |
       (>10)   [1,7,3]         [7,3]
       PRUNE   sum=11          sum=10
                 X               ✓
              (>10)           FOUND!
```

## Псевдокод

```
function backtrack(state):
    if is_solution(state):
        process_solution(state)
        return

    for choice in get_choices(state):
        if is_valid(choice, state):
            make_choice(choice, state)     # Делаем выбор
            backtrack(state)               # Рекурсия
            undo_choice(choice, state)     # Откатываем выбор

# Или с возвратом результата:
function backtrack(state):
    if is_solution(state):
        return [state.copy()]

    results = []
    for choice in get_choices(state):
        if is_valid(choice, state):
            make_choice(choice, state)
            results.extend(backtrack(state))
            undo_choice(choice, state)

    return results
```

### Шаблон на Python

```python
def backtrack(result, current, choices, ...):
    """
    Универсальный шаблон backtracking.

    Args:
        result: список для накопления решений
        current: текущее частичное решение
        choices: доступные варианты выбора
        ...: дополнительные параметры
    """
    # Базовый случай: нашли решение
    if is_complete(current):
        result.append(current.copy())  # Важно: копируем!
        return

    for choice in choices:
        # Отсечение: пропускаем невалидные варианты
        if not is_valid(choice, current):
            continue

        # Делаем выбор
        current.append(choice)  # или другая модификация

        # Рекурсия
        backtrack(result, current, remaining_choices, ...)

        # Откатываем выбор (backtrack)
        current.pop()
```

### Пример: Генерация перестановок

```python
def permutations(nums):
    """Генерация всех перестановок."""
    result = []

    def backtrack(current, remaining):
        if not remaining:
            result.append(current.copy())
            return

        for i, num in enumerate(remaining):
            current.append(num)
            backtrack(current, remaining[:i] + remaining[i+1:])
            current.pop()

    backtrack([], nums)
    return result

print(permutations([1, 2, 3]))
# [[1, 2, 3], [1, 3, 2], [2, 1, 3], [2, 3, 1], [3, 1, 2], [3, 2, 1]]
```

### Пример: Генерация подмножеств

```python
def subsets(nums):
    """Генерация всех подмножеств."""
    result = []

    def backtrack(start, current):
        result.append(current.copy())

        for i in range(start, len(nums)):
            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()

    backtrack(0, [])
    return result

print(subsets([1, 2, 3]))
# [[], [1], [1, 2], [1, 2, 3], [1, 3], [2], [2, 3], [3]]
```

### Пример: Сочетания C(n, k)

```python
def combinations(n, k):
    """Генерация всех сочетаний из n по k."""
    result = []

    def backtrack(start, current):
        if len(current) == k:
            result.append(current.copy())
            return

        # Оптимизация: нужно ещё k - len(current) элементов
        # Останавливаемся, если недостаточно элементов
        for i in range(start, n - (k - len(current)) + 2):
            current.append(i)
            backtrack(i + 1, current)
            current.pop()

    backtrack(1, [])
    return result

print(combinations(4, 2))
# [[1, 2], [1, 3], [1, 4], [2, 3], [2, 4], [3, 4]]
```

## Анализ сложности

| Задача | Время | Пространство |
|--------|-------|--------------|
| Перестановки | O(n × n!) | O(n) |
| Подмножества | O(n × 2^n) | O(n) |
| Сочетания | O(k × C(n,k)) | O(k) |

Backtracking обычно имеет экспоненциальную сложность, но отсечение может значительно уменьшить фактическое время.

## Типичные ошибки и Edge Cases

### 1. Забыли скопировать решение

```python
# ОШИБКА: все решения ссылаются на один объект
def wrong_subsets(nums):
    result = []
    def backtrack(current):
        result.append(current)  # Ссылка!
        # ...
    return result
# Все элементы result будут одинаковые!

# ПРАВИЛЬНО
def correct_subsets(nums):
    result = []
    def backtrack(current):
        result.append(current.copy())  # Копия!
        # ...
```

### 2. Не откатили изменение

```python
# ОШИБКА: состояние "загрязняется"
def wrong_permutations(nums):
    def backtrack(current, remaining):
        current.append(remaining[0])
        backtrack(current, remaining[1:])
        # Забыли current.pop()!

# ПРАВИЛЬНО
def correct_permutations(nums):
    def backtrack(current, remaining):
        current.append(remaining[0])
        backtrack(current, remaining[1:])
        current.pop()  # Обязательно откатываем!
```

### 3. Дубликаты в результате

```python
def subsets_with_dup(nums):
    """Подмножества с дубликатами в исходном массиве."""
    nums.sort()  # Сортируем для обнаружения дубликатов
    result = []

    def backtrack(start, current):
        result.append(current.copy())

        for i in range(start, len(nums)):
            # Пропускаем дубликаты
            if i > start and nums[i] == nums[i-1]:
                continue

            current.append(nums[i])
            backtrack(i + 1, current)
            current.pop()

    backtrack(0, [])
    return result

print(subsets_with_dup([1, 2, 2]))
# [[], [1], [1, 2], [1, 2, 2], [2], [2, 2]]
```

### 4. Неправильное отсечение

```python
# ОШИБКА: отсекли валидные решения
def wrong_combinations_sum(candidates, target):
    def backtrack(start, current, total):
        if total > target:
            return  # Отсечение
        if total == target:
            result.append(current.copy())
            return

        for i in range(start, len(candidates)):
            if candidates[i] > target - total:
                continue  # ОШИБКА: может отсечь валидные!
            # ...

# ПРАВИЛЬНО: сортируем и отсекаем правильно
def correct_combinations_sum(candidates, target):
    candidates.sort()  # Сортируем!
    result = []

    def backtrack(start, current, total):
        if total == target:
            result.append(current.copy())
            return

        for i in range(start, len(candidates)):
            if total + candidates[i] > target:
                break  # Все последующие тоже больше

            current.append(candidates[i])
            backtrack(i, current, total + candidates[i])
            current.pop()

    backtrack(0, [], 0)
    return result
```

### 5. Пустой ввод

```python
def permutations_safe(nums):
    if not nums:
        return [[]]  # Одна пустая перестановка

    # ... остальной код
```

### Рекомендации:
1. Всегда копируйте решение перед добавлением в результат
2. Обязательно откатывайте изменения после рекурсии
3. Используйте отсечение для ускорения (сортировка часто помогает)
4. Проверяйте на дубликаты, если входные данные могут их содержать
5. Рисуйте дерево решений для понимания структуры задачи
