# Static Typing (Статическая типизация)

[prev: 10c-pattern-matching](./10c-pattern-matching.md) | [next: 12-code-formatting](./12-code-formatting.md)
---

## Зачем?

- Раннее обнаружение ошибок
- Документация в коде
- Автодополнение в IDE

---

## Базовый синтаксис

```python
# Переменные
name: str = "Анна"
age: int = 25

# Функции
def greet(name: str) -> str:
    return f"Hello, {name}"

def log(msg: str) -> None:
    print(msg)
```

---

## Составные типы

```python
# Python 3.9+
names: list[str] = ["Анна", "Борис"]
ages: dict[str, int] = {"Анна": 25}
point: tuple[int, int] = (10, 20)

# Optional — может быть None
from typing import Optional
def find(id: int) -> Optional[str]:  # str | None
    ...

# Union — один из типов
from typing import Union
def process(x: Union[int, str]) -> str:  # int | str
    ...

# Python 3.10+ — новый синтаксис
def find(id: int) -> str | None: ...
def process(x: int | str) -> str: ...
```

---

## Специальные типы

```python
from typing import Any, Callable, Literal, TypeVar

# Any — отключает проверку
data: Any

# Callable — функция
handler: Callable[[str, int], bool]  # (str, int) -> bool

# Literal — конкретные значения
def mode(m: Literal["read", "write"]) -> None: ...

# TypeVar — дженерики
T = TypeVar("T")
def first(items: list[T]) -> T:
    return items[0]
```

---

## Protocol — структурная типизация

```python
from typing import Protocol

class Drawable(Protocol):
    def draw(self) -> None: ...

def render(shape: Drawable) -> None:
    shape.draw()

# Любой класс с draw() подойдёт
```

---

## TypedDict

```python
from typing import TypedDict

class User(TypedDict):
    name: str
    age: int

user: User = {"name": "Анна", "age": 25}
```

---

## Type Aliases

```python
from typing import TypeAlias

UserId: TypeAlias = int
UserDict: TypeAlias = dict[str, str | int]
```

---

## mypy — проверка

```bash
pip install mypy
mypy script.py
```

```toml
# pyproject.toml
[tool.mypy]
python_version = "3.11"
strict = true
```

```python
# Игнорировать ошибку
x: int = "s"  # type: ignore
```

---

## ParamSpec — типизация декораторов

Сохраняет сигнатуру функции:

```python
from typing import Callable, ParamSpec, TypeVar

P = ParamSpec("P")  # Захватывает параметры
R = TypeVar("R")

def logged(func: Callable[P, R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        print(f"Calling {func.__name__}")
        return func(*args, **kwargs)
    return wrapper

@logged
def greet(name: str, times: int = 1) -> str:
    return f"Hello, {name}!" * times

# IDE знает: greet(name: str, times: int = 1) -> str
```

### Concatenate — добавление параметров

```python
from typing import Concatenate

def with_user(func: Callable[Concatenate[str, P], R]) -> Callable[P, R]:
    def wrapper(*args: P.args, **kwargs: P.kwargs) -> R:
        return func(get_current_user(), *args, **kwargs)
    return wrapper
```

---

## Variance (Вариантность)

Как соотносятся `Generic[A]` и `Generic[B]`, если `A` подтип `B`?

### Инвариантность (Invariant)

`list[Dog]` **НЕ** подтип `list[Animal]` — по умолчанию.

```python
class Animal: pass
class Dog(Animal): pass

def add_animal(animals: list[Animal]) -> None:
    animals.append(Animal())

dogs: list[Dog] = [Dog()]
add_animal(dogs)  # Error! Иначе Animal попал бы в list[Dog]
```

### Ковариантность (Covariant)

`Sequence[Dog]` подтип `Sequence[Animal]` — только чтение.

```python
from typing import Sequence

def count(animals: Sequence[Animal]) -> int:
    return len(animals)

dogs: list[Dog] = [Dog()]
count(dogs)  # OK! Sequence ковариантен
```

### Контравариантность (Contravariant)

`Callable[[Animal], None]` подтип `Callable[[Dog], None]` — аргументы.

```python
def process_dog(handler: Callable[[Dog], None]) -> None:
    handler(Dog())

def handle_animal(a: Animal) -> None: ...

process_dog(handle_animal)  # OK! Animal handler может обработать Dog
```

### TypeVar с вариантностью

```python
T = TypeVar("T")                           # Инвариант (по умолчанию)
T_co = TypeVar("T_co", covariant=True)     # Ковариант (только чтение)
T_contra = TypeVar("T_contra", contravariant=True)  # Контравариант
```

### Таблица

| Вид | Если `Dog <: Animal` | Когда |
|-----|---------------------|-------|
| Invariant | `list[Dog]` ≠ `list[Animal]` | Чтение + запись |
| Covariant | `Seq[Dog]` <: `Seq[Animal]` | Только чтение |
| Contravariant | `Fn[[Animal]]` <: `Fn[[Dog]]` | Только вход |

---

## Резюме

| Тип | Пример |
|-----|--------|
| Базовые | `int`, `str`, `bool`, `float` |
| Коллекции | `list[str]`, `dict[str, int]` |
| Optional | `str \| None` |
| Union | `int \| str` |
| Callable | `Callable[[int], str]` |
| Literal | `Literal["a", "b"]` |
| Protocol | Структурная типизация |
| TypedDict | Типизированные словари |

- Type hints не влияют на runtime
- mypy проверяет типы статически
- `strict = true` для строгой проверки

---
[prev: 10c-pattern-matching](./10c-pattern-matching.md) | [next: 12-code-formatting](./12-code-formatting.md)