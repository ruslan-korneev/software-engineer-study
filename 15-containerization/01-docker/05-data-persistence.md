# Data Persistence в Docker

## Проблема эфемерности контейнеров

Docker контейнеры по своей природе **эфемерны** (ephemeral) — это означает, что все данные, записанные внутри контейнера, существуют только пока контейнер работает. Когда контейнер удаляется, все данные в его writable layer **безвозвратно теряются**.

```bash
# Создаём контейнер и записываем данные
docker run -it --name test-container ubuntu bash
echo "Important data" > /data.txt
exit

# Удаляем контейнер
docker rm test-container

# Данные потеряны навсегда!
```

### Почему это проблема?

- **Базы данных** — потеря данных при обновлении контейнера
- **Логи приложений** — невозможность анализа после перезапуска
- **Загруженные файлы** — пользовательский контент исчезает
- **Конфигурации** — изменения не сохраняются

### Решение: механизмы персистентности

Docker предоставляет три способа сохранения данных:

| Механизм | Расположение | Управление | Использование |
|----------|--------------|------------|---------------|
| **Volumes** | `/var/lib/docker/volumes/` | Docker | Production, базы данных |
| **Bind Mounts** | Любое место на хосте | Пользователь | Разработка, конфигурации |
| **tmpfs Mounts** | Оперативная память | Docker | Секреты, временные данные |

---

## Volumes (Docker тома)

**Volumes** — это рекомендуемый способ хранения данных в Docker. Они полностью управляются Docker и изолированы от файловой системы хоста.

### Создание и управление томами

```bash
# Создание тома
docker volume create my-data

# Создание тома с метками
docker volume create --label project=myapp --label env=production my-app-data

# Список всех томов
docker volume ls

# Подробная информация о томе
docker volume inspect my-data
# Вывод:
# [
#     {
#         "CreatedAt": "2024-01-15T10:30:00Z",
#         "Driver": "local",
#         "Labels": {},
#         "Mountpoint": "/var/lib/docker/volumes/my-data/_data",
#         "Name": "my-data",
#         "Options": {},
#         "Scope": "local"
#     }
# ]

# Удаление тома
docker volume rm my-data

# Удаление всех неиспользуемых томов
docker volume prune

# Удаление с подтверждением
docker volume prune -f
```

### Монтирование томов при запуске контейнера

Есть два синтаксиса: `-v` (короткий) и `--mount` (явный).

#### Синтаксис -v (--volume)

```bash
# Формат: -v имя_тома:путь_в_контейнере[:опции]
docker run -d \
  -v my-data:/var/lib/data \
  --name my-app \
  nginx

# С режимом только для чтения
docker run -d \
  -v my-data:/var/lib/data:ro \
  --name my-app \
  nginx
```

#### Синтаксис --mount (рекомендуемый)

```bash
# Более явный и читаемый синтаксис
docker run -d \
  --mount type=volume,source=my-data,target=/var/lib/data \
  --name my-app \
  nginx

# С дополнительными опциями
docker run -d \
  --mount type=volume,source=my-data,target=/var/lib/data,readonly \
  --name my-app \
  nginx
```

**Различия между -v и --mount:**

| Аспект | -v | --mount |
|--------|-----|---------|
| Синтаксис | Компактный | Явный, key=value |
| Несуществующий том | Создаёт автоматически | Выдаёт ошибку |
| Несуществующий путь хоста | Создаёт директорию | Выдаёт ошибку |
| Рекомендация | Для быстрых команд | Для production и скриптов |

### Named Volumes vs Anonymous Volumes

#### Named Volumes (именованные тома)

```bash
# Явно создаём том с именем
docker volume create postgres-data

# Или автоматически при запуске контейнера
docker run -d \
  -v postgres-data:/var/lib/postgresql/data \
  --name db \
  postgres:15

# Легко найти и управлять
docker volume ls
# DRIVER    VOLUME NAME
# local     postgres-data
```

#### Anonymous Volumes (анонимные тома)

