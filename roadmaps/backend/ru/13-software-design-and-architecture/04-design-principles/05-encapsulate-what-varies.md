# Encapsulate What Varies — Инкапсулируй то, что меняется

## Определение

**"Encapsulate What Varies"** (Инкапсулируй то, что меняется) — один из фундаментальных принципов объектно-ориентированного проектирования. Он гласит:

> "Определите аспекты вашего приложения, которые изменяются, и отделите их от тех, которые остаются постоянными."

Этот принцип лежит в основе многих паттернов проектирования (Strategy, Factory, Decorator и др.).

---

## Суть принципа

### Что значит "инкапсулировать"

Инкапсуляция в данном контексте означает:
1. **Изолировать** изменяющийся код в отдельный модуль/класс
2. **Скрыть** детали реализации за интерфейсом
3. **Защитить** остальной код от изменений

### Почему это важно

- Изменения локализованы в одном месте
- Остальной код не затрагивается
- Проще добавлять новые варианты поведения
- Легче тестировать изолированные компоненты

---

## Как определить, что может измениться

### Признаки изменчивости

1. **Бизнес-правила** — скидки, налоги, условия
2. **Алгоритмы** — сортировка, поиск, валидация
3. **Внешние системы** — БД, API, сервисы
4. **Форматы данных** — JSON, XML, CSV
5. **UI/UX** — отображение, локализация
6. **Конфигурация** — настройки, параметры

### Вопросы для выявления

- Что может измениться в требованиях?
- Какие части системы зависят от внешних факторов?
- Где используются `if/else` или `switch` с типами?

---

## Примеры применения

### 1. Инкапсуляция алгоритма (Strategy Pattern)

**Проблема — изменяющийся алгоритм в коде:**

```python
class ShippingCalculator:
    def calculate_cost(self, order, shipping_type: str) -> float:
        if shipping_type == "standard":
            return order.weight * 1.5
        elif shipping_type == "express":
            return order.weight * 3.0 + 10.0
        elif shipping_type == "overnight":
            return order.weight * 5.0 + 25.0
        # Добавление нового типа требует изменения этого класса
        else:
            raise ValueError(f"Unknown shipping type: {shipping_type}")
```

**Решение — инкапсуляция в стратегии:**

```python
from abc import ABC, abstractmethod


class ShippingStrategy(ABC):
    """Интерфейс для всех стратегий доставки"""

    @abstractmethod
    def calculate(self, order) -> float:
        pass


class StandardShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 1.5


class ExpressShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 3.0 + 10.0


class OvernightShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 5.0 + 25.0


# Добавление нового типа — просто новый класс
class DroneShipping(ShippingStrategy):
    def calculate(self, order) -> float:
        return order.weight * 4.0 + 15.0


class ShippingCalculator:
    """Калькулятор не знает о деталях расчёта"""

    def __init__(self, strategy: ShippingStrategy):
        self.strategy = strategy

    def calculate_cost(self, order) -> float:
        return self.strategy.calculate(order)


# Использование
calculator = ShippingCalculator(ExpressShipping())
cost = calculator.calculate_cost(order)
```

---

### 2. Инкапсуляция создания объектов (Factory Pattern)

**Проблема — создание зависит от типа:**

```python
class NotificationService:
    def send(self, message: str, channel: str):
        if channel == "email":
            # 10 строк создания email клиента
            smtp = SMTP("smtp.gmail.com", 587)
            smtp.starttls()
            smtp.login("user", "pass")
            smtp.send(message)
        elif channel == "sms":
            # 10 строк создания SMS клиента
            twilio = TwilioClient("sid", "token")
            twilio.messages.create(body=message, to="+1234567890")
        elif channel == "push":
            # 10 строк создания Push клиента
            firebase = FirebaseApp.initialize()
            firebase.messaging().send(message)
```

**Решение — фабрика для создания:**

```python
from abc import ABC, abstractmethod


class NotificationChannel(ABC):
    @abstractmethod
    def send(self, message: str):
        pass


class EmailChannel(NotificationChannel):
    def __init__(self, smtp_config: dict):
        self.smtp = SMTP(smtp_config["host"], smtp_config["port"])
        # настройка...

    def send(self, message: str):
        self.smtp.send(message)


class SMSChannel(NotificationChannel):
    def __init__(self, twilio_config: dict):
        self.client = TwilioClient(twilio_config["sid"], twilio_config["token"])

    def send(self, message: str):
        self.client.messages.create(body=message)


class PushChannel(NotificationChannel):
    def __init__(self, firebase_config: dict):
        self.app = FirebaseApp.initialize(firebase_config)

    def send(self, message: str):
        self.app.messaging().send(message)


class NotificationChannelFactory:
    """Фабрика инкапсулирует логику создания"""

    _channels = {
        "email": EmailChannel,
        "sms": SMSChannel,
        "push": PushChannel,
    }

    @classmethod
    def create(cls, channel_type: str, config: dict) -> NotificationChannel:
        channel_class = cls._channels.get(channel_type)
        if not channel_class:
            raise ValueError(f"Unknown channel: {channel_type}")
        return channel_class(config)


class NotificationService:
    """Сервис не знает, как создаются каналы"""

    def __init__(self, factory: NotificationChannelFactory):
        self.factory = factory

    def send(self, message: str, channel_type: str, config: dict):
        channel = self.factory.create(channel_type, config)
        channel.send(message)
```

---

### 3. Инкапсуляция данных доступа (Repository Pattern)

**Проблема — код работы с БД разбросан:**

