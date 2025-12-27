# ksqlDB

## Описание

**ksqlDB** - это база данных для потоковой обработки, построенная поверх Apache Kafka. Она позволяет выполнять SQL-запросы над потоками данных в реальном времени без написания кода на Java или Scala.

### Основные возможности

- **SQL-синтаксис** для обработки потоков данных
- **Материализованные представления** с автоматическим обновлением
- **Pull и Push запросы** для получения данных
- **Встроенные коннекторы** для интеграции с внешними системами
- **REST API** для управления и выполнения запросов

### Архитектура

```
┌─────────────────────────────────────────────────┐
│                   ksqlDB Server                  │
├─────────────────────────────────────────────────┤
│  SQL Engine  │  Query Planner  │  Executor      │
├─────────────────────────────────────────────────┤
│              Kafka Streams Library               │
├─────────────────────────────────────────────────┤
│                  Apache Kafka                    │
└─────────────────────────────────────────────────┘
```

## Ключевые концепции

### Streams vs Tables

| Концепция | STREAM | TABLE |
|-----------|--------|-------|
| Семантика | Append-only (события) | Upsert (состояние) |
| Аналогия | INSERT | INSERT/UPDATE |
| Ключи | Могут повторяться | Уникальные (последнее значение) |
| Применение | События, логи | Справочники, агрегаты |

### Типы запросов

1. **Push Query** - подписка на непрерывный поток результатов
   ```sql
   SELECT * FROM stream EMIT CHANGES;
   ```

2. **Pull Query** - однократное получение текущего состояния
   ```sql
   SELECT * FROM table WHERE id = 'key123';
   ```

### Материализованные представления

ksqlDB автоматически поддерживает актуальность материализованных представлений при поступлении новых данных.

## Примеры кода

### Установка и запуск

```bash
# Docker Compose для ksqlDB
version: '3'
services:
  ksqldb-server:
    image: confluentinc/ksqldb-server:0.29.0
    hostname: ksqldb-server
    depends_on:
      - kafka
    ports:
      - "8088:8088"
    environment:
      KSQL_LISTENERS: http://0.0.0.0:8088
      KSQL_BOOTSTRAP_SERVERS: kafka:9092
      KSQL_KSQL_LOGGING_PROCESSING_STREAM_AUTO_CREATE: "true"
      KSQL_KSQL_LOGGING_PROCESSING_TOPIC_AUTO_CREATE: "true"

  ksqldb-cli:
    image: confluentinc/ksqldb-cli:0.29.0
    depends_on:
      - ksqldb-server
    entrypoint: /bin/sh
    tty: true
```

### Подключение к ksqlDB CLI

```bash
# Подключение к CLI
docker exec -it ksqldb-cli ksql http://ksqldb-server:8088

# Или напрямую
ksql http://localhost:8088
```

### Создание Stream

```sql
-- Создание стрима из существующего топика
CREATE STREAM orders (
    order_id VARCHAR KEY,
    customer_id VARCHAR,
    product_id VARCHAR,
    quantity INT,
    price DECIMAL(10,2),
    order_time TIMESTAMP
) WITH (
    KAFKA_TOPIC = 'orders',
    VALUE_FORMAT = 'JSON',
    TIMESTAMP = 'order_time'
);

-- Создание стрима с автоматическим созданием топика
CREATE STREAM pageviews (
    user_id VARCHAR KEY,
    page_url VARCHAR,
    view_time TIMESTAMP
) WITH (
    KAFKA_TOPIC = 'pageviews',
    VALUE_FORMAT = 'AVRO',
    PARTITIONS = 6,
    REPLICAS = 3
);
```

### Создание Table

```sql
-- Создание таблицы из топика
CREATE TABLE users (
    user_id VARCHAR PRIMARY KEY,
    name VARCHAR,
    email VARCHAR,
    created_at TIMESTAMP
) WITH (
    KAFKA_TOPIC = 'users',
    VALUE_FORMAT = 'JSON'
);

-- Создание таблицы с агрегацией (материализованное представление)
CREATE TABLE order_counts AS
    SELECT customer_id,
           COUNT(*) AS total_orders,
           SUM(price * quantity) AS total_amount
    FROM orders
    GROUP BY customer_id
    EMIT CHANGES;
```

