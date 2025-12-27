# Миграции

## Что такое миграции и зачем они нужны

**Миграции базы данных** — это способ управления изменениями схемы базы данных с помощью версионируемых скриптов. Каждая миграция представляет собой набор SQL-команд или программный код, который переводит базу данных из одного состояния в другое.

### Проблемы без миграций

```
Без миграций:
- Ручное изменение схемы на каждом сервере
- Отсутствие истории изменений
- Невозможность отката к предыдущей версии
- Рассинхронизация между dev/staging/production
- Сложность работы в команде
```

### Преимущества миграций

1. **Версионирование схемы** — история всех изменений в git
2. **Повторяемость** — одинаковая схема на всех окружениях
3. **Откат** — возможность вернуться к предыдущей версии
4. **Командная работа** — изменения проходят code review
5. **Автоматизация** — миграции запускаются в CI/CD

---

## Принципы управления версиями схемы БД

### 1. Каждое изменение — отдельная миграция

```
migrations/
├── 001_create_users_table.py
├── 002_add_email_to_users.py
├── 003_create_orders_table.py
├── 004_add_user_id_to_orders.py
└── 005_add_status_to_orders.py
```

### 2. Миграции упорядочены

Миграции применяются строго по порядку. Каждая миграция имеет уникальный идентификатор:
- **Числовой префикс**: `001_`, `002_`, ...
- **Timestamp**: `20240115120000_create_users.py`
- **UUID/Hash**: `a1b2c3d4_create_users.py` (Alembic)

### 3. Миграции идемпотентны

Повторное применение миграции не должно вызывать ошибок:

```sql
-- Плохо: вызовет ошибку при повторном запуске
CREATE TABLE users (id INT);

-- Хорошо: идемпотентно
CREATE TABLE IF NOT EXISTS users (id INT);
```

### 4. Миграции неизменяемы

После применения миграции в production её **нельзя изменять**. Новые изменения — только через новые миграции.

### 5. Таблица метаданных

Система миграций хранит информацию о применённых миграциях:

```sql
-- Пример таблицы alembic_version
CREATE TABLE alembic_version (
    version_num VARCHAR(32) NOT NULL
);

-- Пример таблицы Django
CREATE TABLE django_migrations (
    id SERIAL PRIMARY KEY,
    app VARCHAR(255),
    name VARCHAR(255),
    applied TIMESTAMP
);
```

---

## Инструменты миграций

### Alembic (Python/SQLAlchemy)

**Alembic** — стандартный инструмент миграций для SQLAlchemy.

```bash
# Установка
pip install alembic

# Инициализация
alembic init migrations

# Создание миграции
alembic revision -m "create users table"

# Автогенерация миграции (сравнение моделей с БД)
alembic revision --autogenerate -m "add email to users"

# Применение миграций
alembic upgrade head

# Откат последней миграции
alembic downgrade -1

# Просмотр истории
alembic history
```

**Структура проекта с Alembic:**

```
project/
├── alembic.ini          # Конфигурация
├── migrations/
│   ├── env.py           # Настройка окружения
│   ├── script.py.mako   # Шаблон миграции
│   └── versions/        # Файлы миграций
│       ├── 001_create_users.py
│       └── 002_add_email.py
└── models.py            # SQLAlchemy модели
```

### Django Migrations

Django имеет встроенную систему миграций:

```bash
# Создание миграций на основе изменений моделей
python manage.py makemigrations

# Применение миграций
python manage.py migrate

# Просмотр SQL без применения
python manage.py sqlmigrate app_name 0001

# Откат к конкретной миграции
python manage.py migrate app_name 0002

# Просмотр статуса
python manage.py showmigrations
```

### Flyway (Java, SQL)

Flyway использует чистые SQL-файлы:

```bash
# Структура
sql/
├── V1__Create_users_table.sql
├── V2__Add_email_column.sql
└── V3__Create_orders_table.sql

# Применение
flyway migrate

# Информация о состоянии
flyway info

# Откат (только в платной версии)
flyway undo
```

