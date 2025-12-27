# Component-Based Architecture

## Что такое компонентная архитектура?

Компонентная архитектура (Component-Based Architecture, CBA) — это подход к проектированию программного обеспечения, при котором система строится из независимых, переиспользуемых компонентов. Каждый компонент инкапсулирует определённую функциональность и взаимодействует с другими компонентами через чётко определённые интерфейсы.

> "Компонент — это единица композиции с контрактно определёнными интерфейсами и явно указанными зависимостями от контекста." — Клеменс Шиперски

## Основные концепции

```
┌─────────────────────────────────────────────────────────┐
│                     Application                          │
│                                                          │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐      │
│  │  Component  │  │  Component  │  │  Component  │      │
│  │      A      │  │      B      │  │      C      │      │
│  │ ┌─────────┐ │  │ ┌─────────┐ │  │ ┌─────────┐ │      │
│  │ │Interface│ │  │ │Interface│ │  │ │Interface│ │      │
│  │ └────┬────┘ │  │ └────┬────┘ │  │ └────┬────┘ │      │
│  │      │      │  │      │      │  │      │      │      │
│  │ ┌────▼────┐ │  │ ┌────▼────┐ │  │ ┌────▼────┐ │      │
│  │ │  Impl   │ │  │ │  Impl   │ │  │ │  Impl   │ │      │
│  │ └─────────┘ │  │ └─────────┘ │  │ └─────────┘ │      │
│  └─────────────┘  └─────────────┘  └─────────────┘      │
│          │                │                │            │
│          └────────────────┼────────────────┘            │
│                           ▼                             │
│                    ┌─────────────┐                      │
│                    │  Component  │                      │
│                    │  Container  │                      │
│                    └─────────────┘                      │
└─────────────────────────────────────────────────────────┘
```

## Характеристики компонента

```python
from abc import ABC, abstractmethod
from typing import Protocol, Dict, Any, List
from dataclasses import dataclass

# === Основные характеристики компонента ===

@dataclass
class ComponentMetadata:
    """Метаданные компонента."""
    name: str
    version: str
    description: str
    author: str
    dependencies: List[str]
    provides: List[str]  # Предоставляемые интерфейсы
    requires: List[str]  # Требуемые интерфейсы


class ComponentInterface(Protocol):
    """Базовый протокол для всех компонентов."""

    def get_metadata(self) -> ComponentMetadata:
        """Получение метаданных компонента."""
        ...

    def initialize(self, config: Dict[str, Any]) -> None:
        """Инициализация компонента."""
        ...

    def start(self) -> None:
        """Запуск компонента."""
        ...

    def stop(self) -> None:
        """Остановка компонента."""
        ...

    def get_status(self) -> str:
        """Получение статуса компонента."""
        ...
```

## Пример реализации компонентной системы

### Базовая инфраструктура

