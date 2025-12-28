# Природа ошибок в однопоточной конкурентности

[prev: ./readme.md](./readme.md) | [next: ./02-locks.md](./02-locks.md)

---

## Что такое состояние гонки (Race Condition)?

**Состояние гонки** (race condition) - это ситуация, когда результат выполнения программы зависит от порядка или времени выполнения нескольких конкурентных операций. Даже в однопоточном асинхронном коде (asyncio) состояния гонки могут возникать, когда несколько корутин конкурируют за доступ к общим ресурсам.

### Почему это важно в asyncio?

Многие разработчики ошибочно полагают, что раз asyncio работает в одном потоке, то проблем с синхронизацией быть не может. Это заблуждение! Хотя в asyncio нет истинного параллелизма (как в threading), конкурентность всё равно присутствует - корутины могут переключаться в точках `await`, что создаёт возможность для состояний гонки.

## Природа проблемы

### Точки переключения контекста

В asyncio переключение между корутинами происходит в точках `await`. Это означает, что любая операция между двумя `await` выполняется атомарно, но последовательность операций с `await` внутри - нет.

```python
import asyncio

# Общий ресурс
counter = 0

async def increment():
    global counter
    # Эта операция НЕ атомарна в контексте asyncio!
    temp = counter      # Шаг 1: чтение
    await asyncio.sleep(0.001)  # Точка переключения!
    counter = temp + 1  # Шаг 2: запись


async def main():
    global counter
    counter = 0

    # Запускаем 100 корутин "одновременно"
    await asyncio.gather(*[increment() for _ in range(100)])

    print(f"Ожидаемое значение: 100")
    print(f"Фактическое значение: {counter}")


asyncio.run(main())
# Вывод: Фактическое значение: 1 (или другое малое число)
```

### Что произошло?

1. Все 100 корутин прочитали `counter = 0`
2. Все они "заснули" на `await asyncio.sleep(0.001)`
3. Все они записали `0 + 1 = 1`
4. Результат: `counter = 1` вместо ожидаемых 100

## Типичные сценарии состояний гонки

### 1. Check-Then-Act (Проверка-Затем-Действие)

Классический паттерн, порождающий гонки:

```python
import asyncio

cache = {}

async def get_or_create(key: str):
    # ОПАСНО: между проверкой и созданием может произойти переключение
    if key not in cache:
        await asyncio.sleep(0.01)  # Симуляция асинхронной операции
        cache[key] = f"value_for_{key}"
    return cache[key]


async def main():
    # Обе корутины могут пройти проверку "if key not in cache"
    results = await asyncio.gather(
        get_or_create("user_1"),
        get_or_create("user_1")
    )
    print(f"Результаты: {results}")
    print(f"Операций создания: должна быть 1, но могло быть 2")


asyncio.run(main())
```

### 2. Read-Modify-Write (Чтение-Изменение-Запись)

```python
import asyncio

balance = 1000

async def withdraw(amount: int) -> bool:
    global balance

    if balance >= amount:  # Чтение
        await asyncio.sleep(0.01)  # Переключение контекста
        balance -= amount  # Изменение и запись
        return True
    return False


async def main():
    global balance
    balance = 1000

    # Оба снятия могут пройти проверку баланса!
    results = await asyncio.gather(
        withdraw(800),
        withdraw(800)
    )

    print(f"Результаты операций: {results}")
    print(f"Баланс: {balance}")  # Может стать отрицательным!


asyncio.run(main())
# Возможный вывод:
# Результаты операций: [True, True]
# Баланс: -600
```

### 3. Состояние гонки при инициализации

