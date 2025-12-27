# Performance vs Scalability (Производительность vs Масштабируемость)

## Введение

Производительность и масштабируемость — два фундаментальных понятия в системном дизайне, которые часто путают. Хотя они связаны, это разные характеристики системы, и понимание их различий критически важно для принятия правильных архитектурных решений.

---

## 1. Производительность (Performance)

### Определение

**Производительность** — это способность системы выполнять работу за определённое время. Она измеряется в терминах скорости и эффективности обработки запросов.

### Ключевые метрики производительности

#### Время отклика (Response Time / Latency)
Время от отправки запроса до получения ответа.

```
Время отклика = Время обработки + Время сети + Время ожидания в очереди
```

**Типичные значения:**
- Отличное: < 100 мс
- Хорошее: 100-300 мс
- Приемлемое: 300-1000 мс
- Плохое: > 1000 мс

#### Пропускная способность (Throughput)
Количество операций, которое система может обработать за единицу времени.

```
Пропускная способность = Количество запросов / Единица времени
```

**Примеры измерений:**
- RPS (Requests Per Second) — запросов в секунду
- TPS (Transactions Per Second) — транзакций в секунду
- QPS (Queries Per Second) — запросов к БД в секунду

### Пример: производительный сервер

```python
# Высокопроизводительный код
import asyncio
import aiohttp

async def fetch_data(session, url):
    """Асинхронный запрос — не блокирует поток"""
    async with session.get(url) as response:
        return await response.json()

async def process_requests(urls):
    """Параллельная обработка запросов"""
    async with aiohttp.ClientSession() as session:
        tasks = [fetch_data(session, url) for url in urls]
        return await asyncio.gather(*tasks)

# 100 запросов выполняются параллельно за ~200ms вместо 20 секунд
```

### Что влияет на производительность

1. **Алгоритмическая сложность** — O(n) vs O(n²)
2. **Использование кэширования** — Redis, Memcached
3. **Оптимизация запросов к БД** — индексы, правильные JOIN
4. **Асинхронность** — неблокирующий I/O
5. **Эффективное использование памяти** — избежание утечек
6. **Сетевые оптимизации** — HTTP/2, сжатие, CDN

---

## 2. Масштабируемость (Scalability)

### Определение

**Масштабируемость** — это способность системы увеличивать свою производительность пропорционально добавлению ресурсов при росте нагрузки.

### Ключевые характеристики

```
Идеальная масштабируемость:
  2x ресурсов → 2x производительности

Реальная масштабируемость:
  2x ресурсов → 1.5-1.9x производительности (из-за накладных расходов)
```

### Закон Амдала

Теоретический предел ускорения системы ограничен долей последовательного кода:

```
Speedup = 1 / (S + P/N)

где:
S = доля последовательного кода (не параллелизуемого)
P = доля параллельного кода
N = количество процессоров
```

**Пример:**
- Если 10% кода не может быть распараллелено (S=0.1)
- Даже с бесконечным количеством серверов максимальное ускорение = 10x

### Типы масштабируемости

| Тип | Описание | Пример |
|-----|----------|--------|
| Масштабируемость нагрузки | Способность обрабатывать больше запросов | Добавление серверов |
| Масштабируемость данных | Способность хранить больше данных | Шардирование БД |
| Географическая | Способность работать в разных регионах | Multi-region deployment |
| Административная | Способность управлять большей системой | Микросервисы |

---

## 3. Разница между Performance и Scalability

### Ключевое различие

```
Performance: Насколько быстро система работает СЕЙЧАС при текущей нагрузке
Scalability: Насколько хорошо система АДАПТИРУЕТСЯ к увеличению нагрузки
```

### Аналогия

**Производительность** — это максимальная скорость одного автомобиля.
**Масштабируемость** — это способность добавлять полосы на шоссе при увеличении трафика.

### Сравнительная таблица

| Аспект | Performance | Scalability |
|--------|-------------|-------------|
| Фокус | Скорость одной операции | Обработка роста нагрузки |
| Вопрос | "Как быстро?" | "Как вырасти?" |
| Метрики | Latency, Throughput | Линейность роста, Capacity |
| Решения | Оптимизация кода, кэши | Архитектура, распределение |
| Когда важна | Всегда | При росте бизнеса |

