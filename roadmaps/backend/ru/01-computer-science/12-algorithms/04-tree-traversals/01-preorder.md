# Прямой обход дерева (Preorder Traversal)

## Определение

**Прямой обход (Preorder)** — это способ обхода дерева, при котором для каждого узла сначала посещается сам узел, затем рекурсивно обходится левое поддерево, а потом правое поддерево.

Порядок: **Корень → Левое поддерево → Правое поддерево** (NLR — Node, Left, Right)

## Зачем нужен

### Области применения:
- **Копирование дерева** — создание точной копии структуры
- **Сериализация дерева** — сохранение дерева в файл/строку
- **Префиксная нотация выражений** — польская нотация
- **Построение дерева из выражения** — парсеры
- **Файловые системы** — обход директорий

### Преимущества:
- Корень обрабатывается первым
- Естественен для создания копии дерева
- Прост для понимания

## Как работает

### Визуализация процесса

```
Дерево:
        1
       / \
      2   3
     / \   \
    4   5   6

Порядок обхода Preorder:

Шаг 1: Посещаем корень 1
        [1]
       / \
      2   3
     / \   \
    4   5   6

Шаг 2: Идём в левое поддерево, посещаем 2
        1
       / \
     [2]  3
     / \   \
    4   5   6

Шаг 3: Идём в левое поддерево 2, посещаем 4
        1
       / \
      2   3
     / \   \
   [4]  5   6

Шаг 4: У 4 нет детей, возвращаемся, посещаем 5
        1
       / \
      2   3
     / \   \
    4  [5]  6

Шаг 5: Возвращаемся к 1, идём в правое поддерево, посещаем 3
        1
       / \
      2  [3]
     / \   \
    4   5   6

Шаг 6: Идём в правое поддерево 3, посещаем 6
        1
       / \
      2   3
     / \   \
    4   5  [6]

Результат: [1, 2, 4, 5, 3, 6]
```

### Схема обхода

```
              1 (1-й)
             / \
    (2-й)   2   3 (5-й)
           / \   \
  (3-й)   4   5   6 (6-й)
              (4-й)

Путь обхода:
1 → 2 → 4 → (возврат) → 5 → (возврат) → (возврат) → 3 → 6
```

## Псевдокод

```
function preorder(node):
    if node is null:
        return

    visit(node)              # Обработка текущего узла
    preorder(node.left)      # Рекурсия в левое поддерево
    preorder(node.right)     # Рекурсия в правое поддерево
```

### Рекурсивная реализация на Python

```python
class TreeNode:
    def __init__(self, val=0, left=None, right=None):
        self.val = val
        self.left = left
        self.right = right

def preorder_recursive(root):
    """Рекурсивный прямой обход."""
    result = []

    def traverse(node):
        if node is None:
            return

        result.append(node.val)    # Сначала узел
        traverse(node.left)        # Потом левое
        traverse(node.right)       # Потом правое

    traverse(root)
    return result
```

### Итеративная реализация со стеком

```python
def preorder_iterative(root):
    """Итеративный прямой обход со стеком."""
    if root is None:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)

        # Важно: сначала правый, потом левый (LIFO)
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result
```

### Morris Traversal (O(1) память)

```python
def preorder_morris(root):
    """
    Прямой обход с O(1) памяти.
    Использует "threading" — временные связи к предшественникам.
    """
    result = []
    current = root

    while current:
        if current.left is None:
            result.append(current.val)
            current = current.right
        else:
            # Находим предшественника
            predecessor = current.left
            while predecessor.right and predecessor.right != current:
                predecessor = predecessor.right

            if predecessor.right is None:
                # Создаём временную связь
                result.append(current.val)
                predecessor.right = current
                current = current.left
            else:
                # Удаляем временную связь
                predecessor.right = None
                current = current.right

    return result
```

### Генератор для экономии памяти

```python
def preorder_generator(root):
    """Прямой обход как генератор."""
    if root is None:
        return

    yield root.val
    yield from preorder_generator(root.left)
    yield from preorder_generator(root.right)

# Использование
for val in preorder_generator(root):
    print(val)
```

## Анализ сложности

| Метод | Время | Память |
|-------|-------|--------|
| Рекурсивный | O(n) | O(h) — высота дерева |
| Итеративный | O(n) | O(h) — стек |
| Morris | O(n) | O(1) |

