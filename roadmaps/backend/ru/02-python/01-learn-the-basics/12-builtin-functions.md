# Встроенные функции (Built-in Functions)

[prev: 11-file-io](./11-file-io.md) | [next: 01-arrays-and-linked-lists](../02-data-structures-and-algorithms/01-arrays-and-linked-lists.md)
---

## Обзор

Python имеет множество встроенных функций, доступных без импорта. Здесь рассмотрены функции для работы с итерируемыми объектами.

## map() — применение функции к каждому элементу

```python
map(function, iterable, ...)
```

Возвращает итератор с результатами применения функции к каждому элементу.

```python
# Квадраты чисел
numbers = [1, 2, 3, 4, 5]
squares = list(map(lambda x: x ** 2, numbers))
# [1, 4, 9, 16, 25]

# С именованной функцией
def double(x):
    return x * 2

result = list(map(double, numbers))
# [2, 4, 6, 8, 10]

# Преобразование типов
strings = ['1', '2', '3']
integers = list(map(int, strings))  # [1, 2, 3]

# Несколько итерируемых
a = [1, 2, 3]
b = [10, 20, 30]
sums = list(map(lambda x, y: x + y, a, b))  # [11, 22, 33]
```

### map() vs list comprehension

```python
# map — когда функция уже есть
list(map(str, numbers))
list(map(len, strings))

# comprehension — когда нужно выражение
[x ** 2 for x in numbers]
[s.upper() for s in strings]
```

## filter() — фильтрация элементов

```python
filter(function, iterable)
```

Возвращает итератор с элементами, для которых функция вернула `True`.

```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]

# Чётные числа
evens = list(filter(lambda x: x % 2 == 0, numbers))
# [2, 4, 6, 8, 10]

# Числа больше 5
big = list(filter(lambda x: x > 5, numbers))
# [6, 7, 8, 9, 10]

# Непустые строки (None = использовать bool())
words = ['hello', '', 'world', '', 'python']
non_empty = list(filter(None, words))
# ['hello', 'world', 'python']

# С именованной функцией
def is_positive(x):
    return x > 0

positives = list(filter(is_positive, [-2, -1, 0, 1, 2]))
# [1, 2]
```

### filter() vs list comprehension

```python
# filter
list(filter(lambda x: x > 0, numbers))

# comprehension (обычно предпочтительнее)
[x for x in numbers if x > 0]
```

## reduce() — свёртка в одно значение

```python
from functools import reduce
reduce(function, iterable[, initializer])
```

Применяет функцию кумулятивно к элементам, сводя их к одному значению.

```python
from functools import reduce

numbers = [1, 2, 3, 4, 5]

# Сумма
total = reduce(lambda acc, x: acc + x, numbers)  # 15
# Эквивалентно: ((((1+2)+3)+4)+5)

# Произведение
product = reduce(lambda acc, x: acc * x, numbers)  # 120

# Максимум
maximum = reduce(lambda a, b: a if a > b else b, numbers)  # 5

# С начальным значением
total = reduce(lambda acc, x: acc + x, numbers, 100)  # 115
```

### Когда использовать reduce

```python
# ❌ Используйте встроенные функции
sum(numbers)      # Вместо reduce для суммы
max(numbers)      # Вместо reduce для максимума
min(numbers)      # Вместо reduce для минимума

# ✅ reduce для сложных операций
# Объединение словарей
dicts = [{'a': 1}, {'b': 2}, {'c': 3}]
merged = reduce(lambda acc, d: {**acc, **d}, dicts, {})
# {'a': 1, 'b': 2, 'c': 3}
```

## zip() — объединение итерируемых

```python
zip(*iterables)
```

Создаёт итератор кортежей из элементов с одинаковыми индексами.

