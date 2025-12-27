# Триггеры в PostgreSQL

## Введение

Триггер (trigger) — это специальная функция, которая автоматически выполняется в ответ на определённые события в таблице или представлении. Триггеры позволяют реализовать сложную бизнес-логику, аудит изменений, поддержание целостности данных и автоматическое обновление связанных таблиц.

## Типы триггеров

### По времени срабатывания

| Тип | Описание | Использование |
|-----|----------|---------------|
| BEFORE | Выполняется до операции | Валидация, модификация данных |
| AFTER | Выполняется после операции | Аудит, каскадные обновления |
| INSTEAD OF | Заменяет операцию | Только для VIEW |

### По уровню срабатывания

| Уровень | Описание |
|---------|----------|
| ROW | Выполняется для каждой строки |
| STATEMENT | Выполняется один раз для всего оператора |

### По типу события

- **INSERT** — при вставке
- **UPDATE** — при обновлении
- **DELETE** — при удалении
- **TRUNCATE** — при очистке таблицы (только STATEMENT уровень)

## Создание триггера

### Синтаксис

```sql
CREATE TRIGGER trigger_name
    {BEFORE | AFTER | INSTEAD OF} {INSERT | UPDATE | DELETE | TRUNCATE}
    ON table_name
    [FOR EACH {ROW | STATEMENT}]
    [WHEN (condition)]
    EXECUTE FUNCTION trigger_function();
```

### Триггерная функция

```sql
CREATE OR REPLACE FUNCTION trigger_function_name()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Логика триггера
    RETURN NEW;  -- или OLD, или NULL
END;
$$;
```

## Специальные переменные в триггерах

| Переменная | Описание |
|------------|----------|
| NEW | Новая строка (для INSERT, UPDATE) |
| OLD | Старая строка (для UPDATE, DELETE) |
| TG_NAME | Имя триггера |
| TG_TABLE_NAME | Имя таблицы |
| TG_TABLE_SCHEMA | Схема таблицы |
| TG_OP | Тип операции (INSERT, UPDATE, DELETE, TRUNCATE) |
| TG_WHEN | Время срабатывания (BEFORE, AFTER, INSTEAD OF) |
| TG_LEVEL | Уровень (ROW, STATEMENT) |
| TG_ARGV | Массив аргументов триггера |

## Практические примеры

### 1. Автоматическое обновление timestamp

```sql
-- Триггерная функция
CREATE OR REPLACE FUNCTION update_modified_timestamp()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    NEW.updated_at = NOW();
    RETURN NEW;
END;
$$;

-- Применение к таблице
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255) NOT NULL,
    content TEXT,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE TRIGGER trg_articles_updated_at
    BEFORE UPDATE ON articles
    FOR EACH ROW
    EXECUTE FUNCTION update_modified_timestamp();

-- Тестирование
INSERT INTO articles (title, content) VALUES ('Test', 'Content');
UPDATE articles SET content = 'New content' WHERE id = 1;
-- updated_at автоматически обновится
```

### 2. Аудит изменений (Audit Log)

```sql
-- Таблица аудита
CREATE TABLE audit_log (
    id SERIAL PRIMARY KEY,
    table_name VARCHAR(50) NOT NULL,
    operation VARCHAR(10) NOT NULL,
    old_data JSONB,
    new_data JSONB,
    changed_by VARCHAR(50) DEFAULT CURRENT_USER,
    changed_at TIMESTAMP DEFAULT NOW()
);

-- Универсальная функция аудита
CREATE OR REPLACE FUNCTION audit_trigger_function()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        INSERT INTO audit_log (table_name, operation, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(NEW));
        RETURN NEW;

    ELSIF TG_OP = 'UPDATE' THEN
        INSERT INTO audit_log (table_name, operation, old_data, new_data)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD), to_jsonb(NEW));
        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN
        INSERT INTO audit_log (table_name, operation, old_data)
        VALUES (TG_TABLE_NAME, TG_OP, to_jsonb(OLD));
        RETURN OLD;
    END IF;

    RETURN NULL;
END;
$$;

-- Применение к таблице users
CREATE TRIGGER trg_users_audit
    AFTER INSERT OR UPDATE OR DELETE ON users
    FOR EACH ROW
    EXECUTE FUNCTION audit_trigger_function();
```

### 3. Валидация данных

