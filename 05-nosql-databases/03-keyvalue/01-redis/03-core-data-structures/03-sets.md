# Sets (Множества) в Redis

Множества (Sets) в Redis — это неупорядоченные коллекции уникальных строк. Они идеально подходят для хранения уникальных элементов и выполнения операций теории множеств (объединение, пересечение, разность).

## Основные характеристики

- **Уникальность**: каждый элемент встречается только один раз
- **Неупорядоченность**: порядок элементов не гарантирован
- **Быстрый поиск**: проверка наличия элемента за O(1)
- **Операции над множествами**: union, intersection, difference

## Базовые команды

### SADD — добавление элементов

```redis
# Добавить один элемент
SADD myset "Hello"
# Результат: 1 (количество добавленных элементов)

# Добавить несколько элементов
SADD myset "World" "Redis" "Database"
# Результат: 3

# Попытка добавить существующий элемент
SADD myset "Hello"
# Результат: 0 (элемент уже существует)

SADD myset "New" "Hello" "Another"
# Результат: 2 (добавлено 2, "Hello" уже был)
```

### SREM — удаление элементов

```redis
SADD myset "one" "two" "three" "four"

# Удалить один элемент
SREM myset "one"
# Результат: 1

# Удалить несколько элементов
SREM myset "two" "three" "nonexistent"
# Результат: 2 (nonexistent не было в множестве)
```

### SMEMBERS — получить все элементы

```redis
SADD colors "red" "green" "blue"

SMEMBERS colors
# Результат: ["red", "green", "blue"] (порядок не гарантирован)
```

**Внимание**: SMEMBERS возвращает все элементы за O(N). Для больших множеств используйте SSCAN.

### SISMEMBER и SMISMEMBER — проверка наличия

```redis
SADD myset "one" "two" "three"

# Проверить один элемент
SISMEMBER myset "one"
# Результат: 1 (существует)

SISMEMBER myset "four"
# Результат: 0 (не существует)

# Проверить несколько элементов (Redis 6.2+)
SMISMEMBER myset "one" "four" "three"
# Результат: [1, 0, 1]
```

### SCARD — количество элементов

```redis
SADD myset "a" "b" "c"

SCARD myset
# Результат: 3

SCARD nonexistent
# Результат: 0
```

## Извлечение элементов

### SPOP — извлечь случайный элемент

```redis
SADD myset "one" "two" "three" "four"

# Извлечь 1 случайный элемент (удаляется из множества)
SPOP myset
# Результат: "three" (случайный)

# Извлечь несколько элементов
SPOP myset 2
# Результат: ["one", "four"]
```

### SRANDMEMBER — получить случайный элемент (без удаления)

```redis
SADD myset "one" "two" "three"

# Получить 1 случайный элемент
SRANDMEMBER myset
# Результат: "two"

# Получить несколько уникальных элементов
SRANDMEMBER myset 2
# Результат: ["one", "three"]

# Отрицательное число: элементы могут повторяться
SRANDMEMBER myset -5
# Результат: ["one", "two", "one", "three", "two"]
```

### SMOVE — переместить элемент между множествами

```redis
SADD set1 "a" "b" "c"
SADD set2 "x" "y"

SMOVE set1 set2 "c"
# Результат: 1

SMEMBERS set1
# Результат: ["a", "b"]

SMEMBERS set2
# Результат: ["x", "y", "c"]
```

## Операции над множествами

### SUNION — объединение

```redis
SADD set1 "a" "b" "c"
SADD set2 "c" "d" "e"

SUNION set1 set2
# Результат: ["a", "b", "c", "d", "e"]

# Сохранить результат в новое множество
SUNIONSTORE destset set1 set2
# Результат: 5 (количество элементов в destset)
```

### SINTER — пересечение

```redis
SADD set1 "a" "b" "c" "d"
SADD set2 "c" "d" "e" "f"

SINTER set1 set2
# Результат: ["c", "d"]

# Сохранить результат
SINTERSTORE destset set1 set2
# Результат: 2

# Пересечение нескольких множеств
SADD set3 "c" "x"
SINTER set1 set2 set3
# Результат: ["c"]

# SINTERCARD — только количество (Redis 7.0+)
SINTERCARD 2 set1 set2
# Результат: 2

# С лимитом для оптимизации
SINTERCARD 2 set1 set2 LIMIT 1
# Результат: 1 (прекратить подсчёт после 1)
```

