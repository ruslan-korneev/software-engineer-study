# B-деревья (B-Trees)

[prev: 02-red-black-trees](./02-red-black-trees.md) | [next: 04-skip-lists](./04-skip-lists.md)
---

## Определение

**B-дерево** — это самобалансирующееся дерево поиска, оптимизированное для систем, где операции чтения/записи работают с блоками данных (дисковая память, базы данных). В отличие от бинарных деревьев, узел B-дерева может содержать более двух ключей и потомков.

```
B-дерево порядка 3 (2-3 дерево):

                [17]
               /    \
         [8]          [25|35]
        /   \        /   |   \
    [3|5]  [10|12] [20] [27|30] [40|45|48]
```

B-дерево было изобретено Рудольфом Байером и Эдвардом Маккрейтом в 1970 году в Boeing Research Labs.

## Зачем нужно

### Проблема бинарных деревьев на диске

```
BST в памяти:          BST на диске:
     10                 Чтение узла 10: 1 seek
    /  \                Чтение узла 5:  1 seek
   5    15              Чтение узла 2:  1 seek
  / \    \              ────────────────────────
 2   7   20             Итого: 3 операции ввода-вывода
                        для поиска ключа 2

При глубине 20: 20 операций ввода-вывода!
(SSD: ~20мс, HDD: ~200мс)
```

### Решение B-дерева

```
B-дерево минимизирует обращения к диску:

Один узел = один блок диска (4KB - 64KB)
Узел содержит сотни ключей

           [10|20|30|40|50|60|70|80|90]
          /   |   |   |   |   |   |   \

1 узел = 1 чтение с диска
Содержит много ключей → меньше уровней → меньше операций I/O
```

### Применение

- **Базы данных**: MySQL (InnoDB), PostgreSQL, Oracle, SQL Server
- **Файловые системы**: NTFS, HFS+, ext4 (htree), Btrfs, ReiserFS
- **Поисковые системы**: индексы Lucene/Elasticsearch
- **Хранилища ключ-значение**: LevelDB, RocksDB

## Как работает

### Параметры B-дерева

B-дерево порядка **m** (минимальная степень **t**) имеет следующие свойства:

```
Параметр          Минимум              Максимум
───────────────────────────────────────────────────
Ключей в узле     t - 1                2t - 1
Потомков          t                    2t
(кроме корня)

Для корня:        1 ключ               2t - 1 ключей

Пример при t = 3:
- Узел содержит от 2 до 5 ключей
- Узел имеет от 3 до 6 потомков
```

### Структура узла

```
Узел B-дерева:

┌─────────────────────────────────────────────────┐
│ leaf │ n │ key[1] │ key[2] │ ... │ key[n]       │
├─────────────────────────────────────────────────┤
│ c[0] │ c[1] │ c[2] │ ... │ c[n]                 │
└─────────────────────────────────────────────────┘

leaf - является ли узел листом
n    - количество ключей
key  - массив ключей (отсортирован)
c    - массив указателей на потомков
```

### Свойство упорядоченности

```
Для узла с ключами [k₁, k₂, ..., kₙ]:

     ┌────────────────────────────────┐
     │   k₁   │   k₂   │   k₃   │   k₄   │
     └────────────────────────────────┘
    /      \      \      \      \
   c₀      c₁     c₂     c₃     c₄

Все ключи в c₀  <  k₁
k₁  ≤  все ключи в c₁  <  k₂
k₂  ≤  все ключи в c₂  <  k₃
...и так далее
```

### Пример B-дерева (t = 2)

```
Порядок t = 2:
- Минимум 1 ключ, максимум 3 ключа в узле
- Минимум 2, максимум 4 потомка

              [16]
             /    \
       [3|7]        [20|26]
      /  |  \       /   |   \
   [1|2][4|5][8|14][18|19][21|25][27|30]

Высота = 2
Ёмкость = до 4³ = 64 ключа
```

## Инварианты и свойства

### Инварианты B-дерева

1. **Все листья на одном уровне** (идеально сбалансировано)
2. **Упорядоченность ключей** в каждом узле
3. **Минимальное заполнение** (кроме корня): ⌈m/2⌉ - 1 ключей
4. **Максимальное заполнение**: m - 1 ключей

### Высота B-дерева

```
Для n ключей и минимальной степени t:

h ≤ log_t((n + 1) / 2)

Пример:
n = 1,000,000 ключей
t = 100 (типично для дисковых систем)

h ≤ log₁₀₀(500,000) ≈ 2.85

Максимум 3 уровня для миллиона ключей!
(vs ~20 уровней для бинарного дерева)
```

### Сравнение высот

