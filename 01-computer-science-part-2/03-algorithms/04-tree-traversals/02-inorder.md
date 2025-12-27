# Симметричный обход дерева (Inorder Traversal)

## Определение

**Симметричный обход (Inorder)** — это способ обхода дерева, при котором сначала рекурсивно обходится левое поддерево, затем посещается сам узел, а потом рекурсивно обходится правое поддерево.

Порядок: **Левое поддерево → Корень → Правое поддерево** (LNR — Left, Node, Right)

Для **бинарного дерева поиска (BST)** симметричный обход даёт элементы в **отсортированном порядке**.

## Зачем нужен

### Области применения:
- **Получение отсортированных данных из BST** — главное применение
- **Инфиксная нотация выражений** — привычная математическая запись
- **Проверка BST** — элементы должны идти по возрастанию
- **Нахождение k-го элемента в BST**
- **Преобразование BST в отсортированный список**

### Преимущества:
- Для BST возвращает отсортированные элементы
- Естественен для выражений в инфиксной нотации
- Легко понять и реализовать

## Как работает

### Визуализация процесса

```
Дерево (BST):
        4
       / \
      2   6
     / \ / \
    1  3 5  7

Порядок обхода Inorder:

Шаг 1: Идём влево от 4 к 2, затем к 1
        4
       / \
      2   6
     / \ / \
   [1] 3 5  7

Шаг 2: У 1 нет левого ребёнка, посещаем 1, нет правого
        (продолжаем)

Шаг 3: Возвращаемся к 2, посещаем 2
        4
       / \
     [2]  6
     / \ / \
    1  3 5  7

Шаг 4: Идём в правое поддерево 2, посещаем 3
        4
       / \
      2   6
     / \ / \
    1 [3] 5  7

Шаг 5: Возвращаемся к 4, посещаем 4
       [4]
       / \
      2   6
     / \ / \
    1  3 5  7

Шаг 6: Идём в правое поддерево 4, затем влево к 5
        4
       / \
      2   6
     / \ / \
    1  3[5] 7

Шаг 7: Возвращаемся к 6, посещаем 6
        4
       / \
      2  [6]
     / \ / \
    1  3 5  7

Шаг 8: Идём в правое поддерево 6, посещаем 7
        4
       / \
      2   6
     / \ / \
    1  3 5 [7]

Результат: [1, 2, 3, 4, 5, 6, 7] — отсортировано!
```

### Схема обхода

```
              4 (4-й)
             / \
    (2-й)   2   6 (6-й)
           / \ / \
  (1-й)   1  3 5  7 (7-й)
            (3-й) (5-й)

Путь: идём максимально влево, посещаем, потом вправо
```

## Псевдокод

```
function inorder(node):
    if node is null:
        return

    inorder(node.left)       # Сначала левое поддерево
    visit(node)              # Обработка текущего узла
    inorder(node.right)      # Потом правое поддерево
```

### Рекурсивная реализация на Python

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def inorder_recursive(root):
    """Рекурсивный симметричный обход."""
    result = []

    def traverse(node):
        if node is None:
            return

        traverse(node.left)        # Левое
        result.append(node.val)    # Узел
        traverse(node.right)       # Правое

    traverse(root)
    return result
```

### Итеративная реализация со стеком

```python
def inorder_iterative(root):
    """Итеративный симметричный обход."""
    result = []
    stack = []
    current = root

    while current or stack:
        # Идём влево до упора
        while current:
            stack.append(current)
            current = current.left

        # Обрабатываем узел
        current = stack.pop()
        result.append(current.val)

        # Переходим в правое поддерево
        current = current.right

    return result
```

### Morris Traversal (O(1) память)

```python
def inorder_morris(root):
    """
    Симметричный обход с O(1) памяти.
    Использует временные связи к inorder-предшественникам.
    """
    result = []
    current = root

    while current:
        if current.left is None:
            # Нет левого поддерева — посещаем узел
            result.append(current.val)
            current = current.right
        else:
            # Находим inorder-предшественника
            predecessor = current.left
            while predecessor.right and predecessor.right != current:
                predecessor = predecessor.right

            if predecessor.right is None:
                # Создаём временную связь
                predecessor.right = current
                current = current.left
            else:
                # Связь уже есть — удаляем её и посещаем узел
                predecessor.right = None
                result.append(current.val)
                current = current.right

    return result
```

### Генератор

```python
def inorder_generator(root):
    """Симметричный обход как генератор."""
    if root is None:
        return

    yield from inorder_generator(root.left)
    yield root.val
    yield from inorder_generator(root.right)
```

## Анализ сложности

| Метод | Время | Память |
|-------|-------|--------|
| Рекурсивный | O(n) | O(h) |
| Итеративный | O(n) | O(h) |
| Morris | O(n) | O(1) |

Где h — высота дерева (O(log n) для сбалансированного, O(n) для вырожденного)

## Пример с пошаговым разбором

### Обход BST для получения отсортированных данных

```python
#     8
#    / \
#   3   10
#  / \    \
# 1   6    14
#    / \   /
#   4   7 13

