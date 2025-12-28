# Fetch

[prev: 02-push-pull](./02-push-pull.md) | [next: 05-merge-strategies](../05-merge-strategies.md)
---

## Что такое Git Fetch?

**Git fetch** — это команда, которая загружает коммиты, файлы и ссылки из удалённого репозитория в ваш локальный репозиторий, **не изменяя ваши локальные ветки**.

Это "безопасный" способ получить информацию о том, что происходит в удалённом репозитории, без автоматического слияния изменений.

## Fetch vs Pull

| Аспект | `git fetch` | `git pull` |
|--------|-------------|------------|
| Загрузка данных | Да | Да |
| Изменение рабочей директории | Нет | Да |
| Автоматический merge | Нет | Да |
| Безопасность | Безопасен | Может вызвать конфликты |
| Когда использовать | Для просмотра изменений | Для быстрой синхронизации |

```bash
# git pull = git fetch + git merge
git pull origin main
# Эквивалентно:
git fetch origin
git merge origin/main
```

## Базовое использование

### Fetch из конкретного remote

```bash
# Получить все изменения из origin
git fetch origin

# Получить конкретную ветку
git fetch origin main

# Получить все изменения из всех remotes
git fetch --all
```

### Fetch без аргументов

```bash
# Fetch из upstream текущей ветки (или origin по умолчанию)
git fetch
```

## Что происходит при fetch?

При выполнении `git fetch origin`:

1. Git связывается с удалённым репозиторием `origin`
2. Загружает все новые коммиты, ветки и теги
3. Обновляет **remote-tracking ветки** (например, `origin/main`)
4. **Не трогает** ваши локальные ветки и рабочую директорию

### Remote-tracking ветки

После fetch вы увидите ветки вида `origin/main`, `origin/develop`. Это **remote-tracking ветки** — локальные ссылки на состояние веток в удалённом репозитории.

```bash
# Просмотр remote-tracking веток
git branch -r

# Просмотр всех веток (локальных и remote-tracking)
git branch -a
```

Пример вывода:
```
* main
  feature
  remotes/origin/HEAD -> origin/main
  remotes/origin/main
  remotes/origin/develop
  remotes/origin/feature
```

## Просмотр полученных изменений

После fetch вы можете изучить изменения перед их интеграцией:

### Сравнение с локальной веткой

```bash
# Показать коммиты, которые есть в origin/main, но нет в локальном main
git log main..origin/main

# С подробностями
git log main..origin/main --oneline

# Показать diff
git diff main origin/main

# Показать изменённые файлы
git diff --stat main origin/main
```

### Просмотр содержимого

```bash
# Посмотреть файл из remote-tracking ветки
git show origin/main:path/to/file.txt

# Checkout файла (без переключения ветки)
git checkout origin/main -- path/to/file.txt
```

## Интеграция изменений после fetch

### Способ 1: Merge

```bash
git fetch origin
git merge origin/main
```

### Способ 2: Rebase

```bash
git fetch origin
git rebase origin/main
```

### Способ 3: Reset (осторожно!)

```bash
# Полностью синхронизировать локальную ветку с remote
# ВНИМАНИЕ: Потеряете локальные изменения!
git fetch origin
git reset --hard origin/main
```

## Расширенные возможности

### Fetch с prune

Удаляет локальные remote-tracking ветки, которых больше нет на remote:

```bash
# Fetch и очистить устаревшие ссылки
git fetch --prune origin
# или
git fetch -p origin

# Настроить prune по умолчанию
git config --global fetch.prune true
```

### Fetch тегов

```bash
# Fetch с тегами (по умолчанию)
git fetch --tags origin

# Fetch без тегов
git fetch --no-tags origin
```

### Fetch конкретного коммита или ветки

```bash
# Fetch только конкретную ветку
git fetch origin feature-branch

# Fetch конкретного refspec
git fetch origin main:refs/remotes/origin/main
```

### Shallow fetch (поверхностная загрузка)

```bash
# Получить только последние N коммитов
git fetch --depth=1 origin main

# "Углубить" историю позже
git fetch --deepen=10
# или получить полную историю
git fetch --unshallow
```

## Типичные сценарии использования

### Сценарий 1: Проверка изменений перед merge

```bash
# 1. Получить изменения
git fetch origin

# 2. Посмотреть, что нового
git log HEAD..origin/main --oneline

# 3. Просмотреть diff
git diff HEAD origin/main

# 4. Если всё хорошо — merge
git merge origin/main
```

### Сценарий 2: Работа с форком

```bash
# 1. Fetch из оригинального репозитория
git fetch upstream

# 2. Посмотреть изменения в upstream
git log main..upstream/main

# 3. Слить изменения в локальный main
git checkout main
git merge upstream/main

# 4. Push в свой форк
git push origin main
```

