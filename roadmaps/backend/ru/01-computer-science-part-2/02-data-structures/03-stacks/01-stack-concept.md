# Стек — концепция (Stack Concept)

## Определение

**Стек** — это линейная структура данных, работающая по принципу LIFO (Last In, First Out — «последним пришёл, первым вышел»). Элементы добавляются и удаляются только с одного конца, называемого **вершиной** (top) стека.

```
         ┌─────────┐
    top→ │    D    │  ← последний добавленный (первый на выход)
         ├─────────┤
         │    C    │
         ├─────────┤
         │    B    │
         ├─────────┤
         │    A    │  ← первый добавленный (последний на выход)
         └─────────┘
```

## Аналогия

Представьте стопку тарелок:
- Новую тарелку кладут **сверху** стопки
- Берут тарелку тоже **сверху**
- Чтобы достать нижнюю тарелку, нужно сначала убрать все верхние

## Зачем нужно

### Практическое применение:
- **Вызов функций** — Call Stack в программах
- **Undo/Redo** — отмена действий в редакторах
- **Скобочные последовательности** — проверка баланса скобок
- **Обход деревьев и графов** — DFS (Depth-First Search)
- **Вычисление выражений** — калькуляторы, парсеры
- **Браузерная история** — кнопка "назад"
- **Рекурсия** — неявно использует системный стек

## Как работает

### Основные операции

```
PUSH (добавление):                POP (удаление):

    ┌───┐                             ┌───┐
    │ E │ → push                 E ← │ E │ ← pop
    └───┘                             └───┘
    ┌───┐                             ┌───┐
    │ D │     top                     │ D │     top (новый)
    ├───┤                             ├───┤
    │ C │                             │ C │
    ├───┤                             ├───┤
    │ B │                             │ B │
    └───┘                             └───┘

До: [B, C, D]                    После push(E): [B, C, D, E]
После pop(): [B, C, D]           Возвращается: E
```

### Дополнительные операции

```
PEEK (просмотр вершины без удаления):

    ┌───┐
    │ D │ ← peek() возвращает D, но не удаляет
    ├───┤
    │ C │
    ├───┤
    │ B │
    └───┘

IS_EMPTY (проверка пустоты):
    Пустой стек: []  → is_empty() = True
    Не пустой:   [A] → is_empty() = False

SIZE (размер):
    [A, B, C] → size() = 3
```

## Псевдокод основных операций

### Реализация на основе массива

```python
class ArrayStack:
    def __init__(self, capacity=10):
        self.capacity = capacity
        self.data = [None] * capacity
        self.top = -1  # индекс вершины (-1 = стек пуст)

    def is_empty(self):
        """Проверка пустоты - O(1)"""
        return self.top == -1

    def is_full(self):
        """Проверка заполненности - O(1)"""
        return self.top == self.capacity - 1

    def size(self):
        """Размер стека - O(1)"""
        return self.top + 1

    def push(self, element):
        """Добавление элемента - O(1)"""
        if self.is_full():
            raise OverflowError("Stack is full")

        self.top += 1
        self.data[self.top] = element

    def pop(self):
        """Удаление и возврат элемента - O(1)"""
        if self.is_empty():
            raise IndexError("Stack is empty")

        element = self.data[self.top]
        self.data[self.top] = None  # очистка (опционально)
        self.top -= 1
        return element

    def peek(self):
        """Просмотр вершины - O(1)"""
        if self.is_empty():
            raise IndexError("Stack is empty")

        return self.data[self.top]
```

### Реализация на основе динамического массива (Python list)

```python
class DynamicStack:
    def __init__(self):
        self.data = []

    def is_empty(self):
        return len(self.data) == 0

    def size(self):
        return len(self.data)

    def push(self, element):
        """O(1) амортизированное"""
        self.data.append(element)

    def pop(self):
        """O(1)"""
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self.data.pop()

    def peek(self):
        """O(1)"""
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self.data[-1]
```

### Реализация на основе связного списка

```python
class Node:
    def __init__(self, data):
        self.data = data
        self.next = None

class LinkedStack:
    def __init__(self):
        self.top = None  # вершина = голова списка
        self._size = 0

    def is_empty(self):
        return self.top is None

    def size(self):
        return self._size

    def push(self, element):
        """Добавление в начало списка - O(1)"""
        new_node = Node(element)
        new_node.next = self.top
        self.top = new_node
        self._size += 1

    def pop(self):
        """Удаление из начала списка - O(1)"""
        if self.is_empty():
            raise IndexError("Stack is empty")

        element = self.top.data
        self.top = self.top.next
        self._size -= 1
        return element

    def peek(self):
        """O(1)"""
        if self.is_empty():
            raise IndexError("Stack is empty")
        return self.top.data
```