```python
from abc import ABC, abstractmethod
from typing import Dict, Any, Type, Optional, List
from dataclasses import dataclass, field
from enum import Enum
import logging

class ComponentState(Enum):
    CREATED = "created"
    INITIALIZED = "initialized"
    STARTED = "started"
    STOPPED = "stopped"
    ERROR = "error"

@dataclass
class ComponentInfo:
    """Информация о компоненте."""
    name: str
    version: str
    state: ComponentState = ComponentState.CREATED
    dependencies: List[str] = field(default_factory=list)
    config: Dict[str, Any] = field(default_factory=dict)


class Component(ABC):
    """Базовый класс для всех компонентов."""

    def __init__(self, name: str, version: str = "1.0.0"):
        self.info = ComponentInfo(name=name, version=version)
        self.logger = logging.getLogger(f"component.{name}")
        self._container: Optional['ComponentContainer'] = None

    @property
    def name(self) -> str:
        return self.info.name

    @property
    def state(self) -> ComponentState:
        return self.info.state

    def set_container(self, container: 'ComponentContainer'):
        """Установка ссылки на контейнер."""
        self._container = container

    def get_component(self, name: str) -> Optional['Component']:
        """Получение другого компонента через контейнер."""
        if self._container:
            return self._container.get_component(name)
        return None

    @abstractmethod
    def initialize(self, config: Dict[str, Any]) -> None:
        """Инициализация компонента с конфигурацией."""
        pass

    @abstractmethod
    def start(self) -> None:
        """Запуск компонента."""
        pass

    @abstractmethod
    def stop(self) -> None:
        """Остановка компонента."""
        pass

    def get_required_components(self) -> List[str]:
        """Список требуемых компонентов."""
        return self.info.dependencies


class ComponentContainer:
    """Контейнер для управления компонентами."""

    def __init__(self):
        self._components: Dict[str, Component] = {}
        self._component_order: List[str] = []
        self.logger = logging.getLogger("component.container")

    def register(self, component: Component) -> None:
        """Регистрация компонента в контейнере."""
        if component.name in self._components:
            raise ValueError(f"Component {component.name} already registered")

        self._components[component.name] = component
        component.set_container(self)
        self.logger.info(f"Registered component: {component.name}")

    def get_component(self, name: str) -> Optional[Component]:
        """Получение компонента по имени."""
        return self._components.get(name)

    def _resolve_dependencies(self) -> List[str]:
        """Топологическая сортировка компонентов по зависимостям."""
        visited = set()
        order = []

        def visit(name: str):
            if name in visited:
                return
            visited.add(name)

            component = self._components.get(name)
            if component:
                for dep in component.get_required_components():
                    if dep not in self._components:
                        raise ValueError(
                            f"Missing dependency: {dep} for {name}"
                        )
                    visit(dep)
                order.append(name)

        for name in self._components:
            visit(name)

        return order

    def initialize_all(self, configs: Dict[str, Dict[str, Any]] = None) -> None:
        """Инициализация всех компонентов в правильном порядке."""
        configs = configs or {}
        self._component_order = self._resolve_dependencies()

        for name in self._component_order:
            component = self._components[name]
            config = configs.get(name, {})

            try:
                component.initialize(config)
                component.info.state = ComponentState.INITIALIZED
                self.logger.info(f"Initialized component: {name}")
            except Exception as e:
                component.info.state = ComponentState.ERROR
                self.logger.error(f"Failed to initialize {name}: {e}")
                raise

    def start_all(self) -> None:
        """Запуск всех компонентов."""
        for name in self._component_order:
            component = self._components[name]

            if component.state != ComponentState.INITIALIZED:
                raise RuntimeError(
                    f"Component {name} not initialized"
                )

            try:
                component.start()
                component.info.state = ComponentState.STARTED
                self.logger.info(f"Started component: {name}")
            except Exception as e:
                component.info.state = ComponentState.ERROR
                self.logger.error(f"Failed to start {name}: {e}")
                raise

    def stop_all(self) -> None:
        """Остановка всех компонентов в обратном порядке."""
        for name in reversed(self._component_order):
            component = self._components[name]

            if component.state == ComponentState.STARTED:
                try:
                    component.stop()
                    component.info.state = ComponentState.STOPPED
                    self.logger.info(f"Stopped component: {name}")
                except Exception as e:
                    self.logger.error(f"Error stopping {name}: {e}")

    def get_status(self) -> Dict[str, str]:
        """Получение статуса всех компонентов."""
        return {
            name: comp.state.value
            for name, comp in self._components.items()
        }
```

### Примеры компонентов

