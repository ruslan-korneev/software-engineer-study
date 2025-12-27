# Облачные паттерны проектирования (Cloud Design Patterns)

## Введение

Облачные паттерны проектирования — это проверенные решения для типичных проблем, возникающих при разработке распределённых систем в облаке. Эти паттерны помогают решать задачи масштабируемости, отказоустойчивости, безопасности и управляемости.

Облачные системы характеризуются:
- Распределённой природой компонентов
- Высокой вероятностью частичных отказов
- Необходимостью эластичного масштабирования
- Множеством точек интеграции
- Требованиями к независимому развёртыванию

Рассмотрим ключевые паттерны, которые помогают справляться с этими вызовами.

---

## 1. Ambassador Pattern

### Описание

Ambassador (посол) — это паттерн, при котором вспомогательный сервис выступает прокси между основным приложением и внешними сервисами. Ambassador обрабатывает сетевые аспекты: retry, circuit breaking, мониторинг, аутентификацию.

### Когда использовать

- Необходима унификация сетевой логики для разных приложений
- Приложение написано на языке с ограниченной поддержкой сетевых библиотек
- Требуется централизованное управление политиками подключения
- Нужно изолировать сетевую логику от бизнес-логики

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                         Pod                                  │
│  ┌─────────────────┐      ┌─────────────────────────────┐   │
│  │   Application   │──────│        Ambassador           │   │
│  │   Container     │      │   (Envoy/Custom Proxy)      │   │
│  │                 │      │  - Retry logic              │   │
│  │  Business Logic │      │  - Circuit breaking         │   │
│  │                 │      │  - Rate limiting            │   │
│  │                 │      │  - Metrics collection       │   │
│  │                 │      │  - TLS termination          │   │
│  └─────────────────┘      └────────────┬────────────────┘   │
│                                        │                     │
└────────────────────────────────────────┼─────────────────────┘
                                         │
                                         ▼
                              ┌──────────────────┐
                              │  External APIs   │
                              │  Database        │
                              │  Message Queue   │
                              └──────────────────┘
```

### Реализация

```python
# ambassador.py - Custom Ambassador Service
from fastapi import FastAPI, Request, Response
import httpx
import asyncio
from circuitbreaker import circuit
from typing import Optional
import time
from prometheus_client import Counter, Histogram

app = FastAPI()

# Метрики
request_counter = Counter('ambassador_requests_total', 'Total requests', ['service', 'method', 'status'])
request_latency = Histogram('ambassador_request_latency_seconds', 'Request latency', ['service'])

# Конфигурация сервисов
SERVICE_CONFIG = {
    "payment-api": {
        "base_url": "https://payment.example.com",
        "timeout": 10.0,
        "retries": 3,
        "circuit_breaker": {"failure_threshold": 5, "recovery_timeout": 30}
    },
    "inventory-api": {
        "base_url": "https://inventory.example.com",
        "timeout": 5.0,
        "retries": 2,
        "circuit_breaker": {"failure_threshold": 3, "recovery_timeout": 60}
    }
}

class CircuitBreaker:
    def __init__(self, failure_threshold: int = 5, recovery_timeout: int = 30):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.failure_count = 0
        self.last_failure_time = None
        self.state = "closed"  # closed, open, half-open

    def can_execute(self) -> bool:
        if self.state == "closed":
            return True
        if self.state == "open":
            if time.time() - self.last_failure_time >= self.recovery_timeout:
                self.state = "half-open"
                return True
            return False
        return True  # half-open - пробуем один запрос

    def record_success(self):
        self.failure_count = 0
        self.state = "closed"

    def record_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()
        if self.failure_count >= self.failure_threshold:
            self.state = "open"

# Circuit breakers для каждого сервиса
circuit_breakers = {
    service: CircuitBreaker(**config["circuit_breaker"])
    for service, config in SERVICE_CONFIG.items()
}

async def retry_request(
    client: httpx.AsyncClient,
    method: str,
    url: str,
    retries: int = 3,
    **kwargs
) -> httpx.Response:
    """Выполняет запрос с exponential backoff retry"""
    last_exception = None

    for attempt in range(retries):
        try:
            response = await client.request(method, url, **kwargs)
            if response.status_code < 500:  # Не retry на клиентских ошибках
                return response
            last_exception = Exception(f"Server error: {response.status_code}")
        except httpx.RequestError as e:
            last_exception = e

        if attempt < retries - 1:
            wait_time = (2 ** attempt) * 0.5  # 0.5s, 1s, 2s...
            await asyncio.sleep(wait_time)

    raise last_exception

@app.api_route("/{service}/{path:path}", methods=["GET", "POST", "PUT", "DELETE", "PATCH"])
async def proxy_request(service: str, path: str, request: Request):
    """Проксирует запросы к внешним сервисам"""

    if service not in SERVICE_CONFIG:
        return Response(content="Service not found", status_code=404)

    config = SERVICE_CONFIG[service]
    cb = circuit_breakers[service]

    # Проверяем circuit breaker
    if not cb.can_execute():
        request_counter.labels(service=service, method=request.method, status="circuit_open").inc()
        return Response(
            content='{"error": "Service temporarily unavailable"}',
            status_code=503,
            media_type="application/json"
        )

    # Получаем тело запроса
    body = await request.body()

    # Формируем headers (убираем hop-by-hop)
    headers = dict(request.headers)
    for h in ["host", "connection", "keep-alive", "transfer-encoding"]:
        headers.pop(h, None)

    target_url = f"{config['base_url']}/{path}"

    start_time = time.time()
    try:
        async with httpx.AsyncClient(timeout=config["timeout"]) as client:
            response = await retry_request(
                client,
                request.method,
                target_url,
                retries=config["retries"],
                headers=headers,
                content=body,
                params=dict(request.query_params)
            )

        cb.record_success()
        request_counter.labels(
            service=service,
            method=request.method,
            status=response.status_code
        ).inc()

        return Response(
            content=response.content,
            status_code=response.status_code,
            headers=dict(response.headers)
        )

    except Exception as e:
        cb.record_failure()
        request_counter.labels(service=service, method=request.method, status="error").inc()
        return Response(
            content=f'{{"error": "{str(e)}"}}',
            status_code=502,
            media_type="application/json"
        )
    finally:
        request_latency.labels(service=service).observe(time.time() - start_time)
```

### Kubernetes Deployment с Envoy Ambassador

```yaml
# deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-app
spec:
  template:
    spec:
      containers:
        # Основное приложение
        - name: app
          image: my-app:latest
          ports:
            - containerPort: 8080
          env:
            - name: EXTERNAL_SERVICE_URL
              value: "http://localhost:8001"  # Запросы идут через Ambassador

        # Ambassador (Envoy sidecar)
        - name: ambassador
          image: envoyproxy/envoy:v1.25.0
          ports:
            - containerPort: 8001
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
      volumes:
        - name: envoy-config
          configMap:
            name: envoy-config

