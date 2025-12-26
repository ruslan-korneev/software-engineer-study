# API Gateways

## Введение

**API Gateway** — это единая точка входа для всех клиентских запросов к backend-сервисам. Он выступает посредником между клиентами и микросервисами, обеспечивая маршрутизацию, безопасность, мониторинг и другие cross-cutting concerns.

## Архитектурная диаграмма

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                              API Gateway Architecture                        │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│   ┌──────────┐  ┌──────────┐  ┌──────────┐                                  │
│   │  Mobile  │  │   Web    │  │  IoT     │                                  │
│   │   App    │  │   App    │  │ Devices  │                                  │
│   └────┬─────┘  └────┬─────┘  └────┬─────┘                                  │
│        │             │             │                                         │
│        └─────────────┼─────────────┘                                         │
│                      │                                                       │
│                      ▼                                                       │
│   ┌─────────────────────────────────────────────────────────────────────┐   │
│   │                         API Gateway                                  │   │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│   │  │  Auth   │ │  Rate   │ │ Request │ │ Load    │ │ Circuit │       │   │
│   │  │         │ │ Limiting│ │Transform│ │Balancing│ │ Breaker │       │   │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│   │  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────┐       │   │
│   │  │ Caching │ │ Logging │ │ Routing │ │ SSL     │ │Analytics│       │   │
│   │  │         │ │ Metrics │ │         │ │Terminal │ │         │       │   │
│   │  └─────────┘ └─────────┘ └─────────┘ └─────────┘ └─────────┘       │   │
│   └────────────────────────────┬────────────────────────────────────────┘   │
│                                │                                             │
│         ┌──────────────────────┼──────────────────────┐                     │
│         │                      │                      │                     │
│         ▼                      ▼                      ▼                     │
│   ┌───────────┐         ┌───────────┐          ┌───────────┐               │
│   │   User    │         │   Order   │          │  Payment  │               │
│   │  Service  │         │  Service  │          │  Service  │               │
│   └───────────┘         └───────────┘          └───────────┘               │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Основные функции API Gateway

### 1. Маршрутизация запросов (Request Routing)

```python
# Пример конфигурации маршрутизации (Kong/NGINX style)
routes = {
    '/api/v1/users/*': {
        'upstream': 'user-service:8080',
        'strip_prefix': True,
        'methods': ['GET', 'POST', 'PUT', 'DELETE']
    },
    '/api/v1/orders/*': {
        'upstream': 'order-service:8080',
        'strip_prefix': True,
        'methods': ['GET', 'POST']
    },
    '/api/v1/payments/*': {
        'upstream': 'payment-service:8080',
        'strip_prefix': True,
        'methods': ['POST']
    }
}

# Python реализация простого роутера
from fastapi import FastAPI, Request
import httpx

app = FastAPI()

ROUTES = {
    '/users': 'http://user-service:8080',
    '/orders': 'http://order-service:8080',
    '/payments': 'http://payment-service:8080'
}

@app.api_route('/{path:path}', methods=['GET', 'POST', 'PUT', 'DELETE'])
async def gateway(request: Request, path: str):
    """Простой API Gateway роутер."""

    # Определяем целевой сервис
    upstream = None
    for prefix, service in ROUTES.items():
        if f'/{path}'.startswith(prefix):
            upstream = service
            break

    if not upstream:
        return {'error': 'Route not found'}, 404

    # Проксируем запрос
    async with httpx.AsyncClient() as client:
        response = await client.request(
            method=request.method,
            url=f'{upstream}/{path}',
            headers=dict(request.headers),
            content=await request.body()
        )

    return Response(
        content=response.content,
        status_code=response.status_code,
        headers=dict(response.headers)
    )
```

### 2. Аутентификация и авторизация

