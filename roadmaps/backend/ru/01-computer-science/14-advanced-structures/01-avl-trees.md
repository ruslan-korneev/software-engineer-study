# AVL-деревья (AVL Trees)

[prev: 07-classic-np-problems](../13-complexity-classes/07-classic-np-problems.md) | [next: 02-red-black-trees](./02-red-black-trees.md)
---

## Определение

**AVL-дерево** — это самобалансирующееся бинарное дерево поиска (BST), названное в честь его изобретателей Адельсона-Вельского и Ландиса (1962). Главная особенность: для каждого узла высоты левого и правого поддеревьев отличаются не более чем на 1.

```
        Сбалансированное AVL            Несбалансированное BST
              30                              30
             /  \                            /
            20   40                         20
           /  \    \                       /
          10  25   50                     10
                                         /
                                        5
```

## Зачем нужно

### Проблема обычного BST
Обычное бинарное дерево поиска может выродиться в связный список при вставке отсортированных данных:

```
Вставка: 1, 2, 3, 4, 5

BST:        AVL:
  1           2
   \         / \
    2       1   4
     \         / \
      3       3   5
       \
        4
         \
          5

O(n) поиск    O(log n) поиск
```

### Применение
- **Базы данных в памяти** — когда нужны гарантированные O(log n) операции
- **Словари и множества** — реализация `std::set`, `std::map` в некоторых STL
- **Индексы в памяти** — когда важна скорость чтения
- **Файловые системы** — организация метаданных
- **Компиляторы** — таблицы символов

## Как работает

### Фактор баланса (Balance Factor)

Для каждого узла вычисляется **фактор баланса**:
```
BF(node) = height(left_subtree) - height(right_subtree)
```

В AVL-дереве: **BF ∈ {-1, 0, 1}**

```
           30 (BF=1)
          /  \
   (BF=0) 20   40 (BF=0)
         /
  (BF=0) 10

height(left of 30) = 2
height(right of 30) = 1
BF(30) = 2 - 1 = 1 ✓
```

### Ротации

При нарушении баланса выполняются ротации:

#### 1. Левая ротация (Left Rotation) — случай RR

Когда правое поддерево слишком глубокое справа:

```
    x                      y
     \                    / \
      y        =>        x   z
       \
        z

Детально:
    10 (BF=-2)                 20
      \                       /  \
       20 (BF=-1)    =>     10    30
        \
         30
```

#### 2. Правая ротация (Right Rotation) — случай LL

Когда левое поддерево слишком глубокое слева:

```
        z                  y
       /                  / \
      y        =>        x   z
     /
    x

Детально:
         30 (BF=2)            20
        /                    /  \
       20 (BF=1)    =>     10    30
      /
     10
```

#### 3. Лево-правая ротация (Left-Right) — случай LR

Когда левое поддерево глубокое справа:

```
      z                 z                  y
     /                 /                  / \
    x       =>        y        =>        x   z
     \               /
      y             x

Детально:
       30 (BF=2)         30              20
      /                 /               /  \
     10 (BF=-1)  =>   20        =>    10    30
      \              /
       20           10
```

#### 4. Право-левая ротация (Right-Left) — случай RL

Когда правое поддерево глубокое слева:

```
    x                 x                    y
     \                 \                  / \
      z       =>        y        =>      x   z
     /                   \
    y                     z

Детально:
     10 (BF=-2)         10               20
      \                   \             /  \
       30 (BF=1)  =>       20    =>   10    30
      /                     \
     20                      30
```

### Алгоритм определения типа ротации

```
Если BF(node) > 1:           # Левое поддерево перевешивает
    Если BF(left_child) >= 0:
        Правая ротация (LL)
    Иначе:
        Лево-правая ротация (LR)

Если BF(node) < -1:          # Правое поддерево перевешивает
    Если BF(right_child) <= 0:
        Левая ротация (RR)
    Иначе:
        Право-левая ротация (RL)
```

## Инварианты и свойства

