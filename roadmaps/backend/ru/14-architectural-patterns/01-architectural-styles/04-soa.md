# Сервис-ориентированная архитектура (Service-Oriented Architecture, SOA)

## Определение

**Service-Oriented Architecture (SOA)** — это архитектурный стиль, в котором приложение строится как набор слабо связанных, переиспользуемых сервисов, взаимодействующих через стандартизированные интерфейсы и протоколы. Каждый сервис представляет бизнес-возможность (business capability) и может быть использован разными приложениями.

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         ENTERPRISE SERVICE BUS (ESB)                    │
├─────────────────────────────────────────────────────────────────────────┤
│   Routing  │  Transformation  │  Orchestration  │  Protocol Mediation  │
└───────┬─────────────┬──────────────┬──────────────┬─────────────────────┘
        │             │              │              │
        ▼             ▼              ▼              ▼
┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
│  Customer │  │   Order   │  │  Payment  │  │ Inventory │
│  Service  │  │  Service  │  │  Service  │  │  Service  │
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      │              │              │              │
      ▼              ▼              ▼              ▼
┌───────────┐  ┌───────────┐  ┌───────────┐  ┌───────────┐
│  Customer │  │   Order   │  │  Payment  │  │ Inventory │
│    DB     │  │    DB     │  │    DB     │  │    DB     │
└───────────┘  └───────────┘  └───────────┘  └───────────┘
```

SOA возникла в конце 1990-х — начале 2000-х как ответ на сложность интеграции корпоративных систем.

## Ключевые характеристики

### 1. Основные принципы SOA

| Принцип | Описание |
|---------|----------|
| **Loose Coupling** | Минимальные зависимости между сервисами |
| **Service Contract** | Формальное описание интерфейса (WSDL, OpenAPI) |
| **Autonomy** | Сервис контролирует свою логику и данные |
| **Abstraction** | Скрытие внутренней реализации |
| **Reusability** | Переиспользование сервисов разными потребителями |
| **Composability** | Компоновка сервисов в бизнес-процессы |
| **Statelessness** | Минимизация хранения состояния |
| **Discoverability** | Возможность обнаружения сервисов (Service Registry) |

### 2. Компоненты SOA

```
┌─────────────────────────────────────────────────────────────────┐
│                      SOA COMPONENTS                              │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌────────────────┐     ┌────────────────┐                      │
│  │ Service        │     │ Service        │                      │
│  │ Provider       │────▶│ Registry       │◀────┐                │
│  │ (Publisher)    │     │ (UDDI)         │     │                │
│  └────────────────┘     └───────┬────────┘     │                │
│                                 │              │                │
│                          Find   │              │ Bind           │
│                                 ▼              │                │
│                         ┌────────────────┐     │                │
│                         │ Service        │─────┘                │
│                         │ Consumer       │                      │
│                         │ (Requestor)    │                      │
│                         └────────────────┘                      │
│                                                                  │
│  ESB (Enterprise Service Bus):                                  │
│  • Message routing                                              │
│  • Protocol transformation                                       │
│  • Message enhancement                                          │
│  • Service orchestration                                        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 3. Типы сервисов

