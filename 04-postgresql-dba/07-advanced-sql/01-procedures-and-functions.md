# Процедуры и Функции в PostgreSQL

## Введение

Процедуры и функции — это именованные блоки кода, которые хранятся в базе данных и могут быть вызваны для выполнения определённых задач. Они позволяют инкапсулировать логику, повторно использовать код и улучшать производительность за счёт уменьшения сетевого трафика между приложением и базой данных.

## Различия между функциями и процедурами

| Характеристика | Функция (FUNCTION) | Процедура (PROCEDURE) |
|----------------|--------------------|-----------------------|
| Возвращаемое значение | Обязательно | Не обязательно |
| Управление транзакциями | Нет (работает в рамках вызывающей транзакции) | Да (может COMMIT/ROLLBACK) |
| Вызов | SELECT, FROM, WHERE | CALL |
| Появление в PostgreSQL | Изначально | С версии 11 |
| Использование в выражениях | Да | Нет |

## Функции (Functions)

### Создание функции

```sql
CREATE OR REPLACE FUNCTION function_name(param1 type1, param2 type2)
RETURNS return_type
LANGUAGE plpgsql
AS $$
DECLARE
    -- объявление переменных
BEGIN
    -- тело функции
    RETURN value;
END;
$$;
```

### Пример: Функция для расчёта скидки

```sql
CREATE OR REPLACE FUNCTION calculate_discount(
    price NUMERIC,
    discount_percent NUMERIC DEFAULT 10
)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    discounted_price NUMERIC;
BEGIN
    -- Проверка входных данных
    IF price < 0 THEN
        RAISE EXCEPTION 'Цена не может быть отрицательной: %', price;
    END IF;

    IF discount_percent < 0 OR discount_percent > 100 THEN
        RAISE EXCEPTION 'Скидка должна быть от 0 до 100: %', discount_percent;
    END IF;

    discounted_price := price * (1 - discount_percent / 100);

    RETURN ROUND(discounted_price, 2);
END;
$$;

-- Использование
SELECT calculate_discount(100, 15);  -- 85.00
SELECT calculate_discount(100);       -- 90.00 (скидка по умолчанию 10%)
```

### Функции, возвращающие множество строк

```sql
-- Возврат SETOF записей
CREATE OR REPLACE FUNCTION get_active_users()
RETURNS SETOF users
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT * FROM users WHERE is_active = TRUE;
END;
$$;

-- Использование
SELECT * FROM get_active_users();
```

### Функция с TABLE в возврате

```sql
CREATE OR REPLACE FUNCTION get_user_statistics(user_id_param INTEGER)
RETURNS TABLE(
    total_orders INTEGER,
    total_spent NUMERIC,
    avg_order_value NUMERIC,
    last_order_date DATE
)
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN QUERY
    SELECT
        COUNT(*)::INTEGER,
        COALESCE(SUM(amount), 0),
        COALESCE(AVG(amount), 0),
        MAX(order_date)::DATE
    FROM orders
    WHERE user_id = user_id_param;
END;
$$;

-- Использование
SELECT * FROM get_user_statistics(42);
```

### OUT параметры

```sql
CREATE OR REPLACE FUNCTION get_order_summary(
    order_id_param INTEGER,
    OUT total_items INTEGER,
    OUT total_amount NUMERIC,
    OUT status VARCHAR
)
LANGUAGE plpgsql
AS $$
BEGIN
    SELECT
        SUM(quantity),
        SUM(price * quantity),
        o.status
    INTO total_items, total_amount, status
    FROM order_items oi
    JOIN orders o ON o.id = oi.order_id
    WHERE o.id = order_id_param
    GROUP BY o.status;
END;
$$;

-- Использование
SELECT * FROM get_order_summary(123);
```

## Процедуры (Procedures)

### Создание процедуры

```sql
CREATE OR REPLACE PROCEDURE procedure_name(param1 type1, param2 type2)
LANGUAGE plpgsql
AS $$
BEGIN
    -- тело процедуры
END;
$$;
```

### Пример: Процедура перевода средств

```sql
CREATE OR REPLACE PROCEDURE transfer_funds(
    from_account INTEGER,
    to_account INTEGER,
    amount NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    current_balance NUMERIC;
BEGIN
    -- Проверка баланса
    SELECT balance INTO current_balance
    FROM accounts
    WHERE id = from_account
    FOR UPDATE;

    IF current_balance < amount THEN
        RAISE EXCEPTION 'Недостаточно средств. Баланс: %, требуется: %',
            current_balance, amount;
    END IF;

    -- Списание
    UPDATE accounts
    SET balance = balance - amount
    WHERE id = from_account;

    -- Зачисление
    UPDATE accounts
    SET balance = balance + amount
    WHERE id = to_account;

    -- Логирование
    INSERT INTO transactions(from_account, to_account, amount, created_at)
    VALUES (from_account, to_account, amount, NOW());

    -- Фиксация транзакции
    COMMIT;

    RAISE NOTICE 'Перевод выполнен успешно';
END;
$$;

-- Вызов процедуры
CALL transfer_funds(1, 2, 500.00);
```

