# Кейсы и примеры системного дизайна (Case Studies)

[prev: 28-security-basics](./28-security-basics.md) | [next: 01-migration-strategies](../22-building-for-scale/01-migration-strategies.md)

---

## Введение

Системный дизайн — это искусство проектирования масштабируемых, надёжных и эффективных систем. В этом разделе разберём несколько классических задач системного дизайна, которые часто встречаются на собеседованиях и в реальной практике.

Каждый кейс следует структуре:
1. **Требования** — что нужно построить
2. **Оценки масштаба** — расчёт нагрузки
3. **Высокоуровневый дизайн** — основные компоненты
4. **Детальный дизайн** — углубление в ключевые аспекты
5. **Масштабирование** — как система будет расти

---

## 1. URL Shortener (Сокращатель ссылок)

### Требования

**Функциональные:**
- Создание короткой ссылки из длинного URL
- Редирект на оригинальный URL по короткой ссылке
- Опционально: кастомные алиасы, TTL, аналитика

**Нефункциональные:**
- Высокая доступность (99.99%)
- Низкая latency (<100ms)
- Короткие URL должны быть уникальными
- Невозможность предугадать следующий URL

### Оценки масштаба

```
Допущения:
- 100M новых URL в месяц (write)
- 10B редиректов в месяц (read)
- Read:Write ratio = 100:1
- Хранение 5 лет

Расчёты:
Write QPS: 100M / (30 * 24 * 3600) ≈ 40 URL/sec
Read QPS: 10B / (30 * 24 * 3600) ≈ 4000 redirects/sec

Хранение за 5 лет:
100M * 12 * 5 = 6B URL
Средний размер записи: 500 bytes (URL + metadata)
Общий объём: 6B * 500 = 3TB
```

### Высокоуровневый дизайн

```
┌─────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Client    │────►│  Load Balancer  │────►│   API Servers   │
└─────────────┘     └─────────────────┘     └────────┬────────┘
                                                     │
                    ┌────────────────────────────────┼────────────────────────────────┐
                    │                                │                                │
                    ▼                                ▼                                ▼
            ┌───────────────┐              ┌─────────────────┐              ┌─────────────────┐
            │     Cache     │              │    Database     │              │  ID Generator   │
            │    (Redis)    │              │   (Key-Value)   │              │    Service      │
            └───────────────┘              └─────────────────┘              └─────────────────┘
```

### Генерация коротких URL

```python
"""
Варианты генерации коротких ключей:

1. Base62 encoding (a-z, A-Z, 0-9)
   - 6 символов: 62^6 ≈ 56.8B комбинаций
   - 7 символов: 62^7 ≈ 3.5T комбинаций

2. MD5/SHA хеш (первые N символов)
   - Коллизии возможны

3. Counter-based (распределённый счётчик)
   - Предсказуемый порядок
   - Требует координации
"""

import hashlib
import time
from typing import Optional
import redis


# Вариант 1: Base62 encoding с уникальным ID
class Base62Encoder:
    ALPHABET = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"

    @staticmethod
    def encode(num: int) -> str:
        """Кодирование числа в base62."""
        if num == 0:
            return Base62Encoder.ALPHABET[0]

        result = []
        while num:
            result.append(Base62Encoder.ALPHABET[num % 62])
            num //= 62

        return ''.join(reversed(result))

    @staticmethod
    def decode(s: str) -> int:
        """Декодирование base62 в число."""
        num = 0
        for char in s:
            num = num * 62 + Base62Encoder.ALPHABET.index(char)
        return num


# Вариант 2: Distributed ID Generator (Snowflake-like)
class DistributedIdGenerator:
    """
    64-bit ID:
    - 1 bit: sign (always 0)
    - 41 bits: timestamp (69 years)
    - 10 bits: machine ID (1024 machines)
    - 12 bits: sequence (4096 per ms)
    """
    EPOCH = 1609459200000  # 2021-01-01 00:00:00 UTC

    def __init__(self, machine_id: int):
        if machine_id < 0 or machine_id > 1023:
            raise ValueError("Machine ID must be 0-1023")

        self.machine_id = machine_id
        self.sequence = 0
        self.last_timestamp = -1

    def generate(self) -> int:
        timestamp = int(time.time() * 1000)

        if timestamp == self.last_timestamp:
            self.sequence = (self.sequence + 1) & 0xFFF
            if self.sequence == 0:
                # Ждём следующую миллисекунду
                while timestamp <= self.last_timestamp:
                    timestamp = int(time.time() * 1000)
        else:
            self.sequence = 0

        self.last_timestamp = timestamp

        id = ((timestamp - self.EPOCH) << 22) | (self.machine_id << 12) | self.sequence
        return id


class URLShortener:
    def __init__(self, redis_client: redis.Redis, id_generator: DistributedIdGenerator):
        self.redis = redis_client
        self.id_gen = id_generator
        self.encoder = Base62Encoder()

    def shorten(self, long_url: str, custom_alias: Optional[str] = None, ttl: Optional[int] = None) -> str:
        """Создание короткой ссылки."""
        # Проверяем, не сокращали ли уже этот URL
        existing = self.redis.get(f"url:long:{long_url}")
        if existing:
            return existing.decode()

        if custom_alias:
            # Проверяем доступность кастомного алиаса
            if self.redis.exists(f"url:short:{custom_alias}"):
                raise ValueError("Alias already taken")
            short_key = custom_alias
        else:
            # Генерируем уникальный ключ
            unique_id = self.id_gen.generate()
            short_key = self.encoder.encode(unique_id)

        # Сохраняем маппинги
        pipe = self.redis.pipeline()
        pipe.set(f"url:short:{short_key}", long_url)
        pipe.set(f"url:long:{long_url}", short_key)

        if ttl:
            pipe.expire(f"url:short:{short_key}", ttl)
            pipe.expire(f"url:long:{long_url}", ttl)

        pipe.execute()

        return short_key

    def resolve(self, short_key: str) -> Optional[str]:
        """Получение оригинального URL."""
        long_url = self.redis.get(f"url:short:{short_key}")
        if long_url:
            # Инкрементируем счётчик кликов
            self.redis.incr(f"stats:clicks:{short_key}")
            return long_url.decode()
        return None


# API endpoints
from fastapi import FastAPI, HTTPException, Response
from pydantic import BaseModel, HttpUrl

app = FastAPI()


class ShortenRequest(BaseModel):
    url: HttpUrl
    custom_alias: Optional[str] = None
    ttl: Optional[int] = None


@app.post("/shorten")
async def shorten_url(request: ShortenRequest):
    try:
        short_key = shortener.shorten(
            str(request.url),
            request.custom_alias,
            request.ttl
        )
        return {
            "short_url": f"https://short.url/{short_key}",
            "original_url": str(request.url)
        }
    except ValueError as e:
        raise HTTPException(400, str(e))


@app.get("/{short_key}")
async def redirect_to_original(short_key: str):
    long_url = shortener.resolve(short_key)
    if not long_url:
        raise HTTPException(404, "URL not found")

    return Response(
        status_code=301,
        headers={"Location": long_url}
    )
```

### Масштабирование

```
1. Database Sharding
   - Sharding key: short_key (consistent hashing)
   - Каждый shard хранит диапазон ключей

2. Caching Layer
   - Redis cluster для горячих URL
   - LRU eviction policy
   - Cache-aside pattern

3. Rate Limiting
   - По IP для анонимных пользователей
   - По API key для зарегистрированных

4. Analytics Pipeline
   - Async запись кликов в Kafka
   - Batch processing для агрегации
```

---

## 2. Twitter Feed (Лента новостей)

### Требования

**Функциональные:**
- Публикация твитов (текст, медиа)
- Подписка/отписка на пользователей
- Просмотр ленты новостей (timeline)
- Лайки, ретвиты, комментарии

**Нефункциональные:**
- Timeline должен загружаться < 200ms
- Eventual consistency для ленты (допустима задержка до нескольких секунд)
- Поддержка "знаменитостей" (миллионы подписчиков)