```bash
# Создаётся автоматически с хэшем вместо имени
docker run -d \
  -v /var/lib/postgresql/data \
  --name db \
  postgres:15

# Сложно идентифицировать
docker volume ls
# DRIVER    VOLUME NAME
# local     a1b2c3d4e5f6...

# Часто остаются "сиротами" после удаления контейнера
docker volume prune  # Удаляет анонимные неиспользуемые тома
```

**Рекомендация:** Всегда используйте named volumes в production!

### Где хранятся volumes на хосте

На Linux volumes находятся в:
```
/var/lib/docker/volumes/<volume-name>/_data/
```

```bash
# Посмотреть путь
docker volume inspect my-data --format '{{ .Mountpoint }}'
# /var/lib/docker/volumes/my-data/_data

# Просмотр содержимого (требует sudo на Linux)
sudo ls -la /var/lib/docker/volumes/my-data/_data/
```

**Важно:** Не рекомендуется напрямую модифицировать файлы в этой директории!

На macOS и Windows Docker работает в VM, поэтому путь отличается:
- **macOS**: ~/Library/Containers/com.docker.docker/Data/vms/0/data
- **Windows**: \\wsl$\docker-desktop-data\data\docker\volumes

---

## Bind Mounts

**Bind Mounts** позволяют монтировать любую директорию с хост-машины в контейнер. Изменения видны с обеих сторон мгновенно.

### Синтаксис и использование

```bash
# Синтаксис -v с абсолютным путём
docker run -d \
  -v /home/user/project:/app \
  --name dev-app \
  node:18

# Синтаксис -v с относительным путём ($(pwd))
docker run -d \
  -v $(pwd):/app \
  --name dev-app \
  node:18

# Синтаксис --mount (рекомендуемый)
docker run -d \
  --mount type=bind,source=/home/user/project,target=/app \
  --name dev-app \
  node:18

# Режим только для чтения
docker run -d \
  --mount type=bind,source=/home/user/config,target=/etc/app/config,readonly \
  --name my-app \
  nginx
```

### Когда использовать Bind Mounts

#### 1. Разработка с hot-reload

```bash
# React/Node.js разработка
docker run -d \
  -v $(pwd)/src:/app/src \
  -v $(pwd)/package.json:/app/package.json \
  -p 3000:3000 \
  --name react-dev \
  node:18 npm run dev

# Python разработка
docker run -d \
  -v $(pwd):/app \
  -p 8000:8000 \
  --name django-dev \
  python:3.11 python manage.py runserver 0.0.0.0:8000
```

#### 2. Монтирование конфигурационных файлов

```bash
# Nginx конфигурация
docker run -d \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/ssl:/etc/nginx/ssl:ro \
  -p 80:80 \
  -p 443:443 \
  nginx

# Конфигурация приложения
docker run -d \
  -v $(pwd)/config.yaml:/app/config.yaml:ro \
  my-app
```

#### 3. Доступ к логам для анализа

```bash
# Логи приложения на хосте
docker run -d \
  -v /var/log/myapp:/app/logs \
  --name my-app \
  my-app-image
```

### Проблемы с правами доступа

Одна из главных проблем bind mounts — **несоответствие UID/GID** между хостом и контейнером.

#### Проблема

```bash
# Файлы создаются от root внутри контейнера
docker run -v $(pwd)/data:/data alpine touch /data/file.txt

# На хосте файл принадлежит root
ls -la data/
# -rw-r--r-- 1 root root 0 Jan 15 10:00 file.txt
```

#### Решения

**1. Запуск от определённого пользователя:**
```bash
# Узнаём свой UID/GID
id
# uid=1000(user) gid=1000(user)

# Запуск с указанием пользователя
docker run -v $(pwd)/data:/data \
  --user $(id -u):$(id -g) \
  alpine touch /data/file.txt
```

