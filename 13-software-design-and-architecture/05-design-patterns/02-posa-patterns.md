# PoSA Patterns (Pattern-Oriented Software Architecture)

## Введение

**PoSA (Pattern-Oriented Software Architecture)** — это серия книг, описывающая паттерны программной архитектуры на разных уровнях абстракции. В отличие от GoF-паттернов, которые фокусируются на уровне классов и объектов, PoSA-паттерны охватывают более широкий спектр — от архитектурных решений до низкоуровневых идиом.

Основные книги серии:
- **PoSA Volume 1** (1996) — A System of Patterns
- **PoSA Volume 2** (2000) — Patterns for Concurrent and Networked Objects
- **PoSA Volume 3** (2004) — Patterns for Resource Management
- **PoSA Volume 4** (2007) — A Pattern Language for Distributed Computing
- **PoSA Volume 5** (2007) — Pattern and Pattern Languages

## Уровни паттернов PoSA

PoSA классифицирует паттерны по трём уровням:

| Уровень | Описание | Примеры |
|---------|----------|---------|
| **Архитектурные паттерны** | Определяют структуру всей системы | Layers, MVC, Microkernel, Pipes and Filters |
| **Паттерны проектирования** | Уточняют подсистемы и компоненты | Whole-Part, Master-Slave, Proxy, Command Processor |
| **Идиомы** | Низкоуровневые паттерны, специфичные для языка | Counted Pointer, Smart Pointer, RAII |

---

## Архитектурные паттерны

Архитектурные паттерны определяют фундаментальную организацию системы, описывая:
- Набор предопределённых подсистем
- Их обязанности
- Правила взаимодействия между ними

### 1. Layers (Слои)

**Назначение:** Организует систему в виде иерархии слоёв, где каждый слой предоставляет сервисы слою выше и использует сервисы слоя ниже.

**Когда использовать:**
- Когда система имеет различные уровни абстракции
- Когда нужна независимая разработка и тестирование слоёв
- Когда нужна возможность замены слоя без влияния на другие

**Структура:**
```
┌─────────────────────────────────┐
│     Presentation Layer          │  ← UI, API контроллеры
├─────────────────────────────────┤
│     Application Layer           │  ← Бизнес-логика, сервисы
├─────────────────────────────────┤
│     Domain Layer                │  ← Доменные модели, правила
├─────────────────────────────────┤
│     Infrastructure Layer        │  ← БД, внешние сервисы
└─────────────────────────────────┘
```

```python
# Пример многослойной архитектуры

# ========== Domain Layer ==========
from dataclasses import dataclass
from abc import ABC, abstractmethod
from typing import Optional, List
from datetime import datetime

@dataclass
class User:
    id: Optional[int]
    email: str
    name: str
    created_at: datetime = None

    def __post_init__(self):
        if self.created_at is None:
            self.created_at = datetime.now()

class UserRepository(ABC):
    """Абстракция репозитория в Domain Layer"""
    @abstractmethod
    def save(self, user: User) -> User:
        pass

    @abstractmethod
    def find_by_id(self, user_id: int) -> Optional[User]:
        pass

    @abstractmethod
    def find_by_email(self, email: str) -> Optional[User]:
        pass

# ========== Application Layer ==========
class UserAlreadyExistsError(Exception):
    pass

class UserService:
    """Сервис приложения - координирует бизнес-операции"""

    def __init__(self, user_repository: UserRepository):
        self._repository = user_repository

    def register_user(self, email: str, name: str) -> User:
        # Бизнес-правило: email должен быть уникальным
        existing = self._repository.find_by_email(email)
        if existing:
            raise UserAlreadyExistsError(f"User with email {email} already exists")

        user = User(id=None, email=email, name=name)
        return self._repository.save(user)

    def get_user(self, user_id: int) -> Optional[User]:
        return self._repository.find_by_id(user_id)

# ========== Infrastructure Layer ==========
class InMemoryUserRepository(UserRepository):
    """Конкретная реализация репозитория"""

    def __init__(self):
        self._users: dict[int, User] = {}
        self._next_id = 1

    def save(self, user: User) -> User:
        if user.id is None:
            user.id = self._next_id
            self._next_id += 1
        self._users[user.id] = user
        return user

    def find_by_id(self, user_id: int) -> Optional[User]:
        return self._users.get(user_id)

    def find_by_email(self, email: str) -> Optional[User]:
        for user in self._users.values():
            if user.email == email:
                return user
        return None

# ========== Presentation Layer ==========
from dataclasses import asdict
import json

class UserController:
    """REST API контроллер"""

    def __init__(self, user_service: UserService):
        self._service = user_service

    def register(self, request_data: dict) -> dict:
        try:
            user = self._service.register_user(
                email=request_data["email"],
                name=request_data["name"]
            )
            return {"status": "success", "user": asdict(user)}
        except UserAlreadyExistsError as e:
            return {"status": "error", "message": str(e)}

    def get_user(self, user_id: int) -> dict:
        user = self._service.get_user(user_id)
        if user:
            return {"status": "success", "user": asdict(user)}
        return {"status": "error", "message": "User not found"}

# ========== Composition Root ==========
# Собираем приложение (Dependency Injection)
repository = InMemoryUserRepository()
service = UserService(repository)
controller = UserController(service)

# Использование
result = controller.register({"email": "john@example.com", "name": "John"})
print(json.dumps(result, indent=2, default=str))
```

**Преимущества:**
- Чёткое разделение ответственности
- Независимое тестирование слоёв
- Возможность замены реализации слоя

**Недостатки:**
- Накладные расходы на передачу данных между слоями
- Может привести к "анемичной модели"
- Сложность навигации по коду

---

### 2. Pipes and Filters (Каналы и фильтры)

**Назначение:** Предоставляет структуру для систем, обрабатывающих поток данных. Каждый шаг обработки инкапсулирован в компонент-фильтр, а данные передаются через каналы между фильтрами.

**Когда использовать:**
- ETL-процессы
- Компиляторы (лексический анализ → парсинг → оптимизация → генерация кода)
- Обработка изображений и медиа
- Message processing pipelines

