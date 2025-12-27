# GoF Design Patterns (Паттерны "Банды четырёх")

## Введение

**GoF (Gang of Four)** — это группа авторов книги "Design Patterns: Elements of Reusable Object-Oriented Software" (1994): Эрих Гамма, Ричард Хелм, Ральф Джонсон и Джон Влиссидес. Они систематизировали 23 паттерна проектирования, разделив их на три категории:

1. **Порождающие (Creational)** — отвечают за создание объектов
2. **Структурные (Structural)** — отвечают за композицию классов и объектов
3. **Поведенческие (Behavioral)** — отвечают за взаимодействие между объектами

---

## Порождающие паттерны (Creational Patterns)

Эти паттерны абстрагируют процесс инстанцирования объектов, делая систему независимой от способа создания, композиции и представления объектов.

### 1. Singleton (Одиночка)

**Назначение:** Гарантирует, что класс имеет только один экземпляр, и предоставляет глобальную точку доступа к нему.

**Когда использовать:**

- Когда нужен ровно один экземпляр класса (например, логгер, конфигурация, пул соединений)
- Когда нужен контролируемый доступ к единственному экземпляру

```python
# Python - потокобезопасный Singleton
import threading

class Singleton:
    _instance = None
    _lock = threading.Lock()

    def __new__(cls, *args, **kwargs):
        if cls._instance is None:
            with cls._lock:
                # Double-checked locking
                if cls._instance is None:
                    cls._instance = super().__new__(cls)
        return cls._instance

    def __init__(self):
        # Инициализация только один раз
        if not hasattr(self, '_initialized'):
            self._initialized = True
            self.config = {}

# Использование
s1 = Singleton()
s2 = Singleton()
print(s1 is s2)  # True
```

```go
// Go - Singleton с sync.Once
package main

import (
    "sync"
)

type singleton struct {
    config map[string]string
}

var (
    instance *singleton
    once     sync.Once
)

func GetInstance() *singleton {
    once.Do(func() {
        instance = &singleton{
            config: make(map[string]string),
        }
    })
    return instance
}
```

**Best Practices:**

- Используйте ленивую инициализацию (lazy initialization)
- Обеспечьте потокобезопасность в многопоточных приложениях
- Рассмотрите использование Dependency Injection вместо Singleton

**Типичные ошибки:**

- Глобальное состояние затрудняет тестирование
- Нарушение принципа единственной ответственности (SRP)
- Скрытые зависимости в коде

---

### 2. Factory Method (Фабричный метод)

**Назначение:** Определяет интерфейс для создания объекта, но позволяет подклассам решать, какой класс инстанцировать.

**Когда использовать:**

- Когда заранее неизвестны типы создаваемых объектов
- Когда нужно делегировать создание объектов подклассам
- Когда нужно централизовать логику создания объектов

```python
from abc import ABC, abstractmethod

# Абстрактный продукт
class Transport(ABC):
    @abstractmethod
    def deliver(self) -> str:
        pass

# Конкретные продукты
class Truck(Transport):
    def deliver(self) -> str:
        return "Доставка по земле в коробке"

class Ship(Transport):
    def deliver(self) -> str:
        return "Доставка по морю в контейнере"

class Plane(Transport):
    def deliver(self) -> str:
        return "Доставка по воздуху в грузовом отсеке"

# Абстрактный создатель
class Logistics(ABC):
    @abstractmethod
    def create_transport(self) -> Transport:
        """Factory Method"""
        pass

    def plan_delivery(self) -> str:
        transport = self.create_transport()
        return f"Логистика: {transport.deliver()}"

# Конкретные создатели
class RoadLogistics(Logistics):
    def create_transport(self) -> Transport:
        return Truck()

class SeaLogistics(Logistics):
    def create_transport(self) -> Transport:
        return Ship()

class AirLogistics(Logistics):
    def create_transport(self) -> Transport:
        return Plane()

# Использование
def get_logistics(transport_type: str) -> Logistics:
    factories = {
        "road": RoadLogistics,
        "sea": SeaLogistics,
        "air": AirLogistics,
    }
    return factories[transport_type]()

logistics = get_logistics("sea")
print(logistics.plan_delivery())  # Логистика: Доставка по морю в контейнере
```

**Best Practices:**

- Используйте параметризованные фабрики для большей гибкости
- Комбинируйте с Registry паттерном для динамической регистрации типов

---

### 3. Abstract Factory (Абстрактная фабрика)

**Назначение:** Предоставляет интерфейс для создания семейств связанных объектов без указания их конкретных классов.

**Когда использовать:**

- Когда система должна быть независима от способа создания продуктов
- Когда нужно создавать семейства взаимосвязанных объектов
- Когда нужна гибкость в переключении между семействами

```python
from abc import ABC, abstractmethod

# Абстрактные продукты
class Button(ABC):
    @abstractmethod
    def render(self) -> str:
        pass

class Checkbox(ABC):
    @abstractmethod
    def render(self) -> str:
        pass

# Windows семейство
class WindowsButton(Button):
    def render(self) -> str:
        return "Windows Button"

class WindowsCheckbox(Checkbox):
    def render(self) -> str:
        return "Windows Checkbox"

# MacOS семейство
class MacButton(Button):
    def render(self) -> str:
        return "MacOS Button"

class MacCheckbox(Checkbox):
    def render(self) -> str:
        return "MacOS Checkbox"

# Абстрактная фабрика
class GUIFactory(ABC):
    @abstractmethod
    def create_button(self) -> Button:
        pass

    @abstractmethod
    def create_checkbox(self) -> Checkbox:
        pass

# Конкретные фабрики
class WindowsFactory(GUIFactory):
    def create_button(self) -> Button:
        return WindowsButton()

    def create_checkbox(self) -> Checkbox:
        return WindowsCheckbox()

class MacFactory(GUIFactory):
    def create_button(self) -> Button:
        return MacButton()

    def create_checkbox(self) -> Checkbox:
        return MacCheckbox()

# Клиентский код
class Application:
    def __init__(self, factory: GUIFactory):
        self.button = factory.create_button()
        self.checkbox = factory.create_checkbox()

    def render(self) -> str:
        return f"{self.button.render()} | {self.checkbox.render()}"

# Использование
import platform

def create_factory() -> GUIFactory:
    if platform.system() == "Windows":
        return WindowsFactory()
    return MacFactory()

app = Application(create_factory())
print(app.render())
```

---

### 4. Builder (Строитель)

**Назначение:** Разделяет конструирование сложного объекта и его представление, позволяя использовать один процесс конструирования для создания различных представлений.

**Когда использовать:**

- Когда объект имеет множество параметров конфигурации
- Когда нужно создавать разные представления объекта
- Когда конструктор становится слишком сложным (телескопический конструктор)

```python
from dataclasses import dataclass, field
from typing import Optional, List

@dataclass
class Pizza:
    size: str
    cheese: str
    toppings: List[str] = field(default_factory=list)
    sauce: Optional[str] = None
    crust: str = "regular"

class PizzaBuilder:
    def __init__(self):
        self._size = "medium"
        self._cheese = "mozzarella"
        self._toppings = []
        self._sauce = None
        self._crust = "regular"

    def size(self, size: str) -> "PizzaBuilder":
        self._size = size
        return self

    def cheese(self, cheese: str) -> "PizzaBuilder":
        self._cheese = cheese
        return self

    def add_topping(self, topping: str) -> "PizzaBuilder":
        self._toppings.append(topping)
        return self

    def sauce(self, sauce: str) -> "PizzaBuilder":
        self._sauce = sauce
        return self

    def crust(self, crust: str) -> "PizzaBuilder":
        self._crust = crust
        return self

    def build(self) -> Pizza:
        return Pizza(
            size=self._size,
            cheese=self._cheese,
            toppings=self._toppings,
            sauce=self._sauce,
            crust=self._crust
        )

# Fluent interface
pizza = (PizzaBuilder()
    .size("large")
    .cheese("parmesan")
    .add_topping("pepperoni")
    .add_topping("mushrooms")
    .sauce("tomato")
    .crust("thin")
    .build())

print(pizza)
# Pizza(size='large', cheese='parmesan', toppings=['pepperoni', 'mushrooms'],
#       sauce='tomato', crust='thin')
```

