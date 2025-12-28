# Разработка проекта на основе Kafka

[prev: 06-stream-processing-terminology](../02-getting-acquainted/06-stream-processing-terminology.md) | [next: 02-sensor-events](./02-sensor-events.md)

---

## Описание

Разработка проекта с использованием Apache Kafka требует тщательного планирования архитектуры, правильной организации структуры проекта и настройки окружения для разработки. В этом разделе мы рассмотрим полный цикл создания проекта на основе Kafka: от структуры директорий до Docker Compose конфигурации.

## Структура проекта

### Базовая структура для Python-проекта

```
kafka-project/
├── docker/
│   ├── docker-compose.yml
│   ├── docker-compose.override.yml
│   └── kafka/
│       └── server.properties
├── src/
│   ├── __init__.py
│   ├── producers/
│   │   ├── __init__.py
│   │   ├── base_producer.py
│   │   └── sensor_producer.py
│   ├── consumers/
│   │   ├── __init__.py
│   │   ├── base_consumer.py
│   │   └── sensor_consumer.py
│   ├── models/
│   │   ├── __init__.py
│   │   └── events.py
│   ├── serializers/
│   │   ├── __init__.py
│   │   ├── json_serializer.py
│   │   └── avro_serializer.py
│   └── config/
│       ├── __init__.py
│       └── kafka_config.py
├── tests/
│   ├── __init__.py
│   ├── test_producers.py
│   └── test_consumers.py
├── scripts/
│   ├── create_topics.sh
│   └── init_kafka.py
├── requirements.txt
├── pyproject.toml
├── Makefile
└── README.md
```

### Структура для Java/Kotlin проекта (Maven)

```
kafka-project/
├── docker/
│   ├── docker-compose.yml
│   └── kafka/
│       └── server.properties
├── src/
│   ├── main/
│   │   ├── java/
│   │   │   └── com/example/kafka/
│   │   │       ├── Application.java
│   │   │       ├── config/
│   │   │       │   └── KafkaConfig.java
│   │   │       ├── producer/
│   │   │       │   ├── BaseProducer.java
│   │   │       │   └── SensorProducer.java
│   │   │       ├── consumer/
│   │   │       │   ├── BaseConsumer.java
│   │   │       │   └── SensorConsumer.java
│   │   │       ├── model/
│   │   │       │   └── SensorEvent.java
│   │   │       └── serializer/
│   │   │           └── JsonSerializer.java
│   │   └── resources/
│   │       ├── application.yml
│   │       └── logback.xml
│   └── test/
│       └── java/
│           └── com/example/kafka/
│               └── ProducerTest.java
├── pom.xml
└── README.md
```

## Docker Compose для разработки

### Базовая конфигурация с Kafka и Zookeeper

```yaml
# docker/docker-compose.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000
    volumes:
      - zookeeper_data:/var/lib/zookeeper/data
      - zookeeper_logs:/var/lib/zookeeper/log
    healthcheck:
      test: ["CMD", "nc", "-z", "localhost", "2181"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    container_name: kafka
    depends_on:
      zookeeper:
        condition: service_healthy
    ports:
      - "9092:9092"
      - "29092:29092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_GROUP_INITIAL_REBALANCE_DELAY_MS: 0
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_AUTO_CREATE_TOPICS_ENABLE: "true"
    volumes:
      - kafka_data:/var/lib/kafka/data
    healthcheck:
      test: ["CMD", "kafka-broker-api-versions", "--bootstrap-server", "localhost:9092"]
      interval: 10s
      timeout: 5s
      retries: 5

  kafka-ui:
    image: provectuslabs/kafka-ui:latest
    container_name: kafka-ui
    depends_on:
      - kafka
    ports:
      - "8080:8080"
    environment:
      KAFKA_CLUSTERS_0_NAME: local
      KAFKA_CLUSTERS_0_BOOTSTRAPSERVERS: kafka:29092
      KAFKA_CLUSTERS_0_ZOOKEEPER: zookeeper:2181

volumes:
  zookeeper_data:
  zookeeper_logs:
  kafka_data:
```

### Конфигурация с Kafka в режиме KRaft (без Zookeeper)

