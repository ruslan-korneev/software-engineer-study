# Исключения (Exceptions)

[prev: 05-type-casting](./05-type-casting.md) | [next: 07-functions](./07-functions.md)
---

Исключения — это ошибки, возникающие во время выполнения программы.

## Что происходит без обработки?

```python
print(10 / 0)
```

```
Traceback (most recent call last):
  File "main.py", line 1, in <module>
    print(10 / 0)
ZeroDivisionError: division by zero
```

Программа падает. Но мы можем это предотвратить.

## try-except — перехват ошибки

```python
try:
    result = 10 / 0
except ZeroDivisionError:
    print("Деление на ноль!")

print("Программа продолжает работать")
```

## Несколько типов исключений

```python
try:
    number = int(input("Число: "))
    result = 10 / number
except ValueError:
    print("Это не число!")
except ZeroDivisionError:
    print("На ноль делить нельзя!")
```

### Объединение в один блок
```python
try:
    ...
except (ValueError, ZeroDivisionError):
    print("Ошибка ввода")
```

### Разные действия с общим fallback
```python
try:
    ...
except ValueError:
    print("Неверное значение")
except Exception:
    print("Другая ошибка")
```

## except без типа

```python
# Перехватывает ВСЕ исключения (не рекомендуется)
try:
    risky_operation()
except:
    print("Что-то пошло не так")

# Лучше так — перехват с информацией:
try:
    risky_operation()
except Exception as e:
    print(f"Ошибка: {e}")
```

**Не рекомендуется** перехватывать всё без разбора — можешь скрыть баги.

## Получение информации об ошибке

```python
try:
    int("hello")
except ValueError as e:
    print(f"Тип: {type(e).__name__}")  # ValueError
    print(f"Сообщение: {e}")           # invalid literal for int()...

    # Для отладки — полный traceback
    import traceback
    traceback.print_exc()
```

## else — если ошибок не было

```python
try:
    number = int(input("Число: "))
except ValueError:
    print("Некорректное число")
else:
    # Выполняется только если НЕ было исключения
    print(f"Вы ввели: {number}")
    result = 100 / number
```

Используй `else`, когда код должен выполниться только при успехе, но сам не должен быть в try.

## finally — выполнится всегда

```python
try:
    file = open("data.txt")
    data = file.read()
except FileNotFoundError:
    print("Файл не найден")
finally:
    # Выполнится в ЛЮБОМ случае — и при ошибке, и при успехе
    print("Завершение операции")
```

### Типичное использование — освобождение ресурсов
```python
file = None
try:
    file = open("data.txt")
    data = file.read()
except FileNotFoundError:
    print("Файл не найден")
finally:
    if file:
        file.close()  # Закроется в любом случае

# Но лучше использовать with (context manager):
with open("data.txt") as file:
    data = file.read()
# Файл закроется автоматически
```

## Полная структура try

```python
try:
    # Опасный код
    result = risky_operation()
except SpecificError:
    # Обработка конкретной ошибки
    pass
except Exception as e:
    # Обработка остальных ошибок
    pass
else:
    # Если ошибок НЕ было
    pass
finally:
    # Выполнится ВСЕГДА
    pass
```

## raise — вызов исключения

### Генерация исключения
```python
def divide(a, b):
    if b == 0:
        raise ValueError("Делитель не может быть нулём")
    return a / b

try:
    divide(10, 0)
except ValueError as e:
    print(e)  # Делитель не может быть нулём
```

### Перевыброс исключения
```python
try:
    risky()
except Exception:
    print("Логируем ошибку")
    raise  # Пробрасываем исключение дальше
```

### Цепочка исключений
```python
try:
    int("abc")
except ValueError as original:
    raise RuntimeError("Ошибка обработки данных") from original
```

## Частые типы исключений

| Исключение | Когда возникает | Пример |
|------------|-----------------|--------|
| `ValueError` | Неверное значение | `int("abc")` |
| `TypeError` | Неверный тип | `"2" + 2` |
| `KeyError` | Ключ не найден в dict | `d["missing"]` |
| `IndexError` | Индекс за пределами | `lst[100]` |
| `AttributeError` | Атрибут не существует | `"str".foo()` |
| `NameError` | Переменная не определена | `print(undefined)` |
| `FileNotFoundError` | Файл не существует | `open("no.txt")` |
| `ZeroDivisionError` | Деление на ноль | `1 / 0` |
| `ImportError` | Ошибка импорта | `import nomodule` |
| `StopIteration` | Итератор исчерпан | `next(empty_iter)` |
| `RuntimeError` | Общая ошибка выполнения | — |
| `MemoryError` | Недостаточно памяти | — |
| `RecursionError` | Превышена глубина рекурсии | — |

