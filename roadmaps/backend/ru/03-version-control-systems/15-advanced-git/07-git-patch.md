# Git Patch

[prev: 06-submodules](./06-submodules.md) | [next: 01-what-are-relational-databases](../../04-postgresql-dba/01-introduction/01-what-are-relational-databases.md)
---

## Что такое патчи

**Патч (patch)** - это текстовый файл, содержащий описание изменений между двумя версиями файлов. В Git патчи используются для:
- Обмена изменениями по email
- Применения изменений без прямого доступа к репозиторию
- Ревью кода вне GitHub/GitLab
- Переноса изменений между несвязанными репозиториями

### Формат патча

```diff
From abc1234567890 Mon Sep 17 00:00:00 2001
From: Developer <dev@example.com>
Date: Mon, 25 Dec 2023 10:00:00 +0000
Subject: [PATCH] Add user authentication

---
 src/auth.py | 15 +++++++++++++++
 1 file changed, 15 insertions(+)

diff --git a/src/auth.py b/src/auth.py
index abc1234..def5678 100644
--- a/src/auth.py
+++ b/src/auth.py
@@ -1,5 +1,20 @@
 def login(username, password):
-    pass
+    if validate(username, password):
+        return create_session(username)
+    return None
--
2.34.1
```

## Создание патчей

### git format-patch

Создает патчи из коммитов в формате, готовом для отправки по email.

```bash
# Создать патч для последнего коммита
git format-patch -1

# Вывод: 0001-Add-user-authentication.patch

# Создать патчи для последних N коммитов
git format-patch -3

# Создать патчи для диапазона коммитов
git format-patch abc1234..def5678

# Создать патчи для всех коммитов в ветке относительно main
git format-patch main
git format-patch main..feature/auth

# Создать патчи от начала ветки
git format-patch origin/main
```

### Опции format-patch

```bash
# Сохранить в определенную директорию
git format-patch -o patches/ -3

# Один файл со всеми патчами
git format-patch --stdout -3 > all-changes.patch

# Добавить cover letter (письмо с описанием серии патчей)
git format-patch --cover-letter -3

# Нумерация с определенного числа
git format-patch --start-number 5 -3

# Добавить версию (для ревизий)
git format-patch -v2 -3
# Создаст: v2-0001-First-commit.patch

# Без номера версии патча
git format-patch -N -3
```

### git diff

Создает простой diff без метаданных коммита.

```bash
# Diff между коммитами
git diff abc1234 def5678 > changes.patch

# Diff рабочей директории
git diff > unstaged.patch

# Diff staged изменений
git diff --cached > staged.patch

# Diff с бинарными файлами
git diff --binary > with-binary.patch

# Diff конкретного файла
git diff -- src/auth.py > auth-changes.patch

# Сравнение веток
git diff main..feature > feature-changes.patch
```

## Применение патчей

### git apply

Применяет изменения из патча без создания коммита.

```bash
# Применить патч
git apply changes.patch

# Проверить патч без применения (dry-run)
git apply --check changes.patch

# Показать статистику
git apply --stat changes.patch

# Применить в обратном направлении (откатить)
git apply -R changes.patch
git apply --reverse changes.patch

# Игнорировать пробелы
git apply --ignore-whitespace changes.patch

# Применить частично (интерактивно)
git apply --3way changes.patch
```

### git am (apply mailbox)

Применяет патчи, созданные `format-patch`, с сохранением информации о коммите.

```bash
# Применить один патч
git am 0001-Add-feature.patch

# Применить несколько патчей
git am *.patch

# Применить все патчи из директории
git am patches/*.patch

# Применить из stdin
git am < patches.mbox

# С подписью (Signed-off-by)
git am --signoff 0001-Add-feature.patch
```

### Обработка конфликтов в git am

```bash
# При конфликте git am останавливается
# Разрешаем конфликты вручную, затем:

# Добавить исправленные файлы
git add resolved-file.py

# Продолжить применение
git am --continue

# Пропустить проблемный патч
git am --skip

# Отменить всю операцию
git am --abort

# Применить с 3-way merge (помогает при конфликтах)
git am --3way 0001-Add-feature.patch
```

## Email Workflow

### Настройка Git для отправки email

```bash
# Настроить SMTP
git config --global sendemail.smtpserver smtp.gmail.com
git config --global sendemail.smtpserverport 587
git config --global sendemail.smtpencryption tls
git config --global sendemail.smtpuser your-email@gmail.com
# Пароль будет запрошен при отправке

# Для Gmail нужен App Password
# https://myaccount.google.com/apppasswords
```

### Отправка патчей

```bash
# Отправить патчи по email
git send-email --to maintainer@example.com \
    --cc dev-list@example.com \
    0001-Add-feature.patch

# Отправить серию патчей
git send-email --to maintainer@example.com \
    --cover-letter \
    patches/

# Сухой запуск (показать что будет отправлено)
git send-email --dry-run 0001-Add-feature.patch

# Отправить в формате in-reply-to (для серии патчей)
git send-email --in-reply-to=<message-id> patches/
```

### Создание cover letter

```bash
# Создать патчи с cover letter
git format-patch --cover-letter -3

# Редактировать 0000-cover-letter.patch
# Заменить *** SUBJECT HERE *** и *** BLURB HERE ***
```

