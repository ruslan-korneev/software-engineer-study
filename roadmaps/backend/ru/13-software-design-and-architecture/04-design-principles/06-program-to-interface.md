# Program to Interface, Not Implementation

## Определение

**"Program to an interface, not an implementation"** (Программируй на уровне интерфейса, а не реализации) — один из ключевых принципов объектно-ориентированного проектирования из книги "Design Patterns" (Gang of Four).

> "Клиенты должны зависеть от абстракций (интерфейсов), а не от конкретных реализаций."

---

## Что такое "интерфейс" в этом контексте

### Широкое понимание

"Интерфейс" здесь означает не только `interface` из Java или `Protocol` из Python, но и:
- Абстрактные классы
- Публичные методы класса
- Контракт (сигнатура методов и ожидаемое поведение)

### Ключевая идея

Код должен зависеть от **"что делает"**, а не от **"как делает"**.

---

## Почему это важно

### Проблемы зависимости от реализации

1. **Жёсткая связность** — изменение одного класса ломает другие
2. **Сложность тестирования** — нельзя подменить зависимость
3. **Невозможность расширения** — добавление варианта требует изменений
4. **Повторное использование затруднено**

### Преимущества интерфейсов

- **Слабая связность** — компоненты независимы
- **Гибкость** — легко заменить реализацию
- **Тестируемость** — можно использовать моки
- **Расширяемость** — новые реализации без изменения клиента

---

## Примеры

### 1. Зависимость от реализации (плохо)

```python
class MySQLDatabase:
    def __init__(self, host: str, port: int):
        self.connection = mysql.connect(host=host, port=port)

    def execute_query(self, sql: str):
        return self.connection.execute(sql)

    def fetch_all(self):
        return self.connection.fetchall()


class UserService:
    def __init__(self):
        # Жёсткая зависимость от MySQL
        self.db = MySQLDatabase("localhost", 3306)

    def get_users(self):
        self.db.execute_query("SELECT * FROM users")
        return self.db.fetch_all()


# Проблемы:
# - Нельзя использовать PostgreSQL без изменения UserService
# - Нельзя подставить mock для тестирования
# - UserService знает детали подключения к БД
```

### 2. Программирование на интерфейс (хорошо)

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any


class Database(ABC):
    """Интерфейс для работы с БД"""

    @abstractmethod
    def execute(self, query: str, params: tuple = None) -> None:
        pass

    @abstractmethod
    def fetch_all(self) -> List[Dict[str, Any]]:
        pass

    @abstractmethod
    def fetch_one(self) -> Dict[str, Any]:
        pass


class MySQLDatabase(Database):
    """Реализация для MySQL"""

    def __init__(self, host: str, port: int, user: str, password: str):
        self.connection = mysql.connect(
            host=host, port=port, user=user, password=password
        )

    def execute(self, query: str, params: tuple = None) -> None:
        self.connection.execute(query, params)

    def fetch_all(self) -> List[Dict[str, Any]]:
        return self.connection.fetchall()

    def fetch_one(self) -> Dict[str, Any]:
        return self.connection.fetchone()


class PostgreSQLDatabase(Database):
    """Реализация для PostgreSQL"""

    def __init__(self, dsn: str):
        self.connection = psycopg2.connect(dsn)

    def execute(self, query: str, params: tuple = None) -> None:
        with self.connection.cursor() as cursor:
            cursor.execute(query, params)

    def fetch_all(self) -> List[Dict[str, Any]]:
        with self.connection.cursor() as cursor:
            return cursor.fetchall()

    def fetch_one(self) -> Dict[str, Any]:
        with self.connection.cursor() as cursor:
            return cursor.fetchone()


class UserService:
    """Сервис зависит от интерфейса, не от реализации"""

    def __init__(self, database: Database):  # Принимает интерфейс
        self.db = database

    def get_users(self) -> List[Dict[str, Any]]:
        self.db.execute("SELECT * FROM users")
        return self.db.fetch_all()


# Использование
mysql_db = MySQLDatabase("localhost", 3306, "user", "pass")
postgres_db = PostgreSQLDatabase("postgresql://localhost/mydb")