```python
class UserService:
    def get_user(self, user_id: int):
        # Прямой SQL в бизнес-логике
        result = db.execute(
            "SELECT * FROM users WHERE id = %s", (user_id,)
        )
        return User(**result.fetchone())

    def create_user(self, name: str, email: str):
        db.execute(
            "INSERT INTO users (name, email) VALUES (%s, %s)",
            (name, email)
        )
        # Что если мы перейдём на MongoDB?
```

**Решение — репозиторий скрывает детали:**

```python
from abc import ABC, abstractmethod
from typing import Optional


class UserRepository(ABC):
    """Абстрактный репозиторий — интерфейс"""

    @abstractmethod
    def find_by_id(self, user_id: int) -> Optional[User]:
        pass

    @abstractmethod
    def save(self, user: User) -> User:
        pass

    @abstractmethod
    def delete(self, user_id: int) -> bool:
        pass


class PostgresUserRepository(UserRepository):
    """Реализация для PostgreSQL"""

    def __init__(self, connection):
        self.conn = connection

    def find_by_id(self, user_id: int) -> Optional[User]:
        result = self.conn.execute(
            "SELECT * FROM users WHERE id = %s", (user_id,)
        )
        row = result.fetchone()
        return User(**row) if row else None

    def save(self, user: User) -> User:
        # SQL логика
        pass


class MongoUserRepository(UserRepository):
    """Реализация для MongoDB"""

    def __init__(self, collection):
        self.collection = collection

    def find_by_id(self, user_id: int) -> Optional[User]:
        doc = self.collection.find_one({"_id": user_id})
        return User(**doc) if doc else None

    def save(self, user: User) -> User:
        # MongoDB логика
        pass


class UserService:
    """Сервис не знает, какая БД используется"""

    def __init__(self, repository: UserRepository):
        self.repository = repository

    def get_user(self, user_id: int) -> Optional[User]:
        return self.repository.find_by_id(user_id)
```

---

### 4. Инкапсуляция правил валидации

**Проблема — правила встроены в код:**

```python
class OrderService:
    def create_order(self, order):
        # Правила валидации могут меняться
        if order.total < 0:
            raise ValueError("Total cannot be negative")
        if order.items_count == 0:
            raise ValueError("Order must have items")
        if order.total > 10000 and not order.user.is_verified:
            raise ValueError("Large orders require verification")
        if order.shipping_country in ["XX", "YY"]:
            raise ValueError("Shipping not available")

        # Создание заказа...
```

**Решение — правила в отдельных классах:**

```python
from abc import ABC, abstractmethod
from typing import List


class OrderValidationRule(ABC):
    @abstractmethod
    def validate(self, order) -> bool:
        pass

    @abstractmethod
    def get_error_message(self) -> str:
        pass


class PositiveTotalRule(OrderValidationRule):
    def validate(self, order) -> bool:
        return order.total >= 0

    def get_error_message(self) -> str:
        return "Total cannot be negative"


class NonEmptyOrderRule(OrderValidationRule):
    def validate(self, order) -> bool:
        return order.items_count > 0

    def get_error_message(self) -> str:
        return "Order must have items"


class LargeOrderVerificationRule(OrderValidationRule):
    def __init__(self, threshold: float = 10000):
        self.threshold = threshold

    def validate(self, order) -> bool:
        if order.total > self.threshold:
            return order.user.is_verified
        return True

    def get_error_message(self) -> str:
        return f"Orders over {self.threshold} require verification"


class OrderValidator:
    """Композиция правил валидации"""

    def __init__(self, rules: List[OrderValidationRule]):
        self.rules = rules

    def validate(self, order) -> List[str]:
        errors = []
        for rule in self.rules:
            if not rule.validate(order):
                errors.append(rule.get_error_message())
        return errors


class OrderService:
    def __init__(self, validator: OrderValidator):
        self.validator = validator

    def create_order(self, order):
        errors = self.validator.validate(order)
        if errors:
            raise ValueError(errors)
        # Создание заказа...
```

---

## Когда применять принцип

### Применяйте, когда:

1. Видите `if/else` цепочки по типу
2. Код часто меняется в определённом месте
3. Один класс знает слишком много о реализации других
4. Тестирование затруднено из-за жёстких зависимостей

### Не переусердствуйте, если:

1. Код действительно не будет меняться
2. Система маленькая и простая
3. Абстракция сложнее, чем условные операторы

---

## Best Practices

1. **Идентифицируйте точки изменений** заранее
2. **Используйте интерфейсы** для определения контрактов
3. **Применяйте Dependency Injection** для гибкости
4. **Следуйте Open/Closed Principle** — связанный принцип
5. **Тестируйте компоненты изолированно**

## Типичные ошибки

1. **Инкапсуляция всего** — создание абстракций без необходимости
2. **Слишком глубокие иерархии** — сложно понять код
3. **Преждевременная инкапсуляция** — до понимания, что реально меняется
4. **Игнорирование простых случаев** — иногда `if/else` достаточно

---

## Связь с паттернами проектирования

| Паттерн | Что инкапсулирует |
|---------|-------------------|
| Strategy | Алгоритм |
| Factory | Создание объектов |
| Decorator | Дополнительное поведение |
| Template Method | Шаги алгоритма |
| Observer | Реакция на изменения |
| State | Поведение в зависимости от состояния |

---

## Резюме

Принцип "Encapsulate What Varies" учит нас:
- Выделять изменяющиеся части в отдельные компоненты
- Защищать стабильный код от изменений
- Использовать абстракции для гибкости
- Думать о будущих изменениях при проектировании
