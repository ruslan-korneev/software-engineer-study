# Типы данных в PostgreSQL

## Введение

PostgreSQL предоставляет богатый набор встроенных типов данных, а также возможность создавать собственные. Правильный выбор типа данных влияет на:
- **Целостность данных** — ограничения на уровне типа
- **Производительность** — размер хранения и скорость операций
- **Функциональность** — доступные операции и функции

## Числовые типы

### Целые числа

| Тип | Размер | Диапазон | Использование |
|-----|--------|----------|---------------|
| `SMALLINT` | 2 байта | -32,768 до 32,767 | Возраст, рейтинг |
| `INTEGER` | 4 байта | -2.1 млрд до 2.1 млрд | ID, счётчики |
| `BIGINT` | 8 байт | ±9.2 × 10^18 | Большие ID, финансы |

```sql
CREATE TABLE counters (
    small_count SMALLINT,     -- экономия памяти для малых значений
    regular_count INTEGER,    -- стандартный выбор
    big_count BIGINT          -- для очень больших чисел
);

-- Автоинкрементные типы
CREATE TABLE users (
    id SERIAL PRIMARY KEY,          -- INTEGER + sequence
    legacy_id BIGSERIAL,            -- BIGINT + sequence
    small_id SMALLSERIAL            -- SMALLINT + sequence
);

-- Современная альтернатива (рекомендуется)
CREATE TABLE products (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY
);
```

### Числа с плавающей точкой

```sql
-- REAL (4 байта) и DOUBLE PRECISION (8 байт)
-- НЕ используйте для денег — неточные!
CREATE TABLE measurements (
    temperature REAL,                    -- ~6 знаков точности
    precise_value DOUBLE PRECISION       -- ~15 знаков точности
);

-- Демонстрация неточности
SELECT 0.1::REAL + 0.2::REAL;
-- Результат: 0.30000001192092896 (не 0.3!)
```

### Точные числа (NUMERIC/DECIMAL)

```sql
-- NUMERIC(precision, scale) — точные вычисления
-- precision — общее количество цифр
-- scale — цифр после запятой
CREATE TABLE financial (
    price NUMERIC(10, 2),        -- до 99999999.99
    tax_rate NUMERIC(5, 4),      -- до 9.9999 (например, 0.2000)
    balance NUMERIC(15, 2),      -- большие суммы
    arbitrary NUMERIC            -- без ограничений (медленнее)
);

INSERT INTO financial (price, tax_rate) VALUES (1234.56, 0.2000);

-- NUMERIC гарантирует точность
SELECT 0.1::NUMERIC + 0.2::NUMERIC;  -- Результат: 0.3 (точно!)
```

## Строковые типы

### Основные типы

| Тип | Описание | Максимум |
|-----|----------|----------|
| `CHAR(n)` | Фиксированная длина (дополняется пробелами) | 10,485,760 символов |
| `VARCHAR(n)` | Переменная длина с лимитом | 10,485,760 символов |
| `TEXT` | Переменная длина без лимита | ~1 ГБ |

```sql
CREATE TABLE strings_demo (
    fixed_code CHAR(3),           -- всегда 3 символа ('AB' -> 'AB ')
    limited_name VARCHAR(100),    -- до 100 символов
    description TEXT              -- неограниченный текст
);

-- VARCHAR vs TEXT — практически идентичны по производительности
-- VARCHAR(n) полезен для валидации на уровне БД

-- Функции работы со строками
SELECT
    LENGTH('Привет'),           -- 6 (символов)
    OCTET_LENGTH('Привет'),     -- 12 (байт в UTF-8)
    UPPER('hello'),             -- HELLO
    LOWER('HELLO'),             -- hello
    TRIM('  text  '),           -- 'text'
    SUBSTRING('Hello' FROM 2 FOR 3),  -- 'ell'
    CONCAT('Hello', ' ', 'World'),    -- 'Hello World'
    'Hello' || ' ' || 'World';        -- 'Hello World' (конкатенация)
```

