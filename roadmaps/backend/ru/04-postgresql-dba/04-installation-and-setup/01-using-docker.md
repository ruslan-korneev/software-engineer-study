# Установка PostgreSQL с использованием Docker

[prev: 05-query-processing](../03-high-level-database-concepts/05-query-processing.md) | [next: 02-package-managers](./02-package-managers.md)
---

## Введение

Docker — это платформа для контейнеризации приложений, которая позволяет запускать PostgreSQL изолированно от основной системы. Это идеальный вариант для разработки, тестирования и даже production-окружений.

## Преимущества использования Docker

- **Изоляция**: PostgreSQL работает в отдельном контейнере, не влияя на систему
- **Портативность**: одинаковая конфигурация на любой машине
- **Простота управления версиями**: легко переключаться между версиями PostgreSQL
- **Быстрое развертывание**: запуск новой базы данных за секунды
- **Воспроизводимость**: одинаковое окружение для всей команды

## Базовые команды

### Запуск простого контейнера

```bash
# Скачать официальный образ PostgreSQL
docker pull postgres:16

# Запустить контейнер
docker run --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -p 5432:5432 \
  -d postgres:16
```

### Параметры запуска

| Параметр | Описание |
|----------|----------|
| `--name` | Имя контейнера |
| `-e POSTGRES_PASSWORD` | Пароль для пользователя postgres (обязательный) |
| `-e POSTGRES_USER` | Имя пользователя (по умолчанию: postgres) |
| `-e POSTGRES_DB` | Имя базы данных (по умолчанию: postgres) |
| `-p 5432:5432` | Проброс порта хост:контейнер |
| `-d` | Запуск в фоновом режиме (detached) |
| `-v` | Монтирование тома для persistence |

### Persistence данных (Volume)

```bash
# Создать именованный том
docker volume create pgdata

# Запуск с persistence
docker run --name my-postgres \
  -e POSTGRES_PASSWORD=mysecretpassword \
  -v pgdata:/var/lib/postgresql/data \
  -p 5432:5432 \
  -d postgres:16
```

## Docker Compose

Для более сложных конфигураций рекомендуется использовать Docker Compose.

### Базовая конфигурация

```yaml
# docker-compose.yml
version: '3.8'

services:
  postgres:
    image: postgres:16
    container_name: postgres_db
    restart: unless-stopped
    environment:
      POSTGRES_USER: myuser
      POSTGRES_PASSWORD: mypassword
      POSTGRES_DB: myapp
    ports:
      - "5432:5432"
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./init-scripts:/docker-entrypoint-initdb.d

volumes:
  postgres_data:
```

### Расширенная production-конфигурация

```yaml
# docker-compose.prod.yml
version: '3.8'

services:
  postgres:
    image: postgres:16-alpine
    container_name: postgres_prod
    restart: always
    environment:
      POSTGRES_USER: ${DB_USER}
      POSTGRES_PASSWORD: ${DB_PASSWORD}
      POSTGRES_DB: ${DB_NAME}
      POSTGRES_INITDB_ARGS: "--encoding=UTF8 --locale=C"
    ports:
      - "127.0.0.1:5432:5432"  # Только localhost
    volumes:
      - postgres_data:/var/lib/postgresql/data
      - ./postgresql.conf:/etc/postgresql/postgresql.conf:ro
      - ./pg_hba.conf:/etc/postgresql/pg_hba.conf:ro
    command: >
      postgres
        -c config_file=/etc/postgresql/postgresql.conf
        -c hba_file=/etc/postgresql/pg_hba.conf
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U ${DB_USER} -d ${DB_NAME}"]
      interval: 10s
      timeout: 5s
      retries: 5
    deploy:
      resources:
        limits:
          memory: 4G
          cpus: '2'

volumes:
  postgres_data:
    driver: local
```

## Кастомизация образа

### Dockerfile с расширениями

```dockerfile
# Dockerfile
FROM postgres:16-alpine

# Установка дополнительных расширений
RUN apk add --no-cache \
    postgresql16-contrib

# Копирование init-скриптов
COPY ./init-scripts/ /docker-entrypoint-initdb.d/

# Копирование кастомной конфигурации
COPY ./postgresql.conf /etc/postgresql/postgresql.conf

# Установка переменных окружения по умолчанию
ENV POSTGRES_DB=myapp
ENV POSTGRES_USER=appuser

EXPOSE 5432
```

