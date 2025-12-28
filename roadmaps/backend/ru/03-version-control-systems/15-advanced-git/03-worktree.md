# Worktree

[prev: 02-bisect](./02-bisect.md) | [next: 04-git-attributes](./04-git-attributes.md)
---

## Что такое Git Worktree

**Git worktree** позволяет иметь несколько рабочих директорий, связанных с одним репозиторием. Каждая рабочая директория (worktree) может находиться на разной ветке, что позволяет работать над несколькими задачами одновременно без переключения веток.

### Проблема, которую решает worktree

Представьте ситуацию:
- Вы работаете над большой фичей в ветке `feature/auth`
- Приходит срочный баг-репорт, нужно срочно исправить в `main`
- У вас незакоммиченные изменения и запущены процессы (dev-сервер, watch и т.д.)

**Без worktree:**
```bash
git stash
git checkout main
# Исправляем баг...
git checkout feature/auth
git stash pop
# Восстанавливаем dev-сервер...
```

**С worktree:**
```bash
# Создаем отдельную директорию для hotfix
git worktree add ../hotfix main
cd ../hotfix
# Исправляем баг, не трогая основную работу
```

## Основные команды

### Создание worktree

```bash
# Создать worktree для существующей ветки
git worktree add <путь> <ветка>
git worktree add ../feature-branch feature/new-feature

# Создать worktree с новой веткой
git worktree add -b <новая-ветка> <путь> [базовая-ветка]
git worktree add -b hotfix/urgent ../hotfix main

# Создать worktree в detached HEAD состоянии
git worktree add --detach <путь> <коммит>
git worktree add --detach ../review abc1234
```

### Просмотр worktrees

```bash
# Список всех worktrees
git worktree list

# Вывод:
# /home/user/project        abc1234 [main]
# /home/user/project-hotfix def5678 [hotfix/urgent]
# /home/user/project-feat   ghi9012 [feature/auth]

# Подробный вывод
git worktree list --porcelain
```

### Удаление worktree

```bash
# Удалить worktree (безопасно - только если нет изменений)
git worktree remove <путь>
git worktree remove ../hotfix

# Принудительное удаление (даже с изменениями)
git worktree remove --force ../hotfix

# Или вручную удалить директорию и очистить
rm -rf ../hotfix
git worktree prune
```

### Очистка и обслуживание

```bash
# Удалить записи о несуществующих worktrees
git worktree prune

# Показать что будет очищено (dry-run)
git worktree prune --dry-run

# Заблокировать worktree от удаления
git worktree lock <путь>
git worktree lock ../hotfix --reason "Важная работа в процессе"

# Разблокировать
git worktree unlock <путь>
```

## Сценарии использования

### Сценарий 1: Параллельная работа над фичами

```bash
# Основной проект
cd ~/projects/myapp

# Работаем над новой фичей
git checkout -b feature/auth

# Параллельно нужно сделать другую задачу
git worktree add ../myapp-notifications feature/notifications

# Теперь есть две директории:
# ~/projects/myapp              - feature/auth
# ~/projects/myapp-notifications - feature/notifications

# Можно открыть в разных терминалах/IDE
```

### Сценарий 2: Code Review

```bash
# Создаем worktree для ревью PR
git fetch origin pull/123/head:pr-123
git worktree add ../review-pr-123 pr-123

# Смотрим код, тестируем
cd ../review-pr-123
npm install
npm test

# После ревью удаляем
cd ../myapp
git worktree remove ../review-pr-123
git branch -D pr-123
```

### Сценарий 3: Сравнение версий

```bash
# Создать worktrees для разных версий
git worktree add ../v1.0 v1.0.0
git worktree add ../v2.0 v2.0.0

# Теперь можно сравнивать поведение side-by-side
# или запустить оба на разных портах
```

### Сценарий 4: Срочные hotfix

```bash
# Срочно нужен hotfix
git worktree add -b hotfix/critical ../hotfix main

cd ../hotfix
# Исправляем...
git add .
git commit -m "Fix critical bug"
git push origin hotfix/critical

# Возвращаемся и удаляем
cd ../myapp
git worktree remove ../hotfix
```

### Сценарий 5: Тестирование на чистой сборке

```bash
# Создаем чистый worktree для тестов
git worktree add --detach ../clean-test HEAD

cd ../clean-test
npm ci  # чистая установка
npm test
npm run build

# Удаляем после тестов
cd ../myapp
git worktree remove ../clean-test
```

