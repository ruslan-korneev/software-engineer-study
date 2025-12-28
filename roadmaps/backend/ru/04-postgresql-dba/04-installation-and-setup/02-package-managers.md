# Установка PostgreSQL через Package Managers

[prev: 01-using-docker](./01-using-docker.md) | [next: 03-connect-using-psql](./03-connect-using-psql.md)
---

## Введение

Package managers (менеджеры пакетов) — это основной способ установки PostgreSQL на серверы и рабочие станции. Каждая операционная система имеет свой менеджер пакетов с репозиториями PostgreSQL.

## Установка на Ubuntu/Debian

### Из стандартных репозиториев

```bash
# Обновление списка пакетов
sudo apt update

# Установка PostgreSQL
sudo apt install postgresql postgresql-contrib

# Проверка статуса
sudo systemctl status postgresql

# Запуск службы
sudo systemctl start postgresql

# Автозапуск при загрузке системы
sudo systemctl enable postgresql
```

### Из официального репозитория PostgreSQL

Официальный репозиторий содержит более свежие версии PostgreSQL.

```bash
# Установка необходимых пакетов
sudo apt install wget ca-certificates

# Добавление ключа репозитория
sudo install -d /usr/share/postgresql-common/pgdg
sudo curl -o /usr/share/postgresql-common/pgdg/apt.postgresql.org.asc --fail https://www.postgresql.org/media/keys/ACCC4CF8.asc

# Добавление репозитория
sudo sh -c 'echo "deb [signed-by=/usr/share/postgresql-common/pgdg/apt.postgresql.org.asc] https://apt.postgresql.org/pub/repos/apt $(lsb_release -cs)-pgdg main" > /etc/apt/sources.list.d/pgdg.list'

# Обновление и установка
sudo apt update
sudo apt install postgresql-16 postgresql-contrib-16

# Установка конкретной версии
sudo apt install postgresql-15
```

### Установка дополнительных компонентов

```bash
# Клиентские утилиты
sudo apt install postgresql-client-16

# Расширения для разработки
sudo apt install postgresql-server-dev-16

# Популярные расширения
sudo apt install postgresql-16-postgis-3
sudo apt install postgresql-16-pg-stat-statements
```

## Установка на RHEL/CentOS/Rocky Linux

### Из стандартных репозиториев

```bash
# RHEL 8/9 и совместимые дистрибутивы
sudo dnf install postgresql-server postgresql-contrib

# Инициализация базы данных
sudo postgresql-setup --initdb

# Запуск и автозапуск
sudo systemctl start postgresql
sudo systemctl enable postgresql
```

### Из официального репозитория PostgreSQL

```bash
# Установка репозитория (RHEL 9 / Rocky 9)
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Отключение встроенного модуля PostgreSQL
sudo dnf -qy module disable postgresql

# Установка PostgreSQL 16
sudo dnf install -y postgresql16-server postgresql16-contrib

# Инициализация базы данных
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# Запуск и автозапуск
sudo systemctl start postgresql-16
sudo systemctl enable postgresql-16
```

### Расположение файлов на RHEL

```
/var/lib/pgsql/16/data/          # Каталог данных
/var/lib/pgsql/16/data/postgresql.conf  # Основная конфигурация
/var/lib/pgsql/16/data/pg_hba.conf      # Аутентификация
/usr/pgsql-16/bin/               # Бинарные файлы
/var/lib/pgsql/16/backups/       # Каталог для backup
```

## Установка на macOS

### Homebrew (рекомендуется)

```bash
# Установка Homebrew (если не установлен)
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Установка PostgreSQL
brew install postgresql@16

# Добавление в PATH
echo 'export PATH="/opt/homebrew/opt/postgresql@16/bin:$PATH"' >> ~/.zshrc
source ~/.zshrc

# Запуск как сервис
brew services start postgresql@16

# Остановка сервиса
brew services stop postgresql@16

# Проверка статуса
brew services list
```

### Расположение файлов на macOS (Homebrew)

