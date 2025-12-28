# Доступность vs Согласованность (Availability vs Consistency)

[prev: 03-latency-vs-throughput](./03-latency-vs-throughput.md) | [next: 05-cap-theorem](./05-cap-theorem.md)

---

## Введение

В распределённых системах два ключевых свойства часто находятся в конфликте: **доступность** (Availability) и **согласованность** (Consistency). Понимание этого компромисса критически важно для проектирования надёжных систем.

---

## 1. Доступность (Availability)

### Определение

**Доступность** — это процент времени, в течение которого система способна обрабатывать запросы и возвращать корректные ответы.

```
Availability = (Uptime / Total Time) × 100%
```

Высокая доступность означает, что система **всегда готова** обслуживать пользователей, даже при частичных отказах компонентов.

### Характеристики высокодоступной системы

- Отвечает на каждый запрос (успешно или с ошибкой)
- Не имеет единой точки отказа (Single Point of Failure)
- Продолжает работать при выходе из строя отдельных узлов
- Использует репликацию и резервирование

---

## 2. Расчёт доступности — "Девятки"

### Таблица уровней доступности

| Уровень | Процент | Допустимый простой в год | Допустимый простой в месяц |
|---------|---------|--------------------------|----------------------------|
| 99% (две девятки) | 99.0% | 3.65 дня | 7.31 часа |
| 99.9% (три девятки) | 99.9% | 8.77 часа | 43.83 минуты |
| 99.95% | 99.95% | 4.38 часа | 21.92 минуты |
| 99.99% (четыре девятки) | 99.99% | 52.6 минуты | 4.38 минуты |
| 99.999% (пять девяток) | 99.999% | 5.26 минуты | 26.3 секунды |
| 99.9999% (шесть девяток) | 99.9999% | 31.56 секунды | 2.63 секунды |

### Расчёт доступности

```python
def calculate_downtime(availability_percent: float, period_days: int = 365) -> dict:
    """
    Рассчитывает допустимое время простоя для заданного уровня доступности.

    Args:
        availability_percent: Процент доступности (например, 99.99)
        period_days: Период в днях (365 для года, 30 для месяца)

    Returns:
        Словарь с временем простоя в разных единицах
    """
    total_minutes = period_days * 24 * 60
    downtime_percent = 100 - availability_percent
    downtime_minutes = total_minutes * (downtime_percent / 100)

    return {
        "availability": f"{availability_percent}%",
        "period_days": period_days,
        "downtime_minutes": round(downtime_minutes, 2),
        "downtime_hours": round(downtime_minutes / 60, 2),
        "downtime_days": round(downtime_minutes / 60 / 24, 2)
    }

# Примеры
print(calculate_downtime(99.9, 365))    # ~8.77 часов в год
print(calculate_downtime(99.99, 365))   # ~52.6 минут в год
print(calculate_downtime(99.999, 365))  # ~5.26 минут в год
```

### Расчёт доступности последовательных компонентов

Если компоненты расположены **последовательно** (каждый должен работать для работы системы):

```
System Availability = A₁ × A₂ × A₃ × ... × Aₙ
```

**Пример:** Веб-сервер (99.9%) → База данных (99.9%) → Кэш (99.9%)

```python
web_server = 0.999
database = 0.999
cache = 0.999

system_availability = web_server * database * cache
print(f"Доступность системы: {system_availability * 100:.3f}%")  # 99.7%
```

**Важно:** Последовательное соединение **снижает** общую доступность!

### Расчёт доступности параллельных компонентов

Если компоненты расположены **параллельно** (резервирование):

```
System Availability = 1 - (1 - A₁) × (1 - A₂) × ... × (1 - Aₙ)
```

**Пример:** Два сервера в режиме Active-Active, каждый с доступностью 99%

```python
server1 = 0.99
server2 = 0.99

# Вероятность отказа обоих
both_fail = (1 - server1) * (1 - server2)
system_availability = 1 - both_fail

print(f"Доступность с резервированием: {system_availability * 100:.4f}%")  # 99.99%
```

**Важно:** Параллельное соединение **повышает** общую доступность!

---

## 3. Согласованность (Consistency)

### Определение

**Согласованность** — это гарантия того, что все узлы распределённой системы видят **одинаковые данные** в один и тот же момент времени.

В согласованной системе после завершения операции записи все последующие операции чтения вернут обновлённое значение.

### Типы согласованности

#### Strong Consistency (Строгая согласованность)