### SDIFF — разность

```redis
SADD set1 "a" "b" "c" "d"
SADD set2 "c" "d" "e"

# Элементы, которые есть в set1, но нет в set2
SDIFF set1 set2
# Результат: ["a", "b"]

# Элементы, которые есть в set2, но нет в set1
SDIFF set2 set1
# Результат: ["e"]

# Сохранить результат
SDIFFSTORE destset set1 set2
# Результат: 2
```

## Сканирование множества

### SSCAN — итерация по элементам

Для больших множеств используйте SSCAN вместо SMEMBERS:

```redis
SADD myset "elem1" "elem2" "elem3" ... "elem1000"

# Начать сканирование
SSCAN myset 0
# Результат: ["cursor", ["elem1", "elem2", ...]]

# Продолжить с возвращённым курсором
SSCAN myset 176
# ...

# С паттерном
SSCAN myset 0 MATCH "user:*"

# С лимитом количества элементов за итерацию
SSCAN myset 0 COUNT 100
```

## Практические примеры использования

### Теги и категории

```python
import redis

r = redis.Redis(host='localhost', port=6379, db=0)

class TagSystem:
    def add_tags_to_item(self, item_id, tags):
        """Добавить теги к элементу."""
        key = f"item:{item_id}:tags"
        r.sadd(key, *tags)

        # Добавить item в индекс для каждого тега
        for tag in tags:
            r.sadd(f"tag:{tag}:items", item_id)

    def get_item_tags(self, item_id):
        """Получить все теги элемента."""
        return r.smembers(f"item:{item_id}:tags")

    def get_items_by_tag(self, tag):
        """Получить все элементы с тегом."""
        return r.smembers(f"tag:{tag}:items")

    def get_items_by_all_tags(self, tags):
        """Получить элементы, имеющие ВСЕ указанные теги."""
        keys = [f"tag:{tag}:items" for tag in tags]
        return r.sinter(*keys)

    def get_items_by_any_tags(self, tags):
        """Получить элементы, имеющие ЛЮБОЙ из указанных тегов."""
        keys = [f"tag:{tag}:items" for tag in tags]
        return r.sunion(*keys)

    def remove_tag_from_item(self, item_id, tag):
        """Удалить тег с элемента."""
        r.srem(f"item:{item_id}:tags", tag)
        r.srem(f"tag:{tag}:items", item_id)

# Использование
tags = TagSystem()
tags.add_tags_to_item("article:123", ["python", "redis", "database"])
tags.add_tags_to_item("article:124", ["python", "web"])
tags.add_tags_to_item("article:125", ["redis", "performance"])

# Все статьи с тегом python
python_articles = tags.get_items_by_tag("python")
# {"article:123", "article:124"}

# Статьи с python И redis
both = tags.get_items_by_all_tags(["python", "redis"])
# {"article:123"}
```

### Уникальные посетители

```python
from datetime import date

class UniqueVisitors:
    def record_visit(self, page_id, user_id):
        """Записать визит пользователя."""
        today = date.today().isoformat()
        key = f"visitors:{page_id}:{today}"
        r.sadd(key, user_id)

        # Установить TTL на 30 дней
        r.expire(key, 86400 * 30)

    def get_unique_visitors_count(self, page_id, date_str):
        """Получить количество уникальных посетителей."""
        key = f"visitors:{page_id}:{date_str}"
        return r.scard(key)

    def was_user_on_page(self, page_id, user_id, date_str):
        """Проверить, был ли пользователь на странице."""
        key = f"visitors:{page_id}:{date_str}"
        return r.sismember(key, user_id)

    def get_common_visitors(self, page1_id, page2_id, date_str):
        """Пользователи, посетившие обе страницы."""
        key1 = f"visitors:{page1_id}:{date_str}"
        key2 = f"visitors:{page2_id}:{date_str}"
        return r.sinter(key1, key2)

    def get_weekly_visitors(self, page_id, dates):
        """Уникальные посетители за несколько дней."""
        keys = [f"visitors:{page_id}:{d}" for d in dates]
        return r.sunion(*keys)

# Использование
visitors = UniqueVisitors()
visitors.record_visit("homepage", "user:123")
visitors.record_visit("homepage", "user:456")
visitors.record_visit("homepage", "user:123")  # Не добавится повторно

count = visitors.get_unique_visitors_count("homepage", "2024-01-15")
# 2
```

