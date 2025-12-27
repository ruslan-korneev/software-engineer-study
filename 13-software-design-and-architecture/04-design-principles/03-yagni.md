# YAGNI — You Aren't Gonna Need It

## Определение

**YAGNI (You Aren't Gonna Need It)** — принцип экстремального программирования (XP), который гласит:

> "Не добавляйте функциональность, пока она действительно не понадобится."

Принцип противостоит практике "программирования на будущее" — созданию кода для требований, которые могут никогда не появиться.

---

## Суть принципа

### Почему "потом" часто не наступает

1. **Требования меняются** — то, что казалось нужным, становится неактуальным
2. **Приоритеты смещаются** — бизнес может пойти в другом направлении
3. **Технологии устаревают** — "заготовка" может стать бесполезной
4. **Контекст забывается** — через полгода никто не помнит, зачем это нужно

### Цена преждевременного кода

- **Время разработки** — на код, который не используется
- **Поддержка** — тесты, документация, обновления
- **Сложность** — лишний код усложняет систему
- **Технический долг** — устаревший код требует рефакторинга

---

## Примеры нарушения YAGNI

### 1. Преждевременная абстракция

**Плохо — абстракция "на будущее":**

```python
from abc import ABC, abstractmethod
from typing import List


class DataExporter(ABC):
    @abstractmethod
    def export(self, data: List[dict]) -> bytes:
        pass


class JSONExporter(DataExporter):
    def export(self, data: List[dict]) -> bytes:
        import json
        return json.dumps(data).encode()


class XMLExporter(DataExporter):
    """Может понадобиться в будущем..."""
    def export(self, data: List[dict]) -> bytes:
        # 50 строк кода для XML, который никогда не используется
        pass


class CSVExporter(DataExporter):
    """Вдруг попросят CSV..."""
    def export(self, data: List[dict]) -> bytes:
        # Ещё 30 строк неиспользуемого кода
        pass


class ExporterFactory:
    """Фабрика для всех экспортеров"""
    @staticmethod
    def create(format_type: str) -> DataExporter:
        exporters = {
            "json": JSONExporter,
            "xml": XMLExporter,
            "csv": CSVExporter,
        }
        return exporters[format_type]()
```

**Хорошо — только то, что нужно сейчас:**

```python
import json
from typing import List


def export_to_json(data: List[dict]) -> bytes:
    """Простая функция для текущей задачи"""
    return json.dumps(data).encode()


# Когда понадобится XML или CSV — тогда и добавим
```

---

### 2. Избыточные параметры конфигурации

**Плохо:**

```python
class EmailSender:
    def __init__(
        self,
        smtp_host: str,
        smtp_port: int,
        use_ssl: bool = True,
        use_tls: bool = False,
        timeout: int = 30,
        max_retries: int = 3,
        retry_delay: float = 1.0,
        connection_pool_size: int = 10,  # Никогда не использовалось
        custom_headers: dict = None,      # Никогда не использовалось
        proxy_host: str = None,           # Никогда не использовалось
        proxy_port: int = None,           # Никогда не использовалось
        debug_mode: bool = False,         # Никогда не использовалось
        log_level: str = "INFO",          # Никогда не использовалось
    ):
        # Инициализация всех параметров...
        pass
```

**Хорошо:**

```python
class EmailSender:
    def __init__(
        self,
        smtp_host: str,
        smtp_port: int = 587,
        use_ssl: bool = True,
    ):
        """Минимум параметров для рабочего функционала"""
        self.smtp_host = smtp_host
        self.smtp_port = smtp_port
        self.use_ssl = use_ssl
```

---

### 3. Неиспользуемые методы класса

**Плохо:**

```python
class UserRepository:
    def __init__(self, db):
        self.db = db

    def find_by_id(self, user_id: int):
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")

    def find_by_email(self, email: str):
        return self.db.query(f"SELECT * FROM users WHERE email = '{email}'")

    # "На будущее" — но никто не использует
    def find_by_phone(self, phone: str):
        return self.db.query(f"SELECT * FROM users WHERE phone = '{phone}'")

    def find_by_username(self, username: str):
        return self.db.query(f"SELECT * FROM users WHERE username = '{username}'")

    def find_by_country(self, country: str):
        return self.db.query(f"SELECT * FROM users WHERE country = '{country}'")

    def find_active_users(self):
        return self.db.query("SELECT * FROM users WHERE is_active = true")

    def find_inactive_users(self):
        return self.db.query("SELECT * FROM users WHERE is_active = false")
```

**Хорошо:**

```python
class UserRepository:
    def __init__(self, db):
        self.db = db

    def find_by_id(self, user_id: int):
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")

    def find_by_email(self, email: str):
        return self.db.query(f"SELECT * FROM users WHERE email = '{email}'")

    # Добавим другие методы, когда они понадобятся
```

---

### 4. Избыточные модели данных

**Плохо:**

```python
class User:
    def __init__(
        self,
        id: int,
        email: str,
        name: str,
        # Поля "на будущее"
        middle_name: str = None,           # Не используется
        nickname: str = None,              # Не используется
        avatar_url: str = None,            # Не используется
        bio: str = None,                   # Не используется
        website: str = None,               # Не используется
        twitter_handle: str = None,        # Не используется
        facebook_url: str = None,          # Не используется
        linkedin_url: str = None,          # Не используется
        github_username: str = None,       # Не используется
        preferred_language: str = "en",    # Не используется
        timezone: str = "UTC",             # Не используется
        theme: str = "light",              # Не используется
    ):
        pass
```

**Хорошо:**

```python
class User:
    def __init__(
        self,
        id: int,
        email: str,
        name: str,
    ):
        self.id = id
        self.email = email
        self.name = name
```

---

## Как применять YAGNI правильно

### 1. Задавайте вопросы

Перед добавлением функциональности спросите:
- Есть ли требование от пользователя/бизнеса?
- Используется ли это в текущем спринте?
- Что произойдёт, если не добавить это сейчас?

### 2. Откладывайте решения

```python
# Вместо создания сложной системы плагинов "на будущее"
# Просто добавьте TODO и двигайтесь дальше

def process_payment(amount: float) -> bool:
    # TODO: Если понадобится поддержка нескольких платёжных систем,
    # рассмотреть паттерн Strategy
    return stripe.charge(amount)
```

### 3. Рефакторьте, когда нужно

```python
# Итерация 1: Простое решение
def send_notification(user_id: int, message: str):
    email = get_user_email(user_id)
    send_email(email, message)


# Итерация 2: Когда действительно понадобилось SMS
def send_notification(user_id: int, message: str, channel: str = "email"):
    if channel == "email":
        email = get_user_email(user_id)
        send_email(email, message)
    elif channel == "sms":
        phone = get_user_phone(user_id)
        send_sms(phone, message)


# Итерация 3: Когда каналов стало много — тогда абстракция
class NotificationChannel(ABC):
    @abstractmethod
    def send(self, user_id: int, message: str): pass


class EmailChannel(NotificationChannel):
    def send(self, user_id: int, message: str):
        email = get_user_email(user_id)
        send_email(email, message)
```

---

## YAGNI и другие принципы

### YAGNI vs DRY

```python
# Иногда лучше немного дублировать, чем создавать абстракцию
# которая "может понадобиться"

# Если код дублируется 2 раза — возможно, это нормально
# Если 3+ раза — пора рефакторить
```

### YAGNI vs SOLID (Open/Closed)

```python
# Open/Closed говорит: "код должен быть открыт для расширения"
# YAGNI говорит: "не создавай точки расширения заранее"

# Баланс: создавайте расширяемую архитектуру,
# когда есть реальная необходимость в расширении
```

---

## Best Practices

1. **Работайте итеративно** — начинайте с простого решения
2. **Следуйте TDD** — тесты показывают, что реально нужно
3. **Получайте быструю обратную связь** — частые релизы помогают понять реальные потребности
4. **Документируйте отложенные решения** — TODO с контекстом
5. **Удаляйте неиспользуемый код** — не храните "на всякий случай"

## Типичные ошибки

1. **"А что, если..."** — синдром преждевременной оптимизации
2. **Золотое покрытие (Gold Plating)** — добавление фич, которые никто не просил
3. **Копирование архитектуры** — "в крупных проектах так делают"
4. **Страх рефакторинга** — боязнь изменить код позже

---

## Когда YAGNI не применим

1. **Безопасность** — защиту нужно закладывать сразу
2. **Архитектурные решения** — некоторые решения дорого менять
3. **Публичные API** — обратная совместимость важна
4. **Регуляторные требования** — compliance нельзя откладывать

---

## Резюме

> "Simplicity—the art of maximizing the amount of work not done—is essential."
> — Agile Manifesto

YAGNI учит нас:
- Решать текущие проблемы, а не воображаемые
- Доверять своей способности изменить код позже
- Ценить простоту выше "подготовленности к будущему"
