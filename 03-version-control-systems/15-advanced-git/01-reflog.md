# Reflog

## Что такое Reflog

**Reflog** (reference log) - это механизм Git, который записывает историю всех изменений указателей (HEAD и ветки) в локальном репозитории. Это "журнал безопасности", который позволяет восстановить потерянные коммиты даже после "опасных" операций вроде `reset --hard` или удаления веток.

### Ключевые особенности

- **Локальный механизм** - reflog существует только в вашем локальном репозитории
- **Временное хранение** - записи автоматически удаляются через определенное время
- **Отслеживает все изменения** - каждое перемещение HEAD фиксируется
- **Спасательный круг** - позволяет вернуться к любому предыдущему состоянию

## Основные команды

### Просмотр reflog

```bash
# Просмотр reflog для HEAD (по умолчанию)
git reflog

# Полная форма команды
git reflog show HEAD

# Просмотр reflog для конкретной ветки
git reflog show main
git reflog show feature/auth

# Просмотр с датами
git reflog --date=iso
git reflog --date=relative
```

### Пример вывода reflog

```bash
$ git reflog
a1b2c3d (HEAD -> main) HEAD@{0}: commit: Add user authentication
e4f5g6h HEAD@{1}: checkout: moving from feature to main
i7j8k9l HEAD@{2}: commit: Fix login bug
m0n1o2p HEAD@{3}: reset: moving to HEAD~2
q3r4s5t HEAD@{4}: commit: Update README
u6v7w8x HEAD@{5}: merge feature: Fast-forward
```

### Формат записей

Каждая запись в reflog содержит:
- **SHA коммита** - хеш состояния
- **Указатель** - `HEAD@{n}` где n - номер записи (0 = последнее изменение)
- **Действие** - что произошло (commit, checkout, reset, merge и т.д.)
- **Описание** - детали операции

## Восстановление потерянных коммитов

### Сценарий: случайный reset --hard

```bash
# Допустим, вы сделали это по ошибке
git reset --hard HEAD~3

# Коммиты "потеряны", но не удалены!
# Смотрим reflog
git reflog

# Вывод:
# a1b2c3d HEAD@{0}: reset: moving to HEAD~3
# x9y8z7w HEAD@{1}: commit: Important feature
# ...

# Восстанавливаем потерянный коммит
git reset --hard x9y8z7w

# Или создаем новую ветку с потерянными изменениями
git branch recovered-work x9y8z7w
```

### Сценарий: отмена неудачного rebase

```bash
# После неудачного rebase
git reflog

# Находим состояние до rebase
# abc1234 HEAD@{5}: rebase (start): checkout main
# def5678 HEAD@{6}: commit: My last commit before rebase

# Возвращаемся к состоянию до rebase
git reset --hard HEAD@{6}
```

## Восстановление удаленных веток

### Восстановление локальной ветки

```bash
# Удаляем ветку (по ошибке)
git branch -D feature/important

# Ищем последний коммит этой ветки в reflog
git reflog | grep feature/important

# Или смотрим reflog конкретной ветки (если она еще в кеше)
git reflog show feature/important

# Находим SHA последнего коммита и восстанавливаем
git branch feature/important abc1234
```

### Поиск потерянных коммитов

```bash
# Показать все "осиротевшие" коммиты
git fsck --lost-found

# Найти коммиты с определенным сообщением
git reflog | grep "fix bug"

# Показать diff для записи reflog
git show HEAD@{3}

# Сравнить текущее состояние с предыдущим
git diff HEAD@{0} HEAD@{5}
```

## Время жизни записей

### Настройки gc (garbage collection)

```bash
# Время жизни записей для достижимых коммитов (по умолчанию 90 дней)
git config gc.reflogExpire 90.days

# Время жизни для недостижимых коммитов (по умолчанию 30 дней)
git config gc.reflogExpireUnreachable 30.days

# Никогда не удалять записи reflog
git config gc.reflogExpire never
git config gc.reflogExpireUnreachable never

# Установить глобально
git config --global gc.reflogExpire 180.days
```

### Принудительная очистка

```bash
# Удалить старые записи reflog (осторожно!)
git reflog expire --expire=now --all

# Запустить сборку мусора
git gc --prune=now

# После этих команд потерянные коммиты будут удалены навсегда!
```

## Практические примеры

### Пример 1: Восстановление после неудачного merge

```bash
# Слили не ту ветку
git merge wrong-branch

# Смотрим reflog
git reflog
# abc123 HEAD@{0}: merge wrong-branch: Merge made
# def456 HEAD@{1}: commit: Last good commit

# Отменяем merge
git reset --hard HEAD@{1}
```

### Пример 2: Найти когда появился баг

```bash
# Смотрим историю изменений
git reflog --date=relative

# def456 HEAD@{2 days ago}: commit: Add new feature
# abc123 HEAD@{3 days ago}: commit: Fix validation

# Проверяем конкретное состояние
git checkout HEAD@{3.days.ago}
# Тестируем... если баг есть, идем дальше назад
```

### Пример 3: Восстановление stash

```bash
# Если случайно удалили stash
git stash drop

# Ищем в reflog
git fsck --unreachable | grep commit

# Или через git stash list до удаления (если помним хеш)
git stash apply abc1234
```

### Пример 4: Отладка CI/CD проблем

```bash
# Посмотреть что происходило в репозитории
git reflog --date=iso

# Найти состояние репозитория на момент сбоя
git checkout HEAD@{2023-12-25.15:30:00}
```

## Синтаксис ссылок на reflog

```bash
# По номеру записи
HEAD@{0}      # текущее состояние
HEAD@{1}      # предыдущее состояние
main@{2}      # 2 изменения назад для ветки main

# По времени
HEAD@{yesterday}
HEAD@{2.weeks.ago}
HEAD@{2023-12-25}
HEAD@{1.hour.ago}

# Использование в командах
git show HEAD@{3}
git diff HEAD@{1} HEAD@{5}
git log HEAD@{yesterday}..HEAD
git checkout HEAD@{2.days.ago}
```

## Best Practices

### Рекомендации по использованию

1. **Проверяйте reflog перед опасными операциями** - знайте куда вернуться
2. **Не полагайтесь только на reflog** - делайте backup важных веток
3. **Увеличьте время хранения** для важных проектов
4. **Используйте для отладки** - понимайте что произошло в репозитории

### Когда reflog не поможет

- После `git gc --prune=now` с очисткой reflog
- Для удаленных репозиториев (у них свой reflog)
- Если прошло больше времени, чем `gc.reflogExpire`
- Для коммитов, которые никогда не были в локальном репозитории

## Полезные алиасы

```bash
# Удобный просмотр reflog
git config --global alias.rl "reflog --date=relative"

# Поиск в reflog
git config --global alias.rls "reflog show"

# Восстановление последнего состояния
git config --global alias.undo "reset --hard HEAD@{1}"
```

## Заключение

Reflog - это мощный инструмент восстановления, который каждый разработчик должен знать. Он не заменяет хорошие практики работы с Git (частые коммиты, ветки для экспериментов), но служит надежной страховкой от ошибок. Помните: в Git практически невозможно потерять закоммиченную работу, если знать про reflog.
