# Списки с пропусками (Skip Lists)

## Определение

**Skip List** (список с пропусками) — это вероятностная структура данных, представляющая собой многоуровневый связный список, где элементы на верхних уровнях позволяют "перепрыгивать" через множество элементов нижних уровней, обеспечивая быстрый поиск.

```
Уровень 3:  HEAD ─────────────────────────────→ 50 ────────────────────→ NIL
                                                 │
Уровень 2:  HEAD ────────→ 20 ────────────────→ 50 ────────→ 80 ──────→ NIL
                            │                    │            │
Уровень 1:  HEAD ────→ 10 → 20 → 30 ────────→ 50 → 60 ────→ 80 ──────→ NIL
                       │    │    │            │    │        │
Уровень 0:  HEAD → 5 → 10 → 20 → 30 → 40 → 50 → 60 → 70 → 80 → 90 → NIL
```

Skip List был изобретён Уильямом Пью в 1989 году как простая альтернатива сбалансированным деревьям.

## Зачем нужно

### Проблемы альтернатив

```
Связный список:
+ Простая реализация
- Поиск O(n)

Сбалансированные деревья (AVL, Red-Black):
+ Поиск O(log n)
- Сложная реализация балансировки
- Сложное удаление
- Трудно распараллелить
```

### Преимущества Skip List

```
+ Простая реализация (проще Red-Black tree)
+ Средний поиск O(log n)
+ Легко реализовать concurrent версию
+ Хорошая локальность памяти (последовательное выделение)
+ Простые вставка и удаление
+ Эффективные range-запросы
```

### Применение

- **Redis**: отсортированные множества (Sorted Sets)
- **LevelDB/RocksDB**: MemTable
- **Apache Lucene**: некоторые индексы
- **HBase**: MemStore
- **Concurrent контейнеры**: ConcurrentSkipListMap в Java

## Как работает

### Идея многоуровневости

```
Обычный связный список — линейный поиск:

HEAD → 5 → 10 → 20 → 30 → 40 → 50 → 60 → 70 → 80 → 90
Поиск 80: нужно пройти 8 элементов


Добавляем "экспресс-полосы":

Уровень 1: HEAD ────────→ 20 ────────────────→ 50 ────────────────→ 80 → NIL
                          │                    │                    │
Уровень 0: HEAD → 5 → 10 → 20 → 30 → 40 → 50 → 60 → 70 → 80 → 90 → NIL

Поиск 80:
1. Уровень 1: HEAD → 20 (< 80) → 50 (< 80) → 80 (найден!)
2. Если не нашли, спускаемся на уровень 0

Шагов: 3 вместо 8
```

### Вероятностный выбор уровней

При вставке каждый элемент получает случайный уровень:

```
Алгоритм определения уровня:

level = 0
while random() < p and level < MAX_LEVEL:
    level += 1
return level

При p = 0.5:
- 50% элементов на уровне 0
- 25% на уровнях 0 и 1
- 12.5% на уровнях 0, 1 и 2
- и т.д.
```

```
Распределение элементов по уровням (p = 0.5, n = 16):

Уровень 4: ■                                    (1 элемент)
Уровень 3: ■     ■                              (2 элемента)
Уровень 2: ■  ■  ■  ■                           (4 элемента)
Уровень 1: ■ ■■ ■■ ■■ ■■                         (8 элементов)
Уровень 0: ■■■■■■■■■■■■■■■■                      (16 элементов)
```

### Ожидаемое число уровней

```
E[уровней] ≈ log_{1/p}(n)

Для n = 1,000,000 и p = 0.5:
E[уровней] ≈ log_2(1,000,000) ≈ 20

Для n = 1,000,000 и p = 0.25:
E[уровней] ≈ log_4(1,000,000) ≈ 10
```

## Инварианты и свойства

### Структурные свойства

1. **Уровень 0 содержит все элементы** (полный связный список)
2. **Уровень i — подмножество уровня i-1**
3. **Элементы отсортированы** на каждом уровне
4. **Случайная высота** для каждого элемента

### Вероятностные гарантии

```
С высокой вероятностью:
- Высота = O(log n)
- Поиск = O(log n)
- Вставка = O(log n)
- Удаление = O(log n)

Худший случай (очень маловероятен):
- Все элементы на уровне 0 → O(n)
```

### Ожидаемое потребление памяти

