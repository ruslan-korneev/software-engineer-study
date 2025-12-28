# Функции (Functions)

[prev: 06-exceptions](./06-exceptions.md) | [next: 08-lists-tuples-sets](./08-lists-tuples-sets.md)
---

Функция — это именованный блок кода, который можно вызывать многократно.

## Зачем нужны функции?

- **Переиспользование** — пишешь код один раз, вызываешь много
- **Читаемость** — разбиваешь программу на логические части
- **Тестируемость** — легко тестировать отдельные части
- **Абстракция** — скрываешь детали реализации

## Создание функции

```python
def greet():
    print("Привет!")

# Вызов функции
greet()  # Привет!
greet()  # Привет!
```

## Параметры и аргументы

```python
def greet(name):
    print(f"Привет, {name}!")

greet("Ruslan")  # Привет, Ruslan!
greet("Anna")    # Привет, Anna!
```

```python
# Несколько параметров
def add(a, b):
    return a + b

result = add(2, 3)  # 5
```

**Параметр** — переменная в определении функции.
**Аргумент** — значение, переданное при вызове.

## return — возврат значения

```python
def add(a, b):
    return a + b

result = add(2, 3)
print(result)  # 5
```

### Без return функция возвращает None
```python
def no_return():
    print("Hello")

x = no_return()  # выведет Hello
print(x)         # None
```

### Множественный return
```python
def get_status(age):
    if age < 18:
        return "minor"
    return "adult"  # else не нужен — после return функция завершается
```

### Возврат нескольких значений
```python
def divide(a, b):
    quotient = a // b
    remainder = a % b
    return quotient, remainder  # возвращается кортеж

q, r = divide(10, 3)
print(q, r)  # 3 1

# Или получить как кортеж
result = divide(10, 3)
print(result)  # (3, 1)
```

## Значения по умолчанию

```python
def greet(name, greeting="Привет"):
    print(f"{greeting}, {name}!")

greet("Ruslan")                # Привет, Ruslan!
greet("Ruslan", "Здравствуй")  # Здравствуй, Ruslan!
```

### Параметры с default — в конце!
```python
# Правильно
def func(a, b, c=10):
    pass

# Ошибка!
def func(a, b=10, c):  # SyntaxError
    pass
```

### Осторожно с изменяемыми значениями!
```python
# ПЛОХО — список создаётся один раз!
def append_item(item, lst=[]):
    lst.append(item)
    return lst

append_item(1)  # [1]
append_item(2)  # [1, 2] — не [2]!

# ХОРОШО
def append_item(item, lst=None):
    if lst is None:
        lst = []
    lst.append(item)
    return lst
```

## Позиционные и именованные аргументы

```python
def create_user(name, age, city):
    print(f"{name}, {age} лет, {city}")

# Позиционные — по порядку
create_user("Ruslan", 25, "Moscow")

# Именованные — порядок не важен
create_user(age=25, city="Moscow", name="Ruslan")

# Смешанные — позиционные сначала
create_user("Ruslan", city="Moscow", age=25)
```

## *args — произвольное число позиционных аргументов

```python
def sum_all(*numbers):
    total = 0
    for num in numbers:
        total += num
    return total

sum_all(1, 2, 3)        # 6
sum_all(1, 2, 3, 4, 5)  # 15
sum_all()               # 0

# *args — это кортеж
def show_args(*args):
    print(type(args))  # <class 'tuple'>
    print(args)        # (1, 2, 3)

show_args(1, 2, 3)
```

## **kwargs — произвольное число именованных аргументов

```python
def print_info(**kwargs):
    for key, value in kwargs.items():
        print(f"{key}: {value}")

print_info(name="Ruslan", age=25, city="Moscow")
# name: Ruslan
# age: 25
# city: Moscow

# **kwargs — это словарь
def show_kwargs(**kwargs):
    print(type(kwargs))  # <class 'dict'>
    print(kwargs)        # {'a': 1, 'b': 2}

show_kwargs(a=1, b=2)
```

## Комбинация всех типов параметров

```python
def func(a, b, *args, c=10, **kwargs):
    print(f"a={a}, b={b}")
    print(f"args={args}")
    print(f"c={c}")
    print(f"kwargs={kwargs}")

func(1, 2, 3, 4, 5, c=20, x=100, y=200)
# a=1, b=2
# args=(3, 4, 5)
# c=20
# kwargs={'x': 100, 'y': 200}
```

**Порядок параметров:**
1. Обычные позиционные
2. `*args`
3. Именованные с default
4. `**kwargs`

