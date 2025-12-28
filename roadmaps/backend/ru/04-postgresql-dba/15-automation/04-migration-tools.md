# Инструменты миграций базы данных

[prev: 03-migrations](./03-migrations.md) | [next: 01-bulk-loading](../16-data-processing/01-bulk-loading.md)

---

## Обзор инструментов

| Инструмент | Язык | Формат миграций | Особенности |
|------------|------|-----------------|-------------|
| Flyway | Java | SQL/Java | Простота, широкая поддержка |
| Liquibase | Java | XML/YAML/JSON/SQL | Гибкость, rollback |
| Alembic | Python | Python | Интеграция с SQLAlchemy |
| golang-migrate | Go | SQL | Легковесный, CLI |
| sqitch | Perl | SQL | Git-подобный подход |
| Django Migrations | Python | Python | Встроен в Django |
| TypeORM Migrations | TypeScript | TypeScript | Для Node.js проектов |

## Flyway

### Установка

```bash
# macOS
brew install flyway

# Linux (через скачивание)
wget -qO- https://repo1.maven.org/maven2/org/flywaydb/flyway-commandline/9.22.0/flyway-commandline-9.22.0-linux-x64.tar.gz | tar xvz
sudo ln -s $(pwd)/flyway-9.22.0/flyway /usr/local/bin

# Docker
docker pull flyway/flyway

# Проверка
flyway --version
```

### Конфигурация

```properties
# flyway.conf
flyway.url=jdbc:postgresql://localhost:5432/mydb
flyway.user=postgres
flyway.password=secret
flyway.locations=filesystem:./migrations
flyway.schemas=public
flyway.baselineOnMigrate=true
flyway.validateOnMigrate=true
flyway.cleanDisabled=true
```

### Структура проекта

```
project/
├── flyway.conf
├── migrations/
│   ├── V1__create_users_table.sql
│   ├── V2__create_orders_table.sql
│   ├── V3__add_email_index.sql
│   ├── R__refresh_views.sql          # Repeatable migration
│   └── U2__undo_create_orders.sql    # Undo migration (Teams+)
└── sql/
    └── afterMigrate.sql              # Callback script
```

### Именование миграций

```
V{version}__{description}.sql    - Версионированные (применяются один раз)
U{version}__{description}.sql    - Undo миграции (откат)
R__{description}.sql             - Repeatable (применяются при изменении)
```

### Примеры миграций

```sql
-- V1__create_users_table.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW(),
    updated_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- V2__create_orders_table.sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    total NUMERIC(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_orders_user_id ON orders(user_id);
CREATE INDEX idx_orders_status ON orders(status);

-- V3__add_phone_to_users.sql
ALTER TABLE users ADD COLUMN phone VARCHAR(20);

-- R__create_user_stats_view.sql (Repeatable)
CREATE OR REPLACE VIEW user_stats AS
SELECT
    u.id,
    u.email,
    COUNT(o.id) as order_count,
    COALESCE(SUM(o.total), 0) as total_spent
FROM users u
LEFT JOIN orders o ON o.user_id = u.id
GROUP BY u.id, u.email;
```

### Команды Flyway

```bash
# Применить все pending миграции
flyway migrate

# Показать статус миграций
flyway info

# Проверить миграции без применения
flyway validate

# Очистить схему (ОПАСНО!)
flyway clean

# Создать baseline для существующей БД
flyway baseline -baselineVersion=1

# Восстановить таблицу истории
flyway repair

# Откат последней миграции (Teams+)
flyway undo
```

### Использование с Docker

```bash
# Запуск миграций
docker run --rm \
    -v $(pwd)/migrations:/flyway/sql \
    flyway/flyway \
    -url=jdbc:postgresql://host.docker.internal:5432/mydb \
    -user=postgres \
    -password=secret \
    migrate

# Docker Compose
```

