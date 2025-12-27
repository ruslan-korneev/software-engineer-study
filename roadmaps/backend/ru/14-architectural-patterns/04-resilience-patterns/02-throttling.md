# Throttling (Rate Limiting)

## Определение

**Throttling** (также известен как **Rate Limiting**) — это паттерн управления нагрузкой, который ограничивает количество запросов к сервису за определённый период времени. Используется для защиты от перегрузки, DDoS-атак, справедливого распределения ресурсов и монетизации API.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Throttling Overview                          │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   Requests    ┌──────────────┐    Allowed    ┌──────────┐       │
│   ──────────► │   Rate       │ ─────────────► │ Service  │       │
│               │   Limiter    │               └──────────┘       │
│               └──────┬───────┘                                   │
│                      │                                           │
│                      │ Rejected (429 Too Many Requests)          │
│                      ▼                                           │
│               ┌──────────────┐                                   │
│               │ Error/Queue  │                                   │
│               └──────────────┘                                   │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

**Throttling** и **Rate Limiting** часто используются как синонимы, но есть нюанс:
- **Rate Limiting** — жёсткое ограничение (reject сверх лимита)
- **Throttling** — может включать замедление (queue, delay)

## Ключевые характеристики

### Основные алгоритмы Rate Limiting

#### 1. Fixed Window (Фиксированное окно)

```
Время:    |-------- 1 минута --------|-------- 1 минута --------|
Лимит:                100                         100
Запросы:  ████████████████████████   ████████████████████████
          (100 разрешено)             (сброс, снова 100)

Проблема: "burst at boundary"
          |......50 запросов|50 запросов......|
          Можно сделать 100 запросов за 2 секунды на границе окон
```

#### 2. Sliding Window Log (Скользящее окно с логом)

```python
class SlidingWindowLog:
    """
    Хранит timestamp каждого запроса.
    Точный, но требует больше памяти.
    """

    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window = window_seconds
        self.requests: Dict[str, List[float]] = defaultdict(list)

    def is_allowed(self, client_id: str) -> bool:
        now = time.time()
        window_start = now - self.window

        # Удаляем старые запросы
        self.requests[client_id] = [
            ts for ts in self.requests[client_id]
            if ts > window_start
        ]

        if len(self.requests[client_id]) < self.limit:
            self.requests[client_id].append(now)
            return True
        return False
```

#### 3. Sliding Window Counter (Скользящее окно со счётчиком)

```
Текущее окно: 60% пройдено
Предыдущее окно: 80 запросов
Текущее окно: 30 запросов

Взвешенный подсчёт: 80 * 0.4 + 30 = 62 запроса
Лимит: 100 → Разрешено
```

```python
class SlidingWindowCounter:
    """Компромисс между точностью и эффективностью."""

    def __init__(self, limit: int, window_seconds: int):
        self.limit = limit
        self.window = window_seconds
        self.counters: Dict[str, Dict[int, int]] = defaultdict(dict)

    def is_allowed(self, client_id: str) -> bool:
        now = time.time()
        current_window = int(now // self.window)
        previous_window = current_window - 1
        window_progress = (now % self.window) / self.window

        current_count = self.counters[client_id].get(current_window, 0)
        previous_count = self.counters[client_id].get(previous_window, 0)

        # Взвешенный подсчёт
        weighted_count = previous_count * (1 - window_progress) + current_count

        if weighted_count < self.limit:
            self.counters[client_id][current_window] = current_count + 1
            # Очистка старых окон
            self._cleanup(client_id, current_window)
            return True
        return False

    def _cleanup(self, client_id: str, current_window: int):
        old_windows = [w for w in self.counters[client_id] if w < current_window - 1]
        for w in old_windows:
            del self.counters[client_id][w]
```

#### 4. Token Bucket (Корзина токенов)

```
┌────────────────────────────────────────────────────────┐
│                    Token Bucket                         │
├────────────────────────────────────────────────────────┤
│                                                         │
│   Tokens added at rate R     ┌─────────────────┐       │
│          ▼                   │  ○ ○ ○ ○ ○ ○ ○  │       │
│         ╔═╗                  │  Bucket (max B) │       │
│         ║●║ ──────────────►  │  tokens         │       │
│         ╚═╝                  └────────┬────────┘       │
│                                       │                 │
│                                       ▼                 │
│                              Request consumes token     │
│                                                         │
│   Позволяет burst до B, затем ограничение по R         │
└────────────────────────────────────────────────────────┘
```

