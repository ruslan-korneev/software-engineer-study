# Основы логирования

## Введение

**Логирование** (logging) — это процесс записи информации о событиях, происходящих в приложении, для последующего анализа, отладки и мониторинга. Логи являются одним из трёх столпов observability (наблюдаемости), наряду с метриками и трассировкой.

## Зачем нужно логирование?

### Основные цели

1. **Отладка (Debugging)** — поиск и исправление ошибок в коде
2. **Мониторинг** — отслеживание состояния системы в реальном времени
3. **Аудит** — запись действий пользователей и системы для compliance
4. **Аналитика** — анализ поведения пользователей и производительности
5. **Инцидент-менеджмент** — расследование проблем после их возникновения

## Уровни логирования (Log Levels)

Стандартные уровни логирования по возрастанию важности:

| Уровень | Описание | Когда использовать |
|---------|----------|-------------------|
| **TRACE** | Максимально детальная информация | Отладка алгоритмов, пошаговое выполнение |
| **DEBUG** | Отладочная информация | Разработка и тестирование |
| **INFO** | Информационные сообщения | Важные события в нормальном потоке работы |
| **WARN** | Предупреждения | Потенциальные проблемы, которые не прерывают работу |
| **ERROR** | Ошибки | Проблемы, влияющие на функциональность |
| **FATAL/CRITICAL** | Критические ошибки | Ошибки, приводящие к остановке приложения |

### Правила выбора уровня

```python
# TRACE - детальная трассировка
logger.trace(f"Entering function calculate_price with params: {params}")

# DEBUG - отладочная информация
logger.debug(f"Cache hit for key: {cache_key}")

# INFO - важные бизнес-события
logger.info(f"User {user_id} successfully logged in")

# WARN - предупреждения
logger.warning(f"Retry attempt {attempt}/3 for external API call")

# ERROR - ошибки
logger.error(f"Failed to process payment for order {order_id}: {error}")

# CRITICAL - критические ошибки
logger.critical(f"Database connection lost, shutting down")
```

## Структурированное логирование

### Проблема текстовых логов

Традиционные текстовые логи сложно парсить и анализировать:

```
2024-01-15 10:23:45 ERROR Failed to process order 12345 for user john@example.com
```

### Решение: JSON-логи

Структурированные логи в формате JSON легко обрабатывать программно:

```json
{
  "timestamp": "2024-01-15T10:23:45.123Z",
  "level": "ERROR",
  "message": "Failed to process order",
  "order_id": 12345,
  "user_email": "john@example.com",
  "error_type": "PaymentDeclined",
  "service": "order-service",
  "trace_id": "abc123def456"
}
```

## Примеры реализации

### Python с structlog

```python
import structlog
from datetime import datetime

# Настройка structlog
structlog.configure(
    processors=[
        structlog.stdlib.filter_by_level,
        structlog.stdlib.add_logger_name,
        structlog.stdlib.add_log_level,
        structlog.stdlib.PositionalArgumentsFormatter(),
        structlog.processors.TimeStamper(fmt="iso"),
        structlog.processors.StackInfoRenderer(),
        structlog.processors.format_exc_info,
        structlog.processors.UnicodeDecoder(),
        structlog.processors.JSONRenderer()
    ],
    wrapper_class=structlog.stdlib.BoundLogger,
    context_class=dict,
    logger_factory=structlog.stdlib.LoggerFactory(),
    cache_logger_on_first_use=True,
)

logger = structlog.get_logger()

# Использование
def process_order(order_id: int, user_id: int):
    log = logger.bind(order_id=order_id, user_id=user_id)

    log.info("processing_order_started")

    try:
        # Бизнес-логика
        result = do_processing(order_id)
        log.info("processing_order_completed", result=result)
        return result
    except PaymentError as e:
        log.error("payment_failed", error=str(e), error_type=type(e).__name__)
        raise
    except Exception as e:
        log.exception("unexpected_error")
        raise
```

### Python с стандартным logging + python-json-logger

```python
import logging
from pythonjsonlogger import jsonlogger
import sys

# Настройка JSON formatter
class CustomJsonFormatter(jsonlogger.JsonFormatter):
    def add_fields(self, log_record, record, message_dict):
        super().add_fields(log_record, record, message_dict)
        log_record['timestamp'] = datetime.utcnow().isoformat()
        log_record['level'] = record.levelname
        log_record['service'] = 'my-service'

# Настройка логгера
logger = logging.getLogger()
handler = logging.StreamHandler(sys.stdout)
formatter = CustomJsonFormatter('%(timestamp)s %(level)s %(message)s')
handler.setFormatter(formatter)
logger.addHandler(handler)
logger.setLevel(logging.INFO)

# Использование с дополнительным контекстом
logger.info("User action", extra={
    'user_id': 123,
    'action': 'login',
    'ip_address': '192.168.1.1'
})
```

