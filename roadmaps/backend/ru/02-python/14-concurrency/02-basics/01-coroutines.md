# Знакомство с сопрограммами

[prev: ./readme.md](./readme.md) | [next: ./02-sleep.md](./02-sleep.md)

---

## Что такое корутины (сопрограммы)?

**Корутина (coroutine)** - это специальный тип функции в Python, выполнение которой можно приостановить и возобновить. Корутины являются основой асинхронного программирования в Python и модуля `asyncio`.

В отличие от обычных функций, которые выполняются от начала до конца за один вызов, корутины могут "отдавать управление" в определённых точках, позволяя другому коду выполняться, а затем продолжать своё выполнение с того же места.

## Создание корутины

Корутина создаётся с помощью ключевого слова `async` перед определением функции:

```python
async def my_coroutine():
    print("Привет из корутины!")
    return 42
```

## Ключевые слова async и await

### async

Ключевое слово `async` превращает обычную функцию в корутину:

```python
# Обычная функция
def regular_function():
    return "Я обычная функция"

# Корутина
async def coroutine_function():
    return "Я корутина"
```

### await

Ключевое слово `await` используется внутри корутины для:
1. Вызова других корутин
2. Ожидания асинхронных операций
3. Передачи управления event loop

```python
import asyncio

async def fetch_data():
    print("Начинаю загрузку данных...")
    await asyncio.sleep(2)  # Имитация асинхронной операции
    print("Данные загружены!")
    return {"data": "some_value"}

async def main():
    result = await fetch_data()
    print(f"Результат: {result}")
```

## Запуск корутин

### Способ 1: asyncio.run() (рекомендуемый)

```python
import asyncio

async def greet(name):
    print(f"Привет, {name}!")
    await asyncio.sleep(1)
    print(f"До свидания, {name}!")

# Запуск корутины
asyncio.run(greet("Мир"))
```

### Способ 2: await внутри другой корутины

```python
import asyncio

async def inner():
    return "результат"

async def outer():
    result = await inner()  # Вызов корутины через await
    print(result)

asyncio.run(outer())
```

## Объект корутины

Важно понимать, что вызов корутины НЕ выполняет её код, а создаёт объект корутины:

```python
import asyncio

async def my_coroutine():
    print("Выполняюсь!")
    return 42

# Это НЕ выполняет корутину, а создаёт объект
coro = my_coroutine()
print(type(coro))  # <class 'coroutine'>

# Для выполнения нужен await или asyncio.run()
result = asyncio.run(coro)
print(result)  # 42
```

## Корутины vs Генераторы

Корутины похожи на генераторы, но имеют важные отличия:

```python
# Генератор
def generator():
    yield 1
    yield 2
    yield 3

# Корутина
async def coroutine():
    await asyncio.sleep(0)
    return 1

# Генераторы используют yield
# Корутины используют await
```

| Аспект | Генераторы | Корутины |
|--------|------------|----------|
| Синтаксис | `yield` | `async`/`await` |
| Назначение | Ленивая итерация | Асинхронное выполнение |
| Запуск | `next()`, `for` | `await`, `asyncio.run()` |
| Возврат | `yield` значения | `return` значения |

## Цепочки корутин

Корутины можно вызывать друг из друга, создавая цепочки:

```python
import asyncio

async def get_user_id():
    await asyncio.sleep(0.5)
    return 123

async def get_user_data(user_id):
    await asyncio.sleep(0.5)
    return {"id": user_id, "name": "Иван"}

async def get_user_orders(user_id):
    await asyncio.sleep(0.5)
    return [{"order_id": 1}, {"order_id": 2}]

async def main():
    # Последовательное выполнение корутин
    user_id = await get_user_id()
    user_data = await get_user_data(user_id)
    orders = await get_user_orders(user_id)

    print(f"Пользователь: {user_data}")
    print(f"Заказы: {orders}")

asyncio.run(main())
```

## Возврат значений из корутин

Корутины могут возвращать любые значения:

```python
import asyncio

async def calculate():
    await asyncio.sleep(0.1)
    return 100

async def get_list():
    await asyncio.sleep(0.1)
    return [1, 2, 3, 4, 5]

async def get_dict():
    await asyncio.sleep(0.1)
    return {"key": "value"}

async def main():
    number = await calculate()
    items = await get_list()
    data = await get_dict()

    print(f"Число: {number}")
    print(f"Список: {items}")
    print(f"Словарь: {data}")

asyncio.run(main())
```

## Типизация корутин

Для аннотации типов используется `Coroutine` из модуля `typing`:

```python
from typing import Coroutine
import asyncio

async def fetch_number() -> int:
    await asyncio.sleep(0.1)
    return 42

async def fetch_string() -> str:
    await asyncio.sleep(0.1)
    return "hello"

# Аннотация переменной с объектом корутины
coro: Coroutine[None, None, int] = fetch_number()
```

## Распространённые ошибки

### Ошибка 1: Забыли await

```python
import asyncio

async def get_data():
    return "данные"

async def main():
    # НЕПРАВИЛЬНО - забыли await
    result = get_data()  # Это объект корутины, не результат!
    print(result)  # <coroutine object get_data at 0x...>

    # ПРАВИЛЬНО
    result = await get_data()
    print(result)  # "данные"

asyncio.run(main())
```

### Ошибка 2: await вне async функции

```python
import asyncio

async def get_data():
    return "данные"

# НЕПРАВИЛЬНО - await вне корутины
# result = await get_data()  # SyntaxError!

# ПРАВИЛЬНО - через asyncio.run()
result = asyncio.run(get_data())
```

### Ошибка 3: Неиспользованный объект корутины

```python
import asyncio

async def important_task():
    print("Выполняю важную задачу")

async def main():
    # НЕПРАВИЛЬНО - корутина создана, но не выполнена
    important_task()  # RuntimeWarning: coroutine was never awaited

    # ПРАВИЛЬНО
    await important_task()

asyncio.run(main())
```

## Best Practices

1. **Всегда используйте `await`** при вызове корутин
2. **Используйте `asyncio.run()`** для запуска главной корутины
3. **Не смешивайте** синхронный и асинхронный код без необходимости
4. **Добавляйте аннотации типов** для лучшей читаемости
5. **Давайте понятные имена** корутинам, отражающие их асинхронную природу

```python
import asyncio

# Хорошее именование
async def fetch_user_data(user_id: int) -> dict:
    """Асинхронно получает данные пользователя."""
    await asyncio.sleep(0.1)  # Имитация запроса к БД
    return {"id": user_id, "name": "User"}

async def main() -> None:
    user = await fetch_user_data(1)
    print(user)

if __name__ == "__main__":
    asyncio.run(main())
```

---

[prev: ./readme.md](./readme.md) | [next: ./02-sleep.md](./02-sleep.md)
