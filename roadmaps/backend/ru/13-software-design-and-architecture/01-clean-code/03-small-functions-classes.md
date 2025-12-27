# Small Functions and Classes

## Почему размер имеет значение?

Маленькие функции и классы — один из ключевых принципов чистого кода. Компактный код:

- **Легче читать**: меньше информации для восприятия за раз
- **Легче тестировать**: изолированные единицы поведения
- **Легче переиспользовать**: маленькие компоненты более универсальны
- **Легче изменять**: изменения локализованы
- **Меньше багов**: простой код = меньше ошибок

> "Первое правило функций — они должны быть маленькими. Второе правило — они должны быть ещё меньше." — Роберт Мартин

## Принципы для функций

### 1. Функция должна делать одну вещь

Функция должна делать что-то одно, делать это хорошо и не делать ничего другого.

```python
# Плохо — функция делает слишком много
def process_user_registration(user_data):
    # Валидация
    if not user_data.get('email'):
        raise ValueError("Email required")
    if not user_data.get('password'):
        raise ValueError("Password required")
    if len(user_data['password']) < 8:
        raise ValueError("Password too short")

    # Хеширование пароля
    import hashlib
    password_hash = hashlib.sha256(
        user_data['password'].encode()
    ).hexdigest()

    # Создание пользователя
    user = {
        'email': user_data['email'],
        'password_hash': password_hash,
        'created_at': datetime.now()
    }

    # Сохранение в БД
    db.users.insert(user)

    # Отправка email
    send_email(
        to=user_data['email'],
        subject="Welcome!",
        body="Thanks for registering!"
    )

    return user


# Хорошо — разбито на отдельные функции
def register_user(user_data: dict) -> User:
    """Регистрирует нового пользователя."""
    validate_registration_data(user_data)
    user = create_user(user_data)
    save_user(user)
    send_welcome_email(user)
    return user


def validate_registration_data(data: dict) -> None:
    """Проверяет данные регистрации."""
    if not data.get('email'):
        raise ValueError("Email required")
    if not data.get('password'):
        raise ValueError("Password required")
    validate_password_strength(data['password'])


def validate_password_strength(password: str) -> None:
    """Проверяет надёжность пароля."""
    if len(password) < 8:
        raise ValueError("Password too short")


def create_user(data: dict) -> User:
    """Создаёт объект пользователя."""
    return User(
        email=data['email'],
        password_hash=hash_password(data['password']),
        created_at=datetime.now()
    )


def hash_password(password: str) -> str:
    """Хеширует пароль."""
    import hashlib
    return hashlib.sha256(password.encode()).hexdigest()


def save_user(user: User) -> None:
    """Сохраняет пользователя в базу данных."""
    db.users.insert(user.to_dict())


def send_welcome_email(user: User) -> None:
    """Отправляет приветственное письмо."""
    send_email(
        to=user.email,
        subject="Welcome!",
        body="Thanks for registering!"
    )
```

### 2. Один уровень абстракции

Все операции в функции должны быть на одном уровне абстракции:

```python
# Плохо — смешаны уровни абстракции
def generate_report(data):
    # Высокий уровень
    report = create_report_header()

    # Низкий уровень — работа со строками
    for row in data:
        line = f"{row['name']},{row['value']},{row['date']}"
        report += line + "\n"

    # Высокий уровень
    add_report_footer(report)

    # Низкий уровень — работа с файлами
    with open('report.csv', 'w') as f:
        f.write(report)


# Хорошо — единый уровень абстракции
def generate_report(data):
    """Генерирует отчёт из данных."""
    report = create_report_header()
    report += format_data_rows(data)
    report += create_report_footer()
    save_report(report)


def format_data_rows(data):
    """Форматирует строки данных для отчёта."""
    return '\n'.join(format_row(row) for row in data)


def format_row(row):
    """Форматирует одну строку данных."""
    return f"{row['name']},{row['value']},{row['date']}"


def save_report(report):
    """Сохраняет отчёт в файл."""
    with open('report.csv', 'w') as f:
        f.write(report)
```

### 3. Минимум аргументов

Идеально — 0-2 аргумента. Больше 3 — сигнал к рефакторингу.

