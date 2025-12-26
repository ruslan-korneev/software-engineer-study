# Мониторинг и оптимизация Redis

## Команда INFO

Основная команда для получения информации о состоянии Redis:

```redis
INFO [section]
```

### Основные секции INFO

```redis
# Все секции
INFO

# Конкретная секция
INFO memory
INFO stats
INFO replication
```

### Секция Server

```redis
> INFO server
redis_version:7.2.0
os:Linux 5.15.0
uptime_in_seconds:86400
uptime_in_days:1
hz:10
configured_hz:10
```

### Секция Clients

```redis
> INFO clients
connected_clients:10
blocked_clients:0
tracking_clients:0
clients_in_timeout_table:0
```

**Ключевые метрики:**
- `connected_clients` — текущие подключения (следить за ростом)
- `blocked_clients` — клиенты в BLPOP/BRPOP (может указывать на проблемы)

### Секция Memory

```redis
> INFO memory
used_memory:1073741824
used_memory_human:1.00G
used_memory_rss:1200000000
used_memory_peak:1500000000
used_memory_peak_human:1.40G
maxmemory:2147483648
maxmemory_human:2.00G
maxmemory_policy:allkeys-lru
mem_fragmentation_ratio:1.12
```

**Ключевые метрики:**
- `used_memory` — память под данные
- `used_memory_rss` — реальная память от ОС
- `mem_fragmentation_ratio` — >1.5 указывает на фрагментацию
- `maxmemory` — лимит памяти (0 = без лимита)

### Секция Stats

```redis
> INFO stats
total_connections_received:1000
total_commands_processed:50000
instantaneous_ops_per_sec:1500
rejected_connections:0
expired_keys:500
evicted_keys:100
keyspace_hits:45000
keyspace_misses:5000
```

**Важные метрики:**
- `keyspace_hits / keyspace_misses` — hit rate кэша
- `evicted_keys` — ключи, удалённые из-за maxmemory
- `rejected_connections` — отклонённые подключения (проблема!)

### Расчёт hit rate

```
Hit Rate = keyspace_hits / (keyspace_hits + keyspace_misses) × 100%

45000 / (45000 + 5000) = 90% hit rate
```

Цель: >90% для эффективного кэширования.

## MONITOR — отслеживание команд

Показывает все команды в реальном времени:

```redis
> MONITOR
OK
1609459200.123456 [0 127.0.0.1:12345] "GET" "user:123"
1609459200.234567 [0 127.0.0.1:12345] "SET" "user:123" "data"
1609459200.345678 [0 127.0.0.1:54321] "INCR" "counter"
```

**Предупреждение:** MONITOR снижает производительность на 50%! Использовать только для отладки.

## SLOWLOG — анализ медленных запросов

### Конфигурация

```redis
# Порог медленного запроса (микросекунды)
CONFIG SET slowlog-log-slower-than 10000  # 10ms

# Максимальное количество записей
CONFIG SET slowlog-max-len 128
```

### Просмотр медленных запросов

```redis
> SLOWLOG GET 10
1) 1) (integer) 14            # ID записи
   2) (integer) 1609459200    # Unix timestamp
   3) (integer) 15000         # Время выполнения (мкс)
   4) 1) "KEYS"               # Команда
      2) "*"
   5) "127.0.0.1:12345"       # Клиент
   6) ""                       # Имя клиента

> SLOWLOG LEN
(integer) 5

> SLOWLOG RESET
OK
```

### Типичные причины медленных запросов

1. **KEYS pattern** — сканирует все ключи, использовать SCAN
2. **HGETALL** на больших хэшах — использовать HSCAN
3. **SMEMBERS** на больших множествах — использовать SSCAN
4. **Большие значения** — разбивать на части

## Memory Management

### Maxmemory Policy

Что делать, когда память закончилась:

```redis
CONFIG SET maxmemory 2gb
CONFIG SET maxmemory-policy allkeys-lru
```

| Policy | Описание |
|--------|----------|
| `noeviction` | Ошибка при записи (default) |
| `allkeys-lru` | Удаляет LRU ключи |
| `volatile-lru` | Удаляет LRU ключи с TTL |
| `allkeys-random` | Удаляет случайные ключи |
| `volatile-random` | Удаляет случайные ключи с TTL |
| `volatile-ttl` | Удаляет ключи с наименьшим TTL |
| `allkeys-lfu` | Удаляет наименее используемые |
| `volatile-lfu` | Удаляет наименее используемые с TTL |

