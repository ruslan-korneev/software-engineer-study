# Error Handling and Retries (Обработка ошибок и повторные попытки)

[prev: 06-contract-testing](./06-contract-testing.md) | [next: 01-websockets](../10-real-time-apis/01-websockets.md)

---

## Введение

**Обработка ошибок и механизмы повторных попыток** - это критически важные аспекты надежного взаимодействия с API. Правильная реализация этих механизмов позволяет создавать устойчивые системы, которые корректно работают даже при сбоях сети, временной недоступности сервисов и других проблемах.

## Типы ошибок API

### Классификация по HTTP-кодам

| Код | Категория | Описание | Retry? |
|-----|-----------|----------|--------|
| **4xx** | Client Error | Ошибка клиента | Обычно нет |
| 400 | Bad Request | Некорректный запрос | Нет |
| 401 | Unauthorized | Не авторизован | Нет (refresh token) |
| 403 | Forbidden | Доступ запрещен | Нет |
| 404 | Not Found | Ресурс не найден | Нет |
| 422 | Validation Error | Ошибка валидации | Нет |
| 429 | Too Many Requests | Rate limit | Да (с задержкой) |
| **5xx** | Server Error | Ошибка сервера | Да |
| 500 | Internal Error | Внутренняя ошибка | Да |
| 502 | Bad Gateway | Ошибка шлюза | Да |
| 503 | Service Unavailable | Сервис недоступен | Да |
| 504 | Gateway Timeout | Таймаут шлюза | Да |

### Классификация по типу

1. **Transient errors (Временные)** - можно решить повторной попыткой
   - Сетевые таймауты
   - 5xx ошибки
   - Rate limiting

2. **Permanent errors (Постоянные)** - требуют вмешательства
   - 4xx ошибки (кроме 429)
   - Ошибки валидации
   - Ошибки авторизации

## Стратегии повторных попыток

### 1. Fixed Delay (Фиксированная задержка)

```python
import time
import requests

def retry_fixed(url, max_retries=3, delay=1.0):
    """Повтор с фиксированной задержкой."""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response
        except requests.RequestException as e:
            if attempt == max_retries - 1:
                raise
            print(f"Attempt {attempt + 1} failed: {e}")
            time.sleep(delay)
```

### 2. Exponential Backoff (Экспоненциальная задержка)

```python
import time
import random
import requests

def retry_exponential(url, max_retries=5, base_delay=1.0, max_delay=60.0):
    """Повтор с экспоненциальной задержкой."""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response
        except requests.RequestException as e:
            if attempt == max_retries - 1:
                raise

            # Экспоненциальная задержка: 1, 2, 4, 8, 16...
            delay = min(base_delay * (2 ** attempt), max_delay)
            print(f"Attempt {attempt + 1} failed: {e}. Retrying in {delay}s")
            time.sleep(delay)
```

### 3. Exponential Backoff with Jitter (С рандомизацией)

```python
def retry_with_jitter(url, max_retries=5, base_delay=1.0, max_delay=60.0):
    """Повтор с экспоненциальной задержкой и jitter."""
    for attempt in range(max_retries):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response
        except requests.RequestException as e:
            if attempt == max_retries - 1:
                raise

            # Добавляем случайность для избежания thundering herd
            delay = min(base_delay * (2 ** attempt), max_delay)
            jitter = random.uniform(0, delay * 0.1)  # 10% jitter
            actual_delay = delay + jitter

            print(f"Attempt {attempt + 1} failed. Retrying in {actual_delay:.2f}s")
            time.sleep(actual_delay)
```

### 4. Decorrelated Jitter (AWS рекомендация)

```python
def retry_decorrelated_jitter(url, max_retries=5, base_delay=1.0, max_delay=60.0):
    """Повтор с декоррелированным jitter (рекомендация AWS)."""
    delay = base_delay

    for attempt in range(max_retries):
        try:
            response = requests.get(url, timeout=10)
            response.raise_for_status()
            return response
        except requests.RequestException as e:
            if attempt == max_retries - 1:
                raise

            # Decorrelated jitter: delay = random(base, prev_delay * 3)
            delay = min(max_delay, random.uniform(base_delay, delay * 3))
            print(f"Attempt {attempt + 1} failed. Retrying in {delay:.2f}s")
            time.sleep(delay)
```

