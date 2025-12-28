# Обратный обход дерева (Postorder Traversal)

[prev: 02-inorder](./02-inorder.md) | [next: 04-level-order-bfs](./04-level-order-bfs.md)
---

## Определение

**Обратный обход (Postorder)** — это способ обхода дерева, при котором сначала рекурсивно обходится левое поддерево, затем правое поддерево, и только потом посещается сам узел.

Порядок: **Левое поддерево → Правое поддерево → Корень** (LRN — Left, Right, Node)

## Зачем нужен

### Области применения:
- **Удаление дерева** — сначала удаляем детей, потом родителя
- **Вычисление выражений** — постфиксная (обратная польская) нотация
- **Вычисление размера поддеревьев**
- **Освобождение ресурсов** — закрытие файлов, соединений
- **Построение выражения из постфиксной нотации**

### Преимущества:
- Корень обрабатывается последним
- Естественен для задач, где нужно сначала обработать детей
- Безопасен для удаления узлов

## Как работает

### Визуализация процесса

```
Дерево:
        1
       / \
      2   3
     / \   \
    4   5   6

Порядок обхода Postorder:

Шаг 1: Идём влево от 1 → 2 → 4
       4 — листовой узел, посещаем
        1
       / \
      2   3
     / \   \
   [4]  5   6

Шаг 2: Возвращаемся к 2, идём вправо к 5
       5 — листовой, посещаем
        1
       / \
      2   3
     / \   \
    4  [5]  6

Шаг 3: Возвращаемся к 2
       Оба ребёнка обработаны, посещаем 2
        1
       / \
     [2]  3
     / \   \
    4   5   6

Шаг 4: Возвращаемся к 1, идём вправо к 3 → 6
       6 — листовой, посещаем
        1
       / \
      2   3
     / \   \
    4   5  [6]

Шаг 5: Возвращаемся к 3
       Ребёнок обработан, посещаем 3
        1
       / \
      2  [3]
     / \   \
    4   5   6

Шаг 6: Возвращаемся к 1
       Оба поддерева обработаны, посещаем 1
       [1]
       / \
      2   3
     / \   \
    4   5   6

Результат: [4, 5, 2, 6, 3, 1]
```

### Схема обхода

```
              1 (6-й — последний!)
             / \
    (3-й)   2   3 (5-й)
           / \   \
  (1-й)   4   5   6 (4-й)
            (2-й)

Путь: сначала все дети, потом родитель
```

## Псевдокод

```
function postorder(node):
    if node is null:
        return

    postorder(node.left)     # Сначала левое поддерево
    postorder(node.right)    # Потом правое поддерево
    visit(node)              # В конце — сам узел
```

### Рекурсивная реализация на Python

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def postorder_recursive(root):
    """Рекурсивный обратный обход."""
    result = []

    def traverse(node):
        if node is None:
            return

        traverse(node.left)        # Левое
        traverse(node.right)       # Правое
        result.append(node.val)    # Узел

    traverse(root)
    return result
```

### Итеративная реализация (два стека)

```python
def postorder_two_stacks(root):
    """Итеративный обратный обход с двумя стеками."""
    if root is None:
        return []

    stack1 = [root]
    stack2 = []

    while stack1:
        node = stack1.pop()
        stack2.append(node)

        # Добавляем детей (левый первым, потом правый)
        if node.left:
            stack1.append(node.left)
        if node.right:
            stack1.append(node.right)

    # stack2 содержит узлы в обратном postorder порядке
    return [node.val for node in reversed(stack2)]
```

### Итеративная реализация (один стек)

```python
def postorder_one_stack(root):
    """Итеративный обратный обход с одним стеком."""
    if root is None:
        return []

    result = []
    stack = []
    current = root
    last_visited = None

    while current or stack:
        # Идём влево до упора
        while current:
            stack.append(current)
            current = current.left

        # Смотрим на вершину стека
        peek = stack[-1]

        # Если есть правый ребёнок и он ещё не обработан
        if peek.right and peek.right != last_visited:
            current = peek.right
        else:
            # Обрабатываем узел
            result.append(peek.val)
            last_visited = stack.pop()

    return result
```

### Morris-подобный обход (модифицированный)

```python
def postorder_morris(root):
    """
    Postorder с O(1) памяти.
    Используем dummy-узел и reverse-обход.
    """
    def reverse_list(head, tail):
        """Разворачивает часть списка (через right-ссылки)."""
        prev = None
        curr = head
        while prev != tail:
            next_node = curr.right
            curr.right = prev
            prev = curr
            curr = next_node

    def visit_reverse(from_node, to_node, result):
        """Добавляет узлы от from до to в обратном порядке."""
        reverse_list(from_node, to_node)
        curr = to_node
        while True:
            result.append(curr.val)
            if curr == from_node:
                break
            curr = curr.right
        reverse_list(to_node, from_node)

    result = []
    dummy = TreeNode(0)
    dummy.left = root
    current = dummy

    while current:
        if current.left is None:
            current = current.right
        else:
            predecessor = current.left
            while predecessor.right and predecessor.right != current:
                predecessor = predecessor.right

            if predecessor.right is None:
                predecessor.right = current
                current = current.left
            else:
                visit_reverse(current.left, predecessor, result)
                predecessor.right = None
                current = current.right

    return result