---
# envoy-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: envoy-config
data:
  envoy.yaml: |
    static_resources:
      listeners:
        - name: listener_0
          address:
            socket_address:
              address: 0.0.0.0
              port_value: 8001
          filter_chains:
            - filters:
                - name: envoy.filters.network.http_connection_manager
                  typed_config:
                    "@type": type.googleapis.com/envoy.extensions.filters.network.http_connection_manager.v3.HttpConnectionManager
                    stat_prefix: ingress_http
                    route_config:
                      name: local_route
                      virtual_hosts:
                        - name: external_service
                          domains: ["*"]
                          routes:
                            - match: { prefix: "/" }
                              route:
                                cluster: external_service
                                retry_policy:
                                  retry_on: "5xx,reset,connect-failure"
                                  num_retries: 3
                    http_filters:
                      - name: envoy.filters.http.router

      clusters:
        - name: external_service
          connect_timeout: 5s
          type: STRICT_DNS
          lb_policy: ROUND_ROBIN
          circuit_breakers:
            thresholds:
              - max_connections: 100
                max_pending_requests: 100
                max_requests: 100
          load_assignment:
            cluster_name: external_service
            endpoints:
              - lb_endpoints:
                  - endpoint:
                      address:
                        socket_address:
                          address: external-api.example.com
                          port_value: 443
```

---

## 2. Anti-Corruption Layer (ACL)

### Описание

Anti-Corruption Layer (антикоррупционный слой) — это паттерн, который изолирует вашу доменную модель от внешних или legacy систем. ACL переводит данные и запросы между двумя разными моделями, предотвращая "заражение" вашего кода чужими абстракциями.

### Когда использовать

- Интеграция с legacy системами с устаревшей архитектурой
- Работа с внешними API, модель которых не соответствует вашей
- Миграция с одной системы на другую
- Защита domain model от изменений во внешних системах

### Архитектура

```
┌─────────────────────────────────────────────────────────────────────┐
│                        Your Bounded Context                          │
│  ┌────────────────┐     ┌─────────────────────────────────────────┐ │
│  │  Domain Model  │◄────│         Anti-Corruption Layer           │ │
│  │                │     │  ┌─────────────┐  ┌──────────────────┐  │ │
│  │  User          │     │  │  Translator │  │     Adapter      │  │ │
│  │  Order         │     │  │  (mapping)  │  │  (protocol)      │  │ │
│  │  Product       │     │  └─────────────┘  └──────────────────┘  │ │
│  │                │     │  ┌─────────────┐  ┌──────────────────┐  │ │
│  │                │     │  │   Facade    │  │   Anti-         │  │ │
│  │                │     │  │             │  │   Corruption    │  │ │
│  │                │     │  └─────────────┘  │   Interface     │  │ │
│  └────────────────┘     │                   └──────────────────┘  │ │
│                         └──────────────────────┬──────────────────┘ │
└────────────────────────────────────────────────┼────────────────────┘
                                                 │
                                                 ▼
                                    ┌────────────────────────┐
                                    │    Legacy System /     │
                                    │    External API        │
                                    │                        │
                                    │  CUSTOMER_TBL          │
                                    │  ORDR_HDR, ORDR_DTL    │
                                    │  PROD_MASTER           │
                                    └────────────────────────┘
```

### Реализация

```python
# domain/models.py - Чистая доменная модель
from dataclasses import dataclass
from datetime import datetime
from decimal import Decimal
from typing import List, Optional
from enum import Enum

class OrderStatus(Enum):
    PENDING = "pending"
    CONFIRMED = "confirmed"
    SHIPPED = "shipped"
    DELIVERED = "delivered"
    CANCELLED = "cancelled"

@dataclass
class Address:
    street: str
    city: str
    state: str
    postal_code: str
    country: str

@dataclass
class Customer:
    id: str
    email: str
    first_name: str
    last_name: str
    phone: Optional[str]
    shipping_address: Address

@dataclass
class OrderItem:
    product_id: str
    product_name: str
    quantity: int
    unit_price: Decimal

@dataclass
class Order:
    id: str
    customer: Customer
    items: List[OrderItem]
    status: OrderStatus
    created_at: datetime
    total: Decimal

    def calculate_total(self) -> Decimal:
        return sum(item.unit_price * item.quantity for item in self.items)


# acl/legacy_client.py - Клиент для legacy системы
import requests
from typing import Dict, Any

class LegacyERPClient:
    """Клиент для работы с legacy ERP системой"""

    def __init__(self, base_url: str, api_key: str):
        self.base_url = base_url
        self.api_key = api_key

    def get_customer(self, customer_id: str) -> Dict[str, Any]:
        """Получает данные клиента в формате legacy системы"""
        response = requests.get(
            f"{self.base_url}/CUST_API/GET_CUST",
            params={"CUST_ID": customer_id},
            headers={"X-API-KEY": self.api_key}
        )
        return response.json()
        # Возвращает что-то вроде:
        # {
        #     "CUST_ID": "12345",
        #     "CUST_EMAIL": "john@example.com",
        #     "CUST_FNAME": "JOHN",
        #     "CUST_LNAME": "DOE",
        #     "CUST_PHONE": "555-1234",
        #     "SHIP_ADDR1": "123 Main St",
        #     "SHIP_CITY": "NEW YORK",
        #     "SHIP_STATE": "NY",
        #     "SHIP_ZIP": "10001",
        #     "SHIP_CNTRY": "US"
        # }

    def get_order(self, order_id: str) -> Dict[str, Any]:
        response = requests.get(
            f"{self.base_url}/ORD_API/GET_ORD",
            params={"ORD_NUM": order_id},
            headers={"X-API-KEY": self.api_key}
        )
        return response.json()

    def create_order(self, order_data: Dict[str, Any]) -> Dict[str, Any]:
        response = requests.post(
            f"{self.base_url}/ORD_API/CRT_ORD",
            json=order_data,
            headers={"X-API-KEY": self.api_key}
        )
        return response.json()


# acl/translators.py - Трансляторы между моделями
from domain.models import Customer, Address, Order, OrderItem, OrderStatus
from datetime import datetime
from decimal import Decimal

class CustomerTranslator:
    """Переводит данные клиента из legacy формата в доменную модель"""

    @staticmethod
    def to_domain(legacy_data: Dict[str, Any]) -> Customer:
        return Customer(
            id=legacy_data["CUST_ID"],
            email=legacy_data["CUST_EMAIL"].lower(),
            first_name=legacy_data["CUST_FNAME"].title(),
            last_name=legacy_data["CUST_LNAME"].title(),
            phone=legacy_data.get("CUST_PHONE"),
            shipping_address=Address(
                street=legacy_data["SHIP_ADDR1"].title(),
                city=legacy_data["SHIP_CITY"].title(),
                state=legacy_data["SHIP_STATE"],
                postal_code=legacy_data["SHIP_ZIP"],
                country=legacy_data["SHIP_CNTRY"]
            )
        )

    @staticmethod
    def to_legacy(customer: Customer) -> Dict[str, Any]:
        return {
            "CUST_ID": customer.id,
            "CUST_EMAIL": customer.email.upper(),
            "CUST_FNAME": customer.first_name.upper(),
            "CUST_LNAME": customer.last_name.upper(),
            "CUST_PHONE": customer.phone or "",
            "SHIP_ADDR1": customer.shipping_address.street.upper(),
            "SHIP_CITY": customer.shipping_address.city.upper(),
            "SHIP_STATE": customer.shipping_address.state.upper(),
            "SHIP_ZIP": customer.shipping_address.postal_code,
            "SHIP_CNTRY": customer.shipping_address.country.upper()
        }


