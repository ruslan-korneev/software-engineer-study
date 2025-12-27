# Поиск в глубину (DFS — Depth-First Search)

## Определение

**Поиск в глубину (DFS)** — это алгоритм обхода или поиска в графе/дереве, который исследует ветвь как можно глубже, прежде чем возвращаться и исследовать другие ветви. Использует стек (явный или неявный через рекурсию).

## Зачем нужен

### Области применения:
- **Обнаружение циклов** в графе
- **Топологическая сортировка** (DAG)
- **Поиск компонент связности**
- **Поиск пути в лабиринте**
- **Проверка двудольности графа**
- **Решение задач с возвратом (backtracking)**
- **Нахождение мостов и точек сочленения**

### Преимущества:
- Меньше памяти, чем BFS для глубоких графов
- Естественен для рекурсивных задач
- Находит путь (не обязательно кратчайший)

## Как работает

### Визуализация процесса

```
Граф:
    0 ─── 1
    │     │
    │     │
    2 ─── 3 ─── 4

Начинаем с 0:

Шаг 1: visit 0, stack = [0]
       0*─── 1
       │     │
       2 ─── 3 ─── 4

Шаг 2: visit 1 (сосед 0), stack = [0, 1]
       0 ───*1
       │     │
       2 ─── 3 ─── 4

Шаг 3: visit 3 (сосед 1), stack = [0, 1, 3]
       0 ─── 1
       │     │
       2 ───*3 ─── 4

Шаг 4: visit 2 (сосед 3), stack = [0, 1, 3, 2]
       0 ─── 1
       │     │
      *2 ─── 3 ─── 4

Шаг 5: 2's neighbors (0, 3) visited, backtrack
       stack = [0, 1, 3]

Шаг 6: visit 4 (сосед 3), stack = [0, 1, 3, 4]
       0 ─── 1
       │     │
       2 ─── 3 ───*4

Шаг 7: backtrack до конца

Порядок посещения: 0 → 1 → 3 → 2 → 4
```

### Сравнение DFS и BFS

```
        0
       /|\
      1 2 3
     /|   |
    4 5   6

DFS: 0 → 1 → 4 → 5 → 2 → 3 → 6  (вглубь)
BFS: 0 → 1 → 2 → 3 → 4 → 5 → 6  (по уровням)
```

## Псевдокод

```
function DFS(graph, start):
    visited = set()
    stack = [start]

    while stack is not empty:
        vertex = stack.pop()

        if vertex not in visited:
            visited.add(vertex)
            process(vertex)

            for neighbor in graph[vertex]:
                if neighbor not in visited:
                    stack.push(neighbor)

# Рекурсивная версия
function DFS_recursive(graph, vertex, visited):
    visited.add(vertex)
    process(vertex)

    for neighbor in graph[vertex]:
        if neighbor not in visited:
            DFS_recursive(graph, neighbor, visited)
```

### Реализация на Python — итеративная

```python
def dfs_iterative(graph, start):
    """
    Итеративный DFS.

    Args:
        graph: словарь смежности {vertex: [neighbors]}
        start: начальная вершина

    Returns:
        список вершин в порядке посещения
    """
    visited = set()
    stack = [start]
    result = []

    while stack:
        vertex = stack.pop()

        if vertex not in visited:
            visited.add(vertex)
            result.append(vertex)

            # Добавляем соседей в обратном порядке для правильного обхода
            for neighbor in reversed(graph[vertex]):
                if neighbor not in visited:
                    stack.append(neighbor)

    return result
```

### Реализация на Python — рекурсивная

```python
def dfs_recursive(graph, start, visited=None):
    """Рекурсивный DFS."""
    if visited is None:
        visited = set()

    visited.add(start)
    result = [start]

    for neighbor in graph[start]:
        if neighbor not in visited:
            result.extend(dfs_recursive(graph, neighbor, visited))

    return result
```

### Обход всех компонент связности

```python
def dfs_all_components(graph):
    """Обход всех компонент графа."""
    visited = set()
    components = []

    for vertex in graph:
        if vertex not in visited:
            component = []
            stack = [vertex]

            while stack:
                v = stack.pop()
                if v not in visited:
                    visited.add(v)
                    component.append(v)
                    for neighbor in graph[v]:
                        if neighbor not in visited:
                            stack.append(neighbor)

            components.append(component)

    return components
```

### DFS с временными метками

```python
def dfs_with_timestamps(graph, start):
    """DFS с временем входа и выхода."""
    visited = set()
    entry_time = {}
    exit_time = {}
    time = [0]  # Используем список для изменения в замыкании

    def dfs(vertex):
        visited.add(vertex)
        time[0] += 1
        entry_time[vertex] = time[0]

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                dfs(neighbor)

        time[0] += 1
        exit_time[vertex] = time[0]

    dfs(start)
    return entry_time, exit_time
```

## Анализ сложности

| Представление | Время | Память |
|---------------|-------|--------|
| Список смежности | O(V + E) | O(V) |
| Матрица смежности | O(V^2) | O(V) |

Где:
- V — количество вершин
- E — количество рёбер

## Пример с пошаговым разбором

### Обнаружение цикла в ориентированном графе

