# Установка и настройка Docker

## Системные требования

### Linux
- 64-битная версия ОС
- Ядро Linux версии 3.10 или выше (рекомендуется 4.x+)
- Поддержка cgroups и namespaces
- Минимум 2 ГБ ОЗУ (рекомендуется 4 ГБ+)
- Поддержка iptables (версии 1.4+)

### macOS
- macOS 12 (Monterey) или новее
- Apple Silicon (M1/M2/M3) или Intel процессор
- Минимум 4 ГБ ОЗУ
- Включенная виртуализация (VT-x для Intel)

### Windows
- Windows 10/11 64-bit: Pro, Enterprise, или Education (Build 19041+)
- Windows Home с WSL2
- Включенные Hyper-V и WSL2
- Минимум 4 ГБ ОЗУ
- BIOS-level hardware virtualization

---

## Установка на Linux

### Ubuntu / Debian

```bash
# 1. Удаление старых версий (если есть)
sudo apt-get remove docker docker-engine docker.io containerd runc

# 2. Обновление пакетов и установка зависимостей
sudo apt-get update
sudo apt-get install -y \
    ca-certificates \
    curl \
    gnupg \
    lsb-release

# 3. Добавление официального GPG ключа Docker
sudo install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/ubuntu/gpg | sudo gpg --dearmor -o /etc/apt/keyrings/docker.gpg
sudo chmod a+r /etc/apt/keyrings/docker.gpg

# 4. Настройка репозитория
echo \
  "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.gpg] https://download.docker.com/linux/ubuntu \
  $(. /etc/os-release && echo "$VERSION_CODENAME") stable" | \
  sudo tee /etc/apt/sources.list.d/docker.list > /dev/null

# 5. Установка Docker Engine
sudo apt-get update
sudo apt-get install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 6. Проверка установки
sudo docker run hello-world
```

### CentOS / RHEL / Fedora

```bash
# 1. Удаление старых версий
sudo yum remove docker \
    docker-client \
    docker-client-latest \
    docker-common \
    docker-latest \
    docker-latest-logrotate \
    docker-logrotate \
    docker-engine

# 2. Установка зависимостей
sudo yum install -y yum-utils

# 3. Добавление репозитория Docker
sudo yum-config-manager --add-repo https://download.docker.com/linux/centos/docker-ce.repo

# 4. Установка Docker Engine
sudo yum install -y docker-ce docker-ce-cli containerd.io docker-buildx-plugin docker-compose-plugin

# 5. Запуск Docker
sudo systemctl start docker
sudo systemctl enable docker

# 6. Проверка установки
sudo docker run hello-world
```

### Альтернативный метод: convenience script

```bash
# Быстрая установка (не рекомендуется для production)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

> **Важно:** Convenience script подходит только для разработки и тестирования. Для production используйте установку через пакетный менеджер.

---

## Установка на macOS

### Docker Desktop для macOS

1. **Скачайте Docker Desktop:**
   - Перейдите на [docker.com/products/docker-desktop](https://www.docker.com/products/docker-desktop)
   - Выберите версию для вашего процессора (Apple Silicon или Intel)

2. **Установка:**
   ```bash
   # Или через Homebrew
   brew install --cask docker
   ```

3. **Первый запуск:**
   - Откройте Docker.app из папки Applications
   - Примите лицензионное соглашение
   - Дождитесь инициализации Docker Engine

4. **Проверка в терминале:**
   ```bash
   docker version
   docker run hello-world
   ```

### Настройка ресурсов (Docker Desktop)

Откройте Docker Desktop → Settings → Resources:
- **CPUs:** количество ядер для Docker
- **Memory:** объем ОЗУ (рекомендуется 4-8 ГБ)
- **Swap:** размер swap-файла
- **Disk image size:** размер виртуального диска

---

## Установка на Windows

### Предварительные требования

1. **Включение WSL2:**
   ```powershell
   # В PowerShell от имени администратора
   wsl --install

   # Или включите компоненты вручную:
   dism.exe /online /enable-feature /featurename:Microsoft-Windows-Subsystem-Linux /all /norestart
   dism.exe /online /enable-feature /featurename:VirtualMachinePlatform /all /norestart
   ```

2. **Установка ядра WSL2:**
   - Скачайте и установите [WSL2 Linux kernel update package](https://aka.ms/wsl2kernel)

3. **Установка WSL2 как версии по умолчанию:**
   ```powershell
   wsl --set-default-version 2
   ```

### Docker Desktop для Windows

1. **Скачайте установщик:**
   - [Docker Desktop for Windows](https://www.docker.com/products/docker-desktop)

2. **Запустите установку:**
   - Выберите "Use WSL 2 instead of Hyper-V" (рекомендуется)
   - Дождитесь завершения установки
   - Перезагрузите компьютер

3. **Первый запуск:**
   - Запустите Docker Desktop
   - Дождитесь инициализации

4. **Проверка в PowerShell/CMD:**
   ```powershell
   docker version
   docker run hello-world
   ```

### Альтернатива: Docker в WSL2 без Docker Desktop

```bash
# В WSL2 терминале (Ubuntu)
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
sudo usermod -aG docker $USER

