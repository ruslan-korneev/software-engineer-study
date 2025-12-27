# Synchronous vs Asynchronous APIs

## Введение

При проектировании API одним из ключевых архитектурных решений является выбор между синхронным и асинхронным подходом к обработке запросов. Этот выбор существенно влияет на производительность, масштабируемость и пользовательский опыт.

## Синхронные API (Synchronous APIs)

### Определение

Синхронный API — это модель взаимодействия, при которой клиент отправляет запрос и **блокируется** до получения ответа. Клиент не может продолжать работу, пока не получит результат.

### Архитектурная диаграмма

```
┌─────────┐     Запрос      ┌─────────┐     Запрос      ┌──────────┐
│         │ ──────────────> │         │ ──────────────> │          │
│ Клиент  │   (блокировка)  │   API   │   (обработка)   │ Database │
│         │ <────────────── │ Server  │ <────────────── │          │
└─────────┘     Ответ       └─────────┘     Ответ       └──────────┘

Время: ────────────────────────────────────────────────────────────>
       │<──── Клиент заблокирован ────>│
```

### Пример кода: Синхронный REST API

```python
# Flask синхронный endpoint
from flask import Flask, jsonify
import time

app = Flask(__name__)

@app.route('/api/process-payment', methods=['POST'])
def process_payment():
    """
    Синхронная обработка платежа.
    Клиент ждёт завершения всех операций.
    """
    # Валидация данных (100ms)
    validate_payment_data()

    # Проверка баланса (200ms)
    check_balance()

    # Обработка платежа (500ms)
    result = execute_payment()

    # Обновление базы данных (100ms)
    update_database(result)

    # Общее время: ~900ms — клиент заблокирован
    return jsonify({
        'status': 'success',
        'transaction_id': result.id
    })
```

```python
# Клиентский код
import requests

# Клиент блокируется на ~900ms
response = requests.post(
    'http://api.example.com/api/process-payment',
    json={'amount': 100, 'currency': 'USD'},
    timeout=30  # Важно устанавливать timeout
)
print(response.json())
```

### Преимущества синхронных API

| Преимущество | Описание |
|--------------|----------|
| Простота | Легко понять и реализовать |
| Предсказуемость | Результат известен сразу |
| Отладка | Проще отслеживать ошибки |
| Транзакционность | Гарантия целостности операции |

### Недостатки синхронных API

| Недостаток | Описание |
|------------|----------|
| Блокировка | Клиент ждёт завершения |
| Масштабируемость | Ограничена количеством потоков |
| Устойчивость | Сбой = потеря запроса |
| Timeout | Риск истечения времени ожидания |

---

## Асинхронные API (Asynchronous APIs)

### Определение

Асинхронный API — это модель взаимодействия, при которой клиент отправляет запрос и **сразу получает подтверждение**. Результат обработки получается позже через callback, polling или другой механизм.

### Архитектурная диаграмма

```
┌─────────┐   1. Запрос    ┌─────────┐   2. В очередь   ┌─────────┐
│         │ ─────────────> │   API   │ ───────────────> │  Queue  │
│ Клиент  │                │ Server  │                  │         │
│         │ <───────────── └────┬────┘                  └────┬────┘
└────┬────┘  Accepted (202)     │                            │
     │                          │                            │
     │                          │   3. Обработка             │
     │                          │ <──────────────────────────┘
     │                          │                    ┌──────────┐
     │                          │ ──────────────────>│  Worker  │
     │                          │                    └────┬─────┘
     │   4. Webhook/Polling     │                         │
     │ <────────────────────────┼─────────────────────────┘
     │                          │
     ▼                          ▼
Продолжает                 Обрабатывает
работу                     другие запросы
```

### Паттерны асинхронного взаимодействия

#### 1. Request-Acknowledge (Запрос-Подтверждение)

