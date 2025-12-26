# Merging Basics

## Что такое слияние (merge)?

**Слияние (merge)** — это операция объединения изменений из одной ветки в другую. При слиянии Git анализирует историю обеих веток и создаёт новый коммит, объединяющий их изменения.

## Типы слияния

### 1. Fast-forward merge

Происходит, когда целевая ветка не имеет новых коммитов с момента создания сливаемой ветки.

```
До слияния:
main:     A---B
                \
feature:         C---D

После fast-forward merge:
main:     A---B---C---D
```

Git просто перемещает указатель main вперёд, не создавая дополнительных коммитов.

```bash
git checkout main
git merge feature
# Fast-forward
```

### 2. Three-way merge (трёхстороннее слияние)

Происходит, когда обе ветки имеют уникальные коммиты.

```
До слияния:
main:     A---B---E
                \
feature:         C---D

После three-way merge:
main:     A---B---E---M
                \     /
feature:         C---D
```

Git находит общего предка (B), сравнивает изменения в обеих ветках и создаёт merge commit (M).

```bash
git checkout main
git merge feature
# Merge made by the 'ort' strategy.
```

## Основные команды слияния

### Простое слияние

```bash
# Переключиться на целевую ветку
git checkout main

# Слить feature-ветку
git merge feature-branch

# Слить с сообщением
git merge feature-branch -m "Merge feature-branch: add user authentication"
```

### Управление типом слияния

```bash
# Только fast-forward (отменить если невозможно)
git merge --ff-only feature-branch

# Всегда создавать merge commit (даже если возможен fast-forward)
git merge --no-ff feature-branch

# Слить без автоматического коммита
git merge --no-commit feature-branch
```

### Пример --no-ff

```bash
git checkout main
git merge --no-ff feature-login -m "Merge feature-login: user authentication system"
```

Результат:
```
main:     A---B-------M
                \     /
feature:         C---D
```

Преимущество `--no-ff`: сохраняется информация о том, что функциональность разрабатывалась в отдельной ветке.

## Разрешение конфликтов

### Когда возникают конфликты

Конфликты возникают, когда:
- Одни и те же строки изменены в обеих ветках
- Файл удалён в одной ветке и изменён в другой
- Оба добавили файл с одинаковым именем, но разным содержимым

### Пример конфликта

```bash
git merge feature-branch
# Auto-merging file.txt
# CONFLICT (content): Merge conflict in file.txt
# Automatic merge failed; fix conflicts and then commit the result.
```

### Формат конфликта в файле

```
<<<<<<< HEAD
Это изменения из текущей ветки (main)
=======
Это изменения из сливаемой ветки (feature-branch)
>>>>>>> feature-branch
```

### Разрешение конфликта

**Шаг 1: Найти конфликтные файлы**
```bash
git status
# Unmerged paths:
#   both modified:   file.txt
```

**Шаг 2: Отредактировать файлы**

Выбрать нужную версию или объединить вручную:

```
# До (конфликт):
<<<<<<< HEAD
функция_v1()
=======
функция_v2()
>>>>>>> feature-branch

# После (разрешено):
функция_объединенная()
```

**Шаг 3: Отметить как разрешённый**
```bash
git add file.txt
```

**Шаг 4: Завершить слияние**
```bash
git commit -m "Merge feature-branch: resolve conflicts in file.txt"
```

### Инструменты для разрешения конфликтов

```bash
# Использовать встроенный инструмент слияния
git mergetool

# Настроить инструмент (VS Code)
git config --global merge.tool vscode
git config --global mergetool.vscode.cmd 'code --wait $MERGED'

# Другие популярные инструменты
# - vimdiff
# - meld
# - kdiff3
# - Beyond Compare
```

### Отмена слияния

```bash
# Отменить слияние до коммита
git merge --abort

# Отменить слияние после коммита
git reset --hard HEAD~1
# или
git revert -m 1 HEAD
```

## Стратегии слияния

Git поддерживает различные стратегии слияния:

### ort (по умолчанию, Git 2.33+)

```bash
git merge -s ort feature-branch
```

Улучшенная версия recursive. Быстрее и лучше обрабатывает переименования.

