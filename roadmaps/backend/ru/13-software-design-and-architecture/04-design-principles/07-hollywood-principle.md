# Hollywood Principle — Голливудский принцип

## Определение

**Hollywood Principle** (Голливудский принцип) гласит:

> "Don't call us, we'll call you"
> "Не звоните нам, мы сами вам позвоним"

Этот принцип описывает инверсию управления (Inversion of Control, IoC), когда высокоуровневые компоненты определяют, когда и как вызывать низкоуровневые, а не наоборот.

---

## Происхождение названия

Название происходит от фразы, которую часто слышат актёры на кастингах в Голливуде. Вместо того чтобы актёры постоянно звонили студии с вопросом "Вы меня взяли?", студия сама свяжется с ними, когда будет нужно.

В программировании это означает: **компонент не должен сам решать, когда его вызывать — фреймворк или родительский компонент сделает это за него**.

---

## Инверсия управления (IoC)

### Традиционный поток управления

```python
# Код приложения вызывает библиотеку
class Application:
    def run(self):
        data = self.read_input()          # Мы контролируем
        processed = self.process(data)     # Мы контролируем
        self.write_output(processed)       # Мы контролируем
```

### Инвертированный поток управления

```python
# Фреймворк вызывает код приложения
class MyHandler(FrameworkHandler):
    def on_request(self, request):
        # Фреймворк решает, когда вызвать этот метод
        return self.process(request)


# Регистрируем наш обработчик
framework.register_handler("/api/data", MyHandler())
# Фреймворк сам вызовет on_request, когда придёт запрос
```

---

## Примеры применения

### 1. Callback-функции

**Без Hollywood Principle:**

```python
class DataProcessor:
    def process(self, data):
        result = self.transform(data)

        # Процессор сам решает, куда отправить результат
        # Жёсткая связность!
        email_service = EmailService()
        email_service.send(result)

        database = Database()
        database.save(result)

        logger = Logger()
        logger.log(result)
```

**С Hollywood Principle (callbacks):**

```python
from typing import Callable, List


class DataProcessor:
    def __init__(self):
        self._callbacks: List[Callable] = []

    def register_callback(self, callback: Callable):
        """Регистрация колбэка — 'оставьте свой номер'"""
        self._callbacks.append(callback)

    def process(self, data):
        result = self.transform(data)

        # Процессор вызывает зарегистрированные колбэки
        # "Мы вам перезвоним"
        for callback in self._callbacks:
            callback(result)


# Использование
processor = DataProcessor()

# Регистрируем "актёров" (обработчики)
processor.register_callback(lambda r: email_service.send(r))
processor.register_callback(lambda r: database.save(r))
processor.register_callback(lambda r: logger.log(r))

processor.process(data)
```

---

### 2. Template Method Pattern

**Паттерн "Шаблонный метод" — классический пример Hollywood Principle:**

```python
from abc import ABC, abstractmethod


class ReportGenerator(ABC):
    """Базовый класс определяет алгоритм и вызывает методы подклассов"""

    def generate(self, data) -> str:
        """Шаблонный метод — определяет структуру алгоритма"""
        # Базовый класс контролирует поток выполнения
        header = self.create_header()
        body = self.create_body(data)
        footer = self.create_footer()

        return self.format_report(header, body, footer)

    @abstractmethod
    def create_header(self) -> str:
        """Подкласс реализует — базовый класс вызывает"""
        pass

    @abstractmethod
    def create_body(self, data) -> str:
        """Подкласс реализует — базовый класс вызывает"""
        pass

    @abstractmethod
    def create_footer(self) -> str:
        """Подкласс реализует — базовый класс вызывает"""
        pass

    def format_report(self, header: str, body: str, footer: str) -> str:
        """Может быть переопределён при необходимости"""
        return f"{header}\n{body}\n{footer}"


class HTMLReportGenerator(ReportGenerator):
    """Подкласс не вызывает методы — он предоставляет реализацию"""

    def create_header(self) -> str:
        return "<html><head><title>Report</title></head><body>"

    def create_body(self, data) -> str:
        rows = "".join(f"<tr><td>{item}</td></tr>" for item in data)
        return f"<table>{rows}</table>"

    def create_footer(self) -> str:
        return "</body></html>"


class MarkdownReportGenerator(ReportGenerator):
    def create_header(self) -> str:
        return "# Report\n"

    def create_body(self, data) -> str:
        return "\n".join(f"- {item}" for item in data)

    def create_footer(self) -> str:
        return "\n---\nGenerated automatically"


# Подкласс не контролирует, когда вызываются его методы
html_report = HTMLReportGenerator().generate(["item1", "item2"])
md_report = MarkdownReportGenerator().generate(["item1", "item2"])
```

