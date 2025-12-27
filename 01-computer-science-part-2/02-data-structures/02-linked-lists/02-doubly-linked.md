# Двусвязный список (Doubly Linked List)

## Определение

**Двусвязный список** — это линейная структура данных, в которой каждый узел содержит данные и два указателя: на следующий узел (`next`) и на предыдущий (`prev`). Это позволяет перемещаться по списку в обоих направлениях.

```
        ┌──────────────────────────────────────────────────┐
        │                                                  │
        ▼                                                  │
None ◄──┬────────┐    ┌────────┐    ┌────────┐    ┌────────┼──► None
        │   10   │◄──►│   20   │◄──►│   30   │◄──►│   40   │
        └────────┘    └────────┘    └────────┘    └────────┘
           HEAD                                      TAIL
```

## Зачем нужно

### Преимущества перед односвязным списком:
- **Двунаправленный обход** — можно идти в любую сторону
- **Быстрое удаление узла** — O(1) если есть указатель на узел
- **Удаление из конца** — O(1) без необходимости искать предпоследний узел

### Практическое применение:
- **Браузерная навигация** — кнопки "назад" и "вперёд"
- **Редакторы текста** — курсор, undo/redo
- **Кэши LRU** — Least Recently Used cache
- **Deque** — двусторонняя очередь
- **Музыкальные плееры** — переключение треков вперёд/назад

## Как работает

### Структура узла

```
┌─────────────────────────┐
│         Node            │
├─────────────────────────┤
│  prev: Node*            │  ← указатель на предыдущий
├─────────────────────────┤
│  data: любой тип        │  ← хранимое значение
├─────────────────────────┤
│  next: Node*            │  ← указатель на следующий
└─────────────────────────┘
```

### Структура списка

```
DoublyLinkedList
├── head: Node*    ← указатель на первый узел
├── tail: Node*    ← указатель на последний узел
└── size: int      ← количество элементов

Детальная визуализация:

HEAD                                        TAIL
 │                                           │
 ▼                                           ▼
┌─────────┐      ┌─────────┐      ┌─────────┐
│ prev:None│     │ prev:───┼─────►│ prev:───┼─────►
│ data: A  │◄────┼─next:   │◄─────┼─next:   │
│ next:────┼────►│ data: B │      │ data: C │
└─────────┘      └─────────┘      │ next:None│
                                  └─────────┘
```

## Псевдокод основных операций

### Класс узла и списка

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.prev = None
        self.next = None

class DoublyLinkedList:
    def __init__(self):
        self.head = None
        self.tail = None
        self.size = 0
```

### Вставка в начало (Prepend)

```python
def prepend(self, data):
    """Добавление в начало - O(1)"""
    new_node = Node(data)

    if self.head is None:
        # Список пуст
        self.head = new_node
        self.tail = new_node
    else:
        new_node.next = self.head
        self.head.prev = new_node
        self.head = new_node

    self.size += 1
```

```
До:    None ◄── [A] ◄──► [B] ──► None
                 ↑
               head

После: None ◄── [X] ◄──► [A] ◄──► [B] ──► None
                 ↑
               head
```

### Вставка в конец (Append)

```python
def append(self, data):
    """Добавление в конец - O(1)"""
    new_node = Node(data)

    if self.tail is None:
        # Список пуст
        self.head = new_node
        self.tail = new_node
    else:
        new_node.prev = self.tail
        self.tail.next = new_node
        self.tail = new_node

    self.size += 1
```

```
До:    None ◄── [A] ◄──► [B] ──► None
                          ↑
                        tail

После: None ◄── [A] ◄──► [B] ◄──► [X] ──► None
                                   ↑
                                 tail