```python
def has_cycle_directed(graph):
    """
    Обнаружение цикла в ориентированном графе.
    Использует три состояния: WHITE (не посещён), GRAY (в обработке), BLACK (завершён).
    """
    WHITE, GRAY, BLACK = 0, 1, 2
    color = {v: WHITE for v in graph}

    def dfs(vertex):
        color[vertex] = GRAY

        for neighbor in graph[vertex]:
            if color[neighbor] == GRAY:
                return True  # Нашли обратное ребро — цикл!
            if color[neighbor] == WHITE and dfs(neighbor):
                return True

        color[vertex] = BLACK
        return False

    for vertex in graph:
        if color[vertex] == WHITE:
            if dfs(vertex):
                return True

    return False

# Пример
graph = {
    0: [1],
    1: [2],
    2: [0]  # Цикл: 0 → 1 → 2 → 0
}
print(has_cycle_directed(graph))  # True
```

### Топологическая сортировка

```python
def topological_sort(graph):
    """
    Топологическая сортировка DAG через DFS.
    Возвращает вершины в порядке зависимостей.
    """
    visited = set()
    stack = []

    def dfs(vertex):
        visited.add(vertex)

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                dfs(neighbor)

        stack.append(vertex)  # Добавляем после обработки всех потомков

    for vertex in graph:
        if vertex not in visited:
            dfs(vertex)

    return stack[::-1]  # Разворачиваем

# Пример: курсы с зависимостями
courses = {
    'A': ['B', 'C'],  # A зависит от B и C
    'B': ['D'],
    'C': ['D'],
    'D': []
}
print(topological_sort(courses))  # ['D', 'B', 'C', 'A'] или другой валидный порядок
```

### Поиск пути

```python
def find_path_dfs(graph, start, end):
    """Найти путь между двумя вершинами."""
    visited = set()
    path = []

    def dfs(vertex):
        if vertex == end:
            path.append(vertex)
            return True

        visited.add(vertex)
        path.append(vertex)

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                if dfs(neighbor):
                    return True

        path.pop()  # Backtrack
        return False

    if dfs(start):
        return path
    return None

# Пример
graph = {
    'A': ['B', 'C'],
    'B': ['D', 'E'],
    'C': ['F'],
    'D': [],
    'E': ['F'],
    'F': []
}
print(find_path_dfs(graph, 'A', 'F'))  # ['A', 'B', 'E', 'F'] или ['A', 'C', 'F']
```

## Типичные ошибки и Edge Cases

### 1. Бесконечный цикл без проверки visited

```python
# ОШИБКА: бесконечный цикл в графе с циклами
def wrong_dfs(graph, start):
    stack = [start]
    while stack:
        v = stack.pop()
        for n in graph[v]:
            stack.append(n)  # Добавляем даже посещённые!

# ПРАВИЛЬНО
def correct_dfs(graph, start):
    visited = set()
    stack = [start]
    while stack:
        v = stack.pop()
        if v not in visited:
            visited.add(v)
            for n in graph[v]:
                if n not in visited:
                    stack.append(n)
```

### 2. Пустой граф

```python
def dfs_safe(graph, start):
    if not graph or start not in graph:
        return []
    # ...
```

### 3. Несвязный граф

```python
# Для обхода всего графа нужно запустить DFS из каждой непосещённой вершины
def dfs_full(graph):
    visited = set()
    result = []

    for vertex in graph:
        if vertex not in visited:
            # DFS из этой вершины
            stack = [vertex]
            while stack:
                v = stack.pop()
                if v not in visited:
                    visited.add(v)
                    result.append(v)
                    for n in graph[v]:
                        stack.append(n)

    return result
```

### 4. Stack Overflow при глубокой рекурсии

```python
import sys
sys.setrecursionlimit(10000)  # Увеличить лимит (не рекомендуется)

# Лучше использовать итеративную версию для глубоких графов
```

### 5. Порядок обхода соседей

```python
# Итеративный DFS обходит соседей в обратном порядке
# Для правильного порядка — добавляем в reversed

def dfs_correct_order(graph, start):
    visited = set()
    stack = [start]
    result = []

    while stack:
        v = stack.pop()
        if v not in visited:
            visited.add(v)
            result.append(v)
            for n in reversed(graph[v]):  # Обратный порядок!
                stack.append(n)

    return result
```

### 6. DFS на матрице (лабиринт)

```python
def dfs_maze(maze, start, end):
    """DFS на 2D матрице (0 — проход, 1 — стена)."""
    rows, cols = len(maze), len(maze[0])
    visited = set()
    directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]

    def is_valid(r, c):
        return (0 <= r < rows and 0 <= c < cols and
                maze[r][c] == 0 and (r, c) not in visited)

    def dfs(r, c):
        if (r, c) == end:
            return True

        visited.add((r, c))

        for dr, dc in directions:
            nr, nc = r + dr, c + dc
            if is_valid(nr, nc) and dfs(nr, nc):
                return True

        return False

    return dfs(start[0], start[1])

# Пример
maze = [
    [0, 0, 1, 0],
    [1, 0, 1, 0],
    [0, 0, 0, 0]
]
print(dfs_maze(maze, (0, 0), (2, 3)))  # True
```

### Рекомендации:
1. DFS использует меньше памяти для глубоких, узких графов
2. Используйте рекурсивную версию для простоты, итеративную — для больших графов
3. Для обнаружения циклов используйте три цвета (white/gray/black)
4. DFS не находит кратчайший путь — для этого используйте BFS
5. Временные метки полезны для топологической сортировки и поиска мостов
