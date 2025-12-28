# Сбалансированные деревья (Balanced Trees)

[prev: 03-binary-search-tree](./03-binary-search-tree.md) | [next: 05-heap](./05-heap.md)
---

## Определение

**Сбалансированные деревья** — это бинарные деревья поиска, которые автоматически поддерживают баланс (ограничение на разницу высот поддеревьев), гарантируя O(log n) для всех операций.

```
Несбалансированное (BST):     Сбалансированное (AVL):
        1                           4
         \                         / \
          2                       2   6
           \                     / \ / \
            3                   1  3 5  7
             \
              4
               \
                5

Height = n-1, O(n)            Height = log n, O(log n)
```

## Зачем нужно

### Проблема обычного BST:

```
Вставка 1, 2, 3, 4, 5 в BST:

insert(1): 1
insert(2): 1-2
insert(3): 1-2-3
insert(4): 1-2-3-4
insert(5): 1-2-3-4-5

Высота = 4, поиск = O(n)
```

### Решение — самобалансировка:

```
После каждой вставки/удаления дерево перестраивается,
сохраняя высоту O(log n).
```

---

## AVL-дерево

### Свойство AVL

Для каждого узла:
```
|height(left) - height(right)| ≤ 1
```

```
Balance Factor = height(left) - height(right)

Допустимые значения: -1, 0, +1

       4 [0]
      / \
   2 [0] 6 [0]
   / \   / \
  1   3 5   7
[-] [-] [-] [-]

[0] = сбалансирован
[+1] = левое поддерево выше
[-1] = правое поддерево выше
```

### Вращения (Rotations)

#### Правое вращение (Right Rotation)

```
Случай: Left-Left (вставка в левое поддерево левого ребёнка)

    z                    y
   / \                  / \
  y   T4    ──────►    x   z
 / \                  / \ / \
x   T3               T1 T2 T3 T4
/ \
T1 T2
```

```python
def rotate_right(z):
    y = z.left
    T3 = y.right

    y.right = z
    z.left = T3

    # Обновляем высоты
    z.height = 1 + max(get_height(z.left), get_height(z.right))
    y.height = 1 + max(get_height(y.left), get_height(y.right))

    return y  # новый корень
```

#### Левое вращение (Left Rotation)

```
Случай: Right-Right (вставка в правое поддерево правого ребёнка)

  z                      y
 / \                    / \
T1  y      ──────►     z   x
   / \                / \ / \
  T2  x              T1 T2 T3 T4
     / \
    T3  T4
```

```python
def rotate_left(z):
    y = z.right
    T2 = y.left

    y.left = z
    z.right = T2

    z.height = 1 + max(get_height(z.left), get_height(z.right))
    y.height = 1 + max(get_height(y.left), get_height(y.right))

    return y
```

#### Left-Right вращение

```
Случай: Left-Right (вставка в правое поддерево левого ребёнка)

     z                 z                  x
    / \               / \               /   \
   y   T4   ───►     x   T4   ───►     y     z
  / \               / \               / \   / \
 T1  x             y   T3            T1 T2 T3 T4
    / \           / \
   T2  T3        T1  T2

Step 1: Left rotate(y)    Step 2: Right rotate(z)
```

```python
def rotate_left_right(z):
    z.left = rotate_left(z.left)
    return rotate_right(z)
```

#### Right-Left вращение

```
Случай: Right-Left (вставка в левое поддерево правого ребёнка)

   z                   z                    x
  / \                 / \                 /   \
 T1  y      ───►     T1  x      ───►     z     y
    / \                 / \             / \   / \
   x   T4              T2  y           T1 T2 T3 T4
  / \                     / \
 T2  T3                  T3  T4

Step 1: Right rotate(y)   Step 2: Left rotate(z)
```

```python
def rotate_right_left(z):
    z.right = rotate_right(z.right)
    return rotate_left(z)
```

### Реализация AVL

