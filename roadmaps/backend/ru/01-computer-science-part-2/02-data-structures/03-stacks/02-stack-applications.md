# Применение стека (Stack Applications)

## Определение

Стек — одна из наиболее часто используемых структур данных благодаря своей простоте и эффективности. Принцип LIFO находит применение во множестве алгоритмов и систем.

## Основные области применения

1. **Управление памятью** — Call Stack
2. **Парсинг и компиляция** — синтаксический анализ
3. **Навигация** — undo/redo, история браузера
4. **Алгоритмы** — DFS, сортировка, вычисления
5. **Обработка данных** — конвертация выражений

---

## 1. Call Stack — стек вызовов функций

### Как работает

Каждый вызов функции создаёт **stack frame** (кадр стека):

```
Код:
def main():
    a = foo(5)

def foo(x):
    b = bar(x + 1)
    return b

def bar(y):
    return y * 2

main()
```

```
Эволюция Call Stack:

1. main() вызван:
   ┌─────────────┐
   │   main()    │
   │   a = ?     │
   └─────────────┘

2. foo(5) вызван:
   ┌─────────────┐
   │   foo(5)    │
   │   x = 5     │
   │   b = ?     │
   ├─────────────┤
   │   main()    │
   │   a = ?     │
   └─────────────┘

3. bar(6) вызван:
   ┌─────────────┐
   │   bar(6)    │ ← top
   │   y = 6     │
   ├─────────────┤
   │   foo(5)    │
   │   x = 5     │
   │   b = ?     │
   ├─────────────┤
   │   main()    │
   │   a = ?     │
   └─────────────┘

4. bar() возвращает 12:
   ┌─────────────┐
   │   foo(5)    │ ← top (bar удалён)
   │   x = 5     │
   │   b = 12    │
   ├─────────────┤
   │   main()    │
   │   a = ?     │
   └─────────────┘

5. foo() возвращает 12:
   ┌─────────────┐
   │   main()    │ ← top (foo удалён)
   │   a = 12    │
   └─────────────┘

6. main() завершается → стек пуст
```

### Stack Overflow

```python
def infinite_recursion():
    return infinite_recursion()

infinite_recursion()
# RecursionError: maximum recursion depth exceeded
```

```
┌───────────────┐
│ infinite_...  │ ← Переполнение!
├───────────────┤
│ infinite_...  │
├───────────────┤
│     ...       │  (тысячи кадров)
├───────────────┤
│ infinite_...  │
└───────────────┘
```

---

## 2. Синтаксический анализ

### Проверка сбалансированности скобок

```python
def check_brackets(code):
    """
    Проверяет баланс всех типов скобок в коде.
    Возвращает (is_valid, error_position)
    """
    stack = []
    pairs = {')': '(', ']': '[', '}': '{'}
    opening = set('([{')
    closing = set(')]}')

    for i, char in enumerate(code):
        if char in opening:
            stack.append((char, i))
        elif char in closing:
            if not stack:
                return False, i  # закрывающая без открывающей
            if stack[-1][0] != pairs[char]:
                return False, i  # несоответствие типов
            stack.pop()

    if stack:
        return False, stack[-1][1]  # незакрытая скобка

    return True, -1

# Примеры:
check_brackets("function foo() { return [1, 2]; }")  # (True, -1)
check_brackets("if (x > 0 { }")  # (False, 9) — пропущена )
check_brackets("arr = [1, 2, 3")  # (False, 6) — не закрыта [
```

### Проверка HTML-тегов

```python
import re

def validate_html(html):
    """Проверяет правильность вложенности HTML-тегов"""
    stack = []
    # Находим все теги
    tag_pattern = r'<(/?)(\w+)[^>]*>'

    for match in re.finditer(tag_pattern, html):
        is_closing = match.group(1) == '/'
        tag_name = match.group(2).lower()

        # Пропускаем self-closing теги
        if tag_name in {'br', 'hr', 'img', 'input', 'meta', 'link'}:
            continue

        if is_closing:
            if not stack or stack[-1] != tag_name:
                return False, f"Unexpected </{tag_name}>"
            stack.pop()
        else:
            stack.append(tag_name)

    if stack:
        return False, f"Unclosed <{stack[-1]}>"

    return True, "Valid HTML"

# Примеры:
validate_html("<div><p>Hello</p></div>")  # (True, "Valid HTML")
validate_html("<div><p>Hello</div></p>")  # (False, "Unexpected </div>")
validate_html("<div><span>Text")  # (False, "Unclosed <span>")
```

