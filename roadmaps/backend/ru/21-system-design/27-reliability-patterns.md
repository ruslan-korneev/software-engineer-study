# Паттерны надёжности (Reliability Patterns)

[prev: 26-cloud-design-patterns](./26-cloud-design-patterns.md) | [next: 28-security-basics](./28-security-basics.md)

---

## Введение

Паттерны надёжности — это архитектурные решения, которые помогают системам оставаться работоспособными даже при частичных отказах. В распределённых системах сбои неизбежны: сети ненадёжны, сервисы падают, базы данных становятся недоступными. Задача инженера — проектировать системы, которые gracefully degraded (мягко деградируют) вместо полного отказа.

Основные принципы:
- **Fail fast** — быстро обнаруживать проблемы
- **Fail safe** — минимизировать последствия отказов
- **Fail gracefully** — продолжать работу с ограниченной функциональностью

---

## Circuit Breaker (Автоматический выключатель)

### Концепция

Circuit Breaker предотвращает каскадные отказы, "размыкая цепь" при обнаружении проблем с зависимым сервисом. Паттерн назван по аналогии с электрическим автоматом.

### Состояния

```
┌─────────┐    Успех     ┌──────────┐
│  CLOSED │◄────────────│ HALF-OPEN │
│(Работает)│             │(Проверка) │
└────┬────┘             └─────┬─────┘
     │                        │
     │ Порог ошибок          │ Ошибка
     │ превышен              │
     ▼                        │
┌─────────┐    Таймаут       │
│  OPEN   │──────────────────┘
│(Разомкн)│
└─────────┘
```

**CLOSED (Замкнут):** Нормальная работа, запросы проходят.

**OPEN (Разомкнут):** Запросы немедленно отклоняются без попытки вызова сервиса.

**HALF-OPEN (Полуоткрыт):** Пропускается ограниченное количество тестовых запросов для проверки восстановления.

### Реализация на Python

```python
import time
from enum import Enum
from threading import Lock
from functools import wraps
from typing import Callable, Optional
import logging

logger = logging.getLogger(__name__)


class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"


class CircuitBreaker:
    def __init__(
        self,
        failure_threshold: int = 5,
        success_threshold: int = 2,
        timeout: float = 30.0,
        expected_exceptions: tuple = (Exception,)
    ):
        self.failure_threshold = failure_threshold
        self.success_threshold = success_threshold
        self.timeout = timeout
        self.expected_exceptions = expected_exceptions

        self._state = CircuitState.CLOSED
        self._failure_count = 0
        self._success_count = 0
        self._last_failure_time: Optional[float] = None
        self._lock = Lock()

    @property
    def state(self) -> CircuitState:
        with self._lock:
            if self._state == CircuitState.OPEN:
                if time.time() - self._last_failure_time >= self.timeout:
                    self._state = CircuitState.HALF_OPEN
                    self._success_count = 0
                    logger.info("Circuit breaker transitioning to HALF_OPEN")
            return self._state

    def _handle_success(self):
        with self._lock:
            if self._state == CircuitState.HALF_OPEN:
                self._success_count += 1
                if self._success_count >= self.success_threshold:
                    self._state = CircuitState.CLOSED
                    self._failure_count = 0
                    logger.info("Circuit breaker CLOSED")
            elif self._state == CircuitState.CLOSED:
                self._failure_count = 0

    def _handle_failure(self):
        with self._lock:
            self._failure_count += 1
            self._last_failure_time = time.time()

            if self._state == CircuitState.HALF_OPEN:
                self._state = CircuitState.OPEN
                logger.warning("Circuit breaker OPEN (from half-open)")
            elif self._failure_count >= self.failure_threshold:
                self._state = CircuitState.OPEN
                logger.warning(f"Circuit breaker OPEN after {self._failure_count} failures")

    def call(self, func: Callable, *args, **kwargs):
        if self.state == CircuitState.OPEN:
            raise CircuitBreakerOpenError("Circuit breaker is OPEN")

        try:
            result = func(*args, **kwargs)
            self._handle_success()
            return result
        except self.expected_exceptions as e:
            self._handle_failure()
            raise

    def __call__(self, func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            return self.call(func, *args, **kwargs)
        return wrapper


class CircuitBreakerOpenError(Exception):
    pass


# Использование как декоратор
payment_circuit = CircuitBreaker(
    failure_threshold=3,
    timeout=60.0,
    expected_exceptions=(ConnectionError, TimeoutError)
)


@payment_circuit
def process_payment(amount: float, card_token: str) -> dict:
    """Обработка платежа через внешний сервис."""
    response = external_payment_api.charge(amount, card_token)
    return response


# Использование с fallback
def process_payment_safe(amount: float, card_token: str) -> dict:
    try:
        return process_payment(amount, card_token)
    except CircuitBreakerOpenError:
        # Fallback: поставить в очередь для повторной обработки
        queue_payment_for_retry(amount, card_token)
        return {"status": "queued", "message": "Payment will be processed later"}
```

### Библиотеки

- **Python:** `pybreaker`, `circuitbreaker`, `tenacity`
- **Java:** Resilience4j, Hystrix (deprecated)
- **Go:** `sony/gobreaker`

---

## Retry Pattern (Повторные попытки)

### Концепция

Retry паттерн автоматически повторяет неудачные операции, что помогает справляться с временными (transient) сбоями: сетевыми задержками, кратковременной недоступностью сервисов.

### Стратегии повторов

```
1. Fixed Delay (Фиксированная задержка)
   ────●────●────●────●────●────
       1s   1s   1s   1s   1s

2. Exponential Backoff (Экспоненциальная)
   ●──●────●────────●────────────────●
   1s 2s   4s       8s              16s

3. Exponential Backoff with Jitter (С джиттером)
   ●──●────●────────●────────────────●
   1s 2.3s 3.7s     9.2s            14.8s
   (случайное отклонение от базового времени)
```

### Реализация