### Практические примеры различий

#### Пример 1: Высокая производительность, низкая масштабируемость

```python
# Монолитное приложение на одном мощном сервере
# Производительность: отлично (быстрый сервер)
# Масштабируемость: плохо (нельзя добавить серверы)

class MonolithicApp:
    def __init__(self):
        self.cache = {}  # Локальный кэш в памяти
        self.db = LocalDatabase()  # Локальная БД

    def process_request(self, request):
        # Быстро работает на одном сервере
        # Но нельзя распределить между серверами
        if request.id in self.cache:
            return self.cache[request.id]

        result = self.db.heavy_computation(request)
        self.cache[request.id] = result
        return result
```

#### Пример 2: Низкая производительность, высокая масштабируемость

```python
# Распределённая система с избыточной коммуникацией
# Производительность: средняя (много сетевых вызовов)
# Масштабируемость: отлично (легко добавить узлы)

class DistributedApp:
    def process_request(self, request):
        # Каждый запрос делает много сетевых вызовов
        user = self.user_service.get_user(request.user_id)  # ~50ms
        permissions = self.auth_service.check(user)          # ~30ms
        data = self.data_service.fetch(request.data_id)      # ~40ms
        result = self.compute_service.process(data)          # ~60ms

        # Итого: ~180ms на запрос (медленно)
        # Но можно масштабировать каждый сервис независимо
        return result
```

#### Пример 3: Баланс производительности и масштабируемости

```python
# Хорошо спроектированная система
class BalancedApp:
    def __init__(self):
        self.redis = Redis(cluster=True)  # Распределённый кэш
        self.db = PostgresCluster()       # Кластер БД

    async def process_request(self, request):
        # Сначала проверяем кэш (быстро)
        cached = await self.redis.get(request.cache_key)
        if cached:
            return cached  # ~5ms

        # Параллельные запросы к сервисам (оптимизация)
        user, data = await asyncio.gather(
            self.get_user(request.user_id),
            self.get_data(request.data_id)
        )

        result = self.compute(user, data)
        await self.redis.set(request.cache_key, result, ex=3600)

        return result  # ~100ms для cache miss
```

---

## 4. Вертикальное масштабирование (Scale Up)

### Определение

**Вертикальное масштабирование** — увеличение мощности одного сервера путём добавления ресурсов (CPU, RAM, SSD).

```
До:  4 CPU, 16 GB RAM, 256 GB SSD
После: 32 CPU, 256 GB RAM, 2 TB NVMe
```

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| Простота | Не требует изменения архитектуры |
| Консистентность | Все данные на одном сервере |
| Нет сетевых задержек | Всё в одной машине |
| Транзакции | ACID гарантии проще обеспечить |
| Отладка | Проще диагностировать проблемы |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| Лимит | Физический предел мощности сервера |
| Стоимость | Экспоненциальный рост цены |
| Single Point of Failure | Падение сервера = падение системы |
| Downtime | Обновление требует остановки |
| Нет гео-распределения | Один дата-центр |

### Когда использовать

```
✅ Хорошо для:
- Начальный этап стартапа
- Базы данных с высокими требованиями к консистентности
- Приложения с интенсивными вычислениями
- Когда простота важнее гибкости

❌ Плохо для:
- Высокодоступных систем (99.99%)
- Глобально распределённых приложений
- Систем с непредсказуемым ростом нагрузки
```

### Пример: вертикальное масштабирование PostgreSQL

```yaml
# Конфигурация до масштабирования
postgresql:
  server:
    cpu: 4
    memory: 16GB
    storage: 256GB SSD
  config:
    shared_buffers: 4GB
    effective_cache_size: 12GB
    max_connections: 100

# После вертикального масштабирования
postgresql:
  server:
    cpu: 32
    memory: 256GB
    storage: 2TB NVMe
  config:
    shared_buffers: 64GB
    effective_cache_size: 192GB
    max_connections: 500
    work_mem: 256MB
```