```

### Вставка по индексу

```python
def insert(self, index, data):
    """Вставка по индексу - O(n)"""
    if index < 0 or index > self.size:
        raise IndexError("Invalid index")

    if index == 0:
        return self.prepend(data)

    if index == self.size:
        return self.append(data)

    new_node = Node(data)

    # Оптимизация: выбираем направление обхода
    if index < self.size // 2:
        # Идём с начала
        current = self.head
        for _ in range(index):
            current = current.next
    else:
        # Идём с конца
        current = self.tail
        for _ in range(self.size - 1 - index):
            current = current.prev

    # Вставляем перед current
    new_node.prev = current.prev
    new_node.next = current
    current.prev.next = new_node
    current.prev = new_node

    self.size += 1
```

### Удаление узла (по ссылке)

```python
def remove_node(self, node):
    """Удаление узла по ссылке - O(1)"""
    if node is None:
        return

    if node.prev:
        node.prev.next = node.next
    else:
        self.head = node.next  # удаляем голову

    if node.next:
        node.next.prev = node.prev
    else:
        self.tail = node.prev  # удаляем хвост

    self.size -= 1
    return node.data
```

```
Удаление [B]:

До:    [A] ◄──► [B] ◄──► [C]
        │   ←──────────────┘
        └──────────────────►

После: [A] ◄──────────────► [C]
```

### Удаление из начала

```python
def remove_first(self):
    """Удаление первого элемента - O(1)"""
    if self.head is None:
        raise Exception("List is empty")

    removed = self.head.data

    if self.head == self.tail:
        # Один элемент
        self.head = None
        self.tail = None
    else:
        self.head = self.head.next
        self.head.prev = None

    self.size -= 1
    return removed
```

### Удаление из конца

```python
def remove_last(self):
    """Удаление последнего элемента - O(1)"""
    if self.tail is None:
        raise Exception("List is empty")

    removed = self.tail.data

    if self.head == self.tail:
        # Один элемент
        self.head = None
        self.tail = None
    else:
        self.tail = self.tail.prev
        self.tail.next = None

    self.size -= 1
    return removed
```

### Поиск

```python
def find(self, data):
    """Поиск узла по значению - O(n)"""
    current = self.head

    while current:
        if current.data == data:
            return current
        current = current.next

    return None

def get(self, index):
    """Получение элемента по индексу - O(n)"""
    if index < 0 or index >= self.size:
        raise IndexError("Invalid index")

    # Оптимизация: выбираем направление
    if index < self.size // 2:
        current = self.head
        for _ in range(index):
            current = current.next
    else:
        current = self.tail
        for _ in range(self.size - 1 - index):
            current = current.prev

    return current.data
```

### Обратный обход

```python
def traverse_backward(self):
    """Обход списка в обратном порядке"""
    result = []
    current = self.tail

    while current:
        result.append(current.data)
        current = current.prev

    return result
```

### Реверс списка

```python
def reverse(self):
    """Реверс списка in-place - O(n)"""
    current = self.head

    # Меняем местами head и tail
    self.head, self.tail = self.tail, self.head

    # Для каждого узла меняем prev и next
    while current:
        current.prev, current.next = current.next, current.prev
        current = current.prev  # идём по старому prev (новому next)
```

## Анализ сложности

| Операция | Время | Память |
|----------|-------|--------|
| Доступ по индексу | O(n)* | O(1) |
| Поиск | O(n) | O(1) |
| Вставка в начало | **O(1)** | O(1) |
| Вставка в конец | **O(1)** | O(1) |
| Вставка по индексу | O(n)* | O(1) |
| Удаление из начала | **O(1)** | O(1) |
| Удаление из конца | **O(1)** | O(1) |
| Удаление узла (по ссылке) | **O(1)** | O(1) |
| Удаление по значению | O(n) | O(1) |

*Оптимизировано: максимум n/2 шагов благодаря двустороннему обходу

### Память
- **На узел**: O(1) — данные + 2 указателя
- **На список**: O(n)
- **Накладные расходы**: 2 указателя на каждый узел (в 2 раза больше, чем в односвязном)

## Примеры с разбором

### Пример 1: Реализация Deque

```python
class Deque:
    """Двусторонняя очередь на основе двусвязного списка"""

    def __init__(self):
        self.list = DoublyLinkedList()

    def push_front(self, data):
        """Добавление в начало - O(1)"""
        self.list.prepend(data)

    def push_back(self, data):
        """Добавление в конец - O(1)"""
        self.list.append(data)

    def pop_front(self):
        """Удаление из начала - O(1)"""
        return self.list.remove_first()

    def pop_back(self):
        """Удаление из конца - O(1)"""
        return self.list.remove_last()

    def front(self):
        """Просмотр первого элемента - O(1)"""
        if self.list.head:
            return self.list.head.data
        raise Exception("Deque is empty")

    def back(self):
        """Просмотр последнего элемента - O(1)"""
        if self.list.tail:
            return self.list.tail.data
        raise Exception("Deque is empty")