### Система друзей / подписок

```python
class SocialGraph:
    def follow(self, user_id, target_id):
        """Подписаться на пользователя."""
        r.sadd(f"user:{user_id}:following", target_id)
        r.sadd(f"user:{target_id}:followers", user_id)

    def unfollow(self, user_id, target_id):
        """Отписаться от пользователя."""
        r.srem(f"user:{user_id}:following", target_id)
        r.srem(f"user:{target_id}:followers", user_id)

    def get_followers(self, user_id):
        """Получить подписчиков."""
        return r.smembers(f"user:{user_id}:followers")

    def get_following(self, user_id):
        """Получить подписки."""
        return r.smembers(f"user:{user_id}:following")

    def is_following(self, user_id, target_id):
        """Проверить подписку."""
        return r.sismember(f"user:{user_id}:following", target_id)

    def get_mutual_friends(self, user1_id, user2_id):
        """Общие друзья (взаимные подписки)."""
        return r.sinter(
            f"user:{user1_id}:following",
            f"user:{user2_id}:following"
        )

    def get_followers_count(self, user_id):
        """Количество подписчиков."""
        return r.scard(f"user:{user_id}:followers")

    def get_recommendations(self, user_id):
        """Рекомендации: друзья друзей, на которых не подписан."""
        following = r.smembers(f"user:{user_id}:following")

        recommendations = set()
        for friend_id in following:
            friend_following = r.smembers(f"user:{friend_id}:following")
            recommendations.update(friend_following)

        # Исключить себя и тех, на кого уже подписан
        recommendations.discard(user_id)
        recommendations -= following

        return recommendations
```

### Онлайн пользователи

```python
import time

class OnlineUsers:
    def __init__(self, timeout=300):  # 5 минут
        self.timeout = timeout
        self.key = "online_users"

    def heartbeat(self, user_id):
        """Обновить статус онлайн."""
        r.sadd(self.key, user_id)
        r.set(f"user:{user_id}:last_seen", int(time.time()), ex=self.timeout)

    def is_online(self, user_id):
        """Проверить, онлайн ли пользователь."""
        return r.exists(f"user:{user_id}:last_seen")

    def get_online_count(self):
        """Приблизительное количество онлайн пользователей."""
        return r.scard(self.key)

    def cleanup(self):
        """Удалить офлайн пользователей из множества."""
        for user_id in r.sscan_iter(self.key):
            if not r.exists(f"user:{user_id}:last_seen"):
                r.srem(self.key, user_id)
```

### Лотерея / Случайный выбор

```python
class Lottery:
    def __init__(self, name):
        self.key = f"lottery:{name}"

    def enter(self, participant_id):
        """Зарегистрироваться в лотерее."""
        r.sadd(self.key, participant_id)

    def is_entered(self, participant_id):
        """Проверить регистрацию."""
        return r.sismember(self.key, participant_id)

    def pick_winner(self):
        """Выбрать победителя (без удаления)."""
        return r.srandmember(self.key)

    def pick_and_remove_winner(self):
        """Выбрать победителя и удалить его."""
        return r.spop(self.key)

    def pick_multiple_winners(self, count):
        """Выбрать несколько победителей."""
        return r.spop(self.key, count)

    def participants_count(self):
        """Количество участников."""
        return r.scard(self.key)
```

## Сложность операций

| Команда | Сложность |
|---------|-----------|
| SADD | O(N), где N — количество добавляемых элементов |
| SREM | O(N), где N — количество удаляемых элементов |
| SISMEMBER | O(1) |
| SMEMBERS | O(N), где N — размер множества |
| SCARD | O(1) |
| SPOP, SRANDMEMBER | O(N), где N — количество возвращаемых элементов |
| SUNION, SINTER, SDIFF | O(N), где N — общее количество элементов |

## Лучшие практики

1. **Используйте SSCAN** для больших множеств вместо SMEMBERS
2. **SINTERCARD** для подсчёта пересечения без получения элементов
3. **Сохраняйте результаты** операций над множествами (SUNIONSTORE) для повторного использования
4. **Индексируйте обратные связи** (например, tags->items и items->tags)
5. **Устанавливайте TTL** для временных множеств (ежедневные счётчики и т.п.)
