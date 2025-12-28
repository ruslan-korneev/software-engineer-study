# Модель зрелости Kafka

[prev: 06-data-at-rest](../10-security/06-data-at-rest.md) | [next: 02-schema-registry](./02-schema-registry.md)

---

## Описание

Модель зрелости Kafka (Kafka Maturity Model) — это фреймворк, который помогает организациям оценить текущий уровень использования Apache Kafka и определить шаги для улучшения их архитектуры событийно-ориентированных систем. Эта модель была разработана Confluent и описывает эволюционный путь от простого использования Kafka как очереди сообщений до полноценной платформы потоковой обработки данных.

Понимание модели зрелости важно для планирования развития инфраструктуры и определения, когда и зачем нужен Schema Registry.

## Ключевые концепции

### Уровни зрелости

#### Уровень 0: Отсутствие событийной архитектуры
- Система построена на синхронных REST/RPC вызовах
- Тесная связанность между сервисами
- Отсутствие событийного мышления в организации

#### Уровень 1: Kafka как очередь сообщений (Message Queue)
- Kafka используется для простой передачи сообщений
- Формат данных: строки, JSON без схемы
- Отсутствие централизованного управления схемами
- Продюсеры и консьюмеры тесно связаны

```
┌──────────────┐     ┌─────────┐     ┌──────────────┐
│   Producer   │────▶│  Kafka  │────▶│   Consumer   │
│  (JSON/str)  │     │  Topic  │     │  (JSON/str)  │
└──────────────┘     └─────────┘     └──────────────┘
```

#### Уровень 2: Введение схем данных
- Использование Schema Registry
- Сериализация с Avro, Protobuf или JSON Schema
- Централизованное хранение и версионирование схем
- Валидация данных на уровне продюсера

```
┌──────────────┐     ┌─────────────────┐     ┌─────────┐
│   Producer   │────▶│ Schema Registry │     │  Kafka  │
│   (Avro)     │     └─────────────────┘     │  Topic  │
└──────────────┘              │              └─────────┘
                              ▼                   │
                    ┌─────────────────┐           │
                    │   Consumer      │◀──────────┘
                    │   (Avro)        │
                    └─────────────────┘
```

#### Уровень 3: Потоковая обработка (Stream Processing)
- Использование Kafka Streams или ksqlDB
- Обработка данных в реальном времени
- Создание производных топиков
- Агрегации, объединения, обогащение данных

#### Уровень 4: Событийно-ориентированная архитектура (Event-Driven Architecture)
- События как первоклассные граждане
- Event Sourcing и CQRS паттерны
- Полный аудит и история изменений
- Слабая связанность между сервисами

#### Уровень 5: Data Mesh и федеративное управление
- Децентрализованное владение данными
- Self-service платформа для команд
- Каталог данных и управление метаданными
- Межорганизационный обмен событиями

## Примеры кода

### Уровень 1: Простой JSON без схемы

```python
from kafka import KafkaProducer, KafkaConsumer
import json

# Producer - отправка JSON без схемы
producer = KafkaProducer(
    bootstrap_servers=['localhost:9092'],
    value_serializer=lambda v: json.dumps(v).encode('utf-8')
)

# Проблема: нет гарантии структуры данных
message = {
    "user_id": 123,
    "action": "login",
    "timestamp": "2024-01-15T10:30:00Z"
}
producer.send('user-events', value=message)

# Consumer - надежда на правильную структуру
consumer = KafkaConsumer(
    'user-events',
    bootstrap_servers=['localhost:9092'],
    value_deserializer=lambda m: json.loads(m.decode('utf-8'))
)

for msg in consumer:
    # Может упасть, если структура изменилась
    print(f"User {msg.value['user_id']} performed {msg.value['action']}")
```

### Уровень 2: С использованием Schema Registry

```python
from confluent_kafka import Producer, Consumer
from confluent_kafka.serialization import SerializationContext, MessageField
from confluent_kafka.schema_registry import SchemaRegistryClient
from confluent_kafka.schema_registry.avro import AvroSerializer, AvroDeserializer

# Определение схемы Avro
user_event_schema = """
{
    "type": "record",
    "name": "UserEvent",
    "namespace": "com.example.events",
    "fields": [
        {"name": "user_id", "type": "long"},
        {"name": "action", "type": "string"},
        {"name": "timestamp", "type": "string"},
        {"name": "metadata", "type": ["null", "string"], "default": null}
    ]
}
"""

# Настройка Schema Registry
schema_registry_conf = {'url': 'http://localhost:8081'}
schema_registry_client = SchemaRegistryClient(schema_registry_conf)

# Сериализатор с автоматической регистрацией схемы
avro_serializer = AvroSerializer(
    schema_registry_client,
    user_event_schema,
    lambda obj, ctx: obj  # to_dict function
)

# Producer с гарантией схемы
producer_conf = {'bootstrap.servers': 'localhost:9092'}
producer = Producer(producer_conf)

def delivery_report(err, msg):
    if err is not None:
        print(f'Delivery failed: {err}')
    else:
        print(f'Message delivered to {msg.topic()}')

# Отправка сообщения с валидацией схемы
event = {
    "user_id": 123,
    "action": "login",
    "timestamp": "2024-01-15T10:30:00Z",
    "metadata": None
}

producer.produce(
    topic='user-events',
    value=avro_serializer(event, SerializationContext('user-events', MessageField.VALUE)),
    on_delivery=delivery_report
)
producer.flush()
```