```python
# === Компонент базы данных ===

from typing import Optional
import asyncio

class DatabaseComponent(Component):
    """Компонент для работы с базой данных."""

    def __init__(self):
        super().__init__("database", "1.0.0")
        self.connection_pool = None
        self.connection_string: str = ""

    def initialize(self, config: Dict[str, Any]) -> None:
        self.connection_string = config.get(
            "connection_string",
            "postgresql://localhost:5432/app"
        )
        self.pool_size = config.get("pool_size", 10)

    def start(self) -> None:
        # Создание пула соединений
        self.logger.info(
            f"Connecting to database: {self.connection_string}"
        )
        # self.connection_pool = create_pool(...)
        self.connection_pool = {"status": "connected"}

    def stop(self) -> None:
        if self.connection_pool:
            # self.connection_pool.close()
            self.connection_pool = None
            self.logger.info("Database connection closed")

    def get_connection(self):
        """Получение соединения из пула."""
        if not self.connection_pool:
            raise RuntimeError("Database not started")
        return self.connection_pool

    async def execute(self, query: str, params: tuple = None):
        """Выполнение SQL запроса."""
        self.logger.debug(f"Executing query: {query}")
        # Реальная реализация
        return []


# === Компонент кэширования ===

class CacheComponent(Component):
    """Компонент для кэширования данных."""

    def __init__(self):
        super().__init__("cache", "1.0.0")
        self.cache: Dict[str, Any] = {}
        self.redis_client = None

    def initialize(self, config: Dict[str, Any]) -> None:
        self.redis_url = config.get("redis_url", "redis://localhost:6379")
        self.default_ttl = config.get("default_ttl", 3600)

    def start(self) -> None:
        # Подключение к Redis
        self.logger.info(f"Connecting to Redis: {self.redis_url}")
        # self.redis_client = redis.from_url(self.redis_url)
        self.cache = {}

    def stop(self) -> None:
        self.cache.clear()
        self.redis_client = None

    def get(self, key: str) -> Optional[Any]:
        """Получение значения из кэша."""
        return self.cache.get(key)

    def set(self, key: str, value: Any, ttl: int = None) -> None:
        """Сохранение значения в кэш."""
        self.cache[key] = value

    def delete(self, key: str) -> None:
        """Удаление значения из кэша."""
        self.cache.pop(key, None)


# === Компонент аутентификации ===

class AuthComponent(Component):
    """Компонент аутентификации и авторизации."""

    def __init__(self):
        super().__init__("auth", "1.0.0")
        self.info.dependencies = ["database", "cache"]
        self.secret_key: str = ""
        self.token_ttl: int = 3600

    def initialize(self, config: Dict[str, Any]) -> None:
        self.secret_key = config.get("secret_key", "default-secret")
        self.token_ttl = config.get("token_ttl", 3600)

    def start(self) -> None:
        # Получаем зависимые компоненты
        self.db = self.get_component("database")
        self.cache = self.get_component("cache")

        if not self.db or not self.cache:
            raise RuntimeError("Required components not available")

        self.logger.info("Auth component started")

    def stop(self) -> None:
        self.db = None
        self.cache = None

    def authenticate(self, username: str, password: str) -> Optional[str]:
        """Аутентификация пользователя."""
        # Проверка в базе данных
        # user = self.db.execute("SELECT * FROM users WHERE username = ?", (username,))
        # Генерация токена
        import hashlib
        token = hashlib.sha256(
            f"{username}:{self.secret_key}".encode()
        ).hexdigest()

        # Кэширование токена
        self.cache.set(f"token:{token}", username, self.token_ttl)

        return token

    def verify_token(self, token: str) -> Optional[str]:
        """Проверка токена."""
        return self.cache.get(f"token:{token}")


# === Компонент уведомлений ===

class NotificationComponent(Component):
    """Компонент для отправки уведомлений."""

    def __init__(self):
        super().__init__("notification", "1.0.0")
        self.providers: Dict[str, Any] = {}

    def initialize(self, config: Dict[str, Any]) -> None:
        self.email_config = config.get("email", {})
        self.sms_config = config.get("sms", {})
        self.push_config = config.get("push", {})

    def start(self) -> None:
        # Инициализация провайдеров
        if self.email_config:
            self.providers["email"] = self._create_email_provider()
        if self.sms_config:
            self.providers["sms"] = self._create_sms_provider()

        self.logger.info(
            f"Notification providers: {list(self.providers.keys())}"
        )

    def stop(self) -> None:
        self.providers.clear()

    def _create_email_provider(self):
        return {"type": "email", "config": self.email_config}

    def _create_sms_provider(self):
        return {"type": "sms", "config": self.sms_config}

    async def send(
        self,
        channel: str,
        recipient: str,
        message: str
    ) -> bool:
        """Отправка уведомления."""
        provider = self.providers.get(channel)
        if not provider:
            raise ValueError(f"Unknown channel: {channel}")

        self.logger.info(
            f"Sending {channel} notification to {recipient}"
        )
        # Реальная отправка
        return True
```

### Использование компонентной системы

