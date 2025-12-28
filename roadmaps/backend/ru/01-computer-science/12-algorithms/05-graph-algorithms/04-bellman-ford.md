# Алгоритм Беллмана-Форда (Bellman-Ford Algorithm)

[prev: 03-dijkstra](./03-dijkstra.md) | [next: 05-minimum-spanning-tree](./05-minimum-spanning-tree.md)
---

## Определение

**Алгоритм Беллмана-Форда** — это алгоритм поиска кратчайших путей от одной вершины до всех остальных во взвешенном графе. В отличие от алгоритма Дейкстры, **работает с отрицательными весами** рёбер и может **обнаруживать отрицательные циклы**.

## Зачем нужен

### Области применения:
- **Графы с отрицательными весами** — финансовые арбитражи
- **Обнаружение отрицательных циклов** — валютные обмены
- **Протоколы маршрутизации** — RIP (Routing Information Protocol)
- **Системы дифференциальных ограничений**
- **Проверка достижимости** — можно ли бесконечно уменьшать путь

### Сравнение с Дейкстрой:
| Характеристика | Дейкстра | Беллман-Форд |
|----------------|----------|--------------|
| Отрицательные веса | Нет | Да |
| Отрицательные циклы | Не работает | Обнаруживает |
| Сложность | O((V+E) log V) | O(V × E) |
| Скорость | Быстрее | Медленнее |

## Как работает

### Идея алгоритма

1. Инициализировать расстояния: start = 0, остальные = ∞
2. **Релаксация**: повторить V-1 раз для всех рёбер:
   - Если dist[u] + weight < dist[v], обновить dist[v]
3. **Проверка на отрицательный цикл**: ещё одна итерация
   - Если расстояние уменьшилось — есть отрицательный цикл

### Визуализация процесса

```
Граф:
    A ──5── B
     \     /|
      \   / |
      -2 1  -3
        \/  |
         C ─┘

Рёбра: (A,B,5), (A,C,-2), (B,C,1), (C,B,-3)

Начинаем с A:

Итерация 0:
dist = {A: 0, B: ∞, C: ∞}

Итерация 1:
- (A,B,5): dist[B] = min(∞, 0+5) = 5
- (A,C,-2): dist[C] = min(∞, 0-2) = -2
- (B,C,1): dist[C] = min(-2, 5+1) = -2 (не меняется)
- (C,B,-3): dist[B] = min(5, -2-3) = -5
dist = {A: 0, B: -5, C: -2}

Итерация 2 (V-1 = 2):
- (A,B,5): dist[B] = min(-5, 0+5) = -5 (не меняется)
- (A,C,-2): dist[C] = min(-2, 0-2) = -2 (не меняется)
- (B,C,1): dist[C] = min(-2, -5+1) = -4
- (C,B,-3): dist[B] = min(-5, -4-3) = -7
dist = {A: 0, B: -7, C: -4}

Проверка на цикл (итерация 3):
- (C,B,-3): dist[B] = min(-7, -4-3) = -7 (не меняется)
Но если бы изменилось — это отрицательный цикл!

Ой! Фактически здесь ЕСТЬ отрицательный цикл: B → C → B
-3 + 1 = -2 (каждый обход уменьшает на 2)
```

### Схема релаксации

```
Релаксация ребра (u, v, weight):

Если dist[u] + weight < dist[v]:
    dist[v] = dist[u] + weight
    parent[v] = u

    ┌─────┐  weight  ┌─────┐
    │  u  │ ───────► │  v  │
    │ d_u │          │ d_v │
    └─────┘          └─────┘

    Новый путь через u короче?
    d_u + weight < d_v ?
```

## Псевдокод

```
function BellmanFord(graph, start):
    dist = {v: ∞ for v in graph.vertices}
    dist[start] = 0

    # V-1 итераций релаксации
    for i from 1 to V-1:
        for each edge (u, v, weight) in graph.edges:
            if dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight

    # Проверка на отрицательный цикл
    for each edge (u, v, weight) in graph.edges:
        if dist[u] + weight < dist[v]:
            return "Negative cycle detected"

    return dist
```

