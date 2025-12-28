# Взвешенные графы (Weighted Graphs)

[prev: 03-representations](./03-representations.md) | [next: 01-what-is-algorithm](../../12-algorithms/00-intro/01-what-is-algorithm.md)
---

## Определение

**Взвешенный граф** — это граф, в котором каждому ребру присвоено числовое значение (вес). Вес может представлять расстояние, стоимость, время, пропускную способность и т.д.

```
Невзвешенный:           Взвешенный:
   A ─── B                 A ──5── B
   │     │                 │       │
   │     │                 3       2
   │     │                 │       │
   D ─── C                 D ──4── C

Все рёбра равны          Рёбра имеют разные веса
```

## Зачем нужно

### Практическое применение:
- **Навигация** — расстояние между городами
- **Сети** — пропускная способность каналов
- **Финансы** — стоимость транзакций
- **Логистика** — время доставки
- **Игры** — стоимость перемещения

## Представление

### Матрица смежности

```
Граф:                  Матрица:
   A ──5── B              A    B    C    D
   │       │           A [ 0    5    0    3 ]
   3       2           B [ 5    0    2    0 ]
   │       │           C [ 0    2    0    4 ]
   D ──4── C           D [ 3    0    4    0 ]

0 — нет ребра (или ∞ для алгоритмов)
```

```python
class WeightedMatrixGraph:
    def __init__(self, num_vertices):
        self.V = num_vertices
        self.matrix = [[float('inf')] * num_vertices
                       for _ in range(num_vertices)]
        for i in range(num_vertices):
            self.matrix[i][i] = 0

    def add_edge(self, u, v, weight):
        self.matrix[u][v] = weight
        self.matrix[v][u] = weight  # для неориентированного
```

### Список смежности

```
Граф:                  Список:
   A ──5── B           A: [(B, 5), (D, 3)]
   │       │           B: [(A, 5), (C, 2)]
   3       2           C: [(B, 2), (D, 4)]
   │       │           D: [(A, 3), (C, 4)]
   D ──4── C
```

```python
from collections import defaultdict

class WeightedListGraph:
    def __init__(self):
        self.adj = defaultdict(list)

    def add_edge(self, u, v, weight):
        self.adj[u].append((v, weight))
        self.adj[v].append((u, weight))

    def get_weight(self, u, v):
        for neighbor, weight in self.adj[u]:
            if neighbor == v:
                return weight
        return None
```

## Алгоритмы кратчайшего пути

### Алгоритм Дейкстры

Находит кратчайшие пути от источника ко всем вершинам.
**Условие:** неотрицательные веса.

```python
import heapq

def dijkstra(graph, start):
    """
    Алгоритм Дейкстры.
    Время: O((V + E) log V)
    """
    dist = {v: float('inf') for v in graph}
    dist[start] = 0
    prev = {v: None for v in graph}
    pq = [(0, start)]  # (distance, vertex)

    while pq:
        d, u = heapq.heappop(pq)

        if d > dist[u]:
            continue  # устаревшая запись

        for v, weight in graph[u]:
            new_dist = dist[u] + weight

            if new_dist < dist[v]:
                dist[v] = new_dist
                prev[v] = u
                heapq.heappush(pq, (new_dist, v))

    return dist, prev

def get_path(prev, start, end):
    """Восстанавливает путь"""
    path = []
    current = end

    while current is not None:
        path.append(current)
        current = prev[current]

    return path[::-1] if path[-1] == start else None
```

```
Пример:
       A
      /|\
     1 | 4
    /  |  \
   B-2-C-3-D
    \     /
     5   1
      \ /
       E

Дейкстра от A:
dist = {A: 0, B: 1, C: 3, D: 4, E: 5}

Пути:
A → B: 1 (напрямую)
A → C: 3 (A → B → C)
A → D: 4 (A → C → D или A → D)
A → E: 5 (A → D → E)
```

### Алгоритм Беллмана-Форда

Находит кратчайшие пути, **работает с отрицательными весами**.
Может обнаружить отрицательные циклы.

