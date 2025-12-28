# Paradigms (Парадигмы программирования)

[prev: 05-protocols](08-oop/05-protocols.md) | [next: 08b-context-manager](./08b-context-manager.md)
---

Python — **мультипарадигменный** язык: процедурный, ООП, функциональный.

---

## 1. Procedural (Процедурное)

Код выполняется последовательно, организован в функции.

```python
users = [{"name": "Анна", "age": 25}, {"name": "Борис", "age": 17}]

def filter_adults(users):
    return [u for u in users if u["age"] >= 18]

def get_names(users):
    return [u["name"] for u in users]

names = get_names(filter_adults(users))
```

**Когда:** простые скрипты, автоматизация.

---

## 2. Object-Oriented (ООП)

Данные и поведение объединены в классах.

```python
class User:
    def __init__(self, name, age):
        self.name = name
        self.age = age

    def is_adult(self):
        return self.age >= 18

class UserCollection:
    def __init__(self, users):
        self.users = users

    def filter_adults(self):
        return UserCollection([u for u in self.users if u.is_adult()])

    def get_names(self):
        return [u.name for u in self.users]

names = UserCollection(users).filter_adults().get_names()
```

**Когда:** сложные системы, много сущностей.

---

## 3. Functional (Функциональное)

Чистые функции, иммутабельность, функции высшего порядка.

### Принципы:

#### Чистые функции

```python
# Чистая — нет побочных эффектов
def add(a, b):
    return a + b

# Нечистая — изменяет внешнее состояние
total = 0
def add_impure(a):
    global total
    total += a
```

#### Иммутабельность

```python
# Плохо — мутация
def add_item_bad(lst, item):
    lst.append(item)
    return lst

# Хорошо — новый список
def add_item_good(lst, item):
    return [*lst, item]
```

#### Функции высшего порядка

```python
from functools import reduce

users = [{"name": "Анна", "age": 25}, {"name": "Борис", "age": 17}]

# map, filter
adults = filter(lambda u: u["age"] >= 18, users)
names = map(lambda u: u["name"], adults)
list(names)  # ['Анна']

# reduce — свёртка
numbers = [1, 2, 3, 4, 5]
reduce(lambda acc, x: acc + x, numbers, 0)  # 15
```

#### Композиция функций

```python
def compose(*funcs):
    def inner(x):
        for f in reversed(funcs):
            x = f(x)
        return x
    return inner

double = lambda x: x * 2
increment = lambda x: x + 1

process = compose(double, increment)  # (x + 1) * 2
process(5)  # 12
```

---

## Сравнение

| Аспект | Procedural | OOP | Functional |
|--------|-----------|-----|------------|
| Организация | Функции | Классы | Чистые функции |
| Состояние | Глобальное | В объектах | Избегается |
| Данные | Отдельно | Инкапсулированы | Иммутабельные |
| Переиспользование | Функции | Наследование | Композиция |

---

## Python: смешанный подход

На практике комбинируют парадигмы:

```python
from dataclasses import dataclass

@dataclass(frozen=True)  # Иммутабельный класс (OOP + FP)
class User:
    name: str
    age: int

    def is_adult(self) -> bool:
        return self.age >= 18

# Чистая функция (FP)
def filter_adults(users: list[User]) -> list[User]:
    return [u for u in users if u.is_adult()]
```

---

## Резюме

- **Procedural** — просто, для скриптов
- **OOP** — структура, для больших систем
- **Functional** — предсказуемость, для обработки данных
- **Python** — используй то, что подходит задаче

---
[prev: 05-protocols](08-oop/05-protocols.md) | [next: 08b-context-manager](./08b-context-manager.md)