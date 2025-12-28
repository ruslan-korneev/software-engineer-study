# Теги в Git

[prev: 05-rewriting-history](./05-rewriting-history.md) | [next: 01-what-and-why](07-git-hooks/01-what-and-why.md)
---

## Введение

**Теги (tags)** — это именованные ссылки на определённые коммиты в истории. В отличие от веток, теги не перемещаются при новых коммитах — они навсегда привязаны к конкретному моменту в истории.

### Типичные применения тегов

1. **Версии релизов**: v1.0.0, v2.1.3
2. **Важные вехи**: beta, rc1, stable
3. **Точки отката**: before-refactoring, pre-migration
4. **Маркеры для CI/CD**: deploy-production

---

## Типы тегов

### Lightweight Tags (лёгкие)

Простая ссылка на коммит, без дополнительной информации.

```bash
# Создание
git tag v1.0.0

# По сути это просто файл с хэшем коммита
cat .git/refs/tags/v1.0.0
# abc123def456...
```

**Когда использовать**: Временные метки, личные закладки.

### Annotated Tags (аннотированные)

Полноценный объект Git с метаданными:
- Имя автора
- Email
- Дата создания
- Сообщение
- Опционально: GPG подпись

```bash
# Создание
git tag -a v1.0.0 -m "Release version 1.0.0"

# Это создаёт отдельный объект в Git
git cat-file -t v1.0.0
# tag

git cat-file -p v1.0.0
# object abc123def456...
# type commit
# tag v1.0.0
# tagger John Doe <john@example.com> 1703500000 +0300
#
# Release version 1.0.0
```

**Когда использовать**: Официальные релизы, публичные версии.

### Сравнение типов тегов

| Аспект | Lightweight | Annotated |
|--------|-------------|-----------|
| Метаданные | Нет | Да (автор, дата, сообщение) |
| GPG подпись | Нет | Опционально |
| Хранение | refs/tags/name | Отдельный объект |
| Рекомендация | Временное | Релизы |

---

## Создание тегов

### Создание lightweight тега

```bash
# На текущем HEAD
git tag v1.0.0

# На конкретном коммите
git tag v1.0.0 abc123
```

### Создание annotated тега

```bash
# С сообщением в командной строке
git tag -a v1.0.0 -m "Release version 1.0.0"

# С открытием редактора для сообщения
git tag -a v1.0.0

# На конкретном коммите
git tag -a v1.0.0 abc123 -m "Release version 1.0.0"
```

### Создание подписанного тега (GPG)

```bash
# Требует настроенного GPG ключа
git tag -s v1.0.0 -m "Signed release"

# Проверка подписи
git tag -v v1.0.0
```

### Создание тега задним числом

```bash
# Найти нужный коммит
git log --oneline
# abc123 Feature complete

# Создать тег на этом коммите
git tag -a v0.9.0 abc123 -m "Beta release"
```

---

## Просмотр тегов

### Список тегов

```bash
# Все теги
git tag

# С фильтрацией по паттерну
git tag -l "v1.*"
git tag --list "v2.0.*"

# Сортировка по версии (semver)
git tag -l --sort=-version:refname "v*"
# v2.1.0
# v2.0.1
# v2.0.0
# v1.9.0

# Сортировка по дате
git tag -l --sort=-creatordate
```

### Информация о теге

```bash
# Для annotated тега — полная информация
git show v1.0.0
# tag v1.0.0
# Tagger: John Doe <john@example.com>
# Date:   Mon Dec 25 10:30:00 2024 +0300
#
# Release version 1.0.0
#
# commit abc123...
# Author: ...
# ... diff ...

# Только информация о теге (без diff коммита)
git tag -v v1.0.0  # Для подписанных
git cat-file -p v1.0.0  # Для annotated

# Для lightweight — показывает просто коммит
git show v0.5.0
# commit abc123...
```

### Какой тег соответствует коммиту

```bash
# Найти тег, содержащий коммит
git tag --contains abc123

# Найти ближайший тег
git describe --tags
# v1.0.0-5-gabc123
# (5 коммитов после v1.0.0, текущий хэш abc123)

# Точное совпадение
git describe --tags --exact-match
```

---

## Работа с тегами на remote

