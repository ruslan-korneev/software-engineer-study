# Underlying Technologies (Базовые технологии Docker)

Docker — это не просто программа, а комплекс технологий Linux, объединённых в удобный инструмент. Понимание этих технологий поможет глубже разобраться в работе контейнеров и решать сложные проблемы.

---

## 1. Linux Namespaces

**Namespaces** — механизм ядра Linux, обеспечивающий изоляцию ресурсов. Каждый контейнер получает собственный "взгляд" на систему.

### Типы Namespaces

| Namespace | Флаг | Что изолирует |
|-----------|------|---------------|
| **PID** | `CLONE_NEWPID` | Дерево процессов. Процессы внутри контейнера видят только себя (PID 1 — init процесс контейнера) |
| **Network** | `CLONE_NEWNET` | Сетевой стек: интерфейсы, IP-адреса, порты, маршруты, iptables |
| **Mount** | `CLONE_NEWNS` | Точки монтирования. Контейнер видит только свою файловую систему |
| **UTS** | `CLONE_NEWUTS` | Hostname и domain name. Контейнер может иметь своё имя хоста |
| **IPC** | `CLONE_NEWIPC` | Inter-Process Communication: очереди сообщений, семафоры, shared memory |
| **User** | `CLONE_NEWUSER` | UID/GID mapping. Root внутри контейнера ≠ root на хосте |
| **Cgroup** | `CLONE_NEWCGROUP` | Изоляция иерархии cgroups |

### Пример: просмотр namespaces процесса

```bash
# Просмотр namespaces для процесса
ls -la /proc/<PID>/ns/

# Пример вывода:
# lrwxrwxrwx 1 root root 0 Dec 27 10:00 cgroup -> 'cgroup:[4026531835]'
# lrwxrwxrwx 1 root root 0 Dec 27 10:00 ipc -> 'ipc:[4026532486]'
# lrwxrwxrwx 1 root root 0 Dec 27 10:00 mnt -> 'mnt:[4026532484]'
# lrwxrwxrwx 1 root root 0 Dec 27 10:00 net -> 'net:[4026532489]'
# lrwxrwxrwx 1 root root 0 Dec 27 10:00 pid -> 'pid:[4026532487]'
# lrwxrwxrwx 1 root root 0 Dec 27 10:00 user -> 'user:[4026531837]'
# lrwxrwxrwx 1 root root 0 Dec 27 10:00 uts -> 'uts:[4026532485]'
```

### Как Docker использует namespaces

```bash
# Вход в namespace контейнера
docker exec -it <container_id> bash

# Под капотом Docker использует системный вызов:
# unshare() — создание нового namespace
# setns() — присоединение к существующему namespace
```

---

## 2. Control Groups (cgroups)

**Cgroups** — механизм ядра Linux для ограничения, учёта и изоляции ресурсов (CPU, память, I/O, сеть).

### Основные контроллеры cgroups

| Контроллер | Назначение |
|------------|------------|
| `cpu` | Ограничение времени CPU |
| `cpuset` | Привязка к конкретным ядрам |
| `memory` | Лимиты памяти и swap |
| `blkio` | Ограничение I/O к блочным устройствам |
| `devices` | Контроль доступа к устройствам |
| `pids` | Ограничение количества процессов |
| `net_cls` | Классификация сетевых пакетов |

### Примеры использования в Docker

```bash
# Ограничение памяти (512MB)
docker run -m 512m nginx

# Ограничение CPU (0.5 ядра)
docker run --cpus="0.5" nginx

# Ограничение количества процессов
docker run --pids-limit 100 nginx

# Привязка к конкретным ядрам CPU
docker run --cpuset-cpus="0,1" nginx

# Просмотр лимитов контейнера
docker stats <container_id>
```

### Иерархия cgroups v2

```bash
# Расположение cgroups в системе
/sys/fs/cgroup/

# Пример структуры для Docker контейнера:
/sys/fs/cgroup/docker/<container_id>/
├── cpu.max           # CPU limits
├── memory.max        # Memory limit
├── memory.current    # Current memory usage
├── pids.max          # Max processes
└── pids.current      # Current processes count
```

### Cgroups v1 vs v2

| Характеристика | cgroups v1 | cgroups v2 |
|---------------|------------|------------|
| Иерархия | Множество деревьев (по контроллеру) | Единое дерево |
| Сложность | Высокая | Упрощённая |
| eBPF поддержка | Ограниченная | Полная |
| Rootless containers | Проблематично | Нативная поддержка |

