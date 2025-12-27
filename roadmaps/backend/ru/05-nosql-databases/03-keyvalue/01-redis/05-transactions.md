# Транзакции в Redis

## Что такое транзакции в Redis

Транзакция в Redis — это механизм группировки нескольких команд в единый блок, который выполняется **последовательно и изолированно**. Пока транзакция выполняется, никакие другие команды от других клиентов не могут вклиниться между командами транзакции.

### Ключевые особенности

| Характеристика | Описание |
|----------------|----------|
| **Атомарность** | Все команды выполняются как единое целое (но без rollback!) |
| **Изоляция** | Во время выполнения другие клиенты не могут прервать транзакцию |
| **Последовательность** | Команды выполняются в порядке добавления в очередь |
| **Нет блокировок** | Используется оптимистичная блокировка через WATCH |

### Важное отличие от SQL-транзакций

В отличие от реляционных баз данных, Redis **не поддерживает откат (rollback)**. Если одна команда в транзакции завершится с ошибкой, остальные команды всё равно будут выполнены.

---

## Основные команды

### MULTI — начало транзакции

Команда `MULTI` сигнализирует о начале транзакционного блока. После неё все последующие команды будут помещаться в очередь, а не выполняться немедленно.

```redis
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET user:1:name "Alice"
QUEUED
127.0.0.1:6379(TX)> SET user:1:balance 1000
QUEUED
127.0.0.1:6379(TX)> INCR user:1:visits
QUEUED
```

### EXEC — выполнение транзакции

Команда `EXEC` выполняет все команды из очереди атомарно и возвращает массив результатов.

```redis
127.0.0.1:6379(TX)> EXEC
1) OK
2) OK
3) (integer) 1
```

### DISCARD — отмена транзакции

Команда `DISCARD` очищает очередь команд и выходит из транзакционного режима.

```redis
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET key1 "value1"
QUEUED
127.0.0.1:6379(TX)> DISCARD
OK
127.0.0.1:6379> GET key1
(nil)
```

---

## WATCH — оптимистичная блокировка

### Концепция Optimistic Locking

`WATCH` реализует паттерн **Check-and-Set (CAS)**. Вы "наблюдаете" за ключами, и если кто-то изменит их до выполнения `EXEC`, транзакция будет отменена.

```redis
WATCH mykey
val = GET mykey
val = val + 1
MULTI
SET mykey $val
EXEC
```

### Как это работает

1. `WATCH key1 key2 ...` — начинает отслеживать указанные ключи
2. Клиент читает текущие значения
3. `MULTI` — начинает транзакцию
4. Команды добавляются в очередь
5. `EXEC`:
   - Если ключи **не изменились** → транзакция выполняется
   - Если ключи **изменились** → `EXEC` возвращает `nil`, транзакция отменена

### Пример с WATCH

```python
import redis

r = redis.Redis()

def safe_increment(key):
    with r.pipeline() as pipe:
        while True:
            try:
                # Начинаем наблюдение за ключом
                pipe.watch(key)

                # Читаем текущее значение
                current_value = pipe.get(key)
                current_value = int(current_value) if current_value else 0

                # Начинаем транзакцию
                pipe.multi()

                # Добавляем команду в очередь
                pipe.set(key, current_value + 1)

                # Выполняем транзакцию
                pipe.execute()
                break  # Успех — выходим из цикла

            except redis.WatchError:
                # Ключ был изменён другим клиентом, пробуем снова
                continue

safe_increment("counter")
```

### UNWATCH

Команда `UNWATCH` снимает наблюдение со всех ключей.

```redis
WATCH key1
# Передумали делать транзакцию
UNWATCH
```

> **Примечание:** `EXEC` и `DISCARD` автоматически снимают наблюдение со всех ключей.

---

## Атомарность транзакций

### Как работает атомарность в Redis

Redis — однопоточный (в части обработки команд), поэтому транзакции выполняются **последовательно без прерываний**:

```
Клиент A: MULTI → SET x 1 → SET y 2 → EXEC
Клиент B: GET x → GET y

Возможные сценарии:
1. B читает ДО транзакции A: x=nil, y=nil
2. B читает ПОСЛЕ транзакции A: x=1, y=2

Невозможный сценарий:
- B читает x=1, y=nil (частичное чтение)
```

### Что НЕ гарантирует атомарность Redis

- **Rollback при ошибке** — если команда упадёт, остальные продолжат выполняться
- **Изоляцию чтения** — другие клиенты могут читать данные между MULTI и EXEC (данные ещё старые)

---

## Практические примеры

### Пример 1: Перевод денег между счетами

```python
import redis

r = redis.Redis(decode_responses=True)

def transfer_money(from_account, to_account, amount):
    """
    Атомарный перевод денег между счетами с оптимистичной блокировкой.
    """
    from_key = f"account:{from_account}:balance"
    to_key = f"account:{to_account}:balance"

    with r.pipeline() as pipe:
        while True:
            try:
                # Наблюдаем за обоими ключами
                pipe.watch(from_key, to_key)

                # Читаем текущие балансы
                from_balance = int(pipe.get(from_key) or 0)
                to_balance = int(pipe.get(to_key) or 0)

                # Проверяем достаточность средств
                if from_balance < amount:
                    pipe.unwatch()
                    raise ValueError(f"Недостаточно средств: {from_balance} < {amount}")

                # Начинаем транзакцию
                pipe.multi()

                # Списываем и зачисляем
                pipe.set(from_key, from_balance - amount)
                pipe.set(to_key, to_balance + amount)

                # Логируем операцию
                pipe.lpush("transactions:log",
                          f"{from_account} -> {to_account}: {amount}")

                # Выполняем
                pipe.execute()

                print(f"Перевод выполнен: {amount} от {from_account} к {to_account}")
                break

            except redis.WatchError:
                # Кто-то изменил баланс — пробуем снова
                print("Конфликт, повторяем...")
                continue

# Инициализация счетов
r.set("account:alice:balance", 1000)
r.set("account:bob:balance", 500)

# Выполняем перевод
transfer_money("alice", "bob", 200)

# Проверяем результат
print(f"Alice: {r.get('account:alice:balance')}")  # 800
print(f"Bob: {r.get('account:bob:balance')}")      # 700
```

### Пример 2: Атомарный инкремент нескольких ключей

```python
import redis

r = redis.Redis(decode_responses=True)

def increment_multiple_keys(keys, increment=1):
    """
    Атомарно увеличивает значения нескольких ключей.
    """
    with r.pipeline() as pipe:
        while True:
            try:
                # Наблюдаем за всеми ключами
                pipe.watch(*keys)

                # Читаем текущие значения
                current_values = {}
                for key in keys:
                    val = pipe.get(key)
                    current_values[key] = int(val) if val else 0

                # Начинаем транзакцию
                pipe.multi()

                # Увеличиваем все значения
                for key in keys:
                    new_value = current_values[key] + increment
                    pipe.set(key, new_value)

                # Выполняем
                results = pipe.execute()

                return {key: current_values[key] + increment for key in keys}

            except redis.WatchError:
                continue

# Использование
keys = ["counter:page1", "counter:page2", "counter:page3"]
new_values = increment_multiple_keys(keys, increment=5)
print(new_values)
```

### Пример 3: Атомарное обновление с использованием INCRBY

Если операция сводится к инкременту, можно использовать встроенные атомарные команды:

```python
import redis

r = redis.Redis()

def atomic_increment_simple(keys, increment=1):
    """
    Простой инкремент через pipeline (без WATCH).
    INCRBY и так атомарна, но pipeline группирует команды.
    """
    pipe = r.pipeline()

    for key in keys:
        pipe.incrby(key, increment)

    results = pipe.execute()
    return dict(zip(keys, results))

# Использование
results = atomic_increment_simple(["visits:today", "visits:total"], 1)
print(results)  # {'visits:today': 1, 'visits:total': 1}
```

---

## Ограничения транзакций Redis

### 1. Отсутствие Rollback