```
n = 1,000,000 ключей

Структура          Максимальная высота
────────────────────────────────────────
BST (худший)       1,000,000
AVL                ~29
Red-Black          ~40
B-дерево (t=100)   ~3
B-дерево (t=500)   ~2
```

## Псевдокод основных операций

### Структура узла

```python
class BTreeNode:
    def __init__(self, t, leaf=True):
        self.t = t           # Минимальная степень
        self.keys = []       # Ключи
        self.children = []   # Потомки
        self.leaf = leaf     # Является ли листом

    def is_full(self):
        return len(self.keys) == 2 * self.t - 1
```

### Структура дерева

```python
class BTree:
    def __init__(self, t):
        self.t = t
        self.root = BTreeNode(t, leaf=True)
```

### Поиск

```python
def search(self, node, key):
    """
    Поиск ключа в B-дереве

    Время: O(t * log_t(n)) = O(log n)
    Обращений к диску: O(log_t(n)) = O(h)
    """
    i = 0
    # Находим первый ключ >= key
    while i < len(node.keys) and key > node.keys[i]:
        i += 1

    # Ключ найден
    if i < len(node.keys) and key == node.keys[i]:
        return (node, i)

    # Если лист — ключа нет
    if node.leaf:
        return None

    # Рекурсивно ищем в соответствующем потомке
    return self.search(node.children[i], key)
```

### Визуализация поиска

```
Поиск ключа 14 в дереве:

              [16]                   ← 14 < 16, идём влево
             /    \
       [3|7]        [20|26]
      /  |  \       /   |   \
   [1|2][4|5][8|14][18|19][21|25][27|30]
                ↑
           14 > 7, идём в c[2]
           Найден!

Операций I/O: 3 (одна на уровень)
```

### Разделение узла (Split)

```python
def split_child(self, parent, i):
    """
    Разделяет полного потомка parent.children[i]

         [...]                    [...|M|...]
           |                       /     \
    [A|B|M|C|D]          =>    [A|B]     [C|D]

    M — медианный ключ поднимается в родителя
    """
    t = self.t
    full_child = parent.children[i]
    new_child = BTreeNode(t, leaf=full_child.leaf)

    # Медианный ключ
    median_key = full_child.keys[t - 1]

    # Правая половина ключей → новый узел
    new_child.keys = full_child.keys[t:]
    full_child.keys = full_child.keys[:t - 1]

    # Если не лист — разделяем и потомков
    if not full_child.leaf:
        new_child.children = full_child.children[t:]
        full_child.children = full_child.children[:t]

    # Вставляем новый узел и медианный ключ в родителя
    parent.children.insert(i + 1, new_child)
    parent.keys.insert(i, median_key)
```

### Визуализация разделения

```
До разделения (t = 3, узел полон):

Parent: [10|30]
         / | \
        /  |  \
       A [15|17|20|23|25] C
              ↑
           полный узел (5 ключей)

После разделения:

Parent: [10|20|30]        ← 20 поднялся вверх
        /  |  \  \
       A [15|17] [23|25] C
              ↑      ↑
           левая  правая
           часть  часть
```

### Вставка

```python
def insert(self, key):
    """
    Вставка ключа в B-дерево

    Стратегия: превентивное разделение
    (разделяем полные узлы по пути вниз)
    """
    root = self.root

    # Если корень полон — увеличиваем высоту
    if root.is_full():
        new_root = BTreeNode(self.t, leaf=False)
        new_root.children.append(self.root)
        self.split_child(new_root, 0)
        self.root = new_root

    self._insert_non_full(self.root, key)

def _insert_non_full(self, node, key):
    """Вставка в неполный узел"""
    i = len(node.keys) - 1

    if node.leaf:
        # Вставляем ключ на нужную позицию
        node.keys.append(None)  # Расширяем
        while i >= 0 and key < node.keys[i]:
            node.keys[i + 1] = node.keys[i]
            i -= 1
        node.keys[i + 1] = key

    else:
        # Находим потомка для рекурсии
        while i >= 0 and key < node.keys[i]:
            i -= 1
        i += 1

        # Если потомок полон — разделяем его
        if node.children[i].is_full():
            self.split_child(node, i)
            # После разделения определяем, в какую половину идти
            if key > node.keys[i]:
                i += 1

        self._insert_non_full(node.children[i], key)
```

### Пример вставки

```
Вставляем 22 в дерево (t = 2):

Начальное состояние:
           [10]
          /    \
      [5|7]    [15|20|25]  ← полный узел!

Шаг 1: Идём в правого потомка, но он полон.
       Разделяем его ПЕРЕД спуском:

           [10|20]
          /   |   \
      [5|7] [15] [25]

Шаг 2: Теперь 22 > 20, идём в [25]
       Вставляем 22:

           [10|20]
          /   |   \
      [5|7] [15] [22|25]
```

