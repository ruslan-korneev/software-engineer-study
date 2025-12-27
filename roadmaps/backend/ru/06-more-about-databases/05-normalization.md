# Нормализация

## Что такое нормализация и зачем она нужна

**Нормализация** — это процесс организации данных в реляционной базе данных, направленный на устранение избыточности и обеспечение целостности данных. Нормализация разбивает большие таблицы на меньшие связанные таблицы, уменьшая дублирование информации.

### Цели нормализации

1. **Устранение избыточности данных** — одни и те же данные не хранятся в нескольких местах
2. **Обеспечение целостности данных** — изменения применяются в одном месте
3. **Упрощение поддержки** — структура БД становится логичной и понятной
4. **Экономия дискового пространства** — меньше дублирования = меньше данных
5. **Предотвращение аномалий** — избежание проблем при вставке, обновлении и удалении

---

## Аномалии данных

Аномалии возникают, когда данные организованы неправильно (не нормализованы). Рассмотрим таблицу с проблемами:

```sql
-- Проблемная (ненормализованная) таблица
CREATE TABLE orders_denormalized (
    order_id INT,
    customer_name VARCHAR(100),
    customer_email VARCHAR(100),
    customer_address VARCHAR(200),
    product_name VARCHAR(100),
    product_price DECIMAL(10,2),
    quantity INT
);

INSERT INTO orders_denormalized VALUES
(1, 'Иван Петров', 'ivan@mail.ru', 'Москва, ул. Ленина 1', 'Ноутбук', 50000, 1),
(2, 'Иван Петров', 'ivan@mail.ru', 'Москва, ул. Ленина 1', 'Мышь', 1500, 2),
(3, 'Мария Сидорова', 'maria@gmail.com', 'СПб, Невский 10', 'Ноутбук', 50000, 1);
```

### 1. Аномалия вставки (Insertion Anomaly)

Невозможно добавить данные без наличия других связанных данных.

**Проблема:** Нельзя добавить нового клиента, пока он не сделает заказ. Нельзя добавить новый товар, пока его не закажут.

```sql
-- Хотим добавить нового клиента Алексея, но у него ещё нет заказов
-- Придётся вставлять NULL в поля заказа, что нарушает логику
INSERT INTO orders_denormalized (customer_name, customer_email, customer_address)
VALUES ('Алексей Новый', 'alex@mail.ru', 'Казань, ул. Мира 5');
-- Ошибка или неполные данные!
```

### 2. Аномалия обновления (Update Anomaly)

При изменении данных нужно менять их во многих местах, что приводит к рассогласованности.

**Проблема:** Если Иван Петров сменил email, нужно обновить все его заказы. Если забыть обновить хотя бы одну строку — данные станут противоречивыми.

```sql
-- Иван сменил email, но мы обновили только один заказ
UPDATE orders_denormalized
SET customer_email = 'ivan.new@mail.ru'
WHERE order_id = 1;

-- Теперь у Ивана два разных email в базе — противоречие!
SELECT DISTINCT customer_name, customer_email
FROM orders_denormalized
WHERE customer_name = 'Иван Петров';
-- ivan.new@mail.ru (order_id = 1)
-- ivan@mail.ru (order_id = 2)
```

### 3. Аномалия удаления (Deletion Anomaly)

При удалении данных теряется связанная информация, которую нужно было сохранить.

**Проблема:** Если удалить единственный заказ Марии, мы потеряем всю информацию о ней как о клиенте.

```sql
-- Удаляем заказ Марии
DELETE FROM orders_denormalized WHERE order_id = 3;

-- Теперь мы потеряли информацию о клиенте Мария Сидорова!
-- Её email, адрес — всё пропало
```

---

## Нормальные формы

### Первая нормальная форма (1NF)

**Требования 1NF:**
1. Все значения атрибутов должны быть **атомарными** (неделимыми)
2. Каждая строка должна быть **уникальной** (наличие первичного ключа)
3. Порядок строк и столбцов не имеет значения
4. Нет повторяющихся групп данных

