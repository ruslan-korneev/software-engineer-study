# Команды chown и chgrp

## Что такое chown и chgrp?

- **chown** (change owner) — изменить владельца и/или группу файла
- **chgrp** (change group) — изменить только группу файла

Обе команды обычно требуют права root.

## Команда chown

### Синтаксис
```bash
chown [опции] владелец[:группа] файл...
chown [опции] :группа файл...
```

### Формы указания владельца

```bash
$ sudo chown alice file.txt           # изменить владельца
$ sudo chown alice:developers file.txt # владелец и группа
$ sudo chown :developers file.txt     # только группа
$ sudo chown alice: file.txt          # владелец и его группа по умолчанию
```

### Примеры

```bash
# Передать файл другому пользователю
$ sudo chown bob document.txt

# Изменить владельца и группу
$ sudo chown www-data:www-data /var/www/html/index.html

# Только группу (эквивалент chgrp)
$ sudo chown :developers project/

# Владелец и его основная группа
$ sudo chown alice: file.txt
```

### Опции chown

| Опция | Описание |
|-------|----------|
| `-R` | Рекурсивно (для директорий) |
| `-v` | Подробный вывод |
| `-c` | Показывать только изменения |
| `--reference` | Скопировать владельца с другого файла |
| `-h` | Изменить ссылку, а не цель |

### Рекурсивное изменение

```bash
# Изменить владельца всей директории
$ sudo chown -R alice:developers /home/alice/project

# С подробным выводом
$ sudo chown -Rv www-data:www-data /var/www/
changed ownership of '/var/www/html/index.html' from root:root to www-data:www-data
```

### По образцу

```bash
# Скопировать владельца/группу с другого файла
$ sudo chown --reference=template.txt newfile.txt
```

## Команда chgrp

### Синтаксис
```bash
chgrp [опции] группа файл...
```

### Примеры

```bash
# Изменить группу
$ sudo chgrp developers project/

# Рекурсивно
$ sudo chgrp -R www-data /var/www/html

# По образцу
$ sudo chgrp --reference=other.txt file.txt
```

### Опции chgrp

| Опция | Описание |
|-------|----------|
| `-R` | Рекурсивно |
| `-v` | Подробный вывод |
| `-c` | Только изменения |
| `--reference` | По образцу |
| `-h` | Изменить ссылку |

## Практические сценарии

### Настройка веб-сервера

```bash
# Владелец — www-data для веб-файлов
$ sudo chown -R www-data:www-data /var/www/html

# Добавить пользователя в группу для редактирования
$ sudo usermod -aG www-data alice
$ sudo chmod -R g+w /var/www/html
```

### Совместная работа над проектом

```bash
# Создать группу
$ sudo groupadd developers

# Добавить пользователей
$ sudo usermod -aG developers alice
$ sudo usermod -aG developers bob

# Настроить директорию
$ sudo chown -R :developers /opt/project
$ sudo chmod -R g+rwX /opt/project
$ sudo chmod g+s /opt/project   # SGID — новые файлы наследуют группу
```

### Исправление прав домашней директории

```bash
$ sudo chown -R alice:alice /home/alice
$ sudo chmod 700 /home/alice
```

### Docker и права

```bash
# Проблема: файлы созданы от root внутри контейнера
$ sudo chown -R $USER:$USER ./app
```

### Права для базы данных

```bash
$ sudo chown -R mysql:mysql /var/lib/mysql
$ sudo chmod 700 /var/lib/mysql
```

## Числовые ID вместо имён

Можно использовать UID и GID:

```bash
$ sudo chown 1000:1000 file.txt
$ sudo chown 33:33 /var/www/html   # 33 = www-data на Debian
```

Полезно когда:
- Пользователь не существует в системе
- Файлы с удалённой системы
- Контейнеры с другими ID

## Символические ссылки

По умолчанию chown/chgrp работают с целью ссылки:

```bash
$ ls -l
lrwxrwxrwx link -> target.txt
-rw-r--r-- target.txt

$ sudo chown bob link
# Изменится владелец target.txt, не ссылки!

$ sudo chown -h bob link
# Изменится владелец самой ссылки
```

## Обработка ошибок

### Нет прав
```bash
$ chown bob file.txt
chown: changing ownership of 'file.txt': Operation not permitted
# Нужен sudo!
```

### Пользователь не существует
```bash
$ sudo chown nonexistent file.txt
chown: invalid user: 'nonexistent'
```

### Файл не существует
```bash
$ sudo chown alice missing.txt
chown: cannot access 'missing.txt': No such file or directory
```

## Безопасность

### Не делайте

```bash
# Никогда не меняйте владельца системных файлов!
$ sudo chown user /etc/passwd     # ОПАСНО!
$ sudo chown user /bin/            # КРИТИЧЕСКИ ОПАСНО!
```

### Проверяйте рекурсивные операции

```bash
# Сначала посмотрите что будет изменено
$ find /path -user root

# Потом меняйте
$ sudo chown -R user:group /path
```

## Сводка

| Действие | Команда |
|----------|---------|
| Изменить владельца | `chown user file` |
| Изменить группу | `chown :group file` или `chgrp group file` |
| Изменить оба | `chown user:group file` |
| Рекурсивно | `chown -R user:group dir/` |
| По образцу | `chown --reference=ref file` |
| Только ссылку | `chown -h user link` |
