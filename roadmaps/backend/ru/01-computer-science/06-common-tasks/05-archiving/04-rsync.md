# rsync - синхронизация файлов и директорий

## Что такое rsync?

**rsync** (remote sync) - мощная утилита для синхронизации файлов и директорий. Главная особенность - передаёт только изменения (дельта-сжатие), что делает rsync идеальным для резервного копирования и репликации.

## Преимущества rsync

- **Инкрементальная передача** - только изменённые части файлов
- **Сжатие** - данные сжимаются при передаче
- **Сохранение атрибутов** - права, владелец, время, ссылки
- **Работа через SSH** - безопасная передача
- **Возобновление** - продолжение прерванной передачи
- **Исключения** - гибкая фильтрация файлов

## Базовый синтаксис

```bash
rsync [опции] источник назначение

# Локальная синхронизация
rsync -av /source/dir/ /dest/dir/

# На удалённый сервер
rsync -av /local/dir/ user@server:/remote/dir/

# С удалённого сервера
rsync -av user@server:/remote/dir/ /local/dir/
```

### Важно: слеш в конце пути

```bash
# С слешем - копируется содержимое директории
rsync -av source/ dest/
# Результат: dest/file1, dest/file2

# Без слеша - копируется сама директория
rsync -av source dest/
# Результат: dest/source/file1, dest/source/file2
```

## Основные опции

### Часто используемые комбинации

```bash
# -a = archive mode (сохранить всё)
# Включает: -rlptgoD (recursive, links, perms, times, group, owner, devices)
rsync -a source/ dest/

# -v = verbose (подробный вывод)
rsync -av source/ dest/

# -z = compress (сжатие при передаче)
rsync -avz source/ user@server:/dest/

# -P = --progress + --partial (прогресс и возобновление)
rsync -avP source/ dest/

# Типичная команда для бекапа
rsync -avzP source/ user@server:/backup/
```

### Все основные опции

| Опция | Описание |
|-------|----------|
| `-a, --archive` | Режим архива (-rlptgoD) |
| `-v, --verbose` | Подробный вывод |
| `-z, --compress` | Сжатие при передаче |
| `-P` | Прогресс + partial |
| `--progress` | Показать прогресс |
| `--partial` | Сохранять частично переданные файлы |
| `-r, --recursive` | Рекурсивно |
| `-l, --links` | Копировать символические ссылки |
| `-p, --perms` | Сохранять права |
| `-t, --times` | Сохранять время модификации |
| `-g, --group` | Сохранять группу |
| `-o, --owner` | Сохранять владельца |
| `-D` | Устройства и специальные файлы |
| `-H, --hard-links` | Сохранять жёсткие ссылки |
| `-n, --dry-run` | Пробный запуск |
| `--delete` | Удалять лишние файлы в назначении |
| `-u, --update` | Пропускать более новые файлы в назначении |
| `--existing` | Обновлять только существующие файлы |
| `--ignore-existing` | Не обновлять существующие файлы |

## Исключение файлов

```bash
# Исключить файлы по шаблону
rsync -av --exclude="*.log" source/ dest/

# Несколько исключений
rsync -av \
    --exclude="*.log" \
    --exclude="*.tmp" \
    --exclude="node_modules" \
    source/ dest/

# Исключить директорию
rsync -av --exclude="cache/" source/ dest/

# Из файла
rsync -av --exclude-from=exclude.txt source/ dest/

# Содержимое exclude.txt:
# *.log
# *.tmp
# .git/
# node_modules/
```

### Включение файлов

```bash
# Включить только определённые файлы
rsync -av --include="*.py" --exclude="*" source/ dest/

# Сложные правила (порядок важен!)
rsync -av \
    --include="*/" \           # все директории
    --include="*.py" \         # все .py файлы
    --exclude="*" \            # исключить остальное
    source/ dest/
```

### Filter rules

```bash
# Из файла фильтров
rsync -av --filter="merge rsync-filter" source/ dest/

# Содержимое rsync-filter:
# + *.py
# + *.txt
# - *.log
# - cache/
```

## Удаление лишних файлов

```bash
# Удалить файлы, которых нет в источнике
rsync -av --delete source/ dest/

# Удалить до начала передачи
rsync -av --delete-before source/ dest/

# Удалить после передачи
rsync -av --delete-after source/ dest/

# Удалить во время передачи (по умолчанию)
rsync -av --delete-during source/ dest/

# Удалить исключённые файлы в назначении
rsync -av --delete-excluded --exclude="*.tmp" source/ dest/
```

## Работа через SSH

```bash
# По умолчанию rsync использует SSH
rsync -avz source/ user@server:/dest/

# Указать порт SSH
rsync -avz -e "ssh -p 2222" source/ user@server:/dest/

# Использовать определённый ключ
rsync -avz -e "ssh -i ~/.ssh/key" source/ user@server:/dest/

# С опциями SSH
rsync -avz -e "ssh -o StrictHostKeyChecking=no" source/ user@server:/dest/
```

## Пробный запуск (dry-run)