---

## 3. Undo/Redo механизм

### Реализация для текстового редактора

```python
class TextEditor:
    def __init__(self):
        self.text = ""
        self.undo_stack = []
        self.redo_stack = []

    def write(self, content):
        """Добавляет текст"""
        self.undo_stack.append(('write', content, len(self.text)))
        self.text += content
        self.redo_stack.clear()  # новое действие очищает redo

    def delete(self, count):
        """Удаляет последние count символов"""
        if count > len(self.text):
            count = len(self.text)
        deleted = self.text[-count:]
        self.undo_stack.append(('delete', deleted, len(self.text) - count))
        self.text = self.text[:-count]
        self.redo_stack.clear()

    def undo(self):
        """Отменяет последнее действие"""
        if not self.undo_stack:
            return

        action, content, position = self.undo_stack.pop()
        self.redo_stack.append((action, content, position))

        if action == 'write':
            self.text = self.text[:position]
        elif action == 'delete':
            self.text = self.text[:position] + content + self.text[position:]

    def redo(self):
        """Повторяет отменённое действие"""
        if not self.redo_stack:
            return

        action, content, position = self.redo_stack.pop()
        self.undo_stack.append((action, content, position))

        if action == 'write':
            self.text = self.text[:position] + content + self.text[position:]
        elif action == 'delete':
            end = position + len(content)
            self.text = self.text[:position] + self.text[end:]

# Использование:
editor = TextEditor()
editor.write("Hello")       # text: "Hello"
editor.write(" World")      # text: "Hello World"
editor.delete(6)            # text: "Hello"
editor.undo()               # text: "Hello World"
editor.undo()               # text: "Hello"
editor.redo()               # text: "Hello World"
```

```
Визуализация стеков:

После write("Hello"), write(" World"), delete(6):

Undo Stack:                    Redo Stack:
┌─────────────────────┐       ┌─────────────────────┐
│ delete(" World", 5) │       │       (пусто)       │
├─────────────────────┤       └─────────────────────┘
│ write(" World", 5)  │
├─────────────────────┤
│ write("Hello", 0)   │
└─────────────────────┘

После undo():

Undo Stack:                    Redo Stack:
┌─────────────────────┐       ┌─────────────────────┐
│ write(" World", 5)  │       │ delete(" World", 5) │
├─────────────────────┤       └─────────────────────┘
│ write("Hello", 0)   │
└─────────────────────┘
```

---

## 4. Обход графа в глубину (DFS)

### Итеративная реализация с использованием стека

```python
def dfs_iterative(graph, start):
    """
    Обход графа в глубину.
    graph — словарь смежности: {node: [neighbors]}
    """
    visited = set()
    stack = [start]
    result = []

    while stack:
        node = stack.pop()

        if node in visited:
            continue

        visited.add(node)
        result.append(node)

        # Добавляем соседей в стек (в обратном порядке для правильного обхода)
        for neighbor in reversed(graph[node]):
            if neighbor not in visited:
                stack.append(neighbor)

    return result

# Пример графа:
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

print(dfs_iterative(graph, 'A'))  # ['A', 'B', 'D', 'E', 'C', 'F']
```

```
Трассировка DFS:

Шаг 1: stack = [A], visited = {}, result = []
       pop A → visited = {A}, result = [A]
       push C, B → stack = [C, B]

Шаг 2: stack = [C, B], pop B
       visited = {A, B}, result = [A, B]
       push E, D → stack = [C, E, D]

Шаг 3: stack = [C, E, D], pop D
       visited = {A, B, D}, result = [A, B, D]
       no neighbors → stack = [C, E]

Шаг 4: stack = [C, E], pop E
       visited = {A, B, D, E}, result = [A, B, D, E]
       stack = [C]

Шаг 5: stack = [C], pop C
       visited = {A, B, D, E, C}, result = [A, B, D, E, C]
       push F → stack = [F]

Шаг 6: stack = [F], pop F
       visited = {A, B, D, E, C, F}, result = [A, B, D, E, C, F]
       stack = []

Результат: [A, B, D, E, C, F]
```

### Поиск пути в лабиринте