```python
# === Пример использования ===

def main():
    # Создание контейнера
    container = ComponentContainer()

    # Регистрация компонентов
    container.register(DatabaseComponent())
    container.register(CacheComponent())
    container.register(AuthComponent())
    container.register(NotificationComponent())

    # Конфигурация компонентов
    configs = {
        "database": {
            "connection_string": "postgresql://localhost:5432/myapp",
            "pool_size": 20
        },
        "cache": {
            "redis_url": "redis://localhost:6379",
            "default_ttl": 3600
        },
        "auth": {
            "secret_key": "super-secret-key",
            "token_ttl": 7200
        },
        "notification": {
            "email": {
                "smtp_host": "smtp.example.com",
                "smtp_port": 587
            }
        }
    }

    try:
        # Инициализация всех компонентов
        container.initialize_all(configs)

        # Запуск всех компонентов
        container.start_all()

        # Проверка статуса
        print("Component status:", container.get_status())

        # Использование компонентов
        auth = container.get_component("auth")
        if auth:
            token = auth.authenticate("user", "password")
            print(f"Generated token: {token}")

    finally:
        # Остановка при завершении
        container.stop_all()


if __name__ == "__main__":
    logging.basicConfig(level=logging.INFO)
    main()
```

## Интерфейсы компонентов

```python
# === Определение интерфейсов через Protocol ===

from typing import Protocol, runtime_checkable

@runtime_checkable
class StorageInterface(Protocol):
    """Интерфейс для хранилища данных."""

    def save(self, key: str, data: Any) -> bool:
        """Сохранение данных."""
        ...

    def load(self, key: str) -> Optional[Any]:
        """Загрузка данных."""
        ...

    def delete(self, key: str) -> bool:
        """Удаление данных."""
        ...

    def exists(self, key: str) -> bool:
        """Проверка существования."""
        ...


@runtime_checkable
class MessageQueueInterface(Protocol):
    """Интерфейс для очереди сообщений."""

    def publish(self, topic: str, message: Any) -> bool:
        """Публикация сообщения."""
        ...

    def subscribe(self, topic: str, handler: callable) -> None:
        """Подписка на топик."""
        ...

    def unsubscribe(self, topic: str) -> None:
        """Отписка от топика."""
        ...


# === Реализации интерфейсов ===

class FileStorageComponent(Component):
    """Компонент хранилища на основе файлов."""

    def __init__(self):
        super().__init__("file_storage", "1.0.0")
        self.base_path: str = ""

    def initialize(self, config: Dict[str, Any]) -> None:
        self.base_path = config.get("base_path", "/tmp/storage")

    def start(self) -> None:
        import os
        os.makedirs(self.base_path, exist_ok=True)

    def stop(self) -> None:
        pass

    # Реализация StorageInterface
    def save(self, key: str, data: Any) -> bool:
        import json
        import os
        path = os.path.join(self.base_path, f"{key}.json")
        with open(path, 'w') as f:
            json.dump(data, f)
        return True

    def load(self, key: str) -> Optional[Any]:
        import json
        import os
        path = os.path.join(self.base_path, f"{key}.json")
        if os.path.exists(path):
            with open(path, 'r') as f:
                return json.load(f)
        return None

    def delete(self, key: str) -> bool:
        import os
        path = os.path.join(self.base_path, f"{key}.json")
        if os.path.exists(path):
            os.remove(path)
            return True
        return False

    def exists(self, key: str) -> bool:
        import os
        path = os.path.join(self.base_path, f"{key}.json")
        return os.path.exists(path)


class S3StorageComponent(Component):
    """Компонент хранилища на основе S3."""

    def __init__(self):
        super().__init__("s3_storage", "1.0.0")
        self.bucket: str = ""
        self.client = None

    def initialize(self, config: Dict[str, Any]) -> None:
        self.bucket = config.get("bucket", "default-bucket")
        self.region = config.get("region", "us-east-1")

    def start(self) -> None:
        # import boto3
        # self.client = boto3.client('s3', region_name=self.region)
        self.client = {"type": "s3"}

    def stop(self) -> None:
        self.client = None

    # Реализация StorageInterface
    def save(self, key: str, data: Any) -> bool:
        # self.client.put_object(Bucket=self.bucket, Key=key, Body=json.dumps(data))
        return True

    def load(self, key: str) -> Optional[Any]:
        # response = self.client.get_object(Bucket=self.bucket, Key=key)
        return None

    def delete(self, key: str) -> bool:
        # self.client.delete_object(Bucket=self.bucket, Key=key)
        return True

    def exists(self, key: str) -> bool:
        # Проверка через head_object
        return False


# Проверка реализации интерфейса
def validate_storage(storage: StorageInterface) -> bool:
    """Проверка, что объект реализует StorageInterface."""
    return isinstance(storage, StorageInterface)
```

