# Алгоритм Дейкстры (Dijkstra's Algorithm)

[prev: 02-bfs](./02-bfs.md) | [next: 04-bellman-ford](./04-bellman-ford.md)
---

## Определение

**Алгоритм Дейкстры** — это алгоритм поиска кратчайших путей от одной начальной вершины до всех остальных вершин во взвешенном графе с **неотрицательными** весами рёбер.

Назван в честь нидерландского учёного Эдсгера Дейкстры (1959).

## Зачем нужен

### Области применения:
- **GPS-навигация** — кратчайший маршрут
- **Сетевая маршрутизация** — протоколы OSPF, IS-IS
- **Социальные сети** — степени связи между людьми
- **Игры** — поиск пути (pathfinding)
- **Робототехника** — планирование маршрута
- **Авиаперелёты** — оптимальные маршруты

### Ограничения:
- **Работает только с неотрицательными весами**
- Для отрицательных весов используйте Bellman-Ford

## Как работает

### Визуализация процесса

```
Граф:
      A ──2── B
      │       │
      4       1
      │       │
      C ──3── D

Шаг 0: dist = {A: 0, B: ∞, C: ∞, D: ∞}
       pq = [(0, A)]

Шаг 1: Извлекаем A (dist=0)
       Обновляем соседей:
       - B: 0 + 2 = 2 < ∞ ✓
       - C: 0 + 4 = 4 < ∞ ✓
       dist = {A: 0, B: 2, C: 4, D: ∞}
       pq = [(2, B), (4, C)]

Шаг 2: Извлекаем B (dist=2)
       Обновляем соседей:
       - A: уже обработан
       - D: 2 + 1 = 3 < ∞ ✓
       dist = {A: 0, B: 2, C: 4, D: 3}
       pq = [(3, D), (4, C)]

Шаг 3: Извлекаем D (dist=3)
       Обновляем соседей:
       - B: уже обработан
       - C: 3 + 3 = 6 > 4, не обновляем
       dist = {A: 0, B: 2, C: 4, D: 3}
       pq = [(4, C)]

Шаг 4: Извлекаем C (dist=4)
       Все соседи обработаны
       dist = {A: 0, B: 2, C: 4, D: 3}

Результат: кратчайшие расстояния от A
```

### Схема работы

```
              ┌─────────────────────────────────────┐
              │  Priority Queue (min-heap)          │
              │  Содержит вершины с расстояниями    │
              └────────────────┬────────────────────┘
                               │
                               ▼
              ┌─────────────────────────────────────┐
              │  Извлечь вершину с MIN расстоянием  │
              └────────────────┬────────────────────┘
                               │
                               ▼
              ┌─────────────────────────────────────┐
              │  Для каждого соседа:                │
              │  Если новый путь короче → обновить  │
              │  и добавить в очередь               │
              └────────────────┬────────────────────┘
                               │
                               ▼
                    Повторять пока очередь не пуста
```

## Псевдокод

```
function Dijkstra(graph, start):
    dist = {v: ∞ for v in graph}
    dist[start] = 0
    priority_queue = [(0, start)]

    while priority_queue is not empty:
        current_dist, u = extract_min(priority_queue)

        if current_dist > dist[u]:
            continue  # Устаревшая запись

        for (v, weight) in graph[u]:
            new_dist = dist[u] + weight

            if new_dist < dist[v]:
                dist[v] = new_dist
                priority_queue.insert((new_dist, v))

    return dist
```

### Реализация на Python

```python
import heapq

def dijkstra(graph, start):
    """
    Алгоритм Дейкстры.

    Args:
        graph: словарь смежности {vertex: [(neighbor, weight), ...]}
        start: начальная вершина

    Returns:
        словарь кратчайших расстояний от start
    """
    distances = {v: float('inf') for v in graph}
    distances[start] = 0
    pq = [(0, start)]  # (расстояние, вершина)

    while pq:
        current_dist, u = heapq.heappop(pq)

        # Пропускаем устаревшие записи
        if current_dist > distances[u]:
            continue

        for v, weight in graph[u]:
            new_dist = current_dist + weight

            if new_dist < distances[v]:
                distances[v] = new_dist
                heapq.heappush(pq, (new_dist, v))

    return distances
```

