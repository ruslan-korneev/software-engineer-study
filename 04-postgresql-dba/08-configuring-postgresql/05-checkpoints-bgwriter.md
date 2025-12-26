# Checkpoints & Background Writer (Контрольные точки и фоновая запись)

Checkpoints и Background Writer — два ключевых механизма PostgreSQL для записи данных из памяти на диск. Правильная настройка этих компонентов критически важна для производительности и надёжности.

## Checkpoints (Контрольные точки)

### Что такое Checkpoint

Checkpoint — это точка, в которой все изменённые страницы данных (dirty pages) записываются на диск. После успешного checkpoint PostgreSQL может удалить старые WAL-файлы.

### Когда происходит Checkpoint

1. **По времени** — каждые `checkpoint_timeout` секунд
2. **По объёму WAL** — при достижении `max_wal_size`
3. **Вручную** — команда `CHECKPOINT`
4. **При остановке** — корректная остановка сервера
5. **При создании бэкапа** — `pg_backup_start()`

### Основные параметры

```ini
# postgresql.conf

# Максимальное время между checkpoints
checkpoint_timeout = 5min    # по умолчанию
checkpoint_timeout = 15min   # рекомендуется для production

# Максимальный размер WAL между checkpoints
max_wal_size = 1GB           # по умолчанию
max_wal_size = 4GB           # для высоконагруженных систем

# Минимальный размер WAL (не сжимается ниже)
min_wal_size = 80MB          # по умолчанию
min_wal_size = 1GB           # для высоконагруженных систем

# Распределение записи во времени (0.0-1.0)
checkpoint_completion_target = 0.9  # рекомендуется
```

### checkpoint_completion_target

Определяет, какую долю времени между checkpoints использовать для записи dirty pages.

```
Время записи = checkpoint_timeout × checkpoint_completion_target
```

При `checkpoint_timeout = 15min` и `checkpoint_completion_target = 0.9`:
- Запись распределяется на 13.5 минут
- Оставшиеся 1.5 минуты — резерв

```ini
# Агрессивный checkpoint (старые настройки по умолчанию)
checkpoint_completion_target = 0.5  # более резкие пики I/O

# Плавный checkpoint (рекомендуется)
checkpoint_completion_target = 0.9  # равномерное I/O
```

### Мониторинг checkpoints

```sql
-- Статистика checkpoints
SELECT
    checkpoints_timed,      -- по таймауту
    checkpoints_req,        -- по требованию (WAL size, manual)
    checkpoint_write_time,  -- время записи (ms)
    checkpoint_sync_time,   -- время fsync (ms)
    buffers_checkpoint,     -- буферы, записанные checkpoint
    buffers_clean,          -- буферы, записанные bgwriter
    buffers_backend         -- буферы, записанные backends
FROM pg_stat_bgwriter;

-- Процент checkpoints по требованию (должен быть низким)
SELECT
    round(100.0 * checkpoints_req /
        NULLIF(checkpoints_timed + checkpoints_req, 0), 2) AS pct_checkpoints_req
FROM pg_stat_bgwriter;
```

```sql
-- Активный checkpoint (прогресс)
SELECT * FROM pg_stat_progress_checkpoint;

-- Проверка в логах
-- При log_checkpoints = on в логах появляются записи о checkpoints
```

### Настройка логирования

```ini
# postgresql.conf
log_checkpoints = on  # рекомендуется для мониторинга
```

Пример лога:
```
LOG: checkpoint complete: wrote 12345 buffers (75.3%);
     0 WAL file(s) added, 2 removed, 1 recycled;
     write=89.234 s, sync=1.567 s, total=90.891 s
```

## Background Writer (Фоновая запись)

### Что делает Background Writer

Background Writer (bgwriter) периодически записывает dirty pages на диск, чтобы:
- Снизить нагрузку на checkpoints
- Уменьшить вероятность, что backend будет ждать записи
- Равномерно распределить I/O

### Основные параметры

```ini
# postgresql.conf

# Задержка между циклами записи (мс)
bgwriter_delay = 200ms     # по умолчанию

# Максимум страниц за цикл
bgwriter_lru_maxpages = 100    # по умолчанию
bgwriter_lru_maxpages = 200    # для высоконагруженных систем

# Множитель для расчёта количества страниц
bgwriter_lru_multiplier = 2.0  # по умолчанию
```

### Формула расчёта записи

```
pages_to_write = min(
    bgwriter_lru_maxpages,
    pages_needed_recently × bgwriter_lru_multiplier
)
```

### Настройка для разных сценариев

```ini
# Для систем с высокой записью
bgwriter_delay = 100ms
bgwriter_lru_maxpages = 400
bgwriter_lru_multiplier = 4.0

# Для систем с минимальной записью
bgwriter_delay = 500ms
bgwriter_lru_maxpages = 50
bgwriter_lru_multiplier = 1.0
```

### Мониторинг bgwriter