```python
class TokenBucket:
    """
    Token Bucket algorithm.

    Параметры:
    - capacity: максимальное количество токенов (размер burst)
    - refill_rate: токенов в секунду
    """

    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.buckets: Dict[str, Tuple[float, float]] = {}  # (tokens, last_refill)
        self._lock = threading.Lock()

    def is_allowed(self, client_id: str, tokens: int = 1) -> bool:
        with self._lock:
            now = time.time()

            if client_id not in self.buckets:
                self.buckets[client_id] = (self.capacity, now)

            current_tokens, last_refill = self.buckets[client_id]

            # Добавляем токены с момента последнего обновления
            elapsed = now - last_refill
            current_tokens = min(
                self.capacity,
                current_tokens + elapsed * self.refill_rate
            )

            if current_tokens >= tokens:
                self.buckets[client_id] = (current_tokens - tokens, now)
                return True

            self.buckets[client_id] = (current_tokens, now)
            return False

    def get_retry_after(self, client_id: str, tokens: int = 1) -> float:
        """Возвращает время до появления нужного количества токенов."""
        if client_id not in self.buckets:
            return 0

        current_tokens, _ = self.buckets[client_id]
        if current_tokens >= tokens:
            return 0

        tokens_needed = tokens - current_tokens
        return tokens_needed / self.refill_rate
```

#### 5. Leaky Bucket (Дырявое ведро)

```
┌────────────────────────────────────────────────────────┐
│                    Leaky Bucket                         │
├────────────────────────────────────────────────────────┤
│                                                         │
│   Requests (variable rate)                             │
│         │ │ │││ │ │  │ ││                              │
│         ▼ ▼ ▼▼▼ ▼ ▼  ▼ ▼▼                              │
│        ╔═══════════════════╗                           │
│        ║   Queue (Bucket)  ║ ← overflow = reject       │
│        ╚═══════╤═══════════╝                           │
│                │                                        │
│                ▼ (constant rate)                        │
│           ● ● ● ● ●                                    │
│                                                         │
│   Выходная скорость постоянна (smoothing)              │
└────────────────────────────────────────────────────────┘
```

### Сравнение алгоритмов

| Алгоритм | Точность | Память | Burst | Сложность |
|----------|----------|--------|-------|-----------|
| Fixed Window | Низкая | O(n) | Да (2x) | Низкая |
| Sliding Log | Высокая | O(n*m) | Нет | Средняя |
| Sliding Counter | Средняя | O(n) | Нет | Средняя |
| Token Bucket | Высокая | O(n) | Да (контролируемый) | Низкая |
| Leaky Bucket | Высокая | O(n*q) | Нет (queue) | Средняя |

## Когда использовать

### Идеальные сценарии

1. **API Gateway / Public API**
   - Защита от злоупотреблений
   - Монетизация (разные тарифы)
   - Fair usage policy

2. **Микросервисы**
   - Защита downstream сервисов
   - Предотвращение каскадных сбоев
   - Распределение нагрузки

3. **Критические ресурсы**
   - База данных (connection pool)
   - Внешние API с лимитами
   - Дорогие операции (ML inference)

4. **Multi-tenant системы**
   - Изоляция между клиентами
   - SLA enforcement
   - Защита от "шумных соседей"

### Типы rate limiting

```python
# По клиенту (IP, user ID, API key)
limiter.check("user:123")

# По endpoint
limiter.check("endpoint:/api/search")

# Комбинированный
limiter.check("user:123:endpoint:/api/search")

# Глобальный
limiter.check("global:critical_operation")

# По ресурсу
limiter.check("resource:database_writes")
```

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Защита от перегрузки** | Предотвращает падение системы |
| **Справедливость** | Равномерное распределение ресурсов |
| **Предсказуемость** | Гарантированный SLA |
| **Безопасность** | Защита от DDoS и брутфорса |
| **Монетизация** | Разные уровни доступа |
| **Контроль затрат** | Ограничение использования внешних API |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Latency** | Дополнительная проверка на каждый запрос |
| **Сложность в распределённых системах** | Синхронизация между нодами |
| **False positives** | Блокировка легитимных пользователей |
| **Обход** | Распределённые атаки с разных IP |
| **Настройка** | Подбор правильных лимитов |

## Примеры реализации

### Python: Rate Limiter с Redis

