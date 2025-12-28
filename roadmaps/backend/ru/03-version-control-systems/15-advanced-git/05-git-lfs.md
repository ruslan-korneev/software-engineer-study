# Git LFS

[prev: 04-git-attributes](./04-git-attributes.md) | [next: 06-submodules](./06-submodules.md)
---

## Что такое Git LFS

**Git LFS** (Large File Storage) - это расширение Git для работы с большими файлами. Вместо хранения больших файлов непосредственно в репозитории, Git LFS хранит указатели (pointers) на файлы, а сами файлы размещает на отдельном сервере.

### Проблема больших файлов в Git

Git хранит полную историю всех файлов. Для больших бинарных файлов это создает проблемы:

- **Раздувание репозитория** - каждая версия большого файла занимает место
- **Медленное клонирование** - нужно скачать всю историю
- **Неэффективный diff** - бинарные файлы не сжимаются дельтами
- **Проблемы производительности** - операции git становятся медленными

### Как работает Git LFS

```
Обычный Git:              Git LFS:

repo/                     repo/
├── .git/                 ├── .git/
│   └── objects/          │   └── lfs/
│       └── image.psd     │       └── objects/
│           (500MB)       │           └── image.psd (500MB)
└── image.psd             └── image.psd
                              (pointer file, 130 bytes)
```

Pointer file выглядит так:
```
version https://git-lfs.github.com/spec/v1
oid sha256:4d7a214614ab2935c943f9e0ff69d22eadbb8f32b1258daaa5e2ca24d17e2393
size 524288000
```

## Установка и настройка

### Установка Git LFS

```bash
# macOS (Homebrew)
brew install git-lfs

# Ubuntu/Debian
sudo apt install git-lfs

# Windows (Chocolatey)
choco install git-lfs

# Или скачать с https://git-lfs.github.com
```

### Инициализация в системе

```bash
# Один раз для пользователя (добавляет hooks в global config)
git lfs install

# Вывод:
# Updated git hooks.
# Git LFS initialized.

# Проверить версию
git lfs version
```

### Инициализация в репозитории

```bash
# Перейти в репозиторий
cd my-project

# Инициализировать LFS (создает .gitattributes если нужно)
git lfs install

# Или установить только для этого репозитория
git lfs install --local
```

## Основные команды

### git lfs track - отслеживание файлов

```bash
# Отслеживать файлы по расширению
git lfs track "*.psd"
git lfs track "*.zip"
git lfs track "*.mp4"

# Отслеживать конкретный файл
git lfs track "data/large-dataset.csv"

# Отслеживать директорию
git lfs track "assets/**"

# Это добавляет записи в .gitattributes:
# *.psd filter=lfs diff=lfs merge=lfs -text
```

### git lfs untrack - прекращение отслеживания

```bash
# Прекратить отслеживание типа файлов
git lfs untrack "*.psd"

# После этого нужно переконвертировать файлы обратно
git add --renormalize .
```

### git lfs ls-files - список LFS файлов

```bash
# Показать все LFS файлы в репозитории
git lfs ls-files

# Вывод:
# 4d7a214614 * assets/logo.psd
# a3b2c1d4e5 * videos/intro.mp4
# f6g7h8i9j0 - data/archive.zip

# * = pointer file в рабочей директории
# - = LFS объект загружен
```

### git lfs status - статус файлов

```bash
# Показать статус LFS файлов
git lfs status

# Вывод:
# Objects to be pushed to origin/main:
#   assets/new-logo.psd (LFS: 4d7a21)
#
# Objects to be committed:
#   assets/new-logo.psd (LFS: 4d7a21)
#
# Objects not staged for commit:
#   (none)
```

### git lfs fetch и pull

```bash
# Скачать LFS объекты без checkout
git lfs fetch

# Скачать для конкретной ветки
git lfs fetch origin feature/new-assets

# Скачать последние N дней
git lfs fetch --recent

# Скачать и checkout
git lfs pull

# Обычный git pull автоматически вызывает lfs pull
git pull
```

### git lfs push

```bash
# Отправить LFS объекты на сервер
git lfs push origin main

# Отправить все объекты
git lfs push origin --all
```

## Настройка .gitattributes для LFS

### Типичная конфигурация

