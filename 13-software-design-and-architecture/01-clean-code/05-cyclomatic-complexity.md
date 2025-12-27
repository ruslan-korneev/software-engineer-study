# Cyclomatic Complexity

## Что такое цикломатическая сложность?

Цикломатическая сложность (Cyclomatic Complexity) — это метрика, измеряющая количество независимых путей выполнения в коде. Она была предложена Томасом Маккейбом в 1976 году.

Чем выше цикломатическая сложность, тем:
- Труднее понять код
- Больше тестовых случаев нужно написать
- Выше вероятность ошибок
- Сложнее поддерживать код

## Как вычисляется?

### Формула

```
CC = E - N + 2P

где:
E = количество рёбер (переходов) в графе потока управления
N = количество узлов (блоков кода)
P = количество связных компонентов (обычно 1 для одной функции)
```

### Упрощённый подсчёт

Для практического использования достаточно считать точки принятия решений:

```
CC = 1 + количество_точек_ветвления
```

**Точки ветвления:**
- `if`, `elif`, `else`
- `for`, `while`
- `case` в switch
- `and`, `or` в условиях
- `try/catch` (каждый catch)
- Тернарный оператор `? :`

## Примеры расчёта

### Пример 1: Линейный код (CC = 1)

```python
def greet(name: str) -> str:
    """CC = 1 — нет ветвлений."""
    greeting = f"Hello, {name}!"
    return greeting
```

### Пример 2: Одно условие (CC = 2)

```python
def is_adult(age: int) -> bool:
    """CC = 2 — одно ветвление."""
    if age >= 18:  # +1
        return True
    return False
```

### Пример 3: Несколько условий (CC = 4)

```python
def get_ticket_price(age: int) -> int:
    """CC = 4 — три точки ветвления."""
    if age < 3:        # +1
        return 0
    elif age < 12:     # +1
        return 50
    elif age < 65:     # +1
        return 100
    else:
        return 75
```

### Пример 4: Вложенные условия (CC = 5)

```python
def process_payment(user, amount):
    """CC = 5 — сложная логика."""
    if user.is_active:                    # +1
        if user.has_sufficient_balance:   # +1
            if amount > 0:                # +1
                if amount <= user.limit:  # +1
                    return process(amount)
    return None
```

### Пример 5: Логические операторы (CC = 4)

```python
def can_vote(citizen):
    """CC = 4 — and/or добавляют сложность."""
    # Каждый and/or — это дополнительный путь
    if citizen.age >= 18 and citizen.is_registered and not citizen.is_banned:
        #                 ^+1                       ^+1
        return True  # +1 для if
    return False
```

### Пример 6: Циклы (CC = 3)

```python
def find_first_even(numbers: list) -> int:
    """CC = 3 — цикл и условие."""
    for num in numbers:    # +1
        if num % 2 == 0:   # +1
            return num
    return -1
```

## Шкала оценки

| Сложность | Оценка риска | Рекомендации |
|-----------|--------------|--------------|
| 1-10 | Низкий риск | Код легко поддерживать |
| 11-20 | Умеренный риск | Требует внимания |
| 21-50 | Высокий риск | Рекомендуется рефакторинг |
| 50+ | Очень высокий | Требуется немедленный рефакторинг |

**Рекомендуемое значение: не более 10 для функции**

## Реальные примеры и рефакторинг

### Пример: Высокая сложность (CC = 12)

```python
def validate_user_input(data: dict) -> tuple:
    """
    Валидация входных данных пользователя.
    CC = 12 — слишком сложно!
    """
    errors = []

    if 'email' not in data:                    # +1
        errors.append("Email required")
    elif not data['email']:                    # +1
        errors.append("Email cannot be empty")
    elif '@' not in data['email']:             # +1
        errors.append("Invalid email format")

    if 'password' not in data:                 # +1
        errors.append("Password required")
    elif len(data['password']) < 8:            # +1
        errors.append("Password too short")
    elif not any(c.isupper() for c in data['password']):  # +1
        errors.append("Password needs uppercase")
    elif not any(c.isdigit() for c in data['password']):  # +1
        errors.append("Password needs digit")

    if 'age' in data:                          # +1
        if not isinstance(data['age'], int):   # +1
            errors.append("Age must be integer")
        elif data['age'] < 0:                  # +1
            errors.append("Age cannot be negative")
        elif data['age'] > 150:                # +1
            errors.append("Age seems invalid")

    return (len(errors) == 0, errors)
```

