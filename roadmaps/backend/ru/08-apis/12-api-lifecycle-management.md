# API Lifecycle Management

[prev: 05-pii](./11-standards-and-compliance/05-pii.md) | [next: 01-cdn](../09-caching/01-cdn.md)

---

## Что такое жизненный цикл API

**API Lifecycle Management (ALM)** — это комплексный подход к управлению API на всех этапах его существования: от первоначальной идеи до полного вывода из эксплуатации. Это включает процессы, инструменты и практики, которые обеспечивают качество, безопасность и согласованность API на протяжении всего его жизненного цикла.

### Почему это важно

- **Согласованность**: Единые стандарты для всех API в организации
- **Качество**: Систематический контроль на каждом этапе
- **Безопасность**: Встроенные практики безопасности
- **Масштабируемость**: Возможность управлять сотнями API
- **Снижение затрат**: Предотвращение проблем на ранних этапах

---

## Этапы жизненного цикла API

### 1. Планирование (Planning)

На этом этапе определяются бизнес-цели и технические требования к API.

**Ключевые активности:**
- Анализ бизнес-потребностей
- Определение целевой аудитории (внутренние разработчики, партнеры, публичные пользователи)
- Оценка существующих API (избежание дублирования)
- Определение KPI и метрик успеха
- Планирование ресурсов и сроков

```markdown
## API Planning Checklist

### Бизнес-требования
- [ ] Какую проблему решает API?
- [ ] Кто будет использовать API?
- [ ] Какие данные будут предоставляться?
- [ ] Каковы требования к SLA?

### Технические требования
- [ ] Ожидаемая нагрузка (RPS)
- [ ] Требования к задержке (latency)
- [ ] Требования к безопасности
- [ ] Интеграция с существующими системами
```

### 2. Проектирование (Design)

**Design-first подход** — создание спецификации API до написания кода.

**Ключевые аспекты:**
- Выбор архитектурного стиля (REST, GraphQL, gRPC)
- Проектирование эндпоинтов и ресурсов
- Определение форматов данных
- Планирование версионирования
- Создание OpenAPI/Swagger спецификации

```yaml
# openapi.yaml - Design-first пример
openapi: 3.0.3
info:
  title: Order Management API
  description: API для управления заказами
  version: 1.0.0
  contact:
    name: API Team
    email: api-team@company.com

servers:
  - url: https://api.company.com/v1
    description: Production server
  - url: https://api-staging.company.com/v1
    description: Staging server

paths:
  /orders:
    get:
      summary: Получить список заказов
      operationId: listOrders
      tags:
        - Orders
      parameters:
        - name: status
          in: query
          schema:
            type: string
            enum: [pending, processing, completed, cancelled]
        - name: page
          in: query
          schema:
            type: integer
            default: 1
        - name: limit
          in: query
          schema:
            type: integer
            default: 20
            maximum: 100
      responses:
        '200':
          description: Успешный ответ
          content:
            application/json:
              schema:
                $ref: '#/components/schemas/OrderList'
        '401':
          $ref: '#/components/responses/Unauthorized'

components:
  schemas:
    Order:
      type: object
      required:
        - id
        - customerId
        - status
        - createdAt
      properties:
        id:
          type: string
          format: uuid
        customerId:
          type: string
          format: uuid
        status:
          type: string
          enum: [pending, processing, completed, cancelled]
        items:
          type: array
          items:
            $ref: '#/components/schemas/OrderItem'
        totalAmount:
          type: number
          format: decimal
        createdAt:
          type: string
          format: date-time
```

### 3. Разработка (Development)

Реализация API согласно спецификации.

**Практики разработки:**

