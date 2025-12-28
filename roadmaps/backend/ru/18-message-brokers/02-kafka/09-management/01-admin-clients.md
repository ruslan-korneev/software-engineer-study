# Клиенты администрирования Kafka

[prev: 07-cloud-container-storage](../08-kafka-storage/07-cloud-container-storage.md) | [next: 02-systemd-service](./02-systemd-service.md)

---

## Описание

Клиенты администрирования Kafka — это набор инструментов командной строки и программных API, которые позволяют управлять кластером Kafka. Они используются для создания и удаления топиков, управления партициями, настройки конфигурации, мониторинга состояния кластера и выполнения других административных задач.

Apache Kafka предоставляет как CLI-утилиты (shell-скрипты), так и Java Admin API для программного управления.

## Ключевые концепции

### CLI-инструменты Kafka

Kafka поставляется с набором скриптов в директории `bin/`:

| Инструмент | Назначение |
|------------|------------|
| `kafka-topics.sh` | Управление топиками (создание, удаление, изменение) |
| `kafka-configs.sh` | Управление конфигурацией брокеров, топиков, клиентов |
| `kafka-consumer-groups.sh` | Управление группами потребителей |
| `kafka-acls.sh` | Управление списками контроля доступа (ACL) |
| `kafka-reassign-partitions.sh` | Перераспределение партиций между брокерами |
| `kafka-log-dirs.sh` | Информация о директориях логов |
| `kafka-broker-api-versions.sh` | Проверка версий API брокеров |
| `kafka-cluster.sh` | Управление кластером (KRaft mode) |
| `kafka-metadata.sh` | Работа с метаданными (KRaft mode) |

### AdminClient API (Java)

Программный интерфейс для управления Kafka из приложений:

```java
import org.apache.kafka.clients.admin.*;
import java.util.*;

Properties props = new Properties();
props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");

try (AdminClient admin = AdminClient.create(props)) {
    // Операции администрирования
}
```

## Примеры

### Управление топиками

```bash
# Создание топика
kafka-topics.sh --bootstrap-server localhost:9092 \
    --create \
    --topic my-topic \
    --partitions 6 \
    --replication-factor 3 \
    --config retention.ms=604800000

# Список всех топиков
kafka-topics.sh --bootstrap-server localhost:9092 --list

# Подробная информация о топике
kafka-topics.sh --bootstrap-server localhost:9092 \
    --describe \
    --topic my-topic

# Увеличение количества партиций
kafka-topics.sh --bootstrap-server localhost:9092 \
    --alter \
    --topic my-topic \
    --partitions 12

# Удаление топика
kafka-topics.sh --bootstrap-server localhost:9092 \
    --delete \
    --topic my-topic
```

### Управление конфигурацией

```bash
# Просмотр конфигурации топика
kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type topics \
    --entity-name my-topic \
    --describe

# Изменение конфигурации топика
kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type topics \
    --entity-name my-topic \
    --alter \
    --add-config retention.ms=86400000,max.message.bytes=10485760

# Удаление настройки (возврат к значению по умолчанию)
kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type topics \
    --entity-name my-topic \
    --alter \
    --delete-config retention.ms

# Конфигурация брокера
kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type brokers \
    --entity-name 0 \
    --describe

# Динамическое изменение конфигурации брокера
kafka-configs.sh --bootstrap-server localhost:9092 \
    --entity-type brokers \
    --entity-name 0 \
    --alter \
    --add-config log.cleaner.threads=4
```

### Управление группами потребителей

```bash
# Список всех групп потребителей
kafka-consumer-groups.sh --bootstrap-server localhost:9092 --list

# Подробная информация о группе
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe \
    --group my-consumer-group

# Просмотр состояния группы
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --describe \
    --group my-consumer-group \
    --state

# Сброс оффсетов на начало
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --group my-consumer-group \
    --topic my-topic \
    --reset-offsets \
    --to-earliest \
    --execute

# Сброс оффсетов на конкретную дату
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --group my-consumer-group \
    --topic my-topic \
    --reset-offsets \
    --to-datetime 2024-01-15T00:00:00.000 \
    --execute

# Удаление группы потребителей
kafka-consumer-groups.sh --bootstrap-server localhost:9092 \
    --delete \
    --group my-consumer-group
```

