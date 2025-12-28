# Односвязный список (Singly Linked List)

[prev: 03-array-operations](../01-arrays/03-array-operations.md) | [next: 02-doubly-linked](./02-doubly-linked.md)

---
## Определение

**Односвязный список** — это линейная структура данных, состоящая из узлов (nodes), где каждый узел содержит данные и ссылку (указатель) на следующий узел. Последний узел указывает на `null` (или `None`), обозначая конец списка.

```
┌────────────┐    ┌────────────┐    ┌────────────┐    ┌────────────┐
│ data: 10   │    │ data: 20   │    │ data: 30   │    │ data: 40   │
│ next: ─────┼───►│ next: ─────┼───►│ next: ─────┼───►│ next: None │
└────────────┘    └────────────┘    └────────────┘    └────────────┘
     HEAD                                                  TAIL
```

## Зачем нужно

### Преимущества перед массивами:
- **Динамический размер** — не нужно заранее знать количество элементов
- **Эффективная вставка/удаление** — O(1) при наличии указателя на предыдущий узел
- **Нет необходимости в непрерывной памяти** — узлы могут быть разбросаны по памяти

### Практическое применение:
- **Реализация стеков и очередей** — основа для других структур
- **Undo/Redo** — история операций
- **Музыкальный плейлист** — последовательный список песен
- **Цепочки хэш-таблиц** — разрешение коллизий методом цепочек
- **Полиномиальная арифметика** — представление многочленов

## Как работает

### Структура узла

```
┌─────────────────┐
│      Node       │
├─────────────────┤
│  data: любой    │  ← хранимое значение
│  тип данных     │
├─────────────────┤
│  next: Node*    │  ← указатель на следующий узел
└─────────────────┘
```

### Структура списка

```
LinkedList
├── head: Node*    ← указатель на первый узел
├── tail: Node*    ← (опционально) указатель на последний узел
└── size: int      ← (опционально) количество элементов

Пример: список [A, B, C]

head                                      tail
 │                                         │
 ▼                                         ▼
┌─────┐    ┌─────┐    ┌─────┐
│  A  │───►│  B  │───►│  C  │───► None
└─────┘    └─────┘    └─────┘
```

### Обход списка

```
Поиск значения "C":

head
 │
 ▼
[A] → [B] → [C] → None
 ↓     ↓     ↓
"A"≠"C" → "B"≠"C" → "C"="C" ✓ Найдено!
```

## Псевдокод основных операций

### Класс узла и списка

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.next = None

class SinglyLinkedList:
    def __init__(self):
        self.head = None
        self.tail = None
        self.size = 0
```

### Вставка в начало (Prepend)

```python
def prepend(self, data):
    """Добавление в начало списка - O(1)"""
    new_node = Node(data)
    new_node.next = self.head
    self.head = new_node

    if self.tail is None:  # список был пуст
        self.tail = new_node

    self.size += 1
```

```
До:    head → [A] → [B] → None
После: head → [X] → [A] → [B] → None
              ↑
           new_node
```

### Вставка в конец (Append)

```python
def append(self, data):
    """Добавление в конец списка - O(1) с tail, O(n) без tail"""
    new_node = Node(data)

    if self.head is None:  # список пуст
        self.head = new_node
        self.tail = new_node
    else:
        self.tail.next = new_node
        self.tail = new_node

    self.size += 1
```

```
До:    head → [A] → [B] → None
                     ↑
                   tail
После: head → [A] → [B] → [X] → None
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
    current = self.head

    # Идём до узла перед позицией вставки
    for _ in range(index - 1):
        current = current.next

    new_node.next = current.next
    current.next = new_node
    self.size += 1
```

```
Вставка X на позицию 1:

До:    [A] → [B] → [C] → None
        ↑     ↑
     current next

