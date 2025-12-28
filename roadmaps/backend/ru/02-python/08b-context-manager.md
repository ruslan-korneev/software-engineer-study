# Context Manager (Контекстные менеджеры)

[prev: 08a-paradigms](./08a-paradigms.md) | [next: 08c-dataclasses](./08c-dataclasses.md)
---

## Что это?

Объект, управляющий ресурсами через `with`. Гарантирует освобождение даже при ошибках.

```python
# Без with — можно забыть закрыть
f = open("file.txt")
data = f.read()
f.close()  # Легко забыть!

# С with — автоматическое закрытие
with open("file.txt") as f:
    data = f.read()
```

---

## Протокол

| Метод | Когда | Что делает |
|-------|-------|------------|
| `__enter__` | Вход в `with` | Настройка, возвращает объект для `as` |
| `__exit__` | Выход из `with` | Очистка, обработка исключений |

```python
class MyContext:
    def __enter__(self):
        print("Входим")
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Выходим")
        return False  # Не подавлять исключения

with MyContext() as ctx:
    print("Внутри")
# Входим → Внутри → Выходим
```

---

## Параметры `__exit__`

```python
def __exit__(self, exc_type, exc_val, exc_tb):
    # exc_type — тип исключения (или None)
    # exc_val — объект исключения
    # exc_tb — traceback
    # return True — подавить исключение
    # return False — пробросить дальше
```

```python
class SuppressValueError:
    def __enter__(self):
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type is ValueError:
            print(f"Подавили: {exc_val}")
            return True  # Подавить
        return False

with SuppressValueError():
    raise ValueError("Ошибка!")
print("Продолжаем")  # Выполнится
```

---

## Практические примеры

### Таймер

```python
import time

class Timer:
    def __enter__(self):
        self.start = time.time()
        return self

    def __exit__(self, *args):
        print(f"Время: {time.time() - self.start:.3f} сек")
        return False

with Timer():
    sum(range(1000000))
```

### Database connection

```python
class DBConnection:
    def __enter__(self):
        self.conn = connect(...)
        return self.conn

    def __exit__(self, exc_type, exc_val, exc_tb):
        if exc_type:
            self.conn.rollback()
        else:
            self.conn.commit()
        self.conn.close()
        return False
```

---

## `contextlib` — простой способ

### `@contextmanager`

```python
from contextlib import contextmanager

@contextmanager
def timer():
    import time
    start = time.time()
    yield  # Тело with выполняется здесь
    print(f"Время: {time.time() - start:.3f} сек")

with timer():
    sum(range(1000000))
```

### С возвращаемым значением

```python
@contextmanager
def open_file(filename, mode):
    f = open(filename, mode)
    try:
        yield f  # f попадёт в as
    finally:
        f.close()

with open_file("test.txt", "w") as f:
    f.write("Hello!")
```

---

## Встроенные контекстные менеджеры

```python
# Файлы
with open("file.txt") as f: ...

# Блокировки
import threading
with threading.Lock(): ...

# Подавление исключений
from contextlib import suppress
with suppress(FileNotFoundError):
    os.remove("nonexistent.txt")

# Перенаправление stdout
from contextlib import redirect_stdout
import io
f = io.StringIO()
with redirect_stdout(f):
    print("Hello")
```

---

## Вложенные контексты

```python
with open("in.txt") as src, open("out.txt", "w") as dst:
    dst.write(src.read())
```

---

## Резюме

| Способ | Когда |
|--------|-------|
| Класс с `__enter__`/`__exit__` | Сложная логика, состояние |
| `@contextmanager` | Простые случаи |
| `suppress()` | Игнорировать исключения |
| `closing()` | Объекты с `.close()` |

- `with` гарантирует очистку ресурсов
- `__exit__` вызывается даже при исключениях
- `return True` в `__exit__` подавляет исключение

---
[prev: 08a-paradigms](./08a-paradigms.md) | [next: 08c-dataclasses](./08c-dataclasses.md)