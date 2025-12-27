# Hashes (Хэши) в Redis

Хэши в Redis — это структуры данных, представляющие собой коллекции пар "поле-значение". Они идеально подходят для хранения объектов, где каждое поле представляет атрибут объекта.

## Основные характеристики

- **Структура**: набор пар field-value внутри одного ключа
- **Эффективность**: работа с отдельными полями без загрузки всего объекта
- **Экономия памяти**: компактное хранение для небольших хэшей
- **Атомарные операции**: над отдельными полями

## Базовые команды

### HSET — установка полей

```redis
# Установить одно поле
HSET user:1000 name "John"
# Результат: 1 (количество добавленных полей)

# Установить несколько полей
HSET user:1000 email "john@example.com" age "30" city "Moscow"
# Результат: 3

# Обновить существующее поле
HSET user:1000 age "31"
# Результат: 0 (поле обновлено, не добавлено)
```

### HGET и HMGET — получение значений

```redis
# Получить одно поле
HGET user:1000 name
# Результат: "John"

HGET user:1000 nonexistent
# Результат: nil

# Получить несколько полей
HMGET user:1000 name email age
# Результат: ["John", "john@example.com", "31"]

# Несуществующие поля возвращают nil
HMGET user:1000 name nonexistent email
# Результат: ["John", nil, "john@example.com"]
```

### HGETALL — получить все поля и значения

```redis
HSET product:100 name "Laptop" price "999" stock "50"

HGETALL product:100
# Результат:
# ["name", "Laptop", "price", "999", "stock", "50"]
```

**Внимание**: HGETALL имеет сложность O(N). Для больших хэшей используйте HSCAN.

### HDEL — удаление полей

```redis
HSET user:1000 name "John" email "john@example.com" temp_field "temp"

HDEL user:1000 temp_field
# Результат: 1

HDEL user:1000 field1 field2 field3
# Результат: 0 (ни одно поле не существовало)
```

### HEXISTS — проверка существования поля

```redis
HSET user:1000 name "John"

HEXISTS user:1000 name
# Результат: 1

HEXISTS user:1000 nonexistent
# Результат: 0
```

### HLEN — количество полей

```redis
HSET user:1000 name "John" email "john@example.com" age "30"

HLEN user:1000
# Результат: 3
```

### HKEYS и HVALS — получить только ключи или значения

```redis
HSET config timeout "30" retries "3" host "localhost"

HKEYS config
# Результат: ["timeout", "retries", "host"]

HVALS config
# Результат: ["30", "3", "localhost"]
```

## Числовые операции

### HINCRBY — инкремент целого числа

```redis
HSET product:100 stock "50" views "0"

HINCRBY product:100 stock -5
# Результат: 45

HINCRBY product:100 views 1
# Результат: 1

# Если поле не существует, начинает с 0
HINCRBY product:100 sales 10
# Результат: 10
```

### HINCRBYFLOAT — инкремент дробного числа

```redis
HSET product:100 price "99.99"

HINCRBYFLOAT product:100 price 0.01
# Результат: "100"

HINCRBYFLOAT product:100 price -10.50
# Результат: "89.5"
```

## Дополнительные команды

### HSETNX — установить только если поле не существует

```redis
HSET user:1000 name "John"

HSETNX user:1000 name "Jane"
# Результат: 0 (поле уже существует, не изменено)

HSETNX user:1000 nickname "johnny"
# Результат: 1 (поле создано)

HGET user:1000 name
# Результат: "John" (не изменилось)
```

### HSTRLEN — длина значения поля

```redis
HSET user:1000 description "Software developer from Moscow"

HSTRLEN user:1000 description
# Результат: 31
```

### HRANDFIELD (Redis 6.2+) — случайные поля

```redis
HSET colors red "#ff0000" green "#00ff00" blue "#0000ff" yellow "#ffff00"

# Получить одно случайное поле
HRANDFIELD colors
# Результат: "green"

# Получить несколько случайных полей
HRANDFIELD colors 2
# Результат: ["red", "blue"]

# С WITHVALUES — включить значения
HRANDFIELD colors 2 WITHVALUES
# Результат: ["green", "#00ff00", "yellow", "#ffff00"]
```

### HSCAN — итерация по полям

Для больших хэшей используйте HSCAN:

```redis
HSCAN user:1000 0
# Результат: ["cursor", ["field1", "value1", "field2", "value2", ...]]

# С паттерном
HSCAN user:1000 0 MATCH "email*"

# С лимитом
HSCAN user:1000 0 COUNT 100
```

## Практические примеры использования

### Хранение профилей пользователей

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

class UserProfile:
    def __init__(self, user_id):
        self.key = f"user:{user_id}"
        self.user_id = user_id

    def create(self, data):
        """Создать профиль пользователя."""
        r.hset(self.key, mapping={
            "id": self.user_id,
            "name": data.get("name", ""),
            "email": data.get("email", ""),
            "created_at": str(time.time()),
            "status": "active"
        })

    def get(self):
        """Получить весь профиль."""
        return r.hgetall(self.key)

    def get_field(self, field):
        """Получить одно поле."""
        return r.hget(self.key, field)

    def update(self, updates):
        """Обновить несколько полей."""
        updates["updated_at"] = str(time.time())
        r.hset(self.key, mapping=updates)

    def update_field(self, field, value):
        """Обновить одно поле."""
        r.hset(self.key, field, value)

    def delete_field(self, field):
        """Удалить поле."""
        r.hdel(self.key, field)

    def exists(self):
        """Проверить существование профиля."""
        return r.exists(self.key)

    def delete(self):
        """Удалить профиль."""
        r.delete(self.key)

# Использование
profile = UserProfile(1000)
profile.create({
    "name": "John Doe",
    "email": "john@example.com"
})

profile.update({"city": "Moscow", "age": "30"})

user_data = profile.get()
# {'id': '1000', 'name': 'John Doe', 'email': 'john@example.com', ...}
```

### Хранение сессий

```python
import secrets
import time
import json

class SessionManager:
    def __init__(self, ttl=3600):  # 1 час
        self.ttl = ttl

    def create_session(self, user_id, metadata=None):
        """Создать новую сессию."""
        session_id = secrets.token_urlsafe(32)
        key = f"session:{session_id}"

        session_data = {
            "user_id": str(user_id),
            "created_at": str(time.time()),
            "last_activity": str(time.time())
        }

        if metadata:
            for k, v in metadata.items():
                session_data[k] = json.dumps(v) if isinstance(v, (dict, list)) else str(v)

        r.hset(key, mapping=session_data)
        r.expire(key, self.ttl)

        return session_id

    def get_session(self, session_id):
        """Получить данные сессии."""
        key = f"session:{session_id}"
        session = r.hgetall(key)

        if session:
            # Обновить last_activity и продлить TTL
            r.hset(key, "last_activity", str(time.time()))
            r.expire(key, self.ttl)

        return session

    def update_session(self, session_id, data):
        """Обновить данные сессии."""
        key = f"session:{session_id}"
        if r.exists(key):
            r.hset(key, mapping=data)
            r.expire(key, self.ttl)
            return True
        return False

    def destroy_session(self, session_id):
        """Уничтожить сессию."""
        return r.delete(f"session:{session_id}")

    def get_user_id(self, session_id):
        """Получить ID пользователя из сессии."""
        return r.hget(f"session:{session_id}", "user_id")

# Использование
sessions = SessionManager(ttl=7200)

session_id = sessions.create_session(
    user_id=1000,
    metadata={"ip": "192.168.1.1", "user_agent": "Mozilla/5.0"}
)

session = sessions.get_session(session_id)
```

### Хранение конфигураций

```python
class ConfigManager:
    def __init__(self, namespace="config"):
        self.key = f"{namespace}:settings"

    def set(self, name, value):
        """Установить параметр."""
        r.hset(self.key, name, json.dumps(value))

    def get(self, name, default=None):
        """Получить параметр."""
        value = r.hget(self.key, name)
        if value is None:
            return default
        return json.loads(value)

    def get_all(self):
        """Получить все параметры."""
        raw = r.hgetall(self.key)
        return {k: json.loads(v) for k, v in raw.items()}

    def delete(self, name):
        """Удалить параметр."""
        r.hdel(self.key, name)

    def set_if_not_exists(self, name, value):
        """Установить только если не существует."""
        return r.hsetnx(self.key, name, json.dumps(value))

# Использование
config = ConfigManager("myapp")

config.set("max_connections", 100)
config.set("features", {"dark_mode": True, "beta": False})
config.set("timeout", 30)

max_conn = config.get("max_connections")
# 100