### Оценки масштаба

```
Допущения:
- 500M пользователей
- 100M DAU (Daily Active Users)
- Среднее количество подписок: 200
- 500M твитов в день
- Средний размер твита: 500 bytes

Расчёты:
Tweet writes: 500M / 86400 ≈ 5,800 tweets/sec
Timeline reads: 100M * 10 reads/day = 1B reads/day ≈ 11,500 reads/sec

Fanout при публикации (push model):
- Обычный пользователь: 200 подписчиков → 200 writes
- Знаменитость: 10M подписчиков → 10M writes (слишком много!)
```

### Высокоуровневый дизайн

```
┌───────────────────────────────────────────────────────────────────────────┐
│                              Twitter Architecture                          │
├───────────────────────────────────────────────────────────────────────────┤
│                                                                            │
│  ┌─────────┐        ┌──────────────┐        ┌─────────────────────────┐   │
│  │ Client  │───────►│ API Gateway  │───────►│      Tweet Service       │   │
│  └─────────┘        └──────────────┘        │  (write tweets)          │   │
│                            │                 └────────────┬────────────┘   │
│                            │                              │                │
│                            ▼                              ▼                │
│                    ┌──────────────┐        ┌─────────────────────────┐   │
│                    │   Timeline    │        │     Tweet Database       │   │
│                    │    Service    │        │      (Cassandra)        │   │
│                    └──────┬───────┘        └─────────────────────────┘   │
│                           │                              │                │
│                           ▼                              │                │
│                    ┌──────────────┐                      │                │
│                    │   Timeline    │◄─────────────────────┘                │
│                    │    Cache      │        Fanout Service                │
│                    │   (Redis)     │        (async)                       │
│                    └──────────────┘                                       │
│                                                                            │
│  ┌─────────────────────────────────────────────────────────────────────┐ │
│  │                        Fanout Strategies                              │ │
│  ├─────────────────────────────────────────────────────────────────────┤ │
│  │                                                                       │ │
│  │  Push Model (Pre-computed timeline):                                 │ │
│  │  ┌────────┐     ┌───────────────┐                                    │ │
│  │  │ Tweet  │────►│ Fanout Worker │────► Write to all followers' cache │ │
│  │  └────────┘     └───────────────┘                                    │ │
│  │                                                                       │ │
│  │  Pull Model (On-demand):                                             │ │
│  │  ┌─────────────┐     ┌───────────────┐                               │ │
│  │  │ Get Timeline │────►│ Fetch tweets │────► Merge & Sort             │ │
│  │  └─────────────┘     │ from followees│                               │ │
│  │                       └───────────────┘                               │ │
│  │                                                                       │ │
│  │  Hybrid Model (Twitter's approach):                                  │ │
│  │  - Push for regular users (< 10K followers)                          │ │
│  │  - Pull for celebrities (> 10K followers)                            │ │
│  │  - Merge at read time                                                │ │
│  │                                                                       │ │
│  └─────────────────────────────────────────────────────────────────────┘ │
│                                                                            │
└───────────────────────────────────────────────────────────────────────────┘
```

### Реализация

```python
from dataclasses import dataclass
from datetime import datetime
from typing import List, Optional
import asyncio
import redis.asyncio as redis
from cassandra.cluster import Cluster
import json


@dataclass
class Tweet:
    id: str
    user_id: str
    content: str
    created_at: datetime
    media_urls: List[str] = None


@dataclass
class User:
    id: str
    username: str
    followers_count: int
    is_celebrity: bool = False  # > 10K followers


class TweetService:
    def __init__(self, cassandra_session, redis_client: redis.Redis, kafka_producer):
        self.db = cassandra_session
        self.redis = redis_client
        self.kafka = kafka_producer
        self.CELEBRITY_THRESHOLD = 10_000

    async def post_tweet(self, user_id: str, content: str, media_urls: List[str] = None) -> Tweet:
        """Публикация твита."""
        tweet_id = generate_snowflake_id()
        created_at = datetime.utcnow()

        tweet = Tweet(
            id=tweet_id,
            user_id=user_id,
            content=content,
            created_at=created_at,
            media_urls=media_urls or []
        )

        # Сохраняем в Cassandra
        await self._save_tweet(tweet)

        # Отправляем на fanout
        user = await self._get_user(user_id)
        if user.followers_count < self.CELEBRITY_THRESHOLD:
            # Push model для обычных пользователей
            await self._fanout_tweet(tweet)
        # Для знаменитостей используем pull model — ничего не делаем

        return tweet

    async def _save_tweet(self, tweet: Tweet):
        """Сохранение твита в Cassandra."""
        query = """
            INSERT INTO tweets (id, user_id, content, created_at, media_urls)
            VALUES (?, ?, ?, ?, ?)
        """
        await self.db.execute_async(
            query,
            (tweet.id, tweet.user_id, tweet.content, tweet.created_at, tweet.media_urls)
        )

        # Также сохраняем в ленту пользователя (его собственные твиты)
        query_timeline = """
            INSERT INTO user_tweets (user_id, tweet_id, created_at)
            VALUES (?, ?, ?)
        """
        await self.db.execute_async(
            query_timeline,
            (tweet.user_id, tweet.id, tweet.created_at)
        )

    async def _fanout_tweet(self, tweet: Tweet):
        """Fanout твита подписчикам (push model)."""
        # Отправляем в Kafka для асинхронной обработки
        event = {
            "type": "tweet_created",
            "tweet_id": tweet.id,
            "user_id": tweet.user_id,
            "created_at": tweet.created_at.isoformat()
        }
        await self.kafka.send("fanout-events", json.dumps(event).encode())


class FanoutWorker:
    """Воркер для обработки fanout событий."""

    def __init__(self, cassandra_session, redis_client: redis.Redis):
        self.db = cassandra_session
        self.redis = redis_client
        self.BATCH_SIZE = 1000
        self.TIMELINE_SIZE = 800  # Максимум твитов в кеше timeline

    async def process_fanout(self, tweet_id: str, author_id: str):
        """Добавление твита в timeline всех подписчиков."""
        followers = await self._get_followers(author_id)

        # Обрабатываем батчами
        for i in range(0, len(followers), self.BATCH_SIZE):
            batch = followers[i:i + self.BATCH_SIZE]
            await self._update_timelines(batch, tweet_id)

    async def _get_followers(self, user_id: str) -> List[str]:
        """Получение списка подписчиков."""
        query = "SELECT follower_id FROM followers WHERE user_id = ?"
        result = await self.db.execute_async(query, (user_id,))
        return [row.follower_id for row in result]

    async def _update_timelines(self, follower_ids: List[str], tweet_id: str):
        """Обновление timeline для списка пользователей."""
        pipe = self.redis.pipeline()

        for follower_id in follower_ids:
            timeline_key = f"timeline:{follower_id}"
            # ZADD с timestamp как score для сортировки
            pipe.zadd(timeline_key, {tweet_id: time.time()})
            # Обрезаем до максимального размера
            pipe.zremrangebyrank(timeline_key, 0, -self.TIMELINE_SIZE - 1)

        await pipe.execute()


class TimelineService:
    def __init__(self, cassandra_session, redis_client: redis.Redis):
        self.db = cassandra_session
        self.redis = redis_client
        self.CELEBRITY_THRESHOLD = 10_000

    async def get_timeline(self, user_id: str, limit: int = 20, offset: int = 0) -> List[Tweet]:
        """Получение timeline пользователя."""
        # 1. Получаем твиты из кеша (pre-computed timeline)
        cached_tweet_ids = await self._get_cached_timeline(user_id, limit + 50, offset)

        # 2. Получаем твиты от знаменитостей (pull model)
        celebrity_followees = await self._get_celebrity_followees(user_id)
        celebrity_tweets = await self._get_recent_tweets(celebrity_followees, limit)

        # 3. Объединяем и сортируем
        all_tweet_ids = set(cached_tweet_ids)
        all_tweet_ids.update(t.id for t in celebrity_tweets)

        # 4. Получаем полные данные твитов
        tweets = await self._get_tweets_by_ids(list(all_tweet_ids))

        # 5. Сортируем по времени и обрезаем
        tweets.sort(key=lambda t: t.created_at, reverse=True)
        return tweets[offset:offset + limit]

    async def _get_cached_timeline(self, user_id: str, limit: int, offset: int) -> List[str]:
        """Получение tweet IDs из Redis."""
        timeline_key = f"timeline:{user_id}"
        tweet_ids = await self.redis.zrevrange(timeline_key, offset, offset + limit - 1)
        return [tid.decode() for tid in tweet_ids]

    async def _get_celebrity_followees(self, user_id: str) -> List[str]:
        """Получение знаменитостей, на которых подписан пользователь."""
        query = """
            SELECT followee_id FROM following
            WHERE user_id = ? AND followee_is_celebrity = true
        """
        result = await self.db.execute_async(query, (user_id,))
        return [row.followee_id for row in result]

    async def _get_recent_tweets(self, user_ids: List[str], limit: int) -> List[Tweet]:
        """Получение недавних твитов от списка пользователей."""
        if not user_ids:
            return []

        query = """
            SELECT * FROM user_tweets
            WHERE user_id IN ?
            ORDER BY created_at DESC
            LIMIT ?
        """
        result = await self.db.execute_async(query, (user_ids, limit))
        return [self._row_to_tweet(row) for row in result]

    async def _get_tweets_by_ids(self, tweet_ids: List[str]) -> List[Tweet]:
        """Получение твитов по ID."""
        if not tweet_ids:
            return []

        # Сначала проверяем кеш
        cached = await self._get_tweets_from_cache(tweet_ids)
        missing_ids = [tid for tid in tweet_ids if tid not in cached]

        # Получаем отсутствующие из БД
        if missing_ids:
            from_db = await self._get_tweets_from_db(missing_ids)
            # Кешируем
            await self._cache_tweets(from_db)
            cached.update(from_db)

        return list(cached.values())
```

