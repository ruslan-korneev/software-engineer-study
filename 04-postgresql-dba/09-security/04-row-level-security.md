# Row Level Security (RLS) в PostgreSQL

## Введение

Row Level Security (RLS) - это механизм PostgreSQL, позволяющий контролировать доступ к отдельным строкам таблицы на основе политик безопасности. В отличие от обычных привилегий, которые работают на уровне таблицы или столбца, RLS позволяет определить, какие именно строки пользователь может видеть или изменять.

## Когда использовать RLS

- **Multi-tenant приложения**: каждый tenant видит только свои данные
- **Иерархический доступ**: менеджеры видят данные своих подчинённых
- **Конфиденциальность**: ограничение доступа к чувствительным записям
- **Аудит и compliance**: соответствие требованиям GDPR, HIPAA и др.
- **SaaS приложения**: изоляция данных клиентов

## Базовая настройка RLS

### Включение RLS для таблицы

```sql
-- Создаём таблицу
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    content TEXT,
    owner_id INTEGER,
    department VARCHAR(50),
    classification VARCHAR(20) DEFAULT 'public'
);

-- Включаем RLS
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;

-- По умолчанию после включения RLS никто (кроме владельца) не видит данные
-- Нужно создать политики
```

### Отключение RLS

```sql
-- Отключение RLS
ALTER TABLE documents DISABLE ROW LEVEL SECURITY;

-- Принудительное применение RLS даже для владельца таблицы
ALTER TABLE documents FORCE ROW LEVEL SECURITY;

-- Отмена принудительного применения
ALTER TABLE documents NO FORCE ROW LEVEL SECURITY;
```

## Создание политик (Policies)

### Синтаксис CREATE POLICY

```sql
CREATE POLICY policy_name ON table_name
    [ AS { PERMISSIVE | RESTRICTIVE } ]
    [ FOR { ALL | SELECT | INSERT | UPDATE | DELETE } ]
    [ TO { role_name | PUBLIC | CURRENT_ROLE | CURRENT_USER | SESSION_USER } [, ...] ]
    [ USING ( using_expression ) ]
    [ WITH CHECK ( check_expression ) ];
```

### Базовые примеры

```sql
-- Пользователи видят только свои документы
CREATE POLICY own_documents ON documents
    FOR SELECT
    TO PUBLIC
    USING (owner_id = current_user_id());

-- Функция для получения ID текущего пользователя
CREATE OR REPLACE FUNCTION current_user_id() RETURNS INTEGER AS $$
BEGIN
    RETURN current_setting('app.current_user_id', true)::INTEGER;
END;
$$ LANGUAGE plpgsql STABLE;
```

### Политики для разных операций

```sql
-- SELECT: какие строки можно читать
CREATE POLICY documents_select ON documents
    FOR SELECT
    USING (
        owner_id = current_user_id()
        OR classification = 'public'
    );

-- INSERT: какие строки можно добавлять
CREATE POLICY documents_insert ON documents
    FOR INSERT
    WITH CHECK (owner_id = current_user_id());

-- UPDATE: какие строки можно изменять и на что
CREATE POLICY documents_update ON documents
    FOR UPDATE
    USING (owner_id = current_user_id())
    WITH CHECK (owner_id = current_user_id());

-- DELETE: какие строки можно удалять
CREATE POLICY documents_delete ON documents
    FOR DELETE
    USING (owner_id = current_user_id());
```

### Разница между USING и WITH CHECK

- **USING**: определяет, какие существующие строки видны для операции
- **WITH CHECK**: определяет, какие новые или изменённые строки допустимы

```sql
-- Пример: пользователь может изменять свои документы,
-- но не может сделать чужой документ своим
CREATE POLICY documents_update ON documents
    FOR UPDATE
    USING (owner_id = current_user_id())      -- Можно редактировать только свои
    WITH CHECK (owner_id = current_user_id()); -- Нельзя сменить owner_id на чужой
```

## PERMISSIVE vs RESTRICTIVE политики

### PERMISSIVE (по умолчанию)

Если есть несколько PERMISSIVE политик, они объединяются через OR:

```sql
-- Пользователь видит свои документы
CREATE POLICY own_docs ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (owner_id = current_user_id());

-- ИЛИ публичные документы
CREATE POLICY public_docs ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (classification = 'public');

-- Результат: строка видна если (owner_id = current_user_id() OR classification = 'public')
```

### RESTRICTIVE

RESTRICTIVE политики объединяются через AND и применяются к результату PERMISSIVE:

```sql
-- Базовый доступ к своим документам
CREATE POLICY own_docs ON documents
    AS PERMISSIVE
    FOR SELECT
    USING (owner_id = current_user_id());

-- Дополнительное ограничение: только активные документы
CREATE POLICY active_only ON documents
    AS RESTRICTIVE
    FOR SELECT
    USING (status = 'active');

-- Результат: (owner_id = current_user_id()) AND (status = 'active')
```

## Практические сценарии

### Multi-tenant приложение

```sql
-- Таблица с данными нескольких компаний
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    tenant_id INTEGER NOT NULL,
    customer_name VARCHAR(100),
    amount NUMERIC(10, 2),
    created_at TIMESTAMP DEFAULT NOW()
);

ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

-- Функция получения tenant_id из сессии
CREATE OR REPLACE FUNCTION current_tenant_id() RETURNS INTEGER AS $$
BEGIN
    RETURN current_setting('app.tenant_id', true)::INTEGER;
END;
$$ LANGUAGE plpgsql STABLE;

-- Политика: каждый tenant видит только свои заказы
CREATE POLICY tenant_isolation ON orders
    FOR ALL
    USING (tenant_id = current_tenant_id())
    WITH CHECK (tenant_id = current_tenant_id());

-- Использование из приложения
SET app.tenant_id = '42';
SELECT * FROM orders; -- Только заказы tenant_id = 42
```

### Иерархический доступ (менеджеры видят данные подчинённых)

```sql
-- Таблица сотрудников с иерархией
CREATE TABLE employees (
    id INTEGER PRIMARY KEY,
    name VARCHAR(100),
    manager_id INTEGER REFERENCES employees(id),
    salary NUMERIC(10, 2)
);

ALTER TABLE employees ENABLE ROW LEVEL SECURITY;

-- Рекурсивная функция проверки подчинённости
CREATE OR REPLACE FUNCTION is_subordinate(employee_id INTEGER, manager INTEGER)
RETURNS BOOLEAN AS $$
WITH RECURSIVE subordinates AS (
    SELECT id, manager_id
    FROM employees
    WHERE manager_id = manager

    UNION ALL

    SELECT e.id, e.manager_id
    FROM employees e
    JOIN subordinates s ON e.manager_id = s.id
)
SELECT EXISTS (
    SELECT 1 FROM subordinates WHERE id = employee_id
);
$$ LANGUAGE sql STABLE;

-- Политика: видны свои данные и данные подчинённых
CREATE POLICY hierarchical_access ON employees
    FOR SELECT
    USING (
        id = current_user_id()
        OR is_subordinate(id, current_user_id())
    );
```

### Классификация документов

```sql
CREATE TABLE classified_docs (
    id SERIAL PRIMARY KEY,
    title VARCHAR(255),
    content TEXT,
    classification VARCHAR(20) CHECK (classification IN ('public', 'internal', 'confidential', 'secret'))
);

ALTER TABLE classified_docs ENABLE ROW LEVEL SECURITY;

-- Уровни доступа пользователей хранятся в отдельной таблице
CREATE TABLE user_clearances (
    user_name VARCHAR(100) PRIMARY KEY,
    clearance_level INTEGER -- 1=public, 2=internal, 3=confidential, 4=secret
);

-- Маппинг классификации на числовой уровень
CREATE OR REPLACE FUNCTION classification_level(cls VARCHAR)
RETURNS INTEGER AS $$
BEGIN
    RETURN CASE cls
        WHEN 'public' THEN 1
        WHEN 'internal' THEN 2
        WHEN 'confidential' THEN 3
        WHEN 'secret' THEN 4
        ELSE 0
    END;
END;
$$ LANGUAGE plpgsql IMMUTABLE;

-- Функция получения уровня допуска пользователя
CREATE OR REPLACE FUNCTION user_clearance_level()
RETURNS INTEGER AS $$
BEGIN
    RETURN COALESCE(
        (SELECT clearance_level FROM user_clearances WHERE user_name = current_user),
        1  -- По умолчанию только public
    );
END;
$$ LANGUAGE plpgsql STABLE;

-- Политика: доступ к документам согласно уровню допуска
CREATE POLICY clearance_policy ON classified_docs
    FOR SELECT
    USING (classification_level(classification) <= user_clearance_level());
```

