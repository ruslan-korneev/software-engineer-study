# API Gateway

## Что такое API Gateway

**API Gateway** — это сервер, который выступает единой точкой входа для всех клиентских запросов в микросервисной архитектуре. Он принимает запросы от клиентов, направляет их соответствующим сервисам и агрегирует ответы.

```
                                    ┌─────────────────┐
                                    │   Service A     │
                                    └────────┬────────┘
                                             │
┌──────────┐     ┌──────────────────┐        │
│  Client  │────▶│   API Gateway    │────────┼────────▶ Service B
└──────────┘     └──────────────────┘        │
                                             │
                                    ┌────────┴────────┐
                                    │   Service C     │
                                    └─────────────────┘
```

### Зачем нужен API Gateway

1. **Единая точка входа** — клиенты работают с одним API, а не с десятками микросервисов
2. **Инкапсуляция архитектуры** — внутренняя структура скрыта от клиентов
3. **Упрощение клиентов** — сложная логика вынесена на сервер
4. **Централизованная безопасность** — аутентификация в одном месте
5. **Оптимизация коммуникации** — агрегация запросов снижает latency

### Аналогия

API Gateway — это как **ресепшн в отеле**:
- Гость (клиент) обращается к ресепшн (gateway)
- Ресепшн знает, как связаться со всеми службами отеля
- Гость не ходит сам в прачечную, ресторан, службу уборки
- Ресепшн может отказать в обслуживании (security)
- Ресепшн ведёт журнал всех обращений (logging)

---

## Основные функции API Gateway

### 1. Маршрутизация запросов (Request Routing)

API Gateway определяет, какой сервис должен обработать запрос на основе URL, заголовков или других параметров.

```yaml
# Пример конфигурации маршрутов
routes:
  - path: /api/users/*
    service: user-service
    port: 8080

  - path: /api/orders/*
    service: order-service
    port: 8081

  - path: /api/products/*
    service: product-service
    port: 8082

  - path: /api/v2/*
    service: api-v2-service
    port: 8090
```

**Стратегии маршрутизации:**

```python
# Path-based routing
/api/users/123    → user-service
/api/orders/456   → order-service

# Header-based routing
X-API-Version: 2  → service-v2
X-API-Version: 1  → service-v1

# Query-based routing
?region=eu        → eu-service
?region=us        → us-service

# Method-based routing
GET /resource     → read-service
POST /resource    → write-service
```

### 2. Агрегация данных (API Composition)

Один запрос клиента может требовать данных от нескольких сервисов. Gateway агрегирует эти данные.

```python
# Без агрегации - 3 запроса от клиента:
# GET /users/123
# GET /orders?user_id=123
# GET /recommendations?user_id=123

# С агрегацией - 1 запрос:
# GET /users/123/dashboard

async def get_user_dashboard(user_id: str):
    """API Gateway агрегирует данные из нескольких сервисов."""

    # Параллельные запросы к сервисам
    user_task = asyncio.create_task(
        call_service("user-service", f"/users/{user_id}")
    )
    orders_task = asyncio.create_task(
        call_service("order-service", f"/orders?user_id={user_id}")
    )
    recommendations_task = asyncio.create_task(
        call_service("recommendation-service", f"/recommendations/{user_id}")
    )

    # Ожидание всех ответов
    user, orders, recommendations = await asyncio.gather(
        user_task, orders_task, recommendations_task
    )

    # Агрегация результатов
    return {
        "user": user,
        "recent_orders": orders[:5],
        "recommendations": recommendations[:10]
    }
```

**Преимущества агрегации:**
- Меньше сетевых запросов от клиента
- Меньше latency (особенно на мобильных устройствах)
- Упрощённый клиентский код
- Возможность оптимизации запросов на сервере

### 3. Аутентификация и авторизация

API Gateway централизует проверку безопасности.

```python
# Middleware для аутентификации
class AuthenticationMiddleware:
    def __init__(self, auth_service_url: str):
        self.auth_service = auth_service_url

    async def authenticate(self, request: Request) -> Optional[User]:
        # Извлечение токена
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            raise UnauthorizedException("Missing or invalid token")

        token = auth_header.split(" ")[1]

        # Валидация JWT локально (без запроса к auth-service)
        try:
            payload = jwt.decode(
                token,
                PUBLIC_KEY,
                algorithms=["RS256"]
            )
            return User(
                id=payload["sub"],
                roles=payload.get("roles", []),
                permissions=payload.get("permissions", [])
            )
        except jwt.ExpiredSignatureError:
            raise UnauthorizedException("Token expired")
        except jwt.InvalidTokenError:
            raise UnauthorizedException("Invalid token")

# Middleware для авторизации
class AuthorizationMiddleware:
    def __init__(self, policies: dict):
        self.policies = policies

    def authorize(self, user: User, resource: str, action: str) -> bool:
        policy = self.policies.get(resource)
        if not policy:
            return False

        required_roles = policy.get(action, [])
        return any(role in user.roles for role in required_roles)

# Пример политик
POLICIES = {
    "/api/admin/*": {
        "GET": ["admin"],
        "POST": ["admin"],
        "DELETE": ["superadmin"]
    },
    "/api/users/*": {
        "GET": ["user", "admin"],
        "POST": ["admin"],
        "PUT": ["user", "admin"]
    }
}
```

**Схемы аутентификации:**

```
┌─────────────────────────────────────────────────────────────┐
│                    Authentication Methods                    │
├─────────────────────────────────────────────────────────────┤
│                                                              │
│  API Key         │  Простой, для server-to-server           │
│  ────────────────┼─────────────────────────────────────     │
│  X-API-Key: xxx  │  Низкая безопасность, легко украсть      │
│                                                              │
│  JWT             │  Стандарт для web/mobile                  │
│  ────────────────┼─────────────────────────────────────     │
│  Bearer eyJ...   │  Stateless, содержит claims              │
│                                                              │
│  OAuth 2.0       │  Делегированная авторизация               │
│  ────────────────┼─────────────────────────────────────     │
│  Различные flows │  Сложнее, но гибче                        │
│                                                              │
│  mTLS            │  Взаимная аутентификация                  │
│  ────────────────┼─────────────────────────────────────     │
│  Клиент.сертиф.  │  Высокая безопасность, сложность          │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 4. Rate Limiting и Throttling

Защита сервисов от перегрузки и DDoS-атак.

```python
from datetime import datetime, timedelta
import redis