```python
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.security import HTTPBearer, HTTPAuthorizationCredentials
import jwt

app = FastAPI()
security = HTTPBearer()

class AuthMiddleware:
    def __init__(self, jwt_secret: str):
        self.jwt_secret = jwt_secret

    async def verify_token(
        self,
        credentials: HTTPAuthorizationCredentials = Depends(security)
    ) -> dict:
        """Проверяет JWT токен."""
        token = credentials.credentials

        try:
            payload = jwt.decode(
                token,
                self.jwt_secret,
                algorithms=['HS256']
            )
            return payload
        except jwt.ExpiredSignatureError:
            raise HTTPException(status_code=401, detail='Token expired')
        except jwt.InvalidTokenError:
            raise HTTPException(status_code=401, detail='Invalid token')

    def require_role(self, required_role: str):
        """Проверяет наличие роли у пользователя."""
        async def role_checker(user: dict = Depends(self.verify_token)):
            if required_role not in user.get('roles', []):
                raise HTTPException(
                    status_code=403,
                    detail=f'Role {required_role} required'
                )
            return user
        return role_checker

auth = AuthMiddleware(jwt_secret='secret')

@app.get('/api/admin/users')
async def get_all_users(user: dict = Depends(auth.require_role('admin'))):
    """Доступно только администраторам."""
    return await user_service.get_all()

@app.get('/api/users/me')
async def get_current_user(user: dict = Depends(auth.verify_token)):
    """Доступно любому аутентифицированному пользователю."""
    return await user_service.get_by_id(user['user_id'])
```

### 3. Rate Limiting

```python
import time
from collections import defaultdict
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()

class RateLimiter:
    """Token Bucket алгоритм для rate limiting."""

    def __init__(self, rate: int, per: int):
        """
        Args:
            rate: количество запросов
            per: период в секундах
        """
        self.rate = rate
        self.per = per
        self.buckets = defaultdict(lambda: {'tokens': rate, 'last_update': time.time()})

    def is_allowed(self, key: str) -> tuple[bool, dict]:
        """Проверяет, разрешён ли запрос."""
        now = time.time()
        bucket = self.buckets[key]

        # Пополняем токены
        time_passed = now - bucket['last_update']
        bucket['tokens'] = min(
            self.rate,
            bucket['tokens'] + time_passed * (self.rate / self.per)
        )
        bucket['last_update'] = now

        if bucket['tokens'] >= 1:
            bucket['tokens'] -= 1
            return True, {
                'X-RateLimit-Limit': str(self.rate),
                'X-RateLimit-Remaining': str(int(bucket['tokens'])),
                'X-RateLimit-Reset': str(int(now + self.per))
            }

        return False, {
            'X-RateLimit-Limit': str(self.rate),
            'X-RateLimit-Remaining': '0',
            'X-RateLimit-Reset': str(int(now + self.per)),
            'Retry-After': str(int(self.per - time_passed))
        }

# Разные лимиты для разных тарифов
rate_limiters = {
    'free': RateLimiter(rate=100, per=3600),      # 100 req/hour
    'basic': RateLimiter(rate=1000, per=3600),    # 1000 req/hour
    'premium': RateLimiter(rate=10000, per=3600)  # 10000 req/hour
}

@app.middleware('http')
async def rate_limit_middleware(request: Request, call_next):
    """Rate limiting middleware."""

    # Определяем ключ для rate limiting
    client_ip = request.client.host
    user_tier = request.headers.get('X-User-Tier', 'free')

    limiter = rate_limiters.get(user_tier, rate_limiters['free'])
    key = f'{user_tier}:{client_ip}'

    allowed, headers = limiter.is_allowed(key)

    if not allowed:
        response = JSONResponse(
            status_code=429,
            content={'error': 'Rate limit exceeded'}
        )
        for k, v in headers.items():
            response.headers[k] = v
        return response

    response = await call_next(request)
    for k, v in headers.items():
        response.headers[k] = v

    return response
```

### 4. Request/Response Transformation

