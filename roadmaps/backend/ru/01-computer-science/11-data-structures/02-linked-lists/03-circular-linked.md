# Кольцевой (циклический) связный список (Circular Linked List)

[prev: 02-doubly-linked](./02-doubly-linked.md) | [next: 01-stack-concept](../03-stacks/01-stack-concept.md)

---
## Определение

**Кольцевой связный список** — это разновидность связного списка, в которой последний узел указывает не на `null`, а обратно на первый узел, образуя замкнутый цикл. Может быть односвязным или двусвязным.

### Односвязный кольцевой список

```
       ┌───────────────────────────────┐
       │                               │
       ▼                               │
    ┌─────┐    ┌─────┐    ┌─────┐    ┌─┴───┐
    │  A  │───►│  B  │───►│  C  │───►│  D  │
    └─────┘    └─────┘    └─────┘    └─────┘
       ↑
     HEAD
```

### Двусвязный кольцевой список

```
       ┌─────────────────────────────────────────┐
       │                                         │
       ▼                                         │
    ┌─────┐◄──►┌─────┐◄──►┌─────┐◄──►┌─────┐    │
    │  A  │    │  B  │    │  C  │    │  D  ├────┘
    └──┬──┘    └─────┘    └─────┘    └──▲──┘
       │                               │
       └───────────────────────────────┘
```

## Зачем нужно

### Преимущества:
- **Непрерывный обход** — можно бесконечно итерировать по списку
- **Равноправие узлов** — нет "особого" конца списка
- **Эффективный доступ к концу** — если хранить tail, head = tail.next

### Практическое применение:
- **Round-robin планировщик** — распределение ресурсов по кругу
- **Карусель изображений** — бесконечная прокрутка
- **Буфер аудио/видео** — циклическая запись
- **Игры** — очерёдность ходов игроков
- **Многопользовательские системы** — распределение задач между серверами
- **Алгоритм Джозефуса** — классическая задача

## Как работает

### Различие с обычным списком

```
Обычный список:
HEAD → [A] → [B] → [C] → None
                         ↑
                    конец списка

Кольцевой список:
HEAD → [A] → [B] → [C] ─┐
       ↑                │
       └────────────────┘
       нет конца, только начало (HEAD)
```

### Хранение tail вместо head

Удобная альтернатива — хранить указатель на tail:

```
TAIL → [C] ─┐
       ↑    │
       │    ▼
 [B] ◄─┘  [A] = TAIL.next (HEAD)
  ↑        │
  └────────┘

Преимущества:
- head = tail.next — O(1)
- Вставка в начало — O(1)
- Вставка в конец — O(1)
```

## Псевдокод основных операций

### Класс узла и списка

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.next = None

class CircularLinkedList:
    def __init__(self):
        self.tail = None  # храним tail для эффективности
        self.size = 0

    def is_empty(self):
        return self.tail is None

    @property
    def head(self):
        return self.tail.next if self.tail else None
```

### Вставка в пустой список

```python
def _insert_first(self, data):
    """Вставка первого элемента"""
    new_node = Node(data)
    new_node.next = new_node  # указывает на себя
    self.tail = new_node
    self.size = 1
```

```
До:    TAIL = None (пустой список)

После: TAIL ─┐
       ↑     │
       │     ▼
      [A] ◄──┘
       ↑
       └── указывает на себя
```

### Вставка в начало (Prepend)

```python
def prepend(self, data):
    """Добавление в начало - O(1)"""
    if self.is_empty():
        return self._insert_first(data)

    new_node = Node(data)
    new_node.next = self.tail.next  # новый узел → старый head
    self.tail.next = new_node       # tail → новый узел (новый head)
    self.size += 1
```

```
До:    TAIL → [C] → [A] → [B] ─┐
                    ↑          │
                    └──────────┘

После: TAIL → [C] → [X] → [A] → [B] ─┐
                    ↑                │
                    └────────────────┘
              (X - новый head)
```

### Вставка в конец (Append)

```python
def append(self, data):
    """Добавление в конец - O(1)"""
    if self.is_empty():
        return self._insert_first(data)

    new_node = Node(data)
    new_node.next = self.tail.next  # новый узел → head
    self.tail.next = new_node       # старый tail → новый узел
    self.tail = new_node            # обновляем tail
    self.size += 1
```

```
До:    TAIL → [C] → [A] → [B] ─┐
        ↑     ↑               │
        │     head            │
        └─────────────────────┘

После: TAIL → [X] → [A] → [B] → [C] ─┐
        ↑     ↑                      │
        │    head                    │
        └────────────────────────────┘
       (X - новый tail)
```

### Удаление из начала

```python
def remove_first(self):
    """Удаление первого элемента - O(1)"""
    if self.is_empty():
        raise Exception("List is empty")

    head = self.tail.next
    removed = head.data

    if self.tail == head:  # один элемент
        self.tail = None
    else:
        self.tail.next = head.next

    self.size -= 1
    return removed