class OrderTranslator:
    """Переводит данные заказа между форматами"""

    # Маппинг статусов
    STATUS_MAP = {
        "P": OrderStatus.PENDING,
        "C": OrderStatus.CONFIRMED,
        "S": OrderStatus.SHIPPED,
        "D": OrderStatus.DELIVERED,
        "X": OrderStatus.CANCELLED
    }

    REVERSE_STATUS_MAP = {v: k for k, v in STATUS_MAP.items()}

    @staticmethod
    def to_domain(legacy_data: Dict[str, Any], customer: Customer) -> Order:
        items = [
            OrderItem(
                product_id=item["PROD_CD"],
                product_name=item["PROD_DESC"].title(),
                quantity=int(item["QTY"]),
                unit_price=Decimal(str(item["UNIT_PRC"]))
            )
            for item in legacy_data.get("ORD_LINES", [])
        ]

        return Order(
            id=legacy_data["ORD_NUM"],
            customer=customer,
            items=items,
            status=OrderTranslator.STATUS_MAP.get(
                legacy_data["ORD_STAT"],
                OrderStatus.PENDING
            ),
            created_at=datetime.strptime(
                legacy_data["ORD_DT"],
                "%Y%m%d"
            ),
            total=Decimal(str(legacy_data["ORD_TOT"]))
        )

    @staticmethod
    def to_legacy(order: Order) -> Dict[str, Any]:
        return {
            "ORD_NUM": order.id,
            "CUST_ID": order.customer.id,
            "ORD_DT": order.created_at.strftime("%Y%m%d"),
            "ORD_STAT": OrderTranslator.REVERSE_STATUS_MAP[order.status],
            "ORD_TOT": float(order.total),
            "ORD_LINES": [
                {
                    "PROD_CD": item.product_id,
                    "PROD_DESC": item.product_name.upper(),
                    "QTY": item.quantity,
                    "UNIT_PRC": float(item.unit_price)
                }
                for item in order.items
            ]
        }


# acl/repositories.py - Репозитории с ACL
from domain.models import Customer, Order
from acl.legacy_client import LegacyERPClient
from acl.translators import CustomerTranslator, OrderTranslator
from typing import Optional

class CustomerRepository:
    """
    Репозиторий клиентов с антикоррупционным слоем.
    Внутри работает с legacy системой, наружу выдаёт чистые доменные объекты.
    """

    def __init__(self, legacy_client: LegacyERPClient):
        self._legacy = legacy_client
        self._translator = CustomerTranslator()

    def get_by_id(self, customer_id: str) -> Optional[Customer]:
        try:
            legacy_data = self._legacy.get_customer(customer_id)
            return self._translator.to_domain(legacy_data)
        except Exception as e:
            # Логируем и возвращаем None или кидаем доменное исключение
            logger.error(f"Failed to get customer: {e}")
            return None

    def save(self, customer: Customer) -> bool:
        try:
            legacy_data = self._translator.to_legacy(customer)
            self._legacy.update_customer(legacy_data)
            return True
        except Exception as e:
            logger.error(f"Failed to save customer: {e}")
            return False


class OrderRepository:
    """Репозиторий заказов с ACL"""

    def __init__(
        self,
        legacy_client: LegacyERPClient,
        customer_repository: CustomerRepository
    ):
        self._legacy = legacy_client
        self._customer_repo = customer_repository
        self._translator = OrderTranslator()

    def get_by_id(self, order_id: str) -> Optional[Order]:
        try:
            legacy_data = self._legacy.get_order(order_id)
            customer = self._customer_repo.get_by_id(legacy_data["CUST_ID"])
            if not customer:
                return None
            return self._translator.to_domain(legacy_data, customer)
        except Exception as e:
            logger.error(f"Failed to get order: {e}")
            return None

    def create(self, order: Order) -> Optional[str]:
        try:
            legacy_data = self._translator.to_legacy(order)
            result = self._legacy.create_order(legacy_data)
            return result.get("ORD_NUM")
        except Exception as e:
            logger.error(f"Failed to create order: {e}")
            return None


# services/order_service.py - Чистый сервис, работающий с доменными объектами
class OrderService:
    """
    Сервис заказов.
    Не знает ничего о legacy системе - работает только с доменными объектами.
    """

    def __init__(self, order_repository: OrderRepository):
        self._orders = order_repository

    def get_order(self, order_id: str) -> Optional[Order]:
        return self._orders.get_by_id(order_id)

    def place_order(self, customer: Customer, items: List[OrderItem]) -> Order:
        order = Order(
            id=generate_order_id(),
            customer=customer,
            items=items,
            status=OrderStatus.PENDING,
            created_at=datetime.now(),
            total=sum(item.unit_price * item.quantity for item in items)
        )

        order_id = self._orders.create(order)
        if not order_id:
            raise OrderCreationError("Failed to create order")

        order.id = order_id
        return order
```

---

## 3. Backends for Frontends (BFF)

### Описание

Backends for Frontends — паттерн, при котором для каждого типа клиента (web, mobile, IoT) создаётся отдельный backend-сервис. Каждый BFF оптимизирован под специфические нужды своего клиента.

### Когда использовать

- Разные клиенты имеют разные требования к данным
- Mobile нужны оптимизированные payload'ы
- Web и mobile команды работают независимо
- Разные требования к аутентификации для разных платформ

### Архитектура

```
                    ┌─────────────────┐
                    │   Mobile App    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Mobile BFF    │
                    │  - Optimized    │
                    │  - Offline      │
                    │  - Compressed   │
                    └────────┬────────┘
                             │
        ┌────────────────────┼────────────────────┐
        │                    │                    │
        ▼                    ▼                    ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ User Service  │  │ Order Service │  │Product Service│
└───────────────┘  └───────────────┘  └───────────────┘
        ▲                    ▲                    ▲
        │                    │                    │
        └────────────────────┼────────────────────┘
                             │
                    ┌────────┴────────┐
                    │    Web BFF      │
                    │  - Full data    │
                    │  - Rich UI      │
                    │  - Real-time    │
                    └────────┬────────┘
                             │
                    ┌────────▼────────┐
                    │   Web Client    │
                    └─────────────────┘
```

### Реализация

```python
# mobile_bff/app.py - BFF для мобильного приложения
from fastapi import FastAPI, Depends
from pydantic import BaseModel
from typing import List, Optional
import httpx
import asyncio

app = FastAPI(title="Mobile BFF")