```python
from fastapi import FastAPI, Request
from pydantic import BaseModel

app = FastAPI()

class TransformationMiddleware:
    """Трансформация запросов и ответов."""

    async def transform_request(self, request: Request) -> dict:
        """Модифицирует входящий запрос."""
        body = await request.json()

        # Преобразуем camelCase в snake_case
        transformed = self._camel_to_snake(body)

        # Добавляем метаданные
        transformed['_request_id'] = request.headers.get('X-Request-ID')
        transformed['_timestamp'] = datetime.utcnow().isoformat()

        return transformed

    def transform_response(self, data: dict, api_version: str) -> dict:
        """Модифицирует исходящий ответ."""

        if api_version == 'v1':
            # Для старых клиентов — старый формат
            return {
                'data': data,
                'success': True
            }
        else:
            # Для новых клиентов — новый формат
            return {
                'result': data,
                'meta': {
                    'version': api_version,
                    'timestamp': datetime.utcnow().isoformat()
                }
            }

    def _camel_to_snake(self, data):
        """Конвертирует ключи из camelCase в snake_case."""
        if isinstance(data, dict):
            return {
                self._convert_key(k): self._camel_to_snake(v)
                for k, v in data.items()
            }
        elif isinstance(data, list):
            return [self._camel_to_snake(item) for item in data]
        return data

    def _convert_key(self, key: str) -> str:
        import re
        return re.sub(r'(?<!^)(?=[A-Z])', '_', key).lower()

# Aggregation — объединение нескольких API вызовов
@app.get('/api/v1/dashboard')
async def get_dashboard(user_id: str):
    """
    Агрегирует данные из нескольких сервисов
    в один ответ для клиента.
    """
    async with httpx.AsyncClient() as client:
        # Параллельные запросы к сервисам
        user_task = client.get(f'{USER_SERVICE}/users/{user_id}')
        orders_task = client.get(f'{ORDER_SERVICE}/users/{user_id}/orders?limit=5')
        notifications_task = client.get(f'{NOTIFICATION_SERVICE}/users/{user_id}/unread')

        user_resp, orders_resp, notifications_resp = await asyncio.gather(
            user_task, orders_task, notifications_task
        )

    # Агрегируем результаты
    return {
        'user': user_resp.json(),
        'recent_orders': orders_resp.json(),
        'unread_notifications': notifications_resp.json()['count']
    }
```

### 5. Circuit Breaker

```python
import time
from enum import Enum
from dataclasses import dataclass
from typing import Callable, Any

class CircuitState(Enum):
    CLOSED = 'closed'       # Нормальная работа
    OPEN = 'open'           # Ошибка, запросы блокируются
    HALF_OPEN = 'half_open' # Тестовый режим

@dataclass
class CircuitBreakerConfig:
    failure_threshold: int = 5      # Порог ошибок
    success_threshold: int = 2      # Порог успехов для recovery
    timeout: int = 30               # Время в OPEN состоянии

class CircuitBreaker:
    """Реализация паттерна Circuit Breaker."""

    def __init__(self, name: str, config: CircuitBreakerConfig = None):
        self.name = name
        self.config = config or CircuitBreakerConfig()
        self.state = CircuitState.CLOSED
        self.failure_count = 0
        self.success_count = 0
        self.last_failure_time = None

    async def call(self, func: Callable, *args, **kwargs) -> Any:
        """Выполняет вызов с защитой circuit breaker."""

        if self.state == CircuitState.OPEN:
            if self._should_try_reset():
                self.state = CircuitState.HALF_OPEN
            else:
                raise CircuitBreakerOpenError(
                    f'Circuit {self.name} is OPEN'
                )

        try:
            result = await func(*args, **kwargs)
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _on_success(self):
        if self.state == CircuitState.HALF_OPEN:
            self.success_count += 1
            if self.success_count >= self.config.success_threshold:
                self.state = CircuitState.CLOSED
                self.failure_count = 0
                self.success_count = 0
        else:
            self.failure_count = 0

    def _on_failure(self):
        self.failure_count += 1
        self.last_failure_time = time.time()

        if self.failure_count >= self.config.failure_threshold:
            self.state = CircuitState.OPEN

    def _should_try_reset(self) -> bool:
        if self.last_failure_time is None:
            return True
        return time.time() - self.last_failure_time >= self.config.timeout

# Использование в Gateway
class ServiceProxy:
    def __init__(self):
        self.circuits = {}

    def get_circuit(self, service_name: str) -> CircuitBreaker:
        if service_name not in self.circuits:
            self.circuits[service_name] = CircuitBreaker(service_name)
        return self.circuits[service_name]

    async def call_service(self, service_name: str, url: str, **kwargs):
        circuit = self.get_circuit(service_name)

        async def make_request():
            async with httpx.AsyncClient(timeout=10) as client:
                return await client.request(**kwargs, url=url)

        try:
            return await circuit.call(make_request)
        except CircuitBreakerOpenError:
            # Fallback логика
            return await self.get_cached_response(url)
```

