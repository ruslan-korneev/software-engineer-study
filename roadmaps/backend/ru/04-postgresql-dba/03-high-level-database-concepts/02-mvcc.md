# MVCC — Multiversion Concurrency Control

## Введение

**MVCC (Multiversion Concurrency Control)** — это механизм управления параллельным доступом к данным, при котором каждая транзакция видит согласованный снимок (snapshot) базы данных. MVCC позволяет читателям не блокировать писателей и наоборот.

```
┌─────────────────────────────────────────────────────────────────┐
│                    Традиционная блокировка                      │
│                                                                 │
│  Читатель ──────────┐                                          │
│                     │ БЛОКИРОВКА                               │
│  Писатель ──────────┘                                          │
│                                                                 │
├─────────────────────────────────────────────────────────────────┤
│                           MVCC                                  │
│                                                                 │
│  Читатель ─────────────────────────────→ (своя версия данных)  │
│  Писатель ─────────────────────────────→ (создаёт новую версию)│
│                                                                 │
│              Работают параллельно!                              │
└─────────────────────────────────────────────────────────────────┘
```

## Как работает MVCC в PostgreSQL

### Системные столбцы версионирования

Каждая строка в PostgreSQL содержит скрытые системные столбцы:

```sql
-- Просмотр системных столбцов
SELECT xmin, xmax, ctid, * FROM users LIMIT 5;
```

| Столбец | Описание |
|---------|----------|
| `xmin` | ID транзакции, создавшей строку |
| `xmax` | ID транзакции, удалившей/обновившей строку (0 если актуальна) |
| `ctid` | Физическое расположение строки (page, offset) |
| `cmin/cmax` | Порядковый номер команды внутри транзакции |

### Жизненный цикл строки

```
┌──────────────────────────────────────────────────────────────────┐
│                    Создание строки (INSERT)                      │
│                                                                  │
│    xmin = 100 (ID текущей транзакции)                           │
│    xmax = 0   (строка активна)                                   │
├──────────────────────────────────────────────────────────────────┤
│                    Обновление строки (UPDATE)                    │
│                                                                  │
│    Старая версия:  xmin = 100, xmax = 150                       │
│    Новая версия:   xmin = 150, xmax = 0                         │
├──────────────────────────────────────────────────────────────────┤
│                    Удаление строки (DELETE)                      │
│                                                                  │
│    xmin = 100, xmax = 200 (помечена как удалённая)              │
│    Физически остаётся до VACUUM                                  │
└──────────────────────────────────────────────────────────────────┘
```

### Практический пример

```sql
-- Сессия 1: начинаем транзакцию
BEGIN;
SELECT txid_current();  -- Допустим, вернёт 1000

-- Сессия 2: вставляем данные (транзакция 1001)
INSERT INTO products (name, price) VALUES ('Laptop', 999.99);
COMMIT;

-- Сессия 1: не видит новые данные!
SELECT * FROM products WHERE name = 'Laptop';  -- Пусто
-- Потому что snapshot сессии 1 сделан до транзакции 1001

COMMIT;  -- Завершаем транзакцию

-- Теперь видим данные
SELECT * FROM products WHERE name = 'Laptop';  -- Есть!
```

## Snapshot (Снимок данных)

### Структура снимка

Снимок содержит информацию о том, какие транзакции считать завершёнными:

```
Snapshot = {
    xmin: минимальный активный XID (все меньшие — завершены)
    xmax: следующий XID для выдачи (все >= не начались)
    xip:  список активных транзакций между xmin и xmax
}
```

### Правила видимости строки

```sql
-- Псевдокод проверки видимости
FUNCTION is_visible(row, snapshot):
    -- Строка создана текущей транзакцией
    IF row.xmin == current_xid AND row.xmax == 0:
        RETURN true

    -- Строка создана завершённой транзакцией
    IF row.xmin < snapshot.xmin AND row.xmax == 0:
        RETURN true

    -- Строка удалена, но удаляющая транзакция не завершена
    IF row.xmax IN snapshot.active_transactions:
        RETURN true

    RETURN false
```

