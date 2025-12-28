# Красно-чёрные деревья (Red-Black Trees)

[prev: 01-avl-trees](./01-avl-trees.md) | [next: 03-b-trees](./03-b-trees.md)
---

## Определение

**Красно-чёрное дерево** (Red-Black Tree, RB-Tree) — это самобалансирующееся бинарное дерево поиска, в котором каждый узел имеет дополнительный бит для хранения "цвета" (красный или чёрный). Цвета используются для поддержания приблизительной сбалансированности дерева.

```
Пример красно-чёрного дерева:

           13(B)
          /    \
       8(R)     17(R)
       /  \     /   \
    1(B) 11(B) 15(B) 25(B)
      \               /
       6(R)         22(R)

B = Black (чёрный)
R = Red (красный)
```

## Зачем нужно

### Проблема AVL-деревьев

AVL-деревья слишком строго поддерживают баланс, что требует много ротаций при частых вставках/удалениях.

### Преимущества Red-Black

- **Меньше ротаций** при модификации (максимум 2-3)
- **Лучшая производительность** для операций записи
- **Проще реализовать** некоторые операции (особенно удаление)
- **Широко используется** в стандартных библиотеках

### Применение

- **C++ STL**: `std::map`, `std::set`, `std::multimap`, `std::multiset`
- **Java**: `TreeMap`, `TreeSet`
- **Linux Kernel**: CFS (Completely Fair Scheduler), виртуальная память
- **Базы данных**: индексы в памяти
- **Ассоциативные контейнеры**: в большинстве языков программирования

## Как работает

### Пять свойств красно-чёрного дерева

```
1. Каждый узел либо красный, либо чёрный

2. Корень всегда чёрный

3. Все листья (NIL) чёрные

4. Если узел красный, то оба его потомка чёрные
   (нет двух красных узлов подряд)

5. Для каждого узла все простые пути от него до листьев-потомков
   содержат одинаковое число чёрных узлов (чёрная высота)
```

### Визуализация свойств

```
                 13(B)          Чёрная высота = 2
                /    \
             8(R)     17(R)     ← Красные узлы
            /   \    /    \
         1(B) 11(B) 15(B) 25(B) ← Чёрные узлы
         / \   / \   / \   / \
       NIL NIL ...         22(R)
       (все NIL чёрные)    / \
                         NIL NIL

Путь от 13 до любого NIL:
13(B) → 8(R) → 1(B) → NIL  : 2 чёрных узла (не считая NIL)
13(B) → 17(R) → 25(B) → 22(R) → NIL : 2 чёрных узла
```

### Чёрная высота (Black Height)

**Чёрная высота bh(x)** — число чёрных узлов на любом пути от x до листа (не включая x).

```
           13(B)  bh=2
          /    \
       8(R)     17(R)  bh=2
       /  \     /   \
    1(B) 11(B) 15(B) 25(B)  bh=1
      \               /
       6(R)         22(R)  bh=1
```

### Гарантия высоты

Красно-чёрное дерево с n внутренними узлами имеет высоту:
```
h ≤ 2 * log₂(n + 1)
```

**Доказательство идеи:**
- Минимум половина узлов на любом пути — чёрные (из-за свойства 4)
- Чёрная высота ≥ h/2
- Поддерево с корнем x содержит ≥ 2^bh(x) - 1 внутренних узлов
- Следовательно: n ≥ 2^(h/2) - 1, откуда h ≤ 2*log₂(n+1)

## Инварианты и свойства

### Ключевые инварианты

1. **BST-свойство**: левый потомок < родитель < правый потомок
2. **Цветовой инвариант**: нет двух красных узлов подряд
3. **Инвариант чёрной высоты**: одинаковое число чёрных узлов на всех путях

### Следствия из свойств

```
- Красный узел не может иметь красного родителя
- Красный узел не может иметь красного ребёнка
- Чёрный узел может иметь потомков любого цвета
- Самый длинный путь не более чем в 2 раза длиннее самого короткого
```