```python
from abc import ABC, abstractmethod
from typing import TypeVar, Generic, Iterator, Callable, List
from dataclasses import dataclass

T = TypeVar('T')
U = TypeVar('U')

# ========== Filter Interface ==========
class Filter(ABC, Generic[T, U]):
    @abstractmethod
    def process(self, data: T) -> U:
        pass

# ========== Concrete Filters ==========
@dataclass
class LogEntry:
    timestamp: str
    level: str
    message: str
    source: str

class ParseFilter(Filter[str, LogEntry]):
    """Парсит строку лога в структуру"""

    def process(self, data: str) -> LogEntry:
        parts = data.split(" | ")
        return LogEntry(
            timestamp=parts[0],
            level=parts[1],
            message=parts[2],
            source=parts[3] if len(parts) > 3 else "unknown"
        )

class FilterByLevel(Filter[LogEntry, LogEntry | None]):
    """Фильтрует по уровню логирования"""

    def __init__(self, min_level: str):
        self._levels = {"DEBUG": 0, "INFO": 1, "WARNING": 2, "ERROR": 3, "CRITICAL": 4}
        self._min_level = self._levels.get(min_level, 0)

    def process(self, data: LogEntry) -> LogEntry | None:
        if self._levels.get(data.level, 0) >= self._min_level:
            return data
        return None

class EnrichFilter(Filter[LogEntry, LogEntry]):
    """Обогащает данные дополнительной информацией"""

    def __init__(self, metadata: dict):
        self._metadata = metadata

    def process(self, data: LogEntry) -> LogEntry:
        data.message = f"[{self._metadata.get('env', 'unknown')}] {data.message}"
        return data

class FormatFilter(Filter[LogEntry, str]):
    """Форматирует для вывода"""

    def process(self, data: LogEntry) -> str:
        return f"[{data.timestamp}] {data.level}: {data.message} (from: {data.source})"

# ========== Pipeline ==========
class Pipeline:
    """Управляет последовательностью фильтров"""

    def __init__(self):
        self._filters: List[Filter] = []

    def add_filter(self, filter: Filter) -> "Pipeline":
        self._filters.append(filter)
        return self

    def process(self, data):
        result = data
        for filter in self._filters:
            if result is None:
                return None
            result = filter.process(result)
        return result

    def process_stream(self, data_stream: Iterator) -> Iterator:
        for item in data_stream:
            result = self.process(item)
            if result is not None:
                yield result

# ========== Usage ==========
# Исходные данные
log_lines = [
    "2024-01-15 10:00:00 | INFO | User logged in | auth-service",
    "2024-01-15 10:00:01 | DEBUG | Cache hit | cache-service",
    "2024-01-15 10:00:02 | ERROR | Connection timeout | db-service",
    "2024-01-15 10:00:03 | WARNING | High memory usage | monitoring",
    "2024-01-15 10:00:04 | CRITICAL | Database down | db-service",
]

# Строим pipeline
pipeline = (Pipeline()
    .add_filter(ParseFilter())
    .add_filter(FilterByLevel("WARNING"))
    .add_filter(EnrichFilter({"env": "production"}))
    .add_filter(FormatFilter()))

# Обрабатываем поток
print("=== Filtered Logs (WARNING+) ===")
for result in pipeline.process_stream(iter(log_lines)):
    print(result)

# Output:
# [2024-01-15 10:00:02] ERROR: [production] Connection timeout (from: db-service)
# [2024-01-15 10:00:03] WARNING: [production] High memory usage (from: monitoring)
# [2024-01-15 10:00:04] CRITICAL: [production] Database down (from: db-service)
```

**Функциональный подход:**
```python
from functools import reduce
from typing import Callable, TypeVar, Iterable

T = TypeVar('T')

def compose(*functions: Callable) -> Callable:
    """Композиция функций справа налево"""
    return reduce(lambda f, g: lambda x: f(g(x)), functions, lambda x: x)

def pipe(*functions: Callable) -> Callable:
    """Pipeline слева направо"""
    return reduce(lambda f, g: lambda x: g(f(x)), functions, lambda x: x)

# Фильтры как функции
def parse_log(line: str) -> dict:
    parts = line.split(" | ")
    return {"timestamp": parts[0], "level": parts[1], "message": parts[2]}

def filter_errors(log: dict) -> dict | None:
    return log if log["level"] in ("ERROR", "CRITICAL") else None

def format_output(log: dict | None) -> str | None:
    if log is None:
        return None
    return f"[{log['level']}] {log['message']}"

# Создаём pipeline
process_log = pipe(parse_log, filter_errors, format_output)

# Использование
for line in log_lines:
    result = process_log(line)
    if result:
        print(result)
```

---

### 3. Microkernel (Микроядро)

**Назначение:** Разделяет минимальное функциональное ядро от расширенной функциональности и частей, специфичных для клиента.

**Когда использовать:**
- Продукты с плагинами (IDE, браузеры)
- Системы с изменяемыми бизнес-правилами
- Приложения, требующие кастомизации

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, List, Callable
from dataclasses import dataclass

# ========== Plugin Interface ==========
class Plugin(ABC):
    @property
    @abstractmethod
    def name(self) -> str:
        pass

    @abstractmethod
    def initialize(self, kernel: "Microkernel"):
        pass

    @abstractmethod
    def execute(self, context: Dict[str, Any]) -> Dict[str, Any]:
        pass

# ========== Microkernel Core ==========
class Microkernel:
    """Минимальное ядро системы"""

    def __init__(self):
        self._plugins: Dict[str, Plugin] = {}
        self._hooks: Dict[str, List[Callable]] = {}
        self._config: Dict[str, Any] = {}

    # Core functionality
    def register_plugin(self, plugin: Plugin):
        """Регистрирует плагин в ядре"""
        self._plugins[plugin.name] = plugin
        plugin.initialize(self)
        print(f"Plugin '{plugin.name}' registered")

    def unregister_plugin(self, name: str):
        """Удаляет плагин"""
        if name in self._plugins:
            del self._plugins[name]
            print(f"Plugin '{name}' unregistered")

    def get_plugin(self, name: str) -> Plugin | None:
        return self._plugins.get(name)

    # Hook system for plugin communication
    def register_hook(self, hook_name: str, callback: Callable):
        """Регистрирует callback на hook"""
        if hook_name not in self._hooks:
            self._hooks[hook_name] = []
        self._hooks[hook_name].append(callback)

    def trigger_hook(self, hook_name: str, data: Any = None) -> List[Any]:
        """Вызывает все callbacks для hook"""
        results = []
        for callback in self._hooks.get(hook_name, []):
            result = callback(data)
            if result is not None:
                results.append(result)
        return results

    # Configuration
    def set_config(self, key: str, value: Any):
        self._config[key] = value

    def get_config(self, key: str, default: Any = None) -> Any:
        return self._config.get(key, default)

    # Request processing
    def process(self, request: Dict[str, Any]) -> Dict[str, Any]:
        """Обрабатывает запрос через зарегистрированные плагины"""
        context = {"request": request, "response": {}}

        # Pre-processing hook
        self.trigger_hook("pre_process", context)

        # Process through plugins
        plugin_name = request.get("plugin")
        if plugin_name and plugin_name in self._plugins:
            context = self._plugins[plugin_name].execute(context)

        # Post-processing hook
        self.trigger_hook("post_process", context)

        return context["response"]

# ========== Concrete Plugins ==========
class AuthPlugin(Plugin):
    @property
    def name(self) -> str:
        return "auth"

    def initialize(self, kernel: Microkernel):
        kernel.set_config("auth.enabled", True)
        kernel.register_hook("pre_process", self._check_auth)

    def _check_auth(self, context: Dict[str, Any]):
        request = context.get("request", {})
        token = request.get("token")
        if token == "valid-token":
            context["user"] = {"id": 1, "name": "John"}
        else:
            context["response"]["error"] = "Unauthorized"

    def execute(self, context: Dict[str, Any]) -> Dict[str, Any]:
        user = context.get("user")
        if user:
            context["response"]["user"] = user
        return context

class LoggingPlugin(Plugin):
    @property
    def name(self) -> str:
        return "logging"

    def initialize(self, kernel: Microkernel):
        kernel.register_hook("pre_process", self._log_request)
        kernel.register_hook("post_process", self._log_response)

    def _log_request(self, context: Dict[str, Any]):
        print(f"[LOG] Request: {context.get('request')}")

    def _log_response(self, context: Dict[str, Any]):
        print(f"[LOG] Response: {context.get('response')}")

    def execute(self, context: Dict[str, Any]) -> Dict[str, Any]:
        return context

class ValidationPlugin(Plugin):
    @property
    def name(self) -> str:
        return "validation"

    def initialize(self, kernel: Microkernel):
        self._rules: Dict[str, Callable] = {}

    def add_rule(self, field: str, rule: Callable[[Any], bool]):
        self._rules[field] = rule

    def execute(self, context: Dict[str, Any]) -> Dict[str, Any]:
        request = context.get("request", {})
        errors = []

        for field, rule in self._rules.items():
            value = request.get(field)
            if not rule(value):
                errors.append(f"Validation failed for '{field}'")

        if errors:
            context["response"]["errors"] = errors
        else:
            context["response"]["valid"] = True

        return context

# ========== Usage ==========
# Создаём ядро
kernel = Microkernel()

# Регистрируем плагины
kernel.register_plugin(LoggingPlugin())
kernel.register_plugin(AuthPlugin())

