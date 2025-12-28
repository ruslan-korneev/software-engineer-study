# PostgreSQL vs другие RDBMS

[prev: 02-rdbms-benefits-limitations](./02-rdbms-benefits-limitations.md) | [next: 04-postgresql-vs-nosql](./04-postgresql-vs-nosql.md)
---

## Обзор основных RDBMS

### PostgreSQL

**Тип:** Open-source, объектно-реляционная СУБД
**Лицензия:** PostgreSQL License (очень либеральная, похожа на MIT/BSD)
**Слоган:** "The World's Most Advanced Open Source Relational Database"

### MySQL / MariaDB

**Тип:** Open-source, реляционная СУБД
**Лицензия:** GPL (MySQL), GPL/BSD (MariaDB)
**Владелец:** Oracle (MySQL), MariaDB Foundation (MariaDB)

### Oracle Database

**Тип:** Коммерческая, enterprise СУБД
**Лицензия:** Проприетарная (очень дорогая)
**Владелец:** Oracle Corporation

### Microsoft SQL Server

**Тип:** Коммерческая (есть Express бесплатная версия)
**Лицензия:** Проприетарная
**Владелец:** Microsoft

### SQLite

**Тип:** Встраиваемая, serverless СУБД
**Лицензия:** Public Domain

## Сравнение PostgreSQL с MySQL/MariaDB

### Соответствие стандартам SQL

**PostgreSQL** строго следует стандарту SQL:

```sql
-- PostgreSQL: стандартный синтаксис работает
SELECT * FROM users
WHERE created_at BETWEEN '2024-01-01' AND '2024-12-31';

-- PostgreSQL: оконные функции (стандарт SQL:2003)
SELECT
    name,
    salary,
    AVG(salary) OVER (PARTITION BY department_id) as dept_avg
FROM employees;

-- PostgreSQL: CTE (Common Table Expressions)
WITH regional_sales AS (
    SELECT region, SUM(amount) AS total
    FROM orders
    GROUP BY region
)
SELECT * FROM regional_sales WHERE total > 1000000;
```

**MySQL** исторически имел нестандартные расширения:

```sql
-- MySQL: нестандартный GROUP BY (без ONLY_FULL_GROUP_BY)
-- Может работать, но технически некорректно
SELECT name, department, MAX(salary)
FROM employees
GROUP BY department;  -- name не в GROUP BY!

-- В PostgreSQL это вызовет ошибку (правильное поведение)
```

### Типы данных

**PostgreSQL** имеет богатую систему типов:

```sql
-- Массивы
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    tags TEXT[]  -- массив тегов
);
INSERT INTO posts (tags) VALUES (ARRAY['postgresql', 'database']);
SELECT * FROM posts WHERE 'postgresql' = ANY(tags);

-- JSONB с индексированием
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    data JSONB
);
CREATE INDEX idx_events_data ON events USING GIN(data);
SELECT * FROM events WHERE data @> '{"type": "click"}';

-- UUID как нативный тип
CREATE TABLE sessions (
    id UUID DEFAULT gen_random_uuid() PRIMARY KEY
);

-- Диапазоны
CREATE TABLE reservations (
    room_id INTEGER,
    during TSTZRANGE,
    EXCLUDE USING GIST (room_id WITH =, during WITH &&)
);

-- Составные типы
CREATE TYPE address AS (
    street VARCHAR(100),
    city VARCHAR(50),
    postal_code VARCHAR(20)
);
```

**MySQL** имеет более ограниченный набор:

```sql
-- MySQL: JSON (с MySQL 5.7+)
CREATE TABLE events (
    id INT AUTO_INCREMENT PRIMARY KEY,
    data JSON
);

-- MySQL: нет нативных массивов, UUID, диапазонов
-- Приходится использовать VARCHAR или JSON
```

### Расширяемость

**PostgreSQL** — самая расширяемая СУБД:

```sql
-- Создание собственных типов
CREATE TYPE mood AS ENUM ('sad', 'ok', 'happy');

-- Создание собственных функций
CREATE FUNCTION calculate_discount(price NUMERIC, percent INTEGER)
RETURNS NUMERIC AS $$
BEGIN
    RETURN price * (1 - percent / 100.0);
END;
$$ LANGUAGE plpgsql;

-- Создание собственных операторов
CREATE OPERATOR @@ (
    LEFTARG = text,
    RIGHTARG = text,
    FUNCTION = my_match_function
);

-- Расширения
CREATE EXTENSION postgis;      -- Геоданные
CREATE EXTENSION pg_trgm;      -- Триграммы для поиска
CREATE EXTENSION uuid-ossp;    -- UUID функции
```