**2. Использование USER в Dockerfile:**
```dockerfile
FROM node:18

# Создаём пользователя с нужным UID
RUN groupadd -g 1000 appgroup && \
    useradd -u 1000 -g appgroup -m appuser

USER appuser
WORKDIR /app
```

**3. Изменение прав в entrypoint:**
```dockerfile
FROM postgres:15

COPY docker-entrypoint.sh /
RUN chmod +x /docker-entrypoint.sh

ENTRYPOINT ["/docker-entrypoint.sh"]
```

```bash
#!/bin/bash
# docker-entrypoint.sh
chown -R postgres:postgres /var/lib/postgresql/data
exec "$@"
```

---

## tmpfs Mounts

**tmpfs Mounts** хранят данные в оперативной памяти хоста. Данные существуют только пока контейнер работает и **никогда не записываются на диск**.

### Для чего нужны tmpfs

1. **Хранение секретов** — пароли, токены не попадают на диск
2. **Временный кэш** — быстрый доступ к часто используемым данным
3. **Сессии пользователей** — высокая скорость записи/чтения
4. **Тестирование** — данные автоматически очищаются

### Как использовать tmpfs

```bash
# Базовое использование
docker run -d \
  --tmpfs /app/cache \
  --name my-app \
  my-app-image

# С параметрами размера и прав
docker run -d \
  --tmpfs /app/cache:size=100m,mode=1777 \
  --name my-app \
  my-app-image

# Синтаксис --mount
docker run -d \
  --mount type=tmpfs,target=/app/secrets,tmpfs-size=50m,tmpfs-mode=0700 \
  --name secure-app \
  my-app-image
```

### Параметры tmpfs

| Параметр | Описание | Пример |
|----------|----------|--------|
| `size` | Максимальный размер | `size=100m` |
| `mode` | Права доступа (octal) | `mode=1777` |

### Практический пример: секреты

```bash
# Секретный файл в tmpfs
docker run -d \
  --mount type=tmpfs,target=/run/secrets,tmpfs-mode=0700 \
  -e DB_PASSWORD_FILE=/run/secrets/db_password \
  --name secure-app \
  my-app

# Внутри контейнера секрет записывается в tmpfs
# После остановки контейнера — данные полностью стираются
```

---

## Сравнительная таблица: Volumes vs Bind Mounts vs tmpfs

| Характеристика | Volumes | Bind Mounts | tmpfs |
|----------------|---------|-------------|-------|
| **Управление** | Docker | Пользователь | Docker |
| **Расположение** | `/var/lib/docker/volumes/` | Любое место на хосте | RAM |
| **Персистентность** | ✅ Сохраняется | ✅ Сохраняется | ❌ Только в памяти |
| **Переносимость** | Высокая | Низкая (зависит от путей хоста) | Нет |
| **Производительность** | Хорошая | Хорошая (нативная ФС) | Отличная (RAM) |
| **Backup** | Легко (docker volume) | Стандартные инструменты | Невозможно |
| **Безопасность** | Изолирован от хоста | Доступ к хосту | Данные в памяти |
| **Use Case** | Production, БД | Разработка, конфиги | Секреты, кэш |
| **Права доступа** | Управляет Docker | Часто проблемы с UID/GID | Настраивается |

### Когда что использовать?

```
┌─────────────────────────────────────────────────────────────────┐
│                    Выбор типа хранилища                         │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─ Production база данных? ──────────► Volumes                │
│  │                                                              │
│  ├─ Разработка с hot-reload? ─────────► Bind Mounts            │
│  │                                                              │
│  ├─ Конфигурационные файлы? ──────────► Bind Mounts (readonly) │
│  │                                                              │
│  ├─ Секреты и пароли? ────────────────► tmpfs                  │
│  │                                                              │
│  ├─ Shared data между контейнерами? ──► Volumes                │
│  │                                                              │
│  └─ Временный кэш? ───────────────────► tmpfs                  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## Best Practices

### 1. Правильный выбор типа хранилища

```bash
# ❌ Плохо: bind mount для данных БД в production
docker run -d \
  -v /home/user/postgres-data:/var/lib/postgresql/data \
  postgres:15

