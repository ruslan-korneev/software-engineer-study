# Команда pwd

[prev: 01-directory-tree](./01-directory-tree.md) | [next: 03-cd](./03-cd.md)

---
## Что такое pwd?

**pwd** (Print Working Directory) — команда, которая выводит полный путь к текущей рабочей директории. Это одна из самых простых и часто используемых команд.

## Синтаксис

```bash
pwd [опции]
```

## Базовое использование

```bash
$ pwd
/home/username/projects
```

Команда показывает, где именно вы находитесь в файловой системе.

## Опции pwd

### -L (logical) — логический путь
Показывает путь с символическими ссылками (по умолчанию):

```bash
$ pwd -L
/home/user/mylink
```

### -P (physical) — физический путь
Показывает реальный путь без символических ссылок:

```bash
$ pwd -P
/home/user/actual/directory
```

## Практический пример с символическими ссылками

```bash
# Создаём структуру
$ mkdir -p /tmp/actual/deep/path
$ ln -s /tmp/actual /tmp/shortcut

# Переходим по ссылке
$ cd /tmp/shortcut/deep/path

# Сравниваем выводы
$ pwd -L
/tmp/shortcut/deep/path    # путь через ссылку

$ pwd -P
/tmp/actual/deep/path      # реальный путь
```

## Переменная окружения PWD

Текущая директория также хранится в переменной `$PWD`:

```bash
$ echo $PWD
/home/username/projects

$ pwd
/home/username/projects
```

Это одно и то же значение. Переменная `$OLDPWD` хранит предыдущую директорию:

```bash
$ echo $OLDPWD
/home/username
```

## Когда использовать pwd

### 1. Ориентация в системе
После серии переходов `cd` легко потерять ориентацию:
```bash
$ cd ../../../somewhere/else
$ pwd    # где я теперь?
```

### 2. В скриптах
```bash
#!/bin/bash
SCRIPT_DIR=$(pwd)
echo "Скрипт запущен из: $SCRIPT_DIR"
```

### 3. Для копирования пути
```bash
$ pwd    # скопировать вывод
/home/user/very/long/path/to/project
```

## Сочетание с другими командами

### Сохранение пути в переменную
```bash
$ CURRENT=$(pwd)
$ echo $CURRENT
/home/username
```

### Использование в командах
```bash
$ tar -czf backup.tar.gz $(pwd)
```

## Альтернативы pwd

### Prompt (приглашение)
Многие shell отображают путь прямо в приглашении:

```bash
user@host:~/projects$        # ~ означает /home/user
user@host:/var/log$          # текущая директория /var/log
```

### Команда dirs (в bash/zsh)
```bash
$ dirs
~/projects
```

## Распространённые ошибки

### pwd показывает не тот путь
Если вы переходили по символическим ссылкам, `pwd` может показывать логический путь. Используйте `pwd -P` для физического:

```bash
$ pwd
/home/user/link        # это ссылка

$ pwd -P
/var/data/real         # реальный путь
```

## Итоги

- `pwd` — простая но важная команда для навигации
- По умолчанию показывает логический путь (со ссылками)
- `-P` показывает физический (реальный) путь
- Значение также доступно через переменную `$PWD`
- В скриптах используйте `$(pwd)` для получения текущего пути

---

[prev: 01-directory-tree](./01-directory-tree.md) | [next: 03-cd](./03-cd.md)