### Специальные строковые типы

```sql
-- UUID — универсальный уникальный идентификатор
CREATE TABLE sessions (
    id UUID PRIMARY KEY DEFAULT gen_random_uuid(),
    user_id INTEGER
);

-- Примеры UUID
SELECT gen_random_uuid();  -- e.g., 'a0eebc99-9c0b-4ef8-bb6d-6bb9bd380a11'

-- BYTEA — двоичные данные
CREATE TABLE files (
    id SERIAL PRIMARY KEY,
    name VARCHAR(255),
    content BYTEA
);

INSERT INTO files (name, content) VALUES ('test', '\x48656c6c6f');  -- 'Hello' в hex
```

## Типы даты и времени

### Основные типы

| Тип | Размер | Описание |
|-----|--------|----------|
| `DATE` | 4 байта | Только дата (без времени) |
| `TIME` | 8 байт | Только время (без даты) |
| `TIMESTAMP` | 8 байт | Дата и время без часового пояса |
| `TIMESTAMPTZ` | 8 байт | Дата и время с часовым поясом |
| `INTERVAL` | 16 байт | Временной интервал |

```sql
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_date DATE,
    start_time TIME,
    created_at TIMESTAMP,
    scheduled_at TIMESTAMPTZ,  -- рекомендуется для большинства случаев
    duration INTERVAL
);

-- Вставка данных
INSERT INTO events (event_date, start_time, created_at, scheduled_at, duration)
VALUES (
    '2024-03-15',                          -- DATE
    '14:30:00',                            -- TIME
    '2024-03-15 14:30:00',                 -- TIMESTAMP
    '2024-03-15 14:30:00+03',              -- TIMESTAMPTZ (с часовым поясом)
    '2 hours 30 minutes'                   -- INTERVAL
);

-- Текущие дата/время
SELECT
    CURRENT_DATE,              -- 2024-03-15
    CURRENT_TIME,              -- 14:30:00.123456+03
    CURRENT_TIMESTAMP,         -- 2024-03-15 14:30:00.123456+03
    NOW(),                     -- то же что CURRENT_TIMESTAMP
    LOCALTIME,                 -- без часового пояса
    LOCALTIMESTAMP;            -- без часового пояса
```

### Работа с датами

```sql
-- Извлечение компонентов
SELECT
    EXTRACT(YEAR FROM NOW()),
    EXTRACT(MONTH FROM NOW()),
    EXTRACT(DAY FROM NOW()),
    EXTRACT(HOUR FROM NOW()),
    EXTRACT(DOW FROM NOW()),   -- день недели (0 = воскресенье)
    DATE_PART('year', NOW());  -- альтернативный синтаксис

-- Арифметика с датами
SELECT
    NOW() + INTERVAL '1 day',
    NOW() - INTERVAL '1 week',
    NOW() + '3 months'::INTERVAL,
    '2024-12-31'::DATE - '2024-01-01'::DATE,  -- разница в днях
    AGE('2024-01-01', '1990-05-15');          -- возраст

-- Форматирование
SELECT
    TO_CHAR(NOW(), 'DD.MM.YYYY'),           -- 15.03.2024
    TO_CHAR(NOW(), 'HH24:MI:SS'),           -- 14:30:00
    TO_CHAR(NOW(), 'Day, DD Month YYYY');   -- Friday, 15 March 2024

-- Парсинг
SELECT
    TO_DATE('15/03/2024', 'DD/MM/YYYY'),
    TO_TIMESTAMP('15-03-2024 14:30', 'DD-MM-YYYY HH24:MI');
```

### Часовые пояса