### Полнотекстовый поиск

**PostgreSQL** — встроенный полнотекстовый поиск:

```sql
-- Создание полнотекстового индекса
CREATE INDEX idx_articles_fts ON articles
USING GIN(to_tsvector('russian', title || ' ' || content));

-- Поиск
SELECT title, ts_rank(
    to_tsvector('russian', title || ' ' || content),
    to_tsquery('russian', 'PostgreSQL & база')
) AS rank
FROM articles
WHERE to_tsvector('russian', title || ' ' || content)
    @@ to_tsquery('russian', 'PostgreSQL & база')
ORDER BY rank DESC;
```

**MySQL** — FULLTEXT индексы (ограниченные возможности):

```sql
-- MySQL: базовый полнотекстовый поиск
ALTER TABLE articles ADD FULLTEXT(title, content);
SELECT * FROM articles
WHERE MATCH(title, content) AGAINST('PostgreSQL database');
```

### Сравнительная таблица PostgreSQL vs MySQL

| Функция | PostgreSQL | MySQL |
|---------|------------|-------|
| Стандарт SQL | Строгое соответствие | Частичное |
| ACID | Полная поддержка | Зависит от движка (InnoDB — да) |
| Массивы | Да | Нет (через JSON) |
| JSONB | Да, с индексами | JSON (менее эффективный) |
| Полнотекстовый поиск | Продвинутый | Базовый |
| Оконные функции | Полная поддержка | С MySQL 8.0 |
| CTE (WITH) | Да | С MySQL 8.0 |
| Материализованные view | Да | Нет |
| Расширения | Богатая экосистема | Ограниченно |
| Репликация | Логическая + физическая | Master-Slave, Group |
| Производительность чтения | Отличная | Отличная |
| Производительность записи | Отличная | Хорошая (InnoDB) |

## Сравнение PostgreSQL с Oracle

### Схожие возможности

PostgreSQL часто называют "open-source Oracle" из-за схожего функционала:

```sql
-- PL/pgSQL похож на PL/SQL Oracle
CREATE OR REPLACE FUNCTION process_order(order_id INTEGER)
RETURNS VOID AS $$
DECLARE
    v_total NUMERIC;
    v_status VARCHAR(50);
BEGIN
    SELECT total, status INTO v_total, v_status
    FROM orders WHERE id = order_id;

    IF v_status = 'pending' THEN
        -- Обработка заказа
        UPDATE orders SET status = 'processing' WHERE id = order_id;
    END IF;

EXCEPTION
    WHEN NO_DATA_FOUND THEN
        RAISE EXCEPTION 'Order % not found', order_id;
END;
$$ LANGUAGE plpgsql;
```

### Отличия от Oracle

| Аспект | PostgreSQL | Oracle |
|--------|------------|--------|
| **Стоимость** | Бесплатно | $17,500-47,500+ за процессор/год |
| **Лицензирование** | Простое | Сложное, много ограничений |
| **Кластеризация** | Patroni, Citus | RAC (Real Application Clusters) |
| **Партиционирование** | Декларативное (с 10+) | Продвинутое |
| **Flashback** | Нет (но есть pg_back_rest) | Да |
| **Enterprise функции** | Через расширения | Встроенные |
| **Поддержка** | Сообщество + коммерческая | Oracle Support |

### Миграция с Oracle на PostgreSQL

```sql
-- Oracle: NVL
SELECT NVL(column_name, 'default') FROM table_name;

-- PostgreSQL: COALESCE (стандарт SQL)
SELECT COALESCE(column_name, 'default') FROM table_name;

-- Oracle: SYSDATE
SELECT SYSDATE FROM DUAL;

-- PostgreSQL: CURRENT_TIMESTAMP
SELECT CURRENT_TIMESTAMP;

-- Oracle: ROWNUM
SELECT * FROM employees WHERE ROWNUM <= 10;

-- PostgreSQL: LIMIT
SELECT * FROM employees LIMIT 10;

-- Oracle: (+) для outer join
SELECT * FROM a, b WHERE a.id = b.id(+);

-- PostgreSQL: стандартный JOIN
SELECT * FROM a LEFT JOIN b ON a.id = b.id;
```