```yaml
# docker-compose.yml
version: '3.8'
services:
  db:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: secret
    volumes:
      - pgdata:/var/lib/postgresql/data

  flyway:
    image: flyway/flyway
    depends_on:
      - db
    volumes:
      - ./migrations:/flyway/sql
    command: -url=jdbc:postgresql://db:5432/postgres -user=postgres -password=secret migrate

volumes:
  pgdata:
```

## Liquibase

### Установка

```bash
# macOS
brew install liquibase

# Linux
wget https://github.com/liquibase/liquibase/releases/download/v4.25.0/liquibase-4.25.0.tar.gz
tar -xzf liquibase-4.25.0.tar.gz
sudo mv liquibase /opt/
sudo ln -s /opt/liquibase/liquibase /usr/local/bin/

# Скачивание PostgreSQL драйвера
wget https://jdbc.postgresql.org/download/postgresql-42.6.0.jar -P /opt/liquibase/lib/
```

### Конфигурация

```properties
# liquibase.properties
changeLogFile=changelog/db.changelog-master.xml
url=jdbc:postgresql://localhost:5432/mydb
username=postgres
password=secret
driver=org.postgresql.Driver
```

### Структура проекта

```
project/
├── liquibase.properties
└── changelog/
    ├── db.changelog-master.xml
    ├── db.changelog-1.0.xml
    ├── db.changelog-1.1.xml
    └── changes/
        ├── 001-create-users.xml
        ├── 002-create-orders.xml
        └── 003-add-indexes.sql
```

### Форматы changelog

#### XML формат

```xml
<!-- db.changelog-master.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.9.xsd">

    <include file="db.changelog-1.0.xml" relativeToChangelogFile="true"/>
    <include file="db.changelog-1.1.xml" relativeToChangelogFile="true"/>

</databaseChangeLog>

<!-- db.changelog-1.0.xml -->
<?xml version="1.0" encoding="UTF-8"?>
<databaseChangeLog
    xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
    xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
    xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog
        http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.9.xsd">

    <changeSet id="1" author="developer">
        <comment>Create users table</comment>
        <createTable tableName="users">
            <column name="id" type="SERIAL">
                <constraints primaryKey="true" nullable="false"/>
            </column>
            <column name="email" type="VARCHAR(255)">
                <constraints nullable="false" unique="true"/>
            </column>
            <column name="password_hash" type="VARCHAR(255)">
                <constraints nullable="false"/>
            </column>
            <column name="created_at" type="TIMESTAMP" defaultValueComputed="NOW()"/>
        </createTable>
        <rollback>
            <dropTable tableName="users"/>
        </rollback>
    </changeSet>

    <changeSet id="2" author="developer">
        <comment>Create orders table</comment>
        <createTable tableName="orders">
            <column name="id" type="SERIAL">
                <constraints primaryKey="true"/>
            </column>
            <column name="user_id" type="INTEGER">
                <constraints nullable="false"
                    foreignKeyName="fk_orders_user"
                    references="users(id)"/>
            </column>
            <column name="total" type="NUMERIC(10,2)">
                <constraints nullable="false"/>
            </column>
            <column name="status" type="VARCHAR(20)" defaultValue="pending"/>
        </createTable>
    </changeSet>

    <changeSet id="3" author="developer">
        <createIndex tableName="orders" indexName="idx_orders_user_id">
            <column name="user_id"/>
        </createIndex>
    </changeSet>

</databaseChangeLog>
```

#### YAML формат

```yaml
# db.changelog-1.0.yaml
databaseChangeLog:
  - changeSet:
      id: 1
      author: developer
      comment: Create users table
      changes:
        - createTable:
            tableName: users
            columns:
              - column:
                  name: id
                  type: SERIAL
                  constraints:
                    primaryKey: true
                    nullable: false
              - column:
                  name: email
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
                    unique: true
              - column:
                  name: password_hash
                  type: VARCHAR(255)
                  constraints:
                    nullable: false
      rollback:
        - dropTable:
            tableName: users

  - changeSet:
      id: 2
      author: developer
      changes:
        - addColumn:
            tableName: users
            columns:
              - column:
                  name: phone
                  type: VARCHAR(20)
```

