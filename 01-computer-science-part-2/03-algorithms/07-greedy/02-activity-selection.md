# Задача выбора активностей (Activity Selection Problem)

## Определение

**Задача выбора активностей** — классическая задача жадных алгоритмов: выбрать максимальное количество непересекающихся активностей из заданного набора, где каждая активность имеет время начала и окончания.

## Зачем нужен

### Области применения:
- **Планирование встреч** — максимизация использования переговорной
- **Расписание занятий** — составление учебного плана
- **Телевизионное вещание** — планирование программ
- **Производство** — распределение станков по заказам
- **Облачные вычисления** — планирование задач на серверах

### Вариации задачи:
- Максимизация **количества** активностей
- Максимизация **суммарного времени**
- Максимизация **суммарной ценности** (взвешенная версия)

## Как работает

### Жадная стратегия

```
Ключевая идея: выбираем активность, которая
ЗАКАНЧИВАЕТСЯ РАНЬШЕ ВСЕХ.

Почему это работает?
- Освобождаем время для большего количества активностей
- Ранний конец → больше возможностей для следующих
```

### Визуализация

```
Активности (отсортированы по времени окончания):

Время:  0   1   2   3   4   5   6   7   8   9   10  11  12
        │   │   │   │   │   │   │   │   │   │   │   │   │
A1:     ├───────┤                                           (0-3)
A2:         ├───┤                                           (1-4)  X
A3:             ├───────┤                                   (3-7)
A4:                 ├───────────┤                           (4-9)  X
A5:                         ├───────┤                       (6-10) X
A6:                                 ├───────┤               (8-11)
A7:                                     ├───────┤           (9-12) X

Жадный выбор:
1. A1 (0-3) — заканчивается раньше всех ✓
2. A3 (3-7) — не пересекается с A1, заканчивается раньше ✓
3. A6 (8-11) — не пересекается с A3, заканчивается раньше ✓

Выбрано: [A1, A3, A6] — 3 активности (оптимально!)
```

### Почему сортировка по концу, а не по началу?

```
Контрпример для сортировки по началу:

A1: ├───────────────────────────────────────────┤  (0-12)
A2:     ├───┤                                      (1-3)
A3:             ├───┤                              (4-6)
A4:                     ├───┤                      (7-9)

По началу: выбираем A1 → только 1 активность
По концу:  выбираем A2, A3, A4 → 3 активности ✓
```

## Псевдокод

```
function activitySelection(activities):
    # Сортируем по времени окончания
    sort activities by finish_time

    selected = [activities[0]]
    last_finish = activities[0].finish

    for i from 1 to n-1:
        if activities[i].start >= last_finish:
            selected.append(activities[i])
            last_finish = activities[i].finish

    return selected
```

### Реализация на Python

```python
def activity_selection(activities):
    """
    Выбор максимального количества непересекающихся активностей.

    Args:
        activities: список кортежей (start, finish)

    Returns:
        список выбранных активностей
    """
    if not activities:
        return []

    # Сортируем по времени окончания
    sorted_activities = sorted(activities, key=lambda x: x[1])

    selected = [sorted_activities[0]]
    last_finish = sorted_activities[0][1]

    for start, finish in sorted_activities[1:]:
        if start >= last_finish:
            selected.append((start, finish))
            last_finish = finish

    return selected


# Пример
activities = [(1, 4), (3, 5), (0, 6), (5, 7), (3, 9), (5, 9),
              (6, 10), (8, 11), (8, 12), (2, 14), (12, 16)]

result = activity_selection(activities)
print(f"Выбрано {len(result)} активностей: {result}")
# Выбрано 4 активностей: [(1, 4), (5, 7), (8, 11), (12, 16)]
```

### С индексами активностей

```python
def activity_selection_indexed(activities):
    """Возвращает индексы выбранных активностей."""
    if not activities:
        return []

    n = len(activities)
    # Создаём список с индексами
    indexed = [(start, finish, i) for i, (start, finish) in enumerate(activities)]

    # Сортируем по времени окончания
    indexed.sort(key=lambda x: x[1])

    selected_indices = [indexed[0][2]]
    last_finish = indexed[0][1]

    for start, finish, idx in indexed[1:]:
        if start >= last_finish:
            selected_indices.append(idx)
            last_finish = finish

    return selected_indices
```

### Рекурсивная версия

```python
def activity_selection_recursive(activities, k=0, last_finish=0):
    """Рекурсивный жадный выбор."""
    # Находим первую подходящую активность
    m = k
    while m < len(activities) and activities[m][0] < last_finish:
        m += 1

    if m < len(activities):
        return [(activities[m])] + activity_selection_recursive(
            activities, m + 1, activities[m][1]
        )

    return []

# Важно: активности должны быть отсортированы!
activities = sorted([(1, 4), (3, 5), (0, 6), (5, 7)], key=lambda x: x[1])
print(activity_selection_recursive(activities))
```

