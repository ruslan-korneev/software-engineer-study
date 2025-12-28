# Циклы (Loops)

[prev: 03-conditionals](./03-conditionals.md) | [next: 05-type-casting](./05-type-casting.md)
---

Циклы позволяют выполнять код многократно.

## for — перебор элементов

```python
# Перебор списка
fruits = ["apple", "banana", "orange"]

for fruit in fruits:
    print(fruit)

# apple
# banana
# orange
```

```python
# Перебор строки (посимвольно)
for char in "Hello":
    print(char)

# H, e, l, l, o
```

```python
# Перебор словаря
user = {"name": "Ruslan", "age": 25}

# Ключи (по умолчанию)
for key in user:
    print(key)  # name, age

# Значения
for value in user.values():
    print(value)  # Ruslan, 25

# Ключи и значения
for key, value in user.items():
    print(f"{key}: {value}")
```

## range() — диапазон чисел

```python
# range(stop) — от 0 до stop-1
for i in range(5):
    print(i)  # 0, 1, 2, 3, 4

# range(start, stop) — от start до stop-1
for i in range(2, 6):
    print(i)  # 2, 3, 4, 5

# range(start, stop, step) — с шагом
for i in range(0, 10, 2):
    print(i)  # 0, 2, 4, 6, 8

# Обратный отсчёт
for i in range(5, 0, -1):
    print(i)  # 5, 4, 3, 2, 1
```

**Важно:** `range()` не включает последнее значение (stop).

## while — пока условие True

```python
count = 0

while count < 5:
    print(count)
    count += 1

# 0, 1, 2, 3, 4
```

```python
# Ввод до правильного ответа
password = ""
while password != "secret":
    password = input("Пароль: ")

print("Добро пожаловать!")
```

```python
# Бесконечный цикл (осторожно!)
while True:
    command = input("Команда: ")
    if command == "exit":
        break
    print(f"Выполняю: {command}")
```

## break — выход из цикла

Немедленно прерывает цикл:

```python
for i in range(10):
    if i == 5:
        break  # выходим из цикла
    print(i)

# 0, 1, 2, 3, 4
```

```python
# Поиск первого чётного
numbers = [1, 3, 5, 7, 9, 2, 4]

for num in numbers:
    if num % 2 == 0:
        print(f"Первое чётное: {num}")  # 2
        break
```

## continue — пропустить итерацию

Пропускает текущую итерацию и переходит к следующей:

```python
for i in range(5):
    if i == 2:
        continue  # пропускаем 2
    print(i)

# 0, 1, 3, 4
```

```python
# Обработка только положительных
numbers = [1, -2, 3, -4, 5]

for num in numbers:
    if num < 0:
        continue  # пропускаем отрицательные
    print(num)

# 1, 3, 5
```

## pass — пустой оператор

Ничего не делает, заглушка:

```python
for i in range(5):
    if i == 2:
        pass  # TODO: обработать потом
    else:
        print(i)

# Часто используется при написании скелета кода
def future_function():
    pass  # реализуем позже
```

## else в циклах

Выполняется, если цикл завершился **без break**:

```python
for i in range(5):
    if i == 10:  # не найдём
        break
else:
    print("Цикл завершился полностью")  # выполнится
```

```python
# Практический пример: поиск
target = 7
numbers = [1, 3, 5, 9]

for num in numbers:
    if num == target:
        print("Найдено!")
        break
else:
    print("Не найдено")  # выполнится, т.к. break не сработал
```

## enumerate() — индекс + элемент

```python
fruits = ["apple", "banana", "orange"]

for index, fruit in enumerate(fruits):
    print(f"{index}: {fruit}")

# 0: apple
# 1: banana
# 2: orange

# Начать с другого индекса
for index, fruit in enumerate(fruits, start=1):
    print(f"{index}. {fruit}")

# 1. apple
# 2. banana
# 3. orange
```

Без enumerate пришлось бы вручную:
```python
index = 0
for fruit in fruits:
    print(f"{index}: {fruit}")
    index += 1
```

## zip() — параллельный перебор

