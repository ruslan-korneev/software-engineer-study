# Pure Functions

## Что такое чистые функции?

Чистая функция (Pure Function) — это функция, которая:

1. **Детерминирована**: всегда возвращает одинаковый результат для одинаковых входных данных
2. **Без побочных эффектов**: не изменяет внешнее состояние и не зависит от него

Концепция пришла из функционального программирования и является одним из ключевых принципов написания чистого, предсказуемого кода.

## Свойства чистых функций

### 1. Детерминированность

```python
# Чистая функция — всегда одинаковый результат
def add(a: int, b: int) -> int:
    return a + b

# add(2, 3) всегда вернёт 5


# Нечистая функция — результат зависит от внешнего состояния
import random

def add_random(a: int) -> int:
    return a + random.randint(1, 10)

# add_random(5) может вернуть 6, 7, 8... — непредсказуемо


# Нечистая функция — зависит от глобальной переменной
tax_rate = 0.2

def calculate_tax(amount: float) -> float:
    return amount * tax_rate  # Зависит от глобального состояния


# Чистая версия — все зависимости переданы явно
def calculate_tax_pure(amount: float, rate: float) -> float:
    return amount * rate
```

### 2. Отсутствие побочных эффектов

Побочные эффекты — это любые изменения за пределами функции:

```python
# Примеры побочных эффектов:

# 1. Изменение глобальных переменных
counter = 0

def increment():  # Нечистая
    global counter
    counter += 1


# 2. Изменение входных аргументов
def add_item(items: list, item):  # Нечистая
    items.append(item)  # Мутирует входной список


# 3. Ввод/вывод
def log_message(message: str):  # Нечистая
    print(message)  # Вывод в консоль


# 4. Сетевые запросы
def fetch_user(user_id: int):  # Нечистая
    return requests.get(f"/users/{user_id}")


# 5. Работа с файлами
def read_config():  # Нечистая
    with open("config.json") as f:
        return json.load(f)


# 6. Изменение БД
def save_user(user):  # Нечистая
    db.users.insert(user)


# 7. Текущее время
def get_timestamp():  # Нечистая
    return datetime.now()  # Каждый вызов — разный результат
```

### 3. Чистые альтернативы

```python
# Вместо изменения глобальной переменной
def increment(counter: int) -> int:
    return counter + 1


# Вместо мутации списка
def add_item(items: list, item) -> list:
    return items + [item]  # Возвращает новый список


# Вместо зависимости от времени
def create_record(data: dict, timestamp: datetime) -> dict:
    return {**data, "created_at": timestamp}
```

## Преимущества чистых функций

### 1. Предсказуемость

```python
# Легко понять, что делает функция
def calculate_discount(price: float, discount_percent: float) -> float:
    return price * (1 - discount_percent / 100)

# Всегда:
# calculate_discount(100, 20) == 80
# calculate_discount(50, 10) == 45
```

### 2. Тестируемость

```python
# Чистые функции легко тестировать
def calculate_total(items: list[dict]) -> float:
    return sum(item['price'] * item['quantity'] for item in items)


# Тест простой и не требует моков
def test_calculate_total():
    items = [
        {'price': 10, 'quantity': 2},
        {'price': 5, 'quantity': 3}
    ]
    assert calculate_total(items) == 35


# Нечистую функцию тестировать сложнее
def calculate_total_with_db():
    items = db.get_cart_items()  # Нужно мокать БД
    return sum(item.price * item.quantity for item in items)
```

### 3. Кэширование (мемоизация)

```python
from functools import lru_cache


# Чистую функцию можно безопасно кэшировать
@lru_cache(maxsize=100)
def fibonacci(n: int) -> int:
    if n < 2:
        return n
    return fibonacci(n - 1) + fibonacci(n - 2)


# Повторные вызовы с теми же аргументами мгновенные
fibonacci(100)  # Вычисляется
fibonacci(100)  # Берётся из кэша


# Нечистую функцию кэшировать опасно
@lru_cache  # ОПАСНО!
def get_current_price(product_id: int) -> float:
    return api.fetch_price(product_id)  # Цена может измениться
```

### 4. Параллелизм

```python
from concurrent.futures import ProcessPoolExecutor


# Чистые функции безопасны для параллельного выполнения
def process_item(item: dict) -> dict:
    return {
        'id': item['id'],
        'result': expensive_calculation(item['data'])
    }


# Можно безопасно распараллелить
with ProcessPoolExecutor() as executor:
    results = list(executor.map(process_item, items))


# Нечистые функции могут вызвать race conditions
shared_results = []

def process_item_impure(item: dict):
    result = expensive_calculation(item['data'])
    shared_results.append(result)  # Race condition!
```

### 5. Композиция