### Сценарий 3: Обновление информации о ветках коллег

```bash
# Fetch из всех remotes
git fetch --all

# Посмотреть все remote-tracking ветки
git branch -r

# Просмотреть конкретную ветку коллеги
git log origin/feature-colleague --oneline
```

### Сценарий 4: Проверка перед rebase

```bash
# 1. Fetch последние изменения
git fetch origin

# 2. Проверить, сколько коммитов нужно rebas'ить
git log origin/main..HEAD --oneline
# Например: 3 ваших коммита

# 3. Проверить, сколько коммитов добавилось в main
git log HEAD..origin/main --oneline
# Например: 5 новых коммитов

# 4. Выполнить rebase
git rebase origin/main
```

### Сценарий 5: CI/CD — получение без checkout

В скриптах CI часто нужно только получить информацию:

```bash
# Минимальный fetch для CI
git fetch --depth=1 origin main

# Проверить, есть ли изменения
git diff --quiet HEAD origin/main || echo "Есть изменения"
```

## FETCH_HEAD

После каждого fetch Git обновляет специальную ссылку `FETCH_HEAD`:

```bash
# Посмотреть FETCH_HEAD
cat .git/FETCH_HEAD

# Merge того, что было получено
git merge FETCH_HEAD
```

## Автоматизация fetch

### Git GUI клиенты

Большинство GUI клиентов (GitKraken, SourceTree, VS Code) автоматически делают fetch в фоне.

### Скрипт для периодического fetch

```bash
# В cron или как фоновый процесс
*/15 * * * * cd /path/to/repo && git fetch --all --prune
```

### Алиас для fetch + status

```bash
# Добавить в .gitconfig
git config --global alias.sync '!git fetch --all --prune && git status'

# Использование
git sync
```

## Best Practices

### 1. Делайте fetch регулярно

```bash
# Начинайте день с fetch
git fetch --all
```

### 2. Используйте `--prune` для чистоты

```bash
# Настройте prune по умолчанию
git config --global fetch.prune true
```

### 3. Fetch перед важными операциями

```bash
# Перед merge, rebase, создание ветки
git fetch origin
git checkout -b new-feature origin/main
```

### 4. Предпочитайте fetch + merge/rebase вместо pull

Это даёт больше контроля:
```bash
git fetch origin
git log HEAD..origin/main  # Посмотреть изменения
git merge origin/main      # Осознанный merge
```

### 5. Fetch --all при работе с несколькими remotes

```bash
git fetch --all --prune
```

## Типичные ошибки

### 1. Путаница между локальной веткой и remote-tracking

```bash
# НЕПРАВИЛЬНО: это локальная ветка main
git log main

# ПРАВИЛЬНО: это состояние main на remote
git log origin/main
```

### 2. Забыли сделать fetch перед сравнением

```bash
# НЕПРАВИЛЬНО: origin/main может быть устаревшим
git diff main origin/main

# ПРАВИЛЬНО: сначала обновить информацию
git fetch origin
git diff main origin/main
```

### 3. Ожидание изменений в рабочей директории

После fetch файлы не изменятся — нужно сделать merge или rebase:

```bash
git fetch origin       # Файлы не изменились
git merge origin/main  # Теперь файлы обновлены
```

### 4. Fetch не той ветки

```bash
# Fetch всего remote, а не только default branch
git fetch origin           # Все ветки
git fetch origin feature   # Конкретная ветка
```

## Полезные команды (шпаргалка)

```bash
# Базовый fetch
git fetch origin                    # Из origin
git fetch --all                     # Из всех remotes
git fetch origin main               # Конкретную ветку

# С очисткой
git fetch --prune origin            # Удалить устаревшие ссылки
git fetch -p --all                  # Из всех remotes с очисткой

# Просмотр после fetch
git branch -r                       # Remote-tracking ветки
git log HEAD..origin/main           # Новые коммиты
git diff HEAD origin/main           # Изменения

# Интеграция
git merge origin/main               # Merge
git rebase origin/main              # Rebase
git reset --hard origin/main        # Сброс (осторожно!)

# Shallow fetch
git fetch --depth=1 origin          # Только последний коммит
git fetch --unshallow               # Получить полную историю
```

## Заключение

`git fetch` — это безопасный способ получить информацию об изменениях в удалённом репозитории. Используйте его для:

1. **Просмотра изменений** перед их интеграцией
2. **Синхронизации информации** о ветках и тегах
3. **Осознанного merge/rebase** с полным контролем над процессом
4. **CI/CD скриптов** где нужно только получить данные

В отличие от `git pull`, fetch никогда не изменит вашу текущую работу, поэтому его можно выполнять в любой момент без риска.

---
[prev: 02-push-pull](./02-push-pull.md) | [next: 05-merge-strategies](../05-merge-strategies.md)