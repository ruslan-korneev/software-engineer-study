# Naming Conventions

## Что такое соглашения об именовании?

Соглашения об именовании (Naming Conventions) — это набор правил и рекомендаций для выбора имён переменных, функций, классов, модулей и других элементов кода. Правильное именование — один из важнейших аспектов чистого кода, поскольку код читается гораздо чаще, чем пишется.

> "Имена переменных должны раскрывать намерение" — Роберт Мартин, "Чистый код"

## Основные принципы именования

### 1. Имена должны выражать намерение

Имя должно отвечать на три вопроса:
- Зачем эта переменная/функция существует?
- Что она делает?
- Как она используется?

```python
# Плохо - непонятно, что означает d
d = 5

# Хорошо - понятно назначение переменной
days_since_last_login = 5

# Плохо - что делает функция?
def process(data):
    pass

# Хорошо - понятно назначение
def validate_user_credentials(credentials):
    pass
```

### 2. Избегайте дезинформации

Не используйте имена, которые могут ввести в заблуждение:

```python
# Плохо - это не список, а словарь
user_list = {"name": "John", "age": 30}

# Хорошо
user_data = {"name": "John", "age": 30}

# Плохо - O и l похожи на 0 и 1
O = 0
l = 1

# Хорошо
zero_count = 0
line_number = 1
```

### 3. Используйте произносимые имена

```python
# Плохо - невозможно произнести
genymdhms = "2024-01-15"
cstmr_nm = "John"

# Хорошо
generation_timestamp = "2024-01-15"
customer_name = "John"
```

### 4. Используйте имена, удобные для поиска

```python
# Плохо - невозможно найти поиском
for i in range(7):
    if status == 4:
        pass

# Хорошо - легко искать по кодовой базе
DAYS_IN_WEEK = 7
STATUS_COMPLETED = 4

for day in range(DAYS_IN_WEEK):
    if status == STATUS_COMPLETED:
        pass
```

## Стили именования

### camelCase
Первое слово с маленькой буквы, последующие — с большой. Используется для переменных и методов в JavaScript, Java.

```javascript
let userName = "John";
function getUserById(userId) { }
```

### PascalCase
Каждое слово с большой буквы. Используется для классов и конструкторов.

```python
class UserAccount:
    pass

class DatabaseConnection:
    pass
```

### snake_case
Слова разделены подчёркиванием. Стандарт для Python.

```python
user_name = "John"
def get_user_by_id(user_id):
    pass
```

### SCREAMING_SNAKE_CASE
Верхний регистр с подчёркиванием. Используется для констант.

```python
MAX_CONNECTIONS = 100
DATABASE_URL = "postgres://localhost:5432"
API_VERSION = "v2"
```

### kebab-case
Слова разделены дефисом. Используется в URL, CSS-классах, именах файлов.

```
user-profile.html
background-color: red;
/api/user-accounts
```

## Правила для разных элементов кода

### Переменные

```python
# Существительные, описывающие содержимое
user_count = 42
active_users = []
current_page = 1

# Булевы переменные — с префиксами is_, has_, can_, should_
is_active = True
has_permission = False
can_edit = True
should_notify = False

# Коллекции — множественное число
users = []
order_items = {}
selected_products = set()
```

### Функции и методы

```python
# Глаголы, описывающие действие
def calculate_total_price(items):
    pass

def send_email_notification(user, message):
    pass

def validate_input_data(data):
    pass

# Геттеры и сеттеры
def get_user_name(self):
    return self._name

def set_user_name(self, name):
    self._name = name

# Предикаты (возвращают bool)
def is_valid_email(email):
    pass

def has_access_to_resource(user, resource):
    pass

def can_user_edit_document(user, document):
    pass
```

### Классы

```python
# Существительные или именные группы
class User:
    pass

class OrderProcessor:
    pass

class EmailNotificationService:
    pass

class DatabaseConnectionPool:
    pass

# Избегайте общих имён
# Плохо
class Manager:
    pass

class Processor:
    pass

class Data:
    pass

# Хорошо
class UserAccountManager:
    pass

class PaymentProcessor:
    pass

class CustomerData:
    pass
```