```go
// Go - Builder pattern
package main

type Server struct {
    Host     string
    Port     int
    Protocol string
    Timeout  int
    MaxConns int
}

type ServerBuilder struct {
    server Server
}

func NewServerBuilder() *ServerBuilder {
    return &ServerBuilder{
        server: Server{
            Host:     "localhost",
            Port:     8080,
            Protocol: "http",
            Timeout:  30,
            MaxConns: 100,
        },
    }
}

func (b *ServerBuilder) Host(host string) *ServerBuilder {
    b.server.Host = host
    return b
}

func (b *ServerBuilder) Port(port int) *ServerBuilder {
    b.server.Port = port
    return b
}

func (b *ServerBuilder) Protocol(protocol string) *ServerBuilder {
    b.server.Protocol = protocol
    return b
}

func (b *ServerBuilder) Build() Server {
    return b.server
}

// Использование
func main() {
    server := NewServerBuilder().
        Host("api.example.com").
        Port(443).
        Protocol("https").
        Build()
}
```

**Best Practices:**

- Используйте fluent interface (цепочку вызовов)
- Добавьте валидацию в метод `build()`
- Рассмотрите Director для стандартных конфигураций

---

### 5. Prototype (Прототип)

**Назначение:** Позволяет копировать объекты, не вдаваясь в подробности их реализации.

**Когда использовать:**

- Когда создание объекта дорогостоящее (например, загрузка из БД)
- Когда нужно избежать иерархии классов фабрик
- Когда объект имеет множество конфигураций

```python
import copy
from abc import ABC, abstractmethod

class Prototype(ABC):
    @abstractmethod
    def clone(self) -> "Prototype":
        pass

class Document(Prototype):
    def __init__(self, title: str, content: str, author: str):
        self.title = title
        self.content = content
        self.author = author
        self.comments = []  # Вложенный объект

    def clone(self) -> "Document":
        # Глубокое копирование для вложенных объектов
        cloned = copy.deepcopy(self)
        return cloned

    def add_comment(self, comment: str):
        self.comments.append(comment)

# Использование
original = Document("Отчёт", "Содержание отчёта", "Иван")
original.add_comment("Первый комментарий")

# Клонирование
draft = original.clone()
draft.title = "Черновик отчёта"
draft.add_comment("Комментарий к черновику")

print(f"Original: {original.title}, comments: {original.comments}")
# Original: Отчёт, comments: ['Первый комментарий']

print(f"Draft: {draft.title}, comments: {draft.comments}")
# Draft: Черновик отчёта, comments: ['Первый комментарий', 'Комментарий к черновику']
```

**Важно:** Различайте поверхностное (shallow) и глубокое (deep) копирование!

---

## Структурные паттерны (Structural Patterns)

Эти паттерны описывают способы компоновки классов и объектов в более крупные структуры.

### 1. Adapter (Адаптер)

**Назначение:** Преобразует интерфейс класса в другой интерфейс, ожидаемый клиентами.

**Когда использовать:**

- При интеграции с legacy-кодом или внешними библиотеками
- Когда нужно использовать класс с несовместимым интерфейсом
- При работе с разными форматами данных

```python
from abc import ABC, abstractmethod
import json
import xml.etree.ElementTree as ET

# Целевой интерфейс
class DataParser(ABC):
    @abstractmethod
    def parse(self, data: str) -> dict:
        pass

# Существующий JSON парсер
class JSONParser(DataParser):
    def parse(self, data: str) -> dict:
        return json.loads(data)

# Legacy XML парсер с другим интерфейсом
class LegacyXMLParser:
    def parse_xml_string(self, xml_string: str) -> ET.Element:
        return ET.fromstring(xml_string)

    def element_to_dict(self, element: ET.Element) -> dict:
        result = {}
        for child in element:
            result[child.tag] = child.text
        return result

# Адаптер для XML парсера
class XMLParserAdapter(DataParser):
    def __init__(self):
        self._legacy_parser = LegacyXMLParser()

    def parse(self, data: str) -> dict:
        element = self._legacy_parser.parse_xml_string(data)
        return self._legacy_parser.element_to_dict(element)

# Клиентский код работает с единым интерфейсом
def process_data(parser: DataParser, data: str) -> dict:
    return parser.parse(data)

# Использование
json_data = '{"name": "John", "age": 30}'
xml_data = '<root><name>John</name><age>30</age></root>'

json_parser = JSONParser()
xml_parser = XMLParserAdapter()

print(process_data(json_parser, json_data))  # {'name': 'John', 'age': 30}
print(process_data(xml_parser, xml_data))    # {'name': 'John', 'age': '30'}
```

---

### 2. Bridge (Мост)

**Назначение:** Разделяет абстракцию и реализацию, позволяя им изменяться независимо.

**Когда использовать:**

- Когда нужно избежать постоянной привязки абстракции к реализации
- Когда и абстракция, и реализация должны расширяться подклассами
- Когда изменения в реализации не должны влиять на клиента

```python
from abc import ABC, abstractmethod

# Реализация (Implementation)
class MessageSender(ABC):
    @abstractmethod
    def send(self, message: str, recipient: str) -> str:
        pass

class EmailSender(MessageSender):
    def send(self, message: str, recipient: str) -> str:
        return f"Email to {recipient}: {message}"

class SMSSender(MessageSender):
    def send(self, message: str, recipient: str) -> str:
        return f"SMS to {recipient}: {message}"

class PushSender(MessageSender):
    def send(self, message: str, recipient: str) -> str:
        return f"Push to {recipient}: {message}"

# Абстракция
class Notification(ABC):
    def __init__(self, sender: MessageSender):
        self._sender = sender

    @abstractmethod
    def notify(self, recipient: str) -> str:
        pass

class AlertNotification(Notification):
    def __init__(self, sender: MessageSender, alert_level: str):
        super().__init__(sender)
        self.alert_level = alert_level

    def notify(self, recipient: str) -> str:
        message = f"[{self.alert_level}] ALERT!"
        return self._sender.send(message, recipient)

class ReminderNotification(Notification):
    def __init__(self, sender: MessageSender, reminder_text: str):
        super().__init__(sender)
        self.reminder_text = reminder_text

    def notify(self, recipient: str) -> str:
        message = f"Reminder: {self.reminder_text}"
        return self._sender.send(message, recipient)

# Использование - комбинируем любую абстракцию с любой реализацией
email = EmailSender()
sms = SMSSender()

alert_email = AlertNotification(email, "CRITICAL")
reminder_sms = ReminderNotification(sms, "Встреча в 15:00")

print(alert_email.notify("admin@example.com"))
# Email to admin@example.com: [CRITICAL] ALERT!

print(reminder_sms.notify("+7999123456"))
# SMS to +7999123456: Reminder: Встреча в 15:00
```

---

### 3. Composite (Компоновщик)

**Назначение:** Объединяет объекты в древовидные структуры для представления иерархий "часть-целое".

**Когда использовать:**

- Когда нужно представить иерархию объектов
- Когда клиент должен одинаково работать с простыми и составными объектами
- При построении рекурсивных структур (файловая система, DOM, организации)

```python
from abc import ABC, abstractmethod
from typing import List

class FileSystemComponent(ABC):
    def __init__(self, name: str):
        self.name = name

    @abstractmethod
    def get_size(self) -> int:
        pass

    @abstractmethod
    def display(self, indent: int = 0) -> str:
        pass

class File(FileSystemComponent):
    def __init__(self, name: str, size: int):
        super().__init__(name)
        self._size = size

    def get_size(self) -> int:
        return self._size

    def display(self, indent: int = 0) -> str:
        return "  " * indent + f"📄 {self.name} ({self._size} bytes)"

class Directory(FileSystemComponent):
    def __init__(self, name: str):
        super().__init__(name)
        self._children: List[FileSystemComponent] = []

    def add(self, component: FileSystemComponent):
        self._children.append(component)

    def remove(self, component: FileSystemComponent):
        self._children.remove(component)

    def get_size(self) -> int:
        return sum(child.get_size() for child in self._children)

    def display(self, indent: int = 0) -> str:
        result = "  " * indent + f"📁 {self.name}/"
        for child in self._children:
            result += "\n" + child.display(indent + 1)
        return result

# Построение структуры
root = Directory("project")
src = Directory("src")
tests = Directory("tests")

src.add(File("main.py", 1500))
src.add(File("utils.py", 800))
tests.add(File("test_main.py", 500))

root.add(src)
root.add(tests)
root.add(File("README.md", 200))

print(root.display())
# 📁 project/
#   📁 src/
#     📄 main.py (1500 bytes)
#     📄 utils.py (800 bytes)
#   📁 tests/
#     📄 test_main.py (500 bytes)
#   📄 README.md (200 bytes)

print(f"\nTotal size: {root.get_size()} bytes")  # 3000 bytes
```