#### SQL формат

```sql
-- db.changelog-1.0.sql

-- changeset developer:1
-- comment: Create users table
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
-- rollback DROP TABLE users;

-- changeset developer:2
-- comment: Add phone column
ALTER TABLE users ADD COLUMN phone VARCHAR(20);
-- rollback ALTER TABLE users DROP COLUMN phone;

-- changeset developer:3 runOnChange:true
-- comment: Create view (repeatable)
CREATE OR REPLACE VIEW active_users AS
SELECT * FROM users WHERE created_at > NOW() - INTERVAL '30 days';
```

### Команды Liquibase

```bash
# Применить изменения
liquibase update

# Откат последнего changeset
liquibase rollbackCount 1

# Откат до определенного тега
liquibase rollback version_1.0

# Откат до даты
liquibase rollbackToDate 2024-01-01

# Генерация SQL без применения
liquibase updateSQL > update.sql

# Статус изменений
liquibase status

# Тегирование версии
liquibase tag version_1.0

# Генерация changelog из существующей БД
liquibase generateChangeLog --changeLogFile=generated.xml

# Сравнение двух баз
liquibase diff --referenceUrl=jdbc:postgresql://localhost/prod

# Проверка синтаксиса
liquibase validate
```

### Preconditions и Context

```xml
<changeSet id="4" author="developer">
    <!-- Выполнить только если таблица существует -->
    <preConditions onFail="MARK_RAN">
        <tableExists tableName="legacy_users"/>
    </preConditions>

    <dropTable tableName="legacy_users"/>
</changeSet>

<changeSet id="5" author="developer" context="production">
    <!-- Выполнить только в production -->
    <insert tableName="config">
        <column name="key" value="feature_flag"/>
        <column name="value" value="enabled"/>
    </insert>
</changeSet>

<!-- Запуск с контекстом -->
<!-- liquibase --contexts=production update -->
```

## Alembic (Python)

### Установка

```bash
pip install alembic psycopg2-binary
```

### Инициализация

```bash
# Создание структуры Alembic
alembic init alembic

# Для async проекта
alembic init -t async alembic
```

### Структура проекта

```
project/
├── alembic.ini
├── alembic/
│   ├── env.py
│   ├── script.py.mako
│   └── versions/
│       ├── 001_create_users.py
│       └── 002_create_orders.py
└── app/
    └── models.py
```

### Конфигурация

```ini
# alembic.ini
[alembic]
script_location = alembic
prepend_sys_path = .
sqlalchemy.url = postgresql://postgres:secret@localhost/mydb

[logging]
level = INFO
```

```python
# alembic/env.py
from logging.config import fileConfig
from sqlalchemy import engine_from_config, pool
from alembic import context

# Импорт моделей для autogenerate
from app.models import Base

config = context.config
fileConfig(config.config_file_name)

target_metadata = Base.metadata

def run_migrations_offline():
    """Run migrations in 'offline' mode."""
    url = config.get_main_option("sqlalchemy.url")
    context.configure(
        url=url,
        target_metadata=target_metadata,
        literal_binds=True,
        dialect_opts={"paramstyle": "named"},
    )

    with context.begin_transaction():
        context.run_migrations()

def run_migrations_online():
    """Run migrations in 'online' mode."""
    connectable = engine_from_config(
        config.get_section(config.config_ini_section),
        prefix="sqlalchemy.",
        poolclass=pool.NullPool,
    )

    with connectable.connect() as connection:
        context.configure(
            connection=connection,
            target_metadata=target_metadata,
            compare_type=True,
            compare_server_default=True,
        )

        with context.begin_transaction():
            context.run_migrations()

if context.is_offline_mode():
    run_migrations_offline()
else:
    run_migrations_online()
```

