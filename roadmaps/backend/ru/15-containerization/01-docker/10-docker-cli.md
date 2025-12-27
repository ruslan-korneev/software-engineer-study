# Командная строка Docker (Docker CLI)

## Введение

Docker CLI (Command Line Interface) — это основной инструмент для взаимодействия с Docker. Он позволяет управлять контейнерами, образами, сетями, томами и другими ресурсами Docker из терминала. Команда `docker` общается с Docker daemon через REST API.

Docker CLI предоставляет мощный набор команд для:
- Управления жизненным циклом контейнеров
- Работы с образами (создание, загрузка, удаление)
- Настройки сетей и томов
- Мониторинга и отладки
- Системного администрирования Docker

---

## Структура команд Docker

### Старый vs Новый синтаксис

Docker развивался, и со временем структура команд была реорганизована для лучшей читаемости:

| Старый синтаксис | Новый синтаксис | Описание |
|------------------|-----------------|----------|
| `docker ps` | `docker container ls` | Список контейнеров |
| `docker images` | `docker image ls` | Список образов |
| `docker rmi` | `docker image rm` | Удаление образа |
| `docker rm` | `docker container rm` | Удаление контейнера |
| `docker run` | `docker container run` | Запуск контейнера |
| `docker exec` | `docker container exec` | Выполнение команды в контейнере |

**Рекомендация**: Используйте новый синтаксис — он более явный и понятный. Старый синтаксис всё ещё работает для обратной совместимости.

### Общая структура команды

```
docker [OPTIONS] COMMAND [ARG...]
docker [OPTIONS] MANAGEMENT_COMMAND COMMAND [ARG...]
```

Примеры:
```bash
# Старый стиль
docker ps -a

# Новый стиль (рекомендуется)
docker container ls -a
```

### Получение справки

```bash
# Общая справка
docker --help

# Справка по management command
docker container --help

# Справка по конкретной команде
docker container run --help
```

---

## Команды для работы с контейнерами

### docker container ls / docker ps

Показывает список контейнеров.

```bash
# Показать запущенные контейнеры
docker container ls
docker ps

# Показать все контейнеры (включая остановленные)
docker container ls -a
docker ps -a

# Показать только ID контейнеров
docker container ls -q

# Показать последний созданный контейнер
docker container ls -l

# Показать размер контейнеров
docker container ls -s

# Показать N последних контейнеров
docker container ls -n 5
```

**Вывод команды включает:**
- CONTAINER ID — уникальный идентификатор
- IMAGE — образ, из которого создан контейнер
- COMMAND — команда запуска
- CREATED — время создания
- STATUS — состояние (Up, Exited, Created, etc.)
- PORTS — проброшенные порты
- NAMES — имя контейнера

### docker container start/stop/restart

Управление состоянием контейнера.

```bash
# Запустить остановленный контейнер
docker container start my_container

# Запустить несколько контейнеров
docker container start container1 container2 container3

# Запустить и подключиться к stdout
docker container start -a my_container

# Остановить контейнер (SIGTERM, затем SIGKILL через 10 сек)
docker container stop my_container

# Остановить с таймаутом (секунды до SIGKILL)
docker container stop -t 30 my_container

# Перезапустить контейнер
docker container restart my_container

# Перезапустить с таймаутом
docker container restart -t 5 my_container

# Принудительно остановить (SIGKILL сразу)
docker container kill my_container

# Приостановить контейнер (freeze)
docker container pause my_container

# Возобновить контейнер
docker container unpause my_container
```

### docker container rm

Удаление контейнеров.

```bash
# Удалить остановленный контейнер
docker container rm my_container

# Удалить несколько контейнеров
docker container rm container1 container2

# Принудительное удаление (включая запущенные)
docker container rm -f my_container

# Удалить контейнер и его тома
docker container rm -v my_container

# Удалить все остановленные контейнеры
docker container prune

# Удалить без подтверждения
docker container prune -f

# Удалить все контейнеры (хитрость)
docker container rm -f $(docker container ls -aq)
```

### docker container exec

Выполнение команд внутри запущенного контейнера.

