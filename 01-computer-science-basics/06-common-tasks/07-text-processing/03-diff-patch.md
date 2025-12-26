# diff и patch - сравнение и применение изменений

## diff - сравнение файлов

### Что такое diff?

**diff** сравнивает два файла и показывает различия между ними. Используется для:

- Просмотра изменений между версиями файлов
- Создания патчей для распространения изменений
- Отладки и code review

### Базовое использование

```bash
# Сравнить два файла
diff file1.txt file2.txt

# Сравнить файл с stdin
cat file2.txt | diff file1.txt -

# Сравнить директории
diff dir1/ dir2/
```

### Форматы вывода

#### Обычный формат (по умолчанию)

```bash
diff old.txt new.txt
# 2c2
# < old line 2
# ---
# > new line 2
# 5d4
# < deleted line
# 6a6
# > added line

# Обозначения:
# a - add (добавлено)
# d - delete (удалено)
# c - change (изменено)
# < - строка из первого файла
# > - строка из второго файла
```

#### Unified формат (-u)

Наиболее распространённый формат для патчей:

```bash
diff -u old.txt new.txt
# --- old.txt  2024-01-15 10:00:00
# +++ new.txt  2024-01-15 11:00:00
# @@ -1,5 +1,5 @@
#  line 1
# -old line 2
# +new line 2
#  line 3
#  line 4
# -deleted line
# +added line

# Обозначения:
# --- старый файл
# +++ новый файл
# @@ диапазоны строк
# - удалённая строка
# + добавленная строка
# (пробел) неизменённая строка
```

#### Context формат (-c)

```bash
diff -c old.txt new.txt
# *** old.txt  2024-01-15 10:00:00
# --- new.txt  2024-01-15 11:00:00
# ***************
# *** 1,5 ****
#   line 1
# ! old line 2
#   line 3
# --- 1,5 ----
#   line 1
# ! new line 2
#   line 3
```

#### Side-by-side (-y)

```bash
diff -y old.txt new.txt
# line 1                    line 1
# old line 2              | new line 2
# line 3                    line 3
# deleted line            <

# Ширина колонок
diff -y -W 80 old.txt new.txt

# Только различающиеся строки
diff -y --suppress-common-lines old.txt new.txt
```

### Полезные опции

```bash
# Игнорировать пробелы
diff -b old.txt new.txt      # игнорировать изменения в количестве пробелов
diff -w old.txt new.txt      # игнорировать все пробелы
diff -B old.txt new.txt      # игнорировать пустые строки

# Игнорировать регистр
diff -i old.txt new.txt

# Рекурсивное сравнение директорий
diff -r dir1/ dir2/

# Краткий вывод (только имена файлов)
diff -q old.txt new.txt
# Files old.txt and new.txt differ

diff -rq dir1/ dir2/

# Контекст (количество строк вокруг изменений)
diff -u -U 5 old.txt new.txt    # 5 строк контекста (по умолчанию 3)
diff -c -C 5 old.txt new.txt
```

### Исключение файлов

```bash
# Исключить файлы по шаблону
diff -r --exclude="*.pyc" dir1/ dir2/
diff -r --exclude=".git" dir1/ dir2/

# Несколько исключений
diff -r --exclude="*.log" --exclude="*.tmp" dir1/ dir2/

# Из файла
diff -r --exclude-from=exclude.txt dir1/ dir2/
```

### Создание патча

```bash
# Создать патч в unified формате
diff -u old.txt new.txt > changes.patch

# Для директории
diff -ruN old_dir/ new_dir/ > changes.patch
# -r рекурсивно
# -u unified формат
# -N обрабатывать отсутствующие файлы как пустые
```

## patch - применение изменений

### Что такое patch?

**patch** применяет изменения из файла патча к исходным файлам.

### Базовое использование

```bash
# Применить патч
patch < changes.patch

# Или указать файл
patch -i changes.patch

# Указать файл для патчинга
patch file.txt < changes.patch
patch -p0 < changes.patch
```

### Уровень пути (-p)

Опция `-p` определяет сколько компонентов пути отбросить:

```bash
# В патче: --- a/src/file.txt
#          +++ b/src/file.txt

patch -p0 < changes.patch    # использовать путь как есть: a/src/file.txt
patch -p1 < changes.patch    # отбросить a/: src/file.txt
patch -p2 < changes.patch    # отбросить a/src/: file.txt

# Обычно для git патчей используется -p1
```