---

## 5. Горизонтальное масштабирование (Scale Out)

### Определение

**Горизонтальное масштабирование** — добавление большего количества серверов для распределения нагрузки.

```
До:    [Server 1]
После: [Server 1] [Server 2] [Server 3] [Server N]
             ↓
        [Load Balancer]
```

### Преимущества

| Преимущество | Описание |
|--------------|----------|
| Теоретически безлимитно | Можно добавлять серверы |
| Отказоустойчивость | Падение одного сервера не критично |
| Гео-распределение | Серверы в разных регионах |
| Cost-effective | Много дешёвых серверов дешевле одного мощного |
| Zero-downtime | Обновление по одному серверу |

### Недостатки

| Недостаток | Описание |
|------------|----------|
| Сложность | Требует распределённой архитектуры |
| Консистентность | Трудно поддерживать синхронизацию |
| Сетевые задержки | Коммуникация между серверами |
| Stateless требование | Приложение должно быть stateless |
| Operational overhead | Больше серверов = больше управления |

### Паттерны горизонтального масштабирования

#### 1. Stateless приложения

```python
# Плохо: состояние в памяти сервера
class StatefulApp:
    sessions = {}  # Сессии в памяти

    def handle_request(self, session_id, request):
        # Если запрос попадёт на другой сервер — сессия потеряется
        if session_id not in self.sessions:
            raise SessionNotFound()
        return self.process(self.sessions[session_id], request)

# Хорошо: состояние во внешнем хранилище
class StatelessApp:
    def __init__(self):
        self.redis = Redis()  # Внешнее хранилище сессий

    def handle_request(self, session_id, request):
        # Любой сервер может обработать запрос
        session = self.redis.get(f"session:{session_id}")
        if not session:
            raise SessionNotFound()
        return self.process(session, request)
```

#### 2. Шардирование данных

```python
class ShardedDatabase:
    def __init__(self, shard_count=4):
        self.shards = [DatabaseShard(i) for i in range(shard_count)]

    def get_shard(self, user_id: int) -> DatabaseShard:
        """Consistent hashing для определения шарда"""
        shard_index = hash(user_id) % len(self.shards)
        return self.shards[shard_index]

    def get_user(self, user_id: int):
        shard = self.get_shard(user_id)
        return shard.query(f"SELECT * FROM users WHERE id = {user_id}")

    def save_user(self, user_id: int, data: dict):
        shard = self.get_shard(user_id)
        shard.execute(f"INSERT INTO users ...")
```

#### 3. Load Balancing

```nginx
# Nginx конфигурация для горизонтального масштабирования
upstream backend_servers {
    least_conn;  # Алгоритм балансировки

    server backend1.example.com:8080 weight=3;
    server backend2.example.com:8080 weight=2;
    server backend3.example.com:8080 weight=1;

    # Health checks
    server backend4.example.com:8080 backup;
}

server {
    listen 80;

    location / {
        proxy_pass http://backend_servers;
        proxy_next_upstream error timeout http_502;
    }
}
```

### Когда использовать

```
✅ Хорошо для:
- Веб-приложений с высоким трафиком
- Микросервисной архитектуры
- Глобально распределённых систем
- Систем с требованием высокой доступности

❌ Плохо для:
- Приложений с сильной связью между данными
- Систем с требованием строгой консистентности
- Когда сложность архитектуры не оправдана нагрузкой
```

---

## 6. Когда оптимизировать Performance vs Scalability

### Матрица принятия решений

```
                    Низкая нагрузка    Высокая нагрузка
                   ┌─────────────────┬─────────────────┐
Медленный отклик   │  Performance    │  Performance    │
                   │  optimization   │  + Scalability  │
                   ├─────────────────┼─────────────────┤
Быстрый отклик     │  Ничего не      │  Scalability    │
                   │  делать (пока)  │  planning       │
                   └─────────────────┴─────────────────┘
```

### Сначала Performance (оптимизируй код)

**Признаки проблемы производительности:**
- Один запрос выполняется медленно даже при низкой нагрузке
- Высокое использование CPU/памяти на одном запросе
- N+1 проблемы в запросах к БД
- Неоптимальные алгоритмы

