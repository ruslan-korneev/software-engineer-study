# Написание патчей для PostgreSQL

Создание патчей для PostgreSQL — это способ внести вклад в развитие одной из самых мощных СУБД с открытым исходным кодом.

## Подготовка окружения

### Клонирование репозитория

```bash
# Официальный репозиторий
git clone git://git.postgresql.org/git/postgresql.git
cd postgresql

# Или через GitHub mirror
git clone https://github.com/postgres/postgres.git
```

### Зависимости для сборки

**Ubuntu/Debian:**
```bash
sudo apt-get install build-essential libreadline-dev zlib1g-dev \
    flex bison libxml2-dev libxslt1-dev libssl-dev libsystemd-dev \
    pkg-config docbook-xsl libicu-dev
```

**macOS:**
```bash
brew install readline openssl icu4c pkg-config docbook-xsl
```

**Fedora/RHEL:**
```bash
sudo dnf install gcc make readline-devel zlib-devel flex bison \
    libxml2-devel libxslt-devel openssl-devel systemd-devel \
    docbook-style-xsl libicu-devel
```

### Конфигурация и сборка

```bash
# Конфигурация для разработки
./configure \
    --enable-debug \
    --enable-cassert \
    --enable-tap-tests \
    --with-openssl \
    --with-libxml \
    --prefix=$HOME/pg-dev

# Сборка
make -j$(nproc)

# Запуск тестов
make check

# Установка (опционально)
make install
```

### Настройка окружения

```bash
# Добавьте в ~/.bashrc или ~/.zshrc
export PATH=$HOME/pg-dev/bin:$PATH
export PGDATA=$HOME/pg-dev/data
export LD_LIBRARY_PATH=$HOME/pg-dev/lib

# Инициализация кластера для тестов
initdb -D $PGDATA
pg_ctl -D $PGDATA -l logfile start
```

## Coding Style

### Основные правила

PostgreSQL имеет строгий стиль кода:

```c
/*
 * Многострочные комментарии начинаются с /*
 * и каждая строка начинается с *
 */

// Однострочные комментарии через // разрешены в C99

/* Отступы: 4 пробела (не табы!) */
if (condition)
{
    /* Скобки на отдельных строках */
    do_something();
}
else
{
    do_other();
}

/* Однострочные блоки без скобок */
if (condition)
    single_statement();

/* Максимальная длина строки: 79 символов */
```

### Именование

```c
/* Функции: snake_case */
static void
process_query_result(QueryResult *result)
{
    /* Локальные переменные: snake_case */
    int row_count;
    char *error_message;

    /* Макросы: UPPER_CASE */
    #define MAX_BUFFER_SIZE 1024

    /* Типы: CamelCase */
    typedef struct QueryResult
    {
        int     status;
        char   *data;
    } QueryResult;
}
```

### Форматирование указателей

```c
/* Звёздочка у типа */
char *str;
int  *ptr;

/* Выравнивание в структурах */
typedef struct
{
    int         field1;     /* короткие поля */
    char       *field2;     /* указатели */
    Oid         field3;     /* типы PostgreSQL */
} MyStruct;
```

### Инструмент pgindent

```bash
# Автоматическое форматирование
src/tools/pgindent/pgindent src/backend/myfile.c

# Проверка стиля всех изменённых файлов
src/tools/pgindent/pgindent $(git diff --name-only HEAD~1)
```

## Структура исходного кода

```
postgresql/
├── src/
│   ├── backend/           # Серверный код
│   │   ├── access/        # Методы доступа (B-tree, hash, etc.)
│   │   ├── catalog/       # Системные каталоги
│   │   ├── commands/      # SQL команды
│   │   ├── executor/      # Выполнение запросов
│   │   ├── optimizer/     # Оптимизатор запросов
│   │   ├── parser/        # Парсер SQL
│   │   ├── replication/   # Репликация
│   │   ├── storage/       # Управление хранилищем
│   │   └── utils/         # Утилиты
│   ├── bin/               # Клиентские программы (psql, pg_dump)
│   ├── include/           # Заголовочные файлы
│   ├── interfaces/        # Клиентские библиотеки (libpq)
│   ├── pl/                # Процедурные языки
│   └── test/              # Тесты
├── doc/                   # Документация
└── contrib/               # Расширения
```

## Создание патча

### Workflow с git

```bash
# Создайте ветку для работы
git checkout master
git pull origin master
git checkout -b my-feature

# Внесите изменения
# ... редактирование файлов ...

# Коммит изменений
git add -A
git commit -m "Add support for feature X

This patch adds support for feature X by implementing Y.
The approach was discussed on pgsql-hackers in thread [link].

Discussion: https://www.postgresql.org/message-id/..."

# Создание патча
git format-patch master --stdout > my-feature-v1.patch
```

### Формат коммит-сообщения

```
Краткое описание (до 72 символов)

Детальное описание изменений. Объясните ЧТО изменилось
и ПОЧЕМУ это нужно. Не описывайте КАК — это видно из кода.

Перенос строки на 72 символе для удобства чтения в
терминале.

Если патч обсуждался в рассылке, добавьте:
Discussion: https://www.postgresql.org/message-id/...

Если это исправление бага:
Reported-by: User Name <email@example.com>
Bug: #12345

Backpatch-through: 14 (если нужен бэкпорт)
```