Пример cover letter:
```
From: Developer <dev@example.com>
Subject: [PATCH 0/3] Add user authentication system

This patch series adds a complete user authentication system:
- Patch 1: Add basic login/logout functions
- Patch 2: Add session management
- Patch 3: Add password hashing

Tested on Python 3.9 and 3.10.

Developer (3):
  Add basic login/logout functions
  Add session management
  Add password hashing

 src/auth.py    | 50 ++++++++++++++++++++++++++++++++++++++++++++
 src/session.py | 30 +++++++++++++++++++++++++++
 2 files changed, 80 insertions(+)
 create mode 100644 src/auth.py
 create mode 100644 src/session.py
```

## Практические примеры

### Пример 1: Перенос коммита в другой репозиторий

```bash
# В исходном репозитории
cd original-repo
git format-patch -1 abc1234
# Создан: 0001-Add-feature.patch

# В целевом репозитории
cd ../target-repo
git am ../original-repo/0001-Add-feature.patch
```

### Пример 2: Ревью патча перед применением

```bash
# Получили патч от коллеги
# Сначала смотрим что в нем

# Статистика изменений
git apply --stat feature.patch

# Проверка на ошибки
git apply --check feature.patch

# Если ок - применяем
git apply feature.patch

# Проверяем изменения
git diff

# Если все хорошо - коммитим
git add .
git commit -m "Apply feature patch from colleague"
```

### Пример 3: Создание патча для bug report

```bash
# Исправили баг, хотим отправить maintainer'у

# Создаем патч с описанием
git format-patch -1 --cover-letter

# Редактируем cover letter с описанием бага
vim 0000-cover-letter.patch

# Отправляем
git send-email --to bugs@project.org 0000-cover-letter.patch 0001-*.patch
```

### Пример 4: Экспорт и импорт набора изменений

```bash
# Экспортировать все коммиты feature ветки
git checkout feature/new-ui
git format-patch main --stdout > new-ui-feature.patch

# Импортировать в другом месте
git checkout -b new-ui-import
git am new-ui-feature.patch
```

### Пример 5: Частичное применение патча

```bash
# Патч затрагивает несколько файлов, но нужен только один

# Создаем патч
git diff abc1234 def5678 -- src/specific-file.py > partial.patch

# Или применяем только часть существующего патча
git apply --include='src/specific-file.py' full.patch
git apply --exclude='tests/*' full.patch
```

## Работа с бинарными файлами

### Создание патча с бинарными файлами

```bash
# Обычный diff не включает бинарные файлы
git diff > text-only.patch

# Включить бинарные файлы
git diff --binary > with-binary.patch

# format-patch всегда включает бинарные файлы
git format-patch -1
```

### Применение

```bash
# Для бинарных файлов нужен --binary или format-patch
git apply --binary with-binary.patch
```

## Полезные опции и трюки

### Создание патча из stash

```bash
# Показать stash как патч
git stash show -p > stash.patch

# Или конкретный stash
git stash show -p stash@{2} > stash-2.patch
```

### Патч между произвольными состояниями

```bash
# Между тегами
git diff v1.0.0 v1.1.0 > release-changes.patch

# Между ветками
git diff main feature/auth > feature-auth.patch

# Между коммитом и веткой
git diff abc1234 main > to-main.patch
```

### Интерактивное создание патча

```bash
# Выбрать какие изменения включить
git diff > all.patch
git add -p
git diff --cached > selected.patch
git reset HEAD
```

## Best Practices

### Рекомендации по созданию патчей

1. **Атомарные коммиты** - один патч = одно логическое изменение
2. **Хорошие сообщения коммитов** - они станут частью патча
3. **Тестирование** - убедитесь, что патч применяется чисто
4. **Документация** - добавляйте cover letter для серии патчей

### Рекомендации по применению

1. **Проверяйте перед применением**
   ```bash
   git apply --check patch.patch
   ```

2. **Создавайте ветку для тестирования**
   ```bash
   git checkout -b test-patch
   git am patch.patch
   ```

3. **Используйте --3way для сложных случаев**
   ```bash
   git am --3way patch.patch
   ```

### Полезные алиасы

```bash
# Создать патч для последнего коммита
git config --global alias.patch "format-patch -1"

# Применить патч с проверкой
git config --global alias.ap "apply --check"

# Создать патчи для ветки
git config --global alias.patches "format-patch origin/main"
```

## Сравнение способов обмена изменениями

| Метод | Когда использовать |
|-------|-------------------|
| Pull Request | Стандартный workflow в GitHub/GitLab |
| git format-patch + email | Open source проекты (Linux kernel) |
| git diff | Быстрый обмен небольшими изменениями |
| Bundle | Оффлайн передача полной истории |

## Заключение

Git патчи - классический способ обмена изменениями, который по-прежнему актуален:
- **Linux kernel** и многие крупные проекты используют email workflow
- Патчи позволяют обмениваться кодом без доступа к репозиторию
- Это хороший способ понять как работает Git "под капотом"

Основные команды:
- `git format-patch` - создание патчей из коммитов
- `git diff` - создание простых патчей
- `git apply` - применение патчей без коммита
- `git am` - применение патчей с созданием коммитов
- `git send-email` - отправка патчей по email

---
[prev: 06-submodules](./06-submodules.md) | [next: 01-what-are-relational-databases](../../04-postgresql-dba/01-introduction/01-what-are-relational-databases.md)