```bash
# Выполнить команду
docker container exec my_container ls -la

# Интерактивный режим с терминалом (самое частое использование)
docker container exec -it my_container /bin/bash
docker container exec -it my_container sh

# Выполнить от имени другого пользователя
docker container exec -u root my_container whoami

# Выполнить с переменными окружения
docker container exec -e MY_VAR=value my_container env

# Выполнить в определённой рабочей директории
docker container exec -w /app my_container pwd

# Выполнить в detached режиме
docker container exec -d my_container touch /tmp/test.txt
```

**Флаги:**
- `-i` (--interactive) — держать STDIN открытым
- `-t` (--tty) — выделить псевдо-TTY
- `-d` (--detach) — выполнить в фоне
- `-e` (--env) — установить переменную окружения
- `-u` (--user) — пользователь для выполнения
- `-w` (--workdir) — рабочая директория

### docker container logs

Просмотр логов контейнера.

```bash
# Показать все логи
docker container logs my_container

# Следить за логами в реальном времени (как tail -f)
docker container logs -f my_container

# Показать последние N строк
docker container logs --tail 100 my_container

# Показать логи с таймстампами
docker container logs -t my_container

# Показать логи с определённого времени
docker container logs --since 2024-01-01T00:00:00 my_container
docker container logs --since 30m my_container  # за последние 30 минут
docker container logs --since 2h my_container   # за последние 2 часа

# Показать логи до определённого времени
docker container logs --until 2024-01-01T12:00:00 my_container

# Комбинация: следить + таймстампы + последние 50 строк
docker container logs -f -t --tail 50 my_container
```

### docker container inspect

Получение детальной информации о контейнере в формате JSON.

```bash
# Полная информация о контейнере
docker container inspect my_container

# Получить конкретное значение с помощью Go template
docker container inspect --format '{{.State.Status}}' my_container

# Получить IP-адрес контейнера
docker container inspect --format '{{.NetworkSettings.IPAddress}}' my_container

# IP-адрес в конкретной сети
docker container inspect --format '{{.NetworkSettings.Networks.bridge.IPAddress}}' my_container

# Получить маппинг портов
docker container inspect --format '{{.NetworkSettings.Ports}}' my_container

# Получить переменные окружения
docker container inspect --format '{{.Config.Env}}' my_container

# Получить точку монтирования томов
docker container inspect --format '{{.Mounts}}' my_container

# Проверить состояние здоровья (healthcheck)
docker container inspect --format '{{.State.Health.Status}}' my_container

# Inspect нескольких контейнеров
docker container inspect container1 container2
```

### docker container stats

Мониторинг ресурсов в реальном времени.

```bash
# Статистика всех запущенных контейнеров (обновляется в реальном времени)
docker container stats

# Статистика конкретного контейнера
docker container stats my_container

# Однократный снимок (без обновления)
docker container stats --no-stream

# Показать все контейнеры (включая остановленные)
docker container stats -a

# Кастомный формат вывода
docker container stats --format "table {{.Name}}\t{{.CPUPerc}}\t{{.MemUsage}}"
```

**Выводимые метрики:**
- CPU % — использование процессора
- MEM USAGE / LIMIT — используемая / доступная память
- MEM % — процент использования памяти
- NET I/O — сетевой ввод/вывод
- BLOCK I/O — дисковый ввод/вывод
- PIDS — количество процессов

### Другие полезные команды для контейнеров

```bash
# Создать контейнер без запуска
docker container create --name my_container nginx

# Переименовать контейнер
docker container rename old_name new_name

# Скопировать файлы из/в контейнер
docker container cp my_container:/path/to/file ./local/path
docker container cp ./local/file my_container:/path/to/file

# Показать изменения в файловой системе контейнера
docker container diff my_container

# Экспортировать файловую систему контейнера в tar
docker container export my_container > container.tar

# Показать процессы в контейнере
docker container top my_container

# Показать порты контейнера
docker container port my_container

# Подключиться к запущенному контейнеру
docker container attach my_container

# Подождать завершения контейнера
docker container wait my_container

# Обновить ресурсы контейнера
docker container update --memory 512m my_container
```

---

## Команды для работы с образами

### docker image ls

Показывает список локальных образов.

