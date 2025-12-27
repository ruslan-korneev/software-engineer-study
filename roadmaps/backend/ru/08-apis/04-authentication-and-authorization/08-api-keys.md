# API Keys & Management

## Что такое API Keys?

**API Key** — это уникальный идентификатор, используемый для аутентификации и авторизации доступа к API. API ключи — простой способ идентифицировать вызывающую сторону (приложение, сервис или пользователя) без сложных OAuth flows.

## Ключевые концепции

### Типы API Keys

| Тип | Описание | Пример использования |
|-----|----------|---------------------|
| **Public Key** | Можно раскрывать, только идентификация | Client ID в OAuth |
| **Secret Key** | Конфиденциальный, для аутентификации | API Secret |
| **Publishable Key** | Для frontend, ограниченные права | Stripe publishable key |
| **Restricted Key** | С ограниченными scopes | Read-only API key |

### Структура API Key

```
# Простой формат
sk_live_51H8KfJCM2nX1y3z

# Составной формат
<prefix>_<environment>_<random>

# С метаданными (Base64)
eyJhcGlfa2V5IjoiYWJjMTIzIiwidXNlcl9pZCI6MX0=
```

### Компоненты системы API Keys

```
┌─────────────────────────────────────────────────────────────────┐
│                     API Gateway / Server                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   ┌──────────────┐    ┌──────────────┐    ┌──────────────┐      │
│   │  Key Store   │    │ Rate Limiter │    │  Analytics   │      │
│   │              │    │              │    │              │      │
│   │ - Keys       │    │ - Per key    │    │ - Usage      │      │
│   │ - Scopes     │    │ - Per IP     │    │ - Errors     │      │
│   │ - Quotas     │    │ - Per user   │    │ - Endpoints  │      │
│   └──────────────┘    └──────────────┘    └──────────────┘      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

## Практические примеры кода

### Python — генерация и управление API Keys

```python
import secrets
import hashlib
import base64
from datetime import datetime, timedelta
from typing import Optional, List, Dict, Any
from dataclasses import dataclass, field
from enum import Enum
import json

class APIKeyStatus(Enum):
    ACTIVE = "active"
    REVOKED = "revoked"
    EXPIRED = "expired"

@dataclass
class APIKey:
    id: str
    key_hash: str  # Храним только хеш
    prefix: str    # Первые символы для идентификации
    name: str
    user_id: int
    scopes: List[str]
    created_at: datetime
    expires_at: Optional[datetime]
    last_used_at: Optional[datetime]
    status: APIKeyStatus
    rate_limit: int = 1000  # Запросов в час
    allowed_ips: List[str] = field(default_factory=list)
    metadata: Dict[str, Any] = field(default_factory=dict)

class APIKeyManager:
    def __init__(self, key_store: Dict[str, APIKey] = None):
        self.key_store = key_store or {}

    def generate_key(
        self,
        user_id: int,
        name: str,
        scopes: List[str],
        expires_in_days: Optional[int] = None,
        rate_limit: int = 1000,
        allowed_ips: Optional[List[str]] = None
    ) -> tuple[str, APIKey]:
        """
        Генерирует новый API ключ.
        Возвращает (raw_key, api_key_object).
        raw_key показывается пользователю только один раз!
        """
        # Генерируем случайный ключ
        key_bytes = secrets.token_bytes(32)
        raw_key = base64.urlsafe_b64encode(key_bytes).decode('utf-8').rstrip('=')

        # Добавляем префикс для идентификации типа
        prefix = f"sk_live_{raw_key[:8]}"
        full_key = f"sk_live_{raw_key}"

        # Хешируем ключ для хранения
        key_hash = hashlib.sha256(full_key.encode()).hexdigest()

        # Создаём объект ключа
        key_id = secrets.token_urlsafe(16)
        now = datetime.utcnow()

        api_key = APIKey(
            id=key_id,
            key_hash=key_hash,
            prefix=prefix,
            name=name,
            user_id=user_id,
            scopes=scopes,
            created_at=now,
            expires_at=now + timedelta(days=expires_in_days) if expires_in_days else None,
            last_used_at=None,
            status=APIKeyStatus.ACTIVE,
            rate_limit=rate_limit,
            allowed_ips=allowed_ips or []
        )

        self.key_store[key_hash] = api_key

        return full_key, api_key

    def validate_key(self, raw_key: str) -> Optional[APIKey]:
        """Валидирует API ключ и возвращает его данные"""
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        api_key = self.key_store.get(key_hash)

        if not api_key:
            return None

        # Проверяем статус
        if api_key.status != APIKeyStatus.ACTIVE:
            return None

        # Проверяем срок действия
        if api_key.expires_at and datetime.utcnow() > api_key.expires_at:
            api_key.status = APIKeyStatus.EXPIRED
            return None

        # Обновляем время последнего использования
        api_key.last_used_at = datetime.utcnow()

        return api_key

    def revoke_key(self, key_id: str) -> bool:
        """Отзывает API ключ"""
        for api_key in self.key_store.values():
            if api_key.id == key_id:
                api_key.status = APIKeyStatus.REVOKED
                return True
        return False

    def list_user_keys(self, user_id: int) -> List[APIKey]:
        """Список ключей пользователя"""
        return [
            key for key in self.key_store.values()
            if key.user_id == user_id
        ]

    def rotate_key(self, key_id: str) -> Optional[tuple[str, APIKey]]:
        """Ротация ключа — создаёт новый и отзывает старый"""
        old_key = None
        for api_key in self.key_store.values():
            if api_key.id == key_id:
                old_key = api_key
                break

        if not old_key:
            return None

        # Создаём новый ключ с теми же параметрами
        new_raw_key, new_api_key = self.generate_key(
            user_id=old_key.user_id,
            name=f"{old_key.name} (rotated)",
            scopes=old_key.scopes,
            rate_limit=old_key.rate_limit,
            allowed_ips=old_key.allowed_ips
        )

        # Отзываем старый ключ
        old_key.status = APIKeyStatus.REVOKED

        return new_raw_key, new_api_key