**Формат имени файла:**
- `V` — версионная миграция
- `U` — undo-миграция (откат)
- `R` — repeatable миграция (применяется каждый раз)

### Liquibase (Java, XML/YAML/JSON/SQL)

Liquibase поддерживает разные форматы:

```yaml
# changelog.yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: developer
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: int
                  autoIncrement: true
                  constraints:
                    primaryKey: true
              - column:
                  name: name
                  type: varchar(255)
```

```bash
# Применение
liquibase update

# Откат последнего changeset
liquibase rollbackCount 1

# Генерация changelog из существующей БД
liquibase generateChangeLog
```

### Сравнение инструментов

| Особенность | Alembic | Django | Flyway | Liquibase |
|-------------|---------|--------|--------|-----------|
| Язык | Python | Python | SQL/Java | XML/YAML/SQL |
| Автогенерация | Да | Да | Нет | Частично |
| Откат | Да | Да | Платно | Да |
| Ветвление | Да | Нет | Нет | Да |
| Сложность | Средняя | Низкая | Низкая | Высокая |

---

## Структура миграционного файла

### Alembic миграция

```python
"""create users table

Revision ID: a1b2c3d4e5f6
Revises:
Create Date: 2024-01-15 12:00:00.000000

"""
from alembic import op
import sqlalchemy as sa

# Идентификаторы ревизии
revision = 'a1b2c3d4e5f6'
down_revision = None  # Предыдущая миграция (None = первая)
branch_labels = None
depends_on = None


def upgrade() -> None:
    """Применение миграции (вперёд)."""
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('username', sa.String(50), nullable=False, unique=True),
        sa.Column('email', sa.String(100), nullable=False),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now()),
    )

    # Создание индекса
    op.create_index('ix_users_email', 'users', ['email'])


def downgrade() -> None:
    """Откат миграции (назад)."""
    op.drop_index('ix_users_email', 'users')
    op.drop_table('users')
```

### Django миграция

```python
# 0001_initial.py
from django.db import migrations, models


class Migration(migrations.Migration):

    initial = True

    dependencies = []  # Зависимости от других миграций

    operations = [
        migrations.CreateModel(
            name='User',
            fields=[
                ('id', models.BigAutoField(primary_key=True)),
                ('username', models.CharField(max_length=50, unique=True)),
                ('email', models.EmailField()),
                ('created_at', models.DateTimeField(auto_now_add=True)),
            ],
        ),
        migrations.AddIndex(
            model_name='user',
            index=models.Index(fields=['email'], name='users_email_idx'),
        ),
    ]
```

---

## Типы миграций

### 1. Создание и удаление таблиц

```python
# Alembic: создание таблицы
def upgrade():
    op.create_table(
        'products',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('name', sa.String(200), nullable=False),
        sa.Column('price', sa.Numeric(10, 2), nullable=False),
        sa.Column('category_id', sa.Integer(), sa.ForeignKey('categories.id')),
    )

def downgrade():
    op.drop_table('products')
```

### 2. Изменение колонок

```python
# Добавление колонки
def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20), nullable=True))

def downgrade():
    op.drop_column('users', 'phone')


# Изменение типа колонки
def upgrade():
    op.alter_column(
        'products',
        'price',
        type_=sa.Numeric(12, 4),  # Было (10, 2)
        existing_type=sa.Numeric(10, 2)
    )

def downgrade():
    op.alter_column(
        'products',
        'price',
        type_=sa.Numeric(10, 2),
        existing_type=sa.Numeric(12, 4)
    )


# Переименование колонки
def upgrade():
    op.alter_column('users', 'name', new_column_name='full_name')

def downgrade():
    op.alter_column('users', 'full_name', new_column_name='name')


# Добавление NOT NULL с default
def upgrade():
    # Сначала добавляем с NULL
    op.add_column('users', sa.Column('status', sa.String(20), nullable=True))

    # Заполняем существующие записи
    op.execute("UPDATE users SET status = 'active' WHERE status IS NULL")

    # Делаем NOT NULL
    op.alter_column('users', 'status', nullable=False)

def downgrade():
    op.drop_column('users', 'status')
```