### Реализация на Python

```python
def bellman_ford(vertices, edges, start):
    """
    Алгоритм Беллмана-Форда.

    Args:
        vertices: список вершин
        edges: список рёбер [(u, v, weight), ...]
        start: начальная вершина

    Returns:
        (distances, has_negative_cycle)
    """
    # Инициализация
    dist = {v: float('inf') for v in vertices}
    dist[start] = 0

    # V-1 итераций релаксации
    for _ in range(len(vertices) - 1):
        updated = False
        for u, v, weight in edges:
            if dist[u] != float('inf') and dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                updated = True

        # Оптимизация: если ничего не изменилось, можно выйти раньше
        if not updated:
            break

    # Проверка на отрицательный цикл
    for u, v, weight in edges:
        if dist[u] != float('inf') and dist[u] + weight < dist[v]:
            return dist, True  # Есть отрицательный цикл

    return dist, False
```

### С восстановлением пути

```python
def bellman_ford_with_path(vertices, edges, start, end):
    """Беллман-Форд с восстановлением пути."""
    dist = {v: float('inf') for v in vertices}
    parent = {v: None for v in vertices}
    dist[start] = 0

    for _ in range(len(vertices) - 1):
        for u, v, weight in edges:
            if dist[u] != float('inf') and dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                parent[v] = u

    # Проверка на отрицательный цикл
    for u, v, weight in edges:
        if dist[u] != float('inf') and dist[u] + weight < dist[v]:
            return None, None, True  # Отрицательный цикл

    # Восстановление пути
    if dist[end] == float('inf'):
        return float('inf'), [], False

    path = []
    current = end
    while current is not None:
        path.append(current)
        current = parent[current]

    return dist[end], path[::-1], False
```

### Обнаружение вершин в отрицательном цикле

```python
def find_negative_cycle(vertices, edges, start):
    """Найти все вершины, достижимые из отрицательного цикла."""
    dist = {v: float('inf') for v in vertices}
    parent = {v: None for v in vertices}
    dist[start] = 0

    changed_vertex = None

    for i in range(len(vertices)):
        changed_vertex = None
        for u, v, weight in edges:
            if dist[u] != float('inf') and dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                parent[v] = u
                changed_vertex = v

    if changed_vertex is None:
        return None  # Нет отрицательного цикла

    # Находим вершину в цикле (идём по parent V раз)
    cycle_vertex = changed_vertex
    for _ in range(len(vertices)):
        cycle_vertex = parent[cycle_vertex]

    # Восстанавливаем цикл
    cycle = []
    current = cycle_vertex
    while True:
        cycle.append(current)
        current = parent[current]
        if current == cycle_vertex:
            cycle.append(current)
            break

    return cycle[::-1]
```

## Анализ сложности

| Операция | Сложность |
|----------|-----------|
| **Время** | O(V × E) |
| **Память** | O(V) |

Где V — вершины, E — рёбра.

### Оптимизация: ранний выход

```python
def bellman_ford_optimized(vertices, edges, start):
    """С ранним выходом, если нет изменений."""
    dist = {v: float('inf') for v in vertices}
    dist[start] = 0

    for i in range(len(vertices) - 1):
        any_change = False

        for u, v, weight in edges:
            if dist[u] != float('inf') and dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                any_change = True

        if not any_change:
            break  # Раньше времени!

    # Проверка на цикл...
    return dist
```

## Пример с пошаговым разбором

### Задача: Валютный арбитраж