```python
import time
import random
from functools import wraps
from typing import Callable, Type, Tuple, Optional
import logging

logger = logging.getLogger(__name__)


def retry(
    max_attempts: int = 3,
    delay: float = 1.0,
    backoff: float = 2.0,
    jitter: bool = True,
    exceptions: Tuple[Type[Exception], ...] = (Exception,),
    on_retry: Optional[Callable] = None
):
    """
    Декоратор для повторных попыток с экспоненциальной задержкой.

    Args:
        max_attempts: Максимальное количество попыток
        delay: Начальная задержка в секундах
        backoff: Множитель для экспоненциального роста
        jitter: Добавлять случайное отклонение
        exceptions: Типы исключений для перехвата
        on_retry: Callback функция при каждой попытке
    """
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            last_exception = None
            current_delay = delay

            for attempt in range(1, max_attempts + 1):
                try:
                    return func(*args, **kwargs)
                except exceptions as e:
                    last_exception = e

                    if attempt == max_attempts:
                        logger.error(f"All {max_attempts} attempts failed for {func.__name__}")
                        raise

                    # Вычисляем задержку
                    sleep_time = current_delay
                    if jitter:
                        # Добавляем случайное отклонение ±25%
                        sleep_time *= (0.75 + random.random() * 0.5)

                    logger.warning(
                        f"Attempt {attempt}/{max_attempts} failed for {func.__name__}: {e}. "
                        f"Retrying in {sleep_time:.2f}s"
                    )

                    if on_retry:
                        on_retry(attempt, e, sleep_time)

                    time.sleep(sleep_time)
                    current_delay *= backoff

            raise last_exception
        return wrapper
    return decorator


# Использование
@retry(max_attempts=5, delay=1.0, backoff=2.0, exceptions=(ConnectionError, TimeoutError))
def fetch_user_data(user_id: int) -> dict:
    response = requests.get(f"https://api.example.com/users/{user_id}", timeout=5)
    response.raise_for_status()
    return response.json()


# Продвинутый пример с tenacity
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
    before_sleep_log
)


@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=60),
    retry=retry_if_exception_type((ConnectionError, TimeoutError)),
    before_sleep=before_sleep_log(logger, logging.WARNING)
)
def reliable_api_call(endpoint: str) -> dict:
    return requests.get(endpoint, timeout=10).json()
```

### Важные соображения

```python
# НЕПРАВИЛЬНО: Повторять все ошибки
@retry(exceptions=(Exception,))  # НЕТ!
def dangerous_retry():
    pass

# ПРАВИЛЬНО: Повторять только transient ошибки
RETRYABLE_ERRORS = (
    ConnectionError,
    TimeoutError,
    requests.exceptions.ConnectionError,
    requests.exceptions.Timeout,
)

# НЕ повторять:
NON_RETRYABLE = (
    ValueError,              # Ошибка валидации
    AuthenticationError,     # Неверные credentials
    PermissionError,         # Нет прав доступа
    requests.exceptions.HTTPError,  # 4xx ошибки (кроме 429)
)


def is_retryable(exception: Exception) -> bool:
    """Определяет, стоит ли повторять запрос."""
    if isinstance(exception, requests.exceptions.HTTPError):
        status_code = exception.response.status_code
        # Повторять только 5xx и 429 (rate limit)
        return status_code >= 500 or status_code == 429
    return isinstance(exception, RETRYABLE_ERRORS)
```

---

## Timeout Pattern (Таймауты)

### Концепция

Таймауты предотвращают бесконечное ожидание ответа от зависимого сервиса. Без таймаутов один зависший сервис может заблокировать все ресурсы вызывающей стороны.

### Виды таймаутов

```
┌─────────────────────────────────────────────────────────┐
│                    Клиент                               │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Connection Timeout (время на установку соединения) │
│  │  ●────────────────────────────────────────────●     │
│  │  0s                                          5s    │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Read Timeout (время ожидания ответа)              │
│  │  ●────────────────────────────────────────────●     │
│  │  0s                                         30s    │
│  └─────────────────────────────────────────────────┘   │
│                                                         │
│  ┌─────────────────────────────────────────────────┐   │
│  │  Total/Request Timeout (общее время операции)      │
│  │  ●────────────────────────────────────────────●     │
│  │  0s                                         60s    │
│  └─────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────┘
```

### Реализация

```python
import asyncio
import signal
from concurrent.futures import ThreadPoolExecutor, TimeoutError as FuturesTimeoutError
from contextlib import contextmanager
from functools import wraps
from typing import Callable, TypeVar, Optional
import requests

T = TypeVar('T')


# 1. Таймаут для HTTP запросов
def fetch_with_timeout(url: str, timeout: float = 10.0) -> dict:
    """
    HTTP запрос с таймаутом.
    """
    try:
        response = requests.get(
            url,
            timeout=(3.0, timeout)  # (connection_timeout, read_timeout)
        )
        response.raise_for_status()
        return response.json()
    except requests.exceptions.Timeout:
        raise TimeoutError(f"Request to {url} timed out after {timeout}s")


# 2. Универсальный декоратор таймаута
def timeout(seconds: float):
    """Декоратор для ограничения времени выполнения функции."""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            with ThreadPoolExecutor(max_workers=1) as executor:
                future = executor.submit(func, *args, **kwargs)
                try:
                    return future.result(timeout=seconds)
                except FuturesTimeoutError:
                    raise TimeoutError(
                        f"Function {func.__name__} timed out after {seconds}s"
                    )
        return wrapper
    return decorator


@timeout(5.0)
def slow_operation():
    import time
    time.sleep(10)  # Будет прервано через 5 секунд
    return "Done"


# 3. Async таймаут
async def fetch_with_async_timeout(url: str, timeout_seconds: float = 10.0) -> dict:
    """Async HTTP запрос с таймаутом."""
    import aiohttp

    try:
        timeout = aiohttp.ClientTimeout(total=timeout_seconds)
        async with aiohttp.ClientSession(timeout=timeout) as session:
            async with session.get(url) as response:
                return await response.json()
    except asyncio.TimeoutError:
        raise TimeoutError(f"Async request to {url} timed out")


# 4. Каскадные таймауты (для цепочки вызовов)
class TimeoutBudget:
    """
    Бюджет таймаута для распределения между несколькими операциями.
    """
    def __init__(self, total_seconds: float):
        self.total = total_seconds
        self.start_time = time.time()

    @property
    def remaining(self) -> float:
        elapsed = time.time() - self.start_time
        return max(0, self.total - elapsed)

    @property
    def is_expired(self) -> bool:
        return self.remaining <= 0

    def get_timeout(self, default: float) -> float:
        """Возвращает меньшее из оставшегося бюджета и default."""
        return min(self.remaining, default)


def process_order(order_id: str, timeout_budget: TimeoutBudget) -> dict:
    """Обработка заказа с бюджетом таймаута."""
    if timeout_budget.is_expired:
        raise TimeoutError("Timeout budget exhausted before processing")

    # Каждая операция получает часть бюджета
    user = fetch_user(order_id, timeout=timeout_budget.get_timeout(5.0))

    if timeout_budget.is_expired:
        raise TimeoutError("Timeout budget exhausted after fetching user")

    inventory = check_inventory(order_id, timeout=timeout_budget.get_timeout(3.0))

    if timeout_budget.is_expired:
        raise TimeoutError("Timeout budget exhausted after checking inventory")

    payment = process_payment(order_id, timeout=timeout_budget.get_timeout(10.0))

    return {"user": user, "inventory": inventory, "payment": payment}


# Использование
budget = TimeoutBudget(total_seconds=30.0)
result = process_order("order-123", budget)
```