### Визуализация видимости

```
Транзакции:   100    101    102    103    104    105
                │      │      │      │      │      │
Snapshot T104:  ✓      ✓      ?      ✓      ─      ─
                │      │      │      │
              xmin   active  xip    xmax

✓ = видимы (завершены)
? = в списке активных (xip) — не видимы
─ = ещё не начались — не видимы
```

## Уровни изоляции и MVCC

### Read Committed

```sql
-- Каждая команда видит свежий snapshot
BEGIN;
    SELECT * FROM accounts WHERE id = 1;  -- Snapshot #1
    -- Другая транзакция меняет данные и делает COMMIT
    SELECT * FROM accounts WHERE id = 1;  -- Snapshot #2, видит изменения!
COMMIT;
```

### Repeatable Read

```sql
-- Один snapshot на всю транзакцию
BEGIN TRANSACTION ISOLATION LEVEL REPEATABLE READ;
    SELECT * FROM accounts WHERE id = 1;  -- Snapshot создан здесь
    -- Другая транзакция меняет данные и делает COMMIT
    SELECT * FROM accounts WHERE id = 1;  -- Тот же snapshot, изменений не видит
COMMIT;
```

### Serializable

```sql
-- Snapshot + проверка конфликтов при COMMIT
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    SELECT SUM(balance) FROM accounts;
    UPDATE accounts SET balance = balance + 100 WHERE id = 1;
COMMIT;  -- Может упасть с serialization_failure
```

## VACUUM — Очистка старых версий

### Зачем нужен VACUUM

MVCC создаёт новые версии строк при каждом UPDATE/DELETE. Старые версии накапливаются и занимают место — это называется **bloat**.

```
┌─────────────────────────────────────────────────────────────┐
│                    Таблица accounts                         │
│                                                             │
│  Версия 1: xmin=100, xmax=150 (мёртвая)  ←─── VACUUM       │
│  Версия 2: xmin=150, xmax=200 (мёртвая)  ←─── удалит       │
│  Версия 3: xmin=200, xmax=0   (живая)    ←─── оставит      │
└─────────────────────────────────────────────────────────────┘
```

### Типы VACUUM

```sql
-- Обычный VACUUM: освобождает место для повторного использования
VACUUM accounts;

-- VACUUM FULL: полностью перестраивает таблицу, блокирует её
VACUUM FULL accounts;

-- VACUUM ANALYZE: также обновляет статистику
VACUUM ANALYZE accounts;

-- Просмотр мёртвых строк
SELECT relname, n_dead_tup, n_live_tup,
       round(n_dead_tup::numeric / nullif(n_live_tup, 0) * 100, 2) as dead_ratio
FROM pg_stat_user_tables
ORDER BY n_dead_tup DESC;
```

### Autovacuum

```sql
-- Настройки autovacuum
SHOW autovacuum;
SHOW autovacuum_vacuum_threshold;
SHOW autovacuum_vacuum_scale_factor;

-- Формула запуска: dead_tuples > threshold + scale_factor * total_tuples

-- Настройка для конкретной таблицы
ALTER TABLE high_churn_table SET (
    autovacuum_vacuum_threshold = 50,
    autovacuum_vacuum_scale_factor = 0.1
);
```

## Transaction ID Wraparound

### Проблема

PostgreSQL использует 32-битные transaction ID. После ~2 миллиардов транзакций происходит переполнение, и старые данные могут стать "невидимыми".

```
┌─────────────────────────────────────────────────────────────┐
│           Transaction ID Wraparound Problem                  │
│                                                             │
│  XID: 0 ─────────────────────────────────────────→ 2^31    │
│       │                    │                      │         │
│    Прошлое              Текущая                Будущее      │
│                        позиция                              │
│                                                             │
│  При переполнении "прошлое" становится "будущим"!          │
└─────────────────────────────────────────────────────────────┘
```

### Защита: VACUUM FREEZE

