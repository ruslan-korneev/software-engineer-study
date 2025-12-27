# Основы деревьев (Tree Basics)

## Определение

**Дерево** — это иерархическая структура данных, состоящая из узлов (nodes), соединённых рёбрами (edges). Дерево имеет один корневой узел (root), от которого все остальные узлы достижимы по единственному пути.

```
          ┌───┐
          │ A │        ← Корень (root)
          └─┬─┘
     ┌──────┼──────┐
    ┌▼─┐   ┌▼─┐   ┌▼─┐
    │ B│   │ C│   │ D│   ← Внутренние узлы
    └┬─┘   └──┘   └┬─┘
   ┌─┴─┐         ┌─┴─┐
  ┌▼─┐┌▼─┐      ┌▼─┐┌▼─┐
  │ E││ F│      │ G││ H│  ← Листья (leaves)
  └──┘└──┘      └──┘└──┘
```

## Зачем нужно

### Практическое применение:
- **Файловые системы** — директории и файлы
- **DOM** — структура HTML-документа
- **Базы данных** — B-деревья для индексов
- **Компиляторы** — AST (абстрактное синтаксическое дерево)
- **Иерархические данные** — организационные структуры
- **Алгоритмы** — деревья решений, игровые деревья
- **Сжатие** — деревья Хаффмана

## Терминология

```
               ┌───┐
    Level 0    │ A │  ← Root (корень)
               └─┬─┘
          ┌─────┴─────┐
         ┌▼─┐       ┌─▼┐
    Level 1│ B│  Parent│ C│  ← Internal nodes (внутренние узлы)
         └┬─┘    of D,E└─┬┘
        ┌─┴─┐          ┌─┴─┐
       ┌▼─┐┌▼─┐       ┌▼─┐┌▼─┐
    Level 2│ D││ E│       │ F││ G│  ← Leaves (листья)
       └──┘└──┘       └──┘└──┘
           ↑               ↑
       Siblings        Siblings
      (братья)         (братья)

Height (высота дерева) = 2 (макс. путь от корня до листа)
Depth of E = 2 (путь от корня до E)
```

### Основные термины

| Термин | Описание |
|--------|----------|
| **Root (корень)** | Верхний узел дерева, не имеет родителя |
| **Node (узел)** | Элемент дерева, содержит данные |
| **Edge (ребро)** | Связь между родителем и ребёнком |
| **Parent (родитель)** | Узел, имеющий дочерние узлы |
| **Child (ребёнок)** | Узел, имеющий родителя |
| **Siblings (братья)** | Узлы с общим родителем |
| **Leaf (лист)** | Узел без детей |
| **Internal node** | Узел, имеющий хотя бы одного ребёнка |
| **Depth (глубина)** | Расстояние от корня до узла |
| **Height (высота)** | Максимальная глубина в дереве |
| **Level (уровень)** | Набор узлов одинаковой глубины |
| **Subtree (поддерево)** | Узел со всеми его потомками |
| **Degree (степень)** | Количество детей узла |

## Свойства деревьев

```
Для дерева с n узлами:

1. Количество рёбер = n - 1
2. Путь между любыми двумя узлами единственный
3. Дерево — связный граф без циклов
4. Добавление одного ребра создаёт цикл
5. Удаление одного ребра разрывает связность
```

### Типы деревьев по степени узлов

```
Бинарное дерево (степень ≤ 2):
        A
       / \
      B   C
     / \
    D   E

N-арное дерево (степень ≤ n):
        A
      / | \
     B  C  D
    /|\   / \
   E F G H   I
```

## Представление дерева

### Класс узла

```python
class TreeNode:
    def __init__(self, value):
        self.value = value
        self.children = []  # список дочерних узлов

    def add_child(self, child):
        self.children.append(child)

    def is_leaf(self):
        return len(self.children) == 0

class BinaryTreeNode:
    def __init__(self, value):
        self.value = value
        self.left = None   # левый ребёнок
        self.right = None  # правый ребёнок
```

### Представление через массив (для полных бинарных деревьев)

```
Дерево:
       1
      / \
     2   3
    / \ / \
   4  5 6  7

Массив: [1, 2, 3, 4, 5, 6, 7]
Индекс:  0  1  2  3  4  5  6

Формулы для узла с индексом i:
- Родитель: (i - 1) // 2
- Левый ребёнок: 2 * i + 1
- Правый ребёнок: 2 * i + 2
```

### Представление "левый ребёнок — правый брат"

```
Обычное дерево:          Left-Child Right-Sibling:
       A                        A
     / | \                     /
    B  C  D                   B → C → D
   / \                       /
  E   F                     E → F
```

```python
class LCRSNode:
    def __init__(self, value):
        self.value = value
        self.left_child = None    # первый ребёнок
        self.right_sibling = None  # следующий брат
```