### Push тегов

По умолчанию `git push` **не отправляет теги**!

```bash
# Отправить конкретный тег
git push origin v1.0.0

# Отправить все теги
git push origin --tags

# Отправить только annotated теги
git push origin --follow-tags
```

### Настройка автоматической отправки тегов

```bash
# В ~/.gitconfig
[push]
    followTags = true

# Теперь git push будет отправлять annotated теги автоматически
```

### Получение тегов

```bash
# Теги приходят с fetch/pull автоматически
git fetch
git fetch --tags  # Явно запросить все теги

# Получить конкретный тег
git fetch origin tag v1.0.0
```

### Просмотр remote тегов

```bash
git ls-remote --tags origin
# abc123... refs/tags/v1.0.0
# def456... refs/tags/v1.0.0^{}  # Разыменованный (для annotated)
# ghi789... refs/tags/v2.0.0
```

---

## Удаление тегов

### Локальное удаление

```bash
# Удалить один тег
git tag -d v1.0.0

# Удалить несколько
git tag -d v0.1.0 v0.2.0 v0.3.0
```

### Удаление на remote

```bash
# Способ 1: явное удаление
git push origin --delete v1.0.0

# Способ 2: push пустой ссылки
git push origin :refs/tags/v1.0.0

# Удаление всех тегов на remote (осторожно!)
git push origin --delete $(git tag -l)
```

### Переименование тега

Git не поддерживает переименование напрямую:

```bash
# Создать новый тег на том же коммите
git tag new-name old-name

# Удалить старый
git tag -d old-name
git push origin --delete old-name
git push origin new-name
```

---

## Checkout по тегу

### Просмотр состояния на момент тега

```bash
git checkout v1.0.0

# Предупреждение:
# You are in 'detached HEAD' state...
```

Это переводит в **detached HEAD** — вы не на ветке, а на конкретном коммите.

### Создание ветки от тега

```bash
# Способ 1: checkout и создание ветки
git checkout v1.0.0
git checkout -b hotfix-1.0.1

# Способ 2: одной командой
git checkout -b hotfix-1.0.1 v1.0.0
```

### Сравнение с тегом

```bash
# Diff с тегом
git diff v1.0.0
git diff v1.0.0 v2.0.0

# История между тегами
git log v1.0.0..v2.0.0
```

---

## Semantic Versioning (SemVer)

Стандарт версионирования, широко используемый с Git тегами.

### Формат: MAJOR.MINOR.PATCH

```
v1.2.3
│ │ │
│ │ └── PATCH: баг-фиксы, обратно совместимые
│ └──── MINOR: новая функциональность, обратно совместимая
└────── MAJOR: изменения, ломающие совместимость
```

### Примеры

```
v0.1.0  → Начальная разработка
v1.0.0  → Первый стабильный релиз
v1.1.0  → Добавлена новая фича
v1.1.1  → Исправлен баг
v1.2.0  → Ещё фичи
v2.0.0  → Breaking changes
```

### Pre-release версии

```
v1.0.0-alpha
v1.0.0-alpha.1
v1.0.0-beta
v1.0.0-beta.2
v1.0.0-rc.1   (release candidate)
v1.0.0
```

### Build metadata

```
v1.0.0+build.123
v1.0.0+20231225
```

### Сортировка SemVer тегов

```bash
# Правильная сортировка версий
git tag -l "v*" --sort=-version:refname

# Результат:
# v2.0.0
# v1.10.0  (не v1.2.0!)
# v1.9.0
# v1.2.0
# v1.1.0
```

---

## Практические примеры

### Релизный workflow

```bash
# 1. Убедиться, что main актуален
git checkout main
git pull

# 2. Создать release тег
git tag -a v1.2.0 -m "Release 1.2.0

Features:
- Add user authentication
- Add profile page

Bug fixes:
- Fix login redirect
"

# 3. Отправить тег
git push origin v1.2.0

# 4. CI/CD подхватит тег и создаст релиз
```

### Hotfix workflow

```bash
# 1. Создать ветку от тега
git checkout -b hotfix/1.2.1 v1.2.0

# 2. Внести исправления
git commit -m "Fix critical bug"

# 3. Создать новый тег
git tag -a v1.2.1 -m "Hotfix 1.2.1: Fix critical bug"

# 4. Merge в main и push
git checkout main
git merge hotfix/1.2.1
git push origin main --tags
```