### Базовые запросы

```sql
-- Push Query: непрерывный поток
SELECT * FROM orders EMIT CHANGES;

-- Push Query с фильтрацией
SELECT order_id, customer_id, price
FROM orders
WHERE price > 100
EMIT CHANGES;

-- Pull Query: получение текущего состояния
SELECT * FROM order_counts WHERE customer_id = 'CUST001';

-- Просмотр всех стримов и таблиц
SHOW STREAMS;
SHOW TABLES;

-- Описание структуры
DESCRIBE orders;
DESCRIBE EXTENDED order_counts;
```

### Трансформации

```sql
-- Фильтрация
CREATE STREAM large_orders AS
    SELECT *
    FROM orders
    WHERE quantity * price > 1000
    EMIT CHANGES;

-- Проекция и трансформация
CREATE STREAM order_summary AS
    SELECT order_id,
           customer_id,
           quantity * price AS total,
           UCASE(product_id) AS product_id_upper
    FROM orders
    EMIT CHANGES;

-- Flatten вложенных структур
CREATE STREAM flattened AS
    SELECT order_id,
           address->city AS city,
           address->country AS country
    FROM orders_nested
    EMIT CHANGES;
```

### Агрегации

```sql
-- Простой COUNT
CREATE TABLE orders_per_customer AS
    SELECT customer_id,
           COUNT(*) AS order_count
    FROM orders
    GROUP BY customer_id
    EMIT CHANGES;

-- Множественные агрегации
CREATE TABLE customer_stats AS
    SELECT customer_id,
           COUNT(*) AS total_orders,
           SUM(quantity) AS total_items,
           AVG(price) AS avg_price,
           MIN(price) AS min_price,
           MAX(price) AS max_price
    FROM orders
    GROUP BY customer_id
    EMIT CHANGES;

-- HAVING для фильтрации агрегатов
CREATE TABLE vip_customers AS
    SELECT customer_id,
           COUNT(*) AS order_count,
           SUM(price * quantity) AS total_spent
    FROM orders
    GROUP BY customer_id
    HAVING SUM(price * quantity) > 10000
    EMIT CHANGES;
```

### Windowing (оконные операции)

```sql
-- Tumbling Window: непересекающиеся окна
CREATE TABLE orders_per_hour AS
    SELECT customer_id,
           COUNT(*) AS order_count,
           WINDOWSTART AS window_start,
           WINDOWEND AS window_end
    FROM orders
    WINDOW TUMBLING (SIZE 1 HOUR)
    GROUP BY customer_id
    EMIT CHANGES;

-- Hopping Window: скользящие окна
CREATE TABLE orders_sliding AS
    SELECT product_id,
           SUM(quantity) AS total_quantity
    FROM orders
    WINDOW HOPPING (SIZE 10 MINUTES, ADVANCE BY 1 MINUTE)
    GROUP BY product_id
    EMIT CHANGES;

-- Session Window: сессионные окна
CREATE TABLE user_sessions AS
    SELECT user_id,
           COUNT(*) AS actions_count,
           WINDOWSTART AS session_start,
           WINDOWEND AS session_end
    FROM user_actions
    WINDOW SESSION (30 MINUTES)
    GROUP BY user_id
    EMIT CHANGES;
```

### JOIN операции

```sql
-- Stream-Table Join: обогащение потока данными из таблицы
CREATE STREAM enriched_orders AS
    SELECT o.order_id,
           o.product_id,
           o.quantity,
           o.price,
           u.name AS customer_name,
           u.email AS customer_email
    FROM orders o
    INNER JOIN users u ON o.customer_id = u.user_id
    EMIT CHANGES;

-- Stream-Stream Join: объединение двух потоков
CREATE STREAM orders_with_shipments AS
    SELECT o.order_id,
           o.customer_id,
           s.shipment_id,
           s.carrier,
           s.tracking_number
    FROM orders o
    INNER JOIN shipments s
        WITHIN 7 DAYS
        ON o.order_id = s.order_id
    EMIT CHANGES;

-- Table-Table Join
CREATE TABLE customer_orders_summary AS
    SELECT c.customer_id,
           c.name,
           o.total_orders,
           o.total_amount
    FROM customers c
    LEFT JOIN order_counts o ON c.customer_id = o.customer_id
    EMIT CHANGES;
```

