# SOA (Service-Oriented Architecture)

## Что такое SOA?

Service-Oriented Architecture (SOA) — это архитектурный стиль, в котором приложение строится как набор слабосвязанных сервисов, взаимодействующих через стандартизированные интерфейсы. SOA появилась в конце 1990-х как способ интеграции разнородных корпоративных систем.

```
┌─────────────────────────────────────────────────────────────────┐
│                    SOA Architecture                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌─────────────────────────────────────────────────────────┐  │
│   │                    Enterprise Service Bus (ESB)          │  │
│   │   ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │  │
│   │   │Routing  │ │Transform│ │Protocol │ │Security │       │  │
│   │   │         │ │         │ │Convert  │ │         │       │  │
│   │   └─────────┘ └─────────┘ └─────────┘ └─────────┘       │  │
│   └──────────────────────┬──────────────────────────────────┘  │
│                          │                                      │
│          ┌───────────────┼───────────────┐                     │
│          │               │               │                     │
│          ▼               ▼               ▼                     │
│   ┌─────────────┐ ┌─────────────┐ ┌─────────────┐             │
│   │  Customer   │ │   Order     │ │  Inventory  │             │
│   │  Service    │ │  Service    │ │  Service    │             │
│   │  (SOAP/REST)│ │  (SOAP/REST)│ │  (SOAP/REST)│             │
│   └─────────────┘ └─────────────┘ └─────────────┘             │
│                                                                  │
│   Ключевые принципы:                                            │
│   • Стандартизированные контракты (WSDL, XSD)                  │
│   • Слабая связанность между сервисами                         │
│   • Абстракция реализации                                       │
│   • Повторное использование сервисов                           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Принципы SOA

### 1. Стандартизированные сервисные контракты

```python
# Пример WSDL-подобного контракта в Python
from dataclasses import dataclass
from typing import List, Optional
from enum import Enum


# Определение типов данных (аналог XSD)
class OrderStatus(str, Enum):
    PENDING = "PENDING"
    CONFIRMED = "CONFIRMED"
    SHIPPED = "SHIPPED"
    DELIVERED = "DELIVERED"
    CANCELLED = "CANCELLED"


@dataclass
class Address:
    """Стандартизированный тип адреса"""
    street: str
    city: str
    postal_code: str
    country: str


@dataclass
class OrderItem:
    """Элемент заказа"""
    product_id: str
    product_name: str
    quantity: int
    unit_price: float


@dataclass
class Order:
    """Заказ — стандартизированный тип"""
    order_id: str
    customer_id: str
    items: List[OrderItem]
    shipping_address: Address
    status: OrderStatus
    total_amount: float


# Контракт сервиса (интерфейс)
from abc import ABC, abstractmethod


class OrderServiceContract(ABC):
    """
    Контракт сервиса заказов.
    Определяет операции, которые должен предоставлять сервис.
    """

    @abstractmethod
    def create_order(self, customer_id: str, items: List[OrderItem],
                     shipping_address: Address) -> Order:
        """Создать новый заказ"""
        pass

    @abstractmethod
    def get_order(self, order_id: str) -> Optional[Order]:
        """Получить заказ по ID"""
        pass

    @abstractmethod
    def update_order_status(self, order_id: str, status: OrderStatus) -> Order:
        """Обновить статус заказа"""
        pass

    @abstractmethod
    def cancel_order(self, order_id: str, reason: str) -> bool:
        """Отменить заказ"""
        pass

    @abstractmethod
    def get_customer_orders(self, customer_id: str) -> List[Order]:
        """Получить все заказы клиента"""
        pass
```

### 2. Слабая связанность (Loose Coupling)

```python
# Сервис зависит только от контракта, не от реализации

class InventoryServiceContract(ABC):
    """Контракт сервиса инвентаря"""

    @abstractmethod
    def check_availability(self, product_id: str, quantity: int) -> bool:
        pass

    @abstractmethod
    def reserve_items(self, order_id: str, items: List[dict]) -> str:
        pass

    @abstractmethod
    def release_reservation(self, reservation_id: str) -> bool:
        pass


class PaymentServiceContract(ABC):
    """Контракт платёжного сервиса"""

    @abstractmethod
    def process_payment(self, order_id: str, amount: float,
                        payment_method: dict) -> dict:
        pass

    @abstractmethod
    def refund_payment(self, payment_id: str) -> bool:
        pass