```bash
# Показать все образы
docker image ls
docker images

# Показать все образы (включая промежуточные слои)
docker image ls -a

# Показать только ID образов
docker image ls -q

# Показать образы с определённым именем
docker image ls nginx

# Показать образы с фильтром
docker image ls --filter "dangling=true"  # образы без тега

# Показать размеры без усечения
docker image ls --no-trunc

# Форматированный вывод
docker image ls --format "{{.Repository}}:{{.Tag}} - {{.Size}}"
```

### docker image pull

Загрузка образа из registry.

```bash
# Загрузить latest версию
docker image pull nginx

# Загрузить конкретную версию
docker image pull nginx:1.25

# Загрузить все теги
docker image pull -a nginx

# Загрузить с конкретной платформы
docker image pull --platform linux/amd64 nginx

# Загрузить из другого registry
docker image pull gcr.io/google-containers/nginx
docker image pull myregistry.com:5000/myimage:tag
```

### docker image rm / prune

Удаление образов.

```bash
# Удалить образ по имени
docker image rm nginx

# Удалить образ по ID
docker image rm abc123def456

# Удалить несколько образов
docker image rm nginx redis postgres

# Принудительное удаление
docker image rm -f nginx

# Удалить все неиспользуемые образы (dangling)
docker image prune

# Удалить все неиспользуемые образы (не только dangling)
docker image prune -a

# Удалить без подтверждения
docker image prune -f

# Удалить с фильтром (старше 24 часов)
docker image prune -a --filter "until=24h"

# Удалить все образы (хитрость)
docker image rm -f $(docker image ls -q)
```

### docker image inspect

Детальная информация об образе.

```bash
# Полная информация
docker image inspect nginx

# Получить конкретное значение
docker image inspect --format '{{.Os}}' nginx

# Получить архитектуру
docker image inspect --format '{{.Architecture}}' nginx

# Получить точку входа
docker image inspect --format '{{.Config.Entrypoint}}' nginx

# Получить CMD
docker image inspect --format '{{.Config.Cmd}}' nginx

# Получить exposed ports
docker image inspect --format '{{.Config.ExposedPorts}}' nginx

# Получить переменные окружения
docker image inspect --format '{{.Config.Env}}' nginx

# Получить labels
docker image inspect --format '{{.Config.Labels}}' nginx
```

### docker image history

Показывает историю слоёв образа.

```bash
# Показать историю образа
docker image history nginx

# Показать полные команды (без усечения)
docker image history --no-trunc nginx

# Показать только размеры
docker image history --format "{{.Size}}" nginx

# Тихий режим (только ID)
docker image history -q nginx
```

### Другие команды для образов

```bash
# Создать образ из контейнера
docker container commit my_container my_new_image:tag

# Создать образ из Dockerfile
docker image build -t my_image:tag .

# Пометить образ новым тегом
docker image tag nginx:latest nginx:v1.0

# Отправить образ в registry
docker image push myregistry/myimage:tag

# Сохранить образ в tar-архив
docker image save nginx > nginx.tar
docker image save -o nginx.tar nginx

# Загрузить образ из tar-архива
docker image load < nginx.tar
docker image load -i nginx.tar

# Импортировать файловую систему как образ
docker image import container.tar my_image:tag
```

---

## Команды для работы с томами

### docker volume ls

```bash
# Показать все тома
docker volume ls

# Показать только имена томов
docker volume ls -q

# Показать с фильтром
docker volume ls --filter "dangling=true"  # неиспользуемые тома
docker volume ls --filter "driver=local"
docker volume ls --filter "name=my_volume"
```

### docker volume create

```bash
# Создать том с автоматическим именем
docker volume create

# Создать том с именем
docker volume create my_volume

# Создать том с драйвером
docker volume create --driver local my_volume

# Создать том с опциями
docker volume create --opt type=nfs --opt o=addr=192.168.1.1,rw --opt device=:/path/to/dir nfs_volume

# Создать том с метками
docker volume create --label env=production my_volume
```

### docker volume rm / prune

```bash
# Удалить том
docker volume rm my_volume

# Удалить несколько томов
docker volume rm volume1 volume2

# Принудительное удаление
docker volume rm -f my_volume

# Удалить все неиспользуемые тома
docker volume prune

# Удалить без подтверждения
docker volume prune -f

# Удалить с фильтром
docker volume prune --filter "label!=keep"
```