```python
names = ['Alice', 'Bob', 'Charlie']
ages = [25, 30, 35]

# Объединение
pairs = list(zip(names, ages))
# [('Alice', 25), ('Bob', 30), ('Charlie', 35)]

# В цикле
for name, age in zip(names, ages):
    print(f'{name} is {age} years old')

# Создание словаря
name_to_age = dict(zip(names, ages))
# {'Alice': 25, 'Bob': 30, 'Charlie': 35}

# Три и более итерируемых
scores = [85, 90, 88]
for name, age, score in zip(names, ages, scores):
    print(f'{name}: {age} years, score {score}')
```

### Разная длина

```python
# zip останавливается на самом коротком
a = [1, 2, 3]
b = [10, 20]
list(zip(a, b))  # [(1, 10), (2, 20)]

# zip_longest — заполняет пропуски
from itertools import zip_longest
list(zip_longest(a, b, fillvalue=0))
# [(1, 10), (2, 20), (3, 0)]
```

### "Распаковка" zip

```python
pairs = [('a', 1), ('b', 2), ('c', 3)]
letters, numbers = zip(*pairs)
# letters = ('a', 'b', 'c')
# numbers = (1, 2, 3)
```

## enumerate() — индекс + элемент

```python
enumerate(iterable, start=0)
```

Возвращает итератор кортежей `(индекс, элемент)`.

```python
fruits = ['apple', 'banana', 'cherry']

# Базовое использование
for i, fruit in enumerate(fruits):
    print(f'{i}: {fruit}')
# 0: apple
# 1: banana
# 2: cherry

# Начать с 1
for i, fruit in enumerate(fruits, start=1):
    print(f'{i}. {fruit}')
# 1. apple
# 2. banana
# 3. cherry

# Преобразование в список
indexed = list(enumerate(fruits))
# [(0, 'apple'), (1, 'banana'), (2, 'cherry')]
```

### Практические примеры

```python
# Найти индекс элемента
for i, item in enumerate(items):
    if condition(item):
        print(f'Found at index {i}')
        break

# Нумерация строк файла
with open('file.txt') as f:
    for line_num, line in enumerate(f, start=1):
        print(f'{line_num}: {line}', end='')
```

## sorted() — сортировка

```python
sorted(iterable, *, key=None, reverse=False)
```

Возвращает новый отсортированный список.

```python
numbers = [3, 1, 4, 1, 5, 9, 2, 6]
sorted(numbers)  # [1, 1, 2, 3, 4, 5, 6, 9]

# Обратный порядок
sorted(numbers, reverse=True)  # [9, 6, 5, 4, 3, 2, 1, 1]

# Строки
words = ['banana', 'Apple', 'cherry']
sorted(words)  # ['Apple', 'banana', 'cherry'] (регистрозависимо!)
sorted(words, key=str.lower)  # ['Apple', 'banana', 'cherry']
```

### Сортировка с key

```python
# По длине
words = ['cat', 'elephant', 'dog', 'bear']
sorted(words, key=len)  # ['cat', 'dog', 'bear', 'elephant']

# По второму элементу кортежа
pairs = [(1, 'b'), (2, 'a'), (3, 'c')]
sorted(pairs, key=lambda x: x[1])  # [(2, 'a'), (1, 'b'), (3, 'c')]

# По атрибуту объекта
from dataclasses import dataclass

@dataclass
class Person:
    name: str
    age: int

people = [Person('Alice', 30), Person('Bob', 25), Person('Charlie', 35)]
sorted(people, key=lambda p: p.age)
# [Person(name='Bob', age=25), Person(name='Alice', age=30), ...]

# operator.attrgetter — более эффективно
from operator import attrgetter
sorted(people, key=attrgetter('age'))
```

### sorted() vs list.sort()

```python
# sorted() — возвращает новый список
original = [3, 1, 2]
new_list = sorted(original)  # original не изменён

# list.sort() — изменяет на месте
original.sort()  # original изменён
```

## reversed() — обратный порядок

```python
reversed(sequence)
```

Возвращает итератор в обратном порядке.

```python
numbers = [1, 2, 3, 4, 5]

# Итерация
for num in reversed(numbers):
    print(num)  # 5, 4, 3, 2, 1

# В список
list(reversed(numbers))  # [5, 4, 3, 2, 1]

# Строка
''.join(reversed('hello'))  # 'olleh'
```

