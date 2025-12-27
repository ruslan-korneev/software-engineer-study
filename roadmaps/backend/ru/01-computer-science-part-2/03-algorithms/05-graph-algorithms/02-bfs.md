# Поиск в ширину (BFS — Breadth-First Search)

## Определение

**Поиск в ширину (BFS)** — это алгоритм обхода графа, который исследует все вершины на текущем уровне расстояния от начальной вершины, прежде чем переходить к вершинам следующего уровня. Использует очередь (FIFO).

## Зачем нужен

### Области применения:
- **Кратчайший путь в невзвешенном графе** — главное применение
- **Нахождение кратчайшего расстояния**
- **Проверка двудольности графа**
- **Поиск компонент связности**
- **Web-краулеры** — обход страниц
- **Социальные сети** — друзья друзей
- **GPS-навигация** — кратчайший маршрут

### Преимущества:
- Гарантирует кратчайший путь в невзвешенном графе
- Находит все вершины на расстоянии k
- Систематичный обход по уровням

## Как работает

### Визуализация процесса

```
Граф:
    0 ─── 1 ─── 2
    │           │
    3 ─── 4 ─── 5

Начинаем с 0, ищем путь до 5:

Уровень 0: [0]
           0*─── 1 ─── 2
           │           │
           3 ─── 4 ─── 5

Уровень 1: [1, 3]  (соседи 0)
           0 ───*1 ─── 2
           │           │
          *3 ─── 4 ─── 5

Уровень 2: [2, 4]  (соседи 1 и 3)
           0 ─── 1 ───*2
           │           │
           3 ───*4 ─── 5

Уровень 3: [5]  (соседи 2 и 4)
           0 ─── 1 ─── 2
           │           │
           3 ─── 4 ───*5

Кратчайший путь: 0 → 1 → 2 → 5 (или 0 → 3 → 4 → 5), длина = 3
```

### Очередь в действии

```
Шаг 0: queue = [0]
       distances = {0: 0}

Шаг 1: dequeue 0
       enqueue 1, 3
       queue = [1, 3]
       distances = {0: 0, 1: 1, 3: 1}

Шаг 2: dequeue 1
       enqueue 2
       queue = [3, 2]
       distances = {0: 0, 1: 1, 3: 1, 2: 2}

Шаг 3: dequeue 3
       enqueue 4
       queue = [2, 4]
       distances = {0: 0, 1: 1, 3: 1, 2: 2, 4: 2}

Шаг 4: dequeue 2
       enqueue 5
       queue = [4, 5]
       distances = {0: 0, 1: 1, 3: 1, 2: 2, 4: 2, 5: 3}

Шаг 5: dequeue 4 (5 уже в очереди)
       queue = [5]

Шаг 6: dequeue 5 — нашли!
```

## Псевдокод

```
function BFS(graph, start):
    visited = set([start])
    queue = [start]
    distances = {start: 0}

    while queue is not empty:
        vertex = queue.dequeue()

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.enqueue(neighbor)
                distances[neighbor] = distances[vertex] + 1

    return distances
```

### Базовая реализация на Python

```python
from collections import deque

def bfs(graph, start):
    """
    BFS обход графа.

    Args:
        graph: словарь смежности {vertex: [neighbors]}
        start: начальная вершина

    Returns:
        словарь расстояний от start до каждой вершины
    """
    visited = {start}
    queue = deque([start])
    distances = {start: 0}

    while queue:
        vertex = queue.popleft()

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
                distances[neighbor] = distances[vertex] + 1

    return distances
```

### Поиск кратчайшего пути

```python
def shortest_path(graph, start, end):
    """Найти кратчайший путь между двумя вершинами."""
    if start == end:
        return [start]

    visited = {start}
    queue = deque([start])
    parent = {start: None}

    while queue:
        vertex = queue.popleft()

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                visited.add(neighbor)
                parent[neighbor] = vertex
                queue.append(neighbor)

                if neighbor == end:
                    # Восстанавливаем путь
                    path = []
                    current = end
                    while current is not None:
                        path.append(current)
                        current = parent[current]
                    return path[::-1]

    return None  # Путь не найден
```

