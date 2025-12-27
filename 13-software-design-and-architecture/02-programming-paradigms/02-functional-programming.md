# Функциональное программирование

## Что такое функциональное программирование?

**Функциональное программирование (ФП)** — это парадигма программирования, в которой программы строятся путём применения и композиции функций. ФП рассматривает вычисления как оценку математических функций и избегает изменяемого состояния и побочных эффектов.

## Основные концепции

### 1. Чистые функции (Pure Functions)

Чистая функция:
- Всегда возвращает одинаковый результат для одинаковых аргументов
- Не имеет побочных эффектов (не изменяет внешнее состояние)

```python
# Чистая функция
def add(a, b):
    return a + b

# Нечистая функция (зависит от внешнего состояния)
total = 0
def add_to_total(value):
    global total
    total += value  # Побочный эффект!
    return total

# Нечистая функция (недетерминированная)
import random
def get_random_number():
    return random.randint(1, 100)  # Разный результат при каждом вызове
```

### 2. Иммутабельность (Immutability)

Данные не изменяются после создания. Вместо изменения создаётся новая копия:

```python
# Мутабельный подход (избегать в ФП)
def add_item_mutable(items, item):
    items.append(item)  # Изменяет оригинальный список
    return items

# Иммутабельный подход
def add_item_immutable(items, item):
    return [*items, item]  # Создаёт новый список

# Пример с словарями
def update_user_mutable(user, key, value):
    user[key] = value  # Изменяет оригинал
    return user

def update_user_immutable(user, key, value):
    return {**user, key: value}  # Создаёт новый словарь
```

### 3. Функции первого класса (First-Class Functions)

Функции можно присваивать переменным, передавать как аргументы и возвращать из других функций:

```python
# Функция как переменная
greet = lambda name: f"Привет, {name}!"
print(greet("Мир"))  # Привет, Мир!

# Функция как аргумент
def apply_operation(x, y, operation):
    return operation(x, y)

result = apply_operation(5, 3, lambda a, b: a + b)  # 8

# Функция как возвращаемое значение
def create_multiplier(factor):
    def multiplier(x):
        return x * factor
    return multiplier

double = create_multiplier(2)
triple = create_multiplier(3)

print(double(5))  # 10
print(triple(5))  # 15
```

### 4. Функции высшего порядка (Higher-Order Functions)

Функции, которые принимают другие функции как аргументы или возвращают функции:

```python
# map - применяет функцию к каждому элементу
numbers = [1, 2, 3, 4, 5]
squared = list(map(lambda x: x ** 2, numbers))  # [1, 4, 9, 16, 25]

# filter - отбирает элементы по условию
evens = list(filter(lambda x: x % 2 == 0, numbers))  # [2, 4]

# reduce - сворачивает последовательность в одно значение
from functools import reduce
total = reduce(lambda acc, x: acc + x, numbers, 0)  # 15
product = reduce(lambda acc, x: acc * x, numbers, 1)  # 120
```

### 5. Замыкания (Closures)

Функция, которая "запоминает" окружение, в котором была создана:

```python
def create_counter():
    count = 0
    def counter():
        nonlocal count
        count += 1
        return count
    return counter

counter1 = create_counter()
counter2 = create_counter()

print(counter1())  # 1
print(counter1())  # 2
print(counter2())  # 1 (независимый счётчик)

# Практический пример: кэширование
def create_cached_function(func):
    cache = {}
    def cached(*args):
        if args not in cache:
            cache[args] = func(*args)
        return cache[args]
    return cached

@create_cached_function
def expensive_calculation(n):
    print(f"Вычисляем для {n}...")
    return n ** 2
```

### 6. Композиция функций (Function Composition)

Объединение простых функций для создания более сложных:

```python
# Ручная композиция
def compose(f, g):
    return lambda x: f(g(x))

def add_one(x):
    return x + 1

def double(x):
    return x * 2

add_one_then_double = compose(double, add_one)
print(add_one_then_double(5))  # (5 + 1) * 2 = 12

# Более сложная композиция с pipe
from functools import reduce

def pipe(*functions):
    def piped(value):
        return reduce(lambda acc, func: func(acc), functions, value)
    return piped

process = pipe(
    str.strip,
    str.lower,
    lambda s: s.replace(" ", "_")
)

print(process("  Hello World  "))  # "hello_world"
```