```
/opt/homebrew/var/postgresql@16/    # Каталог данных (Apple Silicon)
/usr/local/var/postgresql@16/       # Каталог данных (Intel)
/opt/homebrew/opt/postgresql@16/bin # Бинарные файлы
```

### MacPorts

```bash
# Установка PostgreSQL через MacPorts
sudo port install postgresql16-server

# Инициализация
sudo mkdir -p /opt/local/var/db/postgresql16/defaultdb
sudo chown postgres:postgres /opt/local/var/db/postgresql16/defaultdb
sudo su postgres -c '/opt/local/lib/postgresql16/bin/initdb -D /opt/local/var/db/postgresql16/defaultdb'

# Запуск
sudo port load postgresql16-server
```

## Установка на Windows

### Официальный установщик

1. Скачайте установщик с [postgresql.org/download/windows](https://www.postgresql.org/download/windows/)
2. Запустите установщик от имени администратора
3. Выберите компоненты:
   - PostgreSQL Server
   - pgAdmin 4
   - Stack Builder
   - Command Line Tools
4. Укажите каталог установки и данных
5. Установите пароль для пользователя postgres
6. Выберите порт (по умолчанию 5432)
7. Выберите локаль (рекомендуется: Default locale)

### Chocolatey

```powershell
# Установка Chocolatey (PowerShell от имени администратора)
Set-ExecutionPolicy Bypass -Scope Process -Force
[System.Net.ServicePointManager]::SecurityProtocol = [System.Net.ServicePointManager]::SecurityProtocol -bor 3072
iex ((New-Object System.Net.WebClient).DownloadString('https://chocolatey.org/install.ps1'))

# Установка PostgreSQL
choco install postgresql16

# Добавление в PATH
$env:Path += ";C:\Program Files\PostgreSQL\16\bin"
```

### Scoop

```powershell
# Установка Scoop
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
irm get.scoop.sh | iex

# Добавление bucket
scoop bucket add main

# Установка PostgreSQL
scoop install postgresql
```

### Расположение файлов на Windows

```
C:\Program Files\PostgreSQL\16\        # Каталог установки
C:\Program Files\PostgreSQL\16\data\   # Каталог данных
C:\Program Files\PostgreSQL\16\bin\    # Бинарные файлы
```

## Управление службой PostgreSQL

### Linux (systemd)

```bash
# Запуск
sudo systemctl start postgresql

# Остановка
sudo systemctl stop postgresql

# Перезапуск
sudo systemctl restart postgresql

# Перезагрузка конфигурации (без рестарта)
sudo systemctl reload postgresql

# Проверка статуса
sudo systemctl status postgresql

# Включение автозапуска
sudo systemctl enable postgresql

# Отключение автозапуска
sudo systemctl disable postgresql

# Просмотр логов
sudo journalctl -u postgresql -f
```

### macOS (Homebrew)

```bash
# Запуск
brew services start postgresql@16

# Остановка
brew services stop postgresql@16

# Перезапуск
brew services restart postgresql@16

# Список сервисов
brew services list
```

### Windows (Services)

```powershell
# PowerShell от имени администратора
# Запуск
Start-Service postgresql-x64-16

# Остановка
Stop-Service postgresql-x64-16

# Перезапуск
Restart-Service postgresql-x64-16

# Проверка статуса
Get-Service postgresql-x64-16
```

## Инициализация кластера базы данных

```bash
# Linux/macOS
# Создание нового кластера
initdb -D /path/to/data --encoding=UTF8 --locale=ru_RU.UTF-8

# С указанием пользователя
initdb -D /path/to/data -U postgres --auth-local=peer --auth-host=scram-sha-256

# Windows (PowerShell)
& "C:\Program Files\PostgreSQL\16\bin\initdb.exe" -D "C:\PostgreSQL\data" -U postgres -E UTF8
```

### Параметры initdb

| Параметр | Описание |
|----------|----------|
| `-D, --pgdata` | Каталог для данных |
| `-E, --encoding` | Кодировка базы данных |
| `--locale` | Локаль |
| `-U, --username` | Имя суперпользователя |
| `-W, --pwprompt` | Запрос пароля |
| `--auth-local` | Метод аутентификации для local |
| `--auth-host` | Метод аутентификации для host |

## Пост-установочная настройка

### Базовая конфигурация postgresql.conf

```bash
# Найти файл конфигурации
sudo -u postgres psql -c "SHOW config_file;"

# Редактирование
sudo nano /etc/postgresql/16/main/postgresql.conf
```

Основные параметры для изменения:

```conf
# Сетевые настройки
listen_addresses = 'localhost'  # или '*' для внешних подключений
port = 5432
max_connections = 100

# Память
shared_buffers = 256MB          # 25% от RAM для выделенного сервера
effective_cache_size = 768MB    # 75% от RAM
work_mem = 4MB
maintenance_work_mem = 64MB

# Логирование
log_destination = 'stderr'
logging_collector = on
log_directory = 'pg_log'
log_filename = 'postgresql-%Y-%m-%d_%H%M%S.log'
```

### Настройка pg_hba.conf

```bash
# Расположение файла
sudo -u postgres psql -c "SHOW hba_file;"

# Редактирование
sudo nano /etc/postgresql/16/main/pg_hba.conf
```

Пример конфигурации:

```conf
# TYPE  DATABASE        USER            ADDRESS                 METHOD
local   all             postgres                                peer
local   all             all                                     peer
host    all             all             127.0.0.1/32            scram-sha-256
host    all             all             ::1/128                 scram-sha-256
host    all             all             192.168.1.0/24          scram-sha-256
```

### Применение изменений

```bash
# Перезагрузка конфигурации (не требует рестарта)
sudo systemctl reload postgresql

# Или через psql
sudo -u postgres psql -c "SELECT pg_reload_conf();"

# Полный перезапуск (для некоторых параметров)
sudo systemctl restart postgresql
```

## Управление версиями

### Установка нескольких версий

```bash
# Ubuntu/Debian
sudo apt install postgresql-14 postgresql-15 postgresql-16

# Просмотр установленных версий
dpkg -l | grep postgresql

# Переключение между кластерами
pg_lsclusters
pg_ctlcluster 16 main start
pg_ctlcluster 15 main stop
```

### Обновление версии

```bash
# Ubuntu/Debian - создание нового кластера и миграция
sudo pg_upgradecluster 15 main

# Ручное обновление с pg_upgrade
pg_upgrade \
  --old-datadir=/var/lib/pgsql/15/data \
  --new-datadir=/var/lib/pgsql/16/data \
  --old-bindir=/usr/pgsql-15/bin \
  --new-bindir=/usr/pgsql-16/bin
```

## Проверка установки

```bash
# Проверка версии
psql --version
postgres --version

# Подключение к базе данных
sudo -u postgres psql

# Проверка работы сервера
sudo -u postgres psql -c "SELECT version();"

# Список баз данных
sudo -u postgres psql -l
```

## Best Practices

1. **Используйте официальные репозитории** для получения актуальных версий
2. **Планируйте расположение данных** — отдельный диск для PGDATA
3. **Настройте firewall** — ограничьте доступ к порту 5432
4. **Создайте отдельного пользователя** для приложений (не используйте postgres)
5. **Настройте резервное копирование** сразу после установки
6. **Мониторьте использование ресурсов** — память, диск, CPU
7. **Регулярно обновляйте** для получения патчей безопасности
8. **Документируйте изменения** конфигурации

## Полезные ссылки

- [Официальная документация по установке](https://www.postgresql.org/download/)
- [PostgreSQL APT Repository](https://wiki.postgresql.org/wiki/Apt)
- [PostgreSQL YUM Repository](https://www.postgresql.org/download/linux/redhat/)

---
[prev: 01-using-docker](./01-using-docker.md) | [next: 03-connect-using-psql](./03-connect-using-psql.md)