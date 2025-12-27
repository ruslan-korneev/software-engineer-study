# BRIN индексы

## Введение

BRIN (Block Range Index) — это **компактный индекс**, который хранит сводную информацию о диапазонах блоков данных. BRIN идеально подходит для очень больших таблиц с **естественным физическим упорядочиванием** данных.

---

## Принцип работы

### Как устроен BRIN

BRIN делит таблицу на диапазоны последовательных блоков (по умолчанию 128 блоков = 1 MB) и хранит мин/макс значения для каждого диапазона:

```
Таблица с логами (физически упорядочена по времени):
Block 1-128:    2024-01-01 ... 2024-01-15
Block 129-256:  2024-01-15 ... 2024-01-31
Block 257-384:  2024-02-01 ... 2024-02-15
...

BRIN индекс:
| Block Range | Min        | Max        |
|-------------|------------|------------|
| 1-128       | 2024-01-01 | 2024-01-15 |
| 129-256     | 2024-01-15 | 2024-01-31 |
| 257-384     | 2024-02-01 | 2024-02-15 |
```

### Характеристики

- **Размер**: Крайне маленький (в сотни раз меньше B-Tree)
- **Создание**: Очень быстрое
- **Поиск**: Исключает ненужные блоки, но сканирует блоки полностью
- **Идеален для**: Огромных таблиц с коррелированными данными

---

## Создание BRIN индекса

### Базовый синтаксис

```sql
-- Создание BRIN индекса
CREATE INDEX idx_logs_created ON logs USING brin (created_at);

-- С указанием размера диапазона (pages_per_range)
CREATE INDEX idx_logs_created ON logs USING brin (created_at)
WITH (pages_per_range = 32);  -- 256 KB вместо 1 MB

-- Параллельное создание
CREATE INDEX CONCURRENTLY idx_logs_created ON logs USING brin (created_at);
```

### Параметр pages_per_range

```sql
-- Меньше pages_per_range = точнее индекс, но больше размер
CREATE INDEX idx_small_range ON logs USING brin (created_at)
WITH (pages_per_range = 16);  -- Более точный

-- Больше pages_per_range = меньше индекс, но менее точный
CREATE INDEX idx_large_range ON logs USING brin (created_at)
WITH (pages_per_range = 256);  -- Более компактный
```

---

## Когда BRIN эффективен

### Идеальный сценарий: таблица логов

```sql
-- Таблица с естественным упорядочиванием
CREATE TABLE access_logs (
    id BIGSERIAL PRIMARY KEY,
    created_at TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    ip_address INET,
    url TEXT,
    status_code INTEGER
);

-- BRIN индекс на время создания
CREATE INDEX idx_logs_created ON access_logs USING brin (created_at);

-- Вставка данных (последовательно по времени)
INSERT INTO access_logs (ip_address, url, status_code)
SELECT
    ('192.168.' || (random()*255)::int || '.' || (random()*255)::int)::inet,
    '/page/' || (random()*1000)::int,
    CASE WHEN random() > 0.1 THEN 200 ELSE 500 END
FROM generate_series(1, 10000000);

-- Запрос использует BRIN
EXPLAIN ANALYZE
SELECT * FROM access_logs
WHERE created_at BETWEEN '2024-01-15' AND '2024-01-16';

-- Bitmap Heap Scan on access_logs
--   Recheck Cond: (created_at >= '2024-01-15' AND created_at < '2024-01-16')
--   ->  Bitmap Index Scan on idx_logs_created
```

### Сравнение размеров индексов

```sql
-- Таблица 100 миллионов строк с временной меткой

-- B-Tree индекс
CREATE INDEX idx_btree ON logs (created_at);
-- Размер: ~2.1 GB

-- BRIN индекс
CREATE INDEX idx_brin ON logs USING brin (created_at);
-- Размер: ~100 KB (в 20000 раз меньше!)
```

---

## Корреляция данных

### Что такое корреляция

Корреляция — это степень соответствия физического порядка данных и порядка значений:

```sql
-- Проверка корреляции
SELECT
    attname,
    correlation
FROM pg_stats
WHERE tablename = 'access_logs' AND attname = 'created_at';

-- correlation близка к 1.0 или -1.0 = данные хорошо упорядочены
-- correlation близка к 0 = данные случайно разбросаны
```

| Корреляция | Описание | BRIN |
|------------|----------|------|
| > 0.9 | Отличное упорядочивание | Идеально |
| 0.5-0.9 | Частичное упорядочивание | Работает |
| < 0.5 | Слабое упорядочивание | Неэффективно |
| ~ 0 | Случайный порядок | Не использовать |

### Примеры хорошей корреляции

```sql
-- Данные с естественным упорядочиванием:

-- 1. Временные ряды (логи, метрики, события)
--    Новые записи добавляются в конец
--    Корреляция: ~1.0

-- 2. Автоинкрементные ID
--    SERIAL/BIGSERIAL
--    Корреляция: 1.0

-- 3. Партиционированные таблицы
--    Каждая партиция содержит свой диапазон
--    Корреляция внутри партиции: ~1.0

-- 4. Данные из внешних источников (ETL)
--    Загружаются пакетами, упорядоченные по времени
--    Корреляция: ~1.0
```

### Примеры плохой корреляции

```sql
-- 1. UUID первичный ключ
--    Случайное распределение
--    Корреляция: ~0

-- 2. Часто обновляемые поля
--    Обновления нарушают физический порядок
--    Корреляция: падает со временем

-- 3. Таблица с массовыми UPDATE
--    Обновлённые строки перемещаются в другие блоки
--    Корреляция: нестабильная
```

---

## Практические примеры

### Пример 1: Таблица IoT-данных

```sql
-- Данные с датчиков (миллиарды строк)
CREATE TABLE sensor_data (
    id BIGSERIAL,
    sensor_id INTEGER,
    timestamp TIMESTAMPTZ NOT NULL,
    temperature NUMERIC(5,2),
    humidity NUMERIC(5,2)
) PARTITION BY RANGE (timestamp);

-- Создание партиций по месяцам
CREATE TABLE sensor_data_2024_01 PARTITION OF sensor_data
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

CREATE TABLE sensor_data_2024_02 PARTITION OF sensor_data
    FOR VALUES FROM ('2024-02-01') TO ('2024-03-01');

-- BRIN индексы на каждой партиции
CREATE INDEX idx_sensor_2024_01_ts ON sensor_data_2024_01 USING brin (timestamp);
CREATE INDEX idx_sensor_2024_02_ts ON sensor_data_2024_02 USING brin (timestamp);

-- Также BRIN на sensor_id если данные приходят группами
CREATE INDEX idx_sensor_2024_01_id ON sensor_data_2024_01 USING brin (sensor_id);
```

### Пример 2: Аудит действий

```sql
CREATE TABLE audit_log (
    id BIGSERIAL PRIMARY KEY,
    action_time TIMESTAMPTZ NOT NULL DEFAULT NOW(),
    user_id INTEGER,
    action TEXT,
    details JSONB
);

-- BRIN для временных запросов
CREATE INDEX idx_audit_time ON audit_log USING brin (action_time);

-- Типичные запросы
SELECT * FROM audit_log
WHERE action_time >= NOW() - INTERVAL '24 hours';

SELECT * FROM audit_log
WHERE action_time BETWEEN '2024-01-01' AND '2024-02-01';
```

### Пример 3: Финансовые транзакции

```sql
CREATE TABLE transactions (
    id BIGSERIAL PRIMARY KEY,
    transaction_date DATE NOT NULL,
    account_id INTEGER,
    amount NUMERIC(15,2),
    type TEXT
);

-- BRIN на дату транзакции
CREATE INDEX idx_transactions_date ON transactions USING brin (transaction_date);

-- Отчёты за период
SELECT
    transaction_date,
    SUM(amount) as total
FROM transactions
WHERE transaction_date BETWEEN '2024-01-01' AND '2024-03-31'
GROUP BY transaction_date;
```

