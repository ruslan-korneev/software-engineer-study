# Rate Limiting/Throttling

## Введение

Rate Limiting (ограничение частоты запросов) и Throttling (дросселирование) — это техники контроля количества запросов, которые клиент может отправить к API за определённый период времени. Эти механизмы защищают API от перегрузки, злоупотреблений и обеспечивают справедливое распределение ресурсов.

## Зачем нужен Rate Limiting

```
┌─────────────────────────────────────────────────────────────────┐
│                    Причины использования                         │
├─────────────────────────────────────────────────────────────────┤
│ 1. Защита от DDoS атак                                          │
│ 2. Предотвращение злоупотреблений                               │
│ 3. Справедливое распределение ресурсов                         │
│ 4. Контроль затрат на инфраструктуру                           │
│ 5. Обеспечение стабильности сервиса                            │
│ 6. Монетизация (разные лимиты для разных тарифов)              │
└─────────────────────────────────────────────────────────────────┘
```

## Алгоритмы Rate Limiting

### Fixed Window Counter

```python
import time
from dataclasses import dataclass
from typing import Dict
import threading

@dataclass
class FixedWindowLimiter:
    """
    Fixed Window Counter

    Простейший алгоритм: делим время на фиксированные окна
    и считаем запросы в каждом окне.

    Недостаток: burst на границе окна может удвоить лимит
    """

    max_requests: int  # Максимум запросов за окно
    window_seconds: int  # Размер окна в секундах

    def __post_init__(self):
        self.counters: Dict[str, Dict[str, int]] = {}
        self.lock = threading.Lock()

    def _get_window_key(self) -> str:
        """Получение ключа текущего окна"""
        return str(int(time.time()) // self.window_seconds)

    def is_allowed(self, client_id: str) -> bool:
        """Проверка, разрешён ли запрос"""
        window_key = self._get_window_key()

        with self.lock:
            if client_id not in self.counters:
                self.counters[client_id] = {}

            client_windows = self.counters[client_id]

            # Очищаем старые окна
            client_windows = {
                k: v for k, v in client_windows.items()
                if k >= window_key
            }
            self.counters[client_id] = client_windows

            # Проверяем текущее окно
            current_count = client_windows.get(window_key, 0)

            if current_count >= self.max_requests:
                return False

            client_windows[window_key] = current_count + 1
            return True

    def get_remaining(self, client_id: str) -> int:
        """Получение оставшегося количества запросов"""
        window_key = self._get_window_key()

        with self.lock:
            if client_id not in self.counters:
                return self.max_requests

            current_count = self.counters[client_id].get(window_key, 0)
            return max(0, self.max_requests - current_count)

# Использование
limiter = FixedWindowLimiter(max_requests=100, window_seconds=60)

if limiter.is_allowed("user:123"):
    print("Request allowed")
else:
    print("Rate limit exceeded")
```

### Sliding Window Log

```python
from dataclasses import dataclass, field
from typing import Dict, List
from collections import deque
import time
import threading

@dataclass
class SlidingWindowLogLimiter:
    """
    Sliding Window Log

    Хранит timestamp каждого запроса.
    Более точный, но потребляет больше памяти.
    """

    max_requests: int
    window_seconds: int

    def __post_init__(self):
        self.requests: Dict[str, deque] = {}
        self.lock = threading.Lock()

    def is_allowed(self, client_id: str) -> bool:
        now = time.time()
        window_start = now - self.window_seconds

        with self.lock:
            if client_id not in self.requests:
                self.requests[client_id] = deque()

            # Удаляем устаревшие запросы
            client_requests = self.requests[client_id]
            while client_requests and client_requests[0] < window_start:
                client_requests.popleft()

            if len(client_requests) >= self.max_requests:
                return False

            client_requests.append(now)
            return True

    def get_reset_time(self, client_id: str) -> float:
        """Время до сброса лимита (в секундах)"""
        with self.lock:
            if client_id not in self.requests or not self.requests[client_id]:
                return 0

            oldest_request = self.requests[client_id][0]
            reset_time = oldest_request + self.window_seconds - time.time()
            return max(0, reset_time)
```

### Sliding Window Counter

