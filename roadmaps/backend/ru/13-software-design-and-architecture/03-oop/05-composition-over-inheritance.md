# Composition over Inheritance

## Композиция вместо наследования

"Предпочитайте композицию наследованию" — один из ключевых принципов объектно-ориентированного проектирования. Это не означает, что наследование плохо, но композиция часто является более гибким решением.

---

## Наследование: когда оно уместно

### Отношение "является" (is-a)

Наследование хорошо работает, когда есть чёткое отношение "является":

```python
from abc import ABC, abstractmethod


class Animal(ABC):
    @abstractmethod
    def speak(self) -> str:
        pass


class Dog(Animal):
    """Собака ЯВЛЯЕТСЯ животным."""

    def speak(self) -> str:
        return "Гав!"


class Cat(Animal):
    """Кошка ЯВЛЯЕТСЯ животным."""

    def speak(self) -> str:
        return "Мяу!"
```

### Проблемы наследования

1. **Жёсткая связь**: подкласс тесно связан с родителем
2. **Хрупкий базовый класс**: изменения в родителе могут сломать потомков
3. **Нарушение инкапсуляции**: наследник знает о деталях реализации
4. **Взрыв классов**: комбинирование функциональности требует много подклассов

---

## Пример проблемы: взрыв классов

```python
# Плохой пример: наследование для комбинирования функций

class Report:
    def generate(self) -> str:
        return "Базовый отчёт"


class PDFReport(Report):
    def generate(self) -> str:
        return f"[PDF] {super().generate()}"


class EncryptedReport(Report):
    def generate(self) -> str:
        return f"[ENCRYPTED] {super().generate()}"


class CompressedReport(Report):
    def generate(self) -> str:
        return f"[COMPRESSED] {super().generate()}"


# Проблема: как создать PDF + Encrypted отчёт?
# class PDFEncryptedReport(PDFReport, EncryptedReport)?  # Множественное наследование — сложно
# class EncryptedPDFReport(EncryptedReport, PDFReport)?  # Порядок важен!

# А если добавить ещё форматы?
# PDF + Encrypted + Compressed
# PDF + Compressed
# Encrypted + Compressed
# ... количество классов растёт экспоненциально!
```

---

## Решение: композиция

### Отношение "имеет" (has-a)

```python
from abc import ABC, abstractmethod
from typing import Protocol


# Интерфейсы для компонентов
class Formatter(Protocol):
    def format(self, content: str) -> str:
        ...


class Encryptor(Protocol):
    def encrypt(self, content: str) -> str:
        ...


class Compressor(Protocol):
    def compress(self, content: str) -> str:
        ...


# Конкретные реализации — легко добавлять новые
class PDFFormatter:
    def format(self, content: str) -> str:
        return f"[PDF] {content}"


class HTMLFormatter:
    def format(self, content: str) -> str:
        return f"<html><body>{content}</body></html>"


class PlainFormatter:
    def format(self, content: str) -> str:
        return content


class AESEncryptor:
    def encrypt(self, content: str) -> str:
        return f"[AES-ENCRYPTED] {content}"


class NoEncryption:
    def encrypt(self, content: str) -> str:
        return content


class GzipCompressor:
    def compress(self, content: str) -> str:
        return f"[GZIP] {content}"


class NoCompression:
    def compress(self, content: str) -> str:
        return content


# Основной класс использует композицию
class Report:
    def __init__(
        self,
        formatter: Formatter,
        encryptor: Encryptor,
        compressor: Compressor
    ):
        self.formatter = formatter
        self.encryptor = encryptor
        self.compressor = compressor

    def generate(self, content: str) -> str:
        result = self.formatter.format(content)
        result = self.encryptor.encrypt(result)
        result = self.compressor.compress(result)
        return result


# Использование: любые комбинации без новых классов!
pdf_encrypted = Report(PDFFormatter(), AESEncryptor(), NoCompression())
html_compressed = Report(HTMLFormatter(), NoEncryption(), GzipCompressor())
full_featured = Report(PDFFormatter(), AESEncryptor(), GzipCompressor())

print(pdf_encrypted.generate("Данные"))
# [AES-ENCRYPTED] [PDF] Данные

print(full_featured.generate("Данные"))
# [GZIP] [AES-ENCRYPTED] [PDF] Данные
```

