# Упражнение

[prev: 06-stateful-systems](./06-stateful-systems.md) | [next: 01-topics](../07-topics-partitions/01-topics.md)

---

## Описание

Практические упражнения для закрепления знаний о брокерах Kafka. Упражнения охватывают настройку кластера, репликацию, мониторинг и работу с отказами.

## Подготовка окружения

### Docker Compose для локального кластера

```yaml
# docker-compose.yml
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

  kafka1:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka1
    container_name: kafka1
    depends_on:
      - zookeeper
    ports:
      - "9092:9092"
    environment:
      KAFKA_BROKER_ID: 1
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka1:29092,PLAINTEXT_HOST://localhost:9092
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_NUM_PARTITIONS: 3

  kafka2:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka2
    container_name: kafka2
    depends_on:
      - zookeeper
    ports:
      - "9093:9093"
    environment:
      KAFKA_BROKER_ID: 2
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka2:29093,PLAINTEXT_HOST://localhost:9093
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_NUM_PARTITIONS: 3

  kafka3:
    image: confluentinc/cp-kafka:7.5.0
    hostname: kafka3
    container_name: kafka3
    depends_on:
      - zookeeper
    ports:
      - "9094:9094"
    environment:
      KAFKA_BROKER_ID: 3
      KAFKA_ZOOKEEPER_CONNECT: 'zookeeper:2181'
      KAFKA_LISTENER_SECURITY_PROTOCOL_MAP: PLAINTEXT:PLAINTEXT,PLAINTEXT_HOST:PLAINTEXT
      KAFKA_ADVERTISED_LISTENERS: PLAINTEXT://kafka3:29094,PLAINTEXT_HOST://localhost:9094
      KAFKA_OFFSETS_TOPIC_REPLICATION_FACTOR: 3
      KAFKA_DEFAULT_REPLICATION_FACTOR: 3
      KAFKA_MIN_INSYNC_REPLICAS: 2
      KAFKA_NUM_PARTITIONS: 3
```

### Запуск кластера

```bash
# Запустить кластер
docker-compose up -d

# Проверить статус
docker-compose ps

# Просмотреть логи
docker-compose logs -f kafka1
```

---

## Упражнение 1: Исследование кластера

### Задача

Изучить структуру кластера Kafka и его компоненты.

### Шаги

```bash
# 1. Подключиться к контейнеру kafka1
docker exec -it kafka1 bash

# 2. Проверить список брокеров
kafka-broker-api-versions --bootstrap-server localhost:29092

# 3. Получить информацию о кластере
kafka-metadata --snapshot /var/lib/kafka/data/__cluster_metadata-0/00000000000000000000.log --command "cat /brokers" 2>/dev/null || \
  echo "Используйте ZooKeeper для получения метаданных"

# 4. Через ZooKeeper (если не KRaft)
# Выйти из контейнера kafka и зайти в zookeeper
docker exec -it zookeeper bash
zookeeper-shell localhost:2181 <<< "ls /brokers/ids"
zookeeper-shell localhost:2181 <<< "get /controller"
```

### Вопросы для самопроверки

1. Сколько брокеров в кластере?
2. Какой брокер является контроллером?
3. Какой broker.id у каждого брокера?

---

## Упражнение 2: Создание топика и изучение репликации

### Задача

Создать топик с репликацией и изучить распределение партиций.

### Шаги