validation = ValidationPlugin()
validation.add_rule("email", lambda x: x and "@" in x)
validation.add_rule("age", lambda x: x and x >= 18)
kernel.register_plugin(validation)

# Обрабатываем запросы
print("\n=== Request 1: Valid ===")
response = kernel.process({
    "plugin": "validation",
    "token": "valid-token",
    "email": "john@example.com",
    "age": 25
})
print(f"Response: {response}")

print("\n=== Request 2: Invalid ===")
response = kernel.process({
    "plugin": "validation",
    "token": "valid-token",
    "email": "invalid-email",
    "age": 15
})
print(f"Response: {response}")
```

---

### 4. Broker (Брокер)

**Назначение:** Структурирует распределённые системы с разделёнными компонентами, которые взаимодействуют через удалённые вызовы сервисов.

**Когда использовать:**
- Распределённые системы
- Микросервисная архитектура
- Системы с location transparency

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Callable
from dataclasses import dataclass
import json
import uuid

# ========== Message Types ==========
@dataclass
class Message:
    id: str
    service: str
    method: str
    payload: Dict[str, Any]
    reply_to: str = None

    @staticmethod
    def create(service: str, method: str, payload: Dict[str, Any]) -> "Message":
        return Message(
            id=str(uuid.uuid4()),
            service=service,
            method=method,
            payload=payload
        )

@dataclass
class Response:
    message_id: str
    success: bool
    data: Any
    error: str = None

# ========== Service Interface ==========
class Service(ABC):
    @property
    @abstractmethod
    def name(self) -> str:
        pass

    @abstractmethod
    def handle(self, method: str, payload: Dict[str, Any]) -> Any:
        pass

# ========== Broker ==========
class Broker:
    """Центральный посредник для маршрутизации сообщений"""

    def __init__(self):
        self._services: Dict[str, Service] = {}
        self._pending_requests: Dict[str, Callable] = {}

    def register_service(self, service: Service):
        """Регистрирует сервис в брокере"""
        self._services[service.name] = service
        print(f"[Broker] Service '{service.name}' registered")

    def unregister_service(self, name: str):
        if name in self._services:
            del self._services[name]
            print(f"[Broker] Service '{name}' unregistered")

    def send(self, message: Message) -> Response:
        """Синхронная отправка сообщения"""
        print(f"[Broker] Routing message to '{message.service}.{message.method}'")

        if message.service not in self._services:
            return Response(
                message_id=message.id,
                success=False,
                data=None,
                error=f"Service '{message.service}' not found"
            )

        service = self._services[message.service]
        try:
            result = service.handle(message.method, message.payload)
            return Response(
                message_id=message.id,
                success=True,
                data=result
            )
        except Exception as e:
            return Response(
                message_id=message.id,
                success=False,
                data=None,
                error=str(e)
            )

    def send_async(self, message: Message, callback: Callable[[Response], None]):
        """Асинхронная отправка с callback"""
        response = self.send(message)
        callback(response)

# ========== Client Proxy ==========
class ServiceProxy:
    """Прокси для прозрачного вызова удалённых сервисов"""

    def __init__(self, broker: Broker, service_name: str):
        self._broker = broker
        self._service_name = service_name

    def call(self, method: str, **kwargs) -> Any:
        message = Message.create(self._service_name, method, kwargs)
        response = self._broker.send(message)

        if not response.success:
            raise Exception(response.error)

        return response.data

# ========== Concrete Services ==========
class UserService(Service):
    @property
    def name(self) -> str:
        return "users"

    def __init__(self):
        self._users = {
            1: {"id": 1, "name": "Alice", "email": "alice@example.com"},
            2: {"id": 2, "name": "Bob", "email": "bob@example.com"},
        }

    def handle(self, method: str, payload: Dict[str, Any]) -> Any:
        if method == "get":
            user_id = payload.get("id")
            return self._users.get(user_id)

        elif method == "create":
            new_id = max(self._users.keys()) + 1
            user = {"id": new_id, **payload}
            self._users[new_id] = user
            return user

        elif method == "list":
            return list(self._users.values())

        raise ValueError(f"Unknown method: {method}")

class OrderService(Service):
    @property
    def name(self) -> str:
        return "orders"

    def __init__(self, user_proxy: ServiceProxy):
        self._user_proxy = user_proxy
        self._orders = []

    def handle(self, method: str, payload: Dict[str, Any]) -> Any:
        if method == "create":
            user_id = payload.get("user_id")

            # Вызов другого сервиса через брокер
            user = self._user_proxy.call("get", id=user_id)
            if not user:
                raise ValueError(f"User {user_id} not found")

            order = {
                "id": len(self._orders) + 1,
                "user": user,
                "items": payload.get("items", []),
                "total": payload.get("total", 0)
            }
            self._orders.append(order)
            return order

        elif method == "list":
            return self._orders

        raise ValueError(f"Unknown method: {method}")

# ========== Usage ==========
# Создаём брокер
broker = Broker()

# Регистрируем сервисы
user_service = UserService()
broker.register_service(user_service)

# OrderService использует прокси для вызова UserService
user_proxy = ServiceProxy(broker, "users")
order_service = OrderService(user_proxy)
broker.register_service(order_service)

# Клиентский код
users_client = ServiceProxy(broker, "users")
orders_client = ServiceProxy(broker, "orders")

# Получаем пользователей
print("\n=== Users ===")
users = users_client.call("list")
print(json.dumps(users, indent=2))

# Создаём заказ
print("\n=== Create Order ===")
order = orders_client.call("create", user_id=1, items=["iPhone", "Case"], total=1099.99)
print(json.dumps(order, indent=2))
```

---

### 5. Model-View-Controller (MVC)

**Назначение:** Разделяет приложение на три взаимосвязанных компонента: модель (данные), представление (UI) и контроллер (логика взаимодействия).

