# Submodules

[prev: 05-git-lfs](./05-git-lfs.md) | [next: 07-git-patch](./07-git-patch.md)
---

## Что такое подмодули

**Git Submodules** - это механизм включения одного Git-репозитория внутрь другого как поддиректории. Подмодуль сохраняет свою историю отдельно и указывает на конкретный коммит внешнего репозитория.

### Когда использовать подмодули

- **Общие библиотеки** - код, который используется в нескольких проектах
- **Зависимости от внешних проектов** - включение сторонних репозиториев
- **Микросервисы** - организация связанных сервисов
- **Документация** - хранение документации отдельно от кода

### Структура подмодуля

```
my-project/
├── .git/
├── .gitmodules              # Конфигурация подмодулей
├── src/
└── libs/
    └── shared-lib/          # Подмодуль
        ├── .git             # Файл-ссылка на .git/modules/shared-lib
        └── ...
```

## Добавление подмодуля

### git submodule add

```bash
# Добавить подмодуль
git submodule add <url> [путь]

# Примеры
git submodule add https://github.com/user/shared-lib.git libs/shared-lib
git submodule add git@github.com:company/design-system.git packages/design

# Добавить конкретную ветку
git submodule add -b develop https://github.com/user/lib.git libs/lib
```

### Что создается

```bash
# 1. Директория с содержимым репозитория
ls libs/shared-lib/

# 2. Файл .gitmodules
cat .gitmodules
# [submodule "libs/shared-lib"]
#     path = libs/shared-lib
#     url = https://github.com/user/shared-lib.git

# 3. Запись в .git/config
git config --list | grep submodule
# submodule.libs/shared-lib.url=https://github.com/user/shared-lib.git
# submodule.libs/shared-lib.active=true

# 4. Специальная запись в индексе (gitlink)
git ls-tree HEAD libs/
# 160000 commit abc1234... shared-lib
```

### Коммит изменений

```bash
# Закоммитить добавление подмодуля
git add .gitmodules libs/shared-lib
git commit -m "Add shared-lib as submodule"
```

## Клонирование с подмодулями

### Стандартное клонирование

```bash
# Клонирование НЕ скачивает содержимое подмодулей
git clone https://github.com/user/my-project.git
cd my-project

# Подмодули пустые!
ls libs/shared-lib/
# (пусто)

# Инициализировать и скачать подмодули
git submodule init
git submodule update

# Или одной командой
git submodule update --init
```

### Рекурсивное клонирование

```bash
# Клонировать сразу с подмодулями (рекомендуется)
git clone --recurse-submodules https://github.com/user/my-project.git

# Или короче
git clone --recursive https://github.com/user/my-project.git
```

### Для вложенных подмодулей

```bash
# Если подмодули содержат свои подмодули
git clone --recurse-submodules --recursive https://github.com/user/my-project.git

# Или после клонирования
git submodule update --init --recursive
```

## Обновление подмодулей

### Обновление до последнего коммита

```bash
# Перейти в подмодуль и обновить
cd libs/shared-lib
git fetch
git checkout main
git pull

# Вернуться и закоммитить новую версию
cd ../..
git add libs/shared-lib
git commit -m "Update shared-lib to latest version"
```

### Обновление всех подмодулей

```bash
# Обновить все подмодули до последнего коммита их веток
git submodule update --remote

# Обновить конкретный подмодуль
git submodule update --remote libs/shared-lib

# Merge вместо checkout
git submodule update --remote --merge

# Rebase вместо checkout
git submodule update --remote --rebase
```

### Обновление до зафиксированной версии

```bash
# После pull основного репозитория
git pull

# Обновить подмодули до версий из коммита
git submodule update

# Рекурсивно
git submodule update --recursive
```

### Настройка ветки для подмодуля

```bash
# Указать ветку для отслеживания
git config -f .gitmodules submodule.libs/shared-lib.branch develop

# Теперь update --remote будет брать из develop
git submodule update --remote libs/shared-lib
```

## Удаление подмодуля

### Полное удаление

```bash
# 1. Удалить секцию из .gitmodules
git config -f .gitmodules --remove-section submodule.libs/shared-lib

# 2. Удалить секцию из .git/config
git config --remove-section submodule.libs/shared-lib

# 3. Удалить из индекса
git rm --cached libs/shared-lib

# 4. Удалить метаданные
rm -rf .git/modules/libs/shared-lib

# 5. Удалить директорию
rm -rf libs/shared-lib

# 6. Закоммитить
git add .gitmodules
git commit -m "Remove shared-lib submodule"
```

### Упрощенный способ (Git 2.25+)

```bash
# Одной командой (Git 1.8.5+, но не удаляет из .git/modules)
git rm libs/shared-lib
git commit -m "Remove shared-lib submodule"

# Удалить кеш вручную (опционально)
rm -rf .git/modules/libs/shared-lib
```

## Работа с подмодулями

### Просмотр статуса

```bash
# Статус подмодулей
git submodule status

# Вывод:
# +abc1234 libs/shared-lib (v1.2.3)
# -def5678 libs/other-lib
#  ghi9012 libs/utils

# Символы:
# + подмодуль на другом коммите, чем записано
# - подмодуль не инициализирован
# U подмодуль имеет merge конфликты
```

