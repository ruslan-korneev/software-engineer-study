# Минимальное остовное дерево (Minimum Spanning Tree)

[prev: 04-bellman-ford](./04-bellman-ford.md) | [next: 01-backtracking-concept](../06-backtracking/01-backtracking-concept.md)
---

## Определение

**Минимальное остовное дерево (MST)** — это подграф связного взвешенного неориентированного графа, который:
- Является деревом (связный, без циклов)
- Содержит все вершины исходного графа
- Имеет минимальную суммарную стоимость рёбер

Для графа с V вершинами MST содержит ровно V-1 рёбер.

## Зачем нужен

### Области применения:
- **Сетевое проектирование** — минимальная стоимость кабельной сети
- **Кластерный анализ** — разбиение данных на группы
- **Приближённые алгоритмы** — TSP, Steiner Tree
- **Распознавание образов**
- **Биоинформатика** — филогенетические деревья

## Алгоритм Краскала (Kruskal)

### Идея
1. Отсортировать все рёбра по весу
2. Добавлять рёбра в порядке возрастания веса
3. Пропускать рёбра, создающие цикл (проверка через Union-Find)

### Визуализация

```
Граф:
    A ──4── B
    |\     /|
    2  3  3  5
    |    X   |
    C ──1── D
     \     /
      6   7
       \ /
        E

Рёбра по весу: (C,D,1), (A,C,2), (A,B,3) или (B,D,3), (A,D,4), (B,E,5)...

Шаг 1: (C,D,1) — добавляем
       C───D

Шаг 2: (A,C,2) — добавляем
       A
       |
       C───D

Шаг 3: (A,B,3) — добавляем
       A───B
       |
       C───D

Шаг 4: (B,D,3) — пропускаем (создаёт цикл A-B-D-C-A)

Шаг 5: (A,D,4) — пропускаем (цикл)

Шаг 6: (B,E,5) — добавляем
       A───B
       |   |
       C───D E

Подождите, E не связан! Нужно добавить ребро к E.
Допустим (C,E,6) — добавляем

MST:
       A───B
       |   |
       C───D
        \
         E

Стоимость: 1 + 2 + 3 + 5 = 11
```

### Реализация с Union-Find

```python
class UnionFind:
    """Структура данных для быстрого объединения множеств."""

    def __init__(self, n):
        self.parent = list(range(n))
        self.rank = [0] * n

    def find(self, x):
        """Найти корень множества (с сжатием пути)."""
        if self.parent[x] != x:
            self.parent[x] = self.find(self.parent[x])
        return self.parent[x]

    def union(self, x, y):
        """Объединить два множества (по рангу)."""
        root_x = self.find(x)
        root_y = self.find(y)

        if root_x == root_y:
            return False  # Уже в одном множестве

        if self.rank[root_x] < self.rank[root_y]:
            root_x, root_y = root_y, root_x

        self.parent[root_y] = root_x

        if self.rank[root_x] == self.rank[root_y]:
            self.rank[root_x] += 1

        return True


def kruskal(vertices, edges):
    """
    Алгоритм Краскала для MST.

    Args:
        vertices: список вершин (или их количество)
        edges: список рёбер [(u, v, weight), ...]

    Returns:
        (mst_edges, total_weight)
    """
    # Создаём отображение вершин на индексы
    if isinstance(vertices, int):
        n = vertices
        vertex_to_idx = {i: i for i in range(n)}
    else:
        n = len(vertices)
        vertex_to_idx = {v: i for i, v in enumerate(vertices)}

    # Сортируем рёбра по весу
    sorted_edges = sorted(edges, key=lambda e: e[2])

    uf = UnionFind(n)
    mst = []
    total_weight = 0

    for u, v, weight in sorted_edges:
        u_idx = vertex_to_idx[u]
        v_idx = vertex_to_idx[v]

        # Если объединение успешно — ребро не создаёт цикл
        if uf.union(u_idx, v_idx):
            mst.append((u, v, weight))
            total_weight += weight

            # MST готов, когда есть V-1 рёбер
            if len(mst) == n - 1:
                break

    return mst, total_weight


# Пример
vertices = ['A', 'B', 'C', 'D', 'E']
edges = [
    ('A', 'B', 4), ('A', 'C', 2), ('B', 'C', 3),
    ('B', 'D', 3), ('C', 'D', 1), ('C', 'E', 6),
    ('D', 'E', 5)
]
mst, weight = kruskal(vertices, edges)
print(f"MST: {mst}")
print(f"Total weight: {weight}")
```