После записи **любое** чтение вернёт новое значение. Все узлы синхронизированы.

```
Клиент A: write(x = 5)
          ↓ (запись завершена)
Клиент B: read(x) → всегда 5
```

**Примеры:** Традиционные РСУБД, Google Spanner, CockroachDB

#### Eventual Consistency (Согласованность в конечном счёте)

После записи данные **со временем** распространятся на все узлы. Временно разные узлы могут возвращать разные значения.

```
Клиент A: write(x = 5)
          ↓ (запись завершена)
Клиент B: read(x) → может быть старое значение
          ... через некоторое время ...
Клиент B: read(x) → 5
```

**Примеры:** DNS, Amazon DynamoDB, Cassandra (по умолчанию)

#### Causal Consistency (Причинная согласованность)

Операции, связанные причинно-следственной связью, видны в правильном порядке.

```
Клиент A: write(x = 1)
Клиент A: write(y = x + 1)  // y = 2

Клиент B: read(y) = 2 → read(x) гарантированно вернёт 1
```

#### Read-your-writes Consistency

Клиент всегда видит свои собственные записи.

```
Клиент A: write(x = 5)
Клиент A: read(x) → гарантированно 5
Клиент B: read(x) → может быть старое значение
```

### Сравнение моделей согласованности

| Модель | Гарантии | Latency | Доступность |
|--------|----------|---------|-------------|
| Strong | Все видят одно и то же | Высокая | Низкая |
| Eventual | Со временем синхронизируются | Низкая | Высокая |
| Causal | Порядок причинно связанных операций | Средняя | Средняя |
| Read-your-writes | Клиент видит свои записи | Низкая | Высокая |

---

## 4. Почему нельзя иметь и то, и другое — CAP теорема

### Суть CAP теоремы

**CAP теорема** (Eric Brewer, 2000) утверждает, что распределённая система может одновременно гарантировать только **два из трёх** свойств:

- **C (Consistency)** — Согласованность
- **A (Availability)** — Доступность
- **P (Partition Tolerance)** — Устойчивость к разделению сети

### Почему нужно выбирать?

В реальных распределённых системах **сетевые разделения неизбежны** (P всегда требуется). Поэтому реальный выбор — между C и A.

```
Сценарий: Сетевое разделение между узлами

    Node A ←──×──→ Node B

    Клиент хочет записать данные. Что делать?

    Вариант 1 (CP): Отказать в записи до восстановления связи
                    → Согласованность сохранена
                    → Доступность потеряна

    Вариант 2 (AP): Принять запись на одном узле
                    → Доступность сохранена
                    → Согласованность потеряна (узлы рассинхронизированы)
```

### Визуализация выбора

```
                    Consistency
                        /\
                       /  \
                      /    \
                     /  CP  \
                    /        \
                   /          \
                  /────────────\
                 /              \
                /       CA       \
               /    (невозможно   \
              /    в распределённой\
             /        системе)     \
            /                       \
           /────────────────────────\
          /                          \
         /            AP              \
        /                              \
       ──────────────────────────────────
    Availability              Partition Tolerance
```

### Примеры классификации систем

| Система | Тип | Объяснение |
|---------|-----|------------|
| PostgreSQL (single node) | CA | Нет распределения → нет разделений |
| MongoDB (с write concern majority) | CP | При разделении недоступен для записи |
| Cassandra (по умолчанию) | AP | Принимает записи даже при разделении |
| ZooKeeper | CP | Кворум для согласованности |
| DynamoDB | AP (настраиваемо) | Eventual consistency по умолчанию |

---

## 5. Системы, где важнее доступность

### DNS (Domain Name System)

**Почему доступность критична:**
- DNS — фундамент работы интернета
- Недоступность DNS = невозможность открыть любой сайт
- Лучше вернуть немного устаревшие данные, чем ничего

```
Сценарий: Обновление DNS записи

До: example.com → 1.2.3.4
После: example.com → 5.6.7.8

Время распространения: от минут до 48 часов (TTL)

Пользователь A: Видит 5.6.7.8 (новый IP)
Пользователь B: Видит 1.2.3.4 (старый IP) - ОК, сайт работает
```

### Кэши (Redis, Memcached)

**Почему доступность важнее:**
- Кэш — это оптимизация, не источник правды
- Устаревшие данные в кэше допустимы
- Недоступность кэша → все запросы идут в БД → перегрузка