# Запуск Docker демона
sudo service docker start
```

---

## Docker Desktop vs Docker Engine

| Характеристика | Docker Engine | Docker Desktop |
|---------------|---------------|----------------|
| **Платформы** | Только Linux | Windows, macOS, Linux |
| **Интерфейс** | Только CLI | GUI + CLI |
| **Виртуализация** | Нативная | VM (Hyper-V, HyperKit, QEMU) |
| **Производительность** | Максимальная | Небольшой overhead |
| **Лицензия** | Open Source (Apache 2.0) | Бесплатно для личного использования и малого бизнеса |
| **Компоненты** | Docker Engine, CLI | Engine + Compose + Kubernetes + Extensions |
| **Обновления** | Ручные через пакетный менеджер | Автоматические |

### Когда использовать Docker Engine
- Production серверы на Linux
- CI/CD пайплайны
- Минимальный footprint
- Полный контроль над конфигурацией

### Когда использовать Docker Desktop
- Разработка на Windows/macOS
- Удобный GUI для управления контейнерами
- Встроенный Kubernetes
- Быстрый старт для новичков

---

## Post-installation Steps

### Добавление пользователя в группу docker (Linux)

По умолчанию Docker требует sudo. Чтобы запускать Docker без sudo:

```bash
# Создание группы docker (если не существует)
sudo groupadd docker

# Добавление текущего пользователя в группу
sudo usermod -aG docker $USER

# Активация изменений без перелогина
newgrp docker

# Проверка
docker run hello-world
```

> **Безопасность:** Членство в группе docker эквивалентно root-доступу. Добавляйте только доверенных пользователей.

### Настройка автозапуска Docker

```bash
# Включение автозапуска при загрузке системы
sudo systemctl enable docker.service
sudo systemctl enable containerd.service

# Проверка статуса
sudo systemctl status docker

# Отключение автозапуска (если нужно)
sudo systemctl disable docker.service
sudo systemctl disable containerd.service
```

### Настройка логирования

```bash
# Настройка ротации логов
sudo nano /etc/docker/daemon.json
```

```json
{
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  }
}
```

---

## Проверка установки

### Базовые команды проверки

```bash
# Версия Docker
docker version

# Пример вывода:
# Client: Docker Engine - Community
#  Version:           24.0.7
#  API version:       1.43
#  ...
# Server: Docker Engine - Community
#  Engine:
#   Version:          24.0.7
#   API version:      1.43 (minimum version 1.12)
#   ...

# Информация о системе
docker info

# Тестовый контейнер
docker run hello-world
```

### Расширенная проверка

```bash
# Проверка работы с образами
docker pull nginx:alpine
docker images

# Запуск контейнера
docker run -d -p 8080:80 --name test-nginx nginx:alpine

# Проверка статуса
docker ps

# Проверка в браузере: http://localhost:8080

# Очистка
docker stop test-nginx
docker rm test-nginx
docker rmi nginx:alpine
```

### Проверка Docker Compose

```bash
# Версия Compose
docker compose version

# Или для standalone версии
docker-compose --version
```

---

## Базовая конфигурация daemon.json

Файл конфигурации Docker daemon располагается:
- **Linux:** `/etc/docker/daemon.json`
- **Windows:** `C:\ProgramData\docker\config\daemon.json`
- **macOS (Docker Desktop):** через GUI или `~/.docker/daemon.json`

### Пример полной конфигурации

```json
{
  "data-root": "/var/lib/docker",
  "storage-driver": "overlay2",
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "10m",
    "max-file": "3"
  },
  "default-address-pools": [
    {
      "base": "172.17.0.0/16",
      "size": 24
    }
  ],
  "dns": ["8.8.8.8", "8.8.4.4"],
  "registry-mirrors": [],
  "insecure-registries": [],
  "live-restore": true,
  "debug": false,
  "tls": false,
  "experimental": false,
  "features": {
    "buildkit": true
  },
  "default-ulimits": {
    "nofile": {
      "Name": "nofile",
      "Hard": 65536,
      "Soft": 65536
    }
  }
}
```

### Основные параметры

| Параметр | Описание |
|----------|----------|
| `data-root` | Путь к данным Docker (образы, контейнеры, volumes) |
| `storage-driver` | Драйвер хранилища (overlay2 рекомендуется) |
| `log-driver` | Драйвер логирования |
| `log-opts` | Настройки логирования |
| `dns` | DNS серверы для контейнеров |
| `registry-mirrors` | Зеркала Docker Hub |
| `insecure-registries` | Реестры без HTTPS |
| `live-restore` | Сохранение контейнеров при перезапуске daemon |
| `debug` | Режим отладки |
| `features.buildkit` | Использование BuildKit для сборки |

### Применение изменений

```bash
# Проверка синтаксиса конфигурации
docker info

