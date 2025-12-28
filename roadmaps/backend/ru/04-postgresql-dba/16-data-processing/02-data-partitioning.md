# Партиционирование таблиц (Data Partitioning)

[prev: 01-bulk-loading](./01-bulk-loading.md) | [next: 03-sharding-patterns](./03-sharding-patterns.md)

---

## Введение

Партиционирование (секционирование) - это разделение большой таблицы на несколько меньших физических частей (партиций), которые логически представляются как единая таблица. Это позволяет улучшить производительность запросов и упростить управление данными.

## Зачем нужно партиционирование

### Преимущества

1. **Производительность запросов** - сканирование только нужных партиций (partition pruning)
2. **Эффективное удаление данных** - DROP PARTITION вместо DELETE
3. **Параллельное выполнение** - запросы могут обрабатывать партиции параллельно
4. **Упрощенное обслуживание** - VACUUM и REINDEX для отдельных партиций
5. **Разные storage-опции** - партиции могут храниться на разных tablespace

### Когда использовать

- Таблицы с миллионами/миллиардами записей
- Данные с временной составляющей (логи, события, транзакции)
- Данные с естественным разделением (по регионам, клиентам)
- Необходимость быстрого удаления старых данных

## Типы партиционирования в PostgreSQL

### 1. Range Partitioning (по диапазону)

Разделение по диапазону значений - идеально для временных данных.

```sql
-- Создание партиционированной таблицы по дате
CREATE TABLE orders (
    id BIGSERIAL,
    customer_id INTEGER NOT NULL,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10,2),
    status VARCHAR(20)
) PARTITION BY RANGE (order_date);

-- Создание партиций по месяцам
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE orders_2024_02 PARTITION OF orders
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

CREATE TABLE orders_2024_03 PARTITION OF orders
    FOR VALUES FROM ('2024-03-01') TO ('2024-04-01');

-- Партиция по умолчанию для данных вне диапазонов
CREATE TABLE orders_default PARTITION OF orders DEFAULT;
```

### 2. List Partitioning (по списку)

Разделение по конкретным значениям - для категориальных данных.

```sql
-- Партиционирование по региону
CREATE TABLE customers (
    id BIGSERIAL,
    name VARCHAR(100),
    email VARCHAR(255),
    region VARCHAR(20) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
) PARTITION BY LIST (region);

-- Создание партиций для каждого региона
CREATE TABLE customers_europe PARTITION OF customers
    FOR VALUES IN ('UK', 'DE', 'FR', 'IT', 'ES');

CREATE TABLE customers_asia PARTITION OF customers
    FOR VALUES IN ('CN', 'JP', 'KR', 'IN');

CREATE TABLE customers_americas PARTITION OF customers
    FOR VALUES IN ('US', 'CA', 'BR', 'MX');

CREATE TABLE customers_other PARTITION OF customers DEFAULT;
```

### 3. Hash Partitioning (по хешу)

Равномерное распределение данных - когда нет естественного ключа партиционирования.

```sql
-- Партиционирование по хешу customer_id
CREATE TABLE transactions (
    id BIGSERIAL,
    customer_id INTEGER NOT NULL,
    amount DECIMAL(10,2),
    transaction_date TIMESTAMP
) PARTITION BY HASH (customer_id);

-- Создание 4 партиций с равномерным распределением
CREATE TABLE transactions_p0 PARTITION OF transactions
    FOR VALUES WITH (MODULUS 4, REMAINDER 0);

CREATE TABLE transactions_p1 PARTITION OF transactions
    FOR VALUES WITH (MODULUS 4, REMAINDER 1);

CREATE TABLE transactions_p2 PARTITION OF transactions
    FOR VALUES WITH (MODULUS 4, REMAINDER 2);

CREATE TABLE transactions_p3 PARTITION OF transactions
    FOR VALUES WITH (MODULUS 4, REMAINDER 3);
```

## Многоуровневое партиционирование

PostgreSQL поддерживает вложенные партиции.

```sql
-- Первый уровень - по году
CREATE TABLE events (
    id BIGSERIAL,
    event_type VARCHAR(50) NOT NULL,
    event_date DATE NOT NULL,
    payload JSONB
) PARTITION BY RANGE (event_date);

-- Второй уровень - по типу события
CREATE TABLE events_2024 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2025-01-01')
    PARTITION BY LIST (event_type);

-- Конечные партиции
CREATE TABLE events_2024_clicks PARTITION OF events_2024
    FOR VALUES IN ('click', 'double_click');

CREATE TABLE events_2024_views PARTITION OF events_2024
    FOR VALUES IN ('page_view', 'impression');

CREATE TABLE events_2024_other PARTITION OF events_2024 DEFAULT;
```

