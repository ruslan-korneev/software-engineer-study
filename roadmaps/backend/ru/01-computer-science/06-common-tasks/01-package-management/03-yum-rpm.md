# yum, dnf и rpm - управление пакетами в Red Hat/Fedora/CentOS

## Обзор

В системах семейства Red Hat используется двухуровневая система:

- **rpm** - низкоуровневый инструмент для работы с .rpm файлами
- **yum** - высокоуровневый менеджер (RHEL/CentOS 7)
- **dnf** - современный менеджер (Fedora, RHEL 8+, CentOS Stream)

## rpm - низкоуровневый менеджер

### Установка пакетов

```bash
# Установить .rpm файл
sudo rpm -i package.rpm
sudo rpm --install package.rpm

# Установить с выводом прогресса
sudo rpm -ivh package.rpm
# -i = install
# -v = verbose (подробный вывод)
# -h = hash (показывать прогресс)

# Обновить пакет (или установить если нет)
sudo rpm -Uvh package.rpm
# -U = upgrade

# Обновить только если пакет установлен
sudo rpm -Fvh package.rpm
# -F = freshen
```

### Удаление пакетов

```bash
# Удалить пакет
sudo rpm -e package-name
sudo rpm --erase package-name

# Удалить без проверки зависимостей (опасно!)
sudo rpm -e --nodeps package-name
```

### Запросы к базе RPM

```bash
# Список всех установленных пакетов
rpm -qa
rpm --query --all

# Проверить установлен ли пакет
rpm -q nginx
rpm --query nginx

# Информация о пакете
rpm -qi nginx
rpm --query --info nginx

# Список файлов пакета
rpm -ql nginx
rpm --query --list nginx

# Найти какому пакету принадлежит файл
rpm -qf /usr/sbin/nginx
rpm --query --file /usr/sbin/nginx

# Показать конфигурационные файлы пакета
rpm -qc nginx
rpm --query --configfiles nginx

# Показать документацию пакета
rpm -qd nginx
rpm --query --docfiles nginx

# Показать зависимости
rpm -qR nginx
rpm --query --requires nginx

# Что предоставляет пакет
rpm -q --provides nginx
```

### Запросы к .rpm файлу (не установленному)

```bash
# Добавляем -p для работы с файлом
rpm -qip package.rpm     # информация
rpm -qlp package.rpm     # список файлов
rpm -qRp package.rpm     # зависимости
```

### Проверка целостности

```bash
# Проверить пакет на изменения
rpm -V nginx
rpm --verify nginx

# Проверить все пакеты
rpm -Va

# Значения вывода:
# S = размер изменился
# M = права/тип изменились
# 5 = MD5 сумма изменилась
# D = устройство изменилось
# L = символическая ссылка изменилась
# U = владелец изменился
# G = группа изменилась
# T = время модификации изменилось
```

### Импорт GPG ключей

```bash
# Импортировать ключ
sudo rpm --import https://example.com/RPM-GPG-KEY

# Список импортированных ключей
rpm -qa gpg-pubkey*
```

## yum - менеджер для RHEL/CentOS 7

### Основные операции

```bash
# Обновить кеш метаданных
sudo yum makecache

# Проверить обновления
yum check-update

# Установить пакет
sudo yum install nginx

# Установить группу пакетов
sudo yum groupinstall "Development Tools"

# Обновить пакет
sudo yum update nginx

# Обновить всю систему
sudo yum update

# Удалить пакет
sudo yum remove nginx
sudo yum erase nginx

# Переустановить
sudo yum reinstall nginx
```

### Поиск и информация

```bash
# Поиск пакетов
yum search nginx

# Информация о пакете
yum info nginx

# Список всех пакетов
yum list all

# Установленные пакеты
yum list installed

# Доступные для установки
yum list available

# Обновляемые пакеты
yum list updates

# Найти какой пакет содержит файл
yum provides /usr/sbin/nginx
yum whatprovides /usr/sbin/nginx

# Показать зависимости
yum deplist nginx
```

### Управление репозиториями

```bash
# Список репозиториев
yum repolist
yum repolist all    # включая отключенные

# Информация о репозитории
yum repoinfo base

# Временно отключить репозиторий
sudo yum --disablerepo=epel install nginx

# Временно включить репозиторий
sudo yum --enablerepo=epel-testing install nginx

# Установить пакет из конкретного репо
sudo yum install --enablerepo=epel nginx
```

### История и откат

```bash
# История операций
yum history
yum history list

# Информация о транзакции
yum history info 5

# Откатить транзакцию
sudo yum history undo 5

# Повторить транзакцию
sudo yum history redo 5
```

### Очистка кеша

```bash
# Очистить кеш
sudo yum clean all

# Очистить отдельные компоненты
sudo yum clean packages    # .rpm файлы
sudo yum clean metadata    # метаданные
sudo yum clean expire-cache
```

## dnf - современный менеджер (Fedora, RHEL 8+)

DNF - это переписанный yum с улучшенной производительностью и разрешением зависимостей.

### Основные команды (совместимы с yum)

```bash
# Установка
sudo dnf install nginx

# Удаление
sudo dnf remove nginx

# Обновление
sudo dnf upgrade
sudo dnf upgrade nginx

# Поиск
dnf search nginx

# Информация
dnf info nginx
```

### Новые возможности dnf

