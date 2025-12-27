# Фильтры Блума (Bloom Filters)

## Определение

**Фильтр Блума** — это вероятностная структура данных, позволяющая эффективно проверять принадлежность элемента к множеству. Особенность: фильтр может давать **ложноположительные ответы** ("возможно, есть"), но **никогда не даёт ложноотрицательных** ("точно нет").

```
Фильтр Блума — битовый массив с k хеш-функциями:

Добавляем "apple":
h1("apple") = 2
h2("apple") = 5
h3("apple") = 9

Битовый массив:
Позиция:  0   1   2   3   4   5   6   7   8   9   10  11
Биты:    [0] [0] [1] [0] [0] [1] [0] [0] [0] [1] [0] [0]
                  ↑           ↑               ↑
               h1          h2              h3

Проверка "apple": биты 2, 5, 9 = 1,1,1 → "Возможно есть"
Проверка "banana": биты 3, 7, 11 = 0,0,0 → "Точно нет"
```

Фильтр Блума был предложен Бёртоном Блумом в 1970 году.

## Зачем нужно

### Проблема точных структур

```
Хеш-таблица:
+ Точный ответ O(1)
- Память O(n) — хранит сами элементы

Множество из 1 млрд URL:
- Средняя длина URL: 77 байт
- Память: 77 GB минимум

Фильтр Блума для 1 млрд URL:
- 10 бит на элемент при 1% FPR
- Память: 1.2 GB
```

### Типичные сценарии использования

1. **Кэширование**: "Стоит ли искать в кэше?"
2. **Дедупликация**: "Видели ли мы этот элемент раньше?"
3. **Базы данных**: "Есть ли ключ в SSTable?"
4. **Сети**: "Нужно ли проверять этот URL на спам?"
5. **Распределённые системы**: "Есть ли данные на этом узле?"

### Применение в реальных системах

- **Google BigTable/LevelDB**: фильтрация SSTable перед чтением с диска
- **Apache Cassandra**: проверка существования строк
- **PostgreSQL**: индексы BRIN
- **Chrome**: проверка вредоносных URL
- **Medium**: рекомендации ("не показывать прочитанное")
- **Bitcoin**: SPV-кошельки для фильтрации транзакций

## Как работает

### Инициализация

```
Параметры:
- m: размер битового массива
- k: количество хеш-функций
- n: ожидаемое число элементов

Начальное состояние:
bits = [0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0, 0]  # m = 12
```

### Добавление элемента

```
add("cat"):
  h1("cat") = 1
  h2("cat") = 4
  h3("cat") = 7

  bits[1] = 1
  bits[4] = 1
  bits[7] = 1

До:    [0][0][0][0][0][0][0][0][0][0][0][0]
После: [0][1][0][0][1][0][0][1][0][0][0][0]
           ↑       ↑       ↑
```

### Добавление второго элемента

```
add("dog"):
  h1("dog") = 3
  h2("dog") = 4   ← уже установлен!
  h3("dog") = 10

До:    [0][1][0][0][1][0][0][1][0][0][0][0]
После: [0][1][0][1][1][0][0][1][0][0][1][0]
              ↑                       ↑
```

### Проверка членства

```
contains("cat"):
  h1("cat") = 1  → bits[1] = 1 ✓
  h2("cat") = 4  → bits[4] = 1 ✓
  h3("cat") = 7  → bits[7] = 1 ✓
  Результат: "Возможно есть" (TRUE POSITIVE)

contains("bird"):
  h1("bird") = 1  → bits[1] = 1 ✓  (случайное совпадение)
  h2("bird") = 6  → bits[6] = 0 ✗
  Результат: "Точно нет" (TRUE NEGATIVE)

contains("fox"):
  h1("fox") = 1   → bits[1] = 1 ✓  (от "cat")
  h2("fox") = 3   → bits[3] = 1 ✓  (от "dog")
  h3("fox") = 4   → bits[4] = 1 ✓  (от "cat"/"dog")
  Результат: "Возможно есть" (FALSE POSITIVE!)
```

### Визуализация ложноположительного результата