```python
def find_path(maze, start, end):
    """
    Находит путь в лабиринте с помощью DFS.
    maze — 2D массив: 0 = проход, 1 = стена
    """
    rows, cols = len(maze), len(maze[0])
    visited = set()
    stack = [(start, [start])]  # (позиция, путь до неё)

    while stack:
        (row, col), path = stack.pop()

        if (row, col) == end:
            return path  # нашли выход!

        if (row, col) in visited:
            continue

        visited.add((row, col))

        # Проверяем все направления: вверх, вниз, влево, вправо
        for dr, dc in [(-1, 0), (1, 0), (0, -1), (0, 1)]:
            new_row, new_col = row + dr, col + dc

            if (0 <= new_row < rows and
                0 <= new_col < cols and
                maze[new_row][new_col] == 0 and
                (new_row, new_col) not in visited):
                stack.append(((new_row, new_col), path + [(new_row, new_col)]))

    return None  # путь не найден

# Пример:
maze = [
    [0, 1, 0, 0, 0],
    [0, 1, 0, 1, 0],
    [0, 0, 0, 1, 0],
    [1, 1, 0, 0, 0],
    [0, 0, 0, 1, 0]
]

path = find_path(maze, (0, 0), (4, 4))
# [(0,0), (1,0), (2,0), (2,1), (2,2), (3,2), (3,3), (3,4), (4,4)]
```

---

## 5. Вычисление математических выражений

### Преобразование инфикс → постфикс (Shunting Yard)

```python
def infix_to_postfix(expression):
    """
    Алгоритм сортировочной станции Дейкстры.

    "3 + 4 * 2" → "3 4 2 * +"
    "(3 + 4) * 2" → "3 4 + 2 *"
    "3 + 4 * 2 / ( 1 - 5 ) ^ 2" → "3 4 2 * 1 5 - 2 ^ / +"
    """
    precedence = {'+': 1, '-': 1, '*': 2, '/': 2, '^': 3}
    right_associative = {'^'}

    output = []
    operators = []

    tokens = expression.replace('(', ' ( ').replace(')', ' ) ').split()

    for token in tokens:
        if token.replace('.', '').isdigit():  # число
            output.append(token)

        elif token in precedence:  # оператор
            while (operators and
                   operators[-1] != '(' and
                   operators[-1] in precedence and
                   (precedence[operators[-1]] > precedence[token] or
                    (precedence[operators[-1]] == precedence[token] and
                     token not in right_associative))):
                output.append(operators.pop())
            operators.append(token)

        elif token == '(':
            operators.append(token)

        elif token == ')':
            while operators and operators[-1] != '(':
                output.append(operators.pop())
            operators.pop()  # удаляем '('

    while operators:
        output.append(operators.pop())

    return ' '.join(output)
```

### Полный калькулятор

```python
def calculate(expression):
    """
    Вычисляет математическое выражение.
    Поддерживает: +, -, *, /, ^, скобки
    """
    postfix = infix_to_postfix(expression)
    return evaluate_postfix(postfix)

def evaluate_postfix(postfix):
    """Вычисляет постфиксное выражение"""
    stack = []

    for token in postfix.split():
        if token.replace('.', '').replace('-', '').isdigit():
            stack.append(float(token))
        else:
            b = stack.pop()
            a = stack.pop()

            if token == '+':
                stack.append(a + b)
            elif token == '-':
                stack.append(a - b)
            elif token == '*':
                stack.append(a * b)
            elif token == '/':
                stack.append(a / b)
            elif token == '^':
                stack.append(a ** b)

    return stack[0]

# Примеры:
calculate("3 + 4 * 2")  # 11
calculate("(3 + 4) * 2")  # 14
calculate("2 ^ 3 ^ 2")  # 512 (правая ассоциативность: 2^(3^2))
```

---

## 6. Монотонный стек

### Следующий больший элемент

```python
def next_greater_element(arr):
    """
    Для каждого элемента находит следующий больший справа.
    Если нет большего — возвращает -1.

    [4, 5, 2, 10, 8] → [5, 10, 10, -1, -1]
    """
    n = len(arr)
    result = [-1] * n
    stack = []  # хранит индексы

    for i in range(n):
        # Пока текущий элемент больше элемента на вершине стека
        while stack and arr[i] > arr[stack[-1]]:
            idx = stack.pop()
            result[idx] = arr[i]
        stack.append(i)

    return result

# Трассировка для [4, 5, 2, 10, 8]:
# i=0: arr[0]=4, stack=[], push 0 → stack=[0]
# i=1: arr[1]=5 > arr[0]=4, pop 0, result[0]=5, push 1 → stack=[1]
# i=2: arr[2]=2 < arr[1]=5, push 2 → stack=[1,2]
# i=3: arr[3]=10 > arr[2]=2, pop 2, result[2]=10
#      arr[3]=10 > arr[1]=5, pop 1, result[1]=10
#      push 3 → stack=[3]
# i=4: arr[4]=8 < arr[3]=10, push 4 → stack=[3,4]
# result = [5, 10, 10, -1, -1]
```

