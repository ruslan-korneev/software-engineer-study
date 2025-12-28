# Представления графа (Graph Representations)

[prev: 02-directed-undirected](./02-directed-undirected.md) | [next: 04-weighted-graphs](./04-weighted-graphs.md)
---

## Определение

Граф можно представить в памяти компьютера разными способами. Выбор представления влияет на эффективность операций и использование памяти.

## Основные представления

### 1. Матрица смежности (Adjacency Matrix)

Двумерный массив размера V×V, где V — количество вершин.

```
Граф:                  Матрица смежности:
   0 ─── 1                0   1   2   3
   │ \   │            0 [ 0   1   1   1 ]
   │   \ │            1 [ 1   0   1   0 ]
   3 ─── 2            2 [ 1   1   0   1 ]
                      3 [ 1   0   1   0 ]

matrix[i][j] = 1 если есть ребро (i, j), иначе 0
```

#### Реализация

```python
class AdjacencyMatrix:
    def __init__(self, num_vertices):
        self.V = num_vertices
        self.matrix = [[0] * num_vertices for _ in range(num_vertices)]

    def add_edge(self, u, v, weight=1):
        """Добавляет ребро (неориентированный граф)"""
        self.matrix[u][v] = weight
        self.matrix[v][u] = weight

    def add_directed_edge(self, u, v, weight=1):
        """Добавляет дугу (ориентированный граф)"""
        self.matrix[u][v] = weight

    def remove_edge(self, u, v):
        """Удаляет ребро"""
        self.matrix[u][v] = 0
        self.matrix[v][u] = 0

    def has_edge(self, u, v):
        """Проверяет наличие ребра - O(1)"""
        return self.matrix[u][v] != 0

    def get_neighbors(self, v):
        """Возвращает соседей вершины - O(V)"""
        return [i for i in range(self.V) if self.matrix[v][i] != 0]

    def get_weight(self, u, v):
        """Возвращает вес ребра"""
        return self.matrix[u][v]

    def degree(self, v):
        """Степень вершины - O(V)"""
        return sum(1 for x in self.matrix[v] if x != 0)

    def edge_count(self):
        """Количество рёбер - O(V²)"""
        count = sum(1 for i in range(self.V)
                      for j in range(self.V) if self.matrix[i][j] != 0)
        return count // 2  # для неориентированного

    def print_matrix(self):
        for row in self.matrix:
            print(row)
```

#### Анализ

| Операция | Сложность |
|----------|-----------|
| Память | O(V²) |
| Проверка ребра | O(1) |
| Добавление ребра | O(1) |
| Удаление ребра | O(1) |
| Все соседи | O(V) |
| Все рёбра | O(V²) |
| Добавление вершины | O(V²) |

**Преимущества:**
- Проверка ребра за O(1)
- Простая реализация
- Удобно для плотных графов

**Недостатки:**
- Память O(V²) даже для разреженных графов
- Получение всех соседей O(V)
- Добавление вершины требует перевыделения

---

### 2. Список смежности (Adjacency List)

Массив/словарь списков, где каждый элемент — список соседей вершины.

```
Граф:                  Список смежности:
   0 ─── 1            0: [1, 2, 3]
   │ \   │            1: [0, 2]
   │   \ │            2: [0, 1, 3]
   3 ─── 2            3: [0, 2]
```

#### Реализация

```python
from collections import defaultdict

class AdjacencyList:
    def __init__(self):
        self.adj = defaultdict(list)

    def add_vertex(self, v):
        """Добавляет вершину"""
        if v not in self.adj:
            self.adj[v] = []

    def add_edge(self, u, v, weight=1):
        """Добавляет ребро (неориентированный граф)"""
        self.adj[u].append((v, weight))
        self.adj[v].append((u, weight))

    def add_directed_edge(self, u, v, weight=1):
        """Добавляет дугу"""
        self.adj[u].append((v, weight))

    def remove_edge(self, u, v):
        """Удаляет ребро - O(degree)"""
        self.adj[u] = [(n, w) for n, w in self.adj[u] if n != v]
        self.adj[v] = [(n, w) for n, w in self.adj[v] if n != u]

    def has_edge(self, u, v):
        """Проверяет наличие ребра - O(degree)"""
        return any(n == v for n, _ in self.adj[u])

    def get_neighbors(self, v):
        """Возвращает соседей - O(1)"""
        return [n for n, _ in self.adj[v]]

    def get_weight(self, u, v):
        """Возвращает вес ребра - O(degree)"""
        for n, w in self.adj[u]:
            if n == v:
                return w
        return None

    def degree(self, v):
        """Степень вершины - O(1)"""
        return len(self.adj[v])

    def vertices(self):
        """Все вершины"""
        return list(self.adj.keys())

    def edges(self):
        """Все рёбра - O(V + E)"""
        result = []
        seen = set()
        for u in self.adj:
            for v, w in self.adj[u]:
                if (v, u) not in seen:
                    result.append((u, v, w))
                    seen.add((u, v))
        return result
```