```

### Простой трюк: Preorder в обратном порядке

```python
def postorder_reverse_trick(root):
    """
    Postorder = обратный (Корень → Правое → Левое).
    Preorder NRL, затем reverse.
    """
    if root is None:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)

        # Сначала левый, потом правый (обратно preorder)
        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)

    return result[::-1]  # Разворачиваем
```

## Анализ сложности

| Метод | Время | Память |
|-------|-------|--------|
| Рекурсивный | O(n) | O(h) |
| Два стека | O(n) | O(n) |
| Один стек | O(n) | O(h) |
| Reverse trick | O(n) | O(n) |

## Пример с пошаговым разбором

### Обход дерева выражения

```
Дерево выражения: (2 + 3) * 4

        *
       / \
      +   4
     / \
    2   3

Postorder: 2, 3, +, 4, * (постфиксная/обратная польская нотация)
```

### Пошаговое вычисление постфиксного выражения

```
Выражение: 2 3 + 4 *
Стек вычислений:

Читаем 2: push 2          → stack = [2]
Читаем 3: push 3          → stack = [2, 3]
Читаем +: pop 3, 2; 2+3=5 → stack = [5]
Читаем 4: push 4          → stack = [5, 4]
Читаем *: pop 4, 5; 5*4=20 → stack = [20]

Результат: 20
```

### Удаление дерева

```python
def delete_tree(root):
    """Безопасное удаление дерева (postorder)."""
    if root is None:
        return

    # Сначала удаляем детей
    delete_tree(root.left)
    delete_tree(root.right)

    # Потом родителя
    print(f"Удаляю узел {root.val}")
    root.left = None
    root.right = None
    # В языках с GC узел будет удалён автоматически
```

### Вычисление высоты дерева

```python
def tree_height(root):
    """Высота дерева (postorder подход)."""
    if root is None:
        return -1  # или 0, в зависимости от определения

    # Сначала вычисляем высоту поддеревьев
    left_height = tree_height(root.left)
    right_height = tree_height(root.right)

    # Потом высоту текущего узла
    return max(left_height, right_height) + 1
```

### Вычисление размера поддерева

```python
def subtree_size(root):
    """Размер каждого поддерева."""
    sizes = {}

    def postorder(node):
        if node is None:
            return 0

        left_size = postorder(node.left)
        right_size = postorder(node.right)

        size = left_size + right_size + 1
        sizes[node.val] = size
        return size

    postorder(root)
    return sizes
```

## Типичные ошибки и Edge Cases

### 1. Пустое дерево

```python
def postorder_safe(root):
    if root is None:
        return []
    # ...
```

### 2. Один узел

```python
root = TreeNode(42)
print(postorder_recursive(root))  # [42]
```

### 3. Неправильный порядок в итеративной версии

```python
# ОШИБКА: обрабатываем узел до того, как обработали правого ребёнка
def wrong_postorder(root):
    stack = [root]
    result = []
    while stack:
        node = stack.pop()
        result.append(node.val)  # Слишком рано!
        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)
    return result
# Это даёт NRL (preorder справа), а не LRN (postorder)

# ПРАВИЛЬНО: используйте отслеживание last_visited
```

### 4. Проверка сбалансированности

```python
def is_balanced(root):
    """Проверка сбалансированности (postorder)."""
    def height(node):
        if node is None:
            return 0

        left_h = height(node.left)
        if left_h == -1:
            return -1

        right_h = height(node.right)
        if right_h == -1:
            return -1

        if abs(left_h - right_h) > 1:
            return -1

        return max(left_h, right_h) + 1

    return height(root) != -1
```

### 5. Сумма пути (от корня к листьям)

```python
def path_sum(root, target_sum):
    """Есть ли путь с заданной суммой (использует postorder для листьев)."""
    if root is None:
        return False

    # Проверяем лист
    if root.left is None and root.right is None:
        return root.val == target_sum

    # Проверяем поддеревья
    remaining = target_sum - root.val
    return (path_sum(root.left, remaining) or
            path_sum(root.right, remaining))
```

### 6. N-арное дерево

```python
class NaryNode:
    def __init__(self, val, children=None):
        self.val = val
        self.children = children or []

def postorder_nary(root):
    """Postorder для N-арного дерева."""
    if root is None:
        return []

    result = []
    for child in root.children:
        result.extend(postorder_nary(child))

    result.append(root.val)
    return result
```

### 7. Самый низкий общий предок (LCA)

```python
def lowest_common_ancestor(root, p, q):
    """
    Находит LCA двух узлов.
    Postorder: сначала проверяем детей, потом решаем для родителя.
    """
    if root is None or root == p or root == q:
        return root

    left = lowest_common_ancestor(root.left, p, q)
    right = lowest_common_ancestor(root.right, p, q)

    # Если оба найдены в разных поддеревьях — текущий узел LCA
    if left and right:
        return root

    # Иначе возвращаем непустой результат
    return left or right
```

### Рекомендации:
1. Postorder отлично подходит для удаления/освобождения ресурсов
2. Используйте для вычислений, зависящих от детей
3. Итеративная версия сложнее, чем preorder/inorder
4. Трюк с reverse(NRL) — простой способ получить postorder
5. Помните порядок: Левое → Правое → Корень

---
[prev: 02-inorder](./02-inorder.md) | [next: 04-level-order-bfs](./04-level-order-bfs.md)