## Композиция компонентов

```python
# === Составные компоненты ===

class CompositeComponent(Component):
    """Компонент, состоящий из нескольких подкомпонентов."""

    def __init__(self, name: str):
        super().__init__(name)
        self._sub_components: List[Component] = []

    def add_sub_component(self, component: Component) -> None:
        """Добавление подкомпонента."""
        self._sub_components.append(component)
        component.set_container(self._container)

    def initialize(self, config: Dict[str, Any]) -> None:
        for comp in self._sub_components:
            sub_config = config.get(comp.name, {})
            comp.initialize(sub_config)

    def start(self) -> None:
        for comp in self._sub_components:
            comp.start()

    def stop(self) -> None:
        for comp in reversed(self._sub_components):
            comp.stop()


# === Пример: E-commerce модуль ===

class PaymentGatewayComponent(Component):
    """Компонент платёжного шлюза."""

    def __init__(self):
        super().__init__("payment_gateway")

    def initialize(self, config: Dict[str, Any]) -> None:
        self.api_key = config.get("api_key", "")

    def start(self) -> None:
        self.logger.info("Payment gateway started")

    def stop(self) -> None:
        pass

    def process_payment(self, amount: float, card_token: str) -> dict:
        return {"status": "success", "transaction_id": "txn_123"}


class InventoryComponent(Component):
    """Компонент управления запасами."""

    def __init__(self):
        super().__init__("inventory")
        self.info.dependencies = ["database"]

    def initialize(self, config: Dict[str, Any]) -> None:
        pass

    def start(self) -> None:
        self.db = self.get_component("database")

    def stop(self) -> None:
        pass

    def check_availability(self, product_id: str, quantity: int) -> bool:
        return True

    def reserve(self, product_id: str, quantity: int) -> str:
        return "reservation_123"


class OrderComponent(Component):
    """Компонент заказов."""

    def __init__(self):
        super().__init__("orders")
        self.info.dependencies = ["database", "inventory", "payment_gateway"]

    def initialize(self, config: Dict[str, Any]) -> None:
        pass

    def start(self) -> None:
        self.db = self.get_component("database")
        self.inventory = self.get_component("inventory")
        self.payment = self.get_component("payment_gateway")

    def stop(self) -> None:
        pass

    def create_order(self, user_id: str, items: List[dict]) -> dict:
        """Создание заказа."""
        # Проверка наличия
        for item in items:
            if not self.inventory.check_availability(
                item["product_id"],
                item["quantity"]
            ):
                raise ValueError(f"Product {item['product_id']} not available")

        # Резервирование
        reservations = []
        for item in items:
            res_id = self.inventory.reserve(
                item["product_id"],
                item["quantity"]
            )
            reservations.append(res_id)

        # Создание заказа в БД
        order = {
            "id": "order_123",
            "user_id": user_id,
            "items": items,
            "reservations": reservations,
            "status": "pending"
        }

        return order

    def process_payment(self, order_id: str, payment_info: dict) -> dict:
        """Обработка оплаты заказа."""
        result = self.payment.process_payment(
            amount=payment_info["amount"],
            card_token=payment_info["card_token"]
        )
        return result


class ECommerceModule(CompositeComponent):
    """Модуль электронной коммерции."""

    def __init__(self):
        super().__init__("ecommerce")

        # Добавляем подкомпоненты
        self.add_sub_component(PaymentGatewayComponent())
        self.add_sub_component(InventoryComponent())
        self.add_sub_component(OrderComponent())

    def get_order_component(self) -> OrderComponent:
        """Получение компонента заказов."""
        return next(
            c for c in self._sub_components
            if isinstance(c, OrderComponent)
        )
```

## Фабрика компонентов