```python
from dataclasses import dataclass
from typing import Dict, Tuple
import time
import threading
import math

@dataclass
class SlidingWindowCounterLimiter:
    """
    Sliding Window Counter

    Комбинация Fixed Window и Sliding Window.
    Интерполирует между текущим и предыдущим окном.
    Баланс между точностью и использованием памяти.
    """

    max_requests: int
    window_seconds: int

    def __post_init__(self):
        # {client_id: {window_key: count}}
        self.counters: Dict[str, Dict[str, int]] = {}
        self.lock = threading.Lock()

    def _get_window_info(self) -> Tuple[str, str, float]:
        """
        Возвращает:
        - ключ текущего окна
        - ключ предыдущего окна
        - процент прошедшего времени в текущем окне
        """
        now = time.time()
        current_window = int(now) // self.window_seconds
        window_start = current_window * self.window_seconds
        elapsed_percent = (now - window_start) / self.window_seconds

        return (
            str(current_window),
            str(current_window - 1),
            elapsed_percent
        )

    def is_allowed(self, client_id: str) -> bool:
        current_key, prev_key, elapsed_percent = self._get_window_info()

        with self.lock:
            if client_id not in self.counters:
                self.counters[client_id] = {}

            windows = self.counters[client_id]

            # Получаем счётчики
            current_count = windows.get(current_key, 0)
            prev_count = windows.get(prev_key, 0)

            # Взвешенный подсчёт
            # Чем больше времени прошло в текущем окне,
            # тем меньше вес предыдущего окна
            weighted_count = (
                current_count +
                prev_count * (1 - elapsed_percent)
            )

            if weighted_count >= self.max_requests:
                return False

            # Увеличиваем счётчик текущего окна
            windows[current_key] = current_count + 1

            # Очищаем старые окна
            old_keys = [k for k in windows if k < prev_key]
            for k in old_keys:
                del windows[k]

            return True

# Пример: 100 запросов в минуту с более плавным распределением
limiter = SlidingWindowCounterLimiter(max_requests=100, window_seconds=60)
```

### Token Bucket

```python
from dataclasses import dataclass
from typing import Dict
import time
import threading

@dataclass
class TokenBucketLimiter:
    """
    Token Bucket Algorithm

    Токены добавляются в корзину с постоянной скоростью.
    Каждый запрос потребляет токен.
    Позволяет burst'ы в пределах размера корзины.

    Преимущества:
    - Сглаживает трафик
    - Позволяет короткие всплески
    - Легко настраивается
    """

    bucket_size: int  # Максимальное количество токенов
    refill_rate: float  # Токенов в секунду

    def __post_init__(self):
        # {client_id: (tokens, last_refill_time)}
        self.buckets: Dict[str, tuple] = {}
        self.lock = threading.Lock()

    def _refill(self, client_id: str) -> float:
        """Пополняет корзину и возвращает текущее количество токенов"""
        now = time.time()

        if client_id not in self.buckets:
            self.buckets[client_id] = (self.bucket_size, now)
            return self.bucket_size

        tokens, last_refill = self.buckets[client_id]

        # Вычисляем новые токены
        elapsed = now - last_refill
        new_tokens = min(
            self.bucket_size,
            tokens + elapsed * self.refill_rate
        )

        self.buckets[client_id] = (new_tokens, now)
        return new_tokens

    def is_allowed(self, client_id: str, tokens_needed: int = 1) -> bool:
        """
        Проверка и потребление токенов

        Args:
            client_id: Идентификатор клиента
            tokens_needed: Количество токенов для запроса
                          (полезно для разной "стоимости" операций)
        """
        with self.lock:
            available = self._refill(client_id)

            if available >= tokens_needed:
                tokens, last_refill = self.buckets[client_id]
                self.buckets[client_id] = (tokens - tokens_needed, last_refill)
                return True

            return False

    def get_wait_time(self, client_id: str, tokens_needed: int = 1) -> float:
        """Время ожидания до получения нужного количества токенов"""
        with self.lock:
            available = self._refill(client_id)

            if available >= tokens_needed:
                return 0

            tokens_missing = tokens_needed - available
            return tokens_missing / self.refill_rate

# Пример: 10 запросов/сек с возможностью burst до 50
limiter = TokenBucketLimiter(bucket_size=50, refill_rate=10)

# Разные "стоимости" операций
if limiter.is_allowed("user:123", tokens_needed=1):  # GET - 1 токен
    pass
if limiter.is_allowed("user:123", tokens_needed=5):  # POST - 5 токенов
    pass
```