```
┌─────────────────────────────────────────────────────────────────┐
│                    SERVICE HIERARCHY                             │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Business Process Services                   │    │
│  │         (Orchestration, Composite Services)              │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Business Services                           │    │
│  │         (Customer Service, Order Service)                │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Technical Services                          │    │
│  │         (Auth, Logging, Messaging)                       │    │
│  └───────────────────────────┬─────────────────────────────┘    │
│                              │                                   │
│  ┌─────────────────────────────────────────────────────────┐    │
│  │              Infrastructure Services                     │    │
│  │         (Database, File Storage, Network)                │    │
│  └─────────────────────────────────────────────────────────┘    │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### 4. Протоколы и стандарты

| Категория | Стандарты |
|-----------|-----------|
| **Messaging** | SOAP, XML-RPC, REST |
| **Description** | WSDL (Web Service Description Language) |
| **Discovery** | UDDI (Universal Description, Discovery, Integration) |
| **Security** | WS-Security, SAML, OAuth |
| **Reliability** | WS-ReliableMessaging |
| **Transactions** | WS-AtomicTransaction, WS-Coordination |

## Когда использовать

### Идеальные сценарии

| Сценарий | Почему SOA подходит |
|----------|---------------------|
| **Enterprise интеграция** | Объединение разнородных систем |
| **B2B взаимодействие** | Стандартизированные интерфейсы |
| **Legacy modernization** | Обёртка старых систем сервисами |
| **Переиспользование функций** | Общие сервисы для разных приложений |
| **Регулируемые отрасли** | Банки, страхование, госсектор |
| **Длительные бизнес-процессы** | BPEL оркестрация |

### Типичные enterprise use cases

```
┌─────────────────────────────────────────────────────────────────┐
│                    ENTERPRISE SOA EXAMPLE                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│     ┌─────────┐    ┌─────────┐    ┌─────────┐    ┌─────────┐   │
│     │   CRM   │    │   ERP   │    │   SCM   │    │   HRM   │   │
│     │ System  │    │ System  │    │ System  │    │ System  │   │
│     └────┬────┘    └────┬────┘    └────┬────┘    └────┬────┘   │
│          │              │              │              │         │
│          └──────────────┴──────────────┴──────────────┘         │
│                              │                                   │
│                              ▼                                   │
│                    ┌─────────────────┐                          │
│                    │       ESB       │                          │
│                    └─────────────────┘                          │
│                              │                                   │
│          ┌──────────────────┼──────────────────┐                │
│          │                  │                  │                │
│          ▼                  ▼                  ▼                │
│    ┌───────────┐     ┌───────────┐     ┌───────────┐           │
│    │ Customer  │     │   Order   │     │ Inventory │           │
│    │  Service  │     │  Service  │     │  Service  │           │
│    └───────────┘     └───────────┘     └───────────┘           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Не подходит для

- Простые standalone приложения
- Стартапы с быстро меняющимися требованиями
- Системы с жёсткими требованиями к latency
- Маленькие команды без enterprise инфраструктуры

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Интероперабельность** | Интеграция разных технологий и платформ |
| **Переиспользование** | Сервисы доступны для многих потребителей |
| **Гибкость** | Замена реализации без изменения интерфейса |
| **Maintainability** | Независимое обслуживание сервисов |
| **Governance** | Централизованное управление политиками |
| **Стандартизация** | Единые протоколы и форматы |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Сложность** | ESB, реестры, оркестрация — много инфраструктуры |
| **Performance** | XML parsing, сетевые вызовы, ESB overhead |
| **Vendor lock-in** | Зависимость от ESB платформ (IBM, Oracle) |
| **Overengineering** | Избыточность для простых задач |
| **ESB bottleneck** | Единая точка отказа и узкое место |
| **Governance overhead** | Сложное управление версиями и политиками |

## Примеры реализации

### SOAP Web Service (Python)

```python
# order_service.py
from spyne import Application, Service, Unicode, Integer, Array, ComplexModel
from spyne.decorator import rpc
from spyne.protocol.soap import Soap11
from spyne.server.wsgi import WsgiApplication
from datetime import datetime

# Complex types (XSD)
class OrderItem(ComplexModel):
    product_id = Integer
    quantity = Integer
    price = Unicode

class Order(ComplexModel):
    order_id = Integer
    customer_id = Integer
    items = Array(OrderItem)
    total = Unicode
    status = Unicode
    created_at = Unicode

class CreateOrderRequest(ComplexModel):
    customer_id = Integer
    items = Array(OrderItem)

class CreateOrderResponse(ComplexModel):
    success = Unicode
    order = Order
    message = Unicode

# SOAP Service
class OrderService(Service):
    """
    WSDL будет автоматически сгенерирован:
    http://localhost:8000/?wsdl
    """

    @rpc(CreateOrderRequest, _returns=CreateOrderResponse)
    def CreateOrder(ctx, request):
        """Создание заказа через SOAP"""
        # Бизнес-логика
        order = Order(
            order_id=12345,
            customer_id=request.customer_id,
            items=request.items,
            total="199.99",
            status="PENDING",
            created_at=datetime.now().isoformat()
        )

        return CreateOrderResponse(
            success="true",
            order=order,
            message="Order created successfully"
        )

    @rpc(Integer, _returns=Order)
    def GetOrder(ctx, order_id):
        """Получение заказа по ID"""
        # В реальности — запрос к БД
        return Order(
            order_id=order_id,
            customer_id=100,
            items=[],
            total="199.99",
            status="PENDING",
            created_at=datetime.now().isoformat()
        )

    @rpc(Integer, Unicode, _returns=Unicode)
    def UpdateOrderStatus(ctx, order_id, status):
        """Обновление статуса заказа"""
        # Бизнес-логика обновления
        return f"Order {order_id} status updated to {status}"

# Application setup
application = Application(
    [OrderService],
    tns='http://example.com/order',
    in_protocol=Soap11(validator='lxml'),
    out_protocol=Soap11()
)

wsgi_application = WsgiApplication(application)

if __name__ == '__main__':
    from wsgiref.simple_server import make_server
    server = make_server('0.0.0.0', 8000, wsgi_application)
    print("SOAP Service running on http://localhost:8000")
    print("WSDL available at http://localhost:8000/?wsdl")
    server.serve_forever()
```