class RateLimiter:
    """Реализация rate limiting с использованием Redis."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def is_allowed(
        self,
        client_id: str,
        limit: int,
        window_seconds: int
    ) -> tuple[bool, dict]:
        """
        Sliding window rate limiting.

        Args:
            client_id: Идентификатор клиента (IP, user_id, API key)
            limit: Максимум запросов в окне
            window_seconds: Размер окна в секундах

        Returns:
            (allowed, metadata) - разрешён ли запрос и метаданные
        """
        now = datetime.utcnow()
        window_start = now - timedelta(seconds=window_seconds)
        key = f"rate_limit:{client_id}"

        pipe = self.redis.pipeline()

        # Удаляем старые записи
        pipe.zremrangebyscore(key, 0, window_start.timestamp())

        # Считаем текущие запросы
        pipe.zcard(key)

        # Добавляем текущий запрос
        pipe.zadd(key, {str(now.timestamp()): now.timestamp()})

        # Устанавливаем TTL
        pipe.expire(key, window_seconds)

        results = pipe.execute()
        current_count = results[1]

        remaining = max(0, limit - current_count - 1)
        reset_time = int((window_start + timedelta(seconds=window_seconds)).timestamp())

        metadata = {
            "X-RateLimit-Limit": limit,
            "X-RateLimit-Remaining": remaining,
            "X-RateLimit-Reset": reset_time
        }

        return current_count < limit, metadata


class TieredRateLimiter:
    """Rate limiting с разными лимитами для разных тарифов."""

    TIERS = {
        "free": {"requests": 100, "window": 3600},      # 100 req/hour
        "basic": {"requests": 1000, "window": 3600},    # 1000 req/hour
        "premium": {"requests": 10000, "window": 3600}, # 10000 req/hour
        "enterprise": {"requests": 100000, "window": 3600}
    }

    def get_limit(self, user_tier: str) -> tuple[int, int]:
        tier = self.TIERS.get(user_tier, self.TIERS["free"])
        return tier["requests"], tier["window"]
```

**Алгоритмы rate limiting:**

| Алгоритм | Описание | Pros | Cons |
|----------|----------|------|------|
| **Fixed Window** | Фиксированное окно времени | Простой | Burst на границе окон |
| **Sliding Window** | Скользящее окно | Точный | Больше памяти |
| **Token Bucket** | Токены накапливаются | Допускает burst | Сложнее |
| **Leaky Bucket** | Запросы "вытекают" равномерно | Стабильный поток | Не допускает burst |

### 5. Кэширование

API Gateway может кэшировать ответы для снижения нагрузки на сервисы.

```python
import hashlib
import json
from typing import Optional
import redis

class ResponseCache:
    """Кэширование ответов API."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def _generate_cache_key(
        self,
        method: str,
        path: str,
        query_params: dict,
        headers: dict
    ) -> str:
        """Генерация ключа кэша."""
        # Включаем только релевантные заголовки
        cache_relevant_headers = {
            k: v for k, v in headers.items()
            if k.lower() in ["accept", "accept-language", "authorization"]
        }

        key_data = {
            "method": method,
            "path": path,
            "query": sorted(query_params.items()),
            "headers": sorted(cache_relevant_headers.items())
        }

        key_string = json.dumps(key_data, sort_keys=True)
        return f"cache:{hashlib.sha256(key_string.encode()).hexdigest()}"

    def get(
        self,
        method: str,
        path: str,
        query_params: dict,
        headers: dict
    ) -> Optional[dict]:
        """Получение из кэша."""
        if method not in ["GET", "HEAD"]:
            return None

        key = self._generate_cache_key(method, path, query_params, headers)
        cached = self.redis.get(key)

        if cached:
            return json.loads(cached)
        return None

    def set(
        self,
        method: str,
        path: str,
        query_params: dict,
        headers: dict,
        response: dict,
        ttl: int
    ) -> None:
        """Сохранение в кэш."""
        key = self._generate_cache_key(method, path, query_params, headers)
        self.redis.setex(key, ttl, json.dumps(response))

    def invalidate_pattern(self, pattern: str) -> int:
        """Инвалидация по паттерну."""
        keys = self.redis.keys(f"cache:*{pattern}*")
        if keys:
            return self.redis.delete(*keys)
        return 0


# Конфигурация кэширования
CACHE_CONFIG = {
    "/api/products": {"ttl": 300, "cache": True},      # 5 минут
    "/api/categories": {"ttl": 3600, "cache": True},   # 1 час
    "/api/users/*/profile": {"ttl": 60, "cache": True}, # 1 минута
    "/api/orders": {"ttl": 0, "cache": False},         # Не кэшировать
}
```

**Стратегии кэширования:**

```
Cache-Control заголовки:
────────────────────────
Cache-Control: max-age=300           # Кэшировать 5 минут
Cache-Control: no-cache              # Валидировать перед использованием
Cache-Control: no-store              # Не кэшировать вообще
Cache-Control: private               # Только для одного пользователя
Cache-Control: public                # Можно кэшировать публично

Vary заголовок:
───────────────
Vary: Authorization                  # Отдельный кэш для каждого пользователя
Vary: Accept-Encoding                # Отдельный кэш для gzip/br
Vary: Accept-Language                # Отдельный кэш для языка
```

### 6. Логирование и мониторинг

```python
import time
import uuid
from dataclasses import dataclass
from typing import Optional
import structlog

logger = structlog.get_logger()

@dataclass
class RequestLog:
    """Структура лога запроса."""
    request_id: str
    timestamp: float
    method: str
    path: str
    query_params: dict
    client_ip: str
    user_agent: str
    user_id: Optional[str]

    # Response data
    status_code: int
    response_time_ms: float
    response_size_bytes: int

    # Routing data
    upstream_service: str
    upstream_response_time_ms: float

    # Error data
    error: Optional[str]


class LoggingMiddleware:
    """Middleware для структурированного логирования."""

    async def __call__(self, request: Request, call_next):
        request_id = request.headers.get(
            "X-Request-ID",
            str(uuid.uuid4())
        )
        start_time = time.time()

        # Добавляем request_id в контекст
        structlog.contextvars.bind_contextvars(request_id=request_id)

        # Логируем входящий запрос
        logger.info(
            "request_started",
            method=request.method,
            path=request.url.path,
            client_ip=request.client.host
        )

        try:
            response = await call_next(request)

            # Логируем успешный ответ
            duration = (time.time() - start_time) * 1000
            logger.info(
                "request_completed",
                status_code=response.status_code,
                duration_ms=round(duration, 2)
            )

            # Добавляем заголовки для трейсинга
            response.headers["X-Request-ID"] = request_id
            response.headers["X-Response-Time"] = f"{duration:.2f}ms"

            return response

        except Exception as e:
            duration = (time.time() - start_time) * 1000
            logger.error(
                "request_failed",
                error=str(e),
                duration_ms=round(duration, 2)
            )
            raise


# Метрики для Prometheus
from prometheus_client import Counter, Histogram

REQUEST_COUNT = Counter(
    "api_gateway_requests_total",
    "Total number of requests",
    ["method", "path", "status"]
)