### Уровень 3: Потоковая обработка с Kafka Streams

```java
import org.apache.kafka.streams.KafkaStreams;
import org.apache.kafka.streams.StreamsBuilder;
import org.apache.kafka.streams.kstream.*;
import io.confluent.kafka.streams.serdes.avro.SpecificAvroSerde;

public class UserEventProcessor {
    public static void main(String[] args) {
        StreamsBuilder builder = new StreamsBuilder();

        // Чтение событий пользователей
        KStream<String, UserEvent> userEvents = builder.stream("user-events");

        // Подсчет действий по типу за последний час
        KTable<Windowed<String>, Long> actionCounts = userEvents
            .groupBy((key, value) -> value.getAction())
            .windowedBy(TimeWindows.ofSizeWithNoGrace(Duration.ofHours(1)))
            .count();

        // Запись результатов в производный топик
        actionCounts.toStream()
            .mapValues((windowedKey, count) ->
                new ActionStats(windowedKey.key(), count, windowedKey.window().startTime()))
            .to("action-stats");

        KafkaStreams streams = new KafkaStreams(builder.build(), getConfig());
        streams.start();
    }
}
```

## Диаграмма уровней зрелости

```
                            ┌─────────────────────────────────┐
                            │   Уровень 5: Data Mesh          │
                            │   • Федеративное управление     │
                            │   • Self-service платформа      │
                            └─────────────────────────────────┘
                                           ▲
                            ┌─────────────────────────────────┐
                            │   Уровень 4: Event-Driven       │
                            │   • Event Sourcing              │
                            │   • CQRS                        │
                            └─────────────────────────────────┘
                                           ▲
                            ┌─────────────────────────────────┐
                            │   Уровень 3: Stream Processing  │
                            │   • Kafka Streams / ksqlDB      │
                            │   • Real-time analytics         │
                            └─────────────────────────────────┘
                                           ▲
                            ┌─────────────────────────────────┐
       Schema Registry ───▶ │   Уровень 2: Schema Management  │
       необходим здесь      │   • Avro/Protobuf/JSON Schema   │
                            │   • Версионирование схем        │
                            └─────────────────────────────────┘
                                           ▲
                            ┌─────────────────────────────────┐
                            │   Уровень 1: Message Queue      │
                            │   • Простая передача сообщений  │
                            │   • JSON без схемы              │
                            └─────────────────────────────────┘
                                           ▲
                            ┌─────────────────────────────────┐
                            │   Уровень 0: Нет событий        │
                            │   • REST/RPC только             │
                            └─────────────────────────────────┘
```

## Best Practices

### Оценка текущего уровня

1. **Проведите аудит использования Kafka**:
   - Какие форматы данных используются?
   - Есть ли централизованное управление схемами?
   - Используется ли потоковая обработка?

2. **Определите болевые точки**:
   - Частые ошибки десериализации
   - Сложность с версионированием API
   - Отсутствие документации структуры данных

3. **Составьте план миграции**:
   - Переход между уровнями должен быть постепенным
   - Начните с критичных топиков
   - Обеспечьте обратную совместимость

### Когда переходить на следующий уровень

| Признак | Рекомендуемое действие |
|---------|------------------------|
| Ошибки из-за изменения формата | Внедрить Schema Registry (уровень 2) |
| Необходимость real-time аналитики | Добавить Kafka Streams (уровень 3) |
| Требуется полный аудит изменений | Перейти к Event Sourcing (уровень 4) |
| Много команд работают с Kafka | Рассмотреть Data Mesh (уровень 5) |

### Типичные ошибки при переходе

1. **Пропуск уровней**: Попытка сразу внедрить Event Sourcing без Schema Registry
2. **Big Bang миграция**: Попытка перевести все топики за раз
3. **Игнорирование обратной совместимости**: Ломающие изменения схем
4. **Недостаточное обучение команды**: Внедрение новых паттернов без подготовки

## Связь с Schema Registry

Schema Registry является ключевым компонентом для перехода с уровня 1 на уровень 2 и выше. Он обеспечивает:

- **Централизованное хранение схем** для всех топиков
- **Версионирование** с поддержкой эволюции схем
- **Валидацию** данных перед отправкой в Kafka
- **Документацию** структуры данных для всех команд

Без Schema Registry невозможно построить надежную событийно-ориентированную архитектуру на более высоких уровнях зрелости.

---

[prev: 06-data-at-rest](../10-security/06-data-at-rest.md) | [next: 02-schema-registry](./02-schema-registry.md)