---

### 4. Decorator (Декоратор)

**Назначение:** Динамически добавляет объектам новые обязанности, оставаясь альтернативой подклассам для расширения функциональности.

**Когда использовать:**

- Когда нужно добавлять обязанности объектам динамически и прозрачно
- Когда расширение путём наследования нецелесообразно
- Когда нужна комбинация различных расширений

```python
from abc import ABC, abstractmethod
from functools import wraps
from typing import Callable
import time

# Компонент
class DataSource(ABC):
    @abstractmethod
    def write(self, data: str) -> None:
        pass

    @abstractmethod
    def read(self) -> str:
        pass

# Конкретный компонент
class FileDataSource(DataSource):
    def __init__(self, filename: str):
        self.filename = filename
        self._data = ""

    def write(self, data: str) -> None:
        self._data = data
        print(f"Writing to {self.filename}: {data}")

    def read(self) -> str:
        print(f"Reading from {self.filename}")
        return self._data

# Базовый декоратор
class DataSourceDecorator(DataSource):
    def __init__(self, wrapped: DataSource):
        self._wrapped = wrapped

    def write(self, data: str) -> None:
        self._wrapped.write(data)

    def read(self) -> str:
        return self._wrapped.read()

# Конкретные декораторы
class EncryptionDecorator(DataSourceDecorator):
    def write(self, data: str) -> None:
        encrypted = self._encrypt(data)
        print(f"Encrypting data...")
        super().write(encrypted)

    def read(self) -> str:
        data = super().read()
        print(f"Decrypting data...")
        return self._decrypt(data)

    def _encrypt(self, data: str) -> str:
        # Простое шифрование для примера
        return ''.join(chr(ord(c) + 1) for c in data)

    def _decrypt(self, data: str) -> str:
        return ''.join(chr(ord(c) - 1) for c in data)

class CompressionDecorator(DataSourceDecorator):
    def write(self, data: str) -> None:
        compressed = self._compress(data)
        print(f"Compressing data...")
        super().write(compressed)

    def read(self) -> str:
        data = super().read()
        print(f"Decompressing data...")
        return self._decompress(data)

    def _compress(self, data: str) -> str:
        return f"[compressed]{data}[/compressed]"

    def _decompress(self, data: str) -> str:
        return data.replace("[compressed]", "").replace("[/compressed]", "")

# Использование - комбинируем декораторы
source = FileDataSource("data.txt")
encrypted = EncryptionDecorator(source)
compressed_encrypted = CompressionDecorator(encrypted)

compressed_encrypted.write("Hello, World!")
# Compressing data...
# Encrypting data...
# Writing to data.txt: [compressed]Ifmmp-!Xpsme"[/compressed]

data = compressed_encrypted.read()
# Reading from data.txt
# Decrypting data...
# Decompressing data...
print(f"Result: {data}")  # Hello, World!
```

**Python декораторы функций:**

```python
def timing(func: Callable) -> Callable:
    @wraps(func)
    def wrapper(*args, **kwargs):
        start = time.time()
        result = func(*args, **kwargs)
        print(f"{func.__name__} took {time.time() - start:.4f}s")
        return result
    return wrapper

def retry(times: int = 3):
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            for attempt in range(times):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    if attempt == times - 1:
                        raise
                    print(f"Attempt {attempt + 1} failed, retrying...")
        return wrapper
    return decorator

@timing
@retry(times=3)
def fetch_data(url: str) -> str:
    # Имитация запроса
    return f"Data from {url}"
```

---

### 5. Facade (Фасад)

**Назначение:** Предоставляет унифицированный интерфейс к набору интерфейсов подсистемы.

**Когда использовать:**

- Когда нужно предоставить простой интерфейс к сложной подсистеме
- Когда нужно уменьшить связанность между клиентом и подсистемой
- Для создания точки входа в многослойную систему

```python
# Сложная подсистема видеоконвертации
class VideoFile:
    def __init__(self, filename: str):
        self.filename = filename

class CodecFactory:
    def extract(self, file: VideoFile) -> str:
        return f"codec_for_{file.filename}"

class BitrateReader:
    @staticmethod
    def read(filename: str, codec: str) -> str:
        return f"buffer_{filename}_{codec}"

class AudioMixer:
    def fix(self, buffer: str) -> str:
        return f"fixed_{buffer}"

class VideoConverter:
    def convert(self, buffer: str, format: str) -> str:
        return f"converted_{buffer}_to_{format}"

# Фасад
class VideoConversionFacade:
    """Простой интерфейс для сложной подсистемы конвертации видео"""

    def __init__(self):
        self._codec_factory = CodecFactory()
        self._bitrate_reader = BitrateReader()
        self._audio_mixer = AudioMixer()
        self._converter = VideoConverter()

    def convert(self, filename: str, target_format: str) -> str:
        """Конвертирует видео в указанный формат"""
        print(f"Converting {filename} to {target_format}...")

        file = VideoFile(filename)
        codec = self._codec_factory.extract(file)
        buffer = self._bitrate_reader.read(filename, codec)
        buffer = self._audio_mixer.fix(buffer)
        result = self._converter.convert(buffer, target_format)

        print(f"Conversion complete: {result}")
        return result

# Клиентский код - простой интерфейс
facade = VideoConversionFacade()
facade.convert("movie.avi", "mp4")
```

---

### 6. Flyweight (Приспособленец)

**Назначение:** Использует разделение для эффективной поддержки большого числа мелких объектов.

**Когда использовать:**

- Когда приложение использует множество похожих объектов
- Когда затраты на хранение объектов велики
- Когда большую часть состояния можно вынести во внешнее состояние

```python
from typing import Dict
import sys

class TreeType:
    """Flyweight - разделяемое внутреннее состояние"""
    def __init__(self, name: str, color: str, texture: str):
        self.name = name
        self.color = color
        self.texture = texture

    def draw(self, x: int, y: int) -> str:
        return f"Drawing {self.name} ({self.color}) at ({x}, {y})"

class TreeFactory:
    """Фабрика flyweight объектов"""
    _cache: Dict[str, TreeType] = {}

    @classmethod
    def get_tree_type(cls, name: str, color: str, texture: str) -> TreeType:
        key = f"{name}_{color}_{texture}"
        if key not in cls._cache:
            cls._cache[key] = TreeType(name, color, texture)
            print(f"Created new TreeType: {key}")
        return cls._cache[key]

    @classmethod
    def get_cache_size(cls) -> int:
        return len(cls._cache)

class Tree:
    """Контекст - содержит внешнее состояние"""
    def __init__(self, x: int, y: int, tree_type: TreeType):
        self.x = x
        self.y = y
        self._type = tree_type  # Ссылка на flyweight

    def draw(self) -> str:
        return self._type.draw(self.x, self.y)

class Forest:
    def __init__(self):
        self._trees = []

    def plant_tree(self, x: int, y: int, name: str, color: str, texture: str):
        tree_type = TreeFactory.get_tree_type(name, color, texture)
        tree = Tree(x, y, tree_type)
        self._trees.append(tree)

    def draw(self):
        for tree in self._trees:
            print(tree.draw())

# Использование
forest = Forest()

# Сажаем 1000 деревьев, но создаём только несколько TreeType
import random
tree_configs = [
    ("Oak", "green", "rough"),
    ("Pine", "dark_green", "needle"),
    ("Birch", "light_green", "smooth"),
]

for i in range(1000):
    name, color, texture = random.choice(tree_configs)
    forest.plant_tree(
        x=random.randint(0, 1000),
        y=random.randint(0, 1000),
        name=name,
        color=color,
        texture=texture
    )

print(f"\nTotal trees: 1000")
print(f"TreeType objects created: {TreeFactory.get_cache_size()}")  # 3
```

