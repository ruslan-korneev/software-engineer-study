# Правила совместимости

[prev: 03-registry-components](./03-registry-components.md) | [next: 05-alternatives](./05-alternatives.md)

---

## Описание

Эволюция схем (Schema Evolution) — это процесс изменения структуры данных с течением времени при сохранении совместимости между разными версиями продюсеров и консьюмеров. Schema Registry обеспечивает контроль совместимости, проверяя каждую новую версию схемы перед регистрацией на соответствие заданным правилам.

Правильное управление эволюцией схем критически важно для:
- Независимого деплоя сервисов
- Отсутствия простоев при обновлениях
- Совместного чтения старых и новых сообщений

## Ключевые концепции

### Типы совместимости

Schema Registry поддерживает следующие режимы совместимости:

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         Типы совместимости                              │
├─────────────────────────────────────────────────────────────────────────┤
│                                                                         │
│  BACKWARD (по умолчанию)                                               │
│  ├── Новые консьюмеры могут читать старые данные                       │
│  └── Позволяет: добавить поле с default, удалить поле                  │
│                                                                         │
│  FORWARD                                                                │
│  ├── Старые консьюмеры могут читать новые данные                       │
│  └── Позволяет: удалить поле с default, добавить поле                  │
│                                                                         │
│  FULL                                                                   │
│  ├── И BACKWARD, и FORWARD одновременно                                │
│  └── Позволяет: только добавить/удалить поля с default                 │
│                                                                         │
│  NONE                                                                   │
│  └── Без проверки совместимости (не рекомендуется)                     │
│                                                                         │
│  BACKWARD_TRANSITIVE                                                    │
│  └── BACKWARD со ВСЕМИ предыдущими версиями                            │
│                                                                         │
│  FORWARD_TRANSITIVE                                                     │
│  └── FORWARD со ВСЕМИ предыдущими версиями                             │
│                                                                         │
│  FULL_TRANSITIVE                                                        │
│  └── FULL со ВСЕМИ предыдущими версиями                                │
│                                                                         │
└─────────────────────────────────────────────────────────────────────────┘
```

### Матрица совместимости

| Изменение | BACKWARD | FORWARD | FULL | NONE |
|-----------|----------|---------|------|------|
| Добавить опциональное поле (с default) | Да | Да | Да | Да |
| Добавить обязательное поле (без default) | Нет | Да | Нет | Да |
| Удалить опциональное поле (с default) | Да | Да | Да | Да |
| Удалить обязательное поле (без default) | Да | Нет | Нет | Да |
| Переименовать поле | Нет | Нет | Нет | Да |
| Изменить тип поля | Нет* | Нет* | Нет | Да |
| Добавить значение в enum | Нет | Да | Нет | Да |
| Удалить значение из enum | Да | Нет | Нет | Да |

*Некоторые типы можно продвигать (int → long, float → double)

### Визуализация совместимости

```
                    Время
        ──────────────────────────────▶

        v1          v2          v3
        │           │           │
        ▼           ▼           ▼
      ┌───┐       ┌───┐       ┌───┐
      │ S │       │ S │       │ S │      Схемы
      │ 1 │       │ 2 │       │ 3 │
      └───┘       └───┘       └───┘

BACKWARD: Новый консьюмер читает все версии данных
        ┌─────────────────────────────┐
        │    Consumer v3              │
        │  читает v1, v2, v3 данные   │
        └─────────────────────────────┘
                        │
            ◀───────────┴───────────▶

FORWARD: Старые консьюмеры читают новые данные
        ┌─────────────────────────────┐
        │    Consumer v1              │
        │  читает v1, v2, v3 данные   │
        └─────────────────────────────┘
                        │
            ◀───────────┴───────────▶

FULL: И BACKWARD, и FORWARD
        ┌─────────────────────────────┐
        │ Любой Consumer читает       │
        │ любые версии данных         │
        └─────────────────────────────┘
```

### Transitive vs Non-Transitive

```
Версии схемы: v1 → v2 → v3 → v4

Non-Transitive (BACKWARD):
Проверка: v4 совместима с v3? ✓

Transitive (BACKWARD_TRANSITIVE):
Проверки: v4 совместима с v3? ✓
          v4 совместима с v2? ✓
          v4 совместима с v1? ✓

Когда использовать Transitive:
- Консьюмеры могут читать очень старые сообщения
- Retention период топика большой
- Нужна гарантия совместимости со всей историей
```

## Примеры кода

### Настройка совместимости через REST API

```bash
# Получить глобальный уровень совместимости
curl -X GET http://localhost:8081/config
# {"compatibilityLevel":"BACKWARD"}

# Установить глобальный уровень совместимости
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "FULL"}' \
  http://localhost:8081/config

