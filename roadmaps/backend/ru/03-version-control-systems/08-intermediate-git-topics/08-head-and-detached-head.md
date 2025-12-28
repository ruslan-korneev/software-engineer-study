# HEAD и Detached HEAD

[prev: 03-common-hooks](07-git-hooks/03-common-hooks.md) | [next: 09-reset-modes](./09-reset-modes.md)
---

## Что такое HEAD

**HEAD** — это специальный указатель (ссылка), который показывает, где вы сейчас находитесь в истории Git. Это "текущая позиция" в репозитории.

```
┌─────────────────────────────────────────────────────────────────┐
│                        Git References                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   HEAD ──► main ──► Commit C (abc123)                           │
│                           │                                      │
│                           ▼                                      │
│                     Commit B (def456)                           │
│                           │                                      │
│                           ▼                                      │
│                     Commit A (789xyz)                           │
│                                                                  │
│   HEAD указывает на ветку, ветка указывает на коммит           │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Где хранится HEAD

```bash
# HEAD - это файл в .git директории
cat .git/HEAD

# Обычный вывод (на ветке):
ref: refs/heads/main

# Detached HEAD (на конкретном коммите):
a1b2c3d4e5f6789...
```

### Просмотр HEAD

```bash
# Показать, куда указывает HEAD
git rev-parse HEAD
# a1b2c3d4e5f6789012345678901234567890abcd

# Показать короткий хеш
git rev-parse --short HEAD
# a1b2c3d

# Показать символическую ссылку
git symbolic-ref HEAD
# refs/heads/main

# Показать имя ветки
git symbolic-ref --short HEAD
# main

# Или просто
git branch --show-current
# main
```

## Символические ссылки

HEAD обычно является **символической ссылкой** (symbolic ref) — он указывает не напрямую на коммит, а на ветку.

```
┌─────────────────────────────────────────────────────────────────┐
│                   Reference Chain                               │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   .git/HEAD                                                      │
│       │                                                          │
│       ▼                                                          │
│   ref: refs/heads/main  ─────► .git/refs/heads/main             │
│                                       │                          │
│                                       ▼                          │
│                                 a1b2c3d (SHA)                   │
│                                       │                          │
│                                       ▼                          │
│                               Commit Object                      │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

```bash
# Посмотреть структуру refs
ls -la .git/refs/heads/
# main
# feature-branch
# develop

# Содержимое ветки - это SHA коммита
cat .git/refs/heads/main
# a1b2c3d4e5f6789012345678901234567890abcd
```

## HEAD~, HEAD^, HEAD~3

Git предоставляет специальный синтаксис для навигации по истории относительно HEAD.

### Синтаксис ~ (тильда)

`~` означает "предок в линейной истории" (первый родитель).

```bash
# HEAD~  = HEAD~1 = первый родитель HEAD
# HEAD~2 = второй предок HEAD (родитель родителя)
# HEAD~3 = третий предок HEAD

git show HEAD~    # Предыдущий коммит
git show HEAD~2   # Два коммита назад
git show HEAD~5   # Пять коммитов назад
```

```
     HEAD
       │
       ▼
   ┌───────┐
   │   E   │  HEAD~0 (или просто HEAD)
   └───┬───┘
       │
   ┌───▼───┐
   │   D   │  HEAD~1 (или HEAD~)
   └───┬───┘
       │
   ┌───▼───┐
   │   C   │  HEAD~2
   └───┬───┘
       │
   ┌───▼───┐
   │   B   │  HEAD~3
   └───┬───┘
       │
   ┌───▼───┐
   │   A   │  HEAD~4
   └───────┘
```

### Синтаксис ^ (каретка)

`^` используется для навигации по родителям merge коммитов.

```bash
# HEAD^  = HEAD^1 = первый родитель
# HEAD^2 = второй родитель (для merge коммитов)
# HEAD^3 = третий родитель (для octopus merge)
```

```
       HEAD (Merge commit)
          │
      ┌───▼───┐
      │   M   │
      └─┬───┬─┘
        │   │
   HEAD^1   HEAD^2
        │   │
    ┌───▼───┐   ┌───▼───┐
    │   A   │   │   B   │
    └───────┘   └───────┘
    (main)      (feature)
```

### Комбинирование ~ и ^

```bash
# Можно комбинировать
HEAD~2^2   # Бабушка по второй линии

# Пример для merge коммита:
# HEAD    = Merge commit
# HEAD^1  = Первый родитель (main)
# HEAD^2  = Второй родитель (feature)
# HEAD~1  = То же что HEAD^1
# HEAD^2~1 = Родитель второго родителя
```

