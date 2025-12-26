# Модули (Modules)

**Модуль** — это файл с Python-кодом (`.py`), который можно импортировать и использовать в других файлах.

## Зачем нужны модули?

- **Организация кода** — разбиваем большой проект на логические части
- **Переиспользование** — один раз написал, используешь везде
- **Пространства имён** — избегаем конфликтов имён
- **Стандартная библиотека** — готовые решения (math, os, json и т.д.)

---

## Создание модуля

Любой `.py` файл — это модуль:

```python
# mymodule.py

PI = 3.14159

def greet(name):
    return f"Привет, {name}!"

def add(a, b):
    return a + b

class Calculator:
    def multiply(self, a, b):
        return a * b
```

---

## Импорт модулей

### import — весь модуль

```python
import mymodule

print(mymodule.PI)           # 3.14159
print(mymodule.greet("Мир")) # Привет, Мир!
print(mymodule.add(2, 3))    # 5

calc = mymodule.Calculator()
print(calc.multiply(4, 5))   # 20
```

### from ... import — конкретные объекты

```python
from mymodule import PI, greet

print(PI)           # 3.14159
print(greet("Мир")) # Привет, Мир!

# add и Calculator недоступны без префикса mymodule
```

### from ... import * — всё (не рекомендуется)

```python
from mymodule import *

print(PI)           # работает
print(greet("Мир")) # работает

# Проблема: засоряет пространство имён, неясно откуда что
```

### import ... as — псевдоним

```python
import mymodule as mm

print(mm.PI)
print(mm.greet("Мир"))

# Часто используется для длинных имён
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
```

### from ... import ... as

```python
from mymodule import greet as say_hello

print(say_hello("Мир"))  # Привет, Мир!
```

---

## Стандартная библиотека

Python поставляется с богатой стандартной библиотекой:

### math — математика

```python
import math

math.sqrt(16)      # 4.0
math.pow(2, 10)    # 1024.0
math.floor(3.7)    # 3
math.ceil(3.2)     # 4
math.pi            # 3.141592653589793
math.e             # 2.718281828459045
math.factorial(5)  # 120
math.gcd(12, 18)   # 6
math.log(100, 10)  # 2.0
```

### random — случайные числа

```python
import random

random.random()              # 0.0 до 1.0
random.randint(1, 10)        # целое от 1 до 10 включительно
random.choice([1, 2, 3])     # случайный элемент
random.choices([1, 2, 3], k=5)  # 5 случайных с повторениями
random.sample([1, 2, 3, 4], k=2) # 2 уникальных
random.shuffle(lst)          # перемешать список (in-place)
random.uniform(1.0, 10.0)    # float от 1.0 до 10.0
```

### datetime — дата и время

```python
from datetime import datetime, date, timedelta

# Текущее время
now = datetime.now()
today = date.today()

# Создание даты
d = date(2025, 12, 26)
dt = datetime(2025, 12, 26, 14, 30, 0)

# Форматирование
now.strftime("%Y-%m-%d %H:%M:%S")  # "2025-12-26 14:30:00"
now.strftime("%d.%m.%Y")           # "26.12.2025"

# Парсинг строки
datetime.strptime("26.12.2025", "%d.%m.%Y")

# Арифметика
tomorrow = today + timedelta(days=1)
week_ago = today - timedelta(weeks=1)
diff = date(2025, 12, 31) - today  # timedelta
```

### os — работа с ОС

```python
import os

os.getcwd()                  # текущая директория
os.listdir(".")              # список файлов
os.path.exists("file.txt")   # существует ли
os.path.isfile("file.txt")   # это файл?
os.path.isdir("folder")      # это папка?
os.path.join("a", "b", "c")  # "a/b/c" (кроссплатформенно)
os.makedirs("a/b/c", exist_ok=True)  # создать папки
os.remove("file.txt")        # удалить файл
os.rename("old.txt", "new.txt")
os.environ["PATH"]           # переменные окружения
os.getenv("HOME", "default") # с дефолтом
```

### pathlib — современная работа с путями