# Один и тот же сервис с разными БД
service_mysql = UserService(mysql_db)
service_postgres = UserService(postgres_db)
```

---

### 3. Тестирование с моками

```python
class MockDatabase(Database):
    """Мок для тестирования"""

    def __init__(self, data: List[Dict[str, Any]]):
        self.data = data
        self.last_query = None

    def execute(self, query: str, params: tuple = None) -> None:
        self.last_query = query

    def fetch_all(self) -> List[Dict[str, Any]]:
        return self.data

    def fetch_one(self) -> Dict[str, Any]:
        return self.data[0] if self.data else None


# Тест
def test_get_users():
    mock_data = [
        {"id": 1, "name": "Alice"},
        {"id": 2, "name": "Bob"},
    ]
    mock_db = MockDatabase(mock_data)
    service = UserService(mock_db)

    users = service.get_users()

    assert len(users) == 2
    assert users[0]["name"] == "Alice"
    assert mock_db.last_query == "SELECT * FROM users"
```

---

### 4. Пример с логированием

**Плохо — зависимость от конкретного логгера:**

```python
import logging


class OrderProcessor:
    def __init__(self):
        # Жёсткая привязка к стандартному логгеру
        self.logger = logging.getLogger(__name__)

    def process(self, order):
        self.logger.info(f"Processing order {order.id}")
        # обработка...
        self.logger.info(f"Order {order.id} processed")
```

**Хорошо — интерфейс логирования:**

```python
from abc import ABC, abstractmethod


class Logger(ABC):
    @abstractmethod
    def info(self, message: str) -> None:
        pass

    @abstractmethod
    def error(self, message: str, exc: Exception = None) -> None:
        pass

    @abstractmethod
    def debug(self, message: str) -> None:
        pass


class StandardLogger(Logger):
    def __init__(self, name: str):
        self._logger = logging.getLogger(name)

    def info(self, message: str) -> None:
        self._logger.info(message)

    def error(self, message: str, exc: Exception = None) -> None:
        self._logger.error(message, exc_info=exc)

    def debug(self, message: str) -> None:
        self._logger.debug(message)


class CloudLogger(Logger):
    """Логирование в облачный сервис"""

    def __init__(self, api_key: str):
        self.client = CloudLoggingClient(api_key)

    def info(self, message: str) -> None:
        self.client.send(level="INFO", message=message)

    def error(self, message: str, exc: Exception = None) -> None:
        self.client.send(level="ERROR", message=message, exception=str(exc))

    def debug(self, message: str) -> None:
        self.client.send(level="DEBUG", message=message)


class NullLogger(Logger):
    """Логгер, который ничего не делает (для тестов)"""

    def info(self, message: str) -> None:
        pass

    def error(self, message: str, exc: Exception = None) -> None:
        pass

    def debug(self, message: str) -> None:
        pass


class OrderProcessor:
    def __init__(self, logger: Logger):
        self.logger = logger

    def process(self, order):
        self.logger.info(f"Processing order {order.id}")
        # обработка...
        self.logger.info(f"Order {order.id} processed")


# Разные окружения — разные логгеры
processor_dev = OrderProcessor(StandardLogger("orders"))
processor_prod = OrderProcessor(CloudLogger("api-key"))
processor_test = OrderProcessor(NullLogger())
```

---

### 5. Пример с отправкой уведомлений

```python
from abc import ABC, abstractmethod
from typing import List


class NotificationSender(ABC):
    """Интерфейс отправки уведомлений"""

    @abstractmethod
    def send(self, recipient: str, subject: str, body: str) -> bool:
        pass


class EmailSender(NotificationSender):
    def __init__(self, smtp_host: str, smtp_port: int):
        self.smtp = SMTPClient(smtp_host, smtp_port)

    def send(self, recipient: str, subject: str, body: str) -> bool:
        return self.smtp.send_email(
            to=recipient,
            subject=subject,
            body=body
        )


class SMSSender(NotificationSender):
    def __init__(self, api_key: str):
        self.client = TwilioClient(api_key)

    def send(self, recipient: str, subject: str, body: str) -> bool:
        # SMS не поддерживает subject, используем только body
        return self.client.send_sms(to=recipient, message=body)


