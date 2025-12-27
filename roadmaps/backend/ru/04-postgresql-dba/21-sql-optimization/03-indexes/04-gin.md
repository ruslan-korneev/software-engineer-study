# GIN индексы

## Введение

GIN (Generalized Inverted Index) — это **инвертированный индекс**, оптимизированный для поиска в составных значениях. GIN идеально подходит для индексирования данных, где одно значение содержит множество элементов: массивы, JSONB, полнотекстовые документы.

---

## Принцип работы

### Инвертированный индекс

GIN хранит отображение "элемент → список строк":

```
Обычная таблица:
| id | tags                    |
|----|-------------------------|
| 1  | {python, web, api}      |
| 2  | {python, ml, data}      |
| 3  | {javascript, web, react}|

GIN индекс (инвертированный):
python     → [1, 2]
web        → [1, 3]
api        → [1]
ml         → [2]
data       → [2]
javascript → [3]
react      → [3]
```

### Характеристики

- **Поиск элемента**: O(log n) для нахождения + чтение списка
- **Вставка**: Дороже чем B-Tree (нужно обновить несколько записей)
- **Размер**: Больше чем B-Tree из-за множественных ссылок
- **Идеален для**: статичных данных с частым поиском

---

## Массивы (Arrays)

### Создание индекса на массив

```sql
-- Таблица с тегами
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT,
    tags TEXT[]
);

-- GIN индекс на массив
CREATE INDEX idx_articles_tags ON articles USING gin (tags);

-- Вставка данных
INSERT INTO articles (title, tags) VALUES
('Введение в Python', ARRAY['python', 'programming', 'tutorial']),
('Web разработка на Django', ARRAY['python', 'web', 'django']),
('React для начинающих', ARRAY['javascript', 'web', 'react']);
```

### Операторы для массивов

```sql
-- Содержит элемент (@>)
SELECT * FROM articles WHERE tags @> ARRAY['python'];
-- Результат: статьи с тегом 'python'

-- Содержится в (<@)
SELECT * FROM articles WHERE tags <@ ARRAY['python', 'web', 'django', 'tutorial'];
-- Результат: статьи, все теги которых есть в списке

-- Пересекается (&&)
SELECT * FROM articles WHERE tags && ARRAY['web', 'mobile'];
-- Результат: статьи с тегом 'web' ИЛИ 'mobile'

-- Операторы для GIN массивов:
-- @>   содержит все элементы
-- <@   содержится в
-- &&   имеет общие элементы
-- =    равенство массивов
```

### Практический пример: система тегов

```sql
-- Поиск статей с несколькими тегами (AND)
SELECT * FROM articles
WHERE tags @> ARRAY['python', 'web'];

-- Поиск статей с любым из тегов (OR)
SELECT * FROM articles
WHERE tags && ARRAY['python', 'javascript'];

-- Подсчёт статей по тегам
SELECT tag, COUNT(*) as count
FROM articles, unnest(tags) as tag
GROUP BY tag
ORDER BY count DESC;
```

---

## JSONB

### Индексирование JSONB

GIN — основной способ индексирования JSONB:

```sql
-- Таблица с JSON-данными
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    data JSONB
);

-- GIN индекс на весь JSONB
CREATE INDEX idx_products_data ON products USING gin (data);

-- Вставка данных
INSERT INTO products (name, data) VALUES
('iPhone 15', '{"brand": "Apple", "storage": 128, "colors": ["black", "white"], "5g": true}'),
('Galaxy S24', '{"brand": "Samsung", "storage": 256, "colors": ["green", "black"], "5g": true}'),
('Pixel 8', '{"brand": "Google", "storage": 128, "colors": ["white", "blue"], "5g": true}');
```

### Операторы для JSONB

```sql
-- Содержит (@>)
SELECT * FROM products WHERE data @> '{"brand": "Apple"}';
SELECT * FROM products WHERE data @> '{"storage": 128}';
SELECT * FROM products WHERE data @> '{"5g": true, "brand": "Google"}';

-- Содержится (<@)
SELECT * FROM products
WHERE data <@ '{"brand": "Apple", "storage": 128, "colors": ["black", "white"], "5g": true, "extra": "value"}';

-- Существует ключ (?)
SELECT * FROM products WHERE data ? 'storage';

-- Существует любой ключ (?|)
SELECT * FROM products WHERE data ?| ARRAY['wireless_charging', '5g'];

-- Существуют все ключи (?&)
SELECT * FROM products WHERE data ?& ARRAY['brand', 'storage'];
```

