# Управление PostgreSQL с помощью pg_ctlcluster

## Введение

**pg_ctlcluster** — это утилита, специфичная для дистрибутивов Debian и Ubuntu, которая предоставляет удобный интерфейс для управления несколькими кластерами (инстансами) PostgreSQL на одной системе. Она является частью пакета `postgresql-common` и служит обёрткой над pg_ctl с дополнительной функциональностью.

## Концепция кластеров в Debian/Ubuntu

В отличие от стандартной установки PostgreSQL, где обычно один сервер = один кластер, система пакетов Debian позволяет:
- Запускать несколько версий PostgreSQL одновременно
- Иметь несколько кластеров одной версии
- Автоматически управлять портами и директориями

### Структура именования

```
postgresql-VERSION-CLUSTERNAME

Примеры:
- postgresql-15-main     (основной кластер версии 15)
- postgresql-15-test     (тестовый кластер версии 15)
- postgresql-14-legacy   (старый кластер версии 14)
```

### Структура директорий

```
/etc/postgresql/
├── 15/
│   ├── main/
│   │   ├── postgresql.conf
│   │   ├── pg_hba.conf
│   │   └── pg_ident.conf
│   └── test/
│       └── ...
└── 14/
    └── main/
        └── ...

/var/lib/postgresql/
├── 15/
│   ├── main/
│   └── test/
└── 14/
    └── main/

/var/log/postgresql/
├── postgresql-15-main.log
├── postgresql-15-test.log
└── postgresql-14-main.log
```

## Основные команды

### Синтаксис

```bash
pg_ctlcluster VERSION CLUSTERNAME ACTION [OPTIONS]
```

### Запуск кластера

```bash
# Запуск кластера main версии 15
pg_ctlcluster 15 main start

# Запуск всех кластеров версии 15
pg_ctlcluster 15 '*' start

# Запуск с логированием
pg_ctlcluster 15 main start --foreground
```

### Остановка кластера

```bash
# Обычная остановка (smart)
pg_ctlcluster 15 main stop

# Быстрая остановка (fast) — рекомендуется
pg_ctlcluster 15 main stop --fast

# Немедленная остановка (immediate)
pg_ctlcluster 15 main stop --immediate

# Принудительная остановка с SIGKILL
pg_ctlcluster 15 main stop --force
```

### Перезапуск кластера

```bash
# Обычный перезапуск
pg_ctlcluster 15 main restart

# Быстрый перезапуск
pg_ctlcluster 15 main restart --fast
```

### Перезагрузка конфигурации

```bash
# Перечитать конфигурацию без перезапуска
pg_ctlcluster 15 main reload
```

### Проверка статуса

```bash
# Статус конкретного кластера
pg_ctlcluster 15 main status

# Статус всех кластеров
pg_lsclusters
```

### Пример вывода pg_lsclusters

```
Ver Cluster Port Status Owner    Data directory              Log file
14  main    5433 online postgres /var/lib/postgresql/14/main /var/log/postgresql/postgresql-14-main.log
15  main    5432 online postgres /var/lib/postgresql/15/main /var/log/postgresql/postgresql-15-main.log
15  test    5434 down   postgres /var/lib/postgresql/15/test /var/log/postgresql/postgresql-15-test.log
```

## Создание и удаление кластеров

### Создание нового кластера

```bash
# Создание кластера с настройками по умолчанию
sudo pg_createcluster 15 newcluster

# Создание с указанием порта
sudo pg_createcluster 15 newcluster --port=5555

# Создание с указанием директории данных
sudo pg_createcluster 15 newcluster --datadir=/custom/path/data

# Создание с указанием локали
sudo pg_createcluster 15 newcluster --locale=ru_RU.UTF-8

# Создание без автозапуска
sudo pg_createcluster 15 newcluster --start-conf=manual

# Создание и немедленный запуск
sudo pg_createcluster 15 newcluster --start
```

### Параметры pg_createcluster

```bash
# Полный синтаксис
pg_createcluster [OPTIONS] VERSION CLUSTERNAME

# Важные опции:
--port=PORT           # Порт для прослушивания
--datadir=DIR         # Директория данных
--socketdir=DIR       # Директория для Unix socket
--locale=LOCALE       # Локаль базы данных
--encoding=ENCODING   # Кодировка (UTF8)
--start-conf=MODE     # auto|manual (автозапуск)
--start               # Запустить после создания
--pgoption KEY=VALUE  # Параметры postgresql.conf
```

### Удаление кластера

```bash
# Остановка и удаление кластера
sudo pg_dropcluster 15 newcluster

# Принудительное удаление (с остановкой)
sudo pg_dropcluster 15 newcluster --stop
```

## Обновление кластеров

### pg_upgradecluster

