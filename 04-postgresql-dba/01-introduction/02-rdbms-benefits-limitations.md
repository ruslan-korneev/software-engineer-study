# Преимущества и ограничения RDBMS

## Преимущества реляционных баз данных

### 1. Целостность данных (Data Integrity)

RDBMS обеспечивает строгую целостность данных через ограничения (constraints):

```sql
CREATE TABLE products (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100) NOT NULL,                    -- NOT NULL constraint
    price DECIMAL(10, 2) CHECK (price > 0),        -- CHECK constraint
    sku VARCHAR(50) UNIQUE,                        -- UNIQUE constraint
    category_id INTEGER REFERENCES categories(id)  -- FOREIGN KEY constraint
);
```

**Типы ограничений:**
- `NOT NULL` — значение обязательно
- `UNIQUE` — значение уникально в столбце
- `PRIMARY KEY` — уникальный идентификатор строки
- `FOREIGN KEY` — ссылочная целостность между таблицами
- `CHECK` — проверка условия
- `DEFAULT` — значение по умолчанию

### 2. ACID-транзакции

Гарантия надёжности данных даже при сбоях системы:

```sql
-- Перевод денег между счетами — классический пример транзакции
BEGIN;

-- Списание с одного счёта
UPDATE accounts
SET balance = balance - 1000
WHERE account_number = 'ACC001';

-- Зачисление на другой счёт
UPDATE accounts
SET balance = balance + 1000
WHERE account_number = 'ACC002';

-- Если всё успешно — фиксируем
COMMIT;

-- При ошибке — откатываем все изменения
-- ROLLBACK;
```

### 3. Мощный язык запросов (SQL)

SQL позволяет выполнять сложные выборки, агрегации и аналитику:

```sql
-- Сложный аналитический запрос
SELECT
    c.name AS category,
    COUNT(p.id) AS product_count,
    AVG(p.price) AS avg_price,
    SUM(oi.quantity * oi.price) AS total_revenue
FROM categories c
LEFT JOIN products p ON c.id = p.category_id
LEFT JOIN order_items oi ON p.id = oi.product_id
WHERE oi.created_at >= DATE_TRUNC('month', CURRENT_DATE)
GROUP BY c.id, c.name
HAVING COUNT(p.id) > 5
ORDER BY total_revenue DESC
LIMIT 10;
```

### 4. Нормализация и структурированность

Чёткая структура данных упрощает понимание и поддержку:

```sql
-- Нормализованная структура для интернет-магазина
CREATE TABLE customers (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) UNIQUE NOT NULL,
    name VARCHAR(100)
);

CREATE TABLE addresses (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    street VARCHAR(255),
    city VARCHAR(100),
    postal_code VARCHAR(20),
    is_default BOOLEAN DEFAULT false
);

CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    customer_id INTEGER REFERENCES customers(id),
    shipping_address_id INTEGER REFERENCES addresses(id),
    status VARCHAR(50) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### 5. Индексирование и оптимизация

Эффективный поиск данных через индексы:

```sql
-- Создание различных типов индексов
CREATE INDEX idx_users_email ON users(email);
CREATE INDEX idx_orders_date ON orders(created_at DESC);
CREATE INDEX idx_products_name_gin ON products USING GIN(to_tsvector('russian', name));

-- Составной индекс
CREATE INDEX idx_orders_customer_status ON orders(customer_id, status);

-- Частичный индекс
CREATE INDEX idx_active_users ON users(email) WHERE is_active = true;
```

### 6. Зрелость и экосистема

- Десятилетия развития и оптимизации
- Богатая документация
- Множество инструментов (ORM, GUI-клиенты, мониторинг)
- Большое сообщество и поддержка
- Стандартизированный SQL (с вариациями)

### 7. Безопасность

```sql
-- Гранулярный контроль доступа
CREATE ROLE readonly_user;
GRANT SELECT ON ALL TABLES IN SCHEMA public TO readonly_user;

CREATE ROLE data_analyst;
GRANT SELECT, INSERT ON reports TO data_analyst;

-- Row-Level Security (PostgreSQL)
ALTER TABLE orders ENABLE ROW LEVEL SECURITY;

CREATE POLICY orders_isolation ON orders
    FOR ALL
    TO application_user
    USING (tenant_id = current_setting('app.tenant_id')::INTEGER);
```

## Ограничения реляционных баз данных

### 1. Вертикальное масштабирование

RDBMS традиционно масштабируются вертикально (увеличение мощности сервера), а не горизонтально:

```
Вертикальное масштабирование:
┌─────────────────┐
│   Один сервер   │
│  CPU: 4 → 32    │
│  RAM: 16 → 256  │
│  Disk: SSD RAID │
└─────────────────┘

