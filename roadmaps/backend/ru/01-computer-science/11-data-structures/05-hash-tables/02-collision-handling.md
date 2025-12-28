# Обработка коллизий (Collision Handling)

[prev: 01-hash-functions](./01-hash-functions.md) | [next: 03-hash-table-operations](./03-hash-table-operations.md)

---
## Определение

**Коллизия** — это ситуация, когда два или более различных ключа получают одинаковый хэш-код. Поскольку хэш-функция отображает бесконечное множество ключей в конечное множество индексов, коллизии неизбежны (принцип Дирихле).

```
Ключ "abc" ──┐
             ├──► hash() = 42 ──► Коллизия!
Ключ "xyz" ──┘
```

## Зачем нужно

Эффективная обработка коллизий критически важна для:
- **Корректности** — все элементы должны быть доступны
- **Производительности** — O(1) в среднем случае
- **Масштабируемости** — предсказуемое поведение при росте данных

## Методы обработки коллизий

### 1. Метод цепочек (Separate Chaining)

Каждая ячейка таблицы хранит связный список элементов с одинаковым хэшем.

```
Хэш-таблица с цепочками:

Index │ Цепочка
──────┼─────────────────────────
  0   │ → [("apple", 1)] → None
  1   │ → None
  2   │ → [("banana", 2)] → [("cherry", 3)] → None
  3   │ → [("date", 4)] → None
  4   │ → None

hash("banana") = hash("cherry") = 2 (коллизия)
```

#### Реализация

```python
class ChainedHashTable:
    def __init__(self, capacity=10):
        self.capacity = capacity
        self.size = 0
        self.buckets = [[] for _ in range(capacity)]

    def _hash(self, key):
        return hash(key) % self.capacity

    def put(self, key, value):
        """Вставка - O(1) в среднем"""
        index = self._hash(key)
        bucket = self.buckets[index]

        # Проверяем, есть ли уже такой ключ
        for i, (k, v) in enumerate(bucket):
            if k == key:
                bucket[i] = (key, value)  # обновляем
                return

        bucket.append((key, value))
        self.size += 1

        # Проверяем load factor
        if self.size / self.capacity > 0.75:
            self._resize()

    def get(self, key):
        """Поиск - O(1) в среднем"""
        index = self._hash(key)
        bucket = self.buckets[index]

        for k, v in bucket:
            if k == key:
                return v

        raise KeyError(key)

    def remove(self, key):
        """Удаление - O(1) в среднем"""
        index = self._hash(key)
        bucket = self.buckets[index]

        for i, (k, v) in enumerate(bucket):
            if k == key:
                del bucket[i]
                self.size -= 1
                return v

        raise KeyError(key)

    def _resize(self):
        """Увеличение размера таблицы"""
        old_buckets = self.buckets
        self.capacity *= 2
        self.buckets = [[] for _ in range(self.capacity)]
        self.size = 0

        for bucket in old_buckets:
            for key, value in bucket:
                self.put(key, value)
```

#### Преимущества и недостатки

```
✓ Преимущества:
  - Простая реализация
  - Нет переполнения (списки расширяемы)
  - Хорошо работает при большом load factor
  - Удаление просто

✗ Недостатки:
  - Дополнительная память для указателей
  - Плохая локальность кэша
  - Длинные цепочки при плохой хэш-функции
```

### 2. Открытая адресация (Open Addressing)

Все элементы хранятся в самой таблице. При коллизии ищем следующую свободную ячейку.

#### Линейное пробирование (Linear Probing)

```
Формула: h(k, i) = (h(k) + i) mod m

Где:
- h(k) — начальный хэш
- i — номер попытки (0, 1, 2, ...)
- m — размер таблицы

Пример вставки:
Таблица: [_, _, _, _, _, _, _, _]  m = 8

insert("A", h=2): [_, _, A, _, _, _, _, _]
insert("B", h=2): [_, _, A, B, _, _, _, _]  (пробирование: 2→3)
insert("C", h=3): [_, _, A, B, C, _, _, _]  (пробирование: 3→4)
insert("D", h=2): [_, _, A, B, C, D, _, _]  (пробирование: 2→3→4→5)
```

