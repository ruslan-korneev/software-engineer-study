# GiST индексы

## Введение

GiST (Generalized Search Tree) — это обобщённая структура индекса, которая позволяет индексировать нестандартные типы данных. GiST — это **фреймворк** для создания различных типов индексов, а не конкретная структура данных.

---

## Основные применения GiST

GiST используется для:
- **Геометрических данных** (точки, линии, полигоны)
- **Range-типов** (диапазоны дат, чисел)
- **Полнотекстового поиска** (tsvector)
- **Ближайшего соседа** (KNN-поиск)
- **Иерархических данных** (ltree)

---

## Геометрические данные и PostGIS

### Встроенные геометрические типы

```sql
-- Создание таблицы с геометрическими типами
CREATE TABLE shapes (
    id SERIAL PRIMARY KEY,
    name TEXT,
    location point,        -- Точка (x, y)
    area box,             -- Прямоугольник
    region polygon,       -- Многоугольник
    path_line path        -- Путь/линия
);

-- Создание GiST индекса
CREATE INDEX idx_shapes_location ON shapes USING gist (location);
CREATE INDEX idx_shapes_area ON shapes USING gist (area);

-- Вставка данных
INSERT INTO shapes (name, location, area) VALUES
('Office', point(55.751244, 37.618423), box(point(55.75, 37.61), point(55.76, 37.63))),
('Shop', point(55.753215, 37.622504), box(point(55.75, 37.62), point(55.76, 37.64)));
```

### Геометрические операторы

```sql
-- Поиск точек в прямоугольнике
SELECT * FROM shapes
WHERE location <@ box(point(55.7, 37.5), point(55.8, 37.7));

-- Проверка пересечения областей
SELECT * FROM shapes
WHERE area && box(point(55.75, 37.615), point(55.755, 37.625));

-- Операторы для геометрии:
-- <@  содержится в
-- @>  содержит
-- &&  пересекается
-- <<  слева от
-- >>  справа от
-- ~=  приблизительно равно
```

### PostGIS для реальных геоданных

```sql
-- Установка PostGIS
CREATE EXTENSION postgis;

-- Таблица с геоданными
CREATE TABLE places (
    id SERIAL PRIMARY KEY,
    name TEXT,
    location geography(POINT, 4326)  -- WGS84 координаты
);

-- GiST индекс для геоданных
CREATE INDEX idx_places_location ON places USING gist (location);

-- Вставка данных (долгота, широта)
INSERT INTO places (name, location) VALUES
('Кремль', ST_MakePoint(37.617635, 55.752023)),
('Эрмитаж', ST_MakePoint(30.314533, 59.939848));

-- Поиск мест в радиусе 1 км от точки
SELECT name, ST_Distance(location, ST_MakePoint(37.618, 55.751)::geography) as distance
FROM places
WHERE ST_DWithin(location, ST_MakePoint(37.618, 55.751)::geography, 1000)
ORDER BY distance;

-- Поиск ближайших N мест (KNN)
SELECT name, location <-> ST_MakePoint(37.618, 55.751)::geography as distance
FROM places
ORDER BY location <-> ST_MakePoint(37.618, 55.751)::geography
LIMIT 5;
```

---

## Range Types

### Диапазонные типы данных

PostgreSQL поддерживает range-типы, идеально подходящие для GiST:

```sql
-- Встроенные range-типы:
-- int4range, int8range     — целочисленные диапазоны
-- numrange                  — числовые диапазоны
-- tsrange, tstzrange       — временные диапазоны
-- daterange                 — диапазоны дат

-- Создание таблицы бронирований
CREATE TABLE room_bookings (
    id SERIAL PRIMARY KEY,
    room_id INTEGER,
    booking_period tstzrange,
    guest_name TEXT
);

-- GiST индекс для поиска пересечений
CREATE INDEX idx_bookings_period ON room_bookings USING gist (booking_period);

-- Вставка бронирований
INSERT INTO room_bookings (room_id, booking_period, guest_name) VALUES
(1, '[2024-01-15, 2024-01-20)', 'Иванов'),
(1, '[2024-01-25, 2024-01-30)', 'Петров'),
(2, '[2024-01-18, 2024-01-22)', 'Сидоров');
```