```
             HEAD (Merge)
                │
            ┌───▼───┐
            │   M   │
            └─┬───┬─┘
              │   │
         HEAD^1   HEAD^2
              │   │
          ┌───▼───┐   ┌───▼───┐
          │   C   │   │   F   │  HEAD^2~0
          └───┬───┘   └───┬───┘
              │           │
          ┌───▼───┐   ┌───▼───┐
          │   B   │   │   E   │  HEAD^2~1
          └───┬───┘   └───┬───┘
              │           │
          ┌───▼───┐   ┌───▼───┐
          │   A   │   │   D   │  HEAD^2~2
          └───────┘   └───────┘
```

### Практические примеры

```bash
# Показать изменения в предыдущем коммите
git show HEAD~

# Сравнить с состоянием 3 коммита назад
git diff HEAD~3

# Вернуть файл к состоянию 2 коммита назад
git checkout HEAD~2 -- filename.txt

# Для merge коммитов - показать изменения из feature ветки
git show HEAD^2

# Интерактивный rebase последних 5 коммитов
git rebase -i HEAD~5

# Сбросить последний коммит (сохранив изменения)
git reset HEAD~
```

## Detached HEAD состояние

**Detached HEAD** (отсоединённый HEAD) — это состояние, когда HEAD указывает напрямую на коммит, а не на ветку.

```
┌─────────────────────────────────────────────────────────────────┐
│              Normal vs Detached HEAD                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                  │
│   NORMAL STATE                    DETACHED HEAD                 │
│                                                                  │
│   HEAD ──► main ──► C             HEAD ──► B (directly!)       │
│                     │                      │                    │
│                     ▼                      ▼                    │
│                     B             main ──► C                    │
│                     │                      │                    │
│                     ▼                      ▼                    │
│                     A                      B                    │
│                                            │                    │
│   .git/HEAD contains:             .git/HEAD contains:          │
│   ref: refs/heads/main            abc123def456...              │
│                                                                  │
└─────────────────────────────────────────────────────────────────┘
```

### Как распознать Detached HEAD

```bash
# Git предупреждает при входе в detached HEAD
$ git checkout abc1234
Note: switching to 'abc1234'.

You are in 'detached HEAD' state. You can look around, make experimental
changes and commit them, and you can discard any commits you make in this
state without impacting any branches by switching back to a branch.

If you want to create a new branch to retain commits you create, you may
do so (now or later) by using -c with the switch command. Example:

  git switch -c <new-branch-name>

Or undo this operation with:

  git switch -

HEAD is now at abc1234 Some commit message
```

```bash
# Проверить состояние
git status
# HEAD detached at abc1234

# git branch показывает "(HEAD detached at ...)"
git branch
# * (HEAD detached at abc1234)
#   main
#   feature

# symbolic-ref вернёт ошибку в detached state
git symbolic-ref HEAD
# fatal: ref HEAD is not a symbolic ref
```

## Как попасть в Detached HEAD

### 1. Checkout конкретного коммита

```bash
# По хешу
git checkout abc1234

# По относительной ссылке
git checkout HEAD~3
git checkout main~5
```

### 2. Checkout тега

```bash
git checkout v1.0.0
git checkout tags/release-2.0
```

### 3. Checkout удалённой ветки без создания локальной

```bash
git checkout origin/feature-branch
# Создаст detached HEAD

# Правильно - создать локальную ветку
git checkout -b feature-branch origin/feature-branch
# или
git switch -c feature-branch origin/feature-branch
```

### 4. После rebase или операций с историей

```bash
# Во время интерактивного rebase
git rebase -i HEAD~3
# Git переходит в detached HEAD на время rebase
```

### 5. Через git switch --detach

```bash
git switch --detach main~2
git switch --detach v1.0.0
```

## Как выйти из Detached HEAD

### 1. Вернуться на ветку

```bash
# Вернуться на main
git checkout main

# Или с git switch
git switch main

# Вернуться на предыдущую ветку
git checkout -
git switch -
```

### 2. Создать новую ветку

```bash
# Если хотите сохранить текущую позицию как ветку
git checkout -b new-branch-name

# Или с git switch
git switch -c new-branch-name
```

## Сохранение изменений из Detached HEAD

### Проблема: коммиты в detached HEAD могут быть потеряны

```bash
# Вы в detached HEAD и сделали коммиты
git checkout abc1234
# ... делаете изменения ...
git commit -m "Some changes"
git commit -m "More changes"

# Теперь HEAD указывает на xyz789 (новый коммит)
# Но НИКАКАЯ ветка не указывает на него!

# Если сейчас переключиться на ветку:
git checkout main
# Коммиты xyz789 "потеряны" (но не удалены сразу)
```