### SOAP Client

```python
# soap_client.py
from zeep import Client
from zeep.transports import Transport
from requests import Session

class OrderServiceClient:
    """SOAP клиент для Order Service"""

    def __init__(self, wsdl_url: str):
        session = Session()
        session.verify = True  # SSL verification
        transport = Transport(session=session, timeout=30)
        self.client = Client(wsdl_url, transport=transport)

    def create_order(self, customer_id: int, items: list) -> dict:
        """Создание заказа"""
        # Создание типа из WSDL
        OrderItem = self.client.get_type('ns0:OrderItem')
        CreateOrderRequest = self.client.get_type('ns0:CreateOrderRequest')

        order_items = [
            OrderItem(product_id=item['product_id'],
                     quantity=item['quantity'],
                     price=str(item['price']))
            for item in items
        ]

        request = CreateOrderRequest(
            customer_id=customer_id,
            items=order_items
        )

        response = self.client.service.CreateOrder(request)
        return {
            'success': response.success == 'true',
            'order_id': response.order.order_id,
            'message': response.message
        }

    def get_order(self, order_id: int) -> dict:
        """Получение заказа"""
        order = self.client.service.GetOrder(order_id)
        return {
            'order_id': order.order_id,
            'status': order.status,
            'total': order.total
        }

# Использование
if __name__ == '__main__':
    client = OrderServiceClient('http://localhost:8000/?wsdl')

    # Создание заказа
    result = client.create_order(
        customer_id=100,
        items=[
            {'product_id': 1, 'quantity': 2, 'price': 49.99},
            {'product_id': 2, 'quantity': 1, 'price': 99.99}
        ]
    )
    print(f"Order created: {result}")
```

### Service Registry Pattern

```python
# service_registry.py
from dataclasses import dataclass, field
from typing import Dict, List, Optional
from datetime import datetime
import threading

@dataclass
class ServiceEndpoint:
    service_name: str
    version: str
    url: str
    protocol: str  # SOAP, REST, gRPC
    wsdl_url: Optional[str] = None
    health_check_url: Optional[str] = None
    metadata: Dict = field(default_factory=dict)
    registered_at: datetime = field(default_factory=datetime.now)
    last_health_check: Optional[datetime] = None
    is_healthy: bool = True

class ServiceRegistry:
    """
    Реестр сервисов (упрощённая версия UDDI)
    В реальности используют Consul, Eureka, etcd
    """

    def __init__(self):
        self._services: Dict[str, List[ServiceEndpoint]] = {}
        self._lock = threading.Lock()

    def register(self, endpoint: ServiceEndpoint) -> None:
        """Регистрация сервиса в реестре"""
        with self._lock:
            if endpoint.service_name not in self._services:
                self._services[endpoint.service_name] = []
            self._services[endpoint.service_name].append(endpoint)
            print(f"Registered: {endpoint.service_name} v{endpoint.version} at {endpoint.url}")

    def deregister(self, service_name: str, url: str) -> None:
        """Удаление сервиса из реестра"""
        with self._lock:
            if service_name in self._services:
                self._services[service_name] = [
                    ep for ep in self._services[service_name]
                    if ep.url != url
                ]

    def discover(
        self,
        service_name: str,
        version: Optional[str] = None
    ) -> List[ServiceEndpoint]:
        """Поиск сервисов по имени и версии"""
        with self._lock:
            endpoints = self._services.get(service_name, [])

            if version:
                endpoints = [ep for ep in endpoints if ep.version == version]

            # Только здоровые сервисы
            return [ep for ep in endpoints if ep.is_healthy]

    def get_endpoint(self, service_name: str) -> Optional[ServiceEndpoint]:
        """Получение одного endpoint (load balancing)"""
        healthy = self.discover(service_name)
        if not healthy:
            return None
        # Round-robin или другой алгоритм
        return healthy[0]

# Использование
registry = ServiceRegistry()

# Регистрация сервисов
registry.register(ServiceEndpoint(
    service_name="OrderService",
    version="1.0",
    url="http://orders-v1.internal:8000",
    protocol="SOAP",
    wsdl_url="http://orders-v1.internal:8000/?wsdl"
))

registry.register(ServiceEndpoint(
    service_name="OrderService",
    version="2.0",
    url="http://orders-v2.internal:8000",
    protocol="REST",
    health_check_url="http://orders-v2.internal:8000/health"
))

# Discovery
order_services = registry.discover("OrderService", version="1.0")
```