```python
def bellman_ford(vertices, edges, start):
    """
    Алгоритм Беллмана-Форда.
    Время: O(V × E)

    vertices: список вершин
    edges: список рёбер [(u, v, weight), ...]
    """
    dist = {v: float('inf') for v in vertices}
    dist[start] = 0
    prev = {v: None for v in vertices}

    # Релаксация V-1 раз
    for _ in range(len(vertices) - 1):
        for u, v, weight in edges:
            if dist[u] + weight < dist[v]:
                dist[v] = dist[u] + weight
                prev[v] = u

    # Проверка на отрицательный цикл
    for u, v, weight in edges:
        if dist[u] + weight < dist[v]:
            raise ValueError("Graph contains negative cycle")

    return dist, prev
```

### Алгоритм Флойда-Уоршелла

Находит кратчайшие пути **между всеми парами** вершин.

```python
def floyd_warshall(matrix):
    """
    Алгоритм Флойда-Уоршелла.
    Время: O(V³)
    Память: O(V²)
    """
    n = len(matrix)
    dist = [row[:] for row in matrix]  # копия матрицы

    for k in range(n):
        for i in range(n):
            for j in range(n):
                if dist[i][k] + dist[k][j] < dist[i][j]:
                    dist[i][j] = dist[i][k] + dist[k][j]

    # Проверка на отрицательный цикл
    for i in range(n):
        if dist[i][i] < 0:
            raise ValueError("Negative cycle detected")

    return dist
```

## Минимальное остовное дерево (MST)

### Алгоритм Прима

```python
import heapq

def prim(graph, start):
    """
    Алгоритм Прима.
    Строит MST от заданной вершины.
    Время: O((V + E) log V)
    """
    mst = []
    total_weight = 0
    visited = set([start])
    edges = [(weight, start, v) for v, weight in graph[start]]
    heapq.heapify(edges)

    while edges:
        weight, u, v = heapq.heappop(edges)

        if v in visited:
            continue

        visited.add(v)
        mst.append((u, v, weight))
        total_weight += weight

        for neighbor, w in graph[v]:
            if neighbor not in visited:
                heapq.heappush(edges, (w, v, neighbor))

    return mst, total_weight
```

### Алгоритм Крускала

```python
def kruskal(vertices, edges):
    """
    Алгоритм Крускала.
    Использует Union-Find для обнаружения циклов.
    Время: O(E log E)
    """
    # Union-Find
    parent = {v: v for v in vertices}
    rank = {v: 0 for v in vertices}

    def find(x):
        if parent[x] != x:
            parent[x] = find(parent[x])  # path compression
        return parent[x]

    def union(x, y):
        px, py = find(x), find(y)
        if px == py:
            return False

        if rank[px] < rank[py]:
            px, py = py, px
        parent[py] = px
        if rank[px] == rank[py]:
            rank[px] += 1

        return True

    # Сортируем рёбра по весу
    sorted_edges = sorted(edges, key=lambda e: e[2])
    mst = []
    total_weight = 0

    for u, v, weight in sorted_edges:
        if union(u, v):
            mst.append((u, v, weight))
            total_weight += weight

            if len(mst) == len(vertices) - 1:
                break

    return mst, total_weight
```

```
Пример MST:

Граф:                   MST:
   A--5--B                A    B
   |\   /|                 \   |
   3 \ / 2                  3  2
   |  X  |                  | /
   | / \ |                  |/
   D--4--C                  D----C
                               4

Вес MST = 3 + 2 + 4 = 9
```

## Сравнение алгоритмов

| Алгоритм | Время | Применение |
|----------|-------|------------|
| Дейкстра | O((V+E)log V) | Кратчайший путь, ≥0 веса |
| Беллман-Форд | O(VE) | Кратчайший путь, любые веса |
| Флойд-Уоршелл | O(V³) | Все пары, плотный граф |
| Прим | O((V+E)log V) | MST, плотный граф |
| Крускал | O(E log E) | MST, разреженный граф |

## Примеры с разбором

### Пример 1: Навигация