---

### 7. Proxy (Заместитель)

**Назначение:** Предоставляет суррогат или placeholder для другого объекта для контроля доступа к нему.

**Типы Proxy:**

- **Virtual Proxy** — ленивая инициализация
- **Protection Proxy** — контроль доступа
- **Remote Proxy** — работа с удалёнными объектами
- **Caching Proxy** — кеширование результатов

```python
from abc import ABC, abstractmethod
from typing import Optional
from datetime import datetime, timedelta

# Интерфейс субъекта
class Database(ABC):
    @abstractmethod
    def query(self, sql: str) -> str:
        pass

# Реальный субъект
class RealDatabase(Database):
    def __init__(self, connection_string: str):
        print(f"Connecting to database: {connection_string}")
        self._connection = connection_string
        # Имитация тяжёлой инициализации

    def query(self, sql: str) -> str:
        return f"Results for: {sql}"

# Virtual Proxy - ленивая инициализация
class LazyDatabaseProxy(Database):
    def __init__(self, connection_string: str):
        self._connection_string = connection_string
        self._database: Optional[RealDatabase] = None

    def query(self, sql: str) -> str:
        if self._database is None:
            self._database = RealDatabase(self._connection_string)
        return self._database.query(sql)

# Protection Proxy - контроль доступа
class SecureDatabaseProxy(Database):
    def __init__(self, database: Database, allowed_users: set):
        self._database = database
        self._allowed_users = allowed_users
        self._current_user: Optional[str] = None

    def login(self, username: str):
        self._current_user = username

    def query(self, sql: str) -> str:
        if self._current_user not in self._allowed_users:
            raise PermissionError(f"User {self._current_user} not authorized")

        # Логирование запросов
        print(f"[{datetime.now()}] User {self._current_user}: {sql}")
        return self._database.query(sql)

# Caching Proxy
class CachingDatabaseProxy(Database):
    def __init__(self, database: Database, cache_ttl: int = 60):
        self._database = database
        self._cache = {}
        self._cache_ttl = timedelta(seconds=cache_ttl)

    def query(self, sql: str) -> str:
        now = datetime.now()

        if sql in self._cache:
            result, cached_at = self._cache[sql]
            if now - cached_at < self._cache_ttl:
                print(f"Cache hit for: {sql}")
                return result

        print(f"Cache miss for: {sql}")
        result = self._database.query(sql)
        self._cache[sql] = (result, now)
        return result

# Использование
# Ленивая загрузка - БД подключится только при первом запросе
lazy_db = LazyDatabaseProxy("postgresql://localhost/mydb")
print("Proxy created, database not connected yet")
result = lazy_db.query("SELECT * FROM users")  # Подключение происходит здесь

# Защищённый доступ
secure_db = SecureDatabaseProxy(lazy_db, {"admin", "analyst"})
secure_db.login("admin")
secure_db.query("SELECT * FROM sensitive_data")

# Кеширование
cached_db = CachingDatabaseProxy(lazy_db, cache_ttl=300)
cached_db.query("SELECT * FROM products")  # Cache miss
cached_db.query("SELECT * FROM products")  # Cache hit
```

---

## Поведенческие паттерны (Behavioral Patterns)

Эти паттерны описывают алгоритмы и способы взаимодействия между объектами.

### 1. Chain of Responsibility (Цепочка обязанностей)

**Назначение:** Позволяет передать запрос по цепочке обработчиков, где каждый решает, обработать запрос или передать следующему.

**Когда использовать:**

- Когда есть несколько объектов, способных обработать запрос
- Когда набор обработчиков должен задаваться динамически
- Для реализации middleware, валидаторов, логгеров

```python
from abc import ABC, abstractmethod
from typing import Optional, Any
from dataclasses import dataclass

@dataclass
class Request:
    user: str
    action: str
    amount: float = 0

class Handler(ABC):
    def __init__(self):
        self._next: Optional[Handler] = None

    def set_next(self, handler: "Handler") -> "Handler":
        self._next = handler
        return handler

    def handle(self, request: Request) -> Optional[str]:
        if self._next:
            return self._next.handle(request)
        return None

class AuthenticationHandler(Handler):
    def __init__(self, valid_users: set):
        super().__init__()
        self._valid_users = valid_users

    def handle(self, request: Request) -> Optional[str]:
        if request.user not in self._valid_users:
            return f"Authentication failed for {request.user}"
        print(f"✓ User {request.user} authenticated")
        return super().handle(request)

class AuthorizationHandler(Handler):
    def __init__(self, permissions: dict):
        super().__init__()
        self._permissions = permissions

    def handle(self, request: Request) -> Optional[str]:
        allowed = self._permissions.get(request.user, [])
        if request.action not in allowed:
            return f"User {request.user} not authorized for {request.action}"
        print(f"✓ Action {request.action} authorized")
        return super().handle(request)

class RateLimitHandler(Handler):
    def __init__(self, max_amount: float):
        super().__init__()
        self._max_amount = max_amount

    def handle(self, request: Request) -> Optional[str]:
        if request.amount > self._max_amount:
            return f"Amount {request.amount} exceeds limit {self._max_amount}"
        print(f"✓ Amount {request.amount} within limits")
        return super().handle(request)

class ProcessingHandler(Handler):
    def handle(self, request: Request) -> Optional[str]:
        print(f"✓ Processing {request.action} for {request.user}")
        return f"Success: {request.action} processed"

# Построение цепочки
auth = AuthenticationHandler({"alice", "bob", "admin"})
authz = AuthorizationHandler({
    "alice": ["read"],
    "bob": ["read", "write"],
    "admin": ["read", "write", "delete"],
})
rate_limit = RateLimitHandler(max_amount=1000)
processor = ProcessingHandler()

auth.set_next(authz).set_next(rate_limit).set_next(processor)

# Использование
requests = [
    Request(user="alice", action="read", amount=100),
    Request(user="bob", action="delete", amount=500),
    Request(user="admin", action="write", amount=5000),
    Request(user="hacker", action="read", amount=0),
]

for req in requests:
    print(f"\n--- Processing: {req} ---")
    result = auth.handle(req)
    print(f"Result: {result}")
```

---

### 2. Command (Команда)

**Назначение:** Инкапсулирует запрос как объект, позволяя параметризовать клиентов с различными запросами, ставить запросы в очередь или логировать их, а также поддерживать отмену операций.

**Когда использовать:**

- Для параметризации объектов выполняемым действием
- Для реализации очередей, журналирования, отмены операций
- Для транзакционного поведения

```python
from abc import ABC, abstractmethod
from typing import List
from dataclasses import dataclass, field
from datetime import datetime

class Command(ABC):
    @abstractmethod
    def execute(self) -> None:
        pass

    @abstractmethod
    def undo(self) -> None:
        pass

@dataclass
class Document:
    content: str = ""

    def insert(self, position: int, text: str):
        self.content = self.content[:position] + text + self.content[position:]

    def delete(self, position: int, length: int) -> str:
        deleted = self.content[position:position + length]
        self.content = self.content[:position] + self.content[position + length:]
        return deleted

class InsertCommand(Command):
    def __init__(self, document: Document, position: int, text: str):
        self._document = document
        self._position = position
        self._text = text

    def execute(self) -> None:
        self._document.insert(self._position, self._text)

    def undo(self) -> None:
        self._document.delete(self._position, len(self._text))

class DeleteCommand(Command):
    def __init__(self, document: Document, position: int, length: int):
        self._document = document
        self._position = position
        self._length = length
        self._deleted_text = ""

    def execute(self) -> None:
        self._deleted_text = self._document.delete(self._position, self._length)

    def undo(self) -> None:
        self._document.insert(self._position, self._deleted_text)

class CommandHistory:
    def __init__(self):
        self._history: List[Command] = []
        self._undo_stack: List[Command] = []

    def execute(self, command: Command):
        command.execute()
        self._history.append(command)
        self._undo_stack.clear()  # Очищаем redo после нового действия

    def undo(self) -> bool:
        if not self._history:
            return False
        command = self._history.pop()
        command.undo()
        self._undo_stack.append(command)
        return True

    def redo(self) -> bool:
        if not self._undo_stack:
            return False
        command = self._undo_stack.pop()
        command.execute()
        self._history.append(command)
        return True

# Использование
doc = Document()
history = CommandHistory()

# Выполняем команды
history.execute(InsertCommand(doc, 0, "Hello"))
print(doc.content)  # "Hello"

history.execute(InsertCommand(doc, 5, " World"))
print(doc.content)  # "Hello World"

history.execute(DeleteCommand(doc, 5, 6))
print(doc.content)  # "Hello"

# Отмена
history.undo()
print(doc.content)  # "Hello World"

history.undo()
print(doc.content)  # "Hello"

# Повтор
history.redo()
print(doc.content)  # "Hello World"
```