```python
# Плохо — слишком много аргументов
def create_user(
    name, email, password, age, address,
    city, country, phone, is_admin, department
):
    pass


# Хорошо — группировка в объекты
from dataclasses import dataclass


@dataclass
class UserData:
    name: str
    email: str
    password: str
    age: int


@dataclass
class Address:
    street: str
    city: str
    country: str


@dataclass
class ContactInfo:
    phone: str
    address: Address


def create_user(user_data: UserData, contact: ContactInfo) -> User:
    pass


# Или используйте Builder pattern
user = UserBuilder() \
    .with_name("John") \
    .with_email("john@example.com") \
    .with_password("secret123") \
    .build()
```

### 4. Избегайте флагов-аргументов

Флаг boolean в аргументе — признак того, что функция делает более одной вещи:

```python
# Плохо — флаг указывает на две разные операции
def get_users(include_inactive=False):
    if include_inactive:
        return db.users.all()
    else:
        return db.users.filter(active=True)


# Хорошо — две отдельные функции
def get_active_users():
    return db.users.filter(active=True)


def get_all_users():
    return db.users.all()


# Или если нужна гибкость
def get_users(status: UserStatus = UserStatus.ACTIVE):
    return db.users.filter(status=status)
```

### 5. Не более 20 строк

Рекомендуемый размер функции — 5-20 строк. Если больше, ищите возможности для декомпозиции:

```python
# Плохо — функция слишком длинная
def process_order(order):
    # 50+ строк кода...
    pass


# Хорошо — разбито на логические блоки
def process_order(order: Order) -> ProcessedOrder:
    """Обрабатывает заказ."""
    validate_order(order)
    inventory = reserve_inventory(order)
    payment = process_payment(order)
    confirmation = create_confirmation(order, payment)
    notify_customer(order, confirmation)
    return ProcessedOrder(order, inventory, payment, confirmation)
```

## Принципы для классов

### 1. Single Responsibility Principle (SRP)

Класс должен иметь только одну причину для изменения:

```python
# Плохо — класс делает слишком много
class User:
    def __init__(self, name, email):
        self.name = name
        self.email = email

    def save(self):
        # Логика сохранения в БД
        db.execute("INSERT INTO users ...")

    def send_email(self, subject, body):
        # Логика отправки email
        smtp.send(self.email, subject, body)

    def generate_report(self):
        # Логика генерации отчёта
        return f"User Report: {self.name}..."


# Хорошо — разделение ответственностей
class User:
    """Представляет пользователя системы."""

    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email


class UserRepository:
    """Отвечает за сохранение пользователей."""

    def save(self, user: User) -> None:
        db.execute("INSERT INTO users ...", user)

    def find_by_email(self, email: str) -> Optional[User]:
        return db.query("SELECT * FROM users WHERE email = ?", email)


class EmailService:
    """Отвечает за отправку email."""

    def send(self, to: str, subject: str, body: str) -> None:
        smtp.send(to, subject, body)


class UserReportGenerator:
    """Генерирует отчёты о пользователях."""

    def generate(self, user: User) -> str:
        return f"User Report: {user.name}..."
```

### 2. Маленький публичный интерфейс

Минимизируйте количество публичных методов:

```python
# Плохо — слишком много публичных методов
class OrderProcessor:
    def validate_items(self): pass
    def check_inventory(self): pass
    def calculate_subtotal(self): pass
    def apply_discounts(self): pass
    def calculate_tax(self): pass
    def calculate_shipping(self): pass
    def process_payment(self): pass
    def update_inventory(self): pass
    def send_confirmation(self): pass


# Хорошо — один понятный публичный метод
class OrderProcessor:
    """Обрабатывает заказы."""

    def process(self, order: Order) -> ProcessedOrder:
        """Обрабатывает заказ и возвращает результат."""
        self._validate_items(order)
        self._check_inventory(order)
        total = self._calculate_total(order)
        payment = self._process_payment(order, total)
        self._update_inventory(order)
        self._send_confirmation(order, payment)
        return ProcessedOrder(order, payment)

    def _validate_items(self, order: Order) -> None:
        """Валидирует товары в заказе."""
        pass

    def _check_inventory(self, order: Order) -> None:
        """Проверяет наличие на складе."""
        pass

    def _calculate_total(self, order: Order) -> Decimal:
        """Вычисляет итоговую сумму."""
        subtotal = self._calculate_subtotal(order)
        discounted = self._apply_discounts(subtotal, order)
        with_tax = self._add_tax(discounted)
        with_shipping = self._add_shipping(with_tax, order)
        return with_shipping

    # ... другие приватные методы
```