```python
import redis
import time
from typing import Tuple, Optional
from dataclasses import dataclass


@dataclass
class RateLimitResult:
    allowed: bool
    remaining: int
    reset_at: float
    retry_after: Optional[float] = None


class RedisRateLimiter:
    """
    Rate Limiter на базе Redis с алгоритмом Token Bucket.
    Подходит для распределённых систем.
    """

    def __init__(
        self,
        redis_client: redis.Redis,
        prefix: str = "ratelimit"
    ):
        self.redis = redis_client
        self.prefix = prefix

        # Lua script для атомарной операции
        self.lua_script = self.redis.register_script("""
            local key = KEYS[1]
            local capacity = tonumber(ARGV[1])
            local refill_rate = tonumber(ARGV[2])
            local now = tonumber(ARGV[3])
            local tokens_requested = tonumber(ARGV[4])

            local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
            local tokens = tonumber(bucket[1])
            local last_refill = tonumber(bucket[2])

            if tokens == nil then
                tokens = capacity
                last_refill = now
            end

            -- Refill tokens
            local elapsed = now - last_refill
            tokens = math.min(capacity, tokens + elapsed * refill_rate)

            local allowed = 0
            if tokens >= tokens_requested then
                tokens = tokens - tokens_requested
                allowed = 1
            end

            -- Save state
            redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
            redis.call('EXPIRE', key, math.ceil(capacity / refill_rate) * 2)

            return {allowed, math.floor(tokens), capacity}
        """)

    def check(
        self,
        key: str,
        capacity: int = 100,
        refill_rate: float = 10.0,
        tokens: int = 1
    ) -> RateLimitResult:
        """
        Проверяет rate limit.

        Args:
            key: Идентификатор клиента/ресурса
            capacity: Максимальное количество токенов
            refill_rate: Токенов в секунду
            tokens: Количество токенов для запроса

        Returns:
            RateLimitResult с результатом проверки
        """
        redis_key = f"{self.prefix}:{key}"
        now = time.time()

        result = self.lua_script(
            keys=[redis_key],
            args=[capacity, refill_rate, now, tokens]
        )

        allowed, remaining, cap = result
        reset_at = now + (capacity - remaining) / refill_rate

        return RateLimitResult(
            allowed=bool(allowed),
            remaining=remaining,
            reset_at=reset_at,
            retry_after=None if allowed else (tokens - remaining) / refill_rate
        )


class RateLimitMiddleware:
    """ASGI Middleware для rate limiting."""

    def __init__(
        self,
        app,
        limiter: RedisRateLimiter,
        key_func=None,
        limits: dict = None
    ):
        self.app = app
        self.limiter = limiter
        self.key_func = key_func or self._default_key_func
        self.limits = limits or {"default": (100, 10.0)}  # capacity, rate

    def _default_key_func(self, scope) -> str:
        """Извлекает ключ для rate limiting."""
        # По IP адресу
        client = scope.get("client", ("unknown", 0))
        return f"ip:{client[0]}"

    async def __call__(self, scope, receive, send):
        if scope["type"] != "http":
            return await self.app(scope, receive, send)

        key = self.key_func(scope)
        path = scope.get("path", "/")

        # Определяем лимиты для endpoint
        capacity, rate = self._get_limits(path)

        result = self.limiter.check(key, capacity, rate)

        if not result.allowed:
            response = self._rate_limit_response(result)
            await send({
                "type": "http.response.start",
                "status": 429,
                "headers": response["headers"]
            })
            await send({
                "type": "http.response.body",
                "body": response["body"]
            })
            return

        # Добавляем заголовки rate limit
        async def send_with_headers(message):
            if message["type"] == "http.response.start":
                headers = list(message.get("headers", []))
                headers.extend([
                    (b"X-RateLimit-Limit", str(capacity).encode()),
                    (b"X-RateLimit-Remaining", str(result.remaining).encode()),
                    (b"X-RateLimit-Reset", str(int(result.reset_at)).encode()),
                ])
                message["headers"] = headers
            await send(message)

        await self.app(scope, receive, send_with_headers)

    def _get_limits(self, path: str) -> Tuple[int, float]:
        """Получает лимиты для конкретного endpoint."""
        for pattern, limits in self.limits.items():
            if pattern != "default" and path.startswith(pattern):
                return limits
        return self.limits.get("default", (100, 10.0))

    def _rate_limit_response(self, result: RateLimitResult) -> dict:
        return {
            "headers": [
                (b"Content-Type", b"application/json"),
                (b"Retry-After", str(int(result.retry_after or 1)).encode()),
                (b"X-RateLimit-Remaining", b"0"),
            ],
            "body": b'{"error": "Too Many Requests", "retry_after": %d}' %
                    int(result.retry_after or 1)
        }
```

### FastAPI пример