```sql
CREATE OR REPLACE FUNCTION validate_email_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Проверка формата email
    IF NEW.email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Неверный формат email: %', NEW.email;
    END IF;

    -- Приведение к нижнему регистру
    NEW.email = LOWER(NEW.email);

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_users_validate_email
    BEFORE INSERT OR UPDATE OF email ON users
    FOR EACH ROW
    EXECUTE FUNCTION validate_email_trigger();
```

### 4. Поддержание счётчиков

```sql
-- Таблицы
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    comment_count INTEGER DEFAULT 0
);

CREATE TABLE comments (
    id SERIAL PRIMARY KEY,
    post_id INTEGER REFERENCES posts(id),
    content TEXT
);

-- Триггерная функция
CREATE OR REPLACE FUNCTION update_comment_count()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    IF TG_OP = 'INSERT' THEN
        UPDATE posts
        SET comment_count = comment_count + 1
        WHERE id = NEW.post_id;
        RETURN NEW;

    ELSIF TG_OP = 'DELETE' THEN
        UPDATE posts
        SET comment_count = comment_count - 1
        WHERE id = OLD.post_id;
        RETURN OLD;

    ELSIF TG_OP = 'UPDATE' AND OLD.post_id != NEW.post_id THEN
        UPDATE posts SET comment_count = comment_count - 1 WHERE id = OLD.post_id;
        UPDATE posts SET comment_count = comment_count + 1 WHERE id = NEW.post_id;
        RETURN NEW;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_comments_count
    AFTER INSERT OR UPDATE OR DELETE ON comments
    FOR EACH ROW
    EXECUTE FUNCTION update_comment_count();
```

### 5. Soft Delete (мягкое удаление)

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    price NUMERIC,
    is_deleted BOOLEAN DEFAULT FALSE,
    deleted_at TIMESTAMP
);

CREATE OR REPLACE FUNCTION soft_delete_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- Вместо удаления помечаем запись
    UPDATE products
    SET is_deleted = TRUE, deleted_at = NOW()
    WHERE id = OLD.id;

    -- Возвращаем NULL чтобы отменить реальное удаление
    RETURN NULL;
END;
$$;

CREATE TRIGGER trg_products_soft_delete
    BEFORE DELETE ON products
    FOR EACH ROW
    EXECUTE FUNCTION soft_delete_trigger();

-- Создаём представление для активных продуктов
CREATE VIEW active_products AS
SELECT * FROM products WHERE is_deleted = FALSE;
```

### 6. Каскадное обновление

```sql
CREATE OR REPLACE FUNCTION cascade_category_update()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    -- При изменении имени категории обновляем кэшированное поле
    IF OLD.name != NEW.name THEN
        UPDATE products
        SET category_name_cache = NEW.name
        WHERE category_id = NEW.id;
    END IF;

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_category_cascade_update
    AFTER UPDATE OF name ON categories
    FOR EACH ROW
    EXECUTE FUNCTION cascade_category_update();
```

### 7. Условный триггер с WHEN

```sql
CREATE OR REPLACE FUNCTION log_price_change()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO price_history (product_id, old_price, new_price)
    VALUES (NEW.id, OLD.price, NEW.price);

    RETURN NEW;
END;
$$;

-- Триггер срабатывает только при изменении цены
CREATE TRIGGER trg_log_price_change
    AFTER UPDATE ON products
    FOR EACH ROW
    WHEN (OLD.price IS DISTINCT FROM NEW.price)
    EXECUTE FUNCTION log_price_change();
```

### 8. INSTEAD OF триггер для View

```sql
CREATE VIEW user_full_info AS
SELECT
    u.id,
    u.username,
    u.email,
    p.first_name,
    p.last_name,
    p.phone
FROM users u
LEFT JOIN profiles p ON p.user_id = u.id;

CREATE OR REPLACE FUNCTION user_full_info_insert()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    new_user_id INTEGER;
BEGIN
    -- Вставляем пользователя
    INSERT INTO users (username, email)
    VALUES (NEW.username, NEW.email)
    RETURNING id INTO new_user_id;

    -- Вставляем профиль
    INSERT INTO profiles (user_id, first_name, last_name, phone)
    VALUES (new_user_id, NEW.first_name, NEW.last_name, NEW.phone);

    RETURN NEW;
END;
$$;

CREATE TRIGGER trg_user_full_info_insert
    INSTEAD OF INSERT ON user_full_info
    FOR EACH ROW
    EXECUTE FUNCTION user_full_info_insert();

