# Массивы и связные списки

Две фундаментальные структуры данных для хранения последовательностей.

## Массив (Array)

Элементы хранятся **последовательно в памяти**.

```
Индекс:   0     1     2     3     4
        ┌─────┬─────┬─────┬─────┬─────┐
Память: │  10 │  20 │  30 │  40 │  50 │
        └─────┴─────┴─────┴─────┴─────┘
        0x100 0x104 0x108 0x112 0x116
```

### Плюсы
- Доступ по индексу — **O(1)** (адрес = начало + индекс × размер элемента)
- Компактное хранение в памяти
- Эффективный кэш CPU (данные рядом)

### Минусы
- Вставка/удаление в начале/середине — **O(n)** (нужно сдвинуть элементы)
- Фиксированный размер в низкоуровневых языках

## Связный список (Linked List)

Элементы хранятся **где угодно в памяти**, связаны указателями (ссылками).

```
┌───────────┐    ┌───────────┐    ┌───────────┐
│ 10 | next─┼───►│ 20 | next─┼───►│ 30 | None │
└───────────┘    └───────────┘    └───────────┘
    Head
```

### Плюсы
- Вставка/удаление в начале — **O(1)**
- Динамический размер (растёт по необходимости)
- Эффективная вставка/удаление если есть указатель на место

### Минусы
- Доступ по индексу — **O(n)** (нужно пройти от head)
- Больше памяти (каждый узел хранит указатель)
- Плохо для кэша CPU (данные разбросаны)

## Сравнение сложности

| Операция | Array (list) | Linked List |
|----------|--------------|-------------|
| Доступ по индексу | **O(1)** | O(n) |
| Поиск элемента | O(n) | O(n) |
| Вставка в начало | O(n) | **O(1)** |
| Вставка в конец | O(1)* | O(n) / O(1)** |
| Вставка в середину | O(n) | O(1)*** |
| Удаление из начала | O(n) | **O(1)** |
| Удаление с конца | O(1) | O(n) / O(1)** |

\* амортизированная сложность
\** если храним указатель на tail
\*** если есть указатель на предыдущий узел

## Python list — динамический массив

Python `list` реализован как динамический массив:

```python
lst = [10, 20, 30, 40, 50]

# O(1) — доступ по индексу
lst[2]              # 30

# O(1) амортизировано — добавление в конец
lst.append(60)

# O(n) — вставка в начало (все элементы сдвигаются)
lst.insert(0, 5)

# O(1) — удаление с конца
lst.pop()

# O(n) — удаление из начала
lst.pop(0)

# O(n) — удаление по значению
lst.remove(30)
```

### Динамическое расширение
Когда массив заполняется, Python выделяет больше памяти (обычно в 1.5-2 раза) и копирует элементы. Поэтому `append()` — O(1) амортизировано.

## Реализация односвязного списка

```python
class Node:
    """Узел списка"""
    def __init__(self, data):
        self.data = data
        self.next = None

    def __repr__(self):
        return f"Node({self.data})"


class LinkedList:
    """Односвязный список"""
    def __init__(self):
        self.head = None
        self._size = 0

    def __len__(self):
        return self._size

    def is_empty(self):
        return self.head is None

    def prepend(self, data):
        """Добавить в начало — O(1)"""
        new_node = Node(data)
        new_node.next = self.head
        self.head = new_node
        self._size += 1

    def append(self, data):
        """Добавить в конец — O(n)"""
        new_node = Node(data)
        if not self.head:
            self.head = new_node
        else:
            current = self.head
            while current.next:
                current = current.next
            current.next = new_node
        self._size += 1

    def delete(self, data):
        """Удалить первое вхождение — O(n)"""
        if not self.head:
            return False

        if self.head.data == data:
            self.head = self.head.next
            self._size -= 1
            return True

        current = self.head
        while current.next:
            if current.next.data == data:
                current.next = current.next.next
                self._size -= 1
                return True
            current = current.next
        return False

    def find(self, data):
        """Найти узел — O(n)"""
        current = self.head
        while current:
            if current.data == data:
                return current
            current = current.next
        return None

    def get(self, index):
        """Получить по индексу — O(n)"""
        if index < 0 or index >= self._size:
            raise IndexError("Index out of range")
        current = self.head
        for _ in range(index):
            current = current.next
        return current.data

    def to_list(self):
        """Преобразовать в список Python"""
        result = []
        current = self.head
        while current:
            result.append(current.data)
            current = current.next
        return result

    def __repr__(self):
        return f"LinkedList({self.to_list()})"
```