#### Пример нарушения 1NF

```sql
-- НЕ в 1NF: множественные значения в одном поле
CREATE TABLE students_bad (
    student_id INT PRIMARY KEY,
    name VARCHAR(100),
    phone_numbers VARCHAR(200)  -- "123-45-67, 890-12-34" — НЕ атомарно!
);

INSERT INTO students_bad VALUES
(1, 'Иван', '123-45-67, 890-12-34'),
(2, 'Мария', '555-55-55');
```

#### Приведение к 1NF

```sql
-- Вариант 1: Отдельная таблица для телефонов
CREATE TABLE students (
    student_id INT PRIMARY KEY,
    name VARCHAR(100)
);

CREATE TABLE student_phones (
    phone_id INT PRIMARY KEY,
    student_id INT REFERENCES students(student_id),
    phone_number VARCHAR(20)
);

INSERT INTO students VALUES (1, 'Иван'), (2, 'Мария');
INSERT INTO student_phones VALUES
(1, 1, '123-45-67'),
(2, 1, '890-12-34'),
(3, 2, '555-55-55');

-- Теперь каждое значение атомарно!
```

```sql
-- Ещё пример нарушения 1NF: повторяющиеся группы
CREATE TABLE orders_bad (
    order_id INT,
    customer_name VARCHAR(100),
    product1 VARCHAR(100),
    qty1 INT,
    product2 VARCHAR(100),
    qty2 INT,
    product3 VARCHAR(100),
    qty3 INT
);

-- Приведение к 1NF
CREATE TABLE orders (
    order_id INT,
    customer_name VARCHAR(100),
    PRIMARY KEY (order_id)
);

CREATE TABLE order_items (
    order_id INT,
    product VARCHAR(100),
    quantity INT,
    PRIMARY KEY (order_id, product),
    FOREIGN KEY (order_id) REFERENCES orders(order_id)
);
```

---

### Вторая нормальная форма (2NF)

**Требования 2NF:**
1. Таблица должна быть в **1NF**
2. Все неключевые атрибуты должны полностью зависеть от **всего** первичного ключа (не от его части)

2NF актуальна для таблиц с **составным первичным ключом**. Если ключ простой (один столбец), то таблица в 1NF автоматически соответствует 2NF.

#### Пример нарушения 2NF

```sql
-- НЕ в 2NF: частичная зависимость от составного ключа
CREATE TABLE order_details_bad (
    order_id INT,
    product_id INT,
    product_name VARCHAR(100),    -- Зависит только от product_id, не от всего ключа!
    product_category VARCHAR(50), -- Зависит только от product_id!
    quantity INT,                 -- Зависит от всего ключа (order_id, product_id)
    PRIMARY KEY (order_id, product_id)
);

-- product_name и product_category зависят только от product_id
-- Это частичная зависимость — нарушение 2NF
```

#### Приведение к 2NF

```sql
-- Выносим данные о продуктах в отдельную таблицу
CREATE TABLE products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(100),
    product_category VARCHAR(50)
);

CREATE TABLE order_details (
    order_id INT,
    product_id INT,
    quantity INT,
    PRIMARY KEY (order_id, product_id),
    FOREIGN KEY (product_id) REFERENCES products(product_id)
);

-- Вставка данных
INSERT INTO products VALUES
(1, 'Ноутбук', 'Электроника'),
(2, 'Мышь', 'Аксессуары');

INSERT INTO order_details VALUES
(1, 1, 1),
(1, 2, 2),
(2, 1, 1);
```

---

### Третья нормальная форма (3NF)

**Требования 3NF:**
1. Таблица должна быть в **2NF**
2. Все неключевые атрибуты должны зависеть **только от первичного ключа**, а не от других неключевых атрибутов (отсутствие транзитивных зависимостей)

#### Пример нарушения 3NF

```sql
-- НЕ в 3NF: транзитивная зависимость
CREATE TABLE employees_bad (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT,
    department_name VARCHAR(100),  -- Зависит от department_id, а не от employee_id!
    department_head VARCHAR(100)   -- Тоже зависит от department_id!
);

-- Цепочка зависимостей:
-- employee_id -> department_id -> department_name
-- Это транзитивная зависимость!
```