-- Теперь можно вставлять через View
INSERT INTO user_full_info (username, email, first_name, last_name, phone)
VALUES ('john', 'john@example.com', 'John', 'Doe', '+7999123456');
```

### 9. STATEMENT level триггер

```sql
CREATE TABLE import_log (
    id SERIAL PRIMARY KEY,
    operation VARCHAR(20),
    row_count INTEGER,
    performed_at TIMESTAMP DEFAULT NOW()
);

CREATE OR REPLACE FUNCTION log_bulk_operation()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO import_log (operation, row_count)
    VALUES (TG_OP, (SELECT count(*) FROM inserted_rows));

    RETURN NULL;
END;
$$;

-- Триггер выполняется один раз для всей операции
CREATE TRIGGER trg_bulk_insert_log
    AFTER INSERT ON data_imports
    REFERENCING NEW TABLE AS inserted_rows
    FOR EACH STATEMENT
    EXECUTE FUNCTION log_bulk_operation();
```

### 10. Триггер с аргументами

```sql
CREATE OR REPLACE FUNCTION generic_audit()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    audit_table TEXT;
BEGIN
    -- Получаем имя таблицы аудита из аргументов
    audit_table := TG_ARGV[0];

    EXECUTE format(
        'INSERT INTO %I (original_id, operation, data, changed_at)
         VALUES ($1, $2, $3, NOW())',
        audit_table
    ) USING
        CASE WHEN TG_OP = 'DELETE' THEN OLD.id ELSE NEW.id END,
        TG_OP,
        CASE WHEN TG_OP = 'DELETE' THEN to_jsonb(OLD) ELSE to_jsonb(NEW) END;

    IF TG_OP = 'DELETE' THEN
        RETURN OLD;
    ELSE
        RETURN NEW;
    END IF;
END;
$$;

-- Передаём имя таблицы аудита как аргумент
CREATE TRIGGER trg_orders_audit
    AFTER INSERT OR UPDATE OR DELETE ON orders
    FOR EACH ROW
    EXECUTE FUNCTION generic_audit('orders_audit');
```

## Управление триггерами

### Просмотр триггеров

```sql
-- Все триггеры в базе
SELECT
    trigger_name,
    event_manipulation,
    event_object_table,
    action_timing,
    action_orientation
FROM information_schema.triggers
WHERE trigger_schema = 'public';

-- Детальная информация
SELECT
    tgname AS trigger_name,
    tgrelid::regclass AS table_name,
    tgtype,
    tgenabled,
    tgfoid::regproc AS function_name
FROM pg_trigger
WHERE NOT tgisinternal;
```

### Включение/отключение триггеров

```sql
-- Отключить конкретный триггер
ALTER TABLE users DISABLE TRIGGER trg_users_audit;

-- Включить триггер
ALTER TABLE users ENABLE TRIGGER trg_users_audit;

-- Отключить все триггеры таблицы
ALTER TABLE users DISABLE TRIGGER ALL;

-- Включить все триггеры
ALTER TABLE users ENABLE TRIGGER ALL;

-- Отключить только пользовательские триггеры (не системные)
ALTER TABLE users DISABLE TRIGGER USER;
```

### Удаление триггера

```sql
DROP TRIGGER IF EXISTS trg_users_audit ON users;

-- Удаление функции (если не используется другими триггерами)
DROP FUNCTION IF EXISTS audit_trigger_function();
```

## Порядок выполнения триггеров

Когда на таблице несколько триггеров:

1. **BEFORE STATEMENT** триггеры
2. Для каждой затронутой строки:
   - **BEFORE ROW** триггеры (в алфавитном порядке)
   - Выполнение операции
   - **AFTER ROW** триггеры (в алфавитном порядке)
3. **AFTER STATEMENT** триггеры

```sql
-- Можно использовать именование для контроля порядка
CREATE TRIGGER a_first_trigger ...;
CREATE TRIGGER b_second_trigger ...;
CREATE TRIGGER c_third_trigger ...;
```

## Event Triggers

Event-триггеры срабатывают на DDL-операции (CREATE, ALTER, DROP):

```sql
-- Логирование DDL операций
CREATE OR REPLACE FUNCTION log_ddl_operation()
RETURNS EVENT_TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    INSERT INTO ddl_log (event, command, executed_at)
    VALUES (TG_EVENT, current_query(), NOW());
END;
$$;

CREATE EVENT TRIGGER trg_log_ddl
    ON ddl_command_end
    EXECUTE FUNCTION log_ddl_operation();

