# Hash индексы

[prev: 01-btree](./01-btree.md) | [next: 03-gist](./03-gist.md)

---

## Введение

Hash-индекс — это специализированный тип индекса в PostgreSQL, оптимизированный **только для операций равенства** (`=`). В отличие от B-Tree, Hash-индексы не поддерживают диапазонные запросы, но теоретически могут быть быстрее для точного поиска.

---

## Структура Hash-индекса

### Как работает хэширование

Hash-индекс использует хэш-функцию для преобразования значения ключа в номер "корзины" (bucket):

```
Значение → hash_function() → Номер bucket → Указатели на строки

'john@example.com' → hash() → bucket #42 → [ctid1, ctid2, ...]
'jane@example.com' → hash() → bucket #17 → [ctid3]
```

### Характеристики

- **Время поиска**: O(1) в идеальном случае
- **Поддерживает только**: оператор `=`
- **Не поддерживает**: `<`, `>`, `BETWEEN`, `ORDER BY`, `IS NULL`
- **Размер**: обычно меньше B-Tree для текстовых ключей
- **WAL-совместимость**: полная поддержка с PostgreSQL 10

---

## Создание Hash-индекса

### Базовый синтаксис

```sql
-- Создание Hash-индекса
CREATE INDEX idx_users_email_hash ON users USING hash (email);

-- Создание с параллельной загрузкой (не блокирует таблицу)
CREATE INDEX CONCURRENTLY idx_users_email_hash ON users USING hash (email);
```

### Пример использования

```sql
-- Создаём таблицу
CREATE TABLE sessions (
    id SERIAL PRIMARY KEY,
    session_token TEXT NOT NULL,
    user_id INTEGER,
    created_at TIMESTAMPTZ DEFAULT NOW(),
    expires_at TIMESTAMPTZ
);

-- Hash-индекс для поиска сессии по токену
CREATE INDEX idx_sessions_token ON sessions USING hash (session_token);

-- Этот запрос использует Hash-индекс
SELECT * FROM sessions WHERE session_token = 'abc123xyz789';

-- Этот запрос НЕ может использовать Hash-индекс
SELECT * FROM sessions WHERE session_token LIKE 'abc%';  -- Нужен B-Tree
SELECT * FROM sessions WHERE session_token > 'abc';      -- Нужен B-Tree
```

---

## Hash vs B-Tree: сравнение

### Производительность

```sql
-- Создаём тестовую таблицу
CREATE TABLE test_hash (
    id SERIAL PRIMARY KEY,
    uuid_key TEXT NOT NULL
);

-- Вставляем 10 миллионов записей
INSERT INTO test_hash (uuid_key)
SELECT md5(random()::text)
FROM generate_series(1, 10000000);

-- B-Tree индекс
CREATE INDEX idx_test_btree ON test_hash USING btree (uuid_key);

-- Hash индекс
CREATE INDEX idx_test_hash ON test_hash USING hash (uuid_key);

-- Сравниваем размеры
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size
FROM pg_indexes
WHERE tablename = 'test_hash';

-- Примерный результат:
-- idx_test_btree: 730 MB
-- idx_test_hash:  320 MB  (меньше для длинных текстовых ключей)
```

### Сравнительная таблица

| Критерий | B-Tree | Hash |
|----------|--------|------|
| Операция `=` | Да | Да |
| Диапазоны (`<`, `>`) | Да | Нет |
| `ORDER BY` | Да | Нет |
| `LIKE 'prefix%'` | Да | Нет |
| `IS NULL` | Да | Нет |
| Размер (длинные ключи) | Больше | Меньше |
| WAL-репликация | Да | Да (с PG 10) |
| Многоколоночный | Да | Нет |

---

## Ограничения Hash-индексов

### 1. Только операция равенства