# Перезапуск Docker daemon
sudo systemctl restart docker

# Проверка статуса
sudo systemctl status docker
```

---

## Типичные проблемы при установке и их решение

### 1. Permission denied при запуске docker

**Ошибка:**
```
Got permission denied while trying to connect to the Docker daemon socket
```

**Решение:**
```bash
# Добавить пользователя в группу docker
sudo usermod -aG docker $USER

# Применить изменения
newgrp docker

# Или перелогиниться
```

### 2. Cannot connect to Docker daemon

**Ошибка:**
```
Cannot connect to the Docker daemon at unix:///var/run/docker.sock
```

**Решение:**
```bash
# Проверить статус Docker
sudo systemctl status docker

# Запустить Docker если остановлен
sudo systemctl start docker

# Проверить сокет
ls -la /var/run/docker.sock
```

### 3. Docker Desktop не запускается на Windows

**Причины и решения:**
- **Hyper-V не включен:**
  ```powershell
  Enable-WindowsOptionalFeature -Online -FeatureName Microsoft-Hyper-V -All
  ```
- **WSL2 не установлен:** установите WSL2 kernel update
- **Virtualization отключена в BIOS:** включите VT-x/AMD-V в BIOS

### 4. Slow performance на macOS

**Решения:**
- Используйте VirtioFS вместо gRPC FUSE (Docker Desktop → Settings → General)
- Исключите ненужные директории из file sharing
- Увеличьте выделенную память

### 5. No space left on device

**Решение:**
```bash
# Проверка использования диска Docker
docker system df

# Очистка неиспользуемых данных
docker system prune -a

# Очистка volumes (осторожно!)
docker volume prune

# Проверка размера data-root
sudo du -sh /var/lib/docker
```

### 6. DNS resolution не работает в контейнерах

**Решение:**
```bash
# Добавить DNS в daemon.json
sudo nano /etc/docker/daemon.json
```

```json
{
  "dns": ["8.8.8.8", "8.8.4.4"]
}
```

```bash
sudo systemctl restart docker
```

### 7. Ошибка при pull образов (TLS/Certificate)

**Ошибка:**
```
x509: certificate signed by unknown authority
```

**Решение для private registry:**
```json
{
  "insecure-registries": ["myregistry.local:5000"]
}
```

Или добавьте сертификат:
```bash
sudo mkdir -p /etc/docker/certs.d/myregistry.local:5000
sudo cp ca.crt /etc/docker/certs.d/myregistry.local:5000/
```

### 8. Container занимает слишком много памяти

**Решение:**
```bash
# Ограничение памяти при запуске
docker run -m 512m --memory-swap 1g myimage

# Проверка использования ресурсов
docker stats
```

### 9. Порт уже используется

**Ошибка:**
```
Bind for 0.0.0.0:80 failed: port is already allocated
```

**Решение:**
```bash
# Найти процесс на порту
sudo lsof -i :80
# или
sudo netstat -tulpn | grep :80

# Остановить конфликтующий контейнер
docker ps
docker stop <container_id>

# Использовать другой порт
docker run -p 8080:80 myimage
```

### 10. Проблемы с overlay2 на старых системах

**Решение:**
```bash
# Проверить поддержку
grep overlay /proc/filesystems

# Если overlay2 не поддерживается, использовать vfs (медленнее)
{
  "storage-driver": "vfs"
}
```

---

## Полезные команды для диагностики

```bash
# Полная информация о Docker
docker info

# Проверка логов Docker daemon
sudo journalctl -u docker.service

# Проверка событий Docker
docker events

# Диагностика сети
docker network ls
docker network inspect bridge

# Проверка использования ресурсов
docker system df
docker stats
```

---

## Дополнительные ресурсы

- [Официальная документация Docker](https://docs.docker.com/engine/install/)
- [Docker Desktop Release Notes](https://docs.docker.com/desktop/release-notes/)
- [Docker Engine Release Notes](https://docs.docker.com/engine/release-notes/)
- [Post-installation steps for Linux](https://docs.docker.com/engine/install/linux-postinstall/)