### Перераспределение партиций

```bash
# Генерация плана перераспределения
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --topics-to-move-json-file topics.json \
    --broker-list "1,2,3" \
    --generate

# Файл topics.json
cat << 'EOF' > topics.json
{
  "topics": [
    {"topic": "my-topic"}
  ],
  "version": 1
}
EOF

# Выполнение перераспределения
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --reassignment-json-file reassignment.json \
    --execute

# Проверка статуса перераспределения
kafka-reassign-partitions.sh --bootstrap-server localhost:9092 \
    --reassignment-json-file reassignment.json \
    --verify
```

### AdminClient API примеры

```java
import org.apache.kafka.clients.admin.*;
import org.apache.kafka.common.config.ConfigResource;
import java.util.*;
import java.util.concurrent.ExecutionException;

public class KafkaAdminExample {

    public static void main(String[] args) throws Exception {
        Properties props = new Properties();
        props.put(AdminClientConfig.BOOTSTRAP_SERVERS_CONFIG, "localhost:9092");
        props.put(AdminClientConfig.REQUEST_TIMEOUT_MS_CONFIG, 30000);

        try (AdminClient admin = AdminClient.create(props)) {
            // Создание топика
            createTopic(admin);

            // Получение информации о топике
            describeTopic(admin);

            // Изменение конфигурации
            alterTopicConfig(admin);

            // Получение информации о кластере
            describeCluster(admin);
        }
    }

    static void createTopic(AdminClient admin) throws Exception {
        NewTopic topic = new NewTopic("new-topic", 3, (short) 2);
        topic.configs(Map.of(
            "retention.ms", "604800000",
            "cleanup.policy", "delete"
        ));

        CreateTopicsResult result = admin.createTopics(List.of(topic));
        result.all().get();
        System.out.println("Topic created successfully");
    }

    static void describeTopic(AdminClient admin) throws Exception {
        DescribeTopicsResult result = admin.describeTopics(List.of("new-topic"));
        Map<String, TopicDescription> descriptions = result.allTopicNames().get();

        descriptions.forEach((name, desc) -> {
            System.out.println("Topic: " + name);
            System.out.println("Partitions: " + desc.partitions().size());
            desc.partitions().forEach(p -> {
                System.out.printf("  Partition %d: leader=%d, replicas=%s, isr=%s%n",
                    p.partition(),
                    p.leader().id(),
                    p.replicas(),
                    p.isr()
                );
            });
        });
    }

    static void alterTopicConfig(AdminClient admin) throws Exception {
        ConfigResource resource = new ConfigResource(
            ConfigResource.Type.TOPIC, "new-topic"
        );

        Map<ConfigResource, Collection<AlterConfigOp>> configs = Map.of(
            resource, List.of(
                new AlterConfigOp(
                    new ConfigEntry("retention.ms", "86400000"),
                    AlterConfigOp.OpType.SET
                )
            )
        );

        admin.incrementalAlterConfigs(configs).all().get();
        System.out.println("Config updated successfully");
    }

    static void describeCluster(AdminClient admin) throws Exception {
        DescribeClusterResult result = admin.describeCluster();

        System.out.println("Cluster ID: " + result.clusterId().get());
        System.out.println("Controller: " + result.controller().get());

        result.nodes().get().forEach(node -> {
            System.out.printf("Node: id=%d, host=%s, port=%d%n",
                node.id(), node.host(), node.port()
            );
        });
    }
}
```

### Управление ACL