```python
class AVLNode:
    def __init__(self, value):
        self.value = value
        self.left = None
        self.right = None
        self.height = 1

class AVLTree:
    def __init__(self):
        self.root = None

    def get_height(self, node):
        if node is None:
            return 0
        return node.height

    def get_balance(self, node):
        if node is None:
            return 0
        return self.get_height(node.left) - self.get_height(node.right)

    def insert(self, value):
        self.root = self._insert(self.root, value)

    def _insert(self, node, value):
        # Стандартная BST вставка
        if node is None:
            return AVLNode(value)

        if value < node.value:
            node.left = self._insert(node.left, value)
        elif value > node.value:
            node.right = self._insert(node.right, value)
        else:
            return node  # дубликаты не добавляем

        # Обновляем высоту
        node.height = 1 + max(
            self.get_height(node.left),
            self.get_height(node.right)
        )

        # Проверяем баланс
        balance = self.get_balance(node)

        # Left-Left
        if balance > 1 and value < node.left.value:
            return self.rotate_right(node)

        # Right-Right
        if balance < -1 and value > node.right.value:
            return self.rotate_left(node)

        # Left-Right
        if balance > 1 and value > node.left.value:
            node.left = self.rotate_left(node.left)
            return self.rotate_right(node)

        # Right-Left
        if balance < -1 and value < node.right.value:
            node.right = self.rotate_right(node.right)
            return self.rotate_left(node)

        return node

    def rotate_right(self, z):
        y = z.left
        T3 = y.right

        y.right = z
        z.left = T3

        z.height = 1 + max(self.get_height(z.left), self.get_height(z.right))
        y.height = 1 + max(self.get_height(y.left), self.get_height(y.right))

        return y

    def rotate_left(self, z):
        y = z.right
        T2 = y.left

        y.left = z
        z.right = T2

        z.height = 1 + max(self.get_height(z.left), self.get_height(z.right))
        y.height = 1 + max(self.get_height(y.left), self.get_height(y.right))

        return y
```

---

## Red-Black дерево

### Свойства Red-Black

1. Каждый узел **красный или чёрный**
2. Корень **всегда чёрный**
3. Все листья (NIL) **чёрные**
4. Красный узел имеет только **чёрных детей**
5. Любой путь от узла до листа содержит **одинаковое число чёрных узлов**

```
         B:8
        /   \
      R:3   R:10
      / \      \
   B:1  B:6   B:14
       / \    /
     R:4 R:7 R:13

B = Black (чёрный)
R = Red (красный)
```

### Почему это работает?

```
Правило 4 + 5 гарантирует:
- Нет двух красных узлов подряд
- Чёрная высота одинакова везде

Следствие:
Самый длинный путь ≤ 2 × самый короткий путь
Высота ≤ 2 × log(n+1)
```

### Вставка в Red-Black

```
1. Вставляем как в BST
2. Красим новый узел в КРАСНЫЙ
3. Исправляем нарушения:

Случай 1: Дядя красный
- Перекрашиваем родителя и дядю в чёрный
- Перекрашиваем деда в красный
- Рекурсивно проверяем деда

Случай 2: Дядя чёрный, узел — внутренний ребёнок
- Вращаем родителя

Случай 3: Дядя чёрный, узел — внешний ребёнок
- Вращаем деда
- Перекрашиваем родителя и деда
```

```python
class RBNode:
    RED = True
    BLACK = False

    def __init__(self, value):
        self.value = value
        self.left = None
        self.right = None
        self.parent = None
        self.color = RBNode.RED  # новые узлы всегда красные

class RedBlackTree:
    def __init__(self):
        self.NIL = RBNode(None)
        self.NIL.color = RBNode.BLACK
        self.root = self.NIL

    def insert(self, value):
        new_node = RBNode(value)
        new_node.left = self.NIL
        new_node.right = self.NIL

        # BST вставка
        parent = None
        current = self.root

        while current != self.NIL:
            parent = current
            if value < current.value:
                current = current.left
            else:
                current = current.right

        new_node.parent = parent

        if parent is None:
            self.root = new_node
        elif value < parent.value:
            parent.left = new_node
        else:
            parent.right = new_node

        # Исправляем нарушения
        self._fix_insert(new_node)

    def _fix_insert(self, node):
        while node.parent and node.parent.color == RBNode.RED:
            if node.parent == node.parent.parent.left:
                uncle = node.parent.parent.right

                if uncle.color == RBNode.RED:
                    # Случай 1: дядя красный
                    node.parent.color = RBNode.BLACK
                    uncle.color = RBNode.BLACK
                    node.parent.parent.color = RBNode.RED
                    node = node.parent.parent
                else:
                    if node == node.parent.right:
                        # Случай 2: узел — правый ребёнок
                        node = node.parent
                        self._rotate_left(node)

                    # Случай 3
                    node.parent.color = RBNode.BLACK
                    node.parent.parent.color = RBNode.RED
                    self._rotate_right(node.parent.parent)
            else:
                # Зеркальные случаи
                uncle = node.parent.parent.left

                if uncle.color == RBNode.RED:
                    node.parent.color = RBNode.BLACK
                    uncle.color = RBNode.BLACK
                    node.parent.parent.color = RBNode.RED
                    node = node.parent.parent
                else:
                    if node == node.parent.left:
                        node = node.parent
                        self._rotate_right(node)

                    node.parent.color = RBNode.BLACK
                    node.parent.parent.color = RBNode.RED
                    self._rotate_left(node.parent.parent)

        self.root.color = RBNode.BLACK

    def _rotate_left(self, x):
        y = x.right
        x.right = y.left

        if y.left != self.NIL:
            y.left.parent = x

        y.parent = x.parent

        if x.parent is None:
            self.root = y
        elif x == x.parent.left:
            x.parent.left = y
        else:
            x.parent.right = y

        y.left = x
        x.parent = y

    def _rotate_right(self, y):
        x = y.left
        y.left = x.right

        if x.right != self.NIL:
            x.right.parent = y

        x.parent = y.parent

        if y.parent is None:
            self.root = x
        elif y == y.parent.right:
            y.parent.right = x
        else:
            y.parent.left = x

        x.right = y
        y.parent = x
```