```bash
# .gitattributes

# Изображения
*.png filter=lfs diff=lfs merge=lfs -text
*.jpg filter=lfs diff=lfs merge=lfs -text
*.jpeg filter=lfs diff=lfs merge=lfs -text
*.gif filter=lfs diff=lfs merge=lfs -text
*.psd filter=lfs diff=lfs merge=lfs -text
*.ai filter=lfs diff=lfs merge=lfs -text
*.sketch filter=lfs diff=lfs merge=lfs -text

# Видео
*.mp4 filter=lfs diff=lfs merge=lfs -text
*.mov filter=lfs diff=lfs merge=lfs -text
*.avi filter=lfs diff=lfs merge=lfs -text
*.webm filter=lfs diff=lfs merge=lfs -text

# Аудио
*.mp3 filter=lfs diff=lfs merge=lfs -text
*.wav filter=lfs diff=lfs merge=lfs -text
*.ogg filter=lfs diff=lfs merge=lfs -text

# Архивы
*.zip filter=lfs diff=lfs merge=lfs -text
*.tar.gz filter=lfs diff=lfs merge=lfs -text
*.rar filter=lfs diff=lfs merge=lfs -text

# Данные
*.csv filter=lfs diff=lfs merge=lfs -text
*.parquet filter=lfs diff=lfs merge=lfs -text
*.pkl filter=lfs diff=lfs merge=lfs -text

# Шрифты
*.woff filter=lfs diff=lfs merge=lfs -text
*.woff2 filter=lfs diff=lfs merge=lfs -text
*.ttf filter=lfs diff=lfs merge=lfs -text
*.otf filter=lfs diff=lfs merge=lfs -text

# 3D модели
*.fbx filter=lfs diff=lfs merge=lfs -text
*.obj filter=lfs diff=lfs merge=lfs -text
*.blend filter=lfs diff=lfs merge=lfs -text

# Бинарные данные проекта
*.dll filter=lfs diff=lfs merge=lfs -text
*.so filter=lfs diff=lfs merge=lfs -text
*.dylib filter=lfs diff=lfs merge=lfs -text
```

### Проверка конфигурации

```bash
# Показать что отслеживается LFS
git lfs track

# Вывод:
# Listing tracked patterns
#     *.psd (.gitattributes)
#     *.mp4 (.gitattributes)
#     *.zip (.gitattributes)
```

## Миграция существующего репозитория

### Миграция новых файлов

```bash
# Настроить отслеживание
git lfs track "*.psd"

# Добавить .gitattributes
git add .gitattributes
git commit -m "Add LFS tracking for PSD files"

# Новые .psd файлы автоматически будут в LFS
```

### Миграция истории (git lfs migrate)

```bash
# Показать какие файлы занимают место
git lfs migrate info

# Вывод:
# migrate: Fetching remote refs: ..., done.
# migrate: Sorting commits: ..., done.
# migrate: Examining commits: 100% (1000/1000), done.
# *.psd   524 MB  500/500 files(s)  100%
# *.zip   200 MB  50/50 files(s)    100%
# *.mp4   1.2 GB  20/20 files(s)    100%

# Мигрировать историю для конкретных типов
git lfs migrate import --include="*.psd,*.zip" --everything

# Мигрировать только определенные ветки
git lfs migrate import --include="*.psd" --include-ref=main --include-ref=develop

# После миграции нужен force push
git push origin --force --all
git push origin --force --tags
```

### Экспорт из LFS обратно в Git

```bash
# Вернуть файлы из LFS в обычный Git
git lfs migrate export --include="*.psd" --everything
```

## Лимиты и hosting

### GitHub

```
Бесплатный план:
- Хранилище: 1 GB
- Bandwidth: 1 GB/месяц

GitHub Pro/Team:
- Хранилище: 2 GB
- Bandwidth: 2 GB/месяц

Дополнительные пакеты:
- Data Pack: $5/месяц за 50 GB storage + 50 GB bandwidth
```

```bash
# Проверить использование на GitHub
# Settings -> Billing -> Git LFS Data
```

### GitLab

```
Бесплатный план:
- Хранилище: 5 GB на проект

GitLab Premium/Ultimate:
- Хранилище: 10 GB на проект
- Настраиваемые лимиты

Self-hosted:
- Без ограничений (зависит от сервера)
```

