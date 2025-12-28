# Дек (Deque — Double-Ended Queue)

[prev: 01-queue-concept](./01-queue-concept.md) | [next: 03-priority-queue](./03-priority-queue.md)

---
## Определение

**Дек (Deque)** — это структура данных, которая позволяет добавлять и удалять элементы с обоих концов за O(1). Название происходит от "Double-Ended Queue" — двусторонняя очередь.

```
    Добавление/удаление             Добавление/удаление
           ↓                              ↓
    ┌──────────────────────────────────────────┐
    │  Front    │  B  │  C  │  D  │    Rear   │
    │    A      │     │     │     │     E     │
    └──────────────────────────────────────────┘
           ↑                              ↑
    Добавление/удаление             Добавление/удаление
```

## Аналогия

Представьте колоду карт:
- Можно взять карту **сверху** или **снизу**
- Можно положить карту **сверху** или **снизу**

## Зачем нужно

### Преимущества:
- **Универсальность** — может работать как стек и как очередь
- **Эффективность** — O(1) для всех базовых операций
- **Гибкость** — доступ с обоих концов

### Практическое применение:
- **Скользящее окно** — алгоритмы с окном фиксированного размера
- **Палиндромы** — проверка симметричности
- **История навигации** — браузер с кнопками "назад" и "вперёд"
- **Планировщики** — work stealing алгоритмы
- **Буферы** — двусторонняя обработка данных
- **Undo/Redo с ограничением** — хранение последних N операций

## Как работает

### Основные операции

```
ОПЕРАЦИИ С FRONT (началом):

push_front(X):
До:    [A, B, C]
После: [X, A, B, C]

pop_front():
До:    [A, B, C]
После: [B, C]  (возвращает A)


ОПЕРАЦИИ С REAR/BACK (концом):

push_back(X):
До:    [A, B, C]
После: [A, B, C, X]

pop_back():
До:    [A, B, C]
После: [A, B]  (возвращает C)
```

### Дек как универсальная структура

```
Дек может эмулировать:

СТЕК (LIFO):
- push = push_back
- pop = pop_back
   ┌───┐
   │ C │ ← push/pop
   ├───┤
   │ B │
   ├───┤
   │ A │
   └───┘

ОЧЕРЕДЬ (FIFO):
- enqueue = push_back
- dequeue = pop_front
   ┌───┬───┬───┐
   │ A │ B │ C │
   └───┴───┴───┘
     ↑       ↑
  dequeue  enqueue
```

## Псевдокод основных операций

### Реализация на основе кольцевого массива

```python
class ArrayDeque:
    def __init__(self, capacity=10):
        self.capacity = capacity
        self.data = [None] * capacity
        self.front = 0  # индекс первого элемента
        self.size = 0

    def is_empty(self):
        return self.size == 0

    def is_full(self):
        return self.size == self.capacity

    def _rear_index(self):
        """Индекс последнего элемента"""
        return (self.front + self.size - 1) % self.capacity

    def push_front(self, element):
        """Добавление в начало - O(1)"""
        if self.is_full():
            self._resize()

        # Сдвигаем front назад (циклически)
        self.front = (self.front - 1) % self.capacity
        self.data[self.front] = element
        self.size += 1

    def push_back(self, element):
        """Добавление в конец - O(1)"""
        if self.is_full():
            self._resize()

        rear = (self.front + self.size) % self.capacity
        self.data[rear] = element
        self.size += 1

    def pop_front(self):
        """Удаление из начала - O(1)"""
        if self.is_empty():
            raise IndexError("Deque is empty")

        element = self.data[self.front]
        self.data[self.front] = None
        self.front = (self.front + 1) % self.capacity
        self.size -= 1
        return element

    def pop_back(self):
        """Удаление из конца - O(1)"""
        if self.is_empty():
            raise IndexError("Deque is empty")

        rear = self._rear_index()
        element = self.data[rear]
        self.data[rear] = None
        self.size -= 1
        return element

    def front_element(self):
        """Просмотр первого элемента - O(1)"""
        if self.is_empty():
            raise IndexError("Deque is empty")
        return self.data[self.front]

    def back_element(self):
        """Просмотр последнего элемента - O(1)"""
        if self.is_empty():
            raise IndexError("Deque is empty")
        return self.data[self._rear_index()]

    def _resize(self):
        """Увеличение размера массива - O(n)"""
        new_capacity = self.capacity * 2
        new_data = [None] * new_capacity

        for i in range(self.size):
            new_data[i] = self.data[(self.front + i) % self.capacity]

        self.data = new_data
        self.front = 0
        self.capacity = new_capacity
```

