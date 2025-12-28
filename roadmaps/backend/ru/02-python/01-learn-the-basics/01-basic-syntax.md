# Базовый синтаксис Python

[prev: 05-bloom-filters](../../01-computer-science/14-advanced-structures/05-bloom-filters.md) | [next: 02-variables-and-data-types](./02-variables-and-data-types.md)
---

## Что такое Python?

**Python** — высокоуровневый интерпретируемый язык программирования. Популярен благодаря:
- Простому и читаемому синтаксису
- Огромной экосистеме библиотек
- Универсальности (веб, данные, AI, автоматизация, скрипты)

## Первая программа

```python
print("Hello, World!")
```

`print()` — встроенная функция для вывода текста на экран.

## Отступы (Indentation)

В Python **отступы — часть синтаксиса**. Они определяют блоки кода (вместо `{}` как в других языках).

```python
# Правильно (4 пробела)
if True:
    print("Внутри блока")
    print("Ещё внутри")
print("Снаружи блока")

# Неправильно — ошибка IndentationError!
if True:
print("Нет отступа")
```

**Стандарт:** 4 пробела на уровень (не табы).

## Комментарии

```python
# Однострочный комментарий

"""
Многострочный комментарий
(технически это строка-литерал, но используется как комментарий)
"""

x = 5  # Комментарий в конце строки
```

## Переменные

```python
name = "Ruslan"      # строка (str)
age = 25             # целое число (int)
height = 1.75        # дробное число (float)
is_student = True    # булево значение (bool)

# Множественное присваивание
x, y, z = 1, 2, 3

# Одно значение нескольким переменным
a = b = c = 0
```

**Правила именования:**
- Начинается с буквы или `_`
- Содержит буквы, цифры, `_`
- Регистр важен: `name` ≠ `Name` ≠ `NAME`
- Нельзя использовать ключевые слова (`if`, `for`, `class` и т.д.)

**Стиль (PEP 8):**
```python
# Хорошо — snake_case
user_name = "Ruslan"
total_count = 100
is_valid = True

# Плохо — camelCase (не Python-стиль)
userName = "Ruslan"
```

## Вывод (print)

```python
# Простой вывод
print("Привет")

# Несколько значений
print("Имя:", name, "Возраст:", age)

# f-строки (форматирование) — рекомендуется
print(f"Мне {age} лет")
print(f"Рост: {height:.2f} м")  # 2 знака после запятой

# Другие способы форматирования
print("Имя: {}".format(name))
print("Имя: %s" % name)  # старый стиль
```

**Параметры print:**
```python
print("a", "b", "c", sep="-")   # a-b-c (разделитель)
print("Hello", end=" ")         # без переноса строки
print("World")                  # Hello World
```

## Ввод (input)

```python
name = input("Как тебя зовут? ")
print(f"Привет, {name}!")

# ВАЖНО: input() всегда возвращает строку!
age = input("Сколько лет? ")    # "25" — это строка
age = int(input("Сколько лет? "))  # 25 — теперь число
```

## Арифметические операции

```python
a = 10
b = 3

a + b    # 13    сложение
a - b    # 7     вычитание
a * b    # 30    умножение
a / b    # 3.33  деление (всегда возвращает float!)
a // b   # 3     целочисленное деление
a % b    # 1     остаток от деления (modulo)
a ** b   # 1000  возведение в степень

# Сокращённые операции
x = 10
x += 5   # x = x + 5, теперь x = 15
x -= 3   # x = x - 3, теперь x = 12
x *= 2   # x = x * 2, теперь x = 24
x /= 4   # x = x / 4, теперь x = 6.0
```

## Операции сравнения

```python
x = 5
y = 10

x == y   # False   равно
x != y   # True    не равно
x < y    # True    меньше
x > y    # False   больше
x <= y   # True    меньше или равно
x >= y   # False   больше или равно
```

Возвращают `True` или `False`.

## Логические операции

```python
a = True
b = False

a and b   # False   И (оба должны быть True)
a or b    # True    ИЛИ (хотя бы один True)
not a     # False   НЕ (инвертирует)

# Примеры
age = 25
is_adult = age >= 18                    # True
can_drive = age >= 18 and has_license   # зависит от has_license
```

## Ключевые слова Python

Зарезервированные слова, нельзя использовать как имена переменных:

```
False, True, None, and, or, not, if, elif, else,
for, while, break, continue, pass, def, return,
class, import, from, as, try, except, finally,
raise, with, lambda, yield, global, nonlocal,
assert, del, in, is
```

## Пример программы

```python
# Простой калькулятор
print("=== Калькулятор ===")

num1 = float(input("Первое число: "))
num2 = float(input("Второе число: "))

print(f"Сумма: {num1 + num2}")
print(f"Разность: {num1 - num2}")
print(f"Произведение: {num1 * num2}")

if num2 != 0:
    print(f"Частное: {num1 / num2}")
else:
    print("Деление на ноль невозможно!")
```

---
[prev: 05-bloom-filters](../../01-computer-science/14-advanced-structures/05-bloom-filters.md) | [next: 02-variables-and-data-types](./02-variables-and-data-types.md)