### Удаление

Удаление в B-дереве сложнее и включает несколько случаев:

```python
def delete(self, key):
    """Удаление ключа из B-дерева"""
    self._delete(self.root, key)

    # Если корень пуст и не лист — сжимаем дерево
    if len(self.root.keys) == 0 and not self.root.leaf:
        self.root = self.root.children[0]

def _delete(self, node, key):
    t = self.t
    i = 0
    while i < len(node.keys) and key > node.keys[i]:
        i += 1

    # Случай 1: Ключ в текущем узле
    if i < len(node.keys) and key == node.keys[i]:
        if node.leaf:
            # 1a: Просто удаляем из листа
            node.keys.pop(i)
        else:
            # 1b: Ключ во внутреннем узле
            self._delete_internal(node, i)

    # Случай 2: Ключ в потомке
    else:
        if node.leaf:
            return  # Ключа нет в дереве

        # Гарантируем, что потомок имеет >= t ключей
        if len(node.children[i].keys) < t:
            self._fill(node, i)

        # После fill индекс может измениться
        if i > len(node.keys):
            i -= 1

        self._delete(node.children[i], key)
```

### Вспомогательные операции удаления

```python
def _delete_internal(self, node, i):
    """Удаление ключа из внутреннего узла"""
    t = self.t
    key = node.keys[i]

    # Если левый потомок имеет >= t ключей
    if len(node.children[i].keys) >= t:
        # Заменяем на предшественника
        pred = self._get_predecessor(node, i)
        node.keys[i] = pred
        self._delete(node.children[i], pred)

    # Если правый потомок имеет >= t ключей
    elif len(node.children[i + 1].keys) >= t:
        # Заменяем на преемника
        succ = self._get_successor(node, i)
        node.keys[i] = succ
        self._delete(node.children[i + 1], succ)

    # Оба потомка имеют t-1 ключей — сливаем их
    else:
        self._merge(node, i)
        self._delete(node.children[i], key)

def _get_predecessor(self, node, i):
    """Находит предшественника (максимум в левом поддереве)"""
    current = node.children[i]
    while not current.leaf:
        current = current.children[-1]
    return current.keys[-1]

def _get_successor(self, node, i):
    """Находит преемника (минимум в правом поддереве)"""
    current = node.children[i + 1]
    while not current.leaf:
        current = current.children[0]
    return current.keys[0]
```

### Слияние и заимствование

```python
def _fill(self, node, i):
    """Гарантирует, что children[i] имеет >= t ключей"""
    t = self.t

    # Заимствуем у левого соседа
    if i > 0 and len(node.children[i - 1].keys) >= t:
        self._borrow_from_left(node, i)

    # Заимствуем у правого соседа
    elif i < len(node.keys) and len(node.children[i + 1].keys) >= t:
        self._borrow_from_right(node, i)

    # Сливаем с соседом
    else:
        if i < len(node.keys):
            self._merge(node, i)
        else:
            self._merge(node, i - 1)

def _merge(self, node, i):
    """Сливает children[i] с children[i+1]"""
    left = node.children[i]
    right = node.children[i + 1]

    # Опускаем ключ из родителя
    left.keys.append(node.keys[i])

    # Добавляем ключи и потомков из правого узла
    left.keys.extend(right.keys)
    left.children.extend(right.children)

    # Удаляем ключ из родителя и правый узел
    node.keys.pop(i)
    node.children.pop(i + 1)
```

### Визуализация слияния

```
До слияния (t = 2):

     [10|20|30]
    /   |    \   \
 [5]  [15]  [25] [35]
   ↑     ↑
   минимальные (1 ключ каждый)

Удаляем 15:
Сначала сливаем [5] и бывший [15] с ключом 10:

     [20|30]
    /    \   \
 [5|10] [25] [35]

Затем удаляем 15 (его уже нет).
```

## Анализ сложности

| Операция | Время | Обращений к диску |
|----------|-------|-------------------|
| Поиск    | O(t * log_t n) | O(log_t n) |
| Вставка  | O(t * log_t n) | O(log_t n) |
| Удаление | O(t * log_t n) | O(log_t n) |
| Память   | O(n) | — |

### Практические числа

```
Типичные параметры для дисковых систем:

Размер блока = 4KB
Размер ключа = 8 байт
Размер указателя = 8 байт

Ключей в узле ≈ 4096 / (8 + 8) ≈ 250

Для 1 миллиарда записей:
h = log_250(1,000,000,000) ≈ 4 уровня

4 чтения диска для поиска среди миллиарда записей!
```

## Сравнение с альтернативами

