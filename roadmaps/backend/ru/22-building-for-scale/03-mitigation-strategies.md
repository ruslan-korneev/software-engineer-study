# Стратегии смягчения нагрузки (Mitigation Strategies)

## Определение

**Стратегии смягчения нагрузки** — это набор паттернов и техник, которые помогают системе сохранять работоспособность при высоких нагрузках, сбоях зависимостей или деградации компонентов. Эти стратегии обеспечивают устойчивость (resilience) и отказоустойчивость (fault tolerance) распределённых систем.

Основная идея: **лучше работать частично, чем не работать вовсе**.

```
┌─────────────────────────────────────────────────────────────────────┐
│                    СТРАТЕГИИ СМЯГЧЕНИЯ НАГРУЗКИ                     │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │
│   │  Graceful   │    │  Throttling │    │    Rate     │            │
│   │ Degradation │    │             │    │  Limiting   │            │
│   └─────────────┘    └─────────────┘    └─────────────┘            │
│                                                                     │
│   ┌─────────────┐    ┌─────────────┐    ┌─────────────┐            │
│   │ Backpressure│    │   Circuit   │    │    Load     │            │
│   │             │    │   Breaker   │    │  Shedding   │            │
│   └─────────────┘    └─────────────┘    └─────────────┘            │
│                                                                     │
│   ┌─────────────┐                                                   │
│   │   Retry +   │                                                   │
│   │  Backoff    │                                                   │
│   └─────────────┘                                                   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

---

## 1. Graceful Degradation (Плавная деградация)

### Концепция

Плавная деградация означает, что при сбое части системы, приложение продолжает работать с ограниченной функциональностью вместо полного отказа.

```
┌─────────────────────────────────────────────────────────────────┐
│                  УРОВНИ ДЕГРАДАЦИИ СЕРВИСА                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  100% ████████████████████████████  Полная функциональность    │
│   80% ██████████████████████        Без рекомендаций           │
│   60% ████████████████              Без аналитики              │
│   40% ████████████                  Только чтение              │
│   20% ████████                      Кэшированные данные        │
│    0% ████                          Страница "Технические      │
│                                     работы"                     │
└─────────────────────────────────────────────────────────────────┘
```

### Пример на Python

```python
from typing import Optional, Any
from dataclasses import dataclass
from enum import Enum
import logging

logger = logging.getLogger(__name__)

class ServiceStatus(Enum):
    """Статусы сервисов"""
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNAVAILABLE = "unavailable"

@dataclass
class FeatureFlags:
    """Флаги функций для управления деградацией"""
    recommendations_enabled: bool = True
    analytics_enabled: bool = True
    real_time_updates: bool = True
    full_search: bool = True

class GracefulDegradationService:
    """Сервис с поддержкой плавной деградации"""

    def __init__(self):
        self.feature_flags = FeatureFlags()
        self._service_status = {}

    def update_service_status(self, service: str, status: ServiceStatus):
        """Обновляем статус зависимого сервиса"""
        self._service_status[service] = status
        self._adjust_features()

    def _adjust_features(self):
        """Автоматически отключаем функции при деградации зависимостей"""
        # Если сервис рекомендаций недоступен — отключаем рекомендации
        if self._service_status.get("recommendations") == ServiceStatus.UNAVAILABLE:
            self.feature_flags.recommendations_enabled = False
            logger.warning("Рекомендации отключены из-за недоступности сервиса")

        # Если аналитика перегружена — отключаем её
        if self._service_status.get("analytics") in [ServiceStatus.DEGRADED, ServiceStatus.UNAVAILABLE]:
            self.feature_flags.analytics_enabled = False
            logger.warning("Аналитика отключена")

    def get_product(self, product_id: str) -> dict:
        """Получение продукта с fallback-логикой"""
        result = {"id": product_id, "name": "Product Name", "price": 100}

        # Основная функциональность всегда работает

        # Дополнительные данные — только если сервисы доступны
        if self.feature_flags.recommendations_enabled:
            result["recommendations"] = self._get_recommendations(product_id)
        else:
            result["recommendations"] = []  # Fallback: пустой список

        if self.feature_flags.analytics_enabled:
            result["view_count"] = self._get_view_count(product_id)
        else:
            result["view_count"] = None  # Fallback: данные недоступны

        return result

    def _get_recommendations(self, product_id: str) -> list:
        """Получение рекомендаций с fallback на популярные товары"""
        try:
            # Попытка получить персонализированные рекомендации
            return self._call_recommendation_service(product_id)
        except Exception as e:
            logger.warning(f"Fallback на популярные товары: {e}")
            # Fallback: возвращаем кэшированные популярные товары
            return self._get_cached_popular_products()

    def _get_cached_popular_products(self) -> list:
        """Кэшированные популярные товары как fallback"""
        return [{"id": "popular_1"}, {"id": "popular_2"}]
```

### Приоритизация функций

```python
from enum import IntEnum
from typing import Callable

class FeaturePriority(IntEnum):
    """Приоритеты функций (меньше = важнее)"""
    CRITICAL = 1      # Основной функционал (авторизация, оплата)
    HIGH = 2          # Важные функции (корзина, каталог)
    MEDIUM = 3        # Дополнительные (отзывы, рейтинги)
    LOW = 4           # Необязательные (аналитика, рекомендации)
    OPTIONAL = 5      # Декоративные (анимации, виджеты)

