# Circuit Breaker (Автоматический выключатель)

## Определение

**Circuit Breaker** (Автоматический выключатель) — это паттерн отказоустойчивости, который предотвращает каскадные сбои в распределённых системах, временно прекращая вызовы к неисправному сервису. Название заимствовано из электротехники, где автоматический выключатель защищает электрическую цепь от перегрузки.

Паттерн был популяризирован Майклом Найгардом в книге "Release It!" и стал фундаментальным элементом в построении отказоустойчивых микросервисных архитектур.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Circuit Breaker States                        │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│    ┌──────────┐     failures > threshold    ┌──────────┐        │
│    │  CLOSED  │ ─────────────────────────► │   OPEN   │        │
│    │ (Normal) │                             │ (Fail)   │        │
│    └────▲─────┘                             └────┬─────┘        │
│         │                                        │               │
│         │ success                    timeout     │               │
│         │                                        ▼               │
│         │                              ┌────────────────┐        │
│         └───────────────────────────── │  HALF-OPEN     │        │
│                                        │  (Test)        │        │
│                                        └────────────────┘        │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Ключевые характеристики

### Три состояния Circuit Breaker

1. **Closed (Закрыт)** — нормальная работа
   - Все запросы проходят к целевому сервису
   - Ведётся подсчёт ошибок
   - При превышении порога переход в Open

2. **Open (Открыт)** — сервис недоступен
   - Все запросы немедленно отклоняются
   - Возвращается fallback-ответ или ошибка
   - Запускается таймер ожидания

3. **Half-Open (Полуоткрыт)** — тестовый режим
   - Пропускается ограниченное число запросов
   - При успехе переход в Closed
   - При неудаче возврат в Open

### Ключевые параметры

```python
class CircuitBreakerConfig:
    failure_threshold: int = 5        # Порог ошибок для открытия
    success_threshold: int = 3        # Успехов для закрытия
    timeout: float = 30.0             # Время в Open (секунды)
    half_open_max_calls: int = 3      # Тестовых вызовов в Half-Open
    failure_rate_threshold: float = 0.5  # Процент ошибок
    slow_call_threshold: float = 2.0  # Медленный вызов (секунды)
    slow_call_rate_threshold: float = 0.8  # Процент медленных
```

## Когда использовать

### Идеальные сценарии

1. **Вызовы внешних сервисов**
   - REST API третьих сторон
   - Платёжные шлюзы
   - Сервисы уведомлений

2. **Межсервисная коммуникация**
   - Микросервисы в одном кластере
   - gRPC вызовы между сервисами
   - Message broker consumers

3. **Доступ к базам данных**
   - Реплики для чтения
   - Внешние хранилища данных
   - Cache-серверы

4. **Интеграции с legacy-системами**
   - Нестабильные внутренние API
   - Системы с ограниченными ресурсами

### Когда НЕ использовать

- Локальные операции в памяти
- Операции с гарантированной доступностью
- Критически важные операции без fallback
- Синхронные транзакции с откатом

## Преимущества и недостатки

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| **Fail-Fast** | Быстрый отказ вместо ожидания timeout |
| **Защита ресурсов** | Освобождение потоков и соединений |
| **Предотвращение каскадных сбоев** | Изоляция неисправного сервиса |
| **Самовосстановление** | Автоматическое восстановление работы |
| **Мониторинг** | Видимость состояния зависимостей |
| **Graceful degradation** | Плавная деградация функциональности |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| **Сложность настройки** | Подбор правильных threshold'ов |
| **Ложные срабатывания** | При неправильной конфигурации |
| **Дополнительная задержка** | Overhead на проверку состояния |
| **Сложность тестирования** | Нужны интеграционные тесты |
| **Скрытие проблем** | Может маскировать системные issues |

## Примеры реализации

### Python: Простая реализация

