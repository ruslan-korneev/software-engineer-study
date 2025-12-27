# Comprehensions (Генераторы списков/словарей/множеств)

## Что такое Comprehension?

**Comprehension** — краткий синтаксис для создания коллекций на основе итерируемых объектов. Это одна из самых "питонических" конструкций языка.

```python
# Без comprehension
squares = []
for x in range(10):
    squares.append(x ** 2)

# С list comprehension
squares = [x ** 2 for x in range(10)]
```

## List Comprehension

### Базовый синтаксис

```python
[выражение for элемент in итерируемое]
```

```python
# Квадраты чисел
[x ** 2 for x in range(5)]  # [0, 1, 4, 9, 16]

# Строки в верхний регистр
names = ['anna', 'bob', 'carl']
[name.upper() for name in names]  # ['ANNA', 'BOB', 'CARL']

# Длины строк
[len(name) for name in names]  # [4, 3, 4]
```

### С условием (фильтрация)

```python
[выражение for элемент in итерируемое if условие]
```

```python
# Только чётные числа
[x for x in range(10) if x % 2 == 0]  # [0, 2, 4, 6, 8]

# Числа больше 5
[x ** 2 for x in range(10) if x > 5]  # [36, 49, 64, 81]

# Непустые строки
words = ['hello', '', 'world', '']
[w for w in words if w]  # ['hello', 'world']
```

### С условием if-else (трансформация)

```python
[выражение_if if условие else выражение_else for элемент in итерируемое]
```

```python
# Чётные → сами, нечётные → 0
[x if x % 2 == 0 else 0 for x in range(6)]  # [0, 0, 2, 0, 4, 0]

# Классификация
['even' if x % 2 == 0 else 'odd' for x in range(5)]
# ['even', 'odd', 'even', 'odd', 'even']
```

### Вложенные циклы

```python
# Декартово произведение
[(x, y) for x in [1, 2] for y in ['a', 'b']]
# [(1, 'a'), (1, 'b'), (2, 'a'), (2, 'b')]

# Эквивалентно:
result = []
for x in [1, 2]:
    for y in ['a', 'b']:
        result.append((x, y))
```

```python
# Сглаживание вложенного списка
matrix = [[1, 2], [3, 4], [5, 6]]
[num for row in matrix for num in row]  # [1, 2, 3, 4, 5, 6]
```

## Dict Comprehension

### Базовый синтаксис

```python
{ключ: значение for элемент in итерируемое}
```

```python
# Число → квадрат
{x: x ** 2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# Из списка кортежей
pairs = [('a', 1), ('b', 2), ('c', 3)]
{k: v for k, v in pairs}  # {'a': 1, 'b': 2, 'c': 3}

# Инверсия словаря
original = {'a': 1, 'b': 2}
{v: k for k, v in original.items()}  # {1: 'a', 2: 'b'}
```

### С условием

```python
# Только чётные значения
{x: x ** 2 for x in range(10) if x % 2 == 0}
# {0: 0, 2: 4, 4: 16, 6: 36, 8: 64}

# Фильтрация по значению
scores = {'alice': 85, 'bob': 60, 'carl': 92}
{name: score for name, score in scores.items() if score >= 80}
# {'alice': 85, 'carl': 92}
```

## Set Comprehension

### Базовый синтаксис

```python
{выражение for элемент in итерируемое}
```

```python
# Уникальные квадраты
{x ** 2 for x in [-2, -1, 0, 1, 2]}  # {0, 1, 4}

# Уникальные первые буквы
names = ['Anna', 'Alice', 'Bob', 'Bella']
{name[0] for name in names}  # {'A', 'B'}

# С условием
{x for x in range(10) if x % 2 == 0}  # {0, 2, 4, 6, 8}
```

## Generator Expression

Ленивая версия list comprehension — не создаёт список в памяти, а генерирует элементы по требованию.

```python
# List comprehension — сразу в памяти
squares_list = [x ** 2 for x in range(1000000)]

# Generator expression — ленивый
squares_gen = (x ** 2 for x in range(1000000))
```

### Когда использовать

```python
# Суммирование — не нужен список
sum(x ** 2 for x in range(100))  # Скобки вокруг генератора не нужны

# Проверка условия
any(x > 100 for x in data)
all(len(s) > 0 for s in strings)

# Итерация один раз
for square in (x ** 2 for x in range(10)):
    print(square)
```

### Отличия от list comprehension

