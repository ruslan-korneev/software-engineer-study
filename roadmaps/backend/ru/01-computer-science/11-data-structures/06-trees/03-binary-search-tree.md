# Двоичное дерево поиска (Binary Search Tree)

[prev: 02-binary-tree](./02-binary-tree.md) | [next: 04-balanced-trees](./04-balanced-trees.md)

---
## Определение

**Двоичное дерево поиска (BST)** — это бинарное дерево со свойством упорядоченности: для каждого узла все значения в левом поддереве меньше значения узла, а все значения в правом поддереве больше.

```
         ┌───┐
         │ 8 │
         └─┬─┘
      ┌────┴────┐
    ┌─▼─┐     ┌─▼─┐
    │ 3 │     │ 10│
    └─┬─┘     └─┬─┘
   ┌──┴──┐      └──┐
 ┌─▼─┐ ┌─▼─┐    ┌──▼─┐
 │ 1 │ │ 6 │    │ 14 │
 └───┘ └─┬─┘    └─┬──┘
       ┌─┴─┐    ┌─┴─┐
     ┌─▼─┐┌▼─┐ ┌▼─┐
     │ 4 ││ 7││ 13│
     └───┘└──┘└───┘

Свойство BST:
left.value < node.value < right.value
```

## Зачем нужно

### Преимущества:
- **Быстрый поиск** — O(log n) в среднем
- **Упорядоченность** — inorder обход даёт отсортированную последовательность
- **Динамические операции** — эффективная вставка и удаление

### Практическое применение:
- **Поиск** — словари, телефонные книги
- **Диапазонные запросы** — найти все элементы от A до B
- **Автодополнение** — поиск по префиксу
- **Основа для** — AVL, Red-Black деревьев, TreeMap/TreeSet

## Структура

```python
class TreeNode:
    def __init__(self, value):
        self.value = value
        self.left = None
        self.right = None

class BST:
    def __init__(self):
        self.root = None
```

## Основные операции

### Поиск (Search)

```python
def search(self, value):
    """
    Поиск значения в BST.
    Время: O(log n) в среднем, O(n) в худшем
    """
    return self._search(self.root, value)

def _search(self, node, value):
    if node is None:
        return None

    if value == node.value:
        return node
    elif value < node.value:
        return self._search(node.left, value)
    else:
        return self._search(node.right, value)

# Итеративная версия
def search_iterative(self, value):
    current = self.root

    while current is not None:
        if value == current.value:
            return current
        elif value < current.value:
            current = current.left
        else:
            current = current.right

    return None
```

```
Поиск значения 6:

         8
        / \
       3   10
      / \    \
     1   6    14

Шаг 1: 6 < 8 → идём влево
Шаг 2: 6 > 3 → идём вправо
Шаг 3: 6 == 6 → найдено!
```

### Вставка (Insert)

```python
def insert(self, value):
    """
    Вставка нового значения.
    Время: O(log n) в среднем, O(n) в худшем
    """
    self.root = self._insert(self.root, value)

def _insert(self, node, value):
    if node is None:
        return TreeNode(value)

    if value < node.value:
        node.left = self._insert(node.left, value)
    elif value > node.value:
        node.right = self._insert(node.right, value)
    # Если value == node.value, можно игнорировать или обновить

    return node

# Итеративная версия
def insert_iterative(self, value):
    new_node = TreeNode(value)

    if self.root is None:
        self.root = new_node
        return

    current = self.root
    while True:
        if value < current.value:
            if current.left is None:
                current.left = new_node
                return
            current = current.left
        else:
            if current.right is None:
                current.right = new_node
                return
            current = current.right
```

```
Вставка значения 5:

         8                      8
        / \                    / \
       3   10       →         3   10
      / \    \               / \    \
     1   6    14            1   6    14
                               /
                              5
```

### Удаление (Delete)

Три случая:
1. **Узел — лист**: просто удаляем
2. **Узел с одним ребёнком**: заменяем узел ребёнком
3. **Узел с двумя детьми**: заменяем значением inorder-преемника или предшественника

```python
def delete(self, value):
    """
    Удаление значения из BST.
    Время: O(log n) в среднем
    """
    self.root = self._delete(self.root, value)

def _delete(self, node, value):
    if node is None:
        return None

    if value < node.value:
        node.left = self._delete(node.left, value)
    elif value > node.value:
        node.right = self._delete(node.right, value)
    else:
        # Нашли узел для удаления

        # Случай 1: лист или один ребёнок
        if node.left is None:
            return node.right
        if node.right is None:
            return node.left

        # Случай 2: два ребёнка
        # Находим минимум в правом поддереве (inorder successor)
        successor = self._find_min(node.right)
        node.value = successor.value
        node.right = self._delete(node.right, successor.value)

    return node

def _find_min(self, node):
    """Находит узел с минимальным значением"""
    while node.left is not None:
        node = node.left
    return node
```

