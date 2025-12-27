# Бинарное дерево (Binary Tree)

## Определение

**Бинарное дерево** — это дерево, в котором каждый узел имеет не более двух детей, называемых **левым** и **правым** ребёнком. Это одна из наиболее важных структур данных в computer science.

```
         ┌───┐
         │ A │
         └─┬─┘
      ┌────┴────┐
    ┌─▼─┐     ┌─▼─┐
    │ B │     │ C │
    └─┬─┘     └─┬─┘
   ┌──┴──┐   ┌──┴──┐
 ┌─▼─┐ ┌─▼─┐│ null│┌─▼─┐
 │ D │ │ E ││     ││ F │
 └───┘ └───┘└─────┘└───┘
```

## Зачем нужно

### Практическое применение:
- **BST** — быстрый поиск, вставка, удаление
- **Heap** — приоритетные очереди
- **Expression trees** — вычисление выражений
- **Huffman trees** — сжатие данных
- **Синтаксические деревья** — компиляторы
- **Деревья решений** — машинное обучение

## Типы бинарных деревьев

### Full Binary Tree (полное)

Каждый узел имеет 0 или 2 детей.

```
       A
      / \
     B   C
    / \
   D   E

✓ Полное: у B — 2 ребёнка, у A — 2 ребёнка, у D, E, C — 0 детей
```

### Complete Binary Tree (завершённое)

Все уровни заполнены, кроме последнего. Последний уровень заполняется слева.

```
       A
      / \
     B   C
    / \ /
   D  E F

✓ Завершённое: последний уровень заполняется слева направо
```

### Perfect Binary Tree (идеальное)

Все внутренние узлы имеют 2 детей, все листья на одном уровне.

```
       A          Level 0: 1 node
      / \
     B   C        Level 1: 2 nodes
    / \ / \
   D  E F  G      Level 2: 4 nodes

Total nodes = 2^(h+1) - 1 = 2^3 - 1 = 7
```

### Balanced Binary Tree (сбалансированное)

Разница высот левого и правого поддеревьев любого узла не более 1.

```
       A
      / \
     B   C
    / \
   D   E

✓ Сбалансированное: |height(left) - height(right)| ≤ 1 для всех узлов
```

### Degenerate (вырожденное)

Каждый узел имеет только одного ребёнка (как связный список).

```
   A
  /
 B
  \
   C
  /
 D

✗ Вырожденное: по сути это связный список
Height = n - 1 (максимальная)
```

## Структура узла

```python
class TreeNode:
    def __init__(self, value):
        self.value = value
        self.left = None
        self.right = None

    def __repr__(self):
        return f"TreeNode({self.value})"
```

## Обходы бинарного дерева

### Inorder (симметричный): Left → Root → Right

```python
def inorder(node):
    """
    Левое поддерево → Корень → Правое поддерево

    Для BST возвращает отсортированный порядок!
    """
    if node is None:
        return []

    return inorder(node.left) + [node.value] + inorder(node.right)

# Итеративная версия с явным стеком
def inorder_iterative(root):
    result = []
    stack = []
    current = root

    while current or stack:
        while current:
            stack.append(current)
            current = current.left

        current = stack.pop()
        result.append(current.value)
        current = current.right

    return result
```

```
        4
       / \
      2   6
     / \ / \
    1  3 5  7

Inorder: 1, 2, 3, 4, 5, 6, 7  (отсортированный!)
```

### Preorder (прямой): Root → Left → Right

```python
def preorder(node):
    """
    Корень → Левое поддерево → Правое поддерево

    Используется для: копирование дерева, сериализация
    """
    if node is None:
        return []

    return [node.value] + preorder(node.left) + preorder(node.right)

# Итеративная версия
def preorder_iterative(root):
    if not root:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.value)

        # Правый добавляем первым, чтобы левый обрабатывался раньше
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result
```

```
        4
       / \
      2   6
     / \ / \
    1  3 5  7

Preorder: 4, 2, 1, 3, 6, 5, 7
```

### Postorder (обратный): Left → Right → Root

```python
def postorder(node):
    """
    Левое поддерево → Правое поддерево → Корень

    Используется для: удаление дерева, вычисление выражений
    """
    if node is None:
        return []

    return postorder(node.left) + postorder(node.right) + [node.value]

# Итеративная версия (два стека)
def postorder_iterative(root):
    if not root:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.value)

        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)

    return result[::-1]  # реверсируем
```

```
        4
       / \
      2   6
     / \ / \
    1  3 5  7

Postorder: 1, 3, 2, 5, 7, 6, 4
```

