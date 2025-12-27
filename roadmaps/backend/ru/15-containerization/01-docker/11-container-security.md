# Безопасность контейнеров Docker

## Введение в безопасность контейнеров

Безопасность контейнеров — это комплекс мер и практик, направленных на защиту контейнеризованных приложений на всех этапах их жизненного цикла: от создания образа до запуска в production-среде.

### Почему безопасность контейнеров важна?

1. **Общее ядро** — контейнеры разделяют ядро хост-системы, что создает потенциальную поверхность атаки
2. **Цепочка поставок** — уязвимости в базовых образах распространяются на все производные
3. **Эфемерность** — контейнеры создаются и уничтожаются динамически, что усложняет мониторинг
4. **Плотность развертывания** — на одном хосте может работать множество контейнеров

### Уровни безопасности контейнеров

```
┌─────────────────────────────────────────────────────────┐
│                    Приложение                           │
├─────────────────────────────────────────────────────────┤
│                    Контейнер                            │
├─────────────────────────────────────────────────────────┤
│              Container Runtime (Docker)                  │
├─────────────────────────────────────────────────────────┤
│                  Хост-система (Linux)                   │
├─────────────────────────────────────────────────────────┤
│                    Инфраструктура                       │
└─────────────────────────────────────────────────────────┘
```

Безопасность должна обеспечиваться на каждом уровне этого стека.

---

## Модель безопасности Docker

Docker использует несколько механизмов ядра Linux для изоляции контейнеров.

### Namespaces (пространства имен)

Namespaces обеспечивают изоляцию ресурсов, создавая отдельное представление системы для каждого контейнера.

| Namespace | Что изолирует | Описание |
|-----------|--------------|----------|
| **PID** | Процессы | Контейнер видит только свои процессы, PID 1 — главный процесс |
| **NET** | Сеть | Собственный сетевой стек, интерфейсы, таблицы маршрутизации |
| **MNT** | Файловая система | Изолированная точка монтирования |
| **UTS** | Hostname | Собственное имя хоста |
| **IPC** | IPC | Изолированная межпроцессная коммуникация |
| **USER** | Пользователи | Маппинг UID/GID между контейнером и хостом |
| **CGROUP** | Control Groups | Изоляция view на cgroups |

```bash
# Посмотреть namespaces контейнера
docker inspect --format '{{.State.Pid}}' my-container
ls -la /proc/<PID>/ns/

# Пример вывода
lrwxrwxrwx 1 root root 0 Dec 27 10:00 cgroup -> 'cgroup:[4026531835]'
lrwxrwxrwx 1 root root 0 Dec 27 10:00 ipc -> 'ipc:[4026532456]'
lrwxrwxrwx 1 root root 0 Dec 27 10:00 mnt -> 'mnt:[4026532454]'
lrwxrwxrwx 1 root root 0 Dec 27 10:00 net -> 'net:[4026532459]'
lrwxrwxrwx 1 root root 0 Dec 27 10:00 pid -> 'pid:[4026532457]'
lrwxrwxrwx 1 root root 0 Dec 27 10:00 user -> 'user:[4026531837]'
lrwxrwxrwx 1 root root 0 Dec 27 10:00 uts -> 'uts:[4026532455]'
```

### User Namespaces (remapping)

User namespace позволяет маппить root внутри контейнера на непривилегированного пользователя хоста:

```bash
# Включение user namespace remapping в /etc/docker/daemon.json
{
  "userns-remap": "default"
}

# Docker создаст пользователя dockremap
# UID 0 в контейнере = UID 100000+ на хосте
```

### Control Groups (cgroups)

Cgroups ограничивают и учитывают использование ресурсов контейнером.

```bash
# Ограничение памяти
docker run -d --memory=512m --memory-swap=512m nginx

# Ограничение CPU
docker run -d --cpus=1.5 nginx

# Ограничение CPU + память
docker run -d \
  --memory=1g \
  --memory-reservation=512m \
  --cpus=2 \
  --cpu-shares=1024 \
  nginx

# Просмотр ограничений cgroups
cat /sys/fs/cgroup/memory/docker/<container-id>/memory.limit_in_bytes
```

