# Lua-скрипты в Redis

## Что такое Lua-скрипты и зачем они нужны

Lua-скрипты в Redis — это мощный механизм, позволяющий выполнять сложную логику непосредственно на сервере Redis. Lua — это легковесный, быстрый скриптовый язык, встроенный в Redis начиная с версии 2.6.

### Преимущества использования Lua-скриптов

1. **Атомарность** — весь скрипт выполняется как единая атомарная операция
2. **Снижение latency** — уменьшение количества round-trips между клиентом и сервером
3. **Сложная логика** — возможность выполнять условные операции, циклы, вычисления
4. **Консистентность** — гарантия, что никакие другие команды не вмешаются во время выполнения

### Когда использовать Lua-скрипты

- Операции, требующие чтения и записи в одной транзакции
- Rate limiting и подобные паттерны
- Условные обновления (read-modify-write)
- Операции над несколькими ключами атомарно

---

## Команда EVAL — синтаксис и параметры

Команда `EVAL` выполняет Lua-скрипт на сервере Redis.

### Синтаксис

```
EVAL script numkeys [key [key ...]] [arg [arg ...]]
```

- **script** — текст Lua-скрипта
- **numkeys** — количество ключей, передаваемых в скрипт
- **key** — имена ключей Redis (доступны через массив `KEYS`)
- **arg** — дополнительные аргументы (доступны через массив `ARGV`)

### Простой пример

```bash
# Получить значение ключа
EVAL "return redis.call('GET', KEYS[1])" 1 mykey

# Установить значение
EVAL "return redis.call('SET', KEYS[1], ARGV[1])" 1 mykey "Hello"

# Инкремент значения
EVAL "return redis.call('INCR', KEYS[1])" 1 counter
```

### Пример с несколькими ключами и аргументами

```bash
EVAL "
    local val1 = redis.call('GET', KEYS[1])
    local val2 = redis.call('GET', KEYS[2])
    local sum = (tonumber(val1) or 0) + (tonumber(val2) or 0)
    redis.call('SET', KEYS[3], sum)
    return sum
" 3 num1 num2 result
```

---

## KEYS и ARGV — передача ключей и аргументов

В Lua-скриптах Redis предоставляет два глобальных массива:

### KEYS — массив ключей

```lua
-- KEYS индексируются с 1 (Lua-соглашение)
local key1 = KEYS[1]  -- первый ключ
local key2 = KEYS[2]  -- второй ключ

-- Получить количество ключей
local num_keys = #KEYS
```

### ARGV — массив аргументов

```lua
-- ARGV также индексируется с 1
local arg1 = ARGV[1]  -- первый аргумент
local arg2 = ARGV[2]  -- второй аргумент

-- Количество аргументов
local num_args = #ARGV
```

### Пример с обоими массивами

```bash
# Установить несколько ключей с TTL
EVAL "
    local ttl = tonumber(ARGV[1])
    for i, key in ipairs(KEYS) do
        redis.call('SET', key, ARGV[i + 1])
        redis.call('EXPIRE', key, ttl)
    end
    return #KEYS
" 3 key1 key2 key3 3600 "value1" "value2" "value3"
```

### Важное правило

**Все ключи, к которым обращается скрипт, ДОЛЖНЫ передаваться через KEYS**. Это необходимо для:
- Корректной работы Redis Cluster (ключи маршрутизируются по слотам)
- Правильной репликации
- Анализа зависимостей скриптов

```lua
-- ПРАВИЛЬНО
redis.call('GET', KEYS[1])

-- НЕПРАВИЛЬНО (хардкод ключа)
redis.call('GET', 'mykey')
```

---

## EVALSHA и кэширование скриптов

При частом использовании одного скрипта неэффективно каждый раз передавать его полный текст. Redis предоставляет механизм кэширования скриптов.

### SCRIPT LOAD — загрузка скрипта

```bash
# Загрузить скрипт и получить его SHA1-хеш
SCRIPT LOAD "return redis.call('GET', KEYS[1])"
# Вернёт: "a42059b356c875f0717db19a51f6aaa9161e77a2"
```

### EVALSHA — выполнение по хешу

```bash
# Выполнить скрипт по SHA1-хешу
EVALSHA a42059b356c875f0717db19a51f6aaa9161e77a2 1 mykey
```