```python
from abc import ABC, abstractmethod
from typing import List, Dict, Any, Callable
from dataclasses import dataclass, field

# ========== Observer Pattern для связи Model-View ==========
class Observer(ABC):
    @abstractmethod
    def update(self, data: Any):
        pass

class Observable:
    def __init__(self):
        self._observers: List[Observer] = []

    def attach(self, observer: Observer):
        self._observers.append(observer)

    def detach(self, observer: Observer):
        self._observers.remove(observer)

    def notify(self, data: Any = None):
        for observer in self._observers:
            observer.update(data)

# ========== Model ==========
@dataclass
class Task:
    id: int
    title: str
    completed: bool = False

class TaskModel(Observable):
    def __init__(self):
        super().__init__()
        self._tasks: Dict[int, Task] = {}
        self._next_id = 1

    def add_task(self, title: str) -> Task:
        task = Task(id=self._next_id, title=title)
        self._tasks[task.id] = task
        self._next_id += 1
        self.notify({"action": "added", "task": task})
        return task

    def toggle_task(self, task_id: int) -> Task | None:
        if task_id in self._tasks:
            task = self._tasks[task_id]
            task.completed = not task.completed
            self.notify({"action": "toggled", "task": task})
            return task
        return None

    def delete_task(self, task_id: int) -> bool:
        if task_id in self._tasks:
            task = self._tasks.pop(task_id)
            self.notify({"action": "deleted", "task": task})
            return True
        return False

    def get_all_tasks(self) -> List[Task]:
        return list(self._tasks.values())

    def get_pending_tasks(self) -> List[Task]:
        return [t for t in self._tasks.values() if not t.completed]

    def get_completed_tasks(self) -> List[Task]:
        return [t for t in self._tasks.values() if t.completed]

# ========== View ==========
class TaskView(Observer):
    """Консольное представление списка задач"""

    def __init__(self, model: TaskModel):
        self._model = model
        self._model.attach(self)

    def update(self, data: Any):
        """Вызывается при изменении модели"""
        action = data.get("action")
        task = data.get("task")
        print(f"\n[View Update] Action: {action}, Task: {task.title}")
        self.render()

    def render(self):
        """Отрисовка текущего состояния"""
        print("\n" + "=" * 40)
        print("📋 TODO List")
        print("=" * 40)

        tasks = self._model.get_all_tasks()
        if not tasks:
            print("  No tasks yet")
        else:
            for task in tasks:
                status = "✅" if task.completed else "⬜"
                print(f"  {status} [{task.id}] {task.title}")

        pending = len(self._model.get_pending_tasks())
        completed = len(self._model.get_completed_tasks())
        print(f"\nPending: {pending} | Completed: {completed}")
        print("=" * 40)

    def show_error(self, message: str):
        print(f"\n❌ Error: {message}")

    def show_message(self, message: str):
        print(f"\n✓ {message}")

# ========== Controller ==========
class TaskController:
    """Обрабатывает пользовательский ввод и координирует Model и View"""

    def __init__(self, model: TaskModel, view: TaskView):
        self._model = model
        self._view = view

    def add_task(self, title: str):
        if not title.strip():
            self._view.show_error("Task title cannot be empty")
            return

        task = self._model.add_task(title.strip())
        self._view.show_message(f"Task '{task.title}' added")

    def toggle_task(self, task_id: int):
        task = self._model.toggle_task(task_id)
        if task:
            status = "completed" if task.completed else "pending"
            self._view.show_message(f"Task '{task.title}' marked as {status}")
        else:
            self._view.show_error(f"Task with ID {task_id} not found")

    def delete_task(self, task_id: int):
        if self._model.delete_task(task_id):
            self._view.show_message(f"Task {task_id} deleted")
        else:
            self._view.show_error(f"Task with ID {task_id} not found")

    def show_all(self):
        self._view.render()

# ========== Usage ==========
# Создаём MVC
model = TaskModel()
view = TaskView(model)
controller = TaskController(model, view)

# Взаимодействие через контроллер
controller.add_task("Learn Python")
controller.add_task("Study Design Patterns")
controller.add_task("Build a project")

controller.toggle_task(1)  # Отмечаем первую задачу
controller.toggle_task(2)  # Отмечаем вторую

controller.delete_task(3)  # Удаляем третью

controller.show_all()
```

---

### 6. Blackboard (Классная доска)

**Назначение:** Паттерн для систем, где несколько специализированных подсистем собирают свои знания для построения возможного частичного или приближённого решения.

**Когда использовать:**
- Задачи без детерминированного решения
- Системы распознавания (речь, изображения)
- AI и экспертные системы
- Задачи планирования

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, List
from dataclasses import dataclass, field
from enum import Enum

class Confidence(Enum):
    LOW = 1
    MEDIUM = 2
    HIGH = 3
    CERTAIN = 4

@dataclass
class Hypothesis:
    """Гипотеза на доске"""
    source: str
    category: str
    value: Any
    confidence: Confidence
    supporting_evidence: List[str] = field(default_factory=list)

class Blackboard:
    """Общее пространство данных"""

    def __init__(self):
        self._hypotheses: Dict[str, List[Hypothesis]] = {}
        self._raw_data: Dict[str, Any] = {}

    def set_data(self, key: str, value: Any):
        self._raw_data[key] = value

    def get_data(self, key: str) -> Any:
        return self._raw_data.get(key)

    def add_hypothesis(self, category: str, hypothesis: Hypothesis):
        if category not in self._hypotheses:
            self._hypotheses[category] = []
        self._hypotheses[category].append(hypothesis)
        print(f"[Blackboard] New hypothesis in '{category}': "
              f"{hypothesis.value} (confidence: {hypothesis.confidence.name})")

    def get_hypotheses(self, category: str) -> List[Hypothesis]:
        return self._hypotheses.get(category, [])

    def get_best_hypothesis(self, category: str) -> Hypothesis | None:
        hypotheses = self.get_hypotheses(category)
        if not hypotheses:
            return None
        return max(hypotheses, key=lambda h: h.confidence.value)

class KnowledgeSource(ABC):
    """Базовый класс для источников знаний"""

    def __init__(self, name: str):
        self.name = name

    @abstractmethod
    def can_contribute(self, blackboard: Blackboard) -> bool:
        """Проверяет, может ли источник внести вклад"""
        pass

    @abstractmethod
    def contribute(self, blackboard: Blackboard):
        """Вносит знания на доску"""
        pass

class Controller:
    """Управляет источниками знаний"""

    def __init__(self, blackboard: Blackboard):
        self._blackboard = blackboard
        self._sources: List[KnowledgeSource] = []

    def register_source(self, source: KnowledgeSource):
        self._sources.append(source)
        print(f"[Controller] Registered: {source.name}")

    def run(self, max_iterations: int = 10):
        """Запускает цикл обработки"""
        print("\n[Controller] Starting inference cycle...")

        for i in range(max_iterations):
            print(f"\n--- Iteration {i + 1} ---")
            contributed = False

            for source in self._sources:
                if source.can_contribute(self._blackboard):
                    print(f"[Controller] Activating: {source.name}")
                    source.contribute(self._blackboard)
                    contributed = True

            if not contributed:
                print("[Controller] No more contributions, stopping")
                break

        print("\n[Controller] Inference complete")

# ========== Example: Text Classification ==========

class TokenizerSource(KnowledgeSource):
    def __init__(self):
        super().__init__("Tokenizer")

    def can_contribute(self, blackboard: Blackboard) -> bool:
        return (blackboard.get_data("text") is not None and
                not blackboard.get_data("tokens"))

    def contribute(self, blackboard: Blackboard):
        text = blackboard.get_data("text")
        tokens = text.lower().split()
        blackboard.set_data("tokens", tokens)
        print(f"  Tokens: {tokens}")

class KeywordDetector(KnowledgeSource):
    def __init__(self):
        super().__init__("KeywordDetector")
        self._categories = {
            "spam": ["free", "winner", "click", "urgent", "money"],
            "tech": ["software", "computer", "code", "programming", "python"],
            "greeting": ["hello", "hi", "hey", "welcome"],
        }

    def can_contribute(self, blackboard: Blackboard) -> bool:
        return (blackboard.get_data("tokens") is not None and
                not blackboard.get_hypotheses("category"))

    def contribute(self, blackboard: Blackboard):
        tokens = blackboard.get_data("tokens")

        for category, keywords in self._categories.items():
            matches = [t for t in tokens if t in keywords]
            if matches:
                confidence = (Confidence.HIGH if len(matches) >= 2
                             else Confidence.MEDIUM)
                blackboard.add_hypothesis("category", Hypothesis(
                    source=self.name,
                    category="category",
                    value=category,
                    confidence=confidence,
                    supporting_evidence=matches
                ))

class SentimentAnalyzer(KnowledgeSource):
    def __init__(self):
        super().__init__("SentimentAnalyzer")
        self._positive = ["good", "great", "excellent", "amazing", "love"]
        self._negative = ["bad", "terrible", "awful", "hate", "worst"]

    def can_contribute(self, blackboard: Blackboard) -> bool:
        return (blackboard.get_data("tokens") is not None and
                not blackboard.get_hypotheses("sentiment"))

    def contribute(self, blackboard: Blackboard):
        tokens = blackboard.get_data("tokens")

        positive_count = sum(1 for t in tokens if t in self._positive)
        negative_count = sum(1 for t in tokens if t in self._negative)

        if positive_count > negative_count:
            sentiment = "positive"
            confidence = Confidence.HIGH if positive_count >= 2 else Confidence.MEDIUM
        elif negative_count > positive_count:
            sentiment = "negative"
            confidence = Confidence.HIGH if negative_count >= 2 else Confidence.MEDIUM
        else:
            sentiment = "neutral"
            confidence = Confidence.LOW

        blackboard.add_hypothesis("sentiment", Hypothesis(
            source=self.name,
            category="sentiment",
            value=sentiment,
            confidence=confidence
        ))

