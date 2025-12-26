# Команда chmod

## Что такое chmod?

**chmod** (change mode) — команда для изменения прав доступа к файлам и директориям.

## Синтаксис

```bash
chmod [опции] режим файл...
```

## Два способа указания прав

### 1. Символьный режим

Формат: `[ugoa][+-=][rwx]`

| Кто | Описание |
|-----|----------|
| `u` | user (владелец) |
| `g` | group (группа) |
| `o` | others (остальные) |
| `a` | all (все) |

| Действие | Описание |
|----------|----------|
| `+` | Добавить право |
| `-` | Убрать право |
| `=` | Установить точно |

| Право | Описание |
|-------|----------|
| `r` | Чтение |
| `w` | Запись |
| `x` | Выполнение |

### Примеры символьного режима

```bash
$ chmod u+x script.sh      # добавить выполнение владельцу
$ chmod g-w file.txt       # убрать запись у группы
$ chmod o=r file.txt       # установить только чтение для остальных
$ chmod a+x script.sh      # добавить выполнение всем
$ chmod u+x,g+x script.sh  # несколько изменений
$ chmod ug=rw,o=r file.txt # владелец и группа rw, остальные r
```

### 2. Числовой (восьмеричный) режим

| Право | Значение |
|-------|----------|
| `r` | 4 |
| `w` | 2 |
| `x` | 1 |
| `-` | 0 |

Три цифры: owner, group, others

```bash
$ chmod 755 script.sh      # rwxr-xr-x
$ chmod 644 file.txt       # rw-r--r--
$ chmod 600 secrets.txt    # rw-------
$ chmod 700 private/       # rwx------
$ chmod 777 public.txt     # rwxrwxrwx (не рекомендуется!)
```

### Таблица популярных значений

| Число | Права | Типичное использование |
|-------|-------|----------------------|
| 755 | rwxr-xr-x | Исполняемые файлы, директории |
| 644 | rw-r--r-- | Обычные файлы |
| 700 | rwx------ | Приватные директории |
| 600 | rw------- | Приватные файлы (ключи SSH) |
| 664 | rw-rw-r-- | Файлы для совместной работы |
| 775 | rwxrwxr-x | Общие директории |

## Основные опции

### -R (recursive) — рекурсивно

```bash
$ chmod -R 755 myproject/
# Все файлы и поддиректории
```

### -v (verbose) — подробный вывод

```bash
$ chmod -v 644 file.txt
mode of 'file.txt' changed from 0755 (rwxr-xr-x) to 0644 (rw-r--r--)
```

### -c (changes) — только изменения

```bash
$ chmod -c 644 *.txt
# Выводит только реально изменённые
```

### --reference — скопировать права

```bash
$ chmod --reference=template.txt newfile.txt
# newfile.txt получит права как у template.txt
```

## Практические примеры

### Сделать скрипт исполняемым
```bash
$ chmod +x script.sh
# или
$ chmod 755 script.sh
```

### Защитить приватный ключ SSH
```bash
$ chmod 600 ~/.ssh/id_rsa
$ chmod 644 ~/.ssh/id_rsa.pub
```

### Настройка директории проекта
```bash
$ chmod 755 ~/project
$ chmod 644 ~/project/*.txt
$ chmod 755 ~/project/*.sh
```

### Рекурсивная настройка
```bash
# Все директории — 755
$ find /path -type d -exec chmod 755 {} \;

# Все файлы — 644
$ find /path -type f -exec chmod 644 {} \;

# Все скрипты — 755
$ find /path -name "*.sh" -exec chmod 755 {} \;
```

### Веб-сервер (типичные права)
```bash
$ chmod 755 /var/www/html
$ chmod 644 /var/www/html/*.html
$ chmod 755 /var/www/html/cgi-bin/*.cgi
```

## Специальные биты

### SUID (4000)
```bash
$ chmod u+s file          # установить SUID
$ chmod 4755 file         # числовой формат
```

### SGID (2000)
```bash
$ chmod g+s directory     # установить SGID
$ chmod 2755 directory    # числовой формат
```

### Sticky bit (1000)
```bash
$ chmod +t directory      # установить sticky
$ chmod 1755 directory    # числовой формат
```

### Комбинации
```bash
$ chmod 4755 file         # SUID + rwxr-xr-x
$ chmod 2775 directory    # SGID + rwxrwxr-x
$ chmod 1777 directory    # sticky + rwxrwxrwx (/tmp)
```

## Частые ошибки

### 1. chmod 777 — не делайте так!
```bash
$ chmod 777 file.txt      # Все могут всё — небезопасно!
# Вместо этого:
$ chmod 644 file.txt      # Чтение всем, запись владельцу
```

### 2. Рекурсивный chmod без разделения
```bash
$ chmod -R 755 project/   # Плохо: файлы станут исполняемыми

# Лучше:
$ find project/ -type d -exec chmod 755 {} \;
$ find project/ -type f -exec chmod 644 {} \;
```

### 3. Забыли про директории
```bash
# Для входа в директорию нужен x
$ chmod 644 mydir/        # Нельзя войти!
$ chmod 755 mydir/        # Правильно
```

## Проверка прав

```bash
$ ls -l file.txt
-rw-r--r-- 1 user group 0 Dec 26 10:00 file.txt

$ stat -c "%a %n" file.txt
644 file.txt
```

## Советы

1. **644 для файлов, 755 для директорий** — базовое правило
2. **600 для секретов** — SSH ключи, пароли
3. **Никогда 777** — это дыра в безопасности
4. **Используйте find** для раздельной настройки файлов и директорий
5. **Проверяйте права** командой `ls -l`