### Left-Leaning Red-Black Tree (LLRB)

Упрощённый вариант с дополнительным правилом:
```
- Красные связи направлены только влево

Обычное RB:           LLRB:
    B                   B
   / \                 /
  R   R               R
                     /
                    R
```

## Псевдокод основных операций

### Структура узла

```python
class Color:
    RED = True
    BLACK = False

class RBNode:
    def __init__(self, key):
        self.key = key
        self.left = None
        self.right = None
        self.parent = None
        self.color = Color.RED  # Новые узлы всегда красные
```

### Структура дерева

```python
class RBTree:
    def __init__(self):
        self.NIL = RBNode(None)
        self.NIL.color = Color.BLACK
        self.root = self.NIL
```

### Ротации

```python
def rotate_left(self, x):
    """
         x                 y
        / \               / \
       α   y      =>     x   γ
          / \           / \
         β   γ         α   β
    """
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

def rotate_right(self, x):
    """
         x               y
        / \             / \
       y   γ    =>     α   x
      / \                 / \
     α   β               β   γ
    """
    y = x.left
    x.left = y.right

    if y.right != self.NIL:
        y.right.parent = x

    y.parent = x.parent

    if x.parent is None:
        self.root = y
    elif x == x.parent.right:
        x.parent.right = y
    else:
        x.parent.left = y

    y.right = x
    x.parent = y
```

### Вставка

```python
def insert(self, key):
    # 1. Обычная вставка BST
    new_node = RBNode(key)
    new_node.left = self.NIL
    new_node.right = self.NIL

    parent = None
    current = self.root

    while current != self.NIL:
        parent = current
        if key < current.key:
            current = current.left
        else:
            current = current.right

    new_node.parent = parent

    if parent is None:
        self.root = new_node
    elif key < parent.key:
        parent.left = new_node
    else:
        parent.right = new_node

    # 2. Исправление нарушений
    self.insert_fixup(new_node)
```

### Исправление после вставки

```python
def insert_fixup(self, z):
    """
    Три случая (и их симметричные варианты):

    Случай 1: Дядя красный - перекрашивание
    Случай 2: Дядя чёрный, z - правый ребёнок - левая ротация
    Случай 3: Дядя чёрный, z - левый ребёнок - правая ротация
    """
    while z.parent and z.parent.color == Color.RED:
        if z.parent == z.parent.parent.left:
            uncle = z.parent.parent.right

            # Случай 1: Дядя красный
            if uncle.color == Color.RED:
                z.parent.color = Color.BLACK
                uncle.color = Color.BLACK
                z.parent.parent.color = Color.RED
                z = z.parent.parent

            else:
                # Случай 2: Дядя чёрный, z - правый ребёнок
                if z == z.parent.right:
                    z = z.parent
                    self.rotate_left(z)

                # Случай 3: Дядя чёрный, z - левый ребёнок
                z.parent.color = Color.BLACK
                z.parent.parent.color = Color.RED
                self.rotate_right(z.parent.parent)

        else:
            # Симметричные случаи (parent - правый ребёнок)
            uncle = z.parent.parent.left

            if uncle.color == Color.RED:
                z.parent.color = Color.BLACK
                uncle.color = Color.BLACK
                z.parent.parent.color = Color.RED
                z = z.parent.parent
            else:
                if z == z.parent.left:
                    z = z.parent
                    self.rotate_right(z)

                z.parent.color = Color.BLACK
                z.parent.parent.color = Color.RED
                self.rotate_left(z.parent.parent)

    self.root.color = Color.BLACK
```

### Визуализация случаев вставки

