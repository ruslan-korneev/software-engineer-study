# Обход в ширину / По уровням (Level Order / BFS)

[prev: 03-postorder](./03-postorder.md) | [next: 01-dfs](../05-graph-algorithms/01-dfs.md)
---

## Определение

**Обход в ширину (Level Order Traversal / BFS)** — это способ обхода дерева, при котором узлы посещаются уровень за уровнем, слева направо. Сначала посещается корень (уровень 0), затем все узлы уровня 1, потом уровня 2 и так далее.

BFS = Breadth-First Search (поиск в ширину)

## Зачем нужен

### Области применения:
- **Поиск кратчайшего пути** — в невзвешенных графах
- **Печать дерева по уровням**
- **Поиск на определённом уровне**
- **Нахождение ширины дерева**
- **Сериализация дерева**
- **Проверка полноты дерева**
- **Нахождение правого/левого вида дерева**

### Преимущества:
- Находит кратчайший путь (в невзвешенных графах)
- Обрабатывает ближние узлы раньше дальних
- Естественен для задач по уровням

## Как работает

### Визуализация процесса

```
Дерево:
        1
       / \
      2   3
     / \ / \
    4  5 6  7

Обход по уровням:

Уровень 0:    [1]
               ↓
Уровень 1:  [2, 3]
             ↓   ↓
Уровень 2: [4, 5, 6, 7]

Результат: [1, 2, 3, 4, 5, 6, 7]
```

### Работа очереди

```
Очередь (FIFO):

Шаг 0: queue = [1]
       visit 1, add children
       queue = [2, 3]

Шаг 1: queue = [2, 3]
       visit 2, add children
       queue = [3, 4, 5]

Шаг 2: queue = [3, 4, 5]
       visit 3, add children
       queue = [4, 5, 6, 7]

Шаг 3: queue = [4, 5, 6, 7]
       visit 4 (нет детей)
       queue = [5, 6, 7]

Шаг 4-6: visit 5, 6, 7

Результат: [1, 2, 3, 4, 5, 6, 7]
```

### Сравнение DFS и BFS

```
        1
       / \
      2   3
     / \
    4   5

DFS (preorder):  1 → 2 → 4 → 5 → 3  (глубоко, потом широко)
BFS (level):     1 → 2 → 3 → 4 → 5  (широко, потом глубоко)
```

## Псевдокод

```
function levelOrder(root):
    if root is null:
        return []

    result = []
    queue = [root]

    while queue is not empty:
        node = queue.dequeue()
        result.append(node.value)

        if node.left is not null:
            queue.enqueue(node.left)
        if node.right is not null:
            queue.enqueue(node.right)

    return result
```

### Базовая реализация на Python

```python
from collections import deque

class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def level_order(root):
    """Обход в ширину (BFS)."""
    if root is None:
        return []

    result = []
    queue = deque([root])

    while queue:
        node = queue.popleft()
        result.append(node.val)

        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)

    return result
```

### Обход с группировкой по уровням

```python
def level_order_grouped(root):
    """Обход с группировкой узлов по уровням."""
    if root is None:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        level = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result

# Результат: [[1], [2, 3], [4, 5, 6, 7]]
```

### Zigzag (зигзаг) обход

```python
def zigzag_level_order(root):
    """Обход зигзагом: чётные уровни слева направо, нечётные — справа налево."""
    if root is None:
        return []

    result = []
    queue = deque([root])
    left_to_right = True

    while queue:
        level_size = len(queue)
        level = deque()

        for _ in range(level_size):
            node = queue.popleft()

            if left_to_right:
                level.append(node.val)
            else:
                level.appendleft(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(list(level))
        left_to_right = not left_to_right

    return result

# Пример: [[1], [3, 2], [4, 5, 6, 7]]
```

### Обход снизу вверх

```python
def level_order_bottom(root):
    """Обход по уровням снизу вверх."""
    if root is None:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        level = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result[::-1]  # Разворачиваем
```

### Рекурсивная версия (DFS для BFS результата)

```python
def level_order_recursive(root):
    """Рекурсивный BFS через DFS."""
    result = []

    def dfs(node, level):
        if node is None:
            return

        if len(result) == level:
            result.append([])

        result[level].append(node.val)

        dfs(node.left, level + 1)
        dfs(node.right, level + 1)

    dfs(root, 0)
    return result
```

## Анализ сложности

| Аспект | Сложность |
|--------|-----------|
| **Время** | O(n) — каждый узел посещается один раз |
| **Память** | O(w) — ширина дерева (максимум узлов на одном уровне) |

Ширина:
- Полное дерево: O(n/2) = O(n)
- Вырожденное: O(1)

## Пример с пошаговым разбором