```python
# Чистые функции легко комбинировать
def double(x: int) -> int:
    return x * 2


def increment(x: int) -> int:
    return x + 1


def square(x: int) -> int:
    return x * x


# Композиция функций
from functools import reduce


def compose(*functions):
    """Создаёт композицию функций."""
    return reduce(lambda f, g: lambda x: f(g(x)), functions)


# (x + 1) * 2
transform = compose(double, increment)
result = transform(5)  # (5 + 1) * 2 = 12


# Пайплайн обработки данных
def pipeline(*functions):
    """Применяет функции последовательно."""
    def apply(value):
        for func in functions:
            value = func(value)
        return value
    return apply


process = pipeline(
    lambda x: x.strip(),
    lambda x: x.lower(),
    lambda x: x.replace(' ', '_')
)

result = process("  Hello World  ")  # "hello_world"
```

## Практические паттерны

### 1. Изоляция побочных эффектов

Разделяйте чистую логику и побочные эффекты:

```python
# Плохо — всё смешано
def process_order(order_id: int):
    order = db.get_order(order_id)  # Побочный эффект

    if order.status != 'pending':
        raise ValueError("Order not pending")

    total = sum(item.price * item.quantity for item in order.items)
    discount = calculate_discount(order.customer)
    final_total = total - discount
    tax = final_total * 0.2
    grand_total = final_total + tax

    order.total = grand_total  # Мутация
    order.status = 'processed'  # Мутация

    db.save(order)  # Побочный эффект
    email_service.send_confirmation(order)  # Побочный эффект

    return order


# Хорошо — разделение на чистую логику и побочные эффекты
# Чистая функция — вся бизнес-логика
def calculate_order_totals(items: list, discount: float, tax_rate: float) -> dict:
    """Вычисляет итоги заказа (чистая функция)."""
    subtotal = sum(item['price'] * item['quantity'] for item in items)
    discount_amount = subtotal * discount
    taxable = subtotal - discount_amount
    tax = taxable * tax_rate
    return {
        'subtotal': subtotal,
        'discount': discount_amount,
        'tax': tax,
        'total': taxable + tax
    }


# Оркестратор с побочными эффектами
def process_order(order_id: int):
    """Обрабатывает заказ (имеет побочные эффекты)."""
    # Получение данных
    order = db.get_order(order_id)
    discount_rate = get_customer_discount(order.customer_id)

    # Чистая логика
    totals = calculate_order_totals(
        items=order.items,
        discount=discount_rate,
        tax_rate=0.2
    )

    # Применение изменений
    updated_order = create_processed_order(order, totals)

    # Побочные эффекты
    db.save(updated_order)
    email_service.send_confirmation(updated_order)

    return updated_order
```

### 2. Dependency Injection для побочных эффектов

```python
from typing import Protocol
from abc import abstractmethod


class TimeProvider(Protocol):
    @abstractmethod
    def now(self) -> datetime:
        pass


class RealTimeProvider:
    def now(self) -> datetime:
        return datetime.now()


class FakeTimeProvider:
    def __init__(self, fixed_time: datetime):
        self.fixed_time = fixed_time

    def now(self) -> datetime:
        return self.fixed_time


# Функция становится "чистой" относительно времени
def create_user(
    name: str,
    email: str,
    time_provider: TimeProvider
) -> dict:
    return {
        'name': name,
        'email': email,
        'created_at': time_provider.now()
    }


# В продакшене
user = create_user("John", "john@example.com", RealTimeProvider())

# В тестах
fake_time = FakeTimeProvider(datetime(2024, 1, 1, 12, 0, 0))
user = create_user("John", "john@example.com", fake_time)
assert user['created_at'] == datetime(2024, 1, 1, 12, 0, 0)
```

### 3. Иммутабельные структуры данных

```python
from dataclasses import dataclass
from typing import Tuple


# Мутабельный подход (нечистый)
class MutablePoint:
    def __init__(self, x: int, y: int):
        self.x = x
        self.y = y

    def move(self, dx: int, dy: int):
        self.x += dx  # Мутация
        self.y += dy  # Мутация


# Иммутабельный подход (чистый)
@dataclass(frozen=True)
class Point:
    x: int
    y: int

    def move(self, dx: int, dy: int) -> 'Point':
        return Point(self.x + dx, self.y + dy)  # Новый объект


# Использование
p1 = Point(0, 0)
p2 = p1.move(5, 3)  # p1 не изменился
print(p1)  # Point(x=0, y=0)
print(p2)  # Point(x=5, y=3)


# Иммутабельные коллекции
def add_item(items: Tuple[str, ...], item: str) -> Tuple[str, ...]:
    return items + (item,)


original = ('a', 'b', 'c')
updated = add_item(original, 'd')
print(original)  # ('a', 'b', 'c') — не изменился
print(updated)   # ('a', 'b', 'c', 'd')
```

