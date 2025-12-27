# Списки, кортежи, множества

Коллекции для хранения нескольких значений.

## Сравнение

| Тип | Синтаксис | Изменяемый | Упорядоченный | Дубликаты | Ключ dict |
|-----|-----------|------------|---------------|-----------|-----------|
| `list` | `[1, 2, 3]` | Да | Да | Да | Нет |
| `tuple` | `(1, 2, 3)` | Нет | Да | Да | Да |
| `set` | `{1, 2, 3}` | Да | Нет | Нет | Нет |
| `frozenset` | `frozenset([1,2])` | Нет | Нет | Нет | Да |

---

## List (список)

Упорядоченная изменяемая коллекция.

### Создание
```python
fruits = ["apple", "banana", "orange"]
numbers = [1, 2, 3, 4, 5]
mixed = [1, "hello", True, 3.14]  # разные типы
empty = []
from_range = list(range(5))  # [0, 1, 2, 3, 4]
```

### Индексация и срезы
```python
lst = ["a", "b", "c", "d", "e"]

# Индексация
lst[0]       # "a" (первый)
lst[1]       # "b"
lst[-1]      # "e" (последний)
lst[-2]      # "d"

# Срезы [start:stop:step]
lst[1:3]     # ["b", "c"] (с 1 до 3, не включая 3)
lst[:3]      # ["a", "b", "c"] (первые 3)
lst[2:]      # ["c", "d", "e"] (с индекса 2 до конца)
lst[::2]     # ["a", "c", "e"] (каждый второй)
lst[::-1]    # ["e", "d", "c", "b", "a"] (реверс)
```

### Изменение элементов
```python
lst = [1, 2, 3]

lst[0] = 10           # [10, 2, 3]
lst[1:3] = [20, 30]   # [10, 20, 30] — замена среза
```

### Добавление элементов
```python
lst = [1, 2, 3]

lst.append(4)         # [1, 2, 3, 4] — в конец
lst.insert(0, 0)      # [0, 1, 2, 3, 4] — по индексу
lst.extend([5, 6])    # [0, 1, 2, 3, 4, 5, 6] — добавить несколько
lst += [7, 8]         # то же, что extend
```

### Удаление элементов
```python
lst = [1, 2, 3, 2, 4, 5]

lst.remove(2)         # [1, 3, 2, 4, 5] — удаляет ПЕРВОЕ вхождение
lst.pop()             # возвращает 5, lst = [1, 3, 2, 4]
lst.pop(0)            # возвращает 1, lst = [3, 2, 4]
del lst[1]            # lst = [3, 4]
lst.clear()           # lst = []
```

### Поиск и подсчёт
```python
lst = [3, 1, 4, 1, 5, 9, 2, 6]

len(lst)              # 8 — длина
lst.index(4)          # 2 — индекс первого вхождения
lst.index(1, 2)       # 3 — искать с индекса 2
lst.count(1)          # 2 — сколько раз встречается

1 in lst              # True
99 not in lst         # True
```

### Сортировка
```python
lst = [3, 1, 4, 1, 5]

# Изменяет исходный список
lst.sort()            # [1, 1, 3, 4, 5]
lst.sort(reverse=True)  # [5, 4, 3, 1, 1]

# Возвращает новый список
lst = [3, 1, 4]
sorted_lst = sorted(lst)  # [1, 3, 4], lst не изменён

# Сортировка по ключу
words = ["banana", "apple", "cherry"]
words.sort(key=len)   # ["apple", "banana", "cherry"]
sorted(words, key=str.lower)  # регистронезависимая
```

### Реверс
```python
lst = [1, 2, 3]

lst.reverse()         # [3, 2, 1] — изменяет список
reversed(lst)         # итератор — не изменяет
list(reversed(lst))   # [1, 2, 3] — как список
lst[::-1]             # [3, 2, 1] — через срез
```