---

### 3. Event-Driven Architecture

```python
from typing import Dict, List, Callable
from dataclasses import dataclass


@dataclass
class Event:
    name: str
    data: dict


class EventBus:
    """Шина событий реализует Hollywood Principle"""

    def __init__(self):
        self._subscribers: Dict[str, List[Callable]] = {}

    def subscribe(self, event_name: str, handler: Callable):
        """Подписка — 'оставьте свой номер'"""
        if event_name not in self._subscribers:
            self._subscribers[event_name] = []
        self._subscribers[event_name].append(handler)

    def publish(self, event: Event):
        """Публикация — 'мы вам перезвоним'"""
        handlers = self._subscribers.get(event.name, [])
        for handler in handlers:
            handler(event)


# Обработчики (не контролируют, когда их вызовут)
class OrderNotificationHandler:
    def handle(self, event: Event):
        print(f"Sending notification for order {event.data['order_id']}")


class InventoryHandler:
    def handle(self, event: Event):
        print(f"Updating inventory for order {event.data['order_id']}")


class AnalyticsHandler:
    def handle(self, event: Event):
        print(f"Recording analytics for order {event.data['order_id']}")


# Настройка
event_bus = EventBus()
event_bus.subscribe("order_created", OrderNotificationHandler().handle)
event_bus.subscribe("order_created", InventoryHandler().handle)
event_bus.subscribe("order_created", AnalyticsHandler().handle)

# Когда происходит событие — шина вызывает обработчики
event_bus.publish(Event("order_created", {"order_id": 123}))
```

---

### 4. Dependency Injection Container

```python
from typing import Dict, Type, Callable


class Container:
    """DI-контейнер управляет жизненным циклом зависимостей"""

    def __init__(self):
        self._factories: Dict[Type, Callable] = {}
        self._singletons: Dict[Type, object] = {}

    def register(self, interface: Type, factory: Callable, singleton: bool = False):
        """Регистрация — 'вот как меня создать'"""
        self._factories[interface] = (factory, singleton)

    def resolve(self, interface: Type):
        """Контейнер решает, когда и как создать объект"""
        if interface in self._singletons:
            return self._singletons[interface]

        factory, is_singleton = self._factories[interface]
        instance = factory(self)  # Контейнер вызывает фабрику

        if is_singleton:
            self._singletons[interface] = instance

        return instance


# Определение сервисов
class Logger:
    def log(self, message: str):
        print(f"[LOG] {message}")


class Database:
    def __init__(self, logger: Logger):
        self.logger = logger

    def query(self, sql: str):
        self.logger.log(f"Executing: {sql}")


class UserService:
    def __init__(self, db: Database, logger: Logger):
        self.db = db
        self.logger = logger

    def get_user(self, user_id: int):
        self.logger.log(f"Getting user {user_id}")
        return self.db.query(f"SELECT * FROM users WHERE id = {user_id}")


# Настройка контейнера
container = Container()
container.register(Logger, lambda c: Logger(), singleton=True)
container.register(Database, lambda c: Database(c.resolve(Logger)))
container.register(UserService, lambda c: UserService(
    c.resolve(Database),
    c.resolve(Logger)
))

# Контейнер сам создаёт граф зависимостей
user_service = container.resolve(UserService)
user_service.get_user(1)
```

---

### 5. Web Framework (Flask/FastAPI стиль)