```python
# Python/FastAPI - реализация на основе OpenAPI спецификации
from fastapi import FastAPI, Query, HTTPException, Depends
from pydantic import BaseModel, Field
from typing import List, Optional
from enum import Enum
from uuid import UUID
from datetime import datetime

class OrderStatus(str, Enum):
    PENDING = "pending"
    PROCESSING = "processing"
    COMPLETED = "completed"
    CANCELLED = "cancelled"

class OrderItem(BaseModel):
    product_id: UUID
    quantity: int = Field(gt=0)
    price: float = Field(gt=0)

class Order(BaseModel):
    id: UUID
    customer_id: UUID
    status: OrderStatus
    items: List[OrderItem]
    total_amount: float
    created_at: datetime

    class Config:
        json_schema_extra = {
            "example": {
                "id": "550e8400-e29b-41d4-a716-446655440000",
                "customer_id": "550e8400-e29b-41d4-a716-446655440001",
                "status": "pending",
                "items": [{"product_id": "550e8400-e29b-41d4-a716-446655440002", "quantity": 2, "price": 29.99}],
                "total_amount": 59.98,
                "created_at": "2024-01-15T10:30:00Z"
            }
        }

class OrderList(BaseModel):
    items: List[Order]
    total: int
    page: int
    limit: int

app = FastAPI(
    title="Order Management API",
    version="1.0.0",
    description="API для управления заказами"
)

@app.get("/orders", response_model=OrderList, tags=["Orders"])
async def list_orders(
    status: Optional[OrderStatus] = None,
    page: int = Query(1, ge=1),
    limit: int = Query(20, ge=1, le=100)
):
    """
    Получить список заказов с пагинацией и фильтрацией.

    - **status**: Фильтр по статусу заказа
    - **page**: Номер страницы
    - **limit**: Количество элементов на странице
    """
    # Реализация логики
    orders = await order_service.get_orders(status, page, limit)
    return OrderList(
        items=orders,
        total=len(orders),
        page=page,
        limit=limit
    )
```

### 4. Тестирование (Testing)

Комплексное тестирование API на всех уровнях.

```python
# pytest тесты для API
import pytest
from httpx import AsyncClient
from fastapi import status

@pytest.mark.asyncio
class TestOrdersAPI:

    async def test_list_orders_success(self, client: AsyncClient):
        """Тест успешного получения списка заказов"""
        response = await client.get("/orders")

        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert "items" in data
        assert "total" in data
        assert "page" in data
        assert "limit" in data

    async def test_list_orders_with_status_filter(self, client: AsyncClient):
        """Тест фильтрации по статусу"""
        response = await client.get("/orders?status=pending")

        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        for order in data["items"]:
            assert order["status"] == "pending"

    async def test_list_orders_pagination(self, client: AsyncClient):
        """Тест пагинации"""
        response = await client.get("/orders?page=2&limit=10")

        assert response.status_code == status.HTTP_200_OK
        data = response.json()
        assert data["page"] == 2
        assert data["limit"] == 10

    async def test_list_orders_invalid_limit(self, client: AsyncClient):
        """Тест валидации limit > 100"""
        response = await client.get("/orders?limit=150")

        assert response.status_code == status.HTTP_422_UNPROCESSABLE_ENTITY

    async def test_list_orders_unauthorized(self, client: AsyncClient):
        """Тест без авторизации"""
        response = await client.get("/orders", headers={})

        assert response.status_code == status.HTTP_401_UNAUTHORIZED

# Contract Testing с Pact
from pact import Consumer, Provider

pact = Consumer('OrderService').has_pact_with(Provider('InventoryService'))

def test_get_product_availability():
    """Contract test для проверки доступности товара"""
    expected = {
        "product_id": "123",
        "available": True,
        "quantity": 50
    }

    (pact
     .given('product 123 exists and is in stock')
     .upon_receiving('a request for product availability')
     .with_request('GET', '/products/123/availability')
     .will_respond_with(200, body=expected))

    with pact:
        result = inventory_client.get_availability("123")
        assert result["available"] == True
```

### 5. Развертывание (Deployment)

**Стратегии развертывания:**