### docker volume inspect

```bash
# Полная информация о томе
docker volume inspect my_volume

# Получить путь монтирования
docker volume inspect --format '{{.Mountpoint}}' my_volume

# Получить драйвер
docker volume inspect --format '{{.Driver}}' my_volume

# Получить метки
docker volume inspect --format '{{.Labels}}' my_volume
```

---

## Команды для работы с сетями

### docker network ls

```bash
# Показать все сети
docker network ls

# Показать только ID
docker network ls -q

# Показать с фильтром
docker network ls --filter "driver=bridge"
docker network ls --filter "scope=local"
docker network ls --filter "name=my_network"

# Форматированный вывод
docker network ls --format "{{.Name}}: {{.Driver}}"
```

### docker network create

```bash
# Создать сеть (bridge по умолчанию)
docker network create my_network

# Создать сеть с драйвером
docker network create --driver bridge my_bridge
docker network create --driver overlay my_overlay

# Создать с подсетью и шлюзом
docker network create --subnet 172.20.0.0/16 --gateway 172.20.0.1 my_network

# Создать с диапазоном IP
docker network create --subnet 172.20.0.0/16 --ip-range 172.20.240.0/20 my_network

# Создать внутреннюю сеть (без доступа к внешней сети)
docker network create --internal my_internal_network

# Создать с метками
docker network create --label env=dev my_network

# Создать с опциями драйвера
docker network create --opt encrypted my_overlay
```

### docker network rm

```bash
# Удалить сеть
docker network rm my_network

# Удалить несколько сетей
docker network rm network1 network2

# Удалить все неиспользуемые сети
docker network prune

# Удалить без подтверждения
docker network prune -f
```

### docker network inspect

```bash
# Полная информация о сети
docker network inspect my_network

# Получить подсеть
docker network inspect --format '{{range .IPAM.Config}}{{.Subnet}}{{end}}' my_network

# Получить контейнеры в сети
docker network inspect --format '{{.Containers}}' my_network

# Получить драйвер
docker network inspect --format '{{.Driver}}' my_network
```

### docker network connect / disconnect

```bash
# Подключить контейнер к сети
docker network connect my_network my_container

# Подключить с конкретным IP
docker network connect --ip 172.20.0.100 my_network my_container

# Подключить с алиасом
docker network connect --alias db my_network my_container

# Отключить контейнер от сети
docker network disconnect my_network my_container

# Принудительное отключение
docker network disconnect -f my_network my_container
```

---

## Системные команды

### docker system df

Показывает использование диска Docker.

```bash
# Показать использование диска
docker system df

# Подробный вывод
docker system df -v
```

**Вывод показывает:**
- Images — образы
- Containers — контейнеры
- Local Volumes — тома
- Build Cache — кэш сборки

### docker system prune

Очистка неиспользуемых ресурсов.

```bash
# Очистить:
# - остановленные контейнеры
# - неиспользуемые сети
# - dangling образы
# - build cache
docker system prune

# То же самое без подтверждения
docker system prune -f

# Включая все неиспользуемые образы (не только dangling)
docker system prune -a

# Включая тома
docker system prune --volumes

# Полная очистка
docker system prune -a --volumes -f

# С фильтром по времени (старше 24 часов)
docker system prune --filter "until=24h"
```

### docker info

Показывает информацию о системе Docker.

```bash
# Полная информация о Docker
docker info

# Получить конкретное значение
docker info --format '{{.ServerVersion}}'
docker info --format '{{.NCPU}}'
docker info --format '{{.MemTotal}}'
docker info --format '{{.OperatingSystem}}'
docker info --format '{{.Driver}}'
```

### docker version

Показывает версию Docker.

```bash
# Полная информация о версии
docker version

# Только версия клиента
docker version --format '{{.Client.Version}}'

# Только версия сервера
docker version --format '{{.Server.Version}}'

# Короткий вывод
docker --version
```

### Другие системные команды