```python
from pathlib import Path

# Создание путей
p = Path("folder/file.txt")
p = Path.home() / "Documents" / "file.txt"

# Свойства
p.name          # "file.txt"
p.stem          # "file"
p.suffix        # ".txt"
p.parent        # Path("folder")
p.parts         # ("folder", "file.txt")

# Проверки
p.exists()
p.is_file()
p.is_dir()

# Операции
p.read_text()               # прочитать как строку
p.write_text("content")     # записать строку
p.mkdir(parents=True, exist_ok=True)  # создать папки

# Поиск файлов
Path(".").glob("*.py")      # все .py в текущей папке
Path(".").rglob("*.py")     # рекурсивно во всех подпапках
```

### json — работа с JSON

```python
import json

# Python → JSON (сериализация)
data = {"name": "Ruslan", "age": 25, "skills": ["Python", "SQL"]}
json_str = json.dumps(data)                    # строка
json_str = json.dumps(data, indent=2)          # красиво
json_str = json.dumps(data, ensure_ascii=False) # кириллица

# JSON → Python (десериализация)
data = json.loads('{"name": "Ruslan"}')

# Работа с файлами
with open("data.json", "w") as f:
    json.dump(data, f, indent=2)

with open("data.json", "r") as f:
    data = json.load(f)
```

### collections — специальные контейнеры

```python
from collections import Counter, defaultdict, deque, namedtuple

# Counter — подсчёт элементов
c = Counter("abracadabra")  # {'a': 5, 'b': 2, 'r': 2, 'c': 1, 'd': 1}
c.most_common(2)            # [('a', 5), ('b', 2)]

# defaultdict — словарь с дефолтным значением
d = defaultdict(list)
d["fruits"].append("apple")  # не нужно проверять ключ
d["fruits"].append("banana")

d = defaultdict(int)
d["count"] += 1  # автоматически 0 + 1

# deque — двусторонняя очередь (см. предыдущие темы)
q = deque([1, 2, 3])
q.appendleft(0)
q.popleft()

# namedtuple — именованный кортеж
Point = namedtuple("Point", ["x", "y"])
p = Point(3, 4)
p.x, p.y  # 3, 4
```

### itertools — итераторы

```python
from itertools import count, cycle, repeat, chain, combinations, permutations

# Бесконечные итераторы
count(10)           # 10, 11, 12, ...
cycle([1, 2, 3])    # 1, 2, 3, 1, 2, 3, ...
repeat("x", 3)      # "x", "x", "x"

# Комбинаторика
list(combinations([1, 2, 3], 2))  # [(1,2), (1,3), (2,3)]
list(permutations([1, 2, 3], 2))  # [(1,2), (1,3), (2,1), (2,3), (3,1), (3,2)]

# Объединение
list(chain([1, 2], [3, 4]))  # [1, 2, 3, 4]
```

### functools — функциональные инструменты

```python
from functools import lru_cache, partial, reduce

# lru_cache — мемоизация (см. рекурсия)
@lru_cache(maxsize=None)
def fib(n):
    if n <= 1:
        return n
    return fib(n - 1) + fib(n - 2)

# partial — частичное применение
def power(base, exp):
    return base ** exp

square = partial(power, exp=2)
square(5)  # 25

# reduce — свёртка
reduce(lambda a, b: a + b, [1, 2, 3, 4])  # 10
reduce(lambda a, b: a * b, [1, 2, 3, 4])  # 24
```

### sys — системные параметры

```python
import sys

sys.argv              # аргументы командной строки
sys.path              # пути поиска модулей
sys.version           # версия Python
sys.exit(0)           # выход из программы
sys.stdin, sys.stdout, sys.stderr  # потоки ввода/вывода
sys.getsizeof(obj)    # размер объекта в байтах
```

### re — регулярные выражения

```python
import re

# Поиск
re.search(r"\d+", "abc123def")     # Match object или None
re.findall(r"\d+", "a1b2c3")       # ["1", "2", "3"]

# Замена
re.sub(r"\d+", "X", "a1b2c3")      # "aXbXcX"

# Разделение
re.split(r"[,;]", "a,b;c")         # ["a", "b", "c"]

# Компиляция (для повторного использования)
pattern = re.compile(r"\d+")
pattern.findall("a1b2c3")
```

---

## Пакеты (Packages)

**Пакет** — это папка с модулями и файлом `__init__.py`:

```
mypackage/
├── __init__.py      # делает папку пакетом
├── module1.py
├── module2.py
└── subpackage/
    ├── __init__.py
    └── module3.py
```

### Импорт из пакета

