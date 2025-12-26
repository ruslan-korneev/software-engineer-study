# Rate Limiting (Ограничение частоты запросов)

## Введение

**Rate Limiting** — это техника контроля количества запросов, которые клиент может отправить к API за определённый период времени. Это критически важный механизм защиты API от:

- **DDoS-атак** — злонамеренных попыток перегрузить сервис
- **Злоупотреблений** — чрезмерного использования ресурсов одним клиентом
- **Случайных ошибок** — бесконечных циклов в коде клиента
- **Справедливого распределения** — гарантии доступности для всех пользователей

## Алгоритмы Rate Limiting

### 1. Fixed Window (Фиксированное окно)

Подсчёт запросов в фиксированных временных интервалах.

```
Окно 1 (10:00 - 10:01): ████████░░ 8/10 запросов
Окно 2 (10:01 - 10:02): ██████████ 10/10 (лимит!)
Окно 3 (10:02 - 10:03): ███░░░░░░░ 3/10 запросов
```

**Проблема:** На границе окон можно отправить 2x запросов.

```
10:00:59 - 10 запросов (конец окна 1)
10:01:00 - 10 запросов (начало окна 2)
= 20 запросов за 2 секунды
```

#### Реализация (Python + Redis):

```python
import redis
import time
from fastapi import FastAPI, HTTPException, Request

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379, db=0)

def fixed_window_rate_limit(
    key: str,
    limit: int,
    window_seconds: int
) -> tuple[bool, int]:
    """
    Fixed Window Rate Limiter.
    Возвращает (allowed, remaining).
    """
    current_window = int(time.time() // window_seconds)
    redis_key = f"rate_limit:{key}:{current_window}"

    current_count = redis_client.incr(redis_key)

    if current_count == 1:
        redis_client.expire(redis_key, window_seconds)

    if current_count > limit:
        return False, 0

    return True, limit - current_count

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    # Идентификатор клиента (IP или API key)
    client_id = request.client.host

    allowed, remaining = fixed_window_rate_limit(
        key=client_id,
        limit=100,  # 100 запросов
        window_seconds=60  # в минуту
    )

    if not allowed:
        raise HTTPException(
            status_code=429,
            detail="Rate limit exceeded. Please try again later."
        )

    response = await call_next(request)
    response.headers["X-RateLimit-Limit"] = "100"
    response.headers["X-RateLimit-Remaining"] = str(remaining)
    return response
```

### 2. Sliding Window Log (Скользящее окно с логом)

Хранит timestamp каждого запроса и удаляет устаревшие.

```python
def sliding_window_log_rate_limit(
    key: str,
    limit: int,
    window_seconds: int
) -> tuple[bool, int]:
    """
    Sliding Window Log Rate Limiter.
    """
    now = time.time()
    window_start = now - window_seconds
    redis_key = f"rate_limit:log:{key}"

    pipe = redis_client.pipeline()

    # Удаляем устаревшие записи
    pipe.zremrangebyscore(redis_key, 0, window_start)

    # Добавляем текущий запрос
    pipe.zadd(redis_key, {str(now): now})

    # Получаем количество запросов в окне
    pipe.zcard(redis_key)

    # Устанавливаем TTL
    pipe.expire(redis_key, window_seconds)

    results = pipe.execute()
    count = results[2]

    if count > limit:
        # Удаляем только что добавленный запрос
        redis_client.zrem(redis_key, str(now))
        return False, 0

    return True, limit - count
```

**Преимущества:** Точный подсчёт, нет проблемы границ окон.
**Недостатки:** Высокое потребление памяти.

### 3. Sliding Window Counter (Скользящее окно со счётчиком)

Компромисс между Fixed Window и Sliding Window Log.