# Клиенты для микросервисов
class ServiceClient:
    def __init__(self):
        self.user_service = "http://user-service:8080"
        self.order_service = "http://order-service:8080"
        self.product_service = "http://product-service:8080"
        self._client = httpx.AsyncClient(timeout=5.0)

    async def get_user(self, user_id: str) -> dict:
        response = await self._client.get(f"{self.user_service}/users/{user_id}")
        return response.json()

    async def get_user_orders(self, user_id: str, limit: int = 5) -> list:
        response = await self._client.get(
            f"{self.order_service}/orders",
            params={"user_id": user_id, "limit": limit}
        )
        return response.json()

    async def get_products(self, product_ids: List[str]) -> list:
        response = await self._client.post(
            f"{self.product_service}/products/batch",
            json={"ids": product_ids}
        )
        return response.json()

service_client = ServiceClient()

# Компактные модели для мобильного клиента
class MobileUserProfile(BaseModel):
    id: str
    name: str  # Объединённое имя
    avatar_url: Optional[str]
    # Без лишних полей (email, address и т.д.)

class MobileOrderSummary(BaseModel):
    id: str
    status: str
    total: float
    item_count: int
    created_at: str
    # Без деталей items - они загружаются отдельно

class MobileHomeScreen(BaseModel):
    """Оптимизированный ответ для главного экрана"""
    user: MobileUserProfile
    recent_orders: List[MobileOrderSummary]
    recommended_products: List[dict]

@app.get("/api/v1/home", response_model=MobileHomeScreen)
async def get_home_screen(user_id: str):
    """
    Один запрос для загрузки всего главного экрана.
    Оптимизировано для mobile - минимум данных, один round-trip.
    """
    # Параллельно запрашиваем все данные
    user_task = service_client.get_user(user_id)
    orders_task = service_client.get_user_orders(user_id, limit=3)
    # Рекомендации можно получить из кэша или отдельного сервиса

    user_data, orders_data = await asyncio.gather(user_task, orders_task)

    # Трансформируем в mobile-friendly формат
    return MobileHomeScreen(
        user=MobileUserProfile(
            id=user_data["id"],
            name=f"{user_data['first_name']} {user_data['last_name']}",
            avatar_url=user_data.get("avatar_url")
        ),
        recent_orders=[
            MobileOrderSummary(
                id=order["id"],
                status=order["status"],
                total=order["total"],
                item_count=len(order["items"]),
                created_at=order["created_at"]
            )
            for order in orders_data
        ],
        recommended_products=[]  # Упрощено для примера
    )


# Компрессия для mobile
from fastapi.responses import Response
import gzip

@app.middleware("http")
async def compress_response(request, call_next):
    response = await call_next(request)

    # Сжимаем большие ответы для mobile
    if (
        "gzip" in request.headers.get("accept-encoding", "")
        and response.headers.get("content-type", "").startswith("application/json")
    ):
        body = b""
        async for chunk in response.body_iterator:
            body += chunk

        if len(body) > 500:  # Сжимаем если больше 500 байт
            compressed = gzip.compress(body)
            return Response(
                content=compressed,
                status_code=response.status_code,
                headers={
                    **dict(response.headers),
                    "content-encoding": "gzip",
                    "content-length": str(len(compressed))
                },
                media_type=response.media_type
            )

    return response


# web_bff/app.py - BFF для веб-клиента
from fastapi import FastAPI, WebSocket
from typing import List

app = FastAPI(title="Web BFF")

# Более детальные модели для web
class WebUserProfile(BaseModel):
    id: str
    email: str
    first_name: str
    last_name: str
    phone: Optional[str]
    address: Optional[dict]
    preferences: dict
    created_at: str

class WebOrderDetail(BaseModel):
    id: str
    status: str
    items: List[dict]  # Полная информация о товарах
    shipping: dict
    billing: dict
    total: float
    subtotal: float
    tax: float
    discount: float
    tracking_number: Optional[str]
    estimated_delivery: Optional[str]
    created_at: str
    updated_at: str

@app.get("/api/v1/orders/{order_id}", response_model=WebOrderDetail)
async def get_order_detail(order_id: str):
    """
    Полная информация о заказе для web.
    Включает все детали, которые не нужны в mobile.
    """
    order_data = await service_client.get_order(order_id)

    # Обогащаем данными о продуктах
    product_ids = [item["product_id"] for item in order_data["items"]]
    products = await service_client.get_products(product_ids)
    products_map = {p["id"]: p for p in products}

    return WebOrderDetail(
        id=order_data["id"],
        status=order_data["status"],
        items=[
            {
                **item,
                "product": products_map.get(item["product_id"], {})
            }
            for item in order_data["items"]
        ],
        shipping=order_data.get("shipping", {}),
        billing=order_data.get("billing", {}),
        total=order_data["total"],
        subtotal=order_data.get("subtotal", order_data["total"]),
        tax=order_data.get("tax", 0),
        discount=order_data.get("discount", 0),
        tracking_number=order_data.get("tracking_number"),
        estimated_delivery=order_data.get("estimated_delivery"),
        created_at=order_data["created_at"],
        updated_at=order_data["updated_at"]
    )

# Real-time обновления для web через WebSocket
@app.websocket("/ws/orders/{order_id}")
async def order_updates(websocket: WebSocket, order_id: str):
    await websocket.accept()

    # Подписываемся на обновления заказа
    async for update in subscribe_to_order_updates(order_id):
        await websocket.send_json(update)
```

---

## 4. Bulkhead Pattern

### Описание

Bulkhead (переборка) — паттерн изоляции ресурсов, заимствованный из кораблестроения. Если один отсек корабля заполняется водой, переборки не дают воде проникнуть в другие отсеки. Аналогично в software: отказ одного компонента не должен влиять на другие.

### Когда использовать

- Изоляция критичных операций от некритичных
- Защита от каскадных отказов
- Обеспечение гарантированных ресурсов для важных клиентов
- Работа с ненадёжными внешними сервисами

### Типы Bulkhead

```
1. Thread Pool Isolation
┌─────────────────────────────────────────────────────────┐
│                    Application                           │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────┐ │
│  │ Thread Pool A  │  │ Thread Pool B  │  │ Pool C     │ │
│  │ (Payment API)  │  │ (Inventory)    │  │ (Logging)  │ │
│  │ max: 10        │  │ max: 20        │  │ max: 5     │ │
│  │ ████████       │  │ ██████████████ │  │ ███        │ │
│  └────────────────┘  └────────────────┘  └────────────┘ │
└─────────────────────────────────────────────────────────┘

2. Connection Pool Isolation
┌─────────────────────────────────────────────────────────┐
│                  Database Connections                    │
│  ┌────────────────┐  ┌────────────────┐  ┌────────────┐ │
│  │ Pool: Orders   │  │ Pool: Users    │  │ Pool: Logs │ │
│  │ max: 50        │  │ max: 30        │  │ max: 10    │ │
│  └────────────────┘  └────────────────┘  └────────────┘ │
└─────────────────────────────────────────────────────────┘
```

### Реализация

```python
# bulkhead.py
import asyncio
from concurrent.futures import ThreadPoolExecutor
from typing import Callable, Any
from dataclasses import dataclass
import time

@dataclass
class BulkheadConfig:
    max_concurrent: int
    max_wait_time: float = 5.0
    name: str = "default"