### 4. Функциональная обработка данных

```python
from typing import List, Callable, TypeVar
from functools import reduce

T = TypeVar('T')
U = TypeVar('U')


# Чистые функции для обработки данных
def filter_items(predicate: Callable[[T], bool], items: List[T]) -> List[T]:
    return [item for item in items if predicate(item)]


def map_items(transform: Callable[[T], U], items: List[T]) -> List[U]:
    return [transform(item) for item in items]


def reduce_items(combine: Callable[[U, T], U], items: List[T], initial: U) -> U:
    return reduce(combine, items, initial)


# Пример использования
orders = [
    {'id': 1, 'total': 100, 'status': 'completed'},
    {'id': 2, 'total': 50, 'status': 'pending'},
    {'id': 3, 'total': 200, 'status': 'completed'},
    {'id': 4, 'total': 75, 'status': 'completed'},
]

# Чистый пайплайн обработки
completed_orders = filter_items(
    lambda o: o['status'] == 'completed',
    orders
)

totals = map_items(
    lambda o: o['total'],
    completed_orders
)

grand_total = reduce_items(
    lambda acc, total: acc + total,
    totals,
    0
)

print(grand_total)  # 375
```

## Типичные ошибки

### 1. Скрытые зависимости от состояния

```python
import datetime

# Плохо — скрытая зависимость от системного времени
def is_weekend() -> bool:
    return datetime.date.today().weekday() >= 5


# Хорошо — явная зависимость
def is_weekend(date: datetime.date) -> bool:
    return date.weekday() >= 5
```

### 2. Мутация аргументов

```python
# Плохо — мутирует входной словарь
def set_defaults(config: dict) -> dict:
    config['timeout'] = config.get('timeout', 30)
    config['retries'] = config.get('retries', 3)
    return config


# Хорошо — возвращает новый словарь
def with_defaults(config: dict) -> dict:
    defaults = {'timeout': 30, 'retries': 3}
    return {**defaults, **config}


# Использование
original = {'timeout': 60}
result = with_defaults(original)
print(original)  # {'timeout': 60} — не изменён
print(result)    # {'timeout': 60, 'retries': 3}
```

### 3. Функции с побочными эффектами в названии "геттеров"

```python
# Плохо — get* подразумевает чистую функцию
def get_user_settings(user_id: int) -> dict:
    settings = db.query(f"SELECT * FROM settings WHERE user_id = {user_id}")
    log(f"Fetched settings for user {user_id}")  # Побочный эффект
    return settings


# Хорошо — явное именование
def fetch_user_settings(user_id: int) -> dict:  # fetch = поход в БД
    return db.query(...)
```

## Best Practices

### 1. Максимизируйте чистый код

```
┌─────────────────────────────────────────┐
│           Application Layer              │
│  (побочные эффекты: HTTP, DB, файлы)    │
└────────────────┬────────────────────────┘
                 │
┌────────────────▼────────────────────────┐
│           Domain Layer                   │
│   (чистая бизнес-логика)                │
│   Максимально большой!                  │
└─────────────────────────────────────────┘
```

### 2. Тестируйте чистые функции первыми

Чистые функции — основа unit-тестов:

```python
# Легко тестировать без моков
@pytest.mark.parametrize("items,expected", [
    ([], 0),
    ([{'price': 10, 'qty': 1}], 10),
    ([{'price': 10, 'qty': 2}, {'price': 5, 'qty': 3}], 35),
])
def test_calculate_total(items, expected):
    assert calculate_total(items) == expected
```

### 3. Используйте типизацию

Типы помогают понять, что функция чистая:

```python
from typing import List, Tuple


def sort_users(users: List[dict]) -> List[dict]:
    """Возвращает отсортированный список (не мутирует оригинал)."""
    return sorted(users, key=lambda u: u['name'])


def split_by_status(
    orders: List[dict]
) -> Tuple[List[dict], List[dict]]:
    """Разделяет заказы на активные и завершённые."""
    active = [o for o in orders if o['status'] == 'active']
    completed = [o for o in orders if o['status'] == 'completed']
    return active, completed
```

## Заключение

Чистые функции — мощный инструмент для написания надёжного, тестируемого и предсказуемого кода. Ключевые принципы:

1. **Явные зависимости** — всё через аргументы
2. **Нет побочных эффектов** — не мутируем, не взаимодействуем с внешним миром
3. **Изоляция** — отделяйте чистую логику от побочных эффектов
4. **Иммутабельность** — предпочитайте неизменяемые данные

Стремитесь к тому, чтобы 80% вашего кода было чистым, а побочные эффекты были изолированы на границах системы.