#### Приведение к 3NF

```sql
-- Создаём отдельную таблицу для отделов
CREATE TABLE departments (
    department_id INT PRIMARY KEY,
    department_name VARCHAR(100),
    department_head VARCHAR(100)
);

CREATE TABLE employees (
    employee_id INT PRIMARY KEY,
    employee_name VARCHAR(100),
    department_id INT REFERENCES departments(department_id)
);

-- Вставка данных
INSERT INTO departments VALUES
(1, 'IT', 'Сергей Иванов'),
(2, 'HR', 'Анна Петрова');

INSERT INTO employees VALUES
(1, 'Алексей', 1),
(2, 'Мария', 1),
(3, 'Дмитрий', 2);
```

---

### Нормальная форма Бойса-Кодда (BCNF)

**Требования BCNF:**
1. Таблица должна быть в **3NF**
2. Для каждой функциональной зависимости X → Y, X должен быть **суперключом** (то есть определять все остальные атрибуты)

BCNF — более строгая версия 3NF. Таблица может быть в 3NF, но не в BCNF, когда есть функциональная зависимость от атрибута, который является частью ключа-кандидата, но не суперключом.

#### Пример нарушения BCNF

```sql
-- Сценарий: Преподаватели и предметы
-- Правила:
-- 1. Каждый студент изучает предмет у одного преподавателя
-- 2. Каждый преподаватель ведёт только один предмет
-- 3. Предмет могут вести несколько преподавателей

CREATE TABLE student_courses_bad (
    student_id INT,
    subject VARCHAR(100),
    teacher VARCHAR(100),
    PRIMARY KEY (student_id, subject)
);

-- Ключи-кандидаты: (student_id, subject) и (student_id, teacher)
-- Функциональная зависимость: teacher -> subject
-- teacher не является суперключом — нарушение BCNF!

INSERT INTO student_courses_bad VALUES
(1, 'Математика', 'Иванов'),
(2, 'Математика', 'Иванов'),
(3, 'Физика', 'Петров'),
(1, 'Физика', 'Петров');

-- Проблема: если Иванов начнёт вести другой предмет,
-- нужно обновить все записи
```

#### Приведение к BCNF

```sql
-- Разбиваем на две таблицы
CREATE TABLE teachers (
    teacher VARCHAR(100) PRIMARY KEY,
    subject VARCHAR(100) NOT NULL
);

CREATE TABLE student_teachers (
    student_id INT,
    teacher VARCHAR(100) REFERENCES teachers(teacher),
    PRIMARY KEY (student_id, teacher)
);

INSERT INTO teachers VALUES
('Иванов', 'Математика'),
('Петров', 'Физика');

INSERT INTO student_teachers VALUES
(1, 'Иванов'),
(2, 'Иванов'),
(3, 'Петров'),
(1, 'Петров');

-- Теперь связь teacher -> subject хранится в одном месте
```

---

### Четвёртая нормальная форма (4NF)

**Требования 4NF:**
1. Таблица должна быть в **BCNF**
2. Не должно быть **многозначных зависимостей** (multivalued dependencies), кроме тех, где зависимость определяется суперключом

Многозначная зависимость A →→ B означает, что для каждого значения A существует набор значений B, независимый от других атрибутов.

#### Пример нарушения 4NF

```sql
-- Преподаватель может вести несколько курсов
-- и владеть несколькими языками (независимо друг от друга)
CREATE TABLE teacher_skills_bad (
    teacher VARCHAR(100),
    course VARCHAR(100),
    language VARCHAR(50),
    PRIMARY KEY (teacher, course, language)
);

INSERT INTO teacher_skills_bad VALUES
('Иванов', 'Алгебра', 'Русский'),
('Иванов', 'Алгебра', 'Английский'),
('Иванов', 'Геометрия', 'Русский'),
('Иванов', 'Геометрия', 'Английский');

-- Две независимые многозначные зависимости:
-- teacher →→ course
-- teacher →→ language
-- Избыточность: добавление нового курса требует строк для каждого языка
```

