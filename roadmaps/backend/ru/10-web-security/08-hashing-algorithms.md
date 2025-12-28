# Алгоритмы хэширования

[prev: 07-api-security-best-practices](./07-api-security-best-practices.md) | [next: 01-unit-testing](../11-testing/01-unit-testing.md)

---

## Что такое хэширование?

**Хэширование** — это процесс преобразования данных произвольной длины в строку фиксированной длины (хэш). Хэш-функции являются односторонними: получить исходные данные из хэша невозможно.

### Свойства криптографических хэш-функций

1. **Детерминированность** — одинаковый вход всегда даёт одинаковый выход
2. **Быстрое вычисление** — хэш вычисляется эффективно
3. **Устойчивость к коллизиям** — сложно найти два разных входа с одинаковым хэшем
4. **Лавинный эффект** — малое изменение входа сильно меняет выход
5. **Необратимость** — невозможно восстановить входные данные из хэша

## Обзор алгоритмов

### MD5 (Message Digest 5)

**Статус: УСТАРЕЛ, НЕ ИСПОЛЬЗОВАТЬ ДЛЯ БЕЗОПАСНОСТИ**

- Длина хэша: 128 бит (32 hex символа)
- Скорость: очень быстрый
- Проблемы: найдены коллизии, уязвим к атакам

```python
import hashlib

# НЕ ИСПОЛЬЗУЙТЕ для паролей или безопасности!
text = "Hello, World!"
md5_hash = hashlib.md5(text.encode()).hexdigest()
print(md5_hash)  # '65a8e27d8879283831b664bd8b7f0ad4'

# Допустимые применения:
# - Контрольные суммы файлов (не для безопасности)
# - Кэширование (fingerprint)
# - Дедупликация данных
```

```javascript
const crypto = require('crypto');

// Для контрольных сумм (НЕ для безопасности)
const hash = crypto.createHash('md5').update('Hello, World!').digest('hex');
console.log(hash);  // '65a8e27d8879283831b664bd8b7f0ad4'
```

### SHA-1 (Secure Hash Algorithm 1)

**Статус: УСТАРЕЛ ДЛЯ КРИПТОГРАФИИ**

- Длина хэша: 160 бит (40 hex символов)
- Проблемы: найдены практические коллизии (SHAttered атака)

```python
import hashlib

# НЕ ИСПОЛЬЗУЙТЕ для новых проектов
sha1_hash = hashlib.sha1("Hello, World!".encode()).hexdigest()
print(sha1_hash)  # '0a0a9f2a6772942557ab5355d76af442f8f65e01'
```

### SHA-256 / SHA-512 (SHA-2 семейство)

**Статус: РЕКОМЕНДУЕТСЯ для общего использования**

- SHA-256: 256 бит (64 hex символа)
- SHA-512: 512 бит (128 hex символов)
- Безопасен для подписей, проверки целостности

```python
import hashlib

text = "Hello, World!"

# SHA-256
sha256_hash = hashlib.sha256(text.encode()).hexdigest()
print(f"SHA-256: {sha256_hash}")
# 'dffd6021bb2bd5b0af676290809ec3a53191dd81c7f70a4b28688a362182986f'

# SHA-512
sha512_hash = hashlib.sha512(text.encode()).hexdigest()
print(f"SHA-512: {sha512_hash}")

# Хэширование файла
def hash_file(filepath):
    sha256 = hashlib.sha256()
    with open(filepath, 'rb') as f:
        while chunk := f.read(8192):
            sha256.update(chunk)
    return sha256.hexdigest()
```

```javascript
const crypto = require('crypto');

// SHA-256
const sha256 = crypto.createHash('sha256')
    .update('Hello, World!')
    .digest('hex');

// Для потокового хэширования больших файлов
const fs = require('fs');
const hash = crypto.createHash('sha256');
const stream = fs.createReadStream('large-file.zip');
stream.on('data', data => hash.update(data));
stream.on('end', () => console.log(hash.digest('hex')));
```

### SHA-3 / Keccak

**Статус: РЕКОМЕНДУЕТСЯ (новейший стандарт)**

- Другая архитектура, чем SHA-2
- Устойчив к атакам на SHA-2