class PushNotificationSender(NotificationSender):
    def __init__(self, firebase_config: dict):
        self.firebase = Firebase(firebase_config)

    def send(self, recipient: str, subject: str, body: str) -> bool:
        return self.firebase.send_push(
            token=recipient,
            title=subject,
            body=body
        )


class NotificationService:
    """Сервис работает с любым отправителем"""

    def __init__(self, senders: List[NotificationSender]):
        self.senders = senders

    def notify(self, recipient: str, subject: str, body: str) -> None:
        for sender in self.senders:
            try:
                sender.send(recipient, subject, body)
            except Exception as e:
                # Логирование ошибки, продолжаем с другими
                pass


# Гибкая конфигурация каналов
service = NotificationService([
    EmailSender("smtp.gmail.com", 587),
    PushNotificationSender(firebase_config),
])
```

---

## Python-специфика

### Использование Protocol (Python 3.8+)

```python
from typing import Protocol, List, Dict, Any


class Repository(Protocol):
    """Структурная типизация — не требует явного наследования"""

    def find_by_id(self, id: int) -> Dict[str, Any]:
        ...

    def save(self, entity: Dict[str, Any]) -> int:
        ...

    def delete(self, id: int) -> bool:
        ...


class SQLiteRepository:
    """Не наследует Repository, но реализует его контракт"""

    def __init__(self, db_path: str):
        self.conn = sqlite3.connect(db_path)

    def find_by_id(self, id: int) -> Dict[str, Any]:
        cursor = self.conn.execute("SELECT * FROM items WHERE id = ?", (id,))
        return dict(cursor.fetchone())

    def save(self, entity: Dict[str, Any]) -> int:
        # реализация
        pass

    def delete(self, id: int) -> bool:
        # реализация
        pass


class ItemService:
    def __init__(self, repo: Repository):  # Принимает Protocol
        self.repo = repo


# SQLiteRepository соответствует Protocol — работает!
service = ItemService(SQLiteRepository("data.db"))
```

### Duck Typing

```python
# В Python часто достаточно duck typing
class FileStorage:
    def save(self, key: str, data: bytes) -> None:
        with open(f"/tmp/{key}", "wb") as f:
            f.write(data)


class S3Storage:
    def save(self, key: str, data: bytes) -> None:
        self.client.put_object(Bucket="bucket", Key=key, Body=data)


class DataService:
    def __init__(self, storage):  # Без указания типа
        self.storage = storage

    def store(self, key: str, data: bytes):
        self.storage.save(key, data)  # Duck typing: если есть save — работает


# Оба класса подходят
service1 = DataService(FileStorage())
service2 = DataService(S3Storage())
```

---

## Best Practices

1. **Определяйте интерфейсы от потребителя** — какие методы нужны клиенту?
2. **Маленькие интерфейсы** — следуйте Interface Segregation Principle
3. **Инжектируйте зависимости** — не создавайте внутри класса
4. **Используйте фабрики** для создания конкретных реализаций
5. **Документируйте контракты** — что должна делать реализация

## Типичные ошибки

1. **Интерфейс повторяет реализацию** — слишком специфичный
2. **Слишком много интерфейсов** — каждый класс за интерфейсом
3. **Игнорирование duck typing** — в Python не всегда нужен ABC
4. **Leaky abstractions** — детали реализации просачиваются в интерфейс

---

## Связь с другими принципами

| Принцип | Связь |
|---------|-------|
| SOLID (DIP) | Dependency Inversion — основан на этом принципе |
| SOLID (ISP) | Interface Segregation — как проектировать интерфейсы |
| Encapsulate What Varies | Интерфейс — способ инкапсуляции |
| Loose Coupling | Интерфейсы обеспечивают слабую связность |

---

## Резюме

Программирование на интерфейс означает:
- Зависеть от абстракций, не от реализаций
- Проектировать код для расширяемости
- Обеспечивать тестируемость через подмену зависимостей
- Следовать принципу инверсии зависимостей

> "Depend upon abstractions. Do not depend upon concretions." — Uncle Bob
