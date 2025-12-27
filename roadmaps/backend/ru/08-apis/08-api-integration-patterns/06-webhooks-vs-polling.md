# Webhooks vs Polling

## Введение

При интеграции систем часто возникает задача: как одна система узнает об изменениях в другой? Существуют два основных подхода:

- **Polling** — клиент периодически запрашивает данные
- **Webhooks** — сервер уведомляет клиента о событиях

## Архитектурные диаграммы

### Polling

```
┌────────────────────────────────────────────────────────────────────────┐
│                              POLLING                                    │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Client                              Server                            │
│   ┌─────┐                            ┌─────┐                           │
│   │     │ ──── GET /status ────────> │     │                           │
│   │     │ <─── 200 OK (no change) ── │     │                           │
│   │     │                            │     │                           │
│   │     │   (wait 30 seconds)        │     │                           │
│   │     │                            │     │                           │
│   │     │ ──── GET /status ────────> │     │                           │
│   │     │ <─── 200 OK (no change) ── │     │                           │
│   │     │                            │     │                           │
│   │     │   (wait 30 seconds)        │     │  *** EVENT HAPPENS ***    │
│   │     │                            │     │                           │
│   │     │ ──── GET /status ────────> │     │                           │
│   │     │ <─── 200 OK (new data!) ── │     │                           │
│   └─────┘                            └─────┘                           │
│                                                                         │
│   Клиент постоянно спрашивает: "Есть что-то новое?"                   │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

### Webhooks

```
┌────────────────────────────────────────────────────────────────────────┐
│                              WEBHOOKS                                   │
├────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│   Client                              Server                            │
│   ┌─────┐                            ┌─────┐                           │
│   │     │ ── POST /register-hook ──> │     │                           │
│   │     │ <── 200 OK ─────────────── │     │                           │
│   │     │                            │     │                           │
│   │     │   (client waits...)        │     │                           │
│   │     │                            │     │  *** EVENT HAPPENS ***    │
│   │     │                            │     │                           │
│   │     │ <── POST callback_url ──── │     │                           │
│   │     │ ── 200 OK ───────────────> │     │                           │
│   │     │                            │     │                           │
│   │     │   (instant notification)   │     │                           │
│   └─────┘                            └─────┘                           │
│                                                                         │
│   Сервер сам сообщает: "Произошло событие!"                            │
│                                                                         │
└────────────────────────────────────────────────────────────────────────┘
```

---

## Polling

### Простой Polling

```python
import httpx
import asyncio
from datetime import datetime

class SimplePollingClient:
    def __init__(self, api_url: str, interval: int = 30):
        self.api_url = api_url
        self.interval = interval  # секунды между запросами
        self.last_check = None

    async def poll(self, handler):
        """Простой polling с фиксированным интервалом."""
        async with httpx.AsyncClient() as client:
            while True:
                try:
                    response = await client.get(
                        f'{self.api_url}/updates',
                        params={'since': self.last_check},
                        timeout=10.0
                    )

                    if response.status_code == 200:
                        data = response.json()
                        if data.get('updates'):
                            await handler(data['updates'])

                        self.last_check = datetime.utcnow().isoformat()

                except httpx.RequestError as e:
                    print(f'Polling error: {e}')

                await asyncio.sleep(self.interval)

# Использование
async def handle_updates(updates):
    for update in updates:
        print(f'New update: {update}')