```bash
# Добавление ACL для продюсера
kafka-acls.sh --bootstrap-server localhost:9092 \
    --add \
    --allow-principal User:producer-user \
    --operation Write \
    --topic my-topic

# Добавление ACL для консьюмера
kafka-acls.sh --bootstrap-server localhost:9092 \
    --add \
    --allow-principal User:consumer-user \
    --operation Read \
    --topic my-topic \
    --group my-group

# Список всех ACL
kafka-acls.sh --bootstrap-server localhost:9092 --list

# Удаление ACL
kafka-acls.sh --bootstrap-server localhost:9092 \
    --remove \
    --allow-principal User:producer-user \
    --operation Write \
    --topic my-topic
```

## Best Practices

### Безопасность администрирования

```bash
# Использование SSL для подключения
kafka-topics.sh --bootstrap-server localhost:9093 \
    --command-config admin.properties \
    --list

# admin.properties
security.protocol=SSL
ssl.truststore.location=/path/to/truststore.jks
ssl.truststore.password=password
ssl.keystore.location=/path/to/keystore.jks
ssl.keystore.password=password
```

### Рекомендации по использованию

1. **Используйте отдельные учетные записи** для административных операций
2. **Логируйте все административные действия** для аудита
3. **Тестируйте изменения** в staging-окружении перед production
4. **Используйте скрипты автоматизации** для повторяющихся операций
5. **Применяйте принцип минимальных привилегий** при настройке ACL

### Автоматизация с помощью скриптов

```bash
#!/bin/bash
# Скрипт для создания топика с проверками

BOOTSTRAP_SERVERS="localhost:9092"
TOPIC_NAME=$1
PARTITIONS=${2:-3}
REPLICATION_FACTOR=${3:-2}

# Проверка существования топика
if kafka-topics.sh --bootstrap-server $BOOTSTRAP_SERVERS \
    --list | grep -q "^${TOPIC_NAME}$"; then
    echo "Topic $TOPIC_NAME already exists"
    exit 1
fi

# Создание топика
kafka-topics.sh --bootstrap-server $BOOTSTRAP_SERVERS \
    --create \
    --topic $TOPIC_NAME \
    --partitions $PARTITIONS \
    --replication-factor $REPLICATION_FACTOR \
    --config retention.ms=604800000 \
    --config min.insync.replicas=$((REPLICATION_FACTOR - 1))

echo "Topic $TOPIC_NAME created successfully"
```

### Мониторинг административных операций

```java
// Пример с обработкой ошибок и логированием
public class SafeAdminOperations {

    private static final Logger logger = LoggerFactory.getLogger(SafeAdminOperations.class);

    public void safeCreateTopic(AdminClient admin, String topicName,
                                int partitions, short replicationFactor) {
        try {
            // Проверка существования
            Set<String> existingTopics = admin.listTopics().names().get();
            if (existingTopics.contains(topicName)) {
                logger.warn("Topic {} already exists, skipping creation", topicName);
                return;
            }

            // Создание топика
            NewTopic topic = new NewTopic(topicName, partitions, replicationFactor);
            admin.createTopics(List.of(topic)).all().get(30, TimeUnit.SECONDS);

            logger.info("Successfully created topic: {}", topicName);

        } catch (ExecutionException e) {
            if (e.getCause() instanceof TopicExistsException) {
                logger.warn("Topic {} was created concurrently", topicName);
            } else {
                logger.error("Failed to create topic: {}", topicName, e);
                throw new RuntimeException(e);
            }
        } catch (Exception e) {
            logger.error("Unexpected error creating topic: {}", topicName, e);
            throw new RuntimeException(e);
        }
    }
}
```

## Инструменты управления

### Графические интерфейсы

- **Confluent Control Center** — коммерческий инструмент от Confluent
- **Kafdrop** — легковесный веб-интерфейс
- **AKHQ (Kafka HQ)** — мощный open-source UI
- **Kafka UI** — современный интерфейс управления

### Программные решения

- **Terraform Provider** — управление инфраструктурой как кодом
- **Strimzi Operator** — управление Kafka в Kubernetes
- **Cruise Control** — автоматическая балансировка кластера

---

[prev: 07-cloud-container-storage](../08-kafka-storage/07-cloud-container-storage.md) | [next: 02-systemd-service](./02-systemd-service.md)