---

## Сравнение AVL и Red-Black

| Аспект | AVL | Red-Black |
|--------|-----|-----------|
| Баланс | Строгий (|h| ≤ 1) | Менее строгий |
| Высота | 1.44 × log n | 2 × log n |
| Поиск | Быстрее | Медленнее |
| Вставка | Больше вращений | Меньше вращений |
| Удаление | Больше вращений | Меньше вращений |
| Применение | Read-heavy | Write-heavy |
| Примеры | - | TreeMap, TreeSet (Java) |

### Когда что использовать

```
AVL:
- Поиск чаще, чем модификация
- Базы данных (read-heavy)
- Словари

Red-Black:
- Частые вставки/удаления
- Стандартные библиотеки (Java, C++)
- Ядро Linux
```

---

## B-деревья (обзор)

### Свойство B-дерева порядка m

```
1. Каждый узел имеет до m детей
2. Каждый узел (кроме корня и листьев) имеет ≥ m/2 детей
3. Корень имеет ≥ 2 детей (если не лист)
4. Все листья на одном уровне
5. Узел с k детьми содержит k-1 ключей
```

```
B-дерево порядка 3 (2-3 дерево):

         [30|70]
        /   |   \
    [10|20] [40|50|60] [80|90]
```

### Зачем B-деревья?

```
Оптимизация для дисковых операций:
- Узлы = блоки диска
- Меньше обращений к диску
- Широкие и неглубокие деревья

Применение:
- Файловые системы
- Индексы в базах данных
```

---

## Анализ сложности

| Операция | AVL | Red-Black | B-дерево |
|----------|-----|-----------|----------|
| Search | O(log n) | O(log n) | O(log n) |
| Insert | O(log n) | O(log n) | O(log n) |
| Delete | O(log n) | O(log n) | O(log n) |
| Max rotations (insert) | 2 | 2 | 0 |
| Max rotations (delete) | O(log n) | 3 | 0 |

---

## Типичные ошибки

### 1. Забытое обновление высоты

```python
# НЕПРАВИЛЬНО
def insert(self, node, value):
    ...
    return node  # высота не обновлена!

# ПРАВИЛЬНО
def insert(self, node, value):
    ...
    node.height = 1 + max(height(node.left), height(node.right))
    return node
```

### 2. Неправильный порядок операций при вращении

```python
# При вращении важен порядок!
# Сначала переназначаем связи, потом обновляем высоты

y = z.left
T3 = y.right

# Вращаем
y.right = z
z.left = T3

# ПОТОМ обновляем высоты (z раньше, он теперь ниже!)
z.height = ...
y.height = ...
```

### 3. Забытая балансировка после удаления

```python
# Удаление тоже может нарушить баланс!
# После удаления нужно проверить баланс и вращать
```

---

## Практические реализации

### Python

```python
# Нет встроенного сбалансированного дерева
# Используйте:
from sortedcontainers import SortedList, SortedDict

# Или bisect для отсортированного списка
import bisect
```

### Java

```java
// TreeMap — Red-Black дерево
TreeMap<Integer, String> map = new TreeMap<>();

// TreeSet — Red-Black дерево
TreeSet<Integer> set = new TreeSet<>();
```

### C++

```cpp
// std::map, std::set — Red-Black дерево
std::map<int, std::string> map;
std::set<int> set;
```

---
[prev: 03-binary-search-tree](./03-binary-search-tree.md) | [next: 05-heap](./05-heap.md)