```python
# FastAPI асинхронный endpoint
from fastapi import FastAPI, BackgroundTasks
from pydantic import BaseModel
import uuid

app = FastAPI()

class PaymentRequest(BaseModel):
    amount: float
    currency: str

# Хранилище статусов (в продакшене — Redis/Database)
task_status = {}

@app.post('/api/process-payment', status_code=202)
async def process_payment(
    payment: PaymentRequest,
    background_tasks: BackgroundTasks
):
    """
    Асинхронная обработка платежа.
    Мгновенный ответ, обработка в фоне.
    """
    task_id = str(uuid.uuid4())

    # Ставим задачу в очередь на фоновую обработку
    background_tasks.add_task(
        process_payment_background,
        task_id,
        payment
    )

    task_status[task_id] = {'status': 'processing'}

    # Мгновенный ответ (~5ms)
    return {
        'task_id': task_id,
        'status': 'accepted',
        'status_url': f'/api/payment-status/{task_id}'
    }

async def process_payment_background(task_id: str, payment: PaymentRequest):
    """Фоновая обработка платежа."""
    try:
        # Длительные операции...
        await validate_payment_data(payment)
        await check_balance(payment.amount)
        result = await execute_payment(payment)
        await update_database(result)

        task_status[task_id] = {
            'status': 'completed',
            'transaction_id': result.id
        }
    except Exception as e:
        task_status[task_id] = {
            'status': 'failed',
            'error': str(e)
        }

@app.get('/api/payment-status/{task_id}')
async def get_payment_status(task_id: str):
    """Endpoint для проверки статуса задачи."""
    if task_id not in task_status:
        return {'error': 'Task not found'}, 404
    return task_status[task_id]
```

#### 2. Webhook Callback

```python
# Сервер отправляет результат на указанный URL
import httpx

async def process_with_webhook(task_id: str, payment: PaymentRequest, callback_url: str):
    """Обработка с отправкой результата на webhook."""
    try:
        result = await execute_payment(payment)

        # Отправляем результат на webhook клиента
        async with httpx.AsyncClient() as client:
            await client.post(callback_url, json={
                'task_id': task_id,
                'status': 'completed',
                'transaction_id': result.id
            })
    except Exception as e:
        async with httpx.AsyncClient() as client:
            await client.post(callback_url, json={
                'task_id': task_id,
                'status': 'failed',
                'error': str(e)
            })
```

#### 3. Long Polling

```python
import asyncio

@app.get('/api/payment-status/{task_id}/long-poll')
async def long_poll_status(task_id: str, timeout: int = 30):
    """
    Long polling — держим соединение открытым
    до появления результата или timeout.
    """
    start_time = asyncio.get_event_loop().time()

    while True:
        status = task_status.get(task_id)

        if status and status['status'] != 'processing':
            return status

        elapsed = asyncio.get_event_loop().time() - start_time
        if elapsed >= timeout:
            return {'status': 'processing', 'message': 'Still processing'}

        # Проверяем каждые 500ms
        await asyncio.sleep(0.5)
```

### Преимущества асинхронных API

| Преимущество | Описание |
|--------------|----------|
| Отзывчивость | Мгновенный ответ клиенту |
| Масштабируемость | Легко обрабатывать больше запросов |
| Устойчивость | Повторная обработка при сбоях |
| Развязка | Клиент и сервер независимы |

### Недостатки асинхронных API

| Недостаток | Описание |
|------------|----------|
| Сложность | Требует дополнительной инфраструктуры |
| Отладка | Сложнее отслеживать поток выполнения |
| Eventual Consistency | Данные не сразу согласованы |
| Идемпотентность | Нужно обрабатывать дубликаты |

---

## Сравнительная таблица

| Критерий | Синхронный API | Асинхронный API |
|----------|----------------|-----------------|
| Время ответа | Полное время обработки | Мгновенное подтверждение |
| Сложность клиента | Низкая | Средняя/Высокая |
| Сложность сервера | Низкая | Высокая |
| Масштабируемость | Ограниченная | Высокая |
| Retry логика | Простая | Встроенная |
| Use cases | CRUD, простые операции | Платежи, отчёты, интеграции |

---

## Когда использовать

### Синхронный API подходит для:

- **Быстрых операций** (< 500ms)
- **CRUD операций** с базой данных
- **Аутентификации и авторизации**
- **Получения данных** (GET запросы)
- **Валидации** в реальном времени

### Асинхронный API подходит для:

- **Долгих операций** (> 1-2 секунды)
- **Обработки платежей**
- **Генерации отчётов**
- **Отправки email/SMS**
- **Интеграции с внешними сервисами**
- **Batch операций**
- **Загрузки/обработки файлов**

