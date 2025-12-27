# Основы поиска в Elasticsearch

## Query DSL

Query DSL (Domain Specific Language) — это JSON-based язык запросов Elasticsearch. Он предоставляет полный контроль над поиском, фильтрацией и скорингом документов.

## Структура поискового запроса

```bash
curl -X GET "localhost:9200/products/_search" -H 'Content-Type: application/json' -d'
{
  "query": {
    "match": {
      "name": "laptop"
    }
  },
  "from": 0,
  "size": 10,
  "sort": [
    { "price": "asc" }
  ],
  "_source": ["name", "price", "category"],
  "highlight": {
    "fields": {
      "name": {}
    }
  }
}'
```

### Ответ на поисковый запрос

```json
{
  "took": 5,
  "timed_out": false,
  "_shards": {
    "total": 3,
    "successful": 3,
    "skipped": 0,
    "failed": 0
  },
  "hits": {
    "total": {
      "value": 150,
      "relation": "eq"
    },
    "max_score": 1.5,
    "hits": [
      {
        "_index": "products",
        "_id": "1",
        "_score": 1.5,
        "_source": {
          "name": "Gaming Laptop",
          "price": 1500,
          "category": "electronics"
        },
        "highlight": {
          "name": ["Gaming <em>Laptop</em>"]
        }
      }
    ]
  }
}
```

## Query vs Filter Context

### Query Context (влияет на _score)

```json
{
  "query": {
    "match": {
      "description": "быстрый поиск"
    }
  }
}
```

- Вычисляет релевантность (_score)
- Результаты сортируются по score
- Не кэшируется

### Filter Context (не влияет на _score)

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "published" } },
        { "range": { "price": { "gte": 100, "lte": 500 } } }
      ]
    }
  }
}
```

- Отвечает только "да/нет"
- Не вычисляет score (быстрее)
- Кэшируется для повторного использования

## Match Queries

### match (полнотекстовый поиск)

```json
{
  "query": {
    "match": {
      "name": {
        "query": "wireless keyboard",
        "operator": "and",
        "fuzziness": "AUTO",
        "prefix_length": 2,
        "minimum_should_match": "75%"
      }
    }
  }
}
```

| Параметр | Описание | Значение по умолчанию |
|----------|----------|----------------------|
| **operator** | Логика между терминами (or/and) | or |
| **fuzziness** | Допуск опечаток (0, 1, 2, AUTO) | 0 |
| **prefix_length** | Символы без fuzzy в начале | 0 |
| **minimum_should_match** | Минимум совпадающих терминов | — |
| **analyzer** | Анализатор для запроса | Из маппинга поля |

### match_phrase (поиск фразы)

```json
{
  "query": {
    "match_phrase": {
      "description": {
        "query": "quick brown fox",
        "slop": 2
      }
    }
  }
}
```

- Ищет слова в указанном порядке
- `slop` — допустимое расстояние между словами

### match_phrase_prefix (автодополнение)

```json
{
  "query": {
    "match_phrase_prefix": {
      "name": {
        "query": "gaming lap",
        "max_expansions": 50
      }
    }
  }
}
```

### multi_match (поиск по нескольким полям)

```json
{
  "query": {
    "multi_match": {
      "query": "elasticsearch guide",
      "fields": ["title^3", "description^2", "content"],
      "type": "best_fields",
      "tie_breaker": 0.3,
      "fuzziness": "AUTO"
    }
  }
}
```

**Типы multi_match:**

| Тип | Описание |
|-----|----------|
| **best_fields** | Максимальный score из полей (default) |
| **most_fields** | Сумма scores из всех полей |
| **cross_fields** | Термины ищутся по всем полям вместе |
| **phrase** | match_phrase по каждому полю |
| **phrase_prefix** | match_phrase_prefix по каждому полю |

## Term-level Queries

### term (точное совпадение)

```json
{
  "query": {
    "term": {
      "status": {
        "value": "published",
        "boost": 2.0
      }
    }
  }
}
```

**ВАЖНО:** Используйте `term` только для `keyword` полей!

```json
// Неправильно — text поле анализируется
{ "term": { "name": "Gaming Laptop" } }

