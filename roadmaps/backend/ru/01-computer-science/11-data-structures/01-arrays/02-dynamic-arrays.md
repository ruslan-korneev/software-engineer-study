# Динамические массивы (Dynamic Arrays)

[prev: 01-static-arrays](./01-static-arrays.md) | [next: 03-array-operations](./03-array-operations.md)

---
## Определение

**Динамический массив** — это структура данных, которая представляет собой массив с возможностью автоматического изменения размера. При добавлении элементов сверх текущей ёмкости массив автоматически расширяется, выделяя больший блок памяти.

Примеры реализаций:
- **Python**: `list`
- **Java**: `ArrayList`
- **C++**: `std::vector`
- **JavaScript**: `Array`
- **C#**: `List<T>`

## Зачем нужно

### Практическое применение:
- **Коллекции неизвестного размера** — чтение данных из файла, пользовательский ввод
- **Стеки и очереди** — реализация на основе динамического массива
- **Буферы с переменным размером** — накопление данных перед обработкой
- **Результаты фильтрации** — когда заранее неизвестно количество подходящих элементов
- **Общего назначения** — самая часто используемая структура данных

## Как работает

### Концепция capacity vs size

```
Capacity (ёмкость) = 8   (выделено памяти)
Size (размер) = 5        (занято элементами)

┌───┬───┬───┬───┬───┬───┬───┬───┐
│ A │ B │ C │ D │ E │   │   │   │
└───┴───┴───┴───┴───┴───┴───┴───┘
  0   1   2   3   4   5   6   7
  ▲───────────────▲   ▲───────▲
       size = 5      свободно = 3
```

### Расширение массива (Resizing)

Когда `size == capacity` и нужно добавить элемент:

```
ШАГ 1: Исходный массив (capacity = 4, size = 4)
┌───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │  ← массив полон
└───┴───┴───┴───┘

ШАГ 2: Выделение нового массива (capacity = 8)
┌───┬───┬───┬───┬───┬───┬───┬───┐
│   │   │   │   │   │   │   │   │  ← новый массив
└───┴───┴───┴───┴───┴───┴───┴───┘

ШАГ 3: Копирование элементов
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │   │   │   │   │  ← элементы скопированы
└───┴───┴───┴───┴───┴───┴───┴───┘

ШАГ 4: Добавление нового элемента
┌───┬───┬───┬───┬───┬───┬───┬───┐
│ 1 │ 2 │ 3 │ 4 │ 5 │   │   │   │  ← новый элемент добавлен
└───┴───┴───┴───┴───┴───┴───┴───┘

ШАГ 5: Освобождение старого массива
(старая память возвращается системе)
```

### Коэффициент расширения (Growth Factor)

Обычно используется коэффициент **2** (удвоение):
- Python list: ~1.125 (более экономный)
- Java ArrayList: 1.5
- C++ vector: 2

## Псевдокод основных операций

### Структура данных
```python
class DynamicArray:
    def __init__(self, initial_capacity=4):
        self.capacity = initial_capacity
        self.size = 0
        self.data = allocate_array(self.capacity)
```

### Добавление в конец (Append / Push)
```python
function APPEND(array, element):
    # Проверка необходимости расширения
    if array.size == array.capacity:
        RESIZE(array, array.capacity * 2)

    # Добавление элемента
    array.data[array.size] = element
    array.size += 1

function RESIZE(array, new_capacity):
    new_data = allocate_array(new_capacity)

    # Копирование существующих элементов
    for i from 0 to array.size - 1:
        new_data[i] = array.data[i]

    free(array.data)  # освобождение старой памяти
    array.data = new_data
    array.capacity = new_capacity
```

### Доступ по индексу (Get)
```python
function GET(array, index):
    if index < 0 OR index >= array.size:
        raise IndexOutOfBoundsError
    return array.data[index]
```

### Изменение элемента (Set)
```python
function SET(array, index, value):
    if index < 0 OR index >= array.size:
        raise IndexOutOfBoundsError
    array.data[index] = value
```

### Вставка по индексу (Insert)
```python
function INSERT(array, index, element):
    if index < 0 OR index > array.size:
        raise IndexOutOfBoundsError

    # Расширение при необходимости
    if array.size == array.capacity:
        RESIZE(array, array.capacity * 2)

    # Сдвиг элементов вправо
    for i from array.size - 1 down to index:
        array.data[i + 1] = array.data[i]

    array.data[index] = element
    array.size += 1
```

### Удаление по индексу (Remove)
```python
function REMOVE(array, index):
    if index < 0 OR index >= array.size:
        raise IndexOutOfBoundsError

    removed = array.data[index]

    # Сдвиг элементов влево
    for i from index to array.size - 2:
        array.data[i] = array.data[i + 1]

    array.size -= 1

    # Опционально: сжатие при малом заполнении
    if array.size < array.capacity / 4:
        RESIZE(array, array.capacity / 2)

    return removed
```

### Удаление из конца (Pop)
```python
function POP(array):
    if array.size == 0:
        raise EmptyArrayError

    array.size -= 1
    return array.data[array.size]
```

## Анализ сложности

