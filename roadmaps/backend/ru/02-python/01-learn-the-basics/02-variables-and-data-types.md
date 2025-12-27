# Переменные и типы данных

## Что такое переменная?

**Переменная** — это имя, которое ссылается на значение в памяти.

```python
age = 25
```

```
age  ──────►  [ 25 ]
 имя           значение в памяти
```

Python — язык с **динамической типизацией**: тип переменной определяется автоматически и может меняться.

```python
x = 10        # x — это int
x = "hello"   # теперь x — это str
```

## Основные типы данных

| Тип | Название | Пример | Изменяемый |
|-----|----------|--------|------------|
| `int` | Целое число | `42`, `-10` | Нет |
| `float` | Дробное число | `3.14`, `-0.5` | Нет |
| `str` | Строка | `"Hello"` | Нет |
| `bool` | Булево значение | `True`, `False` | Нет |
| `None` | Отсутствие значения | `None` | - |
| `list` | Список | `[1, 2, 3]` | Да |
| `tuple` | Кортеж | `(1, 2, 3)` | Нет |
| `dict` | Словарь | `{"a": 1}` | Да |
| `set` | Множество | `{1, 2, 3}` | Да |

## int (целые числа)

```python
a = 42
b = -10
c = 0

# Разделение разрядов для читаемости
million = 1_000_000   # 1000000

# Разные системы счисления
binary = 0b1010      # 10 в двоичной
octal = 0o17         # 15 в восьмеричной
hexadecimal = 0xFF   # 255 в шестнадцатеричной

# Размер не ограничен!
big = 10 ** 100      # работает без проблем
```

## float (дробные числа)

```python
pi = 3.14159
temperature = -40.5
price = 99.99

# Научная нотация
avogadro = 6.022e23    # 6.022 × 10²³
small = 1e-10          # 0.0000000001

# Специальные значения
infinity = float('inf')
neg_infinity = float('-inf')
not_a_number = float('nan')
```

**Осторожно с точностью!**
```python
print(0.1 + 0.2)   # 0.30000000000000004 (не 0.3!)

# Для точных вычислений используй модуль decimal
from decimal import Decimal
print(Decimal('0.1') + Decimal('0.2'))  # 0.3
```

## str (строки)

```python
# Одинарные или двойные кавычки — без разницы
name = 'Ruslan'
greeting = "Hello"

# Многострочные строки
text = """Это
многострочный
текст"""

# Экранирование специальных символов
path = "C:\\Users\\name"      # \\ → \
quote = "Он сказал: \"Привет\""
newline = "Строка 1\nСтрока 2"  # \n — перенос строки
tab = "Колонка1\tКолонка2"      # \t — табуляция

# Raw-строки (без экранирования) — r перед кавычками
path = r"C:\Users\name"
```

### Операции со строками

```python
s = "Hello, World"

# Длина
len(s)              # 12

# Регистр
s.upper()           # "HELLO, WORLD"
s.lower()           # "hello, world"
s.title()           # "Hello, World"
s.capitalize()      # "Hello, world"

# Поиск и замена
s.find("World")     # 7 (индекс начала)
s.find("xyz")       # -1 (не найдено)
s.replace("World", "Python")  # "Hello, Python"
s.count("l")        # 3

# Проверки
s.startswith("Hello")  # True
s.endswith("!")        # False
"World" in s           # True
s.isdigit()            # False
s.isalpha()            # False (есть запятая и пробел)

# Разделение и соединение
"a,b,c".split(",")     # ["a", "b", "c"]
"-".join(["a", "b"])   # "a-b"

# Удаление пробелов
"  text  ".strip()     # "text"
"  text  ".lstrip()    # "text  "
"  text  ".rstrip()    # "  text"
```

### Индексация и срезы

```python
s = "Python"

# Индексация (с 0!)
s[0]      # "P"
s[1]      # "y"
s[-1]     # "n" (с конца)
s[-2]     # "o"

# Срезы [start:end:step]
s[0:3]    # "Pyt" (с 0 до 3, не включая 3)
s[:3]     # "Pyt" (от начала)
s[3:]     # "hon" (до конца)
s[::2]    # "Pto" (каждый второй)
s[::-1]   # "nohtyP" (реверс)
```

### Форматирование строк

```python
name = "Ruslan"
age = 25

# f-строки (рекомендуется, Python 3.6+)
f"Имя: {name}, возраст: {age}"

# Форматирование чисел
pi = 3.14159
f"{pi:.2f}"         # "3.14" (2 знака после запятой)
f"{1000000:,}"      # "1,000,000" (разделитель тысяч)
f"{42:05d}"         # "00042" (с ведущими нулями)

# Выравнивание
f"{'text':>10}"     # "      text" (вправо, ширина 10)
f"{'text':<10}"     # "text      " (влево)
f"{'text':^10}"     # "   text   " (по центру)

# Старые способы
"Имя: {}, возраст: {}".format(name, age)
"Имя: %s, возраст: %d" % (name, age)
```

## bool (логический тип)

```python
is_active = True
is_deleted = False

# Результат сравнений — bool
5 > 3      # True
5 == 3     # False

# Логические операции
True and False   # False
True or False    # True
not True         # False
```

### Что считается False (falsy values)

```python
bool(0)         # False
bool(0.0)       # False
bool("")        # False — пустая строка
bool([])        # False — пустой список
bool({})        # False — пустой словарь
bool(())        # False — пустой кортеж
bool(set())     # False — пустое множество
bool(None)      # False

# Всё остальное — True
bool(1)         # True
bool(-1)        # True
bool("text")    # True
bool([0])       # True — непустой список
```

## None

```python
result = None    # переменная существует, но значения нет

# Проверка на None (используй is, не ==)
if result is None:
    print("Значения нет")

if result is not None:
    print("Значение есть")

# Частое использование — значение по умолчанию
def greet(name=None):
    if name is None:
        name = "Guest"
    print(f"Hello, {name}")
```

## Проверка типа

```python
x = 42

# Узнать тип
type(x)              # <class 'int'>
type(x).__name__     # 'int'

# Проверить тип
isinstance(x, int)   # True
isinstance(x, str)   # False

# Проверить несколько типов
isinstance(x, (int, float))  # True

# Проверить, что это число
isinstance(x, (int, float, complex))  # True
```

## Изменяемые vs неизменяемые типы

**Неизменяемые (immutable):** int, float, str, bool, tuple
```python
x = 5
x = x + 1   # создаётся новый объект, старый остаётся
```

**Изменяемые (mutable):** list, dict, set
```python
lst = [1, 2, 3]
lst.append(4)   # изменяется тот же объект
```