```python
# === Фабрика для создания компонентов ===

from typing import Callable, Type

class ComponentFactory:
    """Фабрика для создания компонентов."""

    _registry: Dict[str, Type[Component]] = {}

    @classmethod
    def register(cls, name: str, component_class: Type[Component]) -> None:
        """Регистрация класса компонента."""
        cls._registry[name] = component_class

    @classmethod
    def create(cls, name: str, **kwargs) -> Component:
        """Создание компонента по имени."""
        if name not in cls._registry:
            raise ValueError(f"Unknown component: {name}")

        return cls._registry[name](**kwargs)

    @classmethod
    def get_available(cls) -> List[str]:
        """Получение списка доступных компонентов."""
        return list(cls._registry.keys())


# Декоратор для автоматической регистрации
def component(name: str):
    """Декоратор для регистрации компонента."""
    def decorator(cls: Type[Component]) -> Type[Component]:
        ComponentFactory.register(name, cls)
        return cls
    return decorator


# Использование декоратора
@component("logger")
class LoggerComponent(Component):
    """Компонент логирования."""

    def __init__(self):
        super().__init__("logger")
        self.handlers: List[logging.Handler] = []

    def initialize(self, config: Dict[str, Any]) -> None:
        self.log_level = config.get("level", "INFO")
        self.log_format = config.get(
            "format",
            "%(asctime)s - %(name)s - %(levelname)s - %(message)s"
        )

    def start(self) -> None:
        root_logger = logging.getLogger()
        root_logger.setLevel(getattr(logging, self.log_level))

        handler = logging.StreamHandler()
        handler.setFormatter(logging.Formatter(self.log_format))
        root_logger.addHandler(handler)

        self.handlers.append(handler)

    def stop(self) -> None:
        root_logger = logging.getLogger()
        for handler in self.handlers:
            root_logger.removeHandler(handler)
        self.handlers.clear()


# Создание через фабрику
logger = ComponentFactory.create("logger")
```

## Преимущества компонентной архитектуры

### 1. Переиспользуемость

```python
# Компоненты можно использовать в разных проектах

# Проект 1: E-commerce
container1 = ComponentContainer()
container1.register(DatabaseComponent())
container1.register(CacheComponent())
container1.register(AuthComponent())
container1.register(OrderComponent())

# Проект 2: CRM
container2 = ComponentContainer()
container2.register(DatabaseComponent())  # Тот же компонент
container2.register(CacheComponent())     # Тот же компонент
container2.register(AuthComponent())      # Тот же компонент
container2.register(CustomerComponent())  # Другой компонент
```

### 2. Тестируемость

```python
# Компоненты легко тестировать изолированно

import pytest
from unittest.mock import Mock, MagicMock

class TestAuthComponent:
    def setup_method(self):
        self.auth = AuthComponent()
        self.auth.initialize({"secret_key": "test-secret"})

        # Мокаем зависимости
        self.mock_db = Mock()
        self.mock_cache = Mock()

        # Устанавливаем моки вместо реальных компонентов
        self.auth.db = self.mock_db
        self.auth.cache = self.mock_cache

    def test_authenticate_success(self):
        # Настройка мока
        self.mock_cache.set = MagicMock()

        # Тест
        token = self.auth.authenticate("user", "password")

        # Проверки
        assert token is not None
        self.mock_cache.set.assert_called_once()

    def test_verify_token_cached(self):
        # Настройка мока
        self.mock_cache.get = MagicMock(return_value="user")

        # Тест
        username = self.auth.verify_token("valid_token")

        # Проверки
        assert username == "user"
        self.mock_cache.get.assert_called_once_with("token:valid_token")
```

### 3. Модульность

```python
# Легко добавлять/удалять/заменять компоненты

# Добавление нового компонента
container.register(MetricsComponent())

# Замена реализации
container.unregister("cache")
container.register(RedisClusterCacheComponent())  # Другая реализация

# Условное подключение
if config.get("enable_analytics"):
    container.register(AnalyticsComponent())
```

## Недостатки компонентной архитектуры

### 1. Сложность управления зависимостями