---

## 3. Union File Systems (UnionFS)

**Union File System** — технология объединения нескольких файловых систем в одну с поддержкой слоёв (layers). Это основа эффективности Docker образов.

### Как работает слоистая ФС

```
┌─────────────────────────────────────┐
│     Container Layer (R/W)           │  <- Все изменения пишутся сюда
├─────────────────────────────────────┤
│     Layer 4: COPY app.py            │  <- Read-only
├─────────────────────────────────────┤
│     Layer 3: RUN pip install        │  <- Read-only
├─────────────────────────────────────┤
│     Layer 2: RUN apt-get update     │  <- Read-only
├─────────────────────────────────────┤
│     Layer 1: Base Image (ubuntu)    │  <- Read-only
└─────────────────────────────────────┘
```

### Типы Union File Systems

#### OverlayFS (overlay2) — стандарт в современном Docker

```bash
# Проверка текущего storage driver
docker info | grep "Storage Driver"
# Storage Driver: overlay2

# Структура overlay2
/var/lib/docker/overlay2/
├── <layer-id>/
│   ├── diff/      # Содержимое слоя
│   ├── link       # Короткий ID
│   ├── lower      # Ссылки на нижние слои
│   ├── merged/    # Объединённый вид (только для контейнеров)
│   └── work/      # Рабочая директория overlay
```

#### Принцип Copy-on-Write (CoW)

```
Чтение файла:
1. Поиск файла сверху вниз по слоям
2. Возврат первого найденного экземпляра

Запись файла:
1. Если файл в read-only слое — копирование в R/W слой
2. Модификация копии
3. Оригинал в нижнем слое остаётся неизменным

Удаление файла:
1. Создаётся "whiteout" файл в R/W слое
2. Файл из нижних слоёв становится невидимым
```

### Сравнение storage drivers

| Driver | Backing FS | Производительность | Статус |
|--------|-----------|-------------------|--------|
| **overlay2** | xfs, ext4 | Высокая | Рекомендуется |
| **btrfs** | btrfs | Хорошая | Поддерживается |
| **zfs** | zfs | Хорошая | Поддерживается |
| **devicemapper** | direct-lvm | Средняя | Deprecated |
| **aufs** | ext4, xfs | Средняя | Deprecated |

---

## 4. Container Runtime

**Container Runtime** — программа, непосредственно запускающая контейнеры. Docker использует многоуровневую архитектуру.

### Уровни Container Runtime

```
┌──────────────────────────────────────────────┐
│              Docker CLI (docker)              │
│         Пользовательский интерфейс            │
└───────────────────────┬──────────────────────┘
                        │ REST API
                        ▼
┌──────────────────────────────────────────────┐
│            Docker Daemon (dockerd)            │
│      Управление образами, сетями, volumes     │
└───────────────────────┬──────────────────────┘
                        │ gRPC
                        ▼
┌──────────────────────────────────────────────┐
│              containerd                       │
│    High-level runtime: lifecycle, storage     │
└───────────────────────┬──────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────┐
│         containerd-shim                       │
│   Промежуточный процесс для каждого контейнера│
└───────────────────────┬──────────────────────┘
                        │
                        ▼
┌──────────────────────────────────────────────┐
│                  runc                         │
│    Low-level runtime: создание контейнера     │
│    (namespaces, cgroups, seccomp, etc.)       │
└──────────────────────────────────────────────┘
```

### containerd

**containerd** — high-level container runtime, управляющий жизненным циклом контейнеров.

```bash
# containerd управляет:
# - Загрузкой и хранением образов
# - Запуском и остановкой контейнеров
# - Сетью (через CNI плагины)
# - Хранилищем (snapshots)

# CLI для containerd
ctr images list
ctr containers list
ctr tasks list
```

### runc

**runc** — low-level runtime, реализующий OCI Runtime Specification.

```bash
# runc непосредственно:
# - Создаёт namespaces
# - Настраивает cgroups
# - Применяет seccomp профили
# - Запускает процесс контейнера

# Пример использования runc напрямую
runc spec                    # Создать config.json
runc create <container-id>   # Создать контейнер
runc start <container-id>    # Запустить
runc delete <container-id>   # Удалить
```

### containerd-shim

**Shim** — процесс-посредник между containerd и runc.