| Свойство | B-Tree | B+ Tree | LSM-Tree | Red-Black |
|----------|--------|---------|----------|-----------|
| Поиск | O(log n) | O(log n) | O(log n) | O(log n) |
| Range query | Медленнее | Быстро | Средне | Очень медленно |
| Вставка | O(log n) | O(log n) | O(1) amort | O(log n) |
| Чтение с диска | Отлично | Отлично | Хуже | Плохо |
| Запись на диск | Хорошо | Хорошо | Отлично | Плохо |

### B-Tree vs B+ Tree

```
B-Tree:
- Данные в ВСЕХ узлах (и внутренних, и листьях)
- Меньше дублирования ключей
- Лучше для точечных запросов

          [17|35]
         /   |   \
      [8]  [23] [48]
       ↑    ↑    ↑
      данные везде


B+ Tree:
- Данные ТОЛЬКО в листьях
- Внутренние узлы — только индекс
- Листья связаны в список → быстрые range-запросы
- Используется в большинстве СУБД

          [17|35]           ← только ключи
         /   |   \
      [8]  [23] [48]        ← только ключи
       |    |    |
      [8]→[17]→[23]→[35]→[48]  ← данные + связный список
```

## Примеры использования в реальных системах

### 1. MySQL InnoDB

```sql
-- Первичный ключ хранится в кластеризованном B+ дереве
CREATE TABLE users (
    id INT PRIMARY KEY,      -- Ключ B+ дерева
    name VARCHAR(100),
    email VARCHAR(100)
);

-- Вторичные индексы — отдельные B+ деревья
CREATE INDEX idx_email ON users(email);
```

```
Структура InnoDB:

Кластеризованный индекс (первичный ключ):
[id=1|данные] → [id=2|данные] → [id=3|данные]

Вторичный индекс (email):
[email → id] → позволяет найти id, затем данные
```

### 2. PostgreSQL

```
PostgreSQL использует B+ деревья для:
- Обычных индексов (CREATE INDEX)
- Первичных ключей
- Unique constraints

SELECT * FROM pg_indexes WHERE tablename = 'users';
```

### 3. Файловая система NTFS

```
NTFS Master File Table (MFT):
- B+ дерево для организации записей файлов
- Индекс директорий

Поиск файла в директории с 10,000 файлами:
B-Tree: ~3 обращения к диску
Линейный список: ~1000+ обращений
```

### 4. SQLite

```python
import sqlite3

# SQLite использует B-Tree для таблиц и индексов
conn = sqlite3.connect('example.db')
cursor = conn.cursor()

# Таблица = B-Tree с rowid как ключ
cursor.execute('''
    CREATE TABLE items (
        id INTEGER PRIMARY KEY,
        name TEXT
    )
''')

# Индекс = отдельное B-Tree
cursor.execute('CREATE INDEX idx_name ON items(name)')
```

### 5. LevelDB / RocksDB

```
Хотя LevelDB использует LSM-Tree как основную структуру,
внутренние sstable файлы организованы как отсортированные
блоки с индексом (похоже на B-Tree концептуально).

                Write Path (LSM):
MemTable → L0 → L1 → L2 → ... → Ln

Каждый уровень содержит отсортированные файлы
с блочными индексами для поиска.
```

## B+ Tree — расширение B-Tree

Большинство СУБД используют B+ Tree вместо B-Tree:

```python
class BPlusTreeNode:
    def __init__(self, t, leaf=True):
        self.t = t
        self.keys = []
        self.children = []  # Указатели на потомков или данные
        self.leaf = leaf
        self.next = None    # Для листьев — указатель на следующий лист

class BPlusTree:
    def __init__(self, t):
        self.t = t
        self.root = BPlusTreeNode(t, leaf=True)

    def range_query(self, start, end):
        """
        Эффективный range query благодаря связанным листьям
        """
        # 1. Находим стартовый лист
        leaf = self._find_leaf(start)

        # 2. Сканируем листья последовательно
        results = []
        while leaf:
            for key, value in zip(leaf.keys, leaf.values):
                if key > end:
                    return results
                if key >= start:
                    results.append((key, value))
            leaf = leaf.next  # Переходим к следующему листу

        return results
```

## Резюме

B-дерево — это фундаментальная структура данных для систем хранения, оптимизированная для минимизации обращений к диску. Ключевые особенности:

1. **Высокое ветвление** — сотни ключей в узле, минимальная высота
2. **Идеальная балансировка** — все листья на одном уровне
3. **Эффективность для диска** — один узел = один блок
4. **Гарантированная производительность** — O(log_t n) для всех операций

B+ Tree — расширение со связанными листьями, стандарт для СУБД благодаря эффективным range-запросам.

---
[prev: 02-red-black-trees](./02-red-black-trees.md) | [next: 04-skip-lists](./04-skip-lists.md)