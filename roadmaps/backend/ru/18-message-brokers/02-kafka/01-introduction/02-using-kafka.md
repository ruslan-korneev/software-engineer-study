# Использование Kafka

[prev: 01-what-is-kafka](./01-what-is-kafka.md) | [next: 03-kafka-myths](./03-kafka-myths.md)

---

## Описание

Apache Kafka используется в самых разных сценариях — от простой передачи сообщений между сервисами до построения сложных систем обработки данных в реальном времени. Понимание типичных use cases поможет определить, подходит ли Kafka для вашей задачи и как лучше спроектировать систему.

## Ключевые концепции

### Основные паттерны использования

**1. Messaging (Обмен сообщениями)**
Kafka как замена традиционным message brokers (RabbitMQ, ActiveMQ) для обмена сообщениями между микросервисами.

**2. Activity Tracking (Отслеживание активности)**
Сбор событий о действиях пользователей: клики, просмотры страниц, поисковые запросы.

**3. Metrics & Monitoring (Метрики и мониторинг)**
Агрегация операционных данных из распределённых приложений для централизованного мониторинга.

**4. Log Aggregation (Агрегация логов)**
Сбор логов со множества серверов в единое хранилище для анализа.

**5. Stream Processing (Потоковая обработка)**
Обработка потоков данных в реальном времени с использованием Kafka Streams или Apache Flink.

**6. Event Sourcing**
Хранение всех изменений состояния как последовательности событий.

**7. Commit Log (Журнал изменений)**
Внешний commit log для распределённых систем, обеспечивающий репликацию данных.

## Примеры

### 1. E-commerce: Обработка заказов

```
┌──────────────┐     ┌─────────────────────────────────────────────────┐
│   Website    │     │                 Kafka Cluster                   │
│  (Producer)  │────▶│                                                 │
└──────────────┘     │  Topic: orders.created                          │
                     │  ┌─────────────────────────────────────────┐    │
┌──────────────┐     │  │ {"orderId": "123", "items": [...]}      │    │
│  Mobile App  │────▶│  │ {"orderId": "124", "items": [...]}      │    │
│  (Producer)  │     │  └─────────────────────────────────────────┘    │
└──────────────┘     └──────────────┬──────────────┬──────────────────┘
                                    │              │
                     ┌──────────────┘              └──────────────┐
                     ▼                                            ▼
              ┌──────────────┐                            ┌──────────────┐
              │   Inventory  │                            │   Payment    │
              │   Service    │                            │   Service    │
              └──────────────┘                            └──────────────┘
                     │                                            │
                     ▼                                            ▼
              ┌─────────────────────────────────────────────────────────┐
              │  Topic: orders.processed                                │
              │  Topic: inventory.updated                               │
              │  Topic: payments.completed                              │
              └─────────────────────────────────────────────────────────┘
```

```python
# Producer: Создание заказа
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

order = {
    "orderId": "ORD-12345",
    "customerId": "CUST-789",
    "items": [
        {"productId": "PROD-001", "quantity": 2, "price": 29.99},
        {"productId": "PROD-002", "quantity": 1, "price": 49.99}
    ],
    "totalAmount": 109.97,
    "createdAt": "2024-01-15T10:30:00Z"
}

# Отправка в топик с ключом для партиционирования по customer
producer.send(
    'orders.created',
    key=order['customerId'].encode('utf-8'),
    value=order
)
producer.flush()
```

```python
# Consumer: Сервис инвентаризации
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'orders.created',
    bootstrap_servers=['localhost:9092'],
    group_id='inventory-service',
    value_deserializer=lambda v: json.loads(v.decode('utf-8'))
)

for message in consumer:
    order = message.value
    print(f"Обработка заказа: {order['orderId']}")

    for item in order['items']:
        # Резервирование товара
        reserve_inventory(item['productId'], item['quantity'])

    # Отправка события об обновлении инвентаря
    send_inventory_updated_event(order['orderId'])
```

### 2. Real-time Analytics: Аналитика в реальном времени

```python
# Сбор кликстрима
from kafka import KafkaProducer
import json
from datetime import datetime

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

def track_click(user_id, page_url, element_id):
    event = {
        "eventType": "click",
        "userId": user_id,
        "pageUrl": page_url,
        "elementId": element_id,
        "timestamp": datetime.utcnow().isoformat(),
        "userAgent": "Mozilla/5.0...",
        "ipAddress": "192.168.1.100"
    }

    producer.send(
        'clickstream',
        key=user_id.encode('utf-8'),
        value=event
    )

# Использование
track_click("user_123", "/products/laptop", "add-to-cart-btn")
```