class ClassificationResolver(KnowledgeSource):
    def __init__(self):
        super().__init__("ClassificationResolver")

    def can_contribute(self, blackboard: Blackboard) -> bool:
        return (blackboard.get_hypotheses("category") and
                not blackboard.get_data("final_classification"))

    def contribute(self, blackboard: Blackboard):
        best = blackboard.get_best_hypothesis("category")
        if best and best.confidence.value >= Confidence.MEDIUM.value:
            blackboard.set_data("final_classification", {
                "category": best.value,
                "confidence": best.confidence.name,
                "evidence": best.supporting_evidence
            })
            print(f"  Final classification: {best.value}")

# ========== Usage ==========
blackboard = Blackboard()
controller = Controller(blackboard)

# Регистрируем источники знаний
controller.register_source(TokenizerSource())
controller.register_source(KeywordDetector())
controller.register_source(SentimentAnalyzer())
controller.register_source(ClassificationResolver())

# Входные данные
blackboard.set_data("text", "Hello! This is great Python programming code")

# Запускаем анализ
controller.run()

# Результаты
print("\n=== Results ===")
print(f"Classification: {blackboard.get_data('final_classification')}")
print(f"Sentiment: {blackboard.get_best_hypothesis('sentiment')}")
```

---

## Паттерны проектирования PoSA

Эти паттерны находятся на среднем уровне между архитектурными паттернами и идиомами.

### 1. Whole-Part (Целое-Часть)

**Назначение:** Агрегирует компоненты (части) в семантическую единицу (целое). Целое инкапсулирует части и предоставляет интерфейс к ним.

```python
from abc import ABC, abstractmethod
from typing import List, Dict
from dataclasses import dataclass

@dataclass
class Coordinate:
    x: float
    y: float

class GraphicElement(ABC):
    """Part - часть"""
    @abstractmethod
    def draw(self) -> str:
        pass

    @abstractmethod
    def move(self, dx: float, dy: float):
        pass

    @abstractmethod
    def get_bounds(self) -> tuple[Coordinate, Coordinate]:
        pass

class Circle(GraphicElement):
    def __init__(self, center: Coordinate, radius: float):
        self.center = center
        self.radius = radius

    def draw(self) -> str:
        return f"Circle at ({self.center.x}, {self.center.y}) r={self.radius}"

    def move(self, dx: float, dy: float):
        self.center.x += dx
        self.center.y += dy

    def get_bounds(self) -> tuple[Coordinate, Coordinate]:
        return (
            Coordinate(self.center.x - self.radius, self.center.y - self.radius),
            Coordinate(self.center.x + self.radius, self.center.y + self.radius)
        )

class Rectangle(GraphicElement):
    def __init__(self, top_left: Coordinate, width: float, height: float):
        self.top_left = top_left
        self.width = width
        self.height = height

    def draw(self) -> str:
        return f"Rectangle at ({self.top_left.x}, {self.top_left.y}) {self.width}x{self.height}"

    def move(self, dx: float, dy: float):
        self.top_left.x += dx
        self.top_left.y += dy

    def get_bounds(self) -> tuple[Coordinate, Coordinate]:
        return (
            self.top_left,
            Coordinate(self.top_left.x + self.width, self.top_left.y + self.height)
        )

class Drawing(GraphicElement):
    """Whole - целое, агрегирующее части"""

    def __init__(self, name: str):
        self.name = name
        self._elements: List[GraphicElement] = []

    def add(self, element: GraphicElement):
        self._elements.append(element)

    def remove(self, element: GraphicElement):
        self._elements.remove(element)

    def draw(self) -> str:
        result = [f"Drawing '{self.name}':"]
        for elem in self._elements:
            result.append(f"  - {elem.draw()}")
        return "\n".join(result)

    def move(self, dx: float, dy: float):
        """Перемещает все части вместе"""
        for elem in self._elements:
            elem.move(dx, dy)

    def get_bounds(self) -> tuple[Coordinate, Coordinate]:
        """Вычисляет общие границы"""
        if not self._elements:
            return (Coordinate(0, 0), Coordinate(0, 0))

        min_x = min_y = float('inf')
        max_x = max_y = float('-inf')

        for elem in self._elements:
            top_left, bottom_right = elem.get_bounds()
            min_x = min(min_x, top_left.x)
            min_y = min(min_y, top_left.y)
            max_x = max(max_x, bottom_right.x)
            max_y = max(max_y, bottom_right.y)

        return (Coordinate(min_x, min_y), Coordinate(max_x, max_y))

# Использование
drawing = Drawing("My Diagram")
drawing.add(Circle(Coordinate(50, 50), 25))
drawing.add(Rectangle(Coordinate(100, 100), 60, 40))
drawing.add(Circle(Coordinate(150, 50), 15))

print(drawing.draw())
print(f"\nBounds: {drawing.get_bounds()}")

drawing.move(10, 10)
print("\nAfter moving:")
print(drawing.draw())
```

---

### 2. Master-Slave (Главный-Подчинённый)

**Назначение:** Главный компонент распределяет работу между идентичными подчинёнными и вычисляет конечный результат из их результатов.

```python
from abc import ABC, abstractmethod
from typing import List, TypeVar, Generic, Callable
from concurrent.futures import ThreadPoolExecutor, as_completed
from dataclasses import dataclass
import time
import random

T = TypeVar('T')  # Input type
R = TypeVar('R')  # Result type

class Slave(ABC, Generic[T, R]):
    """Подчинённый - выполняет часть работы"""

    @abstractmethod
    def process(self, task: T) -> R:
        pass

class Master(Generic[T, R]):
    """Главный - распределяет работу и агрегирует результаты"""

    def __init__(self, slaves: List[Slave[T, R]], combiner: Callable[[List[R]], R]):
        self._slaves = slaves
        self._combiner = combiner

    def execute(self, tasks: List[T]) -> R:
        """Распределяет задачи между подчинёнными"""
        results: List[R] = []

        # Простое round-robin распределение
        for i, task in enumerate(tasks):
            slave_idx = i % len(self._slaves)
            result = self._slaves[slave_idx].process(task)
            results.append(result)

        return self._combiner(results)

    def execute_parallel(self, tasks: List[T], max_workers: int = None) -> R:
        """Параллельное выполнение"""
        results: List[R] = []

        with ThreadPoolExecutor(max_workers=max_workers or len(self._slaves)) as executor:
            # Распределяем задачи
            future_to_task = {}
            for i, task in enumerate(tasks):
                slave_idx = i % len(self._slaves)
                future = executor.submit(self._slaves[slave_idx].process, task)
                future_to_task[future] = task

            # Собираем результаты
            for future in as_completed(future_to_task):
                try:
                    result = future.result()
                    results.append(result)
                except Exception as e:
                    print(f"Task failed: {e}")

        return self._combiner(results)

# ========== Example: Parallel Sum Calculation ==========

@dataclass
class SumTask:
    numbers: List[int]

class SumSlave(Slave[SumTask, int]):
    def __init__(self, name: str):
        self.name = name

    def process(self, task: SumTask) -> int:
        # Имитация работы
        time.sleep(random.uniform(0.1, 0.3))
        result = sum(task.numbers)
        print(f"[{self.name}] Processed {len(task.numbers)} numbers, sum = {result}")
        return result

def combine_sums(results: List[int]) -> int:
    return sum(results)

# Использование
slaves = [SumSlave(f"Worker-{i}") for i in range(4)]
master = Master(slaves, combine_sums)

# Создаём задачи - разбиваем большой список на части
all_numbers = list(range(1, 101))  # 1 to 100
chunk_size = 10
tasks = [SumTask(all_numbers[i:i + chunk_size])
         for i in range(0, len(all_numbers), chunk_size)]