### Типы GIN операторов для JSONB

```sql
-- jsonb_ops (по умолчанию) - поддерживает: @>, ?, ?|, ?&
CREATE INDEX idx_data ON products USING gin (data);

-- jsonb_path_ops - только @>, но меньше и быстрее
CREATE INDEX idx_data_path ON products USING gin (data jsonb_path_ops);
```

**Сравнение операторных классов:**

| Оператор | jsonb_ops | jsonb_path_ops |
|----------|-----------|----------------|
| `@>` | Да | Да |
| `?` | Да | Нет |
| `?|` | Да | Нет |
| `?&` | Да | Нет |
| Размер индекса | Больше | Меньше |
| Скорость `@>` | Хорошая | Лучше |

```sql
-- Если нужен только @>, используйте jsonb_path_ops
CREATE INDEX idx_products_data ON products USING gin (data jsonb_path_ops);

-- Если нужны проверки ключей (?), используйте jsonb_ops
CREATE INDEX idx_products_data ON products USING gin (data);
```

---

## Полнотекстовый поиск

### tsvector и GIN

```sql
-- Таблица документов
CREATE TABLE documents (
    id SERIAL PRIMARY KEY,
    title TEXT,
    content TEXT,
    search_vector tsvector
);

-- GIN индекс для полнотекстового поиска
CREATE INDEX idx_documents_search ON documents USING gin (search_vector);

-- Триггер для автоматического обновления
CREATE FUNCTION update_search_vector() RETURNS trigger AS $$
BEGIN
    NEW.search_vector :=
        setweight(to_tsvector('russian', COALESCE(NEW.title, '')), 'A') ||
        setweight(to_tsvector('russian', COALESCE(NEW.content, '')), 'B');
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER documents_search_trigger
BEFORE INSERT OR UPDATE ON documents
FOR EACH ROW EXECUTE FUNCTION update_search_vector();

-- Вставка данных
INSERT INTO documents (title, content) VALUES
('PostgreSQL индексы', 'Подробное руководство по использованию индексов в PostgreSQL'),
('Оптимизация запросов', 'Как ускорить работу базы данных PostgreSQL'),
('Python и базы данных', 'Работа с PostgreSQL из Python приложений');
```

### Полнотекстовые запросы

```sql
-- Простой поиск
SELECT title, ts_rank(search_vector, query) as rank
FROM documents, to_tsquery('russian', 'PostgreSQL') query
WHERE search_vector @@ query
ORDER BY rank DESC;

-- Поиск с операторами
-- & = AND
SELECT * FROM documents
WHERE search_vector @@ to_tsquery('russian', 'PostgreSQL & индексы');

-- | = OR
SELECT * FROM documents
WHERE search_vector @@ to_tsquery('russian', 'Python | индексы');

-- ! = NOT
SELECT * FROM documents
WHERE search_vector @@ to_tsquery('russian', 'PostgreSQL & !Python');

-- Поиск фразы
SELECT * FROM documents
WHERE search_vector @@ phraseto_tsquery('russian', 'базы данных');

-- Префиксный поиск
SELECT * FROM documents
WHERE search_vector @@ to_tsquery('russian', 'оптимиз:*');
```

### GIN vs GiST для полнотекстового поиска

```sql
-- GIN - лучше для поиска (статичные данные)
CREATE INDEX idx_fts_gin ON documents USING gin (search_vector);

-- GiST - лучше для обновления (динамичные данные)
CREATE INDEX idx_fts_gist ON documents USING gist (search_vector);
```

| Критерий | GIN | GiST |
|----------|-----|------|
| Скорость поиска | Быстрее (в 3 раза) | Медленнее |
| Скорость вставки | Медленнее | Быстрее |
| Размер индекса | Больше | Меньше |
| Рекомендуется | Статичные данные | Часто обновляемые |

---

## Trigram (pg_trgm)

### Поиск по подстроке и нечёткий поиск

