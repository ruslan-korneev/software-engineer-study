# Push and Pull

## Введение

**Push** и **Pull** — это две основные операции для синхронизации локального репозитория с удалённым:

- **Push** — отправка локальных коммитов на удалённый репозиторий
- **Pull** — получение и интеграция изменений с удалённого репозитория

## Git Push

### Базовый синтаксис

```bash
git push <remote> <branch>
```

### Примеры использования

```bash
# Отправить текущую ветку на origin
git push origin main

# Отправить ветку feature на origin
git push origin feature

# Сокращённая форма (если upstream настроен)
git push
```

### Первый push новой ветки

При первом push новой ветки нужно установить upstream (отслеживаемую ветку):

```bash
# Создаём ветку и переключаемся на неё
git checkout -b feature/new-login

# Первый push с установкой upstream
git push -u origin feature/new-login
# или
git push --set-upstream origin feature/new-login
```

После этого можно использовать просто `git push` без указания remote и ветки.

### Push всех веток

```bash
# Отправить все локальные ветки
git push --all origin

# Отправить все теги
git push --tags origin
```

### Force push (принудительная отправка)

**Опасная операция!** Перезаписывает историю на remote.

```bash
# Force push (НЕ рекомендуется для публичных веток)
git push --force origin main
# или
git push -f origin main

# Более безопасный вариант: не перезапишет, если кто-то уже добавил коммиты
git push --force-with-lease origin main
```

**Когда нужен force push:**
- После `git rebase`
- После `git commit --amend`
- После `git reset`

**Когда НЕЛЬЗЯ использовать force push:**
- На общих ветках (main, develop)
- Когда другие разработчики уже получили ваши коммиты

### Удаление удалённой ветки

```bash
# Удалить ветку на remote
git push origin --delete feature/old-branch
# или
git push origin :feature/old-branch
```

## Git Pull

### Базовый синтаксис

```bash
git pull <remote> <branch>
```

### Что делает pull?

`git pull` — это комбинация двух команд:

```bash
git pull origin main
# Эквивалентно:
git fetch origin
git merge origin/main
```

### Примеры использования

```bash
# Получить изменения из main
git pull origin main

# Сокращённая форма (если upstream настроен)
git pull

# Pull с указанием стратегии merge
git pull --no-rebase origin main  # Явно использовать merge
```

### Pull с rebase

Вместо merge можно использовать rebase для более чистой истории:

```bash
# Pull с rebase вместо merge
git pull --rebase origin main

# Или настроить как поведение по умолчанию
git config --global pull.rebase true
```

**Разница между merge и rebase при pull:**

```
# После git pull (merge):
* 7890abc (HEAD -> main) Merge branch 'main' of origin
|\
| * 5678def Коммит от коллеги
* | 1234abc Мой локальный коммит
|/
* 0000aaa Общий предок

# После git pull --rebase:
* 1234abc (HEAD -> main) Мой локальный коммит
* 5678def Коммит от коллеги
* 0000aaa Общий предок
```

### Pull со стратегиями разрешения конфликтов

```bash
# При конфликте автоматически выбрать свою версию
git pull -X ours origin main

# При конфликте автоматически выбрать их версию
git pull -X theirs origin main
```

## Tracking Branches (отслеживаемые ветки)

### Просмотр отслеживания

```bash
# Показать все ветки с информацией об отслеживании
git branch -vv
```

Пример вывода:
```
* main       a1b2c3d [origin/main] Last commit message
  feature    d4e5f6g [origin/feature: ahead 2, behind 1] Another commit
  local-only h7i8j9k No tracking branch
```

### Настройка отслеживания

```bash
# Установить upstream для существующей ветки
git branch --set-upstream-to=origin/main main
# или
git branch -u origin/main

# Удалить upstream
git branch --unset-upstream
```

## Типичные workflow

### Сценарий 1: Обычная работа

```bash
# 1. Получить последние изменения перед работой
git pull origin main

# 2. Сделать свои изменения
# ... редактирование файлов ...

# 3. Закоммитить
git add .
git commit -m "Добавлена новая функция"

# 4. Получить изменения, которые могли появиться пока вы работали
git pull origin main

# 5. Если есть конфликты — разрешить их

# 6. Отправить свои изменения
git push origin main
```

### Сценарий 2: Работа с feature branch