```yaml
# Kubernetes deployment с canary release
apiVersion: argoproj.io/v1alpha1
kind: Rollout
metadata:
  name: orders-api
spec:
  replicas: 10
  strategy:
    canary:
      steps:
        - setWeight: 10
        - pause: {duration: 5m}
        - setWeight: 25
        - pause: {duration: 10m}
        - setWeight: 50
        - pause: {duration: 10m}
        - setWeight: 100
      canaryMetadata:
        labels:
          version: canary
      stableMetadata:
        labels:
          version: stable
  selector:
    matchLabels:
      app: orders-api
  template:
    metadata:
      labels:
        app: orders-api
    spec:
      containers:
        - name: orders-api
          image: orders-api:v1.2.0
          ports:
            - containerPort: 8080
          env:
            - name: API_VERSION
              value: "1.2.0"
          readinessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 5
            periodSeconds: 10
          livenessProbe:
            httpGet:
              path: /health
              port: 8080
            initialDelaySeconds: 15
            periodSeconds: 20
```

### 6. Поддержка (Maintenance)

**Мониторинг и обслуживание в production:**

```python
# Мониторинг API с Prometheus метриками
from prometheus_client import Counter, Histogram, generate_latest
from functools import wraps
import time

# Метрики
REQUEST_COUNT = Counter(
    'api_requests_total',
    'Total API requests',
    ['method', 'endpoint', 'status']
)

REQUEST_LATENCY = Histogram(
    'api_request_latency_seconds',
    'API request latency',
    ['method', 'endpoint'],
    buckets=[.01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

ERROR_COUNT = Counter(
    'api_errors_total',
    'Total API errors',
    ['method', 'endpoint', 'error_type']
)

def track_metrics(func):
    """Декоратор для автоматического сбора метрик"""
    @wraps(func)
    async def wrapper(*args, **kwargs):
        method = kwargs.get('request').method
        endpoint = kwargs.get('request').url.path

        start_time = time.time()
        try:
            response = await func(*args, **kwargs)
            REQUEST_COUNT.labels(
                method=method,
                endpoint=endpoint,
                status=response.status_code
            ).inc()
            return response
        except Exception as e:
            ERROR_COUNT.labels(
                method=method,
                endpoint=endpoint,
                error_type=type(e).__name__
            ).inc()
            raise
        finally:
            REQUEST_LATENCY.labels(
                method=method,
                endpoint=endpoint
            ).observe(time.time() - start_time)

    return wrapper

# Health check endpoint
@app.get("/health")
async def health_check():
    """Проверка здоровья сервиса"""
    checks = {
        "database": await check_database(),
        "cache": await check_cache(),
        "dependencies": await check_dependencies()
    }

    all_healthy = all(checks.values())

    return {
        "status": "healthy" if all_healthy else "degraded",
        "checks": checks,
        "version": "1.0.0",
        "timestamp": datetime.utcnow().isoformat()
    }
```

### 7. Вывод из эксплуатации (Deprecation & Retirement)

Процесс постепенного прекращения поддержки API.

```python
# Реализация deprecation headers
from fastapi import FastAPI, Request, Response
from datetime import datetime, timedelta

DEPRECATION_DATE = datetime(2024, 6, 1)
SUNSET_DATE = datetime(2024, 12, 1)

@app.middleware("http")
async def add_deprecation_headers(request: Request, call_next):
    response = await call_next(request)

    # Добавляем заголовки для deprecated endpoints
    if request.url.path.startswith("/v1/"):
        response.headers["Deprecation"] = DEPRECATION_DATE.strftime("%a, %d %b %Y %H:%M:%S GMT")
        response.headers["Sunset"] = SUNSET_DATE.strftime("%a, %d %b %Y %H:%M:%S GMT")
        response.headers["Link"] = '</v2/>; rel="successor-version"'

        # Warning header
        days_until_sunset = (SUNSET_DATE - datetime.now()).days
        response.headers["Warning"] = f'299 - "API v1 is deprecated. {days_until_sunset} days until sunset. Please migrate to v2."'

    return response

# Логирование использования deprecated API
@app.get("/v1/orders", deprecated=True)
async def list_orders_v1(request: Request):
    """
    [DEPRECATED] Получить список заказов.

    Этот endpoint устарел и будет удален 2024-12-01.
    Используйте /v2/orders вместо него.
    """
    # Логируем использование deprecated endpoint
    logger.warning(
        "Deprecated API used",
        extra={
            "endpoint": "/v1/orders",
            "client_id": request.headers.get("X-Client-ID"),
            "user_agent": request.headers.get("User-Agent")
        }
    )

    return await list_orders_v2()
```

