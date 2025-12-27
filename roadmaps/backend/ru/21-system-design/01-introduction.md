# Введение в System Design (Проектирование систем)

## Что такое System Design?

**System Design** (проектирование систем) — это процесс определения архитектуры, компонентов, модулей, интерфейсов и данных системы для удовлетворения заданных требований. Это искусство и наука создания масштабируемых, надёжных и эффективных программных систем.

### Зачем нужен System Design?

1. **Масштабируемость** — система должна справляться с ростом нагрузки
2. **Надёжность** — система должна работать даже при частичных сбоях
3. **Производительность** — система должна отвечать быстро
4. **Поддерживаемость** — код должен быть понятным и легко изменяемым
5. **Экономичность** — ресурсы стоят денег, их нужно использовать эффективно

### Когда нужен System Design?

```
Маленький проект (1 сервер, 100 пользователей):
→ Достаточно базовой архитектуры

Средний проект (несколько серверов, 10K пользователей):
→ Нужно продумать масштабирование и отказоустойчивость

Большой проект (сотни серверов, миллионы пользователей):
→ Требуется тщательное проектирование каждого компонента
```

---

## Основные этапы проектирования системы

### 1. Сбор и уточнение требований (5-10 минут на интервью)

На этом этапе нужно понять, **что именно** мы строим:

```
Пример: "Спроектируйте Twitter"

Плохой подход: Сразу начать рисовать архитектуру

Хороший подход: Задать уточняющие вопросы
- Какие основные функции? (посты, лента, подписки?)
- Сколько пользователей? (1K или 1B?)
- Какие требования по latency?
- Нужна ли поддержка медиа-контента?
```

### 2. Оценка масштаба (Capacity Estimation)

Прикинуть числа, чтобы понять масштаб проблемы:

```python
# Пример расчёта для Twitter-подобной системы

# Предположения
total_users = 500_000_000        # 500M пользователей
daily_active_users = 200_000_000  # 200M DAU
avg_tweets_per_day = 2            # в среднем 2 твита на активного юзера

# Расчёты
tweets_per_day = daily_active_users * avg_tweets_per_day  # 400M твитов/день
tweets_per_second = tweets_per_day / (24 * 3600)          # ~4600 твитов/сек

# Чтение ленты (обычно читают в 100 раз больше, чем пишут)
read_requests_per_second = tweets_per_second * 100        # ~460K запросов/сек

# Хранение (предположим, средний твит = 300 байт)
daily_storage = tweets_per_day * 300                      # ~120 GB/день
yearly_storage = daily_storage * 365                      # ~44 TB/год
```

### 3. Проектирование высокоуровневой архитектуры

Нарисовать основные компоненты и их взаимодействие:

```
┌─────────┐     ┌──────────────┐     ┌─────────────┐
│ Клиенты │────▶│ Load Balancer│────▶│ Web Servers │
└─────────┘     └──────────────┘     └──────┬──────┘
                                            │
                    ┌───────────────────────┼───────────────────────┐
                    ▼                       ▼                       ▼
            ┌──────────────┐       ┌──────────────┐       ┌──────────────┐
            │    Cache     │       │  App Servers │       │    CDN       │
            │   (Redis)    │       │              │       │  (статика)   │
            └──────────────┘       └──────┬───────┘       └──────────────┘
                                          │
                    ┌─────────────────────┼─────────────────────┐
                    ▼                     ▼                     ▼
            ┌──────────────┐     ┌──────────────┐     ┌──────────────┐
            │   Primary DB │     │  Replica DB  │     │ Message Queue│
            │   (Write)    │────▶│   (Read)     │     │   (Kafka)    │
            └──────────────┘     └──────────────┘     └──────────────┘
```

### 4. Детальное проектирование компонентов

Углубиться в конкретные части системы:

- Схема базы данных
- API endpoints
- Алгоритмы (например, генерация ленты)
- Стратегии кэширования

### 5. Обсуждение узких мест и компромиссов

Каждое решение имеет trade-offs:

```
Пример: Где хранить ленту пользователя?

Вариант A: Генерировать на лету (fan-out on read)
+ Простая запись нового поста
- Медленное чтение ленты
→ Подходит для: пользователей с большим количеством подписок

Вариант B: Предгенерировать при публикации (fan-out on write)
+ Быстрое чтение ленты
- Сложная и дорогая запись (особенно для celebrity)
→ Подходит для: большинства обычных пользователей

Вариант C: Гибридный подход
→ Обычные посты: fan-out on write
→ Celebrity посты: fan-out on read
```

---

## Ключевые вопросы перед проектированием

