# Vacuums (Очистка)

[prev: 01-resource-usage](./01-resource-usage.md) | [next: 03-replication-config](./03-replication-config.md)

---

VACUUM — процесс очистки и обслуживания таблиц в PostgreSQL. Из-за MVCC (Multi-Version Concurrency Control) старые версии строк накапливаются и требуют периодической очистки.

## Зачем нужен VACUUM

### Проблема мёртвых кортежей

При UPDATE и DELETE PostgreSQL не удаляет строки физически, а помечает их как "мёртвые":

```sql
-- Создание тестовой таблицы
CREATE TABLE test_vacuum (id serial PRIMARY KEY, data text);
INSERT INTO test_vacuum (data) SELECT md5(random()::text) FROM generate_series(1, 100000);

-- Обновление создаёт мёртвые кортежи
UPDATE test_vacuum SET data = 'updated' WHERE id < 50000;

-- Просмотр статистики
SELECT
    relname,
    n_live_tup,
    n_dead_tup,
    last_vacuum,
    last_autovacuum
FROM pg_stat_user_tables
WHERE relname = 'test_vacuum';
```

### Что делает VACUUM

1. **Освобождает место** от мёртвых кортежей для повторного использования
2. **Обновляет visibility map** для Index-Only Scans
3. **Обновляет Free Space Map** для новых вставок
4. **Предотвращает transaction ID wraparound**

## Типы VACUUM

### Обычный VACUUM

Освобождает место для повторного использования, но не возвращает его ОС.

```sql
-- VACUUM конкретной таблицы
VACUUM test_vacuum;

-- VACUUM с подробным выводом
VACUUM VERBOSE test_vacuum;

-- VACUUM всей базы данных
VACUUM;
```

### VACUUM FULL

Полностью переписывает таблицу, освобождая место в ОС. **Блокирует таблицу!**

```sql
-- Полная очистка с блокировкой
VACUUM FULL test_vacuum;

-- Проверка размера до и после
SELECT pg_size_pretty(pg_relation_size('test_vacuum'));
```

**Когда использовать**: Только когда таблица сильно "раздута" и нужно вернуть место ОС.

### VACUUM ANALYZE

Совмещает очистку с обновлением статистики.

```sql
VACUUM ANALYZE test_vacuum;
```

## Autovacuum

Автоматическая очистка, работающая в фоновом режиме.

### Основные параметры autovacuum

```ini
# postgresql.conf

# Включение autovacuum (по умолчанию on)
autovacuum = on

# Время между запусками демона (1 минута)
autovacuum_naptime = 1min

# Максимальное количество параллельных воркеров
autovacuum_max_workers = 3

# Пороги для запуска vacuum
autovacuum_vacuum_threshold = 50           # минимум мёртвых кортежей
autovacuum_vacuum_scale_factor = 0.2       # процент от таблицы

# Пороги для запуска analyze
autovacuum_analyze_threshold = 50
autovacuum_analyze_scale_factor = 0.1
```

### Формула запуска autovacuum

```
vacuum_threshold = autovacuum_vacuum_threshold + autovacuum_vacuum_scale_factor * число_строк
```

Для таблицы с 1,000,000 строк:
```
threshold = 50 + 0.2 * 1000000 = 200050 мёртвых кортежей
```

### Настройка для отдельных таблиц

```sql
-- Агрессивный autovacuum для горячей таблицы
ALTER TABLE hot_table SET (
    autovacuum_vacuum_scale_factor = 0.01,
    autovacuum_vacuum_threshold = 100,
    autovacuum_analyze_scale_factor = 0.01,
    autovacuum_analyze_threshold = 100
);

-- Просмотр настроек таблицы
SELECT relname, reloptions
FROM pg_class
WHERE relname = 'hot_table';
```

## Throttling (Ограничение нагрузки)

Контроль влияния vacuum на производительность системы.

```ini
# Стоимость операций
vacuum_cost_page_hit = 1        # чтение из shared_buffers
vacuum_cost_page_miss = 10      # чтение с диска
vacuum_cost_page_dirty = 20     # запись грязной страницы

# Лимит и задержка
vacuum_cost_limit = 200         # накопленная стоимость до паузы
vacuum_cost_delay = 0           # задержка в мс (0 = выключено)

# Для autovacuum (отдельные настройки)
autovacuum_vacuum_cost_limit = -1   # -1 = использовать vacuum_cost_limit
autovacuum_vacuum_cost_delay = 2ms  # задержка для autovacuum
```

### Агрессивный autovacuum

```ini
# Для высоконагруженных систем
autovacuum_vacuum_cost_delay = 0
autovacuum_vacuum_cost_limit = 1000
autovacuum_max_workers = 6
```

## Transaction ID Wraparound

PostgreSQL использует 32-битные transaction ID, которые могут переполниться.