client = SimplePollingClient('https://api.example.com', interval=30)
await client.poll(handle_updates)
```

### Long Polling

Сервер держит соединение открытым до появления данных или timeout.

```python
# Клиент
class LongPollingClient:
    def __init__(self, api_url: str):
        self.api_url = api_url

    async def poll(self, handler):
        """Long polling — сервер отвечает, когда есть данные."""
        async with httpx.AsyncClient() as client:
            last_event_id = None

            while True:
                try:
                    # Длинный timeout — сервер держит соединение
                    response = await client.get(
                        f'{self.api_url}/long-poll',
                        params={'last_event_id': last_event_id},
                        timeout=60.0  # 60 секунд ожидания
                    )

                    if response.status_code == 200:
                        data = response.json()

                        if data.get('events'):
                            for event in data['events']:
                                await handler(event)
                                last_event_id = event['id']

                        # Timeout без событий — просто переподключаемся

                except httpx.ReadTimeout:
                    # Нормальное поведение — переподключаемся
                    continue

                except httpx.RequestError as e:
                    print(f'Connection error: {e}')
                    await asyncio.sleep(5)  # Backoff при ошибках

# Сервер (FastAPI)
from fastapi import FastAPI
import asyncio

app = FastAPI()
event_store = []  # В реальности — Redis/DB

@app.get('/long-poll')
async def long_poll(last_event_id: str = None, timeout: int = 30):
    """
    Long polling endpoint.
    Держит соединение до появления новых событий.
    """
    start_time = asyncio.get_event_loop().time()

    while True:
        # Проверяем новые события
        new_events = get_events_after(last_event_id)

        if new_events:
            return {'events': new_events}

        # Проверяем timeout
        elapsed = asyncio.get_event_loop().time() - start_time
        if elapsed >= timeout:
            return {'events': [], 'message': 'timeout'}

        # Ждём немного перед следующей проверкой
        await asyncio.sleep(0.5)
```

### Exponential Backoff

```python
class SmartPollingClient:
    def __init__(self, api_url: str):
        self.api_url = api_url
        self.min_interval = 1      # минимум 1 сек
        self.max_interval = 300    # максимум 5 мин
        self.current_interval = 1

    async def poll(self, handler):
        """Polling с адаптивным интервалом."""
        async with httpx.AsyncClient() as client:
            while True:
                try:
                    response = await client.get(f'{self.api_url}/updates')
                    data = response.json()

                    if data.get('updates'):
                        await handler(data['updates'])
                        # Есть данные — уменьшаем интервал
                        self.current_interval = max(
                            self.min_interval,
                            self.current_interval // 2
                        )
                    else:
                        # Нет данных — увеличиваем интервал
                        self.current_interval = min(
                            self.max_interval,
                            self.current_interval * 2
                        )

                except Exception as e:
                    # Ошибка — максимальный интервал
                    self.current_interval = self.max_interval

                await asyncio.sleep(self.current_interval)
```

### ETag / Conditional Requests

Оптимизация polling с помощью кэширования.

```python
class ETagPollingClient:
    def __init__(self, api_url: str):
        self.api_url = api_url
        self.etag = None

    async def poll(self, handler):
        async with httpx.AsyncClient() as client:
            while True:
                headers = {}
                if self.etag:
                    headers['If-None-Match'] = self.etag

                response = await client.get(
                    f'{self.api_url}/resource',
                    headers=headers
                )

                if response.status_code == 304:
                    # Данные не изменились
                    pass
                elif response.status_code == 200:
                    # Новые данные
                    self.etag = response.headers.get('ETag')
                    await handler(response.json())

                await asyncio.sleep(30)

# Сервер с ETag
from fastapi import FastAPI, Request, Response
import hashlib

@app.get('/resource')
async def get_resource(request: Request):
    data = get_current_data()

    # Генерируем ETag из данных
    etag = hashlib.md5(json.dumps(data).encode()).hexdigest()

    # Проверяем If-None-Match
    client_etag = request.headers.get('If-None-Match')
    if client_etag == etag:
        return Response(status_code=304)

    return Response(
        content=json.dumps(data),
        headers={'ETag': etag}
    )
```

---

## Webhooks

### Реализация Webhook-сервера

```python
from fastapi import FastAPI, Request, HTTPException, BackgroundTasks
from pydantic import BaseModel, HttpUrl
import httpx
import hmac
import hashlib
from typing import Optional
import asyncio

app = FastAPI()