Зачем нужен shim:
- **Независимость от containerd**: контейнеры продолжают работать при перезапуске containerd
- **Сбор exit-кодов**: shim ожидает завершения процесса
- **Управление stdin/stdout**: перенаправление I/O
- **SIGCHLD обработка**: reaping zombie processes

---

## 5. Docker Daemon и Docker Client

### Архитектура Docker

```
┌────────────────────────────────────────────────────────────┐
│                        Клиент                               │
│  ┌─────────┐ ┌─────────┐ ┌─────────┐ ┌─────────────────┐   │
│  │docker   │ │docker   │ │Docker   │ │ Docker SDK      │   │
│  │CLI      │ │compose  │ │Desktop  │ │ (Python, Go...) │   │
│  └────┬────┘ └────┬────┘ └────┬────┘ └───────┬─────────┘   │
└───────┼──────────┼──────────┼───────────────┼──────────────┘
        │          │          │               │
        └──────────┴──────────┴───────────────┘
                          │
                    REST API (HTTP/Unix socket)
                          │
                          ▼
┌────────────────────────────────────────────────────────────┐
│                    Docker Daemon (dockerd)                  │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ Image        │ │ Container    │ │ Network      │        │
│  │ Management   │ │ Management   │ │ Management   │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
│  ┌──────────────┐ ┌──────────────┐ ┌──────────────┐        │
│  │ Volume       │ │ Build        │ │ Plugin       │        │
│  │ Management   │ │ Engine       │ │ System       │        │
│  └──────────────┘ └──────────────┘ └──────────────┘        │
└────────────────────────────────────────────────────────────┘
```

### Docker Client

```bash
# Docker CLI общается с daemon через REST API

# Unix socket (по умолчанию)
/var/run/docker.sock

# TCP socket (для удалённого доступа)
tcp://192.168.1.100:2376

# Переменная окружения для удалённого docker
export DOCKER_HOST=tcp://192.168.1.100:2376
export DOCKER_TLS_VERIFY=1
export DOCKER_CERT_PATH=/path/to/certs

# Пример API запроса
curl --unix-socket /var/run/docker.sock http://localhost/containers/json
```

### Docker Daemon (dockerd)

```bash
# Конфигурация daemon
# /etc/docker/daemon.json
{
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {"base": "172.17.0.0/16", "size": 24}
  ],
  "live-restore": true,
  "userland-proxy": false
}

# Перезапуск daemon
sudo systemctl restart docker

# Просмотр логов daemon
journalctl -u docker.service -f
```

---

## 6. OCI (Open Container Initiative)

**OCI** — организация, разрабатывающая открытые стандарты для контейнеров. Обеспечивает совместимость между различными runtime и registry.

### Спецификации OCI

#### 1. Runtime Specification (runtime-spec)

Описывает как запускать контейнер:
- Формат конфигурации (`config.json`)
- Жизненный цикл контейнера
- Операции: create, start, kill, delete

```json
// Пример config.json (упрощённый)
{
  "ociVersion": "1.0.2",
  "process": {
    "terminal": true,
    "user": { "uid": 0, "gid": 0 },
    "args": ["sh"],
    "cwd": "/"
  },
  "root": {
    "path": "rootfs",
    "readonly": true
  },
  "linux": {
    "namespaces": [
      { "type": "pid" },
      { "type": "network" },
      { "type": "mount" }
    ]
  }
}
```

#### 2. Image Specification (image-spec)

Описывает формат образов:
- Manifest — метаданные образа
- Layers — слои файловой системы
- Configuration — настройки запуска

```json
// Пример Image Manifest
{
  "schemaVersion": 2,
  "mediaType": "application/vnd.oci.image.manifest.v1+json",
  "config": {
    "mediaType": "application/vnd.oci.image.config.v1+json",
    "digest": "sha256:abc123...",
    "size": 1470
  },
  "layers": [
    {
      "mediaType": "application/vnd.oci.image.layer.v1.tar+gzip",
      "digest": "sha256:def456...",
      "size": 28567123
    }
  ]
}
```

#### 3. Distribution Specification (distribution-spec)

Описывает API для registry:
- Push/Pull операции
- Аутентификация
- Манифесты и блобы

```bash
# Пример API вызовов к registry
# Получение манифеста
GET /v2/<name>/manifests/<reference>

# Загрузка слоя
GET /v2/<name>/blobs/<digest>

# Проверка существования образа
HEAD /v2/<name>/manifests/<reference>
```