## Алгоритм Прима (Prim)

### Идея
1. Начать с произвольной вершины
2. На каждом шаге добавлять минимальное ребро, соединяющее дерево с непосещённой вершиной
3. Повторять, пока все вершины не будут включены

### Визуализация

```
Граф (тот же):
    A ──4── B
    |\     /|
    2  3  3  5
    |    X   |
    C ──1── D
     \     /
      6   7
       \ /
        E

Начинаем с A:

Шаг 1: Дерево = {A}
       Доступные рёбра: (A,B,4), (A,C,2), (A,D,3)
       Минимальное: (A,C,2)
       Дерево = {A, C}

Шаг 2: Доступные: (A,B,4), (A,D,3), (C,D,1), (C,E,6)
       Минимальное: (C,D,1)
       Дерево = {A, C, D}

Шаг 3: Доступные: (A,B,4), (D,B,3), (D,E,7), (C,E,6)
       Минимальное: (D,B,3)
       Дерево = {A, C, D, B}

Шаг 4: Доступные: (B,E,5), (D,E,7), (C,E,6)
       Минимальное: (B,E,5)
       Дерево = {A, C, D, B, E}

MST рёбра: (A,C), (C,D), (D,B), (B,E)
Стоимость: 2 + 1 + 3 + 5 = 11
```

### Реализация с приоритетной очередью

```python
import heapq

def prim(graph, start=None):
    """
    Алгоритм Прима для MST.

    Args:
        graph: словарь смежности {vertex: [(neighbor, weight), ...]}
        start: начальная вершина (по умолчанию — первая)

    Returns:
        (mst_edges, total_weight)
    """
    if start is None:
        start = next(iter(graph))

    visited = {start}
    edges = [(weight, start, neighbor) for neighbor, weight in graph[start]]
    heapq.heapify(edges)

    mst = []
    total_weight = 0

    while edges and len(visited) < len(graph):
        weight, u, v = heapq.heappop(edges)

        if v in visited:
            continue

        visited.add(v)
        mst.append((u, v, weight))
        total_weight += weight

        for neighbor, edge_weight in graph[v]:
            if neighbor not in visited:
                heapq.heappush(edges, (edge_weight, v, neighbor))

    return mst, total_weight


# Пример
graph = {
    'A': [('B', 4), ('C', 2), ('D', 3)],
    'B': [('A', 4), ('C', 3), ('D', 3), ('E', 5)],
    'C': [('A', 2), ('B', 3), ('D', 1), ('E', 6)],
    'D': [('A', 3), ('B', 3), ('C', 1), ('E', 7)],
    'E': [('B', 5), ('C', 6), ('D', 7)]
}

mst, weight = prim(graph, 'A')
print(f"MST: {mst}")
print(f"Total weight: {weight}")
```

### Прим для плотных графов (матрица смежности)

```python
def prim_matrix(adj_matrix):
    """Прим для графа в виде матрицы смежности."""
    n = len(adj_matrix)
    INF = float('inf')

    visited = [False] * n
    min_edge = [INF] * n
    parent = [-1] * n

    min_edge[0] = 0
    total_weight = 0

    for _ in range(n):
        # Находим непосещённую вершину с минимальным ребром
        u = -1
        for i in range(n):
            if not visited[i] and (u == -1 or min_edge[i] < min_edge[u]):
                u = i

        if min_edge[u] == INF:
            return None, None  # Граф несвязный

        visited[u] = True
        total_weight += min_edge[u]

        # Обновляем минимальные рёбра для соседей
        for v in range(n):
            if adj_matrix[u][v] > 0 and not visited[v]:
                if adj_matrix[u][v] < min_edge[v]:
                    min_edge[v] = adj_matrix[u][v]
                    parent[v] = u

    # Формируем список рёбер MST
    mst = []
    for i in range(1, n):
        if parent[i] != -1:
            mst.append((parent[i], i, adj_matrix[parent[i]][i]))

    return mst, total_weight
```

