# Ориентированные и неориентированные графы

[prev: 01-graph-basics](./01-graph-basics.md) | [next: 03-representations](./03-representations.md)
---

## Определение

### Неориентированный граф (Undirected Graph)

Рёбра не имеют направления. Если есть ребро (A, B), то можно пройти как от A к B, так и от B к A.

```
Неориентированный граф:

    A ─────── B
    │         │
    │         │
    D ─────── C

Ребро (A, B) = (B, A)
```

### Ориентированный граф (Directed Graph / Digraph)

Рёбра (дуги) имеют направление. Дуга (A, B) означает связь только от A к B.

```
Ориентированный граф:

    A ──────► B
    │         │
    ▼         ▼
    D ◄────── C

Дуга (A, B) ≠ (B, A)
A → B существует, но B → A не существует
```

## Зачем нужно

### Неориентированные графы:
- **Социальные сети** — дружба (симметричная связь)
- **Компьютерные сети** — Ethernet соединения
- **Карты дорог** — двустороннее движение
- **Молекулярные связи** — химические структуры

### Ориентированные графы:
- **Веб-граф** — гиперссылки (A ссылается на B)
- **Twitter** — подписки (A следит за B)
- **Зависимости** — сборка проектов
- **Карты дорог** — односторонние улицы
- **Потоки данных** — workflows, pipelines

## Терминология

### Для неориентированных графов

```
    A ─── B
    │ \   │
    │   \ │
    D ─── C

- Степень (degree): количество рёбер
  deg(A) = 3, deg(B) = 2

- Сумма степеней = 2 × |E|
  (каждое ребро добавляет 1 к двум вершинам)
```

### Для ориентированных графов

```
    A ───► B
    │      │
    ▼      ▼
    D ◄─── C

- Входящая степень (in-degree): количество входящих дуг
  in(D) = 2 (из A и C)

- Исходящая степень (out-degree): количество исходящих дуг
  out(A) = 2 (к B и D)

- Сумма in-degree = сумма out-degree = |E|
```

## Представление в коде

### Неориентированный граф

```python
class UndirectedGraph:
    def __init__(self):
        self.adj = {}

    def add_vertex(self, v):
        if v not in self.adj:
            self.adj[v] = []

    def add_edge(self, u, v):
        """Добавляет ребро в обе стороны"""
        self.add_vertex(u)
        self.add_vertex(v)
        self.adj[u].append(v)
        self.adj[v].append(u)  # симметрично!

    def remove_edge(self, u, v):
        """Удаляет ребро из обеих сторон"""
        if u in self.adj and v in self.adj[u]:
            self.adj[u].remove(v)
            self.adj[v].remove(u)

    def degree(self, v):
        """Степень вершины"""
        return len(self.adj.get(v, []))

    def has_edge(self, u, v):
        return v in self.adj.get(u, [])
```

### Ориентированный граф

```python
class DirectedGraph:
    def __init__(self):
        self.adj = {}      # исходящие дуги
        self.reverse = {}  # входящие дуги (для удобства)

    def add_vertex(self, v):
        if v not in self.adj:
            self.adj[v] = []
            self.reverse[v] = []

    def add_edge(self, u, v):
        """Добавляет дугу u → v"""
        self.add_vertex(u)
        self.add_vertex(v)
        self.adj[u].append(v)      # исходящая для u
        self.reverse[v].append(u)  # входящая для v

    def remove_edge(self, u, v):
        if u in self.adj and v in self.adj[u]:
            self.adj[u].remove(v)
            self.reverse[v].remove(u)

    def out_degree(self, v):
        """Исходящая степень"""
        return len(self.adj.get(v, []))

    def in_degree(self, v):
        """Входящая степень"""
        return len(self.reverse.get(v, []))

    def get_successors(self, v):
        """Вершины, куда ведут дуги из v"""
        return self.adj.get(v, [])

    def get_predecessors(self, v):
        """Вершины, откуда ведут дуги в v"""
        return self.reverse.get(v, [])
```

## Матрица смежности

### Неориентированный граф

```
Граф:          Матрица (симметричная):
   A ─── B        A  B  C
   │     │     A [0  1  0]
   │     │     B [1  0  1]
   C ─────     C [0  1  0]

matrix[i][j] = matrix[j][i]
```