### 7. Каррирование (Currying)

Преобразование функции с несколькими аргументами в цепочку функций с одним аргументом:

```python
# Обычная функция
def add(a, b, c):
    return a + b + c

# Каррированная версия
def add_curried(a):
    def add_b(b):
        def add_c(c):
            return a + b + c
        return add_c
    return add_b

result = add_curried(1)(2)(3)  # 6

# Частичное применение с functools.partial
from functools import partial

def power(base, exponent):
    return base ** exponent

square = partial(power, exponent=2)
cube = partial(power, exponent=3)

print(square(5))  # 25
print(cube(5))    # 125
```

### 8. Рекурсия

В ФП рекурсия часто заменяет циклы:

```python
# Итеративный подход
def factorial_iterative(n):
    result = 1
    for i in range(1, n + 1):
        result *= i
    return result

# Рекурсивный подход
def factorial_recursive(n):
    if n <= 1:
        return 1
    return n * factorial_recursive(n - 1)

# Хвостовая рекурсия (Python не оптимизирует, но концептуально важно)
def factorial_tail(n, accumulator=1):
    if n <= 1:
        return accumulator
    return factorial_tail(n - 1, n * accumulator)
```

## Практические примеры

### Обработка данных в функциональном стиле

```python
# Данные о пользователях
users = [
    {"name": "Alice", "age": 30, "active": True, "balance": 1000},
    {"name": "Bob", "age": 25, "active": False, "balance": 500},
    {"name": "Charlie", "age": 35, "active": True, "balance": 1500},
    {"name": "Diana", "age": 28, "active": True, "balance": 200},
]

# Императивный подход
def get_active_users_total_imperative(users):
    total = 0
    for user in users:
        if user["active"] and user["age"] >= 28:
            total += user["balance"]
    return total

# Функциональный подход
def get_active_users_total_functional(users):
    return reduce(
        lambda acc, user: acc + user["balance"],
        filter(
            lambda u: u["active"] and u["age"] >= 28,
            users
        ),
        0
    )

# Ещё более читаемый ФП подход с промежуточными функциями
is_active = lambda u: u["active"]
is_adult = lambda u: u["age"] >= 28
get_balance = lambda u: u["balance"]

def get_active_users_total_clean(users):
    active_adults = filter(lambda u: is_active(u) and is_adult(u), users)
    balances = map(get_balance, active_adults)
    return sum(balances)
```

### Валидация с использованием ФП

```python
from typing import Callable, List, Tuple, Optional

# Валидатор - функция, возвращающая (успех, сообщение_об_ошибке)
Validator = Callable[[str], Tuple[bool, Optional[str]]]

def not_empty(value: str) -> Tuple[bool, Optional[str]]:
    return (True, None) if value else (False, "Значение не может быть пустым")

def min_length(length: int) -> Validator:
    def validator(value: str) -> Tuple[bool, Optional[str]]:
        if len(value) >= length:
            return (True, None)
        return (False, f"Минимальная длина: {length}")
    return validator

def matches_pattern(pattern: str, message: str) -> Validator:
    import re
    def validator(value: str) -> Tuple[bool, Optional[str]]:
        if re.match(pattern, value):
            return (True, None)
        return (False, message)
    return validator

def validate_all(validators: List[Validator], value: str) -> List[str]:
    """Применяет все валидаторы и возвращает список ошибок"""
    errors = []
    for validator in validators:
        success, error = validator(value)
        if not success:
            errors.append(error)
    return errors

# Использование
email_validators = [
    not_empty,
    min_length(5),
    matches_pattern(r'^[\w\.-]+@[\w\.-]+\.\w+$', "Неверный формат email")
]

errors = validate_all(email_validators, "test")
print(errors)  # ['Минимальная длина: 5', 'Неверный формат email']
```

### Построение пайплайна обработки

```python
from typing import TypeVar, Callable, List
from functools import reduce

T = TypeVar('T')

class Pipeline:
    def __init__(self, value):
        self.value = value

    def pipe(self, func):
        return Pipeline(func(self.value))

    def result(self):
        return self.value

# Использование
result = (
    Pipeline("  Hello, World!  ")
    .pipe(str.strip)
    .pipe(str.lower)
    .pipe(lambda s: s.replace(",", ""))
    .pipe(lambda s: s.split())
    .pipe(lambda words: [w.capitalize() for w in words])
    .pipe(lambda words: " ".join(words))
    .result()
)

print(result)  # "Hello World!"
```