```

### Удаление из конца

```python
def remove_last(self):
    """Удаление последнего элемента - O(n)"""
    if self.is_empty():
        raise Exception("List is empty")

    if self.tail.next == self.tail:  # один элемент
        removed = self.tail.data
        self.tail = None
        self.size -= 1
        return removed

    # Находим предпоследний узел
    current = self.tail.next
    while current.next != self.tail:
        current = current.next

    removed = self.tail.data
    current.next = self.tail.next  # предпоследний → head
    self.tail = current            # обновляем tail
    self.size -= 1
    return removed
```

### Поиск

```python
def find(self, data):
    """Поиск элемента - O(n)"""
    if self.is_empty():
        return None

    current = self.tail.next  # начинаем с head

    # Обходим пока не вернёмся к head
    while True:
        if current.data == data:
            return current
        current = current.next
        if current == self.tail.next:  # вернулись к началу
            break

    return None
```

### Обход списка

```python
def traverse(self):
    """Обход всех элементов - O(n)"""
    if self.is_empty():
        return []

    result = []
    current = self.tail.next  # начинаем с head

    while True:
        result.append(current.data)
        current = current.next
        if current == self.tail.next:  # вернулись к началу
            break

    return result

def __iter__(self):
    """Итератор для for-цикла"""
    if self.is_empty():
        return

    current = self.tail.next
    while True:
        yield current.data
        current = current.next
        if current == self.tail.next:
            break
```

### Бесконечный итератор

```python
def infinite_iterator(self):
    """Бесконечный итератор для round-robin"""
    if self.is_empty():
        return

    current = self.tail.next
    while True:
        yield current.data
        current = current.next
```

## Анализ сложности

| Операция | Время | Примечание |
|----------|-------|------------|
| Вставка в начало | **O(1)** | Благодаря хранению tail |
| Вставка в конец | **O(1)** | Благодаря хранению tail |
| Удаление из начала | **O(1)** | |
| Удаление из конца | O(n) | Нужно найти предпоследний |
| Поиск | O(n) | |
| Доступ по индексу | O(n) | |
| Обход | O(n) | |

### Память
- **На узел**: O(1) — данные + указатель
- **На список**: O(n)
- Такая же, как у обычного односвязного списка

## Примеры с разбором

### Пример 1: Round-Robin планировщик

```python
class RoundRobinScheduler:
    """Планировщик задач по круговому принципу"""

    def __init__(self, time_quantum):
        self.tasks = CircularLinkedList()
        self.time_quantum = time_quantum
        self.current = None

    def add_task(self, task):
        """Добавление задачи"""
        self.tasks.append(task)
        if self.current is None:
            self.current = self.tasks.head

    def remove_task(self, task_id):
        """Удаление завершённой задачи"""
        # ... реализация удаления

    def next_task(self):
        """Получение следующей задачи"""
        if self.tasks.is_empty():
            return None

        task = self.current.data
        self.current = self.current.next
        return task

    def run(self):
        """Основной цикл планировщика"""
        while not self.tasks.is_empty():
            task = self.next_task()
            execute(task, self.time_quantum)

            if task.is_completed():
                self.remove_task(task.id)

# Визуализация:
# Tasks: [T1] → [T2] → [T3] → [T1] → ...
# Выполнение: T1(10ms) → T2(10ms) → T3(10ms) → T1(10ms) → ...
```

### Пример 2: Задача Джозефуса

```python
def josephus(n, k):
    """
    Задача Джозефуса: n человек стоят в круге, каждый k-й выбывает.
    Найти позицию последнего выжившего.

    n = 7, k = 3:
    [1, 2, 3, 4, 5, 6, 7] → удаляем 3
    [1, 2, 4, 5, 6, 7] → удаляем 6
    [1, 2, 4, 5, 7] → удаляем 2
    [1, 4, 5, 7] → удаляем 7
    [1, 4, 5] → удаляем 5
    [1, 4] → удаляем 1
    [4] → выжил!
    """
    # Создаём кольцевой список
    circle = CircularLinkedList()
    for i in range(1, n + 1):
        circle.append(i)

    current = circle.head
    prev = circle.tail

    while circle.size > 1:
        # Отсчитываем k-1 шагов
        for _ in range(k - 1):
            prev = current
            current = current.next

        # Удаляем k-го человека
        print(f"Удаляем: {current.data}")
        prev.next = current.next
        if current == circle.tail:
            circle.tail = prev

        current = prev.next
        circle.size -= 1

    return circle.tail.data

# josephus(7, 3) → 4
```

### Пример 3: Карусель изображений

```python
class ImageCarousel:
    """Карусель изображений с бесконечной прокруткой"""

    def __init__(self):
        self.images = CircularLinkedList()
        self.current = None

    def add_image(self, image_url):
        self.images.append(image_url)
        if self.current is None:
            self.current = self.images.head

    def next(self):
        """Следующее изображение"""
        if self.current:
            self.current = self.current.next
            return self.current.data
        return None

    def prev(self):
        """Предыдущее изображение (для двусвязного варианта)"""
        # Для односвязного - нужен полный обход O(n)
        if self.images.is_empty():
            return None

        target = self.current
        temp = self.current.next
        while temp.next != target:
            temp = temp.next

        self.current = temp
        return self.current.data

    def current_image(self):
        return self.current.data if self.current else None