# Получить уровень совместимости для субъекта
curl -X GET http://localhost:8081/config/orders-value
# {"compatibilityLevel":"BACKWARD"}

# Установить уровень совместимости для субъекта
curl -X PUT -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"compatibility": "BACKWARD_TRANSITIVE"}' \
  http://localhost:8081/config/orders-value

# Удалить переопределение для субъекта (вернуться к глобальному)
curl -X DELETE http://localhost:8081/config/orders-value
```

### Проверка совместимости перед регистрацией

```bash
# Проверить совместимость с последней версией
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{
    "schema": "{\"type\":\"record\",\"name\":\"User\",\"fields\":[{\"name\":\"id\",\"type\":\"long\"},{\"name\":\"name\",\"type\":\"string\"},{\"name\":\"email\",\"type\":[\"null\",\"string\"],\"default\":null}]}"
  }' \
  http://localhost:8081/compatibility/subjects/users-value/versions/latest

# Ответ при успехе: {"is_compatible":true}
# Ответ при неудаче: {"is_compatible":false}

# Проверить совместимость с конкретной версией
curl -X POST -H "Content-Type: application/vnd.schemaregistry.v1+json" \
  --data '{"schema": "..."}' \
  http://localhost:8081/compatibility/subjects/users-value/versions/2
```

### Python: Проверка совместимости

```python
from confluent_kafka.schema_registry import SchemaRegistryClient, Schema

schema_registry_conf = {'url': 'http://localhost:8081'}
client = SchemaRegistryClient(schema_registry_conf)

# Текущая схема (v1)
current_schema = """
{
    "type": "record",
    "name": "User",
    "namespace": "com.example",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"}
    ]
}
"""

# Новая схема (v2) - добавляем опциональное поле
new_schema = """
{
    "type": "record",
    "name": "User",
    "namespace": "com.example",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"},
        {"name": "email", "type": ["null", "string"], "default": null}
    ]
}
"""

subject = "users-value"

# Регистрация первой версии
schema_v1 = Schema(current_schema, "AVRO")
schema_id_v1 = client.register_schema(subject, schema_v1)
print(f"Зарегистрирована схема v1 с ID: {schema_id_v1}")

# Проверка совместимости новой схемы
schema_v2 = Schema(new_schema, "AVRO")
is_compatible = client.test_compatibility(subject, schema_v2)
print(f"Новая схема совместима: {is_compatible}")

if is_compatible:
    schema_id_v2 = client.register_schema(subject, schema_v2)
    print(f"Зарегистрирована схема v2 с ID: {schema_id_v2}")
else:
    print("Ошибка: схема несовместима!")

# Получение информации о совместимости
config = client.get_compatibility(subject)
print(f"Уровень совместимости для {subject}: {config}")
```

### Примеры эволюции схем в Avro

#### BACKWARD-совместимые изменения

```python
# Версия 1: Исходная схема
v1_schema = """
{
    "type": "record",
    "name": "Order",
    "fields": [
        {"name": "order_id", "type": "string"},
        {"name": "customer_id", "type": "long"},
        {"name": "amount", "type": "double"}
    ]
}
"""

# Версия 2: Добавление опционального поля (BACKWARD ✓)
v2_schema = """
{
    "type": "record",
    "name": "Order",
    "fields": [
        {"name": "order_id", "type": "string"},
        {"name": "customer_id", "type": "long"},
        {"name": "amount", "type": "double"},
        {"name": "discount", "type": ["null", "double"], "default": null}
    ]
}
"""

# Версия 3: Удаление поля (BACKWARD ✓, но FORWARD ✗)
# Новый консьюмер просто игнорирует отсутствующее поле
v3_schema = """
{
    "type": "record",
    "name": "Order",
    "fields": [
        {"name": "order_id", "type": "string"},
        {"name": "customer_id", "type": "long"}
    ]
}
"""
# Примечание: amount удалён, новый консьюмер не ожидает это поле
```

#### FORWARD-совместимые изменения

```python
# Версия 1: Исходная схема
v1_schema = """
{
    "type": "record",
    "name": "Product",
    "fields": [
        {"name": "id", "type": "string"},
        {"name": "name", "type": "string"},
        {"name": "category", "type": ["null", "string"], "default": null}
    ]
}
"""

# Версия 2: Удаление опционального поля (FORWARD ✓)
# Старый консьюмер получит null для category
v2_schema = """
{
    "type": "record",
    "name": "Product",
    "fields": [
        {"name": "id", "type": "string"},
        {"name": "name", "type": "string"}
    ]
}
"""