```python
class CacheWithFallback:
    def __init__(self, cache_client, db_client):
        self.cache = cache_client
        self.db = db_client

    def get(self, key: str):
        try:
            # Пробуем кэш (AP система)
            value = self.cache.get(key)
            if value:
                return value  # Может быть немного устаревшим
        except CacheUnavailableError:
            pass  # Кэш недоступен — не проблема

        # Fallback на БД
        value = self.db.query(key)

        # Пытаемся обновить кэш (best effort)
        try:
            self.cache.set(key, value, ttl=300)
        except:
            pass

        return value
```

### CDN (Content Delivery Network)

**Почему доступность важнее:**
- Пользователи ожидают мгновенную загрузку контента
- Слегка устаревшая картинка лучше, чем ошибка
- Edge-серверы могут быть временно рассинхронизированы

### Социальные сети (лента новостей)

**Почему доступность важнее:**
- Пользователь не заметит, если лайк появится с задержкой
- Невозможность открыть ленту — критическая проблема
- Eventual consistency для лайков, комментариев, просмотров

```python
# Facebook-подобный подход
class NewsFeedService:
    def add_like(self, post_id: str, user_id: str):
        # Запись в локальный узел (быстро, доступно)
        self.local_db.add_like(post_id, user_id)

        # Асинхронная репликация (eventually consistent)
        self.replication_queue.push({
            "action": "add_like",
            "post_id": post_id,
            "user_id": user_id,
            "timestamp": time.time()
        })

        return {"status": "success"}  # Мгновенный ответ
```

### E-commerce: Просмотр каталога

**Почему доступность важнее:**
- Клиент должен видеть товары
- Цена может быть неактуальной на секунды
- Итоговая проверка при оформлении заказа

---

## 6. Системы, где важнее согласованность

### Банковские транзакции

**Почему согласованность критична:**
- Деньги не должны появляться или исчезать
- Двойное списание недопустимо
- Лучше отказать в операции, чем потерять деньги

```python
class BankTransferService:
    def transfer(self, from_account: str, to_account: str, amount: Decimal):
        # Начинаем распределённую транзакцию
        with self.distributed_transaction() as tx:
            # Блокируем оба счёта
            from_balance = tx.lock_and_read(from_account)
            to_balance = tx.lock_and_read(to_account)

            if from_balance < amount:
                raise InsufficientFundsError()

            # Атомарное обновление
            tx.write(from_account, from_balance - amount)
            tx.write(to_account, to_balance + amount)

            # Commit с подтверждением от всех узлов
            tx.commit()  # Может быть медленно, но гарантированно согласовано
```

### Системы бронирования

**Почему согласованность критична:**
- Овербукинг недопустим
- Одно место → один билет
- Двойное бронирование = убытки и репутационный ущерб

```python
class BookingSystem:
    def book_seat(self, flight_id: str, seat_id: str, passenger_id: str):
        # Строгая согласованность через блокировку
        with self.db.transaction(isolation_level="SERIALIZABLE"):
            seat = self.db.query(
                "SELECT * FROM seats WHERE flight_id = %s AND seat_id = %s FOR UPDATE",
                (flight_id, seat_id)
            )

            if seat.status != "available":
                raise SeatUnavailableError("Место уже забронировано")

            self.db.execute(
                "UPDATE seats SET status = 'booked', passenger_id = %s WHERE id = %s",
                (passenger_id, seat.id)
            )

            return {"status": "booked", "confirmation": generate_confirmation()}
```

### Инвентаризация (складской учёт)

**Почему согласованность критична:**
- Продажа товара, которого нет на складе = проблема
- Точный подсчёт остатков для закупок
- Финансовая отчётность требует точных данных

```python
class InventoryService:
    def reserve_item(self, item_id: str, quantity: int) -> bool:
        # Используем оптимистичную блокировку с версионированием
        while True:
            item = self.db.query(
                "SELECT id, quantity, version FROM inventory WHERE id = %s",
                (item_id,)
            )

            if item.quantity < quantity:
                return False  # Недостаточно товара

            # Атомарное обновление с проверкой версии
            result = self.db.execute(
                """
                UPDATE inventory
                SET quantity = quantity - %s, version = version + 1
                WHERE id = %s AND version = %s
                """,
                (quantity, item_id, item.version)
            )

            if result.rows_affected == 1:
                return True  # Успешно зарезервировано

            # Версия изменилась — повторяем попытку
            continue
```

