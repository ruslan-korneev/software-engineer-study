# Словари (Dictionaries)

[prev: 08-lists-tuples-sets](./08-lists-tuples-sets.md) | [next: 10-comprehensions](./10-comprehensions.md)
---

Словарь — коллекция пар **ключ: значение**.

```python
user = {
    "name": "Ruslan",
    "age": 25,
    "city": "Moscow"
}
```

## Особенности

- **Ключи уникальны** — дубликаты перезаписываются
- **Ключи неизменяемые** — str, int, tuple, frozenset (не list, не dict, не set)
- **Порядок сохраняется** (Python 3.7+)
- **Доступ по ключу — O(1)** — очень быстро

## Создание

```python
# Литерал
user = {"name": "Ruslan", "age": 25}

# Конструктор dict()
user = dict(name="Ruslan", age=25)

# Из списка пар (ключ, значение)
user = dict([("name", "Ruslan"), ("age", 25)])

# Из двух списков через zip
keys = ["a", "b", "c"]
values = [1, 2, 3]
d = dict(zip(keys, values))  # {"a": 1, "b": 2, "c": 3}

# Пустой словарь
empty = {}
empty = dict()

# Все ключи с одинаковым значением
d = dict.fromkeys(["a", "b", "c"], 0)  # {"a": 0, "b": 0, "c": 0}
d = dict.fromkeys("abc", 0)            # то же
```

## Доступ к элементам

```python
user = {"name": "Ruslan", "age": 25}

# По ключу — KeyError если нет
user["name"]       # "Ruslan"
user["city"]       # KeyError: 'city'

# Безопасный доступ через get()
user.get("name")         # "Ruslan"
user.get("city")         # None (ключа нет)
user.get("city", "N/A")  # "N/A" — значение по умолчанию
```

## Добавление и изменение

```python
user = {"name": "Ruslan", "age": 25}

# Добавить новый ключ
user["city"] = "Moscow"

# Изменить существующий
user["age"] = 26

# Обновить несколько сразу
user.update({"age": 27, "job": "developer"})
user.update(country="Russia", active=True)

# Объединение словарей (Python 3.9+)
user |= {"age": 28}

# Объединение в новый словарь
a = {"x": 1}
b = {"y": 2}
c = a | b              # {"x": 1, "y": 2} — Python 3.9+
c = {**a, **b}         # {"x": 1, "y": 2} — работает везде

# При конфликте ключей — побеждает правый
{**{"a": 1}, **{"a": 2}}  # {"a": 2}
```

## Удаление

```python
user = {"name": "Ruslan", "age": 25, "city": "Moscow"}

# del — удалить по ключу (KeyError если нет)
del user["city"]

# pop() — удалить и вернуть значение
age = user.pop("age")           # 25, user = {"name": "Ruslan"}
city = user.pop("city", None)   # None — без ошибки если нет

# popitem() — удалить последнюю пару (LIFO, Python 3.7+)
key, value = user.popitem()

# clear() — очистить словарь
user.clear()  # {}
```

## Проверки

```python
user = {"name": "Ruslan", "age": 25}

# Проверка ключа
"name" in user         # True
"city" not in user     # True

# ВАЖНО: in проверяет только ключи!
25 in user             # False
25 in user.values()    # True — так проверить значение

# Проверка пары
("name", "Ruslan") in user.items()  # True
```

## Перебор

```python
user = {"name": "Ruslan", "age": 25, "city": "Moscow"}

# Ключи (по умолчанию)
for key in user:
    print(key)  # name, age, city

for key in user.keys():
    print(key)

# Значения
for value in user.values():
    print(value)  # Ruslan, 25, Moscow

# Ключи и значения вместе
for key, value in user.items():
    print(f"{key}: {value}")
```

## Методы keys(), values(), items()

```python
user = {"name": "Ruslan", "age": 25}

# Возвращают view-объекты (не списки!)
user.keys()    # dict_keys(['name', 'age'])
user.values()  # dict_values(['Ruslan', 25])
user.items()   # dict_items([('name', 'Ruslan'), ('age', 25)])

# Преобразование в список
list(user.keys())  # ['name', 'age']

# View обновляется при изменении словаря
keys = user.keys()
user["city"] = "Moscow"
print(keys)  # dict_keys(['name', 'age', 'city'])
```

## setdefault — получить или установить

```python
user = {"name": "Ruslan"}

# Если ключ есть — возвращает значение
# Если ключа нет — устанавливает и возвращает
user.setdefault("age", 25)   # 25, user теперь {"name": "Ruslan", "age": 25}
user.setdefault("age", 30)   # 25 — ключ уже есть, значение НЕ меняется

# Практическое применение: группировка
words = ["apple", "banana", "apricot", "blueberry"]
by_letter = {}

for word in words:
    letter = word[0]
    by_letter.setdefault(letter, []).append(word)

# {"a": ["apple", "apricot"], "b": ["banana", "blueberry"]}
```

## Dict Comprehension

