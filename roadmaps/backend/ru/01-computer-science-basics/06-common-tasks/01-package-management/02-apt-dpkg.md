# apt и dpkg - управление пакетами в Debian/Ubuntu

## Обзор

В системах Debian и Ubuntu используется двухуровневая система управления пакетами:

- **dpkg** - низкоуровневый инструмент для работы с .deb файлами
- **apt** - высокоуровневый инструмент с поддержкой репозиториев и зависимостей

## dpkg - низкоуровневый менеджер

### Установка пакета из файла

```bash
# Установить .deb файл
sudo dpkg -i package.deb

# Установить все .deb файлы в директории
sudo dpkg -i *.deb
```

**Важно:** dpkg не разрешает зависимости автоматически. Если зависимости не установлены:

```bash
# После dpkg -i исправить зависимости
sudo apt install -f
```

### Удаление пакета

```bash
# Удалить пакет (сохранить конфиги)
sudo dpkg -r package-name

# Полное удаление с конфигами
sudo dpkg -P package-name
# или
sudo dpkg --purge package-name
```

### Информация о пакетах

```bash
# Список всех установленных пакетов
dpkg -l
dpkg --list

# Поиск пакета в списке установленных
dpkg -l | grep nginx

# Статус конкретного пакета
dpkg -s nginx
dpkg --status nginx

# Список файлов пакета
dpkg -L nginx
dpkg --listfiles nginx

# Найти какому пакету принадлежит файл
dpkg -S /usr/bin/nginx
dpkg --search /usr/bin/nginx
```

### Работа с .deb файлами

```bash
# Информация о .deb файле (не установленном)
dpkg -I package.deb
dpkg --info package.deb

# Содержимое .deb файла
dpkg -c package.deb
dpkg --contents package.deb

# Распаковать без установки
dpkg -x package.deb /destination/dir
```

### Проверка целостности

```bash
# Проверить все установленные пакеты
sudo dpkg --audit

# Переконфигурировать пакет
sudo dpkg-reconfigure package-name

# Часто используется для:
sudo dpkg-reconfigure tzdata      # настройка часового пояса
sudo dpkg-reconfigure locales     # настройка локали
```

## apt - современный высокоуровневый менеджер

`apt` - это рекомендуемый инструмент для интерактивной работы (заменяет apt-get/apt-cache).

### Обновление информации о пакетах

```bash
# Обновить индексы репозиториев
sudo apt update

# Вывод показывает:
# - Количество обновляемых пакетов
# - Источники, которые были проверены
```

### Установка пакетов

```bash
# Установить пакет
sudo apt install nginx

# Установить несколько пакетов
sudo apt install nginx postgresql redis

# Установить конкретную версию
sudo apt install nginx=1.18.0-0ubuntu1

# Установить без подтверждения
sudo apt install -y nginx

# Только скачать, не устанавливать
sudo apt install --download-only nginx

# Переустановить пакет
sudo apt reinstall nginx
```

### Обновление системы

```bash
# Обновить все пакеты (безопасно)
sudo apt upgrade

# Полное обновление (может удалять пакеты)
sudo apt full-upgrade

# Обновить конкретный пакет
sudo apt install --only-upgrade nginx
```

### Удаление пакетов

```bash
# Удалить пакет
sudo apt remove nginx

# Удалить с конфигурационными файлами
sudo apt purge nginx
# или
sudo apt remove --purge nginx

# Удалить неиспользуемые зависимости
sudo apt autoremove

# Полная очистка
sudo apt purge nginx && sudo apt autoremove
```

### Поиск и информация

```bash
# Поиск пакетов
apt search nginx

# Показать информацию о пакете
apt show nginx

# Показать доступные версии
apt list -a nginx

# Список установленных пакетов
apt list --installed

# Список обновляемых пакетов
apt list --upgradable

# Показать зависимости
apt depends nginx

# Показать обратные зависимости
apt rdepends nginx
```

### Очистка кеша

```bash
# Удалить скачанные .deb файлы
sudo apt clean

# Удалить старые версии .deb файлов
sudo apt autoclean

# Кеш хранится в /var/cache/apt/archives/
```

## apt-get и apt-cache (классические инструменты)

Эти команды по-прежнему доступны и используются в скриптах:

```bash
# apt-get - эквиваленты apt
sudo apt-get update
sudo apt-get install nginx
sudo apt-get remove nginx
sudo apt-get upgrade
sudo apt-get dist-upgrade    # аналог full-upgrade

# apt-cache - поиск и информация
apt-cache search nginx
apt-cache show nginx
apt-cache policy nginx       # показать версии и приоритеты
apt-cache depends nginx
apt-cache rdepends nginx
```

### Разница между apt и apt-get