### SCRIPT EXISTS — проверка наличия скрипта

```bash
# Проверить, загружен ли скрипт (можно несколько хешей)
SCRIPT EXISTS a42059b356c875f0717db19a51f6aaa9161e77a2
# Вернёт: 1) (integer) 1
```

### SCRIPT FLUSH — очистка кэша скриптов

```bash
# Удалить все закэшированные скрипты
SCRIPT FLUSH

# С Redis 6.2+
SCRIPT FLUSH ASYNC  # асинхронно
SCRIPT FLUSH SYNC   # синхронно
```

### Паттерн использования в клиенте

```python
import redis
import hashlib

class LuaScript:
    def __init__(self, client, script):
        self.client = client
        self.script = script
        self.sha = hashlib.sha1(script.encode()).hexdigest()

    def execute(self, keys=[], args=[]):
        try:
            return self.client.evalsha(self.sha, len(keys), *keys, *args)
        except redis.exceptions.NoScriptError:
            # Скрипт не в кэше — загружаем и выполняем
            return self.client.eval(self.script, len(keys), *keys, *args)

# Использование
r = redis.Redis()
script = LuaScript(r, "return redis.call('GET', KEYS[1])")
result = script.execute(keys=['mykey'])
```

---

## Атомарность выполнения скриптов

### Гарантии атомарности

Lua-скрипты в Redis выполняются **атомарно**:
- Никакая другая команда не может быть выполнена, пока скрипт работает
- Все операции скрипта происходят в изоляции
- Скрипт либо выполняется полностью, либо не выполняется вовсе

### Сравнение с транзакциями (MULTI/EXEC)

| Аспект | MULTI/EXEC | Lua-скрипты |
|--------|-----------|-------------|
| Атомарность | Да | Да |
| Условная логика | Нет (только WATCH) | Да |
| Промежуточные вычисления | Нет | Да |
| Чтение во время выполнения | Нет | Да |

### Пример атомарной операции

```lua
-- Атомарный "compare-and-swap"
local current = redis.call('GET', KEYS[1])
if current == ARGV[1] then
    redis.call('SET', KEYS[1], ARGV[2])
    return 1
else
    return 0
end
```

```bash
# Использование
EVAL "..." 1 mykey "expected_value" "new_value"
```

---

## Доступные функции Redis в Lua

### redis.call()

Выполняет команду Redis и возвращает результат. При ошибке **прерывает выполнение скрипта**.

```lua
-- Успешное выполнение
local value = redis.call('GET', 'mykey')

-- При ошибке скрипт остановится
redis.call('INVALID_COMMAND')  -- Error!
```

### redis.pcall()

"Protected call" — выполняет команду, но при ошибке **возвращает объект ошибки** вместо прерывания скрипта.

```lua
-- Безопасный вызов
local result = redis.pcall('GET', 'mykey')

-- Проверка на ошибку
if result.err then
    redis.log(redis.LOG_WARNING, "Error: " .. result.err)
    return nil
end

return result
```

### Таблица преобразования типов

| Redis тип | Lua тип |
|-----------|---------|
| Integer | Number |
| Bulk String | String |
| Array | Table |
| Status (OK) | Table с полем ok |
| Error | Table с полем err |
| Nil | Boolean false |

### Примеры преобразований

```lua
-- Integer -> Number
local count = redis.call('INCR', 'counter')  -- 42

-- Array -> Table
local list = redis.call('LRANGE', 'mylist', 0, -1)
for i, v in ipairs(list) do
    print(i, v)
end

-- Nil -> false
local value = redis.call('GET', 'nonexistent')
if not value then
    print("Key does not exist")
end
```

### Дополнительные функции

```lua
-- Логирование (см. раздел отладки)
redis.log(redis.LOG_WARNING, "message")

-- SHA1 хеширование
local hash = redis.sha1hex("data")

-- Преобразование ошибки в формат Redis
redis.error_reply("Custom error message")

-- Преобразование статуса в формат Redis
redis.status_reply("OK")
```

---

## Практические примеры

### 1. Rate Limiting (Token Bucket)