### 3. Работа с индексами и ограничениями

```python
# Индексы
def upgrade():
    op.create_index('ix_users_email', 'users', ['email'], unique=True)
    op.create_index('ix_orders_created', 'orders', ['created_at'])

    # Составной индекс
    op.create_index('ix_orders_user_status', 'orders', ['user_id', 'status'])

def downgrade():
    op.drop_index('ix_orders_user_status', 'orders')
    op.drop_index('ix_orders_created', 'orders')
    op.drop_index('ix_users_email', 'users')


# Внешние ключи
def upgrade():
    op.create_foreign_key(
        'fk_orders_user',
        'orders', 'users',
        ['user_id'], ['id'],
        ondelete='CASCADE'
    )

def downgrade():
    op.drop_constraint('fk_orders_user', 'orders', type_='foreignkey')


# CHECK constraint
def upgrade():
    op.create_check_constraint(
        'ck_products_price_positive',
        'products',
        'price > 0'
    )
```

### 4. Миграции данных (Data Migrations)

```python
from sqlalchemy.sql import table, column
from sqlalchemy import String, Integer

def upgrade():
    # Определяем структуру таблицы для работы с данными
    users = table(
        'users',
        column('id', Integer),
        column('email', String),
        column('email_verified', Integer)
    )

    # Bulk update
    op.execute(
        users.update()
        .where(users.c.email.like('%@company.com'))
        .values(email_verified=1)
    )

    # Insert начальных данных
    op.bulk_insert(
        table('roles', column('id', Integer), column('name', String)),
        [
            {'id': 1, 'name': 'admin'},
            {'id': 2, 'name': 'user'},
            {'id': 3, 'name': 'moderator'},
        ]
    )

def downgrade():
    op.execute("DELETE FROM roles WHERE id IN (1, 2, 3)")
```

---

## Примеры кода миграций на Alembic

### Полный пример: система заказов

```python
"""create order system tables

Revision ID: abc123def456
Revises: a1b2c3d4e5f6
Create Date: 2024-01-20 14:30:00.000000
"""
from alembic import op
import sqlalchemy as sa
from sqlalchemy.dialects import postgresql

revision = 'abc123def456'
down_revision = 'a1b2c3d4e5f6'
branch_labels = None
depends_on = None


def upgrade() -> None:
    # Создаём ENUM тип для статуса
    order_status = postgresql.ENUM(
        'pending', 'processing', 'shipped', 'delivered', 'cancelled',
        name='order_status'
    )
    order_status.create(op.get_bind())

    # Таблица заказов
    op.create_table(
        'orders',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('user_id', sa.Integer(), sa.ForeignKey('users.id', ondelete='CASCADE'), nullable=False),
        sa.Column('status', postgresql.ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled', name='order_status'), nullable=False, server_default='pending'),
        sa.Column('total_amount', sa.Numeric(12, 2), nullable=False),
        sa.Column('shipping_address', sa.Text(), nullable=True),
        sa.Column('created_at', sa.DateTime(), server_default=sa.func.now()),
        sa.Column('updated_at', sa.DateTime(), server_default=sa.func.now(), onupdate=sa.func.now()),
    )

    # Таблица позиций заказа
    op.create_table(
        'order_items',
        sa.Column('id', sa.Integer(), primary_key=True),
        sa.Column('order_id', sa.Integer(), sa.ForeignKey('orders.id', ondelete='CASCADE'), nullable=False),
        sa.Column('product_id', sa.Integer(), sa.ForeignKey('products.id', ondelete='RESTRICT'), nullable=False),
        sa.Column('quantity', sa.Integer(), nullable=False),
        sa.Column('unit_price', sa.Numeric(10, 2), nullable=False),
        sa.Column('total_price', sa.Numeric(12, 2), nullable=False),
    )

    # Индексы
    op.create_index('ix_orders_user_id', 'orders', ['user_id'])
    op.create_index('ix_orders_status', 'orders', ['status'])
    op.create_index('ix_orders_created_at', 'orders', ['created_at'])
    op.create_index('ix_order_items_order_id', 'order_items', ['order_id'])

    # Ограничения
    op.create_check_constraint('ck_order_items_quantity', 'order_items', 'quantity > 0')
    op.create_check_constraint('ck_orders_total', 'orders', 'total_amount >= 0')


def downgrade() -> None:
    # Удаляем ограничения
    op.drop_constraint('ck_orders_total', 'orders')
    op.drop_constraint('ck_order_items_quantity', 'order_items')

    # Удаляем индексы
    op.drop_index('ix_order_items_order_id', 'order_items')
    op.drop_index('ix_orders_created_at', 'orders')
    op.drop_index('ix_orders_status', 'orders')
    op.drop_index('ix_orders_user_id', 'orders')

    # Удаляем таблицы
    op.drop_table('order_items')
    op.drop_table('orders')

    # Удаляем ENUM тип
    order_status = postgresql.ENUM('pending', 'processing', 'shipped', 'delivered', 'cancelled', name='order_status')
    order_status.drop(op.get_bind())
```

