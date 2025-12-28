# Pattern Matching (Сопоставление с образцом)

[prev: 10b-common-packages](./10b-common-packages.md) | [next: 11-static-typing](./11-static-typing.md)
---

## Введение

**Pattern Matching** (PEP 634, Python 3.10+) — мощный механизм для сопоставления значений с шаблонами. Похоже на `switch/case` в других языках, но гораздо мощнее.

```python
def describe(value):
    match value:
        case 0:
            return "zero"
        case 1:
            return "one"
        case _:
            return "something else"

describe(0)   # "zero"
describe(42)  # "something else"
```

## Literal Patterns (Литералы)

Сопоставление с конкретными значениями:

```python
def http_status(status):
    match status:
        case 200:
            return "OK"
        case 404:
            return "Not Found"
        case 500:
            return "Internal Server Error"
        case _:
            return "Unknown status"

# Строки
def handle_command(command):
    match command:
        case "start":
            return "Starting..."
        case "stop":
            return "Stopping..."
        case "help":
            return "Available commands: start, stop, help"
        case _:
            return "Unknown command"
```

### Комбинация литералов с |

```python
def is_weekend(day):
    match day:
        case "Saturday" | "Sunday":
            return True
        case "Monday" | "Tuesday" | "Wednesday" | "Thursday" | "Friday":
            return False
        case _:
            return None
```

## Capture Patterns (Захват значений)

Переменные в паттернах захватывают значения:

```python
def handle_point(point):
    match point:
        case (0, 0):
            return "Origin"
        case (0, y):
            return f"On Y axis at {y}"
        case (x, 0):
            return f"On X axis at {x}"
        case (x, y):
            return f"Point at ({x}, {y})"

handle_point((0, 0))    # "Origin"
handle_point((0, 5))    # "On Y axis at 5"
handle_point((3, 4))    # "Point at (3, 4)"
```

## Wildcard Pattern (_)

`_` — паттерн-подстановка, соответствует чему угодно, но не захватывает:

```python
def first_element(seq):
    match seq:
        case [first, *_]:  # _ для остальных элементов
            return first
        case _:
            return None

first_element([1, 2, 3])  # 1
first_element([])          # None
```

## Sequence Patterns (Последовательности)

Работа со списками и кортежами:

```python
def analyze_list(lst):
    match lst:
        case []:
            return "Empty list"
        case [x]:
            return f"Single element: {x}"
        case [x, y]:
            return f"Two elements: {x} and {y}"
        case [first, *rest]:
            return f"First: {first}, rest: {rest}"

analyze_list([])           # "Empty list"
analyze_list([1])          # "Single element: 1"
analyze_list([1, 2])       # "Two elements: 1 and 2"
analyze_list([1, 2, 3, 4]) # "First: 1, rest: [2, 3, 4]"
```

### Вложенные последовательности

```python
def handle_nested(data):
    match data:
        case [[a, b], [c, d]]:
            return f"2x2 matrix: {a}, {b}, {c}, {d}"
        case [first, [nested_first, *nested_rest], *rest]:
            return f"Complex: {first}, nested={nested_first}, ..."
        case _:
            return "Unknown structure"
```

## Mapping Patterns (Словари)

```python
def handle_event(event):
    match event:
        case {"type": "click", "x": x, "y": y}:
            return f"Click at ({x}, {y})"
        case {"type": "keypress", "key": key}:
            return f"Key pressed: {key}"
        case {"type": event_type}:
            return f"Unknown event: {event_type}"
        case _:
            return "Invalid event"

handle_event({"type": "click", "x": 100, "y": 200})
# "Click at (100, 200)"

handle_event({"type": "keypress", "key": "Enter", "modifiers": ["ctrl"]})
# "Key pressed: Enter" (лишние ключи игнорируются!)
```

### Захват остальных ключей

```python
def parse_config(config):
    match config:
        case {"host": host, "port": port, **rest}:
            return f"Server at {host}:{port}, extra: {rest}"
        case _:
            return "Invalid config"

parse_config({"host": "localhost", "port": 8080, "debug": True})
# "Server at localhost:8080, extra: {'debug': True}"
```

## Class Patterns (Объекты)

Сопоставление с атрибутами объектов:

```python
from dataclasses import dataclass

@dataclass
class Point:
    x: float
    y: float

@dataclass
class Circle:
    center: Point
    radius: float

@dataclass
class Rectangle:
    corner: Point
    width: float
    height: float

def describe_shape(shape):
    match shape:
        case Point(x=0, y=0):
            return "Origin point"
        case Point(x=x, y=y):
            return f"Point at ({x}, {y})"
        case Circle(center=Point(x=0, y=0), radius=r):
            return f"Circle at origin with radius {r}"
        case Circle(center=c, radius=r):
            return f"Circle at {c} with radius {r}"
        case Rectangle(width=w, height=h) if w == h:
            return f"Square with side {w}"
        case Rectangle(width=w, height=h):
            return f"Rectangle {w}x{h}"

describe_shape(Point(0, 0))
# "Origin point"

describe_shape(Circle(Point(0, 0), 5))
# "Circle at origin with radius 5"
```

### Позиционные аргументы в паттернах

```python
# С __match_args__ можно использовать позиционные паттерны
@dataclass
class Point:
    x: float
    y: float
    # dataclass автоматически создаёт __match_args__ = ('x', 'y')

def handle_point(point):
    match point:
        case Point(0, 0):          # Позиционно
            return "Origin"
        case Point(0, y):          # Смешанно
            return f"Y axis at {y}"
        case Point(x, y):
            return f"({x}, {y})"
```