class Bulkhead:
    """Semaphore-based bulkhead для изоляции ресурсов"""

    def __init__(self, config: BulkheadConfig):
        self.config = config
        self._semaphore = asyncio.Semaphore(config.max_concurrent)
        self._active_count = 0
        self._rejected_count = 0

    async def execute(self, func: Callable, *args, **kwargs) -> Any:
        """Выполняет функцию с ограничением concurrency"""
        try:
            # Пытаемся получить слот с таймаутом
            acquired = await asyncio.wait_for(
                self._semaphore.acquire(),
                timeout=self.config.max_wait_time
            )
            if not acquired:
                raise BulkheadRejectedException(
                    f"Bulkhead {self.config.name} rejected - queue full"
                )

            self._active_count += 1
            try:
                return await func(*args, **kwargs)
            finally:
                self._active_count -= 1
                self._semaphore.release()

        except asyncio.TimeoutError:
            self._rejected_count += 1
            raise BulkheadRejectedException(
                f"Bulkhead {self.config.name} rejected - timeout waiting for slot"
            )

    @property
    def metrics(self) -> dict:
        return {
            "name": self.config.name,
            "active": self._active_count,
            "available": self.config.max_concurrent - self._active_count,
            "rejected": self._rejected_count
        }

class BulkheadRejectedException(Exception):
    pass


# Использование в приложении
class PaymentService:
    def __init__(self):
        # Отдельный bulkhead для критичных платёжных операций
        self.bulkhead = Bulkhead(BulkheadConfig(
            name="payment",
            max_concurrent=10,
            max_wait_time=3.0
        ))

    async def process_payment(self, payment_data: dict) -> dict:
        return await self.bulkhead.execute(
            self._do_process_payment,
            payment_data
        )

    async def _do_process_payment(self, payment_data: dict) -> dict:
        async with httpx.AsyncClient() as client:
            response = await client.post(
                "https://payment-gateway.example.com/charge",
                json=payment_data
            )
            return response.json()


class RecommendationService:
    def __init__(self):
        # Менее критичный сервис - можем отдать больше слотов
        self.bulkhead = Bulkhead(BulkheadConfig(
            name="recommendations",
            max_concurrent=50,
            max_wait_time=1.0  # Меньший timeout - не критично
        ))

    async def get_recommendations(self, user_id: str) -> list:
        try:
            return await self.bulkhead.execute(
                self._fetch_recommendations,
                user_id
            )
        except BulkheadRejectedException:
            # Fallback - возвращаем популярные товары
            return await self._get_fallback_recommendations()


# Thread Pool Bulkhead для CPU-bound задач
class ThreadPoolBulkhead:
    """Изоляция через отдельные thread pools"""

    _pools: dict = {}

    @classmethod
    def get_pool(cls, name: str, max_workers: int = 10) -> ThreadPoolExecutor:
        if name not in cls._pools:
            cls._pools[name] = ThreadPoolExecutor(
                max_workers=max_workers,
                thread_name_prefix=f"bulkhead-{name}"
            )
        return cls._pools[name]

    @classmethod
    async def run_in_pool(
        cls,
        pool_name: str,
        func: Callable,
        *args,
        max_workers: int = 10
    ) -> Any:
        pool = cls.get_pool(pool_name, max_workers)
        loop = asyncio.get_event_loop()
        return await loop.run_in_executor(pool, func, *args)

# Использование
async def process_image(image_data: bytes):
    """CPU-intensive операция в изолированном пуле"""
    return await ThreadPoolBulkhead.run_in_pool(
        "image-processing",
        heavy_image_processing,
        image_data,
        max_workers=4  # Ограничиваем ресурсы
    )

async def analyze_text(text: str):
    """Другая CPU-intensive операция в отдельном пуле"""
    return await ThreadPoolBulkhead.run_in_pool(
        "text-analysis",
        run_ml_model,
        text,
        max_workers=2
    )
```

### Kubernetes Resource Limits как Bulkhead

```yaml
# Изоляция на уровне Kubernetes
apiVersion: apps/v1
kind: Deployment
metadata:
  name: payment-service
spec:
  replicas: 3
  template:
    spec:
      containers:
        - name: payment
          resources:
            requests:
              memory: "256Mi"
              cpu: "500m"
            limits:
              memory: "512Mi"
              cpu: "1000m"
          # Гарантированные ресурсы для критичного сервиса

---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: recommendation-service
spec:
  replicas: 2
  template:
    spec:
      containers:
        - name: recommendations
          resources:
            requests:
              memory: "128Mi"
              cpu: "250m"
            limits:
              memory: "256Mi"
              cpu: "500m"
          # Меньше ресурсов для некритичного сервиса
```

---

## 5. Gateway Pattern (API Gateway)

### Описание

API Gateway — единая точка входа для всех клиентов, которая маршрутизирует запросы к соответствующим микросервисам. Gateway может выполнять: аутентификацию, rate limiting, кэширование, агрегацию, трансформацию протоколов.

### Архитектура

```
                         Clients
            ┌──────────────┼──────────────┐
            │              │              │
        Web App      Mobile App      Partner API
            │              │              │
            └──────────────┼──────────────┘
                           │
                    ┌──────▼──────┐
                    │ API Gateway │
                    ├─────────────┤
                    │ - Auth      │
                    │ - Rate Limit│
                    │ - Routing   │
                    │ - Caching   │
                    │ - Logging   │
                    │ - Transform │
                    └──────┬──────┘
                           │
        ┌──────────────────┼──────────────────┐
        │                  │                  │
        ▼                  ▼                  ▼
┌───────────────┐  ┌───────────────┐  ┌───────────────┐
│ User Service  │  │ Order Service │  │Product Service│
└───────────────┘  └───────────────┘  └───────────────┘
```

### Реализация

```python
# gateway/app.py
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.security import HTTPBearer
from typing import Optional
import httpx
import time
import hashlib
from collections import defaultdict

app = FastAPI(title="API Gateway")

# Конфигурация маршрутов
ROUTES = {
    "/api/users": {
        "service": "http://user-service:8080",
        "prefix": "/users",
        "auth_required": True
    },
    "/api/orders": {
        "service": "http://order-service:8080",
        "prefix": "/orders",
        "auth_required": True
    },
    "/api/products": {
        "service": "http://product-service:8080",
        "prefix": "/products",
        "auth_required": False  # Публичный API
    }
}

# Rate Limiter
class RateLimiter:
    def __init__(self, requests_per_minute: int = 60):
        self.rpm = requests_per_minute
        self.requests = defaultdict(list)

    def is_allowed(self, client_id: str) -> bool:
        now = time.time()
        minute_ago = now - 60

        # Удаляем старые запросы
        self.requests[client_id] = [
            ts for ts in self.requests[client_id]
            if ts > minute_ago
        ]

        if len(self.requests[client_id]) >= self.rpm:
            return False

        self.requests[client_id].append(now)
        return True

rate_limiter = RateLimiter(requests_per_minute=100)

# Кэш
class SimpleCache:
    def __init__(self, ttl: int = 60):
        self.cache = {}
        self.ttl = ttl

    def get(self, key: str) -> Optional[dict]:
        if key in self.cache:
            data, timestamp = self.cache[key]
            if time.time() - timestamp < self.ttl:
                return data
            del self.cache[key]
        return None

    def set(self, key: str, value: dict):
        self.cache[key] = (value, time.time())