### 3. Высокая связность (Cohesion)

Все методы класса должны использовать большинство его полей:

```python
# Плохо — низкая связность
class Employee:
    def __init__(self):
        self.name = ""
        self.department = ""
        self.salary = 0
        self.vacation_days = 0
        self.sick_days = 0
        self.projects = []

    def get_salary_info(self):
        # Использует только salary
        return self.salary

    def calculate_vacation(self):
        # Использует только vacation_days, sick_days
        return self.vacation_days + self.sick_days

    def get_project_count(self):
        # Использует только projects
        return len(self.projects)


# Хорошо — высокая связность, разделение на классы
class Employee:
    def __init__(self, name: str, department: str):
        self.name = name
        self.department = department


class Compensation:
    def __init__(self, base_salary: Decimal, bonus: Decimal):
        self.base_salary = base_salary
        self.bonus = bonus

    def total(self) -> Decimal:
        return self.base_salary + self.bonus


class TimeOff:
    def __init__(self, vacation_days: int, sick_days: int):
        self.vacation_days = vacation_days
        self.sick_days = sick_days

    def total_available(self) -> int:
        return self.vacation_days + self.sick_days

    def use_vacation(self, days: int) -> None:
        self.vacation_days -= days


class ProjectAssignment:
    def __init__(self):
        self.projects: List[Project] = []

    def assign(self, project: Project) -> None:
        self.projects.append(project)

    def count(self) -> int:
        return len(self.projects)
```

### 4. Рекомендуемый размер класса

- **Не более 200-300 строк** — сигнал к разбиению
- **5-10 публичных методов** — если больше, возможно, класс делает много
- **Не более 5-7 полей** — высокая связность

```python
# Признаки того, что класс слишком большой:
# 1. Трудно придумать короткое имя
# 2. Методы группируются по назначению
# 3. Некоторые методы не используют все поля

# Решение: выделить группы методов в отдельные классы
class LargeClass:
    # Группа 1: работа с данными
    def load_data(self): pass
    def save_data(self): pass
    def validate_data(self): pass

    # Группа 2: форматирование
    def format_for_display(self): pass
    def format_for_export(self): pass

    # Группа 3: уведомления
    def send_email(self): pass
    def send_sms(self): pass


# После рефакторинга:
class DataHandler:
    def load(self): pass
    def save(self): pass
    def validate(self): pass


class Formatter:
    def for_display(self, data): pass
    def for_export(self, data): pass


class NotificationService:
    def send_email(self, message): pass
    def send_sms(self, message): pass
```

## Примеры рефакторинга

### Пример 1: Извлечение метода

```python
# До рефакторинга
def print_invoice(invoice):
    print("=" * 40)
    print("INVOICE")
    print("=" * 40)
    print(f"Customer: {invoice.customer_name}")
    print(f"Date: {invoice.date}")
    print("-" * 40)

    total = 0
    for item in invoice.items:
        line_total = item.price * item.quantity
        total += line_total
        print(f"{item.name}: {item.quantity} x ${item.price} = ${line_total}")

    print("-" * 40)
    print(f"Subtotal: ${total}")

    tax = total * 0.1
    print(f"Tax (10%): ${tax}")

    grand_total = total + tax
    print(f"TOTAL: ${grand_total}")
    print("=" * 40)


# После рефакторинга
def print_invoice(invoice: Invoice) -> None:
    """Печатает счёт."""
    print_header(invoice)
    subtotal = print_items(invoice.items)
    print_totals(subtotal)


def print_header(invoice: Invoice) -> None:
    """Печатает заголовок счёта."""
    print("=" * 40)
    print("INVOICE")
    print("=" * 40)
    print(f"Customer: {invoice.customer_name}")
    print(f"Date: {invoice.date}")
    print("-" * 40)


def print_items(items: List[InvoiceItem]) -> Decimal:
    """Печатает позиции и возвращает сумму."""
    total = Decimal(0)
    for item in items:
        line_total = calculate_line_total(item)
        total += line_total
        print(format_item_line(item, line_total))
    return total


def calculate_line_total(item: InvoiceItem) -> Decimal:
    """Вычисляет сумму по позиции."""
    return item.price * item.quantity


def format_item_line(item: InvoiceItem, total: Decimal) -> str:
    """Форматирует строку позиции."""
    return f"{item.name}: {item.quantity} x ${item.price} = ${total}"


def print_totals(subtotal: Decimal) -> None:
    """Печатает итоги."""
    print("-" * 40)
    print(f"Subtotal: ${subtotal}")

    tax = calculate_tax(subtotal)
    print(f"Tax (10%): ${tax}")

    grand_total = subtotal + tax
    print(f"TOTAL: ${grand_total}")
    print("=" * 40)


def calculate_tax(amount: Decimal, rate: Decimal = Decimal("0.1")) -> Decimal:
    """Вычисляет налог."""
    return amount * rate
```