---

## Версионирование API

### Семантическое версионирование (SemVer)

```
MAJOR.MINOR.PATCH

- MAJOR: Несовместимые изменения API (breaking changes)
- MINOR: Новая функциональность с обратной совместимостью
- PATCH: Исправления ошибок
```

**Примеры изменений:**

```python
# MAJOR (2.0.0) - Breaking change: изменение структуры ответа
# До
{"id": "123", "name": "Product", "price": 99.99}

# После
{"data": {"id": "123", "attributes": {"name": "Product", "price": 99.99}}}

# MINOR (1.1.0) - Новое поле (обратно совместимо)
{"id": "123", "name": "Product", "price": 99.99, "description": "New field"}

# PATCH (1.0.1) - Исправление ошибки в вычислении price
```

### URL-based версионирование

```python
# Версия в URL path
from fastapi import APIRouter

v1_router = APIRouter(prefix="/v1")
v2_router = APIRouter(prefix="/v2")

@v1_router.get("/orders")
async def list_orders_v1():
    """V1: Возвращает простой список"""
    return {"orders": [...]}

@v2_router.get("/orders")
async def list_orders_v2():
    """V2: Возвращает с пагинацией и метаданными"""
    return {
        "data": [...],
        "meta": {"total": 100, "page": 1},
        "links": {"next": "/v2/orders?page=2"}
    }

app.include_router(v1_router)
app.include_router(v2_router)
```

**Преимущества:**
- Очевидно и просто для понимания
- Легко кешировать
- Простая маршрутизация

**Недостатки:**
- Нарушает принцип REST (URI должен идентифицировать ресурс)
- Требует изменения URL при обновлении версии

### Header-based версионирование

```python
from fastapi import Header, HTTPException

@app.get("/orders")
async def list_orders(
    api_version: str = Header(default="1.0", alias="X-API-Version")
):
    """Выбор версии через заголовок"""
    if api_version == "1.0":
        return await list_orders_v1()
    elif api_version == "2.0":
        return await list_orders_v2()
    else:
        raise HTTPException(
            status_code=400,
            detail=f"Unsupported API version: {api_version}"
        )

# Альтернатива: Accept header с media type
@app.get("/orders")
async def list_orders(
    accept: str = Header(default="application/vnd.company.v1+json")
):
    """Версионирование через Accept header"""
    if "v1" in accept:
        return await list_orders_v1()
    elif "v2" in accept:
        return await list_orders_v2()
```

### Query parameter версионирование

```python
@app.get("/orders")
async def list_orders(version: str = Query(default="1.0")):
    """Версия через query parameter: /orders?version=2.0"""
    if version == "1.0":
        return await list_orders_v1()
    elif version == "2.0":
        return await list_orders_v2()
```

### Сравнение подходов

| Подход | Pros | Cons | Когда использовать |
|--------|------|------|-------------------|
| URL Path | Простота, явность, кеширование | Нарушает REST, дублирование | Публичные API |
| Header | Чистые URL, гибкость | Сложнее тестировать | Enterprise API |
| Query Param | Простота тестирования | Загрязняет URL | Внутренние API |
| Media Type | Правильный REST | Сложность | REST purists |

---

## Deprecation политики

### Структура политики

```markdown
# API Deprecation Policy

## Сроки
- Минимальное время между объявлением deprecation и sunset: 6 месяцев
- Уведомление клиентов: минимум за 3 месяца до sunset
- Период миграции: поддержка обеих версий минимум 3 месяца

## Коммуникация
1. Email уведомление всем зарегистрированным разработчикам
2. Deprecation warning в API ответах
3. Документация с гайдом миграции
4. Changelog с детальным описанием изменений

## Исключения
- Security vulnerabilities: немедленное отключение с уведомлением
- Critical bugs: ускоренный процесс (минимум 30 дней)
```

### Реализация deprecation workflow

