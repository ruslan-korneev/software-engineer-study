# Запуск контейнеров

## Введение: Жизненный цикл контейнера

Контейнер Docker проходит через несколько состояний в течение своего жизненного цикла:

```
Created → Running → Paused → Stopped → Removed
   ↑         ↓         ↓         ↓
   └─────────┴─────────┴─────────┘
```

**Основные состояния:**
- **Created** — контейнер создан, но не запущен
- **Running** — контейнер выполняется
- **Paused** — контейнер приостановлен (процессы заморожены)
- **Stopped/Exited** — контейнер остановлен
- **Removed** — контейнер удалён

```bash
# Посмотреть состояние контейнеров
docker ps          # только запущенные
docker ps -a       # все контейнеры
docker ps -q       # только ID
```

---

## Команда docker run

### Основной синтаксис

```bash
docker run [OPTIONS] IMAGE [COMMAND] [ARG...]
```

Команда `docker run` выполняет три действия:
1. Создаёт новый контейнер из образа (`docker create`)
2. Запускает контейнер (`docker start`)
3. Присоединяется к выводу контейнера (если не указан флаг `-d`)

### Важные флаги

| Флаг | Описание |
|------|----------|
| `-d, --detach` | Запуск в фоновом режиме |
| `-it` | Интерактивный режим с TTY |
| `--name` | Имя контейнера |
| `--rm` | Автоматическое удаление после остановки |
| `-p, --publish` | Проброс портов |
| `-v, --volume` | Монтирование томов |
| `-e, --env` | Переменные окружения |
| `--network` | Подключение к сети |
| `--restart` | Политика перезапуска |
| `-w, --workdir` | Рабочая директория |
| `-u, --user` | Пользователь для запуска |
| `--entrypoint` | Переопределение ENTRYPOINT |

---

## Режимы запуска

### Foreground mode (передний план)

По умолчанию контейнер запускается на переднем плане:

```bash
# Контейнер выводит логи в терминал
docker run nginx

# Ctrl+C остановит контейнер
```

### Detached mode (фоновый режим)

Флаг `-d` запускает контейнер в фоне:

```bash
# Контейнер работает в фоне
docker run -d nginx

# Вывод: ID контейнера
# a1b2c3d4e5f6...

# Просмотр логов фонового контейнера
docker logs <container_id>
docker logs -f <container_id>  # следить за логами в реальном времени
```

### Interactive mode (интерактивный режим)

Флаги `-i` и `-t` часто используются вместе:

```bash
# -i (--interactive) — держит STDIN открытым
# -t (--tty) — выделяет псевдо-TTY

# Запуск bash в контейнере
docker run -it ubuntu bash

# Запуск Python REPL
docker run -it python:3.11

# Выход из интерактивного режима
# exit или Ctrl+D — остановит контейнер
# Ctrl+P, Ctrl+Q — отсоединиться без остановки (detach)
```

**Комбинация флагов:**

```bash
# Интерактивный режим в фоне (для последующего attach)
docker run -dit --name my-ubuntu ubuntu

# Присоединиться к контейнеру
docker attach my-ubuntu

# Выполнить команду в работающем контейнере
docker exec -it my-ubuntu bash
```

---

## Проброс портов

### Синтаксис флага -p

```bash
docker run -p [host_ip:]host_port:container_port[/protocol]
```

### Примеры проброса портов

```bash
# Базовый проброс: хост:контейнер
docker run -d -p 8080:80 nginx
# Доступ: http://localhost:8080

# Несколько портов
docker run -d -p 8080:80 -p 8443:443 nginx

# Диапазон портов
docker run -d -p 8080-8090:80-90 my-app

# Случайный порт на хосте
docker run -d -p 80 nginx
docker port <container_id>  # узнать назначенный порт

# UDP протокол
docker run -d -p 53:53/udp dns-server

# TCP и UDP одновременно
docker run -d -p 53:53/tcp -p 53:53/udp dns-server
```

### Привязка к конкретному интерфейсу

```bash
# Только localhost (безопаснее)
docker run -d -p 127.0.0.1:8080:80 nginx

# Все интерфейсы (по умолчанию)
docker run -d -p 0.0.0.0:8080:80 nginx

# Конкретный IP
docker run -d -p 192.168.1.100:8080:80 nginx

# IPv6
docker run -d -p [::1]:8080:80 nginx
```