```python
class LinearProbingHashTable:
    def __init__(self, capacity=10):
        self.capacity = capacity
        self.size = 0
        self.keys = [None] * capacity
        self.values = [None] * capacity
        self.DELETED = object()  # маркер удалённого элемента

    def _hash(self, key):
        return hash(key) % self.capacity

    def put(self, key, value):
        """Вставка с линейным пробированием"""
        if self.size >= self.capacity * 0.7:
            self._resize()

        index = self._hash(key)
        first_deleted = None

        for i in range(self.capacity):
            probe_index = (index + i) % self.capacity

            if self.keys[probe_index] is None:
                # Нашли пустую ячейку
                insert_index = first_deleted if first_deleted is not None else probe_index
                self.keys[insert_index] = key
                self.values[insert_index] = value
                self.size += 1
                return

            if self.keys[probe_index] is self.DELETED:
                if first_deleted is None:
                    first_deleted = probe_index

            elif self.keys[probe_index] == key:
                # Ключ уже существует — обновляем
                self.values[probe_index] = value
                return

        raise Exception("Hash table is full")

    def get(self, key):
        """Поиск с линейным пробированием"""
        index = self._hash(key)

        for i in range(self.capacity):
            probe_index = (index + i) % self.capacity

            if self.keys[probe_index] is None:
                raise KeyError(key)

            if self.keys[probe_index] == key:
                return self.values[probe_index]

            # Пропускаем DELETED и продолжаем

        raise KeyError(key)

    def remove(self, key):
        """Удаление с пометкой DELETED"""
        index = self._hash(key)

        for i in range(self.capacity):
            probe_index = (index + i) % self.capacity

            if self.keys[probe_index] is None:
                raise KeyError(key)

            if self.keys[probe_index] == key:
                value = self.values[probe_index]
                self.keys[probe_index] = self.DELETED
                self.values[probe_index] = None
                self.size -= 1
                return value

        raise KeyError(key)

    def _resize(self):
        old_keys = self.keys
        old_values = self.values
        self.capacity *= 2
        self.keys = [None] * self.capacity
        self.values = [None] * self.capacity
        self.size = 0

        for i in range(len(old_keys)):
            if old_keys[i] is not None and old_keys[i] is not self.DELETED:
                self.put(old_keys[i], old_values[i])
```

#### Квадратичное пробирование (Quadratic Probing)

```
Формула: h(k, i) = (h(k) + c₁*i + c₂*i²) mod m

Обычно c₁ = 0, c₂ = 1:
h(k, i) = (h(k) + i²) mod m

Последовательность: 0, 1, 4, 9, 16, 25, ...

Преимущество: уменьшает кластеризацию
Недостаток: может не найти свободную ячейку
```

```python
def _probe_quadratic(self, key, i):
    return (self._hash(key) + i * i) % self.capacity
```

```
Пример:
h(k) = 3, m = 11

i=0: (3 + 0) % 11 = 3
i=1: (3 + 1) % 11 = 4
i=2: (3 + 4) % 11 = 7
i=3: (3 + 9) % 11 = 1
i=4: (3 + 16) % 11 = 8
```

#### Двойное хэширование (Double Hashing)

```
Формула: h(k, i) = (h₁(k) + i * h₂(k)) mod m

Где:
- h₁(k) — первичная хэш-функция
- h₂(k) — вторичная хэш-функция (шаг пробирования)

Важно: h₂(k) ≠ 0 и взаимно просто с m
Обычно: h₂(k) = prime - (k mod prime), где prime < m
```

```python
def _hash1(self, key):
    return hash(key) % self.capacity

def _hash2(self, key):
    # Гарантируем ненулевой шаг
    prime = 7  # маленькое простое число
    return prime - (hash(key) % prime)

def _probe_double(self, key, i):
    return (self._hash1(key) + i * self._hash2(key)) % self.capacity
```

```
Пример:
h₁("abc") = 5, h₂("abc") = 3, m = 11

i=0: (5 + 0*3) % 11 = 5
i=1: (5 + 1*3) % 11 = 8
i=2: (5 + 2*3) % 11 = 0
i=3: (5 + 3*3) % 11 = 3
```