## Библиотека tenacity (Python)

```bash
pip install tenacity
```

### Базовое использование

```python
from tenacity import (
    retry,
    stop_after_attempt,
    wait_exponential,
    retry_if_exception_type,
    before_sleep_log
)
import logging
import requests

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, min=1, max=60),
    retry=retry_if_exception_type(requests.RequestException),
    before_sleep=before_sleep_log(logger, logging.WARNING)
)
def fetch_data(url: str) -> dict:
    """Получение данных с автоматическими повторами."""
    response = requests.get(url, timeout=10)
    response.raise_for_status()
    return response.json()
```

### Продвинутые сценарии

```python
from tenacity import (
    retry,
    stop_after_attempt,
    stop_after_delay,
    wait_exponential,
    wait_random,
    wait_chain,
    retry_if_result,
    retry_if_exception_type,
    RetryError
)
import requests

# Комбинированные условия остановки
@retry(
    stop=(stop_after_attempt(5) | stop_after_delay(30)),
    wait=wait_exponential(multiplier=1, max=10)
)
def fetch_with_time_limit(url: str):
    """Повтор с лимитом по попыткам И времени."""
    response = requests.get(url, timeout=5)
    response.raise_for_status()
    return response.json()

# Повтор по результату
def is_empty_result(result):
    return result is None or len(result) == 0

@retry(
    retry=retry_if_result(is_empty_result),
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, max=10)
)
def fetch_until_data(url: str):
    """Повтор пока не получим данные."""
    response = requests.get(url)
    return response.json().get("items", [])

# Разные задержки для разных попыток
@retry(
    wait=wait_chain(
        wait_fixed(1),  # Первая попытка через 1 сек
        wait_fixed(2),  # Вторая через 2 сек
        wait_exponential(multiplier=1, max=30)  # Далее экспоненциально
    ),
    stop=stop_after_attempt(5)
)
def fetch_with_custom_waits(url: str):
    response = requests.get(url)
    response.raise_for_status()
    return response.json()

# Кастомные исключения
class RetryableError(Exception):
    """Ошибка, которую можно повторить."""
    pass

class NonRetryableError(Exception):
    """Ошибка, которую нельзя повторить."""
    pass

@retry(
    retry=retry_if_exception_type(RetryableError),
    stop=stop_after_attempt(3)
)
def process_with_custom_errors():
    """Обработка с кастомными ошибками."""
    response = requests.get("http://api.example.com/data")

    if response.status_code == 503:
        raise RetryableError("Service unavailable")
    elif response.status_code == 400:
        raise NonRetryableError("Bad request")

    return response.json()
```

### Асинхронные повторы

```python
from tenacity import (
    AsyncRetrying,
    stop_after_attempt,
    wait_exponential
)
import httpx
import asyncio

async def fetch_async_with_retry(url: str) -> dict:
    """Асинхронное получение данных с повторами."""
    async for attempt in AsyncRetrying(
        stop=stop_after_attempt(5),
        wait=wait_exponential(multiplier=1, max=30)
    ):
        with attempt:
            async with httpx.AsyncClient() as client:
                response = await client.get(url, timeout=10)
                response.raise_for_status()
                return response.json()

# Или с декоратором
from tenacity import retry

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, max=30)
)
async def fetch_async(url: str) -> dict:
    async with httpx.AsyncClient() as client:
        response = await client.get(url, timeout=10)
        response.raise_for_status()
        return response.json()
```

## HTTP-клиент с retry (httpx)

```python
import httpx
from httpx import HTTPStatusError, ConnectError, TimeoutException

class RetryTransport(httpx.HTTPTransport):
    """HTTP транспорт с поддержкой retry."""

    def __init__(
        self,
        max_retries: int = 3,
        retry_statuses: set = None,
        **kwargs
    ):
        super().__init__(**kwargs)
        self.max_retries = max_retries
        self.retry_statuses = retry_statuses or {429, 500, 502, 503, 504}

    def handle_request(self, request):
        for attempt in range(self.max_retries + 1):
            try:
                response = super().handle_request(request)

                if response.status_code in self.retry_statuses:
                    if attempt < self.max_retries:
                        # Получаем Retry-After header если есть
                        retry_after = response.headers.get("Retry-After", "1")
                        delay = float(retry_after)
                        time.sleep(delay)
                        continue

                return response

            except (ConnectError, TimeoutException) as e:
                if attempt == self.max_retries:
                    raise
                time.sleep(2 ** attempt)

# Использование
client = httpx.Client(
    transport=RetryTransport(max_retries=3),
    timeout=30.0
)
```

