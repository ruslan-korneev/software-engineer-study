# Операции хэш-таблицы (Hash Table Operations)

## Определение

**Хэш-таблица** — это структура данных, реализующая ассоциативный массив (отображение ключ → значение). Обеспечивает O(1) в среднем для операций поиска, вставки и удаления.

```
┌─────────────────────────────────────────────────┐
│                 Хэш-таблица                     │
├─────────────────────────────────────────────────┤
│  Ключ    │ hash(key) │ Индекс │ Значение       │
├──────────┼───────────┼────────┼────────────────┤
│ "apple"  │ 284937483 │   3    │ 1.50           │
│ "banana" │ 982374651 │   7    │ 0.75           │
│ "cherry" │ 123456789 │   2    │ 2.00           │
└─────────────────────────────────────────────────┘
```

## Основные операции

### 1. Вставка (Insert / Put)

```
put("apple", 1.50)

Шаг 1: Вычисляем хэш
  hash("apple") = 284937483

Шаг 2: Определяем индекс
  index = 284937483 % table_size = 3

Шаг 3: Сохраняем в ячейку
  table[3] = ("apple", 1.50)
```

```python
def put(self, key, value):
    """
    Вставка или обновление элемента.
    Время: O(1) в среднем, O(n) в худшем случае
    """
    # Проверяем load factor
    if self.size >= self.capacity * self.load_factor_threshold:
        self._resize()

    index = self._hash(key)

    # Метод цепочек
    for i, (k, v) in enumerate(self.buckets[index]):
        if k == key:
            self.buckets[index][i] = (key, value)  # обновляем
            return

    self.buckets[index].append((key, value))  # добавляем
    self.size += 1
```

### 2. Поиск (Search / Get)

```
get("banana")

Шаг 1: Вычисляем хэш
  hash("banana") = 982374651

Шаг 2: Определяем индекс
  index = 982374651 % table_size = 7

Шаг 3: Ищем в ячейке (или цепочке)
  table[7] содержит ("banana", 0.75) → возвращаем 0.75
```

```python
def get(self, key):
    """
    Поиск значения по ключу.
    Время: O(1) в среднем, O(n) в худшем случае
    """
    index = self._hash(key)

    for k, v in self.buckets[index]:
        if k == key:
            return v

    raise KeyError(key)

def contains(self, key):
    """Проверка наличия ключа"""
    try:
        self.get(key)
        return True
    except KeyError:
        return False
```

### 3. Удаление (Delete / Remove)

```
remove("cherry")

Шаг 1: Вычисляем индекс
  index = hash("cherry") % table_size = 2

Шаг 2: Находим и удаляем
  table[2]: [("cherry", 2.00)] → []
```

```python
def remove(self, key):
    """
    Удаление элемента.
    Время: O(1) в среднем
    """
    index = self._hash(key)

    for i, (k, v) in enumerate(self.buckets[index]):
        if k == key:
            del self.buckets[index][i]
            self.size -= 1
            return v

    raise KeyError(key)
```

## Полная реализация

```python
class HashTable:
    """Хэш-таблица с методом цепочек"""

    def __init__(self, capacity=16, load_factor=0.75):
        self.capacity = capacity
        self.load_factor_threshold = load_factor
        self.size = 0
        self.buckets = [[] for _ in range(capacity)]

    def _hash(self, key):
        """Вычисление индекса"""
        return hash(key) % self.capacity

    def put(self, key, value):
        """Вставка/обновление"""
        if self.size >= self.capacity * self.load_factor_threshold:
            self._resize()

        index = self._hash(key)

        for i, (k, v) in enumerate(self.buckets[index]):
            if k == key:
                self.buckets[index][i] = (key, value)
                return

        self.buckets[index].append((key, value))
        self.size += 1

    def get(self, key, default=None):
        """Получение значения"""
        index = self._hash(key)

        for k, v in self.buckets[index]:
            if k == key:
                return v

        if default is not None:
            return default
        raise KeyError(key)

    def remove(self, key):
        """Удаление"""
        index = self._hash(key)

        for i, (k, v) in enumerate(self.buckets[index]):
            if k == key:
                del self.buckets[index][i]
                self.size -= 1
                return v

        raise KeyError(key)

    def __contains__(self, key):
        """Оператор in"""
        try:
            self.get(key)
            return True
        except KeyError:
            return False

    def __getitem__(self, key):
        """Оператор []"""
        return self.get(key)

    def __setitem__(self, key, value):
        """Оператор []="""
        self.put(key, value)

    def __delitem__(self, key):
        """Оператор del"""
        self.remove(key)

    def __len__(self):
        return self.size

    def __iter__(self):
        """Итерация по ключам"""
        for bucket in self.buckets:
            for key, value in bucket:
                yield key

    def keys(self):
        """Все ключи"""
        return list(self)

    def values(self):
        """Все значения"""
        result = []
        for bucket in self.buckets:
            for key, value in bucket:
                result.append(value)
        return result

    def items(self):
        """Все пары (ключ, значение)"""
        result = []
        for bucket in self.buckets:
            for key, value in bucket:
                result.append((key, value))
        return result

    def _resize(self):
        """Увеличение размера таблицы"""
        old_buckets = self.buckets
        self.capacity *= 2
        self.buckets = [[] for _ in range(self.capacity)]
        self.size = 0

        for bucket in old_buckets:
            for key, value in bucket:
                self.put(key, value)

    def clear(self):
        """Очистка таблицы"""
        self.buckets = [[] for _ in range(self.capacity)]
        self.size = 0
```