### После рефакторинга: Декомпозиция (CC = 2-3 для каждой)

```python
from typing import List, Tuple, Optional


def validate_user_input(data: dict) -> Tuple[bool, List[str]]:
    """
    Валидация входных данных пользователя.
    CC = 2 — намного проще!
    """
    errors = []

    errors.extend(validate_email(data.get('email')))
    errors.extend(validate_password(data.get('password')))
    errors.extend(validate_age(data.get('age')))

    return (len(errors) == 0, errors)


def validate_email(email: Optional[str]) -> List[str]:
    """CC = 3"""
    if not email:
        return ["Email required"]
    if '@' not in email:
        return ["Invalid email format"]
    return []


def validate_password(password: Optional[str]) -> List[str]:
    """CC = 4"""
    if not password:
        return ["Password required"]

    errors = []

    if len(password) < 8:
        errors.append("Password too short")
    if not any(c.isupper() for c in password):
        errors.append("Password needs uppercase")
    if not any(c.isdigit() for c in password):
        errors.append("Password needs digit")

    return errors


def validate_age(age: Optional[int]) -> List[str]:
    """CC = 3"""
    if age is None:
        return []

    if not isinstance(age, int):
        return ["Age must be integer"]
    if not 0 <= age <= 150:
        return ["Age must be between 0 and 150"]

    return []
```

## Техники снижения сложности

### 1. Ранний возврат (Guard Clauses)

```python
# Высокая сложность — вложенные условия
def process_order(order):
    if order:
        if order.is_valid():
            if order.has_items():
                if order.is_paid():
                    return ship_order(order)
    return None


# Низкая сложность — ранние возвраты
def process_order(order):
    if not order:
        return None
    if not order.is_valid():
        return None
    if not order.has_items():
        return None
    if not order.is_paid():
        return None

    return ship_order(order)
```

### 2. Полиморфизм вместо условий

```python
# Высокая сложность — switch/case через if
def calculate_area(shape):
    if shape['type'] == 'circle':
        return 3.14159 * shape['radius'] ** 2
    elif shape['type'] == 'rectangle':
        return shape['width'] * shape['height']
    elif shape['type'] == 'triangle':
        return 0.5 * shape['base'] * shape['height']
    elif shape['type'] == 'square':
        return shape['side'] ** 2
    else:
        raise ValueError("Unknown shape")


# Низкая сложность — полиморфизм
from abc import ABC, abstractmethod


class Shape(ABC):
    @abstractmethod
    def area(self) -> float:
        pass


class Circle(Shape):
    def __init__(self, radius: float):
        self.radius = radius

    def area(self) -> float:
        return 3.14159 * self.radius ** 2


class Rectangle(Shape):
    def __init__(self, width: float, height: float):
        self.width = width
        self.height = height

    def area(self) -> float:
        return self.width * self.height


# Использование
def calculate_area(shape: Shape) -> float:
    return shape.area()  # CC = 1
```

### 3. Словари вместо switch

```python
# Высокая сложность
def get_http_message(code: int) -> str:
    if code == 200:
        return "OK"
    elif code == 201:
        return "Created"
    elif code == 400:
        return "Bad Request"
    elif code == 401:
        return "Unauthorized"
    elif code == 404:
        return "Not Found"
    elif code == 500:
        return "Internal Server Error"
    else:
        return "Unknown"


# Низкая сложность
HTTP_MESSAGES = {
    200: "OK",
    201: "Created",
    400: "Bad Request",
    401: "Unauthorized",
    404: "Not Found",
    500: "Internal Server Error",
}


def get_http_message(code: int) -> str:
    return HTTP_MESSAGES.get(code, "Unknown")  # CC = 1
```

### 4. Стратегия вместо условий

```python
from typing import Protocol


# Вместо множества if-else для разных стратегий
class PaymentProcessor(Protocol):
    def process(self, amount: float) -> bool:
        pass


class CreditCardProcessor:
    def process(self, amount: float) -> bool:
        # Логика для кредитной карты
        return True


class PayPalProcessor:
    def process(self, amount: float) -> bool:
        # Логика для PayPal
        return True


class CryptoProcessor:
    def process(self, amount: float) -> bool:
        # Логика для криптовалюты
        return True


# Фабрика для создания процессоров
PROCESSORS = {
    'credit_card': CreditCardProcessor,
    'paypal': PayPalProcessor,
    'crypto': CryptoProcessor,
}


def process_payment(method: str, amount: float) -> bool:
    processor_class = PROCESSORS.get(method)
    if not processor_class:
        raise ValueError(f"Unknown payment method: {method}")

    processor = processor_class()
    return processor.process(amount)  # CC = 2
```

