# Инструменты профилирования PostgreSQL

Профилирование позволяет детально анализировать производительность запросов и находить узкие места в работе PostgreSQL. В этом разделе рассмотрим ключевые инструменты профилирования.

## pg_stat_statements

Расширение pg_stat_statements собирает статистику выполнения всех SQL-запросов. Это основной инструмент для анализа производительности.

### Установка и настройка

```sql
-- Включение в postgresql.conf
-- shared_preload_libraries = 'pg_stat_statements'

-- Создание расширения
CREATE EXTENSION pg_stat_statements;

-- Проверка установки
SELECT * FROM pg_stat_statements LIMIT 1;
```

**Параметры конфигурации (postgresql.conf):**

```ini
# Обязательный параметр
shared_preload_libraries = 'pg_stat_statements'

# Количество отслеживаемых запросов (по умолчанию 5000)
pg_stat_statements.max = 10000

# Режим отслеживания: top, all, none
pg_stat_statements.track = all

# Отслеживать вложенные запросы (функции, триггеры)
pg_stat_statements.track_utility = on

# Сохранять статистику при перезапуске
pg_stat_statements.save = on
```

### Анализ запросов

```sql
-- Топ-10 запросов по общему времени выполнения
SELECT
    queryid,
    calls,
    round(total_exec_time::numeric, 2) as total_time_ms,
    round(mean_exec_time::numeric, 2) as mean_time_ms,
    round((100 * total_exec_time / sum(total_exec_time) OVER ())::numeric, 2) as percentage,
    query
FROM pg_stat_statements
ORDER BY total_exec_time DESC
LIMIT 10;

-- Топ-10 самых частых запросов
SELECT
    queryid,
    calls,
    round(mean_exec_time::numeric, 2) as mean_time_ms,
    rows,
    query
FROM pg_stat_statements
ORDER BY calls DESC
LIMIT 10;

-- Запросы с наибольшим средним временем выполнения
SELECT
    queryid,
    calls,
    round(mean_exec_time::numeric, 2) as mean_time_ms,
    round(stddev_exec_time::numeric, 2) as stddev_ms,
    query
FROM pg_stat_statements
WHERE calls > 100  -- Исключаем редкие запросы
ORDER BY mean_exec_time DESC
LIMIT 10;
```

### Анализ I/O операций

```sql
-- Запросы с наибольшим количеством операций чтения
SELECT
    queryid,
    calls,
    shared_blks_read + local_blks_read as total_reads,
    shared_blks_hit + local_blks_hit as cache_hits,
    round(100.0 * (shared_blks_hit + local_blks_hit) /
          NULLIF(shared_blks_hit + local_blks_hit + shared_blks_read + local_blks_read, 0), 2) as cache_hit_ratio,
    query
FROM pg_stat_statements
ORDER BY (shared_blks_read + local_blks_read) DESC
LIMIT 10;

-- Запросы с наибольшим количеством записей (dirty blocks)
SELECT
    queryid,
    calls,
    shared_blks_dirtied,
    shared_blks_written,
    query
FROM pg_stat_statements
WHERE shared_blks_dirtied > 0
ORDER BY shared_blks_dirtied DESC
LIMIT 10;
```

### Сброс статистики

```sql
-- Сброс всей статистики
SELECT pg_stat_statements_reset();

-- Сброс для конкретного пользователя/базы (PostgreSQL 14+)
SELECT pg_stat_statements_reset(userid, dbid, queryid);
```

## auto_explain

Расширение auto_explain автоматически логирует планы выполнения медленных запросов.

### Настройка

```ini
# postgresql.conf

# Загрузка модуля
shared_preload_libraries = 'auto_explain'

# Минимальное время выполнения для логирования (мс)
auto_explain.log_min_duration = 1000  # 1 секунда

# Включить ANALYZE в план
auto_explain.log_analyze = on

# Включить информацию о буферах
auto_explain.log_buffers = on

# Включить временные метки
auto_explain.log_timing = on

# Формат вывода: text, xml, json, yaml
auto_explain.log_format = text

# Логировать вложенные запросы
auto_explain.log_nested_statements = on

# Уровень логирования
auto_explain.log_level = LOG
```

### Динамическое включение для сессии

```sql
-- Загрузка модуля для текущей сессии
LOAD 'auto_explain';

-- Настройка параметров
SET auto_explain.log_min_duration = 500;
SET auto_explain.log_analyze = true;
SET auto_explain.log_buffers = true;

-- Теперь все запросы > 500мс будут логироваться с планом
```

### Пример вывода в логе

```
LOG:  duration: 1523.456 ms  plan:
Query Text: SELECT * FROM orders WHERE customer_id = 12345
Seq Scan on orders  (cost=0.00..25000.00 rows=500 width=100)
                     (actual time=0.015..1520.123 rows=487 loops=1)
  Filter: (customer_id = 12345)
  Rows Removed by Filter: 999513
  Buffers: shared hit=12345 read=5678
Planning Time: 0.123 ms
Execution Time: 1523.321 ms
```

## perf и системное профилирование

Linux perf позволяет профилировать PostgreSQL на уровне CPU.

### Базовое использование

```bash
# Профилирование всех процессов PostgreSQL
sudo perf record -g -p $(pgrep -d',' postgres) -- sleep 60

# Профилирование конкретного backend
sudo perf record -g -p <backend_pid> -- sleep 30

# Просмотр результатов
sudo perf report

# Экспорт для анализа
sudo perf script > perf_output.txt
```

### Профилирование с символами PostgreSQL