**Важные параметры cgroups:**

| Параметр | Описание |
|----------|----------|
| `--memory` | Жесткий лимит памяти |
| `--memory-reservation` | Мягкий лимит памяти |
| `--memory-swap` | Лимит swap (= memory для отключения swap) |
| `--cpus` | Количество CPU (дробное) |
| `--cpu-shares` | Относительный вес CPU |
| `--pids-limit` | Максимум процессов |

### Capabilities (возможности)

Linux capabilities разбивают привилегии root на отдельные единицы. Docker по умолчанию удаляет опасные capabilities.

```bash
# Capabilities, которые Docker оставляет по умолчанию:
# CAP_CHOWN, CAP_DAC_OVERRIDE, CAP_FSETID, CAP_FOWNER,
# CAP_MKNOD, CAP_NET_RAW, CAP_SETGID, CAP_SETUID,
# CAP_SETFCAP, CAP_SETPCAP, CAP_NET_BIND_SERVICE,
# CAP_SYS_CHROOT, CAP_KILL, CAP_AUDIT_WRITE

# Удалить все capabilities
docker run --cap-drop=ALL nginx

# Добавить только нужные
docker run --cap-drop=ALL --cap-add=NET_BIND_SERVICE nginx

# Запуск с привилегированным режимом (ОПАСНО!)
docker run --privileged nginx  # НЕ ИСПОЛЬЗОВАТЬ В PRODUCTION
```

**Опасные capabilities, которые Docker удаляет:**

| Capability | Риск |
|------------|------|
| `CAP_SYS_ADMIN` | Практически полный root — монтирование, namespace operations |
| `CAP_NET_ADMIN` | Управление сетью хоста |
| `CAP_SYS_PTRACE` | Отладка процессов, чтение памяти |
| `CAP_SYS_MODULE` | Загрузка модулей ядра |
| `CAP_SYS_RAWIO` | Прямой доступ к устройствам |

---

## Безопасность образов

### Использование официальных базовых образов

```dockerfile
# ПЛОХО: неизвестный источник
FROM some-random-user/python:latest

# ХОРОШО: официальный образ
FROM python:3.12-slim

# ЛУЧШЕ: образ с конкретной версией и дайджестом
FROM python:3.12.1-slim-bookworm@sha256:abc123...

# ОТЛИЧНО: минимальный официальный образ
FROM python:3.12-alpine
```

**Рекомендации по выбору базового образа:**

1. **Используйте официальные образы** с Docker Hub или проверенных registry
2. **Фиксируйте версии** — никогда не используйте `latest` в production
3. **Используйте дайджесты (SHA256)** для критичных приложений
4. **Выбирайте минимальные образы** — alpine, slim, distroless

### Сканирование на уязвимости

#### Trivy (Aqua Security)

Trivy — популярный open-source сканер уязвимостей:

```bash
# Установка
brew install trivy  # macOS
apt install trivy   # Debian/Ubuntu

# Сканирование образа
trivy image nginx:latest

# Сканирование с фильтрацией по severity
trivy image --severity HIGH,CRITICAL nginx:latest

# Сканирование Dockerfile
trivy config ./Dockerfile

# Сканирование файловой системы (dependencies)
trivy fs --security-checks vuln,config ./

# Формат вывода JSON для CI/CD
trivy image --format json --output results.json nginx:latest

# Fail если есть критические уязвимости
trivy image --exit-code 1 --severity CRITICAL nginx:latest
```

**Пример вывода Trivy:**

```
nginx:latest (debian 12.4)
===========================
Total: 142 (UNKNOWN: 0, LOW: 87, MEDIUM: 41, HIGH: 12, CRITICAL: 2)

┌──────────────────┬────────────────┬──────────┬───────────────────┐
│     Library      │ Vulnerability  │ Severity │  Fixed Version    │
├──────────────────┼────────────────┼──────────┼───────────────────┤
│ libssl3          │ CVE-2024-0727  │ CRITICAL │ 3.0.13-1~deb12u1  │
│ openssl          │ CVE-2024-0727  │ CRITICAL │ 3.0.13-1~deb12u1  │
└──────────────────┴────────────────┴──────────┴───────────────────┘
```

