# Моделирование данных в Elasticsearch

## Введение в маппинги

Маппинг (mapping) — это определение структуры документов в индексе: какие поля существуют, какого они типа и как должны обрабатываться.

## Создание маппинга

### Явное определение маппинга

```bash
curl -X PUT "localhost:9200/products" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "id": {
        "type": "keyword"
      },
      "name": {
        "type": "text",
        "analyzer": "standard",
        "fields": {
          "keyword": {
            "type": "keyword",
            "ignore_above": 256
          }
        }
      },
      "description": {
        "type": "text",
        "analyzer": "russian"
      },
      "price": {
        "type": "float"
      },
      "quantity": {
        "type": "integer"
      },
      "in_stock": {
        "type": "boolean"
      },
      "categories": {
        "type": "keyword"
      },
      "created_at": {
        "type": "date",
        "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
      },
      "location": {
        "type": "geo_point"
      },
      "attributes": {
        "type": "object",
        "properties": {
          "color": { "type": "keyword" },
          "size": { "type": "keyword" }
        }
      }
    }
  }
}'
```

## Основные типы полей

### Строковые типы

```json
{
  "properties": {
    "title": {
      "type": "text"
    },
    "status": {
      "type": "keyword"
    }
  }
}
```

| Тип | Назначение | Анализируется | Подходит для |
|-----|------------|---------------|--------------|
| **text** | Полнотекстовый поиск | Да | Заголовки, описания, статьи |
| **keyword** | Точное совпадение | Нет | ID, статусы, email, теги |

### Multi-fields (комбинированные поля)

```json
{
  "properties": {
    "title": {
      "type": "text",
      "analyzer": "standard",
      "fields": {
        "keyword": {
          "type": "keyword",
          "ignore_above": 256
        },
        "autocomplete": {
          "type": "text",
          "analyzer": "autocomplete_analyzer"
        }
      }
    }
  }
}
```

**Использование:**
```json
{
  "query": {
    "bool": {
      "should": [
        { "match": { "title": "поисковый запрос" } },
        { "term": { "title.keyword": "Точное название" } }
      ]
    }
  }
}
```

### Числовые типы

```json
{
  "properties": {
    "age": { "type": "integer" },
    "price": { "type": "float" },
    "rating": { "type": "half_float" },
    "population": { "type": "long" },
    "precision_value": { "type": "double" },
    "scaled_price": {
      "type": "scaled_float",
      "scaling_factor": 100
    }
  }
}
```

| Тип | Диапазон | Размер | Использование |
|-----|----------|--------|---------------|
| **byte** | -128 to 127 | 1 байт | Маленькие числа |
| **short** | -32768 to 32767 | 2 байта | |
| **integer** | -2^31 to 2^31-1 | 4 байта | Стандартные числа |
| **long** | -2^63 to 2^63-1 | 8 байт | Большие числа, timestamps |
| **float** | 32-bit IEEE 754 | 4 байта | Дробные числа |
| **double** | 64-bit IEEE 754 | 8 байт | Высокая точность |
| **half_float** | 16-bit IEEE 754 | 2 байта | Экономия места |
| **scaled_float** | Масштабированный | | Цены с фиксированной точностью |

### Дата и время

```json
{
  "properties": {
    "created_at": {
      "type": "date",
      "format": "yyyy-MM-dd HH:mm:ss||yyyy-MM-dd||epoch_millis"
    },
    "updated_at": {
      "type": "date"
    },
    "date_range": {
      "type": "date_range",
      "format": "yyyy-MM-dd"
    }
  }
}
```

**Примеры форматов:**
```
strict_date_optional_time    "2024-01-15T10:30:00Z"
epoch_millis                 1705312200000
yyyy-MM-dd                   "2024-01-15"
yyyy-MM-dd HH:mm:ss          "2024-01-15 10:30:00"
```

### Boolean

```json
{
  "properties": {
    "is_active": {
      "type": "boolean"
    }
  }
}
```

**Принимаемые значения:**
```
true:  true, "true", "yes", "on", "1"
false: false, "false", "no", "off", "0", "" (empty string)
```

### Геоданные

```json
{
  "properties": {
    "location": {
      "type": "geo_point"
    },
    "area": {
      "type": "geo_shape"
    }
  }
}
```

**Форматы geo_point:**
```json
{ "location": { "lat": 55.7558, "lon": 37.6173 } }
{ "location": "55.7558,37.6173" }
{ "location": [37.6173, 55.7558] }
{ "location": "u09tvpdj9q3" }
```

### Range типы

```json
{
  "properties": {
    "age_range": { "type": "integer_range" },
    "price_range": { "type": "float_range" },
    "date_range": { "type": "date_range" },
    "ip_range": { "type": "ip_range" }
  }
}
```

