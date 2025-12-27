# Idempotency (Идемпотентность)

## Введение

**Идемпотентность** — это свойство операции, при котором многократное выполнение даёт тот же результат, что и однократное. В контексте API это означает, что повторный запрос не создаёт побочных эффектов.

```
f(x) = f(f(x)) = f(f(f(x))) = ...
```

**Пример из жизни:**
- Нажатие кнопки лифта — идемпотентно (лифт приедет один раз)
- Нажатие кнопки "Купить" — НЕ идемпотентно (каждое нажатие создаёт заказ)

## Зачем нужна идемпотентность?

### Проблема без идемпотентности:

```
Клиент → [POST /orders] → Сервер → "Заказ создан"
         ↑
    Таймаут!
    Клиент не получил ответ
         ↓
Клиент → [POST /orders] → Сервер → "Заказ создан" (ДУБЛИКАТ!)
```

**Реальные сценарии:**
- Сетевые сбои
- Таймауты
- Повторные отправки форм (F5)
- Retry logic в клиентских библиотеках

## Идемпотентность HTTP-методов

| Метод | Идемпотентный | Безопасный | Описание |
|-------|---------------|------------|----------|
| GET | Да | Да | Только чтение |
| HEAD | Да | Да | Только заголовки |
| OPTIONS | Да | Да | Доступные методы |
| PUT | Да | Нет | Замена ресурса целиком |
| DELETE | Да | Нет | Удаление ресурса |
| POST | **Нет** | Нет | Создание ресурса |
| PATCH | **Нет*** | Нет | Частичное обновление |

*PATCH может быть идемпотентным при правильной реализации

### Почему PUT идемпотентен, а POST — нет?

```http
# PUT — заменяет ресурс целиком
PUT /users/123
{"name": "John", "email": "john@example.com"}

# Повторный запрос даёт тот же результат
PUT /users/123
{"name": "John", "email": "john@example.com"}
# Пользователь 123 остаётся тем же

# POST — создаёт новый ресурс
POST /users
{"name": "John", "email": "john@example.com"}
# Создан пользователь с ID 1

POST /users
{"name": "John", "email": "john@example.com"}
# Создан пользователь с ID 2 (ДУБЛИКАТ!)
```

## Реализация идемпотентности

### 1. Idempotency Key (Ключ идемпотентности)

Клиент генерирует уникальный ключ для каждой операции:

```http
POST /api/orders HTTP/1.1
Content-Type: application/json
Idempotency-Key: 8e03978e-40d5-43e8-bc93-6894a57f9324

{
    "product_id": 123,
    "quantity": 2
}
```

Сервер:
1. Проверяет, был ли такой ключ использован
2. Если да — возвращает сохранённый ответ
3. Если нет — выполняет операцию и сохраняет результат

#### Реализация (Python/FastAPI):

```python
import uuid
import json
import hashlib
from fastapi import FastAPI, Header, HTTPException, Request
from pydantic import BaseModel
from typing import Optional
import redis

app = FastAPI()
redis_client = redis.Redis(host='localhost', port=6379, db=0)

IDEMPOTENCY_TTL = 86400  # 24 часа

class Order(BaseModel):
    product_id: int
    quantity: int

class IdempotencyStore:
    @staticmethod
    def get_key(idempotency_key: str, path: str, method: str) -> str:
        """Создаёт уникальный ключ с учётом endpoint."""
        return f"idempotency:{hashlib.sha256(f'{idempotency_key}:{path}:{method}'.encode()).hexdigest()}"

    @staticmethod
    def get(key: str) -> Optional[dict]:
        """Получает сохранённый результат."""
        data = redis_client.get(key)
        if data:
            return json.loads(data)
        return None

    @staticmethod
    def set(key: str, response: dict, ttl: int = IDEMPOTENCY_TTL):
        """Сохраняет результат."""
        redis_client.setex(key, ttl, json.dumps(response))

    @staticmethod
    def set_in_progress(key: str, ttl: int = 60) -> bool:
        """Помечает операцию как выполняющуюся (защита от race condition)."""
        return redis_client.set(key, json.dumps({"status": "in_progress"}), nx=True, ex=ttl)

@app.post("/api/orders")
async def create_order(
    order: Order,
    request: Request,
    idempotency_key: Optional[str] = Header(None, alias="Idempotency-Key")
):
    # Если ключ не предоставлен — работаем как обычно (не идемпотентно)
    if not idempotency_key:
        return await _create_order(order)

    # Валидируем формат ключа (UUID)
    try:
        uuid.UUID(idempotency_key)
    except ValueError:
        raise HTTPException(
            status_code=400,
            detail="Idempotency-Key must be a valid UUID"
        )

    cache_key = IdempotencyStore.get_key(
        idempotency_key,
        str(request.url.path),
        request.method
    )

    # Проверяем, есть ли сохранённый результат
    cached_response = IdempotencyStore.get(cache_key)

    if cached_response:
        if cached_response.get("status") == "in_progress":
            raise HTTPException(
                status_code=409,
                detail="A request with this idempotency key is already in progress"
            )
        # Возвращаем сохранённый ответ
        return cached_response

    # Помечаем как in_progress для защиты от параллельных запросов
    if not IdempotencyStore.set_in_progress(cache_key):
        raise HTTPException(
            status_code=409,
            detail="A request with this idempotency key is already in progress"
        )

    try:
        # Выполняем операцию
        result = await _create_order(order)

        # Сохраняем результат
        IdempotencyStore.set(cache_key, result)

        return result
    except Exception as e:
        # При ошибке удаляем in_progress маркер
        redis_client.delete(cache_key)
        raise

async def _create_order(order: Order) -> dict:
    """Создаёт заказ в БД."""
    order_id = uuid.uuid4().hex[:8]
    return {
        "id": order_id,
        "product_id": order.product_id,
        "quantity": order.quantity,
        "status": "created"
    }
```