### Просмотр изменений между версиями

```bash
# Что изменилось между релизами
git log v1.0.0..v1.1.0 --oneline

# Changelog
git log v1.0.0..v1.1.0 --pretty=format:"* %s" --no-merges

# Файлы, изменённые между версиями
git diff v1.0.0 v1.1.0 --stat
```

### Определение следующей версии

```bash
# Текущая версия
git describe --tags --abbrev=0
# v1.2.0

# Сколько коммитов с последнего тега
git rev-list v1.2.0..HEAD --count
# 15

# Полное описание
git describe --tags
# v1.2.0-15-gabc123
```

---

## Автоматизация тегирования

### Git hooks для проверки тегов

```bash
# .git/hooks/pre-push
#!/bin/bash
# Проверка, что теги соответствуют SemVer

while read local_ref local_sha remote_ref remote_sha; do
    if [[ "$local_ref" =~ ^refs/tags/v ]]; then
        tag_name="${local_ref#refs/tags/}"
        if ! [[ "$tag_name" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
            echo "Error: Tag $tag_name doesn't follow SemVer"
            exit 1
        fi
    fi
done
```

### Скрипт для создания релиза

```bash
#!/bin/bash
# release.sh

VERSION=$1
if [ -z "$VERSION" ]; then
    echo "Usage: ./release.sh v1.2.3"
    exit 1
fi

# Проверка формата
if ! [[ "$VERSION" =~ ^v[0-9]+\.[0-9]+\.[0-9]+$ ]]; then
    echo "Error: Version must be in format vX.Y.Z"
    exit 1
fi

# Создание тега
git tag -a "$VERSION" -m "Release $VERSION"
git push origin "$VERSION"

echo "Released $VERSION"
```

---

## Best Practices

### 1. Используйте annotated теги для релизов

```bash
# Хорошо
git tag -a v1.0.0 -m "First stable release"

# Плохо для релизов
git tag v1.0.0
```

### 2. Следуйте SemVer

```bash
# Понятно и предсказуемо
v1.0.0, v1.0.1, v1.1.0, v2.0.0

# Не делайте так
release-1, version-2, final, latest
```

### 3. Не изменяйте опубликованные теги

```bash
# Плохо: удалить и пересоздать тег
git push origin --delete v1.0.0
git tag -a v1.0.0 <new-commit> -m "..."
git push origin v1.0.0

# Хорошо: создать новую версию
git tag -a v1.0.1 -m "Fixed release"
```

### 4. Документируйте изменения в сообщении тега

```bash
git tag -a v1.5.0 -m "Release 1.5.0

New features:
- Feature A
- Feature B

Bug fixes:
- Fix issue #123
- Fix issue #456

Breaking changes:
- Removed deprecated API
"
```

### 5. Используйте --follow-tags

```bash
git config --global push.followTags true
# Annotated теги будут пушиться автоматически
```

---

## Резюме команд

| Команда | Описание |
|---------|----------|
| `git tag` | Список тегов |
| `git tag v1.0` | Создать lightweight тег |
| `git tag -a v1.0 -m "msg"` | Создать annotated тег |
| `git tag -a v1.0 <hash>` | Тег на конкретный коммит |
| `git tag -s v1.0 -m "msg"` | Подписанный тег |
| `git tag -l "v1.*"` | Фильтрация тегов |
| `git show v1.0` | Информация о теге |
| `git tag -d v1.0` | Удалить локальный тег |
| `git push origin v1.0` | Push одного тега |
| `git push origin --tags` | Push всех тегов |
| `git push origin --delete v1.0` | Удалить remote тег |
| `git checkout v1.0` | Перейти на тег |
| `git checkout -b branch v1.0` | Ветка от тега |
| `git describe --tags` | Описание текущей позиции |
| `git log v1.0..v2.0` | История между тегами |
| `git diff v1.0 v2.0` | Diff между тегами |

---
[prev: 05-rewriting-history](./05-rewriting-history.md) | [next: 01-what-and-why](07-git-hooks/01-what-and-why.md)