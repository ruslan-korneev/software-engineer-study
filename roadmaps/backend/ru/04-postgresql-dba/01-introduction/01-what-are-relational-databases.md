# Что такое реляционные базы данных?

[prev: 07-git-patch](../../03-version-control-systems/15-advanced-git/07-git-patch.md) | [next: 02-rdbms-benefits-limitations](./02-rdbms-benefits-limitations.md)
---

## Определение

**Реляционная база данных** — это тип базы данных, который хранит и организует данные в виде таблиц (отношений), связанных между собой по определённым правилам. Концепция была предложена Эдгаром Коддом в 1970 году в его работе "A Relational Model of Data for Large Shared Data Banks".

## Основные понятия

### Таблица (Relation)

Таблица — это основная единица хранения данных. Она состоит из:
- **Строк (rows/tuples)** — отдельные записи данных
- **Столбцов (columns/attributes)** — характеристики данных

```sql
-- Пример таблицы users
CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    username VARCHAR(50) NOT NULL,
    email VARCHAR(100) UNIQUE,
    created_at TIMESTAMP DEFAULT CURRENT_TIMESTAMP
);
```

### Первичный ключ (Primary Key)

Уникальный идентификатор каждой строки в таблице. Гарантирует, что каждая запись может быть однозначно идентифицирована.

```sql
-- id является первичным ключом
id SERIAL PRIMARY KEY
```

### Внешний ключ (Foreign Key)

Столбец (или группа столбцов), который ссылается на первичный ключ другой таблицы, создавая связь между таблицами.

```sql
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),  -- внешний ключ
    total_amount DECIMAL(10, 2),
    order_date DATE
);
```

### Схема (Schema)

Логическая структура базы данных, определяющая таблицы, их столбцы, типы данных и связи между таблицами.

## Типы связей между таблицами

### Один-к-одному (One-to-One)

Каждая запись в таблице A связана максимум с одной записью в таблице B.

```sql
CREATE TABLE user_profiles (
    id SERIAL PRIMARY KEY,
    user_id INTEGER UNIQUE REFERENCES users(id),
    bio TEXT,
    avatar_url VARCHAR(255)
);
```

### Один-ко-многим (One-to-Many)

Одна запись в таблице A может быть связана с множеством записей в таблице B.

```sql
-- Один пользователь может иметь много заказов
CREATE TABLE orders (
    id SERIAL PRIMARY KEY,
    user_id INTEGER REFERENCES users(id),  -- много заказов -> один пользователь
    product_name VARCHAR(100)
);
```

### Многие-ко-многим (Many-to-Many)

Множество записей в таблице A могут быть связаны с множеством записей в таблице B. Реализуется через промежуточную таблицу.

```sql
-- Студенты и курсы
CREATE TABLE students (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE courses (
    id SERIAL PRIMARY KEY,
    title VARCHAR(100)
);

-- Промежуточная таблица для связи many-to-many
CREATE TABLE student_courses (
    student_id INTEGER REFERENCES students(id),
    course_id INTEGER REFERENCES courses(id),
    enrolled_at DATE,
    PRIMARY KEY (student_id, course_id)
);
```

## Нормализация данных

Нормализация — процесс организации данных для минимизации избыточности и зависимостей.

### Первая нормальная форма (1NF)
- Каждая ячейка содержит атомарное (неделимое) значение
- Нет повторяющихся групп столбцов

### Вторая нормальная форма (2NF)
- Соответствует 1NF
- Все неключевые атрибуты полностью зависят от первичного ключа

### Третья нормальная форма (3NF)
- Соответствует 2NF
- Нет транзитивных зависимостей между неключевыми атрибутами

```sql
-- Пример ПЛОХОЙ структуры (не 3NF):
-- user_id | user_name | city | country
-- country зависит от city, а не напрямую от user_id

-- Правильная структура (3NF):
CREATE TABLE countries (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE cities (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    country_id INTEGER REFERENCES countries(id)
);

CREATE TABLE users (
    id SERIAL PRIMARY KEY,
    name VARCHAR(100),
    city_id INTEGER REFERENCES cities(id)
);
```

## ACID-свойства

Реляционные базы данных гарантируют ACID-свойства транзакций:

| Свойство | Описание |
|----------|----------|
| **Atomicity** (Атомарность) | Транзакция выполняется полностью или не выполняется вовсе |
| **Consistency** (Согласованность) | База данных переходит из одного согласованного состояния в другое |
| **Isolation** (Изолированность) | Параллельные транзакции не влияют друг на друга |
| **Durability** (Долговечность) | Результаты завершённой транзакции сохраняются даже при сбое системы |

```sql
-- Пример транзакции
BEGIN;
    UPDATE accounts SET balance = balance - 100 WHERE id = 1;
    UPDATE accounts SET balance = balance + 100 WHERE id = 2;
COMMIT;
-- Если что-то пойдёт не так, можно откатить: ROLLBACK;
```

## SQL — язык запросов

SQL (Structured Query Language) — стандартный язык для работы с реляционными базами данных.

### Основные операции (CRUD)

```sql
-- CREATE (Создание)
INSERT INTO users (username, email) VALUES ('john', 'john@example.com');

-- READ (Чтение)
SELECT * FROM users WHERE username = 'john';

-- UPDATE (Обновление)
UPDATE users SET email = 'john_new@example.com' WHERE username = 'john';

-- DELETE (Удаление)
DELETE FROM users WHERE username = 'john';
```

### JOIN — объединение таблиц

```sql
-- Получить все заказы с именами пользователей
SELECT users.username, orders.total_amount, orders.order_date
FROM users
INNER JOIN orders ON users.id = orders.user_id;
```

## Примеры реляционных СУБД

- **PostgreSQL** — мощная open-source СУБД с расширенными возможностями
- **MySQL/MariaDB** — популярная open-source СУБД
- **SQLite** — лёгкая встраиваемая СУБД
- **Oracle Database** — коммерческая enterprise СУБД
- **Microsoft SQL Server** — коммерческая СУБД от Microsoft

## Когда использовать реляционные БД

**Подходит для:**
- Структурированных данных с чёткими связями
- Приложений, требующих ACID-транзакции
- Сложных запросов с объединением данных
- Финансовых и банковских систем
- ERP и CRM систем

**Может не подходить для:**
- Неструктурированных или полуструктурированных данных
- Очень высокой нагрузки на запись (миллионы записей в секунду)
- Горизонтального масштабирования на сотни серверов
- Данных с часто меняющейся схемой

---
[prev: 07-git-patch](../../03-version-control-systems/15-advanced-git/07-git-patch.md) | [next: 02-rdbms-benefits-limitations](./02-rdbms-benefits-limitations.md)