### OCI-совместимые инструменты

| Категория | Инструменты |
|-----------|-------------|
| **Runtime** | runc, crun, youki, gVisor, Kata Containers |
| **High-level runtime** | containerd, CRI-O, Podman |
| **Registry** | Docker Registry, Harbor, Quay, GHCR |
| **Build tools** | Docker BuildKit, Buildah, Kaniko |

---

## 7. Как всё работает вместе

### Запуск контейнера: полный путь

```
1. docker run nginx
       │
       ▼
2. Docker CLI → REST API → Docker Daemon
       │
       ▼
3. Daemon проверяет наличие образа
   (если нет — загружает из registry)
       │
       ▼
4. Daemon → gRPC → containerd
   "Создай контейнер с этим образом"
       │
       ▼
5. containerd подготавливает:
   - Распаковывает слои (overlay2)
   - Создаёт snapshot для R/W слоя
   - Генерирует OCI config.json
       │
       ▼
6. containerd запускает containerd-shim
       │
       ▼
7. containerd-shim запускает runc
       │
       ▼
8. runc:
   - Создаёт namespaces (PID, Net, Mount...)
   - Настраивает cgroups (CPU, Memory limits)
   - Применяет seccomp профиль
   - Выполняет pivot_root
   - exec() процесса контейнера
       │
       ▼
9. runc завершается, shim остаётся родителем процесса
       │
       ▼
10. Контейнер работает!
    - shim отслеживает состояние
    - containerd управляет lifecycle
    - daemon предоставляет API
```

### Визуализация компонентов

```
┌─────────────────────────────────────────────────────────────────┐
│                          Host OS (Linux)                         │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                     Kernel                                  │ │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐       │ │
│  │  │Namespaces│ │ cgroups  │ │ seccomp  │ │capabilities│     │ │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘       │ │
│  └────────────────────────────────────────────────────────────┘ │
│                                                                  │
│  ┌─────────────────┐  ┌─────────────────┐  ┌─────────────────┐  │
│  │   Container 1   │  │   Container 2   │  │   Container 3   │  │
│  │  ┌───────────┐  │  │  ┌───────────┐  │  │  ┌───────────┐  │  │
│  │  │  Process  │  │  │  │  Process  │  │  │  │  Process  │  │  │
│  │  └───────────┘  │  │  └───────────┘  │  │  └───────────┘  │  │
│  │  Isolated:      │  │  Isolated:      │  │  Isolated:      │  │
│  │  - PID ns       │  │  - PID ns       │  │  - PID ns       │  │
│  │  - Net ns       │  │  - Net ns       │  │  - Net ns       │  │
│  │  - Mount ns     │  │  - Mount ns     │  │  - Mount ns     │  │
│  │  - cgroup limits│  │  - cgroup limits│  │  - cgroup limits│  │
│  └─────────────────┘  └─────────────────┘  └─────────────────┘  │
│                                                                  │
│  ┌────────────────────────────────────────────────────────────┐ │
│  │                      overlay2 (UnionFS)                     │ │
│  │  Base layers (shared) + Container R/W layers               │ │
│  └────────────────────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────┘
```

---

## Ключевые выводы

1. **Контейнеры — не виртуальные машины**: они используют ядро хоста, изолируя только userspace

2. **Namespaces дают изоляцию**: каждый контейнер видит своё "пространство" процессов, сети, файлов

3. **Cgroups ограничивают ресурсы**: предотвращают "шумных соседей" и OOM на хосте

4. **Union FS экономит место**: слои переиспользуются между образами и контейнерами

5. **OCI стандарты обеспечивают совместимость**: можно использовать разные runtime и registry

6. **Многоуровневая архитектура**: Docker → containerd → runc обеспечивает гибкость и стабильность

---

## Дополнительные ресурсы

- [Linux Namespaces (man 7 namespaces)](https://man7.org/linux/man-pages/man7/namespaces.7.html)
- [Control Groups v2 documentation](https://docs.kernel.org/admin-guide/cgroup-v2.html)
- [OCI Runtime Specification](https://github.com/opencontainers/runtime-spec)
- [OCI Image Specification](https://github.com/opencontainers/image-spec)
- [containerd documentation](https://containerd.io/docs/)
- [runc documentation](https://github.com/opencontainers/runc)