```python
# Квадраты чисел
squares = {x: x**2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# С условием
evens = {x: x**2 for x in range(10) if x % 2 == 0}
# {0: 0, 2: 4, 4: 16, 6: 36, 8: 64}

# Инвертировать ключи и значения
original = {"a": 1, "b": 2}
inverted = {v: k for k, v in original.items()}
# {1: "a", 2: "b"}

# Фильтрация словаря
user = {"name": "Ruslan", "age": 25, "password": "secret"}
public = {k: v for k, v in user.items() if k != "password"}
# {"name": "Ruslan", "age": 25}

# Преобразование значений
prices = {"apple": 100, "banana": 50}
discounted = {k: v * 0.9 for k, v in prices.items()}
# {"apple": 90.0, "banana": 45.0}
```

## Вложенные словари

```python
users = {
    "user1": {"name": "Ruslan", "age": 25},
    "user2": {"name": "Anna", "age": 22},
}

# Доступ
users["user1"]["name"]   # "Ruslan"

# Изменение
users["user1"]["city"] = "Moscow"

# Добавление нового пользователя
users["user3"] = {"name": "Ivan", "age": 30}

# Безопасный доступ к вложенным
name = users.get("user3", {}).get("name", "Unknown")

# Перебор
for user_id, user_data in users.items():
    print(f"{user_id}: {user_data['name']}")
```

## defaultdict — значение по умолчанию

```python
from collections import defaultdict

# Список по умолчанию
groups = defaultdict(list)
groups["a"].append(1)    # не нужно проверять, есть ли ключ
groups["a"].append(2)
groups["b"].append(3)
# defaultdict(<class 'list'>, {"a": [1, 2], "b": [3]})

# int по умолчанию (счётчик)
counts = defaultdict(int)
for char in "hello":
    counts[char] += 1
# {"h": 1, "e": 1, "l": 2, "o": 1}

# set по умолчанию
tags = defaultdict(set)
tags["python"].add("programming")
tags["python"].add("backend")

# Своя функция
from collections import defaultdict
d = defaultdict(lambda: "N/A")
d["missing"]  # "N/A"
```

## Counter — подсчёт элементов

```python
from collections import Counter

# Подсчёт символов
c = Counter("abracadabra")
# Counter({"a": 5, "b": 2, "r": 2, "c": 1, "d": 1})

c["a"]             # 5
c["z"]             # 0 — не KeyError!
c.most_common(2)   # [("a", 5), ("b", 2)]

# Подсчёт слов
words = ["apple", "banana", "apple", "cherry", "banana", "apple"]
word_counts = Counter(words)
# Counter({"apple": 3, "banana": 2, "cherry": 1})

# Арифметика Counter
c1 = Counter(a=3, b=1)
c2 = Counter(a=1, b=2)
c1 + c2    # Counter({"a": 4, "b": 3})
c1 - c2    # Counter({"a": 2}) — отрицательные удаляются

# Обновление
c1.update(["a", "a", "c"])  # добавить элементы
c1.subtract(["a"])           # вычесть
```

## ChainMap — объединение словарей

```python
from collections import ChainMap

defaults = {"color": "red", "size": "medium"}
user_prefs = {"color": "blue"}

# Объединение с приоритетом
settings = ChainMap(user_prefs, defaults)
settings["color"]  # "blue" — из user_prefs
settings["size"]   # "medium" — из defaults
```

## Копирование словарей

```python
original = {"a": 1, "b": {"c": 2}}

# Поверхностная копия
shallow = original.copy()
shallow = dict(original)

shallow["a"] = 10          # original не изменится
shallow["b"]["c"] = 20     # original["b"]["c"] тоже станет 20!

# Глубокая копия
import copy
deep = copy.deepcopy(original)
deep["b"]["c"] = 200       # original не изменится
```

## Практические примеры

### Подсчёт слов в тексте
```python
text = "hello world hello python world"
words = text.split()
counts = {}

for word in words:
    counts[word] = counts.get(word, 0) + 1

# {"hello": 2, "world": 2, "python": 1}

# Или с Counter
from collections import Counter
counts = Counter(text.split())
```

### Группировка по ключу
```python
users = [
    {"name": "Alice", "city": "Moscow"},
    {"name": "Bob", "city": "SPb"},
    {"name": "Charlie", "city": "Moscow"},
]

by_city = {}
for user in users:
    city = user["city"]
    by_city.setdefault(city, []).append(user["name"])

# {"Moscow": ["Alice", "Charlie"], "SPb": ["Bob"]}
```

### Кэширование результатов
```python
cache = {}

def expensive_function(x):
    if x in cache:
        return cache[x]

    result = x ** 2  # дорогое вычисление
    cache[x] = result
    return result
```

### Маппинг действий
```python
def start():
    print("Starting...")

def stop():
    print("Stopping...")

commands = {
    "start": start,
    "stop": stop,
}

action = input("Command: ")
if action in commands:
    commands[action]()
else:
    print("Unknown command")
```

---
[prev: 08-lists-tuples-sets](./08-lists-tuples-sets.md) | [next: 10-comprehensions](./10-comprehensions.md)