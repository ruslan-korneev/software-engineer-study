# Определение схемы базы данных

[prev: ./02-connection.md](./02-connection.md) | [next: ./04-queries.md](./04-queries.md)

---

## Создание схемы базы данных с asyncpg

asyncpg позволяет выполнять любые DDL (Data Definition Language) команды для создания и модификации структуры базы данных.

## Создание таблиц

### Базовое создание таблицы

```python
import asyncio
import asyncpg


async def create_basic_table():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Создание простой таблицы
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                username VARCHAR(50) NOT NULL UNIQUE,
                email VARCHAR(100) NOT NULL UNIQUE,
                password_hash VARCHAR(255) NOT NULL,
                is_active BOOLEAN DEFAULT TRUE,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            )
        ''')
        print("Таблица 'users' создана успешно")

    finally:
        await conn.close()


asyncio.run(create_basic_table())
```

### Создание связанных таблиц

```python
import asyncio
import asyncpg


async def create_related_tables():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Таблица категорий
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS categories (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100) NOT NULL UNIQUE,
                description TEXT,
                parent_id INTEGER REFERENCES categories(id) ON DELETE SET NULL,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        # Таблица продуктов с внешним ключом
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS products (
                id SERIAL PRIMARY KEY,
                name VARCHAR(200) NOT NULL,
                description TEXT,
                price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
                quantity INTEGER NOT NULL DEFAULT 0 CHECK (quantity >= 0),
                category_id INTEGER NOT NULL REFERENCES categories(id) ON DELETE RESTRICT,
                is_available BOOLEAN DEFAULT TRUE,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        # Таблица заказов
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS orders (
                id SERIAL PRIMARY KEY,
                user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
                status VARCHAR(20) NOT NULL DEFAULT 'pending'
                    CHECK (status IN ('pending', 'confirmed', 'shipped', 'delivered', 'cancelled')),
                total_amount DECIMAL(12, 2) NOT NULL DEFAULT 0,
                shipping_address TEXT,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
                updated_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        # Промежуточная таблица для связи многие-ко-многим
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS order_items (
                id SERIAL PRIMARY KEY,
                order_id INTEGER NOT NULL REFERENCES orders(id) ON DELETE CASCADE,
                product_id INTEGER NOT NULL REFERENCES products(id) ON DELETE RESTRICT,
                quantity INTEGER NOT NULL CHECK (quantity > 0),
                unit_price DECIMAL(10, 2) NOT NULL,
                UNIQUE (order_id, product_id)
            )
        ''')

        print("Все связанные таблицы созданы успешно")

    finally:
        await conn.close()


asyncio.run(create_related_tables())
```

## Создание индексов

```python
import asyncio
import asyncpg


async def create_indexes():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Простой индекс
        await conn.execute('''
            CREATE INDEX IF NOT EXISTS idx_users_email
            ON users(email)
        ''')

        # Уникальный индекс
        await conn.execute('''
            CREATE UNIQUE INDEX IF NOT EXISTS idx_users_username_lower
            ON users(LOWER(username))
        ''')

        # Составной индекс
        await conn.execute('''
            CREATE INDEX IF NOT EXISTS idx_products_category_price
            ON products(category_id, price)
        ''')

        # Частичный индекс (только для активных продуктов)
        await conn.execute('''
            CREATE INDEX IF NOT EXISTS idx_products_available
            ON products(name, price)
            WHERE is_available = TRUE
        ''')

        # GIN индекс для полнотекстового поиска
        await conn.execute('''
            CREATE INDEX IF NOT EXISTS idx_products_name_search
            ON products USING GIN (to_tsvector('russian', name))
        ''')

        # BRIN индекс для временных данных
        await conn.execute('''
            CREATE INDEX IF NOT EXISTS idx_orders_created_at
            ON orders USING BRIN (created_at)
        ''')

        print("Все индексы созданы успешно")

    finally:
        await conn.close()


asyncio.run(create_indexes())
```

## Создание ограничений (Constraints)

```python
import asyncio
import asyncpg


async def create_constraints():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Добавление CHECK constraint
        await conn.execute('''
            ALTER TABLE users
            ADD CONSTRAINT check_email_format
            CHECK (email ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$')
        ''')

        # Добавление UNIQUE constraint на несколько колонок
        await conn.execute('''
            ALTER TABLE order_items
            ADD CONSTRAINT unique_order_product
            UNIQUE (order_id, product_id)
        ''')

        # Добавление FOREIGN KEY с именем
        await conn.execute('''
            ALTER TABLE products
            ADD CONSTRAINT fk_products_category
            FOREIGN KEY (category_id) REFERENCES categories(id)
            ON DELETE RESTRICT ON UPDATE CASCADE
        ''')

        # Добавление EXCLUSION constraint (для диапазонов)
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS reservations (
                id SERIAL PRIMARY KEY,
                room_id INTEGER NOT NULL,
                during TSTZRANGE NOT NULL,
                EXCLUDE USING GIST (room_id WITH =, during WITH &&)
            )
        ''')

        print("Все ограничения созданы успешно")

    finally:
        await conn.close()


asyncio.run(create_constraints())
```