-- Запрет удаления таблиц
CREATE OR REPLACE FUNCTION prevent_drop_table()
RETURNS EVENT_TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    obj RECORD;
BEGIN
    FOR obj IN SELECT * FROM pg_event_trigger_dropped_objects() LOOP
        IF obj.object_type = 'table' THEN
            RAISE EXCEPTION 'Удаление таблиц запрещено: %', obj.object_identity;
        END IF;
    END LOOP;
END;
$$;

CREATE EVENT TRIGGER trg_prevent_drop
    ON sql_drop
    EXECUTE FUNCTION prevent_drop_table();
```

## Best Practices

### 1. Краткость и эффективность

```sql
-- Плохо: сложная логика в триггере
CREATE FUNCTION complex_trigger()
RETURNS TRIGGER AS $$
BEGIN
    -- 100 строк сложной логики
END;
$$;

-- Хорошо: минимальная логика, вызов внешних функций
CREATE FUNCTION clean_trigger()
RETURNS TRIGGER AS $$
BEGIN
    PERFORM process_business_logic(NEW);
    RETURN NEW;
END;
$$;
```

### 2. Избегайте рекурсии

```sql
-- Опасно: триггер может вызвать сам себя
CREATE FUNCTION dangerous_trigger()
RETURNS TRIGGER AS $$
BEGIN
    UPDATE same_table SET field = value;  -- Вызовет триггер снова!
    RETURN NEW;
END;
$$;

-- Безопасно: используйте флаг или session variable
CREATE FUNCTION safe_trigger()
RETURNS TRIGGER AS $$
BEGIN
    IF current_setting('myapp.skip_trigger', TRUE) = 'true' THEN
        RETURN NEW;
    END IF;

    SET LOCAL myapp.skip_trigger = 'true';
    UPDATE same_table SET field = value;
    SET LOCAL myapp.skip_trigger = 'false';

    RETURN NEW;
END;
$$;
```

### 3. Обработка ошибок

```sql
CREATE FUNCTION robust_trigger()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
BEGIN
    BEGIN
        -- Основная логика
        PERFORM some_operation(NEW);
    EXCEPTION
        WHEN OTHERS THEN
            -- Логирование ошибки, но не прерывание транзакции
            INSERT INTO trigger_errors (trigger_name, error_message, data)
            VALUES (TG_NAME, SQLERRM, to_jsonb(NEW));
    END;

    RETURN NEW;
END;
$$;
```

### 4. Документирование

```sql
-- Комментарий к триггеру
COMMENT ON TRIGGER trg_users_audit ON users IS
'Аудит всех изменений в таблице users. Логирует INSERT, UPDATE, DELETE.';
```

## Типичные ошибки

### 1. Забыть RETURN

```sql
-- Ошибка: нет RETURN в BEFORE триггере
CREATE FUNCTION bad_trigger() RETURNS TRIGGER AS $$
BEGIN
    NEW.updated_at = NOW();
    -- Забыли RETURN NEW; - строка не будет сохранена!
END;
$$;
```

### 2. Использование RETURN в AFTER триггере

```sql
-- AFTER триггеры игнорируют возвращаемое значение
-- но для ясности возвращайте NEW/OLD
```

### 3. Тяжёлые операции в триггере

```sql
-- Плохо: длительные операции блокируют транзакцию
CREATE FUNCTION slow_trigger() RETURNS TRIGGER AS $$
BEGIN
    -- HTTP запрос или тяжёлый расчёт
    PERFORM pg_sleep(5);  -- Заблокирует на 5 секунд!
    RETURN NEW;
END;
$$;

-- Решение: используйте асинхронные очереди (pg_notify, pgq)
```

### 4. Изменение OLD в DELETE триггере

```sql
-- OLD в DELETE триггере read-only
-- Это не работает:
OLD.some_field = 'new value';  -- Ошибка!
```

## Производительность

```sql
-- Включите логирование для анализа
SET log_statement = 'all';

-- Проверьте время выполнения триггеров
EXPLAIN ANALYZE INSERT INTO users (email) VALUES ('test@test.com');

-- Отключайте триггеры при массовых операциях
ALTER TABLE users DISABLE TRIGGER ALL;
COPY users FROM '/path/to/data.csv';
ALTER TABLE users ENABLE TRIGGER ALL;
```

## Заключение

Триггеры — мощный инструмент для автоматизации на уровне базы данных. Они идеальны для аудита, валидации, поддержания целостности данных и каскадных обновлений. Однако их следует использовать осторожно: сложные триггеры усложняют отладку и могут негативно влиять на производительность. Всегда документируйте триггеры и следите за их взаимодействием друг с другом.