## Circuit Breaker Pattern

Паттерн "Предохранитель" предотвращает каскадные сбои:

```python
from enum import Enum
from datetime import datetime, timedelta
from threading import Lock
from typing import Callable, TypeVar, Generic

T = TypeVar('T')

class CircuitState(Enum):
    CLOSED = "closed"      # Нормальная работа
    OPEN = "open"          # Запросы блокируются
    HALF_OPEN = "half_open"  # Тестовый режим

class CircuitBreaker(Generic[T]):
    """Реализация паттерна Circuit Breaker."""

    def __init__(
        self,
        failure_threshold: int = 5,
        recovery_timeout: int = 30,
        success_threshold: int = 2
    ):
        self.failure_threshold = failure_threshold
        self.recovery_timeout = recovery_timeout
        self.success_threshold = success_threshold

        self.state = CircuitState.CLOSED
        self.failures = 0
        self.successes = 0
        self.last_failure_time = None
        self.lock = Lock()

    def call(self, func: Callable[[], T]) -> T:
        """Выполнение функции с защитой circuit breaker."""
        with self.lock:
            if self.state == CircuitState.OPEN:
                if self._should_attempt_reset():
                    self.state = CircuitState.HALF_OPEN
                else:
                    raise CircuitBreakerOpenError("Circuit breaker is open")

        try:
            result = func()
            self._on_success()
            return result
        except Exception as e:
            self._on_failure()
            raise

    def _should_attempt_reset(self) -> bool:
        """Проверка, можно ли попробовать восстановить соединение."""
        if self.last_failure_time is None:
            return True
        return datetime.now() - self.last_failure_time > timedelta(
            seconds=self.recovery_timeout
        )

    def _on_success(self):
        """Обработка успешного вызова."""
        with self.lock:
            if self.state == CircuitState.HALF_OPEN:
                self.successes += 1
                if self.successes >= self.success_threshold:
                    self.state = CircuitState.CLOSED
                    self.failures = 0
                    self.successes = 0
            else:
                self.failures = 0

    def _on_failure(self):
        """Обработка неудачного вызова."""
        with self.lock:
            self.failures += 1
            self.last_failure_time = datetime.now()

            if self.state == CircuitState.HALF_OPEN:
                self.state = CircuitState.OPEN
                self.successes = 0
            elif self.failures >= self.failure_threshold:
                self.state = CircuitState.OPEN

class CircuitBreakerOpenError(Exception):
    """Исключение при открытом circuit breaker."""
    pass

# Использование
breaker = CircuitBreaker(failure_threshold=5, recovery_timeout=30)

def make_request():
    response = requests.get("http://api.example.com/data")
    response.raise_for_status()
    return response.json()

try:
    result = breaker.call(make_request)
except CircuitBreakerOpenError:
    # Используем fallback или возвращаем кэшированные данные
    result = get_cached_data()
```

### pybreaker - готовая библиотека

```bash
pip install pybreaker
```

```python
import pybreaker
import requests

# Создание circuit breaker
breaker = pybreaker.CircuitBreaker(
    fail_max=5,
    reset_timeout=30
)

@breaker
def get_users():
    response = requests.get("http://api.example.com/users")
    response.raise_for_status()
    return response.json()

# С listener для мониторинга
class LoggingListener(pybreaker.CircuitBreakerListener):
    def state_change(self, cb, old_state, new_state):
        print(f"Circuit breaker {cb.name}: {old_state.name} -> {new_state.name}")

    def failure(self, cb, exc):
        print(f"Circuit breaker {cb.name} failure: {exc}")

breaker = pybreaker.CircuitBreaker(
    fail_max=5,
    reset_timeout=30,
    listeners=[LoggingListener()]
)
```

## Обработка Rate Limiting