### Временной доступ (время действия)

```sql
CREATE TABLE time_sensitive_data (
    id SERIAL PRIMARY KEY,
    data TEXT,
    valid_from TIMESTAMP DEFAULT NOW(),
    valid_until TIMESTAMP
);

ALTER TABLE time_sensitive_data ENABLE ROW LEVEL SECURITY;

-- Видны только действующие записи
CREATE POLICY time_based_access ON time_sensitive_data
    FOR SELECT
    USING (
        NOW() >= valid_from
        AND (valid_until IS NULL OR NOW() < valid_until)
    );
```

## Работа с контекстом приложения

### Использование session variables

```sql
-- Установка переменной из приложения
SET app.current_user_id = '123';
SET app.tenant_id = '456';
SET app.user_role = 'manager';

-- Чтение в политике
CREATE POLICY context_policy ON data
    FOR SELECT
    USING (
        CASE current_setting('app.user_role', true)
            WHEN 'admin' THEN true
            WHEN 'manager' THEN department = current_setting('app.department', true)
            ELSE user_id = current_setting('app.current_user_id', true)::INTEGER
        END
    );
```

### Использование отдельной таблицы сессий

```sql
-- Таблица сессий
CREATE TABLE user_sessions (
    session_id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INTEGER,
    tenant_id INTEGER,
    roles TEXT[],
    created_at TIMESTAMP DEFAULT NOW()
);

-- Функция получения текущей сессии
CREATE OR REPLACE FUNCTION get_current_session()
RETURNS user_sessions AS $$
DECLARE
    session user_sessions;
BEGIN
    SELECT * INTO session
    FROM user_sessions
    WHERE session_id = current_setting('app.session_id', true)::UUID;
    RETURN session;
END;
$$ LANGUAGE plpgsql STABLE;
```

## Bypass RLS

### Роли, обходящие RLS

```sql
-- Суперпользователи и владелец таблицы по умолчанию обходят RLS
-- Для принудительного применения RLS к владельцу:
ALTER TABLE documents FORCE ROW LEVEL SECURITY;

-- Создание роли с атрибутом BYPASSRLS
CREATE ROLE admin_user WITH BYPASSRLS;

-- Удаление атрибута
ALTER ROLE admin_user WITH NOBYPASSRLS;
```

### Временное отключение в сессии

```sql
-- Требуется роль с BYPASSRLS или суперпользователь
SET row_security = OFF;

-- Включить обратно
SET row_security = ON;
```

## Управление политиками

### Просмотр политик

```sql
-- Через psql
\d documents

-- Через системный каталог
SELECT
    schemaname,
    tablename,
    policyname,
    permissive,
    roles,
    cmd,
    qual,
    with_check
FROM pg_policies
WHERE tablename = 'documents';
```

### Изменение политик

```sql
-- Изменение условий
ALTER POLICY own_documents ON documents
    USING (owner_id = current_user_id() OR is_admin());

-- Изменение ролей
ALTER POLICY own_documents ON documents
    TO authenticated_users;

-- Переименование
ALTER POLICY own_documents ON documents
    RENAME TO user_documents;
```

### Удаление политик

```sql
DROP POLICY own_documents ON documents;

-- Удалить все политики при удалении таблицы (CASCADE)
DROP TABLE documents CASCADE;
```

## Производительность

### Влияние на запросы

```sql
-- RLS добавляет условия к запросам
-- Исходный запрос:
SELECT * FROM documents WHERE title LIKE '%report%';

-- С RLS превращается в:
SELECT * FROM documents
WHERE title LIKE '%report%'
  AND (owner_id = current_user_id() OR classification = 'public');
```