#### Snyk

```bash
# Установка Snyk CLI
npm install -g snyk
snyk auth

# Сканирование образа
snyk container test nginx:latest

# Мониторинг (отслеживание новых уязвимостей)
snyk container monitor nginx:latest

# Сканирование Dockerfile
snyk container test --file=Dockerfile nginx:latest
```

#### Docker Scout (встроенный в Docker)

```bash
# Сканирование образа
docker scout cves nginx:latest

# Подробный анализ
docker scout quickview nginx:latest

# Сравнение образов
docker scout compare nginx:1.24 nginx:1.25
```

#### Clair (CoreOS)

Clair работает как сервис для сканирования образов:

```yaml
# docker-compose.yml для Clair
version: '3'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: clair

  clair:
    image: quay.io/projectquay/clair:4.7
    depends_on:
      - postgres
    ports:
      - "6060:6060"
```

#### Интеграция в CI/CD (GitHub Actions)

```yaml
# .github/workflows/security.yml
name: Container Security Scan

on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - name: Build image
        run: docker build -t myapp:${{ github.sha }} .

      - name: Run Trivy vulnerability scanner
        uses: aquasecurity/trivy-action@master
        with:
          image-ref: 'myapp:${{ github.sha }}'
          format: 'sarif'
          output: 'trivy-results.sarif'
          severity: 'CRITICAL,HIGH'

      - name: Upload Trivy scan results
        uses: github/codeql-action/upload-sarif@v2
        with:
          sarif_file: 'trivy-results.sarif'
```

### Минимизация образов

#### Alpine Linux

Alpine — минималистичный дистрибутив (~5MB):

```dockerfile
# Стандартный Python образ: ~1GB
FROM python:3.12

# Alpine Python: ~50MB
FROM python:3.12-alpine

# Установка зависимостей в Alpine
RUN apk add --no-cache \
    gcc \
    musl-dev \
    libffi-dev

COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
```

**Особенности Alpine:**
- Использует musl libc вместо glibc
- Пакетный менеджер apk
- Минимальный размер
- Некоторые библиотеки могут требовать дополнительной компиляции

#### Distroless (Google)

Distroless образы содержат только runtime приложения — нет shell, package manager:

```dockerfile
# Multi-stage build с distroless
FROM python:3.12-slim AS builder

WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir --target=/app/deps -r requirements.txt
COPY . .

# Финальный образ — distroless
FROM gcr.io/distroless/python3-debian12

WORKDIR /app
COPY --from=builder /app/deps /app/deps
COPY --from=builder /app .

ENV PYTHONPATH=/app/deps
CMD ["main.py"]
```

**Distroless для разных языков:**

```dockerfile
# Java
FROM gcr.io/distroless/java21-debian12

# Node.js
FROM gcr.io/distroless/nodejs20-debian12

# Go (статически скомпилированный бинарник)
FROM gcr.io/distroless/static-debian12

# С отладочной оболочкой (для troubleshooting)
FROM gcr.io/distroless/python3-debian12:debug
```

#### Scratch (пустой образ)

Для статически скомпилированных приложений:

```dockerfile
# Go приложение
FROM golang:1.22-alpine AS builder

WORKDIR /app
COPY . .
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o main .

# Финальный образ — полностью пустой
FROM scratch

COPY --from=builder /app/main /main
COPY --from=builder /etc/ssl/certs/ca-certificates.crt /etc/ssl/certs/

ENTRYPOINT ["/main"]
```

#### Сравнение размеров образов

| Базовый образ | Размер | Уязвимости* |
|--------------|--------|-------------|
| ubuntu:22.04 | ~77MB | ~30-50 |
| debian:bookworm-slim | ~80MB | ~40-60 |
| python:3.12 | ~1GB | ~200+ |
| python:3.12-slim | ~150MB | ~50-80 |
| python:3.12-alpine | ~50MB | ~10-20 |
| distroless/python3 | ~50MB | ~5-15 |
| scratch | 0MB | 0 |

*Примерные значения, зависят от версии

---

## Безопасность runtime