### Реализация на основе двусвязного списка

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.prev = None
        self.next = None

class LinkedDeque:
    def __init__(self):
        self.head = None  # front
        self.tail = None  # back
        self._size = 0

    def is_empty(self):
        return self._size == 0

    def size(self):
        return self._size

    def push_front(self, element):
        """O(1)"""
        new_node = Node(element)

        if self.is_empty():
            self.head = self.tail = new_node
        else:
            new_node.next = self.head
            self.head.prev = new_node
            self.head = new_node

        self._size += 1

    def push_back(self, element):
        """O(1)"""
        new_node = Node(element)

        if self.is_empty():
            self.head = self.tail = new_node
        else:
            new_node.prev = self.tail
            self.tail.next = new_node
            self.tail = new_node

        self._size += 1

    def pop_front(self):
        """O(1)"""
        if self.is_empty():
            raise IndexError("Deque is empty")

        element = self.head.data

        if self.head == self.tail:
            self.head = self.tail = None
        else:
            self.head = self.head.next
            self.head.prev = None

        self._size -= 1
        return element

    def pop_back(self):
        """O(1)"""
        if self.is_empty():
            raise IndexError("Deque is empty")

        element = self.tail.data

        if self.head == self.tail:
            self.head = self.tail = None
        else:
            self.tail = self.tail.prev
            self.tail.next = None

        self._size -= 1
        return element

    def front(self):
        if self.is_empty():
            raise IndexError("Deque is empty")
        return self.head.data

    def back(self):
        if self.is_empty():
            raise IndexError("Deque is empty")
        return self.tail.data
```

### Использование collections.deque в Python

```python
from collections import deque

# Создание
d = deque()
d = deque([1, 2, 3])
d = deque(maxlen=5)  # ограниченный размер

# Операции с концами
d.append(4)       # push_back
d.appendleft(0)   # push_front
d.pop()           # pop_back
d.popleft()       # pop_front

# Просмотр
d[0]              # front
d[-1]             # back

# Дополнительные операции
d.extend([5, 6])  # добавить несколько в конец
d.extendleft([0, -1])  # добавить несколько в начало (в обратном порядке!)
d.rotate(2)       # сдвиг вправо на 2
d.rotate(-2)      # сдвиг влево на 2
d.reverse()       # реверс на месте
d.count(x)        # количество вхождений x
d.remove(x)       # удалить первое вхождение x
d.clear()         # очистить
```

## Анализ сложности

| Операция | Массив (кольцевой) | Связный список | collections.deque |
|----------|-------------------|----------------|-------------------|
| push_front | O(1)* | O(1) | O(1) |
| push_back | O(1)* | O(1) | O(1) |
| pop_front | O(1) | O(1) | O(1) |
| pop_back | O(1) | O(1) | O(1) |
| front/back | O(1) | O(1) | O(1) |
| Доступ по индексу | O(1) | O(n) | O(1) |

*Амортизированное (с resize)

### Память
- **Кольцевой массив**: O(capacity)
- **Связный список**: O(n) + указатели
- **collections.deque**: O(n), блочная структура

## Примеры с разбором

### Пример 1: Максимум в скользящем окне (монотонный дек)

```python
from collections import deque

def max_sliding_window(nums, k):
    """
    Находит максимум в каждом окне размера k.
    Использует монотонный дек для O(n) решения.

    nums = [1, 3, -1, -3, 5, 3, 6, 7], k = 3
    Результат: [3, 3, 5, 5, 6, 7]
    """
    if not nums:
        return []

    result = []
    dq = deque()  # хранит индексы в порядке убывания значений

    for i in range(len(nums)):
        # Удаляем индексы вне текущего окна
        while dq and dq[0] < i - k + 1:
            dq.popleft()

        # Удаляем индексы элементов меньше текущего
        # (они уже не могут быть максимумом)
        while dq and nums[dq[-1]] < nums[i]:
            dq.pop()

        dq.append(i)

        # Записываем максимум (элемент на позиции dq[0])
        if i >= k - 1:
            result.append(nums[dq[0]])

    return result