### Выполнение команд во всех подмодулях

```bash
# Выполнить команду во всех подмодулях
git submodule foreach 'git status'

# Примеры
git submodule foreach 'git pull origin main'
git submodule foreach 'git checkout develop'
git submodule foreach 'npm install'

# С вложенными подмодулями
git submodule foreach --recursive 'git fetch'
```

### Изменения в подмодуле

```bash
# Работа в подмодуле
cd libs/shared-lib
git checkout -b feature/new-feature
# Вносим изменения...
git add .
git commit -m "Add feature"
git push origin feature/new-feature

# Вернуться и обновить ссылку
cd ../..
git add libs/shared-lib
git commit -m "Update shared-lib with new feature"
```

## Альтернативы подмодулям

### Git Subtree

```bash
# Добавить subtree
git subtree add --prefix=libs/shared-lib \
    https://github.com/user/shared-lib.git main --squash

# Обновить
git subtree pull --prefix=libs/shared-lib \
    https://github.com/user/shared-lib.git main --squash

# Отправить изменения обратно
git subtree push --prefix=libs/shared-lib \
    https://github.com/user/shared-lib.git main
```

### Сравнение Submodule vs Subtree

| Аспект | Submodule | Subtree |
|--------|-----------|---------|
| История | Отдельная | Встроенная |
| Клонирование | Требует --recursive | Автоматическое |
| Обновление | Явное | Явное |
| Сложность | Выше | Ниже |
| Отправка изменений | Просто | Сложнее |
| Размер репозитория | Меньше | Больше |

### Monorepo

Альтернативный подход - хранить все в одном репозитории:

```
monorepo/
├── packages/
│   ├── shared-lib/
│   ├── service-a/
│   └── service-b/
├── apps/
│   ├── web/
│   └── mobile/
└── package.json
```

Инструменты для monorepo:
- **Lerna** - для JavaScript/TypeScript
- **Nx** - универсальный
- **Turborepo** - для JS/TS с кешированием
- **Bazel** - от Google, для больших проектов

## Типичные проблемы и решения

### Detached HEAD в подмодуле

```bash
# При checkout подмодуль оказывается в detached HEAD
cd libs/shared-lib
git status
# HEAD detached at abc1234

# Решение: переключиться на ветку
git checkout main
```

### Подмодуль показывает изменения при git status

```bash
# Игнорировать изменения в подмодуле
git config diff.ignoreSubmodules dirty

# Или в .gitmodules
[submodule "libs/shared-lib"]
    path = libs/shared-lib
    url = https://github.com/user/shared-lib.git
    ignore = dirty
```

### Конфликты при merge с подмодулями

```bash
# При конфликте в подмодуле
git status
# both modified: libs/shared-lib

# Выбрать версию
git checkout --ours libs/shared-lib   # наша версия
git checkout --theirs libs/shared-lib # их версия

# Или разрешить вручную
cd libs/shared-lib
git merge origin/main
cd ..
git add libs/shared-lib
```

### Подмодуль не скачивается

```bash
# Синхронизировать URL подмодуля
git submodule sync

# Переинициализировать
git submodule deinit libs/shared-lib
git submodule update --init libs/shared-lib
```

### Забыли обновить подмодуль после pull

```bash
# Автоматическое обновление при pull
git config --global submodule.recurse true

# Или при каждом pull
git pull --recurse-submodules
```

## Best Practices

### Рекомендации

1. **Используйте --recursive при клонировании**
   ```bash
   git clone --recursive https://github.com/user/project.git
   ```

2. **Документируйте использование подмодулей**
   ```markdown
   # README.md
   ## Подмодули
   После клонирования выполните:
   git submodule update --init --recursive
   ```

3. **Настройте автоматическое обновление**
   ```bash
   git config --global submodule.recurse true
   ```

4. **Фиксируйте конкретные версии**
   - Не используйте плавающие ссылки на ветки в продакшене
   - Обновляйте осознанно и тестируйте

5. **Проверяйте статус перед коммитом**
   ```bash
   git submodule status
   git status
   ```

### Автоматизация

```bash
# Скрипт для настройки после клонирования
# setup.sh
#!/bin/bash
git submodule update --init --recursive
git config submodule.recurse true
```

### Полезные алиасы

```bash
# Обновить все подмодули до последней версии
git config --global alias.sup "submodule update --remote --merge"

# Статус подмодулей
git config --global alias.ss "submodule status"

# Выполнить команду во всех подмодулях
git config --global alias.se "submodule foreach"

# Инициализировать все
git config --global alias.si "submodule update --init --recursive"
```

## Заключение

Git Submodules - мощный, но сложный инструмент. Основные правила:
- Всегда клонируйте с `--recursive`
- Обновляйте подмодули после `git pull`
- Документируйте использование подмодулей в README
- Рассмотрите альтернативы (subtree, monorepo) для простых случаев

Подмодули хорошо подходят для:
- Больших проектов с четким разделением компонентов
- Когда нужно отслеживать конкретные версии зависимостей
- Когда внешний репозиторий активно развивается отдельной командой

---
[prev: 05-git-lfs](./05-git-lfs.md) | [next: 07-git-patch](./07-git-patch.md)