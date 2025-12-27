# Экскурсия по Kafka

## Описание

Эта экскурсия познакомит вас с основными компонентами и концепциями Apache Kafka. Мы рассмотрим архитектуру системы, ключевые абстракции и то, как все части работают вместе для обеспечения надежной передачи сообщений.

## Общая архитектура Kafka

```
┌─────────────────────────────────────────────────────────────────────────┐
│                           KAFKA ECOSYSTEM                                │
│                                                                          │
│  ┌─────────────┐                                      ┌─────────────┐   │
│  │  Producer   │──────┐                      ┌───────│  Consumer   │   │
│  │  (App 1)    │      │                      │       │  (App 1)    │   │
│  └─────────────┘      │                      │       └─────────────┘   │
│                       │                      │                          │
│  ┌─────────────┐      │    ┌──────────┐      │       ┌─────────────┐   │
│  │  Producer   │──────┼───│  Kafka   │─────┼───────│  Consumer   │   │
│  │  (App 2)    │      │    │  Cluster │      │       │  (App 2)    │   │
│  └─────────────┘      │    └──────────┘      │       └─────────────┘   │
│                       │          │           │                          │
│  ┌─────────────┐      │          │           │       ┌─────────────┐   │
│  │  Producer   │──────┘          │           └───────│  Consumer   │   │
│  │  (App 3)    │                 │                   │  (App 3)    │   │
│  └─────────────┘                 │                   └─────────────┘   │
│                                  │                                      │
│                         ┌────────┴────────┐                            │
│                         │  ZooKeeper /    │                            │
│                         │  KRaft          │                            │
│                         └─────────────────┘                            │
└─────────────────────────────────────────────────────────────────────────┘
```

## Ключевые концепции

### 1. Topics (Топики)

**Топик** — это категория или канал для сообщений. Это логическое имя для потока данных.

```
┌─────────────────────────────────────────────────────────┐
│                      Topic: "orders"                     │
├─────────────────────────────────────────────────────────┤
│  Partition 0: [msg0][msg1][msg2][msg3][msg4]...         │
│  Partition 1: [msg0][msg1][msg2][msg3]...               │
│  Partition 2: [msg0][msg1][msg2][msg3][msg4][msg5]...   │
└─────────────────────────────────────────────────────────┘
```

```bash
# Создание топика
kafka-topics.sh --create \
  --topic orders \
  --bootstrap-server localhost:9092 \
  --partitions 3 \
  --replication-factor 2

# Просмотр топиков
kafka-topics.sh --list --bootstrap-server localhost:9092

# Детальная информация о топике
kafka-topics.sh --describe --topic orders --bootstrap-server localhost:9092
```

### 2. Partitions (Партиции)

**Партиция** — это упорядоченная, неизменяемая последовательность сообщений. Каждое сообщение в партиции имеет уникальный offset.

```
Partition 0:
┌─────┬─────┬─────┬─────┬─────┬─────┐
│  0  │  1  │  2  │  3  │  4  │  5  │  ← Offset
├─────┼─────┼─────┼─────┼─────┼─────┤
│ msg │ msg │ msg │ msg │ msg │ msg │  ← Messages
└─────┴─────┴─────┴─────┴─────┴─────┘
                              ↑
                      Новые сообщения
                    добавляются сюда
```

**Зачем нужны партиции:**
- **Параллелизм** — разные consumers могут читать разные партиции одновременно
- **Масштабирование** — данные распределяются между брокерами
- **Порядок** — сообщения с одинаковым ключом попадают в одну партицию

### 3. Producers (Производители)

**Producer** — клиент, который публикует сообщения в топик.

```python
from kafka import KafkaProducer
import json

producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    key_serializer=lambda k: k.encode('utf-8') if k else None,
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Отправка без ключа (round-robin по партициям)
producer.send('orders', value={'order_id': 1, 'amount': 100})

# Отправка с ключом (все заказы одного пользователя в одну партицию)
producer.send('orders', key='user_123', value={'order_id': 2, 'amount': 200})

producer.flush()
producer.close()
```

### 4. Consumers (Потребители)

**Consumer** — клиент, который читает сообщения из топика.

```python
from kafka import KafkaConsumer
import json

consumer = KafkaConsumer(
    'orders',
    bootstrap_servers=['localhost:9092'],
    group_id='order-processor',
    auto_offset_reset='earliest',
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for message in consumer:
    print(f"Partition: {message.partition}")
    print(f"Offset: {message.offset}")
    print(f"Key: {message.key}")
    print(f"Value: {message.value}")
```

### 5. Consumer Groups

**Consumer Group** — группа consumers, которые совместно читают топик.

```
Topic: orders (3 партиции)
                                          Consumer Group: "order-processors"
┌─────────────┐                          ┌─────────────────────────────────┐
│ Partition 0 │ ─────────────────────────│→ Consumer 1                     │
├─────────────┤                          ├─────────────────────────────────┤
│ Partition 1 │ ─────────────────────────│→ Consumer 2                     │
├─────────────┤                          ├─────────────────────────────────┤
│ Partition 2 │ ─────────────────────────│→ Consumer 3                     │
└─────────────┘                          └─────────────────────────────────┘

Правило: Одна партиция может читаться только одним consumer в группе
```

