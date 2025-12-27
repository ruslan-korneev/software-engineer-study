# Git Attributes

## Что такое .gitattributes

**`.gitattributes`** - это файл конфигурации, который задает атрибуты для файлов и путей в репозитории. Он позволяет настраивать:
- Как Git обрабатывает окончания строк
- Какие файлы считать бинарными
- Как показывать diff для разных типов файлов
- Стратегии merge для конкретных файлов
- Статистику языков на GitHub

### Где размещать

```bash
# В корне репозитория (коммитится, применяется ко всем)
.gitattributes

# В поддиректории (применяется к файлам в ней)
docs/.gitattributes

# Глобально для пользователя
~/.config/git/attributes
# или указать в конфиге
git config --global core.attributesfile ~/.gitattributes
```

### Синтаксис

```
pattern attr1 attr2 ...
```

```bash
# Примеры
*.txt text
*.jpg binary
*.md text eol=lf
Makefile text eol=lf
```

## Основные атрибуты

### text - обработка текстовых файлов

```bash
# Автоопределение (Git решает сам)
* text=auto

# Явно указать как текстовый
*.txt text
*.md text
*.py text

# Явно указать как НЕ текстовый
*.png -text

# Установить без значения (включить атрибут)
*.sh text
```

### eol - окончания строк

```bash
# Всегда LF (Unix-style)
*.sh text eol=lf
*.py text eol=lf
Makefile text eol=lf

# Всегда CRLF (Windows-style)
*.bat text eol=crlf
*.cmd text eol=crlf

# Автоматически (зависит от core.autocrlf)
*.txt text
```

### binary - бинарные файлы

```bash
# Указать как бинарный (отключает text и diff)
*.png binary
*.jpg binary
*.pdf binary
*.exe binary

# Эквивалентно:
# *.png -text -diff
```

### diff - настройка отображения diff

```bash
# Отключить diff для файла
*.min.js -diff
package-lock.json -diff

# Использовать специальный драйвер diff
*.docx diff=word
*.pdf diff=pdf
*.png diff=exif
```

### merge - стратегии слияния

```bash
# Всегда брать нашу версию при конфликте
database.yml merge=ours

# Использовать кастомный драйвер merge
*.po merge=merge-po
```

### filter - фильтры контента

```bash
# Применить фильтр при checkout/commit
*.c filter=indent
secrets.yml filter=git-crypt
```

## Настройка окончаний строк

### Проблема

Разные ОС используют разные символы конца строки:
- **Unix/Linux/macOS**: LF (`\n`)
- **Windows**: CRLF (`\r\n`)

### Рекомендуемая конфигурация

```bash
# .gitattributes
# Автоматическая обработка текстовых файлов
* text=auto

# Явно текстовые файлы
*.md text
*.txt text
*.py text
*.js text
*.ts text
*.json text
*.yml text
*.yaml text
*.xml text
*.html text
*.css text
*.scss text

# Скрипты - всегда LF
*.sh text eol=lf
*.bash text eol=lf
Makefile text eol=lf
Dockerfile text eol=lf

# Windows скрипты - всегда CRLF
*.bat text eol=crlf
*.cmd text eol=crlf
*.ps1 text eol=crlf

# Бинарные файлы
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.ico binary
*.pdf binary
*.zip binary
*.gz binary
*.tar binary
*.woff binary
*.woff2 binary
*.ttf binary
*.eot binary
```

### Нормализация существующего репозитория

```bash
# После добавления .gitattributes
# Переиндексировать все файлы
git add --renormalize .

# Закоммитить изменения
git commit -m "Normalize line endings"
```

## Кастомные diff драйверы

### Настройка драйвера для Word документов

```bash
# .gitattributes
*.docx diff=word
```

```bash
# .gitconfig
[diff "word"]
    textconv = docx2txt
    binary = true
```

### Драйвер для PDF

```bash
# .gitattributes
*.pdf diff=pdf
```

```bash
# .gitconfig
[diff "pdf"]
    textconv = pdftotext -layout
```

### Драйвер для изображений (EXIF метаданные)

```bash
# .gitattributes
*.jpg diff=exif
*.jpeg diff=exif
*.png diff=exif
```

```bash
# .gitconfig
[diff "exif"]
    textconv = exiftool
```

### Драйвер для SQLite баз данных

```bash
# .gitattributes
*.db diff=sqlite
*.sqlite diff=sqlite
```

```bash
# .gitconfig
[diff "sqlite"]
    textconv = sqlite3 $1 .dump
```

## Merge стратегии для файлов

### merge=ours - всегда брать нашу версию

```bash
# Полезно для lock-файлов или конфигов
Gemfile.lock merge=ours
yarn.lock merge=ours
```

Требуется настройка драйвера:
```bash
git config --global merge.ours.driver true
```