### Системы голосования / выборы

**Почему согласованность критична:**
- Один человек = один голос
- Результаты должны быть точными
- Любое расхождение ставит под сомнение легитимность

### Медицинские записи

**Почему согласованность критична:**
- Неправильные данные могут стоить жизни
- История болезни должна быть точной
- Аллергии, противопоказания — критическая информация

---

## 7. Стратегии балансировки

### Стратегия 1: Разделение по типу данных

Разные данные имеют разные требования — используем разные подходы.

```python
class HybridStorage:
    def __init__(self):
        # CP система для критических данных
        self.postgres = PostgresClient(
            sync_replication=True,
            isolation="SERIALIZABLE"
        )

        # AP система для некритических данных
        self.cassandra = CassandraClient(
            consistency_level="ONE",  # Быстро, доступно
            replication_factor=3
        )

    def save_order(self, order: Order):
        # Критические данные — в PostgreSQL (CP)
        self.postgres.save({
            "order_id": order.id,
            "amount": order.amount,
            "payment_status": order.payment_status,
        })

        # Аналитические данные — в Cassandra (AP)
        self.cassandra.save({
            "order_id": order.id,
            "user_agent": order.user_agent,
            "referrer": order.referrer,
            "timestamp": order.timestamp,
        })
```

### Стратегия 2: Tunable Consistency

Настраиваемая согласованность в зависимости от операции.

```python
# Cassandra пример
class CassandraService:
    def read_user_profile(self, user_id: str):
        # Профиль может быть немного устаревшим
        return self.session.execute(
            "SELECT * FROM users WHERE id = %s",
            (user_id,),
            consistency_level=ConsistencyLevel.ONE  # Быстро
        )

    def update_password(self, user_id: str, new_password_hash: str):
        # Критическая операция — нужна согласованность
        return self.session.execute(
            "UPDATE users SET password_hash = %s WHERE id = %s",
            (new_password_hash, user_id),
            consistency_level=ConsistencyLevel.QUORUM  # Согласованно
        )

    def update_last_login(self, user_id: str):
        # Некритические данные
        return self.session.execute(
            "UPDATE users SET last_login = %s WHERE id = %s",
            (datetime.now(), user_id),
            consistency_level=ConsistencyLevel.ANY  # Максимально быстро
        )
```

### Стратегия 3: CQRS (Command Query Responsibility Segregation)

Разделение записи и чтения с разными гарантиями.

```python
class CQRSOrderService:
    def __init__(self):
        # Write model — строгая согласованность
        self.write_db = PostgresClient()

        # Read model — eventual consistency, оптимизирован для чтения
        self.read_db = ElasticsearchClient()

        # Синхронизация через события
        self.event_bus = KafkaClient()

    def create_order(self, order: Order):
        # Запись в основную БД (согласованно)
        self.write_db.save(order)

        # Публикация события для обновления read model
        self.event_bus.publish("order_created", order.to_dict())

        return order.id

    def search_orders(self, query: str):
        # Чтение из оптимизированного хранилища (быстро, доступно)
        # Данные могут отставать на секунды
        return self.read_db.search(query)
```

### Стратегия 4: Saga Pattern для распределённых транзакций

Вместо строгой согласованности — компенсирующие транзакции.

```python
class OrderSaga:
    def __init__(self):
        self.steps = [
            ("reserve_inventory", self.reserve_inventory, self.release_inventory),
            ("charge_payment", self.charge_payment, self.refund_payment),
            ("create_shipment", self.create_shipment, self.cancel_shipment),
        ]

    def execute(self, order: Order):
        completed_steps = []

        try:
            for step_name, execute_fn, compensate_fn in self.steps:
                execute_fn(order)
                completed_steps.append((step_name, compensate_fn))

            return {"status": "success", "order_id": order.id}

        except Exception as e:
            # Откат выполненных шагов в обратном порядке
            for step_name, compensate_fn in reversed(completed_steps):
                try:
                    compensate_fn(order)
                except Exception as comp_error:
                    # Логируем для ручного вмешательства
                    self.log_compensation_failure(order, step_name, comp_error)

            return {"status": "failed", "error": str(e)}
```

### Стратегия 5: Graceful Degradation

Автоматическое снижение гарантий при проблемах.