```lua
-- Rate limiter с использованием Token Bucket
-- KEYS[1]: ключ для хранения токенов
-- ARGV[1]: максимальное количество токенов (bucket size)
-- ARGV[2]: скорость пополнения (токенов в секунду)
-- ARGV[3]: количество запрашиваемых токенов
-- ARGV[4]: текущее время в миллисекундах

local key = KEYS[1]
local max_tokens = tonumber(ARGV[1])
local refill_rate = tonumber(ARGV[2])
local requested = tonumber(ARGV[3])
local now = tonumber(ARGV[4])

-- Получаем текущее состояние
local data = redis.call('HMGET', key, 'tokens', 'last_refill')
local tokens = tonumber(data[1]) or max_tokens
local last_refill = tonumber(data[2]) or now

-- Вычисляем пополнение токенов
local elapsed = (now - last_refill) / 1000
local refill = elapsed * refill_rate
tokens = math.min(max_tokens, tokens + refill)

-- Проверяем, достаточно ли токенов
local allowed = 0
if tokens >= requested then
    tokens = tokens - requested
    allowed = 1
end

-- Сохраняем состояние
redis.call('HSET', key, 'tokens', tokens, 'last_refill', now)
redis.call('EXPIRE', key, math.ceil(max_tokens / refill_rate) + 1)

return allowed
```

```bash
# Использование: 10 токенов макс, 1 токен/сек, запрос 1 токена
EVAL "..." 1 "ratelimit:user:123" 10 1 1 1703683200000
```

### 2. Sliding Window Rate Limiter

```lua
-- Sliding window rate limiter
-- KEYS[1]: ключ для sorted set
-- ARGV[1]: окно в секундах
-- ARGV[2]: максимальное количество запросов
-- ARGV[3]: текущий timestamp

local key = KEYS[1]
local window = tonumber(ARGV[1])
local max_requests = tonumber(ARGV[2])
local now = tonumber(ARGV[3])
local window_start = now - window * 1000

-- Удаляем устаревшие записи
redis.call('ZREMRANGEBYSCORE', key, '-inf', window_start)

-- Считаем текущие запросы
local current = redis.call('ZCARD', key)

if current < max_requests then
    -- Добавляем текущий запрос
    redis.call('ZADD', key, now, now .. ':' .. math.random())
    redis.call('EXPIRE', key, window)
    return 1  -- Разрешено
else
    return 0  -- Отклонено
end
```

### 3. Атомарный инкремент с проверкой

```lua
-- Инкремент только если значение меньше лимита
-- KEYS[1]: ключ счётчика
-- ARGV[1]: лимит
-- ARGV[2]: значение инкремента (опционально, по умолчанию 1)

local key = KEYS[1]
local limit = tonumber(ARGV[1])
local increment = tonumber(ARGV[2]) or 1

local current = tonumber(redis.call('GET', key)) or 0

if current + increment <= limit then
    local new_value = redis.call('INCRBY', key, increment)
    return new_value
else
    return nil  -- Лимит превышен
end
```

```bash
EVAL "..." 1 "counter" 100 5
```

### 4. Работа с несколькими ключами — атомарный перевод

```lua
-- Атомарный перевод "денег" между аккаунтами
-- KEYS[1]: аккаунт отправителя
-- KEYS[2]: аккаунт получателя
-- ARGV[1]: сумма перевода

local from = KEYS[1]
local to = KEYS[2]
local amount = tonumber(ARGV[1])

-- Получаем баланс отправителя
local balance = tonumber(redis.call('GET', from)) or 0

if balance < amount then
    return {err = "Insufficient funds"}
end

-- Выполняем перевод
redis.call('DECRBY', from, amount)
redis.call('INCRBY', to, amount)

return {
    from_balance = redis.call('GET', from),
    to_balance = redis.call('GET', to)
}
```

### 5. Атомарное добавление в список с лимитом

```lua
-- Добавить элемент в список, сохраняя только последние N элементов
-- KEYS[1]: ключ списка
-- ARGV[1]: значение для добавления
-- ARGV[2]: максимальный размер списка

local key = KEYS[1]
local value = ARGV[1]
local max_size = tonumber(ARGV[2])

redis.call('LPUSH', key, value)
redis.call('LTRIM', key, 0, max_size - 1)

return redis.call('LLEN', key)
```

