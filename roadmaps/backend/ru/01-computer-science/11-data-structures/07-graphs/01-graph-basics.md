# Основы графов (Graph Basics)

[prev: 06-trie](../06-trees/06-trie.md) | [next: 02-directed-undirected](./02-directed-undirected.md)
---

## Определение

**Граф** — это структура данных, состоящая из **вершин (узлов)** и **рёбер (связей)** между ними. Граф описывается как G = (V, E), где V — множество вершин, E — множество рёбер.

```
Граф с 5 вершинами и 6 рёбрами:

    A ─────── B
    │ \       │
    │   \     │
    │     \   │
    D ─────── C
     \       /
       \   /
         E
```

## Зачем нужно

### Практическое применение:
- **Социальные сети** — друзья, подписчики
- **Карты и навигация** — дороги, маршруты
- **Интернет** — веб-страницы и ссылки
- **Компиляторы** — зависимости, порядок выполнения
- **Рекомендательные системы** — связи между товарами/пользователями
- **Биоинформатика** — молекулярные структуры

## Терминология

```
         A ────────── B
        /│\          /│
       / │ \        / │
      /  │  \      /  │
     D   │   \    /   │
      \  │    \  /    │
       \ │     \/     │
        \│     /\     │
         E────/──\────C
              F

Термины:
- Вершина (Vertex/Node): A, B, C, D, E, F
- Ребро (Edge): (A,B), (A,D), (A,E), ...
- Смежные вершины (Adjacent): A и B смежны (есть ребро)
- Степень вершины (Degree): количество рёбер
- Путь (Path): последовательность A → B → C
- Цикл (Cycle): путь, начинающийся и заканчивающийся в одной вершине
- Связный граф: есть путь между любыми двумя вершинами
```

### Основные термины

| Термин | Описание | Пример |
|--------|----------|--------|
| **Вершина (Vertex)** | Узел графа | A, B, C |
| **Ребро (Edge)** | Связь между вершинами | (A, B) |
| **Смежность** | Две вершины соединены ребром | A смежна с B |
| **Степень** | Количество рёбер вершины | deg(A) = 3 |
| **Путь** | Последовательность смежных вершин | A → B → C |
| **Длина пути** | Количество рёбер в пути | 2 |
| **Цикл** | Путь из вершины в себя | A → B → C → A |
| **Связность** | Существование пути между вершинами | |
| **Компонента связности** | Максимальный связный подграф | |

## Типы графов

### По направленности рёбер

```
Неориентированный граф:     Ориентированный граф (диграф):
   A ─── B                     A ──► B
   │     │                     │     │
   │     │                     ▼     ▼
   D ─── C                     D ◄── C

Ребро (A,B) = (B,A)           Дуга (A,B) ≠ (B,A)
```

### По наличию весов

```
Невзвешенный:          Взвешенный:
   A ─── B                A ──5── B
   │     │                │       │
   │     │                3       2
   │     │                │       │
   D ─── C                D ──4── C
```

### По связности

```
Связный:               Несвязный:
   A ─── B                A ─── B    E
   │     │                          / \
   │     │                         F   G
   D ─── C

                       2 компоненты связности
```

### Специальные виды графов

```
Дерево:                 Полный граф K₄:
   A                       A
  /|\                     /|\
 B C D                   B-+-C
                          \|/
                           D

Ациклический,           Все вершины
связный                 соединены

Двудольный граф:       DAG (Directed Acyclic Graph):
   A   B                   A → B
   |\ /|                   ↓   ↓
   | X |                   C → D
   |/ \|
   C   D                Ориентированный,
                        без циклов
```

## Представления графа

### 1. Матрица смежности (Adjacency Matrix)

```
Граф:                  Матрица:
   A ─── B               A  B  C  D
   │     │            A [0  1  0  1]
   │     │            B [1  0  1  0]
   D ─── C            C [0  1  0  1]
                      D [1  0  1  0]

matrix[i][j] = 1, если есть ребро (i, j)
```

```python
class GraphMatrix:
    def __init__(self, num_vertices):
        self.num_vertices = num_vertices
        self.matrix = [[0] * num_vertices for _ in range(num_vertices)]

    def add_edge(self, u, v, weight=1):
        self.matrix[u][v] = weight
        self.matrix[v][u] = weight  # для неориентированного

    def has_edge(self, u, v):
        return self.matrix[u][v] != 0

    def get_neighbors(self, v):
        return [i for i in range(self.num_vertices) if self.matrix[v][i] != 0]
```

**Преимущества:**
- Проверка ребра O(1)
- Простая реализация

**Недостатки:**
- Память O(V²)
- Получение соседей O(V)

### 2. Список смежности (Adjacency List)

```
Граф:                  Список:
   A ─── B             A: [B, D]
   │     │             B: [A, C]
   │     │             C: [B, D]
   D ─── C             D: [A, C]
```

```python
from collections import defaultdict

class GraphList:
    def __init__(self):
        self.graph = defaultdict(list)

    def add_edge(self, u, v, weight=1):
        self.graph[u].append((v, weight))
        self.graph[v].append((u, weight))  # для неориентированного

    def has_edge(self, u, v):
        return any(neighbor == v for neighbor, _ in self.graph[u])

    def get_neighbors(self, v):
        return [neighbor for neighbor, _ in self.graph[v]]
```

**Преимущества:**
- Память O(V + E)
- Получение соседей O(degree)
- Эффективен для разреженных графов

**Недостатки:**
- Проверка ребра O(degree)

### 3. Список рёбер (Edge List)

```python
class GraphEdgeList:
    def __init__(self):
        self.edges = []
        self.vertices = set()

    def add_edge(self, u, v, weight=1):
        self.edges.append((u, v, weight))
        self.vertices.add(u)
        self.vertices.add(v)

    def get_edges(self):
        return self.edges
```