```
СЛУЧАЙ 1: Дядя красный — перекрашивание

        G(B)                    G(R)
       /    \                  /    \
     P(R)   U(R)      =>     P(B)   U(B)
     /                       /
   Z(R)                    Z(R)

   (проблема перемещается вверх к G)


СЛУЧАЙ 2: Дядя чёрный, Z — правый ребёнок

        G(B)                    G(B)
       /    \                  /    \
     P(R)   U(B)      =>     Z(R)   U(B)
       \                     /
       Z(R)                P(R)

   (переходит в случай 3)


СЛУЧАЙ 3: Дядя чёрный, Z — левый ребёнок

        G(B)                    P(B)
       /    \                  /    \
     P(R)   U(B)      =>     Z(R)   G(R)
     /                               \
   Z(R)                              U(B)

   (дерево сбалансировано)
```

### Поиск

```python
def search(self, key):
    """Стандартный поиск BST — O(log n)"""
    current = self.root
    while current != self.NIL:
        if key == current.key:
            return current
        elif key < current.key:
            current = current.left
        else:
            current = current.right
    return None
```

### Удаление (упрощённо)

```python
def delete(self, key):
    z = self.search(key)
    if z is None:
        return

    y = z
    y_original_color = y.color

    if z.left == self.NIL:
        x = z.right
        self.transplant(z, z.right)
    elif z.right == self.NIL:
        x = z.left
        self.transplant(z, z.left)
    else:
        y = self.minimum(z.right)
        y_original_color = y.color
        x = y.right

        if y.parent == z:
            x.parent = y
        else:
            self.transplant(y, y.right)
            y.right = z.right
            y.right.parent = y

        self.transplant(z, y)
        y.left = z.left
        y.left.parent = y
        y.color = z.color

    if y_original_color == Color.BLACK:
        self.delete_fixup(x)

def transplant(self, u, v):
    """Заменяет поддерево u поддеревом v"""
    if u.parent is None:
        self.root = v
    elif u == u.parent.left:
        u.parent.left = v
    else:
        u.parent.right = v
    v.parent = u.parent
```

### Исправление после удаления

```python
def delete_fixup(self, x):
    """
    Четыре случая для восстановления баланса после удаления чёрного узла
    """
    while x != self.root and x.color == Color.BLACK:
        if x == x.parent.left:
            sibling = x.parent.right

            # Случай 1: Брат красный
            if sibling.color == Color.RED:
                sibling.color = Color.BLACK
                x.parent.color = Color.RED
                self.rotate_left(x.parent)
                sibling = x.parent.right

            # Случай 2: Брат чёрный, оба его ребёнка чёрные
            if (sibling.left.color == Color.BLACK and
                sibling.right.color == Color.BLACK):
                sibling.color = Color.RED
                x = x.parent

            else:
                # Случай 3: Брат чёрный, правый ребёнок чёрный
                if sibling.right.color == Color.BLACK:
                    sibling.left.color = Color.BLACK
                    sibling.color = Color.RED
                    self.rotate_right(sibling)
                    sibling = x.parent.right

                # Случай 4: Брат чёрный, правый ребёнок красный
                sibling.color = x.parent.color
                x.parent.color = Color.BLACK
                sibling.right.color = Color.BLACK
                self.rotate_left(x.parent)
                x = self.root

        else:
            # Симметричные случаи
            sibling = x.parent.left
            # ... (аналогично, но зеркально)

    x.color = Color.BLACK
```

## Анализ сложности

| Операция | Время | Ротации |
|----------|-------|---------|
| Поиск    | O(log n) | 0 |
| Вставка  | O(log n) | ≤ 2 |
| Удаление | O(log n) | ≤ 3 |
| Память   | O(n) | — |

### Сравнение числа ротаций

```
Операция    AVL         Red-Black
─────────────────────────────────
Вставка     ≤ 2         ≤ 2
Удаление    O(log n)    ≤ 3
```

### Практическая производительность

```
               Много чтений    Много записей
─────────────────────────────────────────────
AVL            Лучше           Хуже
Red-Black      Немного хуже    Лучше
```

## Сравнение с альтернативами