## Создание триггеров

```python
import asyncio
import asyncpg


async def create_triggers():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Функция для автоматического обновления updated_at
        await conn.execute('''
            CREATE OR REPLACE FUNCTION update_updated_at_column()
            RETURNS TRIGGER AS $$
            BEGIN
                NEW.updated_at = CURRENT_TIMESTAMP;
                RETURN NEW;
            END;
            $$ language 'plpgsql'
        ''')

        # Триггер для таблицы users
        await conn.execute('''
            DROP TRIGGER IF EXISTS update_users_updated_at ON users;
            CREATE TRIGGER update_users_updated_at
                BEFORE UPDATE ON users
                FOR EACH ROW
                EXECUTE FUNCTION update_updated_at_column()
        ''')

        # Триггер для таблицы products
        await conn.execute('''
            DROP TRIGGER IF EXISTS update_products_updated_at ON products;
            CREATE TRIGGER update_products_updated_at
                BEFORE UPDATE ON products
                FOR EACH ROW
                EXECUTE FUNCTION update_updated_at_column()
        ''')

        # Функция для логирования изменений
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS audit_log (
                id SERIAL PRIMARY KEY,
                table_name VARCHAR(50) NOT NULL,
                operation VARCHAR(10) NOT NULL,
                old_data JSONB,
                new_data JSONB,
                changed_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP,
                changed_by VARCHAR(100)
            )
        ''')

        await conn.execute('''
            CREATE OR REPLACE FUNCTION audit_trigger_function()
            RETURNS TRIGGER AS $$
            BEGIN
                IF TG_OP = 'INSERT' THEN
                    INSERT INTO audit_log (table_name, operation, new_data)
                    VALUES (TG_TABLE_NAME, 'INSERT', row_to_json(NEW)::jsonb);
                ELSIF TG_OP = 'UPDATE' THEN
                    INSERT INTO audit_log (table_name, operation, old_data, new_data)
                    VALUES (TG_TABLE_NAME, 'UPDATE', row_to_json(OLD)::jsonb, row_to_json(NEW)::jsonb);
                ELSIF TG_OP = 'DELETE' THEN
                    INSERT INTO audit_log (table_name, operation, old_data)
                    VALUES (TG_TABLE_NAME, 'DELETE', row_to_json(OLD)::jsonb);
                END IF;
                RETURN NEW;
            END;
            $$ language 'plpgsql'
        ''')

        await conn.execute('''
            DROP TRIGGER IF EXISTS audit_users ON users;
            CREATE TRIGGER audit_users
                AFTER INSERT OR UPDATE OR DELETE ON users
                FOR EACH ROW
                EXECUTE FUNCTION audit_trigger_function()
        ''')

        print("Все триггеры созданы успешно")

    finally:
        await conn.close()


asyncio.run(create_triggers())
```

## Создание представлений (Views)

```python
import asyncio
import asyncpg


async def create_views():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Простое представление
        await conn.execute('''
            CREATE OR REPLACE VIEW active_users AS
            SELECT id, username, email, created_at
            FROM users
            WHERE is_active = TRUE
        ''')

        # Представление с агрегацией
        await conn.execute('''
            CREATE OR REPLACE VIEW order_summary AS
            SELECT
                o.id AS order_id,
                u.username,
                u.email,
                o.status,
                o.total_amount,
                COUNT(oi.id) AS items_count,
                o.created_at
            FROM orders o
            JOIN users u ON o.user_id = u.id
            LEFT JOIN order_items oi ON o.id = oi.order_id
            GROUP BY o.id, u.username, u.email
        ''')

        # Материализованное представление
        await conn.execute('''
            CREATE MATERIALIZED VIEW IF NOT EXISTS product_stats AS
            SELECT
                c.name AS category_name,
                COUNT(p.id) AS products_count,
                AVG(p.price) AS avg_price,
                MIN(p.price) AS min_price,
                MAX(p.price) AS max_price,
                SUM(p.quantity) AS total_quantity
            FROM categories c
            LEFT JOIN products p ON c.id = p.category_id
            GROUP BY c.id, c.name
            WITH DATA
        ''')

        # Индекс на материализованном представлении
        await conn.execute('''
            CREATE UNIQUE INDEX IF NOT EXISTS idx_product_stats_category
            ON product_stats(category_name)
        ''')

        print("Все представления созданы успешно")

    finally:
        await conn.close()


async def refresh_materialized_view():
    """Обновление материализованного представления."""
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Обновление без блокировки (CONCURRENTLY требует уникальный индекс)
        await conn.execute('REFRESH MATERIALIZED VIEW CONCURRENTLY product_stats')
        print("Материализованное представление обновлено")

    finally:
        await conn.close()


asyncio.run(create_views())
```

## Создание пользовательских типов