# Модели
class WebhookSubscription(BaseModel):
    url: HttpUrl
    events: list[str]  # ['order.created', 'order.shipped']
    secret: Optional[str] = None

# Хранилище подписок (в реальности — DB)
subscriptions = {}

@app.post('/webhooks/subscribe')
async def subscribe(subscription: WebhookSubscription):
    """Регистрация webhook."""
    subscription_id = str(uuid.uuid4())

    # Верификация URL (опционально)
    if not await verify_webhook_url(str(subscription.url)):
        raise HTTPException(400, 'Invalid webhook URL')

    subscriptions[subscription_id] = subscription

    return {
        'subscription_id': subscription_id,
        'status': 'active'
    }

@app.delete('/webhooks/{subscription_id}')
async def unsubscribe(subscription_id: str):
    """Отмена подписки."""
    if subscription_id in subscriptions:
        del subscriptions[subscription_id]
        return {'status': 'unsubscribed'}
    raise HTTPException(404, 'Subscription not found')

# Отправка webhooks
class WebhookSender:
    def __init__(self, max_retries: int = 3):
        self.max_retries = max_retries

    async def send(self, event_type: str, payload: dict):
        """Отправляет webhook всем подписчикам."""
        tasks = []

        for sub_id, sub in subscriptions.items():
            if event_type in sub.events or '*' in sub.events:
                tasks.append(
                    self.deliver_webhook(sub, event_type, payload)
                )

        await asyncio.gather(*tasks, return_exceptions=True)

    async def deliver_webhook(
        self,
        subscription: WebhookSubscription,
        event_type: str,
        payload: dict
    ):
        """Доставляет webhook с retry."""
        body = json.dumps({
            'event': event_type,
            'timestamp': datetime.utcnow().isoformat(),
            'data': payload
        })

        # Подпись для верификации
        signature = self.create_signature(body, subscription.secret)

        headers = {
            'Content-Type': 'application/json',
            'X-Webhook-Signature': signature,
            'X-Webhook-Event': event_type
        }

        for attempt in range(self.max_retries):
            try:
                async with httpx.AsyncClient() as client:
                    response = await client.post(
                        str(subscription.url),
                        content=body,
                        headers=headers,
                        timeout=10.0
                    )

                    if 200 <= response.status_code < 300:
                        return True  # Успех

                    if response.status_code >= 500:
                        # Retry при 5xx
                        await asyncio.sleep(2 ** attempt)
                        continue

                    # 4xx — не retry
                    return False

            except Exception as e:
                if attempt < self.max_retries - 1:
                    await asyncio.sleep(2 ** attempt)
                else:
                    # Все попытки исчерпаны
                    await self.handle_failed_webhook(subscription, e)

        return False

    def create_signature(self, body: str, secret: Optional[str]) -> str:
        if not secret:
            return ''
        return hmac.new(
            secret.encode(),
            body.encode(),
            hashlib.sha256
        ).hexdigest()

# Использование
webhook_sender = WebhookSender()

async def create_order(order_data: dict):
    order = await save_order(order_data)

    # Отправляем webhook
    await webhook_sender.send('order.created', {
        'order_id': order.id,
        'customer_id': order.customer_id,
        'total': order.total
    })

    return order
```

### Реализация Webhook-клиента (получателя)

```python
from fastapi import FastAPI, Request, HTTPException
import hmac
import hashlib

app = FastAPI()

WEBHOOK_SECRET = 'your-secret-key'

def verify_signature(payload: bytes, signature: str) -> bool:
    """Проверяет подпись webhook."""
    expected = hmac.new(
        WEBHOOK_SECRET.encode(),
        payload,
        hashlib.sha256
    ).hexdigest()
    return hmac.compare_digest(expected, signature)