```
Каждый элемент занимает в среднем 1/(1-p) указателей

При p = 0.5: 2 указателя на элемент
При p = 0.25: 1.33 указателя на элемент

Общая память: O(n)
```

## Псевдокод основных операций

### Структура узла

```python
import random

class SkipListNode:
    def __init__(self, key, level):
        self.key = key
        # Массив указателей на следующие узлы на каждом уровне
        self.forward = [None] * (level + 1)
```

### Структура Skip List

```python
class SkipList:
    def __init__(self, max_level=16, p=0.5):
        self.max_level = max_level
        self.p = p
        self.level = 0  # Текущий максимальный уровень
        # HEAD — специальный узел
        self.head = SkipListNode(float('-inf'), max_level)

    def random_level(self):
        """Генерирует случайный уровень для нового узла"""
        level = 0
        while random.random() < self.p and level < self.max_level:
            level += 1
        return level
```

### Поиск

```python
def search(self, key):
    """
    Поиск элемента в Skip List

    Время: O(log n) в среднем
    """
    current = self.head

    # Начинаем с верхнего уровня
    for i in range(self.level, -1, -1):
        # Двигаемся вправо, пока следующий элемент < key
        while (current.forward[i] and
               current.forward[i].key < key):
            current = current.forward[i]

    # Переходим на уровень 0
    current = current.forward[0]

    if current and current.key == key:
        return current
    return None
```

### Визуализация поиска

```
Поиск ключа 70:

Уровень 3: HEAD ─────────────────────→ 50 ─────────────→ NIL
                                        │
           50 < 70, следующий None     ↓ спуск

Уровень 2: HEAD ────→ 20 ────────────→ 50 ────→ 80 ──→ NIL
                                        │        │
           50 < 70, 80 > 70            ↓ спуск  X

Уровень 1: HEAD → 10 → 20 → 30 ─────→ 50 → 60 → 80 → NIL
                                        │    │
           50 < 70, 60 < 70            ─────→ ↓

Уровень 0: HEAD → 5 → ... → 50 → 60 → 70 → 80 → NIL
                                       ↑
                                   Найден!

Путь: 50(L3) → 50(L2) → 60(L1) → 70(L0)
```

### Вставка

```python
def insert(self, key):
    """
    Вставка элемента в Skip List

    Время: O(log n) в среднем
    """
    # update[i] — последний узел на уровне i перед позицией вставки
    update = [None] * (self.max_level + 1)
    current = self.head

    # Находим позицию для вставки (как в поиске)
    for i in range(self.level, -1, -1):
        while (current.forward[i] and
               current.forward[i].key < key):
            current = current.forward[i]
        update[i] = current

    current = current.forward[0]

    # Если элемент уже существует — обновляем
    if current and current.key == key:
        return  # или обновить значение

    # Генерируем случайный уровень для нового узла
    new_level = self.random_level()

    # Если новый уровень выше текущего максимума
    if new_level > self.level:
        for i in range(self.level + 1, new_level + 1):
            update[i] = self.head
        self.level = new_level

    # Создаём новый узел
    new_node = SkipListNode(key, new_level)

    # Вставляем узел на каждом уровне
    for i in range(new_level + 1):
        new_node.forward[i] = update[i].forward[i]
        update[i].forward[i] = new_node
```

### Визуализация вставки

```
Вставляем 35 (случайный уровень = 2):

До вставки:
Уровень 2: HEAD ────→ 20 ─────────────→ 50 ──→ NIL
Уровень 1: HEAD → 10 → 20 → 30 ───────→ 50 ──→ NIL
Уровень 0: HEAD → 10 → 20 → 30 → 40 → 50 ──→ NIL

update массив после поиска позиции:
update[0] = 30
update[1] = 30
update[2] = 20

После вставки:
Уровень 2: HEAD ────→ 20 ─────→ 35 ────→ 50 ──→ NIL
Уровень 1: HEAD → 10 → 20 → 30 → 35 ───→ 50 ──→ NIL
Уровень 0: HEAD → 10 → 20 → 30 → 35 → 40 → 50 ──→ NIL
                              ↑
                          Новый узел
```

### Удаление

