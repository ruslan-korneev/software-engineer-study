# pg_upgrade - Обновление PostgreSQL

## Что такое pg_upgrade

**pg_upgrade** — это утилита PostgreSQL, позволяющая обновить кластер базы данных с одной мажорной версии PostgreSQL на другую без полного дампа и восстановления данных. Это значительно ускоряет процесс миграции, особенно для больших баз данных.

## Принцип работы

pg_upgrade работает путем копирования или связывания (hard link) файлов данных из старого кластера в новый, а затем обновляет системный каталог PostgreSQL для соответствия новой версии. Это возможно, потому что формат хранения данных в таблицах обычно не меняется между мажорными версиями.

### Два режима работы

1. **Copy mode (по умолчанию)** — файлы данных копируются в новый кластер
2. **Link mode (`--link`)** — создаются жесткие ссылки на файлы старого кластера (быстрее, но старый кластер становится непригодным)

## Подготовка к обновлению

### 1. Проверка совместимости

```bash
# Проверка текущей версии
psql -c "SELECT version();"

# Проверка установленных расширений
psql -c "SELECT * FROM pg_extension;"
```

### 2. Создание резервной копии

```bash
# Полный дамп базы данных
pg_dumpall -U postgres > full_backup.sql

# Или с использованием pg_basebackup для физической копии
pg_basebackup -D /backup/pg_basebackup -Ft -z -P
```

### 3. Установка новой версии PostgreSQL

```bash
# Ubuntu/Debian
sudo apt-get install postgresql-16

# CentOS/RHEL
sudo yum install postgresql16-server

# macOS (Homebrew)
brew install postgresql@16
```

### 4. Инициализация нового кластера

```bash
# Инициализация нового кластера
/usr/lib/postgresql/16/bin/initdb -D /var/lib/postgresql/16/main

# Или на CentOS/RHEL
/usr/pgsql-16/bin/initdb -D /var/lib/pgsql/16/data
```

## Процесс обновления

### Основные шаги

```bash
# 1. Остановить старый кластер
sudo systemctl stop postgresql@14-main

# 2. Остановить новый кластер (если запущен)
sudo systemctl stop postgresql@16-main

# 3. Запустить pg_upgrade
sudo -u postgres /usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/var/lib/postgresql/14/main \
    --new-datadir=/var/lib/postgresql/16/main \
    --old-bindir=/usr/lib/postgresql/14/bin \
    --new-bindir=/usr/lib/postgresql/16/bin \
    --check  # Сначала проверка без изменений

# 4. Если проверка прошла успешно, запустить без --check
sudo -u postgres /usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/var/lib/postgresql/14/main \
    --new-datadir=/var/lib/postgresql/16/main \
    --old-bindir=/usr/lib/postgresql/14/bin \
    --new-bindir=/usr/lib/postgresql/16/bin
```

### Использование режима Link (быстрый режим)

```bash
sudo -u postgres /usr/lib/postgresql/16/bin/pg_upgrade \
    --old-datadir=/var/lib/postgresql/14/main \
    --new-datadir=/var/lib/postgresql/16/main \
    --old-bindir=/usr/lib/postgresql/14/bin \
    --new-bindir=/usr/lib/postgresql/16/bin \
    --link
```

## Параметры pg_upgrade

| Параметр | Описание |
|----------|----------|
| `-d, --old-datadir` | Путь к старому каталогу данных |
| `-D, --new-datadir` | Путь к новому каталогу данных |
| `-b, --old-bindir` | Путь к исполняемым файлам старой версии |
| `-B, --new-bindir` | Путь к исполняемым файлам новой версии |
| `-c, --check` | Только проверка без выполнения обновления |
| `-k, --link` | Использовать жесткие ссылки вместо копирования |
| `-j, --jobs` | Количество параллельных процессов |
| `-p, --old-port` | Порт старого кластера |
| `-P, --new-port` | Порт нового кластера |
| `-r, --retain` | Сохранить SQL и лог-файлы после успешного обновления |
| `-v, --verbose` | Подробный вывод |

## Действия после обновления

### 1. Обновление конфигурации