**Решения:**

```python
# Проблема: N+1 запросы
def get_users_with_orders():
    users = User.query.all()  # 1 запрос
    for user in users:
        orders = Order.query.filter_by(user_id=user.id).all()  # N запросов
    return users

# Решение: Eager loading
def get_users_with_orders_optimized():
    users = User.query.options(
        joinedload(User.orders)
    ).all()  # 1 запрос с JOIN
    return users
```

```python
# Проблема: медленный алгоритм O(n²)
def find_duplicates_slow(items):
    duplicates = []
    for i, item in enumerate(items):
        for j, other in enumerate(items):
            if i != j and item == other and item not in duplicates:
                duplicates.append(item)
    return duplicates

# Решение: оптимальный алгоритм O(n)
def find_duplicates_fast(items):
    seen = set()
    duplicates = set()
    for item in items:
        if item in seen:
            duplicates.add(item)
        seen.add(item)
    return list(duplicates)
```

### Потом Scalability (масштабируй архитектуру)

**Признаки проблемы масштабируемости:**
- Производительность падает при увеличении нагрузки
- Один сервер не справляется, но код оптимален
- Время отклика растёт линейно с числом пользователей
- База данных становится bottleneck

**Решения:**

```python
# До: монолитный подход
class MonolithicService:
    def process_order(self, order):
        # Всё в одном месте
        self.validate(order)
        self.charge_payment(order)
        self.update_inventory(order)
        self.send_notification(order)
        return order

# После: распределённый подход
class OrderService:
    async def process_order(self, order):
        # Валидация синхронно (быстро)
        self.validate(order)

        # Остальное через очереди (масштабируемо)
        await self.queue.publish("payments", order)
        await self.queue.publish("inventory", order)
        await self.queue.publish("notifications", order)

        return order
```

### Правило "3x"

```
Проектируй систему для 3x текущей нагрузки
Реализуй для 10x текущей нагрузки
```

- Если ожидается 1000 RPS → архитектура должна выдержать 3000 RPS
- Иметь план масштабирования до 10000 RPS

---

## 7. Практические примеры и кейсы

### Кейс 1: Интернет-магазин

**Проблема:** Сайт тормозит в Чёрную пятницу

```
Обычный день: 100 RPS, latency 200ms
Чёрная пятница: 10,000 RPS, latency 5000ms+ (таймауты)
```

**Анализ:**
1. Проверяем производительность одного запроса → 200ms (нормально)
2. Проверяем под нагрузкой → деградация
3. Диагноз: проблема масштабируемости

**Решение:**

```python
# Архитектура для масштабируемости

# 1. Кэширование каталога товаров
class ProductCatalog:
    def __init__(self):
        self.cache = Redis(cluster=True)

    async def get_product(self, product_id):
        # Cache-aside pattern
        cached = await self.cache.get(f"product:{product_id}")
        if cached:
            return json.loads(cached)

        product = await self.db.get_product(product_id)
        await self.cache.setex(
            f"product:{product_id}",
            3600,  # 1 час
            json.dumps(product)
        )
        return product

# 2. Очередь для заказов
class OrderProcessor:
    async def submit_order(self, order):
        # Быстро принимаем заказ
        order_id = await self.save_pending_order(order)

        # Обработка в фоне
        await self.queue.publish("orders", {
            "order_id": order_id,
            "order": order
        })

        return {"order_id": order_id, "status": "processing"}

# 3. Auto-scaling в Kubernetes
```

```yaml
# kubernetes/hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: shop-api
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: shop-api
  minReplicas: 3
  maxReplicas: 50
  metrics:
    - type: Resource
      resource:
        name: cpu
        target:
          type: Utilization
          averageUtilization: 70
    - type: Pods
      pods:
        metric:
          name: requests_per_second
        target:
          type: AverageValue
          averageValue: "100"
```

### Кейс 2: Аналитический сервис

**Проблема:** Отчёты генерируются 30 минут

```
Запрос отчёта → ожидание 30 минут → timeout
```