### Операторы для range-типов

```sql
-- Проверка пересечения (overlap)
SELECT * FROM room_bookings
WHERE booking_period && '[2024-01-17, 2024-01-19)'::tstzrange;

-- Содержит значение
SELECT * FROM room_bookings
WHERE booking_period @> '2024-01-16'::timestamp;

-- Содержится в другом диапазоне
SELECT * FROM room_bookings
WHERE booking_period <@ '[2024-01-01, 2024-02-01)'::tstzrange;

-- Операторы:
-- &&   пересекается
-- @>   содержит
-- <@   содержится
-- <<   строго до
-- >>   строго после
-- &<   не простирается правее
-- &>   не простирается левее
-- -|-  примыкает
```

### Исключающие ограничения (Exclusion Constraints)

```sql
-- Предотвращение пересечения бронирований одной комнаты
CREATE TABLE room_bookings_v2 (
    id SERIAL PRIMARY KEY,
    room_id INTEGER,
    booking_period tstzrange,
    guest_name TEXT,
    -- Constraint: в одной комнате периоды не могут пересекаться
    EXCLUDE USING gist (
        room_id WITH =,
        booking_period WITH &&
    )
);

-- Попытка вставить пересекающееся бронирование вызовет ошибку
INSERT INTO room_bookings_v2 (room_id, booking_period, guest_name)
VALUES (1, '[2024-01-15, 2024-01-20)', 'Иванов');

INSERT INTO room_bookings_v2 (room_id, booking_period, guest_name)
VALUES (1, '[2024-01-18, 2024-01-22)', 'Петров');
-- ERROR: conflicting key value violates exclusion constraint
```

---

## Полнотекстовый поиск

### tsvector и GiST

```sql
-- Таблица статей
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    body TEXT,
    search_vector tsvector
);

-- GiST индекс для полнотекстового поиска
CREATE INDEX idx_articles_search ON articles USING gist (search_vector);

-- Автоматическое обновление search_vector
CREATE FUNCTION articles_search_trigger() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('russian', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('russian', COALESCE(NEW.body, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER articles_search_update
BEFORE INSERT OR UPDATE ON articles
FOR EACH ROW EXECUTE FUNCTION articles_search_trigger();

-- Поиск
SELECT title, ts_rank(search_vector, query) as rank
FROM articles, to_tsquery('russian', 'программирование & python') query
WHERE search_vector @@ query
ORDER BY rank DESC;
```

### GiST vs GIN для полнотекстового поиска

| Критерий | GiST | GIN |
|----------|------|-----|
| Скорость поиска | Медленнее | Быстрее |
| Скорость обновления | Быстрее | Медленнее |
| Размер индекса | Меньше | Больше |
| Когда использовать | Частые обновления | Редкие обновления, частые поиски |

```sql
-- GiST — для часто обновляемых данных
CREATE INDEX idx_articles_gist ON articles USING gist (search_vector);

-- GIN — для статичных данных с частым поиском
CREATE INDEX idx_articles_gin ON articles USING gin (search_vector);
```

---

## KNN-поиск (K-Nearest Neighbors)

### Оператор расстояния <->

GiST поддерживает эффективный поиск ближайших соседей:

```sql
-- Таблица с координатами
CREATE TABLE locations (
    id SERIAL PRIMARY KEY,
    name TEXT,
    coords point
);

CREATE INDEX idx_locations_coords ON locations USING gist (coords);

-- Вставка данных
INSERT INTO locations (name, coords) VALUES
('Точка A', point(10, 20)),
('Точка B', point(15, 25)),
('Точка C', point(100, 200)),
('Точка D', point(12, 22));

-- Найти 3 ближайших к точке (11, 21)
SELECT name, coords, coords <-> point(11, 21) as distance
FROM locations
ORDER BY coords <-> point(11, 21)
LIMIT 3;

-- Результат:
-- name    | coords  | distance
-- Точка A | (10,20) | 1.414...
-- Точка D | (12,22) | 1.414...
-- Точка B | (15,25) | 5.656...
```