```redis
127.0.0.1:6379> SET foo "hello"
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> INCR foo
QUEUED
127.0.0.1:6379(TX)> SET bar "world"
QUEUED
127.0.0.1:6379(TX)> EXEC
1) (error) ERR value is not an integer or out of range
2) OK
```

Обратите внимание: `SET bar "world"` **выполнился**, несмотря на ошибку в `INCR foo`.

### 2. Нет вложенных транзакций

```redis
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> MULTI
(error) ERR MULTI calls can not be nested
```

### 3. Нельзя принимать решения внутри транзакции

Все команды ставятся в очередь до EXEC, поэтому нельзя использовать результат одной команды для условия другой:

```redis
# ЭТО НЕВОЗМОЖНО в транзакции:
MULTI
val = GET mykey        # Возвращает QUEUED, а не значение!
IF val > 100 THEN      # Нельзя проверить условие
  SET mykey 100
EXEC
```

**Решение:** использовать Lua-скрипты.

### 4. WATCH не блокирует

`WATCH` — это **оптимистичная** блокировка. Другие клиенты могут свободно изменять ключи, просто ваша транзакция будет отменена.

---

## Обработка ошибок

### Два типа ошибок

| Тип ошибки | Когда возникает | Поведение |
|------------|-----------------|-----------|
| **Ошибка очереди** | При добавлении команды (синтаксис) | Транзакция отменяется целиком |
| **Ошибка выполнения** | При EXEC (логическая ошибка) | Остальные команды выполняются |

### Ошибка очереди (Queue Error)

```redis
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> SET foo "bar"
QUEUED
127.0.0.1:6379(TX)> BADCOMMAND
(error) ERR unknown command 'BADCOMMAND'
127.0.0.1:6379(TX)> SET bar "baz"
QUEUED
127.0.0.1:6379(TX)> EXEC
(error) EXECABORT Transaction discarded because of previous errors.
```

Ничего не выполнилось — транзакция отменена полностью.

### Ошибка выполнения (Execution Error)

```redis
127.0.0.1:6379> SET foo "hello"
OK
127.0.0.1:6379> MULTI
OK
127.0.0.1:6379(TX)> INCR foo
QUEUED
127.0.0.1:6379(TX)> SET bar "world"
QUEUED
127.0.0.1:6379(TX)> EXEC
1) (error) ERR value is not an integer or out of range
2) OK
```

`SET bar "world"` выполнился успешно, несмотря на ошибку в первой команде.

### Обработка в Python

```python
import redis

r = redis.Redis(decode_responses=True)

def safe_transaction():
    try:
        pipe = r.pipeline()
        pipe.multi()
        pipe.set("key1", "value1")
        pipe.incr("not_a_number")  # Это вызовет ошибку
        pipe.set("key2", "value2")

        results = pipe.execute(raise_on_error=False)

        for i, result in enumerate(results):
            if isinstance(result, redis.ResponseError):
                print(f"Команда {i} завершилась с ошибкой: {result}")
            else:
                print(f"Команда {i} успешно: {result}")

    except redis.ResponseError as e:
        print(f"Ошибка очереди: {e}")

safe_transaction()
```

---

## Сравнение с Lua-скриптами

| Аспект | Транзакции (MULTI/EXEC) | Lua-скрипты (EVAL) |
|--------|------------------------|-------------------|
| **Условная логика** | Нет (нельзя читать и решать) | Да (полноценный язык) |
| **Атомарность** | Да | Да |
| **Откат при ошибке** | Нет | Нет (но можно реализовать вручную) |
| **Производительность** | Меньше round-trips | Один round-trip |
| **Сложность** | Простые операции | Сложная логика |
| **Оптимистичная блокировка** | WATCH | Не нужен (скрипт атомарен) |
| **Отладка** | Проще | Сложнее |

### Когда использовать транзакции

- Простые операции без условий
- Когда нужна оптимистичная блокировка с повторами
- Батчевые операции записи

### Когда использовать Lua

- Нужна условная логика (if/else)
- Нужно читать и писать в одной атомарной операции
- Сложные вычисления на основе данных