```python
import time
import threading
from enum import Enum
from typing import Callable, Any, Optional
from functools import wraps
from dataclasses import dataclass, field
from collections import deque


class CircuitState(Enum):
    CLOSED = "closed"
    OPEN = "open"
    HALF_OPEN = "half_open"


@dataclass
class CircuitBreakerConfig:
    """Конфигурация Circuit Breaker."""
    failure_threshold: int = 5
    success_threshold: int = 3
    timeout: float = 30.0
    half_open_max_calls: int = 3
    exceptions: tuple = (Exception,)


@dataclass
class CircuitBreakerStats:
    """Статистика Circuit Breaker."""
    failures: int = 0
    successes: int = 0
    total_calls: int = 0
    rejected_calls: int = 0
    last_failure_time: Optional[float] = None
    state_changes: list = field(default_factory=list)


class CircuitBreaker:
    """
    Circuit Breaker реализация.

    Пример использования:
        cb = CircuitBreaker(config=CircuitBreakerConfig(failure_threshold=3))

        @cb
        def call_external_service():
            return requests.get("https://api.example.com/data")
    """

    def __init__(self, config: CircuitBreakerConfig = None):
        self.config = config or CircuitBreakerConfig()
        self._state = CircuitState.CLOSED
        self._stats = CircuitBreakerStats()
        self._lock = threading.RLock()
        self._half_open_calls = 0
        self._half_open_successes = 0

    @property
    def state(self) -> CircuitState:
        with self._lock:
            self._check_state_transition()
            return self._state

    def _check_state_transition(self):
        """Проверяет необходимость смены состояния."""
        if self._state == CircuitState.OPEN:
            if self._should_attempt_reset():
                self._transition_to(CircuitState.HALF_OPEN)

    def _should_attempt_reset(self) -> bool:
        """Проверяет, прошло ли достаточно времени для попытки восстановления."""
        if self._stats.last_failure_time is None:
            return True
        return time.time() - self._stats.last_failure_time >= self.config.timeout

    def _transition_to(self, new_state: CircuitState):
        """Переводит Circuit Breaker в новое состояние."""
        old_state = self._state
        self._state = new_state
        self._stats.state_changes.append({
            "from": old_state.value,
            "to": new_state.value,
            "timestamp": time.time()
        })

        if new_state == CircuitState.HALF_OPEN:
            self._half_open_calls = 0
            self._half_open_successes = 0
        elif new_state == CircuitState.CLOSED:
            self._stats.failures = 0

    def _record_success(self):
        """Записывает успешный вызов."""
        with self._lock:
            self._stats.successes += 1
            self._stats.total_calls += 1

            if self._state == CircuitState.HALF_OPEN:
                self._half_open_successes += 1
                if self._half_open_successes >= self.config.success_threshold:
                    self._transition_to(CircuitState.CLOSED)

    def _record_failure(self, exception: Exception):
        """Записывает неудачный вызов."""
        with self._lock:
            self._stats.failures += 1
            self._stats.total_calls += 1
            self._stats.last_failure_time = time.time()

            if self._state == CircuitState.HALF_OPEN:
                self._transition_to(CircuitState.OPEN)
            elif self._state == CircuitState.CLOSED:
                if self._stats.failures >= self.config.failure_threshold:
                    self._transition_to(CircuitState.OPEN)

    def _can_execute(self) -> bool:
        """Проверяет, можно ли выполнить вызов."""
        with self._lock:
            if self._state == CircuitState.CLOSED:
                return True
            elif self._state == CircuitState.OPEN:
                if self._should_attempt_reset():
                    self._transition_to(CircuitState.HALF_OPEN)
                    return True
                return False
            else:  # HALF_OPEN
                if self._half_open_calls < self.config.half_open_max_calls:
                    self._half_open_calls += 1
                    return True
                return False

    def call(self, func: Callable, *args, **kwargs) -> Any:
        """Выполняет функцию через Circuit Breaker."""
        if not self._can_execute():
            self._stats.rejected_calls += 1
            raise CircuitBreakerOpenError(
                f"Circuit breaker is {self._state.value}"
            )

        try:
            result = func(*args, **kwargs)
            self._record_success()
            return result
        except self.config.exceptions as e:
            self._record_failure(e)
            raise

    def __call__(self, func: Callable) -> Callable:
        """Декоратор для функций."""
        @wraps(func)
        def wrapper(*args, **kwargs):
            return self.call(func, *args, **kwargs)
        return wrapper

    def get_stats(self) -> dict:
        """Возвращает статистику."""
        with self._lock:
            return {
                "state": self._state.value,
                "failures": self._stats.failures,
                "successes": self._stats.successes,
                "total_calls": self._stats.total_calls,
                "rejected_calls": self._stats.rejected_calls
            }


class CircuitBreakerOpenError(Exception):
    """Исключение при открытом Circuit Breaker."""
    pass
```