### 6. Offsets (Смещения)

**Offset** — уникальный идентификатор сообщения в партиции.

```
┌────────────────────────────────────────────────────────┐
│                    Partition 0                          │
├─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┬─────┤
│  0  │  1  │  2  │  3  │  4  │  5  │  6  │  7  │  8  │
└─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┴─────┘
              ↑                       ↑           ↑
         Committed               Current      Latest
          Offset                 Offset       Offset
```

```bash
# Просмотр offsets для consumer group
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
  --describe --group order-processors
```

### 7. Replication (Репликация)

Kafka реплицирует данные между брокерами для отказоустойчивости.

```
Topic: orders, Partition 0, Replication Factor: 3

┌─────────────┐  ┌─────────────┐  ┌─────────────┐
│  Broker 0   │  │  Broker 1   │  │  Broker 2   │
├─────────────┤  ├─────────────┤  ├─────────────┤
│  P0 LEADER  │──│  P0 FOLLOWER│──│  P0 FOLLOWER│
│             │  │   (ISR)     │  │   (ISR)     │
└─────────────┘  └─────────────┘  └─────────────┘
       │
       └── Все записи и чтения идут через лидера
           Followers постоянно синхронизируются
```

**ISR (In-Sync Replicas)** — набор реплик, которые синхронизированы с лидером.

## Примеры кода

### Полный пример Producer

```python
from kafka import KafkaProducer
from kafka.errors import KafkaError
import json
import logging

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class OrderProducer:
    def __init__(self, bootstrap_servers):
        self.producer = KafkaProducer(
            bootstrap_servers=bootstrap_servers,
            key_serializer=lambda k: k.encode('utf-8') if k else None,
            value_serializer=lambda v: json.dumps(v).encode('utf-8'),
            acks='all',  # Ждать подтверждения от всех реплик
            retries=3,
            max_in_flight_requests_per_connection=1
        )

    def send_order(self, user_id, order_data):
        """Отправка заказа в Kafka"""
        try:
            future = self.producer.send(
                'orders',
                key=user_id,
                value=order_data
            )

            # Синхронное ожидание результата
            record_metadata = future.get(timeout=10)

            logger.info(
                f"Заказ отправлен: topic={record_metadata.topic}, "
                f"partition={record_metadata.partition}, "
                f"offset={record_metadata.offset}"
            )
            return True

        except KafkaError as e:
            logger.error(f"Ошибка отправки заказа: {e}")
            return False

    def send_order_async(self, user_id, order_data, callback):
        """Асинхронная отправка с callback"""
        self.producer.send(
            'orders',
            key=user_id,
            value=order_data
        ).add_callback(callback).add_errback(lambda e: logger.error(f"Error: {e}"))

    def close(self):
        self.producer.flush()
        self.producer.close()


# Использование
if __name__ == "__main__":
    producer = OrderProducer(['localhost:9092'])

    order = {
        'order_id': '12345',
        'user_id': 'user_001',
        'items': [
            {'product': 'laptop', 'quantity': 1, 'price': 999.99}
        ],
        'total': 999.99
    }

    producer.send_order('user_001', order)
    producer.close()
```

### Полный пример Consumer

```python
from kafka import KafkaConsumer
from kafka.errors import KafkaError
import json
import logging
import signal
import sys

logging.basicConfig(level=logging.INFO)
logger = logging.getLogger(__name__)

class OrderConsumer:
    def __init__(self, bootstrap_servers, group_id):
        self.consumer = KafkaConsumer(
            'orders',
            bootstrap_servers=bootstrap_servers,
            group_id=group_id,
            auto_offset_reset='earliest',
            enable_auto_commit=False,  # Ручной commit
            value_deserializer=lambda m: json.loads(m.decode('utf-8')),
            key_deserializer=lambda k: k.decode('utf-8') if k else None
        )
        self.running = True

        # Обработка сигналов для graceful shutdown
        signal.signal(signal.SIGINT, self._signal_handler)
        signal.signal(signal.SIGTERM, self._signal_handler)

    def _signal_handler(self, signum, frame):
        logger.info("Получен сигнал остановки")
        self.running = False

    def process_order(self, order):
        """Обработка заказа"""
        logger.info(f"Обработка заказа: {order['order_id']}")
        # Бизнес-логика обработки заказа
        return True

    def run(self):
        """Основной цикл обработки"""
        logger.info("Запуск consumer...")

        try:
            while self.running:
                # Poll с таймаутом
                messages = self.consumer.poll(timeout_ms=1000)

                for topic_partition, records in messages.items():
                    for record in records:
                        logger.info(
                            f"Получено сообщение: partition={record.partition}, "
                            f"offset={record.offset}, key={record.key}"
                        )

                        # Обработка
                        if self.process_order(record.value):
                            # Commit после успешной обработки
                            self.consumer.commit()
                        else:
                            logger.error(f"Ошибка обработки заказа {record.value}")

        except Exception as e:
            logger.error(f"Ошибка в consumer: {e}")

        finally:
            self.consumer.close()
            logger.info("Consumer остановлен")


# Использование
if __name__ == "__main__":
    consumer = OrderConsumer(['localhost:9092'], 'order-processors')
    consumer.run()
```