### Ориентированный граф

```
Граф:          Матрица (несимметричная):
   A ──► B        A  B  C
   │     │     A [0  1  0]
   ▼     ▼     B [0  0  1]
   C ◄────     C [0  0  0]

matrix[i][j] = 1 означает дугу i → j
```

## Алгоритмы

### Поиск цикла

#### В неориентированном графе

```python
def has_cycle_undirected(graph):
    """
    Проверяет наличие цикла в неориентированном графе.
    Используем DFS, проверяем не возвращаемся ли к предку.
    """
    visited = set()

    def dfs(v, parent):
        visited.add(v)

        for neighbor in graph[v]:
            if neighbor not in visited:
                if dfs(neighbor, v):
                    return True
            elif neighbor != parent:  # нашли цикл!
                return True

        return False

    for vertex in graph:
        if vertex not in visited:
            if dfs(vertex, None):
                return True

    return False
```

#### В ориентированном графе

```python
def has_cycle_directed(graph):
    """
    Проверяет наличие цикла в ориентированном графе.
    Три состояния: не посещён, в процессе, завершён.
    """
    WHITE, GRAY, BLACK = 0, 1, 2
    color = {v: WHITE for v in graph}

    def dfs(v):
        color[v] = GRAY  # в процессе обработки

        for neighbor in graph[v]:
            if color[neighbor] == GRAY:  # цикл!
                return True
            if color[neighbor] == WHITE:
                if dfs(neighbor):
                    return True

        color[v] = BLACK  # завершено
        return False

    for vertex in graph:
        if color[vertex] == WHITE:
            if dfs(vertex):
                return True

    return False
```

### Топологическая сортировка (только для DAG)

```python
from collections import deque

def topological_sort_kahn(graph, vertices):
    """
    Алгоритм Кана — BFS версия топологической сортировки.
    Работает только для DAG (Directed Acyclic Graph).
    """
    in_degree = {v: 0 for v in vertices}

    for v in graph:
        for neighbor in graph[v]:
            in_degree[neighbor] += 1

    # Вершины с in_degree = 0
    queue = deque([v for v in vertices if in_degree[v] == 0])
    result = []

    while queue:
        v = queue.popleft()
        result.append(v)

        for neighbor in graph[v]:
            in_degree[neighbor] -= 1
            if in_degree[neighbor] == 0:
                queue.append(neighbor)

    if len(result) != len(vertices):
        raise ValueError("Graph has a cycle!")

    return result

def topological_sort_dfs(graph, vertices):
    """
    DFS версия топологической сортировки.
    """
    visited = set()
    stack = []

    def dfs(v):
        visited.add(v)
        for neighbor in graph.get(v, []):
            if neighbor not in visited:
                dfs(neighbor)
        stack.append(v)

    for vertex in vertices:
        if vertex not in visited:
            dfs(vertex)

    return stack[::-1]  # реверс
```

```
DAG:                  Топологический порядок:
   A → B → D          A, B, C, D или A, C, B, D
   ↓   ↓
   C → ─

Любой допустимый порядок: если A → B, то A перед B.
```

### Сильно связные компоненты (для ориентированных)

```python
def kosaraju_scc(graph, vertices):
    """
    Алгоритм Косарайю для поиска SCC.
    SCC — максимальный подграф, где любая вершина достижима из любой.
    """
    # Шаг 1: DFS и запись в стек по времени завершения
    visited = set()
    stack = []

    def dfs1(v):
        visited.add(v)
        for neighbor in graph.get(v, []):
            if neighbor not in visited:
                dfs1(neighbor)
        stack.append(v)

    for v in vertices:
        if v not in visited:
            dfs1(v)

    # Шаг 2: Построение обратного графа
    reverse_graph = {v: [] for v in vertices}
    for v in graph:
        for neighbor in graph[v]:
            reverse_graph[neighbor].append(v)

    # Шаг 3: DFS на обратном графе в порядке стека
    visited.clear()
    components = []

    def dfs2(v, component):
        visited.add(v)
        component.append(v)
        for neighbor in reverse_graph.get(v, []):
            if neighbor not in visited:
                dfs2(neighbor, component)

    while stack:
        v = stack.pop()
        if v not in visited:
            component = []
            dfs2(v, component)
            components.append(component)

    return components
```