#### Приведение к 4NF

```sql
CREATE TABLE teacher_courses (
    teacher VARCHAR(100),
    course VARCHAR(100),
    PRIMARY KEY (teacher, course)
);

CREATE TABLE teacher_languages (
    teacher VARCHAR(100),
    language VARCHAR(50),
    PRIMARY KEY (teacher, language)
);

INSERT INTO teacher_courses VALUES
('Иванов', 'Алгебра'),
('Иванов', 'Геометрия');

INSERT INTO teacher_languages VALUES
('Иванов', 'Русский'),
('Иванов', 'Английский');

-- Теперь нет избыточности!
```

---

### Пятая нормальная форма (5NF)

**Требования 5NF (Project-Join Normal Form):**
1. Таблица должна быть в **4NF**
2. Не должно быть **зависимостей соединения** (join dependencies), которые не следуют из ключей-кандидатов

5NF применяется в редких случаях, когда данные можно разбить на три или более таблиц без потери информации, и обратное соединение даёт исходную таблицу.

```sql
-- Пример: Поставщик может поставлять детали для определённых проектов
-- Связь трёхсторонняя: поставщик-деталь-проект
-- Если поставщик S поставляет деталь P и работает с проектом J,
-- и деталь P нужна для проекта J,
-- то S поставляет P для J

-- В большинстве практических случаев 5NF не требуется,
-- достаточно BCNF или 4NF
```

---

## Полный пример нормализации

Рассмотрим процесс нормализации от ненормализованной таблицы до 3NF:

### Исходная таблица (UNF — ненормализованная)

```sql
CREATE TABLE library_unnormalized (
    book_id INT,
    title VARCHAR(200),
    authors VARCHAR(500),           -- "Иванов И.И., Петров П.П."
    publisher VARCHAR(100),
    publisher_address VARCHAR(200),
    categories VARCHAR(200),        -- "Программирование, Базы данных"
    borrower_name VARCHAR(100),
    borrower_phone VARCHAR(20),
    borrow_date DATE,
    return_date DATE
);
```

### Шаг 1: Приведение к 1NF

```sql
-- Устраняем неатомарные значения
CREATE TABLE books_1nf (
    book_id INT,
    title VARCHAR(200),
    publisher VARCHAR(100),
    publisher_address VARCHAR(200),
    borrower_name VARCHAR(100),
    borrower_phone VARCHAR(20),
    borrow_date DATE,
    return_date DATE,
    PRIMARY KEY (book_id)
);

CREATE TABLE book_authors_1nf (
    book_id INT,
    author_name VARCHAR(100),
    PRIMARY KEY (book_id, author_name),
    FOREIGN KEY (book_id) REFERENCES books_1nf(book_id)
);

CREATE TABLE book_categories_1nf (
    book_id INT,
    category VARCHAR(100),
    PRIMARY KEY (book_id, category),
    FOREIGN KEY (book_id) REFERENCES books_1nf(book_id)
);
```

### Шаг 2: Приведение к 2NF

```sql
-- Выносим информацию о заёмщиках (частичная зависимость от book_id)
CREATE TABLE books_2nf (
    book_id INT PRIMARY KEY,
    title VARCHAR(200),
    publisher VARCHAR(100),
    publisher_address VARCHAR(200)
);

CREATE TABLE borrowers (
    borrower_id INT PRIMARY KEY,
    borrower_name VARCHAR(100),
    borrower_phone VARCHAR(20)
);

CREATE TABLE book_loans (
    loan_id INT PRIMARY KEY,
    book_id INT REFERENCES books_2nf(book_id),
    borrower_id INT REFERENCES borrowers(borrower_id),
    borrow_date DATE,
    return_date DATE
);

-- Таблицы авторов и категорий остаются как в 1NF
```

### Шаг 3: Приведение к 3NF