### BFS с подсчётом уровней

```python
def bfs_with_levels(graph, start):
    """BFS с группировкой по уровням."""
    visited = {start}
    queue = deque([start])
    levels = [[start]]

    while queue:
        level_size = len(queue)
        current_level = []

        for _ in range(level_size):
            vertex = queue.popleft()

            for neighbor in graph[vertex]:
                if neighbor not in visited:
                    visited.add(neighbor)
                    queue.append(neighbor)
                    current_level.append(neighbor)

        if current_level:
            levels.append(current_level)

    return levels
```

### Многоисходный BFS (Multi-source BFS)

```python
def multi_source_bfs(graph, sources):
    """BFS из нескольких начальных вершин одновременно."""
    visited = set(sources)
    queue = deque(sources)
    distances = {s: 0 for s in sources}

    while queue:
        vertex = queue.popleft()

        for neighbor in graph[vertex]:
            if neighbor not in visited:
                visited.add(neighbor)
                queue.append(neighbor)
                distances[neighbor] = distances[vertex] + 1

    return distances
```

### 0-1 BFS (для рёбер с весами 0 и 1)

```python
def bfs_01(graph, start):
    """
    BFS для графа с рёбрами веса 0 или 1.
    graph[v] = [(neighbor, weight), ...]
    """
    distances = {v: float('inf') for v in graph}
    distances[start] = 0
    deq = deque([start])

    while deq:
        vertex = deq.popleft()

        for neighbor, weight in graph[vertex]:
            new_dist = distances[vertex] + weight

            if new_dist < distances[neighbor]:
                distances[neighbor] = new_dist

                if weight == 0:
                    deq.appendleft(neighbor)  # В начало — приоритет
                else:
                    deq.append(neighbor)  # В конец

    return distances
```

## Анализ сложности

| Представление | Время | Память |
|---------------|-------|--------|
| Список смежности | O(V + E) | O(V) |
| Матрица смежности | O(V^2) | O(V) |

Где V — вершины, E — рёбра.

## Пример с пошаговым разбором

### Проверка двудольности графа

```python
def is_bipartite(graph):
    """
    Проверка двудольности графа через BFS.
    Двудольный = можно раскрасить в 2 цвета так,
    чтобы соседние вершины имели разный цвет.
    """
    color = {}

    for start in graph:
        if start in color:
            continue

        queue = deque([start])
        color[start] = 0

        while queue:
            vertex = queue.popleft()

            for neighbor in graph[vertex]:
                if neighbor not in color:
                    color[neighbor] = 1 - color[vertex]  # Другой цвет
                    queue.append(neighbor)
                elif color[neighbor] == color[vertex]:
                    return False  # Конфликт цветов

    return True

# Пример
graph = {
    0: [1, 3],
    1: [0, 2],
    2: [1, 3],
    3: [0, 2]
}
print(is_bipartite(graph))  # True — это цикл длины 4
```

### Поиск в лабиринте

```python
def maze_shortest_path(maze, start, end):
    """
    Кратчайший путь в лабиринте.
    0 — проход, 1 — стена.
    """
    rows, cols = len(maze), len(maze[0])
    directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]

    if maze[start[0]][start[1]] == 1 or maze[end[0]][end[1]] == 1:
        return -1

    queue = deque([start])
    visited = {start}
    distance = 0

    while queue:
        for _ in range(len(queue)):
            r, c = queue.popleft()

            if (r, c) == end:
                return distance

            for dr, dc in directions:
                nr, nc = r + dr, c + dc

                if (0 <= nr < rows and 0 <= nc < cols and
                    maze[nr][nc] == 0 and (nr, nc) not in visited):
                    visited.add((nr, nc))
                    queue.append((nr, nc))

        distance += 1

    return -1  # Путь не найден

# Пример
maze = [
    [0, 0, 1, 0, 0],
    [0, 0, 0, 0, 1],
    [1, 1, 0, 1, 0],
    [0, 0, 0, 0, 0]
]
print(maze_shortest_path(maze, (0, 0), (3, 4)))  # 7
```