### 6. Caching

```python
import hashlib
import json
from functools import wraps

class CacheMiddleware:
    """Кэширование ответов API."""

    def __init__(self, redis_client):
        self.redis = redis_client

    def cache_key(self, request: Request) -> str:
        """Генерирует ключ кэша."""
        key_parts = [
            request.method,
            str(request.url),
            request.headers.get('Authorization', ''),
            request.headers.get('Accept-Language', '')
        ]
        key_string = '|'.join(key_parts)
        return f'cache:{hashlib.md5(key_string.encode()).hexdigest()}'

    async def get_cached(self, key: str) -> dict | None:
        """Получает закэшированный ответ."""
        cached = await self.redis.get(key)
        if cached:
            return json.loads(cached)
        return None

    async def set_cached(self, key: str, response: dict, ttl: int = 300):
        """Сохраняет ответ в кэш."""
        await self.redis.setex(key, ttl, json.dumps(response))

    def is_cacheable(self, request: Request) -> bool:
        """Определяет, можно ли кэшировать запрос."""
        return (
            request.method == 'GET' and
            'no-cache' not in request.headers.get('Cache-Control', '')
        )

@app.middleware('http')
async def cache_middleware(request: Request, call_next):
    cache = CacheMiddleware(redis)

    if cache.is_cacheable(request):
        key = cache.cache_key(request)
        cached = await cache.get_cached(key)

        if cached:
            response = JSONResponse(content=cached['body'])
            response.headers['X-Cache'] = 'HIT'
            return response

    response = await call_next(request)

    if cache.is_cacheable(request) and response.status_code == 200:
        body = b''
        async for chunk in response.body_iterator:
            body += chunk

        await cache.set_cached(key, {
            'body': json.loads(body),
            'status': response.status_code
        })

        response = Response(
            content=body,
            status_code=response.status_code,
            headers=dict(response.headers)
        )
        response.headers['X-Cache'] = 'MISS'

    return response
```

---

## Паттерны API Gateway

### 1. Single Entry Point

```
┌──────────────────────────────────────────────────────────┐
│                   Single API Gateway                      │
│                                                           │
│   Mobile ─┐                                               │
│           │     ┌─────────────┐     ┌─────────────┐      │
│   Web ────┼────>│ API Gateway │────>│  Services   │      │
│           │     └─────────────┘     └─────────────┘      │
│   IoT ────┘                                               │
│                                                           │
└──────────────────────────────────────────────────────────┘
```

**Плюсы:** Простота, единая точка мониторинга
**Минусы:** Single point of failure, масштабирование

### 2. Backend for Frontend (BFF)