### Пример эквивалента на Lua

Транзакция с WATCH:
```python
# Python с WATCH
def increment_if_less_than(key, max_value):
    with r.pipeline() as pipe:
        while True:
            try:
                pipe.watch(key)
                val = int(pipe.get(key) or 0)
                if val >= max_value:
                    pipe.unwatch()
                    return None
                pipe.multi()
                pipe.incr(key)
                return pipe.execute()[0]
            except redis.WatchError:
                continue
```

Эквивалент на Lua (проще и атомарнее):
```python
lua_script = """
local val = tonumber(redis.call('GET', KEYS[1]) or 0)
if val >= tonumber(ARGV[1]) then
    return nil
end
return redis.call('INCR', KEYS[1])
"""

increment_if_less = r.register_script(lua_script)
result = increment_if_less(keys=['mykey'], args=[100])
```

---

## Best Practices

### 1. Минимизируйте время между WATCH и EXEC

```python
# ❌ Плохо: долгие операции между WATCH и EXEC
pipe.watch(key)
result = expensive_external_api_call()  # Много времени!
pipe.multi()
pipe.set(key, result)
pipe.execute()  # Высокий шанс WatchError

# ✅ Хорошо: подготовьте всё заранее
result = expensive_external_api_call()  # До WATCH
pipe.watch(key)
current = pipe.get(key)  # Быстрая проверка
pipe.multi()
pipe.set(key, result)
pipe.execute()
```

### 2. Обрабатывайте WatchError с лимитом попыток

```python
MAX_RETRIES = 3

def safe_update(key, new_value):
    for attempt in range(MAX_RETRIES):
        try:
            with r.pipeline() as pipe:
                pipe.watch(key)
                pipe.multi()
                pipe.set(key, new_value)
                return pipe.execute()
        except redis.WatchError:
            if attempt == MAX_RETRIES - 1:
                raise Exception("Не удалось выполнить транзакцию")
            continue
```

### 3. Используйте pipeline даже без MULTI для батчинга

```python
# Отправка 1000 команд одним запросом
pipe = r.pipeline(transaction=False)  # Без MULTI/EXEC
for i in range(1000):
    pipe.set(f"key:{i}", f"value:{i}")
pipe.execute()  # Один round-trip
```

### 4. Проверяйте ошибки выполнения

```python
results = pipe.execute(raise_on_error=False)
for result in results:
    if isinstance(result, redis.ResponseError):
        # Логируем или обрабатываем
        logger.error(f"Redis error: {result}")
```

### 5. Используйте Lua для сложной логики

```python
# ❌ Сложная логика с WATCH — много retry, race conditions
# ✅ Lua-скрипт — один атомарный вызов
```

### 6. Избегайте длинных очередей команд

Чем больше команд в транзакции, тем дольше блокируется обработка других клиентов.

```python
# ❌ Плохо: 10000 команд в одной транзакции
pipe.multi()
for i in range(10000):
    pipe.set(f"key:{i}", i)
pipe.execute()

# ✅ Лучше: разбить на части
BATCH_SIZE = 1000
for batch_start in range(0, 10000, BATCH_SIZE):
    pipe = r.pipeline()
    for i in range(batch_start, min(batch_start + BATCH_SIZE, 10000)):
        pipe.set(f"key:{i}", i)
    pipe.execute()
```

---

## Резюме

| Команда | Назначение |
|---------|------------|
| `MULTI` | Начать транзакцию |
| `EXEC` | Выполнить все команды в очереди |
| `DISCARD` | Отменить транзакцию |
| `WATCH key [key ...]` | Следить за ключами для CAS |
| `UNWATCH` | Прекратить слежение |

### Ключевые моменты

1. **Транзакции Redis атомарны**, но без rollback
2. **WATCH** — оптимистичная блокировка для безопасного read-modify-write
3. **Ошибки очереди** отменяют транзакцию, **ошибки выполнения** — нет
4. Для **сложной логики** используйте Lua-скрипты
5. **Минимизируйте** время между WATCH и EXEC
