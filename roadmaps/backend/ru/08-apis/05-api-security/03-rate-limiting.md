# Rate Limiting (Ограничение частоты запросов)

## Что такое Rate Limiting?

**Rate Limiting** — механизм ограничения количества запросов, которые клиент может сделать за определённый период времени. Защищает API от:

- **DDoS-атак** — перегрузка сервера запросами
- **Brute Force** — перебор паролей и токенов
- **Abuse** — злоупотребление API
- **Resource exhaustion** — исчерпание ресурсов
- **Scraping** — массовая выгрузка данных

---

## Алгоритмы Rate Limiting

### 1. Fixed Window (Фиксированное окно)

Подсчёт запросов в фиксированных временных интервалах.

```
Окно: 00:00 - 01:00 → 100 запросов
Окно: 01:00 - 02:00 → 100 запросов
```

**Проблема:** Burst на границе окон — можно сделать 200 запросов за секунду.

```
00:59:59 → 100 запросов ✓
01:00:00 → 100 запросов ✓
# Итого: 200 запросов за 2 секунды
```

**Реализация:**

```python
import time
from collections import defaultdict

class FixedWindowRateLimiter:
    def __init__(self, max_requests: int, window_size: int):
        self.max_requests = max_requests
        self.window_size = window_size  # в секундах
        self.requests: dict[str, dict] = defaultdict(lambda: {"count": 0, "window": 0})

    def is_allowed(self, client_id: str) -> bool:
        current_window = int(time.time() // self.window_size)
        data = self.requests[client_id]

        if data["window"] != current_window:
            data["window"] = current_window
            data["count"] = 0

        if data["count"] >= self.max_requests:
            return False

        data["count"] += 1
        return True

# Использование
limiter = FixedWindowRateLimiter(max_requests=100, window_size=60)
if limiter.is_allowed(client_ip):
    process_request()
else:
    return Response(status_code=429)
```

---

### 2. Sliding Window Log (Скользящее окно с логом)

Хранит временные метки всех запросов и считает запросы за последние N секунд.

**Преимущество:** Точный подсчёт, нет проблемы burst.

**Недостаток:** Высокое потребление памяти.

```python
import time
from collections import defaultdict

class SlidingWindowLogRateLimiter:
    def __init__(self, max_requests: int, window_size: int):
        self.max_requests = max_requests
        self.window_size = window_size
        self.request_logs: dict[str, list[float]] = defaultdict(list)

    def is_allowed(self, client_id: str) -> bool:
        current_time = time.time()
        window_start = current_time - self.window_size

        # Удаляем старые запросы
        self.request_logs[client_id] = [
            timestamp for timestamp in self.request_logs[client_id]
            if timestamp > window_start
        ]

        if len(self.request_logs[client_id]) >= self.max_requests:
            return False

        self.request_logs[client_id].append(current_time)
        return True
```

---

### 3. Sliding Window Counter (Скользящее окно со счётчиком)

Комбинация Fixed Window и Sliding Window — хранит счётчики для текущего и предыдущего окна.

```python
import time
from collections import defaultdict

class SlidingWindowCounterRateLimiter:
    def __init__(self, max_requests: int, window_size: int):
        self.max_requests = max_requests
        self.window_size = window_size
        self.windows: dict[str, dict] = defaultdict(
            lambda: {"prev_count": 0, "curr_count": 0, "curr_window": 0}
        )

    def is_allowed(self, client_id: str) -> bool:
        current_time = time.time()
        current_window = int(current_time // self.window_size)
        window_progress = (current_time % self.window_size) / self.window_size

        data = self.windows[client_id]

        # Сдвигаем окна если нужно
        if data["curr_window"] < current_window:
            data["prev_count"] = data["curr_count"] if data["curr_window"] == current_window - 1 else 0
            data["curr_count"] = 0
            data["curr_window"] = current_window

        # Взвешенный подсчёт запросов
        weighted_count = (
            data["prev_count"] * (1 - window_progress) +
            data["curr_count"]
        )

        if weighted_count >= self.max_requests:
            return False

        data["curr_count"] += 1
        return True
```

---

### 4. Token Bucket (Корзина токенов)

Токены добавляются с постоянной скоростью. Каждый запрос забирает токен.

**Преимущество:** Позволяет контролируемые burst'ы.

```
Ёмкость: 10 токенов
Скорость пополнения: 1 токен/сек

[|||||||||| ]  ← 10 токенов
[|||||||    ]  ← 3 запроса, осталось 7
[|||||||||  ]  ← +2 токена за 2 секунды
```

