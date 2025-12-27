# Итераторы (Iterators)

## Iterable vs Iterator

| Термин | Что это | Примеры |
|--------|---------|---------|
| **Iterable** | Объект, по которому можно итерироваться | `list`, `str`, `dict`, `range` |
| **Iterator** | Объект с состоянием, выдаёт элементы по одному | результат `iter()` |

```python
my_list = [1, 2, 3]        # Iterable
my_iter = iter(my_list)    # Iterator

next(my_iter)  # 1
next(my_iter)  # 2
next(my_iter)  # 3
next(my_iter)  # StopIteration!
```

## Как работает for под капотом

```python
for item in [1, 2, 3]:
    print(item)

# Эквивалентно:
iterator = iter([1, 2, 3])
while True:
    try:
        item = next(iterator)
        print(item)
    except StopIteration:
        break
```

## Протокол итератора

**Iterable** — имеет метод `__iter__()`, возвращающий итератор.

**Iterator** — имеет методы:
- `__iter__()` — возвращает себя
- `__next__()` — возвращает следующий элемент или `StopIteration`

## Создание своего итератора (класс)

```python
class Counter:
    def __init__(self, start, end):
        self.current = start
        self.end = end

    def __iter__(self):
        return self

    def __next__(self):
        if self.current >= self.end:
            raise StopIteration
        self.current += 1
        return self.current - 1

for num in Counter(1, 4):
    print(num)  # 1, 2, 3
```

## Генераторы — простой способ создать итератор

Функция с `yield` вместо `return`:

```python
def counter(start, end):
    current = start
    while current < end:
        yield current
        current += 1

for num in counter(1, 4):
    print(num)  # 1, 2, 3
```

## yield vs return

| `return` | `yield` |
|----------|---------|
| Завершает функцию | Приостанавливает |
| Одно значение | Много значений по одному |
| — | Сохраняет состояние |

```python
def gen():
    print("Шаг 1")
    yield 1
    print("Шаг 2")
    yield 2

g = gen()
next(g)  # Шаг 1 → 1
next(g)  # Шаг 2 → 2
```

## Generator Expression

Ленивая версия list comprehension:

```python
# List — сразу в памяти
squares_list = [x**2 for x in range(1000000)]

# Generator — вычисляет по требованию
squares_gen = (x**2 for x in range(1000000))
```

## Зачем нужны итераторы и генераторы?

1. **Экономия памяти** — не хранят все элементы
2. **Ленивые вычисления** — только когда нужно
3. **Бесконечные последовательности**

```python
def infinite_counter(start=0):
    while True:
        yield start
        start += 1

counter = infinite_counter()
next(counter)  # 0
next(counter)  # 1
# ... бесконечно
```

## itertools — полезные функции

```python
from itertools import count, cycle, repeat, chain, islice

# count — бесконечный счётчик
list(islice(count(10), 3))  # [10, 11, 12]

# cycle — бесконечно повторяет iterable
colors = cycle(['red', 'green', 'blue'])

# chain — объединяет итераторы
list(chain([1, 2], [3, 4]))  # [1, 2, 3, 4]

# islice — срез итератора
list(islice(range(100), 5, 10))  # [5, 6, 7, 8, 9]
```

## yield — углублённо

Генератор — это "замороженная" функция, которая приостанавливается на каждом `yield`:

```python
def gen():
    print("Старт")
    yield 1
    print("После первого yield")
    yield 2

g = gen()       # Ничего не выполняется
next(g)         # Старт → 1, замирает
next(g)         # После первого yield → 2, замирает
next(g)         # StopIteration
```

## yield from — делегирование

Передаёт управление другому итератору:

```python
# Без yield from
def gen():
    for item in [1, 2, 3]:
        yield item

# С yield from — проще
def gen():
    yield from [1, 2, 3]
    yield from [4, 5, 6]

list(gen())  # [1, 2, 3, 4, 5, 6]
```

### Вложенные генераторы

```python
def inner():
    yield 'a'
    yield 'b'

def outer():
    yield 1
    yield from inner()
    yield 2

list(outer())  # [1, 'a', 'b', 2]
```

### Рекурсия с yield from

```python
def flatten(nested):
    for item in nested:
        if isinstance(item, list):
            yield from flatten(item)
        else:
            yield item

nested = [1, [2, 3, [4, 5]], 6]
list(flatten(nested))  # [1, 2, 3, 4, 5, 6]
```

## send() — двусторонняя коммуникация

Отправка значений **в** генератор:

```python
def gen():
    received = yield 1
    print(f"Получил: {received}")
    received = yield 2
    print(f"Получил: {received}")

g = gen()
next(g)          # 1 (первый вызов — обязательно next или send(None))
g.send(100)      # Получил: 100 → 2
g.send(200)      # Получил: 200 → StopIteration
```

### Пример — скользящее среднее

```python
def running_average():
    total = 0
    count = 0
    average = None
    while True:
        value = yield average
        total += value
        count += 1
        average = total / count

avg = running_average()
next(avg)       # Запуск (None)
avg.send(10)    # 10.0
avg.send(20)    # 15.0
avg.send(30)    # 20.0
```

## throw() — отправка исключений

```python
def gen():
    try:
        yield 1
        yield 2
    except ValueError:
        print("Поймал ValueError!")
        yield "recovered"

g = gen()
next(g)                 # 1
g.throw(ValueError)     # Поймал ValueError! → recovered
```

## close() — закрытие генератора

Вызывает `GeneratorExit` внутри генератора:

```python
def gen():
    try:
        yield 1
        yield 2
    finally:
        print("Очистка ресурсов")

g = gen()
next(g)     # 1
g.close()   # Очистка ресурсов
```

## return в генераторах

`return` завершает генератор, значение доступно через `StopIteration.value`:

```python
def gen():
    yield 1
    return "Готово!"

g = gen()
next(g)  # 1
try:
    next(g)
except StopIteration as e:
    print(e.value)  # Готово!
```

### yield from захватывает return

```python
def inner():
    yield 1
    return "результат"

def outer():
    result = yield from inner()
    print(f"inner вернул: {result}")
    yield 2

list(outer())
# inner вернул: результат
# [1, 2]
```

## Сводная таблица генераторов

| Конструкция | Что делает |
|-------------|-----------|
| `yield x` | Отдаёт значение, приостанавливает |
| `yield from iter` | Делегирует другому iterable |
| `next(gen)` | Возобновляет до следующего yield |
| `gen.send(x)` | Возобновляет и передаёт значение |
| `gen.throw(exc)` | Вбрасывает исключение |
| `gen.close()` | Завершает генератор |
| `return x` | Завершает, значение в StopIteration.value |

## Q&A

**Q: В чём разница между iterable и iterator?**
A: Iterable можно передать в `iter()` чтобы получить iterator. Iterator имеет метод `__next__()` и помнит текущую позицию.

**Q: Зачем генераторы, если есть списки?**
A: Генераторы экономят память — не хранят все элементы сразу. Важно для больших данных.

**Q: Можно ли итерироваться по итератору дважды?**
A: Нет. Итератор "исчерпывается". Для повторной итерации нужно создать новый.

**Q: Что такое ленивые вычисления?**
A: Значения вычисляются только когда запрашиваются, а не заранее.