### Копирование
```python
original = [1, 2, 3]

# Поверхностная копия
copy1 = original.copy()
copy2 = original[:]
copy3 = list(original)

# Все три способа создают новый список,
# но вложенные объекты остаются общими!
```

### List Comprehension
```python
# Обычный способ
squares = []
for x in range(5):
    squares.append(x ** 2)

# List comprehension
squares = [x ** 2 for x in range(5)]  # [0, 1, 4, 9, 16]

# С условием
evens = [x for x in range(10) if x % 2 == 0]  # [0, 2, 4, 6, 8]

# С преобразованием
words = ["hello", "world"]
upper = [w.upper() for w in words]  # ["HELLO", "WORLD"]
```

---

## Tuple (кортеж)

Упорядоченная **неизменяемая** коллекция.

### Создание
```python
point = (10, 20)
rgb = (255, 128, 0)
single = (42,)        # одиночный элемент — запятая обязательна!
not_tuple = (42)      # это просто число 42
empty = ()
without_parens = 10, 20, 30  # скобки необязательны
from_list = tuple([1, 2, 3])
```

### Индексация и срезы
```python
t = (1, 2, 3, 4, 5)

t[0]      # 1
t[-1]     # 5
t[1:3]    # (2, 3)

# Изменить нельзя!
# t[0] = 10  # TypeError
```

### Методы
```python
t = (1, 2, 3, 2, 2)

t.count(2)    # 3
t.index(3)    # 2
len(t)        # 5
```

### Распаковка (unpacking)
```python
point = (10, 20, 30)

x, y, z = point
print(x, y, z)  # 10 20 30

# Игнорирование значений
x, _, z = point  # _ — игнорируем y

# Расширенная распаковка
first, *rest = (1, 2, 3, 4, 5)
# first = 1, rest = [2, 3, 4, 5]

first, *middle, last = (1, 2, 3, 4, 5)
# first = 1, middle = [2, 3, 4], last = 5

# Обмен значений
a, b = 1, 2
a, b = b, a  # теперь a=2, b=1
```

### Зачем нужен tuple, если есть list?

1. **Неизменяемость** — защита от случайных изменений
2. **Hashable** — можно использовать как ключ словаря
3. **Быстрее** — немного меньше памяти и быстрее
4. **Семантика** — "это фиксированная структура"

```python
# Ключ словаря — только неизменяемые типы
locations = {
    (55.75, 37.62): "Moscow",
    (59.93, 30.31): "SPb",
}

# Список нельзя использовать как ключ
# {[1, 2]: "value"}  # TypeError: unhashable type: 'list'
```

### Named Tuple
```python
from collections import namedtuple

# Создание типа
Point = namedtuple("Point", ["x", "y"])
User = namedtuple("User", "name age city")  # можно через пробел

# Использование
p = Point(10, 20)
print(p.x, p.y)    # 10 20
print(p[0], p[1])  # 10 20 — как обычный кортеж

user = User("Ruslan", 25, "Moscow")
print(user.name)   # Ruslan

# Преобразование в dict
p._asdict()  # {'x': 10, 'y': 20}
```

---

## Set (множество)

**Неупорядоченная** коллекция **уникальных** элементов.

### Создание
```python
numbers = {1, 2, 3, 4, 5}
unique = {1, 2, 2, 3, 3, 3}  # {1, 2, 3} — дубликаты удаляются
empty = set()  # НЕ {} — это пустой dict!
from_list = set([1, 2, 2, 3])  # {1, 2, 3}
from_string = set("hello")  # {'h', 'e', 'l', 'o'}
```

### Добавление и удаление
```python
s = {1, 2, 3}

s.add(4)          # {1, 2, 3, 4}
s.add(2)          # {1, 2, 3, 4} — уже есть, ничего не происходит

s.remove(2)       # {1, 3, 4} — KeyError если нет
s.discard(99)     # ничего не происходит — без ошибки

s.pop()           # удаляет и возвращает произвольный элемент
s.clear()         # set()
```