### Просмотр метаданных кластера

```python
from kafka import KafkaAdminClient
from kafka.admin import NewTopic

admin = KafkaAdminClient(bootstrap_servers=['localhost:9092'])

# Информация о кластере
cluster_metadata = admin.describe_cluster()
print(f"Cluster ID: {cluster_metadata['cluster_id']}")
print(f"Controller: {cluster_metadata['controller_id']}")

print("\nБрокеры:")
for broker in cluster_metadata['brokers']:
    print(f"  - {broker['host']}:{broker['port']} (ID: {broker['node_id']})")

# Список топиков
topics = admin.list_topics()
print(f"\nТопики: {topics}")

# Создание нового топика
new_topic = NewTopic(
    name='new-orders',
    num_partitions=3,
    replication_factor=1
)
admin.create_topics([new_topic])

admin.close()
```

## Путь сообщения в Kafka

```
1. Producer создает сообщение
   ↓
2. Сериализация (ключ и значение)
   ↓
3. Partitioner выбирает партицию
   ↓
4. Сообщение отправляется на брокер-лидер
   ↓
5. Лидер записывает в лог и реплицирует на followers
   ↓
6. Подтверждение отправителю (acks)
   ↓
7. Consumer делает fetch запрос
   ↓
8. Брокер отдает сообщения из лога
   ↓
9. Десериализация
   ↓
10. Обработка приложением
   ↓
11. Commit offset
```

## Структура директорий Kafka

```
/var/kafka-logs/
├── __consumer_offsets-0/       # Внутренний топик для offsets
├── __consumer_offsets-1/
├── ...
├── orders-0/                   # Партиция 0 топика orders
│   ├── 00000000000000000000.log    # Сегмент лога
│   ├── 00000000000000000000.index  # Индекс
│   └── 00000000000000000000.timeindex  # Временной индекс
├── orders-1/                   # Партиция 1
└── orders-2/                   # Партиция 2
```

## CLI команды для экскурсии

### Работа с топиками

```bash
# Создание топика
kafka-topics.sh --create --topic demo-topic \
  --bootstrap-server localhost:9092 \
  --partitions 3 --replication-factor 1

# Описание топика
kafka-topics.sh --describe --topic demo-topic \
  --bootstrap-server localhost:9092

# Вывод:
# Topic: demo-topic   PartitionCount: 3   ReplicationFactor: 1
# Topic: demo-topic   Partition: 0   Leader: 0   Replicas: 0   Isr: 0
# Topic: demo-topic   Partition: 1   Leader: 0   Replicas: 0   Isr: 0
# Topic: demo-topic   Partition: 2   Leader: 0   Replicas: 0   Isr: 0
```

### Работа с сообщениями

```bash
# Отправка сообщений
kafka-console-producer.sh --topic demo-topic \
  --bootstrap-server localhost:9092

# Чтение сообщений
kafka-console-consumer.sh --topic demo-topic \
  --bootstrap-server localhost:9092 --from-beginning

# Чтение с отображением партиции и offset
kafka-console-consumer.sh --topic demo-topic \
  --bootstrap-server localhost:9092 --from-beginning \
  --property print.partition=true \
  --property print.offset=true
```

### Работа с consumer groups

```bash
# Список групп
kafka-consumer-groups.sh --list --bootstrap-server localhost:9092

# Описание группы
kafka-consumer-groups.sh --describe --group my-group \
  --bootstrap-server localhost:9092

# Сброс offset
kafka-consumer-groups.sh --reset-offsets \
  --group my-group --topic demo-topic \
  --to-earliest --execute \
  --bootstrap-server localhost:9092
```

## Best Practices

### 1. Проектирование топиков

- Используйте осмысленные имена: `domain.entity.action` (например, `payments.transactions.created`)
- Не создавайте слишком много мелких топиков
- Планируйте retention в зависимости от use case

### 2. Выбор количества партиций

```
Формула для расчета:
partitions = max(throughput_producer / throughput_per_partition,
                 throughput_consumer / throughput_per_partition)

Рекомендации:
- Минимум: количество consumers в самой большой группе
- Начните с небольшого числа, увеличивайте по мере необходимости
- Учитывайте, что партиции нельзя уменьшить
```

### 3. Настройка репликации

```
Production рекомендации:
- replication.factor = 3
- min.insync.replicas = 2
- acks = all

Это обеспечивает:
- Переживание отказа одного брокера
- Гарантию записи как минимум на 2 реплики
```

### 4. Мониторинг

Ключевые метрики:
- Under-replicated partitions
- Offline partitions
- Consumer lag
- Request latency
- Disk usage

## Дополнительные ресурсы

- [Kafka Documentation](https://kafka.apache.org/documentation/)
- [Kafka: The Definitive Guide](https://www.confluent.io/resources/kafka-the-definitive-guide/)
- [Confluent Developer](https://developer.confluent.io/)