# Использование:
carousel = ImageCarousel()
carousel.add_image("img1.jpg")
carousel.add_image("img2.jpg")
carousel.add_image("img3.jpg")

# Бесконечная прокрутка:
# img1 → img2 → img3 → img1 → img2 → ...
```

### Пример 4: Кольцевой буфер

```python
class CircularBuffer:
    """
    Кольцевой буфер фиксированного размера.
    Когда буфер полон, новые элементы перезаписывают старые.
    """

    def __init__(self, capacity):
        self.capacity = capacity
        self.buffer = [None] * capacity
        self.head = 0  # индекс для чтения
        self.tail = 0  # индекс для записи
        self.size = 0

    def write(self, data):
        """Запись данных"""
        self.buffer[self.tail] = data
        self.tail = (self.tail + 1) % self.capacity

        if self.size < self.capacity:
            self.size += 1
        else:
            # Буфер полон, сдвигаем head (теряем старые данные)
            self.head = (self.head + 1) % self.capacity

    def read(self):
        """Чтение данных"""
        if self.size == 0:
            raise Exception("Buffer is empty")

        data = self.buffer[self.head]
        self.head = (self.head + 1) % self.capacity
        self.size -= 1
        return data

    def is_full(self):
        return self.size == self.capacity

    def is_empty(self):
        return self.size == 0

# Визуализация (capacity = 4):
#
# После write(A), write(B), write(C):
# [A, B, C, _]
#  ↑        ↑
# head    tail
#
# После write(D), write(E):
# [E, B, C, D]  ← A перезаписан на E
#     ↑     ↑
#   head  tail
```

## Двусвязный кольцевой список

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.prev = None
        self.next = None

class DoublyCircularLinkedList:
    def __init__(self):
        self.head = None
        self.size = 0

    def append(self, data):
        new_node = Node(data)

        if self.head is None:
            new_node.next = new_node
            new_node.prev = new_node
            self.head = new_node
        else:
            tail = self.head.prev  # последний элемент
            new_node.next = self.head
            new_node.prev = tail
            tail.next = new_node
            self.head.prev = new_node

        self.size += 1

    def move_forward(self, current, steps):
        """Перемещение вперёд на steps шагов"""
        for _ in range(steps):
            current = current.next
        return current

    def move_backward(self, current, steps):
        """Перемещение назад на steps шагов"""
        for _ in range(steps):
            current = current.prev
        return current
```

## Типичные ошибки

### 1. Бесконечный цикл при обходе

```python
# НЕПРАВИЛЬНО
current = self.head
while current is not None:  # никогда не станет None!
    print(current.data)
    current = current.next

# ПРАВИЛЬНО
current = self.head
while True:
    print(current.data)
    current = current.next
    if current == self.head:  # вернулись к началу
        break
```

### 2. Забытый случай одного элемента

```python
# НЕПРАВИЛЬНО
def remove_first(self):
    head = self.tail.next
    self.tail.next = head.next
    # Если был один элемент, tail.next теперь указывает на удалённый узел!

# ПРАВИЛЬНО
def remove_first(self):
    head = self.tail.next
    if head == self.tail:  # один элемент
        self.tail = None
    else:
        self.tail.next = head.next
```

### 3. Неправильная инициализация первого элемента

```python
# НЕПРАВИЛЬНО
def append(self, data):
    new_node = Node(data)
    if self.tail is None:
        self.tail = new_node
        # new_node.next остаётся None — список не кольцевой!

# ПРАВИЛЬНО
def append(self, data):
    new_node = Node(data)
    if self.tail is None:
        new_node.next = new_node  # указывает на себя!
        self.tail = new_node
```

### 4. Потеря узлов при вставке

```python
# НЕПРАВИЛЬНО
def prepend(self, data):
    new_node = Node(data)
    self.tail.next = new_node  # потеряли старый head!
    new_node.next = self.tail.next  # теперь указывает на себя!

# ПРАВИЛЬНО
def prepend(self, data):
    new_node = Node(data)
    new_node.next = self.tail.next  # сначала связываем новый узел
    self.tail.next = new_node       # потом обновляем tail.next
```

## Сравнение с обычным связным списком

| Аспект | Обычный список | Кольцевой список |
|--------|----------------|------------------|
| Конец списка | `None` | Ссылка на head |
| Обход | Остановка на `None` | Остановка на head |
| Бесконечный обход | Невозможен | Естественный |
| Хранение | head (+ tail) | tail (head = tail.next) |
| Вставка в конец | O(1) с tail | O(1) с tail |
| Применение | Общего назначения | Round-robin, карусели |

---

[prev: 02-doubly-linked](./02-doubly-linked.md) | [next: 01-stack-concept](../03-stacks/01-stack-concept.md)