**Индексация:**
```json
{
  "age_range": {
    "gte": 18,
    "lte": 65
  },
  "date_range": {
    "gte": "2024-01-01",
    "lt": "2024-12-31"
  }
}
```

## Сложные типы данных

### Object (вложенные объекты)

```json
{
  "properties": {
    "author": {
      "type": "object",
      "properties": {
        "first_name": { "type": "text" },
        "last_name": { "type": "text" },
        "email": { "type": "keyword" }
      }
    }
  }
}
```

**Документ:**
```json
{
  "title": "Elasticsearch Guide",
  "author": {
    "first_name": "John",
    "last_name": "Doe",
    "email": "john@example.com"
  }
}
```

**Внутреннее представление (flattened):**
```json
{
  "title": "Elasticsearch Guide",
  "author.first_name": "John",
  "author.last_name": "Doe",
  "author.email": "john@example.com"
}
```

### Проблема с массивами объектов

```json
{
  "comments": [
    { "author": "Alice", "text": "Great!" },
    { "author": "Bob", "text": "Bad!" }
  ]
}
```

**Flattened (теряется связь):**
```json
{
  "comments.author": ["Alice", "Bob"],
  "comments.text": ["Great!", "Bad!"]
}
```

**Запрос найдёт документ, хотя Alice не писала "Bad!":**
```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "comments.author": "Alice" } },
        { "match": { "comments.text": "Bad" } }
      ]
    }
  }
}
```

### Nested тип (решение проблемы)

```json
{
  "mappings": {
    "properties": {
      "comments": {
        "type": "nested",
        "properties": {
          "author": { "type": "keyword" },
          "text": { "type": "text" },
          "date": { "type": "date" }
        }
      }
    }
  }
}
```

**Правильный запрос для nested:**
```json
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            { "match": { "comments.author": "Alice" } },
            { "match": { "comments.text": "Great" } }
          ]
        }
      }
    }
  }
}
```

### Join тип (Parent-Child)

```json
{
  "mappings": {
    "properties": {
      "my_join_field": {
        "type": "join",
        "relations": {
          "question": "answer"
        }
      },
      "title": { "type": "text" },
      "body": { "type": "text" }
    }
  }
}
```

**Индексация parent:**
```json
PUT /forum/_doc/1
{
  "title": "How to use Elasticsearch?",
  "my_join_field": "question"
}
```

**Индексация child:**
```json
PUT /forum/_doc/2?routing=1
{
  "body": "Read the documentation!",
  "my_join_field": {
    "name": "answer",
    "parent": "1"
  }
}
```

**Запрос children:**
```json
{
  "query": {
    "has_parent": {
      "parent_type": "question",
      "query": {
        "match": { "title": "Elasticsearch" }
      }
    }
  }
}
```

## Динамические маппинги

### Автоматическое определение типов

```json
{
  "mappings": {
    "dynamic": true
  }
}
```

| JSON тип | Elasticsearch тип |
|----------|------------------|
| null | Поле игнорируется |
| true/false | boolean |
| floating point | float |
| integer | long |
| object | object |
| array | Тип первого элемента |
| string (date format) | date |
| string (number) | float или long |
| string | text + keyword (multi-field) |

### Режимы dynamic

```json
{
  "mappings": {
    "dynamic": "strict",
    "properties": {
      "known_field": { "type": "text" }
    }
  }
}
```

| Режим | Новые поля | Индексация документа |
|-------|------------|---------------------|
| **true** | Автоматически добавляются | Успешно |
| **false** | Игнорируются (не индексируются) | Успешно |
| **strict** | Ошибка | Отклоняется |
| **runtime** | Создаются как runtime fields | Успешно |

### Dynamic templates

```json
{
  "mappings": {
    "dynamic_templates": [
      {
        "strings_as_keywords": {
          "match_mapping_type": "string",
          "match": "*_id",
          "mapping": {
            "type": "keyword"
          }
        }
      },
      {
        "longs_as_integers": {
          "match_mapping_type": "long",
          "mapping": {
            "type": "integer"
          }
        }
      },
      {
        "russian_text": {
          "match": "*_ru",
          "mapping": {
            "type": "text",
            "analyzer": "russian"
          }
        }
      },
      {
        "unindexed_objects": {
          "match_mapping_type": "object",
          "match": "metadata*",
          "mapping": {
            "type": "object",
            "enabled": false
          }
        }
      }
    ]
  }
}
```

### Условия для dynamic templates