```bash
# Компиляция PostgreSQL с отладочными символами
./configure --enable-debug CFLAGS="-O2 -g"

# Или установка debug-пакета
sudo apt install postgresql-15-dbgsym
```

### Анализ CPU hotspots

```bash
# Топ функций по времени CPU
sudo perf top -p $(pgrep -d',' postmaster)

# Запись с высокой частотой сэмплирования
sudo perf record -F 99 -g -p <pid> -- sleep 30

# Аннотация кода
sudo perf annotate -s <symbol_name>
```

## Flame Graphs

Flame Graphs визуализируют стек вызовов и помогают быстро найти горячие точки.

### Генерация Flame Graph

```bash
# 1. Клонирование репозитория FlameGraph
git clone https://github.com/brendangregg/FlameGraph.git
cd FlameGraph

# 2. Запись данных perf
sudo perf record -F 99 -g -p $(pgrep -d',' postgres) -- sleep 60

# 3. Экспорт данных
sudo perf script > out.perf

# 4. Свёртка стеков
./stackcollapse-perf.pl out.perf > out.folded

# 5. Генерация SVG
./flamegraph.pl out.folded > postgres_flamegraph.svg
```

### Интерпретация Flame Graph

- **Ширина блока** - доля времени CPU
- **Высота** - глубина стека вызовов
- **Цвет** - случайный (для различения)
- **Платформа стека** - чем шире, тем больше времени тратится

**Типичные паттерны:**

```
# Широкий блок ExecSort - много времени на сортировку
# -> Проверить индексы или увеличить work_mem

# Широкий блок hash_search - проблемы с хэш-таблицами
# -> Возможно, нужно увеличить shared_buffers

# Широкий блок LWLockAcquire - конкуренция за блокировки
# -> Проблема с параллелизмом
```

## pg_stat_kcache

Расширение для мониторинга системных ресурсов на уровне запросов.

```sql
-- Установка
CREATE EXTENSION pg_stat_kcache;

-- Анализ системных вызовов по запросам
SELECT
    s.query,
    k.reads,              -- Физические чтения
    k.writes,             -- Физические записи
    k.user_time,          -- CPU время (user)
    k.system_time,        -- CPU время (system)
    k.minflts,            -- Minor page faults
    k.majflts             -- Major page faults
FROM pg_stat_statements s
JOIN pg_stat_kcache k ON s.queryid = k.queryid
ORDER BY k.reads + k.writes DESC
LIMIT 10;
```

## Практические сценарии профилирования

### Сценарий 1: Поиск медленных запросов

```sql
-- 1. Включить pg_stat_statements
-- 2. Подождать накопления статистики
-- 3. Анализ

-- Найти запросы, занимающие > 10% общего времени
WITH total AS (
    SELECT sum(total_exec_time) as total_time
    FROM pg_stat_statements
)
SELECT
    query,
    calls,
    round(total_exec_time::numeric, 2) as total_ms,
    round(100.0 * total_exec_time / t.total_time, 2) as pct
FROM pg_stat_statements, total t
WHERE total_exec_time / t.total_time > 0.1
ORDER BY total_exec_time DESC;
```

### Сценарий 2: Анализ конкретного запроса

```sql
-- Получить queryid проблемного запроса
SELECT queryid, query
FROM pg_stat_statements
WHERE query LIKE '%orders%';

-- Детальная статистика
SELECT
    calls,
    rows,
    round(total_exec_time::numeric, 2) as total_ms,
    round(mean_exec_time::numeric, 2) as mean_ms,
    round(min_exec_time::numeric, 2) as min_ms,
    round(max_exec_time::numeric, 2) as max_ms,
    round(stddev_exec_time::numeric, 2) as stddev_ms
FROM pg_stat_statements
WHERE queryid = <queryid>;
```

### Сценарий 3: Мониторинг производительности в реальном времени

```bash
#!/bin/bash
# realtime_stats.sh

while true; do
    clear
    echo "=== Top 5 queries by time (last reset) ==="
    psql -c "
        SELECT
            left(query, 60) as query,
            calls,
            round(mean_exec_time::numeric, 1) as avg_ms
        FROM pg_stat_statements
        ORDER BY total_exec_time DESC
        LIMIT 5;
    "
    sleep 5
done
```

## Рекомендации по профилированию

1. **Начинайте с pg_stat_statements** - это даёт общую картину
2. **Используйте auto_explain для детализации** - планы выполнения
3. **perf для CPU-проблем** - если запросы быстрые, но CPU загружен
4. **Flame Graphs для визуализации** - быстрый обзор горячих точек

## Полезные запросы для ежедневного мониторинга

```sql
-- Создание представления для мониторинга
CREATE VIEW v_query_stats AS
SELECT
    queryid,
    calls,
    round(total_exec_time::numeric, 2) as total_ms,
    round(mean_exec_time::numeric, 2) as mean_ms,
    round(100.0 * shared_blks_hit /
          NULLIF(shared_blks_hit + shared_blks_read, 0), 2) as cache_hit_pct,
    left(query, 100) as query_preview
FROM pg_stat_statements
WHERE calls > 10
ORDER BY total_exec_time DESC;

-- Использование
SELECT * FROM v_query_stats LIMIT 20;
```

## Заключение

Комбинация инструментов профилирования позволяет:
- **pg_stat_statements** - найти проблемные запросы
- **auto_explain** - понять почему они медленные
- **perf/Flame Graphs** - диагностировать проблемы на уровне CPU
- **pg_stat_kcache** - связать запросы с системными ресурсами

Регулярное профилирование помогает предотвратить проблемы до их появления в production.