```bash
# События Docker в реальном времени
docker events

# События с фильтром
docker events --filter 'type=container'
docker events --filter 'event=start'
docker events --filter 'container=my_container'

# События за период
docker events --since '2024-01-01'
docker events --until '2024-01-02'

# Войти в registry
docker login
docker login myregistry.com

# Выйти из registry
docker logout
docker logout myregistry.com

# Поиск образов на Docker Hub
docker search nginx
docker search --filter "is-official=true" nginx
docker search --limit 5 nginx
```

---

## Полезные флаги и форматирование вывода

### Флаг --format с Go templates

Docker использует Go templates для форматирования вывода. Это мощный инструмент для получения нужных данных.

```bash
# Базовый синтаксис
docker container ls --format '{{.Names}}'

# Несколько полей
docker container ls --format '{{.Names}} - {{.Status}}'

# Табличный формат
docker container ls --format 'table {{.Names}}\t{{.Status}}\t{{.Ports}}'

# JSON вывод
docker container ls --format '{{json .}}'

# Красивый JSON
docker container ls --format '{{json .}}' | jq

# Условный вывод
docker container ls --format '{{if eq .Status "running"}}{{.Names}}{{end}}'
```

**Доступные поля для контейнеров:**
- `.ID` — ID контейнера
- `.Names` — имя контейнера
- `.Image` — имя образа
- `.Command` — команда
- `.CreatedAt` — время создания
- `.Status` — статус
- `.Ports` — порты
- `.Size` — размер
- `.Labels` — метки
- `.Mounts` — точки монтирования
- `.Networks` — сети

**Доступные поля для образов:**
- `.ID` — ID образа
- `.Repository` — репозиторий
- `.Tag` — тег
- `.Size` — размер
- `.CreatedAt` — время создания
- `.Digest` — дайджест

### Флаг --filter

Фильтрация результатов.

```bash
# Фильтры для контейнеров
docker container ls --filter "status=running"
docker container ls --filter "status=exited"
docker container ls --filter "name=web"
docker container ls --filter "ancestor=nginx"
docker container ls --filter "label=env=production"
docker container ls --filter "health=healthy"
docker container ls --filter "network=my_network"
docker container ls --filter "volume=my_volume"
docker container ls --filter "before=container_name"
docker container ls --filter "since=container_name"

# Множественные фильтры (AND)
docker container ls --filter "status=running" --filter "ancestor=nginx"

# Фильтры для образов
docker image ls --filter "dangling=true"
docker image ls --filter "label=maintainer=me"
docker image ls --filter "before=nginx:1.25"
docker image ls --filter "since=nginx:1.24"
docker image ls --filter "reference=nginx*"

# Фильтры для томов
docker volume ls --filter "dangling=true"
docker volume ls --filter "driver=local"
docker volume ls --filter "label=env=dev"
docker volume ls --filter "name=my_*"

# Фильтры для сетей
docker network ls --filter "driver=bridge"
docker network ls --filter "scope=local"
docker network ls --filter "type=custom"
docker network ls --filter "name=my_*"
```

---

## Best Practices работы с CLI

### 1. Именование ресурсов

```bash
# Всегда давайте понятные имена контейнерам
docker container run --name web-frontend nginx

# Используйте теги для образов
docker image build -t myapp:v1.2.3 .

# Используйте метки для организации
docker container run --label app=web --label env=prod nginx
```

### 2. Очистка ресурсов

```bash
# Регулярно очищайте неиспользуемые ресурсы
docker system prune -f

# Для production — осторожнее с томами
docker system prune  # без --volumes!
```

### 3. Используйте новый синтаксис

```bash
# Лучше
docker container ls
docker image ls

# Хуже (но работает)
docker ps
docker images
```

### 4. Логирование и мониторинг

```bash
# Всегда проверяйте логи при проблемах
docker container logs -f --tail 100 my_container

# Мониторьте ресурсы
docker container stats --no-stream
```

### 5. Используйте --rm для временных контейнеров

```bash
# Контейнер автоматически удалится после остановки
docker container run --rm -it alpine sh
```

### 6. Ограничивайте ресурсы

```bash
# Всегда задавайте лимиты для production
docker container run --memory 512m --cpus 1.5 nginx
```

---

## Полезные алиасы и хитрости

### Bash/Zsh алиасы

Добавьте в `~/.bashrc` или `~/.zshrc`:

```bash
# Основные алиасы
alias d='docker'
alias dc='docker compose'
alias dps='docker container ls'
alias dpsa='docker container ls -a'
alias di='docker image ls'
alias dv='docker volume ls'
alias dn='docker network ls'

# Часто используемые операции
alias dstop='docker container stop $(docker container ls -q)'
alias drm='docker container rm $(docker container ls -aq)'
alias drmi='docker image rm $(docker image ls -q)'
alias dprune='docker system prune -af --volumes'

# Быстрый exec
alias dex='docker container exec -it'

# Логи
alias dlogs='docker container logs -f'

# Статистика
alias dstats='docker container stats'

# Войти в контейнер
function dbash() {
    docker container exec -it $1 /bin/bash
}

function dsh() {
    docker container exec -it $1 /bin/sh
}

# Получить IP контейнера
function dip() {
    docker container inspect --format '{{.NetworkSettings.IPAddress}}' $1
}

# Остановить и удалить контейнер
function drm() {
    docker container stop $1 && docker container rm $1
}
```

### Полезные однострочники

```bash
# Остановить все контейнеры
docker container stop $(docker container ls -q)

# Удалить все остановленные контейнеры
docker container rm $(docker container ls -aq -f status=exited)

# Удалить все образы
docker image rm -f $(docker image ls -q)

# Удалить все dangling образы
docker image rm $(docker image ls -qf dangling=true)

# Удалить все тома
docker volume rm $(docker volume ls -q)

# Показать IP всех контейнеров
docker container inspect --format '{{.Name}}: {{.NetworkSettings.IPAddress}}' $(docker container ls -q)

# Показать размер контейнеров
docker container ls --size

# Показать запущенные контейнеры по образу
docker container ls --filter ancestor=nginx

# Следить за событиями Docker
docker events --filter 'type=container'

# Экспорт всех образов
for img in $(docker image ls --format '{{.Repository}}:{{.Tag}}'); do
    docker image save $img > $(echo $img | tr '/:' '_').tar
done

# Показать топ процессов во всех контейнерах
for container in $(docker container ls -q); do
    echo "=== $(docker container inspect --format '{{.Name}}' $container) ==="
    docker container top $container
done

# Очистка build cache
docker builder prune -f

# Показать слои образа с размерами
docker image history --no-trunc nginx | head -20

# Найти контейнер по его процессу
docker container ls --filter "status=running" --format '{{.Names}}' | xargs -I{} sh -c 'echo "=== {} ===" && docker container top {}'
```

### Автодополнение

Docker CLI поддерживает автодополнение. Настройка:

**Для Bash:**
```bash
# Установить docker-completion
sudo curl https://raw.githubusercontent.com/docker/cli/master/contrib/completion/bash/docker -o /etc/bash_completion.d/docker
```

**Для Zsh (с Oh My Zsh):**
```bash
# Добавьте в plugins в ~/.zshrc
plugins=(... docker docker-compose)
```

### Переменные окружения Docker

```bash
# Подключение к удалённому Docker daemon
export DOCKER_HOST=tcp://192.168.1.100:2375

# Использование TLS
export DOCKER_HOST=tcp://192.168.1.100:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=~/.docker/certs

# Формат вывода по умолчанию
export DOCKER_DEFAULT_FORMAT='table {{.Names}}\t{{.Status}}\t{{.Ports}}'

# Не использовать контекст по умолчанию
export DOCKER_CONTEXT=my-context
```

---

## Заключение

Docker CLI — мощный инструмент с богатым набором команд. Основные рекомендации:

1. **Используйте новый синтаксис** (`docker container ls` вместо `docker ps`) — он более понятный
2. **Создавайте алиасы** для часто используемых команд
3. **Используйте форматирование** (`--format`) для получения нужных данных
4. **Применяйте фильтры** (`--filter`) для поиска ресурсов
5. **Регулярно очищайте** неиспользуемые ресурсы с `docker system prune`
6. **Именуйте ресурсы** — это упрощает управление
7. **Используйте `--rm`** для временных контейнеров

Понимание Docker CLI — основа эффективной работы с контейнерами. С практикой команды станут привычными, и вы сможете быстро управлять любой Docker-инфраструктурой.