**Рекомендации:**
- Кэш: `allkeys-lru` или `allkeys-lfu`
- Сессии: `volatile-ttl`
- Важные данные: `noeviction` + мониторинг

### Анализ памяти

```redis
# Память под конкретный ключ
MEMORY USAGE key

# Статистика памяти
MEMORY STATS

# Рекомендации по памяти
MEMORY DOCTOR
```

### Поиск больших ключей

```bash
# Через redis-cli
redis-cli --bigkeys

# Результат:
# Biggest string: user:profile:123 (15000 bytes)
# Biggest list: queue:tasks (50000 items)
# Biggest set: users:online (10000 members)
```

## redis-benchmark

Инструмент для тестирования производительности:

```bash
# Базовый тест
redis-benchmark -h localhost -p 6379

# Конкретные тесты
redis-benchmark -t set,get -n 100000 -c 50

# С pipeline
redis-benchmark -t set -n 100000 -P 16
```

### Параметры

| Параметр | Описание |
|----------|----------|
| `-n` | Количество запросов |
| `-c` | Количество параллельных клиентов |
| `-P` | Количество команд в pipeline |
| `-t` | Тесты (set,get,incr,lpush,etc) |
| `-d` | Размер данных (bytes) |
| `-q` | Тихий режим (только результаты) |

### Пример вывода

```
====== SET ======
  100000 requests completed in 1.50 seconds
  50 parallel clients
  3 bytes payload

  99.50% <= 1 milliseconds
  99.99% <= 2 milliseconds
  100.00% <= 2 milliseconds
  66666.67 requests per second
```

## Инструменты мониторинга

### redis-cli --stat

Реальный мониторинг в консоли:

```bash
redis-cli --stat

------- data ------ --------------------- load -------------------- - child -
keys       mem      clients blocked requests            connections
10000      50.00M   10      0       100000 (+0)         100
10001      50.01M   10      0       100100 (+100)       100
```

### redis-cli --latency

Измерение задержки:

```bash
redis-cli --latency

min: 0, max: 1, avg: 0.20 (1000 samples)
```

### RedisInsight

Графический инструмент от Redis Labs:
- Визуализация ключей и структур данных
- Профилирование запросов
- Мониторинг в реальном времени
- Анализ памяти

## Best Practices оптимизации

### 1. Используйте правильные структуры данных

```redis
# Плохо: много отдельных ключей
SET user:1:name "John"
SET user:1:email "john@example.com"
SET user:1:age "30"

# Хорошо: один хэш
HSET user:1 name "John" email "john@example.com" age "30"
```

### 2. Используйте Pipeline

```python
# Плохо: 100 отдельных запросов
for i in range(100):
    r.set(f"key:{i}", f"value:{i}")

# Хорошо: один pipeline
pipe = r.pipeline()
for i in range(100):
    pipe.set(f"key:{i}", f"value:{i}")
pipe.execute()
```

### 3. Избегайте KEYS, используйте SCAN

```redis
# Плохо (блокирует Redis)
KEYS user:*

# Хорошо (итеративно)
SCAN 0 MATCH user:* COUNT 100
```

### 4. Устанавливайте TTL

```redis
# Всегда устанавливайте expire для кэша
SET session:abc123 "data" EX 3600
```

### 5. Сжимайте данные

```python
import zlib
import json

# Сжатие перед записью
data = {"large": "data" * 1000}
compressed = zlib.compress(json.dumps(data).encode())
r.set("key", compressed)

# Распаковка при чтении
data = json.loads(zlib.decompress(r.get("key")))
```

### 6. Используйте connection pooling

```python
# Плохо: новое соединение на каждый запрос
r = redis.Redis()
r.get("key")

# Хорошо: пул соединений
pool = redis.ConnectionPool(max_connections=10)
r = redis.Redis(connection_pool=pool)
```

## Алерты для production

Мониторьте эти метрики:

| Метрика | Warning | Critical |
|---------|---------|----------|
| Memory usage | >70% | >90% |
| Hit rate | <80% | <50% |
| Connected clients | >80% max | >95% max |
| Evicted keys | >0 | >100/min |
| Rejected connections | >0 | — |
| Replication lag | >1s | >10s |