## Только позиционные / только именованные (Python 3.8+)

```python
# / — до него только позиционные
def func(a, b, /, c, d):
    pass

func(1, 2, 3, d=4)    # OK
func(1, 2, c=3, d=4)  # OK
func(a=1, b=2, c=3, d=4)  # Ошибка! a и b только позиционные

# * — после него только именованные
def func(a, b, *, c, d):
    pass

func(1, 2, c=3, d=4)  # OK
func(1, 2, 3, 4)      # Ошибка! c и d только именованные
```

## Распаковка при вызове

```python
def add(a, b, c):
    return a + b + c

# Распаковка списка/кортежа через *
nums = [1, 2, 3]
add(*nums)  # = add(1, 2, 3) = 6

# Распаковка словаря через **
params = {"a": 1, "b": 2, "c": 3}
add(**params)  # = add(a=1, b=2, c=3) = 6
```

## Область видимости (Scope)

### LEGB — порядок поиска переменных
- **L**ocal — внутри функции
- **E**nclosing — во внешней функции (для вложенных)
- **G**lobal — на уровне модуля
- **B**uilt-in — встроенные (print, len, ...)

```python
x = "global"

def outer():
    x = "enclosing"

    def inner():
        x = "local"
        print(x)  # local

    inner()
    print(x)  # enclosing

outer()
print(x)  # global
```

### global — изменить глобальную переменную
```python
count = 0

def increment():
    global count
    count += 1

increment()
increment()
print(count)  # 2
```

### nonlocal — изменить переменную внешней функции
```python
def outer():
    x = 10

    def inner():
        nonlocal x
        x = 20

    inner()
    print(x)  # 20

outer()
```

## Docstring — документация функции

```python
def calculate_area(width, height):
    """
    Вычисляет площадь прямоугольника.

    Args:
        width: Ширина прямоугольника
        height: Высота прямоугольника

    Returns:
        Площадь прямоугольника

    Raises:
        ValueError: Если width или height отрицательные
    """
    if width < 0 or height < 0:
        raise ValueError("Размеры должны быть положительными")
    return width * height

# Доступ к документации
print(calculate_area.__doc__)
help(calculate_area)
```

## Функция как объект

В Python функции — это объекты первого класса:

```python
def greet(name):
    return f"Hello, {name}"

# Присвоить переменной
say_hello = greet
say_hello("Ruslan")  # Hello, Ruslan

# Передать в другую функцию
def apply(func, value):
    return func(value)

apply(greet, "Anna")  # Hello, Anna

# Вернуть из функции
def create_multiplier(n):
    def multiply(x):
        return x * n
    return multiply

double = create_multiplier(2)
double(5)  # 10

# Хранить в коллекциях
operations = {
    "add": lambda a, b: a + b,
    "sub": lambda a, b: a - b,
}
operations["add"](2, 3)  # 5
```

## Рекурсия

Функция вызывает сама себя:

```python
def factorial(n):
    if n <= 1:
        return 1
    return n * factorial(n - 1)

factorial(5)  # 120 = 5 * 4 * 3 * 2 * 1
```

```python
# Числа Фибоначчи
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

fib(10)  # 55
```

**Важно:** Python ограничивает глубину рекурсии (~1000 вызовов).

## Аннотации типов (Type Hints)

```python
def greet(name: str) -> str:
    return f"Hello, {name}"

def add(a: int, b: int) -> int:
    return a + b

def process(items: list[str]) -> dict[str, int]:
    return {item: len(item) for item in items}
```

Аннотации не проверяются Python, но помогают IDE и инструментам (mypy).

## Примеры

### Валидация с ранним выходом
```python
def process_user(user):
    if user is None:
        return None
    if not user.get("active"):
        return None
    if user.get("age", 0) < 18:
        return None

    # Основная логика
    return user["name"].upper()
```

### Функция с настройками
```python
def fetch_data(url, *, timeout=30, retries=3, headers=None):
    if headers is None:
        headers = {}
    # ...
    pass

# Вызов — параметры понятны
fetch_data("http://api.com", timeout=60, retries=5)
```

### Фабричная функция
```python
def create_counter(start=0):
    count = start

    def counter():
        nonlocal count
        count += 1
        return count

    return counter

my_counter = create_counter()
my_counter()  # 1
my_counter()  # 2
my_counter()  # 3
```

---
[prev: 06-exceptions](./06-exceptions.md) | [next: 08-lists-tuples-sets](./08-lists-tuples-sets.md)