```python
import time

class TokenBucketRateLimiter:
    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate  # токенов в секунду
        self.buckets: dict[str, dict] = {}

    def is_allowed(self, client_id: str, tokens: int = 1) -> bool:
        current_time = time.time()

        if client_id not in self.buckets:
            self.buckets[client_id] = {
                "tokens": self.capacity,
                "last_refill": current_time
            }

        bucket = self.buckets[client_id]

        # Пополняем токены
        time_passed = current_time - bucket["last_refill"]
        bucket["tokens"] = min(
            self.capacity,
            bucket["tokens"] + time_passed * self.refill_rate
        )
        bucket["last_refill"] = current_time

        # Проверяем и забираем токены
        if bucket["tokens"] >= tokens:
            bucket["tokens"] -= tokens
            return True

        return False

# Использование: 100 запросов в минуту, burst до 10
limiter = TokenBucketRateLimiter(capacity=10, refill_rate=100/60)
```

---

### 5. Leaky Bucket (Дырявое ведро)

Запросы обрабатываются с постоянной скоростью. Избыточные ставятся в очередь.

```
Входящие запросы → [Очередь] → Выходящие запросы (фиксированная скорость)
```

**Преимущество:** Сглаживает трафик.

**Недостаток:** Добавляет задержку.

```python
import time
import asyncio
from collections import deque

class LeakyBucketRateLimiter:
    def __init__(self, capacity: int, leak_rate: float):
        self.capacity = capacity
        self.leak_rate = leak_rate  # запросов в секунду
        self.queue: deque = deque()
        self.last_leak = time.time()

    async def acquire(self, client_id: str) -> bool:
        current_time = time.time()

        # "Протекание" — удаляем обработанные запросы
        leaked = int((current_time - self.last_leak) * self.leak_rate)
        for _ in range(min(leaked, len(self.queue))):
            self.queue.popleft()
        self.last_leak = current_time

        # Проверяем место в очереди
        if len(self.queue) >= self.capacity:
            return False

        # Добавляем в очередь и ждём своей очереди
        position = len(self.queue)
        self.queue.append(client_id)

        wait_time = position / self.leak_rate
        await asyncio.sleep(wait_time)

        return True
```

---

## Реализация в FastAPI

### Базовая реализация с Redis

```python
import redis
from fastapi import FastAPI, Request, HTTPException, Depends
from functools import wraps

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379, db=0)

class RateLimiter:
    def __init__(self, requests: int, window: int):
        self.requests = requests
        self.window = window

    async def check(self, key: str) -> tuple[bool, dict]:
        pipe = redis_client.pipeline()

        # Увеличиваем счётчик
        pipe.incr(key)
        pipe.ttl(key)

        results = pipe.execute()
        count = results[0]
        ttl = results[1]

        # Устанавливаем TTL при первом запросе
        if ttl == -1:
            redis_client.expire(key, self.window)
            ttl = self.window

        remaining = max(0, self.requests - count)
        reset_time = ttl

        headers = {
            "X-RateLimit-Limit": str(self.requests),
            "X-RateLimit-Remaining": str(remaining),
            "X-RateLimit-Reset": str(reset_time),
        }

        return count <= self.requests, headers

def rate_limit(requests: int = 100, window: int = 60):
    """Декоратор rate limiting"""
    limiter = RateLimiter(requests=requests, window=window)

    def decorator(func):
        @wraps(func)
        async def wrapper(request: Request, *args, **kwargs):
            # Ключ: IP + путь
            client_ip = request.client.host
            key = f"rate_limit:{client_ip}:{request.url.path}"

            allowed, headers = await limiter.check(key)

            if not allowed:
                raise HTTPException(
                    status_code=429,
                    detail="Too many requests",
                    headers=headers
                )

            response = await func(request, *args, **kwargs)

            # Добавляем заголовки к ответу
            for header, value in headers.items():
                response.headers[header] = value

            return response
        return wrapper
    return decorator

# Использование
@app.get("/api/search")
@rate_limit(requests=10, window=60)  # 10 запросов в минуту
async def search(request: Request, q: str):
    return {"results": []}
```

### Middleware подход

```python
from starlette.middleware.base import BaseHTTPMiddleware
from starlette.responses import JSONResponse

class RateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, default_limit: int = 100, window: int = 60):
        super().__init__(app)
        self.default_limit = default_limit
        self.window = window
        self.redis = redis.Redis()

    async def dispatch(self, request: Request, call_next):
        client_ip = request.client.host
        key = f"rate_limit:{client_ip}"

        # Sliding window counter в Redis
        current = self.redis.incr(key)
        if current == 1:
            self.redis.expire(key, self.window)

        if current > self.default_limit:
            return JSONResponse(
                status_code=429,
                content={"detail": "Rate limit exceeded"},
                headers={
                    "Retry-After": str(self.redis.ttl(key)),
                    "X-RateLimit-Limit": str(self.default_limit),
                    "X-RateLimit-Remaining": "0",
                }
            )

        response = await call_next(request)
        response.headers["X-RateLimit-Limit"] = str(self.default_limit)
        response.headers["X-RateLimit-Remaining"] = str(self.default_limit - current)
        return response

app.add_middleware(RateLimitMiddleware, default_limit=100, window=60)
```