```
Элементы в фильтре: "cat", "dog"

bits: [0][1][0][1][1][0][0][1][0][0][1][0]

Проверка "fox" (не добавлялся):

h1("fox") = 1  ───────────────┐
                              ↓
bits: [0][1][0][1][1][0][0][1][0][0][1][0]
           ↑   ↑   ↑
h2("fox") = 3  ─────────┘   │
                            │
h3("fox") = 4  ─────────────┘

Все биты = 1, но элемент никогда не добавлялся!
Это FALSE POSITIVE — цена компактности.
```

## Инварианты и свойства

### Гарантии фильтра Блума

```
1. FALSE NEGATIVES = 0 (никогда!)
   Если contains() вернул false → элемента ТОЧНО нет

2. FALSE POSITIVES ≤ заданный уровень
   Если contains() вернул true → элемент ВОЗМОЖНО есть

3. Удаление невозможно
   Обнуление бита может повлиять на другие элементы
```

### Вероятность ложноположительного результата

```
После добавления n элементов, вероятность FP:

P(FP) ≈ (1 - e^(-kn/m))^k

Где:
- m = размер массива (биты)
- k = число хеш-функций
- n = число добавленных элементов
```

### Оптимальные параметры

```
Оптимальное число хеш-функций:
k_opt = (m/n) * ln(2) ≈ 0.693 * (m/n)

При оптимальном k:
P(FP) ≈ (0.6185)^(m/n)

Или обратно — размер фильтра для заданного FPR:
m = -n * ln(p) / (ln(2))^2

Где p — желаемая вероятность FP
```

### Таблица типичных параметров

```
Для n = 1,000,000 элементов:

Желаемый FPR  |  m (биты)  |  k (хешей)  |  Память
─────────────────────────────────────────────────────
10%           |  4.8M      |  3          |  585 KB
1%            |  9.6M      |  7          |  1.2 MB
0.1%          |  14.4M     |  10         |  1.8 MB
0.01%         |  19.2M     |  13         |  2.4 MB
```

## Псевдокод основных операций

### Базовая реализация

```python
import math
import mmh3  # MurmurHash3

class BloomFilter:
    def __init__(self, expected_elements, false_positive_rate):
        """
        Инициализация фильтра Блума

        :param expected_elements: ожидаемое число элементов (n)
        :param false_positive_rate: допустимый уровень FP (p)
        """
        self.n = expected_elements
        self.p = false_positive_rate

        # Вычисляем оптимальный размер
        self.m = self._optimal_size()
        # Вычисляем оптимальное число хешей
        self.k = self._optimal_hash_count()

        # Битовый массив
        self.bits = [0] * self.m
        self.count = 0

    def _optimal_size(self):
        """m = -n * ln(p) / (ln(2))^2"""
        m = -(self.n * math.log(self.p)) / (math.log(2) ** 2)
        return int(math.ceil(m))

    def _optimal_hash_count(self):
        """k = (m/n) * ln(2)"""
        k = (self.m / self.n) * math.log(2)
        return int(math.ceil(k))
```

### Хеш-функции

```python
def _hashes(self, item):
    """
    Генерирует k хеш-значений для элемента

    Использует технику "double hashing":
    h(i) = h1 + i*h2 (mod m)

    Это позволяет использовать только 2 хеш-функции
    для генерации произвольного числа хешей
    """
    # Два независимых хеша
    h1 = mmh3.hash(str(item), 0) % self.m
    h2 = mmh3.hash(str(item), 1) % self.m

    for i in range(self.k):
        yield (h1 + i * h2) % self.m
```

### Добавление элемента

```python
def add(self, item):
    """
    Добавляет элемент в фильтр

    Время: O(k) — k хеш-вычислений
    """
    for position in self._hashes(item):
        self.bits[position] = 1
    self.count += 1
```

### Проверка членства

```python
def contains(self, item):
    """
    Проверяет, может ли элемент быть в множестве

    Возвращает:
    - False: элемента ТОЧНО нет
    - True: элемент ВОЗМОЖНО есть

    Время: O(k)
    """
    for position in self._hashes(item):
        if self.bits[position] == 0:
            return False  # Точно нет
    return True  # Возможно есть
```

### Оценка текущего FPR

```python
def current_false_positive_rate(self):
    """
    Оценивает текущую вероятность ложноположительного результата
    на основе числа добавленных элементов
    """
    # Вероятность того, что конкретный бит = 0
    prob_bit_zero = (1 - 1/self.m) ** (self.k * self.count)

    # Вероятность FP = вероятность, что все k битов = 1
    return (1 - prob_bit_zero) ** self.k
```