## Иерархия исключений

```
BaseException
├── SystemExit
├── KeyboardInterrupt
├── GeneratorExit
└── Exception
    ├── ValueError
    ├── TypeError
    ├── KeyError
    ├── IndexError
    ├── FileNotFoundError (OSError)
    ├── ZeroDivisionError (ArithmeticError)
    └── ...
```

`except Exception` — перехватывает всё кроме системных (SystemExit, KeyboardInterrupt).

## Создание своих исключений

```python
# Простое исключение
class ValidationError(Exception):
    pass

# С дополнительными данными
class AgeError(Exception):
    def __init__(self, age, message="Некорректный возраст"):
        self.age = age
        self.message = message
        super().__init__(self.message)

    def __str__(self):
        return f"{self.message}: {self.age}"

# Иерархия исключений
class AppError(Exception):
    """Базовое исключение приложения"""
    pass

class DatabaseError(AppError):
    pass

class ValidationError(AppError):
    pass
```

### Использование
```python
def set_age(age):
    if age < 0:
        raise AgeError(age, "Возраст не может быть отрицательным")
    if age > 150:
        raise AgeError(age, "Слишком большой возраст")
    return age

try:
    set_age(-5)
except AgeError as e:
    print(e)  # Возраст не может быть отрицательным: -5
```

## LBYL vs EAFP

### LBYL (Look Before You Leap) — проверка заранее
```python
# Проверяем перед действием
if key in dictionary:
    value = dictionary[key]
else:
    value = default
```

### EAFP (Easier to Ask Forgiveness than Permission) — Pythonic way
```python
# Пробуем, ловим ошибку
try:
    value = dictionary[key]
except KeyError:
    value = default

# Или проще:
value = dictionary.get(key, default)
```

В Python предпочитается **EAFP** — это часто быстрее и читаемее.

## Практические примеры

### Безопасный ввод числа
```python
def get_number(prompt, type_=float):
    while True:
        try:
            return type_(input(prompt))
        except ValueError:
            print("Некорректное значение. Попробуйте снова.")

age = get_number("Возраст: ", int)
```

### Безопасное чтение файла
```python
def read_file(path, default=""):
    try:
        with open(path) as f:
            return f.read()
    except FileNotFoundError:
        return default
    except PermissionError:
        print(f"Нет доступа к файлу: {path}")
        return default

content = read_file("config.txt", "{}")
```

### Валидация данных
```python
def validate_user(data):
    errors = []

    if not data.get("email"):
        errors.append("Email обязателен")
    elif "@" not in data["email"]:
        errors.append("Некорректный email")

    if not data.get("age"):
        errors.append("Возраст обязателен")
    elif data["age"] < 0:
        errors.append("Возраст не может быть отрицательным")

    if errors:
        raise ValidationError(errors)

    return True
```

## assert — проверка условий (для отладки)

```python
def divide(a, b):
    assert b != 0, "Делитель не может быть нулём"
    return a / b

# assert отключается при запуске с -O (оптимизация)
# python -O script.py
```

**Используй assert для:**
- Проверок во время разработки
- Инвариантов, которые "никогда не должны нарушаться"

**НЕ используй для:**
- Валидации пользовательского ввода
- Проверок в продакшене

---

## Q&A

### Чем отличается Exception от BaseException?

**Иерархия:**
```
BaseException           ← корень всех исключений
├── SystemExit          ← sys.exit()
├── KeyboardInterrupt   ← Ctrl+C
├── GeneratorExit       ← закрытие генератора
└── Exception           ← все "обычные" ошибки
    ├── ValueError
    ├── TypeError
    └── ...
```

**Ключевое отличие:**
```python
# except Exception НЕ ловит системные исключения
try:
    while True:
        pass
except Exception:
    print("Поймано!")  # Ctrl+C НЕ поймается — программа завершится

# except BaseException ловит ВСЁ
try:
    while True:
        pass
except BaseException:
    print("Поймано!")  # Ctrl+C поймается — опасно!
```

**Почему важно:** SystemExit, KeyboardInterrupt — не ошибки, а сигналы. Если их ловить, пользователь не сможет остановить программу через Ctrl+C.

**Правило:**
| Наследуй от | Когда |
|-------------|-------|
| `Exception` | Всегда (99.9% случаев) |
| `BaseException` | Практически никогда |

```python
# Правильно
class MyError(Exception):
    pass

# Неправильно
class MyError(BaseException):
    pass
```

**Исключение** — логирование с перевыбросом:
```python
try:
    run_app()
except BaseException as e:
    log_error(e)
    raise  # обязательно перевыбрасываем!
```

---
[prev: 05-type-casting](./05-type-casting.md) | [next: 07-functions](./07-functions.md)