# Версия 3: Добавление обязательного поля (FORWARD ✓, но BACKWARD ✗)
# Старый консьюмер просто игнорирует новое поле
v3_schema = """
{
    "type": "record",
    "name": "Product",
    "fields": [
        {"name": "id", "type": "string"},
        {"name": "name", "type": "string"},
        {"name": "sku", "type": "string"}
    ]
}
"""
```

#### FULL-совместимые изменения

```python
# Только добавление/удаление опциональных полей с default

# Версия 1
v1_schema = """
{
    "type": "record",
    "name": "Customer",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"},
        {"name": "email", "type": ["null", "string"], "default": null}
    ]
}
"""

# Версия 2: Добавление опционального поля (FULL ✓)
v2_schema = """
{
    "type": "record",
    "name": "Customer",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"},
        {"name": "email", "type": ["null", "string"], "default": null},
        {"name": "phone", "type": ["null", "string"], "default": null}
    ]
}
"""

# Версия 3: Удаление опционального поля (FULL ✓)
v3_schema = """
{
    "type": "record",
    "name": "Customer",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"},
        {"name": "phone", "type": ["null", "string"], "default": null}
    ]
}
"""
```

### Эволюция схем в Protobuf

```protobuf
// Версия 1
message User {
    int64 id = 1;
    string name = 2;
}

// Версия 2: Добавление поля (совместимо)
message User {
    int64 id = 1;
    string name = 2;
    optional string email = 3;  // Новое поле
}

// Версия 3: Удаление поля (осторожно!)
message User {
    int64 id = 1;
    string name = 2;
    reserved 3;  // email удалён, номер зарезервирован
    reserved "email";
    optional string phone = 4;
}

// ВАЖНО: Никогда не переиспользуйте номера полей!
// reserved защищает от случайного переиспользования
```

### Эволюция схем в JSON Schema

```python
# Версия 1
v1_schema = {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
        "id": {"type": "integer"},
        "name": {"type": "string"}
    },
    "required": ["id", "name"],
    "additionalProperties": False
}

# Версия 2: Добавление опционального свойства
v2_schema = {
    "$schema": "http://json-schema.org/draft-07/schema#",
    "type": "object",
    "properties": {
        "id": {"type": "integer"},
        "name": {"type": "string"},
        "email": {"type": ["string", "null"]}  # Опциональное
    },
    "required": ["id", "name"],  # email не в required
    "additionalProperties": False
}

# ВАЖНО для JSON Schema:
# - additionalProperties: false требует осторожности при эволюции
# - Лучше использовать additionalProperties: true для гибкости
# - Или удалить additionalProperties для разрешения дополнительных полей
```

## Стратегии миграции схем

### Стратегия 1: Инкрементальная миграция

```
Шаг 1: Добавить новое поле как опциональное
       v1: {id, name}
       v2: {id, name, email?}

Шаг 2: Обновить всех продюсеров
       Все начинают заполнять email

Шаг 3: Обновить всех консьюмеров
       Все начинают использовать email

Шаг 4: Сделать поле обязательным (если нужно)
       v3: {id, name, email}
```

### Стратегия 2: Dual-write (двойная запись)

```python
# Переименование поля: user_name → name

# Шаг 1: Добавить новое поле, писать в оба
v2_schema = """
{
    "type": "record",
    "name": "User",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "user_name", "type": "string"},
        {"name": "name", "type": ["null", "string"], "default": null}
    ]
}
"""

# Producer пишет в оба поля
def produce_user(user):
    record = {
        "id": user.id,
        "user_name": user.name,  # Старое поле
        "name": user.name        # Новое поле
    }
    producer.produce("users", value=serialize(record))

# Шаг 2: Обновить консьюмеров читать из name
# Шаг 3: Удалить user_name
v3_schema = """
{
    "type": "record",
    "name": "User",
    "fields": [
        {"name": "id", "type": "long"},
        {"name": "name", "type": "string"}
    ]
}
"""
```

### Стратегия 3: Новый топик

```
Когда изменения слишком большие:

1. Создать новый топик с новой схемой
   orders-v1 → orders-v2

2. Настроить dual-write в оба топика
   Producer → orders-v1
           → orders-v2

3. Мигрировать консьюмеров на новый топик
   Consumer ← orders-v2

4. Отключить запись в старый топик
   Producer → orders-v2

5. Удалить старый топик после истечения retention
```

## Автоматизация проверки совместимости

### CI/CD интеграция

```yaml
# .github/workflows/schema-check.yml
name: Schema Compatibility Check

on:
  pull_request:
    paths:
      - 'schemas/**'