| List Comprehension | Generator Expression |
|--------------------|---------------------|
| `[x for x in ...]` | `(x for x in ...)` |
| Сразу в памяти | Ленивые вычисления |
| Можно итерировать много раз | Только один раз |
| Занимает память | Экономит память |

## Вложенные Comprehensions

### Создание матрицы

```python
# Матрица 3x3
matrix = [[i * 3 + j for j in range(3)] for i in range(3)]
# [[0, 1, 2], [3, 4, 5], [6, 7, 8]]
```

### Транспонирование матрицы

```python
matrix = [
    [1, 2, 3],
    [4, 5, 6],
    [7, 8, 9]
]

transposed = [[row[i] for row in matrix] for i in range(3)]
# [[1, 4, 7], [2, 5, 8], [3, 6, 9]]
```

## Best Practices

### Когда использовать comprehensions

```python
# ✅ Хорошо — простая трансформация
squares = [x ** 2 for x in numbers]

# ✅ Хорошо — простая фильтрация
evens = [x for x in numbers if x % 2 == 0]

# ✅ Хорошо — создание словаря из списка
name_to_length = {name: len(name) for name in names}
```

### Когда НЕ использовать

```python
# ❌ Плохо — слишком сложно
result = [
    transform(x)
    for x in data
    if condition1(x) and condition2(x) or condition3(x)
    for y in process(x)
    if another_condition(y)
]

# ✅ Лучше — обычный цикл
result = []
for x in data:
    if condition1(x) and condition2(x) or condition3(x):
        for y in process(x):
            if another_condition(y):
                result.append(transform(x))
```

```python
# ❌ Плохо — побочные эффекты
[print(x) for x in range(10)]  # Не используйте для side effects!

# ✅ Хорошо
for x in range(10):
    print(x)
```

### Правило читаемости

Если comprehension занимает больше 80 символов или содержит больше 2 условий — используйте обычный цикл.

```python
# Допустимо разбить на несколько строк
result = [
    transform(item)
    for item in collection
    if condition(item)
]
```

## Производительность

### List Comprehension vs цикл

```python
import timeit

# List comprehension быстрее
timeit.timeit('[x ** 2 for x in range(1000)]', number=10000)
# ~0.8 сек

# Обычный цикл медленнее
timeit.timeit('''
result = []
for x in range(1000):
    result.append(x ** 2)
''', number=10000)
# ~1.2 сек
```

### Generator vs List для больших данных

```python
# Память: list — O(n), generator — O(1)

# ❌ Может исчерпать память
sum([x ** 2 for x in range(10_000_000)])

# ✅ Экономит память
sum(x ** 2 for x in range(10_000_000))
```

## Примеры из практики

### Обработка данных

```python
# Извлечение email'ов
users = [{'name': 'Alice', 'email': 'a@x.com'}, {'name': 'Bob', 'email': 'b@x.com'}]
emails = [user['email'] for user in users]

# Фильтрация по полю
active_users = [u for u in users if u.get('active', False)]

# Группировка
from collections import defaultdict
by_domain = defaultdict(list)
for email in emails:
    domain = email.split('@')[1]
    by_domain[domain].append(email)
```

### Работа с файлами

```python
# Чтение непустых строк
with open('file.txt') as f:
    lines = [line.strip() for line in f if line.strip()]

# Парсинг CSV-подобных данных
data = "1,2,3\n4,5,6\n7,8,9"
matrix = [[int(x) for x in line.split(',')] for line in data.split('\n')]
```

### Словари для быстрого поиска

```python
# Индексация по ID
users = [{'id': 1, 'name': 'Alice'}, {'id': 2, 'name': 'Bob'}]
users_by_id = {u['id']: u for u in users}
users_by_id[1]  # {'id': 1, 'name': 'Alice'}
```

## Q&A

**Q: Что быстрее — list comprehension или map()?**
A: Примерно одинаково, но list comprehension читабельнее. Используйте map() когда функция уже существует: `list(map(str, numbers))`.

**Q: Можно ли использовать walrus operator (:=) в comprehensions?**
A: Да, в Python 3.8+:
```python
[y for x in data if (y := expensive_func(x)) > 0]
```

**Q: Как отлаживать сложные comprehensions?**
A: Разбейте на части и проверяйте промежуточные результаты:
```python
# Вместо одного сложного — несколько простых
filtered = [x for x in data if condition(x)]
transformed = [transform(x) for x in filtered]
```