**Анализ:**
1. Один запрос выполняется 30 минут
2. Даже один пользователь ждёт долго
3. Диагноз: проблема производительности

**Решение:**

```python
# Оптимизация производительности

# 1. Предагрегация данных
class AnalyticsService:
    async def generate_report_slow(self, params):
        # Плохо: агрегация в реальном времени
        raw_data = await self.db.query("""
            SELECT date, SUM(amount), COUNT(*)
            FROM transactions
            WHERE date BETWEEN %s AND %s
            GROUP BY date
        """, params)  # 30 минут для миллиардов записей
        return self.format_report(raw_data)

    async def generate_report_fast(self, params):
        # Хорошо: чтение преагрегированных данных
        aggregated = await self.db.query("""
            SELECT date, total_amount, transaction_count
            FROM daily_aggregates
            WHERE date BETWEEN %s AND %s
        """, params)  # Миллисекунды
        return self.format_report(aggregated)

# 2. Фоновая агрегация
class AggregationWorker:
    @scheduler.scheduled_job('cron', hour=1)
    async def aggregate_daily(self):
        yesterday = date.today() - timedelta(days=1)

        # Агрегируем ночью, когда нагрузка низкая
        await self.db.execute("""
            INSERT INTO daily_aggregates (date, total_amount, transaction_count)
            SELECT
                date,
                SUM(amount),
                COUNT(*)
            FROM transactions
            WHERE date = %s
            GROUP BY date
        """, yesterday)
```

### Кейс 3: Чат-приложение

**Проблема:** Сообщения доставляются с задержкой при большом количестве пользователей

**Решение: комбинация Performance + Scalability**

```python
# Performance: WebSockets вместо polling
class ChatService:
    def __init__(self):
        self.redis_pubsub = Redis()
        self.connections = {}  # user_id -> WebSocket

    async def connect(self, user_id, websocket):
        self.connections[user_id] = websocket

        # Подписка на канал пользователя
        pubsub = self.redis_pubsub.pubsub()
        await pubsub.subscribe(f"user:{user_id}")

        # Слушаем входящие сообщения
        async for message in pubsub.listen():
            if message['type'] == 'message':
                await websocket.send(message['data'])

    async def send_message(self, from_user, to_user, message):
        # Публикуем в канал получателя
        await self.redis_pubsub.publish(
            f"user:{to_user}",
            json.dumps({
                "from": from_user,
                "message": message,
                "timestamp": time.time()
            })
        )
```

```yaml
# Scalability: распределённые WebSocket серверы
# docker-compose.yml
services:
  chat-server-1:
    image: chat-service
    environment:
      - REDIS_URL=redis://redis:6379

  chat-server-2:
    image: chat-service
    environment:
      - REDIS_URL=redis://redis:6379

  chat-server-3:
    image: chat-service
    environment:
      - REDIS_URL=redis://redis:6379

  nginx:
    image: nginx
    # Sticky sessions для WebSocket
    # Все серверы общаются через Redis PubSub

  redis:
    image: redis:alpine
```

---

## 8. Метрики для измерения

### Метрики производительности

| Метрика | Описание | Целевое значение |
|---------|----------|------------------|
| P50 Latency | Медиана времени отклика | < 100ms |
| P95 Latency | 95-й перцентиль | < 500ms |
| P99 Latency | 99-й перцентиль | < 1000ms |
| Throughput | Запросов в секунду | Зависит от бизнеса |
| Error Rate | Процент ошибок | < 0.1% |
| Apdex | Application Performance Index | > 0.9 |

```python
# Сбор метрик с помощью Prometheus
from prometheus_client import Histogram, Counter

request_latency = Histogram(
    'http_request_duration_seconds',
    'Request latency in seconds',
    ['method', 'endpoint', 'status'],
    buckets=[.005, .01, .025, .05, .1, .25, .5, 1, 2.5, 5, 10]
)

request_count = Counter(
    'http_requests_total',
    'Total request count',
    ['method', 'endpoint', 'status']
)

async def middleware(request, call_next):
    start_time = time.time()
    response = await call_next(request)

    duration = time.time() - start_time

    request_latency.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).observe(duration)

    request_count.labels(
        method=request.method,
        endpoint=request.url.path,
        status=response.status_code
    ).inc()

    return response
```