### Использование с fallback

```python
import requests
from typing import Optional


class UserService:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker(
            config=CircuitBreakerConfig(
                failure_threshold=3,
                timeout=60.0,
                exceptions=(requests.RequestException,)
            )
        )
        self._cache = {}  # Простой кеш для fallback

    def get_user(self, user_id: str) -> dict:
        """Получает пользователя с fallback на кеш."""
        try:
            return self._get_user_from_api(user_id)
        except CircuitBreakerOpenError:
            return self._get_user_fallback(user_id)
        except requests.RequestException:
            return self._get_user_fallback(user_id)

    @circuit_breaker
    def _get_user_from_api(self, user_id: str) -> dict:
        """Запрос к внешнему API."""
        response = requests.get(
            f"https://api.users.com/users/{user_id}",
            timeout=5.0
        )
        response.raise_for_status()
        user = response.json()
        self._cache[user_id] = user  # Обновляем кеш
        return user

    def _get_user_fallback(self, user_id: str) -> dict:
        """Fallback: возврат из кеша или дефолтные данные."""
        if user_id in self._cache:
            return {**self._cache[user_id], "_cached": True}
        return {
            "id": user_id,
            "name": "Unknown",
            "status": "unavailable",
            "_cached": True
        }
```

### Resilience4j (Java/Kotlin)

```java
import io.github.resilience4j.circuitbreaker.CircuitBreaker;
import io.github.resilience4j.circuitbreaker.CircuitBreakerConfig;
import io.github.resilience4j.circuitbreaker.CircuitBreakerRegistry;

import java.time.Duration;

public class PaymentService {

    private final CircuitBreaker circuitBreaker;
    private final PaymentGateway gateway;

    public PaymentService(PaymentGateway gateway) {
        this.gateway = gateway;

        CircuitBreakerConfig config = CircuitBreakerConfig.custom()
            .failureRateThreshold(50)                     // 50% ошибок
            .slowCallRateThreshold(80)                    // 80% медленных
            .slowCallDurationThreshold(Duration.ofSeconds(2))
            .waitDurationInOpenState(Duration.ofSeconds(60))
            .permittedNumberOfCallsInHalfOpenState(3)
            .minimumNumberOfCalls(10)
            .slidingWindowType(CircuitBreakerConfig.SlidingWindowType.COUNT_BASED)
            .slidingWindowSize(10)
            .build();

        CircuitBreakerRegistry registry = CircuitBreakerRegistry.of(config);
        this.circuitBreaker = registry.circuitBreaker("paymentService");

        // Event listeners для мониторинга
        circuitBreaker.getEventPublisher()
            .onStateTransition(event ->
                log.info("Circuit breaker state: {} -> {}",
                    event.getStateTransition().getFromState(),
                    event.getStateTransition().getToState()));
    }

    public PaymentResult processPayment(Payment payment) {
        return circuitBreaker.executeSupplier(() -> {
            return gateway.process(payment);
        });
    }

    public PaymentResult processPaymentWithFallback(Payment payment) {
        return CircuitBreaker.decorateSupplier(circuitBreaker,
            () -> gateway.process(payment))
            .recover(throwable -> fallbackPayment(payment))
            .get();
    }

    private PaymentResult fallbackPayment(Payment payment) {
        // Альтернативный процессинг или отложенная обработка
        return new PaymentResult(PaymentStatus.PENDING,
            "Payment queued for later processing");
    }
}
```

### Go: Реализация с Hystrix-style