```
Связный список как стек:

push(A): top → [A] → None
push(B): top → [B] → [A] → None
push(C): top → [C] → [B] → [A] → None
pop():   top → [B] → [A] → None  (возвращает C)
```

## Анализ сложности

| Операция | Массив | Динамический массив | Связный список |
|----------|--------|---------------------|----------------|
| push | O(1) | O(1)* | O(1) |
| pop | O(1) | O(1) | O(1) |
| peek | O(1) | O(1) | O(1) |
| is_empty | O(1) | O(1) | O(1) |
| size | O(1) | O(1) | O(1) |

*Амортизированное время

### Память
- **Массив**: O(capacity) — фиксированный размер
- **Динамический массив**: O(n) — с накладными расходами на расширение
- **Связный список**: O(n) — дополнительная память на указатели

## Примеры с разбором

### Пример 1: Проверка баланса скобок

```python
def is_balanced(expression):
    """
    Проверяет, сбалансированы ли скобки в выражении.

    "(a + b) * (c + d)" → True
    "((a + b)" → False (нет закрывающей)
    "(a + b))" → False (лишняя закрывающая)
    "([a + b])" → True
    "([a + b)]" → False (неправильный порядок)
    """
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}

    for char in expression:
        if char in '([{':
            stack.append(char)
        elif char in ')]}':
            if not stack:
                return False  # нет открывающей скобки
            if stack.pop() != pairs[char]:
                return False  # скобки не совпадают

    return len(stack) == 0  # все скобки закрыты

# Трассировка для "([a + b])":
# '(' → stack: ['(']
# '[' → stack: ['(', '[']
# ']' → pop '[', совпадает → stack: ['(']
# ')' → pop '(', совпадает → stack: []
# Результат: True
```

### Пример 2: Реверс строки

```python
def reverse_string(s):
    """Реверс строки с помощью стека"""
    stack = []

    # Помещаем все символы в стек
    for char in s:
        stack.append(char)

    # Извлекаем в обратном порядке
    result = ""
    while stack:
        result += stack.pop()

    return result

# reverse_string("hello") → "olleh"

# Шаги:
# Push: h, e, l, l, o → stack: ['h', 'e', 'l', 'l', 'o']
# Pop: o, l, l, e, h → result: "olleh"
```

### Пример 3: Вычисление постфиксного выражения

```python
def evaluate_postfix(expression):
    """
    Вычисляет выражение в постфиксной (обратной польской) нотации.

    "3 4 +" = 3 + 4 = 7
    "3 4 + 2 *" = (3 + 4) * 2 = 14
    "5 1 2 + 4 * + 3 -" = 5 + ((1 + 2) * 4) - 3 = 14
    """
    stack = []
    operators = {'+', '-', '*', '/'}

    for token in expression.split():
        if token in operators:
            b = stack.pop()  # второй операнд
            a = stack.pop()  # первый операнд

            if token == '+':
                result = a + b
            elif token == '-':
                result = a - b
            elif token == '*':
                result = a * b
            else:  # '/'
                result = a / b

            stack.append(result)
        else:
            stack.append(float(token))

    return stack.pop()

# Трассировка "3 4 + 2 *":
# '3' → stack: [3]
# '4' → stack: [3, 4]
# '+' → pop 4, pop 3, push 3+4=7 → stack: [7]
# '2' → stack: [7, 2]
# '*' → pop 2, pop 7, push 7*2=14 → stack: [14]
# Результат: 14
```

### Пример 4: История браузера (простая версия)

```python
class BrowserHistory:
    def __init__(self, homepage):
        self.back_stack = []
        self.forward_stack = []
        self.current = homepage

    def visit(self, url):
        """Переход на новую страницу"""
        self.back_stack.append(self.current)
        self.current = url
        self.forward_stack.clear()  # очищаем forward историю

    def back(self):
        """Кнопка 'Назад'"""
        if self.back_stack:
            self.forward_stack.append(self.current)
            self.current = self.back_stack.pop()
        return self.current

    def forward(self):
        """Кнопка 'Вперёд'"""
        if self.forward_stack:
            self.back_stack.append(self.current)
            self.current = self.forward_stack.pop()
        return self.current

# Использование:
browser = BrowserHistory("google.com")
browser.visit("youtube.com")  # back: [google], current: youtube
browser.visit("facebook.com")  # back: [google, youtube], current: facebook
browser.back()  # → youtube, forward: [facebook]
browser.back()  # → google, forward: [facebook, youtube]
browser.forward()  # → youtube, forward: [facebook]
browser.visit("twitter.com")  # forward очищается
```