### Использование

```python
ll = LinkedList()
ll.append(10)
ll.append(20)
ll.append(30)
ll.prepend(5)

print(ll)           # LinkedList([5, 10, 20, 30])
print(len(ll))      # 4
print(ll.get(2))    # 20
print(ll.find(20))  # Node(20)

ll.delete(10)
print(ll)           # LinkedList([5, 20, 30])
```

## Двусвязный список (Doubly Linked List)

Каждый узел имеет указатели на предыдущий и следующий элементы:

```
       ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
None ◄─┤ prev│10│next ├───►│ prev│20│next ├───►│ prev│30│next ├─► None
       └──────────────┘◄───┴──────────────┘◄───┴──────────────┘
           Head                                      Tail
```

### Преимущества
- Можно идти в обе стороны
- Удаление с конца — O(1) если храним tail
- Удаление узла по указателю — O(1)

### Реализация узла

```python
class DoublyNode:
    def __init__(self, data):
        self.data = data
        self.prev = None
        self.next = None
```

## collections.deque — двусторонняя очередь

Python `deque` реализован как двусвязный список блоков:

```python
from collections import deque

d = deque([1, 2, 3])

# O(1) — добавление с обоих концов
d.appendleft(0)    # deque([0, 1, 2, 3])
d.append(4)        # deque([0, 1, 2, 3, 4])

# O(1) — удаление с обоих концов
d.popleft()        # 0, deque([1, 2, 3, 4])
d.pop()            # 4, deque([1, 2, 3])

# O(1) — просмотр концов
d[0]               # 1 (первый)
d[-1]              # 3 (последний)

# O(n) — доступ к середине
d[1]               # 2

# Ротация
d.rotate(1)        # deque([3, 1, 2]) — вправо
d.rotate(-1)       # deque([1, 2, 3]) — влево
```

## array — типизированный массив

Для числовых данных эффективнее использовать `array`:

```python
from array import array

# Массив целых чисел (signed int)
arr = array('i', [1, 2, 3, 4, 5])

arr.append(6)
arr[0]  # 1

# Типы: 'b' (int8), 'i' (int32), 'f' (float32), 'd' (float64)
```

Для научных вычислений — `numpy.array`.

## Когда что использовать?

| Задача | Лучший выбор |
|--------|--------------|
| Частый доступ по индексу | `list` |
| Частая вставка/удаление в начале | `deque` |
| Частая вставка/удаление с обоих концов | `deque` |
| Очередь (FIFO) | `deque` |
| Стек (LIFO) | `list` (append/pop) |
| Числовые вычисления | `numpy.array` |
| Нужна собственная логика | свой LinkedList |

## Практические задачи

### Развернуть связный список

```python
def reverse_linked_list(head):
    prev = None
    current = head
    while current:
        next_node = current.next
        current.next = prev
        prev = current
        current = next_node
    return prev  # новый head
```

### Найти середину списка

```python
def find_middle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
    return slow  # средний узел
```

### Обнаружить цикл

```python
def has_cycle(head):
    slow = fast = head
    while fast and fast.next:
        slow = slow.next
        fast = fast.next.next
        if slow == fast:
            return True
    return False
```

---

## Q&A

### Что такое FIFO и LIFO?

**FIFO — First In, First Out** ("Первым пришёл — первым вышел")

Как очередь в магазине:
```
Вход →  [Петя] [Маша] [Вася]  → Выход
Вася пришёл первым → Вася выйдет первым
```

```python
from collections import deque
queue = deque()
queue.append("Вася")    # добавить в конец
queue.append("Маша")
queue.popleft()         # "Вася" — первый вышел
```

**Где используется:** очереди задач, буфер печати, BFS.

---

**LIFO — Last In, First Out** ("Последним пришёл — первым вышел")

Как стопка тарелок:
```
    ┌───────┐
    │ Петя  │ ← последний (сверху)
    ├───────┤
    │ Маша  │
    ├───────┤
    │ Вася  │ ← первый (снизу)
    └───────┘
Петя положен последним → Петю возьмут первым
```

```python
stack = []
stack.append("Вася")    # добавить сверху
stack.append("Петя")
stack.pop()             # "Петя" — последний вышел первым
```

**Где используется:** Ctrl+Z (отмена), история браузера, call stack, DFS.

---

| | FIFO (Очередь) | LIFO (Стек) |
|---|----------------|-------------|
| Удаление | Из начала | Из конца |
| Python | `deque` | `list` |
| Аналогия | Очередь в магазине | Стопка тарелок |