```
┌──────────────────────────────────────────────────────────────────┐
│                     Backend for Frontend                          │
│                                                                   │
│   ┌────────┐     ┌─────────────┐                                 │
│   │ Mobile │────>│ Mobile BFF  │──┐                              │
│   └────────┘     └─────────────┘  │                              │
│                                    │     ┌─────────────────┐     │
│   ┌────────┐     ┌─────────────┐  ├────>│    Services     │     │
│   │  Web   │────>│  Web BFF    │──┤     └─────────────────┘     │
│   └────────┘     └─────────────┘  │                              │
│                                    │                              │
│   ┌────────┐     ┌─────────────┐  │                              │
│   │  IoT   │────>│  IoT BFF    │──┘                              │
│   └────────┘     └─────────────┘                                 │
│                                                                   │
└──────────────────────────────────────────────────────────────────┘
```

```python
# Mobile BFF — оптимизирован для мобильных устройств
@mobile_bff.get('/api/products/{product_id}')
async def get_product_mobile(product_id: str):
    """
    Для мобильных: минимум данных,
    оптимизированные изображения.
    """
    product = await product_service.get(product_id)

    return {
        'id': product.id,
        'name': product.name,
        'price': product.price,
        'image': product.image_url_small,  # Маленькое изображение
        'rating': product.rating
        # Без описания и деталей — экономим трафик
    }

# Web BFF — полная информация
@web_bff.get('/api/products/{product_id}')
async def get_product_web(product_id: str):
    """
    Для веба: полные данные,
    включая рекомендации и отзывы.
    """
    product = await product_service.get(product_id)
    reviews = await review_service.get_for_product(product_id, limit=10)
    recommendations = await recommendation_service.get(product_id)

    return {
        'id': product.id,
        'name': product.name,
        'description': product.description,  # Полное описание
        'price': product.price,
        'images': product.all_images,  # Все изображения
        'specifications': product.specs,
        'reviews': reviews,
        'recommendations': recommendations
    }
```

### 3. Gateway Routing Pattern

```python
# Routing based on request properties

class SmartRouter:
    def __init__(self):
        self.routes = []

    def add_route(self, matcher, handler):
        self.routes.append((matcher, handler))

    async def route(self, request: Request):
        for matcher, handler in self.routes:
            if matcher(request):
                return await handler(request)
        return None

# Routing по версии API
def version_matcher(version: str):
    def match(request: Request):
        return request.headers.get('API-Version') == version
    return match

# Routing по A/B тесту
def ab_test_matcher(experiment: str, variant: str):
    def match(request: Request):
        user_id = request.headers.get('X-User-ID')
        user_variant = get_user_variant(user_id, experiment)
        return user_variant == variant
    return match

router = SmartRouter()
router.add_route(version_matcher('v2'), handle_v2_request)
router.add_route(version_matcher('v1'), handle_v1_request)
router.add_route(ab_test_matcher('new-checkout', 'B'), handle_new_checkout)
```

---

## Популярные API Gateway решения

### 1. Kong

```yaml
# kong.yml - декларативная конфигурация
_format_version: "2.1"

services:
  - name: user-service
    url: http://user-service:8080
    routes:
      - name: user-route
        paths:
          - /api/v1/users
    plugins:
      - name: rate-limiting
        config:
          minute: 100
          policy: local
      - name: jwt
        config:
          secret_is_base64: false
      - name: cors
        config:
          origins:
            - "*"
          methods:
            - GET
            - POST

  - name: order-service
    url: http://order-service:8080
    routes:
      - name: order-route
        paths:
          - /api/v1/orders
    plugins:
      - name: request-transformer
        config:
          add:
            headers:
              - "X-Service-Name:api-gateway"
```

### 2. AWS API Gateway