### Константы

```python
# Верхний регистр, описательные имена
MAX_RETRY_ATTEMPTS = 3
DEFAULT_TIMEOUT_SECONDS = 30
HTTP_STATUS_OK = 200
DATABASE_CONNECTION_STRING = "postgresql://localhost:5432/mydb"
```

## Типичные ошибки

### 1. Слишком короткие имена

```python
# Плохо
def c(u, p):
    return u + p

# Хорошо
def calculate_total(unit_price, quantity):
    return unit_price * quantity
```

### 2. Слишком длинные имена

```python
# Плохо
def get_all_active_users_from_database_who_logged_in_today():
    pass

# Хорошо
def get_todays_active_users():
    pass
```

### 3. Венгерская нотация (устарела)

```python
# Плохо - тип уже определён языком
str_name = "John"
int_age = 30
lst_users = []

# Хорошо
name = "John"
age = 30
users = []
```

### 4. Избыточный контекст

```python
# Плохо - класс уже в контексте User
class User:
    def __init__(self):
        self.user_name = ""
        self.user_email = ""
        self.user_age = 0

# Хорошо
class User:
    def __init__(self):
        self.name = ""
        self.email = ""
        self.age = 0
```

### 5. Однобуквенные переменные

```python
# Допустимо только в коротких циклах
for i in range(10):
    print(i)

# Для координат
x, y, z = get_coordinates()

# Для lambda
sorted_users = sorted(users, key=lambda u: u.name)

# Во всех остальных случаях — плохо
def calculate(a, b, c, d):  # Что это?
    return a * b + c / d
```

## Best Practices

### 1. Согласованность в проекте

Выберите стиль и придерживайтесь его во всём проекте:

```python
# Выберите один стиль и используйте везде
# Не смешивайте:
getUserName()  # camelCase
get_user_email()  # snake_case
GetUserAge()  # PascalCase
```

### 2. Используйте доменную терминологию

```python
# Если работаете с электронной коммерцией
class ShoppingCart:
    def add_item(self, product, quantity):
        pass

    def apply_discount_code(self, code):
        pass

    def calculate_subtotal(self):
        pass

# Если работаете с банковской сферой
class BankAccount:
    def deposit(self, amount):
        pass

    def withdraw(self, amount):
        pass

    def get_balance(self):
        pass
```

### 3. Ревью и рефакторинг имён

При code review обращайте внимание на:
- Понятность имён без дополнительного контекста
- Соответствие имени реальному поведению
- Консистентность с остальным кодом

```python
# Если функция изменилась, измените и имя
# Было: получали одного пользователя
def get_user(user_id):
    pass

# Стало: получаем список пользователей
# Имя должно измениться!
def get_users(user_ids):
    pass
```

### 4. Избегайте аббревиатур

```python
# Плохо
def calc_avg_usr_rtng(usr_id):
    pass

# Хорошо
def calculate_average_user_rating(user_id):
    pass

# Исключения — общепринятые аббревиатуры
url = "https://example.com"
html = "<div>Content</div>"
api = get_api_client()
id = get_user_id()
```

## Примеры рефакторинга

### До рефакторинга:

```python
def p(l):
    t = 0
    for i in l:
        if i['s'] == 'a':
            t += i['p'] * i['q']
    return t
```

### После рефакторинга:

```python
def calculate_active_orders_total(orders):
    """Подсчитывает общую сумму активных заказов."""
    total = 0
    for order in orders:
        if order['status'] == 'active':
            total += order['price'] * order['quantity']
    return total
```

## Инструменты для проверки именования

- **Linters**: pylint, flake8 (Python), ESLint (JavaScript)
- **IDE подсказки**: PyCharm, VS Code с расширениями
- **Code review**: командные практики

## Заключение

Хорошее именование — это инвестиция в читаемость и поддерживаемость кода. Потратив время на выбор правильного имени сейчас, вы сэкономите часы на понимание кода в будущем. Помните: код пишется один раз, а читается многократно.