```go
package circuitbreaker

import (
    "errors"
    "sync"
    "time"
)

type State int

const (
    StateClosed State = iota
    StateOpen
    StateHalfOpen
)

var ErrCircuitOpen = errors.New("circuit breaker is open")

type CircuitBreaker struct {
    mu sync.RWMutex

    state            State
    failures         int
    successes        int
    failureThreshold int
    successThreshold int
    timeout          time.Duration
    lastFailure      time.Time
    halfOpenCalls    int
    maxHalfOpenCalls int
}

func New(failureThreshold, successThreshold int, timeout time.Duration) *CircuitBreaker {
    return &CircuitBreaker{
        state:            StateClosed,
        failureThreshold: failureThreshold,
        successThreshold: successThreshold,
        timeout:          timeout,
        maxHalfOpenCalls: 3,
    }
}

func (cb *CircuitBreaker) Execute(fn func() error) error {
    if !cb.canExecute() {
        return ErrCircuitOpen
    }

    err := fn()

    if err != nil {
        cb.recordFailure()
        return err
    }

    cb.recordSuccess()
    return nil
}

func (cb *CircuitBreaker) canExecute() bool {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    switch cb.state {
    case StateClosed:
        return true
    case StateOpen:
        if time.Since(cb.lastFailure) >= cb.timeout {
            cb.toHalfOpen()
            return true
        }
        return false
    case StateHalfOpen:
        if cb.halfOpenCalls < cb.maxHalfOpenCalls {
            cb.halfOpenCalls++
            return true
        }
        return false
    }
    return false
}

func (cb *CircuitBreaker) recordSuccess() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.successes++

    if cb.state == StateHalfOpen {
        if cb.successes >= cb.successThreshold {
            cb.toClosed()
        }
    }
}

func (cb *CircuitBreaker) recordFailure() {
    cb.mu.Lock()
    defer cb.mu.Unlock()

    cb.failures++
    cb.lastFailure = time.Now()

    switch cb.state {
    case StateClosed:
        if cb.failures >= cb.failureThreshold {
            cb.toOpen()
        }
    case StateHalfOpen:
        cb.toOpen()
    }
}

func (cb *CircuitBreaker) toOpen() {
    cb.state = StateOpen
    cb.halfOpenCalls = 0
    cb.successes = 0
}

func (cb *CircuitBreaker) toHalfOpen() {
    cb.state = StateHalfOpen
    cb.failures = 0
    cb.successes = 0
    cb.halfOpenCalls = 0
}

func (cb *CircuitBreaker) toClosed() {
    cb.state = StateClosed
    cb.failures = 0
    cb.successes = 0
}

func (cb *CircuitBreaker) State() State {
    cb.mu.RLock()
    defer cb.mu.RUnlock()
    return cb.state
}
```

## Best Practices и антипаттерны

### Best Practices

#### 1. Правильная настройка threshold'ов

```python
# Хорошо: настройка на основе SLA и метрик
config = CircuitBreakerConfig(
    failure_threshold=5,           # Быстрое открытие при серьёзных проблемах
    timeout=30.0,                  # Даём время на восстановление
    success_threshold=3,           # Требуем стабильности перед закрытием
)

# Плохо: слишком чувствительный или слишком медленный
bad_config = CircuitBreakerConfig(
    failure_threshold=1,           # Одна ошибка = открытие (слишком чувствительно)
    timeout=300.0,                 # 5 минут ожидания (слишком долго)
    success_threshold=1,           # Закрытие после 1 успеха (ненадёжно)
)
```

#### 2. Разные Circuit Breaker для разных зависимостей

```python
class OrderService:
    def __init__(self):
        # Критичный сервис: быстрое восстановление
        self.payment_cb = CircuitBreaker(
            config=CircuitBreakerConfig(
                failure_threshold=3,
                timeout=15.0
            )
        )

        # Менее критичный: можем ждать дольше
        self.notification_cb = CircuitBreaker(
            config=CircuitBreakerConfig(
                failure_threshold=10,
                timeout=60.0
            )
        )
```

#### 3. Мониторинг и алерты