```json
{
  "dynamic_templates": [
    {
      "template_name": {
        "match": "field_*",
        "unmatch": "*_skip",
        "match_mapping_type": "string",
        "path_match": "parent.child.*",
        "path_unmatch": "*.internal",
        "mapping": {
          "type": "keyword"
        }
      }
    }
  ]
}
```

## Параметры полей

### Основные параметры

```json
{
  "properties": {
    "content": {
      "type": "text",
      "index": true,
      "store": false,
      "analyzer": "standard",
      "search_analyzer": "standard",
      "boost": 2.0,
      "copy_to": "full_content",
      "eager_global_ordinals": false,
      "fielddata": false,
      "index_options": "positions",
      "index_phrases": false,
      "index_prefixes": {
        "min_chars": 2,
        "max_chars": 5
      },
      "norms": true,
      "position_increment_gap": 100,
      "similarity": "BM25",
      "term_vector": "no"
    }
  }
}
```

| Параметр | Описание |
|----------|----------|
| **index** | Индексировать ли поле (default: true) |
| **store** | Хранить ли отдельно от _source (default: false) |
| **analyzer** | Анализатор для индексации |
| **search_analyzer** | Анализатор для поиска |
| **boost** | Вес поля при scoring (deprecated in 7.x) |
| **copy_to** | Копировать значение в другое поле |
| **norms** | Хранить нормы для scoring (default: true для text) |
| **doc_values** | Хранить column-oriented данные (default: true) |
| **null_value** | Значение для null |
| **ignore_above** | Максимальная длина для keyword |
| **coerce** | Приведение типов (default: true) |

### Пример copy_to

```json
{
  "mappings": {
    "properties": {
      "first_name": {
        "type": "text",
        "copy_to": "full_name"
      },
      "last_name": {
        "type": "text",
        "copy_to": "full_name"
      },
      "full_name": {
        "type": "text"
      }
    }
  }
}
```

### Runtime fields

```json
{
  "mappings": {
    "runtime": {
      "day_of_week": {
        "type": "keyword",
        "script": {
          "source": "emit(doc['@timestamp'].value.dayOfWeekEnum.getDisplayName(TextStyle.FULL, Locale.ROOT))"
        }
      },
      "price_with_tax": {
        "type": "double",
        "script": {
          "source": "emit(doc['price'].value * 1.2)"
        }
      }
    },
    "properties": {
      "price": { "type": "double" },
      "@timestamp": { "type": "date" }
    }
  }
}
```

## Изменение маппинга

### Добавление новых полей

```bash
# Можно добавлять новые поля
curl -X PUT "localhost:9200/products/_mapping" -H 'Content-Type: application/json' -d'
{
  "properties": {
    "new_field": {
      "type": "keyword"
    }
  }
}'
```

### Изменение существующих полей (Reindex)

```bash
# Нельзя изменить тип существующего поля!
# Нужно создать новый индекс и переиндексировать

# 1. Создать новый индекс с правильным маппингом
curl -X PUT "localhost:9200/products_v2" -H 'Content-Type: application/json' -d'
{
  "mappings": {
    "properties": {
      "price": { "type": "double" }
    }
  }
}'

# 2. Переиндексировать данные
curl -X POST "localhost:9200/_reindex" -H 'Content-Type: application/json' -d'
{
  "source": { "index": "products" },
  "dest": { "index": "products_v2" }
}'

# 3. Использовать alias для переключения
curl -X POST "localhost:9200/_aliases" -H 'Content-Type: application/json' -d'
{
  "actions": [
    { "remove": { "index": "products", "alias": "products_current" } },
    { "add": { "index": "products_v2", "alias": "products_current" } }
  ]
}'
```

## Best Practices

### Проектирование маппингов

```
✅ Всегда определяйте маппинг явно перед загрузкой данных
✅ Используйте keyword для ID, статусов, тегов
✅ Используйте multi-fields для полей, требующих разный анализ
✅ Отключайте norms для полей без scoring
✅ Используйте doc_values: false для полей без сортировки/агрегаций
✅ Планируйте nested/join заранее — их сложно добавить позже
✅ Используйте aliases для версионирования индексов
```

### Оптимизация хранения

```json
{
  "properties": {
    "internal_id": {
      "type": "keyword",
      "doc_values": false,
      "index": false
    },
    "log_message": {
      "type": "text",
      "norms": false,
      "index_options": "freqs"
    },
    "metadata": {
      "type": "object",
      "enabled": false
    }
  }
}
```

### Типичные ошибки

```
❌ Полагаться на dynamic mapping в production
❌ Использовать object для массивов связанных объектов (нужен nested)
❌ Слишком много nested документов (лимит 10000 по умолчанию)
❌ Не использовать ignore_above для keyword полей
❌ Хранить большие тексты как keyword
```