---

### 3. Iterator (Итератор)

**Назначение:** Предоставляет способ последовательного доступа к элементам составного объекта без раскрытия его внутреннего представления.

```python
from typing import TypeVar, Generic, Iterator, List
from collections.abc import Iterable

T = TypeVar('T')

class TreeNode(Generic[T]):
    def __init__(self, value: T):
        self.value = value
        self.left: TreeNode[T] | None = None
        self.right: TreeNode[T] | None = None

class BinaryTree(Generic[T]):
    def __init__(self, root: TreeNode[T]):
        self.root = root

    def __iter__(self) -> Iterator[T]:
        """Итератор in-order обхода по умолчанию"""
        return self.in_order()

    def in_order(self) -> Iterator[T]:
        """In-order (left, root, right)"""
        def traverse(node: TreeNode[T] | None) -> Iterator[T]:
            if node:
                yield from traverse(node.left)
                yield node.value
                yield from traverse(node.right)
        return traverse(self.root)

    def pre_order(self) -> Iterator[T]:
        """Pre-order (root, left, right)"""
        def traverse(node: TreeNode[T] | None) -> Iterator[T]:
            if node:
                yield node.value
                yield from traverse(node.left)
                yield from traverse(node.right)
        return traverse(self.root)

    def post_order(self) -> Iterator[T]:
        """Post-order (left, right, root)"""
        def traverse(node: TreeNode[T] | None) -> Iterator[T]:
            if node:
                yield from traverse(node.left)
                yield from traverse(node.right)
                yield node.value
        return traverse(self.root)

    def level_order(self) -> Iterator[T]:
        """Breadth-first (level by level)"""
        from collections import deque
        queue = deque([self.root])
        while queue:
            node = queue.popleft()
            if node:
                yield node.value
                queue.append(node.left)
                queue.append(node.right)

# Построение дерева
#       4
#      / \
#     2   6
#    / \ / \
#   1  3 5  7

root = TreeNode(4)
root.left = TreeNode(2)
root.right = TreeNode(6)
root.left.left = TreeNode(1)
root.left.right = TreeNode(3)
root.right.left = TreeNode(5)
root.right.right = TreeNode(7)

tree = BinaryTree(root)

print("In-order:   ", list(tree.in_order()))    # [1, 2, 3, 4, 5, 6, 7]
print("Pre-order:  ", list(tree.pre_order()))   # [4, 2, 1, 3, 6, 5, 7]
print("Post-order: ", list(tree.post_order()))  # [1, 3, 2, 5, 7, 6, 4]
print("Level-order:", list(tree.level_order())) # [4, 2, 6, 1, 3, 5, 7]

# Использование в for
for value in tree:
    print(value, end=" ")  # 1 2 3 4 5 6 7
```

---

### 4. Mediator (Посредник)

**Назначение:** Определяет объект, инкапсулирующий способ взаимодействия множества объектов. Способствует слабой связанности, избавляя объекты от необходимости явно ссылаться друг на друга.

```python
from abc import ABC, abstractmethod
from typing import Dict, List

class Mediator(ABC):
    @abstractmethod
    def notify(self, sender: "Component", event: str, data: any = None):
        pass

class Component:
    def __init__(self, name: str):
        self.name = name
        self._mediator: Mediator | None = None

    def set_mediator(self, mediator: Mediator):
        self._mediator = mediator

    def send(self, event: str, data: any = None):
        if self._mediator:
            self._mediator.notify(self, event, data)

# Конкретные компоненты
class Button(Component):
    def click(self):
        print(f"Button '{self.name}' clicked")
        self.send("click")

class TextBox(Component):
    def __init__(self, name: str):
        super().__init__(name)
        self.text = ""

    def set_text(self, text: str):
        self.text = text
        print(f"TextBox '{self.name}' text set to: {text}")
        self.send("text_changed", text)

    def clear(self):
        self.text = ""
        print(f"TextBox '{self.name}' cleared")

class CheckBox(Component):
    def __init__(self, name: str):
        super().__init__(name)
        self.checked = False

    def toggle(self):
        self.checked = not self.checked
        print(f"CheckBox '{self.name}' is now {'checked' if self.checked else 'unchecked'}")
        self.send("toggled", self.checked)

class ListBox(Component):
    def __init__(self, name: str):
        super().__init__(name)
        self.items: List[str] = []
        self.visible = True

    def add_item(self, item: str):
        self.items.append(item)
        print(f"ListBox '{self.name}' added: {item}")

    def set_visible(self, visible: bool):
        self.visible = visible
        print(f"ListBox '{self.name}' visibility: {visible}")

# Конкретный посредник (Dialog)
class AuthDialog(Mediator):
    def __init__(self):
        self.login_button = Button("Login")
        self.cancel_button = Button("Cancel")
        self.username_input = TextBox("Username")
        self.password_input = TextBox("Password")
        self.remember_me = CheckBox("Remember Me")
        self.user_list = ListBox("Users")

        # Регистрация компонентов
        for component in [
            self.login_button, self.cancel_button,
            self.username_input, self.password_input,
            self.remember_me, self.user_list
        ]:
            component.set_mediator(self)

    def notify(self, sender: Component, event: str, data: any = None):
        if sender == self.login_button and event == "click":
            self._handle_login()

        elif sender == self.cancel_button and event == "click":
            self._handle_cancel()

        elif sender == self.remember_me and event == "toggled":
            self.user_list.set_visible(data)

        elif sender == self.username_input and event == "text_changed":
            # Показываем подсказки при вводе
            if len(data) >= 2:
                self.user_list.items.clear()
                self.user_list.add_item(f"Suggestion: {data}@example.com")

    def _handle_login(self):
        username = self.username_input.text
        password = self.password_input.text
        remember = self.remember_me.checked

        print(f"\n=== Login attempt ===")
        print(f"Username: {username}")
        print(f"Password: {'*' * len(password)}")
        print(f"Remember: {remember}")

    def _handle_cancel(self):
        self.username_input.clear()
        self.password_input.clear()
        print("Dialog cancelled")

# Использование
dialog = AuthDialog()

dialog.username_input.set_text("john")
dialog.password_input.set_text("secret123")
dialog.remember_me.toggle()
dialog.login_button.click()
```

---

### 5. Memento (Снимок)

**Назначение:** Позволяет сохранять и восстанавливать предыдущее состояние объекта без раскрытия деталей его реализации.