### Просмотр маппинга портов

```bash
# Все порты контейнера
docker port my-nginx

# Конкретный порт
docker port my-nginx 80

# Вывод: 0.0.0.0:8080
```

---

## Монтирование томов

### Bind mounts (привязка директорий)

Монтирование директории хоста в контейнер:

```bash
# Синтаксис -v
docker run -v /path/on/host:/path/in/container image

# Примеры
docker run -v $(pwd):/app node:18 npm start
docker run -v /var/log/app:/var/log nginx

# Только для чтения
docker run -v /config:/app/config:ro nginx

# Синтаксис --mount (более явный)
docker run --mount type=bind,source=/host/path,target=/container/path image

docker run --mount type=bind,source=$(pwd),target=/app,readonly node:18
```

### Named volumes (именованные тома)

Docker управляет хранением данных:

```bash
# Создание тома
docker volume create my-data

# Использование тома
docker run -v my-data:/var/lib/mysql mysql:8

# Синтаксис --mount
docker run --mount type=volume,source=my-data,target=/var/lib/mysql mysql:8

# Автоматическое создание тома
docker run -v new-volume:/data alpine
```

### Anonymous volumes (анонимные тома)

```bash
# Создаётся автоматически, имя генерируется
docker run -v /data alpine

# Полезно для временных данных
docker run --rm -v /tmp alpine
```

### Сравнение bind mounts и volumes

| Характеристика | Bind mount | Volume |
|----------------|------------|--------|
| Расположение | Любое место на хосте | Управляется Docker |
| Производительность | Зависит от FS хоста | Оптимизировано Docker |
| Бэкап | Вручную | `docker volume` команды |
| Переносимость | Привязан к хосту | Легко переносится |
| Использование | Разработка, конфиги | Production данные |

---

## Переменные окружения

### Флаг -e (--env)

```bash
# Одна переменная
docker run -e DATABASE_URL=postgres://localhost:5432/db myapp

# Несколько переменных
docker run \
  -e DATABASE_URL=postgres://localhost:5432/db \
  -e REDIS_URL=redis://localhost:6379 \
  -e DEBUG=true \
  myapp

# Передача переменной из текущего окружения
export API_KEY=secret123
docker run -e API_KEY myapp  # передаст значение из хоста

# Переменная без значения (пустая строка)
docker run -e EMPTY_VAR= myapp
```

### Файлы переменных окружения

```bash
# Файл .env
# DATABASE_URL=postgres://localhost:5432/db
# REDIS_URL=redis://localhost:6379
# SECRET_KEY=my-secret-key

docker run --env-file .env myapp

# Несколько файлов
docker run --env-file .env --env-file .env.local myapp

# Переопределение переменных из файла
docker run --env-file .env -e DEBUG=true myapp
```

**Формат файла .env:**

```env
# Комментарии поддерживаются
DATABASE_URL=postgres://user:pass@host:5432/db
REDIS_URL=redis://localhost:6379

# Без кавычек
API_KEY=abc123

# С кавычками (кавычки будут частью значения!)
# WRONG_VAR="value"  # значение будет "value" с кавычками
```

---

## Управление ресурсами

### Ограничение CPU

```bash
# Количество CPU (дробное значение)
docker run --cpus=1.5 myapp  # 1.5 CPU ядра

# CPU shares (относительный приоритет, по умолчанию 1024)
docker run --cpu-shares=512 myapp   # половина приоритета
docker run --cpu-shares=2048 myapp  # двойной приоритет

# Привязка к конкретным ядрам
docker run --cpuset-cpus="0,1" myapp     # ядра 0 и 1
docker run --cpuset-cpus="0-3" myapp     # ядра 0, 1, 2, 3

# CPU период и квота (продвинутое)
docker run --cpu-period=100000 --cpu-quota=50000 myapp  # 50% CPU
```

### Ограничение памяти

```bash
# Жёсткий лимит памяти
docker run -m 512m myapp        # 512 МБ
docker run --memory=1g myapp    # 1 ГБ

# Память + swap
docker run -m 512m --memory-swap=1g myapp  # 512MB RAM + 512MB swap

# Отключение swap
docker run -m 512m --memory-swap=512m myapp

# Мягкий лимит (memory reservation)
docker run -m 1g --memory-reservation=512m myapp

# OOM killer priority (-1000 до 1000)
docker run --oom-score-adj=-500 myapp  # менее вероятен kill

# Отключение OOM killer (опасно!)
docker run --oom-kill-disable -m 512m myapp
```