### Best Practices для таймаутов

```python
# 1. Всегда устанавливайте таймауты
# ПЛОХО
response = requests.get(url)  # Может висеть вечно!

# ХОРОШО
response = requests.get(url, timeout=10)


# 2. Разные таймауты для разных операций
TIMEOUTS = {
    "health_check": 2.0,      # Быстрая проверка
    "read_operation": 10.0,   # Чтение данных
    "write_operation": 30.0,  # Запись может быть медленнее
    "batch_operation": 120.0, # Пакетные операции
}


# 3. Учитывайте downstream таймауты
# Если вызываемый сервис имеет таймаут 30с,
# ваш таймаут должен быть чуть больше (35с)
# чтобы получить информативную ошибку вместо своего таймаута
```

---

## Bulkhead Pattern (Переборки)

### Концепция

Bulkhead изолирует ресурсы (потоки, соединения, память) для разных компонентов системы. Название происходит от переборок на кораблях, которые предотвращают затопление всего судна при пробоине в одном отсеке.

### Виды изоляции

```
┌───────────────────────────────────────────────────────────┐
│                      Приложение                            │
│                                                            │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐        │
│  │  Bulkhead A │  │  Bulkhead B │  │  Bulkhead C │        │
│  │ (Payments)  │  │  (Orders)   │  │   (Users)   │        │
│  │             │  │             │  │             │        │
│  │ ○○○○○       │  │ ○○○○○○○○    │  │ ○○○         │        │
│  │ 5 threads   │  │ 8 threads   │  │ 3 threads   │        │
│  │             │  │             │  │             │        │
│  │ ████░       │  │ ██████░░    │  │ ███         │        │
│  │ 10 conn     │  │ 20 conn     │  │ 5 conn      │        │
│  └─────────────┘  └─────────────┘  └─────────────┘        │
│                                                            │
│  Если Payment API упадёт, Orders и Users продолжат работу │
└───────────────────────────────────────────────────────────┘
```

### Реализация

```python
import asyncio
from concurrent.futures import ThreadPoolExecutor, Semaphore
from contextlib import contextmanager
from dataclasses import dataclass
from typing import Callable, Dict, Optional
import threading
import time


@dataclass
class BulkheadConfig:
    max_concurrent: int
    max_wait_time: float = 10.0
    name: str = "default"


class Bulkhead:
    """
    Изоляция ресурсов с помощью семафора.
    """
    def __init__(self, config: BulkheadConfig):
        self.config = config
        self._semaphore = Semaphore(config.max_concurrent)
        self._active_calls = 0
        self._rejected_calls = 0
        self._lock = threading.Lock()

    @property
    def stats(self) -> dict:
        with self._lock:
            return {
                "name": self.config.name,
                "max_concurrent": self.config.max_concurrent,
                "active_calls": self._active_calls,
                "rejected_calls": self._rejected_calls,
                "available_permits": self.config.max_concurrent - self._active_calls,
            }

    @contextmanager
    def acquire(self):
        """Контекстный менеджер для захвата слота."""
        acquired = self._semaphore.acquire(timeout=self.config.max_wait_time)

        if not acquired:
            with self._lock:
                self._rejected_calls += 1
            raise BulkheadFullError(
                f"Bulkhead '{self.config.name}' is full. "
                f"Max concurrent: {self.config.max_concurrent}"
            )

        with self._lock:
            self._active_calls += 1

        try:
            yield
        finally:
            self._semaphore.release()
            with self._lock:
                self._active_calls -= 1

    def __call__(self, func: Callable) -> Callable:
        """Использование как декоратор."""
        @wraps(func)
        def wrapper(*args, **kwargs):
            with self.acquire():
                return func(*args, **kwargs)
        return wrapper


class BulkheadFullError(Exception):
    pass


# Создаём изолированные пулы для разных сервисов
class ServiceClients:
    def __init__(self):
        self.payment_bulkhead = Bulkhead(BulkheadConfig(
            name="payment",
            max_concurrent=5,
            max_wait_time=5.0
        ))

        self.inventory_bulkhead = Bulkhead(BulkheadConfig(
            name="inventory",
            max_concurrent=10,
            max_wait_time=3.0
        ))

        self.notification_bulkhead = Bulkhead(BulkheadConfig(
            name="notification",
            max_concurrent=20,
            max_wait_time=1.0
        ))

    def process_payment(self, order_id: str, amount: float) -> dict:
        with self.payment_bulkhead.acquire():
            # Критический путь — ограниченное количество слотов
            return payment_api.charge(order_id, amount)

    def check_inventory(self, product_ids: list) -> dict:
        with self.inventory_bulkhead.acquire():
            return inventory_api.check(product_ids)

    def send_notification(self, user_id: str, message: str):
        with self.notification_bulkhead.acquire():
            # Некритичная операция — больше слотов
            notification_api.send(user_id, message)


# Thread Pool Bulkhead
class ThreadPoolBulkhead:
    """
    Изоляция с использованием отдельных пулов потоков.
    """
    def __init__(self, pools_config: Dict[str, int]):
        self._pools: Dict[str, ThreadPoolExecutor] = {}
        for name, max_workers in pools_config.items():
            self._pools[name] = ThreadPoolExecutor(
                max_workers=max_workers,
                thread_name_prefix=f"bulkhead-{name}"
            )

    def submit(self, pool_name: str, func: Callable, *args, **kwargs):
        if pool_name not in self._pools:
            raise ValueError(f"Unknown pool: {pool_name}")
        return self._pools[pool_name].submit(func, *args, **kwargs)

    def shutdown(self, wait: bool = True):
        for pool in self._pools.values():
            pool.shutdown(wait=wait)


# Использование
pools = ThreadPoolBulkhead({
    "critical": 5,      # Критические операции
    "standard": 20,     # Обычные операции
    "background": 50,   # Фоновые задачи
})

# Критические операции не могут занять все потоки
future1 = pools.submit("critical", process_payment, order_id=123)
future2 = pools.submit("standard", fetch_user_data, user_id=456)
future3 = pools.submit("background", send_analytics, event="purchase")
```