```python
from dataclasses import dataclass, field
from typing import List, Dict, Any
from datetime import datetime
from copy import deepcopy

@dataclass
class EditorMemento:
    """Memento - снимок состояния"""
    _state: Dict[str, Any]
    _created_at: datetime = field(default_factory=datetime.now)

    def get_state(self) -> Dict[str, Any]:
        return deepcopy(self._state)

    def get_info(self) -> str:
        return f"Snapshot at {self._created_at.strftime('%H:%M:%S')}"

class TextEditor:
    """Originator - создатель снимков"""
    def __init__(self):
        self._text = ""
        self._cursor_position = 0
        self._selection_start = 0
        self._selection_end = 0

    def type_text(self, text: str):
        self._text = (
            self._text[:self._cursor_position] +
            text +
            self._text[self._cursor_position:]
        )
        self._cursor_position += len(text)

    def delete(self, count: int = 1):
        start = max(0, self._cursor_position - count)
        self._text = self._text[:start] + self._text[self._cursor_position:]
        self._cursor_position = start

    def move_cursor(self, position: int):
        self._cursor_position = max(0, min(position, len(self._text)))

    def select(self, start: int, end: int):
        self._selection_start = start
        self._selection_end = end

    def save(self) -> EditorMemento:
        """Создаёт снимок текущего состояния"""
        state = {
            "text": self._text,
            "cursor": self._cursor_position,
            "selection_start": self._selection_start,
            "selection_end": self._selection_end,
        }
        return EditorMemento(state)

    def restore(self, memento: EditorMemento):
        """Восстанавливает состояние из снимка"""
        state = memento.get_state()
        self._text = state["text"]
        self._cursor_position = state["cursor"]
        self._selection_start = state["selection_start"]
        self._selection_end = state["selection_end"]

    def __str__(self) -> str:
        cursor_display = (
            self._text[:self._cursor_position] +
            "|" +
            self._text[self._cursor_position:]
        )
        return f"Text: '{cursor_display}'"

class History:
    """Caretaker - хранитель снимков"""
    def __init__(self, editor: TextEditor):
        self._editor = editor
        self._snapshots: List[EditorMemento] = []
        self._current = -1

    def backup(self):
        # Удаляем "будущие" снимки при новом сохранении
        self._snapshots = self._snapshots[:self._current + 1]
        self._snapshots.append(self._editor.save())
        self._current += 1

    def undo(self) -> bool:
        if self._current <= 0:
            return False
        self._current -= 1
        self._editor.restore(self._snapshots[self._current])
        return True

    def redo(self) -> bool:
        if self._current >= len(self._snapshots) - 1:
            return False
        self._current += 1
        self._editor.restore(self._snapshots[self._current])
        return True

    def show_history(self):
        print("\n--- History ---")
        for i, snapshot in enumerate(self._snapshots):
            marker = " <-- current" if i == self._current else ""
            print(f"  [{i}] {snapshot.get_info()}{marker}")

# Использование
editor = TextEditor()
history = History(editor)

# Сохраняем начальное состояние
history.backup()

editor.type_text("Hello")
history.backup()
print(editor)  # Text: 'Hello|'

editor.type_text(" World")
history.backup()
print(editor)  # Text: 'Hello World|'

editor.type_text("!")
history.backup()
print(editor)  # Text: 'Hello World!|'

# Отмена
history.undo()
print(editor)  # Text: 'Hello World|'

history.undo()
print(editor)  # Text: 'Hello|'

# Повтор
history.redo()
print(editor)  # Text: 'Hello World|'

history.show_history()
```

---

### 6. Observer (Наблюдатель)

**Назначение:** Определяет зависимость "один ко многим" между объектами так, что при изменении состояния одного объекта все зависящие от него оповещаются и обновляются автоматически.

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any, Callable
from dataclasses import dataclass
from enum import Enum

class EventType(Enum):
    PRICE_CHANGED = "price_changed"
    STOCK_LOW = "stock_low"
    ITEM_SOLD = "item_sold"

@dataclass
class Event:
    type: EventType
    data: Dict[str, Any]

# Классический подход с интерфейсами
class Observer(ABC):
    @abstractmethod
    def update(self, event: Event):
        pass

class Subject(ABC):
    @abstractmethod
    def attach(self, event_type: EventType, observer: Observer):
        pass

    @abstractmethod
    def detach(self, event_type: EventType, observer: Observer):
        pass

    @abstractmethod
    def notify(self, event: Event):
        pass

class Product(Subject):
    def __init__(self, name: str, price: float, stock: int):
        self.name = name
        self._price = price
        self._stock = stock
        self._observers: Dict[EventType, List[Observer]] = {
            event_type: [] for event_type in EventType
        }

    def attach(self, event_type: EventType, observer: Observer):
        self._observers[event_type].append(observer)

    def detach(self, event_type: EventType, observer: Observer):
        self._observers[event_type].remove(observer)

    def notify(self, event: Event):
        for observer in self._observers[event.type]:
            observer.update(event)

    @property
    def price(self) -> float:
        return self._price

    @price.setter
    def price(self, value: float):
        old_price = self._price
        self._price = value
        self.notify(Event(
            EventType.PRICE_CHANGED,
            {"product": self.name, "old": old_price, "new": value}
        ))

    @property
    def stock(self) -> int:
        return self._stock

    def sell(self, quantity: int = 1):
        if self._stock >= quantity:
            self._stock -= quantity
            self.notify(Event(
                EventType.ITEM_SOLD,
                {"product": self.name, "quantity": quantity, "remaining": self._stock}
            ))
            if self._stock < 5:
                self.notify(Event(
                    EventType.STOCK_LOW,
                    {"product": self.name, "remaining": self._stock}
                ))

# Конкретные наблюдатели
class EmailNotifier(Observer):
    def update(self, event: Event):
        if event.type == EventType.PRICE_CHANGED:
            print(f"📧 Email: Price of {event.data['product']} "
                  f"changed from ${event.data['old']} to ${event.data['new']}")

class InventoryManager(Observer):
    def update(self, event: Event):
        if event.type == EventType.STOCK_LOW:
            print(f"📦 Inventory Alert: {event.data['product']} "
                  f"low stock ({event.data['remaining']} remaining). Reorder needed!")

class SalesAnalytics(Observer):
    def __init__(self):
        self.total_sales = 0

    def update(self, event: Event):
        if event.type == EventType.ITEM_SOLD:
            self.total_sales += event.data['quantity']
            print(f"📊 Analytics: {event.data['quantity']} units sold. "
                  f"Total sales: {self.total_sales}")

# Использование
product = Product("iPhone", 999.99, 10)

# Подписка наблюдателей
email = EmailNotifier()
inventory = InventoryManager()
analytics = SalesAnalytics()

product.attach(EventType.PRICE_CHANGED, email)
product.attach(EventType.STOCK_LOW, inventory)
product.attach(EventType.ITEM_SOLD, analytics)

# События
product.price = 899.99  # Email уведомление
product.sell(3)         # Analytics
product.sell(4)         # Analytics + Inventory Alert (осталось 3)
```

**Современный подход с callback'ами:**

```python
from typing import Callable, Dict, List
from dataclasses import dataclass

class EventEmitter:
    def __init__(self):
        self._listeners: Dict[str, List[Callable]] = {}

    def on(self, event: str, callback: Callable):
        if event not in self._listeners:
            self._listeners[event] = []
        self._listeners[event].append(callback)
        return lambda: self.off(event, callback)  # Возвращаем unsubscribe функцию

    def off(self, event: str, callback: Callable):
        if event in self._listeners:
            self._listeners[event].remove(callback)

    def emit(self, event: str, *args, **kwargs):
        for callback in self._listeners.get(event, []):
            callback(*args, **kwargs)

# Использование
emitter = EventEmitter()

# Подписка с автоматической отпиской
unsubscribe = emitter.on("data", lambda x: print(f"Received: {x}"))

emitter.emit("data", {"value": 42})
unsubscribe()  # Отписываемся
emitter.emit("data", {"value": 100})  # Не будет обработано
```

---

### 7. State (Состояние)

**Назначение:** Позволяет объекту изменять своё поведение при изменении внутреннего состояния. Объект будет казаться изменившим свой класс.

```python
from abc import ABC, abstractmethod
from typing import Optional

class State(ABC):
    """Базовое состояние"""
    @property
    def context(self) -> "MediaPlayer":
        return self._context

    @context.setter
    def context(self, context: "MediaPlayer"):
        self._context = context

    @abstractmethod
    def play(self) -> str:
        pass

    @abstractmethod
    def pause(self) -> str:
        pass

    @abstractmethod
    def stop(self) -> str:
        pass

    @abstractmethod
    def next_track(self) -> str:
        pass

class StoppedState(State):
    def play(self) -> str:
        self.context.change_state(PlayingState())
        return "▶️ Starting playback"

    def pause(self) -> str:
        return "⏹️ Already stopped"

    def stop(self) -> str:
        return "⏹️ Already stopped"

    def next_track(self) -> str:
        self.context.current_track += 1
        return f"⏭️ Skipped to track {self.context.current_track}"

