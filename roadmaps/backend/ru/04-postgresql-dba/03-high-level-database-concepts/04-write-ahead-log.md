# Write-Ahead Log (WAL) — Журнал упреждающей записи

## Введение

**Write-Ahead Log (WAL)** — это механизм журналирования, который обеспечивает долговечность (durability) и атомарность транзакций в PostgreSQL. Основной принцип: изменения сначала записываются в журнал, и только потом применяются к файлам данных.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Принцип WAL                                  │
│                                                                 │
│   1. Транзакция изменяет данные                                │
│                  ↓                                              │
│   2. Изменения записываются в WAL buffer                       │
│                  ↓                                              │
│   3. WAL buffer сбрасывается на диск (fsync при COMMIT)        │
│                  ↓                                              │
│   4. COMMIT возвращается клиенту (данные защищены!)            │
│                  ↓                                              │
│   5. Позже: checkpoint записывает данные в основные файлы     │
└─────────────────────────────────────────────────────────────────┘
```

## Зачем нужен WAL

### Проблема без WAL

```
┌─────────────────────────────────────────────────────────────────┐
│  Без WAL: прямая запись в файлы данных                         │
│                                                                 │
│  UPDATE → Изменить страницу в памяти → Записать на диск        │
│                                  ↓                              │
│                           СБОЙ СИСТЕМЫ                          │
│                                  ↓                              │
│                     Частично записанная страница               │
│                     = ПОВРЕЖДЕНИЕ ДАННЫХ                        │
└─────────────────────────────────────────────────────────────────┘
```

### Решение с WAL

```
┌─────────────────────────────────────────────────────────────────┐
│  С WAL: последовательная запись в журнал                       │
│                                                                 │
│  UPDATE → WAL запись → fsync журнала → COMMIT                  │
│                                  ↓                              │
│                           СБОЙ СИСТЕМЫ                          │
│                                  ↓                              │
│              При старте: воспроизведение WAL                    │
│              = ДАННЫЕ ВОССТАНОВЛЕНЫ                             │
└─────────────────────────────────────────────────────────────────┘
```

## Архитектура WAL

### Структура WAL-файлов

```
$PGDATA/pg_wal/
├── 000000010000000000000001  (16 MB)
├── 000000010000000000000002  (16 MB)
├── 000000010000000000000003  (16 MB)
└── archive_status/
    ├── 000000010000000000000001.done
    └── 000000010000000000000002.ready
```

### Имена WAL-файлов

```
000000010000000000000001
│       │               │
│       │               └── Номер сегмента (24 бита)
│       └────────────────── Номер лога (32 бита)
└────────────────────────── Timeline ID (8 бит)

Пример: 00000002 0000000A 0000001F
         Timeline  Log ID   Segment
```

### WAL-записи

```sql
-- Структура WAL-записи (упрощённо)
WAL Record = {
    xl_tot_len:   размер записи
    xl_xid:       ID транзакции
    xl_prev:      LSN предыдущей записи
    xl_info:      тип операции
    xl_rmid:      resource manager (heap, btree, etc.)
    xl_crc:       контрольная сумма
    data:         данные записи
}
```

## LSN (Log Sequence Number)

**LSN** — это уникальный адрес записи в WAL, состоящий из номера сегмента и смещения.

```sql
-- Текущая позиция записи в WAL
SELECT pg_current_wal_lsn();
-- Результат: 0/16B6A30

-- Разбор LSN
SELECT pg_walfile_name('0/16B6A30');
-- Результат: 000000010000000000000001

-- Текущая позиция вставки
SELECT pg_current_wal_insert_lsn();

-- Разница между LSN в байтах
SELECT pg_wal_lsn_diff('0/16B6A30', '0/16B5000');
```

## Checkpoint — Контрольные точки

### Что такое Checkpoint

Checkpoint — это процесс синхронизации данных из памяти на диск и фиксации позиции в WAL, до которой данные гарантированно записаны.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Процесс Checkpoint                           │
│                                                                 │
│  1. Пометить начало checkpoint в WAL                           │
│  2. Записать все "грязные" страницы из shared buffers на диск  │
│  3. Записать checkpoint record в WAL                           │
│  4. Обновить pg_control с позицией checkpoint                  │
│  5. Удалить старые WAL-файлы (до позиции checkpoint)           │
└─────────────────────────────────────────────────────────────────┘
```

### Настройка Checkpoint