### 5. Функциональный подход

```python
# Вместо сложных циклов с условиями
def process_users(users):
    result = []
    for user in users:
        if user.is_active:
            if user.age >= 18:
                if user.has_verified_email:
                    result.append({
                        'id': user.id,
                        'name': user.name.upper()
                    })
    return result


# Функциональный подход
def process_users(users):
    return [
        {'id': user.id, 'name': user.name.upper()}
        for user in users
        if is_eligible(user)
    ]


def is_eligible(user) -> bool:
    return (
        user.is_active
        and user.age >= 18
        and user.has_verified_email
    )
```

## Инструменты измерения

### Python

```bash
# radon — популярный инструмент для Python
pip install radon

# Измерение цикломатической сложности
radon cc my_module.py -a

# Вывод:
# my_module.py
#     F 1:0 my_function - A (3)
#     F 10:0 complex_function - C (15)
#
# Average complexity: B (9.0)

# Уровни:
# A (1-5) - низкая сложность
# B (6-10) - умеренная
# C (11-20) - высокая
# D (21-30) - очень высокая
# F (31+) - критическая
```

### Конфигурация линтеров

```ini
# .flake8 или setup.cfg
[flake8]
max-complexity = 10

# pylint
[DESIGN]
max-complexity = 10
```

```yaml
# .pre-commit-config.yaml
repos:
  - repo: https://github.com/pycqa/flake8
    rev: 6.0.0
    hooks:
      - id: flake8
        args: ['--max-complexity=10']
```

### JavaScript/TypeScript

```bash
# ESLint с правилом complexity
npm install eslint --save-dev
```

```json
// .eslintrc.json
{
  "rules": {
    "complexity": ["error", 10]
  }
}
```

## Best Practices

### 1. Устанавливайте лимиты в CI/CD

```yaml
# GitHub Actions
- name: Check complexity
  run: |
    radon cc src/ --min C --show-complexity
    if [ $? -ne 0 ]; then
      echo "High complexity detected!"
      exit 1
    fi
```

### 2. Рефакторьте постепенно

Не пытайтесь исправить всё сразу. Начните с самых сложных функций:

```bash
# Найти самые сложные функции
radon cc src/ --min C --order SCORE
```

### 3. Пишите тесты перед рефакторингом

Высокая сложность = высокий риск регрессий. Покройте тестами перед изменениями.

### 4. Документируйте сложные алгоритмы

Если сложность неизбежна (сложный алгоритм), документируйте:

```python
def complex_matching_algorithm(patterns, text):
    """
    Реализация алгоритма Ахо-Корасик для поиска множества паттернов.

    Цикломатическая сложность высокая из-за природы алгоритма.
    Не рефакторить без понимания алгоритма!

    Complexity: O(n + m + z), где z — количество совпадений.
    """
    # ... сложная логика
```

## Связь с тестированием

Цикломатическая сложность показывает **минимальное** количество тестов:

```python
def categorize_age(age: int) -> str:
    """CC = 4 — нужно минимум 4 теста."""
    if age < 0:
        return "invalid"
    elif age < 13:
        return "child"
    elif age < 20:
        return "teenager"
    else:
        return "adult"


# Минимальные тесты
def test_categorize_age():
    assert categorize_age(-1) == "invalid"  # Путь 1
    assert categorize_age(5) == "child"     # Путь 2
    assert categorize_age(15) == "teenager" # Путь 3
    assert categorize_age(25) == "adult"    # Путь 4
```

## Заключение

Цикломатическая сложность — важная метрика качества кода:

1. **Держите CC ≤ 10** для каждой функции
2. **Используйте инструменты** для автоматической проверки
3. **Применяйте техники** снижения сложности:
   - Ранний возврат
   - Полиморфизм
   - Словари вместо switch
   - Декомпозиция
4. **Интегрируйте в CI/CD** для предотвращения роста сложности

Помните: простой код — это код, который легко читать, тестировать и изменять.
