# Sorted Sets (Сортированные множества) в Redis

Sorted Sets (ZSets) — это коллекции уникальных строк, где каждый элемент связан с числовым значением (score). Элементы автоматически сортируются по score, что делает Sorted Sets идеальными для лидербордов, временных рядов и приоритетных очередей.

## Основные характеристики

- **Уникальность**: каждый элемент встречается только один раз
- **Сортировка**: элементы упорядочены по score (от меньшего к большему)
- **Быстрый доступ**: O(log N) для большинства операций
- **Диапазонные запросы**: эффективная выборка по score или позиции

## Базовые команды

### ZADD — добавление элементов

```redis
# Добавить элемент со score
ZADD leaderboard 100 "player:1"
# Результат: 1 (количество добавленных элементов)

# Добавить несколько элементов
ZADD leaderboard 200 "player:2" 150 "player:3" 300 "player:4"
# Результат: 3

# Обновить score существующего элемента
ZADD leaderboard 250 "player:1"
# Результат: 0 (элемент обновлён, не добавлен)

# Опции ZADD (Redis 3.0.2+)
ZADD leaderboard NX 500 "player:5"  # Только добавить новый
ZADD leaderboard XX 350 "player:2"  # Только обновить существующий
ZADD leaderboard GT 400 "player:1"  # Обновить, если новый score больше
ZADD leaderboard LT 100 "player:1"  # Обновить, если новый score меньше
ZADD leaderboard CH 600 "player:6"  # Вернуть кол-во изменённых (не только добавленных)
```

### ZREM — удаление элементов

```redis
ZADD myset 1 "one" 2 "two" 3 "three"

ZREM myset "two"
# Результат: 1

ZREM myset "one" "three" "nonexistent"
# Результат: 2
```

### ZSCORE — получить score элемента

```redis
ZADD leaderboard 100 "player:1" 200 "player:2"

ZSCORE leaderboard "player:1"
# Результат: "100"

ZSCORE leaderboard "nonexistent"
# Результат: nil
```

### ZRANK и ZREVRANK — получить позицию элемента

```redis
ZADD leaderboard 100 "alice" 200 "bob" 150 "charlie"

# Позиция от начала (меньший score = меньший ранг)
ZRANK leaderboard "alice"
# Результат: 0 (первый)

ZRANK leaderboard "bob"
# Результат: 2 (третий, т.к. score наибольший)

# Позиция с конца (больший score = меньший ранг)
ZREVRANK leaderboard "bob"
# Результат: 0 (первый с конца)

ZREVRANK leaderboard "alice"
# Результат: 2 (третий с конца)
```

### ZCARD — количество элементов

```redis
ZADD myset 1 "a" 2 "b" 3 "c"

ZCARD myset
# Результат: 3
```

## Диапазонные запросы

### ZRANGE и ZREVRANGE — по индексам

```redis
ZADD scores 100 "alice" 200 "bob" 150 "charlie" 175 "dave"

# Элементы по индексам (от 0)
ZRANGE scores 0 2
# Результат: ["alice", "charlie", "dave"] (первые 3 по возрастанию score)

# Все элементы
ZRANGE scores 0 -1
# Результат: ["alice", "charlie", "dave", "bob"]

# С конца (по убыванию score)
ZREVRANGE scores 0 2
# Результат: ["bob", "dave", "charlie"]

# С scores (WITHSCORES)
ZRANGE scores 0 -1 WITHSCORES
# Результат: ["alice", "100", "charlie", "150", "dave", "175", "bob", "200"]
```

### ZRANGEBYSCORE — по диапазону score

```redis
ZADD salaries 3000 "john" 4500 "jane" 5000 "bob" 6000 "alice" 8000 "charlie"

# Элементы со score от 4000 до 6000
ZRANGEBYSCORE salaries 4000 6000
# Результат: ["jane", "bob", "alice"]

# Исключить границы с помощью (
ZRANGEBYSCORE salaries (4000 (6000
# Результат: ["jane", "bob"]

# С отрицательной и положительной бесконечностью
ZRANGEBYSCORE salaries -inf +inf
# Результат: все элементы

ZRANGEBYSCORE salaries -inf 5000
# Результат: ["john", "jane", "bob"]

# С LIMIT (offset, count)
ZRANGEBYSCORE salaries 3000 8000 LIMIT 1 2
# Результат: ["jane", "bob"] (пропустить 1, взять 2)

# В обратном порядке
ZREVRANGEBYSCORE salaries 6000 4000
# Результат: ["alice", "bob", "jane"]
```

### ZRANGEBYLEX — лексикографическая сортировка

Когда все элементы имеют одинаковый score:

```redis
ZADD myset 0 "a" 0 "b" 0 "c" 0 "d" 0 "e" 0 "f" 0 "g"

# Элементы от "b" до "e" включительно
ZRANGEBYLEX myset [b [e
# Результат: ["b", "c", "d", "e"]

# Исключить границу с помощью (
ZRANGEBYLEX myset (b (e
# Результат: ["c", "d"]

# От начала до "c"
ZRANGEBYLEX myset - [c
# Результат: ["a", "b", "c"]

# От "d" до конца
ZRANGEBYLEX myset [d +
# Результат: ["d", "e", "f", "g"]
```

### ZCOUNT — количество элементов в диапазоне score

```redis
ZADD scores 10 "a" 20 "b" 30 "c" 40 "d" 50 "e"

ZCOUNT scores 20 40
# Результат: 3

ZCOUNT scores -inf +inf
# Результат: 5
```

## Модификация score

### ZINCRBY — инкремент score

```redis
ZADD leaderboard 100 "player:1"

ZINCRBY leaderboard 50 "player:1"
# Результат: "150"

ZINCRBY leaderboard -30 "player:1"
# Результат: "120"

# Если элемент не существует, создаётся с указанным score
ZINCRBY leaderboard 100 "player:new"
# Результат: "100"
```

## Операции над множествами

### ZUNIONSTORE — объединение

```redis
ZADD zset1 1 "a" 2 "b" 3 "c"
ZADD zset2 2 "b" 3 "c" 4 "d"

# Объединить с суммированием scores
ZUNIONSTORE out 2 zset1 zset2
# out: {"a": 1, "b": 4, "c": 6, "d": 4}

# С весами
ZUNIONSTORE out 2 zset1 zset2 WEIGHTS 1 2
# zset2 scores умножаются на 2
# out: {"a": 1, "b": 8, "c": 9, "d": 8}

# С агрегацией MIN, MAX или SUM
ZUNIONSTORE out 2 zset1 zset2 AGGREGATE MAX
# out: {"a": 1, "b": 3, "c": 3, "d": 4}
```

### ZINTERSTORE — пересечение

```redis
ZADD zset1 1 "a" 2 "b" 3 "c"
ZADD zset2 2 "b" 3 "c" 4 "d"

ZINTERSTORE out 2 zset1 zset2
# out: {"b": 4, "c": 6} (только общие элементы, scores суммируются)

ZINTERSTORE out 2 zset1 zset2 AGGREGATE MIN
# out: {"b": 2, "c": 3}
```

### ZDIFF (Redis 6.2+) — разность

```redis
ZADD zset1 1 "a" 2 "b" 3 "c"
ZADD zset2 2 "b" 4 "d"

ZDIFF 2 zset1 zset2
# Результат: ["a", "c"]

ZDIFF 2 zset1 zset2 WITHSCORES
# Результат: ["a", "1", "c", "3"]
```

## Удаление элементов

### ZREMRANGEBYRANK — по позиции

```redis
ZADD myset 1 "a" 2 "b" 3 "c" 4 "d" 5 "e"

# Удалить первые 2 элемента
ZREMRANGEBYRANK myset 0 1
# Результат: 2
# Осталось: {"c": 3, "d": 4, "e": 5}
```

### ZREMRANGEBYSCORE — по score

```redis
ZADD myset 1 "a" 2 "b" 3 "c" 4 "d" 5 "e"

# Удалить элементы со score от 2 до 4
ZREMRANGEBYSCORE myset 2 4
# Результат: 3
# Осталось: {"a": 1, "e": 5}
```

### ZPOPMIN и ZPOPMAX — извлечь крайние элементы

```redis
ZADD myset 1 "a" 2 "b" 3 "c" 4 "d"

# Извлечь элемент с минимальным score
ZPOPMIN myset
# Результат: ["a", "1"]

# Извлечь 2 элемента с максимальным score
ZPOPMAX myset 2
# Результат: ["d", "4", "c", "3"]

# Блокирующие версии
BZPOPMIN myset 30  # Ждать до 30 секунд
BZPOPMAX myset 0   # Ждать бесконечно
```

## Практические примеры использования

### Лидерборд (Таблица лидеров)

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0, decode_responses=True)