### Нахождение максимальной глубины

```python
def max_depth_bfs(root):
    """Максимальная глубина дерева через BFS."""
    if root is None:
        return 0

    depth = 0
    queue = deque([root])

    while queue:
        depth += 1
        level_size = len(queue)

        for _ in range(level_size):
            node = queue.popleft()

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return depth
```

### Правый вид дерева

```python
def right_side_view(root):
    """Правый вид дерева (последний узел каждого уровня)."""
    if root is None:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)

        for i in range(level_size):
            node = queue.popleft()

            # Последний узел на уровне
            if i == level_size - 1:
                result.append(node.val)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

    return result
```

### Минимальная глубина (до первого листа)

```python
def min_depth(root):
    """Минимальная глубина — расстояние до ближайшего листа."""
    if root is None:
        return 0

    queue = deque([(root, 1)])

    while queue:
        node, depth = queue.popleft()

        # Нашли лист!
        if node.left is None and node.right is None:
            return depth

        if node.left:
            queue.append((node.left, depth + 1))
        if node.right:
            queue.append((node.right, depth + 1))

    return 0
```

### Проверка полного бинарного дерева

```python
def is_complete_tree(root):
    """Проверка, является ли дерево полным."""
    if root is None:
        return True

    queue = deque([root])
    found_null = False

    while queue:
        node = queue.popleft()

        if node is None:
            found_null = True
        else:
            if found_null:
                return False  # Узел после None — не полное
            queue.append(node.left)
            queue.append(node.right)

    return True
```

## Типичные ошибки и Edge Cases

### 1. Использование списка вместо deque

```python
# НЕЭФФЕКТИВНО: pop(0) — O(n)
def bfs_slow(root):
    queue = [root]
    while queue:
        node = queue.pop(0)  # O(n)!
        # ...

# ПРАВИЛЬНО: popleft() — O(1)
from collections import deque

def bfs_fast(root):
    queue = deque([root])
    while queue:
        node = queue.popleft()  # O(1)
        # ...
```

### 2. Пустое дерево

```python
def level_order_safe(root):
    if root is None:
        return []
    # ...
```

### 3. Дерево из одного узла

```python
root = TreeNode(42)
print(level_order(root))  # [42]
print(level_order_grouped(root))  # [[42]]
```

### 4. Забыли проверить наличие детей

```python
# ОШИБКА: добавление None в очередь
def bfs_wrong(root):
    queue = deque([root])
    while queue:
        node = queue.popleft()
        result.append(node.val)
        queue.append(node.left)   # Может быть None!
        queue.append(node.right)  # Может быть None!

# ПРАВИЛЬНО
def bfs_correct(root):
    queue = deque([root])
    while queue:
        node = queue.popleft()
        result.append(node.val)
        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)
```

### 5. N-арное дерево

```python
class NaryNode:
    def __init__(self, val, children=None):
        self.val = val
        self.children = children or []

def level_order_nary(root):
    """BFS для N-арного дерева."""
    if root is None:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        level = []

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.val)

            for child in node.children:
                queue.append(child)

        result.append(level)

    return result
```

### 6. Максимальная ширина дерева

```python
def width_of_binary_tree(root):
    """
    Максимальная ширина дерева.
    Ширина = расстояние между крайними узлами на уровне.
    """
    if root is None:
        return 0

    max_width = 0
    queue = deque([(root, 0)])  # (node, position)

    while queue:
        level_size = len(queue)
        _, first_pos = queue[0]
        _, last_pos = queue[-1]

        max_width = max(max_width, last_pos - first_pos + 1)

        for _ in range(level_size):
            node, pos = queue.popleft()

            if node.left:
                queue.append((node.left, 2 * pos))
            if node.right:
                queue.append((node.right, 2 * pos + 1))

    return max_width
```

### 7. Среднее значение на каждом уровне

```python
def average_of_levels(root):
    """Среднее значение узлов на каждом уровне."""
    if root is None:
        return []

    result = []
    queue = deque([root])

    while queue:
        level_size = len(queue)
        level_sum = 0

        for _ in range(level_size):
            node = queue.popleft()
            level_sum += node.val

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level_sum / level_size)

    return result
```

### Рекомендации:
1. BFS использует очередь (FIFO), DFS использует стек (LIFO)
2. Используйте `collections.deque` для эффективной работы с очередью
3. BFS находит кратчайший путь в невзвешенных графах
4. Для задач по уровням — группируйте узлы по размеру уровня
5. Помните: BFS использует больше памяти, чем DFS для глубоких деревьев

---
[prev: 03-postorder](./03-postorder.md) | [next: 01-dfs](../05-graph-algorithms/01-dfs.md)