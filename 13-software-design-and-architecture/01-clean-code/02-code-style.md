# Code Style

## Что такое стиль кода?

Стиль кода (Code Style) — это набор соглашений о форматировании, структуре и организации исходного кода. Единый стиль кода в проекте обеспечивает:

- **Читаемость**: код легче понимать
- **Согласованность**: весь код выглядит одинаково
- **Уменьшение когнитивной нагрузки**: не нужно привыкать к разным стилям
- **Упрощение code review**: фокус на логике, а не на форматировании
- **Лёгкую интеграцию**: новые разработчики быстрее включаются

## Основные элементы стиля кода

### 1. Отступы и выравнивание

Отступы определяют структуру кода и вложенность блоков.

```python
# Пробелы (рекомендуется для Python — 4 пробела)
def calculate_total(items):
    total = 0
    for item in items:
        if item.is_active:
            total += item.price
    return total

# Табуляция (используется в Go, Makefile)
# Важно: не смешивайте табы и пробелы!
```

### 2. Длина строки

Ограничение длины строки улучшает читаемость:

```python
# PEP 8 рекомендует 79 символов для кода, 72 для комментариев
# Многие проекты используют 88 (black) или 120 символов

# Плохо — слишком длинная строка
result = some_function(very_long_argument_name, another_long_argument, yet_another_argument, and_one_more)

# Хорошо — разбито на несколько строк
result = some_function(
    very_long_argument_name,
    another_long_argument,
    yet_another_argument,
    and_one_more
)

# Или с использованием переменных
args = {
    'first': very_long_argument_name,
    'second': another_long_argument,
}
result = some_function(**args)
```

### 3. Пустые строки

Пустые строки разделяют логические блоки:

```python
# Две пустые строки между функциями верхнего уровня и классами
class User:
    pass


class Order:
    pass


def process_order(order):
    pass


# Одна пустая строка между методами класса
class UserService:

    def create_user(self, data):
        pass

    def update_user(self, user_id, data):
        pass

    def delete_user(self, user_id):
        pass


# Пустые строки внутри функций для логического разделения
def process_payment(order):
    # Валидация
    if not order.is_valid():
        raise ValidationError("Invalid order")

    # Расчёт суммы
    total = order.calculate_total()
    tax = calculate_tax(total)
    final_amount = total + tax

    # Обработка платежа
    payment_result = payment_gateway.process(final_amount)

    # Обновление статуса
    order.status = 'paid'
    order.save()

    return payment_result
```

### 4. Пробелы вокруг операторов

```python
# Хорошо — пробелы вокруг операторов присваивания и сравнения
x = 5
y = x + 10
if x == y:
    pass

# Плохо — нет пробелов
x=5
y=x+10
if x==y:
    pass

# Исключение — именованные аргументы и значения по умолчанию
def function(arg1, arg2=None, arg3=True):
    pass

function(arg1='value', arg2=10)

# Пробелы после запятых
items = [1, 2, 3, 4, 5]
result = function(a, b, c)

# Нет пробелов внутри скобок
# Плохо
function( arg1, arg2 )
items[ 0 ]

# Хорошо
function(arg1, arg2)
items[0]
```

### 5. Кавычки для строк

```python
# Python: одинарные или двойные — главное, консистентно
name = 'John'  # или "John"
message = "It's a test"  # двойные, если есть апостроф
sql = 'SELECT * FROM "users"'  # одинарные, если есть двойные внутри

# Docstrings — всегда двойные кавычки
def function():
    """Это документация функции."""
    pass

# f-строки для форматирования
greeting = f"Hello, {name}!"

# Многострочные строки
long_text = """
This is a long text
that spans multiple lines.
"""
```

### 6. Импорты

```python
# Порядок импортов (PEP 8):
# 1. Стандартная библиотека
# 2. Сторонние пакеты
# 3. Локальные модули

# Группы разделены пустой строкой
import os
import sys
from datetime import datetime

import requests
from flask import Flask, request

from myapp.models import User
from myapp.utils import helper


# Каждый импорт на отдельной строке
# Плохо
import os, sys, json

# Хорошо
import os
import sys
import json

# Исключение — from import
from typing import List, Dict, Optional
```

## Стиль для разных языков

### Python (PEP 8)

```python
# Именование
variable_name = "snake_case"
CONSTANT_NAME = "SCREAMING_SNAKE_CASE"
ClassName = "PascalCase"
_private_variable = "с подчёркиванием"

# Максимальная длина строки: 79-88 символов
# Отступы: 4 пробела

# Docstrings
def calculate_area(width: float, height: float) -> float:
    """
    Вычисляет площадь прямоугольника.

    Args:
        width: Ширина прямоугольника.
        height: Высота прямоугольника.

    Returns:
        Площадь прямоугольника.

    Raises:
        ValueError: Если ширина или высота отрицательны.
    """
    if width < 0 or height < 0:
        raise ValueError("Dimensions must be positive")
    return width * height
```

### JavaScript/TypeScript

```javascript
// Именование
const variableName = 'camelCase';
const CONSTANT_NAME = 'SCREAMING_SNAKE_CASE';
class ClassName { }

// Точки с запятой — зависит от стиля проекта
// С точками с запятой
const name = 'John';
function greet() { }

// Без (менее распространено)
const name = 'John'
function greet() { }

// Фигурные скобки
// K&R style (рекомендуется)
if (condition) {
    doSomething();
} else {
    doSomethingElse();
}

// Стрелочные функции
const add = (a, b) => a + b;

const processUser = (user) => {
    // обработка
    return result;
};
```

### Go