### Вопросы о пользователях и масштабе

```markdown
1. Сколько пользователей будет использовать систему?
   - Сейчас: 10K, 100K, 1M, 100M, 1B?
   - Через год? Через 5 лет?

2. Какова географическая распределённость?
   - Один регион или глобально?
   - Нужна ли мультирегиональность?

3. Какой паттерн нагрузки?
   - Равномерная в течение дня?
   - Пиковые часы?
   - Сезонность?
```

### Вопросы о функциональности

```markdown
4. Какие основные use cases?
   - Что пользователи делают чаще всего?
   - Какие действия критичны?

5. Какие данные обрабатываются?
   - Тип: текст, изображения, видео, файлы?
   - Размер: байты, килобайты, гигабайты?
   - Структурированность: SQL или NoSQL?

6. Как данные связаны между собой?
   - Много связей (social graph)?
   - Иерархические данные?
   - Временные ряды?
```

### Вопросы о требованиях к качеству

```markdown
7. Какая допустимая задержка (latency)?
   - < 100ms (real-time)
   - < 1s (interactive)
   - < 10s (background processing)

8. Какая допустимая недоступность?
   - 99% (7.3 часа простоя в месяц)
   - 99.9% (43 минуты в месяц)
   - 99.99% (4.3 минуты в месяц)

9. Consistency vs Availability?
   - Критично ли видеть самые свежие данные?
   - Или важнее, чтобы система всегда отвечала?
```

---

## Функциональные и нефункциональные требования

### Функциональные требования (Functional Requirements)

Описывают **что** система должна делать:

```markdown
Пример для URL Shortener:

Функциональные требования:
1. Пользователь может создать короткую ссылку из длинной
2. При переходе по короткой ссылке происходит редирект на оригинал
3. Пользователь может указать custom alias для ссылки
4. Ссылки могут иметь срок действия (expiration)
5. Пользователь может видеть статистику переходов

API:
POST /shorten
  Request:  { "url": "https://very-long-url.com/...", "custom_alias": "my-link" }
  Response: { "short_url": "https://short.ly/my-link" }

GET /{short_code}
  Response: 301 Redirect to original URL
```

### Нефункциональные требования (Non-Functional Requirements)

Описывают **как хорошо** система должна работать:

```markdown
Пример для URL Shortener:

Нефункциональные требования:

1. Масштабируемость (Scalability)
   - 100M новых ссылок в месяц
   - 10B редиректов в месяц
   - Соотношение чтение:запись = 100:1

2. Производительность (Performance)
   - Редирект < 100ms (p99)
   - Создание ссылки < 500ms (p99)

3. Доступность (Availability)
   - 99.9% uptime
   - Graceful degradation при частичных сбоях

4. Надёжность (Reliability)
   - Ссылки не должны теряться
   - Данные реплицируются между датацентрами

5. Безопасность (Security)
   - Rate limiting против abuse
   - Проверка URL на вредоносность
```

### Таблица сравнения

| Аспект | Функциональные | Нефункциональные |
|--------|----------------|------------------|
| Вопрос | Что делает? | Как хорошо делает? |
| Пример | "Создать пост" | "Создать пост за < 200ms" |
| Тестирование | Unit/Integration tests | Load/Stress tests |
| Изменчивость | Часто меняются | Относительно стабильны |

---

## Примеры типичных задач на System Design интервью

### Уровень Junior/Middle

```markdown
1. URL Shortener (bit.ly)
   Ключевые темы: hashing, база данных, кэширование

2. Pastebin
   Ключевые темы: хранение текста, генерация ID, expiration

3. Rate Limiter
   Ключевые темы: алгоритмы (token bucket, sliding window), Redis
```

### Уровень Middle/Senior

```markdown
4. Twitter/Instagram Feed
   Ключевые темы: fan-out, timeline generation, caching

5. Chat System (WhatsApp, Slack)
   Ключевые темы: WebSocket, message queues, присутствие пользователей

6. Notification System
   Ключевые темы: push notifications, приоритеты, дедупликация

7. Search Autocomplete
   Ключевые темы: trie, кэширование, ranking
```

### Уровень Senior/Staff

```markdown
8. YouTube/Netflix
   Ключевые темы: video streaming, CDN, encoding, recommendations

9. Google Drive/Dropbox
   Ключевые темы: file sync, chunking, conflict resolution

10. Distributed Cache (Redis)
    Ключевые темы: consistent hashing, replication, eviction

11. Web Crawler
    Ключевые темы: distributed processing, politeness, deduplication

12. Ticketmaster (Event Booking)
    Ключевые темы: inventory management, race conditions, waiting queue
```