```bash
# Обновление кластера с версии 14 на 15
sudo pg_upgradecluster 14 main

# Обновление на конкретную версию
sudo pg_upgradecluster 14 main 15

# Обновление с указанием нового имени
sudo pg_upgradecluster 14 main 15 upgraded

# Сохранение старого кластера
sudo pg_upgradecluster --keep 14 main
```

### Процесс обновления

1. Создаётся новый кластер целевой версии
2. Данные копируются/переносятся
3. Старый кластер останавливается
4. Новый кластер настраивается на тот же порт
5. Старый кластер переименовывается (имя + `_upgr`)

```bash
# После обновления
pg_lsclusters
# Ver Cluster    Port Status Owner    Data directory
# 14  main_upgr  5433 down   postgres /var/lib/postgresql/14/main
# 15  main       5432 online postgres /var/lib/postgresql/15/main
```

## Конфигурация автозапуска

### Файл start.conf

Каждый кластер имеет файл `/etc/postgresql/VERSION/CLUSTER/start.conf`:

```bash
# Автоматический запуск при загрузке системы
auto

# Ручной запуск (не запускается автоматически)
manual

# Отключен (pg_ctlcluster откажется запускать)
disabled
```

### Изменение режима автозапуска

```bash
# Включить автозапуск
echo "auto" | sudo tee /etc/postgresql/15/main/start.conf

# Отключить автозапуск
echo "manual" | sudo tee /etc/postgresql/15/main/start.conf

# Полностью отключить кластер
echo "disabled" | sudo tee /etc/postgresql/15/main/start.conf
```

## Интеграция с systemd

pg_ctlcluster интегрируется с systemd через службы:

```bash
# Эквивалентные команды
pg_ctlcluster 15 main start
sudo systemctl start postgresql@15-main

pg_ctlcluster 15 main stop
sudo systemctl stop postgresql@15-main

# Статус через systemd
sudo systemctl status postgresql@15-main
```

### Автозапуск через systemd

```bash
# Включить автозапуск для кластера
sudo systemctl enable postgresql@15-main

# Отключить автозапуск
sudo systemctl disable postgresql@15-main
```

## Практические сценарии

### Создание тестового окружения

```bash
#!/bin/bash

# Создание тестового кластера
sudo pg_createcluster 15 test \
    --port=5555 \
    --start-conf=manual \
    --pgoption="log_statement=all" \
    --pgoption="log_connections=on" \
    --start

# Создание тестовой базы данных
sudo -u postgres createdb -p 5555 testdb

# Создание тестового пользователя
sudo -u postgres psql -p 5555 -c "CREATE USER testuser WITH PASSWORD 'test123';"
sudo -u postgres psql -p 5555 -c "GRANT ALL ON DATABASE testdb TO testuser;"

echo "Test cluster ready on port 5555"
```

### Миграция на новую версию

```bash
#!/bin/bash

OLD_VERSION=14
NEW_VERSION=15
CLUSTER=main

# Проверка наличия новой версии
if ! dpkg -l postgresql-$NEW_VERSION > /dev/null 2>&1; then
    echo "Installing PostgreSQL $NEW_VERSION..."
    sudo apt-get update
    sudo apt-get install -y postgresql-$NEW_VERSION
fi

# Проверка текущего состояния
pg_lsclusters

# Обновление с сохранением старого кластера
sudo pg_upgradecluster --keep $OLD_VERSION $CLUSTER

# Проверка результата
pg_lsclusters

# Тестирование нового кластера
sudo -u postgres psql -p 5432 -c "SELECT version();"

echo "Upgrade complete. Old cluster renamed to ${CLUSTER}_upgr"
echo "After verification, remove old cluster with:"
echo "sudo pg_dropcluster $OLD_VERSION ${CLUSTER}_upgr"
```

### Скрипт резервного копирования всех кластеров

```bash
#!/bin/bash

BACKUP_DIR="/var/backups/postgresql"
DATE=$(date +%Y%m%d_%H%M%S)

mkdir -p "$BACKUP_DIR"

# Получение списка всех кластеров
pg_lsclusters -h | while read ver cluster port status owner datadir logfile; do
    if [ "$status" = "online" ]; then
        echo "Backing up cluster $ver/$cluster..."

        BACKUP_FILE="$BACKUP_DIR/${ver}_${cluster}_${DATE}.sql.gz"

        sudo -u postgres pg_dumpall -p "$port" | gzip > "$BACKUP_FILE"

        echo "Backup saved to $BACKUP_FILE"
    fi
done

# Удаление старых бэкапов (старше 7 дней)
find "$BACKUP_DIR" -name "*.sql.gz" -mtime +7 -delete
```

### Настройка кластера для production