```python
class ResilientService:
    def __init__(self):
        self.primary_db = PostgresClient()  # CP
        self.cache = RedisClient()  # AP
        self.circuit_breaker = CircuitBreaker()

    def get_user(self, user_id: str):
        # Попытка 1: Согласованное чтение из primary
        if self.circuit_breaker.is_closed():
            try:
                user = self.primary_db.query(user_id)
                self.cache.set(f"user:{user_id}", user)
                return user
            except DatabaseUnavailableError:
                self.circuit_breaker.record_failure()

        # Fallback: Чтение из кэша (может быть устаревшим)
        cached_user = self.cache.get(f"user:{user_id}")
        if cached_user:
            # Помечаем, что данные могут быть устаревшими
            cached_user["_stale"] = True
            return cached_user

        raise ServiceUnavailableError("Cannot retrieve user data")
```

---

## 8. SLA и SLO — определение требований к доступности

### Определения

**SLA (Service Level Agreement)** — юридически обязывающее соглашение с клиентом, определяющее уровень сервиса и компенсации за его нарушение.

**SLO (Service Level Objective)** — внутренняя цель по уровню сервиса, обычно строже SLA.

**SLI (Service Level Indicator)** — конкретная метрика, измеряющая уровень сервиса.

### Иерархия

```
SLI (что измеряем)
  ↓
SLO (какой уровень хотим достичь)
  ↓
SLA (что обещаем клиентам и что будет при нарушении)
```

### Примеры SLI

```python
class SLIMetrics:
    def availability_sli(self):
        """
        Процент успешных запросов.
        """
        successful = self.metrics.count("http_status < 500")
        total = self.metrics.count("all_requests")
        return (successful / total) * 100

    def latency_sli(self, percentile: int = 99):
        """
        Задержка на заданном перцентиле.
        """
        return self.metrics.percentile("response_time_ms", percentile)

    def error_rate_sli(self):
        """
        Процент запросов с ошибками.
        """
        errors = self.metrics.count("http_status >= 500")
        total = self.metrics.count("all_requests")
        return (errors / total) * 100

    def throughput_sli(self):
        """
        Запросов в секунду.
        """
        return self.metrics.rate("requests", interval="1s")
```

### Примеры SLO

```yaml
# Пример SLO документа
service: payment-api

objectives:
  availability:
    target: 99.95%
    measurement_window: 30 days
    calculation: successful_requests / total_requests

  latency:
    p50_target: 100ms
    p99_target: 500ms
    p999_target: 1000ms
    measurement_window: 30 days

  error_rate:
    target: < 0.1%
    measurement_window: 7 days

error_budget:
  monthly_budget_minutes: 21.92  # (100% - 99.95%) * 30 * 24 * 60
  alert_threshold: 50%  # Алерт когда потрачено 50% бюджета
```

### Пример SLA

```markdown
# Service Level Agreement — Payment Processing API

## Availability Commitment
- Monthly Uptime Percentage: 99.9%
- Excluded: Scheduled maintenance (with 48h notice)

## Service Credits
| Monthly Uptime | Credit |
|----------------|--------|
| < 99.9% but >= 99.0% | 10% of monthly fee |
| < 99.0% but >= 95.0% | 25% of monthly fee |
| < 95.0% | 50% of monthly fee |

## Latency Commitment
- 99th percentile response time: < 500ms
- Measurement: Every 5 minutes from 3 geographic regions

## Exclusions
- Force majeure events
- Customer-caused issues
- Third-party service outages
```

### Error Budget — бюджет ошибок

**Error Budget** — допустимое количество ошибок/простоя за период.

```python
class ErrorBudgetTracker:
    def __init__(self, slo_target: float, measurement_window_days: int):
        self.slo_target = slo_target  # например, 0.999 (99.9%)
        self.window_days = measurement_window_days

    def calculate_budget(self) -> dict:
        """
        Рассчитывает бюджет ошибок.
        """
        total_minutes = self.window_days * 24 * 60
        allowed_downtime = total_minutes * (1 - self.slo_target)

        return {
            "total_budget_minutes": allowed_downtime,
            "total_budget_percent": (1 - self.slo_target) * 100
        }

    def remaining_budget(self, incidents: list) -> dict:
        """
        Рассчитывает оставшийся бюджет после инцидентов.
        """
        budget = self.calculate_budget()

        used_minutes = sum(inc["duration_minutes"] for inc in incidents)
        remaining = budget["total_budget_minutes"] - used_minutes

        return {
            "total_budget": budget["total_budget_minutes"],
            "used_minutes": used_minutes,
            "remaining_minutes": remaining,
            "remaining_percent": (remaining / budget["total_budget_minutes"]) * 100,
            "is_exhausted": remaining <= 0
        }

# Пример использования
tracker = ErrorBudgetTracker(slo_target=0.999, measurement_window_days=30)

incidents = [
    {"name": "Database failover", "duration_minutes": 5},
    {"name": "Network issue", "duration_minutes": 12},
    {"name": "Deploy rollback", "duration_minutes": 3},
]

budget_status = tracker.remaining_budget(incidents)
print(f"Использовано: {budget_status['used_minutes']} мин")
print(f"Осталось: {budget_status['remaining_minutes']:.1f} мин")
print(f"Осталось: {budget_status['remaining_percent']:.1f}%")
```