```python
import time
import requests
from dataclasses import dataclass
from typing import Optional

@dataclass
class RateLimitInfo:
    limit: int
    remaining: int
    reset_at: float

def extract_rate_limit_info(response: requests.Response) -> Optional[RateLimitInfo]:
    """Извлечение информации о rate limit из заголовков."""
    headers = response.headers

    if "X-RateLimit-Limit" in headers:
        return RateLimitInfo(
            limit=int(headers.get("X-RateLimit-Limit", 0)),
            remaining=int(headers.get("X-RateLimit-Remaining", 0)),
            reset_at=float(headers.get("X-RateLimit-Reset", 0))
        )

    # Альтернативные заголовки
    if "RateLimit-Limit" in headers:
        return RateLimitInfo(
            limit=int(headers.get("RateLimit-Limit", 0)),
            remaining=int(headers.get("RateLimit-Remaining", 0)),
            reset_at=time.time() + int(headers.get("RateLimit-Reset", 0))
        )

    return None

class RateLimitedClient:
    """HTTP-клиент с обработкой rate limiting."""

    def __init__(self, base_url: str):
        self.base_url = base_url
        self.session = requests.Session()

    def get(self, path: str, **kwargs) -> requests.Response:
        url = f"{self.base_url}{path}"

        while True:
            response = self.session.get(url, **kwargs)

            if response.status_code == 429:
                # Получаем время ожидания
                retry_after = response.headers.get("Retry-After")
                if retry_after:
                    wait_time = float(retry_after)
                else:
                    rate_info = extract_rate_limit_info(response)
                    if rate_info:
                        wait_time = rate_info.reset_at - time.time()
                    else:
                        wait_time = 60  # По умолчанию

                print(f"Rate limited. Waiting {wait_time:.2f} seconds")
                time.sleep(max(0, wait_time))
                continue

            return response
```

## Тестирование обработки ошибок

```python
# test_error_handling.py
import pytest
import responses
import requests
from unittest.mock import patch

class TestRetryMechanism:
    """Тесты механизма повторных попыток."""

    @responses.activate
    def test_retry_on_500_error(self):
        """Тест повтора при 500 ошибке."""
        # Первые 2 запроса возвращают 500, третий - успех
        responses.add(responses.GET, "http://api.example.com/data", status=500)
        responses.add(responses.GET, "http://api.example.com/data", status=500)
        responses.add(
            responses.GET, "http://api.example.com/data",
            json={"data": "success"}, status=200
        )

        result = fetch_data_with_retry("http://api.example.com/data")

        assert result == {"data": "success"}
        assert len(responses.calls) == 3

    @responses.activate
    def test_no_retry_on_400_error(self):
        """Тест отсутствия повтора при 400 ошибке."""
        responses.add(
            responses.GET, "http://api.example.com/data",
            json={"error": "Bad request"}, status=400
        )

        with pytest.raises(requests.HTTPError):
            fetch_data_with_retry("http://api.example.com/data")

        assert len(responses.calls) == 1

    @responses.activate
    def test_retry_respects_retry_after_header(self):
        """Тест соблюдения заголовка Retry-After."""
        responses.add(
            responses.GET, "http://api.example.com/data",
            headers={"Retry-After": "2"},
            status=429
        )
        responses.add(
            responses.GET, "http://api.example.com/data",
            json={"data": "success"}, status=200
        )

        with patch("time.sleep") as mock_sleep:
            result = fetch_data_with_retry("http://api.example.com/data")

            mock_sleep.assert_called_once_with(2.0)
            assert result == {"data": "success"}

    @responses.activate
    def test_max_retries_exceeded(self):
        """Тест превышения максимального числа попыток."""
        for _ in range(10):
            responses.add(
                responses.GET, "http://api.example.com/data",
                status=503
            )

        with pytest.raises(requests.HTTPError):
            fetch_data_with_retry(
                "http://api.example.com/data",
                max_retries=3
            )

        assert len(responses.calls) == 4  # 1 + 3 retries


class TestCircuitBreaker:
    """Тесты circuit breaker."""

    def test_circuit_opens_after_failures(self):
        """Тест открытия circuit breaker после сбоев."""
        breaker = CircuitBreaker(failure_threshold=3)

        def failing_func():
            raise Exception("Simulated failure")

        # 3 сбоя должны открыть breaker
        for _ in range(3):
            with pytest.raises(Exception):
                breaker.call(failing_func)

        # Следующий вызов должен вернуть CircuitBreakerOpenError
        with pytest.raises(CircuitBreakerOpenError):
            breaker.call(failing_func)

    def test_circuit_closes_after_success(self):
        """Тест закрытия circuit breaker после успехов."""
        breaker = CircuitBreaker(
            failure_threshold=2,
            recovery_timeout=0,  # Мгновенное восстановление для теста
            success_threshold=1
        )

        call_count = 0
        def sometimes_failing_func():
            nonlocal call_count
            call_count += 1
            if call_count <= 2:
                raise Exception("Fail")
            return "success"

        # Первые 2 вызова - сбой
        for _ in range(2):
            with pytest.raises(Exception):
                breaker.call(sometimes_failing_func)

        # Breaker открыт, но сразу пробуем half-open
        result = breaker.call(sometimes_failing_func)

        assert result == "success"
        assert breaker.state == CircuitState.CLOSED


class TestRateLimiting:
    """Тесты обработки rate limiting."""

    @responses.activate
    def test_handles_rate_limit(self):
        """Тест обработки rate limit."""
        responses.add(
            responses.GET, "http://api.example.com/data",
            headers={"Retry-After": "1"},
            status=429
        )
        responses.add(
            responses.GET, "http://api.example.com/data",
            json={"data": "success"}, status=200
        )

        client = RateLimitedClient("http://api.example.com")

        with patch("time.sleep"):
            response = client.get("/data")

        assert response.status_code == 200
        assert len(responses.calls) == 2
```