# Использование
manager = APIKeyManager()

# Создание ключа
raw_key, api_key = manager.generate_key(
    user_id=1,
    name="Production API Key",
    scopes=["read:users", "write:orders"],
    expires_in_days=365,
    rate_limit=5000
)

print(f"API Key (save it!): {raw_key}")
print(f"Key ID: {api_key.id}")
print(f"Prefix: {api_key.prefix}")

# Валидация
validated = manager.validate_key(raw_key)
if validated:
    print(f"Valid key for user {validated.user_id}")
    print(f"Scopes: {validated.scopes}")
```

### Python (FastAPI) — полная реализация

```python
from fastapi import FastAPI, Depends, HTTPException, Header, Request
from pydantic import BaseModel
from typing import List, Optional
from datetime import datetime, timedelta
import secrets
import hashlib
import redis
from functools import wraps

app = FastAPI()

# Redis для хранения ключей и rate limiting
redis_client = redis.from_url("redis://localhost:6379")

# Модели
class APIKeyCreate(BaseModel):
    name: str
    scopes: List[str]
    expires_in_days: Optional[int] = 365
    rate_limit: int = 1000
    allowed_ips: Optional[List[str]] = None

class APIKeyResponse(BaseModel):
    id: str
    prefix: str
    name: str
    scopes: List[str]
    created_at: datetime
    expires_at: Optional[datetime]
    last_used_at: Optional[datetime]
    status: str

class APIKeyCreateResponse(BaseModel):
    key: str  # Полный ключ — показывается только один раз!
    details: APIKeyResponse