### Взвешенная версия (Weighted Activity Selection)

```python
def weighted_activity_selection(activities):
    """
    Максимизация суммарной ценности активностей.
    activities = [(start, finish, weight), ...]

    Эта задача требует DP, не чисто жадного подхода!
    """
    if not activities:
        return 0, []

    n = len(activities)
    # Сортируем по времени окончания
    activities = sorted(activities, key=lambda x: x[1])

    # dp[i] = максимальная ценность, используя первые i активностей
    dp = [0] * (n + 1)
    parent = [-1] * (n + 1)

    def find_last_compatible(i):
        """Находит последнюю активность, совместимую с i."""
        for j in range(i - 1, -1, -1):
            if activities[j][1] <= activities[i][0]:
                return j
        return -1

    for i in range(n):
        # Не включаем i-ю активность
        exclude = dp[i]

        # Включаем i-ю активность
        last = find_last_compatible(i)
        include = activities[i][2] + (dp[last + 1] if last >= 0 else 0)

        if include > exclude:
            dp[i + 1] = include
            parent[i + 1] = i
        else:
            dp[i + 1] = exclude
            parent[i + 1] = parent[i]

    # Восстанавливаем решение
    selected = []
    i = n
    while i > 0:
        if parent[i] >= 0 and (not selected or activities[parent[i]][1] <= selected[-1][0]):
            selected.append(activities[parent[i]])
        i -= 1

    return dp[n], selected[::-1]


# Пример
activities = [(1, 3, 5), (2, 5, 6), (4, 6, 5), (6, 7, 4), (5, 8, 11), (7, 9, 2)]
value, selected = weighted_activity_selection(activities)
print(f"Максимальная ценность: {value}")
```

## Анализ сложности

| Операция | Сложность |
|----------|-----------|
| Сортировка | O(n log n) |
| Выбор активностей | O(n) |
| **Общая** | **O(n log n)** |

Пространство: O(1) дополнительно (или O(n) для хранения результата)

## Типичные ошибки и Edge Cases

### 1. Неправильный критерий сортировки

```python
# ОШИБКА: сортировка по началу
activities.sort(key=lambda x: x[0])  # Неправильно!

# ОШИБКА: сортировка по длительности
activities.sort(key=lambda x: x[1] - x[0])  # Тоже неправильно!

# ПРАВИЛЬНО: сортировка по концу
activities.sort(key=lambda x: x[1])
```

### 2. Неправильное условие совместимости

```python
# ОШИБКА: строгое неравенство
if start > last_finish:  # Пропустим активности, начинающиеся сразу после

# ПРАВИЛЬНО: нестрогое неравенство (или строгое, в зависимости от задачи)
if start >= last_finish:  # Можно начать в момент окончания предыдущей
```

### 3. Пустой список активностей

```python
def activity_selection_safe(activities):
    if not activities:
        return []
    # ...
```

### 4. Одинаковые времена

```python
# Активности могут иметь одинаковое время окончания
activities = [(1, 3), (2, 3), (0, 3)]

# После сортировки порядок стабилен — зависит от реализации sort
# Лучше добавить вторичный критерий:
activities.sort(key=lambda x: (x[1], x[0]))  # По концу, затем по началу
```

### 5. Интервалы переговорных комнат

```python
def min_meeting_rooms(meetings):
    """
    Связанная задача: минимум переговорных для всех встреч.
    """
    if not meetings:
        return 0

    starts = sorted([m[0] for m in meetings])
    ends = sorted([m[1] for m in meetings])

    rooms = 0
    max_rooms = 0
    s = e = 0

    while s < len(meetings):
        if starts[s] < ends[e]:
            rooms += 1
            max_rooms = max(max_rooms, rooms)
            s += 1
        else:
            rooms -= 1
            e += 1

    return max_rooms

meetings = [(0, 30), (5, 10), (15, 20)]
print(min_meeting_rooms(meetings))  # 2
```

### 6. Визуализация расписания

```python
def print_schedule(activities, selected):
    """Визуализация выбранных активностей."""
    max_time = max(a[1] for a in activities)
    selected_set = set(selected)

    print("Timeline:")
    print("0" + "".join([str(i % 10) for i in range(1, max_time + 1)]))

    for i, (start, finish) in enumerate(activities):
        line = " " * start + "─" * (finish - start)
        status = "✓" if (start, finish) in selected_set else " "
        print(f"{line.ljust(max_time)} {status} ({start}-{finish})")

activities = [(1, 4), (3, 5), (0, 6), (5, 7), (6, 10), (8, 11)]
selected = activity_selection(activities)
print_schedule(activities, selected)
```

### Рекомендации:
1. Всегда сортируйте по времени **окончания**
2. Жадный выбор: берём активность с самым ранним окончанием
3. Для взвешенной версии используйте DP
4. Проверяйте граничные случаи: пустой список, одна активность
5. Условие совместимости: start >= last_finish (или > в зависимости от задачи)