### Инварианты AVL
1. **Свойство BST**: левый потомок < родитель < правый потомок
2. **Свойство баланса**: |BF(node)| ≤ 1 для всех узлов
3. **Высота**: h = O(log n), точнее h ≤ 1.44 * log₂(n+2)

### Связь с числами Фибоначчи

Минимальное число узлов в AVL-дереве высоты h:
```
N(h) = N(h-1) + N(h-2) + 1

N(0) = 1
N(1) = 2
N(2) = 4
N(3) = 7
N(4) = 12
...
```

Это даёт: n ≥ Fib(h+2) - 1, где Fib — числа Фибоначчи.

## Псевдокод основных операций

### Структура узла

```python
class AVLNode:
    def __init__(self, key):
        self.key = key
        self.left = None
        self.right = None
        self.height = 1  # Высота поддерева с корнем в этом узле
```

### Вспомогательные функции

```python
def height(node):
    if node is None:
        return 0
    return node.height

def balance_factor(node):
    if node is None:
        return 0
    return height(node.left) - height(node.right)

def update_height(node):
    node.height = 1 + max(height(node.left), height(node.right))
```

### Ротации

```python
def rotate_right(z):
    """
         z                y
        / \              / \
       y   T4    =>     x   z
      / \                  / \
     x   T3               T3  T4
    """
    y = z.left
    T3 = y.right

    # Выполняем ротацию
    y.right = z
    z.left = T3

    # Обновляем высоты (порядок важен!)
    update_height(z)
    update_height(y)

    return y  # Новый корень

def rotate_left(z):
    """
       z                   y
      / \                 / \
     T1  y       =>      z   x
        / \             / \
       T2  x           T1  T2
    """
    y = z.right
    T2 = y.left

    # Выполняем ротацию
    y.left = z
    z.right = T2

    # Обновляем высоты
    update_height(z)
    update_height(y)

    return y  # Новый корень
```

### Балансировка

```python
def rebalance(node):
    update_height(node)
    bf = balance_factor(node)

    # Левое поддерево перевешивает
    if bf > 1:
        # Случай LR: сначала левая ротация на левом ребёнке
        if balance_factor(node.left) < 0:
            node.left = rotate_left(node.left)
        # Случай LL (или после LR): правая ротация
        return rotate_right(node)

    # Правое поддерево перевешивает
    if bf < -1:
        # Случай RL: сначала правая ротация на правом ребёнке
        if balance_factor(node.right) > 0:
            node.right = rotate_right(node.right)
        # Случай RR (или после RL): левая ротация
        return rotate_left(node)

    return node
```

### Вставка

```python
def insert(node, key):
    # 1. Обычная вставка BST
    if node is None:
        return AVLNode(key)

    if key < node.key:
        node.left = insert(node.left, key)
    elif key > node.key:
        node.right = insert(node.right, key)
    else:
        return node  # Дубликаты не разрешены

    # 2. Балансировка после вставки
    return rebalance(node)
```

### Поиск

```python
def search(node, key):
    # Стандартный поиск в BST (балансировка не нужна)
    if node is None:
        return None

    if key == node.key:
        return node
    elif key < node.key:
        return search(node.left, key)
    else:
        return search(node.right, key)
```

### Удаление

```python
def find_min(node):
    """Находит узел с минимальным ключом"""
    current = node
    while current.left is not None:
        current = current.left
    return current

def delete(node, key):
    # 1. Обычное удаление BST
    if node is None:
        return None

    if key < node.key:
        node.left = delete(node.left, key)
    elif key > node.key:
        node.right = delete(node.right, key)
    else:
        # Узел найден
        if node.left is None:
            return node.right
        elif node.right is None:
            return node.left

        # Узел с двумя детьми: заменяем на inorder-преемника
        successor = find_min(node.right)
        node.key = successor.key
        node.right = delete(node.right, successor.key)

    # 2. Балансировка после удаления
    return rebalance(node)
```

### Пример работы вставки