## Guards (Условия)

Добавление условий к паттернам:

```python
def categorize_number(n):
    match n:
        case x if x < 0:
            return "negative"
        case x if x == 0:
            return "zero"
        case x if x < 10:
            return "small positive"
        case x if x < 100:
            return "medium positive"
        case _:
            return "large positive"

def validate_user(user):
    match user:
        case {"name": name, "age": age} if age >= 18:
            return f"{name} is an adult"
        case {"name": name, "age": age} if age > 0:
            return f"{name} is a minor"
        case {"name": name}:
            return f"{name} has no age specified"
        case _:
            return "Invalid user"
```

## AS Pattern (Именование)

Захват части паттерна в переменную:

```python
def handle_command(command):
    match command:
        case ["go", ("north" | "south" | "east" | "west") as direction]:
            return f"Going {direction}"
        case ["attack", target]:
            return f"Attacking {target}"

handle_command(["go", "north"])  # "Going north"
```

```python
def process_data(data):
    match data:
        case {"user": {"name": name, "address": addr} as user_data}:
            return f"User {name} with full data: {user_data}"
```

## Практические примеры

### Парсинг команд

```python
def parse_command(command: str):
    parts = command.split()
    match parts:
        case ["quit" | "exit" | "q"]:
            return {"action": "quit"}
        case ["help"]:
            return {"action": "help"}
        case ["go", direction]:
            return {"action": "move", "direction": direction}
        case ["take", item]:
            return {"action": "take", "item": item}
        case ["use", item, "on", target]:
            return {"action": "use", "item": item, "target": target}
        case _:
            return {"action": "unknown", "input": command}
```

### Обработка JSON API

```python
def handle_api_response(response: dict):
    match response:
        case {"status": "success", "data": data}:
            return process_data(data)
        case {"status": "error", "code": code, "message": msg}:
            return handle_error(code, msg)
        case {"status": "pending", "retry_after": seconds}:
            return schedule_retry(seconds)
        case _:
            raise ValueError(f"Unknown response format: {response}")
```

### Обработка AST

```python
def evaluate(node):
    match node:
        case {"type": "number", "value": n}:
            return n
        case {"type": "add", "left": left, "right": right}:
            return evaluate(left) + evaluate(right)
        case {"type": "multiply", "left": left, "right": right}:
            return evaluate(left) * evaluate(right)
        case {"type": "negate", "operand": operand}:
            return -evaluate(operand)

ast = {
    "type": "add",
    "left": {"type": "number", "value": 5},
    "right": {
        "type": "multiply",
        "left": {"type": "number", "value": 3},
        "right": {"type": "number", "value": 2}
    }
}
evaluate(ast)  # 11 (5 + 3*2)
```

### State Machine

```python
from enum import Enum, auto

class State(Enum):
    IDLE = auto()
    RUNNING = auto()
    PAUSED = auto()
    STOPPED = auto()

def transition(state: State, action: str) -> State:
    match (state, action):
        case (State.IDLE, "start"):
            return State.RUNNING
        case (State.RUNNING, "pause"):
            return State.PAUSED
        case (State.RUNNING, "stop"):
            return State.STOPPED
        case (State.PAUSED, "resume"):
            return State.RUNNING
        case (State.PAUSED, "stop"):
            return State.STOPPED
        case (State.STOPPED, "reset"):
            return State.IDLE
        case _:
            raise ValueError(f"Invalid transition: {state} + {action}")
```

## Сравнение с if/elif

```python
# С if/elif — многословно
def process_old(data):
    if isinstance(data, dict):
        if "type" in data and data["type"] == "user":
            if "name" in data and "age" in data:
                name, age = data["name"], data["age"]
                if age >= 18:
                    return f"Adult user: {name}"
    return "Unknown"

# С pattern matching — элегантно
def process_new(data):
    match data:
        case {"type": "user", "name": name, "age": age} if age >= 18:
            return f"Adult user: {name}"
        case _:
            return "Unknown"
```

## Best Practices

### 1. Порядок паттернов важен

```python
# ❌ Общий паттерн первым — остальные недостижимы
match value:
    case _:
        return "catch all"
    case 0:  # Никогда не выполнится!
        return "zero"

# ✅ От частного к общему
match value:
    case 0:
        return "zero"
    case _:
        return "catch all"
```

### 2. Используйте guards для сложных условий

```python
# ✅ Guard для сложной логики
match user:
    case {"age": age} if 0 < age < 18:
        return "minor"
    case {"age": age} if age >= 18:
        return "adult"
```

### 3. Не забывайте _ для полноты

```python
def handle(x):
    match x:
        case 1:
            return "one"
        case 2:
            return "two"
        case _:  # Всегда добавляйте fallback
            return "other"
```

## Q&A

**Q: Можно ли использовать match для проверки типов?**
A: Да, через class patterns: `case int() as n:` или `case str() as s:`

**Q: Работает ли match с regex?**
A: Напрямую нет, но можно через guards: `case s if re.match(pattern, s):`

**Q: Можно ли использовать константы в паттернах?**
A: Да, но нужно использовать dotted names: `case Status.OK:`, а не просто `case OK:` (OK будет захватом)

**Q: match влияет на производительность?**
A: Компилятор оптимизирует match, производительность сравнима с if/elif.

---
[prev: 10b-common-packages](./10b-common-packages.md) | [next: 11-static-typing](./11-static-typing.md)