### Async Bulkhead

```python
import asyncio
from typing import TypeVar, Callable, Awaitable

T = TypeVar('T')


class AsyncBulkhead:
    """Async версия Bulkhead с использованием asyncio.Semaphore."""

    def __init__(self, max_concurrent: int, name: str = "default"):
        self.max_concurrent = max_concurrent
        self.name = name
        self._semaphore = asyncio.Semaphore(max_concurrent)
        self._active = 0

    async def execute(self, coro: Awaitable[T]) -> T:
        async with self._semaphore:
            self._active += 1
            try:
                return await coro
            finally:
                self._active -= 1

    def __call__(self, func: Callable[..., Awaitable[T]]) -> Callable[..., Awaitable[T]]:
        @wraps(func)
        async def wrapper(*args, **kwargs):
            return await self.execute(func(*args, **kwargs))
        return wrapper


# Использование
payment_bulkhead = AsyncBulkhead(max_concurrent=5, name="payment")
inventory_bulkhead = AsyncBulkhead(max_concurrent=10, name="inventory")


@payment_bulkhead
async def process_payment_async(order_id: str) -> dict:
    async with aiohttp.ClientSession() as session:
        async with session.post(f"/payments/{order_id}") as resp:
            return await resp.json()
```

---

## Fallback Pattern (Запасной вариант)

### Концепция

Fallback предоставляет альтернативное поведение, когда основная операция недоступна. Это позволяет системе продолжать работу с деградированной, но приемлемой функциональностью.

### Стратегии Fallback

```
┌─────────────────────────────────────────────────────────┐
│                  Fallback Strategies                     │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Cache Fallback                                       │
│     Primary: API Call → Failed                          │
│     Fallback: Return cached data (stale but available)  │
│                                                          │
│  2. Default Value                                        │
│     Primary: Get user preferences → Failed              │
│     Fallback: Return default preferences                │
│                                                          │
│  3. Alternative Service                                  │
│     Primary: Payment via Stripe → Failed                │
│     Fallback: Payment via PayPal                        │
│                                                          │
│  4. Graceful Degradation                                 │
│     Primary: Full product recommendations → Failed      │
│     Fallback: Show popular products instead             │
│                                                          │
│  5. Queue for Later                                      │
│     Primary: Send email now → Failed                    │
│     Fallback: Queue for retry                           │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Реализация

```python
from typing import Callable, TypeVar, Optional, Generic, Any
from functools import wraps
import logging

T = TypeVar('T')
logger = logging.getLogger(__name__)


class FallbackHandler(Generic[T]):
    """
    Обработчик fallback логики с цепочкой запасных вариантов.
    """
    def __init__(self, primary: Callable[..., T]):
        self.primary = primary
        self.fallbacks: list[Callable[..., T]] = []
        self.on_fallback: Optional[Callable] = None

    def add_fallback(self, fallback: Callable[..., T]) -> 'FallbackHandler[T]':
        self.fallbacks.append(fallback)
        return self

    def with_callback(self, callback: Callable) -> 'FallbackHandler[T]':
        self.on_fallback = callback
        return self

    def execute(self, *args, **kwargs) -> T:
        # Пробуем основной метод
        try:
            return self.primary(*args, **kwargs)
        except Exception as primary_error:
            logger.warning(f"Primary method failed: {primary_error}")

            # Пробуем fallback'и по очереди
            for i, fallback in enumerate(self.fallbacks):
                try:
                    result = fallback(*args, **kwargs)
                    if self.on_fallback:
                        self.on_fallback(i, fallback.__name__, primary_error)
                    return result
                except Exception as fallback_error:
                    logger.warning(f"Fallback {i} failed: {fallback_error}")
                    continue

            # Все варианты исчерпаны
            raise RuntimeError(
                f"All fallbacks exhausted. Primary error: {primary_error}"
            )


# Декоратор для простых случаев
def with_fallback(
    fallback_value: Any = None,
    fallback_func: Optional[Callable] = None,
    exceptions: tuple = (Exception,)
):
    """Декоратор с fallback значением или функцией."""
    def decorator(func: Callable) -> Callable:
        @wraps(func)
        def wrapper(*args, **kwargs):
            try:
                return func(*args, **kwargs)
            except exceptions as e:
                logger.warning(f"{func.__name__} failed: {e}. Using fallback.")
                if fallback_func:
                    return fallback_func(*args, **kwargs)
                return fallback_value
        return wrapper
    return decorator


# Практические примеры

# 1. Cache Fallback
class ProductService:
    def __init__(self, api_client, cache):
        self.api = api_client
        self.cache = cache

    def get_product(self, product_id: str) -> dict:
        handler = FallbackHandler(
            primary=lambda: self.api.get_product(product_id)
        ).add_fallback(
            lambda: self.cache.get(f"product:{product_id}")
        ).add_fallback(
            lambda: {"id": product_id, "name": "Unknown", "available": False}
        )

        return handler.execute()


# 2. Alternative Service
class PaymentService:
    def __init__(self):
        self.stripe = StripeClient()
        self.paypal = PayPalClient()
        self.queue = PaymentQueue()

    def process_payment(self, order_id: str, amount: float) -> dict:
        handler = FallbackHandler(
            primary=lambda: self.stripe.charge(order_id, amount)
        ).add_fallback(
            lambda: self.paypal.charge(order_id, amount)
        ).add_fallback(
            lambda: self._queue_payment(order_id, amount)
        ).with_callback(
            lambda idx, name, err: self._log_fallback(order_id, idx, name, err)
        )

        return handler.execute()

    def _queue_payment(self, order_id: str, amount: float) -> dict:
        self.queue.add(order_id, amount)
        return {"status": "queued", "order_id": order_id}

    def _log_fallback(self, order_id, fallback_idx, name, error):
        logger.info(f"Order {order_id} used fallback #{fallback_idx} ({name})")