```java
// Kafka Streams: Подсчёт кликов по страницам
import org.apache.kafka.streams.*;
import org.apache.kafka.streams.kstream.*;

public class ClickAnalytics {
    public static void main(String[] args) {
        Properties props = new Properties();
        props.put(StreamsConfig.APPLICATION_ID_CONFIG, "click-analytics");
        props.put(StreamsConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

        StreamsBuilder builder = new StreamsBuilder();

        // Читаем поток кликов
        KStream<String, ClickEvent> clicks = builder.stream("clickstream");

        // Группируем по URL страницы и считаем за последние 5 минут
        KTable<Windowed<String>, Long> pageCounts = clicks
            .groupBy((key, value) -> value.getPageUrl())
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofMinutes(5)))
            .count();

        // Записываем результаты
        pageCounts.toStream()
            .map((windowedKey, count) -> new KeyValue<>(
                windowedKey.key(),
                new PageStats(windowedKey.key(), count, windowedKey.window().end())
            ))
            .to("page-view-stats");

        KafkaStreams streams = new KafkaStreams(builder.build(), props);
        streams.start();
    }
}
```

### 3. Log Aggregation: Централизованный сбор логов

```
┌────────────┐
│  Server 1  │──┐
│  (nginx)   │  │
└────────────┘  │
                │     ┌─────────────────┐     ┌─────────────────┐
┌────────────┐  ├────▶│  Kafka Cluster  │────▶│  Elasticsearch  │
│  Server 2  │──┤     │  Topic: logs    │     │    + Kibana     │
│  (app)     │  │     └─────────────────┘     └─────────────────┘
└────────────┘  │              │
                │              ▼
┌────────────┐  │     ┌─────────────────┐
│  Server 3  │──┘     │   Log Analyzer  │
│  (db)      │        │   (Alerting)    │
└────────────┘        └─────────────────┘
```

```python
# Filebeat альтернатива на Python
from kafka import KafkaProducer
import json
import tailer  # pip install tailer

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

def tail_and_send(log_file, topic, source_host):
    for line in tailer.follow(open(log_file)):
        log_entry = {
            "message": line,
            "source": log_file,
            "host": source_host,
            "timestamp": datetime.utcnow().isoformat()
        }
        producer.send(topic, value=log_entry)

# Запуск для нескольких файлов
tail_and_send("/var/log/nginx/access.log", "logs.nginx", "web-server-1")
```

### 4. Event Sourcing: Хранение состояния как событий

```python
# Пример банковского счёта с Event Sourcing
from dataclasses import dataclass
from typing import List
from kafka import KafkaProducer, KafkaConsumer
import json

@dataclass
class AccountEvent:
    account_id: str
    event_type: str  # "created", "deposited", "withdrawn"
    amount: float
    timestamp: str

# Producer: Создание событий
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v.__dict__).encode('utf-8')
)

def create_account(account_id: str, initial_deposit: float):
    event = AccountEvent(
        account_id=account_id,
        event_type="created",
        amount=initial_deposit,
        timestamp=datetime.utcnow().isoformat()
    )
    producer.send('account-events', key=account_id.encode(), value=event)

def deposit(account_id: str, amount: float):
    event = AccountEvent(
        account_id=account_id,
        event_type="deposited",
        amount=amount,
        timestamp=datetime.utcnow().isoformat()
    )
    producer.send('account-events', key=account_id.encode(), value=event)

def withdraw(account_id: str, amount: float):
    event = AccountEvent(
        account_id=account_id,
        event_type="withdrawn",
        amount=amount,
        timestamp=datetime.utcnow().isoformat()
    )
    producer.send('account-events', key=account_id.encode(), value=event)

# Consumer: Восстановление состояния из событий
def rebuild_account_state(account_id: str) -> float:
    consumer = KafkaConsumer(
        'account-events',
        bootstrap_servers=['localhost:9092'],
        group_id=None,  # Читаем все сообщения
        auto_offset_reset='earliest',
        value_deserializer=lambda v: json.loads(v.decode('utf-8'))
    )

    balance = 0.0
    for message in consumer:
        event = message.value
        if event['account_id'] == account_id:
            if event['event_type'] == 'created':
                balance = event['amount']
            elif event['event_type'] == 'deposited':
                balance += event['amount']
            elif event['event_type'] == 'withdrawn':
                balance -= event['amount']

    return balance
```

### 5. Микросервисная интеграция

```
┌─────────────────────────────────────────────────────────────────┐
│                     Kafka as Event Bus                          │
│                                                                 │
│   users.created    orders.placed    payments.completed          │
│        │               │                   │                    │
│        ▼               ▼                   ▼                    │
│   ┌─────────┐     ┌─────────┐        ┌─────────┐               │
│   │ User    │     │ Order   │        │ Payment │               │
│   │ Service │     │ Service │        │ Service │               │
│   └─────────┘     └─────────┘        └─────────┘               │
│        │               │                   │                    │
│        └───────────────┼───────────────────┘                    │
│                        ▼                                        │
│                  ┌───────────┐                                  │
│                  │Notification│                                 │
│                  │  Service   │                                 │
│                  └───────────┘                                  │
└─────────────────────────────────────────────────────────────────┘
```