---

## BRIN vs B-Tree

### Сравнительная таблица

| Критерий | BRIN | B-Tree |
|----------|------|--------|
| Размер индекса | Очень маленький | Большой |
| Время создания | Быстрое | Медленное |
| Точность поиска | Приблизительная | Точная |
| Требования к данным | Корреляция | Любые |
| Скорость поиска | Зависит от корреляции | Стабильно быстрая |
| Index Only Scan | Нет | Да |
| Обновление | Дешёвое | Дорогое |

### Когда что выбрать

```sql
-- Используйте BRIN:
-- 1. Огромные таблицы (100M+ строк)
-- 2. Данные с высокой корреляцией
-- 3. Диапазонные запросы (временные ряды)
-- 4. Ограниченное дисковое пространство

-- Используйте B-Tree:
-- 1. Небольшие таблицы
-- 2. Нужен Index Only Scan
-- 3. Данные без корреляции
-- 4. Точечные запросы (WHERE id = ?)
```

---

## Многоколоночные BRIN индексы

### Создание

```sql
-- Многоколоночный BRIN
CREATE INDEX idx_multi ON sensor_data USING brin (timestamp, sensor_id);

-- Эффективен когда обе колонки коррелированы
-- Например, данные приходят группами по sensor_id и по времени
```

### Ограничения

```sql
-- BRIN не поддерживает "левый префикс" как B-Tree
-- Индекс (a, b) НЕ будет использован для WHERE b = ?
-- Нужен отдельный индекс на b
```

---

## Оператор summarize

### Ручное обновление BRIN

После массовых операций рекомендуется обновить сводку:

```sql
-- Обновить статистику для всех диапазонов
SELECT brin_summarize_new_values('idx_logs_created');

-- Проверить несуммированные диапазоны
SELECT * FROM brin_page_items(get_raw_page('idx_logs_created', 2), 'idx_logs_created');
```

### Автоматическое обновление

```sql
-- PostgreSQL автоматически суммирует новые блоки
-- Но можно настроить autosummarize
CREATE INDEX idx_logs ON logs USING brin (created_at)
WITH (autosummarize = on);  -- По умолчанию off
```

---

## BRIN и партиционирование

### Комбинация стратегий

```sql
-- Партиционированная таблица
CREATE TABLE events (
    id BIGSERIAL,
    event_time TIMESTAMPTZ NOT NULL,
    event_type TEXT,
    data JSONB
) PARTITION BY RANGE (event_time);

-- Партиции по месяцам
CREATE TABLE events_2024_01 PARTITION OF events
    FOR VALUES FROM ('2024-01-01') TO ('2024-02-01');

-- BRIN на партиции (дополнительная фильтрация внутри партиции)
CREATE INDEX idx_events_2024_01 ON events_2024_01 USING brin (event_time)
WITH (pages_per_range = 64);

-- Запрос использует partition pruning + BRIN
EXPLAIN SELECT * FROM events
WHERE event_time BETWEEN '2024-01-15 10:00' AND '2024-01-15 12:00';
```

---

## Типы данных и BRIN

### Поддерживаемые типы

```sql
-- Встроенная поддержка BRIN:

-- Числовые
CREATE INDEX idx ON table USING brin (integer_col);
CREATE INDEX idx ON table USING brin (numeric_col);
CREATE INDEX idx ON table USING brin (float_col);

-- Временные
CREATE INDEX idx ON table USING brin (timestamp_col);
CREATE INDEX idx ON table USING brin (date_col);
CREATE INDEX idx ON table USING brin (interval_col);

-- Текстовые (менее эффективно)
CREATE INDEX idx ON table USING brin (text_col);

-- Сетевые
CREATE INDEX idx ON table USING brin (inet_col);
CREATE INDEX idx ON table USING brin (macaddr_col);

-- UUID
CREATE INDEX idx ON table USING brin (uuid_col);

-- Геометрические (через расширение)
CREATE INDEX idx ON table USING brin (point_col);
```

