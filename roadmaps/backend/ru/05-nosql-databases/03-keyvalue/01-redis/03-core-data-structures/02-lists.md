# Lists (Списки) в Redis

Списки в Redis — это упорядоченные коллекции строк, реализованные как связные списки. Они позволяют добавлять элементы с обоих концов с временной сложностью O(1). Максимальное количество элементов в списке — более 4 миллиардов.

## Основные характеристики

- **Упорядоченность**: элементы хранятся в порядке добавления
- **Дубликаты разрешены**: один и тот же элемент может встречаться несколько раз
- **Индексация**: доступ к элементам по индексу (от 0, отрицательные индексы с конца)
- **Двусторонние операции**: добавление/удаление с обоих концов

## Команды добавления элементов

### LPUSH и RPUSH

```redis
# LPUSH — добавить элементы в начало списка (слева)
LPUSH mylist "world"
LPUSH mylist "hello"
# Список: ["hello", "world"]

# Добавить несколько элементов за раз
LPUSH mylist "c" "b" "a"
# Список: ["a", "b", "c", "hello", "world"]
# Обратите внимание: элементы добавляются по одному слева

# RPUSH — добавить элементы в конец списка (справа)
RPUSH mylist "!"
# Список: ["a", "b", "c", "hello", "world", "!"]

# RPUSH с несколькими элементами
RPUSH tasks "task1" "task2" "task3"
# Список: ["task1", "task2", "task3"]
```

### LPUSHX и RPUSHX

Добавляют элемент только если список уже существует:

```redis
# Если mylist не существует, ничего не произойдёт
LPUSHX mylist "new_element"
RPUSHX mylist "new_element"

# Возвращают длину списка или 0, если список не существует
```

### LINSERT

Вставка элемента перед или после указанного:

```redis
RPUSH mylist "Hello" "World"

# Вставить "There" перед "World"
LINSERT mylist BEFORE "World" "There"
# Список: ["Hello", "There", "World"]

# Вставить "!" после "World"
LINSERT mylist AFTER "World" "!"
# Список: ["Hello", "There", "World", "!"]
```

## Команды извлечения элементов

### LPOP и RPOP

```redis
RPUSH mylist "one" "two" "three"

# Извлечь элемент с начала (слева)
LPOP mylist
# Результат: "one"
# Список: ["two", "three"]

# Извлечь элемент с конца (справа)
RPOP mylist
# Результат: "three"
# Список: ["two"]

# Извлечь несколько элементов (Redis 6.2+)
RPUSH numbers "1" "2" "3" "4" "5"
LPOP numbers 2
# Результат: ["1", "2"]

RPOP numbers 2
# Результат: ["5", "4"]
```

### LMOVE (Redis 6.2+)

Атомарное перемещение элемента между списками:

```redis
RPUSH src "one" "two" "three"

# Переместить элемент из src в dst
LMOVE src dst LEFT RIGHT
# Из начала src в конец dst
# src: ["two", "three"]
# dst: ["one"]

LMOVE src dst RIGHT LEFT
# Из конца src в начало dst
# src: ["two"]
# dst: ["three", "one"]
```

## Команды чтения

### LRANGE

Получить диапазон элементов (не удаляя их):

```redis
RPUSH mylist "a" "b" "c" "d" "e"

# Получить первые 3 элемента
LRANGE mylist 0 2
# Результат: ["a", "b", "c"]

# Получить все элементы
LRANGE mylist 0 -1
# Результат: ["a", "b", "c", "d", "e"]

# Получить последние 2 элемента
LRANGE mylist -2 -1
# Результат: ["d", "e"]

# Элементы с 1 по 3
LRANGE mylist 1 3
# Результат: ["b", "c", "d"]
```

### LINDEX

Получить элемент по индексу:

```redis
RPUSH mylist "Hello" "World"

LINDEX mylist 0
# Результат: "Hello"

LINDEX mylist 1
# Результат: "World"

LINDEX mylist -1
# Результат: "World" (последний элемент)

LINDEX mylist 100
# Результат: nil (индекс за пределами списка)
```

### LLEN

Получить длину списка:

```redis
RPUSH mylist "a" "b" "c"

LLEN mylist
# Результат: 3

LLEN nonexistent
# Результат: 0
```

## Модификация списка

### LSET

Установить значение элемента по индексу:

```redis
RPUSH mylist "one" "two" "three"

LSET mylist 1 "hello"
# Список: ["one", "hello", "three"]

# Ошибка, если индекс за пределами списка
LSET mylist 100 "value"
# (error) ERR index out of range
```

### LREM

Удалить элементы по значению:

```redis
RPUSH mylist "hello" "hello" "foo" "hello"

# Удалить 2 элемента "hello" с начала
LREM mylist 2 "hello"
# Список: ["foo", "hello"]

# count > 0: удалять с начала
# count < 0: удалять с конца
# count = 0: удалить все вхождения

RPUSH mylist2 "a" "b" "a" "c" "a"
LREM mylist2 -1 "a"  # Удалить 1 элемент с конца
# Список: ["a", "b", "a", "c"]

LREM mylist2 0 "a"   # Удалить все "a"
# Список: ["b", "c"]
```

### LTRIM

Обрезать список до указанного диапазона:

```redis
RPUSH mylist "one" "two" "three" "four" "five"

# Оставить только первые 3 элемента
LTRIM mylist 0 2
# Список: ["one", "two", "three"]

# Оставить последние 2 элемента
RPUSH mylist2 "a" "b" "c" "d" "e"
LTRIM mylist2 -2 -1
# Список: ["d", "e"]
```

## Блокирующие операции

Блокирующие команды ожидают появления элементов в списке, если он пуст.

### BLPOP и BRPOP