print("=== Sequential Execution ===")
start = time.time()
result = master.execute(tasks)
print(f"Total sum: {result} (took {time.time() - start:.2f}s)")

print("\n=== Parallel Execution ===")
start = time.time()
result = master.execute_parallel(tasks)
print(f"Total sum: {result} (took {time.time() - start:.2f}s)")

# Проверка
print(f"\nExpected: {sum(range(1, 101))}")  # 5050
```

---

### 3. Reactor (Реактор)

**Назначение:** Демультиплексирует и диспетчеризирует запросы на сервисные обработчики. Позволяет событийно-ориентированным приложениям синхронно ожидать и обрабатывать события от одного или нескольких источников.

```python
from abc import ABC, abstractmethod
from typing import Dict, Callable, List, Any
from dataclasses import dataclass
from enum import Enum, auto
import selectors
import socket
import time

class EventType(Enum):
    READ = auto()
    WRITE = auto()
    TIMER = auto()
    SIGNAL = auto()

@dataclass
class Event:
    type: EventType
    source: Any
    data: Any = None

class EventHandler(ABC):
    """Обработчик событий"""

    @abstractmethod
    def handle(self, event: Event):
        pass

class Reactor:
    """Синхронный событийный демультиплексор"""

    def __init__(self):
        self._handlers: Dict[Any, Dict[EventType, EventHandler]] = {}
        self._running = False
        self._timers: List[tuple[float, Callable]] = []

    def register(self, source: Any, event_type: EventType, handler: EventHandler):
        """Регистрирует обработчик для источника и типа события"""
        if source not in self._handlers:
            self._handlers[source] = {}
        self._handlers[source][event_type] = handler
        print(f"[Reactor] Registered handler for {event_type.name} on {source}")

    def unregister(self, source: Any, event_type: EventType = None):
        """Удаляет обработчик"""
        if source in self._handlers:
            if event_type:
                self._handlers[source].pop(event_type, None)
            else:
                del self._handlers[source]

    def add_timer(self, delay: float, callback: Callable):
        """Добавляет таймер"""
        trigger_time = time.time() + delay
        self._timers.append((trigger_time, callback))
        self._timers.sort(key=lambda x: x[0])

    def dispatch(self, event: Event):
        """Диспетчеризирует событие обработчику"""
        handlers = self._handlers.get(event.source, {})
        handler = handlers.get(event.type)

        if handler:
            handler.handle(event)
        else:
            print(f"[Reactor] No handler for {event.type.name} on {event.source}")

    def run(self, timeout: float = None):
        """Основной цикл обработки событий"""
        self._running = True
        start_time = time.time()

        print("[Reactor] Starting event loop...")

        while self._running:
            # Проверяем таймауты
            now = time.time()
            if timeout and (now - start_time) > timeout:
                print("[Reactor] Timeout reached")
                break

            # Обрабатываем таймеры
            while self._timers and self._timers[0][0] <= now:
                _, callback = self._timers.pop(0)
                callback()

            # Симуляция ожидания событий
            time.sleep(0.01)

    def stop(self):
        """Останавливает цикл"""
        self._running = False
        print("[Reactor] Stopping...")

# ========== Example: Simple Event System ==========

class MessageHandler(EventHandler):
    def handle(self, event: Event):
        print(f"[MessageHandler] Received: {event.data}")

class ConnectionHandler(EventHandler):
    def __init__(self, reactor: Reactor):
        self._reactor = reactor

    def handle(self, event: Event):
        print(f"[ConnectionHandler] New connection from {event.source}")
        # Регистрируем обработчик сообщений для нового соединения
        self._reactor.register(event.source, EventType.READ, MessageHandler())

class TimerHandler:
    def __init__(self, message: str):
        self._message = message

    def __call__(self):
        print(f"[Timer] {self._message}")

# Использование
reactor = Reactor()

# Регистрируем обработчики
reactor.register("connection_listener", EventType.READ, ConnectionHandler(reactor))
reactor.register("channel_1", EventType.READ, MessageHandler())

# Добавляем таймеры
reactor.add_timer(0.5, TimerHandler("First timer fired!"))
reactor.add_timer(1.0, TimerHandler("Second timer fired!"))
reactor.add_timer(1.5, lambda: reactor.stop())

# Симулируем события
def simulate_events():
    time.sleep(0.2)
    reactor.dispatch(Event(EventType.READ, "channel_1", "Hello, World!"))
    time.sleep(0.3)
    reactor.dispatch(Event(EventType.READ, "connection_listener", {"addr": "192.168.1.1"}))
    time.sleep(0.2)
    reactor.dispatch(Event(EventType.READ, "channel_1", "Another message"))

import threading
threading.Thread(target=simulate_events, daemon=True).start()

# Запускаем реактор
reactor.run(timeout=2.0)
```

---

### 4. Proactor (Проактор)

**Назначение:** Позволяет событийно-ориентированным приложениям эффективно демультиплексировать и диспетчеризировать запросы, инициированные завершением асинхронных операций.

**Отличие от Reactor:**
- Reactor: приложение ожидает готовности I/O, затем выполняет операцию синхронно
- Proactor: операция инициируется асинхронно, приложение уведомляется о завершении

```python
import asyncio
from abc import ABC, abstractmethod
from typing import Dict, Any, Callable, Coroutine
from dataclasses import dataclass
from enum import Enum, auto

class CompletionType(Enum):
    READ_COMPLETE = auto()
    WRITE_COMPLETE = auto()
    CONNECT_COMPLETE = auto()
    TIMER_COMPLETE = auto()

@dataclass
class CompletionEvent:
    """Событие завершения асинхронной операции"""
    type: CompletionType
    source: str
    result: Any
    error: Exception = None

class CompletionHandler(ABC):
    """Обработчик завершения"""

    @abstractmethod
    async def handle_completion(self, event: CompletionEvent):
        pass

class AsyncOperation:
    """Асинхронная операция"""

    def __init__(self, name: str, coro: Coroutine, handler: CompletionHandler):
        self.name = name
        self._coro = coro
        self._handler = handler

    async def execute(self) -> CompletionEvent:
        try:
            result = await self._coro
            event = CompletionEvent(
                type=CompletionType.READ_COMPLETE,
                source=self.name,
                result=result
            )
        except Exception as e:
            event = CompletionEvent(
                type=CompletionType.READ_COMPLETE,
                source=self.name,
                result=None,
                error=e
            )

        await self._handler.handle_completion(event)
        return event

class Proactor:
    """Асинхронный инициатор и демультиплексор"""

    def __init__(self):
        self._pending: Dict[str, asyncio.Task] = {}
        self._completion_handlers: Dict[str, CompletionHandler] = {}

    def initiate_operation(self, operation: AsyncOperation):
        """Инициирует асинхронную операцию"""
        task = asyncio.create_task(operation.execute())
        self._pending[operation.name] = task
        print(f"[Proactor] Initiated: {operation.name}")

    async def run(self):
        """Ожидает завершения всех операций"""
        if not self._pending:
            return

        print(f"[Proactor] Waiting for {len(self._pending)} operations...")
        await asyncio.gather(*self._pending.values(), return_exceptions=True)
        self._pending.clear()

# ========== Example: Async File/Network Operations ==========

class DataHandler(CompletionHandler):
    def __init__(self, name: str):
        self.name = name
        self.results = []

    async def handle_completion(self, event: CompletionEvent):
        if event.error:
            print(f"[{self.name}] Error: {event.error}")
        else:
            print(f"[{self.name}] Completed: {event.source} -> {event.result}")
            self.results.append(event.result)

# Симуляция асинхронных операций
async def fetch_url(url: str) -> str:
    await asyncio.sleep(0.5)  # Симуляция сетевого запроса
    return f"Content from {url}"

async def read_file(path: str) -> str:
    await asyncio.sleep(0.3)  # Симуляция чтения файла
    return f"Data from {path}"