// Правильно — keyword или multi-field
{ "term": { "name.keyword": "Gaming Laptop" } }
```

### terms (множественное совпадение)

```json
{
  "query": {
    "terms": {
      "category": ["electronics", "computers", "accessories"]
    }
  }
}
```

### terms lookup (из другого документа)

```json
{
  "query": {
    "terms": {
      "category": {
        "index": "user_preferences",
        "id": "user123",
        "path": "favorite_categories"
      }
    }
  }
}
```

### range (диапазон значений)

```json
{
  "query": {
    "range": {
      "price": {
        "gte": 100,
        "lte": 500,
        "boost": 2.0
      }
    }
  }
}
```

```json
// Для дат
{
  "query": {
    "range": {
      "created_at": {
        "gte": "now-1M/d",
        "lt": "now/d",
        "time_zone": "+03:00"
      }
    }
  }
}
```

| Оператор | Описание |
|----------|----------|
| **gt** | Больше (>) |
| **gte** | Больше или равно (>=) |
| **lt** | Меньше (<) |
| **lte** | Меньше или равно (<=) |

### Date math expressions

```
now                   # Текущий момент
now+1h                # Через час
now-1d                # Вчера
now/d                 # Начало текущего дня
2024-01-01||+1M/d     # 2024-02-01 00:00:00
```

### exists (поле существует)

```json
{
  "query": {
    "exists": {
      "field": "description"
    }
  }
}
```

### prefix (префикс)

```json
{
  "query": {
    "prefix": {
      "name.keyword": {
        "value": "Gaming",
        "case_insensitive": true
      }
    }
  }
}
```

### wildcard (шаблон)

```json
{
  "query": {
    "wildcard": {
      "email": {
        "value": "*@example.com",
        "case_insensitive": true
      }
    }
  }
}
```

**Символы:**
- `*` — любое количество символов
- `?` — один символ

### regexp (регулярные выражения)

```json
{
  "query": {
    "regexp": {
      "sku": {
        "value": "PRD-[0-9]{4}-[A-Z]+",
        "flags": "ALL",
        "max_determinized_states": 10000
      }
    }
  }
}
```

### fuzzy (поиск с опечатками)

```json
{
  "query": {
    "fuzzy": {
      "name": {
        "value": "laptpo",
        "fuzziness": "AUTO",
        "prefix_length": 2,
        "max_expansions": 50,
        "transpositions": true
      }
    }
  }
}
```

### ids (по идентификаторам)

```json
{
  "query": {
    "ids": {
      "values": ["1", "2", "3", "100"]
    }
  }
}
```

## Bool Query

Bool query — основной инструмент для комбинирования запросов.

### Структура

```json
{
  "query": {
    "bool": {
      "must": [],
      "filter": [],
      "should": [],
      "must_not": [],
      "minimum_should_match": 1
    }
  }
}
```

| Clause | Score | Описание |
|--------|-------|----------|
| **must** | Да | Обязательное совпадение, влияет на score |
| **filter** | Нет | Обязательное совпадение, не влияет на score |
| **should** | Да | Желательное совпадение, увеличивает score |
| **must_not** | Нет | Не должно совпадать |

### Пример комплексного запроса

```json
{
  "query": {
    "bool": {
      "must": [
        {
          "multi_match": {
            "query": "gaming laptop",
            "fields": ["name^2", "description"]
          }
        }
      ],
      "filter": [
        { "term": { "in_stock": true } },
        { "range": { "price": { "gte": 500, "lte": 2000 } } },
        { "terms": { "category": ["electronics", "computers"] } }
      ],
      "should": [
        { "term": { "featured": true } },
        { "range": { "rating": { "gte": 4.5 } } }
      ],
      "must_not": [
        { "term": { "status": "discontinued" } }
      ],
      "minimum_should_match": 1
    }
  }
}
```

### Вложенные bool queries

```json
{
  "query": {
    "bool": {
      "must": [
        { "match": { "title": "elasticsearch" } }
      ],
      "should": [
        {
          "bool": {
            "must": [
              { "match": { "category": "books" } },
              { "range": { "price": { "lt": 50 } } }
            ]
          }
        },
        {
          "bool": {
            "must": [
              { "match": { "category": "courses" } },
              { "term": { "format": "video" } }
            ]
          }
        }
      ],
      "minimum_should_match": 1
    }
  }
}
```

## Поиск по вложенным объектам

### Nested Query

```json
{
  "query": {
    "nested": {
      "path": "comments",
      "query": {
        "bool": {
          "must": [
            { "match": { "comments.author": "John" } },
            { "range": { "comments.stars": { "gte": 4 } } }
          ]
        }
      },
      "score_mode": "avg",
      "inner_hits": {
        "size": 3,
        "highlight": {
          "fields": {
            "comments.text": {}
          }
        }
      }
    }
  }
}
```

**score_mode:**
- `avg` — среднее score вложенных документов
- `max` — максимальное
- `min` — минимальное
- `sum` — сумма
- `none` — не учитывать

### Has Child / Has Parent

```json
// Найти родителей с определёнными детьми
{
  "query": {
    "has_child": {
      "type": "answer",
      "query": {
        "match": { "body": "elasticsearch" }
      },
      "score_mode": "max",
      "min_children": 1,
      "max_children": 10
    }
  }
}
```

```json
// Найти детей с определёнными родителями
{
  "query": {
    "has_parent": {
      "parent_type": "question",
      "query": {
        "match": { "title": "elasticsearch" }
      },
      "score": true
    }
  }
}
```

## Пагинация и сортировка

### From / Size

```json
{
  "from": 20,
  "size": 10,
  "query": {
    "match_all": {}
  }
}
```

**Ограничение:** `from + size <= 10000` (index.max_result_window)

### Search After (глубокая пагинация)

```json
// Первый запрос
{
  "size": 100,
  "query": { "match_all": {} },
  "sort": [
    { "created_at": "desc" },
    { "_id": "asc" }
  ]
}