### Процедура с управлением транзакциями

```sql
CREATE OR REPLACE PROCEDURE batch_update_prices(
    category_id INTEGER,
    increase_percent NUMERIC
)
LANGUAGE plpgsql
AS $$
DECLARE
    product_record RECORD;
    batch_size INTEGER := 100;
    updated_count INTEGER := 0;
BEGIN
    FOR product_record IN
        SELECT id FROM products WHERE category = category_id
    LOOP
        UPDATE products
        SET price = price * (1 + increase_percent / 100)
        WHERE id = product_record.id;

        updated_count := updated_count + 1;

        -- Коммит каждые batch_size записей
        IF updated_count % batch_size = 0 THEN
            COMMIT;
            RAISE NOTICE 'Обновлено записей: %', updated_count;
        END IF;
    END LOOP;

    -- Финальный коммит
    COMMIT;
    RAISE NOTICE 'Всего обновлено: %', updated_count;
END;
$$;
```

## Языки программирования

### PL/pgSQL (основной)

```sql
CREATE OR REPLACE FUNCTION plpgsql_example()
RETURNS TEXT
LANGUAGE plpgsql
AS $$
BEGIN
    RETURN 'Hello from PL/pgSQL';
END;
$$;
```

### SQL (простые функции)

```sql
CREATE OR REPLACE FUNCTION sql_example(x INTEGER, y INTEGER)
RETURNS INTEGER
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT x + y;
$$;
```

### PL/Python (требует расширение)

```sql
CREATE EXTENSION plpython3u;

CREATE OR REPLACE FUNCTION python_example(name TEXT)
RETURNS TEXT
LANGUAGE plpython3u
AS $$
    return f"Hello, {name}!"
$$;
```

## Модификаторы функций

### VOLATILE, STABLE, IMMUTABLE

```sql
-- IMMUTABLE: всегда возвращает одинаковый результат для одинаковых аргументов
-- Не обращается к базе, может кэшироваться
CREATE FUNCTION add_numbers(a INTEGER, b INTEGER)
RETURNS INTEGER
IMMUTABLE
LANGUAGE sql
AS $$ SELECT a + b; $$;

-- STABLE: возвращает одинаковый результат в пределах одного запроса
-- Только читает данные
CREATE FUNCTION get_setting_value(key TEXT)
RETURNS TEXT
STABLE
LANGUAGE sql
AS $$ SELECT value FROM settings WHERE name = key; $$;

-- VOLATILE (по умолчанию): может возвращать разные значения
-- Может изменять данные
CREATE FUNCTION generate_uuid()
RETURNS UUID
VOLATILE
LANGUAGE sql
AS $$ SELECT gen_random_uuid(); $$;
```

### SECURITY DEFINER vs SECURITY INVOKER

```sql
-- SECURITY DEFINER: выполняется с правами создателя функции
CREATE FUNCTION admin_only_function()
RETURNS VOID
SECURITY DEFINER
SET search_path = public
LANGUAGE plpgsql
AS $$
BEGIN
    -- Даже обычный пользователь получит права создателя
    DELETE FROM sensitive_logs WHERE created_at < NOW() - INTERVAL '30 days';
END;
$$;

-- SECURITY INVOKER (по умолчанию): выполняется с правами вызывающего
```

### PARALLEL SAFE

```sql
-- Функция безопасна для параллельного выполнения
CREATE FUNCTION safe_calculation(x NUMERIC)
RETURNS NUMERIC
PARALLEL SAFE
IMMUTABLE
LANGUAGE sql
AS $$ SELECT x * 2; $$;
```

## Обработка ошибок

```sql
CREATE OR REPLACE FUNCTION safe_divide(a NUMERIC, b NUMERIC)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
DECLARE
    result NUMERIC;
BEGIN
    result := a / b;
    RETURN result;
EXCEPTION
    WHEN division_by_zero THEN
        RAISE NOTICE 'Деление на ноль, возвращаем NULL';
        RETURN NULL;
    WHEN numeric_value_out_of_range THEN
        RAISE EXCEPTION 'Результат вне допустимого диапазона';
    WHEN OTHERS THEN
        RAISE EXCEPTION 'Неизвестная ошибка: %', SQLERRM;
END;
$$;
```

### Пользовательские исключения

```sql
CREATE OR REPLACE FUNCTION validate_email(email TEXT)
RETURNS BOOLEAN
LANGUAGE plpgsql
AS $$
BEGIN
    IF email IS NULL OR email = '' THEN
        RAISE EXCEPTION 'Email не может быть пустым'
            USING ERRCODE = 'check_violation';
    END IF;

    IF email !~ '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$' THEN
        RAISE EXCEPTION 'Неверный формат email: %', email
            USING ERRCODE = 'check_violation',
                  HINT = 'Email должен быть в формате user@domain.com';
    END IF;

    RETURN TRUE;
END;
$$;
```

## Полиморфные функции