### 6. Распределённая блокировка (Lock)

```lua
-- Попытка получить блокировку
-- KEYS[1]: ключ блокировки
-- ARGV[1]: уникальный идентификатор владельца
-- ARGV[2]: TTL в секундах

local key = KEYS[1]
local owner = ARGV[1]
local ttl = tonumber(ARGV[2])

-- Пытаемся установить блокировку (SET NX EX)
local result = redis.call('SET', key, owner, 'NX', 'EX', ttl)

if result then
    return 1  -- Блокировка получена
else
    return 0  -- Блокировка занята
end
```

```lua
-- Снятие блокировки (только владельцем)
-- KEYS[1]: ключ блокировки
-- ARGV[1]: уникальный идентификатор владельца

local key = KEYS[1]
local owner = ARGV[1]

local current_owner = redis.call('GET', key)

if current_owner == owner then
    redis.call('DEL', key)
    return 1  -- Успешно снята
else
    return 0  -- Не владелец
end
```

---

## Отладка скриптов (redis.log)

### Уровни логирования

```lua
redis.log(redis.LOG_DEBUG, "Debug message")
redis.log(redis.LOG_VERBOSE, "Verbose message")
redis.log(redis.LOG_NOTICE, "Notice message")
redis.log(redis.LOG_WARNING, "Warning message")
```

### Пример отладки

```lua
local key = KEYS[1]
local value = ARGV[1]

redis.log(redis.LOG_DEBUG, "Processing key: " .. key)
redis.log(redis.LOG_DEBUG, "Value: " .. tostring(value))

local current = redis.call('GET', key)
redis.log(redis.LOG_DEBUG, "Current value: " .. tostring(current))

-- Логика скрипта...

redis.log(redis.LOG_NOTICE, "Script completed successfully")
return result
```

### Просмотр логов

Логи записываются в лог-файл Redis. Для просмотра:

```bash
# В конфигурации Redis
loglevel debug

# Просмотр логов
tail -f /var/log/redis/redis-server.log
```

### Советы по отладке

1. **Используйте MONITOR** для отслеживания всех команд
   ```bash
   redis-cli MONITOR
   ```

2. **Возвращайте промежуточные результаты** во время разработки
   ```lua
   return {
       debug_current = current,
       debug_new = new_value,
       result = final_result
   }
   ```

3. **Тестируйте локально** перед деплоем

---

## Ограничения и таймауты

### Таймаут выполнения скрипта

По умолчанию скрипт может выполняться **5 секунд**. После этого:

```bash
# Настройка в redis.conf (в миллисекундах)
lua-time-limit 5000

# Или динамически
CONFIG SET lua-time-limit 10000
```

### Поведение при превышении таймаута

1. Redis начинает отвечать на запросы с ошибкой `BUSY`
2. Можно выполнить только:
   - `SCRIPT KILL` — прервать скрипт (если он не выполнял запись)
   - `SHUTDOWN NOSAVE` — экстренное выключение

```bash
# Прервать долгий скрипт
SCRIPT KILL
```

### Ограничения Lua в Redis

1. **Нет доступа к системным функциям**
   - Нет файлового I/O
   - Нет сетевых операций
   - Нет загрузки внешних модулей

2. **Ограниченная стандартная библиотека**
   - Доступны: `table`, `string`, `math`, `bit`, `struct`, `cjson`, `cmsgpack`
   - Недоступны: `os`, `io`, `debug` (кроме `debug.traceback`)

3. **Максимальный размер возвращаемого значения**
   - Зависит от `proto-max-bulk-len` (по умолчанию 512 MB)

4. **Нет глобального состояния между вызовами**
   - Каждый вызов скрипта изолирован
   - Глобальные переменные сбрасываются

### Пример работы с CJSON

```lua
local cjson = require('cjson')

-- Чтение JSON
local json_str = redis.call('GET', KEYS[1])
if json_str then
    local data = cjson.decode(json_str)
    data.updated_at = ARGV[1]
    redis.call('SET', KEYS[1], cjson.encode(data))
end

return "OK"
```

---

## Best Practices

### 1. Детерминизм скриптов

Скрипты **должны быть детерминированными** — при одинаковых входных данных давать одинаковый результат.