### Operator classes

```sql
-- Стандартный minmax (по умолчанию)
CREATE INDEX idx ON table USING brin (col);
-- Эквивалентно:
CREATE INDEX idx ON table USING brin (col int4_minmax_ops);

-- Bloom filter (PostgreSQL 14+) - для низкокардинальных данных
CREATE INDEX idx ON table USING brin (col int4_bloom_ops);

-- Multi minmax (PostgreSQL 14+) - для данных с выбросами
CREATE INDEX idx ON table USING brin (col int4_minmax_multi_ops);
```

---

## Мониторинг BRIN индексов

### Анализ эффективности

```sql
-- Статистика использования
SELECT
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE indexrelname LIKE '%brin%';

-- Сравнение с размером таблицы
SELECT
    relname,
    pg_size_pretty(pg_relation_size(oid)) as table_size,
    pg_size_pretty(pg_indexes_size(oid)) as indexes_size
FROM pg_class
WHERE relname = 'access_logs';
```

### Проверка корреляции

```sql
-- Корреляция колонок
SELECT
    attname,
    n_distinct,
    correlation
FROM pg_stats
WHERE tablename = 'access_logs'
ORDER BY abs(correlation) DESC;
```

---

## Best Practices

### 1. Проверяйте корреляцию перед созданием

```sql
ANALYZE table_name;
SELECT attname, correlation FROM pg_stats
WHERE tablename = 'table_name';
-- Корреляция должна быть > 0.7 для эффективности BRIN
```

### 2. Выбирайте правильный pages_per_range

```sql
-- Меньше диапазон = точнее, но больше индекс
-- Для временных рядов с запросами по часам:
CREATE INDEX idx ON logs USING brin (created_at)
WITH (pages_per_range = 32);

-- Для запросов по дням/месяцам:
CREATE INDEX idx ON logs USING brin (created_at)
WITH (pages_per_range = 128);  -- default
```

### 3. Комбинируйте с партиционированием

```sql
-- Партиции для крупного деления (месяцы)
-- BRIN для точной фильтрации внутри партиции
```

### 4. Обновляйте статистику после bulk load

```sql
-- После массовой загрузки
SELECT brin_summarize_new_values('idx_name');
ANALYZE table_name;
```

### 5. Используйте BRIN для write-heavy таблиц

```sql
-- BRIN минимально влияет на скорость INSERT
-- В отличие от B-Tree, который нужно обновлять
```

---

## Типичные ошибки

### 1. BRIN на данные без корреляции

```sql
-- Плохо: UUID случайно распределены
CREATE INDEX idx ON users USING brin (uuid);  -- Не эффективно

-- Хорошо: ID упорядочен
CREATE INDEX idx ON users USING brin (id);  -- Корреляция ~1.0
```

### 2. Ожидание точности B-Tree

```sql
-- BRIN может вернуть ложноположительные результаты
-- (блоки, которые не содержат искомых значений)
-- Это нормально — Recheck Cond фильтрует их
```

### 3. Маленькие таблицы

```sql
-- Для таблиц < 1GB B-Tree обычно лучше
-- BRIN эффективен для очень больших таблиц
```

### 4. Забывают про summarize

```sql
-- После большого INSERT новые блоки не суммированы
-- Включите autosummarize или вызывайте вручную
SELECT brin_summarize_new_values('idx_name');
```

---

## Заключение

BRIN — это **специализированный инструмент** для:

- **Очень больших таблиц** (сотни миллионов строк)
- **Данных с естественным упорядочиванием** (временные ряды, логи)
- **Диапазонных запросов** (WHERE date BETWEEN ...)
- **Экономии дискового пространства** (индекс в 1000-10000 раз меньше B-Tree)

Не используйте BRIN для:
- Точечных запросов (WHERE id = ?)
- Данных без корреляции
- Маленьких таблиц
- Когда нужен Index Only Scan