### Откат патча (-R)

```bash
# Отменить применённый патч
patch -R < changes.patch

# Или
patch --reverse < changes.patch
```

### Пробный прогон (--dry-run)

```bash
# Проверить без применения
patch --dry-run < changes.patch

# Проверить можно ли откатить
patch -R --dry-run < changes.patch
```

### Работа с отклонениями

```bash
# При конфликте patch создаёт .rej файлы

# Принудительно применить (даже с ошибками)
patch -f < changes.patch
patch --force < changes.patch

# Создать бекап оригинала
patch -b < changes.patch          # создаёт file.txt.orig
patch -b -z .bak < changes.patch  # создаёт file.txt.bak

# Игнорировать уже применённые патчи
patch -N < changes.patch
```

### Полезные опции

```bash
# Тихий режим
patch -s < changes.patch

# Подробный вывод
patch -v < changes.patch

# Указать директорию
patch -d /path/to/dir < changes.patch

# Удалить пустые файлы после патчинга
patch -E < changes.patch
```

## Практические примеры

### Создание и применение патча

```bash
# 1. Создать патч
diff -u original.py modified.py > fix.patch

# 2. Проверить что будет применено
patch --dry-run original.py < fix.patch

# 3. Применить
patch original.py < fix.patch

# 4. При необходимости откатить
patch -R original.py < fix.patch
```

### Патч для проекта

```bash
# 1. Создать патч для всей директории
diff -ruN project_v1/ project_v2/ > update.patch

# 2. Проверить
cd project_v1/
patch -p1 --dry-run < ../update.patch

# 3. Применить
patch -p1 < ../update.patch
```

### Работа с Git

```bash
# Git создаёт патчи в совместимом формате
git diff > changes.patch
git diff HEAD~3 > last_3_commits.patch
git format-patch -1 HEAD    # патч для последнего коммита

# Применение git патча
git apply changes.patch

# Или через patch
patch -p1 < changes.patch
```

### Сравнение вывода команд

```bash
# Сравнить вывод двух команд
diff <(ls dir1/) <(ls dir2/)

# Сравнить файлы на разных серверах
diff <(ssh server1 cat /etc/config) <(ssh server2 cat /etc/config)

# Сравнить до и после
before=$(cat file.txt)
# ... изменения ...
echo "$before" | diff - file.txt
```

## Альтернативы

### colordiff

```bash
# Установка
sudo apt install colordiff

# Использование
colordiff old.txt new.txt
diff old.txt new.txt | colordiff

# Алиас
alias diff='colordiff'
```

### vimdiff

```bash
# Визуальное сравнение в Vim
vimdiff old.txt new.txt

# Три файла
vimdiff file1 file2 file3

# В Vim:
# ]c - следующее различие
# [c - предыдущее различие
# do - получить изменение из другого файла
# dp - отправить изменение в другой файл
```

### meld (GUI)

```bash
# Установка
sudo apt install meld

# Использование
meld file1.txt file2.txt
meld dir1/ dir2/
```

### sdiff

```bash
# Side-by-side сравнение с интерактивным редактированием
sdiff old.txt new.txt

# Создать объединённый файл
sdiff -o merged.txt old.txt new.txt
```

### comm - сравнение отсортированных файлов

```bash
# Для отсортированных файлов
comm file1.txt file2.txt
# Колонка 1: только в file1
# Колонка 2: только в file2
# Колонка 3: в обоих

# Только уникальные для file1
comm -23 file1.txt file2.txt

# Только общие
comm -12 file1.txt file2.txt

# Только уникальные для file2
comm -13 file1.txt file2.txt
```

## Резюме команд

| Команда | Описание |
|---------|----------|
| `diff f1 f2` | Сравнить файлы |
| `diff -u f1 f2` | Unified формат |
| `diff -c f1 f2` | Context формат |
| `diff -y f1 f2` | Side-by-side |
| `diff -r d1 d2` | Рекурсивно |
| `diff -q` | Только имена файлов |
| `diff -w` | Игнорировать пробелы |
| `patch < p.patch` | Применить патч |
| `patch -R` | Откатить патч |
| `patch -p1` | Убрать 1 уровень пути |
| `patch --dry-run` | Пробный прогон |
| `patch -b` | С бекапом |