### 2. Сравнение тела запроса

Дополнительная проверка: тело запроса должно совпадать.

```python
class IdempotencyStore:
    @staticmethod
    def get_key_with_body(
        idempotency_key: str,
        path: str,
        method: str,
        body: bytes
    ) -> str:
        """Создаёт ключ с учётом тела запроса."""
        body_hash = hashlib.sha256(body).hexdigest()
        return f"idempotency:{hashlib.sha256(f'{idempotency_key}:{path}:{method}:{body_hash}'.encode()).hexdigest()}"

@app.post("/api/orders")
async def create_order(
    request: Request,
    idempotency_key: Optional[str] = Header(None, alias="Idempotency-Key")
):
    body = await request.body()

    cache_key = IdempotencyStore.get_key_with_body(
        idempotency_key,
        str(request.url.path),
        request.method,
        body
    )

    # ... остальная логика
```

### 3. PUT vs POST для создания

Альтернатива: клиент генерирует ID ресурса.

```http
# Вместо POST (неидемпотентный)
POST /api/orders
{"product_id": 123}

# Используем PUT с клиентским ID (идемпотентный)
PUT /api/orders/550e8400-e29b-41d4-a716-446655440000
{"product_id": 123}
```

```python
@app.put("/api/orders/{order_id}")
async def create_or_update_order(order_id: str, order: Order):
    existing = await db.get_order(order_id)

    if existing:
        # Заказ уже существует — возвращаем его
        return existing

    # Создаём новый заказ
    new_order = {
        "id": order_id,
        "product_id": order.product_id,
        "quantity": order.quantity
    }
    await db.save_order(new_order)
    return new_order
```

## Реализация на Node.js

```javascript
const express = require('express');
const Redis = require('ioredis');
const crypto = require('crypto');
const { v4: uuidv4, validate: uuidValidate } = require('uuid');

const app = express();
app.use(express.json());

const redis = new Redis();

const IDEMPOTENCY_TTL = 86400; // 24 часа

// Middleware для идемпотентности
const idempotencyMiddleware = (options = {}) => {
    const { methods = ['POST', 'PATCH'] } = options;

    return async (req, res, next) => {
        // Применяем только к указанным методам
        if (!methods.includes(req.method)) {
            return next();
        }

        const idempotencyKey = req.headers['idempotency-key'];

        // Если ключ не предоставлен — пропускаем
        if (!idempotencyKey) {
            return next();
        }

        // Валидация UUID
        if (!uuidValidate(idempotencyKey)) {
            return res.status(400).json({
                error: 'Idempotency-Key must be a valid UUID'
            });
        }

        const cacheKey = `idempotency:${crypto
            .createHash('sha256')
            .update(`${idempotencyKey}:${req.path}:${req.method}`)
            .digest('hex')}`;

        // Проверяем кэш
        const cached = await redis.get(cacheKey);

        if (cached) {
            const cachedResponse = JSON.parse(cached);

            if (cachedResponse.status === 'in_progress') {
                return res.status(409).json({
                    error: 'Request with this idempotency key is in progress'
                });
            }

            // Возвращаем сохранённый ответ
            return res
                .status(cachedResponse.statusCode)
                .set(cachedResponse.headers)
                .json(cachedResponse.body);
        }

        // Помечаем как in_progress
        const locked = await redis.set(
            cacheKey,
            JSON.stringify({ status: 'in_progress' }),
            'EX', 60,
            'NX'
        );

        if (!locked) {
            return res.status(409).json({
                error: 'Request with this idempotency key is in progress'
            });
        }

        // Сохраняем оригинальные методы res
        const originalJson = res.json.bind(res);
        const originalSend = res.send.bind(res);

        // Перехватываем ответ
        res.json = async (body) => {
            const responseData = {
                statusCode: res.statusCode,
                headers: res.getHeaders(),
                body
            };

            await redis.setex(cacheKey, IDEMPOTENCY_TTL, JSON.stringify(responseData));
            return originalJson(body);
        };

        // Обработка ошибок
        res.on('error', async () => {
            await redis.del(cacheKey);
        });

        next();
    };
};

app.use('/api/', idempotencyMiddleware({ methods: ['POST', 'PATCH'] }));

app.post('/api/orders', async (req, res) => {
    const { product_id, quantity } = req.body;

    // Создаём заказ
    const order = {
        id: uuidv4().slice(0, 8),
        product_id,
        quantity,
        status: 'created'
    };

    res.status(201).json(order);
});

app.listen(3000);
```