```python
import asyncio
import asyncpg


async def create_custom_types():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        # Создание ENUM типа
        await conn.execute('''
            DO $$ BEGIN
                CREATE TYPE order_status AS ENUM (
                    'pending', 'confirmed', 'processing',
                    'shipped', 'delivered', 'cancelled'
                );
            EXCEPTION
                WHEN duplicate_object THEN null;
            END $$
        ''')

        # Создание композитного типа
        await conn.execute('''
            DO $$ BEGIN
                CREATE TYPE address AS (
                    street VARCHAR(200),
                    city VARCHAR(100),
                    postal_code VARCHAR(20),
                    country VARCHAR(100)
                );
            EXCEPTION
                WHEN duplicate_object THEN null;
            END $$
        ''')

        # Создание DOMAIN типа с ограничениями
        await conn.execute('''
            DO $$ BEGIN
                CREATE DOMAIN email_type AS VARCHAR(100)
                CHECK (VALUE ~* '^[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}$');
            EXCEPTION
                WHEN duplicate_object THEN null;
            END $$
        ''')

        await conn.execute('''
            DO $$ BEGIN
                CREATE DOMAIN positive_integer AS INTEGER
                CHECK (VALUE > 0);
            EXCEPTION
                WHEN duplicate_object THEN null;
            END $$
        ''')

        # Использование пользовательских типов
        await conn.execute('''
            CREATE TABLE IF NOT EXISTS customers (
                id SERIAL PRIMARY KEY,
                name VARCHAR(100) NOT NULL,
                email email_type NOT NULL,
                shipping_address address,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            )
        ''')

        print("Все пользовательские типы созданы успешно")

    finally:
        await conn.close()


asyncio.run(create_custom_types())
```

## Миграции с использованием asyncpg

```python
import asyncio
import asyncpg
from datetime import datetime


class AsyncMigration:
    """Простая система миграций на asyncpg."""

    def __init__(self, conn: asyncpg.Connection):
        self.conn = conn

    async def init_migrations_table(self):
        """Создание таблицы для отслеживания миграций."""
        await self.conn.execute('''
            CREATE TABLE IF NOT EXISTS schema_migrations (
                id SERIAL PRIMARY KEY,
                version VARCHAR(50) NOT NULL UNIQUE,
                description TEXT,
                applied_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            )
        ''')

    async def is_applied(self, version: str) -> bool:
        """Проверка, была ли миграция применена."""
        result = await self.conn.fetchval(
            'SELECT 1 FROM schema_migrations WHERE version = $1',
            version
        )
        return result is not None

    async def apply(self, version: str, description: str, sql: str):
        """Применение миграции."""
        if await self.is_applied(version):
            print(f"Миграция {version} уже применена, пропуск...")
            return

        async with self.conn.transaction():
            await self.conn.execute(sql)
            await self.conn.execute(
                'INSERT INTO schema_migrations (version, description) VALUES ($1, $2)',
                version, description
            )

        print(f"Миграция {version} применена: {description}")


# Определение миграций
MIGRATIONS = [
    {
        'version': '001',
        'description': 'Create users table',
        'sql': '''
            CREATE TABLE IF NOT EXISTS users (
                id SERIAL PRIMARY KEY,
                username VARCHAR(50) NOT NULL UNIQUE,
                email VARCHAR(100) NOT NULL UNIQUE,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            )
        '''
    },
    {
        'version': '002',
        'description': 'Add password_hash to users',
        'sql': '''
            ALTER TABLE users
            ADD COLUMN IF NOT EXISTS password_hash VARCHAR(255)
        '''
    },
    {
        'version': '003',
        'description': 'Create posts table',
        'sql': '''
            CREATE TABLE IF NOT EXISTS posts (
                id SERIAL PRIMARY KEY,
                user_id INTEGER NOT NULL REFERENCES users(id) ON DELETE CASCADE,
                title VARCHAR(200) NOT NULL,
                content TEXT,
                created_at TIMESTAMP WITH TIME ZONE DEFAULT CURRENT_TIMESTAMP
            )
        '''
    },
]


async def run_migrations():
    conn = await asyncpg.connect('postgresql://postgres:password@localhost/mydb')

    try:
        migration = AsyncMigration(conn)
        await migration.init_migrations_table()

        for m in MIGRATIONS:
            await migration.apply(m['version'], m['description'], m['sql'])

        print("\nВсе миграции применены успешно!")

    finally:
        await conn.close()


asyncio.run(run_migrations())
```

## Лучшие практики

1. **Используйте `IF NOT EXISTS`** для идемпотентности DDL команд
2. **Создавайте индексы CONCURRENTLY** для production баз данных
3. **Используйте транзакции** для группы связанных DDL операций
4. **Документируйте схему** с помощью COMMENT
5. **Применяйте миграции** последовательно и версионируйте их
6. **Тестируйте миграции** на копии базы данных перед применением в production

```python
# Добавление комментариев к объектам схемы
await conn.execute('''
    COMMENT ON TABLE users IS 'Таблица пользователей системы';
    COMMENT ON COLUMN users.email IS 'Электронная почта пользователя (уникальная)';
    COMMENT ON INDEX idx_users_email IS 'Индекс для быстрого поиска по email';
''')
```

---

[prev: ./02-connection.md](./02-connection.md) | [next: ./04-queries.md](./04-queries.md)