```sql
-- Работает с Hash
SELECT * FROM users WHERE email = 'test@example.com';

-- НЕ работает с Hash (нужен B-Tree)
SELECT * FROM users WHERE email > 'a';
SELECT * FROM users WHERE email BETWEEN 'a' AND 'z';
SELECT * FROM users WHERE email LIKE 'test%';
SELECT * FROM users WHERE email IS NULL;
```

### 2. Одна колонка

```sql
-- Нельзя создать многоколоночный Hash-индекс
CREATE INDEX idx_multi ON users USING hash (first_name, last_name);
-- ERROR: access method "hash" does not support multicolumn indexes
```

### 3. Нет уникальности

```sql
-- Нельзя создать уникальный Hash-индекс
CREATE UNIQUE INDEX idx_unique ON users USING hash (email);
-- ERROR: access method "hash" does not support unique indexes
```

### 4. Нет сортировки

```sql
-- Hash-индекс не помогает с ORDER BY
SELECT * FROM users ORDER BY email;  -- Нужен B-Tree
```

---

## Когда использовать Hash-индексы

### Идеальные сценарии

1. **Поиск по уникальному идентификатору (UUID, токен)**
```sql
CREATE TABLE api_keys (
    id SERIAL PRIMARY KEY,
    key TEXT NOT NULL,  -- UUID или случайная строка
    user_id INTEGER,
    created_at TIMESTAMPTZ
);

-- Hash-индекс для длинных случайных строк
CREATE INDEX idx_api_keys_key ON api_keys USING hash (key);

-- Типичный запрос
SELECT * FROM api_keys WHERE key = 'a1b2c3d4-e5f6-7890-abcd-ef1234567890';
```

2. **Session tokens**
```sql
CREATE TABLE sessions (
    session_token TEXT PRIMARY KEY,  -- Использует B-Tree по умолчанию
    user_id INTEGER,
    data JSONB
);

-- Альтернатива с Hash-индексом
CREATE TABLE sessions_v2 (
    id SERIAL PRIMARY KEY,
    session_token TEXT NOT NULL,
    user_id INTEGER,
    data JSONB
);
CREATE INDEX idx_sessions_token ON sessions_v2 USING hash (session_token);
```

3. **Кэш-таблицы**
```sql
CREATE TABLE cache (
    cache_key TEXT NOT NULL,
    value BYTEA,
    expires_at TIMESTAMPTZ
);

CREATE INDEX idx_cache_key ON cache USING hash (cache_key);

-- Поиск кэша
SELECT value FROM cache WHERE cache_key = 'user:123:profile';
```

### Когда НЕ использовать

1. **Нужны диапазонные запросы**
```sql
-- Не подходит для Hash
SELECT * FROM logs WHERE created_at > '2024-01-01';
```

2. **Нужна сортировка**
```sql
-- Не подходит для Hash
SELECT * FROM users ORDER BY username;
```

3. **Префиксный поиск**
```sql
-- Не подходит для Hash
SELECT * FROM products WHERE name LIKE 'iPhone%';
```

4. **Низкая кардинальность**
```sql
-- Плохо для Hash (и для B-Tree тоже)
CREATE INDEX idx_status ON orders USING hash (status);  -- Всего 3-5 значений
```

---

## История Hash-индексов в PostgreSQL

### До PostgreSQL 10

```sql
-- ПРЕДУПРЕЖДЕНИЕ: Hash-индексы не записывались в WAL
-- При сбое требовалась перестройка
-- Не реплицировались на standby-серверы
```

### PostgreSQL 10+

- Полная поддержка WAL
- Безопасны для репликации
- Рекомендуются для production (в подходящих случаях)

### PostgreSQL 11+

- Улучшенная производительность
- Лучшее сжатие

---

## Практический пример: сравнение производительности