# API Key Manager
class APIKeyService:
    KEY_PREFIX = "apikey:"
    RATE_LIMIT_PREFIX = "ratelimit:"

    def __init__(self, redis_client):
        self.redis = redis_client

    def create_key(self, user_id: int, data: APIKeyCreate) -> tuple[str, dict]:
        """Создаёт новый API ключ"""
        # Генерируем ключ
        random_part = secrets.token_urlsafe(32)
        raw_key = f"sk_live_{random_part}"
        prefix = f"sk_live_{random_part[:8]}..."

        # Хешируем для хранения
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        key_id = secrets.token_urlsafe(16)

        now = datetime.utcnow()
        expires_at = now + timedelta(days=data.expires_in_days) if data.expires_in_days else None

        key_data = {
            "id": key_id,
            "key_hash": key_hash,
            "prefix": prefix,
            "name": data.name,
            "user_id": user_id,
            "scopes": data.scopes,
            "created_at": now.isoformat(),
            "expires_at": expires_at.isoformat() if expires_at else None,
            "last_used_at": None,
            "status": "active",
            "rate_limit": data.rate_limit,
            "allowed_ips": data.allowed_ips or []
        }

        # Сохраняем в Redis
        self.redis.hset(f"{self.KEY_PREFIX}{key_hash}", mapping={
            k: str(v) if isinstance(v, (list, dict)) else v
            for k, v in key_data.items()
            if v is not None
        })

        # Индекс по user_id
        self.redis.sadd(f"user_keys:{user_id}", key_hash)

        return raw_key, key_data

    def validate_key(self, raw_key: str, client_ip: str = None) -> Optional[dict]:
        """Валидирует API ключ"""
        key_hash = hashlib.sha256(raw_key.encode()).hexdigest()
        key_data = self.redis.hgetall(f"{self.KEY_PREFIX}{key_hash}")

        if not key_data:
            return None

        # Декодируем bytes в строки
        key_data = {k.decode(): v.decode() for k, v in key_data.items()}

        # Проверяем статус
        if key_data.get("status") != "active":
            return None

        # Проверяем срок действия
        expires_at = key_data.get("expires_at")
        if expires_at and expires_at != "None":
            if datetime.fromisoformat(expires_at) < datetime.utcnow():
                self.redis.hset(f"{self.KEY_PREFIX}{key_hash}", "status", "expired")
                return None

        # Проверяем IP (если указаны ограничения)
        allowed_ips = key_data.get("allowed_ips")
        if allowed_ips and allowed_ips != "[]" and client_ip:
            import ast
            ips = ast.literal_eval(allowed_ips)
            if ips and client_ip not in ips:
                return None

        # Обновляем last_used_at
        self.redis.hset(
            f"{self.KEY_PREFIX}{key_hash}",
            "last_used_at",
            datetime.utcnow().isoformat()
        )

        return key_data

    def check_rate_limit(self, key_hash: str, rate_limit: int) -> tuple[bool, int]:
        """Проверяет rate limit. Возвращает (allowed, remaining)"""
        key = f"{self.RATE_LIMIT_PREFIX}{key_hash}"
        current = self.redis.get(key)

        if current is None:
            # Первый запрос в этом окне
            self.redis.setex(key, 3600, 1)  # 1 час
            return True, rate_limit - 1

        current = int(current)
        if current >= rate_limit:
            return False, 0

        self.redis.incr(key)
        return True, rate_limit - current - 1

    def revoke_key(self, key_id: str, user_id: int) -> bool:
        """Отзывает ключ"""
        # Находим ключ по ID
        for key_hash in self.redis.smembers(f"user_keys:{user_id}"):
            key_hash = key_hash.decode()
            key_data = self.redis.hgetall(f"{self.KEY_PREFIX}{key_hash}")
            if key_data and key_data.get(b"id", b"").decode() == key_id:
                self.redis.hset(f"{self.KEY_PREFIX}{key_hash}", "status", "revoked")
                return True
        return False

    def list_user_keys(self, user_id: int) -> List[dict]:
        """Список ключей пользователя"""
        keys = []
        for key_hash in self.redis.smembers(f"user_keys:{user_id}"):
            key_hash = key_hash.decode()
            key_data = self.redis.hgetall(f"{self.KEY_PREFIX}{key_hash}")
            if key_data:
                key_data = {k.decode(): v.decode() for k, v in key_data.items()}
                # Не возвращаем hash
                key_data.pop("key_hash", None)
                keys.append(key_data)
        return keys

api_key_service = APIKeyService(redis_client)

# Dependency для проверки API ключа
async def verify_api_key(
    request: Request,
    x_api_key: str = Header(..., alias="X-API-Key")
) -> dict:
    """Dependency для проверки API ключа"""
    client_ip = request.client.host

    key_data = api_key_service.validate_key(x_api_key, client_ip)
    if not key_data:
        raise HTTPException(status_code=401, detail="Invalid API key")

    # Проверяем rate limit
    rate_limit = int(key_data.get("rate_limit", 1000))
    allowed, remaining = api_key_service.check_rate_limit(
        key_data["key_hash"],
        rate_limit
    )

    if not allowed:
        raise HTTPException(
            status_code=429,
            detail="Rate limit exceeded",
            headers={"X-RateLimit-Remaining": "0"}
        )

    # Добавляем заголовки rate limit
    request.state.rate_limit_remaining = remaining

    return key_data

def require_scope(required_scope: str):
    """Decorator для проверки scope"""
    async def scope_checker(key_data: dict = Depends(verify_api_key)):
        import ast
        scopes = ast.literal_eval(key_data.get("scopes", "[]"))

        if required_scope not in scopes and "*" not in scopes:
            raise HTTPException(
                status_code=403,
                detail=f"Scope '{required_scope}' required"
            )
        return key_data
    return scope_checker