```python
def delete(self, key):
    """
    Удаление элемента из Skip List

    Время: O(log n) в среднем
    """
    update = [None] * (self.max_level + 1)
    current = self.head

    # Находим узел (как в поиске)
    for i in range(self.level, -1, -1):
        while (current.forward[i] and
               current.forward[i].key < key):
            current = current.forward[i]
        update[i] = current

    current = current.forward[0]

    # Если элемент найден — удаляем
    if current and current.key == key:
        # Обновляем указатели на каждом уровне
        for i in range(self.level + 1):
            if update[i].forward[i] != current:
                break
            update[i].forward[i] = current.forward[i]

        # Уменьшаем уровень, если нужно
        while self.level > 0 and self.head.forward[self.level] is None:
            self.level -= 1
```

### Визуализация удаления

```
Удаляем 35:

До удаления:
Уровень 2: HEAD → 20 ─────→ 35 ────→ 50 → NIL
Уровень 1: HEAD → 20 → 30 → 35 ────→ 50 → NIL
Уровень 0: HEAD → 20 → 30 → 35 → 40 → 50 → NIL

update[0] = 30
update[1] = 30
update[2] = 20

После удаления:
Уровень 2: HEAD → 20 ─────────────→ 50 → NIL
Уровень 1: HEAD → 20 → 30 ────────→ 50 → NIL
Уровень 0: HEAD → 20 → 30 → 40 → 50 → NIL
```

### Range Query

```python
def range_query(self, start, end):
    """
    Получение всех элементов в диапазоне [start, end]

    Время: O(log n + k), где k — число элементов в диапазоне
    """
    results = []
    current = self.head

    # Находим начало диапазона
    for i in range(self.level, -1, -1):
        while (current.forward[i] and
               current.forward[i].key < start):
            current = current.forward[i]

    current = current.forward[0]

    # Сканируем до конца диапазона
    while current and current.key <= end:
        results.append(current.key)
        current = current.forward[0]

    return results
```

## Анализ сложности

| Операция | Среднее | Худшее |
|----------|---------|--------|
| Поиск    | O(log n) | O(n) |
| Вставка  | O(log n) | O(n) |
| Удаление | O(log n) | O(n) |
| Range Query | O(log n + k) | O(n) |
| Память   | O(n) | O(n log n) |

### Доказательство O(log n) для поиска

```
Теорема: Ожидаемое число шагов при поиске = O(log n)

Доказательство (идея):
1. На каждом уровне мы делаем ожидаемое O(1/p) шагов
2. Число уровней ≈ log_{1/p}(n)
3. Общее число шагов = O((1/p) * log_{1/p}(n)) = O(log n)

При p = 0.5:
Шагов на уровень ≈ 2
Уровней ≈ log_2(n)
Всего ≈ 2 * log_2(n) шагов
```

### Выбор параметра p

```
p = 0.5:
+ Меньше уровней
+ Быстрее поиск
- Больше памяти (2 указателя на элемент)

p = 0.25:
+ Меньше памяти (1.33 указателя)
- Больше уровней
- Чуть медленнее поиск

Рекомендация:
- p = 0.5 для скорости
- p = 0.25 для экономии памяти
```

## Сравнение с альтернативами

| Свойство | Skip List | Red-Black | AVL | Hash Table |
|----------|-----------|-----------|-----|------------|
| Поиск (avg) | O(log n) | O(log n) | O(log n) | O(1) |
| Поиск (worst) | O(n) | O(log n) | O(log n) | O(n) |
| Вставка (avg) | O(log n) | O(log n) | O(log n) | O(1) |
| Удаление (avg) | O(log n) | O(log n) | O(log n) | O(1) |
| Range Query | O(log n + k) | O(n) | O(n) | O(n) |
| Сложность кода | Низкая | Высокая | Средняя | Низкая |
| Concurrency | Легко | Сложно | Сложно | Средне |
| Упорядоченность | Да | Да | Да | Нет |

### Преимущества Skip List

```
1. Простота реализации
   - Нет сложной балансировки
   - Легко отлаживать

2. Concurrent доступ
   - Lock-free реализации проще
   - Меньше конфликтов

3. Range queries
   - Естественная поддержка
   - O(log n + k)

4. Память
   - Хорошая локальность
   - Нет накладных расходов на балансировку
```

### Недостатки Skip List

```
1. Вероятностные гарантии
   - Худший случай O(n)
   - Нет детерминированных границ

2. Кэш-эффективность
   - Хуже, чем у массивов
   - Много pointer chasing

3. Накладные расходы
   - Дополнительные указатели
   - Генерация случайных чисел
```

## Примеры использования в реальных системах

### 1. Redis — Sorted Sets