# ✅ Хорошо: volume для данных БД
docker run -d \
  -v postgres-data:/var/lib/postgresql/data \
  postgres:15
```

### 2. Именуйте volumes осмысленно

```bash
# ❌ Плохо
docker volume create vol1

# ✅ Хорошо
docker volume create postgres-myapp-data
docker volume create redis-sessions-cache
docker volume create elasticsearch-logs-data
```

### 3. Используйте readonly где возможно

```bash
# Конфигурации только для чтения
docker run -d \
  --mount type=bind,source=$(pwd)/config,target=/app/config,readonly \
  --mount type=volume,source=app-data,target=/app/data \
  my-app
```

### 4. Backup и Restore Volumes

#### Backup тома

```bash
# Способ 1: Копирование через временный контейнер
docker run --rm \
  -v postgres-data:/source:ro \
  -v $(pwd)/backup:/backup \
  alpine tar czf /backup/postgres-backup-$(date +%Y%m%d).tar.gz -C /source .

# Способ 2: Использование docker cp
docker run -d --name temp-backup -v postgres-data:/data alpine sleep infinity
docker cp temp-backup:/data ./backup/
docker rm -f temp-backup
```

#### Restore тома

```bash
# Восстановление из архива
docker volume create postgres-data-restored

docker run --rm \
  -v postgres-data-restored:/target \
  -v $(pwd)/backup:/backup:ro \
  alpine tar xzf /backup/postgres-backup-20240115.tar.gz -C /target
```

### 5. Sharing Data между контейнерами

```bash
# Создаём общий том
docker volume create shared-data

# Контейнер-producer записывает данные
docker run -d \
  --name producer \
  -v shared-data:/data \
  alpine sh -c "while true; do date >> /data/log.txt; sleep 5; done"

# Контейнер-consumer читает данные
docker run -d \
  --name consumer \
  -v shared-data:/data:ro \
  alpine tail -f /data/log.txt
```

---

## Практические примеры для баз данных

### PostgreSQL с Volume

```bash
# Создаём volume для данных
docker volume create postgres-data

# Запуск PostgreSQL
docker run -d \
  --name postgres-db \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydb \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15

# Проверка персистентности
docker exec -it postgres-db psql -U myuser -d mydb -c "CREATE TABLE test (id SERIAL PRIMARY KEY, name TEXT);"
docker exec -it postgres-db psql -U myuser -d mydb -c "INSERT INTO test (name) VALUES ('persistent data');"

# Удаляем контейнер
docker rm -f postgres-db

# Создаём новый с тем же volume — данные на месте!
docker run -d \
  --name postgres-db-new \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -e POSTGRES_DB=mydb \
  -v postgres-data:/var/lib/postgresql/data \
  -p 5432:5432 \
  postgres:15

docker exec -it postgres-db-new psql -U myuser -d mydb -c "SELECT * FROM test;"
# id |      name
# ----+----------------
#  1 | persistent data
```

### PostgreSQL с custom конфигурацией

```bash
# Файл postgresql.conf на хосте
cat > ./postgres-config/postgresql.conf << 'EOF'
max_connections = 200
shared_buffers = 256MB
effective_cache_size = 768MB
maintenance_work_mem = 64MB
checkpoint_completion_target = 0.7
wal_buffers = 7864kB
default_statistics_target = 100
random_page_cost = 1.1
effective_io_concurrency = 200
work_mem = 1310kB
min_wal_size = 1GB
max_wal_size = 4GB
EOF

# Запуск с bind mount для конфига и volume для данных
docker run -d \
  --name postgres-tuned \
  -e POSTGRES_USER=myuser \
  -e POSTGRES_PASSWORD=mypassword \
  -v postgres-data:/var/lib/postgresql/data \
  -v $(pwd)/postgres-config/postgresql.conf:/etc/postgresql/postgresql.conf:ro \
  -p 5432:5432 \
  postgres:15 -c 'config_file=/etc/postgresql/postgresql.conf'