```
Удаление узла с двумя детьми (значение 3):

         8                      8
        / \                    / \
       3   10       →         4   10
      / \    \               / \    \
     1   6    14            1   6    14
        /                       /
       4                       5
      /
     5

Шаг 1: Находим inorder successor (4)
Шаг 2: Заменяем 3 на 4
Шаг 3: Удаляем старую 4
```

### Минимум и максимум

```python
def find_min(self):
    """Минимальный элемент — самый левый узел"""
    if self.root is None:
        return None

    current = self.root
    while current.left is not None:
        current = current.left
    return current.value

def find_max(self):
    """Максимальный элемент — самый правый узел"""
    if self.root is None:
        return None

    current = self.root
    while current.right is not None:
        current = current.right
    return current.value
```

### Inorder Traversal (отсортированный порядок)

```python
def inorder(self):
    """
    Inorder обход BST даёт элементы в отсортированном порядке!
    """
    result = []
    self._inorder(self.root, result)
    return result

def _inorder(self, node, result):
    if node is not None:
        self._inorder(node.left, result)
        result.append(node.value)
        self._inorder(node.right, result)
```

### Предшественник и преемник

```python
def inorder_successor(self, value):
    """
    Следующий элемент в отсортированном порядке.
    """
    node = self.search(value)
    if node is None:
        return None

    # Если есть правое поддерево — минимум в нём
    if node.right is not None:
        return self._find_min(node.right).value

    # Иначе — поднимаемся вверх
    successor = None
    current = self.root

    while current is not None:
        if value < current.value:
            successor = current
            current = current.left
        elif value > current.value:
            current = current.right
        else:
            break

    return successor.value if successor else None

def inorder_predecessor(self, value):
    """Предыдущий элемент в отсортированном порядке."""
    node = self.search(value)
    if node is None:
        return None

    # Если есть левое поддерево — максимум в нём
    if node.left is not None:
        return self._find_max(node.left).value

    # Иначе — поднимаемся вверх
    predecessor = None
    current = self.root

    while current is not None:
        if value > current.value:
            predecessor = current
            current = current.right
        elif value < current.value:
            current = current.left
        else:
            break

    return predecessor.value if predecessor else None
```

## Полная реализация BST

```python
class BST:
    def __init__(self):
        self.root = None
        self._size = 0

    def insert(self, value):
        if self.root is None:
            self.root = TreeNode(value)
        else:
            self._insert(self.root, value)
        self._size += 1

    def _insert(self, node, value):
        if value < node.value:
            if node.left is None:
                node.left = TreeNode(value)
            else:
                self._insert(node.left, value)
        elif value > node.value:
            if node.right is None:
                node.right = TreeNode(value)
            else:
                self._insert(node.right, value)

    def search(self, value):
        return self._search(self.root, value) is not None

    def _search(self, node, value):
        if node is None or node.value == value:
            return node
        if value < node.value:
            return self._search(node.left, value)
        return self._search(node.right, value)

    def delete(self, value):
        self.root = self._delete(self.root, value)
        self._size -= 1

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

            min_node = node.right
            while min_node.left:
                min_node = min_node.left
            node.value = min_node.value
            node.right = self._delete(node.right, min_node.value)

        return node

    def __len__(self):
        return self._size

    def __contains__(self, value):
        return self.search(value)
```

## Примеры с разбором

### Пример 1: Валидация BST

```python
def is_valid_bst(root, min_val=float('-inf'), max_val=float('inf')):
    """
    Проверяет, является ли дерево корректным BST.
    Важно: проверять не только соседей, но и глобальные границы!
    """
    if root is None:
        return True

    if root.value <= min_val or root.value >= max_val:
        return False

    return (is_valid_bst(root.left, min_val, root.value) and
            is_valid_bst(root.right, root.value, max_val))

# Альтернатива: проверка через inorder traversal
def is_valid_bst_inorder(root):
    prev = [float('-inf')]

    def inorder(node):
        if node is None:
            return True

        if not inorder(node.left):
            return False

        if node.value <= prev[0]:
            return False
        prev[0] = node.value

        return inorder(node.right)

    return inorder(root)
```

### Пример 2: Диапазонный запрос

```python
def range_query(root, low, high):
    """
    Находит все значения в диапазоне [low, high].
    Время: O(h + k), где k — количество элементов в диапазоне
    """
    result = []

    def traverse(node):
        if node is None:
            return

        if node.value > low:
            traverse(node.left)

        if low <= node.value <= high:
            result.append(node.value)

        if node.value < high:
            traverse(node.right)

    traverse(root)
    return result

# Пример:
#       8
#      / \
#     3   10
#    / \    \
#   1   6    14

# range_query(root, 4, 10) → [6, 8, 10]
```

### Пример 3: K-й наименьший элемент

