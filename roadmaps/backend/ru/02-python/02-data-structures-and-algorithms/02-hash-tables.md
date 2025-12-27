# Хэш-таблицы (Hash Tables)

Структура данных для хранения пар **ключ-значение** с быстрым доступом.

## Идея

```
Ключ  →  hash(ключ)  →  индекс  →  значение

"name" →  hash("name") = 3829...  →  index = 42  →  "Ruslan"
```

**Хэш-функция** превращает ключ в число, которое используется как индекс массива.

## Почему это быстро?

```
Массив/Список:  O(n) — нужно перебрать все элементы
Хэш-таблица:    O(1) — сразу вычисляем индекс по ключу
```

```python
# Поиск в списке — O(n)
for item in items:
    if item.key == "name":
        return item.value

# Поиск в хэш-таблице — O(1)
return table[hash("name") % size]
```

## Как работает hash()

```python
hash("hello")       # 3829830242315480425 (большое число)
hash(42)            # 42
hash((1, 2, 3))     # 529344067295497451
hash(3.14)          # 322818021289917443

# Один и тот же объект — всегда одинаковый хэш
hash("test") == hash("test")  # True
```

### Что можно хэшировать?

```python
# Неизменяемые типы — можно
hash("string")      # OK
hash(42)            # OK
hash((1, 2, 3))     # OK — tuple
hash(frozenset([1, 2]))  # OK

# Изменяемые — нельзя
hash([1, 2, 3])     # TypeError: unhashable type: 'list'
hash({1, 2, 3})     # TypeError: unhashable type: 'set'
hash({"a": 1})      # TypeError: unhashable type: 'dict'
```

**Почему?** Если объект изменится, его хэш изменится, и мы не найдём его в таблице.

## Структура хэш-таблицы

```
Индекс:  0      1      2      3      4      5
       ┌──────┬──────┬──────┬──────┬──────┬──────┐
       │      │ name │      │ age  │      │ city │
       │      │Ruslan│      │  25  │      │Moscow│
       └──────┴──────┴──────┴──────┴──────┴──────┘

hash("name") % 6 = 1
hash("age") % 6 = 3
hash("city") % 6 = 5
```

## Коллизии

Разные ключи могут дать одинаковый индекс:

```python
hash("abc") % 10  # = 3
hash("xyz") % 10  # = 3  ← коллизия!
```

### Решение 1: Цепочки (Chaining)

В каждой ячейке хранится список пар:

```
bucket[3] → [("abc", 1), ("xyz", 2)]
```

```python
# Поиск: идём по списку в bucket
for key, value in bucket[index]:
    if key == target_key:
        return value
```

### Решение 2: Открытая адресация

При коллизии ищем следующую свободную ячейку:

```
hash("abc") = 3  → bucket[3] занят
                 → проверяем bucket[4]
                 → проверяем bucket[5] — свободен!
```

Python dict использует **открытую адресацию** с оптимизациями.

## Python dict — это хэш-таблица

```python
d = {"name": "Ruslan", "age": 25}

# Все операции O(1) в среднем
d["name"]           # получить — O(1)
d["city"] = "Msk"   # добавить — O(1)
del d["age"]        # удалить — O(1)
"name" in d         # проверить — O(1)
len(d)              # размер — O(1)
```

### Особенности Python dict

- **Сохраняет порядок** вставки (с Python 3.7)
- **Автоматически расширяется** при заполнении
- **Оптимизирован** для строковых ключей
- **Load factor** ~2/3 (расширяется когда заполнен на 66%)

## Реализация простой хэш-таблицы

```python
class HashTable:
    def __init__(self, size=10):
        self.size = size
        self.buckets = [[] for _ in range(size)]  # список списков
        self.count = 0

    def _hash(self, key):
        """Вычислить индекс для ключа"""
        return hash(key) % self.size

    def set(self, key, value):
        """Добавить или обновить пару ключ-значение"""
        index = self._hash(key)
        bucket = self.buckets[index]

        # Проверяем, есть ли уже такой ключ
        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket[i] = (key, value)  # обновить
                return

        # Ключа нет — добавляем
        bucket.append((key, value))
        self.count += 1

    def get(self, key):
        """Получить значение по ключу"""
        index = self._hash(key)
        bucket = self.buckets[index]

        for k, v in bucket:
            if k == key:
                return v

        raise KeyError(key)

    def delete(self, key):
        """Удалить пару по ключу"""
        index = self._hash(key)
        bucket = self.buckets[index]

        for i, (k, v) in enumerate(bucket):
            if k == key:
                del bucket[i]
                self.count -= 1
                return

        raise KeyError(key)

    def contains(self, key):
        """Проверить наличие ключа"""
        try:
            self.get(key)
            return True
        except KeyError:
            return False

    def __repr__(self):
        items = []
        for bucket in self.buckets:
            items.extend(bucket)
        return f"HashTable({dict(items)})"
```