```python
def find_shortest_route(graph, start, end):
    """Находит кратчайший маршрут между городами"""
    dist, prev = dijkstra(graph, start)

    if dist[end] == float('inf'):
        return None, float('inf')

    path = get_path(prev, start, end)
    return path, dist[end]

# Граф городов с расстояниями
cities = {
    'Moscow': [('SPb', 700), ('Kazan', 800)],
    'SPb': [('Moscow', 700), ('Helsinki', 400)],
    'Kazan': [('Moscow', 800), ('Yekaterinburg', 900)],
    'Helsinki': [('SPb', 400)],
    'Yekaterinburg': [('Kazan', 900)]
}

path, distance = find_shortest_route(cities, 'Moscow', 'Helsinki')
# path = ['Moscow', 'SPb', 'Helsinki']
# distance = 1100
```

### Пример 2: Сетевая задержка

```python
def network_delay(times, n, source):
    """
    times: [[from, to, delay], ...]
    Возвращает минимальное время для достижения всех узлов.
    """
    graph = defaultdict(list)
    for u, v, w in times:
        graph[u].append((v, w))

    dist = {i: float('inf') for i in range(1, n + 1)}
    dist[source] = 0
    pq = [(0, source)]

    while pq:
        d, u = heapq.heappop(pq)
        if d > dist[u]:
            continue

        for v, w in graph[u]:
            if dist[u] + w < dist[v]:
                dist[v] = dist[u] + w
                heapq.heappush(pq, (dist[v], v))

    max_delay = max(dist.values())
    return max_delay if max_delay < float('inf') else -1
```

### Пример 3: Минимальная стоимость соединения точек

```python
def min_cost_connect_points(points):
    """
    Находит минимальную стоимость соединения всех точек.
    Стоимость = Manhattan distance.
    """
    n = len(points)
    edges = []

    # Строим полный граф
    for i in range(n):
        for j in range(i + 1, n):
            dist = abs(points[i][0] - points[j][0]) + \
                   abs(points[i][1] - points[j][1])
            edges.append((i, j, dist))

    # Применяем Крускала
    vertices = list(range(n))
    _, total = kruskal(vertices, edges)
    return total

# points = [[0,0], [2,2], [3,10], [5,2], [7,0]]
# Result: 20
```

## Типичные ошибки

### 1. Дейкстра с отрицательными весами

```python
# НЕПРАВИЛЬНО — Дейкстра не работает с отрицательными весами
graph = {
    'A': [('B', -1), ('C', 4)],
    'B': [('C', 3)],
    'C': []
}
dijkstra(graph, 'A')  # Неверный результат!

# ПРАВИЛЬНО — использовать Беллман-Форд
bellman_ford(['A', 'B', 'C'],
             [('A', 'B', -1), ('A', 'C', 4), ('B', 'C', 3)],
             'A')
```

### 2. Забытая проверка устаревших записей

```python
# НЕПРАВИЛЬНО — обрабатываем устаревшие записи
while pq:
    d, u = heapq.heappop(pq)
    for v, w in graph[u]:
        # ...

# ПРАВИЛЬНО
while pq:
    d, u = heapq.heappop(pq)
    if d > dist[u]:
        continue  # пропускаем устаревшее
    for v, w in graph[u]:
        # ...
```

### 3. Неправильная инициализация dist

```python
# НЕПРАВИЛЬНО
dist = {}  # dist[v] вызовет KeyError

# ПРАВИЛЬНО
dist = {v: float('inf') for v in graph}
dist[start] = 0
```

## Когда что использовать

**Дейкстра:**
- Один источник, неотрицательные веса
- Большинство практических задач

**Беллман-Форд:**
- Есть отрицательные веса
- Нужно обнаружить отрицательные циклы

**Флойд-Уоршелл:**
- Нужны все пары кратчайших путей
- Небольшой плотный граф

**Прим:**
- MST для плотного графа
- Известна начальная вершина

**Крускал:**
- MST для разреженного графа
- Рёбра уже в списке

---
[prev: 03-representations](./03-representations.md) | [next: 01-what-is-algorithm](../../12-algorithms/00-intro/01-what-is-algorithm.md)