class PlayingState(State):
    def play(self) -> str:
        return "▶️ Already playing"

    def pause(self) -> str:
        self.context.change_state(PausedState())
        return "⏸️ Paused"

    def stop(self) -> str:
        self.context.change_state(StoppedState())
        return "⏹️ Stopped"

    def next_track(self) -> str:
        self.context.current_track += 1
        return f"⏭️ Playing track {self.context.current_track}"

class PausedState(State):
    def play(self) -> str:
        self.context.change_state(PlayingState())
        return "▶️ Resuming playback"

    def pause(self) -> str:
        return "⏸️ Already paused"

    def stop(self) -> str:
        self.context.change_state(StoppedState())
        return "⏹️ Stopped"

    def next_track(self) -> str:
        self.context.current_track += 1
        return f"⏭️ Skipped to track {self.context.current_track} (still paused)"

class MediaPlayer:
    """Context - контекст"""
    def __init__(self):
        self._state: State = StoppedState()
        self._state.context = self
        self.current_track = 1

    def change_state(self, state: State):
        print(f"  State: {type(self._state).__name__} → {type(state).__name__}")
        self._state = state
        self._state.context = self

    def play(self) -> str:
        return self._state.play()

    def pause(self) -> str:
        return self._state.pause()

    def stop(self) -> str:
        return self._state.stop()

    def next_track(self) -> str:
        return self._state.next_track()

# Использование
player = MediaPlayer()

print(player.play())       # Starting playback (Stopped → Playing)
print(player.next_track()) # Playing track 2
print(player.pause())      # Paused (Playing → Paused)
print(player.play())       # Resuming playback (Paused → Playing)
print(player.stop())       # Stopped (Playing → Stopped)
print(player.pause())      # Already stopped
```

---

### 8. Strategy (Стратегия)

**Назначение:** Определяет семейство алгоритмов, инкапсулирует каждый из них и делает их взаимозаменяемыми.

```python
from abc import ABC, abstractmethod
from typing import List, Callable
from dataclasses import dataclass

@dataclass
class Order:
    items: List[str]
    subtotal: float
    shipping_address: str
    weight: float  # в кг

# Интерфейс стратегии
class ShippingStrategy(ABC):
    @abstractmethod
    def calculate(self, order: Order) -> float:
        pass

    @abstractmethod
    def get_description(self) -> str:
        pass

# Конкретные стратегии
class StandardShipping(ShippingStrategy):
    def calculate(self, order: Order) -> float:
        return 5.0 + order.weight * 0.5

    def get_description(self) -> str:
        return "Standard Shipping (5-7 business days)"

class ExpressShipping(ShippingStrategy):
    def calculate(self, order: Order) -> float:
        return 15.0 + order.weight * 1.5

    def get_description(self) -> str:
        return "Express Shipping (2-3 business days)"

class FreeShipping(ShippingStrategy):
    def __init__(self, min_order: float = 50.0):
        self.min_order = min_order

    def calculate(self, order: Order) -> float:
        if order.subtotal >= self.min_order:
            return 0.0
        return 5.0 + order.weight * 0.5

    def get_description(self) -> str:
        return f"Free Shipping (orders over ${self.min_order})"

class PickupStrategy(ShippingStrategy):
    def calculate(self, order: Order) -> float:
        return 0.0

    def get_description(self) -> str:
        return "Store Pickup (same day)"

# Контекст
class ShoppingCart:
    def __init__(self):
        self._items: List[str] = []
        self._subtotal = 0.0
        self._weight = 0.0
        self._shipping_strategy: ShippingStrategy = StandardShipping()

    def add_item(self, name: str, price: float, weight: float):
        self._items.append(name)
        self._subtotal += price
        self._weight += weight

    def set_shipping_strategy(self, strategy: ShippingStrategy):
        self._shipping_strategy = strategy

    def get_order(self) -> Order:
        return Order(
            items=self._items,
            subtotal=self._subtotal,
            shipping_address="",
            weight=self._weight
        )

    def calculate_total(self) -> dict:
        order = self.get_order()
        shipping = self._shipping_strategy.calculate(order)
        return {
            "subtotal": self._subtotal,
            "shipping": shipping,
            "shipping_method": self._shipping_strategy.get_description(),
            "total": self._subtotal + shipping
        }

# Использование
cart = ShoppingCart()
cart.add_item("Laptop", 999.99, 2.5)
cart.add_item("Mouse", 29.99, 0.1)

# Разные стратегии доставки
strategies = [
    StandardShipping(),
    ExpressShipping(),
    FreeShipping(min_order=500),
    PickupStrategy()
]

for strategy in strategies:
    cart.set_shipping_strategy(strategy)
    result = cart.calculate_total()
    print(f"\n{result['shipping_method']}")
    print(f"  Subtotal: ${result['subtotal']:.2f}")
    print(f"  Shipping: ${result['shipping']:.2f}")
    print(f"  Total:    ${result['total']:.2f}")
```

**Функциональный подход:**

```python
from typing import Callable

# Стратегии как функции
ShippingCalculator = Callable[[Order], float]

def standard_shipping(order: Order) -> float:
    return 5.0 + order.weight * 0.5

def express_shipping(order: Order) -> float:
    return 15.0 + order.weight * 1.5

def create_free_shipping(min_order: float) -> ShippingCalculator:
    def calculate(order: Order) -> float:
        return 0.0 if order.subtotal >= min_order else 5.0 + order.weight * 0.5
    return calculate

# Использование
order = Order(items=["Laptop"], subtotal=999.99, shipping_address="", weight=2.5)

calculators = {
    "standard": standard_shipping,
    "express": express_shipping,
    "free_over_500": create_free_shipping(500),
}

for name, calc in calculators.items():
    print(f"{name}: ${calc(order):.2f}")
```

---

### 9. Template Method (Шаблонный метод)

**Назначение:** Определяет скелет алгоритма в базовом классе, позволяя подклассам переопределять определённые шаги, не изменяя структуру алгоритма.

```python
from abc import ABC, abstractmethod
from typing import Dict, Any
import json
import csv
from io import StringIO

class DataMiner(ABC):
    """Шаблонный метод определяет скелет алгоритма"""

    def mine(self, path: str) -> Dict[str, Any]:
        """Template Method - неизменяемый алгоритм"""
        raw_data = self.open_file(path)
        parsed_data = self.parse_data(raw_data)
        analyzed_data = self.analyze_data(parsed_data)
        report = self.create_report(analyzed_data)
        self.send_report(report)
        return report

    # Абстрактные методы - ДОЛЖНЫ быть переопределены
    @abstractmethod
    def open_file(self, path: str) -> str:
        pass

    @abstractmethod
    def parse_data(self, raw_data: str) -> list:
        pass

    # Конкретные методы - общая реализация
    def analyze_data(self, data: list) -> Dict[str, Any]:
        return {
            "count": len(data),
            "data": data
        }

    def create_report(self, analyzed_data: Dict[str, Any]) -> Dict[str, Any]:
        return {
            "status": "success",
            "records_processed": analyzed_data["count"],
            "summary": analyzed_data
        }

    # Hook методы - могут быть переопределены
    def send_report(self, report: Dict[str, Any]) -> None:
        print(f"Report generated: {report['records_processed']} records")

class JSONDataMiner(DataMiner):
    def open_file(self, path: str) -> str:
        # Имитация чтения файла
        return '[{"name": "Alice", "age": 30}, {"name": "Bob", "age": 25}]'

    def parse_data(self, raw_data: str) -> list:
        return json.loads(raw_data)

class CSVDataMiner(DataMiner):
    def open_file(self, path: str) -> str:
        return "name,age\nAlice,30\nBob,25"

    def parse_data(self, raw_data: str) -> list:
        reader = csv.DictReader(StringIO(raw_data))
        return list(reader)

    # Переопределяем hook
    def send_report(self, report: Dict[str, Any]) -> None:
        super().send_report(report)
        print("  → Также отправляем CSV-отчёт по email")

class XMLDataMiner(DataMiner):
    def open_file(self, path: str) -> str:
        return "<users><user name='Alice' age='30'/><user name='Bob' age='25'/></users>"

    def parse_data(self, raw_data: str) -> list:
        import xml.etree.ElementTree as ET
        root = ET.fromstring(raw_data)
        return [user.attrib for user in root.findall('user')]

    # Расширяем анализ
    def analyze_data(self, data: list) -> Dict[str, Any]:
        base = super().analyze_data(data)
        base["format"] = "XML"
        base["average_age"] = sum(int(d["age"]) for d in data) / len(data)
        return base

