# Потоковая обработка и терминология

## Описание

Потоковая обработка данных (Stream Processing) — это парадигма обработки данных, при которой данные обрабатываются непрерывно по мере их поступления, а не пакетами (batch). Apache Kafka и его экосистема (Kafka Streams, ksqlDB) предоставляют мощные инструменты для реализации потоковой обработки.

## Ключевые концепции

### Batch vs Stream Processing

```
┌─────────────────────────────────────────────────────────────────────┐
│                    BATCH PROCESSING                                  │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [Данные] ──► [Накопление] ──► [Обработка] ──► [Результат]          │
│              (часы/дни)        (пакетом)       (отложенный)         │
│                                                                      │
│  Примеры: ETL, отчеты, аналитика за период                          │
└─────────────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────────────┐
│                    STREAM PROCESSING                                 │
├─────────────────────────────────────────────────────────────────────┤
│                                                                      │
│  [Событие] ──► [Обработка] ──► [Результат]                          │
│  (real-time)   (мгновенно)     (немедленный)                        │
│                                                                      │
│  [Событие] ──► [Обработка] ──► [Результат]                          │
│                                                                      │
│  Примеры: мониторинг, fraud detection, real-time analytics          │
└─────────────────────────────────────────────────────────────────────┘
```

### Основные термины

#### 1. Event (Событие)

**Событие** — это неизменяемая запись о факте, который произошел в определенный момент времени.

```python
# Пример события
event = {
    "event_type": "order.created",
    "event_id": "evt_123456",
    "timestamp": "2024-01-15T10:30:00Z",
    "data": {
        "order_id": "ord_789",
        "user_id": "user_001",
        "amount": 99.99,
        "items": ["item_1", "item_2"]
    }
}
```

**Характеристики события:**
- Неизменяемость (immutable)
- Временная метка (timestamp)
- Ключ для партиционирования
- Payload (полезная нагрузка)

#### 2. Stream (Поток)

**Поток** — это непрерывная, неограниченная последовательность событий.

```
Stream "orders":

t1      t2      t3      t4      t5      ...
│       │       │       │       │
▼       ▼       ▼       ▼       ▼
┌───┐   ┌───┐   ┌───┐   ┌───┐   ┌───┐
│ E1│───│ E2│───│ E3│───│ E4│───│ E5│───► ∞
└───┘   └───┘   └───┘   └───┘   └───┘

E = Event
```

**Типы потоков:**
- **KStream** — поток событий (каждое событие независимо)
- **KTable** — таблица изменений (последнее значение для каждого ключа)
- **GlobalKTable** — глобальная таблица (реплицирована на все инстансы)

#### 3. Producer и Consumer

```
┌──────────────┐                              ┌──────────────┐
│   Producer   │                              │   Consumer   │
│              │                              │              │
│  Источник    │────────►  Kafka  ────────►  │  Обработчик  │
│  событий     │          Cluster            │  событий     │
│              │                              │              │
└──────────────┘                              └──────────────┘
     ▲                                              │
     │                                              │
     └──────────────────────────────────────────────┘
                    (опционально)
```

#### 4. Topology (Топология)

**Топология** — это граф обработки данных, состоящий из источников, процессоров и приемников.

```
            ┌─────────────────────────────────────────┐
            │              TOPOLOGY                    │
            │                                          │
            │  ┌────────┐                              │
Source ────►│  │ Filter │                              │
            │  └────┬───┘                              │
            │       │                                  │
            │       ▼                                  │
            │  ┌────────┐    ┌─────────┐              │
            │  │  Map   │───►│ GroupBy │              │
            │  └────────┘    └────┬────┘              │
            │                     │                    │
            │                     ▼                    │
            │               ┌──────────┐              │
            │               │ Aggregate│───────► Sink │
            │               └──────────┘              │
            └─────────────────────────────────────────┘
```

#### 5. State Store (Хранилище состояния)

**State Store** — локальное хранилище для stateful операций (агрегации, joins).

```
┌────────────────────────────────────────────────┐
│                 Kafka Streams App               │
│                                                 │
│  ┌─────────────┐        ┌──────────────────┐   │
│  │  Processor  │◄──────►│   State Store    │   │
│  │  (aggregate)│        │   (RocksDB)      │   │
│  └─────────────┘        └──────────────────┘   │
│                                │                │
│                                ▼                │
│                         ┌────────────┐         │
│                         │ Changelog  │         │
│                         │   Topic    │         │
│                         └────────────┘         │
└────────────────────────────────────────────────┘
```

### Время в потоковой обработке

#### Event Time vs Processing Time

