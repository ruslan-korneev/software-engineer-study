# Forking vs Cloning

[prev: 06-profile-readme](../06-github-essentials/06-profile-readme.md) | [next: 02-issues](./02-issues.md)
---

## Введение

При работе с GitHub существует два основных способа получить копию репозитория: **fork** (форк) и **clone** (клонирование). Понимание разницы между ними критически важно для эффективной работы в команде и участия в open source проектах.

---

## Что такое Clone (Клонирование)

**Clone** — это создание локальной копии репозитория на вашем компьютере. При клонировании вы получаете полную историю коммитов и все ветки.

```bash
# Клонирование репозитория
git clone https://github.com/owner/repository.git

# Клонирование с указанием имени папки
git clone https://github.com/owner/repository.git my-folder

# Клонирование конкретной ветки
git clone -b develop https://github.com/owner/repository.git

# Shallow clone (только последние N коммитов)
git clone --depth 1 https://github.com/owner/repository.git
```

### Когда использовать Clone

- Вы являетесь членом команды с правами на запись
- Работа над приватным репозиторием вашей организации
- Локальная разработка собственного проекта
- Простое изучение кода проекта

---

## Что такое Fork (Форк)

**Fork** — это создание полной копии репозитория в вашем аккаунте GitHub. Форк существует на сервере GitHub и связан с оригинальным репозиторием.

### Процесс форка

1. Нажмите кнопку "Fork" на странице репозитория
2. Выберите свой аккаунт или организацию
3. GitHub создаст копию в вашем пространстве

### Когда использовать Fork

- Участие в open source проектах
- У вас нет прав на запись в оригинальный репозиторий
- Вы хотите экспериментировать без риска для оригинала
- Создание своей версии проекта на базе существующего

---

## Сравнение Fork и Clone

| Характеристика | Clone | Fork |
|---------------|-------|------|
| Где создается | Локально на вашем компьютере | На GitHub в вашем аккаунте |
| Связь с оригиналом | origin указывает на оригинал | Независимая копия со связью |
| Права на push | Нужны права на запись | Всегда есть (это ваш репозиторий) |
| Синхронизация | Через pull/fetch | Через upstream remote |
| Видимость | Только локально | Публично на GitHub |
| Использование | Командная работа | Контрибуция в чужие проекты |

---

## Upstream и Origin

При работе с форками важно понимать два remote-а:

### Origin

**Origin** — это ваш форк репозитория на GitHub.

```bash
# После клонирования форка
git remote -v
# origin  https://github.com/YOUR-USERNAME/repository.git (fetch)
# origin  https://github.com/YOUR-USERNAME/repository.git (push)
```

### Upstream

**Upstream** — это оригинальный репозиторий, от которого вы сделали форк.

```bash
# Добавление upstream
git remote add upstream https://github.com/ORIGINAL-OWNER/repository.git

# Проверка remote-ов
git remote -v
# origin    https://github.com/YOUR-USERNAME/repository.git (fetch)
# origin    https://github.com/YOUR-USERNAME/repository.git (push)
# upstream  https://github.com/ORIGINAL-OWNER/repository.git (fetch)
# upstream  https://github.com/ORIGINAL-OWNER/repository.git (push)
```

### Схема работы

```
[Upstream Repository]  <---- Оригинальный проект
         |
         | (fork)
         v
[Origin - Your Fork]   <---- Ваша копия на GitHub
         |
         | (clone)
         v
[Local Repository]     <---- Локальная копия
```

---

## Синхронизация форка с оригиналом

Важно регулярно синхронизировать форк, чтобы не отставать от оригинального проекта.

### Способ 1: Через командную строку

```bash
# 1. Получить изменения из upstream
git fetch upstream

# 2. Переключиться на main ветку
git checkout main

# 3. Слить изменения из upstream/main
git merge upstream/main

# 4. Отправить обновления в ваш форк (origin)
git push origin main
```

### Способ 2: Rebase вместо merge