# Эндпоинты управления ключами
@app.post("/api-keys", response_model=APIKeyCreateResponse)
async def create_api_key(data: APIKeyCreate, user_id: int = 1):  # user_id из auth
    """Создаёт новый API ключ"""
    raw_key, key_data = api_key_service.create_key(user_id, data)

    return APIKeyCreateResponse(
        key=raw_key,
        details=APIKeyResponse(
            id=key_data["id"],
            prefix=key_data["prefix"],
            name=key_data["name"],
            scopes=key_data["scopes"],
            created_at=datetime.fromisoformat(key_data["created_at"]),
            expires_at=datetime.fromisoformat(key_data["expires_at"]) if key_data["expires_at"] else None,
            last_used_at=None,
            status=key_data["status"]
        )
    )

@app.get("/api-keys")
async def list_api_keys(user_id: int = 1):
    """Список API ключей пользователя"""
    keys = api_key_service.list_user_keys(user_id)
    return {"keys": keys}

@app.delete("/api-keys/{key_id}")
async def revoke_api_key(key_id: str, user_id: int = 1):
    """Отзывает API ключ"""
    if api_key_service.revoke_key(key_id, user_id):
        return {"message": "Key revoked"}
    raise HTTPException(status_code=404, detail="Key not found")

# Защищённые эндпоинты
@app.get("/protected")
async def protected_endpoint(key_data: dict = Depends(verify_api_key)):
    """Эндпоинт, защищённый API ключом"""
    return {"message": "Access granted", "user_id": key_data.get("user_id")}

@app.get("/users")
async def list_users(key_data: dict = Depends(require_scope("read:users"))):
    """Требует scope read:users"""
    return {"users": ["user1", "user2"]}

@app.post("/orders")
async def create_order(key_data: dict = Depends(require_scope("write:orders"))):
    """Требует scope write:orders"""
    return {"order_id": 12345}
```

### Node.js (Express)

```javascript
const express = require('express');
const crypto = require('crypto');
const Redis = require('ioredis');

const app = express();
app.use(express.json());

const redis = new Redis();

// Генерация API ключа
function generateAPIKey() {
    const randomBytes = crypto.randomBytes(32);
    const key = randomBytes.toString('base64url');
    return `sk_live_${key}`;
}

// Хеширование ключа
function hashKey(key) {
    return crypto.createHash('sha256').update(key).digest('hex');
}

// Middleware для проверки API ключа
async function verifyAPIKey(req, res, next) {
    const apiKey = req.headers['x-api-key'];

    if (!apiKey) {
        return res.status(401).json({ error: 'API key required' });
    }

    const keyHash = hashKey(apiKey);
    const keyData = await redis.hgetall(`apikey:${keyHash}`);

    if (!keyData || Object.keys(keyData).length === 0) {
        return res.status(401).json({ error: 'Invalid API key' });
    }

    if (keyData.status !== 'active') {
        return res.status(401).json({ error: 'API key is not active' });
    }

    // Проверка срока действия
    if (keyData.expires_at && new Date(keyData.expires_at) < new Date()) {
        await redis.hset(`apikey:${keyHash}`, 'status', 'expired');
        return res.status(401).json({ error: 'API key expired' });
    }

    // Rate limiting
    const rateLimitKey = `ratelimit:${keyHash}`;
    const current = await redis.incr(rateLimitKey);

    if (current === 1) {
        await redis.expire(rateLimitKey, 3600); // 1 час
    }

    const rateLimit = parseInt(keyData.rate_limit) || 1000;
    if (current > rateLimit) {
        return res.status(429).json({
            error: 'Rate limit exceeded',
            limit: rateLimit,
            reset: await redis.ttl(rateLimitKey)
        });
    }

    // Обновляем last_used_at
    await redis.hset(`apikey:${keyHash}`, 'last_used_at', new Date().toISOString());

    req.apiKey = keyData;
    res.set('X-RateLimit-Remaining', rateLimit - current);

    next();
}

// Проверка scope
function requireScope(scope) {
    return (req, res, next) => {
        const scopes = JSON.parse(req.apiKey.scopes || '[]');

        if (!scopes.includes(scope) && !scopes.includes('*')) {
            return res.status(403).json({ error: `Scope '${scope}' required` });
        }

        next();
    };
}

// Создание API ключа
app.post('/api-keys', async (req, res) => {
    const { name, scopes, expires_in_days = 365, rate_limit = 1000 } = req.body;
    const userId = 1; // Из аутентификации

    const rawKey = generateAPIKey();
    const keyHash = hashKey(rawKey);
    const keyId = crypto.randomBytes(16).toString('hex');

    const now = new Date();
    const expiresAt = new Date(now.getTime() + expires_in_days * 24 * 60 * 60 * 1000);

    const keyData = {
        id: keyId,
        key_hash: keyHash,
        prefix: `${rawKey.substring(0, 15)}...`,
        name,
        user_id: userId.toString(),
        scopes: JSON.stringify(scopes),
        created_at: now.toISOString(),
        expires_at: expiresAt.toISOString(),
        status: 'active',
        rate_limit: rate_limit.toString()
    };

    await redis.hset(`apikey:${keyHash}`, keyData);
    await redis.sadd(`user_keys:${userId}`, keyHash);

    res.json({
        key: rawKey, // Показать только один раз!
        id: keyId,
        prefix: keyData.prefix,
        name,
        expires_at: expiresAt
    });
});