```python
names = ["Alice", "Bob", "Charlie"]
ages = [25, 30, 35]

for name, age in zip(names, ages):
    print(f"{name}: {age} лет")

# Alice: 25 лет
# Bob: 30 лет
# Charlie: 35 лет
```

```python
# Три списка
names = ["Alice", "Bob"]
ages = [25, 30]
cities = ["Moscow", "SPb"]

for name, age, city in zip(names, ages, cities):
    print(f"{name}, {age}, {city}")
```

**Важно:** zip останавливается на самом коротком списке.

## reversed() — обратный порядок

```python
for i in reversed(range(5)):
    print(i)  # 4, 3, 2, 1, 0

fruits = ["apple", "banana", "orange"]
for fruit in reversed(fruits):
    print(fruit)  # orange, banana, apple
```

## sorted() — отсортированный перебор

```python
numbers = [3, 1, 4, 1, 5, 9]

for num in sorted(numbers):
    print(num)  # 1, 1, 3, 4, 5, 9

for num in sorted(numbers, reverse=True):
    print(num)  # 9, 5, 4, 3, 1, 1
```

## Вложенные циклы

```python
for i in range(3):
    for j in range(3):
        print(f"({i}, {j})", end=" ")
    print()  # новая строка

# (0, 0) (0, 1) (0, 2)
# (1, 0) (1, 1) (1, 2)
# (2, 0) (2, 1) (2, 2)
```

```python
# Таблица умножения
for i in range(1, 6):
    for j in range(1, 6):
        print(f"{i*j:3}", end=" ")
    print()

#   1   2   3   4   5
#   2   4   6   8  10
#   3   6   9  12  15
#   4   8  12  16  20
#   5  10  15  20  25
```

## List Comprehension

Короткий способ создать список:

```python
# Обычный способ
squares = []
for x in range(5):
    squares.append(x ** 2)

# List comprehension
squares = [x ** 2 for x in range(5)]
# [0, 1, 4, 9, 16]
```

```python
# С условием (фильтрация)
evens = [x for x in range(10) if x % 2 == 0]
# [0, 2, 4, 6, 8]

# С преобразованием и условием
words = ["hello", "world", "python"]
upper_long = [w.upper() for w in words if len(w) > 4]
# ['HELLO', 'WORLD', 'PYTHON']
```

```python
# Вложенный (двумерный)
matrix = [[i * j for j in range(3)] for i in range(3)]
# [[0, 0, 0], [0, 1, 2], [0, 2, 4]]
```

## Dict и Set Comprehension

```python
# Dict comprehension
squares = {x: x**2 for x in range(5)}
# {0: 0, 1: 1, 2: 4, 3: 9, 4: 16}

# Set comprehension
unique_lengths = {len(word) for word in ["hi", "hello", "bye"]}
# {2, 3, 5}
```

## Частые ошибки

### Изменение списка во время перебора
```python
# ПЛОХО — непредсказуемое поведение
numbers = [1, 2, 3, 4, 5]
for num in numbers:
    if num % 2 == 0:
        numbers.remove(num)  # Не делай так!

# ХОРОШО — создай новый список
numbers = [num for num in numbers if num % 2 != 0]

# Или перебирай копию
for num in numbers[:]:  # [:] создаёт копию
    if num % 2 == 0:
        numbers.remove(num)
```

### Бесконечный цикл
```python
# Забыли увеличить счётчик
i = 0
while i < 5:
    print(i)
    # i += 1  # забыли!
```

## Примеры

### Сумма чисел
```python
total = 0
for i in range(1, 101):
    total += i
print(f"Сумма 1-100: {total}")  # 5050

# Или проще
print(sum(range(1, 101)))
```

### Факториал
```python
n = 5
factorial = 1
for i in range(1, n + 1):
    factorial *= i
print(f"{n}! = {factorial}")  # 120
```

### Поиск максимума
```python
numbers = [3, 1, 4, 1, 5, 9, 2, 6]
maximum = numbers[0]

for num in numbers:
    if num > maximum:
        maximum = num

print(f"Максимум: {maximum}")  # 9

# Или встроенной функцией
print(max(numbers))
```

---
[prev: 03-conditionals](./03-conditionals.md) | [next: 05-type-casting](./05-type-casting.md)