После: [A] → [X] → [B] → [C] → None
```

### Удаление из начала

```python
def remove_first(self):
    """Удаление первого элемента - O(1)"""
    if self.head is None:
        raise Exception("List is empty")

    removed = self.head.data
    self.head = self.head.next

    if self.head is None:  # список стал пустым
        self.tail = None

    self.size -= 1
    return removed
```

### Удаление из конца

```python
def remove_last(self):
    """Удаление последнего элемента - O(n)"""
    if self.head is None:
        raise Exception("List is empty")

    if self.head == self.tail:  # один элемент
        removed = self.head.data
        self.head = None
        self.tail = None
        self.size -= 1
        return removed

    # Находим предпоследний узел
    current = self.head
    while current.next != self.tail:
        current = current.next

    removed = self.tail.data
    current.next = None
    self.tail = current
    self.size -= 1
    return removed
```

### Удаление по значению

```python
def remove(self, data):
    """Удаление первого вхождения значения - O(n)"""
    if self.head is None:
        raise Exception("List is empty")

    # Удаление из головы
    if self.head.data == data:
        return self.remove_first()

    # Поиск узла перед удаляемым
    current = self.head
    while current.next is not None:
        if current.next.data == data:
            if current.next == self.tail:
                self.tail = current
            current.next = current.next.next
            self.size -= 1
            return data
        current = current.next

    raise ValueError("Value not found")
```

### Поиск

```python
def find(self, data):
    """Поиск индекса элемента - O(n)"""
    current = self.head
    index = 0

    while current is not None:
        if current.data == data:
            return index
        current = current.next
        index += 1

    return -1  # не найден

def contains(self, data):
    """Проверка наличия элемента - O(n)"""
    return self.find(data) != -1
```

### Доступ по индексу

```python
def get(self, index):
    """Получение элемента по индексу - O(n)"""
    if index < 0 or index >= self.size:
        raise IndexError("Invalid index")

    current = self.head
    for _ in range(index):
        current = current.next

    return current.data
```

### Реверс списка

```python
def reverse(self):
    """Реверс списка in-place - O(n)"""
    self.tail = self.head
    prev = None
    current = self.head

    while current is not None:
        next_node = current.next  # сохраняем следующий
        current.next = prev       # переворачиваем указатель
        prev = current            # сдвигаем prev
        current = next_node       # сдвигаем current

    self.head = prev
```

```
Шаг за шагом:

Initial: None ← [A] → [B] → [C] → None
                prev  curr  next

Step 1:  None ← [A]   [B] → [C] → None
                prev  curr  next

Step 2:  None ← [A] ← [B]   [C] → None
                      prev  curr

Step 3:  None ← [A] ← [B] ← [C]
                            prev = new head
```

## Анализ сложности

| Операция | Время | Память |
|----------|-------|--------|
| Доступ по индексу | O(n) | O(1) |
| Поиск | O(n) | O(1) |
| Вставка в начало | **O(1)** | O(1) |
| Вставка в конец (с tail) | **O(1)** | O(1) |
| Вставка в конец (без tail) | O(n) | O(1) |
| Вставка по индексу | O(n) | O(1) |
| Удаление из начала | **O(1)** | O(1) |
| Удаление из конца | O(n) | O(1) |
| Удаление по значению | O(n) | O(1) |

### Память
- **На узел**: O(1) — данные + указатель
- **На список**: O(n) — n узлов
- **Накладные расходы**: каждый узел хранит дополнительный указатель

## Примеры с разбором

### Пример 1: Простой список

```python
# Создание списка [1, 2, 3]
lst = SinglyLinkedList()
lst.append(1)
lst.append(2)
lst.append(3)

# Обход и печать
current = lst.head
while current:
    print(current.data, end=" → ")
    current = current.next
# Вывод: 1 → 2 → 3 →
```

### Пример 2: Определение цикла (алгоритм Флойда)

```python
def has_cycle(head):
    """Определение наличия цикла в списке"""
    if head is None:
        return False

    slow = head       # медленный указатель
    fast = head       # быстрый указатель

    while fast is not None and fast.next is not None:
        slow = slow.next       # шаг 1
        fast = fast.next.next  # шаг 2

        if slow == fast:       # встретились = есть цикл
            return True

    return False

