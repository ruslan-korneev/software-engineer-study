# Кучи, стеки и очереди

[prev: 02-hash-tables](./02-hash-tables.md) | [next: 04-binary-search-tree](./04-binary-search-tree.md)
---

Три важные структуры данных с разным поведением.

## Stack (Стек) — LIFO

**Last In, First Out** — последний пришёл, первый вышел.

```
    push(3)     push(7)     pop()
    ┌───┐       ┌───┐       ┌───┐
    │ 3 │       │ 7 │       │ 3 │
    └───┘       ├───┤       └───┘
                │ 3 │
                └───┘       return 7
```

### Операции

| Операция | Описание | Сложность |
|----------|----------|-----------|
| `push` | Добавить на вершину | O(1) |
| `pop` | Удалить с вершины | O(1) |
| `peek/top` | Посмотреть вершину | O(1) |
| `isEmpty` | Проверить пустоту | O(1) |

### Python — используй list

```python
stack = []

# push
stack.append(1)
stack.append(2)
stack.append(3)
# stack = [1, 2, 3]

# peek — посмотреть верхний
stack[-1]       # 3

# pop — удалить и вернуть верхний
stack.pop()     # 3, stack = [1, 2]
stack.pop()     # 2, stack = [1]

# isEmpty
len(stack) == 0  # False
not stack        # False
```

### Реализация класса Stack

```python
class Stack:
    def __init__(self):
        self._items = []

    def push(self, item):
        self._items.append(item)

    def pop(self):
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self._items.pop()

    def peek(self):
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self._items[-1]

    def is_empty(self):
        return len(self._items) == 0

    def size(self):
        return len(self._items)
```

### Применение стека

- **История браузера** — кнопка "назад"
- **Ctrl+Z** — отмена действий
- **Call stack** — вызовы функций
- **Проверка скобок** — `((()))`
- **DFS** — поиск в глубину
- **Вычисление выражений** — постфиксная нотация

### Пример: проверка скобок

```python
def is_valid_parentheses(s):
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}

    for char in s:
        if char in '([{':
            stack.append(char)
        elif char in ')]}':
            if not stack or stack[-1] != pairs[char]:
                return False
            stack.pop()

    return len(stack) == 0

is_valid_parentheses("((()))")   # True
is_valid_parentheses("([{}])")   # True
is_valid_parentheses("(()")      # False
is_valid_parentheses("([)]")     # False
```

---

## Queue (Очередь) — FIFO

**First In, First Out** — первый пришёл, первый вышел.

```
enqueue(1)   enqueue(2)   enqueue(3)   dequeue()

┌───┐        ┌───┬───┐    ┌───┬───┬───┐   ┌───┬───┐
│ 1 │   →    │ 1 │ 2 │ →  │ 1 │ 2 │ 3 │ → │ 2 │ 3 │
└───┘        └───┴───┘    └───┴───┴───┘   └───┴───┘
front        front  back                  front back
                                          return 1
```

### Операции

| Операция | Описание | Сложность |
|----------|----------|-----------|
| `enqueue` | Добавить в конец | O(1) |
| `dequeue` | Удалить из начала | O(1) |
| `front` | Посмотреть первый | O(1) |
| `isEmpty` | Проверить пустоту | O(1) |

### Python — используй deque

```python
from collections import deque

queue = deque()

# enqueue — добавить в конец
queue.append(1)
queue.append(2)
queue.append(3)
# deque([1, 2, 3])

# front — посмотреть первый
queue[0]          # 1

# dequeue — удалить из начала
queue.popleft()   # 1, deque([2, 3])
queue.popleft()   # 2, deque([3])

# isEmpty
len(queue) == 0   # False
```

### Почему не list?

```python
# list.pop(0) — O(n), сдвигает все элементы!
lst = [1, 2, 3, 4, 5]
lst.pop(0)  # O(n) — плохо для очереди

# deque.popleft() — O(1)
from collections import deque
d = deque([1, 2, 3, 4, 5])
d.popleft()  # O(1) — хорошо!
```

### Реализация класса Queue

```python
from collections import deque

class Queue:
    def __init__(self):
        self._items = deque()

    def enqueue(self, item):
        self._items.append(item)

    def dequeue(self):
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self._items.popleft()

    def front(self):
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self._items[0]

    def is_empty(self):
        return len(self._items) == 0

    def size(self):
        return len(self._items)
```

### Применение очереди

- **Очередь задач** — обработка запросов
- **Буфер печати** — документы печатаются по порядку
- **BFS** — поиск в ширину
- **Буферы I/O** — чтение/запись данных

### Пример: BFS (поиск в ширину)

```python
from collections import deque

def bfs(graph, start):
    visited = set()
    queue = deque([start])
    result = []

    while queue:
        node = queue.popleft()
        if node not in visited:
            visited.add(node)
            result.append(node)
            queue.extend(graph.get(node, []))

    return result

graph = {
    'A': ['B', 'C'],
    'B': ['D', 'E'],
    'C': ['F'],
    'D': [], 'E': [], 'F': []
}
bfs(graph, 'A')  # ['A', 'B', 'C', 'D', 'E', 'F']
```

---

## Heap (Куча) — приоритетная очередь

Бинарное дерево со свойством кучи:
- **Min-Heap**: родитель ≤ детей (минимум в корне)
- **Max-Heap**: родитель ≥ детей (максимум в корне)