```

### Пример 2: LRU Cache

```python
class LRUCache:
    """Least Recently Used Cache с O(1) операциями"""

    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = {}  # key -> node
        self.list = DoublyLinkedList()

    def get(self, key):
        """Получение значения - O(1)"""
        if key not in self.cache:
            return None

        node = self.cache[key]
        # Перемещаем в конец (most recently used)
        self._move_to_end(node)
        return node.data[1]  # возвращаем value

    def put(self, key, value):
        """Добавление/обновление значения - O(1)"""
        if key in self.cache:
            node = self.cache[key]
            node.data = (key, value)
            self._move_to_end(node)
        else:
            if len(self.cache) >= self.capacity:
                # Удаляем least recently used (голову)
                lru_node = self.list.head
                del self.cache[lru_node.data[0]]
                self.list.remove_node(lru_node)

            # Добавляем новый элемент в конец
            self.list.append((key, value))
            self.cache[key] = self.list.tail

    def _move_to_end(self, node):
        """Перемещение узла в конец списка - O(1)"""
        if node == self.list.tail:
            return

        # Удаляем из текущей позиции
        self.list.remove_node(node)

        # Добавляем в конец
        node.prev = self.list.tail
        node.next = None
        if self.list.tail:
            self.list.tail.next = node
        self.list.tail = node
        self.list.size += 1
```

```
LRU Cache с capacity=3:

Операции:
put(1, "a") → [1] tail (most recent)
put(2, "b") → [1] ◄─► [2] tail
put(3, "c") → [1] ◄─► [2] ◄─► [3] tail
get(1)      → [2] ◄─► [3] ◄─► [1] tail (1 moved to end)
put(4, "d") → [3] ◄─► [1] ◄─► [4] tail (2 evicted - LRU)
```

### Пример 3: Браузерная история

```python
class BrowserHistory:
    def __init__(self, homepage):
        self.current = Node(homepage)

    def visit(self, url):
        """Посещение новой страницы"""
        new_node = Node(url)
        new_node.prev = self.current
        self.current.next = new_node
        self.current = new_node
        # Forward history очищается

    def back(self, steps):
        """Переход назад на steps шагов"""
        while steps > 0 and self.current.prev:
            self.current = self.current.prev
            steps -= 1
        return self.current.data

    def forward(self, steps):
        """Переход вперёд на steps шагов"""
        while steps > 0 and self.current.next:
            self.current = self.current.next
            steps -= 1
        return self.current.data

