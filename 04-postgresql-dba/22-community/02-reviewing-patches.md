# Ревью патчей PostgreSQL

Ревью патчей — важная часть участия в сообществе PostgreSQL. Это помогает улучшить качество кода и ускорить включение новых функций в релизы.

## Что такое CommitFest

**CommitFest (CF)** — это организованный период ревью патчей, который проходит несколько раз в год.

### Расписание CommitFest

- **Январь** — после feature freeze
- **Март** — основная разработка
- **Июль** — подготовка к следующему релизу
- **Сентябрь** — финальный перед feature freeze
- **Ноябрь** — последний шанс для фич текущего релиза

### Веб-интерфейс CommitFest

https://commitfest.postgresql.org/

```
Статусы патчей:
- Needs review — требует ревью
- Waiting on Author — ожидает ответа от автора
- Ready for Committer — готов к коммиту
- Committed — закоммичен
- Rejected — отклонён
- Withdrawn — отозван автором
- Returned with Feedback — возвращён с комментариями
```

## Как начать ревью

### Выбор патча

1. Перейдите на https://commitfest.postgresql.org/
2. Выберите активный CommitFest
3. Найдите патчи со статусом "Needs review"
4. Выберите патч по интересующей теме

### Критерии выбора для начинающих

```
Хорошо для начала:
- Документация (doc)
- Небольшие патчи (< 200 строк)
- Тесты
- Исправления опечаток

Сложнее:
- Изменения в оптимизаторе
- Новые типы данных
- Изменения WAL
- Репликация
```

## Что проверять при ревью

### 1. Компиляция

```bash
# Скачайте и примените патч
git clone git://git.postgresql.org/git/postgresql.git
cd postgresql
git checkout master
git apply /path/to/patch.patch

# Скомпилируйте
./configure --enable-debug --enable-cassert
make -j$(nproc)
make check  # запуск тестов
```

### 2. Тесты

```bash
# Все регрессионные тесты
make check-world

# Изоляционные тесты
make isolation-check

# TAP-тесты
make prove
```

### 3. Код

**Стиль кода:**
```c
// PostgreSQL использует свой стиль
// 4 пробела для отступов (не табы)
// Скобки на той же строке
if (condition)
{
    do_something();
}
```

**Проверьте:**
- Соответствие coding style (см. src/tools/pgindent)
- Отсутствие утечек памяти
- Обработка ошибок (ereport/elog)
- Комментарии к сложному коду

### 4. Документация

- Обновлена ли документация?
- Добавлена ли информация в release notes?
- Понятны ли примеры?

### 5. Обратная совместимость

- Ломает ли патч существующий функционал?
- Нужна ли миграция данных?
- Влияет ли на производительность?

## Как давать обратную связь

### Формат письма в pgsql-hackers

```
Subject: Re: [PATCH] Add feature X

Hi,

I reviewed the attached patch for adding feature X.

=== Build ===
The patch applies cleanly to current master (commit abc123).
Build: OK
Regression tests: OK

=== Review ===

1. General comments:
   - The feature looks useful
   - Documentation is clear

2. Code comments:

src/backend/executor/nodeHash.c:
+    if (bucket == NULL)
+        return false;

This check seems redundant because bucket is already
validated in the caller. Consider removing.

3. Suggestions:
   - Add test for edge case when...
   - Consider using pg_bswap32 instead of...

=== Testing ===
I tested the following scenarios:
- Basic functionality: works
- Edge case X: works
- Concurrent access: not tested

Overall, the patch looks good with minor adjustments.

Thanks,
Your Name
```

### Что включать в ревью

1. **Версия PostgreSQL/commit** на котором тестировали
2. **Результат компиляции** — собирается ли
3. **Результат тестов** — проходят ли
4. **Конкретные комментарии** к коду с указанием файла и строки
5. **Общая оценка** — готов к коммиту или нужны доработки

### Тон обратной связи

```
✓ Хорошо:
"Consider using X instead of Y because..."
"This could be simplified by..."
"I'm not sure about this approach, could you explain..."

✗ Плохо:
"This is wrong"
"Terrible code"
"You should know better"
```

## Статусы после ревью

### Если патч хороший
Установите статус "Ready for Committer" и напишите:
```
I've reviewed this patch and it looks good.
All tests pass. Ready for committer.
```

### Если нужны исправления
Установите статус "Waiting on Author":
```
The patch needs some changes:
1. Fix issue in file X
2. Add test for case Y
3. Update documentation

Moving to "Waiting on Author".
```

### Если патч нужно отклонить
Статус "Returned with Feedback":
```
After discussion, this approach doesn't seem right because...
Returning with feedback. The author might want to
reconsider the design.
```

## Инструменты для ревью

### Локальный diff

```bash
# Просмотр изменений
git diff --stat HEAD~1
git diff --word-diff HEAD~1

# Интерактивный просмотр
git log -p
```

### Онлайн инструменты

- **GitHub mirror:** https://github.com/postgres/postgres
- **cgit:** https://git.postgresql.org/gitweb/
- **Patchwork:** https://patchwork.postgresql.org/

### Статический анализ

```bash
# Проверка стиля
src/tools/pgindent/pgindent src/backend/modified_file.c

# Clang static analyzer
scan-build make

# Coverity (для коммиттеров)
```

## Чек-лист ревьюера

```
□ Патч применяется к текущему master
□ Код компилируется без предупреждений
□ Все тесты проходят (make check-world)
□ Новый код имеет тесты
□ Документация обновлена
□ Стиль кода соответствует правилам
□ Комментарии понятны
□ Обработка ошибок корректна
□ Нет утечек памяти
□ Обратная совместимость сохранена
□ Release notes обновлены (если нужно)
```

## Best Practices

1. **Начните с простых патчей** — документация, тесты, небольшие исправления
2. **Ревьюйте регулярно** — даже 1-2 патча в месяц помогает
3. **Будьте конструктивны** — предлагайте решения, не только критикуйте
4. **Задавайте вопросы** — если что-то непонятно, спросите автора
5. **Проверяйте edge cases** — часто баги там
6. **Тестируйте вручную** — не только автотесты
7. **Читайте ревью других** — учитесь на примерах

## Полезные ресурсы

- [PostgreSQL Development Information](https://wiki.postgresql.org/wiki/Developer_FAQ)
- [Submitting a Patch](https://wiki.postgresql.org/wiki/Submitting_a_Patch)
- [CommitFest](https://commitfest.postgresql.org/)
- [Mailing List Archives](https://www.postgresql.org/list/pgsql-hackers/)
