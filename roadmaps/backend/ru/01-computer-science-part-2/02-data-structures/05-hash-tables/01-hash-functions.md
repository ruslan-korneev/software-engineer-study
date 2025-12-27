# Хэш-функции (Hash Functions)

## Определение

**Хэш-функция** — это функция, которая преобразует входные данные произвольного размера в выходное значение фиксированного размера (хэш-код, хэш, дайджест). Хорошая хэш-функция должна быть детерминированной, быстрой и обеспечивать равномерное распределение значений.

```
Входные данные           Хэш-функция          Хэш-код
─────────────────────────────────────────────────────────
"hello"          ───────►  h(x)  ───────►  2314539217
"Hello"          ───────►  h(x)  ───────►  4081352318
"hello world"    ───────►  h(x)  ───────►  1794106052
```

## Зачем нужно

### Практическое применение:
- **Хэш-таблицы** — быстрый поиск за O(1)
- **Контрольные суммы** — проверка целостности данных
- **Криптография** — цифровые подписи, пароли
- **Кэширование** — ключи кэша
- **Дедупликация** — обнаружение дубликатов
- **Распределённые системы** — консистентное хэширование

## Свойства хорошей хэш-функции

### 1. Детерминированность
Одинаковые входные данные всегда дают одинаковый хэш.

```python
hash("hello") == hash("hello")  # Всегда True
```

### 2. Равномерность (Uniformity)
Хэши равномерно распределены по диапазону значений.

```
ПЛОХО (кластеризация):        ХОРОШО (равномерное):
▓▓▓▓▓▓░░░░░░░░░░░░░░         ▓░▓░▓░▓░▓░▓░▓░▓░▓░
       ↑                      ↑
  все хэши здесь         хэши распределены равномерно
```

### 3. Быстрота вычисления
O(длина ключа), минимальные накладные расходы.

### 4. Лавинный эффект (Avalanche Effect)
Малое изменение входа → большое изменение хэша.

```python
h("hello")  = 0b10110011...
h("hallo")  = 0b01001100...  # ~50% бит изменились
              ↑↑↑↑↑↑↑↑
```

### 5. Минимизация коллизий
Разные входы редко дают одинаковый хэш.

## Типы хэш-функций

### 1. Простые хэш-функции (для хэш-таблиц)

```python
# Хэш для целых чисел
def hash_int(key, table_size):
    return key % table_size

# Хэш для строк — полиномиальный
def hash_string(s, table_size):
    h = 0
    base = 31  # простое число
    for char in s:
        h = (h * base + ord(char)) % table_size
    return h

# Пример:
# hash_string("abc", 1000)
# = ((0 * 31 + 97) * 31 + 98) * 31 + 99
# = (97 * 31 + 98) * 31 + 99
# = (3007 + 98) * 31 + 99
# = 3105 * 31 + 99
# = 96354 % 1000 = 354
```

### 2. Криптографические хэш-функции

```python
import hashlib

# MD5 (устарел, не для безопасности!)
hashlib.md5(b"hello").hexdigest()
# 5d41402abc4b2a76b9719d911017c592

# SHA-256 (рекомендуется)
hashlib.sha256(b"hello").hexdigest()
# 2cf24dba5fb0a30e26e83b2ac5b9e29e1b161e5c1fa7425e73043362938b9824

# SHA-3
hashlib.sha3_256(b"hello").hexdigest()

# BLAKE2 (быстрый и безопасный)
hashlib.blake2b(b"hello").hexdigest()
```

### 3. Некриптографические хэш-функции (быстрые)

```
MurmurHash  — быстрый, хорошее распределение
xxHash      — очень быстрый
CityHash    — Google, для строк
FarmHash    — Google, развитие CityHash
SipHash     — защита от HashDoS атак
```

## Популярные алгоритмы хэширования

### Division Method (метод деления)

```python
def division_hash(key, m):
    """
    h(k) = k mod m
    m — размер таблицы (лучше простое число)
    """
    return key % m

# Примеры:
# division_hash(123, 10) = 3
# division_hash(456, 10) = 6

# Рекомендация: m = простое число, не близкое к степени 2
# Хорошо: 97, 193, 389, 769, 1543, 3079
# Плохо: 100, 256, 1024
```

### Multiplication Method (метод умножения)

```python
def multiplication_hash(key, m):
    """
    h(k) = floor(m * (k * A mod 1))
    A — константа, 0 < A < 1
    Knuth рекомендует A = (√5 - 1) / 2 ≈ 0.6180339887
    """
    A = 0.6180339887
    return int(m * ((key * A) % 1))

# Преимущество: m может быть любым (не обязательно простым)
```

### Mid-Square Method (метод середины квадрата)