class FeatureManager:
    """Управление функциями на основе нагрузки системы"""

    def __init__(self):
        self._features: dict[str, tuple[FeaturePriority, Callable]] = {}
        self._current_threshold = FeaturePriority.OPTIONAL

    def register_feature(self, name: str, priority: FeaturePriority, handler: Callable):
        """Регистрация функции с приоритетом"""
        self._features[name] = (priority, handler)

    def set_load_level(self, cpu_percent: float, memory_percent: float):
        """Автоматическое отключение функций при высокой нагрузке"""
        if cpu_percent > 90 or memory_percent > 90:
            self._current_threshold = FeaturePriority.CRITICAL
        elif cpu_percent > 80 or memory_percent > 80:
            self._current_threshold = FeaturePriority.HIGH
        elif cpu_percent > 70 or memory_percent > 70:
            self._current_threshold = FeaturePriority.MEDIUM
        else:
            self._current_threshold = FeaturePriority.OPTIONAL

    def is_feature_enabled(self, name: str) -> bool:
        """Проверка, включена ли функция"""
        if name not in self._features:
            return False
        priority, _ = self._features[name]
        return priority <= self._current_threshold
```

---

## 2. Throttling (Ограничение скорости обработки)

### Алгоритмы Throttling

```
┌─────────────────────────────────────────────────────────────────────┐
│                      АЛГОРИТМЫ THROTTLING                           │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  FIXED WINDOW (Фиксированное окно)                                 │
│  ┌─────────┬─────────┬─────────┬─────────┐                         │
│  │ 0:00-   │ 0:01-   │ 0:02-   │ 0:03-   │  Окна по 1 минуте      │
│  │  0:01   │  0:02   │  0:03   │  0:04   │  Лимит: 100 req/окно   │
│  │ [87/100]│ [100/100│ [45/100]│ [...]   │                         │
│  └─────────┴─────────┴─────────┴─────────┘                         │
│                                                                     │
│  SLIDING WINDOW (Скользящее окно)                                  │
│       ←────────── 1 минута ──────────→                             │
│  ────●───●●●──●─────●●──●───●●●●───●───→ время                     │
│      Окно "скользит" с каждым запросом                             │
│                                                                     │
│  TOKEN BUCKET (Корзина токенов)                                    │
│  ┌─────────────────────┐                                           │
│  │  ○ ○ ○ ○ ○ ○ ○ ○   │  Capacity: 10 токенов                     │
│  │        ↓↓↓         │  Refill: 1 токен/сек                       │
│  │     ┌─────┐        │                                            │
│  │     │ REQ │ ←── Запрос забирает 1 токен                         │
│  │     └─────┘        │                                            │
│  └─────────────────────┘                                           │
│                                                                     │
│  LEAKY BUCKET (Дырявое ведро)                                      │
│  ┌─────────────────────┐                                           │
│  │  Входящие запросы   │                                           │
│  │     ↓ ↓ ↓ ↓ ↓      │  Запросы падают в ведро                   │
│  │  ████████████████   │  Ведро переполняется → отказ              │
│  │  ████████████████   │                                           │
│  │       │○│          │  Обработка с фиксированной                 │
│  └───────┘ └──────────┘  скоростью (утечка)                        │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Реализация Token Bucket на Python

```python
import time
import threading
from dataclasses import dataclass

@dataclass
class TokenBucket:
    """
    Реализация алгоритма Token Bucket для throttling.

    Токены накапливаются с фиксированной скоростью.
    Каждый запрос потребляет один или несколько токенов.
    """
    capacity: int           # Максимальное количество токенов
    refill_rate: float      # Токенов в секунду

    def __post_init__(self):
        self._tokens = float(self.capacity)
        self._last_refill = time.monotonic()
        self._lock = threading.Lock()

    def _refill(self):
        """Пополнение токенов на основе прошедшего времени"""
        now = time.monotonic()
        elapsed = now - self._last_refill
        # Добавляем токены пропорционально прошедшему времени
        self._tokens = min(
            self.capacity,
            self._tokens + elapsed * self.refill_rate
        )
        self._last_refill = now

    def acquire(self, tokens: int = 1) -> bool:
        """
        Попытка получить токены.

        Returns:
            True если токены получены, False если недостаточно токенов
        """
        with self._lock:
            self._refill()
            if self._tokens >= tokens:
                self._tokens -= tokens
                return True
            return False

    def wait_for_token(self, tokens: int = 1, timeout: float = None) -> bool:
        """Блокирующее ожидание токена с таймаутом"""
        start = time.monotonic()
        while True:
            if self.acquire(tokens):
                return True
            if timeout and (time.monotonic() - start) >= timeout:
                return False
            # Ждём время, необходимое для генерации одного токена
            time.sleep(1.0 / self.refill_rate)


# Пример использования
class ThrottledApiClient:
    """API клиент с ограничением скорости запросов"""

    def __init__(self, requests_per_second: float = 10):
        # 10 запросов в секунду, burst до 20
        self.bucket = TokenBucket(
            capacity=20,
            refill_rate=requests_per_second
        )

    def make_request(self, endpoint: str) -> dict:
        """Выполнение запроса с throttling"""
        if not self.bucket.acquire():
            raise Exception("Rate limit exceeded. Повторите позже.")

        # Выполняем реальный запрос
        return {"status": "ok", "endpoint": endpoint}
```

### Реализация Sliding Window на Python

```python
import time
from collections import deque
import threading

class SlidingWindowRateLimiter:
    """
    Rate limiter на основе скользящего окна.
    Более точный, чем Fixed Window, избегает проблемы boundary burst.
    """

    def __init__(self, max_requests: int, window_seconds: float):
        self.max_requests = max_requests
        self.window_seconds = window_seconds
        # Храним временные метки запросов
        self._timestamps: deque = deque()
        self._lock = threading.Lock()

    def _cleanup_old_requests(self):
        """Удаляем запросы, вышедшие за пределы окна"""
        cutoff = time.monotonic() - self.window_seconds
        while self._timestamps and self._timestamps[0] < cutoff:
            self._timestamps.popleft()

    def allow_request(self) -> bool:
        """Проверка и регистрация запроса"""
        with self._lock:
            self._cleanup_old_requests()

            if len(self._timestamps) < self.max_requests:
                self._timestamps.append(time.monotonic())
                return True
            return False

    def get_wait_time(self) -> float:
        """Время до освобождения слота"""
        with self._lock:
            self._cleanup_old_requests()
            if len(self._timestamps) < self.max_requests:
                return 0.0
            # Время до истечения самого старого запроса
            oldest = self._timestamps[0]
            return (oldest + self.window_seconds) - time.monotonic()
```

---

## 3. Rate Limiting (Ограничение количества запросов)

### Стратегии Rate Limiting

```
┌─────────────────────────────────────────────────────────────────────┐
│                    УРОВНИ RATE LIMITING                             │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  ┌─────────────────────────────────────────────────────────────┐   │
│  │                    GLOBAL LIMIT                              │   │
│  │                 1,000,000 req/min                           │   │
│  │  ┌─────────────────────────────────────────────────────┐   │   │
│  │  │                PER-TIER LIMIT                        │   │   │
│  │  │  Free: 100/min  |  Pro: 1000/min  |  Enterprise: ∞  │   │   │
│  │  │  ┌─────────────────────────────────────────────┐    │   │   │
│  │  │  │            PER-USER LIMIT                    │    │   │   │
│  │  │  │         user_123: 60 req/min                │    │   │   │
│  │  │  │  ┌─────────────────────────────────────┐    │    │   │   │
│  │  │  │  │        PER-IP LIMIT                  │    │    │   │   │
│  │  │  │  │     192.168.1.1: 30 req/min         │    │    │   │   │
│  │  │  │  │  ┌─────────────────────────────┐    │    │    │   │   │
│  │  │  │  │  │    PER-ENDPOINT LIMIT       │    │    │    │   │   │
│  │  │  │  │  │  /api/search: 10 req/min   │    │    │    │   │   │
│  │  │  │  │  └─────────────────────────────┘    │    │    │   │   │
│  │  │  │  └─────────────────────────────────────┘    │    │   │   │
│  │  │  └─────────────────────────────────────────────┘    │   │   │
│  │  └─────────────────────────────────────────────────────┘   │   │
│  └─────────────────────────────────────────────────────────────┘   │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Реализация с Redis

```python
import redis
import time
from typing import Optional
from dataclasses import dataclass

@dataclass
class RateLimitResult:
    """Результат проверки rate limit"""
    allowed: bool
    remaining: int
    reset_at: float
    retry_after: Optional[float] = None

class RedisRateLimiter:
    """
    Распределённый Rate Limiter на основе Redis.
    Поддерживает несколько уровней лимитов.
    """

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    def check_rate_limit(
        self,
        key: str,
        max_requests: int,
        window_seconds: int
    ) -> RateLimitResult:
        """
        Проверка rate limit с использованием Redis MULTI/EXEC.

        Args:
            key: Уникальный ключ (user_id, ip, endpoint и т.д.)
            max_requests: Максимум запросов в окне
            window_seconds: Размер окна в секундах
        """
        now = time.time()
        window_start = now - window_seconds
        redis_key = f"ratelimit:{key}"

        # Атомарная операция с использованием pipeline
        pipe = self.redis.pipeline()

        # Удаляем старые записи
        pipe.zremrangebyscore(redis_key, 0, window_start)
        # Добавляем текущий запрос
        pipe.zadd(redis_key, {str(now): now})
        # Считаем запросы в окне
        pipe.zcard(redis_key)
        # Устанавливаем TTL для автоочистки
        pipe.expire(redis_key, window_seconds)

        _, _, request_count, _ = pipe.execute()

        remaining = max(0, max_requests - request_count)
        reset_at = now + window_seconds

        if request_count > max_requests:
            # Находим время самого старого запроса для retry_after
            oldest = self.redis.zrange(redis_key, 0, 0, withscores=True)
            retry_after = oldest[0][1] + window_seconds - now if oldest else window_seconds

            return RateLimitResult(
                allowed=False,
                remaining=0,
                reset_at=reset_at,
                retry_after=retry_after
            )

        return RateLimitResult(
            allowed=True,
            remaining=remaining,
            reset_at=reset_at
        )


class MultiLevelRateLimiter:
    """Rate limiter с несколькими уровнями проверки"""

    def __init__(self, redis_client: redis.Redis):
        self.limiter = RedisRateLimiter(redis_client)

        # Конфигурация лимитов
        self.limits = {
            "global": (1_000_000, 60),      # 1M req/min глобально
            "per_ip": (100, 60),             # 100 req/min на IP
            "per_user": (1000, 60),          # 1000 req/min на пользователя
            "per_endpoint": {
                "/api/search": (20, 60),     # 20 req/min для поиска
                "/api/export": (5, 300),     # 5 req/5min для экспорта
            }
        }

    def check_all_limits(
        self,
        ip: str,
        user_id: Optional[str],
        endpoint: str
    ) -> RateLimitResult:
        """Проверка всех уровней лимитов"""

        # Проверяем от самого специфичного к общему
        checks = [
            ("global", "global", *self.limits["global"]),
            ("ip", f"ip:{ip}", *self.limits["per_ip"]),
        ]

        if user_id:
            checks.append(("user", f"user:{user_id}", *self.limits["per_user"]))

        if endpoint in self.limits["per_endpoint"]:
            ep_limits = self.limits["per_endpoint"][endpoint]
            checks.append(("endpoint", f"endpoint:{endpoint}:{ip}", *ep_limits))

        for level, key, max_req, window in checks:
            result = self.limiter.check_rate_limit(key, max_req, window)
            if not result.allowed:
                return result

        return RateLimitResult(allowed=True, remaining=999, reset_at=time.time() + 60)
```

---

## 4. Backpressure (Противодавление)

### Концепция

Backpressure — механизм, при котором медленный consumer сигнализирует быстрому producer о необходимости замедлиться.

```
┌─────────────────────────────────────────────────────────────────────┐
│                        BACKPRESSURE                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  БЕЗ BACKPRESSURE (проблема):                                      │
│  ┌──────────┐     ┌──────────────────────┐     ┌──────────┐        │
│  │ Producer │ ══► │ ████████████████████ │ ══► │ Consumer │        │
│  │ 1000/sec │     │ Очередь переполнена! │     │  100/sec │        │
│  └──────────┘     └──────────────────────┘     └──────────┘        │
│                           ↓                                         │
│                    OOM / Потеря данных                              │
│                                                                     │
│  С BACKPRESSURE (решение):                                         │
│  ┌──────────┐     ┌──────────────────────┐     ┌──────────┐        │
│  │ Producer │ ◄── │ "Замедлись!"         │ ◄── │ Consumer │        │
│  │  100/sec │     │ ████████             │     │  100/sec │        │
│  └──────────┘     │ Очередь под контролем│     └──────────┘        │
│       ↓           └──────────────────────┘                          │
│  Адаптировался                                                      │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Реализация Backpressure с asyncio

```python
import asyncio
from typing import Callable, Any
from dataclasses import dataclass

@dataclass
class BackpressureConfig:
    """Конфигурация backpressure"""
    queue_size: int = 100         # Размер буфера
    high_watermark: int = 80      # Порог замедления producer
    low_watermark: int = 20       # Порог возобновления

class BackpressureQueue:
    """
    Очередь с поддержкой backpressure.
    Producer блокируется, когда очередь заполнена.
    """

    def __init__(self, config: BackpressureConfig):
        self.config = config
        self._queue: asyncio.Queue = asyncio.Queue(maxsize=config.queue_size)
        self._paused = asyncio.Event()
        self._paused.set()  # Изначально не на паузе

    async def put(self, item: Any):
        """
        Добавление элемента с учётом backpressure.
        Блокируется, если очередь переполнена.
        """
        # Ждём, пока не будет сигнала продолжать
        await self._paused.wait()

        await self._queue.put(item)

        # Проверяем, нужно ли приостановить producer
        if self._queue.qsize() >= self.config.high_watermark:
            self._paused.clear()  # Приостанавливаем producer

    async def get(self) -> Any:
        """Получение элемента из очереди"""
        item = await self._queue.get()

        # Проверяем, можно ли возобновить producer
        if self._queue.qsize() <= self.config.low_watermark:
            self._paused.set()  # Возобновляем producer

        return item

    @property
    def is_producer_paused(self) -> bool:
        """Приостановлен ли producer"""
        return not self._paused.is_set()


# Пример использования
async def producer(queue: BackpressureQueue, name: str):
    """Производитель данных"""
    for i in range(1000):
        await queue.put(f"{name}-item-{i}")
        # Автоматически замедляется при переполнении очереди

async def consumer(queue: BackpressureQueue, name: str):
    """Медленный потребитель данных"""
    while True:
        item = await queue.get()
        # Медленная обработка
        await asyncio.sleep(0.1)
        print(f"{name} обработал: {item}")


async def main():
    config = BackpressureConfig(queue_size=50, high_watermark=40, low_watermark=10)
    queue = BackpressureQueue(config)

    # Запускаем producer и consumer
    await asyncio.gather(
        producer(queue, "fast-producer"),
        consumer(queue, "slow-consumer")
    )
```

---

## 5. Circuit Breaker (Автоматический выключатель)

### Диаграмма состояний

```
┌─────────────────────────────────────────────────────────────────────┐
│                   CIRCUIT BREAKER STATES                            │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│                      ┌─────────────────┐                            │
│                      │     CLOSED      │                            │
│                      │  (Нормальная    │                            │
│     Успех            │   работа)       │         Порог ошибок      │
│    ┌────────────────►│                 │─────────────────┐          │
│    │                 └────────┬────────┘                 │          │
│    │                          │                          ▼          │
│    │                          │              ┌─────────────────┐    │
│    │                          │              │      OPEN       │    │
│    │                          │              │  (Все запросы   │    │
│    │                          │              │  отклоняются)   │    │
│    │                          │              └────────┬────────┘    │
│    │                          │                       │             │
│    │                          │         Таймаут       │             │
│    │                          │         истёк         │             │
│    │                          │                       ▼             │
│    │                 ┌────────┴────────┐    ┌─────────────────┐    │
│    │                 │   HALF-OPEN     │◄───│                 │    │
│    │                 │ (Пробный режим) │    │                 │    │
│    │                 └────────┬────────┘    └─────────────────┘    │
│    │                          │                       ▲             │
│    │        Успех             │          Ошибка       │             │
│    └──────────────────────────┘          ────────────►│             │
│                                                                     │
│  Параметры:                                                         │
│  • failure_threshold: 5 ошибок → открытие                          │
│  • timeout: 30 сек в открытом состоянии                            │
│  • success_threshold: 3 успеха в half-open → закрытие              │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Реализация на Python

```python
import time
import threading
from enum import Enum
from typing import Callable, Any, Optional
from dataclasses import dataclass, field
from functools import wraps

class CircuitState(Enum):
    """Состояния Circuit Breaker"""
    CLOSED = "closed"       # Нормальная работа
    OPEN = "open"           # Запросы блокируются
    HALF_OPEN = "half_open" # Пробный режим

@dataclass
class CircuitBreakerConfig:
    """Конфигурация Circuit Breaker"""
    failure_threshold: int = 5        # Порог ошибок для открытия
    success_threshold: int = 3        # Успехов для закрытия из half-open
    timeout: float = 30.0             # Время в открытом состоянии (сек)
    expected_exceptions: tuple = (Exception,)  # Какие исключения считать ошибками

@dataclass
class CircuitBreakerStats:
    """Статистика Circuit Breaker"""
    failures: int = 0
    successes: int = 0
    last_failure_time: Optional[float] = None
    total_calls: int = 0
    total_failures: int = 0

class CircuitBreakerOpenError(Exception):
    """Исключение при открытом Circuit Breaker"""
    def __init__(self, circuit_name: str, retry_after: float):
        self.circuit_name = circuit_name
        self.retry_after = retry_after
        super().__init__(f"Circuit '{circuit_name}' is open. Retry after {retry_after:.1f}s")

class CircuitBreaker:
    """
    Реализация паттерна Circuit Breaker.

    Защищает от каскадных сбоев при отказе зависимостей.
    """

    def __init__(self, name: str, config: CircuitBreakerConfig = None):
        self.name = name
        self.config = config or CircuitBreakerConfig()
        self._state = CircuitState.CLOSED
        self._stats = CircuitBreakerStats()
        self._lock = threading.Lock()

    @property
    def state(self) -> CircuitState:
        """Текущее состояние с учётом таймаута"""
        with self._lock:
            if self._state == CircuitState.OPEN:
                # Проверяем, не пора ли перейти в half-open
                if self._time_since_last_failure() >= self.config.timeout:
                    self._state = CircuitState.HALF_OPEN
                    self._stats.successes = 0
            return self._state

    def _time_since_last_failure(self) -> float:
        """Время с последней ошибки"""
        if self._stats.last_failure_time is None:
            return float('inf')
        return time.monotonic() - self._stats.last_failure_time

    def _record_success(self):
        """Регистрация успешного вызова"""
        with self._lock:
            self._stats.successes += 1
            self._stats.total_calls += 1

            if self._state == CircuitState.HALF_OPEN:
                if self._stats.successes >= self.config.success_threshold:
                    # Достаточно успехов — закрываем circuit
                    self._state = CircuitState.CLOSED
                    self._stats.failures = 0

    def _record_failure(self):
        """Регистрация неудачного вызова"""
        with self._lock:
            self._stats.failures += 1
            self._stats.total_failures += 1
            self._stats.total_calls += 1
            self._stats.last_failure_time = time.monotonic()

            if self._state == CircuitState.HALF_OPEN:
                # Одна ошибка в half-open — снова открываем
                self._state = CircuitState.OPEN
            elif self._state == CircuitState.CLOSED:
                if self._stats.failures >= self.config.failure_threshold:
                    # Достигнут порог ошибок — открываем circuit
                    self._state = CircuitState.OPEN

    def call(self, func: Callable, *args, **kwargs) -> Any:
        """Выполнение вызова через Circuit Breaker"""
        current_state = self.state

        if current_state == CircuitState.OPEN:
            retry_after = self.config.timeout - self._time_since_last_failure()
            raise CircuitBreakerOpenError(self.name, retry_after)

        try:
            result = func(*args, **kwargs)
            self._record_success()
            return result
        except self.config.expected_exceptions as e:
            self._record_failure()
            raise

    def __call__(self, func: Callable) -> Callable:
        """Декоратор для функций"""
        @wraps(func)
        def wrapper(*args, **kwargs):
            return self.call(func, *args, **kwargs)
        return wrapper


# Пример использования
payment_circuit = CircuitBreaker(
    "payment_service",
    CircuitBreakerConfig(
        failure_threshold=3,
        timeout=60.0,
        expected_exceptions=(ConnectionError, TimeoutError)
    )
)

@payment_circuit
def process_payment(amount: float) -> dict:
    """Обработка платежа с защитой Circuit Breaker"""
    # Вызов внешнего платёжного сервиса
    response = external_payment_api.charge(amount)
    return response
```

### Использование resilience4j (Java)

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;

import java.time.Duration;
import java.util.function.Supplier;

public class PaymentService {

    private final CircuitBreaker circuitBreaker;

    public PaymentService() {
        // Конфигурация Circuit Breaker
        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)                // 50% ошибок → открытие
            .slowCallRateThreshold(50)               // 50% медленных вызовов → открытие
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(30))
            .permittedNumberOfCallsInHalfOpenState(3)
            .minimumNumberOfCalls(5)                 // Минимум вызовов для расчёта
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(10)
            .build();

        CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(config);
        this.circuitBreaker = registry.circuitBreaker("paymentService");

        // Слушатели событий
        circuitBreaker.getEventPublisher()
            .onStateTransition(event ->
                System.out.println("Circuit state: " + event.getStateTransition()));
    }

    public PaymentResult processPayment(PaymentRequest request) {
        // Декорируем вызов Circuit Breaker'ом
        Supplier<PaymentResult> decoratedSupplier = CircuitBreaker
            .decorateSupplier(circuitBreaker, () -> callPaymentApi(request));

        try {
            return decoratedSupplier.get();
        } catch (Exception e) {
            // Fallback при открытом circuit
            return new PaymentResult(false, "Сервис временно недоступен");
        }
    }

    private PaymentResult callPaymentApi(PaymentRequest request) {
        // Реальный вызов API
        return externalApi.process(request);
    }
}
```

---

## 6. Load Shedding (Сброс нагрузки)

### Концепция

Load Shedding — контролируемый отказ от части запросов для сохранения работоспособности системы.

```
┌─────────────────────────────────────────────────────────────────────┐
│                       LOAD SHEDDING                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  Нагрузка   │                                                       │
│  (req/s)    │                    ████████                           │
│             │                   ██████████  ← Пиковая нагрузка      │
│     1000    ├──────────────────██████████████──────────────────     │
│             │                 ████████████████                      │
│             │    ████████████████████████████████████               │
│             │  ████████████████████████████████████████             │
│             │████████████████████████████████████████████           │
│      500    ├═══════════════════════════════════════════════════    │
│             │            ↑ Максимальная пропускная способность      │
│             │                                                       │
│             └──────────────────────────────────────────────────►    │
│                               Время                                 │
│                                                                     │
│  Стратегия: отбрасываем запросы ВЫШЕ линии 500 req/s               │
│                                                                     │
│  Приоритеты:                                                        │
│  1. Premium пользователи — никогда не отбрасываем                  │
│  2. Аутентифицированные — отбрасываем в последнюю очередь          │
│  3. Анонимные — отбрасываем первыми                                │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Реализация Load Shedding

```python
import time
import random
from typing import Callable, Any, Optional
from dataclasses import dataclass
from enum import IntEnum

class RequestPriority(IntEnum):
    """Приоритеты запросов"""
    CRITICAL = 0      # Системные, healthcheck
    PREMIUM = 1       # Платные пользователи
    AUTHENTICATED = 2 # Авторизованные
    ANONYMOUS = 3     # Анонимные

@dataclass
class LoadMetrics:
    """Метрики нагрузки системы"""
    current_rps: float
    cpu_percent: float
    memory_percent: float
    queue_size: int

class LoadShedder:
    """
    Механизм сброса нагрузки на основе текущих метрик системы.
    """

    def __init__(
        self,
        max_rps: float = 1000,
        cpu_threshold: float = 80,
        memory_threshold: float = 85
    ):
        self.max_rps = max_rps
        self.cpu_threshold = cpu_threshold
        self.memory_threshold = memory_threshold
        self._metrics = LoadMetrics(0, 0, 0, 0)

    def update_metrics(self, metrics: LoadMetrics):
        """Обновление текущих метрик"""
        self._metrics = metrics

    def should_shed(self, priority: RequestPriority) -> bool:
        """
        Определяет, нужно ли отбросить запрос.

        Returns:
            True если запрос должен быть отброшен
        """
        # Критические запросы никогда не отбрасываем
        if priority == RequestPriority.CRITICAL:
            return False

        # Рассчитываем уровень перегрузки (0.0 - 1.0+)
        overload = max(
            self._metrics.current_rps / self.max_rps,
            self._metrics.cpu_percent / self.cpu_threshold,
            self._metrics.memory_percent / self.memory_threshold
        )

        if overload <= 1.0:
            # Нет перегрузки
            return False

        # Вероятность сброса зависит от приоритета и уровня перегрузки
        shed_probability = self._calculate_shed_probability(priority, overload)

        return random.random() < shed_probability

    def _calculate_shed_probability(
        self,
        priority: RequestPriority,
        overload: float
    ) -> float:
        """
        Рассчитывает вероятность сброса запроса.

        Чем выше приоритет (меньше число), тем меньше вероятность сброса.
        """
        base_probability = min(1.0, overload - 1.0)  # 0-100%

        # Множители для разных приоритетов
        priority_multipliers = {
            RequestPriority.CRITICAL: 0.0,      # Никогда
            RequestPriority.PREMIUM: 0.1,       # Очень редко
            RequestPriority.AUTHENTICATED: 0.5, # Половина
            RequestPriority.ANONYMOUS: 1.0,     # Полностью
        }

        return base_probability * priority_multipliers[priority]


# Middleware для FastAPI
from fastapi import FastAPI, Request, HTTPException

app = FastAPI()
load_shedder = LoadShedder(max_rps=500)

@app.middleware("http")
async def load_shedding_middleware(request: Request, call_next):
    """Middleware для сброса нагрузки"""

    # Определяем приоритет запроса
    priority = get_request_priority(request)

    # Проверяем, нужно ли отбросить запрос
    if load_shedder.should_shed(priority):
        raise HTTPException(
            status_code=503,
            detail="Service overloaded. Please retry later.",
            headers={"Retry-After": "30"}
        )

    return await call_next(request)

def get_request_priority(request: Request) -> RequestPriority:
    """Определение приоритета запроса"""
    # Healthcheck — критический
    if request.url.path in ["/health", "/ready"]:
        return RequestPriority.CRITICAL

    # Проверяем токен пользователя
    user = getattr(request.state, "user", None)
    if user:
        if user.is_premium:
            return RequestPriority.PREMIUM
        return RequestPriority.AUTHENTICATED

    return RequestPriority.ANONYMOUS
```

---

## 7. Retry с Exponential Backoff

### Стратегии повторных попыток

```
┌─────────────────────────────────────────────────────────────────────┐
│                    RETRY STRATEGIES                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                     │
│  IMMEDIATE RETRY (плохо):                                          │
│  Попытка 1 → Попытка 2 → Попытка 3 → Попытка 4 → ...              │
│  ──●──●──●──●──●──●──●──●──●──●──► Перегружает сервис!            │
│                                                                     │
│  FIXED INTERVAL (лучше):                                           │
│  Попытка 1    Попытка 2    Попытка 3    Попытка 4                  │
│  ──●──────────●──────────●──────────●──────────►                   │
│     1 сек        1 сек       1 сек                                  │
│                                                                     │
│  EXPONENTIAL BACKOFF (рекомендуется):                              │
│  Попытка 1  Попытка 2      Попытка 3              Попытка 4        │
│  ──●────────●──────────────●──────────────────────●─────────►      │
│     1 сек       2 сек             4 сек               8 сек        │
│                                                                     │
│  EXPONENTIAL BACKOFF + JITTER (идеально):                          │
│  ──●─────────●────────────────●─────────────────────────●───►      │
│    1.2 сек     2.7 сек           5.1 сек             9.4 сек       │
│     ↑           ↑                   ↑                    ↑          │
│   +jitter    +jitter            +jitter              +jitter        │
│                                                                     │
│  Jitter предотвращает "thundering herd" — когда все клиенты        │
│  повторяют запросы одновременно после сбоя                         │
│                                                                     │
└─────────────────────────────────────────────────────────────────────┘
```

### Реализация на Python

```python
import time
import random
import asyncio
from typing import Callable, TypeVar, Any, Tuple
from dataclasses import dataclass
from functools import wraps

T = TypeVar('T')

@dataclass
class RetryConfig:
    """Конфигурация повторных попыток"""
    max_attempts: int = 5
    base_delay: float = 1.0           # Начальная задержка (сек)
    max_delay: float = 60.0           # Максимальная задержка
    exponential_base: float = 2.0     # База экспоненты
    jitter: bool = True               # Добавлять случайность
    retryable_exceptions: Tuple = (Exception,)

class RetryError(Exception):
    """Исключение после исчерпания попыток"""
    def __init__(self, message: str, last_exception: Exception):
        super().__init__(message)
        self.last_exception = last_exception

def calculate_delay(
    attempt: int,
    config: RetryConfig
) -> float:
    """
    Расчёт задержки перед следующей попыткой.

    Формула: min(max_delay, base_delay * (exponential_base ^ attempt))
    С jitter: delay * random(0.5, 1.5)
    """
    delay = min(
        config.max_delay,
        config.base_delay * (config.exponential_base ** attempt)
    )

    if config.jitter:
        # Full jitter: равномерное распределение от 0 до delay
        delay = random.uniform(0, delay)
        # Или decorrelated jitter: delay * random(0.5, 1.5)
        # delay = delay * random.uniform(0.5, 1.5)

    return delay


def retry_sync(config: RetryConfig = None):
    """Синхронный декоратор retry с exponential backoff"""
    config = config or RetryConfig()

    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @wraps(func)
        def wrapper(*args, **kwargs) -> T:
            last_exception = None

            for attempt in range(config.max_attempts):
                try:
                    return func(*args, **kwargs)
                except config.retryable_exceptions as e:
                    last_exception = e

                    if attempt < config.max_attempts - 1:
                        delay = calculate_delay(attempt, config)
                        print(f"Попытка {attempt + 1} не удалась. "
                              f"Повтор через {delay:.2f} сек...")
                        time.sleep(delay)

            raise RetryError(
                f"Все {config.max_attempts} попыток исчерпаны",
                last_exception
            )
        return wrapper
    return decorator


def retry_async(config: RetryConfig = None):
    """Асинхронный декоратор retry с exponential backoff"""
    config = config or RetryConfig()

    def decorator(func: Callable[..., T]) -> Callable[..., T]:
        @wraps(func)
        async def wrapper(*args, **kwargs) -> T:
            last_exception = None

            for attempt in range(config.max_attempts):
                try:
                    return await func(*args, **kwargs)
                except config.retryable_exceptions as e:
                    last_exception = e

                    if attempt < config.max_attempts - 1:
                        delay = calculate_delay(attempt, config)
                        await asyncio.sleep(delay)

            raise RetryError(
                f"Все {config.max_attempts} попыток исчерпаны",
                last_exception
            )
        return wrapper
    return decorator


# Примеры использования
@retry_sync(RetryConfig(
    max_attempts=5,
    base_delay=1.0,
    retryable_exceptions=(ConnectionError, TimeoutError)
))
def fetch_data_from_api(url: str) -> dict:
    """Получение данных с автоматическими повторами"""
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    return response.json()


@retry_async(RetryConfig(
    max_attempts=3,
    base_delay=0.5,
    max_delay=10.0
))
async def send_notification(user_id: str, message: str):
    """Отправка уведомления с повторами"""
    async with aiohttp.ClientSession() as session:
        async with session.post(
            "https://notifications.example.com/send",
            json={"user_id": user_id, "message": message}
        ) as response:
            return await response.json()
```

---

## Best Practices

### 1. Комбинирование стратегий

```python
class ResilientService:
    """Сервис с комбинацией всех стратегий защиты"""

    def __init__(self):
        # Circuit Breaker для защиты от каскадных сбоев
        self.circuit_breaker = CircuitBreaker("external_api")

        # Rate Limiter для защиты от перегрузки
        self.rate_limiter = SlidingWindowRateLimiter(100, 60)

        # Retry для обработки временных сбоев
        self.retry_config = RetryConfig(max_attempts=3, base_delay=1.0)

    async def call_external_service(self, request: dict) -> dict:
        """Защищённый вызов внешнего сервиса"""

        # 1. Проверяем rate limit
        if not self.rate_limiter.allow_request():
            raise RateLimitError("Превышен лимит запросов")

        # 2. Проверяем circuit breaker
        if self.circuit_breaker.state == CircuitState.OPEN:
            return self._fallback_response(request)

        # 3. Выполняем с retry
        @retry_async(self.retry_config)
        async def _call():
            return await self.circuit_breaker.call(
                self._actual_api_call, request
            )

        try:
            return await _call()
        except RetryError:
            return self._fallback_response(request)

    def _fallback_response(self, request: dict) -> dict:
        """Fallback ответ при недоступности сервиса"""
        return {"status": "degraded", "data": self._get_cached_data(request)}
```

### 2. Мониторинг и алерты

```python
# Метрики для Prometheus
from prometheus_client import Counter, Gauge, Histogram

circuit_breaker_state = Gauge(
    'circuit_breaker_state',
    'Current state of circuit breaker',
    ['service']
)

rate_limit_exceeded = Counter(
    'rate_limit_exceeded_total',
    'Total rate limit exceeded events',
    ['endpoint', 'user_tier']
)

request_latency = Histogram(
    'request_latency_seconds',
    'Request latency with retry attempts',
    ['service', 'attempt']
)
```

---

## Частые ошибки

1. **Retry без backoff** — создаёт дополнительную нагрузку на уже перегруженный сервис

2. **Одинаковый retry интервал** — приводит к thundering herd эффекту

3. **Circuit Breaker с слишком низким порогом** — часто открывается при нормальной работе

4. **Rate limiting только на application уровне** — не защищает от DDoS

5. **Отсутствие fallback** — система падает полностью вместо частичной деградации

6. **Игнорирование приоритетов** — важные запросы отклоняются наравне с неважными

---

## Резюме

| Стратегия | Назначение | Когда использовать |
|-----------|------------|-------------------|
| **Graceful Degradation** | Частичная работа вместо полного отказа | Зависимость от нескольких сервисов |
| **Throttling** | Ограничение скорости обработки | Защита от перегрузки |
| **Rate Limiting** | Ограничение количества запросов | API, защита от abuse |
| **Backpressure** | Сигнал замедления producer'у | Асинхронные системы, очереди |
| **Circuit Breaker** | Защита от каскадных сбоев | Вызовы внешних сервисов |
| **Load Shedding** | Контролируемый отказ от запросов | Пиковые нагрузки |
| **Retry + Backoff** | Обработка временных сбоев | Сетевые операции |

Все эти стратегии должны использоваться вместе для построения по-настоящему устойчивых систем.