```
Min-Heap:           Max-Heap:
      1                   9
     / \                 / \
    3   2               7   8
   / \                 / \
  7   4               3   5
```

### Операции

| Операция | Описание | Сложность |
|----------|----------|-----------|
| `push` | Добавить элемент | O(log n) |
| `pop` | Удалить корень (мин/макс) | O(log n) |
| `peek` | Посмотреть корень | O(1) |
| `heapify` | Создать кучу из списка | O(n) |

### Python — модуль heapq (min-heap)

```python
import heapq

heap = []

# push — добавить
heapq.heappush(heap, 3)
heapq.heappush(heap, 1)
heapq.heappush(heap, 4)
heapq.heappush(heap, 1)
heapq.heappush(heap, 5)
# heap = [1, 1, 4, 3, 5] — внутреннее представление

# peek — посмотреть минимум
heap[0]  # 1

# pop — извлечь минимум
heapq.heappop(heap)  # 1
heapq.heappop(heap)  # 1
heapq.heappop(heap)  # 3
```

### heapify — создать кучу из списка

```python
import heapq

lst = [5, 3, 8, 1, 2]
heapq.heapify(lst)  # O(n) — эффективнее чем n × push
# lst = [1, 2, 8, 5, 3]

heapq.heappop(lst)  # 1
heapq.heappop(lst)  # 2
```

### Max-Heap через отрицание

Python heapq — только min-heap. Для max-heap используй отрицание:

```python
import heapq

max_heap = []

# Добавляем с минусом
heapq.heappush(max_heap, -5)
heapq.heappush(max_heap, -2)
heapq.heappush(max_heap, -8)
heapq.heappush(max_heap, -1)

# Извлекаем и убираем минус
-heapq.heappop(max_heap)  # 8 (максимум)
-heapq.heappop(max_heap)  # 5
```

### nlargest / nsmallest

```python
import heapq

nums = [3, 1, 4, 1, 5, 9, 2, 6]

heapq.nlargest(3, nums)   # [9, 6, 5] — 3 наибольших
heapq.nsmallest(3, nums)  # [1, 1, 2] — 3 наименьших

# С ключом
users = [
    {"name": "Alice", "age": 30},
    {"name": "Bob", "age": 25},
    {"name": "Charlie", "age": 35}
]
heapq.nlargest(2, users, key=lambda x: x["age"])
# [{"name": "Charlie", "age": 35}, {"name": "Alice", "age": 30}]
```

### Применение кучи

- **Топ-K элементов** — k наибольших/наименьших
- **Медиана в потоке** — две кучи (min + max)
- **Алгоритм Дейкстры** — кратчайший путь
- **Планировщик задач** — по приоритету
- **Merge K sorted lists** — слияние отсортированных списков

### Пример: топ-K частых элементов

```python
import heapq
from collections import Counter

def top_k_frequent(nums, k):
    counts = Counter(nums)
    return heapq.nlargest(k, counts.keys(), key=counts.get)

top_k_frequent([1, 1, 1, 2, 2, 3], 2)  # [1, 2]
```

---

## Deque — двусторонняя очередь

Добавление и удаление с обоих концов за O(1).

```python
from collections import deque

d = deque([2, 3, 4])

# Добавление
d.append(5)       # [2, 3, 4, 5] — справа
d.appendleft(1)   # [1, 2, 3, 4, 5] — слева

# Удаление
d.pop()           # 5, [1, 2, 3, 4] — справа
d.popleft()       # 1, [2, 3, 4] — слева

# Ротация
d = deque([1, 2, 3, 4, 5])
d.rotate(2)       # [4, 5, 1, 2, 3] — вправо
d.rotate(-2)      # [1, 2, 3, 4, 5] — влево

# Ограниченный размер
d = deque(maxlen=3)
d.append(1)  # [1]
d.append(2)  # [1, 2]
d.append(3)  # [1, 2, 3]
d.append(4)  # [2, 3, 4] — первый вытеснен
```

---

## Priority Queue (из модуля queue)

Потокобезопасная приоритетная очередь:

```python
from queue import PriorityQueue

pq = PriorityQueue()

# Добавляем (приоритет, данные)
pq.put((2, "средний приоритет"))
pq.put((1, "высокий приоритет"))
pq.put((3, "низкий приоритет"))

# Извлекаем — сначала с меньшим числом
pq.get()  # (1, "высокий приоритет")
pq.get()  # (2, "средний приоритет")
pq.get()  # (3, "низкий приоритет")
```

---

## Сравнение структур

| Структура | Порядок извлечения | Python | Применение |
|-----------|-------------------|--------|------------|
| Stack | LIFO (последний) | `list` | Отмена, DFS, скобки |
| Queue | FIFO (первый) | `deque` | Задачи, BFS |
| Heap | По приоритету | `heapq` | Top-K, Дейкстра |
| Deque | С любого конца | `deque` | Скользящее окно |
| PriorityQueue | По приоритету (потокобезопасно) | `queue.PriorityQueue` | Многопоточность |

## Сложность операций

| Операция | Stack | Queue | Heap |
|----------|-------|-------|------|
| Добавить | O(1) | O(1) | O(log n) |
| Удалить | O(1) | O(1) | O(log n) |
| Посмотреть | O(1) | O(1) | O(1) |
| Поиск | O(n) | O(n) | O(n) |

---
[prev: 02-hash-tables](./02-hash-tables.md) | [next: 04-binary-search-tree](./04-binary-search-tree.md)