```redis
# Ожидать элемент из списка до 10 секунд
BLPOP mylist 10
# Если список пуст, команда заблокируется на 10 секунд

# Результат при появлении элемента:
# 1) "mylist"
# 2) "element_value"

# Ожидать из нескольких списков
BLPOP list1 list2 list3 30
# Вернёт элемент из первого непустого списка

# Бесконечное ожидание
BLPOP mylist 0
```

### BLMOVE (Redis 6.2+)

Блокирующая версия LMOVE:

```redis
# Переместить элемент или ждать, если источник пуст
BLMOVE src dst LEFT RIGHT 30
```

### LMPOP (Redis 7.0+)

Извлечь элементы из первого непустого списка:

```redis
LMPOP 3 list1 list2 list3 LEFT COUNT 2
# Извлечь до 2 элементов слева из первого непустого списка
```

## Практические примеры использования

### Очередь задач (Queue)

```python
import redis
import json

r = redis.Redis(host='localhost', port=6379, db=0)

class TaskQueue:
    def __init__(self, name):
        self.queue_key = f"queue:{name}"

    def enqueue(self, task_data):
        """Добавить задачу в очередь."""
        task_json = json.dumps(task_data)
        r.rpush(self.queue_key, task_json)

    def dequeue(self, timeout=0):
        """Извлечь задачу из очереди (блокирующе)."""
        result = r.blpop(self.queue_key, timeout=timeout)
        if result:
            _, task_json = result
            return json.loads(task_json)
        return None

    def length(self):
        """Получить количество задач в очереди."""
        return r.llen(self.queue_key)

# Использование
queue = TaskQueue("emails")
queue.enqueue({"to": "user@example.com", "subject": "Hello"})

# В worker-процессе:
while True:
    task = queue.dequeue(timeout=30)
    if task:
        send_email(task)
```

### Стек (Stack — LIFO)

```python
class Stack:
    def __init__(self, name):
        self.key = f"stack:{name}"

    def push(self, value):
        r.lpush(self.key, value)

    def pop(self):
        return r.lpop(self.key)

    def peek(self):
        return r.lindex(self.key, 0)

    def size(self):
        return r.llen(self.key)
```

### Лента активности / История действий

```python
class ActivityFeed:
    def __init__(self, user_id, max_items=100):
        self.key = f"feed:{user_id}"
        self.max_items = max_items

    def add_activity(self, activity):
        """Добавить активность в ленту."""
        activity_json = json.dumps({
            **activity,
            "timestamp": time.time()
        })

        pipe = r.pipeline()
        pipe.lpush(self.key, activity_json)
        pipe.ltrim(self.key, 0, self.max_items - 1)  # Ограничить размер
        pipe.execute()

    def get_recent(self, count=20):
        """Получить последние N активностей."""
        items = r.lrange(self.key, 0, count - 1)
        return [json.loads(item) for item in items]

    def get_page(self, page=1, per_page=20):
        """Получить страницу активностей."""
        start = (page - 1) * per_page
        end = start + per_page - 1
        items = r.lrange(self.key, start, end)
        return [json.loads(item) for item in items]

# Использование
feed = ActivityFeed(user_id=123)
feed.add_activity({"action": "login", "ip": "192.168.1.1"})
feed.add_activity({"action": "post_created", "post_id": 456})

recent = feed.get_recent(10)
```

### Круговой буфер логов

```python
class CircularLogBuffer:
    def __init__(self, name, max_size=1000):
        self.key = f"logs:{name}"
        self.max_size = max_size

    def log(self, message):
        """Записать лог."""
        entry = json.dumps({
            "message": message,
            "timestamp": time.time()
        })

        pipe = r.pipeline()
        pipe.rpush(self.key, entry)
        pipe.ltrim(self.key, -self.max_size, -1)
        pipe.execute()

    def get_logs(self, count=100):
        """Получить последние логи."""
        items = r.lrange(self.key, -count, -1)
        return [json.loads(item) for item in items]
```

### Надёжная очередь с подтверждением

```python
class ReliableQueue:
    """Очередь с механизмом подтверждения обработки."""

    def __init__(self, name):
        self.main_queue = f"queue:{name}"
        self.processing_queue = f"queue:{name}:processing"

    def enqueue(self, task):
        r.rpush(self.main_queue, json.dumps(task))

    def dequeue(self, timeout=0):
        """Переместить задачу в processing и вернуть."""
        result = r.blmove(
            self.main_queue,
            self.processing_queue,
            timeout,
            "LEFT",
            "RIGHT"
        )
        if result:
            return json.loads(result)
        return None

    def complete(self, task):
        """Подтвердить успешную обработку."""
        r.lrem(self.processing_queue, 1, json.dumps(task))

    def requeue_failed(self):
        """Вернуть необработанные задачи в очередь."""
        while True:
            task = r.rpoplpush(self.processing_queue, self.main_queue)
            if not task:
                break
```

## Сложность операций

| Команда | Сложность |
|---------|-----------|
| LPUSH, RPUSH | O(N), где N — кол-во добавляемых элементов |
| LPOP, RPOP | O(N), где N — кол-во извлекаемых элементов |
| LLEN | O(1) |
| LINDEX | O(N), где N — позиция элемента |
| LRANGE | O(S+N), где S — offset, N — кол-во элементов |
| LSET | O(N) для доступа к середине списка |
| LINSERT | O(N), где N — позиция pivot-элемента |
| LREM | O(N), где N — длина списка |

## Лучшие практики

1. **Используйте LTRIM** для ограничения размера списков
2. **Предпочитайте операции с краями** — они O(1)
3. **Используйте блокирующие команды** для очередей вместо polling
4. **LRANGE 0 -1** для чтения всего списка, но будьте осторожны с большими списками
5. **Для больших списков** рассмотрите Streams вместо Lists