```bash
# 1. Создать топик с 3 партициями и RF=3
docker exec kafka1 kafka-topics --create \
  --bootstrap-server localhost:29092 \
  --topic test-replication \
  --partitions 3 \
  --replication-factor 3 \
  --config min.insync.replicas=2

# 2. Описать топик
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic test-replication

# Пример вывода:
# Topic: test-replication    Partition: 0    Leader: 1    Replicas: 1,2,3    Isr: 1,2,3
# Topic: test-replication    Partition: 1    Leader: 2    Replicas: 2,3,1    Isr: 2,3,1
# Topic: test-replication    Partition: 2    Leader: 3    Replicas: 3,1,2    Isr: 3,1,2

# 3. Записать сообщения
docker exec -it kafka1 kafka-console-producer \
  --bootstrap-server localhost:29092 \
  --topic test-replication \
  --property "parse.key=true" \
  --property "key.separator=:"
# Введите:
# key1:message1
# key2:message2
# key3:message3
# Ctrl+C для выхода

# 4. Прочитать сообщения
docker exec kafka1 kafka-console-consumer \
  --bootstrap-server localhost:29092 \
  --topic test-replication \
  --from-beginning \
  --max-messages 3
```

### Вопросы для самопроверки

1. Как распределены лидеры по брокерам?
2. Что такое ISR и кто входит в ISR для каждой партиции?
3. Что означает Replicas в выводе describe?

---

## Упражнение 3: Симуляция отказа брокера

### Задача

Изучить поведение кластера при отказе брокера.

### Шаги

```bash
# 1. Проверить текущее состояние
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic test-replication

# 2. Запомнить, кто лидер партиции 0
# Допустим, лидер = broker 1

# 3. Остановить брокер-лидера
docker stop kafka1

# 4. Подождать 5-10 секунд и проверить состояние
docker exec kafka2 kafka-topics --describe \
  --bootstrap-server localhost:29093 \
  --topic test-replication

# Ожидаемый результат: новый лидер выбран из ISR

# 5. Проверить, можем ли читать данные
docker exec kafka2 kafka-console-consumer \
  --bootstrap-server localhost:29093 \
  --topic test-replication \
  --from-beginning \
  --max-messages 3

# 6. Проверить, можем ли писать
docker exec -it kafka2 kafka-console-producer \
  --bootstrap-server localhost:29093 \
  --topic test-replication
# Введите: test-after-failure
# Ctrl+C

# 7. Запустить брокер обратно
docker start kafka1

# 8. Дождаться синхронизации и проверить ISR
sleep 30
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic test-replication
```

### Вопросы для самопроверки

1. Сколько времени заняло переключение лидера?
2. Были ли потеряны данные?
3. Вернулся ли брокер в ISR после перезапуска?

---

## Упражнение 4: Проверка min.insync.replicas

### Задача

Понять, как работает min.insync.replicas.

### Шаги

```bash
# 1. Проверить текущие настройки топика
docker exec kafka1 kafka-configs --describe \
  --bootstrap-server localhost:29092 \
  --entity-type topics \
  --entity-name test-replication

# 2. Остановить два брокера (оставить только один)
docker stop kafka2 kafka3

# 3. Попробовать записать сообщение с acks=all
docker exec kafka1 kafka-console-producer \
  --bootstrap-server localhost:29092 \
  --topic test-replication \
  --producer-property acks=all
# Введите сообщение и наблюдайте ошибку:
# ERROR: NotEnoughReplicasException

# 4. Попробовать с acks=1
docker exec kafka1 kafka-console-producer \
  --bootstrap-server localhost:29092 \
  --topic test-replication \
  --producer-property acks=1
# Это должно работать (менее надёжно)

# 5. Запустить брокеры обратно
docker start kafka2 kafka3
```

### Вопросы для самопроверки

1. Почему запись с acks=all не работает при одном живом брокере?
2. В чём разница между acks=all и acks=1?
3. Когда использовать min.insync.replicas=2 vs min.insync.replicas=1?

---

## Упражнение 5: Перебалансировка лидеров

### Задача

Изучить механизм preferred leader election.

### Шаги

```bash
# 1. Проверить текущих лидеров
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic test-replication

# 2. Остановить и запустить один из брокеров
docker stop kafka2
sleep 10
docker start kafka2
sleep 30

# 3. Проверить лидеров снова
# Некоторые партиции могут иметь "неоптимальных" лидеров
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic test-replication

# 4. Выполнить preferred leader election
docker exec kafka1 kafka-leader-election \
  --bootstrap-server localhost:29092 \
  --election-type PREFERRED \
  --topic test-replication

# 5. Проверить результат
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic test-replication
```