### Go с zerolog

```go
package main

import (
    "os"
    "time"

    "github.com/rs/zerolog"
    "github.com/rs/zerolog/log"
)

func init() {
    // Настройка глобального логгера
    zerolog.TimeFieldFormat = time.RFC3339Nano

    // Production: JSON логи
    log.Logger = zerolog.New(os.Stdout).
        With().
        Timestamp().
        Str("service", "order-service").
        Logger()
}

func ProcessOrder(orderID int, userID int) error {
    // Создаём логгер с контекстом
    logger := log.With().
        Int("order_id", orderID).
        Int("user_id", userID).
        Logger()

    logger.Info().Msg("processing order started")

    // Симуляция обработки
    if err := validateOrder(orderID); err != nil {
        logger.Error().
            Err(err).
            Str("step", "validation").
            Msg("order validation failed")
        return err
    }

    logger.Info().
        Str("status", "completed").
        Dur("duration", time.Since(start)).
        Msg("processing order completed")

    return nil
}
```

### Go с zap (высокопроизводительный логгер)

```go
package main

import (
    "go.uber.org/zap"
    "go.uber.org/zap/zapcore"
)

func initLogger() *zap.Logger {
    config := zap.NewProductionConfig()
    config.EncoderConfig.TimeKey = "timestamp"
    config.EncoderConfig.EncodeTime = zapcore.ISO8601TimeEncoder

    logger, _ := config.Build()
    return logger
}

func main() {
    logger := initLogger()
    defer logger.Sync()

    // Добавляем контекст
    logger = logger.With(
        zap.String("service", "payment-service"),
        zap.String("version", "1.2.3"),
    )

    // Использование
    logger.Info("processing payment",
        zap.Int("order_id", 12345),
        zap.String("currency", "USD"),
        zap.Float64("amount", 99.99),
    )

    // Sugar logger для более простого синтаксиса
    sugar := logger.Sugar()
    sugar.Infow("payment processed",
        "order_id", 12345,
        "status", "success",
    )
}
```

## Correlation ID и распределённая трассировка

### Что такое Correlation ID?

**Correlation ID** (идентификатор корреляции) — уникальный идентификатор, который передаётся между всеми сервисами при обработке одного запроса. Позволяет связать логи разных сервисов.

### Реализация в Python (FastAPI)

```python
import uuid
from contextvars import ContextVar
from fastapi import FastAPI, Request
from starlette.middleware.base import BaseHTTPMiddleware
import structlog

# Context variable для хранения correlation ID
correlation_id_ctx: ContextVar[str] = ContextVar('correlation_id', default='')

class CorrelationIdMiddleware(BaseHTTPMiddleware):
    async def dispatch(self, request: Request, call_next):
        # Получаем или генерируем correlation ID
        correlation_id = request.headers.get('X-Correlation-ID', str(uuid.uuid4()))
        correlation_id_ctx.set(correlation_id)

        # Добавляем в логгер
        structlog.contextvars.bind_contextvars(correlation_id=correlation_id)

        response = await call_next(request)
        response.headers['X-Correlation-ID'] = correlation_id

        return response

app = FastAPI()
app.add_middleware(CorrelationIdMiddleware)

@app.get("/orders/{order_id}")
async def get_order(order_id: int):
    logger = structlog.get_logger()
    logger.info("fetching_order", order_id=order_id)
    # Все логи будут содержать correlation_id
    return {"order_id": order_id}
```

### Реализация в Go

```go
package middleware

import (
    "context"
    "net/http"

    "github.com/google/uuid"
    "github.com/rs/zerolog/log"
)

type correlationKey struct{}

func CorrelationMiddleware(next http.Handler) http.Handler {
    return http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
        correlationID := r.Header.Get("X-Correlation-ID")
        if correlationID == "" {
            correlationID = uuid.New().String()
        }

        // Добавляем в контекст
        ctx := context.WithValue(r.Context(), correlationKey{}, correlationID)

        // Добавляем в логгер
        logger := log.With().Str("correlation_id", correlationID).Logger()
        ctx = logger.WithContext(ctx)

        w.Header().Set("X-Correlation-ID", correlationID)
        next.ServeHTTP(w, r.WithContext(ctx))
    })
}

// Получение correlation ID из контекста
func GetCorrelationID(ctx context.Context) string {
    if id, ok := ctx.Value(correlationKey{}).(string); ok {
        return id
    }
    return ""
}
```

