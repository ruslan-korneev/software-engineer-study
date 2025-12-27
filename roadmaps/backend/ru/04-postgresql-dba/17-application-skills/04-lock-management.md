# Управление блокировками в PostgreSQL

## Концепция блокировок

**Блокировки (locks)** обеспечивают согласованность данных при конкурентном доступе. PostgreSQL использует несколько уровней блокировок для балансировки между параллелизмом и целостностью данных.

## Типы блокировок

### Table-Level Locks (блокировки таблиц)

| Режим | Конфликтует с | Использование |
|-------|---------------|---------------|
| ACCESS SHARE | ACCESS EXCLUSIVE | SELECT |
| ROW SHARE | EXCLUSIVE, ACCESS EXCLUSIVE | SELECT FOR UPDATE/SHARE |
| ROW EXCLUSIVE | SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | UPDATE, DELETE, INSERT |
| SHARE UPDATE EXCLUSIVE | SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | VACUUM, ANALYZE, CREATE INDEX CONCURRENTLY |
| SHARE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | CREATE INDEX |
| SHARE ROW EXCLUSIVE | ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | CREATE TRIGGER |
| EXCLUSIVE | ROW SHARE, ROW EXCLUSIVE, SHARE UPDATE EXCLUSIVE, SHARE, SHARE ROW EXCLUSIVE, EXCLUSIVE, ACCESS EXCLUSIVE | Редко используется |
| ACCESS EXCLUSIVE | Все режимы | DROP, ALTER TABLE, VACUUM FULL |

```sql
-- Явная блокировка таблицы
LOCK TABLE my_table IN ACCESS EXCLUSIVE MODE;

-- Блокировка с таймаутом
SET lock_timeout = '5s';
LOCK TABLE my_table IN SHARE MODE;
```

### Row-Level Locks (блокировки строк)

```sql
-- FOR UPDATE: эксклюзивная блокировка для изменения
SELECT * FROM accounts WHERE id = 1 FOR UPDATE;

-- FOR NO KEY UPDATE: как FOR UPDATE, но не блокирует FK
SELECT * FROM accounts WHERE id = 1 FOR NO KEY UPDATE;

-- FOR SHARE: разделяемая блокировка (можно читать, нельзя менять)
SELECT * FROM accounts WHERE id = 1 FOR SHARE;

-- FOR KEY SHARE: только защита от DELETE и изменения ключа
SELECT * FROM accounts WHERE id = 1 FOR KEY SHARE;

-- SKIP LOCKED: пропустить заблокированные строки
SELECT * FROM tasks WHERE status = 'pending'
FOR UPDATE SKIP LOCKED LIMIT 1;

-- NOWAIT: не ждать, сразу ошибка если заблокировано
SELECT * FROM accounts WHERE id = 1 FOR UPDATE NOWAIT;
```

### Advisory Locks (рекомендательные блокировки)

Приложение само определяет логику — PostgreSQL только предоставляет механизм.

```sql
-- Сессионная блокировка (держится до конца сессии)
SELECT pg_advisory_lock(12345);
-- ... делаем что-то критичное ...
SELECT pg_advisory_unlock(12345);

-- Транзакционная блокировка (освобождается при COMMIT/ROLLBACK)
SELECT pg_advisory_xact_lock(12345);

-- Неблокирующие варианты
SELECT pg_try_advisory_lock(12345);  -- возвращает true/false

-- Двухключевые блокировки
SELECT pg_advisory_lock(100, 200);  -- блокировка по паре ключей
```

**Применение Advisory Locks:**
- Координация между воркерами
- Предотвращение дублирования задач
- Распределенные мьютексы

## Мониторинг блокировок

### Просмотр текущих блокировок

```sql
-- Все блокировки в системе
SELECT
    l.locktype,
    l.database,
    l.relation::regclass,
    l.page,
    l.tuple,
    l.virtualxid,
    l.transactionid,
    l.mode,
    l.granted,
    a.pid,
    a.usename,
    a.query,
    a.query_start
FROM pg_locks l
JOIN pg_stat_activity a ON l.pid = a.pid
WHERE l.relation IS NOT NULL
ORDER BY l.relation, l.mode;
```

### Обнаружение ожидающих блокировок

```sql
-- Кто кого блокирует
SELECT
    blocked.pid AS blocked_pid,
    blocked.usename AS blocked_user,
    blocking.pid AS blocking_pid,
    blocking.usename AS blocking_user,
    blocked.query AS blocked_query,
    blocking.query AS blocking_query,
    blocked.wait_event_type,
    now() - blocked.query_start AS blocked_duration
FROM pg_stat_activity blocked
JOIN pg_locks blocked_locks ON blocked.pid = blocked_locks.pid
JOIN pg_locks blocking_locks
    ON blocked_locks.locktype = blocking_locks.locktype
    AND blocked_locks.database IS NOT DISTINCT FROM blocking_locks.database
    AND blocked_locks.relation IS NOT DISTINCT FROM blocking_locks.relation
    AND blocked_locks.page IS NOT DISTINCT FROM blocking_locks.page
    AND blocked_locks.tuple IS NOT DISTINCT FROM blocking_locks.tuple
    AND blocked_locks.virtualxid IS NOT DISTINCT FROM blocking_locks.virtualxid
    AND blocked_locks.transactionid IS NOT DISTINCT FROM blocking_locks.transactionid
    AND blocked_locks.pid != blocking_locks.pid
JOIN pg_stat_activity blocking ON blocking_locks.pid = blocking.pid
WHERE NOT blocked_locks.granted;
```