---

## Базовые компоненты любой системы

### 1. Клиенты (Clients)

Точки входа пользователей в систему:

```
Типы клиентов:
├── Web Browser (React, Vue, Angular)
├── Mobile Apps (iOS, Android)
├── Desktop Apps
├── CLI tools
├── IoT устройства
└── Другие сервисы (B2B API)

Особенности:
- Ненадёжное соединение (особенно mobile)
- Разные возможности устройств
- Необходимость offline-режима
- Кэширование на стороне клиента
```

### 2. DNS (Domain Name System)

Преобразует доменные имена в IP-адреса:

```
Запрос: api.example.com
                │
                ▼
        ┌──────────────┐
        │  DNS Server  │
        └──────┬───────┘
               │
               ▼
Ответ: 192.168.1.100

Особенности для System Design:
- DNS-based load balancing
- GeoDNS для направления в ближайший датацентр
- TTL для кэширования
- Failover через DNS
```

### 3. CDN (Content Delivery Network)

Географически распределённая сеть серверов для статического контента:

```
Без CDN:
Пользователь в Токио ──────────────────▶ Сервер в Нью-Йорке
                     (высокая latency)

С CDN:
Пользователь в Токио ─────▶ CDN Edge в Токио ─────▶ Origin в Нью-Йорке
                     (низкая latency, кэш)

Что хранить в CDN:
- Статические файлы (JS, CSS, изображения)
- Видео и аудио контент
- API responses (с осторожностью)
```

### 4. Load Balancer (Балансировщик нагрузки)

Распределяет запросы между серверами:

```
                    ┌─────────────────┐
                    │  Load Balancer  │
                    └────────┬────────┘
                             │
          ┌──────────────────┼──────────────────┐
          ▼                  ▼                  ▼
    ┌──────────┐       ┌──────────┐       ┌──────────┐
    │ Server 1 │       │ Server 2 │       │ Server 3 │
    └──────────┘       └──────────┘       └──────────┘

Алгоритмы балансировки:
- Round Robin: по очереди
- Least Connections: к наименее загруженному
- IP Hash: sticky sessions
- Weighted: с учётом мощности серверов

Примеры: Nginx, HAProxy, AWS ALB/NLB
```

### 5. Web/Application Servers

Обрабатывают бизнес-логику:

```python
# Пример структуры application server

class ApplicationServer:
    def __init__(self):
        self.cache = RedisClient()
        self.db = PostgresClient()
        self.queue = KafkaClient()

    async def handle_request(self, request):
        # 1. Валидация
        validated_data = self.validate(request)

        # 2. Проверка кэша
        cached = await self.cache.get(cache_key)
        if cached:
            return cached

        # 3. Бизнес-логика
        result = await self.process(validated_data)

        # 4. Сохранение
        await self.db.save(result)

        # 5. Отправка события
        await self.queue.publish("entity.created", result)

        # 6. Обновление кэша
        await self.cache.set(cache_key, result)

        return result

Характеристики:
- Stateless (состояние в БД/кэше)
- Горизонтально масштабируемые
- Контейнеризированные (Docker, K8s)
```

### 6. Базы данных (Databases)

Долговременное хранение данных:

```
SQL (реляционные):
├── PostgreSQL - универсальная, ACID
├── MySQL - популярная, хорошо масштабируется
└── Использовать когда: структурированные данные, сложные запросы, транзакции

NoSQL:
├── MongoDB - документы (JSON-like)
├── Cassandra - колоночная, высокая доступность
├── DynamoDB - key-value, AWS managed
└── Использовать когда: гибкая схема, горизонтальное масштабирование

Специализированные:
├── Redis - in-memory, кэш и структуры данных
├── Elasticsearch - полнотекстовый поиск
├── ClickHouse - аналитика, колоночная
└── Neo4j - графы и связи
```

### 7. Кэш (Cache)

Быстрое хранилище для часто запрашиваемых данных:

```
Уровни кэширования:
┌─────────────────────────────────────────────────────────┐
│ Client Cache (браузер, приложение)           ~ms       │
├─────────────────────────────────────────────────────────┤
│ CDN Cache (edge servers)                     ~10ms     │
├─────────────────────────────────────────────────────────┤
│ Application Cache (Redis, Memcached)         ~1-5ms    │
├─────────────────────────────────────────────────────────┤
│ Database Cache (query cache, buffer pool)    ~10ms     │
└─────────────────────────────────────────────────────────┘

Стратегии кэширования:
- Cache-Aside: приложение управляет кэшем
- Write-Through: запись сначала в кэш, потом в БД
- Write-Behind: асинхронная запись в БД
- Read-Through: кэш сам загружает данные из БД
```