### Мониторинг возраста

```sql
-- Возраст самой старой транзакции в таблицах
SELECT
    relname,
    age(relfrozenxid) as xid_age,
    pg_size_pretty(pg_relation_size(oid)) as size
FROM pg_class
WHERE relkind = 'r'
ORDER BY age(relfrozenxid) DESC
LIMIT 10;

-- Возраст по базам данных
SELECT
    datname,
    age(datfrozenxid) as age
FROM pg_database
ORDER BY age DESC;
```

### Параметры заморозки

```ini
# Когда начинать заморозку
vacuum_freeze_min_age = 50000000        # 50 млн транзакций

# Принудительный vacuum при достижении
vacuum_freeze_table_age = 150000000     # 150 млн

# Аварийный autovacuum
autovacuum_freeze_max_age = 200000000   # 200 млн
```

```sql
-- Проверка таблиц, требующих заморозки
SELECT
    schemaname || '.' || relname as table_name,
    age(relfrozenxid) as xid_age,
    CASE WHEN age(relfrozenxid) > 1000000000 THEN 'CRITICAL'
         WHEN age(relfrozenxid) > 500000000 THEN 'WARNING'
         ELSE 'OK'
    END as status
FROM pg_stat_user_tables
ORDER BY age(relfrozenxid) DESC;
```

## Мониторинг VACUUM

### Текущие процессы vacuum

```sql
-- Активные vacuum процессы
SELECT
    pid,
    query,
    state,
    wait_event_type,
    wait_event,
    backend_type
FROM pg_stat_activity
WHERE query LIKE '%vacuum%' OR backend_type = 'autovacuum worker';

-- Прогресс vacuum
SELECT
    relid::regclass as table_name,
    phase,
    heap_blks_total,
    heap_blks_scanned,
    heap_blks_vacuumed,
    index_vacuum_count,
    max_dead_tuples,
    num_dead_tuples
FROM pg_stat_progress_vacuum;
```

### Статистика autovacuum

```sql
-- Последние операции autovacuum
SELECT
    schemaname,
    relname,
    last_vacuum,
    last_autovacuum,
    vacuum_count,
    autovacuum_count,
    n_dead_tup,
    n_live_tup
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC
LIMIT 20;
```

## Bloat (Раздувание таблиц)

### Обнаружение bloat

```sql
-- Оценка bloat (упрощённая версия)
SELECT
    schemaname || '.' || relname as table_name,
    pg_size_pretty(pg_relation_size(schemaname || '.' || relname)) as size,
    n_live_tup,
    n_dead_tup,
    CASE WHEN n_live_tup > 0
         THEN round(100.0 * n_dead_tup / (n_live_tup + n_dead_tup), 2)
         ELSE 0
    END as dead_tup_percent
FROM pg_stat_user_tables
WHERE n_dead_tup > 1000
ORDER BY n_dead_tup DESC;
```

### Устранение bloat

```sql
-- Способ 1: VACUUM FULL (с блокировкой)
VACUUM FULL bloated_table;

-- Способ 2: pg_repack (без блокировки)
-- Установка расширения
CREATE EXTENSION pg_repack;

-- Использование из командной строки
-- pg_repack -d mydb -t bloated_table

-- Способ 3: CLUSTER (пересоздание по индексу)
CLUSTER bloated_table USING idx_primary;
```

## Типичные ошибки

1. **Отключение autovacuum** — приводит к bloat и wraparound
2. **Слишком высокий scale_factor** — большие таблицы очищаются редко
3. **Мало воркеров autovacuum** — не справляется с нагрузкой
4. **Игнорирование wraparound** — база может остановиться для аварийного vacuum
5. **VACUUM FULL в прайм-тайм** — блокирует таблицу

## Best Practices

1. **Никогда не отключайте autovacuum** глобально
2. **Настраивайте параметры для горячих таблиц** индивидуально
3. **Мониторьте возраст transaction ID** и dead tuples
4. **Планируйте VACUUM FULL** на периоды минимальной нагрузки
5. **Используйте pg_repack** для онлайн-дефрагментации
6. **Увеличивайте autovacuum_max_workers** при высокой нагрузке на запись
7. **Следите за bloat** и устраняйте его до критических значений

```sql
-- Пример настройки для высоконагруженной системы
ALTER SYSTEM SET autovacuum_max_workers = 6;
ALTER SYSTEM SET autovacuum_vacuum_cost_delay = '2ms';
ALTER SYSTEM SET autovacuum_vacuum_cost_limit = 1000;
ALTER SYSTEM SET autovacuum_vacuum_scale_factor = 0.05;
SELECT pg_reload_conf();
```

---

[prev: 01-resource-usage](./01-resource-usage.md) | [next: 03-replication-config](./03-replication-config.md)