## Основные операции

### Обходы дерева (Traversals)

```python
class TreeNode:
    def __init__(self, value):
        self.value = value
        self.children = []

def dfs_preorder(node):
    """
    Preorder (прямой обход): корень → дети
    Используется для: копирование, сериализация
    """
    if node is None:
        return []

    result = [node.value]
    for child in node.children:
        result.extend(dfs_preorder(child))
    return result

def dfs_postorder(node):
    """
    Postorder (обратный обход): дети → корень
    Используется для: удаление, подсчёт размера
    """
    if node is None:
        return []

    result = []
    for child in node.children:
        result.extend(dfs_postorder(child))
    result.append(node.value)
    return result

def bfs(root):
    """
    BFS (обход в ширину): по уровням
    Используется для: поиск кратчайшего пути, уровневые операции
    """
    if root is None:
        return []

    result = []
    queue = [root]

    while queue:
        node = queue.pop(0)
        result.append(node.value)
        queue.extend(node.children)

    return result
```

```
Пример обходов:

        A
       /|\
      B C D
     /|   |
    E F   G

Preorder:  A, B, E, F, C, D, G
Postorder: E, F, B, C, G, D, A
BFS:       A, B, C, D, E, F, G
```

### Вычисление свойств

```python
def height(node):
    """Высота дерева"""
    if node is None or not node.children:
        return 0

    return 1 + max(height(child) for child in node.children)

def size(node):
    """Количество узлов"""
    if node is None:
        return 0

    return 1 + sum(size(child) for child in node.children)

def depth(node, target, current_depth=0):
    """Глубина узла с заданным значением"""
    if node is None:
        return -1

    if node.value == target:
        return current_depth

    for child in node.children:
        d = depth(child, target, current_depth + 1)
        if d != -1:
            return d

    return -1

def count_leaves(node):
    """Количество листьев"""
    if node is None:
        return 0

    if not node.children:
        return 1

    return sum(count_leaves(child) for child in node.children)
```

### Поиск

```python
def find(node, value):
    """Поиск узла по значению (DFS)"""
    if node is None:
        return None

    if node.value == value:
        return node

    for child in node.children:
        result = find(child, value)
        if result:
            return result

    return None

def find_path(node, value, path=None):
    """Поиск пути от корня до узла"""
    if path is None:
        path = []

    if node is None:
        return None

    path.append(node.value)

    if node.value == value:
        return path

    for child in node.children:
        result = find_path(child, value, path.copy())
        if result:
            return result

    return None
```

## Примеры с разбором

### Пример 1: Файловая система

```python
class FileSystemNode:
    def __init__(self, name, is_dir=False):
        self.name = name
        self.is_dir = is_dir
        self.children = []
        self.size = 0  # размер файла в байтах

    def add(self, node):
        if not self.is_dir:
            raise ValueError("Cannot add to file")
        self.children.append(node)

    def list_contents(self, indent=0):
        """Рекурсивный вывод содержимого"""
        prefix = "  " * indent
        print(f"{prefix}{self.name}{'/' if self.is_dir else ''}")
        for child in self.children:
            child.list_contents(indent + 1)

    def total_size(self):
        """Общий размер директории"""
        if not self.is_dir:
            return self.size

        return sum(child.total_size() for child in self.children)

# Использование:
root = FileSystemNode("/", True)
home = FileSystemNode("home", True)
user = FileSystemNode("user", True)
file1 = FileSystemNode("document.txt")
file1.size = 1024

root.add(home)
home.add(user)
user.add(file1)

root.list_contents()
# /
#   home/
#     user/
#       document.txt
```

### Пример 2: Организационная структура

```python
class Employee:
    def __init__(self, name, title):
        self.name = name
        self.title = title
        self.subordinates = []

    def add_subordinate(self, employee):
        self.subordinates.append(employee)

    def count_subordinates(self):
        """Считает всех подчинённых (прямых и косвенных)"""
        count = len(self.subordinates)
        for sub in self.subordinates:
            count += sub.count_subordinates()
        return count

    def find_employee(self, name):
        """Ищет сотрудника по имени"""
        if self.name == name:
            return self

        for sub in self.subordinates:
            result = sub.find_employee(name)
            if result:
                return result

        return None

    def print_org_chart(self, level=0):
        print("  " * level + f"{self.name} ({self.title})")
        for sub in self.subordinates:
            sub.print_org_chart(level + 1)

# Использование:
ceo = Employee("Alice", "CEO")
cto = Employee("Bob", "CTO")
cfo = Employee("Carol", "CFO")
dev1 = Employee("Dave", "Developer")
dev2 = Employee("Eve", "Developer")

ceo.add_subordinate(cto)
ceo.add_subordinate(cfo)
cto.add_subordinate(dev1)
cto.add_subordinate(dev2)

ceo.print_org_chart()
# Alice (CEO)
#   Bob (CTO)
#     Dave (Developer)
#     Eve (Developer)
#   Carol (CFO)
```