| Свойство | Red-Black | AVL | B-Tree | Skip List |
|----------|-----------|-----|--------|-----------|
| Высота | ≤ 2 log n | ≤ 1.44 log n | log_m n | — |
| Вставка | O(log n), ≤2 rot | O(log n), ≤2 rot | O(log n) | O(log n) avg |
| Удаление | O(log n), ≤3 rot | O(log n), O(log n) rot | O(log n) | O(log n) avg |
| Реализация | Средняя | Средняя | Сложнее | Проще |
| Кэш-эффективность | Низкая | Низкая | Высокая | Низкая |

### Когда выбирать Red-Black

**Выбирайте Red-Black:**
- Много вставок и удалений
- Нужна стабильная производительность
- Реализуете ассоциативный контейнер общего назначения

**Выбирайте AVL:**
- Преобладают операции поиска
- Критична минимальная высота дерева

**Выбирайте B-Tree:**
- Данные на диске
- Важна кэш-эффективность

## Примеры использования в реальных системах

### 1. C++ STL (libstdc++, libc++)

```cpp
#include <map>
#include <set>

std::map<int, std::string> m;     // Red-Black Tree внутри
std::set<int> s;                   // Red-Black Tree внутри

m[1] = "one";
m[2] = "two";
s.insert(10);
```

### 2. Java Collections

```java
import java.util.TreeMap;
import java.util.TreeSet;

TreeMap<Integer, String> map = new TreeMap<>();  // Red-Black Tree
TreeSet<Integer> set = new TreeSet<>();          // Red-Black Tree
```

### 3. Linux Kernel — CFS Scheduler

```
Completely Fair Scheduler использует Red-Black Tree
для организации задач по virtual runtime:

          task_3 (vruntime=100)
              /     \
    task_1(50)       task_5(150)
       /   \            /   \
   task_0  task_2   task_4  task_6

Крайний левый узел — задача с минимальным vruntime
(следующая для выполнения)
```

### 4. Linux Kernel — Управление памятью

```c
// Структура vm_area_struct организована в Red-Black Tree
// для быстрого поиска при page fault

struct mm_struct {
    struct rb_root mm_rb;  // Корень Red-Black дерева
    // ...
};
```

### 5. Nginx — таймеры и очереди

```
Nginx использует Red-Black Tree для управления
таймерами соединений:

- Быстрая вставка/удаление таймеров: O(log n)
- Быстрый поиск истёкших таймеров: O(1) + удаление
```

### 6. Epoll в Linux

```
epoll использует Red-Black Tree для хранения
отслеживаемых файловых дескрипторов:

- Вставка fd: O(log n)
- Удаление fd: O(log n)
- Поиск fd: O(log n)
```

## Оптимизации и вариации

### 1. Left-Leaning Red-Black Tree (LLRB)

Упрощённая версия с меньшим числом случаев:

```
Дополнительное правило:
- Красные связи направлены только влево

Преимущества:
- Меньше кода
- Меньше случаев при вставке/удалении
- Легче для понимания

Недостатки:
- Чуть больше ротаций в среднем
```

### 2. AA-дерево (Arne Andersson)

Ещё более простая вариация:

```
- Красные связи только справа
- Левые связи всегда чёрные
- Только 2 операции: skew и split
```

### 3. 2-3-4 деревья

Red-Black дерево — это бинарное представление 2-3-4 дерева:

```
2-3-4 дерево:          Эквивалентное Red-Black:

   [10|20]                    20(B)
   /  |  \                   /    \
  5   15  30              10(R)   30(B)
                          /
                        5(B)
```

## Резюме

Красно-чёрное дерево — это самобалансирующееся BST, которое использует раскраску узлов для поддержания приблизительного баланса. Благодаря более слабым требованиям к балансу (по сравнению с AVL), оно требует меньше ротаций при модификациях, что делает его идеальным выбором для реализации стандартных ассоциативных контейнеров. Пять свойств (цвета корня, листьев, отсутствие красных пар, одинаковая чёрная высота) гарантируют высоту не более 2*log(n+1), обеспечивая O(log n) для всех операций.

---
[prev: 01-avl-trees](./01-avl-trees.md) | [next: 03-b-trees](./03-b-trees.md)