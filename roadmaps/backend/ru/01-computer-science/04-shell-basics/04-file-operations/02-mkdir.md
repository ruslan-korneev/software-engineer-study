# Команда mkdir

[prev: 01-wildcards](./01-wildcards.md) | [next: 03-cp](./03-cp.md)

---
## Что такое mkdir?

**mkdir** (make directory) — команда для создания новых директорий (папок).

## Синтаксис

```bash
mkdir [опции] директория...
```

## Базовое использование

### Создание одной директории
```bash
$ mkdir projects
$ ls
projects/
```

### Создание нескольких директорий
```bash
$ mkdir docs images scripts
$ ls
docs/  images/  scripts/
```

### Создание с указанием пути
```bash
$ mkdir /home/user/newdir
$ mkdir ~/newproject
```

## Основные опции

### -p (parents) — создать родительские директории

Создаёт всю цепочку директорий, если они не существуют:

```bash
# Без -p — ошибка, если parent не существует
$ mkdir projects/app/src
mkdir: cannot create directory 'projects/app/src': No such file or directory

# С -p — создаёт всю структуру
$ mkdir -p projects/app/src
$ tree projects
projects/
└── app/
    └── src/
```

Также `-p` не выдаёт ошибку, если директория уже существует:
```bash
$ mkdir existing_dir
mkdir: cannot create directory 'existing_dir': File exists

$ mkdir -p existing_dir
# Без ошибки
```

### -v (verbose) — подробный вывод

```bash
$ mkdir -v one two three
mkdir: created directory 'one'
mkdir: created directory 'two'
mkdir: created directory 'three'
```

### -m (mode) — установить права доступа

```bash
$ mkdir -m 755 public
$ mkdir -m 700 private
$ ls -l
drwxr-xr-x  2 user group 4096 public
drwx------  2 user group 4096 private
```

## Практические примеры

### Создание структуры проекта
```bash
$ mkdir -p myproject/{src,tests,docs,config}
$ tree myproject
myproject/
├── config/
├── docs/
├── src/
└── tests/
```

### Структура для веб-проекта
```bash
$ mkdir -p webapp/{css,js,images,fonts}
$ mkdir -p webapp/src/{components,pages,utils}
```

### Структура для Python-проекта
```bash
$ mkdir -p pyproject/{src/mypackage,tests,docs}
$ touch pyproject/src/mypackage/__init__.py
```

### Создание датированных директорий
```bash
$ mkdir "backup-$(date +%Y-%m-%d)"
$ ls
backup-2024-12-26/
```

### Создание с номерами
```bash
$ mkdir chapter{01..10}
$ ls
chapter01/  chapter02/  chapter03/  ...  chapter10/
```

## Права доступа по умолчанию

Права новой директории определяются значением `umask`:

```bash
$ umask
0022

$ mkdir test
$ ls -ld test
drwxr-xr-x  2 user group 4096 test

# 777 - 022 = 755 (rwxr-xr-x)
```

Чтобы изменить:
```bash
$ mkdir -m 700 private_dir   # только владелец
$ mkdir -m 777 public_dir    # все права всем (не рекомендуется)
```

## Обработка ошибок

### Директория уже существует
```bash
$ mkdir existing
mkdir: cannot create directory 'existing': File exists

# Решение: используйте -p
$ mkdir -p existing   # OK
```

### Нет прав на создание
```bash
$ mkdir /root/test
mkdir: cannot create directory '/root/test': Permission denied

# Решение: используйте sudo
$ sudo mkdir /root/test
```

### Родительская директория не существует
```bash
$ mkdir a/b/c/d
mkdir: cannot create directory 'a/b/c/d': No such file or directory

# Решение: используйте -p
$ mkdir -p a/b/c/d
```

## Сочетание с другими командами

### Создать и перейти
```bash
$ mkdir newdir && cd newdir
```

### Создать и изменить владельца
```bash
$ sudo mkdir /var/www/mysite && sudo chown user:group /var/www/mysite
```

### Создать несколько уровней с разными правами
```bash
$ mkdir -p -m 755 project/public
$ mkdir -p -m 700 project/private
```

## Альтернативы mkdir

### install -d
```bash
$ install -d -m 755 mydir
```
Создаёт директорию с правами, аналогично `mkdir -p -m`.

### В скриптах
```bash
#!/bin/bash
DIR="/path/to/dir"
[ -d "$DIR" ] || mkdir -p "$DIR"
```

## Советы

1. **Всегда используйте -p** для надёжности:
   ```bash
   mkdir -p path/to/dir   # не упадёт, если уже существует
   ```

2. **Brace expansion** для создания структуры:
   ```bash
   mkdir -p project/{src,test,docs}/{main,utils}
   ```

3. **Проверяйте права** перед созданием в системных директориях

4. **Используйте -v** для отладки:
   ```bash
   mkdir -pv complex/path/structure
   ```

---

[prev: 01-wildcards](./01-wildcards.md) | [next: 03-cp](./03-cp.md)