# Трассировка для [1, 3, -1, -3, 5, 3, 6, 7], k=3:
#
# i=0: nums[0]=1, dq=[], append 0 → dq=[0]
# i=1: nums[1]=3 > nums[0]=1, pop 0, append 1 → dq=[1]
# i=2: nums[2]=-1 < nums[1]=3, append 2 → dq=[1,2]
#      window complete, max = nums[1] = 3
# i=3: 1 >= 3-3+1=1, 1 stays. nums[3]=-3 < nums[2], append 3 → dq=[1,2,3]
#      max = nums[1] = 3
# i=4: 1 < 4-3+1=2, pop 1 → dq=[2,3]
#      nums[4]=5 > nums[3] and nums[2], pop both → dq=[4]
#      max = nums[4] = 5
# ...
```

### Пример 2: Проверка палиндрома

```python
from collections import deque

def is_palindrome(s):
    """
    Проверяет, является ли строка палиндромом.
    Использует дек для сравнения символов с концов.
    """
    # Фильтруем только буквы и цифры
    cleaned = deque(c.lower() for c in s if c.isalnum())

    while len(cleaned) > 1:
        if cleaned.popleft() != cleaned.pop():
            return False

    return True

# Примеры:
is_palindrome("A man, a plan, a canal: Panama")  # True
is_palindrome("race a car")  # False
is_palindrome("Was it a car or a cat I saw?")  # True
```

```
Визуализация для "kayak":

Исходный дек: [k, a, y, a, k]

Шаг 1: popleft() = 'k', pop() = 'k' → равны ✓
       дек: [a, y, a]

Шаг 2: popleft() = 'a', pop() = 'a' → равны ✓
       дек: [y]

Шаг 3: len == 1, выход

Результат: True
```

### Пример 3: Ограниченная история (LRU-подобная)

```python
from collections import deque

class BoundedHistory:
    """
    История с ограниченным размером.
    Автоматически удаляет старые записи.
    """
    def __init__(self, max_size=100):
        self.history = deque(maxlen=max_size)

    def add(self, item):
        """Добавляет запись в историю"""
        self.history.append(item)
        # Старые записи удаляются автоматически

    def get_recent(self, n=10):
        """Получает n последних записей"""
        return list(self.history)[-n:]

    def search(self, query):
        """Поиск в истории"""
        return [item for item in self.history if query in item]

# Использование:
history = BoundedHistory(max_size=1000)
history.add("git status")
history.add("git add .")
history.add("git commit -m 'fix'")

history.get_recent(2)  # ['git add .', 'git commit -m "fix"']
history.search("git")  # все команды git
```

### Пример 4: Work Stealing Scheduler

```python
from collections import deque
import threading

class WorkStealingQueue:
    """
    Очередь для алгоритма work stealing.
    Владелец добавляет/забирает с конца (LIFO).
    Воры забирают с начала (FIFO).
    """
    def __init__(self):
        self.deque = deque()
        self.lock = threading.Lock()

    def push(self, task):
        """Владелец добавляет задачу (O(1))"""
        self.deque.append(task)

    def pop(self):
        """Владелец забирает задачу (LIFO, O(1))"""
        if not self.deque:
            return None
        return self.deque.pop()

    def steal(self):
        """Вор забирает задачу (FIFO, O(1))"""
        with self.lock:
            if not self.deque:
                return None
            return self.deque.popleft()

# Использование:
# Каждый поток имеет свою очередь
# Когда очередь пуста, поток "ворует" у других
```

### Пример 5: Первый отрицательный в каждом окне

```python
from collections import deque

def first_negative_in_window(nums, k):
    """
    Находит первый отрицательный элемент в каждом окне размера k.
    Если нет отрицательного — возвращает 0.

    nums = [12, -1, -7, 8, -15, 30, 16, 28], k = 3
    Окна: [12, -1, -7] → -1
          [-1, -7, 8] → -1
          [-7, 8, -15] → -7
          [8, -15, 30] → -15
          [-15, 30, 16] → -15
          [30, 16, 28] → 0
    Результат: [-1, -1, -7, -15, -15, 0]
    """
    result = []
    negatives = deque()  # индексы отрицательных элементов

    for i in range(len(nums)):
        # Добавляем отрицательные элементы
        if nums[i] < 0:
            negatives.append(i)

        # Удаляем элементы вне окна
        while negatives and negatives[0] < i - k + 1:
            negatives.popleft()

        # Записываем результат для полных окон
        if i >= k - 1:
            if negatives:
                result.append(nums[negatives[0]])
            else:
                result.append(0)

    return result