## Анализ сложности

| Алгоритм | Время | Структура данных |
|----------|-------|------------------|
| **Краскал** | O(E log E) | Union-Find |
| **Прим (куча)** | O(E log V) | Бинарная куча |
| **Прим (матрица)** | O(V^2) | Массив |
| **Прим (Фибоначчи)** | O(E + V log V) | Фибоначчиева куча |

### Когда какой использовать:
- **Краскал**: разреженные графы (E << V^2)
- **Прим с кучей**: разреженные графы
- **Прим с матрицей**: плотные графы (E ≈ V^2)

## Типичные ошибки и Edge Cases

### 1. Несвязный граф

```python
def has_mst(vertices, edges):
    """Проверка, существует ли MST (граф связный?)."""
    mst, _ = kruskal(vertices, edges)
    return len(mst) == len(vertices) - 1
```

### 2. Несколько MST

```python
# Если есть рёбра с одинаковыми весами, может быть несколько MST
# Оба алгоритма найдут одно из них (зависит от порядка обработки)
```

### 3. Одна вершина

```python
def mst_safe(vertices, edges):
    if len(vertices) <= 1:
        return [], 0
    # ...
```

### 4. Отрицательные веса

```python
# MST работает с отрицательными весами!
# (в отличие от Дейкстры)
edges = [
    ('A', 'B', -5),  # Отрицательный вес — это нормально
    ('B', 'C', 3),
    ('A', 'C', 2)
]
# MST: (A,B,-5), (A,C,2) или (B,C,3)
# Стоимость: -5 + 2 = -3 или -5 + 3 = -2
```

### 5. Второе по минимальности остовное дерево

```python
def second_best_mst(vertices, edges):
    """Найти второе MST."""
    mst_edges, mst_weight = kruskal(vertices, edges)
    mst_set = set((min(u, v), max(u, v)) for u, v, _ in mst_edges)

    second_best = float('inf')

    # Пробуем исключить каждое ребро MST
    for excluded in mst_edges:
        u, v, w = excluded
        excl_key = (min(u, v), max(u, v))

        # Строим MST без этого ребра
        filtered_edges = [e for e in edges if (min(e[0], e[1]), max(e[0], e[1])) != excl_key]
        new_mst, new_weight = kruskal(vertices, filtered_edges)

        if len(new_mst) == len(vertices) - 1:  # Граф остался связным
            second_best = min(second_best, new_weight)

    return second_best if second_best != float('inf') else None
```

### 6. MST для точек на плоскости

```python
import math

def euclidean_mst(points):
    """MST для точек на плоскости."""
    n = len(points)
    edges = []

    for i in range(n):
        for j in range(i + 1, n):
            dist = math.sqrt(
                (points[i][0] - points[j][0]) ** 2 +
                (points[i][1] - points[j][1]) ** 2
            )
            edges.append((i, j, dist))

    return kruskal(n, edges)

points = [(0, 0), (1, 1), (2, 0), (1, 2)]
mst, weight = euclidean_mst(points)
print(f"MST weight: {weight:.2f}")
```

### Рекомендации:
1. Краскал проще реализовать с Union-Find
2. Прим лучше для плотных графов и когда нужно начать с конкретной вершины
3. Оба алгоритма дают правильный результат
4. MST работает с отрицательными весами
5. Проверяйте связность графа перед построением MST

---
[prev: 04-bellman-ford](./04-bellman-ford.md) | [next: 01-backtracking-concept](../06-backtracking/01-backtracking-concept.md)