### KNN с PostGIS

```sql
-- Найти 5 ближайших ресторанов
SELECT
    name,
    ST_Distance(location, ST_MakePoint(37.6, 55.75)::geography) as distance_meters
FROM restaurants
ORDER BY location <-> ST_MakePoint(37.6, 55.75)::geography
LIMIT 5;
```

---

## Иерархические данные (ltree)

### Расширение ltree

```sql
-- Установка расширения
CREATE EXTENSION ltree;

-- Таблица категорий
CREATE TABLE categories (
    id SERIAL PRIMARY KEY,
    name TEXT,
    path ltree
);

-- GiST индекс для ltree
CREATE INDEX idx_categories_path ON categories USING gist (path);

-- Вставка иерархии
INSERT INTO categories (name, path) VALUES
('Электроника', 'electronics'),
('Телефоны', 'electronics.phones'),
('Смартфоны', 'electronics.phones.smartphones'),
('iPhone', 'electronics.phones.smartphones.iphone'),
('Android', 'electronics.phones.smartphones.android'),
('Компьютеры', 'electronics.computers'),
('Ноутбуки', 'electronics.computers.laptops');
```

### Операторы ltree

```sql
-- Найти все подкатегории электроники
SELECT * FROM categories
WHERE path <@ 'electronics';

-- Найти родительские категории для iPhone
SELECT * FROM categories
WHERE path @> 'electronics.phones.smartphones.iphone';

-- Проверка по паттерну
SELECT * FROM categories
WHERE path ~ 'electronics.*.smartphones';

-- Получить уровень вложенности
SELECT name, nlevel(path) as depth FROM categories;

-- Операторы:
-- @>   предок (содержит)
-- <@   потомок (содержится в)
-- ~    соответствует lquery паттерну
-- @    соответствует ltxtquery
```

---

## Когда использовать GiST

### Идеальные сценарии

1. **Геопространственные данные**
```sql
-- Поиск объектов в радиусе
-- Поиск пересечений полигонов
-- KNN-поиск ближайших объектов
```

2. **Диапазоны с проверкой пересечений**
```sql
-- Бронирования
-- Расписания
-- Временные интервалы
```

3. **Иерархические структуры**
```sql
-- Деревья категорий
-- Организационные структуры
```

4. **Часто обновляемые полнотекстовые данные**
```sql
-- Логи с поиском
-- Комментарии
```

### Когда НЕ использовать

- Простые B-Tree операции (=, <, >)
- Точные совпадения (Hash или B-Tree)
- Редко обновляемый полнотекстовый поиск (лучше GIN)
- Массивы и JSONB (лучше GIN)

---

## Практические примеры

### Пример 1: Система бронирования переговорок

```sql
CREATE TABLE meeting_rooms (
    id SERIAL PRIMARY KEY,
    name TEXT
);

CREATE TABLE bookings (
    id SERIAL PRIMARY KEY,
    room_id INTEGER REFERENCES meeting_rooms(id),
    time_slot tstzrange NOT NULL,
    booked_by TEXT,
    EXCLUDE USING gist (
        room_id WITH =,
        time_slot WITH &&
    )
);

CREATE INDEX idx_bookings_time ON bookings USING gist (time_slot);

-- Вставка бронирования
INSERT INTO bookings (room_id, time_slot, booked_by)
VALUES (1, '[2024-01-15 10:00, 2024-01-15 11:00)', 'Иванов');

-- Найти свободные слоты
SELECT * FROM meeting_rooms m
WHERE NOT EXISTS (
    SELECT 1 FROM bookings b
    WHERE b.room_id = m.id
    AND b.time_slot && '[2024-01-15 14:00, 2024-01-15 15:00)'::tstzrange
);
```