### Data Model (Cassandra)

```cql
-- Твиты
CREATE TABLE tweets (
    id TEXT,
    user_id TEXT,
    content TEXT,
    created_at TIMESTAMP,
    media_urls LIST<TEXT>,
    likes_count COUNTER,
    retweets_count COUNTER,
    PRIMARY KEY (id)
);

-- Твиты пользователя (для его профиля)
CREATE TABLE user_tweets (
    user_id TEXT,
    tweet_id TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, created_at, tweet_id)
) WITH CLUSTERING ORDER BY (created_at DESC);

-- Подписчики
CREATE TABLE followers (
    user_id TEXT,
    follower_id TEXT,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, follower_id)
);

-- Подписки
CREATE TABLE following (
    user_id TEXT,
    followee_id TEXT,
    followee_is_celebrity BOOLEAN,
    created_at TIMESTAMP,
    PRIMARY KEY (user_id, followee_id)
);
```

---

## 3. WhatsApp (Мессенджер)

### Требования

**Функциональные:**
- 1-to-1 чаты
- Групповые чаты (до 256 участников)
- Отправка текста, медиа, документов
- Статусы доставки (sent, delivered, read)
- Push notifications

**Нефункциональные:**
- Доставка сообщений < 500ms
- Гарантированная доставка (at-least-once)
- End-to-end шифрование
- Поддержка офлайн-режима

### Оценки масштаба

```
Допущения:
- 2B пользователей
- 500M DAU
- 100 сообщений на пользователя в день
- Средний размер сообщения: 100 bytes

Расчёты:
Сообщений в день: 500M * 100 = 50B
Messages per second: 50B / 86400 ≈ 580K msg/sec

Хранение за день:
50B * 100 bytes = 5TB/day

Connections:
500M одновременных WebSocket соединений
```

### Высокоуровневый дизайн

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                          WhatsApp Architecture                               │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌──────────┐                                                               │
│  │  Mobile  │                                                               │
│  │   App    │                                                               │
│  └────┬─────┘                                                               │
│       │                                                                      │
│       │ WebSocket                                                           │
│       ▼                                                                      │
│  ┌────────────────┐        ┌─────────────────────────────────────────────┐ │
│  │  Connection    │        │              Message Flow                    │ │
│  │    Gateway     │───────►│                                              │ │
│  │  (WebSocket)   │        │  1. Client A sends message                  │ │
│  └────────────────┘        │  2. Gateway routes to Message Service       │ │
│       │                     │  3. Message stored in Cassandra             │ │
│       │                     │  4. If Client B online → deliver via WS     │ │
│       ▼                     │  5. If Client B offline → push notification │ │
│  ┌────────────────┐        │  6. Client B comes online → sync messages   │ │
│  │   Session      │        │                                              │ │
│  │    Store       │        └─────────────────────────────────────────────┘ │
│  │   (Redis)      │                                                        │
│  └────────────────┘                                                        │
│       │                                                                      │
│       ▼                                                                      │
│  ┌────────────────────────────────────────────────────────┐                 │
│  │                    Message Service                      │                 │
│  │  ┌──────────────┐  ┌──────────────┐  ┌──────────────┐  │                 │
│  │  │   Routing    │  │   Storage    │  │  Delivery    │  │                 │
│  │  │   Manager    │  │   Manager    │  │   Manager    │  │                 │
│  │  └──────────────┘  └──────────────┘  └──────────────┘  │                 │
│  └────────────────────────────────────────────────────────┘                 │
│       │                    │                    │                           │
│       ▼                    ▼                    ▼                           │
│  ┌──────────┐        ┌──────────┐        ┌──────────────┐                   │
│  │  Group   │        │ Message  │        │    Push      │                   │
│  │ Service  │        │   DB     │        │ Notification │                   │
│  │          │        │(Cassandra)│        │   Service    │                   │
│  └──────────┘        └──────────┘        └──────────────┘                   │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Реализация