## Best Practices

### 1. Идемпотентность запросов

```python
import uuid

def make_idempotent_request(data: dict):
    """Запрос с идемпотентным ключом."""
    idempotency_key = str(uuid.uuid4())

    response = requests.post(
        "http://api.example.com/payments",
        json=data,
        headers={"Idempotency-Key": idempotency_key}
    )
    return response
```

### 2. Таймауты

```python
# Всегда устанавливайте таймауты
response = requests.get(
    url,
    timeout=(3.05, 27)  # (connect timeout, read timeout)
)
```

### 3. Логирование ошибок

```python
import logging
from tenacity import before_sleep_log, after_log

logger = logging.getLogger(__name__)

@retry(
    stop=stop_after_attempt(5),
    wait=wait_exponential(multiplier=1, max=30),
    before_sleep=before_sleep_log(logger, logging.WARNING),
    after=after_log(logger, logging.INFO)
)
def fetch_with_logging(url: str):
    response = requests.get(url)
    response.raise_for_status()
    return response.json()
```

### 4. Graceful Degradation

```python
def get_user_data(user_id: int) -> dict:
    """Получение данных с fallback."""
    try:
        return fetch_from_api(user_id)
    except Exception as e:
        logger.warning(f"API failed: {e}, using cache")
        cached = get_from_cache(user_id)
        if cached:
            return cached
        return get_default_user_data()
```

## Инструменты

| Инструмент | Описание |
|------------|----------|
| **tenacity** | Универсальная библиотека retry для Python |
| **backoff** | Декораторы для retry с backoff |
| **pybreaker** | Circuit breaker для Python |
| **resilience4j** | Resilience библиотека для Java |
| **Polly** | Resilience библиотека для .NET |

## Заключение

Правильная обработка ошибок и механизмы повторных попыток:

- **Повышают надежность** - система устойчива к временным сбоям
- **Предотвращают каскадные сбои** - circuit breaker защищает от перегрузки
- **Улучшают UX** - пользователь не видит временные проблемы
- **Упрощают отладку** - хорошее логирование помогает находить проблемы

Ключевые правила:
- Всегда устанавливайте таймауты
- Используйте exponential backoff с jitter
- Различайте retryable и non-retryable ошибки
- Реализуйте circuit breaker для защиты от каскадных сбоев
- Логируйте все ошибки и повторы

---

[prev: 06-contract-testing](./06-contract-testing.md) | [next: 01-websockets](../10-real-time-apis/01-websockets.md)