### Создание миграций

```bash
# Автоматическая генерация миграции из моделей
alembic revision --autogenerate -m "create users table"

# Ручное создание миграции
alembic revision -m "add phone to users"
```

### Примеры миграций

```python
# alembic/versions/001_create_users.py
"""create users table

Revision ID: 001
Revises:
Create Date: 2024-01-15 10:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

# revision identifiers
revision = '001'
down_revision = None
branch_labels = None
depends_on = None

def upgrade():
    op.create_table(
        'users',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('email', sa.String(255), nullable=False),
        sa.Column('password_hash', sa.String(255), nullable=False),
        sa.Column('created_at', sa.DateTime(), server_default=sa.text('NOW()')),
        sa.PrimaryKeyConstraint('id'),
        sa.UniqueConstraint('email')
    )
    op.create_index('idx_users_email', 'users', ['email'])

def downgrade():
    op.drop_index('idx_users_email', table_name='users')
    op.drop_table('users')


# alembic/versions/002_create_orders.py
"""create orders table

Revision ID: 002
Revises: 001
Create Date: 2024-01-15 11:00:00.000000
"""
from alembic import op
import sqlalchemy as sa

revision = '002'
down_revision = '001'
branch_labels = None
depends_on = None

def upgrade():
    op.create_table(
        'orders',
        sa.Column('id', sa.Integer(), nullable=False),
        sa.Column('user_id', sa.Integer(), nullable=False),
        sa.Column('total', sa.Numeric(10, 2), nullable=False),
        sa.Column('status', sa.String(20), server_default='pending'),
        sa.Column('created_at', sa.DateTime(), server_default=sa.text('NOW()')),
        sa.PrimaryKeyConstraint('id'),
        sa.ForeignKeyConstraint(['user_id'], ['users.id'], name='fk_orders_user')
    )
    op.create_index('idx_orders_user_id', 'orders', ['user_id'])

def downgrade():
    op.drop_index('idx_orders_user_id', table_name='orders')
    op.drop_table('orders')


# alembic/versions/003_add_phone_to_users.py
"""add phone to users

Revision ID: 003
Revises: 002
"""
from alembic import op
import sqlalchemy as sa

revision = '003'
down_revision = '002'

def upgrade():
    op.add_column('users', sa.Column('phone', sa.String(20), nullable=True))

def downgrade():
    op.drop_column('users', 'phone')
```

### Продвинутые операции

```python
# Batch операции для SQLite и больших таблиц
def upgrade():
    with op.batch_alter_table('users') as batch_op:
        batch_op.add_column(sa.Column('phone', sa.String(20)))
        batch_op.create_index('idx_users_phone', ['phone'])

# Выполнение raw SQL
def upgrade():
    op.execute("""
        CREATE INDEX CONCURRENTLY idx_users_email_lower
        ON users(lower(email))
    """)

# Data migration
def upgrade():
    # Добавление колонки
    op.add_column('users', sa.Column('full_name', sa.String(200)))

    # Миграция данных
    connection = op.get_bind()
    connection.execute(sa.text("""
        UPDATE users
        SET full_name = first_name || ' ' || last_name
    """))

    # Удаление старых колонок
    op.drop_column('users', 'first_name')
    op.drop_column('users', 'last_name')
```

### Команды Alembic

```bash
# Применить все миграции
alembic upgrade head

# Применить до конкретной ревизии
alembic upgrade 002

# Откат на одну ревизию
alembic downgrade -1

# Откат до конкретной ревизии
alembic downgrade 001

# Показать текущую ревизию
alembic current

# История миграций
alembic history

# Генерация SQL без применения
alembic upgrade head --sql > migration.sql
```

## golang-migrate

### Установка

```bash
# macOS
brew install golang-migrate

# Linux
curl -L https://github.com/golang-migrate/migrate/releases/download/v4.16.2/migrate.linux-amd64.tar.gz | tar xvz
sudo mv migrate /usr/local/bin/

# Go
go install -tags 'postgres' github.com/golang-migrate/migrate/v4/cmd/migrate@latest
```