## Особенности и ограничения

### Одна ветка - один worktree

```bash
# Нельзя открыть одну ветку в нескольких worktrees
git worktree add ../second-main main
# fatal: 'main' is already checked out at '/home/user/project'

# Исключение - detached HEAD
git worktree add --detach ../review main
```

### Общий .git

```bash
# Все worktrees используют один .git репозиторий
# В дополнительных worktrees .git - это файл-ссылка

cat ../hotfix/.git
# gitdir: /home/user/project/.git/worktrees/hotfix

# Это значит:
# - Один reflog для всех
# - Общие настройки репозитория
# - Меньше дискового пространства
```

### Что можно делать в worktree

- Коммитить изменения
- Создавать ветки
- Делать merge и rebase
- Push и fetch
- Все обычные git операции

### Что нельзя/осторожно

```bash
# Нельзя удалить ветку, которая открыта в worktree
git branch -d feature/auth
# error: Cannot delete branch 'feature/auth' checked out at '...'

# Нельзя переключить worktree на ветку другого worktree
cd ../hotfix
git checkout feature/auth
# fatal: 'feature/auth' is already checked out at '...'
```

## Best Practices

### Организация директорий

```bash
# Рекомендуемая структура
~/projects/
├── myapp/              # основной worktree (main)
├── myapp-feature/      # feature ветка
├── myapp-hotfix/       # hotfix
└── myapp-review/       # для code review

# Или с общим префиксом
~/projects/myapp/
├── main/              # основной worktree
├── feature-auth/      # feature ветка
└── hotfix-bug123/     # hotfix
```

### Именование

```bash
# Хорошо - понятно что это за worktree
git worktree add ../myapp-feature-auth feature/auth
git worktree add ../myapp-hotfix-login hotfix/login-fix

# Плохо - непонятно
git worktree add ../wt1 feature/auth
git worktree add ../temp hotfix/login-fix
```

### Автоматизация

```bash
# Скрипт для быстрого создания worktree для PR
#!/bin/bash
# review-pr.sh

PR_NUM=$1
git fetch origin pull/${PR_NUM}/head:pr-${PR_NUM}
git worktree add ../review-pr-${PR_NUM} pr-${PR_NUM}
cd ../review-pr-${PR_NUM}
npm install
code .
```

### Очистка

```bash
# Регулярно проверяйте список worktrees
git worktree list

# Удаляйте ненужные
git worktree remove ../old-feature

# Очищайте "мертвые" записи
git worktree prune
```

## Интеграция с IDE

### VS Code

```bash
# Открыть worktree в новом окне
code ../hotfix

# Или добавить в текущий workspace
# File -> Add Folder to Workspace...
```

### JetBrains IDEs

```bash
# Открыть как отдельный проект
# File -> Open... -> выбрать директорию worktree
```

## Сравнение с альтернативами

### Worktree vs git stash

| Аспект | Worktree | Stash |
|--------|----------|-------|
| Параллельная работа | Да | Нет |
| Простота | Средняя | Высокая |
| Сохранение контекста | Полный | Только изменения |
| Дисковое пространство | Больше | Минимально |

### Worktree vs клонирование

| Аспект | Worktree | Clone |
|--------|----------|-------|
| Дисковое пространство | Меньше | Больше |
| Общие объекты | Да | Нет |
| Синхронизация | Автоматическая | Ручная |
| Независимость | Связаны | Полная |

## Полезные алиасы

```bash
# Быстрый список worktrees
git config --global alias.wl "worktree list"

# Создание worktree с новой веткой
git config --global alias.wa "worktree add"

# Удаление worktree
git config --global alias.wr "worktree remove"

# Очистка
git config --global alias.wp "worktree prune --dry-run"
```

## Заключение

Git worktree - мощный инструмент для разработчиков, которые часто переключаются между задачами. Вместо постоянного stash/checkout, вы можете держать несколько веток открытыми одновременно. Особенно полезен для:
- Code review
- Срочных hotfix
- Параллельной разработки фич
- Тестирования разных версий

Главное помнить про ограничение "одна ветка - один worktree" и регулярно очищать неиспользуемые worktrees.

---
[prev: 02-bisect](./02-bisect.md) | [next: 04-git-attributes](./04-git-attributes.md)