# Визуализация:
# [1] → [2] → [3] → [4]
#              ↑      ↓
#              └──────┘
#
# slow: 1 → 2 → 3 → 4 → 3 → 4...
# fast: 1 → 3 → 3 → 3 (встреча!)
```

### Пример 3: Нахождение середины списка

```python
def find_middle(head):
    """Нахождение среднего элемента за один проход"""
    slow = head
    fast = head

    while fast is not None and fast.next is not None:
        slow = slow.next
        fast = fast.next.next

    return slow.data

# [1] → [2] → [3] → [4] → [5]
# slow: 1 → 2 → 3 (стоп)
# fast: 1 → 3 → 5 → None
# Середина = 3
```

### Пример 4: Слияние двух отсортированных списков

```python
def merge_sorted_lists(l1, l2):
    """Слияние двух отсортированных списков"""
    dummy = Node(0)  # фиктивный узел
    current = dummy

    while l1 is not None and l2 is not None:
        if l1.data <= l2.data:
            current.next = l1
            l1 = l1.next
        else:
            current.next = l2
            l2 = l2.next
        current = current.next

    # Присоединяем оставшийся список
    current.next = l1 if l1 else l2

    return dummy.next

# l1: [1] → [3] → [5]
# l2: [2] → [4] → [6]
# Результат: [1] → [2] → [3] → [4] → [5] → [6]
```

## Типичные ошибки

### 1. Потеря указателя на голову

```python
# НЕПРАВИЛЬНО
def print_list(head):
    while head:
        print(head.data)
        head = head.next  # теряем исходный head!

# После вызова head указывает на None

# ПРАВИЛЬНО
def print_list(head):
    current = head  # используем временную переменную
    while current:
        print(current.data)
        current = current.next
```

### 2. Забытое обновление tail

```python
# НЕПРАВИЛЬНО
def remove_last(self):
    current = self.head
    while current.next.next:
        current = current.next
    current.next = None
    # tail всё ещё указывает на удалённый узел!

# ПРАВИЛЬНО
def remove_last(self):
    current = self.head
    while current.next != self.tail:
        current = current.next
    current.next = None
    self.tail = current  # обновляем tail
```

### 3. NullPointerException при пустом списке

```python
# НЕПРАВИЛЬНО
def get_first(self):
    return self.head.data  # NullPointerException если head = None

# ПРАВИЛЬНО
def get_first(self):
    if self.head is None:
        raise Exception("List is empty")
    return self.head.data
```

### 4. Неправильный порядок операций при вставке

```python
# НЕПРАВИЛЬНО
current.next = new_node
new_node.next = current.next  # new_node.next = new_node !

# ПРАВИЛЬНО
new_node.next = current.next  # сначала связываем новый узел
current.next = new_node       # потом переключаем указатель
```

### 5. Утечка памяти (в языках без GC)

```c
// В C/C++ нужно освобождать память вручную
void remove_first(List* list) {
    Node* temp = list->head;
    list->head = list->head->next;
    free(temp);  // не забыть освободить!
}
```

## Сравнение с массивом

| Операция | Массив | Связный список |
|----------|--------|----------------|
| Доступ по индексу | **O(1)** | O(n) |
| Поиск | O(n) | O(n) |
| Вставка в начало | O(n) | **O(1)** |
| Вставка в конец | O(1)* | O(1)** |
| Удаление из начала | O(n) | **O(1)** |
| Удаление из конца | O(1) | O(n) |
| Память | Компактная | Накладные расходы |

*Амортизированное для динамического массива
**С указателем на tail

---

[prev: 03-array-operations](../01-arrays/03-array-operations.md) | [next: 02-doubly-linked](./02-doubly-linked.md)