```sql
-- Просмотр текущего часового пояса
SHOW timezone;

-- Установка для сессии
SET timezone TO 'Europe/Moscow';

-- TIMESTAMP vs TIMESTAMPTZ
CREATE TABLE tz_demo (
    without_tz TIMESTAMP,
    with_tz TIMESTAMPTZ
);

INSERT INTO tz_demo VALUES (
    '2024-03-15 12:00:00',
    '2024-03-15 12:00:00+00'  -- UTC
);

-- При выводе TIMESTAMPTZ конвертируется в текущий часовой пояс
SET timezone TO 'Europe/Moscow';
SELECT * FROM tz_demo;
-- without_tz: 2024-03-15 12:00:00
-- with_tz:    2024-03-15 15:00:00+03 (конвертировано из UTC)
```

## Логический тип

```sql
CREATE TABLE features (
    id SERIAL PRIMARY KEY,
    is_active BOOLEAN DEFAULT TRUE,
    is_premium BOOLEAN DEFAULT FALSE
);

-- Допустимые значения
INSERT INTO features (is_active, is_premium) VALUES
    (TRUE, FALSE),
    ('t', 'f'),
    ('true', 'false'),
    ('yes', 'no'),
    ('on', 'off'),
    ('1', '0');

-- В условиях
SELECT * FROM features WHERE is_active;  -- то же что is_active = TRUE
SELECT * FROM features WHERE NOT is_premium;
```

## JSON и JSONB

### Различия JSON и JSONB

| Аспект | JSON | JSONB |
|--------|------|-------|
| Хранение | Текст как есть | Бинарный формат |
| Дубликаты ключей | Сохраняются | Последний остаётся |
| Порядок ключей | Сохраняется | Не гарантирован |
| Индексация | Нет | GIN индексы |
| Производительность | Медленнее | Быстрее |

```sql
-- Используйте JSONB в большинстве случаев
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    attributes JSONB DEFAULT '{}'
);

-- Вставка JSON
INSERT INTO products (name, attributes) VALUES
    ('Ноутбук', '{"brand": "Dell", "ram": 16, "ssd": true}'),
    ('Телефон', '{"brand": "Apple", "model": "iPhone 15", "colors": ["black", "white"]}');

-- Операторы доступа
SELECT
    attributes->'brand',           -- JSONB: "Dell"
    attributes->>'brand',          -- TEXT: Dell
    attributes->'colors'->0,       -- JSONB: "black"
    attributes#>'{colors,0}',      -- JSONB: "black" (путь)
    attributes#>>'{colors,0}';     -- TEXT: black

-- Проверка наличия ключа
SELECT * FROM products WHERE attributes ? 'ssd';

-- Проверка значения
SELECT * FROM products WHERE attributes @> '{"brand": "Dell"}';

-- Модификация JSONB
UPDATE products SET
    attributes = attributes || '{"warranty": "2 years"}'  -- добавить
WHERE name = 'Ноутбук';

UPDATE products SET
    attributes = attributes - 'warranty'  -- удалить ключ
WHERE name = 'Ноутбук';

-- Индексация JSONB
CREATE INDEX idx_products_attrs ON products USING GIN (attributes);
CREATE INDEX idx_products_brand ON products USING BTREE ((attributes->>'brand'));
```

## Массивы

```sql
CREATE TABLE posts (
    id SERIAL PRIMARY KEY,
    title VARCHAR(200),
    tags TEXT[],
    scores INTEGER[]
);

-- Вставка массивов
INSERT INTO posts (title, tags, scores) VALUES
    ('PostgreSQL Tips', ARRAY['database', 'postgresql', 'tips'], ARRAY[5, 4, 5]),
    ('Web Development', '{"web", "frontend", "react"}', '{4, 3, 5}');

-- Операции с массивами
SELECT
    tags[1],                      -- первый элемент (индексация с 1!)
    array_length(tags, 1),        -- длина массива
    array_upper(tags, 1),         -- верхняя граница
    array_to_string(tags, ', '); -- 'database, postgresql, tips'

-- Поиск в массивах
SELECT * FROM posts WHERE 'postgresql' = ANY(tags);
SELECT * FROM posts WHERE tags @> ARRAY['database'];  -- содержит
SELECT * FROM posts WHERE tags && ARRAY['react', 'vue'];  -- пересечение

-- Добавление элемента
UPDATE posts SET tags = array_append(tags, 'sql') WHERE id = 1;
UPDATE posts SET tags = tags || 'tutorial' WHERE id = 1;

-- Удаление элемента
UPDATE posts SET tags = array_remove(tags, 'tips') WHERE id = 1;

-- Индексация массивов
CREATE INDEX idx_posts_tags ON posts USING GIN (tags);
```