### Leaky Bucket

```python
from dataclasses import dataclass
from typing import Dict
from collections import deque
import time
import threading
import asyncio

@dataclass
class LeakyBucketLimiter:
    """
    Leaky Bucket Algorithm

    Запросы добавляются в очередь и обрабатываются
    с постоянной скоростью. "Протекает" с фиксированной скоростью.

    Преимущества:
    - Гарантирует постоянную скорость обработки
    - Сглаживает burst'ы

    Недостатки:
    - Добавляет задержку
    - Требует очередь
    """

    bucket_size: int  # Размер очереди
    leak_rate: float  # Запросов в секунду

    def __post_init__(self):
        self.queues: Dict[str, deque] = {}
        self.last_leak: Dict[str, float] = {}
        self.lock = threading.Lock()

    def _leak(self, client_id: str):
        """Обработка накопившихся запросов"""
        now = time.time()

        if client_id not in self.last_leak:
            self.last_leak[client_id] = now
            return

        elapsed = now - self.last_leak[client_id]
        requests_to_process = int(elapsed * self.leak_rate)

        if requests_to_process > 0 and client_id in self.queues:
            queue = self.queues[client_id]
            for _ in range(min(requests_to_process, len(queue))):
                queue.popleft()
            self.last_leak[client_id] = now

    def is_allowed(self, client_id: str) -> bool:
        """Добавление запроса в очередь"""
        with self.lock:
            self._leak(client_id)

            if client_id not in self.queues:
                self.queues[client_id] = deque()

            queue = self.queues[client_id]

            if len(queue) >= self.bucket_size:
                return False

            queue.append(time.time())
            return True

    def get_queue_position(self, client_id: str) -> int:
        """Позиция в очереди"""
        with self.lock:
            if client_id not in self.queues:
                return 0
            return len(self.queues[client_id])

# Пример: очередь на 100 запросов, обработка 10 запросов/сек
limiter = LeakyBucketLimiter(bucket_size=100, leak_rate=10)
```

### Сравнение алгоритмов

```python
"""
┌──────────────────────────────────────────────────────────────────────────┐
│ Алгоритм           │ Память │ Точность │ Burst  │ Сложность │ Случай    │
├──────────────────────────────────────────────────────────────────────────┤
│ Fixed Window       │ O(1)   │ Низкая   │ 2x     │ Простая   │ Базовый   │
│ Sliding Window Log │ O(n)   │ Высокая  │ Нет    │ Простая   │ Точный    │
│ Sliding Window Cnt │ O(1)   │ Средняя  │ 1x     │ Средняя   │ Баланс    │
│ Token Bucket       │ O(1)   │ Высокая  │ Config │ Средняя   │ API       │
│ Leaky Bucket       │ O(n)   │ Высокая  │ Очередь│ Средняя   │ Сглажив.  │
└──────────────────────────────────────────────────────────────────────────┘
"""
```

## Реализация с Redis

### Distributed Rate Limiter