## Анализ сложности

| Операция | Среднее | Худшее | Примечание |
|----------|---------|--------|------------|
| put | O(1) | O(n) | При многих коллизиях |
| get | O(1) | O(n) | При многих коллизиях |
| remove | O(1) | O(n) | При многих коллизиях |
| contains | O(1) | O(n) | |
| resize | O(n) | O(n) | Амортизированно O(1) |
| keys/values/items | O(n) | O(n) | Обход всей таблицы |

### Условия для O(1)

```
Для достижения O(1):
1. Хорошая хэш-функция (равномерное распределение)
2. Load factor < 0.75
3. Своевременное resize

Когда O(n):
1. Все ключи имеют одинаковый хэш (HashDoS атака)
2. Очень высокий load factor
3. Плохая хэш-функция
```

## Дополнительные операции

### setdefault — вставка с значением по умолчанию

```python
def setdefault(self, key, default=None):
    """
    Если ключ существует — возвращает значение.
    Если нет — вставляет default и возвращает его.
    """
    try:
        return self.get(key)
    except KeyError:
        self.put(key, default)
        return default

# Использование:
table.setdefault("count", 0)
table["count"] += 1
```

### update — слияние с другой таблицей

```python
def update(self, other):
    """Добавляет все элементы из other"""
    if isinstance(other, dict):
        for key, value in other.items():
            self.put(key, value)
    elif hasattr(other, 'items'):
        for key, value in other.items():
            self.put(key, value)
    else:
        for key, value in other:
            self.put(key, value)

# Использование:
table.update({"a": 1, "b": 2})
table.update([("c", 3), ("d", 4)])
```

### pop — удаление с возвратом значения

```python
def pop(self, key, default=None):
    """Удаляет и возвращает значение"""
    try:
        return self.remove(key)
    except KeyError:
        if default is not None:
            return default
        raise
```

### fromkeys — создание из ключей

```python
@classmethod
def fromkeys(cls, keys, value=None):
    """Создаёт таблицу из последовательности ключей"""
    table = cls()
    for key in keys:
        table.put(key, value)
    return table

# Использование:
table = HashTable.fromkeys(["a", "b", "c"], 0)
# {"a": 0, "b": 0, "c": 0}
```

## Примеры использования

### Пример 1: Подсчёт частоты слов

```python
def word_frequency(text):
    """Подсчитывает частоту слов в тексте"""
    freq = HashTable()

    for word in text.lower().split():
        word = word.strip(".,!?;:")
        count = freq.get(word, 0)
        freq.put(word, count + 1)

    return freq

# Использование:
text = "the quick brown fox jumps over the lazy dog"
freq = word_frequency(text)
print(freq.get("the"))  # 2
```

### Пример 2: Кэш с TTL

```python
import time

class TTLCache:
    """Кэш с временем жизни записей"""

    def __init__(self, ttl_seconds=300):
        self.ttl = ttl_seconds
        self.cache = HashTable()
        self.timestamps = HashTable()

    def put(self, key, value):
        self.cache.put(key, value)
        self.timestamps.put(key, time.time())

    def get(self, key):
        if key not in self.cache:
            raise KeyError(key)

        timestamp = self.timestamps.get(key)
        if time.time() - timestamp > self.ttl:
            # Запись устарела
            self.cache.remove(key)
            self.timestamps.remove(key)
            raise KeyError(key)

        return self.cache.get(key)

    def cleanup(self):
        """Удаляет устаревшие записи"""
        current_time = time.time()
        expired = []

        for key in self.cache:
            if current_time - self.timestamps.get(key) > self.ttl:
                expired.append(key)

        for key in expired:
            self.cache.remove(key)
            self.timestamps.remove(key)
```

### Пример 3: Группировка по ключу

```python
def group_by(items, key_func):
    """
    Группирует элементы по ключу.

    group_by([1, 2, 3, 4, 5], lambda x: x % 2)
    → {0: [2, 4], 1: [1, 3, 5]}
    """
    groups = HashTable()

    for item in items:
        key = key_func(item)
        group = groups.get(key, [])
        group.append(item)
        groups.put(key, group)

    return groups

# Пример: группировка слов по первой букве
words = ["apple", "banana", "apricot", "blueberry", "cherry"]
by_letter = group_by(words, lambda w: w[0])
# {"a": ["apple", "apricot"], "b": ["banana", "blueberry"], "c": ["cherry"]}
```

### Пример 4: Two Sum (классическая задача)