```sql
-- Просмотр настроек
SHOW checkpoint_timeout;      -- Максимальное время между checkpoint (5min)
SHOW checkpoint_completion_target;  -- Растягивание checkpoint (0.9)
SHOW max_wal_size;           -- Когда начинать checkpoint по объёму WAL (1GB)
SHOW min_wal_size;           -- Минимум WAL для хранения (80MB)

-- Ручной checkpoint
CHECKPOINT;

-- Статистика checkpoint
SELECT * FROM pg_stat_bgwriter;
```

### Проблема: checkpoint storms

```
┌─────────────────────────────────────────────────────────────────┐
│               Checkpoint Storm                                  │
│                                                                 │
│  Много грязных страниц → Массовая запись на диск               │
│                              ↓                                  │
│                    Перегрузка I/O                               │
│                              ↓                                  │
│                    Замедление запросов                          │
└─────────────────────────────────────────────────────────────────┘
```

```ini
# Решение: растягивание checkpoint
checkpoint_completion_target = 0.9  # Использовать 90% времени между checkpoint
```

## Конфигурация WAL

### Основные параметры

```ini
# postgresql.conf

# Уровень WAL
wal_level = replica  # minimal, replica, logical

# Размер WAL буфера
wal_buffers = 64MB

# Метод синхронизации
wal_sync_method = fdatasync

# Компрессия WAL
wal_compression = on

# Размер WAL сегмента (при initdb)
# wal_segment_size = 16MB

# Checkpoint настройки
checkpoint_timeout = 10min
max_wal_size = 2GB
min_wal_size = 512MB
checkpoint_completion_target = 0.9
```

### Уровни WAL

```sql
-- minimal: только для crash recovery
-- replica: + данные для репликации и PITR
-- logical: + информация для логической репликации

SHOW wal_level;

-- Изменение требует перезапуска
ALTER SYSTEM SET wal_level = 'logical';
-- pg_ctl restart
```

## Synchronous Commit

### Режимы синхронизации

```sql
-- Полностью синхронный (по умолчанию)
SET synchronous_commit = on;
-- Ждёт записи WAL на диск локально

-- Асинхронный
SET synchronous_commit = off;
-- Не ждёт записи WAL, риск потери данных при crash

-- Только локальный диск
SET synchronous_commit = local;

-- Для реплик
SET synchronous_commit = remote_write;  -- Ждёт записи на реплику
SET synchronous_commit = remote_apply;  -- Ждёт применения на реплике
```

### Компромисс производительность vs надёжность

```sql
-- Для некритичных логов можно отключить sync
BEGIN;
SET LOCAL synchronous_commit = off;
INSERT INTO access_logs (ip, path, timestamp) VALUES (...);
COMMIT;  -- Быстро, но может потеряться при crash

-- Для критичных данных — всегда sync
BEGIN;
SET LOCAL synchronous_commit = on;
UPDATE accounts SET balance = balance - 1000 WHERE id = 1;
COMMIT;  -- Гарантированно записано
```

## WAL Archiving

### Настройка архивирования

```ini
# postgresql.conf

# Включение архивирования
archive_mode = on

# Команда для архивирования
archive_command = 'cp %p /backup/wal_archive/%f'
# или
archive_command = 'rsync -a %p backup_server:/archive/%f'

# Таймаут для archive
archive_timeout = 300  # Архивировать каждые 5 минут даже неполные сегменты
```

### Мониторинг архивирования

```sql
-- Статус архивирования
SELECT * FROM pg_stat_archiver;

-- Ожидающие архивирования файлы
SELECT COUNT(*) FROM pg_ls_dir('pg_wal/archive_status')
WHERE pg_ls_dir LIKE '%.ready';

-- Последний заархивированный файл
SELECT archived_count, last_archived_wal, last_archived_time
FROM pg_stat_archiver;
```

## Point-in-Time Recovery (PITR)

### Создание базового backup

```bash
# Использование pg_basebackup
pg_basebackup -D /backup/base -Ft -z -P -X stream

# Структура backup
/backup/
├── base/
│   ├── base.tar.gz
│   └── pg_wal.tar.gz
└── wal_archive/
    ├── 000000010000000000000001
    ├── 000000010000000000000002
    └── ...
```

### Восстановление на момент времени

```bash
# 1. Остановить PostgreSQL
pg_ctl stop

# 2. Распаковать базовый backup
cd $PGDATA
tar xzf /backup/base.tar.gz

# 3. Создать recovery.signal
touch $PGDATA/recovery.signal

# 4. Настроить восстановление в postgresql.conf
restore_command = 'cp /backup/wal_archive/%f %p'
recovery_target_time = '2024-01-15 14:30:00'
# или recovery_target_xid = '123456'
# или recovery_target_name = 'before_migration'

# 5. Запустить PostgreSQL
pg_ctl start
```