Горизонтальное масштабирование (сложно для RDBMS):
┌────────┐ ┌────────┐ ┌────────┐ ┌────────┐
│ Node 1 │ │ Node 2 │ │ Node 3 │ │ Node 4 │
└────────┘ └────────┘ └────────┘ └────────┘
     ↑ Требует шардирования и репликации
```

### 2. Жёсткая схема данных

Изменение структуры может быть сложным и требовать миграций:

```sql
-- Добавление столбца в большую таблицу может занять время
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- Изменение типа данных требует осторожности
ALTER TABLE products ALTER COLUMN price TYPE NUMERIC(12, 4);

-- Удаление столбца с зависимостями
ALTER TABLE orders DROP COLUMN legacy_field CASCADE;
```

### 3. Object-Relational Impedance Mismatch

Несоответствие между объектной моделью приложения и реляционной моделью БД:

```python
# Объект в коде
class Order:
    def __init__(self):
        self.id = None
        self.customer = Customer()  # Вложенный объект
        self.items = []             # Список объектов
        self.shipping_address = Address()

# В реляционной БД это 4 таблицы с JOIN'ами
# orders, customers, order_items, addresses
```

### 4. Производительность на больших объёмах

При очень больших объёмах данных JOIN'ы могут становиться медленными:

```sql
-- Этот запрос может быть медленным на миллиардах записей
SELECT *
FROM orders o
JOIN customers c ON o.customer_id = c.id
JOIN order_items oi ON o.id = oi.order_id
JOIN products p ON oi.product_id = p.id
WHERE o.created_at > '2024-01-01';
```

**Решения:**
- Правильное индексирование
- Партиционирование таблиц
- Денормализация для чтения
- Материализованные представления

```sql
-- Партиционирование по дате
CREATE TABLE orders (
    id SERIAL,
    customer_id INTEGER,
    created_at TIMESTAMP
) PARTITION BY RANGE (created_at);

CREATE TABLE orders_2024_q1 PARTITION OF orders
    FOR VALUES FROM ('2024-01-01') TO ('2024-04-01');

-- Материализованное представление для отчётов
CREATE MATERIALIZED VIEW monthly_sales AS
SELECT
    DATE_TRUNC('month', created_at) AS month,
    SUM(total) AS revenue
FROM orders
GROUP BY DATE_TRUNC('month', created_at);
```

### 5. Неструктурированные данные

RDBMS не оптимальны для хранения неструктурированных данных:

```sql
-- Попытка хранить JSON — работает, но не всегда эффективно
CREATE TABLE events (
    id SERIAL PRIMARY KEY,
    event_type VARCHAR(50),
    payload JSONB  -- PostgreSQL поддерживает JSON
);

-- Запросы к JSON могут быть медленнее
SELECT * FROM events
WHERE payload->>'user_id' = '123';
```

### 6. Сложность распределённых транзакций

Транзакции между несколькими базами данных сложны в реализации:

```
База данных A          База данных B
┌───────────┐          ┌───────────┐
│ UPDATE... │ ←─────── │ UPDATE... │
└───────────┘   2PC    └───────────┘
                ↑
    Two-Phase Commit (медленный, сложный)
```

## Сравнительная таблица

| Аспект | Преимущество | Ограничение |
|--------|-------------|-------------|
| **Целостность** | Строгие ограничения | Требует планирования схемы |
| **Запросы** | Мощный SQL | Сложные JOIN'ы на больших данных |
| **Масштабирование** | Надёжное вертикальное | Сложное горизонтальное |
| **Схема** | Структурированность | Жёсткость, миграции |
| **Транзакции** | ACID-гарантии | Распределённые транзакции сложны |
| **Данные** | Структурированные | Неструктурированные — хуже |

## Когда RDBMS — правильный выбор

**Идеально подходит:**
- Финансовые системы (банкинг, платежи)
- E-commerce (заказы, инвентарь)
- ERP/CRM системы
- Любые системы с важностью целостности данных
- Сложная аналитика и отчётность

**Может не подходить:**
- Хранение логов/событий (миллиарды записей)
- Социальные графы (лучше графовые БД)
- Кэширование (лучше Redis/Memcached)
- Полнотекстовый поиск (лучше Elasticsearch)
- Временные ряды (лучше TimescaleDB/InfluxDB)

## Best Practices

1. **Проектируйте схему заранее** — изменения могут быть дорогими
2. **Используйте правильные типы данных** — VARCHAR(255) не для всего
3. **Создавайте индексы осознанно** — они ускоряют чтение, но замедляют запись
4. **Нормализуйте, но знайте меру** — иногда денормализация оправдана
5. **Используйте транзакции** — не пренебрегайте ACID
6. **Планируйте масштабирование** — read replicas, партиционирование
7. **Мониторьте производительность** — EXPLAIN ANALYZE ваш друг