### Проверка вхождения — O(1)!
```python
s = {1, 2, 3, 4, 5}

3 in s            # True — очень быстро!
99 not in s       # True

# Для списка это O(n), для set — O(1)
```

### Математические операции
```python
a = {1, 2, 3, 4}
b = {3, 4, 5, 6}

# Объединение (union)
a | b             # {1, 2, 3, 4, 5, 6}
a.union(b)        # то же

# Пересечение (intersection)
a & b             # {3, 4}
a.intersection(b)

# Разность
a - b             # {1, 2}
a.difference(b)
b - a             # {5, 6}

# Симметричная разность (XOR)
a ^ b             # {1, 2, 5, 6}
a.symmetric_difference(b)
```

### Сравнение множеств
```python
a = {1, 2, 3}
b = {1, 2, 3, 4, 5}

a <= b            # True — a подмножество b
a < b             # True — a строгое подмножество b
b >= a            # True — b надмножество a
a.issubset(b)     # True
b.issuperset(a)   # True

a.isdisjoint({4, 5})  # True — нет общих элементов
```

### Изменяющие операции
```python
a = {1, 2, 3}
b = {3, 4, 5}

a.update(b)              # a = {1, 2, 3, 4, 5}
a.intersection_update(b) # a = {3, 4, 5}
a.difference_update(b)   # a = set()
```

### Когда использовать set?

```python
# 1. Удаление дубликатов
lst = [1, 2, 2, 3, 3, 3]
unique = list(set(lst))  # [1, 2, 3] (порядок не гарантирован)

# С сохранением порядка (Python 3.7+)
unique_ordered = list(dict.fromkeys(lst))  # [1, 2, 3]

# 2. Быстрая проверка вхождения
allowed_roles = {"admin", "moderator", "user"}
if user_role in allowed_roles:  # O(1)
    pass

# 3. Поиск общих/уникальных элементов
today = {"alice", "bob", "charlie"}
yesterday = {"bob", "diana"}

new_users = today - yesterday       # {"alice", "charlie"}
returning = today & yesterday       # {"bob"}
all_users = today | yesterday       # {"alice", "bob", "charlie", "diana"}
```

### Frozenset — неизменяемое множество
```python
fs = frozenset([1, 2, 3])

# Нельзя изменить
# fs.add(4)  # AttributeError

# Можно использовать как ключ словаря
cache = {
    frozenset([1, 2]): "result1",
    frozenset([3, 4]): "result2",
}

# Можно использовать во множестве
sets = {frozenset([1, 2]), frozenset([3, 4])}
```

---

## Общие операции для коллекций

```python
collection = [1, 2, 3, 4, 5]  # или tuple, set

len(collection)       # 5 — длина
min(collection)       # 1 — минимум
max(collection)       # 5 — максимум
sum(collection)       # 15 — сумма
sorted(collection)    # [1, 2, 3, 4, 5] — отсортированный список

any([False, False, True])   # True — хоть один True
all([True, True, False])    # False — все должны быть True

list(collection)      # преобразование в список
tuple(collection)     # в кортеж
set(collection)       # в множество
```

---

## Копирование — поверхностная vs глубокая

```python
# Поверхностная копия — вложенные объекты общие
original = [1, 2, [3, 4]]
shallow = original.copy()  # или original[:] или list(original)

shallow[0] = 10       # original не изменится
shallow[2][0] = 30    # original[2][0] тоже станет 30!

# Глубокая копия — полностью независимая
import copy
deep = copy.deepcopy(original)
deep[2][0] = 300      # original не изменится
```

---

## Производительность

| Операция | list | set |
|----------|------|-----|
| `x in collection` | O(n) | O(1) |
| `append` / `add` | O(1) | O(1) |
| `insert(0, x)` | O(n) | — |
| `pop()` | O(1) | O(1) |
| `pop(0)` | O(n) | — |

Используй `set` для частых проверок вхождения!