```sql
-- Установка расширения
CREATE EXTENSION pg_trgm;

-- Таблица
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT
);

-- GIN индекс с триграммами
CREATE INDEX idx_products_name_trgm ON products USING gin (name gin_trgm_ops);

-- Вставка данных
INSERT INTO products (name) VALUES
('Apple iPhone 15 Pro'),
('Samsung Galaxy S24'),
('Google Pixel 8 Pro'),
('OnePlus 12');
```

### Операторы pg_trgm

```sql
-- Поиск подстроки (LIKE с GIN!)
SELECT * FROM products WHERE name LIKE '%iPhone%';
SELECT * FROM products WHERE name ILIKE '%galaxy%';

-- Регулярные выражения
SELECT * FROM products WHERE name ~ 'Pro$';

-- Похожесть (similarity)
SELECT name, similarity(name, 'iPhone') as sim
FROM products
WHERE name % 'iPhone'  -- порог по умолчанию 0.3
ORDER BY sim DESC;

-- Изменить порог похожести
SET pg_trgm.similarity_threshold = 0.1;

-- Расстояние (для сортировки)
SELECT name, name <-> 'iPhone' as distance
FROM products
ORDER BY name <-> 'iPhone'
LIMIT 5;
```

### Практический пример: автодополнение

```sql
-- Таблица городов
CREATE TABLE cities (
    id SERIAL PRIMARY KEY,
    name TEXT
);

CREATE INDEX idx_cities_name ON cities USING gin (name gin_trgm_ops);

-- Вставка данных
INSERT INTO cities (name) VALUES
('Москва'), ('Санкт-Петербург'), ('Новосибирск'),
('Екатеринбург'), ('Нижний Новгород'), ('Казань');

-- Автодополнение по вводу пользователя
SELECT name FROM cities
WHERE name ILIKE '%нов%'
ORDER BY name <-> 'нов'
LIMIT 10;

-- Результат: Новосибирск, Нижний Новгород (оба содержат 'нов')
```

### Исправление опечаток

```sql
-- Найти товары, даже если пользователь опечатался
SELECT name, similarity(name, 'Samsang') as sim
FROM products
WHERE name % 'Samsang'
ORDER BY sim DESC;

-- Результат: Samsung Galaxy S24 (найден несмотря на опечатку)
```

---

## Настройка производительности GIN

### Pending list

GIN использует "pending list" для ускорения вставки:

```sql
-- Настройка размера pending list
CREATE INDEX idx_products_tags ON products USING gin (tags)
WITH (fastupdate = on, gin_pending_list_limit = 4096);  -- 4MB по умолчанию

-- Принудительное слияние pending list
SELECT gin_clean_pending_list('idx_products_tags');

-- Отключение fastupdate для максимальной скорости поиска
CREATE INDEX idx_static_data ON static_table USING gin (data)
WITH (fastupdate = off);
```

### Когда включать/выключать fastupdate

| Сценарий | fastupdate |
|----------|------------|
| Много INSERT/UPDATE | on (по умолчанию) |
| Статичные данные | off |
| Важна скорость поиска | off |
| Важна скорость вставки | on |

---

## Практические примеры

### Пример 1: E-commerce фильтры

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name TEXT,
    category TEXT,
    attributes JSONB,
    tags TEXT[]
);

-- Индексы
CREATE INDEX idx_products_attrs ON products USING gin (attributes);
CREATE INDEX idx_products_tags ON products USING gin (tags);

-- Вставка данных
INSERT INTO products (name, category, attributes, tags) VALUES
('iPhone 15', 'phones', '{"brand": "Apple", "storage": 128, "color": "black"}', ARRAY['5g', 'wireless-charging']),
('Galaxy S24', 'phones', '{"brand": "Samsung", "storage": 256, "color": "green"}', ARRAY['5g', 'stylus']);

-- Сложный фильтр: бренд + тег + атрибут
SELECT * FROM products
WHERE attributes @> '{"brand": "Apple"}'
  AND tags && ARRAY['5g']
  AND (attributes->>'storage')::int >= 128;
```

### Пример 2: Поисковый движок

```sql
CREATE TABLE pages (
    id SERIAL PRIMARY KEY,
    url TEXT,
    title TEXT,
    content TEXT,
    search_vector tsvector,
    last_indexed TIMESTAMPTZ
);

CREATE INDEX idx_pages_search ON pages USING gin (search_vector);
CREATE INDEX idx_pages_url ON pages USING gin (url gin_trgm_ops);