class Leaderboard:
    def __init__(self, name):
        self.key = f"leaderboard:{name}"

    def add_score(self, player_id, score):
        """Добавить или обновить счёт игрока."""
        r.zadd(self.key, {player_id: score})

    def increment_score(self, player_id, delta):
        """Увеличить счёт игрока."""
        return r.zincrby(self.key, delta, player_id)

    def get_rank(self, player_id):
        """Получить ранг игрока (1-based)."""
        rank = r.zrevrank(self.key, player_id)
        return rank + 1 if rank is not None else None

    def get_score(self, player_id):
        """Получить счёт игрока."""
        return r.zscore(self.key, player_id)

    def get_top(self, count=10):
        """Получить топ игроков."""
        results = r.zrevrange(self.key, 0, count - 1, withscores=True)
        return [
            {"player": player, "score": int(score), "rank": i + 1}
            for i, (player, score) in enumerate(results)
        ]

    def get_around_player(self, player_id, count=5):
        """Получить игроков вокруг указанного."""
        rank = r.zrevrank(self.key, player_id)
        if rank is None:
            return []

        start = max(0, rank - count // 2)
        end = start + count - 1

        results = r.zrevrange(self.key, start, end, withscores=True)
        return [
            {"player": player, "score": int(score), "rank": start + i + 1}
            for i, (player, score) in enumerate(results)
        ]

    def get_player_count(self):
        """Количество игроков в таблице."""
        return r.zcard(self.key)

    def remove_player(self, player_id):
        """Удалить игрока из таблицы."""
        r.zrem(self.key, player_id)

# Использование
lb = Leaderboard("game:monthly")

lb.add_score("player:1", 1500)
lb.add_score("player:2", 2300)
lb.add_score("player:3", 1800)

lb.increment_score("player:1", 200)  # Теперь 1700

top10 = lb.get_top(10)
# [{"player": "player:2", "score": 2300, "rank": 1}, ...]

rank = lb.get_rank("player:1")
# 3
```

### Rate Limiting (Скользящее окно)

```python
import time

class SlidingWindowRateLimiter:
    """Rate limiter на основе скользящего окна."""

    def __init__(self, name, max_requests, window_seconds):
        self.key_prefix = f"ratelimit:{name}"
        self.max_requests = max_requests
        self.window_seconds = window_seconds

    def is_allowed(self, identifier):
        """
        Проверить, разрешён ли запрос.
        Возвращает (allowed: bool, remaining: int, retry_after: float)
        """
        key = f"{self.key_prefix}:{identifier}"
        now = time.time()
        window_start = now - self.window_seconds

        pipe = r.pipeline()

        # Удалить старые записи
        pipe.zremrangebyscore(key, 0, window_start)

        # Подсчитать текущие запросы
        pipe.zcard(key)

        # Добавить текущий запрос
        pipe.zadd(key, {str(now): now})

        # Установить TTL
        pipe.expire(key, self.window_seconds)

        results = pipe.execute()
        current_count = results[1]

        if current_count >= self.max_requests:
            # Получить время самого старого запроса
            oldest = r.zrange(key, 0, 0, withscores=True)
            if oldest:
                retry_after = oldest[0][1] + self.window_seconds - now
            else:
                retry_after = self.window_seconds

            # Удалить добавленный запрос
            r.zrem(key, str(now))

            return False, 0, max(0, retry_after)

        remaining = self.max_requests - current_count - 1
        return True, remaining, 0

    def get_usage(self, identifier):
        """Получить текущее использование."""
        key = f"{self.key_prefix}:{identifier}"
        now = time.time()
        window_start = now - self.window_seconds

        r.zremrangebyscore(key, 0, window_start)
        count = r.zcard(key)

        return {
            "used": count,
            "remaining": max(0, self.max_requests - count),
            "limit": self.max_requests,
            "window": self.window_seconds
        }

# Использование
limiter = SlidingWindowRateLimiter("api", max_requests=100, window_seconds=60)

allowed, remaining, retry_after = limiter.is_allowed("user:123")
if allowed:
    print(f"Request allowed. Remaining: {remaining}")
else:
    print(f"Rate limited. Retry after: {retry_after:.2f}s")
```

### Временные ряды

```python
class TimeSeries:
    """Простые временные ряды на Sorted Sets."""

    def __init__(self, name, retention_seconds=86400):
        self.key = f"timeseries:{name}"
        self.retention = retention_seconds

    def add(self, value, timestamp=None):
        """Добавить значение."""
        if timestamp is None:
            timestamp = time.time()

        # Используем timestamp как score, value:timestamp как member
        member = f"{value}:{timestamp}"
        r.zadd(self.key, {member: timestamp})

        # Удалить старые данные
        cutoff = time.time() - self.retention
        r.zremrangebyscore(self.key, 0, cutoff)

    def get_range(self, start_time, end_time):
        """Получить значения в диапазоне времени."""
        results = r.zrangebyscore(self.key, start_time, end_time, withscores=True)
        return [
            {"value": float(member.split(":")[0]), "timestamp": score}
            for member, score in results
        ]

    def get_last(self, count=10):
        """Получить последние N значений."""
        results = r.zrevrange(self.key, 0, count - 1, withscores=True)
        return [
            {"value": float(member.split(":")[0]), "timestamp": score}
            for member, score in reversed(results)
        ]

    def get_count(self, start_time, end_time):
        """Количество точек в диапазоне."""
        return r.zcount(self.key, start_time, end_time)

# Использование
ts = TimeSeries("temperature:sensor1", retention_seconds=3600)

ts.add(23.5)
ts.add(24.1)
ts.add(23.8)

last_hour = ts.get_range(time.time() - 3600, time.time())
```

### Приоритетная очередь

```python
class PriorityQueue:
    """Приоритетная очередь на Sorted Sets."""

    def __init__(self, name):
        self.key = f"pqueue:{name}"

    def enqueue(self, item, priority):
        """
        Добавить элемент с приоритетом.
        Меньший priority = выше приоритет.
        """
        r.zadd(self.key, {item: priority})

    def dequeue(self):
        """Извлечь элемент с наивысшим приоритетом."""
        result = r.zpopmin(self.key)
        if result:
            return {"item": result[0][0], "priority": result[0][1]}
        return None

    def dequeue_blocking(self, timeout=0):
        """Блокирующее извлечение."""
        result = r.bzpopmin(self.key, timeout)
        if result:
            return {"item": result[1], "priority": result[2]}
        return None

    def peek(self):
        """Посмотреть элемент без извлечения."""
        result = r.zrange(self.key, 0, 0, withscores=True)
        if result:
            return {"item": result[0][0], "priority": result[0][1]}
        return None

    def size(self):
        """Размер очереди."""
        return r.zcard(self.key)

    def update_priority(self, item, new_priority):
        """Изменить приоритет элемента."""
        r.zadd(self.key, {item: new_priority}, xx=True)

# Использование
pq = PriorityQueue("tasks")

pq.enqueue("critical_task", priority=1)
pq.enqueue("normal_task", priority=5)
pq.enqueue("low_priority_task", priority=10)

task = pq.dequeue()
# {"item": "critical_task", "priority": 1}
```

### Отложенные задачи

```python
import json

class DelayedQueue:
    """Очередь отложенных задач."""

    def __init__(self, name):
        self.key = f"delayed:{name}"

    def schedule(self, task_data, execute_at):
        """Запланировать задачу на определённое время."""
        task_json = json.dumps(task_data)
        r.zadd(self.key, {task_json: execute_at})

    def schedule_delay(self, task_data, delay_seconds):
        """Запланировать задачу с задержкой."""
        execute_at = time.time() + delay_seconds
        self.schedule(task_data, execute_at)

    def get_ready_tasks(self, limit=10):
        """Получить задачи, готовые к выполнению."""
        now = time.time()
        tasks = r.zrangebyscore(self.key, 0, now, start=0, num=limit)
        return [json.loads(task) for task in tasks]

    def pop_ready_tasks(self, limit=10):
        """Извлечь задачи, готовые к выполнению."""
        now = time.time()

        # Атомарно получить и удалить с помощью Lua-скрипта
        script = """
        local tasks = redis.call('ZRANGEBYSCORE', KEYS[1], 0, ARGV[1], 'LIMIT', 0, ARGV[2])
        if #tasks > 0 then
            redis.call('ZREM', KEYS[1], unpack(tasks))
        end
        return tasks
        """
        tasks = r.eval(script, 1, self.key, now, limit)
        return [json.loads(task) for task in tasks]

    def pending_count(self):
        """Количество ожидающих задач."""
        return r.zcard(self.key)

# Использование
queue = DelayedQueue("emails")

# Отправить email через 5 минут
queue.schedule_delay(
    {"type": "email", "to": "user@example.com", "template": "welcome"},
    delay_seconds=300
)

# В worker-процессе
while True:
    tasks = queue.pop_ready_tasks(limit=10)
    for task in tasks:
        process_task(task)
    time.sleep(1)
```

## Сложность операций

| Команда | Сложность |
|---------|-----------|
| ZADD | O(log N) для каждого элемента |
| ZREM | O(M * log N), M — кол-во удаляемых |
| ZSCORE, ZRANK | O(1) |
| ZRANGE, ZREVRANGE | O(log N + M), M — кол-во элементов |
| ZRANGEBYSCORE | O(log N + M) |
| ZCOUNT | O(log N) |
| ZINCRBY | O(log N) |
| ZUNIONSTORE, ZINTERSTORE | O(N * K + M * log M) |

## Лучшие практики

1. **Используйте ZRANGEBYSCORE с LIMIT** для пагинации
2. **ZINCRBY для атомарного обновления** score
3. **ZREMRANGEBYSCORE** для очистки старых данных
4. **Блокирующие BZPOPMIN/BZPOPMAX** для очередей
5. **Комбинируйте с Lua-скриптами** для атомарных операций
6. **Используйте NX/XX/GT/LT** опции ZADD для условных обновлений