### Распространение "заражения" (Rotting Oranges)

```python
def rotting_oranges(grid):
    """
    Сколько минут нужно, чтобы все апельсины сгнили?
    0 — пусто, 1 — свежий, 2 — гнилой.
    """
    rows, cols = len(grid), len(grid[0])
    queue = deque()
    fresh = 0

    # Находим все гнилые апельсины и считаем свежие
    for r in range(rows):
        for c in range(cols):
            if grid[r][c] == 2:
                queue.append((r, c))
            elif grid[r][c] == 1:
                fresh += 1

    if fresh == 0:
        return 0

    directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]
    minutes = 0

    while queue and fresh > 0:
        minutes += 1

        for _ in range(len(queue)):
            r, c = queue.popleft()

            for dr, dc in directions:
                nr, nc = r + dr, c + dc

                if (0 <= nr < rows and 0 <= nc < cols and
                    grid[nr][nc] == 1):
                    grid[nr][nc] = 2
                    fresh -= 1
                    queue.append((nr, nc))

    return minutes if fresh == 0 else -1
```

## Типичные ошибки и Edge Cases

### 1. Использование списка вместо deque

```python
# НЕЭФФЕКТИВНО: pop(0) — O(n)
queue = []
queue.append(start)
vertex = queue.pop(0)  # O(n)!

# ПРАВИЛЬНО: popleft() — O(1)
from collections import deque
queue = deque([start])
vertex = queue.popleft()  # O(1)
```

### 2. Добавление в visited после извлечения из очереди

```python
# ОШИБКА: дубликаты в очереди
def wrong_bfs(graph, start):
    queue = deque([start])
    while queue:
        v = queue.popleft()
        visited.add(v)  # Слишком поздно!
        for n in graph[v]:
            if n not in visited:
                queue.append(n)  # n может добавиться много раз!

# ПРАВИЛЬНО: добавляем в visited сразу
def correct_bfs(graph, start):
    visited = {start}  # Добавляем сразу
    queue = deque([start])
    while queue:
        v = queue.popleft()
        for n in graph[v]:
            if n not in visited:
                visited.add(n)  # Сразу при обнаружении
                queue.append(n)
```

### 3. Несвязный граф

```python
def bfs_all_components(graph):
    """Обход всех компонент связности."""
    visited = set()
    components = []

    for vertex in graph:
        if vertex not in visited:
            component = []
            queue = deque([vertex])
            visited.add(vertex)

            while queue:
                v = queue.popleft()
                component.append(v)

                for n in graph[v]:
                    if n not in visited:
                        visited.add(n)
                        queue.append(n)

            components.append(component)

    return components
```

### 4. BFS на взвешенном графе

```python
# BFS НЕ работает для взвешенных графов!
# Используйте Dijkstra или Bellman-Ford

# Исключение: 0-1 BFS для весов только 0 и 1
```

### 5. Бесконечный граф / граф с формулой

```python
def bfs_formula_graph(start, end, max_val=10000):
    """
    BFS на графе, заданном формулой.
    Пример: из числа x можно перейти в x+1, x-1, x*2.
    """
    if start == end:
        return 0

    visited = {start}
    queue = deque([(start, 0)])

    while queue:
        current, steps = queue.popleft()

        for next_val in [current + 1, current - 1, current * 2]:
            if next_val == end:
                return steps + 1

            if 0 <= next_val <= max_val and next_val not in visited:
                visited.add(next_val)
                queue.append((next_val, steps + 1))

    return -1
```

### Рекомендации:
1. BFS гарантирует кратчайший путь только в невзвешенных графах
2. Используйте `collections.deque` для эффективной работы с очередью
3. Добавляйте в visited сразу при обнаружении, не при извлечении
4. Для взвешенных графов используйте Dijkstra
5. Multi-source BFS полезен для задач с несколькими стартовыми точками