### Встроенные функции

```sql
-- Строковые функции
SELECT UCASE(name),
       LCASE(email),
       SUBSTRING(product_id, 1, 3),
       CONCAT(first_name, ' ', last_name),
       TRIM(description),
       REPLACE(text, 'old', 'new')
FROM stream_name
EMIT CHANGES;

-- Математические функции
SELECT ABS(value),
       ROUND(price, 2),
       CEIL(amount),
       FLOOR(amount),
       POWER(base, exponent)
FROM stream_name
EMIT CHANGES;

-- Функции даты/времени
SELECT UNIX_TIMESTAMP(),
       TIMESTAMPTOSTRING(order_time, 'yyyy-MM-dd HH:mm:ss'),
       STRINGTOTIMESTAMP(date_str, 'yyyy-MM-dd'),
       DATEADD(DAYS, 7, order_date)
FROM orders
EMIT CHANGES;

-- Условные функции
SELECT order_id,
       CASE
           WHEN price > 1000 THEN 'premium'
           WHEN price > 100 THEN 'standard'
           ELSE 'basic'
       END AS tier,
       COALESCE(discount, 0) AS discount,
       IFNULL(notes, 'No notes') AS notes
FROM orders
EMIT CHANGES;
```

### User-Defined Functions (UDF)

```java
// Создание UDF на Java
import io.confluent.ksql.function.udf.Udf;
import io.confluent.ksql.function.udf.UdfDescription;
import io.confluent.ksql.function.udf.UdfParameter;

@UdfDescription(
    name = "calculate_discount",
    description = "Calculates discount based on quantity"
)
public class CalculateDiscountUdf {

    @Udf(description = "Calculate discount percentage")
    public double calculateDiscount(
        @UdfParameter(value = "quantity") final int quantity,
        @UdfParameter(value = "price") final double price
    ) {
        if (quantity > 100) return price * 0.20;
        if (quantity > 50) return price * 0.10;
        if (quantity > 10) return price * 0.05;
        return 0;
    }
}
```

```sql
-- Использование UDF
SELECT order_id,
       price,
       quantity,
       calculate_discount(quantity, price) AS discount
FROM orders
EMIT CHANGES;
```

### Connectors

```sql
-- Создание Source Connector (чтение из внешней системы)
CREATE SOURCE CONNECTOR jdbc_source WITH (
    'connector.class' = 'io.confluent.connect.jdbc.JdbcSourceConnector',
    'connection.url' = 'jdbc:postgresql://postgres:5432/mydb',
    'connection.user' = 'user',
    'connection.password' = 'password',
    'table.whitelist' = 'users,products',
    'mode' = 'incrementing',
    'incrementing.column.name' = 'id',
    'topic.prefix' = 'jdbc_'
);

-- Создание Sink Connector (запись во внешнюю систему)
CREATE SINK CONNECTOR elasticsearch_sink WITH (
    'connector.class' = 'io.confluent.connect.elasticsearch.ElasticsearchSinkConnector',
    'connection.url' = 'http://elasticsearch:9200',
    'topics' = 'enriched_orders',
    'type.name' = 'order',
    'key.ignore' = 'true',
    'schema.ignore' = 'true'
);

-- Просмотр коннекторов
SHOW CONNECTORS;
DESCRIBE CONNECTOR jdbc_source;

-- Удаление коннектора
DROP CONNECTOR jdbc_source;
```

### REST API

```bash
# Проверка здоровья сервера
curl http://localhost:8088/healthcheck

# Получение информации о сервере
curl http://localhost:8088/info

# Выполнение запроса
curl -X POST http://localhost:8088/ksql \
  -H "Content-Type: application/vnd.ksql.v1+json" \
  -d '{
    "ksql": "SELECT * FROM orders EMIT CHANGES;",
    "streamsProperties": {}
  }'

# Получение статуса запроса
curl http://localhost:8088/status

# Завершение запроса
curl -X POST http://localhost:8088/ksql \
  -H "Content-Type: application/vnd.ksql.v1+json" \
  -d '{
    "ksql": "TERMINATE QUERY_ID;"
  }'
```