### Миграция с разделением на batch операции (для SQLite)

```python
"""add constraints to users

Revision ID: def789ghi012
"""
from alembic import op
import sqlalchemy as sa

revision = 'def789ghi012'
down_revision = 'abc123def456'


def upgrade() -> None:
    # SQLite не поддерживает ALTER TABLE для добавления constraints
    # Используем batch mode
    with op.batch_alter_table('users') as batch_op:
        batch_op.add_column(sa.Column('age', sa.Integer(), nullable=True))
        batch_op.create_check_constraint('ck_users_age', 'age >= 0 AND age <= 150')
        batch_op.alter_column('email', nullable=False)


def downgrade() -> None:
    with op.batch_alter_table('users') as batch_op:
        batch_op.alter_column('email', nullable=True)
        batch_op.drop_constraint('ck_users_age')
        batch_op.drop_column('age')
```

---

## Откат миграций (Downgrade)

### Команды отката

```bash
# Alembic
alembic downgrade -1          # Откатить одну миграцию
alembic downgrade -3          # Откатить три миграции
alembic downgrade abc123      # Откатить до конкретной ревизии
alembic downgrade base        # Откатить все миграции

# Django
python manage.py migrate app_name 0005  # Откатить до миграции 0005
python manage.py migrate app_name zero  # Откатить все миграции приложения

# Flyway (платная версия)
flyway undo

# Liquibase
liquibase rollbackCount 1
liquibase rollback v1.0       # Откат до тега
```

### Правила написания downgrade

```python
def upgrade():
    # 1. Добавляем колонку
    op.add_column('users', sa.Column('nickname', sa.String(50)))

    # 2. Заполняем данными
    op.execute("UPDATE users SET nickname = username")

    # 3. Делаем NOT NULL
    op.alter_column('users', 'nickname', nullable=False)

def downgrade():
    # Обратный порядок!
    # 3. Убираем NOT NULL (необязательно, так как удаляем колонку)
    # 2. Данные теряются при удалении
    # 1. Удаляем колонку
    op.drop_column('users', 'nickname')
```

### Необратимые миграции

Некоторые миграции нельзя откатить без потери данных:

```python
def upgrade():
    # Удаление колонки — необратимо!
    op.drop_column('users', 'old_field')

def downgrade():
    # Можем восстановить структуру, но не данные
    op.add_column('users', sa.Column('old_field', sa.String(100)))
    # Данные потеряны навсегда!

    # Или явно запретить откат
    raise Exception("This migration cannot be reversed. Data was deleted.")
```