### Версионирование патчей

```bash
# Первая версия
my-feature-v1.patch

# После ревью — новая версия
git rebase -i master  # исправления
git format-patch master --stdout > my-feature-v2.patch

# Если несколько патчей в серии
git format-patch master -o patches/
# Получите: 0001-first-change.patch, 0002-second-change.patch
```

## Добавление тестов

### Регрессионные тесты

```sql
-- src/test/regress/sql/my_test.sql

-- Тест новой функции
SELECT my_new_function(1, 2, 3);

-- Ожидаемые ошибки
SELECT my_new_function(NULL);  -- должна быть ошибка

-- Добавьте в src/test/regress/expected/my_test.out
-- ожидаемый вывод
```

```makefile
# Добавьте тест в src/test/regress/parallel_schedule
test: my_test
```

### TAP тесты

```perl
# src/test/recovery/t/099_my_test.pl
use strict;
use warnings;
use PostgreSQL::Test::Cluster;
use PostgreSQL::Test::Utils;
use Test::More;

my $node = PostgreSQL::Test::Cluster->new('primary');
$node->init;
$node->start;

# Тест
my $result = $node->safe_psql('postgres', 'SELECT 1');
is($result, '1', 'basic query works');

$node->stop;
done_testing();
```

### Изоляционные тесты

```
# src/test/isolation/specs/my_isolation_test.spec

setup
{
    CREATE TABLE test (id int PRIMARY KEY, value int);
    INSERT INTO test VALUES (1, 0);
}

teardown
{
    DROP TABLE test;
}

session s1
step s1_update { UPDATE test SET value = value + 1 WHERE id = 1; }

session s2
step s2_update { UPDATE test SET value = value + 1 WHERE id = 1; }

permutation s1_update s2_update
```

## Отправка патча

### Подготовка

1. Убедитесь, что все тесты проходят:
```bash
make check-world
```

2. Проверьте стиль кода:
```bash
src/tools/pgindent/pgindent $(git diff --name-only master)
```

3. Обновите документацию (если нужно)

### Отправка в pgsql-hackers

```
To: pgsql-hackers@lists.postgresql.org
Subject: [PATCH v1] Add support for feature X

Hi hackers,

This patch adds support for feature X.

== Motivation ==
Explain why this feature is needed.

== Implementation ==
Brief description of the approach.

== Testing ==
How the patch was tested.

== Open Questions ==
Any design decisions that need input.

Patch attached.

Thanks,
Your Name
```

### Регистрация в CommitFest

1. Перейдите на https://commitfest.postgresql.org/
2. Войдите через PostgreSQL Community Account
3. Нажмите "New Patch"
4. Заполните форму:
   - Name: Название патча
   - Topic: Категория (например, SQL Commands)
   - Status: Needs review
   - Message-ID: ID письма в рассылке

## Процесс коммита

### Этапы

```
1. Отправка патча → pgsql-hackers
2. Обсуждение и ревью → итерации патча
3. Статус "Ready for Committer"
4. Коммиттер проверяет и коммитит
5. Патч включён в master
```

### Реагирование на ревью

```bash
# После получения комментариев
git checkout my-feature

# Внесите исправления
# ... редактирование ...

# Новая версия патча
git add -A
git commit --amend  # или новый коммит + rebase
git format-patch master --stdout > my-feature-v2.patch
```

```
Subject: Re: [PATCH v1] Add support for feature X

Thanks for the review!

> Comment about issue 1
Fixed in v2.

> Comment about issue 2
I decided to keep the original approach because...

Attached is v2 of the patch.

Changes from v1:
- Fixed issue 1
- Added test for edge case
- Updated documentation
```

## Best Practices

### Для начинающих

1. **Начните с малого:**
   - Исправление опечаток в документации
   - Добавление тестов
   - Исправление небольших багов

2. **Изучите код:**
   - Читайте существующий код
   - Изучайте историю коммитов
   - Смотрите как делают другие

3. **Участвуйте в обсуждениях:**
   - Следите за pgsql-hackers
   - Задавайте вопросы
   - Ревьюйте патчи других

### Общие советы

- **Один патч — одна задача:** не смешивайте несколько изменений
- **Маленькие патчи легче ревьюить:** разбивайте большие изменения
- **Документируйте всё:** код без документации не примут
- **Тестируйте edge cases:** покрывайте все сценарии
- **Будьте терпеливы:** процесс ревью занимает время
- **Не принимайте близко к сердцу:** критика направлена на код, не на вас

## Полезные ресурсы

- [PostgreSQL Developer FAQ](https://wiki.postgresql.org/wiki/Developer_FAQ)
- [Submitting a Patch](https://wiki.postgresql.org/wiki/Submitting_a_Patch)
- [Coding Conventions](https://www.postgresql.org/docs/current/source.html)
- [Running Tests](https://wiki.postgresql.org/wiki/Running_tests)
- [PostgreSQL Git Repository](https://git.postgresql.org/)
- [CommitFest](https://commitfest.postgresql.org/)