```python
def mid_square_hash(key, digits):
    """
    1. Возводим ключ в квадрат
    2. Берём средние цифры
    """
    squared = key ** 2
    s = str(squared)

    mid = len(s) // 2
    start = mid - digits // 2
    end = start + digits

    return int(s[start:end]) if start >= 0 else int(s[:digits])

# Пример: key = 1234
# 1234² = 1522756
# Средние 2 цифры: 27 или 22
```

### Folding Method (метод свёртки)

```python
def folding_hash(key, table_size):
    """
    Разбиваем ключ на части и складываем
    """
    key_str = str(key)
    chunk_size = len(str(table_size - 1))
    total = 0

    for i in range(0, len(key_str), chunk_size):
        chunk = key_str[i:i + chunk_size]
        total += int(chunk)

    return total % table_size

# Пример: key = 123456789, table_size = 1000
# Части: 123, 456, 789
# Сумма: 123 + 456 + 789 = 1368
# Хэш: 1368 % 1000 = 368
```

### Polynomial Rolling Hash (полиномиальный)

```python
def polynomial_hash(s, base=31, mod=10**9 + 9):
    """
    h(s) = s[0] + s[1]*p + s[2]*p² + ... + s[n-1]*p^(n-1)

    Используется для:
    - Rabin-Karp алгоритм поиска подстроки
    - Сравнение строк за O(1)
    """
    h = 0
    p_pow = 1

    for char in s:
        h = (h + (ord(char) - ord('a') + 1) * p_pow) % mod
        p_pow = (p_pow * base) % mod

    return h

# Преимущество: можно быстро пересчитать хэш при сдвиге окна
```

### Python's hash() — SipHash

```python
# Python использует SipHash-2-4 для защиты от HashDoS

# Хэширование разных типов:
hash(42)           # целое число
hash(3.14)         # float
hash("hello")      # строка
hash((1, 2, 3))    # кортеж
hash(frozenset([1, 2]))  # frozenset

# ВАЖНО: хэш строк рандомизирован при каждом запуске Python!
# Для воспроизводимости: PYTHONHASHSEED=0
```

## Коллизии

**Коллизия** — ситуация, когда разные ключи дают одинаковый хэш.

```
h("abc") = 42
h("xyz") = 42  ← коллизия!
```

### Парадокс дней рождения

```
Вероятность коллизии при n элементах и m возможных хэшей:

P(коллизия) ≈ 1 - e^(-n²/2m)

Пример: 23 человека, 365 дней
P ≈ 1 - e^(-529/730) ≈ 50.7%

Для хэш-таблицы:
При n = √m элементов, P(коллизия) ≈ 50%
При 10000 элементов и 32-битном хэше (4.3 млрд значений):
P(хотя бы одна коллизия) ≈ 1.2%
```

## Примеры с разбором

### Пример 1: Хэш-функция для структуры

```python
class Point:
    def __init__(self, x, y):
        self.x = x
        self.y = y

    def __hash__(self):
        # Комбинируем хэши полей
        return hash((self.x, self.y))

    def __eq__(self, other):
        return self.x == other.x and self.y == other.y

# Теперь Point можно использовать как ключ в dict:
points = {Point(1, 2): "A", Point(3, 4): "B"}
```

### Пример 2: Консистентное хэширование

```python
import hashlib
import bisect

class ConsistentHash:
    """
    Консистентное хэширование для распределённых систем.
    При добавлении/удалении сервера перераспределяется
    минимальное количество ключей.
    """
    def __init__(self, nodes=None, replicas=100):
        self.replicas = replicas
        self.ring = {}  # hash -> node
        self.sorted_keys = []

        if nodes:
            for node in nodes:
                self.add_node(node)

    def _hash(self, key):
        return int(hashlib.md5(key.encode()).hexdigest(), 16)

    def add_node(self, node):
        """Добавляет узел с виртуальными репликами"""
        for i in range(self.replicas):
            key = f"{node}:{i}"
            h = self._hash(key)
            self.ring[h] = node
            bisect.insort(self.sorted_keys, h)

    def remove_node(self, node):
        """Удаляет узел"""
        for i in range(self.replicas):
            key = f"{node}:{i}"
            h = self._hash(key)
            del self.ring[h]
            self.sorted_keys.remove(h)

    def get_node(self, key):
        """Находит узел для ключа"""
        if not self.ring:
            return None

        h = self._hash(key)
        idx = bisect.bisect(self.sorted_keys, h)

        if idx == len(self.sorted_keys):
            idx = 0

        return self.ring[self.sorted_keys[idx]]

# Использование:
ch = ConsistentHash(["server1", "server2", "server3"])
ch.get_node("user123")  # → "server2"
ch.get_node("user456")  # → "server1"
```

### Пример 3: Rabin-Karp поиск подстроки