# 3. Graceful Degradation для рекомендаций
class RecommendationService:
    @with_fallback(fallback_func=lambda user_id: get_popular_products())
    def get_personalized_recommendations(self, user_id: str) -> list:
        return ml_service.get_recommendations(user_id)

    @with_fallback(fallback_value=[])
    def get_similar_products(self, product_id: str) -> list:
        return ml_service.get_similar(product_id)


# 4. Feature Flag Fallback
def get_feature_config(feature_name: str) -> dict:
    """Получение конфигурации feature flag с fallback."""
    try:
        # Пробуем получить из внешнего сервиса
        return feature_service.get_config(feature_name)
    except Exception:
        try:
            # Fallback на локальный кэш
            return local_cache.get(f"feature:{feature_name}")
        except Exception:
            # Fallback на дефолтные значения
            return DEFAULT_FEATURE_CONFIGS.get(
                feature_name,
                {"enabled": False}
            )
```

---

## Health Check Pattern (Проверка здоровья)

### Концепция

Health Check позволяет определить, готова ли система или её компоненты обрабатывать запросы. Load balancer'ы и оркестраторы используют эти проверки для маршрутизации трафика.

### Виды Health Check

```
┌─────────────────────────────────────────────────────────┐
│                    Health Check Types                    │
├─────────────────────────────────────────────────────────┤
│                                                          │
│  1. Liveness Check (Проверка живучести)                 │
│     Вопрос: "Процесс работает?"                         │
│     При fail: Перезапустить контейнер                   │
│     Endpoint: /health/live                              │
│                                                          │
│  2. Readiness Check (Проверка готовности)               │
│     Вопрос: "Готов принимать трафик?"                   │
│     При fail: Убрать из балансировки                    │
│     Endpoint: /health/ready                             │
│                                                          │
│  3. Startup Check (Проверка запуска)                    │
│     Вопрос: "Приложение полностью стартовало?"          │
│     При fail: Подождать, не перезапускать               │
│     Endpoint: /health/startup                           │
│                                                          │
│  4. Deep Health Check (Глубокая проверка)               │
│     Проверяет: DB, Cache, External APIs                 │
│     Использование: Мониторинг, алерты                   │
│     Endpoint: /health/deep                              │
│                                                          │
└─────────────────────────────────────────────────────────┘
```

### Реализация

```python
from enum import Enum
from dataclasses import dataclass, field
from datetime import datetime
from typing import Dict, List, Callable, Optional
import asyncio
import time
from fastapi import FastAPI, Response, status

app = FastAPI()


class HealthStatus(Enum):
    HEALTHY = "healthy"
    DEGRADED = "degraded"
    UNHEALTHY = "unhealthy"


@dataclass
class ComponentHealth:
    name: str
    status: HealthStatus
    message: str = ""
    latency_ms: float = 0.0
    details: Dict = field(default_factory=dict)


@dataclass
class HealthCheckResult:
    status: HealthStatus
    components: List[ComponentHealth]
    timestamp: datetime = field(default_factory=datetime.utcnow)

    def to_dict(self) -> dict:
        return {
            "status": self.status.value,
            "timestamp": self.timestamp.isoformat(),
            "components": [
                {
                    "name": c.name,
                    "status": c.status.value,
                    "message": c.message,
                    "latency_ms": c.latency_ms,
                    "details": c.details,
                }
                for c in self.components
            ]
        }


class HealthChecker:
    def __init__(self):
        self._checks: Dict[str, Callable] = {}

    def register(self, name: str, check: Callable):
        """Регистрация проверки здоровья компонента."""
        self._checks[name] = check

    async def check_component(self, name: str, check: Callable) -> ComponentHealth:
        """Выполнение одной проверки."""
        start = time.time()
        try:
            if asyncio.iscoroutinefunction(check):
                result = await check()
            else:
                result = check()

            latency = (time.time() - start) * 1000

            if isinstance(result, bool):
                status = HealthStatus.HEALTHY if result else HealthStatus.UNHEALTHY
                return ComponentHealth(name=name, status=status, latency_ms=latency)
            elif isinstance(result, dict):
                return ComponentHealth(
                    name=name,
                    status=result.get("status", HealthStatus.HEALTHY),
                    message=result.get("message", ""),
                    latency_ms=latency,
                    details=result.get("details", {})
                )
        except Exception as e:
            latency = (time.time() - start) * 1000
            return ComponentHealth(
                name=name,
                status=HealthStatus.UNHEALTHY,
                message=str(e),
                latency_ms=latency
            )

    async def check_all(self) -> HealthCheckResult:
        """Выполнение всех проверок."""
        tasks = [
            self.check_component(name, check)
            for name, check in self._checks.items()
        ]

        components = await asyncio.gather(*tasks)

        # Определяем общий статус
        if any(c.status == HealthStatus.UNHEALTHY for c in components):
            overall_status = HealthStatus.UNHEALTHY
        elif any(c.status == HealthStatus.DEGRADED for c in components):
            overall_status = HealthStatus.DEGRADED
        else:
            overall_status = HealthStatus.HEALTHY

        return HealthCheckResult(status=overall_status, components=list(components))


# Создаём глобальный checker
health_checker = HealthChecker()


# Регистрируем проверки компонентов
async def check_database() -> dict:
    """Проверка подключения к БД."""
    try:
        await database.execute("SELECT 1")
        return {"status": HealthStatus.HEALTHY}
    except Exception as e:
        return {"status": HealthStatus.UNHEALTHY, "message": str(e)}


async def check_redis() -> dict:
    """Проверка Redis."""
    try:
        await redis_client.ping()
        return {"status": HealthStatus.HEALTHY}
    except Exception as e:
        return {"status": HealthStatus.UNHEALTHY, "message": str(e)}


async def check_external_api() -> dict:
    """Проверка внешнего API (некритичный компонент)."""
    try:
        response = await http_client.get("https://api.example.com/health", timeout=5)
        if response.status_code == 200:
            return {"status": HealthStatus.HEALTHY}
        else:
            # API работает, но возвращает ошибку — degraded
            return {"status": HealthStatus.DEGRADED, "message": f"Status: {response.status_code}"}
    except Exception as e:
        # Внешний API недоступен, но это не критично
        return {"status": HealthStatus.DEGRADED, "message": str(e)}