```python
from dataclasses import dataclass
from datetime import datetime, date
from typing import Optional
from enum import Enum

class DeprecationStatus(Enum):
    ACTIVE = "active"
    DEPRECATED = "deprecated"
    SUNSET = "sunset"
    RETIRED = "retired"

@dataclass
class APIVersion:
    version: str
    status: DeprecationStatus
    deprecation_date: Optional[date] = None
    sunset_date: Optional[date] = None
    successor_version: Optional[str] = None
    migration_guide_url: Optional[str] = None

class VersionManager:
    def __init__(self):
        self.versions = {
            "1.0": APIVersion(
                version="1.0",
                status=DeprecationStatus.DEPRECATED,
                deprecation_date=date(2024, 1, 1),
                sunset_date=date(2024, 7, 1),
                successor_version="2.0",
                migration_guide_url="https://docs.company.com/migration/v1-to-v2"
            ),
            "2.0": APIVersion(
                version="2.0",
                status=DeprecationStatus.ACTIVE
            )
        }

    def get_version_info(self, version: str) -> APIVersion:
        return self.versions.get(version)

    def is_deprecated(self, version: str) -> bool:
        v = self.versions.get(version)
        return v and v.status == DeprecationStatus.DEPRECATED

    def is_sunset(self, version: str) -> bool:
        v = self.versions.get(version)
        if v and v.sunset_date:
            return date.today() >= v.sunset_date
        return False

    def get_deprecation_message(self, version: str) -> str:
        v = self.versions.get(version)
        if not v or v.status == DeprecationStatus.ACTIVE:
            return ""

        days_left = (v.sunset_date - date.today()).days if v.sunset_date else 0

        return (
            f"API version {version} is deprecated. "
            f"Sunset date: {v.sunset_date}. "
            f"Days remaining: {days_left}. "
            f"Please migrate to version {v.successor_version}. "
            f"Migration guide: {v.migration_guide_url}"
        )
```

---

## API Governance

### Что такое API Governance

API Governance — это набор политик, стандартов и процессов для обеспечения согласованности, качества и безопасности API в организации.

### Компоненты governance

```yaml
# api-governance-policy.yaml
governance:
  naming_conventions:
    resources:
      style: kebab-case
      plural: true
      examples:
        - /user-accounts
        - /order-items

    parameters:
      style: camelCase
      examples:
        - pageSize
        - sortOrder

    headers:
      style: X-Company-*
      examples:
        - X-Company-Request-Id
        - X-Company-Client-Version

  design_standards:
    versioning:
      strategy: url-path
      format: /v{major}/

    pagination:
      style: cursor-based
      default_limit: 20
      max_limit: 100
      parameters:
        - cursor
        - limit

    error_format:
      type: RFC7807
      required_fields:
        - type
        - title
        - status
        - detail
        - instance

  security_requirements:
    authentication:
      - OAuth 2.0
      - API Keys (internal only)

    transport: HTTPS only

    rate_limiting:
      enabled: true
      default_limit: 1000/hour

  documentation:
    format: OpenAPI 3.0+
    required_sections:
      - description
      - examples
      - error responses

    review_process:
      - API Design Review
      - Security Review
      - Documentation Review
```

### Автоматизация governance

```python
# Spectral - линтер для OpenAPI спецификаций
# .spectral.yaml

extends: spectral:oas

rules:
  # Правила именования
  path-must-be-kebab-case:
    description: "Paths must be kebab-case"
    given: "$.paths[*]~"
    then:
      function: pattern
      functionOptions:
        match: "^(/[a-z][a-z0-9-]*)+$"
    severity: error

  # Требования к документации
  operation-description:
    description: "All operations must have a description"
    given: "$.paths[*][*]"
    then:
      field: description
      function: truthy
    severity: error

  # Пагинация
  collection-must-have-pagination:
    description: "GET endpoints returning arrays must support pagination"
    given: "$.paths[*].get.responses.200.content.application/json.schema"
    then:
      function: schema
      functionOptions:
        schema:
          type: object
          required:
            - items
            - pagination
    severity: warn

  # Версионирование
  path-must-include-version:
    description: "All paths must include version"
    given: "$.paths[*]~"
    then:
      function: pattern
      functionOptions:
        match: "^/v[0-9]+/"
    severity: error

  # Безопасность
  security-must-be-defined:
    description: "All endpoints must have security defined"
    given: "$.paths[*][*]"
    then:
      field: security
      function: truthy
    severity: error
```