```
Event Time (Время события):
Когда событие ПРОИЗОШЛО в реальном мире

Processing Time (Время обработки):
Когда событие ОБРАБОТАНО системой

                     Event Time        Processing Time
                          │                   │
Event Created ────────────┼───────────────────┤
                          ▼                   ▼
                     10:00:00            10:00:05
                                          (задержка 5 сек)
```

```python
# Пример работы с временем
event = {
    "event_time": "2024-01-15T10:00:00Z",  # Время события
    "ingestion_time": "2024-01-15T10:00:02Z",  # Время попадания в Kafka
    "processing_time": None  # Заполняется при обработке
}
```

#### Windowing (Окна)

**Окна** позволяют группировать события по времени.

```
Tumbling Window (Скользящее окно без перекрытия):

Timeline: ─────────────────────────────────────────►
          │    5 min    │    5 min    │    5 min    │
          ├─────────────┼─────────────┼─────────────┤
Events:   │ E1  E2  E3  │  E4  E5     │  E6  E7  E8 │
          └─────────────┴─────────────┴─────────────┘
          Window 1       Window 2       Window 3


Hopping Window (Скользящее окно с перекрытием):

Timeline: ─────────────────────────────────────────►
          │──── 10 min ────│
          │    │──── 10 min ────│
          │    │    │──── 10 min ────│
          ├────┼────┼────┼────┼────┼────┤
          0    5    10   15   20   25   30  (minutes)

          Window 1: [0-10]
          Window 2: [5-15]
          Window 3: [10-20]


Session Window (Сессионное окно):

Timeline: ─────────────────────────────────────────►
          E1 E2 E3      (gap > 5min)       E4 E5
          ├──────────┤                 ├──────────┤
          │ Session1 │                 │ Session2 │
          └──────────┘                 └──────────┘
```

#### Watermarks (Водяные знаки)

**Watermark** — маркер, указывающий, что все события до определенного времени уже получены.

```
Event Stream:   E1(t=10)  E2(t=12)  E3(t=9)  E4(t=15)
                    │         │        │         │
                    ▼         ▼        ▼         ▼
Watermark:      ────10───────12───────12────────15────►

После Watermark = 15:
- Можно закрыть окно [0-10]
- События с t < 15 считаются "поздними"
```

### Stateless vs Stateful обработка

#### Stateless операции

Не требуют хранения состояния между событиями:

```python
# Примеры stateless операций

# Filter — фильтрация событий
stream.filter(lambda k, v: v['amount'] > 100)

# Map — преобразование
stream.map(lambda k, v: (k, v['amount'] * 1.1))

# FlatMap — разворачивание
stream.flatMap(lambda k, v: [(k, item) for item in v['items']])

# Branch — разветвление
stream.branch(
    lambda k, v: v['type'] == 'A',
    lambda k, v: v['type'] == 'B'
)
```

#### Stateful операции

Требуют хранения состояния:

```python
# Примеры stateful операций

# Aggregate — агрегация
stream.groupByKey().aggregate(
    initializer=0,
    aggregator=lambda k, v, agg: agg + v['amount']
)

# Count — подсчет
stream.groupByKey().count()

# Reduce — свертка
stream.groupByKey().reduce(
    lambda v1, v2: v1 + v2
)

# Join — соединение потоков
stream1.join(stream2, joiner=lambda v1, v2: {...})
```

## Примеры кода

### Kafka Streams (Java)

```java
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;
import org.apache.kafka.common.serialization.Serdes;

public class StreamProcessingExample {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "stream-processor");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(StreamsConfig.DEFAULT_KEY_SERDE_CLASS_CONFIG, Serdes.String().getClass());
        props.put(StreamsConfig.DEFAULT_VALUE_SERDE_CLASS_CONFIG, Serdes.String().getClass());

        StreamsBuilder builder = new StreamsBuilder();

        // Чтение потока заказов
        KStream<String, String> orders = builder.stream("orders");

        // Stateless: фильтрация и преобразование
        KStream<String, String> highValueOrders = orders
            .filter((key, value) -> {
                Order order = parseOrder(value);
                return order.getAmount() > 100;
            })
            .mapValues(value -> {
                Order order = parseOrder(value);
                order.setStatus("HIGH_VALUE");
                return order.toJson();
            });

        // Запись результата
        highValueOrders.to("high-value-orders");

        // Stateful: подсчет заказов по пользователю
        KTable<String, Long> orderCounts = orders
            .groupBy((key, value) -> parseOrder(value).getUserId())
            .count();

        orderCounts.toStream().to("order-counts");

        // Stateful: сумма заказов за 1 час
        KTable<Windowed<String>, Double> hourlyTotals = orders
            .groupBy((key, value) -> parseOrder(value).getUserId())
            .windowedBy(TimeWindows.of(Duration.ofHours(1)))
            .aggregate(
                () -> 0.0,
                (key, value, aggregate) -> aggregate + parseOrder(value).getAmount(),
                Materialized.with(Serdes.String(), Serdes.Double())
            );

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();

        Runtime.getRuntime().addShutdownHook(new Thread(streams::close));
    }
}
```