#### Вариант без весов

```python
class SimpleAdjList:
    def __init__(self):
        self.adj = defaultdict(set)  # set для быстрой проверки

    def add_edge(self, u, v):
        self.adj[u].add(v)
        self.adj[v].add(u)

    def has_edge(self, u, v):
        return v in self.adj[u]  # O(1) для set
```

#### Анализ

| Операция | Сложность |
|----------|-----------|
| Память | O(V + E) |
| Проверка ребра | O(degree) или O(1)* |
| Добавление ребра | O(1) |
| Удаление ребра | O(degree) |
| Все соседи | O(degree) |
| Все рёбра | O(V + E) |
| Добавление вершины | O(1) |

*O(1) если использовать set вместо list

**Преимущества:**
- Память O(V + E) — эффективно для разреженных графов
- Быстрое получение соседей
- Легко добавлять вершины

**Недостатки:**
- Проверка ребра O(degree) (если не set)
- Сложнее удаление ребра

---

### 3. Список рёбер (Edge List)

Простой список всех рёбер.

```
Граф:                  Список рёбер:
   0 ─── 1            [(0, 1), (0, 2), (0, 3), (1, 2), (2, 3)]
   │ \   │
   │   \ │            Или с весами:
   3 ─── 2            [(0, 1, 5), (0, 2, 3), ...]
```

#### Реализация

```python
class EdgeList:
    def __init__(self):
        self.edges = []
        self.vertices = set()

    def add_edge(self, u, v, weight=1):
        """Добавляет ребро"""
        self.edges.append((u, v, weight))
        self.vertices.add(u)
        self.vertices.add(v)

    def has_edge(self, u, v):
        """Проверяет наличие ребра - O(E)"""
        return any((a == u and b == v) or (a == v and b == u)
                   for a, b, _ in self.edges)

    def get_edges(self):
        """Все рёбра - O(1)"""
        return self.edges

    def get_vertices(self):
        """Все вершины"""
        return self.vertices

    def sort_by_weight(self):
        """Сортировка по весу (для Крускала)"""
        self.edges.sort(key=lambda e: e[2])
```

#### Анализ

| Операция | Сложность |
|----------|-----------|
| Память | O(E) |
| Проверка ребра | O(E) |
| Добавление ребра | O(1) |
| Удаление ребра | O(E) |
| Все рёбра | O(1) |
| Все соседи | O(E) |

**Используется для:**
- Алгоритма Крускала (MST)
- Алгоритма Беллмана-Форда
- Простое хранение графа

---

### 4. Матрица инцидентности (Incidence Matrix)

Матрица V×E, где V — вершины, E — рёбра.

```
Граф:                  Матрица инцидентности:
   0 ─── 1                e0  e1  e2  e3  e4
   │ \   │            0 [ 1   1   1   0   0 ]
   │   \ │            1 [ 1   0   0   1   0 ]
   3 ─── 2            2 [ 0   1   0   1   1 ]
                      3 [ 0   0   1   0   1 ]

e0 = (0,1), e1 = (0,2), e2 = (0,3), e3 = (1,2), e4 = (2,3)
```

**Редко используется** из-за памяти O(V×E).

---

## Сравнение представлений

| | Матрица | Список | Рёбра |
|---|---------|--------|-------|
| **Память** | O(V²) | O(V+E) | O(E) |
| **Проверка ребра** | O(1) | O(d) | O(E) |
| **Добавление ребра** | O(1) | O(1) | O(1) |
| **Удаление ребра** | O(1) | O(d) | O(E) |
| **Соседи** | O(V) | O(d) | O(E) |
| **Все рёбра** | O(V²) | O(E) | O(1) |

d = degree (степень вершины)

### Когда что использовать

```
Матрица смежности:
✓ Плотный граф (E ≈ V²)
✓ Частая проверка наличия ребра
✓ Небольшое V

Список смежности:
✓ Разреженный граф (E << V²)
✓ Частый обход соседей
✓ Большой граф

Список рёбер:
✓ Алгоритмы на рёбрах (Крускал, Беллман-Форд)
✓ Простое хранение
✓ Редкие запросы
```

---

## Примеры с разбором

### Пример 1: Конвертация между представлениями

```python
def matrix_to_list(matrix):
    """Матрица → Список смежности"""
    n = len(matrix)
    adj = {i: [] for i in range(n)}

    for i in range(n):
        for j in range(n):
            if matrix[i][j] != 0:
                adj[i].append((j, matrix[i][j]))

    return adj

def list_to_matrix(adj_list, n):
    """Список смежности → Матрица"""
    matrix = [[0] * n for _ in range(n)]

    for u in adj_list:
        for v, w in adj_list[u]:
            matrix[u][v] = w

    return matrix

def edges_to_list(edges):
    """Список рёбер → Список смежности"""
    adj = defaultdict(list)

    for u, v, w in edges:
        adj[u].append((v, w))
        adj[v].append((u, w))

    return adj
```