### Запуск от non-root пользователя

**ВАЖНО:** Никогда не запускайте контейнеры от root в production!

```dockerfile
# Создание пользователя в Dockerfile
FROM python:3.12-slim

# Создаем пользователя и группу
RUN groupadd --gid 1000 appgroup && \
    useradd --uid 1000 --gid appgroup --shell /bin/bash --create-home appuser

# Создаем директории и устанавливаем права
WORKDIR /app
COPY --chown=appuser:appgroup . .

# Переключаемся на пользователя
USER appuser

# Или использовать числовой UID (более безопасно)
USER 1000:1000

CMD ["python", "main.py"]
```

**Проверка пользователя в контейнере:**

```bash
# Запуск с указанием пользователя
docker run --user 1000:1000 nginx

# Проверка кто запустил процесс
docker exec container_name whoami
docker exec container_name id
```

### Read-only файловая система

```bash
# Запуск с read-only файловой системой
docker run --read-only nginx

# С tmpfs для временных файлов
docker run --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=100m \
  --tmpfs /var/run:rw,noexec,nosuid \
  nginx

# В docker-compose
services:
  web:
    image: nginx
    read_only: true
    tmpfs:
      - /tmp
      - /var/run
```

**Dockerfile для read-only:**

```dockerfile
FROM nginx:alpine

# Создаем нужные директории заранее
RUN mkdir -p /var/cache/nginx /var/run && \
    chown -R nginx:nginx /var/cache/nginx /var/run

USER nginx
```

### Ограничение capabilities

```bash
# Минимальные capabilities
docker run \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --cap-add=CHOWN \
  --cap-add=SETGID \
  --cap-add=SETUID \
  nginx
```

**Типичные наборы capabilities:**

```bash
# Веб-сервер
--cap-drop=ALL --cap-add=NET_BIND_SERVICE --cap-add=CHOWN --cap-add=SETGID --cap-add=SETUID

# Приложение без привилегий
--cap-drop=ALL

# Мониторинг (нужен ptrace)
--cap-drop=ALL --cap-add=SYS_PTRACE

# Сетевая диагностика
--cap-drop=ALL --cap-add=NET_RAW --cap-add=NET_ADMIN
```

### Seccomp профили

Seccomp (secure computing mode) фильтрует системные вызовы:

```bash
# Docker использует default seccomp профиль
# Блокирует ~44 опасных syscalls из ~300+

# Просмотр default профиля
docker info --format '{{json .SecurityOptions}}'

# Запуск без seccomp (ОПАСНО!)
docker run --security-opt seccomp=unconfined nginx

# Запуск с кастомным профилем
docker run --security-opt seccomp=/path/to/profile.json nginx
```

**Пример кастомного seccomp профиля:**

```json
{
  "defaultAction": "SCMP_ACT_ERRNO",
  "defaultErrnoRet": 1,
  "architectures": [
    "SCMP_ARCH_X86_64",
    "SCMP_ARCH_X86"
  ],
  "syscalls": [
    {
      "names": [
        "read",
        "write",
        "open",
        "close",
        "stat",
        "fstat",
        "mmap",
        "mprotect",
        "munmap",
        "brk",
        "rt_sigaction",
        "rt_sigprocmask",
        "ioctl",
        "access",
        "pipe",
        "select",
        "sched_yield",
        "dup",
        "dup2",
        "socket",
        "connect",
        "accept",
        "bind",
        "listen",
        "exit",
        "exit_group"
      ],
      "action": "SCMP_ACT_ALLOW"
    }
  ]
}
```

**Создание профиля на основе strace:**

```bash
# Запись системных вызовов приложения
strace -c -f ./myapp 2>&1 | grep -v "^strace"

# Использование OCI seccomp profiler
docker run --security-opt seccomp=./profile.json myimage
```

### AppArmor профили

AppArmor — система MAC (Mandatory Access Control) для Linux:

```bash
# Проверка поддержки AppArmor
cat /sys/module/apparmor/parameters/enabled

# Просмотр профилей
sudo aa-status

# Docker default профиль
cat /etc/apparmor.d/docker

# Запуск с кастомным профилем
docker run --security-opt apparmor=my-custom-profile nginx

# Запуск без AppArmor (ОПАСНО!)
docker run --security-opt apparmor=unconfined nginx
```