### Использование

```python
ht = HashTable()
ht.set("name", "Ruslan")
ht.set("age", 25)
ht.set("city", "Moscow")

print(ht.get("name"))     # Ruslan
print(ht.contains("age")) # True

ht.set("age", 26)         # обновить
print(ht.get("age"))      # 26

ht.delete("city")
print(ht)                 # HashTable({'name': 'Ruslan', 'age': 26})
```

## Сложность операций

| Операция | Среднее | Худшее* |
|----------|---------|---------|
| Поиск | **O(1)** | O(n) |
| Вставка | **O(1)** | O(n) |
| Удаление | **O(1)** | O(n) |
| Итерация | O(n) | O(n) |

*Худший случай — все ключи попали в один bucket (все коллизии).

При хорошей хэш-функции и правильном размере таблицы коллизии редки.

## set — хэш-таблица без значений

```python
s = {1, 2, 3, 4, 5}

# O(1) проверка вхождения
5 in s      # True — мгновенно
100 in s    # False — мгновенно

# По сути это dict, где значения не важны
# set ≈ {key: None, key: None, ...}
```

## Практические применения

### 1. Быстрый поиск
```python
# O(n) — плохо
def find_in_list(lst, target):
    return target in lst  # перебирает все элементы

# O(1) — хорошо
def find_in_set(s, target):
    return target in s    # вычисляет хэш
```

### 2. Удаление дубликатов
```python
lst = [1, 2, 2, 3, 3, 3]
unique = list(set(lst))  # [1, 2, 3]
```

### 3. Подсчёт элементов
```python
text = "hello world"
counts = {}

for char in text:
    counts[char] = counts.get(char, 0) + 1

# {'h': 1, 'e': 1, 'l': 3, 'o': 2, ' ': 1, 'w': 1, 'r': 1, 'd': 1}

# Или с Counter
from collections import Counter
counts = Counter(text)
```

### 4. Группировка
```python
from collections import defaultdict

words = ["apple", "banana", "apricot", "blueberry", "avocado"]
by_letter = defaultdict(list)

for word in words:
    by_letter[word[0]].append(word)

# {'a': ['apple', 'apricot', 'avocado'], 'b': ['banana', 'blueberry']}
```

### 5. Кэширование
```python
cache = {}

def expensive_computation(x):
    if x in cache:
        return cache[x]

    result = x ** 2  # дорогое вычисление
    cache[x] = result
    return result
```

### 6. Два указателя с хэш-таблицей
```python
def two_sum(nums, target):
    """Найти два числа, дающих target"""
    seen = {}

    for i, num in enumerate(nums):
        complement = target - num
        if complement in seen:
            return [seen[complement], i]
        seen[num] = i

    return []

two_sum([2, 7, 11, 15], 9)  # [0, 1] — nums[0] + nums[1] = 9
```

## Требования к хэш-функции

Хорошая хэш-функция должна:

1. **Детерминированность** — одинаковый вход → одинаковый выход
2. **Равномерность** — распределять ключи равномерно
3. **Скорость** — вычисляться быстро
4. **Лавинный эффект** — малое изменение входа → большое изменение хэша

```python
# Плохая хэш-функция
def bad_hash(s):
    return len(s)  # много коллизий!

# "ab", "cd", "xy" — все дадут 2
```

## Когда использовать хэш-таблицу?

| Задача | Используй |
|--------|-----------|
| Быстрый поиск по ключу | `dict` |
| Подсчёт элементов | `Counter` |
| Уникальные значения | `set` |
| Кэширование результатов | `dict` или `@lru_cache` |
| Группировка по ключу | `defaultdict(list)` |
| Проверка "уже видели?" | `set` |