### Вопросы для самопроверки

1. Что такое preferred leader?
2. Почему автоматическая перебалансировка не всегда происходит?
3. Как настроить автоматическую перебалансировку?

---

## Упражнение 6: Мониторинг через JMX

### Задача

Настроить базовый мониторинг брокера.

### Обновлённый docker-compose.yml

```yaml
# Добавить в environment каждого kafka:
KAFKA_JMX_PORT: 9999
KAFKA_JMX_HOSTNAME: localhost

# Добавить порты:
ports:
  - "9092:9092"
  - "9999:9999"  # JMX
```

### Шаги

```bash
# 1. Перезапустить с JMX
docker-compose down
# Обновить docker-compose.yml
docker-compose up -d

# 2. Использовать jmxterm или JConsole для подключения
# Или использовать kafka-run-class для получения метрик

# 3. Посмотреть ключевые метрики через API
docker exec kafka1 kafka-broker-api-versions \
  --bootstrap-server localhost:29092

# 4. Проверить under-replicated партиции
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --under-replicated-partitions

# 5. Проверить offline партиции
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --unavailable-partitions
```

---

## Упражнение 7: Работа с конфигурацией

### Задача

Изучить динамическое изменение конфигурации.

### Шаги

```bash
# 1. Посмотреть текущую конфигурацию брокера
docker exec kafka1 kafka-configs --describe \
  --bootstrap-server localhost:29092 \
  --entity-type brokers \
  --entity-name 1

# 2. Изменить динамическую настройку
docker exec kafka1 kafka-configs --alter \
  --bootstrap-server localhost:29092 \
  --entity-type brokers \
  --entity-name 1 \
  --add-config log.cleaner.threads=2

# 3. Проверить изменение
docker exec kafka1 kafka-configs --describe \
  --bootstrap-server localhost:29092 \
  --entity-type brokers \
  --entity-name 1

# 4. Изменить настройку топика
docker exec kafka1 kafka-configs --alter \
  --bootstrap-server localhost:29092 \
  --entity-type topics \
  --entity-name test-replication \
  --add-config retention.ms=3600000

# 5. Проверить
docker exec kafka1 kafka-configs --describe \
  --bootstrap-server localhost:29092 \
  --entity-type topics \
  --entity-name test-replication

# 6. Удалить динамическую настройку
docker exec kafka1 kafka-configs --alter \
  --bootstrap-server localhost:29092 \
  --entity-type brokers \
  --entity-name 1 \
  --delete-config log.cleaner.threads
```

---

## Упражнение 8: Перераспределение партиций

### Задача

Выполнить ручное перераспределение партиций.

### Шаги

```bash
# 1. Создать файл с планом перераспределения
cat > /tmp/reassignment.json << 'EOF'
{
  "version": 1,
  "partitions": [
    {"topic": "test-replication", "partition": 0, "replicas": [3, 2, 1]},
    {"topic": "test-replication", "partition": 1, "replicas": [1, 3, 2]},
    {"topic": "test-replication", "partition": 2, "replicas": [2, 1, 3]}
  ]
}
EOF

# 2. Скопировать в контейнер
docker cp /tmp/reassignment.json kafka1:/tmp/

# 3. Выполнить перераспределение
docker exec kafka1 kafka-reassign-partitions \
  --bootstrap-server localhost:29092 \
  --reassignment-json-file /tmp/reassignment.json \
  --execute

# 4. Проверить статус
docker exec kafka1 kafka-reassign-partitions \
  --bootstrap-server localhost:29092 \
  --reassignment-json-file /tmp/reassignment.json \
  --verify

# 5. Проверить результат
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic test-replication
```

---

## Упражнение 9: Unclean Leader Election

### Задача