```python
def rabin_karp(text, pattern):
    """
    Поиск подстроки с использованием rolling hash.
    Время: O(n + m) в среднем, O(n*m) в худшем случае
    """
    n, m = len(text), len(pattern)
    if m > n:
        return -1

    base = 256
    mod = 10**9 + 7

    # Вычисляем base^(m-1) для удаления первого символа
    h = pow(base, m - 1, mod)

    # Хэш паттерна
    pattern_hash = 0
    for c in pattern:
        pattern_hash = (pattern_hash * base + ord(c)) % mod

    # Хэш первого окна
    window_hash = 0
    for c in text[:m]:
        window_hash = (window_hash * base + ord(c)) % mod

    # Поиск
    for i in range(n - m + 1):
        if window_hash == pattern_hash:
            # Проверяем посимвольно (защита от коллизий)
            if text[i:i + m] == pattern:
                return i

        # Сдвигаем окно
        if i < n - m:
            # Удаляем первый символ, добавляем следующий
            window_hash = (window_hash - ord(text[i]) * h) % mod
            window_hash = (window_hash * base + ord(text[i + m])) % mod
            window_hash = (window_hash + mod) % mod  # защита от отрицательных

    return -1

# Пример:
rabin_karp("hello world", "world")  # 6
```

### Пример 4: Bloom Filter (вероятностная структура)

```python
import hashlib

class BloomFilter:
    """
    Bloom Filter — вероятностная структура данных.
    Может давать false positive, но никогда false negative.
    """
    def __init__(self, size, num_hashes):
        self.size = size
        self.num_hashes = num_hashes
        self.bit_array = [False] * size

    def _hashes(self, item):
        """Генерирует num_hashes хэшей для элемента"""
        hashes = []
        for i in range(self.num_hashes):
            h = hashlib.sha256(f"{item}{i}".encode()).hexdigest()
            hashes.append(int(h, 16) % self.size)
        return hashes

    def add(self, item):
        """Добавляет элемент"""
        for h in self._hashes(item):
            self.bit_array[h] = True

    def contains(self, item):
        """Проверяет наличие (может быть false positive!)"""
        return all(self.bit_array[h] for h in self._hashes(item))

# Использование:
bf = BloomFilter(size=1000, num_hashes=3)
bf.add("apple")
bf.add("banana")

bf.contains("apple")   # True
bf.contains("banana")  # True
bf.contains("cherry")  # Скорее всего False, но может быть True
```

## Типичные ошибки

### 1. Хэширование изменяемых объектов

```python
# НЕПРАВИЛЬНО — list нельзя хэшировать
d = {}
d[[1, 2, 3]] = "value"  # TypeError: unhashable type: 'list'

# ПРАВИЛЬНО — используем tuple
d[(1, 2, 3)] = "value"

# Или frozenset для множеств:
d[frozenset([1, 2, 3])] = "value"
```

### 2. Несоответствие __hash__ и __eq__

```python
# НЕПРАВИЛЬНО
class Bad:
    def __init__(self, x):
        self.x = x

    def __hash__(self):
        return hash(self.x)

    # Забыли __eq__!

# Объекты с одинаковым хэшем не будут считаться равными

# ПРАВИЛЬНО
class Good:
    def __init__(self, x):
        self.x = x

    def __hash__(self):
        return hash(self.x)

    def __eq__(self, other):
        return isinstance(other, Good) and self.x == other.x
```

### 3. Использование нестабильного хэша

```python
# Python рандомизирует хэши строк при каждом запуске!

# Запуск 1:
hash("hello")  # 123456789

# Запуск 2:
hash("hello")  # 987654321 (другое значение!)

# Для стабильного хэша используйте hashlib:
import hashlib
hashlib.sha256(b"hello").hexdigest()  # всегда одинаковый
```

### 4. Плохой выбор размера таблицы

```python
# ПЛОХО — степень 2
table_size = 256  # много коллизий для данных с паттернами

# ПЛОХО — чётное число
table_size = 100

# ХОРОШО — простое число
table_size = 97
table_size = 193
table_size = 389
```

### 5. Неправильная обработка отрицательных хэшей

```python
# В Python hash() может вернуть отрицательное число
hash(-1)  # -2

# При использовании как индекс:
index = hash(key) % table_size  # может быть отрицательным в C!

# В Python % всегда возвращает неотрицательный результат
# Но в C/Java нужно:
index = ((hash(key) % table_size) + table_size) % table_size
```

## Сравнение хэш-функций

| Функция | Скорость | Безопасность | Применение |
|---------|----------|--------------|------------|
| Division | Очень быстрая | Нет | Хэш-таблицы |
| Polynomial | Быстрая | Нет | Строки, Rabin-Karp |
| MurmurHash | Очень быстрая | Нет | Хэш-таблицы, кэши |
| xxHash | Очень быстрая | Нет | Контрольные суммы |
| SipHash | Быстрая | Средняя (HashDoS) | Python dict |
| MD5 | Средняя | Нет (взломан) | Только контрольные суммы |
| SHA-256 | Медленная | Высокая | Криптография |
| BLAKE2 | Быстрая | Высокая | Криптография, общее |