## Best Practices

### 1. Предпочитайте чистые функции

```python
# Плохо: функция с побочным эффектом
log_entries = []

def process_and_log(data):
    result = data.upper()
    log_entries.append(result)  # Побочный эффект!
    return result

# Хорошо: чистая функция + отдельное логирование
def process_data(data):
    return data.upper()

def log_entry(entry, log):
    return [*log, entry]
```

### 2. Избегайте изменения аргументов

```python
# Плохо
def sort_and_filter(items):
    items.sort()  # Изменяет оригинал!
    return [x for x in items if x > 0]

# Хорошо
def sort_and_filter(items):
    sorted_items = sorted(items)  # Создаёт новый список
    return [x for x in sorted_items if x > 0]
```

### 3. Используйте list/dict comprehensions

```python
# Менее функциональный подход
result = []
for x in range(10):
    if x % 2 == 0:
        result.append(x ** 2)

# Функциональный подход
result = [x ** 2 for x in range(10) if x % 2 == 0]

# Словарь
users_by_id = {user["id"]: user for user in users}
```

### 4. Используйте генераторы для ленивых вычислений

```python
# Ленивая обработка больших данных
def process_large_file(filename):
    with open(filename) as f:
        lines = (line.strip() for line in f)  # Генератор
        non_empty = (line for line in lines if line)
        parsed = (parse_line(line) for line in non_empty)
        yield from parsed
```

## Типичные ошибки

### 1. Скрытая мутация

```python
# Плохо: неочевидная мутация через метод
def add_default_values(config):
    config.setdefault("timeout", 30)  # Изменяет оригинал!
    return config

# Хорошо
def add_default_values(config):
    return {"timeout": 30, **config}
```

### 2. Игнорирование ленивости

```python
# Ошибка: map возвращает итератор, не список
numbers = map(lambda x: x * 2, [1, 2, 3])
print(numbers[0])  # TypeError!

# Правильно
numbers = list(map(lambda x: x * 2, [1, 2, 3]))
print(numbers[0])  # 2
```

### 3. Избыточное усложнение

```python
# Слишком сложно
result = reduce(
    lambda acc, x: acc + [x ** 2],
    filter(lambda x: x % 2 == 0, range(10)),
    []
)

# Просто и читаемо
result = [x ** 2 for x in range(10) if x % 2 == 0]
```

## Библиотеки для ФП в Python

### functools

```python
from functools import reduce, partial, lru_cache

# lru_cache - мемоизация
@lru_cache(maxsize=100)
def fibonacci(n):
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)
```

### itertools

```python
from itertools import chain, groupby, takewhile, dropwhile

# chain - объединение итераторов
all_items = list(chain([1, 2], [3, 4], [5, 6]))  # [1, 2, 3, 4, 5, 6]

# groupby - группировка
data = [("A", 1), ("A", 2), ("B", 3), ("B", 4)]
for key, group in groupby(data, key=lambda x: x[0]):
    print(key, list(group))
```

### toolz / funcy

```python
# Внешние библиотеки для расширенного ФП
# pip install toolz

from toolz import pipe, curry, compose

@curry
def add(x, y):
    return x + y

add_five = add(5)
print(add_five(10))  # 15

result = pipe(
    [1, 2, 3, 4, 5],
    lambda x: map(lambda n: n * 2, x),
    list,
    sum
)
```

## Преимущества ФП

1. **Предсказуемость** — чистые функции легко понять и тестировать
2. **Параллелизм** — отсутствие общего состояния упрощает параллельное выполнение
3. **Отладка** — легко отследить поток данных
4. **Повторное использование** — маленькие функции легко комбинировать
5. **Математическая основа** — формальная верифицируемость

## Заключение

Функциональное программирование — мощная парадигма, которая помогает писать более чистый и надёжный код. Даже если вы не используете чисто функциональный язык, принципы ФП (чистые функции, иммутабельность, композиция) значительно улучшают качество кода в любом языке программирования.