-- Комбинированный поиск
SELECT
    url,
    title,
    ts_rank(search_vector, query) as rank
FROM pages, websearch_to_tsquery('russian', 'PostgreSQL оптимизация') query
WHERE search_vector @@ query
ORDER BY rank DESC
LIMIT 20;
```

### Пример 3: Система разрешений (permissions)

```sql
CREATE TABLE resources (
    id SERIAL PRIMARY KEY,
    name TEXT,
    permissions JSONB  -- {"read": ["user1", "user2"], "write": ["admin"]}
);

CREATE INDEX idx_resources_perms ON resources USING gin (permissions);

-- Найти ресурсы, где user1 имеет право на чтение
SELECT * FROM resources
WHERE permissions->'read' ? 'user1';

-- Найти ресурсы с любыми правами для user1
SELECT * FROM resources
WHERE permissions @> '{"read": ["user1"]}'
   OR permissions @> '{"write": ["user1"]}';
```

---

## Когда использовать GIN

### Идеально подходит для:
- Полнотекстовый поиск (tsvector)
- JSONB с запросами @>, ?, ?|
- Массивы с операторами @>, &&
- Поиск подстроки (LIKE '%text%' с pg_trgm)
- Нечёткий поиск и автодополнение

### Не подходит для:
- Простые операции сравнения (=, <, >) — используйте B-Tree
- Диапазонные запросы — используйте B-Tree или GiST
- Часто обновляемые данные — рассмотрите GiST для FTS
- Геометрические данные — используйте GiST

---

## Обслуживание GIN индексов

### Мониторинг

```sql
-- Статистика использования
SELECT
    indexrelname,
    idx_scan,
    idx_tup_read,
    pg_size_pretty(pg_relation_size(indexrelid)) as size
FROM pg_stat_user_indexes
WHERE indexrelname LIKE '%gin%';

-- Размер pending list
SELECT * FROM pg_stat_user_tables WHERE relname = 'your_table';
```

### Обслуживание

```sql
-- Очистка pending list
SELECT gin_clean_pending_list('idx_name');

-- Перестроение индекса
REINDEX INDEX idx_products_tags;
REINDEX INDEX CONCURRENTLY idx_products_tags;  -- PostgreSQL 12+

-- Анализ после массовой вставки
ANALYZE products;
```

---

## Best Practices

1. **Выбирайте правильный operator class для JSONB**
   - `jsonb_ops` — если нужны операторы `?`, `?|`, `?&`
   - `jsonb_path_ops` — если только `@>` (меньше, быстрее)

2. **Используйте fastupdate = off для статичных данных**
```sql
CREATE INDEX idx ON table USING gin (column) WITH (fastupdate = off);
```

3. **Комбинируйте с B-Tree для сложных фильтров**
```sql
-- GIN для массивов
CREATE INDEX idx_tags ON products USING gin (tags);
-- B-Tree для категории
CREATE INDEX idx_category ON products (category);

-- Эффективный запрос с обоими индексами
SELECT * FROM products
WHERE category = 'phones' AND tags @> ARRAY['5g'];
```

4. **Для полнотекстового поиска используйте отдельную колонку tsvector**
```sql
-- Лучше создать отдельную колонку и обновлять триггером
ALTER TABLE articles ADD COLUMN search_vector tsvector;
CREATE INDEX idx_search ON articles USING gin (search_vector);
```

5. **Регулярно выполняйте ANALYZE**
```sql
ANALYZE table_with_gin_index;
```

---

## Типичные ошибки

1. **Использование GIN для простых операций**
```sql
-- Плохо: для = лучше B-Tree
CREATE INDEX idx ON users USING gin (ARRAY[email]);
-- Хорошо:
CREATE INDEX idx ON users (email);
```

2. **Забывают про jsonb_path_ops**
```sql
-- Если не нужны ?, ?|, ?&, используйте path_ops
CREATE INDEX idx ON products USING gin (data jsonb_path_ops);
```

3. **Игнорирование pending list при массовой загрузке**
```sql
-- После bulk insert
SELECT gin_clean_pending_list('idx_name');
ANALYZE table_name;
```

4. **Ожидание скорости B-Tree для точных совпадений**
```sql
-- GIN не эффективен для равенства простых значений
-- Для data->>'key' = 'value' лучше создать B-Tree индекс
CREATE INDEX idx ON products ((data->>'category'));
```