### Сравнение методов пробирования

```
Линейное:      0  1  2  3  4  5  6  7  8  ...
               h  →  →  →  →  →  →  →  →

Квадратичное:  0  1  4  9  16 25 36 49 64 ...
               h  →     →        →

Двойное:       0  3  6  9  12 15 18 21 24 ... (при h₂=3)
               h     →     →     →
```

## Кластеризация

### Первичная кластеризация (Primary Clustering)

Проблема линейного пробирования:

```
Таблица: [_, X, X, X, _, X, X, _]
                    ↑
Если h(k) попадает сюда, нужно пройти весь кластер!

Цепочка: h → h+1 → h+2 → h+3 → ... (долгий поиск)
```

### Вторичная кластеризация (Secondary Clustering)

Проблема квадратичного пробирования:

```
Все ключи с одинаковым h(k) следуют одной траектории:

h(k)=3: 3 → 4 → 7 → 1 → 8 → ...
Все ключи с h=3 идут по этому пути!
```

### Решение — двойное хэширование

```
Разные ключи с одинаковым h₁ имеют разный h₂:

k₁: h₁=3, h₂=2: 3 → 5 → 7 → 9 → ...
k₂: h₁=3, h₂=5: 3 → 8 → 2 → 7 → ...

Разные траектории!
```

## Robin Hood Hashing

Оптимизация открытой адресации: "богатые отдают бедным".

```python
def put_robin_hood(self, key, value):
    """
    При вставке, если новый элемент "беднее"
    (дальше от своей идеальной позиции),
    он занимает место "богатого" элемента.
    """
    index = self._hash(key)
    distance = 0  # расстояние от идеальной позиции

    while True:
        probe_index = (index + distance) % self.capacity

        if self.keys[probe_index] is None:
            self.keys[probe_index] = key
            self.values[probe_index] = value
            self.distances[probe_index] = distance
            return

        existing_distance = self.distances[probe_index]

        if distance > existing_distance:
            # Новый элемент "беднее" — забираем место
            self.keys[probe_index], key = key, self.keys[probe_index]
            self.values[probe_index], value = value, self.values[probe_index]
            self.distances[probe_index], distance = distance, existing_distance

        distance += 1
```

```
Преимущества Robin Hood:
- Выравнивает длину цепочек
- Уменьшает максимальное время поиска
- Лучше предсказуемость производительности
```

## Cuckoo Hashing

Использует две хэш-функции и две таблицы:

```python
class CuckooHashTable:
    """
    Каждый ключ может быть в одной из двух позиций:
    - table1[h1(key)]
    - table2[h2(key)]

    Поиск всегда O(1) — проверяем только 2 позиции!
    """
    def __init__(self, capacity=10):
        self.capacity = capacity
        self.table1 = [None] * capacity
        self.table2 = [None] * capacity

    def _hash1(self, key):
        return hash(key) % self.capacity

    def _hash2(self, key):
        return (hash(key) // self.capacity) % self.capacity

    def get(self, key):
        """Поиск за O(1) — всегда!"""
        h1 = self._hash1(key)
        if self.table1[h1] and self.table1[h1][0] == key:
            return self.table1[h1][1]

        h2 = self._hash2(key)
        if self.table2[h2] and self.table2[h2][0] == key:
            return self.table2[h2][1]

        raise KeyError(key)

    def put(self, key, value):
        """
        Вставка: если место занято, выталкиваем старый элемент
        (как кукушка выталкивает яйца из гнезда)
        """
        MAX_ITERATIONS = self.capacity * 2

        current_key, current_value = key, value
        use_table1 = True

        for _ in range(MAX_ITERATIONS):
            if use_table1:
                h = self._hash1(current_key)
                if self.table1[h] is None:
                    self.table1[h] = (current_key, current_value)
                    return
                # Выталкиваем старый элемент
                old = self.table1[h]
                self.table1[h] = (current_key, current_value)
                current_key, current_value = old
            else:
                h = self._hash2(current_key)
                if self.table2[h] is None:
                    self.table2[h] = (current_key, current_value)
                    return
                old = self.table2[h]
                self.table2[h] = (current_key, current_value)
                current_key, current_value = old

            use_table1 = not use_table1

        # Цикл — нужен rehash
        self._rehash()
        self.put(current_key, current_value)
```