```yaml
# docker/docker-compose-kraft.yml
version: '3.8'

services:
  kafka:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka
    container_name: kafka
    ports:
      - "9092:9092"
      - "9093:9093"
    environment:
      KAFKA_NODE_ID: 1
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: CONTROLLER:PLAINTEXT,PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENERS: PLAINTEXT://kafka:29092,CONTROLLER://kafka:9093,PLAINTEXT_HOST://0.0.0.0:9092
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_CONTROLLER_LISTENER_NAMES: CONTROLLER
      KAFKA_CONTROLLER_QUORUM_VOTERS: 1@kafka:9093
      KAFKA_PROCESS_ROLES: broker,controller
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_REPLICATION_FACTOR: 1
      KAFKA_TRANSACTION_STATE_LOG_MIN_ISR: 1
      KAFKA_LOG_DIRS: /var/lib/kafka/data
      CLUSTER_ID: MkU3OEVBNTcwNTJENDM2Qk
    volumes:
      - kafka_kraft_data:/var/lib/kafka/data

volumes:
  kafka_kraft_data:
```

### Расширенная конфигурация для продакшен-подобной среды

```yaml
# docker/docker-compose-full.yml
version: '3.8'

services:
  zookeeper:
    image: confluentinc/cp-zookeeper:7.5.0
    hostname: zookeeper
    container_name: zookeeper
    ports:
      - "2181:2181"
    environment:
      ZOOKEEPER_CLIENT_PORT: 2181
      ZOOKEEPER_TICK_TIME: 2000

  kafka-1:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka-1
    container_name: kafka-1
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-1:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2

  kafka-2:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka-2
    container_name: kafka-2
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-2:29092,PLAINTEXT_HOST://localhost:9093
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2

  kafka-3:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka-3
    container_name: kafka-3
    depends_on:
      - zookeeper
    ports:
      - "9094:9094"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: zookeeper:2181
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka-3:29092,PLAINTEXT_HOST://localhost:9094
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_INTER_BROKER_LISTENER_NAME: PLAINTEXT
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2

  schema-registry:
    image: confluentinc/cp-schema-registry:7.5.0
    hostname: schema-registry
    container_name: schema-registry
    depends_on:
      - kafka-1
      - kafka-2
      - kafka-3
    ports:
      - "8081:8081"
    environment:
      SCHEMA_REGISTRY_HOST_NAME: schema-registry
      SCHEMA_REGISTRY_KAFKASTORE_BOOTSTRAP_SERVERS: kafka-1:29092,kafka-2:29092,kafka-3:29092
      SCHEMA_REGISTRY_LISTENERS: http://0.0.0.0:8081

  kafka-connect:
    image: confluentinc/cp-kafka-connect:7.5.0
    hostname: kafka-connect
    container_name: kafka-connect
    depends_on:
      - kafka-1
      - schema-registry
    ports:
      - "8083:8083"
    environment:
      CONNECT_BOOTSTRAP_SERVERS: kafka-1:29092,kafka-2:29092,kafka-3:29092
      CONNECT_REST_PORT: 8083
      CONNECT_GROUP_ID: kafka-connect-group
      CONNECT_CONFIG_STORAGE_TOPIC: _connect-configs
      CONNECT_OFFSET_STORAGE_TOPIC: _connect-offsets
      CONNECT_STATUS_STORAGE_TOPIC: _connect-status
      CONNECT_KEY_CONVERTER: org.apache.kafka.connect.storage.StringConverter
      CONNECT_VALUE_CONVERTER: io.confluent.connect.avro.AvroConverter
      CONNECT_VALUE_CONVERTER_SCHEMA_REGISTRY_URL: http://schema-registry:8081
      CONNECT_REST_ADVERTISED_HOST_NAME: kafka-connect
      CONNECT_CONFIG_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_OFFSET_STORAGE_REPLICATION_FACTOR: 3
      CONNECT_STATUS_STORAGE_REPLICATION_FACTOR: 3
```

## Конфигурация проекта

### Python: Конфигурационный модуль