```bash
# Rebase сохраняет линейную историю
git fetch upstream
git checkout main
git rebase upstream/main
git push origin main --force-with-lease
```

### Способ 3: Через GitHub UI

1. Перейдите на страницу вашего форка
2. Нажмите "Sync fork" (если есть новые коммиты)
3. Подтвердите синхронизацию

### Автоматизация синхронизации

```bash
# Создайте alias для быстрой синхронизации
git config --global alias.sync-fork '!git fetch upstream && git checkout main && git merge upstream/main && git push origin main'

# Использование
git sync-fork
```

---

## Workflow для контрибуции в Open Source

### Полный процесс

```bash
# 1. Форкните репозиторий на GitHub (через UI)

# 2. Клонируйте ваш форк
git clone https://github.com/YOUR-USERNAME/project.git
cd project

# 3. Добавьте upstream
git remote add upstream https://github.com/ORIGINAL-OWNER/project.git

# 4. Синхронизируйте с upstream
git fetch upstream
git checkout main
git merge upstream/main

# 5. Создайте feature branch
git checkout -b feature/my-contribution

# 6. Внесите изменения и закоммитьте
git add .
git commit -m "feat: add new feature"

# 7. Отправьте ветку в ваш форк
git push origin feature/my-contribution

# 8. Создайте Pull Request на GitHub (через UI)
```

### Обновление ветки после ревью

```bash
# Если в upstream появились новые изменения
git fetch upstream
git checkout feature/my-contribution
git rebase upstream/main

# Если были конфликты, решите их и продолжите
git add .
git rebase --continue

# Отправьте обновленную ветку
git push origin feature/my-contribution --force-with-lease
```

---

## Best Practices

### Для форков

1. **Всегда добавляйте upstream** сразу после клонирования форка
2. **Регулярно синхронизируйте** форк с оригиналом
3. **Создавайте feature branches** для каждого изменения
4. **Никогда не работайте напрямую в main** вашего форка
5. **Используйте rebase** для чистой истории перед PR

### Для клонов

1. **Проверяйте права доступа** перед началом работы
2. **Используйте SSH** для удобной аутентификации
3. **Настройте upstream tracking** для веток

```bash
# Настройка SSH вместо HTTPS
git remote set-url origin git@github.com:username/repository.git
```

---

## Типичные ошибки

### Ошибка 1: Push в upstream вместо origin

```bash
# Неправильно - попытка push в чужой репозиторий
git push upstream main  # Ошибка доступа!

# Правильно - push в свой форк
git push origin main
```

### Ошибка 2: Забыли добавить upstream

```bash
# Проверьте, есть ли upstream
git remote -v

# Если нет - добавьте
git remote add upstream https://github.com/ORIGINAL/repo.git
```

### Ошибка 3: Конфликты при синхронизации

```bash
# При конфликтах во время merge
git merge upstream/main
# CONFLICT (content): Merge conflict in file.txt

# Решите конфликты вручную, затем
git add file.txt
git commit -m "Merge upstream/main, resolve conflicts"
```

---

## Полезные команды

```bash
# Просмотр всех remote-ов
git remote -v

# Информация о конкретном remote
git remote show upstream

# Переименование remote
git remote rename origin old-origin

# Удаление remote
git remote remove upstream

# Изменение URL remote
git remote set-url origin https://github.com/NEW-URL/repo.git

# Просмотр веток upstream
git branch -r | grep upstream
```

---

## Заключение

- **Clone** — для работы над проектами, где у вас есть права на запись
- **Fork** — для контрибуции в проекты, где прав на запись нет
- **Origin** — указывает на ваш репозиторий (форк или оригинал)
- **Upstream** — указывает на оригинальный репозиторий (при работе с форком)
- Регулярная синхронизация форка предотвращает конфликты при создании PR

---
[prev: 06-profile-readme](../06-github-essentials/06-profile-readme.md) | [next: 02-issues](./02-issues.md)