### Оптимизация

```sql
-- Создавайте индексы по полям, используемым в политиках
CREATE INDEX idx_documents_owner ON documents(owner_id);
CREATE INDEX idx_documents_tenant ON orders(tenant_id);

-- Используйте STABLE или IMMUTABLE функции
CREATE OR REPLACE FUNCTION current_tenant_id()
RETURNS INTEGER AS $$
    SELECT current_setting('app.tenant_id', true)::INTEGER;
$$ LANGUAGE sql STABLE;  -- STABLE позволяет оптимизатору кэшировать

-- Проверяйте план запроса
EXPLAIN ANALYZE SELECT * FROM documents;
```

### Избегайте дорогих операций

```sql
-- Плохо: подзапрос в каждой строке
CREATE POLICY bad_policy ON documents
    FOR SELECT
    USING (
        owner_id IN (SELECT subordinate_id FROM get_subordinates(current_user_id()))
    );

-- Лучше: использовать JOIN или материализованное представление
-- Или предвычисленную таблицу связей
```

## Best Practices

### 1. Всегда тестируйте политики

```sql
-- Создайте тестовых пользователей
CREATE ROLE test_user1;
CREATE ROLE test_user2;

-- Добавьте тестовые данные
INSERT INTO documents VALUES (1, 'Doc1', 'Content', 1, 'HR', 'public');
INSERT INTO documents VALUES (2, 'Doc2', 'Content', 2, 'IT', 'confidential');

-- Проверьте от имени каждой роли
SET ROLE test_user1;
SET app.current_user_id = '1';
SELECT * FROM documents; -- Должен видеть Doc1

SET ROLE test_user2;
SET app.current_user_id = '2';
SELECT * FROM documents; -- Должен видеть Doc2 и Doc1 (публичный)
```

### 2. Используйте роли для групповых политик

```sql
CREATE ROLE managers;
CREATE ROLE employees;

CREATE POLICY managers_policy ON employees
    FOR SELECT
    TO managers
    USING (true);  -- Менеджеры видят всех

CREATE POLICY employees_policy ON employees
    FOR SELECT
    TO employees
    USING (id = current_user_id());  -- Сотрудники только себя
```

### 3. Документируйте политики

```sql
COMMENT ON POLICY tenant_isolation ON orders IS
    'Изоляция данных по tenant_id для мультитенантной архитектуры';

COMMENT ON POLICY time_based_access ON time_sensitive_data IS
    'Скрывает записи за пределами valid_from - valid_until';
```

### 4. Централизуйте логику безопасности

```sql
-- Создайте функции для типичных проверок
CREATE OR REPLACE FUNCTION can_access_document(doc_id INTEGER)
RETURNS BOOLEAN AS $$
    -- Централизованная логика доступа
$$ LANGUAGE plpgsql STABLE SECURITY DEFINER;
```

## Типичные ошибки

### 1. Забыли включить RLS

```sql
-- Создали политику, но RLS не включён
CREATE POLICY my_policy ON documents ...;
-- Политика есть, но не работает!

-- Нужно:
ALTER TABLE documents ENABLE ROW LEVEL SECURITY;
```

### 2. Политики не покрывают все операции

```sql
-- Есть только SELECT
CREATE POLICY read_policy ON documents FOR SELECT ...;

-- Пользователь не может INSERT/UPDATE/DELETE
-- Добавьте политики для других операций или используйте FOR ALL
```

### 3. Бесконечная рекурсия

```sql
-- Политика обращается к другой таблице с RLS
CREATE POLICY check_permissions ON documents
    USING (
        EXISTS (SELECT 1 FROM user_permissions WHERE ...)  -- Тоже с RLS!
    );

-- Решение: используйте SECURITY DEFINER функции или отключите RLS для служебных таблиц
```

## Связанные темы

- [Привилегии объектов](./01-object-privileges.md) - базовые привилегии
- [GRANT и REVOKE](./02-grant-revoke.md) - управление привилегиями
- [Роли](./06-roles.md) - система ролей PostgreSQL