---

## Best Practices

### Для синхронных API

```python
# 1. Всегда устанавливайте timeout
response = requests.get(url, timeout=(3.05, 27))

# 2. Используйте connection pooling
from requests.adapters import HTTPAdapter
from urllib3.util.retry import Retry

session = requests.Session()
retries = Retry(total=3, backoff_factor=0.1)
session.mount('http://', HTTPAdapter(max_retries=retries))

# 3. Ограничивайте время выполнения на сервере
from functools import wraps
import signal

def timeout(seconds):
    def decorator(func):
        @wraps(func)
        def wrapper(*args, **kwargs):
            signal.alarm(seconds)
            try:
                return func(*args, **kwargs)
            finally:
                signal.alarm(0)
        return wrapper
    return decorator
```

### Для асинхронных API

```python
# 1. Реализуйте идемпотентность
@app.post('/api/process-payment')
async def process_payment(payment: PaymentRequest, idempotency_key: str):
    # Проверяем, не обрабатывали ли мы уже этот запрос
    existing = await redis.get(f'idempotency:{idempotency_key}')
    if existing:
        return json.loads(existing)

    result = await process_payment_internal(payment)

    # Сохраняем результат для повторных запросов
    await redis.setex(
        f'idempotency:{idempotency_key}',
        86400,  # 24 часа
        json.dumps(result)
    )
    return result

# 2. Реализуйте механизм отслеживания статуса
# 3. Добавляйте retry с exponential backoff
# 4. Логируйте correlation_id для трассировки
```

---

## Типичные ошибки

### 1. Синхронный вызов для долгих операций

```python
# ❌ Плохо: клиент ждёт 30 секунд
@app.post('/api/generate-report')
def generate_report():
    report = create_huge_report()  # 30 секунд
    return report

# ✅ Хорошо: асинхронная генерация
@app.post('/api/generate-report', status_code=202)
async def generate_report(background_tasks: BackgroundTasks):
    task_id = create_task_id()
    background_tasks.add_task(create_huge_report, task_id)
    return {'task_id': task_id, 'status_url': f'/api/reports/{task_id}'}
```

### 2. Отсутствие timeout

```python
# ❌ Плохо: бесконечное ожидание
response = requests.get(external_api_url)

# ✅ Хорошо: с timeout
response = requests.get(external_api_url, timeout=10)
```

### 3. Игнорирование идемпотентности

```python
# ❌ Плохо: повторный запрос создаст дубликат
@app.post('/api/create-order')
async def create_order(order: Order):
    return await db.orders.insert(order)

# ✅ Хорошо: проверка на дубликат
@app.post('/api/create-order')
async def create_order(order: Order, idempotency_key: str = Header(...)):
    existing = await db.orders.find_one({'idempotency_key': idempotency_key})
    if existing:
        return existing
    order.idempotency_key = idempotency_key
    return await db.orders.insert(order)
```

---

## Гибридный подход

В реальных системах часто используется комбинация обоих подходов:

```python
@app.post('/api/checkout')
async def checkout(cart: Cart, background_tasks: BackgroundTasks):
    """
    Гибридный подход:
    - Синхронно: валидация, резервирование
    - Асинхронно: оплата, уведомления
    """
    # Синхронная часть — критически важно получить ответ сразу
    validation_result = await validate_cart(cart)
    if not validation_result.success:
        return {'error': validation_result.message}, 400

    reservation = await reserve_items(cart)
    if not reservation.success:
        return {'error': 'Items not available'}, 409

    order_id = await create_order(cart, reservation)

    # Асинхронная часть — можно выполнить в фоне
    background_tasks.add_task(process_payment, order_id)
    background_tasks.add_task(send_confirmation_email, order_id)
    background_tasks.add_task(notify_warehouse, order_id)

    return {
        'order_id': order_id,
        'status': 'processing',
        'message': 'Order created, payment processing'
    }
```

---

## Заключение

Выбор между синхронным и асинхронным API зависит от конкретного use case:

- Используйте **синхронные API** для простых, быстрых операций
- Используйте **асинхронные API** для длительных, ресурсоёмких задач
- Рассмотрите **гибридный подход** для сложных сценариев

Ключевое правило: **не заставляйте пользователя ждать то, чего можно не ждать**.