```python
def sliding_window_counter_rate_limit(
    key: str,
    limit: int,
    window_seconds: int
) -> tuple[bool, int]:
    """
    Sliding Window Counter — взвешенный подсчёт.
    """
    now = time.time()
    current_window = int(now // window_seconds)
    previous_window = current_window - 1

    current_key = f"rate_limit:{key}:{current_window}"
    previous_key = f"rate_limit:{key}:{previous_window}"

    # Получаем счётчики обоих окон
    pipe = redis_client.pipeline()
    pipe.get(current_key)
    pipe.get(previous_key)
    results = pipe.execute()

    current_count = int(results[0] or 0)
    previous_count = int(results[1] or 0)

    # Вычисляем вес предыдущего окна
    elapsed = now % window_seconds
    weight = 1 - (elapsed / window_seconds)

    # Взвешенный подсчёт
    weighted_count = current_count + (previous_count * weight)

    if weighted_count >= limit:
        return False, 0

    # Инкрементируем текущее окно
    pipe = redis_client.pipeline()
    pipe.incr(current_key)
    pipe.expire(current_key, window_seconds * 2)
    pipe.execute()

    return True, int(limit - weighted_count - 1)
```

### 4. Token Bucket (Корзина с токенами)

Токены накапливаются с постоянной скоростью. Каждый запрос потребляет токен.

```
Корзина: [●●●●●●●●●●] 10/10 токенов

Запрос → [●●●●●●●●●○] 9/10
Запрос → [●●●●●●●●○○] 8/10
...
Пополнение (каждую секунду): +1 токен
```

```python
def token_bucket_rate_limit(
    key: str,
    capacity: int,  # Максимум токенов
    refill_rate: float,  # Токенов в секунду
    tokens_per_request: int = 1
) -> tuple[bool, int]:
    """
    Token Bucket Rate Limiter.
    """
    now = time.time()
    redis_key = f"rate_limit:bucket:{key}"

    # Получаем текущее состояние
    data = redis_client.hgetall(redis_key)

    if data:
        tokens = float(data.get(b'tokens', capacity))
        last_update = float(data.get(b'last_update', now))
    else:
        tokens = capacity
        last_update = now

    # Пополняем токены
    elapsed = now - last_update
    tokens = min(capacity, tokens + elapsed * refill_rate)

    if tokens < tokens_per_request:
        return False, 0

    # Потребляем токены
    tokens -= tokens_per_request

    # Сохраняем состояние
    redis_client.hset(redis_key, mapping={
        'tokens': tokens,
        'last_update': now
    })
    redis_client.expire(redis_key, int(capacity / refill_rate) + 1)

    return True, int(tokens)
```

**Преимущества:**
- Позволяет всплески (bursts) трафика
- Сглаживает нагрузку
- Гибкая настройка

### 5. Leaky Bucket (Дырявое ведро)

Запросы добавляются в очередь и обрабатываются с постоянной скоростью.

```
Входящие запросы → [Очередь: ████████░░] → Обработка (N/сек)
                              ↓ overflow
                          Отклонение
```

```python
import asyncio
from collections import deque

class LeakyBucket:
    def __init__(self, capacity: int, leak_rate: float):
        self.capacity = capacity
        self.leak_rate = leak_rate  # запросов в секунду
        self.queue = deque()
        self.last_leak = time.time()

    def _leak(self):
        """Обрабатывает запросы из очереди."""
        now = time.time()
        elapsed = now - self.last_leak
        leaks = int(elapsed * self.leak_rate)

        for _ in range(min(leaks, len(self.queue))):
            self.queue.popleft()

        if leaks > 0:
            self.last_leak = now

    def allow(self) -> bool:
        """Проверяет, можно ли добавить запрос."""
        self._leak()

        if len(self.queue) < self.capacity:
            self.queue.append(time.time())
            return True
        return False

# Использование
bucket = LeakyBucket(capacity=10, leak_rate=1.0)

if bucket.allow():
    # Обработать запрос
    pass
else:
    # 429 Too Many Requests
    pass
```

## HTTP заголовки Rate Limit

### Стандартные заголовки (RFC 6585 + draft-ietf-httpapi-ratelimit-headers):

```http
HTTP/1.1 200 OK
RateLimit-Limit: 100
RateLimit-Remaining: 95
RateLimit-Reset: 1640995200
```