| apt | apt-get / apt-cache | Описание |
|-----|---------------------|----------|
| `apt install` | `apt-get install` | Установка |
| `apt remove` | `apt-get remove` | Удаление |
| `apt update` | `apt-get update` | Обновить индексы |
| `apt upgrade` | `apt-get upgrade` | Обновить пакеты |
| `apt full-upgrade` | `apt-get dist-upgrade` | Полное обновление |
| `apt search` | `apt-cache search` | Поиск |
| `apt show` | `apt-cache show` | Информация |
| `apt list` | `dpkg -l` | Список пакетов |

**apt** имеет:
- Прогресс-бар при установке
- Цветной вывод
- Более дружелюбные сообщения

## Управление репозиториями

### Конфигурация источников

Основной файл: `/etc/apt/sources.list`
Дополнительные: `/etc/apt/sources.list.d/*.list`

```bash
# Формат строки:
deb [options] uri distribution [component1] [component2]

# Пример для Ubuntu 22.04:
deb http://archive.ubuntu.com/ubuntu jammy main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-updates main restricted universe multiverse
deb http://archive.ubuntu.com/ubuntu jammy-security main restricted universe multiverse
```

### Компоненты Ubuntu

- **main** - свободное ПО с поддержкой Canonical
- **restricted** - проприетарные драйверы
- **universe** - свободное ПО от сообщества
- **multiverse** - несвободное ПО

### Добавление PPA (Personal Package Archive)

```bash
# Добавить PPA
sudo add-apt-repository ppa:user/ppa-name
sudo apt update

# Удалить PPA
sudo add-apt-repository --remove ppa:user/ppa-name
```

### Добавление стороннего репозитория

```bash
# 1. Добавить GPG ключ
curl -fsSL https://example.com/gpg.key | sudo gpg --dearmor -o /usr/share/keyrings/example.gpg

# 2. Добавить репозиторий
echo "deb [signed-by=/usr/share/keyrings/example.gpg] https://example.com/repo stable main" | \
    sudo tee /etc/apt/sources.list.d/example.list

# 3. Обновить индексы
sudo apt update
```

## Блокировка версий пакетов

```bash
# Заблокировать пакет (не обновлять)
sudo apt-mark hold nginx

# Разблокировать
sudo apt-mark unhold nginx

# Показать заблокированные пакеты
apt-mark showhold

# Пометить как автоматически установленный
sudo apt-mark auto package

# Пометить как вручную установленный
sudo apt-mark manual package
```

## Решение проблем

### Сломанные зависимости

```bash
# Исправить сломанные зависимости
sudo apt install -f
sudo apt --fix-broken install

# Проверить на сломанные пакеты
sudo dpkg --audit
```

### Заблокированный dpkg

```bash
# Если apt завис или был прерван
sudo rm /var/lib/dpkg/lock-frontend
sudo rm /var/lib/dpkg/lock
sudo dpkg --configure -a
```

### Очистка после проблем

```bash
# Полная очистка и переустановка
sudo apt clean
sudo apt update
sudo apt upgrade
```

## Полезные опции apt

```bash
# Симуляция (показать что будет сделано)
apt install --dry-run nginx
apt install -s nginx

# Показать версии для установки
apt install -V nginx

# Не устанавливать рекомендуемые пакеты
apt install --no-install-recommends nginx

# Игнорировать отсутствующие пакеты
apt install nginx postgresql --ignore-missing
```

## Практические примеры

### Установка LAMP стека

```bash
sudo apt update
sudo apt install -y apache2 mysql-server php libapache2-mod-php php-mysql
```

### Поиск и установка библиотеки разработки

```bash
# Найти dev-пакет
apt search libssl | grep dev

# Установить
sudo apt install libssl-dev
```

### Узнать откуда пакет

```bash
# Показать источник пакета
apt policy nginx

# Вывод:
# nginx:
#   Installed: 1.18.0-0ubuntu1
#   Candidate: 1.18.0-0ubuntu1
#   Version table:
#  *** 1.18.0-0ubuntu1 500
#         500 http://archive.ubuntu.com/ubuntu jammy/main amd64 Packages
```

## Резюме команд

| Задача | Команда |
|--------|---------|
| Обновить индексы | `sudo apt update` |
| Установить пакет | `sudo apt install package` |
| Удалить пакет | `sudo apt remove package` |
| Удалить с конфигами | `sudo apt purge package` |
| Обновить все | `sudo apt upgrade` |
| Поиск | `apt search term` |
| Информация | `apt show package` |
| Список файлов | `dpkg -L package` |
| Найти владельца файла | `dpkg -S /path/to/file` |
| Установить .deb | `sudo dpkg -i file.deb` |
| Исправить зависимости | `sudo apt install -f` |
