# Задача N ферзей (N-Queens Problem)

## Определение

**Задача N ферзей** — классическая задача на backtracking: расставить N ферзей на шахматной доске N×N так, чтобы ни один ферзь не атаковал другого.

Ферзь атакует по горизонтали, вертикали и обеим диагоналям.

## Зачем нужен

### Значение задачи:
- **Классика backtracking** — отличный пример для изучения
- **Тестирование алгоритмов** — бенчмарк для оптимизаций
- **Constraint Satisfaction Problems (CSP)**
- **Параллельные вычисления** — хорошо распараллеливается

### Интересные факты:
- Для N=8 существует 92 решения (12 уникальных с учётом симметрии)
- Для N=1 — 1 решение, для N=2 и N=3 — решений нет
- Количество решений растёт примерно экспоненциально

## Как работает

### Визуализация атак ферзя

```
    0   1   2   3   4   5   6   7
  ┌───┬───┬───┬───┬───┬───┬───┬───┐
0 │ ↖ │   │   │ ↑ │   │   │   │ ↗ │
  ├───┼───┼───┼───┼───┼───┼───┼───┤
1 │   │ ↖ │   │ ↑ │   │   │ ↗ │   │
  ├───┼───┼───┼───┼───┼───┼───┼───┤
2 │   │   │ ↖ │ ↑ │   │ ↗ │   │   │
  ├───┼───┼───┼───┼───┼───┼───┼───┤
3 │ ← │ ← │ ← │ Q │ → │ → │ → │ → │  Ферзь атакует
  ├───┼───┼───┼───┼───┼───┼───┼───┤  все эти клетки
4 │   │   │ ↙ │ ↓ │   │ ↘ │   │   │
  ├───┼───┼───┼───┼───┼───┼───┼───┤
5 │   │ ↙ │   │ ↓ │   │   │ ↘ │   │
  ├───┼───┼───┼───┼───┼───┼───┼───┤
6 │ ↙ │   │   │ ↓ │   │   │   │ ↘ │
  ├───┼───┼───┼───┼───┼───┼───┼───┤
7 │   │   │   │ ↓ │   │   │   │   │
  └───┴───┴───┴───┴───┴───┴───┴───┘
```

### Процесс backtracking

```
Размещаем ферзей по строкам (row by row):

Строка 0: Пробуем col=0
┌───┬───┬───┬───┐
│ Q │   │   │   │  ← Ферзь в (0,0)
├───┼───┼───┼───┤
│   │   │   │   │
├───┼───┼───┼───┤
│   │   │   │   │
├───┼───┼───┼───┤
│   │   │   │   │
└───┴───┴───┴───┘

Строка 1: col=0 атакован, col=1 атакован, пробуем col=2
┌───┬───┬───┬───┐
│ Q │ X │   │   │
├───┼───┼───┼───┤
│ X │ X │ Q │   │  ← Ферзь в (1,2)
├───┼───┼───┼───┤
│   │   │   │   │
├───┼───┼───┼───┤
│   │   │   │   │
└───┴───┴───┴───┘

Строка 2: все атакованы! BACKTRACK к строке 1

Строка 1: пробуем col=3
┌───┬───┬───┬───┐
│ Q │   │   │   │
├───┼───┼───┼───┤
│   │   │   │ Q │  ← Ферзь в (1,3)
├───┼───┼───┼───┤
│   │   │   │   │
├───┼───┼───┼───┤
│   │   │   │   │
└───┴───┴───┴───┘

Строка 2: col=0 атакован, пробуем col=1
┌───┬───┬───┬───┐
│ Q │   │   │   │
├───┼───┼───┼───┤
│   │   │   │ Q │
├───┼───┼───┼───┤
│   │ Q │   │   │  ← Ферзь в (2,1)
├───┼───┼───┼───┤
│   │   │   │   │
└───┴───┴───┴───┘

... и так далее
```

## Псевдокод

```
function solveNQueens(n):
    solutions = []
    board = empty n×n board

    function backtrack(row):
        if row == n:
            solutions.add(copy(board))
            return

        for col from 0 to n-1:
            if is_safe(row, col):
                place_queen(row, col)
                backtrack(row + 1)
                remove_queen(row, col)

    backtrack(0)
    return solutions
```

### Базовая реализация на Python

```python
def solve_n_queens(n):
    """
    Решение задачи N ферзей.

    Returns:
        Список всех возможных расстановок.
    """
    solutions = []
    board = [['.' for _ in range(n)] for _ in range(n)]

    def is_safe(row, col):
        """Проверка, можно ли поставить ферзя."""
        # Проверяем столбец (выше текущей строки)
        for i in range(row):
            if board[i][col] == 'Q':
                return False

        # Проверяем левую диагональ
        i, j = row - 1, col - 1
        while i >= 0 and j >= 0:
            if board[i][j] == 'Q':
                return False
            i -= 1
            j -= 1

        # Проверяем правую диагональ
        i, j = row - 1, col + 1
        while i >= 0 and j < n:
            if board[i][j] == 'Q':
                return False
            i -= 1
            j += 1

        return True

    def backtrack(row):
        if row == n:
            # Нашли решение
            solution = [''.join(r) for r in board]
            solutions.append(solution)
            return

        for col in range(n):
            if is_safe(row, col):
                board[row][col] = 'Q'
                backtrack(row + 1)
                board[row][col] = '.'

    backtrack(0)
    return solutions


# Пример для N=4
for solution in solve_n_queens(4):
    for row in solution:
        print(row)
    print()
```

