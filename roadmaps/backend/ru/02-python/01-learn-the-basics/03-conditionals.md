# Условные операторы (Conditionals)

Условия позволяют выполнять разный код в зависимости от ситуации.

## if

```python
age = 18

if age >= 18:
    print("Доступ разрешён")
```

Код внутри `if` выполняется только если условие `True`.

## if-else

```python
age = 16

if age >= 18:
    print("Доступ разрешён")
else:
    print("Доступ запрещён")
```

## if-elif-else

```python
score = 75

if score >= 90:
    grade = "A"
elif score >= 80:
    grade = "B"
elif score >= 70:
    grade = "C"
elif score >= 60:
    grade = "D"
else:
    grade = "F"

print(f"Оценка: {grade}")  # C
```

- `elif` = "else if"
- Проверяется по порядку сверху вниз
- Выполняется только первый блок, где условие True
- `else` выполняется, если ничего не подошло

## Вложенные условия

```python
has_ticket = True
age = 15

if has_ticket:
    if age >= 18:
        print("Добро пожаловать на фильм 18+")
    else:
        print("Фильм только для взрослых")
else:
    print("Купите билет")
```

Лучше избегать глубокой вложенности — сложно читать.

## Составные условия

### and — И (оба должны быть True)
```python
age = 25
has_license = True

if age >= 18 and has_license:
    print("Можете водить")
```

### or — ИЛИ (хотя бы одно True)
```python
if age < 18 or age > 65:
    print("Скидка!")
```

### not — НЕ (инвертирует)
```python
if not has_license:
    print("Нужны права")
```

### Комбинация (используй скобки для ясности)
```python
if (age >= 18 and has_license) or is_emergency:
    print("Можно ехать")
```

## Приоритет операторов

```
not  →  and  →  or
(высший)    (низший)
```

```python
# Это:
a or b and c

# Означает:
a or (b and c)

# Поэтому лучше явно ставить скобки
```

## Тернарный оператор

Короткая запись if-else в одну строку:

```python
age = 20

# Обычный способ
if age >= 18:
    status = "adult"
else:
    status = "minor"

# Тернарный оператор
status = "adult" if age >= 18 else "minor"

# Можно вкладывать (но лучше не злоупотреблять)
category = "child" if age < 13 else "teen" if age < 18 else "adult"
```

## Проверка на вхождение (in)

```python
fruits = ["apple", "banana", "orange"]

if "apple" in fruits:
    print("Яблоко есть!")

if "grape" not in fruits:
    print("Винограда нет")

# Для строк
email = "user@example.com"
if "@" in email:
    print("Похоже на email")

# Для словарей (проверяет ключи)
user = {"name": "Ruslan", "age": 25}
if "name" in user:
    print("Имя указано")
```

## Проверка типа

```python
value = 42

if isinstance(value, int):
    print("Это целое число")
elif isinstance(value, str):
    print("Это строка")
elif isinstance(value, (list, tuple)):
    print("Это список или кортеж")

# Также можно через type(), но isinstance() предпочтительнее
if type(value) == int:
    print("Это int")
```

## Проверка на None

```python
result = None

# Правильно — используй is / is not
if result is None:
    print("Нет результата")

if result is not None:
    print("Результат есть")

# Не делай так (работает, но неидиоматично)
if result == None:  # плохо
    pass
```

## Truthy и Falsy

Python автоматически преобразует значения в bool:

**Falsy (считаются False):**
- `False`
- `None`
- `0`, `0.0`
- `""` (пустая строка)
- `[]`, `()`, `{}`, `set()` (пустые коллекции)

**Truthy (считаются True):**
- Всё остальное

```python
name = ""

# Короткая проверка на пустоту
if name:
    print(f"Привет, {name}")
else:
    print("Имя не указано")

# Эквивалентно:
if len(name) > 0:
    ...

# Для списков
items = []
if items:
    print("Список не пустой")
else:
    print("Список пустой")
```

## Короткое замыкание (Short-circuit)

Python не вычисляет второе условие, если результат уже известен:

```python
# and: если первое False, второе не проверяется
if False and some_function():  # some_function() не вызовется
    pass

# or: если первое True, второе не проверяется
if True or some_function():   # some_function() не вызовется
    pass

# Полезно для безопасных проверок
if user and user.is_active:  # если user=None, второе не проверится
    pass

# Значение по умолчанию
name = user_input or "Guest"  # если user_input пустой/None, будет "Guest"
```

## match-case (Python 3.10+)

Структурное сопоставление с образцом:

```python
command = "start"

match command:
    case "start":
        print("Запуск")
    case "stop":
        print("Остановка")
    case "restart":
        print("Перезапуск")
    case _:
        print("Неизвестная команда")
```

`_` — wildcard (любое значение, как `else`).

### Сложные паттерны

```python
point = (0, 5)

match point:
    case (0, 0):
        print("Начало координат")
    case (0, y):
        print(f"На оси Y, y={y}")
    case (x, 0):
        print(f"На оси X, x={x}")
    case (x, y):
        print(f"Точка ({x}, {y})")

# С условием (guard)
match value:
    case x if x > 0:
        print("Положительное")
    case x if x < 0:
        print("Отрицательное")
    case _:
        print("Ноль")
```

## Примеры

### Проверка возраста
```python
age = int(input("Возраст: "))

if age < 0:
    print("Некорректный возраст")
elif age < 13:
    print("Ребёнок")
elif age < 18:
    print("Подросток")
elif age < 65:
    print("Взрослый")
else:
    print("Пенсионер")
```

### Валидация ввода
```python
password = input("Пароль: ")

if len(password) < 8:
    print("Пароль слишком короткий")
elif password.isdigit():
    print("Пароль не может состоять только из цифр")
elif password.islower():
    print("Добавьте заглавные буквы")
else:
    print("Пароль принят")
```