```python
# src/config/kafka_config.py
import os
from dataclasses import dataclass
from typing import Optional, List


@dataclass
class KafkaConfig:
    """Конфигурация для подключения к Kafka."""

    bootstrap_servers: str
    client_id: str
    group_id: Optional[str] = None
    auto_offset_reset: str = "earliest"
    enable_auto_commit: bool = True
    max_poll_records: int = 500
    session_timeout_ms: int = 30000

    # Настройки безопасности
    security_protocol: str = "PLAINTEXT"
    sasl_mechanism: Optional[str] = None
    sasl_username: Optional[str] = None
    sasl_password: Optional[str] = None

    @classmethod
    def from_env(cls) -> "KafkaConfig":
        """Создание конфигурации из переменных окружения."""
        return cls(
            bootstrap_servers=os.getenv("KAFKA_BOOTSTRAP_SERVERS", "localhost:9092"),
            client_id=os.getenv("KAFKA_CLIENT_ID", "python-client"),
            group_id=os.getenv("KAFKA_GROUP_ID"),
            auto_offset_reset=os.getenv("KAFKA_AUTO_OFFSET_RESET", "earliest"),
            enable_auto_commit=os.getenv("KAFKA_ENABLE_AUTO_COMMIT", "true").lower() == "true",
            security_protocol=os.getenv("KAFKA_SECURITY_PROTOCOL", "PLAINTEXT"),
            sasl_mechanism=os.getenv("KAFKA_SASL_MECHANISM"),
            sasl_username=os.getenv("KAFKA_SASL_USERNAME"),
            sasl_password=os.getenv("KAFKA_SASL_PASSWORD"),
        )

    def to_producer_config(self) -> dict:
        """Получить конфигурацию для Producer."""
        config = {
            "bootstrap.servers": self.bootstrap_servers,
            "client.id": self.client_id,
        }
        self._add_security_config(config)
        return config

    def to_consumer_config(self) -> dict:
        """Получить конфигурацию для Consumer."""
        config = {
            "bootstrap.servers": self.bootstrap_servers,
            "client.id": self.client_id,
            "group.id": self.group_id,
            "auto.offset.reset": self.auto_offset_reset,
            "enable.auto.commit": self.enable_auto_commit,
            "max.poll.records": self.max_poll_records,
            "session.timeout.ms": self.session_timeout_ms,
        }
        self._add_security_config(config)
        return config

    def _add_security_config(self, config: dict) -> None:
        """Добавить настройки безопасности."""
        if self.security_protocol != "PLAINTEXT":
            config["security.protocol"] = self.security_protocol
            if self.sasl_mechanism:
                config["sasl.mechanism"] = self.sasl_mechanism
                config["sasl.username"] = self.sasl_username
                config["sasl.password"] = self.sasl_password


# Настройки для разных окружений
class DevelopmentConfig(KafkaConfig):
    """Конфигурация для разработки."""
    def __init__(self):
        super().__init__(
            bootstrap_servers="localhost:9092",
            client_id="dev-client",
            group_id="dev-group",
        )


class ProductionConfig(KafkaConfig):
    """Конфигурация для продакшена."""
    def __init__(self):
        super().__init__(
            bootstrap_servers=os.getenv("KAFKA_BOOTSTRAP_SERVERS"),
            client_id=os.getenv("KAFKA_CLIENT_ID"),
            group_id=os.getenv("KAFKA_GROUP_ID"),
            security_protocol="SASL_SSL",
            sasl_mechanism="PLAIN",
            sasl_username=os.getenv("KAFKA_SASL_USERNAME"),
            sasl_password=os.getenv("KAFKA_SASL_PASSWORD"),
        )


def get_config() -> KafkaConfig:
    """Получить конфигурацию в зависимости от окружения."""
    env = os.getenv("ENVIRONMENT", "development")
    if env == "production":
        return ProductionConfig()
    return DevelopmentConfig()
```

## Makefile для автоматизации

```makefile
# Makefile
.PHONY: help start stop restart logs clean create-topics test lint

DOCKER_COMPOSE = docker compose -f docker/docker-compose.yml

help:
	@echo "Доступные команды:"
	@echo "  make start         - Запустить Kafka кластер"
	@echo "  make stop          - Остановить Kafka кластер"
	@echo "  make restart       - Перезапустить Kafka кластер"
	@echo "  make logs          - Показать логи"
	@echo "  make clean         - Удалить все контейнеры и volumes"
	@echo "  make create-topics - Создать необходимые топики"
	@echo "  make test          - Запустить тесты"
	@echo "  make lint          - Проверить код линтером"

start:
	$(DOCKER_COMPOSE) up -d
	@echo "Ожидание запуска Kafka..."
	@sleep 10
	@echo "Kafka кластер запущен!"

stop:
	$(DOCKER_COMPOSE) down

restart: stop start

logs:
	$(DOCKER_COMPOSE) logs -f

clean:
	$(DOCKER_COMPOSE) down -v --remove-orphans

create-topics:
	docker exec kafka kafka-topics --bootstrap-server localhost:9092 \
		--create --if-not-exists \
		--topic sensor-events \
		--partitions 3 \
		--replication-factor 1
	docker exec kafka kafka-topics --bootstrap-server localhost:9092 \
		--create --if-not-exists \
		--topic processed-events \
		--partitions 3 \
		--replication-factor 1
	@echo "Топики созданы!"

test:
	pytest tests/ -v

lint:
	ruff check src/
	mypy src/

run-producer:
	python -m src.producers.sensor_producer

run-consumer:
	python -m src.consumers.sensor_consumer
```