```python
import redis.asyncio as redis
import time
from typing import Optional, Tuple
from dataclasses import dataclass

@dataclass
class RedisRateLimiter:
    """
    Распределённый Rate Limiter на Redis

    Использует Lua скрипты для атомарных операций
    """

    redis_client: redis.Redis
    key_prefix: str = "ratelimit"

    # Lua скрипт для Token Bucket
    TOKEN_BUCKET_SCRIPT = """
    local key = KEYS[1]
    local bucket_size = tonumber(ARGV[1])
    local refill_rate = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])
    local tokens_needed = tonumber(ARGV[4])

    local bucket = redis.call('HMGET', key, 'tokens', 'last_refill')
    local tokens = tonumber(bucket[1]) or bucket_size
    local last_refill = tonumber(bucket[2]) or now

    -- Refill tokens
    local elapsed = now - last_refill
    tokens = math.min(bucket_size, tokens + elapsed * refill_rate)

    -- Check and consume
    if tokens >= tokens_needed then
        tokens = tokens - tokens_needed
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, math.ceil(bucket_size / refill_rate) + 1)
        return {1, tokens}
    else
        redis.call('HMSET', key, 'tokens', tokens, 'last_refill', now)
        redis.call('EXPIRE', key, math.ceil(bucket_size / refill_rate) + 1)
        return {0, tokens}
    """

    # Lua скрипт для Sliding Window
    SLIDING_WINDOW_SCRIPT = """
    local key = KEYS[1]
    local window_size = tonumber(ARGV[1])
    local max_requests = tonumber(ARGV[2])
    local now = tonumber(ARGV[3])

    -- Remove old entries
    local window_start = now - window_size
    redis.call('ZREMRANGEBYSCORE', key, 0, window_start)

    -- Count current requests
    local count = redis.call('ZCARD', key)

    if count < max_requests then
        redis.call('ZADD', key, now, now .. ':' .. math.random())
        redis.call('EXPIRE', key, window_size + 1)
        return {1, max_requests - count - 1}
    else
        redis.call('EXPIRE', key, window_size + 1)
        return {0, 0}
    """

    def __post_init__(self):
        self._token_bucket_sha = None
        self._sliding_window_sha = None

    async def _ensure_scripts(self):
        """Загрузка Lua скриптов"""
        if self._token_bucket_sha is None:
            self._token_bucket_sha = await self.redis_client.script_load(
                self.TOKEN_BUCKET_SCRIPT
            )
        if self._sliding_window_sha is None:
            self._sliding_window_sha = await self.redis_client.script_load(
                self.SLIDING_WINDOW_SCRIPT
            )

    async def check_token_bucket(
        self,
        client_id: str,
        bucket_size: int,
        refill_rate: float,
        tokens_needed: int = 1
    ) -> Tuple[bool, int]:
        """
        Проверка Token Bucket

        Returns:
            (allowed, remaining_tokens)
        """
        await self._ensure_scripts()

        key = f"{self.key_prefix}:token:{client_id}"
        now = time.time()

        result = await self.redis_client.evalsha(
            self._token_bucket_sha,
            1,
            key,
            bucket_size,
            refill_rate,
            now,
            tokens_needed
        )

        return bool(result[0]), int(result[1])

    async def check_sliding_window(
        self,
        client_id: str,
        window_seconds: int,
        max_requests: int
    ) -> Tuple[bool, int]:
        """
        Проверка Sliding Window

        Returns:
            (allowed, remaining_requests)
        """
        await self._ensure_scripts()

        key = f"{self.key_prefix}:window:{client_id}"
        now = time.time()

        result = await self.redis_client.evalsha(
            self._sliding_window_sha,
            1,
            key,
            window_seconds,
            max_requests,
            now
        )

        return bool(result[0]), int(result[1])

# Использование
redis_client = redis.from_url("redis://localhost:6379")
limiter = RedisRateLimiter(redis_client=redis_client)

async def check_request(user_id: str):
    allowed, remaining = await limiter.check_token_bucket(
        client_id=user_id,
        bucket_size=100,
        refill_rate=10,  # 10 запросов в секунду
        tokens_needed=1
    )
    return allowed, remaining
```

## Интеграция с FastAPI

### Rate Limiting Middleware