**Пример AppArmor профиля:**

```
#include <tunables/global>

profile my-nginx-profile flags=(attach_disconnected) {
  #include <abstractions/base>
  #include <abstractions/nameservice>

  # Разрешить сетевые операции
  network inet tcp,
  network inet udp,

  # Разрешить чтение конфигов
  /etc/nginx/** r,
  /usr/share/nginx/** r,

  # Разрешить запись логов
  /var/log/nginx/** rw,

  # Разрешить запуск nginx
  /usr/sbin/nginx ix,

  # Запретить все остальное
  deny /etc/shadow r,
  deny /etc/passwd w,
}
```

### Полный пример безопасного запуска

```bash
docker run -d \
  --name secure-app \
  --user 1000:1000 \
  --read-only \
  --tmpfs /tmp:rw,noexec,nosuid,size=50m \
  --cap-drop=ALL \
  --cap-add=NET_BIND_SERVICE \
  --security-opt=no-new-privileges:true \
  --security-opt seccomp=./seccomp-profile.json \
  --security-opt apparmor=docker-default \
  --memory=512m \
  --cpus=1 \
  --pids-limit=100 \
  --restart=on-failure:5 \
  myapp:latest
```

**В docker-compose.yml:**

```yaml
version: '3.8'

services:
  secure-app:
    image: myapp:latest
    user: "1000:1000"
    read_only: true
    tmpfs:
      - /tmp:rw,noexec,nosuid,size=50m
    cap_drop:
      - ALL
    cap_add:
      - NET_BIND_SERVICE
    security_opt:
      - no-new-privileges:true
      - seccomp:./seccomp-profile.json
    deploy:
      resources:
        limits:
          memory: 512M
          cpus: '1.0'
          pids: 100
    restart: on-failure
```

---

## Безопасность сети

### Изоляция сетей

```bash
# Создание изолированной сети
docker network create --driver bridge --internal isolated-network

# --internal означает отсутствие доступа в интернет

# Создание сетей для разных сред
docker network create frontend-net
docker network create backend-net --internal
docker network create db-net --internal
```

**Архитектура с сетевой изоляцией:**

```yaml
# docker-compose.yml
version: '3.8'

services:
  nginx:
    image: nginx
    networks:
      - frontend
    ports:
      - "80:80"

  api:
    image: myapi
    networks:
      - frontend
      - backend
    # НЕТ ports — недоступен снаружи

  database:
    image: postgres
    networks:
      - backend
    # Изолирована от frontend

networks:
  frontend:
    driver: bridge
  backend:
    driver: bridge
    internal: true  # Нет доступа в интернет
```

### Ограничение портов

```bash
# Привязка только к localhost
docker run -p 127.0.0.1:3000:3000 myapp

# НЕ делать: открытие для всех
docker run -p 3000:3000 myapp  # 0.0.0.0:3000

# Ограничение исходящих соединений через iptables
iptables -I DOCKER-USER -s 172.17.0.0/16 -j DROP
iptables -I DOCKER-USER -s 172.17.0.0/16 -d allowed-ip -j ACCEPT
```

### Network Policies (в Docker Swarm/Kubernetes)

```yaml
# Для Docker Swarm используйте overlay сети
docker network create --driver overlay --attachable secure-overlay

# Для Kubernetes используйте NetworkPolicy (отдельная тема)
```

### Отключение ICC (inter-container communication)

```bash
# В /etc/docker/daemon.json
{
  "icc": false
}

# Теперь контейнеры не могут общаться напрямую
# Нужно явно связывать через --link или docker-compose
```

---

## Secrets Management

### Docker Secrets (Swarm mode)

```bash
# Создание секрета
echo "my-super-secret-password" | docker secret create db_password -

# Создание из файла
docker secret create ssl_cert ./server.crt

# Просмотр секретов
docker secret ls

# Использование в сервисе
docker service create \
  --name db \
  --secret db_password \
  postgres

# Секрет доступен в /run/secrets/db_password
```