### CI/CD интеграция governance

```yaml
# .github/workflows/api-governance.yml
name: API Governance Check

on:
  pull_request:
    paths:
      - 'api/**/*.yaml'
      - 'api/**/*.json'

jobs:
  lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run Spectral
        uses: stoplightio/spectral-action@v0.8.10
        with:
          file_glob: 'api/**/*.yaml'
          spectral_ruleset: .spectral.yaml

      - name: Check breaking changes
        run: |
          npx @openapitools/openapi-diff \
            api/openapi-main.yaml \
            api/openapi.yaml \
            --fail-on-incompatible

      - name: Security scan
        run: |
          npx @42crunch/api-security-audit \
            api/openapi.yaml \
            --min-score 70

  design-review:
    runs-on: ubuntu-latest
    needs: lint
    steps:
      - name: Request design review
        if: contains(github.event.pull_request.labels.*.name, 'api-change')
        uses: actions/github-script@v6
        with:
          script: |
            github.rest.pulls.requestReviewers({
              owner: context.repo.owner,
              repo: context.repo.repo,
              pull_number: context.issue.number,
              team_reviewers: ['api-design-team']
            })
```

---

## Инструменты для управления жизненным циклом

### API Gateway

```yaml
# Kong API Gateway configuration
services:
  - name: orders-api
    url: http://orders-service:8080
    routes:
      - name: orders-v1
        paths:
          - /v1/orders
        strip_path: false
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local

      - name: jwt
        config:
          secret_is_base64: false
          claims_to_verify:
            - exp

      - name: request-transformer
        config:
          add:
            headers:
              - "X-Request-Id:$(uuid)"

      - name: response-transformer
        config:
          add:
            headers:
              - "X-API-Version:1.0"

      - name: prometheus
        config:
          per_consumer: true

  - name: orders-api-v2
    url: http://orders-service-v2:8080
    routes:
      - name: orders-v2
        paths:
          - /v2/orders
```

### API Management Platforms

```python
# Пример интеграции с Azure API Management
from azure.mgmt.apimanagement import ApiManagementClient
from azure.identity import DefaultAzureCredential

class APIManagementService:
    def __init__(self, subscription_id: str, resource_group: str, service_name: str):
        self.client = ApiManagementClient(
            credential=DefaultAzureCredential(),
            subscription_id=subscription_id
        )
        self.resource_group = resource_group
        self.service_name = service_name

    def create_api(self, api_id: str, openapi_spec_url: str):
        """Создание API из OpenAPI спецификации"""
        return self.client.api.begin_create_or_update(
            self.resource_group,
            self.service_name,
            api_id,
            {
                "format": "openapi-link",
                "value": openapi_spec_url,
                "path": f"/{api_id}",
                "protocols": ["https"]
            }
        ).result()

    def set_policy(self, api_id: str, policy_xml: str):
        """Установка политики для API"""
        return self.client.api_policy.create_or_update(
            self.resource_group,
            self.service_name,
            api_id,
            {
                "format": "xml",
                "value": policy_xml
            }
        )

    def deprecate_api(self, api_id: str, sunset_date: str):
        """Пометка API как deprecated"""
        deprecation_policy = f"""
        <policies>
            <inbound>
                <set-header name="Deprecation" exists-action="override">
                    <value>true</value>
                </set-header>
                <set-header name="Sunset" exists-action="override">
                    <value>{sunset_date}</value>
                </set-header>
            </inbound>
        </policies>
        """
        return self.set_policy(api_id, deprecation_policy)
```

### Developer Portal