### Как определить требования к доступности

```python
def determine_slo_requirements(system_info: dict) -> dict:
    """
    Помогает определить подходящий SLO на основе характеристик системы.
    """
    recommendations = {
        "availability_target": None,
        "latency_p99_target": None,
        "rationale": []
    }

    # Критичность для бизнеса
    if system_info["business_critical"]:
        recommendations["availability_target"] = "99.99%"
        recommendations["rationale"].append(
            "Критическая система требует высокой доступности"
        )
    elif system_info["customer_facing"]:
        recommendations["availability_target"] = "99.9%"
        recommendations["rationale"].append(
            "Клиентский сервис требует хорошей доступности"
        )
    else:
        recommendations["availability_target"] = "99.5%"
        recommendations["rationale"].append(
            "Внутренний сервис может иметь умеренную доступность"
        )

    # Тип операций
    if system_info["financial_transactions"]:
        recommendations["consistency"] = "strong"
        recommendations["rationale"].append(
            "Финансовые операции требуют строгой согласованности"
        )

    # Интерактивность
    if system_info["real_time_interaction"]:
        recommendations["latency_p99_target"] = "100ms"
        recommendations["rationale"].append(
            "Интерактивные системы требуют низкой задержки"
        )
    elif system_info["api_service"]:
        recommendations["latency_p99_target"] = "500ms"
    else:
        recommendations["latency_p99_target"] = "2000ms"

    return recommendations

# Пример
payment_system = {
    "business_critical": True,
    "customer_facing": True,
    "financial_transactions": True,
    "real_time_interaction": True,
    "api_service": True
}

print(determine_slo_requirements(payment_system))
# → availability: 99.99%, latency: 100ms, consistency: strong
```

---

## Практические рекомендации

### Чек-лист при проектировании

1. **Определите критичность данных**
   - Финансовые операции → Strong Consistency
   - Аналитика, логи → Eventual Consistency
   - Сессии пользователей → Read-your-writes

2. **Определите требования к доступности**
   - Критический сервис (99.99%+) → Active-Active, Multi-region
   - Стандартный сервис (99.9%) → Репликация + Failover
   - Внутренний сервис (99.5%) → Базовая репликация

3. **Используйте правильные инструменты**
   - CP: PostgreSQL, MongoDB (majority write), ZooKeeper
   - AP: Cassandra, DynamoDB, Redis Cluster

4. **Планируйте отказы**
   - Что произойдёт при отказе БД?
   - Что произойдёт при сетевом разделении?
   - Какой уровень деградации допустим?

### Антипаттерны

```python
# ПЛОХО: Одинаковый подход ко всем данным
class BadService:
    def save_anything(self, data):
        # Всегда строгая согласованность — медленно и недоступно
        self.db.execute_with_strong_consistency(data)

# ХОРОШО: Разный подход к разным данным
class GoodService:
    def save_payment(self, payment):
        # Финансы — строгая согласованность
        self.postgres.save(payment, sync_replication=True)

    def save_analytics(self, event):
        # Аналитика — eventual consistency
        self.cassandra.save(event, consistency="ONE")
```

---

## Заключение

Выбор между доступностью и согласованностью — это не бинарное решение, а спектр возможностей:

1. **Анализируйте требования** — разные части системы имеют разные потребности
2. **Используйте правильные инструменты** — не все данные требуют строгой согласованности
3. **Определяйте SLO осознанно** — на основе реальных бизнес-требований
4. **Планируйте отказы** — система должна gracefully degradate
5. **Мониторьте и измеряйте** — используйте SLI для отслеживания реального состояния

Современные системы часто используют **гибридный подход**: строгая согласованность для критических операций и eventual consistency для остального.