### С восстановлением пути

```python
def dijkstra_with_path(graph, start, end):
    """Дейкстра с восстановлением пути."""
    distances = {v: float('inf') for v in graph}
    distances[start] = 0
    parent = {start: None}
    pq = [(0, start)]

    while pq:
        current_dist, u = heapq.heappop(pq)

        if u == end:
            # Восстанавливаем путь
            path = []
            current = end
            while current is not None:
                path.append(current)
                current = parent[current]
            return distances[end], path[::-1]

        if current_dist > distances[u]:
            continue

        for v, weight in graph[u]:
            new_dist = current_dist + weight

            if new_dist < distances[v]:
                distances[v] = new_dist
                parent[v] = u
                heapq.heappush(pq, (new_dist, v))

    return float('inf'), []

# Пример
graph = {
    'A': [('B', 2), ('C', 4)],
    'B': [('A', 2), ('D', 1)],
    'C': [('A', 4), ('D', 3)],
    'D': [('B', 1), ('C', 3)]
}
dist, path = dijkstra_with_path(graph, 'A', 'D')
print(f"Расстояние: {dist}, Путь: {path}")  # Расстояние: 3, Путь: ['A', 'B', 'D']
```

### Для матрицы смежности

```python
def dijkstra_matrix(matrix, start):
    """Дейкстра для графа в виде матрицы смежности."""
    n = len(matrix)
    distances = [float('inf')] * n
    distances[start] = 0
    visited = [False] * n

    for _ in range(n):
        # Находим непосещённую вершину с минимальным расстоянием
        u = -1
        for i in range(n):
            if not visited[i] and (u == -1 or distances[i] < distances[u]):
                u = i

        if distances[u] == float('inf'):
            break

        visited[u] = True

        # Обновляем расстояния до соседей
        for v in range(n):
            if matrix[u][v] > 0 and not visited[v]:
                new_dist = distances[u] + matrix[u][v]
                if new_dist < distances[v]:
                    distances[v] = new_dist

    return distances
```

### Bidirectional Dijkstra

```python
def bidirectional_dijkstra(graph, reverse_graph, start, end):
    """
    Двунаправленный Дейкстра — ищем с обеих сторон.
    Эффективнее для поиска пути между двумя вершинами.
    """
    if start == end:
        return 0

    dist_forward = {start: 0}
    dist_backward = {end: 0}

    pq_forward = [(0, start)]
    pq_backward = [(0, end)]

    processed_forward = set()
    processed_backward = set()

    best_path = float('inf')

    while pq_forward or pq_backward:
        # Шаг вперёд
        if pq_forward:
            d, u = heapq.heappop(pq_forward)
            if u in processed_forward:
                continue
            processed_forward.add(u)

            if u in processed_backward:
                best_path = min(best_path, dist_forward[u] + dist_backward[u])

            for v, w in graph[u]:
                new_dist = d + w
                if v not in dist_forward or new_dist < dist_forward[v]:
                    dist_forward[v] = new_dist
                    heapq.heappush(pq_forward, (new_dist, v))

        # Шаг назад
        if pq_backward:
            d, u = heapq.heappop(pq_backward)
            if u in processed_backward:
                continue
            processed_backward.add(u)

            if u in processed_forward:
                best_path = min(best_path, dist_forward[u] + dist_backward[u])

            for v, w in reverse_graph[u]:
                new_dist = d + w
                if v not in dist_backward or new_dist < dist_backward[v]:
                    dist_backward[v] = new_dist
                    heapq.heappush(pq_backward, (new_dist, v))

        # Ранняя остановка
        if processed_forward and processed_backward:
            min_forward = pq_forward[0][0] if pq_forward else float('inf')
            min_backward = pq_backward[0][0] if pq_backward else float('inf')
            if min_forward + min_backward >= best_path:
                break

    return best_path
```

## Анализ сложности

| Реализация | Время | Память |
|------------|-------|--------|
| Массив (наивная) | O(V^2) | O(V) |
| Бинарная куча | O((V + E) log V) | O(V) |
| Фибоначчиева куча | O(V log V + E) | O(V) |