```python
def two_sum(nums, target):
    """
    Находит два числа, дающих в сумме target.
    Возвращает их индексы.

    two_sum([2, 7, 11, 15], 9) → [0, 1]
    """
    seen = HashTable()

    for i, num in enumerate(nums):
        complement = target - num

        if complement in seen:
            return [seen.get(complement), i]

        seen.put(num, i)

    return None
```

### Пример 5: LRU Cache с хэш-таблицей

```python
class LRUCache:
    """
    LRU Cache: комбинация хэш-таблицы и двусвязного списка.
    Все операции O(1).
    """

    class Node:
        def __init__(self, key, value):
            self.key = key
            self.value = value
            self.prev = None
            self.next = None

    def __init__(self, capacity):
        self.capacity = capacity
        self.cache = HashTable()  # key → Node

        # Фиктивные узлы для упрощения
        self.head = self.Node(0, 0)
        self.tail = self.Node(0, 0)
        self.head.next = self.tail
        self.tail.prev = self.head

    def _add_to_front(self, node):
        """Добавляет узел сразу после head"""
        node.prev = self.head
        node.next = self.head.next
        self.head.next.prev = node
        self.head.next = node

    def _remove(self, node):
        """Удаляет узел из списка"""
        node.prev.next = node.next
        node.next.prev = node.prev

    def _move_to_front(self, node):
        """Перемещает узел в начало"""
        self._remove(node)
        self._add_to_front(node)

    def get(self, key):
        """O(1)"""
        if key not in self.cache:
            return -1

        node = self.cache.get(key)
        self._move_to_front(node)
        return node.value

    def put(self, key, value):
        """O(1)"""
        if key in self.cache:
            node = self.cache.get(key)
            node.value = value
            self._move_to_front(node)
        else:
            node = self.Node(key, value)
            self.cache.put(key, node)
            self._add_to_front(node)

            if len(self.cache) > self.capacity:
                # Удаляем LRU (перед tail)
                lru = self.tail.prev
                self._remove(lru)
                self.cache.remove(lru.key)
```

## Сравнение с другими структурами

### Python dict vs list

| Операция | dict (хэш-таблица) | list (массив) |
|----------|-------------------|---------------|
| Поиск по ключу | O(1) | O(n) |
| Вставка | O(1) | O(1)* или O(n) |
| Удаление по ключу | O(1) | O(n) |
| Доступ по индексу | O(1)** | O(1) |
| Упорядоченность | Да (Python 3.7+) | Да |

*В конец
**Если известен ключ

### dict vs set

```python
# dict — хранит пары (ключ, значение)
d = {"a": 1, "b": 2}
d["a"]  # 1

# set — хранит только ключи (значение = None)
s = {"a", "b"}
"a" in s  # True

# Внутренняя реализация одинакова!
```

## Типичные ошибки

### 1. Изменяемые ключи

```python
# НЕПРАВИЛЬНО — list нельзя хэшировать
d = {}
d[[1, 2]] = "value"  # TypeError!

# ПРАВИЛЬНО
d[tuple([1, 2])] = "value"
d[(1, 2)] = "value"
```

### 2. Изменение при итерации

```python
# НЕПРАВИЛЬНО
for key in table:
    if some_condition(key):
        table.remove(key)  # RuntimeError!

# ПРАВИЛЬНО
to_remove = [key for key in table if some_condition(key)]
for key in to_remove:
    table.remove(key)
```

### 3. Проверка наличия перед доступом

```python
# НЕОПТИМАЛЬНО
if key in table:
    value = table[key]
else:
    value = default

# ОПТИМАЛЬНО (один поиск вместо двух)
value = table.get(key, default)
```

### 4. Предположение о порядке

```python
# В Python 3.6 dict упорядочен, но это деталь реализации
# В Python 3.7+ это гарантия

# Если нужен порядок — используй OrderedDict
from collections import OrderedDict
```

## Оптимизации в Python dict

```python
# Python dict использует:
# 1. Open addressing с pseudo-random probing
# 2. Compact dict (Python 3.6+) — отдельные массивы для индексов и данных
# 3. Key sharing (для экземпляров классов)
# 4. SipHash для защиты от HashDoS

# Compact dict layout:
# indices: [None, 1, None, 0, 2, None, None, None]
# entries: [(hash, key, value), (hash, key, value), ...]

# Преимущества:
# - Меньше памяти
# - Сохраняет порядок вставки
# - Лучшая локальность кэша
```

## Когда использовать хэш-таблицу

**Используй хэш-таблицу, когда:**
- Нужен быстрый поиск по ключу
- Ключи уникальны
- Не важен порядок (или Python 3.7+)
- Нужна проверка принадлежности (in)
- Подсчёт частоты
- Кэширование

**НЕ используй хэш-таблицу, когда:**
- Нужен поиск по диапазону (используй TreeMap)
- Нужен порядок по ключам (используй TreeMap)
- Ключи — изменяемые объекты
- Мало элементов (list может быть быстрее)