```sql
-- Устраняем транзитивную зависимость publisher -> publisher_address
CREATE TABLE publishers (
    publisher_id INT PRIMARY KEY,
    publisher_name VARCHAR(100),
    publisher_address VARCHAR(200)
);

CREATE TABLE books_3nf (
    book_id INT PRIMARY KEY,
    title VARCHAR(200),
    publisher_id INT REFERENCES publishers(publisher_id)
);

-- Также выносим авторов в отдельную сущность
CREATE TABLE authors (
    author_id INT PRIMARY KEY,
    author_name VARCHAR(100)
);

CREATE TABLE book_authors (
    book_id INT REFERENCES books_3nf(book_id),
    author_id INT REFERENCES authors(author_id),
    PRIMARY KEY (book_id, author_id)
);

CREATE TABLE categories (
    category_id INT PRIMARY KEY,
    category_name VARCHAR(100)
);

CREATE TABLE book_categories (
    book_id INT REFERENCES books_3nf(book_id),
    category_id INT REFERENCES categories(category_id),
    PRIMARY KEY (book_id, category_id)
);
```

### Финальная схема в 3NF

```
publishers (publisher_id, publisher_name, publisher_address)
authors (author_id, author_name)
categories (category_id, category_name)
books (book_id, title, publisher_id)
book_authors (book_id, author_id)
book_categories (book_id, category_id)
borrowers (borrower_id, borrower_name, borrower_phone)
book_loans (loan_id, book_id, borrower_id, borrow_date, return_date)
```

---

## Денормализация

**Денормализация** — это намеренное введение избыточности в базу данных для повышения производительности чтения.

### Когда применять денормализацию

1. **Частые JOIN-запросы** замедляют работу
2. **Отчёты и аналитика** требуют агрегированных данных
3. **Кэширование вычислений** (хранение рассчитанных значений)
4. **Архивные/исторические данные** не изменяются
5. **Read-heavy приложения** (чтение намного чаще записи)

### Примеры денормализации

#### 1. Добавление избыточных полей

```sql
-- Нормализованная версия
SELECT o.order_id, SUM(oi.quantity * p.price) as total
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY o.order_id;

-- Денормализованная версия: храним total в таблице заказов
ALTER TABLE orders ADD COLUMN total_amount DECIMAL(10,2);

-- При добавлении/изменении товаров обновляем сумму
UPDATE orders SET total_amount = (
    SELECT SUM(oi.quantity * p.price)
    FROM order_items oi
    JOIN products p ON oi.product_id = p.product_id
    WHERE oi.order_id = orders.order_id
);
```

#### 2. Дублирование данных из связанных таблиц

```sql
-- Нормализованная версия требует JOIN
SELECT o.order_id, c.customer_name
FROM orders o
JOIN customers c ON o.customer_id = c.customer_id;

-- Денормализованная версия: копируем имя в заказ
ALTER TABLE orders ADD COLUMN customer_name VARCHAR(100);

-- Плюс: быстрое чтение без JOIN
-- Минус: при смене имени клиента нужно обновлять все заказы
```

#### 3. Материализованные представления

```sql
-- Создаём материализованное представление для отчётов
CREATE MATERIALIZED VIEW sales_summary AS
SELECT
    DATE_TRUNC('month', order_date) as month,
    product_category,
    SUM(quantity) as total_quantity,
    SUM(amount) as total_amount,
    COUNT(*) as order_count
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
GROUP BY DATE_TRUNC('month', order_date), product_category;

-- Периодически обновляем
REFRESH MATERIALIZED VIEW sales_summary;
```

#### 4. Хранение исторических снимков

```sql
-- Вместо JOIN с текущими ценами сохраняем цену на момент заказа
CREATE TABLE order_items (
    order_item_id INT PRIMARY KEY,
    order_id INT,
    product_id INT,
    quantity INT,
    unit_price DECIMAL(10,2),  -- Цена на момент заказа (денормализация)
    product_name VARCHAR(100)  -- Название на момент заказа (денормализация)
);
```

---