### 8. Message Queue (Очередь сообщений)

Асинхронная коммуникация между сервисами:

```
Producer ─────▶ [Message Queue] ─────▶ Consumer

Примеры:
- RabbitMQ: классические очереди, routing
- Apache Kafka: streaming, высокая пропускная способность
- AWS SQS: managed, простой в использовании

Когда использовать:
1. Отложенная обработка (отправка email)
2. Пиковые нагрузки (буферизация)
3. Микросервисная коммуникация
4. Event-driven архитектура
```

### Как компоненты работают вместе

```
Пример: Пользователь публикует пост

1. Client → Load Balancer → Web Server
   [Получение запроса]

2. Web Server → Cache (Redis)
   [Проверка rate limit]

3. Web Server → Application Server
   [Валидация, бизнес-логика]

4. Application Server → Primary Database
   [Сохранение поста]

5. Application Server → Message Queue
   [Событие "post.created"]

6. Background Worker ← Message Queue
   [Обновление лент подписчиков]

7. Background Worker → Cache
   [Инвалидация кэша лент]

8. CDN
   [Кэширование медиа-файлов]
```

---

## Как подходить к решению задач System Design

### Фреймворк RESHADED

```
R - Requirements (Требования)
    └── Функциональные и нефункциональные

E - Estimation (Оценка)
    └── Трафик, хранение, bandwidth

S - Storage Schema (Схема данных)
    └── Модели данных, выбор БД

H - High-level Design (Высокоуровневая архитектура)
    └── Основные компоненты и их связи

A - API Design (Проектирование API)
    └── Endpoints, форматы запросов/ответов

D - Detailed Design (Детальное проектирование)
    └── Углубление в критичные компоненты

E - Evaluation (Оценка)
    └── Bottlenecks, single points of failure

D - Distinctive Component (Уникальный компонент)
    └── Что делает эту систему особенной
```

### Пошаговый пример: URL Shortener

#### Шаг 1: Requirements

```markdown
Функциональные:
- Создание короткой ссылки из длинной
- Редирект по короткой ссылке
- Опциональный custom alias
- Статистика переходов

Нефункциональные:
- 100M новых URL в месяц
- 10B редиректов в месяц (100:1 read/write)
- Latency редиректа < 100ms
- 99.9% availability
- Ссылки хранятся 5 лет
```

#### Шаг 2: Estimation

```python
# Запись
urls_per_month = 100_000_000
urls_per_second = urls_per_month / (30 * 24 * 3600)  # ~40 URL/sec

# Чтение
redirects_per_month = 10_000_000_000
redirects_per_second = redirects_per_month / (30 * 24 * 3600)  # ~4000 req/sec

# Хранение
url_size = 500  # bytes (short + long URL + metadata)
storage_per_month = urls_per_month * url_size  # ~50 GB/month
storage_5_years = storage_per_month * 60  # ~3 TB
```

#### Шаг 3: Storage Schema

```sql
-- Основная таблица
CREATE TABLE urls (
    id BIGINT PRIMARY KEY,
    short_code VARCHAR(10) UNIQUE NOT NULL,
    original_url TEXT NOT NULL,
    user_id BIGINT,
    created_at TIMESTAMP DEFAULT NOW(),
    expires_at TIMESTAMP,
    click_count BIGINT DEFAULT 0
);

-- Индексы
CREATE INDEX idx_short_code ON urls(short_code);
CREATE INDEX idx_user_id ON urls(user_id);
```

#### Шаг 4: High-level Design

```
                                    ┌─────────────┐
                                    │     CDN     │
                                    └──────┬──────┘
                                           │
┌─────────┐     ┌──────────────┐     ┌─────┴─────┐
│ Client  │────▶│ Load Balancer│────▶│ API Server│
└─────────┘     └──────────────┘     └─────┬─────┘
                                           │
                         ┌─────────────────┼─────────────────┐
                         ▼                 ▼                 ▼
                  ┌────────────┐    ┌────────────┐    ┌────────────┐
                  │   Cache    │    │  Database  │    │  Analytics │
                  │  (Redis)   │    │ (Postgres) │    │  (Kafka)   │
                  └────────────┘    └────────────┘    └────────────┘
```

#### Шаг 5: API Design