# Использование
miners = [
    ("JSON", JSONDataMiner()),
    ("CSV", CSVDataMiner()),
    ("XML", XMLDataMiner()),
]

for name, miner in miners:
    print(f"\n=== {name} Miner ===")
    report = miner.mine(f"data.{name.lower()}")
```

---

### 10. Visitor (Посетитель)

**Назначение:** Позволяет добавлять новые операции к объектам, не изменяя их классы.

```python
from abc import ABC, abstractmethod
from typing import List
from dataclasses import dataclass

# Элементы
class Shape(ABC):
    @abstractmethod
    def accept(self, visitor: "ShapeVisitor") -> any:
        pass

@dataclass
class Circle(Shape):
    radius: float
    x: float = 0
    y: float = 0

    def accept(self, visitor: "ShapeVisitor") -> any:
        return visitor.visit_circle(self)

@dataclass
class Rectangle(Shape):
    width: float
    height: float
    x: float = 0
    y: float = 0

    def accept(self, visitor: "ShapeVisitor") -> any:
        return visitor.visit_rectangle(self)

@dataclass
class Triangle(Shape):
    a: float  # стороны
    b: float
    c: float
    x: float = 0
    y: float = 0

    def accept(self, visitor: "ShapeVisitor") -> any:
        return visitor.visit_triangle(self)

# Visitor интерфейс
class ShapeVisitor(ABC):
    @abstractmethod
    def visit_circle(self, circle: Circle) -> any:
        pass

    @abstractmethod
    def visit_rectangle(self, rectangle: Rectangle) -> any:
        pass

    @abstractmethod
    def visit_triangle(self, triangle: Triangle) -> any:
        pass

# Конкретные посетители
class AreaCalculator(ShapeVisitor):
    """Вычисляет площадь фигуры"""

    def visit_circle(self, circle: Circle) -> float:
        import math
        return math.pi * circle.radius ** 2

    def visit_rectangle(self, rectangle: Rectangle) -> float:
        return rectangle.width * rectangle.height

    def visit_triangle(self, triangle: Triangle) -> float:
        # Формула Герона
        import math
        s = (triangle.a + triangle.b + triangle.c) / 2
        return math.sqrt(s * (s - triangle.a) * (s - triangle.b) * (s - triangle.c))

class PerimeterCalculator(ShapeVisitor):
    """Вычисляет периметр фигуры"""

    def visit_circle(self, circle: Circle) -> float:
        import math
        return 2 * math.pi * circle.radius

    def visit_rectangle(self, rectangle: Rectangle) -> float:
        return 2 * (rectangle.width + rectangle.height)

    def visit_triangle(self, triangle: Triangle) -> float:
        return triangle.a + triangle.b + triangle.c

class SVGExporter(ShapeVisitor):
    """Экспортирует фигуру в SVG"""

    def visit_circle(self, circle: Circle) -> str:
        return f'<circle cx="{circle.x}" cy="{circle.y}" r="{circle.radius}" />'

    def visit_rectangle(self, rectangle: Rectangle) -> str:
        return f'<rect x="{rectangle.x}" y="{rectangle.y}" width="{rectangle.width}" height="{rectangle.height}" />'

    def visit_triangle(self, triangle: Triangle) -> str:
        # Упрощённый SVG для треугольника
        return f'<polygon points="{triangle.x},{triangle.y} {triangle.x + triangle.a},{triangle.y} {triangle.x + triangle.a/2},{triangle.y - triangle.b}" />'

class JSONSerializer(ShapeVisitor):
    """Сериализует фигуру в JSON"""

    def visit_circle(self, circle: Circle) -> dict:
        return {"type": "circle", "radius": circle.radius, "x": circle.x, "y": circle.y}

    def visit_rectangle(self, rectangle: Rectangle) -> dict:
        return {"type": "rectangle", "width": rectangle.width, "height": rectangle.height}

    def visit_triangle(self, triangle: Triangle) -> dict:
        return {"type": "triangle", "sides": [triangle.a, triangle.b, triangle.c]}

# Использование
shapes: List[Shape] = [
    Circle(radius=5, x=10, y=10),
    Rectangle(width=4, height=6, x=20, y=20),
    Triangle(a=3, b=4, c=5, x=30, y=30)
]

# Разные операции без изменения классов фигур
area_calc = AreaCalculator()
perim_calc = PerimeterCalculator()
svg_export = SVGExporter()
json_serial = JSONSerializer()

print("=== Shape Analysis ===")
for shape in shapes:
    print(f"\n{type(shape).__name__}:")
    print(f"  Area: {shape.accept(area_calc):.2f}")
    print(f"  Perimeter: {shape.accept(perim_calc):.2f}")
    print(f"  SVG: {shape.accept(svg_export)}")
    print(f"  JSON: {shape.accept(json_serial)}")
```

---

## Сравнительная таблица паттернов

| Категория      | Паттерн                 | Основная идея                        | Когда использовать                      |
| -------------- | ----------------------- | ------------------------------------ | --------------------------------------- |
| **Creational** | Singleton               | Один экземпляр                       | Логгеры, конфиги                        |
|                | Factory Method          | Делегирование создания               | Когда тип неизвестен заранее            |
|                | Abstract Factory        | Семейства объектов                   | Кросс-платформенный UI                  |
|                | Builder                 | Пошаговое построение                 | Сложные объекты с множеством параметров |
|                | Prototype               | Клонирование                         | Дорогое создание объектов               |
| **Structural** | Adapter                 | Преобразование интерфейса            | Интеграция legacy кода                  |
|                | Bridge                  | Разделение абстракции                | Независимое развитие иерархий           |
|                | Composite               | Древовидные структуры                | Файловые системы, UI                    |
|                | Decorator               | Динамическое расширение              | Middleware, потоки                      |
|                | Facade                  | Упрощение интерфейса                 | Сложные подсистемы                      |
|                | Flyweight               | Разделение состояния                 | Много похожих объектов                  |
|                | Proxy                   | Контроль доступа                     | Кеширование, безопасность               |
| **Behavioral** | Chain of Responsibility | Цепочка обработчиков                 | Middleware, валидация                   |
|                | Command                 | Инкапсуляция действия                | Undo/redo, очереди                      |
|                | Iterator                | Последовательный доступ              | Обход коллекций                         |
|                | Mediator                | Централизация связей                 | Диалоги, чаты                           |
|                | Memento                 | Сохранение состояния                 | Undo, снимки                            |
|                | Observer                | Подписка на события                  | Event-driven системы                    |
|                | State                   | Поведение на основе состояния        | Конечные автоматы                       |
|                | Strategy                | Взаимозаменяемые алгоритмы           | Выбор алгоритма в runtime               |
|                | Template Method         | Скелет алгоритма                     | Фреймворки, ETL                         |
|                | Visitor                 | Новые операции без изменения классов | Компиляторы, сериализация               |

---

## Best Practices

1. **Не злоупотребляйте паттернами** — используйте только когда есть реальная необходимость
2. **Знайте альтернативы** — иногда простое решение лучше паттерна
3. **Комбинируйте паттерны** — многие паттерны хорошо работают вместе
4. **Следуйте SOLID** — паттерны часто реализуют принципы SOLID
5. **Документируйте намерения** — объясняйте, почему выбран конкретный паттерн

## Типичные ошибки

1. **Over-engineering** — применение паттернов там, где они не нужны
2. **Неправильный выбор** — использование паттерна не по назначению
3. **Игнорирование контекста** — паттерн должен соответствовать задаче
4. **Жёсткая привязка к реализации** — паттерны должны сохранять гибкость

---

## Дополнительные ресурсы

- [Refactoring Guru - Design Patterns](https://refactoring.guru/design-patterns)
- [Design Patterns: Elements of Reusable Object-Oriented Software (GoF Book)](https://www.amazon.com/Design-Patterns-Elements-Reusable-Object-Oriented/dp/0201633612)
- [Head First Design Patterns](https://www.amazon.com/Head-First-Design-Patterns-Brain-Friendly/dp/0596007124)