```

### MongoDB с Volume

```bash
# Создаём volumes для данных и конфигурации
docker volume create mongo-data
docker volume create mongo-config

# Запуск MongoDB
docker run -d \
  --name mongodb \
  -e MONGO_INITDB_ROOT_USERNAME=admin \
  -e MONGO_INITDB_ROOT_PASSWORD=secretpassword \
  -v mongo-data:/data/db \
  -v mongo-config:/data/configdb \
  -p 27017:27017 \
  mongo:7

# Тест персистентности
docker exec -it mongodb mongosh -u admin -p secretpassword --eval "
  use testdb;
  db.users.insertOne({name: 'John', email: 'john@example.com'});
  db.users.find();
"

# Перезапуск — данные сохранены
docker restart mongodb
docker exec -it mongodb mongosh -u admin -p secretpassword --eval "
  use testdb;
  db.users.find();
"
```

### Redis с персистентностью

```bash
# Volume для данных Redis
docker volume create redis-data

# Redis с RDB и AOF персистентностью
docker run -d \
  --name redis \
  -v redis-data:/data \
  -p 6379:6379 \
  redis:7 redis-server --appendonly yes --save 60 1000

# Проверка
docker exec -it redis redis-cli SET mykey "persistent value"
docker restart redis
docker exec -it redis redis-cli GET mykey
# "persistent value"
```

### Docker Compose с volumes

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: mydb
    volumes:
      - postgres-data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d:ro
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
    command: postgres -c config_file=/etc/postgresql/postgresql.conf
    ports:
      - "5432:5432"

  mongodb:
    image: mongo:7
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: secretpassword
    volumes:
      - mongo-data:/data/db
      - mongo-config:/data/configdb
    ports:
      - "27017:27017"

  redis:
    image: redis:7
    volumes:
      - redis-data:/data
    command: redis-server --appendonly yes
    ports:
      - "6379:6379"

volumes:
  postgres-data:
    name: myapp-postgres-data
  mongo-data:
    name: myapp-mongo-data
  mongo-config:
    name: myapp-mongo-config
  redis-data:
    name: myapp-redis-data
```

```bash
# Управление
docker-compose up -d
docker-compose down           # Контейнеры удаляются, volumes остаются
docker-compose down -v        # Удаляет и volumes (осторожно!)

# Backup всех volumes
docker-compose stop
for vol in postgres-data mongo-data mongo-config redis-data; do
  docker run --rm \
    -v myapp-$vol:/source:ro \
    -v $(pwd)/backup:/backup \
    alpine tar czf /backup/$vol-$(date +%Y%m%d).tar.gz -C /source .
done
docker-compose start
```

---

## Полезные команды

```bash
# Список volumes с размерами (Linux)
docker system df -v

# Найти orphan volumes (не используемые контейнерами)
docker volume ls -f dangling=true

# Удалить все orphan volumes
docker volume prune

# Скопировать данные между volumes
docker run --rm \
  -v source-vol:/source:ro \
  -v target-vol:/target \
  alpine cp -a /source/. /target/

# Инспектировать volume через временный контейнер
docker run --rm -it -v my-volume:/data alpine sh

# Проверить какие контейнеры используют volume
docker ps -a --filter volume=my-volume
```

---

## Заключение

Правильное управление данными в Docker — ключевой навык для работы с контейнерами:

1. **Volumes** — основной инструмент для production. Управляются Docker, легко бэкапить и переносить
2. **Bind Mounts** — идеальны для разработки и конфигураций. Прямой доступ к файлам хоста
3. **tmpfs** — для секретов и временных данных. Максимальная безопасность и скорость

**Золотые правила:**
- Всегда используйте named volumes в production
- Применяйте readonly где возможно
- Регулярно делайте backup важных volumes
- Не храните секреты в обычных volumes — используйте tmpfs или Docker Secrets