---

## Best Practices

### 1. Атомарность миграций

Каждая миграция должна быть атомарной — либо полностью применяется, либо полностью откатывается.

```python
# Хорошо: одно логическое изменение
def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20)))

# Плохо: несколько несвязанных изменений
def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20)))
    op.create_table('products', ...)
    op.alter_column('orders', 'status', ...)
```

### 2. Backward Compatibility

Миграции должны поддерживать старый код во время деплоя:

```python
# Этап 1: Добавляем новую колонку (nullable)
def upgrade_step1():
    op.add_column('users', sa.Column('full_name', sa.String(200), nullable=True))

# Деплоим новый код, который пишет в обе колонки

# Этап 2: Копируем данные и делаем NOT NULL
def upgrade_step2():
    op.execute("UPDATE users SET full_name = first_name || ' ' || last_name WHERE full_name IS NULL")
    op.alter_column('users', 'full_name', nullable=False)

# Убеждаемся, что новый код работает

# Этап 3: Удаляем старые колонки (отдельный деплой!)
def upgrade_step3():
    op.drop_column('users', 'first_name')
    op.drop_column('users', 'last_name')
```

### 3. Zero-Downtime миграции

```python
# ПЛОХО: блокирует таблицу
def upgrade():
    op.add_column('orders', sa.Column('status', sa.String(20), nullable=False))

# ХОРОШО: поэтапно
def upgrade():
    # Шаг 1: Добавляем nullable колонку (быстро, без блокировки)
    op.add_column('orders', sa.Column('status', sa.String(20), nullable=True))

# Отдельная миграция после заполнения данных
def upgrade():
    # Шаг 2: Устанавливаем default и NOT NULL
    op.execute("UPDATE orders SET status = 'pending' WHERE status IS NULL")
    op.alter_column('orders', 'status', nullable=False, server_default='pending')
```

**Опасные операции для больших таблиц:**
- `ALTER TABLE ... ADD COLUMN ... NOT NULL` — блокирует таблицу
- `CREATE INDEX` — блокирует запись (используйте `CONCURRENTLY` в PostgreSQL)
- `ALTER TABLE ... ADD CONSTRAINT` — сканирует всю таблицу

```python
# Создание индекса без блокировки (PostgreSQL)
def upgrade():
    op.execute("CREATE INDEX CONCURRENTLY ix_orders_date ON orders(created_at)")

def downgrade():
    op.execute("DROP INDEX CONCURRENTLY ix_orders_date")
```

### 4. Тестирование миграций

```python
# tests/test_migrations.py
import pytest
from alembic import command
from alembic.config import Config

@pytest.fixture
def alembic_config():
    config = Config("alembic.ini")
    return config

def test_upgrade_downgrade_cycle(alembic_config, test_database):
    """Проверяем, что все миграции могут быть применены и откачены."""
    # Применяем все миграции
    command.upgrade(alembic_config, "head")

    # Откатываем до начала
    command.downgrade(alembic_config, "base")

    # Снова применяем
    command.upgrade(alembic_config, "head")

def test_migration_generates_valid_sql(alembic_config):
    """Проверяем, что миграция генерирует валидный SQL."""
    # Получаем SQL без применения
    command.upgrade(alembic_config, "head", sql=True)
```

### 5. Naming Conventions

```python
# Файлы миграций
20240115_120000_create_users_table.py     # Timestamp prefix
001_create_users_table.py                  # Numeric prefix

# Constraints
fk_orders_user_id                          # Foreign key
ix_users_email                             # Index
ck_products_price_positive                 # Check constraint
uq_users_username                          # Unique constraint
pk_users                                   # Primary key
```

---

## Типичные ошибки

### 1. Изменение уже применённой миграции

```python
# ❌ НИКОГДА не делайте так!
# Миграция уже в production, а вы её изменили

# ✅ Создайте новую миграцию для исправления
alembic revision -m "fix users table"
```

### 2. Зависимость от данных в миграции