### Пример 2: BFS с разными представлениями

```python
from collections import deque

def bfs_matrix(matrix, start):
    """BFS с матрицей смежности"""
    n = len(matrix)
    visited = [False] * n
    queue = deque([start])
    visited[start] = True
    result = []

    while queue:
        v = queue.popleft()
        result.append(v)

        for u in range(n):  # O(V) на каждую вершину!
            if matrix[v][u] != 0 and not visited[u]:
                visited[u] = True
                queue.append(u)

    return result

def bfs_list(adj, start):
    """BFS со списком смежности"""
    visited = {start}
    queue = deque([start])
    result = []

    while queue:
        v = queue.popleft()
        result.append(v)

        for u, _ in adj[v]:  # O(degree) на каждую вершину!
            if u not in visited:
                visited.add(u)
                queue.append(u)

    return result

# Общая сложность:
# Матрица: O(V²)
# Список: O(V + E)
```

### Пример 3: Плотность графа

```python
def graph_density(V, E):
    """
    Плотность графа: отношение рёбер к максимально возможному.
    0 = нет рёбер, 1 = полный граф
    """
    max_edges = V * (V - 1) / 2  # для неориентированного
    return E / max_edges if max_edges > 0 else 0

def recommend_representation(V, E):
    """Рекомендует представление на основе плотности"""
    density = graph_density(V, E)

    if density > 0.5:
        return "Матрица смежности"
    else:
        return "Список смежности"
```

### Пример 4: Оптимизированный список смежности для алгоритмов

```python
class OptimizedGraph:
    """
    Оптимизированный список смежности для алгоритмов.
    Использует массивы вместо словарей для скорости.
    """
    def __init__(self, num_vertices):
        self.V = num_vertices
        self.adj = [[] for _ in range(num_vertices)]

    def add_edge(self, u, v, weight=1):
        self.adj[u].append((v, weight))
        self.adj[v].append((u, weight))

    def dijkstra(self, start):
        """Алгоритм Дейкстры"""
        import heapq

        dist = [float('inf')] * self.V
        dist[start] = 0
        pq = [(0, start)]

        while pq:
            d, u = heapq.heappop(pq)

            if d > dist[u]:
                continue

            for v, w in self.adj[u]:  # быстрый доступ к соседям
                if dist[u] + w < dist[v]:
                    dist[v] = dist[u] + w
                    heapq.heappush(pq, (dist[v], v))

        return dist
```

---

## Специальные представления

### Сжатый список смежности (CSR)

Для очень больших графов, оптимизация памяти и cache.

```python
class CSRGraph:
    """
    Compressed Sparse Row формат.
    Три массива:
    - values: веса рёбер
    - col_idx: индексы соседей
    - row_ptr: начало списка соседей каждой вершины
    """
    def __init__(self, adj_list, n):
        self.n = n
        self.values = []
        self.col_idx = []
        self.row_ptr = [0]

        for v in range(n):
            for neighbor, weight in adj_list.get(v, []):
                self.values.append(weight)
                self.col_idx.append(neighbor)
            self.row_ptr.append(len(self.col_idx))

    def get_neighbors(self, v):
        """Возвращает соседей вершины v"""
        start = self.row_ptr[v]
        end = self.row_ptr[v + 1]
        return list(zip(
            self.col_idx[start:end],
            self.values[start:end]
        ))

# Преимущества:
# - Лучшая локальность кэша
# - Меньше накладных расходов памяти
# - Быстрый обход
```

---

## Типичные ошибки

### 1. Добавление ребра только в одну сторону

```python
# НЕПРАВИЛЬНО для неориентированного графа
adj[u].append(v)
# Забыли adj[v].append(u)!

# ПРАВИЛЬНО
adj[u].append(v)
adj[v].append(u)
```

### 2. Дублирование рёбер

```python
# НЕПРАВИЛЬНО — может добавить ребро несколько раз
def add_edge(adj, u, v):
    adj[u].append(v)
    adj[v].append(u)

# ПРАВИЛЬНО — проверяем на дубликат
def add_edge_safe(adj, u, v):
    if v not in adj[u]:
        adj[u].append(v)
        adj[v].append(u)

# Или используем set
adj = defaultdict(set)
adj[u].add(v)
adj[v].add(u)
```

### 3. Неправильная индексация матрицы

```python
# Для ориентированного графа:
# matrix[from][to] = 1  (не наоборот!)

# Дуга A → B:
matrix[A][B] = 1  # ПРАВИЛЬНО
matrix[B][A] = 1  # НЕПРАВИЛЬНО (это B → A)
```

---
[prev: 02-directed-undirected](./02-directed-undirected.md) | [next: 04-weighted-graphs](./04-weighted-graphs.md)