### Пример init-скрипта

```sql
-- /init-scripts/01-extensions.sql
CREATE EXTENSION IF NOT EXISTS "uuid-ossp";
CREATE EXTENSION IF NOT EXISTS "pg_trgm";
CREATE EXTENSION IF NOT EXISTS "btree_gin";

-- Создание дополнительных схем
CREATE SCHEMA IF NOT EXISTS app;
CREATE SCHEMA IF NOT EXISTS analytics;

-- Создание ролей
CREATE ROLE readonly;
CREATE ROLE readwrite;
```

## Управление контейнером

```bash
# Просмотр логов
docker logs my-postgres
docker logs -f my-postgres  # Follow режим

# Остановка контейнера
docker stop my-postgres

# Запуск остановленного контейнера
docker start my-postgres

# Перезапуск
docker restart my-postgres

# Удаление контейнера
docker rm my-postgres

# Удаление контейнера вместе с volume
docker rm -v my-postgres

# Подключение к контейнеру
docker exec -it my-postgres bash

# Прямое подключение к psql
docker exec -it my-postgres psql -U postgres
```

## Команды Docker Compose

```bash
# Запуск сервисов
docker-compose up -d

# Остановка
docker-compose down

# Остановка с удалением volumes
docker-compose down -v

# Просмотр логов
docker-compose logs postgres
docker-compose logs -f postgres

# Перезапуск конкретного сервиса
docker-compose restart postgres

# Выполнение команды в контейнере
docker-compose exec postgres psql -U myuser -d myapp
```

## Backup и Restore в Docker

### Создание backup

```bash
# pg_dump в файл на хосте
docker exec -t my-postgres pg_dump -U postgres mydb > backup.sql

# pg_dump с сжатием
docker exec -t my-postgres pg_dump -U postgres -Fc mydb > backup.dump

# pg_dumpall для всех баз
docker exec -t my-postgres pg_dumpall -U postgres > backup_all.sql
```

### Восстановление из backup

```bash
# Восстановление из SQL
docker exec -i my-postgres psql -U postgres -d mydb < backup.sql

# Восстановление из custom format
docker exec -i my-postgres pg_restore -U postgres -d mydb < backup.dump
```

## Настройка для Production

### Оптимизация postgresql.conf

```conf
# postgresql.conf
# Память
shared_buffers = 1GB
effective_cache_size = 3GB
work_mem = 64MB
maintenance_work_mem = 256MB

# Checkpoints
checkpoint_completion_target = 0.9
wal_buffers = 64MB
max_wal_size = 2GB
min_wal_size = 512MB

# Сетевые настройки
listen_addresses = '*'
max_connections = 200

# Логирование
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d.log'
log_statement = 'ddl'
log_duration = on
log_min_duration_statement = 1000  # Логировать запросы > 1 сек
```

### Настройка pg_hba.conf

```conf
# pg_hba.conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             all                                     trust
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
host    all             all             172.16.0.0/12           scram-sha-256
host    replication     replicator      172.16.0.0/12           scram-sha-256
```

## Best Practices

1. **Всегда используйте volumes** для persistence данных
2. **Не храните пароли в docker-compose.yml** — используйте `.env` файлы или secrets
3. **Ограничивайте ресурсы** контейнера в production
4. **Используйте alpine-образы** для меньшего размера
5. **Настройте healthcheck** для мониторинга состояния
6. **Привязывайте порт к localhost** (`127.0.0.1:5432:5432`) в production
7. **Регулярно обновляйте образы** для получения патчей безопасности
8. **Делайте backup** перед обновлением версии PostgreSQL

## Полезные ссылки

- [Официальный образ PostgreSQL на Docker Hub](https://hub.docker.com/_/postgres)
- [Docker Compose документация](https://docs.docker.com/compose/)
- [PostgreSQL Docker Best Practices](https://www.postgresql.org/docs/)

---
[prev: 05-query-processing](../03-high-level-database-concepts/05-query-processing.md) | [next: 02-package-managers](./02-package-managers.md)