## Сложные сценарии

### 1. Финансовые операции

```python
@app.post("/api/payments")
async def create_payment(
    payment: PaymentRequest,
    idempotency_key: str = Header(..., alias="Idempotency-Key")
):
    cache_key = get_idempotency_key(idempotency_key)

    # Начинаем транзакцию
    async with db.transaction():
        # Проверяем в БД (не только в Redis)
        existing = await db.get_payment_by_idempotency_key(idempotency_key)

        if existing:
            return existing

        # Создаём запись с idempotency_key
        payment_record = await db.create_payment(
            amount=payment.amount,
            currency=payment.currency,
            idempotency_key=idempotency_key
        )

        # Выполняем платёж
        result = await payment_gateway.charge(payment_record)

        # Обновляем статус
        await db.update_payment_status(payment_record.id, result.status)

    return payment_record
```

### 2. Составные операции

```python
@app.post("/api/transfer")
async def transfer_money(
    transfer: TransferRequest,
    idempotency_key: str = Header(...)
):
    """Перевод между счетами — атомарная идемпотентная операция."""

    async with db.transaction():
        # Проверяем, был ли такой перевод
        existing = await db.get_transfer_by_idempotency_key(idempotency_key)
        if existing:
            return existing

        # Создаём запись о переводе
        transfer_record = await db.create_transfer(
            from_account=transfer.from_account,
            to_account=transfer.to_account,
            amount=transfer.amount,
            idempotency_key=idempotency_key,
            status="pending"
        )

        try:
            # Списание
            await db.debit_account(transfer.from_account, transfer.amount)

            # Зачисление
            await db.credit_account(transfer.to_account, transfer.amount)

            # Обновляем статус
            transfer_record.status = "completed"
            await db.update_transfer(transfer_record)

        except Exception as e:
            transfer_record.status = "failed"
            transfer_record.error = str(e)
            await db.update_transfer(transfer_record)
            raise

    return transfer_record
```

### 3. Обработка ошибок

```python
class IdempotencyError(Exception):
    pass

@app.exception_handler(IdempotencyError)
async def idempotency_error_handler(request: Request, exc: IdempotencyError):
    return JSONResponse(
        status_code=422,
        content={
            "error": "idempotency_error",
            "message": str(exc)
        }
    )

@app.post("/api/orders")
async def create_order(
    order: Order,
    idempotency_key: str = Header(...)
):
    cached = await get_cached_response(idempotency_key)

    if cached:
        # Проверяем, что тело запроса совпадает
        if cached.get("request_body_hash") != hash_body(order):
            raise IdempotencyError(
                "Idempotency key was already used with different request body"
            )
        return cached["response"]

    # ... создание заказа
```

## Примеры из реальных API

### Stripe

```http
POST /v1/charges HTTP/1.1
Idempotency-Key: KG5LxwFBepaKHyUD

amount=2000&currency=usd&source=tok_visa
```

Stripe хранит результаты 24 часа и возвращает:
- Тот же результат для повторных запросов
- Ошибку 400, если тело запроса отличается

### PayPal

```http
POST /v2/checkout/orders HTTP/1.1
PayPal-Request-Id: 7b92603e-77ed-4896-8e78-5dea2050476a

{...}
```

### Amazon AWS

```http
POST / HTTP/1.1
X-Amzn-Client-Token: unique-client-token-123

{...}
```

## Best Practices

### Что делать:

1. **Всегда поддерживайте Idempotency-Key для POST:**
   ```python
   idempotency_key: str = Header(..., alias="Idempotency-Key")
   ```

2. **Храните результаты достаточно долго:**
   - Минимум 24 часа для большинства случаев
   - Дольше для финансовых операций

3. **Защищайтесь от race conditions:**
   ```python
   redis.set(key, value, nx=True)  # Только если не существует
   ```

4. **Валидируйте формат ключа:**
   - UUID v4 — хороший выбор

5. **Включайте endpoint в ключ кэша:**
   ```python
   f"{idempotency_key}:{path}:{method}"
   ```

6. **Логируйте повторные запросы:**
   ```python
   logger.info(f"Returning cached response for key {idempotency_key}")
   ```

### Типичные ошибки:

1. **Игнорирование идемпотентности:**
   - Дубликаты платежей, заказов

2. **Слишком короткий TTL:**
   - Клиент может повторить запрос позже

3. **Ключ только по телу запроса:**
   - Разные операции с одинаковым телом

4. **Нет защиты от race conditions:**
   - Параллельные запросы могут выполниться оба

5. **Хранение только в памяти:**
   - Потеря при перезапуске сервера

## Заключение

Идемпотентность — критически важное свойство для надёжных API:

- **GET, PUT, DELETE** — идемпотентны по дизайну
- **POST** — требует явной реализации через Idempotency-Key
- **Храните результаты** — для возврата при повторных запросах
- **Защищайтесь от race conditions** — используйте атомарные операции

Правильная реализация идемпотентности делает API устойчивым к сетевым сбоям и ошибкам клиентов.