class OrderService(OrderServiceContract):
    """
    Реализация сервиса заказов.
    Зависит от контрактов, не от конкретных реализаций.
    """

    def __init__(self,
                 inventory_service: InventoryServiceContract,
                 payment_service: PaymentServiceContract,
                 order_repository):
        # Внедрение зависимостей через интерфейсы
        self._inventory = inventory_service
        self._payment = payment_service
        self._repository = order_repository

    def create_order(self, customer_id: str, items: List[OrderItem],
                     shipping_address: Address) -> Order:
        # Проверяем доступность через контракт
        for item in items:
            if not self._inventory.check_availability(item.product_id, item.quantity):
                raise ValueError(f"Product {item.product_id} not available")

        # Создаём заказ
        order = Order(
            order_id=self._generate_id(),
            customer_id=customer_id,
            items=items,
            shipping_address=shipping_address,
            status=OrderStatus.PENDING,
            total_amount=sum(i.unit_price * i.quantity for i in items)
        )

        # Резервируем товары
        reservation_id = self._inventory.reserve_items(
            order.order_id,
            [{"product_id": i.product_id, "quantity": i.quantity} for i in items]
        )

        self._repository.save(order)
        return order
```

### 3. Абстракция сервиса

```python
# Сервис скрывает детали реализации

class CustomerService:
    """
    Сервис клиентов абстрагирует работу с разными источниками данных.
    Потребители не знают, откуда берутся данные.
    """

    def __init__(self):
        # Внутренние детали скрыты
        self._legacy_crm = LegacyCRMAdapter()
        self._modern_db = CustomerDatabase()
        self._cache = CustomerCache()

    def get_customer(self, customer_id: str) -> dict:
        """
        Публичный метод — единственное, что видит потребитель.
        Детали реализации скрыты.
        """
        # Сначала проверяем кэш
        cached = self._cache.get(customer_id)
        if cached:
            return cached

        # Пробуем современную БД
        customer = self._modern_db.find(customer_id)
        if customer:
            self._cache.set(customer_id, customer)
            return customer

        # Fallback на legacy CRM
        customer = self._legacy_crm.fetch_customer(customer_id)
        if customer:
            # Синхронизируем с современной БД
            self._modern_db.save(customer)
            self._cache.set(customer_id, customer)

        return customer

    def update_customer(self, customer_id: str, data: dict) -> dict:
        """Обновление клиента — абстрагировано от источников"""
        # Обновляем везде, чтобы поддерживать консистентность
        self._modern_db.update(customer_id, data)
        self._legacy_crm.update_customer(customer_id, data)
        self._cache.invalidate(customer_id)

        return self.get_customer(customer_id)
```

### 4. Повторное использование сервисов

```python
# Сервис аутентификации используется всеми другими сервисами

class AuthenticationService:
    """
    Централизованный сервис аутентификации.
    Используется всеми приложениями в организации.
    """

    def authenticate(self, credentials: dict) -> dict:
        """Аутентификация пользователя"""
        username = credentials.get("username")
        password = credentials.get("password")

        user = self._validate_credentials(username, password)
        if not user:
            raise AuthenticationError("Invalid credentials")

        token = self._generate_token(user)
        return {
            "user_id": user["id"],
            "token": token,
            "expires_at": self._get_expiration()
        }

    def validate_token(self, token: str) -> dict:
        """Валидация токена"""
        payload = self._decode_token(token)
        if not payload or self._is_expired(payload):
            raise AuthenticationError("Invalid or expired token")
        return payload

    def get_user_permissions(self, user_id: str) -> list:
        """Получить права пользователя"""
        return self._permission_store.get_permissions(user_id)


# Использование в других сервисах
class SecureOrderService:
    """Сервис заказов с аутентификацией"""

    def __init__(self, auth_service: AuthenticationService,
                 order_service: OrderServiceContract):
        self._auth = auth_service
        self._orders = order_service

    def create_order(self, token: str, order_data: dict) -> Order:
        # Используем общий сервис аутентификации
        user = self._auth.validate_token(token)

        # Проверяем права
        permissions = self._auth.get_user_permissions(user["user_id"])
        if "create_order" not in permissions:
            raise PermissionError("Not authorized to create orders")

        return self._orders.create_order(
            customer_id=user["user_id"],
            items=order_data["items"],
            shipping_address=order_data["shipping_address"]
        )