```

### Пример 6: Реверс подмассива

```python
from collections import deque

def reverse_first_k(d, k):
    """
    Разворачивает первые k элементов дека.

    d = [1, 2, 3, 4, 5], k = 3
    Результат: [3, 2, 1, 4, 5]
    """
    if k > len(d):
        k = len(d)

    # Извлекаем первые k элементов
    temp = []
    for _ in range(k):
        temp.append(d.popleft())

    # Добавляем обратно в обратном порядке
    for item in temp:
        d.appendleft(item)

    return d

# Или используя rotate:
def reverse_first_k_v2(d, k):
    """Альтернативная реализация"""
    # Переносим первые k элементов в конец
    d.rotate(-(k))  # теперь [4, 5, 1, 2, 3]

    # Разворачиваем "хвост" (бывшие первые k элементов)
    # Нужно вручную развернуть последние k элементов

    # Проще использовать стек:
    stack = [d.pop() for _ in range(k)]
    d.extend(stack)  # [4, 5, 3, 2, 1]
    d.rotate(k)      # [3, 2, 1, 4, 5]

    return d
```

## Типичные ошибки

### 1. Путаница с направлением rotate

```python
from collections import deque

d = deque([1, 2, 3, 4, 5])

d.rotate(2)   # вправо: [4, 5, 1, 2, 3]
d.rotate(-2)  # влево: [3, 4, 5, 1, 2]

# rotate(n) эквивалентно:
# for _ in range(n): d.appendleft(d.pop())
```

### 2. extendleft добавляет в обратном порядке

```python
from collections import deque

d = deque([1, 2, 3])
d.extendleft([4, 5, 6])  # НЕ [4, 5, 6, 1, 2, 3]!
print(d)  # deque([6, 5, 4, 1, 2, 3])

# Если нужен порядок [4, 5, 6, 1, 2, 3]:
d = deque([1, 2, 3])
d.extendleft(reversed([4, 5, 6]))
```

### 3. Модификация дека во время итерации

```python
from collections import deque

d = deque([1, 2, 3, 4, 5])

# НЕПРАВИЛЬНО
for item in d:
    if item % 2 == 0:
        d.remove(item)  # RuntimeError или пропуск элементов

# ПРАВИЛЬНО
d = deque(item for item in d if item % 2 != 0)
```

### 4. Забытый случай пустого дека

```python
# НЕПРАВИЛЬНО
def get_max(dq):
    return dq[0]  # IndexError если дек пуст

# ПРАВИЛЬНО
def get_max(dq):
    if not dq:
        return None
    return dq[0]
```

### 5. Неправильный индекс в монотонном деке

```python
# В монотонном деке храним ИНДЕКСЫ, не значения!

# НЕПРАВИЛЬНО
while dq and dq[-1] < nums[i]:  # сравниваем индексы с значениями!
    dq.pop()

# ПРАВИЛЬНО
while dq and nums[dq[-1]] < nums[i]:  # сравниваем значения
    dq.pop()
```

## Сравнение структур

| Операция | Stack | Queue | Deque |
|----------|-------|-------|-------|
| push_front | - | - | O(1) |
| push_back | O(1) | O(1) | O(1) |
| pop_front | - | O(1) | O(1) |
| pop_back | O(1) | - | O(1) |
| Эмулирует | - | - | Stack + Queue |

## Когда использовать Deque

**Используй deque, когда:**
- Нужны O(1) операции с обоих концов
- Реализуешь скользящее окно
- Нужен стек И очередь
- Нужна ограниченная история (maxlen)
- Work stealing алгоритмы

**НЕ используй deque, когда:**
- Нужен только LIFO (стек проще)
- Нужен только FIFO (queue проще)
- Нужен доступ по произвольному индексу (массив лучше)
- Нужна приоритизация (используй heap)

---

[prev: 01-queue-concept](./01-queue-concept.md) | [next: 03-priority-queue](./03-priority-queue.md)