```python
import asyncio
from dataclasses import dataclass, field
from datetime import datetime
from enum import Enum
from typing import Dict, List, Optional, Set
import json
import websockets
import redis.asyncio as redis


class MessageStatus(Enum):
    SENT = "sent"           # Доставлено на сервер
    DELIVERED = "delivered" # Доставлено получателю
    READ = "read"           # Прочитано


@dataclass
class Message:
    id: str
    chat_id: str
    sender_id: str
    content: str
    message_type: str  # text, image, video, etc.
    created_at: datetime
    status: MessageStatus = MessageStatus.SENT


@dataclass
class Chat:
    id: str
    participants: List[str]
    is_group: bool = False
    group_name: Optional[str] = None
    last_message_at: Optional[datetime] = None


class ConnectionManager:
    """Управление WebSocket соединениями."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        self.local_connections: Dict[str, websockets.WebSocketServerProtocol] = {}
        self.server_id = generate_server_id()

    async def connect(self, user_id: str, websocket: websockets.WebSocketServerProtocol):
        """Регистрация нового соединения."""
        self.local_connections[user_id] = websocket

        # Регистрируем в Redis для cross-server маршрутизации
        await self.redis.hset(
            "user_connections",
            user_id,
            json.dumps({"server_id": self.server_id, "connected_at": datetime.utcnow().isoformat()})
        )

        # Подписываемся на сообщения для этого пользователя
        await self._subscribe_to_messages(user_id)

    async def disconnect(self, user_id: str):
        """Удаление соединения."""
        if user_id in self.local_connections:
            del self.local_connections[user_id]

        await self.redis.hdel("user_connections", user_id)

    async def send_to_user(self, user_id: str, message: dict):
        """Отправка сообщения пользователю."""
        # Проверяем локальное соединение
        if user_id in self.local_connections:
            ws = self.local_connections[user_id]
            await ws.send(json.dumps(message))
            return True

        # Проверяем, подключен ли на другом сервере
        connection_info = await self.redis.hget("user_connections", user_id)
        if connection_info:
            info = json.loads(connection_info)
            # Публикуем в Redis pub/sub для другого сервера
            await self.redis.publish(
                f"user_messages:{info['server_id']}",
                json.dumps({"user_id": user_id, "message": message})
            )
            return True

        return False  # Пользователь офлайн

    async def _subscribe_to_messages(self, user_id: str):
        """Подписка на входящие сообщения через Redis pub/sub."""
        pubsub = self.redis.pubsub()
        await pubsub.subscribe(f"user_messages:{self.server_id}")

        async for message in pubsub.listen():
            if message["type"] == "message":
                data = json.loads(message["data"])
                target_user = data["user_id"]
                if target_user in self.local_connections:
                    ws = self.local_connections[target_user]
                    await ws.send(json.dumps(data["message"]))


class MessageService:
    def __init__(
        self,
        cassandra_session,
        redis_client: redis.Redis,
        connection_manager: ConnectionManager,
        push_service
    ):
        self.db = cassandra_session
        self.redis = redis_client
        self.connections = connection_manager
        self.push = push_service

    async def send_message(
        self,
        sender_id: str,
        chat_id: str,
        content: str,
        message_type: str = "text"
    ) -> Message:
        """Отправка сообщения."""
        message = Message(
            id=generate_message_id(),
            chat_id=chat_id,
            sender_id=sender_id,
            content=content,
            message_type=message_type,
            created_at=datetime.utcnow()
        )

        # 1. Сохраняем сообщение
        await self._save_message(message)

        # 2. Получаем участников чата
        chat = await self._get_chat(chat_id)
        recipients = [uid for uid in chat.participants if uid != sender_id]

        # 3. Доставляем каждому получателю
        delivery_tasks = [
            self._deliver_to_user(message, recipient_id)
            for recipient_id in recipients
        ]
        await asyncio.gather(*delivery_tasks)

        return message

    async def _save_message(self, message: Message):
        """Сохранение сообщения в Cassandra."""
        query = """
            INSERT INTO messages (id, chat_id, sender_id, content, message_type, created_at, status)
            VALUES (?, ?, ?, ?, ?, ?, ?)
        """
        await self.db.execute_async(
            query,
            (message.id, message.chat_id, message.sender_id,
             message.content, message.message_type,
             message.created_at, message.status.value)
        )

    async def _deliver_to_user(self, message: Message, user_id: str):
        """Доставка сообщения конкретному пользователю."""
        message_data = {
            "type": "new_message",
            "message": {
                "id": message.id,
                "chat_id": message.chat_id,
                "sender_id": message.sender_id,
                "content": message.content,
                "created_at": message.created_at.isoformat()
            }
        }

        # Пробуем доставить через WebSocket
        delivered = await self.connections.send_to_user(user_id, message_data)

        if delivered:
            await self._update_message_status(message.id, MessageStatus.DELIVERED)
        else:
            # Пользователь офлайн — сохраняем для синхронизации
            await self._save_pending_message(user_id, message.id)
            # Отправляем push notification
            await self.push.send_notification(
                user_id,
                title="New message",
                body=f"{message.sender_id}: {message.content[:50]}..."
            )

    async def sync_messages(self, user_id: str, last_synced_at: datetime) -> List[Message]:
        """Синхронизация сообщений при подключении."""
        # Получаем чаты пользователя
        chats = await self._get_user_chats(user_id)
        chat_ids = [chat.id for chat in chats]

        # Получаем новые сообщения
        query = """
            SELECT * FROM messages
            WHERE chat_id IN ?
            AND created_at > ?
            ORDER BY created_at ASC
        """
        result = await self.db.execute_async(query, (chat_ids, last_synced_at))

        messages = [self._row_to_message(row) for row in result]

        # Обновляем статусы на DELIVERED
        for msg in messages:
            if msg.sender_id != user_id:
                await self._update_message_status(msg.id, MessageStatus.DELIVERED)
                # Отправляем статус отправителю
                await self._send_status_update(msg.sender_id, msg.id, MessageStatus.DELIVERED)

        return messages

    async def mark_as_read(self, user_id: str, message_ids: List[str]):
        """Отметка сообщений как прочитанных."""
        for message_id in message_ids:
            message = await self._get_message(message_id)
            if message.sender_id != user_id:
                await self._update_message_status(message_id, MessageStatus.READ)
                await self._send_status_update(message.sender_id, message_id, MessageStatus.READ)

    async def _send_status_update(self, user_id: str, message_id: str, status: MessageStatus):
        """Отправка обновления статуса отправителю."""
        update = {
            "type": "status_update",
            "message_id": message_id,
            "status": status.value
        }
        await self.connections.send_to_user(user_id, update)


class GroupService:
    """Сервис для групповых чатов."""

    def __init__(self, cassandra_session, redis_client: redis.Redis):
        self.db = cassandra_session
        self.redis = redis_client
        self.MAX_GROUP_SIZE = 256

    async def create_group(
        self,
        creator_id: str,
        name: str,
        participant_ids: List[str]
    ) -> Chat:
        """Создание группы."""
        if len(participant_ids) > self.MAX_GROUP_SIZE:
            raise ValueError(f"Group cannot exceed {self.MAX_GROUP_SIZE} members")

        all_participants = list(set([creator_id] + participant_ids))

        chat = Chat(
            id=generate_chat_id(),
            participants=all_participants,
            is_group=True,
            group_name=name,
            last_message_at=datetime.utcnow()
        )

        await self._save_chat(chat)

        # Добавляем записи для каждого участника
        for user_id in all_participants:
            await self._add_user_chat(user_id, chat.id)

        return chat

    async def add_participant(self, group_id: str, user_id: str, added_by: str):
        """Добавление участника в группу."""
        chat = await self._get_chat(group_id)

        if not chat.is_group:
            raise ValueError("Cannot add participants to 1-to-1 chat")

        if len(chat.participants) >= self.MAX_GROUP_SIZE:
            raise ValueError("Group is full")

        if user_id in chat.participants:
            raise ValueError("User already in group")

        chat.participants.append(user_id)
        await self._update_chat(chat)
        await self._add_user_chat(user_id, group_id)

        # Отправляем системное сообщение
        await self._send_system_message(
            group_id,
            f"{added_by} added {user_id} to the group"
        )
```

### End-to-End Encryption