---

## Паттерн: Стратегия (Strategy)

Позволяет менять алгоритмы во время выполнения.

```python
from abc import ABC, abstractmethod
from decimal import Decimal


class ShippingStrategy(ABC):
    @abstractmethod
    def calculate(self, weight: float, distance: float) -> Decimal:
        pass


class StandardShipping(ShippingStrategy):
    def calculate(self, weight: float, distance: float) -> Decimal:
        return Decimal(str(weight * 0.5 + distance * 0.1))


class ExpressShipping(ShippingStrategy):
    def calculate(self, weight: float, distance: float) -> Decimal:
        return Decimal(str(weight * 1.0 + distance * 0.3))


class FreeShipping(ShippingStrategy):
    def calculate(self, weight: float, distance: float) -> Decimal:
        return Decimal("0")


class Order:
    def __init__(self, items: list[dict]):
        self.items = items
        self._shipping_strategy: ShippingStrategy = StandardShipping()

    @property
    def total_weight(self) -> float:
        return sum(item["weight"] for item in self.items)

    def set_shipping(self, strategy: ShippingStrategy) -> None:
        """Стратегию можно менять в любой момент."""
        self._shipping_strategy = strategy

    def calculate_shipping(self, distance: float) -> Decimal:
        return self._shipping_strategy.calculate(self.total_weight, distance)


# Использование
order = Order([{"name": "Ноутбук", "weight": 2.5}])

order.set_shipping(StandardShipping())
print(f"Стандартная доставка: {order.calculate_shipping(100)}")

order.set_shipping(ExpressShipping())
print(f"Экспресс доставка: {order.calculate_shipping(100)}")
```

---

## Паттерн: Декоратор (Decorator)

Добавляет функциональность без наследования.

```python
from abc import ABC, abstractmethod
from typing import Protocol


class DataSource(Protocol):
    def write(self, data: str) -> None:
        ...

    def read(self) -> str:
        ...


class FileDataSource:
    def __init__(self, filename: str):
        self.filename = filename
        self._data = ""

    def write(self, data: str) -> None:
        self._data = data
        print(f"Записано в {self.filename}: {data}")

    def read(self) -> str:
        return self._data


class DataSourceDecorator(ABC):
    """Базовый декоратор."""

    def __init__(self, source: DataSource):
        self._wrapped = source

    def write(self, data: str) -> None:
        self._wrapped.write(data)

    def read(self) -> str:
        return self._wrapped.read()


class EncryptionDecorator(DataSourceDecorator):
    def write(self, data: str) -> None:
        encrypted = f"[ENCRYPTED]{data}[/ENCRYPTED]"
        super().write(encrypted)

    def read(self) -> str:
        data = super().read()
        # Убираем теги шифрования
        return data.replace("[ENCRYPTED]", "").replace("[/ENCRYPTED]", "")


class CompressionDecorator(DataSourceDecorator):
    def write(self, data: str) -> None:
        compressed = f"[COMPRESSED]{data}[/COMPRESSED]"
        super().write(compressed)

    def read(self) -> str:
        data = super().read()
        return data.replace("[COMPRESSED]", "").replace("[/COMPRESSED]", "")


# Использование: декораторы можно комбинировать!
source = FileDataSource("data.txt")
encrypted = EncryptionDecorator(source)
compressed_and_encrypted = CompressionDecorator(EncryptionDecorator(source))

# Простая запись
source.write("Hello")

# Зашифрованная запись
encrypted.write("Secret")

# Сжатая и зашифрованная
compressed_and_encrypted.write("Important data")
```

---

## Делегирование vs Наследование

### Делегирование

```python
class Engine:
    def start(self) -> str:
        return "Двигатель запущен"

    def stop(self) -> str:
        return "Двигатель остановлен"


class Wheels:
    def rotate(self) -> str:
        return "Колёса вращаются"


class Car:
    """Машина ИМЕЕТ двигатель и колёса (композиция)."""

    def __init__(self):
        self._engine = Engine()
        self._wheels = Wheels()

    def start(self) -> str:
        """Делегирование к Engine."""
        return self._engine.start()

    def drive(self) -> str:
        engine_status = self._engine.start()
        wheels_status = self._wheels.rotate()
        return f"{engine_status}, {wheels_status}"


car = Car()
print(car.drive())
# Двигатель запущен, Колёса вращаются
```

