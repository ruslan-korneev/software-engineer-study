# Отладка (Debugging)

[prev: 16-logging](./16-logging.md) | [next: 01-what-is-version-control](../03-version-control-systems/01-learn-the-basics/01-what-is-version-control.md)
---

## Методы отладки

1. **Print debugging** — простой, но ограниченный
2. **pdb** — интерактивный отладчик Python
3. **IDE debuggers** — визуальная отладка
4. **Профилирование** — анализ производительности

## Print Debugging

Самый простой способ, но не самый эффективный:

```python
def calculate_total(items):
    print(f"DEBUG: items = {items}")  # Отладочный вывод
    total = 0
    for item in items:
        print(f"DEBUG: processing {item}")
        total += item['price'] * item['quantity']
        print(f"DEBUG: total now = {total}")
    return total
```

### Улучшенный print с f-strings

```python
# Используйте = для вывода выражения и значения
x = 10
y = 20
print(f"{x=}, {y=}, {x+y=}")
# Вывод: x=10, y=20, x+y=30

# Работает со сложными выражениями
items = [1, 2, 3]
print(f"{len(items)=}, {sum(items)=}")
# Вывод: len(items)=3, sum(items)=6
```

### icecream — лучше print

```python
# pip install icecream
from icecream import ic

def foo(x):
    result = x * 2
    ic(result)  # Выводит: ic| result: 20
    return result

foo(10)

# Можно без аргументов — покажет место вызова
def bar():
    ic()  # Выводит: ic| bar.py:10 in bar()
    pass
```

## pdb — Python Debugger

### Запуск отладчика

```python
# Способ 1: breakpoint() (Python 3.7+)
def problematic_function(data):
    breakpoint()  # Отладчик остановится здесь
    result = process(data)
    return result

# Способ 2: import pdb
import pdb

def problematic_function(data):
    pdb.set_trace()  # Старый способ
    result = process(data)
    return result
```

### Запуск скрипта с отладчиком

```bash
# Запустить и остановиться на первой строке
python -m pdb script.py

# Запустить и остановиться при исключении
python -m pdb -c continue script.py
```

### Команды pdb

| Команда | Сокращение | Описание |
|---------|------------|----------|
| `help` | `h` | Показать справку |
| `next` | `n` | Выполнить текущую строку (не входя в функции) |
| `step` | `s` | Выполнить текущую строку (входя в функции) |
| `continue` | `c` | Продолжить выполнение до следующего breakpoint |
| `list` | `l` | Показать код вокруг текущей строки |
| `longlist` | `ll` | Показать весь код функции |
| `print` | `p` | Вывести значение выражения |
| `pp` | — | Pretty-print значения |
| `where` | `w` | Показать стек вызовов |
| `up` | `u` | Перейти на уровень выше в стеке |
| `down` | `d` | Перейти на уровень ниже в стеке |
| `break` | `b` | Установить breakpoint |
| `clear` | `cl` | Удалить breakpoint |
| `return` | `r` | Выполнить до return текущей функции |
| `quit` | `q` | Выйти из отладчика |

### Примеры использования pdb

```python
def buggy_function(items):
    total = 0
    for i, item in enumerate(items):
        breakpoint()  # Остановимся здесь
        # В pdb:
        # (Pdb) p item
        # (Pdb) p total
        # (Pdb) n  # следующая строка
        total += item['price']
    return total
```

### Условные breakpoints

```python
# В коде
for i in range(1000):
    if i == 500:
        breakpoint()

# Или в pdb:
# (Pdb) b 15, i == 500  # Остановиться на строке 15 когда i == 500
```

### post_mortem — отладка после исключения

```python
import pdb

try:
    buggy_code()
except Exception:
    pdb.post_mortem()  # Отладчик в момент исключения
```

## breakpoint() и PYTHONBREAKPOINT

```python
# breakpoint() вызывает функцию из PYTHONBREAKPOINT
# По умолчанию: pdb.set_trace()

# Отключить все breakpoints:
# PYTHONBREAKPOINT=0 python script.py

# Использовать другой отладчик:
# PYTHONBREAKPOINT=ipdb.set_trace python script.py
# PYTHONBREAKPOINT=pudb.set_trace python script.py
```

## ipdb — улучшенный pdb

```python
# pip install ipdb
import ipdb

def function():
    ipdb.set_trace()  # IPython-подобный интерфейс
    # - Автодополнение
    # - Подсветка синтаксиса
    # - Лучшее форматирование
```

## Профилирование

### cProfile — встроенный профилировщик

```bash
# Профилирование скрипта
python -m cProfile script.py

# С сортировкой по времени
python -m cProfile -s cumtime script.py

# Сохранить результаты
python -m cProfile -o output.prof script.py
```

```python
import cProfile
import pstats

# Профилирование функции
def slow_function():
    total = 0
    for i in range(1000000):
        total += i
    return total

cProfile.run('slow_function()')

# Более детально
profiler = cProfile.Profile()
profiler.enable()
slow_function()
profiler.disable()

stats = pstats.Stats(profiler)
stats.sort_stats('cumulative')
stats.print_stats(10)  # Top 10 по времени
```

