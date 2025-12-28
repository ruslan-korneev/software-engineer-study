# Решение лабиринта (Maze Solving)

[prev: 02-n-queens](./02-n-queens.md) | [next: 01-greedy-concept](../07-greedy/01-greedy-concept.md)
---

## Определение

**Решение лабиринта** — классическая задача на backtracking, где нужно найти путь от начальной точки до конечной, избегая препятствий. Алгоритм пробует различные направления и возвращается назад, если заходит в тупик.

## Зачем нужен

### Области применения:
- **Робототехника** — автономная навигация
- **Игры** — поиск пути (pathfinding)
- **GPS-навигация** — маршрутизация
- **Планирование** — логистика, распределение ресурсов
- **Биоинформатика** — поиск путей в молекулярных структурах

### Варианты задачи:
- Найти **любой путь**
- Найти **все пути**
- Найти **кратчайший путь** (лучше использовать BFS)
- Найти **путь с минимальной стоимостью** (Dijkstra, A*)

## Как работает

### Представление лабиринта

```
0 — проход
1 — стена
S — старт
E — конец

Лабиринт:
┌───┬───┬───┬───┬───┐
│ S │ 0 │ 1 │ 0 │ 0 │
├───┼───┼───┼───┼───┤
│ 0 │ 0 │ 1 │ 0 │ 1 │
├───┼───┼───┼───┼───┤
│ 1 │ 0 │ 0 │ 0 │ 0 │
├───┼───┼───┼───┼───┤
│ 0 │ 1 │ 1 │ 1 │ 0 │
├───┼───┼───┼───┼───┤
│ 0 │ 0 │ 0 │ 0 │ E │
└───┴───┴───┴───┴───┘
```

### Визуализация backtracking

```
Шаг 1: Начинаем в (0,0), идём вправо
┌───┬───┬───┬───┬───┐
│ * │ * │ █ │   │   │
├───┼───┼───┼───┼───┤
│   │   │ █ │   │ █ │
├───┼───┼───┼───┼───┤
│ █ │   │   │   │   │
├───┼───┼───┼───┼───┤
│   │ █ │ █ │ █ │   │
├───┼───┼───┼───┼───┤
│   │   │   │   │ E │
└───┴───┴───┴───┴───┘

Шаг 2: Стена справа! Идём вниз
┌───┬───┬───┬───┬───┐
│ * │ * │ █ │   │   │
├───┼───┼───┼───┼───┤
│   │ * │ █ │   │ █ │  ← Спустились
├───┼───┼───┼───┼───┤
│ █ │   │   │   │   │
├───┼───┼───┼───┼───┤
│   │ █ │ █ │ █ │   │
├───┼───┼───┼───┼───┤
│   │   │   │   │ E │
└───┴───┴───┴───┴───┘

... продолжаем, пока не найдём E или не BACKTRACK
```

### Дерево решений

```
                    (0,0)
                   /     \
              (0,1)       (1,0)
              /   \          \
          WALL   (1,1)      (2,0)
                 /    \       WALL
            (2,1)    WALL
            /    \
        (2,2)   WALL
           |
        (2,3)
           |
        (2,4)
           |
        (3,4)
           |
        (4,4) ✓ FOUND!
```

## Псевдокод

```
function solveMaze(maze, start, end):
    path = []

    function backtrack(row, col):
        if (row, col) == end:
            path.append((row, col))
            return True

        if not isValid(row, col):
            return False

        # Помечаем как посещённую
        maze[row][col] = VISITED
        path.append((row, col))

        # Пробуем все направления
        for (dr, dc) in [(0,1), (1,0), (0,-1), (-1,0)]:
            if backtrack(row + dr, col + dc):
                return True

        # Откат
        path.pop()
        maze[row][col] = OPEN  # Опционально: для поиска всех путей

        return False

    backtrack(start[0], start[1])
    return path
```

### Базовая реализация на Python

