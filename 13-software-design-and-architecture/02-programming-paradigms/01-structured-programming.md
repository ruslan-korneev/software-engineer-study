# Структурное программирование

## Что такое структурное программирование?

**Структурное программирование** — это парадигма программирования, основанная на использовании подпрограмм, блочных структур и циклов вместо простых условных переходов (goto). Эта парадигма появилась в конце 1960-х годов как ответ на "кризис программного обеспечения" и была популяризирована Эдсгером Дейкстрой.

## Основные принципы

### 1. Теорема Бёма-Якопини

Любой алгоритм может быть выражен с помощью трёх базовых управляющих структур:

1. **Последовательность (Sequence)** — выполнение инструкций одна за другой
2. **Ветвление (Selection)** — условный выбор между двумя или более путями (if/else, switch)
3. **Итерация (Iteration)** — повторение блока кода (for, while, do-while)

### 2. Отказ от GOTO

Одна из ключевых идей — избегать оператора `goto`, который приводит к "спагетти-коду":

```python
# Плохо: использование goto-подобной логики
# (Python не поддерживает goto, но концептуально)

# Хорошо: структурированный код
def process_items(items):
    for item in items:
        if not is_valid(item):
            continue
        process(item)
```

### 3. Модульность

Код разбивается на небольшие, независимые подпрограммы (функции, процедуры):

```python
# Структурированный подход с модульностью
def validate_user_input(data):
    """Валидация входных данных пользователя"""
    if not data:
        return False
    if len(data) < 3:
        return False
    return True

def sanitize_input(data):
    """Очистка входных данных"""
    return data.strip().lower()

def process_user_data(data):
    """Основная функция обработки"""
    if not validate_user_input(data):
        return None

    clean_data = sanitize_input(data)
    return clean_data
```

## Управляющие структуры

### Последовательность

```python
# Инструкции выполняются по порядку
name = input("Введите имя: ")
name = name.strip()
name = name.capitalize()
print(f"Привет, {name}!")
```

### Ветвление (Selection)

```python
# if-elif-else
def get_grade(score):
    if score >= 90:
        return "A"
    elif score >= 80:
        return "B"
    elif score >= 70:
        return "C"
    elif score >= 60:
        return "D"
    else:
        return "F"

# match-case (Python 3.10+)
def get_day_type(day):
    match day:
        case "Saturday" | "Sunday":
            return "Weekend"
        case _:
            return "Weekday"
```

### Итерация (Loops)

```python
# for loop - когда известно количество итераций
def sum_list(numbers):
    total = 0
    for num in numbers:
        total += num
    return total

# while loop - когда условие определяет завершение
def find_first_even(numbers):
    index = 0
    while index < len(numbers):
        if numbers[index] % 2 == 0:
            return numbers[index]
        index += 1
    return None

# do-while эквивалент в Python
def get_valid_input():
    while True:
        user_input = input("Введите положительное число: ")
        if user_input.isdigit() and int(user_input) > 0:
            return int(user_input)
        print("Неверный ввод, попробуйте снова")
```

## Пример: Рефакторинг неструктурированного кода

### До (неструктурированный подход)

```python
# Сложный для понимания код с вложенной логикой
def process_order(order):
    if order:
        if order.get('items'):
            total = 0
            for item in order['items']:
                if item.get('price'):
                    if item.get('quantity'):
                        total += item['price'] * item['quantity']
            if total > 0:
                if order.get('discount'):
                    total = total * (1 - order['discount'])
                return total
    return 0
```

### После (структурированный подход)

```python
def validate_order(order):
    """Проверяет валидность заказа"""
    if not order:
        return False
    if not order.get('items'):
        return False
    return True

def calculate_item_total(item):
    """Вычисляет стоимость одной позиции"""
    price = item.get('price', 0)
    quantity = item.get('quantity', 0)
    return price * quantity

def calculate_subtotal(items):
    """Вычисляет промежуточную сумму"""
    return sum(calculate_item_total(item) for item in items)

def apply_discount(total, discount):
    """Применяет скидку к сумме"""
    if discount and 0 < discount < 1:
        return total * (1 - discount)
    return total

def process_order(order):
    """Главная функция обработки заказа"""
    if not validate_order(order):
        return 0

    subtotal = calculate_subtotal(order['items'])

    if subtotal <= 0:
        return 0

    return apply_discount(subtotal, order.get('discount'))
```

