# Системы управления пакетами

## Что такое пакет?

**Пакет** (package) - это архив, содержащий файлы программы и метаданные, необходимые для её установки. Пакет включает:

- Исполняемые файлы программы
- Конфигурационные файлы
- Библиотеки
- Документацию
- Информацию о зависимостях
- Скрипты установки/удаления

## Зачем нужны пакетные менеджеры?

До появления пакетных менеджеров установка программ в Unix/Linux была сложным процессом:

1. Скачать исходный код
2. Разрешить зависимости вручную
3. Скомпилировать программу
4. Установить в нужные директории
5. Настроить конфигурацию

**Пакетный менеджер** автоматизирует все эти шаги и решает ключевые проблемы:

- **Управление зависимостями** - автоматически устанавливает нужные библиотеки
- **Централизованная установка** - один инструмент для всего ПО
- **Обновление** - легко обновить все пакеты одной командой
- **Удаление** - чистое удаление без "мусора"
- **Безопасность** - пакеты проверяются цифровыми подписями

## Типы пакетных систем

### Низкоуровневые инструменты

Работают непосредственно с файлами пакетов:

| Инструмент | Семейство | Формат пакета |
|------------|-----------|---------------|
| `dpkg` | Debian/Ubuntu | `.deb` |
| `rpm` | Red Hat/Fedora/CentOS | `.rpm` |

### Высокоуровневые инструменты

Работают с репозиториями и автоматически разрешают зависимости:

| Инструмент | Семейство | На базе |
|------------|-----------|---------|
| `apt` | Debian/Ubuntu | dpkg |
| `apt-get` | Debian/Ubuntu | dpkg |
| `yum` | RHEL/CentOS 7 | rpm |
| `dnf` | Fedora/RHEL 8+ | rpm |
| `pacman` | Arch Linux | - |
| `brew` | macOS | - |
| `zypper` | openSUSE | rpm |

## Основные операции пакетных менеджеров

### 1. Поиск пакетов

```bash
# Debian/Ubuntu
apt search nginx
apt-cache search nginx

# Red Hat/Fedora
yum search nginx
dnf search nginx

# Arch Linux
pacman -Ss nginx

# macOS
brew search nginx
```

### 2. Информация о пакете

```bash
# Debian/Ubuntu
apt show nginx
dpkg -s nginx          # для установленного пакета

# Red Hat/Fedora
yum info nginx
dnf info nginx
rpm -qi nginx          # для установленного пакета

# Arch Linux
pacman -Si nginx       # из репозитория
pacman -Qi nginx       # установленный

# macOS
brew info nginx
```

### 3. Установка пакетов

```bash
# Debian/Ubuntu
sudo apt install nginx

# Red Hat/Fedora
sudo yum install nginx
sudo dnf install nginx

# Arch Linux
sudo pacman -S nginx

# macOS
brew install nginx
```

### 4. Обновление пакетов

```bash
# Debian/Ubuntu
sudo apt update        # обновить список пакетов
sudo apt upgrade       # обновить все пакеты

# Red Hat/Fedora
sudo yum update
sudo dnf upgrade

# Arch Linux
sudo pacman -Syu

# macOS
brew update
brew upgrade
```

### 5. Удаление пакетов

```bash
# Debian/Ubuntu
sudo apt remove nginx       # удалить пакет
sudo apt purge nginx        # удалить с конфигами
sudo apt autoremove         # удалить ненужные зависимости

# Red Hat/Fedora
sudo yum remove nginx
sudo dnf remove nginx

# Arch Linux
sudo pacman -R nginx        # удалить
sudo pacman -Rs nginx       # с зависимостями

# macOS
brew uninstall nginx
```

## Репозитории

**Репозиторий** - это сервер (или локальное хранилище), содержащий пакеты и индексы для их поиска.

### Типы репозиториев

- **Официальные** - поддерживаются разработчиками дистрибутива
- **Сторонние** - PPA (Ubuntu), COPR (Fedora), AUR (Arch)
- **Локальные** - для внутреннего использования в организациях

### Конфигурация репозиториев

```bash
# Debian/Ubuntu - файлы в /etc/apt/sources.list.d/
# Пример содержимого:
deb http://archive.ubuntu.com/ubuntu jammy main restricted

# Red Hat/Fedora - файлы в /etc/yum.repos.d/
# Пример .repo файла:
[nginx-stable]
name=nginx stable repo
baseurl=http://nginx.org/packages/centos/$releasever/$basearch/
gpgcheck=1
enabled=1

# Arch Linux - /etc/pacman.conf
[core]
Include = /etc/pacman.d/mirrorlist
```

## Сравнение форматов .deb и .rpm

| Характеристика | .deb | .rpm |
|----------------|------|------|
| Формат архива | ar + tar.gz | cpio + gzip |
| Метаданные | control файл | spec файл |
| Скрипты | preinst, postinst, prerm, postrm | %pre, %post, %preun, %postun |
| Запросы | dpkg-query | rpm -q |
| Проверка | dpkg --verify | rpm -V |

## Практические советы

### 1. Всегда обновляйте индексы перед установкой

```bash
sudo apt update && sudo apt install package
```

### 2. Проверяйте, что установится

```bash
# Debian/Ubuntu - показать что будет установлено
apt install --dry-run nginx

# Показать зависимости
apt depends nginx
```

### 3. Очищайте кеш

```bash
# Debian/Ubuntu
sudo apt clean
sudo apt autoclean

# Red Hat
sudo yum clean all
sudo dnf clean all

# macOS
brew cleanup
```

### 4. Фиксируйте версии критичных пакетов

```bash
# Debian/Ubuntu - заблокировать обновление
sudo apt-mark hold nginx

# Разблокировать
sudo apt-mark unhold nginx
```

## Альтернативные форматы пакетов

Современные универсальные форматы, не зависящие от дистрибутива:

### Flatpak

```bash
# Установка
flatpak install flathub org.gimp.GIMP

# Запуск
flatpak run org.gimp.GIMP
```

### Snap (Ubuntu)

```bash
# Установка
sudo snap install vlc

# Список установленных
snap list
```

### AppImage

- Не требует установки
- Один файл = одно приложение
- Скачал, сделал исполняемым, запустил

```bash
chmod +x application.AppImage
./application.AppImage
```

## Резюме

- Пакетные менеджеры - основной способ установки ПО в Linux
- Два семейства: Debian (apt/dpkg) и Red Hat (yum/dnf/rpm)
- Всегда используйте официальные репозитории когда возможно
- Регулярно обновляйте систему для безопасности
- Новые форматы (Flatpak, Snap) решают проблему совместимости между дистрибутивами