### Пример использования

```python
# Создаём фильтр для 1 млн элементов с 1% FPR
bf = BloomFilter(expected_elements=1_000_000, false_positive_rate=0.01)

print(f"Размер: {bf.m} бит ({bf.m // 8 // 1024} KB)")
print(f"Хеш-функций: {bf.k}")

# Добавляем элементы
urls = ["https://example.com", "https://google.com", "https://github.com"]
for url in urls:
    bf.add(url)

# Проверяем
print(bf.contains("https://example.com"))  # True (точно)
print(bf.contains("https://unknown.com"))  # False или True (возможен FP)
```

## Анализ сложности

| Операция | Время | Пространство |
|----------|-------|--------------|
| add()    | O(k)  | — |
| contains() | O(k) | — |
| Память   | — | O(m) бит |

### Сравнение с другими структурами

```
Для n = 10,000,000 URL (средняя длина 77 байт)

Структура        | Память    | contains() | Точность
─────────────────────────────────────────────────────
HashSet          | 770 MB    | O(1)       | 100%
Bloom (1% FPR)   | 12 MB     | O(k)       | 99%
Bloom (0.1% FPR) | 18 MB     | O(k)       | 99.9%
```

## Сравнение с альтернативами

| Свойство | Bloom | Cuckoo | Count-Min | Hash Set |
|----------|-------|--------|-----------|----------|
| Ложноотрицательные | 0% | 0% | 0% | 0% |
| Ложноположительные | Есть | Есть | Есть | 0% |
| Удаление | Нет | Да | Да | Да |
| Подсчёт | Нет | Нет | Да | — |
| Память | Очень мало | Мало | Мало | Много |
| Простота | Высокая | Средняя | Средняя | Высокая |

### Counting Bloom Filter

Расширение для поддержки удаления:

```
Вместо битов используем счётчики:

Обычный:     [0][1][0][1][1][0][0][1][0][0]
Counting:    [0][2][0][1][3][0][0][1][0][0]
                 ↑   ↑   ↑
             2 элемента  3 элемента
             установили  установили
             этот бит    этот бит

add():      счётчик++
remove():   счётчик-- (если > 0)
contains(): все счётчики > 0?

Недостаток: больше памяти (обычно 4 бита на счётчик)
```

### Cuckoo Filter

Альтернатива с поддержкой удаления:

```
- Хранит fingerprints элементов
- Поддерживает удаление
- Лучшая производительность при низком FPR
- Более сложная реализация
```

## Примеры использования в реальных системах

### 1. LevelDB / RocksDB — фильтрация SSTable

```
Без Bloom Filter:
Запрос: GET key123
1. Проверить MemTable (в памяти) ✓
2. Проверить SSTable-1 (диск) ← дорого!
3. Проверить SSTable-2 (диск) ← дорого!
4. ...

С Bloom Filter:
Запрос: GET key123
1. Проверить MemTable ✓
2. Bloom(SSTable-1): "точно нет" → пропустить
3. Bloom(SSTable-2): "возможно есть" → проверить диск
4. Bloom(SSTable-3): "точно нет" → пропустить

Сэкономлено: 2 чтения с диска
```

```python
# Псевдокод LevelDB
class SSTable:
    def __init__(self, data):
        self.bloom = BloomFilter(len(data), 0.01)
        self.data = data
        for key in data:
            self.bloom.add(key)

    def get(self, key):
        # Быстрая проверка Bloom Filter
        if not self.bloom.contains(key):
            return None  # Точно нет — не читаем диск

        # Bloom сказал "возможно" — читаем с диска
        return self._disk_read(key)
```

### 2. Google Chrome — Safe Browsing

```
Проверка URL на вредоносность:

1. Локальный Bloom Filter (~1 MB) содержит хеши вредоносных URL

2. При открытии URL:
   if bloom.contains(url_hash):
       # "Возможно опасный" — запрос к серверу Google
       result = server.check(url)
   else:
       # "Точно безопасный" — нет сетевого запроса
       allow()

Результат:
- 99% URL не требуют сетевого запроса
- Приватность: Google не видит безопасные URL
```

