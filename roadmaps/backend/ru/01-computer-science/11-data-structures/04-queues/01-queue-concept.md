# Очередь — концепция (Queue Concept)

[prev: 02-stack-applications](../03-stacks/02-stack-applications.md) | [next: 02-deque](./02-deque.md)

---
## Определение

**Очередь** — это линейная структура данных, работающая по принципу FIFO (First In, First Out — «первым пришёл, первым вышел»). Элементы добавляются в конец очереди (rear/back) и удаляются из начала (front).

```
Добавление (enqueue)                    Удаление (dequeue)
      │                                       │
      ▼                                       ▼
┌─────────────────────────────────────────────────┐
│  Front    │  B  │  C  │  D  │  E  │    Rear    │
│    A      │     │     │     │     │     F      │
└─────────────────────────────────────────────────┘
      │                                       │
      └──── выход (первый)     вход (последний) ────┘
```

## Аналогия

Представьте очередь в магазине:
- Новые покупатели встают **в конец** очереди
- Обслуживаются покупатели **в начале** очереди
- Кто раньше пришёл — тот раньше уйдёт

## Зачем нужно

### Практическое применение:
- **Планировщики задач** — task queues, job schedulers
- **Буферы данных** — сетевые пакеты, потоки ввода/вывода
- **Обход графов в ширину (BFS)** — поиск кратчайшего пути
- **Асинхронная обработка** — message queues (RabbitMQ, Kafka)
- **Печать документов** — очередь печати
- **Обслуживание клиентов** — call centers, техподдержка

## Как работает

### Основные операции

```
ENQUEUE (добавление в конец):

До:    front → [A] → [B] → [C] ← rear

       enqueue(D)
       ↓
После: front → [A] → [B] → [C] → [D] ← rear


DEQUEUE (удаление из начала):

До:    front → [A] → [B] → [C] ← rear
               ↓
         dequeue() возвращает A

После: front → [B] → [C] ← rear
```

### Дополнительные операции

```
PEEK / FRONT (просмотр первого элемента):

front → [A] → [B] → [C] ← rear
         ↑
    front() возвращает A, но не удаляет


IS_EMPTY (проверка пустоты):

Пустая:   front = rear = null → is_empty() = True
Не пустая: front → [A] → is_empty() = False


SIZE (размер):

front → [A] → [B] → [C] ← rear → size() = 3
```

## Псевдокод основных операций

### Реализация на основе массива (кольцевой буфер)

```python
class ArrayQueue:
    def __init__(self, capacity=10):
        self.capacity = capacity
        self.data = [None] * capacity
        self.front = 0  # индекс первого элемента
        self.rear = 0   # индекс следующей свободной позиции
        self.size = 0

    def is_empty(self):
        """Проверка пустоты - O(1)"""
        return self.size == 0

    def is_full(self):
        """Проверка заполненности - O(1)"""
        return self.size == self.capacity

    def enqueue(self, element):
        """Добавление элемента - O(1)"""
        if self.is_full():
            raise OverflowError("Queue is full")

        self.data[self.rear] = element
        self.rear = (self.rear + 1) % self.capacity  # кольцевой переход
        self.size += 1

    def dequeue(self):
        """Удаление и возврат элемента - O(1)"""
        if self.is_empty():
            raise IndexError("Queue is empty")

        element = self.data[self.front]
        self.data[self.front] = None  # очистка
        self.front = (self.front + 1) % self.capacity  # кольцевой переход
        self.size -= 1
        return element

    def peek(self):
        """Просмотр первого элемента - O(1)"""
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self.data[self.front]
```

```
Кольцевой буфер — визуализация:

Массив: [_, _, _, _, _, _, _, _]  capacity = 8
         0  1  2  3  4  5  6  7

После enqueue(A, B, C, D, E):
        [A, B, C, D, E, _, _, _]
         ↑              ↑
       front          rear

После dequeue(), dequeue():
        [_, _, C, D, E, _, _, _]
               ↑        ↑
             front    rear

После enqueue(F, G, H, I):
        [I, _, C, D, E, F, G, H]
            ↑  ↑
          rear front

Кольцевой переход: rear = (rear + 1) % capacity
```

### Реализация на основе динамического массива (Python list)

```python
class SimpleQueue:
    def __init__(self):
        self.data = []

    def is_empty(self):
        return len(self.data) == 0

    def size(self):
        return len(self.data)

    def enqueue(self, element):
        """O(1) амортизированное"""
        self.data.append(element)

    def dequeue(self):
        """O(n) — нужно сдвигать все элементы!"""
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self.data.pop(0)  # НЕЭФФЕКТИВНО!

    def peek(self):
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self.data[0]
```

**Важно**: `pop(0)` имеет сложность O(n)! Для эффективной очереди используйте `collections.deque`.

### Реализация на основе связного списка

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.next = None