```bash
# Показать что будет сделано, но не делать
rsync -avz --dry-run source/ dest/
rsync -avzn source/ dest/

# С выводом изменений
rsync -avzn --itemize-changes source/ dest/

# Формат itemize-changes:
# >f..t...... file.txt   # файл будет передан
# .d..t...... directory/ # директория обновится
# *deleting   oldfile    # будет удалён (с --delete)
```

## Ограничение скорости

```bash
# Ограничить скорость (KB/s)
rsync -avz --bwlimit=1000 source/ dest/

# Мегабайты в секунду
rsync -avz --bwlimit=10M source/ dest/
```

## Сохранение частичных файлов

```bash
# Сохранять частично переданные файлы
rsync -av --partial source/ dest/

# Сохранять в отдельную директорию
rsync -av --partial-dir=.rsync-partial source/ dest/

# -P = --partial + --progress
rsync -avP source/ dest/
```

## Практические сценарии

### Резервное копирование

```bash
# Простой бекап
rsync -avz /home/user/ /backup/home/

# С удалением устаревших файлов
rsync -avz --delete /home/user/ /backup/home/

# Исключить кеш и временные файлы
rsync -avz --delete \
    --exclude=".cache" \
    --exclude=".local/share/Trash" \
    --exclude="node_modules" \
    /home/user/ /backup/home/
```

### Инкрементальный бекап с историей

```bash
#!/bin/bash
# Бекап с хард-линками на предыдущую версию

DATE=$(date +%Y-%m-%d)
SOURCE=/home/user/
DEST=/backup/
CURRENT=$DEST/current
HISTORY=$DEST/$DATE

# Создать новый бекап с хард-линками на current
rsync -avz --delete \
    --link-dest=$CURRENT \
    $SOURCE $HISTORY/

# Обновить ссылку current
rm -f $CURRENT
ln -s $HISTORY $CURRENT
```

### Синхронизация проекта с сервером

```bash
# Деплой (исключить .git, node_modules)
rsync -avz --delete \
    --exclude=".git" \
    --exclude="node_modules" \
    --exclude="*.env" \
    ./project/ user@server:/var/www/project/

# Только обновить изменённые файлы
rsync -avzu ./project/ user@server:/var/www/project/
```

### Зеркалирование сервера

```bash
# Полное зеркало
rsync -avzP --delete user@source:/data/ /local/mirror/

# Только проверить изменения
rsync -avzn --delete user@source:/data/ /local/mirror/
```

### Миграция сервера

```bash
# Скопировать всю систему
sudo rsync -avxP \
    --exclude="/dev/*" \
    --exclude="/proc/*" \
    --exclude="/sys/*" \
    --exclude="/tmp/*" \
    --exclude="/run/*" \
    --exclude="/mnt/*" \
    --exclude="/media/*" \
    / user@newserver:/

# -x = не переходить на другие файловые системы
```

## Rsync daemon

Для регулярных синхронизаций можно запустить rsync как демон:

```bash
# /etc/rsyncd.conf
[backup]
    path = /backup
    read only = no
    list = yes
    uid = nobody
    gid = nogroup
    auth users = backupuser
    secrets file = /etc/rsyncd.secrets

# Запуск
rsync --daemon

# Подключение
rsync -avz source/ rsync://backupuser@server/backup/
```

## Альтернативы rsync

### rclone - для облачных хранилищ

```bash
# Установка
curl https://rclone.org/install.sh | sudo bash

# Синхронизация с облаком
rclone sync /local/dir remote:bucket/dir

# Поддержка: Google Drive, S3, Dropbox, и др.
```

### lsyncd - реальное время

```bash
# Непрерывная синхронизация (inotify + rsync)
sudo apt install lsyncd

# Конфигурация /etc/lsyncd/lsyncd.conf.lua
sync {
    default.rsync,
    source = "/home/user/documents",
    target = "/backup/documents"
}
```

## Отладка и устранение проблем

```bash
# Подробный вывод
rsync -avvv source/ dest/

# Показать, почему файл передаётся
rsync -avvi source/ dest/

# Проверить что будет передано
rsync -avzn source/ dest/

# Показать статистику
rsync -av --stats source/ dest/

# Логирование
rsync -av --log-file=rsync.log source/ dest/
```

## Резюме команд

| Задача | Команда |
|--------|---------|
| Базовая синхронизация | `rsync -av source/ dest/` |
| Со сжатием | `rsync -avz source/ dest/` |
| С прогрессом | `rsync -avP source/ dest/` |
| На сервер | `rsync -avz source/ user@server:/dest/` |
| С удалением | `rsync -av --delete source/ dest/` |
| Исключить | `rsync -av --exclude="*.log" source/ dest/` |
| Dry-run | `rsync -avn source/ dest/` |
| Ограничить скорость | `rsync -av --bwlimit=1000 source/ dest/` |
| SSH порт | `rsync -av -e "ssh -p 2222" source/ dest/` |

### Популярные комбинации

```bash
# Бекап
rsync -avzP --delete source/ dest/

# Деплой
rsync -avz --delete --exclude=".git" source/ server:/dest/

# Зеркало
rsync -avz --delete user@source:/data/ /mirror/
```