### Пример 3: Парсинг HTML (упрощённо)

```python
class HTMLNode:
    def __init__(self, tag, content=""):
        self.tag = tag
        self.content = content
        self.children = []
        self.attributes = {}

    def add_child(self, node):
        self.children.append(node)

    def to_html(self, indent=0):
        """Генерирует HTML-строку"""
        spaces = "  " * indent
        attrs = " ".join(f'{k}="{v}"' for k, v in self.attributes.items())
        attrs_str = f" {attrs}" if attrs else ""

        if not self.children and not self.content:
            return f"{spaces}<{self.tag}{attrs_str}/>"

        if not self.children:
            return f"{spaces}<{self.tag}{attrs_str}>{self.content}</{self.tag}>"

        lines = [f"{spaces}<{self.tag}{attrs_str}>"]
        if self.content:
            lines.append(f"{spaces}  {self.content}")
        for child in self.children:
            lines.append(child.to_html(indent + 1))
        lines.append(f"{spaces}</{self.tag}>")

        return "\n".join(lines)

# Использование:
html = HTMLNode("html")
body = HTMLNode("body")
div = HTMLNode("div")
div.attributes["class"] = "container"
p = HTMLNode("p", "Hello, World!")

html.add_child(body)
body.add_child(div)
div.add_child(p)

print(html.to_html())
```

### Пример 4: Дерево решений

```python
class DecisionNode:
    def __init__(self, question, yes=None, no=None, result=None):
        self.question = question
        self.yes = yes  # ветка "да"
        self.no = no    # ветка "нет"
        self.result = result  # результат (для листьев)

    def is_leaf(self):
        return self.result is not None

    def decide(self):
        """Интерактивный обход дерева решений"""
        node = self

        while not node.is_leaf():
            answer = input(f"{node.question} (y/n): ").lower()
            node = node.yes if answer == 'y' else node.no

        return node.result

# Дерево для выбора фрукта:
tree = DecisionNode(
    "Хотите кислое?",
    yes=DecisionNode(
        "Хотите цитрус?",
        yes=DecisionNode(None, result="Лимон"),
        no=DecisionNode(None, result="Кислое яблоко")
    ),
    no=DecisionNode(
        "Хотите сладкое?",
        yes=DecisionNode(None, result="Банан"),
        no=DecisionNode(None, result="Авокадо")
    )
)

# result = tree.decide()
```

## Анализ сложности

| Операция | Время | Примечание |
|----------|-------|------------|
| Поиск | O(n) | В худшем случае обход всего дерева |
| Вставка | O(1)* | *Если известен родитель |
| Удаление | O(n) | Поиск + удаление поддерева |
| Обход | O(n) | Посещаем каждый узел |
| Высота | O(n) | Обход всего дерева |
| Размер | O(n) | Подсчёт всех узлов |

### Память
- **На узел**: O(k), где k — количество детей
- **Всего**: O(n)

## Типичные ошибки

### 1. Забытая проверка на None

```python
# НЕПРАВИЛЬНО
def height(node):
    return 1 + max(height(c) for c in node.children)
# Упадёт, если node = None

# ПРАВИЛЬНО
def height(node):
    if node is None:
        return -1
    if not node.children:
        return 0
    return 1 + max(height(c) for c in node.children)
```

### 2. Путаница с высотой и глубиной

```python
# Высота — от узла ВНИЗ до самого далёкого листа
# Глубина — от корня ВНИЗ до узла

# Высота листа = 0
# Глубина корня = 0
```

### 3. Модификация при обходе

```python
# НЕПРАВИЛЬНО
for child in node.children:
    if should_remove(child):
        node.children.remove(child)  # изменяем во время итерации!

# ПРАВИЛЬНО
to_remove = [c for c in node.children if should_remove(c)]
for child in to_remove:
    node.children.remove(child)
```

### 4. Циклические ссылки

```python
# НЕПРАВИЛЬНО — создаёт цикл
a = TreeNode("A")
b = TreeNode("B")
a.children.append(b)
b.children.append(a)  # цикл! Это уже не дерево

# Дерево НЕ должно содержать циклов
```

## Виды деревьев

```
1. Общее дерево (General Tree)
   - Любое количество детей

2. Бинарное дерево (Binary Tree)
   - Максимум 2 ребёнка

3. Двоичное дерево поиска (BST)
   - Левые < корень < правые

4. Сбалансированные деревья
   - AVL, Red-Black, B-деревья

5. Куча (Heap)
   - Свойство порядка (мин или макс)

6. Trie (префиксное дерево)
   - Для строк и автодополнения
```