Где V — вершины, E — рёбра.

## Пример с пошаговым разбором

### Задача: Сеть городов

```python
#       10
#   A ─────── B
#   │ \       │
#   5   2     3
#   │     \   │
#   C ─── 9 ─ D
#     \     /
#      4   6
#       \ /
#        E

graph = {
    'A': [('B', 10), ('C', 5), ('D', 2)],
    'B': [('A', 10), ('D', 3)],
    'C': [('A', 5), ('D', 9), ('E', 4)],
    'D': [('A', 2), ('B', 3), ('C', 9), ('E', 6)],
    'E': [('C', 4), ('D', 6)]
}

distances = dijkstra(graph, 'A')
print(distances)
# {'A': 0, 'B': 5, 'C': 5, 'D': 2, 'E': 8}

# Объяснение:
# A → A: 0
# A → D: 2 (прямо)
# A → B: A → D → B = 2 + 3 = 5
# A → C: 5 (прямо)
# A → E: A → D → E = 2 + 6 = 8
```

## Типичные ошибки и Edge Cases

### 1. Отрицательные веса — Дейкстра НЕ работает!

```python
# НЕПРАВИЛЬНО: отрицательные веса
graph = {
    'A': [('B', 1)],
    'B': [('C', -2)],  # Отрицательный вес!
    'C': [('A', 1)]
}
# Дейкстра даст неправильный ответ!
# Используйте Bellman-Ford

# Если есть отрицательный цикл — кратчайшего пути не существует
```

### 2. Несвязный граф

```python
def dijkstra_safe(graph, start):
    distances = {v: float('inf') for v in graph}
    distances[start] = 0
    # ...
    return distances

# Недостижимые вершины останутся с dist = inf
```

### 3. Дубликаты в очереди

```python
# Дубликаты — это нормально! Проверяем устаревшие записи:
def dijkstra(graph, start):
    # ...
    while pq:
        current_dist, u = heapq.heappop(pq)

        # Пропускаем устаревшие записи!
        if current_dist > distances[u]:
            continue
        # ...
```

### 4. Граф на сетке (матрица)

```python
def dijkstra_grid(grid, start, end):
    """Дейкстра на 2D сетке с весами."""
    rows, cols = len(grid), len(grid[0])
    directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]

    distances = [[float('inf')] * cols for _ in range(rows)]
    distances[start[0]][start[1]] = 0

    pq = [(0, start[0], start[1])]

    while pq:
        dist, r, c = heapq.heappop(pq)

        if (r, c) == end:
            return dist

        if dist > distances[r][c]:
            continue

        for dr, dc in directions:
            nr, nc = r + dr, c + dc

            if 0 <= nr < rows and 0 <= nc < cols:
                # Вес = значение в клетке
                new_dist = dist + grid[nr][nc]

                if new_dist < distances[nr][nc]:
                    distances[nr][nc] = new_dist
                    heapq.heappush(pq, (new_dist, nr, nc))

    return float('inf')
```

### 5. K кратчайших путей

```python
def k_shortest_paths(graph, start, end, k):
    """Найти k кратчайших путей."""
    pq = [(0, start, [start])]
    count = {v: 0 for v in graph}
    paths = []

    while pq and len(paths) < k:
        dist, u, path = heapq.heappop(pq)

        count[u] += 1

        if u == end:
            paths.append((dist, path))
            continue

        if count[u] > k:
            continue

        for v, weight in graph[u]:
            heapq.heappush(pq, (dist + weight, v, path + [v]))

    return paths
```

### Рекомендации:
1. Всегда проверяйте на отрицательные веса перед применением
2. Используйте бинарную кучу (heapq) для эффективности
3. Пропускайте устаревшие записи в очереди
4. Для поиска пути между двумя вершинами — остановитесь сразу при достижении цели
5. Bidirectional Dijkstra может быть значительно быстрее для больших графов

---
[prev: 02-bfs](./02-bfs.md) | [next: 04-bellman-ford](./04-bellman-ford.md)