async def compute_heavy(n: int) -> int:
    await asyncio.sleep(0.4)  # Симуляция вычислений
    return n * n

async def main():
    proactor = Proactor()
    handler = DataHandler("MainHandler")

    # Инициируем несколько асинхронных операций
    proactor.initiate_operation(AsyncOperation(
        "fetch_api",
        fetch_url("https://api.example.com/data"),
        handler
    ))

    proactor.initiate_operation(AsyncOperation(
        "read_config",
        read_file("/etc/config.json"),
        handler
    ))

    proactor.initiate_operation(AsyncOperation(
        "compute",
        compute_heavy(42),
        handler
    ))

    # Ожидаем завершения всех операций
    await proactor.run()

    print(f"\nAll results: {handler.results}")

# Запуск
asyncio.run(main())
```

---

### 5. Half-Sync/Half-Async

**Назначение:** Разделяет синхронную и асинхронную обработку, позволяя синхронным сервисам использовать преимущества асинхронного I/O.

```python
import asyncio
import queue
import threading
from typing import Callable, Any
from dataclasses import dataclass
from concurrent.futures import ThreadPoolExecutor

@dataclass
class WorkItem:
    """Единица работы"""
    id: int
    task: Callable[[], Any]
    callback: Callable[[Any], None] = None

class AsyncLayer:
    """Асинхронный уровень - обрабатывает I/O"""

    def __init__(self, work_queue: queue.Queue):
        self._queue = work_queue
        self._running = False

    async def receive_request(self, request_id: int, data: Any):
        """Асинхронно получает запрос"""
        print(f"[Async] Received request {request_id}")
        # Помещаем в очередь для синхронной обработки
        self._queue.put(WorkItem(
            id=request_id,
            task=lambda: self._process_sync(data),
            callback=lambda result: print(f"[Async] Sending response for {request_id}: {result}")
        ))

    def _process_sync(self, data: Any) -> Any:
        """Заглушка для синхронной обработки"""
        return f"Processed: {data}"

class QueueingLayer:
    """Промежуточный уровень - очередь между async и sync"""

    def __init__(self):
        self._queue: queue.Queue = queue.Queue()

    @property
    def queue(self) -> queue.Queue:
        return self._queue

    def enqueue(self, item: WorkItem):
        self._queue.put(item)

    def dequeue(self, timeout: float = None) -> WorkItem | None:
        try:
            return self._queue.get(timeout=timeout)
        except queue.Empty:
            return None

class SyncLayer:
    """Синхронный уровень - обрабатывает бизнес-логику"""

    def __init__(self, work_queue: queue.Queue, num_workers: int = 4):
        self._queue = work_queue
        self._num_workers = num_workers
        self._executor = ThreadPoolExecutor(max_workers=num_workers)
        self._running = False

    def start(self):
        """Запускает worker threads"""
        self._running = True
        for i in range(self._num_workers):
            self._executor.submit(self._worker, i)
        print(f"[Sync] Started {self._num_workers} workers")

    def stop(self):
        """Останавливает workers"""
        self._running = False
        # Отправляем poison pills
        for _ in range(self._num_workers):
            self._queue.put(None)
        self._executor.shutdown(wait=True)
        print("[Sync] All workers stopped")

    def _worker(self, worker_id: int):
        """Рабочий поток"""
        print(f"[Sync] Worker {worker_id} started")

        while self._running:
            try:
                item = self._queue.get(timeout=1.0)

                if item is None:  # Poison pill
                    break

                print(f"[Sync] Worker {worker_id} processing item {item.id}")

                # Выполняем синхронную задачу
                result = item.task()

                # Вызываем callback с результатом
                if item.callback:
                    item.callback(result)

            except queue.Empty:
                continue
            except Exception as e:
                print(f"[Sync] Worker {worker_id} error: {e}")

        print(f"[Sync] Worker {worker_id} stopped")

class HalfSyncHalfAsync:
    """Фасад для всей системы"""

    def __init__(self, num_sync_workers: int = 4):
        self._queuing = QueueingLayer()
        self._async_layer = AsyncLayer(self._queuing.queue)
        self._sync_layer = SyncLayer(self._queuing.queue, num_sync_workers)

    def start(self):
        self._sync_layer.start()

    def stop(self):
        self._sync_layer.stop()

    async def handle_request(self, request_id: int, data: Any):
        await self._async_layer.receive_request(request_id, data)

# ========== Usage ==========
async def main():
    system = HalfSyncHalfAsync(num_sync_workers=2)
    system.start()

    # Симулируем асинхронные запросы
    tasks = [
        system.handle_request(1, "Request A"),
        system.handle_request(2, "Request B"),
        system.handle_request(3, "Request C"),
    ]

    await asyncio.gather(*tasks)

    # Даём время на обработку
    await asyncio.sleep(1)

    system.stop()

asyncio.run(main())
```

---

## Идиомы

Идиомы — это низкоуровневые паттерны, специфичные для конкретного языка программирования. Они описывают best practices для типичных задач.

### 1. RAII (Resource Acquisition Is Initialization)

**Назначение:** Связывает жизненный цикл ресурса с временем жизни объекта.

```python
# Python эквивалент RAII через context managers

class DatabaseConnection:
    """RAII-подобное управление соединением с БД"""

    def __init__(self, connection_string: str):
        self.connection_string = connection_string
        self._connection = None

    def __enter__(self):
        print(f"Opening connection to {self.connection_string}")
        self._connection = {"connected": True}  # Симуляция
        return self

    def __exit__(self, exc_type, exc_val, exc_tb):
        print("Closing connection")
        self._connection = None
        # Возвращаем False, чтобы исключения пробрасывались
        return False

    def query(self, sql: str) -> str:
        if not self._connection:
            raise RuntimeError("Not connected")
        return f"Result of: {sql}"

# Использование
with DatabaseConnection("postgresql://localhost/db") as db:
    result = db.query("SELECT * FROM users")
    print(result)
# Соединение автоматически закрывается

# Даже при исключении
try:
    with DatabaseConnection("postgresql://localhost/db") as db:
        raise ValueError("Something went wrong")
except ValueError:
    print("Error handled, but connection was closed!")
```

```python
# contextlib для создания context managers

from contextlib import contextmanager
import time

@contextmanager
def timer(name: str):
    """RAII-стиль для измерения времени"""
    start = time.time()
    print(f"[{name}] Started")
    try:
        yield
    finally:
        elapsed = time.time() - start
        print(f"[{name}] Finished in {elapsed:.3f}s")

@contextmanager
def transaction(db):
    """RAII для транзакций"""
    print("BEGIN TRANSACTION")
    try:
        yield
        print("COMMIT")
    except Exception as e:
        print(f"ROLLBACK due to: {e}")
        raise

# Использование
with timer("Database operation"):
    time.sleep(0.5)
```

---

### 2. Counted Pointer / Smart Pointer

**Назначение:** Автоматическое управление памятью через подсчёт ссылок.

```python
from typing import TypeVar, Generic, Optional
from dataclasses import dataclass
import weakref

T = TypeVar('T')

class RefCounted(Generic[T]):
    """Подсчёт ссылок (аналог shared_ptr)"""

    _instances: dict = {}

    def __init__(self, value: T):
        self._value = value
        self._ref_count = 1
        self._id = id(value)
        RefCounted._instances[self._id] = self

    @property
    def value(self) -> T:
        return self._value

    @property
    def ref_count(self) -> int:
        return self._ref_count

    def acquire(self) -> "RefCounted[T]":
        """Увеличивает счётчик ссылок"""
        self._ref_count += 1
        print(f"Acquired: ref_count = {self._ref_count}")
        return self

    def release(self):
        """Уменьшает счётчик ссылок"""
        self._ref_count -= 1
        print(f"Released: ref_count = {self._ref_count}")
        if self._ref_count == 0:
            print(f"Destroying resource: {self._value}")
            del RefCounted._instances[self._id]
            self._value = None