**Используется для:**
- Алгоритма Крускала
- Алгоритма Беллмана-Форда

## Сравнение представлений

| Операция | Матрица | Список |
|----------|---------|--------|
| Память | O(V²) | O(V + E) |
| Проверка ребра | O(1) | O(degree) |
| Все соседи | O(V) | O(degree) |
| Все рёбра | O(V²) | O(E) |
| Добавление ребра | O(1) | O(1) |
| Удаление ребра | O(1) | O(degree) |

**Когда использовать матрицу:**
- Плотный граф (E ≈ V²)
- Частая проверка наличия рёбер

**Когда использовать список:**
- Разреженный граф (E << V²)
- Итерация по соседям

## Обходы графа

### BFS (Breadth-First Search)

```python
from collections import deque

def bfs(graph, start):
    """
    Обход в ширину.
    Находит кратчайшие пути в невзвешенном графе.
    """
    visited = set([start])
    queue = deque([start])
    order = []

    while queue:
        vertex = queue.popleft()
        order.append(vertex)

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

    return order
```

```
Граф:      BFS от A:
  A─B
  |X|      Очередь: A → B,C → D
  C─D
           Порядок: A, B, C, D (по уровням)
```

### DFS (Depth-First Search)

```python
def dfs(graph, start):
    """
    Обход в глубину.
    Используется для поиска циклов, топологической сортировки и т.д.
    """
    visited = set()
    order = []

    def explore(vertex):
        visited.add(vertex)
        order.append(vertex)

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                explore(neighbor)

    explore(start)
    return order

# Итеративная версия
def dfs_iterative(graph, start):
    visited = set()
    stack = [start]
    order = []

    while stack:
        vertex = stack.pop()
        if vertex in visited:
            continue

        visited.add(vertex)
        order.append(vertex)

        for neighbor in reversed(graph[vertex]):
            if neighbor not in visited:
                stack.append(neighbor)

    return order
```

```
Граф:      DFS от A:
  A─B
  |X|      Стек: A → B,C → D (идём вглубь)
  C─D
           Порядок: A, B, D, C (в глубину)
```

## Примеры с разбором

### Пример 1: Поиск пути между вершинами

```python
def find_path(graph, start, end):
    """Находит путь между двумя вершинами (BFS)"""
    if start == end:
        return [start]

    visited = {start}
    queue = deque([(start, [start])])

    while queue:
        vertex, path = queue.popleft()

        for neighbor in graph[vertex]:
            if neighbor == end:
                return path + [neighbor]

            if neighbor not in visited:
                visited.add(neighbor)
                queue.append((neighbor, path + [neighbor]))

    return None  # путь не найден
```

### Пример 2: Подсчёт компонент связности

```python
def count_components(graph, vertices):
    """Считает количество компонент связности"""
    visited = set()
    components = 0

    def dfs(v):
        visited.add(v)
        for neighbor in graph[v]:
            if neighbor not in visited:
                dfs(neighbor)

    for vertex in vertices:
        if vertex not in visited:
            dfs(vertex)
            components += 1

    return components
```

### Пример 3: Проверка двудольности

```python
def is_bipartite(graph, vertices):
    """
    Проверяет, является ли граф двудольным.
    Двудольный = можно раскрасить в 2 цвета без конфликтов.
    """
    color = {}

    def bfs(start):
        queue = deque([start])
        color[start] = 0

        while queue:
            vertex = queue.popleft()

            for neighbor in graph[vertex]:
                if neighbor not in color:
                    color[neighbor] = 1 - color[vertex]
                    queue.append(neighbor)
                elif color[neighbor] == color[vertex]:
                    return False

        return True

    for vertex in vertices:
        if vertex not in color:
            if not bfs(vertex):
                return False

    return True
```

## Анализ сложности

| Алгоритм | Время | Память |
|----------|-------|--------|
| BFS | O(V + E) | O(V) |
| DFS | O(V + E) | O(V) |

Где V — количество вершин, E — количество рёбер.

## Типичные ошибки

### 1. Забытая проверка visited

```python
# НЕПРАВИЛЬНО — бесконечный цикл
def dfs_wrong(graph, start):
    for neighbor in graph[start]:
        dfs_wrong(graph, neighbor)  # может вернуться!

# ПРАВИЛЬНО
def dfs_correct(graph, start, visited=None):
    if visited is None:
        visited = set()

    visited.add(start)
    for neighbor in graph[start]:
        if neighbor not in visited:
            dfs_correct(graph, neighbor, visited)
```

### 2. BFS без отметки при добавлении

```python
# НЕПРАВИЛЬНО — одна вершина может добавиться несколько раз
while queue:
    v = queue.popleft()
    visited.add(v)  # поздно!
    for n in graph[v]:
        if n not in visited:
            queue.append(n)

# ПРАВИЛЬНО — отмечаем сразу при добавлении
while queue:
    v = queue.popleft()
    for n in graph[v]:
        if n not in visited:
            visited.add(n)  # сразу!
            queue.append(n)
```

### 3. Путаница с направленными рёбрами

```python
# Для неориентированного графа добавляем в обе стороны
graph[u].append(v)
graph[v].append(u)

# Для ориентированного — только в одну
graph[u].append(v)  # дуга u → v
```

## Когда использовать граф

**Используй граф, когда:**
- Данные имеют связи "многие ко многим"
- Нужен поиск путей
- Есть зависимости между объектами
- Нужно моделировать сети

**НЕ используй граф, когда:**
- Достаточно простой иерархии (дерево)
- Нет связей между объектами
- Достаточно линейной структуры (список)

---
[prev: 06-trie](../06-trees/06-trie.md) | [next: 02-directed-undirected](./02-directed-undirected.md)