features = config.get("features")
# {"dark_mode": True, "beta": False}
```

### Корзина покупок

```python
class ShoppingCart:
    def __init__(self, user_id):
        self.key = f"cart:{user_id}"

    def add_item(self, product_id, quantity=1):
        """Добавить товар в корзину."""
        current = r.hget(self.key, product_id)
        if current:
            quantity += int(current)
        r.hset(self.key, product_id, quantity)

        # Установить TTL корзины (7 дней)
        r.expire(self.key, 86400 * 7)

    def remove_item(self, product_id):
        """Удалить товар из корзины."""
        r.hdel(self.key, product_id)

    def update_quantity(self, product_id, quantity):
        """Обновить количество товара."""
        if quantity <= 0:
            self.remove_item(product_id)
        else:
            r.hset(self.key, product_id, quantity)

    def get_items(self):
        """Получить все товары в корзине."""
        items = r.hgetall(self.key)
        return {k: int(v) for k, v in items.items()}

    def get_item_count(self):
        """Количество уникальных товаров."""
        return r.hlen(self.key)

    def get_total_items(self):
        """Общее количество товаров."""
        items = r.hvals(self.key)
        return sum(int(qty) for qty in items)

    def clear(self):
        """Очистить корзину."""
        r.delete(self.key)

    def merge_with(self, other_cart_key):
        """Объединить с другой корзиной."""
        other_items = r.hgetall(other_cart_key)
        for product_id, quantity in other_items.items():
            self.add_item(product_id, int(quantity))
        r.delete(other_cart_key)

# Использование
cart = ShoppingCart(user_id=1000)

cart.add_item("prod:123", 2)
cart.add_item("prod:456", 1)
cart.add_item("prod:123", 1)  # Теперь 3 штуки

items = cart.get_items()
# {"prod:123": 3, "prod:456": 1}
```

### Счётчики и метрики

```python
class Metrics:
    def __init__(self, name):
        self.key = f"metrics:{name}"

    def increment(self, metric, value=1):
        """Увеличить метрику."""
        return r.hincrby(self.key, metric, value)

    def increment_float(self, metric, value):
        """Увеличить метрику с дробным значением."""
        return r.hincrbyfloat(self.key, metric, value)

    def get(self, metric):
        """Получить значение метрики."""
        value = r.hget(self.key, metric)
        return int(value) if value else 0

    def get_all(self):
        """Получить все метрики."""
        metrics = r.hgetall(self.key)
        return {k: int(v) for k, v in metrics.items()}

    def reset(self, metric=None):
        """Сбросить метрику или все метрики."""
        if metric:
            r.hdel(self.key, metric)
        else:
            r.delete(self.key)

# Использование для веб-аналитики
class PageMetrics:
    def record_pageview(self, page_id, user_id=None):
        """Записать просмотр страницы."""
        key = f"page:{page_id}:metrics"

        pipe = r.pipeline()
        pipe.hincrby(key, "views", 1)
        if user_id:
            pipe.hincrby(key, "logged_in_views", 1)
        pipe.execute()

    def record_event(self, page_id, event_type):
        """Записать событие на странице."""
        key = f"page:{page_id}:metrics"
        r.hincrby(key, f"event:{event_type}", 1)

    def get_stats(self, page_id):
        """Получить статистику страницы."""
        return r.hgetall(f"page:{page_id}:metrics")
```

## Сложность операций

| Команда | Сложность |
|---------|-----------|
| HSET, HGET, HDEL | O(1) для одного поля, O(N) для N полей |
| HMGET, HMSET | O(N), где N — количество полей |
| HGETALL, HKEYS, HVALS | O(N), где N — размер хэша |
| HLEN | O(1) |
| HINCRBY, HINCRBYFLOAT | O(1) |
| HEXISTS | O(1) |
| HSCAN | O(1) за итерацию, O(N) для полного обхода |

## Лучшие практики

1. **Используйте хэши вместо отдельных ключей** для связанных данных — это экономит память
2. **Не храните слишком много полей** — хэши оптимизированы для < 512 полей
3. **Используйте HMGET** для получения нескольких полей вместо нескольких HGET
4. **HSCAN для больших хэшей** вместо HGETALL
5. **HINCRBY атомарен** — используйте для счётчиков
6. **Устанавливайте TTL** на весь хэш для временных данных (сессии, корзины)