## Индексы в партиционированных таблицах

### Глобальные индексы

```sql
-- Индекс на родительской таблице автоматически создается на всех партициях
CREATE INDEX idx_orders_customer ON orders(customer_id);

-- Уникальный индекс ДОЛЖЕН включать ключ партиционирования
CREATE UNIQUE INDEX idx_orders_id_date ON orders(id, order_date);
```

### Локальные индексы

```sql
-- Создание индекса только на конкретной партиции
CREATE INDEX idx_orders_2024_01_status ON orders_2024_01(status);
```

## Ограничения (Constraints)

```sql
-- PRIMARY KEY должен включать ключ партиционирования
CREATE TABLE orders (
    id BIGSERIAL,
    order_date DATE NOT NULL,
    customer_id INTEGER,
    PRIMARY KEY (id, order_date)  -- order_date обязателен
) PARTITION BY RANGE (order_date);

-- FOREIGN KEY на партиционированные таблицы (PostgreSQL 12+)
ALTER TABLE order_items
ADD CONSTRAINT fk_order
FOREIGN KEY (order_id, order_date) REFERENCES orders(id, order_date);
```

## Управление партициями

### Добавление новой партиции

```sql
-- Добавление партиции на будущий месяц
CREATE TABLE orders_2024_04 PARTITION OF orders
    FOR VALUES FROM ('2024-04-01') TO ('2024-05-01');
```

### Удаление партиции

```sql
-- Отсоединение партиции (данные сохраняются)
ALTER TABLE orders DETACH PARTITION orders_2024_01;

-- Полное удаление партиции с данными
DROP TABLE orders_2024_01;

-- DETACH CONCURRENTLY (PostgreSQL 14+) - без блокировки
ALTER TABLE orders DETACH PARTITION orders_2024_01 CONCURRENTLY;
```

### Присоединение существующей таблицы

```sql
-- Создание таблицы с такой же структурой
CREATE TABLE orders_2024_05 (LIKE orders INCLUDING ALL);

-- Добавление constraint для проверки данных
ALTER TABLE orders_2024_05 ADD CONSTRAINT check_date
    CHECK (order_date >= '2024-05-01' AND order_date < '2024-06-01');

-- Присоединение как партиция
ALTER TABLE orders ATTACH PARTITION orders_2024_05
    FOR VALUES FROM ('2024-05-01') TO ('2024-06-01');
```

## Автоматическое создание партиций

### Использование pg_partman

```sql
-- Установка расширения
CREATE EXTENSION pg_partman;

-- Создание партиционированной таблицы
CREATE TABLE events (
    id BIGSERIAL,
    event_date TIMESTAMP NOT NULL,
    data JSONB
) PARTITION BY RANGE (event_date);

-- Настройка автоматического партиционирования
SELECT partman.create_parent(
    p_parent_table := 'public.events',
    p_control := 'event_date',
    p_type := 'native',
    p_interval := 'daily',
    p_premake := 7  -- создать 7 партиций заранее
);

-- Настройка автоматического обслуживания
SELECT partman.run_maintenance();
```

### Триггер для автоматического создания партиций

```sql
-- Функция для автоматического создания партиций
CREATE OR REPLACE FUNCTION create_partition_if_not_exists()
RETURNS TRIGGER AS $$
DECLARE
    partition_date DATE;
    partition_name TEXT;
    start_date DATE;
    end_date DATE;
BEGIN
    partition_date := DATE_TRUNC('month', NEW.order_date);
    partition_name := 'orders_' || TO_CHAR(partition_date, 'YYYY_MM');
    start_date := partition_date;
    end_date := partition_date + INTERVAL '1 month';

    -- Проверяем существование партиции
    IF NOT EXISTS (
        SELECT 1 FROM pg_class WHERE relname = partition_name
    ) THEN
        EXECUTE format(
            'CREATE TABLE %I PARTITION OF orders FOR VALUES FROM (%L) TO (%L)',
            partition_name, start_date, end_date
        );
    END IF;

    RETURN NEW;
END;
$$ LANGUAGE plpgsql;
```

## Partition Pruning (отсечение партиций)

PostgreSQL автоматически исключает ненужные партиции из запроса.