```python
# Автоматическая публикация документации
from fastapi import FastAPI
from fastapi.openapi.utils import get_openapi
import httpx

app = FastAPI()

def publish_to_developer_portal():
    """Публикация OpenAPI spec в Developer Portal"""
    openapi_schema = get_openapi(
        title=app.title,
        version=app.version,
        description=app.description,
        routes=app.routes,
    )

    # Публикация в Readme.io
    response = httpx.put(
        "https://dash.readme.com/api/v1/api-specification",
        headers={
            "Authorization": f"Basic {README_API_KEY}",
            "Content-Type": "application/json"
        },
        json={"spec": openapi_schema}
    )

    return response.status_code == 200
```

---

## Best Practices

### 1. Design-First подход

```markdown
## Design-First Workflow

1. **Создать спецификацию** до написания кода
2. **Получить feedback** от потребителей API
3. **Итерировать** дизайн на основе обратной связи
4. **Зафиксировать** контракт
5. **Генерировать** серверные стабы и клиентские SDK
6. **Имплементировать** бизнес-логику
```

### 2. Backward Compatibility

```python
# Правила обратной совместимости

# ДОПУСТИМО (не ломает клиентов):
# - Добавление новых эндпоинтов
# - Добавление новых опциональных полей
# - Добавление новых опциональных query параметров
# - Добавление новых значений в enum (если клиент обрабатывает unknown)

# НЕДОПУСТИМО (breaking changes):
# - Удаление эндпоинтов
# - Удаление полей из ответа
# - Изменение типов данных
# - Переименование полей
# - Изменение обязательности полей
# - Изменение URL структуры

class BackwardCompatibilityChecker:
    @staticmethod
    def check_breaking_changes(old_spec: dict, new_spec: dict) -> list:
        """Проверка breaking changes между версиями спецификации"""
        breaking_changes = []

        # Проверка удаленных путей
        old_paths = set(old_spec.get("paths", {}).keys())
        new_paths = set(new_spec.get("paths", {}).keys())
        removed_paths = old_paths - new_paths

        if removed_paths:
            breaking_changes.append({
                "type": "removed_path",
                "paths": list(removed_paths)
            })

        # Проверка удаленных полей в схемах
        # ... дополнительные проверки

        return breaking_changes
```

### 3. API Registry

```python
# Централизованный реестр API
from dataclasses import dataclass, field
from typing import List, Dict
from datetime import datetime

@dataclass
class APIEntry:
    id: str
    name: str
    version: str
    status: str  # active, deprecated, retired
    owner_team: str
    spec_url: str
    documentation_url: str
    base_url: str
    created_at: datetime
    updated_at: datetime
    tags: List[str] = field(default_factory=list)
    dependencies: List[str] = field(default_factory=list)
    consumers: List[str] = field(default_factory=list)

class APIRegistry:
    def __init__(self):
        self.apis: Dict[str, APIEntry] = {}

    def register(self, api: APIEntry):
        """Регистрация нового API"""
        self.apis[api.id] = api

    def find_by_tag(self, tag: str) -> List[APIEntry]:
        """Поиск API по тегу"""
        return [api for api in self.apis.values() if tag in api.tags]

    def find_consumers(self, api_id: str) -> List[str]:
        """Найти всех потребителей API"""
        return self.apis[api_id].consumers if api_id in self.apis else []

    def impact_analysis(self, api_id: str) -> Dict:
        """Анализ влияния изменений API"""
        consumers = self.find_consumers(api_id)
        dependent_apis = [
            api for api in self.apis.values()
            if api_id in api.dependencies
        ]

        return {
            "direct_consumers": consumers,
            "dependent_apis": [api.id for api in dependent_apis],
            "total_impact": len(consumers) + len(dependent_apis)
        }
```

### 4. Changelog и Release Notes