## Best Practices

### 1. Один вход — один выход

Стремитесь к функциям с единственной точкой возврата:

```python
# Менее предпочтительно: множественные return
def check_value(x):
    if x < 0:
        return "negative"
    if x == 0:
        return "zero"
    return "positive"

# Более структурировано: одна точка выхода
def check_value_structured(x):
    result = None
    if x < 0:
        result = "negative"
    elif x == 0:
        result = "zero"
    else:
        result = "positive"
    return result
```

### 2. Ограничение вложенности

Избегайте глубокой вложенности (более 3 уровней):

```python
# Плохо: глубокая вложенность
def process(data):
    if data:
        if data.valid:
            if data.items:
                for item in data.items:
                    if item.active:
                        # обработка
                        pass

# Хорошо: ранний выход и плоская структура
def process_better(data):
    if not data or not data.valid or not data.items:
        return

    active_items = [item for item in data.items if item.active]
    for item in active_items:
        # обработка
        pass
```

### 3. Маленькие функции

Каждая функция должна делать одну вещь:

```python
# Плохо: функция делает слишком много
def handle_user_registration(email, password, name):
    # Валидация email
    # Валидация пароля
    # Хеширование пароля
    # Создание пользователя в БД
    # Отправка письма подтверждения
    # Логирование
    pass

# Хорошо: разделение ответственности
def validate_email(email):
    pass

def validate_password(password):
    pass

def hash_password(password):
    pass

def create_user_in_db(email, password_hash, name):
    pass

def send_confirmation_email(email):
    pass

def handle_user_registration(email, password, name):
    validate_email(email)
    validate_password(password)
    password_hash = hash_password(password)
    create_user_in_db(email, password_hash, name)
    send_confirmation_email(email)
```

## Типичные ошибки

### 1. Избыточная сложность условий

```python
# Плохо
if not (x != 5 and not y):
    do_something()

# Хорошо
if x == 5 or y:
    do_something()
```

### 2. Дублирование кода

```python
# Плохо: дублирование
def process_admin_user(user):
    print(f"Processing: {user.name}")
    validate(user)
    save(user)
    log(f"Saved admin: {user.name}")

def process_regular_user(user):
    print(f"Processing: {user.name}")
    validate(user)
    save(user)
    log(f"Saved user: {user.name}")

# Хорошо: общая функция
def process_user(user, user_type="user"):
    print(f"Processing: {user.name}")
    validate(user)
    save(user)
    log(f"Saved {user_type}: {user.name}")
```

### 3. Магические числа

```python
# Плохо
if user.age >= 18:
    allow_access()

# Хорошо
MINIMUM_AGE = 18

if user.age >= MINIMUM_AGE:
    allow_access()
```

## Преимущества структурного программирования

1. **Читаемость** — код легче понять и поддерживать
2. **Тестируемость** — модульные функции легко тестировать изолированно
3. **Отладка** — ошибки легче локализовать
4. **Повторное использование** — функции можно использовать в разных частях программы
5. **Командная работа** — разные разработчики могут работать над разными модулями

## Связь с другими парадигмами

Структурное программирование является фундаментом для:
- **Процедурного программирования** — расширяет идею подпрограмм
- **Объектно-ориентированного программирования** — методы классов следуют структурным принципам
- **Функционального программирования** — чистые функции используют структурные конструкции

## Заключение

Структурное программирование — это базовая парадигма, принципы которой актуальны и сегодня. Даже при использовании ООП или ФП, внутри методов и функций мы применяем структурные конструкции: последовательность, ветвление и итерацию. Понимание этих основ критически важно для написания качественного кода.