@app.post('/webhooks/receive')
async def receive_webhook(request: Request):
    """Endpoint для получения webhooks."""
    # Получаем сырое тело запроса
    body = await request.body()

    # Проверяем подпись
    signature = request.headers.get('X-Webhook-Signature', '')
    if not verify_signature(body, signature):
        raise HTTPException(401, 'Invalid signature')

    # Парсим событие
    event_type = request.headers.get('X-Webhook-Event')
    payload = json.loads(body)

    # Обрабатываем
    await handle_webhook_event(event_type, payload['data'])

    # Важно: отвечаем быстро!
    return {'status': 'received'}

async def handle_webhook_event(event_type: str, data: dict):
    """Обработка webhook события."""
    handlers = {
        'order.created': handle_order_created,
        'order.shipped': handle_order_shipped,
        'payment.completed': handle_payment_completed
    }

    handler = handlers.get(event_type)
    if handler:
        await handler(data)
```

### Идемпотентность Webhooks

```python
from redis import asyncio as aioredis

class IdempotentWebhookHandler:
    def __init__(self, redis_url: str):
        self.redis = aioredis.from_url(redis_url)

    async def handle(self, request: Request):
        body = await request.body()
        payload = json.loads(body)

        # Используем уникальный ID события
        event_id = payload.get('event_id') or request.headers.get('X-Webhook-ID')

        if not event_id:
            raise HTTPException(400, 'Missing event ID')

        # Проверяем, обрабатывали ли мы это событие
        key = f'webhook:processed:{event_id}'
        already_processed = await self.redis.get(key)

        if already_processed:
            return {'status': 'already_processed'}

        # Обрабатываем событие
        try:
            await self.process_event(payload)

            # Помечаем как обработанное (с TTL)
            await self.redis.setex(key, 86400 * 7, '1')  # 7 дней

            return {'status': 'processed'}

        except Exception as e:
            # Не помечаем как обработанное — повторная попытка возможна
            raise HTTPException(500, str(e))
```

### Очередь для обработки Webhooks

```python
# Быстро подтверждаем получение, обрабатываем асинхронно

from fastapi import FastAPI, BackgroundTasks
import aio_pika

app = FastAPI()

@app.post('/webhooks/receive')
async def receive_webhook(request: Request, background_tasks: BackgroundTasks):
    """Быстро отвечаем, обрабатываем в фоне."""
    body = await request.body()

    # Валидация подписи
    if not verify_signature(body, request.headers.get('X-Webhook-Signature')):
        raise HTTPException(401, 'Invalid signature')

    # Ставим в очередь на обработку
    background_tasks.add_task(enqueue_webhook, body)

    # Мгновенный ответ
    return {'status': 'received'}

async def enqueue_webhook(body: bytes):
    """Добавляет webhook в очередь."""
    connection = await aio_pika.connect_robust('amqp://localhost')
    channel = await connection.channel()
    queue = await channel.declare_queue('webhooks', durable=True)

    await channel.default_exchange.publish(
        aio_pika.Message(body=body),
        routing_key='webhooks'
    )

# Отдельный worker обрабатывает очередь
async def webhook_worker():
    connection = await aio_pika.connect_robust('amqp://localhost')
    channel = await connection.channel()
    queue = await channel.declare_queue('webhooks', durable=True)

    async with queue.iterator() as queue_iter:
        async for message in queue_iter:
            async with message.process():
                payload = json.loads(message.body)
                await process_webhook(payload)
