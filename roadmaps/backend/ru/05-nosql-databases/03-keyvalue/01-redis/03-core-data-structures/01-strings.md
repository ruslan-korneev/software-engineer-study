# Strings (Строки) в Redis

Строки — это самый базовый и наиболее часто используемый тип данных в Redis. Строка в Redis может содержать любую последовательность байтов: текст, сериализованные объекты, бинарные данные или числа. Максимальный размер строки — 512 МБ.

## Базовые команды

### SET и GET

Основные команды для записи и чтения значений:

```redis
# Установить значение
SET key "value"

# Получить значение
GET key
# Результат: "value"

# SET с опциями
SET key "value" EX 3600      # Истекает через 3600 секунд
SET key "value" PX 10000     # Истекает через 10000 миллисекунд
SET key "value" NX           # Только если ключ НЕ существует
SET key "value" XX           # Только если ключ УЖЕ существует

# Установить несколько значений за один запрос
MSET key1 "value1" key2 "value2" key3 "value3"

# Получить несколько значений
MGET key1 key2 key3
# Результат: ["value1", "value2", "value3"]
```

### APPEND и STRLEN

Команды для работы с содержимым строки:

```redis
SET greeting "Hello"

# Добавить текст к существующему значению
APPEND greeting " World"
# Результат: 11 (новая длина строки)

GET greeting
# Результат: "Hello World"

# Получить длину строки
STRLEN greeting
# Результат: 11
```

### GETRANGE и SETRANGE

Работа с подстроками:

```redis
SET message "Hello World"

# Получить подстроку (индексы от 0)
GETRANGE message 0 4
# Результат: "Hello"

GETRANGE message -5 -1
# Результат: "World"

# Заменить часть строки
SETRANGE message 6 "Redis"
GET message
# Результат: "Hello Redis"
```

## Числовые операции

Redis умеет работать со строками как с числами, если они содержат числовое представление:

### INCR и DECR

```redis
SET counter "10"

# Увеличить на 1
INCR counter
# Результат: 11

# Уменьшить на 1
DECR counter
# Результат: 10

# Если ключ не существует, начинает с 0
INCR new_counter
# Результат: 1
```

### INCRBY и DECRBY

```redis
SET counter "100"

# Увеличить на указанное значение
INCRBY counter 50
# Результат: 150

# Уменьшить на указанное значение
DECRBY counter 25
# Результат: 125
```

### INCRBYFLOAT

Для работы с дробными числами:

```redis
SET temperature "36.6"

INCRBYFLOAT temperature 0.5
# Результат: "37.1"

INCRBYFLOAT temperature -1.5
# Результат: "35.6"
```

**Важно**: Все атомарные операции инкремента/декремента являются потокобезопасными. Redis гарантирует, что даже при одновременных запросах значение будет корректно обновлено.

## Битовые операции

Redis позволяет работать со строками на уровне отдельных битов:

### SETBIT и GETBIT

```redis
# Установить бит на позиции 7
SETBIT flags 7 1
# Результат: 0 (предыдущее значение бита)

# Получить значение бита
GETBIT flags 7
# Результат: 1

GETBIT flags 0
# Результат: 0 (бит не был установлен)
```

### BITCOUNT

Подсчёт установленных битов:

```redis
SET mykey "foobar"

# Подсчитать все установленные биты
BITCOUNT mykey
# Результат: 26

# Подсчитать биты в диапазоне байтов
BITCOUNT mykey 0 0
# Результат: 4 (биты в первом байте "f")
```

### BITOP

Побитовые операции между ключами:

```redis
SET key1 "foof"
SET key2 "foof"

# AND, OR, XOR, NOT
BITOP AND destkey key1 key2
BITOP OR destkey key1 key2
BITOP XOR destkey key1 key2
BITOP NOT destkey key1
```

## Практические примеры использования

### Счётчики посещений

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

# Увеличить счётчик посещений страницы
def increment_page_views(page_id):
    key = f"page:views:{page_id}"
    return r.incr(key)

# Получить количество просмотров
def get_page_views(page_id):
    key = f"page:views:{page_id}"
    views = r.get(key)
    return int(views) if views else 0

# Ежедневный счётчик с автоматическим истечением
def increment_daily_counter(metric_name):
    from datetime import date
    today = date.today().isoformat()
    key = f"daily:{metric_name}:{today}"

    pipe = r.pipeline()
    pipe.incr(key)
    pipe.expire(key, 86400 * 7)  # Хранить 7 дней
    pipe.execute()
```

### Кэширование данных

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, db=0)

def get_user_from_cache(user_id):
    key = f"user:{user_id}"
    cached = r.get(key)

    if cached:
        return json.loads(cached)
    return None

def cache_user(user_id, user_data, ttl=3600):
    key = f"user:{user_id}"
    r.set(key, json.dumps(user_data), ex=ttl)

# Паттерн Cache-Aside
def get_user(user_id):
    # Попытка получить из кэша
    user = get_user_from_cache(user_id)

    if user is None:
        # Загрузить из базы данных
        user = fetch_user_from_database(user_id)
        if user:
            cache_user(user_id, user)

    return user
```

### Rate Limiting (Ограничение частоты запросов)

```python
import redis
import time

r = redis.Redis(host='localhost', port=6379, db=0)

def is_rate_limited(user_id, max_requests=100, window_seconds=60):
    """
    Простой rate limiter на основе счётчика.
    Возвращает True, если лимит превышен.
    """
    key = f"ratelimit:{user_id}:{int(time.time()) // window_seconds}"

    current = r.incr(key)

    if current == 1:
        r.expire(key, window_seconds)

    return current > max_requests
```

### Распределённые блокировки

```python
import redis
import uuid
import time

r = redis.Redis(host='localhost', port=6379, db=0)

def acquire_lock(lock_name, acquire_timeout=10, lock_timeout=10):
    """Получить распределённую блокировку."""
    identifier = str(uuid.uuid4())
    lock_key = f"lock:{lock_name}"
    end = time.time() + acquire_timeout

    while time.time() < end:
        # NX — установить только если не существует
        if r.set(lock_key, identifier, nx=True, ex=lock_timeout):
            return identifier
        time.sleep(0.001)

    return None

def release_lock(lock_name, identifier):
    """Освободить блокировку, если она принадлежит нам."""
    lock_key = f"lock:{lock_name}"

    # Lua скрипт для атомарной проверки и удаления
    script = """
    if redis.call("get", KEYS[1]) == ARGV[1] then
        return redis.call("del", KEYS[1])
    else
        return 0
    end
    """
    return r.eval(script, 1, lock_key, identifier)
```

## Лучшие практики

1. **Используйте осмысленные имена ключей**: `user:1234:email` вместо `u1234e`
2. **Устанавливайте TTL** для данных, которые должны истекать
3. **Используйте MGET/MSET** для пакетных операций — это уменьшает сетевые задержки
4. **Предпочитайте атомарные операции** (INCR вместо GET + SET) для избежания race conditions
5. **Помните о максимальном размере** — 512 МБ на строку

## Сложность операций

| Команда | Сложность |
|---------|-----------|
| SET, GET | O(1) |
| MSET, MGET | O(N), где N — количество ключей |
| APPEND | O(1) амортизированная |
| STRLEN | O(1) |
| INCR, DECR | O(1) |
| SETRANGE | O(1) для маленьких строк |
| GETRANGE | O(N), где N — длина возвращаемой подстроки |