```yaml
# serverless.yml
service: my-api

provider:
  name: aws
  runtime: python3.9

functions:
  getUsers:
    handler: handlers.get_users
    events:
      - http:
          path: /users
          method: get
          cors: true
          authorizer:
            type: COGNITO_USER_POOLS
            authorizerId:
              Ref: ApiGatewayAuthorizer
          request:
            parameters:
              querystrings:
                limit: false
                offset: false

  createOrder:
    handler: handlers.create_order
    events:
      - http:
          path: /orders
          method: post
          cors: true

resources:
  Resources:
    ApiGatewayAuthorizer:
      Type: AWS::ApiGateway::Authorizer
      Properties:
        Name: CognitoAuthorizer
        Type: COGNITO_USER_POOLS
        IdentitySource: method.request.header.Authorization
        RestApiId:
          Ref: ApiGatewayRestApi
        ProviderARNs:
          - !GetAtt UserPool.Arn
```

### 3. NGINX as API Gateway

```nginx
# nginx.conf

upstream user_service {
    server user-service:8080;
    keepalive 32;
}

upstream order_service {
    server order-service:8080;
    keepalive 32;
}

# Rate limiting zone
limit_req_zone $binary_remote_addr zone=api_limit:10m rate=10r/s;

server {
    listen 80;
    server_name api.example.com;

    # SSL termination
    listen 443 ssl;
    ssl_certificate /etc/nginx/ssl/cert.pem;
    ssl_certificate_key /etc/nginx/ssl/key.pem;

    # CORS headers
    add_header 'Access-Control-Allow-Origin' '*' always;
    add_header 'Access-Control-Allow-Methods' 'GET, POST, PUT, DELETE' always;

    # Rate limiting
    limit_req zone=api_limit burst=20 nodelay;

    # User service
    location /api/v1/users {
        proxy_pass http://user_service;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Request-ID $request_id;

        # Circuit breaker
        proxy_next_upstream error timeout http_502 http_503;
        proxy_next_upstream_tries 3;
    }

    # Order service
    location /api/v1/orders {
        # Auth check
        auth_request /auth;

        proxy_pass http://order_service;
        proxy_http_version 1.1;
        proxy_set_header Connection "";
    }

    # Internal auth endpoint
    location = /auth {
        internal;
        proxy_pass http://auth-service:8080/verify;
        proxy_pass_request_body off;
        proxy_set_header Content-Length "";
        proxy_set_header X-Original-URI $request_uri;
    }

    # Health check
    location /health {
        access_log off;
        return 200 "OK";
    }
}
```

---

## Best Practices

### 1. Observability

```python
import time
import uuid
from opentelemetry import trace
from opentelemetry.trace import Status, StatusCode

tracer = trace.get_tracer(__name__)

@app.middleware('http')
async def observability_middleware(request: Request, call_next):
    """Middleware для трассировки и метрик."""

    # Генерируем или извлекаем request ID
    request_id = request.headers.get('X-Request-ID', str(uuid.uuid4()))

    # Создаём span для трассировки
    with tracer.start_as_current_span(
        name=f'{request.method} {request.url.path}',
        attributes={
            'http.method': request.method,
            'http.url': str(request.url),
            'http.request_id': request_id
        }
    ) as span:
        start_time = time.time()

        try:
            response = await call_next(request)

            # Записываем метрики
            duration = time.time() - start_time
            REQUEST_DURATION.labels(
                method=request.method,
                path=request.url.path,
                status=response.status_code
            ).observe(duration)

            span.set_status(Status(StatusCode.OK))
            response.headers['X-Request-ID'] = request_id

            return response

        except Exception as e:
            span.set_status(Status(StatusCode.ERROR, str(e)))
            span.record_exception(e)
            raise
```

### 2. Graceful Degradation