REQUEST_LATENCY = Histogram(
    "api_gateway_request_duration_seconds",
    "Request latency in seconds",
    ["method", "path"],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

UPSTREAM_LATENCY = Histogram(
    "api_gateway_upstream_duration_seconds",
    "Upstream service latency",
    ["service"],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)
```

### 7. Трансформация запросов и ответов

```python
class RequestTransformer:
    """Трансформация запросов перед отправкой в сервис."""

    def transform(self, request: Request, route_config: dict) -> Request:
        # Добавление заголовков
        request.headers["X-Forwarded-For"] = request.client.host
        request.headers["X-Forwarded-Proto"] = request.url.scheme
        request.headers["X-Request-ID"] = str(uuid.uuid4())

        # Удаление чувствительных заголовков от клиента
        sensitive_headers = ["X-Internal-Token", "X-Debug"]
        for header in sensitive_headers:
            request.headers.pop(header, None)

        # Переписывание пути
        if route_config.get("strip_prefix"):
            prefix = route_config["strip_prefix"]
            request.url.path = request.url.path.replace(prefix, "", 1)

        # Добавление query параметров
        if route_config.get("add_query_params"):
            for key, value in route_config["add_query_params"].items():
                request.query_params[key] = value

        return request


class ResponseTransformer:
    """Трансформация ответов перед отправкой клиенту."""

    def transform(self, response: Response, route_config: dict) -> Response:
        # Удаление внутренних заголовков
        internal_headers = [
            "X-Internal-Service-Version",
            "X-Debug-Info",
            "Server"  # Скрываем информацию о сервере
        ]
        for header in internal_headers:
            response.headers.pop(header, None)

        # Добавление CORS заголовков
        if route_config.get("cors_enabled"):
            response.headers["Access-Control-Allow-Origin"] = "*"
            response.headers["Access-Control-Allow-Methods"] = "GET, POST, PUT, DELETE"
            response.headers["Access-Control-Allow-Headers"] = "Content-Type, Authorization"

        # Добавление security заголовков
        response.headers["X-Content-Type-Options"] = "nosniff"
        response.headers["X-Frame-Options"] = "DENY"
        response.headers["X-XSS-Protection"] = "1; mode=block"

        return response


# Пример: преобразование формата ответа
class ResponseFormatter:
    """Форматирование ответов в единый формат."""

    def format_success(self, data: any, meta: dict = None) -> dict:
        return {
            "success": True,
            "data": data,
            "meta": meta or {},
            "errors": None
        }

    def format_error(self, error: Exception, status_code: int) -> dict:
        return {
            "success": False,
            "data": None,
            "meta": {},
            "errors": [{
                "code": error.__class__.__name__,
                "message": str(error),
                "status": status_code
            }]
        }
```

---

## Архитектурные паттерны

### 1. Backend for Frontend (BFF)

Отдельный API Gateway для каждого типа клиента.

```
┌─────────────┐     ┌──────────────────┐
│   Web App   │────▶│    Web BFF       │───┐
└─────────────┘     └──────────────────┘   │
                                            │
┌─────────────┐     ┌──────────────────┐   │    ┌─────────────┐
│ Mobile App  │────▶│   Mobile BFF     │───┼───▶│  Services   │
└─────────────┘     └──────────────────┘   │    └─────────────┘
                                            │
┌─────────────┐     ┌──────────────────┐   │
│ Third Party │────▶│   Public API     │───┘
└─────────────┘     └──────────────────┘
```

**Пример реализации BFF:**

```python
# Mobile BFF - оптимизирован для мобильных устройств
class MobileBFF:
    """BFF для мобильных приложений."""

    async def get_home_screen(self, user_id: str) -> dict:
        """
        Агрегирует данные для главного экрана мобильного приложения.
        Минимизирует количество данных для экономии трафика.
        """
        async with asyncio.TaskGroup() as tg:
            user_task = tg.create_task(self.get_user_summary(user_id))
            feed_task = tg.create_task(self.get_compact_feed(user_id, limit=10))
            notifications_task = tg.create_task(self.get_notification_count(user_id))

        return {
            "user": user_task.result(),
            "feed": feed_task.result(),
            "unread_notifications": notifications_task.result()
        }

    async def get_user_summary(self, user_id: str) -> dict:
        """Минимальные данные о пользователе."""
        user = await self.user_service.get(user_id)
        return {
            "name": user["name"],
            "avatar_url": user["avatar_url"],
            "notification_settings": user["push_enabled"]
        }


# Web BFF - оптимизирован для веб-приложений
class WebBFF:
    """BFF для веб-приложений."""

    async def get_dashboard(self, user_id: str) -> dict:
        """
        Агрегирует данные для дашборда.
        Включает больше данных, так как веб менее ограничен.
        """
        async with asyncio.TaskGroup() as tg:
            user_task = tg.create_task(self.get_full_user_profile(user_id))
            feed_task = tg.create_task(self.get_detailed_feed(user_id, limit=50))
            stats_task = tg.create_task(self.get_user_statistics(user_id))
            recommendations_task = tg.create_task(self.get_recommendations(user_id))

        return {
            "user": user_task.result(),
            "feed": feed_task.result(),
            "statistics": stats_task.result(),
            "recommendations": recommendations_task.result()
        }
```

**Когда использовать BFF:**
- Разные клиенты имеют разные требования к данным
- Мобильное приложение нуждается в оптимизации трафика
- Разные команды разрабатывают разные клиенты
- Нужна изоляция изменений между платформами

### 2. API Composition Pattern

Объединение нескольких API в один запрос.

```python
from typing import List, Dict, Any
import asyncio

class APIComposer:
    """Композиция API - объединение нескольких вызовов."""

    def __init__(self, services: Dict[str, ServiceClient]):
        self.services = services

    async def compose(
        self,
        queries: List[Dict[str, Any]]
    ) -> Dict[str, Any]:
        """
        Выполняет несколько запросов и объединяет результаты.

        queries = [
            {"alias": "user", "service": "user-service", "path": "/users/123"},
            {"alias": "orders", "service": "order-service", "path": "/orders?user_id=123"},
            {"alias": "balance", "service": "billing-service", "path": "/balance/123"}
        ]
        """
        tasks = {}
        async with asyncio.TaskGroup() as tg:
            for query in queries:
                alias = query["alias"]
                service = self.services[query["service"]]
                tasks[alias] = tg.create_task(
                    service.request(
                        method=query.get("method", "GET"),
                        path=query["path"],
                        data=query.get("data")
                    )
                )

        return {alias: task.result() for alias, task in tasks.items()}


# GraphQL-like запросы через REST
@app.post("/api/compose")
async def compose_api(request: ComposeRequest):
    """
    POST /api/compose
    {
        "queries": [
            {
                "alias": "user",
                "service": "user-service",
                "path": "/users/123",
                "fields": ["id", "name", "email"]  # Выбираем только нужные поля
            },
            {
                "alias": "orders",
                "service": "order-service",
                "path": "/orders",
                "params": {"user_id": "123", "limit": 5}
            }
        ]
    }
    """
    composer = APIComposer(services)
    results = await composer.compose(request.queries)

    # Фильтруем поля если указаны
    for query in request.queries:
        if "fields" in query:
            alias = query["alias"]
            results[alias] = filter_fields(results[alias], query["fields"])

    return results
```

### 3. Strangler Fig Pattern

Постепенная миграция с монолита на микросервисы через API Gateway.

```
Этап 1: Gateway перед монолитом
──────────────────────────────────
┌────────┐     ┌─────────┐     ┌──────────┐
│ Client │────▶│ Gateway │────▶│ Monolith │
└────────┘     └─────────┘     └──────────┘

Этап 2: Часть функционала в новом сервисе
─────────────────────────────────────────────
                               ┌──────────────┐
                          ┌───▶│ New Service  │
┌────────┐     ┌─────────┐│    └──────────────┘
│ Client │────▶│ Gateway │┤
└────────┘     └─────────┘│    ┌──────────────┐
                          └───▶│   Monolith   │
                               └──────────────┘

Этап 3: Большинство функций вынесено
───────────────────────────────────────
                               ┌───────────┐
                          ┌───▶│ Service A │
                          │    └───────────┘
┌────────┐     ┌─────────┐│    ┌───────────┐
│ Client │────▶│ Gateway │├───▶│ Service B │
└────────┘     └─────────┘│    └───────────┘
                          │    ┌───────────┐
                          └───▶│ Legacy    │ (оставшийся код)
                               └───────────┘
```

```python
class StranglerRouter:
    """Роутер для постепенной миграции."""

    def __init__(self):
        # Маршруты к новым сервисам
        self.new_routes = {
            "/api/users": "user-service",
            "/api/orders": "order-service",
            # Постепенно добавляем маршруты
        }

        # Все остальное идёт в монолит
        self.legacy_url = "http://monolith:8080"

    async def route(self, request: Request) -> Response:
        path = request.url.path

        # Проверяем, есть ли маршрут к новому сервису
        for route_prefix, service in self.new_routes.items():
            if path.startswith(route_prefix):
                return await self.forward_to_service(request, service)

        # Иначе отправляем в монолит
        return await self.forward_to_legacy(request)

    async def forward_to_service(
        self,
        request: Request,
        service: str
    ) -> Response:
        service_url = await self.discover_service(service)
        return await self.proxy_request(request, service_url)

    async def forward_to_legacy(self, request: Request) -> Response:
        return await self.proxy_request(request, self.legacy_url)
```

---

## Популярные решения

### 1. Kong

Популярный open-source API Gateway на базе NGINX.

```yaml
# kong.yml - декларативная конфигурация
_format_version: "3.0"

services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: user-routes
        paths:
          - /api/users
        strip_path: true
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
          redis_host: redis
      - name: key-auth
        config:
          key_names:
            - X-API-Key
      - name: cors
        config:
          origins:
            - "*"
          methods:
            - GET
            - POST
            - PUT
            - DELETE

  - name: order-service
    url: http://order-service:8080
    routes:
      - name: order-routes
        paths:
          - /api/orders
    plugins:
      - name: jwt
        config:
          claims_to_verify:
            - exp
      - name: request-transformer
        config:
          add:
            headers:
              - "X-Consumer-ID:$(consumer.id)"

consumers:
  - username: mobile-app
    keyauth_credentials:
      - key: mobile-api-key-xxx

  - username: web-app
    jwt_secrets:
      - key: web-app
        secret: shared-secret-xxx
```

**Kong плагины:**
- `rate-limiting` - ограничение запросов
- `key-auth` / `jwt` / `oauth2` - аутентификация
- `cors` - CORS headers
- `request-transformer` - модификация запросов
- `response-transformer` - модификация ответов
- `prometheus` - метрики
- `zipkin` - distributed tracing

### 2. AWS API Gateway

Managed решение от AWS.

```yaml
# serverless.yml (Serverless Framework)
service: my-api

provider:
  name: aws
  runtime: python3.11

functions:
  getUsers:
    handler: handlers.get_users
    events:
      - http:
          path: users
          method: get
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId: !Ref ApiGatewayAuthorizer

  createUser:
    handler: handlers.create_user
    events:
      - http:
          path: users
          method: post
          cors: true
          request:
            schemas:
              application/json: ${file(schemas/create_user.json)}

resources:
  Resources:
    # API Gateway настройки
    ApiGatewayRestApi:
      Type: AWS::ApiGateway::RestApi
      Properties:
        Name: my-api
        Description: My API Gateway

    # Throttling
    ApiGatewayUsagePlan:
      Type: AWS::ApiGateway::UsagePlan
      Properties:
        UsagePlanName: basic-plan
        Throttle:
          BurstLimit: 100
          RateLimit: 50
        Quota:
          Limit: 10000
          Period: MONTH

    # WAF для защиты
    WebACL:
      Type: AWS::WAFv2::WebACL
      Properties:
        DefaultAction:
          Allow: {}
        Rules:
          - Name: RateLimitRule
            Priority: 1
            Action:
              Block: {}
            Statement:
              RateBasedStatement:
                Limit: 2000
                AggregateKeyType: IP
```

**Особенности AWS API Gateway:**
- Интеграция с Lambda, ECS, EC2
- Встроенный WAF
- Автоматическое масштабирование
- Поддержка WebSocket
- Canary deployments

### 3. NGINX как API Gateway

```nginx
# nginx.conf

upstream user_service {
    server user-service-1:8080 weight=5;
    server user-service-2:8080 weight=5;
    keepalive 32;
}

upstream order_service {
    server order-service-1:8080;
    server order-service-2:8080;
    keepalive 32;
}

# Rate limiting zone
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;
limit_req_zone $http_x_api_key zone=api_key_limit:10m rate=100r/s;

# Cache zone
proxy_cache_path /var/cache/nginx levels=1:2 keys_zone=api_cache:10m max_size=1g inactive=60m;

server {
    listen 80;
    server_name api.example.com;

    # Security headers
    add_header X-Frame-Options "DENY" always;
    add_header X-Content-Type-Options "nosniff" always;
    add_header X-XSS-Protection "1; mode=block" always;

    # Logging
    access_log /var/log/nginx/api_access.log json_combined;
    error_log /var/log/nginx/api_error.log;

    # Rate limiting
    limit_req zone=api_limit burst=20 nodelay;

    # Health check endpoint
    location /health {
        access_log off;
        return 200 "OK";
    }

    # User service
    location /api/users {
        # Auth check
        auth_request /auth;
        auth_request_set $user_id $upstream_http_x_user_id;

        # Add headers
        proxy_set_header X-User-ID $user_id;
        proxy_set_header X-Request-ID $request_id;
        proxy_set_header X-Real-IP $remote_addr;

        # Caching for GET requests
        proxy_cache api_cache;
        proxy_cache_methods GET HEAD;
        proxy_cache_valid 200 5m;
        proxy_cache_key "$scheme$request_method$host$request_uri$http_authorization";

        proxy_pass http://user_service;
    }

    # Order service
    location /api/orders {
        auth_request /auth;
        auth_request_set $user_id $upstream_http_x_user_id;

        proxy_set_header X-User-ID $user_id;
        proxy_pass http://order_service;
    }

    # Auth subrequest
    location = /auth {
        internal;
        proxy_pass http://auth-service:8080/validate;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
        proxy_set_header X-Original-Method $request_method;
    }
}
```

### 4. Traefik

Современный edge router с автоматическим service discovery.

```yaml
# docker-compose.yml
version: '3.8'

services:
  traefik:
    image: traefik:v3.0
    command:
      - "--api.insecure=true"
      - "--providers.docker=true"
      - "--providers.docker.exposedbydefault=false"
      - "--entrypoints.web.address=:80"
      - "--entrypoints.websecure.address=:443"
      - "--certificatesresolvers.letsencrypt.acme.httpchallenge=true"
      - "--certificatesresolvers.letsencrypt.acme.email=admin@example.com"
      - "--certificatesresolvers.letsencrypt.acme.storage=/letsencrypt/acme.json"
      - "--metrics.prometheus=true"
    ports:
      - "80:80"
      - "443:443"
      - "8080:8080"  # Dashboard
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock:ro
      - ./letsencrypt:/letsencrypt

  user-service:
    image: user-service:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.users.rule=Host(`api.example.com`) && PathPrefix(`/api/users`)"
      - "traefik.http.routers.users.entrypoints=websecure"
      - "traefik.http.routers.users.tls.certresolver=letsencrypt"
      # Middleware
      - "traefik.http.routers.users.middlewares=auth,ratelimit"
      - "traefik.http.middlewares.auth.forwardauth.address=http://auth-service:8080/validate"
      - "traefik.http.middlewares.ratelimit.ratelimit.average=100"
      - "traefik.http.middlewares.ratelimit.ratelimit.burst=50"
    deploy:
      replicas: 3

  order-service:
    image: order-service:latest
    labels:
      - "traefik.enable=true"
      - "traefik.http.routers.orders.rule=Host(`api.example.com`) && PathPrefix(`/api/orders`)"
      - "traefik.http.routers.orders.entrypoints=websecure"
      - "traefik.http.routers.orders.middlewares=auth,ratelimit,retry"
      - "traefik.http.middlewares.retry.retry.attempts=3"
```

```yaml
# traefik.yml - файловая конфигурация
entryPoints:
  web:
    address: ":80"
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https

  websecure:
    address: ":443"

http:
  routers:
    api-router:
      rule: "Host(`api.example.com`)"
      service: api-service
      middlewares:
        - auth
        - ratelimit
        - compress

  middlewares:
    auth:
      forwardAuth:
        address: "http://auth-service:8080/validate"
        trustForwardHeader: true
        authResponseHeaders:
          - "X-User-ID"
          - "X-User-Role"

    ratelimit:
      rateLimit:
        average: 100
        burst: 50
        period: 1m

    compress:
      compress:
        excludedContentTypes:
          - text/event-stream

  services:
    api-service:
      loadBalancer:
        servers:
          - url: "http://backend-1:8080"
          - url: "http://backend-2:8080"
        healthCheck:
          path: /health
          interval: 10s
```

### 5. Ambassador (Emissary-Ingress)

Kubernetes-native API Gateway на базе Envoy.

```yaml
# ambassador-mapping.yaml
apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: user-service-mapping
  namespace: default
spec:
  hostname: api.example.com
  prefix: /api/users/
  service: user-service:8080
  timeout_ms: 30000
  retry_policy:
    retry_on: "5xx"
    num_retries: 3
  labels:
    ambassador:
      - request_label:
        - user-service
---
apiVersion: getambassador.io/v3alpha1
kind: Mapping
metadata:
  name: order-service-mapping
spec:
  prefix: /api/orders/
  service: order-service:8080
  circuit_breakers:
    - priority: default
      max_connections: 1000
      max_pending_requests: 1000
      max_requests: 1000
---
# Rate limiting
apiVersion: getambassador.io/v3alpha1
kind: RateLimitService
metadata:
  name: rate-limit
spec:
  service: rate-limit-service:8080
  protocol_version: v3
---
# Authentication
apiVersion: getambassador.io/v3alpha1
kind: Filter
metadata:
  name: jwt-filter
spec:
  JWT:
    jwksURI: https://auth.example.com/.well-known/jwks.json
    validAlgorithms:
      - RS256
    audience: api.example.com
    requireAudience: true
---
apiVersion: getambassador.io/v3alpha1
kind: FilterPolicy
metadata:
  name: jwt-policy
spec:
  rules:
    - host: api.example.com
      path: /api/*
      filters:
        - name: jwt-filter
```

### Сравнение решений

| Критерий | Kong | AWS API GW | NGINX | Traefik | Ambassador |
|----------|------|------------|-------|---------|------------|
| **Тип** | Open Source | Managed | Open Source | Open Source | Open Source |
| **База** | NGINX/Lua | AWS | NGINX | Go | Envoy |
| **Kubernetes** | Хорошо | Внешний | Средне | Отлично | Отлично |
| **Плагины** | Много | Средне | Мало | Средне | Много |
| **Service Discovery** | Да | AWS только | Нет | Да | Да |
| **Сложность** | Средняя | Низкая | Высокая | Низкая | Средняя |
| **Цена** | Free/Enterprise | Pay-per-use | Free | Free/Enterprise | Free/Enterprise |

---

## API Gateway vs Load Balancer vs Reverse Proxy

```
┌───────────────────────────────────────────────────────────────────────────┐
│                        Сравнение компонентов                               │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  Reverse Proxy                                                             │
│  ─────────────                                                             │
│  • Пересылает запросы от имени клиента                                     │
│  • Скрывает бэкенд серверы                                                 │
│  • SSL termination                                                         │
│  • Кэширование                                                             │
│  • Сжатие                                                                  │
│  Пример: NGINX, HAProxy                                                    │
│                                                                            │
│  Load Balancer                                                             │
│  ─────────────                                                             │
│  • Распределяет нагрузку между серверами                                   │
│  • Health checks                                                           │
│  • Различные алгоритмы балансировки                                        │
│  • Session affinity                                                        │
│  • Layer 4 (TCP) или Layer 7 (HTTP)                                        │
│  Пример: HAProxy, AWS ALB, NGINX                                           │
│                                                                            │
│  API Gateway                                                               │
│  ───────────                                                               │
│  • Всё вышеперечисленное +                                                 │
│  • Аутентификация/Авторизация                                              │
│  • Rate limiting                                                           │
│  • Request/Response трансформация                                          │
│  • API версионирование                                                     │
│  • Агрегация данных                                                        │
│  • API analytics                                                           │
│  • Developer portal                                                        │
│  Пример: Kong, AWS API Gateway, Apigee                                     │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘

Архитектурно:

Internet → CDN → Load Balancer → API Gateway → Services
                      │
                      ↓
            (Multiple API Gateway instances)
```

**Когда что использовать:**

| Сценарий | Решение |
|----------|---------|
| Простой веб-сервер | Reverse Proxy (NGINX) |
| Несколько одинаковых бэкендов | Load Balancer |
| Микросервисы с API | API Gateway |
| Публичный API для партнёров | API Gateway + Developer Portal |
| Высокие требования к безопасности | API Gateway + WAF |

---

## Безопасность API Gateway

### 1. Защита от атак

```python
class SecurityMiddleware:
    """Middleware безопасности для API Gateway."""

    # Защита от SQL Injection и XSS
    DANGEROUS_PATTERNS = [
        r"(\%27)|(\')|(\-\-)|(\%23)|(#)",  # SQL Injection
        r"((\%3C)|<)((\%2F)|\/)*[a-z0-9\%]+((\%3E)|>)",  # XSS
        r"((\%3C)|<)((\%69)|i|(\%49))((\%6D)|m|(\%4D))((\%67)|g|(\%47))",  # XSS img
    ]

    def __init__(self):
        self.patterns = [re.compile(p, re.IGNORECASE) for p in self.DANGEROUS_PATTERNS]

    def check_injection(self, value: str) -> bool:
        """Проверка на опасные паттерны."""
        for pattern in self.patterns:
            if pattern.search(value):
                return True
        return False

    async def validate_request(self, request: Request) -> None:
        # Проверка URL
        if self.check_injection(str(request.url)):
            raise SecurityException("Potential injection in URL")

        # Проверка заголовков
        for header, value in request.headers.items():
            if self.check_injection(value):
                raise SecurityException(f"Potential injection in header: {header}")

        # Проверка body
        if request.body:
            body = await request.body()
            if self.check_injection(body.decode()):
                raise SecurityException("Potential injection in body")


class RequestValidator:
    """Валидация входящих запросов."""

    MAX_BODY_SIZE = 10 * 1024 * 1024  # 10MB
    MAX_HEADER_SIZE = 8 * 1024  # 8KB
    ALLOWED_CONTENT_TYPES = [
        "application/json",
        "application/x-www-form-urlencoded",
        "multipart/form-data"
    ]

    async def validate(self, request: Request) -> None:
        # Размер body
        content_length = request.headers.get("content-length")
        if content_length and int(content_length) > self.MAX_BODY_SIZE:
            raise ValidationError("Request body too large")

        # Content-Type
        content_type = request.headers.get("content-type", "")
        if request.method in ["POST", "PUT", "PATCH"]:
            if not any(ct in content_type for ct in self.ALLOWED_CONTENT_TYPES):
                raise ValidationError(f"Unsupported content type: {content_type}")

        # Проверка на опасные заголовки
        dangerous_headers = ["X-Forwarded-Host", "X-Original-URL"]
        for header in dangerous_headers:
            if header in request.headers:
                del request.headers[header]
```

### 2. TLS/mTLS

```yaml
# Kong с mTLS
services:
  - name: internal-service
    url: https://internal-service:8443
    client_certificate:
      id: client-cert-uuid
    tls_verify: true
    ca_certificates:
      - ca-cert-uuid

certificates:
  - id: client-cert-uuid
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
    key: |
      -----BEGIN RSA PRIVATE KEY-----
      ...
      -----END RSA PRIVATE KEY-----

ca_certificates:
  - id: ca-cert-uuid
    cert: |
      -----BEGIN CERTIFICATE-----
      ...
      -----END CERTIFICATE-----
```

```nginx
# NGINX с mTLS
server {
    listen 443 ssl;

    # Server certificate
    ssl_certificate /etc/nginx/ssl/server.crt;
    ssl_certificate_key /etc/nginx/ssl/server.key;

    # Client certificate verification
    ssl_client_certificate /etc/nginx/ssl/ca.crt;
    ssl_verify_client on;
    ssl_verify_depth 2;

    # Modern TLS config
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
    ssl_prefer_server_ciphers on;

    location /api {
        # Pass client cert info to backend
        proxy_set_header X-Client-Cert $ssl_client_cert;
        proxy_set_header X-Client-Cert-DN $ssl_client_s_dn;
        proxy_pass http://backend;
    }
}
```

### 3. OAuth 2.0 / OIDC интеграция

```python
from authlib.integrations.starlette_client import OAuth

oauth = OAuth()
oauth.register(
    name='keycloak',
    client_id='api-gateway',
    client_secret='secret',
    server_metadata_url='https://keycloak.example.com/realms/main/.well-known/openid-configuration',
    client_kwargs={'scope': 'openid profile email'}
)

class OAuthMiddleware:
    """OAuth 2.0 middleware для API Gateway."""

    def __init__(self, oauth_client):
        self.oauth = oauth_client

    async def validate_token(self, request: Request) -> dict:
        auth_header = request.headers.get("Authorization")
        if not auth_header or not auth_header.startswith("Bearer "):
            raise UnauthorizedException("Missing bearer token")

        token = auth_header.split(" ")[1]

        # Интроспекция токена
        async with aiohttp.ClientSession() as session:
            async with session.post(
                self.oauth.introspection_endpoint,
                data={
                    "token": token,
                    "client_id": self.oauth.client_id,
                    "client_secret": self.oauth.client_secret
                }
            ) as response:
                token_info = await response.json()

                if not token_info.get("active"):
                    raise UnauthorizedException("Token is not active")

                return token_info
```

### 4. Web Application Firewall (WAF)

```yaml
# AWS WAF Rules
Resources:
  APIGatewayWebACL:
    Type: AWS::WAFv2::WebACL
    Properties:
      DefaultAction:
        Allow: {}
      Scope: REGIONAL
      Rules:
        # Rate limiting
        - Name: RateLimitRule
          Priority: 1
          Action:
            Block: {}
          Statement:
            RateBasedStatement:
              Limit: 2000
              AggregateKeyType: IP
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: RateLimitRule

        # SQL Injection protection
        - Name: SQLInjectionRule
          Priority: 2
          Action:
            Block: {}
          Statement:
            SqliMatchStatement:
              FieldToMatch:
                Body: {}
              TextTransformations:
                - Priority: 0
                  Type: URL_DECODE
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: SQLInjectionRule

        # XSS protection
        - Name: XSSRule
          Priority: 3
          Action:
            Block: {}
          Statement:
            XssMatchStatement:
              FieldToMatch:
                Body: {}
              TextTransformations:
                - Priority: 0
                  Type: HTML_ENTITY_DECODE
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: XSSRule

        # Geo blocking (example: block certain countries)
        - Name: GeoBlockRule
          Priority: 4
          Action:
            Block: {}
          Statement:
            GeoMatchStatement:
              CountryCodes:
                - XX  # Replace with actual country codes
          VisibilityConfig:
            SampledRequestsEnabled: true
            CloudWatchMetricsEnabled: true
            MetricName: GeoBlockRule
```

---

## Best Practices

### 1. Проектирование

```
✅ DO:
─────
• Держите gateway stateless - храните состояние во внешних системах
• Используйте health checks для upstream сервисов
• Реализуйте graceful degradation при недоступности сервисов
• Версионируйте API (/v1/, /v2/)
• Используйте correlation IDs для трейсинга
• Документируйте API (OpenAPI/Swagger)
• Мониторьте latency, error rates, throughput

❌ DON'T:
─────────
• Не размещайте бизнес-логику в gateway
• Не кэшируйте авторизационные решения надолго
• Не храните секреты в конфигурации (используйте Vault)
• Не игнорируйте таймауты к upstream сервисам
• Не забывайте про CORS для браузерных клиентов
```

### 2. Resilience Patterns

```python
import asyncio
from circuitbreaker import circuit

class ResilientGateway:
    """Gateway с паттернами устойчивости."""

    def __init__(self):
        self.circuit_breakers = {}

    @circuit(failure_threshold=5, recovery_timeout=30)
    async def call_service(self, service: str, request: dict) -> dict:
        """Вызов сервиса с circuit breaker."""
        timeout = aiohttp.ClientTimeout(total=5)

        async with aiohttp.ClientSession(timeout=timeout) as session:
            async with session.request(**request) as response:
                if response.status >= 500:
                    raise ServiceUnavailableError(f"{service} returned {response.status}")
                return await response.json()

    async def call_with_retry(
        self,
        service: str,
        request: dict,
        max_retries: int = 3,
        backoff_factor: float = 0.5
    ) -> dict:
        """Вызов с retry и exponential backoff."""
        last_exception = None

        for attempt in range(max_retries):
            try:
                return await self.call_service(service, request)
            except (aiohttp.ClientError, asyncio.TimeoutError) as e:
                last_exception = e
                if attempt < max_retries - 1:
                    wait_time = backoff_factor * (2 ** attempt)
                    await asyncio.sleep(wait_time)

        raise last_exception

    async def call_with_fallback(
        self,
        service: str,
        request: dict,
        fallback_data: dict
    ) -> dict:
        """Вызов с fallback при ошибке."""
        try:
            return await self.call_service(service, request)
        except Exception as e:
            logger.warning(f"Service {service} failed, using fallback: {e}")
            return fallback_data

    async def call_with_timeout(
        self,
        service: str,
        request: dict,
        timeout_seconds: float = 5.0
    ) -> dict:
        """Вызов с жёстким таймаутом."""
        try:
            return await asyncio.wait_for(
                self.call_service(service, request),
                timeout=timeout_seconds
            )
        except asyncio.TimeoutError:
            raise GatewayTimeoutError(f"Service {service} timed out")
```

### 3. Конфигурация для production

```yaml
# config/gateway.yaml
gateway:
  # Server settings
  server:
    host: 0.0.0.0
    port: 8080
    workers: auto  # CPU cores * 2 + 1
    keepalive_timeout: 75

  # Connection pool
  upstream:
    max_connections: 1000
    max_connections_per_host: 100
    connection_timeout: 5s
    read_timeout: 30s
    write_timeout: 30s
    keepalive_connections: 100

  # Rate limiting
  rate_limiting:
    enabled: true
    default_limit: 100
    window: 60s
    redis:
      host: redis-cluster
      port: 6379
      db: 0

  # Caching
  caching:
    enabled: true
    default_ttl: 300
    max_size: 1GB
    redis:
      host: redis-cache
      port: 6379

  # Security
  security:
    cors:
      enabled: true
      allow_origins:
        - https://app.example.com
        - https://admin.example.com
      allow_methods:
        - GET
        - POST
        - PUT
        - DELETE
      allow_headers:
        - Authorization
        - Content-Type
      max_age: 86400

    tls:
      enabled: true
      min_version: TLSv1.2
      cipher_suites:
        - TLS_AES_256_GCM_SHA384
        - TLS_CHACHA20_POLY1305_SHA256

  # Observability
  observability:
    logging:
      level: INFO
      format: json
      include_body: false  # Не логируем тело запроса

    metrics:
      enabled: true
      endpoint: /metrics

    tracing:
      enabled: true
      jaeger:
        agent_host: jaeger-agent
        agent_port: 6831
        sampler_rate: 0.1  # 10% requests
```

### 4. Health Checks

```python
from enum import Enum
from dataclasses import dataclass
from typing import Dict, List

class HealthStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"

@dataclass
class ServiceHealth:
    name: str
    status: HealthStatus
    latency_ms: float
    last_check: datetime
    error: Optional[str] = None

class HealthChecker:
    """Проверка здоровья upstream сервисов."""

    def __init__(self, services: List[str]):
        self.services = services
        self.health_cache: Dict[str, ServiceHealth] = {}

    async def check_service(self, service: str) -> ServiceHealth:
        """Проверка одного сервиса."""
        start = time.time()

        try:
            async with aiohttp.ClientSession() as session:
                async with session.get(
                    f"http://{service}/health",
                    timeout=aiohttp.ClientTimeout(total=5)
                ) as response:
                    latency = (time.time() - start) * 1000

                    if response.status == 200:
                        status = HealthStatus.HEALTHY
                    elif response.status < 500:
                        status = HealthStatus.DEGRADED
                    else:
                        status = HealthStatus.UNHEALTHY

                    return ServiceHealth(
                        name=service,
                        status=status,
                        latency_ms=latency,
                        last_check=datetime.utcnow()
                    )
        except Exception as e:
            return ServiceHealth(
                name=service,
                status=HealthStatus.UNHEALTHY,
                latency_ms=-1,
                last_check=datetime.utcnow(),
                error=str(e)
            )

    async def check_all(self) -> Dict[str, ServiceHealth]:
        """Проверка всех сервисов."""
        tasks = [self.check_service(s) for s in self.services]
        results = await asyncio.gather(*tasks)

        for result in results:
            self.health_cache[result.name] = result

        return self.health_cache

    def get_overall_status(self) -> HealthStatus:
        """Общий статус gateway."""
        if not self.health_cache:
            return HealthStatus.UNHEALTHY

        statuses = [h.status for h in self.health_cache.values()]

        if all(s == HealthStatus.HEALTHY for s in statuses):
            return HealthStatus.HEALTHY
        elif any(s == HealthStatus.UNHEALTHY for s in statuses):
            return HealthStatus.DEGRADED
        else:
            return HealthStatus.DEGRADED


# Health endpoint
@app.get("/health")
async def health():
    """Liveness probe."""
    return {"status": "ok"}

@app.get("/health/ready")
async def readiness():
    """Readiness probe - проверяет зависимости."""
    checker = HealthChecker(["user-service", "order-service"])
    health = await checker.check_all()
    overall = checker.get_overall_status()

    status_code = 200 if overall != HealthStatus.UNHEALTHY else 503

    return JSONResponse(
        content={
            "status": overall.value,
            "services": {
                name: {
                    "status": h.status.value,
                    "latency_ms": h.latency_ms
                }
                for name, h in health.items()
            }
        },
        status_code=status_code
    )
```

---

## Типичные ошибки

### 1. Single Point of Failure

```
❌ Неправильно:
─────────────
┌────────┐     ┌─────────┐     ┌──────────┐
│ Client │────▶│ Gateway │────▶│ Services │
└────────┘     └─────────┘     └──────────┘
                    ↑
            Единственный экземпляр

✅ Правильно:
─────────────
                    ┌─────────┐
               ┌───▶│Gateway 1│────┐
┌────────┐     │    └─────────┘    │    ┌──────────┐
│ Client │────▶│LB                  │───▶│ Services │
└────────┘     │    ┌─────────┐    │    └──────────┘
               └───▶│Gateway 2│────┘
                    └─────────┘
```

### 2. Gateway как bottleneck

```python
# ❌ Неправильно: синхронная агрегация
def get_dashboard():
    user = requests.get("http://user-service/user/123").json()
    orders = requests.get("http://order-service/orders?user=123").json()
    stats = requests.get("http://stats-service/stats/123").json()
    # Общее время = user_time + orders_time + stats_time
    return {"user": user, "orders": orders, "stats": stats}

# ✅ Правильно: асинхронная агрегация
async def get_dashboard():
    async with asyncio.TaskGroup() as tg:
        user_task = tg.create_task(fetch("http://user-service/user/123"))
        orders_task = tg.create_task(fetch("http://order-service/orders?user=123"))
        stats_task = tg.create_task(fetch("http://stats-service/stats/123"))

    # Общее время = max(user_time, orders_time, stats_time)
    return {
        "user": user_task.result(),
        "orders": orders_task.result(),
        "stats": stats_task.result()
    }
```

### 3. Отсутствие таймаутов

```python
# ❌ Неправильно: нет таймаутов
async def forward_request(request):
    response = await aiohttp.get(upstream_url)  # Может висеть вечно
    return response

# ✅ Правильно: с таймаутами
async def forward_request(request):
    timeout = aiohttp.ClientTimeout(
        total=30,      # Общий таймаут
        connect=5,     # Таймаут подключения
        sock_read=10   # Таймаут чтения
    )

    try:
        async with aiohttp.ClientSession(timeout=timeout) as session:
            async with session.get(upstream_url) as response:
                return await response.json()
    except asyncio.TimeoutError:
        raise GatewayTimeoutError("Upstream service timeout")
```

### 4. Логирование секретов

```python
# ❌ Неправильно: логирование всех заголовков
logger.info(f"Request headers: {request.headers}")  # Включает Authorization!

# ✅ Правильно: фильтрация чувствительных данных
SENSITIVE_HEADERS = {"authorization", "x-api-key", "cookie", "x-auth-token"}

def safe_headers(headers: dict) -> dict:
    return {
        k: "***REDACTED***" if k.lower() in SENSITIVE_HEADERS else v
        for k, v in headers.items()
    }

logger.info(f"Request headers: {safe_headers(request.headers)}")
```

### 5. Игнорирование ошибок upstream

```python
# ❌ Неправильно: возврат ошибки upstream напрямую
async def forward_request(request):
    response = await call_upstream(request)
    return response  # Клиент видит внутреннюю ошибку

# ✅ Правильно: преобразование ошибок
async def forward_request(request):
    try:
        response = await call_upstream(request)

        if response.status >= 500:
            # Не раскрываем внутренние ошибки
            logger.error(f"Upstream error: {await response.text()}")
            raise HTTPException(
                status_code=502,
                detail="Service temporarily unavailable"
            )

        return response

    except aiohttp.ClientError as e:
        logger.error(f"Connection error: {e}")
        raise HTTPException(
            status_code=503,
            detail="Service unavailable"
        )
```

### 6. Неправильное кэширование

```python
# ❌ Неправильно: кэширование с Authorization
cache_key = f"{method}:{path}"  # Один кэш для всех пользователей!

# ✅ Правильно: включаем пользователя в ключ
def generate_cache_key(request: Request) -> str:
    user_id = request.headers.get("X-User-ID", "anonymous")
    return f"{request.method}:{request.url.path}:{user_id}"

# Или не кэшировать запросы с Authorization вообще
def should_cache(request: Request) -> bool:
    return (
        request.method in ["GET", "HEAD"] and
        "Authorization" not in request.headers and
        "Cookie" not in request.headers
    )
```

---

## Примеры конфигурации

### Полный пример с Kong

```yaml
# kong.yml
_format_version: "3.0"

# Сервисы
services:
  - name: user-service
    url: http://user-service:8080
    connect_timeout: 5000
    write_timeout: 60000
    read_timeout: 60000
    retries: 3

    routes:
      - name: user-api
        paths:
          - /api/v1/users
        methods:
          - GET
          - POST
          - PUT
          - DELETE
        strip_path: false

    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: redis
          redis_host: redis
          redis_port: 6379
          hide_client_headers: false

      - name: jwt
        config:
          key_claim_name: kid
          claims_to_verify:
            - exp

      - name: acl
        config:
          allow:
            - users-read
            - users-write

      - name: request-transformer
        config:
          add:
            headers:
              - "X-Gateway-Version:1.0"

      - name: response-transformer
        config:
          remove:
            headers:
              - X-Powered-By
              - Server

  - name: order-service
    url: http://order-service:8080

    routes:
      - name: order-api
        paths:
          - /api/v1/orders
        methods:
          - GET
          - POST
          - PUT
          - DELETE

    plugins:
      - name: rate-limiting
        config:
          minute: 50

      - name: jwt

      - name: request-termination
        enabled: false  # Для maintenance mode
        config:
          status_code: 503
          message: "Service under maintenance"

# Consumers (API клиенты)
consumers:
  - username: mobile-app
    custom_id: mobile-app-001

    keyauth_credentials:
      - key: mobile-app-api-key

    acls:
      - group: users-read
      - group: orders-read
      - group: orders-write

  - username: admin-app
    custom_id: admin-app-001

    jwt_secrets:
      - key: admin-key
        secret: admin-secret
        algorithm: HS256

    acls:
      - group: users-read
      - group: users-write
      - group: orders-read
      - group: orders-write
      - group: admin

# Глобальные плагины
plugins:
  - name: correlation-id
    config:
      header_name: X-Correlation-ID
      generator: uuid
      echo_downstream: true

  - name: prometheus
    config:
      per_consumer: true

  - name: file-log
    config:
      path: /var/log/kong/access.log
      reopen: true

  - name: ip-restriction
    enabled: false  # Включить при необходимости
    config:
      allow:
        - 10.0.0.0/8
        - 192.168.0.0/16
```

### Полный пример с NGINX

```nginx
# /etc/nginx/nginx.conf

user nginx;
worker_processes auto;
error_log /var/log/nginx/error.log warn;
pid /var/run/nginx.pid;

events {
    worker_connections 4096;
    use epoll;
    multi_accept on;
}

http {
    include /etc/nginx/mime.types;
    default_type application/octet-stream;

    # Logging
    log_format json_combined escape=json '{'
        '"time_local":"$time_local",'
        '"remote_addr":"$remote_addr",'
        '"request_method":"$request_method",'
        '"request_uri":"$request_uri",'
        '"status":$status,'
        '"body_bytes_sent":$body_bytes_sent,'
        '"request_time":$request_time,'
        '"upstream_response_time":"$upstream_response_time",'
        '"http_referer":"$http_referer",'
        '"http_user_agent":"$http_user_agent",'
        '"request_id":"$request_id"'
    '}';

    access_log /var/log/nginx/access.log json_combined;

    # Performance
    sendfile on;
    tcp_nopush on;
    tcp_nodelay on;
    keepalive_timeout 65;
    types_hash_max_size 2048;

    # Gzip
    gzip on;
    gzip_vary on;
    gzip_proxied any;
    gzip_comp_level 6;
    gzip_types text/plain text/css text/xml application/json application/javascript;

    # Rate limiting zones
    limit_req_zone $binary_remote_addr zone=ip_limit:10m rate=10r/s;
    limit_req_zone $http_x_api_key zone=api_key_limit:10m rate=100r/s;
    limit_conn_zone $binary_remote_addr zone=conn_limit:10m;

    # Cache
    proxy_cache_path /var/cache/nginx
        levels=1:2
        keys_zone=api_cache:100m
        max_size=10g
        inactive=60m
        use_temp_path=off;

    # Upstreams
    upstream user_service {
        zone user_service 64k;
        server user-service-1:8080 weight=5 max_fails=3 fail_timeout=30s;
        server user-service-2:8080 weight=5 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    upstream order_service {
        zone order_service 64k;
        server order-service-1:8080 max_fails=3 fail_timeout=30s;
        server order-service-2:8080 max_fails=3 fail_timeout=30s;
        keepalive 32;
    }

    upstream auth_service {
        zone auth_service 64k;
        server auth-service:8080 max_fails=3 fail_timeout=30s;
        keepalive 16;
    }

    # API Gateway server
    server {
        listen 443 ssl http2;
        server_name api.example.com;

        # SSL
        ssl_certificate /etc/nginx/ssl/server.crt;
        ssl_certificate_key /etc/nginx/ssl/server.key;
        ssl_session_timeout 1d;
        ssl_session_cache shared:SSL:50m;
        ssl_protocols TLSv1.2 TLSv1.3;
        ssl_ciphers ECDHE-ECDSA-AES128-GCM-SHA256:ECDHE-RSA-AES128-GCM-SHA256;
        ssl_prefer_server_ciphers off;

        # Security headers
        add_header X-Frame-Options "DENY" always;
        add_header X-Content-Type-Options "nosniff" always;
        add_header X-XSS-Protection "1; mode=block" always;
        add_header Strict-Transport-Security "max-age=31536000; includeSubDomains" always;

        # Rate limiting
        limit_req zone=ip_limit burst=20 nodelay;
        limit_conn conn_limit 50;

        # Request ID
        set $request_id $request_id;
        if ($http_x_request_id) {
            set $request_id $http_x_request_id;
        }

        # Health check
        location /health {
            access_log off;
            return 200 '{"status":"ok"}';
            add_header Content-Type application/json;
        }

        # Auth subrequest
        location = /_auth {
            internal;
            proxy_pass http://auth_service/validate;
            proxy_pass_request_body off;
            proxy_set_header Content-Length "";
            proxy_set_header X-Original-URI $request_uri;
            proxy_set_header X-Original-Method $request_method;
            proxy_set_header Authorization $http_authorization;
            proxy_cache_valid 200 1m;
        }

        # User service
        location /api/v1/users {
            auth_request /_auth;
            auth_request_set $user_id $upstream_http_x_user_id;
            auth_request_set $user_roles $upstream_http_x_user_roles;

            # Proxy headers
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Forwarded-Proto $scheme;
            proxy_set_header X-Request-ID $request_id;
            proxy_set_header X-User-ID $user_id;
            proxy_set_header X-User-Roles $user_roles;

            # Caching for GET
            proxy_cache api_cache;
            proxy_cache_methods GET HEAD;
            proxy_cache_valid 200 5m;
            proxy_cache_key "$scheme$request_method$host$request_uri$http_authorization";
            proxy_cache_use_stale error timeout http_500 http_502 http_503 http_504;
            add_header X-Cache-Status $upstream_cache_status;

            # Timeouts
            proxy_connect_timeout 5s;
            proxy_send_timeout 60s;
            proxy_read_timeout 60s;

            proxy_pass http://user_service;
        }

        # Order service
        location /api/v1/orders {
            auth_request /_auth;
            auth_request_set $user_id $upstream_http_x_user_id;

            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
            proxy_set_header X-Request-ID $request_id;
            proxy_set_header X-User-ID $user_id;

            # No caching for orders (dynamic data)
            proxy_no_cache 1;
            proxy_cache_bypass 1;

            proxy_pass http://order_service;
        }

        # Error handling
        error_page 401 = @error401;
        error_page 403 = @error403;
        error_page 404 = @error404;
        error_page 500 502 503 504 = @error5xx;

        location @error401 {
            default_type application/json;
            return 401 '{"error":"Unauthorized","message":"Authentication required"}';
        }

        location @error403 {
            default_type application/json;
            return 403 '{"error":"Forbidden","message":"Access denied"}';
        }

        location @error404 {
            default_type application/json;
            return 404 '{"error":"Not Found","message":"Resource not found"}';
        }

        location @error5xx {
            default_type application/json;
            return 502 '{"error":"Service Unavailable","message":"Please try again later"}';
        }
    }

    # Redirect HTTP to HTTPS
    server {
        listen 80;
        server_name api.example.com;
        return 301 https://$server_name$request_uri;
    }
}
```

---

## Резюме

**API Gateway** — критически важный компонент микросервисной архитектуры, который:

1. **Упрощает клиентов** — единая точка входа вместо множества сервисов
2. **Централизует сквозную функциональность** — auth, logging, rate limiting
3. **Защищает сервисы** — от перегрузки, атак, прямого доступа
4. **Обеспечивает наблюдаемость** — метрики, трейсинг, логирование

**Ключевые принципы:**
- Gateway должен быть stateless и масштабируемым
- Не размещайте бизнес-логику в gateway
- Используйте паттерны resilience (circuit breaker, retry, timeout)
- Мониторьте и защищайте gateway как критический компонент
- Выбирайте решение под ваши требования (managed vs self-hosted)

**Когда использовать:**
- Микросервисная архитектура
- Публичный API для партнёров/клиентов
- Необходимость централизованной безопасности
- Агрегация данных из нескольких сервисов
- BFF для разных типов клиентов