```python
import asyncio

class DatabaseConnection:
    _instance = None
    _initialized = False

    @classmethod
    async def get_instance(cls):
        if cls._instance is None:
            print("Создание нового подключения...")
            await asyncio.sleep(0.1)  # Симуляция подключения
            cls._instance = cls()
            cls._initialized = True
        return cls._instance


async def worker(name: str):
    conn = await DatabaseConnection.get_instance()
    print(f"{name} получил подключение: {id(conn)}")


async def main():
    # Могут создаться несколько экземпляров!
    await asyncio.gather(
        worker("Worker-1"),
        worker("Worker-2"),
        worker("Worker-3")
    )


asyncio.run(main())
# Возможный вывод:
# Создание нового подключения...
# Создание нового подключения...
# Создание нового подключения...
# Worker-1 получил подключение: 140234567890
# Worker-2 получил подключение: 140234567891  # Разные объекты!
# Worker-3 получил подключение: 140234567892
```

## Как обнаружить состояния гонки

### 1. Анализ кода

Ищите паттерны:
- Проверка условия, затем `await`, затем действие на основе проверки
- Чтение переменной, `await`, запись в переменную
- Любые операции с общими ресурсами, разделённые `await`

### 2. Тестирование с asyncio.sleep(0)

```python
import asyncio

async def suspicious_operation():
    # Добавляем sleep(0) для принудительного переключения
    # Это помогает выявить скрытые race conditions
    await asyncio.sleep(0)
```

### 3. Стресс-тестирование

```python
import asyncio

async def stress_test():
    errors = 0
    for i in range(1000):
        global counter
        counter = 0
        await asyncio.gather(*[increment() for _ in range(10)])
        if counter != 10:
            errors += 1
    print(f"Ошибок из 1000 запусков: {errors}")
```

## Когда синхронизация НЕ нужна

Не все операции требуют синхронизации:

```python
import asyncio

# Это БЕЗОПАСНО - между операциями нет await
counter = 0

async def safe_increment():
    global counter
    counter += 1  # Атомарно в контексте asyncio!


async def main():
    global counter
    counter = 0
    await asyncio.gather(*[safe_increment() for _ in range(1000)])
    print(f"Счётчик: {counter}")  # Всегда 1000


asyncio.run(main())
```

## Последствия состояний гонки

1. **Потеря данных** - записи перезаписывают друг друга
2. **Нарушение инвариантов** - например, отрицательный баланс
3. **Дублирование операций** - двойные платежи, двойная отправка
4. **Непредсказуемое поведение** - ошибки, которые сложно воспроизвести
5. **Утечки ресурсов** - например, незакрытые соединения

## Методы предотвращения

### 1. Минимизация общего состояния

```python
# Плохо: общее изменяемое состояние
results = []

async def bad_worker(data):
    result = await process(data)
    results.append(result)  # Небезопасно!

# Хорошо: возврат результата
async def good_worker(data):
    return await process(data)

async def main():
    results = await asyncio.gather(*[good_worker(d) for d in data_list])
```

### 2. Использование примитивов синхронизации

- `asyncio.Lock` - для взаимоисключающего доступа
- `asyncio.Semaphore` - для ограничения числа конкурентных операций
- `asyncio.Event` - для сигнализации между корутинами
- `asyncio.Condition` - для сложной координации

### 3. Атомарные операции

```python
import asyncio

# Используем asyncio.Queue вместо списка
queue = asyncio.Queue()

async def producer():
    await queue.put("item")  # Атомарная операция

async def consumer():
    item = await queue.get()  # Атомарная операция
```

## Практические рекомендации

1. **Идентифицируйте общие ресурсы** перед написанием кода
2. **Минимизируйте критические секции** - код между lock.acquire() и lock.release()
3. **Используйте контекстные менеджеры** для блокировок
4. **Тестируйте конкурентность** с множеством одновременных операций
5. **Документируйте** требования к синхронизации

## Резюме

- Состояния гонки возможны даже в однопоточном asyncio
- Точки `await` - это места потенциального переключения контекста
- Паттерны check-then-act и read-modify-write особенно уязвимы
- Синхронизация необходима при работе с общим изменяемым состоянием
- asyncio предоставляет примитивы: Lock, Semaphore, Event, Condition

---

[prev: ./readme.md](./readme.md) | [next: ./02-locks.md](./02-locks.md)