**docker-compose.yml с секретами:**

```yaml
version: '3.8'

services:
  db:
    image: postgres
    secrets:
      - db_password
    environment:
      POSTGRES_PASSWORD_FILE: /run/secrets/db_password

secrets:
  db_password:
    external: true  # Создан заранее

  # Или из файла (только для development!)
  db_password:
    file: ./secrets/db_password.txt
```

### Переменные окружения (НЕ безопасно для секретов!)

```bash
# ПЛОХО: секрет виден в docker inspect
docker run -e DB_PASSWORD=secret123 myapp

# ПЛОХО: секрет в Dockerfile
ENV API_KEY=abc123

# ЧУТЬ ЛУЧШЕ: передача через файл .env (не коммитить!)
docker run --env-file .env myapp
```

### Vault (HashiCorp)

```dockerfile
# Интеграция с Vault
FROM python:3.12-slim

RUN pip install hvac  # Python client для Vault

COPY app.py .

CMD ["python", "app.py"]
```

```python
# app.py
import hvac
import os

client = hvac.Client(url=os.environ['VAULT_ADDR'])
client.token = os.environ['VAULT_TOKEN']

# Чтение секрета
secret = client.secrets.kv.read_secret_version(path='myapp/db')
db_password = secret['data']['data']['password']
```

### AWS Secrets Manager / GCP Secret Manager

```bash
# AWS Secrets Manager
aws secretsmanager get-secret-value --secret-id myapp/db-password

# В контейнере через IAM роль
docker run \
  --env AWS_REGION=us-east-1 \
  myapp
```

### Безопасное хранение секретов — чек-лист

1. **Никогда** не храните секреты в Dockerfile или образе
2. **Никогда** не коммитьте секреты в git
3. Используйте `.dockerignore` для исключения файлов секретов
4. Используйте Docker Secrets или внешний vault
5. Ротируйте секреты регулярно
6. Используйте принцип минимальных привилегий

---

## Docker Content Trust (подпись образов)

Docker Content Trust (DCT) обеспечивает целостность и подлинность образов через криптографические подписи.

### Включение DCT

```bash
# Глобально
export DOCKER_CONTENT_TRUST=1

# Или в /etc/docker/daemon.json
{
  "content-trust": {
    "mode": "enforced"
  }
}
```

### Подпись образов

```bash
# Включаем DCT
export DOCKER_CONTENT_TRUST=1

# При первом push создаются ключи
docker push myregistry/myimage:1.0

# Вас попросят создать:
# 1. Root key (offline key) — хранить очень безопасно!
# 2. Repository key — для подписи тегов

# Просмотр подписей
docker trust inspect myregistry/myimage

# Добавление подписчика (делегирование)
docker trust signer add --key cert.pem alice myregistry/myimage
```

### Notary

Docker Content Trust использует Notary для управления подписями:

```bash
# Просмотр информации через notary
notary -d ~/.docker/trust list docker.io/library/nginx

# Инициализация репозитория
notary -d ~/.docker/trust init docker.io/myrepo/myimage
```

### Проверка подписей

```bash
# С включенным DCT, неподписанные образы блокируются
export DOCKER_CONTENT_TRUST=1
docker pull unsigned-image  # Ошибка!

# Проверка конкретного образа
docker trust inspect --pretty nginx:latest
```

**Вывод docker trust inspect:**

```
Signatures for nginx:latest

SIGNED TAG    DIGEST                                                             SIGNERS
latest        abc123...                                                          Repo Admin

Administrative keys for nginx:latest

  Repository Key: abc...
  Root Key:       def...
```

---

## Best Practices безопасности

### Dockerfile Best Practices