```python
def kth_smallest(root, k):
    """
    Находит k-й наименьший элемент.
    Время: O(h + k)
    """
    stack = []
    current = root
    count = 0

    while stack or current:
        while current:
            stack.append(current)
            current = current.left

        current = stack.pop()
        count += 1

        if count == k:
            return current.value

        current = current.right

    return None  # k > размера дерева
```

### Пример 4: Конвертация в отсортированный связный список

```python
def bst_to_sorted_list(root):
    """
    Конвертирует BST в отсортированный двусвязный список.
    Модифицирует дерево на месте.
    """
    if root is None:
        return None

    head = [None]
    prev = [None]

    def inorder(node):
        if node is None:
            return

        inorder(node.left)

        if prev[0] is None:
            head[0] = node
        else:
            prev[0].right = node
            node.left = prev[0]

        prev[0] = node

        inorder(node.right)

    inorder(root)

    # Замыкаем в кольцо (опционально)
    # head[0].left = prev[0]
    # prev[0].right = head[0]

    return head[0]
```

### Пример 5: Построение сбалансированного BST из отсортированного массива

```python
def sorted_array_to_bst(nums):
    """
    Создаёт сбалансированное BST из отсортированного массива.
    Время: O(n)
    """
    if not nums:
        return None

    mid = len(nums) // 2
    root = TreeNode(nums[mid])

    root.left = sorted_array_to_bst(nums[:mid])
    root.right = sorted_array_to_bst(nums[mid + 1:])

    return root

# nums = [1, 2, 3, 4, 5, 6, 7]
#
# Результат:
#       4
#      / \
#     2   6
#    / \ / \
#   1  3 5  7
```

## Анализ сложности

| Операция | Среднее | Худшее | Лучшее |
|----------|---------|--------|--------|
| Search | O(log n) | O(n) | O(1) |
| Insert | O(log n) | O(n) | O(1) |
| Delete | O(log n) | O(n) | O(log n) |
| Min/Max | O(log n) | O(n) | O(1) |
| Inorder | O(n) | O(n) | O(n) |

### Когда худший случай?

```
Вставка отсортированной последовательности:
insert(1), insert(2), insert(3), insert(4), insert(5)

Результат — вырожденное дерево (связный список):
1
 \
  2
   \
    3
     \
      4
       \
        5

Высота = n - 1, все операции O(n)
```

## Типичные ошибки

### 1. Неправильная проверка BST

```python
# НЕПРАВИЛЬНО — проверяет только соседей
def is_bst_wrong(node):
    if node is None:
        return True

    if node.left and node.left.value > node.value:
        return False
    if node.right and node.right.value < node.value:
        return False

    return is_bst_wrong(node.left) and is_bst_wrong(node.right)

# Пропустит некорректное дерево:
#       10
#      /  \
#     5    15
#    / \
#   1   20  ← 20 > 10, но эта проверка не отловит!

# ПРАВИЛЬНО — передаём допустимые границы
def is_bst_correct(node, min_val=float('-inf'), max_val=float('inf')):
    if node is None:
        return True

    if node.value <= min_val or node.value >= max_val:
        return False

    return (is_bst_correct(node.left, min_val, node.value) and
            is_bst_correct(node.right, node.value, max_val))
```

### 2. Забытое обновление при удалении

```python
# НЕПРАВИЛЬНО
def delete(node, value):
    if value < node.value:
        delete(node.left, value)  # не присваиваем результат!
    ...

# ПРАВИЛЬНО
def delete(node, value):
    if value < node.value:
        node.left = delete(node.left, value)  # присваиваем!
    ...
```

### 3. Дубликаты

```python
# BST обычно не хранит дубликаты или хранит их в одной ветке

# Вариант 1: игнорируем дубликаты
if value == node.value:
    return node  # ничего не делаем

# Вариант 2: храним счётчик
class TreeNode:
    def __init__(self, value):
        self.value = value
        self.count = 1  # количество вхождений
        self.left = None
        self.right = None

# Вариант 3: храним в правом поддереве (< и >=)
if value < node.value:
    # левое поддерево
else:
    # правое поддерево (включая равные)
```

## Улучшения BST

### Проблема вырождения

```
Решения:
1. AVL-деревья — балансировка через вращения
2. Red-Black деревья — гарантия O(log n)
3. Splay-деревья — самобалансировка через доступ
4. Treap — рандомизированная балансировка
```

### Когда использовать BST

**Используй BST, когда:**
- Нужен упорядоченный набор данных
- Частые операции поиска
- Нужны диапазонные запросы
- Нужны min/max, successor/predecessor

**НЕ используй обычный BST, когда:**
- Данные поступают отсортированно (используй сбалансированное дерево)
- Нужен только поиск (хэш-таблица быстрее)
- Мало данных (линейный поиск может быть быстрее)

---

[prev: 02-binary-tree](./02-binary-tree.md) | [next: 04-balanced-trees](./04-balanced-trees.md)