```bash
#!/bin/bash

VERSION=15
CLUSTER=production

# Создание production кластера
sudo pg_createcluster $VERSION $CLUSTER \
    --port=5432 \
    --locale=ru_RU.UTF-8 \
    --encoding=UTF8 \
    --start-conf=auto

# Настройка postgresql.conf
CONF_DIR="/etc/postgresql/$VERSION/$CLUSTER"

sudo tee -a "$CONF_DIR/postgresql.conf" << EOF

# Production settings
shared_buffers = 2GB
effective_cache_size = 6GB
maintenance_work_mem = 512MB
checkpoint_completion_target = 0.9
wal_buffers = 64MB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
min_wal_size = 1GB
max_wal_size = 4GB
max_worker_processes = 4
max_parallel_workers_per_gather = 2
max_parallel_workers = 4
max_parallel_maintenance_workers = 2

# Logging
log_destination = 'csvlog'
logging_collector = on
log_directory = '/var/log/postgresql'
log_filename = 'postgresql-$VERSION-$CLUSTER-%Y-%m-%d_%H%M%S.log'
log_statement = 'ddl'
log_min_duration_statement = 1000
EOF

# Запуск кластера
pg_ctlcluster $VERSION $CLUSTER start
```

## Полезные команды

### pg_lsclusters — список всех кластеров

```bash
# Подробный вывод
pg_lsclusters

# Только версии и имена
pg_lsclusters -h | awk '{print $1, $2}'

# Только работающие кластеры
pg_lsclusters | grep online
```

### pg_conftool — управление конфигурацией

```bash
# Чтение параметра
pg_conftool 15 main show shared_buffers

# Установка параметра
sudo pg_conftool 15 main set shared_buffers 2GB

# Удаление параметра (сброс к default)
sudo pg_conftool 15 main remove shared_buffers

# Показать все изменённые параметры
pg_conftool 15 main show all
```

### pg_renamecluster — переименование кластера

```bash
# Переименование кластера
sudo pg_renamecluster 15 main production
```

## Best Practices

### 1. Используйте описательные имена кластеров

```bash
# Хорошо
pg_createcluster 15 production
pg_createcluster 15 staging
pg_createcluster 15 development

# Плохо
pg_createcluster 15 cluster1
pg_createcluster 15 cluster2
```

### 2. Разделяйте порты логически

```bash
# Production
5432 - основной production
5433 - production replica

# Development
5440 - development
5441 - testing
```

### 3. Настраивайте start.conf правильно

```bash
# Production кластеры
echo "auto" | sudo tee /etc/postgresql/15/production/start.conf

# Development кластеры
echo "manual" | sudo tee /etc/postgresql/15/development/start.conf
```

### 4. Всегда используйте --fast для остановки

```bash
pg_ctlcluster 15 main stop --fast
```

### 5. Проверяйте после обновления

```bash
# После pg_upgradecluster
sudo -u postgres psql -c "SELECT version();"
sudo -u postgres psql -c "\l"  # список баз
```

## Типичные ошибки

### 1. Конфликт портов

**Ошибка:**
```
Error: port 5432 is already used by cluster 14/main
```

**Решение:**
```bash
# Проверить занятые порты
pg_lsclusters

# Использовать другой порт
pg_createcluster 15 newcluster --port=5555
```

### 2. Попытка запуска disabled кластера

**Ошибка:**
```
Error: Cluster is disabled
```

**Решение:**
```bash
echo "manual" | sudo tee /etc/postgresql/15/main/start.conf
pg_ctlcluster 15 main start
```

### 3. Забыть указать версию и имя кластера

```bash
# Неправильно
pg_ctlcluster start

# Правильно
pg_ctlcluster 15 main start
```

### 4. Неправильные права после ручного создания файлов

```bash
# Исправление прав
sudo chown -R postgres:postgres /etc/postgresql/15/main/
sudo chmod 755 /etc/postgresql/15/main/
sudo chmod 640 /etc/postgresql/15/main/*.conf
```

### 5. Обновление без достаточного места на диске

```bash
# Проверить место перед обновлением
df -h /var/lib/postgresql/

# pg_upgradecluster требует ~2x размера данных
```

## Сравнение с другими инструментами

| Возможность | pg_ctlcluster | pg_ctl | systemctl |
|-------------|---------------|--------|-----------|
| Множественные кластеры | Да (встроенно) | Вручную | Да |
| Создание кластеров | pg_createcluster | Вручную | Нет |
| Обновление версий | pg_upgradecluster | Вручную | Нет |
| Управление портами | Автоматически | Вручную | Вручную |
| Кроссплатформенность | Только Debian/Ubuntu | Да | Только Linux |

## Заключение

pg_ctlcluster и сопутствующие инструменты (pg_createcluster, pg_lsclusters, pg_upgradecluster) предоставляют удобный способ управления PostgreSQL в Debian/Ubuntu:

- Простое управление несколькими кластерами и версиями
- Автоматическое распределение портов
- Упрощённое обновление между версиями
- Интеграция с системой пакетов и systemd

Эти инструменты особенно полезны для:
- Серверов с несколькими проектами
- Тестирования разных версий PostgreSQL
- Плавной миграции между версиями
- Разделения production и development окружений