### 3. Apache Cassandra — проверка существования

```java
// Cassandra SSTable с Bloom Filter
public class SSTableReader {
    private BloomFilter bloomFilter;
    private File dataFile;

    public Row getRow(DecoratedKey key) {
        // Первая проверка — в памяти
        if (!bloomFilter.isPresent(key)) {
            return null;  // Нет в этом SSTable
        }

        // Bloom сказал "возможно" — читаем
        return readFromDisk(key);
    }
}
```

### 4. Medium — рекомендации постов

```python
# Не показывать пользователю уже прочитанные посты
class RecommendationSystem:
    def __init__(self):
        # Bloom Filter для каждого пользователя
        self.read_posts = {}

    def get_recommendations(self, user_id, all_posts):
        user_bloom = self.read_posts.get(user_id, BloomFilter(10000, 0.01))

        recommendations = []
        for post in all_posts:
            if not user_bloom.contains(post.id):
                # Пользователь точно не читал этот пост
                recommendations.append(post)

        return recommendations

    def mark_read(self, user_id, post_id):
        if user_id not in self.read_posts:
            self.read_posts[user_id] = BloomFilter(10000, 0.01)
        self.read_posts[user_id].add(post_id)
```

### 5. Сетевое оборудование — маршрутизация

```
Быстрая проверка принадлежности IP к списку:

# Черный список IP (миллионы адресов)
blacklist = BloomFilter(10_000_000, 0.001)

def process_packet(packet):
    src_ip = packet.source_ip

    if blacklist.contains(src_ip):
        # "Возможно" в черном списке
        # Точная проверка (редко)
        if exact_blacklist.contains(src_ip):
            drop(packet)
            return

    # Точно не в черном списке — быстрый путь
    forward(packet)
```

### 6. Bitcoin SPV-кошельки

```
SPV-кошелёк хочет получить только свои транзакции:

1. Кошелёк создаёт Bloom Filter со своими адресами
2. Отправляет фильтр полной ноде
3. Нода отправляет только транзакции, соответствующие фильтру

Преимущества:
- Экономия трафика
- Частичная приватность (FP добавляет "шум")
```

## Вариации и расширения

### Scalable Bloom Filter

Динамически растущий фильтр:

```python
class ScalableBloomFilter:
    def __init__(self, initial_capacity, error_rate, scale=2):
        self.filters = []
        self.scale = scale
        self.error_rate = error_rate
        self._add_filter(initial_capacity)

    def _add_filter(self, capacity):
        # Каждый следующий фильтр имеет меньший error rate
        # чтобы общий FPR оставался приемлемым
        filter_error = self.error_rate * (0.5 ** len(self.filters))
        self.filters.append(BloomFilter(capacity, filter_error))

    def add(self, item):
        # Добавляем в последний фильтр
        current_filter = self.filters[-1]
        if current_filter.is_full():
            self._add_filter(current_filter.capacity * self.scale)
            current_filter = self.filters[-1]
        current_filter.add(item)

    def contains(self, item):
        # Проверяем все фильтры
        return any(f.contains(item) for f in self.filters)
```

### Partitioned Bloom Filter

Разделённый для лучшей cache-эффективности:

```
Вместо одного массива m бит используем k массивов по m/k бит:

Обычный:
hash1, hash2, hash3 → [___________________________]
                           все хеши в одном массиве

Partitioned:
hash1 → [_________]
hash2 → [_________]
hash3 → [_________]

Каждый хеш отвечает за свой сегмент
```

## Резюме

Фильтр Блума — это компактная вероятностная структура для проверки членства с гарантией отсутствия ложноотрицательных результатов. Ключевые особенности:

1. **Компактность** — на порядки меньше памяти, чем точные структуры
2. **Скорость** — O(k) для всех операций, где k мало
3. **Trade-off** — память vs точность (ложноположительные)
4. **Нет удаления** — базовый Bloom Filter не поддерживает remove()

Фильтры Блума незаменимы в системах, где:
- Нужна быстрая предварительная проверка перед дорогой операцией
- Допустимы редкие "лишние" проверки (false positives)
- Критична экономия памяти/сетевого трафика

Широко применяется в базах данных (LevelDB, Cassandra), сетевых системах (Chrome Safe Browsing), и распределённых системах.
