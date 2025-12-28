# Бинарное дерево поиска (Binary Search Tree)

[prev: 03-heaps-stacks-queues](./03-heaps-stacks-queues.md) | [next: 05-recursion](./05-recursion.md)
---

**BST** — дерево, где для каждого узла:
- Все элементы **слева меньше**
- Все элементы **справа больше**

```
        8
       / \
      3   10
     / \    \
    1   6    14
       / \   /
      4   7 13
```

## Основные понятия

- **Корень (root)** — верхний узел (8)
- **Лист (leaf)** — узел без детей (1, 4, 7, 13)
- **Высота** — максимальное расстояние от корня до листа
- **Глубина узла** — расстояние от корня до узла

## Узел дерева

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.left = None
        self.right = None

    def __repr__(self):
        return f"Node({self.value})"
```

## Поиск — O(log n)

На каждом шаге отбрасываем половину дерева:

```python
def search(node, target):
    if node is None:
        return None

    if target == node.value:
        return node
    elif target < node.value:
        return search(node.left, target)
    else:
        return search(node.right, target)
```

```
Ищем 6 в дереве:
    8 → 6 < 8, идём влево
    3 → 6 > 3, идём вправо
    6 → нашли!

Всего 3 шага вместо перебора всех элементов.
```

### Итеративный вариант

```python
def search_iterative(node, target):
    while node is not None:
        if target == node.value:
            return node
        elif target < node.value:
            node = node.left
        else:
            node = node.right
    return None
```

## Вставка — O(log n)

```python
def insert(node, value):
    if node is None:
        return Node(value)

    if value < node.value:
        node.left = insert(node.left, value)
    elif value > node.value:
        node.right = insert(node.right, value)
    # Если value == node.value — дубликаты игнорируем или обрабатываем

    return node
```

```
Вставляем 5:
    8 → 5 < 8, влево
    3 → 5 > 3, вправо
    6 → 5 < 6, влево
    4 → 5 > 4, вправо
    None → создаём Node(5)

        8
       / \
      3   10
     / \
    1   6
       /
      4
       \
        5  ← новый узел
```

## Удаление — O(log n)

Три случая:

### 1. Узел — лист (нет детей)
Просто удаляем.

### 2. Узел имеет одного ребёнка
Заменяем узел его ребёнком.

### 3. Узел имеет двух детей
Находим **минимум в правом поддереве** (или максимум в левом), копируем его значение, удаляем тот узел.

```python
def delete(node, value):
    if node is None:
        return None

    if value < node.value:
        node.left = delete(node.left, value)
    elif value > node.value:
        node.right = delete(node.right, value)
    else:
        # Нашли узел для удаления

        # Случай 1 и 2: нет левого или правого ребёнка
        if node.left is None:
            return node.right
        if node.right is None:
            return node.left

        # Случай 3: два ребёнка
        # Находим минимальный узел в правом поддереве
        min_node = find_min(node.right)
        node.value = min_node.value
        node.right = delete(node.right, min_node.value)

    return node


def find_min(node):
    """Найти минимальный узел (самый левый)"""
    while node.left:
        node = node.left
    return node


def find_max(node):
    """Найти максимальный узел (самый правый)"""
    while node.right:
        node = node.right
    return node
```

## Обходы дерева

```
        8
       / \
      3   10
     / \
    1   6
```

### In-order (симметричный): левый → корень → правый

Даёт элементы в **отсортированном порядке**!

```python
def inorder(node):
    if node:
        inorder(node.left)
        print(node.value, end=" ")
        inorder(node.right)

# Вывод: 1 3 6 8 10
```

### Pre-order (прямой): корень → левый → правый

Полезен для копирования дерева.

```python
def preorder(node):
    if node:
        print(node.value, end=" ")
        preorder(node.left)
        preorder(node.right)

# Вывод: 8 3 1 6 10
```

### Post-order (обратный): левый → правый → корень

Полезен для удаления дерева.

```python
def postorder(node):
    if node:
        postorder(node.left)
        postorder(node.right)
        print(node.value, end=" ")

# Вывод: 1 6 3 10 8
```

### Level-order (по уровням): BFS

```python
from collections import deque

def level_order(root):
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

# Вывод: [8, 3, 10, 1, 6]
```

## Сложность операций

| Операция | Среднее | Худшее* |
|----------|---------|---------|
| Поиск | O(log n) | O(n) |
| Вставка | O(log n) | O(n) |
| Удаление | O(log n) | O(n) |
| Мин/Макс | O(log n) | O(n) |

*Худшее — когда дерево вырождается в "список":

```
Вставка 1, 2, 3, 4, 5 по порядку:

1
 \
  2
   \
    3
     \
      4
       \
        5

Высота = n, все операции O(n)
```

## Сбалансированные деревья

Чтобы избежать вырождения, используют **самобалансирующиеся** деревья:

| Дерево | Описание | Где используется |
|--------|----------|------------------|
| **AVL** | Разница высот поддеревьев ≤ 1 | Частый поиск |
| **Red-Black** | Более "ленивая" балансировка | std::map, std::set (C++) |
| **B-Tree** | Много детей у узла | Базы данных, файловые системы |
| **Splay** | Часто используемые элементы наверху | Кэши |

## Полная реализация BST

```python
class Node:
    def __init__(self, value):
        self.value = value
        self.left = None
        self.right = None


class BST:
    def __init__(self):
        self.root = None

    def insert(self, value):
        """Вставить значение"""
        self.root = self._insert(self.root, value)

    def _insert(self, node, value):
        if node is None:
            return Node(value)
        if value < node.value:
            node.left = self._insert(node.left, value)
        elif value > node.value:
            node.right = self._insert(node.right, value)
        return node

    def search(self, value):
        """Найти узел по значению"""
        return self._search(self.root, value)

    def _search(self, node, value):
        if node is None or node.value == value:
            return node
        if value < node.value:
            return self._search(node.left, value)
        return self._search(node.right, value)

    def contains(self, value):
        """Проверить наличие значения"""
        return self.search(value) is not None

    def delete(self, value):
        """Удалить значение"""
        self.root = self._delete(self.root, value)

    def _delete(self, node, value):
        if node is None:
            return None

        if value < node.value:
            node.left = self._delete(node.left, value)
        elif value > node.value:
            node.right = self._delete(node.right, value)
        else:
            if node.left is None:
                return node.right
            if node.right is None:
                return node.left

            min_node = self._find_min(node.right)
            node.value = min_node.value
            node.right = self._delete(node.right, min_node.value)

        return node

    def _find_min(self, node):
        while node.left:
            node = node.left
        return node

    def min(self):
        """Минимальное значение"""
        if self.root is None:
            return None
        return self._find_min(self.root).value

    def max(self):
        """Максимальное значение"""
        if self.root is None:
            return None
        node = self.root
        while node.right:
            node = node.right
        return node.value

    def inorder(self):
        """Получить отсортированный список"""
        result = []
        self._inorder(self.root, result)
        return result

    def _inorder(self, node, result):
        if node:
            self._inorder(node.left, result)
            result.append(node.value)
            self._inorder(node.right, result)

    def height(self):
        """Высота дерева"""
        return self._height(self.root)

    def _height(self, node):
        if node is None:
            return 0
        return 1 + max(self._height(node.left), self._height(node.right))
```

### Использование

```python
bst = BST()

# Вставка
for val in [8, 3, 10, 1, 6, 14, 4, 7]:
    bst.insert(val)

# Поиск
bst.search(6)       # Node(6)
bst.contains(100)   # False

# Обход
bst.inorder()       # [1, 3, 4, 6, 7, 8, 10, 14]

# Мин/Макс
bst.min()           # 1
bst.max()           # 14

# Высота
bst.height()        # 4

# Удаление
bst.delete(3)
bst.inorder()       # [1, 4, 6, 7, 8, 10, 14]
```

## Практические задачи

### Проверить, является ли дерево BST

```python
def is_valid_bst(node, min_val=float('-inf'), max_val=float('inf')):
    if node is None:
        return True

    if node.value <= min_val or node.value >= max_val:
        return False

    return (is_valid_bst(node.left, min_val, node.value) and
            is_valid_bst(node.right, node.value, max_val))
```

### Найти k-й наименьший элемент

```python
def kth_smallest(root, k):
    stack = []
    node = root

    while stack or node:
        while node:
            stack.append(node)
            node = node.left

        node = stack.pop()
        k -= 1
        if k == 0:
            return node.value

        node = node.right

    return None
```

### Найти общего предка (LCA)

```python
def lca(root, p, q):
    """Lowest Common Ancestor для BST"""
    while root:
        if p < root.value and q < root.value:
            root = root.left
        elif p > root.value and q > root.value:
            root = root.right
        else:
            return root
    return None
```

---
[prev: 03-heaps-stacks-queues](./03-heaps-stacks-queues.md) | [next: 05-recursion](./05-recursion.md)