```python
class GracefulDegradation:
    """Graceful degradation при сбоях сервисов."""

    def __init__(self, cache, circuit_breaker):
        self.cache = cache
        self.cb = circuit_breaker

    async def get_with_fallback(
        self,
        primary_call,
        fallback_call=None,
        cache_key=None
    ):
        """Попытка получить данные с fallback."""

        try:
            # Пробуем основной источник
            result = await self.cb.call(primary_call)

            # Кэшируем успешный результат
            if cache_key:
                await self.cache.set(cache_key, result, ttl=3600)

            return result

        except CircuitBreakerOpenError:
            # Circuit breaker открыт — используем fallback
            pass
        except Exception as e:
            logger.error(f'Primary call failed: {e}')

        # Fallback 1: Кэш
        if cache_key:
            cached = await self.cache.get(cache_key)
            if cached:
                logger.info(f'Serving from cache: {cache_key}')
                return cached

        # Fallback 2: Альтернативный источник
        if fallback_call:
            try:
                return await fallback_call()
            except Exception as e:
                logger.error(f'Fallback call failed: {e}')

        # Fallback 3: Дефолтный ответ
        return {'status': 'degraded', 'data': None}
```

### 3. Request Validation

```python
from pydantic import BaseModel, validator
from fastapi import HTTPException

class CreateOrderRequest(BaseModel):
    user_id: str
    items: list[dict]
    shipping_address: dict

    @validator('items')
    def validate_items(cls, v):
        if not v:
            raise ValueError('Order must have at least one item')
        if len(v) > 100:
            raise ValueError('Order cannot have more than 100 items')
        return v

    @validator('shipping_address')
    def validate_address(cls, v):
        required_fields = ['street', 'city', 'country', 'postal_code']
        for field in required_fields:
            if field not in v:
                raise ValueError(f'Missing required field: {field}')
        return v

@app.post('/api/v1/orders')
async def create_order(request: CreateOrderRequest):
    """Валидация происходит автоматически на уровне Gateway."""
    return await order_service.create(request.dict())
```

---

## Типичные ошибки

### 1. Gateway как Single Point of Failure

```python
# ❌ Плохо: один инстанс Gateway
# gateway-deployment.yaml
# replicas: 1  # SPOF!

# ✅ Хорошо: несколько инстансов с load balancer
# replicas: 3
# + Load Balancer впереди
# + Health checks
# + Auto-scaling
```

### 2. Слишком много логики в Gateway

```python
# ❌ Плохо: бизнес-логика в Gateway
@app.post('/api/orders')
async def create_order(data: dict):
    # Валидация бизнес-правил — это должно быть в сервисе!
    if data['total'] > 10000:
        await notify_manager(data)
    if data['user'].is_vip:
        data['discount'] = 0.1
    ...

# ✅ Хорошо: Gateway только проксирует
@app.post('/api/orders')
async def create_order(data: dict):
    # Только cross-cutting concerns
    await validate_auth(request)
    await check_rate_limit(request)
    return await proxy_to_service('order-service', data)
```

### 3. Отсутствие timeout и retry

```python
# ❌ Плохо: бесконечное ожидание
response = await client.get(service_url)

# ✅ Хорошо: с timeout и retry
response = await client.get(
    service_url,
    timeout=httpx.Timeout(10.0, connect=5.0),
    # + Circuit Breaker
    # + Retry с exponential backoff
)
```

---

## Сравнение решений

| Критерий | Kong | AWS API GW | NGINX | Custom |
|----------|------|------------|-------|--------|
| Сложность | Средняя | Низкая | Низкая | Высокая |
| Масштабируемость | Высокая | Высокая | Средняя | Зависит |
| Стоимость | Open Source/Enterprise | Pay-per-use | Free/Plus | Разработка |
| Plugins | 100+ | Limited | Lua | Custom |
| Observability | Встроена | CloudWatch | Отдельно | Custom |

---

## Заключение

API Gateway — критически важный компонент современной микросервисной архитектуры. Ключевые функции:

1. **Единая точка входа** — упрощает клиентское взаимодействие
2. **Безопасность** — централизованная аутентификация и авторизация
3. **Rate Limiting** — защита от перегрузки
4. **Observability** — мониторинг и трассировка
5. **Resilience** — Circuit Breaker и Graceful Degradation

Выбирайте решение исходя из требований: готовое (Kong, AWS) для быстрого старта, или custom для полного контроля.
