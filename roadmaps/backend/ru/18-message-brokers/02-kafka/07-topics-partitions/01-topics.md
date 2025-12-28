# Темы (Topics)

[prev: 07-exercise](../06-brokers/07-exercise.md) | [next: 02-partitions](./02-partitions.md)

---

## Описание

**Topic (тема/топик)** — это фундаментальная абстракция в Apache Kafka, представляющая собой категорию или канал для публикации сообщений. Топик можно рассматривать как аналог таблицы в базе данных или папки в файловой системе, но с особенностями, специфичными для потоковой обработки данных.

Каждый топик имеет уникальное имя в пределах кластера Kafka и может содержать неограниченное количество сообщений. Продюсеры публикуют сообщения в топики, а консьюмеры подписываются на топики для получения данных.

### Основные характеристики топиков

- **Неизменяемость**: сообщения, записанные в топик, не могут быть изменены
- **Append-only**: новые сообщения добавляются только в конец лога
- **Распределенность**: топик разделен на партиции, распределенные по брокерам
- **Персистентность**: данные хранятся на диске в течение заданного времени
- **Многопользовательность**: множество консьюмеров могут читать из одного топика

## Ключевые концепции

### Структура топика

```
Topic: orders
├── Partition 0: [msg0, msg1, msg2, msg3, ...]
├── Partition 1: [msg0, msg1, msg2, ...]
└── Partition 2: [msg0, msg1, msg2, msg3, msg4, ...]
```

### Конфигурационные параметры топика

| Параметр | Описание | Значение по умолчанию |
|----------|----------|----------------------|
| `num.partitions` | Количество партиций | 1 |
| `replication.factor` | Фактор репликации | 1 |
| `retention.ms` | Время хранения сообщений | 604800000 (7 дней) |
| `retention.bytes` | Максимальный размер данных | -1 (без ограничений) |
| `segment.bytes` | Размер сегмента лога | 1073741824 (1 GB) |
| `cleanup.policy` | Политика очистки | delete |
| `min.insync.replicas` | Минимум синхронных реплик | 1 |
| `compression.type` | Тип сжатия | producer |
| `max.message.bytes` | Максимальный размер сообщения | 1048588 |

### Именование топиков

Правила именования:
- Длина: от 1 до 249 символов
- Допустимые символы: буквы, цифры, точки, подчеркивания, дефисы
- Регистрозависимые имена
- Нельзя использовать `.` и `_` в одном имени (внутренние конфликты)

Рекомендации по именованию:
```
<domain>.<entity>.<action>.<version>
```

Примеры:
- `payments.orders.created.v1`
- `inventory.products.updated`
- `users.notifications.email`

## Примеры

### Создание топика через CLI

```bash
# Базовое создание топика
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic my-topic \
  --partitions 3 \
  --replication-factor 2

# Создание с дополнительными конфигурациями
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=86400000 \
  --config cleanup.policy=compact \
  --config min.insync.replicas=2
```

### Просмотр информации о топике

```bash
# Список всех топиков
kafka-topics.sh --list --bootstrap-server localhost:9092

# Детальная информация о топике
kafka-topics.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic orders

# Вывод:
# Topic: orders	PartitionCount: 6	ReplicationFactor: 3
# Topic: orders	Partition: 0	Leader: 1	Replicas: 1,2,3	Isr: 1,2,3
# Topic: orders	Partition: 1	Leader: 2	Replicas: 2,3,1	Isr: 2,3,1
# ...
```

### Изменение конфигурации топика

```bash
# Изменение retention
kafka-configs.sh --alter \
  --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name orders \
  --add-config retention.ms=172800000

# Увеличение количества партиций (только увеличение!)
kafka-topics.sh --alter \
  --bootstrap-server localhost:9092 \
  --topic orders \
  --partitions 12
```

### Программное создание топика (Java)

```java
import org.apache.kafka.clients.admin.*;
import java.util.*;

public class TopicManager {

    public static void createTopic(String bootstrapServers,
                                   String topicName,
                                   int partitions,
                                   short replicationFactor) {

        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);

        try (AdminClient adminClient = AdminClient.create(props)) {

            // Конфигурация топика
            Map<String, String> topicConfig = new HashMap<>();
            topicConfig.put("retention.ms", "86400000");
            topicConfig.put("cleanup.policy", "delete");
            topicConfig.put("min.insync.replicas", "2");

            NewTopic newTopic = new NewTopic(topicName, partitions, replicationFactor)
                .configs(topicConfig);

            CreateTopicsResult result = adminClient.createTopics(
                Collections.singleton(newTopic)
            );

            // Ожидание завершения
            result.all().get();
            System.out.println("Topic created: " + topicName);

        } catch (Exception e) {
            e.printStackTrace();
        }
    }

    public static void describeTopic(String bootstrapServers, String topicName) {
        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, bootstrapServers);

        try (AdminClient adminClient = AdminClient.create(props)) {

            DescribeTopicsResult result = adminClient.describeTopics(
                Collections.singleton(topicName)
            );

            TopicDescription description = result.values().get(topicName).get();

            System.out.println("Topic: " + description.name());
            System.out.println("Partitions: " + description.partitions().size());

            for (TopicPartitionInfo partition : description.partitions()) {
                System.out.printf("Partition %d: Leader=%d, Replicas=%s, ISR=%s%n",
                    partition.partition(),
                    partition.leader().id(),
                    partition.replicas(),
                    partition.isr()
                );
            }

        } catch (Exception e) {
            e.printStackTrace();
        }
    }
}
```

### Программное создание топика (Python)