### line_profiler — построчное профилирование

```python
# pip install line_profiler

# Декоратор @profile добавляется при запуске
@profile
def slow_function():
    result = []
    for i in range(1000):
        result.append(i ** 2)
    return sum(result)

# Запуск: kernprof -l -v script.py
```

### memory_profiler — профилирование памяти

```python
# pip install memory_profiler

from memory_profiler import profile

@profile
def memory_hungry():
    big_list = [i for i in range(1000000)]
    return sum(big_list)

memory_hungry()
```

## timeit — измерение времени

```python
import timeit

# Измерение одного выражения
time = timeit.timeit('sum(range(1000))', number=10000)
print(f"Time: {time:.4f} seconds")

# Сравнение вариантов
setup = "data = list(range(1000))"
time1 = timeit.timeit('sum(data)', setup=setup, number=10000)
time2 = timeit.timeit('sum(x for x in data)', setup=setup, number=10000)
print(f"sum(): {time1:.4f}s, generator: {time2:.4f}s")

# В IPython/Jupyter
# %timeit sum(range(1000))
# %%timeit (для блока кода)
```

### Декоратор для измерения времени

```python
import time
from functools import wraps

def timer(func):
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.perf_counter()
        result = func(*args, **kwargs)
        end = time.perf_counter()
        print(f"{func.__name__} took {end - start:.4f} seconds")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
    return "done"
```

## Отладка в IDE

### VS Code

```json
// .vscode/launch.json
{
    "version": "0.2.0",
    "configurations": [
        {
            "name": "Python: Current File",
            "type": "python",
            "request": "launch",
            "program": "${file}",
            "console": "integratedTerminal",
            "justMyCode": false
        }
    ]
}
```

### PyCharm

- F8: Step Over (n в pdb)
- F7: Step Into (s в pdb)
- Shift+F8: Step Out (r в pdb)
- F9: Resume (c в pdb)
- Ctrl+F8: Toggle Breakpoint

## Отладка асинхронного кода

```python
import asyncio
import pdb

async def async_function():
    await asyncio.sleep(1)
    breakpoint()  # Работает в async функциях
    return "result"

# Для Python < 3.8 нужен aiomonitor или специальные инструменты
```

## Практические советы

### 1. Локализуйте проблему

```python
# Бинарный поиск бага
def process_data(data):
    step1_result = step1(data)
    print(f"After step1: {step1_result}")  # OK?

    step2_result = step2(step1_result)
    print(f"After step2: {step2_result}")  # Проблема здесь?

    return step3(step2_result)
```

### 2. Проверяйте предположения

```python
def function(items):
    assert items is not None, "items should not be None"
    assert len(items) > 0, "items should not be empty"
    assert all(isinstance(i, int) for i in items), "items should be integers"
    # ...
```

### 3. Используйте logging вместо print

```python
import logging

logging.basicConfig(level=logging.DEBUG)
logger = logging.getLogger(__name__)

def function(x):
    logger.debug(f"function called with x={x}")
    # ...
```

### 4. Воспроизводите проблему

```python
# Минимальный воспроизводимый пример
def test_bug():
    """
    Воспроизведение бага:
    1. Вызвать function с пустым списком
    2. Ожидается 0, получаем ошибку
    """
    result = function([])  # Здесь падает
    assert result == 0
```

### 5. Читайте traceback снизу вверх

```
Traceback (most recent call last):
  File "main.py", line 10, in <module>      # 3. Вызвано отсюда
    result = process(data)
  File "utils.py", line 25, in process      # 2. Которая вызвала
    return transform(item)
  File "utils.py", line 15, in transform    # 1. Ошибка здесь
    return item['key']
KeyError: 'key'                              # Тип ошибки
```

## Полезные инструменты

| Инструмент | Назначение |
|------------|-----------|
| `pdb` | Встроенный отладчик |
| `ipdb` | pdb с IPython |
| `pudb` | Консольный GUI отладчик |
| `pdb++` | Улучшенный pdb |
| `icecream` | Улучшенный print |
| `cProfile` | Профилирование CPU |
| `line_profiler` | Построчное профилирование |
| `memory_profiler` | Профилирование памяти |
| `py-spy` | Sampling profiler (без изменения кода) |
| `tracemalloc` | Отслеживание аллокаций памяти |

## Q&A

**Q: Когда использовать print, а когда pdb?**
A: print — для быстрой проверки. pdb — когда нужно исследовать состояние программы интерактивно.

**Q: Как отлаживать код в Docker?**
A: Используйте `docker exec -it container python -m pdb script.py` или настройте remote debugging в IDE.

**Q: breakpoint() не работает?**
A: Проверьте PYTHONBREAKPOINT. Если установлен в 0, breakpoints отключены.

**Q: Как профилировать production код?**
A: Используйте py-spy — он может профилировать запущенный процесс без модификации кода.

---
[prev: 16-logging](./16-logging.md) | [next: 01-what-is-version-control](../03-version-control-systems/01-learn-the-basics/01-what-is-version-control.md)