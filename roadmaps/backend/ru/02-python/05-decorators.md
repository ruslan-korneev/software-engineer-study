# Декораторы (Decorators)

## Что такое декоратор?

**Декоратор** — функция, которая принимает другую функцию и расширяет её поведение, не изменяя её код.

Декоратор "оборачивает" функцию, добавляя логику до или после её выполнения.

## Предпосылки: функции — это объекты

```python
def greet():
    return "Привет!"

# Функция — объект, можно присвоить переменной
say_hello = greet
say_hello()  # Привет!

# Можно передать как аргумент
def call_twice(func):
    func()
    func()
```

## Простейший декоратор

```python
def my_decorator(func):
    def wrapper():
        print("До вызова")
        func()
        print("После вызова")
    return wrapper

@my_decorator
def say_hello():
    print("Hello!")

say_hello()
# До вызова
# Hello!
# После вызова
```

Синтаксис `@my_decorator` — это сокращение для `say_hello = my_decorator(say_hello)`.

## Декоратор с аргументами функции

Используем `*args, **kwargs` чтобы принять любые аргументы:

```python
def my_decorator(func):
    def wrapper(*args, **kwargs):
        print(f"Вызов {func.__name__}")
        result = func(*args, **kwargs)
        return result
    return wrapper

@my_decorator
def add(a, b):
    return a + b

add(2, 3)  # Вызов add → 5
```

## Практические примеры

### Измерение времени выполнения

```python
import time

def timer(func):
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        end = time.time()
        print(f"{func.__name__} выполнилась за {end - start:.4f} сек")
        return result
    return wrapper

@timer
def slow_function():
    time.sleep(1)
```

### Логирование

```python
def log(func):
    def wrapper(*args, **kwargs):
        print(f"Вызов: {func.__name__}({args}, {kwargs})")
        return func(*args, **kwargs)
    return wrapper

@log
def multiply(a, b):
    return a * b
```

### Проверка авторизации

```python
def require_auth(func):
    def wrapper(user, *args, **kwargs):
        if not user.get('is_authenticated'):
            raise PermissionError("Требуется авторизация!")
        return func(user, *args, **kwargs)
    return wrapper

@require_auth
def delete_post(user, post_id):
    print(f"Пост {post_id} удалён")
```

## functools.wraps — сохраняем метаданные

Без `@wraps` теряется имя и docstring оригинальной функции:

```python
from functools import wraps

def my_decorator(func):
    @wraps(func)  # Сохраняет __name__, __doc__
    def wrapper(*args, **kwargs):
        return func(*args, **kwargs)
    return wrapper
```

**Правило:** всегда используй `@wraps(func)` в декораторах.

## Декоратор с параметрами

Добавляем ещё один уровень вложенности:

```python
def repeat(times):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            for _ in range(times):
                result = func(*args, **kwargs)
            return result
        return wrapper
    return decorator

@repeat(times=3)
def say_hi():
    print("Hi!")

say_hi()
# Hi!
# Hi!
# Hi!
```

## Несколько декораторов

Применяются снизу вверх:

```python
@decorator1
@decorator2
def func():
    pass

# Эквивалентно:
# func = decorator1(decorator2(func))
```

## Встроенные декораторы Python

| Декоратор | Назначение |
|-----------|------------|
| `@staticmethod` | Метод без доступа к self/cls |
| `@classmethod` | Метод с доступом к классу (cls) |
| `@property` | Геттер для атрибута |
| `@functools.lru_cache` | Кеширование результатов |
| `@dataclasses.dataclass` | Автогенерация __init__, __repr__ и др. |

## Q&A

**Q: Зачем нужны декораторы?**
A: Чтобы добавить общую логику (логирование, проверки, кеширование) без изменения кода функции. DRY-принцип.

**Q: Что такое `*args, **kwargs` в wrapper?**
A: Позволяет принять любые аргументы и передать их в оригинальную функцию.

**Q: Зачем `@wraps(func)`?**
A: Сохраняет `__name__`, `__doc__` и другие атрибуты оригинальной функции.

**Q: Как работает декоратор с параметрами?**
A: Это функция, которая возвращает декоратор. Три уровня: параметры → декоратор → wrapper.