## Сравнение PostgreSQL с SQL Server

### Основные различия

```sql
-- SQL Server: TOP
SELECT TOP 10 * FROM users;

-- PostgreSQL: LIMIT
SELECT * FROM users LIMIT 10;

-- SQL Server: идентификаторы в []
SELECT [column name] FROM [table name];

-- PostgreSQL: идентификаторы в ""
SELECT "column name" FROM "table name";

-- SQL Server: GETDATE()
SELECT GETDATE();

-- PostgreSQL: CURRENT_TIMESTAMP или NOW()
SELECT CURRENT_TIMESTAMP;

-- SQL Server: ISNULL
SELECT ISNULL(column, 'default') FROM table_name;

-- PostgreSQL: COALESCE
SELECT COALESCE(column, 'default') FROM table_name;
```

### Сравнительная таблица

| Аспект | PostgreSQL | SQL Server |
|--------|------------|------------|
| **Платформы** | Linux, Windows, macOS | Windows, Linux |
| **Стоимость** | Бесплатно | $931-15,123+ за ядро |
| **Интеграция** | Универсальная | Лучше с .NET/Azure |
| **GUI** | pgAdmin, DBeaver | SSMS (отличный) |
| **BI интеграция** | Через коннекторы | SSIS, SSRS, SSAS |
| **CLR интеграция** | Нет | Да (.NET в БД) |
| **Производительность** | Отличная | Отличная |

## Сравнение PostgreSQL с SQLite

SQLite — это совершенно другой класс СУБД:

| Аспект | PostgreSQL | SQLite |
|--------|------------|--------|
| **Архитектура** | Клиент-сервер | Встраиваемая |
| **Конкурентность** | Высокая | Ограниченная (file locking) |
| **Размер БД** | Терабайты | Гигабайты |
| **Использование** | Сервер приложений | Мобильные, десктоп, тесты |
| **Настройка** | Требуется | Не нужна |
| **Типизация** | Строгая | Гибкая (affinity) |

```sql
-- SQLite: динамическая типизация
CREATE TABLE test (value);
INSERT INTO test VALUES ('text');
INSERT INTO test VALUES (123);      -- Работает!
INSERT INTO test VALUES (45.67);    -- Работает!

-- PostgreSQL: строгая типизация
CREATE TABLE test (value INTEGER);
INSERT INTO test VALUES ('text');   -- Ошибка!
```

## Почему выбирают PostgreSQL

### 1. Лицензия

PostgreSQL License позволяет:
- Использовать в коммерческих проектах бесплатно
- Модифицировать код
- Не раскрывать свои изменения
- Нет ограничений на количество ядер/памяти

### 2. Функциональность

- Полная поддержка ACID
- Продвинутые типы данных (JSON, массивы, геоданные)
- Расширения для любых задач
- Полнотекстовый поиск
- Оконные функции, CTE, lateral joins

### 3. Надёжность

- Десятилетия разработки
- Строгое тестирование
- Нет "vendor lock-in"
- Активное сообщество

### 4. Масштабируемость

- Репликация (streaming, logical)
- Партиционирование
- Расширения для шардирования (Citus)
- Connection pooling (PgBouncer)

## Best Practices при выборе СУБД

1. **Оцените требования проекта** — не все проекты требуют enterprise СУБД
2. **Учитывайте экспертизу команды** — знакомство с СУБД важно
3. **Планируйте миграцию** — переход между СУБД может быть дорогим
4. **Тестируйте производительность** — бенчмарки на реальных данных
5. **Считайте TCO** — не только лицензия, но и поддержка, обучение
6. **Проверяйте совместимость** — с вашим стеком технологий

## Заключение

PostgreSQL — отличный выбор для большинства проектов благодаря:
- Бесплатности и открытости
- Богатой функциональности
- Строгому соответствию стандартам
- Активному развитию
- Отличной производительности

MySQL остаётся хорошим выбором для:
- Простых веб-приложений
- Проектов с существующей экспертизой в MySQL
- Интеграции с PHP/WordPress экосистемой

Oracle/SQL Server оправданы когда:
- Есть бюджет на лицензии
- Требуется enterprise поддержка
- Существующая инфраструктура завязана на этих СУБД

---
[prev: 02-rdbms-benefits-limitations](./02-rdbms-benefits-limitations.md) | [next: 04-postgresql-vs-nosql](./04-postgresql-vs-nosql.md)