```redis
# Redis использует Skip List для ZSET
ZADD leaderboard 100 "player1"
ZADD leaderboard 200 "player2"
ZADD leaderboard 150 "player3"

# Range query — эффективен благодаря Skip List
ZRANGEBYSCORE leaderboard 100 200
# Результат: ["player1", "player3", "player2"]

# Ранжирование
ZRANK leaderboard "player3"  # => 1
```

```
Внутренняя структура Redis ZSET:

Level 3: HEAD ─────────────────────→ player2(200) → NIL
Level 2: HEAD ─────→ player3(150) ─→ player2(200) → NIL
Level 1: HEAD → player1(100) → player3(150) → player2(200) → NIL
```

### 2. LevelDB/RocksDB — MemTable

```cpp
// LevelDB использует Skip List для MemTable
// (буфер записи в памяти)

class MemTable {
  SkipList<Slice, Comparator> table_;

 public:
  void Add(const Slice& key, const Slice& value) {
    // O(log n) вставка в Skip List
    table_.Insert(EncodeEntry(key, value));
  }

  bool Get(const Slice& key, std::string* value) {
    // O(log n) поиск
    auto iter = table_.Seek(key);
    // ...
  }
};
```

### 3. Java ConcurrentSkipListMap

```java
import java.util.concurrent.ConcurrentSkipListMap;

// Thread-safe sorted map на основе Skip List
ConcurrentSkipListMap<Integer, String> map =
    new ConcurrentSkipListMap<>();

map.put(3, "three");
map.put(1, "one");
map.put(2, "two");

// Range query
SortedMap<Integer, String> subMap = map.subMap(1, 3);
// {1=one, 2=two}

// Concurrent operations безопасны
ExecutorService executor = Executors.newFixedThreadPool(10);
for (int i = 0; i < 1000; i++) {
    final int key = i;
    executor.submit(() -> map.put(key, "value" + key));
}
```

### 4. Apache Lucene

```
Lucene использует Skip List для:
- Пропуска документов при обходе posting lists
- Эффективного пересечения списков при AND queries

Posting List с Skip Pointers:
doc_1 → doc_3 → doc_5 → doc_8 → doc_12 → ...
  ↓              ↓               ↓
Skip: doc_5    Skip: doc_12   Skip: doc_25

При поиске документа 10:
1. Skip до doc_5 (< 10)
2. Skip до doc_12 (> 10, не подходит)
3. Линейно от doc_5: doc_5 → doc_8 → doc_12
4. doc_8 < 10 < doc_12, значит doc_10 не существует
```

## Concurrent Skip List

Skip List легко адаптировать для многопоточности:

```python
import threading

class ConcurrentSkipList:
    def __init__(self, max_level=16, p=0.5):
        self.max_level = max_level
        self.p = p
        self.head = SkipListNode(float('-inf'), max_level)
        self.lock = threading.Lock()  # Простой вариант

    def insert(self, key):
        with self.lock:
            # ... стандартная вставка
            pass

# Lock-free вариант использует CAS операции
class LockFreeSkipList:
    def insert(self, key):
        while True:
            # 1. Найти позицию
            update, current = self._find(key)

            # 2. Попробовать вставить атомарно
            new_node = SkipListNode(key, self.random_level())

            # CAS: compare-and-swap
            if self._cas_insert(update, new_node):
                break
            # Иначе retry
```

### Преимущества для concurrency

```
1. Локальные модификации
   - Изменения затрагивают только соседние узлы
   - Меньше конфликтов, чем в деревьях

2. Простые retry
   - При неудаче CAS легко повторить
   - Нет сложной перебалансировки

3. Нет блокировки всего дерева
   - В Red-Black ротации могут затронуть много узлов
   - Skip List: только локальные указатели
```

## Резюме

Skip List — это элегантная вероятностная структура данных, которая обеспечивает O(log n) операции при значительно более простой реализации, чем сбалансированные деревья. Ключевые особенности:

1. **Многоуровневая структура** — "экспресс-полосы" для быстрого поиска
2. **Вероятностная балансировка** — случайный выбор высоты узлов
3. **Простота** — нет сложных ротаций и перебалансировок
4. **Concurrency-friendly** — легко реализовать lock-free версию
5. **Эффективные range queries** — естественная поддержка упорядоченности

Skip List широко используется в высоконагруженных системах (Redis, LevelDB) благодаря сочетанию производительности и простоты реализации.