```python
"""
Валюты: USD, EUR, GBP
Курсы:
- 1 USD = 0.9 EUR
- 1 EUR = 0.8 GBP
- 1 GBP = 1.4 USD

Есть ли возможность арбитража?
(Купить → продать → заработать)
"""

import math

def currency_arbitrage(rates):
    """
    Ищем отрицательный цикл в графе обмена валют.
    Используем -log(rate) для преобразования произведения в сумму.
    """
    currencies = list(rates.keys())

    # Создаём рёбра: -log(rate)
    edges = []
    for src in rates:
        for dst, rate in rates[src].items():
            edges.append((src, dst, -math.log(rate)))

    # Запускаем Беллман-Форд
    dist = {c: float('inf') for c in currencies}
    dist[currencies[0]] = 0

    for _ in range(len(currencies) - 1):
        for u, v, w in edges:
            if dist[u] != float('inf') and dist[u] + w < dist[v]:
                dist[v] = dist[u] + w

    # Проверяем на отрицательный цикл
    for u, v, w in edges:
        if dist[u] != float('inf') and dist[u] + w < dist[v]:
            return True  # Арбитраж возможен!

    return False

rates = {
    'USD': {'EUR': 0.9},
    'EUR': {'GBP': 0.8},
    'GBP': {'USD': 1.4}
}

print(currency_arbitrage(rates))  # True!
# 1 USD → 0.9 EUR → 0.72 GBP → 1.008 USD (прибыль!)
```

### SPFA — оптимизация Беллмана-Форда

```python
from collections import deque

def spfa(graph, start):
    """
    Shortest Path Faster Algorithm.
    Оптимизация Беллмана-Форда с использованием очереди.
    Средний случай: O(E), худший: O(V × E).
    """
    dist = {v: float('inf') for v in graph}
    dist[start] = 0

    queue = deque([start])
    in_queue = {start}
    count = {v: 0 for v in graph}  # Для обнаружения цикла

    while queue:
        u = queue.popleft()
        in_queue.discard(u)

        for v, weight in graph[u]:
            if dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight

                if v not in in_queue:
                    queue.append(v)
                    in_queue.add(v)
                    count[v] += 1

                    if count[v] >= len(graph):
                        return None  # Отрицательный цикл

    return dist
```

## Типичные ошибки и Edge Cases

### 1. Забыли проверить dist[u] != inf

```python
# ОШИБКА: релаксация из недостижимой вершины
for u, v, weight in edges:
    if dist[u] + weight < dist[v]:  # inf + w может быть < inf!
        dist[v] = dist[u] + weight

# ПРАВИЛЬНО
for u, v, weight in edges:
    if dist[u] != float('inf') and dist[u] + weight < dist[v]:
        dist[v] = dist[u] + weight
```

### 2. V-1 vs V итераций

```python
# V-1 итераций для нахождения кратчайших путей
# V-я итерация — только для проверки на цикл
for _ in range(len(vertices) - 1):  # V-1!
    # релаксация

for u, v, w in edges:  # Проверка
    if dist[u] + w < dist[v]:
        return "Negative cycle"
```

### 3. Несвязный граф

```python
def bellman_ford_all(vertices, edges):
    """Для несвязного графа — запускаем из каждой компоненты."""
    visited = set()
    all_dist = {}

    for start in vertices:
        if start in visited:
            continue

        dist, has_cycle = bellman_ford(vertices, edges, start)

        for v, d in dist.items():
            if d != float('inf'):
                visited.add(v)
                all_dist[v] = d

    return all_dist
```

### 4. Ориентированный vs неориентированный граф

```python
# Для неориентированного графа — добавляем оба направления
def make_undirected(edges):
    result = []
    for u, v, w in edges:
        result.append((u, v, w))
        result.append((v, u, w))  # Обратное ребро
    return result
```

### 5. Целочисленное переполнение

```python
# В некоторых языках inf + weight может переполниться
# Python обрабатывает это корректно, но в C++ будьте осторожны!

# C++: используйте проверку
if (dist[u] != INF && dist[u] + weight < dist[v])
```

### Рекомендации:
1. Используйте Беллман-Форд только при наличии отрицательных весов
2. Для положительных весов — Дейкстра быстрее
3. SPFA быстрее в среднем, но имеет ту же худшую сложность
4. Всегда проверяйте на отрицательные циклы
5. Для валютного арбитража используйте -log(rate)

---
[prev: 03-dijkstra](./03-dijkstra.md) | [next: 05-minimum-spanning-tree](./05-minimum-spanning-tree.md)