```python
from fastapi import FastAPI, Request, Response, HTTPException
from fastapi.responses import JSONResponse
from starlette.middleware.base import BaseHTTPMiddleware
from typing import Callable, Optional, Dict
import time

class RateLimitConfig:
    """Конфигурация Rate Limiting"""

    def __init__(
        self,
        requests_per_minute: int = 60,
        requests_per_hour: int = 1000,
        requests_per_day: int = 10000,
        burst_size: int = 10
    ):
        self.requests_per_minute = requests_per_minute
        self.requests_per_hour = requests_per_hour
        self.requests_per_day = requests_per_day
        self.burst_size = burst_size

class RateLimitMiddleware(BaseHTTPMiddleware):
    """Middleware для Rate Limiting"""

    def __init__(
        self,
        app,
        limiter: RedisRateLimiter,
        config: RateLimitConfig = None,
        key_func: Callable[[Request], str] = None
    ):
        super().__init__(app)
        self.limiter = limiter
        self.config = config or RateLimitConfig()
        self.key_func = key_func or self._default_key_func

    def _default_key_func(self, request: Request) -> str:
        """Получение ключа клиента по IP"""
        forwarded = request.headers.get("X-Forwarded-For")
        if forwarded:
            return forwarded.split(",")[0].strip()
        return request.client.host if request.client else "unknown"

    async def dispatch(self, request: Request, call_next):
        client_id = self.key_func(request)

        # Проверяем лимит
        allowed, remaining = await self.limiter.check_token_bucket(
            client_id=client_id,
            bucket_size=self.config.burst_size,
            refill_rate=self.config.requests_per_minute / 60
        )

        if not allowed:
            return JSONResponse(
                status_code=429,
                content={
                    "error": "Too Many Requests",
                    "message": "Rate limit exceeded. Please try again later.",
                    "retry_after": 60
                },
                headers={
                    "Retry-After": "60",
                    "X-RateLimit-Limit": str(self.config.requests_per_minute),
                    "X-RateLimit-Remaining": "0",
                    "X-RateLimit-Reset": str(int(time.time()) + 60)
                }
            )

        response = await call_next(request)

        # Добавляем заголовки rate limit
        response.headers["X-RateLimit-Limit"] = str(self.config.requests_per_minute)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        response.headers["X-RateLimit-Reset"] = str(int(time.time()) + 60)

        return response

# Использование
app = FastAPI()
app.add_middleware(
    RateLimitMiddleware,
    limiter=limiter,
    config=RateLimitConfig(requests_per_minute=100)
)
```

### Rate Limiting Decorator

```python
from functools import wraps
from fastapi import Request, HTTPException
from typing import Callable, Optional

def rate_limit(
    requests: int,
    window: int,
    key_func: Optional[Callable[[Request], str]] = None,
    scope: str = "endpoint"
):
    """
    Декоратор для rate limiting на уровне endpoint

    Args:
        requests: Максимум запросов
        window: Окно в секундах
        key_func: Функция для получения ключа клиента
        scope: "endpoint", "user", "global"
    """
    def decorator(func):
        @wraps(func)
        async def wrapper(*args, request: Request = None, **kwargs):
            if request is None:
                # Ищем request в args
                for arg in args:
                    if isinstance(arg, Request):
                        request = arg
                        break

            if request is None:
                return await func(*args, **kwargs)

            # Получаем ключ клиента
            if key_func:
                client_key = key_func(request)
            else:
                client_key = request.client.host if request.client else "unknown"

            # Формируем полный ключ
            if scope == "endpoint":
                full_key = f"{request.url.path}:{client_key}"
            elif scope == "user":
                full_key = f"user:{client_key}"
            else:
                full_key = f"global:{client_key}"

            # Проверяем лимит
            allowed, remaining = await limiter.check_sliding_window(
                client_id=full_key,
                window_seconds=window,
                max_requests=requests
            )

            if not allowed:
                raise HTTPException(
                    status_code=429,
                    detail={
                        "error": "Rate limit exceeded",
                        "limit": requests,
                        "window": window,
                        "retry_after": window
                    },
                    headers={"Retry-After": str(window)}
                )

            return await func(*args, **kwargs)

        return wrapper
    return decorator

# Использование
@app.get("/api/expensive-operation")
@rate_limit(requests=10, window=60, scope="endpoint")
async def expensive_operation(request: Request):
    return {"result": "success"}

@app.post("/api/login")
@rate_limit(requests=5, window=300, scope="endpoint")  # 5 попыток за 5 минут
async def login(request: Request):
    return {"token": "..."}
```

## Многоуровневый Rate Limiting