```go
// Go имеет встроенный форматтер gofmt
// Отступы: табуляция
// Фигурная скобка на той же строке (обязательно!)

func calculateTotal(items []Item) float64 {
    total := 0.0
    for _, item := range items {
        if item.IsActive {
            total += item.Price
        }
    }
    return total
}

// Экспортируемые имена — с большой буквы
type User struct {
    Name  string  // экспортируется
    email string  // не экспортируется
}
```

## Инструменты для поддержания стиля

### Форматтеры (Formatters)

Автоматически форматируют код:

```bash
# Python
black myfile.py          # Автоформатирование
autopep8 --in-place myfile.py

# JavaScript
prettier --write myfile.js

# Go
gofmt -w myfile.go
```

### Линтеры (Linters)

Проверяют код на соответствие стилю и потенциальные ошибки:

```bash
# Python
flake8 myfile.py
pylint myfile.py

# JavaScript
eslint myfile.js

# Go
golint myfile.go
go vet myfile.go
```

### Конфигурационные файлы

```yaml
# .flake8 или setup.cfg для Python
[flake8]
max-line-length = 88
exclude = .git,__pycache__,venv
ignore = E203, W503

# .prettierrc для JavaScript
{
  "semi": true,
  "singleQuote": true,
  "tabWidth": 2,
  "trailingComma": "es5"
}
```

```toml
# pyproject.toml для black
[tool.black]
line-length = 88
target-version = ['py39']
include = '\.pyi?$'
exclude = '''
/(
    \.git
  | \.venv
  | build
)/
'''
```

### Pre-commit хуки

Автоматическая проверка перед коммитом:

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/psf/black
    rev: 23.1.0
    hooks:
      - id: black

  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8

  - repo: https://github.com/pycqa/isort
    rev: 5.12.0
    hooks:
      - id: isort
```

## Типичные ошибки

### 1. Несогласованный стиль в проекте

```python
# Файл 1
def getUserName(userId):
    pass

# Файл 2
def get_user_email(user_id):
    pass

# Решение: выбрать один стиль и применить ко всему проекту
def get_user_name(user_id):
    pass

def get_user_email(user_id):
    pass
```

### 2. Слишком глубокая вложенность

```python
# Плохо — трудно читать
def process_order(order):
    if order:
        if order.is_valid():
            if order.has_items():
                for item in order.items:
                    if item.is_available():
                        if item.quantity > 0:
                            # наконец-то логика...
                            pass

# Хорошо — ранний возврат (guard clauses)
def process_order(order):
    if not order:
        return None

    if not order.is_valid():
        raise ValidationError("Invalid order")

    if not order.has_items():
        raise ValidationError("Order has no items")

    for item in order.items:
        if not item.is_available():
            continue

        if item.quantity <= 0:
            continue

        # логика обработки
        process_item(item)
```

### 3. Магические числа и строки

```python
# Плохо
if user.age >= 18:
    pass

if status == 1:
    pass

# Хорошо
MINIMUM_AGE = 18
STATUS_ACTIVE = 1

if user.age >= MINIMUM_AGE:
    pass

if status == STATUS_ACTIVE:
    pass
```

### 4. Закомментированный код

```python
# Плохо — мёртвый код загрязняет базу
def calculate_total(items):
    total = 0
    # old_total = sum(item.price for item in items)
    # if use_discount:
    #     total = old_total * 0.9
    for item in items:
        total += item.price
    return total

# Хорошо — удалите ненужный код, git сохранит историю
def calculate_total(items):
    total = 0
    for item in items:
        total += item.price
    return total
```

## Best Practices

### 1. Используйте автоформатирование

```bash
# Настройте IDE на автоформатирование при сохранении
# Или добавьте в CI/CD пайплайн
black --check src/
flake8 src/
```

### 2. Договоритесь о стиле в команде

- Выберите стандарт (PEP 8 для Python, Airbnb для JS)
- Задокументируйте исключения
- Используйте конфигурационные файлы в репозитории

### 3. Интегрируйте в CI/CD

```yaml
# GitHub Actions пример
- name: Check code style
  run: |
    black --check .
    flake8 .
    mypy .
```

### 4. Постепенная миграция

Если проект большой, мигрируйте постепенно:

```bash
# Форматируйте только изменённые файлы
black $(git diff --name-only --diff-filter=d HEAD~1 | grep '.py$')
```

## Примеры хорошего стиля

### Чистая функция с документацией

```python
from typing import List, Optional
from decimal import Decimal


def calculate_order_total(
    items: List[dict],
    discount_percent: Optional[Decimal] = None,
    tax_rate: Decimal = Decimal("0.20")
) -> Decimal:
    """
    Вычисляет итоговую сумму заказа с учётом скидки и налога.

    Args:
        items: Список товаров с ключами 'price' и 'quantity'.
        discount_percent: Процент скидки (0-100), опционально.
        tax_rate: Ставка налога (по умолчанию 20%).

    Returns:
        Итоговая сумма заказа.

    Example:
        >>> items = [{'price': Decimal('10'), 'quantity': 2}]
        >>> calculate_order_total(items, discount_percent=Decimal('10'))
        Decimal('21.60')
    """
    subtotal = sum(
        item['price'] * item['quantity']
        for item in items
    )

    if discount_percent:
        discount = subtotal * (discount_percent / 100)
        subtotal -= discount

    tax = subtotal * tax_rate
    total = subtotal + tax

    return total.quantize(Decimal('0.01'))
```

## Заключение

Стиль кода — это не просто эстетика. Это инструмент коммуникации между разработчиками. Единообразный стиль:

- Снижает когнитивную нагрузку
- Ускоряет code review
- Упрощает onboarding новых разработчиков
- Уменьшает количество ошибок

Главное правило: **будьте последовательны**. Какой бы стиль вы ни выбрали, применяйте его во всём проекте.