```
     До checkout main:          После checkout main:

     HEAD                        main, HEAD
       │                            │
       ▼                            ▼
   ┌───────┐                    ┌───────┐
   │ xyz789│ ← Новый коммит     │  def  │
   └───┬───┘                    └───┬───┘
       │                            │
   ┌───▼───┐   ┌───────┐            ▼
   │  ...  │   │  def  │ ← main   ...
   └───┬───┘   └───┬───┘
       │           │            ┌───────┐
   ┌───▼───┐       ▼            │ xyz789│ ← Потерянный коммит!
   │abc1234│     ...            └───┬───┘   (orphaned)
   └───────┘                        │
                                    ▼
                                  ...
```

### Решение 1: Создать ветку ДО переключения

```bash
# Пока ещё в detached HEAD с новыми коммитами
git branch save-my-work
# или
git checkout -b save-my-work

# Теперь безопасно переключаться
git checkout main
```

### Решение 2: Создать ветку ПОСЛЕ переключения

```bash
# Упс, уже переключились на main
# Нужно найти "потерянный" коммит

# Git сохраняет историю HEAD в reflog
git reflog
# abc1234 HEAD@{0}: checkout: moving from xyz789 to main
# xyz789 HEAD@{1}: commit: More changes
# def456 HEAD@{2}: commit: Some changes
# abc1234 HEAD@{3}: checkout: moving from main to abc1234

# Создаём ветку на потерянном коммите
git branch recovered-work xyz789
# или
git checkout -b recovered-work xyz789
```

### Решение 3: Cherry-pick потерянных коммитов

```bash
# Если нужно перенести коммиты на текущую ветку
git cherry-pick xyz789
git cherry-pick def456
```

### Время жизни orphaned коммитов

```bash
# Git не удаляет коммиты сразу
# Они остаются до сборки мусора (gc)

# По умолчанию orphaned коммиты живут 30 дней
git config gc.reflogExpireUnreachable
# 30.days

# Принудительная сборка мусора (осторожно!)
git gc --prune=now
```

## Полезные use cases для Detached HEAD

### 1. Исследование истории

```bash
# Посмотреть состояние проекта в прошлом
git checkout v1.0.0
ls -la
cat README.md

# Вернуться
git checkout main
```

### 2. Тестирование старой версии

```bash
# Запустить тесты на старой версии
git checkout HEAD~10
npm test

# Вернуться
git checkout main
```

### 3. Создание hotfix от старой версии

```bash
# Найти проблемный коммит
git bisect start
git bisect bad HEAD
git bisect good v1.0.0
# ... bisect находит проблемный коммит abc123 ...

# Создать ветку для исправления
git checkout abc123~1  # Коммит ДО проблемы
git checkout -b hotfix/critical-bug
# ... делаем исправление ...
```

### 4. Временные эксперименты

```bash
# Попробовать что-то без создания ветки
git checkout HEAD~5
# ... эксперименты ...

# Не понравилось - просто уходим
git checkout main
# Все изменения автоматически "забыты"
```

## Best Practices

### 1. Избегайте коммитов в Detached HEAD

```bash
# Если планируете делать коммиты - сразу создайте ветку
git checkout -b experiment abc1234
# вместо
git checkout abc1234
```

### 2. Используйте git switch для безопасности

```bash
# git switch не позволяет случайно попасть в detached HEAD
git switch main      # OK
git switch abc1234   # Error!

# Нужно явно указать --detach
git switch --detach abc1234
```

### 3. Проверяйте git status

```bash
# Всегда знайте, где вы находитесь
git status
# On branch main  - всё хорошо
# HEAD detached at abc1234 - осторожно!
```

### 4. Используйте reflog для восстановления

```bash
# Если потеряли коммиты - не паникуйте
git reflog  # Здесь вся история
```

## Команды для работы с HEAD

```bash
# Показать текущий коммит
git show HEAD

# Показать родителя
git show HEAD^
git show HEAD~

# Показать предка N уровней назад
git show HEAD~N

# Сравнить с прошлым состоянием
git diff HEAD~3

# Вернуть файл к прошлому состоянию
git checkout HEAD~2 -- file.txt

# Создать ветку на текущем HEAD
git branch new-branch

# Переместить HEAD (reset)
git reset --soft HEAD~    # Только HEAD
git reset --mixed HEAD~   # HEAD + staging
git reset --hard HEAD~    # HEAD + staging + working dir

# Проверить, detached ли HEAD
git symbolic-ref -q HEAD && echo "On branch" || echo "Detached HEAD"
```

---
[prev: 03-common-hooks](07-git-hooks/03-common-hooks.md) | [next: 09-reset-modes](./09-reset-modes.md)