```dockerfile
# 1. Используйте конкретные версии
FROM python:3.12.1-slim-bookworm@sha256:abc123...

# 2. Не запускайте от root
RUN groupadd -r appgroup && useradd -r -g appgroup appuser

# 3. Минимизируйте слои и очищайте кэш
RUN apt-get update && \
    apt-get install -y --no-install-recommends \
      package1 \
      package2 && \
    apt-get clean && \
    rm -rf /var/lib/apt/lists/*

# 4. Используйте COPY вместо ADD
COPY requirements.txt .

# 5. Не храните секреты
# ПЛОХО: ENV API_KEY=secret
# ХОРОШО: используйте Docker Secrets или mount

# 6. Используйте .dockerignore
# 7. Проверяйте HEALTHCHECK
HEALTHCHECK --interval=30s --timeout=3s \
  CMD curl -f http://localhost/health || exit 1

# 8. Устанавливайте USER в конце
USER appuser

# 9. Используйте multi-stage builds
```

**.dockerignore:**

```
.git
.env
*.secret
secrets/
node_modules/
__pycache__/
*.pyc
.pytest_cache/
coverage/
*.log
Dockerfile
docker-compose*.yml
README.md
```

### Runtime Best Practices

```bash
# Полный безопасный запуск
docker run -d \
  --name app \
  --user 1000:1000 \
  --read-only \
  --tmpfs /tmp \
  --cap-drop ALL \
  --cap-add NET_BIND_SERVICE \
  --security-opt no-new-privileges:true \
  --security-opt seccomp=default \
  --memory 512m \
  --cpus 1 \
  --pids-limit 100 \
  --restart on-failure:5 \
  --health-cmd "curl -f http://localhost/health" \
  --health-interval 30s \
  --network isolated-network \
  app:1.0
```

### Чек-лист безопасности Docker

#### Образы
- [ ] Используются официальные/проверенные базовые образы
- [ ] Образы сканируются на уязвимости в CI/CD
- [ ] Используются конкретные версии и SHA256 дайджесты
- [ ] Минимальный размер образа (alpine/distroless)
- [ ] Multi-stage builds для production
- [ ] Включен Docker Content Trust

#### Runtime
- [ ] Контейнеры запускаются от non-root
- [ ] Read-only файловая система где возможно
- [ ] Capabilities ограничены (--cap-drop ALL)
- [ ] Seccomp профиль включен
- [ ] Ресурсы ограничены (memory, cpu, pids)
- [ ] no-new-privileges включен

#### Сеть
- [ ] Используются изолированные сети
- [ ] Порты привязаны к localhost где возможно
- [ ] Внутренние сервисы не экспонируют порты

#### Секреты
- [ ] Секреты не хранятся в образах
- [ ] Используются Docker Secrets или внешний vault
- [ ] .dockerignore настроен правильно

#### Инфраструктура
- [ ] Docker daemon socket защищен
- [ ] Регулярные обновления Docker
- [ ] Логирование и мониторинг настроены
- [ ] Аудит безопасности проводится регулярно

---

## Инструменты аудита безопасности

### Docker Bench Security

Автоматическая проверка соответствия CIS Docker Benchmark:

```bash
# Запуск Docker Bench
docker run --rm --net host --pid host --userns host --cap-add audit_control \
  -e DOCKER_CONTENT_TRUST=$DOCKER_CONTENT_TRUST \
  -v /etc:/etc:ro \
  -v /usr/bin/containerd:/usr/bin/containerd:ro \
  -v /usr/bin/runc:/usr/bin/runc:ro \
  -v /usr/lib/systemd:/usr/lib/systemd:ro \
  -v /var/lib:/var/lib:ro \
  -v /var/run/docker.sock:/var/run/docker.sock:ro \
  docker/docker-bench-security

# Пример вывода:
# [INFO] 1 - Host Configuration
# [PASS] 1.1.1 - Ensure a separate partition for containers has been created
# [WARN] 1.1.2 - Ensure the container host has been Hardened
# [INFO] 1.2 - Linux Hosts Specific Configuration
# ...
```

### Trivy (комплексный сканер)

```bash
# Сканирование образа
trivy image nginx:latest

# Сканирование Dockerfile (misconfigurations)
trivy config ./Dockerfile

# Сканирование кластера Kubernetes
trivy k8s --report summary cluster

# Генерация SBOM (Software Bill of Materials)
trivy sbom nginx:latest --format spdx-json
```

### Falco (runtime security)

Falco мониторит syscalls в runtime:

```yaml
# docker-compose.yml для Falco
version: '3'
services:
  falco:
    image: falcosecurity/falco
    privileged: true
    volumes:
      - /var/run/docker.sock:/var/run/docker.sock
      - /dev:/host/dev
      - /proc:/host/proc:ro
      - /boot:/host/boot:ro
      - /lib/modules:/host/lib/modules:ro
      - /usr:/host/usr:ro
      - /etc:/host/etc:ro
```

**Пример правила Falco:**

```yaml
- rule: Terminal shell in container
  desc: Detect a shell being spawned in a container
  condition: >
    spawned_process and container and shell_procs
  output: >
    Shell spawned in container (user=%user.name container=%container.name
    shell=%proc.name parent=%proc.pname cmdline=%proc.cmdline)
  priority: WARNING
```

### Anchore Engine

```bash
# Установка Anchore
docker-compose -f docker-compose-anchore.yml up -d

# Добавление образа для анализа
anchore-cli image add nginx:latest

# Проверка уязвимостей
anchore-cli image vuln nginx:latest os

# Проверка policy compliance
anchore-cli evaluate check nginx:latest
```

### Snyk Container

```bash
# Сканирование
snyk container test nginx:latest

# Мониторинг с уведомлениями
snyk container monitor nginx:latest --project-name=production-nginx

# Интеграция в CI
snyk container test nginx:latest --severity-threshold=high --fail-on=upgradable
```

### Hadolint (линтер Dockerfile)

```bash
# Установка
brew install hadolint

# Проверка Dockerfile
hadolint Dockerfile

# Пример вывода
# Dockerfile:3 DL3008 warning: Pin versions in apt-get install
# Dockerfile:5 DL3009 info: Delete apt-get lists after installing
# Dockerfile:10 DL3003 warning: Use WORKDIR instead of cd

# Игнорирование правил
hadolint --ignore DL3008 Dockerfile

# В CI (GitHub Actions)
- name: Lint Dockerfile
  uses: hadolint/hadolint-action@v3.1.0
  with:
    dockerfile: Dockerfile
```

### Dockle (линтер образов)

```bash
# Установка
brew install goodwithtech/r/dockle

# Проверка образа
dockle nginx:latest

# Пример вывода:
# FATAL - CIS-DI-0001: Create a user for the container
# WARN  - CIS-DI-0005: Enable Content trust for Docker
# INFO  - CIS-DI-0006: Add HEALTHCHECK instruction
# PASS  - CIS-DI-0007: Do not use update instructions alone
```

### Сводная таблица инструментов

| Инструмент | Тип | Что проверяет |
|------------|-----|---------------|
| **Trivy** | Scanner | Уязвимости, misconfig, SBOM |
| **Snyk** | Scanner | Уязвимости, лицензии |
| **Docker Scout** | Scanner | Уязвимости, recommendations |
| **Clair** | Scanner | Уязвимости |
| **Anchore** | Scanner | Уязвимости, policies |
| **Docker Bench** | Audit | CIS Benchmark compliance |
| **Falco** | Runtime | Syscall monitoring |
| **Hadolint** | Linter | Dockerfile best practices |
| **Dockle** | Linter | Image best practices |

---

## Заключение

Безопасность контейнеров Docker — это многоуровневый процесс, включающий:

1. **Безопасные образы** — официальные, минимальные, просканированные
2. **Безопасный runtime** — non-root, read-only, ограниченные capabilities
3. **Изоляция** — сетевая, resource limits, namespaces
4. **Управление секретами** — Docker Secrets, Vault
5. **Подпись и верификация** — Docker Content Trust
6. **Мониторинг и аудит** — регулярное сканирование, runtime protection

Применение этих практик значительно снижает риски эксплуатации уязвимостей и минимизирует потенциальный ущерб от компрометации.

### Полезные ресурсы

- [CIS Docker Benchmark](https://www.cisecurity.org/benchmark/docker)
- [Docker Security Documentation](https://docs.docker.com/engine/security/)
- [OWASP Docker Security Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Docker_Security_Cheat_Sheet.html)
- [Aqua Security Blog](https://blog.aquasec.com/)
- [Falco Documentation](https://falco.org/docs/)