### Пример 2: Геолокационный сервис

```sql
CREATE EXTENSION IF NOT EXISTS postgis;

CREATE TABLE delivery_zones (
    id SERIAL PRIMARY KEY,
    name TEXT,
    zone geometry(POLYGON, 4326),
    delivery_cost NUMERIC
);

CREATE INDEX idx_zones_geom ON delivery_zones USING gist (zone);

-- Определить зону доставки по адресу
SELECT name, delivery_cost
FROM delivery_zones
WHERE ST_Contains(zone, ST_MakePoint(37.618, 55.751)::geometry);

-- Найти все зоны в радиусе 10 км
SELECT name
FROM delivery_zones
WHERE ST_DWithin(
    zone::geography,
    ST_MakePoint(37.618, 55.751)::geography,
    10000  -- метры
);
```

### Пример 3: IP-диапазоны

```sql
-- Для IP-диапазонов используем int8range
CREATE TABLE ip_blocks (
    id SERIAL PRIMARY KEY,
    ip_range int8range,
    country TEXT,
    isp TEXT
);

CREATE INDEX idx_ip_range ON ip_blocks USING gist (ip_range);

-- Преобразование IP в число
CREATE FUNCTION ip_to_bigint(ip inet) RETURNS bigint AS $$
    SELECT (split_part(host(ip), '.', 1)::bigint << 24) +
           (split_part(host(ip), '.', 2)::bigint << 16) +
           (split_part(host(ip), '.', 3)::bigint << 8) +
           split_part(host(ip), '.', 4)::bigint;
$$ LANGUAGE sql IMMUTABLE;

-- Поиск информации по IP
SELECT country, isp FROM ip_blocks
WHERE ip_range @> ip_to_bigint('192.168.1.100'::inet);
```

---

## Обслуживание GiST индексов

### Перестроение

```sql
-- Перестройка индекса
REINDEX INDEX idx_locations_coords;

-- Без блокировки (PostgreSQL 12+)
REINDEX INDEX CONCURRENTLY idx_locations_coords;
```

### Настройка

```sql
-- Параметр fillfactor (по умолчанию 90)
CREATE INDEX idx_example ON table USING gist (column)
WITH (fillfactor = 70);  -- Больше места для обновлений

-- buffering (для больших таблиц)
CREATE INDEX idx_big ON big_table USING gist (geom)
WITH (buffering = auto);  -- auto, on, off
```

### Мониторинг

```sql
-- Статистика использования
SELECT
    indexrelname,
    idx_scan,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE indexrelname LIKE '%gist%';
```

---

## Best Practices

1. **Выбирайте GiST для специфичных типов данных**
   - Геометрия, диапазоны, ltree, полнотекстовый поиск (с частыми обновлениями)

2. **Используйте Exclusion Constraints**
   - Для предотвращения пересечения диапазонов на уровне БД

3. **Сравнивайте с GIN для полнотекстового поиска**
   - GiST быстрее обновляется, GIN быстрее ищет

4. **Оптимизируйте KNN-запросы**
   - Используйте `ORDER BY ... <->` с `LIMIT` для эффективного поиска

5. **Учитывайте размер индекса**
   - GiST индексы могут быть большими для сложных геометрий

---

## Типичные ошибки

1. **Использование GiST для простых B-Tree операций**
```sql
-- Плохо: для = лучше B-Tree или Hash
CREATE INDEX idx ON table USING gist (simple_column);
```

2. **Игнорирование ANALYZE после загрузки данных**
```sql
-- Статистика важна для планировщика
ANALYZE table_with_gist_index;
```

3. **Неправильный выбор между GiST и GIN для FTS**
```sql
-- Для редко обновляемых данных GIN эффективнее
```

4. **Отсутствие индекса для exclusion constraint**
```sql
-- Constraint неявно создаёт индекс, но проверьте его наличие
```
