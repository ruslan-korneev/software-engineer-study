# Жадные алгоритмы (Greedy Algorithms)

## Определение

**Жадный алгоритм** — это алгоритмическая парадигма, которая на каждом шаге делает **локально оптимальный выбор** в надежде получить **глобально оптимальное решение**.

Жадный алгоритм никогда не пересматривает свои решения — однажды сделанный выбор остаётся окончательным.

## Зачем нужен

### Области применения:
- **Оптимизационные задачи** — минимизация/максимизация
- **Задачи планирования** — расписания, распределение ресурсов
- **Графовые алгоритмы** — Дейкстра, Прим, Краскал
- **Сжатие данных** — кодирование Хаффмана
- **Размен монет** — при определённых номиналах

### Преимущества:
- **Простота** реализации
- **Эффективность** — обычно O(n log n) или лучше
- **Интуитивность** — легко объяснить

### Недостатки:
- **Не всегда оптимален** — не гарантирует лучшее решение для всех задач
- **Требует доказательства** корректности

## Как работает

### Принцип работы

```
Жадный алгоритм:

1. Начать с пустого решения
2. На каждом шаге:
   a) Рассмотреть все доступные варианты
   b) Выбрать ЛУЧШИЙ вариант (по локальному критерию)
   c) Добавить к решению
   d) НЕ возвращаться и НЕ пересматривать
3. Повторять, пока не получим полное решение
```

### Сравнение с другими подходами

```
Задача: Найти оптимальный путь A → D

      B ─── 2 ─── D
     / \         /
    1   4       3
   /     \     /
  A ───── C ──┘

Жадный алгоритм:
  A → ? Выбираем ближайшего: B (1)
  B → ? Выбираем ближайшего: D (2)
  Путь: A → B → D, стоимость = 3 ✓

Динамическое программирование:
  Рассматриваем ВСЕ пути:
  A → B → D = 1 + 2 = 3
  A → B → C → D = 1 + 4 + 3 = 8
  A → C → D = 1 + 3 = 4 (если бы было прямое ребро)
  Выбираем лучший: A → B → D = 3 ✓

Перебор (Brute Force):
  Пробуем все возможные комбинации
  Выбираем лучшую
```

### Когда жадный работает

```
Жадный алгоритм оптимален, если задача имеет:

1. ЖАДНЫЙ ВЫБОР (Greedy Choice Property):
   Локально оптимальный выбор ведёт к глобальному оптимуму

2. ОПТИМАЛЬНАЯ ПОДСТРУКТУРА (Optimal Substructure):
   Оптимальное решение содержит оптимальные решения подзадач
```

## Псевдокод

```
function greedy_algorithm(problem):
    solution = empty
    candidates = all_possible_choices(problem)

    while solution is not complete:
        # Выбираем лучший локальный вариант
        best = select_best(candidates)

        if is_feasible(solution + best):
            solution = solution + best
            update_candidates(candidates, best)

    return solution
```

### Шаблон на Python

```python
def greedy_template(items, constraint):
    """
    Универсальный шаблон жадного алгоритма.

    Args:
        items: список элементов для выбора
        constraint: ограничение (ёмкость, время и т.д.)
    """
    # Сортируем по критерию жадности
    items.sort(key=lambda x: greedy_criterion(x), reverse=True)

    result = []
    current_value = 0

    for item in items:
        if can_add(item, result, constraint):
            result.append(item)
            current_value += item.value

    return result, current_value
```

## Классические примеры

### Размен монет

```python
def coin_change_greedy(amount, coins):
    """
    Жадный размен монет.
    ВНИМАНИЕ: работает не для всех систем монет!
    """
    coins.sort(reverse=True)  # Сначала большие
    result = []

    for coin in coins:
        while amount >= coin:
            result.append(coin)
            amount -= coin

    if amount == 0:
        return result
    return None  # Невозможно разменять

# Работает для [1, 5, 10, 25] (стандартная система)
print(coin_change_greedy(67, [25, 10, 5, 1]))
# [25, 25, 10, 5, 1, 1] — 6 монет, оптимально!

# НЕ работает для [1, 3, 4]
print(coin_change_greedy(6, [4, 3, 1]))
# [4, 1, 1] — 3 монеты
# Оптимально: [3, 3] — 2 монеты!
```

### Задача о рюкзаке (дробная версия)

```python
def fractional_knapsack(capacity, items):
    """
    Дробный рюкзак — можно брать части предметов.
    items = [(weight, value), ...]
    """
    # Сортируем по ценности на единицу веса
    items_with_ratio = [(w, v, v/w) for w, v in items]
    items_with_ratio.sort(key=lambda x: x[2], reverse=True)

    total_value = 0
    selected = []

    for weight, value, ratio in items_with_ratio:
        if capacity >= weight:
            # Берём целиком
            selected.append((weight, value, 1.0))
            total_value += value
            capacity -= weight
        else:
            # Берём часть
            fraction = capacity / weight
            selected.append((weight, value, fraction))
            total_value += value * fraction
            break

    return total_value, selected

# Пример
items = [(10, 60), (20, 100), (30, 120)]  # (вес, ценность)
value, selected = fractional_knapsack(50, items)
print(f"Максимальная ценность: {value}")  # 240
```