cache = SimpleCache(ttl=30)

# JWT Validation
security = HTTPBearer(auto_error=False)

async def validate_token(request: Request) -> Optional[dict]:
    """Валидирует JWT токен"""
    auth = request.headers.get("Authorization")
    if not auth or not auth.startswith("Bearer "):
        return None

    token = auth.split(" ")[1]
    # В реальности - проверка JWT
    # Здесь упрощённо
    try:
        # payload = jwt.decode(token, SECRET_KEY, algorithms=["HS256"])
        return {"user_id": "123", "role": "user"}
    except:
        return None

def get_client_id(request: Request) -> str:
    """Получает идентификатор клиента для rate limiting"""
    # По токену или IP
    if "Authorization" in request.headers:
        return hashlib.md5(request.headers["Authorization"].encode()).hexdigest()
    return request.client.host

# Gateway Middleware
@app.middleware("http")
async def gateway_middleware(request: Request, call_next):
    start_time = time.time()

    # 1. Rate Limiting
    client_id = get_client_id(request)
    if not rate_limiter.is_allowed(client_id):
        return JSONResponse(
            status_code=429,
            content={"error": "Rate limit exceeded"},
            headers={"Retry-After": "60"}
        )

    # 2. Маршрутизация
    path = request.url.path
    route_config = None

    for prefix, config in ROUTES.items():
        if path.startswith(prefix):
            route_config = config
            break

    if not route_config:
        return await call_next(request)

    # 3. Аутентификация
    if route_config["auth_required"]:
        user = await validate_token(request)
        if not user:
            return JSONResponse(
                status_code=401,
                content={"error": "Unauthorized"}
            )
        request.state.user = user

    # 4. Проксирование
    response = await proxy_request(request, route_config)

    # 5. Метрики
    duration = time.time() - start_time
    log_request(request, response.status_code, duration)

    return response

async def proxy_request(request: Request, config: dict):
    """Проксирует запрос к целевому сервису"""
    # Формируем URL
    target_path = request.url.path.replace(
        list(ROUTES.keys())[0],
        config["prefix"],
        1
    )
    target_url = f"{config['service']}{target_path}"

    if request.url.query:
        target_url += f"?{request.url.query}"

    # Проверяем кэш для GET запросов
    if request.method == "GET":
        cache_key = f"{target_url}:{request.headers.get('Authorization', '')}"
        cached = cache.get(cache_key)
        if cached:
            return JSONResponse(
                content=cached["body"],
                status_code=cached["status"],
                headers={"X-Cache": "HIT"}
            )

    # Формируем headers
    headers = dict(request.headers)
    headers.pop("host", None)

    # Добавляем информацию о пользователе
    if hasattr(request.state, "user"):
        headers["X-User-Id"] = request.state.user["user_id"]
        headers["X-User-Role"] = request.state.user["role"]

    # Отправляем запрос
    body = await request.body()

    async with httpx.AsyncClient(timeout=30.0) as client:
        try:
            response = await client.request(
                method=request.method,
                url=target_url,
                headers=headers,
                content=body
            )

            response_body = response.json() if response.headers.get(
                "content-type", ""
            ).startswith("application/json") else response.text

            # Кэшируем успешные GET запросы
            if request.method == "GET" and response.status_code == 200:
                cache.set(cache_key, {
                    "body": response_body,
                    "status": response.status_code
                })

            return JSONResponse(
                content=response_body,
                status_code=response.status_code,
                headers={"X-Cache": "MISS"}
            )

        except httpx.RequestError as e:
            return JSONResponse(
                status_code=502,
                content={"error": "Service unavailable", "details": str(e)}
            )

# Request Aggregation
@app.get("/api/v1/dashboard")
async def get_dashboard(request: Request):
    """
    Агрегирует данные из нескольких сервисов в один ответ.
    Уменьшает количество round-trips для клиента.
    """
    user = request.state.user

    async with httpx.AsyncClient(timeout=10.0) as client:
        # Параллельные запросы к сервисам
        tasks = [
            client.get(f"http://user-service:8080/users/{user['user_id']}"),
            client.get(f"http://order-service:8080/orders?user_id={user['user_id']}&limit=5"),
            client.get("http://notification-service:8080/notifications/unread")
        ]

        responses = await asyncio.gather(*tasks, return_exceptions=True)

    # Собираем результат
    return {
        "user": responses[0].json() if not isinstance(responses[0], Exception) else None,
        "recent_orders": responses[1].json() if not isinstance(responses[1], Exception) else [],
        "notifications": responses[2].json() if not isinstance(responses[2], Exception) else []
    }
```

---

## 6. Sidecar Pattern

### Описание

Sidecar — это паттерн развёртывания, при котором вспомогательный контейнер запускается рядом с основным приложением в одном pod'е. Sidecar расширяет функциональность основного приложения без изменения его кода.

### Когда использовать

- Добавление observability (логирование, мониторинг, трейсинг)
- Управление сетевым трафиком (proxy, service mesh)
- Секреты и конфигурация
- Синхронизация данных
- Аутентификация/авторизация

### Архитектура

```
┌─────────────────────────────────────────────────────────────┐
│                           Pod                                │
│  ┌─────────────────────┐     ┌─────────────────────────┐    │
│  │   Main Container    │     │    Sidecar Container    │    │
│  │   (Application)     │────▶│    (Envoy Proxy)        │    │
│  │                     │     │                         │    │
│  │   localhost:8080    │     │   - mTLS               │    │
│  │                     │     │   - Load balancing      │    │
│  │                     │     │   - Circuit breaking    │    │
│  │                     │     │   - Metrics             │    │
│  └─────────────────────┘     └─────────────────────────┘    │
│                                        │                     │
│  Shared:                               │                     │
│  - Network namespace                   │                     │
│  - Volumes                             ▼                     │
│  - localhost communication     External Traffic              │
└─────────────────────────────────────────────────────────────┘
```

### Примеры Sidecar

```yaml
# Пример 1: Logging Sidecar
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-logging
spec:
  template:
    spec:
      containers:
        # Основное приложение
        - name: app
          image: my-app:latest
          ports:
            - containerPort: 8080
          volumeMounts:
            - name: logs
              mountPath: /var/log/app

        # Sidecar для отправки логов
        - name: log-shipper
          image: fluent/fluent-bit:latest
          volumeMounts:
            - name: logs
              mountPath: /var/log/app
              readOnly: true
            - name: fluent-bit-config
              mountPath: /fluent-bit/etc

      volumes:
        - name: logs
          emptyDir: {}
        - name: fluent-bit-config
          configMap:
            name: fluent-bit-config

---
# Пример 2: Vault Agent Sidecar для секретов
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-secrets
spec:
  template:
    metadata:
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/role: "my-app"
        vault.hashicorp.com/agent-inject-secret-config: "secret/data/my-app/config"
    spec:
      serviceAccountName: my-app
      containers:
        - name: app
          image: my-app:latest
          env:
            - name: DB_PASSWORD_FILE
              value: /vault/secrets/config