```sql
-- Заморозка старых XID
VACUUM FREEZE accounts;

-- Мониторинг возраста транзакций
SELECT datname, age(datfrozenxid) as xid_age,
       current_setting('autovacuum_freeze_max_age')::bigint as max_age
FROM pg_database
ORDER BY xid_age DESC;

-- Предупреждение появляется при приближении к лимиту
-- PostgreSQL может перейти в read-only режим для предотвращения проблемы
```

## HOT Updates (Heap-Only Tuples)

### Оптимизация обновлений

Если обновляемые столбцы не входят в индексы, PostgreSQL может использовать HOT:

```sql
-- Таблица с индексом на email
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255),
    last_login TIMESTAMP
);
CREATE INDEX idx_email ON users(email);

-- HOT update: меняется только last_login (не в индексе)
UPDATE users SET last_login = NOW() WHERE id = 1;
-- Новая версия строки на той же странице, индекс не обновляется

-- Обычный update: меняется email (в индексе)
UPDATE users SET email = 'new@example.com' WHERE id = 1;
-- Создаётся новая версия, индекс обновляется
```

### Мониторинг HOT

```sql
-- Статистика HOT обновлений
SELECT relname,
       n_tup_upd as updates,
       n_tup_hot_upd as hot_updates,
       round(n_tup_hot_upd::numeric / nullif(n_tup_upd, 0) * 100, 2) as hot_ratio
FROM pg_stat_user_tables
WHERE n_tup_upd > 0
ORDER BY n_tup_upd DESC;
```

## Best Practices

### 1. Настройка FILLFACTOR для частых UPDATE

```sql
-- Оставляем место для HOT updates
CREATE TABLE frequently_updated (
    id SERIAL PRIMARY KEY,
    status VARCHAR(20),
    updated_at TIMESTAMP
) WITH (fillfactor = 70);  -- 30% места для новых версий

ALTER TABLE existing_table SET (fillfactor = 80);
-- После изменения нужен VACUUM FULL для применения
```

### 2. Мониторинг bloat

```sql
-- Оценка bloat таблиц
SELECT
    schemaname || '.' || tablename as table,
    pg_size_pretty(pg_total_relation_size(schemaname || '.' || tablename)) as total_size,
    pg_size_pretty(pg_relation_size(schemaname || '.' || tablename)) as table_size
FROM pg_tables
WHERE schemaname = 'public'
ORDER BY pg_total_relation_size(schemaname || '.' || tablename) DESC
LIMIT 10;
```

### 3. Избегайте долгих транзакций

```sql
-- Поиск долгих транзакций
SELECT pid,
       age(clock_timestamp(), xact_start) as duration,
       query,
       state
FROM pg_stat_activity
WHERE xact_start IS NOT NULL
  AND state != 'idle'
ORDER BY xact_start;

-- Долгие транзакции блокируют VACUUM!
```

### 4. Правильный уровень изоляции

```sql
-- Для большинства операций Read Committed достаточно
-- Используйте Repeatable Read/Serializable только когда нужно

-- Плохо: всегда Serializable
SET default_transaction_isolation = 'serializable';

-- Хорошо: выбор по необходимости
BEGIN TRANSACTION ISOLATION LEVEL SERIALIZABLE;
    -- Критичная операция
COMMIT;
```

## Сравнение MVCC с блокировками

| Аспект | MVCC | Традиционные блокировки |
|--------|------|------------------------|
| Читатели vs Писатели | Не блокируют друг друга | Блокируют |
| Использование памяти | Больше (версии строк) | Меньше |
| Сложность реализации | Высокая | Средняя |
| VACUUM | Требуется | Не требуется |
| Deadlock при чтении | Невозможен | Возможен |
| Подходит для | OLTP с высокой конкурентностью | Простые сценарии |

## Заключение

MVCC — это ключевой механизм PostgreSQL, обеспечивающий высокую производительность при параллельном доступе. Понимание MVCC критически важно для:

- Правильного выбора уровня изоляции
- Настройки VACUUM и предотвращения bloat
- Оптимизации UPDATE-тяжёлых нагрузок
- Диагностики проблем с производительностью