class WeakRef(Generic[T]):
    """Слабая ссылка (аналог weak_ptr)"""

    def __init__(self, ref_counted: RefCounted[T]):
        self._weak_ref = weakref.ref(ref_counted)

    def lock(self) -> Optional[RefCounted[T]]:
        """Пытается получить сильную ссылку"""
        obj = self._weak_ref()
        if obj and obj.ref_count > 0:
            return obj.acquire()
        return None

    def expired(self) -> bool:
        obj = self._weak_ref()
        return obj is None or obj.ref_count == 0

# Использование
print("=== RefCounted Demo ===")
resource = RefCounted({"data": "important"})
print(f"Initial ref_count: {resource.ref_count}")

# Создаём ещё одну ссылку
ref2 = resource.acquire()

# Освобождаем
ref2.release()
resource.release()  # Ресурс уничтожается
```

---

### 3. Execute Around Method

**Назначение:** Инкапсулирует пару связанных операций (например, lock/unlock, open/close) вокруг изменяемой логики.

```python
from typing import Callable, TypeVar
from functools import wraps
import threading
import time

R = TypeVar('R')

def with_lock(lock: threading.Lock):
    """Декоратор: Execute Around для блокировки"""
    def decorator(func: Callable[..., R]) -> Callable[..., R]:
        @wraps(func)
        def wrapper(*args, **kwargs) -> R:
            lock.acquire()
            try:
                return func(*args, **kwargs)
            finally:
                lock.release()
        return wrapper
    return decorator

def with_timing(func: Callable[..., R]) -> Callable[..., R]:
    """Декоратор: измерение времени"""
    @wraps(func)
    def wrapper(*args, **kwargs) -> R:
        start = time.time()
        try:
            return func(*args, **kwargs)
        finally:
            elapsed = time.time() - start
            print(f"{func.__name__} took {elapsed:.4f}s")
    return wrapper

def with_retry(max_attempts: int = 3, delay: float = 1.0):
    """Декоратор: повторные попытки"""
    def decorator(func: Callable[..., R]) -> Callable[..., R]:
        @wraps(func)
        def wrapper(*args, **kwargs) -> R:
            last_exception = None
            for attempt in range(max_attempts):
                try:
                    return func(*args, **kwargs)
                except Exception as e:
                    last_exception = e
                    print(f"Attempt {attempt + 1} failed: {e}")
                    if attempt < max_attempts - 1:
                        time.sleep(delay)
            raise last_exception
        return wrapper
    return decorator

def with_transaction(db_connection):
    """Execute Around для транзакций"""
    def decorator(func: Callable[..., R]) -> Callable[..., R]:
        @wraps(func)
        def wrapper(*args, **kwargs) -> R:
            db_connection.begin()
            try:
                result = func(*args, **kwargs)
                db_connection.commit()
                return result
            except Exception:
                db_connection.rollback()
                raise
        return wrapper
    return decorator

# Использование
lock = threading.Lock()

@with_timing
@with_lock(lock)
@with_retry(max_attempts=3, delay=0.5)
def critical_operation(data: str) -> str:
    print(f"Processing: {data}")
    if data == "fail":
        raise ValueError("Simulated failure")
    return f"Processed: {data}"

result = critical_operation("success")
print(f"Result: {result}")

try:
    critical_operation("fail")
except ValueError:
    print("Operation failed after all retries")
```

---

### 4. Type-Safe Enum

**Назначение:** Обеспечивает типобезопасные перечисления.

```python
from enum import Enum, auto, unique
from typing import Dict, Any

@unique
class OrderStatus(Enum):
    """Типобезопасное перечисление статусов заказа"""

    PENDING = auto()
    CONFIRMED = auto()
    SHIPPED = auto()
    DELIVERED = auto()
    CANCELLED = auto()

    def can_transition_to(self, new_status: "OrderStatus") -> bool:
        """Проверяет допустимость перехода"""
        transitions: Dict[OrderStatus, set] = {
            OrderStatus.PENDING: {OrderStatus.CONFIRMED, OrderStatus.CANCELLED},
            OrderStatus.CONFIRMED: {OrderStatus.SHIPPED, OrderStatus.CANCELLED},
            OrderStatus.SHIPPED: {OrderStatus.DELIVERED},
            OrderStatus.DELIVERED: set(),
            OrderStatus.CANCELLED: set(),
        }
        return new_status in transitions.get(self, set())

    @property
    def is_final(self) -> bool:
        return self in {OrderStatus.DELIVERED, OrderStatus.CANCELLED}

class Order:
    def __init__(self, order_id: int):
        self.order_id = order_id
        self._status = OrderStatus.PENDING

    @property
    def status(self) -> OrderStatus:
        return self._status

    def change_status(self, new_status: OrderStatus):
        if not self._status.can_transition_to(new_status):
            raise ValueError(
                f"Cannot transition from {self._status.name} to {new_status.name}"
            )
        print(f"Order {self.order_id}: {self._status.name} -> {new_status.name}")
        self._status = new_status

# Использование
order = Order(123)
print(f"Initial status: {order.status.name}")

order.change_status(OrderStatus.CONFIRMED)
order.change_status(OrderStatus.SHIPPED)
order.change_status(OrderStatus.DELIVERED)

print(f"Is final: {order.status.is_final}")

# Попытка недопустимого перехода
try:
    order.change_status(OrderStatus.PENDING)
except ValueError as e:
    print(f"Error: {e}")
```

---

## Сравнение PoSA и GoF паттернов

| Аспект | GoF | PoSA |
|--------|-----|------|
| **Уровень** | Классы и объекты | От архитектуры до идиом |
| **Масштаб** | Микро-дизайн | Макро- и микро-дизайн |
| **Фокус** | Повторное использование кода | Структура системы |
| **Примеры** | Singleton, Factory, Observer | Layers, MVC, Reactor |
| **Применимость** | Внутри компонентов | Между компонентами и системами |

---

## Best Practices

1. **Выбирайте правильный уровень абстракции**
   - Архитектурные паттерны — для общей структуры
   - Паттерны проектирования — для взаимодействия компонентов
   - Идиомы — для типичных конструкций языка

2. **Комбинируйте паттерны**
   - Layers + MVC = многоуровневое web-приложение
   - Reactor + Chain of Responsibility = event-driven middleware
   - Microkernel + Observer = расширяемая плагин-система

3. **Учитывайте контекст**
   - Требования к производительности
   - Размер и сложность системы
   - Навыки команды

4. **Документируйте решения**
   - Почему выбран конкретный паттерн
   - Какие компромиссы были сделаны
   - Как паттерн влияет на развитие системы

---

## Типичные ошибки

1. **Применение архитектурных паттернов к малым системам**
   - Layers для простого скрипта — overkill

2. **Смешение уровней абстракции**
   - Бизнес-логика в Presentation Layer
   - Инфраструктурные детали в Domain Layer

3. **Игнорирование trade-offs**
   - Layers добавляет накладные расходы
   - Microkernel усложняет отладку

4. **Слепое следование паттерну**
   - Адаптируйте паттерн под свои нужды
   - Не каждая система требует полной реализации

---

## Дополнительные ресурсы

- [Pattern-Oriented Software Architecture Volume 1](https://www.amazon.com/Pattern-Oriented-Software-Architecture-System-Patterns/dp/0471958697)
- [Pattern-Oriented Software Architecture Volume 2](https://www.amazon.com/Pattern-Oriented-Software-Architecture-Concurrent-Networked/dp/0471606952)
- [POSA Website](http://www.cs.wustl.edu/~schmidt/POSA/)
- [Martin Fowler - Patterns of Enterprise Application Architecture](https://martinfowler.com/eaaCatalog/)