```sql
-- Основные метрики
SELECT
    buffers_clean,              -- записано bgwriter
    maxwritten_clean,           -- сколько раз достигнут лимит
    buffers_backend,            -- записано backends (плохо!)
    buffers_backend_fsync,      -- fsync backends (очень плохо!)
    buffers_alloc               -- выделено новых буферов
FROM pg_stat_bgwriter;

-- Соотношение записи
SELECT
    round(100.0 * buffers_clean /
        NULLIF(buffers_clean + buffers_checkpoint + buffers_backend, 0), 2)
        AS pct_bgwriter,
    round(100.0 * buffers_backend /
        NULLIF(buffers_clean + buffers_checkpoint + buffers_backend, 0), 2)
        AS pct_backend
FROM pg_stat_bgwriter;
```

**Проблема**: Если `buffers_backend` высок, backends вынуждены сами записывать dirty pages, что замедляет запросы.

## WAL Writer

Записывает WAL из буфера на диск.

```ini
# postgresql.conf

# Задержка между записями WAL
wal_writer_delay = 200ms

# Размер буфера WAL
wal_buffers = -1    # -1 = автоматически (1/32 от shared_buffers)
wal_buffers = 64MB  # явное значение для больших систем
```

## Взаимодействие компонентов

```
┌─────────────────────────────────────────────────────────────┐
│                     shared_buffers                          │
│                                                             │
│  ┌─────────┐  ┌─────────┐  ┌─────────┐  ┌─────────┐        │
│  │ Clean   │  │ Dirty   │  │ Dirty   │  │ Clean   │   ...  │
│  │ Page    │  │ Page    │  │ Page    │  │ Page    │        │
│  └─────────┘  └─────────┘  └─────────┘  └─────────┘        │
└───────────────────┬──────────────┬──────────────────────────┘
                    │              │
        ┌───────────┘              └───────────┐
        ▼                                      ▼
┌───────────────┐                    ┌─────────────────┐
│  Background   │                    │   Checkpoint    │
│    Writer     │                    │    Process      │
└───────┬───────┘                    └────────┬────────┘
        │                                     │
        │    Постоянно записывает             │  Периодически
        │    dirty pages                      │  записывает ВСЕ
        ▼                                     ▼  dirty pages
┌─────────────────────────────────────────────────────────────┐
│                         Disk                                 │
└─────────────────────────────────────────────────────────────┘
```

## Оптимальная конфигурация

### Для OLTP (много мелких транзакций)

```ini
# Частые небольшие checkpoints
checkpoint_timeout = 10min
max_wal_size = 2GB
checkpoint_completion_target = 0.9

# Активный bgwriter
bgwriter_delay = 100ms
bgwriter_lru_maxpages = 200
bgwriter_lru_multiplier = 2.0
```

### Для OLAP (большие аналитические запросы)

```ini
# Редкие большие checkpoints
checkpoint_timeout = 30min
max_wal_size = 8GB
checkpoint_completion_target = 0.9

# Менее активный bgwriter
bgwriter_delay = 200ms
bgwriter_lru_maxpages = 100
bgwriter_lru_multiplier = 2.0
```

### Для высокой записи

```ini
checkpoint_timeout = 15min
max_wal_size = 16GB
min_wal_size = 4GB
checkpoint_completion_target = 0.9

bgwriter_delay = 50ms
bgwriter_lru_maxpages = 500
bgwriter_lru_multiplier = 4.0

wal_buffers = 64MB
```

## Диагностика проблем

### Слишком частые checkpoints

```sql
-- Проверка
SELECT
    checkpoints_req,
    checkpoints_timed,
    round(100.0 * checkpoints_req / (checkpoints_req + checkpoints_timed), 2) AS pct_forced
FROM pg_stat_bgwriter;
```

**Решение**: Увеличить `max_wal_size`.

### Backends пишут на диск

```sql
SELECT buffers_backend, buffers_clean
FROM pg_stat_bgwriter;
```

**Решение**: Увеличить `bgwriter_lru_maxpages` и уменьшить `bgwriter_delay`.

### Долгие checkpoints

```sql
-- Средняя длительность checkpoint
SELECT
    checkpoint_write_time / 1000.0 / checkpoints_timed AS avg_write_sec,
    checkpoint_sync_time / 1000.0 / checkpoints_timed AS avg_sync_sec
FROM pg_stat_bgwriter;
```

**Решение**: Увеличить `checkpoint_timeout`, уменьшить `checkpoint_completion_target`.

## Типичные ошибки

1. **Малый max_wal_size** — слишком частые checkpoints
2. **checkpoint_completion_target = 0.5** — пиковые нагрузки I/O
3. **Игнорирование buffers_backend** — backends тормозят на записи
4. **Выключенный log_checkpoints** — нет информации для диагностики
5. **Слишком маленький wal_buffers** — задержки при записи WAL

## Best Practices

1. **Установите log_checkpoints = on** для мониторинга
2. **Целевое значение checkpoints_req** — менее 10% от общего числа
3. **Мониторьте buffers_backend** — должен быть минимальным
4. **Настраивайте под тип хранилища** — SSD позволяет более агрессивные настройки
5. **Используйте checkpoint_completion_target = 0.9** для плавного I/O
6. **Увеличьте max_wal_size** если видите частые forced checkpoints
7. **Балансируйте между временем восстановления и производительностью**

```sql
-- Сброс статистики для fresh monitoring
SELECT pg_stat_reset_shared('bgwriter');
```