### Level Order (по уровням): BFS

```python
from collections import deque

def level_order(root):
    """
    Обход по уровням (BFS)

    Используется для: печать уровнями, поиск кратчайшего пути
    """
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        node = queue.popleft()
        result.append(node.value)

        if node.left:
            queue.append(node.left)
        if node.right:
            queue.append(node.right)

    return result

def level_order_by_levels(root):
    """Возвращает список списков — каждый уровень отдельно"""
    if not root:
        return []

    result = []
    queue = deque([root])

    while queue:
        level = []
        level_size = len(queue)

        for _ in range(level_size):
            node = queue.popleft()
            level.append(node.value)

            if node.left:
                queue.append(node.left)
            if node.right:
                queue.append(node.right)

        result.append(level)

    return result
```

```
        4
       / \
      2   6
     / \ / \
    1  3 5  7

Level order: 4, 2, 6, 1, 3, 5, 7
By levels: [[4], [2, 6], [1, 3, 5, 7]]
```

## Основные операции

### Высота дерева

```python
def height(node):
    """
    Высота — максимальная глубина от узла до листа
    Высота пустого дерева = -1
    Высота листа = 0
    """
    if node is None:
        return -1

    return 1 + max(height(node.left), height(node.right))
```

### Размер дерева

```python
def size(node):
    """Количество узлов в дереве"""
    if node is None:
        return 0

    return 1 + size(node.left) + size(node.right)
```

### Количество листьев

```python
def count_leaves(node):
    if node is None:
        return 0

    if node.left is None and node.right is None:
        return 1

    return count_leaves(node.left) + count_leaves(node.right)
```

### Проверка на сбалансированность

```python
def is_balanced(node):
    """Проверяет, сбалансировано ли дерево"""
    def check(node):
        if node is None:
            return 0

        left_height = check(node.left)
        if left_height == -1:
            return -1

        right_height = check(node.right)
        if right_height == -1:
            return -1

        if abs(left_height - right_height) > 1:
            return -1

        return 1 + max(left_height, right_height)

    return check(node) != -1
```

### Поиск LCA (наименьший общий предок)

```python
def lca(root, p, q):
    """
    Lowest Common Ancestor — наименьший общий предок для узлов p и q
    """
    if root is None:
        return None

    if root == p or root == q:
        return root

    left = lca(root.left, p, q)
    right = lca(root.right, p, q)

    if left and right:
        return root  # p и q в разных поддеревьях

    return left if left else right
```

## Примеры с разбором

### Пример 1: Дерево выражений

```python
class ExpressionTree:
    """
    Дерево для математических выражений.
    Операторы — внутренние узлы, операнды — листья.
    """

    def __init__(self, value):
        self.value = value
        self.left = None
        self.right = None

    def evaluate(self):
        """Вычисляет значение выражения (postorder)"""
        if self.left is None and self.right is None:
            return float(self.value)

        left_val = self.left.evaluate()
        right_val = self.right.evaluate()

        if self.value == '+':
            return left_val + right_val
        elif self.value == '-':
            return left_val - right_val
        elif self.value == '*':
            return left_val * right_val
        elif self.value == '/':
            return left_val / right_val

    def to_infix(self):
        """Преобразует в инфиксную нотацию"""
        if self.left is None and self.right is None:
            return str(self.value)

        left_str = self.left.to_infix()
        right_str = self.right.to_infix()

        return f"({left_str} {self.value} {right_str})"

# Дерево для выражения (3 + 4) * 2:
#       *
#      / \
#     +   2
#    / \
#   3   4

mult = ExpressionTree('*')
plus = ExpressionTree('+')
mult.left = plus
mult.right = ExpressionTree('2')
plus.left = ExpressionTree('3')
plus.right = ExpressionTree('4')

print(mult.to_infix())   # ((3 + 4) * 2)
print(mult.evaluate())   # 14.0
```

### Пример 2: Сериализация и десериализация

```python
def serialize(root):
    """Преобразует дерево в строку (preorder с null-маркерами)"""
    result = []

    def dfs(node):
        if node is None:
            result.append("null")
            return

        result.append(str(node.value))
        dfs(node.left)
        dfs(node.right)

    dfs(root)
    return ",".join(result)

def deserialize(data):
    """Восстанавливает дерево из строки"""
    values = iter(data.split(","))

    def dfs():
        val = next(values)
        if val == "null":
            return None

        node = TreeNode(int(val))
        node.left = dfs()
        node.right = dfs()
        return node

    return dfs()

# Пример:
#     1
#    / \
#   2   3
#      / \
#     4   5

# serialize: "1,2,null,null,3,4,null,null,5,null,null"
```