---
# Пример 3: Envoy Sidecar для mTLS
apiVersion: apps/v1
kind: Deployment
metadata:
  name: app-with-envoy
spec:
  template:
    spec:
      containers:
        - name: app
          image: my-app:latest
          ports:
            - containerPort: 8080
          env:
            # Приложение общается только с localhost
            - name: UPSTREAM_URL
              value: "http://localhost:9001"

        - name: envoy
          image: envoyproxy/envoy:v1.25.0
          ports:
            - containerPort: 9001  # Outbound proxy
            - containerPort: 9002  # Inbound proxy
          volumeMounts:
            - name: envoy-config
              mountPath: /etc/envoy
            - name: certs
              mountPath: /etc/certs
              readOnly: true
```

### Python Sidecar для мониторинга

```python
# sidecar/metrics_collector.py
"""
Sidecar для сбора метрик из основного приложения
и экспорта в Prometheus
"""
import asyncio
import aiohttp
from prometheus_client import start_http_server, Gauge, Counter
import json
import time

# Метрики
app_health = Gauge('app_health_status', 'Application health status')
app_requests = Counter('app_requests_total', 'Total requests from app metrics')
app_response_time = Gauge('app_response_time_ms', 'Average response time')

async def collect_metrics():
    """Собирает метрики из основного приложения"""
    app_url = "http://localhost:8080/metrics"  # Приложение на localhost

    async with aiohttp.ClientSession() as session:
        while True:
            try:
                async with session.get(app_url, timeout=5) as response:
                    if response.status == 200:
                        data = await response.json()

                        # Обновляем Prometheus метрики
                        app_health.set(1 if data.get("healthy") else 0)
                        app_requests.inc(data.get("requests_since_last", 0))
                        app_response_time.set(data.get("avg_response_time_ms", 0))
                    else:
                        app_health.set(0)

            except Exception as e:
                print(f"Error collecting metrics: {e}")
                app_health.set(0)

            await asyncio.sleep(15)  # Собираем каждые 15 секунд

async def health_check():
    """Проверяет здоровье основного приложения"""
    app_url = "http://localhost:8080/health"

    async with aiohttp.ClientSession() as session:
        while True:
            try:
                async with session.get(app_url, timeout=3) as response:
                    if response.status != 200:
                        print(f"App unhealthy: status {response.status}")
                        # Можно отправить алерт или перезапустить pod
            except Exception as e:
                print(f"Health check failed: {e}")

            await asyncio.sleep(5)

def main():
    # Запускаем Prometheus endpoint
    start_http_server(9090)
    print("Metrics sidecar started on :9090")

    # Запускаем сбор метрик и health checks
    loop = asyncio.get_event_loop()
    loop.create_task(collect_metrics())
    loop.create_task(health_check())
    loop.run_forever()

if __name__ == "__main__":
    main()
```

---

## 7. Strangler Fig Pattern

### Описание

Strangler Fig (душитель) — паттерн постепенной миграции с legacy системы на новую. Название происходит от тропического растения, которое обвивает дерево и постепенно заменяет его. Новая система постепенно "душит" старую, перехватывая всё больше функциональности.

### Когда использовать

- Миграция с монолита на микросервисы
- Замена legacy системы без Big Bang подхода
- Снижение рисков при модернизации
- Поддержка обратной совместимости во время миграции

### Этапы миграции

```
Этап 1: Facade перед legacy
┌──────────────┐
│   Clients    │
└──────┬───────┘
       │
┌──────▼───────┐
│   Facade     │──────────────┐
│  (Router)    │              │
└──────┬───────┘              │
       │ 100%                 │ 0%
┌──────▼───────┐      ┌───────▼───────┐
│   Legacy     │      │  New Service  │
│   System     │      │   (empty)     │
└──────────────┘      └───────────────┘

Этап 2: Частичная миграция
┌──────────────┐
│   Clients    │
└──────┬───────┘
       │
┌──────▼───────┐
│   Facade     │──────────────┐
│  (Router)    │              │
└──────┬───────┘              │
       │ 60%                  │ 40%
┌──────▼───────┐      ┌───────▼───────┐
│   Legacy     │      │  New Service  │
│   System     │      │  (Users API)  │
└──────────────┘      └───────────────┘

Этап 3: Полная миграция
┌──────────────┐
│   Clients    │
└──────┬───────┘
       │
┌──────▼───────┐
│   Facade     │──────────────┐
│  (optional)  │              │
└──────────────┘              │
                              │ 100%
                      ┌───────▼───────┐
                      │  New Service  │
                      │  (Complete)   │
                      └───────────────┘
```

### Реализация

```python
# strangler/facade.py
from fastapi import FastAPI, Request, Response
from typing import Dict, Optional
import httpx
import asyncio
from dataclasses import dataclass
from enum import Enum

app = FastAPI(title="Strangler Facade")

class RoutingStrategy(Enum):
    LEGACY = "legacy"
    NEW = "new"
    SHADOW = "shadow"  # Отправляем в оба, возвращаем из legacy
    CANARY = "canary"  # Процент трафика в new

@dataclass
class RouteConfig:
    pattern: str
    strategy: RoutingStrategy
    legacy_url: str
    new_url: Optional[str] = None
    canary_percent: int = 0  # Для CANARY стратегии

# Конфигурация маршрутов (может быть в БД или config service)
ROUTE_CONFIG: Dict[str, RouteConfig] = {
    # Полностью мигрировано
    "/api/users": RouteConfig(
        pattern="/api/users",
        strategy=RoutingStrategy.NEW,
        legacy_url="http://legacy-system:8080",
        new_url="http://user-service:8080"
    ),
    # В процессе миграции - canary
    "/api/orders": RouteConfig(
        pattern="/api/orders",
        strategy=RoutingStrategy.CANARY,
        legacy_url="http://legacy-system:8080",
        new_url="http://order-service:8080",
        canary_percent=20  # 20% трафика в новый сервис
    ),
    # Shadow mode - тестируем новый сервис
    "/api/products": RouteConfig(
        pattern="/api/products",
        strategy=RoutingStrategy.SHADOW,
        legacy_url="http://legacy-system:8080",
        new_url="http://product-service:8080"
    ),
    # Ещё не мигрировано
    "/api/reports": RouteConfig(
        pattern="/api/reports",
        strategy=RoutingStrategy.LEGACY,
        legacy_url="http://legacy-system:8080"
    )
}

import random
import hashlib

def get_route_config(path: str) -> RouteConfig:
    """Находит конфигурацию маршрута по пути"""
    for pattern, config in ROUTE_CONFIG.items():
        if path.startswith(pattern):
            return config
    # По умолчанию - legacy
    return RouteConfig(
        pattern="/*",
        strategy=RoutingStrategy.LEGACY,
        legacy_url="http://legacy-system:8080"
    )