### Плохой пример: наследование вместо композиции

```python
# ПЛОХО: Stack не ЯВЛЯЕТСЯ ArrayList
class BadStack(list):
    """Плохо: наследуем от list."""

    def push(self, item):
        self.append(item)

    def pop_item(self):
        return self.pop()


stack = BadStack()
stack.push(1)
stack.push(2)
# Проблема: унаследованы ненужные методы
stack.insert(0, 999)  # Нарушает семантику стека!
stack[0] = 888        # Тоже нарушает!


# ХОРОШО: композиция
class GoodStack:
    """Хорошо: используем list как внутреннюю реализацию."""

    def __init__(self):
        self._items: list = []

    def push(self, item) -> None:
        self._items.append(item)

    def pop(self):
        if not self._items:
            raise IndexError("Stack is empty")
        return self._items.pop()

    def peek(self):
        if not self._items:
            raise IndexError("Stack is empty")
        return self._items[-1]

    def is_empty(self) -> bool:
        return len(self._items) == 0

    def __len__(self) -> int:
        return len(self._items)


good_stack = GoodStack()
good_stack.push(1)
good_stack.push(2)
# good_stack.insert(0, 999)  # AttributeError: нет такого метода
```

---

## Mixin'ы: компромисс между наследованием и композицией

Mixin'ы — это классы, предоставляющие дополнительную функциональность через множественное наследование.

```python
import json
from datetime import datetime


class JsonSerializableMixin:
    """Mixin для JSON-сериализации."""

    def to_json(self) -> str:
        return json.dumps(self.__dict__, default=str)

    @classmethod
    def from_json(cls, json_str: str):
        data = json.loads(json_str)
        return cls(**data)


class TimestampMixin:
    """Mixin для отслеживания времени создания/обновления."""

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.created_at = datetime.now()
        self.updated_at = datetime.now()

    def touch(self) -> None:
        """Обновить время последнего изменения."""
        self.updated_at = datetime.now()


class LoggableMixin:
    """Mixin для логирования."""

    def log(self, message: str) -> None:
        class_name = self.__class__.__name__
        print(f"[{class_name}] {message}")


# Использование нескольких mixin'ов
class User(JsonSerializableMixin, TimestampMixin, LoggableMixin):
    def __init__(self, name: str, email: str):
        self.name = name
        self.email = email
        super().__init__()  # Вызывает TimestampMixin.__init__

    def update_email(self, new_email: str) -> None:
        self.log(f"Изменение email с {self.email} на {new_email}")
        self.email = new_email
        self.touch()


user = User("Иван", "ivan@example.com")
print(user.to_json())
user.update_email("ivan.new@example.com")
```

---

## Внедрение зависимостей (Dependency Injection)

Композиция часто идёт вместе с внедрением зависимостей.

```python
from abc import ABC, abstractmethod
from typing import Protocol


# Интерфейсы
class Logger(Protocol):
    def log(self, message: str) -> None:
        ...


class Database(Protocol):
    def query(self, sql: str) -> list:
        ...


class Cache(Protocol):
    def get(self, key: str) -> str | None:
        ...

    def set(self, key: str, value: str) -> None:
        ...


# Реализации
class ConsoleLogger:
    def log(self, message: str) -> None:
        print(f"[LOG] {message}")


class PostgresDatabase:
    def query(self, sql: str) -> list:
        print(f"PostgreSQL: {sql}")
        return []


class RedisCache:
    def __init__(self):
        self._data: dict = {}

    def get(self, key: str) -> str | None:
        return self._data.get(key)

    def set(self, key: str, value: str) -> None:
        self._data[key] = value


# Сервис с внедрёнными зависимостями
class UserService:
    def __init__(
        self,
        database: Database,
        cache: Cache,
        logger: Logger
    ):
        self.db = database
        self.cache = cache
        self.logger = logger

    def get_user(self, user_id: int) -> dict | None:
        self.logger.log(f"Получение пользователя {user_id}")

        # Сначала проверяем кэш
        cached = self.cache.get(f"user:{user_id}")
        if cached:
            self.logger.log("Найдено в кэше")
            return {"id": user_id, "data": cached}

        # Запрос к базе
        result = self.db.query(f"SELECT * FROM users WHERE id = {user_id}")
        return result[0] if result else None


# Сборка приложения
logger = ConsoleLogger()
database = PostgresDatabase()
cache = RedisCache()

user_service = UserService(database, cache, logger)
user_service.get_user(123)


# Для тестов — можно подставить моки
class MockDatabase:
    def query(self, sql: str) -> list:
        return [{"id": 1, "name": "Test User"}]


class MockCache:
    def get(self, key: str) -> str | None:
        return None

    def set(self, key: str, value: str) -> None:
        pass


class MockLogger:
    def log(self, message: str) -> None:
        pass  # Ничего не делаем в тестах


# Тестовый экземпляр
test_service = UserService(MockDatabase(), MockCache(), MockLogger())
```