### Пример 3: Инвертирование дерева

```python
def invert_tree(root):
    """
    Инвертирует дерево (зеркально отражает).
    Известная задача — "90% инженеров не могут инвертировать дерево".
    """
    if root is None:
        return None

    # Меняем местами левое и правое поддеревья
    root.left, root.right = root.right, root.left

    invert_tree(root.left)
    invert_tree(root.right)

    return root

#    4                 4
#   / \               / \
#  2   7     →       7   2
# / \ / \           / \ / \
#1  3 6  9         9  6 3  1
```

### Пример 4: Максимальная глубина и диаметр

```python
def max_depth(root):
    """Максимальная глубина (равна высоте)"""
    if root is None:
        return 0

    return 1 + max(max_depth(root.left), max_depth(root.right))

def diameter(root):
    """
    Диаметр — длина самого длинного пути между любыми двумя узлами.
    Путь не обязательно проходит через корень.
    """
    result = [0]

    def height(node):
        if node is None:
            return 0

        left_height = height(node.left)
        right_height = height(node.right)

        # Обновляем диаметр
        result[0] = max(result[0], left_height + right_height)

        return 1 + max(left_height, right_height)

    height(root)
    return result[0]
```

### Пример 5: Проверка симметричности

```python
def is_symmetric(root):
    """Проверяет, симметрично ли дерево относительно центра"""
    def is_mirror(left, right):
        if left is None and right is None:
            return True
        if left is None or right is None:
            return False

        return (left.value == right.value and
                is_mirror(left.left, right.right) and
                is_mirror(left.right, right.left))

    if root is None:
        return True

    return is_mirror(root.left, root.right)

#     1
#    / \
#   2   2
#  / \ / \
# 3  4 4  3
#
# ✓ Симметричное
```

## Построение дерева

### Из inorder и preorder

```python
def build_tree(preorder, inorder):
    """
    Строит дерево по preorder и inorder обходам.
    """
    if not preorder or not inorder:
        return None

    root_val = preorder[0]
    root = TreeNode(root_val)

    mid = inorder.index(root_val)

    root.left = build_tree(
        preorder[1:mid + 1],
        inorder[:mid]
    )
    root.right = build_tree(
        preorder[mid + 1:],
        inorder[mid + 1:]
    )

    return root

# preorder = [3, 9, 20, 15, 7]
# inorder  = [9, 3, 15, 20, 7]
#
# Дерево:
#     3
#    / \
#   9  20
#      / \
#     15  7
```

## Анализ сложности

| Операция | Время | Примечание |
|----------|-------|------------|
| Обход | O(n) | Посещаем каждый узел |
| Поиск | O(n) | В худшем случае |
| Высота | O(n) | Обход всего дерева |
| Вставка | O(n) | Если нужно найти место |
| Удаление | O(n) | Поиск + удаление |

### Для сбалансированного дерева

| Операция | Время |
|----------|-------|
| Поиск | O(log n) |
| Вставка | O(log n) |
| Удаление | O(log n) |

## Типичные ошибки

### 1. Неправильный базовый случай

```python
# НЕПРАВИЛЬНО
def height(node):
    if node.left is None and node.right is None:
        return 0
    # Упадёт если node = None!

# ПРАВИЛЬНО
def height(node):
    if node is None:
        return -1  # или 0, в зависимости от определения
    ...
```

### 2. Забытый return

```python
# НЕПРАВИЛЬНО
def search(node, val):
    if node is None:
        return None
    if node.value == val:
        return node
    search(node.left, val)   # забыли return!
    search(node.right, val)

# ПРАВИЛЬНО
def search(node, val):
    if node is None:
        return None
    if node.value == val:
        return node
    left = search(node.left, val)
    if left:
        return left
    return search(node.right, val)
```

### 3. Путаница left/right

```python
# Preorder: Root → Left → Right
# Важно: при итеративной реализации добавляем RIGHT первым!

stack.append(node.right)  # первым
stack.append(node.left)   # вторым — обработается раньше
```

## Формулы для бинарных деревьев

```
Для дерева высоты h:

Perfect binary tree:
- Узлов на уровне i: 2^i
- Всего узлов: 2^(h+1) - 1
- Листьев: 2^h
- Внутренних узлов: 2^h - 1

Complete binary tree с n узлами:
- Высота: floor(log₂(n))
- Листьев: ceil(n/2)

Full binary tree с n узлами:
- Листьев: (n+1)/2
- Внутренних узлов: (n-1)/2
```