### Расширенные заголовки:

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 100
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 2024-01-15T10:30:00Z
X-RateLimit-Policy: "100;w=60"
```

### Реализация заголовков:

```python
from fastapi import FastAPI, Request, Response
from datetime import datetime, timezone

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    client_id = request.client.host
    limit = 100
    window_seconds = 60

    allowed, remaining = sliding_window_counter_rate_limit(
        key=client_id,
        limit=limit,
        window_seconds=window_seconds
    )

    # Вычисляем время сброса
    current_window = int(time.time() // window_seconds)
    reset_time = (current_window + 1) * window_seconds

    if not allowed:
        return Response(
            content='{"error": "Rate limit exceeded"}',
            status_code=429,
            headers={
                "Content-Type": "application/json",
                "Retry-After": str(int(reset_time - time.time())),
                "X-RateLimit-Limit": str(limit),
                "X-RateLimit-Remaining": "0",
                "X-RateLimit-Reset": str(int(reset_time))
            }
        )

    response = await call_next(request)

    response.headers["X-RateLimit-Limit"] = str(limit)
    response.headers["X-RateLimit-Remaining"] = str(remaining)
    response.headers["X-RateLimit-Reset"] = str(int(reset_time))

    return response
```

## Стратегии идентификации клиента

### По IP-адресу

```python
def get_client_identifier(request: Request) -> str:
    # Учитываем прокси
    forwarded = request.headers.get("X-Forwarded-For")
    if forwarded:
        return forwarded.split(",")[0].strip()
    return request.client.host
```

### По API ключу

```python
def get_client_identifier(request: Request) -> str:
    api_key = request.headers.get("X-API-Key")
    if api_key:
        return f"api_key:{api_key}"
    return f"ip:{request.client.host}"
```

### По пользователю (после аутентификации)

```python
def get_client_identifier(request: Request) -> str:
    if hasattr(request.state, 'user'):
        return f"user:{request.state.user.id}"
    return f"ip:{request.client.host}"
```

## Гибкие лимиты

### Разные лимиты для разных планов

```python
RATE_LIMITS = {
    "free": {"limit": 100, "window": 3600},      # 100/час
    "basic": {"limit": 1000, "window": 3600},    # 1000/час
    "premium": {"limit": 10000, "window": 3600}, # 10000/час
    "enterprise": {"limit": 100000, "window": 3600}
}

async def get_user_plan(api_key: str) -> str:
    # Запрос к БД или кэшу
    return "basic"

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    api_key = request.headers.get("X-API-Key", "anonymous")
    plan = await get_user_plan(api_key)

    limits = RATE_LIMITS.get(plan, RATE_LIMITS["free"])

    allowed, remaining = token_bucket_rate_limit(
        key=api_key,
        capacity=limits["limit"],
        refill_rate=limits["limit"] / limits["window"]
    )

    # ...
```

### Разные лимиты для разных endpoints

```python
ENDPOINT_LIMITS = {
    "/api/search": {"limit": 10, "window": 60},
    "/api/users": {"limit": 100, "window": 60},
    "/api/auth/login": {"limit": 5, "window": 300},
    "default": {"limit": 60, "window": 60}
}

@app.middleware("http")
async def rate_limit_middleware(request: Request, call_next):
    path = request.url.path
    limits = ENDPOINT_LIMITS.get(path, ENDPOINT_LIMITS["default"])

    # Особый лимит для чувствительных endpoints
    client_id = get_client_identifier(request)
    key = f"{client_id}:{path}"

    allowed, remaining = fixed_window_rate_limit(
        key=key,
        limit=limits["limit"],
        window_seconds=limits["window"]
    )
    # ...
```

## Реализация на Node.js

### С использованием express-rate-limit

```javascript
const express = require('express');
const rateLimit = require('express-rate-limit');
const RedisStore = require('rate-limit-redis');
const Redis = require('ioredis');

const app = express();
const redis = new Redis();

// Базовый rate limiter
const limiter = rateLimit({
    store: new RedisStore({
        sendCommand: (...args) => redis.call(...args),
    }),
    windowMs: 60 * 1000, // 1 минута
    max: 100, // 100 запросов
    standardHeaders: true,
    legacyHeaders: false,
    message: {
        error: 'Too many requests',
        retryAfter: 60
    },
    keyGenerator: (req) => {
        return req.headers['x-api-key'] || req.ip;
    }
});

app.use('/api/', limiter);

// Отдельный limiter для аутентификации
const authLimiter = rateLimit({
    windowMs: 15 * 60 * 1000, // 15 минут
    max: 5, // 5 попыток
    message: {
        error: 'Too many login attempts',
        retryAfter: 900
    }
});

app.use('/api/auth/login', authLimiter);

app.listen(3000);
```

### Собственная реализация

```javascript
const Redis = require('ioredis');
const redis = new Redis();

class RateLimiter {
    constructor(options = {}) {
        this.limit = options.limit || 100;
        this.windowSeconds = options.windowSeconds || 60;
    }

    async isAllowed(key) {
        const now = Date.now();
        const windowStart = now - (this.windowSeconds * 1000);
        const redisKey = `rate_limit:${key}`;

        const multi = redis.multi();
        multi.zremrangebyscore(redisKey, 0, windowStart);
        multi.zadd(redisKey, now, `${now}`);
        multi.zcard(redisKey);
        multi.expire(redisKey, this.windowSeconds);

        const results = await multi.exec();
        const count = results[2][1];

        if (count > this.limit) {
            await redis.zrem(redisKey, `${now}`);
            return {
                allowed: false,
                remaining: 0,
                resetAt: Math.ceil(now / 1000 / this.windowSeconds + 1) * this.windowSeconds
            };
        }

        return {
            allowed: true,
            remaining: this.limit - count,
            resetAt: Math.ceil(now / 1000 / this.windowSeconds + 1) * this.windowSeconds
        };
    }
}

// Middleware
const rateLimiter = new RateLimiter({ limit: 100, windowSeconds: 60 });

const rateLimitMiddleware = async (req, res, next) => {
    const key = req.headers['x-api-key'] || req.ip;
    const result = await rateLimiter.isAllowed(key);

    res.set({
        'X-RateLimit-Limit': '100',
        'X-RateLimit-Remaining': result.remaining.toString(),
        'X-RateLimit-Reset': result.resetAt.toString()
    });

    if (!result.allowed) {
        res.status(429).json({
            error: 'Rate limit exceeded',
            retryAfter: result.resetAt - Math.floor(Date.now() / 1000)
        });
        return;
    }

    next();
};

app.use('/api/', rateLimitMiddleware);
```

## Примеры из реальных API

### GitHub API

```http
X-RateLimit-Limit: 5000
X-RateLimit-Remaining: 4999
X-RateLimit-Reset: 1640995200
X-RateLimit-Used: 1
X-RateLimit-Resource: core
```

Лимиты:
- Authenticated: 5000 запросов/час
- Unauthenticated: 60 запросов/час

### Twitter API v2

```http
x-rate-limit-limit: 300
x-rate-limit-remaining: 299
x-rate-limit-reset: 1640995200
```

### Stripe API

```http
# При превышении лимита:
HTTP/1.1 429 Too Many Requests
Retry-After: 1
```

Лимит: 100 запросов/сек в live mode

## Best Practices

### Что делать:

1. **Возвращайте информативные заголовки:**
   - `X-RateLimit-Limit`, `X-RateLimit-Remaining`, `X-RateLimit-Reset`

2. **Используйте Retry-After:**
   ```http
   HTTP/1.1 429 Too Many Requests
   Retry-After: 60
   ```

3. **Дифференцируйте лимиты:**
   - По плану подписки
   - По типу endpoint
   - По методу (GET vs POST)

4. **Логируйте превышения:**
   - Для анализа и выявления злоупотреблений

5. **Используйте распределённое хранилище:**
   - Redis для масштабируемости

6. **Предоставляйте graceful degradation:**
   - Снижайте функциональность вместо полного отказа

### Типичные ошибки:

1. **Отсутствие rate limiting:**
   - API уязвим для атак

2. **Только по IP:**
   - Не учитывает NAT и прокси

3. **Слишком жёсткие лимиты:**
   - Мешают нормальной работе

4. **Нет информации о лимитах:**
   - Клиенты не могут адаптироваться

5. **In-memory хранение:**
   - Не работает при масштабировании

## Заключение

Rate limiting — необходимый компонент любого production API. Выбор алгоритма зависит от требований:

- **Fixed Window** — простота, подходит для начала
- **Sliding Window** — точность, хорошо для большинства случаев
- **Token Bucket** — гибкость, позволяет bursts
- **Leaky Bucket** — равномерная нагрузка

Всегда предоставляйте клиентам информацию о лимитах через HTTP-заголовки и документируйте политику rate limiting в API документации.