### Message Broker / ESB Pattern (упрощённая версия)

```python
# simple_esb.py
import asyncio
from dataclasses import dataclass
from typing import Dict, Callable, Any, List
from enum import Enum
import json

class MessageType(Enum):
    REQUEST = "request"
    RESPONSE = "response"
    EVENT = "event"

@dataclass
class Message:
    message_type: MessageType
    source: str
    destination: str
    payload: Dict[str, Any]
    correlation_id: str
    headers: Dict[str, str] = None

class SimpleESB:
    """
    Упрощённая реализация Enterprise Service Bus
    В реальности используют MuleSoft, IBM Integration Bus, Apache Camel
    """

    def __init__(self):
        self._routes: Dict[str, str] = {}  # logical -> physical
        self._transformers: Dict[str, Callable] = {}
        self._handlers: Dict[str, Callable] = {}
        self._message_queue: asyncio.Queue = asyncio.Queue()

    def add_route(self, logical_name: str, physical_endpoint: str):
        """Добавление маршрута"""
        self._routes[logical_name] = physical_endpoint

    def add_transformer(
        self,
        from_format: str,
        to_format: str,
        transformer: Callable
    ):
        """Добавление трансформера сообщений"""
        key = f"{from_format}:{to_format}"
        self._transformers[key] = transformer

    def register_handler(self, destination: str, handler: Callable):
        """Регистрация обработчика для destination"""
        self._handlers[destination] = handler

    async def send(self, message: Message):
        """Отправка сообщения через ESB"""
        # 1. Routing — определение физического адреса
        physical = self._routes.get(message.destination, message.destination)

        # 2. Transformation — преобразование формата при необходимости
        source_format = message.headers.get('content-type', 'json') if message.headers else 'json'
        dest_format = self._get_destination_format(physical)

        if source_format != dest_format:
            transformer_key = f"{source_format}:{dest_format}"
            if transformer_key in self._transformers:
                message.payload = self._transformers[transformer_key](message.payload)

        # 3. Delivery — доставка сообщения
        if physical in self._handlers:
            await self._handlers[physical](message)
        else:
            await self._message_queue.put((physical, message))

        print(f"ESB: Routed {message.source} -> {physical}")

    def _get_destination_format(self, destination: str) -> str:
        # В реальности — lookup в registry
        return 'json'

# Пример использования
async def main():
    esb = SimpleESB()

    # Настройка маршрутов
    esb.add_route("OrderService", "http://orders.internal:8000")
    esb.add_route("PaymentService", "http://payments.internal:8001")
    esb.add_route("NotificationService", "http://notifications.internal:8002")

    # Трансформер XML -> JSON
    esb.add_transformer("xml", "json", lambda x: json.loads(x))

    # Обработчики
    async def order_handler(msg: Message):
        print(f"OrderService received: {msg.payload}")

    esb.register_handler("http://orders.internal:8000", order_handler)

    # Отправка сообщения
    await esb.send(Message(
        message_type=MessageType.REQUEST,
        source="WebApp",
        destination="OrderService",
        payload={"action": "create", "customer_id": 100},
        correlation_id="uuid-123"
    ))

asyncio.run(main())
```

## Best practices и антипаттерны

### Best Practices

#### 1. Contract-First Development
```yaml
# Сначала определяем контракт (OpenAPI / WSDL)
openapi: 3.0.0
info:
  title: Order Service API
  version: 1.0.0
paths:
  /orders:
    post:
      operationId: createOrder
      requestBody:
        content:
          application/json:
            schema:
              $ref: '#/components/schemas/CreateOrderRequest'
      responses:
        '201':
          description: Order created
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/Order'
```

#### 2. Версионирование сервисов
```python
# URL versioning
/api/v1/orders
/api/v2/orders

# Header versioning
Accept: application/vnd.example.v2+json

# WSDL namespace versioning
tns='http://example.com/order/v2'
```

#### 3. Service Granularity
```
ХОРОШО: Business-aligned services
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│   Customer    │  │     Order     │  │    Payment    │
│   Service     │  │    Service    │  │   Service     │
└───────────────┘  └───────────────┘  └───────────────┘

ПЛОХО: CRUD-oriented services (слишком мелко)
┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐
│ Create  │ │  Read   │ │ Update  │ │ Delete  │
│ Service │ │ Service │ │ Service │ │ Service │
└─────────┘ └─────────┘ └─────────┘ └─────────┘
```