```bash
# Скопировать настройки из старого postgresql.conf
# Обратить внимание на изменения параметров в новой версии

# Скопировать pg_hba.conf
cp /var/lib/postgresql/14/main/pg_hba.conf \
   /var/lib/postgresql/16/main/pg_hba.conf
```

### 2. Запуск нового кластера

```bash
sudo systemctl start postgresql@16-main
```

### 3. Обновление статистики

pg_upgrade создает скрипт для обновления статистики:

```bash
# Запустить скрипт анализа (создается pg_upgrade)
./analyze_new_cluster.sh

# Или вручную
vacuumdb --all --analyze-in-stages
```

### 4. Удаление старого кластера

```bash
# Если все работает корректно
./delete_old_cluster.sh

# Или вручную
rm -rf /var/lib/postgresql/14/main
```

## Обработка расширений

### Проверка совместимости расширений

```bash
# Список установленных расширений
psql -c "SELECT extname, extversion FROM pg_extension;"

# Проверить наличие расширений для новой версии
apt-cache search postgresql-16 | grep extension
```

### Типичные проблемы с расширениями

```sql
-- После обновления может потребоваться
ALTER EXTENSION postgis UPDATE;
ALTER EXTENSION pg_stat_statements UPDATE;
```

## Миграция с минимальным простоем

### Использование логической репликации

```sql
-- На старом сервере (publisher)
CREATE PUBLICATION my_pub FOR ALL TABLES;

-- На новом сервере (subscriber)
CREATE SUBSCRIPTION my_sub
    CONNECTION 'host=old_server dbname=mydb user=replication_user'
    PUBLICATION my_pub;
```

### Этапы миграции с минимальным простоем

1. Настроить логическую репликацию старый -> новый
2. Дождаться синхронизации
3. Переключить приложение на новый сервер
4. Отключить репликацию

## Best Practices

### Подготовка

1. **Всегда делайте полную резервную копию** перед обновлением
2. **Тестируйте обновление** на копии продакшен-данных
3. **Проверьте совместимость** всех используемых расширений
4. **Читайте release notes** целевой версии PostgreSQL
5. **Планируйте окно простоя** (для классического pg_upgrade)

### Во время обновления

```bash
# Используйте флаг --check первым
pg_upgrade --check ...

# Используйте параллельное выполнение для больших баз
pg_upgrade --jobs 4 ...

# Записывайте логи
pg_upgrade ... 2>&1 | tee pg_upgrade.log
```

### После обновления

1. **Проверьте работу приложений**
2. **Запустите тесты** на обновленной базе
3. **Обновите статистику** через ANALYZE
4. **Мониторьте производительность** первые дни после обновления

## Типичные ошибки и решения

### Ошибка: "database ... is not compatible"

```bash
# Причина: неправильная кодировка или locale
# Решение: создать новый кластер с правильными параметрами
initdb -D /new/data --encoding=UTF8 --locale=en_US.UTF-8
```

### Ошибка: "could not load library"

```bash
# Причина: расширение не установлено для новой версии
# Решение: установить пакет расширения
apt-get install postgresql-16-postgis-3
```

### Ошибка: "tablespace ... is not empty"

```bash
# Причина: остались файлы от предыдущих попыток
# Решение: очистить новый каталог данных
rm -rf /var/lib/postgresql/16/main/*
initdb -D /var/lib/postgresql/16/main
```

### Ошибка при использовании --link

```bash
# Причина: файловые системы на разных устройствах
# Решение: использовать режим копирования (без --link)
# или убедиться, что оба каталога на одной файловой системе
```

## Сравнение методов обновления

| Метод | Простой | Объем данных | Скорость |
|-------|---------|--------------|----------|
| pg_dump/pg_restore | Да | Любой | Медленно |
| pg_upgrade (copy) | Да | Большой | Быстро |
| pg_upgrade (link) | Да | Большой | Очень быстро |
| Логическая репликация | Минимальный | Любой | Зависит от нагрузки |

## Полезные ссылки

- [Официальная документация pg_upgrade](https://www.postgresql.org/docs/current/pgupgrade.html)
- [PostgreSQL Release Notes](https://www.postgresql.org/docs/release/)
- [PostgreSQL Upgrade Best Practices](https://wiki.postgresql.org/wiki/Upgrade)
