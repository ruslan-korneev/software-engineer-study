# Switching Branches

[prev: 03-deleting-branches](./03-deleting-branches.md) | [next: 05-merging-basics](./05-merging-basics.md)
---

## Что происходит при переключении?

При переключении веток Git выполняет следующие действия:
1. **Обновляет HEAD** — указатель переходит на новую ветку
2. **Обновляет индекс (staging area)** — соответствует новой ветке
3. **Обновляет рабочую директорию** — файлы меняются на версии из новой ветки

## Основные команды переключения

### Классический способ (checkout)

```bash
# Переключиться на существующую ветку
git checkout <имя-ветки>

# Примеры
git checkout main
git checkout feature-login
git checkout develop
```

### Современный способ (switch) — Git 2.23+

```bash
# Переключиться на существующую ветку
git switch <имя-ветки>

# Примеры
git switch main
git switch feature-login
```

Команда `git switch` была введена для разделения функциональности `checkout`, которая используется и для переключения веток, и для восстановления файлов.

### Переключение с созданием ветки

```bash
# checkout
git checkout -b <новая-ветка>
git checkout -b <новая-ветка> <базовая-ветка>

# switch
git switch -c <новая-ветка>
git switch -c <новая-ветка> <базовая-ветка>

# Примеры
git checkout -b feature-api
git switch -c feature-api develop
```

## Переключение при наличии изменений

### Если изменения не конфликтуют

Git позволит переключиться, сохранив незакоммиченные изменения:

```bash
# Изменили file.txt (который одинаков в обеих ветках)
git switch other-branch
# Успех: изменения сохраняются в рабочей директории
```

### Если изменения конфликтуют

Git откажется переключаться:

```bash
# Изменили file.txt (который отличается в ветках)
git switch other-branch
# error: Your local changes to the following files would be overwritten by checkout:
#        file.txt
# Please commit your changes or stash them before you switch branches.
```

### Решения конфликтных ситуаций

**Вариант 1: Закоммитить изменения**
```bash
git add .
git commit -m "WIP: save changes before switching"
git switch other-branch
```

**Вариант 2: Спрятать изменения (stash)**
```bash
git stash
git switch other-branch
# Позже вернуть изменения
git stash pop
```

**Вариант 3: Принудительно переключиться (потеря изменений!)**
```bash
# checkout
git checkout -f other-branch

# switch
git switch -f other-branch
git switch --discard-changes other-branch
```

**Внимание:** Принудительное переключение удалит незакоммиченные изменения!

## Переключение на удалённые ветки

### Автоматическое создание tracking ветки

```bash
# Если ветки нет локально, но есть на remote
git switch feature-from-remote
# Автоматически создаст локальную ветку, отслеживающую origin/feature-from-remote

# Или явно
git switch -c feature-local --track origin/feature-remote
```

### Переключение в detached HEAD state

```bash
# Переключиться на конкретный коммит
git checkout abc1234

# Переключиться на тег
git checkout v1.0.0

# switch требует явного флага
git switch --detach abc1234
git switch --detach v1.0.0
```

В состоянии detached HEAD вы не находитесь ни на какой ветке. Создайте новую ветку, если хотите сохранить изменения.

## Быстрое переключение

### Вернуться на предыдущую ветку

```bash
# Переключиться на предыдущую ветку
git checkout -
git switch -

# Пример
git switch main
git switch develop
git switch -      # вернётся на main
git switch -      # вернётся на develop
```

### Переключение с использованием паттернов

```bash
# Если имя ветки уникально определяется префиксом
git checkout feat<TAB>    # автодополнение

# Через glob (если настроен bash completion)
git checkout 'feature-*'  # не работает напрямую
```

## Переключение и подмодули

```bash
# Переключиться и обновить подмодули
git checkout other-branch
git submodule update --init --recursive

# Или одной командой
git checkout other-branch --recurse-submodules
```

## Best Practices

### Чистый рабочий каталог

```bash
# Перед переключением проверить статус
git status

# Если есть изменения — решить что с ними делать:
# - commit: git add . && git commit -m "message"
# - stash: git stash
# - discard: git checkout -- .
```

### Использование stash для временных изменений

```bash
# Сохранить изменения
git stash push -m "WIP: feature description"

# Переключиться
git switch other-branch

# Сделать что нужно
# ...

# Вернуться и восстановить
git switch original-branch
git stash pop
```

### Проверка ветки перед переключением

```bash
# Посмотреть, что на ветке
git log --oneline other-branch -5

# Сравнить с текущей
git diff other-branch

# Посмотреть какие файлы изменятся
git diff --stat other-branch
```

## Типичные ошибки

### 1. Переключение с незакоммиченными изменениями

```bash
# ОШИБКА: забыли про изменения
git switch other-branch
# error: Your local changes would be overwritten

# РЕШЕНИЕ: stash или commit
git stash
git switch other-branch
```

### 2. Переключение в detached HEAD без понимания

```bash
# ОШИБКА: сделали коммиты в detached HEAD
git checkout abc1234
# делаем изменения...
git commit -m "Some changes"
git switch main
# Коммиты "потеряны"!

# РЕШЕНИЕ: создать ветку до переключения
git branch save-my-work
git switch main
```

### 3. Путаница checkout/switch

```bash
# checkout используется для разных целей:
git checkout branch-name    # переключить ветку
git checkout -- file.txt    # отменить изменения в файле
git checkout abc1234        # переключиться на коммит

# switch только для веток:
git switch branch-name      # переключить ветку
git restore file.txt        # отменить изменения в файле (новая команда)
git switch --detach abc1234 # переключиться на коммит
```

### 4. Забыли обновить после переключения

```bash
# ОШИБКА: переключились на устаревшую ветку
git switch develop
# develop отстаёт от origin/develop

# ПРАВИЛЬНО: обновить после переключения
git switch develop
git pull
```

## Полезные команды

```bash
# Показать текущую ветку
git branch --show-current

# Показать на каком коммите находимся
git rev-parse HEAD

# Показать историю переключений
git reflog | grep checkout

# Показать все ветки с текущей помеченной звёздочкой
git branch

# Переключиться и сбросить к remote версии
git switch main
git reset --hard origin/main
```

## Сравнение checkout и switch

| Действие | checkout | switch |
|----------|----------|--------|
| Переключить ветку | `git checkout branch` | `git switch branch` |
| Создать и переключить | `git checkout -b new` | `git switch -c new` |
| Вернуться назад | `git checkout -` | `git switch -` |
| На коммит | `git checkout abc123` | `git switch --detach abc123` |
| Принудительно | `git checkout -f branch` | `git switch -f branch` |
| Отменить изменения | `git checkout -- file` | `git restore file` |

**Рекомендация:** Для новых проектов используйте `git switch` и `git restore` — они более интуитивно понятны и разделены по функциональности.

---
[prev: 03-deleting-branches](./03-deleting-branches.md) | [next: 05-merging-basics](./05-merging-basics.md)