```python
def solve_maze(maze, start, end):
    """
    Решение лабиринта методом backtracking.

    Args:
        maze: 2D список (0 — проход, 1 — стена)
        start: кортеж (row, col) начальной позиции
        end: кортеж (row, col) конечной позиции

    Returns:
        Путь как список координат или None
    """
    rows, cols = len(maze), len(maze[0])
    directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]  # Право, Низ, Лево, Верх
    path = []

    def is_valid(r, c):
        return (0 <= r < rows and 0 <= c < cols and
                maze[r][c] == 0)

    def backtrack(r, c):
        if (r, c) == end:
            path.append((r, c))
            return True

        if not is_valid(r, c):
            return False

        # Помечаем как посещённую
        maze[r][c] = 2  # или любое значение != 0, 1
        path.append((r, c))

        # Пробуем все направления
        for dr, dc in directions:
            if backtrack(r + dr, c + dc):
                return True

        # Откат
        path.pop()
        maze[r][c] = 0  # Восстанавливаем

        return False

    if backtrack(start[0], start[1]):
        return path
    return None


# Пример
maze = [
    [0, 0, 1, 0, 0],
    [0, 0, 1, 0, 1],
    [1, 0, 0, 0, 0],
    [0, 1, 1, 1, 0],
    [0, 0, 0, 0, 0]
]

path = solve_maze(maze, (0, 0), (4, 4))
print(path)
# [(0, 0), (0, 1), (1, 1), (2, 1), (2, 2), (2, 3), (2, 4), (3, 4), (4, 4)]
```

### Поиск всех путей

```python
def find_all_paths(maze, start, end):
    """Найти все возможные пути."""
    rows, cols = len(maze), len(maze[0])
    directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]
    all_paths = []

    def backtrack(r, c, path):
        if (r, c) == end:
            all_paths.append(path + [(r, c)])
            return

        if not (0 <= r < rows and 0 <= c < cols) or maze[r][c] != 0:
            return

        maze[r][c] = 2  # Посещена
        path.append((r, c))

        for dr, dc in directions:
            backtrack(r + dr, c + dc, path)

        path.pop()
        maze[r][c] = 0  # Восстанавливаем для других путей!

    backtrack(start[0], start[1], [])
    return all_paths
```

### Поиск кратчайшего пути (лучше BFS!)

```python
from collections import deque

def shortest_path_bfs(maze, start, end):
    """
    Кратчайший путь с помощью BFS.
    Для кратчайшего пути BFS лучше, чем backtracking!
    """
    rows, cols = len(maze), len(maze[0])
    directions = [(0, 1), (1, 0), (0, -1), (-1, 0)]

    queue = deque([(start[0], start[1], [start])])
    visited = {start}

    while queue:
        r, c, path = queue.popleft()

        if (r, c) == end:
            return path

        for dr, dc in directions:
            nr, nc = r + dr, c + dc

            if (0 <= nr < rows and 0 <= nc < cols and
                maze[nr][nc] == 0 and (nr, nc) not in visited):

                visited.add((nr, nc))
                queue.append((nr, nc, path + [(nr, nc)]))

    return None
```

### Лабиринт с диагональным движением

```python
def solve_maze_diagonal(maze, start, end):
    """Лабиринт с 8 направлениями движения."""
    rows, cols = len(maze), len(maze[0])

    # 8 направлений: 4 основных + 4 диагональных
    directions = [
        (0, 1), (1, 0), (0, -1), (-1, 0),  # Право, Низ, Лево, Верх
        (1, 1), (1, -1), (-1, 1), (-1, -1)  # Диагонали
    ]

    path = []
    visited = set()

    def backtrack(r, c):
        if (r, c) == end:
            path.append((r, c))
            return True

        if (not (0 <= r < rows and 0 <= c < cols) or
            maze[r][c] == 1 or (r, c) in visited):
            return False

        visited.add((r, c))
        path.append((r, c))

        for dr, dc in directions:
            if backtrack(r + dr, c + dc):
                return True

        path.pop()
        visited.remove((r, c))

        return False

    if backtrack(start[0], start[1]):
        return path
    return None
```