### Оптимизированная версия с множествами

```python
def solve_n_queens_optimized(n):
    """
    Оптимизированное решение с O(1) проверкой атаки.
    Используем множества для отслеживания атакованных позиций.
    """
    solutions = []
    queens = []  # Позиции ферзей (row, col)

    # Множества для атакованных столбцов и диагоналей
    cols = set()
    diag1 = set()  # row - col
    diag2 = set()  # row + col

    def backtrack(row):
        if row == n:
            # Формируем решение
            board = []
            for r, c in queens:
                board.append('.' * c + 'Q' + '.' * (n - c - 1))
            solutions.append(board)
            return

        for col in range(n):
            d1 = row - col
            d2 = row + col

            if col in cols or d1 in diag1 or d2 in diag2:
                continue

            # Делаем ход
            queens.append((row, col))
            cols.add(col)
            diag1.add(d1)
            diag2.add(d2)

            backtrack(row + 1)

            # Откатываем
            queens.pop()
            cols.remove(col)
            diag1.remove(d1)
            diag2.remove(d2)

    backtrack(0)
    return solutions
```

### Подсчёт количества решений

```python
def count_n_queens(n):
    """Только подсчёт решений (быстрее, чем построение)."""
    count = [0]

    cols = set()
    diag1 = set()
    diag2 = set()

    def backtrack(row):
        if row == n:
            count[0] += 1
            return

        for col in range(n):
            d1, d2 = row - col, row + col

            if col in cols or d1 in diag1 or d2 in diag2:
                continue

            cols.add(col)
            diag1.add(d1)
            diag2.add(d2)

            backtrack(row + 1)

            cols.remove(col)
            diag1.remove(d1)
            diag2.remove(d2)

    backtrack(0)
    return count[0]

# Количество решений для разных N
for n in range(1, 13):
    print(f"N={n}: {count_n_queens(n)} solutions")
```

### Битовые операции (максимальная оптимизация)

```python
def count_n_queens_bitwise(n):
    """Сверхбыстрая версия с битовыми операциями."""
    count = [0]
    all_ones = (1 << n) - 1  # n единиц

    def backtrack(cols, diag1, diag2):
        if cols == all_ones:
            count[0] += 1
            return

        # Доступные позиции
        available = all_ones & ~(cols | diag1 | diag2)

        while available:
            # Выбираем младший установленный бит
            pos = available & -available
            available -= pos

            # Рекурсия
            backtrack(
                cols | pos,
                (diag1 | pos) << 1,
                (diag2 | pos) >> 1
            )

    backtrack(0, 0, 0)
    return count[0]
```

## Анализ сложности

| Аспект | Сложность |
|--------|-----------|
| **Время (наивный)** | O(n! × n) |
| **Время (с отсечением)** | O(n!) в худшем, значительно меньше на практике |
| **Пространство** | O(n) — глубина рекурсии |

### Количество решений:
| N | Решений | Уникальных |
|---|---------|------------|
| 1 | 1 | 1 |
| 4 | 2 | 1 |
| 8 | 92 | 12 |
| 10 | 724 | 92 |
| 12 | 14,200 | 1,787 |

## Типичные ошибки и Edge Cases

### 1. N = 2 и N = 3 — нет решений

```python
def solve_n_queens_safe(n):
    if n < 1:
        return []
    if n == 2 or n == 3:
        return []  # Нет решений!
    # ...
```

### 2. Проверка только верхней части доски

```python
# ПРАВИЛЬНО: проверяем только выше текущей строки,
# так как ниже ферзей ещё нет!
def is_safe(row, col):
    for i in range(row):  # Только 0..row-1
        # ...
```

### 3. Неправильная формула диагонали

```python
# Диагонали определяются так:
# Главная диагональ (↘): row - col = const
# Побочная диагональ (↗): row + col = const

# Проверка
def on_same_diagonal(r1, c1, r2, c2):
    return abs(r1 - r2) == abs(c1 - c2)
```

### 4. Вывод одного решения

```python
def solve_n_queens_one(n):
    """Найти только одно решение (быстрее)."""
    queens = []

    cols = set()
    diag1 = set()
    diag2 = set()

    def backtrack(row):
        if row == n:
            return True  # Нашли!

        for col in range(n):
            d1, d2 = row - col, row + col

            if col in cols or d1 in diag1 or d2 in diag2:
                continue

            queens.append(col)
            cols.add(col)
            diag1.add(d1)
            diag2.add(d2)

            if backtrack(row + 1):
                return True  # Прекращаем поиск!

            queens.pop()
            cols.remove(col)
            diag1.remove(d1)
            diag2.remove(d2)

        return False

    if backtrack(0):
        return queens
    return None
```

### 5. Визуализация решения

```python
def print_board(n, queens):
    """Красивый вывод доски."""
    for row in range(n):
        line = ""
        for col in range(n):
            if queens[row] == col:
                line += "Q "
            else:
                line += ". "
        print(line)
    print()

# Пример
solution = solve_n_queens_one(8)
if solution:
    print_board(8, solution)
```

### Рекомендации:
1. Используйте множества для O(1) проверки атаки
2. Обрабатывайте строки по порядку — гарантированно не будет конфликтов по горизонтали
3. Битовые операции — максимальная скорость для подсчёта
4. Для N > 15 рассмотрите параллельные вычисления
5. N=2 и N=3 не имеют решений — это не ошибка