| Операция | Среднее время | Худшее время | Пояснение |
|----------|---------------|--------------|-----------|
| Доступ по индексу | O(1) | O(1) | Прямой доступ |
| Изменение элемента | O(1) | O(1) | Прямой доступ |
| Append (в конец) | **O(1)*** | O(n) | *Амортизированное |
| Insert (в начало) | O(n) | O(n) | Сдвиг всех элементов |
| Insert (в середину) | O(n) | O(n) | Сдвиг части элементов |
| Remove (из конца) | O(1) | O(1) | Без сдвига |
| Remove (из начала) | O(n) | O(n) | Сдвиг всех элементов |
| Поиск | O(n) | O(n) | Линейный перебор |

### Амортизированный анализ Append

```
Операции:    1   2   3   4   5   6   7   8   9  ...
Capacity:    1   2   4   4   8   8   8   8   16 ...
Resize:      ✓   ✓   ✓       ✓               ✓

Стоимость resize при n операциях:
1 + 2 + 4 + 8 + ... + n = 2n - 1 ≈ O(n)

Амортизированная стоимость одной операции:
O(n) / n = O(1)
```

### Память
- **Пространственная сложность**: O(n)
- **Накладные расходы**: до 50% (при size = capacity/2 + 1)

## Примеры с разбором

### Пример 1: Чтение строк из файла
```python
lines = []  # динамический массив

with open("data.txt") as f:
    for line in f:
        lines.append(line.strip())

# Размер файла заранее неизвестен
# Массив автоматически расширяется по мере чтения
print(f"Прочитано {len(lines)} строк")
```

### Пример 2: Фильтрация данных
```python
numbers = [1, 2, 3, 4, 5, 6, 7, 8, 9, 10]
evens = []  # результат неизвестного размера

for n in numbers:
    if n % 2 == 0:
        evens.append(n)

# evens = [2, 4, 6, 8, 10]
```

### Пример 3: Стек на основе динамического массива
```python
class Stack:
    def __init__(self):
        self.items = []  # динамический массив

    def push(self, item):
        self.items.append(item)  # O(1) амортизированное

    def pop(self):
        if not self.items:
            raise Exception("Stack is empty")
        return self.items.pop()  # O(1)

    def peek(self):
        if not self.items:
            raise Exception("Stack is empty")
        return self.items[-1]  # O(1)
```

### Пример 4: Анализ роста capacity
```python
import sys

arr = []
prev_size = 0

for i in range(100):
    arr.append(i)
    current_size = sys.getsizeof(arr)

    if current_size != prev_size:
        print(f"len={len(arr):3d}, size={current_size} bytes")
        prev_size = current_size

# Вывод (примерный):
# len=  1, size= 88 bytes
# len=  5, size=120 bytes
# len=  9, size=184 bytes
# len= 17, size=248 bytes
# ...
```

## Типичные ошибки

### 1. Модификация во время итерации
```python
arr = [1, 2, 3, 4, 5]

# НЕПРАВИЛЬНО — пропуск элементов или IndexError
for i in range(len(arr)):
    if arr[i] % 2 == 0:
        arr.pop(i)

# ПРАВИЛЬНО — обратная итерация
for i in range(len(arr) - 1, -1, -1):
    if arr[i] % 2 == 0:
        arr.pop(i)

# ЛУЧШЕ — list comprehension
arr = [x for x in arr if x % 2 != 0]
```

### 2. Непонимание амортизации
```python
# НЕПРАВИЛЬНОЕ ожидание: "append всегда O(1)"
# На самом деле: иногда O(n) из-за resize

# Для критичного по времени кода:
arr = [None] * expected_size  # предварительное выделение
# или
arr = []
arr.extend([None] * expected_size)
```

### 3. Вставка в начало — O(n)!
```python
arr = list(range(1000000))

# НЕПРАВИЛЬНО (медленно!) — O(n) каждый раз
for item in new_items:
    arr.insert(0, item)  # сдвигает миллион элементов!

# ПРАВИЛЬНО — использовать deque для частых вставок в начало
from collections import deque
d = deque(arr)
d.appendleft(item)  # O(1)
```

### 4. Утечка памяти при удалении
```python
# В некоторых реализациях:
arr = [large_object] * 1000
arr.pop()  # размер уменьшился, но память не освободилась

# Принудительное сжатие (если нужно):
arr = arr.copy()  # создаёт новый массив точного размера
```

### 5. Shallow vs Deep copy
```python
# НЕПРАВИЛЬНО — изменяет оригинал
original = [[1, 2], [3, 4]]
copy = original[:]  # shallow copy
copy[0][0] = 999
print(original[0][0])  # 999 — оригинал изменился!

# ПРАВИЛЬНО
import copy
deep = copy.deepcopy(original)
deep[0][0] = 999
print(original[0][0])  # 1 — оригинал не изменился
```

## Сравнение реализаций

| Язык | Класс | Growth Factor | Особенности |
|------|-------|---------------|-------------|
| Python | list | ~1.125 | Overallocation для мелких списков |
| Java | ArrayList | 1.5 | Явный ensureCapacity() |
| C++ | vector | 2 | reserve(), shrink_to_fit() |
| C# | List<T> | 2 | Capacity property |
| Go | slice | 2 (при малых), 1.25 | Встроенный тип |

## Когда НЕ использовать динамический массив

1. **Частые вставки/удаления в начале** → используй `deque`
2. **Частые вставки/удаления в середине** → используй связный список
3. **Нужен быстрый поиск по значению** → используй `set` или `dict`
4. **Критичная детерминированность времени** → используй статический массив
5. **Очень большие данные с частым resize** → оценивай capacity заранее

---

[prev: 01-static-arrays](./01-static-arrays.md) | [next: 03-array-operations](./03-array-operations.md)