### Структура миграций

```
migrations/
├── 000001_create_users_table.up.sql
├── 000001_create_users_table.down.sql
├── 000002_create_orders_table.up.sql
├── 000002_create_orders_table.down.sql
├── 000003_add_phone_to_users.up.sql
└── 000003_add_phone_to_users.down.sql
```

### Примеры миграций

```sql
-- 000001_create_users_table.up.sql
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    email VARCHAR(255) NOT NULL UNIQUE,
    password_hash VARCHAR(255) NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);

CREATE INDEX idx_users_email ON users(email);

-- 000001_create_users_table.down.sql
DROP INDEX IF EXISTS idx_users_email;
DROP TABLE IF EXISTS users;

-- 000002_create_orders_table.up.sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER NOT NULL REFERENCES users(id),
    total NUMERIC(10,2) NOT NULL,
    status VARCHAR(20) DEFAULT 'pending',
    created_at TIMESTAMP DEFAULT NOW()
);

-- 000002_create_orders_table.down.sql
DROP TABLE IF EXISTS orders;
```

### Команды

```bash
# Создание новой миграции
migrate create -ext sql -dir migrations -seq create_products_table

# Применить все миграции
migrate -path migrations -database "postgresql://user:pass@localhost/db?sslmode=disable" up

# Применить N миграций
migrate -path migrations -database "postgresql://user:pass@localhost/db?sslmode=disable" up 2

# Откат всех миграций
migrate -path migrations -database "postgresql://user:pass@localhost/db?sslmode=disable" down

# Откат N миграций
migrate -path migrations -database "postgresql://user:pass@localhost/db?sslmode=disable" down 1

# Перейти к версии
migrate -path migrations -database "postgresql://user:pass@localhost/db?sslmode=disable" goto 3

# Показать версию
migrate -path migrations -database "postgresql://user:pass@localhost/db?sslmode=disable" version

# Force версия (исправление после ошибки)
migrate -path migrations -database "postgresql://user:pass@localhost/db?sslmode=disable" force 2
```

### Использование в Go коде

```go
package main

import (
    "database/sql"
    "log"

    "github.com/golang-migrate/migrate/v4"
    "github.com/golang-migrate/migrate/v4/database/postgres"
    _ "github.com/golang-migrate/migrate/v4/source/file"
    _ "github.com/lib/pq"
)

func main() {
    db, err := sql.Open("postgres", "postgresql://user:pass@localhost/db?sslmode=disable")
    if err != nil {
        log.Fatal(err)
    }

    driver, err := postgres.WithInstance(db, &postgres.Config{})
    if err != nil {
        log.Fatal(err)
    }

    m, err := migrate.NewWithDatabaseInstance(
        "file://migrations",
        "postgres",
        driver,
    )
    if err != nil {
        log.Fatal(err)
    }

    // Применить все миграции
    if err := m.Up(); err != nil && err != migrate.ErrNoChange {
        log.Fatal(err)
    }

    log.Println("Migrations applied successfully")
}
```

## Django Migrations

### Создание и применение

```bash
# Создание миграции из изменений моделей
python manage.py makemigrations app_name

# Применение миграций
python manage.py migrate

# Показать SQL миграции
python manage.py sqlmigrate app_name 0001

# Статус миграций
python manage.py showmigrations

# Откат до миграции
python manage.py migrate app_name 0001
```

### Пример модели и миграции