```
Визуализация Cuckoo:

insert(A): table1[h1(A)] = A
insert(B): table1[h1(B)] = B
insert(C): h1(C) занят → выталкиваем A → A идёт в table2[h2(A)]
           table1[h1(C)] = C, table2[h2(A)] = A
```

## Load Factor и Rehashing

**Load Factor (коэффициент заполнения)**:
```
α = n / m

Где:
- n — количество элементов
- m — размер таблицы

Рекомендации:
- Chaining: α < 1.0 (можно больше)
- Open Addressing: α < 0.7 (обычно 0.5-0.7)
```

**Rehashing (перехеширование)**:
```python
def _rehash(self):
    """Увеличиваем таблицу и перехешируем все элементы"""
    old_data = self.data
    self.capacity *= 2
    self.data = [None] * self.capacity
    self.size = 0

    for item in old_data:
        if item is not None and item is not DELETED:
            self.put(item.key, item.value)
```

## Сравнение методов

| Метод | Поиск (успех) | Поиск (неуспех) | Память | Локальность |
|-------|---------------|-----------------|--------|-------------|
| Chaining | O(1 + α) | O(1 + α) | Высокая | Плохая |
| Linear Probing | O(1/(1-α)) | O(1/(1-α)²) | Низкая | Хорошая |
| Quadratic | O(1/(1-α)) | O(1/(1-α)) | Низкая | Средняя |
| Double Hashing | O(1/(1-α)) | O(1/(1-α)) | Низкая | Плохая |
| Cuckoo | **O(1)** | **O(1)** | 2x таблица | Хорошая |

## Типичные ошибки

### 1. Удаление без маркера в открытой адресации

```python
# НЕПРАВИЛЬНО
def remove(self, key):
    index = self._find(key)
    self.table[index] = None  # разрывает цепочку поиска!

# Пример проблемы:
# insert A (h=2), insert B (h=2→3), remove A
# table: [_, _, None, B, ...]
# get B: h=2 → None → KeyError!  (хотя B на позиции 3)

# ПРАВИЛЬНО
def remove(self, key):
    index = self._find(key)
    self.table[index] = DELETED  # маркер, поиск продолжится
```

### 2. Бесконечный цикл в Cuckoo

```python
# НЕПРАВИЛЬНО — без ограничения итераций
def put(self, key, value):
    while True:
        # может зациклиться при определённых хэшах
        ...

# ПРАВИЛЬНО
def put(self, key, value):
    for _ in range(MAX_ITERATIONS):
        ...
    self._rehash()  # при цикле — перехеширование
```

### 3. Неправильный размер таблицы

```python
# ПЛОХО для открытой адресации
capacity = 100  # при α > 0.7 будет много коллизий

# Для квадратичного пробирования:
# capacity должен быть простым числом или степенью 2

# ХОРОШО
capacity = 97   # простое число
capacity = 128  # степень 2 (для битовых операций)
```

### 4. h₂(k) = 0 в двойном хэшировании

```python
# НЕПРАВИЛЬНО — бесконечный цикл если h₂ = 0
def _hash2(self, key):
    return hash(key) % 7  # может вернуть 0!

# ПРАВИЛЬНО
def _hash2(self, key):
    return 1 + (hash(key) % 6)  # всегда от 1 до 6
```

## Практические рекомендации

1. **Для большинства случаев** — используй chaining (проще, надёжнее)
2. **Для высокой производительности** — linear probing с хорошей хэш-функцией
3. **Для гарантированного O(1)** — Cuckoo hashing
4. **Load factor** — держи ниже 0.7 для открытой адресации
5. **Размер таблицы** — простое число или степень 2

---

[prev: 01-hash-functions](./01-hash-functions.md) | [next: 03-hash-table-operations](./03-hash-table-operations.md)