```python
"""
Упрощённая схема E2E шифрования (Signal Protocol):

1. Каждый клиент генерирует:
   - Identity Key Pair (долгосрочная идентификация)
   - Signed Pre Key (обновляется периодически)
   - One-Time Pre Keys (одноразовые ключи)

2. При первом сообщении:
   - Отправитель получает публичные ключи получателя с сервера
   - Выполняется Extended Triple Diffie-Hellman (X3DH)
   - Генерируется shared secret

3. Для каждого сообщения:
   - Используется Double Ratchet Algorithm
   - Обеспечивает forward secrecy
"""

from cryptography.hazmat.primitives.asymmetric import x25519
from cryptography.hazmat.primitives import hashes
from cryptography.hazmat.primitives.kdf.hkdf import HKDF
from cryptography.hazmat.primitives.ciphers.aead import AESGCM
import os


class KeyBundle:
    """Набор публичных ключей пользователя."""
    def __init__(self, identity_key: bytes, signed_prekey: bytes, one_time_prekey: bytes):
        self.identity_key = identity_key
        self.signed_prekey = signed_prekey
        self.one_time_prekey = one_time_prekey


class E2EEncryption:
    def __init__(self):
        # Identity Key Pair (долгосрочный)
        self.identity_private = x25519.X25519PrivateKey.generate()
        self.identity_public = self.identity_private.public_key()

        # Signed Pre Key (обновляется раз в неделю)
        self.signed_prekey_private = x25519.X25519PrivateKey.generate()
        self.signed_prekey_public = self.signed_prekey_private.public_key()

        # One-Time Pre Keys (пул одноразовых ключей)
        self.one_time_prekeys = self._generate_prekeys(100)

    def _generate_prekeys(self, count: int) -> dict:
        keys = {}
        for i in range(count):
            private = x25519.X25519PrivateKey.generate()
            keys[i] = {"private": private, "public": private.public_key()}
        return keys

    def get_public_bundle(self) -> KeyBundle:
        """Публичные ключи для отправки на сервер."""
        # Выдаём один one-time prekey
        otpk_id, otpk = self.one_time_prekeys.popitem()
        return KeyBundle(
            identity_key=self.identity_public.public_bytes_raw(),
            signed_prekey=self.signed_prekey_public.public_bytes_raw(),
            one_time_prekey=otpk["public"].public_bytes_raw()
        )

    def establish_session(self, recipient_bundle: KeyBundle) -> bytes:
        """X3DH Key Agreement для установки сессии."""
        # Загружаем публичные ключи получателя
        ik_recipient = x25519.X25519PublicKey.from_public_bytes(recipient_bundle.identity_key)
        spk_recipient = x25519.X25519PublicKey.from_public_bytes(recipient_bundle.signed_prekey)
        opk_recipient = x25519.X25519PublicKey.from_public_bytes(recipient_bundle.one_time_prekey)

        # Ephemeral key для этой сессии
        ephemeral_private = x25519.X25519PrivateKey.generate()

        # X3DH: 4 DH операции
        dh1 = self.identity_private.exchange(spk_recipient)
        dh2 = ephemeral_private.exchange(ik_recipient)
        dh3 = ephemeral_private.exchange(spk_recipient)
        dh4 = ephemeral_private.exchange(opk_recipient)

        # Объединяем результаты
        shared_secret = dh1 + dh2 + dh3 + dh4

        # Derive keys
        return HKDF(
            algorithm=hashes.SHA256(),
            length=32,
            salt=None,
            info=b"WhatsApp E2E"
        ).derive(shared_secret)

    def encrypt_message(self, session_key: bytes, plaintext: str) -> bytes:
        """Шифрование сообщения."""
        aesgcm = AESGCM(session_key)
        nonce = os.urandom(12)
        ciphertext = aesgcm.encrypt(nonce, plaintext.encode(), None)
        return nonce + ciphertext

    def decrypt_message(self, session_key: bytes, ciphertext: bytes) -> str:
        """Расшифровка сообщения."""
        aesgcm = AESGCM(session_key)
        nonce = ciphertext[:12]
        encrypted = ciphertext[12:]
        plaintext = aesgcm.decrypt(nonce, encrypted, None)
        return plaintext.decode()
```

---

## 4. Uber (Сервис такси)

### Требования

**Функциональные:**
- Запрос поездки
- Matching водитель-пассажир
- Отслеживание в реальном времени
- Расчёт стоимости и оплата
- История поездок

**Нефункциональные:**
- Matching < 30 секунд
- Обновление локации < 1 секунда
- Высокая доступность (99.99%)
- Масштабирование на миллионы пользователей

### Высокоуровневый дизайн

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Uber Architecture                                 │
├─────────────────────────────────────────────────────────────────────────────┤
│                                                                              │
│  ┌───────────────┐                              ┌───────────────┐           │
│  │  Rider App    │                              │  Driver App   │           │
│  └───────┬───────┘                              └───────┬───────┘           │
│          │                                              │                    │
│          │              ┌──────────────────┐           │                    │
│          └─────────────►│   API Gateway    │◄──────────┘                    │
│                         └────────┬─────────┘                                │
│                                  │                                          │
│          ┌───────────────────────┼───────────────────────┐                 │
│          │                       │                       │                  │
│          ▼                       ▼                       ▼                  │
│  ┌───────────────┐     ┌───────────────┐     ┌───────────────┐             │
│  │   Trip        │     │   Matching    │     │   Location    │             │
│  │   Service     │     │   Service     │     │   Service     │             │
│  └───────────────┘     └───────────────┘     └───────────────┘             │
│          │                    │                       │                     │
│          │                    │                       ▼                     │
│          │                    │              ┌───────────────────┐          │
│          │                    │              │  Geospatial Index │          │
│          │                    │              │  (Quadtree/S2)    │          │
│          │                    │              └───────────────────┘          │
│          │                    │                                             │
│          ▼                    ▼                                             │
│  ┌───────────────────────────────────────────────────────────────────────┐ │
│  │                       Supporting Services                              │ │
│  │  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐  ┌──────────┐ │ │
│  │  │  Pricing │  │  Payment │  │  Rating  │  │   ETA    │  │  Surge   │ │ │
│  │  │ Service  │  │ Service  │  │ Service  │  │ Service  │  │ Pricing  │ │ │
│  │  └──────────┘  └──────────┘  └──────────┘  └──────────┘  └──────────┘ │ │
│  └───────────────────────────────────────────────────────────────────────┘ │
│                                                                              │
└─────────────────────────────────────────────────────────────────────────────┘
```

### Location Service

```python
"""
Геопространственное индексирование с использованием Quadtree.

Альтернативы:
- Google S2 Geometry
- Geohash
- H3 (Uber's Hexagonal Hierarchical Geospatial Indexing)
"""

from dataclasses import dataclass
from typing import List, Optional, Set, Tuple
import math
import redis.asyncio as redis


@dataclass
class Location:
    lat: float
    lng: float


@dataclass
class Driver:
    id: str
    location: Location
    is_available: bool
    vehicle_type: str  # economy, comfort, premium


class QuadTreeNode:
    """Узел квадродерева для геопространственного индекса."""

    def __init__(self, bounds: Tuple[float, float, float, float], capacity: int = 50):
        self.bounds = bounds  # (min_lat, min_lng, max_lat, max_lng)
        self.capacity = capacity
        self.drivers: List[Driver] = []
        self.children: List['QuadTreeNode'] = []
        self.is_divided = False

    def contains(self, location: Location) -> bool:
        min_lat, min_lng, max_lat, max_lng = self.bounds
        return (min_lat <= location.lat <= max_lat and
                min_lng <= location.lng <= max_lng)

    def intersects(self, bounds: Tuple[float, float, float, float]) -> bool:
        min_lat, min_lng, max_lat, max_lng = self.bounds
        b_min_lat, b_min_lng, b_max_lat, b_max_lng = bounds
        return not (max_lat < b_min_lat or min_lat > b_max_lat or
                    max_lng < b_min_lng or min_lng > b_max_lng)

    def subdivide(self):
        """Разделение узла на 4 дочерних."""
        min_lat, min_lng, max_lat, max_lng = self.bounds
        mid_lat = (min_lat + max_lat) / 2
        mid_lng = (min_lng + max_lng) / 2

        self.children = [
            QuadTreeNode((min_lat, min_lng, mid_lat, mid_lng), self.capacity),  # SW
            QuadTreeNode((mid_lat, min_lng, max_lat, mid_lng), self.capacity),  # NW
            QuadTreeNode((min_lat, mid_lng, mid_lat, max_lng), self.capacity),  # SE
            QuadTreeNode((mid_lat, mid_lng, max_lat, max_lng), self.capacity),  # NE
        ]
        self.is_divided = True

    def insert(self, driver: Driver) -> bool:
        """Вставка водителя в дерево."""
        if not self.contains(driver.location):
            return False

        if len(self.drivers) < self.capacity and not self.is_divided:
            self.drivers.append(driver)
            return True

        if not self.is_divided:
            self.subdivide()
            # Перемещаем существующих водителей в дочерние узлы
            for d in self.drivers:
                for child in self.children:
                    if child.insert(d):
                        break
            self.drivers = []

        for child in self.children:
            if child.insert(driver):
                return True

        return False

    def remove(self, driver_id: str) -> bool:
        """Удаление водителя из дерева."""
        self.drivers = [d for d in self.drivers if d.id != driver_id]

        for child in self.children:
            child.remove(driver_id)

        return True

    def query_range(self, bounds: Tuple[float, float, float, float]) -> List[Driver]:
        """Поиск водителей в заданной области."""
        if not self.intersects(bounds):
            return []

        found = []
        for driver in self.drivers:
            if (bounds[0] <= driver.location.lat <= bounds[2] and
                bounds[1] <= driver.location.lng <= bounds[3]):
                found.append(driver)

        for child in self.children:
            found.extend(child.query_range(bounds))

        return found