### Просмотр использования ресурсов

```bash
# Статистика в реальном времени
docker stats

# Конкретный контейнер
docker stats my-container

# Однократный вывод
docker stats --no-stream

# Форматированный вывод
docker stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

---

## Restart policies (политики перезапуска)

### Типы политик

| Политика | Описание |
|----------|----------|
| `no` | Не перезапускать (по умолчанию) |
| `always` | Всегда перезапускать |
| `on-failure[:max-retries]` | Перезапускать при ошибке |
| `unless-stopped` | Всегда, кроме ручной остановки |

### Примеры использования

```bash
# Не перезапускать (по умолчанию)
docker run --restart=no nginx

# Всегда перезапускать (включая после reboot хоста)
docker run -d --restart=always nginx

# Перезапуск при ошибке (exit code != 0)
docker run -d --restart=on-failure nginx

# Максимум 3 попытки перезапуска
docker run -d --restart=on-failure:3 nginx

# Перезапускать, если не остановлен вручную
docker run -d --restart=unless-stopped nginx
```

### Разница между always и unless-stopped

```bash
# Сценарий: контейнер запущен, Docker daemon перезапускается

# --restart=always
# Контейнер перезапустится после рестарта демона

# --restart=unless-stopped
# Если контейнер был остановлен вручную ДО рестарта демона,
# он НЕ запустится автоматически
```

### Изменение политики для работающего контейнера

```bash
docker update --restart=always my-container
```

---

## Практические примеры запуска

### Веб-сервер Nginx

```bash
# Простой запуск
docker run -d --name web -p 80:80 nginx

# С кастомной конфигурацией
docker run -d \
  --name web \
  -p 80:80 \
  -v $(pwd)/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v $(pwd)/html:/usr/share/nginx/html:ro \
  nginx

# Production-ready
docker run -d \
  --name web \
  --restart=unless-stopped \
  -p 80:80 \
  -p 443:443 \
  -v /etc/nginx/nginx.conf:/etc/nginx/nginx.conf:ro \
  -v /var/www/html:/usr/share/nginx/html:ro \
  -v /etc/letsencrypt:/etc/letsencrypt:ro \
  --memory=256m \
  --cpus=0.5 \
  nginx:alpine
```

### База данных PostgreSQL

```bash
# Разработка
docker run -d \
  --name postgres \
  -e POSTGRES_USER=dev \
  -e POSTGRES_PASSWORD=devpass \
  -e POSTGRES_DB=myapp \
  -p 5432:5432 \
  postgres:16

# Production с персистентностью
docker run -d \
  --name postgres \
  --restart=unless-stopped \
  -e POSTGRES_USER=prod \
  -e POSTGRES_PASSWORD_FILE=/run/secrets/db_password \
  -e POSTGRES_DB=production \
  -v postgres-data:/var/lib/postgresql/data \
  -v /path/to/init:/docker-entrypoint-initdb.d:ro \
  -p 127.0.0.1:5432:5432 \
  --memory=2g \
  --cpus=2 \
  postgres:16-alpine
```

### Redis

```bash
# Простой запуск
docker run -d --name redis -p 6379:6379 redis:7

# С персистентностью и паролем
docker run -d \
  --name redis \
  --restart=unless-stopped \
  -v redis-data:/data \
  -p 127.0.0.1:6379:6379 \
  redis:7-alpine \
  redis-server --appendonly yes --requirepass "secret"
```

### Node.js приложение

```bash
# Разработка с hot reload
docker run -it --rm \
  --name node-dev \
  -v $(pwd):/app \
  -w /app \
  -p 3000:3000 \
  -e NODE_ENV=development \
  node:20-alpine \
  npm run dev

# Production
docker run -d \
  --name node-app \
  --restart=unless-stopped \
  -p 3000:3000 \
  -e NODE_ENV=production \
  --env-file .env.production \
  --memory=512m \
  --cpus=1 \
  my-node-app:latest