### Пример 5: Преобразование инфиксного выражения в постфиксное

```python
def infix_to_postfix(expression):
    """
    Преобразует инфикс в постфикс (алгоритм Shunting Yard).

    "3 + 4" → "3 4 +"
    "3 + 4 * 2" → "3 4 2 * +"
    "(3 + 4) * 2" → "3 4 + 2 *"
    """
    precedence = {'+': 1, '-': 1, '*': 2, '/': 2, '^': 3}
    right_associative = {'^'}
    output = []
    operator_stack = []

    for token in expression.split():
        if token.isdigit():
            output.append(token)
        elif token in precedence:
            while (operator_stack and
                   operator_stack[-1] != '(' and
                   (precedence[operator_stack[-1]] > precedence[token] or
                    (precedence[operator_stack[-1]] == precedence[token] and
                     token not in right_associative))):
                output.append(operator_stack.pop())
            operator_stack.append(token)
        elif token == '(':
            operator_stack.append(token)
        elif token == ')':
            while operator_stack and operator_stack[-1] != '(':
                output.append(operator_stack.pop())
            operator_stack.pop()  # удаляем '('

    while operator_stack:
        output.append(operator_stack.pop())

    return ' '.join(output)
```

## Типичные ошибки

### 1. Stack Underflow — извлечение из пустого стека

```python
# НЕПРАВИЛЬНО
def pop(self):
    return self.data.pop()  # IndexError если стек пуст

# ПРАВИЛЬНО
def pop(self):
    if self.is_empty():
        raise IndexError("Stack is empty")
    return self.data.pop()
```

### 2. Stack Overflow — переполнение стека фиксированного размера

```python
# НЕПРАВИЛЬНО
def push(self, element):
    self.top += 1
    self.data[self.top] = element  # IndexError

# ПРАВИЛЬНО
def push(self, element):
    if self.is_full():
        raise OverflowError("Stack is full")
    self.top += 1
    self.data[self.top] = element
```

### 3. Путаница с индексом вершины

```python
# Вариант 1: top = индекс последнего элемента
# Пустой стек: top = -1
# После push: top = 0, 1, 2, ...

# Вариант 2: top = индекс следующей свободной позиции
# Пустой стек: top = 0
# После push: top = 1, 2, 3, ...

# Важно быть последовательным!
```

### 4. Забытый возврат значения при pop

```python
# НЕПРАВИЛЬНО
def pop(self):
    if self.is_empty():
        raise IndexError("Stack is empty")
    self.top -= 1  # забыли вернуть элемент!

# ПРАВИЛЬНО
def pop(self):
    if self.is_empty():
        raise IndexError("Stack is empty")
    element = self.data[self.top]
    self.top -= 1
    return element
```

### 5. Неправильный порядок операндов

```python
# При вычислении выражений порядок важен!

# "5 3 -" = 5 - 3 = 2
b = stack.pop()  # 3 (второй операнд)
a = stack.pop()  # 5 (первый операнд)
result = a - b   # 5 - 3 = 2, НЕ 3 - 5!
```

## Сравнение реализаций

| Аспект | Массив | Динамический массив | Связный список |
|--------|--------|---------------------|----------------|
| Размер | Фиксированный | Динамический | Динамический |
| Память | Компактная | С накладными | Указатели |
| Cache locality | Хорошая | Хорошая | Плохая |
| Переполнение | Возможно | Нет | Нет |
| Простота | Простая | Очень простая (Python list) | Средняя |

## Когда использовать стек

**Используй стек, когда:**
- Нужен порядок LIFO
- Операции только с одним концом
- Отслеживание состояния (undo, история)
- Рекурсивные алгоритмы (итеративная замена)
- Обработка вложенных структур (скобки, HTML-теги)

**НЕ используй стек, когда:**
- Нужен доступ к произвольным элементам
- Нужен порядок FIFO (используй очередь)
- Нужно удалять из середины