```sql
-- Создаём таблицу с длинными ключами
CREATE TABLE long_keys (
    id SERIAL PRIMARY KEY,
    key_value TEXT NOT NULL
);

-- Генерируем 5 миллионов записей с UUID
INSERT INTO long_keys (key_value)
SELECT gen_random_uuid()::text
FROM generate_series(1, 5000000);

-- Создаём оба типа индексов
CREATE INDEX idx_btree ON long_keys USING btree (key_value);
CREATE INDEX idx_hash ON long_keys USING hash (key_value);

-- Анализируем таблицу
ANALYZE long_keys;

-- Проверяем размеры
SELECT
    indexname,
    pg_size_pretty(pg_relation_size(indexname::regclass)) as size
FROM pg_indexes
WHERE tablename = 'long_keys';

-- Примерный вывод:
-- idx_btree: 420 MB
-- idx_hash:  180 MB

-- Сравнение производительности
-- Сначала очищаем кэш между тестами

-- B-Tree
SET enable_hashjoin = off;
DROP INDEX idx_hash;
EXPLAIN ANALYZE SELECT * FROM long_keys WHERE key_value = 'some-uuid-here';

-- Hash
DROP INDEX idx_btree;
CREATE INDEX idx_hash ON long_keys USING hash (key_value);
EXPLAIN ANALYZE SELECT * FROM long_keys WHERE key_value = 'some-uuid-here';
```

---

## Обслуживание Hash-индексов

### Перестроение

```sql
-- Перестроить индекс (требует блокировки)
REINDEX INDEX idx_sessions_token;

-- Перестроить без блокировки (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_sessions_token;
```

### Мониторинг

```sql
-- Статистика использования
SELECT
    indexrelname,
    idx_scan,
    idx_tup_read,
    idx_tup_fetch
FROM pg_stat_user_indexes
WHERE indexrelname = 'idx_sessions_token';

-- Размер индекса
SELECT pg_size_pretty(pg_relation_size('idx_sessions_token'));
```

---

## Best Practices

### 1. Выбирайте Hash для специфичных случаев

```sql
-- Хорошо: длинный уникальный ключ, только поиск по равенству
CREATE INDEX idx_tokens ON tokens USING hash (token);

-- Плохо: лучше использовать B-Tree
CREATE INDEX idx_names ON users USING hash (name);  -- Может понадобиться LIKE
```

### 2. Тестируйте производительность

```sql
-- Всегда сравнивайте с B-Tree на ваших данных
EXPLAIN ANALYZE SELECT * FROM table WHERE key = 'value';
```

### 3. Учитывайте размер индекса

```sql
-- Hash может быть значительно меньше для длинных строк
-- Это экономит место и улучшает cache hit ratio
```

### 4. Помните об ограничениях

```sql
-- Если сомневаетесь — используйте B-Tree
-- Hash — специализированный инструмент
```

---

## Типичные ошибки

1. **Использование Hash когда нужны диапазоны**
   ```sql
   -- Ошибка: Hash не поможет
   CREATE INDEX idx_date ON logs USING hash (created_at);
   SELECT * FROM logs WHERE created_at > '2024-01-01';  -- Seq Scan!
   ```

2. **Ожидание значительного ускорения**
   - Hash редко быстрее B-Tree на современном PostgreSQL
   - Основное преимущество — меньший размер

3. **Забывают про NULL**
   ```sql
   -- Hash не индексирует NULL
   SELECT * FROM table WHERE column IS NULL;  -- Seq Scan
   ```

4. **Попытка создать уникальный Hash**
   ```sql
   -- Не работает
   CREATE UNIQUE INDEX idx ON table USING hash (column);
   ```

---

## Заключение

Hash-индексы — это **специализированный инструмент** для конкретных случаев:

- Длинные случайные ключи (UUID, токены, хэши)
- Только поиск по равенству
- Экономия дискового пространства

**В большинстве случаев B-Tree — лучший выбор** благодаря универсальности. Используйте Hash только когда:
1. Точно знаете, что нужны только запросы на равенство
2. Ключи достаточно длинные, чтобы экономия места была значительной
3. Протестировали производительность на реальных данных

---

[prev: 01-btree](./01-btree.md) | [next: 03-gist](./03-gist.md)