```python
from dataclasses import dataclass
from typing import Dict, List, Optional, Tuple
from enum import Enum

class RateLimitTier(Enum):
    FREE = "free"
    BASIC = "basic"
    PRO = "pro"
    ENTERPRISE = "enterprise"

@dataclass
class TierLimits:
    requests_per_second: int
    requests_per_minute: int
    requests_per_hour: int
    requests_per_day: int
    burst_size: int

# Конфигурация лимитов по тарифам
TIER_CONFIGS: Dict[RateLimitTier, TierLimits] = {
    RateLimitTier.FREE: TierLimits(
        requests_per_second=1,
        requests_per_minute=30,
        requests_per_hour=500,
        requests_per_day=5000,
        burst_size=5
    ),
    RateLimitTier.BASIC: TierLimits(
        requests_per_second=10,
        requests_per_minute=300,
        requests_per_hour=5000,
        requests_per_day=50000,
        burst_size=20
    ),
    RateLimitTier.PRO: TierLimits(
        requests_per_second=50,
        requests_per_minute=1500,
        requests_per_hour=25000,
        requests_per_day=250000,
        burst_size=100
    ),
    RateLimitTier.ENTERPRISE: TierLimits(
        requests_per_second=200,
        requests_per_minute=6000,
        requests_per_hour=100000,
        requests_per_day=1000000,
        burst_size=500
    ),
}

class MultiTierRateLimiter:
    """Многоуровневый Rate Limiter с поддержкой тарифов"""

    def __init__(self, limiter: RedisRateLimiter):
        self.limiter = limiter

    async def check_all_limits(
        self,
        client_id: str,
        tier: RateLimitTier
    ) -> Tuple[bool, Dict[str, int]]:
        """
        Проверка всех уровней лимитов

        Returns:
            (allowed, remaining_by_level)
        """
        config = TIER_CONFIGS[tier]

        # Проверяем все уровни параллельно
        results = await asyncio.gather(
            self.limiter.check_sliding_window(
                f"{client_id}:second",
                1,
                config.requests_per_second
            ),
            self.limiter.check_sliding_window(
                f"{client_id}:minute",
                60,
                config.requests_per_minute
            ),
            self.limiter.check_sliding_window(
                f"{client_id}:hour",
                3600,
                config.requests_per_hour
            ),
            self.limiter.check_sliding_window(
                f"{client_id}:day",
                86400,
                config.requests_per_day
            ),
        )

        remaining = {
            "second": results[0][1],
            "minute": results[1][1],
            "hour": results[2][1],
            "day": results[3][1]
        }

        # Разрешено только если все уровни пройдены
        allowed = all(r[0] for r in results)

        return allowed, remaining

# Использование
multi_limiter = MultiTierRateLimiter(limiter)

async def handle_request(user_id: str, tier: RateLimitTier):
    allowed, remaining = await multi_limiter.check_all_limits(user_id, tier)

    if not allowed:
        # Определяем, какой лимит исчерпан
        exhausted = [
            level for level, count in remaining.items()
            if count == 0
        ]
        raise HTTPException(
            status_code=429,
            detail={
                "error": "Rate limit exceeded",
                "exhausted_limits": exhausted,
                "remaining": remaining
            }
        )
```

## Rate Limiting по операциям

```python
from enum import Enum
from dataclasses import dataclass

class OperationType(Enum):
    READ = "read"
    WRITE = "write"
    DELETE = "delete"
    SEARCH = "search"
    EXPORT = "export"

@dataclass
class OperationCost:
    """Стоимость операции в токенах"""
    operation: OperationType
    cost: int

# Разные операции имеют разную "стоимость"
OPERATION_COSTS = {
    OperationType.READ: 1,
    OperationType.WRITE: 5,
    OperationType.DELETE: 10,
    OperationType.SEARCH: 3,
    OperationType.EXPORT: 50,
}

class CostBasedRateLimiter:
    """Rate Limiter с учётом стоимости операций"""

    def __init__(
        self,
        limiter: RedisRateLimiter,
        token_budget: int = 1000,  # Токенов в минуту
        refill_rate: float = 16.67  # Токенов в секунду
    ):
        self.limiter = limiter
        self.token_budget = token_budget
        self.refill_rate = refill_rate

    async def check_operation(
        self,
        client_id: str,
        operation: OperationType
    ) -> Tuple[bool, int]:
        """Проверка возможности выполнить операцию"""
        cost = OPERATION_COSTS[operation]

        return await self.limiter.check_token_bucket(
            client_id=client_id,
            bucket_size=self.token_budget,
            refill_rate=self.refill_rate,
            tokens_needed=cost
        )

# Использование
cost_limiter = CostBasedRateLimiter(limiter)

@app.get("/api/items/{item_id}")
async def get_item(item_id: str, request: Request):
    allowed, remaining = await cost_limiter.check_operation(
        request.client.host,
        OperationType.READ
    )
    if not allowed:
        raise HTTPException(status_code=429)
    return {"item": item_id}

@app.post("/api/items")
async def create_item(request: Request, data: dict):
    allowed, remaining = await cost_limiter.check_operation(
        request.client.host,
        OperationType.WRITE  # Стоит 5 токенов
    )
    if not allowed:
        raise HTTPException(status_code=429)
    return {"created": True}

@app.get("/api/export")
async def export_data(request: Request):
    allowed, remaining = await cost_limiter.check_operation(
        request.client.host,
        OperationType.EXPORT  # Стоит 50 токенов
    )
    if not allowed:
        raise HTTPException(
            status_code=429,
            detail="Export operation requires 50 tokens"
        )
    return {"data": [...]}
```