## Специальные типы

### Сетевые типы

```sql
CREATE TABLE network_config (
    id SERIAL PRIMARY KEY,
    ip_address INET,        -- IP адрес (v4 или v6) с маской
    network CIDR,           -- сеть
    mac_address MACADDR
);

INSERT INTO network_config (ip_address, network, mac_address) VALUES
    ('192.168.1.100/24', '192.168.1.0/24', '08:00:2b:01:02:03');

-- Операции
SELECT
    ip_address << '192.168.0.0/16',  -- содержится в сети?
    network >> ip_address,            -- содержит IP?
    host(ip_address);                 -- только IP без маски
```

### Геометрические типы

```sql
CREATE TABLE shapes (
    id SERIAL PRIMARY KEY,
    location POINT,
    area BOX,
    path PATH
);

INSERT INTO shapes (location, area) VALUES
    ('(10, 20)', '((0,0), (100,100))');

-- Расстояние между точками
SELECT '(0,0)'::POINT <-> '(3,4)'::POINT;  -- 5
```

### Range типы

```sql
CREATE TABLE reservations (
    id SERIAL PRIMARY KEY,
    room_id INTEGER,
    during TSTZRANGE  -- диапазон временных меток
);

INSERT INTO reservations (room_id, during) VALUES
    (1, '[2024-03-15 10:00, 2024-03-15 12:00)');

-- Проверка пересечений
SELECT * FROM reservations
WHERE during && '[2024-03-15 11:00, 2024-03-15 13:00)';

-- Constraint исключения (exclusion constraint) — предотвращает пересечения
ALTER TABLE reservations ADD CONSTRAINT no_overlap
    EXCLUDE USING GIST (room_id WITH =, during WITH &&);
```

## Создание собственных типов

```sql
-- Перечисление (ENUM)
CREATE TYPE order_status AS ENUM ('pending', 'confirmed', 'shipped', 'delivered');

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    status order_status DEFAULT 'pending'
);

-- Составной тип (Composite)
CREATE TYPE address AS (
    street VARCHAR(100),
    city VARCHAR(50),
    postal_code VARCHAR(10)
);

CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    home_address address
);

-- Домен (Domain) — тип с ограничениями
CREATE DOMAIN email AS VARCHAR(255)
    CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    user_email email  -- валидация на уровне типа
);
```

## Best Practices

1. **Для ID используйте INTEGER или BIGINT** с GENERATED ALWAYS AS IDENTITY
2. **Для денег используйте NUMERIC**, никогда FLOAT
3. **Для дат используйте TIMESTAMPTZ**, храните в UTC
4. **Для JSON используйте JSONB** вместо JSON
5. **VARCHAR vs TEXT** — используйте VARCHAR(n) если нужна валидация длины
6. **Не злоупотребляйте массивами** — для связей используйте отдельные таблицы
7. **Выбирайте наименьший подходящий тип** — экономит место и ускоряет запросы

```sql
-- Хороший пример таблицы с правильными типами
CREATE TABLE orders (
    id INTEGER GENERATED ALWAYS AS IDENTITY PRIMARY KEY,
    customer_id INTEGER NOT NULL REFERENCES customers(id),
    order_number VARCHAR(20) UNIQUE NOT NULL,
    total_amount NUMERIC(12, 2) NOT NULL CHECK (total_amount >= 0),
    status order_status DEFAULT 'pending',
    metadata JSONB DEFAULT '{}',
    created_at TIMESTAMPTZ DEFAULT NOW(),
    updated_at TIMESTAMPTZ DEFAULT NOW()
);
```