```python
class SimpleWebFramework:
    """Простой веб-фреймворк демонстрирует Hollywood Principle"""

    def __init__(self):
        self._routes: Dict[str, Callable] = {}
        self._middleware: List[Callable] = []

    def route(self, path: str):
        """Декоратор для регистрации обработчика"""
        def decorator(handler: Callable):
            self._routes[path] = handler
            return handler
        return decorator

    def middleware(self, func: Callable):
        """Регистрация middleware"""
        self._middleware.append(func)
        return func

    def handle_request(self, path: str, request: dict) -> dict:
        """Фреймворк вызывает обработчики — не наоборот"""
        # Сначала middleware
        for mw in self._middleware:
            request = mw(request)

        # Затем обработчик маршрута
        handler = self._routes.get(path)
        if handler:
            return handler(request)
        return {"error": "Not found"}


# Использование
app = SimpleWebFramework()


@app.middleware
def logging_middleware(request: dict) -> dict:
    """Middleware не знает, когда его вызовут"""
    print(f"Request: {request}")
    return request


@app.route("/users")
def get_users(request: dict) -> dict:
    """Обработчик не контролирует, когда его вызовут"""
    return {"users": ["Alice", "Bob"]}


@app.route("/orders")
def get_orders(request: dict) -> dict:
    return {"orders": [1, 2, 3]}


# Фреймворк решает, какой обработчик вызвать
response = app.handle_request("/users", {"method": "GET"})
```

---

## Сравнение подходов

### Без Hollywood Principle

```python
class Client:
    def do_work(self):
        # Клиент контролирует всё
        service = ServiceA()
        result = service.process()

        if result.success:
            ServiceB().handle(result)
        else:
            ServiceC().log_error(result)
```

### С Hollywood Principle

```python
class Framework:
    def __init__(self):
        self.services = []

    def register(self, service):
        self.services.append(service)

    def run(self, data):
        # Фреймворк контролирует поток
        for service in self.services:
            data = service.process(data)
        return data


class MyService:
    def process(self, data):
        # Сервис только обрабатывает — не решает, когда
        return transform(data)


framework = Framework()
framework.register(MyService())
framework.run(data)
```

---

## Преимущества

1. **Слабая связность** — компоненты не знают друг о друге напрямую
2. **Расширяемость** — легко добавить новые обработчики
3. **Тестируемость** — можно тестировать компоненты изолированно
4. **Переиспользование** — компоненты не привязаны к конкретному контексту
5. **Разделение ответственности** — управление в одном месте

## Недостатки

1. **Сложность отладки** — поток управления неочевиден
2. **Callback hell** — при злоупотреблении колбэками
3. **Неявные зависимости** — сложно понять, что от чего зависит
4. **Накладные расходы** — дополнительные абстракции

---

## Best Practices

1. **Используйте фреймворки** — они реализуют IoC из коробки
2. **Документируйте контракты** — какие методы будут вызваны и когда
3. **Избегайте глубокой вложенности** — callback hell усложняет код
4. **Комбинируйте с DI** — Dependency Injection хорошо сочетается
5. **Используйте события для связи** — Event-Driven Architecture

## Типичные ошибки

1. **Слишком много уровней абстракции** — код становится нечитаемым
2. **Нарушение в критичных местах** — иногда прямой вызов проще
3. **Игнорирование порядка вызовов** — важно документировать
4. **Циклические зависимости** — приводят к проблемам

---

## Связь с другими принципами

| Концепция | Связь с Hollywood Principle |
|-----------|----------------------------|
| SOLID (DIP) | Инверсия зависимостей — частный случай IoC |
| Strategy Pattern | Алгоритм "вызывается", а не "вызывает" |
| Observer Pattern | Наблюдатели не контролируют, когда их уведомят |
| Plugin Architecture | Плагины регистрируются — фреймворк вызывает |

---

## Резюме

Hollywood Principle учит нас:
- Передавать контроль потока выполнения фреймворку или родительскому компоненту
- Создавать компоненты, которые **предоставляют** поведение, а не **контролируют** его
- Использовать колбэки, события и шаблонные методы для инверсии управления
- Строить расширяемые системы с слабой связностью