### Faust (Python)

Faust — это Python-библиотека для потоковой обработки, вдохновленная Kafka Streams.

```python
import faust
from datetime import timedelta

# Создание приложения
app = faust.App(
    'order-processor',
    broker='kafka://localhost:9092',
    store='rocksdb://'
)

# Определение модели события
class Order(faust.Record):
    order_id: str
    user_id: str
    amount: float
    items: list

# Топики
orders_topic = app.topic('orders', value_type=Order)
high_value_topic = app.topic('high-value-orders', value_type=Order)

# Таблица для агрегации
order_counts = app.Table('order_counts', default=int)
user_totals = app.Table('user_totals', default=float)


# Stateless обработка
@app.agent(orders_topic)
async def process_orders(orders):
    async for order in orders:
        # Фильтрация и отправка
        if order.amount > 100:
            await high_value_topic.send(value=order)

        # Обновление счетчиков
        order_counts[order.user_id] += 1
        user_totals[order.user_id] += order.amount

        print(f'Обработан заказ: {order.order_id}')
        print(f'Всего заказов пользователя: {order_counts[order.user_id]}')


# Windowed агрегация
class HourlyStats(faust.Record):
    user_id: str
    total: float
    count: int

hourly_stats = app.Table(
    'hourly_stats',
    default=lambda: HourlyStats(user_id='', total=0.0, count=0)
).tumbling(
    timedelta(hours=1),
    expires=timedelta(hours=24)
)

@app.agent(orders_topic)
async def hourly_aggregation(orders):
    async for order in orders:
        stats = hourly_stats[order.user_id]
        stats.user_id = order.user_id
        stats.total += order.amount
        stats.count += 1
        hourly_stats[order.user_id] = stats


# Веб-интерфейс для просмотра состояния
@app.page('/stats/{user_id}/')
async def get_stats(web, request, user_id):
    return web.json({
        'user_id': user_id,
        'order_count': order_counts.get(user_id, 0),
        'total_spent': user_totals.get(user_id, 0.0)
    })


if __name__ == '__main__':
    app.main()
```

### ksqlDB

ksqlDB — это SQL-движок для потоковой обработки данных в Kafka.

```sql
-- Создание потока из топика
CREATE STREAM orders (
    order_id VARCHAR KEY,
    user_id VARCHAR,
    amount DOUBLE,
    items ARRAY<VARCHAR>,
    created_at VARCHAR
) WITH (
    KAFKA_TOPIC = 'orders',
    VALUE_FORMAT = 'JSON'
);

-- Создание производного потока (фильтрация)
CREATE STREAM high_value_orders AS
SELECT *
FROM orders
WHERE amount > 100
EMIT CHANGES;

-- Создание таблицы агрегации
CREATE TABLE order_counts AS
SELECT
    user_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_amount
FROM orders
GROUP BY user_id
EMIT CHANGES;

-- Оконная агрегация
CREATE TABLE hourly_stats AS
SELECT
    user_id,
    COUNT(*) AS order_count,
    SUM(amount) AS total_amount,
    WINDOWSTART AS window_start,
    WINDOWEND AS window_end
FROM orders
WINDOW TUMBLING (SIZE 1 HOUR)
GROUP BY user_id
EMIT CHANGES;

-- Join потоков
CREATE STREAM enriched_orders AS
SELECT
    o.order_id,
    o.user_id,
    o.amount,
    u.name AS user_name,
    u.email AS user_email
FROM orders o
INNER JOIN users u
    ON o.user_id = u.user_id
EMIT CHANGES;

-- Запросы к таблицам
SELECT * FROM order_counts WHERE user_id = 'user_001';

-- Push-запрос (непрерывный)
SELECT * FROM orders EMIT CHANGES;
```

## Паттерны потоковой обработки

### 1. Event Sourcing

```
┌─────────────────────────────────────────────────────────────┐
│                    EVENT SOURCING                            │
│                                                              │
│  Event Store (Kafka Topic):                                  │
│  ┌──────────────────────────────────────────────────────┐   │
│  │ Created │ Updated │ Updated │ Deleted │ Restored │...│   │
│  └──────────────────────────────────────────────────────┘   │
│       │         │         │         │          │            │
│       ▼         ▼         ▼         ▼          ▼            │
│  ┌──────────────────────────────────────────────────────┐   │
│  │          AGGREGATE (текущее состояние)                │   │
│  │          = сумма всех событий                         │   │
│  └──────────────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────────────┘
```