## any() и all() — логические проверки

### any() — хотя бы один True

```python
any(iterable)
```

```python
# Есть хотя бы одно положительное?
numbers = [-1, -2, 3, -4]
any(x > 0 for x in numbers)  # True

# Есть хотя бы одна строка?
items = [0, '', None, 'hello', []]
any(items)  # True (из-за 'hello')

# Пустой итерируемый
any([])  # False
```

### all() — все True

```python
all(iterable)
```

```python
# Все положительные?
numbers = [1, 2, 3, 4]
all(x > 0 for x in numbers)  # True

numbers = [1, 2, -3, 4]
all(x > 0 for x in numbers)  # False

# Все непустые?
items = ['a', 'b', 'c']
all(items)  # True

# Пустой итерируемый
all([])  # True (vacuously true)
```

### Практические примеры

```python
# Валидация
def validate_form(data):
    required_fields = ['name', 'email', 'password']
    return all(field in data for field in required_fields)

# Проверка прав доступа
def has_any_permission(user, permissions):
    return any(p in user.permissions for p in permissions)

# Проверка файлов
from pathlib import Path
files = ['a.txt', 'b.txt', 'c.txt']
all(Path(f).exists() for f in files)  # Все существуют?
```

## sum(), min(), max()

```python
numbers = [1, 2, 3, 4, 5]

sum(numbers)       # 15
sum(numbers, 10)   # 25 (с начальным значением)

min(numbers)       # 1
max(numbers)       # 5

# С key
words = ['cat', 'elephant', 'dog']
min(words, key=len)  # 'cat'
max(words, key=len)  # 'elephant'

# Несколько аргументов
min(3, 1, 4, 1, 5)  # 1
max(3, 1, 4, 1, 5)  # 5
```

## len(), abs(), round()

```python
# len — длина
len([1, 2, 3])     # 3
len('hello')       # 5
len({'a': 1})      # 1

# abs — модуль числа
abs(-5)            # 5
abs(3.14)          # 3.14
abs(-3 + 4j)       # 5.0 (модуль комплексного числа)

# round — округление
round(3.14159, 2)  # 3.14
round(3.5)         # 4 (банковское округление!)
round(2.5)         # 2 (к ближайшему чётному)
```

## Преобразование типов

```python
# В список/кортеж/множество
list('abc')        # ['a', 'b', 'c']
tuple([1, 2, 3])   # (1, 2, 3)
set([1, 2, 2, 3])  # {1, 2, 3}

# В числа
int('42')          # 42
float('3.14')      # 3.14
int('ff', 16)      # 255 (из 16-ричной)

# В строку
str(42)            # '42'
str([1, 2, 3])     # '[1, 2, 3]'

# В bool
bool(0)            # False
bool(1)            # True
bool([])           # False
bool([1])          # True
```

## Комбинирование функций

```python
# Сумма квадратов чётных чисел
numbers = range(1, 11)

# С функциями
sum(map(lambda x: x ** 2, filter(lambda x: x % 2 == 0, numbers)))
# 220

# С comprehension (читабельнее)
sum(x ** 2 for x in numbers if x % 2 == 0)
# 220
```

## Q&A

**Q: map/filter или comprehension?**
A: Comprehension обычно читабельнее. map/filter — когда функция уже существует (`map(str, items)`).

**Q: Почему map/filter возвращают итераторы, а не списки?**
A: Для экономии памяти. Используйте `list()` если нужен список.

**Q: Чем sorted() отличается от sort()?**
A: `sorted()` возвращает новый список, `sort()` изменяет на месте. `sorted()` работает с любым iterable.

**Q: any([]) возвращает False, а all([]) — True. Почему?**
A: `any` ищет хоть один True (не нашёл = False). `all` ищет хоть один False (не нашёл = True). Это "vacuous truth" в логике.

---
[prev: 11-file-io](./11-file-io.md) | [next: 01-arrays-and-linked-lists](../02-data-structures-and-algorithms/01-arrays-and-linked-lists.md)