```sql
-- Функция, работающая с любым типом массива
CREATE OR REPLACE FUNCTION array_first(arr ANYARRAY)
RETURNS ANYELEMENT
LANGUAGE sql
IMMUTABLE
AS $$
    SELECT arr[1];
$$;

-- Использование
SELECT array_first(ARRAY[1, 2, 3]);        -- 1
SELECT array_first(ARRAY['a', 'b', 'c']);  -- 'a'
```

## Best Practices

### 1. Именование

```sql
-- Используйте понятные имена с префиксами
-- fn_ для функций, sp_ для процедур
CREATE FUNCTION fn_calculate_tax(...);
CREATE PROCEDURE sp_process_order(...);
```

### 2. Документирование

```sql
CREATE OR REPLACE FUNCTION calculate_shipping_cost(
    weight_kg NUMERIC,
    distance_km NUMERIC,
    express BOOLEAN DEFAULT FALSE
)
RETURNS NUMERIC
LANGUAGE plpgsql
AS $$
/*
 * Расчёт стоимости доставки
 *
 * Параметры:
 *   weight_kg  - вес посылки в килограммах
 *   distance_km - расстояние в километрах
 *   express    - экспресс-доставка (x2 к стоимости)
 *
 * Возвращает: стоимость доставки в рублях
 *
 * Автор: Иванов И.И.
 * Дата: 2024-01-15
 */
DECLARE
    base_cost NUMERIC := 100;
    per_kg NUMERIC := 50;
    per_km NUMERIC := 2;
    total NUMERIC;
BEGIN
    total := base_cost + (weight_kg * per_kg) + (distance_km * per_km);

    IF express THEN
        total := total * 2;
    END IF;

    RETURN ROUND(total, 2);
END;
$$;

-- Добавление комментария к функции
COMMENT ON FUNCTION calculate_shipping_cost(NUMERIC, NUMERIC, BOOLEAN)
IS 'Расчёт стоимости доставки на основе веса и расстояния';
```

### 3. Безопасность

```sql
-- Всегда устанавливайте search_path для SECURITY DEFINER
CREATE FUNCTION sensitive_operation()
RETURNS VOID
SECURITY DEFINER
SET search_path = public, pg_temp
LANGUAGE plpgsql
AS $$
BEGIN
    -- Защита от SQL injection через search_path
END;
$$;
```

### 4. Производительность

```sql
-- Используйте правильные модификаторы
-- IMMUTABLE для чистых функций
CREATE FUNCTION format_phone(phone TEXT)
RETURNS TEXT
IMMUTABLE
PARALLEL SAFE
LANGUAGE sql
AS $$
    SELECT regexp_replace(phone, '(\d{3})(\d{3})(\d{4})', '+7 (\1) \2-\3');
$$;
```

## Типичные ошибки

### 1. Игнорирование NULL

```sql
-- Плохо: не обрабатывает NULL
CREATE FUNCTION bad_concat(a TEXT, b TEXT)
RETURNS TEXT
LANGUAGE sql
AS $$ SELECT a || b; $$;

-- Хорошо: обрабатывает NULL
CREATE FUNCTION good_concat(a TEXT, b TEXT)
RETURNS TEXT
LANGUAGE sql
AS $$ SELECT COALESCE(a, '') || COALESCE(b, ''); $$;
```

### 2. Утечка соединений в циклах

```sql
-- Плохо: много мелких запросов в цикле
FOR rec IN SELECT id FROM items LOOP
    SELECT * INTO details FROM item_details WHERE item_id = rec.id;
END LOOP;

-- Хорошо: один запрос с JOIN
FOR rec IN
    SELECT i.*, d.*
    FROM items i
    JOIN item_details d ON d.item_id = i.id
LOOP
    -- обработка
END LOOP;
```

### 3. Неправильное использование EXCEPTION

```sql
-- Плохо: EXCEPTION блок создаёт подтранзакцию
BEGIN
    -- код
EXCEPTION
    WHEN OTHERS THEN NULL;  -- Игнорировать все ошибки — плохая практика
END;

-- Хорошо: обрабатывать конкретные исключения
BEGIN
    -- код
EXCEPTION
    WHEN unique_violation THEN
        -- конкретная обработка
END;
```

## Управление функциями

```sql
-- Просмотр всех функций
SELECT proname, proargtypes, prosrc
FROM pg_proc
WHERE pronamespace = 'public'::regnamespace;

-- Удаление функции
DROP FUNCTION IF EXISTS function_name(param_types);

-- Удаление процедуры
DROP PROCEDURE IF EXISTS procedure_name(param_types);

-- Изменение владельца
ALTER FUNCTION function_name(param_types) OWNER TO new_owner;

-- Выдача прав
GRANT EXECUTE ON FUNCTION function_name(param_types) TO role_name;
```

## Заключение

Функции и процедуры — мощный инструмент PostgreSQL для инкапсуляции бизнес-логики на уровне базы данных. Функции идеальны для вычислений и возврата данных, процедуры — для операций с управлением транзакциями. Правильное использование модификаторов (IMMUTABLE, STABLE, VOLATILE) и обработка ошибок критически важны для производительности и надёжности.