```markdown
# API Changelog

## [2.1.0] - 2024-03-15

### Added
- Новый эндпоинт `GET /orders/{id}/history` для получения истории изменений заказа
- Поле `metadata` в ответе `GET /orders` для кастомных данных
- Поддержка фильтрации по дате создания: `?created_after=2024-01-01`

### Changed
- Улучшена производительность `GET /orders` на 40%
- Увеличен лимит пагинации с 100 до 200 элементов

### Deprecated
- Параметр `sort` будет удален в v3.0. Используйте `sort_by` и `sort_order`

### Fixed
- Исправлена ошибка с timezone в поле `created_at`
- Исправлен расчет `total_amount` при скидках

## [2.0.0] - 2024-01-01

### Breaking Changes
- Изменена структура ответа: данные теперь в поле `data`
- Удален эндпоинт `GET /orders/count`. Используйте `GET /orders?count_only=true`
- Переименовано поле `order_date` -> `created_at`

### Migration Guide
См. [Migration Guide v1 to v2](./migration/v1-to-v2.md)
```

---

## Примеры из реальных проектов

### Stripe API

```markdown
## Stripe API Versioning

Stripe использует date-based версионирование:
- Версия = дата релиза (например, 2024-01-01)
- Клиент указывает версию в заголовке Stripe-Version
- Без заголовка используется версия аккаунта
- Обратная совместимость внутри версии гарантирована
- Changelog для каждого изменения версии
```

```python
# Stripe-подобное версионирование
import stripe

stripe.api_key = "sk_test_..."
stripe.api_version = "2024-01-01"  # Фиксированная версия

# Или через заголовок в каждом запросе
charge = stripe.Charge.create(
    amount=2000,
    currency="usd",
    source="tok_visa",
    stripe_version="2024-01-01"  # Override для конкретного запроса
)
```

### GitHub API

```markdown
## GitHub API

- URL версионирование: api.github.com (v3 по умолчанию)
- GraphQL API: отдельный endpoint
- Deprecation через GitHub блог и email
- Preview features через Accept header
- Rate limiting: 5000 requests/hour (authenticated)
```

### Twilio API

```markdown
## Twilio API Lifecycle

1. **Beta** - API в разработке, может меняться
2. **GA (General Availability)** - стабильный, production-ready
3. **Maintenance** - только исправления безопасности
4. **Deprecated** - 12 месяцев до sunset
5. **End of Life** - API отключен

Twilio использует:
- URL версионирование (/2010-04-01/)
- Длинные даты релиза как версии
- Минимум 12 месяцев deprecation period
```

---

## Чек-лист API Lifecycle Management

```markdown
## Planning Phase
- [ ] Определены бизнес-требования
- [ ] Проведен анализ существующих API
- [ ] Определена целевая аудитория
- [ ] Установлены SLA и метрики

## Design Phase
- [ ] Создана OpenAPI спецификация
- [ ] Проведен design review
- [ ] Определена стратегия версионирования
- [ ] Спланированы breaking changes

## Development Phase
- [ ] Код соответствует спецификации
- [ ] Реализована валидация входных данных
- [ ] Добавлено логирование и метрики
- [ ] Написаны unit и integration тесты

## Testing Phase
- [ ] Пройдены все тесты
- [ ] Проведено нагрузочное тестирование
- [ ] Выполнен security audit
- [ ] Проверена обратная совместимость

## Deployment Phase
- [ ] Настроен API Gateway
- [ ] Сконфигурирован мониторинг
- [ ] Настроены алерты
- [ ] Опубликована документация

## Maintenance Phase
- [ ] Мониторинг работает корректно
- [ ] Собирается обратная связь
- [ ] Обновляется документация
- [ ] Отслеживается usage analytics

## Deprecation Phase
- [ ] Уведомлены все потребители
- [ ] Опубликован migration guide
- [ ] Добавлены deprecation headers
- [ ] Установлена sunset дата
```

---

## Заключение

Эффективное управление жизненным циклом API требует:

1. **Стандартизации** — единые правила для всех API
2. **Автоматизации** — CI/CD, линтинг, тестирование
3. **Прозрачности** — документация, changelog, метрики
4. **Коммуникации** — уведомления о изменениях
5. **Планирования** — продуманный deprecation процесс

Инвестиции в API Lifecycle Management окупаются снижением затрат на поддержку, улучшением developer experience и повышением надежности интеграций.

---

[prev: 05-pii](./11-standards-and-compliance/05-pii.md) | [next: 01-cdn](../09-caching/01-cdn.md)