### Bitbucket

```
Все планы:
- Хранилище: 1 GB на репозиторий
- Дополнительное хранилище платное
```

### Self-hosted LFS сервер

```bash
# Настройка кастомного LFS сервера
git config lfs.url "https://lfs.mycompany.com/myrepo"

# Или в .lfsconfig (коммитится)
[lfs]
    url = https://lfs.mycompany.com/myrepo
```

## Продвинутое использование

### Частичный клон (sparse checkout с LFS)

```bash
# Клонировать без LFS файлов
GIT_LFS_SKIP_SMUDGE=1 git clone https://github.com/user/repo.git

# Скачать LFS файлы только для нужных путей
cd repo
git lfs pull --include="assets/icons/*"
```

### Настройка fetch для производительности

```bash
# Настроить сколько "недавних" дней кешировать
git config lfs.fetchrecentcommitsdays 7
git config lfs.fetchrecentrefsdays 7

# Включить параллельные загрузки
git config lfs.concurrenttransfers 8
```

### Работа без подключения к LFS серверу

```bash
# Работать только с локальным кешем
git config lfs.skipdownloaderrors true

# Показать какие файлы недоступны
git lfs pointer --file=image.psd --check
```

### Локирование файлов (Git LFS Locking)

```bash
# Заблокировать файл от редактирования другими
git lfs lock assets/logo.psd

# Показать заблокированные файлы
git lfs locks

# Вывод:
# assets/logo.psd    developer    ID:123

# Разблокировать файл
git lfs unlock assets/logo.psd

# Принудительно разблокировать (если заблокирован другим)
git lfs unlock assets/logo.psd --force
```

### Настройка lockable файлов

```bash
# В .gitattributes
*.psd filter=lfs diff=lfs merge=lfs -text lockable
```

## Best Practices

### Когда использовать Git LFS

**Используйте для:**
- Изображений (PSD, PNG, JPG > 1MB)
- Видео и аудио файлов
- Бинарных данных (датасетов, моделей ML)
- Архивов и сборок
- Шрифтов и 3D моделей

**Не используйте для:**
- Маленьких изображений (иконки, favicon)
- Текстовых файлов (даже больших)
- Файлов, которые часто изменяются (LFS хранит полные копии)

### Рекомендации

1. **Настройте LFS до добавления больших файлов**
   ```bash
   # Сначала track, потом add
   git lfs track "*.psd"
   git add .gitattributes
   git add assets/
   git commit -m "Add design assets with LFS"
   ```

2. **Документируйте использование LFS**
   ```markdown
   # README.md
   ## Требования
   Этот репозиторий использует Git LFS. Установите его перед клонированием:
   brew install git-lfs && git lfs install
   ```

3. **Проверяйте размер репозитория**
   ```bash
   git lfs ls-files -s
   ```

4. **Регулярно очищайте кеш**
   ```bash
   git lfs prune
   ```

### Типичные проблемы

```bash
# Файл не отслеживается LFS
git lfs track "*.psd"
git rm --cached large-file.psd
git add large-file.psd
git commit -m "Move to LFS"

# LFS файл показывает pointer вместо контента
git lfs pull

# Ошибка "this exceeds GitHub's file size limit"
# Файл был добавлен до настройки LFS
git lfs migrate import --include="large-file.psd" --everything
git push --force
```

## Полезные команды

```bash
# Информация о LFS в репозитории
git lfs env

# Очистить старые LFS объекты
git lfs prune

# Проверить целостность LFS объектов
git lfs fsck

# Показать размер LFS хранилища
git lfs ls-files -s | awk '{sum+=$1} END {print sum/1024/1024 " MB"}'

# Дедупликация
git lfs dedup
```

## Заключение

Git LFS - необходимый инструмент для проектов с большими бинарными файлами. Он позволяет:
- Держать репозиторий компактным
- Ускорить клонирование и операции Git
- Эффективно работать с медиа-файлами

Ключ к успеху - настроить LFS до добавления больших файлов и документировать это для команды.

---
[prev: 04-git-attributes](./04-git-attributes.md) | [next: 06-submodules](./06-submodules.md)