```bash
# 1. Обновить main
git checkout main
git pull origin main

# 2. Создать feature branch
git checkout -b feature/user-auth

# 3. Работать над функцией
# ... коммиты ...

# 4. Перед созданием PR — обновиться из main
git checkout main
git pull origin main
git checkout feature/user-auth
git rebase main

# 5. Push feature branch
git push -u origin feature/user-auth
```

### Сценарий 3: Синхронизация форка

```bash
# 1. Получить изменения из оригинального репозитория
git fetch upstream

# 2. Переключиться на main
git checkout main

# 3. Слить изменения из upstream
git merge upstream/main

# 4. Отправить обновлённый main в свой форк
git push origin main
```

## Обработка конфликтов при pull

### Когда возникают конфликты

Конфликты при pull возникают, когда:
- Вы и кто-то другой изменили одни и те же строки
- Один разработчик удалил файл, другой его изменил

### Разрешение конфликтов

```bash
# 1. Pull обнаруживает конфликт
git pull origin main
# Auto-merging file.txt
# CONFLICT (content): Merge conflict in file.txt
# Automatic merge failed; fix conflicts and then commit the result.

# 2. Посмотреть файлы с конфликтами
git status

# 3. Открыть файл и разрешить конфликт
# Конфликт выглядит так:
<<<<<<< HEAD
Ваш код
=======
Код из remote
>>>>>>> origin/main

# 4. После редактирования — добавить и закоммитить
git add file.txt
git commit -m "Resolve merge conflict in file.txt"
```

### Отмена pull

```bash
# Если pull ещё не завершён (конфликт)
git merge --abort

# Если pull с rebase
git rebase --abort
```

## Best Practices

### 1. Pull перед push

```bash
# Всегда получайте изменения перед отправкой
git pull origin main
git push origin main
```

### 2. Используйте `--rebase` для чистой истории

```bash
# Настройте как поведение по умолчанию
git config --global pull.rebase true
```

### 3. Используйте `--force-with-lease` вместо `--force`

```bash
# Безопаснее, чем просто --force
git push --force-with-lease origin feature
```

### 4. Не делайте force push в общие ветки

```bash
# ПЛОХО: Никогда не делайте так с main/develop
git push -f origin main  # ОПАСНО!

# ХОРОШО: Force push только в личные feature-ветки
git push -f origin feature/my-personal-branch
```

### 5. Регулярно синхронизируйтесь

Не накапливайте много локальных коммитов — чаще делайте push и pull.

### 6. Проверяйте статус перед push

```bash
git status
git log origin/main..HEAD  # Показать коммиты, которые будут отправлены
```

## Типичные ошибки

### 1. Rejected push

```
! [rejected]        main -> main (fetch first)
error: failed to push some refs to 'origin'
```

**Причина:** На remote есть коммиты, которых нет у вас.

**Решение:**
```bash
git pull origin main
# Разрешите конфликты, если есть
git push origin main
```

### 2. Non-fast-forward push

```
! [rejected]        main -> main (non-fast-forward)
```

**Причина:** Ваша история разошлась с remote (обычно после rebase или amend).

**Решение:**
```bash
# Если это ваша личная ветка
git push --force-with-lease origin main

# Если это общая ветка — НЕ используйте force, сделайте pull и merge
git pull origin main
git push origin main
```

### 3. Detached HEAD после pull

**Причина:** Вы не на ветке, а на конкретном коммите.

**Решение:**
```bash
git checkout main
git pull origin main
```

### 4. Upstream не настроен

```
fatal: The current branch feature has no upstream branch.
```

**Решение:**
```bash
git push -u origin feature
```

### 5. Authentication failed

```
fatal: Authentication failed for 'https://github.com/...'
```

**Решение:**
- Проверьте логин/пароль или токен
- Для HTTPS используйте Personal Access Token вместо пароля
- Рассмотрите переход на SSH

## Полезные команды (шпаргалка)

```bash
# Push
git push origin main               # Push ветки main
git push -u origin feature         # Push с установкой upstream
git push --all origin              # Push всех веток
git push --tags                    # Push тегов
git push --force-with-lease        # Безопасный force push
git push origin --delete branch    # Удалить удалённую ветку

# Pull
git pull origin main               # Pull с merge
git pull --rebase origin main      # Pull с rebase
git pull --no-rebase               # Явно с merge

# Отслеживание
git branch -vv                     # Показать upstream веток
git branch -u origin/main          # Установить upstream

# Диагностика
git log origin/main..HEAD          # Коммиты для push
git log HEAD..origin/main          # Коммиты для pull
```