### Python клиент

```python
from ksql import KSQLAPI

# Подключение
client = KSQLAPI('http://localhost:8088')

# Выполнение statement
client.ksql('CREATE STREAM test_stream (id VARCHAR KEY, value VARCHAR) '
            'WITH (KAFKA_TOPIC=\'test\', VALUE_FORMAT=\'JSON\');')

# Pull Query
result = client.query('SELECT * FROM users_table WHERE user_id = \'123\';')
for row in result:
    print(row)

# Push Query (streaming)
for row in client.query('SELECT * FROM orders EMIT CHANGES;', stream=True):
    print(row)
```

## Best Practices

### Проектирование схемы

1. **Выбирайте правильный тип данных**
   ```sql
   -- Используйте KEY для ключей партиционирования
   CREATE STREAM orders (
       order_id VARCHAR KEY,  -- Это ключ сообщения в Kafka
       data VARCHAR
   ) WITH (...);
   ```

2. **Используйте STRUCT для вложенных объектов**
   ```sql
   CREATE STREAM orders (
       order_id VARCHAR KEY,
       customer STRUCT<
           id VARCHAR,
           name VARCHAR,
           address STRUCT<
               city VARCHAR,
               country VARCHAR
           >
       >
   ) WITH (VALUE_FORMAT = 'JSON', ...);
   ```

3. **Правильно настраивайте TIMESTAMP**
   ```sql
   CREATE STREAM events (
       event_id VARCHAR KEY,
       event_type VARCHAR,
       event_time TIMESTAMP
   ) WITH (
       TIMESTAMP = 'event_time',  -- Использовать для windowing
       ...
   );
   ```

### Оптимизация производительности

1. **Используйте Pull Queries для точечных запросов**
   ```sql
   -- Эффективно: точечный запрос по ключу
   SELECT * FROM users_table WHERE user_id = 'USER123';

   -- Менее эффективно: scan всей таблицы
   SELECT * FROM users_table;
   ```

2. **Ограничивайте результаты Push Queries**
   ```sql
   -- Используйте LIMIT для отладки
   SELECT * FROM orders EMIT CHANGES LIMIT 10;
   ```

3. **Настройка параллелизма**
   ```sql
   -- Установка количества партиций
   CREATE STREAM high_volume_stream (...)
   WITH (PARTITIONS = 12, ...);
   ```

### Обработка ошибок

```sql
-- Настройка обработки ошибок десериализации
SET 'ksql.fail.on.deserialization.error' = 'false';

-- Логирование ошибок в отдельный топик
SET 'ksql.logging.processing.topic.auto.create' = 'true';
```

### Мониторинг

```sql
-- Просмотр активных запросов
SHOW QUERIES;

-- Подробности о запросе
EXPLAIN <query_id>;

-- Метрики запроса
DESCRIBE EXTENDED <stream_or_table>;
```

### Управление запросами

```sql
-- Остановка запроса
TERMINATE QUERY_ID;

-- Удаление стрима (без удаления топика)
DROP STREAM IF EXISTS stream_name;

-- Удаление стрима с топиком
DROP STREAM IF EXISTS stream_name DELETE TOPIC;

-- Удаление таблицы
DROP TABLE IF EXISTS table_name;
```

## Сравнение с Kafka Streams

| Аспект | ksqlDB | Kafka Streams |
|--------|--------|---------------|
| Язык | SQL | Java/Scala |
| Развертывание | Отдельный сервер | Встроенная библиотека |
| Сложность | Низкая | Средняя/Высокая |
| Гибкость | Ограниченная | Высокая |
| Отладка | Интерактивная | Требует IDE |
| Use Cases | Ad-hoc анализ, ETL | Сложная бизнес-логика |

### Когда использовать ksqlDB

- Быстрое прототипирование
- ETL и трансформации данных
- Обогащение потоков
- Материализованные представления
- Ad-hoc анализ данных

### Когда использовать Kafka Streams

- Сложная бизнес-логика
- Интеграция с существующим Java-кодом
- Максимальный контроль над обработкой
- Кастомные операции с состоянием