# Inorder: [1, 3, 4, 6, 7, 8, 10, 13, 14]
```

### Пошаговый разбор итеративного алгоритма

```
Начало: current = 8, stack = []

1. Идём влево: 8 → 3 → 1
   stack = [8, 3, 1]
   current = None

2. Pop 1, visit, current = 1.right = None
   result = [1]
   stack = [8, 3]

3. Pop 3, visit, current = 3.right = 6
   result = [1, 3]
   stack = [8]

4. Идём влево от 6: 6 → 4
   stack = [8, 6, 4]
   current = None

5. Pop 4, visit, current = 4.right = None
   result = [1, 3, 4]
   stack = [8, 6]

6. Pop 6, visit, current = 6.right = 7
   result = [1, 3, 4, 6]
   stack = [8]

7. stack = [8, 7], current = None
   Pop 7, visit, current = 7.right = None
   result = [1, 3, 4, 6, 7]
   stack = [8]

8. Pop 8, visit, current = 8.right = 10
   result = [1, 3, 4, 6, 7, 8]
   stack = []

9. Идём влево от 10: нет левого
   stack = [10], current = None
   Pop 10, visit, current = 10.right = 14
   result = [1, 3, 4, 6, 7, 8, 10]

10. Идём влево от 14: 14 → 13
    stack = [14, 13], current = None
    Pop 13, visit, current = None
    result = [1, 3, 4, 6, 7, 8, 10, 13]
    Pop 14, visit, current = None
    result = [1, 3, 4, 6, 7, 8, 10, 13, 14]

Финал: [1, 3, 4, 6, 7, 8, 10, 13, 14]
```

### Проверка BST

```python
def is_valid_bst(root):
    """Проверка, является ли дерево BST через inorder."""
    prev = float('-inf')

    def inorder(node):
        nonlocal prev
        if node is None:
            return True

        # Левое поддерево
        if not inorder(node.left):
            return False

        # Текущий узел должен быть больше предыдущего
        if node.val <= prev:
            return False
        prev = node.val

        # Правое поддерево
        return inorder(node.right)

    return inorder(root)
```

### Нахождение k-го наименьшего элемента

```python
def kth_smallest(root, k):
    """Найти k-й наименьший элемент в BST."""
    stack = []
    current = root
    count = 0

    while current or stack:
        while current:
            stack.append(current)
            current = current.left

        current = stack.pop()
        count += 1
        if count == k:
            return current.val

        current = current.right

    return None
```

## Типичные ошибки и Edge Cases

### 1. Пустое дерево

```python
def inorder_safe(root):
    if root is None:
        return []
    # ...
```

### 2. Дерево из одного узла

```python
root = TreeNode(42)
print(inorder_recursive(root))  # [42]
```

### 3. Вырожденное дерево (всё влево или вправо)

```python
#     3
#    /
#   2
#  /
# 1
#
# Inorder: [1, 2, 3] — правильно

#  1
#   \
#    2
#     \
#      3
#
# Inorder: [1, 2, 3] — тоже правильно
```

### 4. Преобразование BST в отсортированный двусвязный список

```python
def bst_to_dll(root):
    """Преобразование BST в двусвязный список (inorder)."""
    if root is None:
        return None

    first = None  # Первый узел списка
    last = None   # Последний обработанный узел

    def inorder(node):
        nonlocal first, last

        if node is None:
            return

        inorder(node.left)

        if last is None:
            first = node
        else:
            last.right = node
            node.left = last
        last = node

        inorder(node.right)

    inorder(root)

    # Замыкаем в кольцо (опционально)
    # first.left = last
    # last.right = first

    return first
```

### 5. Обход дерева выражения

```python
#       +
#      / \
#     *   5
#    / \
#   2   3
#
# Inorder: 2 * 3 + 5 (инфиксная нотация)
# Но без скобок приоритет неправильный!

def inorder_expression(node):
    """Inorder с учётом приоритета операций."""
    if node is None:
        return ""

    if node.left is None and node.right is None:
        return str(node.val)

    left = inorder_expression(node.left)
    right = inorder_expression(node.right)

    return f"({left} {node.val} {right})"

# Результат: "((2 * 3) + 5)"
```

### 6. Итератор для BST

```python
class BSTIterator:
    """Итератор для BST, возвращающий элементы в порядке возрастания."""

    def __init__(self, root):
        self.stack = []
        self._push_left(root)

    def _push_left(self, node):
        while node:
            self.stack.append(node)
            node = node.left

    def next(self):
        node = self.stack.pop()
        self._push_left(node.right)
        return node.val

    def hasNext(self):
        return len(self.stack) > 0

# Использование
iterator = BSTIterator(root)
while iterator.hasNext():
    print(iterator.next())
```

### Рекомендации:
1. Inorder — ключевой обход для BST (отсортированный порядок)
2. Используйте для проверки корректности BST
3. Morris traversal экономит память, но изменяет дерево временно
4. Итератор полезен для lazy evaluation
5. Помните порядок: Левое → Корень → Правое