## HTTP Headers для Rate Limiting

```python
from fastapi import Response
from typing import Dict
import time

class RateLimitHeaders:
    """Стандартные заголовки Rate Limiting"""

    @staticmethod
    def add_headers(
        response: Response,
        limit: int,
        remaining: int,
        reset_timestamp: int,
        policy: str = None
    ):
        """
        Добавляет стандартные заголовки rate limit

        Заголовки:
        - X-RateLimit-Limit: максимум запросов
        - X-RateLimit-Remaining: осталось запросов
        - X-RateLimit-Reset: timestamp сброса
        - RateLimit-Policy: политика (RFC draft)
        """
        response.headers["X-RateLimit-Limit"] = str(limit)
        response.headers["X-RateLimit-Remaining"] = str(remaining)
        response.headers["X-RateLimit-Reset"] = str(reset_timestamp)

        # RFC Draft format
        response.headers["RateLimit-Limit"] = str(limit)
        response.headers["RateLimit-Remaining"] = str(remaining)
        response.headers["RateLimit-Reset"] = str(reset_timestamp - int(time.time()))

        if policy:
            response.headers["RateLimit-Policy"] = policy

    @staticmethod
    def get_retry_after_headers(retry_seconds: int) -> Dict[str, str]:
        """Заголовки для 429 ответа"""
        return {
            "Retry-After": str(retry_seconds),
            "X-RateLimit-Remaining": "0"
        }

# Пример ответа 429
@app.exception_handler(RateLimitExceeded)
async def rate_limit_handler(request: Request, exc: RateLimitExceeded):
    return JSONResponse(
        status_code=429,
        content={
            "error": "rate_limit_exceeded",
            "message": "Too many requests. Please slow down.",
            "retry_after": exc.retry_after,
            "documentation": "https://api.example.com/docs/rate-limits"
        },
        headers=RateLimitHeaders.get_retry_after_headers(exc.retry_after)
    )
```

## Мониторинг и метрики

```python
from prometheus_client import Counter, Histogram, Gauge
from dataclasses import dataclass

class RateLimitMetrics:
    """Метрики Rate Limiting для Prometheus"""

    # Счётчик запросов по результату
    requests_total = Counter(
        'rate_limit_requests_total',
        'Total rate limited requests',
        ['client_tier', 'endpoint', 'result']  # result: allowed, rejected
    )

    # Гистограмма оставшихся запросов
    remaining_requests = Histogram(
        'rate_limit_remaining',
        'Remaining requests when checked',
        ['client_tier'],
        buckets=[0, 1, 5, 10, 25, 50, 100, 250, 500, 1000]
    )

    # Текущая загрузка по клиентам
    current_usage = Gauge(
        'rate_limit_current_usage_percent',
        'Current usage percentage',
        ['client_id', 'limit_type']
    )

    @classmethod
    def record_request(
        cls,
        client_tier: str,
        endpoint: str,
        allowed: bool,
        remaining: int
    ):
        result = "allowed" if allowed else "rejected"
        cls.requests_total.labels(
            client_tier=client_tier,
            endpoint=endpoint,
            result=result
        ).inc()

        cls.remaining_requests.labels(
            client_tier=client_tier
        ).observe(remaining)

# Интеграция
async def check_with_metrics(
    client_id: str,
    tier: str,
    endpoint: str
) -> Tuple[bool, int]:
    allowed, remaining = await limiter.check_token_bucket(...)

    RateLimitMetrics.record_request(
        client_tier=tier,
        endpoint=endpoint,
        allowed=allowed,
        remaining=remaining
    )

    return allowed, remaining
```