### merge=binary - бинарное слияние

```bash
# Для бинарных файлов - выбор одной из версий
*.sketch merge=binary
```

### Кастомный merge драйвер

```bash
# .gitattributes
*.po merge=merge-po
```

```bash
# .gitconfig
[merge "merge-po"]
    name = Gettext PO file merge driver
    driver = msgcat -o %A %O %A %B
```

## Linguist атрибуты для GitHub

GitHub использует Linguist для определения языков в репозитории. Атрибуты позволяют настроить это поведение.

### linguist-language - указать язык

```bash
# Указать что файлы на определенном языке
*.blade.php linguist-language=PHP
*.tmpl linguist-language=Go
Jenkinsfile linguist-language=Groovy
```

### linguist-vendored - исключить из статистики

```bash
# Внешние/сгенерированные файлы
vendor/* linguist-vendored
node_modules/* linguist-vendored
third_party/* linguist-vendored
```

### linguist-generated - сгенерированные файлы

```bash
# Не показывать в PR и статистике
*.min.js linguist-generated
*.min.css linguist-generated
dist/* linguist-generated
build/* linguist-generated
```

### linguist-documentation - документация

```bash
# Исключить из статистики языков
docs/* linguist-documentation
*.md linguist-documentation
```

### linguist-detectable - включить/исключить из определения

```bash
# Исключить из определения основного языка
*.html linguist-detectable=false

# Включить (если обычно игнорируется)
*.inc linguist-detectable=true
```

## Атрибуты для export

### export-ignore - исключить из архива

```bash
# Не включать в git archive
.gitignore export-ignore
.gitattributes export-ignore
.github/ export-ignore
tests/ export-ignore
docs/ export-ignore
Makefile export-ignore
*.md export-ignore
```

### export-subst - подстановка переменных

```bash
# Подставить информацию о коммите при экспорте
VERSION.txt export-subst
```

Содержимое VERSION.txt:
```
Version: $Format:%H$
Date: $Format:%ci$
```

## Практические примеры

### Пример 1: Полный .gitattributes для веб-проекта

```bash
# Авто-определение для большинства файлов
* text=auto

#
# Исходный код
#
*.js text
*.jsx text
*.ts text
*.tsx text
*.css text
*.scss text
*.less text
*.html text
*.htm text
*.vue text
*.svelte text

#
# Конфигурация
#
*.json text
*.yaml text
*.yml text
*.xml text
*.toml text
*.ini text
.editorconfig text
.gitignore text
.env text
.env.* text

#
# Скрипты (Unix line endings)
#
*.sh text eol=lf
Makefile text eol=lf
Dockerfile text eol=lf

#
# Изображения
#
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.ico binary
*.svg text
*.webp binary

#
# Шрифты
#
*.woff binary
*.woff2 binary
*.ttf binary
*.eot binary
*.otf binary

#
# Архивы
#
*.zip binary
*.gz binary
*.tar binary
*.7z binary

#
# Diff настройки
#
*.min.js -diff
*.min.css -diff
package-lock.json -diff
yarn.lock -diff

#
# GitHub Linguist
#
*.min.js linguist-generated
*.min.css linguist-generated
dist/* linguist-generated
vendor/* linguist-vendored
node_modules/* linguist-vendored
docs/* linguist-documentation
```

### Пример 2: .gitattributes для Python проекта

```bash
* text=auto

# Python
*.py text diff=python
*.pyx text diff=python
*.pxd text diff=python
*.pyi text

# Requirements
requirements*.txt text
Pipfile text
Pipfile.lock text -diff

# Jupyter notebooks
*.ipynb text

# Docs
*.md text
*.rst text
*.txt text

# Configs
*.cfg text
*.ini text
*.yaml text
*.yml text
*.toml text
pyproject.toml text
setup.cfg text

# Scripts
*.sh text eol=lf
Makefile text eol=lf

# Data
*.csv text
*.json text

# Binary
*.pkl binary
*.pickle binary
*.npy binary
*.npz binary
*.h5 binary
*.hdf5 binary
*.parquet binary

# Images
*.png binary
*.jpg binary
*.jpeg binary
*.gif binary
*.svg text
```

## Проверка атрибутов

```bash
# Показать атрибуты для файла
git check-attr -a README.md

# Показать конкретный атрибут
git check-attr text README.md
git check-attr eol *.sh

# Показать для всех файлов
git ls-files | xargs git check-attr text eol binary
```

## Заключение

`.gitattributes` - важный файл для правильной настройки Git в команде:
- Гарантирует одинаковую обработку файлов на всех платформах
- Предотвращает проблемы с окончаниями строк
- Улучшает отображение diff и merge
- Настраивает представление репозитория на GitHub

Рекомендуется добавлять `.gitattributes` в каждый проект с самого начала.