```lua
-- НЕПРАВИЛЬНО: использование случайных чисел
local id = math.random()

-- ПРАВИЛЬНО: передавать случайное значение как аргумент
local id = ARGV[1]  -- сгенерировано клиентом

-- НЕПРАВИЛЬНО: использование текущего времени
local now = os.time()  -- os недоступен

-- ПРАВИЛЬНО: передавать время как аргумент
local now = tonumber(ARGV[1])
```

**Почему это важно:**
- Репликация: скрипты выполняются на репликах
- AOF: скрипты могут переигрываться
- Если скрипт недетерминирован, данные на репликах могут разойтись

### 2. Всегда передавайте ключи через KEYS

```lua
-- ПРАВИЛЬНО
redis.call('GET', KEYS[1])

-- НЕПРАВИЛЬНО
redis.call('GET', 'hardcoded_key')
redis.call('GET', 'prefix:' .. ARGV[1])  -- динамический ключ

-- ПРАВИЛЬНО для динамических ключей
-- Клиент должен вычислить все ключи заранее
```

### 3. Оптимизация производительности

```lua
-- Используйте локальные переменные
local call = redis.call  -- кэширование функции

for i = 1, 1000 do
    call('SET', 'key:' .. i, i)
end

-- Batch операции вместо циклов где возможно
redis.call('MSET', 'k1', 'v1', 'k2', 'v2', 'k3', 'v3')

-- Используйте pipeling внутри скрипта осторожно
-- Все команды всё равно выполняются последовательно
```

### 4. Обработка ошибок

```lua
-- Валидация входных данных
if not KEYS[1] then
    return redis.error_reply("Missing key argument")
end

local amount = tonumber(ARGV[1])
if not amount or amount <= 0 then
    return redis.error_reply("Invalid amount")
end

-- Используйте pcall для некритичных операций
local result = redis.pcall('GET', KEYS[1])
if type(result) == 'table' and result.err then
    redis.log(redis.LOG_WARNING, "Error: " .. result.err)
end
```

### 5. Держите скрипты короткими

```lua
-- ПЛОХО: делать всё в одном огромном скрипте

-- ХОРОШО: разбивать на логические единицы
-- Скрипт 1: rate_limit.lua
-- Скрипт 2: update_stats.lua
-- Скрипт 3: notify_subscribers.lua
```

### 6. Используйте EVALSHA в продакшене

```python
# Паттерн в Python
class ScriptManager:
    def __init__(self, redis_client):
        self.redis = redis_client
        self.scripts = {}

    def register(self, name, script):
        sha = self.redis.script_load(script)
        self.scripts[name] = sha

    def run(self, name, keys=[], args=[]):
        sha = self.scripts[name]
        try:
            return self.redis.evalsha(sha, len(keys), *keys, *args)
        except NoScriptError:
            # После SCRIPT FLUSH или перезагрузки
            script = self.get_script_source(name)
            sha = self.redis.script_load(script)
            self.scripts[name] = sha
            return self.redis.evalsha(sha, len(keys), *keys, *args)
```

### 7. Документируйте скрипты

```lua
--[[
    Rate limiter using sliding window algorithm

    KEYS:
        [1] - Key for storing the sliding window (sorted set)

    ARGV:
        [1] - Window size in seconds
        [2] - Maximum allowed requests
        [3] - Current timestamp in milliseconds

    Returns:
        1 if request is allowed
        0 if rate limit exceeded

    Example:
        EVALSHA <sha> 1 "ratelimit:user:123" 60 100 1703683200000
--]]

local key = KEYS[1]
-- ... script body
```

---

## Связанные темы

- [Redis Transactions](./03-transactions.md) — альтернативный способ атомарных операций
- [Redis Pub/Sub](./05-pub-sub.md) — паттерны с использованием Lua
- [Redis Functions](https://redis.io/docs/manual/programmability/functions-intro/) — новый способ (Redis 7.0+)

## Дополнительные ресурсы

- [Redis Lua API Reference](https://redis.io/docs/manual/programmability/lua-api/)
- [EVAL command documentation](https://redis.io/commands/eval/)
- [Lua 5.1 Reference Manual](https://www.lua.org/manual/5.1/)