---

## Когда использовать наследование

Наследование уместно когда:

1. **Есть чёткое отношение "является"**
   - `Dog` является `Animal`
   - `SavingsAccount` является `BankAccount`

2. **Нужно переиспользовать реализацию, а не только интерфейс**
   - Общая логика в базовом классе

3. **Иерархия стабильна и не будет часто меняться**

4. **Используете Template Method паттерн**

```python
from abc import ABC, abstractmethod


class DataParser(ABC):
    """Template Method: общий алгоритм, детали в подклассах."""

    def parse(self, raw_data: str) -> dict:
        """Шаблонный метод — определяет алгоритм."""
        data = self._read_data(raw_data)
        data = self._validate(data)
        data = self._transform(data)
        return data

    @abstractmethod
    def _read_data(self, raw: str) -> dict:
        """Подклассы определяют, как читать данные."""
        pass

    def _validate(self, data: dict) -> dict:
        """Общая валидация — можно переопределить."""
        if not data:
            raise ValueError("Empty data")
        return data

    @abstractmethod
    def _transform(self, data: dict) -> dict:
        """Подклассы определяют трансформацию."""
        pass


class JsonParser(DataParser):
    def _read_data(self, raw: str) -> dict:
        import json
        return json.loads(raw)

    def _transform(self, data: dict) -> dict:
        return {k.lower(): v for k, v in data.items()}


class XmlParser(DataParser):
    def _read_data(self, raw: str) -> dict:
        # Упрощённо
        return {"xml": raw}

    def _transform(self, data: dict) -> dict:
        return data
```

---

## Best Practices

1. **Начинайте с композиции**, переходите к наследованию только при явной необходимости

2. **Используйте интерфейсы** (Protocol/ABC) для определения контрактов

3. **Внедряйте зависимости** вместо создания внутри класса

4. **Предпочитайте делегирование** прямому наследованию

5. **Ограничивайте глубину наследования** (2-3 уровня максимум)

---

## Типичные ошибки

1. **Наследование ради переиспользования кода**
   ```python
   # Плохо
   class Button(Rectangle):  # Кнопка НЕ является прямоугольником
       pass

   # Хорошо
   class Button:
       def __init__(self):
           self._shape = Rectangle()  # Кнопка ИМЕЕТ форму
   ```

2. **Глубокие иерархии**
   ```python
   # Плохо: 5+ уровней наследования
   class A: pass
   class B(A): pass
   class C(B): pass
   class D(C): pass
   class E(D): pass  # Кошмар для поддержки
   ```

3. **Нарушение LSP (Liskov Substitution Principle)**
   ```python
   # Плохо: квадрат нарушает контракт прямоугольника
   class Rectangle:
       def set_width(self, w): self._w = w
       def set_height(self, h): self._h = h

   class Square(Rectangle):
       def set_width(self, w):
           self._w = self._h = w  # Меняет и высоту — неожиданно!
   ```

4. **Жёсткие зависимости внутри класса**
   ```python
   # Плохо
   class UserService:
       def __init__(self):
           self.db = PostgresDatabase()  # Жёсткая зависимость

   # Хорошо
   class UserService:
       def __init__(self, db: Database):
           self.db = db  # Внедрение зависимости
   ```