class LocationService:
    """Сервис отслеживания локации водителей."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client
        # Границы покрытия (пример: весь мир)
        self.quadtree = QuadTreeNode((-90, -180, 90, 180))
        self.EARTH_RADIUS_KM = 6371

    async def update_driver_location(self, driver_id: str, lat: float, lng: float, is_available: bool):
        """Обновление локации водителя."""
        # Удаляем старую позицию
        self.quadtree.remove(driver_id)

        driver = Driver(
            id=driver_id,
            location=Location(lat, lng),
            is_available=is_available,
            vehicle_type="economy"  # В реальности из БД
        )

        # Вставляем новую
        self.quadtree.insert(driver)

        # Также сохраняем в Redis для быстрого доступа
        await self.redis.geoadd(
            "driver_locations",
            (lng, lat, driver_id)
        )

        # Сохраняем статус
        await self.redis.hset(
            f"driver:{driver_id}",
            mapping={
                "lat": lat,
                "lng": lng,
                "is_available": int(is_available),
                "updated_at": datetime.utcnow().isoformat()
            }
        )

    async def find_nearby_drivers(
        self,
        lat: float,
        lng: float,
        radius_km: float = 5.0,
        limit: int = 10
    ) -> List[Driver]:
        """Поиск ближайших доступных водителей."""
        # Конвертируем радиус в градусы (приблизительно)
        lat_delta = radius_km / 111  # 1 градус ≈ 111 км
        lng_delta = radius_km / (111 * math.cos(math.radians(lat)))

        bounds = (lat - lat_delta, lng - lng_delta, lat + lat_delta, lng + lng_delta)

        # Поиск в квадродереве
        candidates = self.quadtree.query_range(bounds)

        # Фильтруем доступных
        available = [d for d in candidates if d.is_available]

        # Сортируем по расстоянию
        available.sort(key=lambda d: self._haversine_distance(
            lat, lng, d.location.lat, d.location.lng
        ))

        return available[:limit]

    def _haversine_distance(self, lat1: float, lng1: float, lat2: float, lng2: float) -> float:
        """Вычисление расстояния между двумя точками в км."""
        lat1_rad = math.radians(lat1)
        lat2_rad = math.radians(lat2)
        delta_lat = math.radians(lat2 - lat1)
        delta_lng = math.radians(lng2 - lng1)

        a = (math.sin(delta_lat / 2) ** 2 +
             math.cos(lat1_rad) * math.cos(lat2_rad) * math.sin(delta_lng / 2) ** 2)
        c = 2 * math.atan2(math.sqrt(a), math.sqrt(1 - a))

        return self.EARTH_RADIUS_KM * c


class MatchingService:
    """Сервис подбора водителя."""

    def __init__(self, location_service: LocationService, redis_client: redis.Redis):
        self.location = location_service
        self.redis = redis_client
        self.MATCH_TIMEOUT = 30  # секунд
        self.MAX_SEARCH_RADIUS = 10  # км

    async def request_ride(
        self,
        rider_id: str,
        pickup_lat: float,
        pickup_lng: float,
        dropoff_lat: float,
        dropoff_lng: float,
        vehicle_type: str = "economy"
    ) -> Optional[str]:
        """Запрос поездки и поиск водителя."""
        trip_id = generate_trip_id()

        # Сохраняем запрос
        trip_data = {
            "rider_id": rider_id,
            "pickup_lat": pickup_lat,
            "pickup_lng": pickup_lng,
            "dropoff_lat": dropoff_lat,
            "dropoff_lng": dropoff_lng,
            "vehicle_type": vehicle_type,
            "status": "searching",
            "created_at": datetime.utcnow().isoformat()
        }
        await self.redis.hset(f"trip:{trip_id}", mapping=trip_data)

        # Ищем водителей с увеличивающимся радиусом
        radius = 2.0  # начинаем с 2 км
        while radius <= self.MAX_SEARCH_RADIUS:
            drivers = await self.location.find_nearby_drivers(
                pickup_lat, pickup_lng, radius, limit=5
            )

            for driver in drivers:
                if driver.vehicle_type == vehicle_type:
                    accepted = await self._offer_to_driver(driver.id, trip_id)
                    if accepted:
                        await self._confirm_match(trip_id, driver.id)
                        return trip_id

            radius += 2.0  # увеличиваем радиус

        # Не нашли водителя
        await self.redis.hset(f"trip:{trip_id}", "status", "no_drivers")
        return None

    async def _offer_to_driver(self, driver_id: str, trip_id: str) -> bool:
        """Отправка предложения водителю."""
        # Отправляем push notification и ждём ответа
        offer_key = f"offer:{trip_id}:{driver_id}"
        await self.redis.setex(offer_key, 15, "pending")  # 15 секунд на ответ

        # В реальности здесь WebSocket или push
        # Ждём ответа
        for _ in range(15):
            response = await self.redis.get(offer_key)
            if response == b"accepted":
                return True
            if response == b"declined":
                return False
            await asyncio.sleep(1)

        return False  # Таймаут

    async def _confirm_match(self, trip_id: str, driver_id: str):
        """Подтверждение матча."""
        await self.redis.hset(f"trip:{trip_id}", mapping={
            "status": "matched",
            "driver_id": driver_id,
            "matched_at": datetime.utcnow().isoformat()
        })

        # Помечаем водителя как занятого
        await self.redis.hset(f"driver:{driver_id}", "is_available", 0)


class ETAService:
    """Расчёт времени прибытия."""

    def __init__(self, maps_client):
        self.maps = maps_client  # Google Maps API или аналог

    async def calculate_eta(
        self,
        from_lat: float,
        from_lng: float,
        to_lat: float,
        to_lng: float
    ) -> int:
        """Расчёт ETA в секундах."""
        # Используем внешний API для точного расчёта с учётом трафика
        route = await self.maps.get_directions(
            origin=(from_lat, from_lng),
            destination=(to_lat, to_lng),
            departure_time="now"
        )

        return route.duration_in_traffic


class PricingService:
    """Сервис ценообразования."""

    def __init__(self, redis_client: redis.Redis):
        self.redis = redis_client

    async def calculate_price(
        self,
        pickup_lat: float,
        pickup_lng: float,
        dropoff_lat: float,
        dropoff_lng: float,
        vehicle_type: str
    ) -> dict:
        """Расчёт стоимости поездки."""
        # Базовые тарифы по типу авто
        BASE_FARE = {"economy": 2.0, "comfort": 3.5, "premium": 5.0}
        PER_KM = {"economy": 1.0, "comfort": 1.5, "premium": 2.5}
        PER_MINUTE = {"economy": 0.15, "comfort": 0.25, "premium": 0.40}

        # Расстояние
        distance_km = self._calculate_distance(
            pickup_lat, pickup_lng, dropoff_lat, dropoff_lng
        )

        # Примерное время (средняя скорость 30 км/ч в городе)
        estimated_minutes = (distance_km / 30) * 60

        # Проверяем surge pricing
        surge_multiplier = await self._get_surge_multiplier(pickup_lat, pickup_lng)

        base = BASE_FARE[vehicle_type]
        distance_fare = distance_km * PER_KM[vehicle_type]
        time_fare = estimated_minutes * PER_MINUTE[vehicle_type]

        total = (base + distance_fare + time_fare) * surge_multiplier

        return {
            "base_fare": base,
            "distance_fare": round(distance_fare, 2),
            "time_fare": round(time_fare, 2),
            "surge_multiplier": surge_multiplier,
            "total": round(total, 2),
            "currency": "USD",
            "estimated_distance_km": round(distance_km, 1),
            "estimated_minutes": round(estimated_minutes)
        }

    async def _get_surge_multiplier(self, lat: float, lng: float) -> float:
        """Получение коэффициента surge pricing для зоны."""
        # Определяем зону
        zone = self._get_zone(lat, lng)

        # Получаем текущий коэффициент
        surge = await self.redis.get(f"surge:{zone}")
        return float(surge) if surge else 1.0

    def _get_zone(self, lat: float, lng: float) -> str:
        """Определение зоны для surge pricing."""
        # Простое разбиение на сетку
        zone_lat = int(lat * 100)
        zone_lng = int(lng * 100)
        return f"{zone_lat}_{zone_lng}"
```

---

## 5. Netflix (Стриминговый сервис)

### Требования

**Функциональные:**
- Каталог контента (фильмы, сериалы)
- Поиск и рекомендации
- Видео стриминг (adaptive bitrate)
- Профили пользователей
- Продолжение просмотра

**Нефункциональные:**
- Latency для начала видео < 2 секунды
- 99.99% availability
- Поддержка миллионов concurrent viewers
- Глобальная доставка контента

### Высокоуровневый дизайн

```
┌──────────────────────────────────────────────────────────────────────────────┐
│                           Netflix Architecture                                │
├──────────────────────────────────────────────────────────────────────────────┤
│                                                                               │
│  ┌─────────┐     ┌─────────────────────────────────────────────────────────┐│
│  │ Client  │────►│                    Control Plane                        ││
│  │  Apps   │     │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     ││
│  │         │     │  │  API        │  │  User       │  │   Search    │     ││
│  └────┬────┘     │  │  Gateway    │  │  Service    │  │   Service   │     ││
│       │          │  └─────────────┘  └─────────────┘  └─────────────┘     ││
│       │          │  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐     ││
│       │          │  │  Recommend  │  │  Playback   │  │   Content   │     ││
│       │          │  │  Service    │  │  Service    │  │   Service   │     ││
│       │          │  └─────────────┘  └─────────────┘  └─────────────┘     ││
│       │          └─────────────────────────────────────────────────────────┘│
│       │                                                                      │
│       │          ┌─────────────────────────────────────────────────────────┐│
│       └─────────►│                     Data Plane                          ││
│                  │                                                          ││
│                  │  ┌─────────────────────────────────────────────────┐    ││
│                  │  │                  Open Connect CDN               │    ││
│                  │  │                                                  │    ││
│                  │  │    ┌─────┐   ┌─────┐   ┌─────┐   ┌─────┐       │    ││
│                  │  │    │ OCA │   │ OCA │   │ OCA │   │ OCA │       │    ││
│                  │  │    │ NYC │   │ LA  │   │Tokyo│   │London│       │    ││
│                  │  │    └─────┘   └─────┘   └─────┘   └─────┘       │    ││
│                  │  │                                                  │    ││
│                  │  └─────────────────────────────────────────────────┘    ││
│                  │                                                          ││
│                  └─────────────────────────────────────────────────────────┘│
│                                                                               │
│  ┌─────────────────────────────────────────────────────────────────────────┐│
│  │                        Content Pipeline                                  ││
│  │                                                                          ││
│  │  ┌─────────┐    ┌──────────────┐    ┌────────────┐    ┌────────────┐   ││
│  │  │ Ingest  │───►│  Transcoding │───►│   Package  │───►│ Distribute ││   ││
│  │  │ (S3)    │    │  (Multiple   │    │  (HLS/DASH)│    │  to OCAs   ││   ││
│  │  │         │    │   bitrates)  │    │            │    │            ││   ││
│  │  └─────────┘    └──────────────┘    └────────────┘    └────────────┘   ││
│  │                                                                          ││
│  └─────────────────────────────────────────────────────────────────────────┘│
│                                                                               │
└──────────────────────────────────────────────────────────────────────────────┘
```

### Video Streaming

```python
"""
Adaptive Bitrate Streaming (ABR):
- Видео кодируется в несколько качеств
- Клиент динамически переключается между ними
- Используется HLS или DASH протокол
"""

from dataclasses import dataclass
from typing import List, Dict
import aiohttp


@dataclass
class VideoQuality:
    resolution: str      # "1080p", "720p", etc.
    bitrate: int         # kbps
    codec: str           # "h264", "h265", "av1"
    segment_duration: int  # секунды


@dataclass
class VideoManifest:
    video_id: str
    duration: int        # общая длительность в секундах
    qualities: List[VideoQuality]
    segments: Dict[str, List[str]]  # quality -> list of segment URLs


class PlaybackService:
    def __init__(self, cdn_client, user_service, content_db):
        self.cdn = cdn_client
        self.users = user_service
        self.content = content_db

    async def get_playback_info(self, user_id: str, video_id: str) -> dict:
        """Получение информации для начала воспроизведения."""
        # Проверяем права доступа
        user = await self.users.get_user(user_id)
        video = await self.content.get_video(video_id)

        if not self._can_watch(user, video):
            raise PermissionError("Content not available in your region")

        # Получаем оптимальный CDN endpoint
        cdn_endpoint = await self._get_nearest_cdn(user.location)

        # Получаем позицию продолжения просмотра
        watch_position = await self._get_watch_position(user_id, video_id)

        # Генерируем manifest URL с авторизацией
        manifest_url = await self._generate_manifest_url(video_id, cdn_endpoint)

        return {
            "video_id": video_id,
            "manifest_url": manifest_url,
            "start_position": watch_position,
            "available_qualities": video.qualities,
            "subtitles": video.subtitle_tracks,
            "audio_tracks": video.audio_tracks
        }

    async def _get_nearest_cdn(self, user_location: dict) -> str:
        """Выбор ближайшего CDN endpoint."""
        # Netflix использует DNS-based routing и BGP anycast
        # Здесь упрощённая версия
        endpoints = await self.cdn.get_healthy_endpoints()

        # Сортируем по latency
        endpoints.sort(key=lambda e: self._estimate_latency(user_location, e.location))

        return endpoints[0].url

    async def _generate_manifest_url(self, video_id: str, cdn_endpoint: str) -> str:
        """Генерация подписанного URL для manifest."""
        expires = int(time.time()) + 3600  # 1 час
        signature = self._sign_url(video_id, expires)

        return f"{cdn_endpoint}/videos/{video_id}/manifest.mpd?expires={expires}&sig={signature}"

    async def update_watch_position(self, user_id: str, video_id: str, position: int):
        """Обновление позиции просмотра."""
        await self.content.update_watch_position(user_id, video_id, position)

        # Если досмотрели до конца (>95%), помечаем как просмотренное
        video = await self.content.get_video(video_id)
        if position / video.duration > 0.95:
            await self.users.mark_as_watched(user_id, video_id)


class ContentDeliveryService:
    """Управление Open Connect CDN."""

    def __init__(self, redis_client, storage_client):
        self.redis = redis_client
        self.storage = storage_client  # S3-like

    async def get_video_segment(
        self,
        video_id: str,
        quality: str,
        segment_number: int
    ) -> bytes:
        """Получение сегмента видео."""
        # Проверяем локальный кеш OCA
        cache_key = f"segment:{video_id}:{quality}:{segment_number}"

        cached = await self.redis.get(cache_key)
        if cached:
            return cached

        # Загружаем из origin storage
        segment_path = f"videos/{video_id}/{quality}/segment_{segment_number}.m4s"
        segment_data = await self.storage.get_object(segment_path)

        # Кешируем на OCA
        await self.redis.setex(
            cache_key,
            86400,  # 24 часа
            segment_data
        )

        return segment_data


class RecommendationService:
    """Сервис рекомендаций."""

    def __init__(self, ml_service, user_service, content_db):
        self.ml = ml_service
        self.users = user_service
        self.content = content_db

    async def get_home_feed(self, user_id: str) -> List[dict]:
        """Генерация персонализированной главной страницы."""
        user = await self.users.get_user(user_id)
        profile = await self.users.get_active_profile(user_id)

        # Получаем различные ряды контента
        rows = []

        # 1. Продолжить просмотр
        continue_watching = await self._get_continue_watching(user_id)
        if continue_watching:
            rows.append({
                "title": "Continue Watching",
                "items": continue_watching
            })

        # 2. Персонализированные рекомендации (ML)
        ml_recommendations = await self.ml.get_recommendations(
            user_id=user_id,
            profile_embedding=profile.embedding,
            num_items=50
        )
        rows.append({
            "title": "Top Picks for You",
            "items": ml_recommendations[:10]
        })

        # 3. Trending сейчас
        trending = await self.content.get_trending(user.region)
        rows.append({
            "title": "Trending Now",
            "items": trending[:10]
        })

        # 4. По жанрам на основе предпочтений
        favorite_genres = await self._get_favorite_genres(user_id)
        for genre in favorite_genres[:3]:
            genre_content = await self.content.get_by_genre(genre, user.region)
            rows.append({
                "title": f"{genre} Movies & TV",
                "items": genre_content[:10]
            })

        # 5. Новые релизы
        new_releases = await self.content.get_new_releases(user.region)
        rows.append({
            "title": "New Releases",
            "items": new_releases[:10]
        })

        return rows

    async def _get_continue_watching(self, user_id: str) -> List[dict]:
        """Контент для продолжения просмотра."""
        watch_history = await self.users.get_watch_history(user_id, limit=20)

        continue_items = []
        for item in watch_history:
            video = await self.content.get_video(item.video_id)

            # Пропускаем досмотренные
            if item.position / video.duration > 0.95:
                continue

            continue_items.append({
                "video": video,
                "progress": item.position / video.duration,
                "last_watched": item.updated_at
            })

        return continue_items[:10]
```

### Content Pipeline

```python
"""
Pipeline обработки видео контента:
1. Ingest - загрузка исходного видео
2. Transcoding - кодирование в разные качества
3. Packaging - создание сегментов для streaming
4. Distribution - распространение на CDN
"""

import asyncio
from dataclasses import dataclass
from enum import Enum
from typing import List


class TranscodingProfile(Enum):
    # (resolution, bitrate_kbps, codec)
    HD_1080P_H264 = ("1920x1080", 8000, "h264")
    HD_720P_H264 = ("1280x720", 5000, "h264")
    SD_480P_H264 = ("854x480", 2500, "h264")
    SD_360P_H264 = ("640x360", 1000, "h264")

    HD_1080P_HEVC = ("1920x1080", 5000, "hevc")  # Более эффективный кодек
    HD_720P_HEVC = ("1280x720", 3000, "hevc")

    UHD_4K_HEVC = ("3840x2160", 15000, "hevc")
    UHD_4K_AV1 = ("3840x2160", 10000, "av1")    # Новейший кодек


@dataclass
class TranscodingJob:
    id: str
    video_id: str
    source_path: str
    profiles: List[TranscodingProfile]
    status: str
    progress: float


class ContentPipeline:
    def __init__(self, storage, transcoder, cdn_manager, job_queue):
        self.storage = storage
        self.transcoder = transcoder
        self.cdn = cdn_manager
        self.queue = job_queue

    async def process_new_content(self, video_id: str, source_path: str, metadata: dict):
        """Обработка нового контента."""
        # 1. Создаём job
        job = TranscodingJob(
            id=generate_job_id(),
            video_id=video_id,
            source_path=source_path,
            profiles=self._select_profiles(metadata),
            status="pending",
            progress=0.0
        )

        # 2. Добавляем в очередь
        await self.queue.enqueue(job)

        return job.id

    def _select_profiles(self, metadata: dict) -> List[TranscodingProfile]:
        """Выбор профилей транскодирования на основе исходного качества."""
        source_height = metadata.get("height", 1080)
        profiles = []

        if source_height >= 2160:
            profiles.extend([
                TranscodingProfile.UHD_4K_HEVC,
                TranscodingProfile.UHD_4K_AV1,
            ])

        if source_height >= 1080:
            profiles.extend([
                TranscodingProfile.HD_1080P_H264,
                TranscodingProfile.HD_1080P_HEVC,
            ])

        # Всегда добавляем lower quality для плохого интернета
        profiles.extend([
            TranscodingProfile.HD_720P_H264,
            TranscodingProfile.SD_480P_H264,
            TranscodingProfile.SD_360P_H264,
        ])

        return profiles

    async def run_transcoding(self, job: TranscodingJob):
        """Выполнение транскодирования."""
        output_files = {}

        for i, profile in enumerate(job.profiles):
            resolution, bitrate, codec = profile.value

            output_path = f"transcoded/{job.video_id}/{profile.name}.mp4"

            # Запускаем FFmpeg или аналог
            await self.transcoder.transcode(
                input_path=job.source_path,
                output_path=output_path,
                resolution=resolution,
                bitrate=bitrate,
                codec=codec
            )

            output_files[profile.name] = output_path
            job.progress = (i + 1) / len(job.profiles)

        return output_files

    async def package_for_streaming(self, video_id: str, transcoded_files: dict):
        """Упаковка в HLS/DASH формат."""
        # Создаём сегменты по 4 секунды
        segment_duration = 4

        for quality_name, file_path in transcoded_files.items():
            segments = await self.transcoder.segment(
                input_path=file_path,
                output_dir=f"packaged/{video_id}/{quality_name}/",
                segment_duration=segment_duration
            )

        # Генерируем master manifest
        manifest = await self._generate_manifest(video_id, transcoded_files)

        await self.storage.put_object(
            f"packaged/{video_id}/manifest.mpd",
            manifest
        )

    async def distribute_to_cdn(self, video_id: str):
        """Распространение на CDN endpoints."""
        # Получаем список популярных регионов
        popular_regions = await self.cdn.get_popular_regions()

        # Предзагружаем на OCA в этих регионах
        for region in popular_regions:
            endpoints = await self.cdn.get_endpoints(region)
            for endpoint in endpoints:
                await self.cdn.preload(
                    endpoint=endpoint,
                    content_path=f"packaged/{video_id}/"
                )
```

---

## Best Practices для System Design

### Структура ответа на собеседовании

```
1. Clarify Requirements (3-5 минут)
   - Функциональные требования
   - Нефункциональные требования (scale, latency, availability)
   - Ограничения и допущения

2. Back-of-envelope Calculations (3-5 минут)
   - Оценка QPS
   - Оценка хранения
   - Оценка bandwidth

3. High-Level Design (10-15 минут)
   - Основные компоненты
   - Потоки данных
   - API design

4. Deep Dive (10-15 минут)
   - Детали критичных компонентов
   - Trade-offs
   - Edge cases

5. Scale & Wrap-up (5-10 минут)
   - Как масштабировать
   - Мониторинг и alerting
   - Future improvements
```

### Ключевые паттерны

```
1. Data Partitioning
   - Horizontal: sharding by key
   - Vertical: split by feature
   - Functional: different DBs for different purposes

2. Caching Strategies
   - Cache-aside (lazy loading)
   - Write-through
   - Write-behind (write-back)

3. Load Balancing
   - Round Robin
   - Least Connections
   - IP Hash (sticky sessions)

4. Communication Patterns
   - Sync: REST, gRPC
   - Async: Message queues, Event streaming

5. Consistency Models
   - Strong consistency
   - Eventual consistency
   - Causal consistency

6. Failure Handling
   - Circuit Breaker
   - Retry with exponential backoff
   - Fallbacks
```

### Типичные ошибки

```
1. Начинать с деталей без понимания требований
2. Забывать о масштабе (designing for 100 users vs 100M)
3. Не обсуждать trade-offs
4. Игнорировать failure scenarios
5. Чрезмерно усложнять решение
6. Забывать о мониторинге и операционной стороне
```

---

## Дополнительные ресурсы

- **Книги:**
  - "Designing Data-Intensive Applications" - Martin Kleppmann
  - "System Design Interview" - Alex Xu

- **Блоги:**
  - Netflix Tech Blog
  - Uber Engineering Blog
  - Discord Engineering Blog
  - Instagram Engineering

- **Практика:**
  - leetcode.com (System Design section)
  - educative.io (Grokking System Design)