```python
from kafka.admin import KafkaAdminClient, NewTopic
from kafka.errors import TopicAlreadyExistsError

def create_topic(bootstrap_servers: str,
                 topic_name: str,
                 num_partitions: int = 3,
                 replication_factor: int = 1):
    """Создание топика в Kafka"""

    admin_client = KafkaAdminClient(
        bootstrap_servers=bootstrap_servers,
        client_id='topic-manager'
    )

    topic = NewTopic(
        name=topic_name,
        num_partitions=num_partitions,
        replication_factor=replication_factor,
        topic_configs={
            'retention.ms': '86400000',
            'cleanup.policy': 'delete'
        }
    )

    try:
        admin_client.create_topics(new_topics=[topic], validate_only=False)
        print(f"Topic '{topic_name}' created successfully")
    except TopicAlreadyExistsError:
        print(f"Topic '{topic_name}' already exists")
    finally:
        admin_client.close()


def list_topics(bootstrap_servers: str):
    """Получение списка всех топиков"""

    admin_client = KafkaAdminClient(
        bootstrap_servers=bootstrap_servers,
        client_id='topic-manager'
    )

    topics = admin_client.list_topics()
    print("Available topics:")
    for topic in topics:
        print(f"  - {topic}")

    admin_client.close()
    return topics


def delete_topic(bootstrap_servers: str, topic_name: str):
    """Удаление топика"""

    admin_client = KafkaAdminClient(
        bootstrap_servers=bootstrap_servers,
        client_id='topic-manager'
    )

    admin_client.delete_topics(topics=[topic_name])
    print(f"Topic '{topic_name}' deleted")
    admin_client.close()


# Использование
if __name__ == "__main__":
    servers = "localhost:9092"

    create_topic(servers, "user-events", num_partitions=6)
    list_topics(servers)
```

### Работа с топиками в Spring Boot

```java
import org.apache.kafka.clients.admin.NewTopic;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.kafka.config.TopicBuilder;

@Configuration
public class KafkaTopicConfig {

    @Bean
    public NewTopic ordersTopic() {
        return TopicBuilder.name("orders")
            .partitions(6)
            .replicas(3)
            .config("retention.ms", "604800000")
            .config("min.insync.replicas", "2")
            .build();
    }

    @Bean
    public NewTopic paymentsTopic() {
        return TopicBuilder.name("payments")
            .partitions(3)
            .replicas(3)
            .compact()  // cleanup.policy=compact
            .build();
    }

    @Bean
    public NewTopic notificationsTopic() {
        return TopicBuilder.name("notifications")
            .partitions(12)
            .replicas(2)
            .config("retention.bytes", "1073741824")
            .build();
    }
}
```

## Best Practices

### 1. Планирование количества партиций

```
Формула: partitions = max(throughput / producer_throughput, throughput / consumer_throughput)

Пример:
- Требуемая пропускная способность: 1 GB/s
- Пропускная способность одного продюсера: 100 MB/s
- Пропускная способность одного консьюмера: 50 MB/s

partitions = max(1000/100, 1000/50) = max(10, 20) = 20 партиций
```

### 2. Выбор фактора репликации

- **Development**: replication-factor = 1
- **Production**: replication-factor = 3 (минимум)
- **Critical data**: replication-factor = 3-5

### 3. Настройка retention

```bash
# Для событий (event sourcing)
retention.ms=-1  # Бессрочное хранение

# Для логов
retention.ms=604800000  # 7 дней

# Для метрик
retention.ms=86400000  # 24 часа

# Для потоковой обработки
retention.ms=3600000  # 1 час
```

### 4. Организация топиков по доменам

```
# Микросервисная архитектура
user-service.users.created
user-service.users.updated
order-service.orders.placed
order-service.orders.shipped
payment-service.payments.processed

# Event-driven архитектура
events.user.registered
events.order.created
events.payment.completed
```

### 5. Версионирование топиков

```
# При изменении схемы данных
orders.v1  → orders.v2
users.events.v1 → users.events.v2

# Миграция данных
1. Создать новый топик (v2)
2. Начать писать в оба топика
3. Мигрировать консьюмеров на v2
4. Удалить старый топик (v1)
```

### 6. Мониторинг топиков

```bash
# Проверка отставания консьюмеров
kafka-consumer-groups.sh --describe \
  --bootstrap-server localhost:9092 \
  --group my-consumer-group

# Проверка размера топика
kafka-log-dirs.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic-list orders
```

### 7. Защита от случайного удаления

```properties
# server.properties
delete.topic.enable=false  # Запретить удаление топиков

# Или использовать ACL
kafka-acls.sh --add \
  --allow-principal User:admin \
  --operation Delete \
  --topic orders \
  --bootstrap-server localhost:9092
```

### 8. Оптимизация производительности

```properties
# Конфигурация для высокой пропускной способности
compression.type=lz4
segment.bytes=1073741824
segment.ms=604800000

# Конфигурация для низкой задержки
segment.bytes=104857600
segment.ms=300000
```

## Типичные ошибки

1. **Слишком мало партиций**: ограничивает параллелизм
2. **Слишком много партиций**: увеличивает время восстановления
3. **Низкий replication-factor**: риск потери данных
4. **Отсутствие планирования retention**: переполнение диска
5. **Неудачное именование**: сложности с управлением

## Связанные темы

- [Партиции](./02-partitions.md)
- [Сжатые топики](./04-compacted-topics.md)
- [Тестирование с EmbeddedKafka](./03-embedded-kafka-testing.md)

---

[prev: 07-exercise](../06-brokers/07-exercise.md) | [next: 02-partitions](./02-partitions.md)