```python
# Проблема: циклические зависимости

class ComponentA(Component):
    def __init__(self):
        super().__init__("A")
        self.info.dependencies = ["B"]  # A зависит от B

class ComponentB(Component):
    def __init__(self):
        super().__init__("B")
        self.info.dependencies = ["A"]  # B зависит от A - ЦИКЛ!

# Решение: использование событий или интерфейсов

class ComponentA(Component):
    def on_b_ready(self, component_b):
        """Вызывается когда B готов."""
        self.component_b = component_b

class ComponentB(Component):
    def start(self) -> None:
        # Уведомляем A после старта
        comp_a = self.get_component("A")
        if comp_a:
            comp_a.on_b_ready(self)
```

### 2. Накладные расходы на абстракции

```python
# Проблема: дополнительные слои абстракции

# Прямой вызов (быстро)
result = cache.get(key)

# Через компонент (медленнее из-за дополнительных проверок)
cache_component = container.get_component("cache")
if cache_component and cache_component.state == ComponentState.STARTED:
    result = cache_component.get(key)
```

## Best Practices

### 1. Определяйте чёткие контракты

```python
from typing import TypedDict

class UserData(TypedDict):
    id: str
    email: str
    name: str

class UserServiceInterface(Protocol):
    """Чёткий контракт для сервиса пользователей."""

    def get_user(self, user_id: str) -> Optional[UserData]:
        """Получение пользователя по ID."""
        ...

    def create_user(self, data: UserData) -> UserData:
        """Создание пользователя."""
        ...

    def update_user(self, user_id: str, data: UserData) -> UserData:
        """Обновление пользователя."""
        ...

    def delete_user(self, user_id: str) -> bool:
        """Удаление пользователя."""
        ...
```

### 2. Используйте события для слабой связанности

```python
from typing import Callable, Dict, List

class EventBus:
    """Шина событий для компонентов."""

    def __init__(self):
        self._handlers: Dict[str, List[Callable]] = {}

    def subscribe(self, event_type: str, handler: Callable) -> None:
        if event_type not in self._handlers:
            self._handlers[event_type] = []
        self._handlers[event_type].append(handler)

    def publish(self, event_type: str, data: Any = None) -> None:
        for handler in self._handlers.get(event_type, []):
            handler(data)


class EventAwareComponent(Component):
    """Компонент с поддержкой событий."""

    def __init__(self, name: str, event_bus: EventBus):
        super().__init__(name)
        self.event_bus = event_bus

    def emit(self, event_type: str, data: Any = None) -> None:
        self.event_bus.publish(f"{self.name}.{event_type}", data)

    def on(self, event_type: str, handler: Callable) -> None:
        self.event_bus.subscribe(event_type, handler)
```

### 3. Версионируйте компоненты

```python
from packaging import version

class VersionedComponentContainer(ComponentContainer):
    """Контейнер с поддержкой версионирования."""

    def check_compatibility(
        self,
        component: Component,
        required_version: str
    ) -> bool:
        """Проверка совместимости версий."""
        return version.parse(component.info.version) >= version.parse(required_version)

    def register_with_version_check(
        self,
        component: Component,
        min_versions: Dict[str, str] = None
    ) -> None:
        """Регистрация с проверкой версий зависимостей."""
        min_versions = min_versions or {}

        for dep_name, min_ver in min_versions.items():
            dep = self._components.get(dep_name)
            if dep and not self.check_compatibility(dep, min_ver):
                raise ValueError(
                    f"Component {dep_name} version {dep.info.version} "
                    f"is incompatible. Required >= {min_ver}"
                )

        self.register(component)
```

## Когда использовать?

### Подходит для:

1. **Крупных корпоративных приложений** с множеством модулей
2. **Систем с требованиями к переиспользуемости кода**
3. **Plug-in архитектур** и расширяемых систем
4. **Команд, работающих над разными частями системы**

### Не подходит для:

1. **Простых приложений** с минимальной функциональностью
2. **Прототипов и MVP** где важна скорость разработки
3. **Систем с очень жёсткими требованиями к производительности**

## Заключение

Компонентная архитектура — это мощный подход к организации сложных систем, обеспечивающий переиспользуемость, тестируемость и модульность кода. Ключ к успеху — правильное определение границ компонентов, чётких интерфейсов и эффективное управление зависимостями. При правильном применении эта архитектура значительно упрощает разработку и поддержку больших программных систем.