#### 4. Idempotency
```python
class OrderService:
    def create_order(self, idempotency_key: str, request: CreateOrderRequest):
        # Проверка идемпотентности
        existing = self.cache.get(f"order:{idempotency_key}")
        if existing:
            return existing  # Возврат существующего результата

        # Создание заказа
        order = self._create_order_internal(request)

        # Сохранение для идемпотентности
        self.cache.set(f"order:{idempotency_key}", order, ttl=3600)

        return order
```

### Антипаттерны

#### 1. ESB as Business Logic Container
```python
# ПЛОХО: бизнес-логика в ESB
class ESBOrderProcessor:
    def process(self, order):
        # Вся логика в ESB — антипаттерн!
        if order.total > 1000:
            order.discount = 0.1
        order.tax = self.calculate_tax(order)
        order.shipping = self.calculate_shipping(order)
        # ...

# ХОРОШО: ESB только для маршрутизации и трансформации
# Бизнес-логика в сервисах
```

#### 2. Chatty Services
```python
# ПЛОХО: много мелких вызовов
customer = customer_service.get_customer(id)
address = address_service.get_address(customer.address_id)
orders = order_service.get_orders(customer.id)
payments = payment_service.get_payments(customer.id)

# ХОРОШО: агрегирующий сервис
customer_profile = customer_service.get_full_profile(id)
# Возвращает всё сразу
```

#### 3. Point-to-Point Integration
```
ПЛОХО: Spaghetti integration
┌───┐    ┌───┐
│ A │───▶│ B │
└─┬─┘    └─┬─┘
  │        │
  ▼        ▼
┌───┐    ┌───┐
│ C │◀───│ D │
└───┘    └───┘

ХОРОШО: Hub-and-spoke через ESB
    ┌───┐
    │ A │
    └─┬─┘
      │
┌─────▼─────┐
│    ESB    │
└───────────┘
  │   │   │
  ▼   ▼   ▼
┌───┐┌───┐┌───┐
│ B ││ C ││ D │
└───┘└───┘└───┘
```

## Связанные паттерны

| Паттерн | Связь |
|---------|-------|
| **Microservices** | Эволюция SOA, более лёгкий вес |
| **ESB Pattern** | Центральный компонент SOA |
| **Service Mesh** | Современная альтернатива ESB |
| **API Gateway** | Единая точка входа для клиентов |
| **CQRS** | Часто используется в SOA |
| **Event-Driven Architecture** | Комплементарный подход |
| **Saga Pattern** | Распределённые транзакции в SOA |

## SOA vs Microservices

| Аспект | SOA | Microservices |
|--------|-----|---------------|
| **Размер сервисов** | Крупные, enterprise-level | Мелкие, single responsibility |
| **Коммуникация** | ESB, SOAP | Lightweight (REST, gRPC) |
| **Данные** | Shared database допустим | Database per service |
| **Governance** | Централизованный | Децентрализованный |
| **Технологии** | Стандартизированные | Polyglot |
| **Оркестрация** | ESB, BPEL | Choreography, lightweight |

## Ресурсы для изучения

### Книги
- **"SOA Patterns"** — Arnon Rotem-Gal-Oz
- **"Enterprise Integration Patterns"** — Gregor Hohpe, Bobby Woolf
- **"SOA Design Patterns"** — Thomas Erl

### Статьи
- [Service-Oriented Architecture — Microsoft](https://docs.microsoft.com/en-us/dotnet/architecture/microservices/architect-microservice-container-applications/service-oriented-architecture)
- [SOA vs Microservices — Martin Fowler](https://martinfowler.com/articles/microservices.html)

### Стандарты
- [SOAP 1.2 Specification](https://www.w3.org/TR/soap12/)
- [WSDL 2.0](https://www.w3.org/TR/wsdl20/)
- [WS-* Specifications](https://www.oasis-open.org/)

---

## Резюме

SOA — это enterprise-подход к интеграции, характерный для крупных организаций:

- **Ключевые концепции**: сервисы, контракты, ESB, реестр
- **Преимущества**: интероперабельность, переиспользование, governance
- **Недостатки**: сложность, overhead, vendor lock-in
- **Эволюция**: многие идеи SOA перешли в микросервисы в упрощённом виде

SOA по-прежнему актуальна для enterprise-интеграций, но для новых проектов чаще выбирают микросервисную архитектуру.