---

## Стратегии идентификации клиента

### 1. По IP адресу

```python
def get_client_identifier(request: Request) -> str:
    # Учитываем прокси
    forwarded = request.headers.get("X-Forwarded-For")
    if forwarded:
        return forwarded.split(",")[0].strip()
    return request.client.host
```

**Проблемы:**
- NAT — много пользователей за одним IP
- Прокси — можно обойти, меняя IP
- IPv6 — огромное пространство адресов

### 2. По API ключу / токену

```python
def get_client_identifier(request: Request) -> str:
    # Аутентифицированные пользователи
    api_key = request.headers.get("X-API-Key")
    if api_key:
        return f"api_key:{api_key}"

    # Для неаутентифицированных — по IP
    return f"ip:{request.client.host}"
```

### 3. По User ID

```python
async def get_client_identifier(
    request: Request,
    current_user: User = Depends(get_current_user_optional)
) -> str:
    if current_user:
        return f"user:{current_user.id}"
    return f"ip:{request.client.host}"
```

### 4. Комбинированный подход

```python
def get_rate_limit_key(request: Request, user: User | None) -> str:
    """
    Иерархия лимитов:
    1. Аутентифицированный пользователь — по user_id
    2. С API ключом — по ключу
    3. Анонимный — по IP (строже)
    """
    if user:
        return f"user:{user.id}"

    api_key = request.headers.get("X-API-Key")
    if api_key:
        return f"apikey:{api_key}"

    return f"anon:{request.client.host}"
```

---

## Различные лимиты для разных случаев

```python
from dataclasses import dataclass
from enum import Enum

class RateLimitTier(Enum):
    ANONYMOUS = "anonymous"
    FREE = "free"
    PRO = "pro"
    ENTERPRISE = "enterprise"

@dataclass
class RateLimitConfig:
    requests_per_minute: int
    requests_per_hour: int
    requests_per_day: int
    burst_limit: int

RATE_LIMITS: dict[RateLimitTier, RateLimitConfig] = {
    RateLimitTier.ANONYMOUS: RateLimitConfig(
        requests_per_minute=10,
        requests_per_hour=100,
        requests_per_day=500,
        burst_limit=5
    ),
    RateLimitTier.FREE: RateLimitConfig(
        requests_per_minute=60,
        requests_per_hour=1000,
        requests_per_day=10000,
        burst_limit=20
    ),
    RateLimitTier.PRO: RateLimitConfig(
        requests_per_minute=300,
        requests_per_hour=10000,
        requests_per_day=100000,
        burst_limit=50
    ),
    RateLimitTier.ENTERPRISE: RateLimitConfig(
        requests_per_minute=1000,
        requests_per_hour=50000,
        requests_per_day=500000,
        burst_limit=100
    ),
}

class TieredRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client

    async def check(self, user_id: str, tier: RateLimitTier) -> tuple[bool, dict]:
        config = RATE_LIMITS[tier]

        # Проверяем все окна
        checks = [
            ("minute", config.requests_per_minute, 60),
            ("hour", config.requests_per_hour, 3600),
            ("day", config.requests_per_day, 86400),
        ]

        for window_name, limit, window_size in checks:
            key = f"ratelimit:{user_id}:{window_name}"
            count = await self.redis.incr(key)

            if count == 1:
                await self.redis.expire(key, window_size)

            if count > limit:
                return False, {"window": window_name, "limit": limit}

        return True, {}
```

---

## Лимиты на разные эндпоинты

```python
# Разные лимиты для разных типов операций
ENDPOINT_LIMITS = {
    "/api/auth/login": {"requests": 5, "window": 60},        # 5/мин — защита от brute force
    "/api/search": {"requests": 30, "window": 60},           # 30/мин — тяжёлые запросы
    "/api/users": {"requests": 100, "window": 60},           # 100/мин — обычные запросы
    "/api/export": {"requests": 5, "window": 3600},          # 5/час — очень тяжёлые
    "default": {"requests": 60, "window": 60},
}

class EndpointRateLimiter:
    def get_limit_for_path(self, path: str) -> dict:
        for pattern, limit in ENDPOINT_LIMITS.items():
            if pattern != "default" and path.startswith(pattern):
                return limit
        return ENDPOINT_LIMITS["default"]

    async def check(self, client_id: str, path: str) -> bool:
        limit = self.get_limit_for_path(path)
        key = f"ratelimit:{client_id}:{path}"

        count = await self.redis.incr(key)
        if count == 1:
            await self.redis.expire(key, limit["window"])

        return count <= limit["requests"]
```