```

## Enterprise Service Bus (ESB)

ESB — центральный компонент SOA, обеспечивающий интеграцию сервисов.

```python
# Упрощённая реализация ESB

from typing import Callable, Dict, Any
import json
from abc import ABC, abstractmethod


class Message:
    """Сообщение в ESB"""

    def __init__(self, headers: dict, body: Any):
        self.headers = headers
        self.body = body
        self.id = str(uuid.uuid4())
        self.timestamp = datetime.now()

    def to_dict(self) -> dict:
        return {
            "id": self.id,
            "timestamp": self.timestamp.isoformat(),
            "headers": self.headers,
            "body": self.body
        }


class MessageTransformer(ABC):
    """Трансформатор сообщений"""

    @abstractmethod
    def transform(self, message: Message) -> Message:
        pass


class XMLToJSONTransformer(MessageTransformer):
    """Преобразование XML в JSON"""

    def transform(self, message: Message) -> Message:
        if message.headers.get("content-type") == "application/xml":
            # Преобразуем XML body в JSON
            json_body = self._xml_to_json(message.body)
            message.body = json_body
            message.headers["content-type"] = "application/json"
        return message


class Router:
    """Маршрутизатор сообщений"""

    def __init__(self):
        self.routes: Dict[str, str] = {}
        self.content_based_routes: list = []

    def add_route(self, pattern: str, destination: str):
        """Добавить маршрут по паттерну"""
        self.routes[pattern] = destination

    def add_content_based_route(self, condition: Callable, destination: str):
        """Добавить маршрут на основе содержимого"""
        self.content_based_routes.append((condition, destination))

    def route(self, message: Message) -> str:
        """Определить назначение сообщения"""
        # Проверяем content-based routes
        for condition, destination in self.content_based_routes:
            if condition(message):
                return destination

        # Проверяем pattern-based routes
        service = message.headers.get("target-service")
        if service in self.routes:
            return self.routes[service]

        raise RoutingError(f"No route for message: {message.id}")


class ServiceBus:
    """Enterprise Service Bus"""

    def __init__(self):
        self.services: Dict[str, Any] = {}
        self.transformers: list = []
        self.router = Router()
        self.interceptors: list = []

    def register_service(self, name: str, service: Any):
        """Зарегистрировать сервис"""
        self.services[name] = service

    def add_transformer(self, transformer: MessageTransformer):
        """Добавить трансформатор"""
        self.transformers.append(transformer)

    def add_interceptor(self, interceptor: Callable):
        """Добавить перехватчик (для логирования, безопасности и т.д.)"""
        self.interceptors.append(interceptor)

    def send(self, message: Message) -> Any:
        """Отправить сообщение через ESB"""
        # Применяем перехватчики
        for interceptor in self.interceptors:
            message = interceptor(message)
            if message is None:
                raise SecurityError("Message blocked by interceptor")

        # Трансформируем сообщение
        for transformer in self.transformers:
            message = transformer.transform(message)

        # Маршрутизируем
        destination = self.router.route(message)

        # Доставляем
        service = self.services.get(destination)
        if not service:
            raise ServiceNotFoundError(f"Service not found: {destination}")

        return self._invoke_service(service, message)

    def _invoke_service(self, service: Any, message: Message) -> Any:
        """Вызвать сервис"""
        operation = message.headers.get("operation")
        method = getattr(service, operation, None)

        if not method:
            raise OperationNotFoundError(
                f"Operation {operation} not found in service"
            )

        return method(**message.body)


# Пример использования ESB
esb = ServiceBus()

# Регистрация сервисов
esb.register_service("orders", OrderService())
esb.register_service("customers", CustomerService())
esb.register_service("inventory", InventoryService())

# Настройка маршрутизации
esb.router.add_route("orders", "orders")
esb.router.add_route("customers", "customers")

# Content-based routing
esb.router.add_content_based_route(
    lambda m: m.body.get("priority") == "high",
    "priority-orders"
)

# Добавляем логирование
def logging_interceptor(message: Message) -> Message:
    print(f"[ESB] Processing message {message.id} to {message.headers.get('target-service')}")
    return message

esb.add_interceptor(logging_interceptor)

# Отправка сообщения
message = Message(
    headers={
        "target-service": "orders",
        "operation": "create_order",
        "content-type": "application/json"
    },
    body={
        "customer_id": "CUST-123",
        "items": [{"product_id": "PROD-1", "quantity": 2}]
    }
)