```python
# models.py
from django.db import models

class User(models.Model):
    email = models.EmailField(unique=True)
    password_hash = models.CharField(max_length=255)
    phone = models.CharField(max_length=20, blank=True, null=True)
    created_at = models.DateTimeField(auto_now_add=True)

# migrations/0001_initial.py (автогенерация)
from django.db import migrations, models

class Migration(migrations.Migration):
    initial = True

    dependencies = []

    operations = [
        migrations.CreateModel(
            name='User',
            fields=[
                ('id', models.BigAutoField(auto_created=True, primary_key=True)),
                ('email', models.EmailField(max_length=254, unique=True)),
                ('password_hash', models.CharField(max_length=255)),
                ('created_at', models.DateTimeField(auto_now_add=True)),
            ],
        ),
    ]

# Data migration
from django.db import migrations

def forwards_func(apps, schema_editor):
    User = apps.get_model('myapp', 'User')
    User.objects.filter(email__endswith='@old.com').update(
        email=models.F('email').replace('@old.com', '@new.com')
    )

def reverse_func(apps, schema_editor):
    pass  # Необратимая операция

class Migration(migrations.Migration):
    dependencies = [('myapp', '0002_add_phone')]

    operations = [
        migrations.RunPython(forwards_func, reverse_func),
    ]
```

## Сравнение и выбор инструмента

### Критерии выбора

| Критерий | Flyway | Liquibase | Alembic | golang-migrate |
|----------|--------|-----------|---------|----------------|
| Простота | Высокая | Средняя | Средняя | Высокая |
| Гибкость | Средняя | Высокая | Высокая | Низкая |
| Rollback | Платный | Бесплатный | Бесплатный | Бесплатный |
| Autogenerate | Нет | Нет | Да | Нет |
| Язык проекта | Любой | Любой | Python | Go |
| Enterprise | Да | Да | Нет | Нет |

### Рекомендации

- **Java/JVM проекты**: Flyway или Liquibase
- **Python проекты**: Alembic (особенно с SQLAlchemy)
- **Go проекты**: golang-migrate
- **Node.js проекты**: TypeORM Migrations или Knex.js
- **Простые проекты**: Flyway или golang-migrate
- **Сложные требования**: Liquibase

## Best Practices для всех инструментов

### 1. CI/CD интеграция

```yaml
# GitHub Actions
name: Database Migrations

on:
  push:
    branches: [main]
    paths:
      - 'migrations/**'

jobs:
  migrate:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Run migrations
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL }}
        run: |
          flyway -url=$DATABASE_URL migrate
```

### 2. Безопасность

```bash
# Никогда не храните пароли в конфигах
# Используйте переменные окружения
export DATABASE_URL="postgresql://user:${DB_PASSWORD}@host/db"

# Или секреты CI/CD систем
flyway -url=${DATABASE_URL} migrate
```

### 3. Тестирование

```bash
# Тестирование на чистой базе
createdb test_db
flyway -url=jdbc:postgresql://localhost/test_db migrate
flyway -url=jdbc:postgresql://localhost/test_db undo -target=001
flyway -url=jdbc:postgresql://localhost/test_db migrate
dropdb test_db
```

### 4. Документация

```sql
-- V010__add_audit_columns.sql
--
-- Добавляет колонки аудита во все таблицы
-- Автор: security-team
-- Задача: SEC-123
--
-- Изменения:
-- - created_by: ID пользователя создавшего запись
-- - updated_by: ID пользователя обновившего запись
-- - deleted_at: Soft delete timestamp

ALTER TABLE users ADD COLUMN created_by INTEGER;
-- ...
```

## Заключение

Выбор инструмента миграций зависит от:

1. **Языка проекта** - используйте нативные инструменты когда возможно
2. **Требований к rollback** - важен ли автоматический откат
3. **Размера команды** - enterprise функции для больших команд
4. **Сложности схемы** - XML/YAML для сложных сценариев

Независимо от выбранного инструмента, ключевые принципы остаются одинаковыми:
- Версионирование всех изменений
- Тестирование миграций
- Автоматизация через CI/CD
- Документирование изменений

---

[prev: 03-migrations](./03-migrations.md) | [next: 01-bulk-loading](../16-data-processing/01-bulk-loading.md)