### Именованные точки восстановления

```sql
-- Создание точки восстановления
SELECT pg_create_restore_point('before_major_update');

-- Теперь можно восстановиться до этой точки
-- recovery_target_name = 'before_major_update'
```

## Streaming Replication

### Настройка Primary

```ini
# postgresql.conf на primary
wal_level = replica
max_wal_senders = 10
wal_keep_size = 1GB

# pg_hba.conf
host replication replicator 10.0.0.0/24 scram-sha-256
```

### Настройка Replica

```ini
# postgresql.conf на replica
primary_conninfo = 'host=primary port=5432 user=replicator password=secret'
```

```bash
# Создание реплики
pg_basebackup -h primary -D /var/lib/postgresql/data -U replicator -P -R
```

### Мониторинг репликации

```sql
-- На primary: слоты репликации
SELECT * FROM pg_replication_slots;

-- Отставание реплик
SELECT client_addr,
       state,
       sent_lsn,
       write_lsn,
       flush_lsn,
       replay_lsn,
       pg_wal_lsn_diff(sent_lsn, replay_lsn) AS replication_lag
FROM pg_stat_replication;

-- На replica: статус восстановления
SELECT pg_is_in_recovery();
SELECT pg_last_wal_receive_lsn();
SELECT pg_last_wal_replay_lsn();
```

## pg_wal утилиты

### pg_waldump — просмотр содержимого WAL

```bash
# Просмотр WAL записей
pg_waldump /var/lib/postgresql/data/pg_wal/000000010000000000000001

# Фильтрация по транзакции
pg_waldump -x 12345 000000010000000000000001

# Фильтрация по типу ресурса
pg_waldump -r Heap 000000010000000000000001
```

### pg_resetwal — сброс WAL (опасно!)

```bash
# Только когда нет других вариантов восстановления!
pg_resetwal -D /var/lib/postgresql/data
```

## Best Practices

### 1. Настройка под нагрузку

```ini
# Для высоконагруженных систем
wal_buffers = 64MB              # Больше буфер
checkpoint_timeout = 15min       # Реже checkpoint
max_wal_size = 4GB              # Больше места для WAL
checkpoint_completion_target = 0.9

# Для SSD
wal_sync_method = fdatasync
full_page_writes = on           # Важно для надёжности
```

### 2. Мониторинг WAL

```sql
-- Скорость генерации WAL
SELECT pg_wal_lsn_diff(
    pg_current_wal_lsn(),
    '0/0'
) / (EXTRACT(EPOCH FROM (now() - pg_postmaster_start_time())))
AS wal_bytes_per_second;

-- Объём WAL за последний час (примерно)
SELECT pg_size_pretty(
    pg_wal_lsn_diff(pg_current_wal_lsn(), stat.redo_lsn)
) AS wal_since_checkpoint
FROM pg_control_checkpoint() stat;
```

### 3. Размещение WAL на отдельном диске

```bash
# Перемещение pg_wal на отдельный диск
pg_ctl stop
mv $PGDATA/pg_wal /fast_ssd/pg_wal
ln -s /fast_ssd/pg_wal $PGDATA/pg_wal
pg_ctl start
```

### 4. Настройка архивирования для PITR

```ini
# Надёжная конфигурация архивирования
archive_mode = on
archive_command = 'test ! -f /archive/%f && cp %p /archive/%f'
archive_timeout = 300

# Или с проверкой и сжатием
archive_command = 'gzip < %p > /archive/%f.gz && test -f /archive/%f.gz'
```

## Диагностика проблем

### Медленные COMMIT

```sql
-- Проверка wal_sync_method
SHOW wal_sync_method;

-- Время ожидания WAL записи
SELECT * FROM pg_stat_wal;  -- PostgreSQL 14+
```

### WAL накапливается

```sql
-- Проверка replication slots
SELECT slot_name,
       pg_wal_lsn_diff(pg_current_wal_lsn(), restart_lsn) as lag_bytes,
       active
FROM pg_replication_slots;

-- Удаление неиспользуемого слота
SELECT pg_drop_replication_slot('unused_slot');
```

## Заключение

WAL — это критически важный компонент PostgreSQL, обеспечивающий:

1. **Durability** — данные не теряются при сбое
2. **Atomicity** — транзакции применяются целиком или не применяются вовсе
3. **Replication** — потоковая репликация на standby серверы
4. **PITR** — восстановление на любой момент времени

Правильная настройка WAL влияет на производительность и надёжность всей системы.