```
Граф:                 SCC:
  A → B → C           {A, B, E}, {C, D, F}
  ↑   ↓   ↓
  E ← D → F

A → B → D → E → A — цикл, все в одной SCC
```

## Примеры с разбором

### Пример 1: Проверка достижимости

```python
def is_reachable(graph, start, end):
    """Проверяет, достижим ли end из start"""
    if start == end:
        return True

    visited = set([start])
    queue = [start]

    while queue:
        v = queue.pop(0)
        for neighbor in graph.get(v, []):
            if neighbor == end:
                return True
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)

    return False
```

### Пример 2: Порядок компиляции (зависимости)

```python
def build_order(projects, dependencies):
    """
    projects: список проектов
    dependencies: [(a, b)] означает b зависит от a

    Возвращает порядок сборки.
    """
    graph = {p: [] for p in projects}

    for a, b in dependencies:
        graph[a].append(b)  # a должен быть собран до b

    return topological_sort_kahn(graph, projects)

# Пример:
# projects = ['a', 'b', 'c', 'd', 'e', 'f']
# dependencies = [('a', 'd'), ('f', 'b'), ('b', 'd'), ('f', 'a'), ('d', 'c')]
# Результат: ['e', 'f', 'a', 'b', 'd', 'c'] или похожий
```

### Пример 3: Обнаружение deadlock (циклических зависимостей)

```python
def has_deadlock(resources, processes, allocations, requests):
    """
    Проверяет наличие deadlock в системе.

    Строим граф ожидания:
    - Процесс → Ресурс: процесс запрашивает ресурс
    - Ресурс → Процесс: ресурс выделен процессу

    Deadlock = цикл в графе.
    """
    graph = {}

    # Строим граф
    for process, resource in requests:
        if process not in graph:
            graph[process] = []
        graph[process].append(resource)

    for resource, process in allocations:
        if resource not in graph:
            graph[resource] = []
        graph[resource].append(process)

    return has_cycle_directed(graph)
```

## Сравнение

| Аспект | Неориентированный | Ориентированный |
|--------|-------------------|-----------------|
| Ребро | Симметричное | Направленное |
| Степень | degree | in-degree, out-degree |
| Связность | Связный граф | Сильная/слабая связность |
| Цикл | Простой цикл | Направленный цикл |
| Топосортировка | Не применимо | Для DAG |
| Пример | Дружба | Подписки |

## Типичные ошибки

### 1. Забытое добавление обратного ребра

```python
# НЕПРАВИЛЬНО для неориентированного
def add_edge(graph, u, v):
    graph[u].append(v)
    # Забыли graph[v].append(u)!

# ПРАВИЛЬНО
def add_edge(graph, u, v):
    graph[u].append(v)
    graph[v].append(u)
```

### 2. Неправильная проверка цикла в неориентированном графе

```python
# НЕПРАВИЛЬНО — любое ребро выглядит как цикл
def dfs_wrong(v, visited, graph):
    visited.add(v)
    for n in graph[v]:
        if n in visited:
            return True  # Это может быть просто предок!
        if dfs_wrong(n, visited, graph):
            return True

# ПРАВИЛЬНО — исключаем родителя
def dfs_correct(v, parent, visited, graph):
    visited.add(v)
    for n in graph[v]:
        if n not in visited:
            if dfs_correct(n, v, visited, graph):
                return True
        elif n != parent:  # цикл только если это не родитель
            return True
    return False
```

### 3. Топосортировка графа с циклом

```python
# Всегда проверяйте на цикл!
def topological_sort_safe(graph, vertices):
    if has_cycle_directed(graph):
        raise ValueError("Cannot topologically sort a cyclic graph")
    return topological_sort_dfs(graph, vertices)
```

## Когда что использовать

**Неориентированный граф:**
- Связь симметрична
- Дружба, родственные связи
- Физические соединения

**Ориентированный граф:**
- Связь имеет направление
- Зависимости, потоки данных
- Ссылки, подписки
- Порядок выполнения

---
[prev: 01-graph-basics](./01-graph-basics.md) | [next: 03-representations](./03-representations.md)