```python
import hashlib

# SHA3-256
sha3_hash = hashlib.sha3_256("Hello, World!".encode()).hexdigest()
print(f"SHA3-256: {sha3_hash}")

# SHAKE (переменная длина)
shake = hashlib.shake_256("Hello, World!".encode())
print(f"SHAKE256 (64 bytes): {shake.hexdigest(64)}")
```

## Алгоритмы для паролей

**ВАЖНО:** SHA-256 и другие быстрые хэш-функции НЕ подходят для паролей!

Проблема быстрых алгоритмов:
```
MD5:     ~10 миллиардов хэшей/сек (GPU)
SHA-256: ~5 миллиардов хэшей/сек (GPU)
bcrypt:  ~50 тысяч хэшей/сек (специализированный)
Argon2:  ~10 тысяч хэшей/сек (специализированный)
```

### bcrypt

**Статус: РЕКОМЕНДУЕТСЯ для паролей**

- Медленный по дизайну
- Настраиваемая сложность (work factor)
- Встроенная соль

```python
import bcrypt

# Хэширование пароля
password = "MySecurePassword123!"
salt = bcrypt.gensalt(rounds=12)  # cost factor 2^12 = 4096 итераций
password_hash = bcrypt.hashpw(password.encode(), salt)
print(password_hash)
# b'$2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4EQmM.rZl0c/pASi'

# Проверка пароля
def verify_password(password: str, password_hash: bytes) -> bool:
    return bcrypt.checkpw(password.encode(), password_hash)

# Использование
is_valid = verify_password("MySecurePassword123!", password_hash)
print(f"Password valid: {is_valid}")  # True

# Увеличение cost factor со временем
def needs_rehash(password_hash: bytes, desired_rounds: int = 12) -> bool:
    # Извлекаем текущий cost из хэша
    current_rounds = int(password_hash.decode().split('$')[2])
    return current_rounds < desired_rounds
```

```javascript
const bcrypt = require('bcrypt');

// Хэширование
const password = 'MySecurePassword123!';
const saltRounds = 12;

// Асинхронный вариант (рекомендуется)
const hash = await bcrypt.hash(password, saltRounds);
console.log(hash);
// $2b$12$LQv3c1yqBWVHxkd0LHAkCOYz6TtxMQJqhN8/X4EQmM.rZl0c/pASi

// Проверка
const isValid = await bcrypt.compare(password, hash);
console.log(`Password valid: ${isValid}`);
```

### Argon2

**Статус: САМЫЙ РЕКОМЕНДУЕМЫЙ для паролей (победитель Password Hashing Competition)**

Преимущества:
- Устойчивость к GPU-атакам (memory-hard)
- Настраиваемое использование памяти
- Защита от side-channel атак

Варианты:
- **Argon2d** — максимальная устойчивость к GPU (для серверов)
- **Argon2i** — устойчивость к side-channel атакам
- **Argon2id** — гибрид (рекомендуется)

```python
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

# Создание хэшера с параметрами
ph = PasswordHasher(
    time_cost=3,        # Количество итераций
    memory_cost=65536,  # 64MB памяти
    parallelism=4,      # 4 потока
    hash_len=32,        # Длина хэша
    salt_len=16         # Длина соли
)

# Хэширование
password = "MySecurePassword123!"
hash = ph.hash(password)
print(hash)
# $argon2id$v=19$m=65536,t=3,p=4$c29tZXNhbHQ$RdescudvJCsgt3ub+b+dWRWJTmaaJObG

# Проверка
def verify_password(password: str, hash: str) -> bool:
    try:
        ph.verify(hash, password)
        return True
    except VerifyMismatchError:
        return False

# Проверка необходимости перехэширования
def needs_rehash(hash: str) -> bool:
    return ph.check_needs_rehash(hash)

# Полный flow аутентификации
def authenticate_user(username: str, password: str):
    user = db.get_user(username)
    if not user:
        # Выполняем dummy-проверку для предотвращения timing-атак
        ph.verify("$argon2id$v=19$m=65536,t=3,p=4$dummy", "dummy")
        return None

    try:
        ph.verify(user.password_hash, password)

        # Перехэширование если параметры устарели
        if ph.check_needs_rehash(user.password_hash):
            user.password_hash = ph.hash(password)
            db.save(user)

        return user
    except VerifyMismatchError:
        return None
```

