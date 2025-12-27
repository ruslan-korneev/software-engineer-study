# Lambda-функции

## Что такое lambda?

**Lambda** — это анонимная (безымянная) функция, записываемая в одну строку.

```python
# Обычная функция
def add(x, y):
    return x + y

# То же самое через lambda
add = lambda x, y: x + y
```

## Синтаксис

```python
lambda аргументы: выражение
```

- Нет `def` и имени функции
- Нет `return` — результат возвращается автоматически
- **Только одно выражение** (не блок кода)

## Примеры

```python
# Квадрат числа
square = lambda x: x ** 2
square(5)  # 25

# Проверка на чётность
is_even = lambda x: x % 2 == 0
is_even(4)  # True

# Несколько аргументов
multiply = lambda a, b, c: a * b * c
multiply(2, 3, 4)  # 24

# Без аргументов
greet = lambda: "Привет!"
```

## Главное применение — функции высшего порядка

### map — применить функцию к каждому элементу

```python
numbers = [1, 2, 3, 4, 5]
doubled = list(map(lambda x: x * 2, numbers))
# [2, 4, 6, 8, 10]
```

### filter — отфильтровать элементы

```python
numbers = [1, 2, 3, 4, 5]
evens = list(filter(lambda x: x % 2 == 0, numbers))
# [2, 4]
```

### sorted — кастомная сортировка

```python
words = ['banana', 'apple', 'cherry']
sorted(words, key=lambda w: len(w))
# ['apple', 'banana', 'cherry']

# Сортировка словарей по полю
users = [{'name': 'Bob', 'age': 25}, {'name': 'Alice', 'age': 30}]
sorted(users, key=lambda u: u['age'])
# [{'name': 'Bob', 'age': 25}, {'name': 'Alice', 'age': 30}]
```

## Ограничения lambda

1. **Только одно выражение** — нельзя несколько строк
2. **Нет присваиваний** внутри
3. **Плохая читаемость** при сложной логике

```python
# ❌ Плохо — сложная lambda
result = list(map(lambda x: x ** 2 if x > 0 else -x ** 2 if x < 0 else 0, nums))

# ✅ Лучше — обычная функция
def transform(x):
    if x > 0:
        return x ** 2
    elif x < 0:
        return -x ** 2
    return 0

result = list(map(transform, nums))
```

## Когда использовать lambda?

| Ситуация | Что использовать |
|----------|------------------|
| Простая операция, используется один раз | ✅ lambda |
| Сложная логика | ❌ обычная функция |
| Нужно переиспользовать | ❌ обычная функция |

## Q&A

**Q: Чем lambda отличается от обычной функции?**
A: Lambda — анонимная, в одну строку, только одно выражение, нет явного return.

**Q: Можно ли присвоить lambda переменной?**
A: Да, но PEP 8 рекомендует в таких случаях использовать обычную функцию с def.

**Q: Что такое функции высшего порядка?**
A: Функции, которые принимают другие функции как аргументы (map, filter, sorted, reduce).

## Практика

### Задача 1 — Куб числа
```python
cube = lambda x: x ** 3
cube(3)  # 27
```

### Задача 2 — filter: отрицательные числа
```python
numbers = [5, -3, 8, -1, 0, -7, 2]
negatives = filter(lambda x: x < 0, numbers)
list(negatives)  # [-3, -1, -7]
```

### Задача 3 — map: длины строк
```python
words = ['hello', 'world', 'python', 'lambda']
lengths = map(lambda x: len(x), words)
list(lengths)  # [5, 5, 6, 6]
```

**Заметка:** можно короче — `map(len, words)`, т.к. `len` уже функция.

### Задача 4 — sorted: сортировка по второму элементу
```python
pairs = [(1, 'b'), (3, 'a'), (2, 'c')]
sorted_pairs = sorted(pairs, key=lambda x: x[1])
# [(3, 'a'), (1, 'b'), (2, 'c')]
```

**Заметка:** для вторичной сортировки можно вернуть кортеж: `key=lambda x: (x[1], x[0])`.

### Задача 5 — Комбо: filter + sorted
```python
products = [
    {'name': 'Phone', 'price': 500},
    {'name': 'Cable', 'price': 10},
    {'name': 'Headphones', 'price': 150},
    {'name': 'Case', 'price': 25},
    {'name': 'Charger', 'price': 120}
]

# Товары дороже 100, отсортированные по цене (убывание)
result = sorted(
    filter(lambda x: x['price'] > 100, products),
    key=lambda x: x['price'],
    reverse=True
)
# [{'name': 'Phone', 'price': 500}, {'name': 'Headphones', 'price': 150}, {'name': 'Charger', 'price': 120}]
```

**Альтернатива через list comprehension:**
```python
result = sorted(
    [p for p in products if p['price'] > 100],
    key=lambda x: x['price'],
    reverse=True
)