```python
# ❌ Опасно: миграция зависит от конкретных данных
def upgrade():
    user = session.query(User).filter_by(email='admin@example.com').first()
    user.role = 'superadmin'

# ✅ Безопасно: используем SQL напрямую
def upgrade():
    op.execute("UPDATE users SET role = 'superadmin' WHERE email = 'admin@example.com'")
```

### 3. Отсутствие downgrade

```python
# ❌ Нет возможности отката
def upgrade():
    op.create_table('new_table', ...)

def downgrade():
    pass  # Забыли реализовать!

# ✅ Всегда реализуйте downgrade
def downgrade():
    op.drop_table('new_table')
```

### 4. Блокирующие операции на больших таблицах

```python
# ❌ Заблокирует таблицу с миллионами записей
def upgrade():
    op.create_index('ix_huge_table_col', 'huge_table', ['column'])

# ✅ Используйте CONCURRENTLY (PostgreSQL)
def upgrade():
    op.execute("CREATE INDEX CONCURRENTLY ix_huge_table_col ON huge_table(column)")
```

### 5. Неправильный порядок операций

```python
# ❌ Ошибка: создаём FK до таблицы
def upgrade():
    op.add_column('orders',
        sa.Column('product_id', sa.Integer(),
        sa.ForeignKey('products.id')))  # products ещё не существует!
    op.create_table('products', ...)

# ✅ Правильный порядок
def upgrade():
    op.create_table('products', ...)  # Сначала таблица
    op.add_column('orders',
        sa.Column('product_id', sa.Integer(),
        sa.ForeignKey('products.id')))  # Потом FK
```

### 6. Игнорирование различий между СУБД

```python
# ❌ Работает только в PostgreSQL
def upgrade():
    op.execute("ALTER TABLE users ADD COLUMN data JSONB")

# ✅ Кроссплатформенно через SQLAlchemy
def upgrade():
    from sqlalchemy.dialects.postgresql import JSONB
    from sqlalchemy import JSON

    # Определяем тип в зависимости от БД
    dialect = op.get_bind().dialect.name
    json_type = JSONB() if dialect == 'postgresql' else JSON()

    op.add_column('users', sa.Column('data', json_type))
```

---

## Работа с ветками миграций (Alembic)

При работе в команде могут возникнуть конфликты миграций:

```
main:    A -> B -> C
                     \
feature:              D (конфликт с C!)
```

### Решение конфликтов

```bash
# Смотрим текущие головы (heads)
alembic heads

# Если несколько голов — создаём merge-миграцию
alembic merge -m "merge branches" head1 head2

# Или указываем зависимость вручную
alembic revision --head head1 -m "new migration"
```

```python
# merge_migration.py
revision = 'merge_abc_def'
down_revision = ('abc123', 'def456')  # Две родительские миграции

def upgrade():
    pass  # Merge-миграция обычно пустая

def downgrade():
    pass
```

---

## Резюме

### Ключевые принципы

1. **Версионирование** — каждое изменение схемы фиксируется в миграции
2. **Упорядоченность** — миграции применяются строго последовательно
3. **Неизменяемость** — после применения миграцию нельзя менять
4. **Обратимость** — всегда реализуйте downgrade
5. **Атомарность** — одна миграция = одно логическое изменение

### Workflow миграций

```
1. Изменяем модели в коде
2. Генерируем миграцию (autogenerate или вручную)
3. Проверяем сгенерированный код
4. Тестируем локально (upgrade + downgrade)
5. Коммитим миграцию вместе с изменениями кода
6. В CI/CD миграции применяются автоматически
```

### Checklist перед деплоем

- [ ] Миграция протестирована локально (upgrade/downgrade)
- [ ] downgrade реализован корректно
- [ ] Нет блокирующих операций на больших таблицах
- [ ] Миграция backward-compatible с текущим кодом
- [ ] Есть план отката при проблемах
- [ ] Миграция прошла code review