```javascript
const argon2 = require('argon2');

// Хэширование с Argon2id (рекомендуется)
const hash = await argon2.hash('MySecurePassword123!', {
    type: argon2.argon2id,
    memoryCost: 65536,  // 64MB
    timeCost: 3,
    parallelism: 4
});

console.log(hash);

// Проверка
const isValid = await argon2.verify(hash, 'MySecurePassword123!');
console.log(`Password valid: ${isValid}`);

// Проверка необходимости перехэширования
const needsRehash = argon2.needsRehash(hash, {
    memoryCost: 65536,
    timeCost: 3
});
```

### scrypt

**Статус: РЕКОМЕНДУЕТСЯ (memory-hard)**

```python
import hashlib
import os

# Хэширование
password = "MySecurePassword123!"
salt = os.urandom(16)

# Параметры: N=2^14, r=8, p=1
derived_key = hashlib.scrypt(
    password.encode(),
    salt=salt,
    n=16384,      # CPU/memory cost (должен быть степенью 2)
    r=8,          # Block size
    p=1,          # Parallelization
    dklen=64      # Длина ключа
)

# Сохраняем соль вместе с хэшем
stored = salt + derived_key

# Проверка
def verify_scrypt(password: str, stored: bytes) -> bool:
    salt = stored[:16]
    stored_key = stored[16:]

    derived_key = hashlib.scrypt(
        password.encode(),
        salt=salt,
        n=16384, r=8, p=1,
        dklen=64
    )

    return derived_key == stored_key
```

## HMAC (Hash-based Message Authentication Code)

HMAC используется для проверки целостности и аутентичности данных.

```python
import hmac
import hashlib

secret_key = b'super-secret-key'
message = b'Important message'

# Создание HMAC
signature = hmac.new(secret_key, message, hashlib.sha256).hexdigest()
print(f"HMAC: {signature}")

# Безопасное сравнение (защита от timing-атак)
def verify_hmac(message: bytes, signature: str, secret_key: bytes) -> bool:
    expected = hmac.new(secret_key, message, hashlib.sha256).hexdigest()
    return hmac.compare_digest(expected, signature)

# Использование для API подписей
def sign_request(payload: dict, secret: str) -> str:
    import json
    message = json.dumps(payload, sort_keys=True).encode()
    return hmac.new(secret.encode(), message, hashlib.sha256).hexdigest()
```

```javascript
const crypto = require('crypto');

// Создание HMAC
const secret = 'super-secret-key';
const message = 'Important message';

const hmac = crypto.createHmac('sha256', secret)
    .update(message)
    .digest('hex');

console.log(`HMAC: ${hmac}`);

// Безопасное сравнение
function verifyHmac(message, signature, secret) {
    const expected = crypto.createHmac('sha256', secret)
        .update(message)
        .digest('hex');

    return crypto.timingSafeEqual(
        Buffer.from(signature),
        Buffer.from(expected)
    );
}
```

## Соль (Salt)

**Соль** — случайные данные, добавляемые к паролю перед хэшированием.

### Зачем нужна соль?

```
Без соли:
password123 -> 5f4dcc3b5aa765d61d8327deb882cf99

Атакующий может использовать rainbow tables:
5f4dcc3b5aa765d61d8327deb882cf99 -> password123

С солью:
password123 + "x7Kj9m2P" -> a1b2c3d4e5f6...
password123 + "Qw3rTy1!" -> f6e5d4c3b2a1...

Каждый пользователь получает уникальный хэш!
```

### Правильная реализация

```python
import os
import hashlib

def hash_password_with_salt(password: str) -> tuple[bytes, bytes]:
    """Возвращает (соль, хэш)"""
    salt = os.urandom(32)  # 256-битная соль
    hash = hashlib.pbkdf2_hmac(
        'sha256',
        password.encode(),
        salt,
        iterations=100000,  # Минимум 100,000 итераций
        dklen=32
    )
    return salt, hash

def verify_password(password: str, salt: bytes, stored_hash: bytes) -> bool:
    hash = hashlib.pbkdf2_hmac(
        'sha256',
        password.encode(),
        salt,
        iterations=100000,
        dklen=32
    )
    return hash == stored_hash
```