# Регистрация
health_checker.register("database", check_database)
health_checker.register("redis", check_redis)
health_checker.register("external_api", check_external_api)


# Endpoints

@app.get("/health/live")
async def liveness():
    """
    Liveness probe — проверяет, что процесс жив.
    Kubernetes перезапустит pod при fail.
    """
    return {"status": "alive"}


@app.get("/health/ready")
async def readiness(response: Response):
    """
    Readiness probe — проверяет готовность принимать трафик.
    При fail трафик не направляется на этот instance.
    """
    # Проверяем только критичные компоненты
    db_health = await health_checker.check_component("database", check_database)
    redis_health = await health_checker.check_component("redis", check_redis)

    if db_health.status == HealthStatus.UNHEALTHY or \
       redis_health.status == HealthStatus.UNHEALTHY:
        response.status_code = status.HTTP_503_SERVICE_UNAVAILABLE
        return {"status": "not ready", "reason": "Critical dependency unavailable"}

    return {"status": "ready"}


@app.get("/health/deep")
async def deep_health_check(response: Response):
    """
    Полная проверка всех компонентов.
    Используется для мониторинга и алертов.
    """
    result = await health_checker.check_all()

    if result.status == HealthStatus.UNHEALTHY:
        response.status_code = status.HTTP_503_SERVICE_UNAVAILABLE
    elif result.status == HealthStatus.DEGRADED:
        response.status_code = status.HTTP_200_OK  # Работает, но с ограничениями

    return result.to_dict()


# Kubernetes конфигурация
"""
apiVersion: v1
kind: Pod
spec:
  containers:
  - name: app
    livenessProbe:
      httpGet:
        path: /health/live
        port: 8080
      initialDelaySeconds: 10
      periodSeconds: 10
      failureThreshold: 3
    readinessProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 5
      periodSeconds: 5
      failureThreshold: 3
    startupProbe:
      httpGet:
        path: /health/ready
        port: 8080
      initialDelaySeconds: 0
      periodSeconds: 5
      failureThreshold: 30
"""
```

---

## Rate Limiting (Ограничение скорости)

### Концепция

Rate Limiting защищает систему от перегрузки, ограничивая количество запросов от клиентов. Это предотвращает DDoS атаки, abuse API и обеспечивает fair usage.

### Алгоритмы

```
┌─────────────────────────────────────────────────────────────────┐
│                    Rate Limiting Algorithms                      │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│  1. Fixed Window (Фиксированное окно)                           │
│     [     Минута 1: 100 запросов     ][     Минута 2: 100 запросов     ]   │
│     Проблема: 200 запросов за 2 секунды на границе окон         │
│                                                                  │
│  2. Sliding Window Log (Скользящее окно с логом)                │
│     Хранит timestamp каждого запроса                            │
│     [req1, req2, req3, ...] — удаляет старые                    │
│     Точный, но затратный по памяти                              │
│                                                                  │
│  3. Sliding Window Counter (Скользящее окно со счётчиком)       │
│     Взвешенное среднее между текущим и предыдущим окном         │
│     Компромисс между точностью и ресурсами                      │
│                                                                  │
│  4. Token Bucket (Корзина токенов)                              │
│     ○○○○○○○○○○ — 10 токенов                                     │
│     Каждый запрос забирает токен                                │
│     Токены добавляются с фиксированной скоростью                │
│     Позволяет burst трафик                                      │
│                                                                  │
│  5. Leaky Bucket (Протекающая корзина)                          │
│     Запросы помещаются в очередь (bucket)                       │
│     Обрабатываются с постоянной скоростью                       │
│     Сглаживает burst трафик                                     │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Реализация