class LinkedQueue:
    def __init__(self):
        self.front = None  # голова для dequeue
        self.rear = None   # хвост для enqueue
        self._size = 0

    def is_empty(self):
        return self.front is None

    def size(self):
        return self._size

    def enqueue(self, element):
        """Добавление в конец - O(1)"""
        new_node = Node(element)

        if self.rear is None:
            self.front = self.rear = new_node
        else:
            self.rear.next = new_node
            self.rear = new_node

        self._size += 1

    def dequeue(self):
        """Удаление из начала - O(1)"""
        if self.is_empty():
            raise IndexError("Queue is empty")

        element = self.front.data
        self.front = self.front.next

        if self.front is None:
            self.rear = None

        self._size -= 1
        return element

    def peek(self):
        """O(1)"""
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self.front.data
```

```
Связный список как очередь:

enqueue(A): front → [A] ← rear
enqueue(B): front → [A] → [B] ← rear
enqueue(C): front → [A] → [B] → [C] ← rear
dequeue():  front → [B] → [C] ← rear  (возвращает A)
```

### Использование collections.deque (рекомендуется в Python)

```python
from collections import deque

class EfficientQueue:
    def __init__(self):
        self.data = deque()

    def is_empty(self):
        return len(self.data) == 0

    def size(self):
        return len(self.data)

    def enqueue(self, element):
        """O(1)"""
        self.data.append(element)

    def dequeue(self):
        """O(1) — в отличие от list.pop(0)!"""
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self.data.popleft()

    def peek(self):
        if self.is_empty():
            raise IndexError("Queue is empty")
        return self.data[0]
```

## Анализ сложности

| Операция | Массив (линейный) | Массив (кольцевой) | Связный список | deque |
|----------|-------------------|--------------------|--------------------|-------|
| enqueue | O(1)* | O(1) | O(1) | O(1) |
| dequeue | **O(n)** | O(1) | O(1) | O(1) |
| peek | O(1) | O(1) | O(1) | O(1) |
| is_empty | O(1) | O(1) | O(1) | O(1) |

*Амортизированное для динамического массива

### Память
- **Массив**: O(capacity) — фиксированная или динамическая
- **Связный список**: O(n) — дополнительная память на указатели
- **deque**: O(n) — оптимизированная блочная структура

## Примеры с разбором

### Пример 1: Обход графа в ширину (BFS)

```python
from collections import deque

def bfs(graph, start):
    """
    Обход графа в ширину.
    Возвращает порядок посещения вершин.
    """
    visited = set()
    queue = deque([start])
    result = []

    while queue:
        node = queue.popleft()  # dequeue

        if node in visited:
            continue

        visited.add(node)
        result.append(node)

        for neighbor in graph[node]:
            if neighbor not in visited:
                queue.append(neighbor)  # enqueue

    return result

# Граф:
#     A
#    / \
#   B   C
#  / \   \
# D   E   F

graph = {
    'A': ['B', 'C'],
    'B': ['D', 'E'],
    'C': ['F'],
    'D': [],
    'E': [],
    'F': []
}

print(bfs(graph, 'A'))  # ['A', 'B', 'C', 'D', 'E', 'F']
```

```
Трассировка BFS:

Шаг 1: queue = [A], visited = {}
       dequeue A → visited = {A}, result = [A]
       enqueue B, C → queue = [B, C]

Шаг 2: queue = [B, C]
       dequeue B → visited = {A, B}, result = [A, B]
       enqueue D, E → queue = [C, D, E]

Шаг 3: queue = [C, D, E]
       dequeue C → visited = {A, B, C}, result = [A, B, C]
       enqueue F → queue = [D, E, F]

Шаг 4-6: dequeue D, E, F
       result = [A, B, C, D, E, F]
```

### Пример 2: Поиск кратчайшего пути в лабиринте

```python
from collections import deque

def shortest_path(maze, start, end):
    """
    Находит кратчайший путь в лабиринте.
    BFS гарантирует нахождение кратчайшего пути!
    """
    rows, cols = len(maze), len(maze[0])
    visited = set()
    queue = deque([(start, [start])])  # (позиция, путь)

    while queue:
        (row, col), path = queue.popleft()

        if (row, col) == end:
            return path

        if (row, col) in visited:
            continue

        visited.add((row, col))

        # Все направления
        for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            new_row, new_col = row + dr, col + dc

            if (0 <= new_row < rows and
                0 <= new_col < cols and
                maze[new_row][new_col] == 0 and
                (new_row, new_col) not in visited):
                queue.append(((new_row, new_col), path + [(new_row, new_col)]))

    return None  # путь не найден
```

### Пример 3: Очередь задач с таймаутом

```python
import time
from collections import deque

class TaskQueue:
    def __init__(self, timeout=5):
        self.queue = deque()
        self.timeout = timeout

    def add_task(self, task):
        """Добавляет задачу с временной меткой"""
        self.queue.append({
            'task': task,
            'timestamp': time.time()
        })

    def get_task(self):
        """Получает задачу, пропуская устаревшие"""
        current_time = time.time()

        while self.queue:
            item = self.queue.popleft()

            if current_time - item['timestamp'] <= self.timeout:
                return item['task']
            else:
                print(f"Task expired: {item['task']}")

        return None  # нет активных задач

# Использование:
tq = TaskQueue(timeout=10)
tq.add_task("process_payment")
tq.add_task("send_email")