// Следующие запросы — используем sort values последнего документа
{
  "size": 100,
  "query": { "match_all": {} },
  "sort": [
    { "created_at": "desc" },
    { "_id": "asc" }
  ],
  "search_after": ["2024-01-15T10:30:00Z", "abc123"]
}
```

### Scroll API (для экспорта)

```bash
# Инициализация scroll
curl -X POST "localhost:9200/products/_search?scroll=1m" -H 'Content-Type: application/json' -d'
{
  "size": 1000,
  "query": { "match_all": {} }
}'

# Получение следующих batch
curl -X POST "localhost:9200/_search/scroll" -H 'Content-Type: application/json' -d'
{
  "scroll": "1m",
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAA..."
}'

# Очистка scroll
curl -X DELETE "localhost:9200/_search/scroll" -H 'Content-Type: application/json' -d'
{
  "scroll_id": "DXF1ZXJ5QW5kRmV0Y2gBAAAAAA..."
}'
```

### Point in Time (PIT) — рекомендуется вместо scroll

```bash
# Создание PIT
curl -X POST "localhost:9200/products/_pit?keep_alive=1m"

# Поиск с PIT
curl -X POST "localhost:9200/_search" -H 'Content-Type: application/json' -d'
{
  "size": 100,
  "query": { "match_all": {} },
  "pit": {
    "id": "46ToAwMDaWR5...",
    "keep_alive": "1m"
  },
  "sort": [
    { "created_at": "desc" },
    { "_id": "asc" }
  ],
  "search_after": ["2024-01-15T10:30:00Z", "abc123"]
}'

# Закрытие PIT
curl -X DELETE "localhost:9200/_pit" -H 'Content-Type: application/json' -d'
{
  "id": "46ToAwMDaWR5..."
}'
```

### Сортировка

```json
{
  "sort": [
    { "price": { "order": "asc", "missing": "_last" } },
    { "rating": { "order": "desc" } },
    { "_score": { "order": "desc" } },
    "_id"
  ]
}
```

```json
// Сортировка по nested полю
{
  "sort": [
    {
      "offers.price": {
        "order": "asc",
        "nested": {
          "path": "offers",
          "filter": {
            "term": { "offers.active": true }
          }
        }
      }
    }
  ]
}
```

```json
// Сортировка по geo_distance
{
  "sort": [
    {
      "_geo_distance": {
        "location": { "lat": 55.75, "lon": 37.62 },
        "order": "asc",
        "unit": "km",
        "mode": "min"
      }
    }
  ]
}
```

## Полезные запросы

### match_all / match_none

```json
{ "query": { "match_all": {} } }
{ "query": { "match_none": {} } }
```

### boosting (понижение релевантности)

```json
{
  "query": {
    "boosting": {
      "positive": {
        "match": { "name": "laptop" }
      },
      "negative": {
        "term": { "brand": "unknown" }
      },
      "negative_boost": 0.5
    }
  }
}
```

### constant_score (фиксированный score)

```json
{
  "query": {
    "constant_score": {
      "filter": {
        "term": { "status": "published" }
      },
      "boost": 1.2
    }
  }
}
```

### dis_max (disjunction max)

```json
{
  "query": {
    "dis_max": {
      "queries": [
        { "match": { "title": "elasticsearch" } },
        { "match": { "body": "elasticsearch" } }
      ],
      "tie_breaker": 0.7
    }
  }
}
```

## Best Practices

### Оптимизация запросов

```
✅ Используйте filter context для фильтров без scoring
✅ Ставьте cheapest filters первыми в bool query
✅ Используйте term для keyword полей
✅ Используйте match для text полей
✅ Избегайте wildcard с leading wildcard (*term)
✅ Используйте search_after вместо deep pagination
✅ Кэшируйте часто используемые filter queries
```

### Типичные ошибки

```
❌ term query для text полей
❌ from + size > 10000
❌ Wildcard в начале паттерна
❌ Слишком сложные nested queries
❌ Игнорирование _score при полнотекстовом поиске
```

### Пример оптимизированного запроса

```json
{
  "query": {
    "bool": {
      "filter": [
        { "term": { "status": "published" } },
        { "range": { "stock": { "gt": 0 } } }
      ],
      "must": [
        {
          "multi_match": {
            "query": "gaming laptop",
            "fields": ["name^3", "description"],
            "type": "best_fields",
            "fuzziness": "AUTO"
          }
        }
      ],
      "should": [
        { "term": { "featured": { "value": true, "boost": 2.0 } } }
      ]
    }
  },
  "size": 20,
  "sort": [
    "_score",
    { "created_at": "desc" }
  ],
  "_source": ["name", "price", "image_url"]
}
```