```

### Python/Django приложение

```bash
# Разработка
docker run -it --rm \
  -v $(pwd):/app \
  -w /app \
  -p 8000:8000 \
  -e DJANGO_DEBUG=true \
  python:3.11-slim \
  python manage.py runserver 0.0.0.0:8000

# С зависимостями
docker run -d \
  --name django \
  -v $(pwd):/app \
  -w /app \
  -p 8000:8000 \
  --env-file .env \
  python:3.11-slim \
  sh -c "pip install -r requirements.txt && python manage.py runserver 0.0.0.0:8000"
```

### Многоконтейнерное приложение (без Compose)

```bash
# Создание сети
docker network create myapp-net

# База данных
docker run -d \
  --name db \
  --network myapp-net \
  -e POSTGRES_PASSWORD=secret \
  -v db-data:/var/lib/postgresql/data \
  postgres:16

# Redis
docker run -d \
  --name cache \
  --network myapp-net \
  redis:7

# Приложение
docker run -d \
  --name app \
  --network myapp-net \
  -p 8000:8000 \
  -e DATABASE_URL=postgres://postgres:secret@db:5432/postgres \
  -e REDIS_URL=redis://cache:6379 \
  myapp:latest
```

---

## Best Practices

### 1. Именование контейнеров

```bash
# Используйте понятные имена
docker run -d --name web-frontend nginx
docker run -d --name api-backend myapp
docker run -d --name db-primary postgres

# Избегайте случайных имён
docker run -d nginx  # имя будет типа "eager_tesla"
```

### 2. Используйте --rm для временных контейнеров

```bash
# Контейнер удалится после остановки
docker run --rm -it python:3.11 python

# Полезно для одноразовых задач
docker run --rm -v $(pwd):/app node:20 npm install
```

### 3. Минимальные привилегии

```bash
# Запуск от непривилегированного пользователя
docker run -u 1000:1000 myapp

# Запрет эскалации привилегий
docker run --security-opt=no-new-privileges myapp

# Read-only файловая система
docker run --read-only myapp

# Временные директории для записи
docker run --read-only --tmpfs /tmp --tmpfs /var/run myapp
```

### 4. Ограничивайте ресурсы

```bash
# Всегда устанавливайте лимиты в production
docker run -d \
  --memory=512m \
  --memory-reservation=256m \
  --cpus=1 \
  myapp
```

### 5. Привязывайте порты к localhost в разработке

```bash
# Безопаснее в разработке
docker run -p 127.0.0.1:5432:5432 postgres
docker run -p 127.0.0.1:6379:6379 redis

# Только для production с firewall
docker run -p 0.0.0.0:80:80 nginx
```

### 6. Используйте конкретные теги образов

```bash
# Плохо: latest может измениться
docker run nginx:latest

# Хорошо: фиксированная версия
docker run nginx:1.25.3-alpine
```

### 7. Логирование

```bash
# Настройка драйвера логов
docker run -d \
  --log-driver=json-file \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# Отключение логов (осторожно!)
docker run -d --log-driver=none myapp
```

### 8. Health checks при запуске

```bash
# Проверка здоровья
docker run -d \
  --health-cmd="curl -f http://localhost/ || exit 1" \
  --health-interval=30s \
  --health-timeout=10s \
  --health-retries=3 \
  --health-start-period=40s \
  nginx
```

---

## Типичные ошибки

### 1. Забыли флаг -d

```bash
# Проблема: терминал заблокирован
docker run nginx

# Решение: фоновый режим
docker run -d nginx
```

### 2. Контейнер сразу останавливается

```bash
# Проблема: процесс завершается мгновенно
docker run -d ubuntu
docker ps  # контейнер не виден

# Причина: нет foreground процесса

# Решение 1: интерактивный режим
docker run -dit ubuntu

# Решение 2: команда, которая не завершается
docker run -d ubuntu tail -f /dev/null

# Решение 3: правильный образ с сервисом
docker run -d nginx  # nginx работает в foreground
```

### 3. Порт уже занят

```bash
# Ошибка: "port is already allocated"
docker run -d -p 80:80 nginx  # Error!

# Решение 1: использовать другой порт
docker run -d -p 8080:80 nginx

# Решение 2: найти и остановить конфликтующий контейнер
docker ps | grep 80
docker stop <container>