## Pepper (Секретная соль)

**Pepper** — серверный секрет, добавляемый ко всем паролям.

```python
import os
import hmac
import hashlib

# Хранится в переменных окружения, НЕ в базе данных
PEPPER = os.environ.get('PASSWORD_PEPPER', '').encode()

def hash_with_pepper(password: str, salt: bytes) -> bytes:
    # Добавляем pepper через HMAC
    peppered = hmac.new(PEPPER, password.encode(), hashlib.sha256).digest()

    # Хэшируем с солью
    return hashlib.pbkdf2_hmac(
        'sha256',
        peppered,
        salt,
        iterations=100000,
        dklen=32
    )
```

## Сравнение алгоритмов

| Алгоритм | Для паролей | Для подписей | Скорость | Memory-hard |
|----------|-------------|--------------|----------|-------------|
| MD5 | Нет | Нет | Очень быстрый | Нет |
| SHA-1 | Нет | Нет | Быстрый | Нет |
| SHA-256 | Нет | Да | Быстрый | Нет |
| SHA-3 | Нет | Да | Быстрый | Нет |
| bcrypt | Да | Нет | Медленный | Нет |
| scrypt | Да | Нет | Медленный | Да |
| Argon2 | Да | Нет | Медленный | Да |

## Рекомендации по выбору

### Для паролей
1. **Argon2id** — лучший выбор
2. **bcrypt** — проверенная альтернатива
3. **scrypt** — если Argon2 недоступен

### Для подписей и целостности
1. **SHA-256** или **SHA-512**
2. **SHA-3** — для новых проектов
3. **HMAC-SHA256** — для аутентификации сообщений

### Для контрольных сумм (не безопасность)
1. **SHA-256** — надёжно
2. **MD5** — только для legacy-систем

## Практические рекомендации

```python
# config.py — настройки хэширования
class HashingConfig:
    # Argon2 параметры (OWASP рекомендации)
    ARGON2_TIME_COST = 3
    ARGON2_MEMORY_COST = 65536  # 64MB
    ARGON2_PARALLELISM = 4

    # bcrypt параметры
    BCRYPT_ROUNDS = 12  # Увеличивайте со временем

    # Pepper (из переменных окружения)
    PASSWORD_PEPPER = os.environ.get('PASSWORD_PEPPER')

# Пример сервиса паролей
from argon2 import PasswordHasher
from argon2.exceptions import VerifyMismatchError

class PasswordService:
    def __init__(self, config: HashingConfig):
        self.ph = PasswordHasher(
            time_cost=config.ARGON2_TIME_COST,
            memory_cost=config.ARGON2_MEMORY_COST,
            parallelism=config.ARGON2_PARALLELISM
        )
        self.pepper = config.PASSWORD_PEPPER

    def hash(self, password: str) -> str:
        # Добавляем pepper
        if self.pepper:
            password = hmac.new(
                self.pepper.encode(),
                password.encode(),
                hashlib.sha256
            ).hexdigest()

        return self.ph.hash(password)

    def verify(self, password: str, hash: str) -> bool:
        if self.pepper:
            password = hmac.new(
                self.pepper.encode(),
                password.encode(),
                hashlib.sha256
            ).hexdigest()

        try:
            self.ph.verify(hash, password)
            return True
        except VerifyMismatchError:
            return False

    def needs_rehash(self, hash: str) -> bool:
        return self.ph.check_needs_rehash(hash)
```

## Заключение

Правильный выбор алгоритма хэширования критически важен для безопасности:

1. **Никогда не используйте MD5/SHA-1 для безопасности**
2. **Для паролей — только Argon2, bcrypt или scrypt**
3. **Всегда используйте уникальную соль**
4. **Рассмотрите использование pepper**
5. **Регулярно увеличивайте параметры сложности**
6. **Реализуйте перехэширование при входе пользователя**

---

[prev: 07-api-security-best-practices](./07-api-security-best-practices.md) | [next: 01-unit-testing](../11-testing/01-unit-testing.md)