### recursive (старый default)

```bash
git merge -s recursive feature-branch
```

Используется для слияния двух веток с общим предком.

### ours

```bash
git merge -s ours feature-branch
```

Игнорирует все изменения из сливаемой ветки, но записывает слияние в историю.

### octopus

```bash
git merge feature1 feature2 feature3
```

Для слияния более двух веток одновременно.

## Best Practices

### Подготовка к слиянию

```bash
# 1. Обновить main
git checkout main
git pull origin main

# 2. Обновить feature-ветку
git checkout feature-branch
git pull origin feature-branch

# 3. Влить main в feature для проверки конфликтов
git merge main
# Разрешить конфликты здесь

# 4. Теперь безопасно слить в main
git checkout main
git merge feature-branch
```

### Правила слияния

1. **Всегда сливайте в обновлённую ветку**
   ```bash
   git checkout main && git pull
   ```

2. **Используйте --no-ff для feature-веток**
   ```bash
   git merge --no-ff feature-x -m "Merge feature-x: description"
   ```

3. **Проверяйте изменения перед слиянием**
   ```bash
   git log main..feature-branch --oneline
   git diff main...feature-branch
   ```

4. **Пишите понятные merge commit сообщения**
   ```bash
   git merge feature-auth -m "Merge feature-auth: JWT authentication with refresh tokens"
   ```

### Merge vs Rebase

| Аспект | Merge | Rebase |
|--------|-------|--------|
| История | Сохраняет полную историю | Линейная история |
| Merge commit | Создаёт | Не создаёт |
| Безопасность | Безопаснее | Требует осторожности |
| Shared ветки | Подходит | Не рекомендуется |
| Feature ветки | Подходит | Предпочтительнее |

## Типичные ошибки

### 1. Слияние без обновления

```bash
# ОШИБКА: main устарел
git checkout main
git merge feature-branch
# Потом конфликты с origin/main!

# ПРАВИЛЬНО: сначала обновить
git checkout main
git pull origin main
git merge feature-branch
```

### 2. Потеря изменений при разрешении конфликтов

```bash
# ОШИБКА: случайно удалили нужный код
<<<<<<< HEAD
important_function()  # удалили случайно
=======
new_function()
>>>>>>> feature

# ПРАВИЛЬНО: внимательно анализировать обе версии
important_function()  # оставить
new_function()        # добавить
```

### 3. Merge в неправильную ветку

```bash
# ОШИБКА: слили main в feature вместо feature в main
git checkout feature-branch
git merge main
# Это не то, что нужно!

# ПРАВИЛЬНО:
git checkout main
git merge feature-branch
```

### 4. Слияние незавершённой работы

```bash
# ОШИБКА: слили ветку с недоделанным кодом
git merge feature-wip
# В main теперь сломанный код!

# ПРАВИЛЬНО: сначала убедиться что feature готова
git checkout feature-wip
npm test
npm run build
git checkout main
git merge feature-wip
```

## Полезные команды

```bash
# Показать коммиты, которые будут слиты
git log main..feature-branch --oneline

# Показать файлы, которые изменятся
git diff main...feature-branch --stat

# Проверить, была ли ветка слита
git branch --merged main

# Показать граф слияний
git log --graph --oneline --all

# Показать последний merge commit
git log --merges -1

# Показать родителей merge commit
git log --format="%H %P" -1 <merge-commit>
```

## Merge в различных Git workflows

### GitHub Flow

```bash
# 1. Создать ветку от main
git checkout main && git checkout -b feature-x

# 2. Работать и коммитить
git commit -m "Add feature x"

# 3. Открыть Pull Request

# 4. После review — merge через GitHub UI
# (Squash and merge / Merge commit / Rebase and merge)
```

### Git Flow

```bash
# Feature в develop
git checkout develop
git merge --no-ff feature-x

# Release в main и develop
git checkout main
git merge --no-ff release-1.0
git checkout develop
git merge --no-ff release-1.0

# Hotfix в main и develop
git checkout main
git merge --no-ff hotfix-1.0.1
git checkout develop
git merge --no-ff hotfix-1.0.1
```