---

## HTTP заголовки Rate Limiting

### Стандартные заголовки (RFC 6585 / draft-ietf-httpapi-ratelimit-headers)

```http
HTTP/1.1 200 OK
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 95
X-RateLimit-Reset: 1640000000
RateLimit-Policy: 100;w=60

HTTP/1.1 429 Too Many Requests
Retry-After: 30
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1640000000
```

**Реализация:**

```python
from fastapi import Response
from datetime import datetime

def add_rate_limit_headers(
    response: Response,
    limit: int,
    remaining: int,
    reset_timestamp: int
):
    response.headers["X-RateLimit-Limit"] = str(limit)
    response.headers["X-RateLimit-Remaining"] = str(remaining)
    response.headers["X-RateLimit-Reset"] = str(reset_timestamp)

    # RFC draft формат
    response.headers["RateLimit-Policy"] = f"{limit};w=60"

    if remaining == 0:
        response.headers["Retry-After"] = str(reset_timestamp - int(datetime.utcnow().timestamp()))
```

---

## Распределённый Rate Limiting

При нескольких серверах нужно синхронизировать счётчики.

### Redis Lua Script (атомарная операция)

```python
RATE_LIMIT_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local current = redis.call('INCR', key)

if current == 1 then
    redis.call('EXPIRE', key, window)
end

if current > limit then
    return {0, redis.call('TTL', key), current}
else
    return {1, redis.call('TTL', key), current}
end
"""

class DistributedRateLimiter:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.script = self.redis.register_script(RATE_LIMIT_SCRIPT)

    async def check(self, key: str, limit: int, window: int) -> tuple[bool, int, int]:
        result = await self.script(keys=[key], args=[limit, window])
        allowed = bool(result[0])
        ttl = result[1]
        current = result[2]
        return allowed, ttl, current
```

### Sliding Window в Redis

```python
SLIDING_WINDOW_SCRIPT = """
local key = KEYS[1]
local limit = tonumber(ARGV[1])
local window = tonumber(ARGV[2])
local now = tonumber(ARGV[3])

-- Удаляем старые записи
redis.call('ZREMRANGEBYSCORE', key, '-inf', now - window)

-- Считаем текущие запросы
local count = redis.call('ZCARD', key)

if count < limit then
    -- Добавляем новый запрос
    redis.call('ZADD', key, now, now .. ':' .. math.random())
    redis.call('EXPIRE', key, window)
    return {1, limit - count - 1}
else
    return {0, 0}
end
"""
```

---

## Graceful Degradation

Что делать, если Redis недоступен?

```python
class ResilientRateLimiter:
    def __init__(self, redis_client, fallback_limit: int = 1000):
        self.redis = redis_client
        self.fallback_limit = fallback_limit
        self.local_counters: dict[str, int] = {}

    async def check(self, key: str, limit: int, window: int) -> bool:
        try:
            return await self._redis_check(key, limit, window)
        except redis.RedisError:
            # Fallback на локальный счётчик с более высоким лимитом
            return self._local_check(key, self.fallback_limit)

    def _local_check(self, key: str, limit: int) -> bool:
        count = self.local_counters.get(key, 0) + 1
        self.local_counters[key] = count
        return count <= limit
```

---

## Best Practices

1. **Возвращайте понятные ошибки** — код 429 + заголовки + Retry-After
2. **Логируйте превышения лимитов** — для анализа и настройки
3. **Используйте разные лимиты** — для разных пользователей и эндпоинтов
4. **Документируйте лимиты** — в OpenAPI и документации
5. **Мониторьте** — алерты на массовые 429
6. **Graceful degradation** — не ломайте всё при отказе Redis
7. **Тестируйте** — проверяйте поведение под нагрузкой

---

## Типичные ошибки

| Ошибка | Последствие | Решение |
|--------|-------------|---------|
| Нет rate limiting | DDoS, brute force | Реализовать базовый лимит |
| Только по IP | Обход через прокси | Добавить лимит по user/API key |
| Одинаковый лимит везде | Критичные эндпоинты не защищены | Разные лимиты для разных путей |
| Нет заголовков | Клиент не знает о лимитах | Добавить X-RateLimit-* |
| Fixed window | Burst на границе | Sliding window или Token bucket |

---

## Чек-лист

- [ ] Реализован rate limiting для всех публичных эндпоинтов
- [ ] Строгие лимиты на аутентификацию (login, password reset)
- [ ] Разные лимиты для анонимных и аутентифицированных пользователей
- [ ] Возвращается код 429 с заголовком Retry-After
- [ ] Добавлены заголовки X-RateLimit-*
- [ ] Лимиты задокументированы
- [ ] Настроен мониторинг превышений
- [ ] Есть fallback при отказе Redis