```python
import time
import asyncio
from abc import ABC, abstractmethod
from dataclasses import dataclass
from typing import Optional
import redis.asyncio as redis
from collections import deque
import threading


@dataclass
class RateLimitResult:
    allowed: bool
    remaining: int
    reset_at: float
    retry_after: Optional[float] = None


class RateLimiter(ABC):
    @abstractmethod
    async def is_allowed(self, key: str) -> RateLimitResult:
        pass


# 1. Token Bucket Implementation
class TokenBucket(RateLimiter):
    """
    Token Bucket алгоритм.
    Позволяет burst трафик до размера корзины.
    """
    def __init__(
        self,
        capacity: int,          # Максимум токенов в корзине
        refill_rate: float,     # Токенов в секунду
        redis_client: redis.Redis
    ):
        self.capacity = capacity
        self.refill_rate = refill_rate
        self.redis = redis_client

    async def is_allowed(self, key: str) -> RateLimitResult:
        now = time.time()
        bucket_key = f"ratelimit:token_bucket:{key}"

        # Lua script для атомарной операции
        lua_script = """
        local key = KEYS[1]
        local capacity = tonumber(ARGV[1])
        local refill_rate = tonumber(ARGV[2])
        local now = tonumber(ARGV[3])

        local bucket = redis.call('HMGET', key, 'tokens', 'last_update')
        local tokens = tonumber(bucket[1]) or capacity
        local last_update = tonumber(bucket[2]) or now

        -- Пополняем токены
        local elapsed = now - last_update
        local new_tokens = math.min(capacity, tokens + elapsed * refill_rate)

        if new_tokens >= 1 then
            -- Есть токены — разрешаем запрос
            redis.call('HMSET', key, 'tokens', new_tokens - 1, 'last_update', now)
            redis.call('EXPIRE', key, 3600)
            return {1, math.floor(new_tokens - 1)}
        else
            -- Нет токенов — отклоняем
            redis.call('HMSET', key, 'tokens', new_tokens, 'last_update', now)
            redis.call('EXPIRE', key, 3600)
            local wait_time = (1 - new_tokens) / refill_rate
            return {0, 0, wait_time}
        end
        """

        result = await self.redis.eval(
            lua_script, 1, bucket_key,
            self.capacity, self.refill_rate, now
        )

        allowed = bool(result[0])
        remaining = int(result[1])
        retry_after = float(result[2]) if len(result) > 2 else None

        return RateLimitResult(
            allowed=allowed,
            remaining=remaining,
            reset_at=now + (self.capacity - remaining) / self.refill_rate,
            retry_after=retry_after
        )


# 2. Sliding Window Counter
class SlidingWindowCounter(RateLimiter):
    """
    Sliding Window Counter алгоритм.
    Более справедливый, чем Fixed Window.
    """
    def __init__(
        self,
        limit: int,             # Лимит запросов
        window_size: int,       # Размер окна в секундах
        redis_client: redis.Redis
    ):
        self.limit = limit
        self.window_size = window_size
        self.redis = redis_client

    async def is_allowed(self, key: str) -> RateLimitResult:
        now = time.time()
        current_window = int(now // self.window_size)
        previous_window = current_window - 1

        current_key = f"ratelimit:sliding:{key}:{current_window}"
        previous_key = f"ratelimit:sliding:{key}:{previous_window}"

        # Получаем счётчики
        pipe = self.redis.pipeline()
        pipe.get(current_key)
        pipe.get(previous_key)
        results = await pipe.execute()

        current_count = int(results[0] or 0)
        previous_count = int(results[1] or 0)

        # Вычисляем взвешенное значение
        window_progress = (now % self.window_size) / self.window_size
        weighted_count = (
            previous_count * (1 - window_progress) +
            current_count
        )

        if weighted_count < self.limit:
            # Разрешаем и увеличиваем счётчик
            await self.redis.incr(current_key)
            await self.redis.expire(current_key, self.window_size * 2)

            return RateLimitResult(
                allowed=True,
                remaining=int(self.limit - weighted_count - 1),
                reset_at=(current_window + 1) * self.window_size
            )
        else:
            # Лимит превышен
            return RateLimitResult(
                allowed=False,
                remaining=0,
                reset_at=(current_window + 1) * self.window_size,
                retry_after=(current_window + 1) * self.window_size - now
            )


# 3. In-Memory Token Bucket (для локального использования)
class InMemoryTokenBucket:
    """Простая in-memory реализация для single-instance приложений."""

    def __init__(self, capacity: int, refill_rate: float):
        self.capacity = capacity
        self.refill_rate = refill_rate
        self._buckets: dict = {}
        self._lock = threading.Lock()

    def is_allowed(self, key: str) -> bool:
        now = time.time()

        with self._lock:
            if key not in self._buckets:
                self._buckets[key] = {"tokens": self.capacity, "last_update": now}

            bucket = self._buckets[key]

            # Пополняем токены
            elapsed = now - bucket["last_update"]
            bucket["tokens"] = min(
                self.capacity,
                bucket["tokens"] + elapsed * self.refill_rate
            )
            bucket["last_update"] = now

            if bucket["tokens"] >= 1:
                bucket["tokens"] -= 1
                return True
            return False


# FastAPI Middleware
from fastapi import Request, HTTPException
from starlette.middleware.base import BaseHTTPMiddleware


class RateLimitMiddleware(BaseHTTPMiddleware):
    def __init__(self, app, rate_limiter: RateLimiter, key_func=None):
        super().__init__(app)
        self.rate_limiter = rate_limiter
        self.key_func = key_func or self._default_key_func

    def _default_key_func(self, request: Request) -> str:
        """По умолчанию используем IP клиента."""
        forwarded = request.headers.get("X-Forwarded-For")
        if forwarded:
            return forwarded.split(",")[0].strip()
        return request.client.host

    async def dispatch(self, request: Request, call_next):
        key = self.key_func(request)
        result = await self.rate_limiter.is_allowed(key)

        if not result.allowed:
            raise HTTPException(
                status_code=429,
                detail="Too Many Requests",
                headers={
                    "X-RateLimit-Limit": str(self.rate_limiter.limit),
                    "X-RateLimit-Remaining": str(result.remaining),
                    "X-RateLimit-Reset": str(int(result.reset_at)),
                    "Retry-After": str(int(result.retry_after or 1))
                }
            )

        response = await call_next(request)

        # Добавляем заголовки rate limit в ответ
        response.headers["X-RateLimit-Remaining"] = str(result.remaining)
        response.headers["X-RateLimit-Reset"] = str(int(result.reset_at))

        return response


# Использование
app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379)

# 100 запросов в минуту с возможностью burst до 20
rate_limiter = TokenBucket(
    capacity=20,
    refill_rate=100/60,  # 100 в минуту = ~1.67 в секунду
    redis_client=redis_client
)

app.add_middleware(RateLimitMiddleware, rate_limiter=rate_limiter)


# Разные лимиты для разных endpoints
class TieredRateLimiter:
    """Разные лимиты для разных типов запросов."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

        # Лимиты по тирам
        self.limiters = {
            "free": TokenBucket(capacity=10, refill_rate=10/60, redis_client=redis_client),
            "basic": TokenBucket(capacity=50, refill_rate=100/60, redis_client=redis_client),
            "premium": TokenBucket(capacity=200, refill_rate=1000/60, redis_client=redis_client),
        }

    async def check(self, user_id: str, tier: str) -> RateLimitResult:
        limiter = self.limiters.get(tier, self.limiters["free"])
        return await limiter.is_allowed(f"user:{user_id}")
```

---

## Комбинирование паттернов

### Resilient Service Client