## Скрипт инициализации топиков

```python
# scripts/init_kafka.py
"""Скрипт для инициализации Kafka топиков."""

from confluent_kafka.admin import AdminClient, NewTopic
import sys
import time


def wait_for_kafka(bootstrap_servers: str, timeout: int = 60) -> bool:
    """Ожидание готовности Kafka."""
    admin = AdminClient({"bootstrap.servers": bootstrap_servers})

    start_time = time.time()
    while time.time() - start_time < timeout:
        try:
            # Попытка получить метаданные кластера
            admin.list_topics(timeout=5)
            print("Kafka готова к работе!")
            return True
        except Exception as e:
            print(f"Ожидание Kafka... ({e})")
            time.sleep(2)

    return False


def create_topics(bootstrap_servers: str, topics: list[dict]) -> None:
    """Создание топиков в Kafka."""
    admin = AdminClient({"bootstrap.servers": bootstrap_servers})

    new_topics = [
        NewTopic(
            topic=t["name"],
            num_partitions=t.get("partitions", 3),
            replication_factor=t.get("replication_factor", 1),
            config=t.get("config", {})
        )
        for t in topics
    ]

    # Создание топиков
    futures = admin.create_topics(new_topics)

    for topic, future in futures.items():
        try:
            future.result()  # Ожидание завершения
            print(f"Топик '{topic}' создан успешно")
        except Exception as e:
            if "already exists" in str(e):
                print(f"Топик '{topic}' уже существует")
            else:
                print(f"Ошибка создания топика '{topic}': {e}")


def main():
    bootstrap_servers = "localhost:9092"

    # Ожидание готовности Kafka
    if not wait_for_kafka(bootstrap_servers):
        print("Kafka недоступна!")
        sys.exit(1)

    # Определение топиков
    topics = [
        {
            "name": "sensor-events",
            "partitions": 6,
            "replication_factor": 1,
            "config": {
                "retention.ms": str(7 * 24 * 60 * 60 * 1000),  # 7 дней
                "cleanup.policy": "delete",
            }
        },
        {
            "name": "processed-events",
            "partitions": 3,
            "replication_factor": 1,
            "config": {
                "retention.ms": str(30 * 24 * 60 * 60 * 1000),  # 30 дней
            }
        },
        {
            "name": "alerts",
            "partitions": 1,
            "replication_factor": 1,
            "config": {
                "retention.ms": str(90 * 24 * 60 * 60 * 1000),  # 90 дней
            }
        },
    ]

    create_topics(bootstrap_servers, topics)


if __name__ == "__main__":
    main()
```

## Best Practices

### Организация проекта

1. **Разделение ответственности**: Разделяйте Producer и Consumer в отдельные модули
2. **Конфигурация через окружение**: Используйте переменные окружения для всех настроек
3. **Централизованная конфигурация**: Храните все настройки Kafka в одном месте
4. **Версионирование**: Используйте семантическое версионирование для релизов

### Docker Compose

1. **Health checks**: Всегда добавляйте проверки здоровья для контейнеров
2. **Volumes**: Используйте именованные volumes для сохранения данных
3. **Сети**: Создавайте изолированные сети для сервисов
4. **Зависимости**: Используйте `depends_on` с условиями для правильного порядка запуска

### Безопасность

1. **Не храните секреты в коде**: Используйте .env файлы или менеджеры секретов
2. **SSL/TLS**: В продакшене всегда используйте шифрование
3. **SASL**: Настраивайте аутентификацию для всех клиентов

### Тестирование

1. **Интеграционные тесты**: Используйте testcontainers для тестов с реальной Kafka
2. **Моки**: Для unit-тестов мокируйте Kafka клиенты
3. **CI/CD**: Автоматизируйте запуск тестов при каждом коммите

## Дополнительные ресурсы

- [Confluent Docker Images](https://hub.docker.com/u/confluentinc)
- [Apache Kafka Documentation](https://kafka.apache.org/documentation/)
- [confluent-kafka-python](https://github.com/confluentinc/confluent-kafka-python)
- [Kafka UI](https://github.com/provectus/kafka-ui)

---

[prev: 06-stream-processing-terminology](../02-getting-acquainted/06-stream-processing-terminology.md) | [next: 02-sensor-events](./02-sensor-events.md)