def should_route_to_new(config: RouteConfig, request: Request) -> bool:
    """Определяет, направить ли запрос в новый сервис"""
    if config.strategy == RoutingStrategy.NEW:
        return True
    if config.strategy == RoutingStrategy.LEGACY:
        return False
    if config.strategy == RoutingStrategy.CANARY:
        # Sticky routing по user_id для консистентности
        user_id = request.headers.get("X-User-Id", request.client.host)
        hash_value = int(hashlib.md5(user_id.encode()).hexdigest(), 16)
        return (hash_value % 100) < config.canary_percent
    return False

async def proxy_request(url: str, request: Request) -> httpx.Response:
    """Проксирует запрос"""
    async with httpx.AsyncClient(timeout=30.0) as client:
        body = await request.body()
        headers = dict(request.headers)
        headers.pop("host", None)

        return await client.request(
            method=request.method,
            url=url + request.url.path,
            headers=headers,
            content=body,
            params=dict(request.query_params)
        )

@app.api_route("/{path:path}", methods=["GET", "POST", "PUT", "DELETE", "PATCH"])
async def route_request(path: str, request: Request):
    """Главный роутер Strangler Facade"""
    config = get_route_config(f"/{path}")

    if config.strategy == RoutingStrategy.LEGACY:
        # Только legacy
        response = await proxy_request(config.legacy_url, request)
        return Response(
            content=response.content,
            status_code=response.status_code,
            headers=dict(response.headers)
        )

    elif config.strategy == RoutingStrategy.NEW:
        # Только новый сервис
        response = await proxy_request(config.new_url, request)
        return Response(
            content=response.content,
            status_code=response.status_code,
            headers=dict(response.headers)
        )

    elif config.strategy == RoutingStrategy.SHADOW:
        # Shadow: отправляем в оба, возвращаем legacy
        legacy_task = proxy_request(config.legacy_url, request)
        new_task = proxy_request(config.new_url, request)

        legacy_response, new_response = await asyncio.gather(
            legacy_task,
            new_task,
            return_exceptions=True
        )

        # Сравниваем ответы (для аналитики)
        if not isinstance(new_response, Exception):
            compare_responses(legacy_response, new_response, request.url.path)

        # Возвращаем legacy ответ
        if isinstance(legacy_response, Exception):
            raise legacy_response

        return Response(
            content=legacy_response.content,
            status_code=legacy_response.status_code,
            headers={
                **dict(legacy_response.headers),
                "X-Shadow-Mode": "true"
            }
        )

    elif config.strategy == RoutingStrategy.CANARY:
        # Canary: процент трафика в новый сервис
        use_new = should_route_to_new(config, request)
        target_url = config.new_url if use_new else config.legacy_url

        response = await proxy_request(target_url, request)

        return Response(
            content=response.content,
            status_code=response.status_code,
            headers={
                **dict(response.headers),
                "X-Routed-To": "new" if use_new else "legacy"
            }
        )

def compare_responses(legacy: httpx.Response, new: httpx.Response, path: str):
    """Сравнивает ответы legacy и нового сервиса"""
    if legacy.status_code != new.status_code:
        log_difference("status_code", path, legacy.status_code, new.status_code)

    try:
        legacy_json = legacy.json()
        new_json = new.json()

        if legacy_json != new_json:
            log_difference("body", path, legacy_json, new_json)
    except:
        if legacy.content != new.content:
            log_difference("content", path, legacy.content[:100], new.content[:100])

def log_difference(diff_type: str, path: str, legacy_value, new_value):
    """Логирует различия для анализа"""
    print(f"DIFF [{diff_type}] {path}: legacy={legacy_value}, new={new_value}")
    # В production - отправка в систему мониторинга


# Административный API для управления миграцией
@app.put("/admin/routes/{pattern:path}")
async def update_route(pattern: str, config: RouteConfig):
    """Обновляет конфигурацию маршрута"""
    ROUTE_CONFIG[f"/{pattern}"] = config
    return {"status": "updated", "pattern": pattern}

@app.get("/admin/routes")
async def get_routes():
    """Возвращает текущую конфигурацию маршрутов"""
    return {
        pattern: {
            "strategy": config.strategy.value,
            "legacy_url": config.legacy_url,
            "new_url": config.new_url,
            "canary_percent": config.canary_percent
        }
        for pattern, config in ROUTE_CONFIG.items()
    }
```

### Feature Flags для Strangler

```python
# strangler/feature_flags.py
from typing import Dict, Any
import httpx

class FeatureFlagClient:
    """Клиент для сервиса feature flags"""

    def __init__(self, base_url: str):
        self.base_url = base_url
        self._cache = {}

    async def is_enabled(self, flag: str, context: Dict[str, Any] = None) -> bool:
        """Проверяет, включён ли feature flag"""
        cache_key = f"{flag}:{hash(str(context))}"

        if cache_key in self._cache:
            return self._cache[cache_key]

        async with httpx.AsyncClient() as client:
            response = await client.post(
                f"{self.base_url}/evaluate",
                json={"flag": flag, "context": context or {}}
            )
            result = response.json().get("enabled", False)
            self._cache[cache_key] = result
            return result

# Использование в Strangler Facade
feature_flags = FeatureFlagClient("http://feature-flag-service:8080")

async def should_use_new_service(path: str, user_id: str) -> bool:
    """Определяет маршрут на основе feature flags"""

    # Проверяем глобальный флаг миграции
    if not await feature_flags.is_enabled("migration_enabled"):
        return False

    # Проверяем флаг для конкретного эндпоинта
    endpoint_flag = f"migrate_{path.replace('/', '_')}"
    if not await feature_flags.is_enabled(endpoint_flag):
        return False

    # Проверяем, включён ли пользователь в canary
    return await feature_flags.is_enabled(
        "new_service_canary",
        context={"user_id": user_id}
    )
```

---

## Best Practices

### Выбор паттерна

| Проблема | Паттерн |
|----------|---------|
| Нужна изоляция сетевой логики | Ambassador |
| Интеграция с legacy API | Anti-Corruption Layer |
| Разные клиенты = разные потребности | Backends for Frontends |
| Защита от каскадных отказов | Bulkhead |
| Единая точка входа для микросервисов | Gateway |
| Добавление функциональности без изменения кода | Sidecar |
| Постепенная миграция с legacy | Strangler Fig |

### Общие рекомендации

1. **Начинайте просто** — не используйте сложные паттерны, если можно обойтись без них
2. **Мониторинг обязателен** — все паттерны требуют хорошего observability
3. **Тестируйте отказы** — chaos engineering для проверки resilience
4. **Документируйте решения** — ADR (Architecture Decision Records) для объяснения выбора
5. **Итеративный подход** — начните с MVP паттерна и развивайте

---

## Заключение

Облачные паттерны проектирования — это инструменты для решения типичных проблем распределённых систем:

- **Ambassador** и **Sidecar** — расширение функциональности без изменения кода
- **Anti-Corruption Layer** — защита доменной модели от внешних влияний
- **BFF** — оптимизация API под конкретных клиентов
- **Bulkhead** — изоляция и предотвращение каскадных отказов
- **Gateway** — централизованное управление трафиком
- **Strangler Fig** — безопасная миграция с legacy систем

Правильное применение этих паттернов помогает создавать устойчивые, масштабируемые и поддерживаемые системы.