### Пример 2: Извлечение класса

```python
# До рефакторинга
class Person:
    def __init__(self):
        self.name = ""
        self.street = ""
        self.city = ""
        self.state = ""
        self.zip_code = ""

    def get_full_address(self):
        return f"{self.street}, {self.city}, {self.state} {self.zip_code}"

    def is_valid_address(self):
        return all([self.street, self.city, self.state, self.zip_code])


# После рефакторинга
@dataclass
class Address:
    """Представляет почтовый адрес."""
    street: str
    city: str
    state: str
    zip_code: str

    def full(self) -> str:
        """Возвращает полный адрес."""
        return f"{self.street}, {self.city}, {self.state} {self.zip_code}"

    def is_valid(self) -> bool:
        """Проверяет валидность адреса."""
        return all([self.street, self.city, self.state, self.zip_code])


@dataclass
class Person:
    """Представляет человека."""
    name: str
    address: Address
```

## Типичные ошибки

### 1. Функции с побочными эффектами в названии

```python
# Плохо — имя не отражает побочный эффект
def check_password(user, password):
    if user.password == hash(password):
        Session.initialize()  # Скрытый побочный эффект!
        return True
    return False


# Хорошо — явное разделение
def verify_password(user, password):
    return user.password == hash(password)


def login(user, password):
    if verify_password(user, password):
        Session.initialize()
        return True
    return False
```

### 2. Классы с названием на -er или -or

```python
# Часто признак процедурного кода в ООП обёртке
class UserManager:
    def create_user(self, data): pass
    def update_user(self, user): pass
    def delete_user(self, user): pass


# Лучше — использовать репозитории, сервисы с конкретной ответственностью
class UserRepository:  # Конкретная ответственность: хранение
    def save(self, user): pass
    def find(self, id): pass
    def delete(self, id): pass


class UserRegistrationService:  # Конкретная ответственность: регистрация
    def register(self, data): pass
```

### 3. God classes

```python
# Плохо — класс знает и делает всё
class Application:
    def connect_database(self): pass
    def run_migrations(self): pass
    def start_server(self): pass
    def handle_request(self): pass
    def render_template(self): pass
    def send_email(self): pass
    def generate_report(self): pass
    # ... ещё 100 методов


# Хорошо — разделение на компоненты
class Application:
    def __init__(self):
        self.database = Database()
        self.server = Server()
        self.mailer = Mailer()

    def start(self):
        self.database.connect()
        self.server.start()
```

## Best Practices

1. **Правило 5 строк**: если функция больше 5 строк, подумайте о разбиении
2. **Один экран**: функция должна помещаться на одном экране без прокрутки
3. **Именование**: если трудно назвать функцию, она делает слишком много
4. **Тестируемость**: если функцию трудно тестировать, она слишком сложная
5. **Ревью**: при code review обращайте внимание на размер функций и классов

## Заключение

Маленькие функции и классы — это не цель сама по себе, а средство достижения:
- Читаемого кода
- Тестируемого кода
- Переиспользуемого кода
- Легко изменяемого кода

Помните: **проще понять 10 маленьких функций по 5 строк, чем одну функцию на 50 строк**.