```

---

## Сравнение подходов

| Критерий | Polling | Long Polling | Webhooks |
|----------|---------|--------------|----------|
| Latency | Высокая (interval) | Средняя | Низкая (real-time) |
| Server Load | Высокая | Средняя | Низкая |
| Client Load | Низкая | Средняя | Требует endpoint |
| Firewall | Без проблем | Без проблем | Может блокировать |
| Reliability | Простая | Средняя | Сложная |
| Implementation | Простая | Средняя | Сложная |

### Когда использовать Polling

- Firewall блокирует входящие соединения
- Простая инфраструктура без публичных endpoints
- Редкие обновления (раз в час/день)
- Не критична задержка в несколько минут

### Когда использовать Webhooks

- Требуется real-time уведомление
- Много клиентов с редкими событиями
- Есть публичный endpoint для приёма
- Критична нагрузка на сервер

---

## Гибридный подход

Комбинация webhooks + polling для надёжности.

```python
class HybridClient:
    """
    Использует webhooks для real-time,
    polling как fallback для пропущенных событий.
    """

    def __init__(self, api_url: str, webhook_url: str):
        self.api_url = api_url
        self.webhook_url = webhook_url
        self.last_event_id = None

    async def setup(self):
        # Регистрируем webhook
        await self.register_webhook()

        # Запускаем fallback polling
        asyncio.create_task(self.fallback_polling())

    async def register_webhook(self):
        async with httpx.AsyncClient() as client:
            await client.post(
                f'{self.api_url}/webhooks/subscribe',
                json={
                    'url': self.webhook_url,
                    'events': ['*']
                }
            )

    async def fallback_polling(self):
        """
        Проверяем пропущенные события каждые 5 минут.
        """
        while True:
            await asyncio.sleep(300)  # 5 минут

            try:
                async with httpx.AsyncClient() as client:
                    response = await client.get(
                        f'{self.api_url}/events',
                        params={'since_id': self.last_event_id}
                    )

                    events = response.json().get('events', [])
                    for event in events:
                        if not await self.is_processed(event['id']):
                            await self.process_event(event)
                        self.last_event_id = event['id']

            except Exception as e:
                print(f'Fallback polling error: {e}')

    async def handle_webhook(self, event: dict):
        """Обработчик webhook."""
        event_id = event['id']

        if await self.is_processed(event_id):
            return  # Уже обработано через polling

        await self.process_event(event)
        self.last_event_id = max(self.last_event_id or '', event_id)
```

---

## Best Practices

### Для Webhooks (отправитель)

```python
class RobustWebhookSender:
    def __init__(self):
        self.retry_delays = [1, 5, 30, 120, 600]  # секунды

    async def send_with_retry(self, url: str, payload: dict):
        """Надёжная отправка с exponential backoff."""
        for attempt, delay in enumerate(self.retry_delays):
            try:
                response = await self.send(url, payload)

                if response.status_code == 200:
                    return True

                if response.status_code == 410:  # Gone
                    await self.disable_subscription(url)
                    return False

                if response.status_code >= 500:
                    await asyncio.sleep(delay)
                    continue

                # 4xx — не retry
                return False

            except Exception:
                await asyncio.sleep(delay)

        # Все попытки исчерпаны
        await self.mark_as_failed(url, payload)
        return False

    def send(self, url: str, payload: dict):
        """Отправка с таймаутом и подписью."""
        body = json.dumps(payload)
        signature = self.sign(body)

        return httpx.post(
            url,
            content=body,
            headers={
                'Content-Type': 'application/json',
                'X-Webhook-Signature': f'sha256={signature}',
                'X-Webhook-Timestamp': str(int(time.time())),
                'User-Agent': 'MyApp-Webhooks/1.0'
            },
            timeout=10.0
        )
```

### Для Webhooks (получатель)

```python
# 1. Проверяйте подпись
def verify_webhook(request: Request):
    signature = request.headers.get('X-Webhook-Signature')
    timestamp = request.headers.get('X-Webhook-Timestamp')

    # Защита от replay attacks
    if abs(time.time() - int(timestamp)) > 300:  # 5 минут
        raise HTTPException(400, 'Request too old')

    expected = hmac.new(
        SECRET.encode(),
        f'{timestamp}.{request.body}'.encode(),
        hashlib.sha256
    ).hexdigest()

    if not hmac.compare_digest(f'sha256={expected}', signature):
        raise HTTPException(401, 'Invalid signature')

# 2. Отвечайте быстро
@app.post('/webhook')
async def webhook(request: Request, bg: BackgroundTasks):
    verify_webhook(request)
    bg.add_task(process_webhook, await request.json())
    return {'status': 'ok'}  # < 100ms