```yaml
POST /api/v1/shorten
  Request:
    {
      "url": "https://very-long-url.com/path",
      "custom_alias": "my-link",  # optional
      "expires_in": 86400         # optional, seconds
    }
  Response:
    {
      "short_url": "https://short.ly/abc123",
      "expires_at": "2024-12-31T23:59:59Z"
    }

GET /{short_code}
  Response: 301 Redirect
  Headers:
    Location: https://original-url.com

GET /api/v1/stats/{short_code}
  Response:
    {
      "clicks": 12345,
      "created_at": "2024-01-01T00:00:00Z",
      "top_countries": ["US", "RU", "DE"]
    }
```

#### Шаг 6: Detailed Design - Генерация Short Code

```python
import hashlib
import base64

class ShortCodeGenerator:
    """Несколько подходов к генерации короткого кода"""

    # Подход 1: Base62 encoding счётчика
    def generate_from_counter(self, counter: int) -> str:
        """
        + Простой и быстрый
        + Гарантированно уникальный
        - Предсказуемый (можно угадать следующий)
        - Требует централизованный счётчик
        """
        chars = "0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ"
        result = []
        while counter > 0:
            result.append(chars[counter % 62])
            counter //= 62
        return ''.join(reversed(result)).zfill(7)

    # Подход 2: Hash + collision handling
    def generate_from_hash(self, url: str) -> str:
        """
        + Детерминированный (один URL = один код)
        + Распределённый (не нужен центральный счётчик)
        - Возможны коллизии
        - Нужна проверка уникальности
        """
        hash_bytes = hashlib.md5(url.encode()).digest()
        encoded = base64.urlsafe_b64encode(hash_bytes).decode()
        return encoded[:7]

    # Подход 3: Pre-generated keys (Key Generation Service)
    def get_pregenerated_key(self, key_db) -> str:
        """
        + Очень быстрый (просто достаём из пула)
        + Гарантированно уникальный
        - Требует отдельный сервис
        - Нужно заранее генерировать ключи
        """
        return key_db.get_unused_key()
```

#### Шаг 7: Evaluation - Bottlenecks

```markdown
Потенциальные проблемы:

1. Database bottleneck при высокой нагрузке
   Решение: Read replicas + кэширование в Redis

2. Single point of failure
   Решение: Multiple load balancers, DB replication

3. Hot keys в кэше (вирусные ссылки)
   Решение: Local cache + distributed cache

4. Key collision при hash-based подходе
   Решение: Retry с добавлением timestamp
```

---

## Чеклист для System Design интервью

```markdown
□ Уточнил требования (5-10 мин)
  □ Основные use cases
  □ Масштаб (пользователи, данные, запросы)
  □ Нефункциональные требования

□ Сделал оценку (5 мин)
  □ QPS (queries per second)
  □ Объём хранения
  □ Bandwidth

□ Нарисовал высокоуровневую архитектуру (10-15 мин)
  □ Основные компоненты
  □ Data flow
  □ API endpoints

□ Углубился в детали (15-20 мин)
  □ Схема данных
  □ Ключевые алгоритмы
  □ Особенности реализации

□ Обсудил trade-offs (5-10 мин)
  □ Bottlenecks
  □ Single points of failure
  □ Масштабирование
  □ Consistency vs Availability
```

---

## Полезные числа для оценок (Latency Numbers)

```
Операция                          Время
─────────────────────────────────────────
L1 cache reference                0.5 ns
L2 cache reference                7 ns
RAM reference                     100 ns
SSD random read                   150 μs
HDD random read                   10 ms
Network round trip (same DC)      0.5 ms
Network round trip (US to EU)     150 ms

Примерные throughput:
─────────────────────────────────────────
SSD sequential read               500 MB/s
HDD sequential read               100 MB/s
1 Gbps network                    125 MB/s
10 Gbps network                   1.25 GB/s
```

---

## Дополнительные ресурсы

1. **Книги:**
   - "Designing Data-Intensive Applications" - Martin Kleppmann
   - "System Design Interview" - Alex Xu

2. **Практика:**
   - github.com/donnemartin/system-design-primer
   - highscalability.com

3. **Видео:**
   - Канал "System Design Interview" на YouTube
   - InfoQ presentations

---

## Резюме

System Design — это навык, который развивается с практикой. Ключевые моменты:

1. **Всегда начинайте с требований** — не проектируйте в вакууме
2. **Думайте о масштабе** — что работает для 1K пользователей, может не работать для 1M
3. **Знайте компромиссы** — каждое решение имеет плюсы и минусы
4. **Практикуйтесь** — проектируйте известные системы, объясняйте вслух
5. **Изучайте реальные системы** — читайте engineering блоги больших компаний

> "Хороший системный дизайн — это не идеальное решение, а подходящее решение для конкретных требований."