```python
# Event Sourcing пример
class OrderAggregate:
    def __init__(self):
        self.events = []
        self.state = {}

    def apply(self, event):
        self.events.append(event)

        if event['type'] == 'OrderCreated':
            self.state = {
                'id': event['order_id'],
                'user_id': event['user_id'],
                'items': event['items'],
                'status': 'created'
            }
        elif event['type'] == 'OrderPaid':
            self.state['status'] = 'paid'
            self.state['paid_at'] = event['timestamp']
        elif event['type'] == 'OrderShipped':
            self.state['status'] = 'shipped'
            self.state['tracking'] = event['tracking_number']

    def get_state(self):
        return self.state
```

### 2. CQRS (Command Query Responsibility Segregation)

```
┌─────────────────────────────────────────────────────────────┐
│                         CQRS                                 │
│                                                              │
│  Commands (записи)           Queries (чтения)               │
│       │                           ▲                          │
│       ▼                           │                          │
│  ┌─────────┐                 ┌─────────┐                    │
│  │ Command │                 │  Read   │                    │
│  │ Service │                 │ Service │                    │
│  └────┬────┘                 └────┬────┘                    │
│       │                           ▲                          │
│       ▼                           │                          │
│  ┌─────────┐    Kafka       ┌─────────┐                    │
│  │ Write   │ ────────────►  │  Read   │                    │
│  │ Store   │   (events)     │  Store  │                    │
│  └─────────┘                └─────────┘                    │
│                                                              │
└─────────────────────────────────────────────────────────────┘
```

### 3. Saga Pattern

```
┌─────────────────────────────────────────────────────────────┐
│                    SAGA (Choreography)                       │
│                                                              │
│  Order Service    Payment Service    Inventory Service       │
│       │                 │                  │                 │
│       │ OrderCreated    │                  │                 │
│       ├────────────────►│                  │                 │
│       │                 │ PaymentProcessed │                 │
│       │                 ├─────────────────►│                 │
│       │                 │                  │ InventoryReserved│
│       │◄────────────────┴──────────────────┤                 │
│       │                                    │                 │
│       │ OrderCompleted                     │                 │
│       ▼                                    ▼                 │
│                                                              │
│  Компенсация при ошибке:                                    │
│       │ InventoryFailed                    │                 │
│       │◄───────────────────────────────────┤                 │
│       │ PaymentRefunded                    │                 │
│       │◄──────────────────┤                │                 │
│       │                   │                │                 │
└─────────────────────────────────────────────────────────────┘
```

## Best Practices

### 1. Проектирование событий

```python
# Хорошая структура события
good_event = {
    "event_id": "evt_uuid",           # Уникальный ID
    "event_type": "order.created",    # Тип события
    "event_version": "1.0",           # Версия схемы
    "timestamp": "2024-01-15T10:00:00Z",  # Event time
    "source": "order-service",        # Источник
    "correlation_id": "req_123",      # ID для трейсинга
    "data": {
        # Бизнес-данные
    },
    "metadata": {
        # Дополнительная информация
    }
}
```

### 2. Обработка late events

```python
# Настройка допустимой задержки
from datetime import timedelta

# Faust: grace period для окон
table = app.Table('stats').tumbling(
    timedelta(hours=1),
    expires=timedelta(hours=24)
).relative_to_stream()  # Использовать event time
```

### 3. Exactly-once обработка

```python
# Включение транзакций
producer_config = {
    'bootstrap.servers': 'localhost:9092',
    'transactional.id': 'my-transactional-id',
    'enable.idempotence': True
}

consumer_config = {
    'bootstrap.servers': 'localhost:9092',
    'group.id': 'my-group',
    'isolation.level': 'read_committed',
    'enable.auto.commit': False
}
```

### 4. Мониторинг потоковой обработки

```
Ключевые метрики:

1. Consumer Lag — отставание обработки
2. Processing Rate — скорость обработки
3. Error Rate — частота ошибок
4. State Store Size — размер хранилища состояния
5. Rebalance Count — количество перебалансировок
```

## Дополнительные ресурсы

- [Kafka Streams Documentation](https://kafka.apache.org/documentation/streams/)
- [ksqlDB Documentation](https://docs.ksqldb.io/)
- [Faust Documentation](https://faust.readthedocs.io/)
- [Designing Event-Driven Systems](https://www.confluent.io/designing-event-driven-systems/)
- [Stream Processing with Apache Kafka](https://www.confluent.io/blog/stream-processing-apache-kafka/)