Понять последствия unclean leader election.

### Шаги

```bash
# 1. Создать топик БЕЗ unclean leader election (по умолчанию)
docker exec kafka1 kafka-topics --create \
  --bootstrap-server localhost:29092 \
  --topic test-unclean \
  --partitions 1 \
  --replication-factor 3

# 2. Записать данные
docker exec kafka1 kafka-console-producer \
  --bootstrap-server localhost:29092 \
  --topic test-unclean <<< "important-data"

# 3. Проверить лидера
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic test-unclean

# 4. Остановить лидера и ещё один брокер (оставить только OSR)
# Определите, какие брокеры в ISR
docker stop kafka1 kafka2
# (предполагая, что kafka3 был OSR)

# 5. Попробовать читать
docker exec kafka3 kafka-console-consumer \
  --bootstrap-server localhost:29094 \
  --topic test-unclean \
  --from-beginning \
  --timeout-ms 5000
# Ошибка: партиция недоступна

# 6. Запустить брокеры
docker start kafka1 kafka2
```

### Вопросы для самопроверки

1. Почему партиция стала недоступной?
2. Что произойдёт, если включить unclean.leader.election.enable=true?
3. Когда может понадобиться unclean leader election?

---

## Упражнение 10: Комплексный сценарий

### Задача

Выполнить полный цикл операций с кластером.

### Сценарий

```
1. Создать production-like топик
2. Нагрузить его данными
3. Симулировать отказ одного брокера
4. Добавить "новый" брокер (перезапуск с очисткой данных)
5. Перераспределить партиции
6. Проверить целостность данных
```

### Шаги

```bash
# 1. Создать топик
docker exec kafka1 kafka-topics --create \
  --bootstrap-server localhost:29092 \
  --topic production-topic \
  --partitions 6 \
  --replication-factor 3 \
  --config min.insync.replicas=2 \
  --config retention.ms=604800000

# 2. Нагрузить данными (1000 сообщений)
docker exec kafka1 bash -c '
for i in $(seq 1 1000); do
  echo "message-$i"
done | kafka-console-producer \
  --bootstrap-server localhost:29092 \
  --topic production-topic'

# 3. Проверить количество сообщений
docker exec kafka1 kafka-run-class kafka.tools.GetOffsetShell \
  --broker-list localhost:29092 \
  --topic production-topic

# 4. Симулировать отказ
docker stop kafka3

# 5. Проверить состояние
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic production-topic \
  --under-replicated-partitions

# 6. Продолжить запись
docker exec kafka1 bash -c '
for i in $(seq 1001 1100); do
  echo "message-$i"
done | kafka-console-producer \
  --bootstrap-server localhost:29092 \
  --topic production-topic'

# 7. Запустить брокер
docker start kafka3

# 8. Дождаться синхронизации
sleep 60

# 9. Проверить ISR
docker exec kafka1 kafka-topics --describe \
  --bootstrap-server localhost:29092 \
  --topic production-topic

# 10. Прочитать все сообщения для проверки
docker exec kafka1 kafka-console-consumer \
  --bootstrap-server localhost:29092 \
  --topic production-topic \
  --from-beginning \
  --timeout-ms 10000 | wc -l
# Должно быть 1100
```

---

## Очистка

```bash
# Остановить и удалить все контейнеры
docker-compose down -v

# Удалить все данные
docker volume prune -f
```

## Best Practices для практики

1. **Ведите журнал экспериментов** — записывайте команды и результаты
2. **Используйте отдельное окружение** — никогда не экспериментируйте на production
3. **Изучайте логи** — `docker-compose logs -f kafka1` поможет понять происходящее
4. **Тестируйте граничные случаи** — что если все брокеры упадут?
5. **Практикуйте восстановление** — умение восстановить кластер важнее умения его настроить

---

[prev: 06-stateful-systems](./06-stateful-systems.md) | [next: 01-topics](../07-topics-partitions/01-topics.md)