// Список ключей
app.get('/api-keys', async (req, res) => {
    const userId = 1;
    const keyHashes = await redis.smembers(`user_keys:${userId}`);

    const keys = [];
    for (const hash of keyHashes) {
        const keyData = await redis.hgetall(`apikey:${hash}`);
        if (keyData) {
            delete keyData.key_hash;
            keys.push(keyData);
        }
    }

    res.json({ keys });
});

// Отзыв ключа
app.delete('/api-keys/:keyId', async (req, res) => {
    const userId = 1;
    const keyHashes = await redis.smembers(`user_keys:${userId}`);

    for (const hash of keyHashes) {
        const keyData = await redis.hgetall(`apikey:${hash}`);
        if (keyData && keyData.id === req.params.keyId) {
            await redis.hset(`apikey:${hash}`, 'status', 'revoked');
            return res.json({ message: 'Key revoked' });
        }
    }

    res.status(404).json({ error: 'Key not found' });
});

// Защищённые эндпоинты
app.get('/protected', verifyAPIKey, (req, res) => {
    res.json({ message: 'Access granted', user_id: req.apiKey.user_id });
});

app.get('/users', verifyAPIKey, requireScope('read:users'), (req, res) => {
    res.json({ users: ['user1', 'user2'] });
});

app.listen(3000, () => console.log('Server running on port 3000'));
```

## Способы передачи API Key

### 1. Header (рекомендуется)

```http
GET /api/resource HTTP/1.1
Host: api.example.com
X-API-Key: sk_live_abc123...
```

### 2. Query Parameter (менее безопасно)

```http
GET /api/resource?api_key=sk_live_abc123... HTTP/1.1
```

**Проблемы:**
- Попадает в логи сервера
- Видно в истории браузера
- Может утечь через Referer header

### 3. Basic Auth

```http
GET /api/resource HTTP/1.1
Authorization: Basic base64(api_key:)
```

## Плюсы API Keys

| Преимущество | Описание |
|--------------|----------|
| **Простота** | Легко реализовать и использовать |
| **Идентификация** | Легко отследить, кто делает запросы |
| **Rate Limiting** | Легко ограничить по ключу |
| **Analytics** | Статистика использования |
| **Revocation** | Можно мгновенно отозвать |

## Минусы API Keys

| Недостаток | Описание |
|------------|----------|
| **Статичные** | Не истекают автоматически |
| **Кража** | Украденный ключ даёт полный доступ |
| **Нет контекста** | Не знаем, кто именно использует |
| **Не для пользователей** | Не подходит для user-facing приложений |

## Best Practices

1. **Никогда не храни ключи в открытом виде** — только хеши
2. **Используй HTTPS** — ключ передаётся в каждом запросе
3. **Устанавливай TTL** — ключи должны истекать
4. **Реализуй rate limiting** — по ключу
5. **Логируй использование** — для аудита
6. **Поддерживай ротацию** — без простоя
7. **Используй scopes** — ограничивай права
8. **Передавай в header** — не в URL

## Типичные ошибки

### Ошибка 1: Ключ в URL

```bash
# ПЛОХО: ключ в URL
curl "https://api.example.com/users?api_key=sk_live_secret"

# ХОРОШО: ключ в header
curl -H "X-API-Key: sk_live_secret" https://api.example.com/users
```

### Ошибка 2: Хранение ключа без хеширования

```python
# ПЛОХО
store_key(api_key)

# ХОРОШО
store_key_hash(hashlib.sha256(api_key.encode()).hexdigest())
```

### Ошибка 3: Нет ротации

```python
# ХОРОШО: поддержка ротации
def rotate_key(old_key_id):
    # Создаём новый ключ
    new_key = create_key()
    # Старый ключ остаётся активным какое-то время
    schedule_revocation(old_key_id, delay=timedelta(hours=24))
    return new_key
```

## Резюме

API Keys — это простой и эффективный способ аутентификации для machine-to-machine взаимодействия. Они хорошо подходят для публичных API, интеграций и сервисов. Главное — правильно хранить (хешировать), передавать (в header, не в URL) и управлять (scopes, rate limiting, ротация).