### Крысиный лабиринт (Rat in a Maze)

```python
def rat_maze(maze):
    """
    Классическая задача: крыса из (0,0) в (n-1, n-1).
    Можно двигаться только вправо или вниз.
    """
    n = len(maze)
    solution = [[0] * n for _ in range(n)]

    def backtrack(r, c):
        if r == n - 1 and c == n - 1 and maze[r][c] == 1:
            solution[r][c] = 1
            return True

        if 0 <= r < n and 0 <= c < n and maze[r][c] == 1:
            solution[r][c] = 1

            # Только вправо или вниз
            if backtrack(r, c + 1):  # Вправо
                return True
            if backtrack(r + 1, c):  # Вниз
                return True

            solution[r][c] = 0  # Откат
            return False

        return False

    if backtrack(0, 0):
        return solution
    return None
```

## Анализ сложности

| Аспект | Сложность |
|--------|-----------|
| **Время (худший)** | O(4^(m×n)) без оптимизации |
| **Время (с visited)** | O(m × n) |
| **Пространство** | O(m × n) для visited + O(m + n) для стека |

## Типичные ошибки и Edge Cases

### 1. Забыли восстановить клетку при откате

```python
# ОШИБКА: maze[r][c] остаётся помеченной
def wrong_solve(maze, r, c):
    maze[r][c] = 2  # Помечаем
    for dr, dc in directions:
        if wrong_solve(maze, r + dr, c + dc):
            return True
    # Забыли maze[r][c] = 0!
    return False

# ПРАВИЛЬНО
def correct_solve(maze, r, c):
    maze[r][c] = 2
    for dr, dc in directions:
        if correct_solve(maze, r + dr, c + dc):
            return True
    maze[r][c] = 0  # Восстанавливаем!
    return False
```

### 2. Бесконечный цикл без проверки посещённых

```python
# ОШИБКА: ходим по кругу
def infinite_loop(maze, r, c, end):
    if (r, c) == end:
        return True
    if maze[r][c] != 0:  # Только стены!
        return False
    # Можем вернуться в уже посещённую клетку!

# ПРАВИЛЬНО: отмечаем посещённые
def no_loop(maze, r, c, end):
    if (r, c) == end:
        return True
    if maze[r][c] != 0:
        return False
    maze[r][c] = 2  # Помечаем!
    # ...
```

### 3. Старт или финиш на стене

```python
def solve_maze_safe(maze, start, end):
    if maze[start[0]][start[1]] == 1:
        return None  # Старт на стене
    if maze[end[0]][end[1]] == 1:
        return None  # Финиш на стене
    # ...
```

### 4. Пустой лабиринт

```python
def solve_maze_safe(maze, start, end):
    if not maze or not maze[0]:
        return None
    # ...
```

### 5. Визуализация пути

```python
def print_maze_with_path(maze, path):
    """Красивый вывод лабиринта с путём."""
    path_set = set(path)
    symbols = {0: ' ', 1: '█', 2: '·'}

    for r, row in enumerate(maze):
        line = ""
        for c, cell in enumerate(row):
            if (r, c) in path_set:
                line += '* '
            elif cell == 1:
                line += '█ '
            else:
                line += '  '
        print(line)

# Пример
maze = [
    [0, 0, 1, 0],
    [0, 0, 1, 0],
    [1, 0, 0, 0],
    [0, 0, 0, 0]
]
path = solve_maze(maze.copy(), (0, 0), (3, 3))
print_maze_with_path(maze, path)
```

### Рекомендации:
1. Для кратчайшего пути используйте BFS, не backtracking
2. Обязательно отмечайте посещённые клетки
3. Восстанавливайте состояние при откате (для поиска всех путей)
4. Проверяйте граничные случаи: старт/финиш на стене, пустой лабиринт
5. Для взвешенного лабиринта используйте Dijkstra или A*

---
[prev: 02-n-queens](./02-n-queens.md) | [next: 01-greedy-concept](../07-greedy/01-greedy-concept.md)