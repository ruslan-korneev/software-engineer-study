# Символические и жёсткие ссылки

[prev: 03-less-command](./03-less-command.md) | [next: 01-wildcards](../04-file-operations/01-wildcards.md)

---
## Что такое ссылка?

**Ссылка** — это способ создать несколько имён для одного файла или директории. В Unix/Linux существует два типа ссылок: жёсткие (hard links) и символические (symbolic/soft links).

## Как хранятся файлы

Для понимания ссылок нужно знать, как работает файловая система:

```
Имя файла → inode → Данные на диске
              ↑
          метаданные
          (права, владелец,
          размер, время)
```

- **inode** — структура данных, содержащая метаданные и указатель на данные
- Имя файла — это просто "указатель" на inode

## Жёсткие ссылки (Hard Links)

**Жёсткая ссылка** — это дополнительное имя для существующего inode. Все жёсткие ссылки равноправны — нет "оригинала" и "копии".

### Создание жёсткой ссылки

```bash
$ ln original.txt hardlink.txt
```

### Схема работы

```
original.txt ───┐
                ├──→ inode 12345 → Данные
hardlink.txt ───┘
```

### Характеристики жёстких ссылок

1. **Указывают на тот же inode**
   ```bash
   $ ls -li
   12345 -rw-r--r-- 2 user group 100 original.txt
   12345 -rw-r--r-- 2 user group 100 hardlink.txt
   # Одинаковый inode (12345), счётчик ссылок = 2
   ```

2. **Нельзя создать для директорий** (кроме `.` и `..`)
   ```bash
   $ ln dir/ linkdir
   ln: dir/: hard link not allowed for directory
   ```

3. **Нельзя создать между разными файловыми системами**
   ```bash
   $ ln /home/file /mnt/usb/link
   ln: failed to create hard link: Invalid cross-device link
   ```

4. **Файл удаляется только когда удалены все ссылки**
   ```bash
   $ rm original.txt
   # Данные остаются, hardlink.txt всё ещё работает
   ```

## Символические ссылки (Symbolic/Soft Links)

**Символическая ссылка** — это специальный файл, содержащий путь к другому файлу.

### Создание символической ссылки

```bash
$ ln -s original.txt symlink.txt
$ ln -s /path/to/target linkname
```

### Схема работы

```
symlink.txt → "original.txt" (текст пути)
                    │
                    ↓
             original.txt → inode → Данные
```

### Характеристики символических ссылок

1. **Имеют собственный inode**
   ```bash
   $ ls -li
   12345 -rw-r--r-- 1 user group 100 original.txt
   67890 lrwxrwxrwx 1 user group  12 symlink.txt -> original.txt
   # Разные inode, тип 'l' = symbolic link
   ```

2. **Могут указывать на директории**
   ```bash
   $ ln -s /var/log logs
   $ cd logs    # работает!
   ```

3. **Могут пересекать файловые системы**
   ```bash
   $ ln -s /home/file /mnt/usb/link    # работает
   ```

4. **Могут быть "битыми"** (указывать на несуществующий файл)
   ```bash
   $ rm original.txt
   $ cat symlink.txt
   cat: symlink.txt: No such file or directory
   ```

## Сравнение

| Характеристика | Жёсткая ссылка | Символическая ссылка |
|----------------|----------------|---------------------|
| Собственный inode | Нет (тот же) | Да |
| На директории | Нет | Да |
| Между ФС | Нет | Да |
| Может быть битой | Нет | Да |
| Размер | Не занимает места | Размер = длина пути |

## Практические примеры

### Создание версионных ссылок
```bash
$ ls -la /usr/bin/python*
python -> python3.10
python3 -> python3.10
python3.10
```

Легко переключить версию:
```bash
$ ln -sf python3.11 /usr/bin/python3
```

### Ссылка на конфигурацию
```bash
$ ln -s /etc/nginx/sites-available/mysite /etc/nginx/sites-enabled/
```

### Ссылка на часто используемые директории
```bash
$ ln -s /very/long/path/to/project ~/project
$ cd ~/project    # быстрый доступ
```

### Жёсткая ссылка для бэкапа
```bash
$ ln important.txt important-backup.txt
# Удаление одного файла не удалит данные
```

## Определение типа ссылки

### Команда ls -l
```bash
$ ls -l
lrwxrwxrwx 1 user group 12 symlink -> original
-rw-r--r-- 2 user group 100 hardlink1
-rw-r--r-- 2 user group 100 hardlink2
```

- `l` в начале — символическая ссылка
- Число `2` — количество жёстких ссылок

### Команда readlink
```bash
$ readlink symlink
original.txt

$ readlink -f symlink    # полный путь
/home/user/original.txt
```

### Команда stat
```bash
$ stat original.txt
  Inode: 12345
  Links: 2        # количество жёстких ссылок
```

## Удаление ссылок

```bash
$ rm symlink          # удаляет только ссылку, не цель
$ unlink symlink      # альтернатива rm для ссылок
```

При удалении символической ссылки на директорию:
```bash
$ rm linkdir          # правильно
$ rm -r linkdir/      # ОПАСНО! Удалит содержимое целевой директории!
```

## Советы

1. Используйте **символические ссылки** в большинстве случаев — они гибче
2. **Жёсткие ссылки** полезны для бэкапов и экономии места
3. Будьте осторожны с `rm -r` на символических ссылках
4. Используйте `ln -s` с абсолютными путями для надёжности

---

[prev: 03-less-command](./03-less-command.md) | [next: 01-wildcards](../04-file-operations/01-wildcards.md)