```
Вставляем: 10, 20, 30, 40, 50, 25

1. insert(10)        2. insert(20)       3. insert(30) — нужна ротация!
      10                  10                  10 (BF=-2)
                           \                    \
                            20                   20 (BF=-1)
                                                  \
                                                   30

                              Левая ротация:
                                   20
                                  /  \
                                 10   30

4. insert(40)              5. insert(50) — ротация!
      20                          20 (BF=-2)
     /  \                        /  \
    10   30                     10   30 (BF=-2)
          \                           \
           40                          40
                                        \
                                         50

                         Левая ротация на 30:
                              20
                             /  \
                            10   40
                                /  \
                               30   50

6. insert(25) — LR ротация!
      20 (BF=-2)
     /  \
    10   40 (BF=1)
        /  \
       30   50
      /
     25

  Право-левая ротация:
        30
       /  \
      20   40
     /  \    \
    10  25   50
```

## Анализ сложности

| Операция | Среднее | Худшее  |
|----------|---------|---------|
| Поиск    | O(log n)| O(log n)|
| Вставка  | O(log n)| O(log n)|
| Удаление | O(log n)| O(log n)|
| Память   | O(n)    | O(n)    |

### Константы

- Вставка требует **максимум 2 ротации**
- Удаление может потребовать **O(log n) ротаций** (по одной на каждом уровне)
- Каждая ротация — O(1)

### Высота дерева

Для n узлов высота AVL-дерева:
```
h ≤ 1.44 * log₂(n + 2) - 0.328
```

Это примерно на 44% больше идеально сбалансированного дерева, но всё равно O(log n).

## Сравнение с альтернативами

| Свойство | AVL | Red-Black | Splay | Skip List |
|----------|-----|-----------|-------|-----------|
| Поиск (худш.) | O(log n) | O(log n) | O(n) | O(n) |
| Вставка (худш.) | O(log n) | O(log n) | O(n) | O(n) |
| Высота | ≤1.44 log n | ≤2 log n | — | — |
| Ротации при вставке | ≤2 | ≤2 | O(log n) | — |
| Ротации при удалении | O(log n) | ≤3 | O(log n) | — |

### Когда выбирать AVL

**Предпочитать AVL:**
- Много операций поиска, мало модификаций
- Нужна строгая гарантия баланса
- Важна предсказуемость времени отклика

**Предпочитать Red-Black:**
- Много вставок и удалений
- Допустима чуть большая глубина дерева
- Используется в стандартных библиотеках (std::map, TreeMap)

## Примеры использования в реальных системах

### 1. Linux Kernel — виртуальная память
AVL-деревья использовались для управления регионами памяти процессов (до перехода на Red-Black).

### 2. Базы данных в памяти
- **H2 Database** — использует AVL для индексов в памяти
- **SQLite** — опционально использует AVL для определённых структур

### 3. Словари и таблицы символов
```python
# Пример: таблица символов компилятора
class SymbolTable:
    def __init__(self):
        self.root = None

    def declare(self, name, symbol_info):
        self.root = insert(self.root, name, symbol_info)

    def lookup(self, name):
        return search(self.root, name)
```

### 4. Реализации множеств
- **Windows NT** — использовал AVL для Generic Table
- **C++ STL** (некоторые реализации) — для std::set

### 5. Геометрические алгоритмы
AVL применяется в алгоритмах вычислительной геометрии для поддержания отсортированного множества событий (sweep line algorithms).

## Резюме

AVL-дерево — это классическая самобалансирующаяся структура данных с гарантированной высотой O(log n). Оно обеспечивает более строгий баланс, чем Red-Black дерево, что делает его оптимальным для сценариев с преобладанием операций чтения. Основной механизм балансировки — четыре типа ротаций (LL, RR, LR, RL), которые восстанавливают инвариант |BF| ≤ 1 после каждой модификации.

---
[prev: 07-classic-np-problems](../13-complexity-classes/07-classic-np-problems.md) | [next: 02-red-black-trees](./02-red-black-trees.md)