jobs:
  check-compatibility:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3

      - name: Start Schema Registry
        run: |
          docker-compose up -d schema-registry
          sleep 10

      - name: Check Schema Compatibility
        run: |
          for schema_file in schemas/*.avsc; do
            subject=$(basename "$schema_file" .avsc)-value
            schema=$(cat "$schema_file" | jq -c .)

            result=$(curl -s -X POST \
              -H "Content-Type: application/vnd.schemaregistry.v1+json" \
              --data "{\"schema\": $(echo $schema | jq -Rs .)}" \
              "http://localhost:8081/compatibility/subjects/${subject}/versions/latest")

            is_compatible=$(echo $result | jq -r '.is_compatible')

            if [ "$is_compatible" != "true" ]; then
              echo "Schema $subject is NOT compatible!"
              exit 1
            fi

            echo "Schema $subject is compatible"
          done
```

### Python скрипт для проверки

```python
#!/usr/bin/env python3
"""Скрипт проверки совместимости схем для CI/CD"""

import sys
import json
import glob
from pathlib import Path
from confluent_kafka.schema_registry import SchemaRegistryClient, Schema

def check_schemas(registry_url: str, schemas_dir: str) -> bool:
    client = SchemaRegistryClient({'url': registry_url})
    all_compatible = True

    for schema_file in glob.glob(f"{schemas_dir}/*.avsc"):
        subject = Path(schema_file).stem + "-value"

        with open(schema_file) as f:
            schema_str = f.read()

        schema = Schema(schema_str, "AVRO")

        try:
            # Проверяем, существует ли субъект
            versions = client.get_versions(subject)

            # Если существует, проверяем совместимость
            is_compatible = client.test_compatibility(subject, schema)

            if is_compatible:
                print(f"[OK] {subject}: совместима")
            else:
                print(f"[FAIL] {subject}: НЕ совместима!")
                all_compatible = False

        except Exception as e:
            if "Subject not found" in str(e):
                print(f"[NEW] {subject}: новая схема")
            else:
                print(f"[ERROR] {subject}: {e}")
                all_compatible = False

    return all_compatible

if __name__ == "__main__":
    registry_url = sys.argv[1] if len(sys.argv) > 1 else "http://localhost:8081"
    schemas_dir = sys.argv[2] if len(sys.argv) > 2 else "./schemas"

    success = check_schemas(registry_url, schemas_dir)
    sys.exit(0 if success else 1)
```

## Best Practices

### Выбор уровня совместимости

| Сценарий | Рекомендуемый уровень |
|----------|----------------------|
| Стандартный случай | BACKWARD |
| Критичные данные | FULL_TRANSITIVE |
| Быстрая разработка | BACKWARD |
| Много legacy консьюмеров | FORWARD или FULL |
| Короткий retention | BACKWARD (не transitive) |
| Длинный retention | BACKWARD_TRANSITIVE |

### Правила проектирования схем для эволюции

1. **Всегда добавляйте default для новых полей**:
   ```json
   {"name": "email", "type": ["null", "string"], "default": null}
   ```

2. **Используйте union типы для nullable полей**:
   ```json
   {"name": "optional_field", "type": ["null", "string"], "default": null}
   ```

3. **Документируйте изменения**:
   ```json
   {
       "name": "status",
       "type": "string",
       "doc": "Добавлено в v2. Возможные значения: ACTIVE, INACTIVE"
   }
   ```

4. **Резервируйте удаленные поля** (Protobuf):
   ```protobuf
   reserved 3, 4, 5;
   reserved "old_field", "deprecated_field";
   ```

5. **Версионируйте схемы в Git**:
   ```
   schemas/
   ├── users/
   │   ├── v1.avsc
   │   ├── v2.avsc
   │   └── current.avsc -> v2.avsc
   └── orders/
       ├── v1.avsc
       └── current.avsc -> v1.avsc
   ```

### Чеклист перед изменением схемы

- [ ] Определён уровень совместимости для субъекта
- [ ] Новые поля имеют значения по умолчанию
- [ ] Проверена совместимость через API или тесты
- [ ] Обновлена документация схемы
- [ ] План отката при проблемах
- [ ] Уведомлены владельцы консьюмеров

### Типичные ошибки

1. **Изменение типа поля**:
   ```
   Плохо: "type": "int" → "type": "string"
   Хорошо: Создать новое поле и мигрировать данные
   ```

2. **Удаление обязательного поля без default в BACKWARD**:
   ```
   v1: {"name": "email", "type": "string"}  // Обязательное
   v2: // email удалён - ОШИБКА!

   Решение: Сначала добавить default в v1.5
   ```

3. **Переименование поля**:
   ```
   Плохо: "name": "user_name" → "name": "name"
   Хорошо: Dual-write с постепенной миграцией
   ```

4. **Игнорирование aliases в Avro**:
   ```json
   {
       "name": "name",
       "type": "string",
       "aliases": ["user_name", "username"]
   }
   ```

---

[prev: 03-registry-components](./03-registry-components.md) | [next: 05-alternatives](./05-alternatives.md)