### Упрощенный вид блокировок (PostgreSQL 14+)

```sql
-- Использование pg_blocking_pids
SELECT
    pid,
    usename,
    pg_blocking_pids(pid) AS blocked_by,
    query,
    wait_event_type,
    wait_event
FROM pg_stat_activity
WHERE cardinality(pg_blocking_pids(pid)) > 0;
```

## Deadlocks (взаимоблокировки)

**Deadlock** — ситуация, когда две или более транзакции ждут друг друга.

```
Транзакция A                     Транзакция B
─────────────                    ─────────────
LOCK row 1  ─────────────────►   ◄─────────────── LOCK row 2
(ждет row 2) ◄────────────────   ────────────────► (ждет row 1)
                DEADLOCK!
```

### Обнаружение и разрешение

PostgreSQL автоматически обнаруживает deadlock и отменяет одну из транзакций:

```sql
-- Настройка времени обнаружения deadlock
SHOW deadlock_timeout;  -- по умолчанию 1s

-- Уменьшение для быстрого обнаружения
ALTER SYSTEM SET deadlock_timeout = '500ms';
```

### Просмотр deadlock в логах

```
ERROR: deadlock detected
DETAIL: Process 12345 waits for ShareLock on transaction 67890;
        blocked by process 67891.
        Process 67891 waits for ShareLock on transaction 12346;
        blocked by process 12345.
HINT: See server log for query details.
```

### Предотвращение Deadlocks

```sql
-- 1. Блокируйте ресурсы в одинаковом порядке
-- Плохо:
-- TX1: UPDATE accounts SET balance = balance - 100 WHERE id = 1;
--      UPDATE accounts SET balance = balance + 100 WHERE id = 2;
-- TX2: UPDATE accounts SET balance = balance - 50 WHERE id = 2;
--      UPDATE accounts SET balance = balance + 50 WHERE id = 1;

-- Хорошо (всегда по возрастанию id):
BEGIN;
SELECT * FROM accounts WHERE id IN (1, 2) ORDER BY id FOR UPDATE;
-- Теперь безопасно обновлять
COMMIT;

-- 2. Используйте короткие транзакции
-- 3. Избегайте user interaction в транзакции
-- 4. Используйте NOWAIT или SKIP LOCKED
```

## Lock Timeouts

```sql
-- Установка таймаута ожидания блокировки
SET lock_timeout = '10s';

-- Глобально
ALTER SYSTEM SET lock_timeout = '30s';

-- На уровне транзакции
BEGIN;
SET LOCAL lock_timeout = '5s';
SELECT * FROM table FOR UPDATE;
COMMIT;
```

## Lightweight Locks (LWLocks)

Внутренние блокировки PostgreSQL для защиты структур в памяти:

```sql
-- Просмотр ожиданий LWLock
SELECT wait_event_type, wait_event, count(*)
FROM pg_stat_activity
WHERE wait_event_type = 'LWLock'
GROUP BY 1, 2
ORDER BY 3 DESC;
```

Частые LWLock contention указывает на:
- **WALWriteLock**: высокая нагрузка на WAL
- **buffer_content**: конкуренция за буферы
- **lock_manager**: много блокировок

## Практические сценарии

### Паттерн: безопасный UPDATE с проверкой

```sql
-- Атомарная операция с блокировкой
UPDATE accounts
SET balance = balance - 100
WHERE id = 1 AND balance >= 100
RETURNING *;
```

### Паттерн: очередь задач

```sql
-- Получение задачи из очереди без конфликтов
WITH task AS (
    SELECT id FROM tasks
    WHERE status = 'pending'
    ORDER BY created_at
    FOR UPDATE SKIP LOCKED
    LIMIT 1
)
UPDATE tasks
SET status = 'processing', worker_id = current_setting('app.worker_id')
FROM task
WHERE tasks.id = task.id
RETURNING tasks.*;
```

### Паттерн: координация через advisory lock

```sql
-- Синглтон-процесс (только один экземпляр)
DO $$
BEGIN
    IF NOT pg_try_advisory_lock(hashtext('my-singleton-job')) THEN
        RAISE NOTICE 'Another instance is running';
        RETURN;
    END IF;

    -- Выполняем работу...

    PERFORM pg_advisory_unlock(hashtext('my-singleton-job'));
END $$;
```

## Best Practices

1. **Короткие транзакции**: минимизируйте время удержания блокировок
2. **Последовательный порядок**: всегда блокируйте ресурсы в одном порядке
3. **lock_timeout**: устанавливайте разумные таймауты
4. **SKIP LOCKED**: используйте для систем очередей
5. **Мониторинг**: настройте алерты на долгие блокировки
6. **Избегайте ACCESS EXCLUSIVE**: используйте CREATE INDEX CONCURRENTLY вместо CREATE INDEX
7. **Advisory locks**: для координации на уровне приложения

```sql
-- Создание индекса без блокировки записи
CREATE INDEX CONCURRENTLY idx_users_email ON users(email);
```
