# Команды su и sudo

## Обзор

**su** и **sudo** — команды для выполнения действий от имени другого пользователя (обычно root).

| Команда | Описание |
|---------|----------|
| `su` | Switch User — сменить пользователя полностью |
| `sudo` | Super User Do — выполнить одну команду от root |

## Команда su

### Синтаксис
```bash
su [опции] [пользователь]
```

### Базовое использование

```bash
$ su                    # стать root (нужен пароль root)
Password:
#

$ su alice              # стать пользователем alice
Password:               # пароль alice
alice$

$ su - alice            # стать alice с его окружением
```

### Опции su

| Опция | Описание |
|-------|----------|
| `-` или `-l` | Login shell (полная смена окружения) |
| `-c "command"` | Выполнить команду и выйти |
| `-s /bin/bash` | Использовать указанную оболочку |

### Примеры

```bash
# Полностью стать root
$ su -
Password:
root#

# Выполнить одну команду
$ su -c "apt update"
Password:
# команда выполнена, вы снова обычный пользователь

# С определённой оболочкой
$ su -s /bin/zsh alice
```

### Разница между su и su -

```bash
# su — сохраняет окружение текущего пользователя
$ su
# pwd → /home/user (остались тут)
# echo $HOME → /home/user

# su - — полная смена окружения
$ su -
# pwd → /root
# echo $HOME → /root
```

## Команда sudo

### Синтаксис
```bash
sudo [опции] команда
```

### Базовое использование

```bash
$ sudo apt update       # выполнить apt update от root
[sudo] password for user:
# вводите свой пароль, не root!

$ sudo -i               # интерактивный shell root
#

$ sudo -u alice cat /home/alice/file.txt
# выполнить от имени alice
```

### Основные опции

| Опция | Описание |
|-------|----------|
| `-i` | Login shell как root |
| `-s` | Shell как root (без login) |
| `-u user` | От имени указанного пользователя |
| `-k` | Забыть пароль (требовать снова) |
| `-v` | Продлить время действия пароля |
| `-l` | Показать разрешённые команды |

### Примеры

```bash
# Редактировать системный файл
$ sudo vim /etc/hosts

# Стать root интерактивно
$ sudo -i

# Выполнить от другого пользователя
$ sudo -u www-data touch /var/www/html/test.txt

# Посмотреть свои права sudo
$ sudo -l
User alice may run the following commands on this host:
    (ALL : ALL) ALL

# Сбросить кэш пароля
$ sudo -k
```

## Сравнение su и sudo

| Аспект | su | sudo |
|--------|----|----|
| Пароль | Целевого пользователя (root) | Свой пароль |
| Логирование | Минимальное | Подробное (/var/log/auth.log) |
| Гранулярность | Всё или ничего | Можно ограничить команды |
| Рекомендуется | Для полной смены пользователя | Для отдельных команд |

## Настройка sudo

### Файл /etc/sudoers

Редактировать только через `visudo`:
```bash
$ sudo visudo
```

### Примеры конфигурации

```sudoers
# Разрешить alice всё
alice ALL=(ALL:ALL) ALL

# Разрешить группе admin всё
%admin ALL=(ALL:ALL) ALL

# Разрешить bob перезапускать nginx без пароля
bob ALL=(ALL) NOPASSWD: /usr/bin/systemctl restart nginx

# Разрешить группе developers деплой
%developers ALL=(ALL) NOPASSWD: /usr/local/bin/deploy.sh
```

### Формат строки sudoers

```
пользователь хост=(от_пользователя:от_группы) команды
```

## Практические сценарии

### Установка пакетов
```bash
$ sudo apt install nginx
$ sudo yum install httpd
```

### Редактирование системных файлов
```bash
$ sudo nano /etc/nginx/nginx.conf
$ sudo vim /etc/hosts
```

### Управление сервисами
```bash
$ sudo systemctl start nginx
$ sudo systemctl enable docker
```

### Работа с Docker
```bash
# Без добавления в группу docker
$ sudo docker ps

# Или добавить в группу (рекомендуется)
$ sudo usermod -aG docker $USER
# После перелогина docker доступен без sudo
```

### Просмотр логов
```bash
$ sudo tail -f /var/log/syslog
$ sudo less /var/log/auth.log
```

## Безопасность

### Не делайте

```bash
# Не запускайте GUI-программы через sudo
$ sudo firefox           # Плохо!

# Не редактируйте чужие файлы без необходимости
$ sudo vim /home/alice/file.txt

# Избегайте sudo bash
$ sudo bash              # Лучше sudo -i
```

### Рекомендации

1. **Минимальные права** — давайте только необходимые права
2. **Используйте sudo** вместо su для отдельных команд
3. **Логи** — sudo логирует всё, что важно для аудита
4. **NOPASSWD осторожно** — только для безопасных команд
5. **Не сидите под root** — выполняйте отдельные команды

## Часто используемые комбинации

```bash
# Повторить последнюю команду с sudo
$ sudo !!

# Редактировать с сохранением прав
$ sudoedit /etc/hosts
# или
$ sudo -e /etc/hosts

# Стать root для серии команд
$ sudo -i
# выполнить команды...
# exit
```