## Best Practices

### 1. Что логировать

**Обязательно логировать:**
- Входящие и исходящие запросы (HTTP, gRPC, очереди)
- Ошибки и исключения с полным стектрейсом
- Важные бизнес-события (регистрация, оплата, заказ)
- Метрики производительности (время выполнения критических операций)
- Изменения состояния системы (старт, остановка, конфигурация)

**Не логировать:**
- Пароли, токены, ключи API
- Персональные данные (PII) — или маскировать
- Данные банковских карт (PCI DSS)
- Медицинские данные (HIPAA)

### 2. Маскирование чувствительных данных

```python
import re

def mask_sensitive_data(data: dict) -> dict:
    """Маскирует чувствительные данные в логах"""
    sensitive_keys = ['password', 'token', 'api_key', 'secret', 'credit_card']

    masked = data.copy()
    for key, value in masked.items():
        if any(s in key.lower() for s in sensitive_keys):
            masked[key] = '***MASKED***'
        elif isinstance(value, str):
            # Маскируем email
            masked[key] = re.sub(
                r'(\w{2})\w+@\w+(\.\w+)',
                r'\1***@***\2',
                value
            )
    return masked
```

### 3. Правила именования

```python
# Плохо
logger.info("done")
logger.error("error occurred")

# Хорошо - используем snake_case для event names
logger.info("order_processing_completed", order_id=123, duration_ms=450)
logger.error("payment_validation_failed", order_id=123, error="insufficient_funds")
```

### 4. Контекстное логирование

```python
# Создаём логгер с базовым контекстом для всего запроса
def handle_request(request):
    log = logger.bind(
        request_id=request.id,
        user_id=request.user.id,
        endpoint=request.path,
    )

    # Все последующие логи будут содержать этот контекст
    log.info("request_received")

    result = process_request(request, log)

    log.info("request_completed", status=result.status)
```

## Распространённые ошибки

### 1. Логирование внутри циклов

```python
# Плохо - создаёт огромное количество логов
for item in items:
    logger.debug(f"Processing item {item.id}")
    process(item)

# Хорошо - логируем агрегированную информацию
logger.info("batch_processing_started", total_items=len(items))
for item in items:
    process(item)
logger.info("batch_processing_completed", processed=len(items))
```

### 2. Форматирование строк вместо параметров

```python
# Плохо - строка форматируется даже если лог не выводится
logger.debug(f"Processing order {order.to_dict()}")

# Хорошо - ленивое вычисление
logger.debug("processing_order", order_id=order.id)
```

### 3. Отсутствие структуры

```python
# Плохо
logger.error(f"Failed to process order {order_id} for user {user_id}: {error}")

# Хорошо
logger.error("order_processing_failed",
    order_id=order_id,
    user_id=user_id,
    error=str(error),
    error_type=type(error).__name__
)
```

### 4. Игнорирование уровней логирования

```python
# Плохо - всё на одном уровне
logger.info("Entering function")
logger.info("DB query executed")
logger.info("User not found")
logger.info("Application crashed!")

# Хорошо - правильные уровни
logger.debug("entering_function", function="process_order")
logger.debug("db_query_executed", query_time_ms=45)
logger.warning("user_not_found", user_id=123)
logger.critical("application_crashed", error="OutOfMemory")
```

## Настройка уровней для production

```python
# config.py
import os

LOG_LEVELS = {
    'development': 'DEBUG',
    'staging': 'INFO',
    'production': 'WARNING',
}

LOG_LEVEL = LOG_LEVELS.get(os.environ.get('ENVIRONMENT', 'development'), 'INFO')
```

## Ротация логов

При записи в файлы необходимо настроить ротацию:

```python
from logging.handlers import RotatingFileHandler, TimedRotatingFileHandler

# По размеру файла (5MB, храним 5 файлов)
handler = RotatingFileHandler(
    'app.log',
    maxBytes=5*1024*1024,
    backupCount=5
)

# По времени (ежедневно, храним 7 дней)
handler = TimedRotatingFileHandler(
    'app.log',
    when='midnight',
    interval=1,
    backupCount=7
)
```

## Вывод

Правильное логирование — это фундамент для observability системы. Структурированные логи с correlation ID позволяют эффективно отлаживать распределённые системы и быстро находить причины проблем. В следующем разделе мы рассмотрим, как собирать и анализировать логи с помощью Grafana Loki.
