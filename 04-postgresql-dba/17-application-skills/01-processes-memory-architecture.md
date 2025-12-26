# Архитектура процессов и памяти PostgreSQL

## Обзор архитектуры

PostgreSQL использует **мультипроцессную архитектуру** (process-per-connection), где каждое клиентское подключение обслуживается отдельным серверным процессом. Это обеспечивает высокую стабильность — сбой одного процесса не влияет на другие.

## Основные процессы PostgreSQL

### Postmaster (postgres)

**Postmaster** — главный процесс PostgreSQL, который:
- Запускается первым при старте сервера
- Слушает входящие подключения на указанном порту (по умолчанию 5432)
- Создает (fork) дочерние backend-процессы для каждого клиента
- Управляет жизненным циклом всех фоновых процессов
- Обрабатывает сигналы для graceful shutdown

```bash
# Просмотр процессов PostgreSQL
ps aux | grep postgres

# Типичный вывод:
# postgres: postmaster
# postgres: checkpointer
# postgres: background writer
# postgres: walwriter
# postgres: autovacuum launcher
# postgres: stats collector
# postgres: logical replication launcher
```

### Backend Processes (серверные процессы)

Каждый **backend process** обслуживает одно клиентское подключение:
- Выполняет SQL-запросы от имени клиента
- Имеет собственную локальную память
- Использует разделяемую память для доступа к данным
- Завершается при отключении клиента

```sql
-- Просмотр активных подключений
SELECT pid, usename, application_name, client_addr, state, query
FROM pg_stat_activity
WHERE backend_type = 'client backend';
```

### Background Worker Processes

PostgreSQL запускает несколько важных фоновых процессов:

| Процесс | Назначение |
|---------|------------|
| **Background Writer (bgwriter)** | Периодически записывает "грязные" страницы из буферного кэша на диск |
| **Checkpointer** | Выполняет контрольные точки — сброс всех грязных буферов |
| **WAL Writer** | Записывает WAL-буферы в WAL-файлы |
| **Autovacuum Launcher** | Запускает autovacuum workers для очистки таблиц |
| **Stats Collector** | Собирает статистику использования базы данных |
| **Logical Replication Launcher** | Управляет логической репликацией |
| **Archiver** | Архивирует заполненные WAL-сегменты |

```sql
-- Просмотр всех фоновых процессов
SELECT pid, backend_type, state
FROM pg_stat_activity
WHERE backend_type != 'client backend';
```

## Архитектура памяти

PostgreSQL делит память на две категории:

### Shared Memory (разделяемая память)

Доступна всем процессам PostgreSQL. Основные компоненты:

#### Shared Buffer Pool
```sql
-- Основной кэш данных
SHOW shared_buffers;  -- по умолчанию 128MB, рекомендуется 25% RAM
```

Хранит страницы данных (8KB блоки), прочитанные с диска. Все backend-процессы работают с этим кэшем.

#### WAL Buffers
```sql
SHOW wal_buffers;  -- буферы для записи WAL
```

Буфер для WAL-записей перед их сбросом на диск.

#### Commit Log (CLOG)
Хранит статусы транзакций (in progress, committed, aborted).

#### Lock Tables
Таблицы блокировок для координации доступа к данным.

```sql
-- Размер разделяемой памяти
SELECT name, setting, unit
FROM pg_settings
WHERE name IN ('shared_buffers', 'wal_buffers', 'huge_pages');
```

### Local Memory (локальная память)

Выделяется для каждого backend-процесса индивидуально:

#### Work Memory
```sql
SHOW work_mem;  -- по умолчанию 4MB
```

Используется для операций сортировки, хеш-таблиц, GROUP BY. Каждая операция может использовать до `work_mem` памяти.

#### Maintenance Work Memory
```sql
SHOW maintenance_work_mem;  -- по умолчанию 64MB
```

Для операций обслуживания: VACUUM, CREATE INDEX, ALTER TABLE.

#### Temp Buffers
```sql
SHOW temp_buffers;  -- для временных таблиц
```

### Схема памяти

```
┌─────────────────────────────────────────────────────────────┐
│                    Shared Memory                            │
│  ┌─────────────────┐  ┌──────────────┐  ┌───────────────┐  │
│  │  Shared Buffers │  │  WAL Buffers │  │  CLOG/Lock    │  │
│  │  (shared_buffers)│  │ (wal_buffers)│  │  Tables       │  │
│  └─────────────────┘  └──────────────┘  └───────────────┘  │
└─────────────────────────────────────────────────────────────┘
           ▲                    ▲                   ▲
           │                    │                   │
    ┌──────┴──────┐      ┌──────┴──────┐     ┌──────┴──────┐
    │  Backend 1  │      │  Backend 2  │     │  Backend N  │
    │ ┌─────────┐ │      │ ┌─────────┐ │     │ ┌─────────┐ │
    │ │work_mem │ │      │ │work_mem │ │     │ │work_mem │ │
    │ │temp_bufs│ │      │ │temp_bufs│ │     │ │temp_bufs│ │
    │ └─────────┘ │      │ └─────────┘ │     │ └─────────┘ │
    └─────────────┘      └─────────────┘     └─────────────┘
         Local Memory         Local Memory       Local Memory
```

## Мониторинг процессов и памяти

```sql
-- Использование памяти backend-процессами
SELECT pid,
       pg_size_pretty(backend_memory_contexts.total_bytes) as total_memory
FROM pg_stat_activity
JOIN LATERAL (
    SELECT sum(total_bytes) as total_bytes
    FROM pg_backend_memory_contexts
) AS backend_memory_contexts ON true
WHERE pid = pg_backend_pid();

-- Статистика shared buffers
SELECT
    c.relname,
    count(*) as buffers,
    pg_size_pretty(count(*) * 8192) as buffer_size
FROM pg_buffercache b
JOIN pg_class c ON b.relfilenode = pg_relation_filenode(c.oid)
WHERE b.reldatabase = (SELECT oid FROM pg_database WHERE datname = current_database())
GROUP BY c.relname
ORDER BY 2 DESC
LIMIT 10;
```

## Best Practices

1. **shared_buffers**: установите 25% от RAM (но не более 8GB для большинства случаев)
2. **work_mem**: осторожно увеличивайте — умножается на количество операций
3. **maintenance_work_mem**: можно увеличить для ускорения VACUUM и CREATE INDEX
4. **effective_cache_size**: установите 50-75% от RAM (используется планировщиком)

```sql
-- Рекомендуемые настройки для сервера с 32GB RAM
ALTER SYSTEM SET shared_buffers = '8GB';
ALTER SYSTEM SET work_mem = '64MB';
ALTER SYSTEM SET maintenance_work_mem = '2GB';
ALTER SYSTEM SET effective_cache_size = '24GB';
```