```sql
-- Запрос использует только одну партицию
EXPLAIN SELECT * FROM orders WHERE order_date = '2024-02-15';

-- Результат показывает сканирование только orders_2024_02
-- Seq Scan on orders_2024_02 orders
--   Filter: (order_date = '2024-02-15'::date)

-- Включение runtime partition pruning
SET enable_partition_pruning = on;  -- по умолчанию включено
```

### Проверка эффективности partition pruning

```sql
-- Анализ плана выполнения
EXPLAIN (ANALYZE, COSTS, BUFFERS)
SELECT * FROM orders
WHERE order_date BETWEEN '2024-02-01' AND '2024-02-28';
```

## Миграция существующей таблицы на партиционированную

```sql
-- 1. Переименование старой таблицы
ALTER TABLE orders RENAME TO orders_old;

-- 2. Создание партиционированной таблицы
CREATE TABLE orders (
    id BIGSERIAL,
    customer_id INTEGER,
    order_date DATE NOT NULL,
    total_amount DECIMAL(10,2)
) PARTITION BY RANGE (order_date);

-- 3. Создание партиций
CREATE TABLE orders_2024_01 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');
-- ... остальные партиции

-- 4. Копирование данных
INSERT INTO orders SELECT * FROM orders_old;

-- 5. Удаление старой таблицы
DROP TABLE orders_old;
```

### Миграция без простоя (zero-downtime)

```sql
-- 1. Создание новой партиционированной таблицы
CREATE TABLE orders_partitioned (...) PARTITION BY RANGE (order_date);

-- 2. Настройка репликации через триггер
CREATE OR REPLACE FUNCTION replicate_to_partitioned()
RETURNS TRIGGER AS $$
BEGIN
    INSERT INTO orders_partitioned VALUES (NEW.*);
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER trg_replicate
AFTER INSERT ON orders_old
FOR EACH ROW EXECUTE FUNCTION replicate_to_partitioned();

-- 3. Копирование исторических данных
INSERT INTO orders_partitioned SELECT * FROM orders_old WHERE id <= <last_id>;

-- 4. Переключение (в транзакции)
BEGIN;
ALTER TABLE orders_old RENAME TO orders_archive;
ALTER TABLE orders_partitioned RENAME TO orders;
COMMIT;
```

## Мониторинг партиций

```sql
-- Просмотр всех партиций таблицы
SELECT
    parent.relname AS parent_table,
    child.relname AS partition_name,
    pg_size_pretty(pg_relation_size(child.oid)) AS size,
    pg_stat_get_live_tuples(child.oid) AS row_count
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
WHERE parent.relname = 'orders'
ORDER BY child.relname;

-- Информация о партиционировании
SELECT
    nmsp_parent.nspname AS parent_schema,
    parent.relname AS parent_table,
    nmsp_child.nspname AS child_schema,
    child.relname AS child_table,
    pg_get_expr(child.relpartbound, child.oid) AS partition_expression
FROM pg_inherits
JOIN pg_class parent ON pg_inherits.inhparent = parent.oid
JOIN pg_class child ON pg_inherits.inhrelid = child.oid
JOIN pg_namespace nmsp_parent ON parent.relnamespace = nmsp_parent.oid
JOIN pg_namespace nmsp_child ON child.relnamespace = nmsp_child.oid
WHERE parent.relname = 'orders';
```

## Best Practices

### 1. Выбор ключа партиционирования

```sql
-- Хорошо: часто используемый в WHERE clause
CREATE TABLE logs (...) PARTITION BY RANGE (log_date);

-- Плохо: редко используемый столбец
CREATE TABLE logs (...) PARTITION BY RANGE (internal_id);
```

### 2. Оптимальный размер партиции

- Не слишком маленькие (overhead на управление)
- Не слишком большие (нет преимуществ)
- Рекомендуемый размер: 10GB - 100GB на партицию

### 3. Обслуживание партиций

```sql
-- VACUUM отдельных партиций
VACUUM ANALYZE orders_2024_01;

-- REINDEX партиции
REINDEX TABLE orders_2024_01;
```

## Типичные ошибки

1. **Неправильный выбор ключа партиционирования** - ключ должен быть в большинстве запросов
2. **Слишком много партиций** - создает overhead, рекомендуется < 1000 партиций
3. **Забыть про DEFAULT партицию** - данные могут не вставиться
4. **Неиспользование partition pruning** - проверяйте EXPLAIN
5. **Отсутствие автоматизации** - используйте pg_partman для автоматического управления

---

[prev: 01-bulk-loading](./01-bulk-loading.md) | [next: 03-sharding-patterns](./03-sharding-patterns.md)