```python
from fastapi import FastAPI, Request, HTTPException, Depends
from fastapi.responses import JSONResponse
import redis.asyncio as redis


app = FastAPI()

# Rate limit конфигурация
RATE_LIMITS = {
    "/api/search": (20, 2.0),      # 20 запросов, 2/сек refill
    "/api/expensive": (5, 0.5),    # 5 запросов, 0.5/сек refill
    "default": (100, 10.0)
}


@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_ip = request.client.host
    path = request.url.path

    # Получаем лимиты
    capacity, rate = RATE_LIMITS.get(path, RATE_LIMITS["default"])

    # Проверяем rate limit
    result = await check_rate_limit(client_ip, path, capacity, rate)

    if not result["allowed"]:
        return JSONResponse(
            status_code=429,
            content={
                "error": "Rate limit exceeded",
                "retry_after": result["retry_after"]
            },
            headers={
                "Retry-After": str(result["retry_after"]),
                "X-RateLimit-Remaining": "0"
            }
        )

    response = await call_next(request)

    # Добавляем заголовки
    response.headers["X-RateLimit-Limit"] = str(capacity)
    response.headers["X-RateLimit-Remaining"] = str(result["remaining"])

    return response


# Decorator-based rate limiting
from functools import wraps

def rate_limit(capacity: int, rate: float):
    """Декоратор для rate limiting отдельных endpoints."""
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, request: Request, **kwargs):
            client_key = f"{request.client.host}:{func.__name__}"
            result = await check_rate_limit(client_key, capacity, rate)

            if not result["allowed"]:
                raise HTTPException(
                    status_code=429,
                    detail="Rate limit exceeded",
                    headers={"Retry-After": str(result["retry_after"])}
                )

            return await func(*args, request=request, **kwargs)
        return wrapper
    return decorator


@app.get("/api/limited")
@rate_limit(capacity=10, rate=1.0)
async def limited_endpoint(request: Request):
    return {"message": "This endpoint is rate limited"}
```

### Go: Rate Limiter

```go
package ratelimit

import (
    "context"
    "fmt"
    "time"

    "github.com/go-redis/redis/v8"
)

type RateLimiter struct {
    redis    *redis.Client
    prefix   string
    capacity int64
    rate     float64
}

func NewRateLimiter(redis *redis.Client, prefix string, capacity int64, rate float64) *RateLimiter {
    return &RateLimiter{
        redis:    redis,
        prefix:   prefix,
        capacity: capacity,
        rate:     rate,
    }
}

func (rl *RateLimiter) Allow(ctx context.Context, key string) (bool, int64, error) {
    fullKey := fmt.Sprintf("%s:%s", rl.prefix, key)
    now := float64(time.Now().UnixNano()) / 1e9

    script := `
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        local data = redis.call('HMGET', key, 'tokens', 'last')
        local tokens = tonumber(data[1]) or capacity
        local last = tonumber(data[2]) or now

        local elapsed = now - last
        tokens = math.min(capacity, tokens + elapsed * rate)

        local allowed = 0
        if tokens >= 1 then
            tokens = tokens - 1
            allowed = 1
        end

        redis.call('HMSET', key, 'tokens', tokens, 'last', now)
        redis.call('EXPIRE', key, math.ceil(capacity / rate) * 2)

        return {allowed, math.floor(tokens)}
    `

    result, err := rl.redis.Eval(ctx, script, []string{fullKey}, rl.capacity, rl.rate, now).Result()
    if err != nil {
        return false, 0, err
    }

    values := result.([]interface{})
    allowed := values[0].(int64) == 1
    remaining := values[1].(int64)

    return allowed, remaining, nil
}

// Middleware для HTTP
func RateLimitMiddleware(limiter *RateLimiter) func(http.Handler) http.Handler {
    return func(next http.Handler) http.Handler {
        return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
            ip := getClientIP(r)

            allowed, remaining, err := limiter.Allow(r.Context(), ip)
            if err != nil {
                http.Error(w, "Internal Server Error", 500)
                return
            }

            w.Header().Set("X-RateLimit-Remaining", fmt.Sprintf("%d", remaining))

            if !allowed {
                w.Header().Set("Retry-After", "1")
                http.Error(w, "Too Many Requests", 429)
                return
            }

            next.ServeHTTP(w, r)
        })
    }
}
```

## Best Practices и антипаттерны

### Best Practices

#### 1. Информативные заголовки ответа

```python
headers = {
    "X-RateLimit-Limit": "100",           # Лимит
    "X-RateLimit-Remaining": "42",        # Осталось
    "X-RateLimit-Reset": "1640000000",    # Timestamp сброса
    "Retry-After": "30",                  # Секунд до retry
}
```

#### 2. Многоуровневое ограничение