## Компромиссы: нормализация vs производительность

| Аспект | Нормализация | Денормализация |
|--------|--------------|----------------|
| **Дисковое пространство** | Меньше (нет дублирования) | Больше (есть дублирование) |
| **Скорость записи** | Быстрее (меньше обновлений) | Медленнее (обновление в нескольких местах) |
| **Скорость чтения** | Медленнее (много JOIN) | Быстрее (меньше JOIN) |
| **Целостность данных** | Легко поддерживать | Риск рассогласованности |
| **Сложность запросов** | Сложнее (много таблиц) | Проще (меньше таблиц) |
| **Гибкость изменений** | Высокая | Низкая |

### Правило принятия решения

```
                    Записи > Чтения?
                          │
              ┌───────────┴───────────┐
              │ ДА                    │ НЕТ
              ▼                       ▼
      Нормализация              Критична ли
      (3NF/BCNF)                скорость чтения?
                                      │
                          ┌───────────┴───────────┐
                          │ ДА                    │ НЕТ
                          ▼                       ▼
                   Денормализация           Нормализация
                   (с осторожностью)        (3NF)
```

---

## Best Practices

### 1. Начинайте с нормализации

```sql
-- Всегда проектируйте схему в 3NF
-- Денормализуйте только когда есть доказанная проблема производительности
```

### 2. Документируйте денормализацию

```sql
-- Комментарий в схеме
COMMENT ON COLUMN orders.customer_name IS
    'Денормализовано из customers.name для ускорения отчётов.
     Обновляется триггером update_order_customer_name';
```

### 3. Используйте триггеры для поддержания согласованности

```sql
-- Триггер для синхронизации денормализованных данных
CREATE OR REPLACE FUNCTION sync_customer_name()
RETURNS TRIGGER AS $$
BEGIN
    IF TG_OP = 'UPDATE' AND OLD.customer_name != NEW.customer_name THEN
        UPDATE orders
        SET customer_name = NEW.customer_name
        WHERE customer_id = NEW.customer_id;
    END IF;
    RETURN NEW;
END;
$$ LANGUAGE plpgsql;

CREATE TRIGGER customer_name_sync
AFTER UPDATE ON customers
FOR EACH ROW EXECUTE FUNCTION sync_customer_name();
```

### 4. Разделяйте OLTP и OLAP

```sql
-- OLTP база (нормализованная) — для транзакций
-- Основная база приложения

-- OLAP база/хранилище (денормализованная) — для аналитики
-- Data Warehouse с ETL-процессами
```

### 5. Тестируйте производительность

```sql
-- Перед денормализацией измерьте
EXPLAIN ANALYZE
SELECT ... -- ваш медленный запрос

-- После денормализации сравните
EXPLAIN ANALYZE
SELECT ... -- тот же запрос
```

### 6. Уровень нормализации зависит от контекста

- **OLTP-системы**: обычно 3NF или BCNF
- **Справочники/словари**: 3NF
- **Аналитические хранилища**: часто 2NF или денормализованные star/snowflake схемы
- **Кэши и отчёты**: денормализация уместна

---

## Резюме

| Нормальная форма | Основное требование |
|------------------|---------------------|
| **1NF** | Атомарные значения, уникальные строки |
| **2NF** | 1NF + нет частичных зависимостей от составного ключа |
| **3NF** | 2NF + нет транзитивных зависимостей |
| **BCNF** | Каждый детерминант — суперключ |
| **4NF** | BCNF + нет многозначных зависимостей |
| **5NF** | 4NF + нет зависимостей соединения |

### Практические выводы

1. **3NF достаточна** для большинства приложений
2. **BCNF** — когда есть сложные ключи-кандидаты
3. **4NF/5NF** — редко нужны на практике
4. **Денормализация** — осознанный выбор для производительности
5. **Измеряйте**, прежде чем оптимизировать
6. **Документируйте** все отклонения от нормальных форм

```
Нормализация — это про правильность данных
Денормализация — это про скорость доступа
Мастерство — в балансе между ними
```
