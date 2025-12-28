# Рассылки PostgreSQL сообщества

[prev: 05-brin](../21-sql-optimization/03-indexes/05-brin.md) | [next: 02-reviewing-patches](./02-reviewing-patches.md)

---

Рассылки (mailing lists) — основной канал коммуникации в сообществе PostgreSQL. Через них обсуждаются новые фичи, баги, анонсы релизов и общие вопросы.

## Основные рассылки

### pgsql-announce
**Назначение:** Официальные анонсы релизов, обновлений безопасности и важных событий.

- Низкий трафик (несколько писем в месяц)
- Только чтение — обычные пользователи не могут писать
- Рекомендуется подписаться всем DBA и разработчикам

### pgsql-general
**Назначение:** Общие вопросы по использованию PostgreSQL.

- Средний трафик (10-50 писем в день)
- Можно задавать вопросы по администрированию, SQL, настройке
- Хорошее место для начинающих

```
Типичные темы:
- Как настроить репликацию?
- Почему запрос работает медленно?
- Как мигрировать с MySQL?
```

### pgsql-hackers
**Назначение:** Разработка ядра PostgreSQL.

- Высокий трафик (50-200 писем в день)
- Обсуждение патчей, архитектуры, новых фич
- Требует глубокого понимания кода PostgreSQL

```
Типичные темы:
- Ревью патчей
- Обсуждение дизайна новых возможностей
- Технические дискуссии о внутренней архитектуре
```

### pgsql-bugs
**Назначение:** Отчёты об ошибках в PostgreSQL.

- Средний трафик
- Сюда отправляют баг-репорты
- Разработчики отслеживают и исправляют проблемы

### pgsql-performance
**Назначение:** Вопросы производительности.

- Низкий/средний трафик
- Оптимизация запросов, настройка конфигурации
- Анализ планов выполнения

### pgsql-docs
**Назначение:** Документация PostgreSQL.

- Низкий трафик
- Обсуждение улучшений документации
- Отчёты об ошибках в документах

## Как подписаться

### Через веб-интерфейс

1. Перейдите на https://www.postgresql.org/list/
2. Выберите нужную рассылку
3. Нажмите "Subscribe"
4. Введите email и подтвердите подписку

### Через email

Отправьте пустое письмо на адрес:
```
<имя-рассылки>-subscribe@postgresql.org

Примеры:
pgsql-general-subscribe@postgresql.org
pgsql-hackers-subscribe@postgresql.org
```

## Этикет рассылок

### Перед отправкой письма

1. **Поищите в архивах** — возможно, вопрос уже задавали
2. **Проверьте документацию** — многие ответы там
3. **Выберите правильную рассылку** — не пишите вопросы новичков в pgsql-hackers

### Форматирование письма

```
Тема: Ясно описывает суть вопроса

Тело письма:
1. Версия PostgreSQL
2. Операционная система
3. Описание проблемы
4. Что уже пробовали
5. Минимальный воспроизводимый пример
```

### Правила хорошего тона

- **Пишите на английском** — это международное сообщество
- **Используйте plain text** — не HTML
- **Отвечайте inline** — цитируйте только релевантные части
- **Не топпостите** — ответ должен идти после цитаты
- **Будьте вежливы** — это волонтёрское сообщество
- **Не отправляйте вложения** — используйте pastebin для больших примеров

### Пример хорошего письма

```
Subject: Slow query with large IN clause on PostgreSQL 16

Hi,

I'm experiencing slow query performance with a large IN clause.

Environment:
- PostgreSQL 16.2
- Ubuntu 22.04
- 32GB RAM, SSD storage

Problem:
A query with IN clause containing 10,000 values takes 30 seconds.

Query:
SELECT * FROM orders WHERE customer_id IN (1, 2, 3, ... 10000 values);

EXPLAIN ANALYZE output:
[paste here]

I've tried:
1. Creating index on customer_id
2. Using ANY(array[...])

Any suggestions?

Thanks,
John
```

## Поиск в архивах

### Официальные архивы

Все рассылки архивируются: https://www.postgresql.org/list/

Поиск:
1. Перейдите на страницу рассылки
2. Используйте поле поиска
3. Можно фильтровать по дате

### Альтернативные источники

- **PostgreSQL Archive Search:** https://www.postgresql.org/search/
- **MARC (Mstrn Archive):** https://marc.info/
- **Google:** `site:postgresql.org pgsql-general "ваш запрос"`

### Полезные поисковые запросы

```bash
# Поиск по конкретной теме
site:postgresql.org pgsql-hackers "logical replication"

# Поиск обсуждений патча
site:postgresql.org pgsql-hackers "CF" "patch-name"

# Поиск решений проблем
site:postgresql.org pgsql-general "error message"
```

## Дополнительные рассылки

| Рассылка | Назначение |
|----------|------------|
| pgsql-admin | Администрирование |
| pgsql-sql | Вопросы по SQL |
| pgsql-novice | Для начинающих |
| pgsql-interfaces | Драйверы и библиотеки |
| pgsql-www | Веб-сайт PostgreSQL |
| pgsql-translators | Перевод документации |

## Другие каналы коммуникации

### IRC/Matrix
- Канал #postgresql на Libera.Chat
- Matrix: #postgresql:libera.chat

### Slack
- PostgreSQL Community Slack
- Требуется приглашение

### Stack Overflow
- Тег [postgresql]
- Хорошо для быстрых ответов на конкретные вопросы

## Best Practices

1. **Начните с pgsql-general** — даже если кажется, что вопрос для hackers
2. **Подпишитесь на pgsql-announce** — важно знать о релизах
3. **Читайте pgsql-hackers** — чтобы понимать направление развития
4. **Используйте дайджест** — если не хотите много писем
5. **Отвечайте на вопросы других** — так вы учитесь и помогаете сообществу

---

[prev: 05-brin](../21-sql-optimization/03-indexes/05-brin.md) | [next: 02-reviewing-patches](./02-reviewing-patches.md)