```python
# Импорт модуля из пакета
import mypackage.module1
from mypackage import module1
from mypackage.module1 import some_function

# Импорт из подпакета
from mypackage.subpackage import module3
from mypackage.subpackage.module3 import something
```

### `__init__.py`

```python
# mypackage/__init__.py

# Пустой файл — просто делает папку пакетом

# Или можно определить что экспортировать
from .module1 import func1, func2
from .module2 import ClassA

__all__ = ["func1", "func2", "ClassA"]  # для from package import *
```

---

## Специальные переменные

### `__name__`

```python
# mymodule.py

def main():
    print("Главная функция")

# Выполнится только при прямом запуске, не при импорте
if __name__ == "__main__":
    main()
```

```bash
python mymodule.py  # __name__ == "__main__", main() выполнится
```

```python
import mymodule     # __name__ == "mymodule", main() НЕ выполнится
```

### `__all__`

Определяет, что экспортируется при `from module import *`:

```python
# mymodule.py

__all__ = ["public_func", "PublicClass"]

def public_func():
    pass

def _private_func():  # конвенция: _ = приватное
    pass

class PublicClass:
    pass
```

### `__file__`

Путь к файлу модуля:

```python
import os
print(__file__)  # /path/to/script.py

# Полезно для поиска ресурсов относительно модуля
base_dir = os.path.dirname(__file__)
config_path = os.path.join(base_dir, "config.json")
```

---

## Как Python ищет модули

Порядок поиска (sys.path):

1. **Текущая директория** (откуда запущен скрипт)
2. **PYTHONPATH** (переменная окружения)
3. **Стандартная библиотека**
4. **site-packages** (установленные пакеты)

```python
import sys
print(sys.path)

# Можно добавить свой путь
sys.path.append("/path/to/my/modules")
```

---

## Относительный vs абсолютный импорт

```
project/
├── main.py
└── mypackage/
    ├── __init__.py
    ├── module_a.py
    └── module_b.py
```

```python
# mypackage/module_a.py

# Абсолютный импорт (рекомендуется)
from mypackage.module_b import something

# Относительный импорт (внутри пакета)
from .module_b import something      # текущий пакет
from ..other_package import thing    # родительский пакет
```

---

## Полезные практики

### 1. Структура проекта

```
myproject/
├── src/
│   └── mypackage/
│       ├── __init__.py
│       ├── core.py
│       └── utils.py
├── tests/
│   └── test_core.py
├── requirements.txt
├── setup.py
└── README.md
```

### 2. Ленивый импорт

```python
# Импорт только когда нужно (экономит время запуска)
def process_data(data):
    import pandas as pd  # импорт внутри функции
    df = pd.DataFrame(data)
    return df
```

### 3. Условный импорт

```python
try:
    import ujson as json  # быстрее
except ImportError:
    import json           # стандартный

# Или для разных версий
import sys
if sys.version_info >= (3, 11):
    from tomllib import load
else:
    from tomli import load
```

### 4. Проверка установленного пакета

```python
try:
    import requests
except ImportError:
    print("Установите: pip install requests")
    sys.exit(1)
```

---

## Установка сторонних модулей

```bash
# pip — менеджер пакетов Python
pip install requests        # установить
pip install requests==2.28  # конкретная версия
pip uninstall requests      # удалить
pip list                    # список установленных
pip freeze > requirements.txt  # сохранить зависимости

# Установка из requirements.txt
pip install -r requirements.txt
```

---

## Популярные сторонние модули

| Модуль | Назначение |
|--------|------------|
| `requests` | HTTP-запросы |
| `numpy` | Числовые вычисления |
| `pandas` | Работа с данными |
| `flask` / `fastapi` | Веб-фреймворки |
| `sqlalchemy` | ORM для баз данных |
| `pytest` | Тестирование |
| `black` / `ruff` | Форматирование кода |
| `pydantic` | Валидация данных |
| `asyncio` | Асинхронность (встроен) |
| `celery` | Фоновые задачи |

---

## Советы

1. **Используй стандартную библиотеку** — там много готовых решений
2. **Не делай `from x import *`** — засоряет пространство имён
3. **Используй псевдонимы** для длинных имён (`import numpy as np`)
4. **Организуй код в пакеты** при росте проекта
5. **Пиши `if __name__ == "__main__"`** — позволяет и импортировать, и запускать