# 3. Делайте обработку идемпотентной
# 4. Логируйте все входящие webhooks
# 5. Имейте мониторинг и алерты
```

### Для Polling

```python
class OptimizedPollingClient:
    def __init__(self):
        self.session = httpx.AsyncClient(
            timeout=30.0,
            limits=httpx.Limits(max_keepalive_connections=5)
        )

    async def poll(self):
        # 1. Используйте HTTP/2 для эффективности
        # 2. Используйте conditional requests (ETag/If-Modified-Since)
        # 3. Адаптивный интервал polling
        # 4. Backoff при ошибках
        # 5. Мониторинг успешности запросов
        pass
```

---

## Типичные ошибки

### Polling

```python
# ❌ Плохо: фиксированный короткий интервал
while True:
    poll()
    time.sleep(1)  # 1 запрос в секунду — перегрузка!

# ✅ Хорошо: адаптивный интервал
while True:
    has_data = poll()
    if has_data:
        interval = max(1, interval // 2)
    else:
        interval = min(300, interval * 2)
    time.sleep(interval)
```

### Webhooks

```python
# ❌ Плохо: синхронная обработка
@app.post('/webhook')
def webhook(request: Request):
    data = request.json()
    process_order(data)  # 5 секунд
    update_inventory(data)  # 3 секунды
    send_notification(data)  # 2 секунды
    return {'ok': True}  # Отправитель ждёт 10 секунд!

# ✅ Хорошо: быстрый ответ + асинхронная обработка
@app.post('/webhook')
async def webhook(request: Request, bg: BackgroundTasks):
    data = await request.json()
    bg.add_task(process_webhook_async, data)
    return {'ok': True}  # Мгновенный ответ
```

```python
# ❌ Плохо: нет проверки подписи
@app.post('/webhook')
async def webhook(request: Request):
    data = await request.json()
    process(data)  # Любой может отправить!

# ✅ Хорошо: проверка подписи
@app.post('/webhook')
async def webhook(request: Request):
    verify_signature(request)  # 401 если невалидная
    data = await request.json()
    process(data)
```

---

## Примеры из реальных сервисов

### GitHub Webhooks

```python
# GitHub отправляет webhook при push
{
    "action": "push",
    "repository": {
        "id": 123,
        "full_name": "user/repo"
    },
    "commits": [...],
    "pusher": {...}
}

# Headers:
# X-GitHub-Event: push
# X-GitHub-Delivery: uuid
# X-Hub-Signature-256: sha256=...
```

### Stripe Webhooks

```python
import stripe

@app.post('/stripe/webhook')
async def stripe_webhook(request: Request):
    payload = await request.body()
    sig_header = request.headers.get('Stripe-Signature')

    try:
        event = stripe.Webhook.construct_event(
            payload, sig_header, STRIPE_WEBHOOK_SECRET
        )
    except ValueError:
        raise HTTPException(400, 'Invalid payload')
    except stripe.error.SignatureVerificationError:
        raise HTTPException(400, 'Invalid signature')

    # Обработка по типу события
    if event['type'] == 'payment_intent.succeeded':
        await handle_payment_success(event['data']['object'])
    elif event['type'] == 'payment_intent.failed':
        await handle_payment_failure(event['data']['object'])

    return {'status': 'success'}
```

---

## Заключение

| Сценарий | Рекомендация |
|----------|--------------|
| Real-time уведомления | Webhooks |
| Простая интеграция | Polling |
| За firewall | Long Polling |
| Критичная надёжность | Webhooks + Polling fallback |
| Много подписчиков | Webhooks |
| Редкие проверки | Polling |

**Ключевые принципы:**

1. **Webhooks**: Проверяйте подписи, отвечайте быстро, обрабатывайте идемпотентно
2. **Polling**: Используйте адаптивные интервалы, conditional requests, backoff
3. **Гибрид**: Webhooks для real-time + polling для надёжности