task = tq.get_task()  # "process_payment"
```

### Пример 4: Hot Potato (игра)

```python
from collections import deque
import random

def hot_potato(names, num_passes):
    """
    Игра "Горячая картошка".
    Дети передают картошку по кругу. Когда счёт заканчивается,
    держащий картошку выбывает.
    """
    queue = deque(names)

    while len(queue) > 1:
        # Передаём картошку num_passes раз
        for _ in range(num_passes):
            # Передаём: убираем первого, добавляем в конец
            queue.append(queue.popleft())

        # Кто держит картошку — выбывает
        eliminated = queue.popleft()
        print(f"{eliminated} выбывает!")

    return queue[0]  # победитель

# Пример:
names = ["Аня", "Боря", "Вася", "Галя", "Даня"]
winner = hot_potato(names, 7)
print(f"Победитель: {winner}")
```

### Пример 5: Скользящее окно (максимум в окне)

```python
from collections import deque

def max_sliding_window(nums, k):
    """
    Находит максимум в каждом окне размера k.

    nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3
    Окна:  [1, 3, -1] = 3
           [3, -1, -3] = 3
           [-1, -3, 5] = 5
           [-3, 5, 3] = 5
           [5, 3, 6] = 6
           [3, 6, 7] = 7
    Результат: [3, 3, 5, 5, 6, 7]
    """
    if not nums or k == 0:
        return []

    result = []
    dq = deque()  # хранит индексы в порядке убывания значений

    for i, num in enumerate(nums):
        # Удаляем элементы вне текущего окна
        while dq and dq[0] < i - k + 1:
            dq.popleft()

        # Удаляем элементы меньше текущего (они не могут быть максимумом)
        while dq and nums[dq[-1]] < num:
            dq.pop()

        dq.append(i)

        # Записываем максимум, начиная с полного окна
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result
```

## Типичные ошибки

### 1. Использование list.pop(0) вместо deque

```python
# НЕПРАВИЛЬНО — O(n) на каждый dequeue!
class SlowQueue:
    def dequeue(self):
        return self.data.pop(0)  # сдвигает все элементы

# ПРАВИЛЬНО — O(1)
from collections import deque
class FastQueue:
    def dequeue(self):
        return self.data.popleft()
```

### 2. Забытое обновление rear в связном списке

```python
# НЕПРАВИЛЬНО
def dequeue(self):
    element = self.front.data
    self.front = self.front.next
    return element
    # Если удалили последний элемент, rear остаётся висячим!

# ПРАВИЛЬНО
def dequeue(self):
    element = self.front.data
    self.front = self.front.next
    if self.front is None:
        self.rear = None  # очередь опустела
    return element
```

### 3. Ошибка границ в кольцевом буфере

```python
# НЕПРАВИЛЬНО
def enqueue(self, element):
    self.rear += 1  # выйдет за границы!
    self.data[self.rear] = element

# ПРАВИЛЬНО
def enqueue(self, element):
    self.data[self.rear] = element
    self.rear = (self.rear + 1) % self.capacity  # кольцевой переход
```

### 4. Проверка полноты кольцевого буфера

```python
# Проблема: как отличить пустую очередь от полной?
# Если front == rear — пусто или полно?

# Решение 1: хранить size
def is_full(self):
    return self.size == self.capacity

# Решение 2: оставлять одну ячейку пустой
def is_full(self):
    return (self.rear + 1) % self.capacity == self.front
```

### 5. Неправильный порядок в BFS

```python
# НЕПРАВИЛЬНО — пропуск вершин
queue = deque([start])
while queue:
    node = queue.popleft()
    visited.add(node)  # добавляем при извлечении
    for neighbor in graph[node]:
        if neighbor not in visited:
            queue.append(neighbor)
    # Проблема: одна вершина может добавиться несколько раз!

# ПРАВИЛЬНО — отмечаем сразу при добавлении
queue = deque([start])
visited = {start}  # отмечаем сразу
while queue:
    node = queue.popleft()
    for neighbor in graph[node]:
        if neighbor not in visited:
            visited.add(neighbor)  # отмечаем при добавлении
            queue.append(neighbor)
```

## Сравнение Stack и Queue

| Аспект | Stack (LIFO) | Queue (FIFO) |
|--------|--------------|--------------|
| Принцип | Последний вошёл — первый вышел | Первый вошёл — первый вышел |
| Добавление | push (на вершину) | enqueue (в конец) |
| Удаление | pop (с вершины) | dequeue (из начала) |
| Просмотр | peek/top | front/peek |
| Применение | DFS, undo, вызовы функций | BFS, очереди задач |
| Аналогия | Стопка тарелок | Очередь в магазине |

## Когда использовать очередь

**Используй очередь, когда:**
- Нужен порядок FIFO
- Обработка в порядке поступления
- Буферизация данных между producer и consumer
- Обход графа в ширину (BFS)
- Планирование задач

**НЕ используй очередь, когда:**
- Нужен порядок LIFO (используй стек)
- Нужен доступ к произвольным элементам
- Нужна приоритизация (используй priority queue)

---

[prev: 02-stack-applications](../03-stacks/02-stack-applications.md) | [next: 02-deque](./02-deque.md)