Где:
- n — количество узлов
- h — высота дерева
- h = O(log n) для сбалансированного дерева
- h = O(n) для вырожденного дерева

## Пример с пошаговым разбором

### Обход дерева выражения (2 + 3) * 4

```
Дерево:
        *
       / \
      +   4
     / \
    2   3

Preorder: * + 2 3 4 (префиксная нотация)

Пошаговый разбор:

Стек: [*]
Шаг 1: pop *, visit *, push 4, push +
       result = [*], stack = [4, +]

Шаг 2: pop +, visit +, push 3, push 2
       result = [*, +], stack = [4, 3, 2]

Шаг 3: pop 2, visit 2, нет детей
       result = [*, +, 2], stack = [4, 3]

Шаг 4: pop 3, visit 3, нет детей
       result = [*, +, 2, 3], stack = [4]

Шаг 5: pop 4, visit 4, нет детей
       result = [*, +, 2, 3, 4], stack = []

Результат: [*, +, 2, 3, 4]
Это префиксная нотация: * + 2 3 4 = (2 + 3) * 4 = 20
```

### Копирование дерева

```python
def clone_tree(root):
    """Создание копии дерева используя preorder."""
    if root is None:
        return None

    # Preorder: сначала создаём узел, потом детей
    new_node = TreeNode(root.val)
    new_node.left = clone_tree(root.left)
    new_node.right = clone_tree(root.right)

    return new_node
```

### Сериализация и десериализация

```python
def serialize(root):
    """Сериализация дерева в строку (preorder)."""
    result = []

    def preorder(node):
        if node is None:
            result.append('null')
            return
        result.append(str(node.val))
        preorder(node.left)
        preorder(node.right)

    preorder(root)
    return ','.join(result)

def deserialize(data):
    """Восстановление дерева из строки."""
    values = iter(data.split(','))

    def build():
        val = next(values)
        if val == 'null':
            return None
        node = TreeNode(int(val))
        node.left = build()
        node.right = build()
        return node

    return build()

# Пример
#     1
#    / \
#   2   3
#      / \
#     4   5
#
# serialize: "1,2,null,null,3,4,null,null,5,null,null"
```

## Типичные ошибки и Edge Cases

### 1. Пустое дерево

```python
def preorder_safe(root):
    if root is None:
        return []
    # ...
```

### 2. Дерево из одного узла

```python
# Должно работать корректно
root = TreeNode(42)
print(preorder_recursive(root))  # [42]
```

### 3. Вырожденное дерево (связанный список)

```python
# Дерево-"лоза":
#     1
#      \
#       2
#        \
#         3
#
# Preorder: [1, 2, 3]
# Рекурсия может вызвать Stack Overflow при большой глубине
# Решение: использовать итеративный подход
```

### 4. Неправильный порядок добавления в стек

```python
# ОШИБКА: неправильный порядок
def wrong_preorder(root):
    stack = [root]
    result = []
    while stack:
        node = stack.pop()
        result.append(node.val)
        if node.left:
            stack.append(node.left)   # Левый добавляется первым
        if node.right:
            stack.append(node.right)  # Правый добавляется последним
    return result
# Результат будет: правый обход (NRL), а не левый (NLR)!

# ПРАВИЛЬНО: правый добавляется ПЕРВЫМ
def correct_preorder(root):
    stack = [root]
    result = []
    while stack:
        node = stack.pop()
        result.append(node.val)
        if node.right:
            stack.append(node.right)  # Правый добавляется первым
        if node.left:
            stack.append(node.left)   # Левый добавляется последним
    return result
```

### 5. Обход N-арного дерева

```python
class NaryNode:
    def __init__(self, val, children=None):
        self.val = val
        self.children = children or []

def preorder_nary(root):
    """Preorder для N-арного дерева."""
    if root is None:
        return []

    result = [root.val]
    for child in root.children:
        result.extend(preorder_nary(child))

    return result
```

### Рекомендации:
1. Preorder отлично подходит для копирования/клонирования деревьев
2. Используйте для сериализации — легко восстановить структуру
3. Для больших деревьев предпочтите итеративный подход
4. Morris traversal экономит память, но сложнее в понимании
5. Помните порядок: Корень → Левое → Правое