### Минимальное количество платформ

```python
def min_platforms(arrivals, departures):
    """
    Минимальное количество платформ для обслуживания поездов.
    """
    arrivals.sort()
    departures.sort()

    platforms_needed = 0
    max_platforms = 0

    i = j = 0
    n = len(arrivals)

    while i < n and j < n:
        if arrivals[i] <= departures[j]:
            platforms_needed += 1
            max_platforms = max(max_platforms, platforms_needed)
            i += 1
        else:
            platforms_needed -= 1
            j += 1

    return max_platforms

arrivals = [9.00, 9.40, 9.50, 11.00, 15.00, 18.00]
departures = [9.10, 12.00, 11.20, 11.30, 19.00, 20.00]
print(min_platforms(arrivals, departures))  # 3
```

### Интервальное покрытие (Interval Covering)

```python
def min_intervals_cover(intervals, target):
    """
    Минимальное количество интервалов для покрытия target.
    """
    intervals.sort()  # По началу
    result = []
    i = 0
    n = len(intervals)
    current_end = target[0]

    while current_end < target[1]:
        best_end = current_end

        # Находим интервал, который начинается до current_end
        # и заканчивается как можно дальше
        while i < n and intervals[i][0] <= current_end:
            best_end = max(best_end, intervals[i][1])
            i += 1

        if best_end == current_end:
            return None  # Невозможно покрыть

        result.append(intervals[i - 1])
        current_end = best_end

    return result
```

## Анализ сложности

| Задача | Время | Доминирующая операция |
|--------|-------|----------------------|
| Размен монет | O(n) | Итерация по номиналам |
| Дробный рюкзак | O(n log n) | Сортировка |
| Activity Selection | O(n log n) | Сортировка |
| Huffman Coding | O(n log n) | Очередь с приоритетом |

## Типичные ошибки и Edge Cases

### 1. Жадный не всегда оптимален!

```python
# Задача 0/1 рюкзака — жадный НЕ работает!
def knapsack_greedy_wrong(capacity, items):
    # Сортируем по ценности/вес
    items.sort(key=lambda x: x[1]/x[0], reverse=True)
    total = 0
    for w, v in items:
        if capacity >= w:
            total += v
            capacity -= w
    return total

# Пример, где жадный ошибается:
items = [(10, 60), (20, 100), (30, 120)]
# Жадный возьмёт (10, 60), затем (20, 100) = 160
# Оптимально: (20, 100) + (30, 120) = 220

# Для 0/1 рюкзака используйте динамическое программирование!
```

### 2. Неправильный критерий жадности

```python
# Размен монет — важен ПОРЯДОК
def coin_change_wrong(amount, coins):
    coins.sort()  # ОШИБКА: сначала маленькие!
    # ...

def coin_change_correct(amount, coins):
    coins.sort(reverse=True)  # Сначала большие!
    # ...
```

### 3. Пустой ввод

```python
def greedy_safe(items):
    if not items:
        return [], 0
    # ...
```

### 4. Доказательство корректности

```python
"""
Чтобы убедиться, что жадный алгоритм оптимален,
нужно доказать:

1. Greedy Choice Property:
   - Допустим, есть оптимальное решение без жадного выбора
   - Покажем, что замена на жадный выбор не ухудшит решение

2. Optimal Substructure:
   - После жадного выбора остаётся подзадача
   - Оптимальное решение подзадачи + жадный выбор = оптимум
"""
```

### 5. Жадный vs Динамическое программирование

```python
"""
Используйте жадный, если:
- Задача имеет свойство жадного выбора
- Нужна максимальная скорость
- Точность критична и жадный доказуемо оптимален

Используйте DP, если:
- Жадный не даёт оптимального решения
- Задача имеет перекрывающиеся подзадачи
- Нужно точное решение

Примеры:
- Дробный рюкзак → Жадный ✓
- 0/1 рюкзак → DP ✓
- Кратчайший путь (без отрицательных) → Жадный (Дейкстра) ✓
- Кратчайший путь (с отрицательными) → DP (Bellman-Ford) ✓
"""
```

### Рекомендации:
1. Всегда проверяйте, оптимален ли жадный для вашей задачи
2. Ищите контрпримеры перед реализацией
3. Сортировка — ключевая часть большинства жадных алгоритмов
4. Если жадный не работает — попробуйте DP
5. Жадный часто даёт хорошее приближение, даже если не оптимален