```python
def setup_circuit_breaker_monitoring(cb: CircuitBreaker, service_name: str):
    """Настройка мониторинга для Circuit Breaker."""

    def on_state_change(old_state: CircuitState, new_state: CircuitState):
        metrics.increment(
            "circuit_breaker.state_change",
            tags={
                "service": service_name,
                "from": old_state.value,
                "to": new_state.value
            }
        )

        if new_state == CircuitState.OPEN:
            alerting.send_alert(
                severity="warning",
                message=f"Circuit breaker opened for {service_name}"
            )

    cb.on_state_change = on_state_change
```

#### 4. Graceful fallback стратегии

```python
class ProductCatalog:
    def get_product(self, product_id: str) -> Product:
        try:
            return self._fetch_from_main_service(product_id)
        except CircuitBreakerOpenError:
            # Уровень 1: Кеш
            if cached := self._get_from_cache(product_id):
                return cached

            # Уровень 2: Резервный сервис
            try:
                return self._fetch_from_backup_service(product_id)
            except Exception:
                pass

            # Уровень 3: Статические данные
            return self._get_default_product(product_id)
```

### Антипаттерны

#### 1. Общий Circuit Breaker для всех сервисов

```python
# Плохо: один CB на все зависимости
class BadService:
    shared_cb = CircuitBreaker()  # Одна проблема блокирует всё

    def call_service_a(self):
        return self.shared_cb.call(self._service_a)

    def call_service_b(self):
        return self.shared_cb.call(self._service_b)  # Заблокирован из-за A
```

#### 2. Игнорирование метрик

```python
# Плохо: нет наблюдаемости
def process():
    try:
        return circuit_breaker.call(external_api)
    except CircuitBreakerOpenError:
        pass  # Тихий fallback без логирования
```

#### 3. Неправильная обработка Half-Open

```python
# Плохо: слишком много тестовых запросов
config = CircuitBreakerConfig(
    half_open_max_calls=100,  # DDoS на восстанавливающийся сервис
)
```

## Связанные паттерны

### Комбинация с Retry

```
Request → Retry → Circuit Breaker → Service
          ↓              ↓
       Backoff      Fail-Fast
```

```python
from tenacity import retry, stop_after_attempt, wait_exponential

@retry(
    stop=stop_after_attempt(3),
    wait=wait_exponential(multiplier=1, min=1, max=10)
)
@circuit_breaker
def call_service():
    return requests.get("https://api.example.com")
```

### Комбинация с Bulkhead

```python
class ResilientService:
    def __init__(self):
        self.circuit_breaker = CircuitBreaker()
        self.semaphore = threading.Semaphore(10)  # Bulkhead

    def call(self):
        if not self.semaphore.acquire(blocking=False):
            raise BulkheadFullError()
        try:
            return self.circuit_breaker.call(self._external_call)
        finally:
            self.semaphore.release()
```

### Связанные паттерны

| Паттерн | Связь с Circuit Breaker |
|---------|-------------------------|
| **Retry** | CB предотвращает бесконечные retry |
| **Bulkhead** | Изоляция ресурсов + CB для защиты |
| **Timeout** | CB реагирует на timeout как на failure |
| **Fallback** | CB активирует fallback стратегии |
| **Health Check** | Может влиять на состояние CB |

## Ресурсы для изучения

### Книги
- "Release It!" — Michael Nygard
- "Building Microservices" — Sam Newman
- "Microservices Patterns" — Chris Richardson

### Библиотеки
- **Python**: `pybreaker`, `circuitbreaker`
- **Java**: `Resilience4j`, `Hystrix` (deprecated)
- **Go**: `sony/gobreaker`, `afex/hystrix-go`
- **JavaScript**: `opossum`, `cockatiel`

### Статьи и документация
- [Martin Fowler: Circuit Breaker](https://martinfowler.com/bliki/CircuitBreaker.html)
- [Microsoft: Circuit Breaker Pattern](https://docs.microsoft.com/en-us/azure/architecture/patterns/circuit-breaker)
- [Resilience4j Documentation](https://resilience4j.readme.io/docs/circuitbreaker)

### Видео
- "Building Resilient Microservices" — Nygard at QCon
- "Fault Tolerance in Distributed Systems" — various conferences