```python
from dataclasses import dataclass
from typing import Callable, TypeVar, Optional
import time

T = TypeVar('T')


@dataclass
class ResilienceConfig:
    # Circuit Breaker
    failure_threshold: int = 5
    recovery_timeout: float = 30.0

    # Retry
    max_retries: int = 3
    retry_delay: float = 1.0
    retry_backoff: float = 2.0

    # Timeout
    timeout: float = 10.0

    # Bulkhead
    max_concurrent: int = 10

    # Rate Limit
    requests_per_second: float = 100.0


class ResilientClient:
    """
    Клиент, объединяющий все паттерны надёжности.
    """
    def __init__(self, config: ResilienceConfig):
        self.config = config

        self.circuit_breaker = CircuitBreaker(
            failure_threshold=config.failure_threshold,
            timeout=config.recovery_timeout
        )

        self.bulkhead = Bulkhead(BulkheadConfig(
            max_concurrent=config.max_concurrent
        ))

        self.rate_limiter = InMemoryTokenBucket(
            capacity=int(config.requests_per_second),
            refill_rate=config.requests_per_second
        )

    def execute(
        self,
        operation: Callable[..., T],
        fallback: Optional[Callable[..., T]] = None,
        *args,
        **kwargs
    ) -> T:
        """
        Выполняет операцию с применением всех паттернов надёжности.

        Порядок: Rate Limit -> Bulkhead -> Circuit Breaker -> Retry -> Timeout
        """
        # 1. Rate Limiting
        if not self.rate_limiter.is_allowed("default"):
            if fallback:
                return fallback(*args, **kwargs)
            raise RateLimitExceededError("Rate limit exceeded")

        # 2. Bulkhead
        try:
            with self.bulkhead.acquire():
                # 3. Circuit Breaker
                if self.circuit_breaker.state == CircuitState.OPEN:
                    if fallback:
                        return fallback(*args, **kwargs)
                    raise CircuitBreakerOpenError("Circuit breaker is open")

                # 4. Retry with Timeout
                last_exception = None
                delay = self.config.retry_delay

                for attempt in range(self.config.max_retries):
                    try:
                        # 5. Execute with Timeout
                        result = self._execute_with_timeout(
                            operation,
                            self.config.timeout,
                            *args,
                            **kwargs
                        )
                        self.circuit_breaker._handle_success()
                        return result

                    except Exception as e:
                        last_exception = e
                        self.circuit_breaker._handle_failure()

                        if attempt < self.config.max_retries - 1:
                            time.sleep(delay)
                            delay *= self.config.retry_backoff

                # Все попытки исчерпаны
                if fallback:
                    return fallback(*args, **kwargs)
                raise last_exception

        except BulkheadFullError:
            if fallback:
                return fallback(*args, **kwargs)
            raise

    def _execute_with_timeout(
        self,
        operation: Callable,
        timeout: float,
        *args,
        **kwargs
    ):
        # Реализация с ThreadPoolExecutor как показано выше
        with ThreadPoolExecutor(max_workers=1) as executor:
            future = executor.submit(operation, *args, **kwargs)
            return future.result(timeout=timeout)


# Использование
config = ResilienceConfig(
    failure_threshold=5,
    recovery_timeout=60.0,
    max_retries=3,
    timeout=10.0,
    max_concurrent=20,
    requests_per_second=100.0
)

client = ResilientClient(config)


def fetch_user(user_id: int) -> dict:
    response = requests.get(f"https://api.example.com/users/{user_id}")
    response.raise_for_status()
    return response.json()


def get_cached_user(user_id: int) -> dict:
    return cache.get(f"user:{user_id}") or {"id": user_id, "name": "Unknown"}


# Вызов с полной защитой
user = client.execute(
    operation=lambda: fetch_user(123),
    fallback=lambda: get_cached_user(123)
)
```

---

## Best Practices

### 1. Правильный выбор паттернов

```
┌─────────────────────────────────────────────────────────┐
│                 Когда использовать                       │
├─────────────────────────────────────────────────────────┤
│ Circuit Breaker: Защита от каскадных отказов            │
│                  Когда downstream сервис может упасть   │
│                                                          │
│ Retry:           Transient ошибки (сетевые проблемы)    │
│                  НЕ использовать для бизнес-ошибок     │
│                                                          │
│ Timeout:         ВСЕГДА для внешних вызовов             │
│                  Разные значения для разных операций    │
│                                                          │
│ Bulkhead:        Изоляция критичных от некритичных      │
│                  Защита от noisy neighbor               │
│                                                          │
│ Fallback:        Graceful degradation                   │
│                  Когда есть альтернативные источники    │
│                                                          │
│ Health Check:    Kubernetes/Cloud environments          │
│                  Load balancing                          │
│                                                          │
│ Rate Limiting:   Публичные API                          │
│                  Защита от abuse                        │
└─────────────────────────────────────────────────────────┘
```

### 2. Мониторинг и метрики

```python
# Важные метрики для каждого паттерна
METRICS = {
    "circuit_breaker": [
        "circuit_breaker_state",           # Текущее состояние
        "circuit_breaker_failures_total",  # Счётчик ошибок
        "circuit_breaker_success_total",   # Счётчик успехов
        "circuit_breaker_rejected_total",  # Отклонённые запросы
    ],
    "retry": [
        "retry_attempts_total",            # Количество повторов
        "retry_exhausted_total",           # Исчерпанные повторы
    ],
    "timeout": [
        "request_timeout_total",           # Таймауты
        "request_duration_seconds",        # Время выполнения
    ],
    "bulkhead": [
        "bulkhead_available_permits",      # Доступные слоты
        "bulkhead_rejected_total",         # Отклонённые запросы
    ],
    "rate_limit": [
        "rate_limit_total",                # Все запросы
        "rate_limit_rejected_total",       # Отклонённые
    ],
}
```

### 3. Типичные ошибки

```python
# ОШИБКА 1: Retry без backoff
@retry(max_attempts=10, delay=0.1)  # Молотит сервер!
def bad_retry():
    pass

# ОШИБКА 2: Retry non-idempotent операций
@retry(max_attempts=3)
def create_order():  # Может создать дубликаты!
    pass

# ОШИБКА 3: Слишком большой timeout
response = requests.get(url, timeout=300)  # 5 минут — слишком много

# ОШИБКА 4: Нет fallback при критичных операциях
def get_user(user_id):
    return api.get_user(user_id)  # Что если API недоступен?

# ОШИБКА 5: Одинаковые лимиты для всех операций
# Тяжёлые и лёгкие операции должны иметь разные лимиты
```

---

## Заключение

Паттерны надёжности — это фундамент отказоустойчивых систем. Ключевые принципы:

1. **Design for failure** — предполагайте, что всё может сломаться
2. **Fail fast** — быстро обнаруживайте проблемы
3. **Graceful degradation** — лучше работать частично, чем не работать совсем
4. **Isolation** — изолируйте компоненты друг от друга
5. **Observability** — мониторьте всё

Библиотеки для Python:
- `tenacity` — retry с множеством стратегий
- `pybreaker` — circuit breaker
- `aiobreaker` — async circuit breaker
- `resilience4j` (Java) — комплексное решение

Инструменты:
- **Istio/Envoy** — service mesh с встроенными паттернами
- **AWS App Mesh** — managed service mesh
- **Linkerd** — lightweight service mesh