```bash
# Автоматическое удаление ненужных зависимостей
sudo dnf autoremove

# Обновить систему с автоудалением
sudo dnf upgrade --refresh

# Скачать пакет без установки
dnf download nginx
dnf download --resolve nginx    # с зависимостями

# Переустановить пакет
sudo dnf reinstall nginx

# Понизить версию
sudo dnf downgrade nginx
```

### Модули (RHEL 8+, Fedora)

Модульность позволяет устанавливать разные версии ПО:

```bash
# Список модулей
dnf module list

# Список версий модуля
dnf module list nodejs

# Информация о модуле
dnf module info nodejs

# Включить модуль (версию)
sudo dnf module enable nodejs:18

# Установить модуль
sudo dnf module install nodejs:18

# Переключить версию
sudo dnf module reset nodejs
sudo dnf module enable nodejs:20
sudo dnf module install nodejs:20
```

### Группы пакетов

```bash
# Список групп
dnf group list

# Информация о группе
dnf group info "Development Tools"

# Установить группу
sudo dnf group install "Development Tools"

# Удалить группу
sudo dnf group remove "Development Tools"
```

### История в dnf

```bash
# История операций
dnf history

# Детали транзакции
dnf history info 5

# Откат
sudo dnf history undo 5

# Откат к состоянию до транзакции
sudo dnf history rollback 5
```

## Конфигурация репозиториев

### Файлы конфигурации

Репозитории настраиваются в `/etc/yum.repos.d/*.repo`

```ini
# Пример /etc/yum.repos.d/nginx.repo
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true

[nginx-mainline]
name=nginx mainline repo
baseurl=http://nginx.org/packages/mainline/centos/$releasever/$basearch/
gpgcheck=1
enabled=0
gpgkey=https://nginx.org/keys/nginx_signing.key
module_hotfixes=true
```

### Переменные в конфигурации

- `$releasever` - версия релиза (7, 8, 9)
- `$basearch` - архитектура (x86_64, aarch64)

### EPEL - Extra Packages for Enterprise Linux

```bash
# RHEL/CentOS 7
sudo yum install epel-release

# RHEL/CentOS 8/9
sudo dnf install epel-release

# Fedora - EPEL не нужен, пакеты уже есть
```

### Добавление репозитория вручную

```bash
# Способ 1: добавить .repo файл
sudo curl -o /etc/yum.repos.d/example.repo https://example.com/example.repo

# Способ 2: использовать yum-config-manager
sudo yum-config-manager --add-repo https://example.com/example.repo

# Способ 3: dnf config-manager
sudo dnf config-manager --add-repo https://example.com/example.repo
```

## Блокировка пакетов

### yum

```bash
# Установить плагин
sudo yum install yum-plugin-versionlock

# Заблокировать версию
sudo yum versionlock nginx

# Список заблокированных
yum versionlock list

# Разблокировать
sudo yum versionlock delete nginx

# Очистить все блокировки
sudo yum versionlock clear
```

### dnf

```bash
# Установить плагин
sudo dnf install python3-dnf-plugin-versionlock

# Заблокировать
sudo dnf versionlock add nginx

# Список
dnf versionlock list

# Разблокировать
sudo dnf versionlock delete nginx
```

## Практические примеры

### Установка LAMP стека (RHEL/CentOS)

```bash
# CentOS 7
sudo yum install httpd mariadb-server php php-mysqlnd

# CentOS 8+ / Fedora
sudo dnf install httpd mariadb-server php php-mysqlnd
```

### Поиск пакета по файлу

```bash
# Какой пакет содержит команду htpasswd?
yum provides */htpasswd
# или
dnf provides */htpasswd

# Результат: httpd-tools
```

### Проверка безопасности обновлений

```bash
# Только обновления безопасности
sudo dnf upgrade --security

# Список обновлений безопасности
dnf updateinfo list security

# Информация об уязвимостях
dnf updateinfo info security
```

### Работа с локальными .rpm

```bash
# Установить локальный .rpm с разрешением зависимостей
sudo yum localinstall package.rpm
# или
sudo dnf install ./package.rpm
```

## Сравнение yum и dnf

| Функция | yum | dnf |
|---------|-----|-----|
| Производительность | Медленнее | Быстрее |
| Разрешение зависимостей | Базовое | Улучшенное (libsolv) |
| API Python | Нестабильный | Стабильный |
| Модули | Нет | Да |
| Автоудаление зависимостей | Ограниченное | Полное |
| Обратная совместимость | - | Совместим с yum |

## Резюме команд

| Задача | rpm | yum | dnf |
|--------|-----|-----|-----|
| Установить | `rpm -ivh pkg.rpm` | `yum install pkg` | `dnf install pkg` |
| Удалить | `rpm -e pkg` | `yum remove pkg` | `dnf remove pkg` |
| Обновить | `rpm -Uvh pkg.rpm` | `yum update` | `dnf upgrade` |
| Поиск | - | `yum search` | `dnf search` |
| Информация | `rpm -qi pkg` | `yum info pkg` | `dnf info pkg` |
| Список файлов | `rpm -ql pkg` | `yum provides` | `dnf provides` |
| Владелец файла | `rpm -qf /path` | `yum provides` | `dnf provides` |
| Очистка кеша | - | `yum clean all` | `dnf clean all` |