result = esb.send(message)
```

## SOAP Web Services

SOAP — традиционный протокол для SOA.

```python
# Пример SOAP сервиса с использованием zeep (клиент) и spyne (сервер)

# Сервер (spyne)
from spyne import Application, rpc, ServiceBase, Unicode, Integer, Array
from spyne.protocol.soap import Soap11
from spyne.server.wsgi import WsgiApplication


class OrderWebService(ServiceBase):
    """SOAP веб-сервис для заказов"""

    @rpc(Unicode, Array(Unicode), _returns=Unicode)
    def create_order(ctx, customer_id, product_ids):
        """
        Операция создания заказа.
        WSDL автоматически генерируется из сигнатуры метода.
        """
        order_id = generate_order_id()
        # Бизнес-логика создания заказа
        return order_id

    @rpc(Unicode, _returns=Unicode)
    def get_order_status(ctx, order_id):
        """Получить статус заказа"""
        # Логика получения статуса
        return "CONFIRMED"

    @rpc(Unicode, Unicode, _returns=Unicode)
    def update_order_status(ctx, order_id, new_status):
        """Обновить статус заказа"""
        # Логика обновления
        return "SUCCESS"


# Создание WSGI приложения
application = Application(
    [OrderWebService],
    'http://example.com/orders',
    in_protocol=Soap11(validator='lxml'),
    out_protocol=Soap11()
)

wsgi_application = WsgiApplication(application)


# Клиент (zeep)
from zeep import Client


def call_soap_service():
    """Вызов SOAP сервиса"""
    client = Client('http://localhost:8000/orders?wsdl')

    # Создание заказа
    order_id = client.service.create_order(
        customer_id="CUST-123",
        product_ids=["PROD-1", "PROD-2"]
    )

    # Получение статуса
    status = client.service.get_order_status(order_id=order_id)

    print(f"Order {order_id} status: {status}")
```

## SOA vs Микросервисы

```
┌────────────────────────────────────────────────────────────────────────┐
│                      SOA vs Микросервисы                                │
├──────────────────────────────┬─────────────────────────────────────────┤
│            SOA               │           Микросервисы                  │
├──────────────────────────────┼─────────────────────────────────────────┤
│                              │                                         │
│  Централизованное управление │  Децентрализованное управление         │
│  через ESB                   │  Сервисы общаются напрямую             │
│                              │                                         │
│  Enterprise-масштаб          │  Любой масштаб                         │
│                              │                                         │
│  SOAP, WS-* стандарты       │  REST, gRPC, любые протоколы           │
│                              │                                         │
│  Сложные трансформации      │  Простые API                            │
│  сообщений                   │                                         │
│                              │                                         │
│  Общая база данных возможна │  Database per service                   │
│                              │                                         │
│  Оркестрация через ESB      │  Хореография событий                    │
│                              │                                         │
│  Сервисы могут быть          │  Сервисы всегда маленькие              │
│  крупными                    │  и сфокусированные                     │
│                              │                                         │
└──────────────────────────────┴─────────────────────────────────────────┘
```

## Плюсы и минусы SOA

### Плюсы

1. **Стандартизация** — чёткие контракты и протоколы
2. **Интеграция** — объединение разнородных систем
3. **Повторное использование** — сервисы используются многими приложениями
4. **Управляемость** — централизованное управление через ESB
5. **Масштабируемость** — сервисы можно масштабировать независимо

### Минусы

1. **Сложность ESB** — ESB становится узким местом и точкой отказа
2. **Overhead** — SOAP/XML создают накладные расходы
3. **Связанность через ESB** — централизация создаёт зависимости
4. **Стоимость** — дорогие enterprise инструменты
5. **Медленное развёртывание** — изменения требуют координации

## Когда использовать SOA

### Используйте SOA когда:

- Enterprise-среда с множеством legacy систем
- Нужна интеграция разнородных систем
- Важна стандартизация и governance
- Есть команда для поддержки ESB
- Нужны сложные трансформации данных

### Не используйте SOA когда:

- Greenfield проект без legacy
- Небольшая команда
- Нужна высокая скорость развёртывания
- Бюджет ограничен
- Нет необходимости в ESB

## Заключение

SOA — архитектурный стиль для интеграции корпоративных систем. Хотя сегодня микросервисы более популярны, принципы SOA (слабая связанность, контракты, повторное использование) остаются актуальными. В крупных организациях SOA и микросервисы часто сосуществуют.