```python
limits = {
    # Глобальный лимит по IP
    "ip": RateLimit(requests=1000, period=3600),

    # По пользователю (аутентифицированный)
    "user": RateLimit(requests=5000, period=3600),

    # По API ключу
    "api_key": RateLimit(requests=10000, period=3600),

    # По endpoint (дополнительно)
    "endpoint:/api/search": RateLimit(requests=100, period=60),
    "endpoint:/api/export": RateLimit(requests=10, period=3600),
}
```

#### 3. Graceful degradation

```python
async def handle_request(request):
    result = rate_limiter.check(request.client_id)

    if result.allowed:
        return await full_response()

    # Вместо полного отказа — сокращённый ответ
    if result.remaining > -10:  # Небольшое превышение
        return await limited_response()

    # Полный отказ только при значительном превышении
    return error_response(429, retry_after=result.retry_after)
```

#### 4. Разные стратегии для разных клиентов

```python
def get_rate_limit(client: Client) -> RateLimit:
    if client.is_premium:
        return RateLimit(10000, 3600)
    elif client.is_authenticated:
        return RateLimit(1000, 3600)
    else:
        return RateLimit(100, 3600)
```

### Антипаттерны

#### 1. Rate limiting только по IP

```python
# Плохо: легко обойти через прокси/VPN
key = request.client.ip

# Лучше: комбинация факторов
key = f"{request.client.ip}:{request.user_id}:{request.fingerprint}"
```

#### 2. Слишком агрессивные лимиты

```python
# Плохо: блокирует легитимных пользователей
limiter = RateLimiter(capacity=10, rate=0.1)  # 10 запросов, 1 в 10 секунд

# Лучше: начинать с мягких лимитов, ужесточать по необходимости
limiter = RateLimiter(capacity=100, rate=10)
```

#### 3. Отсутствие мониторинга

```python
# Плохо: не знаем, кто блокируется
if not allowed:
    return error_429()

# Лучше: собираем метрики
if not allowed:
    metrics.increment("rate_limit.blocked", tags={
        "client_type": client.type,
        "endpoint": request.path
    })
    return error_429()
```

## Связанные паттерны

### Throttling + Circuit Breaker

```python
class ResilientService:
    def __init__(self):
        self.rate_limiter = RateLimiter()
        self.circuit_breaker = CircuitBreaker()

    async def call(self, request):
        # Сначала проверяем rate limit (защита от перегрузки)
        if not self.rate_limiter.allow(request.client_id):
            return error_429()

        # Затем проверяем circuit breaker (защита от каскадных сбоев)
        try:
            return await self.circuit_breaker.call(self._do_call, request)
        except CircuitBreakerOpen:
            return error_503()
```

### Throttling + Backpressure

```
┌──────────┐     ┌──────────────┐     ┌───────┐     ┌─────────┐
│ Clients  │ ──► │ Rate Limiter │ ──► │ Queue │ ──► │ Workers │
└──────────┘     └──────────────┘     └───────┘     └─────────┘
                                           │
                                    Backpressure signal
                                    (queue full → reject)
```

### Связанные паттерны

| Паттерн | Связь с Throttling |
|---------|---------------------|
| **Circuit Breaker** | CB защищает downstream, throttling — upstream |
| **Bulkhead** | Изоляция ресурсов + throttling для каждого |
| **Backpressure** | Throttling как реакция на backpressure |
| **Load Shedding** | Более агрессивное отбрасывание при перегрузке |
| **Retry with Backoff** | Клиент реагирует на 429 |

## Ресурсы для изучения

### Документация
- [Stripe: Rate Limiting](https://stripe.com/docs/rate-limits)
- [GitHub API Rate Limiting](https://docs.github.com/en/rest/rate-limit)
- [Google Cloud: Rate Limiting Strategies](https://cloud.google.com/architecture/rate-limiting-strategies-techniques)

### Библиотеки
- **Python**: `slowapi`, `flask-limiter`, `limits`
- **Go**: `golang.org/x/time/rate`, `uber-go/ratelimit`
- **Java**: `Bucket4j`, `Resilience4j RateLimiter`
- **Node.js**: `rate-limiter-flexible`, `express-rate-limit`

### Инструменты
- **Kong**: API Gateway с rate limiting
- **Envoy**: Service mesh с rate limiting
- **Redis**: Для распределённого rate limiting

### Статьи
- [Figma: Rate Limiting System Design](https://www.figma.com/blog/rate-limiting/)
- [Cloudflare: How Rate Limiting Works](https://blog.cloudflare.com/rate-limiting/)
- [System Design: Rate Limiter](https://systemdesign.one/rate-limiter-system-design/)
