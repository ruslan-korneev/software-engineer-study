# Срок хранения данных (Data Retention)

[prev: 04-compacted-topics](../07-topics-partitions/04-compacted-topics.md) | [next: 02-data-movement](./02-data-movement.md)

---

## Описание

Kafka хранит сообщения на диске в виде логов (commit logs). В отличие от традиционных брокеров сообщений, Kafka не удаляет сообщения после их потребления — они остаются доступными для повторного чтения. Политики хранения (retention policies) определяют, как долго данные будут храниться в топике и когда они будут удалены.

## Ключевые концепции

### Типы политик хранения

#### 1. Retention по времени (Time-based)

Сообщения удаляются после истечения заданного времени с момента их записи.

```properties
# Время хранения в миллисекундах (по умолчанию 7 дней)
log.retention.ms=604800000

# Альтернативные настройки (менее приоритетные)
log.retention.minutes=10080
log.retention.hours=168
```

#### 2. Retention по размеру (Size-based)

Старые сегменты удаляются, когда общий размер партиции превышает лимит.

```properties
# Максимальный размер партиции в байтах (-1 = без ограничений)
log.retention.bytes=1073741824  # 1 GB
```

#### 3. Комбинированная политика

Когда заданы обе политики, данные удаляются при достижении **любого** из лимитов.

### Log Segments (Сегменты лога)

Лог партиции разбит на сегменты — файлы фиксированного размера:

```properties
# Размер одного сегмента (по умолчанию 1 GB)
log.segment.bytes=1073741824

# Время ротации сегмента (7 дней)
log.roll.ms=604800000
log.roll.hours=168
```

**Важно:** Retention применяется к целым сегментам, а не к отдельным сообщениям. Активный сегмент (куда идёт запись) никогда не удаляется.

### Log Cleanup Policies

```properties
# Политика очистки: delete или compact
log.cleanup.policy=delete
```

| Политика | Описание |
|----------|----------|
| `delete` | Удаление старых сегментов по времени/размеру |
| `compact` | Сохранение только последнего значения для каждого ключа |
| `compact,delete` | Комбинация обоих подходов |

## Примеры конфигурации

### Настройка на уровне брокера (server.properties)

```properties
# Глобальные настройки retention
log.retention.hours=168
log.retention.bytes=-1
log.segment.bytes=1073741824
log.retention.check.interval.ms=300000

# Минимальное время жизни сегмента перед удалением
log.segment.delete.delay.ms=60000
```

### Настройка на уровне топика

```bash
# Создание топика с кастомным retention
kafka-topics.sh --create \
  --bootstrap-server localhost:9092 \
  --topic events \
  --partitions 6 \
  --replication-factor 3 \
  --config retention.ms=86400000 \
  --config retention.bytes=10737418240

# Изменение retention для существующего топика
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name events \
  --alter \
  --add-config retention.ms=172800000

# Просмотр текущих настроек
kafka-configs.sh --bootstrap-server localhost:9092 \
  --entity-type topics \
  --entity-name events \
  --describe
```

### Специальные значения retention

```properties
# Бесконечное хранение (никогда не удалять)
retention.ms=-1
retention.bytes=-1

# Минимальное хранение (удалять как можно быстрее)
retention.ms=0
```

## Log Compaction (Сжатие логов)

Log compaction — альтернативная политика, сохраняющая только последнее значение для каждого ключа.

```properties
log.cleanup.policy=compact

# Минимальное время перед compaction
min.compaction.lag.ms=0

# Максимальное время без compaction
max.compaction.lag.ms=9223372036854775807

# Минимальный "грязный" коэффициент для запуска
min.cleanable.dirty.ratio=0.5

# Размер дедупликационного буфера
log.cleaner.dedupe.buffer.size=134217728
```

### Как работает compaction

```
До compaction:
Offset  Key     Value
0       user1   {"name": "Alice"}
1       user2   {"name": "Bob"}
2       user1   {"name": "Alice Updated"}
3       user1   {"name": "Alice Final"}
4       user2   {"name": "Bob Updated"}

После compaction:
Offset  Key     Value
3       user1   {"name": "Alice Final"}
4       user2   {"name": "Bob Updated"}
```

### Tombstone records (Маркеры удаления)

```java
// Для удаления ключа из compacted топика отправьте null
producer.send(new ProducerRecord<>("users", "user1", null));
```

```properties
# Время хранения tombstone перед удалением
log.cleaner.delete.retention.ms=86400000  # 24 часа
```

## Best Practices

### 1. Расчёт необходимого retention

```
Требуемое хранилище = Throughput × Retention Time × Replication Factor

Пример:
- Throughput: 100 MB/s
- Retention: 7 дней
- Replication: 3

Хранилище = 100 MB/s × 86400 s × 7 × 3 = 181 TB
```

### 2. Рекомендации по сегментам

```properties
# Оптимальный размер сегмента
log.segment.bytes=1073741824  # 1 GB

# Для топиков с низким throughput используйте time-based roll
log.roll.hours=24

# Для hot топиков с высоким throughput
log.segment.bytes=536870912  # 512 MB
```

### 3. Выбор политики очистки

| Сценарий | Политика |
|----------|----------|
| Логи событий, метрики | `delete` |
| Справочные данные (CDC) | `compact` |
| Changelog для state stores | `compact` |
| Аудит с TTL | `compact,delete` |

### 4. Мониторинг retention

```bash
# Проверка размера топика
kafka-log-dirs.sh --describe \
  --bootstrap-server localhost:9092 \
  --topic-list events

# Проверка earliest offset
kafka-run-class.sh kafka.tools.GetOffsetShell \
  --broker-list localhost:9092 \
  --topic events \
  --time -2
```

### 5. Типичные проблемы

**Проблема:** Данные удаляются раньше, чем ожидалось
```properties
# Проверьте, что segment roll происходит достаточно часто
log.roll.ms=3600000  # 1 час
```

**Проблема:** Диск заполняется быстрее ожидаемого
```properties
# Уменьшите retention или используйте size-based
retention.bytes=107374182400  # 100 GB на партицию
```

**Проблема:** Log compaction не срабатывает
```properties
# Увеличьте количество cleaner threads
log.cleaner.threads=2
# Уменьшите dirty ratio
min.cleanable.dirty.ratio=0.3
```

## Связанные темы

- [Log Segments](./02-data-movement.md) — подробнее о структуре сегментов
- [Инструменты](./03-tools.md) — утилиты для работы с данными
- [Архитектуры](./05-architectures.md) — паттерны использования retention

---

[prev: 04-compacted-topics](../07-topics-partitions/04-compacted-topics.md) | [next: 02-data-movement](./02-data-movement.md)