## Best Practices

### 1. Выбор стратегии идентификации клиента

```python
from fastapi import Request
from typing import Optional

class ClientIdentification:
    """Стратегии идентификации клиента"""

    @staticmethod
    def by_ip(request: Request) -> str:
        """По IP адресу (базовый вариант)"""
        forwarded = request.headers.get("X-Forwarded-For")
        if forwarded:
            return forwarded.split(",")[0].strip()
        return request.client.host if request.client else "unknown"

    @staticmethod
    def by_api_key(request: Request) -> Optional[str]:
        """По API ключу (для authenticated API)"""
        return request.headers.get("X-API-Key")

    @staticmethod
    def by_user(request: Request) -> Optional[str]:
        """По ID пользователя (для logged-in users)"""
        # Предполагаем, что user уже в request.state
        if hasattr(request.state, 'user'):
            return f"user:{request.state.user.id}"
        return None

    @staticmethod
    def composite(request: Request) -> str:
        """Комбинированный подход"""
        # Приоритет: user > api_key > ip
        if hasattr(request.state, 'user'):
            return f"user:{request.state.user.id}"

        api_key = request.headers.get("X-API-Key")
        if api_key:
            return f"apikey:{api_key[:8]}"

        return f"ip:{ClientIdentification.by_ip(request)}"
```

### 2. Graceful Degradation

```python
class GracefulRateLimiter:
    """Rate Limiter с graceful degradation при сбоях Redis"""

    def __init__(self, limiter: RedisRateLimiter, fallback_allow: bool = True):
        self.limiter = limiter
        self.fallback_allow = fallback_allow
        self.local_cache: Dict[str, int] = {}

    async def check(
        self,
        client_id: str,
        max_requests: int,
        window: int
    ) -> Tuple[bool, int]:
        try:
            return await self.limiter.check_sliding_window(
                client_id, window, max_requests
            )
        except Exception as e:
            # Redis недоступен — используем fallback
            logger.warning(f"Rate limiter fallback: {e}")

            if self.fallback_allow:
                # Разрешаем запросы с локальным fallback counter
                return self._local_fallback(client_id, max_requests)
            else:
                # Запрещаем все запросы для безопасности
                return False, 0

    def _local_fallback(
        self,
        client_id: str,
        max_requests: int
    ) -> Tuple[bool, int]:
        """Локальный fallback с in-memory счётчиком"""
        count = self.local_cache.get(client_id, 0)

        if count >= max_requests:
            return False, 0

        self.local_cache[client_id] = count + 1
        return True, max_requests - count - 1
```

### 3. Rate Limit Response Best Practices

```python
from pydantic import BaseModel
from typing import Optional, Dict

class RateLimitErrorResponse(BaseModel):
    """Структура ответа при превышении лимита"""
    error: str = "rate_limit_exceeded"
    message: str
    limit: int
    remaining: int
    reset_at: int  # Unix timestamp
    retry_after: int  # Секунды
    tier: Optional[str] = None
    upgrade_url: Optional[str] = None
    documentation: str = "https://api.example.com/docs/rate-limits"

def create_rate_limit_response(
    limit: int,
    reset_at: int,
    tier: str = None
) -> RateLimitErrorResponse:
    """Создание информативного ответа"""
    retry_after = max(0, reset_at - int(time.time()))

    return RateLimitErrorResponse(
        message=f"Rate limit of {limit} requests exceeded. Please wait {retry_after} seconds.",
        limit=limit,
        remaining=0,
        reset_at=reset_at,
        retry_after=retry_after,
        tier=tier,
        upgrade_url="https://example.com/pricing" if tier == "free" else None
    )
```

## Заключение

Эффективный Rate Limiting требует:

1. **Правильного выбора алгоритма** — Token Bucket для API, Sliding Window для точности
2. **Распределённой реализации** — Redis или аналоги для масштабируемости
3. **Многоуровневых лимитов** — разные тарифы, разные лимиты
4. **Информативных ответов** — клиенты должны понимать ситуацию
5. **Мониторинга** — отслеживайте rejected запросы и паттерны использования
6. **Graceful degradation** — система должна работать при сбоях

Rate Limiting — это баланс между защитой API и удобством для пользователей.