# Использование:
history = BrowserHistory("google.com")
history.visit("facebook.com")
history.visit("youtube.com")
history.back(1)      # → "facebook.com"
history.back(1)      # → "google.com"
history.forward(1)   # → "facebook.com"
history.visit("twitter.com")  # forward history cleared
history.forward(1)   # → "twitter.com" (нет вперёд)
```

### Пример 4: Текстовый редактор с курсором

```python
class TextEditor:
    def __init__(self):
        self.list = DoublyLinkedList()
        self.cursor = None  # указывает на узел перед курсором

    def add_left(self, char):
        """Добавление символа слева от курсора"""
        if self.cursor is None:
            self.list.prepend(char)
            self.cursor = self.list.head
        else:
            new_node = Node(char)
            new_node.prev = self.cursor
            new_node.next = self.cursor.next
            if self.cursor.next:
                self.cursor.next.prev = new_node
            else:
                self.list.tail = new_node
            self.cursor.next = new_node
            self.cursor = new_node

    def delete_left(self):
        """Удаление символа слева от курсора (backspace)"""
        if self.cursor is None:
            return None
        deleted = self.cursor.data
        if self.cursor.prev:
            self.cursor.prev.next = self.cursor.next
        else:
            self.list.head = self.cursor.next
        if self.cursor.next:
            self.cursor.next.prev = self.cursor.prev
        else:
            self.list.tail = self.cursor.prev
        self.cursor = self.cursor.prev
        return deleted

    def move_left(self):
        """Перемещение курсора влево"""
        if self.cursor and self.cursor.prev:
            self.cursor = self.cursor.prev

    def move_right(self):
        """Перемещение курсора вправо"""
        if self.cursor is None:
            self.cursor = self.list.head
        elif self.cursor.next:
            self.cursor = self.cursor.next
```

## Типичные ошибки

### 1. Забытое обновление обоих указателей

```python
# НЕПРАВИЛЬНО
def insert_after(self, node, data):
    new_node = Node(data)
    new_node.next = node.next
    node.next = new_node
    # Забыли: new_node.prev = node
    # Забыли: if new_node.next: new_node.next.prev = new_node

# ПРАВИЛЬНО
def insert_after(self, node, data):
    new_node = Node(data)
    new_node.prev = node
    new_node.next = node.next
    if node.next:
        node.next.prev = new_node
    node.next = new_node
```

### 2. Неправильный порядок операций

```python
# НЕПРАВИЛЬНО
node.prev.next = node.next
node.next.prev = node.prev  # NullPointerException если node.next = None!

# ПРАВИЛЬНО
if node.prev:
    node.prev.next = node.next
if node.next:
    node.next.prev = node.prev
```

### 3. Забытое обновление head/tail

```python
# НЕПРАВИЛЬНО
def remove_first(self):
    removed = self.head
    self.head = self.head.next
    # Забыли: self.head.prev = None
    # Забыли: если список стал пустым, обновить tail

# ПРАВИЛЬНО
def remove_first(self):
    removed = self.head
    self.head = self.head.next
    if self.head:
        self.head.prev = None
    else:
        self.tail = None
    return removed.data
```

### 4. Потеря связности при реверсе

```python
# НЕПРАВИЛЬНО (теряем узлы)
def reverse(self):
    current = self.head
    while current:
        current.prev, current.next = current.next, current.prev
        current = current.next  # теперь next - это старый prev!

# ПРАВИЛЬНО
def reverse(self):
    current = self.head
    self.head, self.tail = self.tail, self.head
    while current:
        current.prev, current.next = current.next, current.prev
        current = current.prev  # идём по новому prev (старому next)
```

## Сравнение структур

| Операция | Массив | Односвязный | Двусвязный |
|----------|--------|-------------|------------|
| Доступ по индексу | **O(1)** | O(n) | O(n)* |
| Вставка в начало | O(n) | **O(1)** | **O(1)** |
| Вставка в конец | O(1)** | O(1)*** | **O(1)** |
| Удаление из начала | O(n) | **O(1)** | **O(1)** |
| Удаление из конца | O(1) | O(n) | **O(1)** |
| Удаление узла | O(n) | O(n) | **O(1)** |
| Обратный обход | O(n) | O(n)**** | **O(n)** |
| Память на элемент | 1 слот | +1 указатель | +2 указателя |

*Оптимизировано до O(n/2)
**Амортизированное
***С указателем на tail
****Требует предварительного реверса или O(n) памяти