### Максимальный прямоугольник в гистограмме

```python
def largest_rectangle_histogram(heights):
    """
    Находит площадь максимального прямоугольника в гистограмме.

    heights = [2, 1, 5, 6, 2, 3]

        █
      █ █
      █ █   █
    █ █ █ █ █
    █ █ █ █ █ █
    ─────────────
    2 1 5 6 2 3

    Максимальный прямоугольник: высота 5, ширина 2 → площадь 10
    """
    stack = []  # хранит индексы
    max_area = 0
    heights = heights + [0]  # добавляем 0 для обработки оставшихся

    for i, h in enumerate(heights):
        start = i

        while stack and stack[-1][1] > h:
            idx, height = stack.pop()
            width = i - idx
            area = height * width
            max_area = max(max_area, area)
            start = idx

        stack.append((start, h))

    return max_area

# largest_rectangle_histogram([2, 1, 5, 6, 2, 3]) → 10
```

---

## 7. Обработка рекурсии итеративно

### Преобразование рекурсивной функции

```python
# Рекурсивная версия
def factorial_recursive(n):
    if n <= 1:
        return 1
    return n * factorial_recursive(n - 1)

# Итеративная версия с явным стеком
def factorial_iterative(n):
    stack = []
    result = 1

    # Имитируем рекурсивные вызовы
    while n > 1:
        stack.append(n)
        n -= 1

    # Имитируем возвраты
    while stack:
        result *= stack.pop()

    return result
```

### Обход дерева без рекурсии

```python
class TreeNode:
    def __init__(self, val):
        self.val = val
        self.left = None
        self.right = None

def inorder_iterative(root):
    """
    Inorder обход (левый → корень → правый) без рекурсии.
    """
    result = []
    stack = []
    current = root

    while current or stack:
        # Идём максимально влево
        while current:
            stack.append(current)
            current = current.left

        # Обрабатываем узел
        current = stack.pop()
        result.append(current.val)

        # Переходим к правому поддереву
        current = current.right

    return result

def preorder_iterative(root):
    """Preorder обход (корень → левый → правый)"""
    if not root:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)

        # Правый добавляем первым, чтобы левый обработался раньше
        if node.right:
            stack.append(node.right)
        if node.left:
            stack.append(node.left)

    return result

def postorder_iterative(root):
    """Postorder обход (левый → правый → корень)"""
    if not root:
        return []

    result = []
    stack = [root]

    while stack:
        node = stack.pop()
        result.append(node.val)

        if node.left:
            stack.append(node.left)
        if node.right:
            stack.append(node.right)

    return result[::-1]  # реверсируем результат
```

---

## Сводная таблица применений

| Применение | Что храним в стеке | Сложность |
|------------|-------------------|-----------|
| Call Stack | Stack frames | O(глубина вызовов) |
| Баланс скобок | Открывающие скобки | O(n) |
| Undo/Redo | Операции | O(1) на операцию |
| DFS | Вершины для посещения | O(V + E) |
| Инфикс → Постфикс | Операторы | O(n) |
| Вычисление выражений | Операнды | O(n) |
| Монотонный стек | Индексы/значения | O(n) |
| Итеративный обход дерева | Узлы | O(n) |

## Типичные ошибки

### 1. Забытая очистка стека Redo при новом действии

```python
# НЕПРАВИЛЬНО
def new_action(self):
    self.undo_stack.append(action)
    # redo_stack не очищен — можно сделать redo после нового действия!

# ПРАВИЛЬНО
def new_action(self):
    self.undo_stack.append(action)
    self.redo_stack.clear()  # очищаем историю redo
```

### 2. Неправильный порядок добавления соседей в DFS

```python
# Если нужен определённый порядок обхода
for neighbor in graph[node]:  # обход в обратном порядке
    stack.append(neighbor)

# Правильно для порядка [A, B, C]:
for neighbor in reversed(graph[node]):
    stack.append(neighbor)
```

### 3. Бесконечный цикл в лабиринте без visited

```python
# НЕПРАВИЛЬНО
stack = [start]
while stack:
    pos = stack.pop()
    for neighbor in get_neighbors(pos):
        stack.append(neighbor)  # может добавить уже посещённые!

# ПРАВИЛЬНО
visited = set()
while stack:
    pos = stack.pop()
    if pos in visited:
        continue
    visited.add(pos)
    # ...
```