### Метрики масштабируемости

| Метрика | Описание | Как измерять |
|---------|----------|--------------|
| Throughput vs Instances | Линейность роста | График RPS/серверы |
| Cost per Request | Стоимость обработки | $/1M запросов |
| Capacity Headroom | Запас мощности | (Max - Current) / Max |
| Scale-up Time | Время добавления ресурса | Секунды до готовности |
| Efficiency | Эффективность использования | CPU/Memory utilization |

```python
# Тестирование масштабируемости
import asyncio
import httpx
import time

async def load_test(url: str, concurrent: int, total: int):
    """Нагрузочное тестирование"""
    results = []

    async def make_request():
        start = time.time()
        async with httpx.AsyncClient() as client:
            response = await client.get(url)
        duration = time.time() - start
        return {
            "status": response.status_code,
            "duration": duration
        }

    start_time = time.time()

    # Выполняем запросы с ограничением конкурентности
    semaphore = asyncio.Semaphore(concurrent)

    async def limited_request():
        async with semaphore:
            return await make_request()

    tasks = [limited_request() for _ in range(total)]
    results = await asyncio.gather(*tasks)

    total_time = time.time() - start_time

    # Анализ результатов
    durations = [r["duration"] for r in results]
    successful = sum(1 for r in results if r["status"] == 200)

    return {
        "total_requests": total,
        "successful": successful,
        "total_time": total_time,
        "rps": total / total_time,
        "avg_latency": sum(durations) / len(durations),
        "p50": sorted(durations)[len(durations) // 2],
        "p95": sorted(durations)[int(len(durations) * 0.95)],
        "p99": sorted(durations)[int(len(durations) * 0.99)],
    }

# Тест масштабируемости: запускаем с разным количеством серверов
# 1 сервер: 500 RPS
# 2 сервера: 950 RPS (1.9x - хорошо)
# 4 сервера: 1800 RPS (3.6x - хорошо)
# 8 серверов: 3200 RPS (6.4x - начинается насыщение)
```

### Dashboards для мониторинга

```yaml
# Grafana dashboard для Performance vs Scalability

# Panel 1: Request Latency Distribution
- type: heatmap
  query: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket[5m]))

# Panel 2: Throughput vs Instance Count
- type: graph
  queries:
    - sum(rate(http_requests_total[1m])) as "Total RPS"
    - count(up{job="api"}) as "Instance Count"

# Panel 3: Scalability Efficiency
- type: gauge
  query: |
    sum(rate(http_requests_total[5m])) / count(up{job="api"})
  # RPS на один инстанс — должен быть стабильным при масштабировании

# Panel 4: Resource Utilization
- type: graph
  queries:
    - avg(container_cpu_usage_seconds_total) by (pod)
    - avg(container_memory_usage_bytes) by (pod)
```

---

## Резюме

### Ключевые выводы

1. **Performance** — скорость работы системы при текущей нагрузке
2. **Scalability** — способность системы расти с нагрузкой
3. Это разные проблемы с разными решениями
4. Сначала оптимизируй код (Performance), потом масштабируй архитектуру (Scalability)
5. Вертикальное масштабирование проще, но ограничено
6. Горизонтальное масштабирование сложнее, но практически безлимитно
7. Измеряй обе характеристики с помощью метрик

### Чек-лист для backend-разработчика

```
Performance оптимизация:
[ ] Профилирование кода (cProfile, py-spy)
[ ] Оптимизация запросов к БД (EXPLAIN ANALYZE)
[ ] Добавление индексов
[ ] Кэширование (Redis, in-memory)
[ ] Асинхронный I/O
[ ] Устранение N+1 проблем

Scalability проектирование:
[ ] Stateless приложение
[ ] Внешнее хранение сессий
[ ] Готовность к горизонтальному масштабированию
[ ] Очереди для тяжёлых операций
[ ] Шардирование данных (при необходимости)
[ ] Health checks для load balancer
[ ] Graceful shutdown
```