# Решение 3: найти процесс на порту
lsof -i :80
```

### 4. Проблемы с правами на volumes

```bash
# Ошибка: Permission denied при записи в volume
docker run -v $(pwd)/data:/data myapp

# Решение 1: запуск от root (не рекомендуется)
docker run -u root -v $(pwd)/data:/data myapp

# Решение 2: исправить права на хосте
chmod -R 777 ./data  # осторожно с правами

# Решение 3: использовать правильный UID
docker run -u $(id -u):$(id -g) -v $(pwd)/data:/data myapp
```

### 5. Контейнер не видит другие контейнеры

```bash
# Проблема: не работает обращение по имени
docker run -d --name db postgres
docker run myapp  # не может подключиться к db:5432

# Решение: использовать общую сеть
docker network create mynet
docker run -d --name db --network mynet postgres
docker run --network mynet myapp  # теперь db доступен
```

### 6. Потеря данных при удалении контейнера

```bash
# Проблема: данные пропадают
docker run -d --name db postgres
docker rm -f db  # данные потеряны!

# Решение: использовать volumes
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres
docker rm -f db
docker run -d --name db -v pgdata:/var/lib/postgresql/data postgres
# данные сохранены!
```

### 7. Переменные окружения с пробелами

```bash
# Проблема: неправильное экранирование
docker run -e MESSAGE=Hello World myapp  # World интерпретируется как команда

# Решение: использовать кавычки
docker run -e "MESSAGE=Hello World" myapp
docker run -e 'MESSAGE=Hello World' myapp
```

### 8. Использование --rm с -d

```bash
# Внимание: контейнер удалится после остановки
docker run -d --rm nginx

# Это может быть неожиданно, если вы хотите:
docker stop nginx
docker start nginx  # Error: container doesn't exist!

# --rm полезен для одноразовых задач, не для сервисов
```

### 9. Забыли про --restart в production

```bash
# Проблема: после перезагрузки сервера контейнеры не запустились

# Решение: добавить политику перезапуска
docker run -d --restart=unless-stopped nginx

# Или обновить существующий контейнер
docker update --restart=unless-stopped nginx
```

### 10. Слишком большие логи

```bash
# Проблема: диск переполнен логами контейнера
du -sh /var/lib/docker/containers/*/

# Решение: ограничить размер логов
docker run -d \
  --log-opt max-size=10m \
  --log-opt max-file=3 \
  myapp

# Или настроить глобально в /etc/docker/daemon.json
# {
#   "log-driver": "json-file",
#   "log-opts": {
#     "max-size": "10m",
#     "max-file": "3"
#   }
# }
```

---

## Полезные команды для управления контейнерами

```bash
# Список контейнеров
docker ps                    # запущенные
docker ps -a                 # все
docker ps -a -q              # только ID
docker ps --filter "status=exited"  # по статусу

# Остановка
docker stop container        # graceful (SIGTERM, затем SIGKILL)
docker kill container        # принудительно (SIGKILL)
docker stop $(docker ps -q)  # остановить все

# Запуск остановленного контейнера
docker start container
docker start -ai container   # с attach

# Перезапуск
docker restart container

# Удаление
docker rm container          # удалить остановленный
docker rm -f container       # принудительно (даже запущенный)
docker rm $(docker ps -aq)   # удалить все остановленные

# Логи
docker logs container
docker logs -f container     # follow
docker logs --tail 100 container  # последние 100 строк

# Выполнение команды в контейнере
docker exec container command
docker exec -it container bash

# Копирование файлов
docker cp file.txt container:/path/
docker cp container:/path/file.txt ./

# Информация
docker inspect container
docker top container         # процессы
docker stats container       # ресурсы
```

---

## Резюме

| Задача | Команда |
|--------|---------|
| Простой запуск | `docker run nginx` |
| Фоновый запуск | `docker run -d nginx` |
| Интерактивный | `docker run -it ubuntu bash` |
| С именем | `docker run --name web nginx` |
| С пробросом порта | `docker run -p 8080:80 nginx` |
| С volume | `docker run -v data:/app/data myapp` |
| С переменными | `docker run -e KEY=value myapp` |
| С автоудалением | `docker run --rm alpine echo hi` |
| С перезапуском | `docker run --restart=always nginx` |
| Production-ready | `docker run -d --name web --restart=unless-stopped -p 80:80 -m 512m --cpus=1 nginx` |