```python
# Notification Service: слушает несколько топиков
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'users.created',
    'orders.placed',
    'payments.completed',
    bootstrap_servers=['localhost:9092'],
    group_id='notification-service',
    value_deserializer=lambda v: json.loads(v.decode('utf-8'))
)

def send_email(to, subject, body):
    print(f"Sending email to {to}: {subject}")

def send_sms(phone, message):
    print(f"Sending SMS to {phone}: {message}")

for message in consumer:
    topic = message.topic
    event = message.value

    if topic == 'users.created':
        send_email(
            event['email'],
            "Добро пожаловать!",
            f"Здравствуйте, {event['name']}!"
        )

    elif topic == 'orders.placed':
        send_email(
            event['customerEmail'],
            f"Заказ #{event['orderId']} оформлен",
            f"Сумма: {event['totalAmount']} руб."
        )

    elif topic == 'payments.completed':
        send_sms(
            event['customerPhone'],
            f"Оплата {event['amount']} руб. прошла успешно"
        )
```

## Best Practices

### Выбор ключа партиционирования

```python
# ПРАВИЛЬНО: связанные события в одной партиции
producer.send('orders', key=order_id.encode(), value=order_data)
producer.send('order-updates', key=order_id.encode(), value=update_data)

# НЕПРАВИЛЬНО: случайное распределение
producer.send('orders', value=order_data)  # Нет ключа = random partition
```

### Идемпотентность обработки

```python
# Используйте идемпотентные операции
def process_order(order_id, operation):
    # Проверка, не обработан ли уже
    if is_already_processed(order_id):
        return

    try:
        perform_operation(order_id, operation)
        mark_as_processed(order_id)
    except Exception as e:
        # При ошибке сообщение будет перечитано
        raise
```

### Error Handling и Dead Letter Queue

```python
from kafka import KafkaProducer, KafkaConsumer

dlq_producer = KafkaProducer(bootstrap_servers=['localhost:9092'])

def process_with_dlq(message):
    try:
        process_message(message)
    except Exception as e:
        # Отправляем в Dead Letter Queue
        dlq_producer.send(
            'orders.dlq',
            key=message.key,
            value=message.value,
            headers=[
                ('error', str(e).encode()),
                ('original-topic', message.topic.encode()),
                ('original-partition', str(message.partition).encode())
            ]
        )
```

### Выбор правильного топика

| Сценарий | Рекомендация |
|----------|--------------|
| Высокая нагрузка, много партиций | Один топик на тип события |
| Разные SLA для разных событий | Отдельные топики |
| Связанные события одной сущности | Один топик с ключом = entity_id |
| Разные retention requirements | Отдельные топики |

### Мониторинг

```python
# Отслеживание consumer lag
from kafka import KafkaAdminClient
from kafka.admin import ConsumerGroupDescription

admin = KafkaAdminClient(bootstrap_servers=['localhost:9092'])

def get_consumer_lag(group_id, topic):
    # Получаем текущие offsets группы
    group_offsets = admin.list_consumer_group_offsets(group_id)

    # Получаем конечные offsets топика
    from kafka import KafkaConsumer
    consumer = KafkaConsumer(bootstrap_servers=['localhost:9092'])
    end_offsets = consumer.end_offsets(consumer.partitions_for_topic(topic))

    total_lag = 0
    for tp, offset in group_offsets.items():
        if tp.topic == topic:
            total_lag += end_offsets[tp] - offset.offset

    return total_lag
```

## Когда использовать Kafka

### Подходит для:
- Высоконагруженные системы (>10K сообщений/сек)
- Системы с множеством потребителей одних данных
- Потоковая обработка и аналитика
- Event-driven архитектуры
- Долгосрочное хранение событий

### Не подходит для:
- Простые request-response паттерны (используйте HTTP)
- Маленькие объёмы данных (overhead не оправдан)
- Сложная маршрутизация сообщений (используйте RabbitMQ)
- Транзакции между несколькими топиками (ограниченная поддержка)

## Краткое резюме

Kafka универсальна и применяется в самых разных сценариях: от простого обмена сообщениями до сложных систем потоковой обработки данных. Ключ к успешному использованию — правильный выбор паттерна для конкретной задачи, грамотное проектирование топиков и ключей партиционирования, а также обеспечение идемпотентности обработки.

---

[prev: 01-what-is-kafka](./01-what-is-kafka.md) | [next: 03-kafka-myths](./03-kafka-myths.md)
